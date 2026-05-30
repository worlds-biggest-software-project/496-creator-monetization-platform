# Data Model Suggestion 2: Event-Sourced / CQRS Model

## Approach

An event-sourced architecture with Command Query Responsibility Segregation (CQRS). Instead of storing current state, every state change is recorded as an immutable domain event. The write side appends events to an event store, while read-optimized projections (materialized views) are built from those events for queries and dashboards.

## Why This Suits the Domain

Creator monetization platforms have several properties that align well with event sourcing:

- **Financial auditability**: Every payment, subscription change, refund, and payout is naturally an event. An append-only log provides a complete, tamper-resistant audit trail -- critical for PCI DSS and financial compliance.
- **Temporal queries**: "What was this subscriber's tier on March 15?" or "How did revenue build over the month?" are trivially answered by replaying events up to a point in time.
- **AI/analytics pipeline**: The event stream is a natural source for machine learning pipelines (churn prediction, segmentation) without needing to build separate ETL.
- **Multi-projection flexibility**: The same events can power a creator dashboard, an admin view, a subscriber portal, and an analytics engine -- each with its own optimized read model.

The main trade-off is increased complexity. Developers must understand eventual consistency, event schema evolution, and projection rebuilds. Simple CRUD operations (updating a creator profile) feel over-engineered when modeled as events.

---

## Event Store Schema

The event store is the single source of truth. All other tables are derived projections that can be rebuilt from events.

```sql
-- ============================================================
-- EVENT STORE (the single source of truth)
-- ============================================================

CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(100) NOT NULL,   -- 'Creator', 'Subscription', 'Order', etc.
    aggregate_id    UUID NOT NULL,
    sequence_number BIGINT NOT NULL,          -- per-aggregate ordering
    event_type      VARCHAR(200) NOT NULL,    -- 'SubscriptionCreated', 'PaymentSucceeded', etc.
    event_data      JSONB NOT NULL,           -- the full event payload
    metadata        JSONB NOT NULL DEFAULT '{}', -- correlation_id, causation_id, user_id
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (aggregate_id, sequence_number)
);

-- Primary query path: load all events for an aggregate in order
CREATE INDEX idx_events_aggregate ON event_store (aggregate_id, sequence_number);

-- Support projection rebuilds and subscriptions by type + time
CREATE INDEX idx_events_type_time ON event_store (event_type, occurred_at);

-- Global ordering for catch-up subscriptions
CREATE INDEX idx_events_occurred ON event_store (occurred_at);

-- Partition by month for long-term manageability
-- CREATE TABLE event_store_2026_01 PARTITION OF event_store
--     FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

-- ============================================================
-- SNAPSHOTS (optional, for performance)
-- ============================================================

CREATE TABLE aggregate_snapshots (
    aggregate_type  VARCHAR(100) NOT NULL,
    aggregate_id    UUID NOT NULL,
    sequence_number BIGINT NOT NULL,
    snapshot_data   JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_id, sequence_number)
);
```

---

## Domain Events Catalogue

Below are the core domain events grouped by aggregate. Each event is stored as JSONB in `event_data`.

### Creator Aggregate

```json
// CreatorRegistered
{
  "creator_id": "uuid",
  "email": "creator@example.com",
  "display_name": "Jane Creator",
  "slug": "jane-creator"
}

// CreatorProfileUpdated
{
  "creator_id": "uuid",
  "changes": { "bio": "New bio text", "avatar_url": "https://..." }
}

// StripeAccountConnected
{
  "creator_id": "uuid",
  "stripe_account_id": "acct_xxx",
  "onboarding_complete": true
}

// MembershipTierCreated
{
  "tier_id": "uuid",
  "creator_id": "uuid",
  "name": "Gold",
  "price_cents": 999,
  "currency": "USD",
  "billing_interval": "month",
  "benefits": ["Exclusive posts", "Monthly Q&A"]
}

// MembershipTierUpdated
{
  "tier_id": "uuid",
  "changes": { "price_cents": 1299 },
  "effective_for_new_subscribers_only": true
}
```

