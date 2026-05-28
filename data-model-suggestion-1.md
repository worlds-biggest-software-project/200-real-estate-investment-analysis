# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Real Estate Investment Analysis · Created: 2026-05-20

## Philosophy

This model follows a traditional normalized relational design where every domain concept gets its own table with strict foreign key relationships. Each entity -- properties, deals, investors, leases, cash flows, scenarios -- is stored in a dedicated table with well-defined columns and constraints. Junction tables handle many-to-many relationships (e.g., investors participating in multiple deals, properties belonging to multiple portfolios).

The approach mirrors how institutional platforms like ARGUS structure their data internally: a property has leases, leases generate income streams, income streams feed into cash flow projections, and cash flow projections produce valuation outputs. Every relationship is explicit and enforced at the database level. This aligns with NCREIF PREA Reporting Standards, which expect structured, auditable data fields at both asset and fund levels.

Normalized relational models are the gold standard when regulatory compliance, data integrity, and complex cross-entity queries are paramount. The trade-off is a higher table count (60+), more complex migrations when the schema evolves, and JOIN-heavy queries for reporting.

**Best for:** Institutional-grade platforms where data integrity, regulatory compliance (GIPS, NCREIF/PREA, Reg D), and complex cross-entity analytics are non-negotiable requirements.

**Trade-offs:**
- Pro: Maximum data integrity via foreign keys and constraints
- Pro: Complex analytical queries are straightforward with standard SQL JOINs
- Pro: Well-understood by database administrators and ORM tooling
- Pro: Natural alignment with NCREIF/PREA reporting field requirements
- Con: High table count increases migration complexity
- Con: Schema changes require careful migration planning
- Con: Adding new asset types or deal structures may require new tables
- Con: JOIN-heavy queries can be slower without careful indexing

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| NCREIF PREA Reporting Standards (2025) | Asset-level fields (discount_rate, dscr, weighted_avg_lease_term, ltv) stored as explicit columns in dedicated tables |
| GIPS (CFA Institute) | Fund performance tables structured to produce GIPS-compliant composite returns |
| RESO Data Dictionary | Property attribute fields follow RESO naming conventions where applicable (bedrooms_total, bathrooms_total, living_area) |
| IBPDI Common Data Model | Building and area measurement entities aligned with IBPDI schema clusters |
| ISO 3166-1/2 | Jurisdiction codes for properties and fund registrations use ISO country/subdivision codes |
| ISO 4217 | All monetary amounts paired with a currency_code column using ISO 4217 codes |
| SEC Regulation D | Investor accreditation and Form D tracking in dedicated compliance tables |

---

## Core Identity & Multi-Tenancy

```sql
-- Organization / tenant table
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    org_type        VARCHAR(50) NOT NULL CHECK (org_type IN ('fund_manager', 'syndicator', 'family_office', 'individual', 'brokerage')),
    website         VARCHAR(500),
    logo_url        VARCHAR(500),
    timezone        VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    default_currency VARCHAR(3) NOT NULL DEFAULT 'USD',  -- ISO 4217
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organizations_slug ON organizations (slug);

-- Users within an organization
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           VARCHAR(320) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL CHECK (role IN ('admin', 'manager', 'analyst', 'viewer', 'lp_viewer')),
    phone           VARCHAR(50),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);

CREATE INDEX idx_users_org ON users (organization_id);
CREATE INDEX idx_users_email ON users (email);
```

---

## Property Management

