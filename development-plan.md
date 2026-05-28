# Development Plan: Real Estate Investment Analysis

> Candidate #200 · Plan Created: 2026-05-25

## Technology Decisions

| Category | Decision | Rationale |
|----------|----------|-----------|
| **Primary Language** | TypeScript (full-stack) | Type safety across API and UI; shared types between frontend and backend; strong ecosystem for financial SaaS; aligns with mid-market developer availability |
| **Backend Framework** | Node.js + Fastify | Lower overhead than Express; native TypeScript support; JSON Schema validation built in; plugin architecture suits modular domain modules (deals, funds, investors) |
| **Frontend Framework** | Next.js 15 (App Router) | SSR for SEO on public report pages; RSC for data-heavy dashboards; API routes co-located with pages; Vercel deployment path; strongest React meta-framework |
| **Database** | PostgreSQL 16 | JSONB for asset-type variability (Data Model Suggestion 2 hybrid approach); GIN indexes for flexible property queries; PostGIS for geo queries; mature audit/RLS capabilities; NCREIF PREA field alignment |
| **Data Model** | Hybrid Relational + JSONB (Suggestion 2) | Best fit for MVP targeting mid-market; typed columns for financial metrics, JSONB for asset-type-specific attributes; 22 tables vs 24 normalized; fastest iteration on new deal types without migrations |
| **ORM** | Drizzle ORM | Type-safe SQL; excellent PostgreSQL JSONB support; lightweight compared to Prisma; raw SQL escape hatch for complex financial queries |
| **Cache** | Redis 7 | Session management; market data caching; rate limiting for data provider API calls; pub/sub for real-time dashboard updates |
| **Object Storage** | S3-compatible (AWS S3 / MinIO) | Document storage for OMs, rent rolls, T12s, appraisals; MinIO for self-hosted deployments |
| **Authentication** | Auth.js (NextAuth v5) + OIDC | OAuth 2.0 / OpenID Connect for enterprise SSO; credential fallback for smaller operators; multi-tenant session management |
| **AI/LLM Integration** | Anthropic Claude API (primary) + OpenAI fallback | Assumption generation, document parsing, report narrative generation; Claude for structured output quality; provider-agnostic adapter pattern |
| **PDF Generation** | React-PDF + Puppeteer | React-PDF for investor reports with charts; Puppeteer for complex HTML-to-PDF conversion; consistent with React component model |
| **Data Providers** | ATTOM API + BatchData API | 155M+ US properties; affordable compared to CoStar; REST/JSON with good documentation; viable alternatives to paywalled institutional data |
| **Task Queue** | BullMQ (Redis-backed) | Background jobs for cash flow computation, AI assumption generation, report generation, data provider sync; retry logic; priority queues |
| **Testing** | Vitest + Playwright | Vitest for unit/integration (fast, native ESM); Playwright for E2E; financial calculation tests demand high precision assertions |
| **Monorepo** | Turborepo | Shared types package; independent build caching for API, web, and calculation engine; parallel task execution |
| **CI/CD** | GitHub Actions | Standard for open-source; matrix testing across Node versions; automated deployment to Vercel (web) and Railway/Fly.io (API) |
| **API Documentation** | OpenAPI 3.1 (auto-generated) | Standards requirement from research; enables SDK generation; Fastify's built-in schema produces OpenAPI specs |

---

## Project Directory Structure

```
real-estate-investment-analysis/
├── apps/
│   ├── web/                          # Next.js 15 frontend
│   │   ├── app/
│   │   │   ├── (auth)/               # Login, register, SSO flows
│   │   │   ├── (dashboard)/          # Authenticated app shell
│   │   │   │   ├── deals/            # Deal underwriting views
│   │   │   │   ├── properties/       # Property management
│   │   │   │   ├── portfolio/        # Portfolio dashboard
│   │   │   │   ├── funds/            # Fund & syndication management
│   │   │   │   ├── investors/        # Investor management
│   │   │   │   ├── reports/          # Report generation & history
│   │   │   │   └── settings/         # Org settings, users, billing
│   │   │   ├── (public)/             # Public report viewer
│   │   │   └── api/                  # Next.js API routes (auth, webhooks)
│   │   ├── components/
│   │   │   ├── deals/                # Deal-specific UI components
│   │   │   ├── charts/               # Financial charts (IRR waterfall, cash flow)
│   │   │   ├── tables/               # Data tables (rent roll, T12, scenarios)
│   │   │   ├── forms/                # Multi-step deal entry forms
│   │   │   ├── reports/              # Report template components
│   │   │   └── ui/                   # Shared UI primitives (shadcn/ui)
│   │   └── lib/
│   │       ├── hooks/                # React hooks
│   │       └── utils/                # Client utilities
│   └── api/                          # Fastify API server
│       ├── src/
│       │   ├── modules/
│       │   │   ├── auth/             # Authentication & authorization
│       │   │   ├── organizations/    # Multi-tenant org management
│       │   │   ├── properties/       # Property CRUD & search
│       │   │   ├── deals/            # Deal underwriting & scenarios
│       │   │   ├── cashflow/         # Cash flow projection engine
│       │   │   ├── funds/            # Fund & syndication management
│       │   │   ├── investors/        # Investor management & compliance
│       │   │   ├── distributions/    # Waterfall calculation & distributions
│       │   │   ├── reports/          # Report generation pipeline
│       │   │   ├── market-data/      # Data provider integration
│       │   │   ├── documents/        # Document upload & AI extraction
│       │   │   └── ai/              # AI assumption engine & NL deal entry
│       │   ├── plugins/              # Fastify plugins (auth, RLS, audit)
│       │   ├── middleware/           # Request middleware
│       │   └── utils/               # Server utilities
│       └── tests/
│           ├── unit/
│           ├── integration/
│           └── fixtures/
├── packages/
│   ├── types/                        # Shared TypeScript types & interfaces
│   ├── financial-engine/             # Core financial calculation library
│   │   ├── src/
│   │   │   ├── irr.ts
│   │   │   ├── npv.ts
│   │   │   ├── dcf.ts
│   │   │   ├── dscr.ts
│   │   │   ├── waterfall.ts
│   │   │   ├── amortization.ts
│   │   │   └── sensitivity.ts
│   │   └── tests/
│   ├── db/                           # Database schema, migrations, seeds
│   │   ├── schema/
│   │   ├── migrations/
│   │   └── seeds/
│   └── ai-client/                    # AI provider adapter (Claude, OpenAI)
│       ├── src/
│       │   ├── providers/
│       │   ├── prompts/
│       │   └── parsers/
│       └── tests/
├── docker/
│   ├── docker-compose.yml            # Local dev stack (Postgres, Redis, MinIO)
│   └── docker-compose.prod.yml       # Self-hosted production stack
├── docs/
│   ├── api/                          # Generated OpenAPI docs
│   └── architecture/                 # Architecture decision records
├── turbo.json
├── package.json
└── .github/
    └── workflows/
        ├── ci.yml
        └── deploy.yml
```

---

## Phase 1: Project Foundation & Database

**Objective:** Establish the monorepo, database schema, and development environment so all subsequent phases have a working foundation.

**Dependencies:** None (starting phase)

### Task 1.1: Monorepo Setup & Tooling

**What:** Initialize Turborepo monorepo with apps/web, apps/api, and packages/types, packages/financial-engine, packages/db. Configure TypeScript, ESLint, Prettier, and shared tsconfig.

**Design:**

```typescript
// turbo.json
{
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**", ".next/**"] },
    "dev": { "cache": false, "persistent": true },
    "test": { "dependsOn": ["^build"] },
    "lint": {},
    "typecheck": { "dependsOn": ["^build"] }
  }
}

// packages/types/src/index.ts — shared type exports
export type { Organization, User, UserRole } from './identity';
export type { Property, PropertyType, PropertyStatus } from './property';
export type { Deal, DealType, DealStrategy, DealStatus } from './deal';
export type { DealScenario, ScenarioType, CashFlowPeriod } from './scenario';
export type { Fund, FundType, Investor, InvestorType } from './syndication';
export type { Report, ReportType, ReportFormat } from './report';
```

**Testing:**
- `test_monorepo_build` — `turbo build` completes without errors across all packages
- `test_typescript_strict` — `turbo typecheck` passes with strict mode enabled
- `test_lint_clean` — `turbo lint` reports zero errors
- `test_dev_starts` — `turbo dev` starts web and api concurrently

### Task 1.2: Database Schema & Migrations

**What:** Implement the Hybrid Relational + JSONB schema (Data Model Suggestion 2) using Drizzle ORM migrations. Create all 22 tables with indexes and constraints.

**Design:**

