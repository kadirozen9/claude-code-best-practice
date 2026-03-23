# Appointment Chatbot SaaS — Design Spec

**Date:** 2026-03-23
**Status:** Approved
**Authors:** User + Claude

---

## 1. Product Overview

A SaaS platform that gives service businesses (salons, clinics, spas) an AI chatbot that handles customer conversations on WhatsApp and Instagram — booking appointments, answering FAQs, sending reminders, and managing cancellations — 24/7 without human involvement.

Business owners access a white-labeled Cal.com dashboard to view, manually add, edit, and cancel appointments.

**Pricing:** 5,000–10,000 TRY/month per business (~$113–226 USD at March 2026 rates)
**Target market:** Beauty salons, clinics, spas — Turkey-focused initially

---

## 2. Architecture Overview

```
Customer (WhatsApp / Instagram)
        ↓
[Messaging Layer]
  - WhatsApp: Meta Cloud API (Embedded Signup)
  - Instagram: Meta Graph API
        ↓ webhook
[Orchestration Layer] — n8n (self-hosted)
        ↓
[AI Chatbot] — Claude API
  - System prompt per business (services, staff, hours)
  - Conversation state in PostgreSQL
        ↓ structured actions
[Scheduling Layer] — Cal.com (self-hosted)
  - Per-business organization/team
  - Business owner dashboard
        ↑
[Billing] — İyzico
  - Monthly subscription per business
```

**Infrastructure:** Single Hetzner CX31 VPS (~$12–16/month), Docker Compose
**Stack:** Cal.com, n8n, PostgreSQL, Redis, Meta APIs, Claude API, İyzico

---

## 3. System Components

### 3.1 Messaging Layer

**WhatsApp — Meta Cloud API**
- Platform registers one Meta Business App
- Businesses connect their own WhatsApp Business number via **Embedded Signup** (OAuth widget embedded in onboarding)
- Inbound messages webhook to n8n
- Outbound messages sent via Meta Cloud API
- Conversation fees: ~$0.01–0.04/24h window (charged to platform, factored into subscription)
- **Template messages required** for all platform-initiated messages (reminders, confirmations sent outside a 24h customer-initiated window). Templates must be pre-approved by Meta before launch.
- **Opt-in required** before sending template messages. Bot collects consent at end of booking flow.

**Instagram — Meta Graph API**
- Businesses connect their Instagram Business account via Meta OAuth
- Single Facebook App handles all clients
- No conversation window restrictions — reminders can be sent freely
- Requires Meta App Review before first client can connect (plan 1–4 weeks in timeline)

**WhatsApp Business Verification:**
Each business must have their Facebook Business Manager verified by Meta before connecting. Guide clients through this during onboarding; budget 1–3 days per client initially.

---

### 3.2 AI Chatbot — Claude API via n8n

Each inbound message triggers this n8n flow:

1. Identify business (by receiving phone number or Instagram account ID)
2. Fetch last 10 messages of this customer's conversation from PostgreSQL
3. Call Claude API with:
   - **System prompt**: business name, services + durations, staff list, working hours, staff assignment mode (specific or auto-assign), booking rules
   - **Conversation history**: last 10 messages
   - **Current message**: customer's new message
4. Claude returns either:
   - A **text reply** (FAQ answer, confirmation message, clarifying question)
   - A **structured action**: `{"action": "check_availability", "service": "...", "staff": "...", "date": "..."}`
     or `{"action": "book", "service": "...", "staff": "...", "datetime": "..."}`
     or `{"action": "cancel", "booking_id": "..."}`
     or `{"action": "reschedule", "booking_id": "...", "new_datetime": "..."}`
     or `{"action": "handoff"}` (escalate to human)
5. n8n executes the action via Cal.com API
6. n8n sends reply to customer via appropriate channel
7. n8n saves exchange to PostgreSQL conversation table

**Human handoff:** After 2–3 failed understanding attempts, Claude returns `{"action": "handoff"}`. n8n notifies the business owner via WhatsApp or email and informs the customer they'll be contacted by the team.

**Business hours:** System prompt includes operating hours. Claude will not offer slots outside these hours and responds appropriately when the business is closed.

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

**Availability and booking integrity:**
- Availability check and booking creation treated as atomic: n8n checks availability, presents options to customer, then immediately books on customer confirmation — no delay between check and book to prevent double booking.
- Cal.com handles buffer times, staff-specific availability, and concurrent booking conflicts natively.

---

### 3.4 Orchestration — n8n (Self-Hosted)

n8n runs on the same Hetzner VPS. Responsibilities:

