# Data Model Suggestion 1: Normalized Relational Model (PostgreSQL)

## Approach

A traditional third-normal-form (3NF+) relational schema in PostgreSQL. Every entity gets its own table with proper foreign keys, indexes, and constraints. This is the most widely understood approach and provides strong data integrity guarantees out of the box.

## Why This Suits the Domain

Creator monetization platforms have well-defined entities (creators, subscribers, subscriptions, products, orders, payments, content) with clear relationships between them. The transactional nature of payments and subscriptions demands ACID guarantees. A normalized schema prevents data anomalies in financial records, enforces referential integrity across the membership/billing lifecycle, and maps naturally to the domain's entity hierarchy.

The main trade-off is that complex analytics queries (e.g. "revenue by tier by month with churn overlay") require multi-table joins that can become expensive at scale. Read-heavy dashboards may eventually need materialized views or a separate analytics store.

---

## Schema Definition

```sql
-- ============================================================
-- CORE IDENTITY
-- ============================================================

CREATE TABLE creators (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    bio             TEXT,
    avatar_url      TEXT,
    website_url     TEXT,
    stripe_account_id VARCHAR(255),       -- Stripe Connect account
    stripe_onboarding_complete BOOLEAN NOT NULL DEFAULT FALSE,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_creators_slug ON creators (slug);
CREATE INDEX idx_creators_stripe ON creators (stripe_account_id) WHERE stripe_account_id IS NOT NULL;

CREATE TABLE subscribers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255),
    avatar_url      TEXT,
    password_hash   VARCHAR(255),
    email_verified  BOOLEAN NOT NULL DEFAULT FALSE,
    gdpr_consent_at TIMESTAMPTZ,
    ccpa_opt_out    BOOLEAN NOT NULL DEFAULT FALSE,
    stripe_customer_id VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_subscribers_email ON subscribers (email);
CREATE INDEX idx_subscribers_stripe ON subscribers (stripe_customer_id) WHERE stripe_customer_id IS NOT NULL;

-- ============================================================
-- MEMBERSHIP & SUBSCRIPTIONS
-- ============================================================

CREATE TABLE membership_tiers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    price_cents     INTEGER NOT NULL CHECK (price_cents >= 0),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    billing_interval VARCHAR(20) NOT NULL DEFAULT 'month'
                    CHECK (billing_interval IN ('month', 'quarter', 'year')),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (creator_id, slug)
);

CREATE INDEX idx_membership_tiers_creator ON membership_tiers (creator_id);

CREATE TABLE tier_benefits (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tier_id         UUID NOT NULL REFERENCES membership_tiers(id) ON DELETE CASCADE,
    benefit_text    VARCHAR(500) NOT NULL,
    sort_order      INTEGER NOT NULL DEFAULT 0
);

CREATE INDEX idx_tier_benefits_tier ON tier_benefits (tier_id);

CREATE TABLE subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_id   UUID NOT NULL REFERENCES subscribers(id) ON DELETE RESTRICT,
    tier_id         UUID NOT NULL REFERENCES membership_tiers(id) ON DELETE RESTRICT,
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE RESTRICT,
    status          VARCHAR(30) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'past_due', 'cancelled', 'paused', 'expired')),
    stripe_subscription_id VARCHAR(255),
    current_period_start TIMESTAMPTZ NOT NULL,
    current_period_end   TIMESTAMPTZ NOT NULL,
    cancelled_at    TIMESTAMPTZ,
    cancel_at_period_end BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_subscriptions_subscriber ON subscriptions (subscriber_id);
CREATE INDEX idx_subscriptions_creator ON subscriptions (creator_id);
CREATE INDEX idx_subscriptions_tier ON subscriptions (tier_id);
CREATE INDEX idx_subscriptions_status ON subscriptions (status);
CREATE INDEX idx_subscriptions_renewal ON subscriptions (current_period_end) WHERE status = 'active';

-- ============================================================
-- TIPS & ONE-TIME DONATIONS
-- ============================================================

CREATE TABLE tips (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_id   UUID REFERENCES subscribers(id) ON DELETE SET NULL,
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE RESTRICT,
    amount_cents    INTEGER NOT NULL CHECK (amount_cents > 0),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    message         TEXT,
    is_anonymous    BOOLEAN NOT NULL DEFAULT FALSE,
    stripe_payment_intent_id VARCHAR(255),
    status          VARCHAR(30) NOT NULL DEFAULT 'succeeded'
                    CHECK (status IN ('pending', 'succeeded', 'failed', 'refunded')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tips_creator ON tips (creator_id);
CREATE INDEX idx_tips_subscriber ON tips (subscriber_id) WHERE subscriber_id IS NOT NULL;
CREATE INDEX idx_tips_created ON tips (created_at);

-- ============================================================
-- DIGITAL PRODUCTS & ORDERS
-- ============================================================

CREATE TABLE digital_products (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    price_cents     INTEGER NOT NULL CHECK (price_cents >= 0),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    product_type    VARCHAR(50) NOT NULL DEFAULT 'download'
                    CHECK (product_type IN ('download', 'template', 'course', 'media', 'other')),
    file_url        TEXT,
    file_size_bytes BIGINT,
    preview_url     TEXT,
    is_published    BOOLEAN NOT NULL DEFAULT FALSE,
    download_limit  INTEGER,   -- NULL = unlimited
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (creator_id, slug)
);

CREATE INDEX idx_digital_products_creator ON digital_products (creator_id);
CREATE INDEX idx_digital_products_type ON digital_products (product_type);

CREATE TABLE orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_id   UUID NOT NULL REFERENCES subscribers(id) ON DELETE RESTRICT,
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE RESTRICT,
    total_cents     INTEGER NOT NULL CHECK (total_cents >= 0),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(30) NOT NULL DEFAULT 'completed'
                    CHECK (status IN ('pending', 'completed', 'refunded', 'failed')),
    stripe_payment_intent_id VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_orders_subscriber ON orders (subscriber_id);
CREATE INDEX idx_orders_creator ON orders (creator_id);
CREATE INDEX idx_orders_created ON orders (created_at);

CREATE TABLE order_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id      UUID NOT NULL REFERENCES digital_products(id) ON DELETE RESTRICT,
    price_cents     INTEGER NOT NULL CHECK (price_cents >= 0),
    quantity        INTEGER NOT NULL DEFAULT 1 CHECK (quantity > 0)
);

CREATE INDEX idx_order_items_order ON order_items (order_id);
CREATE INDEX idx_order_items_product ON order_items (product_id);

CREATE TABLE download_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_id   UUID NOT NULL REFERENCES subscribers(id) ON DELETE CASCADE,
    product_id      UUID NOT NULL REFERENCES digital_products(id) ON DELETE CASCADE,
    order_id        UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    ip_address      INET,
    downloaded_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_downloads_subscriber_product ON download_records (subscriber_id, product_id);

-- ============================================================
-- CONTENT & PUBLISHING
-- ============================================================

CREATE TABLE posts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    title           VARCHAR(500) NOT NULL,
    slug            VARCHAR(200) NOT NULL,
    body            TEXT,
    excerpt         TEXT,
    cover_image_url TEXT,
    content_type    VARCHAR(30) NOT NULL DEFAULT 'text'
                    CHECK (content_type IN ('text', 'image', 'video', 'audio', 'mixed')),
    visibility      VARCHAR(30) NOT NULL DEFAULT 'public'
                    CHECK (visibility IN ('public', 'members_only', 'tier_restricted', 'draft')),
    minimum_tier_id UUID REFERENCES membership_tiers(id) ON DELETE SET NULL,
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (creator_id, slug)
);

CREATE INDEX idx_posts_creator ON posts (creator_id);
CREATE INDEX idx_posts_published ON posts (published_at DESC) WHERE visibility != 'draft';
CREATE INDEX idx_posts_visibility ON posts (visibility);

CREATE TABLE courses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    title           VARCHAR(500) NOT NULL,
    slug            VARCHAR(200) NOT NULL,
    description     TEXT,
    price_cents     INTEGER CHECK (price_cents >= 0),  -- NULL = included with tier
    currency        CHAR(3) DEFAULT 'USD',
    minimum_tier_id UUID REFERENCES membership_tiers(id) ON DELETE SET NULL,
    is_published    BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (creator_id, slug)
);

CREATE TABLE course_lessons (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id       UUID NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
    title           VARCHAR(500) NOT NULL,
    body            TEXT,
    video_url       TEXT,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    duration_seconds INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_course_lessons_course ON course_lessons (course_id);

CREATE TABLE lesson_progress (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_id   UUID NOT NULL REFERENCES subscribers(id) ON DELETE CASCADE,
    lesson_id       UUID NOT NULL REFERENCES course_lessons(id) ON DELETE CASCADE,
    completed       BOOLEAN NOT NULL DEFAULT FALSE,
    progress_pct    SMALLINT NOT NULL DEFAULT 0 CHECK (progress_pct BETWEEN 0 AND 100),
    last_accessed   TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (subscriber_id, lesson_id)
);

-- ============================================================
-- COMMUNITY
-- ============================================================

CREATE TABLE community_spaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    minimum_tier_id UUID REFERENCES membership_tiers(id) ON DELETE SET NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE community_posts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_id        UUID NOT NULL REFERENCES community_spaces(id) ON DELETE CASCADE,
    author_id       UUID NOT NULL REFERENCES subscribers(id) ON DELETE CASCADE,
    title           VARCHAR(500),
    body            TEXT NOT NULL,
    parent_id       UUID REFERENCES community_posts(id) ON DELETE CASCADE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_community_posts_space ON community_posts (space_id, created_at DESC);
CREATE INDEX idx_community_posts_parent ON community_posts (parent_id) WHERE parent_id IS NOT NULL;

-- ============================================================
-- EMAIL MARKETING
-- ============================================================

CREATE TABLE email_campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    subject         VARCHAR(500) NOT NULL,
    body_html       TEXT NOT NULL,
    body_text       TEXT,
    campaign_type   VARCHAR(30) NOT NULL DEFAULT 'broadcast'
                    CHECK (campaign_type IN ('broadcast', 'drip', 'automated')),
    target_tier_id  UUID REFERENCES membership_tiers(id) ON DELETE SET NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'scheduled', 'sending', 'sent')),
    scheduled_at    TIMESTAMPTZ,
    sent_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_email_campaigns_creator ON email_campaigns (creator_id);

CREATE TABLE email_sends (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id     UUID NOT NULL REFERENCES email_campaigns(id) ON DELETE CASCADE,
    subscriber_id   UUID NOT NULL REFERENCES subscribers(id) ON DELETE CASCADE,
    sent_at         TIMESTAMPTZ,
    opened_at       TIMESTAMPTZ,
    clicked_at      TIMESTAMPTZ,
    bounced         BOOLEAN NOT NULL DEFAULT FALSE,
    unsubscribed    BOOLEAN NOT NULL DEFAULT FALSE
);

CREATE INDEX idx_email_sends_campaign ON email_sends (campaign_id);
CREATE INDEX idx_email_sends_subscriber ON email_sends (subscriber_id);

-- ============================================================
-- PAYMENTS & PAYOUTS
-- ============================================================

CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE RESTRICT,
    subscriber_id   UUID REFERENCES subscribers(id) ON DELETE SET NULL,
    source_type     VARCHAR(30) NOT NULL
                    CHECK (source_type IN ('subscription', 'order', 'tip')),
    source_id       UUID NOT NULL,
    gross_amount_cents INTEGER NOT NULL,
    platform_fee_cents INTEGER NOT NULL DEFAULT 0,
    stripe_fee_cents   INTEGER NOT NULL DEFAULT 0,
    net_amount_cents   INTEGER NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    stripe_payment_intent_id VARCHAR(255),
    stripe_charge_id VARCHAR(255),
    status          VARCHAR(30) NOT NULL DEFAULT 'succeeded'
                    CHECK (status IN ('pending', 'succeeded', 'failed', 'refunded')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_creator ON payments (creator_id, created_at DESC);
CREATE INDEX idx_payments_subscriber ON payments (subscriber_id) WHERE subscriber_id IS NOT NULL;
CREATE INDEX idx_payments_source ON payments (source_type, source_id);
CREATE INDEX idx_payments_status ON payments (status);

CREATE TABLE payouts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE RESTRICT,
    amount_cents    INTEGER NOT NULL CHECK (amount_cents > 0),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    stripe_payout_id VARCHAR(255),
    status          VARCHAR(30) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'in_transit', 'paid', 'failed', 'cancelled')),
    period_start    TIMESTAMPTZ NOT NULL,
    period_end      TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payouts_creator ON payouts (creator_id, created_at DESC);
CREATE INDEX idx_payouts_status ON payouts (status);

-- ============================================================
-- ANALYTICS & AI
-- ============================================================

CREATE TABLE engagement_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    subscriber_id   UUID REFERENCES subscribers(id) ON DELETE SET NULL,
    event_type      VARCHAR(50) NOT NULL,  -- page_view, post_read, download, etc.
    resource_type   VARCHAR(50),
    resource_id     UUID,
    session_id      UUID,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_engagement_creator_time ON engagement_events (creator_id, created_at DESC);
CREATE INDEX idx_engagement_subscriber ON engagement_events (subscriber_id, created_at DESC)
    WHERE subscriber_id IS NOT NULL;
CREATE INDEX idx_engagement_type ON engagement_events (event_type);

CREATE TABLE subscriber_segments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    segment_type    VARCHAR(30) NOT NULL DEFAULT 'manual'
                    CHECK (segment_type IN ('manual', 'ai_generated', 'rule_based')),
    criteria        TEXT,   -- serialised rule or AI label
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE subscriber_segment_members (
    segment_id      UUID NOT NULL REFERENCES subscriber_segments(id) ON DELETE CASCADE,
    subscriber_id   UUID NOT NULL REFERENCES subscribers(id) ON DELETE CASCADE,
    score           NUMERIC(5,4),   -- relevance / churn probability
    added_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (segment_id, subscriber_id)
);

CREATE TABLE churn_predictions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID NOT NULL REFERENCES subscriptions(id) ON DELETE CASCADE,
    churn_probability NUMERIC(5,4) NOT NULL CHECK (churn_probability BETWEEN 0 AND 1),
    predicted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    model_version   VARCHAR(50) NOT NULL
);

CREATE INDEX idx_churn_subscription ON churn_predictions (subscription_id, predicted_at DESC);

CREATE TABLE pricing_experiments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    product_id      UUID NOT NULL REFERENCES digital_products(id) ON DELETE CASCADE,
    variant_name    VARCHAR(100) NOT NULL,
    price_cents     INTEGER NOT NULL CHECK (price_cents >= 0),
    impressions     INTEGER NOT NULL DEFAULT 0,
    conversions     INTEGER NOT NULL DEFAULT 0,
    revenue_cents   BIGINT NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    ended_at        TIMESTAMPTZ
);

CREATE INDEX idx_pricing_exp_product ON pricing_experiments (product_id);

-- ============================================================
-- WEBHOOKS & API
-- ============================================================

CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    key_hash        VARCHAR(255) NOT NULL UNIQUE,
    label           VARCHAR(255),
    scopes          TEXT[] NOT NULL DEFAULT '{}',
    last_used_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_endpoints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    url             TEXT NOT NULL,
    events          TEXT[] NOT NULL,
    secret_hash     VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_deliveries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    endpoint_id     UUID NOT NULL REFERENCES webhook_endpoints(id) ON DELETE CASCADE,
    event_type      VARCHAR(100) NOT NULL,
    payload         TEXT NOT NULL,
    response_status INTEGER,
    response_body   TEXT,
    attempts        SMALLINT NOT NULL DEFAULT 1,
    delivered_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_webhook_deliveries_endpoint ON webhook_deliveries (endpoint_id, created_at DESC);
```

