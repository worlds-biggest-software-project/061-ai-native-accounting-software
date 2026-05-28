# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: AI-Native Accounting Software · Created: 2026-05-12

## Philosophy

This model follows the classical normalized relational approach: every domain concept gets its own table, relationships are expressed through foreign keys, and referential integrity is enforced at the database level. The chart of accounts, journal entries, invoices, bank transactions, and revenue recognition schedules each have dedicated tables with explicit column definitions. This is the architecture used by LedgerSMB (PostgreSQL-backed), GnuCash (SQLite mode), and the conceptual structure behind the ISO 21378 Audit Data Collection standard, which defines 71 tables across eight modules (GL, AR, AP, SAL, PUR, INV, PPE, BAS).

The normalized approach treats the database as the single source of truth and the enforcer of business rules. Double-entry integrity is guaranteed by database constraints (every journal entry must balance to zero). Audit trails are maintained through separate audit tables rather than event replay. AI features operate as consumers of the relational data — reading structured tables to categorise transactions, generate entries, and answer natural-language queries — rather than being embedded in the storage layer itself.

This architecture maps directly to how accountants think: accounts, periods, journals, ledgers. It is the easiest model for accounting professionals to audit, the most compatible with existing reporting tools (any SQL-based BI tool), and the most straightforward to export to XBRL or integrate with standards like ISO 21378.

**Best for:** Teams with strong SQL expertise building a compliance-first platform where auditability, regulatory reporting (XBRL, GAAP), and integration with existing accounting ecosystems are the top priorities.

**Trade-offs:**
- (+) Maximum data integrity via foreign keys and CHECK constraints
- (+) Direct mapping to ISO 21378 and GAAP chart of accounts structure
- (+) Easy to query with standard SQL; compatible with any BI tool
- (+) Simplest mental model for accountants and auditors
- (-) High table count increases migration complexity
- (-) Schema changes require migrations; adding jurisdiction-specific fields means ALTER TABLE
- (-) No built-in temporal querying ("what was the balance on date X?") without additional patterns
- (-) Audit trail is a separate concern, not inherent in the storage model

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 21378:2019 (Audit Data Collection) | Table structure mirrors the 8-module breakdown: GL, AR, AP, SAL, PUR, INV, PPE, BAS. Column naming conventions follow ISO field definitions where applicable. |
| GAAP / IFRS | Chart of accounts hierarchy enforces GAAP account classification (Assets, Liabilities, Equity, Revenue, Expenses). Financial statement generation queries the normalized tables directly. |
| ASC 606 / IFRS 15 | Dedicated `revenue_contract`, `performance_obligation`, and `revenue_schedule` tables model the five-step revenue recognition framework. |
| XBRL 2026 US GAAP Taxonomy | Account taxonomy codes stored in `account.xbrl_element` enable direct mapping to XBRL elements for SEC filing output. |
| PEPPOL BIS Billing 3.0 / UBL 2.1 | Invoice tables include fields for PEPPOL endpoint IDs, UBL document references, and tax category codes per EN 16931. |
| ISO 4217 | Currency codes stored as CHAR(3) conforming to ISO 4217. |
| ISO 3166-1 | Country codes for addresses and jurisdiction references. |
| FDX v6.5 / Plaid | Bank feed import tables map Plaid transaction fields (merchant, category, counterparty) to the normalized transaction model. |
| OAuth 2.0 / OpenID Connect | User authentication tables support OAuth tokens and OIDC identity claims for audit trail attribution. |

---

## Organisation & Multi-Tenancy

```sql
-- ============================================================
-- ORGANISATION & MULTI-TENANCY
-- ============================================================

CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    legal_name      TEXT,
    tax_id          TEXT,                          -- EIN, VAT number, etc.
    country_code    CHAR(2) NOT NULL,              -- ISO 3166-1 alpha-2
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD', -- ISO 4217
    fiscal_year_end_month SMALLINT NOT NULL DEFAULT 12,  -- 1-12
    peppol_endpoint_id TEXT,                       -- PEPPOL participant ID
    settings        JSONB NOT NULL DEFAULT '{}',   -- org-level config
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organisation_country ON organisation(country_code);

-- Row-level security policy for multi-tenancy
ALTER TABLE organisation ENABLE ROW LEVEL SECURITY;

CREATE TABLE org_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    user_id         UUID NOT NULL,                 -- from auth provider (OIDC sub claim)
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    role            TEXT NOT NULL CHECK (role IN ('owner', 'admin', 'accountant', 'bookkeeper', 'viewer')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    invited_at      TIMESTAMPTZ,
    accepted_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, user_id)
);

CREATE INDEX idx_org_user_org ON org_user(organisation_id);
CREATE INDEX idx_org_user_user ON org_user(user_id);
```