### Subscription Aggregate

```json
// SubscriptionCreated
{
  "subscription_id": "uuid",
  "subscriber_id": "uuid",
  "creator_id": "uuid",
  "tier_id": "uuid",
  "price_cents": 999,
  "currency": "USD",
  "billing_interval": "month",
  "period_start": "2026-05-01T00:00:00Z",
  "period_end": "2026-06-01T00:00:00Z",
  "stripe_subscription_id": "sub_xxx"
}

// SubscriptionRenewed
{
  "subscription_id": "uuid",
  "new_period_start": "2026-06-01T00:00:00Z",
  "new_period_end": "2026-07-01T00:00:00Z",
  "payment_id": "uuid"
}

// SubscriptionTierChanged
{
  "subscription_id": "uuid",
  "old_tier_id": "uuid",
  "new_tier_id": "uuid",
  "effective_at": "2026-06-01T00:00:00Z",
  "proration_amount_cents": -500
}

// SubscriptionCancelled
{
  "subscription_id": "uuid",
  "reason": "too_expensive",
  "cancel_at_period_end": true,
  "feedback": "Love the content but budget is tight"
}

// SubscriptionPaused
{
  "subscription_id": "uuid",
  "resume_at": "2026-08-01T00:00:00Z"
}

// SubscriptionResumed
{
  "subscription_id": "uuid",
  "new_period_start": "2026-08-01T00:00:00Z"
}
```

### Payment Aggregate

```json
// PaymentSucceeded
{
  "payment_id": "uuid",
  "creator_id": "uuid",
  "subscriber_id": "uuid",
  "source_type": "subscription",
  "source_id": "uuid",
  "gross_amount_cents": 999,
  "platform_fee_cents": 50,
  "stripe_fee_cents": 59,
  "net_amount_cents": 890,
  "currency": "USD",
  "stripe_payment_intent_id": "pi_xxx"
}

// PaymentFailed
{
  "payment_id": "uuid",
  "source_type": "subscription",
  "source_id": "uuid",
  "failure_code": "card_declined",
  "failure_message": "Insufficient funds",
  "retry_count": 1,
  "next_retry_at": "2026-05-04T00:00:00Z"
}

// RefundIssued
{
  "refund_id": "uuid",
  "original_payment_id": "uuid",
  "amount_cents": 999,
  "reason": "requested_by_customer"
}

// PayoutInitiated
{
  "payout_id": "uuid",
  "creator_id": "uuid",
  "amount_cents": 45000,
  "currency": "USD",
  "period_start": "2026-05-01T00:00:00Z",
  "period_end": "2026-05-31T23:59:59Z",
  "stripe_payout_id": "po_xxx"
}

// PayoutCompleted
{
  "payout_id": "uuid",
  "arrived_at": "2026-06-03T00:00:00Z"
}
```

### Order / Product Aggregate

```json
// DigitalProductCreated
{
  "product_id": "uuid",
  "creator_id": "uuid",
  "name": "Design Templates Pack",
  "price_cents": 2999,
  "product_type": "download"
}

// OrderPlaced
{
  "order_id": "uuid",
  "subscriber_id": "uuid",
  "creator_id": "uuid",
  "items": [
    { "product_id": "uuid", "price_cents": 2999, "quantity": 1 }
  ],
  "total_cents": 2999
}

// OrderFulfilled
{
  "order_id": "uuid",
  "download_urls": [{ "product_id": "uuid", "url": "https://...", "expires_at": "..." }]
}

// TipReceived
{
  "tip_id": "uuid",
  "creator_id": "uuid",
  "subscriber_id": "uuid",
  "amount_cents": 500,
  "message": "Great video!",
  "is_anonymous": false
}
```

### Content Aggregate

```json
// PostPublished
{
  "post_id": "uuid",
  "creator_id": "uuid",
  "title": "My Latest Update",
  "visibility": "members_only",
  "minimum_tier_id": "uuid",
  "content_type": "text"
}

// PostViewed
{
  "post_id": "uuid",
  "subscriber_id": "uuid",
  "view_duration_seconds": 120
}

// CourseCreated
{
  "course_id": "uuid",
  "creator_id": "uuid",
  "title": "Intro to Illustration",
  "lesson_count": 12
}

// LessonCompleted
{
  "course_id": "uuid",
  "lesson_id": "uuid",
  "subscriber_id": "uuid"
}
```

