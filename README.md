# SaasCRM — AI-Assisted Lead Automation Platform

**A production-grade multi-tenant SaaS backend + frontend built from scratch**

---

## What This Is

SaasCRM is a full-stack SaaS platform that helps small and medium businesses automate lead capture, manage customer conversations across WhatsApp, Messenger, and Telegram, and use AI to draft replies — all from a single unified inbox.

Built as a solo end-to-end project covering system design, backend engineering, database architecture, AI integration, and frontend development.

---

## Live Architecture Overview

```
Customer (WhatsApp / Messenger / Telegram)
         │
         ▼
   Meta Cloud API
         │
         ▼
   NGINX (reverse proxy, SSL, rate limiting)
         │
         ▼
   Spring Boot Backend (port 8084)
    ├── JWT Auth Filter
    ├── Webhook Controllers (signature verified)
    ├── Lead Ingestion Service (deduplication)
    ├── Conversation State Machine
    ├── Transactional Outbox (PostgreSQL LISTEN/NOTIFY)
    ├── AI Pipeline (Groq / Gemini / OpenAI / Anthropic)
    ├── Confidence Gate (auto-send or human review)
    ├── SSE Realtime (inbox updates)
    └── Scheduler Jobs (timeouts, quota reset, retention)
         │
         ▼
   PostgreSQL 17
    ├── 16 Flyway migrations
    ├── Multi-tenant row-level isolation
    └── Outbox trigger (NOTIFY on insert)
         │
         ▼
   Next.js 15 Frontend (port 3000)
    ├── TanStack Query (server state + caching)
    ├── Zustand (client state)
    ├── SSE connection (live inbox updates)
    └── Role-based UI (Owner / Admin / Agent / Viewer)
```

---

## Tech Stack

### Backend
| Layer | Technology |
|---|---|
| Language | Java 21 (Virtual Threads) |
| Framework | Spring Boot 3.2 |
| Database | PostgreSQL 17 |
| ORM | Hibernate 6 + Spring Data JPA |
| Migrations | Flyway |
| Auth | JWT (JJWT) + BCrypt |
| HTTP Client | Spring RestClient |
| Connection Pool | HikariCP |
| Scheduler Lock | ShedLock (PostgreSQL backend) |
| Build | Maven |

### Frontend
| Layer | Technology |
|---|---|
| Framework | Next.js 15 (App Router) |
| Language | TypeScript |
| UI | Tailwind CSS + shadcn/ui + Radix UI |
| Server State | TanStack Query v5 |
| Client State | Zustand |
| Forms | React Hook Form + Zod |
| Charts | Recharts |
| Animations | Framer Motion |
| Drag and Drop | dnd-kit |
| HTTP | Axios |

### AI Providers
- Groq (llama-3.3-70b-versatile) — primary, ultra-fast inference
- Google Gemini (gemini-2.0-flash) — secondary
- OpenAI (gpt-4o-mini) — tertiary
---

## Key Engineering Decisions

### 1. Transactional Outbox Pattern

Every domain event (lead created, message received, AI reply ready) is written to an `outbox_events` table in the same database transaction as the originating entity change. This guarantees zero event loss even if the process crashes.

Two processing paths run in parallel:
- **PostgreSQL LISTEN/NOTIFY** triggers immediate processing (sub-second latency)
- **30-second safety poller** catches any missed notifications

Both use `SELECT FOR UPDATE SKIP LOCKED` so concurrent instances never process the same event twice.

### 2. Conversation State Machine with Optimistic Locking

Conversations follow a strict state machine:

```
AI_ACTIVE → HUMAN_ACTIVE → HUMAN_PAUSED_AI
          ↘              ↘
           AI_WAITING     CLOSED
```

Every state transition uses JPA `@Version` optimistic locking with automatic retry (up to 3 attempts). This prevents race conditions when a human takeover and an AI reply arrive simultaneously.

### 3. AI Confidence Gate

Every AI-generated response carries:
- `risk_level` (LOW / MEDIUM / HIGH / CRITICAL)
- `moderation_status` (PASSED / FLAGGED / BLOCKED)

