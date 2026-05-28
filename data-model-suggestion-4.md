# Data Model Suggestion 4: Time-Series / Analytics-First

> Project: Dynamic Pricing Engine · Created: 2026-05-19

## Philosophy

This model is designed around the premise that a dynamic pricing engine is fundamentally a time-series system: competitor prices are observed over time, demand fluctuates over time, prices change over time, and ML models need temporally-ordered training data. Rather than treating time-series data as a secondary concern (append-only tables with timestamp indexes), this architecture makes time-series storage the primary pattern using PostgreSQL with the TimescaleDB extension.

Core operational tables (products, channels, strategies) remain standard relational, but all high-volume, time-indexed data -- competitor price observations, own price history, sales transactions, demand forecasts, inventory snapshots, and RL agent episodes -- are stored in TimescaleDB hypertables. These automatically partition by time, enable native continuous aggregates (materialised rollups that refresh incrementally), and provide built-in compression for historical data that reduces storage by 90%+ while keeping it queryable.

This approach is inspired by financial trading platforms that store tick-by-tick price data, energy companies that model real-time demand and pricing, and the TimescaleDB reference architecture for IoT/telemetry. In dynamic pricing, the ability to efficiently query "what was the average competitor price for category X over the last 90 days, aggregated by week, broken down by competitor?" is the core analytical workload.

**Best for:** High-frequency pricing environments (marketplace repricing, airline/hotel yield management) where millions of price observations per day must be stored, queried, and fed into ML training pipelines efficiently.

**Trade-offs:**
- Pro: Orders-of-magnitude faster time-range queries than standard PostgreSQL tables
- Pro: Automatic partitioning -- no manual partition management
- Pro: Continuous aggregates provide pre-computed rollups (hourly, daily, weekly) without batch ETL
- Pro: Built-in compression reduces storage costs by 90%+ for historical data
- Pro: Native time-series functions (time_bucket, first, last, interpolate) simplify analytics queries
- Pro: Same PostgreSQL -- standard SQL, same tooling, same ORMs
- Con: Requires TimescaleDB extension (not available on all managed PostgreSQL providers)
- Con: Hypertables have restrictions (no foreign keys referencing hypertables, limited ALTER TABLE)
- Con: Continuous aggregates add operational complexity (refresh policies, invalidation)
- Con: Not a good fit for low-volume pricing (< 10k price observations/day)
- Con: Chunk management (retention, compression) requires understanding TimescaleDB internals

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 GTIN | Product identifier in dimension tables, used as join key into time-series facts |
| ISO 4217 | Currency codes in all price measurement hypertables |
| ISO 3166-1/2 | Jurisdiction dimension table for region-segmented analytics |
| EU Omnibus Directive | Continuous aggregate computes 30-day minimum price automatically |
| ISO 8601 | All timestamps stored as TIMESTAMPTZ; time_bucket uses ISO-compliant intervals |
| OpenAPI 3.1 | Analytics API endpoints documented with time-range query parameters |

---

## Dimension Tables (Standard PostgreSQL)

These are low-volume reference tables with standard relational design.

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    plan VARCHAR(50) NOT NULL DEFAULT 'free',
    base_currency CHAR(3) NOT NULL DEFAULT 'USD',
    timezone VARCHAR(50) NOT NULL DEFAULT 'UTC',
    settings JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL DEFAULT 'viewer',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    sku VARCHAR(100) NOT NULL,
    gtin VARCHAR(14),
    name VARCHAR(500) NOT NULL,
    brand VARCHAR(255),
    category_path TEXT,
    cost_price NUMERIC(12,4),
    cost_currency CHAR(3) DEFAULT 'USD',
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    attributes JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, sku)
);

CREATE INDEX idx_products_tenant ON products(tenant_id);
CREATE INDEX idx_products_gtin ON products(gtin) WHERE gtin IS NOT NULL;

