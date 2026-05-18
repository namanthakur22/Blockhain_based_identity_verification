# Product Requirements Document (PRD): QuickBotify (Telegram Bot Platform + Langflow Workflows)

## 1. Executive Summary

**Problem Statement**
QuickBotify users want to deploy feature-rich Telegram bots (reply/forward/moderation + AI workflows etc) without building and hosting custom infrastructure, but they still need reliable webhook processing, accurate credit billing, and secure token handling.

**Proposed Solution**
Provide a web-based platform that lets users create and manage Telegram bot “instances”, configure modular bot capabilities (Reply Bot, Forwarder Bot, Group Moderation Bot, and Langflow-powered AI workflows), and pay via a credit system backed by Razorpay.

**Success Criteria**
- **Webhook reliability**: >= 99.9% of valid Telegram updates return HTTP 200 within 2 seconds; >= 99.9% of updates are processed without unhandled server errors.
- **Credit deduction accuracy**: 0 known double-charges per `update_id` per `tg_bot_id`; daily reconciliation finds < 0.1% mismatch between (starting balance + credits added − credits consumed) and stored balance.
- **Payment processing accuracy**: 0 credits granted without a verified Razorpay signature; 100% of verified payments are credited exactly once (idempotent completion).
- **Security baseline**: no plaintext Telegram bot tokens stored; webhook endpoint rejects invalid secret header with 401; admin-only operational endpoints require superuser auth.
- **Extensibility**: a new Telegram bot module can be added via a registry (no edits to core webhook handler) with <= 1 day engineering effort including unit tests and credit mapping.

---

## 2. User Experience & Functionality

### User Personas
| Persona | Description | Primary Goals |
|---|---|---|
| Bot Owner | Purchases/configures bots for their Telegram communities | Fast setup, predictable billing, reliable behavior |
| Community Moderator | Uses moderation tooling via the bot | Reduce spam, enforce rules, ban/warn reliably |
| Forwarding Admin | Runs forwarding/anonymous forwarding workflows | Safe forwarding, anonymity guarantees |
| AI Workflow Builder | Uses Langflow workflows to power an AI bot | Build and iterate workflows, verify outputs |
| Platform Admin | Operates QuickBotify (support + abuse prevention) | Monitor health, manage products, handle incidents |

### User Stories + Acceptance Criteria

**Story 1 — Sign up and initialize credits**
- As a **Bot Owner**, I want to create an account and immediately receive monthly free credits so that I can try the platform.
- **Acceptance Criteria**
  - On first account creation, a credit balance is created automatically.
  - A `monthly_free` credit transaction is recorded for the initial allocation.
  - The initial allocation and monthly free-credit cap is configurable via `MONTHLY_FREE_CREDITS` (default: 500).
  - On day 1 of each month, the system grants a monthly top-up exactly once per user: `topup = min(last_month_processed_consumption, MONTHLY_FREE_CREDITS)` and records a `monthly_free` transaction.
  - Credits roll over month-to-month (the monthly job adds credits; it does not reset the balance).

**Story 2 — Install and activate a Telegram bot instance**
- As a **Bot Owner**, I want to connect my Telegram bot token so that QuickBotify can receive webhooks and operate my bot.
- **Acceptance Criteria**
  - User can save a Telegram bot token; platform validates it via Telegram `getMe`.
  - Token is stored encrypted at rest; a non-reversible hash is stored for dedupe checks.
  - When a bot is activated, the platform sets the Telegram webhook to `BASE_URL/telegram_webhook/<tg_bot_id>` and configures Telegram’s `secret_token`.
  - When deactivating the last active module using a token, the webhook is removed.

**Story 3 — Attach modular bot capabilities to one Telegram bot**
- As a **Bot Owner**, I want to enable multiple modules (Reply Bot, Forwarder Bot, Group Moderation Bot, AI Workflow) on the same Telegram bot so that one bot can do multiple jobs.
- **Acceptance Criteria**
  - A single Telegram bot token (`tg_bot_id`) can have multiple active modules attached.
  - A single incoming Telegram update can be processed by multiple modules deterministically.
  - Credit consumption is computed per module activity and then finalized once per update.
  - If the user’s available credits are 0, the system must not execute chargeable module actions and must send a low-credit notification/message.

