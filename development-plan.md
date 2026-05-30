# Creator Monetization Platform — Phased Development Plan

> Project: 496-creator-monetization-platform · Created: 2026-05-31
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan turns the research, feature survey, standards reference, and data-model
suggestions for the Creator Monetization Platform into an implementable, phased
roadmap. The product is an **open-source, self-hostable, AI-native platform** that
unifies tiered memberships, one-time tips, digital products, content publishing,
email marketing, community, and predictive analytics for the underserved middle tier
of the creator economy — without the 10%+ take rates of proprietary incumbents.

The core schema follows **Data Model Suggestion 1 (normalized relational PostgreSQL)**,
augmented with **JSONB metadata columns from Suggestion 3** for creator-configured,
type-varying data, and a **TimescaleDB migration path from Suggestion 4** for
engagement analytics at scale (Phase 9). Payments delegate PCI scope to Stripe
Connect (SAQ-A), in line with the standards reference.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend) | **TypeScript (Node.js 22 LTS)** | The product is API/integration/frontend-heavy (Stripe Connect, webhooks, embeddable widgets, REST API, dashboard). A single language across server, widgets, and dashboard reduces context-switching. Stripe's Node SDK is first-class. The ML/AI features are bounded (segmentation, churn, pricing) and run via a thin Python sidecar where needed, not the whole stack. |
| API framework | **Fastify 5** | High throughput, first-class JSON Schema validation (aligns with JSON Schema 2020-12), and built-in OpenAPI 3.1 generation via `@fastify/swagger`. Lighter and faster than Express/Nest for an API-first product. |
| Validation / contracts | **TypeBox** | Produces JSON Schema at runtime used both for Fastify request/response validation and OpenAPI 3.1 spec generation — one source of truth per the standards reference. |
| Database | **PostgreSQL 16** | Domain has well-defined transactional entities (creators, subscriptions, orders, payments) demanding ACID guarantees. Matches Data Model Suggestion 1 directly. |
| Migrations / query | **Drizzle ORM + drizzle-kit** | Type-safe SQL-first ORM; migrations are plain SQL, auditable for financial schema review. Maps cleanly to the DDL in Suggestion 1. |
| Cache / queue broker | **Redis 7** | Backs the async job queue and session/rate-limit storage. |
| Job queue | **BullMQ** | Durable, Redis-backed queues for webhook delivery, email sending, payout reconciliation, and AI batch jobs — the project's async workloads. Built-in retry/backoff aligns with Standard Webhooks retry semantics. |
| Payments | **Stripe Connect (Express accounts)** | Mandated by research/standards as the category's payment backbone. Express accounts give creators fast onboarding while keeping PCI scope at SAQ-A (no cardholder data on our servers). Handles marketplace splits, application fees, subscriptions, and payouts. |
| Object storage | **S3-compatible (AWS S3 / MinIO)** | Digital product files, post media, course videos. MinIO for self-hosters; S3 in managed mode. Presigned URLs keep large files off the app server. |
| Email delivery | **Nodemailer + pluggable transport (SMTP / SES / Resend)** | Self-hosters use SMTP; managed deployments use SES/Resend. CAN-SPAM unsubscribe + physical address enforced at the template layer. |
| Frontend (creator dashboard) | **Next.js 15 (App Router) + React 19 + Tailwind + shadcn/ui** | Server components for fast dashboards, route handlers proxy to the API. shadcn/ui gives accessible primitives quickly. |
| Embeddable widgets | **Preact + vanilla web component bundle (`<script>` embed)** | Widgets must load on any third-party site with minimal footprint. Preact keeps the bundle small; an iframe-isolated checkout keeps the host page out of PCI scope. |
| AI / ML | **Python 3.12 sidecar (FastAPI) + scikit-learn**, LLM calls via provider-agnostic gateway | Segmentation (KMeans), churn (gradient-boosted classifier), and pricing (Thompson sampling bandit) are classical ML — scikit-learn is the right tool. LLM features (content calendar, segment labelling) call an external provider through a thin gateway. |
| MCP server | **`@modelcontextprotocol/sdk` (TypeScript)** | Greenfield differentiator per standards.md — expose creator analytics/subscriber/content data to AI assistants. Reuses the API service layer. |
| Auth | **OAuth 2.0 + OIDC (Lucia-based sessions for dashboard; JWT bearer + API keys for API)** | RFC 6749/6750 + OIDC for social login; API keys (hashed) for programmatic access per Suggestion 1's `api_keys` table. |
| Webhooks (outbound) | **Standard Webhooks spec** | HMAC-SHA256 signing, `webhook-id`/`webhook-timestamp`/`webhook-signature` headers, retry with exponential backoff — a documented differentiator vs incumbents. |
| Containerisation | **Docker + docker-compose** | Self-hosted is the primary deployment model; compose bundles app, Postgres, Redis, MinIO, and the ML sidecar. |
| Testing | **Vitest (unit/integration) + Playwright (E2E) + Testcontainers (real Postgres/Redis)** | Vitest is fast and TS-native; Testcontainers gives real-dependency integration tests; Playwright drives the dashboard and embedded widget. Python sidecar uses **pytest**. |
| Code quality | **ESLint + Prettier + `tsc --noEmit` (strict)**; **ruff + mypy** for Python | Enforced in CI per Definition of Done. |
| Package manager | **pnpm (workspaces monorepo)** | Manages the API, dashboard, widgets, and shared packages in one repo with efficient installs. |
| CI | **GitHub Actions** | Lint, type-check, test matrix, Docker build, OpenAPI spec drift check. |

### Project Structure

