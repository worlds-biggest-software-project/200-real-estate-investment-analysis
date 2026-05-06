# Real Estate Investment Analysis — Feature & Functionality Survey

> Candidate #200 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| ARGUS Intelligence Platform | Desktop/SaaS hybrid | Commercial — enterprise contract | https://www.altusgroup.com/argus/ |
| DealCheck | Web + Mobile (iOS/Android) | Commercial — freemium, Pro ~$20/mo | https://dealcheck.io/ |
| DealWorthIt | SaaS | Commercial — custom pricing | https://dealworthit.com/ |
| PropStream | SaaS | Commercial — from ~$99/mo | https://www.propstream.com/ |
| Stessa | SaaS | Commercial — freemium | https://www.stessa.com/ |
| PropRise (Primer) | SaaS | Commercial — enterprise/institutional | https://www.proprise.ai/ |
| Blooma | SaaS | Commercial — enterprise | https://www.blooma.ai/ |
| Cherre Agent.STUDIO | SaaS | Commercial — enterprise | https://cherre.com/products/agent-studio/ |
| CoStar Analytics | SaaS | Commercial — from ~$200–$500/user/mo | https://www.costar.com/products |

---

## Feature Analysis by Solution

### ARGUS Intelligence Platform (formerly ARGUS Enterprise)

**Core features**
- Lease-by-lease DCF modelling for commercial real estate assets
- Multiple valuation methods: DCF, traditional capitalisation (hardcore, term and reversion, initial yield)
- Cash flow forecasting and budgeting at asset and portfolio level
- Sensitivity analysis and scenario testing for individual properties or entire portfolios
- Portfolio dashboards with visibility into performance by custom asset groupings
- Integration with property management systems (Yardi Voyager via ARGUS Connector)
- ARGUS API for programmatic read/write access to models and results
- LP reporting and fund performance management (via ARGUS Intelligence layer)

**Differentiating features**
- De facto institutional standard for CRE valuation — required or preferred by most institutional lenders and buyers
- Supports global valuation methodologies across different legal and accounting frameworks
- Largest installed base in CRE; results are defensible in audit, covenant review, and sale processes

**UX patterns**
- Traditionally desktop-first; migrating to cloud-enabled SaaS via ARGUS Intelligence Platform
- Steep learning curve; taught in 200+ universities as a specialised professional skill
- Heavy reliance on menus and property-level forms; limited visual/spatial interface

**Integration points**
- ARGUS API (REST) for reading and writing model data programmatically
- Pre-built connector for Yardi Voyager (property management)
- Integration ecosystem with MRI, RealPage and other CRE platforms via API partnerships
- Data export to Excel and standard financial reporting formats

**Known gaps**
- Very high cost; inaccessible to small investors, syndicators, and family offices
- Legacy UX creates significant onboarding friction
- No native AI-driven assumption generation or market data integration
- No self-service onboarding; deployment requires professional services
- ARGUS Enterprise no longer sold standalone — forces platform upgrade

**Licence / IP notes**
- Proprietary commercial software; no open-source components
- Data model and methodology are trade secrets of Altus Group (TSX: AIF)

---

### DealCheck

**Core features**
- Analysis of buy-and-hold rentals, short-term/Airbnb, BRRRR, fix-and-flip, and house-hack deal types
- Automated calculation of cash flow, cap rate, cash-on-cash return, IRR, and equity multiple
- Import wizard pulling property details from public records and MLS (list price, estimated value, taxes, rent comps, photos)
- Reverse valuation offer calculator — calculates maximum purchase price to hit target returns
- Side-by-side property comparison across multiple deals
- Interactive shareable reports and downloadable PDF output with charts and assumptions
- Web and mobile (iOS/Android) apps

**Differentiating features**
- Reverse valuation offer calculator is uniquely simple and actionable for acquisition negotiations
- Shareable property-specific links allow partners or lenders to explore analysis without logging in
- 350,000+ user base signals strong product-market fit in the prosumer/individual investor segment

**UX patterns**
- Mobile-first design with progressive disclosure; beginners can run an analysis in minutes
- Pre-filled defaults reduce blank-form anxiety; analysts can override any assumption
- Strong onboarding; freemium model lowers barrier to entry

**Integration points**
- MLS and public records data import
- PDF and print export
- No documented public API or webhook integration

**Known gaps**
- Not designed for complex commercial assets (multi-tenant office, industrial, retail)
- No institutional-grade DCF methodology or cap rate benchmarking
- No portfolio-level analytics or fund reporting
- No market data integration for validating rent or expense assumptions
- No team collaboration or workflow features

**Licence / IP notes**
- Proprietary SaaS; no open-source components

---