The tenant-configurable confidence threshold decides whether the reply auto-sends or goes to a human review queue. CRITICAL and HIGH risk responses are never auto-sent regardless of confidence score. Responses in the review queue expire after 30 minutes to prevent sending stale context.

### 4. Webhook Idempotency

Meta's Cloud API retries failed webhooks automatically. Every inbound message carries a `provider_message_id` which is stored with a unique constraint:

```sql
UNIQUE(provider_name, provider_message_id)
```

Duplicate webhook deliveries are silently dropped before any processing begins — no duplicate leads, conversations, or AI calls.

### 5. Multi-Tenant Isolation

Every entity that holds tenant data extends `BaseTenantEntity` which enforces a `tenant_id` column that is set on insert and never updated. Every repository query is tenant-scoped. A missing `tenant_id` filter is treated as a critical security bug and caught by ArchUnit tests at compile time.

### 6. Refresh Token Rotation with Replay Detection

Refresh tokens are stored as SHA-256 hashes, never plaintext. On every use, the old token is revoked and a new one is issued (rotation). If a revoked token is presented (replay attack), the entire token family is revoked — forcing re-login on all devices.

### 7. Virtual Threads

`spring.threads.virtual.enabled=true` enables Java 21 virtual threads for all request handling. This allows the application to handle thousands of concurrent SSE connections and blocking I/O operations without thread pool exhaustion, on a single VPS.

---

## Database Schema (16 Migrations)

```
tenants                 → workspace root
users                   → tenant-scoped accounts
refresh_tokens          → hashed, family-based rotation
leads                   → contacts with channel identifiers
conversations           → state machine with optimistic lock version
messages                → unified inbox with provider dedup
outbox_events           → transactional event bus
ai_responses            → confidence + risk metadata, review queue
follow_ups              → scheduled automation
billing_subscriptions   → plan limits per tenant
tenant_feature_flags    → per-tenant feature overrides
provider_credentials    → encrypted channel config
analytics_events        → pre-bucketed for fast aggregation
shedlock                → distributed scheduler deduplication
+ triggers              → NOTIFY on outbox insert, updated_at maintenance
+ indexes               → 20+ covering inbox, scheduler, analytics queries
```

---

## AI Pipeline Flow

```
1. WhatsApp message arrives at webhook
2. HMAC-SHA256 signature verified
3. provider_message_id checked for duplicates
4. Lead resolved or created (5-level deduplication)
5. Conversation resolved or created (state checked)
6. Message persisted
7. AI_REPLY_REQUESTED outbox event written (same transaction)
8. Outbox poller picks up event
9. Conversation context loaded (rolling 10-message window)
10. AI provider called (NO active database transaction during call)
11. Response confidence scored + risk assessed
12. If confidence >= threshold AND risk is LOW/MEDIUM:
        → auto-send (write to outbox → WhatsApp API call)
13. If below threshold OR high risk:
        → queue for human review (SSE notification to agents)
14. Token usage tracked against tenant monthly quota
15. Analytics event emitted
```

---

## Multi-Provider AI Fallback

The active AI provider is selected at startup via `app.ai.provider`. Switching providers requires only a config change and restart — no code changes. Each provider implementation is isolated behind the `AiProvider` interface with its own confidence heuristic and risk assessment logic.

```
app.ai.provider=groq      # current (fastest)
app.ai.provider=gemini    # switch to Gemini
app.ai.provider=openai    # switch to OpenAI
```

---

## Realtime Architecture

Agents receive live inbox updates without polling via Server-Sent Events:

```
Backend event occurs (message, state change, AI review)
        ↓
OutboxEventRouter handles event
        ↓
SseEmitterRegistry.broadcastToTenant()
        ↓
All connected agents in that tenant receive SSE event
        ↓
TanStack Query cache invalidated → UI updates immediately
```

SSE connection per user, stored in a `ConcurrentHashMap`. Reconnects automatically with exponential backoff on disconnect. Designed for under 200 concurrent users per instance (architecture Section 15). WebSocket migration path is documented for when scale requires it.

