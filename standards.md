# Standards & API Reference

> Project: Dynamic Pricing Engine · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 21041:2018 — Guidance on Unit Pricing**
- URL: https://www.iso.org/standard/69727.html
- Provides principles and best-practice guidelines for unit pricing displayed by written, printed, or electronic means. Relevant for pricing engines that surface per-unit or per-weight price representations in multi-channel retail contexts.

**ISO 20022 — Universal Financial Industry Message Scheme**
- URL: https://www.iso20022.org/
- Open global standard for structured financial data interchange, covering payments, securities, and trade. Relevant where a dynamic pricing engine integrates with ERP systems, invoicing workflows, or financial reconciliation pipelines that use ISO 20022 message schemas (e.g. SEPA payments, cross-border B2B commerce).

**GS1 GTIN / GDSN — Global Trade Item Number & Data Synchronisation Network**
- URL: https://www.gs1.org/standards/id-keys/gtin
- GTINs (8–14 digits) are the universal product identifiers used as keys to retrieve price data, product attributes, and inventory records in supply chain systems. A dynamic pricing engine must resolve prices by GTIN when integrating with retail systems. The GS1 Global Data Synchronisation Network (GDSN) is the EDI/XML protocol for sharing product master data (including reference prices) between suppliers and retailers.

### W3C & IETF Standards

**RFC 8259 — The JavaScript Object Notation (JSON) Data Interchange Format**
- URL: https://www.rfc-editor.org/rfc/rfc8259
- The normative specification for JSON, the de facto data format used by all major pricing APIs (Pricefx, Prisync, Omnia, Amazon SP-API). Pricing event payloads, webhook bodies, and API responses should all conform to RFC 8259.

**RFC 7230–7235 — HTTP/1.1 Protocol (and RFC 9110–9114 for HTTP/2 & HTTP/3)**
- URL: https://datatracker.ietf.org/doc/html/rfc9110
- Governs HTTP semantics underpinning all REST pricing APIs. Key considerations: correct use of HTTP verbs (GET for price reads, PUT/PATCH for price updates), status codes (200, 201, 400, 429 for rate limiting), and caching headers (Cache-Control, ETag) for high-throughput pricing data feeds.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The dominant authentication standard for pricing platform APIs. Most SaaS pricing tools (Pricefx, Prisync, Omnia) use OAuth 2.0 bearer tokens or API keys built on OAuth 2.0 flows. Required when integrating pricing engines with ERP systems (SAP, Oracle) or marketplace APIs (Amazon SP-API uses LWA, a variant of OAuth 2.0).

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- JWTs are the token format used by many pricing platform APIs (Omnia uses JWT Bearer tokens). Pricing API integrations must validate JWT claims for scope, expiry, and issuer to avoid security vulnerabilities.

**RFC 5988 / RFC 8288 — Web Linking (pagination)**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Specifies the `Link` header for pagination of large pricing datasets. Relevant when paginating through product catalogues, competitor price histories, or repricing logs returned by pricing platform APIs.

### Data Model & API Specifications

**OpenAPI Specification 3.1 (OAS 3.1)**
- URL: https://spec.openapis.org/oas/v3.1.0.html
- The industry standard for describing RESTful APIs. All pricing platform integrations should be documented in OpenAPI 3.1 format. OpenAPI 3.1 aligns fully with JSON Schema 2020-12, enabling precise schema definitions for pricing data models (product, offer, price recommendation, price history).

**JSON Schema 2020-12**
- URL: https://json-schema.org/specification.html
- Used in conjunction with OpenAPI to define and validate pricing data payloads: product price objects (current price, reference price, minimum price, currency), competitor price records, and repricing event schemas. Supports validation of EU Omnibus Directive fields (30-day lowest price, prior price reference).

**AsyncAPI 2.x / 3.x**
- URL: https://www.asyncapi.com/docs/reference/specification/v3.0.0
- Standard for documenting event-driven and webhook-based APIs. Relevant for pricing engines that push repricing events (price changed, repricing triggered, competitor price update detected) to downstream systems via webhooks or message queues (Kafka, SQS, RabbitMQ).

**IATA NDC — New Distribution Capability (XML standard)**
- URL: https://developer.iata.org/en/ndc/
- XML-based API standard developed by IATA for airline dynamic pricing and offer distribution. Enables airlines to create unlimited dynamic price points and distribute rich offers beyond static fare files through GDSs, OTAs, and direct channels. Relevant for travel-sector dynamic pricing engines. The ATPCO dynamic pricing working group (~20 airlines, 3 GDSs) is developing interoperability specifications for dynamic fare adjustment on top of NDC.

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) + OpenID Connect (OIDC)**
- URL: https://openid.net/connect/
- OIDC extends OAuth 2.0 with identity tokens, enabling pricing platforms to authenticate integration partners securely and scope access (e.g., read-only price data vs. write repricing rules). Required for GDPR-compliant scoped consent when personalised pricing uses customer data.

