# Demand Planning Platform — Phased Development Plan

> Project: 209-demand-planning-platform · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | Python 3.12+ (backend) + TypeScript (frontend) | Python is the dominant language for statistical forecasting (statsmodels, Prophet, scikit-learn, NeuralProphet) and ML model management; TypeScript for the planner dashboard |
| API framework | FastAPI | Async for concurrent model training; auto-generates OpenAPI 3.1 — a key differentiator since incumbents gate API docs; Pydantic v2 for strict forecast data validation |
| Database | PostgreSQL 16 | Dimensional schema from data-model-suggestion-1; JSONB for external signal metadata; excellent for the cross-dimensional analytical queries demand planning requires |
| Migrations | Alembic | Version-controlled schema changes |
| ORM | SQLAlchemy 2.0 (async) | Complex dimensional queries across product × location × time hierarchies |
| Task queue | Celery + Redis | Async model training (20+ models per SKU can take minutes), batch forecast generation, demand sensing refresh, bias detection |
| Time-series DB | TimescaleDB (PostgreSQL extension) | High-volume time-series data: daily/weekly demand actuals, forecast values, external signals — hypertables with automatic partitioning |
| Forecasting | statsmodels + Prophet + scikit-learn + NeuralProphet | Multi-model ensemble: ARIMA/ETS (statsmodels), Prophet (seasonality/holidays), XGBoost/LightGBM (scikit-learn), NeuralProphet (deep learning) |
| Model management | MLflow | Track model versions, parameters, and accuracy per SKU; automated model selection based on holdout MAPE |
| LLM integration | Anthropic Python SDK (Claude) | Natural-language scenario co-pilot; consensus reconciliation agent; forecast explainability narratives |
| Frontend | Next.js 15 (App Router) + Tailwind CSS | Planner dashboard with forecast charts, accuracy grids, S&OP workflow; responsive for tablet use in planning meetings |
| Charting | Recharts + D3.js | Forecast vs. actuals time-series charts; accuracy heatmaps; hierarchy drill-down |
| Data ingestion | pandas + Celery | CSV/JSON/webhook ingestion; data cleansing pipeline (outlier detection, zero-demand handling) |
| Containerisation | Docker + docker-compose | PostgreSQL+TimescaleDB, Redis, FastAPI, Celery workers, MLflow, Next.js |
| Testing | pytest + pytest-asyncio + httpx + Vitest | Backend (pytest); Frontend (Vitest); E2E (Playwright) |
| Linting | Ruff (Python) + Biome (TS) | Fast linting/formatting |
| Type checking | mypy (strict) + TypeScript strict | Both stacks type-checked |
| Package manager | uv (Python) + pnpm (TS) | Fast, lockfile-based |

### Project Structure

