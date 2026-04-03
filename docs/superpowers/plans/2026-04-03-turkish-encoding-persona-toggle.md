# Turkish Encoding Fix & Chatbot Persona Toggle Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix missing Turkish characters in chatbot responses and add a configurable AI/human persona mode to the onboarding flow.

**Architecture:** Encoding is fixed by rewriting `generateSystemPrompt.js` with proper UTF-8 Turkish characters; the model was mirroring the ASCII-only style of the prompt. Persona branching adds `chatbot_persona`, `greeting_style`, and `assistant_name` keys to the existing `chatbot_config` JSONB column (no migration). Validation is extracted into a pure helper. Frontend gets new controls in the existing chatbot config section.

**Tech Stack:** Node.js/Express (webhook service), Jest + Supertest (tests), plain HTML/JS (onboarding frontend), PostgreSQL via `pg` pool.

**Repo root:** `C:/Users/merve/OneDrive/Belgeler/GitHub/appointment-chatbot`
**Run tests from:** `C:/Users/merve/OneDrive/Belgeler/GitHub/appointment-chatbot/webhook`

---

## File Map

| File | Action | Purpose |
|---|---|---|
| `webhook/tests/generateSystemPrompt.test.js` | Create | Unit tests for encoding + persona branching |
| `webhook/src/helpers/generateSystemPrompt.js` | Rewrite | UTF-8 encoding fix + persona branching logic |
| `webhook/tests/validateProfileConfig.test.js` | Create | Unit tests for persona validation rule |
| `webhook/src/helpers/validateProfileConfig.js` | Create | Pure function: validates assistant_name requirement |
| `webhook/src/routes/onboarding.js` | Modify | Wire validation into `/profile` handler |
| `onboarding/index.html` | Modify | Add persona + greeting_style + assistant_name controls |
| `onboarding/app.js` | Modify | collectChatbotConfig, restoreChatbotConfig, togglePersonaUI |

---

## Task 1: Write failing tests for generateSystemPrompt

**Files:**
- Create: `webhook/tests/generateSystemPrompt.test.js`

- [ ] **Step 1: Create the test file**

```js
// webhook/tests/generateSystemPrompt.test.js
'use strict';

const { generateSystemPrompt } = require('../src/helpers/generateSystemPrompt');

const BASE = {
  business_name: 'Test Kuaför',
  services: [{ name: 'Saç Kesimi', duration_minutes: 30, price_cents: 15000 }],
  opening_hours: { mon: { open: '09:00', close: '18:00' }, tue: null },
  staff: [{ name: 'Ali', specialties: ['Saç'] }],
  staff_mode: 'auto',
  chatbot_config: {},
};

// --- Encoding tests ---

test('uses proper Turkish ş/ğ/ü/ö/ı/ç in output', () => {
  const prompt = generateSystemPrompt(BASE);
  expect(prompt).toContain('Müşteri');
  expect(prompt).toContain('Çalışma');
  expect(prompt).toContain('yardımcı');
  expect(prompt).toContain('Türkçe');
  expect(prompt).toContain('başarısız');
});

test('does not contain ASCII-transliterated Turkish', () => {
  const prompt = generateSystemPrompt(BASE);
  expect(prompt).not.toContain('Musteri');
  expect(prompt).not.toContain('Calisma');
  expect(prompt).not.toContain('yardimci');
  expect(prompt).not.toContain('Turkce');
  expect(prompt).not.toContain('basarisiz');
});

test('day names use proper Turkish characters', () => {
  const input = { ...BASE, opening_hours: { tue: { open: '09:00', close: '17:00' }, wed: { open: '10:00', close: '18:00' }, thu: { open: '09:00', close: '17:00' } } };
  const prompt = generateSystemPrompt(input);
  expect(prompt).toContain('Salı');
  expect(prompt).toContain('Çarşamba');
  expect(prompt).toContain('Perşembe');
});

// --- AI persona (default) ---

test('AI mode: role block mentions yapay zeka', () => {
  const prompt = generateSystemPrompt(BASE);
  expect(prompt).toContain('yapay zeka randevu asistanısın');
});

test('AI mode: intro rule mentions AI asistanıyım', () => {
  const prompt = generateSystemPrompt(BASE);
  expect(prompt).toContain('AI asistanıyım');
});

test('AI mode: no human disclosure rule', () => {
  const prompt = generateSystemPrompt(BASE);
  expect(prompt).not.toContain('DOĞRUDAN sorulursa');
});

// --- Human persona ---

const HUMAN_BASE = { ...BASE, chatbot_config: { chatbot_persona: 'human', greeting_style: 'business_name' } };

test('human mode: role block uses müşteri temsilcisisin', () => {
  const prompt = generateSystemPrompt(HUMAN_BASE);
  expect(prompt).toContain('müşteri temsilcisisin');
  expect(prompt).not.toContain('yapay zeka randevu asistanısın');
});

test('human mode, business_name greeting: uses business name in intro', () => {
  const prompt = generateSystemPrompt(HUMAN_BASE);
  expect(prompt).toContain('Test Kuaför ekibinden');
  expect(prompt).not.toContain('AI asistanıyım');
});

test('human mode, assistant_name greeting: uses assistant name', () => {
  const input = { ...BASE, chatbot_config: { chatbot_persona: 'human', greeting_style: 'assistant_name', assistant_name: 'Ayşe' } };
  const prompt = generateSystemPrompt(input);
  expect(prompt).toContain('Ben Ayşe');
  expect(prompt).toContain('Test Kuaför ekibinden');
});

test('human mode, none greeting: no intro rule in output', () => {
  const input = { ...BASE, chatbot_config: { chatbot_persona: 'human', greeting_style: 'none' } };
  const prompt = generateSystemPrompt(input);
  expect(prompt).not.toContain('İlk mesajda');
});

test('human mode: includes AI disclosure rule', () => {
  const prompt = generateSystemPrompt(HUMAN_BASE);
  expect(prompt).toContain('DOĞRUDAN sorulursa');
});
```