```typescript
// packages/db/schema/properties.ts
import { pgTable, uuid, varchar, decimal, jsonb, timestamp, index } from 'drizzle-orm/pg-core';
import { organizations } from './identity';

export const properties = pgTable('properties', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id),
  name: varchar('name', { length: 255 }).notNull(),
  propertyType: varchar('property_type', { length: 50 }).notNull(),
  status: varchar('status', { length: 50 }).notNull().default('prospect'),
  city: varchar('city', { length: 100 }),
  stateOrProvince: varchar('state_or_province', { length: 100 }),
  countryCode: varchar('country_code', { length: 2 }).notNull().default('US'),
  postalCode: varchar('postal_code', { length: 20 }),
  latitude: decimal('latitude', { precision: 10, scale: 7 }),
  longitude: decimal('longitude', { precision: 10, scale: 7 }),
  addressDetails: jsonb('address_details').notNull().default({}),
  physicalAttributes: jsonb('physical_attributes').notNull().default({}),
  taxInfo: jsonb('tax_info').notNull().default({}),
  externalIds: jsonb('external_ids').notNull().default({}),
  zoning: jsonb('zoning').notNull().default({}),
  notes: varchar('notes'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  orgIdx: index('idx_properties_org').on(table.organizationId),
  typeIdx: index('idx_properties_type').on(table.propertyType),
  statusIdx: index('idx_properties_status').on(table.status),
  locationIdx: index('idx_properties_location').on(table.stateOrProvince, table.city),
}));

// packages/db/schema/deals.ts
export const deals = pgTable('deals', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id),
  propertyId: uuid('property_id').notNull().references(() => properties.id),
  dealName: varchar('deal_name', { length: 255 }).notNull(),
  dealType: varchar('deal_type', { length: 50 }).notNull(),
  dealStrategy: varchar('deal_strategy', { length: 50 }),
  status: varchar('status', { length: 50 }).notNull().default('draft'),
  currencyCode: varchar('currency_code', { length: 3 }).notNull().default('USD'),
  purchasePrice: decimal('purchase_price', { precision: 16, scale: 2 }),
  totalProjectCost: decimal('total_project_cost', { precision: 16, scale: 2 }),
  holdPeriodMonths: integer('hold_period_months').notNull().default(60),
  exitCapRate: decimal('exit_cap_rate', { precision: 6, scale: 4 }),
  discountRate: decimal('discount_rate', { precision: 6, scale: 4 }),
  acquisitionAssumptions: jsonb('acquisition_assumptions').notNull().default({}),
  exitAssumptions: jsonb('exit_assumptions').notNull().default({}),
  aiMetadata: jsonb('ai_metadata').notNull().default({}),
  assignedAnalystId: uuid('assigned_analyst_id').references(() => users.id),
  notes: varchar('notes'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  orgIdx: index('idx_deals_org').on(table.organizationId),
  propertyIdx: index('idx_deals_property').on(table.propertyId),
  statusIdx: index('idx_deals_status').on(table.status),
}));
```

**Testing:**
- `test_migration_up` — All migrations apply cleanly to a fresh database
- `test_migration_down` — All migrations roll back without errors
- `test_migration_idempotent` — Running migrations twice produces no changes
- `test_rls_tenant_isolation` — Row-level security prevents cross-tenant data access
- `test_schema_22_tables` — Exactly 22 application tables exist after migration

### Task 1.3: Docker Development Environment

**What:** Create docker-compose.yml for local development with PostgreSQL 16, Redis 7, and MinIO. Include seed data for a demo organization with sample properties.

**Design:**

```yaml
# docker/docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: reia_dev
      POSTGRES_USER: reia
      POSTGRES_PASSWORD: dev_password
    ports: ["5432:5432"]
    volumes:
      - pgdata:/var/lib/postgresql/data
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minio
      MINIO_ROOT_PASSWORD: minio_dev
    ports: ["9000:9000", "9001:9001"]
    volumes:
      - miniodata:/data
volumes:
  pgdata:
  miniodata:
```

**Testing:**
- `test_docker_compose_up` — All three services start and accept connections
- `test_postgres_connection` — Application connects to PostgreSQL with configured credentials
- `test_redis_connection` — Redis PING returns PONG
- `test_minio_bucket_creation` — Default buckets created on startup

---

## Phase 2: Authentication & Multi-Tenancy

**Objective:** Implement user authentication, organization management, and tenant isolation so all subsequent features operate within a secure multi-tenant context.

**Dependencies:** Phase 1

### Task 2.1: Authentication System

**What:** Implement Auth.js (NextAuth v5) with credential-based login, Google OAuth, and OpenID Connect for enterprise SSO. Include JWT session management with organization context.

**Design:**

```typescript
// apps/web/app/api/auth/[...nextauth]/route.ts
import NextAuth from 'next-auth';
import Credentials from 'next-auth/providers/credentials';
import Google from 'next-auth/providers/google';

export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [
    Credentials({
      credentials: { email: {}, password: {} },
      authorize: async (credentials) => {
        // Verify against users table, return user with organizationId
      },
    }),
    Google({ clientId: process.env.GOOGLE_CLIENT_ID!, clientSecret: process.env.GOOGLE_CLIENT_SECRET! }),
  ],
  callbacks: {
    jwt({ token, user }) {
      if (user) {
        token.organizationId = user.organizationId;
        token.role = user.role;
      }
      return token;
    },
    session({ session, token }) {
      session.user.organizationId = token.organizationId as string;
      session.user.role = token.role as string;
      return session;
    },
  },
});

// packages/types/src/identity.ts
export interface Organization {
  id: string;
  name: string;
  slug: string;
  orgType: 'fund_manager' | 'syndicator' | 'family_office' | 'individual' | 'brokerage';
  defaultCurrency: string;
  settings: OrganizationSettings;
}

export type UserRole = 'admin' | 'manager' | 'analyst' | 'viewer' | 'lp_viewer';

export interface User {
  id: string;
  organizationId: string;
  email: string;
  fullName: string;
  role: UserRole;
  isActive: boolean;
}
```

**Testing:**
- `test_credential_login_success` — Valid email/password returns JWT with org context
- `test_credential_login_invalid` — Wrong password returns 401
- `test_google_oauth_flow` — Google OAuth redirects and creates user on first login
- `test_session_contains_org_id` — Session token includes organizationId and role
- `test_inactive_user_blocked` — Deactivated user cannot authenticate

### Task 2.2: Organization & User Management

**What:** Build organization CRUD, user invitation flow, and role-based access control (RBAC). Admin can create org, invite users, assign roles.

**Design:**

```typescript
// apps/api/src/modules/organizations/organization.routes.ts
import { FastifyPluginAsync } from 'fastify';

const organizationRoutes: FastifyPluginAsync = async (fastify) => {
  // POST /api/organizations — create new org (self-service signup)
  fastify.post('/', { schema: createOrgSchema }, createOrganizationHandler);

  // GET /api/organizations/:slug — get org details (admin/manager)
  fastify.get('/:slug', { schema: getOrgSchema }, getOrganizationHandler);

  // PATCH /api/organizations/:slug — update org settings
  fastify.patch('/:slug', { schema: updateOrgSchema }, updateOrganizationHandler);

  // POST /api/organizations/:slug/users — invite user
  fastify.post('/:slug/users', { schema: inviteUserSchema }, inviteUserHandler);

  // PATCH /api/organizations/:slug/users/:userId — update role
  fastify.patch('/:slug/users/:userId', { schema: updateUserSchema }, updateUserHandler);
};

// Role-based permission matrix
export const PERMISSIONS: Record<UserRole, string[]> = {
  admin:    ['org:manage', 'user:manage', 'deal:*', 'property:*', 'fund:*', 'investor:*', 'report:*'],
  manager:  ['deal:*', 'property:*', 'fund:*', 'investor:*', 'report:*'],
  analyst:  ['deal:read', 'deal:write', 'property:read', 'property:write', 'report:read', 'report:write'],
  viewer:   ['deal:read', 'property:read', 'fund:read', 'report:read'],
  lp_viewer: ['fund:read:own', 'report:read:own'],
};
```

**Testing:**
- `test_create_organization` — POST creates org and assigns creator as admin
- `test_invite_user_sends_email` — Invitation generates magic link and queues email
- `test_role_assignment` — Admin can change user roles
- `test_rbac_analyst_cannot_manage_users` — Analyst role returns 403 on user management endpoints
- `test_rbac_lp_viewer_sees_own_fund_only` — LP viewer scoped to their fund positions
- `test_multi_tenant_isolation` — Org A user cannot access Org B data

### Task 2.3: Audit Logging

**What:** Implement audit log middleware that records all write operations (create, update, delete) with user context, entity references, and change diffs.

**Design:**

```typescript
// apps/api/src/plugins/audit.plugin.ts
import { FastifyPluginAsync } from 'fastify';

interface AuditEntry {
  organizationId: string;
  userId: string;
  entityType: string;    // 'deal', 'property', 'investor', 'fund'
  entityId: string;
  action: 'create' | 'update' | 'delete' | 'view' | 'export' | 'share';
  changes: Record<string, { old: unknown; new: unknown }> | null;
  requestContext: {
    ipAddress: string;
    userAgent: string;
    sessionId: string;
  };
}

export const auditPlugin: FastifyPluginAsync = async (fastify) => {
  fastify.addHook('onResponse', async (request, reply) => {
    if (['POST', 'PUT', 'PATCH', 'DELETE'].includes(request.method)) {
      const entry = request.auditEntry as AuditEntry | undefined;
      if (entry) {
        await fastify.db.insert(auditLog).values(entry);
      }
    }
  });
};
```