```
creator-monetization-platform/
├── pnpm-workspace.yaml
├── package.json
├── docker-compose.yml              # app + postgres + redis + minio + ml-sidecar
├── Dockerfile                      # multi-stage build for api + dashboard
├── .github/workflows/ci.yml
├── packages/
│   ├── shared/                     # shared TS types, TypeBox schemas, money utils
│   │   └── src/
│   │       ├── schemas/            # TypeBox models -> JSON Schema -> OpenAPI
│   │       ├── money.ts            # integer-cents currency helpers
│   │       └── events.ts           # domain event type catalogue
│   ├── api/                        # Fastify backend (the core service)
│   │   └── src/
│   │       ├── server.ts
│   │       ├── config.ts           # env parsing + validation (TypeBox)
│   │       ├── db/
│   │       │   ├── schema/         # Drizzle table definitions (per Suggestion 1)
│   │       │   ├── migrations/     # SQL migrations (drizzle-kit)
│   │       │   └── client.ts
│   │       ├── modules/            # business logic, grouped by concern
│   │       │   ├── creators/
│   │       │   ├── subscribers/
│   │       │   ├── memberships/
│   │       │   ├── tips/
│   │       │   ├── products/
│   │       │   ├── content/
│   │       │   ├── community/
│   │       │   ├── email/
│   │       │   ├── payments/       # Stripe Connect integration
│   │       │   ├── analytics/
│   │       │   ├── ai/             # client to ML sidecar + LLM gateway
│   │       │   ├── webhooks/       # outbound (Standard Webhooks) + inbound (Stripe)
│   │       │   └── api-keys/
│   │       ├── routes/             # HTTP route registration per module
│   │       ├── queue/              # BullMQ queues + workers
│   │       ├── auth/               # OIDC, sessions, API-key & JWT verification
│   │       ├── plugins/            # fastify plugins (auth, rbac, rate-limit, errors)
│   │       └── mcp/                # MCP server exposing analytics/content tools
│   ├── dashboard/                  # Next.js 15 creator dashboard
│   │   └── src/app/
│   ├── widgets/                    # Preact embeddable widgets + checkout iframe
│   │   └── src/
│   └── ml/                         # Python FastAPI sidecar (scikit-learn)
│       ├── pyproject.toml
│       └── app/
│           ├── main.py
│           ├── segmentation.py
│           ├── churn.py
│           └── pricing.py
└── tests/
    ├── e2e/                        # Playwright specs
    └── fixtures/                   # sample creators, Stripe event payloads, feeds
```

The structure is grouped **by concern** (module folders), so each phase adds modules,
routes, schemas, and migrations without restructuring earlier work.

---

## Phase 1: Foundation, Config & Data Layer

### Purpose
Establish the monorepo, the database schema for the MVP entities, configuration
handling, and the running Fastify server with health checks and OpenAPI generation.
After this phase, the application boots, connects to Postgres and Redis, exposes a
documented (empty) API surface, and migrations run cleanly — the platform on which
every later phase builds.

### Tasks

#### 1.1 — Monorepo & tooling scaffold

**What**: Initialise the pnpm workspace with `shared`, `api`, `dashboard`, `widgets`, `ml` packages, linting, formatting, strict TypeScript, and CI.

**Design**:
- `pnpm-workspace.yaml` lists `packages/*`.
- Root `tsconfig.base.json`: `"strict": true`, `"noUncheckedIndexedAccess": true`, `"target": "ES2023"`, `"moduleResolution": "bundler"`.
- ESLint flat config + Prettier; ruff + mypy for `packages/ml`.
- `.github/workflows/ci.yml` jobs: `lint`, `typecheck`, `test`, `docker-build`, `openapi-drift`.
- `docker-compose.yml` services: `postgres:16`, `redis:7`, `minio`, `ml-sidecar`, `api`, `dashboard`.

**Testing**:
- `Unit: pnpm -r build` succeeds across all packages.
- `Unit: pnpm lint` and `pnpm typecheck` exit 0 on the scaffold.
- `Integration: docker compose up` brings up all services; `docker compose ps` shows healthy.

#### 1.2 — Configuration & secrets

**What**: Typed, validated configuration loaded from environment with safe defaults and fail-fast validation.

**Design**:
```ts
// packages/api/src/config.ts
import { Type, type Static } from '@sinclair/typebox';
export const ConfigSchema = Type.Object({
  NODE_ENV: Type.Union([Type.Literal('development'), Type.Literal('test'), Type.Literal('production')]),
  PORT: Type.Number({ default: 3000 }),
  DATABASE_URL: Type.String(),
  REDIS_URL: Type.String(),
  STRIPE_SECRET_KEY: Type.String(),
  STRIPE_WEBHOOK_SECRET: Type.String(),
  STRIPE_CONNECT_CLIENT_ID: Type.String(),
  PLATFORM_FEE_BPS: Type.Number({ default: 300 }),         // 3% = 300 basis points
  S3_ENDPOINT: Type.String(), S3_BUCKET: Type.String(),
  S3_ACCESS_KEY: Type.String(), S3_SECRET_KEY: Type.String(),
  SMTP_URL: Type.Optional(Type.String()),
  EMAIL_FROM: Type.String(),
  EMAIL_PHYSICAL_ADDRESS: Type.String(),                   // CAN-SPAM requirement
  ML_SIDECAR_URL: Type.String({ default: 'http://ml-sidecar:8000' }),
  OIDC_ISSUER: Type.Optional(Type.String()),
  SESSION_SECRET: Type.String({ minLength: 32 }),
});
export type Config = Static<typeof ConfigSchema>;
```
- Validate `process.env` against the schema at boot; on failure, log the missing/invalid keys and `process.exit(1)`.
- `PLATFORM_FEE_BPS` defaults to **300** (3%) — the open-source value proposition vs incumbents' 10%.

**Testing**:
- `Unit: complete env -> Config object with defaults applied (PORT=3000, PLATFORM_FEE_BPS=300)`.
- `Unit: missing DATABASE_URL -> validation error naming DATABASE_URL, process exits non-zero`.
- `Unit: SESSION_SECRET shorter than 32 chars -> validation error`.

#### 1.3 — Database schema & migrations (MVP entities)

