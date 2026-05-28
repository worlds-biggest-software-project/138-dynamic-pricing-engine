# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Dynamic Pricing Engine · Created: 2026-05-19

## Philosophy

This model uses standard relational tables for core entities with well-known schemas (products, channels, competitors) while storing variable, domain-specific, and rapidly-evolving data in JSONB columns. Pricing rules, ML model configurations, channel-specific attributes, and jurisdiction-specific compliance fields live in JSONB rather than requiring new columns or junction tables for every variation.

This approach is inspired by how modern SaaS platforms like Shopify handle variability (product metafields as JSON), how Pricefx stores flexible pricing formulas, and how platforms operating across jurisdictions need to accommodate region-specific data (EU Omnibus fields, US MAP constraints, UK VAT rules) without schema migrations for each market expansion. PostgreSQL's JSONB type with GIN indexes provides efficient querying of structured data within JSON columns while maintaining full SQL query capabilities.

The hybrid model is best suited for teams that want a rapid MVP with the flexibility to add new pricing rule types, channel integrations, and compliance fields without schema migrations, while still getting the benefits of relational integrity for core entities and foreign key relationships.

**Best for:** Fast-moving teams building a multi-market pricing engine where rule types, channel attributes, and compliance requirements vary by jurisdiction and evolve frequently.

**Trade-offs:**
- Pro: New pricing rule types, channel attributes, and compliance fields require zero schema migrations
- Pro: Fewer tables than fully normalized (~20 vs ~30+), simpler to understand and maintain
- Pro: PostgreSQL JSONB with GIN indexes provides sub-millisecond lookups on JSON fields
- Pro: Jurisdiction-specific fields (EU Omnibus, US MAP, UK post-Brexit rules) coexist in one table
- Pro: ML model parameters and hyperparameters stored as JSONB avoid rigid column definitions
- Con: JSONB fields lack database-level type constraints -- validation must happen in application code or via JSON Schema
- Con: Complex JSONB queries (`->`, `->>`, `@>`, `?`) are less readable than standard SQL
- Con: JSONB columns cannot participate in foreign key constraints
- Con: Schema documentation is split between DDL and JSON Schema definitions
- Con: JSONB indexes are larger than B-tree indexes on typed columns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 GTIN | `products.gtin` as a typed VARCHAR column (not in JSONB) for indexed cross-system matching |
| ISO 4217 | `price` and `currency` are always relational columns, never buried in JSONB |
| ISO 3166-1/2 | `jurisdictions.iso_code` is relational; jurisdiction-specific compliance rules are JSONB |
| JSON Schema 2020-12 | Defines and validates the structure of all JSONB columns programmatically |
| EU Omnibus Directive | Omnibus fields stored in `channel_products.compliance JSONB` with jurisdiction-aware validation |
| OpenAPI 3.1 | API responses map JSONB fields to typed OpenAPI schemas for client code generation |
| AsyncAPI 3.x | Webhook event payloads documented as AsyncAPI messages |

---

## Tenancy & Users

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    plan VARCHAR(50) NOT NULL DEFAULT 'free',
    settings JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "base_currency": "USD",
    --   "timezone": "America/New_York",
    --   "repricing_frequency_minutes": 60,
    --   "approval_required_above_pct": 0.10,
    --   "omnibus_enabled": true,
    --   "map_enforcement": "strict",
    --   "notification_channels": ["email", "slack"]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255),
    role VARCHAR(50) NOT NULL DEFAULT 'viewer',
    permissions JSONB NOT NULL DEFAULT '[]',
    -- permissions example:
    -- ["prices:read", "prices:write", "rules:manage", "experiments:create"]
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
```

## Products & Categories

```sql
CREATE TABLE product_categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    parent_id UUID REFERENCES product_categories(id) ON DELETE SET NULL,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL,
    path TEXT NOT NULL DEFAULT '',
    attributes JSONB NOT NULL DEFAULT '{}',
    -- attributes example:
    -- {
    --   "default_margin_target": 0.35,
    --   "markdown_velocity": "slow",
    --   "seasonality_pattern": "holiday_peak"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, slug)
);

CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    category_id UUID REFERENCES product_categories(id) ON DELETE SET NULL,
    sku VARCHAR(100) NOT NULL,
    gtin VARCHAR(14),
    name VARCHAR(500) NOT NULL,
    brand VARCHAR(255),
    cost_price NUMERIC(12,4),
    cost_currency CHAR(3) DEFAULT 'USD',
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    tags TEXT[] DEFAULT '{}',
    attributes JSONB NOT NULL DEFAULT '{}',
    -- attributes example:
    -- {
    --   "weight_kg": 0.45,
    --   "is_perishable": false,
    --   "shelf_life_days": null,
    --   "unit_of_measure": "each",
    --   "unit_quantity": 1,
    --   "supplier": "WidgetCo Inc",
    --   "lead_time_days": 14,
    --   "substitute_skus": ["WIDGET-002", "WIDGET-003"],
    --   "custom_fields": {
    --     "color": "blue",
    --     "material": "aluminum"
    --   }
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, sku)
);

CREATE INDEX idx_products_tenant ON products(tenant_id);
CREATE INDEX idx_products_gtin ON products(gtin) WHERE gtin IS NOT NULL;
CREATE INDEX idx_products_attributes ON products USING GIN (attributes);
CREATE INDEX idx_products_tags ON products USING GIN (tags);
```

## Channels & Channel Products

```sql
CREATE TABLE channels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(50) NOT NULL,                       -- shopify, woocommerce, amazon, ebay, pos, custom
    config JSONB NOT NULL DEFAULT '{}',
    -- config example (Shopify):
    -- {
    --   "store_url": "mystore.myshopify.com",
    --   "api_version": "2026-04",
    --   "sync_interval_minutes": 15,
    --   "use_graphql": true,
    --   "price_list_id": "gid://shopify/PriceList/123"
    -- }
    -- config example (Amazon):
    -- {
    --   "marketplace_id": "ATVPDKIKX0DER",
    --   "seller_id": "A1B2C3D4E5",
    --   "region": "na",
    --   "batch_size": 20,
    --   "use_automated_pricing": false
    -- }
    credentials_encrypted BYTEA,
    jurisdictions VARCHAR(10)[] DEFAULT '{}',         -- ISO 3166 codes this channel serves
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
    external_ids JSONB NOT NULL DEFAULT '{}',
    -- external_ids example:
    -- {
    --   "product_id": "7891234567",
    --   "variant_id": "43210987654",
    --   "asin": "B09XYZ1234",
    --   "listing_id": "012ABC"
    -- }
    current_price NUMERIC(12,4),
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    constraints JSONB NOT NULL DEFAULT '{}',
    -- constraints example:
    -- {
    --   "map_price": 24.99,
    --   "msrp": 34.99,
    --   "price_floor": 20.00,
    --   "price_ceiling": 49.99,
    --   "rounding": "charm_99"
    -- }
    compliance JSONB NOT NULL DEFAULT '{}',
    -- compliance example (EU market):
    -- {
    --   "omnibus_reference_price": 25.99,
    --   "omnibus_calculated_at": "2026-05-19T10:00:00Z",
    --   "omnibus_lookback_days": 30,
    --   "vat_rate": 0.19,
    --   "vat_included_in_price": true
    -- }
    -- compliance example (US market):
    -- {
    --   "map_enforced": true,
    --   "map_source": "brand_agreement",
    --   "sales_tax_exempt": false
    -- }
    is_listed BOOLEAN NOT NULL DEFAULT true,
    last_price_push_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(channel_id, product_id)
);