```sql
-- Core property entity
CREATE TABLE properties (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    name                VARCHAR(255) NOT NULL,
    property_type       VARCHAR(50) NOT NULL CHECK (property_type IN (
        'multifamily', 'single_family', 'office', 'retail', 'industrial',
        'mixed_use', 'hotel', 'self_storage', 'data_center', 'land', 'other'
    )),
    property_subtype    VARCHAR(100),
    status              VARCHAR(50) NOT NULL DEFAULT 'prospect' CHECK (status IN (
        'prospect', 'under_analysis', 'under_contract', 'owned', 'sold', 'passed'
    )),

    -- Address (RESO Data Dictionary aligned)
    street_address      VARCHAR(500),
    unit_number         VARCHAR(50),
    city                VARCHAR(100),
    state_or_province   VARCHAR(100),      -- ISO 3166-2 subdivision
    postal_code         VARCHAR(20),
    country_code        CHAR(2) NOT NULL DEFAULT 'US',  -- ISO 3166-1 alpha-2
    county              VARCHAR(100),
    submarket           VARCHAR(200),
    latitude            DECIMAL(10, 7),
    longitude           DECIMAL(10, 7),

    -- Physical characteristics (RESO-aligned field names)
    year_built          INTEGER,
    year_renovated      INTEGER,
    total_units         INTEGER,            -- for multifamily
    rentable_sqft       DECIMAL(12, 2),     -- IBPDI area measurement
    lot_size_sqft       DECIMAL(12, 2),
    lot_size_acres      DECIMAL(10, 4),
    stories             INTEGER,
    parking_spaces      INTEGER,
    bedrooms_total      INTEGER,            -- RESO field name
    bathrooms_total     DECIMAL(4, 1),      -- RESO field name
    living_area         DECIMAL(12, 2),     -- RESO field name

    -- Zoning and entitlements
    zoning_code         VARCHAR(50),
    zoning_description  VARCHAR(500),

    -- Tax information
    tax_parcel_id       VARCHAR(100),
    annual_tax_amount   DECIMAL(14, 2),
    tax_assessment_year INTEGER,

    -- External IDs for data provider integration
    attom_id            VARCHAR(100),
    mls_id              VARCHAR(100),

    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_properties_org ON properties (organization_id);
CREATE INDEX idx_properties_type ON properties (property_type);
CREATE INDEX idx_properties_status ON properties (status);
CREATE INDEX idx_properties_location ON properties (state_or_province, city);
CREATE INDEX idx_properties_geo ON properties USING gist (
    point(longitude, latitude)
);

-- Property images and documents
CREATE TABLE property_documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
    document_type   VARCHAR(50) NOT NULL CHECK (document_type IN (
        'photo', 'floorplan', 'site_plan', 'offering_memorandum', 'appraisal',
        'inspection_report', 'environmental', 'title_report', 'survey',
        'rent_roll', 't12', 'tax_return', 'insurance', 'lease', 'other'
    )),
    file_name       VARCHAR(500) NOT NULL,
    file_url        VARCHAR(1000) NOT NULL,
    file_size_bytes BIGINT,
    mime_type       VARCHAR(100),
    description     TEXT,
    uploaded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_property_docs_property ON property_documents (property_id);

-- Comparable transactions for benchmarking
CREATE TABLE comparables (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id),
    comp_type       VARCHAR(20) NOT NULL CHECK (comp_type IN ('sale', 'lease', 'rent')),
    comp_address    VARCHAR(500),
    comp_city       VARCHAR(100),
    comp_state      VARCHAR(100),
    comp_property_type VARCHAR(50),
    sale_price      DECIMAL(16, 2),
    price_per_sqft  DECIMAL(10, 2),
    price_per_unit  DECIMAL(12, 2),
    cap_rate        DECIMAL(6, 4),
    sale_date       DATE,
    sqft            DECIMAL(12, 2),
    units           INTEGER,
    year_built      INTEGER,
    distance_miles  DECIMAL(6, 2),
    data_source     VARCHAR(100),         -- 'attom', 'batchdata', 'manual', etc.
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comparables_property ON comparables (property_id);
CREATE INDEX idx_comparables_type ON comparables (comp_type);
```

---

## Deal Underwriting