- [ ] **Step 2: Run tests — expect ALL to fail**

```bash
cd C:/Users/merve/OneDrive/Belgeler/GitHub/appointment-chatbot/webhook
npx jest tests/generateSystemPrompt.test.js --runInBand
```

Expected: all 11 tests FAIL (encoding tests fail because current code has ASCII; persona tests fail because branching doesn't exist yet).

---

## Task 2: Rewrite generateSystemPrompt.js (encoding fix + persona branching)

**Files:**
- Modify: `webhook/src/helpers/generateSystemPrompt.js`

- [ ] **Step 1: Replace the entire file**

```js
// webhook/src/helpers/generateSystemPrompt.js
'use strict';

/**
 * Generates a Turkish system prompt for the appointment chatbot.
 * Called by /activate (initial provisioning) and /regenerate-prompt.
 *
 * chatbot_config keys used here:
 *   tone: 'siz' | 'sen'
 *   chatbot_persona: 'ai' | 'human'  (default: 'ai')
 *   greeting_style: 'business_name' | 'assistant_name' | 'none'  (default: 'business_name')
 *   assistant_name: string  (required when greeting_style === 'assistant_name')
 *   walkin_policy, payment_methods, description, cancellation_policy,
 *   min_booking_lead, location_info, faq  (unchanged)
 */

const DAY_NAMES = {
  mon: 'Pazartesi',
  tue: 'Salı',
  wed: 'Çarşamba',
  thu: 'Perşembe',
  fri: 'Cuma',
  sat: 'Cumartesi',
  sun: 'Pazar',
};

function generateSystemPrompt({ business_name, services, opening_hours, staff, staff_mode, chatbot_config }) {
  const config = chatbot_config || {};
  const persona = config.chatbot_persona || 'ai';
  const greetingStyle = config.greeting_style || 'business_name';
  const assistantName = config.assistant_name || '';

  // --- Tone ---
  const tone = config.tone || 'siz';
  const toneRule = tone === 'sen'
    ? 'Müşteri ile SAMİMİ ton kullan — "sen" ile hitap et'
    : 'Müşteri ile RESMİ ton kullan — "siz" ile hitap et';

  // --- Services ---
  const servicesList = services.map(s =>
    `- **${s.name}**: ${s.duration_minutes} dakika` +
    (s.price_cents ? `, ${Math.round(s.price_cents / 100)} TL` : '')
  ).join('\n');

  // --- Hours ---
  const hoursList = Object.entries(opening_hours || {})
    .filter(([, v]) => v && v.open)
    .map(([day, times]) => `- ${DAY_NAMES[day] || day}: ${times.open} - ${times.close}`)
    .join('\n');

  // --- Staff ---
  const staffList = staff.map(s => {
    const specs = Array.isArray(s.specialties) ? s.specialties : [];
    return `- ${s.name}` + (specs.length ? ` (${specs.join(', ')})` : '');
  }).join('\n');

  const staffModeText = staff_mode === 'specific'
    ? 'Müşteri tercih ettiği personeli seçer'
    : 'Sistem uygun personeli otomatik atar';

  // --- Walk-in policy ---
  const walkinMap = {
    evet: 'Evet, randevusuz geliş kabul edilir',
    hayir: 'Hayır, randevu zorunludur',
    'ara-sor': 'Önce arayıp sormak gerekir',
  };
  const walkinText = walkinMap[config.walkin_policy] || 'Randevu zorunludur';

  // --- Payment methods ---
  const paymentMap = { nakit: 'Nakit', kredi_karti: 'Kredi/Banka Kartı', havale: 'Havale/EFT' };
  const paymentMethods = Array.isArray(config.payment_methods) ? config.payment_methods : [];
  const paymentText = paymentMethods.length
    ? paymentMethods.map(p => paymentMap[p] || p).join(', ')
    : 'Belirtilmemiş';

  // --- Optional sections ---
  const descriptionLine = config.description
    ? `\n**İşletme Hakkında:** ${config.description}\n`
    : '';

  const cancellationLine = config.cancellation_policy || 'Belirtilmemiş';
  const minBookingLine = config.min_booking_lead || 'Belirtilmemiş';

  const locationSection = config.location_info
    ? `\n## Konum ve Erişim\n${config.location_info}\n`
    : '';

  // --- FAQ ---
  const faqItems = Array.isArray(config.faq) ? config.faq.filter(f => f.q && f.a) : [];
  const faqSection = faqItems.length
    ? '\n# SIKÇA SORULAN SORULAR\n\n' + faqItems.map(f => `**S: ${f.q}**\nC: ${f.a}`).join('\n\n')
    : '';

  // --- Persona-specific blocks ---
  let roleBlock, introRuleLine, humanDisclosureRule;

  if (persona === 'human') {
    roleBlock = `Sen **${business_name}** işletmesinin bir müşteri temsilcisisin. Müşterilerle WhatsApp ve Instagram üzerinden iletişim kuruyorsun.`;

    if (greetingStyle === 'business_name') {
      introRuleLine = `\n2. **İlk mesajda** kendini şöyle tanıt: "Merhaba! ${business_name} ekibinden, size nasıl yardımcı olabilirim?"`;
    } else if (greetingStyle === 'assistant_name') {
      introRuleLine = `\n2. **İlk mesajda** kendini şöyle tanıt: "Merhaba! Ben ${assistantName}, ${business_name} ekibinden. Nasıl yardımcı olabilirim?"`;
    } else {
      introRuleLine = ''; // greeting_style: 'none' — no forced intro
    }

    humanDisclosureRule = '\n8. Yapay zeka veya bot olup olmadığın **DOĞRUDAN sorulursa** dürüstçe söyle: "Evet, yapay zeka destekli bir asistanım."';
  } else {
    // AI persona (default)
    roleBlock = `Sen **${business_name}** işletmesinin yapay zeka randevu asistanısın. Müşterilerle WhatsApp ve Instagram üzerinden iletişim kuruyorsun.`;
    introRuleLine = `\n2. **İlk mesajda** kendini şöyle tanıtmayı unutma: "Merhaba! Ben ${business_name} AI asistanıyım. Size nasıl yardımcı olabilirim?" — Bu META politikası gereğince ZORUNLUDUR.`;
    humanDisclosureRule = '';
  }

  return `# GÖREV VE ROL

${roleBlock}
${descriptionLine}
**Temel görevlerin:**
1. Randevu almak, iptal etmek ve yeniden planlamak
2. Hizmetler, fiyatlar ve uygunluk hakkında bilgi vermek
3. Sık sorulan soruları yanıtlamak

# KRİTİK KURALLAR

1. **HER ZAMAN Türkçe yaz.** Başka dile geçme.${introRuleLine}
3. **${toneRule}** — tüm mesajlarda bu hitap biçimini koru.
4. Müşteriyi **en fazla 2 kez** anlamaya çalış. 2 başarısız denemeden sonra MUTLAKA \`human_handoff\` aksiyonu ver: "Sizi bir yetkiliye bağlayacağım."
5. **Sadece aşağıdaki Hizmetler listesindeki hizmetler** için randevu al. Listede olmayan hizmet için: "Bu hizmeti şu an sunmuyoruz, başka bir konuda yardımcı olabilir miyim?"
6. Randevu için şu bilgilerin TÜMÜ gereklidir: **hizmet + tarih + saat**. Eksik varsa sor, tahmin etme veya uydurma.
7. Yanlış fiyat, yanlış saat veya uydurma bilgi VERME. Emin olmadığında: "Emin olmak için sizi yetkiliye bağlayayım."${humanDisclosureRule}

# AKSİYON ŞEMASI

Her yanıtının sonunda aşağıdaki formattan BİRİNİ JSON olarak ekle. Bu zorunludur:

Randevu al: \`{"action": "book_appointment", "service": "...", "staff": "...", "date": "YYYY-MM-DD", "time": "HH:MM"}\`
İptal et: \`{"action": "cancel_appointment", "booking_id": "..."}\`
Yeniden planla: \`{"action": "reschedule_appointment", "booking_id": "...", "new_date": "YYYY-MM-DD", "new_time": "HH:MM"}\`
Uygunluk sorgula: \`{"action": "check_availability", "service": "...", "date": "YYYY-MM-DD"}\`
Mesaj gönder: \`{"action": "send_message", "message": "..."}\`
İnsan yetkilisi: \`{"action": "human_handoff", "reason": "..."}\`

# İŞLETME BİLGİLERİ

## Hizmetler
${servicesList}

## Çalışma Saatleri
${hoursList}

## Personel
${staffList}
**Personel atama:** ${staffModeText}
${locationSection}
# POLİTİKALAR

| Konu | Bilgi |
|------|-------|
| İptal/Değişiklik | ${cancellationLine} |
| En erken randevu | ${minBookingLine} |
| Randevusuz geliş (walk-in) | ${walkinText} |
| Ödeme yöntemleri | ${paymentText} |
${faqSection}`;
}

