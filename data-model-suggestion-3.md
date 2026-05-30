# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Demand Planning Platform · Created: 2026-05-20

## Philosophy

This model uses a pragmatic hybrid approach: core structural fields (identifiers, foreign keys, timestamps, status flags) are stored as typed relational columns, while variable, domain-specific, and extensible fields are stored in PostgreSQL JSONB columns. This gives the best of both worlds — relational integrity where it matters and schema-free flexibility where the domain is variable.

Demand planning is a domain where flexibility is essential. Different industries need different product attributes (apparel has size/color/season; food has shelf life/temperature class; electronics has generation/configuration). Different regions have different regulatory requirements. Different customers demand different planning granularities. A hybrid JSONB approach handles all of these without requiring DDL migrations for every new attribute or jurisdiction.

This pattern is widely used in modern SaaS platforms (Stripe, Shopify, Linear) where the core data model is relational but extensible properties are stored as JSON. PostgreSQL's JSONB support — including GIN indexes, containment operators, and jsonpath queries — makes this performant at scale.

**Best for:** Teams building a multi-tenant SaaS platform that must support diverse industries, regions, and customer configurations without per-tenant schema customization.

**Trade-offs:**
- Pro: Rapid feature development — new fields don't require schema migrations
- Pro: Multi-industry support without separate schemas per vertical
- Pro: Jurisdiction-specific attributes without table explosion
- Pro: JSONB GIN indexes provide fast queries on variable fields
- Con: JSONB fields lack database-enforced constraints (validation moves to application layer)
- Con: Reporting queries on JSONB fields are syntactically verbose
- Con: Schema documentation must be maintained manually (no DDL to read)
- Con: Risk of schema drift if JSONB field governance is weak

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 General Specifications v25.0 | GTIN/GLN stored as typed relational columns; additional GS1 attributes in JSONB `identifiers` |
| SCOR-DS v14.0 (ASCM) | Plan process taxonomy stored in relational columns; SCOR-specific metadata in JSONB `scor_attributes` |
| IBF Best Practices | MAPE, bias, WAPE as relational columns; extended metrics and custom KPIs in JSONB `metrics` |
| JSON Schema Draft 2020-12 | JSONB columns validated against registered JSON Schemas at application layer |
| OpenAPI 3.1.0 | API resources use JSON Schema `$ref` for JSONB field documentation |
| ISO 3166-1/2 | Country codes as relational columns; subdivision and regional metadata in JSONB |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE tenant (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    industry VARCHAR(100),              -- retail, cpg, manufacturing, pharma, etc.
    config JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "default_period_granularity": "week",
    --   "planning_horizon_months": 18,
    --   "forecast_confidence_level": 0.95,
    --   "enabled_modules": ["demand_sensing", "sop", "collaboration"],
    --   "default_currency": "USD",
    --   "custom_dimensions": ["brand", "pack_size"],
    --   "jurisdiction_settings": {
    --     "eu": { "gdpr_retention_days": 730 },
    --     "us": { "sarbanes_oxley_audit": true }
    --   }
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE "user" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    email VARCHAR(255) NOT NULL,
    display_name VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL DEFAULT 'planner',  -- admin, planner, manager, executive, viewer
    permissions JSONB NOT NULL DEFAULT '[]',
    preferences JSONB NOT NULL DEFAULT '{}',
    -- preferences example:
    -- {
    --   "default_view": "exceptions",
    --   "timezone": "America/New_York",
    --   "dashboard_layout": { ... }
    -- }
    oidc_subject VARCHAR(500),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_user_tenant ON "user"(tenant_id);
```

---

## Product Dimension

```sql
CREATE TABLE product (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    sku VARCHAR(100) NOT NULL,
    gtin VARCHAR(14),                   -- GS1 Global Trade Item Number
    name VARCHAR(500) NOT NULL,
    unit_of_measure VARCHAR(20) NOT NULL DEFAULT 'EA',
    lifecycle_status VARCHAR(30) NOT NULL DEFAULT 'active',

    -- Relational hierarchy (flattened for query performance)
    family VARCHAR(100),
    category VARCHAR(100),
    subcategory VARCHAR(100),

    -- JSONB for variable attributes that differ by industry
    attributes JSONB NOT NULL DEFAULT '{}',
    -- Retail/apparel example:
    -- {
    --   "brand": "Acme",
    --   "color": "navy",
    --   "size": "L",
    --   "season": "FW2026",
    --   "pack_size": 6,
    --   "shelf_life_days": null
    -- }
    --
    -- Food/CPG example:
    -- {
    --   "brand": "FreshCo",
    --   "shelf_life_days": 14,
    --   "temperature_class": "chilled",
    --   "allergens": ["gluten", "dairy"],
    --   "case_count": 24
    -- }
    --
    -- Pharma example:
    -- {
    --   "ndc_code": "12345-678-90",
    --   "controlled_substance_schedule": null,
    --   "cold_chain": true,
    --   "lot_tracked": true
    -- }

    identifiers JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "gtin_14": "00012345678905",
    --   "upc": "012345678905",
    --   "ean": "0012345678905",
    --   "internal_erp_id": "MAT-001234",
    --   "supplier_part_number": "SP-9876"
    -- }

    launch_date DATE,
    discontinuation_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, sku)
);

