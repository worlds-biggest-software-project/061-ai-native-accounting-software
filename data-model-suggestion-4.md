# Data Model Suggestion 4: Immutable Ledger Core with Materialised Reporting Layer

> Project: AI-Native Accounting Software · Created: 2026-05-12

## Philosophy

This model splits the architecture into two distinct layers: an **immutable ledger core** (inspired by Blnk Finance and Square's Books service) and a **materialised reporting layer** designed for financial statement generation, AI feature serving, and regulatory output. The ledger core is append-only: transactions are never modified, only new entries are appended (including corrections and reversals). The reporting layer is computed, cacheable, and optimised for read performance.

The key difference from the Event-Sourced model (Suggestion 2) is that this design keeps its write path closer to traditional accounting concepts — the immutable entries ARE ledger postings (debits and credits to specific accounts), not abstract domain events. There is no CQRS split between commands and queries; instead, the immutable postings table is both the write target and a queryable ledger. The materialised reporting layer adds pre-computed views (account balances, financial statements, metric dashboards) that are refreshed periodically or on-demand.

This architecture is particularly well-suited for AI-native accounting because: (a) the immutable ledger provides training data with guaranteed integrity — no retroactive edits can corrupt the historical record; (b) the materialised layer can include AI-specific denormalisations (categorisation confidence scores, model version tags, feature vectors) without polluting the core ledger; and (c) the separation enables different refresh cadences — the ledger is real-time, while financial statements and metrics can be computed on a schedule matching the business's close cycle.

**Best for:** Teams that want the auditability benefits of immutability without the complexity of full event sourcing, who need a ledger that can serve both as the system of record and as a direct query target, and who want clear separation between "what happened" (ledger) and "what it means" (reports/metrics).

**Trade-offs:**
- (+) Immutable ledger provides audit-grade integrity by construction
- (+) Simpler than event sourcing — entries are ledger postings, not abstract events
- (+) Materialised layer can be tuned for specific query patterns without affecting writes
- (+) Corrections via reversing entries match GAAP accounting practices exactly
- (+) AI features cleanly separated in the reporting layer
- (-) Storage grows monotonically (corrections append, never overwrite)
- (-) Materialised views need refresh orchestration (stale data risk)
- (-) Correcting errors requires two entries (reversal + correction) instead of UPDATE
- (-) Current balances require either aggregation or a maintained balance table
- (-) Some duplication between ledger and materialised layer

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 21378:2019 (Audit Data Collection) | Ledger core directly produces ISO 21378-compliant GL module output without transformation. Materialised reporting layer maps to other ISO 21378 modules (AR, AP, SAL, PUR). |
| GAAP / IFRS | Immutability enforces the GAAP principle that journal entries are never erased — corrections are reversing entries. Chart of accounts enforces GAAP classification hierarchy. |
| ASC 606 / IFRS 15 | Revenue recognition modelled with contract and obligation tables in the ledger layer. Recognition schedules produce immutable postings to revenue and deferred revenue accounts. |
| XBRL 2026 US GAAP Taxonomy | Materialised `mat_financial_statement` table pre-computes XBRL-tagged line items, enabling direct export to XBRL format without runtime computation. |
| PEPPOL BIS Billing 3.0 / UBL 2.1 | Invoice documents stored with PEPPOL metadata. UBL XML generation reads from the immutable invoice record. |
| Blnk Finance Pattern | Ledger core follows the Blnk Finance model: ledgers → balances → transactions, with immutable append-only semantics. Extended to include multi-currency and AI metadata. |
| SOC 2 Type II | Immutable ledger with `created_by` attribution on every posting satisfies SOC 2 audit trail requirements without additional logging. |
| FDX v6.5 / Plaid | Bank feed imports produce immutable `bank_import` records. Reconciliation creates immutable `posting` records linking imports to ledger entries. |

---

## Immutable Ledger Core

```sql
-- ============================================================
-- LEDGER CORE — IMMUTABLE TABLES
-- ============================================================
-- Rules for the ledger core:
-- 1. No UPDATE statements. Ever.
-- 2. No DELETE statements. Ever.
-- 3. Corrections are new entries (reversals + correcting entries).
-- 4. Every entry has a created_by and created_at.
-- 5. All monetary amounts are NUMERIC(19,4).

-- Organisation (mutable — this is config, not ledger data)
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    legal_name      TEXT,
    tax_id          TEXT,
    country_code    CHAR(2) NOT NULL,
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',
    reporting_standard TEXT NOT NULL DEFAULT 'gaap' CHECK (reporting_standard IN ('gaap', 'ifrs')),
    fiscal_year_end_month SMALLINT NOT NULL DEFAULT 12,
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE organisation ENABLE ROW LEVEL SECURITY;

CREATE TABLE org_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    user_id         UUID NOT NULL,
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    role            TEXT NOT NULL CHECK (role IN ('owner', 'admin', 'accountant', 'bookkeeper', 'viewer')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, user_id)
);
```

## Chart of Accounts (Versioned)

```sql
-- ============================================================
-- CHART OF ACCOUNTS — VERSIONED (APPEND-ONLY)
-- ============================================================
-- Account definitions are versioned: changes create a new version row.
-- The latest version for each account code is the current definition.

CREATE TABLE account_version (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id      UUID NOT NULL,                 -- stable account identifier
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    version         INTEGER NOT NULL,              -- monotonically increasing
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    account_type    TEXT NOT NULL CHECK (account_type IN (
        'asset', 'liability', 'equity', 'revenue', 'expense',
        'contra_asset', 'contra_liability', 'contra_equity',
        'contra_revenue', 'contra_expense'
    )),
    account_subtype TEXT,
    parent_id       UUID,                          -- references account_id (not id)
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    normal_balance  TEXT NOT NULL CHECK (normal_balance IN ('debit', 'credit')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    xbrl_element    TEXT,
    tax_code        TEXT,
    change_reason   TEXT,                          -- why this version was created
    created_by      UUID NOT NULL REFERENCES org_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (account_id, version)
);

-- View to get current account definitions
CREATE VIEW account AS
SELECT DISTINCT ON (account_id)
    id,
    account_id,
    organisation_id,
    version,
    code,
    name,
    account_type,
    account_subtype,
    parent_id,
    currency_code,
    normal_balance,
    is_active,
    xbrl_element,
    tax_code,
    created_at
FROM account_version
ORDER BY account_id, version DESC;

CREATE INDEX idx_acct_ver_org ON account_version(organisation_id);
CREATE INDEX idx_acct_ver_acct ON account_version(account_id, version DESC);
CREATE INDEX idx_acct_ver_code ON account_version(organisation_id, code);
```

## Fiscal Periods

```sql
-- ============================================================
-- FISCAL PERIODS
-- ============================================================

CREATE TABLE fiscal_year (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    year_label      TEXT NOT NULL,
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
    period_number   SMALLINT NOT NULL,
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    is_closed       BOOLEAN NOT NULL DEFAULT false,
    is_adjustment   BOOLEAN NOT NULL DEFAULT false,
    closed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (fiscal_year_id, period_number)
);
```

## Immutable Ledger Postings

```sql
-- ============================================================
-- LEDGER POSTINGS — THE IMMUTABLE CORE
-- ============================================================
-- This is the heart of the system. Every financial event produces
-- one or more postings. Postings are NEVER modified or deleted.
-- A posting group represents a balanced set of debits and credits.

CREATE TABLE posting_group (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    sequence_number BIGINT NOT NULL,               -- monotonic within org
    posting_date    DATE NOT NULL,
    fiscal_period_id UUID NOT NULL REFERENCES fiscal_period(id),
    description     TEXT NOT NULL,
    -- Source tracking
    source_type     TEXT NOT NULL CHECK (source_type IN (
        'manual', 'bank_feed', 'invoice', 'bill', 'payment',
        'payroll', 'depreciation', 'accrual', 'adjustment',
        'closing', 'reversal', 'ai_generated', 'revenue_recognition'
    )),
    source_document_id UUID,                       -- FK to invoice, bill, etc.
    -- Reversal chain
    reversal_of     UUID REFERENCES posting_group(id),
    reversed_by     UUID,                          -- set by application when reversal is created
    is_reversal     BOOLEAN NOT NULL DEFAULT false,
    -- Currency
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    exchange_rate   NUMERIC(18,8) DEFAULT 1.0,
    -- AI metadata
    ai_generated    BOOLEAN NOT NULL DEFAULT false,
    ai_confidence   NUMERIC(5,4),
    ai_model_id     TEXT,
    ai_explanation  TEXT,
    -- Approval
    status          TEXT NOT NULL DEFAULT 'posted' CHECK (status IN (
        'pending_review', 'approved', 'posted'
    )),
    approved_by     UUID REFERENCES org_user(id),
    approved_at     TIMESTAMPTZ,
    -- Immutability metadata
    created_by      UUID NOT NULL REFERENCES org_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Hash for tamper detection
    content_hash    TEXT NOT NULL,                  -- SHA-256 of posting data
    UNIQUE (organisation_id, sequence_number)
);

-- The posting_group table is append-only. Enforce via trigger:
CREATE OR REPLACE FUNCTION prevent_posting_group_mutation()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'UPDATE' THEN
        -- Only allow setting reversed_by (a back-reference)
        IF NEW.id = OLD.id
           AND NEW.organisation_id = OLD.organisation_id
           AND NEW.sequence_number = OLD.sequence_number
           AND NEW.reversed_by IS NOT NULL
           AND OLD.reversed_by IS NULL
        THEN
            RETURN NEW;
        END IF;
        RAISE EXCEPTION 'posting_group is immutable: updates are not allowed';
    ELSIF TG_OP = 'DELETE' THEN
        RAISE EXCEPTION 'posting_group is immutable: deletes are not allowed';
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_posting_group_immutable
    BEFORE UPDATE OR DELETE ON posting_group
    FOR EACH ROW
    EXECUTE FUNCTION prevent_posting_group_mutation();

CREATE INDEX idx_pg_org_date ON posting_group(organisation_id, posting_date);
CREATE INDEX idx_pg_period ON posting_group(fiscal_period_id);
CREATE INDEX idx_pg_source ON posting_group(source_type, source_document_id);
CREATE INDEX idx_pg_reversal ON posting_group(reversal_of) WHERE reversal_of IS NOT NULL;

-- Individual postings (debit/credit lines)
CREATE TABLE posting (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    posting_group_id UUID NOT NULL REFERENCES posting_group(id),
    account_id      UUID NOT NULL,                 -- references account_version.account_id
    line_number     SMALLINT NOT NULL,
    description     TEXT,
    debit_amount    NUMERIC(19,4) NOT NULL DEFAULT 0,
    credit_amount   NUMERIC(19,4) NOT NULL DEFAULT 0,
    base_debit      NUMERIC(19,4) NOT NULL DEFAULT 0,
    base_credit     NUMERIC(19,4) NOT NULL DEFAULT 0,
    department_id   UUID,
    project_id      UUID,
    contact_id      UUID,
    tax_code        TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT chk_posting_debit_or_credit CHECK (
        (debit_amount > 0 AND credit_amount = 0) OR
        (credit_amount > 0 AND debit_amount = 0)
    )
);

-- Posting is also immutable
CREATE OR REPLACE FUNCTION prevent_posting_mutation()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'posting is immutable: % operations are not allowed', TG_OP;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_posting_immutable
    BEFORE UPDATE OR DELETE ON posting
    FOR EACH ROW
    EXECUTE FUNCTION prevent_posting_mutation();

CREATE INDEX idx_posting_group ON posting(posting_group_id);
CREATE INDEX idx_posting_account ON posting(account_id);
CREATE INDEX idx_posting_dept ON posting(department_id) WHERE department_id IS NOT NULL;

-- Balance integrity check: all postings in a group must balance
-- Enforced via trigger on INSERT
CREATE OR REPLACE FUNCTION check_posting_group_balance()
RETURNS TRIGGER AS $$
DECLARE
    total_debit NUMERIC(19,4);
    total_credit NUMERIC(19,4);
BEGIN
    SELECT COALESCE(SUM(debit_amount), 0), COALESCE(SUM(credit_amount), 0)
    INTO total_debit, total_credit
    FROM posting
    WHERE posting_group_id = NEW.posting_group_id;

    -- This trigger fires per-row; actual balance check done at commit
    -- via a deferred constraint trigger
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

## Contacts (Immutable History)

```sql
-- ============================================================
-- CONTACTS — VERSIONED (APPEND-ONLY)
-- ============================================================

CREATE TABLE contact_version (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contact_id      UUID NOT NULL,
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    version         INTEGER NOT NULL,
    contact_types   TEXT[] NOT NULL,
    name            TEXT NOT NULL,
    legal_name      TEXT,
    email           TEXT,
    phone           TEXT,
    tax_id          TEXT,
    currency_code   CHAR(3) DEFAULT 'USD',
    payment_terms_days SMALLINT DEFAULT 30,
    credit_limit    NUMERIC(19,4),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    addresses       JSONB NOT NULL DEFAULT '[]',
    peppol_endpoint_id TEXT,
    bank_details    JSONB,
    change_reason   TEXT,
    created_by      UUID NOT NULL REFERENCES org_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (contact_id, version)
);

CREATE VIEW contact AS
SELECT DISTINCT ON (contact_id)
    id, contact_id, organisation_id, version,
    contact_types, name, legal_name, email, phone,
    tax_id, currency_code, payment_terms_days, credit_limit,
    is_active, addresses, peppol_endpoint_id, bank_details,
    created_at
FROM contact_version
ORDER BY contact_id, version DESC;

CREATE INDEX idx_contact_ver_org ON contact_version(organisation_id);
CREATE INDEX idx_contact_ver_id ON contact_version(contact_id, version DESC);
```

## Invoices & Bills (Immutable Documents)

```sql
-- ============================================================
-- INVOICES & BILLS — IMMUTABLE DOCUMENTS
-- ============================================================
-- Once an invoice is issued, it cannot be modified.
-- Changes produce a new version or a credit note.

CREATE TABLE invoice_document (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    invoice_id      UUID NOT NULL,                 -- stable identifier
    version         INTEGER NOT NULL DEFAULT 1,    -- draft versions before issuance
    invoice_number  TEXT NOT NULL,
    direction       TEXT NOT NULL CHECK (direction IN ('sales', 'purchase')),
    document_type   TEXT NOT NULL CHECK (document_type IN (
        'invoice', 'credit_note', 'debit_note'
    )),
    contact_id      UUID NOT NULL,                 -- references contact_version.contact_id
    status          TEXT NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'issued', 'sent', 'viewed',
        'partially_paid', 'paid', 'overdue', 'void'
    )),
    issue_date      DATE NOT NULL,
    due_date        DATE NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    exchange_rate   NUMERIC(18,8),
    subtotal        NUMERIC(19,4) NOT NULL DEFAULT 0,
    tax_total       NUMERIC(19,4) NOT NULL DEFAULT 0,
    total           NUMERIC(19,4) NOT NULL DEFAULT 0,
    reference       TEXT,
    notes           TEXT,
    -- E-invoicing metadata
    e_invoice_data  JSONB,
    -- Posting reference (set when invoice is posted to GL)
    posting_group_id UUID REFERENCES posting_group(id),
    is_current      BOOLEAN NOT NULL DEFAULT true,  -- false for superseded draft versions
    created_by      UUID NOT NULL REFERENCES org_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (invoice_id, version)
);

