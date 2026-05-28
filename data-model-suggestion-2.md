# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Real Estate Investment Analysis · Created: 2026-05-20

## Philosophy

This model uses PostgreSQL's JSONB columns strategically alongside relational tables to handle the inherent variability in real estate investment analysis. The core structural entities (organizations, users, properties, deals, investors) remain fully relational with typed columns and foreign keys. However, domain areas with high variability -- asset-type-specific attributes, jurisdiction-dependent fields, configurable waterfall structures, and multi-format financial inputs -- use JSONB columns that flex without schema migrations.

Real estate is fundamentally heterogeneous: a multifamily deal has units and rent rolls, a NNN retail deal has single-tenant lease structures, a fix-and-flip has rehab budgets and ARV calculations, and a ground-up development has construction draws and absorption schedules. A fully normalized model requires either a wide sparse table or a proliferation of asset-type-specific tables. The JSONB hybrid avoids both problems by storing variable attributes in validated JSONB while keeping queryable financial metrics as typed columns.

This pattern is widely used in modern SaaS platforms (Stripe, Shopify, HubSpot) where the core entities are stable but edge-case variability is high. PostgreSQL's GIN indexes on JSONB columns provide efficient querying, and JSON Schema validation (enforced at the application layer or via CHECK constraints) prevents garbage data.

**Best for:** Rapid MVP development targeting the mid-market (syndicators, family offices, small PE funds) where supporting multiple asset types and deal structures quickly is more important than institutional-grade schema rigidity.

**Trade-offs:**
- Pro: Fewer tables (roughly half the normalized model), faster iteration
- Pro: New asset types and deal structures require no migrations -- just new JSONB shapes
- Pro: Jurisdiction-specific or custom fields are trivial to add per-tenant
- Pro: Natural fit for API responses -- JSONB columns map directly to JSON API payloads
- Con: JSONB fields lack database-level type enforcement -- validation shifts to application layer
- Con: Complex JSONB queries can be less intuitive than JOINs for analysts writing ad hoc SQL
- Con: Schema documentation requires discipline -- JSONB shapes must be documented externally
- Con: Reporting queries on nested JSONB fields are slower than indexed relational columns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| NCREIF PREA Reporting Standards (2025) | Core performance metrics (IRR, TVPI, DPI, RVPI, DSCR) stored as typed columns; extended asset-level reporting fields in JSONB |
| RESO Data Dictionary | Property attributes in JSONB follow RESO field naming conventions; JSONB shape documented per RESO resource type |
| IBPDI Common Data Model | Building attributes cluster stored as JSONB aligned with IBPDI entity definitions |
| ISO 4217 | Currency codes as typed columns on monetary records |
| ISO 3166-1/2 | Country and subdivision codes on address objects within property JSONB |
| SEC Regulation D | Investor accreditation tracking as typed columns; Form D filing details in JSONB |
| JSON Schema | JSONB columns validated against published JSON Schema definitions at the application layer |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    org_type        VARCHAR(50) NOT NULL,
    default_currency VARCHAR(3) NOT NULL DEFAULT 'USD',
    -- Flexible settings: branding, defaults, feature flags, notification prefs
    settings        JSONB NOT NULL DEFAULT '{}',
    /*
    settings example:
    {
        "branding": { "primary_color": "#1a365d", "logo_url": "..." },
        "defaults": {
            "hold_period_months": 60,
            "discount_rate": 0.08,
            "closing_costs_pct": 0.02
        },
        "features": { "ai_assumptions": true, "lp_portal": true },
        "notification_prefs": { "deal_status_change": "email", "distribution_posted": "email" }
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           VARCHAR(320) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL,
    profile         JSONB NOT NULL DEFAULT '{}',
    /*
    profile example:
    {
        "phone": "+1-555-0100",
        "title": "VP Acquisitions",
        "preferences": { "timezone": "America/Chicago", "date_format": "MM/DD/YYYY" }
    }
    */
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

## Properties

