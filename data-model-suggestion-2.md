# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Dynamic Pricing Engine · Created: 2026-05-19

## Philosophy

This model treats every state change as an immutable event recorded in an append-only event store. The current state of any entity -- a product's price, a rule's configuration, an experiment's status -- is derived by replaying its event stream rather than reading a mutable row. Read-optimised materialised views (projections) are maintained separately for query performance, following the Command Query Responsibility Segregation (CQRS) pattern.

This architecture is inspired by financial ledger systems, the Azure Event Sourcing pattern, and platforms like Competera that need full audit trails of every pricing decision. In dynamic pricing, the ability to answer "what price was this product at 3pm last Tuesday, and what rule and competitor data caused that price?" is not a nice-to-have -- it is a regulatory requirement under the EU Omnibus Directive and a business necessity for debugging AI pricing decisions.

Event sourcing is best suited for teams that need an immutable, tamper-proof audit trail of all pricing activity, want to replay events to train ML models or back-test pricing strategies, and are comfortable with the additional complexity of maintaining separate write and read models.

**Best for:** Compliance-heavy deployments where full auditability, temporal queries, and the ability to replay pricing history for ML training are paramount.

**Trade-offs:**
- Pro: Complete, immutable audit trail of every pricing decision -- satisfies EU Omnibus, GDPR data subject access requests, and internal compliance
- Pro: Temporal queries are natural ("what was the state at time T?") via event replay
- Pro: Event streams serve as training data for RL pricing agents without a separate ETL pipeline
- Pro: New read models can be built by replaying the event store -- no data migration needed
- Pro: Events are the integration contract -- downstream systems subscribe to event types, not table schemas
- Con: Higher storage requirements (events are never deleted)
- Con: Eventual consistency between event store and read models requires careful handling
- Con: More complex application code (command handlers, event handlers, projectors)
- Con: Debugging requires understanding event replay, not just reading a row
- Con: Schema evolution of event types needs versioning strategy

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 GTIN | Product identifier embedded in `ProductRegistered` and all product-related events |
| ISO 4217 | Currency codes in all price-related event payloads |
| ISO 3166-1/2 | Jurisdiction codes in `ChannelConfigured` and compliance events |
| EU Omnibus Directive | `OmnibusReferencePriceCalculated` event type tracks 30-day lowest price computation |
| AsyncAPI 3.x | Event types and schemas documented in AsyncAPI format for webhook/queue consumers |
| RFC 8259 (JSON) | All event payloads serialised as JSON |
| CloudEvents 1.0 | Event envelope follows CloudEvents specification (type, source, time, data) |

---

## Event Store (Write Model)

The event store is the single source of truth. Every state change is recorded as an immutable event.

```sql
-- Core event store: append-only, immutable
CREATE TABLE events (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id UUID NOT NULL,                         -- aggregate root ID (product, strategy, experiment)
    stream_type VARCHAR(50) NOT NULL,                -- 'Product', 'PricingStrategy', 'Experiment', 'Competitor'
    event_type VARCHAR(100) NOT NULL,                -- e.g. 'PriceChanged', 'RuleCreated', 'CompetitorPriceObserved'
    event_version INTEGER NOT NULL,                  -- schema version for this event type (for evolution)
    sequence_number BIGINT NOT NULL,                 -- per-stream ordering
    tenant_id UUID NOT NULL,
    caused_by_event_id UUID,                         -- causal chain: which event triggered this one?
    correlation_id UUID,                             -- groups related events across streams
    actor_id UUID,                                   -- user or system that caused the event
    actor_type VARCHAR(20) NOT NULL DEFAULT 'system', -- 'user', 'system', 'api_client', 'ml_model'
    payload JSONB NOT NULL,                          -- event-specific data
    metadata JSONB NOT NULL DEFAULT '{}',            -- trace IDs, IP addresses, etc.
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(stream_id, sequence_number)
);

-- Partitioned by month for efficient retention management
CREATE INDEX idx_events_stream ON events(stream_id, sequence_number);
CREATE INDEX idx_events_type ON events(event_type, created_at);
CREATE INDEX idx_events_tenant ON events(tenant_id, created_at);
CREATE INDEX idx_events_correlation ON events(correlation_id) WHERE correlation_id IS NOT NULL;
CREATE INDEX idx_events_created ON events(created_at);

-- Snapshot store: periodic snapshots to avoid replaying full event streams
CREATE TABLE snapshots (
    stream_id UUID NOT NULL,
    stream_type VARCHAR(50) NOT NULL,
    sequence_number BIGINT NOT NULL,                 -- snapshot taken at this sequence number
    state JSONB NOT NULL,                            -- serialised aggregate state
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, sequence_number)
);
```