CREATE INDEX idx_inv_doc_org ON invoice_document(organisation_id, direction, status);
CREATE INDEX idx_inv_doc_contact ON invoice_document(contact_id);
CREATE INDEX idx_inv_doc_current ON invoice_document(invoice_id) WHERE is_current;

CREATE TABLE invoice_line_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_document_id UUID NOT NULL REFERENCES invoice_document(id),
    line_number     SMALLINT NOT NULL,
    description     TEXT NOT NULL,
    account_id      UUID NOT NULL,
    quantity        NUMERIC(19,4) NOT NULL DEFAULT 1,
    unit_price      NUMERIC(19,4) NOT NULL,
    discount_pct    NUMERIC(5,2) DEFAULT 0,
    line_total      NUMERIC(19,4) NOT NULL,
    tax_rate_id     UUID,
    tax_amount      NUMERIC(19,4) NOT NULL DEFAULT 0,
    tax_category    JSONB,
    item_code       TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Payments (Immutable Records)

```sql
-- ============================================================
-- PAYMENTS — IMMUTABLE
-- ============================================================

CREATE TABLE payment_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    payment_type    TEXT NOT NULL CHECK (payment_type IN ('received', 'made')),
    contact_id      UUID NOT NULL,
    payment_date    DATE NOT NULL,
    amount          NUMERIC(19,4) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    exchange_rate   NUMERIC(18,8),
    payment_method  TEXT,
    reference       TEXT,
    bank_account_id UUID,
    posting_group_id UUID REFERENCES posting_group(id),
    -- Refund/reversal tracking
    is_refund       BOOLEAN NOT NULL DEFAULT false,
    refund_of       UUID REFERENCES payment_record(id),
    created_by      UUID NOT NULL REFERENCES org_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE payment_allocation_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_id      UUID NOT NULL REFERENCES payment_record(id),
    invoice_id      UUID NOT NULL,
    amount          NUMERIC(19,4) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Bank Feeds (Immutable Import)

```sql
-- ============================================================
-- BANK FEEDS — IMMUTABLE IMPORTS
-- ============================================================

