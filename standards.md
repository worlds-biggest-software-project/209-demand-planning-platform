# Standards & API Reference

> Project: Demand Planning Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### Supply Chain Process Standards

**SCOR Digital Standard (SCOR-DS) v14.0 — ASCM**
- URL: https://www.ascm.org/corporate-solutions/standards-tools/scor-ds/
- The Supply Chain Operations Reference Digital Standard defines the six Level 1 supply chain processes (Plan, Order, Source, Transform, Fulfill, Return) and provides a linked data information model using W3C Provenance Ontology for digital interoperability. The 2025 revision (v14.0) includes an Information Model enabling data exchange between planning platforms. Relevant for defining demand plan data structures and process taxonomy.

**CPFR (Collaborative Planning, Forecasting, and Replenishment) — GS1 US**
- URL: https://www.gs1.org/standards
- The CPFR standard defines a nine-step business process framework for joint demand planning between retailers and suppliers. It specifies the information exchange model for sharing sales forecasts, replenishment orders, and exception alerts. API-based integrations are the recommended modern implementation; EDI remains common in legacy deployments. Relevant for retailer-supplier collaboration features.

**GS1 General Specifications v25.0**
- URL: https://www.gs1.org/standards/barcodes-epcrfid-id-keys
- The GS1 General Specifications (updated January 2025, v25.0) provide the global standard for product identification (GTIN, GLN, SSCC). These identifiers are the canonical keys for product master data in any demand planning system interoperating with retail or CPG supply chains. Essential for data model design of product and location dimensions.

**IBF Best Practice Standards — Institute of Business Forecasting**
- URL: https://ibf.org/
- The IBF defines professional standards and canonical metrics for demand forecasting process quality: MAPE (Mean Absolute Percentage Error), MPE/Bias, WAPE (Weighted Absolute Percentage Error), and FVA (Forecast Value Added). FVA measures the improvement each process step contributes versus a naïve baseline. These metrics are industry-expected outputs for any demand planning platform's accuracy reporting module.

**S&OP / IBP Process Framework — IBF / APICS**
- URL: https://ibf.org/knowledge/glossary/forecast-value-added-fva-131
- The standard monthly Sales & Operations Planning (S&OP) and Integrated Business Planning (IBP) cadence defines the process stages that a demand planning platform must support: statistical forecast generation, commercial override, consensus review, executive sign-off. Any platform targeting enterprise buyers must map its workflow states to this cadence.

---

### EDI & Data Exchange Standards

**UN/EDIFACT DELFOR — UN/CEFACT**
- URL: https://www.tracelink.com/orchestration-integration-transactions/transform-your-supply-chain-seamless-edi-integration/edi-edifact-delfor-forecast-planning-schedule
- The EDIFACT DELFOR (DELivery FORecast) message is the UN/EDIFACT-standard transaction for communicating forecasted demand and planning schedules from buyers to suppliers. It defines the message structure for conveying expected future requirements over a planning horizon. Relevant for supplier-facing demand signal sharing in manufacturing and retail supply chains.

**ANSI X12 830 — Planning Schedule with Release Capability**
- URL: https://www.edi2xml.com/blog/edi-846-edi-852-edi-830/
- The X12 830 transaction set is the North American EDI standard equivalent to DELFOR, used to transmit production sequences, shipping schedules, and demand forecasts from buyers to suppliers. The X12 846 (Inventory Inquiry/Advice) and 852 (Product Activity Data / POS data) are companion transactions for inventory and point-of-sale demand signal exchange. Relevant for North American retail and manufacturing integrations.

---

### API & Data Format Standards

**OpenAPI Specification v3.1.0 — OpenAPI Initiative**
- URL: https://spec.openapis.org/oas/v3.1.0
- The OpenAPI Specification is the industry-standard format for describing RESTful APIs. OpenAPI 3.1.0 aligns fully with JSON Schema Draft 2020-12 and is the expected specification format for any demand planning platform's public API. The specification should describe all planning data endpoints, authentication flows, and error schemas.

**JSON Schema Draft 2020-12 — IETF / json-schema.org**
- URL: https://json-schema.org/specification
- JSON Schema provides the vocabulary for annotating and validating JSON documents. It is the recommended format for defining demand plan data payloads (forecast series, product hierarchies, planning parameters). OpenAPI 3.1.0 embeds JSON Schema Draft 2020-12 natively.

**RFC 8259 — JSON Data Interchange Format — IETF**
- URL: https://www.rfc-editor.org/rfc/rfc8259
- RFC 8259 is the canonical specification for JSON, the de-facto data interchange format for modern demand planning APIs. All REST API payloads should conform to this standard.

**RFC 9110 — HTTP Semantics — IETF**
- URL: https://www.rfc-editor.org/rfc/rfc9110
- RFC 9110 defines the semantics of HTTP/1.1 and HTTP/2, including methods, status codes, headers, and content negotiation. A demand planning REST API must conform to HTTP semantics for correct interoperability with clients, gateways, and integration platforms.

**RFC 8288 — Web Linking — IETF**
- URL: https://www.rfc-editor.org/rfc/rfc8288
- RFC 8288 defines the `Link` header for hypermedia-driven pagination and resource relationships in REST APIs. Relevant for paginating large forecast series and plan data exports.