### Event Type Catalogue

Key event types and their payload structures:

```sql
-- Example event payloads (stored in events.payload JSONB column):

-- ProductRegistered
-- {
--   "sku": "WIDGET-001",
--   "gtin": "01onal234567890",
--   "name": "Premium Widget",
--   "brand": "WidgetCo",
--   "cost_price": 12.50,
--   "cost_currency": "USD",
--   "category_path": "/electronics/widgets"
-- }

-- ProductListedOnChannel
-- {
--   "channel_id": "uuid-...",
--   "channel_type": "shopify",
--   "external_product_id": "7891234567",
--   "initial_price": 29.99,
--   "currency": "USD",
--   "map_price": 24.99,
--   "msrp": 34.99
-- }

-- CompetitorPriceObserved
-- {
--   "competitor_id": "uuid-...",
--   "competitor_name": "MegaStore",
--   "product_sku": "WIDGET-001",
--   "price": 27.49,
--   "currency": "USD",
--   "was_in_stock": true,
--   "is_promotional": false,
--   "source": "scrape",
--   "match_confidence": 0.95
-- }

-- PricingRuleCreated
-- {
--   "strategy_id": "uuid-...",
--   "rule_type": "competitor_beat",
--   "competitor_offset_pct": -0.02,
--   "price_floor": 20.00,
--   "scope": {"type": "category", "value": "/electronics/widgets"},
--   "priority": 100
-- }

-- PriceRecommendationGenerated
-- {
--   "channel_product_id": "uuid-...",
--   "current_price": 29.99,
--   "recommended_price": 26.99,
--   "rule_id": "uuid-...",
--   "competitor_prices": [{"competitor": "MegaStore", "price": 27.49}],
--   "elasticity_estimate": -1.45,
--   "predicted_demand_change_pct": 0.12,
--   "omnibus_reference_price": 25.99,
--   "constraints_applied": ["map_floor", "omnibus_floor"]
-- }

-- PriceChanged
-- {
--   "channel_product_id": "uuid-...",
--   "old_price": 29.99,
--   "new_price": 26.99,
--   "currency": "USD",
--   "reason": "rule_applied",
--   "rule_id": "uuid-...",
--   "omnibus_reference_price": 25.99,
--   "approval_status": "auto_approved"
-- }

-- ExperimentStarted
-- {
--   "name": "Widget price sensitivity test",
--   "variants": [
--     {"name": "control", "traffic_weight": 50, "price_modifier_pct": 0},
--     {"name": "variant_a", "traffic_weight": 50, "price_modifier_pct": 0.05}
--   ],
--   "product_scope": {"category": "/electronics/widgets"},
--   "channel_ids": ["uuid-..."]
-- }

-- OmnibusReferencePriceCalculated
-- {
--   "channel_product_id": "uuid-...",
--   "reference_price": 25.99,
--   "lookback_days": 30,
--   "jurisdiction": "EU",
--   "calculation_method": "min_price_in_window"
-- }

-- DemandForecastGenerated
-- {
--   "product_id": "uuid-...",
--   "channel_id": "uuid-...",
--   "forecast_date": "2026-05-20",
--   "horizon_days": 7,
--   "predicted_units": 142.5,
--   "confidence_interval": [128.0, 157.0],
--   "model_version": "xgboost-v3.2",
--   "features": ["price", "seasonality", "competitor_avg", "weather"]
-- }
```

## Read Models (Materialised Projections)

These tables are derived from the event store and can be rebuilt at any time by replaying events.