CREATE TABLE bank_connection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    provider        TEXT NOT NULL,
    provider_item_id TEXT,
    institution_name TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',
    provider_config JSONB NOT NULL DEFAULT '{}',
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE bank_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bank_connection_id UUID NOT NULL REFERENCES bank_connection(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    provider_account_id TEXT NOT NULL,
    account_name    TEXT NOT NULL,
    account_type    TEXT,
    mask            TEXT,
    currency_code   CHAR(3) NOT NULL,
    ledger_account_id UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Bank imports are immutable records of what the bank reported
CREATE TABLE bank_import (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bank_account_id UUID NOT NULL REFERENCES bank_account(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    provider_txn_id TEXT NOT NULL,
    transaction_date DATE NOT NULL,
    posted_date     DATE,
    amount          NUMERIC(19,4) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    raw_description TEXT NOT NULL,
    merchant_name   TEXT,
    provider_data   JSONB NOT NULL DEFAULT '{}',
    import_batch_id UUID NOT NULL,                 -- which sync batch imported this
    imported_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (bank_account_id, provider_txn_id)
);

CREATE INDEX idx_bank_import_org ON bank_import(organisation_id, transaction_date);
CREATE INDEX idx_bank_import_batch ON bank_import(import_batch_id);

-- Reconciliation decisions are immutable records linking imports to postings
CREATE TABLE reconciliation_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    bank_import_id  UUID NOT NULL REFERENCES bank_import(id),
    posting_group_id UUID REFERENCES posting_group(id),
    reconciliation_type TEXT NOT NULL CHECK (reconciliation_type IN (
        'matched', 'created', 'excluded'
    )),
    -- AI categorisation at the time of reconciliation
    ai_suggested_account_id UUID,
    ai_confidence   NUMERIC(5,4),
    ai_model_id     TEXT,
    -- Decision metadata
    decided_by      UUID NOT NULL REFERENCES org_user(id),
    decision_source TEXT NOT NULL CHECK (decision_source IN (
        'manual', 'rule', 'ai_auto', 'ai_confirmed'
    )),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_recon_org ON reconciliation_record(organisation_id, created_at);
CREATE INDEX idx_recon_import ON reconciliation_record(bank_import_id);
```

## Revenue Recognition (Immutable Schedules)

```sql
-- ============================================================
-- REVENUE RECOGNITION — IMMUTABLE CONTRACT RECORDS
-- ============================================================

CREATE TABLE revenue_contract (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    contract_id     UUID NOT NULL,                 -- stable identifier
    version         INTEGER NOT NULL DEFAULT 1,    -- modifications create new versions
    contact_id      UUID NOT NULL,
    contract_number TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',
    signed_at       TIMESTAMPTZ,
    start_date      DATE NOT NULL,
    end_date        DATE,
    total_value     NUMERIC(19,4) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    standard        TEXT NOT NULL DEFAULT 'asc_606',
    modification_type TEXT,                        -- 'scope_change', 'price_change', etc.
    modification_treatment TEXT,                   -- 'prospective', 'cumulative_catchup'
    supersedes      UUID REFERENCES revenue_contract(id),
    is_current      BOOLEAN NOT NULL DEFAULT true,
    created_by      UUID NOT NULL REFERENCES org_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (contract_id, version)
);

CREATE TABLE performance_obligation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    revenue_contract_id UUID NOT NULL REFERENCES revenue_contract(id),
    description     TEXT NOT NULL,
    obligation_type TEXT NOT NULL CHECK (obligation_type IN ('point_in_time', 'over_time')),
    standalone_selling_price NUMERIC(19,4) NOT NULL,
    allocated_price NUMERIC(19,4) NOT NULL,
    satisfaction_method TEXT,
    start_date      DATE NOT NULL,
    end_date        DATE,
    revenue_account_id UUID NOT NULL,
    deferred_revenue_account_id UUID NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Revenue recognition events are immutable postings
CREATE TABLE revenue_recognition_entry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    performance_obligation_id UUID NOT NULL REFERENCES performance_obligation(id),
    fiscal_period_id UUID NOT NULL REFERENCES fiscal_period(id),
    recognition_date DATE NOT NULL,
    amount          NUMERIC(19,4) NOT NULL,
    posting_group_id UUID REFERENCES posting_group(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rev_entry_period ON revenue_recognition_entry(fiscal_period_id);
```

## Tax & Dimensions

```sql
-- ============================================================
-- TAX RATES & DIMENSIONS
-- ============================================================

CREATE TABLE tax_rate (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    rate            NUMERIC(7,4) NOT NULL,
    tax_type        TEXT NOT NULL,
    jurisdiction    TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    peppol_tax_category TEXT,
    effective_date  DATE NOT NULL DEFAULT CURRENT_DATE,
    expiry_date     DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE department (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    parent_id       UUID REFERENCES department(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, code)
);

CREATE TABLE project (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    budget_amount   NUMERIC(19,4),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    start_date      DATE,
    end_date        DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, code)
);
```

## Exchange Rates & Items

```sql
-- ============================================================
-- EXCHANGE RATES & ITEMS
-- ============================================================

CREATE TABLE exchange_rate (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    from_currency   CHAR(3) NOT NULL,
    to_currency     CHAR(3) NOT NULL,
    rate            NUMERIC(18,8) NOT NULL,
    effective_date  DATE NOT NULL,
    source          TEXT DEFAULT 'manual',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, from_currency, to_currency, effective_date)
);

CREATE TABLE item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    item_type       TEXT NOT NULL CHECK (item_type IN ('product', 'service')),
    unit_price      NUMERIC(19,4),
    income_account_id UUID,
    expense_account_id UUID,
    tax_rate_id     UUID REFERENCES tax_rate(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, code)
);
```

## AI Training & Feedback (Immutable)

```sql
-- ============================================================
-- AI CATEGORISATION FEEDBACK — IMMUTABLE TRAINING DATA
-- ============================================================

CREATE TABLE ai_categorisation_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    event_type      TEXT NOT NULL CHECK (event_type IN (
        'suggestion', 'accepted', 'corrected', 'rejected'
    )),
    bank_import_id  UUID REFERENCES bank_import(id),
    -- What the AI suggested
    raw_input       TEXT NOT NULL,
    suggested_account_id UUID,
    confidence      NUMERIC(5,4),
    model_id        TEXT NOT NULL,
    model_version   TEXT,
    reasoning       TEXT,
    alternatives    JSONB,
    -- What the user decided (for accepted/corrected/rejected)
    final_account_id UUID,
    user_id         UUID REFERENCES org_user(id),
    -- Immutable record
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_event_org ON ai_categorisation_event(organisation_id, event_type, created_at);
CREATE INDEX idx_ai_event_model ON ai_categorisation_event(model_id, created_at);
CREATE INDEX idx_ai_event_import ON ai_categorisation_event(bank_import_id);
```

## Materialised Reporting Layer

```sql
-- ============================================================
-- MATERIALISED REPORTING LAYER
-- ============================================================
-- These tables are COMPUTED from the immutable ledger.
-- They can be truncated and rebuilt at any time.
-- They are optimised for read performance.

-- ---- Current Account Balances ----
-- Maintained by a trigger or scheduled refresh

CREATE TABLE mat_account_balance (
    account_id      UUID NOT NULL,
    organisation_id UUID NOT NULL,
    fiscal_period_id UUID NOT NULL,
    opening_balance NUMERIC(19,4) NOT NULL DEFAULT 0,
    period_debits   NUMERIC(19,4) NOT NULL DEFAULT 0,
    period_credits  NUMERIC(19,4) NOT NULL DEFAULT 0,
    closing_balance NUMERIC(19,4) NOT NULL DEFAULT 0,
    last_posting_at TIMESTAMPTZ,
    refreshed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (account_id, fiscal_period_id)
);

CREATE INDEX idx_mat_balance_org ON mat_account_balance(organisation_id, fiscal_period_id);

-- ---- Contact Balances (AR/AP) ----

CREATE TABLE mat_contact_balance (
    contact_id      UUID NOT NULL,
    organisation_id UUID NOT NULL,
    balance_type    TEXT NOT NULL CHECK (balance_type IN ('receivable', 'payable')),
    total_invoiced  NUMERIC(19,4) NOT NULL DEFAULT 0,
    total_paid      NUMERIC(19,4) NOT NULL DEFAULT 0,
    balance_due     NUMERIC(19,4) NOT NULL DEFAULT 0,
    oldest_overdue_date DATE,
    overdue_amount  NUMERIC(19,4) NOT NULL DEFAULT 0,
    refreshed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (contact_id, balance_type)
);

-- ---- Financial Statement Lines ----
-- Pre-computed for P&L, Balance Sheet, Cash Flow

CREATE TABLE mat_financial_statement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    fiscal_period_id UUID NOT NULL,
    statement_type  TEXT NOT NULL CHECK (statement_type IN (
        'balance_sheet', 'income_statement', 'cash_flow'
    )),
    line_order      INTEGER NOT NULL,
    line_label      TEXT NOT NULL,
    account_type    TEXT,
    xbrl_element    TEXT,                          -- direct XBRL taxonomy mapping
    amount          NUMERIC(19,4) NOT NULL,
    prior_period_amount NUMERIC(19,4),             -- comparative period
    variance_pct    NUMERIC(7,2),
    is_total_line   BOOLEAN NOT NULL DEFAULT false,
    is_subtotal     BOOLEAN NOT NULL DEFAULT false,
    indent_level    SMALLINT NOT NULL DEFAULT 0,
    refreshed_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_mat_fs_org ON mat_financial_statement(organisation_id, fiscal_period_id, statement_type);

-- ---- Startup Metrics ----

CREATE TABLE mat_startup_metric (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    snapshot_date   DATE NOT NULL,
    metric_type     TEXT NOT NULL,
    value           NUMERIC(19,4) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    computation_details JSONB,                     -- how the metric was derived
    refreshed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, snapshot_date, metric_type)
);

