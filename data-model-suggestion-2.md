# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Demand Planning Platform · Created: 2026-05-20

## Philosophy

This model treats every change to the demand planning system as an immutable event stored in an append-only event store. The current state of any forecast, plan, or override is derived by replaying events rather than reading from mutable rows. A CQRS (Command Query Responsibility Segregation) pattern separates the write path (event store) from the read path (materialized views optimized for planner queries).

This approach is inspired by financial ledger systems where every transaction is immutable and auditable. In demand planning, this is particularly powerful because planners and executives frequently need to answer temporal questions: "What did we forecast for Q3 back in March?" or "Who changed the consensus number for SKU X and why?" Event sourcing answers these questions natively by preserving the complete history of every planning decision, override, and model run.

PostgreSQL serves as both the event store (append-only tables with strict ordering) and the read database (materialized views refreshed from events). This avoids the operational complexity of separate event store infrastructure while leveraging PostgreSQL's LISTEN/NOTIFY for event-driven view updates.

**Best for:** Organisations where full audit trails, temporal queries ("what was true on date X?"), and regulatory compliance are critical requirements — particularly those operating in regulated industries or undergoing frequent S&OP process audits.

**Trade-offs:**
- Pro: Complete, immutable audit trail of every planning decision
- Pro: Native temporal queries — reconstruct any past state by replaying events to a point in time
- Pro: Naturally supports FVA analysis by tracking every forecast step as an event
- Pro: Event replay enables AI training on historical planning patterns
- Con: Higher write amplification — every change generates an event + view update
- Con: Read model rebuilds can be slow for large event histories
- Con: More complex application code — developers must think in events, not CRUD
- Con: Schema evolution requires event versioning strategies (upcasting)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SCOR-DS v14.0 (ASCM) | Plan domain events mapped to SCOR process stages (Plan, Forecast, Consensus) |
| IBF Best Practices | FVA computed by replaying forecast events at each process step and comparing accuracy |
| GS1 General Specifications v25.0 | GTIN and GLN stored as immutable attributes in entity-created events |
| CPFR (GS1/VICS) | Retailer-supplier forecast sharing modeled as collaboration events |
| ISO 8601 | Event timestamps use TIMESTAMPTZ; planning periods use ISO conventions |
| W3C PROV Ontology | Event metadata aligns with PROV-O concepts (wasGeneratedBy, wasAttributedTo) |

---

## Event Store Core

```sql
-- The immutable event store — the single source of truth
CREATE TABLE event_store (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    stream_id UUID NOT NULL,            -- aggregate root identifier
    stream_type VARCHAR(100) NOT NULL,  -- 'forecast', 'sop_cycle', 'product', 'demand_history'
    event_type VARCHAR(200) NOT NULL,   -- e.g. 'ForecastCreated', 'OverrideApplied', 'ConsensusApproved'
    event_version INT NOT NULL,         -- schema version for this event type (for upcasting)
    sequence_number BIGINT NOT NULL,    -- ordering within the stream
    payload JSONB NOT NULL,             -- event-specific data
    metadata JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {"user_id": "...", "ip_address": "...", "correlation_id": "...", "causation_id": "..."}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_number)
);

-- Global ordering for cross-stream projections
CREATE SEQUENCE event_global_position;
ALTER TABLE event_store ADD COLUMN global_position BIGINT NOT NULL DEFAULT nextval('event_global_position');

CREATE INDEX idx_event_store_stream ON event_store(stream_id, sequence_number);
CREATE INDEX idx_event_store_type ON event_store(stream_type, event_type, created_at);
CREATE INDEX idx_event_store_tenant ON event_store(tenant_id, created_at);
CREATE INDEX idx_event_store_global ON event_store(global_position);
```

---

## Event Type Catalogue