**What**: Drizzle table definitions and the initial SQL migration for MVP core entities from Data Model Suggestion 1.

**Design**:
- Implement these tables verbatim from Suggestion 1 in `packages/api/src/db/schema/`:
  `creators`, `subscribers`, `membership_tiers`, `tier_benefits`, `subscriptions`,
  `tips`, `posts`, `payments`, `payouts`, plus a JSONB `metadata` column added to
  `creators`, `membership_tiers`, and `posts` (per Suggestion 3) for creator-configured
  flexible data, with GIN indexes.
- Money is **always integer cents** with an explicit `currency CHAR(3)`; a `shared/money.ts` enforces this (no floats).
- All status fields use the CHECK-constrained enums from Suggestion 1 (e.g. subscription status: `active | past_due | cancelled | paused | expired`).
- `updated_at` maintained by a shared trigger function `set_updated_at()`.
- Migration files are plain SQL emitted by `drizzle-kit generate`, committed and reviewed.

**Testing**:
- `Integration (Testcontainers real Postgres): run migrations up -> all tables, indexes, and CHECK constraints exist (query information_schema)`.
- `Integration: insert subscription with status 'bogus' -> CHECK constraint violation`.
- `Integration: insert tip with amount_cents = 0 -> CHECK (amount_cents > 0) violation`.
- `Integration: migrations down then up -> idempotent, no errors`.

#### 1.4 — Server bootstrap, health, error handling & OpenAPI

**What**: Fastify server with health/readiness endpoints, a uniform error envelope, request-ID logging, and auto-generated OpenAPI 3.1 spec.

**Design**:
- `GET /healthz` -> `{ status: 'ok' }` (liveness). `GET /readyz` -> checks Postgres + Redis, 200 or 503.
- Error envelope (used everywhere):
```ts
interface ApiError { error: { code: string; message: string; details?: unknown }; requestId: string; }
```
- Register `@fastify/swagger` (OpenAPI 3.1) + `@fastify/swagger-ui` at `/docs`; spec served at `/openapi.json`.
- Global error handler maps validation errors -> 422, auth errors -> 401/403, not-found -> 404, unexpected -> 500 (no stack leak in production).
- `pino` logger with per-request `requestId` (from `X-Request-Id` or generated).

**Testing**:
- `Integration: GET /healthz -> 200 { status: 'ok' }`.
- `Integration (real deps down): GET /readyz with Postgres unreachable -> 503`.
- `Integration: unknown route -> 404 with ApiError envelope and requestId`.
- `Integration: GET /openapi.json -> valid OpenAPI 3.1 document (validated with a schema validator)`.

---

## Phase 2: Identity, Auth & Authorization

### Purpose
Add creator and subscriber accounts, session-based auth for the dashboard, OIDC social
login, API keys for programmatic access, and a role/ownership authorization layer. Every
subsequent module depends on knowing *who* is calling and *what they may touch* — this
phase enforces OWASP API Security #1 (Broken Object Level Authorization) and #2 (Broken
Authentication) as a cross-cutting plugin.

### Tasks

#### 2.1 — Creator & subscriber accounts

**What**: Registration, login, email verification, and profile management for both account types.

**Design**:
- `POST /v1/creators` `{ email, displayName, slug, password }` -> creates creator, sends verification email. Slug uniqueness enforced; reserved slugs (`api`, `admin`, `docs`) rejected.
- `POST /v1/subscribers` `{ email, displayName?, password? }` — subscribers may also be created passwordlessly during checkout (Phase 4).
- Passwords hashed with **argon2id**. Email verification via signed, expiring token.
- GDPR fields populated: `subscribers.gdpr_consent_at` set when consent checkbox submitted.

**Testing**:
- `Unit: password hashing round-trips; wrong password fails verify`.
- `Integration: POST /v1/creators with duplicate slug -> 409`.
- `Integration: POST /v1/creators with reserved slug 'admin' -> 422`.
- `Integration: verify-email with expired token -> 410 Gone`.

#### 2.2 — Sessions, OIDC & login

**What**: Cookie-based sessions for the dashboard and OIDC social login (Google/Apple).