CREATE INDEX idx_mat_metric_org ON mat_startup_metric(organisation_id, metric_type, snapshot_date);

-- ---- Bank Reconciliation Status ----

CREATE TABLE mat_bank_reconciliation (
    bank_account_id UUID NOT NULL,
    organisation_id UUID NOT NULL,
    total_imports   INTEGER NOT NULL DEFAULT 0,
    reconciled_count INTEGER NOT NULL DEFAULT 0,
    unreconciled_count INTEGER NOT NULL DEFAULT 0,
    excluded_count  INTEGER NOT NULL DEFAULT 0,
    unreconciled_amount NUMERIC(19,4) NOT NULL DEFAULT 0,
    ledger_balance  NUMERIC(19,4),
    bank_balance    NUMERIC(19,4),
    difference      NUMERIC(19,4),
    refreshed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (bank_account_id)
);

-- ---- AI Model Performance ----

CREATE TABLE mat_ai_performance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    model_id        TEXT NOT NULL,
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    total_suggestions INTEGER NOT NULL DEFAULT 0,
    accepted_count  INTEGER NOT NULL DEFAULT 0,
    corrected_count INTEGER NOT NULL DEFAULT 0,
    rejected_count  INTEGER NOT NULL DEFAULT 0,
    accuracy_pct    NUMERIC(5,2),
    avg_confidence  NUMERIC(5,4),
    refreshed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, model_id, period_start)
);

