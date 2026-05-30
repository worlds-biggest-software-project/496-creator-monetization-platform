# Data Model Suggestion 3: Hybrid Relational + JSONB Model (PostgreSQL)

## Approach

A PostgreSQL schema that keeps core transactional entities (payments, subscriptions, orders) in traditional normalized relational columns while using JSONB columns for data that varies between creators, evolves frequently, or has nested/flexible structure. GIN indexes on JSONB columns enable fast queries without sacrificing schema flexibility.

## Why This Suits the Domain

Creator monetization platforms face a fundamental tension: financial data demands strict schemas and referential integrity, while creator-facing features need rapid iteration and per-creator customization. A creator selling digital art templates has different product metadata than one selling online courses. Membership tier benefits vary wildly. Email campaign segmentation rules are deeply nested.

The hybrid approach resolves this tension by using relational columns for data that participates in JOINs, WHERE clauses, and financial calculations (amounts, statuses, foreign keys, timestamps), while JSONB handles everything that is "creator-configured," "varies by type," or "subject to frequent feature additions."

This avoids the common failure mode of pure relational schemas for multi-tenant platforms: an ever-growing ALTER TABLE backlog as each new creator use-case demands new columns.

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
    -- Flexible profile data: bio, social links, branding colors, custom fields
    profile         JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- Stripe Connect details
    stripe_account_id VARCHAR(255),
    stripe_onboarding_complete BOOLEAN NOT NULL DEFAULT FALSE,
    -- Platform settings: timezone, currency, notification preferences, widget config
    settings        JSONB NOT NULL DEFAULT '{
        "timezone": "UTC",
        "default_currency": "USD",
        "payout_schedule": "monthly",
        "notification_preferences": {
            "email_on_new_subscriber": true,
            "email_on_tip": true,
            "email_on_churn_risk": true
        }
    }'::jsonb,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_creators_slug ON creators (slug);
CREATE INDEX idx_creators_stripe ON creators (stripe_account_id) WHERE stripe_account_id IS NOT NULL;
-- Allow queries like "all creators who have connected Instagram"
CREATE INDEX idx_creators_profile ON creators USING GIN (profile jsonb_path_ops);

CREATE TABLE subscribers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255),
    password_hash   VARCHAR(255),
    email_verified  BOOLEAN NOT NULL DEFAULT FALSE,
    stripe_customer_id VARCHAR(255),
    -- Flexible profile: avatar, preferences, consent records, custom fields
    profile         JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- Privacy compliance stored as structured data
    privacy         JSONB NOT NULL DEFAULT '{
        "gdpr_consent_at": null,
        "ccpa_opt_out": false,
        "data_retention_policy": "standard",
        "marketing_consent": false
    }'::jsonb,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_subscribers_email ON subscribers (email);
CREATE INDEX idx_subscribers_stripe ON subscribers (stripe_customer_id) WHERE stripe_customer_id IS NOT NULL;
CREATE INDEX idx_subscribers_privacy ON subscribers USING GIN (privacy jsonb_path_ops);

-- ============================================================
-- MEMBERSHIP & SUBSCRIPTIONS
-- ============================================================

CREATE TABLE membership_tiers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    price_cents     INTEGER NOT NULL CHECK (price_cents >= 0),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    billing_interval VARCHAR(20) NOT NULL DEFAULT 'month'
                    CHECK (billing_interval IN ('month', 'quarter', 'year')),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- Benefits, descriptions, and feature flags as flexible JSONB
    -- Avoids a separate tier_benefits table and allows rich nested structures
    details         JSONB NOT NULL DEFAULT '{
        "description": "",
        "benefits": [],
        "feature_flags": {},
        "welcome_message": null,
        "badge_emoji": null,
        "content_access_rules": []
    }'::jsonb,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (creator_id, slug)
);

CREATE INDEX idx_tiers_creator ON membership_tiers (creator_id);
-- Query tier benefits content
CREATE INDEX idx_tiers_details ON membership_tiers USING GIN (details jsonb_path_ops);

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
    -- Flexible metadata: cancellation reason, pause details, upgrade/downgrade history
    metadata        JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_subscriptions_subscriber ON subscriptions (subscriber_id);