**OWASP API Security Top 10**
- URL: https://owasp.org/www-project-api-security/
- Defines the most critical API security risks. Directly applicable to pricing APIs: broken object-level authorisation (a seller accessing another seller's price data), excessive data exposure (returning full pricing strategies when only a price is needed), and rate limiting (preventing competitor scraping via API abuse).

**GDPR / EU ePrivacy Directive**
- URL: https://gdpr-info.eu/
- The EU General Data Protection Regulation governs the use of customer-level data for personalised pricing. Pricing engines must obtain lawful basis (consent or legitimate interest) before using individual purchase history, browsing data, or device signals to set customer-specific prices. GDPR's data minimisation and purpose limitation principles constrain what customer data may be ingested by an AI pricing model.

**EU Omnibus Directive (Directive 2019/2161/EU)**
- URL: https://commission.europa.eu/law/law-topic/consumer-protection-law/unfair-commercial-practices-and-price-indication/price-indication-directive_en
- Mandates that any advertised price reduction must reference the lowest price charged in the prior 30 days. Requires pricing APIs and data models to record and expose a `priorPrice` (or `priceBeforeDiscount`) field reflecting the 30-day minimum. Non-compliance carries fines up to 4% of annual turnover. All EU-facing pricing engines must implement Omnibus-compliant price history logging.

**MAP (Minimum Advertised Price) Compliance**
- No single international standard; enforced via retailer and brand agreements (US-centric). Pricing engines must support MAP floor constraints per SKU and per channel, and must not publish prices below MAP thresholds in advertising contexts. The Amazon SP-API enforces MAP programmatically through the automated pricing rules API.

### MCP Server Specifications

The Model Context Protocol (MCP) is not directly standardised for dynamic pricing at present. However, an AI-native pricing engine could expose an MCP server to allow LLM-based agents to query price recommendations, retrieve elasticity estimates, or invoke repricing actions through natural-language tool calls. The MCP specification (from Anthropic) provides the relevant foundation:
- URL: https://modelcontextprotocol.io/docs/

---

## Similar Products — Developer Documentation & APIs

### Pricefx

- **Description:** Cloud-native price management and optimisation platform for B2B and retail, covering CPQ, promotions, and margin analytics.
- **API Documentation:** https://api.pricefx.com/rest-api
- **Overview:** https://api.pricefx.com/rest-api/overview/
- **Getting Started / Integration Guide:** https://api.pricefx.com/rest-api/integration-guide
- **Groovy API (backend extensions):** https://developer.pricefx.eu/api/
- **Data Formats:** JSON over HTTPS (TLS 1.2/1.3), UTF-8, HTTP POST for data writes, GET for reads. Follows RFC 2616 HTTP conventions.
- **Standards:** REST/JSON; custom JSON data model (no public OpenAPI spec found)
- **Authentication:** Token-based (access tokens obtained via authentication endpoint; specific flow not publicly documented — requires contacting Pricefx)

### Prisync

- **Description:** Competitor price tracking and dynamic repricing for e-commerce SMBs, supporting bulk product management and automated rule-based repricing.
- **API Documentation:** https://prisync.com/api
- **API Version:** v2.0.0
- **Key Endpoints:** `/add/product`, `/add/url`, `/add/batch` (up to 1,000 products), `/list/product`
- **SDKs/Libraries:** Language-agnostic REST API callable from PHP, Python, Java, Perl, and any HTTP client
- **Authentication:** API Key + API Token (obtained from account settings under API Access Details tab)
- **Standards:** REST/JSON

### Omnia Retail (Pricemonitor)

- **Description:** Automated pricing and competitor price monitoring platform for mid-market to enterprise retailers, with visual strategy builder and algorithmic repricing.
- **API Documentation:** https://api-docs.omniaretail.dev/
- **API Name:** Omnia 2.0 / Pricemonitor API
- **Key Operations:** Manage products, retrieve price recommendations, manage competitor offers
- **SDKs/Libraries:** GitHub organisation with client libraries — https://github.com/omniaretail
- **Authentication:** Basic Authentication or JWT Bearer tokens
- **Standards:** REST/JSON; OpenAPI-documented endpoints

### Amazon Selling Partner API (SP-API) — Product Pricing

- **Description:** Amazon's official API for marketplace sellers to retrieve competitive pricing, monitor featured offer prices, and manage automated repricing rules.
- **API Documentation:** https://developer-docs.amazon.com/sp-api/docs/product-pricing-api
- **Pricing FAQ:** https://developer-docs.amazon.com/sp-api/docs/pricing-faq
- **Automated Pricing Rules:** https://developer-docs.amazon.com/sp-api/docs/manage-automated-pricing-rules
- **API Reference (v0):** https://developer-docs.amazon.com/sp-api/reference/product-pricing-v0
- **SP-API Models:** https://developer-docs.amazon.com/sp-api/docs/sp-api-models
- **SDKs/Libraries:** Official SDK models available; community SDKs in Python, Java, JavaScript
- **Key Endpoints:** `getCompetitiveSummary`, `getFeaturedOfferExpectedPriceBatch` (up to 20 ASINs per batch; 40 FOEP requests per batch)
- **Authentication:** Login with Amazon (LWA) — OAuth 2.0 variant with scoped refresh tokens
- **Standards:** REST/JSON; OpenAPI-based model definitions; sandbox environment available
- **Access:** Sellers only (not vendors); Pricing permission required. Note: from January 2026, a $1,400/year developer fee applies; from April 2026, a tiered monthly usage fee structure applies.

### Shopify Admin API (Pricing & Products)

- **Description:** Shopify's primary API for e-commerce product and pricing management, supporting price rules, discount codes, variant pricing, and webhook-driven price update events.
- **API Documentation:** https://shopify.dev/docs/api
- **Price Rules Reference:** https://shopify.dev/docs/api/admin-rest/latest/resources/pricerule
- **Webhooks Guide:** https://shopify.dev/docs/apps/build/webhooks
- **SDKs/Libraries:** Official Shopify Ruby, Python, Node.js, PHP client libraries; GraphQL clients
- **Authentication:** OAuth 2.0 (public apps); Admin API access token (custom apps)
- **Standards:** GraphQL Admin API (primary from April 2025; all new public apps must use GraphQL); REST Admin API (legacy). OpenAPI-documented. Webhooks use HTTPS POST with HMAC-SHA256 signature verification.
- **Note:** GraphQL is the primary path forward for all new integrations as of 2025.

### Wiser Solutions

- **Description:** Price intelligence platform combining online competitor monitoring and in-store field-data capture, with API and webhook delivery of price alerts and competitive data.
- **API Tracker Entry:** https://apitracker.io/a/wiser
- **Datarade Profile:** https://datarade.ai/data-providers/wiser-solutions/profile
- **Key Capabilities:** Real-time pricing data and alerts via API and webhook integrations; integration with enterprise and mid-market ecommerce platforms
- **Authentication:** Not publicly documented — enterprise contract required
- **Standards:** REST/JSON; webhooks supported

### Competera

- **Description:** AI-driven retail pricing optimisation SaaS using deep learning models across demand forecasting, competitor analysis, and automated repricing. API available for ERP integration and automated pricing decisions.
- **Website:** https://competera.ai/
- **API Access:** Available to customers for ERP integration and competitor data ingestion; real-time market data collection over API with 99% SLA
- **Documentation:** Not publicly available — requires enterprise onboarding (implementation cycle: up to 14 days for Competitive Data; up to 60 days for Price Optimisation)
- **Authentication:** Not publicly documented
- **Standards:** REST/JSON (inferred from integration descriptions)

### Revionics (Aptos)

- **Description:** Enterprise retail pricing software covering regular, promotional, and markdown pricing, with demand-driven optimisation and multi-banner support.
- **API Tracker Entry:** https://apitracker.io/a/revionics
- **Okta SSO Integration:** https://www.okta.com/integrations/revionics/ (SSO supported)
- **Documentation:** Not publicly available — enterprise implementation required
- **Authentication:** SSO via Okta (SAML 2.0 / OIDC); API details require enterprise engagement
- **Standards:** REST/JSON (inferred); OpenAPI support noted in API Tracker

---

## Notes

**Gap — No unified pricing data interchange standard:** Unlike financial messaging (ISO 20022) or product identification (GS1), there is no universally adopted open standard for pricing event payloads or price recommendation schemas across the retail industry. Each platform (Pricefx, Prisync, Omnia, Amazon) defines its own JSON schemas. An open-source AI pricing engine has an opportunity to define and publish a canonical OpenAPI 3.1 schema for pricing events, which could become a community reference standard.

**Evolving — ATPCO airline dynamic pricing specifications:** The ATPCO dynamic pricing working group (20 airlines, 3 GDSs) is actively developing interoperability specifications targeting 80% of airline offers being dynamically created by 2026, up from 23% currently. This is the closest thing to a formal industry-standard dynamic pricing specification but is airline/travel-specific.

**Regulatory trend:** The EU Omnibus Directive's `priorPrice` field requirement (30-day lowest price) is becoming a de facto data model standard for European e-commerce pricing APIs, as platforms (Shopify, Centra, Omnia) have implemented it as a required field. Any pricing engine targeting EU markets must treat this as mandatory.

**Amazon SP-API pricing changes (2026):** From January 2026, a $1,400/year developer fee applies to all SP-API integrations. From April 2026, a tiered monthly usage fee based on GET call volume takes effect. Builders integrating Amazon repricing should factor these costs into architecture decisions and batch API calls efficiently.