- Receive and route inbound webhooks (WhatsApp + Instagram)
- Manage the message → Claude API → Cal.com API flow
- Run daily cron job for appointment reminders
- Handle retry logic for failed API calls
- Store/retrieve conversation state from PostgreSQL

**Reminder cron job (daily):**
1. Query Cal.com for appointments in the next 24 hours
2. For each appointment with customer opt-in:
   - WhatsApp: send pre-approved template message
   - Instagram: send free-form reminder message
3. Customer can reply to cancel/reschedule — bot handles the response

---

### 3.5 Infrastructure — Docker Compose on Hetzner

**Server:** Hetzner CX31 — 4 vCPU, 8GB RAM (~$12–16/month)
Handles up to ~50 clients before needing a VPS upgrade.

```yaml
services:
  calcom:        # Cal.com Next.js app
  postgres:      # Shared PostgreSQL database
  redis:         # Required by Cal.com
  n8n:           # Workflow automation
  webhook:       # Lightweight service receiving Meta webhooks → n8n
```

All services communicate on a private Docker network. Only the webhook service and Cal.com are exposed publicly (behind nginx + SSL).

---

### 3.6 Business Onboarding Flow

1. Business discovers platform, lands on marketing page
2. Selects plan, pays via **İyzico** (Turkish payment gateway, handles KDV automatically)
3. Fills in business profile: name, services + durations, staff, working hours
4. Connects WhatsApp via Embedded Signup widget
5. Connects Instagram via Meta OAuth
6. Platform provisions:
   - Cal.com Organization for the business
   - Conversation history table in PostgreSQL (scoped to business)
   - System prompt generated from business profile
7. Bot goes live — confirmed via test message

---

### 3.7 Billing — İyzico

- Monthly recurring subscription at 5,000–10,000 TRY/month
- İyzico handles KDV (Turkish VAT) automatically
- Business owner manages their subscription from a simple billing page
- Failed payment → grace period (3 days) → bot paused → business notified

---

## 4. Data Model (Key Tables)

```sql
businesses          -- id, name, whatsapp_number, instagram_account_id, calcom_org_id, system_prompt, staff_mode, active
conversations       -- id, business_id, customer_identifier, channel (whatsapp|instagram), created_at
messages            -- id, conversation_id, role (user|assistant), content, created_at
whatsapp_opt_ins    -- id, business_id, customer_phone, opted_in, created_at
subscriptions       -- id, business_id, iyzico_subscription_id, plan, status, next_billing_date
```

---

## 5. Chatbot Capabilities (MVP)

| Capability | Supported |
|---|---|
| Book new appointment | ✓ |
| Cancel appointment | ✓ |
| Reschedule appointment | ✓ |
| Answer FAQs (services, prices, hours, location) | ✓ |
| Send appointment reminders | ✓ |
| Collect reminder opt-in | ✓ |
| Auto-assign or specific staff selection | ✓ (per business config) |
| Human handoff on failure | ✓ |
| Payment processing | ✗ (in person only) |

---

## 6. Cost Model

| Cost | Amount |
|---|---|
| Hetzner VPS (CX31) | ~$14/month |
| Meta conversation fees | ~$6–24/client/month |
| Claude API | ~$5–15/client/month |
| Domain + SSL | ~$1/month |
| **Total at 10 clients** | **~$130–280/month** |
| **Revenue at 10 clients (5,000 TRY)** | **~$1,130/month** |
| **Gross margin** | **~75–88%** |

---

## 7. Key Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Meta App Review delays (1–4 weeks) | Submit early; start with WhatsApp-only if needed |
| WhatsApp Business verification per client (1–3 days) | Build a guided checklist into onboarding |
| Template message approval (days–weeks) | Prepare and submit templates before launch |
| Double booking race condition | Atomic check+book in n8n; Cal.com handles DB-level conflicts |
| VPS overload at scale | Upgrade to CX41 (~$24/month) at ~50 clients; split services at ~100 |
| İyzico integration complexity | Use their hosted checkout page for MVP; custom integration later |

---

## 8. Build Sequence (Recommended)

1. **Infrastructure**: Hetzner VPS, Docker Compose, Cal.com self-hosted, n8n, PostgreSQL
2. **Meta setup**: Register Facebook App, configure WhatsApp Cloud API, Instagram Graph API, submit App Review
3. **Prepare WhatsApp templates**: Draft and submit reminder/confirmation templates to Meta
4. **n8n flows**: Webhook routing, Claude API integration, Cal.com API calls, reminder cron
5. **Onboarding flow**: Business profile form, Embedded Signup (WhatsApp), Meta OAuth (Instagram), İyzico billing
6. **Test end-to-end**: Book → confirm → remind → cancel flow on both channels
7. **Launch with first client**