**OData v4.01 — OASIS**
- URL: https://www.oasis-open.org/committees/tc_home.php?wg_abbrev=odata
- OData v4.01 is the OASIS standard for building and consuming RESTful data services, widely used in SAP and Microsoft ecosystems. SAP IBP exposes its planning data via OData V2 services (IBP API Service on api.sap.com). Building OData-compatible endpoints would simplify integration for SAP-ecosystem customers.

---

### Security & Authentication Standards

**OAuth 2.0 — RFC 6749 — IETF**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- OAuth 2.0 is the industry-standard authorization framework for securing API access. All enterprise demand planning integrations are expected to support OAuth 2.0, typically using the Client Credentials flow for machine-to-machine integration and the Authorization Code flow for user-facing integrations.

**OpenID Connect 1.0 (OIDC) — OpenID Foundation**
- URL: https://openid.net/connect/
- OIDC is the identity layer built on top of OAuth 2.0, required for secure user authentication in multi-tenant SaaS applications. Any demand planning platform serving multiple enterprise tenants must implement OIDC for SSO interoperability with corporate identity providers (Okta, Azure AD, Google Workspace).

**ISO/IEC 27001:2022 — Information Security Management**
- URL: https://www.iso.org/standard/27001
- ISO 27001 is the international standard for information security management systems. Enterprise procurement teams require ISO 27001 certification (or SOC 2 Type II as the US equivalent) as a prerequisite for approving any cloud planning platform. Relevant for security architecture and compliance documentation.

**GDPR — General Data Protection Regulation — European Union**
- URL: https://gdpr.eu/
- GDPR applies to demand planning platforms that process personal data of EU individuals. In supply chain planning, personal data may arise in retailer collaboration (named contacts), workforce planning modules, or commercial forecast inputs linked to named accounts. Platforms must implement data processing agreements, data minimisation, and retention controls. Fines of up to €20M or 4% of global annual revenue apply for non-compliance.

---

### AI & Agent Standards

**Model Context Protocol (MCP) — Anthropic (Open Standard)**
- URL: https://modelcontextprotocol.io/
- MCP is an open standard (released November 2024, now adopted by OpenAI, Microsoft, AWS, and Google as of early 2026) for standardising how AI models and agents access external tools, data sources, and context. Supply chain research confirms MCP adoption in planning contexts reduces forecast errors by enabling AI agents to share persistent context across planning domains. Building an MCP server for a demand planning platform would enable AI planning agents to read and write forecast data, trigger scenario runs, and retrieve plan exceptions via a standardised protocol.

---

## Similar Products — Developer Documentation & APIs

### Anaplan

- **Description:** Connected planning platform for demand planning, S&OP, and financial integration. Most developer-accessible enterprise planning platform with publicly documented REST APIs.
- **API Documentation:** https://help.anaplan.com/anaplan-api-844c6d40-a21c-423d-8435-ebaaa0372b76
- **API Reference (Apiary):** https://anaplan.docs.apiary.io/
- **SDKs/Libraries:** Java SDK — https://github.com/anaplaninc/anaplan-java-client (Apache 2.0)
- **Developer Guide:** https://help.anaplan.com/integration-api-v20-399496b0-d66e-4a84-895a-8d1ffdee2e6b
- **Standards:** REST/JSON; bulk and transactional API patterns
- **Authentication:** OAuth 2.0 (Certificate-based and Basic Auth also supported)

### SAP Integrated Business Planning (IBP)

- **Description:** Enterprise demand planning and S&OP platform tightly integrated with SAP S/4HANA. Exposes planning data via OData V2 services on the SAP Business Accelerator Hub.
- **API Documentation:** https://api.sap.com/package/IBPAPIService/odata (SAP ID required)
- **SAP Help Portal:** https://help.sap.com/docs/SAP_INTEGRATED_BUSINESS_PLANNING
- **SDKs/Libraries:** SAP Integration Suite (Cloud Integration); no standalone SDK
- **Developer Guide:** https://community.sap.com/t5/technology-blog-posts-by-members/access-sap-ibp-master-key-figures-data-using-odata-integration/ba-p/13511827
- **Standards:** OData V2 (OASIS); SAP_COM_0720 communication scenario
- **Authentication:** Basic Auth; Certificate-based (SSL Client certs); OAuth 2.0 via SAP BTP

### Kinaxis Maestro (Developer Studio)

- **Description:** AI-powered concurrent supply chain planning platform. Developer Studio provides API-based extensibility with visual integration job designer and low-code custom app capabilities.
- **API Documentation:** https://www.kinaxis.com/en/developer-studio (customer portal; login required)
- **Knowledge Network:** https://knowledge.kinaxis.com/
- **SDKs/Libraries:** No public SDK; Developer Studio provides in-platform integration toolkit
- **Developer Guide:** https://github.com/SimioLLC/KinaxisRapidResponse (community integration flows)
- **Standards:** REST API (specifications gated); pre-built connectors for SAP, Oracle, Salesforce
- **Authentication:** Enterprise SSO; details gated behind customer contract

