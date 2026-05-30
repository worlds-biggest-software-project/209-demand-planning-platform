# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Demand Planning Platform · Created: 2026-05-20

## Philosophy

This model combines a relational core for transactional CRUD with a property graph layer for modeling the complex relationships that define a demand planning network. Inspired by o9 Solutions' Enterprise Knowledge Graph and Graph Cube architecture, it recognizes that demand planning is fundamentally a network problem: products flow through supply chains, forecasts propagate through hierarchies, promotions cannibalize across product families, and signals influence demand across geographic and temporal dimensions.

The graph layer is implemented using PostgreSQL tables (`graph_node` and `graph_edge`) rather than a separate graph database, keeping the operational stack simple while enabling relationship-rich queries. Edges carry typed relationships (SUPPLIES, CANNIBALIZES, INFLUENCES, BELONGS_TO, FORECASTS_FOR) with properties (weight, confidence, time range). This enables queries that are natural in demand planning but awkward in pure relational models: "Which products are affected if this supplier is disrupted?" or "What is the promotional cannibalization network for this product family?"

For teams that later need dedicated graph database performance (Neo4j, Amazon Neptune), the `graph_node`/`graph_edge` tables can serve as a synchronization source. But for most demand planning workloads — where the graph has thousands to hundreds of thousands of nodes rather than millions — PostgreSQL with recursive CTEs and ltree extensions handles graph traversals efficiently.

**Best for:** Organisations with complex supply chain networks, multi-level product hierarchies, supplier dependency analysis, promotional cannibalization modeling, or AI-powered demand signal propagation across interconnected planning dimensions.

**Trade-offs:**
- Pro: Natural representation of supply chain relationships (supplier -> product -> location -> customer)
- Pro: Enables impact analysis queries ("what happens if supplier X goes down?")
- Pro: Cannibalization and halo effect modeling for promotions
- Pro: Graph structure feeds AI/ML models that exploit relational features
- Pro: Flexible — new relationship types added without schema changes
- Con: Graph queries (recursive CTEs) are harder to write and optimize than flat SQL
- Con: Dual storage (relational + graph) means data synchronization overhead
- Con: Graph traversal performance degrades without careful index management
- Con: Fewer developers are familiar with graph query patterns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SCOR-DS v14.0 (ASCM) | Supply chain processes modeled as graph relationships between Plan, Source, Transform, Fulfill entities |
| W3C PROV Ontology | Graph edges align with PROV-O relationships (wasGeneratedBy, wasDerivedFrom, wasInfluencedBy) |
| GS1 General Specifications v25.0 | GTIN/GLN as node identifiers enabling cross-system graph linking |
| CPFR (GS1/VICS) | Retailer-supplier collaboration modeled as bidirectional graph edges |
| IBF Best Practices | FVA analysis traverses the graph of forecast versions linked by derivation edges |
| ISO 3166-1/2 | Geographic hierarchy modeled as graph nodes with BELONGS_TO edges |

---

## Graph Layer