**Design**:
- Sessions stored in Redis; `Set-Cookie` with `HttpOnly; Secure; SameSite=Lax`.
- OIDC Authorization Code + PKCE (RFC 6749 + OpenID Connect Core 1.0). Callback maps `sub`/`email` to an existing or new account.
- `POST /v1/auth/login`, `POST /v1/auth/logout`, `GET /v1/auth/oidc/start`, `GET /v1/auth/oidc/callback`.
- Failed login uses constant-time comparison and a per-IP + per-account rate limit (Redis token bucket) to mitigate credential stuffing (OWASP API #2).

**Testing**:
- `Integration: login with valid creds -> session cookie set, GET /v1/me returns the creator`.
- `Integration: 5 failed logins for one account -> 429 on the 6th`.
- `Integration (mocked OIDC provider): callback with valid code -> session created`.
- `Integration: logout -> session destroyed in Redis, subsequent /v1/me -> 401`.

#### 2.3 — API keys & bearer auth

**What**: Hashed API keys with scopes for programmatic access, plus JWT bearer support.

**Design**:
- Implements Suggestion 1 `api_keys` (key_hash, scopes TEXT[], expires_at, last_used_at).
- Key format `cmp_live_<32-random-base62>`; only the SHA-256 hash is stored. Plaintext shown once on creation.
- Scopes: `read:analytics`, `read:subscribers`, `write:products`, `write:content`, `read:payments`, etc.
- `POST /v1/api-keys`, `GET /v1/api-keys`, `DELETE /v1/api-keys/:id`.
- Auth plugin resolves caller from (a) session cookie, (b) `Authorization: Bearer <jwt>`, or (c) `Authorization: Bearer <api-key>`.

**Testing**:
- `Unit: generated key verifies against its stored hash; tampered key fails`.
- `Integration: request with key lacking 'read:analytics' -> 403`.
- `Integration: request with expired key -> 401; last_used_at not updated`.
- `Integration: valid key updates last_used_at`.

#### 2.4 — Authorization (ownership + RBAC) plugin

**What**: A cross-cutting authorization layer enforcing that creators may only access their own resources.

**Design**:
- `requireCreatorOwnership(resourceLoader)` decorator: loads the resource, compares `resource.creator_id` to the authenticated creator id; 403 on mismatch, 404 if absent (no existence leak).
- Subscriber-facing routes check subscription/entitlement (used by Phase 5 content gating).
- Centralised so every module reuses it — directly addresses OWASP API #1 (BOLA).

**Testing**:
- `Integration: creator A requests creator B's tier by id -> 403`.
- `Integration: request for non-existent resource id -> 404 (not 403, but no detail leak)`.
- `Unit: ownership decorator returns 403 when creator_id differs`.

---

## Phase 3: Stripe Connect & Payments Core

### Purpose
Wire up Stripe Connect Express onboarding and the inbound webhook pipeline, and build
the canonical `payments` ledger. This is the financial heart of the platform: every
revenue stream (memberships, tips, products) in later phases settles through this layer.
PCI scope stays at SAQ-A because no cardholder data touches our servers — Stripe Checkout
and Elements handle it.

### Tasks

#### 3.1 — Stripe Connect onboarding

**What**: Creator onboarding to a Stripe Connect Express account with status tracking.

**Design**:
- `POST /v1/creators/me/stripe/onboard` -> creates a Connect Express account (if none), returns an Account Link onboarding URL.
- `GET /v1/creators/me/stripe/status` -> returns `{ onboardingComplete, chargesEnabled, payoutsEnabled }` from the Stripe account.
- Persists `creators.stripe_account_id` and `stripe_onboarding_complete`.
- Application fee on every charge = `PLATFORM_FEE_BPS` (default 3%); funds settle directly to the connected account.

**Testing**:
- `Integration (mocked Stripe): onboard with no existing account -> account created, account_id persisted, link URL returned`.
- `Integration (mocked Stripe): status reflects chargesEnabled=false until onboarding complete`.
- `Unit: application fee computed from PLATFORM_FEE_BPS (e.g. 1000 cents @ 300 bps = 30 cents)`.

#### 3.2 — Inbound Stripe webhook pipeline

**What**: Signature-verified ingestion of Stripe events into the BullMQ queue for idempotent processing.

**Design**:
- `POST /v1/webhooks/stripe` — verifies `Stripe-Signature` using `STRIPE_WEBHOOK_SECRET` against the **raw body** (raw-body parser on this route only). Invalid signature -> 400, no processing.
- Verified events enqueued to `stripe-events` BullMQ queue keyed by Stripe `event.id` for idempotency (dedupe via Redis set / unique job id).
- Handlers for: `account.updated`, `checkout.session.completed`, `invoice.paid`, `invoice.payment_failed`, `customer.subscription.updated`, `customer.subscription.deleted`, `charge.refunded`, `payout.paid`.
- Each handler is idempotent — re-delivery of the same `event.id` is a no-op.

**Testing**:
- `Integration (fixture payloads): valid signature -> 200, job enqueued`.
- `Integration: tampered body / invalid signature -> 400, no job enqueued`.
- `Integration: same event.id delivered twice -> processed once (idempotent)`.
- `Unit: signature verification rejects events older than tolerance window`.

#### 3.3 — Payments ledger

**What**: The `payments` table populated from webhook events, recording gross, platform fee, Stripe fee, and net per transaction.

**Design**:
- Implements Suggestion 1 `payments` with polymorphic `source_type` (`subscription | order | tip`) + `source_id`. Because Postgres can't FK a polymorphic column, the handler validates `source_id` exists in the right table before insert (application-level integrity, as noted in Suggestion 1's trade-offs).
- `GET /v1/payments?from&to&source_type` (creator-scoped, requires `read:payments`) — paginated with RFC 8288 `Link` headers.
- Refunds (`charge.refunded`) insert a corresponding negative/`refunded`-status payment row linked to the original.

**Testing**:
- `Integration: checkout.session.completed for a tip -> payment row with correct gross/fee/net and source_type='tip'`.
- `Integration: charge.refunded -> original marked refunded + reversing entry created`.
- `Integration: payment insert with source_id not present in tips -> rejected, error logged`.
- `Integration: GET /v1/payments paginates with Link rel=next`.

---

## Phase 4: Memberships & Tips (MVP Revenue Streams)

### Purpose
Deliver the two MVP revenue streams that ship the core value proposition: tiered
recurring memberships and one-time tips, both checked out through Stripe with funds
flowing to the creator's connected account. After this phase a creator can publish tiers,
a fan can subscribe or tip, and money moves — the product is monetising.

### Tasks

#### 4.1 — Membership tiers & benefits CRUD

**What**: Creators manage tiers and their benefit lists; each tier maps to a Stripe Price.

**Design**:
- Implements Suggestion 1 `membership_tiers` + `tier_benefits`.
- `POST/PATCH/DELETE /v1/tiers`, `GET /v1/creators/:slug/tiers` (public). On create/price-change, create or update a Stripe Product + recurring Price on the connected account; store the price id in `metadata`.
- `billing_interval` ∈ `month | quarter | year`. Soft-delete via `is_active=false` to preserve historical subscriptions.

**Testing**:
- `Integration (mocked Stripe): create tier -> Stripe Price created, tier persisted with price id in metadata`.
- `Integration: deactivate tier with active subscriptions -> is_active=false, existing subscriptions untouched`.
- `Integration: non-owner attempts PATCH tier -> 403 (Phase 2 ownership)`.

#### 4.2 — Membership subscription checkout & lifecycle

**What**: Fans subscribe to a tier via Stripe Checkout; the subscription lifecycle is driven by webhooks from Phase 3.

