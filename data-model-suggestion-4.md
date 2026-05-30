# Data Model Suggestion 4: Time-Series Hypertable Model (TimescaleDB + PostgreSQL)

## Approach

A hybrid architecture that uses TimescaleDB (a PostgreSQL extension) for all time-series and event data -- engagement analytics, payment transactions, email tracking, AI predictions -- while keeping entity/reference data in standard PostgreSQL relational tables. TimescaleDB hypertables automatically partition time-series data, provide built-in continuous aggregates (materialized rollups), and offer compression that reduces storage by 90%+ for historical data.

## Why This Approach Suits the Domain

Creator monetization platforms generate several distinct categories of time-series data that grow without bound:

1. **Engagement events**: Page views, post reads, video watches, downloads -- potentially millions of rows per day across all creators.
2. **Financial transactions**: Payments, subscription renewals, tips, refunds -- each timestamped and requiring aggregation for dashboards.
3. **Email tracking**: Send, open, click, bounce events across campaigns -- high volume, append-only, queried by time range.
4. **AI prediction history**: Churn scores, segment assignments, pricing recommendations -- tracked over time to measure model accuracy.

All four categories share the same access pattern: append-only writes, queries filtered by time range and creator_id, and aggregation (SUM, COUNT, AVG) over time buckets (hourly, daily, monthly). This is exactly what time-series databases are optimized for.

Standard PostgreSQL handles these workloads poorly at scale because B-tree indexes on timestamp columns become fragmented, VACUUM struggles with append-heavy tables, and aggregation queries scan too many rows. TimescaleDB solves all three problems with automatic chunk-based partitioning, chunk-level compression, and continuous aggregates that incrementally maintain rollup tables.

The key advantage over a separate analytics database (e.g. ClickHouse) is that TimescaleDB is a PostgreSQL extension -- it runs in the same database instance, uses the same SQL, and supports JOINs between hypertables and regular tables. No ETL pipeline, no data synchronization, no second database to operate.

---

## Schema Definition

### Entity Tables (Standard PostgreSQL)

```sql
-- ============================================================
-- REFERENCE / ENTITY TABLES (standard relational)
-- These tables are small, rarely change, and are JOINed with hypertables.
-- ============================================================

CREATE TABLE creators (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    bio             TEXT,
    avatar_url      TEXT,
    website_url     TEXT,
    stripe_account_id VARCHAR(255),
    stripe_onboarding_complete BOOLEAN NOT NULL DEFAULT FALSE,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_creators_slug ON creators (slug);

CREATE TABLE subscribers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255),
    password_hash   VARCHAR(255),
    email_verified  BOOLEAN NOT NULL DEFAULT FALSE,
    stripe_customer_id VARCHAR(255),
    gdpr_consent_at TIMESTAMPTZ,
    ccpa_opt_out    BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

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
    benefits        TEXT[],
    sort_order      INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (creator_id, slug)
);

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
CREATE INDEX idx_subscriptions_status ON subscriptions (status);

CREATE TABLE digital_products (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    price_cents     INTEGER NOT NULL CHECK (price_cents >= 0),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    product_type    VARCHAR(50) NOT NULL DEFAULT 'download',
    file_url        TEXT,
    is_published    BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (creator_id, slug)
);

CREATE TABLE posts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    title           VARCHAR(500) NOT NULL,
    slug            VARCHAR(200) NOT NULL,
    body            TEXT,
    content_type    VARCHAR(30) NOT NULL DEFAULT 'text',
    visibility      VARCHAR(30) NOT NULL DEFAULT 'public',
    minimum_tier_id UUID REFERENCES membership_tiers(id) ON DELETE SET NULL,
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (creator_id, slug)
);

CREATE TABLE courses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    title           VARCHAR(500) NOT NULL,
    slug            VARCHAR(200) NOT NULL,
    description     TEXT,
    price_cents     INTEGER CHECK (price_cents >= 0),
    currency        CHAR(3) DEFAULT 'USD',
    minimum_tier_id UUID REFERENCES membership_tiers(id) ON DELETE SET NULL,
    is_published    BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (creator_id, slug)
);

CREATE TABLE course_lessons (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id       UUID NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
    title           VARCHAR(500) NOT NULL,
    body            TEXT,
    video_url       TEXT,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    duration_seconds INTEGER
);

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE email_campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    subject         VARCHAR(500) NOT NULL,
    body_html       TEXT NOT NULL,
    campaign_type   VARCHAR(30) NOT NULL DEFAULT 'broadcast',
    target_tier_id  UUID REFERENCES membership_tiers(id) ON DELETE SET NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    scheduled_at    TIMESTAMPTZ,
    sent_at         TIMESTAMPTZ,
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
```