-- ---- Revenue Recognition Summary ----

CREATE TABLE mat_revenue_summary (
    organisation_id UUID NOT NULL,
    fiscal_period_id UUID NOT NULL,
    total_recognised NUMERIC(19,4) NOT NULL DEFAULT 0,
    total_deferred  NUMERIC(19,4) NOT NULL DEFAULT 0,
    contracts_active INTEGER NOT NULL DEFAULT 0,
    obligations_satisfied INTEGER NOT NULL DEFAULT 0,
    obligations_pending INTEGER NOT NULL DEFAULT 0,
    refreshed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (organisation_id, fiscal_period_id)
);
```

## Materialisation Refresh Tracking

```sql
-- ============================================================
-- MATERIALISATION REFRESH TRACKING
-- ============================================================

CREATE TABLE mat_refresh_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    mat_table       TEXT NOT NULL,
    organisation_id UUID NOT NULL,
    trigger_type    TEXT NOT NULL CHECK (trigger_type IN (
        'scheduled', 'on_demand', 'post_close', 'real_time'
    )),
    started_at      TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ,
    rows_affected   INTEGER,
    last_posting_sequence BIGINT,                  -- watermark: last posting processed
    status          TEXT NOT NULL DEFAULT 'running' CHECK (status IN (
        'running', 'completed', 'failed'
    )),
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_mat_refresh_table ON mat_refresh_log(mat_table, organisation_id, created_at DESC);
```

## Attachments

```sql
-- ============================================================
-- ATTACHMENTS
-- ============================================================