### DealWorthIt

**Core features**
- Automated calculation of FCF, IRR, ROI, equity multiple with methodology breakdowns
- Property discovery across 150M+ on- and off-market properties with tax history and owner information
- T12 and rent roll import for historical financial data upload
- Force Appreciation Modelling — models NOI improvement impact on property value through value-add
- Skip tracing for owner contact information
- Comparable transaction data and rent trend access
- Team collaboration features for shared underwriting and real-time change tracking
- Syndication-specific reporting showing waterfall distributions and investor-level metrics

**Differentiating features**
- Force Appreciation Modelling is a standout feature for value-add investors
- Syndication waterfall breakdowns differentiate it for GP/LP deal structures
- Built-in skip tracing bridges the gap between deal finding and deal closing
- T12 and rent roll import reduces manual data entry for multifamily deals

**UX patterns**
- Designed for speed — advertises analysis completion in under 60 seconds
- Report-first UI surfacing key metrics prominently before detailed assumptions
- Aimed at sophisticated investors but not institutional-grade workflows

**Integration points**
- Property data feeds from aggregated public records
- Report export for sharing with investors
- No documented public API

**Known gaps**
- Limited track record at institutional scale
- No integration with ARGUS or major CRE data platforms
- Relatively narrow asset class coverage; primarily multifamily-focused
- No regulatory compliance tooling for Reg D investor reporting

**Licence / IP notes**
- Proprietary SaaS; no open-source components

---

### PropStream

**Core features**
- Nationwide data on 160M+ properties with 165+ lead filter criteria
- MLS comps and AVM (Automated Valuation Model) data
- ADU and Rehab Calculator for improvement cost estimation
- Rental ROI Calculator for cash flow and long-term return forecasting
- Fix & Flip Analyzer for profit potential and maximum allowable offer calculation
- Heat maps filtering by property value, foreclosure rates, rent prices
- Demographic data covering population trends, income levels, and homeowner statistics
- Skip tracing with compliance features and contact ranking
- Predictive AI for identifying likely sellers and market trend forecasting
- List stacking to combine multiple lead lists and identify overlapping motivated sellers

**Differentiating features**
- Predictive AI for seller identification is differentiated in the lead-generation / top-of-funnel segment
- Heat map visualisation for market-level trend identification is a strong spatial analysis feature
- List stacking for wholesale and off-market strategies is unique to this segment

**UX patterns**
- Map-centric interface with layered overlays for visual market analysis
- Designed for high-volume deal screening rather than deep single-asset modelling
- Mobile app available; primarily web-based dashboard

**Integration points**
- MLS data feeds and public records aggregation
- Lead Automator for marketing list management and campaign automation
- No documented public API for third-party integration

**Known gaps**
- Not a replacement for deep underwriting tools; primarily a top-of-funnel product
- Lacks advanced predictive analytics and integrated valuation models for institutional use
- Users must export to Excel or other tools for detailed deal modelling
- No DCF modelling, no fund-level reporting

**Licence / IP notes**
- Proprietary SaaS; no open-source components

---

### Stessa

**Core features**
- Portfolio-level income and expense tracking with automatic bank feed imports
- Centralized dashboard for monitoring income, expenses, and key performance metrics in real-time
- Income Statement, Net Cash Flow report, Schedule of Real Estate Owned, and Tenant Ledger
- Lease tracking and tenant management with online rent collection
- Short-term rental rent roll for Airbnb data analysis
- Annual tax package for Schedule E filing preparation
- Tenant screening integration
- Mobile app for on-the-go portfolio monitoring

**Differentiating features**
- Only tool in this survey specifically designed to bridge deal analysis to ongoing financial management
- Schedule E tax preparation is highly valued by individual landlords and avoids accountant costs
- Strong integration with financial institutions for automated transaction import

**UX patterns**
- Accountant-style interface focused on financial transactions and reports
- Free tier with unlimited properties is a strong acquisition strategy
- Progressive feature unlock on paid tiers without hiding core functionality

**Integration points**
- Bank feed connections (Plaid-style) for transaction import
- Export to tax reporting formats
- Limited third-party integrations beyond financial institution connections

**Known gaps**
- Does not include tools for finding or evaluating new properties
- No market trend analysis or forward-looking deal underwriting
- Not suitable for commercial assets; residential/small portfolio only
- No fund or syndication reporting

**Licence / IP notes**
- Proprietary SaaS; owned by Roofstock
- No open-source components

---

### PropRise (Primer)

