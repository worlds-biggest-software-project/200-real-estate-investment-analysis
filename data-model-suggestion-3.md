# Data Model Suggestion 3: Event-Sourced / Audit-First (CQRS)

> Project: Real Estate Investment Analysis · Created: 2026-05-20

## Philosophy

This model treats the event store as the single source of truth. Every change to every entity -- a deal assumption updated, a distribution posted, a lease renewed, a valuation revised -- is recorded as an immutable event in an append-only log. The current state of any entity is derived by replaying its event stream. Materialised read models (projections) are built from the event stream to serve dashboards, reports, and API queries efficiently.

This architecture is directly inspired by financial ledger systems where "the journal is the truth." In real estate investment analysis, the audit trail is not a secondary concern -- institutional LPs, fund auditors, and regulators increasingly require the ability to answer questions like "What assumptions were in the model when the IC approved this deal?" or "What was the fund NAV as of December 31 before the year-end adjustment?" An event-sourced model answers these questions natively because the full history is preserved, not overwritten.

The CQRS (Command Query Responsibility Segregation) pattern separates write operations (commands that generate events) from read operations (queries against materialised projections). This enables independent scaling: the write side handles deal updates and distribution calculations, while the read side serves LP portal dashboards and portfolio analytics from denormalized views optimised for each use case.

**Best for:** Platforms where full audit history, temporal queries ("what was true on date X?"), regulatory compliance, and AI-driven analytics on change patterns are core requirements -- particularly institutional fund management with LP reporting obligations.

**Trade-offs:**
- Pro: Complete, immutable audit trail for every data change -- built in, not bolted on
- Pro: Temporal queries are native -- reconstruct any entity state at any past point in time
- Pro: Read models can be optimised independently for different consumers (LP portal, analyst dashboard, API)
- Pro: Event replay enables powerful analytics (assumption drift, decision patterns, change frequency)
- Pro: Natural fit for AI training -- event streams provide rich labeled data for pattern recognition
- Con: Higher complexity -- developers must think in terms of events and projections, not CRUD
- Con: Event store grows continuously -- requires archival and compaction strategies
- Con: Eventual consistency between write side and read projections introduces latency
- Con: Schema evolution for events requires careful versioning (upcasters/downcasters)
- Con: More infrastructure -- event store + projection store + potentially a message bus

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| NCREIF PREA Reporting Standards (2025) | Projections materialise NCREIF PREA-compliant asset-level and fund-level reporting fields from event streams |
| GIPS (CFA Institute) | Fund performance projections produce GIPS-compliant composite return snapshots at any historical date |
| OCSF (Open Cybersecurity Schema Framework) | Event schema structure inspired by OCSF event categorisation for audit logs |
| ISO 4217 | Currency codes embedded in every monetary event |
| ISO 8601 | All event timestamps in UTC ISO 8601 format |
| SEC Regulation D | Investor compliance events (accreditation, subscription, Form D filing) tracked as first-class events |
| RESO Data Dictionary | Property attribute events reference RESO field names for MLS data import provenance |

---

## Event Store (Source of Truth)

