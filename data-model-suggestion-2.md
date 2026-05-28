# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: AI-Native Accounting Software · Created: 2026-05-12

## Philosophy

This model treats accounting as what it fundamentally is: an append-only log of financial events. Every mutation to the financial state is captured as an immutable event in a single event store. The current state of any account, invoice, or reconciliation is a derived materialisation — computed by replaying events through projections. This is the CQRS (Command Query Responsibility Segregation) pattern: commands write events, queries read materialised views.

Double-entry bookkeeping is inherently event-sourced. An accountant never erases a journal entry; they post a reversing entry. This model elevates that principle to the architectural level. The event store IS the ledger. Corrections are new events (reversals, adjustments), not mutations. The audit trail is not a separate concern — it IS the data model. Every question about "who changed what, when, and why" is answered by querying the event stream, not a separate audit table.

Square's "Books" service (open-sourced design) and Blnk Finance's immutable ledger both use this pattern. Event sourcing is particularly powerful for an AI-native platform because the event stream provides perfect training data for ML models: every categorisation decision, every correction, every approval is an event with full temporal context. AI agents can replay events to understand patterns and generate predictions with complete provenance.

**Best for:** Teams building a platform where full audit trail integrity is non-negotiable, where temporal queries ("what was the balance on March 15?") are core requirements, and where AI features benefit from a rich, immutable event history for training and explainability.

**Trade-offs:**
- (+) Perfect audit trail by construction — impossible to lose history
- (+) Temporal queries are trivial: replay events to any point in time
- (+) AI training benefits from rich, timestamped event stream
- (+) Schema evolution via new event types without ALTER TABLE
- (+) Natural support for "undo" via compensating events
- (-) Higher storage requirements (events accumulate forever)
- (-) Query complexity: simple "current balance" requires materialised views
- (-) Eventual consistency between event store and read models
- (-) Steeper learning curve for developers unfamiliar with CQRS
- (-) Snapshot management needed for performance as event count grows

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 21378:2019 (Audit Data Collection) | Read model projections conform to ISO 21378 table structures, enabling standards-compliant audit data extraction from materialised views. |
| GAAP / IFRS | Double-entry integrity enforced at the command handler level: every `JournalEntryPosted` event must contain balanced debit/credit lines. |
| ASC 606 / IFRS 15 | Revenue recognition modelled as event chains: `ContractCreated` → `ObligationIdentified` → `PriceAllocated` → `RevenueSatisfied`. Temporal replay enables restatement. |
| XBRL 2026 US GAAP Taxonomy | XBRL mappings stored in account metadata events; financial statement generation replays events through XBRL-tagged projections. |
| PEPPOL BIS Billing 3.0 / UBL 2.1 | Invoice lifecycle modelled as events (`InvoiceIssued`, `InvoiceSent`, `PaymentReceived`); UBL XML generated from event replay at export time. |
| SOC 2 Type II | Immutable event store directly satisfies audit trail requirements without additional logging infrastructure. |
| FDX v6.5 / Plaid | Bank feed import produces `BankTransactionImported` events; reconciliation produces `TransactionMatched` events. Full provenance chain preserved. |

---

## Event Store (Core)