```
demand-planning/
├── pyproject.toml
├── docker-compose.yml
├── Dockerfile.api
├── Dockerfile.frontend
├── alembic.ini
├── alembic/versions/
├── src/
│   ├── main.py
│   ├── config.py
│   ├── database.py
│   ├── models/                      # SQLAlchemy ORM
│   │   ├── tenant.py
│   │   ├── product.py               # Product hierarchy (SKU → subcategory → category → family)
│   │   ├── location.py              # Location hierarchy (site → region → country)
│   │   ├── channel.py
│   │   ├── demand_history.py        # Actual demand time series
│   │   ├── forecast.py              # Forecast versions and values
│   │   ├── plan.py                  # Consensus plans
│   │   ├── accuracy.py              # MAPE, bias, WAPE, FVA records
│   │   ├── promotion.py
│   │   ├── external_signal.py
│   │   └── audit_log.py
│   ├── schemas/
│   │   ├── forecasts.py
│   │   ├── plans.py
│   │   ├── accuracy.py
│   │   ├── data_import.py
│   │   └── scenarios.py
│   ├── api/
│   │   ├── forecasts.py
│   │   ├── plans.py
│   │   ├── accuracy.py
│   │   ├── products.py
│   │   ├── locations.py
│   │   ├── data_import.py
│   │   ├── promotions.py
│   │   ├── scenarios.py
│   │   ├── sop.py
│   │   └── ai.py
│   ├── services/
│   │   ├── forecast_engine.py       # Multi-model ensemble
│   │   ├── model_selector.py        # Automated model selection per SKU
│   │   ├── data_cleanser.py         # Outlier detection, event isolation
│   │   ├── demand_sensor.py         # Short-horizon demand sensing
│   │   ├── accuracy_calculator.py   # MAPE, bias, WAPE, FVA
│   │   ├── bias_detector.py         # Automated bias detection + correction
│   │   ├── consensus_manager.py     # S&OP consensus workflow
│   │   ├── promotion_modeller.py    # Promotional uplift
│   │   ├── npi_forecaster.py        # New product introduction
│   │   ├── scenario_engine.py       # What-if scenarios
│   │   └── nl_copilot.py            # Natural language scenario co-pilot
│   ├── ml/
│   │   ├── arima.py
│   │   ├── ets.py
│   │   ├── prophet_model.py
│   │   ├── xgboost_model.py
│   │   ├── neural_prophet.py
│   │   └── ensemble.py
│   ├── tasks/
│   │   ├── forecast.py
│   │   ├── accuracy.py
│   │   ├── demand_sensing.py
│   │   ├── bias.py
│   │   └── data_import.py
│   └── lib/
│       ├── auth.py
│       ├── hierarchy.py             # Dimensional hierarchy traversal
│       ├── metrics.py               # MAPE, bias, WAPE calculations
│       └── cleansing.py             # Outlier detection, zero-demand
├── frontend/
│   ├── package.json
│   └── src/app/
│       ├── dashboard/
│       ├── forecasts/
│       ├── accuracy/
│       ├── plans/
│       ├── promotions/
│       ├── scenarios/
│       ├── sop/
│       ├── data-import/
│       └── settings/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
│       ├── sample_demand_history.csv
│       ├── sample_products.json
│       └── sample_forecast.json
└── docs/
```

---

## Phase 1: Foundation

### Purpose
Establish the project skeleton, database schema from data-model-suggestion-1 (tenants, product hierarchy, location hierarchy, demand history, forecasts), authentication, and Docker environment.

### Tasks

#### 1.1 — Project Scaffold and Configuration

**What**: Create pyproject.toml, FastAPI app, Pydantic Settings, Docker with PostgreSQL+TimescaleDB, Redis, MLflow.

**Design**:

```python
class Settings(BaseSettings):
    database_url: str = "postgresql+asyncpg://demand:demand@localhost:5432/demand"
    redis_url: str = "redis://localhost:6379/0"
    mlflow_tracking_uri: str = "http://localhost:5001"
    anthropic_api_key: str = ""
    site_url: str = "http://localhost:3000"
    forecast_models: list[str] = ["arima", "ets", "prophet", "xgboost"]
    model_config = {"env_prefix": "DEMAND_"}
```

**Testing**:
- Unit: config parses env correctly
- Integration: `docker-compose up -d` → PostgreSQL, TimescaleDB, Redis, MLflow healthy

#### 1.2 — Database Schema — Dimensional Hierarchy and Demand History

**What**: Implement product hierarchy, location hierarchy, channel, demand_history (TimescaleDB hypertable), and forecast tables from data-model-suggestion-1.

**Design**:

Product hierarchy: `product_family` → `product_category` → `product_subcategory` → `product` (SKU). Each level has aggregation rules. Location hierarchy: `country` → `region` → `site`. GS1 identifiers: product.gtin, location.gln.

```python
# TimescaleDB hypertable for demand actuals
class DemandHistory(Base):
    __tablename__ = "demand_history"
    time: Mapped[datetime]  # TimescaleDB time column
    tenant_id: Mapped[UUID]
    product_id: Mapped[UUID]
    location_id: Mapped[UUID]
    channel_id: Mapped[UUID | None]
    quantity: Mapped[Decimal]
    revenue_cents: Mapped[int | None]
    data_source: Mapped[str]  # "erp", "pos", "manual", "csv"

# CREATE hypertable after table creation:
# SELECT create_hypertable('demand_history', 'time');

# Forecast values table (also TimescaleDB hypertable)
class ForecastValue(Base):
    __tablename__ = "forecast_values"
    time: Mapped[datetime]
    forecast_version_id: Mapped[UUID]
    product_id: Mapped[UUID]
    location_id: Mapped[UUID]
    quantity: Mapped[Decimal]
    confidence_lower: Mapped[Decimal | None]  # P10
    confidence_upper: Mapped[Decimal | None]  # P90
    model_name: Mapped[str]  # which model produced this value
```