**Design**:
- `POST /v1/checkout/subscription` `{ tierId, subscriberEmail }` -> creates a Stripe Checkout Session (mode=subscription) with `application_fee_percent` and `transfer_data[destination]` to the creator's account; returns the hosted checkout URL.
- Webhook `checkout.session.completed` -> upsert `subscribers`, create `subscriptions` row (status `active`, period bounds from the Stripe subscription).
- `invoice.paid` -> roll `current_period_start/end`; `invoice.payment_failed` -> status `past_due`; `customer.subscription.deleted` -> `cancelled`.
- `POST /v1/subscriptions/:id/cancel` -> sets `cancel_at_period_end=true` via Stripe.

**Testing**:
- `Integration (mocked Stripe + fixture webhooks): full flow checkout -> active subscription row created`.
- `Integration: invoice.payment_failed -> status past_due`.
- `Integration: cancel -> cancel_at_period_end true; period end reached (subscription.deleted) -> cancelled`.
- `E2E (Playwright + Stripe test mode): fan completes test-card checkout -> subscription visible on dashboard`.

#### 4.3 — One-time tips

**What**: Fans send tips of arbitrary or preset amounts, optionally with a message, optionally anonymously.

**Design**:
- Implements Suggestion 1 `tips`.
- `POST /v1/checkout/tip` `{ creatorSlug, amountCents, message?, isAnonymous? }` -> Stripe Checkout Session (mode=payment) with application fee + destination charge. Minimum amount enforced (`amount_cents > 0`, configurable floor e.g. 100).
- `checkout.session.completed` -> insert `tips` row (status `succeeded`) and a `payments` ledger entry (`source_type='tip'`).

**Testing**:
- `Integration: tip below configured floor -> 422`.
- `Integration (fixture webhook): tip checkout completed -> tip row + payment ledger entry`.
- `Integration: anonymous tip -> subscriber_id null, is_anonymous true`.

#### 4.4 — Subscriber & subscription management views

**What**: Creator-facing endpoints to list/segment subscribers and subscriptions.

**Design**:
- `GET /v1/subscribers?tier&status&search` (creator-scoped, paginated, `read:subscribers`).
- `GET /v1/subscriptions?status` with renewal-date sort (uses `idx_subscriptions_renewal`).
- Returns aggregate counts (active subscribers, MRR computed from active tier prices).

**Testing**:
- `Integration: list subscribers filtered by tier -> only that tier's active subscribers`.
- `Unit: MRR computation normalises quarter/year intervals to monthly`.
- `Integration: pagination Link headers correct at boundaries`.

---

## Phase 5: Content Publishing & Access Gating

### Purpose
Give creators a publishing system for text/image/embedded media posts, with visibility
controls (public, members-only, tier-restricted) enforced by the subscription
entitlements from Phase 4. This is the relationship surface that retains members and
generates the engagement signals the AI phases later consume.

### Tasks

#### 5.1 — Posts CRUD & media uploads

**What**: Create/edit/publish posts with cover images and embedded media via S3 presigned uploads.

**Design**:
- Implements Suggestion 1 `posts` (+ JSONB `metadata` for embeds/SEO).
- `POST/PATCH /v1/posts`, `POST /v1/posts/:id/publish` (sets `published_at`, flips `visibility` off `draft`).
- `POST /v1/uploads/presign` `{ contentType, sizeBytes }` -> presigned S3 PUT URL; size/type allowlist enforced; file served back via presigned GET or CDN.

**Testing**:
- `Integration: create draft -> not returned by public feed`.
- `Integration: presign rejects disallowed content type (e.g. application/x-msdownload) -> 422`.
- `Integration: publish sets published_at and visibility`.

#### 5.2 — Access gating & entitlement checks

**What**: Enforce post visibility against the requesting subscriber's active subscription tier.

**Design**:
- Entitlement resolver: a subscriber may read a post if `visibility='public'`, OR (`members_only` and any active subscription to the creator), OR (`tier_restricted` and active subscription whose tier `sort_order >= minimum_tier_id.sort_order`).
- `GET /v1/creators/:slug/posts` returns public + entitled posts; gated posts show a teaser (`excerpt`) with a `locked: true` flag for non-entitled viewers.
- Reuses Phase 2 authorization plugin.

**Testing**:
- `Integration: anonymous viewer -> only public posts, gated posts locked with excerpt`.
- `Integration: tier-1 subscriber requests tier-2 post -> locked`.
- `Integration: tier-2 subscriber requests tier-1 post -> full body (higher tier sees lower)`.
- `Unit: entitlement resolver truth table covers all visibility × subscription combinations`.

#### 5.3 — RSS/Atom feeds

**What**: Per-creator content feeds and private member-only podcast feeds.

**Design**:
- `GET /v1/creators/:slug/feed.xml` (RSS 2.0) and `/feed.atom` (RFC 4287) for public posts.
- Private audio feeds: per-subscriber tokenised URL (`/feed/private/:token.xml`) following PSP-1 (RSS 2.0 + iTunes + Podcast namespaces) — token grants access only to entitled audio posts; revoked on subscription cancellation.

**Testing**:
- `Integration: public feed validates against RSS 2.0 schema; gated posts excluded`.
- `Integration: private feed token for active member -> entitled audio enclosures present`.
- `Integration: private feed token after cancellation -> 403`.
- `Fixture: generated podcast feed validates against PSP-1 namespaces`.

---

## Phase 6: Creator Dashboard & Analytics Core

### Purpose
Build the Next.js creator dashboard and the engagement-event ingestion plus aggregate
analytics that power it. This turns the API into a usable product for non-technical
creators and lays the data foundation (engagement events) the AI phase consumes.

### Tasks

#### 6.1 — Engagement event ingestion

**What**: Capture page views, post reads, downloads, and other engagement signals.

**Design**:
- Implements Suggestion 1 `engagement_events`. Recommend declarative monthly partitioning by `created_at` from day one (per Suggestion 1 scalability note).
- `POST /v1/events` (batched, fire-and-forget, accepts up to 50 events) -> validates and enqueues to BullMQ `events` queue for buffered bulk insert.
- Event types: `page_view`, `post_read`, `download`, `checkout_started`, `tip_started`.