**Core features**
- AI document intelligence ingesting OMs, rent rolls, T12s, and operating statements
- Direct mapping of extracted data into existing Excel underwriting models (preserves existing workflows)
- Automated concession detection at comparable properties
- Embedded market reports covering rental trends, employment, housing affordability, and construction pipelines
- Traceable, defensible data extraction with source citations
- AI-powered deal sourcing and screening for institutional acquisition teams

**Differentiating features**
- Designed specifically for institutional CRE acquisition teams (managing $50B+ in CRE)
- Excel-native workflow preservation is a key differentiator — doesn't replace existing models
- Concession detection at comps is a novel AI capability that reduces manual market research
- Full document-to-underwriting pipeline in a single workflow

**UX patterns**
- Document upload → extraction → Excel population workflow
- Built for analyst-level users who already understand CRE underwriting
- Institutional positioning with white-glove onboarding

**Integration points**
- Excel integration via data mapping
- Document ingestion from any format (PDF, Word, spreadsheet)
- No documented public API

**Known gaps**
- Enterprise-only; no self-service tier for smaller operators
- Primarily a document intelligence layer, not a full underwriting platform
- Does not generate assumptions from scratch; depends on source documents
- Limited to institutional teams; not accessible to private syndicators or family offices

**Licence / IP notes**
- Proprietary SaaS — YC-backed startup
- No open-source components

---

### Blooma

**Core features**
- AI-powered CRE lending platform automating ~80% of pre-flight underwriting for lenders
- Automated analysis and parsing of tax returns, personal financial statements, and schedules of real estate
- 5,000+ data points analysed per deal against customisable lender credit policy
- Deal Scoring (1–100) based on lender-defined investment profile parameters
- Portfolio monitoring with stress-testing against rate changes, cap rate expansion, and vacancy shifts
- AI and OCR document processing with stated 99% accuracy on data ingestion
- Integration with LOS (Loan Origination Systems) and CRM via API
- Claims 400% increase in deals processed per underwriter

**Differentiating features**
- Lender-focused (commercial banks, credit unions, debt funds) rather than equity investor focused
- Policy-aware deal scoring tied to each lender's own credit policy is a unique capability
- Portfolio stress-testing module differentiates it from deal-by-deal analysis tools

**UX patterns**
- Enterprise SaaS with workflow-based loan origination focus
- Designed for underwriting teams, not individual investors
- API-first for integration into existing lender infrastructure

**Integration points**
- API integration with LOS and CRM systems
- Third-party data provider connections for real-time market data
- No documented public API for general use

**Known gaps**
- Lender/debt focus means it does not cover equity underwriting or LP reporting
- Not useful for acquisitions-side investors or syndicators
- High cost and enterprise complexity; not accessible to smaller credit shops

**Licence / IP notes**
- Proprietary SaaS
- No open-source components

---

### Cherre Agent.STUDIO

**Core features**
- Agentic AI workflow platform purpose-built for institutional real estate investors
- Universal and Semantic Data Models and Knowledge Graph for intelligent cross-data-source decisions
- Pre-built AI Agent Marketplace (Work Packages) for specific CRE business outcomes
- Model flexibility — ability to swap between AI models (LLM-agnostic)
- Comprehensive ecosystem of application connectors and third-party data integrations
- Client-configurable security controls; environments not used for model training
- Custom agent design, deployment, and scaling for bespoke investment processes

**Differentiating features**
- First major AI-native institutional CRE analytics platform (launched July 2025)
- Knowledge Graph approach to unifying disparate real estate data sources is architecturally differentiated
- Agent Marketplace allows clients to deploy pre-built outcomes without bespoke development
- LLM-agnostic design protects clients from vendor lock-in on AI models

**UX patterns**
- Platform/developer-oriented interface for building and deploying agents
- Enterprise sales and deployment with dedicated Cherre expert support
- Not a consumer-facing analysis dashboard; more of an enterprise AI infrastructure layer

**Integration points**
- REST APIs and application connectors for CRM, data providers, and property management systems
- Third-party data marketplace connections
- Documented enterprise API for institutional integration

**Known gaps**
- Very new product; limited production track record
- Enterprise-only pricing; inaccessible to non-institutional users
- Complexity of agentic workflows requires significant internal expertise or consulting support
- AI agent quality dependent on quality of connected data

**Licence / IP notes**
- Proprietary SaaS — $50M+ venture-backed
- No open-source components

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- DCF cash flow modelling with configurable hold period, discount rate, and exit cap rate
- Key return metrics: IRR, NPV, cash-on-cash return, equity multiple, cap rate
- Debt sizing and debt service coverage ratio (DSCR) calculation
- Rent roll and T12 import or manual entry for income/expense modelling
- Report generation (PDF or shareable link) suitable for lender or investor review
- Mobile or web access without desktop software installation
- Scenario/sensitivity analysis (at minimum, three-scenario modelling: bear/base/bull)