```sql
-- ============================================================
-- EVENT STORE — THE SINGLE SOURCE OF TRUTH
-- ============================================================

-- All financial state changes are events. This table is append-only.
-- No UPDATE or DELETE operations are ever performed on this table.

CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                 -- aggregate root identifier
    stream_type     TEXT NOT NULL,                  -- 'Account', 'JournalEntry', 'Invoice', etc.
    event_type      TEXT NOT NULL,                  -- 'JournalEntryPosted', 'InvoiceIssued', etc.
    event_version   BIGINT NOT NULL,               -- monotonically increasing per stream
    organisation_id UUID NOT NULL,
    payload         JSONB NOT NULL,                -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',   -- causation_id, correlation_id, user_id, ip, ai_model
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, event_version)
);

-- Partition by month for performance (events accumulate forever)
-- In production, use declarative partitioning:
-- CREATE TABLE event_store (...) PARTITION BY RANGE (created_at);

CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_event_type ON event_store(event_type, created_at);
CREATE INDEX idx_event_org ON event_store(organisation_id, created_at);
CREATE INDEX idx_event_org_stream_type ON event_store(organisation_id, stream_type, created_at);

-- Optimistic concurrency: before appending, check max(event_version) for the stream
-- and reject if it doesn't match the expected version.

-- ============================================================
-- EVENT TYPE CATALOGUE
-- ============================================================
-- The following event types form the domain language:
--
-- Organisation:
--   OrganisationCreated, OrganisationUpdated, UserInvited, UserRoleChanged
--
-- Chart of Accounts:
--   AccountCreated, AccountUpdated, AccountDeactivated, AccountReparented
--
-- General Ledger:
--   JournalEntryDrafted, JournalEntryPosted, JournalEntryReversed,
--   JournalEntryApproved, JournalEntryRejected
--
-- Invoicing:
--   InvoiceCreated, InvoiceIssued, InvoiceSent, InvoiceViewed,
--   InvoicePaymentReceived, InvoicePaid, InvoiceVoided
--
-- Bills:
--   BillReceived, BillApproved, BillPaymentMade, BillPaid, BillVoided
--
-- Bank Feeds:
--   BankConnectionEstablished, BankConnectionDisconnected,
--   BankTransactionImported, TransactionCategorised,
--   TransactionMatchSuggested, TransactionReconciled, TransactionExcluded
--
-- Revenue Recognition:
--   RevenueContractCreated, PerformanceObligationIdentified,
--   TransactionPriceAllocated, RevenueSatisfied, RevenueDeferred,
--   ContractModified
--
-- AI Events:
--   AICategorisationSuggested, AICategorisationAccepted,
--   AICategorisationCorrected, AICategorisationRejected,
--   AIJournalEntryDrafted, AIFluxCommentaryGenerated,
--   AIAnomalyDetected
--
-- Fiscal:
--   FiscalPeriodOpened, FiscalPeriodClosed, FiscalYearClosed
--
-- Metrics:
--   MetricComputed (burn_rate, runway, arr, mrr, etc.)
```

## Event Payload Examples

```sql
-- Example: JournalEntryPosted event payload
-- {
--   "entry_number": 1042,
--   "entry_date": "2026-05-10",
--   "description": "Monthly office rent",
--   "source_type": "bill",
--   "source_id": "550e8400-e29b-41d4-a716-446655440000",
--   "currency_code": "USD",
--   "lines": [
--     {
--       "account_id": "a1b2c3...",
--       "account_code": "6100",
--       "description": "Office rent - May 2026",
--       "debit": 5000.00,
--       "credit": 0,
--       "department_id": "d1e2f3..."
--     },
--     {
--       "account_id": "x9y8z7...",
--       "account_code": "2000",
--       "description": "Rent payable",
--       "debit": 0,
--       "credit": 5000.00,
--       "department_id": null
--     }
--   ]
-- }
--
-- metadata: {
--   "user_id": "u123...",
--   "causation_id": "cmd-456...",
--   "correlation_id": "flow-789...",
--   "ip_address": "192.168.1.1",
--   "ai_model_id": null
-- }

-- Example: AICategorisationSuggested event payload
-- {
--   "bank_transaction_id": "bt-123...",
--   "raw_description": "STRIPE TRANSFER 2026-05-09",
--   "merchant_name": "Stripe",
--   "amount": 12450.00,
--   "suggested_account_id": "a1b2c3...",
--   "suggested_account_code": "4000",
--   "confidence": 0.94,
--   "model_id": "categoriser-v3.2",
--   "reasoning": "Matched pattern: payment processor deposit → Revenue account 4000",
--   "alternatives": [
--     {"account_id": "a4b5c6...", "confidence": 0.04},
--     {"account_id": "a7b8c9...", "confidence": 0.02}
--   ]
-- }

-- Example: RevenueContractCreated event payload
-- {
--   "contract_number": "RC-2026-0042",
--   "customer_id": "c-123...",
--   "start_date": "2026-01-01",
--   "end_date": "2026-12-31",
--   "total_value": 120000.00,
--   "currency_code": "USD",
--   "standard": "asc_606",
--   "obligations": [
--     {
--       "description": "SaaS platform access",
--       "type": "over_time",
--       "standalone_selling_price": 96000.00,
--       "satisfaction_method": "straight_line"
--     },
--     {
--       "description": "Implementation services",
--       "type": "point_in_time",
--       "standalone_selling_price": 24000.00,
--       "satisfaction_method": "output"
--     }
--   ]
-- }
```