```sql
CREATE TABLE properties (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    name                VARCHAR(255) NOT NULL,
    property_type       VARCHAR(50) NOT NULL,
    status              VARCHAR(50) NOT NULL DEFAULT 'prospect',

    -- Core address fields (always queried, so relational)
    city                VARCHAR(100),
    state_or_province   VARCHAR(100),
    country_code        CHAR(2) NOT NULL DEFAULT 'US',
    postal_code         VARCHAR(20),
    latitude            DECIMAL(10, 7),
    longitude           DECIMAL(10, 7),

    -- All other property attributes in JSONB (flexible per asset type)
    address_details     JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "street_address": "123 Main St",
        "unit_number": "Suite 200",
        "county": "Travis",
        "submarket": "Austin CBD"
    }
    */

    physical_attributes JSONB NOT NULL DEFAULT '{}',
    /*
    Multifamily example:
    {
        "year_built": 2005,
        "total_units": 200,
        "rentable_sqft": 180000,
        "stories": 3,
        "parking_spaces": 250,
        "amenities": ["pool", "fitness_center", "dog_park"],
        "unit_mix": [
            { "type": "1BR/1BA", "count": 80, "avg_sqft": 750, "avg_rent": 1400 },
            { "type": "2BR/2BA", "count": 100, "avg_sqft": 1050, "avg_rent": 1850 },
            { "type": "3BR/2BA", "count": 20, "avg_sqft": 1300, "avg_rent": 2200 }
        ]
    }

    Single-family fix-and-flip example:
    {
        "year_built": 1985,
        "bedrooms_total": 3,
        "bathrooms_total": 2,
        "living_area": 1800,
        "lot_size_sqft": 7200,
        "garage_spaces": 2,
        "condition": "fair",
        "arv_estimate": 350000,
        "rehab_scope": [
            { "item": "kitchen_remodel", "estimate": 25000 },
            { "item": "bathroom_remodel", "estimate": 12000, "count": 2 },
            { "item": "flooring", "estimate": 8000 },
            { "item": "paint_exterior", "estimate": 5000 }
        ]
    }

    NNN Retail example:
    {
        "year_built": 2010,
        "rentable_sqft": 5000,
        "lot_size_acres": 0.75,
        "tenant_name": "Walgreens",
        "lease_type": "absolute_nnn",
        "remaining_lease_years": 12,
        "credit_rating": "BBB"
    }
    */

    tax_info            JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "parcel_id": "12345-67-890",
        "annual_tax": 45000,
        "assessment_year": 2025,
        "assessed_value": 2800000
    }
    */

    external_ids        JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "attom_id": "AT-12345",
        "mls_id": "MLS-67890",
        "batchdata_id": "BD-11111"
    }
    */

    zoning              JSONB NOT NULL DEFAULT '{}',
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_properties_org ON properties (organization_id);
CREATE INDEX idx_properties_type ON properties (property_type);
CREATE INDEX idx_properties_status ON properties (status);
CREATE INDEX idx_properties_location ON properties (state_or_province, city);
CREATE INDEX idx_properties_geo ON properties USING gist (point(longitude, latitude));
-- GIN indexes for JSONB querying
CREATE INDEX idx_properties_physical ON properties USING gin (physical_attributes);
CREATE INDEX idx_properties_external ON properties USING gin (external_ids);
```

---

## Deals & Underwriting