## Chart of Accounts

```sql
-- ============================================================
-- CHART OF ACCOUNTS (GAAP-COMPLIANT HIERARCHY)
-- ============================================================

CREATE TABLE account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    code            TEXT NOT NULL,                  -- e.g., '1000', '1010'
    name            TEXT NOT NULL,                  -- e.g., 'Cash and Cash Equivalents'
    account_type    TEXT NOT NULL CHECK (account_type IN (
        'asset', 'liability', 'equity', 'revenue', 'expense',
        'contra_asset', 'contra_liability', 'contra_equity',
        'contra_revenue', 'contra_expense'
    )),
    account_subtype TEXT,                           -- e.g., 'current_asset', 'long_term_liability'
    parent_id       UUID REFERENCES account(id),   -- hierarchy via adjacency list
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD', -- ISO 4217
    is_active       BOOLEAN NOT NULL DEFAULT true,
    is_system       BOOLEAN NOT NULL DEFAULT false, -- protected system accounts
    description     TEXT,
    xbrl_element    TEXT,                           -- XBRL US GAAP taxonomy element name
    normal_balance  TEXT NOT NULL CHECK (normal_balance IN ('debit', 'credit')),
    tax_code        TEXT,                           -- jurisdiction-specific tax code
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, code)
);

CREATE INDEX idx_account_org ON account(organisation_id);
CREATE INDEX idx_account_parent ON account(parent_id);
CREATE INDEX idx_account_type ON account(organisation_id, account_type);

-- Materialised path for fast hierarchy queries (alternative to recursive CTE)
-- Stored as ltree for PostgreSQL ltree extension
-- Example: 'assets.current.cash'
ALTER TABLE account ADD COLUMN path TEXT;
CREATE INDEX idx_account_path ON account USING gist (path gist_trgm_ops);
```

## Fiscal Periods

```sql
-- ============================================================
-- FISCAL PERIODS
-- ============================================================

CREATE TABLE fiscal_year (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    year_label      TEXT NOT NULL,                  -- e.g., 'FY2026'
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    is_closed       BOOLEAN NOT NULL DEFAULT false,
    closed_at       TIMESTAMPTZ,
    closed_by       UUID REFERENCES org_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, year_label)
);

CREATE TABLE fiscal_period (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fiscal_year_id  UUID NOT NULL REFERENCES fiscal_year(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    period_number   SMALLINT NOT NULL,             -- 1-12 (or 1-13 for adjustment period)
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    is_closed       BOOLEAN NOT NULL DEFAULT false,
    is_adjustment   BOOLEAN NOT NULL DEFAULT false, -- year-end adjustment period
    closed_at       TIMESTAMPTZ,
    closed_by       UUID REFERENCES org_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (fiscal_year_id, period_number)
);

CREATE INDEX idx_fiscal_period_org ON fiscal_period(organisation_id);
CREATE INDEX idx_fiscal_period_dates ON fiscal_period(organisation_id, start_date, end_date);
```

## General Ledger — Journal Entries