### Hypertables (TimescaleDB Time-Series)

```sql
-- ============================================================
-- ENABLE TIMESCALEDB
-- ============================================================
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- ============================================================
-- HYPERTABLE: Financial Transactions
-- The core financial event stream. Every money movement is a row.
-- ============================================================

CREATE TABLE financial_transactions (
    id              UUID NOT NULL DEFAULT gen_random_uuid(),
    ts              TIMESTAMPTZ NOT NULL DEFAULT now(),
    creator_id      UUID NOT NULL,
    subscriber_id   UUID,
    transaction_type VARCHAR(30) NOT NULL,
        -- 'subscription_payment', 'product_sale', 'tip',
        -- 'refund', 'payout', 'platform_fee'
    source_type     VARCHAR(30),           -- 'subscription', 'order', 'tip'
    source_id       UUID,
    gross_amount_cents INTEGER NOT NULL,
    platform_fee_cents INTEGER NOT NULL DEFAULT 0,
    stripe_fee_cents   INTEGER NOT NULL DEFAULT 0,
    net_amount_cents   INTEGER NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(30) NOT NULL DEFAULT 'succeeded',
    stripe_id       VARCHAR(255),          -- payment_intent, charge, or payout ID
    metadata        JSONB DEFAULT '{}'
);

-- Convert to hypertable with 1-week chunks
SELECT create_hypertable('financial_transactions', 'ts',
    chunk_time_interval => INTERVAL '1 week');

-- Primary query: creator dashboard revenue over time
CREATE INDEX idx_fin_creator_ts ON financial_transactions (creator_id, ts DESC);
CREATE INDEX idx_fin_type ON financial_transactions (transaction_type, ts DESC);
CREATE INDEX idx_fin_subscriber ON financial_transactions (subscriber_id, ts DESC)
    WHERE subscriber_id IS NOT NULL;

-- ============================================================
-- HYPERTABLE: Engagement Events
-- All user interactions: views, reads, clicks, downloads, etc.
-- ============================================================

CREATE TABLE engagement_events (
    id              UUID NOT NULL DEFAULT gen_random_uuid(),
    ts              TIMESTAMPTZ NOT NULL DEFAULT now(),
    creator_id      UUID NOT NULL,
    subscriber_id   UUID,
    event_type      VARCHAR(50) NOT NULL,
        -- 'page_view', 'post_read', 'video_watch', 'download',
        -- 'product_view', 'checkout_start', 'checkout_complete',
        -- 'community_post', 'community_reply', 'course_lesson_view'
    resource_type   VARCHAR(50),           -- 'post', 'product', 'course', 'page'
    resource_id     UUID,
    session_id      UUID,
    duration_seconds INTEGER,
    referrer        TEXT,
    user_agent      TEXT,
    ip_address      INET,
    metadata        JSONB DEFAULT '{}'
);

SELECT create_hypertable('engagement_events', 'ts',
    chunk_time_interval => INTERVAL '1 day');

CREATE INDEX idx_engage_creator_ts ON engagement_events (creator_id, ts DESC);
CREATE INDEX idx_engage_type ON engagement_events (event_type, ts DESC);
CREATE INDEX idx_engage_resource ON engagement_events (resource_type, resource_id, ts DESC)
    WHERE resource_id IS NOT NULL;
CREATE INDEX idx_engage_subscriber ON engagement_events (subscriber_id, ts DESC)
    WHERE subscriber_id IS NOT NULL;

-- ============================================================
-- HYPERTABLE: Email Tracking Events
-- Every email send/open/click/bounce as a time-series event
-- ============================================================

CREATE TABLE email_events (
    id              UUID NOT NULL DEFAULT gen_random_uuid(),
    ts              TIMESTAMPTZ NOT NULL DEFAULT now(),
    campaign_id     UUID NOT NULL,
    creator_id      UUID NOT NULL,
    subscriber_id   UUID NOT NULL,
    event_type      VARCHAR(30) NOT NULL,
        -- 'sent', 'delivered', 'opened', 'clicked', 'bounced', 'unsubscribed'
    link_url        TEXT,                  -- for click events
    metadata        JSONB DEFAULT '{}'
);

SELECT create_hypertable('email_events', 'ts',
    chunk_time_interval => INTERVAL '1 week');

CREATE INDEX idx_email_ev_campaign ON email_events (campaign_id, ts DESC);
CREATE INDEX idx_email_ev_subscriber ON email_events (subscriber_id, ts DESC);

-- ============================================================
-- HYPERTABLE: AI Prediction Log
-- Track all ML model predictions over time for model monitoring
-- ============================================================

CREATE TABLE prediction_log (
    id              UUID NOT NULL DEFAULT gen_random_uuid(),
    ts              TIMESTAMPTZ NOT NULL DEFAULT now(),
    creator_id      UUID NOT NULL,
    prediction_type VARCHAR(50) NOT NULL,
        -- 'churn_risk', 'segment_assignment', 'price_optimization',
        -- 'content_performance', 'optimal_send_time'
    target_type     VARCHAR(50) NOT NULL,
    target_id       UUID NOT NULL,
    model_version   VARCHAR(50) NOT NULL,
    score           NUMERIC(7,4),          -- primary prediction value
    prediction_data JSONB NOT NULL,        -- full prediction details
    outcome_data    JSONB                  -- filled later when ground truth is known
);

SELECT create_hypertable('prediction_log', 'ts',
    chunk_time_interval => INTERVAL '1 month');

CREATE INDEX idx_pred_target ON prediction_log (target_type, target_id, ts DESC);
CREATE INDEX idx_pred_type ON prediction_log (prediction_type, ts DESC);
CREATE INDEX idx_pred_model ON prediction_log (model_version, ts DESC);

-- ============================================================
-- HYPERTABLE: Subscription State Changes
-- Immutable log of every subscription status transition
-- ============================================================

CREATE TABLE subscription_events (
    id              UUID NOT NULL DEFAULT gen_random_uuid(),
    ts              TIMESTAMPTZ NOT NULL DEFAULT now(),
    subscription_id UUID NOT NULL,
    creator_id      UUID NOT NULL,
    subscriber_id   UUID NOT NULL,
    event_type      VARCHAR(50) NOT NULL,
        -- 'created', 'renewed', 'upgraded', 'downgraded',
        -- 'paused', 'resumed', 'cancelled', 'expired', 'payment_failed'
    old_tier_id     UUID,
    new_tier_id     UUID,
    old_status      VARCHAR(30),
    new_status      VARCHAR(30),
    metadata        JSONB DEFAULT '{}'
);

SELECT create_hypertable('subscription_events', 'ts',
    chunk_time_interval => INTERVAL '1 month');

CREATE INDEX idx_sub_ev_subscription ON subscription_events (subscription_id, ts DESC);
CREATE INDEX idx_sub_ev_creator ON subscription_events (creator_id, ts DESC);

-- ============================================================
-- HYPERTABLE: Webhook Deliveries
-- ============================================================

CREATE TABLE webhook_deliveries (
    id              UUID NOT NULL DEFAULT gen_random_uuid(),
    ts              TIMESTAMPTZ NOT NULL DEFAULT now(),
    endpoint_id     UUID NOT NULL,
    creator_id      UUID NOT NULL,
    event_type      VARCHAR(100) NOT NULL,
    payload         JSONB NOT NULL,
    response_status INTEGER,
    response_body   TEXT,
    attempts        SMALLINT NOT NULL DEFAULT 1,
    delivered       BOOLEAN NOT NULL DEFAULT FALSE
);

SELECT create_hypertable('webhook_deliveries', 'ts',
    chunk_time_interval => INTERVAL '1 month');

CREATE INDEX idx_wh_endpoint ON webhook_deliveries (endpoint_id, ts DESC);
```