## Snapshots (Performance Optimisation)

```sql
-- ============================================================
-- SNAPSHOTS — PERIODIC STATE CHECKPOINTS
-- ============================================================

-- As event counts grow, replaying from the beginning becomes slow.
-- Snapshots store the materialised state at a point in time,
-- allowing replay to start from the latest snapshot.

CREATE TABLE event_snapshot (
    snapshot_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,
    stream_type     TEXT NOT NULL,
    last_event_version BIGINT NOT NULL,            -- replay from here
    state           JSONB NOT NULL,                -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_snapshot_stream ON event_snapshot(stream_id, last_event_version DESC);

-- Snapshots are created periodically (e.g., every 100 events per stream)
-- or at fiscal period close.
```

## Materialised Read Models

```sql
-- ============================================================
-- READ MODELS (MATERIALISED PROJECTIONS)
-- ============================================================

-- These tables are DERIVED from the event store.
-- They can be rebuilt from scratch by replaying all events.
-- They are optimised for query performance, not for write integrity.

-- ---- Organisation Read Model ----

CREATE TABLE rm_organisation (
    id              UUID PRIMARY KEY,
    name            TEXT NOT NULL,
    legal_name      TEXT,
    tax_id          TEXT,
    country_code    CHAR(2) NOT NULL,
    base_currency   CHAR(3) NOT NULL,
    fiscal_year_end_month SMALLINT NOT NULL,
    peppol_endpoint_id TEXT,
    settings        JSONB NOT NULL DEFAULT '{}',
    last_event_version BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

-- ---- Chart of Accounts Read Model ----

CREATE TABLE rm_account (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    account_type    TEXT NOT NULL,
    account_subtype TEXT,
    parent_id       UUID,
    currency_code   CHAR(3) NOT NULL,
    is_active       BOOLEAN NOT NULL,
    xbrl_element    TEXT,
    normal_balance  TEXT NOT NULL,
    current_balance NUMERIC(19,4) NOT NULL DEFAULT 0,  -- running balance!
    last_event_version BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_account_org ON rm_account(organisation_id);
CREATE INDEX idx_rm_account_code ON rm_account(organisation_id, code);

-- ---- Account Balance Read Model (period-level) ----

CREATE TABLE rm_account_balance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    account_id      UUID NOT NULL,
    fiscal_period_id UUID NOT NULL,
    opening_balance NUMERIC(19,4) NOT NULL DEFAULT 0,
    total_debits    NUMERIC(19,4) NOT NULL DEFAULT 0,
    total_credits   NUMERIC(19,4) NOT NULL DEFAULT 0,
    closing_balance NUMERIC(19,4) NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL,
    UNIQUE (account_id, fiscal_period_id)
);

CREATE INDEX idx_rm_balance_org_period ON rm_account_balance(organisation_id, fiscal_period_id);

-- ---- Journal Entry Read Model ----

CREATE TABLE rm_journal_entry (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    entry_number    BIGINT NOT NULL,
    entry_date      DATE NOT NULL,
    description     TEXT NOT NULL,
    source_type     TEXT NOT NULL,
    source_id       UUID,
    status          TEXT NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    total_debit     NUMERIC(19,4) NOT NULL,
    total_credit    NUMERIC(19,4) NOT NULL,
    is_reversed     BOOLEAN NOT NULL DEFAULT false,
    reversal_of     UUID,
    posted_by       UUID,
    posted_at       TIMESTAMPTZ,
    ai_confidence   NUMERIC(5,4),
    ai_model_id     TEXT,
    ai_explanation  TEXT,
    created_at      TIMESTAMPTZ NOT NULL,
    last_event_version BIGINT NOT NULL
);

CREATE INDEX idx_rm_je_org_date ON rm_journal_entry(organisation_id, entry_date);
CREATE INDEX idx_rm_je_status ON rm_journal_entry(organisation_id, status);

CREATE TABLE rm_journal_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    journal_entry_id UUID NOT NULL REFERENCES rm_journal_entry(id),
    account_id      UUID NOT NULL,
    account_code    TEXT NOT NULL,
    line_number     SMALLINT NOT NULL,
    description     TEXT,
    debit_amount    NUMERIC(19,4) NOT NULL DEFAULT 0,
    credit_amount   NUMERIC(19,4) NOT NULL DEFAULT 0,
    department_id   UUID,
    project_id      UUID
);

CREATE INDEX idx_rm_jl_entry ON rm_journal_line(journal_entry_id);
CREATE INDEX idx_rm_jl_account ON rm_journal_line(account_id);

-- ---- Contact Read Model ----

CREATE TABLE rm_contact (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    contact_type    TEXT NOT NULL,
    name            TEXT NOT NULL,
    legal_name      TEXT,
    email           TEXT,
    tax_id          TEXT,
    currency_code   CHAR(3),
    payment_terms_days SMALLINT,
    is_active       BOOLEAN NOT NULL,
    peppol_endpoint_id TEXT,
    balance_due     NUMERIC(19,4) NOT NULL DEFAULT 0,  -- running AR/AP balance
    last_event_version BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_contact_org ON rm_contact(organisation_id, contact_type);

-- ---- Invoice Read Model ----

CREATE TABLE rm_invoice (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    invoice_number  TEXT NOT NULL,
    invoice_type    TEXT NOT NULL,
    contact_id      UUID NOT NULL,
    contact_name    TEXT NOT NULL,                  -- denormalised for read performance
    status          TEXT NOT NULL,
    issue_date      DATE NOT NULL,
    due_date        DATE NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    subtotal        NUMERIC(19,4) NOT NULL,
    tax_total       NUMERIC(19,4) NOT NULL,
    total           NUMERIC(19,4) NOT NULL,
    amount_paid     NUMERIC(19,4) NOT NULL DEFAULT 0,
    amount_due      NUMERIC(19,4) NOT NULL,
    days_overdue    INTEGER GENERATED ALWAYS AS (
        CASE WHEN status NOT IN ('paid', 'void')
             THEN GREATEST(0, CURRENT_DATE - due_date)
             ELSE 0 END
    ) STORED,
    last_event_version BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_invoice_org ON rm_invoice(organisation_id, status);
CREATE INDEX idx_rm_invoice_due ON rm_invoice(organisation_id, due_date)
    WHERE status NOT IN ('paid', 'void');

-- ---- Bank Transaction Read Model ----

CREATE TABLE rm_bank_transaction (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    bank_account_id UUID NOT NULL,
    provider_txn_id TEXT NOT NULL,
    transaction_date DATE NOT NULL,
    amount          NUMERIC(19,4) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    description     TEXT NOT NULL,
    merchant_name   TEXT,
    plaid_category_primary TEXT,
    plaid_category_detailed TEXT,
    ai_suggested_account_id UUID,
    ai_confidence   NUMERIC(5,4),
    reconciliation_status TEXT NOT NULL DEFAULT 'unmatched',
    matched_journal_entry_id UUID,
    last_event_version BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_bank_txn_org ON rm_bank_transaction(organisation_id, transaction_date);
CREATE INDEX idx_rm_bank_txn_status ON rm_bank_transaction(organisation_id, reconciliation_status);

-- ---- Revenue Recognition Read Model ----

CREATE TABLE rm_revenue_contract (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    contract_number TEXT NOT NULL,
    customer_id     UUID NOT NULL,
    customer_name   TEXT NOT NULL,
    status          TEXT NOT NULL,
    start_date      DATE NOT NULL,
    end_date        DATE,
    total_value     NUMERIC(19,4) NOT NULL,
    recognised_to_date NUMERIC(19,4) NOT NULL DEFAULT 0,
    deferred_balance NUMERIC(19,4) NOT NULL DEFAULT 0,
    currency_code   CHAR(3) NOT NULL,
    standard        TEXT NOT NULL,
    last_event_version BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE TABLE rm_performance_obligation (
    id              UUID PRIMARY KEY,
    contract_id     UUID NOT NULL,
    description     TEXT NOT NULL,
    obligation_type TEXT NOT NULL,
    allocated_price NUMERIC(19,4) NOT NULL,
    recognised_amount NUMERIC(19,4) NOT NULL DEFAULT 0,
    deferred_amount NUMERIC(19,4) NOT NULL DEFAULT 0,
    is_satisfied    BOOLEAN NOT NULL DEFAULT false,
    satisfied_at    TIMESTAMPTZ,
    last_event_version BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

-- ---- Startup Metrics Read Model ----

CREATE TABLE rm_metric (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    metric_type     TEXT NOT NULL,
    snapshot_date   DATE NOT NULL,
    value           NUMERIC(19,4) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    computed_at     TIMESTAMPTZ NOT NULL,
    UNIQUE (organisation_id, metric_type, snapshot_date)
);

CREATE INDEX idx_rm_metric_org ON rm_metric(organisation_id, metric_type, snapshot_date);

-- ---- Fiscal Period Read Model ----

CREATE TABLE rm_fiscal_period (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    fiscal_year_id  UUID NOT NULL,
    year_label      TEXT NOT NULL,
    period_number   SMALLINT NOT NULL,
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    is_closed       BOOLEAN NOT NULL DEFAULT false,
    last_event_version BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
```