```sql
CREATE TABLE deals (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    property_id         UUID NOT NULL REFERENCES properties(id),
    deal_name           VARCHAR(255) NOT NULL,
    deal_type           VARCHAR(50) NOT NULL,
    deal_strategy       VARCHAR(50),
    status              VARCHAR(50) NOT NULL DEFAULT 'draft',
    currency_code       CHAR(3) NOT NULL DEFAULT 'USD',

    -- Core financial assumptions (always queried, typed columns)
    purchase_price          DECIMAL(16, 2),
    total_project_cost      DECIMAL(16, 2),
    hold_period_months      INTEGER NOT NULL DEFAULT 60,
    exit_cap_rate           DECIMAL(6, 4),
    discount_rate           DECIMAL(6, 4),

    -- Variable assumptions in JSONB (differ by deal type)
    acquisition_assumptions JSONB NOT NULL DEFAULT '{}',
    /*
    Buy-and-hold example:
    {
        "closing_costs_pct": 0.02,
        "capex_budget": 150000,
        "capex_schedule": [
            { "year": 1, "amount": 100000, "description": "Unit renovations" },
            { "year": 3, "amount": 50000, "description": "Common area refresh" }
        ],
        "acquisition_date": "2026-06-01"
    }

    Fix-and-flip example:
    {
        "closing_costs_pct": 0.015,
        "rehab_budget": 75000,
        "rehab_timeline_months": 4,
        "arv": 350000,
        "holding_costs_monthly": 2500,
        "selling_costs_pct": 0.06
    }

    Development example:
    {
        "land_cost": 2000000,
        "hard_costs": 15000000,
        "soft_costs": 3000000,
        "construction_months": 18,
        "absorption_months": 12,
        "draw_schedule": [
            { "month": 1, "pct": 0.10 },
            { "month": 6, "pct": 0.30 },
            { "month": 12, "pct": 0.35 },
            { "month": 18, "pct": 0.25 }
        ]
    }
    */

    exit_assumptions    JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "exit_cap_rate": 0.055,
        "selling_costs_pct": 0.03,
        "terminal_noi": 850000,
        "terminal_value": 15454545
    }
    */

    -- AI provenance tracking
    ai_metadata         JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "assumptions_source": "hybrid",
        "ai_suggestions": [
            {
                "field": "exit_cap_rate",
                "value": 0.055,
                "confidence": 0.82,
                "sources": ["ATTOM submarket data Q1 2026", "3 comparable sales"],
                "accepted": true,
                "accepted_by": "user-uuid",
                "accepted_at": "2026-05-15T10:30:00Z"
            }
        ]
    }
    */

    assigned_analyst_id     UUID REFERENCES users(id),
    notes                   TEXT,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deals_org ON deals (organization_id);
CREATE INDEX idx_deals_property ON deals (property_id);
CREATE INDEX idx_deals_status ON deals (status);
CREATE INDEX idx_deals_ai ON deals USING gin (ai_metadata);

-- Financing structures
CREATE TABLE deal_financing (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id             UUID NOT NULL REFERENCES deals(id) ON DELETE CASCADE,
    loan_name           VARCHAR(255) NOT NULL DEFAULT 'Senior Debt',
    loan_type           VARCHAR(50) NOT NULL,
    loan_amount         DECIMAL(16, 2) NOT NULL,
    interest_rate       DECIMAL(8, 6) NOT NULL,
    loan_term_months    INTEGER NOT NULL,

    -- Variable loan terms in JSONB
    loan_details        JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "rate_type": "fixed",
        "ltv_ratio": 0.65,
        "amortization_months": 360,
        "io_period_months": 24,
        "origination_fee_pct": 0.01,
        "prepayment_penalty": "yield_maintenance_3yr",
        "lender_name": "Regional Bank Corp",
        "rate_cap": { "strike": 0.065, "term_months": 36, "cost": 125000 },
        "covenants": {
            "min_dscr": 1.25,
            "max_ltv": 0.75,
            "min_debt_yield": 0.08
        }
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deal_financing_deal ON deal_financing (deal_id);
```

---

## Scenarios & Cash Flows