**Testing:**
- `test_audit_log_on_create` — Creating a deal generates an audit entry with action 'create'
- `test_audit_log_captures_diff` — Updating a deal records old/new values in changes JSONB
- `test_audit_log_records_user_context` — Audit entry includes IP, user agent, user ID
- `test_audit_log_read_operations_excluded` — GET requests do not generate audit entries by default
- `test_audit_log_queryable_by_entity` — Can retrieve full audit history for a given entity

---

## Phase 3: Property Management & Data Import

**Objective:** Build the property CRUD, document management, rent roll import, and T12 import capabilities that form the foundation for deal underwriting.

**Dependencies:** Phase 2

### Task 3.1: Property CRUD & Search

**What:** Implement property creation, editing, listing, and search with support for all property types. Include geo-based search and filtering by type, status, and submarket.

**Design:**

```typescript
// packages/types/src/property.ts
export type PropertyType =
  | 'multifamily' | 'single_family' | 'office' | 'retail' | 'industrial'
  | 'mixed_use' | 'hotel' | 'self_storage' | 'data_center' | 'land' | 'other';

export type PropertyStatus = 'prospect' | 'under_analysis' | 'under_contract' | 'owned' | 'sold' | 'passed';

export interface PropertyPhysicalAttributes {
  year_built?: number;
  total_units?: number;
  rentable_sqft?: number;
  lot_size_sqft?: number;
  stories?: number;
  parking_spaces?: number;
  bedrooms_total?: number;      // RESO field name
  bathrooms_total?: number;     // RESO field name
  living_area?: number;         // RESO field name
  amenities?: string[];
  unit_mix?: UnitMixEntry[];
  condition?: string;
  [key: string]: unknown;       // extensible per asset type
}

export interface UnitMixEntry {
  type: string;       // '1BR/1BA', '2BR/2BA', etc.
  count: number;
  avg_sqft: number;
  avg_rent: number;
}

// apps/api/src/modules/properties/property.routes.ts
// GET /api/properties?type=multifamily&status=owned&city=Austin&page=1&limit=20
// GET /api/properties/:id
// POST /api/properties
// PATCH /api/properties/:id
// DELETE /api/properties/:id
// GET /api/properties/search?lat=30.27&lng=-97.74&radius=10&type=multifamily
```

**Testing:**
- `test_create_property_multifamily` — Creates multifamily with unit_mix in physical_attributes
- `test_create_property_single_family` — Creates SFR with bedrooms/bathrooms RESO fields
- `test_create_property_nnn_retail` — Creates NNN with tenant and lease details in JSONB
- `test_list_properties_paginated` — Returns paginated list scoped to organization
- `test_filter_by_type_and_status` — Filters correctly by property_type and status
- `test_geo_search_radius` — Returns properties within N miles of a coordinate
- `test_jsonb_search_units` — Finds multifamily with 100+ units via JSONB query
- `test_property_update_partial` — PATCH updates only provided fields

### Task 3.2: Document Management

**What:** Implement file upload to S3/MinIO, document categorization (OM, rent roll, T12, appraisal, etc.), and document listing per property/deal/fund.

**Design:**

```typescript
// apps/api/src/modules/documents/document.service.ts
export type DocumentType =
  | 'photo' | 'floorplan' | 'site_plan' | 'offering_memorandum' | 'appraisal'
  | 'inspection_report' | 'environmental' | 'title_report' | 'survey'
  | 'rent_roll' | 't12' | 'tax_return' | 'insurance' | 'lease' | 'other';

export interface UploadDocumentInput {
  organizationId: string;
  propertyId?: string;
  dealId?: string;
  fundId?: string;
  documentType: DocumentType;
  file: MultipartFile;
}

export interface DocumentMetadata {
  file_size_bytes: number;
  mime_type: string;
  description?: string;
  uploaded_by: string;
  ai_extracted?: boolean;
  extraction_confidence?: number;
}

// POST /api/documents — multipart upload
// GET /api/documents?propertyId=xxx&type=rent_roll
// GET /api/documents/:id/download — presigned URL
// DELETE /api/documents/:id
```

**Testing:**
- `test_upload_pdf_document` — Uploads PDF, stores in S3, creates database record
- `test_upload_size_limit` — Rejects files exceeding 50MB
- `test_upload_type_validation` — Rejects non-allowed MIME types
- `test_list_documents_by_property` — Returns documents filtered by propertyId
- `test_download_presigned_url` — Returns time-limited presigned URL for download
- `test_delete_removes_s3_object` — Deleting document removes both DB record and S3 object

### Task 3.3: Rent Roll Import

**What:** Build CSV/Excel rent roll import with column mapping, validation, and storage as JSONB array in rent_rolls table. Support manual entry fallback.

**Design:**

```typescript
// apps/api/src/modules/properties/rent-roll.service.ts
export interface RentRollUnit {
  unit: string;
  type?: string;          // '1BR/1BA'
  sqft?: number;
  status: 'occupied' | 'vacant' | 'model' | 'down';
  tenant?: string;
  monthly_rent?: number;
  market_rent?: number;
  lease_start?: string;   // ISO date
  lease_end?: string;     // ISO date
  deposit?: number;
  escalation_pct?: number;
  days_vacant?: number;
}

export interface RentRollSummary {
  total_units: number;
  occupied_units: number;
  occupancy_pct: number;
  total_monthly_rent: number;
  avg_rent_per_unit: number;
  avg_rent_per_sqft?: number;
  weighted_avg_lease_term_months?: number;
}

export interface RentRollImportResult {
  rentRollId: string;
  unitsImported: number;
  validationErrors: ValidationError[];
  summary: RentRollSummary;
}

// POST /api/properties/:id/rent-rolls — upload CSV/XLSX file
// POST /api/properties/:id/rent-rolls/manual — manual entry
// GET /api/properties/:id/rent-rolls — list by date
// GET /api/properties/:id/rent-rolls/:id — single rent roll with units
```

**Testing:**
- `test_import_csv_200_units` — Imports 200-unit CSV and produces correct summary
- `test_import_xlsx_format` — Handles Excel files with multiple sheets
- `test_column_mapping_flexibility` — Maps alternate column headers (e.g., "Unit #" -> unit)
- `test_validation_negative_rent` — Rejects negative rent values with specific error
- `test_occupancy_calculation` — Summary occupancy_pct = occupied / total
- `test_manual_entry_single_unit` — Manual entry creates rent roll with one unit
- `test_historical_rent_rolls` — Multiple imports create separate records with different as_of_date
- `test_weighted_avg_lease_term` — WALT calculation matches hand-computed value

### Task 3.4: Operating Statement (T12) Import

**What:** Build CSV/Excel T12 import with line item categorization and storage in operating_statements table. Auto-calculate NOI from imported data.

**Design:**

```typescript
// apps/api/src/modules/properties/operating-statement.service.ts
export interface OperatingStatementLineItem {
  category: string;   // 'rental_income', 'vacancy', 'utilities', 'payroll', etc.
  label: string;      // 'Gross Potential Rent', 'Water & Sewer', etc.
  amount: number;
}

export interface OperatingStatementInput {
  propertyId: string;
  statementType: 't12' | 'annual' | 'budget' | 'monthly';
  periodStart: string;
  periodEnd: string;
  lineItems: {
    revenue: OperatingStatementLineItem[];
    expenses: OperatingStatementLineItem[];
  };
}

// POST /api/properties/:id/operating-statements — upload or manual entry
// GET /api/properties/:id/operating-statements — list by period
// GET /api/properties/:id/operating-statements/:id — single statement
```

**Testing:**
- `test_import_t12_csv` — Imports T12 with revenue and expense line items
- `test_noi_auto_calculation` — net_operating_income = total_revenue - total_expenses
- `test_expense_categorization` — Groups line items by category correctly
- `test_multiple_periods` — Can import T12, prior year annual, and budget separately
- `test_line_item_validation` — Rejects line items with missing category or label

---

## Phase 4: Financial Calculation Engine

**Objective:** Build the core financial calculation library that computes IRR, NPV, DCF, DSCR, equity multiple, amortization schedules, and cash flow projections.

**Dependencies:** Phase 1 (packages/financial-engine has no API/DB dependency)

### Task 4.1: Core Return Metrics

**What:** Implement IRR, NPV, cash-on-cash return, equity multiple, and cap rate calculations as pure functions with high-precision decimal arithmetic.

**Design:**

```typescript
// packages/financial-engine/src/irr.ts
/**
 * Calculate Internal Rate of Return using Newton-Raphson method.
 * @param cashFlows Array of cash flows where index 0 is initial investment (negative)
 * @param guess Initial guess for IRR (default 0.10)
 * @param tolerance Convergence tolerance (default 1e-10)
 * @param maxIterations Maximum iterations (default 1000)
 * @returns IRR as a decimal (e.g., 0.142 = 14.2%)
 */
export function calculateIRR(
  cashFlows: number[],
  guess: number = 0.10,
  tolerance: number = 1e-10,
  maxIterations: number = 1000
): number | null;

// packages/financial-engine/src/npv.ts
/**
 * Calculate Net Present Value.
 * @param discountRate Annual discount rate as decimal
 * @param cashFlows Array of periodic cash flows (index 0 = time 0)
 * @returns NPV as a number
 */
export function calculateNPV(discountRate: number, cashFlows: number[]): number;

// packages/financial-engine/src/metrics.ts
export interface InvestmentMetrics {
  irr: number | null;
  npv: number;
  equityMultiple: number;
  cashOnCashReturn: number;    // Year 1 cash flow / total equity invested
  capRate: number;             // NOI / purchase price
  dscr: number;               // NOI / annual debt service
}

export function calculateInvestmentMetrics(input: {
  purchasePrice: number;
  totalEquity: number;
  annualNOI: number;
  annualDebtService: number;
  discountRate: number;
  cashFlows: number[];          // full hold period including exit
  year1CashFlow: number;
}): InvestmentMetrics;
```