CREATE INDEX idx_channel_products_product ON channel_products(product_id);
CREATE INDEX idx_channel_products_compliance ON channel_products USING GIN (compliance);
```

## Competitors & Price Observations

```sql
CREATE TABLE competitors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    config JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "website_url": "https://megastore.com",
    --   "scrape_frequency_hours": 6,
    --   "type": "retailer",
    --   "markets": ["US", "UK", "DE"],
    --   "reliability_score": 0.92
    -- }
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE competitor_product_matches (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    competitor_id UUID NOT NULL REFERENCES competitors(id) ON DELETE CASCADE,
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    match_data JSONB NOT NULL DEFAULT '{}',
    -- match_data example:
    -- {
    --   "external_url": "https://megastore.com/widget-premium",
    --   "external_product_id": "MS-12345",
    --   "match_method": "gtin",
    --   "confidence": 0.98,
    --   "verified_by": "user-uuid",
    --   "verified_at": "2026-05-10T14:00:00Z"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(competitor_id, product_id)
);

CREATE TABLE competitor_prices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    match_id UUID NOT NULL REFERENCES competitor_product_matches(id) ON DELETE CASCADE,
    price NUMERIC(12,4) NOT NULL,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    price_converted NUMERIC(12,4),
    observation JSONB NOT NULL DEFAULT '{}',
    -- observation example:
    -- {
    --   "was_in_stock": true,
    --   "is_promotional": false,
    --   "promotion_label": null,
    --   "shipping_cost": 5.99,
    --   "shipping_free_threshold": 35.00,
    --   "buy_box_winner": true,
    --   "seller_rating": 4.7,
    --   "source": "api"
    -- }
    observed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_competitor_prices_match_time ON competitor_prices(match_id, observed_at DESC);
CREATE INDEX idx_competitor_prices_observed ON competitor_prices(observed_at);
```

## Pricing Strategies & Rules

This is where the JSONB approach truly shines -- different rule types have fundamentally different parameters.

```sql
CREATE TABLE pricing_strategies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    is_active BOOLEAN NOT NULL DEFAULT false,
    priority INTEGER NOT NULL DEFAULT 100,
    schedule JSONB,
    -- schedule example:
    -- {
    --   "type": "recurring",
    --   "frequency": "hourly",
    --   "active_hours": {"start": "06:00", "end": "22:00"},
    --   "active_days": ["mon", "tue", "wed", "thu", "fri"],
    --   "timezone": "America/New_York"
    -- }
    created_by UUID REFERENCES users(id),
    approved_by UUID REFERENCES users(id),
    approved_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pricing_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    strategy_id UUID NOT NULL REFERENCES pricing_strategies(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    rule_type VARCHAR(50) NOT NULL,                  -- cost_plus, competitor_match, competitor_beat,
                                                     -- margin_floor, inventory_markdown, demand_based,
                                                     -- time_decay, bundle, clearance
    priority INTEGER NOT NULL DEFAULT 100,
    is_active BOOLEAN NOT NULL DEFAULT true,

    -- Scope: which products does this rule apply to?
    scope JSONB NOT NULL DEFAULT '{"type": "all"}',
    -- scope examples:
    -- {"type": "all"}
    -- {"type": "category", "values": ["/electronics/widgets"]}
    -- {"type": "products", "skus": ["WIDGET-001", "WIDGET-002"]}
    -- {"type": "brand", "values": ["WidgetCo"]}
    -- {"type": "tags", "values": ["clearance", "seasonal"]}
    -- {"type": "filter", "min_price": 10, "max_price": 100, "categories": ["/electronics"]}

    -- Parameters: rule-type-specific configuration
    params JSONB NOT NULL DEFAULT '{}',
    -- params example (cost_plus):
    -- {
    --   "markup_pct": 0.40,
    --   "include_shipping": true
    -- }
    -- params example (competitor_beat):
    -- {
    --   "offset_pct": -0.02,
    --   "offset_amount": null,
    --   "competitor_ids": ["uuid-1", "uuid-2"],
    --   "use_lowest": true,
    --   "ignore_out_of_stock": true
    -- }
    -- params example (inventory_markdown):
    -- {
    --   "thresholds": [
    --     {"days_of_supply_above": 90, "markdown_pct": 0.10},
    --     {"days_of_supply_above": 120, "markdown_pct": 0.20},
    --     {"days_of_supply_above": 180, "markdown_pct": 0.35}
    --   ]
    -- }
    -- params example (demand_based):
    -- {
    --   "elasticity_source": "latest_estimate",
    --   "target_metric": "margin",
    --   "price_sensitivity": 0.5,
    --   "max_increase_pct": 0.15,
    --   "max_decrease_pct": 0.25
    -- }
    -- params example (time_decay / perishable):
    -- {
    --   "decay_start_days_before_expiry": 7,
    --   "min_price_pct_of_cost": 0.50,
    --   "decay_curve": "exponential"
    -- }

    -- Constraints: guardrails applied after price calculation
    constraints JSONB NOT NULL DEFAULT '{}',
    -- constraints example:
    -- {
    --   "enforce_map": true,
    --   "enforce_msrp_ceiling": true,
    --   "enforce_omnibus": true,
    --   "min_margin_pct": 0.10,
    --   "max_price_change_pct": 0.15,
    --   "rounding": "charm_99"
    -- }

    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pricing_rules_strategy ON pricing_rules(strategy_id);
CREATE INDEX idx_pricing_rules_scope ON pricing_rules USING GIN (scope);
CREATE INDEX idx_pricing_rules_type ON pricing_rules(rule_type);
```

## Price History

```sql
CREATE TABLE price_changes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    channel_product_id UUID NOT NULL REFERENCES channel_products(id) ON DELETE CASCADE,
    old_price NUMERIC(12,4),
    new_price NUMERIC(12,4) NOT NULL,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    reason VARCHAR(50) NOT NULL,
    context JSONB NOT NULL DEFAULT '{}',
    -- context example:
    -- {
    --   "rule_id": "uuid-...",
    --   "rule_type": "competitor_beat",
    --   "strategy_id": "uuid-...",
    --   "competitor_trigger": {
    --     "competitor_name": "MegaStore",
    --     "competitor_price": 27.49,
    --     "price_delta_pct": -0.083
    --   },
    --   "constraints_applied": ["map_floor", "charm_99_rounding"],
    --   "omnibus_reference_price": 25.99,
    --   "elasticity_estimate": -1.45,
    --   "demand_forecast_units": 142,
    --   "experiment_id": null,
    --   "approval": "auto"
    -- }
    changed_by UUID REFERENCES users(id),
    changed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_price_changes_channel_product ON price_changes(channel_product_id, changed_at DESC);
CREATE INDEX idx_price_changes_tenant ON price_changes(tenant_id, changed_at DESC);
```

## ML Models & Forecasting

```sql
CREATE TABLE ml_models (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    model_type VARCHAR(50) NOT NULL,                 -- demand_forecast, elasticity, rl_agent, cross_elasticity
    version VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'training',  -- training, active, retired, failed
    config JSONB NOT NULL DEFAULT '{}',
    -- config example (demand_forecast):
    -- {
    --   "algorithm": "xgboost",
    --   "features": ["price", "day_of_week", "is_holiday", "competitor_avg", "weather_temp"],
    --   "training_window_days": 365,
    --   "hyperparameters": {
    --     "n_estimators": 500,
    --     "max_depth": 8,
    --     "learning_rate": 0.05
    --   }
    -- }
    -- config example (rl_agent):
    -- {
    --   "algorithm": "PPO",
    --   "state_space": ["current_price", "competitor_prices", "inventory", "day_of_week", "demand_trend"],
    --   "action_space": {"type": "discrete", "price_adjustments_pct": [-0.10, -0.05, -0.02, 0, 0.02, 0.05, 0.10]},
    --   "reward": "margin",
    --   "discount_factor": 0.95,
    --   "training_episodes": 10000
    -- }
    metrics JSONB NOT NULL DEFAULT '{}',
    -- metrics example:
    -- {
    --   "mae": 12.3,
    --   "rmse": 18.7,
    --   "r_squared": 0.87,
    --   "training_samples": 45000,
    --   "validation_samples": 5000,
    --   "trained_at": "2026-05-18T04:00:00Z"
    -- }
    artifact_path VARCHAR(500),                      -- S3/GCS path to serialised model
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE demand_forecasts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    channel_id UUID REFERENCES channels(id),
    model_id UUID REFERENCES ml_models(id),
    forecast_date DATE NOT NULL,
    horizon_days INTEGER NOT NULL DEFAULT 7,
    predictions JSONB NOT NULL,
    -- predictions example:
    -- {
    --   "units": 142.5,
    --   "revenue": 4132.75,
    --   "confidence_lower": 128.0,
    --   "confidence_upper": 157.0,
    --   "price_at_forecast": 28.99,
    --   "features_importance": {
    --     "price": 0.35,
    --     "seasonality": 0.25,
    --     "competitor_avg": 0.20,
    --     "weather": 0.10,
    --     "trend": 0.10
    --   }
    -- }
    generated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_forecasts_product ON demand_forecasts(product_id, forecast_date DESC);

CREATE TABLE elasticity_estimates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    channel_id UUID REFERENCES channels(id),
    model_id UUID REFERENCES ml_models(id),
    estimate JSONB NOT NULL,
    -- estimate example:
    -- {
    --   "price_elasticity": -1.45,
    --   "confidence_interval": 0.22,
    --   "method": "log_log_regression",
    --   "sample_size": 1200,
    --   "segment": "all_customers",
    --   "cross_elasticities": [
    --     {"product_sku": "WIDGET-002", "elasticity": 0.35, "relationship": "substitute"},
    --     {"product_sku": "ACCESSORY-001", "elasticity": -0.18, "relationship": "complement"}
    --   ]
    -- }
    valid_from DATE NOT NULL,
    valid_to DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_elasticity_product ON elasticity_estimates(product_id, valid_from DESC);
```

## Experiments

```sql
CREATE TABLE experiments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    hypothesis TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'draft',
    config JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "traffic_pct": 50,
    --   "variants": [
    --     {"name": "control", "weight": 50, "price_modifier_pct": 0},
    --     {"name": "higher_price", "weight": 50, "price_modifier_pct": 0.05}
    --   ],
    --   "scope": {"type": "category", "values": ["/electronics/widgets"]},
    --   "channels": ["uuid-1"],
    --   "success_metric": "margin_per_unit",
    --   "min_sample_size": 500,
    --   "significance_level": 0.05
    -- }
    results JSONB,
    -- results example (populated after experiment ends):
    -- {
    --   "control": {"impressions": 5200, "conversions": 312, "revenue": 9048.88, "avg_margin": 8.42},
    --   "higher_price": {"impressions": 5180, "conversions": 285, "revenue": 8863.50, "avg_margin": 9.71},
    --   "winner": "higher_price",
    --   "p_value": 0.023,
    --   "confidence_level": 0.977,
    --   "lift_pct": 0.153
    -- }
    starts_at TIMESTAMPTZ,
    ends_at TIMESTAMPTZ,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_experiments_tenant ON experiments(tenant_id, status);
```

## Inventory & Sales

```sql
CREATE TABLE inventory_levels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    channel_id UUID REFERENCES channels(id),
    quantity_on_hand INTEGER NOT NULL DEFAULT 0,
    quantity_reserved INTEGER NOT NULL DEFAULT 0,
    days_of_supply NUMERIC(6,1),
    details JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "location_code": "WH-EAST",
    --   "reorder_point": 50,
    --   "expiry_date": "2026-07-15",
    --   "last_restock_at": "2026-05-01T08:00:00Z",
    --   "inbound_units": 200,
    --   "inbound_eta": "2026-05-25"
    -- }
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(product_id, channel_id)
);