## Projection Tracking

```sql
-- ============================================================
-- PROJECTION TRACKING
-- ============================================================

-- Tracks which event each projection has processed up to,
-- enabling incremental updates and rebuild from a known position.

CREATE TABLE projection_checkpoint (
    projection_name TEXT PRIMARY KEY,              -- e.g., 'rm_account', 'rm_invoice'
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- When rebuilding a read model:
-- 1. TRUNCATE the read model table
-- 2. DELETE FROM projection_checkpoint WHERE projection_name = '...'
-- 3. Replay all events from event_store, applying the projection logic
-- 4. INSERT the new checkpoint
```

## Outbox Pattern (for Integrations)

```sql
-- ============================================================
-- OUTBOX — RELIABLE EVENT PUBLISHING
-- ============================================================

-- Events that need to be published to external systems (webhooks,
-- MCP server, bank feed sync) are written to the outbox within
-- the same transaction as the event store insert.

CREATE TABLE event_outbox (
    outbox_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL,                 -- references event_store
    destination     TEXT NOT NULL,                  -- 'webhook', 'mcp', 'bank_sync'
    payload         JSONB NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'processing', 'delivered', 'failed'
    )),
    attempts        SMALLINT NOT NULL DEFAULT 0,
    last_attempt_at TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_outbox_pending ON event_outbox(status, created_at)
    WHERE status IN ('pending', 'failed');
```