```sql
-- Registry of all known event types and their JSON schemas
CREATE TABLE event_type_registry (
    event_type VARCHAR(200) PRIMARY KEY,
    stream_type VARCHAR(100) NOT NULL,
    description TEXT NOT NULL,
    payload_schema JSONB NOT NULL,      -- JSON Schema for validating event payloads
    current_version INT NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event types for demand planning:
--
-- Stream: product
--   ProductCreated        { sku, gtin, name, subcategory_id, unit_of_measure }
--   ProductUpdated        { fields: { name: { old, new } } }
--   ProductDiscontinued   { discontinuation_date, reason }
--
-- Stream: demand_history
--   DemandRecorded        { product_id, location_id, channel_id, period_start, quantity, revenue, source }
--   OutlierFlagged        { demand_id, reason, cleansed_quantity }
--   DemandCorrected       { demand_id, original_quantity, corrected_quantity, reason }
--
-- Stream: forecast
--   ForecastGenerated     { model_id, algorithm, parameters, line_count, horizon_start, horizon_end }
--   ForecastLineProduced  { product_id, location_id, period_start, quantity, confidence_lower, confidence_upper }
--   ForecastReviewed      { reviewer_id, status, comments }
--   OverrideApplied       { product_id, location_id, period_start, original_qty, override_qty, reason, source }
--   OverrideReverted      { override_event_id, reason }
--   BiasDetected          { product_id, location_id, bias_value, bias_direction, auto_corrected }
--   BiasCorrected         { product_id, location_id, correction_factor, applied_to_periods }
--
-- Stream: sop_cycle
--   SopCycleOpened        { cycle_month, owner_id }
--   StatisticalReviewCompleted  { forecast_id, reviewer_id }
--   CommercialInputReceived     { user_id, override_count, source }
--   ConsensusReached      { consensus_forecast_id, participants, disagreements_resolved }
--   ExecutiveApproved     { approver_id, final_forecast_id }
--   SopCycleClosed        { closed_by, summary_metrics }
--
-- Stream: collaboration
--   ForecastShared        { partner_id, forecast_id, format, recipient }
--   PartnerResponseReceived  { partner_id, status, response_data }
```

---

## Read Model: Product Dimension (Materialized View)

```sql
-- Materialized from ProductCreated / ProductUpdated / ProductDiscontinued events
CREATE TABLE rm_product (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    sku VARCHAR(100) NOT NULL,
    gtin VARCHAR(14),
    name VARCHAR(500) NOT NULL,
    description TEXT,
    unit_of_measure VARCHAR(20) NOT NULL,
    family_name VARCHAR(255),
    category_name VARCHAR(255),
    subcategory_name VARCHAR(255),
    lifecycle_status VARCHAR(30) NOT NULL,
    launch_date DATE,
    discontinuation_date DATE,
    last_event_id UUID NOT NULL,
    last_updated TIMESTAMPTZ NOT NULL
);

CREATE UNIQUE INDEX idx_rm_product_sku ON rm_product(tenant_id, sku);
CREATE INDEX idx_rm_product_tenant ON rm_product(tenant_id);
```

---

## Read Model: Current Forecast

```sql
-- Materialized from ForecastGenerated / ForecastLineProduced / OverrideApplied events
CREATE TABLE rm_current_forecast (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    forecast_stream_id UUID NOT NULL,   -- references the forecast event stream
    version_type VARCHAR(30) NOT NULL,  -- statistical, demand_sensing, consensus, final
    status VARCHAR(30) NOT NULL,
    product_id UUID NOT NULL,
    location_id UUID NOT NULL,
    channel_id UUID,
    period_start DATE NOT NULL,
    quantity NUMERIC(18,4) NOT NULL,
    confidence_lower NUMERIC(18,4),
    confidence_upper NUMERIC(18,4),
    confidence_level NUMERIC(5,4),
    algorithm VARCHAR(100),
    has_override BOOLEAN NOT NULL DEFAULT false,
    override_quantity NUMERIC(18,4),
    override_reason VARCHAR(500),
    last_event_id UUID NOT NULL,
    last_updated TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_forecast_tenant ON rm_current_forecast(tenant_id, version_type);
CREATE INDEX idx_rm_forecast_product ON rm_current_forecast(product_id, period_start);
CREATE INDEX idx_rm_forecast_stream ON rm_current_forecast(forecast_stream_id);
```

---

## Read Model: Forecast Accuracy

