# Appointment Chatbot SaaS — Design Spec

**Date:** 2026-03-23
**Status:** Approved
**Authors:** User + Claude

---

## 1. Product Overview

A SaaS platform that gives service businesses (salons, clinics, spas) an AI chatbot that handles customer conversations on WhatsApp and Instagram — booking appointments, answering FAQs, sending reminders, and managing cancellations — 24/7 without human involvement.

Business owners access a self-hosted Cal.com dashboard to view, manually add, edit, and cancel appointments.

**Pricing:** 5,000–10,000 TRY/month per business (~$113–226 USD at March 2026 rates)
**Target market:** Beauty salons, clinics, spas — Turkey-focused initially
**Language:** Turkish (all chatbot responses in Turkish; system prompts authored in Turkish)

---

## 2. Architecture Overview

```
Customer (WhatsApp / Instagram)
        ↓
[Messaging Layer]
  - WhatsApp: Meta Cloud API (Embedded Signup)
  - Instagram: Meta Graph API
        ↓ webhook (HMAC-verified)
[Webhook Receiver] — lightweight Express service
        ↓
[Orchestration Layer] — n8n (self-hosted)
        ↓
[AI Chatbot] — Claude API
  - System prompt per business (services, staff, hours)
  - Conversation state in PostgreSQL
        ↓ structured actions (validated)
[Scheduling Layer] — Cal.com (self-hosted)
  - Per-business organization
  - Business owner calendar dashboard
        ↑
[Platform Admin] — internal dashboard (platform operator)
        ↑
[Billing] — İyzico Subscription API
  - Monthly recurring subscription per business
```

**Infrastructure:** Single Hetzner CX41 VPS (~$20–24/month, 4 vCPU, 16GB RAM), Docker Compose
**Stack:** Cal.com, n8n, PostgreSQL, Redis, Meta APIs, Claude API, İyzico, Express (webhook receiver)

> **Pre-build validation required:** Before committing to self-hosted Cal.com Organizations for multi-tenancy, verify that the Organizations feature works fully in self-hosted mode. Cal.com's self-hosted documentation lags behind the cloud product — specifically check that per-organization staff, services, and availability isolation work as expected by deploying a test instance first.

---

## 3. System Components

### 3.1 Messaging Layer

**WhatsApp — Meta Cloud API**
- Platform registers one Meta Business App
- Businesses connect their own WhatsApp Business number via **Embedded Signup** (OAuth widget embedded in onboarding)
- Inbound messages webhook to the Express webhook receiver, then forwarded to n8n
- All inbound webhooks verified with **HMAC-SHA256** (`X-Hub-Signature-256` header) in the Express service before forwarding. Invalid signatures are rejected with HTTP 403.
- Outbound messages sent via Meta Cloud API
- Conversation fees: ~$0.01–0.04/24h window (charged to platform, factored into subscription)
- **Template messages required** for all platform-initiated messages (reminders, confirmations sent outside a 24h customer-initiated window). Templates must be pre-approved by Meta before launch.
- **Opt-in required** before sending template messages (see Section 3.2 for opt-in flow).
- **Bot transparency:** The chatbot must identify itself as automated in the first message of every conversation, per Meta's Business Messaging Policy (e.g., *"Merhaba! Ben [İşletme Adı]'nın yapay zeka asistanıyım."*).

**Instagram — Meta Graph API**
- Businesses connect their Instagram Business account via Meta OAuth
- Single Facebook App handles all clients
- Requires Meta App Review before first client can connect (plan 1–4 weeks in timeline)
- **Note on Instagram messaging policy:** Meta's Standard Messaging policy restricts sending unsolicited messages to Instagram users. Verify current policy before relying on Instagram for proactive reminders; WhatsApp template messages are the safer primary reminder channel.

**WhatsApp Business Verification:**
Each business must have their Facebook Business Manager verified by Meta before connecting. Guide clients through this during onboarding; budget 1–3 days per client initially.

---

### 3.2 AI Chatbot — Claude API via n8n

**Conversation state machine:**