---

## Role-Based Access Control

| Permission | Owner | Admin | Agent | Viewer |
|---|---|---|---|---|
| Inbox + messaging | ✓ | ✓ | ✓ | — |
| Lead management | ✓ | ✓ | ✓ | read |
| AI review queue | ✓ | ✓ | ✓ | — |
| Delete leads | ✓ | ✓ | — | — |
| Team management | ✓ | ✓ | — | — |
| Analytics | ✓ | ✓ | — | ✓ |
| Billing | ✓ | — | — | — |
| Channel settings | ✓ | — | — | — |

Enforced at three layers: JWT claims, `@PreAuthorize` on controllers, and `PlanFeatureGuard` AOP aspect for billing-gated features.

---

## Billing Plans

| Feature | Starter | Growth | Enterprise |
|---|---|---|---|
| WhatsApp | ✓ | ✓ | ✓ |
| Messenger + Telegram | — | ✓ | ✓ |
| Follow-up automation | — | ✓ | ✓ |
| Advanced analytics | — | — | ✓ |
| Facebook Lead Ads | — | — | ✓ |
| Custom AI threshold | — | — | ✓ |
| Data retention | 90 days | 1 year | Custom |
| Leads | 500 | Unlimited | Unlimited |
| Users | 3 | 10 | Unlimited |
| Monthly AI tokens | 500K | 2M | Custom |

Feature gates are enforced by a Spring AOP aspect (`PlanFeatureGuard`) using `@PlanRequired(Feature.X)` annotations on service methods.

---

## Scheduler Jobs

| Job | Schedule | Purpose |
|---|---|---|
| ConversationTimeoutScheduler | Every hour | Return stale HUMAN_ACTIVE conversations to AI_ACTIVE |
| MonthlyQuotaResetScheduler | 1st of month, midnight UTC | Reset AI token counters |
| FollowUpScheduler | Every 60 seconds | Dispatch due follow-ups via outbox |
| AiResponseExpiryScheduler | Every 10 minutes | Expire stale AI review queue items |
| DataRetentionScheduler | 2:00 AM UTC daily | Clean processed outbox events and expired tokens |
| OutboxPoller | Every 30 seconds | Safety-net for missed NOTIFY signals |

All jobs use ShedLock with PostgreSQL backend — safe to run on multiple instances without double-execution.

---

## Security Measures

- HMAC-SHA256 webhook signature verification on all inbound webhooks
- Constant-time signature comparison (prevents timing oracle attacks)
- BCrypt password hashing (strength 12, ~300ms per hash)
- JWT access tokens (15 min) + rotating refresh tokens (7 days)
- Refresh token replay attack detection with family revocation
- Tenant isolation enforced at repository layer + ArchUnit tests
- IP-based rate limiting with Bucket4j (10/min on auth endpoints)
- NGINX outer rate limiting layer
- Actuator endpoints blocked from public access via NGINX
- No stack traces in API error responses
- Audit log for all sensitive actions (immutable, REQUIRES_NEW transaction)

---

## ArchUnit Enforcement Tests

Architecture rules enforced at test time, not runtime:

- Controllers must not access repositories directly
- All audit writes must go through `AuditService` only
- All outbox events must be published via `OutboxEventService` only
- All tenant-scoped entities must extend `BaseTenantEntity`
- DTOs must never be annotated as JPA entities
- Webhook controllers must reside in the webhook package
- Schedulers must reside in the scheduler package

---

- **Credential encryption**: AES-256 encrypt provider tokens before DB storage (schema V12 is already prepared with `encrypted_config` column)
- **Horizontal scaling**: Replace in-memory `SseEmitterRegistry` with Redis pub/sub, add Redis-backed Bucket4j for rate limiting across instances
- **Facebook Lead Ads**: V17 migration and `LeadIngestionService` extension already roadmapped
- **Workflow builder**: Visual drag-and-drop follow-up sequence designer
- **Instagram DM**: Shares Meta webhook infrastructure, minimal additional work