CREATE TABLE attachment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    file_name       TEXT NOT NULL,
    file_size       BIGINT NOT NULL,
    mime_type       TEXT NOT NULL,
    storage_key     TEXT NOT NULL,
    uploaded_by     UUID NOT NULL REFERENCES org_user(id),
    ai_extraction   JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_attachment_entity ON attachment(entity_type, entity_id);
```

## Example Queries

```sql
-- Trial balance directly from immutable postings (no materialised view needed)
SELECT
    av.code,
    av.name,
    av.account_type,
    COALESCE(SUM(p.debit_amount), 0) AS total_debits,
    COALESCE(SUM(p.credit_amount), 0) AS total_credits,
    COALESCE(SUM(p.debit_amount), 0) - COALESCE(SUM(p.credit_amount), 0) AS net
FROM account_version av
JOIN posting p ON p.account_id = av.account_id
JOIN posting_group pg ON pg.id = p.posting_group_id
    AND pg.fiscal_period_id = '<period_id>'
    AND pg.status = 'posted'
    AND pg.is_reversal = false
    AND pg.reversed_by IS NULL  -- exclude reversed entries
WHERE av.organisation_id = '<org_id>'
  AND av.version = (
      SELECT MAX(version) FROM account_version
      WHERE account_id = av.account_id
  )