```sql
CREATE TABLE deal_scenarios (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id             UUID NOT NULL REFERENCES deals(id) ON DELETE CASCADE,
    scenario_name       VARCHAR(100) NOT NULL,
    scenario_type       VARCHAR(20) NOT NULL DEFAULT 'custom',
    is_primary          BOOLEAN NOT NULL DEFAULT false,

    -- Core assumptions as typed columns (always compared across scenarios)
    rent_growth_pct     DECIMAL(6, 4),
    expense_growth_pct  DECIMAL(6, 4),
    vacancy_rate_pct    DECIMAL(6, 4),
    exit_cap_rate       DECIMAL(6, 4),
    discount_rate       DECIMAL(6, 4),

    -- Computed results (typed for dashboards and comparisons)
    irr                 DECIMAL(8, 4),
    npv                 DECIMAL(16, 2),
    equity_multiple     DECIMAL(8, 4),
    cash_on_cash_y1     DECIMAL(8, 4),
    dscr_y1             DECIMAL(8, 4),

    -- Extended / custom assumptions in JSONB
    custom_assumptions  JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "rent_loss_to_lease_pct": 0.03,
        "capex_per_unit_annual": 500,
        "management_fee_pct": 0.04,
        "insurance_growth_pct": 0.05,
        "tax_growth_pct": 0.03,
        "concessions_pct": 0.02,
        "year_specific_overrides": {
            "1": { "vacancy_rate_pct": 0.12, "concessions_pct": 0.05 },
            "2": { "vacancy_rate_pct": 0.08 }
        }
    }
    */

    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deal_scenarios_deal ON deal_scenarios (deal_id);

-- Cash flow projections stored as JSONB array per scenario
-- Instead of one row per period, the entire cash flow stream is one document
CREATE TABLE cash_flow_streams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scenario_id     UUID NOT NULL REFERENCES deal_scenarios(id) ON DELETE CASCADE,
    period_type     VARCHAR(10) NOT NULL DEFAULT 'annual',
    periods         JSONB NOT NULL,
    /*
    [
        {
            "period": 1,
            "start_date": "2026-06-01",
            "end_date": "2027-05-31",
            "gross_potential_rent": 2400000,
            "vacancy_loss": -192000,
            "other_income": 48000,
            "effective_gross_income": 2256000,
            "operating_expenses": -900000,
            "real_estate_taxes": -180000,
            "insurance": -60000,
            "management_fee": -90240,
            "reserves": -40000,
            "net_operating_income": 985760,
            "debt_service": -650000,
            "capital_expenditures": -100000,
            "cash_flow_before_tax": 235760
        },
        {
            "period": 2,
            "start_date": "2027-06-01",
            ...
        }
    ]
    */

    -- Summary metrics (denormalized for quick access)
    total_noi           DECIMAL(16, 2),
    total_cash_flow     DECIMAL(16, 2),
    avg_cash_on_cash    DECIMAL(8, 4),

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (scenario_id)
);

CREATE INDEX idx_cf_streams_scenario ON cash_flow_streams (scenario_id);
```

---

## Rent Roll & Operating History

```sql
-- Rent roll as a combination of relational + JSONB
CREATE TABLE rent_rolls (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
    as_of_date      DATE NOT NULL,
    source          VARCHAR(50) NOT NULL DEFAULT 'manual',  -- 'manual', 'csv_import', 'ai_extracted'
    summary         JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "total_units": 200,
        "occupied_units": 186,
        "occupancy_pct": 0.93,
        "total_monthly_rent": 285000,
        "avg_rent_per_unit": 1532,
        "avg_rent_per_sqft": 1.72,
        "weighted_avg_lease_term_months": 8.5
    }
    */
    units           JSONB NOT NULL,
    /*
    [
        {
            "unit": "101",
            "type": "1BR/1BA",
            "sqft": 750,
            "status": "occupied",
            "tenant": "Smith, J.",
            "monthly_rent": 1400,
            "market_rent": 1500,
            "lease_start": "2025-08-01",
            "lease_end": "2026-07-31",
            "deposit": 1400,
            "escalation_pct": 0.03
        },
        {
            "unit": "102",
            "type": "2BR/2BA",
            "sqft": 1050,
            "status": "vacant",
            "market_rent": 1850,
            "days_vacant": 22
        }
    ]
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rent_rolls_property ON rent_rolls (property_id);
CREATE INDEX idx_rent_rolls_date ON rent_rolls (as_of_date);

-- Operating statements (T12, annual, budget) as structured documents
CREATE TABLE operating_statements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
    statement_type  VARCHAR(20) NOT NULL,          -- 't12', 'annual', 'budget', 'monthly'
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    source          VARCHAR(50) NOT NULL DEFAULT 'manual',

    -- Top-level summary as typed columns for quick queries
    total_revenue       DECIMAL(14, 2),
    total_expenses      DECIMAL(14, 2),
    net_operating_income DECIMAL(14, 2),

    -- Detailed line items in JSONB
    line_items      JSONB NOT NULL,
    /*
    {
        "revenue": [
            { "category": "rental_income", "label": "Gross Potential Rent", "amount": 2880000 },
            { "category": "vacancy", "label": "Vacancy & Credit Loss", "amount": -230400 },
            { "category": "other_income", "label": "Laundry Income", "amount": 24000 },
            { "category": "other_income", "label": "Parking Income", "amount": 36000 },
            { "category": "other_income", "label": "Late Fees", "amount": 8400 }
        ],
        "expenses": [
            { "category": "payroll", "label": "On-Site Payroll", "amount": 180000 },
            { "category": "repairs", "label": "Repairs & Maintenance", "amount": 120000 },
            { "category": "utilities", "label": "Water & Sewer", "amount": 96000 },
            { "category": "utilities", "label": "Electric (Common Area)", "amount": 48000 },
            { "category": "taxes", "label": "Real Estate Taxes", "amount": 180000 },
            { "category": "insurance", "label": "Property Insurance", "amount": 60000 },
            { "category": "management", "label": "Management Fee", "amount": 107520 },
            { "category": "admin", "label": "Legal & Accounting", "amount": 24000 },
            { "category": "marketing", "label": "Advertising & Marketing", "amount": 18000 }
        ]
    }
    */

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_op_statements_property ON operating_statements (property_id);
CREATE INDEX idx_op_statements_period ON operating_statements (period_start, period_end);
```

