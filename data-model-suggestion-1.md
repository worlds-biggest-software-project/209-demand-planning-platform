# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Demand Planning Platform · Created: 2026-05-20

## Philosophy

This model follows traditional normalized relational design with a dimensional (snowflake) schema for analytical queries. Every domain concept — products, locations, channels, forecasts, plans, accuracy metrics — gets its own table with strict foreign key relationships. The schema enforces referential integrity at the database level and uses GS1-aligned identifiers (GTIN for products, GLN for locations) as canonical keys.

The approach draws on proven patterns from ERP systems (SAP, Oracle) and data warehousing best practices where supply chain dimensions are normalized into hierarchies (SKU -> Subcategory -> Category -> Family, Location -> Region -> Country). This mirrors how SCOR-DS structures its Plan domain entities and how IBF metrics (MAPE, bias, WAPE, FVA) are computed across dimensional intersections.

This is the most straightforward model for teams with strong SQL skills and existing relational database infrastructure. It provides excellent query flexibility, strong data integrity, and straightforward compliance with audit requirements through standard database constraints and triggers.

**Best for:** Teams building a production-grade platform where data integrity, regulatory compliance, and complex cross-dimensional analytics are paramount.

**Trade-offs:**
- Pro: Strong referential integrity enforced at database level
- Pro: Standard SQL queries work naturally across all dimensions
- Pro: Well-understood by most development teams
- Pro: Easy to align with SCOR-DS and GS1 standards
- Con: High table count increases schema complexity and migration burden
- Con: Schema changes require DDL migrations for every new attribute
- Con: Junction tables for many-to-many relationships add query complexity
- Con: Less flexible for jurisdiction-specific or customer-specific fields

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 General Specifications v25.0 | GTIN used as canonical product identifier; GLN used for location identification |
| SCOR-DS v14.0 (ASCM) | Plan domain entities mapped to `demand_plan`, `consensus_plan`, `forecast_version` tables |
| IBF Best Practices | MAPE, bias, WAPE, FVA metrics modeled as first-class entities in `forecast_accuracy` |
| CPFR (GS1/VICS) | Retailer-supplier collaboration modeled through `collaboration_partner` and `shared_forecast` |
| ISO 3166-1/2 | Country and subdivision codes used in `location` hierarchy |
| ISO 8601 | All temporal fields use TIMESTAMPTZ; planning periods use ISO week/month conventions |
| OpenAPI 3.1.0 | Table structure designed to map 1:1 to REST API resource representations |
| OAuth 2.0 / OIDC | `tenant`, `user`, `role` tables support multi-tenant RBAC with SSO |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE tenant (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    settings JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE "user" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    email VARCHAR(255) NOT NULL,
    display_name VARCHAR(255) NOT NULL,
    oidc_subject VARCHAR(500),          -- OpenID Connect subject identifier
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE role (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    name VARCHAR(100) NOT NULL,         -- e.g. 'demand_planner', 'sop_manager', 'admin'
    permissions JSONB NOT NULL DEFAULT '[]',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE user_role (
    user_id UUID NOT NULL REFERENCES "user"(id),
    role_id UUID NOT NULL REFERENCES role(id),
    assigned_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id)
);

CREATE INDEX idx_user_tenant ON "user"(tenant_id);
CREATE INDEX idx_user_role_user ON user_role(user_id);
```

---

## Product Hierarchy

```sql
CREATE TABLE product_family (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    name VARCHAR(255) NOT NULL,
    code VARCHAR(50) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE product_category (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    family_id UUID NOT NULL REFERENCES product_family(id),
    name VARCHAR(255) NOT NULL,
    code VARCHAR(50) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE product_subcategory (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    category_id UUID NOT NULL REFERENCES product_category(id),
    name VARCHAR(255) NOT NULL,
    code VARCHAR(50) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE product (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    subcategory_id UUID NOT NULL REFERENCES product_subcategory(id),
    sku VARCHAR(100) NOT NULL,
    gtin VARCHAR(14),                   -- GS1 Global Trade Item Number
    name VARCHAR(500) NOT NULL,
    description TEXT,
    unit_of_measure VARCHAR(20) NOT NULL DEFAULT 'EA',  -- EA, CS, PL, KG, etc.
    lifecycle_status VARCHAR(30) NOT NULL DEFAULT 'active',  -- active, new, discontinued
    launch_date DATE,
    discontinuation_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, sku)
);

CREATE INDEX idx_product_tenant ON product(tenant_id);
CREATE INDEX idx_product_gtin ON product(gtin) WHERE gtin IS NOT NULL;
CREATE INDEX idx_product_subcategory ON product(subcategory_id);
CREATE INDEX idx_product_lifecycle ON product(tenant_id, lifecycle_status);
```

---

## Location Hierarchy

```sql
CREATE TABLE country (
    code CHAR(2) PRIMARY KEY,           -- ISO 3166-1 alpha-2
    name VARCHAR(255) NOT NULL,
    numeric_code CHAR(3)                -- ISO 3166-1 numeric
);

CREATE TABLE region (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    country_code CHAR(2) NOT NULL REFERENCES country(code),
    name VARCHAR(255) NOT NULL,
    code VARCHAR(50) NOT NULL,
    UNIQUE (tenant_id, code)
);

CREATE TABLE location (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    region_id UUID NOT NULL REFERENCES region(id),
    gln VARCHAR(13),                    -- GS1 Global Location Number
    name VARCHAR(255) NOT NULL,
    code VARCHAR(50) NOT NULL,
    location_type VARCHAR(30) NOT NULL, -- warehouse, store, dc, plant, virtual
    timezone VARCHAR(50) NOT NULL DEFAULT 'UTC',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE INDEX idx_location_tenant ON location(tenant_id);
CREATE INDEX idx_location_region ON location(region_id);
CREATE INDEX idx_location_gln ON location(gln) WHERE gln IS NOT NULL;
```

---

## Channel & Customer Dimensions

```sql
CREATE TABLE channel (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    name VARCHAR(255) NOT NULL,         -- e.g. 'Retail', 'eCommerce', 'Wholesale', 'DTC'
    code VARCHAR(50) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE customer (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    name VARCHAR(500) NOT NULL,
    code VARCHAR(100) NOT NULL,
    channel_id UUID REFERENCES channel(id),
    country_code CHAR(2) REFERENCES country(code),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE INDEX idx_customer_tenant ON customer(tenant_id);
CREATE INDEX idx_customer_channel ON customer(channel_id);
```

---

## Historical Demand (Actuals)

```sql
CREATE TABLE demand_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    product_id UUID NOT NULL REFERENCES product(id),
    location_id UUID NOT NULL REFERENCES location(id),
    channel_id UUID REFERENCES channel(id),
    customer_id UUID REFERENCES customer(id),
    period_start DATE NOT NULL,         -- start of the time bucket (day, week, or month)
    period_granularity VARCHAR(10) NOT NULL DEFAULT 'week',  -- day, week, month
    quantity NUMERIC(18,4) NOT NULL,
    revenue NUMERIC(18,4),
    unit_of_measure VARCHAR(20) NOT NULL DEFAULT 'EA',
    is_outlier BOOLEAN NOT NULL DEFAULT false,
    outlier_reason VARCHAR(255),
    cleansed_quantity NUMERIC(18,4),    -- quantity after outlier correction
    source VARCHAR(50) NOT NULL,        -- erp_import, pos_feed, manual, api
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_demand_history_unique ON demand_history(
    tenant_id, product_id, location_id, COALESCE(channel_id, '00000000-0000-0000-0000-000000000000'),
    COALESCE(customer_id, '00000000-0000-0000-0000-000000000000'), period_start, period_granularity
);
CREATE INDEX idx_demand_history_product ON demand_history(product_id, period_start);
CREATE INDEX idx_demand_history_period ON demand_history(tenant_id, period_start);
```

---

## Forecasting Engine

```sql
CREATE TABLE forecast_model (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    name VARCHAR(255) NOT NULL,
    algorithm VARCHAR(100) NOT NULL,    -- arima, ets, prophet, xgboost, lstm, ensemble
    parameters JSONB NOT NULL DEFAULT '{}',
    is_default BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE forecast_version (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    name VARCHAR(255) NOT NULL,
    version_type VARCHAR(30) NOT NULL,  -- statistical, demand_sensing, consensus, final
    status VARCHAR(30) NOT NULL DEFAULT 'draft',  -- draft, reviewed, approved, superseded
    planning_horizon_start DATE NOT NULL,
    planning_horizon_end DATE NOT NULL,
    period_granularity VARCHAR(10) NOT NULL DEFAULT 'week',
    created_by UUID NOT NULL REFERENCES "user"(id),
    approved_by UUID REFERENCES "user"(id),
    approved_at TIMESTAMPTZ,
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
    confidence_lower NUMERIC(18,4),     -- lower bound of confidence interval
    confidence_upper NUMERIC(18,4),     -- upper bound of confidence interval
    confidence_level NUMERIC(5,4),      -- e.g. 0.9500 for 95%
    forecast_model_id UUID REFERENCES forecast_model(id),
    unit_of_measure VARCHAR(20) NOT NULL DEFAULT 'EA',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_forecast_line_version ON forecast_line(forecast_version_id);
CREATE INDEX idx_forecast_line_product ON forecast_line(product_id, period_start);
CREATE INDEX idx_forecast_line_period ON forecast_line(forecast_version_id, period_start);
```

---

## Forecast Accuracy & FVA

```sql
CREATE TABLE forecast_accuracy (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    forecast_version_id UUID NOT NULL REFERENCES forecast_version(id),
    product_id UUID NOT NULL REFERENCES product(id),
    location_id UUID NOT NULL REFERENCES location(id),
    channel_id UUID REFERENCES channel(id),
    period_start DATE NOT NULL,
    actual_quantity NUMERIC(18,4) NOT NULL,
    forecast_quantity NUMERIC(18,4) NOT NULL,
    mape NUMERIC(10,4),                 -- Mean Absolute Percentage Error (IBF standard)
    bias NUMERIC(10,4),                 -- Forecast bias / MPE (IBF standard)
    wape NUMERIC(10,4),                 -- Weighted Absolute Percentage Error
    absolute_error NUMERIC(18,4),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE forecast_value_added (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    planning_cycle_id UUID NOT NULL,    -- references the S&OP cycle
    step_name VARCHAR(100) NOT NULL,    -- 'naive_baseline', 'statistical', 'planner_override', 'consensus'
    step_order INT NOT NULL,
    product_id UUID REFERENCES product(id),
    location_id UUID REFERENCES location(id),
    period_start DATE NOT NULL,
    mape_at_step NUMERIC(10,4) NOT NULL,
    mape_improvement NUMERIC(10,4),     -- improvement vs previous step
    is_value_adding BOOLEAN NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_accuracy_version ON forecast_accuracy(forecast_version_id);
CREATE INDEX idx_accuracy_product ON forecast_accuracy(product_id, period_start);
CREATE INDEX idx_fva_cycle ON forecast_value_added(planning_cycle_id);
```

---

## S&OP / Consensus Planning

```sql
CREATE TABLE sop_cycle (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    name VARCHAR(255) NOT NULL,         -- e.g. 'June 2026 S&OP'
    cycle_month DATE NOT NULL,          -- first day of the planning month
    status VARCHAR(30) NOT NULL DEFAULT 'statistical_review',
    -- Workflow states: statistical_review -> commercial_override ->
    --                  consensus_review -> executive_approval -> closed
    statistical_forecast_id UUID REFERENCES forecast_version(id),
    consensus_forecast_id UUID REFERENCES forecast_version(id),
    final_forecast_id UUID REFERENCES forecast_version(id),
    owner_id UUID NOT NULL REFERENCES "user"(id),
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
    override_reason VARCHAR(500) NOT NULL,
    override_source VARCHAR(50) NOT NULL,  -- sales_input, marketing_input, finance_input, planner
    created_by UUID NOT NULL REFERENCES "user"(id),
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
    promotion_type VARCHAR(50) NOT NULL,  -- price_discount, bogo, seasonal, new_launch
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'planned',  -- planned, active, completed, cancelled
    expected_uplift_pct NUMERIC(8,4),
    actual_uplift_pct NUMERIC(8,4),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE promotion_product (
    promotion_id UUID NOT NULL REFERENCES promotion(id),
    product_id UUID NOT NULL REFERENCES product(id),
    location_id UUID REFERENCES location(id),
    channel_id UUID REFERENCES channel(id),
    expected_uplift_quantity NUMERIC(18,4),
    PRIMARY KEY (promotion_id, product_id)
);

CREATE INDEX idx_promotion_tenant ON promotion(tenant_id, start_date);
```

---

## External Signals

```sql
CREATE TABLE signal_source (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    name VARCHAR(255) NOT NULL,         -- e.g. 'OpenWeather', 'Google Trends', 'BLS CPI'
    signal_type VARCHAR(50) NOT NULL,   -- weather, economic, social, web_traffic, pos
    api_endpoint VARCHAR(1000),
    refresh_frequency VARCHAR(20) NOT NULL DEFAULT 'daily',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE signal_data (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    signal_source_id UUID NOT NULL REFERENCES signal_source(id),
    location_id UUID REFERENCES location(id),
    observation_date DATE NOT NULL,
    metric_name VARCHAR(100) NOT NULL,  -- temperature_avg, cpi_index, search_volume
    metric_value NUMERIC(18,6) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_signal_data_source ON signal_data(signal_source_id, observation_date);
CREATE INDEX idx_signal_data_date ON signal_data(observation_date);
```

---

## Collaboration Partners (CPFR)

```sql
CREATE TABLE collaboration_partner (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    name VARCHAR(500) NOT NULL,
    partner_type VARCHAR(30) NOT NULL,  -- retailer, supplier, distributor
    gln VARCHAR(13),                    -- GS1 Global Location Number
    contact_email VARCHAR(255),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE shared_forecast (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    partner_id UUID NOT NULL REFERENCES collaboration_partner(id),
    forecast_version_id UUID NOT NULL REFERENCES forecast_version(id),
    shared_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    share_format VARCHAR(30) NOT NULL DEFAULT 'json',  -- json, csv, edi_830, edi_delfor
    status VARCHAR(30) NOT NULL DEFAULT 'sent',  -- sent, acknowledged, accepted, rejected
    partner_response JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_shared_forecast_partner ON shared_forecast(partner_id);
```

---

## Planner Alerts & Exceptions

```sql
CREATE TABLE alert (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    alert_type VARCHAR(50) NOT NULL,    -- stockout_risk, forecast_deviation, bias_threshold, accuracy_drop
    severity VARCHAR(20) NOT NULL,      -- critical, warning, info
    product_id UUID REFERENCES product(id),
    location_id UUID REFERENCES location(id),
    channel_id UUID REFERENCES channel(id),
    title VARCHAR(500) NOT NULL,
    description TEXT,
    threshold_value NUMERIC(18,4),
    actual_value NUMERIC(18,4),
    is_acknowledged BOOLEAN NOT NULL DEFAULT false,
    acknowledged_by UUID REFERENCES "user"(id),
    acknowledged_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alert_tenant ON alert(tenant_id, created_at DESC);
CREATE INDEX idx_alert_unacknowledged ON alert(tenant_id, is_acknowledged) WHERE NOT is_acknowledged;
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    user_id UUID REFERENCES "user"(id),
    action VARCHAR(50) NOT NULL,        -- create, update, delete, approve, override, share
    entity_type VARCHAR(100) NOT NULL,  -- forecast_version, sop_cycle, product, etc.
    entity_id UUID NOT NULL,
    changes JSONB,                      -- { "field": { "old": ..., "new": ... } }
    ip_address INET,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 4 | tenant, user, role, user_role |
| Product Hierarchy | 4 | product_family, product_category, product_subcategory, product |
| Location Hierarchy | 3 | country, region, location |
| Channel & Customer | 2 | channel, customer |
| Historical Demand | 1 | demand_history |
| Forecasting Engine | 3 | forecast_model, forecast_version, forecast_line |
| Accuracy & FVA | 2 | forecast_accuracy, forecast_value_added |
| S&OP Planning | 2 | sop_cycle, sop_override |
| Promotions | 2 | promotion, promotion_product |
| External Signals | 2 | signal_source, signal_data |
| Collaboration (CPFR) | 2 | collaboration_partner, shared_forecast |
| Alerts | 1 | alert |
| Audit | 1 | audit_log |
| **Total** | **29** | |

---

## Key Design Decisions

1. **GS1 identifiers as optional canonical keys.** Products carry an optional `gtin` field and locations an optional `gln` field, enabling interoperability with GS1-compliant supply chains without mandating them for all tenants.

2. **Explicit product hierarchy tables rather than self-referential tree.** Four-level hierarchy (family -> category -> subcategory -> product) is modeled as separate tables to enforce consistent depth and enable efficient aggregation queries at each level without recursive CTEs.

3. **Forecast versions as immutable snapshots.** Each `forecast_version` captures a complete point-in-time forecast. Versions are never updated in place; instead, new versions are created. The S&OP cycle links to statistical, consensus, and final versions explicitly.

4. **Confidence intervals stored per forecast line.** Probabilistic forecasting is supported by storing lower/upper bounds and confidence level alongside point estimates, rather than requiring a separate probability distribution table.

5. **FVA as a first-class entity.** Forecast Value Added is modeled as its own table tracking accuracy improvement at each S&OP process step, aligned with IBF best practice methodology.

6. **Multi-tenant row-level isolation.** Every business table carries a `tenant_id` foreign key. Row-level security (RLS) policies can be applied using `SET app.current_tenant = '...'` in PostgreSQL.

7. **Separate outlier tracking in demand history.** Rather than discarding or silently correcting outliers, the schema preserves both original and cleansed quantities with an outlier flag and reason, supporting audit and model training decisions.

8. **EDI format support via shared forecast.** The `shared_forecast` table supports multiple export formats including EDI 830 and EDIFACT DELFOR alongside modern JSON, bridging legacy and modern integration patterns per the CPFR standard.

9. **External signals decoupled from forecast lines.** Signal data is stored separately and linked by location and date rather than embedded in forecast lines, allowing signal sources to be added, removed, or reweighted without schema changes.

10. **ISO 3166 country reference table.** Countries are stored as a static reference table with ISO alpha-2 codes, enabling standards-compliant geography modeling for multi-region deployments.
