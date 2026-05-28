# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Real Estate Investment Analysis · Created: 2026-05-20

## Philosophy

This model combines a relational PostgreSQL core for transactional CRUD operations with a property graph layer for relationship-heavy queries. The relational tables handle standard operations: creating deals, recording cash flows, managing investor records, and generating reports. The graph layer -- implemented either as PostgreSQL tables (`graph_nodes` / `graph_edges`) or as a dedicated graph database (Neo4j, Apache AGE for PostgreSQL) -- handles queries that traverse ownership chains, investor networks, fund-to-property-to-market relationships, and conflict-of-interest analysis.

Real estate investment is fundamentally a relationship domain. An investor participates in multiple funds, each fund owns stakes in multiple properties, each property sits in a submarket, each submarket has comparable transactions, and the investor may also be a guarantor on a loan for a different property in the same fund. These multi-hop relationship queries are exactly where graph databases excel and where relational JOINs become unwieldy: "Show me all properties within 2 degrees of separation from Investor X" or "Find all investors who have exposure to Austin multifamily through any fund."

Cherre's Agent.STUDIO, the most architecturally advanced competitor, uses a Knowledge Graph as its core data unification layer. This model takes a similar approach but keeps the operational database relational (for transactional integrity and compatibility with standard ORMs) while adding a synchronised graph layer for analytics, discovery, and AI-powered insight generation.

**Best for:** Platforms that need to surface complex relationship insights -- ownership network analysis, investor conflict-of-interest detection, geographic concentration risk, cross-fund exposure mapping, and AI-powered deal recommendation based on relationship context.