CREATE INDEX idx_product_tenant ON product(tenant_id);
CREATE INDEX idx_product_gtin ON product(gtin) WHERE gtin IS NOT NULL;
CREATE INDEX idx_product_hierarchy ON product(tenant_id, family, category, subcategory);
CREATE INDEX idx_product_attributes ON product USING GIN (attributes);
CREATE INDEX idx_product_lifecycle ON product(tenant_id, lifecycle_status);
```

---

## Location Dimension

```sql
CREATE TABLE location (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    code VARCHAR(50) NOT NULL,
    name VARCHAR(255) NOT NULL,
    gln VARCHAR(13),                    -- GS1 Global Location Number
    location_type VARCHAR(30) NOT NULL, -- warehouse, store, dc, plant, virtual
    country_code CHAR(2) NOT NULL,      -- ISO 3166-1 alpha-2
    region VARCHAR(100),
    timezone VARCHAR(50) NOT NULL DEFAULT 'UTC',

    attributes JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "address": { "street": "...", "city": "...", "state": "...", "postal": "..." },
    --   "capacity_units": 50000,
    --   "lead_time_days": 3,
    --   "operating_hours": "06:00-22:00",
    --   "geo": { "lat": 40.7128, "lng": -74.0060 }
    -- }

    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE INDEX idx_location_tenant ON location(tenant_id);
CREATE INDEX idx_location_country ON location(tenant_id, country_code);
CREATE INDEX idx_location_attributes ON location USING GIN (attributes);
```

---

## Channel & Customer

```sql
CREATE TABLE channel (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    code VARCHAR(50) NOT NULL,
    name VARCHAR(255) NOT NULL,
    attributes JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE customer (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    code VARCHAR(100) NOT NULL,
    name VARCHAR(500) NOT NULL,
    channel_id UUID REFERENCES channel(id),
    country_code CHAR(2),
    attributes JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "tier": "strategic",
    --   "annual_revenue": 5000000,
    --   "payment_terms_days": 30,
    --   "cpfr_partner": true,
    --   "edi_capable": true,
    --   "preferred_forecast_format": "edi_830"
    -- }
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE INDEX idx_customer_tenant ON customer(tenant_id);
CREATE INDEX idx_customer_attributes ON customer USING GIN (attributes);
```

---

## Demand History

```sql
CREATE TABLE demand_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    product_id UUID NOT NULL REFERENCES product(id),
    location_id UUID NOT NULL REFERENCES location(id),
    channel_id UUID REFERENCES channel(id),
    customer_id UUID REFERENCES customer(id),
    period_start DATE NOT NULL,
    period_granularity VARCHAR(10) NOT NULL DEFAULT 'week',
    quantity NUMERIC(18,4) NOT NULL,
    cleansed_quantity NUMERIC(18,4),
    is_outlier BOOLEAN NOT NULL DEFAULT false,

    metrics JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "revenue": 12500.00,
    --   "units_returned": 12,
    --   "promotion_id": "promo-uuid-here",
    --   "outlier_reason": "one-time bulk order",
    --   "demand_type": "base",
    --   "custom_dimensions": { "brand": "Acme", "region_group": "Northeast" }
    -- }

    source VARCHAR(50) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_demand_history_lookup ON demand_history(tenant_id, product_id, location_id, period_start);