```sql
-- Generic graph node table — every domain entity gets a node
CREATE TABLE graph_node (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    node_type VARCHAR(50) NOT NULL,
    -- Node types: product, location, channel, customer, supplier,
    --             forecast_version, sop_cycle, promotion, signal_source,
    --             product_family, product_category, region, country
    entity_id UUID NOT NULL,            -- FK to the corresponding relational table
    label VARCHAR(500) NOT NULL,        -- human-readable label
    properties JSONB NOT NULL DEFAULT '{}',
    -- Properties vary by node_type:
    -- product:  { "sku": "ABC-001", "gtin": "00012345678905", "lifecycle": "active" }
    -- location: { "code": "WH-NYC", "gln": "0012345000011", "type": "warehouse" }
    -- supplier: { "code": "SUP-100", "reliability_score": 0.92, "lead_time_days": 14 }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, node_type, entity_id)
);

CREATE INDEX idx_graph_node_tenant ON graph_node(tenant_id, node_type);
CREATE INDEX idx_graph_node_entity ON graph_node(entity_id);
CREATE INDEX idx_graph_node_properties ON graph_node USING GIN (properties);

-- Generic graph edge table — typed, weighted, optionally temporal
CREATE TABLE graph_edge (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    source_node_id UUID NOT NULL REFERENCES graph_node(id),
    target_node_id UUID NOT NULL REFERENCES graph_node(id),
    edge_type VARCHAR(50) NOT NULL,
    -- Edge types:
    --   BELONGS_TO        product -> subcategory -> category -> family
    --   LOCATED_AT        product -> location (stocking relationship)
    --   SOLD_VIA          product -> channel
    --   SUPPLIED_BY       product -> supplier
    --   SUPPLIES          supplier -> product
    --   CANNIBALIZES      product -> product (promotional cannibalization)
    --   HALO_EFFECT       product -> product (promotional lift on related products)
    --   SUBSTITUTE_FOR    product -> product (demand substitution)
    --   INFLUENCES        signal_source -> product/location (demand signal influence)
    --   DERIVED_FROM      forecast_version -> forecast_version (consensus derived from statistical)
    --   FORECASTS_FOR     forecast_version -> product+location (planning scope)
    --   COLLABORATES_WITH tenant -> customer (CPFR partnership)
    --   PART_OF_REGION    location -> region -> country (geographic hierarchy)

    weight NUMERIC(10,6) DEFAULT 1.0,   -- relationship strength / influence factor
    confidence NUMERIC(5,4),            -- confidence in this relationship (0-1)
    properties JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "since": "2024-01-01",
    --   "contract_end": "2027-12-31",
    --   "share_of_supply": 0.65,
    --   "cannibalization_factor": 0.15,
    --   "signal_lag_days": 3
    -- }

    valid_from DATE,                    -- temporal validity start
    valid_to DATE,                      -- temporal validity end (null = current)
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_edge_source ON graph_edge(source_node_id, edge_type);
CREATE INDEX idx_graph_edge_target ON graph_edge(target_node_id, edge_type);
CREATE INDEX idx_graph_edge_type ON graph_edge(tenant_id, edge_type);
CREATE INDEX idx_graph_edge_temporal ON graph_edge(valid_from, valid_to) WHERE is_active;
CREATE INDEX idx_graph_edge_properties ON graph_edge USING GIN (properties);
```

---

## Relational Core: Products & Locations

```sql
CREATE TABLE tenant (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    config JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE "user" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    email VARCHAR(255) NOT NULL,
    display_name VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL DEFAULT 'planner',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE product (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    sku VARCHAR(100) NOT NULL,
    gtin VARCHAR(14),
    name VARCHAR(500) NOT NULL,
    unit_of_measure VARCHAR(20) NOT NULL DEFAULT 'EA',
    lifecycle_status VARCHAR(30) NOT NULL DEFAULT 'active',
    family VARCHAR(100),
    category VARCHAR(100),
    subcategory VARCHAR(100),
    attributes JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, sku)
);

CREATE TABLE location (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    code VARCHAR(50) NOT NULL,
    name VARCHAR(255) NOT NULL,
    gln VARCHAR(13),
    location_type VARCHAR(30) NOT NULL,
    country_code CHAR(2) NOT NULL,
    region VARCHAR(100),
    timezone VARCHAR(50) NOT NULL DEFAULT 'UTC',
    attributes JSONB NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE channel (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    code VARCHAR(50) NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE customer (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    code VARCHAR(100) NOT NULL,
    name VARCHAR(500) NOT NULL,
    channel_id UUID REFERENCES channel(id),
    attributes JSONB NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE supplier (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    code VARCHAR(100) NOT NULL,
    name VARCHAR(500) NOT NULL,
    country_code CHAR(2),
    reliability_score NUMERIC(5,4),     -- 0-1, computed from delivery performance
    avg_lead_time_days INT,
    attributes JSONB NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE INDEX idx_product_tenant ON product(tenant_id);
CREATE INDEX idx_location_tenant ON location(tenant_id);
CREATE INDEX idx_supplier_tenant ON supplier(tenant_id);
```

---