```sql
-- The immutable event log -- append-only, never updated, never deleted
CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                -- aggregate root ID (deal, property, fund, investor)
    stream_type     VARCHAR(50) NOT NULL,          -- 'Deal', 'Property', 'Fund', 'Investor', 'Portfolio'
    event_type      VARCHAR(100) NOT NULL,         -- 'DealCreated', 'AssumptionUpdated', 'DistributionPosted'
    event_version   INTEGER NOT NULL,              -- sequential within a stream (optimistic concurrency)
    event_data      JSONB NOT NULL,                -- the event payload
    metadata        JSONB NOT NULL DEFAULT '{}',   -- causation_id, correlation_id, user context
    schema_version  INTEGER NOT NULL DEFAULT 1,    -- for event schema evolution
    organization_id UUID NOT NULL,                 -- tenant partition key
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Optimistic concurrency: no two events for the same stream can share a version
    UNIQUE (stream_id, event_version)
);

-- Partition by organization for multi-tenant isolation and performance
-- (In production, consider time-based partitioning as well)
CREATE INDEX idx_events_stream ON events (stream_id, event_version);
CREATE INDEX idx_events_type ON events (event_type);
CREATE INDEX idx_events_org_time ON events (organization_id, created_at);
CREATE INDEX idx_events_stream_type ON events (stream_type, created_at);

-- Event type catalogue (for documentation and schema validation)
CREATE TABLE event_types (
    event_type      VARCHAR(100) PRIMARY KEY,
    stream_type     VARCHAR(50) NOT NULL,
    description     TEXT NOT NULL,
    schema_version  INTEGER NOT NULL DEFAULT 1,
    json_schema     JSONB,                        -- JSON Schema for validating event_data
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Example Events

```sql
-- Deal created
INSERT INTO events (stream_id, stream_type, event_type, event_version, event_data, metadata, organization_id)
VALUES (
    'deal-uuid-001', 'Deal', 'DealCreated', 1,
    '{
        "deal_name": "Oakwood Apartments Acquisition",
        "deal_type": "acquisition",
        "deal_strategy": "buy_and_hold",
        "property_id": "prop-uuid-001",
        "purchase_price": 12500000,
        "currency_code": "USD",
        "assigned_analyst_id": "user-uuid-001"
    }',
    '{
        "user_id": "user-uuid-001",
        "correlation_id": "req-abc-123",
        "ip_address": "192.168.1.100",
        "source": "web_app"
    }',
    'org-uuid-001'
);

-- Assumption updated (with AI provenance)
INSERT INTO events (stream_id, stream_type, event_type, event_version, event_data, metadata, organization_id)
VALUES (
    'deal-uuid-001', 'Deal', 'AssumptionUpdated', 5,
    '{
        "field": "exit_cap_rate",
        "old_value": 0.060,
        "new_value": 0.055,
        "source": "ai_generated",
        "ai_confidence": 0.82,
        "ai_data_sources": ["ATTOM Austin CBD Q1 2026", "3 comparable sales within 1mi"],
        "ai_reasoning": "Trailing 12-month cap rate compression in Austin CBD multifamily suggests exit cap rate of 5.3-5.7%"
    }',
    '{
        "user_id": "user-uuid-002",
        "correlation_id": "req-def-456",
        "source": "ai_assistant"
    }',
    'org-uuid-001'
);

-- Distribution posted
INSERT INTO events (stream_id, stream_type, event_type, event_version, event_data, metadata, organization_id)
VALUES (
    'fund-uuid-001', 'Fund', 'DistributionPosted', 42,
    '{
        "distribution_date": "2026-03-31",
        "distribution_type": "cash_flow",
        "total_amount": 125000,
        "investor_distributions": [
            {
                "investor_id": "inv-uuid-001",
                "gross_amount": 50000,
                "net_amount": 50000,
                "waterfall_tier": "Preferred Return",
                "cumulative_irr": 0.092
            },
            {
                "investor_id": "inv-uuid-002",
                "gross_amount": 75000,
                "net_amount": 75000,
                "waterfall_tier": "Preferred Return",
                "cumulative_irr": 0.088
            }
        ]
    }',
    '{
        "user_id": "user-uuid-001",
        "correlation_id": "dist-batch-q1-2026",
        "source": "distribution_engine"
    }',
    'org-uuid-001'
);

-- Investor accreditation verified (compliance event)
INSERT INTO events (stream_id, stream_type, event_type, event_version, event_data, metadata, organization_id)
VALUES (
    'inv-uuid-001', 'Investor', 'AccreditationVerified', 3,
    '{
        "accreditation_status": "accredited",
        "accreditation_method": "income",
        "verification_date": "2026-01-15",
        "expiry_date": "2027-01-15",
        "verified_by": "VerifyInvestor.com",
        "document_ids": ["doc-uuid-001", "doc-uuid-002"]
    }',
    '{
        "user_id": "user-uuid-001",
        "source": "kyc_integration"
    }',
    'org-uuid-001'
);
```

### Complete Event Type Catalogue

```sql
INSERT INTO event_types (event_type, stream_type, description) VALUES
-- Property events
('PropertyCreated',           'Property', 'New property record created'),
('PropertyUpdated',           'Property', 'Property attributes modified'),
('PropertyValuationRecorded', 'Property', 'New appraisal or valuation recorded'),
('RentRollImported',         'Property', 'Rent roll data imported from CSV/Excel/AI extraction'),
('OperatingStatementImported','Property', 'T12 or annual operating statement imported'),
('LeaseRecorded',            'Property', 'Individual lease created or updated'),
('ComparableAdded',          'Property', 'Comparable transaction added for benchmarking'),