```sql
-- ============================================================
-- GENERAL LEDGER — JOURNAL ENTRIES
-- ============================================================

CREATE TABLE journal_entry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    fiscal_period_id UUID NOT NULL REFERENCES fiscal_period(id),
    entry_number    BIGINT NOT NULL,               -- sequential within org
    entry_date      DATE NOT NULL,
    description     TEXT NOT NULL,
    source_type     TEXT NOT NULL CHECK (source_type IN (
        'manual', 'bank_feed', 'invoice', 'bill', 'payroll',
        'depreciation', 'accrual', 'adjustment', 'closing',
        'reversal', 'ai_generated'
    )),
    source_id       UUID,                          -- FK to originating document
    status          TEXT NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'pending_review', 'approved', 'posted', 'reversed'
    )),
    is_reversing    BOOLEAN NOT NULL DEFAULT false,
    reversal_of     UUID REFERENCES journal_entry(id),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    exchange_rate   NUMERIC(18,8),                 -- rate to base currency
    posted_at       TIMESTAMPTZ,
    posted_by       UUID REFERENCES org_user(id),
    created_by      UUID NOT NULL REFERENCES org_user(id),
    ai_confidence   NUMERIC(5,4),                  -- 0.0000-1.0000 for AI-generated entries
    ai_model_id     TEXT,                          -- model version that generated this entry
    ai_explanation  TEXT,                          -- LLM-generated explanation
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, entry_number)
);

CREATE INDEX idx_journal_entry_org_date ON journal_entry(organisation_id, entry_date);
CREATE INDEX idx_journal_entry_period ON journal_entry(fiscal_period_id);
CREATE INDEX idx_journal_entry_status ON journal_entry(organisation_id, status);
CREATE INDEX idx_journal_entry_source ON journal_entry(source_type, source_id);

CREATE TABLE journal_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    journal_entry_id UUID NOT NULL REFERENCES journal_entry(id) ON DELETE CASCADE,
    account_id      UUID NOT NULL REFERENCES account(id),
    line_number     SMALLINT NOT NULL,
    description     TEXT,
    debit_amount    NUMERIC(19,4) NOT NULL DEFAULT 0,
    credit_amount   NUMERIC(19,4) NOT NULL DEFAULT 0,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    base_debit      NUMERIC(19,4) NOT NULL DEFAULT 0,  -- in org base currency
    base_credit     NUMERIC(19,4) NOT NULL DEFAULT 0,
    department_id   UUID REFERENCES department(id),
    project_id      UUID REFERENCES project(id),
    contact_id      UUID REFERENCES contact(id),
    tax_code        TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT chk_debit_or_credit CHECK (
        (debit_amount > 0 AND credit_amount = 0) OR
        (credit_amount > 0 AND debit_amount = 0)
    )
);

CREATE INDEX idx_journal_line_entry ON journal_line(journal_entry_id);
CREATE INDEX idx_journal_line_account ON journal_line(account_id);
CREATE INDEX idx_journal_line_dept ON journal_line(department_id) WHERE department_id IS NOT NULL;

-- Constraint: every journal entry must balance (enforced via trigger)
-- SUM(debit_amount) = SUM(credit_amount) for all lines in a journal_entry
```

## Dimensions (Cost Centres)

```sql
-- ============================================================
-- DIMENSIONS (COST CENTRES, DEPARTMENTS, PROJECTS)
-- ============================================================

CREATE TABLE department (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    parent_id       UUID REFERENCES department(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, code)
);

CREATE TABLE project (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    start_date      DATE,
    end_date        DATE,
    budget_amount   NUMERIC(19,4),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, code)
);
```

## Contacts (Customers, Vendors, Employees)

```sql
-- ============================================================
-- CONTACTS (UNIFIED PARTY MODEL)
-- ============================================================

CREATE TABLE contact (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    contact_type    TEXT NOT NULL CHECK (contact_type IN (
        'customer', 'vendor', 'employee', 'other'
    )),
    name            TEXT NOT NULL,
    legal_name      TEXT,
    email           TEXT,
    phone           TEXT,
    tax_id          TEXT,                          -- customer/vendor tax ID
    currency_code   CHAR(3) DEFAULT 'USD',        -- preferred currency
    payment_terms_days SMALLINT DEFAULT 30,
    credit_limit    NUMERIC(19,4),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    peppol_endpoint_id TEXT,                       -- for e-invoicing
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contact_org_type ON contact(organisation_id, contact_type);
CREATE INDEX idx_contact_name ON contact(organisation_id, name);

CREATE TABLE contact_address (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contact_id      UUID NOT NULL REFERENCES contact(id) ON DELETE CASCADE,
    address_type    TEXT NOT NULL CHECK (address_type IN ('billing', 'shipping', 'registered')),
    line1           TEXT NOT NULL,
    line2           TEXT,
    city            TEXT NOT NULL,
    state_province  TEXT,
    postal_code     TEXT,
    country_code    CHAR(2) NOT NULL,              -- ISO 3166-1
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Invoicing & Accounts Receivable

```sql
-- ============================================================
-- INVOICING & ACCOUNTS RECEIVABLE
-- ============================================================