**Testing**:
- `Integration: batch of 50 events -> all inserted; 51 -> 422`.
- `Integration: bulk insert worker writes to current month partition`.
- `Unit: malformed event in batch -> whole batch rejected with field path`.

#### 6.2 — Aggregate analytics endpoints

**What**: Revenue, subscriber, and engagement aggregates for dashboard widgets.

**Design**:
- `GET /v1/analytics/revenue?from&to&granularity=day|week|month` -> series of `{ period, grossCents, netCents, source breakdown }` from `payments`.
- `GET /v1/analytics/subscribers` -> active count, new/churned this period, MRR, by-tier breakdown.
- `GET /v1/analytics/content/top` -> posts ranked by reads/conversions in window.
- Backed by materialized views refreshed on a BullMQ cron (per Suggestion 1 scalability path).

**Testing**:
- `Integration (seeded data): revenue by month matches hand-computed sums`.
- `Integration: subscriber churn count matches cancelled-in-window subscriptions`.
- `Unit: granularity bucketing aligns to UTC period boundaries`.

#### 6.3 — Dashboard UI

**What**: Next.js dashboard with onboarding, tiers, posts, subscribers, and analytics screens.

**Design**:
- Routes: `/onboarding` (Stripe Connect), `/tiers`, `/posts`, `/subscribers`, `/analytics`, `/settings`.
- Server components fetch via the API using the session cookie; charts via a lightweight chart lib; shadcn/ui primitives for accessibility (WCAG AA targets).
- Guided creator onboarding mirroring Patreon/Ko-fi UX patterns (tier setup wizard).

**Testing**:
- `E2E (Playwright): new creator signs up -> completes Stripe onboarding (test) -> creates a tier -> publishes a post`.
- `E2E: analytics page renders revenue chart with seeded data`.
- `Accessibility: axe scan on each route -> no critical violations`.

---

## Phase 7: Digital Products Shop & Embeddable Widgets

### Purpose
Add the v1.1 digital-product revenue stream (downloads, templates, media) with secure
delivery, and ship the embeddable checkout/membership widgets that let creators monetise
from any existing website — a key differentiator (Gumroad/Fourthwall parity) without
forcing migration.

### Tasks

#### 7.1 — Digital products & secure delivery

**What**: Creators sell downloadable products; buyers receive limited, audited download access.

**Design**:
- Implements Suggestion 1 `digital_products`, `orders`, `order_items`, `download_records`.
- `POST /v1/products`, `GET /v1/creators/:slug/products` (public). File uploaded via presign (Phase 5.1) to a **private** bucket.
- `POST /v1/checkout/product` -> Stripe Checkout (mode=payment, destination charge) -> on completion, create `orders` + `order_items` + `payments` entry.
- `GET /v1/downloads/:orderItemId` -> verifies ownership + `download_limit`, issues a short-TTL presigned GET, writes a `download_records` row (ip, timestamp).

**Testing**:
- `Integration (fixture webhook): product purchase -> order + order_items + payment entry`.
- `Integration: download beyond download_limit -> 403`.
- `Integration: non-purchaser requests download -> 403`.
- `Integration: each download writes a download_records row with TTL'd URL`.

#### 7.2 — Embeddable widgets bundle

**What**: A small `<script>`-embeddable bundle rendering tip buttons, membership prompts, and product cards on any third-party site.

**Design**:
- `packages/widgets` builds a Preact bundle served from the API/CDN: `<script src=".../embed.js" data-creator="slug" data-widget="tip|membership|product"></script>`.
- Widget calls public read endpoints (CORS-allowlisted) and opens checkout in an **iframe-isolated** Stripe Checkout (host page never sees card data — keeps host at zero PCI scope).
- Theming via `data-*` attributes; graceful no-JS fallback link.

**Testing**:
- `Integration: embed.js loads tier data from public API with correct CORS headers`.
- `E2E (Playwright on a fixture host page): clicking tip widget opens iframe checkout`.
- `Unit: bundle gzipped size under budget (e.g. < 40 KB)`.

#### 7.3 — Outbound webhooks (Standard Webhooks)

**What**: Let creators register webhook endpoints and receive signed event notifications.

**Design**:
- Implements Suggestion 1 `webhook_endpoints` + `webhook_deliveries`.
- Follows the **Standard Webhooks** spec: headers `webhook-id`, `webhook-timestamp`, `webhook-signature` (HMAC-SHA256 over `id.timestamp.body`, base64).
- Event catalogue: `subscription.created/updated/cancelled`, `tip.created`, `order.completed`, `payout.paid`.
- Delivery via BullMQ with exponential backoff retries (e.g. 5 attempts); each attempt recorded in `webhook_deliveries`.
- `POST/GET/DELETE /v1/webhook-endpoints`.

**Testing**:
- `Unit: signature matches Standard Webhooks reference vectors`.
- `Integration (mock receiver): subscription.created -> signed POST delivered, delivery row recorded`.
- `Integration: receiver returns 500 -> retried with backoff, attempts incremented`.
- `Integration: receiver verifies signature with stored secret -> accepts; tampered payload -> rejects`.

---

## Phase 8: Email Marketing & Community

### Purpose
Add the v1.1 engagement features: email campaigns (broadcast + drip) with CAN-SPAM-compliant
delivery, and community discussion spaces gated by tier. These deepen the creator-fan
relationship and produce further engagement signals for the AI phase.

### Tasks

#### 8.1 — Email campaigns & sending

**What**: Compose broadcast and drip campaigns targeted by tier; track delivery and engagement.