```
IDLE → GREETING → COLLECTING_SERVICE → COLLECTING_STAFF (if specific mode)
     → COLLECTING_DATETIME → CONFIRMING → BOOKED
     → CANCELLING → RESCHEDULING
     → FAQ
     → HANDOFF
```

State is stored as a `state` field on the `conversations` table. Claude receives the current state in its system prompt and transitions state based on its structured action responses.

**Session lifecycle:**
- A conversation session begins on the first message from a customer
- Sessions expire after **30 minutes of inactivity** — n8n resets state to `IDLE` via a scheduled check
- On session reset, a new greeting is sent if the customer messages again

**Per-message flow (n8n workflow):**

1. Webhook received → business identified (by phone number / Instagram account ID)
2. Check per-customer rate limit: max 20 messages per 5 minutes per customer. If exceeded, send *"Çok fazla mesaj gönderdiniz, lütfen biraz bekleyin."* and drop the message.
3. Fetch last 15 messages of this customer's conversation from PostgreSQL
4. Call Claude API (timeout: 60 seconds — configure n8n HTTP node accordingly) with:
   - **System prompt**: see Section 3.2a
   - **Conversation history**: last 15 messages
   - **Current state**: conversation state machine value
   - **Current message**: customer's new message
5. Parse Claude's response:
   - Attempt JSON parse. If valid JSON with an `action` field → execute action (Section 3.2b)
   - If plain text or unparseable → treat as text reply, send to customer
   - If JSON parse fails unexpectedly → log error, send fallback: *"Üzgünüm, bir sorun oluştu. Lütfen tekrar deneyin."*
6. Execute action via Cal.com API (if applicable)
7. Send reply to customer via appropriate channel
8. Save exchange + updated state to PostgreSQL

**3.2a — System Prompt Template (skeleton):**

```
Sen [İşletme Adı]'nın randevu asistanısın. Müşterilerle yalnızca Türkçe konuşursun.
Bu bir yapay zeka asistanı olduğunu ilk mesajda belirt.

İşletme bilgileri:
- Hizmetler: [Hizmet Adı] ([Süre] dk, [Fiyat] TL), ...
- Çalışma saatleri: [Gün]: [Saat aralığı], ...
- Personel: [Personel Adı] ([Uzmanlık]), ...
- Personel atama modu: [specific | auto-assign]

Yapabileceklerin: randevu al, iptal et, yeniden planla, SSS yanıtla, hatırlatıcı için onay al.
Yapamayacakların: ödeme işlemi, mesai dışı randevu.

Bir işlem gerektiğinde yanıtını şu formatta ver:
{"action": "book", "service": "...", "staff": "...", "datetime": "YYYY-MM-DDTHH:MM"}

Kullanılabilir aksiyonlar: check_availability, book, cancel, reschedule, handoff, text_reply
```

**3.2b — Action execution:**

| Action | n8n behaviour |
|---|---|
| `check_availability` | Call Cal.com API → return available slots → Claude presents them to customer |
| `book` | Call Cal.com API to create booking → confirm to customer → collect opt-in |
| `cancel` | Call Cal.com API to cancel booking → confirm to customer |
| `reschedule` | Cancel old booking + create new → confirm to customer |
| `handoff` | Notify business owner (WhatsApp/email) → inform customer |
| `text_reply` | Send `message` field directly to customer |

**Human handoff:** After 2–3 failed understanding attempts (tracked in conversation state), Claude returns `{"action": "handoff"}`. n8n notifies the business owner and informs the customer they'll be contacted by the team.

**Opt-in collection flow:**
After a successful booking, bot sends: *"Randevunuzdan önce hatırlatma mesajı almak ister misiniz? (Evet / Hayır)"*
- "Evet" → `whatsapp_opt_ins` record set to `opted_in: true`
- "Hayır" or no response → `opted_in: false`; no reminders sent
- Opt-in persists across future bookings for that customer at that business
- Re-consent requested once every 6 months