CREATE INDEX idx_demand_history_period ON demand_history(tenant_id, period_start);
CREATE INDEX idx_demand_history_metrics ON demand_history USING GIN (metrics);
```

---

## Forecast Management

```sql
CREATE TABLE forecast_model (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    name VARCHAR(255) NOT NULL,
    algorithm VARCHAR(100) NOT NULL,
    config JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "seasonal_periods": [7, 52],
    --   "changepoint_prior_scale": 0.05,
    --   "growth": "logistic",
    --   "regressors": ["temperature", "cpi_index"],
    --   "ensemble_members": ["arima", "prophet", "xgboost"],
    --   "ensemble_weights": [0.3, 0.4, 0.3]
    -- }
    is_default BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE forecast_version (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    name VARCHAR(255) NOT NULL,
    version_type VARCHAR(30) NOT NULL,  -- statistical, demand_sensing, consensus, final
    status VARCHAR(30) NOT NULL DEFAULT 'draft',
    planning_horizon_start DATE NOT NULL,
    planning_horizon_end DATE NOT NULL,
    period_granularity VARCHAR(10) NOT NULL DEFAULT 'week',
    created_by UUID NOT NULL REFERENCES "user"(id),
    approved_by UUID REFERENCES "user"(id),
    approved_at TIMESTAMPTZ,

    run_metadata JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "model_id": "...",
    --   "training_window_months": 24,
    --   "signals_used": ["pos_data", "weather", "promotions"],
    --   "total_skus_forecast": 5432,
    --   "execution_time_seconds": 87,
    --   "accuracy_summary": {
    --     "overall_mape": 12.3,
    --     "overall_bias": -1.2,
    --     "overall_wape": 10.8
    --   }
    -- }

    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE forecast_line (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    forecast_version_id UUID NOT NULL REFERENCES forecast_version(id),
    product_id UUID NOT NULL REFERENCES product(id),
    location_id UUID NOT NULL REFERENCES location(id),
    channel_id UUID REFERENCES channel(id),
    customer_id UUID REFERENCES customer(id),
    period_start DATE NOT NULL,
    quantity NUMERIC(18,4) NOT NULL,
    confidence_lower NUMERIC(18,4),
    confidence_upper NUMERIC(18,4),
    confidence_level NUMERIC(5,4),

    details JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "model_used": "prophet",
    --   "ensemble_contribution": { "arima": 0.3, "prophet": 0.45, "xgboost": 0.25 },
    --   "causal_factors": {
    --     "temperature": { "impact": 0.12, "direction": "positive" },
    --     "promotion_active": { "impact": 0.35, "direction": "positive" }
    --   },
    --   "demand_components": {
    --     "base": 850,
    --     "trend": 20,
    --     "seasonal": 130,
    --     "promotional": 200,
    --     "external_signal": 50
    --   }
    -- }

    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_forecast_line_version ON forecast_line(forecast_version_id);
CREATE INDEX idx_forecast_line_product ON forecast_line(product_id, period_start);
CREATE INDEX idx_forecast_line_details ON forecast_line USING GIN (details);
```

---

## Forecast Accuracy

```sql
CREATE TABLE forecast_accuracy (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    forecast_version_id UUID NOT NULL REFERENCES forecast_version(id),
    product_id UUID NOT NULL REFERENCES product(id),
    location_id UUID NOT NULL REFERENCES location(id),
    period_start DATE NOT NULL,
    forecast_quantity NUMERIC(18,4) NOT NULL,
    actual_quantity NUMERIC(18,4) NOT NULL,
    mape NUMERIC(10,4),
    bias NUMERIC(10,4),
    wape NUMERIC(10,4),

    extended_metrics JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "rmse": 45.2,
    --   "mae": 38.1,
    --   "tracking_signal": 2.3,
    --   "forecast_lag_days": 14,
    --   "fva_vs_naive": 8.5,
    --   "within_confidence_band": true
    -- }

    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_accuracy_version ON forecast_accuracy(forecast_version_id);