module.exports = { generateSystemPrompt };
```

- [ ] **Step 2: Run tests — expect ALL to pass**

```bash
cd C:/Users/merve/OneDrive/Belgeler/GitHub/appointment-chatbot/webhook
npx jest tests/generateSystemPrompt.test.js --runInBand
```

Expected: all 11 tests PASS.

- [ ] **Step 3: Run full test suite to check no regressions**

```bash
npm test
```

Expected: all previously passing tests still pass.

- [ ] **Step 4: Commit**

```bash
cd C:/Users/merve/OneDrive/Belgeler/GitHub/appointment-chatbot
git add webhook/src/helpers/generateSystemPrompt.js webhook/tests/generateSystemPrompt.test.js
git commit -m "fix: correct Turkish encoding and add AI/human persona branching in system prompt"
```

---

## Task 3: Write failing tests for validateProfileConfig

**Files:**
- Create: `webhook/tests/validateProfileConfig.test.js`

- [ ] **Step 1: Create the test file**

```js
// webhook/tests/validateProfileConfig.test.js
'use strict';

const { validateProfileConfig } = require('../src/helpers/validateProfileConfig');

test('returns null when chatbot_config is absent', () => {
  expect(validateProfileConfig({})).toBeNull();
});

test('returns null for AI persona (no assistant_name constraint)', () => {
  expect(validateProfileConfig({ chatbot_config: { chatbot_persona: 'ai' } })).toBeNull();
});