**Design**:
- Implements Suggestion 1 `email_campaigns` + `email_sends`.
- `POST /v1/campaigns` (draft) -> `POST /v1/campaigns/:id/schedule` or `/send`. Sending fans out one `email_sends` row per recipient via the BullMQ `email` queue (rate-limited to transport limits).
- **CAN-SPAM enforced at template layer**: every email includes a working unsubscribe link and `EMAIL_PHYSICAL_ADDRESS`; missing either -> send blocked.
- Open/click tracking pixels + redirect links update `email_sends.opened_at/clicked_at`. Unsubscribe sets a suppression flag honored on all future sends.

**Testing**:
- `Integration: campaign without unsubscribe link -> send blocked with explanatory error`.
- `Integration (mock transport): broadcast to tier -> one email_sends row per active member of that tier`.
- `Integration: unsubscribed subscriber excluded from subsequent sends`.
- `Integration: click on tracked link -> redirect 302 + clicked_at set`.

#### 8.2 — Drip sequences

**What**: Time-delayed automated email sequences triggered by events (e.g. new subscription).

**Design**:
- Drip steps stored in campaign `metadata` (JSONB) as ordered `{ delayHours, subject, bodyHtml }`.
- Trigger `subscription.created` enqueues delayed BullMQ jobs per step; cancellation removes pending jobs.

**Testing**:
- `Integration: new subscription enqueues step 1 immediately, step 2 delayed`.
- `Integration: cancellation before step 2 -> step 2 job removed, not sent`.

#### 8.3 — Community spaces & threaded discussion

**What**: Tier-gated discussion spaces with threaded posts.

**Design**:
- Implements Suggestion 1 `community_spaces` + `community_posts` (self-referential `parent_id` for threads).
- `POST /v1/spaces`, `POST /v1/spaces/:id/posts`, `GET /v1/spaces/:id/posts` (paginated, threaded). Access gated by `minimum_tier_id` via the Phase 5 entitlement resolver.
- Basic moderation: creator can delete any post in their space.

**Testing**:
- `Integration: non-entitled subscriber posting to gated space -> 403`.
- `Integration: threaded reply -> parent_id set, returned nested`.
- `Integration: creator deletes a member post -> cascade removes replies`.

---

## Phase 9: AI Intelligence Layer

### Purpose
Deliver the AI-native differentiators that no incumbent offers: subscriber segmentation,
churn prediction with retention triggers, and dynamic product pricing. Runs in the Python
ML sidecar over the engagement/payment data accumulated in earlier phases. This is the
project's headline competitive advantage. Optionally migrate engagement analytics to
TimescaleDB here (Suggestion 4) if event volume warrants.

### Tasks

#### 9.1 — ML sidecar service

**What**: A FastAPI Python service exposing segmentation, churn, and pricing endpoints, called by the API's `ai` module.

**Design**:
- `packages/ml` FastAPI app; endpoints `POST /segment`, `POST /churn/score`, `POST /pricing/recommend`.
- Feature extraction reads a read-replica / analytics view (no direct write coupling). Models persisted to S3 with a `model_version` string written into prediction rows.
- API `ai` module calls the sidecar over `ML_SIDECAR_URL`; sidecar is internal-only (not exposed publicly).

**Testing**:
- `pytest: sidecar /segment with synthetic feature matrix -> stable cluster assignments`.
- `Integration: API ai module handles sidecar timeout -> 503 with retry, no dashboard crash`.

#### 9.2 — Subscriber segmentation

**What**: ML clustering of subscribers by engagement depth and spending potential into labelled segments.

**Design**:
- Implements Suggestion 1 `subscriber_segments` + `subscriber_segment_members` (with `score`).
- Features per subscriber: recency, frequency, monetary (RFM) from `payments`/`engagement_events`; KMeans (k chosen by silhouette). Segments labelled (optionally via LLM gateway: "high-value advocates", "at-risk lurkers").
- BullMQ cron recomputes segments daily; `segment_type='ai_generated'`.

**Testing**:
- `pytest: RFM feature extraction matches hand-computed values on fixture`.
- `Integration: segmentation run -> segment + member rows with scores in [0,1]`.
- `Integration: segments power the analytics endpoint (Phase 6) and email targeting (Phase 8)`.

#### 9.3 — Churn prediction & retention triggers

**What**: Score active subscriptions for cancellation risk and trigger personalised retention offers before renewal.

**Design**:
- Implements Suggestion 1 `churn_predictions` (probability, predicted_at, model_version).
- Gradient-boosted classifier trained on historical cancellations; features include engagement decline, days-since-last-read, payment failures.
- Daily scoring; subscriptions over a configurable threshold (e.g. 0.6) within N days of renewal trigger a `subscription.at_risk` outbound webhook + an automated retention email (Phase 8 drip).

**Testing**:
- `pytest: classifier AUC above baseline on held-out fixture set`.
- `Integration: high churn score near renewal -> retention drip enqueued + at_risk webhook fired`.
- `Integration: churn_predictions row stamped with model_version`.

#### 9.4 — Dynamic pricing experiments

**What**: Algorithmically test price points for digital products to maximise revenue per item.

**Design**:
- Implements Suggestion 1 `pricing_experiments` (variant_name, price_cents, impressions, conversions, revenue_cents).
- Thompson-sampling multi-armed bandit selects which price variant a product page shows per impression; conversions feed back to update the posterior.
- `POST /v1/products/:id/pricing-experiment` to start; results surfaced on dashboard; winner can be promoted to the product's base price.

**Testing**:
- `pytest: bandit converges to the higher-revenue arm on simulated traffic`.
- `Integration: impression increments chosen variant; purchase increments conversions + revenue`.
- `Integration: promoting a winner updates digital_products.price_cents and ends the experiment`.

---

## Phase 10: MCP Server, API Hardening & Compliance

### Purpose
Expose the platform to AI assistants via a Model Context Protocol server (a greenfield
differentiator), and complete the cross-cutting hardening required for production:
rate limiting, OWASP API Top 10 coverage, GDPR/CCPA data-subject workflows, and the
finalized public OpenAPI surface. This phase makes the platform safe and integrable.