### Continuous Aggregates (Automatic Rollups)

```sql
-- ============================================================
-- CONTINUOUS AGGREGATE: Daily Revenue per Creator
-- Automatically maintained by TimescaleDB as new data arrives
-- ============================================================

CREATE MATERIALIZED VIEW daily_revenue
WITH (timescaledb.continuous) AS
SELECT
    creator_id,
    time_bucket('1 day', ts) AS day,
    SUM(CASE WHEN transaction_type = 'subscription_payment' THEN net_amount_cents ELSE 0 END)
        AS subscription_revenue_cents,
    SUM(CASE WHEN transaction_type = 'product_sale' THEN net_amount_cents ELSE 0 END)
        AS product_revenue_cents,
    SUM(CASE WHEN transaction_type = 'tip' THEN net_amount_cents ELSE 0 END)
        AS tip_revenue_cents,
    SUM(CASE WHEN transaction_type = 'refund' THEN net_amount_cents ELSE 0 END)
        AS refund_cents,
    SUM(net_amount_cents) AS total_net_cents,
    COUNT(*) AS transaction_count
FROM financial_transactions
WHERE status = 'succeeded'
GROUP BY creator_id, time_bucket('1 day', ts)
WITH NO DATA;

-- Refresh policy: update daily aggregate every hour, covering the last 3 days
SELECT add_continuous_aggregate_policy('daily_revenue',
    start_offset => INTERVAL '3 days',
    end_offset   => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');

-- ============================================================
-- CONTINUOUS AGGREGATE: Hourly Engagement per Creator
-- Powers the real-time engagement dashboard
-- ============================================================

CREATE MATERIALIZED VIEW hourly_engagement
WITH (timescaledb.continuous) AS
SELECT
    creator_id,
    time_bucket('1 hour', ts) AS hour,
    event_type,
    COUNT(*) AS event_count,
    COUNT(DISTINCT subscriber_id) AS unique_visitors,
    AVG(duration_seconds) AS avg_duration
FROM engagement_events
GROUP BY creator_id, time_bucket('1 hour', ts), event_type
WITH NO DATA;

SELECT add_continuous_aggregate_policy('hourly_engagement',
    start_offset => INTERVAL '2 days',
    end_offset   => INTERVAL '30 minutes',
    schedule_interval => INTERVAL '30 minutes');

-- ============================================================
-- CONTINUOUS AGGREGATE: Daily Subscription Metrics
-- New subscriptions, cancellations, upgrades, downgrades per day
-- ============================================================

CREATE MATERIALIZED VIEW daily_subscription_metrics
WITH (timescaledb.continuous) AS
SELECT
    creator_id,
    time_bucket('1 day', ts) AS day,
    COUNT(*) FILTER (WHERE event_type = 'created') AS new_subscriptions,
    COUNT(*) FILTER (WHERE event_type = 'cancelled') AS cancellations,
    COUNT(*) FILTER (WHERE event_type = 'upgraded') AS upgrades,
    COUNT(*) FILTER (WHERE event_type = 'downgraded') AS downgrades,
    COUNT(*) FILTER (WHERE event_type = 'renewed') AS renewals,
    COUNT(*) FILTER (WHERE event_type = 'payment_failed') AS payment_failures
FROM subscription_events
GROUP BY creator_id, time_bucket('1 day', ts)
WITH NO DATA;

SELECT add_continuous_aggregate_policy('daily_subscription_metrics',
    start_offset => INTERVAL '3 days',
    end_offset   => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');

-- ============================================================
-- CONTINUOUS AGGREGATE: Email Campaign Performance
-- ============================================================

CREATE MATERIALIZED VIEW campaign_performance
WITH (timescaledb.continuous) AS
SELECT
    campaign_id,
    creator_id,
    time_bucket('1 hour', ts) AS hour,
    COUNT(*) FILTER (WHERE event_type = 'sent') AS sent,
    COUNT(*) FILTER (WHERE event_type = 'delivered') AS delivered,
    COUNT(*) FILTER (WHERE event_type = 'opened') AS opened,
    COUNT(*) FILTER (WHERE event_type = 'clicked') AS clicked,
    COUNT(*) FILTER (WHERE event_type = 'bounced') AS bounced,
    COUNT(*) FILTER (WHERE event_type = 'unsubscribed') AS unsubscribed
FROM email_events
GROUP BY campaign_id, creator_id, time_bucket('1 hour', ts)
WITH NO DATA;

SELECT add_continuous_aggregate_policy('campaign_performance',
    start_offset => INTERVAL '2 days',
    end_offset   => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');
```