```sql
-- A deal represents an investment opportunity being analysed
CREATE TABLE deals (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    property_id         UUID NOT NULL REFERENCES properties(id),
    deal_name           VARCHAR(255) NOT NULL,
    deal_type           VARCHAR(50) NOT NULL CHECK (deal_type IN (
        'acquisition', 'refinance', 'disposition', 'development', 'value_add'
    )),
    deal_strategy       VARCHAR(50) CHECK (deal_strategy IN (
        'buy_and_hold', 'fix_and_flip', 'brrrr', 'ground_up', 'syndication',
        'joint_venture', 'wholesale', 'nnn_lease'
    )),
    status              VARCHAR(50) NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'in_review', 'approved', 'under_contract', 'closed', 'passed', 'dead'
    )),

    -- Acquisition assumptions
    purchase_price          DECIMAL(16, 2),
    closing_costs_pct       DECIMAL(6, 4) DEFAULT 0.02,
    closing_costs_amount    DECIMAL(14, 2),
    capex_budget            DECIMAL(14, 2),
    total_project_cost      DECIMAL(16, 2),  -- computed or entered

    -- Hold period
    hold_period_months      INTEGER NOT NULL DEFAULT 60,
    acquisition_date        DATE,

    -- Exit assumptions
    exit_cap_rate           DECIMAL(6, 4),
    exit_selling_costs_pct  DECIMAL(6, 4) DEFAULT 0.03,
    terminal_value          DECIMAL(16, 2),

    -- Discount rate for DCF
    discount_rate           DECIMAL(6, 4),    -- NCREIF PREA field

    -- Currency
    currency_code           CHAR(3) NOT NULL DEFAULT 'USD',  -- ISO 4217

    -- AI-sourced flag
    assumptions_source      VARCHAR(50) DEFAULT 'manual' CHECK (assumptions_source IN (
        'manual', 'ai_generated', 'market_data', 'hybrid'
    )),

    assigned_analyst_id     UUID REFERENCES users(id),
    notes                   TEXT,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deals_org ON deals (organization_id);
CREATE INDEX idx_deals_property ON deals (property_id);
CREATE INDEX idx_deals_status ON deals (status);

-- Financing / debt structures attached to a deal
CREATE TABLE deal_financing (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id             UUID NOT NULL REFERENCES deals(id) ON DELETE CASCADE,
    loan_name           VARCHAR(255) NOT NULL DEFAULT 'Senior Debt',
    loan_type           VARCHAR(50) NOT NULL CHECK (loan_type IN (
        'permanent', 'bridge', 'construction', 'mezzanine', 'preferred_equity', 'seller_financing'
    )),
    loan_amount         DECIMAL(16, 2) NOT NULL,
    ltv_ratio           DECIMAL(6, 4),        -- NCREIF PREA field
    interest_rate       DECIMAL(8, 6) NOT NULL,
    rate_type           VARCHAR(20) NOT NULL DEFAULT 'fixed' CHECK (rate_type IN ('fixed', 'variable', 'hybrid')),
    amortization_months INTEGER,
    loan_term_months    INTEGER NOT NULL,
    io_period_months    INTEGER DEFAULT 0,     -- interest-only period
    origination_fee_pct DECIMAL(6, 4) DEFAULT 0.01,
    prepayment_penalty  VARCHAR(100),
    lender_name         VARCHAR(255),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deal_financing_deal ON deal_financing (deal_id);

-- Scenario modelling (bear / base / bull)
CREATE TABLE deal_scenarios (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id             UUID NOT NULL REFERENCES deals(id) ON DELETE CASCADE,
    scenario_name       VARCHAR(100) NOT NULL,  -- 'Bear', 'Base', 'Bull', custom
    scenario_type       VARCHAR(20) NOT NULL DEFAULT 'custom' CHECK (scenario_type IN ('bear', 'base', 'bull', 'custom')),
    is_primary          BOOLEAN NOT NULL DEFAULT false,

    -- Override assumptions for this scenario
    rent_growth_pct     DECIMAL(6, 4),
    expense_growth_pct  DECIMAL(6, 4),
    vacancy_rate_pct    DECIMAL(6, 4),
    exit_cap_rate       DECIMAL(6, 4),
    discount_rate       DECIMAL(6, 4),

    -- Computed results (denormalized for performance)
    irr                 DECIMAL(8, 4),
    npv                 DECIMAL(16, 2),
    equity_multiple     DECIMAL(8, 4),
    cash_on_cash_y1     DECIMAL(8, 4),
    dscr_y1             DECIMAL(8, 4),        -- NCREIF PREA field

    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deal_scenarios_deal ON deal_scenarios (deal_id);
```