---

## Investors & Syndication

```sql
CREATE TABLE investors (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    investor_type       VARCHAR(30) NOT NULL,
    legal_name          VARCHAR(500) NOT NULL,
    email               VARCHAR(320),

    -- Accreditation (typed for compliance queries)
    accreditation_status VARCHAR(30),
    accreditation_expiry DATE,

    -- KYC / AML
    kyc_status          VARCHAR(20) DEFAULT 'pending',
    kyc_verified_date   DATE,

    -- All other investor details in JSONB (varies by investor type)
    details             JSONB NOT NULL DEFAULT '{}',
    /*
    Individual example:
    {
        "display_name": "John Smith",
        "phone": "+1-555-0100",
        "mailing_address": { "street": "456 Oak Ave", "city": "Austin", "state": "TX", "zip": "78701" },
        "tax_id_encrypted": "...",
        "accreditation_method": "income",
        "accreditation_date": "2026-01-15",
        "data_consent_given": true,
        "data_consent_date": "2026-01-15T00:00:00Z"
    }

    Entity example:
    {
        "display_name": "Smith Family Trust",
        "entity_type": "revocable_trust",
        "trustee_name": "John Smith",
        "phone": "+1-555-0100",
        "ein_encrypted": "...",
        "state_of_formation": "Texas",
        "authorized_signers": ["John Smith", "Jane Smith"]
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_investors_org ON investors (organization_id);
CREATE INDEX idx_investors_accreditation ON investors (accreditation_status, accreditation_expiry);

CREATE TABLE funds (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id         UUID NOT NULL REFERENCES organizations(id),
    fund_name               VARCHAR(255) NOT NULL,
    fund_type               VARCHAR(50) NOT NULL,
    status                  VARCHAR(30) NOT NULL DEFAULT 'raising',
    total_raise_target      DECIMAL(16, 2),
    minimum_investment      DECIMAL(14, 2),

    -- Waterfall structure as JSONB (fully flexible for any structure)
    waterfall_structure     JSONB NOT NULL DEFAULT '[]',
    /*
    [
        {
            "tier": 1,
            "name": "Return of Capital",
            "hurdle_type": "flat",
            "lp_split": 1.00,
            "gp_split": 0.00
        },
        {
            "tier": 2,
            "name": "Preferred Return",
            "hurdle_type": "irr",
            "hurdle_rate": 0.08,
            "lp_split": 1.00,
            "gp_split": 0.00
        },
        {
            "tier": 3,
            "name": "GP Catch-Up",
            "hurdle_type": "irr",
            "hurdle_rate": 0.08,
            "is_catchup": true,
            "catchup_pct": 1.00,
            "lp_split": 0.00,
            "gp_split": 1.00,
            "catchup_target_gp_share": 0.20
        },
        {
            "tier": 4,
            "name": "Residual Split",
            "lp_split": 0.80,
            "gp_split": 0.20
        }
    ]
    */

    -- Fee structure and regulatory details in JSONB
    fund_details        JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "entity_type": "LLC",
        "state_of_formation": "Delaware",
        "formation_date": "2026-03-01",
        "management_fee_pct": 0.015,
        "acquisition_fee_pct": 0.01,
        "disposition_fee_pct": 0.01,
        "reg_d_filing_type": "506c",
        "form_d_filed_date": "2026-04-01",
        "investor_classes": [
            { "class": "A", "preferred_return": 0.07, "promote_above_pref": 0.20 },
            { "class": "B", "preferred_return": 0.10, "promote_above_pref": 0.35 }
        ]
    }
    */

    -- NCREIF PREA performance metrics (typed for benchmarking queries)
    gross_irr           DECIMAL(8, 4),
    net_irr             DECIMAL(8, 4),
    tvpi                DECIMAL(8, 4),
    dpi                 DECIMAL(8, 4),
    rvpi                DECIMAL(8, 4),

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_funds_org ON funds (organization_id);
CREATE INDEX idx_funds_status ON funds (status);

-- Fund <-> Property junction
CREATE TABLE fund_properties (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fund_id         UUID NOT NULL REFERENCES funds(id) ON DELETE CASCADE,
    property_id     UUID NOT NULL REFERENCES properties(id),
    ownership_pct   DECIMAL(8, 6) NOT NULL DEFAULT 1.0,
    details         JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "acquisition_date": "2026-06-15",
        "acquisition_price": 12500000,
        "disposition_date": null
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (fund_id, property_id)
);

-- Investor commitments
CREATE TABLE investor_commitments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fund_id             UUID NOT NULL REFERENCES funds(id),
    investor_id         UUID NOT NULL REFERENCES investors(id),
    commitment_amount   DECIMAL(16, 2) NOT NULL,
    paid_in_capital     DECIMAL(16, 2) NOT NULL DEFAULT 0,
    ownership_pct       DECIMAL(8, 6),
    investor_class      VARCHAR(30) DEFAULT 'class_a',
    status              VARCHAR(20) NOT NULL DEFAULT 'committed',
    details             JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "subscription_date": "2026-04-01",
        "side_letter_terms": {
            "reduced_management_fee": 0.01,
            "co_invest_rights": true,
            "mfn_clause": true
        }
    }
    */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (fund_id, investor_id)
);

CREATE INDEX idx_commitments_fund ON investor_commitments (fund_id);
CREATE INDEX idx_commitments_investor ON investor_commitments (investor_id);

-- Distributions
CREATE TABLE distributions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fund_id             UUID NOT NULL REFERENCES funds(id),
    investor_id         UUID NOT NULL REFERENCES investors(id),
    distribution_date   DATE NOT NULL,
    distribution_type   VARCHAR(30) NOT NULL,
    gross_amount        DECIMAL(14, 2) NOT NULL,
    net_amount          DECIMAL(14, 2) NOT NULL,
    waterfall_details   JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "tier_applied": "Preferred Return",
        "cumulative_irr_at_distribution": 0.092,
        "cumulative_equity_multiple": 1.15,
        "breakdown": {
            "return_of_capital": 5000,
            "preferred_return": 8000,
            "promote": 0
        }
    }
    */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_distributions_fund ON distributions (fund_id);
CREATE INDEX idx_distributions_investor ON distributions (investor_id);
CREATE INDEX idx_distributions_date ON distributions (distribution_date);
```

