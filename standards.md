# Standards & API Reference

> Project: Real Estate Investment Analysis · Generated: 2026-05-03

## Industry Standards & Specifications

### Performance Reporting Standards

**NCREIF PREA Reporting Standards (2025 Edition)**
- URL: https://reportingstandards.info/ and https://ncreif.org/about/standards/
- Co-sponsored by the National Council of Real Estate Investment Fiduciaries (NCREIF) and the Pension Real Estate Association (PREA). The 2025 edition introduces an IRR hierarchy (Levels 1a, 1b, 4) to enforce consistent gross and net IRR calculation methodology across funds and managers, and mandates pairing IRR with paid-in capital multiples (TVPI, DPI, RVPI). The 2025 expansion adds recommended asset- and investment-level reporting fields including valuation inputs, leverage measures, and operating metrics relevant to LP review and audit processes.

**NCREIF Property Index (NPI)**
- URL: https://user.ncreif.org/data-products/property/
- The primary benchmark index for measuring institutional real estate investment performance in the United States. A real estate investment analysis tool targeting institutional users must be able to produce returns in formats comparable to NPI benchmarks.

**GIPS — Global Investment Performance Standards (CFA Institute)**
- URL: https://www.cfainstitute.org/en/ethics-standards/codes/gips-standards
- CFA Institute's globally recognised standards for investment performance reporting. Increasingly adopted by real estate fund managers for LP reporting. An AI-native tool should produce outputs that can be mapped to GIPS-compliant reporting templates. Does not fully address real estate-specific needs; NCREIF/PREA Reporting Standards serve as the domain-specific complement.

**INREV / ANREV — Global Definitions Database (GDD)**
- URL: https://www.inrev.org/definitions/EN/D0342
- Maintained jointly by NCREIF, PREA, INREV (European Association for Investors in Non-Listed Real Estate), and ANREV (Asian Association for Investors in Non-Listed Real Estate). Provides a unified glossary of real estate investment terms to reduce cross-border reporting inconsistencies. Relevant for any platform targeting international institutional investors.

### Regulatory Standards

**SEC Regulation D / Rule 506 (b) and (c)**
- URL: https://www.sec.gov/rules/proposed/2022/33-11078.pdf and SEC EDGAR
- Governs private placement offerings for real estate syndications in the United States. Requires Form D filing within 15 days of first sale. Shapes the disclosure requirements and investor communication standards embedded in syndicator-facing tools. An investment analysis platform serving syndicators should track Reg D compliance status and facilitate investor accreditation verification.

**GDPR — General Data Protection Regulation (EU 2016/679)**
- URL: https://gdpr.eu/
- Applicable to any platform processing personal data of EU-based investors or property owners. Requires explicit consent, right to erasure, data portability, and breach notification within 72 hours. Real estate investment platforms storing investor PII (personal financial statements, accreditation documents, LP agreements) must implement GDPR-compliant data handling. Requires role-based access controls and encryption of personal data at rest and in transit.

**CCPA — California Consumer Privacy Act**
- URL: https://oag.ca.gov/privacy/ccpa
- US state-level equivalent of GDPR for California residents. Relevant for platforms with US investor or property owner data. Requires disclosures about data collection and opt-out rights for personal data sales.

### Data & API Standards

**RESO Web API (Real Estate Standards Organization)**
- URL: https://www.reso.org/ and https://www.reso.org/data-dictionary/
- The modern standard for exchanging MLS real estate listing data between systems. REST-style API using HTTP, JSON, and the OData protocol. Exposes RESO-compliant resources (Property, Member, Office). RETS (the older XML-based transport) is being retired by most MLSs in 2025–2026. Compliance with RESO Web API is essential for any tool ingesting MLS listing data for comparable analysis or rental income benchmarking. In 2025, the standard is expanding to include Add/Edit capabilities, Validation Expressions, and EntityEvent/Webhooks.

**RESO Data Dictionary**
- URL: https://www.reso.org/data-dictionary/
- A set of guidelines defining standard property listing data fields across 700+ US MLSs. Provides standardised field names, data types, and enumerations for property attributes. Allows local MLS extensions for market-specific fields. Implementing RESO Data Dictionary compliance ensures portability of property data ingested from different MLS sources.

**OpenAPI Specification (OAS 3.x)**
- URL: https://spec.openapis.org/oas/latest.html
- The de facto standard for documenting REST APIs. An AI-native real estate investment analysis platform should publish its own API using OpenAPI 3.1 to enable third-party integrations, SDK generation, and ecosystem development.