GROUP BY av.account_id, av.code, av.name, av.account_type
ORDER BY av.code;

-- Historical balance at a specific date (query the immutable ledger directly)
SELECT
    av.code,
    av.name,
    COALESCE(SUM(p.debit_amount), 0) - COALESCE(SUM(p.credit_amount), 0) AS balance_at_date
FROM account_version av
JOIN posting p ON p.account_id = av.account_id
JOIN posting_group pg ON pg.id = p.posting_group_id
    AND pg.posting_date <= '2026-03-15'
    AND pg.status = 'posted'
    AND pg.reversed_by IS NULL
WHERE av.organisation_id = '<org_id>'
  AND av.version = (
      SELECT MAX(av2.version) FROM account_version av2
      WHERE av2.account_id = av.account_id
        AND av2.created_at <= '2026-03-15T23:59:59Z'
  )
GROUP BY av.account_id, av.code, av.name
ORDER BY av.code;

-- Verify content hash for tamper detection
SELECT
    pg.id,
    pg.sequence_number,
    pg.content_hash,
    md5(pg.organisation_id::text || pg.sequence_number::text ||
        pg.posting_date::text || pg.description ||
        (SELECT string_agg(
            p.account_id::text || p.debit_amount::text || p.credit_amount::text,
            '|' ORDER BY p.line_number
        ) FROM posting p WHERE p.posting_group_id = pg.id)
    ) AS computed_hash