**Testing:**
- `test_irr_known_answer` — IRR of [-1000, 300, 400, 500] = ~14.49%
- `test_irr_single_period` — IRR of [-100, 110] = 10%
- `test_irr_no_convergence` — Returns null for cash flows with no real IRR
- `test_irr_negative_return` — Handles deals that lose money (negative IRR)
- `test_npv_zero_discount` — NPV at 0% discount = sum of cash flows
- `test_npv_positive_discount` — NPV at 10% discount matches hand-calculated value
- `test_equity_multiple_calculation` — Total distributions / total equity = correct multiple
- `test_cap_rate_calculation` — NOI / purchase price with correct precision
- `test_dscr_calculation` — NOI / debt service with edge cases (zero debt)
- `test_cash_on_cash_return` — Year 1 cash flow / equity invested

### Task 4.2: DCF Cash Flow Projection

**What:** Build the DCF engine that generates annual or monthly cash flow projections from deal assumptions, including revenue growth, expense growth, vacancy, and debt service.

**Design:**

```typescript
// packages/financial-engine/src/dcf.ts
export interface DCFInput {
  purchasePrice: number;
  closingCostsPct: number;
  capexBudget: number;
  holdPeriodYears: number;
  periodType: 'annual' | 'monthly';

  // Revenue assumptions
  grossPotentialRent: number;      // Year 1
  rentGrowthPct: number;
  vacancyRatePct: number;
  otherIncome: number;

  // Expense assumptions
  operatingExpenses: number;       // Year 1
  expenseGrowthPct: number;
  realEstateTaxes: number;
  insurance: number;
  managementFeePct: number;
  reserves: number;

  // Exit assumptions
  exitCapRate: number;
  sellingCostsPct: number;

  // Financing
  financing: FinancingInput[];

  // Discount rate for NPV
  discountRate: number;
}

export interface CashFlowPeriod {
  period: number;
  startDate: string;
  endDate: string;
  grossPotentialRent: number;
  vacancyLoss: number;
  otherIncome: number;
  effectiveGrossIncome: number;
  operatingExpenses: number;
  realEstateTaxes: number;
  insurance: number;
  managementFee: number;
  reserves: number;
  totalExpenses: number;
  netOperatingIncome: number;
  debtService: number;
  capitalExpenditures: number;
  cashFlowBeforeTax: number;
  netSaleProceeds?: number;        // only in final period
}

export interface DCFResult {
  periods: CashFlowPeriod[];
  metrics: InvestmentMetrics;
  totalNOI: number;
  totalCashFlow: number;
  avgCashOnCash: number;
  terminalValue: number;
}

export function projectCashFlows(input: DCFInput): DCFResult;
```

**Testing:**
- `test_dcf_5_year_multifamily` — 5-year hold with known inputs matches hand-calculated cash flows
- `test_dcf_10_year_office` — 10-year hold with NNN assumptions
- `test_dcf_rent_growth_compounds` — Year 5 GPR = Year 1 GPR * (1 + growth)^4
- `test_dcf_vacancy_applied` — EGI = GPR - vacancy_loss + other_income
- `test_dcf_expense_growth_compounds` — Expenses grow at specified rate
- `test_dcf_management_fee_percentage` — Management fee = EGI * managementFeePct
- `test_dcf_terminal_value` — Terminal value = Final NOI / exit cap rate
- `test_dcf_net_sale_proceeds` — Final period includes sale proceeds minus selling costs and loan payoff
- `test_dcf_monthly_periods` — Monthly projection produces 60 periods for 5-year hold
- `test_dcf_zero_debt` — All-cash deal has no debt service

### Task 4.3: Amortization & Debt Service

**What:** Build amortization schedule calculation supporting fixed-rate, interest-only periods, and variable rate structures.

**Design:**

```typescript
// packages/financial-engine/src/amortization.ts
export interface FinancingInput {
  loanAmount: number;
  interestRate: number;          // annual
  rateType: 'fixed' | 'variable';
  loanTermMonths: number;
  amortizationMonths: number;
  ioPeriodsMonths: number;       // interest-only period
  originationFeePct: number;
}

export interface AmortizationPeriod {
  month: number;
  beginningBalance: number;
  payment: number;
  principal: number;
  interest: number;
  endingBalance: number;
}

export interface AmortizationResult {
  schedule: AmortizationPeriod[];
  annualDebtService: number;
  totalInterest: number;
  balloonPayment: number;        // remaining balance at loan maturity
}

export function calculateAmortization(input: FinancingInput): AmortizationResult;
```

**Testing:**
- `test_amort_fixed_30yr` — 30-year fixed matches standard amort table
- `test_amort_io_period` — First 24 months are interest-only, then amortizing
- `test_amort_balloon` — 5-year term / 30-year amort produces balloon payment
- `test_amort_annual_debt_service` — Sum of 12 monthly payments matches annual debt service
- `test_amort_total_interest` — Total interest over loan term is accurate
- `test_amort_multiple_loans` — Aggregates debt service from senior + mezzanine

### Task 4.4: Scenario & Sensitivity Analysis

**What:** Build scenario modelling (bear/base/bull) and sensitivity table generation with configurable variable axes.

**Design:**

```typescript
// packages/financial-engine/src/sensitivity.ts
export interface ScenarioOverrides {
  rentGrowthPct?: number;
  expenseGrowthPct?: number;
  vacancyRatePct?: number;
  exitCapRate?: number;
  discountRate?: number;
}

export interface SensitivityTableInput {
  baseCaseInput: DCFInput;
  rowVariable: keyof ScenarioOverrides;
  rowValues: number[];
  colVariable: keyof ScenarioOverrides;
  colValues: number[];
  outputMetric: 'irr' | 'npv' | 'equityMultiple' | 'cashOnCashReturn';
}

export interface SensitivityTable {
  rowVariable: string;
  colVariable: string;
  outputMetric: string;
  rows: { label: string; values: (number | null)[] }[];
  colLabels: string[];
}

export function runScenarios(
  baseCaseInput: DCFInput,
  overrides: Record<string, ScenarioOverrides>
): Record<string, DCFResult>;

export function generateSensitivityTable(input: SensitivityTableInput): SensitivityTable;
```

**Testing:**
- `test_three_scenarios` — Bear/Base/Bull produce three distinct DCFResults
- `test_scenario_override_single_field` — Overriding only exit_cap_rate changes only exit metrics
- `test_sensitivity_table_5x5` — Generates 5x5 grid of IRR for cap rate vs vacancy
- `test_sensitivity_table_symmetry` — Base case value appears at intersection of base assumptions
- `test_scenario_comparison` — Bear IRR < Base IRR < Bull IRR for standard assumptions

---

## Phase 5: Deal Underwriting API & UI

**Objective:** Build the deal creation, editing, scenario management, and cash flow projection API endpoints and the primary underwriting UI.

**Dependencies:** Phase 3, Phase 4

### Task 5.1: Deal CRUD API

**What:** Implement deal creation, listing, updating, and deletion with support for all deal types (acquisition, refinance, value_add, fix_and_flip, development). Include financing structure management.

**Design:**

```typescript
// apps/api/src/modules/deals/deal.routes.ts
// POST   /api/deals                              — create deal
// GET    /api/deals?status=draft&page=1           — list deals
// GET    /api/deals/:id                           — get deal with scenarios
// PATCH  /api/deals/:id                           — update deal assumptions
// DELETE /api/deals/:id                           — soft delete

// POST   /api/deals/:id/financing                 — add financing structure
// PATCH  /api/deals/:id/financing/:finId          — update financing
// DELETE /api/deals/:id/financing/:finId          — remove financing

// POST   /api/deals/:id/scenarios                 — create scenario
// PATCH  /api/deals/:id/scenarios/:scenId         — update scenario overrides
// DELETE /api/deals/:id/scenarios/:scenId         — remove scenario
// POST   /api/deals/:id/scenarios/:scenId/compute — trigger cash flow computation

export interface CreateDealInput {
  propertyId: string;
  dealName: string;
  dealType: DealType;
  dealStrategy?: DealStrategy;
  purchasePrice?: number;
  holdPeriodMonths?: number;
  exitCapRate?: number;
  discountRate?: number;
  acquisitionAssumptions?: Record<string, unknown>;
  financing?: FinancingInput[];
}

export interface DealResponse {
  deal: Deal;
  property: PropertySummary;
  financing: DealFinancing[];
  scenarios: DealScenarioSummary[];
}
```