**JSON Schema**
- URL: https://json-schema.org/
- Standard for validating JSON document structure. Relevant for defining and validating deal input schemas, rent roll formats, and financial model data exchange formats between systems.

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749)**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The standard authorisation framework for delegated access to APIs. Required for any integration with property management systems, data providers, or MLS feeds that use token-based authentication.

**OpenID Connect 1.0 (OIDC)**
- URL: https://openid.net/connect/
- Identity layer built on OAuth 2.0; provides user authentication in addition to API authorisation. Standard for enterprise SSO integration, which institutional investors expect when onboarding to SaaS platforms.

**OWASP Application Security Standards**
- URL: https://owasp.org/
- The OWASP Top 10 and ASVS (Application Security Verification Standard) provide a framework for securing web applications handling sensitive financial data. Relevant given that investment analysis platforms store proprietary deal data, LP financial information, and fund performance data.

**ISO/IEC 27001:2022 — Information Security Management**
- URL: https://www.iso.org/standard/82875.html
- International standard for information security management systems (ISMS). Enterprise customers (institutional investors, banks, fund managers) increasingly require ISO 27001 certification from SaaS vendors as a procurement prerequisite.

**TLS 1.3 (RFC 8446)**
- URL: https://datatracker.ietf.org/doc/html/rfc8446
- The current standard for transport layer security. All API communications and user-facing web traffic must use TLS 1.3 minimum to satisfy institutional security requirements.

### MCP Server Specifications

The Model Context Protocol (MCP) is relevant for an AI-native investment analysis platform as it defines how LLM-based agents interact with external tools and data sources.

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/
- Open standard (Anthropic) defining how AI models interact with external tools, APIs, and data sources. An AI-native real estate investment analysis platform could expose MCP server endpoints for deal analysis, comps retrieval, and market data queries — enabling LLM-based agents (Claude, GPT-4, etc.) to invoke underwriting workflows programmatically.

---

## Similar Products — Developer Documentation & APIs

### ARGUS API (Altus Group)

- **Description:** REST API providing programmatic read/write access to ARGUS Enterprise and ARGUS Intelligence Platform. Allows integration of ARGUS cash flow models with third-party property management, asset management, and data systems.
- **API Documentation:** https://www.altusgroup.com/argus/products/integration-solutions
- **SDKs/Libraries:** No public SDKs documented; enterprise customer access via Altus Group account
- **Developer Guide:** https://www.altusgroup.com/insights/argus-api-is-now-available/
- **Standards:** REST/JSON; enterprise partner integration model
- **Authentication:** Enterprise API key / OAuth 2.0 via Altus Group account management

### ATTOM Data API

- **Description:** REST API covering 158M+ US properties with 9,000+ data points per property including property characteristics, ownership, transaction history, tax assessments, AVM valuations, and neighbourhood analytics. Widely used for investment screening, underwriting data enrichment, and comparable analysis.
- **API Documentation:** https://api.developer.attomdata.com/docs
- **SDKs/Libraries:** Python, PHP, Ruby, JavaScript examples at https://api.developer.attomdata.com/documentation; Postman collection available
- **Developer Guide:** https://api.developer.attomdata.com/docs/guides
- **Standards:** REST/JSON and XML; OData-compatible querying
- **Authentication:** API key (30-day free trial available)

### BatchData API

- **Description:** Real estate data API covering 155M+ US properties with 700+ attributes including valuations, tax data, sales history, owner information, and contact data. Provides both transactional API access and bulk cloud delivery for large-scale ML and analytics workloads.
- **API Documentation:** https://developer.batchdata.com/docs/batchdata/welcome-to-batchdata
- **SDKs/Libraries:** Python and Node.js client libraries; full documentation at developer.batchdata.com
- **Developer Guide:** https://batchdata.io/blog/real-estate-api-documentation-examples
- **Standards:** REST/JSON; continuously updated live production database
- **Authentication:** API key

### HouseCanary Analytics API

- **Description:** AI-powered real estate data and valuation API providing AVM (Automated Valuation Model) estimates, rental value estimates, market forecasts, and property-level analytics for 100M+ US residential properties. Used by investors, lenders, and proptech developers.
- **API Documentation:** https://api-docs.housecanary.com/
- **SDKs/Libraries:** Postman collection with 20+ language examples; Order Manager SDK at https://www.housecanary.com/resources/developer-tools
- **Developer Guide:** https://www.housecanary.com/blog/data-explorer-api-quick-start-guide
- **Standards:** REST/JSON; versioned endpoints (v2, v3); endpoint format `{VERSION}/{LEVEL}/{TARGET}`
- **Authentication:** API key; Pro plan or higher required for production access

### Zillow Group Data API (Bridge Interactive)

