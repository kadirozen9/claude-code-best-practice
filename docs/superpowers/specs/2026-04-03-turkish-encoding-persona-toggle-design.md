# Turkish Encoding Fix & Chatbot Persona Toggle

**Date:** 2026-04-03
**Project:** Appointment Chatbot SaaS
**Status:** Approved

## Problem Statement

Two issues discovered during first live WhatsApp test:

1. **Turkish encoding bug** — The chatbot responds in Turkish but uses ASCII-only characters (missing ş, ğ, ü, ö, ı, ç, İ etc.). Root cause: `generateSystemPrompt.js` uses ASCII-transliterated Turkish throughout, and the model mirrors this style in its replies.

2. **AI disclosure** — The system prompt hardcodes a mandatory first-message rule announcing the bot is an AI. Some businesses do not want this disclosed upfront. A configurable persona mode is needed.

---

## Design

### 1. Data Model

No database migration required. Two new keys added to the existing `chatbot_config` JSONB column on the `businesses` table:

```json
{
  "chatbot_persona": "ai" | "human",
  "assistant_name": "Ayşe",
  "greeting_style": "business_name" | "assistant_name" | "none"
}
```

**Defaults:**
- `chatbot_persona`: `"ai"` (existing behavior preserved when field is absent)
- `greeting_style`: `"business_name"`
- `assistant_name`: `null`

**Validation rule** (enforced in `/api/onboard/profile`):
- If `chatbot_persona === 'human'` AND `greeting_style === 'assistant_name'` AND `assistant_name` is empty → HTTP 400

### 2. System Prompt Logic (`generateSystemPrompt.js`)

#### 2a — Encoding fix

All ASCII-transliterated Turkish strings replaced with proper UTF-8 characters. The source file is already UTF-8; the transliteration was an oversight. Every occurrence of words like `musteri → müşteri`, `Carsamba → Çarşamba`, `asistanisin → asistanısın`, `yardimci → yardımcı`, etc. is corrected.

#### 2b — Persona branching

Two sections of the prompt change based on `chatbot_config.chatbot_persona`:

**ROLE block (top of prompt):**

| Mode | Role text |
|---|---|
| `ai` | `"Sen [İşletme] yapay zeka randevu asistanısın."` |
| `human` | `"Sen [İşletme] ekibinin bir müşteri temsilcisisin."` |

**CRITICAL RULES — Rule 2 (intro message):**

| Mode | `greeting_style` | Rule text |
|---|---|---|
| `ai` | — | `"İlk mesajda: 'Merhaba! Ben [X] AI asistanıyım. Size nasıl yardımcı olabilirim?'"` |
| `human` | `business_name` | `"İlk mesajda: 'Merhaba! [İşletme] ekibinden, size nasıl yardımcı olabilirim?'"` |
| `human` | `assistant_name` | `"İlk mesajda: 'Merhaba! Ben [assistant_name], [İşletme] ekibinden. Nasıl yardımcı olabilirim?'"` |
| `human` | `none` | Rule 2 omitted entirely — no forced intro |

**Additional rule in human mode only:**

> "Yapay zeka veya bot olup olmadığın DOĞRUDAN sorulursa dürüstçe söyle: 'Evet, yapay zeka destekli bir asistanım.'"

All other sections (booking rules, action schema, business data, policies) are identical regardless of persona.

### 3. Onboarding Frontend

#### `index.html`

Add a new control group inside the existing "Chatbot AI Ayarları" section, positioned after the tone radio group:

```
Chatbot Kimliği
  ○ Yapay Zeka  — "Bu bir AI asistanıdır" şeffaflığıyla çalışır
  ○ İnsan       — Ekip üyesi gibi davranır; sadece doğrudan sorulursa AI olduğunu söyler

[shown only when "İnsan" selected]
Karşılama Şekli
  ○ İşletme adıyla  — "Merhaba! [İşletme] ekibinden..."
  ○ Asistan adıyla  — "Merhaba! Ben [Ad], [İşletme] ekibinden..."
  ○ Karşılama yok   — Müşterinin mesajına doğrudan yanıt verir

[shown only when "Asistan adıyla" selected]
Asistan Adı: [text input]
```

#### `app.js`

1. **`collectChatbotConfig()`** — read `chatbot_persona`, `greeting_style`, `assistant_name` and include in returned config object
2. **`restoreChatbotConfig()`** — restore those three fields and call `togglePersonaUI()` to set visibility
3. **`togglePersonaUI()`** — new helper called on persona radio `change` event; shows/hides greeting_style group and assistant_name input based on current selections

#### `onboarding.js` (server-side)

Add validation: if `chatbot_persona === 'human'` AND `greeting_style === 'assistant_name'` AND `assistant_name` is empty → return 400 with message `"Asistan adıyla karşılama için asistan adı gereklidir"`.

---

## Test Plan

1. Switch existing test business to `chatbot_persona: 'human'`, `greeting_style: 'business_name'` via direct DB update → send WhatsApp message → verify no AI disclosure in greeting
2. Send "Sen bir bot musun?" → verify chatbot discloses AI status
3. Switch to `greeting_style: 'assistant_name'`, `assistant_name: 'Ayşe'` → verify personalized greeting
4. Switch to `greeting_style: 'none'` → verify no greeting, direct response
5. Switch back to `chatbot_persona: 'ai'` → verify original AI intro returns
6. Verify Turkish characters appear correctly in all response modes (ş, ğ, ü, ö, ı, ç)
7. Submit onboarding form with `greeting_style: 'assistant_name'` and empty name → verify 400 response

---

## Files Changed

| File | Change |
|---|---|
| `webhook/src/helpers/generateSystemPrompt.js` | Encoding fix + persona branching |
| `webhook/src/routes/onboarding.js` | Validation for assistant_name |
| `onboarding/index.html` | Persona + greeting_style + assistant_name controls |
| `onboarding/app.js` | `collectChatbotConfig`, `restoreChatbotConfig`, `togglePersonaUI` |

No database migration. No new dependencies.