**Testing:**
- `test_create_deal_acquisition` — Creates acquisition deal linked to property
- `test_create_deal_fix_and_flip` — Creates flip deal with rehab assumptions in JSONB
- `test_deal_pipeline_list` — Lists deals filtered by status, sorted by creation date
- `test_update_deal_assumptions` — PATCH updates only specified assumption fields
- `test_add_financing` — Adds senior debt and mezzanine structures to deal
- `test_create_three_scenarios` — Creates bear/base/bull scenarios for a deal
- `test_compute_cash_flows` — Triggers DCF engine and stores result in cash_flow_streams
- `test_deal_response_includes_scenarios` — GET deal returns all scenarios with metrics

### Task 5.2: Deal Underwriting UI

**What:** Build the multi-step deal creation form, assumption editor, financing configuration, and scenario comparison interface.

**Design:**

```typescript
// apps/web/components/deals/DealWizard.tsx
// Step 1: Select property (search or create new)
// Step 2: Deal type & strategy selection
// Step 3: Acquisition assumptions (purchase price, closing costs, capex)
// Step 4: Revenue assumptions (GPR, vacancy, rent growth, other income)
// Step 5: Expense assumptions (opex, taxes, insurance, management, reserves)
// Step 6: Financing configuration (add one or more loan structures)
// Step 7: Exit assumptions (exit cap rate, selling costs, hold period)
// Step 8: Review & create scenarios

// apps/web/components/deals/ScenarioComparison.tsx
interface ScenarioComparisonProps {
  scenarios: DealScenarioWithMetrics[];
}
// Side-by-side table showing: IRR, NPV, equity multiple, DSCR, cash-on-cash
// for each scenario with color-coded indicators (red/yellow/green)

// apps/web/components/deals/CashFlowTable.tsx
interface CashFlowTableProps {
  periods: CashFlowPeriod[];
  showMonthly: boolean;
}
// Spreadsheet-style table with expandable sections for revenue, expenses, NOI, debt service
```

**Testing:**
- `test_deal_wizard_complete_flow` — E2E: walk through all 8 steps, create deal
- `test_deal_wizard_back_navigation` — Can go back to previous steps without losing data
- `test_scenario_comparison_renders` — Side-by-side comparison table shows three scenarios
- `test_cash_flow_table_annual` — Cash flow table displays correct periods
- `test_assumption_edit_triggers_recompute` — Changing an assumption recomputes cash flows
- `test_responsive_layout` — Deal UI adapts to mobile viewport

### Task 5.3: Sensitivity Analysis UI

**What:** Build interactive sensitivity table component where users select two variables and an output metric, and see a color-coded heat map of results.

**Design:**

```typescript
// apps/web/components/deals/SensitivityHeatMap.tsx
interface SensitivityHeatMapProps {
  table: SensitivityTable;
  baseCase: { row: number; col: number };
}
// Color gradient: red (below base case) -> white (base case) -> green (above base case)
// Hovering a cell shows full metrics for that scenario
// Users can select row/col variables from dropdowns:
// - Exit cap rate, Vacancy rate, Rent growth, Expense growth, Discount rate, Purchase price
```

**Testing:**
- `test_sensitivity_table_renders_5x5` — Renders 25 cells with correct values
- `test_sensitivity_color_coding` — Base case cell is white; worse outcomes are red
- `test_sensitivity_variable_swap` — Changing row/col variable regenerates table
- `test_sensitivity_metric_toggle` — Switching from IRR to NPV updates all cells

---

## Phase 6: Market Data Integration

**Objective:** Integrate external data providers (ATTOM, BatchData) for property data enrichment, comparable transactions, and submarket metrics.

**Dependencies:** Phase 3

### Task 6.1: Data Provider Adapter Layer

**What:** Build a provider-agnostic adapter layer that abstracts ATTOM and BatchData APIs behind a common interface. Include caching, rate limiting, and error handling.

**Design:**

```typescript
// apps/api/src/modules/market-data/providers/types.ts
export interface MarketDataProvider {
  name: string;
  getPropertyData(address: string): Promise<PropertyEnrichmentData>;
  getComparableSales(params: CompSearchParams): Promise<ComparableTransaction[]>;
  getSubmarketMetrics(params: SubmarketParams): Promise<SubmarketMetrics>;
  getRentEstimate(address: string): Promise<RentEstimate>;
}

export interface PropertyEnrichmentData {
  attomId?: string;
  assessedValue?: number;
  lastSalePrice?: number;
  lastSaleDate?: string;
  yearBuilt?: number;
  totalUnits?: number;
  lotSizeAcres?: number;
  taxAmount?: number;
  zoning?: string;
  ownerName?: string;
}

export interface ComparableTransaction {
  address: string;
  city: string;
  state: string;
  propertyType: string;
  salePrice: number;
  pricePerUnit?: number;
  pricePerSqft?: number;
  capRate?: number;
  saleDate: string;
  units?: number;
  sqft?: number;
  yearBuilt?: number;
  distanceMiles: number;
  dataSource: string;
}

// apps/api/src/modules/market-data/providers/attom.provider.ts
export class ATTOMProvider implements MarketDataProvider { ... }

// apps/api/src/modules/market-data/providers/batchdata.provider.ts
export class BatchDataProvider implements MarketDataProvider { ... }
```

**Testing:**
- `test_attom_property_lookup` — Fetches property data from ATTOM API (mocked)
- `test_batchdata_property_lookup` — Fetches property data from BatchData API (mocked)
- `test_provider_fallback` — If ATTOM fails, falls back to BatchData
- `test_cache_hit` — Second call for same address returns cached result from Redis
- `test_rate_limiting` — Respects API rate limits with backoff
- `test_provider_error_handling` — Returns partial data when some fields unavailable

### Task 6.2: Comparable Transaction Search

**What:** Build comparable transaction search that queries external providers and stores results in the comparables table. Include radius-based, property-type, and date-range filtering.

**Design:**

```typescript
// apps/api/src/modules/market-data/comparables.service.ts
export interface CompSearchParams {
  propertyId: string;            // anchor property
  radiusMiles: number;           // search radius
  propertyType?: string;
  minSaleDate?: string;
  maxSaleDate?: string;
  minUnits?: number;
  maxUnits?: number;
  limit?: number;
}

// GET  /api/properties/:id/comparables?radius=2&minDate=2025-01-01
// POST /api/properties/:id/comparables/refresh — fetch fresh comps from providers
```

**Testing:**
- `test_comp_search_returns_results` — Returns comps within radius
- `test_comp_search_filters_by_date` — Only returns sales after minSaleDate
- `test_comp_search_filters_by_type` — Only returns matching property type
- `test_comp_refresh_stores_in_db` — Refresh fetches from provider and persists
- `test_comp_deduplication` — Same transaction from two providers stored once

### Task 6.3: Submarket Data & Market Data Table

**What:** Build submarket metrics collection and storage. Periodically sync market data from providers and store in market_data table for use in AI assumption generation.

**Design:**

```typescript
// apps/api/src/modules/market-data/submarket.service.ts
export interface SubmarketMetrics {
  submarket: string;
  propertyType: string;
  dataDate: string;
  dataSource: string;
  metrics: {
    avg_cap_rate?: number;
    avg_rent_per_sqft?: number;
    avg_vacancy_rate?: number;
    avg_price_per_unit?: number;
    median_sale_price?: number;
    rent_growth_yoy?: number;
    population_growth?: number;
    employment_growth?: number;
  };
}

// GET /api/market-data?submarket=Austin+CBD&propertyType=multifamily
// POST /api/market-data/sync — trigger sync for specified submarkets
```

**Testing:**
- `test_submarket_metrics_fetch` — Retrieves and stores submarket data
- `test_submarket_historical_query` — Can query market data at different dates
- `test_sync_job_creates_records` — Background sync creates market_data records
- `test_market_data_by_property_type` — Filters metrics by property type correctly

---

## Phase 7: Report Generation

**Objective:** Build investor-ready PDF reports for deal summaries and basic portfolio views. Include shareable public links.

**Dependencies:** Phase 5

### Task 7.1: Deal Summary Report

**What:** Generate PDF deal summary reports containing property overview, assumptions, cash flow projections, scenario comparison, and sensitivity analysis.

**Design:**

```typescript
// apps/api/src/modules/reports/deal-summary.generator.ts
export interface DealSummaryReportInput {
  dealId: string;
  includeScenarios: boolean;
  includeSensitivity: boolean;
  includeCashFlows: boolean;
  includeComps: boolean;
}

export interface DealSummaryReport {
  sections: {
    coverPage: CoverPageData;
    executiveSummary: ExecutiveSummaryData;
    propertyOverview: PropertyOverviewData;
    investmentAssumptions: AssumptionsData;
    cashFlowProjections: CashFlowPeriod[];
    scenarioComparison: ScenarioComparisonData;
    sensitivityAnalysis?: SensitivityTable;
    comparableTransactions?: ComparableTransaction[];
    disclaimer: string;
  };
}

// POST /api/reports/deal-summary — generate report
// GET  /api/reports/:id — get report metadata
// GET  /api/reports/:id/download — download PDF
// POST /api/reports/:id/share — generate shareable link
```