## Relational Core: Demand & Forecasts

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
    source VARCHAR(50) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE forecast_version (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    name VARCHAR(255) NOT NULL,
    version_type VARCHAR(30) NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'draft',
    planning_horizon_start DATE NOT NULL,
    planning_horizon_end DATE NOT NULL,
    period_granularity VARCHAR(10) NOT NULL DEFAULT 'week',
    created_by UUID NOT NULL REFERENCES "user"(id),
    run_metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE forecast_line (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    forecast_version_id UUID NOT NULL REFERENCES forecast_version(id),
    product_id UUID NOT NULL REFERENCES product(id),
    location_id UUID NOT NULL REFERENCES location(id),
    channel_id UUID REFERENCES channel(id),
    period_start DATE NOT NULL,
    quantity NUMERIC(18,4) NOT NULL,
    confidence_lower NUMERIC(18,4),
    confidence_upper NUMERIC(18,4),
    details JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

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
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_demand_history_lookup ON demand_history(tenant_id, product_id, location_id, period_start);
CREATE INDEX idx_forecast_line_version ON forecast_line(forecast_version_id);
CREATE INDEX idx_forecast_line_product ON forecast_line(product_id, period_start);
CREATE INDEX idx_accuracy_version ON forecast_accuracy(forecast_version_id);
```

---

## Relational Core: S&OP & Promotions

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
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sop_override (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sop_cycle_id UUID NOT NULL REFERENCES sop_cycle(id),
    product_id UUID NOT NULL REFERENCES product(id),
    location_id UUID NOT NULL REFERENCES location(id),
    period_start DATE NOT NULL,
    original_quantity NUMERIC(18,4) NOT NULL,
    override_quantity NUMERIC(18,4) NOT NULL,
    override_source VARCHAR(50) NOT NULL,
    reason VARCHAR(500),
    created_by UUID NOT NULL REFERENCES "user"(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE promotion (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    name VARCHAR(500) NOT NULL,
    promotion_type VARCHAR(50) NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'planned',
    expected_uplift_pct NUMERIC(8,4),
    details JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE signal_source (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    name VARCHAR(255) NOT NULL,
    signal_type VARCHAR(50) NOT NULL,
    config JSONB NOT NULL DEFAULT '{}',
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
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    user_id UUID REFERENCES "user"(id),
    action VARCHAR(50) NOT NULL,
    entity_type VARCHAR(100) NOT NULL,
    entity_id UUID NOT NULL,
    changes JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sop_cycle_tenant ON sop_cycle(tenant_id, cycle_month);
CREATE INDEX idx_sop_override_cycle ON sop_override(sop_cycle_id);
CREATE INDEX idx_promotion_tenant ON promotion(tenant_id, start_date);
CREATE INDEX idx_signal_data_lookup ON signal_data(signal_source_id, observation_date);
CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id, created_at DESC);
```

---

## Graph Query Examples

```sql
-- ============================================================
-- 1. SUPPLIER DISRUPTION IMPACT ANALYSIS
-- "If Supplier X goes down, which products and locations are affected?"
-- ============================================================
WITH RECURSIVE impact_chain AS (
    -- Start from the disrupted supplier node
    SELECT
        gn.id AS node_id,
        gn.node_type,
        gn.label,
        gn.entity_id,
        0 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_node gn
    WHERE gn.entity_id = '<supplier-uuid>'
      AND gn.node_type = 'supplier'

    UNION ALL

    -- Traverse outgoing edges
    SELECT
        target.id,
        target.node_type,
        target.label,
        target.entity_id,
        ic.depth + 1,
        ic.path || target.id
    FROM impact_chain ic
    JOIN graph_edge ge ON ge.source_node_id = ic.node_id
    JOIN graph_node target ON target.id = ge.target_node_id
    WHERE ge.is_active = true
      AND ge.edge_type IN ('SUPPLIES', 'LOCATED_AT', 'SOLD_VIA')
      AND ic.depth < 5
      AND NOT (target.id = ANY(ic.path))  -- prevent cycles
)
SELECT node_type, label, entity_id, depth
FROM impact_chain
WHERE depth > 0
ORDER BY depth, node_type;

-- ============================================================
-- 2. PROMOTIONAL CANNIBALIZATION NETWORK
-- "What products will be cannibalized by promoting Product Y?"
-- ============================================================
SELECT
    promoted.label AS promoted_product,
    cannibalized.label AS cannibalized_product,
    ge.weight AS cannibalization_factor,
    ge.properties->>'estimated_volume_loss' AS volume_loss
FROM graph_node promoted
JOIN graph_edge ge ON ge.source_node_id = promoted.id
    AND ge.edge_type = 'CANNIBALIZES'
    AND ge.is_active = true
JOIN graph_node cannibalized ON cannibalized.id = ge.target_node_id
WHERE promoted.entity_id = '<product-uuid>'
ORDER BY ge.weight DESC;

-- ============================================================
-- 3. DEMAND SIGNAL INFLUENCE MAPPING
-- "Which external signals influence demand for products in the Beverages family?"
-- ============================================================
SELECT
    signal.label AS signal_source,
    product.label AS product,
    ge.weight AS influence_weight,
    ge.properties->>'signal_lag_days' AS lag_days,
    ge.confidence AS confidence
FROM graph_node family_node
JOIN graph_edge family_edge ON family_edge.target_node_id = family_node.id
    AND family_edge.edge_type = 'BELONGS_TO'
JOIN graph_node product ON product.id = family_edge.source_node_id
    AND product.node_type = 'product'
JOIN graph_edge ge ON ge.target_node_id = product.id
    AND ge.edge_type = 'INFLUENCES'
    AND ge.is_active = true
JOIN graph_node signal ON signal.id = ge.source_node_id
    AND signal.node_type = 'signal_source'
WHERE family_node.label = 'Beverages'
  AND family_node.tenant_id = '<tenant-uuid>'
ORDER BY ge.weight DESC;

-- ============================================================
-- 4. FORECAST DERIVATION LINEAGE
-- "Show the lineage of the final approved forecast — what was it derived from?"
-- ============================================================
WITH RECURSIVE lineage AS (
    SELECT
        gn.id AS node_id,
        gn.label,
        gn.properties->>'version_type' AS version_type,
        gn.properties->>'status' AS status,
        0 AS depth
    FROM graph_node gn
    WHERE gn.entity_id = '<final-forecast-uuid>'
      AND gn.node_type = 'forecast_version'

    UNION ALL

    SELECT
        parent.id,
        parent.label,
        parent.properties->>'version_type',
        parent.properties->>'status',
        l.depth + 1
    FROM lineage l
    JOIN graph_edge ge ON ge.source_node_id = l.node_id
        AND ge.edge_type = 'DERIVED_FROM'
    JOIN graph_node parent ON parent.id = ge.target_node_id
    WHERE l.depth < 10
)
SELECT * FROM lineage ORDER BY depth;

-- ============================================================
-- 5. PRODUCT SUBSTITUTION RECOMMENDATIONS
-- "If Product Z stocks out, what substitutes are available at Location W?"
-- ============================================================
SELECT
    substitute.label AS substitute_product,
    ge.weight AS substitution_score,
    ge.properties->>'substitution_type' AS sub_type,
    loc_edge.properties->>'current_stock' AS stock_at_location
FROM graph_node product_node
JOIN graph_edge ge ON ge.source_node_id = product_node.id
    AND ge.edge_type = 'SUBSTITUTE_FOR'
    AND ge.is_active = true
JOIN graph_node substitute ON substitute.id = ge.target_node_id
-- Check the substitute is stocked at the target location
JOIN graph_edge loc_edge ON loc_edge.source_node_id = substitute.id
    AND loc_edge.edge_type = 'LOCATED_AT'
    AND loc_edge.is_active = true
JOIN graph_node loc ON loc.id = loc_edge.target_node_id
    AND loc.entity_id = '<location-uuid>'
WHERE product_node.entity_id = '<product-uuid>'
ORDER BY ge.weight DESC;
```

---

## Graph Synchronization Trigger

```sql
-- Automatically create/update graph nodes when relational entities change
CREATE OR REPLACE FUNCTION sync_product_to_graph()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO graph_node (tenant_id, node_type, entity_id, label, properties)
    VALUES (
        NEW.tenant_id,
        'product',
        NEW.id,
        NEW.name,
        jsonb_build_object(
            'sku', NEW.sku,
            'gtin', NEW.gtin,
            'lifecycle', NEW.lifecycle_status,
            'family', NEW.family,
            'category', NEW.category,
            'subcategory', NEW.subcategory
        )
    )
    ON CONFLICT (tenant_id, node_type, entity_id)
    DO UPDATE SET
        label = EXCLUDED.label,
        properties = EXCLUDED.properties,
        updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_product_graph_sync
    AFTER INSERT OR UPDATE ON product
    FOR EACH ROW EXECUTE FUNCTION sync_product_to_graph();

-- Similar triggers for location, supplier, customer, forecast_version, etc.
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_node, graph_edge |
| Identity & Multi-Tenancy | 2 | tenant, user |
| Product & Supply Chain | 3 | product, supplier, location |
| Channel & Customer | 2 | channel, customer |
| Historical Demand | 1 | demand_history |
| Forecasting | 3 | forecast_version, forecast_line, forecast_accuracy |
| S&OP Planning | 2 | sop_cycle, sop_override |
| Promotions & Signals | 3 | promotion, signal_source, signal_data |
| Audit | 1 | audit_log |
| **Total** | **19 relational + 2 graph = 21** | Graph layer adds only 2 tables |

---

## Key Design Decisions

1. **Generic graph tables rather than dedicated relationship tables.** Instead of separate `product_supplier`, `product_location`, `product_substitution` junction tables, all relationships flow through the two graph tables. This keeps the relational schema clean while enabling unlimited relationship types without DDL changes.

2. **Temporal edges with valid_from/valid_to.** Graph edges carry temporal validity, enabling historical network analysis. "Who supplied this product in Q1 2025?" is a natural query. This is critical for demand planning where supplier relationships, stocking decisions, and channel assignments change over time.

3. **Weighted edges for influence modeling.** The `weight` column on edges quantifies relationship strength. For demand signals, it represents correlation coefficient. For cannibalization, it represents the fraction of demand redirected. For supply, it represents share of supply. AI models can learn and update these weights.

4. **Dual storage with trigger synchronization.** Relational tables remain the system of record for CRUD operations. Triggers automatically synchronize changes to graph nodes. This ensures the graph is always current without requiring application code to dual-write.

5. **Supplier as a first-class entity.** Unlike the other data model suggestions, this model includes a dedicated `supplier` table because supplier relationships are central to graph-based impact analysis. Supplier disruption scenarios ("what if this supplier goes down?") are a key use case.

6. **Cannibalization and halo effects as graph edges.** Promotional cannibalization (Product A promotion reduces Product B demand) and halo effects (Product A promotion increases Product C demand) are modeled as typed, weighted edges between product nodes. This enables network-aware promotional planning.

7. **Forecast derivation lineage.** The DERIVED_FROM edge type creates a directed acyclic graph of forecast versions, showing how the final approved plan was derived from statistical forecasts through commercial overrides to consensus. This provides built-in FVA lineage tracking.

8. **Signal influence as a learnable graph.** External signal influences on products are modeled as INFLUENCES edges with weights and lag properties. AI models can discover and update these edges automatically, creating a self-improving demand signal network.

9. **Path-based queries for impact analysis.** Recursive CTE queries traverse the graph to answer network questions. The depth limit (typically 5) prevents runaway traversals. The `path` array prevents cycles. These queries are the primary differentiation over flat relational models.

10. **Migration path to dedicated graph database.** If graph traversal performance becomes a bottleneck, the `graph_node` and `graph_edge` tables can be exported to Neo4j or Amazon Neptune. The relational core continues to serve CRUD operations, and the graph database handles complex traversals. The generic table structure maps directly to property graph databases.