### Compression and Retention Policies

```sql
-- ============================================================
-- COMPRESSION: Reduce storage for historical data by 90%+
-- ============================================================

-- Compress engagement events older than 7 days
ALTER TABLE engagement_events SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'creator_id',
    timescaledb.compress_orderby = 'ts DESC'
);
SELECT add_compression_policy('engagement_events', INTERVAL '7 days');

-- Compress financial transactions older than 30 days
ALTER TABLE financial_transactions SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'creator_id',
    timescaledb.compress_orderby = 'ts DESC'
);
SELECT add_compression_policy('financial_transactions', INTERVAL '30 days');

-- Compress email events older than 14 days
ALTER TABLE email_events SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'creator_id,campaign_id',
    timescaledb.compress_orderby = 'ts DESC'
);
SELECT add_compression_policy('email_events', INTERVAL '14 days');

-- Compress prediction logs older than 30 days
ALTER TABLE prediction_log SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'creator_id,prediction_type',
    timescaledb.compress_orderby = 'ts DESC'
);
SELECT add_compression_policy('prediction_log', INTERVAL '30 days');

-- ============================================================
-- RETENTION: Drop raw data older than retention window
-- (continuous aggregates preserve the rollups indefinitely)
-- ============================================================

-- Keep raw engagement events for 90 days
SELECT add_retention_policy('engagement_events', INTERVAL '90 days');

-- Keep raw email events for 180 days
SELECT add_retention_policy('email_events', INTERVAL '180 days');

-- Keep webhook deliveries for 30 days
SELECT add_retention_policy('webhook_deliveries', INTERVAL '30 days');

-- Financial transactions and subscription events: NEVER auto-delete
-- (required for compliance and audit purposes)
```