CREATE TABLE channels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(50) NOT NULL,
    config JSONB NOT NULL DEFAULT '{}',
    jurisdictions VARCHAR(10)[] DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE channel_products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    channel_id UUID NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    external_ids JSONB NOT NULL DEFAULT '{}',
    current_price NUMERIC(12,4),
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    map_price NUMERIC(12,4),
    msrp NUMERIC(12,4),
    is_listed BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(channel_id, product_id)
);

CREATE TABLE competitors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    website_url VARCHAR(500),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE competitor_product_matches (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    competitor_id UUID NOT NULL REFERENCES competitors(id) ON DELETE CASCADE,
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    match_confidence NUMERIC(3,2) NOT NULL DEFAULT 1.00,
    external_url VARCHAR(1000),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(competitor_id, product_id)
);

CREATE TABLE pricing_strategies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(50) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT false,
    priority INTEGER NOT NULL DEFAULT 100,
    rules JSONB NOT NULL DEFAULT '[]',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ml_models (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    model_type VARCHAR(50) NOT NULL,
    version VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'training',
    config JSONB NOT NULL DEFAULT '{}',
    metrics JSONB NOT NULL DEFAULT '{}',
    artifact_path VARCHAR(500),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Fact Hypertables (TimescaleDB)

These are the high-volume time-series tables that benefit from automatic partitioning, compression, and continuous aggregates.

```sql
-- Enable TimescaleDB
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- ============================================================
-- COMPETITOR PRICE OBSERVATIONS
-- Expected volume: 10k-10M rows/day depending on catalogue size
-- ============================================================
CREATE TABLE ts_competitor_prices (
    observed_at TIMESTAMPTZ NOT NULL,
    tenant_id UUID NOT NULL,
    competitor_id UUID NOT NULL,
    product_id UUID NOT NULL,
    match_id UUID NOT NULL,
    price NUMERIC(12,4) NOT NULL,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    price_converted NUMERIC(12,4),                   -- normalised to tenant base currency
    was_in_stock BOOLEAN,
    is_promotional BOOLEAN NOT NULL DEFAULT false,
    shipping_cost NUMERIC(12,4),
    source VARCHAR(20) DEFAULT 'scrape'              -- scrape, api, manual
);

SELECT create_hypertable('ts_competitor_prices', 'observed_at',
    chunk_time_interval => INTERVAL '1 day');

CREATE INDEX idx_ts_comp_prices_product ON ts_competitor_prices(product_id, observed_at DESC);
CREATE INDEX idx_ts_comp_prices_competitor ON ts_competitor_prices(competitor_id, observed_at DESC);
CREATE INDEX idx_ts_comp_prices_tenant ON ts_competitor_prices(tenant_id, observed_at DESC);

-- Compress chunks older than 7 days (90%+ storage reduction)
ALTER TABLE ts_competitor_prices SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'tenant_id, product_id, competitor_id',
    timescaledb.compress_orderby = 'observed_at DESC'
);
SELECT add_compression_policy('ts_competitor_prices', INTERVAL '7 days');

-- Retain detailed data for 1 year, then drop
SELECT add_retention_policy('ts_competitor_prices', INTERVAL '365 days');


-- ============================================================
-- OWN PRICE CHANGES
-- Every price change across all channels
-- ============================================================
CREATE TABLE ts_price_changes (
    changed_at TIMESTAMPTZ NOT NULL,
    tenant_id UUID NOT NULL,
    channel_product_id UUID NOT NULL,
    product_id UUID NOT NULL,
    channel_id UUID NOT NULL,
    old_price NUMERIC(12,4),
    new_price NUMERIC(12,4) NOT NULL,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    reason VARCHAR(50) NOT NULL,                     -- rule_applied, manual, experiment, sync, markdown
    rule_id UUID,
    strategy_id UUID,
    experiment_id UUID,
    changed_by UUID                                  -- user ID or NULL for system
);

SELECT create_hypertable('ts_price_changes', 'changed_at',
    chunk_time_interval => INTERVAL '7 days');

CREATE INDEX idx_ts_price_changes_product ON ts_price_changes(product_id, changed_at DESC);
CREATE INDEX idx_ts_price_changes_channel_product ON ts_price_changes(channel_product_id, changed_at DESC);

ALTER TABLE ts_price_changes SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'tenant_id, product_id',
    timescaledb.compress_orderby = 'changed_at DESC'
);
SELECT add_compression_policy('ts_price_changes', INTERVAL '30 days');


-- ============================================================
-- SALES TRANSACTIONS
-- Individual sale events for demand analysis
-- ============================================================
CREATE TABLE ts_sales (
    transaction_at TIMESTAMPTZ NOT NULL,
    tenant_id UUID NOT NULL,
    product_id UUID NOT NULL,
    channel_id UUID NOT NULL,
    quantity INTEGER NOT NULL,
    unit_price NUMERIC(12,4) NOT NULL,
    total_price NUMERIC(14,4) NOT NULL,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    cost_price NUMERIC(12,4),
    margin NUMERIC(12,4),
    discount_amount NUMERIC(12,4) DEFAULT 0,
    customer_segment VARCHAR(100),
    external_order_id VARCHAR(255)
);

SELECT create_hypertable('ts_sales', 'transaction_at',
    chunk_time_interval => INTERVAL '7 days');

CREATE INDEX idx_ts_sales_product ON ts_sales(product_id, transaction_at DESC);
CREATE INDEX idx_ts_sales_tenant ON ts_sales(tenant_id, transaction_at DESC);

ALTER TABLE ts_sales SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'tenant_id, product_id, channel_id',
    timescaledb.compress_orderby = 'transaction_at DESC'
);
SELECT add_compression_policy('ts_sales', INTERVAL '30 days');


-- ============================================================
-- DEMAND FORECASTS (time-series of predictions)
-- ============================================================
CREATE TABLE ts_demand_forecasts (
    generated_at TIMESTAMPTZ NOT NULL,
    tenant_id UUID NOT NULL,
    product_id UUID NOT NULL,
    channel_id UUID,
    model_id UUID,
    forecast_date DATE NOT NULL,
    horizon_days INTEGER NOT NULL DEFAULT 7,
    predicted_units NUMERIC(12,2) NOT NULL,
    predicted_revenue NUMERIC(14,4),
    confidence_lower NUMERIC(12,2),
    confidence_upper NUMERIC(12,2),
    price_at_forecast NUMERIC(12,4),
    model_version VARCHAR(50)
);

SELECT create_hypertable('ts_demand_forecasts', 'generated_at',
    chunk_time_interval => INTERVAL '7 days');

CREATE INDEX idx_ts_forecasts_product ON ts_demand_forecasts(product_id, generated_at DESC);

ALTER TABLE ts_demand_forecasts SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'tenant_id, product_id',
    timescaledb.compress_orderby = 'generated_at DESC'
);
SELECT add_compression_policy('ts_demand_forecasts', INTERVAL '30 days');


-- ============================================================
-- INVENTORY SNAPSHOTS (periodic snapshots for trend analysis)
-- ============================================================
CREATE TABLE ts_inventory_snapshots (
    snapshot_at TIMESTAMPTZ NOT NULL,
    tenant_id UUID NOT NULL,
    product_id UUID NOT NULL,
    channel_id UUID,
    quantity_on_hand INTEGER NOT NULL,
    quantity_reserved INTEGER NOT NULL DEFAULT 0,
    quantity_available INTEGER NOT NULL,
    days_of_supply NUMERIC(6,1),
    location_code VARCHAR(50)
);

SELECT create_hypertable('ts_inventory_snapshots', 'snapshot_at',
    chunk_time_interval => INTERVAL '1 day');

CREATE INDEX idx_ts_inventory_product ON ts_inventory_snapshots(product_id, snapshot_at DESC);

ALTER TABLE ts_inventory_snapshots SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'tenant_id, product_id',
    timescaledb.compress_orderby = 'snapshot_at DESC'
);
SELECT add_compression_policy('ts_inventory_snapshots', INTERVAL '7 days');


-- ============================================================
-- RL AGENT EPISODES (state-action-reward for training)
-- ============================================================
CREATE TABLE ts_rl_episodes (
    step_at TIMESTAMPTZ NOT NULL,
    tenant_id UUID NOT NULL,
    agent_id UUID NOT NULL,                          -- ML model ID
    product_id UUID NOT NULL,
    channel_id UUID,
    episode_id UUID NOT NULL,
    step_number INTEGER NOT NULL,
    state JSONB NOT NULL,
    -- state example:
    -- {
    --   "current_price": 29.99,
    --   "cost_price": 12.50,
    --   "competitor_min_price": 27.49,
    --   "competitor_avg_price": 30.15,
    --   "inventory_days": 45,
    --   "demand_trend": 0.03,
    --   "day_of_week": 1,
    --   "is_holiday": false
    -- }
    action JSONB NOT NULL,
    -- action example:
    -- {"price_adjustment_pct": -0.02, "new_price": 29.39}
    reward NUMERIC(14,4) NOT NULL,                   -- typically: margin earned
    cumulative_reward NUMERIC(14,4),
    is_terminal BOOLEAN NOT NULL DEFAULT false
);

SELECT create_hypertable('ts_rl_episodes', 'step_at',
    chunk_time_interval => INTERVAL '1 day');

CREATE INDEX idx_ts_rl_agent ON ts_rl_episodes(agent_id, step_at DESC);
CREATE INDEX idx_ts_rl_episode ON ts_rl_episodes(episode_id, step_number);

ALTER TABLE ts_rl_episodes SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'tenant_id, agent_id',
    timescaledb.compress_orderby = 'step_at DESC'
);
SELECT add_compression_policy('ts_rl_episodes', INTERVAL '14 days');


-- ============================================================
-- REPRICING EVENTS (log of every repricing evaluation)
-- ============================================================
CREATE TABLE ts_repricing_events (
    evaluated_at TIMESTAMPTZ NOT NULL,
    tenant_id UUID NOT NULL,
    product_id UUID NOT NULL,
    channel_product_id UUID NOT NULL,
    strategy_id UUID,
    rule_id UUID,
    current_price NUMERIC(12,4),
    recommended_price NUMERIC(12,4),
    final_price NUMERIC(12,4),
    was_changed BOOLEAN NOT NULL DEFAULT false,
    reason VARCHAR(50),                              -- rule_applied, no_change_needed, constrained, rejected
    constraints_applied TEXT[],                       -- e.g. {'map_floor', 'omnibus_floor', 'max_change_pct'}
    competitor_trigger_price NUMERIC(12,4),
    elasticity_used NUMERIC(8,4),
    demand_forecast_units NUMERIC(12,2)
);

SELECT create_hypertable('ts_repricing_events', 'evaluated_at',
    chunk_time_interval => INTERVAL '1 day');

CREATE INDEX idx_ts_repricing_product ON ts_repricing_events(product_id, evaluated_at DESC);
CREATE INDEX idx_ts_repricing_changed ON ts_repricing_events(tenant_id, evaluated_at DESC)
    WHERE was_changed = true;

ALTER TABLE ts_repricing_events SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'tenant_id, product_id',
    timescaledb.compress_orderby = 'evaluated_at DESC'
);
SELECT add_compression_policy('ts_repricing_events', INTERVAL '7 days');
```

## Continuous Aggregates

These are materialised views that TimescaleDB refreshes incrementally, providing pre-computed rollups without batch ETL.

```sql
-- ============================================================
-- HOURLY COMPETITOR PRICE SUMMARY
-- ============================================================
CREATE MATERIALIZED VIEW cagg_competitor_prices_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', observed_at) AS bucket,
    tenant_id,
    product_id,
    competitor_id,
    AVG(price_converted) AS avg_price,
    MIN(price_converted) AS min_price,
    MAX(price_converted) AS max_price,
    last(price_converted, observed_at) AS latest_price,
    last(was_in_stock, observed_at) AS latest_in_stock,
    COUNT(*) AS observation_count
FROM ts_competitor_prices
GROUP BY bucket, tenant_id, product_id, competitor_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('cagg_competitor_prices_hourly',
    start_offset => INTERVAL '3 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');

-- ============================================================
-- DAILY SALES SUMMARY
-- ============================================================
CREATE MATERIALIZED VIEW cagg_sales_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', transaction_at) AS bucket,
    tenant_id,
    product_id,
    channel_id,
    SUM(quantity) AS units_sold,
    SUM(total_price) AS gross_revenue,
    SUM(margin) AS total_margin,
    AVG(unit_price) AS avg_selling_price,
    COUNT(*) AS transaction_count
FROM ts_sales
GROUP BY bucket, tenant_id, product_id, channel_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('cagg_sales_daily',
    start_offset => INTERVAL '3 days',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');

-- ============================================================
-- DAILY PRICE CHANGE SUMMARY
-- ============================================================
CREATE MATERIALIZED VIEW cagg_price_changes_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', changed_at) AS bucket,
    tenant_id,
    product_id,
    COUNT(*) AS change_count,
    AVG(new_price) AS avg_price,
    MIN(new_price) AS min_price,
    MAX(new_price) AS max_price,
    last(new_price, changed_at) AS closing_price,
    first(new_price, changed_at) AS opening_price
FROM ts_price_changes
GROUP BY bucket, tenant_id, product_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('cagg_price_changes_daily',
    start_offset => INTERVAL '3 days',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');

-- ============================================================
-- OMNIBUS REFERENCE PRICE (30-day rolling minimum)
-- Used for EU Omnibus Directive compliance
-- ============================================================
CREATE MATERIALIZED VIEW cagg_omnibus_reference
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', changed_at) AS bucket,
    tenant_id,
    channel_product_id,
    product_id,
    MIN(new_price) AS daily_min_price
FROM ts_price_changes
GROUP BY bucket, tenant_id, channel_product_id, product_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('cagg_omnibus_reference',
    start_offset => INTERVAL '32 days',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');

-- Query to get current Omnibus reference price:
-- SELECT MIN(daily_min_price) AS omnibus_reference_price
-- FROM cagg_omnibus_reference
-- WHERE channel_product_id = 'uuid-here'
--   AND bucket >= now() - INTERVAL '30 days';


-- ============================================================
-- WEEKLY REPRICING EFFECTIVENESS
-- ============================================================
CREATE MATERIALIZED VIEW cagg_repricing_weekly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('7 days', evaluated_at) AS bucket,
    tenant_id,
    strategy_id,
    COUNT(*) AS total_evaluations,
    COUNT(*) FILTER (WHERE was_changed) AS prices_changed,
    AVG(CASE WHEN was_changed THEN final_price - current_price END) AS avg_price_delta,
    COUNT(*) FILTER (WHERE 'map_floor' = ANY(constraints_applied)) AS map_constrained,
    COUNT(*) FILTER (WHERE 'omnibus_floor' = ANY(constraints_applied)) AS omnibus_constrained
FROM ts_repricing_events
GROUP BY bucket, tenant_id, strategy_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('cagg_repricing_weekly',
    start_offset => INTERVAL '14 days',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day');
```

## Experiments (Standard Relational)

```sql
CREATE TABLE experiments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    hypothesis TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'draft',
    config JSONB NOT NULL DEFAULT '{}',
    results JSONB,
    starts_at TIMESTAMPTZ,
    ends_at TIMESTAMPTZ,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Experiment observations stored in time-series
CREATE TABLE ts_experiment_observations (
    observed_at TIMESTAMPTZ NOT NULL,
    tenant_id UUID NOT NULL,
    experiment_id UUID NOT NULL,
    variant_name VARCHAR(100) NOT NULL,
    product_id UUID NOT NULL,
    channel_id UUID,
    event_type VARCHAR(50) NOT NULL,                 -- impression, click, add_to_cart, purchase
    quantity INTEGER DEFAULT 1,
    revenue NUMERIC(14,4),
    margin NUMERIC(14,4)
);

SELECT create_hypertable('ts_experiment_observations', 'observed_at',
    chunk_time_interval => INTERVAL '1 day');

CREATE INDEX idx_ts_exp_experiment ON ts_experiment_observations(experiment_id, observed_at DESC);
```

---

## Example Queries

### Competitor price trend over 90 days (weekly buckets)

```sql
SELECT
    time_bucket('7 days', bucket) AS week,
    competitor_id,
    AVG(avg_price) AS weekly_avg_price,
    MIN(min_price) AS weekly_min,
    MAX(max_price) AS weekly_max
FROM cagg_competitor_prices_hourly
WHERE product_id = 'product-uuid-here'
  AND tenant_id = 'tenant-uuid-here'
  AND bucket >= now() - INTERVAL '90 days'
GROUP BY week, competitor_id
ORDER BY week;
```

### Price-demand correlation for elasticity estimation

```sql
-- Join own price changes with sales data to estimate elasticity
WITH price_periods AS (
    SELECT
        changed_at AS period_start,
        LEAD(changed_at) OVER (ORDER BY changed_at) AS period_end,
        new_price
    FROM ts_price_changes
    WHERE product_id = 'product-uuid-here'
      AND channel_id = 'channel-uuid-here'
    ORDER BY changed_at
)
SELECT
    pp.new_price,
    SUM(s.quantity) AS units_sold,
    COUNT(DISTINCT time_bucket('1 day', s.transaction_at)) AS days_in_period,
    SUM(s.quantity)::numeric / COUNT(DISTINCT time_bucket('1 day', s.transaction_at)) AS daily_avg_units
FROM price_periods pp
JOIN ts_sales s ON s.product_id = 'product-uuid-here'
    AND s.transaction_at >= pp.period_start
    AND s.transaction_at < COALESCE(pp.period_end, now())
GROUP BY pp.new_price
ORDER BY pp.new_price;
```

### RL training data extraction

```sql
-- Extract state-action-reward tuples for RL model training
SELECT
    step_at,
    state,
    action,
    reward,
    cumulative_reward,
    LEAD(state) OVER (PARTITION BY episode_id ORDER BY step_number) AS next_state,
    is_terminal
FROM ts_rl_episodes
WHERE agent_id = 'agent-uuid-here'
  AND step_at >= now() - INTERVAL '30 days'
ORDER BY episode_id, step_number;
```

### Omnibus Directive compliance check

```sql
-- Get the 30-day lowest price for Omnibus compliance
SELECT
    cp.id AS channel_product_id,
    p.sku,
    cp.current_price,
    MIN(o.daily_min_price) AS omnibus_reference_price,
    CASE
        WHEN cp.current_price < MIN(o.daily_min_price)
        THEN 'VIOLATION: price below 30-day reference'
        ELSE 'COMPLIANT'
    END AS compliance_status
FROM channel_products cp
JOIN products p ON cp.product_id = p.id
JOIN channels c ON cp.channel_id = c.id
LEFT JOIN cagg_omnibus_reference o
    ON o.channel_product_id = cp.id
    AND o.bucket >= now() - INTERVAL '30 days'
WHERE c.jurisdictions && ARRAY['DE', 'FR', 'IT', 'ES', 'NL']
  AND cp.is_listed = true
GROUP BY cp.id, p.sku, cp.current_price;
```

---

## Table Count Summary

| Category | Tables | Type | Notes |
|----------|--------|------|-------|
| Tenancy & Auth | 2 | Relational | tenants, users |
| Product Catalogue | 2 | Relational | products, channel_products |
| Channels | 1 | Relational | channels |
| Competitors | 2 | Relational | competitors, competitor_product_matches |
| Pricing Strategies | 1 | Relational | pricing_strategies (rules as JSONB) |
| ML Models | 1 | Relational | ml_models |
| Experiments | 1 | Relational | experiments |
| Competitor Prices | 1 | Hypertable | ts_competitor_prices |
| Price Changes | 1 | Hypertable | ts_price_changes |
| Sales | 1 | Hypertable | ts_sales |
| Forecasts | 1 | Hypertable | ts_demand_forecasts |
| Inventory | 1 | Hypertable | ts_inventory_snapshots |
| RL Episodes | 1 | Hypertable | ts_rl_episodes |
| Repricing | 1 | Hypertable | ts_repricing_events |
| Experiment Data | 1 | Hypertable | ts_experiment_observations |
| Continuous Aggregates | 5 | Mat. View | hourly competitor, daily sales, daily prices, omnibus, weekly repricing |
| **Total** | **18 + 5 views** | | 8 hypertables, 10 relational, 5 continuous aggregates |

---

## Key Design Decisions

1. **Hypertables for all high-volume time-series data** -- competitor prices, own price changes, sales, forecasts, inventory snapshots, RL episodes, and repricing events are all stored in TimescaleDB hypertables with automatic time-based partitioning. This is the single most impactful architectural choice.

2. **Compression policies per hypertable** -- each hypertable has a compression policy tuned to its query pattern. Competitor prices compress after 7 days (frequently queried recent data), price changes after 30 days (Omnibus lookback window), RL episodes after 14 days (training data extraction window).

3. **Retention policies for cost management** -- detailed competitor price observations are retained for 1 year; older data is dropped but continuous aggregates (hourly/daily rollups) are retained indefinitely. This provides 90-day detail + multi-year trends.

4. **Continuous aggregates replace batch ETL** -- instead of nightly batch jobs to compute daily sales summaries or weekly competitor price trends, TimescaleDB continuously materialises these aggregates incrementally. The `cagg_omnibus_reference` aggregate automatically tracks the 30-day minimum price required by the EU Omnibus Directive.

5. **No foreign keys on hypertables** -- TimescaleDB hypertables do not support being referenced by foreign keys. Join integrity is maintained at the application level. Dimension table IDs (product_id, channel_id) are stored as UUID columns without FK constraints.

6. **Segment-by columns for compression** -- compression is configured with `segmentby = 'tenant_id, product_id'` (or similar), meaning decompression during queries only reads the relevant segments. This ensures that tenant-scoped queries remain fast even on compressed data.

7. **RL episode storage as time-series** -- storing reinforcement learning state-action-reward tuples as time-series data enables efficient extraction of training windows ("last 30 days of experience") and correlation with actual sales outcomes in the same time range.

8. **Dimension tables remain standard relational** -- products, channels, competitors, strategies, and experiments are low-volume, frequently-joined tables that benefit from foreign keys and standard indexes. Only the high-volume measurement data uses hypertables.

9. **`ts_` prefix convention** -- all hypertables are prefixed with `ts_` to distinguish them from standard relational tables, signaling that they have different constraints (no FKs to them, different ALTER TABLE rules) and query patterns.

10. **Omnibus compliance via continuous aggregate** -- rather than computing the 30-day minimum price on every API request or storing it as a denormalized column, the `cagg_omnibus_reference` continuous aggregate maintains daily minimum prices automatically. A simple `MIN()` query over the last 30 days of the aggregate yields the reference price in sub-millisecond time.