---

## Cash Flow Projections

```sql
-- Annual (or monthly) cash flow line items for a scenario
CREATE TABLE cash_flow_projections (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scenario_id         UUID NOT NULL REFERENCES deal_scenarios(id) ON DELETE CASCADE,
    period_number       INTEGER NOT NULL,       -- 1, 2, 3... (year or month)
    period_type         VARCHAR(10) NOT NULL DEFAULT 'annual' CHECK (period_type IN ('annual', 'monthly')),
    period_start_date   DATE NOT NULL,
    period_end_date     DATE NOT NULL,

    -- Revenue
    gross_potential_rent    DECIMAL(14, 2),
    vacancy_loss           DECIMAL(14, 2),
    concessions            DECIMAL(14, 2),
    other_income           DECIMAL(14, 2),
    effective_gross_income DECIMAL(14, 2),

    -- Expenses
    operating_expenses     DECIMAL(14, 2),
    real_estate_taxes      DECIMAL(14, 2),
    insurance              DECIMAL(14, 2),
    management_fee         DECIMAL(14, 2),
    reserves               DECIMAL(14, 2),
    total_expenses         DECIMAL(14, 2),

    -- NOI
    net_operating_income   DECIMAL(14, 2),

    -- Debt service
    debt_service           DECIMAL(14, 2),
    interest_payment       DECIMAL(14, 2),
    principal_payment      DECIMAL(14, 2),

    -- Capital expenditures
    capital_expenditures   DECIMAL(14, 2),

    -- Cash flow
    cash_flow_before_tax   DECIMAL(14, 2),
    cash_flow_after_tax    DECIMAL(14, 2),

    -- Terminal value (only for final period)
    net_sale_proceeds      DECIMAL(16, 2),

    created_at             TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cf_projections_scenario ON cash_flow_projections (scenario_id);
CREATE INDEX idx_cf_projections_period ON cash_flow_projections (scenario_id, period_number);
```

---

## Rent Roll & Leases

```sql
-- Individual lease / unit records
CREATE TABLE leases (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id         UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
    unit_identifier     VARCHAR(100) NOT NULL,   -- unit number, suite, etc.
    tenant_name         VARCHAR(255),
    lease_type          VARCHAR(50) NOT NULL CHECK (lease_type IN (
        'gross', 'net', 'nnn', 'modified_gross', 'percentage', 'month_to_month', 'vacant'
    )),
    status              VARCHAR(30) NOT NULL DEFAULT 'active' CHECK (status IN (
        'active', 'expired', 'pending', 'terminated', 'vacant'
    )),

    -- Rent
    monthly_rent        DECIMAL(12, 2),
    annual_rent         DECIMAL(14, 2),
    rent_per_sqft       DECIMAL(10, 2),
    market_rent_per_sqft DECIMAL(10, 2),       -- for mark-to-market analysis
    unit_sqft           DECIMAL(10, 2),

    -- Lease terms
    lease_start_date    DATE,
    lease_end_date      DATE,
    weighted_avg_lease_term_years DECIMAL(6, 2),  -- NCREIF PREA field

    -- Escalations
    annual_escalation_pct   DECIMAL(6, 4),
    escalation_type         VARCHAR(30) CHECK (escalation_type IN ('fixed', 'cpi', 'market', 'step')),

    -- Tenant improvements and concessions
    ti_allowance_per_sqft   DECIMAL(10, 2),
    free_rent_months        INTEGER DEFAULT 0,

    -- Security
    security_deposit        DECIMAL(12, 2),

    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_leases_property ON leases (property_id);
CREATE INDEX idx_leases_status ON leases (status);
CREATE INDEX idx_leases_expiry ON leases (lease_end_date);

-- Operating statement / T12 line items (historical)
CREATE TABLE operating_statements (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id         UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
    statement_type      VARCHAR(20) NOT NULL CHECK (statement_type IN ('t12', 't3', 'annual', 'monthly', 'budget')),
    period_start        DATE NOT NULL,
    period_end          DATE NOT NULL,
    line_item_category  VARCHAR(100) NOT NULL,   -- 'rental_income', 'utilities', 'repairs', etc.
    line_item_name      VARCHAR(255) NOT NULL,
    amount              DECIMAL(14, 2) NOT NULL,
    is_income           BOOLEAN NOT NULL DEFAULT false,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_op_statements_property ON operating_statements (property_id);
CREATE INDEX idx_op_statements_period ON operating_statements (period_start, period_end);
```