```sql
-- Materialized by comparing ForecastLineProduced events with DemandRecorded events
CREATE TABLE rm_forecast_accuracy (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    forecast_stream_id UUID NOT NULL,
    product_id UUID NOT NULL,
    location_id UUID NOT NULL,
    period_start DATE NOT NULL,
    forecast_quantity NUMERIC(18,4) NOT NULL,
    actual_quantity NUMERIC(18,4) NOT NULL,
    mape NUMERIC(10,4),
    bias NUMERIC(10,4),
    wape NUMERIC(10,4),
    absolute_error NUMERIC(18,4),
    last_updated TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_accuracy_product ON rm_forecast_accuracy(product_id, period_start);
```

---

## Read Model: FVA Analysis

```sql
-- Computed by replaying forecast events at each S&OP step
CREATE TABLE rm_fva_analysis (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    sop_cycle_stream_id UUID NOT NULL,
    step_name VARCHAR(100) NOT NULL,
    step_order INT NOT NULL,
    product_id UUID,
    location_id UUID,
    period_start DATE NOT NULL,
    mape_at_step NUMERIC(10,4) NOT NULL,
    mape_improvement NUMERIC(10,4),
    is_value_adding BOOLEAN NOT NULL,
    computed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_fva_cycle ON rm_fva_analysis(sop_cycle_stream_id);
```

---

## Read Model: S&OP Cycle Status

```sql
CREATE TABLE rm_sop_cycle (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    cycle_month DATE NOT NULL,
    status VARCHAR(30) NOT NULL,
    owner_id UUID NOT NULL,
    statistical_forecast_id UUID,
    consensus_forecast_id UUID,
    final_forecast_id UUID,
    override_count INT NOT NULL DEFAULT 0,
    total_disagreements INT NOT NULL DEFAULT 0,
    started_at TIMESTAMPTZ,
    closed_at TIMESTAMPTZ,
    last_event_id UUID NOT NULL,
    last_updated TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_sop_tenant ON rm_sop_cycle(tenant_id, cycle_month);
```

---

## Read Model: Alerts

```sql
CREATE TABLE rm_alert (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    alert_type VARCHAR(50) NOT NULL,
    severity VARCHAR(20) NOT NULL,
    product_id UUID,
    location_id UUID,
    title VARCHAR(500) NOT NULL,
    description TEXT,
    threshold_value NUMERIC(18,4),
    actual_value NUMERIC(18,4),
    is_acknowledged BOOLEAN NOT NULL DEFAULT false,
    acknowledged_by UUID,
    source_event_id UUID NOT NULL,
    created_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_alert_unack ON rm_alert(tenant_id, is_acknowledged) WHERE NOT is_acknowledged;
```

---

## Projection Checkpoint (for rebuilding read models)

```sql
CREATE TABLE projection_checkpoint (
    projection_name VARCHAR(200) PRIMARY KEY,
    last_processed_position BIGINT NOT NULL DEFAULT 0,
    last_processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    status VARCHAR(30) NOT NULL DEFAULT 'running',  -- running, paused, rebuilding
    error_message TEXT
);

-- Projections:
-- 'rm_product'           -> processes product stream events
-- 'rm_current_forecast'  -> processes forecast stream events
-- 'rm_forecast_accuracy' -> processes forecast + demand_history events
-- 'rm_fva_analysis'      -> processes sop_cycle + forecast events
-- 'rm_sop_cycle'         -> processes sop_cycle events
-- 'rm_alert'             -> processes cross-stream exception events
```

---

## Snapshot Store (for performance)

```sql
-- Periodic snapshots to avoid replaying entire event history
CREATE TABLE event_snapshot (
    stream_id UUID NOT NULL,
    stream_type VARCHAR(100) NOT NULL,
    snapshot_version INT NOT NULL,
    sequence_number BIGINT NOT NULL,    -- event sequence at time of snapshot
    state JSONB NOT NULL,               -- serialized aggregate state
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);

CREATE INDEX idx_snapshot_stream ON event_snapshot(stream_id, sequence_number DESC);
```

---

## Temporal Query Examples