CREATE INDEX idx_subscriptions_creator ON subscriptions (creator_id);
CREATE INDEX idx_subscriptions_status ON subscriptions (status);
CREATE INDEX idx_subscriptions_renewal ON subscriptions (current_period_end) WHERE status = 'active';
CREATE INDEX idx_subscriptions_meta ON subscriptions USING GIN (metadata jsonb_path_ops);

-- ============================================================
-- TIPS & DONATIONS
-- ============================================================

CREATE TABLE tips (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_id   UUID REFERENCES subscribers(id) ON DELETE SET NULL,
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE RESTRICT,
    amount_cents    INTEGER NOT NULL CHECK (amount_cents > 0),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    is_anonymous    BOOLEAN NOT NULL DEFAULT FALSE,
    stripe_payment_intent_id VARCHAR(255),
    status          VARCHAR(30) NOT NULL DEFAULT 'succeeded'
                    CHECK (status IN ('pending', 'succeeded', 'failed', 'refunded')),
    -- Message, reactions, display preferences
    details         JSONB NOT NULL DEFAULT '{"message": null}'::jsonb,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tips_creator ON tips (creator_id, created_at DESC);

-- ============================================================
-- DIGITAL PRODUCTS & ORDERS
-- ============================================================

CREATE TABLE digital_products (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    price_cents     INTEGER NOT NULL CHECK (price_cents >= 0),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    product_type    VARCHAR(50) NOT NULL DEFAULT 'download'
                    CHECK (product_type IN ('download', 'template', 'course', 'media', 'bundle', 'other')),
    is_published    BOOLEAN NOT NULL DEFAULT FALSE,
    -- Type-specific metadata varies wildly between product types:
    -- Downloads: { file_url, file_size_bytes, format, preview_url }
    -- Templates: { file_url, format, software_compatibility: ["Figma", "Sketch"], preview_images: [] }
    -- Courses: { lesson_count, total_duration_seconds, difficulty, prerequisites: [] }
    -- Bundles: { included_product_ids: [], discount_pct }
    product_data    JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- SEO, tags, categories as flexible data
    catalog_data    JSONB NOT NULL DEFAULT '{
        "description": "",
        "tags": [],
        "categories": [],
        "seo_title": null,
        "seo_description": null,
        "cover_image_url": null
    }'::jsonb,
    -- Pricing rules: download limits, license terms, regional pricing
    pricing_rules   JSONB NOT NULL DEFAULT '{
        "download_limit": null,
        "license_type": "personal",
        "regional_pricing": {}
    }'::jsonb,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (creator_id, slug)
);

CREATE INDEX idx_products_creator ON digital_products (creator_id);
CREATE INDEX idx_products_type ON digital_products (product_type);
CREATE INDEX idx_products_catalog ON digital_products USING GIN (catalog_data jsonb_path_ops);
CREATE INDEX idx_products_data ON digital_products USING GIN (product_data jsonb_path_ops);

CREATE TABLE orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_id   UUID NOT NULL REFERENCES subscribers(id) ON DELETE RESTRICT,
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE RESTRICT,
    total_cents     INTEGER NOT NULL CHECK (total_cents >= 0),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(30) NOT NULL DEFAULT 'completed'
                    CHECK (status IN ('pending', 'completed', 'refunded', 'failed')),
    stripe_payment_intent_id VARCHAR(255),
    -- Line items stored inline: avoids a separate join table for simple cases
    -- Each item: { product_id, product_name, price_cents, quantity, product_type }
    items           JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- Fulfillment details: download URLs, expiry times, access grants
    fulfillment     JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_orders_subscriber ON orders (subscriber_id);
CREATE INDEX idx_orders_creator ON orders (creator_id, created_at DESC);
CREATE INDEX idx_orders_items ON orders USING GIN (items jsonb_path_ops);

-- ============================================================
-- CONTENT & PUBLISHING
-- ============================================================