**Testing:**
- `test_deal_summary_pdf_generation` — Generates valid PDF file
- `test_deal_summary_includes_metrics` — PDF contains IRR, NPV, equity multiple
- `test_deal_summary_includes_cash_flows` — PDF includes cash flow table
- `test_deal_summary_scenario_comparison` — PDF shows side-by-side scenarios
- `test_shareable_link_works` — Public link renders report without authentication
- `test_shareable_link_expiry` — Expired links return 404

### Task 7.2: Report Template System

**What:** Build a template engine that allows users to customize which sections appear in reports and apply organization branding (logo, colors, fonts).

**Design:**

```typescript
// apps/api/src/modules/reports/template.service.ts
export interface ReportTemplate {
  id: string;
  organizationId: string;
  name: string;
  reportType: 'deal_summary' | 'lp_quarterly' | 'portfolio_performance';
  sections: string[];
  branding: {
    logoUrl?: string;
    primaryColor: string;
    secondaryColor: string;
    fontFamily: string;
    headerText?: string;
    footerText?: string;
  };
}

// GET    /api/report-templates
// POST   /api/report-templates
// PATCH  /api/report-templates/:id
// DELETE /api/report-templates/:id
```

**Testing:**
- `test_custom_template_creation` — Creates template with subset of sections
- `test_branding_applied_to_pdf` — Generated PDF uses org logo and colors
- `test_default_template_fallback` — Missing template falls back to system default
- `test_template_section_ordering` — Sections appear in user-specified order

---

## Phase 8: Investor & Syndication Management

**Objective:** Build investor management, fund/syndication structures, waterfall calculations, and distribution tracking.

**Dependencies:** Phase 5

### Task 8.1: Investor Management

**What:** Implement investor CRUD with accreditation tracking, KYC status, and GDPR/CCPA consent management.

**Design:**

```typescript
// packages/types/src/syndication.ts
export type InvestorType = 'individual' | 'entity' | 'trust' | 'ira' | 'joint';
export type AccreditationStatus = 'accredited' | 'non_accredited' | 'qualified_purchaser' | 'pending' | 'expired';

export interface Investor {
  id: string;
  organizationId: string;
  investorType: InvestorType;
  legalName: string;
  email?: string;
  accreditationStatus?: AccreditationStatus;
  accreditationExpiry?: string;
  kycStatus: 'pending' | 'verified' | 'failed';
  details: InvestorDetails;      // JSONB: varies by investor type
}

// POST   /api/investors
// GET    /api/investors?accreditationStatus=accredited
// GET    /api/investors/:id
// PATCH  /api/investors/:id
// GET    /api/investors/:id/positions — all fund commitments
// POST   /api/investors/:id/accreditation — record accreditation verification
```

**Testing:**
- `test_create_individual_investor` — Creates individual with accreditation fields
- `test_create_entity_investor` — Creates entity with EIN and authorized signers
- `test_accreditation_expiry_warning` — API flags investors with expiry within 30 days
- `test_kyc_status_update` — Updates KYC status with verification date
- `test_gdpr_consent_recorded` — Records data consent date and flag
- `test_investor_positions_aggregation` — Lists all fund positions for an investor

### Task 8.2: Fund & Syndication Structure

**What:** Build fund management with configurable waterfall structures stored as JSONB. Support single-asset and blind-pool fund types.

**Design:**

```typescript
// packages/types/src/syndication.ts
export type FundType = 'single_asset' | 'blind_pool' | 'specified_pool' | 'open_end' | 'closed_end';

export interface WaterfallTier {
  tier: number;
  name: string;
  hurdleType: 'irr' | 'equity_multiple' | 'flat';
  hurdleRate?: number;
  lpSplit: number;
  gpSplit: number;
  isCatchup?: boolean;
  catchupPct?: number;
}

export interface Fund {
  id: string;
  organizationId: string;
  fundName: string;
  fundType: FundType;
  status: 'raising' | 'closed' | 'operating' | 'liquidating' | 'terminated';
  totalRaiseTarget?: number;
  waterfallStructure: WaterfallTier[];
  fundDetails: FundDetails;     // JSONB: entity type, reg D, fees
  grossIrr?: number;
  netIrr?: number;
  tvpi?: number;
  dpi?: number;
  rvpi?: number;
}

// POST   /api/funds
// GET    /api/funds?status=operating
// GET    /api/funds/:id
// PATCH  /api/funds/:id
// POST   /api/funds/:id/properties — add property to fund
// DELETE /api/funds/:id/properties/:propId — remove property from fund
// POST   /api/funds/:id/commitments — record investor commitment
// GET    /api/funds/:id/investors — list investors with positions
```

**Testing:**
- `test_create_fund_with_waterfall` — Creates fund with 4-tier American waterfall
- `test_add_property_to_fund` — Links property with ownership percentage
- `test_record_investor_commitment` — Records commitment with class and subscription date
- `test_fund_investor_list` — Returns all investors with commitment amounts and ownership pct
- `test_fund_status_lifecycle` — Transitions raising -> closed -> operating -> liquidating
- `test_reg_d_tracking` — Stores Form D filing date and amendment dates

### Task 8.3: Waterfall Distribution Calculator

**What:** Build the waterfall distribution engine that calculates GP/LP splits according to the fund's waterfall structure. Support American and European waterfall styles.

**Design:**

```typescript
// packages/financial-engine/src/waterfall.ts
export interface WaterfallInput {
  tiers: WaterfallTier[];
  totalDistributableCash: number;
  investors: {
    investorId: string;
    investorClass: string;
    commitmentAmount: number;
    paidInCapital: number;
    priorDistributions: number;
  }[];
  fundTotalEquity: number;
  gpCommitment: number;
  waterfallStyle: 'american' | 'european';
}

export interface WaterfallResult {
  investorDistributions: {
    investorId: string;
    grossAmount: number;
    netAmount: number;
    tierBreakdown: { tierName: string; amount: number }[];
    cumulativeIrr: number;
    cumulativeEquityMultiple: number;
  }[];
  gpDistribution: {
    grossAmount: number;
    tierBreakdown: { tierName: string; amount: number }[];
  };
}

export function calculateWaterfallDistribution(input: WaterfallInput): WaterfallResult;
```

**Testing:**
- `test_waterfall_return_of_capital_first` — All capital returned before preferred return
- `test_waterfall_preferred_return_8pct` — 8% preferred return calculated correctly
- `test_waterfall_gp_catchup` — GP catch-up tier distributes 100% to GP until 20/80 split achieved
- `test_waterfall_promote_split` — After catch-up, remaining cash splits 80/20 LP/GP
- `test_waterfall_multiple_investor_classes` — Class A and Class B get different preferred returns
- `test_waterfall_european_style` — European waterfall defers GP promote until full capital return
- `test_waterfall_partial_distribution` — Distributes correctly when cash is insufficient for full pref
- `test_waterfall_cumulative_irr` — Tracks cumulative IRR per investor across distributions

### Task 8.4: Distribution Recording & History

**What:** Build distribution recording, history tracking, and investor position summary with NCREIF PREA-aligned metrics (TVPI, DPI, RVPI).

**Design:**

```typescript
// POST /api/funds/:id/distributions — record distribution batch
// GET  /api/funds/:id/distributions — distribution history
// GET  /api/funds/:id/performance — fund-level TVPI, DPI, RVPI, IRR
// GET  /api/investors/:id/distributions — investor's distribution history
```

**Testing:**
- `test_record_distribution_batch` — Records distributions to all investors in a fund
- `test_distribution_history_chronological` — Returns distributions in date order
- `test_tvpi_calculation` — TVPI = (distributions + NAV) / paid-in capital
- `test_dpi_calculation` — DPI = total distributions / paid-in capital
- `test_rvpi_calculation` — RVPI = current NAV / paid-in capital
- `test_investor_specific_history` — Returns distributions for a single investor across funds

---

## Phase 9: AI-Powered Assumption Engine

**Objective:** Build the AI layer that generates underwriting assumptions from market data, parses documents, and provides natural language deal entry.

**Dependencies:** Phase 6, Phase 5

### Task 9.1: AI Assumption Generation

**What:** Build the AI assumption engine that takes a property address and deal type, retrieves submarket data from the market_data table and data providers, and generates calibrated underwriting assumptions with source citations.

**Design:**

```typescript
// packages/ai-client/src/assumption-engine.ts
export interface AssumptionGenerationInput {
  propertyId: string;
  dealType: DealType;
  propertyType: PropertyType;
  submarket: string;
  city: string;
  stateOrProvince: string;
}

export interface AIAssumption {
  field: string;                 // 'rent_growth_pct', 'exit_cap_rate', etc.
  value: number;
  confidence: number;            // 0.0 to 1.0
  sources: string[];             // 'ATTOM Austin CBD Q1 2026', '5 comparable sales'
  reasoning: string;
}

export interface AssumptionGenerationResult {
  assumptions: AIAssumption[];
  marketContext: {
    submarket: string;
    dataDate: string;
    avgCapRate: number;
    avgVacancy: number;
    rentGrowthYoy: number;
  };
}

// POST /api/ai/assumptions — generate assumptions for a deal
// POST /api/ai/assumptions/:id/accept — analyst accepts an assumption
// POST /api/ai/assumptions/:id/reject — analyst rejects an assumption
```