CREATE TABLE sales_daily (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    product_id UUID NOT NULL REFERENCES products(id),
    channel_id UUID NOT NULL REFERENCES channels(id),
    sale_date DATE NOT NULL,
    units_sold INTEGER NOT NULL DEFAULT 0,
    gross_revenue NUMERIC(14,4) NOT NULL DEFAULT 0,
    net_revenue NUMERIC(14,4) NOT NULL DEFAULT 0,
    total_cost NUMERIC(14,4) NOT NULL DEFAULT 0,
    total_margin NUMERIC(14,4) NOT NULL DEFAULT 0,
    avg_selling_price NUMERIC(12,4),
    metrics JSONB NOT NULL DEFAULT '{}',
    -- metrics example:
    -- {
    --   "returns": 2,
    --   "cancellations": 1,
    --   "discount_units": 15,
    --   "avg_discount_pct": 0.08,
    --   "page_views": 450,
    --   "conversion_rate": 0.042
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, product_id, channel_id, sale_date)
);

CREATE INDEX idx_sales_daily_product ON sales_daily(product_id, sale_date DESC);
```

## Audit Log

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    actor_id UUID,
    actor_type VARCHAR(20) NOT NULL DEFAULT 'user',  -- user, system, api_client
    action VARCHAR(100) NOT NULL,                    -- strategy.created, rule.updated, price.changed, etc.
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    changes JSONB NOT NULL DEFAULT '{}',
    -- changes example:
    -- {
    --   "before": {"is_active": false, "priority": 200},
    --   "after": {"is_active": true, "priority": 100}
    -- }
    ip_address INET,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
```

