# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: AI-Native Accounting Software · Created: 2026-05-12

## Philosophy

This model uses a pragmatic hybrid approach: core financial fields that are queried, aggregated, and constrained (amounts, dates, account codes, statuses) are stored in typed relational columns. Variable, jurisdiction-specific, and extensible data is stored in JSONB columns on the same tables. This avoids both the rigidity of a fully normalised schema (where adding a VAT-specific field requires a migration) and the query opacity of a fully document-oriented approach (where summing debits requires JSONB extraction).

PostgreSQL's JSONB support makes this viable: GIN indexes on JSONB columns enable fast containment queries, and JSONB operators allow extraction in SELECT/WHERE clauses without deserialisation. The pattern is used in practice by modern SaaS platforms that need to support multiple markets: Stripe's API objects have typed core fields and a `metadata` JSONB bag; Xero's Custom Fields API (December 2025) adds JSONB-style flexible metadata to their relational models.

For an AI-native accounting platform targeting global SMBs, this is particularly powerful. Different jurisdictions require different data: Germany needs ZUGFeRD/Factur-X XML references, France needs ChorusPro submission IDs, the US needs 1099 classification codes, and the UK needs MTD (Making Tax Digital) submission tokens. Rather than adding nullable columns for every jurisdiction or maintaining junction tables, a `jurisdiction_data` JSONB column on invoices and tax records holds jurisdiction-specific fields with JSONB Schema validation in the application layer.

**Best for:** Teams building a multi-jurisdiction platform that needs to ship an MVP quickly, support jurisdiction-specific data without schema migrations, and maintain query performance on core financial operations. Ideal when the development team is strong in PostgreSQL and wants a single-database deployment without separate event stores or graph databases.

**Trade-offs:**
- (+) Fast MVP: new fields via JSONB without schema migration
- (+) Multi-jurisdiction flexibility without nullable column sprawl
- (+) Core financial queries remain fast (typed columns, standard indexes)
- (+) Single database, single deployment model — simplest ops
- (+) JSONB columns serve as natural containers for AI feature data
- (-) JSONB fields lack database-level type constraints; validation is application-side
- (-) JSONB columns are harder to document and discover than typed columns
- (-) GIN index performance degrades on very large JSONB objects
- (-) Risk of "JSONB drift" where different orgs have different shapes in the same column
- (-) Harder to enforce referential integrity within JSONB data

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 21378:2019 (Audit Data Collection) | Core relational columns align with ISO 21378 field definitions. JSONB `extended_data` fields hold module-specific attributes beyond the ISO baseline. |
| GAAP / IFRS | Chart of accounts and journal entry columns enforce GAAP structure. `reporting_standard` field on organisation enables GAAP vs IFRS behavioural switches. |
| ASC 606 / IFRS 15 | Revenue contracts use typed columns for the five-step framework. Variable consideration and contract modification history stored in JSONB for flexibility across deal structures. |
| XBRL 2026 US GAAP Taxonomy | Account `xbrl_mapping` JSONB stores taxonomy element name, period type, and balance type — supporting multiple taxonomy versions without schema changes. |
| PEPPOL BIS Billing 3.0 / EN 16931 | Invoice `e_invoice_data` JSONB holds PEPPOL-specific fields (endpoint IDs, scheme identifiers, tax category codes per EN 16931) without polluting the core invoice schema. |
| ISO 4217 / ISO 3166 | Currency and country codes in typed CHAR columns for constraint enforcement. |
| FDX v6.5 / Plaid | Bank transaction `provider_data` JSONB preserves the full Plaid response (categories, counterparties, location, merchant logo URL) without needing to model every Plaid field relationally. |
| GDPR | `privacy_settings` JSONB on contact records holds consent flags, data processing basis, and erasure request tracking. |

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
    tax_id          TEXT,
    country_code    CHAR(2) NOT NULL,              -- ISO 3166-1
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD', -- ISO 4217
    reporting_standard TEXT NOT NULL DEFAULT 'gaap' CHECK (reporting_standard IN ('gaap', 'ifrs')),
    fiscal_year_end_month SMALLINT NOT NULL DEFAULT 12,
    -- Jurisdiction-specific org data
    jurisdiction_data JSONB NOT NULL DEFAULT '{}',
    -- Example jurisdiction_data:
    -- {
    --   "us": {"ein": "12-3456789", "state_registrations": ["CA", "NY", "TX"]},
    --   "gb": {"company_number": "12345678", "mtd_vat_registration": "GB123456789"},
    --   "de": {"handelsregister": "HRB 12345", "ust_id": "DE123456789"}
    -- }
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings:
    -- {
    --   "default_payment_terms_days": 30,
    --   "auto_categorisation_enabled": true,
    --   "ai_journal_entry_auto_post": false,
    --   "multi_currency_enabled": true,
    --   "peppol_enabled": false
    -- }
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
    permissions     JSONB NOT NULL DEFAULT '{}',
    -- Example permissions (role overrides):
    -- {
    --   "can_post_journal_entries": true,
    --   "can_approve_bills": false,
    --   "max_approval_amount": 10000,
    --   "restricted_departments": ["d-123", "d-456"]
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, user_id)
);