**Testing**:
- Integration: migrations apply, TimescaleDB hypertables created
- Integration: create product hierarchy (family→category→subcategory→SKU) → FK chain holds
- Integration: insert 10,000 demand_history rows → TimescaleDB partitions correctly

#### 1.3 — Authentication and Roles

**What**: Multi-tenant auth with demand planning roles: admin, demand_planner, supply_planner, commercial_user, viewer, api_service.

**Design**:

Roles: `demand_planner` (create/edit forecasts, run models), `supply_planner` (view forecasts, create supply plans), `commercial_user` (add market intelligence, promotional events), `admin` (full access), `viewer` (read-only dashboards).

**Testing**:
- Unit: demand_planner can create forecast → allowed
- Unit: commercial_user cannot modify statistical forecast → 403
- Unit: viewer can access accuracy dashboard → allowed

---

## Phase 2: Data Ingestion and Cleansing

### Purpose
Import historical demand data from ERPs and POS systems; cleanse it for forecasting (outlier removal, event isolation, zero-demand handling). Clean data is the prerequisite for accurate forecasting.

### Tasks

#### 2.1 — Data Import Pipeline

**What**: Import demand history via CSV upload, REST API, and webhook; validate and store in TimescaleDB.

**Design**:

```python
# POST /api/v1/data-import/demand-history
# Content-Type: multipart/form-data (CSV) or application/json
class DemandImportRow(BaseModel):
    date: date
    sku: str                       # matched to product.sku
    location_code: str             # matched to location.code
    channel: str | None = None
    quantity: Decimal
    revenue_cents: int | None = None

# Validation:
# 1. SKU exists in product table
# 2. Location exists in location table
# 3. Quantity >= 0
# 4. Date is in the past
# 5. No duplicate (same sku + location + date + channel)
```

Bulk import: CSV with up to 1M rows processed asynchronously via Celery task. Progress tracked via polling endpoint.

**Testing**:
- Unit: valid CSV with 100 rows → 100 demand_history records
- Unit: unknown SKU → row rejected with error detail
- Unit: duplicate row → skipped (upsert)
- Integration: upload 100K row CSV → processed within 30 seconds

#### 2.2 — Data Cleansing Pipeline

**What**: Detect and handle outliers, promotional events, and zero-demand periods in historical data.

**Design**:

```python
class DataCleanser:
    def cleanse(self, history: pd.DataFrame, product_id: UUID) -> CleansedData:
        """
        1. Outlier detection: IQR method (values > Q3 + 1.5*IQR flagged)
        2. Event isolation: known promotions tagged; their demand separated
           into base demand + promotional uplift
        3. Zero-demand handling: distinguish between true zero-demand
           (product exists, no sales) and missing data (product not yet listed)
        4. Trend adjustment: detect level shifts (new store opening, product discontinuation)
        5. Return: cleansed_series (for forecasting), outliers (for review),
           events (tagged), quality_score (0-100)
        """

@dataclass
class CleansedData:
    cleansed_series: pd.Series
    outliers: list[OutlierRecord]
    events: list[EventRecord]
    quality_score: float  # 0-100, higher = cleaner data
```

**Testing**:
- Unit: demand spike 4× average during known promotion → tagged as promotional uplift
- Unit: random spike with no promotion → flagged as outlier
- Unit: product listed mid-year → zeros before listing date → marked as "not yet listed"
- Unit: quality score = 100 for perfectly clean series

---

## Phase 3: Multi-Model Forecasting Engine

### Purpose
Generate demand forecasts using multiple statistical and ML models with automated model selection per SKU/location. This is the core value proposition.

### Tasks

#### 3.1 — Statistical and ML Model Library

**What**: Implement ARIMA, ETS, Prophet, XGBoost, and NeuralProphet forecasting models with a common interface.

**Design**:

```python
class ForecastModel(ABC):
    @abstractmethod
    def fit(self, history: pd.DataFrame, config: ModelConfig) -> None: ...

    @abstractmethod
    def predict(self, periods: int) -> pd.DataFrame:
        """Return DataFrame with columns: date, quantity, lower, upper"""

class ARIMAModel(ForecastModel):
    def fit(self, history: pd.DataFrame, config: ModelConfig) -> None:
        self.model = pm.auto_arima(history["quantity"], seasonal=True, m=config.seasonality_period)

class ProphetModel(ForecastModel):
    def fit(self, history: pd.DataFrame, config: ModelConfig) -> None:
        self.model = Prophet(yearly_seasonality=True, weekly_seasonality=config.weekly)
        self.model.fit(history.rename(columns={"date": "ds", "quantity": "y"}))

class XGBoostModel(ForecastModel):
    def fit(self, history: pd.DataFrame, config: ModelConfig) -> None:
        features = self._engineer_features(history)  # lag, rolling_mean, day_of_week, month
        self.model = XGBRegressor().fit(features, history["quantity"])
```

Each model produces point forecast + confidence intervals (P10, P90).

**Testing**:
- Unit: ARIMA on 2 years of weekly data → 12-week forecast with reasonable values
- Unit: Prophet with strong seasonality → captures seasonal pattern
- Unit: XGBoost with engineered features → forecast within 20% MAPE on holdout
- Unit: all models return same DataFrame schema (date, quantity, lower, upper)

#### 3.2 — Automated Model Selection and Ensemble

**What**: Evaluate all models per SKU/location on holdout data; select best model or create weighted ensemble.

**Design**:

```python
class ModelSelector:
    def select(self, product_id: UUID, location_id: UUID,
               history: pd.DataFrame, models: list[str]) -> ModelSelectionResult:
        """
        1. Split history: last 12 periods as holdout, rest as training
        2. Train each model on training set
        3. Predict holdout period
        4. Compute MAPE, bias, WAPE for each model on holdout
        5. Select: best single model (lowest MAPE) OR weighted ensemble
        6. Ensemble weights: inverse MAPE weighting, normalised to sum to 1
        7. Log results to MLflow for tracking
        """

@dataclass
class ModelSelectionResult:
    selected_model: str             # "prophet" or "ensemble"
    model_scores: dict[str, float]  # {"arima": 12.5, "prophet": 8.3, "xgboost": 9.1}
    ensemble_weights: dict[str, float] | None  # {"prophet": 0.45, "xgboost": 0.35, "arima": 0.20}
    holdout_mape: float
    mlflow_run_id: str
```

Celery task: `train-all-models` runs models for all SKU/location combinations; can process 1,000+ SKUs in parallel across workers.

**Testing**:
- Unit: 3 models trained, Prophet lowest MAPE → Prophet selected
- Unit: ensemble with weights [0.5, 0.3, 0.2] → weighted average of forecasts
- Unit: MLflow run logged with parameters and MAPE metric
- Integration: run for 10 SKUs → 10 model selections with MLflow tracking

#### 3.3 — Forecast Generation and Storage

**What**: Generate forecasts for all SKU/location combinations; store in forecast_values TimescaleDB table.

**Design**:

```python
# POST /api/v1/forecasts/generate
class ForecastGenerateRequest(BaseModel):
    horizon_periods: int = 52         # weeks
    granularity: Literal["daily", "weekly", "monthly"] = "weekly"
    model_selection: Literal["auto", "ensemble", "prophet", "arima", "xgboost"] = "auto"
    product_filter: list[UUID] | None = None  # None = all products
    location_filter: list[UUID] | None = None

# Creates a new forecast_version record
# Generates forecast_values per SKU/location/period
# State: generating → completed | failed
```

**Testing**:
- Unit: generate for 10 SKUs × 5 locations × 52 weeks → 2,600 forecast_value rows
- Unit: forecast_version status transitions: generating → completed
- E2E: trigger generation → forecast visible in dashboard with chart

---

## Phase 4: Forecast Accuracy and Bias Detection

### Purpose
Measure forecast accuracy (MAPE, bias, WAPE, FVA) and detect systematic bias. Essential for continuous forecasting improvement and S&OP credibility.

### Tasks

#### 4.1 — Accuracy Calculation Engine

**What**: Compute MAPE, bias, WAPE, and FVA per SKU, location, channel, and planning hierarchy level.

**Design**:

```python
class AccuracyCalculator:
    def compute(self, forecast_version_id: UUID, actuals_through: date) -> AccuracyReport:
        """
        For each SKU/location where actuals exist:
        MAPE = mean(|forecast - actual| / actual) × 100
        Bias = mean(forecast - actual) / mean(actual) × 100  (positive = over-forecast)
        WAPE = sum(|forecast - actual|) / sum(actual) × 100
        FVA = 1 - (MAPE_this_step / MAPE_naive_forecast)
             where naive = last period's actual
        """

@dataclass
class AccuracyReport:
    forecast_version_id: UUID
    overall_mape: float
    overall_bias: float
    overall_wape: float
    by_product_family: list[AccuracyByDimension]
    by_location_region: list[AccuracyByDimension]
    fva_by_step: list[FVARecord]  # FVA for statistical, demand sensing, consensus
```

BullMQ job: recalculates accuracy weekly as new actuals arrive.

**Testing**:
- Unit: forecast 100, actual 90 → MAPE = 11.1%, bias = +11.1%
- Unit: forecast 100, actual 110 → MAPE = 9.1%, bias = -9.1%
- Unit: FVA positive → this forecast step adds value over naive
- E2E: accuracy dashboard → heatmap by product family × region

#### 4.2 — Automated Bias Detection and Correction

**What**: Detect systematic forecast bias and apply corrections automatically.

**Design**:

```python
class BiasDetector:
    BIAS_THRESHOLD_PCT = 5.0  # flag if bias > ±5%
    CORRECTION_WINDOW = 12    # last 12 periods

    async def detect_and_correct(self, product_id: UUID, location_id: UUID) -> BiasResult:
        """
        1. Compute rolling bias over last CORRECTION_WINDOW periods
        2. If |bias| > BIAS_THRESHOLD_PCT → flag bias
        3. If bias is consistently positive (over-forecasting) → apply downward correction
        4. Correction factor = 1 - (bias / 100)
        5. Log correction with before/after values for audit trail
        """

@dataclass
class BiasResult:
    product_id: UUID
    rolling_bias_pct: float
    is_biased: bool
    correction_applied: bool
    correction_factor: float
    audit_trail: str  # "Bias detected: +8.2% over 12 periods. Applied correction factor 0.918."
```

**Testing**:
- Unit: 12 periods consistently over-forecasted by 10% → bias detected, correction = 0.90
- Unit: bias within ±5% → no correction applied
- Unit: correction applied → audit trail records before/after values

---

## Phase 5: Planner Dashboard and Accuracy Views

### Purpose
Build the demand planner's primary workspace: forecast charts, accuracy grids, hierarchy drill-down, and exception alerts.

### Tasks

#### 5.1 — Forecast Dashboard

**What**: Interactive forecast vs. actuals chart with hierarchy navigation and drill-down.

**Design**:

`/forecasts` page: time-series chart (Recharts) showing forecast line with confidence band + actuals dots. Hierarchy drill-down: click product family → categories → subcategories → SKUs. Location filter. Channel filter. Forecast version selector. Period toggle (weekly/monthly).

Exception alerts: red badges on SKUs where MAPE > threshold or bias > threshold.

**Testing**:
- E2E: select product family → chart shows aggregated forecast and actuals
- E2E: drill down to SKU → chart shows individual SKU forecast
- E2E: exception badge on high-MAPE SKU → visible in hierarchy tree

#### 5.2 — Accuracy Heatmap and FVA View

**What**: Accuracy dashboard with heatmap grid and Forecast Value Added analysis.

**Design**:

`/accuracy` page: heatmap grid (product families × regions) coloured by MAPE (green < 15%, yellow 15-30%, red > 30%). FVA waterfall chart showing accuracy contribution of each planning step (statistical → demand sensing → consensus override). Trend view: accuracy improvement over time.

**Testing**:
- E2E: heatmap renders with correct colour coding per cell
- E2E: FVA waterfall → shows which steps add/subtract value
- Unit: accuracy data for 50 product×location combinations → heatmap correct

---

## Phase 6: Collaborative Planning and S&OP

### Purpose
Enable multi-user collaborative demand planning with structured S&OP consensus workflow. Transforms the platform from a forecasting tool into a planning platform.

### Tasks

#### 6.1 — Multi-User Collaborative Editing

**What**: Planners can override statistical forecasts with manual adjustments; all changes attributed to users.

**Design**:

```python
# POST /api/v1/plans/{planId}/overrides
class ForecastOverride(BaseModel):
    product_id: UUID
    location_id: UUID
    period_start: date
    override_quantity: Decimal
    reason: str                     # "Promotional event", "Customer intelligence", etc.
    override_type: Literal["absolute", "percentage"]  # absolute value or % change

# All overrides stored with user_id, timestamp, and reason
# FVA tracks whether overrides improve or degrade accuracy
```

Concurrent editing: optimistic locking per SKU/location/period cell. If two planners edit the same cell, last-write-wins with conflict notification.

**Testing**:
- Unit: override forecast from 100 to 120 → stored with user attribution
- Unit: FVA computes accuracy with and without override → shows if override helped
- E2E: two planners edit different SKUs → both changes saved
- E2E: two planners edit same SKU → conflict notification, latest saved

#### 6.2 — S&OP Consensus Workflow

**What**: Structured monthly S&OP process with demand review, supply review, and executive approval stages.

**Design**:

S&OP plan state machine: `statistical_generated` → `demand_review` → `supply_review` → `pre_sop` → `executive_sop` → `approved` → `published`.

Each stage has: designated reviewers, due date, approval actions (approve/revise/reject). When approved, the consensus plan is frozen and published to downstream supply planning.

```python
# POST /api/v1/sop/plans/{planId}/advance
class SOPAdvanceRequest(BaseModel):
    action: Literal["approve", "revise", "reject"]
    comments: str | None = None
```

**Testing**:
- Unit: demand review approval → advances to supply_review
- Unit: executive rejection → returns to demand_review with comments
- Unit: published plan → frozen, no further edits allowed
- E2E: full S&OP cycle from statistical → published in UI

---

## Phase 7: Promotions and New Product Forecasting

### Purpose
Model promotional uplift and forecast demand for new products with no sales history. Critical for CPG and retail demand planning.

### Tasks

#### 7.1 — Promotion Modelling

**What**: Define promotional events with expected uplift; model historical promotion impact on demand.

**Design**:

```python
# POST /api/v1/promotions
class PromotionCreate(BaseModel):
    name: str
    product_ids: list[UUID]
    location_ids: list[UUID] | None  # None = all locations
    start_date: date
    end_date: date
    promotion_type: Literal["price_discount", "bogo", "feature_display", "ad_campaign", "seasonal"]
    expected_uplift_pct: float | None  # manual estimate, optional
    historical_promotion_id: UUID | None  # reference similar past promotion for modelling
```

Historical promotion modelling: for past promotions, the forecast engine decomposes demand into base + uplift. For future promotions, uplift is estimated from similar past promotions or manual planner input.

**Testing**:
- Unit: past promotion with 30% uplift → demand decomposed into base + 30% uplift
- Unit: future promotion referencing similar past → uplift estimated from historical
- Unit: manual uplift of 20% → forecast increased by 20% for promotion period

#### 7.2 — New Product Introduction (NPI) Forecasting

**What**: Forecast demand for new products using similar-item analogues.

**Design**:

```python
class NPIForecaster:
    async def forecast_npi(self, new_product_id: UUID,
                           analogue_product_ids: list[UUID],
                           scaling_factor: float = 1.0) -> pd.DataFrame:
        """
        1. Identify analogue products (similar category, price point, attributes)
        2. Take first 12 months of analogue demand as the base curve
        3. Average across analogues if multiple
        4. Apply scaling factor (e.g., 0.8 if expecting lower volume)
        5. Return forecast with confidence intervals wider than established products
        """
```

**Testing**:
- Unit: 2 analogues with different curves → averaged base curve
- Unit: scaling_factor = 0.5 → forecast at 50% of analogue average
- Unit: NPI forecast has wider confidence intervals than established products

---

## Phase 8: AI-Native Features (v1.1)

### Purpose
Add AI differentiators: demand sensing from real-time signals, natural-language scenario co-pilot, and consensus reconciliation agent.

### Tasks

#### 8.1 — Demand Sensing

**What**: Short-horizon forecast refresh using near-real-time order/POS data and external signals.

**Design**:

```python
class DemandSensor:
    async def sense(self, product_id: UUID, location_id: UUID) -> SensedForecast:
        """
        1. Fetch last 7 days of POS/order data
        2. Compare to statistical forecast for same period
        3. If actual trend deviates > 10% from forecast → adjust short-horizon (next 4 weeks)
        4. Incorporate external signals: weather forecast, economic indicators
        5. Produce sensed forecast with shorter confidence intervals
        """
```