**Booking integrity (double-booking prevention):**
The check+book sequence is **not transactionally atomic** at the n8n layer — it is two sequential HTTP calls. Cal.com's database-level constraints handle actual conflict detection. If a booking fails with a conflict error after the customer confirms, n8n re-fetches availability and prompts the customer to pick again: *"Üzgünüm, bu saat dolu. Lütfen başka bir zaman seçin."*

**Staff assignment modes:**
- **Auto-assign:** Claude does not ask about staff. n8n passes `staff: "any"` to Cal.com, which assigns the next available member.
- **Specific:** Claude asks *"Hangi personelimizle randevu almak istersiniz? [liste]"* If the chosen staff is unavailable at the requested time, Claude offers alternative times for that staff member or suggests switching to auto-assign.

**Claude API spend control:**
- Per-business monthly token budget tracked in the `businesses` table (`monthly_token_count`, `token_limit`)
- n8n increments token count after each API call using Claude's usage response fields
- If a business exceeds their limit (default: 500,000 tokens/month), bot responds: *"Şu anda servisimiz geçici olarak kullanılamıyor."* and platform operator is alerted
- Default limit covers ~500–1,000 conversations/month (well above typical salon usage)

**Turkish character handling:** PostgreSQL database uses `UTF8` encoding. All API payloads (Meta, Cal.com, Claude) sent with `Content-Type: application/json; charset=utf-8`. n8n HTTP nodes configured to preserve encoding.

---

### 3.3 Scheduling Layer — Self-Hosted Cal.com