-- Deal events
('DealCreated',              'Deal', 'New deal created for underwriting'),
('DealStatusChanged',        'Deal', 'Deal moved to a new pipeline stage'),
('AssumptionUpdated',        'Deal', 'Individual underwriting assumption changed'),
('FinancingStructured',      'Deal', 'Debt or financing terms added or modified'),
('ScenarioCreated',          'Deal', 'New bear/base/bull/custom scenario added'),
('ScenarioAssumptionSet',    'Deal', 'Scenario-level assumption override applied'),
('CashFlowProjected',       'Deal', 'Cash flow projection computed for a scenario'),
('DealScoredByAI',          'Deal', 'AI deal screening score generated'),

-- Fund / syndication events
('FundCreated',              'Fund', 'New fund or syndication entity created'),
('FundStatusChanged',        'Fund', 'Fund lifecycle status change'),
('WaterfallDefined',         'Fund', 'GP/LP waterfall structure defined or modified'),
('InvestorCommitted',        'Fund', 'Investor commitment recorded'),
('CapitalCalled',            'Fund', 'Capital call issued to investors'),
('CapitalContributed',       'Fund', 'Capital contribution received from investor'),
('DistributionPosted',       'Fund', 'Cash distribution to investors'),
('PropertyAddedToFund',      'Fund', 'Property acquired into fund'),
('PropertyRemovedFromFund',  'Fund', 'Property disposed from fund'),
('NAVCalculated',            'Fund', 'Net Asset Value calculated for reporting period'),
('FormDFiled',               'Fund', 'SEC Form D filing recorded'),

-- Investor events
('InvestorCreated',          'Investor', 'New investor record created'),
('InvestorUpdated',          'Investor', 'Investor details modified'),
('AccreditationVerified',    'Investor', 'Investor accreditation status verified'),
('AccreditationExpired',     'Investor', 'Investor accreditation has expired'),
('KYCCompleted',             'Investor', 'KYC/AML verification completed'),
('DataConsentRecorded',      'Investor', 'GDPR/CCPA data consent recorded'),