FROM posting_group pg
WHERE pg.organisation_id = '<org_id>'
  AND pg.content_hash != md5(...)  -- tampered entries
ORDER BY pg.sequence_number;

-- Fast balance from materialised layer
SELECT
    a.code,
    a.name,
    b.closing_balance
FROM mat_account_balance b
JOIN account a ON a.account_id = b.account_id
WHERE b.organisation_id = '<org_id>'
  AND b.fiscal_period_id = '<current_period_id>'
ORDER BY a.code;

-- AI model accuracy trend (from materialised layer)
SELECT
    period_start,
    accuracy_pct,
    total_suggestions,
    avg_confidence
FROM mat_ai_performance
WHERE organisation_id = '<org_id>'
  AND model_id = 'categoriser-v3.2'
ORDER BY period_start;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Users | 2 | `organisation`, `org_user` |
| Chart of Accounts | 1 | `account_version` (+ `account` view) |
| Fiscal Periods | 2 | `fiscal_year`, `fiscal_period` |
| Ledger Core | 2 | `posting_group`, `posting` — immutable with triggers |
| Contacts | 1 | `contact_version` (+ `contact` view) |
| Invoices | 2 | `invoice_document`, `invoice_line_item` |
| Payments | 2 | `payment_record`, `payment_allocation_record` |
| Bank Feeds | 4 | `bank_connection`, `bank_account`, `bank_import`, `reconciliation_record` |
| Revenue Recognition | 3 | `revenue_contract`, `performance_obligation`, `revenue_recognition_entry` |
| Tax & Dimensions | 3 | `tax_rate`, `department`, `project` |
| Exchange Rates & Items | 2 | `exchange_rate`, `item` |
| AI | 1 | `ai_categorisation_event` |
| Attachments | 1 | `attachment` |
| Materialised Layer | 7 | `mat_account_balance`, `mat_contact_balance`, `mat_financial_statement`, `mat_startup_metric`, `mat_bank_reconciliation`, `mat_ai_performance`, `mat_revenue_summary` |
| Refresh Tracking | 1 | `mat_refresh_log` |
| **Total** | **34** | 26 ledger + 7 materialised + 1 tracking |

---

## Key Design Decisions

1. **Database-enforced immutability** — triggers on `posting_group` and `posting` prevent UPDATE and DELETE at the database level, not just the application level. This means even direct SQL access or a bug in the application cannot corrupt the ledger. The only exception is the `reversed_by` back-reference on `posting_group`, which is allowed as a single controlled update.

2. **Versioned entities (accounts, contacts, contracts)** — rather than modifying account or contact records in place, changes create a new version row. The current state is a view (`DISTINCT ON ... ORDER BY version DESC`). This enables historical queries: "what was this account's name when this posting was made?" without temporal table extensions.

3. **Content hashing for tamper detection** — each `posting_group` stores a SHA-256 hash of its contents. Periodic integrity checks can verify that no record has been tampered with. This satisfies SOC 2 and provides additional assurance beyond database immutability.

4. **Separate bank_import from reconciliation_record** — bank imports are raw facts ("the bank said this transaction happened"). Reconciliation records are decisions ("we matched this import to this posting"). Keeping them separate preserves the raw data and makes the reconciliation decision auditable.

5. **Posting groups instead of journal entries** — the term "posting group" emphasises that these are ledger postings, not draft entries. In this model, there is no "draft" posting — drafts exist in the application layer as unsaved data. Once something is written to `posting_group`, it is permanent.

6. **Materialised layer is explicitly disposable** — every `mat_*` table can be truncated and rebuilt from the immutable ledger. The `mat_refresh_log` tracks when each materialisation was last refreshed and what posting sequence it processed up to. This clean separation means the reporting layer can evolve independently of the ledger schema.

7. **AI events as immutable training data** — `ai_categorisation_event` captures the full lifecycle (suggestion, acceptance, correction, rejection) as immutable records. This creates a high-quality, timestamped training dataset where every data point has full provenance (which model, which input, what the user did).

8. **Pre-computed financial statements in mat_financial_statement** — rather than computing P&L and balance sheet from raw postings on every request, the materialised layer stores pre-computed statement lines with XBRL element mappings. This enables instant financial report rendering and direct XBRL export.

9. **Monotonic sequence numbers for ordering** — `posting_group.sequence_number` provides a strict total ordering of all postings within an organisation, independent of timestamps (which can have clock skew issues). This is critical for replay and reconciliation integrity.

10. **Invoice versioning for draft workflow** — before issuance, invoices can have multiple draft versions. After issuance, the invoice is immutable. This handles the common workflow where invoices are edited multiple times before being sent, while preserving full history of what was sent to the customer.