CREATE TABLE posts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    title           VARCHAR(500) NOT NULL,
    slug            VARCHAR(200) NOT NULL,
    content_type    VARCHAR(30) NOT NULL DEFAULT 'text'
                    CHECK (content_type IN ('text', 'image', 'video', 'audio', 'mixed')),
    visibility      VARCHAR(30) NOT NULL DEFAULT 'public'
                    CHECK (visibility IN ('public', 'members_only', 'tier_restricted', 'draft')),
    minimum_tier_id UUID REFERENCES membership_tiers(id) ON DELETE SET NULL,
    published_at    TIMESTAMPTZ,
    -- The content body and all rich media as JSONB (supports block-based editors)
    content         JSONB NOT NULL DEFAULT '{
        "body": "",
        "excerpt": "",
        "blocks": [],
        "cover_image_url": null,
        "embedded_media": [],
        "attachments": []
    }'::jsonb,
    -- SEO and social sharing metadata
    seo             JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- Engagement counters (denormalized for read performance)
    stats           JSONB NOT NULL DEFAULT '{
        "view_count": 0,
        "like_count": 0,
        "comment_count": 0
    }'::jsonb,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (creator_id, slug)
);

CREATE INDEX idx_posts_creator ON posts (creator_id, published_at DESC);
CREATE INDEX idx_posts_visibility ON posts (visibility);
CREATE INDEX idx_posts_content ON posts USING GIN (content jsonb_path_ops);

CREATE TABLE courses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    title           VARCHAR(500) NOT NULL,
    slug            VARCHAR(200) NOT NULL,
    price_cents     INTEGER CHECK (price_cents >= 0),
    currency        CHAR(3) DEFAULT 'USD',
    minimum_tier_id UUID REFERENCES membership_tiers(id) ON DELETE SET NULL,
    is_published    BOOLEAN NOT NULL DEFAULT FALSE,
    -- Course structure stored as nested JSONB:
    -- { description, difficulty, prerequisites: [],
    --   modules: [{ title, sort_order, lessons: [{ id, title, body, video_url, duration_seconds }] }] }
    -- This avoids separate course_modules and course_lessons tables
    structure       JSONB NOT NULL DEFAULT '{"description": "", "modules": []}'::jsonb,
    -- Aggregate stats
    stats           JSONB NOT NULL DEFAULT '{"enrolled_count": 0, "completion_rate": 0}'::jsonb,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (creator_id, slug)
);

CREATE INDEX idx_courses_creator ON courses (creator_id);

CREATE TABLE course_progress (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_id   UUID NOT NULL REFERENCES subscribers(id) ON DELETE CASCADE,
    course_id       UUID NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
    -- Progress tracked per-lesson as JSONB:
    -- { "lesson-uuid-1": { completed: true, progress_pct: 100, last_accessed: "..." },
    --   "lesson-uuid-2": { completed: false, progress_pct: 45, last_accessed: "..." } }
    lesson_progress JSONB NOT NULL DEFAULT '{}'::jsonb,
    overall_pct     SMALLINT NOT NULL DEFAULT 0 CHECK (overall_pct BETWEEN 0 AND 100),
    last_accessed   TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (subscriber_id, course_id)
);

-- ============================================================
-- COMMUNITY
-- ============================================================