---

## Portfolio, Market Data & Reports

```sql
CREATE TABLE portfolios (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE portfolio_properties (
    portfolio_id    UUID NOT NULL REFERENCES portfolios(id) ON DELETE CASCADE,
    property_id     UUID NOT NULL REFERENCES properties(id),
    PRIMARY KEY (portfolio_id, property_id)
);

-- Market data with JSONB for variable metric sets per source
CREATE TABLE market_data (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    submarket       VARCHAR(200) NOT NULL,
    property_type   VARCHAR(50) NOT NULL,
    data_date       DATE NOT NULL,
    data_source     VARCHAR(100) NOT NULL,
    metrics         JSONB NOT NULL,
    /*
    {
        "avg_cap_rate": 0.055,
        "avg_rent_per_sqft": 1.85,
        "avg_vacancy_rate": 0.06,
        "avg_price_per_unit": 185000,
        "median_sale_price": 14500000,
        "rent_growth_yoy": 0.034,
        "population_growth": 0.018,
        "employment_growth": 0.022,
        "new_construction_units": 1200,
        "absorption_rate": 0.85
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_market_data_lookup ON market_data (submarket, property_type, data_date);
CREATE INDEX idx_market_data_metrics ON market_data USING gin (metrics);

-- Property valuations
CREATE TABLE property_valuations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id         UUID NOT NULL REFERENCES properties(id),
    valuation_date      DATE NOT NULL,
    appraised_value     DECIMAL(16, 2) NOT NULL,
    valuation_details   JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "method": "dcf",
        "cap_rate": 0.055,
        "noi_used": 850000,
        "discount_rate": 0.08,
        "appraiser_name": "CBRE Valuation",
        "is_draft": false
    }
    */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_valuations_property ON property_valuations (property_id);

-- Comparables
CREATE TABLE comparables (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id),
    comp_type       VARCHAR(20) NOT NULL,
    comp_data       JSONB NOT NULL,
    /*
    {
        "address": "789 Elm St, Austin, TX",
        "property_type": "multifamily",
        "sale_price": 14200000,
        "price_per_unit": 177500,
        "cap_rate": 0.052,
        "sale_date": "2026-02-15",
        "units": 80,
        "year_built": 2008,
        "distance_miles": 1.2,
        "data_source": "attom"
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comparables_property ON comparables (property_id);

-- Documents (property or deal level)
CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    property_id     UUID REFERENCES properties(id),
    deal_id         UUID REFERENCES deals(id),
    fund_id         UUID REFERENCES funds(id),
    document_type   VARCHAR(50) NOT NULL,
    file_name       VARCHAR(500) NOT NULL,
    file_url        VARCHAR(1000) NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "file_size_bytes": 2456789,
        "mime_type": "application/pdf",
        "description": "2025 T12 Operating Statement",
        "uploaded_by": "user-uuid",
        "ai_extracted": true,
        "extraction_confidence": 0.95
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_documents_org ON documents (organization_id);
CREATE INDEX idx_documents_property ON documents (property_id);
CREATE INDEX idx_documents_deal ON documents (deal_id);

-- Reports
CREATE TABLE reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    report_type     VARCHAR(50) NOT NULL,
    report_name     VARCHAR(255) NOT NULL,
    fund_id         UUID REFERENCES funds(id),
    deal_id         UUID REFERENCES deals(id),
    file_url        VARCHAR(1000),
    report_config   JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "format": "pdf",
        "generated_by": "ai_generated",
        "period_start": "2026-01-01",
        "period_end": "2026-03-31",
        "is_gips_compliant": false,
        "sections_included": ["executive_summary", "financial_performance", "market_update", "outlook"]
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_reports_org ON reports (organization_id);

-- Audit log
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    user_id         UUID REFERENCES users(id),
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(30) NOT NULL,
    changes         JSONB,
    request_context JSONB,
    /*
    {
        "ip_address": "192.168.1.100",
        "user_agent": "Mozilla/5.0...",
        "session_id": "sess-abc123"
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_log_org_time ON audit_log (organization_id, created_at);
```