### Engagement / AI Aggregate

```json
// SubscriberSegmented
{
  "subscriber_id": "uuid",
  "creator_id": "uuid",
  "segment_name": "high_engagement_low_spend",
  "score": 0.87,
  "model_version": "seg-v2.3"
}

// ChurnRiskPredicted
{
  "subscription_id": "uuid",
  "churn_probability": 0.73,
  "risk_factors": ["declining_engagement", "payment_failure_history"],
  "model_version": "churn-v1.5",
  "recommended_action": "send_retention_offer"
}

// PricingExperimentStarted
{
  "experiment_id": "uuid",
  "product_id": "uuid",
  "variants": [
    { "name": "control", "price_cents": 2999 },
    { "name": "discount_10", "price_cents": 2699 }
  ]
}
```

---

## Command Handlers

Commands validate business rules and emit events. Example pseudocode:

```
CreateSubscription(command):
    validate subscriber exists
    validate tier exists and is active
    validate no existing active subscription for this creator
    validate payment method on file
    emit SubscriptionCreated { ... }
    emit PaymentSucceeded { ... }

CancelSubscription(command):
    load subscription aggregate from events
    validate subscription is active
    validate subscriber owns this subscription
    emit SubscriptionCancelled { reason, cancel_at_period_end }

PlaceOrder(command):
    validate all products exist and are published
    validate subscriber has payment method
    process payment via Stripe
    if payment succeeds:
        emit OrderPlaced { ... }
        emit PaymentSucceeded { ... }
        emit OrderFulfilled { ... }
    else:
        emit PaymentFailed { ... }

RenewSubscriptions(scheduled_job):
    for each subscription where period_end <= now():
        load subscription aggregate
        attempt payment via Stripe
        if payment succeeds:
            emit SubscriptionRenewed { ... }
            emit PaymentSucceeded { ... }
        else:
            emit PaymentFailed { retry_count, next_retry_at }
            if retry_count >= 3:
                emit SubscriptionCancelled { reason: "payment_failed" }
```

---

## Read Projections (Query Side)

Projections consume events and maintain denormalized read models optimized for specific queries.