CREATE TABLE invoice (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    invoice_number  TEXT NOT NULL,
    invoice_type    TEXT NOT NULL CHECK (invoice_type IN ('sales', 'credit_note')),
    contact_id      UUID NOT NULL REFERENCES contact(id),
    status          TEXT NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'sent', 'viewed', 'partially_paid', 'paid', 'overdue', 'void'
    )),
    issue_date      DATE NOT NULL,
    due_date        DATE NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    exchange_rate   NUMERIC(18,8),
    subtotal        NUMERIC(19,4) NOT NULL DEFAULT 0,
    tax_total       NUMERIC(19,4) NOT NULL DEFAULT 0,
    total           NUMERIC(19,4) NOT NULL DEFAULT 0,
    amount_paid     NUMERIC(19,4) NOT NULL DEFAULT 0,
    amount_due      NUMERIC(19,4) NOT NULL DEFAULT 0,
    reference       TEXT,                          -- PO number, external reference
    notes           TEXT,
    -- PEPPOL / UBL fields
    ubl_document_id TEXT,                          -- UBL InvoiceDocumentReference
    peppol_profile  TEXT,                          -- e.g., 'urn:fdc:peppol.eu:2017:poacc:billing:01:1.0'
    -- Journal linkage
    journal_entry_id UUID REFERENCES journal_entry(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, invoice_number)
);

CREATE INDEX idx_invoice_org_status ON invoice(organisation_id, status);
CREATE INDEX idx_invoice_contact ON invoice(contact_id);
CREATE INDEX idx_invoice_due ON invoice(organisation_id, due_date) WHERE status NOT IN ('paid', 'void');