---

## Example Queries

### Find all products matching a rule's JSONB scope

```sql
-- Products in the "electronics/widgets" category matching a rule scope
SELECT p.id, p.sku, p.name
FROM products p
JOIN product_categories pc ON p.category_id = pc.id
JOIN pricing_rules pr ON pr.scope @> '{"type": "category"}'
WHERE pc.path LIKE ANY(
    SELECT '%' || jsonb_array_elements_text(pr.scope->'values') || '%'
)
AND pr.id = 'rule-uuid-here';
```

### Query products by JSONB attributes

```sql
-- Find perishable products with less than 30 days of supply
SELECT p.sku, p.name, il.days_of_supply,
       il.details->>'expiry_date' AS expiry_date
FROM products p
JOIN inventory_levels il ON p.id = il.product_id
WHERE p.attributes @> '{"is_perishable": true}'
  AND il.days_of_supply < 30
  AND p.tenant_id = 'tenant-uuid-here';
```

### Omnibus compliance check across jurisdictions

```sql
-- All channel products in EU jurisdictions missing Omnibus reference prices
SELECT cp.id, p.sku, c.name AS channel_name
FROM channel_products cp
JOIN products p ON cp.product_id = p.id
JOIN channels c ON cp.channel_id = c.id
WHERE c.jurisdictions && ARRAY['DE', 'FR', 'IT', 'ES', 'NL']
  AND (cp.compliance->>'omnibus_reference_price') IS NULL
  AND cp.is_listed = true;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenancy & Auth | 2 | tenants, users |
| Product Catalogue | 2 | product_categories, products |
| Channels | 2 | channels, channel_products |
| Competitors | 3 | competitors, competitor_product_matches, competitor_prices |
| Pricing Rules | 2 | pricing_strategies, pricing_rules |
| Price History | 1 | price_changes |
| ML & Forecasting | 3 | ml_models, demand_forecasts, elasticity_estimates |
| Experiments | 1 | experiments (variants and results are JSONB) |
| Inventory & Sales | 2 | inventory_levels, sales_daily |
| Audit | 1 | audit_log |
| **Total** | **19** | ~35% fewer tables than normalized model |

---

## Key Design Decisions

1. **JSONB for variability, relational for identity** -- columns that participate in JOINs, WHERE clauses, and UNIQUE constraints (id, tenant_id, sku, gtin, price, currency) are always typed relational columns. Fields that vary by rule type, channel, or jurisdiction go in JSONB.

2. **GIN indexes on JSONB columns** -- `products.attributes`, `pricing_rules.scope`, and `channel_products.compliance` have GIN indexes enabling efficient containment queries (`@>`) without full table scans.

3. **Rule parameters as JSONB** -- a `cost_plus` rule has different parameters than an `inventory_markdown` rule. Rather than having 20+ nullable columns or a separate table per rule type, `params JSONB` holds rule-type-specific configuration validated by JSON Schema at the application layer.

4. **Experiment variants and results as JSONB** -- experiments typically have 2-4 variants. Storing them as JSONB in the experiment row (rather than separate `experiment_variants` and `experiment_results` tables) reduces JOINs and keeps the experiment as a self-contained document.

5. **Channel-specific external IDs as JSONB** -- Amazon needs ASIN + listing_id, Shopify needs product_id + variant_id, eBay needs item_id. Rather than separate columns per platform, `external_ids JSONB` adapts to any channel.

6. **Compliance fields as JSONB per channel-product** -- EU Omnibus fields (reference price, lookback days, VAT) differ from US MAP fields. The `compliance JSONB` column accommodates any jurisdiction's requirements without schema migrations.

7. **ML model config and metrics as JSONB** -- ML model hyperparameters, feature lists, and evaluation metrics change frequently as models evolve. JSONB avoids schema migrations for every model iteration.

8. **Pre-aggregated daily sales** -- `sales_daily` stores pre-computed daily aggregates with a `metrics JSONB` column for optional per-day attributes (returns, page views, conversion rates). This avoids querying raw transaction tables for dashboard views.

9. **Audit log with before/after JSONB diffs** -- the `changes` column stores the delta between old and new state, enabling efficient audit queries without needing to diff adjacent event records.

10. **Array columns for tags and jurisdictions** -- PostgreSQL arrays (`TEXT[]`) with GIN indexes are used for tags and jurisdiction lists, avoiding junction tables for simple set-membership queries.
