# AI-Native Accounting Software

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source, AI-native double-entry accounting platform built ground-up around LLM reasoning rather than retrofitted onto 1990s-era bookkeeping software.

AI-Native Accounting Software is a self-hostable accounting core that combines a GAAP-compliant double-entry general ledger, bank-feed connectivity, and LLM-driven categorisation, journal entry generation, and financial intelligence. It targets startups, SMBs, accounting firms, and regulated organisations that need modern AI capabilities without the lock-in, cost, or US-centric assumptions of commercial vendors like QuickBooks, Xero, Rillet, Puzzle, and Zeni.

---

## Why AI-Native Accounting Software?

- **Incumbents bolted AI onto legacy cores.** QuickBooks (62% US SMB share) and Xero added AI-assisted categorisation on top of codebases designed for manual entry; LLM reasoning is not embedded in the data model.
- **AI-native commercial offerings are gated and US-centric.** Rillet ($100M+ raised), Puzzle ($66.5M raised), and Zeni (~$549/month base) are closed-source, expensive, and weak on international tax/VAT.
- **Open-source options are stuck in the past.** GnuCash, LedgerSMB, and Beancount have true double-entry but no bank feeds, no AI features, and no startup-specific metrics.
- **Startups pay extra to model basics.** Burn rate, runway, ASC 606 revenue recognition, and cohort ARR are missing from QuickBooks/Xero and absent from every OSS tool, forcing spreadsheets or expensive vertical SaaS.
- **Plain-text accounting is underexploited.** Ledger CLI and Beancount prove Git-versionable, LLM-parseable financial data is viable; no project pairs that paradigm with a modern UI and bank-feed layer.

---

## Key Features

### Double-Entry Core

- GAAP-compliant double-entry general ledger with full chart of accounts hierarchy
- Multi-currency support with exchange rate management
- Audit trail with immutable, timestamped, user-attributed change log
- Role-based access control with accountant and owner permission tiers
- PostgreSQL-backed storage for direct SQL reporting and ETL

### Bank Feeds and Automation

- Bank feed connectivity via Plaid and Open Banking / PSD2 APIs, with OFX/CSV fallback
- AI-assisted transaction categorisation using LLM few-shot reasoning over the chart of accounts
- AI journal entry generation from structured inputs: invoices, contracts, payroll exports
- Continuous close monitoring with real-time anomaly detection rather than batch month-end review
- Bank reconciliation with AI-suggested matches

### Invoicing, AP, and AR

- Invoice generation with payment status tracking
- Bill management and accounts payable workflow
- Accounts receivable tracking with aging reports
- Standard financial statement output: P&L, balance sheet, cash flow

### Startup and Growth Intelligence

- ASC 606 / IFRS 15 revenue recognition engine for SaaS and subscription businesses
- Native burn rate, runway, headcount cost, and ARR metrics derived continuously from the GL
- Multi-entity consolidation with intercompany elimination
- LLM-powered natural language query interface over the general ledger

### Compliance and Interoperability

- PEPPOL / UBL e-invoicing support for EU mandate compliance
- XBRL output for SEC filing integration
- Plain-text / Git-versionable export format (Beancount-compatible)
- AI-generated flux commentary for period-over-period variance analysis
- Tax compliance monitoring agent that tracks regulatory changes by jurisdiction

---

## AI-Native Advantage

Rather than treating AI as a categorisation bolt-on, this project embeds LLM reasoning at the data model level: the chart of accounts and journal entries are structured context for continuous agent-driven categorisation, anomaly detection, and close automation. AI agents can read contracts and invoices to draft standard accruals, generate flux commentary against budget and prior periods, monitor IRS/HMRC guidance for rule changes, and answer natural-language questions directly over the GL. Plain-text storage makes the entire ledger auditable, diffable, and directly LLM-queryable without proprietary APIs.

---

## Tech Stack & Deployment

The project is designed to be self-hostable, with PostgreSQL as the system of record and a plain-text export path compatible with Beancount for version-controlled workflows. Bank feeds are intended to use Plaid plus Open Banking / PSD2 for international coverage, with OFX/CSV import as a fallback. Compliance touchpoints include GAAP, IFRS, ASC 606 / IFRS 15, XBRL, PEPPOL e-invoicing, and IRS Modernized e-File. AI features are model-agnostic via an LLM abstraction layer, with original training on the project's own transaction data to avoid the patented categorisation methods filed by Intuit and Xero between 2017 and 2022.

---

## Market Context

The global accounting software market is projected at $22.72B in 2026 growing to $37.34B by 2030 (13.2% CAGR, The Business Research Company), with the AI-in-accounting sub-segment at $10.87B in 2026 growing at 44.6% CAGR through 2031 (Mordor Intelligence). Commercial pricing ranges from $15–$200/month for QuickBooks and Xero, $549+/month for Zeni, and custom enterprise pricing for Rillet and Puzzle. Primary buyers are startup CFOs and controllers, SMB owner-operators, accounting firms managing multiple clients, and enterprise finance teams needing multi-entity consolidation.

Candidate metadata: Complexity 8/10, Domain Availability Medium, Demand High, Category: Accounting & Finance.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context. Note: GPL-2.0 OSS accounting projects (GnuCash, Beancount core, LedgerSMB) cannot be re-licensed as proprietary derivatives; Beancount's Fava UI (MIT) and Blnk Finance (Apache 2.0) offer more permissive integration paths.