**Testing:**
- `test_assumption_generation_multifamily` — Generates rent growth, vacancy, cap rate, expenses
- `test_assumption_sources_populated` — Every assumption has at least one source citation
- `test_assumption_confidence_range` — Confidence scores between 0.0 and 1.0
- `test_assumption_uses_market_data` — Generated assumptions reflect submarket metrics
- `test_assumption_acceptance_flow` — Accepting an assumption applies it to the deal
- `test_assumption_provenance_stored` — AI metadata stored in deal's ai_metadata JSONB
- `test_assumption_without_market_data` — Gracefully handles missing submarket data with lower confidence

### Task 9.2: Document AI Extraction

**What:** Build AI-powered extraction of financial data from uploaded offering memorandums, rent rolls, and T12 operating statements (PDF/Excel).

**Design:**

```typescript
// packages/ai-client/src/document-extraction.ts
export interface DocumentExtractionInput {
  documentId: string;
  documentType: 'offering_memorandum' | 'rent_roll' | 't12';
  fileUrl: string;
}

export interface ExtractionResult {
  documentType: string;
  confidence: number;
  extractedData: RentRollUnit[] | OperatingStatementLineItem[] | OMExtractionData;
  rawText?: string;
  warnings: string[];
}

export interface OMExtractionData {
  propertyName?: string;
  askingPrice?: number;
  noi?: number;
  capRate?: number;
  totalUnits?: number;
  occupancy?: number;
  yearBuilt?: number;
  address?: string;
  unitMix?: UnitMixEntry[];
}

// POST /api/ai/extract — extract data from uploaded document
// GET  /api/ai/extract/:id/review — review extracted data before import
// POST /api/ai/extract/:id/confirm — confirm and import extracted data
```

**Testing:**
- `test_extract_rent_roll_pdf` — Extracts unit data from sample rent roll PDF
- `test_extract_t12_excel` — Extracts line items from T12 Excel file
- `test_extract_om_key_fields` — Extracts price, NOI, cap rate from offering memo
- `test_extraction_confidence_threshold` — Low-confidence extractions flagged for review
- `test_extraction_review_and_confirm` — Analyst can edit extracted data before import
- `test_extraction_error_on_unreadable` — Returns meaningful error for corrupted files

### Task 9.3: Natural Language Deal Entry

**What:** Build the NL deal entry feature where an analyst describes a deal in plain text and the system populates a full underwriting model with sourced assumptions.

**Design:**

```typescript
// packages/ai-client/src/nl-deal-entry.ts
export interface NLDealEntryInput {
  organizationId: string;
  description: string;
  // Example: "200-unit multifamily in Austin TX, built 2005, asking $15M,
  //           current NOI $900K, 93% occupied, planning 5-year hold with
  //           value-add strategy, $500K capex for unit renovations"
}

export interface NLDealEntryResult {
  parsedFields: {
    propertyType: PropertyType;
    city: string;
    state: string;
    totalUnits?: number;
    yearBuilt?: number;
    purchasePrice?: number;
    noi?: number;
    occupancy?: number;
    holdPeriodYears?: number;
    dealStrategy?: DealStrategy;
    capexBudget?: number;
  };
  suggestedDeal: CreateDealInput;
  suggestedProperty: CreatePropertyInput;
  aiAssumptions: AIAssumption[];
  confidence: number;
}

// POST /api/ai/nl-deal-entry — parse NL description into deal model
```

**Testing:**
- `test_nl_entry_multifamily_description` — Parses all fields from multifamily description
- `test_nl_entry_minimal_description` — Handles "3BR house in Austin for $300K"
- `test_nl_entry_fills_missing_assumptions` — AI fills unmentioned fields (rent growth, etc.)
- `test_nl_entry_creates_property_and_deal` — Confirmation creates both property and deal records
- `test_nl_entry_ambiguous_input` — Returns clarification questions for ambiguous descriptions

---

## Phase 10: Portfolio Dashboard

**Objective:** Build the portfolio-level dashboard showing aggregate performance across all properties and funds.

**Dependencies:** Phase 5, Phase 8

### Task 10.1: Portfolio Dashboard API

**What:** Build API endpoints that aggregate performance metrics across properties, deals, and funds for dashboard display.

**Design:**

```typescript
// apps/api/src/modules/portfolio/portfolio.routes.ts
export interface PortfolioDashboard {
  totalProperties: number;
  totalValueAUM: number;         // assets under management
  totalEquityDeployed: number;
  activeDeals: number;
  activeFunds: number;
  totalInvestors: number;

  performanceSummary: {
    portfolioIRR: number;
    portfolioEquityMultiple: number;
    avgCashOnCash: number;
    avgDSCR: number;
  };

  propertyAllocation: {
    byType: { type: string; count: number; value: number }[];
    byState: { state: string; count: number; value: number }[];
    byStatus: { status: string; count: number }[];
  };

  recentActivity: AuditLogEntry[];
  upcomingLeaseExpirations: LeaseExpiration[];
  accreditationExpirations: AccreditationExpiration[];
}

// GET /api/portfolio/dashboard
// GET /api/portfolio/properties — properties with latest valuations
// GET /api/portfolio/performance — time-series performance data
```

**Testing:**
- `test_dashboard_aggregation` — Returns correct counts and totals
- `test_dashboard_allocation_by_type` — Property type allocation sums correctly
- `test_dashboard_allocation_by_state` — Geographic allocation sums correctly
- `test_dashboard_performance_metrics` — Portfolio IRR and equity multiple computed
- `test_dashboard_upcoming_lease_expirations` — Shows leases expiring within 90 days
- `test_dashboard_accreditation_warnings` — Flags investors with expiring accreditation

### Task 10.2: Portfolio Dashboard UI

**What:** Build the dashboard page with KPI cards, allocation charts (pie/bar), property map, deal pipeline Kanban, and activity feed.

**Design:**

```typescript
// apps/web/app/(dashboard)/portfolio/page.tsx
// Layout:
// Row 1: KPI cards (AUM, Properties, Active Deals, Portfolio IRR, Avg Cash-on-Cash)
// Row 2: Allocation donut chart (by type) | Geographic bar chart (by state)
// Row 3: Property map (Mapbox/Leaflet with property markers)
// Row 4: Deal pipeline Kanban (draft -> in_review -> approved -> under_contract -> closed)
// Row 5: Activity feed (recent audit log entries)

// apps/web/components/charts/AllocationDonut.tsx
// apps/web/components/charts/PerformanceTimeSeries.tsx
// apps/web/components/charts/PropertyMap.tsx
// apps/web/components/deals/DealPipelineKanban.tsx
```

**Testing:**
- `test_dashboard_renders_kpi_cards` — E2E: all 5 KPI cards visible with data
- `test_dashboard_allocation_chart` — Donut chart renders with correct segments
- `test_dashboard_property_map` — Map displays markers at property coordinates
- `test_dashboard_deal_kanban` — Deals displayed in correct pipeline columns
- `test_dashboard_empty_state` — New organization sees empty state prompts

---

## Phase 11: LP Portal & Quarterly Reports

**Objective:** Build the limited partner portal where investors can view their positions, distributions, and fund performance. Build automated quarterly LP report generation.

**Dependencies:** Phase 8, Phase 7

### Task 11.1: LP Portal

**What:** Build a read-only investor portal where LPs can view their fund positions, distribution history, commitment details, and download reports.

**Design:**

```typescript
// apps/web/app/(dashboard)/investor-portal/page.tsx
// Accessible to users with role 'lp_viewer'
// Shows:
// - Fund summary cards (each fund the LP is committed to)
// - Position details: commitment, paid-in, distributions, ownership %
// - Performance: net IRR, equity multiple, TVPI, DPI, RVPI
// - Distribution history table with waterfall tier breakdown
// - Downloadable reports (quarterly, K-1, tax documents)

export interface LPPortalData {
  investor: InvestorSummary;
  positions: FundPosition[];
  distributions: DistributionHistory[];
  reports: ReportDownload[];
}

// GET /api/investor-portal — LP-scoped data
// GET /api/investor-portal/funds/:fundId — detailed fund position
// GET /api/investor-portal/distributions — all distributions across funds
```

**Testing:**
- `test_lp_portal_shows_own_positions` — LP sees only their fund positions
- `test_lp_portal_does_not_show_other_investors` — LP cannot see other investors' data
- `test_lp_portal_distribution_history` — Shows chronological distribution list
- `test_lp_portal_performance_metrics` — Displays net IRR, equity multiple, TVPI
- `test_lp_portal_report_downloads` — LP can download their quarterly reports

### Task 11.2: Quarterly LP Report Generator

**What:** Build automated quarterly LP report generation with performance summary, property updates, market commentary (AI-generated), and distribution details.

**Design:**