---

## Trade-offs

**Strengths:**
- Strong referential integrity prevents orphaned records in financial data.
- Standard SQL tooling, ORMs, and migration frameworks work out of the box.
- Clear audit trail through the payments table linking to source types.
- Easy to reason about for new developers joining the project.

**Weaknesses:**
- Analytics queries over engagement_events will slow down as the table grows into hundreds of millions of rows. Partitioning by `created_at` (monthly or weekly) is recommended early.
- The polymorphic `source_type`/`source_id` pattern in payments cannot enforce a foreign key at the database level. Application-level validation is required.
- Schema migrations for new features require careful coordination in production.

**Scalability Path:**
1. Partition `engagement_events`, `email_sends`, and `payments` by month using PostgreSQL declarative partitioning.
2. Add materialized views for dashboard aggregates (daily revenue, subscriber counts by tier).
3. When analytics load exceeds what the primary can handle, replicate to a read replica dedicated to dashboards and AI model training.
4. Consider moving engagement events to a time-series or columnar store (TimescaleDB, ClickHouse) if volume exceeds tens of millions of rows per month.

**Migration Path:**
This normalized schema serves as an excellent starting point. It can evolve toward the hybrid JSONB approach (Suggestion 3) by adding JSONB columns to specific tables for flexible metadata without restructuring the core schema.