Cal.com runs on the Hetzner VPS. Each business gets their own **Organization** (Cal.com's multi-tenant structure).

**Per-business configuration:**
- Service catalog (service name, duration, price)
- Staff members with individual working hours and assigned services
- Staff assignment mode toggle: **specific** (customer picks staff) or **auto-assign** (next available)

**Business owner dashboard** (Cal.com admin panel):
- View all upcoming appointments (calendar view)
- Manually add appointments
- Edit appointment details
- Cancel appointments
- Manage staff and services

Cal.com handles buffer times, staff-specific availability, and concurrent booking conflicts natively at the database level.

---

### 3.4 Orchestration — n8n (Self-Hosted)

n8n runs on the same Hetzner VPS. Responsibilities:

- Receive routed webhooks from the Express webhook receiver
- Manage the message → Claude API → Cal.com API flow
- Run daily cron job for appointment reminders
- Handle retry logic for failed API calls (max 3 retries with exponential backoff)
- Store/retrieve conversation state from PostgreSQL
- Track per-business Claude API token usage

**n8n HTTP node configuration:**
- Default timeout increased to 60 seconds to accommodate Claude API response times on complex conversations
- All outbound API calls use credential-based auth (stored in n8n's encrypted credential store)

**Reminder cron job (runs daily at 09:00 Turkey time, UTC+3):**
1. Query Cal.com for appointments in the next 24 hours
2. For each appointment, check `whatsapp_opt_ins` for opt-in status
3. Check `reminders_sent` table to prevent duplicate reminders (idempotent)
4. Send reminder via the channel the customer originally used:
   - WhatsApp: pre-approved template message
   - Instagram: verify current Meta policy before enabling; fall back to WhatsApp if restricted
5. Record sent reminder in `reminders_sent` table

---

### 3.5 Infrastructure — Docker Compose on Hetzner

**Server:** Hetzner CX41 — 4 vCPU, 16GB RAM (~$20–24/month)

> Cal.com alone can consume 4–6 GB RAM under moderate load. CX31 (8GB) is insufficient once Cal.com, n8n, PostgreSQL, and Redis share the same server. CX41 provides safe headroom. Plan to upgrade to CX51 (~$38/month) at ~20 clients, or split Cal.com to its own server.

```yaml
services:
  calcom:        # Cal.com Next.js app (internal port 3000)
  postgres:      # Shared PostgreSQL (UTF8 encoding)
  redis:         # Required by Cal.com
  n8n:           # Workflow automation (internal port 5678)
  webhook:       # Express service: receives Meta webhooks, verifies HMAC, forwards to n8n
  nginx:         # Reverse proxy + SSL termination (Let's Encrypt)
```

All services on a private Docker network. Only `nginx` is exposed on ports 80/443. nginx routes:
- `/webhook/*` → webhook Express service
- `/cal/*` → Cal.com
- `/n8n/*` → n8n (restricted to platform admin IP)

**Backups:**
- Hetzner automated volume snapshots: daily, 7-day retention (~$2/month)
- PostgreSQL `pg_dump` to Hetzner Object Storage: nightly, 30-day retention
- n8n workflow exports backed up to Object Storage weekly

---

### 3.6 Business Onboarding Flow

**Platform auth:** Business owners authenticate to the platform using a lightweight custom auth layer (email + password, JWT tokens) built as a small Next.js app or used via Cal.com's built-in auth. This is separate from Cal.com's internal admin. A simple solution for MVP: use Cal.com's own account system — each business owner gets a Cal.com account under the platform's Organization, giving them access to their dashboard directly.

**Onboarding steps:**
1. Business lands on marketing/signup page
2. Creates account (email + password)
3. Selects plan, pays via **İyzico Subscription API** (recurring monthly charge — see Section 3.7)
4. Fills in business profile: name, services + durations + prices, staff names + hours, language preferences
5. Connects WhatsApp via Embedded Signup widget
6. Connects Instagram via Meta OAuth
7. Platform automatically provisions:
   - Cal.com Organization for the business
   - Staff members in Cal.com
   - Services in Cal.com
   - `businesses` record in PostgreSQL
   - System prompt generated from business profile (stored in `businesses.system_prompt`)
8. Bot goes live — test message sent to confirm

---

### 3.7 Billing — İyzico

**İyzico Subscription API** (not hosted checkout — hosted checkout does not support automatic recurring billing):
- Use İyzico's `subscription` product for automatic monthly charges
- Platform stores `iyzico_subscription_id` and `iyzico_customer_id` in `subscriptions` table
- İyzico sends webhook events to platform on: payment success, payment failure, cancellation
- Platform consumes these events to update `subscriptions.status`

**Subscription lifecycle:**
- Active → bot runs normally
- Failed payment → 3-day grace period → bot paused → business notified by email
- Cancelled → bot deactivated → business data retained for 90 days (KVKK compliance) → deleted after retention period
- KDV (Turkish VAT at 20%) handled automatically by İyzico

---

### 3.8 Platform Admin Dashboard

An internal dashboard (accessible only to platform operators, restricted by IP in nginx) for managing the platform. MVP requirements:

- List all businesses (name, status, subscription plan, last activity)
- Activate / deactivate a business manually
- View a business's recent conversation logs (for support)
- Trigger manual system prompt regeneration for a business
- View platform-wide metrics: total active clients, total bookings this month, Claude API spend this month
- Rotate API keys per business (Meta tokens, Cal.com credentials)

Implementation: a simple password-protected Next.js admin page or n8n dashboard with restricted access. Can be minimal for MVP — a few n8n workflows triggered manually are sufficient initially.

---

## 4. Data Model (Key Tables)

```sql
businesses
  id, name, whatsapp_number, instagram_account_id,
  calcom_org_id, system_prompt, staff_mode (specific|auto),
  monthly_token_count, token_limit, active, created_at

staff
  id, business_id, name, calcom_user_id, specialties, created_at

conversations
  id, business_id, customer_identifier, channel (whatsapp|instagram),
  state (IDLE|GREETING|COLLECTING_SERVICE|...|BOOKED|HANDOFF),
  last_activity_at, created_at

messages
  id, conversation_id, role (user|assistant), content, created_at

whatsapp_opt_ins
  id, business_id, customer_phone, opted_in, consented_at, expires_at

reminders_sent
  id, business_id, booking_id, customer_identifier, sent_at

subscriptions
  id, business_id, iyzico_subscription_id, iyzico_customer_id,
  plan, status (active|paused|cancelled), next_billing_date
```

---

## 5. Chatbot Capabilities (MVP)

| Capability | Supported |
|---|---|
| Book new appointment | ✓ |
| Cancel appointment | ✓ |
| Reschedule appointment | ✓ |
| Answer FAQs (services, prices, hours, location) | ✓ |
| Send appointment reminders (with opt-in) | ✓ |
| Collect reminder opt-in | ✓ |
| Auto-assign or specific staff selection | ✓ (per business config) |
| Human handoff on failure | ✓ |
| Bot self-identification as AI | ✓ (first message always) |
| Payment processing | ✗ (in person only) |

---

## 6. Cost Model

| Cost | Amount |
|---|---|
| Hetzner VPS (CX41) | ~$22/month |
| Hetzner backups + object storage | ~$3/month |
| Meta conversation fees | ~$6–24/client/month |
| Claude API | ~$5–15/client/month |
| Domain + SSL (Let's Encrypt) | ~$1/month |
| **Total at 10 clients** | **~$140–290/month** |
| **Revenue at 10 clients (5,000 TRY)** | **~$1,130/month** |
| **Gross margin** | **~74–88%** |

---

## 7. Compliance

**KVKK (Turkish Data Protection Law):**
- Platform must register as a data controller with KVKK authority
- Privacy policy required, displayed before onboarding and booking
- Customer data (phone numbers, conversation history, appointment data) retained for maximum 2 years or until customer deletion request
- Cancelled business data retained 90 days then purged
- Consent for data processing collected at first chatbot interaction

**Meta Policy:**
- All bots must identify as automated (handled in system prompt — first message always states this)
- No deceptive automation; bot must offer human handoff path
- WhatsApp opt-in for proactive messages is mandatory (handled in booking confirmation flow)

---

## 8. Key Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Cal.com self-hosted Organizations not fully functional | Deploy test instance and verify before building onboarding; have Cal.com cloud as fallback ($37/user/month) |
| Meta App Review delays (1–4 weeks) | Submit early; start WhatsApp-only if Instagram review delays |
| WhatsApp Business verification per client (1–3 days) | Guided checklist in onboarding flow |
| Template message approval (days–weeks) | Draft and submit 3–5 core templates before launch |
| Double booking conflict | Cal.com handles DB-level conflicts; n8n re-prompts customer on conflict error |
| VPS overload | CX41 conservatively handles ~20 clients; upgrade to CX51 or split Cal.com at that point |
| İyzico webhook missed (billing state drift) | Reconciliation job: nightly compare İyzico subscription status vs local DB |
| Claude API cost spike (abusive customer) | Per-business token budget with hard cap; alert on >80% usage |
| Instagram DM policy restriction on reminders | WhatsApp is primary reminder channel; Instagram reminders optional/verified |
| KVKK non-compliance | Register data controller, add consent flows before first paying client |
| VPS disk failure (data loss) | Daily Hetzner snapshots + nightly pg_dump to Object Storage |
| Meta API version deprecation (Embedded Signup) | Monitor Meta changelog; plan annual review of API version compatibility |

---

## 9. Build Sequence (Recommended)

1. **Infrastructure**: Hetzner CX41, Docker Compose with Cal.com + n8n + PostgreSQL + Redis + nginx
2. **Validate Cal.com Organizations** in self-hosted mode before proceeding
3. **Meta setup**: Register Facebook App, configure WhatsApp Cloud API + Embedded Signup, Instagram Graph API, submit App Review
4. **Prepare WhatsApp templates**: Draft and submit reminder/confirmation/opt-in templates to Meta
5. **Webhook receiver**: Express service with HMAC verification, routing to n8n
6. **n8n flows**: Message routing, Claude API integration with output validation, Cal.com API calls, conversation state management, reminder cron, token tracking
7. **Onboarding flow**: Signup/auth, business profile form, İyzico Subscription API integration, Embedded Signup (WhatsApp), Meta OAuth (Instagram), Cal.com Organization provisioning
8. **Platform admin dashboard**: Basic tenant management, logs, metrics
9. **Test end-to-end**: Book → confirm → opt-in → remind → cancel flow on both channels
10. **KVKK registration + privacy policy**
11. **Launch with first client**