```typescript
// apps/api/src/modules/reports/lp-quarterly.generator.ts
export interface LPQuarterlyReportInput {
  fundId: string;
  quarter: string;             // 'Q1 2026'
  periodStart: string;         // '2026-01-01'
  periodEnd: string;           // '2026-03-31'
  includeAICommentary: boolean;
}

export interface LPQuarterlyReport {
  sections: {
    coverPage: CoverPageData;
    executiveSummary: string;           // AI-generated narrative
    fundPerformance: FundPerformanceData;
    propertyUpdates: PropertyUpdateData[];
    distributionSummary: DistributionSummaryData;
    marketCommentary: string;           // AI-generated
    capitalAccountSummary: CapitalAccountData[];
    appendix: AppendixData;
  };
}

// POST /api/reports/lp-quarterly — generate quarterly report for a fund
// POST /api/reports/lp-quarterly/bulk — generate for all active funds
```

**Testing:**
- `test_quarterly_report_generation` — Generates valid PDF with all sections
- `test_quarterly_report_ai_commentary` — AI generates market commentary from submarket data
- `test_quarterly_report_per_investor` — Each LP gets a report with their specific capital account
- `test_quarterly_report_performance_metrics` — Report includes TVPI, DPI, RVPI, net IRR
- `test_quarterly_report_bulk_generation` — Generates reports for all active funds

---

## Phase 12: API-First Architecture & Integrations

**Objective:** Finalize the public API with OpenAPI documentation, build integration endpoints for property management systems, and implement webhook notifications.

**Dependencies:** Phase 10, Phase 11

### Task 12.1: OpenAPI Documentation & SDK

**What:** Generate OpenAPI 3.1 spec from Fastify route schemas. Publish interactive API documentation and generate TypeScript/Python client SDKs.

**Design:**

```typescript
// apps/api/src/plugins/openapi.plugin.ts
import fastifySwagger from '@fastify/swagger';
import fastifySwaggerUi from '@fastify/swagger-ui';

export const openapiPlugin: FastifyPluginAsync = async (fastify) => {
  await fastify.register(fastifySwagger, {
    openapi: {
      info: {
        title: 'Real Estate Investment Analysis API',
        version: '1.0.0',
        description: 'AI-native real estate deal underwriting, portfolio tracking, and LP reporting',
      },
      servers: [{ url: 'https://api.reia.dev' }],
      components: {
        securitySchemes: {
          bearerAuth: { type: 'http', scheme: 'bearer', bearerFormat: 'JWT' },
          apiKey: { type: 'apiKey', in: 'header', name: 'X-API-Key' },
        },
      },
    },
  });
};
```

**Testing:**
- `test_openapi_spec_valid` — Generated spec passes OpenAPI 3.1 validation
- `test_all_routes_documented` — Every route has a schema definition
- `test_swagger_ui_accessible` — /docs renders interactive API documentation
- `test_typescript_sdk_generation` — openapi-typescript-codegen produces working client
- `test_api_key_authentication` — API key auth works for programmatic access

### Task 12.2: Webhook System

**What:** Build a webhook system that notifies external systems when key events occur (deal status change, distribution posted, report generated).

**Design:**

```typescript
// apps/api/src/modules/webhooks/webhook.service.ts
export type WebhookEvent =
  | 'deal.created' | 'deal.status_changed' | 'deal.assumptions_updated'
  | 'distribution.posted' | 'report.generated' | 'investor.accreditation_expiring';

export interface WebhookSubscription {
  id: string;
  organizationId: string;
  url: string;
  events: WebhookEvent[];
  secret: string;              // for HMAC signature verification
  isActive: boolean;
}

// POST   /api/webhooks — create subscription
// GET    /api/webhooks — list subscriptions
// PATCH  /api/webhooks/:id — update subscription
// DELETE /api/webhooks/:id — remove subscription
// POST   /api/webhooks/:id/test — send test payload
```

**Testing:**
- `test_webhook_delivery_on_deal_status_change` — Deal status change triggers webhook
- `test_webhook_hmac_signature` — Payload includes valid HMAC-SHA256 signature
- `test_webhook_retry_on_failure` — Failed delivery retries with exponential backoff
- `test_webhook_test_endpoint` — Test delivery sends sample payload to subscriber URL
- `test_webhook_event_filtering` — Only delivers events matching subscription filter

### Task 12.3: MCP Server Endpoint

**What:** Implement a Model Context Protocol (MCP) server that exposes deal analysis, comps retrieval, and market data queries as tools for LLM-based agents.

**Design:**

```typescript
// apps/api/src/modules/mcp/mcp-server.ts
// Expose as MCP-compatible tool definitions:
// - analyze_deal: Run DCF analysis for given assumptions
// - get_comparables: Retrieve comparable transactions for a property
// - get_market_data: Query submarket metrics
// - create_deal: Create a new deal from structured input
// - generate_report: Generate a deal summary or LP quarterly report

export const mcpToolDefinitions = [
  {
    name: 'analyze_deal',
    description: 'Run DCF cash flow analysis and compute IRR, NPV, equity multiple for a real estate deal',
    inputSchema: {
      type: 'object',
      properties: {
        propertyAddress: { type: 'string' },
        purchasePrice: { type: 'number' },
        holdPeriodYears: { type: 'number' },
        // ... full schema
      },
    },
  },
  // ... other tools
];
```

**Testing:**
- `test_mcp_tool_definitions_valid` — All tool definitions conform to MCP spec
- `test_mcp_analyze_deal_returns_metrics` — analyze_deal tool returns valid DCF results
- `test_mcp_get_comparables_returns_data` — get_comparables tool returns comp transactions
- `test_mcp_auth_required` — MCP endpoints require authentication

---

## Phase Dependency Graph

```
Phase 1: Foundation & Database
    │
    ├── Phase 2: Auth & Multi-Tenancy
    │       │
    │       ├── Phase 3: Property Management & Data Import
    │       │       │
    │       │       ├── Phase 5: Deal Underwriting API & UI ──────────┐
    │       │       │       │                                         │
    │       │       │       ├── Phase 7: Report Generation            │
    │       │       │       │       │                                  │
    │       │       │       │       └── Phase 11: LP Portal ──────────┤
    │       │       │       │                                         │
    │       │       │       ├── Phase 8: Investor & Syndication ──────┤
    │       │       │       │       │                                  │
    │       │       │       │       └── Phase 10: Portfolio Dashboard ─┤
    │       │       │       │                                          │
    │       │       │       └── Phase 9: AI Assumption Engine ─────────┤
    │       │       │                                                  │
    │       │       └── Phase 6: Market Data Integration ──────────────┘
    │       │                                                          │
    │       └──────────────────────────────────────────────────────────│
    │                                                                  │
    ├── Phase 4: Financial Calculation Engine (independent) ───────────┘
    │                                                                  │
    └──────────────────────────────────────────────────────────────────│
                                                                       │
                                               Phase 12: API & Integrations
```

---

## Definition of Done Checklist

A phase is considered complete when ALL of the following criteria are met:

- [ ] **All tasks implemented** — Every task in the phase has been coded and merged to main
- [ ] **Unit tests passing** — All named test scenarios pass with >90% line coverage for the phase's code
- [ ] **Integration tests passing** — API endpoints return correct responses for valid and invalid inputs
- [ ] **E2E tests passing** (UI phases) — Playwright tests pass for critical user flows
- [ ] **TypeScript strict mode clean** — No type errors across the monorepo
- [ ] **Lint clean** — Zero ESLint errors or warnings
- [ ] **Database migrations tested** — Up and down migrations work on a fresh database
- [ ] **API documentation updated** — OpenAPI spec includes all new endpoints
- [ ] **Audit logging verified** — All write operations generate audit log entries
- [ ] **Multi-tenant isolation verified** — No cross-tenant data leakage in new endpoints
- [ ] **Performance baseline met** — API response times <200ms for list endpoints, <500ms for computation endpoints
- [ ] **Security review passed** — No OWASP Top 10 vulnerabilities in new code
- [ ] **Code reviewed** — All PRs reviewed by at least one other developer
- [ ] **Environment variables documented** — Any new env vars added to .env.example with descriptions

---

## Estimated Scope Summary

| Phase | Name | Tasks | Estimated Effort |
|-------|------|-------|-----------------|
| 1 | Foundation & Database | 3 | 1-2 weeks |
| 2 | Auth & Multi-Tenancy | 3 | 1-2 weeks |
| 3 | Property Management & Data Import | 4 | 2-3 weeks |
| 4 | Financial Calculation Engine | 4 | 2-3 weeks |
| 5 | Deal Underwriting API & UI | 3 | 2-3 weeks |
| 6 | Market Data Integration | 3 | 1-2 weeks |
| 7 | Report Generation | 2 | 1-2 weeks |
| 8 | Investor & Syndication | 4 | 2-3 weeks |
| 9 | AI Assumption Engine | 3 | 2-3 weeks |
| 10 | Portfolio Dashboard | 2 | 1-2 weeks |
| 11 | LP Portal & Quarterly Reports | 2 | 1-2 weeks |
| 12 | API & Integrations | 3 | 1-2 weeks |
| **Total** | | **36** | **17-29 weeks** |

**Notes:**
- Phases 1-5 constitute the MVP (core deal underwriting)
- Phases 4 and 6 can be developed in parallel with other phases
- Phase 9 (AI) depends on Phase 6 (market data) for assumption sourcing
- Phase 12 should begin once the core API surface stabilizes after Phase 10