CREATE INDEX idx_org_user_org ON org_user(organisation_id);
```

## Chart of Accounts

```sql
-- ============================================================
-- CHART OF ACCOUNTS
-- ============================================================

CREATE TABLE account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    account_type    TEXT NOT NULL CHECK (account_type IN (
        'asset', 'liability', 'equity', 'revenue', 'expense',
        'contra_asset', 'contra_liability', 'contra_equity',
        'contra_revenue', 'contra_expense'
    )),
    account_subtype TEXT,
    parent_id       UUID REFERENCES account(id),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    normal_balance  TEXT NOT NULL CHECK (normal_balance IN ('debit', 'credit')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    is_system       BOOLEAN NOT NULL DEFAULT false,
    -- XBRL mapping as JSONB (supports multiple taxonomy versions)
    xbrl_mapping    JSONB,
    -- Example xbrl_mapping:
    -- {
    --   "us_gaap_2026": {
    --     "element": "us-gaap:CashAndCashEquivalentsAtCarryingValue",
    --     "period_type": "instant",
    --     "balance_type": "debit"
    --   },
    --   "ifrs_2026": {
    --     "element": "ifrs-full:CashAndCashEquivalents"
    --   }
    -- }
    -- Tax and jurisdiction-specific account config
    tax_config      JSONB NOT NULL DEFAULT '{}',
    -- Example tax_config:
    -- {
    --   "us_1099_category": "rents",
    --   "uk_vat_box": 6,
    --   "de_skr04_code": "1800"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, code)
);

CREATE INDEX idx_account_org ON account(organisation_id);
CREATE INDEX idx_account_parent ON account(parent_id);
CREATE INDEX idx_account_type ON account(organisation_id, account_type);
CREATE INDEX idx_account_xbrl ON account USING gin (xbrl_mapping);
```

## Journal Entries

```sql
-- ============================================================
-- GENERAL LEDGER — JOURNAL ENTRIES
-- ============================================================

CREATE TABLE journal_entry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    entry_number    BIGINT NOT NULL,
    entry_date      DATE NOT NULL,
    fiscal_period_id UUID NOT NULL,
    description     TEXT NOT NULL,
    source_type     TEXT NOT NULL CHECK (source_type IN (
        'manual', 'bank_feed', 'invoice', 'bill', 'payroll',
        'depreciation', 'accrual', 'adjustment', 'closing',
        'reversal', 'ai_generated', 'revenue_recognition'
    )),
    source_id       UUID,
    status          TEXT NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'pending_review', 'approved', 'posted', 'reversed'
    )),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    exchange_rate   NUMERIC(18,8),
    is_reversing    BOOLEAN NOT NULL DEFAULT false,
    reversal_of     UUID REFERENCES journal_entry(id),
    posted_at       TIMESTAMPTZ,
    posted_by       UUID REFERENCES org_user(id),
    created_by      UUID NOT NULL REFERENCES org_user(id),
    -- AI data embedded as JSONB — avoids separate AI tables for simple cases
    ai_data         JSONB,
    -- Example ai_data:
    -- {
    --   "confidence": 0.94,
    --   "model_id": "categoriser-v3.2",
    --   "model_version": "2026-05-01",
    --   "explanation": "Matched recurring pattern: monthly SaaS subscription",
    --   "alternatives": [
    --     {"account_code": "6200", "confidence": 0.04},
    --     {"account_code": "6300", "confidence": 0.02}
    --   ],
    --   "feedback": {
    --     "status": "accepted",
    --     "corrected_by": null,
    --     "corrected_at": null
    --   }
    -- }
    -- Extended data for jurisdiction/integration-specific fields
    extended_data   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, entry_number)
);