## Example Queries

```sql
-- Replay account balance at a specific point in time
-- (This is the killer feature of event sourcing)
SELECT
    (payload->>'lines')::jsonb AS lines
FROM event_store
WHERE stream_type = 'JournalEntry'
  AND event_type = 'JournalEntryPosted'
  AND organisation_id = '<org_id>'
  AND created_at <= '2026-03-15T23:59:59Z'
ORDER BY created_at;
-- Application code sums debits/credits per account from the replayed events

-- AI categorisation accuracy over time
SELECT
    date_trunc('month', created_at) AS month,
    COUNT(*) FILTER (WHERE event_type = 'AICategorisationAccepted') AS accepted,
    COUNT(*) FILTER (WHERE event_type = 'AICategorisationCorrected') AS corrected,
    COUNT(*) FILTER (WHERE event_type = 'AICategorisationRejected') AS rejected,
    ROUND(
        COUNT(*) FILTER (WHERE event_type = 'AICategorisationAccepted')::NUMERIC /
        NULLIF(COUNT(*), 0) * 100, 1
    ) AS accuracy_pct
FROM event_store
WHERE organisation_id = '<org_id>'
  AND event_type IN ('AICategorisationAccepted', 'AICategorisationCorrected', 'AICategorisationRejected')
GROUP BY date_trunc('month', created_at)
ORDER BY month;

-- Full audit trail for a specific journal entry (no separate audit table needed)
SELECT
    event_type,
    payload,
    metadata->>'user_id' AS user_id,
    metadata->>'ip_address' AS ip,
    created_at
FROM event_store
WHERE stream_id = '<journal_entry_id>'
ORDER BY event_version;

-- Revenue recognition timeline for a contract
SELECT
    event_type,
    payload->>'amount' AS amount,
    payload->>'obligation_description' AS obligation,
    created_at
FROM event_store
WHERE stream_id = '<contract_id>'
  AND stream_type = 'RevenueContract'
ORDER BY event_version;

-- Current state from read model (fast, no replay needed)
SELECT
    a.code,
    a.name,
    a.account_type,
    a.current_balance
FROM rm_account a
WHERE a.organisation_id = '<org_id>'
  AND a.is_active = true
ORDER BY a.code;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | `event_store` — the single source of truth |
| Snapshots | 1 | `event_snapshot` — performance checkpoints |
| Projection Tracking | 1 | `projection_checkpoint` |
| Outbox | 1 | `event_outbox` — reliable event publishing |
| Read Models — Organisation | 1 | `rm_organisation` |
| Read Models — Chart of Accounts | 2 | `rm_account`, `rm_account_balance` |
| Read Models — General Ledger | 2 | `rm_journal_entry`, `rm_journal_line` |
| Read Models — Contacts | 1 | `rm_contact` |
| Read Models — Invoicing | 1 | `rm_invoice` |
| Read Models — Bank Feeds | 1 | `rm_bank_transaction` |
| Read Models — Revenue Recognition | 2 | `rm_revenue_contract`, `rm_performance_obligation` |
| Read Models — Metrics | 1 | `rm_metric` |
| Read Models — Fiscal | 1 | `rm_fiscal_period` |
| **Total** | **16** | 1 event store + 3 infrastructure + 12 read models |

---

## Key Design Decisions

1. **Single event_store table with JSONB payload** — rather than one table per event type. This keeps the append path simple and fast. The trade-off is that event payloads are schemaless; validation happens in the command handler before events are persisted.

2. **Stream-based partitioning** — events are grouped by `stream_id` (aggregate root). This enables efficient aggregate reconstruction: load all events for a journal entry or invoice by stream_id, apply them in order.

3. **Optimistic concurrency via event_version** — before appending an event, the command handler checks that `max(event_version)` for the stream matches the expected version. This prevents lost updates without pessimistic locking.

4. **Read models are denormalised and disposable** — every `rm_*` table can be dropped and rebuilt from the event store. This means read models can be optimised aggressively (denormalised contact names on invoices, computed `days_overdue` columns) without worrying about data integrity.

5. **Running balances on read model accounts** — `rm_account.current_balance` is updated by the projection whenever a `JournalEntryPosted` or `JournalEntryReversed` event is processed. This makes balance queries instant (no aggregation needed) at the cost of eventual consistency.

6. **AI events as first-class citizens** — `AICategorisationSuggested`, `AICategorisationAccepted`, `AICategorisationCorrected` are event types alongside financial events. This creates a self-documenting ML pipeline: model performance can be measured by querying the event stream, and training data can be extracted by replaying categorisation event sequences.

7. **Outbox pattern for external integrations** — rather than publishing events directly to message queues, events destined for external systems are written to `event_outbox` within the same database transaction. A background worker polls the outbox and delivers events, ensuring at-least-once delivery.

8. **Snapshots for aggregate replay performance** — for aggregates with many events (e.g., a high-volume bank account with thousands of transactions), periodic snapshots store the serialised state. Replay starts from the latest snapshot rather than from event zero.

9. **Metadata envelope on every event** — `causation_id` traces what caused this event (the command ID), `correlation_id` links events across aggregates in the same business flow, and `user_id` / `ip_address` / `ai_model_id` provide audit attribution. This is richer than a traditional audit log.

10. **No separate audit_log table** — the event store IS the audit log. Every state change is an event. Querying "who changed invoice X and when?" is a simple `SELECT ... WHERE stream_id = '<invoice_id>' ORDER BY event_version`. This eliminates the dual-write problem where audit logs and data tables can diverge.