```sql
-- Current product state (projected from ProductRegistered, ProductUpdated, etc.)
CREATE TABLE rm_products (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    sku VARCHAR(100) NOT NULL,
    gtin VARCHAR(14),
    name VARCHAR(500) NOT NULL,
    brand VARCHAR(255),
    cost_price NUMERIC(12,4),
    cost_currency CHAR(3),
    category_path TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    last_event_sequence BIGINT NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE UNIQUE INDEX idx_rm_products_tenant_sku ON rm_products(tenant_id, sku);

-- Current prices per channel (projected from PriceChanged events)
CREATE TABLE rm_current_prices (
    channel_product_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    product_id UUID NOT NULL,
    channel_id UUID NOT NULL,
    channel_type VARCHAR(50),
    current_price NUMERIC(12,4) NOT NULL,
    currency CHAR(3) NOT NULL,
    map_price NUMERIC(12,4),
    msrp NUMERIC(12,4),
    omnibus_reference_price NUMERIC(12,4),
    last_price_change_at TIMESTAMPTZ,
    last_rule_id UUID,
    last_event_sequence BIGINT NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_prices_tenant ON rm_current_prices(tenant_id);
CREATE INDEX idx_rm_prices_product ON rm_current_prices(product_id);

-- Latest competitor prices (projected from CompetitorPriceObserved)
CREATE TABLE rm_competitor_latest (
    competitor_product_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    competitor_id UUID NOT NULL,
    competitor_name VARCHAR(255),
    product_id UUID NOT NULL,
    latest_price NUMERIC(12,4),
    currency CHAR(3),
    was_in_stock BOOLEAN,
    is_promotional BOOLEAN,
    observed_at TIMESTAMPTZ,
    last_event_sequence BIGINT NOT NULL
);

CREATE INDEX idx_rm_competitor_product ON rm_competitor_latest(product_id);

-- Active pricing strategies (projected from PricingRuleCreated, PricingRuleUpdated, etc.)
CREATE TABLE rm_active_strategies (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    name VARCHAR(255),
    type VARCHAR(50),
    is_active BOOLEAN,
    rules_count INTEGER DEFAULT 0,
    last_event_sequence BIGINT NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

-- Experiment dashboard (projected from ExperimentStarted, ExperimentResultRecorded, etc.)
CREATE TABLE rm_experiments (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    name VARCHAR(255),
    status VARCHAR(20),
    variants JSONB,                                  -- [{name, traffic_weight, conversions, revenue}]
    started_at TIMESTAMPTZ,
    ended_at TIMESTAMPTZ,
    last_event_sequence BIGINT NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

-- Repricing activity feed (projected from PriceRecommendationGenerated, PriceChanged)
CREATE TABLE rm_repricing_feed (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    product_id UUID NOT NULL,
    product_name VARCHAR(500),
    channel_type VARCHAR(50),
    old_price NUMERIC(12,4),
    new_price NUMERIC(12,4),
    currency CHAR(3),
    reason VARCHAR(50),
    rule_name VARCHAR(255),
    changed_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_repricing_tenant ON rm_repricing_feed(tenant_id, changed_at DESC);

-- Daily sales aggregation (projected from SaleRecorded events)
CREATE TABLE rm_daily_sales (
    tenant_id UUID NOT NULL,
    product_id UUID NOT NULL,
    channel_id UUID NOT NULL,
    sale_date DATE NOT NULL,
    units_sold INTEGER NOT NULL DEFAULT 0,
    total_revenue NUMERIC(14,4) NOT NULL DEFAULT 0,
    total_margin NUMERIC(14,4) NOT NULL DEFAULT 0,
    avg_selling_price NUMERIC(12,4),
    last_event_sequence BIGINT NOT NULL,
    PRIMARY KEY (tenant_id, product_id, channel_id, sale_date)
);
```

## Projection Management