```sql
-- "What was the consensus forecast for product X in location Y
--  as of March 15, 2026?"
SELECT payload
FROM event_store
WHERE stream_type = 'forecast'
  AND event_type IN ('ForecastLineProduced', 'OverrideApplied')
  AND created_at <= '2026-03-15T23:59:59Z'
  AND payload->>'product_id' = '<product-uuid>'
  AND payload->>'location_id' = '<location-uuid>'
ORDER BY sequence_number ASC;
-- Application replays these events to reconstruct the forecast state at that point in time.

-- "Show me all overrides applied during the May 2026 S&OP cycle"
SELECT
    e.created_at,
    e.payload->>'product_id' AS product_id,
    e.payload->>'original_qty' AS original,
    e.payload->>'override_qty' AS overridden,
    e.payload->>'reason' AS reason,
    e.metadata->>'user_id' AS changed_by
FROM event_store e
WHERE e.stream_type = 'forecast'
  AND e.event_type = 'OverrideApplied'
  AND e.stream_id IN (
      SELECT stream_id FROM event_store
      WHERE stream_type = 'sop_cycle'
        AND event_type = 'SopCycleOpened'
        AND payload->>'cycle_month' = '2026-05-01'
  )
ORDER BY e.created_at;

-- "What is the cumulative FVA for each step in the last 6 S&OP cycles?"
SELECT
    step_name,
    step_order,
    AVG(mape_improvement) AS avg_mape_improvement,
    COUNT(*) FILTER (WHERE is_value_adding) * 100.0 / COUNT(*) AS pct_value_adding
FROM rm_fva_analysis
WHERE sop_cycle_stream_id IN (
    SELECT id FROM rm_sop_cycle
    WHERE tenant_id = '<tenant-uuid>'
    ORDER BY cycle_month DESC LIMIT 6
)
GROUP BY step_name, step_order
ORDER BY step_order;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | Single append-only event_store table |
| Event Infrastructure | 3 | event_type_registry, projection_checkpoint, event_snapshot |
| Read Model: Dimensions | 1 | rm_product (location, channel, customer similar) |
| Read Model: Forecasting | 2 | rm_current_forecast, rm_forecast_accuracy |
| Read Model: Planning | 3 | rm_fva_analysis, rm_sop_cycle, rm_alert |
| **Total** | **~10 core + read models** | Read models can be added/rebuilt without schema migration |

---

## Key Design Decisions

1. **Single event store table for all streams.** Rather than separate tables per aggregate, all events flow through one `event_store` table partitioned by `stream_type`. This simplifies cross-stream projections and global ordering while keeping the schema minimal.

2. **Global position sequence for cross-stream consistency.** The `global_position` column provides a total ordering across all streams, enabling projections that need to process events from multiple stream types (e.g., comparing forecast events with demand history events for accuracy calculation).

3. **Event versioning with upcasting.** The `event_version` field on each event supports schema evolution. When an event payload structure changes, the application upcasts old events to the new format during replay rather than migrating stored events.

4. **Snapshot store for replay performance.** For long-lived aggregates (products with years of changes, forecasts with thousands of line items), periodic snapshots avoid replaying the full event history. Snapshots are created every N events and replay starts from the latest snapshot.

5. **Read models as regular tables, not PostgreSQL materialized views.** Read models are maintained by application-level projections that process events and update tables, giving full control over update timing and error handling. PostgreSQL `LISTEN/NOTIFY` triggers event processing.

6. **Temporal queries are first-class.** The fundamental advantage: any past state can be reconstructed by replaying events up to a given timestamp. This natively supports "what did we forecast in March?" and FVA step-by-step analysis without maintaining separate history tables.

7. **Metadata captures provenance.** Every event carries metadata including `user_id`, `ip_address`, `correlation_id`, and `causation_id`, aligning with W3C PROV-O concepts for full decision traceability.

8. **FVA computed from event replay.** Forecast Value Added is not a stored metric but a derived computation: replay the forecast events at each S&OP step, compare accuracy against actuals, and compute the marginal improvement. This ensures FVA is always consistent with the actual event history.

9. **Read models are disposable and rebuildable.** If a read model becomes corrupt or a new view is needed, it can be rebuilt from scratch by replaying events from `global_position = 0`. The `projection_checkpoint` table tracks rebuild progress.

10. **Lower table count, higher application complexity.** The event store itself is only one table. The trade-off is that application code must implement event handling, projection logic, and snapshot management — a significant increase in code complexity compared to the normalized relational model.
