# Demand Planning Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source demand planning platform combining statistical forecasting, real-time demand sensing, and consensus planning for supply chain teams.

Demand Planning Platform is a candidate open-source alternative to enterprise demand planning suites such as SAP IBP, Kinaxis Maestro, Blue Yonder Luminate, and o9 Solutions. It targets supply chain, demand planning, and S&OP teams who need modern AI-driven forecasting without seven-figure licence commitments or 12–18 month implementations.

---

## Why Demand Planning Platform?

- **Enterprise platforms are prohibitively priced.** SAP IBP starts at ~$100,000/year; Kinaxis, Blue Yonder, and o9 are custom-quoted at similar enterprise tiers, locking out mid-market and smaller buyers.
- **Implementations take 12–18 months.** SAP IBP and Kinaxis projects routinely run more than a year and require certified consulting partners, while Logility ships in 3–6 months for the mid-market.
- **APIs are gated behind contracts.** Only Anaplan and Prediko publish developer documentation; every other incumbent hides API specs behind enterprise customer portals, blocking evaluation and integration.
- **AI depth is uneven.** Statistical-only methods underperform in volatile markets, yet most platforms still require manual model selection, manual bias correction, and meeting-driven consensus reconciliation.
- **No open standards exposure.** No incumbent natively exposes data in SCOR-DS or GS1-compliant formats, limiting interoperability across the planning stack.

---

## Key Features

### Forecasting Core

- Multi-model statistical and ML demand forecasting with automated model selection per SKU/location
- Demand hierarchy management across product, location, channel, customer, and time dimensions
- Historical data cleansing: outlier detection, event isolation, and zero-demand handling
- Seasonality, trend, and promotional uplift decomposition
- Forecast accuracy dashboard covering MAPE, bias, and WAPE with actuals overlay and drill-down

### Demand Sensing & Signals

- Short-horizon forecast refresh from near-real-time order or POS signals
- Probabilistic forecast output with configurable confidence intervals
- External signal ingestion for weather, economic indicators, social sentiment, and web traffic
- Bias detection and correction layer monitoring forecast bias by product, channel, and region

### Collaborative & Consensus Planning

- Multi-user concurrent editing with change attribution
- S&OP process support with structured monthly consensus workflow and approval states
- Forecast Value Added (FVA) analysis tracking the contribution of each planning step
- New product forecasting using similar-item analogues
- Promotion and event management with manual uplift entry and historical promotion modelling

### AI Co-pilot

- Natural language scenario co-pilot translating free-text "what-if" assumptions into quantitative scenarios
- Consensus reconciliation agent aggregating commercial, marketing, and finance inputs and flagging disagreements
- Automated bias detection and correction with explainable audit trail
- Automated statistical model selection and ensemble management evaluating 20+ models per SKU

### Integration & Extensibility

- REST API with public OpenAPI documentation for all core planning data objects
- ERP data ingestion via REST API, CSV, JSON, and webhook
- Exception-driven planner alerts for stock-out risk, forecast deviation, and bias threshold breach
- Export of demand plan to downstream supply and inventory planning processes

---

## AI-Native Advantage

Unlike incumbents whose AI features are bolted onto legacy planning engines, this platform treats multi-signal demand sensing, automated ensemble selection, and bias auto-correction as first-class capabilities. A natural-language scenario co-pilot lets planners express assumptions in prose rather than configuring model parameters, and a consensus reconciliation agent collapses meeting-driven S&OP cycles into continuous, auditable planning. Forecast explainability and FVA scoring are built into every model so AI-generated forecasts can be defended in S&OP reviews.

---

## Tech Stack & Deployment

The platform targets cloud-native deployment with a publicly documented REST API as the primary integration surface — directly addressing the gap that enterprise incumbents leave by gating API documentation behind contracts. Planned alignment with industry standards includes SCOR (ASCM), CPFR (GS1/VICS), IBF forecasting metrics (MAPE, bias, FVA), and S&OP/IBP process cadences. An MCP server implementation is on the backlog for AI agent integration with downstream supply planning tools.

---

## Market Context

The supply chain planning market is valued in the tens of billions globally, with demand planning the largest segment and the AI-native segment growing fastest. Incumbent pricing ranges from $100,000+/year (SAP IBP) to custom enterprise quotes (Kinaxis, Blue Yonder, o9), with mid-market alternatives (Logility, Flowlity) more accessible and AWS Forecast pay-per-use. Primary buyers are VPs of Supply Chain or Demand Planning, demand and supply planners, commercial finance teams, and procurement leaders driving S&OP processes; an IBM survey cited in the research found 90% of executives expect supply chain workflows to include AI assistance by 2026.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