---

## Trade-offs

**Strengths:**
- Purpose-built for the analytics-heavy workload that differentiates this platform from competitors. Dashboard queries that would require complex window functions and full table scans in vanilla PostgreSQL are pre-computed by continuous aggregates.
- Automatic compression reduces storage costs by 90%+ for historical data without application changes.
- Retention policies automate data lifecycle management -- critical for GDPR compliance (engagement data can be auto-deleted while financial records are preserved).
- Single PostgreSQL instance -- no separate analytics database, no ETL, no data synchronization. JOINs between hypertables and regular tables work natively.
- The AI/ML pipeline can query the prediction_log hypertable to compare predictions against outcomes (stored in outcome_data), enabling model performance monitoring over time.

**Weaknesses:**
- TimescaleDB is an extension, not vanilla PostgreSQL. Some managed PostgreSQL providers do not support it (though Timescale Cloud, AWS, and Aura do).
- Hypertables have restrictions: no foreign key constraints on hypertables, no unique constraints across chunks (except on the partitioning column). Application-level validation is required for referential integrity on time-series tables.
- Compressed chunks are read-only -- late-arriving updates (e.g. backfilling an outcome_data field on prediction_log) require decompression first.
- Continuous aggregates add background worker load. With many aggregates, the refresh scheduler needs monitoring.
- Developers unfamiliar with TimescaleDB need to learn chunk management, compression behavior, and continuous aggregate semantics.

**Scalability Considerations:**
- TimescaleDB scales writes linearly with chunk count. A single node handles millions of inserts per day comfortably.
- For extreme scale (100M+ events/day), TimescaleDB multinode distributes chunks across multiple PostgreSQL instances.
- Continuous aggregates can be layered: hourly aggregates feed daily aggregates, which feed monthly aggregates, creating a rollup hierarchy.
- Compression is the primary cost-control lever: 1TB of raw engagement data compresses to approximately 50-100GB.

**Migration Path:**
- Start with the normalized relational model (Suggestion 1) using standard PostgreSQL. When analytics performance degrades or storage costs become significant, add TimescaleDB and migrate the engagement_events and payments tables to hypertables. The SQL remains identical -- only the underlying storage engine changes.
- Continuous aggregates can replace hand-maintained materialized views incrementally.
- The entity tables never need to change; only the time-series tables are affected by the TimescaleDB migration.