### Blue Yonder Luminate Planning

- **Description:** AI-driven demand planning and supply chain execution platform. Blue Yonder Connect provides API and integration capabilities; self-service partner portal for developers.
- **API Documentation:** https://success.blueyonder.com/ (customer login required)
- **Blue Yonder Connect Overview:** https://info.blueyonder.com/blue-yonder-platform/what-is-blue-yonder-connect-api-expansion-pack
- **SDKs/Libraries:** No public SDK; pre-built connectors via Blue Yonder Connect
- **Developer Guide:** Available via partner portal (gated)
- **Standards:** REST and SOAP APIs; no public OpenAPI spec
- **Authentication:** Enterprise credentials required; OAuth 2.0 presumed for REST endpoints

### o9 Solutions

- **Description:** AI-native integrated business planning platform with graph-based Digital Brain. Developer documentation is enterprise-gated.
- **API Documentation:** https://o9solutions.com/ (customer portal; no public reference)
- **SDKs/Libraries:** No public SDK
- **Developer Guide:** Available via certified partner program
- **Standards:** REST APIs; no published OpenAPI spec
- **Authentication:** Enterprise SSO; OAuth 2.0 presumed

### Prediko

- **Description:** AI-powered demand forecasting and inventory planning for Shopify/DTC brands. Most developer-accessible SMB-tier platform with public REST API documentation.
- **API Documentation:** https://api.prediko.io/docs/ (publicly accessible)
- **Help Center:** https://help.prediko.io/en/
- **SDKs/Libraries:** Python, JavaScript, and cURL examples in API docs
- **Developer Guide:** https://api.prediko.io/docs/ — time-to-first-request under 5 minutes
- **Standards:** REST/JSON; OpenAPI-style documentation
- **Authentication:** API Key (Bearer token)

### Shopify (as demand signal source for e-commerce demand planning)

- **Description:** Leading e-commerce platform whose APIs are a primary data source for DTC demand planning tools. Provides inventory, order, and product data essential for SMB demand forecasting.
- **API Documentation:** https://shopify.dev/docs/api
- **SDKs/Libraries:** JavaScript, Ruby, Python, PHP, and others — https://shopify.dev/docs/api/admin-rest
- **Developer Guide:** https://shopify.dev/docs/apps/build
- **Standards:** REST Admin API and GraphQL Admin API; both publicly documented
- **Authentication:** OAuth 2.0 (app installation flow); API Key for private apps

### IBM watsonx.ai Time Series Forecasting API

- **Description:** Managed ML forecasting API enabling developers to call trained time-series models without managing infrastructure. GA since February 2025. Relevant for building custom demand sensing layers.
- **API Documentation:** https://www.ibm.com/think/tutorials/time-series-api-watsonx-ai
- **SDKs/Libraries:** Python SDK via IBM Cloud; REST API
- **Developer Guide:** https://www.ibm.com/think/tutorials/time-series-api-watsonx-ai
- **Standards:** REST/JSON; OpenAPI specification available in IBM Cloud API catalogue
- **Authentication:** IBM Cloud API Key (Bearer token via IAM)

### ForecastAPI

- **Description:** Lightweight managed forecasting service that automatically selects the best statistical model for a given time series. Applicable to sales, inventory, and demand planning use cases.
- **API Documentation:** https://forecastapi.com/
- **SDKs/Libraries:** REST API (language-agnostic)
- **Developer Guide:** https://forecastapi.com/ (documentation publicly accessible)
- **Standards:** REST/JSON
- **Authentication:** API Key

---

## Notes

**API openness gap**: Of the eight demand planning solutions surveyed, only Anaplan (REST API with public Apiary documentation), Prediko (public API docs at api.prediko.io), and ForecastAPI offer publicly accessible developer documentation without an enterprise contract. SAP IBP, Kinaxis, Blue Yonder, o9, and Logility all gate API documentation behind customer portals. This represents a significant opportunity for an open-source or developer-friendly alternative to differentiate on API accessibility.

**EDI legacy vs. API modernity**: The demand planning market operates on a spectrum from legacy EDI (EDIFACT DELFOR, X12 830) for supplier collaboration to modern REST APIs for internal integration. Any new platform should support both patterns — REST APIs for forward-looking integrations and EDI gateway compatibility for trading partner networks that have not yet modernised.

**MCP as an emerging standard**: The Model Context Protocol is gaining rapid adoption in supply chain AI contexts as the standard for AI agent interoperability. As of early 2026, over 10,000 MCP servers are registered with 177,000+ tools. Building an MCP server interface for a demand planning platform would enable AI agents from any MCP-compatible host (Claude, Copilot Studio, AWS Bedrock) to directly query and update demand plans without bespoke integrations.

**SCOR-DS alignment**: The 2025 ASCM SCOR Digital Standard v14.0 provides a linked data information model using W3C Provenance Ontology. Aligning a demand planning platform's data model to SCOR-DS taxonomy (Plan process domain, with Forecast, Demand Signal, and Consensus Plan as data entities) would differentiate it as interoperable with other SCOR-compliant tools and consultancies.