BullMQ job runs daily for all active SKU/location combinations.

**Testing**:
- Unit: POS data 20% above forecast → sensed forecast adjusted upward for next 4 weeks
- Unit: POS data within 10% → no adjustment (statistical forecast retained)
- Integration: daily sensing run → sensed forecasts visible in dashboard alongside statistical

#### 8.2 — Natural-Language Scenario Co-Pilot

**What**: Planner describes a what-if scenario in plain English; AI translates to quantitative forecast adjustments.

**Design**:

```python
# POST /api/v1/ai/scenario
# Request: { "description": "What if we lose our top 3 customers in the Northeast region?" }
# Response: {
#   "scenario": {
#     "adjustments": [
#       {"product_ids": [...], "location_ids": [...], "adjustment_pct": -15, "period": "next_6_months"}
#     ],
#     "impact_summary": "Total demand reduction of 12% across NE region...",
#     "confidence": 0.7
#   }
# }

class ScenarioCoPilot:
    SYSTEM_PROMPT = """You are a demand planning AI assistant. Given a natural-language
    scenario description, translate it into quantitative forecast adjustments. Use the
    provided product hierarchy, location hierarchy, and historical demand data as context.
    Return structured JSON with specific product/location/period adjustments."""
```

**Testing**:
- Unit (mocked): "lose top 3 customers in NE" → adjustments targeting NE locations, -15% on affected products
- Unit (mocked): "launch in 5 new stores" → adjustments adding demand at new locations
- Integration: scenario applied → dashboard shows scenario forecast overlay vs. base

#### 8.3 — Consensus Reconciliation Agent

**What**: AI agent aggregates commercial, operations, and finance forecast inputs; flags disagreements and proposes reconciled plan.

**Design**:

```python
class ConsensusAgent:
    async def reconcile(self, plan_id: UUID) -> ReconciliationResult:
        """
        1. Gather inputs: statistical forecast, demand planner overrides,
           commercial team adjustments, finance revenue targets
        2. Identify disagreements: SKU/location/period cells where inputs diverge > 10%
        3. For each disagreement, Claude analyses rationale from each input
        4. Propose reconciled value with justification
        5. Flag remaining unresolvable conflicts for S&OP meeting
        """
```

**Testing**:
- Unit (mocked): statistical = 100, commercial = 130, finance = 110 → reconciled ≈ 115 with justification
- Unit (mocked): all inputs agree within 5% → no conflicts flagged
- Integration: reconciliation run → dashboard shows reconciled plan with conflict highlights

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                     ─── required by everything
    │
Phase 2: Data Ingestion & Cleansing     ─── requires Phase 1
    │
Phase 3: Multi-Model Forecasting        ─── requires Phase 2
    │
Phase 4: Accuracy & Bias Detection      ─── requires Phase 3
    │
Phase 5: Planner Dashboard              ─── requires Phase 3 + Phase 4
    │
Phase 6: Collaborative Planning & S&OP  ─── requires Phase 5
    │
Phase 7: Promotions & NPI               ─── requires Phase 3 (parallel with Phases 4-6)
    │
Phase 8: AI-Native Features             ─── requires Phase 3 + Phase 6
```

Parallelism opportunities:
- Phase 7 (Promotions/NPI) can be developed concurrently with Phases 4-6 after Phase 3
- Phase 8 tasks can be developed independently after their dependencies
- Phase 4 and Phase 5 can overlap (accuracy engine first, then dashboard)

---

## Definition of Done (per phase)

1. All tasks implemented with code matching the design specification.
2. All unit and integration tests pass (`pytest` + `vitest`).
3. Ruff linting passes (Python); Biome passes (TypeScript).
4. mypy strict passes; tsc --noEmit passes.
5. Docker build succeeds.
6. `docker-compose up` brings all services to healthy state.
7. Feature works end-to-end.
8. Forecast accuracy metrics (MAPE, bias, WAPE) compute correctly against known test data.
9. TimescaleDB hypertables functioning correctly for demand_history and forecast_values.
10. MLflow tracking records model parameters and accuracy for each training run.
11. New API endpoints appear in auto-generated OpenAPI spec.
12. Database migrations created and tested.
13. New environment variables documented in config.py.