---

## Investor & Syndication Management

```sql
-- Investor entities (LPs)
CREATE TABLE investors (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    investor_type       VARCHAR(30) NOT NULL CHECK (investor_type IN (
        'individual', 'entity', 'trust', 'ira', 'joint'
    )),
    legal_name          VARCHAR(500) NOT NULL,
    display_name        VARCHAR(255),
    email               VARCHAR(320),
    phone               VARCHAR(50),
    tax_id_encrypted    BYTEA,                    -- encrypted SSN/EIN
    mailing_address     TEXT,

    -- Accreditation (SEC Reg D)
    accreditation_status VARCHAR(30) CHECK (accreditation_status IN (
        'accredited', 'non_accredited', 'qualified_purchaser', 'pending', 'expired'
    )),
    accreditation_date   DATE,
    accreditation_expiry DATE,
    accreditation_method VARCHAR(50),             -- 'income', 'net_worth', 'professional', 'entity'

    -- KYC / AML
    kyc_status          VARCHAR(20) DEFAULT 'pending' CHECK (kyc_status IN ('pending', 'verified', 'failed')),
    kyc_verified_date   DATE,

    -- GDPR / CCPA compliance
    data_consent_given  BOOLEAN DEFAULT false,
    data_consent_date   TIMESTAMPTZ,

    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_investors_org ON investors (organization_id);
CREATE INDEX idx_investors_accreditation ON investors (accreditation_status, accreditation_expiry);

-- Syndication / fund structure
CREATE TABLE funds (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id         UUID NOT NULL REFERENCES organizations(id),
    fund_name               VARCHAR(255) NOT NULL,
    fund_type               VARCHAR(50) NOT NULL CHECK (fund_type IN (
        'single_asset', 'blind_pool', 'specified_pool', 'open_end', 'closed_end'
    )),
    entity_type             VARCHAR(50),            -- 'LLC', 'LP', 'REIT', etc.
    ein_encrypted           BYTEA,
    state_of_formation      VARCHAR(100),
    formation_date          DATE,

    -- Offering details
    total_raise_target      DECIMAL(16, 2),
    minimum_investment      DECIMAL(14, 2),
    maximum_investment      DECIMAL(16, 2),

    -- Fee structure
    management_fee_pct      DECIMAL(6, 4),
    acquisition_fee_pct     DECIMAL(6, 4),
    disposition_fee_pct     DECIMAL(6, 4),

    -- Waterfall structure
    preferred_return_pct    DECIMAL(6, 4),          -- e.g., 0.08 for 8%
    gp_promote_pct          DECIMAL(6, 4),          -- above pref
    gp_catchup_pct          DECIMAL(6, 4),          -- 0.50 or 1.00

    -- Reg D tracking
    reg_d_filing_type       VARCHAR(10) CHECK (reg_d_filing_type IN ('506b', '506c')),
    form_d_filed_date       DATE,
    form_d_amendment_dates  DATE[],

    -- Performance (NCREIF PREA aligned, denormalized for dashboards)
    gross_irr               DECIMAL(8, 4),
    net_irr                 DECIMAL(8, 4),
    tvpi                    DECIMAL(8, 4),          -- Total Value to Paid-In
    dpi                     DECIMAL(8, 4),          -- Distributions to Paid-In
    rvpi                    DECIMAL(8, 4),          -- Residual Value to Paid-In

    status                  VARCHAR(30) NOT NULL DEFAULT 'raising' CHECK (status IN (
        'raising', 'closed', 'operating', 'liquidating', 'terminated'
    )),

    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_funds_org ON funds (organization_id);
CREATE INDEX idx_funds_status ON funds (status);

-- Junction: fund <-> property (a fund can hold multiple properties)
CREATE TABLE fund_properties (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fund_id         UUID NOT NULL REFERENCES funds(id) ON DELETE CASCADE,
    property_id     UUID NOT NULL REFERENCES properties(id),
    ownership_pct   DECIMAL(8, 6) NOT NULL DEFAULT 1.0,
    acquisition_date DATE,
    disposition_date DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (fund_id, property_id)
);

CREATE INDEX idx_fund_properties_fund ON fund_properties (fund_id);
CREATE INDEX idx_fund_properties_property ON fund_properties (property_id);

-- Investor commitments to funds
CREATE TABLE investor_commitments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fund_id             UUID NOT NULL REFERENCES funds(id),
    investor_id         UUID NOT NULL REFERENCES investors(id),
    commitment_amount   DECIMAL(16, 2) NOT NULL,
    paid_in_capital     DECIMAL(16, 2) NOT NULL DEFAULT 0,
    ownership_pct       DECIMAL(8, 6),
    investor_class      VARCHAR(30) DEFAULT 'class_a' CHECK (investor_class IN ('class_a', 'class_b', 'gp', 'co_invest')),
    subscription_date   DATE,
    status              VARCHAR(20) NOT NULL DEFAULT 'committed' CHECK (status IN (
        'committed', 'funded', 'partially_funded', 'redeemed'
    )),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (fund_id, investor_id)
);

CREATE INDEX idx_investor_commitments_fund ON investor_commitments (fund_id);
CREATE INDEX idx_investor_commitments_investor ON investor_commitments (investor_id);

-- Waterfall distribution tiers
CREATE TABLE waterfall_tiers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fund_id             UUID NOT NULL REFERENCES funds(id) ON DELETE CASCADE,
    tier_order          INTEGER NOT NULL,
    tier_name           VARCHAR(100) NOT NULL,     -- 'Return of Capital', 'Preferred Return', 'GP Catch-Up', 'Promote Tier 1'
    hurdle_type         VARCHAR(20) CHECK (hurdle_type IN ('irr', 'equity_multiple', 'flat')),
    hurdle_rate         DECIMAL(8, 4),
    lp_split_pct        DECIMAL(6, 4) NOT NULL,
    gp_split_pct        DECIMAL(6, 4) NOT NULL,
    is_catchup          BOOLEAN DEFAULT false,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (fund_id, tier_order)
);

CREATE INDEX idx_waterfall_tiers_fund ON waterfall_tiers (fund_id);

-- Actual distributions made
CREATE TABLE distributions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fund_id             UUID NOT NULL REFERENCES funds(id),
    investor_id         UUID NOT NULL REFERENCES investors(id),
    distribution_date   DATE NOT NULL,
    distribution_type   VARCHAR(30) NOT NULL CHECK (distribution_type IN (
        'cash_flow', 'refinance_proceeds', 'sale_proceeds', 'return_of_capital'
    )),
    gross_amount        DECIMAL(14, 2) NOT NULL,
    net_amount          DECIMAL(14, 2) NOT NULL,    -- after withholding
    waterfall_tier_id   UUID REFERENCES waterfall_tiers(id),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_distributions_fund ON distributions (fund_id);
CREATE INDEX idx_distributions_investor ON distributions (investor_id);
CREATE INDEX idx_distributions_date ON distributions (distribution_date);
```