---

## Example Queries

### Querying JSONB: Find all multifamily properties with 100+ units

```sql
SELECT id, name, city, state_or_province,
       (physical_attributes->>'total_units')::int AS total_units
FROM properties
WHERE property_type = 'multifamily'
  AND (physical_attributes->>'total_units')::int >= 100
  AND organization_id = $1
ORDER BY (physical_attributes->>'total_units')::int DESC;
```

### Querying JSONB: Find vacant units in a rent roll

```sql
SELECT r.id, r.as_of_date, unit->>'unit' AS unit_number,
       (unit->>'market_rent')::numeric AS market_rent,
       (unit->>'days_vacant')::int AS days_vacant
FROM rent_rolls r,
     jsonb_array_elements(r.units) AS unit
WHERE r.property_id = $1
  AND unit->>'status' = 'vacant'
ORDER BY (unit->>'days_vacant')::int DESC;
```

### Querying JSONB: Calculate total distribution by waterfall tier

```sql
SELECT d.distribution_date,
       d.waterfall_details->>'tier_applied' AS tier,
       SUM(d.gross_amount) AS total_distributed
FROM distributions d
WHERE d.fund_id = $1
GROUP BY d.distribution_date, d.waterfall_details->>'tier_applied'
ORDER BY d.distribution_date;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Identity & Multi-Tenancy | 2 | organizations, users |
| Properties | 1 | Single table with JSONB for asset-type-specific attributes |
| Deals & Underwriting | 2 | deals, deal_financing |
| Scenarios & Cash Flows | 2 | deal_scenarios, cash_flow_streams |
| Rent Roll & Operating History | 2 | rent_rolls (JSONB array of units), operating_statements |
| Investors & Syndication | 5 | investors, funds, fund_properties, investor_commitments, distributions |
| Portfolio & Market Data | 5 | portfolios, portfolio_properties, market_data, property_valuations, comparables |
| Documents & Reports | 2 | documents, reports |
| Audit | 1 | audit_log |
| **Total** | **22** | Compared to 24 in the normalized model, but with significantly more flexibility |

---

## Key Design Decisions

1. **JSONB for asset-type variability, typed columns for financial metrics.** The `physical_attributes` JSONB on properties holds 100+ possible fields that vary by asset type (unit mix for multifamily, rehab scope for flips, credit ratings for NNN). But IRR, NPV, equity multiple, and DSCR are always typed DECIMAL columns because they drive dashboards, sorting, and filtering.

2. **Rent roll as a JSONB document.** Rather than a separate `leases` table with one row per unit, the entire rent roll is stored as a JSONB array within `rent_rolls`. This matches how rent rolls are imported (as spreadsheets) and consumed (as complete snapshots). Historical rent rolls are preserved as separate rows with different `as_of_date` values.

3. **Waterfall structure as JSONB array.** The GP/LP waterfall is stored as an ordered JSON array on the `funds` table rather than a separate `waterfall_tiers` table. This allows arbitrarily complex structures (multiple LP classes, hybrid IRR/equity multiple hurdles, European vs. American waterfalls) without schema changes.

4. **Cash flow stream as single JSONB document.** Instead of one row per period in `cash_flow_projections`, the entire cash flow stream for a scenario is one JSONB document. This simplifies the common operation of reading/writing the entire stream and reduces JOIN overhead for report generation.

5. **GIN indexes on JSONB columns.** PostgreSQL GIN indexes on `physical_attributes`, `external_ids`, `ai_metadata`, and `metrics` enable efficient containment queries (`@>`) without sacrificing flexibility.

6. **AI metadata embedded in deals.** Rather than a separate `ai_assumptions` table, AI suggestion provenance is stored directly on the deal record. This keeps the AI audit trail co-located with the data it describes and simplifies API responses.

7. **Documents as a polymorphic table.** A single `documents` table with optional foreign keys to `properties`, `deals`, and `funds` replaces asset-type-specific document tables. The `metadata` JSONB handles variable document attributes.

8. **Application-layer JSON Schema validation.** Each JSONB column has a documented JSON Schema (shown in comments). Validation is enforced at the API/service layer rather than via database CHECK constraints, allowing schema evolution without migrations.

9. **Summary fields denormalized on parent records.** `rent_rolls.summary`, `cash_flow_streams.total_noi`, and `operating_statements.total_revenue` provide pre-computed aggregates so dashboard queries avoid JSONB traversal.

10. **Fewer tables, same domain coverage.** 22 tables vs. 24 in the normalized model, but with far more flexibility for new asset types, custom fields, and jurisdiction-specific requirements -- all without schema migrations.