CREATE TABLE invoice_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoice(id) ON DELETE CASCADE,
    line_number     SMALLINT NOT NULL,
    description     TEXT NOT NULL,
    account_id      UUID NOT NULL REFERENCES account(id),
    quantity        NUMERIC(19,4) NOT NULL DEFAULT 1,
    unit_price      NUMERIC(19,4) NOT NULL,
    discount_pct    NUMERIC(5,2) DEFAULT 0,
    line_total      NUMERIC(19,4) NOT NULL,
    tax_rate_id     UUID REFERENCES tax_rate(id),
    tax_amount      NUMERIC(19,4) NOT NULL DEFAULT 0,
    item_id         UUID REFERENCES item(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE payment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    payment_type    TEXT NOT NULL CHECK (payment_type IN ('received', 'made')),
    contact_id      UUID NOT NULL REFERENCES contact(id),
    payment_date    DATE NOT NULL,
    amount          NUMERIC(19,4) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    payment_method  TEXT CHECK (payment_method IN (
        'bank_transfer', 'credit_card', 'cash', 'cheque', 'direct_debit', 'other'
    )),
    reference       TEXT,
    bank_account_id UUID REFERENCES account(id),   -- cash/bank account
    journal_entry_id UUID REFERENCES journal_entry(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE payment_allocation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_id      UUID NOT NULL REFERENCES payment(id) ON DELETE CASCADE,
    invoice_id      UUID NOT NULL REFERENCES invoice(id),
    amount          NUMERIC(19,4) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Bills & Accounts Payable

```sql
-- ============================================================
-- BILLS & ACCOUNTS PAYABLE
-- ============================================================

CREATE TABLE bill (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    bill_number     TEXT NOT NULL,
    contact_id      UUID NOT NULL REFERENCES contact(id),  -- vendor
    status          TEXT NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'awaiting_approval', 'approved', 'partially_paid', 'paid', 'void'
    )),
    issue_date      DATE NOT NULL,
    due_date        DATE NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    exchange_rate   NUMERIC(18,8),
    subtotal        NUMERIC(19,4) NOT NULL DEFAULT 0,
    tax_total       NUMERIC(19,4) NOT NULL DEFAULT 0,
    total           NUMERIC(19,4) NOT NULL DEFAULT 0,
    amount_paid     NUMERIC(19,4) NOT NULL DEFAULT 0,
    amount_due      NUMERIC(19,4) NOT NULL DEFAULT 0,
    vendor_invoice_ref TEXT,                       -- vendor's invoice number
    journal_entry_id UUID REFERENCES journal_entry(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, bill_number)
);

CREATE TABLE bill_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bill_id         UUID NOT NULL REFERENCES bill(id) ON DELETE CASCADE,
    line_number     SMALLINT NOT NULL,
    description     TEXT NOT NULL,
    account_id      UUID NOT NULL REFERENCES account(id),
    quantity        NUMERIC(19,4) NOT NULL DEFAULT 1,
    unit_price      NUMERIC(19,4) NOT NULL,
    line_total      NUMERIC(19,4) NOT NULL,
    tax_rate_id     UUID REFERENCES tax_rate(id),
    tax_amount      NUMERIC(19,4) NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Tax Configuration

```sql
-- ============================================================
-- TAX CONFIGURATION
-- ============================================================

CREATE TABLE tax_rate (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,                  -- e.g., 'Standard VAT', 'Sales Tax - CA'
    rate            NUMERIC(7,4) NOT NULL,          -- e.g., 20.0000 for 20%
    tax_type        TEXT NOT NULL CHECK (tax_type IN (
        'sales_tax', 'vat', 'gst', 'withholding', 'exempt', 'zero_rated'
    )),
    jurisdiction    TEXT,                           -- e.g., 'US-CA', 'GB', 'DE'
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- PEPPOL tax category code (EN 16931)
    peppol_tax_category TEXT,                       -- e.g., 'S' (standard), 'Z' (zero-rated), 'E' (exempt)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Bank Feeds & Reconciliation

```sql
-- ============================================================
-- BANK FEEDS & RECONCILIATION
-- ============================================================

CREATE TABLE bank_connection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    provider        TEXT NOT NULL CHECK (provider IN ('plaid', 'open_banking', 'manual')),
    provider_item_id TEXT,                          -- Plaid item_id
    institution_name TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
        'active', 'requires_reauth', 'disconnected', 'error'
    )),
    consent_expires_at TIMESTAMPTZ,                -- Open Banking consent expiry
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE bank_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bank_connection_id UUID NOT NULL REFERENCES bank_connection(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    provider_account_id TEXT NOT NULL,              -- Plaid account_id
    account_name    TEXT NOT NULL,
    account_type    TEXT,                           -- checking, savings, credit
    mask            TEXT,                           -- last 4 digits
    currency_code   CHAR(3) NOT NULL,
    ledger_account_id UUID REFERENCES account(id), -- mapped GL account
    current_balance NUMERIC(19,4),
    available_balance NUMERIC(19,4),
    balance_updated_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE bank_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bank_account_id UUID NOT NULL REFERENCES bank_account(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    provider_txn_id TEXT NOT NULL,                  -- Plaid transaction_id
    transaction_date DATE NOT NULL,
    posted_date     DATE,
    amount          NUMERIC(19,4) NOT NULL,         -- positive = inflow, negative = outflow
    currency_code   CHAR(3) NOT NULL,
    description     TEXT NOT NULL,                  -- raw bank description
    merchant_name   TEXT,                           -- Plaid extracted merchant
    -- Plaid Personal Finance Category
    plaid_category_primary TEXT,
    plaid_category_detailed TEXT,
    plaid_confidence TEXT,
    -- AI categorisation
    ai_suggested_account_id UUID REFERENCES account(id),
    ai_confidence   NUMERIC(5,4),
    ai_categorised_at TIMESTAMPTZ,
    -- Reconciliation
    reconciliation_status TEXT NOT NULL DEFAULT 'unmatched' CHECK (reconciliation_status IN (
        'unmatched', 'ai_suggested', 'matched', 'reconciled', 'excluded'
    )),
    matched_journal_entry_id UUID REFERENCES journal_entry(id),
    reconciled_at   TIMESTAMPTZ,
    reconciled_by   UUID REFERENCES org_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (bank_account_id, provider_txn_id)
);

CREATE INDEX idx_bank_txn_org_date ON bank_transaction(organisation_id, transaction_date);
CREATE INDEX idx_bank_txn_status ON bank_transaction(organisation_id, reconciliation_status);
CREATE INDEX idx_bank_txn_account ON bank_transaction(bank_account_id);
```

## Revenue Recognition (ASC 606 / IFRS 15)

```sql
-- ============================================================
-- REVENUE RECOGNITION (ASC 606 / IFRS 15)
-- ============================================================

CREATE TABLE revenue_contract (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    contact_id      UUID NOT NULL REFERENCES contact(id),
    contract_number TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
        'draft', 'active', 'modified', 'completed', 'cancelled'
    )),
    signed_at       TIMESTAMPTZ,
    start_date      DATE NOT NULL,
    end_date        DATE,
    total_value     NUMERIC(19,4) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    standard        TEXT NOT NULL DEFAULT 'asc_606' CHECK (standard IN ('asc_606', 'ifrs_15')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, contract_number)
);

CREATE TABLE performance_obligation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    revenue_contract_id UUID NOT NULL REFERENCES revenue_contract(id) ON DELETE CASCADE,
    description     TEXT NOT NULL,
    obligation_type TEXT NOT NULL CHECK (obligation_type IN (
        'point_in_time', 'over_time'
    )),
    standalone_selling_price NUMERIC(19,4) NOT NULL,
    allocated_price NUMERIC(19,4) NOT NULL,        -- allocated from total contract value
    satisfaction_method TEXT CHECK (satisfaction_method IN (
        'output', 'input', 'straight_line'
    )),
    start_date      DATE NOT NULL,
    end_date        DATE,
    is_satisfied    BOOLEAN NOT NULL DEFAULT false,
    satisfied_at    TIMESTAMPTZ,
    revenue_account_id UUID NOT NULL REFERENCES account(id),
    deferred_revenue_account_id UUID NOT NULL REFERENCES account(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE revenue_schedule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    performance_obligation_id UUID NOT NULL REFERENCES performance_obligation(id) ON DELETE CASCADE,
    fiscal_period_id UUID NOT NULL REFERENCES fiscal_period(id),
    recognition_date DATE NOT NULL,
    amount          NUMERIC(19,4) NOT NULL,
    is_recognised   BOOLEAN NOT NULL DEFAULT false,
    journal_entry_id UUID REFERENCES journal_entry(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rev_schedule_period ON revenue_schedule(fiscal_period_id);
CREATE INDEX idx_rev_schedule_recognised ON revenue_schedule(is_recognised) WHERE NOT is_recognised;
```

## Items (Products / Services)

```sql
-- ============================================================
-- ITEMS (PRODUCTS / SERVICES)
-- ============================================================

CREATE TABLE item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    item_type       TEXT NOT NULL CHECK (item_type IN ('product', 'service')),
    description     TEXT,
    unit_price      NUMERIC(19,4),
    income_account_id UUID REFERENCES account(id),
    expense_account_id UUID REFERENCES account(id),
    tax_rate_id     UUID REFERENCES tax_rate(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, code)
);
```

## Startup Metrics

```sql
-- ============================================================
-- STARTUP METRICS (DERIVED & CACHED)
-- ============================================================

CREATE TABLE metric_snapshot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    snapshot_date   DATE NOT NULL,
    metric_type     TEXT NOT NULL CHECK (metric_type IN (
        'arr', 'mrr', 'burn_rate', 'runway_months',
        'headcount_cost', 'gross_margin', 'net_revenue_retention',
        'customer_count', 'arpu'
    )),
    value           NUMERIC(19,4) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    computation_method TEXT NOT NULL DEFAULT 'system', -- 'system' or 'manual'
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, snapshot_date, metric_type)
);

CREATE INDEX idx_metric_snapshot_org_type ON metric_snapshot(organisation_id, metric_type, snapshot_date);
```

## Exchange Rates

```sql
-- ============================================================
-- EXCHANGE RATES
-- ============================================================

CREATE TABLE exchange_rate (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    from_currency   CHAR(3) NOT NULL,              -- ISO 4217
    to_currency     CHAR(3) NOT NULL,              -- ISO 4217
    rate            NUMERIC(18,8) NOT NULL,
    effective_date  DATE NOT NULL,
    source          TEXT DEFAULT 'manual',          -- 'manual', 'ecb', 'openexchangerates'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, from_currency, to_currency, effective_date)
);

CREATE INDEX idx_exchange_rate_lookup ON exchange_rate(organisation_id, from_currency, to_currency, effective_date);
```

## Audit Trail

```sql
-- ============================================================
-- AUDIT TRAIL
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    user_id         UUID REFERENCES org_user(id),
    action          TEXT NOT NULL CHECK (action IN (
        'create', 'update', 'delete', 'post', 'approve', 'reverse', 'reconcile',
        'login', 'export', 'ai_categorise', 'ai_generate'
    )),
    entity_type     TEXT NOT NULL,                  -- 'journal_entry', 'invoice', etc.
    entity_id       UUID NOT NULL,
    changes         JSONB,                         -- {field: {old: x, new: y}}
    ip_address      INET,
    user_agent      TEXT,
    ai_model_id     TEXT,                          -- if action was AI-driven
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Partitioned by month for performance
CREATE INDEX idx_audit_log_org_date ON audit_log(organisation_id, created_at);
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_user ON audit_log(user_id);
```

## AI Categorisation Rules

```sql
-- ============================================================
-- AI CATEGORISATION RULES & FEEDBACK
-- ============================================================

CREATE TABLE categorisation_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    rule_type       TEXT NOT NULL CHECK (rule_type IN ('user_defined', 'ai_learned')),
    match_field     TEXT NOT NULL CHECK (match_field IN (
        'merchant_name', 'description', 'amount_range', 'plaid_category'
    )),
    match_pattern   TEXT NOT NULL,                  -- regex or exact match
    target_account_id UUID NOT NULL REFERENCES account(id),
    priority        SMALLINT NOT NULL DEFAULT 100,
    hit_count       INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ai_feedback (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    bank_transaction_id UUID REFERENCES bank_transaction(id),
    journal_entry_id UUID REFERENCES journal_entry(id),
    feedback_type   TEXT NOT NULL CHECK (feedback_type IN (
        'accepted', 'corrected', 'rejected'
    )),
    ai_suggested_account_id UUID REFERENCES account(id),
    correct_account_id UUID REFERENCES account(id),
    user_id         UUID NOT NULL REFERENCES org_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_feedback_org ON ai_feedback(organisation_id, created_at);
```

## Attachments & Documents

```sql
-- ============================================================
-- ATTACHMENTS & DOCUMENTS
-- ============================================================

CREATE TABLE attachment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    entity_type     TEXT NOT NULL,                  -- 'invoice', 'bill', 'journal_entry'
    entity_id       UUID NOT NULL,
    file_name       TEXT NOT NULL,
    file_size       BIGINT NOT NULL,
    mime_type       TEXT NOT NULL,
    storage_key     TEXT NOT NULL,                  -- S3 key or file path
    uploaded_by     UUID NOT NULL REFERENCES org_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_attachment_entity ON attachment(entity_type, entity_id);
```

## Example Queries

```sql
-- Trial Balance for a period
SELECT
    a.code,
    a.name,
    a.account_type,
    COALESCE(SUM(jl.debit_amount), 0) AS total_debits,
    COALESCE(SUM(jl.credit_amount), 0) AS total_credits,
    COALESCE(SUM(jl.debit_amount), 0) - COALESCE(SUM(jl.credit_amount), 0) AS net_balance
FROM account a
LEFT JOIN journal_line jl ON jl.account_id = a.id
LEFT JOIN journal_entry je ON je.id = jl.journal_entry_id
    AND je.status = 'posted'
    AND je.fiscal_period_id = '<period_id>'
WHERE a.organisation_id = '<org_id>'
GROUP BY a.id, a.code, a.name, a.account_type
ORDER BY a.code;

-- Chart of accounts hierarchy (recursive CTE)
WITH RECURSIVE account_tree AS (
    SELECT id, code, name, parent_id, 0 AS depth
    FROM account
    WHERE parent_id IS NULL AND organisation_id = '<org_id>'
    UNION ALL
    SELECT a.id, a.code, a.name, a.parent_id, at.depth + 1
    FROM account a
    JOIN account_tree at ON a.parent_id = at.id
)
SELECT * FROM account_tree ORDER BY code;

-- Burn rate calculation (last 3 months average operating expenses)
SELECT
    AVG(monthly_opex) AS avg_monthly_burn
FROM (
    SELECT
        fp.period_number,
        SUM(jl.debit_amount) - SUM(jl.credit_amount) AS monthly_opex
    FROM journal_line jl
    JOIN journal_entry je ON je.id = jl.journal_entry_id AND je.status = 'posted'
    JOIN fiscal_period fp ON fp.id = je.fiscal_period_id
    JOIN account a ON a.id = jl.account_id AND a.account_type = 'expense'
    WHERE je.organisation_id = '<org_id>'
    ORDER BY fp.start_date DESC
    LIMIT 3
) monthly;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Multi-Tenancy | 2 | `organisation`, `org_user` |
| Chart of Accounts | 1 | `account` with adjacency list hierarchy |
| Fiscal Periods | 2 | `fiscal_year`, `fiscal_period` |
| General Ledger | 2 | `journal_entry`, `journal_line` |
| Dimensions | 2 | `department`, `project` |
| Contacts | 2 | `contact`, `contact_address` |
| Invoicing & AR | 4 | `invoice`, `invoice_line`, `payment`, `payment_allocation` |
| Bills & AP | 2 | `bill`, `bill_line` |
| Tax | 1 | `tax_rate` |
| Bank Feeds | 3 | `bank_connection`, `bank_account`, `bank_transaction` |
| Revenue Recognition | 3 | `revenue_contract`, `performance_obligation`, `revenue_schedule` |
| Items | 1 | `item` |
| Startup Metrics | 1 | `metric_snapshot` |
| Exchange Rates | 1 | `exchange_rate` |
| Audit Trail | 1 | `audit_log` |
| AI Features | 2 | `categorisation_rule`, `ai_feedback` |
| Attachments | 1 | `attachment` |
| **Total** | **31** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation, safe for multi-region deployment, and prevents enumeration attacks on financial data.

2. **Adjacency list with optional materialised path for chart of accounts** — adjacency list is simple and handles account reparenting easily; materialised path (ltree) can be added for performance when hierarchy depth exceeds 6-8 levels.

3. **Separate debit_amount and credit_amount columns** — rather than a single signed amount. This matches how accountants think and makes the balance constraint (`debit = credit`) explicit and enforceable.

4. **Unified contact table with type discriminator** — customers, vendors, and employees share a common structure. A contact can be both a customer and a vendor (common in small businesses). This mirrors the Xero API model.

5. **AI columns co-located on transactional tables** — `ai_confidence`, `ai_model_id`, and `ai_explanation` live directly on `journal_entry` and `bank_transaction` rather than in separate tables. This keeps the AI context immediately visible during reconciliation and review.

6. **Separate audit_log table rather than event sourcing** — the audit trail is a secondary record of changes, not the source of truth. This is simpler to implement and query but means reconstructing historical state requires replaying audit entries rather than simply querying events.

7. **Revenue recognition as a first-class module** — three dedicated tables model the ASC 606 five-step framework. Performance obligations are tracked independently from billing, enabling correct recognition for multi-element arrangements.

8. **Plaid fields stored directly on bank_transaction** — rather than normalising Plaid categories into a separate taxonomy table. This keeps the import pipeline simple and preserves the provider's original categorisation alongside the AI re-categorisation.

9. **Fiscal period closure enforced in application logic** — the `is_closed` flag on `fiscal_period` prevents new postings via application-layer checks rather than database triggers, allowing flexibility for adjustment entries by authorised users.

10. **Row-level security for multi-tenancy** — PostgreSQL RLS policies on `organisation_id` provide defence-in-depth beyond application-layer filtering, critical for a platform handling multiple companies' financial data.