CREATE INDEX idx_accuracy_product ON forecast_accuracy(product_id, period_start);
```

---

## S&OP Consensus Planning

```sql
CREATE TABLE sop_cycle (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    name VARCHAR(255) NOT NULL,
    cycle_month DATE NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'statistical_review',
    owner_id UUID NOT NULL REFERENCES "user"(id),
    statistical_forecast_id UUID REFERENCES forecast_version(id),
    consensus_forecast_id UUID REFERENCES forecast_version(id),
    final_forecast_id UUID REFERENCES forecast_version(id),

    config JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "review_deadline": "2026-06-05",
    --   "consensus_meeting_date": "2026-06-10",
    --   "executive_review_date": "2026-06-15",
    --   "participants": ["user-uuid-1", "user-uuid-2"],
    --   "review_scope": { "families": ["beverages", "snacks"], "regions": ["NA", "EU"] }
    -- }

    started_at TIMESTAMPTZ,
    closed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sop_override (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sop_cycle_id UUID NOT NULL REFERENCES sop_cycle(id),
    product_id UUID NOT NULL REFERENCES product(id),
    location_id UUID NOT NULL REFERENCES location(id),
    channel_id UUID REFERENCES channel(id),
    period_start DATE NOT NULL,
    original_quantity NUMERIC(18,4) NOT NULL,
    override_quantity NUMERIC(18,4) NOT NULL,
    override_source VARCHAR(50) NOT NULL,
    created_by UUID NOT NULL REFERENCES "user"(id),

    context JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "reason": "New marketing campaign launching in this period",
    --   "supporting_data": "Campaign reach estimate: 2M impressions",
    --   "confidence": "high",
    --   "discussion_thread_id": "thread-uuid",
    --   "attachments": ["file-uuid-1"]
    -- }

    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sop_cycle_tenant ON sop_cycle(tenant_id, cycle_month);
CREATE INDEX idx_sop_override_cycle ON sop_override(sop_cycle_id);
```

---

## Promotions & Events

```sql
CREATE TABLE promotion (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    name VARCHAR(500) NOT NULL,
    promotion_type VARCHAR(50) NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'planned',

    details JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "discount_pct": 20,
    --   "mechanic": "buy_one_get_one",
    --   "media_channels": ["tv", "digital", "in_store"],
    --   "estimated_reach": 500000,
    --   "budget": 75000,
    --   "products": ["product-uuid-1", "product-uuid-2"],
    --   "locations": ["location-uuid-1"],
    --   "channels": ["retail"],
    --   "expected_uplift_pct": 35,
    --   "actual_uplift_pct": null,
    --   "baseline_quantity": 1200,
    --   "promoted_quantity": null,
    --   "cannibalization_products": ["product-uuid-3"]
    -- }

    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_promotion_tenant ON promotion(tenant_id, start_date);