---

## Portfolio & Performance Tracking

```sql
-- Portfolio groupings (a user-defined way to group properties/funds)
CREATE TABLE portfolios (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_portfolios_org ON portfolios (organization_id);

CREATE TABLE portfolio_properties (
    portfolio_id    UUID NOT NULL REFERENCES portfolios(id) ON DELETE CASCADE,
    property_id     UUID NOT NULL REFERENCES properties(id),
    added_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (portfolio_id, property_id)
);

-- Property valuations over time (NCREIF PREA aligned)
CREATE TABLE property_valuations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id         UUID NOT NULL REFERENCES properties(id),
    valuation_date      DATE NOT NULL,
    valuation_method    VARCHAR(50) NOT NULL CHECK (valuation_method IN (
        'appraisal', 'dcf', 'direct_cap', 'comparable_sales', 'avm', 'cost_approach'
    )),
    appraised_value     DECIMAL(16, 2) NOT NULL,
    cap_rate            DECIMAL(6, 4),
    noi_used            DECIMAL(14, 2),
    discount_rate_used  DECIMAL(6, 4),            -- NCREIF PREA field
    appraiser_name      VARCHAR(255),
    report_document_id  UUID REFERENCES property_documents(id),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_valuations_property ON property_valuations (property_id);
CREATE INDEX idx_valuations_date ON property_valuations (valuation_date);

-- Market data snapshots for a submarket
CREATE TABLE market_data (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    submarket_name      VARCHAR(200) NOT NULL,
    city                VARCHAR(100),
    state_or_province   VARCHAR(100),
    property_type       VARCHAR(50) NOT NULL,
    data_date           DATE NOT NULL,
    data_source         VARCHAR(100) NOT NULL,    -- 'attom', 'batchdata', 'housecanary', 'manual'

    avg_cap_rate        DECIMAL(6, 4),
    avg_rent_per_sqft   DECIMAL(10, 2),
    avg_vacancy_rate    DECIMAL(6, 4),
    avg_price_per_sqft  DECIMAL(10, 2),
    avg_price_per_unit  DECIMAL(12, 2),
    median_sale_price   DECIMAL(16, 2),
    rent_growth_yoy     DECIMAL(6, 4),
    population_growth   DECIMAL(6, 4),
    employment_growth   DECIMAL(6, 4),

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_market_data_submarket ON market_data (submarket_name, property_type);
CREATE INDEX idx_market_data_date ON market_data (data_date);
```