### Differentiating Features
- Lease-by-lease modelling for multi-tenant commercial assets (ARGUS)
- Agentic AI workflows and knowledge graph data unification (Cherre)
- Excel-native document-to-model population with source tracing (PropRise)
- Force appreciation modelling for value-add underwriting (DealWorthIt)
- Lender credit-policy-aware deal scoring (Blooma)
- Predictive AI seller identification and heat-map market visualisation (PropStream)
- Automated bank feeds bridging acquisition analysis to ongoing portfolio tracking (Stessa)

### Underserved Areas / Opportunities
- **Mid-market gap**: The space between prosumer tools (DealCheck, ~$20/month) and enterprise platforms (ARGUS, tens of thousands/year) is largely unserved; small private equity funds, syndicators, and family offices rely on Excel
- **Assumption sourcing**: No tool automatically validates rent, vacancy, or cap rate assumptions against current submarket data — analysts must do this manually or rely on static benchmarks
- **Cross-asset-class portability**: Most tools are optimised for one asset class (multifamily, residential, or commercial office); investors with diversified portfolios must use multiple tools
- **LP reporting automation**: Converting raw asset-level data into quarterly investor reports with narrative commentary remains largely manual
- **Acquisition-to-operations continuity**: No single platform spans the full deal lifecycle from initial screening through acquisition underwriting to ongoing performance tracking
- **Natural language deal entry**: No tool allows an analyst to describe a deal in plain language and have a model auto-populated with sourced assumptions
- **Regulatory compliance tooling**: Reg D Form D filing, investor accreditation verification, and GIPS-compliant reporting are handled by separate specialist tools; no investment analysis platform bundles these

### AI-Augmentation Candidates
- **Assumption generation**: AI can synthesise current submarket comps, rent trends, and vacancy data to auto-populate underwriting assumptions (replacing manual research)
- **Document ingestion**: AI OCR and LLMs can parse OMs, rent rolls, and T12s and populate models — PropRise and Blooma have begun this; still room for open-source alternative
- **Narrative report generation**: LLMs can convert raw financial model outputs into investor-ready quarterly reports with benchmarking commentary
- **Scenario stress-testing**: AI can generate probabilistic scenario ranges calibrated to historical volatility in the relevant submarket rather than analyst-defined ranges
- **Deal screening and ranking**: AI can continuously score inbound deal flow against a fund's defined criteria without analyst time
- **Anomaly detection**: AI can flag unusual expense items, below-market rents, or structurally risky assumptions in submitted models before investment committee review

---

## Legal & IP Summary

All nine solutions surveyed are proprietary commercial software with no open-source licensing. No patents on specific features were identified in public research, though Altus Group's ARGUS methodology and data model are trade secrets protected under commercial licensing. The ARGUS API terms restrict competitive use of extracted model data. PropRise, Cherre, DealWorthIt, and Blooma are VC-backed startups; their source code and AI models are proprietary. There are no licence compatibility concerns for an independent open-source AI-native real estate investment analysis tool, provided it does not reproduce ARGUS's proprietary DCF engine verbatim or misappropriate trade-secret methodology documentation. Standard financial algorithms (IRR, NPV, DCF) are mathematical calculations in the public domain and freely implementable.

---

## Recommended Feature Scope

**Must-have (MVP)**
- DCF cash flow model with configurable hold period, discount rate, exit cap rate, and financing terms
- Support for core deal types: multifamily, single-family buy-and-hold, fix-and-flip, and commercial (NNN, mixed-use)
- Automated return metric calculation: IRR, NPV, cash-on-cash, equity multiple, DSCR, cap rate
- Rent roll and T12 import (CSV/Excel) with automated model population
- Scenario modelling (bear/base/bull) with sensitivity tables
- Shareable investor-ready PDF and interactive report output
- AI-powered assumption pre-fill using submarket data from integrated data providers (ATTOM, BatchData)

**Should-have (v1.1)**
- Natural language deal entry — describe a deal and have the model auto-populate with sourced assumptions
- LP waterfall calculator for GP/LP syndication structures with investor-level returns
- Portfolio dashboard aggregating performance across multiple assets
- Comparable transaction analysis with market benchmarking for assumption validation
- API-first architecture for integration with property management systems (Yardi, MRI)

**Nice-to-have (backlog)**
- Agentic deal screener that scores inbound listings against fund-defined criteria
- Automated quarterly LP report generation with narrative commentary from LLM
- Reg D compliance module (Form D tracking, investor accreditation status)
- NCREIF/PREA-compliant performance reporting for institutional fund managers
- Integration with ARGUS API for data exchange with institutional counterparts