CREATE INDEX idx_promotion_details ON promotion USING GIN (details);
```

---

## External Signals

```sql
CREATE TABLE signal_source (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    name VARCHAR(255) NOT NULL,
    signal_type VARCHAR(50) NOT NULL,
    config JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "api_endpoint": "https://api.openweathermap.org/data/3.0/onecall",
    --   "api_key_secret_ref": "weather_api_key",
    --   "refresh_frequency": "daily",
    --   "metrics": ["temperature_avg", "precipitation_mm", "humidity_pct"],
    --   "geographic_scope": ["US", "DE", "JP"],
    --   "data_lag_hours": 6
    -- }
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE signal_data (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    signal_source_id UUID NOT NULL REFERENCES signal_source(id),
    location_id UUID REFERENCES location(id),
    observation_date DATE NOT NULL,
    metric_name VARCHAR(100) NOT NULL,
    metric_value NUMERIC(18,6) NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_signal_data_lookup ON signal_data(signal_source_id, observation_date);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    user_id UUID REFERENCES "user"(id),
    action VARCHAR(50) NOT NULL,
    entity_type VARCHAR(100) NOT NULL,
    entity_id UUID NOT NULL,
    changes JSONB NOT NULL,
    -- {
    --   "field_changes": { "status": { "old": "draft", "new": "approved" } },
    --   "ip_address": "203.0.113.42",
    --   "user_agent": "...",
    --   "correlation_id": "req-uuid-123"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
```

---

## JSONB Query Examples

```sql
-- Find all chilled products in the food category
SELECT id, sku, name, attributes->>'shelf_life_days' AS shelf_life
FROM product
WHERE tenant_id = '<tenant-uuid>'
  AND category = 'food'
  AND attributes @> '{"temperature_class": "chilled"}';

-- Find forecast lines where prophet was the dominant model
SELECT fl.product_id, fl.period_start, fl.quantity,
       fl.details->'ensemble_contribution'->>'prophet' AS prophet_weight
FROM forecast_line fl
WHERE fl.forecast_version_id = '<version-uuid>'
  AND (fl.details->'ensemble_contribution'->>'prophet')::numeric > 0.5;

-- Find all CPFR-capable customers
SELECT id, name, code
FROM customer
WHERE tenant_id = '<tenant-uuid>'
  AND attributes @> '{"cpfr_partner": true}';

-- Aggregate accuracy metrics plus extended metrics
SELECT
    p.family,
    AVG(fa.mape) AS avg_mape,
    AVG(fa.bias) AS avg_bias,
    AVG((fa.extended_metrics->>'tracking_signal')::numeric) AS avg_tracking_signal
FROM forecast_accuracy fa
JOIN product p ON p.id = fa.product_id
WHERE fa.tenant_id = '<tenant-uuid>'
GROUP BY p.family;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 2 | tenant, user (roles in JSONB permissions) |
| Product Dimension | 1 | Flattened hierarchy + JSONB attributes |
| Location & Geography | 1 | Country/region as columns + JSONB attributes |
| Channel & Customer | 2 | channel, customer |
| Historical Demand | 1 | demand_history with JSONB metrics |
| Forecasting | 3 | forecast_model, forecast_version, forecast_line |
| Accuracy | 1 | forecast_accuracy with JSONB extended metrics |
| S&OP Planning | 2 | sop_cycle, sop_override |
| Promotions | 1 | promotion with JSONB details (no junction table) |
| External Signals | 2 | signal_source, signal_data |
| Audit | 1 | audit_log |
| **Total** | **17** | ~40% fewer tables than normalized model |

---

## Key Design Decisions

1. **Flattened product hierarchy.** Instead of four hierarchy tables (family -> category -> subcategory -> product), the hierarchy is stored as denormalized columns on the product table. This eliminates three tables and four JOINs at the cost of some data redundancy. Hierarchy management is enforced at the application layer.

2. **Industry-specific attributes in JSONB.** The `product.attributes` JSONB field holds industry-specific fields (shelf_life_days for food, ndc_code for pharma, color/size for apparel). This allows a single product table to serve all verticals without DDL changes.

3. **GIN indexes on all JSONB columns.** PostgreSQL GIN indexes on JSONB fields enable fast containment queries (`@>` operator), making filtered queries on variable attributes performant. Example: `WHERE attributes @> '{"temperature_class": "chilled"}'`.

4. **Promotions as a single table with JSONB details.** Rather than a promotion table plus a junction table for product associations, the JSONB `details` field contains product lists, location scope, and uplift data. This trades referential integrity for simplicity — appropriate for a domain where promotion structures vary widely.

5. **Forecast line decomposition in JSONB.** Each forecast line stores demand component decomposition (base, trend, seasonal, promotional, external) and causal factor attribution in a `details` JSONB field. This enables explainable AI without requiring separate decomposition tables.

6. **Tenant configuration in JSONB.** Tenant-specific settings (planning horizon, enabled modules, jurisdiction settings, custom dimensions) are stored as a JSONB `config` field rather than a separate settings table. This keeps all tenant configuration in one queryable location.

7. **Roles as a column rather than a junction table.** For most demand planning deployments, role granularity is simple (admin, planner, manager, executive, viewer). A VARCHAR column with a JSONB permissions array replaces the role and user_role tables, reducing table count.

8. **JSON Schema validation at application layer.** All JSONB fields have documented JSON Schemas (registered in a schema registry or embedded in OpenAPI specs). Validation happens at the API layer before writes, not at the database level. This is a conscious trade-off: database-level constraints would be stronger, but JSONB CHECK constraints are limited in expressiveness.

9. **Override context as JSONB.** S&OP overrides carry a `context` JSONB field containing the planner's reasoning, supporting data, and discussion references. This is richer than a simple `reason` VARCHAR and can evolve without schema changes.

10. **~40% fewer tables than normalized model.** The hybrid approach reduces the schema from 29 tables to 17 by consolidating hierarchy tables, junction tables, and configuration tables into JSONB fields. Each eliminated table is one fewer migration to manage, one fewer ORM model, and one fewer API endpoint to build.