```sql
-- ============================================================
-- PROJECTION: Creator Dashboard
-- ============================================================

CREATE TABLE proj_creator_dashboard (
    creator_id          UUID PRIMARY KEY,
    display_name        VARCHAR(255),
    total_subscribers   INTEGER NOT NULL DEFAULT 0,
    active_subscribers  INTEGER NOT NULL DEFAULT 0,
    mrr_cents           BIGINT NOT NULL DEFAULT 0,
    total_revenue_cents BIGINT NOT NULL DEFAULT 0,
    total_tips_cents    BIGINT NOT NULL DEFAULT 0,
    total_product_sales INTEGER NOT NULL DEFAULT 0,
    avg_churn_risk      NUMERIC(5,4) DEFAULT 0,
    last_updated        TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- PROJECTION: Active Subscriptions (for access control)
-- ============================================================

CREATE TABLE proj_active_subscriptions (
    subscription_id     UUID PRIMARY KEY,
    subscriber_id       UUID NOT NULL,
    creator_id          UUID NOT NULL,
    tier_id             UUID NOT NULL,
    tier_name           VARCHAR(255),
    price_cents         INTEGER NOT NULL,
    status              VARCHAR(30) NOT NULL,
    current_period_end  TIMESTAMPTZ NOT NULL,
    last_updated        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_subs_subscriber ON proj_active_subscriptions (subscriber_id);
CREATE INDEX idx_proj_subs_creator ON proj_active_subscriptions (creator_id);

-- ============================================================
-- PROJECTION: Revenue Timeline (for charts)
-- ============================================================

CREATE TABLE proj_revenue_daily (
    creator_id          UUID NOT NULL,
    date                DATE NOT NULL,
    subscription_revenue_cents BIGINT NOT NULL DEFAULT 0,
    product_revenue_cents      BIGINT NOT NULL DEFAULT 0,
    tip_revenue_cents          BIGINT NOT NULL DEFAULT 0,
    refund_cents               BIGINT NOT NULL DEFAULT 0,
    net_revenue_cents          BIGINT NOT NULL DEFAULT 0,
    new_subscribers     INTEGER NOT NULL DEFAULT 0,
    churned_subscribers INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (creator_id, date)
);

-- ============================================================
-- PROJECTION: Subscriber Profile (for subscriber portal)
-- ============================================================

CREATE TABLE proj_subscriber_profile (
    subscriber_id       UUID PRIMARY KEY,
    email               VARCHAR(255) NOT NULL,
    display_name        VARCHAR(255),
    total_spent_cents   BIGINT NOT NULL DEFAULT 0,
    active_subscriptions INTEGER NOT NULL DEFAULT 0,
    products_purchased  INTEGER NOT NULL DEFAULT 0,
    member_since        TIMESTAMPTZ,
    last_activity       TIMESTAMPTZ,
    last_updated        TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- PROJECTION: Content Feed (for subscriber content access)
-- ============================================================

CREATE TABLE proj_content_feed (
    post_id             UUID PRIMARY KEY,
    creator_id          UUID NOT NULL,
    title               VARCHAR(500),
    excerpt             TEXT,
    content_type        VARCHAR(30),
    visibility          VARCHAR(30),
    minimum_tier_id     UUID,
    published_at        TIMESTAMPTZ,
    view_count          INTEGER NOT NULL DEFAULT 0,
    last_updated        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_feed_creator ON proj_content_feed (creator_id, published_at DESC);

-- ============================================================
-- PROJECTION: AI Segments (for targeted campaigns)
-- ============================================================

CREATE TABLE proj_subscriber_segments (
    creator_id          UUID NOT NULL,
    subscriber_id       UUID NOT NULL,
    segment_name        VARCHAR(255) NOT NULL,
    score               NUMERIC(5,4),
    churn_probability   NUMERIC(5,4),
    last_predicted      TIMESTAMPTZ,
    PRIMARY KEY (creator_id, subscriber_id, segment_name)
);

CREATE INDEX idx_proj_segments_churn ON proj_subscriber_segments (creator_id, churn_probability DESC);
```

---

## Trade-offs

**Strengths:**
- Complete audit trail of every financial and business event -- essential for disputes, compliance, and debugging.
- Time-travel queries are trivial: replay events up to any point to reconstruct past state.
- The event stream naturally feeds AI/ML pipelines without building separate ETL infrastructure.
- Read models can be independently optimized, rebuilt, or replaced without touching the source of truth.
- Natural fit for webhook delivery: domain events map directly to webhook payloads.

**Weaknesses:**
- Significantly higher complexity for developers unfamiliar with the pattern. Simple profile updates feel over-engineered.
- Eventual consistency between write and read sides requires careful UI design (e.g. "your subscription is being processed").
- Event schema evolution requires discipline: events are immutable, so new versions must coexist with old ones (upcasting).
- Projection rebuilds for large event volumes can take hours. Snapshots mitigate this but add another moving part.
- Harder to perform ad-hoc queries against the event store compared to a normalized relational database.

**Scalability Considerations:**
- The event store grows monotonically. Partition by month and archive old partitions to cold storage.
- Projections can be horizontally scaled as independent read replicas.
- Event publishing can use PostgreSQL LISTEN/NOTIFY for low volumes, or Kafka/NATS for high-throughput scenarios.
- Snapshots should be taken every 100-500 events per aggregate to bound replay time.

**Migration Path:**
- Start with a simpler relational model (Suggestion 1) and introduce event sourcing selectively for the payment/subscription domain where auditability matters most. The content and community domains can remain CRUD-based initially.
- Use the "strangler fig" pattern: wrap existing tables with event-emitting adapters, then gradually replace direct writes with command handlers.