test('returns null for human persona with business_name greeting', () => {
  expect(validateProfileConfig({
    chatbot_config: { chatbot_persona: 'human', greeting_style: 'business_name' },
  })).toBeNull();
});

test('returns null for human persona with none greeting', () => {
  expect(validateProfileConfig({
    chatbot_config: { chatbot_persona: 'human', greeting_style: 'none' },
  })).toBeNull();
});

test('returns error for human + assistant_name greeting with empty name', () => {
  expect(validateProfileConfig({
    chatbot_config: { chatbot_persona: 'human', greeting_style: 'assistant_name', assistant_name: '' },
  })).toBe('Asistan adıyla karşılama için asistan adı gereklidir');
});

test('returns error for human + assistant_name greeting with whitespace-only name', () => {
  expect(validateProfileConfig({
    chatbot_config: { chatbot_persona: 'human', greeting_style: 'assistant_name', assistant_name: '   ' },
  })).toBe('Asistan adıyla karşılama için asistan adı gereklidir');
});

test('returns null for human + assistant_name greeting with valid name', () => {
  expect(validateProfileConfig({
    chatbot_config: { chatbot_persona: 'human', greeting_style: 'assistant_name', assistant_name: 'Ayşe' },
  })).toBeNull();
});
```

- [ ] **Step 2: Run — expect FAIL (module not found)**

```bash
cd C:/Users/merve/OneDrive/Belgeler/GitHub/appointment-chatbot/webhook
npx jest tests/validateProfileConfig.test.js --runInBand
```

Expected: FAIL — `Cannot find module '../src/helpers/validateProfileConfig'`

---

## Task 4: Create validateProfileConfig helper and wire into onboarding.js

**Files:**
- Create: `webhook/src/helpers/validateProfileConfig.js`
- Modify: `webhook/src/routes/onboarding.js` (lines 131-136, after existing input checks)

- [ ] **Step 1: Create the helper**

```js
// webhook/src/helpers/validateProfileConfig.js
'use strict';