CREATE TABLE community_spaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    minimum_tier_id UUID REFERENCES membership_tiers(id) ON DELETE SET NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    settings        JSONB NOT NULL DEFAULT '{"description": "", "rules": [], "moderation": "manual"}'::jsonb,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE community_posts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_id        UUID NOT NULL REFERENCES community_spaces(id) ON DELETE CASCADE,
    author_id       UUID NOT NULL REFERENCES subscribers(id) ON DELETE CASCADE,
    parent_id       UUID REFERENCES community_posts(id) ON DELETE CASCADE,
    -- Post content as JSONB to support rich text, images, polls, etc.
    content         JSONB NOT NULL DEFAULT '{"title": null, "body": "", "attachments": []}'::jsonb,
    -- Reactions, flags, moderation state
    engagement      JSONB NOT NULL DEFAULT '{"reactions": {}, "flags": 0, "is_pinned": false}'::jsonb,
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
    campaign_type   VARCHAR(30) NOT NULL DEFAULT 'broadcast'
                    CHECK (campaign_type IN ('broadcast', 'drip', 'automated')),
    target_tier_id  UUID REFERENCES membership_tiers(id) ON DELETE SET NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'scheduled', 'sending', 'sent')),
    scheduled_at    TIMESTAMPTZ,
    sent_at         TIMESTAMPTZ,
    -- Email content (HTML + text), template variables, A/B test variants
    content         JSONB NOT NULL DEFAULT '{"body_html": "", "body_text": ""}'::jsonb,
    -- Targeting rules: segments, filters, exclusions
    targeting       JSONB NOT NULL DEFAULT '{"segments": [], "filters": {}, "exclusions": []}'::jsonb,
    -- Drip/automation configuration
    automation      JSONB,  -- NULL for broadcast; { trigger, delay, conditions } for drip/automated
    -- Aggregate send stats
    stats           JSONB NOT NULL DEFAULT '{
        "total_sent": 0, "opened": 0, "clicked": 0,
        "bounced": 0, "unsubscribed": 0
    }'::jsonb,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_campaigns_creator ON email_campaigns (creator_id);

CREATE TABLE email_sends (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id     UUID NOT NULL REFERENCES email_campaigns(id) ON DELETE CASCADE,
    subscriber_id   UUID NOT NULL REFERENCES subscribers(id) ON DELETE CASCADE,
    sent_at         TIMESTAMPTZ,
    -- Tracking events as JSONB array: [{ type: "opened", at: "..." }, { type: "clicked", at: "...", url: "..." }]
    events          JSONB NOT NULL DEFAULT '[]'::jsonb,
    bounced         BOOLEAN NOT NULL DEFAULT FALSE,
    unsubscribed    BOOLEAN NOT NULL DEFAULT FALSE
);

CREATE INDEX idx_email_sends_campaign ON email_sends (campaign_id);
CREATE INDEX idx_email_sends_subscriber ON email_sends (subscriber_id);

-- ============================================================
-- PAYMENTS & PAYOUTS (strictly relational -- financial data)
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
    -- Stripe response details, refund info, dispute details
    stripe_details  JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_creator ON payments (creator_id, created_at DESC);
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
    -- Breakdown, included payment IDs, bank details
    details         JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payouts_creator ON payouts (creator_id, created_at DESC);

-- ============================================================
-- ANALYTICS & AI
-- ============================================================

CREATE TABLE engagement_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL,
    subscriber_id   UUID,
    event_type      VARCHAR(50) NOT NULL,
    resource_type   VARCHAR(50),
    resource_id     UUID,
    -- All event-specific data as JSONB: session_id, ip, user_agent, duration, referrer, etc.
    event_data      JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE engagement_events_2026_05 PARTITION OF engagement_events
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE engagement_events_2026_06 PARTITION OF engagement_events
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

CREATE INDEX idx_engagement_creator_time ON engagement_events (creator_id, created_at DESC);
CREATE INDEX idx_engagement_type ON engagement_events (event_type);
CREATE INDEX idx_engagement_data ON engagement_events USING GIN (event_data jsonb_path_ops);

CREATE TABLE ai_predictions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    prediction_type VARCHAR(50) NOT NULL
                    CHECK (prediction_type IN ('churn_risk', 'segment', 'pricing', 'content_performance')),
    target_type     VARCHAR(50) NOT NULL,  -- 'subscription', 'subscriber', 'product', 'post'
    target_id       UUID NOT NULL,
    -- Prediction results as flexible JSONB:
    -- churn_risk: { probability: 0.73, risk_factors: [...], recommended_action: "..." }
    -- segment: { segment_name: "...", score: 0.87, features: {...} }
    -- pricing: { optimal_price_cents: 2499, confidence: 0.82, elasticity: -1.3 }
    -- content_performance: { predicted_views: 1200, optimal_publish_time: "..." }
    prediction      JSONB NOT NULL,
    model_version   VARCHAR(50) NOT NULL,
    predicted_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_predictions_target ON ai_predictions (target_type, target_id, predicted_at DESC);