**Story 4 — Configure Reply Bot behavior**
- As a **Bot Owner**, I want to configure condition-based replies so that my bot auto-responds to common messages/commands.
- **Acceptance Criteria**
  - Supports condition matching types: `equals`, `contains`, `starts_with`, `ends_with`, `does_not_contain`.
  - Config changes take effect within 60 seconds (cache invalidation or immediate reload).
  - Reply actions are logged to an activity timeline.

**Story 5 — Configure Forwarder Bot behavior (including anonymous forwarding)**
- As a **Forwarding Admin**, I want to forward messages from one group/channel to others with optional anonymity so that users can safely post.
- **Acceptance Criteria**
  - Supports filtering by text pattern + match type and sender allowlist.
  - Supports multi-destination forwarding.
  - Anonymous mode must not expose the original sender identity in forwarded messages.
  - Forward actions are logged; failures include an error reason.

**Story 6 — Configure Group Moderation rules**
- As a **Community Moderator**, I want to automatically warn/ban users and enforce URL/badword rules so that spam is reduced.
- **Acceptance Criteria**
  - Warn count is tracked per user; on threshold, user is banned.
  - Messages matching delete rules can be removed.
  - Allowed URL list is respected.
  - Approved admins list prevents enforcement actions against privileged users.

**Story 7 — Build and run AI workflows (Langflow)**
- As an **AI Workflow Builder**, I want to view/manage the workflow linked to my bot so that I can ship AI-driven experiences.
- **Acceptance Criteria**
  - Platform can link a bot to a Langflow flow (`langflow_flow_id`) and mark it as template-based or custom.
  - Workflow viewer loads and renders a display-friendly component list.
  - Incoming Telegram updates for AI-enabled bots execute the flow and send text responses back to Telegram.
  - If credits are insufficient, the user receives a clear “insufficient credits” message in Telegram and no workflow call is made.
  - Each workflow execution consumes credits based on a configurable rule (default: 1 credit per workflow execution) and is recorded in the consumption ledger.

**Story 8 — Purchase credits via Razorpay and view usage analytics**
- As a **Bot Owner**, I want to buy credits and see my usage so that I can control spend.
- **Acceptance Criteria**
  - Creating an order returns a Razorpay order id; credits are only finalized after signature verification.
  - Payment completion is idempotent: repeated callbacks do not double-credit.
  - Checkout uses `USD` and collects `amount_usd` as an integer major-unit amount; the backend creates the Razorpay order using `amount_subunits = amount_usd * 100`.
  - Credits granted are computed from `amount_usd` using: `credits = amount_usd * (UNIT_CREDITS_COUNT // CREDITS_UNIT_CHARGE)` (default: 1 USD → 500 credits).
  - User can view current credit balance and the last 10 credit transactions on the profile page.
  - User can view a usage dashboard (last N days, max 90) broken down by bot.

### Non-Goals
- Multi-platform bot support (WhatsApp/Discord/etc.) in the initial release.
- Payment providers other than Razorpay for credit purchase.
- Building a full in-browser Langflow editor; QuickBotify integrates with Langflow and provides a viewer + linking, not an IDE replacement.
- Advanced marketplace features for third-party bot developers (publish/reviews/revenue share) unless explicitly added later.

---

## 3. AI System Requirements (Langflow)

### Tool Requirements
- **Langflow API**
  - Must support: execute flow by `flow_id`, fetch flow details, and return structured outputs.
  - Auth: API key-based auth for server-to-server calls.
- **Telegram Bot API**
  - Must support sending messages, editing messages (optional), moderation actions (delete/ban).

### Evaluation Strategy
- **Offline evaluation (pre-release)**
  - Create a test suite of at least 50 representative Telegram inputs (support, moderation queries, FAQ-style queries).
  - Pass criteria: >= 90% of outputs match expected intent and do not violate safety rules.
- **Online evaluation (post-release)**
  - Track: workflow error rate, median workflow latency, and user “thumbs up/down” feedback (TBD if exposed in UI).
  - Safety: blocklist/guardrails for toxic/harassing outputs; log and review flagged outputs.

---

## 4. Technical Specifications

### Architecture Overview
**Core components (recommended baseline, aligned with current system)**
- Django web app (server-rendered pages + small JSON endpoints for AJAX)
- PostgreSQL (source of truth)
- Redis (cache + Celery broker)
- Celery (scheduled tasks + async processing)
- Telegram webhook ingestion endpoint
- Langflow service (local Docker or cloud)
- Razorpay payments