/**
 * Validates chatbot_config persona fields from a /profile request body.
 * Returns an error string if invalid, null if valid.
 */
function validateProfileConfig({ chatbot_config }) {
  const config = chatbot_config || {};
  if (
    config.chatbot_persona === 'human' &&
    config.greeting_style === 'assistant_name' &&
    !String(config.assistant_name || '').trim()
  ) {
    return 'Asistan adıyla karşılama için asistan adı gereklidir';
  }
  return null;
}

module.exports = { validateProfileConfig };
```

- [ ] **Step 2: Run validation tests — expect PASS**

```bash
cd C:/Users/merve/OneDrive/Belgeler/GitHub/appointment-chatbot/webhook
npx jest tests/validateProfileConfig.test.js --runInBand
```

Expected: all 7 tests PASS.

- [ ] **Step 3: Wire validation into onboarding.js**

In `webhook/src/routes/onboarding.js`, add the import at the top (after existing requires):

```js
const { validateProfileConfig } = require('../helpers/validateProfileConfig');
```

Then in the `POST /profile` handler, add validation after the existing input checks (after line checking `!opening_hours`) and before `pool.connect()`:

```js
// Validate persona config
const configError = validateProfileConfig({ chatbot_config });
if (configError) return res.status(400).json({ error: configError });
```

The section should look like this after the change:

```js
router.post('/profile', tokenAuth, async (req, res) => {
  try {
    const { services, staff: staffMembers, opening_hours, staff_mode, chatbot_config } = req.body;
    const businessId = req.business.id;

    if (!services?.length) return res.status(400).json({ error: 'At least one service required' });
    if (!staffMembers?.length) return res.status(400).json({ error: 'At least one staff member required' });
    if (!opening_hours) return res.status(400).json({ error: 'Opening hours required' });

    // Validate persona config
    const configError = validateProfileConfig({ chatbot_config });
    if (configError) return res.status(400).json({ error: configError });

    const client = await pool.connect();
    // ... rest unchanged
```

- [ ] **Step 4: Run full test suite**

```bash
cd C:/Users/merve/OneDrive/Belgeler/GitHub/appointment-chatbot/webhook
npm test
```

Expected: all tests pass.

- [ ] **Step 5: Commit**

```bash
cd C:/Users/merve/OneDrive/Belgeler/GitHub/appointment-chatbot
git add webhook/src/helpers/validateProfileConfig.js webhook/tests/validateProfileConfig.test.js webhook/src/routes/onboarding.js
git commit -m "feat: add persona config validation to onboarding profile route"
```

---

## Task 5: Add persona controls to onboarding index.html

**Files:**
- Modify: `onboarding/index.html`

- [ ] **Step 1: Find the insertion point**

Open `onboarding/index.html`. Locate the line:
```html
<h3>Chatbot AI Ayarlari</h3>
```

The tone radio group follows immediately. Insert the new persona block **after the closing `</div>` of the tone radio group** and **before the walk-in policy section** (search for `<!-- Walk-in` or the `h4` for walk-in).

- [ ] **Step 2: Insert the persona HTML block**

Add this block after the tone radio group's closing `</div>`:

```html
        <!-- Chatbot Persona -->
        <h4>Chatbot Kimliği</h4>
        <div class="radio-group">
          <label class="radio-label">
            <input type="radio" name="chatbot_persona" value="ai" checked>
            <span><strong>Yapay Zeka</strong> — AI asistanı olduğunu açıkça belirtir</span>
          </label>
          <label class="radio-label">
            <input type="radio" name="chatbot_persona" value="human">
            <span><strong>İnsan</strong> — Ekip üyesi gibi davranır; sadece doğrudan sorulursa AI olduğunu söyler</span>
          </label>
        </div>

        <div id="persona-human-options" style="display:none">
          <h4>Karşılama Şekli</h4>
          <div class="radio-group">
            <label class="radio-label">
              <input type="radio" name="greeting_style" value="business_name" checked>
              <span><strong>İşletme adıyla</strong> — "Merhaba! [İşletme] ekibinden..."</span>
            </label>
            <label class="radio-label">
              <input type="radio" name="greeting_style" value="assistant_name">
              <span><strong>Asistan adıyla</strong> — "Merhaba! Ben [Ad], [İşletme] ekibinden..."</span>
            </label>
            <label class="radio-label">
              <input type="radio" name="greeting_style" value="none">
              <span><strong>Karşılama yok</strong> — Müşterinin mesajına doğrudan yanıt verir</span>
            </label>
          </div>

          <div id="assistant-name-section" style="display:none">
            <label for="chatbot_assistant_name">Asistan Adı</label>
            <input type="text" id="chatbot_assistant_name" placeholder="örn. Ayşe" style="margin-top:4px">
          </div>
        </div>
```

- [ ] **Step 3: Verify HTML is well-formed**

Open `onboarding/index.html` and confirm the new block is inside the `chatbot-config-section` div and all tags are properly closed.

---

## Task 6: Update app.js (collectChatbotConfig, restoreChatbotConfig, togglePersonaUI)

**Files:**
- Modify: `onboarding/app.js`

- [ ] **Step 1: Add togglePersonaUI function**

Find the `collectChatbotConfig` function declaration in `app.js`. Insert the following **before** it:

```js
  function togglePersonaUI() {
    var persona = document.querySelector('input[name="chatbot_persona"]:checked');
    var humanOptions = document.getElementById('persona-human-options');
    var assistantNameSection = document.getElementById('assistant-name-section');
    var greetingStyle = document.querySelector('input[name="greeting_style"]:checked');

    if (!humanOptions) return;
    humanOptions.style.display = (persona && persona.value === 'human') ? 'block' : 'none';

    if (assistantNameSection) {
      assistantNameSection.style.display = (greetingStyle && greetingStyle.value === 'assistant_name') ? 'block' : 'none';
    }
  }
```

- [ ] **Step 2: Add event listeners for persona and greeting_style radios**

Find where the existing `chatbot_tone` change listener is set up (search for `chatbot_tone` in the event listener section of `app.js`). Add these two listener registrations alongside it:

```js
    // Persona toggle
    document.querySelectorAll('input[name="chatbot_persona"]').forEach(function (radio) {
      radio.addEventListener('change', togglePersonaUI);
    });
    document.querySelectorAll('input[name="greeting_style"]').forEach(function (radio) {
      radio.addEventListener('change', togglePersonaUI);
    });
```

- [ ] **Step 3: Update collectChatbotConfig to read persona fields**

In the existing `collectChatbotConfig` function, locate where `tone` and `walkin` are read. Add these three lines in the same block:

```js
    var persona = document.querySelector('input[name="chatbot_persona"]:checked');
    var greetingStyle = document.querySelector('input[name="greeting_style"]:checked');
    var assistantName = document.getElementById('chatbot_assistant_name');
```

Then in the returned config object, add the three new keys alongside `tone` and `walkin_policy`:

```js
      chatbot_persona: persona ? persona.value : 'ai',
      greeting_style: greetingStyle ? greetingStyle.value : 'business_name',
      assistant_name: assistantName ? (assistantName.value || '').trim() : '',
```

- [ ] **Step 4: Update restoreChatbotConfig to restore persona fields**

In the existing `restoreChatbotConfig` function, after the walk-in restore block, add:

```js
    // Chatbot persona
    if (config.chatbot_persona) {
      var personaRadio = document.querySelector('input[name="chatbot_persona"][value="' + config.chatbot_persona + '"]');
      if (personaRadio) personaRadio.checked = true;
    }

    // Greeting style
    if (config.greeting_style) {
      var greetingRadio = document.querySelector('input[name="greeting_style"][value="' + config.greeting_style + '"]');
      if (greetingRadio) greetingRadio.checked = true;
    }

    // Assistant name
    var assistantNameEl = document.getElementById('chatbot_assistant_name');
    if (assistantNameEl && config.assistant_name) assistantNameEl.value = config.assistant_name;

    // Sync visibility after restoring
    togglePersonaUI();
```

- [ ] **Step 5: Call togglePersonaUI on page load**

Find the section where `restoreChatbotConfig` is called on page load (search for `restoreChatbotConfig(savedConfig)`). Add `togglePersonaUI()` immediately after that call:

```js
    if (savedConfig) {
      restoreChatbotConfig(savedConfig);
    }
    togglePersonaUI(); // sync initial visibility state
```

- [ ] **Step 6: Commit**

```bash
cd C:/Users/merve/OneDrive/Belgeler/GitHub/appointment-chatbot
git add onboarding/index.html onboarding/app.js
git commit -m "feat: add persona toggle controls to onboarding UI"
```

---

## Task 7: Switch test business to human mode and regenerate prompt

This lets you test immediately without going through the full onboarding flow again.

- [ ] **Step 1: Update chatbot_config in the database**

Run via the PostgreSQL MCP (connected to `chatbot_db` on `localhost:5432`):

```sql
UPDATE businesses
SET chatbot_config = chatbot_config || '{"chatbot_persona":"human","greeting_style":"business_name"}'::jsonb
WHERE whatsapp_number = (SELECT current_setting('app.test_phone_number_id', true))
   OR name ILIKE '%test%'
RETURNING id, name, chatbot_config;
```

If that doesn't match, identify the business first:

```sql
SELECT id, name, whatsapp_number, chatbot_config FROM businesses ORDER BY created_at DESC LIMIT 5;
```

Then update by id:

```sql
UPDATE businesses
SET chatbot_config = chatbot_config || '{"chatbot_persona":"human","greeting_style":"business_name"}'::jsonb
WHERE id = '<business-id-from-above>'
RETURNING id, name, chatbot_config;
```

- [ ] **Step 2: Regenerate the system prompt**

The webhook service must be running. Call the regenerate-prompt endpoint using the business's access_token:

```bash
# Get the access token first
# (via DB: SELECT access_token FROM businesses WHERE id = '<id>')

curl -X POST http://localhost:3001/api/onboard/regenerate-prompt \
  -H "Authorization: Bearer <access_token>"
```

Expected response: `{"success":true}`

- [ ] **Step 3: Verify the new prompt in the DB**

```sql
SELECT name, system_prompt FROM businesses WHERE id = '<id>';
```

Confirm the prompt:
- Contains `müşteri temsilcisisin` (not `yapay zeka randevu asistanısın`)
- Contains `ekibinden` in the intro rule
- Does NOT contain `AI asistanıyım`
- Contains `DOĞRUDAN sorulursa`
- Uses proper Turkish characters throughout (ş, ğ, ü, ö, ı, ç)

- [ ] **Step 4: Send a WhatsApp test message**

Send a message via the test payload or directly from your phone. Verify:
1. Greeting uses business name, no AI disclosure
2. Response contains proper Turkish characters (şş, ğğ, üü, öö, ıı, çç)
3. Send "Sen bir bot musun?" → chatbot discloses it's AI

---