```sql
-- Tracks the position of each projector in the event stream
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,        -- e.g. 'rm_current_prices', 'rm_competitor_latest'
    last_processed_event_id UUID,
    last_processed_at TIMESTAMPTZ,
    status VARCHAR(20) NOT NULL DEFAULT 'running',   -- running, paused, rebuilding, failed
    error_message TEXT,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Dead letter queue for events that failed processing
CREATE TABLE projection_dead_letters (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    projection_name VARCHAR(100) NOT NULL,
    event_id UUID NOT NULL,
    error_message TEXT NOT NULL,
    retry_count INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Subscription & Outbox

```sql
-- Outbox pattern: events waiting to be published to external consumers
CREATE TABLE event_outbox (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id UUID NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    tenant_id UUID NOT NULL,
    destination VARCHAR(255) NOT NULL,               -- webhook URL, queue name, etc.
    payload JSONB NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',   -- pending, sent, failed
    attempt_count INTEGER NOT NULL DEFAULT 0,
    last_attempt_at TIMESTAMPTZ,
    next_retry_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_outbox_pending ON event_outbox(next_retry_at)
    WHERE status IN ('pending', 'failed');

-- Event subscriptions (which tenants want which event types delivered where)
CREATE TABLE event_subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    event_type_pattern VARCHAR(100) NOT NULL,        -- e.g. 'PriceChanged', 'Competitor*', '*'
    delivery_type VARCHAR(20) NOT NULL,              -- webhook, sqs, kafka
    delivery_target VARCHAR(1000) NOT NULL,
    secret_hash VARCHAR(255),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example Queries

### Reconstruct product price at a specific point in time

```sql
-- What was the price of product X on channel Y at 3pm on May 15?
SELECT payload->>'new_price' AS price,
       payload->>'currency' AS currency,
       payload->>'rule_id' AS rule_applied,
       created_at
FROM events
WHERE stream_type = 'Product'
  AND stream_id = 'product-uuid-here'
  AND event_type = 'PriceChanged'
  AND (payload->>'channel_product_id') = 'channel-product-uuid-here'
  AND created_at <= '2026-05-15 15:00:00+00'
ORDER BY sequence_number DESC
LIMIT 1;
```

### Full audit trail for a pricing decision

```sql
-- Show the causal chain: competitor price observed -> recommendation -> price changed
WITH RECURSIVE causal_chain AS (
    SELECT event_id, event_type, payload, caused_by_event_id, created_at, 0 AS depth
    FROM events
    WHERE event_id = 'price-changed-event-uuid'

    UNION ALL

    SELECT e.event_id, e.event_type, e.payload, e.caused_by_event_id, e.created_at, cc.depth + 1
    FROM events e
    JOIN causal_chain cc ON e.event_id = cc.caused_by_event_id
    WHERE cc.depth < 10
)
SELECT event_type, payload, created_at, depth
FROM causal_chain
ORDER BY depth DESC;
```

### Replay events to compute Omnibus reference price

```sql
-- Calculate the lowest price in the last 30 days for Omnibus compliance
SELECT MIN((payload->>'new_price')::numeric) AS omnibus_reference_price
FROM events
WHERE stream_type = 'Product'
  AND stream_id = 'product-uuid-here'
  AND event_type = 'PriceChanged'
  AND (payload->>'channel_product_id') = 'channel-product-uuid-here'
  AND created_at >= now() - INTERVAL '30 days';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | events (source of truth), snapshots |
| Read Models | 7 | rm_products, rm_current_prices, rm_competitor_latest, rm_active_strategies, rm_experiments, rm_repricing_feed, rm_daily_sales |
| Projection Infrastructure | 2 | projection_checkpoints, projection_dead_letters |
| Event Delivery | 2 | event_outbox, event_subscriptions |
| **Total** | **13** | Read models are disposable and rebuildable |

---

## Key Design Decisions

1. **Single event table, not per-aggregate** -- all events live in one `events` table, partitioned by `created_at` month. This simplifies cross-aggregate queries (e.g., "all events for tenant X in the last hour") and avoids the complexity of managing dozens of event tables.

2. **Causal event chaining via `caused_by_event_id`** -- when a `CompetitorPriceObserved` event triggers a `PriceRecommendationGenerated` event which triggers a `PriceChanged` event, the chain is explicit. This enables full "why did this price change?" audit trails.

3. **Correlation IDs group related events** -- a single repricing run generates a `correlation_id` shared across all events it produces, enabling efficient querying of "everything that happened in this repricing batch."

4. **Event versioning for schema evolution** -- `event_version` allows the system to handle old and new event payload formats simultaneously. Event upcasters transform old versions to current format during replay.

5. **Snapshots prevent unbounded replay** -- for entities with long event histories (e.g., a product with thousands of price changes), periodic snapshots avoid replaying from the beginning.

6. **Read models are explicitly disposable** -- prefixed with `rm_` to signal they can be dropped and rebuilt from the event store. No application logic should depend on read model data being the authoritative source.

7. **Outbox pattern for reliable event delivery** -- events are published to external consumers via the outbox table, ensuring at-least-once delivery even if webhook endpoints are temporarily unavailable.

8. **CloudEvents-compatible envelope** -- event metadata (type, source, time, data) aligns with the CloudEvents 1.0 specification, making it straightforward to publish events to cloud-native event brokers (EventBridge, Cloud Events).

9. **RL training data is free** -- the event stream of `PriceChanged`, `SaleRecorded`, `CompetitorPriceObserved` events is exactly the state-action-reward data needed to train reinforcement learning pricing agents. No separate data pipeline is needed.

10. **Omnibus compliance via event replay** -- rather than maintaining a separate 30-day price tracking table, the system replays `PriceChanged` events to compute the reference price on demand or via a scheduled projection, ensuring the calculation is always consistent with the authoritative event history.