CREATE INDEX idx_je_org_date ON journal_entry(organisation_id, entry_date);
CREATE INDEX idx_je_status ON journal_entry(organisation_id, status);
CREATE INDEX idx_je_source ON journal_entry(source_type, source_id);
CREATE INDEX idx_je_ai ON journal_entry USING gin (ai_data) WHERE ai_data IS NOT NULL;

CREATE TABLE journal_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    journal_entry_id UUID NOT NULL REFERENCES journal_entry(id) ON DELETE CASCADE,
    account_id      UUID NOT NULL REFERENCES account(id),
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
    -- Line-level extended data
    tags            JSONB NOT NULL DEFAULT '[]',
    -- Example tags: ["recurring", "q2-accrual", "landlord-abc"]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT chk_debit_or_credit CHECK (
        (debit_amount > 0 AND credit_amount = 0) OR
        (credit_amount > 0 AND debit_amount = 0)
    )
);

CREATE INDEX idx_jl_entry ON journal_line(journal_entry_id);
CREATE INDEX idx_jl_account ON journal_line(account_id);
CREATE INDEX idx_jl_tags ON journal_line USING gin (tags);
```

## Contacts

```sql
-- ============================================================
-- CONTACTS (CUSTOMERS, VENDORS, EMPLOYEES)
-- ============================================================

