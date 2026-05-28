# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Dynamic Pricing Engine · Created: 2026-05-19

## Philosophy

This model follows classical relational database design principles: every real-world concept gets its own table, foreign keys enforce referential integrity, and junction tables handle many-to-many relationships. The schema is fully normalized to third normal form (3NF), eliminating data redundancy and ensuring consistency through database constraints rather than application logic.

This approach mirrors how enterprise pricing platforms like Pricefx and Revionics structure their data internally -- separate entities for products, channels, competitors, pricing rules, elasticity models, and experiments. The schema is designed for complex cross-entity queries: "show me the margin impact of all active pricing rules on products in category X across channels Y and Z where competitor prices dropped more than 5% this week."

The normalized design is best suited for teams building a robust, long-lived platform where data integrity is paramount, regulatory compliance (EU Omnibus Directive, MAP/MSRP) requires precise field-level tracking, and the query patterns span many entity types simultaneously.

**Best for:** Enterprise-grade deployments where data integrity, compliance auditability, and complex cross-entity reporting are the primary concerns.

**Trade-offs:**
- Pro: Strong referential integrity -- the database prevents orphaned records and inconsistent state
- Pro: Complex analytical queries are natural (JOINs across normalized tables)
- Pro: Standards-aligned field naming (ISO 4217 currencies, GS1 GTINs, ISO 3166 jurisdictions)
- Pro: Schema is self-documenting -- table structure reveals the domain model
- Con: Higher table count (~45+ tables) increases migration complexity
- Con: Write-heavy workloads (e.g., ingesting millions of competitor prices per hour) may suffer from FK constraint overhead
- Con: Adding jurisdiction-specific or channel-specific fields requires schema migrations
- Con: JOIN-heavy queries can be slower than denormalized alternatives at very high scale

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 GTIN | `products.gtin` stores the Global Trade Item Number as the universal product identifier for cross-system matching |
| ISO 4217 | All monetary columns pair with a `currency_code CHAR(3)` column using ISO 4217 codes (USD, EUR, GBP) |
| ISO 3166-1/2 | `jurisdictions.iso_code` stores country and subdivision codes for region-specific pricing rules |
| ISO 21041:2018 | Unit pricing fields (`unit_price`, `unit_of_measure`) follow ISO 21041 guidance |
| EU Omnibus Directive | `price_history` table tracks 30-day lowest price via `omnibus_reference_price` column |
| GS1 GDSN | Product master data fields aligned with GDSN attribute set for supplier data synchronization |
| OpenAPI 3.1 | API resource structure maps 1:1 to table names for clean REST design |
| OAuth 2.0 / JWT | `api_clients` and `api_tokens` tables support OAuth 2.0 client credentials flow |

---

## Core Domain: Tenancy & Authentication

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    plan VARCHAR(50) NOT NULL DEFAULT 'free',       -- free, starter, professional, enterprise
    settings JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255),
    role VARCHAR(50) NOT NULL DEFAULT 'viewer',      -- admin, manager, analyst, viewer
    is_active BOOLEAN NOT NULL DEFAULT true,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);