### Tasks

#### 10.1 — MCP server

**What**: An MCP server exposing read-only creator analytics, subscriber segments, and content performance as tools/resources for LLM clients.

**Design**:
- `packages/api/src/mcp` using `@modelcontextprotocol/sdk` (spec 2025-11-25).
- Tools: `get_revenue_summary`, `list_top_content`, `get_subscriber_segments`, `get_churn_at_risk`. Resources: creator profile, recent posts.
- Authenticated via API key with `read:analytics`; reuses the API service layer (no logic duplication). Read-only — no mutating tools.

**Testing**:
- `Integration (MCP client harness): list tools -> expected catalogue; call get_revenue_summary -> matches analytics endpoint output`.
- `Integration: MCP call with key lacking read:analytics -> denied`.

#### 10.2 — Rate limiting & resource limits

**What**: Per-key/per-IP rate limiting and payload/resource caps (OWASP API #4 Unrestricted Resource Consumption).

**Design**:
- Redis token-bucket rate limiter plugin: default 100 req/min per API key, configurable per scope; `429` with `Retry-After` and `RateLimit-*` headers.
- Global body-size limit, pagination max page size (100), and query-complexity guards on analytics endpoints.

**Testing**:
- `Integration: exceed rate limit -> 429 with Retry-After`.
- `Integration: request body over limit -> 413`.
- `Integration: page size > 100 -> clamped to 100`.

#### 10.3 — GDPR/CCPA data-subject workflows

**What**: Subscriber data export and deletion (right to access / erasure) and consent management.

**Design**:
- `POST /v1/subscribers/me/export` -> async job compiles all subscriber data into a JSON archive, delivered via TTL'd presigned link.
- `DELETE /v1/subscribers/me` -> anonymises PII (email/name nulled, `subscribers.id` retained for financial-record integrity per Suggestion 1's `ON DELETE RESTRICT`), honors `ccpa_opt_out`.
- Jurisdiction-aware consent flags (`gdpr_consent_at`, `ccpa_opt_out`) surfaced and enforced.

**Testing**:
- `Integration: export request -> archive contains subscriptions, orders, tips, email history`.
- `Integration: deletion anonymises PII but preserves payment ledger referential integrity`.
- `Integration: ccpa_opt_out -> subscriber excluded from AI segmentation/marketing`.

#### 10.4 — Public API finalization & OpenAPI publication

**What**: Lock the versioned public API, publish the OpenAPI 3.1 spec, and add a drift check to CI.

**Design**:
- All public routes under `/v1`; deprecation policy documented. `/openapi.json` reflects every route via TypeBox schemas.
- CI `openapi-drift` job fails if the committed spec differs from the generated one.
- Security review against OWASP API Security Top 10 with a checklist artifact.

**Testing**:
- `Integration: generated OpenAPI validates as 3.1 and includes all /v1 routes with security schemes`.
- `CI: introducing an undocumented route fails the drift check`.
- `Security: automated OWASP API Top 10 checklist passes (BOLA, auth, resource limits verified by earlier phase tests)`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Config & Data Layer        ─── required by everything
    │
Phase 2: Identity, Auth & Authorization         ─── requires Phase 1
    │
Phase 3: Stripe Connect & Payments Core         ─── requires Phase 2
    │
Phase 4: Memberships & Tips (MVP revenue)       ─── requires Phase 3
    │
    ├── Phase 5: Content Publishing & Gating     ─── requires Phase 4 (entitlements)
    │
    └── Phase 6: Dashboard & Analytics Core      ─── requires Phase 4
             │
             ├── Phase 7: Products & Widgets      ─── requires Phase 3,6 │ can parallel Phase 8
             │
             └── Phase 8: Email & Community        ─── requires Phase 5,6 │ can parallel Phase 7
                      │
             Phase 9: AI Intelligence Layer        ─── requires Phase 6,7,8 (needs accumulated data)
                      │
             Phase 10: MCP, Hardening & Compliance ─── requires Phase 9 (and all public surfaces)
```

**Parallelism opportunities:**
- **Phases 7 and 8** can be developed concurrently once Phases 5 and 6 are complete (products/widgets vs email/community are independent module sets).
- Within Phase 6, the **dashboard UI (6.3)** can be built in parallel with **analytics endpoints (6.2)** against mocked API responses.
- The **Python ML sidecar scaffold (9.1)** can be started early (after Phase 6 produces engagement data shapes) in parallel with Phases 7/8.

**MVP cut line:** Phases 1–6 deliver the README's "Must-have (MVP)" scope (tiers, tips,
content, dashboard, Stripe Connect, plus embeddable widgets land early in 7.2). Phases 7–8
are "Should-have (v1.1)". Phases 9–10 deliver the AI-native differentiators and the
integration/compliance surface.

---

## Definition of Done (per phase)

Every phase must satisfy all of the following before it is considered complete:

1. All tasks in the phase implemented.
2. All unit and integration tests pass (`pnpm test`; `pytest` for any `packages/ml` work).
3. Real-dependency integration tests (Testcontainers Postgres/Redis) pass.
4. Linting and formatting pass (`pnpm lint`, `prettier --check`; `ruff` for Python).
5. Type checking passes with strict mode (`tsc --noEmit`; `mypy` for Python).
6. `docker compose up` builds and runs the full stack; `/readyz` returns 200.
7. The phase's primary capability works end-to-end (verified by at least one E2E or fixture-driven test).
8. New configuration options documented in the env schema and README.
9. New public API endpoints appear in the generated OpenAPI 3.1 spec, and the `openapi-drift` CI check passes.
10. New database changes shipped as committed, reversible Drizzle SQL migrations that apply cleanly up and down.
11. For payment-touching phases: no cardholder data persisted on platform servers (SAQ-A scope preserved); webhook handlers are idempotent.
12. For data-subject-touching phases: GDPR/CCPA consent flags respected; deletions preserve financial referential integrity.
```