- **Description:** Official Zillow Group API providing access to Zestimates, property valuations, listing data, and rental estimates for ~100M US properties. The original Zillow API was deprecated in 2021; the replacement is Bridge Interactive, which is a partner programme with enterprise pricing.
- **API Documentation:** https://www.zillowgroup.com/developers/ and https://www.bridgeinteractive.com/developers/zillow-group-data/
- **SDKs/Libraries:** No public SDKs; partner-only API access requiring enterprise application
- **Developer Guide:** https://www.zillowgroup.com/developers/api/zestimate/zestimates-api/
- **Standards:** REST/JSON (partner API); 1,000 daily call limit on approved non-commercial use
- **Authentication:** Partner API key; enterprise application required (~$500/month minimum)

### CoStar Risk Analytics API

- **Description:** Desktop application and web service API for commercial real estate loan risk analysis. Calculates Probability of Default (PD), Loss Given Default (LGD), Expected Loss (EL), and Confidence Interval results for individual loans or portfolios. Used by CRE lenders for credit risk assessment.
- **API Documentation:** https://www.costarriskanalytics.com/Home/CRAHome
- **SDKs/Libraries:** Not publicly documented; enterprise partner access
- **Developer Guide:** Enterprise-only; contact CoStar for API access terms
- **Standards:** Web service API (SOAP/REST — not publicly specified); JSON output
- **Authentication:** Enterprise account; API access negotiated commercially

### Cherre Data Platform API

- **Description:** Institutional real estate data platform providing a Universal Data Model and Knowledge Graph connecting property, market, ownership, and transactional data from multiple sources. Agent.STUDIO exposes agentic AI workflows via API for institutional CRE analytics.
- **API Documentation:** https://cherre.com/products/agent-studio/
- **SDKs/Libraries:** Enterprise integration; no public SDK documented
- **Developer Guide:** Available to enterprise customers; contact Cherre directly
- **Standards:** REST/JSON; GraphQL-compatible queries on Knowledge Graph (reported)
- **Authentication:** OAuth 2.0 enterprise SSO; client-configurable access controls

### DealCheck API

- **Description:** Real estate investment analysis API providing programmatic access to cash flow modelling, return metric calculations, and property analysis for rental, flip, and BRRRR deals. Primarily a consumer-facing product; API availability limited.
- **API Documentation:** https://dealcheck.io/ (no public API documentation found)
- **SDKs/Libraries:** No documented public SDK
- **Developer Guide:** Not publicly available as of May 2026
- **Standards:** Not publicly documented
- **Authentication:** Not publicly documented

### NCREIF Data Products API

- **Description:** NCREIF provides data products and index access for institutional real estate performance benchmarking, including the NCREIF Property Index (NPI), ODCE index, and other institutional benchmarks. Data access is subscription-based for NCREIF members.
- **API Documentation:** https://user.ncreif.org/data-products/property/
- **SDKs/Libraries:** No public SDK; data available via member portal and data feed
- **Developer Guide:** https://ncreif.org/__static/494f1e600333bddebfbf93e847412f47/NCREIF-Data-and-Products-Guide-2025.pdf
- **Standards:** NCREIF PREA Reporting Standards; GIPS-aligned
- **Authentication:** NCREIF member subscription required

---

## Notes

**Emerging Standards — AI and Data Governance**
The rapid adoption of AI in CRE investment analysis is outpacing formal standards development. There are currently no ISO or industry-specific standards governing AI-generated underwriting assumptions, AI valuation outputs, or AI-driven LP reporting. NCREIF and PREA are expected to address AI-generated data in future reporting standards revisions. Platforms should implement internal model governance documentation to demonstrate defensibility of AI-generated assumptions to institutional investors and regulators.

**Data Access Constraints**
CoStar's commercial data (the most comprehensive CRE transaction database) does not have a public API and is aggressively protected from third-party redistribution. Any platform integrating CoStar data must negotiate an enterprise commercial agreement. This represents a structural moat for incumbent platforms and a meaningful data sourcing challenge for new entrants. ATTOM, BatchData, and HouseCanary provide viable alternatives for residential and lower-tier commercial market data.

**RETS Deprecation**
Many MLS providers are retiring legacy RETS feeds in 2025–2026 in favour of RESO Web API. Any platform building MLS data integration should implement RESO Web API compliance from the outset rather than building on RETS.

**Mid-Market Data Gap**
There is no comprehensive, affordable, programmatic source for institutional-quality CRE transaction comps (lease comps, sale comps, cap rate benchmarks) outside of CoStar's paywalled ecosystem. This gap is the single largest data infrastructure challenge for any open-source or mid-market real estate investment analysis platform.