CREATE TABLE api_clients (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    client_id VARCHAR(64) NOT NULL UNIQUE,
    client_secret_hash VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    scopes TEXT[] NOT NULL DEFAULT '{}',              -- e.g. {'prices:read', 'rules:write'}
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Product Catalogue

```sql
CREATE TABLE product_categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    parent_id UUID REFERENCES product_categories(id) ON DELETE SET NULL,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL,
    depth INTEGER NOT NULL DEFAULT 0,
    path TEXT NOT NULL DEFAULT '',                    -- materialised path e.g. '/electronics/phones/smartphones'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, slug)
);

CREATE INDEX idx_product_categories_parent ON product_categories(parent_id);
CREATE INDEX idx_product_categories_path ON product_categories(path text_pattern_ops);

CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    category_id UUID REFERENCES product_categories(id) ON DELETE SET NULL,
    sku VARCHAR(100) NOT NULL,
    gtin VARCHAR(14),                                -- GS1 Global Trade Item Number
    name VARCHAR(500) NOT NULL,
    brand VARCHAR(255),
    description TEXT,
    cost_price NUMERIC(12,4),                        -- cost of goods
    cost_currency CHAR(3) DEFAULT 'USD',             -- ISO 4217
    unit_of_measure VARCHAR(20),                     -- e.g. 'each', 'kg', 'litre' (ISO 21041)
    unit_quantity NUMERIC(10,4) DEFAULT 1,
    weight_kg NUMERIC(10,4),
    is_perishable BOOLEAN NOT NULL DEFAULT false,
    shelf_life_days INTEGER,
    status VARCHAR(20) NOT NULL DEFAULT 'active',    -- active, discontinued, seasonal
    tags TEXT[] DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, sku)
);

CREATE INDEX idx_products_tenant ON products(tenant_id);
CREATE INDEX idx_products_gtin ON products(gtin) WHERE gtin IS NOT NULL;
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_status ON products(tenant_id, status);
```

## Channels & Marketplaces

```sql
CREATE TABLE channels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(50) NOT NULL,                       -- shopify, woocommerce, amazon, ebay, pos, custom
    platform_store_id VARCHAR(255),                  -- external store/shop ID
    base_url VARCHAR(500),
    credentials_encrypted BYTEA,                     -- encrypted API credentials
    sync_enabled BOOLEAN NOT NULL DEFAULT true,
    last_sync_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE channel_products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    channel_id UUID NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    external_product_id VARCHAR(255),                -- ASIN, Shopify product ID, etc.
    external_variant_id VARCHAR(255),
    current_price NUMERIC(12,4),
    current_currency CHAR(3) DEFAULT 'USD',
    map_price NUMERIC(12,4),                         -- minimum advertised price
    msrp NUMERIC(12,4),                              -- manufacturer suggested retail price
    is_listed BOOLEAN NOT NULL DEFAULT true,
    last_price_push_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(channel_id, product_id)
);

CREATE INDEX idx_channel_products_product ON channel_products(product_id);
CREATE INDEX idx_channel_products_external ON channel_products(channel_id, external_product_id);
```

## Jurisdictions & Compliance

```sql
CREATE TABLE jurisdictions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    iso_code VARCHAR(10) NOT NULL UNIQUE,            -- ISO 3166-1 alpha-2 or ISO 3166-2
    name VARCHAR(255) NOT NULL,
    parent_code VARCHAR(10) REFERENCES jurisdictions(iso_code),
    omnibus_directive_applies BOOLEAN NOT NULL DEFAULT false,
    reference_price_lookback_days INTEGER DEFAULT 30, -- EU Omnibus: 30 days
    vat_rate NUMERIC(5,4),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE channel_jurisdictions (
    channel_id UUID NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id) ON DELETE CASCADE,
    PRIMARY KEY (channel_id, jurisdiction_id)
);
```

## Competitor Monitoring

```sql
CREATE TABLE competitors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    website_url VARCHAR(500),
    type VARCHAR(50) NOT NULL DEFAULT 'retailer',    -- retailer, marketplace_seller, manufacturer
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE competitor_products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    competitor_id UUID NOT NULL REFERENCES competitors(id) ON DELETE CASCADE,
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    external_url VARCHAR(1000),
    external_product_id VARCHAR(255),
    match_confidence NUMERIC(3,2) NOT NULL DEFAULT 1.00,  -- 0.00-1.00
    match_method VARCHAR(50) DEFAULT 'manual',       -- manual, gtin, title_match, ml_matched
    is_verified BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(competitor_id, product_id)
);

CREATE INDEX idx_competitor_products_product ON competitor_products(product_id);

CREATE TABLE competitor_prices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    competitor_product_id UUID NOT NULL REFERENCES competitor_products(id) ON DELETE CASCADE,
    price NUMERIC(12,4) NOT NULL,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    price_converted NUMERIC(12,4),                   -- normalised to tenant base currency
    was_in_stock BOOLEAN,
    is_promotional BOOLEAN NOT NULL DEFAULT false,
    promotion_label VARCHAR(255),
    shipping_cost NUMERIC(12,4),
    observed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    source VARCHAR(50) DEFAULT 'scrape',             -- scrape, api, manual
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_competitor_prices_product_time ON competitor_prices(competitor_product_id, observed_at DESC);
CREATE INDEX idx_competitor_prices_observed ON competitor_prices(observed_at);
```

## Pricing Rules & Strategies

```sql
CREATE TABLE pricing_strategies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    type VARCHAR(50) NOT NULL,                       -- rule_based, algorithmic, hybrid
    priority INTEGER NOT NULL DEFAULT 100,           -- lower = higher priority
    is_active BOOLEAN NOT NULL DEFAULT false,
    starts_at TIMESTAMPTZ,
    ends_at TIMESTAMPTZ,
    created_by UUID REFERENCES users(id),
    approved_by UUID REFERENCES users(id),
    approved_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pricing_strategies_tenant ON pricing_strategies(tenant_id, is_active);

CREATE TABLE pricing_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    strategy_id UUID NOT NULL REFERENCES pricing_strategies(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    rule_type VARCHAR(50) NOT NULL,                  -- cost_plus, competitor_match, competitor_beat,
                                                     -- margin_floor, margin_target, map_floor,
                                                     -- msrp_ceiling, omnibus_floor, inventory_markdown
    priority INTEGER NOT NULL DEFAULT 100,
    -- Rule parameters (normalised)
    target_margin_pct NUMERIC(6,4),
    competitor_offset_pct NUMERIC(6,4),              -- e.g. -0.0200 = 2% below competitor
    competitor_offset_amount NUMERIC(12,4),
    price_floor NUMERIC(12,4),
    price_ceiling NUMERIC(12,4),
    rounding_strategy VARCHAR(20) DEFAULT 'none',    -- none, charm_99, round_up, round_nearest
    rounding_precision INTEGER DEFAULT 2,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pricing_rule_scope (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_id UUID NOT NULL REFERENCES pricing_rules(id) ON DELETE CASCADE,
    scope_type VARCHAR(50) NOT NULL,                 -- all_products, category, product, brand, channel, tag
    scope_value VARCHAR(255),                        -- category slug, product ID, brand name, etc.
    include BOOLEAN NOT NULL DEFAULT true,           -- true = include, false = exclude
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pricing_rule_scope_rule ON pricing_rule_scope(rule_id);
```

## Price History & Omnibus Compliance

```sql
CREATE TABLE price_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    channel_product_id UUID NOT NULL REFERENCES channel_products(id) ON DELETE CASCADE,
    old_price NUMERIC(12,4),
    new_price NUMERIC(12,4) NOT NULL,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    change_reason VARCHAR(50) NOT NULL,              -- rule_applied, manual, experiment, sync, markdown
    rule_id UUID REFERENCES pricing_rules(id),
    experiment_id UUID,                              -- FK added after experiments table
    omnibus_reference_price NUMERIC(12,4),           -- lowest price in prior 30 days (EU Omnibus)
    changed_by UUID REFERENCES users(id),
    changed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_price_history_channel_product ON price_history(channel_product_id, changed_at DESC);
CREATE INDEX idx_price_history_omnibus ON price_history(channel_product_id, changed_at)
    WHERE omnibus_reference_price IS NOT NULL;
```

## Demand & Elasticity

```sql
CREATE TABLE demand_forecasts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    channel_id UUID REFERENCES channels(id) ON DELETE SET NULL,
    forecast_date DATE NOT NULL,
    horizon_days INTEGER NOT NULL DEFAULT 7,
    predicted_units NUMERIC(12,2) NOT NULL,
    predicted_revenue NUMERIC(14,4),
    confidence_lower NUMERIC(12,2),
    confidence_upper NUMERIC(12,2),
    model_version VARCHAR(50) NOT NULL,
    features_used TEXT[],                             -- e.g. {'price', 'seasonality', 'weather', 'competitor_price'}
    generated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_demand_forecasts_product ON demand_forecasts(product_id, forecast_date DESC);

CREATE TABLE elasticity_estimates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    channel_id UUID REFERENCES channels(id),
    segment VARCHAR(100),                            -- customer segment, if applicable
    price_elasticity NUMERIC(8,4) NOT NULL,          -- typically negative (e.g. -1.5)
    cross_elasticity_product_id UUID REFERENCES products(id),
    cross_elasticity NUMERIC(8,4),                   -- substitution (+) or complement (-)
    confidence_interval NUMERIC(6,4),
    sample_size INTEGER,
    estimation_method VARCHAR(50),                   -- ols, log_log, ml_gradient_boost
    valid_from DATE NOT NULL,
    valid_to DATE,
    model_version VARCHAR(50),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_elasticity_product ON elasticity_estimates(product_id, valid_from DESC);
```

## Inventory

```sql
CREATE TABLE inventory_levels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    channel_id UUID REFERENCES channels(id),
    location_code VARCHAR(50),
    quantity_on_hand INTEGER NOT NULL DEFAULT 0,
    quantity_reserved INTEGER NOT NULL DEFAULT 0,
    quantity_available INTEGER GENERATED ALWAYS AS (quantity_on_hand - quantity_reserved) STORED,
    reorder_point INTEGER,
    days_of_supply NUMERIC(6,1),
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(product_id, channel_id, location_code)
);

CREATE INDEX idx_inventory_product ON inventory_levels(product_id);
```

## Experiments (A/B Testing)

```sql
CREATE TABLE experiments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    hypothesis TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'draft',     -- draft, running, paused, completed, cancelled
    traffic_pct NUMERIC(5,2) NOT NULL DEFAULT 50.00, -- % of traffic in experiment
    starts_at TIMESTAMPTZ,
    ends_at TIMESTAMPTZ,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE experiment_variants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id UUID NOT NULL REFERENCES experiments(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,                      -- 'control', 'variant_a', 'variant_b'
    is_control BOOLEAN NOT NULL DEFAULT false,
    traffic_weight NUMERIC(5,2) NOT NULL DEFAULT 50.00,
    strategy_id UUID REFERENCES pricing_strategies(id),
    price_modifier_pct NUMERIC(6,4),                 -- e.g. +0.05 = 5% above control
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE experiment_assignments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id UUID NOT NULL REFERENCES experiments(id) ON DELETE CASCADE,
    variant_id UUID NOT NULL REFERENCES experiment_variants(id) ON DELETE CASCADE,
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    channel_id UUID REFERENCES channels(id),
    assigned_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(experiment_id, product_id, channel_id)
);

CREATE TABLE experiment_results (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    variant_id UUID NOT NULL REFERENCES experiment_variants(id) ON DELETE CASCADE,
    product_id UUID NOT NULL REFERENCES products(id),
    metric_date DATE NOT NULL,
    impressions INTEGER DEFAULT 0,
    conversions INTEGER DEFAULT 0,
    revenue NUMERIC(14,4) DEFAULT 0,
    margin NUMERIC(14,4) DEFAULT 0,
    units_sold INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(variant_id, product_id, metric_date)
);
```

## Repricing Actions & Audit

```sql
CREATE TABLE repricing_runs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    strategy_id UUID REFERENCES pricing_strategies(id),
    trigger_type VARCHAR(50) NOT NULL,               -- scheduled, competitor_change, manual, inventory_alert
    status VARCHAR(20) NOT NULL DEFAULT 'running',   -- running, completed, failed, partial
    products_evaluated INTEGER DEFAULT 0,
    prices_changed INTEGER DEFAULT 0,
    errors INTEGER DEFAULT 0,
    started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE repricing_actions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id UUID NOT NULL REFERENCES repricing_runs(id) ON DELETE CASCADE,
    channel_product_id UUID NOT NULL REFERENCES channel_products(id),
    old_price NUMERIC(12,4),
    recommended_price NUMERIC(12,4) NOT NULL,
    final_price NUMERIC(12,4),                       -- after MAP/MSRP/Omnibus constraints
    rule_id UUID REFERENCES pricing_rules(id),
    status VARCHAR(20) NOT NULL DEFAULT 'pending',   -- pending, approved, applied, rejected, failed
    rejection_reason TEXT,
    applied_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_repricing_actions_run ON repricing_actions(run_id);
CREATE INDEX idx_repricing_actions_status ON repricing_actions(status) WHERE status = 'pending';
```

## Sales & Performance Data

```sql
CREATE TABLE sales_transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    channel_id UUID NOT NULL REFERENCES channels(id),
    product_id UUID NOT NULL REFERENCES products(id),
    external_order_id VARCHAR(255),
    quantity INTEGER NOT NULL,
    unit_price NUMERIC(12,4) NOT NULL,
    total_price NUMERIC(14,4) NOT NULL,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    discount_amount NUMERIC(12,4) DEFAULT 0,
    cost_price NUMERIC(12,4),
    margin NUMERIC(12,4),
    customer_segment VARCHAR(100),
    transaction_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sales_tenant_time ON sales_transactions(tenant_id, transaction_at DESC);
CREATE INDEX idx_sales_product_time ON sales_transactions(product_id, transaction_at DESC);
CREATE INDEX idx_sales_channel_time ON sales_transactions(channel_id, transaction_at DESC);
```

## Notifications & Webhooks

```sql
CREATE TABLE webhook_subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    event_type VARCHAR(100) NOT NULL,                -- price.changed, competitor.price_drop, experiment.completed
    target_url VARCHAR(1000) NOT NULL,
    secret_hash VARCHAR(255) NOT NULL,               -- HMAC-SHA256 signing secret
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_deliveries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID NOT NULL REFERENCES webhook_subscriptions(id) ON DELETE CASCADE,
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    response_status INTEGER,
    response_body TEXT,
    attempt_count INTEGER NOT NULL DEFAULT 1,
    delivered_at TIMESTAMPTZ,
    next_retry_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_webhook_deliveries_retry ON webhook_deliveries(next_retry_at)
    WHERE response_status IS NULL OR response_status >= 400;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenancy & Auth | 3 | tenants, users, api_clients |
| Product Catalogue | 3 | product_categories, products, (inventory_levels) |
| Channels | 3 | channels, channel_products, channel_jurisdictions |
| Jurisdictions | 1 | jurisdictions |
| Competitors | 3 | competitors, competitor_products, competitor_prices |
| Pricing Rules | 3 | pricing_strategies, pricing_rules, pricing_rule_scope |
| Price History | 1 | price_history |
| Demand & Elasticity | 2 | demand_forecasts, elasticity_estimates |
| Inventory | 1 | inventory_levels |
| Experiments | 4 | experiments, experiment_variants, experiment_assignments, experiment_results |
| Repricing | 2 | repricing_runs, repricing_actions |
| Sales | 1 | sales_transactions |
| Webhooks | 2 | webhook_subscriptions, webhook_deliveries |
| **Total** | **29** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** -- enables distributed ID generation across microservices and avoids sequential ID enumeration attacks in APIs.

2. **Tenant-scoped uniqueness constraints** -- `UNIQUE(tenant_id, sku)` rather than global uniqueness, supporting multi-tenant deployment in shared tables with PostgreSQL Row-Level Security.

3. **Materialised category paths** -- `product_categories.path` stores the full path string (`/electronics/phones/smartphones`) enabling efficient prefix queries (`WHERE path LIKE '/electronics/%'`) without recursive CTEs for most read operations.

4. **Separate competitor_prices table** -- competitor prices are append-only observations, not updates to a single row. This preserves full price history for trend analysis and supports the EU Omnibus Directive's 30-day lookback requirement.

5. **Pricing rule scoping via junction table** -- `pricing_rule_scope` allows rules to target any combination of products, categories, brands, channels, and tags without modifying the rules table structure.

6. **Omnibus reference price in price_history** -- the `omnibus_reference_price` column pre-computes the 30-day lowest price at the time of each price change, avoiding expensive window queries at display time.

7. **Experiment-product assignment granularity** -- A/B tests are assigned at the product-channel level, not the user level, because pricing experiments in retail typically vary the listed price rather than personalising per visitor.

8. **Encrypted channel credentials** -- `channels.credentials_encrypted` stores API keys as encrypted binary, never as plaintext, with decryption handled at the application layer.

9. **Generated column for available inventory** -- `quantity_available` is a stored generated column, ensuring consistency without application-level calculation.

10. **Append-only webhook delivery log** -- delivery attempts are logged with response status for debugging failed integrations, with an index on `next_retry_at` for efficient retry polling.