-- Report events
('ReportGenerated',          'Report', 'Report created (manual or AI-generated)'),
('ReportShared',             'Report', 'Report shared with investors or stakeholders');
```

---

## Materialised Read Models (Projections)

These tables are built by projection handlers that consume the event stream. They can be rebuilt at any time by replaying events.

```sql
-- ============================================================
-- PROJECTION: Current state of deals (analyst dashboard)
-- Built from: DealCreated, AssumptionUpdated, DealStatusChanged,
--             FinancingStructured, ScenarioCreated, CashFlowProjected
-- ============================================================
CREATE TABLE v_deals (
    id                  UUID PRIMARY KEY,
    organization_id     UUID NOT NULL,
    property_id         UUID NOT NULL,
    deal_name           VARCHAR(255) NOT NULL,
    deal_type           VARCHAR(50) NOT NULL,
    deal_strategy       VARCHAR(50),
    status              VARCHAR(50) NOT NULL,
    purchase_price      DECIMAL(16, 2),
    total_project_cost  DECIMAL(16, 2),
    hold_period_months  INTEGER,
    exit_cap_rate       DECIMAL(6, 4),
    discount_rate       DECIMAL(6, 4),
    currency_code       CHAR(3) NOT NULL DEFAULT 'USD',

    -- Best-case scenario results (denormalized)
    base_case_irr       DECIMAL(8, 4),
    base_case_npv       DECIMAL(16, 2),
    base_case_equity_multiple DECIMAL(8, 4),
    base_case_cash_on_cash_y1 DECIMAL(8, 4),
    base_case_dscr_y1   DECIMAL(8, 4),

    -- Financing summary
    total_debt          DECIMAL(16, 2),
    blended_rate        DECIMAL(8, 6),
    ltv_ratio           DECIMAL(6, 4),

    -- AI metadata
    ai_assumption_count INTEGER DEFAULT 0,
    ai_confidence_avg   DECIMAL(4, 2),

    assigned_analyst_id UUID,
    last_event_version  INTEGER NOT NULL,
    last_modified_at    TIMESTAMPTZ NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_v_deals_org ON v_deals (organization_id);
CREATE INDEX idx_v_deals_status ON v_deals (status);

-- ============================================================
-- PROJECTION: Current state of properties
-- Built from: PropertyCreated, PropertyUpdated, PropertyValuationRecorded
-- ============================================================
CREATE TABLE v_properties (
    id                  UUID PRIMARY KEY,
    organization_id     UUID NOT NULL,
    name                VARCHAR(255) NOT NULL,
    property_type       VARCHAR(50) NOT NULL,
    status              VARCHAR(50) NOT NULL,
    city                VARCHAR(100),
    state_or_province   VARCHAR(100),
    country_code        CHAR(2) DEFAULT 'US',
    latitude            DECIMAL(10, 7),
    longitude           DECIMAL(10, 7),
    physical_attributes JSONB NOT NULL DEFAULT '{}',
    latest_valuation    DECIMAL(16, 2),
    latest_valuation_date DATE,
    latest_cap_rate     DECIMAL(6, 4),
    latest_noi          DECIMAL(14, 2),
    last_event_version  INTEGER NOT NULL,
    last_modified_at    TIMESTAMPTZ NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_v_properties_org ON v_properties (organization_id);
CREATE INDEX idx_v_properties_type ON v_properties (property_type);
CREATE INDEX idx_v_properties_location ON v_properties (state_or_province, city);

-- ============================================================
-- PROJECTION: Current state of funds (GP dashboard)
-- Built from: FundCreated, FundStatusChanged, WaterfallDefined,
--             InvestorCommitted, DistributionPosted, NAVCalculated
-- ============================================================
CREATE TABLE v_funds (
    id                  UUID PRIMARY KEY,
    organization_id     UUID NOT NULL,
    fund_name           VARCHAR(255) NOT NULL,
    fund_type           VARCHAR(50) NOT NULL,
    status              VARCHAR(30) NOT NULL,
    total_raise_target  DECIMAL(16, 2),
    total_committed     DECIMAL(16, 2),
    total_paid_in       DECIMAL(16, 2),
    total_distributed   DECIMAL(16, 2),
    investor_count      INTEGER,
    property_count      INTEGER,

    -- NCREIF PREA metrics
    gross_irr           DECIMAL(8, 4),
    net_irr             DECIMAL(8, 4),
    tvpi                DECIMAL(8, 4),
    dpi                 DECIMAL(8, 4),
    rvpi                DECIMAL(8, 4),
    latest_nav          DECIMAL(16, 2),
    latest_nav_date     DATE,

    -- Waterfall structure
    waterfall_structure JSONB NOT NULL DEFAULT '[]',

    -- Regulatory
    reg_d_filing_type   VARCHAR(10),
    form_d_filed_date   DATE,

    last_event_version  INTEGER NOT NULL,
    last_modified_at    TIMESTAMPTZ NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_v_funds_org ON v_funds (organization_id);
CREATE INDEX idx_v_funds_status ON v_funds (status);

-- ============================================================
-- PROJECTION: Investor portal view
-- Built from: InvestorCommitted, CapitalContributed, DistributionPosted
-- ============================================================
CREATE TABLE v_investor_positions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL,
    investor_id         UUID NOT NULL,
    fund_id             UUID NOT NULL,
    investor_name       VARCHAR(500) NOT NULL,
    investor_class      VARCHAR(30),
    commitment_amount   DECIMAL(16, 2) NOT NULL,
    paid_in_capital     DECIMAL(16, 2) NOT NULL DEFAULT 0,
    total_distributions DECIMAL(16, 2) NOT NULL DEFAULT 0,
    ownership_pct       DECIMAL(8, 6),
    net_irr             DECIMAL(8, 4),
    equity_multiple     DECIMAL(8, 4),
    accreditation_status VARCHAR(30),
    accreditation_expiry DATE,
    last_distribution_date DATE,
    last_event_version  INTEGER NOT NULL,
    last_modified_at    TIMESTAMPTZ NOT NULL,
    UNIQUE (investor_id, fund_id)
);

CREATE INDEX idx_v_investor_positions_investor ON v_investor_positions (investor_id);
CREATE INDEX idx_v_investor_positions_fund ON v_investor_positions (fund_id);
CREATE INDEX idx_v_investor_positions_org ON v_investor_positions (organization_id);

-- ============================================================
-- PROJECTION: Distribution history (LP quarterly reports)
-- Built from: DistributionPosted
-- ============================================================
CREATE TABLE v_distribution_history (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL,
    fund_id             UUID NOT NULL,
    investor_id         UUID NOT NULL,
    distribution_date   DATE NOT NULL,
    distribution_type   VARCHAR(30) NOT NULL,
    gross_amount        DECIMAL(14, 2) NOT NULL,
    net_amount          DECIMAL(14, 2) NOT NULL,
    waterfall_tier      VARCHAR(100),
    cumulative_irr      DECIMAL(8, 4),
    cumulative_equity_multiple DECIMAL(8, 4),
    source_event_id     UUID NOT NULL,          -- link back to event
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_dist_history_fund ON v_distribution_history (fund_id, distribution_date);
CREATE INDEX idx_v_dist_history_investor ON v_distribution_history (investor_id, distribution_date);

-- ============================================================
-- PROJECTION: Assumption change timeline (for AI analytics)
-- Built from: AssumptionUpdated
-- ============================================================
CREATE TABLE v_assumption_timeline (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL,
    deal_id             UUID NOT NULL,
    field_name          VARCHAR(100) NOT NULL,
    old_value           DECIMAL(14, 4),
    new_value           DECIMAL(14, 4),
    change_source       VARCHAR(50) NOT NULL,    -- 'manual', 'ai_generated', 'market_data'
    ai_confidence       DECIMAL(4, 2),
    ai_data_sources     TEXT[],
    changed_by_user_id  UUID,
    changed_at          TIMESTAMPTZ NOT NULL,
    source_event_id     UUID NOT NULL
);

CREATE INDEX idx_v_assumption_timeline_deal ON v_assumption_timeline (deal_id, changed_at);
CREATE INDEX idx_v_assumption_timeline_field ON v_assumption_timeline (field_name);

-- ============================================================
-- PROJECTION: Deal pipeline (Kanban board view)
-- Built from: DealCreated, DealStatusChanged
-- ============================================================
CREATE TABLE v_deal_pipeline (
    id                  UUID PRIMARY KEY,
    organization_id     UUID NOT NULL,
    deal_name           VARCHAR(255) NOT NULL,
    deal_type           VARCHAR(50) NOT NULL,
    status              VARCHAR(50) NOT NULL,
    property_name       VARCHAR(255),
    city                VARCHAR(100),
    state_or_province   VARCHAR(100),
    purchase_price      DECIMAL(16, 2),
    base_case_irr       DECIMAL(8, 4),
    assigned_analyst    VARCHAR(255),
    days_in_stage       INTEGER,
    status_changed_at   TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_v_pipeline_org_status ON v_deal_pipeline (organization_id, status);
```

---

## Snapshots (Performance Optimisation)

```sql
-- Periodic snapshots to avoid replaying long event streams
CREATE TABLE snapshots (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    event_version   INTEGER NOT NULL,              -- version at which snapshot was taken
    state           JSONB NOT NULL,                -- full aggregate state as JSON
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, event_version)
);

-- Snapshot policy: take a snapshot every 100 events or at significant milestones
-- Projection rebuilds start from the latest snapshot + remaining events
CREATE INDEX idx_snapshots_stream ON snapshots (stream_id, event_version DESC);
```

---

## Supporting Tables (Not Event-Sourced)

```sql
-- Reference data: not event-sourced because it's static configuration

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
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);

-- Market data (reference, not event-sourced)
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

-- Documents (file references, not event-sourced)
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

CREATE INDEX idx_documents_entity ON documents (entity_type, entity_id);

-- Projection checkpoints (tracks which event each projection has processed)
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Temporal Query Examples

### Reconstruct deal state at a specific date

```sql
-- What were the assumptions for Deal X when the IC approved it on 2026-04-15?
SELECT event_type, event_data, created_at
FROM events
WHERE stream_id = $1                          -- deal UUID
  AND stream_type = 'Deal'
  AND created_at <= '2026-04-15T23:59:59Z'
ORDER BY event_version ASC;

-- Application code replays these events to reconstruct the state
```

### Find all assumption changes for a deal (audit trail)

```sql
SELECT
    created_at AS changed_at,
    event_data->>'field' AS field_name,
    event_data->>'old_value' AS old_value,
    event_data->>'new_value' AS new_value,
    event_data->>'source' AS change_source,
    metadata->>'user_id' AS changed_by
FROM events
WHERE stream_id = $1
  AND event_type = 'AssumptionUpdated'
ORDER BY event_version ASC;
```

### Fund NAV history (from NAVCalculated events)

```sql
SELECT
    event_data->>'calculation_date' AS nav_date,
    (event_data->>'total_nav')::numeric AS nav,
    (event_data->>'nav_per_unit')::numeric AS nav_per_unit
FROM events
WHERE stream_id = $1
  AND event_type = 'NAVCalculated'
ORDER BY event_version ASC;
```

### AI assumption acceptance rate

```sql
SELECT
    event_data->>'field' AS assumption_field,
    COUNT(*) AS total_suggestions,
    COUNT(*) FILTER (WHERE (event_data->>'source') = 'ai_generated') AS ai_generated,
    AVG((event_data->>'ai_confidence')::numeric) AS avg_confidence
FROM events
WHERE stream_type = 'Deal'
  AND event_type = 'AssumptionUpdated'
  AND organization_id = $1
GROUP BY event_data->>'field'
ORDER BY total_suggestions DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | events, event_types |
| Snapshots | 1 | snapshots (performance optimisation) |
| Read Projections | 7 | v_deals, v_properties, v_funds, v_investor_positions, v_distribution_history, v_assumption_timeline, v_deal_pipeline |
| Supporting (Non-Event-Sourced) | 6 | organizations, users, market_data, documents, projection_checkpoints |
| **Total** | **16** | But the event store is the source of truth; projections are derived and rebuildable |

---

## Key Design Decisions

1. **Single event table, partitioned by organization.** All events for all aggregate types live in one table. This simplifies cross-aggregate queries (e.g., "all events in this org in the last 24 hours") and subscription patterns. The `stream_type` column enables filtering by aggregate type. In production, partition by `organization_id` and optionally by time range for large tenants.

2. **Optimistic concurrency via stream version.** The `UNIQUE (stream_id, event_version)` constraint prevents concurrent writes to the same aggregate from producing inconsistent state. If two analysts update the same deal simultaneously, one will get a version conflict and must retry against the latest state.

3. **Events carry full context, not just deltas.** The `AssumptionUpdated` event includes both `old_value` and `new_value`, plus the source (manual vs. AI), AI confidence, and data sources. This makes the event self-contained for audit without needing to replay previous events to understand context.

4. **Projections are prefixed with `v_` to signal they are views/materialised models.** They can be dropped and rebuilt from the event stream at any time. This convention makes it clear to developers that projections are derived, not authoritative.

5. **Snapshot table for long-lived aggregates.** A deal that has been underwritten, revised, and re-analysed dozens of times could accumulate hundreds of events. Periodic snapshots avoid replaying the full stream on every read. The snapshot stores the full aggregate state as JSONB.

6. **Projection checkpoints track replay position.** Each projection records which event it last processed, enabling incremental catch-up after downtime. This is the foundation for reliable eventual consistency.

7. **Not everything is event-sourced.** Organizations, users, market data, and documents are stored as regular CRUD tables. These are reference data or supporting infrastructure that don't benefit from full event history. The event store covers the business domain: deals, properties, funds, and investors.

8. **Event type catalogue with JSON Schema.** The `event_types` table documents every event type and optionally includes a JSON Schema for validating event payloads. This serves as living documentation and enables automated schema validation in the event pipeline.

9. **Multiple projections serve different consumers.** `v_deal_pipeline` is optimised for the Kanban board UI. `v_investor_positions` is optimised for the LP portal. `v_assumption_timeline` is optimised for AI analytics. Each projection denormalizes exactly what its consumer needs, avoiding one-size-fits-all compromises.

10. **Temporal queries are native, not simulated.** Unlike the relational model where answering "what was the exit cap rate on March 1?" requires an audit log JOIN, the event-sourced model simply replays events up to March 1 and reads the resulting state. This is the fundamental architectural advantage for institutional fund management.