CREATE INDEX idx_predictions_type ON ai_predictions (prediction_type);
CREATE INDEX idx_predictions_data ON ai_predictions USING GIN (prediction jsonb_path_ops);

-- ============================================================
-- WEBHOOKS & API
-- ============================================================

CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    key_hash        VARCHAR(255) NOT NULL UNIQUE,
    label           VARCHAR(255),
    scopes          JSONB NOT NULL DEFAULT '[]'::jsonb,
    last_used_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_endpoints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    url             TEXT NOT NULL,
    secret_hash     VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- Subscribed events and configuration as JSONB
    config          JSONB NOT NULL DEFAULT '{"events": [], "version": "v1"}'::jsonb,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_deliveries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    endpoint_id     UUID NOT NULL REFERENCES webhook_endpoints(id) ON DELETE CASCADE,
    event_type      VARCHAR(100) NOT NULL,
    -- Full request/response payload for debugging
    delivery        JSONB NOT NULL DEFAULT '{
        "request_body": null, "response_status": null,
        "response_body": null, "attempts": 1
    }'::jsonb,
    delivered_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_webhook_deliveries_endpoint ON webhook_deliveries (endpoint_id, created_at DESC);
```

---

## JSONB Query Examples

```sql
-- Find all creators who have connected Instagram
SELECT id, display_name FROM creators
WHERE profile @> '{"social_links": {"instagram": true}}';

-- Find all products tagged "design" in any category
SELECT id, name FROM digital_products
WHERE catalog_data @> '{"tags": ["design"]}';

-- Find high churn-risk subscribers for a creator
SELECT target_id AS subscription_id,
       prediction->>'probability' AS churn_prob,
       prediction->'risk_factors' AS factors
FROM ai_predictions
WHERE creator_id = $1
  AND prediction_type = 'churn_risk'
  AND (prediction->>'probability')::numeric > 0.7
ORDER BY (prediction->>'probability')::numeric DESC;

-- Get a course's lesson structure without JOINs
SELECT title, structure->'modules' AS modules
FROM courses WHERE id = $1;
```

---

## Trade-offs

**Strengths:**
- Best of both worlds: relational integrity for financial data, flexibility for creator-facing features.
- No schema migrations needed when adding new product metadata fields, tier benefits, or campaign targeting rules.
- GIN indexes on JSONB provide fast containment and existence queries.
- Fewer tables overall (no `tier_benefits`, `order_items`, `course_lessons` as separate tables), reducing JOIN complexity.
- PostgreSQL is the only infrastructure dependency -- no additional databases to operate.

**Weaknesses:**
- JSONB columns bypass CHECK constraints and foreign keys -- application-level validation is essential.
- JSONB updates are copy-on-write at the row level -- frequently updated JSONB columns (like stats counters) cause write amplification.
- Developers must understand both SQL and JSONB query syntax.
- JSONB data is harder to enforce consistency on across rows (e.g. ensuring all course structures have the same shape).
- GIN index maintenance adds overhead on write-heavy JSONB columns.

**Scalability Considerations:**
- Engagement events are already partitioned by month; the same pattern works for `email_sends` and `webhook_deliveries`.
- JSONB columns that grow unbounded (e.g. `course_progress.lesson_progress` for courses with 500+ lessons) should be monitored for row size.
- For analytics workloads, consider materializing JSONB data into flat columnar tables via background jobs.

**Migration Path:**
- This schema can start as the initial production schema. If JSONB columns prove too unstructured over time, individual fields can be "promoted" to proper relational columns with a simple `ALTER TABLE ADD COLUMN` plus a data backfill.
- The financial tables (payments, payouts, subscriptions) are already fully relational, so adopting event sourcing (Suggestion 2) for those domains later requires no JSONB migration.