**Trade-offs:**
- Pro: Multi-hop relationship queries that would require 5+ JOINs in SQL are simple graph traversals
- Pro: Natural fit for ownership chains, investor networks, and geographic clustering analysis
- Pro: Knowledge graph can power AI agents that reason about relationships (similar to Cherre's approach)
- Pro: Flexible -- new relationship types can be added without schema migrations
- Pro: Enables visual network exploration UIs (investor maps, portfolio topology)
- Con: Dual-store complexity -- relational + graph must be kept in sync
- Con: Graph databases have different operational characteristics (backup, monitoring, scaling)
- Con: Developers need to learn graph query languages (Cypher, PGQL, or Apache AGE's openCypher)
- Con: Transactional writes should go through the relational store; graph is eventually consistent
- Con: Overhead is not justified if the platform does not need relationship traversal features

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| NCREIF PREA Reporting Standards (2025) | Relational tables store NCREIF PREA-compliant asset and fund metrics; graph enables portfolio-level aggregation across fund hierarchies |
| IBPDI Common Data Model | Graph node types aligned with IBPDI entity clusters (Building, Area, Cost) |
| ISO 3166-1/2 | Geographic nodes use ISO country/subdivision codes for jurisdiction modelling |
| ISO 4217 | Currency codes on monetary fields and graph edge properties |
| RESO Data Dictionary | Property node attributes follow RESO naming conventions |
| SEC Regulation D | Investor-to-fund edges carry accreditation status for compliance traversal queries |
| GIPS (CFA Institute) | Fund performance metrics stored relationally for GIPS-compliant composite reporting |

---

## Relational Core (PostgreSQL)

### Identity & Multi-Tenancy

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    org_type        VARCHAR(50) NOT NULL,
    default_currency VARCHAR(3) NOT NULL DEFAULT 'USD',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           VARCHAR(320) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);
```

### Properties, Deals, Financing

```sql
CREATE TABLE properties (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    name                VARCHAR(255) NOT NULL,
    property_type       VARCHAR(50) NOT NULL,
    status              VARCHAR(50) NOT NULL DEFAULT 'prospect',
    city                VARCHAR(100),
    state_or_province   VARCHAR(100),
    country_code        CHAR(2) NOT NULL DEFAULT 'US',
    postal_code         VARCHAR(20),
    submarket           VARCHAR(200),
    latitude            DECIMAL(10, 7),
    longitude           DECIMAL(10, 7),
    year_built          INTEGER,
    total_units         INTEGER,
    rentable_sqft       DECIMAL(12, 2),
    attributes          JSONB NOT NULL DEFAULT '{}',  -- asset-type-specific
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_properties_org ON properties (organization_id);
CREATE INDEX idx_properties_location ON properties (state_or_province, city);
CREATE INDEX idx_properties_submarket ON properties (submarket);

CREATE TABLE deals (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    property_id         UUID NOT NULL REFERENCES properties(id),
    deal_name           VARCHAR(255) NOT NULL,
    deal_type           VARCHAR(50) NOT NULL,
    status              VARCHAR(50) NOT NULL DEFAULT 'draft',
    purchase_price      DECIMAL(16, 2),
    total_project_cost  DECIMAL(16, 2),
    hold_period_months  INTEGER NOT NULL DEFAULT 60,
    exit_cap_rate       DECIMAL(6, 4),
    discount_rate       DECIMAL(6, 4),
    currency_code       CHAR(3) NOT NULL DEFAULT 'USD',
    assumptions         JSONB NOT NULL DEFAULT '{}',
    ai_metadata         JSONB NOT NULL DEFAULT '{}',
    assigned_analyst_id UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deals_org ON deals (organization_id);
CREATE INDEX idx_deals_property ON deals (property_id);
CREATE INDEX idx_deals_status ON deals (status);

CREATE TABLE deal_financing (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id         UUID NOT NULL REFERENCES deals(id) ON DELETE CASCADE,
    loan_name       VARCHAR(255) NOT NULL,
    loan_type       VARCHAR(50) NOT NULL,
    loan_amount     DECIMAL(16, 2) NOT NULL,
    interest_rate   DECIMAL(8, 6) NOT NULL,
    loan_term_months INTEGER NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE deal_scenarios (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id         UUID NOT NULL REFERENCES deals(id) ON DELETE CASCADE,
    scenario_name   VARCHAR(100) NOT NULL,
    scenario_type   VARCHAR(20) NOT NULL DEFAULT 'custom',
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    assumptions     JSONB NOT NULL DEFAULT '{}',
    results         JSONB NOT NULL DEFAULT '{}',
    /*
    results example:
    {
        "irr": 0.142,
        "npv": 1850000,
        "equity_multiple": 1.82,
        "cash_on_cash_y1": 0.068,
        "dscr_y1": 1.35
    }
    */
    cash_flows      JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scenarios_deal ON deal_scenarios (deal_id);
```

### Investors, Funds, Syndication

```sql
CREATE TABLE investors (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    investor_type       VARCHAR(30) NOT NULL,
    legal_name          VARCHAR(500) NOT NULL,
    email               VARCHAR(320),
    accreditation_status VARCHAR(30),
    accreditation_expiry DATE,
    kyc_status          VARCHAR(20) DEFAULT 'pending',
    details             JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_investors_org ON investors (organization_id);

CREATE TABLE funds (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    fund_name           VARCHAR(255) NOT NULL,
    fund_type           VARCHAR(50) NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'raising',
    total_raise_target  DECIMAL(16, 2),
    waterfall_structure JSONB NOT NULL DEFAULT '[]',
    fund_details        JSONB NOT NULL DEFAULT '{}',
    -- NCREIF PREA metrics
    gross_irr           DECIMAL(8, 4),
    net_irr             DECIMAL(8, 4),
    tvpi                DECIMAL(8, 4),
    dpi                 DECIMAL(8, 4),
    rvpi                DECIMAL(8, 4),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_funds_org ON funds (organization_id);

CREATE TABLE fund_properties (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fund_id         UUID NOT NULL REFERENCES funds(id) ON DELETE CASCADE,
    property_id     UUID NOT NULL REFERENCES properties(id),
    ownership_pct   DECIMAL(8, 6) NOT NULL DEFAULT 1.0,
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (fund_id, property_id)
);

CREATE TABLE investor_commitments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fund_id         UUID NOT NULL REFERENCES funds(id),
    investor_id     UUID NOT NULL REFERENCES investors(id),
    commitment_amount DECIMAL(16, 2) NOT NULL,
    paid_in_capital DECIMAL(16, 2) NOT NULL DEFAULT 0,
    ownership_pct   DECIMAL(8, 6),
    investor_class  VARCHAR(30) DEFAULT 'class_a',
    status          VARCHAR(20) NOT NULL DEFAULT 'committed',
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (fund_id, investor_id)
);

CREATE TABLE distributions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fund_id         UUID NOT NULL REFERENCES funds(id),
    investor_id     UUID NOT NULL REFERENCES investors(id),
    distribution_date DATE NOT NULL,
    distribution_type VARCHAR(30) NOT NULL,
    gross_amount    DECIMAL(14, 2) NOT NULL,
    net_amount      DECIMAL(14, 2) NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_distributions_fund ON distributions (fund_id);
CREATE INDEX idx_distributions_investor ON distributions (investor_id);
```

### Supporting Tables

```sql
CREATE TABLE rent_rolls (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
    as_of_date      DATE NOT NULL,
    summary         JSONB NOT NULL DEFAULT '{}',
    units           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE operating_statements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
    statement_type  VARCHAR(20) NOT NULL,
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    total_revenue   DECIMAL(14, 2),
    total_expenses  DECIMAL(14, 2),
    net_operating_income DECIMAL(14, 2),
    line_items      JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE comparables (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id),
    comp_type       VARCHAR(20) NOT NULL,
    comp_data       JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE market_data (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    submarket       VARCHAR(200) NOT NULL,
    property_type   VARCHAR(50) NOT NULL,
    data_date       DATE NOT NULL,
    data_source     VARCHAR(100) NOT NULL,
    metrics         JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_market_data_lookup ON market_data (submarket, property_type, data_date);

CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    document_type   VARCHAR(50) NOT NULL,
    file_name       VARCHAR(500) NOT NULL,
    file_url        VARCHAR(1000) NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    report_type     VARCHAR(50) NOT NULL,
    report_name     VARCHAR(255) NOT NULL,
    fund_id         UUID REFERENCES funds(id),
    deal_id         UUID REFERENCES deals(id),
    file_url        VARCHAR(1000),
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    user_id         UUID REFERENCES users(id),
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(30) NOT NULL,
    changes         JSONB,
    request_context JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_log_org_time ON audit_log (organization_id, created_at);
```

---

## Graph Layer

The graph layer can be implemented in two ways:

### Option A: PostgreSQL-native graph tables (simpler deployment)

```sql
-- Generic graph node table
-- Every relational entity that participates in the graph gets a corresponding node
CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY,             -- same UUID as the relational entity
    node_type       VARCHAR(50) NOT NULL,          -- 'Property', 'Deal', 'Fund', 'Investor', 'Submarket', 'Lender', 'Analyst'
    organization_id UUID NOT NULL,
    label           VARCHAR(255) NOT NULL,         -- display name
    properties      JSONB NOT NULL DEFAULT '{}',   -- node attributes for graph queries
    /*
    Property node example:
    {
        "property_type": "multifamily",
        "city": "Austin",
        "state": "TX",
        "submarket": "Austin CBD",
        "total_units": 200,
        "latest_valuation": 15000000,
        "status": "owned"
    }

    Investor node example:
    {
        "investor_type": "individual",
        "accreditation_status": "accredited",
        "total_committed": 2500000,
        "fund_count": 3
    }

    Submarket node example:
    {
        "avg_cap_rate": 0.055,
        "avg_vacancy": 0.06,
        "rent_growth_yoy": 0.034,
        "population_growth": 0.018
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_nodes_type ON graph_nodes (node_type);
CREATE INDEX idx_graph_nodes_org ON graph_nodes (organization_id);
CREATE INDEX idx_graph_nodes_properties ON graph_nodes USING gin (properties);

-- Generic graph edge table
CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id       UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_id       UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type       VARCHAR(50) NOT NULL,
    organization_id UUID NOT NULL,
    properties      JSONB NOT NULL DEFAULT '{}',
    weight          DECIMAL(10, 6) DEFAULT 1.0,    -- for weighted graph algorithms
    valid_from      TIMESTAMPTZ,                    -- temporal edges
    valid_to        TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_edges_source ON graph_edges (source_id);
CREATE INDEX idx_graph_edges_target ON graph_edges (target_id);
CREATE INDEX idx_graph_edges_type ON graph_edges (edge_type);
CREATE INDEX idx_graph_edges_org ON graph_edges (organization_id);
CREATE INDEX idx_graph_edges_temporal ON graph_edges (valid_from, valid_to);
```

### Edge Type Catalogue

```sql
-- Document all edge types and their semantics
CREATE TABLE graph_edge_types (
    edge_type       VARCHAR(50) PRIMARY KEY,
    source_type     VARCHAR(50) NOT NULL,
    target_type     VARCHAR(50) NOT NULL,
    description     TEXT NOT NULL,
    is_directional  BOOLEAN NOT NULL DEFAULT true,
    property_schema JSONB                          -- JSON Schema for edge properties
);

INSERT INTO graph_edge_types (edge_type, source_type, target_type, description, is_directional) VALUES
-- Investment relationships
('INVESTED_IN',       'Investor',  'Fund',       'Investor has committed capital to a fund', true),
('OWNS',              'Fund',      'Property',   'Fund holds ownership stake in a property', true),
('ANALYSED_BY',       'Property',  'Deal',       'Property is being analysed in a deal', true),
('ASSIGNED_TO',       'Deal',      'Analyst',    'Deal is assigned to an analyst for underwriting', true),
('FINANCED_BY',       'Deal',      'Lender',     'Deal has financing from a lender', true),

-- Geographic relationships
('LOCATED_IN',        'Property',  'Submarket',  'Property is located in a submarket', true),
('ADJACENT_TO',       'Submarket', 'Submarket',  'Submarkets are geographically adjacent', false),

-- Comparable relationships
('COMPARABLE_TO',     'Property',  'Property',   'Properties are comparable for valuation', false),
('SAME_SUBMARKET_AS', 'Property',  'Property',   'Properties share the same submarket', false),

-- Investor relationships
('CO_INVESTED_WITH',  'Investor',  'Investor',   'Investors have co-invested in the same fund', false),
('GUARANTOR_OF',      'Investor',  'Deal',       'Investor is a personal guarantor on a deal', true),
('RELATED_TO',        'Investor',  'Investor',   'Investors have a declared relationship (spouse, business partner)', false),

-- Fund relationships
('MANAGED_BY',        'Fund',      'Organization','Fund is managed by an organization', true),
('CO_GP_WITH',        'Organization','Organization','Organizations co-manage a fund as joint GPs', false),

-- Market data relationships
('DATA_SOURCE_FOR',   'MarketData','Submarket',  'Market data record provides metrics for submarket', true);
```

### Example: Populating the Graph

```sql
-- When a new investor commits to a fund, create/update graph nodes and edges

-- 1. Ensure investor node exists
INSERT INTO graph_nodes (id, node_type, organization_id, label, properties)
VALUES (
    'inv-uuid-001', 'Investor', 'org-uuid-001', 'John Smith',
    '{"investor_type": "individual", "accreditation_status": "accredited"}'
)
ON CONFLICT (id) DO UPDATE SET
    properties = EXCLUDED.properties,
    updated_at = now();

-- 2. Ensure fund node exists
INSERT INTO graph_nodes (id, node_type, organization_id, label, properties)
VALUES (
    'fund-uuid-001', 'Fund', 'org-uuid-001', 'Oakwood Multifamily Fund I',
    '{"fund_type": "single_asset", "status": "raising", "total_raise": 5000000}'
)
ON CONFLICT (id) DO UPDATE SET
    properties = EXCLUDED.properties,
    updated_at = now();

-- 3. Create INVESTED_IN edge
INSERT INTO graph_edges (source_id, target_id, edge_type, organization_id, properties, valid_from)
VALUES (
    'inv-uuid-001', 'fund-uuid-001', 'INVESTED_IN', 'org-uuid-001',
    '{"commitment_amount": 250000, "investor_class": "class_a", "ownership_pct": 0.05}',
    now()
);

-- 4. Find all co-investors and create CO_INVESTED_WITH edges
INSERT INTO graph_edges (source_id, target_id, edge_type, organization_id, properties)
SELECT
    'inv-uuid-001',
    e.source_id,
    'CO_INVESTED_WITH',
    'org-uuid-001',
    jsonb_build_object('shared_fund', 'fund-uuid-001')
FROM graph_edges e
WHERE e.target_id = 'fund-uuid-001'
  AND e.edge_type = 'INVESTED_IN'
  AND e.source_id != 'inv-uuid-001'
ON CONFLICT DO NOTHING;
```

---

## Graph Query Examples (PostgreSQL Recursive CTEs)

### Find all properties an investor has exposure to (through any fund)

```sql
-- Investor -> INVESTED_IN -> Fund -> OWNS -> Property
SELECT
    p.id AS property_id,
    p.label AS property_name,
    p.properties->>'city' AS city,
    p.properties->>'property_type' AS property_type,
    f.label AS fund_name,
    (e1.properties->>'commitment_amount')::numeric AS commitment,
    (e2.properties->>'ownership_pct')::numeric AS fund_ownership_in_property
FROM graph_edges e1
JOIN graph_nodes f ON e1.target_id = f.id AND f.node_type = 'Fund'
JOIN graph_edges e2 ON e2.source_id = f.id AND e2.edge_type = 'OWNS'
JOIN graph_nodes p ON e2.target_id = p.id AND p.node_type = 'Property'
WHERE e1.source_id = $1                    -- investor UUID
  AND e1.edge_type = 'INVESTED_IN'
ORDER BY commitment DESC;
```

### Find all investors with exposure to a specific submarket

```sql
-- Investor -> INVESTED_IN -> Fund -> OWNS -> Property -> LOCATED_IN -> Submarket
SELECT DISTINCT
    inv.id AS investor_id,
    inv.label AS investor_name,
    (e_invest.properties->>'commitment_amount')::numeric AS total_exposure
FROM graph_nodes sm
JOIN graph_edges e_loc ON e_loc.target_id = sm.id AND e_loc.edge_type = 'LOCATED_IN'
JOIN graph_nodes prop ON e_loc.source_id = prop.id
JOIN graph_edges e_own ON e_own.target_id = prop.id AND e_own.edge_type = 'OWNS'
JOIN graph_nodes fund ON e_own.source_id = fund.id
JOIN graph_edges e_invest ON e_invest.target_id = fund.id AND e_invest.edge_type = 'INVESTED_IN'
JOIN graph_nodes inv ON e_invest.source_id = inv.id
WHERE sm.label = 'Austin CBD'
  AND sm.node_type = 'Submarket'
ORDER BY total_exposure DESC;
```

### Detect potential conflicts of interest (investor in competing deals)

```sql
-- Find investors who are invested in funds that own properties in the same submarket
WITH investor_submarkets AS (
    SELECT
        inv.id AS investor_id,
        inv.label AS investor_name,
        sm.label AS submarket,
        fund.label AS fund_name,
        prop.label AS property_name
    FROM graph_edges e1
    JOIN graph_nodes inv ON e1.source_id = inv.id AND inv.node_type = 'Investor'
    JOIN graph_nodes fund ON e1.target_id = fund.id AND fund.node_type = 'Fund'
    JOIN graph_edges e2 ON e2.source_id = fund.id AND e2.edge_type = 'OWNS'
    JOIN graph_nodes prop ON e2.target_id = prop.id
    JOIN graph_edges e3 ON e3.source_id = prop.id AND e3.edge_type = 'LOCATED_IN'
    JOIN graph_nodes sm ON e3.target_id = sm.id AND sm.node_type = 'Submarket'
    WHERE e1.edge_type = 'INVESTED_IN'
      AND e1.organization_id = $1
)
SELECT
    investor_id, investor_name, submarket,
    array_agg(DISTINCT fund_name) AS funds,
    array_agg(DISTINCT property_name) AS properties,
    count(DISTINCT fund_name) AS fund_count
FROM investor_submarkets
GROUP BY investor_id, investor_name, submarket
HAVING count(DISTINCT fund_name) > 1
ORDER BY fund_count DESC;
```

### Geographic concentration risk analysis

```sql
-- What percentage of portfolio value is concentrated in each submarket?
SELECT
    sm.label AS submarket,
    sm.properties->>'avg_cap_rate' AS market_cap_rate,
    COUNT(DISTINCT prop.id) AS property_count,
    SUM((prop.properties->>'latest_valuation')::numeric) AS total_value,
    ROUND(
        SUM((prop.properties->>'latest_valuation')::numeric) * 100.0 /
        NULLIF(SUM(SUM((prop.properties->>'latest_valuation')::numeric)) OVER (), 0),
        2
    ) AS pct_of_portfolio
FROM graph_edges e
JOIN graph_nodes prop ON e.source_id = prop.id AND prop.node_type = 'Property'
JOIN graph_nodes sm ON e.target_id = sm.id AND sm.node_type = 'Submarket'
WHERE e.edge_type = 'LOCATED_IN'
  AND e.organization_id = $1
GROUP BY sm.id, sm.label, sm.properties->>'avg_cap_rate'
ORDER BY total_value DESC;
```

### Find similar properties for AI-powered comp analysis (graph path)

```sql
-- Using recursive CTE to find properties within N hops via COMPARABLE_TO edges
WITH RECURSIVE comp_chain AS (
    -- Start from the target property
    SELECT
        e.target_id AS property_id,
        1 AS depth,
        ARRAY[e.source_id, e.target_id] AS path
    FROM graph_edges e
    WHERE e.source_id = $1                 -- starting property UUID
      AND e.edge_type = 'COMPARABLE_TO'

    UNION ALL

    -- Traverse further
    SELECT
        e.target_id,
        cc.depth + 1,
        cc.path || e.target_id
    FROM comp_chain cc
    JOIN graph_edges e ON e.source_id = cc.property_id AND e.edge_type = 'COMPARABLE_TO'
    WHERE cc.depth < 3                     -- max 3 hops
      AND NOT e.target_id = ANY(cc.path)   -- prevent cycles
)
SELECT DISTINCT
    p.id,
    p.label AS property_name,
    p.properties->>'city' AS city,
    p.properties->>'property_type' AS type,
    (p.properties->>'latest_valuation')::numeric AS valuation,
    cc.depth AS similarity_distance
FROM comp_chain cc
JOIN graph_nodes p ON cc.property_id = p.id
ORDER BY cc.depth, valuation DESC;
```

---

### Option B: Apache AGE Extension (Cypher queries in PostgreSQL)

If the PostgreSQL Apache AGE extension is available, the same graph can be queried using Cypher syntax directly within PostgreSQL:

```sql
-- Using Apache AGE (PostgreSQL extension for property graph queries)
-- Equivalent of the "investor exposure" query above

SELECT * FROM cypher('investment_graph', $$
    MATCH (inv:Investor {id: 'inv-uuid-001'})
          -[:INVESTED_IN]->(fund:Fund)
          -[:OWNS]->(prop:Property)
          -[:LOCATED_IN]->(sm:Submarket)
    RETURN prop.name AS property,
           fund.name AS fund,
           sm.name AS submarket,
           inv.commitment_amount AS exposure
    ORDER BY inv.commitment_amount DESC
$$) AS (property TEXT, fund TEXT, submarket TEXT, exposure NUMERIC);
```

---

## Graph Synchronisation Strategy

```sql
-- Trigger function to sync relational changes to graph nodes
CREATE OR REPLACE FUNCTION sync_property_to_graph()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO graph_nodes (id, node_type, organization_id, label, properties)
    VALUES (
        NEW.id,
        'Property',
        NEW.organization_id,
        NEW.name,
        jsonb_build_object(
            'property_type', NEW.property_type,
            'city', NEW.city,
            'state', NEW.state_or_province,
            'submarket', NEW.submarket,
            'total_units', NEW.total_units,
            'rentable_sqft', NEW.rentable_sqft,
            'status', NEW.status
        )
    )
    ON CONFLICT (id) DO UPDATE SET
        label = EXCLUDED.label,
        properties = EXCLUDED.properties,
        updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_property_graph
AFTER INSERT OR UPDATE ON properties
FOR EACH ROW EXECUTE FUNCTION sync_property_to_graph();

-- Similar triggers for investors, funds, deals, etc.
-- In production, consider async sync via a change data capture (CDC) pipeline
-- for better performance and decoupling.
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Identity & Multi-Tenancy | 2 | organizations, users |
| Properties & Market | 4 | properties, comparables, market_data, rent_rolls |
| Deals & Underwriting | 3 | deals, deal_financing, deal_scenarios |
| Operating History | 1 | operating_statements |
| Investors & Syndication | 4 | investors, funds, investor_commitments, distributions |
| Fund-Property Junction | 1 | fund_properties |
| Documents & Reports | 2 | documents, reports |
| Audit | 1 | audit_log |
| Graph Layer | 3 | graph_nodes, graph_edges, graph_edge_types |
| **Total** | **21** | Plus 3 graph tables that mirror relational entities |

---

## Key Design Decisions

1. **Relational for writes, graph for reads.** All transactional operations (creating deals, recording distributions, importing rent rolls) go through the relational tables with full ACID guarantees. The graph layer is synchronised from the relational store and serves relationship-heavy read queries. This avoids the complexity of graph transactions while getting the query benefits.

2. **PostgreSQL-native graph tables over external graph database.** Using `graph_nodes` and `graph_edges` tables within PostgreSQL keeps the deployment stack simple (one database) and enables transactional consistency via triggers. For larger scale, Apache AGE adds Cypher query support without leaving PostgreSQL.

3. **Temporal edges with `valid_from` / `valid_to`.** Investment relationships change over time (an investor exits a fund, a fund sells a property). Temporal edges preserve historical relationships without deleting data, enabling "as-of-date" graph queries.

4. **Submarket as a first-class graph node.** Rather than treating submarket as just a string attribute on properties, it becomes a graph node that can have its own properties (market metrics) and edges (ADJACENT_TO other submarkets). This enables geographic clustering and concentration analysis.

5. **CO_INVESTED_WITH edges are derived.** When an investor commits to a fund, the system automatically creates CO_INVESTED_WITH edges between all investors in that fund. This precomputes the "co-investor network" so traversal queries are fast.

6. **Graph edge properties carry financial context.** The INVESTED_IN edge carries `commitment_amount`, `ownership_pct`, and `investor_class`. This means graph traversal queries can answer "what is this investor's total dollar exposure?" without joining back to relational tables.

7. **Trigger-based synchronisation for simplicity.** PostgreSQL triggers keep the graph layer in sync with relational changes. In production, this could be replaced with CDC (Change Data Capture) via Debezium or similar for better performance and decoupling, especially if the graph is in a separate database.

8. **Graph enables AI agent reasoning.** An AI deal screening agent can traverse the graph to find: "Properties in submarkets where the fund already has exposure, offered by sellers the fund has transacted with before, and priced below the submarket average." This kind of multi-relationship query is natural in a graph but requires complex SQL in a purely relational model.

9. **Conflict-of-interest detection is a graph problem.** Identifying investors who have competing interests across multiple funds in the same submarket is a classic graph pattern-matching problem. The graph layer makes this a simple query rather than a multi-JOIN relational nightmare.

10. **The graph layer is optional.** The relational core is fully functional without the graph tables. The graph layer adds analytical power but is not required for core CRUD operations. This allows phased implementation: build the relational MVP first, add the graph layer when relationship analytics become a priority.