---

## Reporting & Audit

```sql
-- Generated reports (LP quarterly, deal summary, etc.)
CREATE TABLE reports (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    report_type         VARCHAR(50) NOT NULL CHECK (report_type IN (
        'deal_summary', 'lp_quarterly', 'portfolio_performance', 'market_report',
        'investor_k1', 'sensitivity_analysis', 'custom'
    )),
    report_name         VARCHAR(255) NOT NULL,
    report_format       VARCHAR(20) NOT NULL DEFAULT 'pdf' CHECK (report_format IN ('pdf', 'excel', 'html', 'json')),
    generated_by        VARCHAR(50) NOT NULL DEFAULT 'manual' CHECK (generated_by IN ('manual', 'ai_generated', 'scheduled')),
    file_url            VARCHAR(1000),
    -- Link to fund or deal
    fund_id             UUID REFERENCES funds(id),
    deal_id             UUID REFERENCES deals(id),
    period_start        DATE,
    period_end          DATE,
    -- GIPS / NCREIF compliance flag
    is_gips_compliant   BOOLEAN DEFAULT false,
    created_by_user_id  UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_reports_org ON reports (organization_id);
CREATE INDEX idx_reports_type ON reports (report_type);

-- Audit log for compliance
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    user_id         UUID REFERENCES users(id),
    entity_type     VARCHAR(50) NOT NULL,         -- 'deal', 'property', 'investor', etc.
    entity_id       UUID NOT NULL,
    action          VARCHAR(30) NOT NULL CHECK (action IN ('create', 'update', 'delete', 'view', 'export', 'share')),
    changes         JSONB,                         -- { "field": { "old": x, "new": y } }
    ip_address      INET,
    user_agent      VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_org ON audit_log (organization_id);
CREATE INDEX idx_audit_log_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_log_user ON audit_log (user_id);
CREATE INDEX idx_audit_log_created ON audit_log (created_at);

-- AI assumption provenance tracking
CREATE TABLE ai_assumptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id         UUID NOT NULL REFERENCES deals(id),
    scenario_id     UUID REFERENCES deal_scenarios(id),
    field_name      VARCHAR(100) NOT NULL,         -- 'rent_growth_pct', 'vacancy_rate', etc.
    suggested_value DECIMAL(14, 4),
    data_sources    TEXT[] NOT NULL,                -- array of source descriptions
    confidence_score DECIMAL(4, 2),                -- 0.00 to 1.00
    reasoning       TEXT,
    accepted        BOOLEAN,
    accepted_by     UUID REFERENCES users(id),
    accepted_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_assumptions_deal ON ai_assumptions (deal_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Identity & Multi-Tenancy | 2 | organizations, users |
| Property Management | 3 | properties, property_documents, comparables |
| Deal Underwriting | 3 | deals, deal_financing, deal_scenarios |
| Cash Flow Projections | 1 | cash_flow_projections |
| Rent Roll & Leases | 2 | leases, operating_statements |
| Investor & Syndication | 6 | investors, funds, fund_properties, investor_commitments, waterfall_tiers, distributions |
| Portfolio & Performance | 4 | portfolios, portfolio_properties, property_valuations, market_data |
| Reporting & Audit | 3 | reports, audit_log, ai_assumptions |
| **Total** | **24** | |

---

## Key Design Decisions

1. **Explicit columns over JSONB for all financial fields.** Every monetary amount, percentage, and metric is a typed DECIMAL column with constraints. This enables SQL aggregation, indexing, and ensures data integrity at the database level -- critical for financial calculations where silent type coercion or missing fields could produce incorrect IRR/NPV results.

2. **NCREIF PREA field alignment.** Fields like `discount_rate`, `dscr`, `weighted_avg_lease_term`, `ltv_ratio`, `tvpi`, `dpi`, `rvpi` are stored as explicit columns matching the 2025 NCREIF PREA Reporting Standards asset- and fund-level reporting requirements.

3. **Multi-tenant via organization_id foreign keys.** Every tenant-scoped table includes an `organization_id` column. Row Level Security (RLS) policies should be applied in production to enforce tenant isolation at the database level.

4. **Waterfall tiers as a separate table.** Rather than encoding the GP/LP waterfall structure as a JSONB blob, each tier is a row in `waterfall_tiers` with explicit split percentages and hurdle rates. This enables SQL-based waterfall calculations and auditable tier-by-tier distribution tracking.

5. **Scenario modelling as first-class entities.** Each deal can have multiple scenarios (bear/base/bull/custom), each with its own assumption overrides and computed results. Cash flow projections are linked to scenarios, not directly to deals, enabling side-by-side comparison.

6. **AI assumption provenance.** The `ai_assumptions` table tracks every AI-generated assumption with its data sources, confidence score, and whether the analyst accepted it. This provides the audit trail institutional investors require for AI-sourced underwriting inputs.

7. **Encrypted sensitive fields.** Tax IDs (SSN/EIN) are stored as `BYTEA` (encrypted at the application layer) rather than plaintext `VARCHAR`. GDPR consent tracking is included on the investor record.

8. **RESO Data Dictionary alignment for property attributes.** Field names like `bedrooms_total`, `bathrooms_total`, and `living_area` follow RESO naming conventions to simplify MLS data import via RESO Web API.

9. **Separation of historical data (operating_statements) from projections (cash_flow_projections).** Historical T12 and actuals are stored independently from forward-looking projections, enabling comparison of projected vs. actual performance.

10. **Market data as a standalone table.** Submarket-level metrics from data providers (ATTOM, BatchData, HouseCanary) are stored in a dedicated `market_data` table, decoupled from individual properties. This enables AI assumption generation by querying current submarket benchmarks.