**Data flow (Telegram update)**
1. Telegram POSTs update to `POST /telegram_webhook/<tg_bot_id>` with `X-Telegram-Bot-Api-Secret-Token`.
2. Server validates secret header; rejects invalid with 401.
3. Server enforces idempotency: `(update_id, tg_bot_id)` processed at most once.
4. Server loads all active modules attached to that `tg_bot_id` and executes them.
5. Server records actions + activity logs; computes credit consumption; finalizes deduction atomically.
6. Server returns HTTP 200.

### Integration Points
| System | Purpose | Notes |
|---|---|---|
| Telegram Bot API | Webhooks + actions (send/delete/ban/etc.) | Webhook must be set with secret token |
| Razorpay | Credit purchase checkout | Signature verification required |
| Langflow | AI workflow execution + flow detail fetch | Feature-flagged by `LANGFLOW_ENABLED` |
| PostgreSQL | Persistence (bots, settings, credits, logs) | Must support transactional updates |
| Redis | Caching + broker | Reply condition mapping + quiz cache patterns |
| Celery Beat | Monthly free credit top-up | Runs on day 1 of month |
| Optional S3 | Media storage | Enabled via `USE_S3` |

### Data Model (Minimum Required)
| Entity | Key Fields | Notes |
|---|---|---|
| Bot Instance | `user_id`, `tg_bot_id`, `token_encrypted`, `token_hash`, `is_active` | Token encrypted at rest |
| Module Subscription | `bot_instance_id`, `product/module_type`, module-specific settings | Allows multiple modules per bot |
| Module Settings | Condition rules / moderation rules / forwarding rules | Typed values; validated |
| CreditBalance | `user_id`, `total_credits` | Updated atomically |
| CreditTransaction | `user_id`, `credit_type`, `amount`, `reference_id`, vendor ids | Ledger of adds |
| CreditConsumption | `bot_instance_id`, `credits_used`, `is_processed` | Ledger of spends |
| CreditMapping | `module_type`, `activity`, `base_credits`, limit rules | Supports base + extra credits |
| UpdateLog | `tg_bot_id`, `update_id` | Unique together for idempotency |
| MonitoringEvent | status/action/category/message | For stats + ops |

### Security & Privacy
- **Token security**: encrypt Telegram tokens at rest; never log tokens; store `token_hash` for dedupe.
- **Webhook authentication**: validate `X-Telegram-Bot-Api-Secret-Token` against server secret.
- **Idempotency**: enforce unique `(update_id, tg_bot_id)` processing.
- **Authorization**: bot settings and workflow views require authenticated ownership checks.
- **Payments**: verify Razorpay signature server-side; idempotent completion keyed by order/payment ids.
- **Rate limiting & abuse** (planned improvement): apply basic per-bot and per-IP limits to webhook endpoint.
- **Data retention (TBD)**: define retention windows for update logs, monitoring events, and activity logs.

---

## 5. Risks & Roadmap

### Phased Rollout
- **MVP (v1.0)**
  - Telegram-only platform
  - Modules: Reply Bot, Forwarder Bot (incl. anonymous), Group Moderation Bot
  - Credits: balance, mapping, consumption, monthly free credits, Razorpay purchase
  - Observability: activity logs + basic stats
  - Langflow: workflow linking + viewer + execution for AI-enabled bots
- **v1.1 (Reliability + Extensibility)**
  - Queue-based webhook processing (ack fast; process async) to hit 99.9% reliability target
  - Bot module registry + clear developer contract for adding new bot types
  - Stronger idempotency and reconciliation jobs (credits + payments)
- **v2.0 (Growth)**
  - Additional messaging platforms (WhatsApp/Discord) if required
  - Marketplace expansion (TBD)

### Technical Risks
- **Webhook latency/timeouts**: synchronous processing can exceed Telegram timeouts; mitigate with async queues and safe retries.
- **Multi-module routing ambiguity**: must define deterministic ordering and conflict resolution when multiple modules act on the same message.
- **Credits correctness under concurrency**: requires DB transactions + idempotency to avoid double spend.
- **Langflow dependency**: workflow downtime or schema changes can break bots; require retries, circuit breakers, and fallback messaging.
- **Moderation safety**: false positives in badword/URL rules can disrupt communities; require audit logs and easy rollback of settings.