CREATE TABLE contact (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    contact_types   TEXT[] NOT NULL,                -- {'customer', 'vendor'} — supports dual roles
    name            TEXT NOT NULL,
    legal_name      TEXT,
    email           TEXT,
    phone           TEXT,
    tax_id          TEXT,
    currency_code   CHAR(3) DEFAULT 'USD',
    payment_terms_days SMALLINT DEFAULT 30,
    credit_limit    NUMERIC(19,4),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- All addresses in JSONB array (avoids separate address table)
    addresses       JSONB NOT NULL DEFAULT '[]',
    -- Example addresses:
    -- [
    --   {
    --     "type": "billing",
    --     "line1": "123 Main St",
    --     "city": "San Francisco",
    --     "state": "CA",
    --     "postal_code": "94102",
    --     "country_code": "US",
    --     "is_primary": true
    --   },
    --   {
    --     "type": "shipping",
    --     "line1": "456 Warehouse Blvd",
    --     "city": "Oakland",
    --     "state": "CA",
    --     "postal_code": "94612",
    --     "country_code": "US",
    --     "is_primary": false
    --   }
    -- ]
    -- Jurisdiction-specific contact data
    jurisdiction_data JSONB NOT NULL DEFAULT '{}',
    -- Example jurisdiction_data:
    -- {
    --   "peppol_endpoint_id": "0088:1234567890",
    --   "peppol_scheme": "iso6523-actorid-upis",
    --   "gdpr_consent": {
    --     "processing_basis": "legitimate_interest",
    --     "consented_at": "2026-01-15T10:00:00Z",
    --     "erasure_requested": false
    --   }
    -- }
    -- Banking details for payment
    bank_details    JSONB,
    -- Example bank_details:
    -- {
    --   "account_name": "Acme Corp",
    --   "iban": "DE89370400440532013000",
    --   "bic": "COBADEFFXXX",
    --   "routing_number": null,
    --   "account_number": null
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contact_org ON contact(organisation_id);
CREATE INDEX idx_contact_types ON contact USING gin (contact_types);
CREATE INDEX idx_contact_name ON contact(organisation_id, name);
```

## Invoicing & Bills

```sql
-- ============================================================
-- INVOICING (SALES & PURCHASE)
-- ============================================================

CREATE TABLE invoice (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    invoice_number  TEXT NOT NULL,
    direction       TEXT NOT NULL CHECK (direction IN ('sales', 'purchase')),  -- replaces separate bill table
    document_type   TEXT NOT NULL CHECK (document_type IN (
        'invoice', 'credit_note', 'debit_note'
    )),
    contact_id      UUID NOT NULL REFERENCES contact(id),
    status          TEXT NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'awaiting_approval', 'approved', 'sent',
        'viewed', 'partially_paid', 'paid', 'overdue', 'void'
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
    reference       TEXT,
    notes           TEXT,
    journal_entry_id UUID REFERENCES journal_entry(id),
    -- E-invoicing data (PEPPOL, ZUGFeRD, Factur-X)
    e_invoice_data  JSONB,
    -- Example e_invoice_data:
    -- {
    --   "format": "peppol_bis_3",
    --   "profile_id": "urn:fdc:peppol.eu:2017:poacc:billing:01:1.0",
    --   "customization_id": "urn:cen.eu:en16931:2017#compliant#...",
    --   "buyer_reference": "PO-2026-0042",
    --   "document_reference": "INV-2026-0001",
    --   "delivery_date": "2026-05-15",
    --   "payment_means_code": "30",
    --   "submission_id": "chorus-pro-abc123",
    --   "zugferd_profile": "extended",
    --   "xml_blob_key": "s3://invoices/peppol/INV-2026-0001.xml"
    -- }
    -- Approval workflow
    approval_data   JSONB,
    -- Example approval_data:
    -- {
    --   "required_approvers": ["user-123", "user-456"],
    --   "approvals": [
    --     {"user_id": "user-123", "approved_at": "2026-05-10T14:30:00Z", "notes": "OK"}
    --   ],
    --   "auto_approved": false,
    --   "approval_rule": "any_one"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, invoice_number, direction)
);

CREATE INDEX idx_invoice_org_status ON invoice(organisation_id, status);
CREATE INDEX idx_invoice_contact ON invoice(contact_id);
CREATE INDEX idx_invoice_due ON invoice(organisation_id, due_date)
    WHERE status NOT IN ('paid', 'void');
CREATE INDEX idx_invoice_direction ON invoice(organisation_id, direction);
CREATE INDEX idx_invoice_einvoice ON invoice USING gin (e_invoice_data)
    WHERE e_invoice_data IS NOT NULL;

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
    tax_rate        NUMERIC(7,4),
    tax_amount      NUMERIC(19,4) NOT NULL DEFAULT 0,
    -- EN 16931 tax category per line (for PEPPOL compliance)
    tax_category    JSONB,
    -- Example tax_category:
    -- {
    --   "code": "S",
    --   "percent": 19.00,
    --   "scheme_id": "UN/ECE 5305"
    -- }
    item_code       TEXT,
    item_name       TEXT,
    extended_data   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Payments

```sql
-- ============================================================
-- PAYMENTS
-- ============================================================

CREATE TABLE payment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    payment_type    TEXT NOT NULL CHECK (payment_type IN ('received', 'made')),
    contact_id      UUID NOT NULL REFERENCES contact(id),
    payment_date    DATE NOT NULL,
    amount          NUMERIC(19,4) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    exchange_rate   NUMERIC(18,8),
    payment_method  TEXT,
    reference       TEXT,
    bank_account_id UUID REFERENCES account(id),
    journal_entry_id UUID REFERENCES journal_entry(id),
    -- Payment provider data
    provider_data   JSONB,
    -- Example provider_data:
    -- {
    --   "provider": "stripe",
    --   "payment_intent_id": "pi_3abc...",
    --   "charge_id": "ch_1xyz...",
    --   "fee_amount": 2.90,
    --   "fee_currency": "USD"
    -- }
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

## Bank Feeds

```sql
-- ============================================================
-- BANK FEEDS & RECONCILIATION
-- ============================================================

CREATE TABLE bank_connection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    provider        TEXT NOT NULL CHECK (provider IN ('plaid', 'open_banking', 'fdx', 'manual')),
    provider_item_id TEXT,
    institution_name TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',
    -- Full provider configuration in JSONB
    provider_config JSONB NOT NULL DEFAULT '{}',
    -- Example provider_config (Plaid):
    -- {
    --   "item_id": "plaid-item-abc",
    --   "access_token_ref": "vault://plaid/item-abc",
    --   "products": ["transactions", "balance"],
    --   "webhook_url": "https://api.example.com/webhooks/plaid",
    --   "consent_expiration": "2027-05-12T00:00:00Z"
    -- }
    -- Example provider_config (Open Banking):
    -- {
    --   "aspsp_id": "barclays-personal",
    --   "consent_id": "ob-consent-xyz",
    --   "consent_status": "authorised",
    --   "tpp_id": "our-tpp-id"
    -- }
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
    ledger_account_id UUID REFERENCES account(id),
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
    provider_txn_id TEXT NOT NULL,
    transaction_date DATE NOT NULL,
    posted_date     DATE,
    amount          NUMERIC(19,4) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    description     TEXT NOT NULL,
    merchant_name   TEXT,
    -- Full Plaid/provider response preserved as JSONB
    provider_data   JSONB NOT NULL DEFAULT '{}',
    -- Example provider_data (Plaid):
    -- {
    --   "personal_finance_category": {
    --     "primary": "RENT_AND_UTILITIES",
    --     "detailed": "RENT_AND_UTILITIES_RENT",
    --     "confidence_level": "VERY_HIGH"
    --   },
    --   "counterparties": [
    --     {"name": "WeWork", "type": "merchant", "logo_url": "..."}
    --   ],
    --   "location": {
    --     "city": "San Francisco",
    --     "region": "CA",
    --     "country": "US"
    --   },
    --   "payment_channel": "in store",
    --   "merchant_entity_id": "plaid-merchant-123"
    -- }
    -- AI categorisation
    ai_categorisation JSONB,
    -- Example ai_categorisation:
    -- {
    --   "suggested_account_id": "acct-123",
    --   "suggested_account_code": "6100",
    --   "confidence": 0.94,
    --   "model_id": "categoriser-v3.2",
    --   "reasoning": "Matched pattern: 'WeWork' → Office rent (6100)",
    --   "alternatives": [
    --     {"account_id": "acct-456", "code": "6200", "confidence": 0.04}
    --   ],
    --   "categorised_at": "2026-05-10T14:30:00Z"
    -- }
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
CREATE INDEX idx_bank_txn_provider ON bank_transaction USING gin (provider_data);
CREATE INDEX idx_bank_txn_ai ON bank_transaction USING gin (ai_categorisation)
    WHERE ai_categorisation IS NOT NULL;
```

## Tax

```sql
-- ============================================================
-- TAX CONFIGURATION
-- ============================================================

CREATE TABLE tax_rate (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    rate            NUMERIC(7,4) NOT NULL,
    tax_type        TEXT NOT NULL CHECK (tax_type IN (
        'sales_tax', 'vat', 'gst', 'withholding', 'exempt', 'zero_rated'
    )),
    jurisdiction    TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Jurisdiction-specific tax configuration
    jurisdiction_config JSONB NOT NULL DEFAULT '{}',
    -- Example jurisdiction_config:
    -- {
    --   "peppol_category_code": "S",
    --   "peppol_scheme_id": "UN/ECE 5305",
    --   "uk_mtd_box": 1,
    --   "de_tax_code": "19",
    --   "us_state_nexus": ["CA", "NY"],
    --   "effective_date": "2026-01-01",
    --   "expiry_date": null
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
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
    standard        TEXT NOT NULL DEFAULT 'asc_606',
    -- Contract details and modification history in JSONB
    contract_details JSONB NOT NULL DEFAULT '{}',
    -- Example contract_details:
    -- {
    --   "billing_model": "annual_upfront",
    --   "renewal_type": "auto",
    --   "variable_consideration": {
    --     "method": "expected_value",
    --     "estimates": [
    --       {"scenario": "base", "probability": 0.7, "amount": 120000},
    --       {"scenario": "upside", "probability": 0.3, "amount": 150000}
    --     ]
    --   },
    --   "modifications": [
    --     {
    --       "date": "2026-06-15",
    --       "type": "scope_change",
    --       "description": "Added premium support tier",
    --       "price_adjustment": 24000,
    --       "treatment": "cumulative_catchup"
    --     }
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, contract_number)
);

CREATE TABLE performance_obligation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    revenue_contract_id UUID NOT NULL REFERENCES revenue_contract(id) ON DELETE CASCADE,
    description     TEXT NOT NULL,
    obligation_type TEXT NOT NULL CHECK (obligation_type IN ('point_in_time', 'over_time')),
    standalone_selling_price NUMERIC(19,4) NOT NULL,
    allocated_price NUMERIC(19,4) NOT NULL,
    satisfaction_method TEXT,
    start_date      DATE NOT NULL,
    end_date        DATE,
    is_satisfied    BOOLEAN NOT NULL DEFAULT false,
    satisfied_at    TIMESTAMPTZ,
    revenue_account_id UUID NOT NULL REFERENCES account(id),
    deferred_revenue_account_id UUID NOT NULL REFERENCES account(id),
    -- Fulfilment tracking in JSONB
    fulfilment_data JSONB NOT NULL DEFAULT '{}',
    -- Example fulfilment_data:
    -- {
    --   "milestones": [
    --     {"name": "Implementation kickoff", "date": "2026-01-15", "complete": true},
    --     {"name": "Go-live", "date": "2026-03-01", "complete": true},
    --     {"name": "Training complete", "date": "2026-04-01", "complete": false}
    --   ],
    --   "usage_metrics": {
    --     "2026-01": {"api_calls": 45000, "users": 12},
    --     "2026-02": {"api_calls": 62000, "users": 15}
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE revenue_schedule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    performance_obligation_id UUID NOT NULL REFERENCES performance_obligation(id) ON DELETE CASCADE,
    fiscal_period_id UUID NOT NULL,
    recognition_date DATE NOT NULL,
    amount          NUMERIC(19,4) NOT NULL,
    is_recognised   BOOLEAN NOT NULL DEFAULT false,
    journal_entry_id UUID REFERENCES journal_entry(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rev_schedule_period ON revenue_schedule(fiscal_period_id);
```

## Fiscal Periods & Dimensions

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

-- ============================================================
-- DIMENSIONS
-- ============================================================

CREATE TABLE dimension (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    dimension_type  TEXT NOT NULL CHECK (dimension_type IN (
        'department', 'project', 'cost_centre', 'location', 'custom'
    )),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    parent_id       UUID REFERENCES dimension(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example metadata for project:
    -- {"budget": 50000, "currency": "USD", "start_date": "2026-01-01", "end_date": "2026-12-31"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, dimension_type, code)
);

CREATE INDEX idx_dimension_org ON dimension(organisation_id, dimension_type);
```

## Audit & AI

```sql
-- ============================================================
-- AUDIT TRAIL
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    user_id         UUID REFERENCES org_user(id),
    action          TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,                         -- {field: {old: x, new: y}}
    context         JSONB NOT NULL DEFAULT '{}',
    -- Example context:
    -- {
    --   "ip_address": "192.168.1.1",
    --   "user_agent": "Mozilla/5.0...",
    --   "ai_model_id": "categoriser-v3.2",
    --   "request_id": "req-abc-123",
    --   "api_key_id": "key-xyz"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org_date ON audit_log(organisation_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);

-- ============================================================
-- AI TRAINING FEEDBACK
-- ============================================================

CREATE TABLE ai_feedback (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    feedback_type   TEXT NOT NULL CHECK (feedback_type IN (
        'categorisation_accepted', 'categorisation_corrected',
        'categorisation_rejected', 'journal_accepted', 'journal_corrected',
        'anomaly_confirmed', 'anomaly_dismissed'
    )),
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    -- Full feedback context in JSONB for ML training
    feedback_data   JSONB NOT NULL,
    -- Example feedback_data:
    -- {
    --   "ai_suggestion": {"account_id": "a1", "confidence": 0.91},
    --   "user_correction": {"account_id": "a2"},
    --   "raw_input": "STRIPE PAYOUT 05/09",
    --   "model_id": "categoriser-v3.2",
    --   "features_used": ["merchant_name", "amount_range", "day_of_month"]
    -- }
    user_id         UUID NOT NULL REFERENCES org_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_feedback_org ON ai_feedback(organisation_id, feedback_type, created_at);
CREATE INDEX idx_ai_feedback_data ON ai_feedback USING gin (feedback_data);
```

## Startup Metrics & Exchange Rates

```sql
-- ============================================================
-- STARTUP METRICS
-- ============================================================

CREATE TABLE metric_snapshot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    snapshot_date   DATE NOT NULL,
    -- All metrics for this date in a single JSONB document
    metrics         JSONB NOT NULL,
    -- Example metrics:
    -- {
    --   "arr": 1440000,
    --   "mrr": 120000,
    --   "burn_rate": 85000,
    --   "runway_months": 14.1,
    --   "headcount_cost": 62000,
    --   "gross_margin_pct": 78.5,
    --   "net_revenue_retention_pct": 112,
    --   "customer_count": 42,
    --   "arpu": 2857.14,
    --   "ltv_cac_ratio": 3.2,
    --   "cash_balance": 1200000,
    --   "currency": "USD"
    -- }
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, snapshot_date)
);

CREATE INDEX idx_metric_org_date ON metric_snapshot(organisation_id, snapshot_date);

-- ============================================================
-- EXCHANGE RATES
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
    -- AI extraction results (OCR, document parsing)
    ai_extraction   JSONB,
    -- Example ai_extraction:
    -- {
    --   "document_type": "invoice",
    --   "extracted_fields": {
    --     "vendor_name": "Amazon Web Services",
    --     "invoice_number": "INV-2026-05-001",
    --     "total": 1234.56,
    --     "currency": "USD",
    --     "date": "2026-05-01"
    --   },
    --   "confidence": 0.96,
    --   "model_id": "doc-parser-v2.1"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_attachment_entity ON attachment(entity_type, entity_id);
```

## Example Queries

```sql
-- Query JSONB: find all invoices submitted to ChorusPro (French e-invoicing)
SELECT id, invoice_number, total, e_invoice_data->>'submission_id' AS chorus_id
FROM invoice
WHERE organisation_id = '<org_id>'
  AND e_invoice_data @> '{"format": "peppol_bis_3"}'
  AND e_invoice_data ? 'submission_id';

-- Query JSONB: find contacts with PEPPOL endpoints
SELECT id, name, jurisdiction_data->'peppol_endpoint_id' AS peppol_id
FROM contact
WHERE organisation_id = '<org_id>'
  AND jurisdiction_data ? 'peppol_endpoint_id';

-- Query JSONB: AI categorisation accuracy for a specific model version
SELECT
    feedback_type,
    COUNT(*) AS count
FROM ai_feedback
WHERE organisation_id = '<org_id>'
  AND feedback_data->>'model_id' = 'categoriser-v3.2'
GROUP BY feedback_type;

-- Mixed query: financial report with JSONB startup metrics
SELECT
    ms.snapshot_date,
    (ms.metrics->>'arr')::NUMERIC AS arr,
    (ms.metrics->>'burn_rate')::NUMERIC AS burn_rate,
    (ms.metrics->>'runway_months')::NUMERIC AS runway
FROM metric_snapshot ms
WHERE ms.organisation_id = '<org_id>'
ORDER BY ms.snapshot_date DESC
LIMIT 12;

-- Query JSONB: bank transactions with Plaid category filtering
SELECT id, transaction_date, amount, merchant_name,
       provider_data->'personal_finance_category'->>'primary' AS category
FROM bank_transaction
WHERE organisation_id = '<org_id>'
  AND provider_data->'personal_finance_category'->>'primary' = 'RENT_AND_UTILITIES'
ORDER BY transaction_date DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Multi-Tenancy | 2 | `organisation`, `org_user` — with JSONB permissions |
| Chart of Accounts | 1 | `account` — with JSONB xbrl_mapping and tax_config |
| Fiscal Periods | 2 | `fiscal_year`, `fiscal_period` |
| General Ledger | 2 | `journal_entry`, `journal_line` — with JSONB ai_data |
| Contacts | 1 | `contact` — addresses and bank details in JSONB |
| Invoicing | 3 | `invoice`, `invoice_line`, unified sales/purchase |
| Payments | 2 | `payment`, `payment_allocation` |
| Bank Feeds | 3 | `bank_connection`, `bank_account`, `bank_transaction` |
| Tax | 1 | `tax_rate` — with JSONB jurisdiction_config |
| Revenue Recognition | 3 | `revenue_contract`, `performance_obligation`, `revenue_schedule` |
| Dimensions | 1 | `dimension` — unified table with type discriminator |
| Audit & AI | 2 | `audit_log`, `ai_feedback` |
| Metrics | 1 | `metric_snapshot` — all metrics in one JSONB row per date |
| Exchange Rates | 1 | `exchange_rate` |
| Attachments | 1 | `attachment` — with JSONB ai_extraction |
| **Total** | **26** | Fewer tables than Model 1 due to JSONB consolidation |

---

## Key Design Decisions

1. **Unified invoice table for sales and purchases** — instead of separate `invoice` and `bill` tables, a single `invoice` table with a `direction` discriminator ('sales' or 'purchase'). This reduces table count, shares identical line-item logic, and simplifies reporting. The trade-off is that AP-specific and AR-specific workflows share a status field.

2. **Addresses stored as JSONB array on contact** — eliminates the `contact_address` junction table. Most contacts have 1-3 addresses. JSONB array is faster for the common case (read all addresses for a contact) and avoids a JOIN. The trade-off is that you cannot query "all contacts in ZIP 94102" without a GIN index on the JSONB array.

3. **Unified dimension table** — instead of separate `department` and `project` tables, a single `dimension` table with a `dimension_type` discriminator. Additional dimension types (cost centre, location, custom) can be added without schema changes.

4. **JSONB for jurisdiction-specific data everywhere** — `organisation.jurisdiction_data`, `contact.jurisdiction_data`, `invoice.e_invoice_data`, `tax_rate.jurisdiction_config`. This means adding support for a new country (e.g., India GST) requires zero schema migrations — just new application logic that reads/writes the appropriate JSONB keys.

5. **Full Plaid response preserved in provider_data** — rather than normalising selected Plaid fields into typed columns. This ensures no data loss during import and future-proofs against Plaid API changes (new category fields, counterparty data). The AI categorisation layer reads from `provider_data` and writes suggestions to a separate `ai_categorisation` JSONB column.

6. **Metrics as a single JSONB document per snapshot date** — instead of one row per metric type. This makes it trivial to add new metrics (LTV/CAC ratio, net dollar retention) without schema changes. The trade-off is that querying a single metric across time requires JSONB extraction.

7. **AI data co-located but in JSONB** — `journal_entry.ai_data` and `bank_transaction.ai_categorisation` store the full AI context (model ID, confidence, explanation, alternatives, feedback status) without needing separate AI-specific tables. The JSONB structure can evolve as AI features mature.

8. **Approval workflow in JSONB** — `invoice.approval_data` stores the full approval chain rather than requiring a separate `approval_step` table. For most SMBs, approval workflows are simple (1-2 approvers). Enterprise customers with complex multi-level approvals may eventually warrant a dedicated table, but JSONB handles the 90% case at MVP.

9. **GIN indexes on frequently queried JSONB columns** — `provider_data`, `ai_categorisation`, `e_invoice_data`, and `feedback_data` all have GIN indexes to support containment queries (`@>`) and existence checks (`?`). This keeps JSONB query performance competitive with typed columns for the most common access patterns.

10. **Contact types as a PostgreSQL array** — `contact_types TEXT[]` instead of a single `contact_type` enum. This natively supports contacts that are both customer and vendor without a junction table, and PostgreSQL's `@>` array operator enables efficient filtering.
