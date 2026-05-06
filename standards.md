# Standards & API Reference

> Project: AI-Native Accounting Software · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

**ISO 21378:2019 — Audit Data Collection**
- URL: https://www.iso.org/standard/70823.html
- Establishes common definitions of accounting data elements and provides the information necessary to extract relevant audit data. Defines eight modules: Base (BAS), General Ledger (GL), Accounts Receivable (AR), Sales (SAL), Purchase (PUR), Accounts Payable (AP), Inventory (INV), and Property, Plant and Equipment (PPE) — 71 tables in total. Directly relevant as the canonical data element vocabulary for any standards-compliant GL implementation. Confirmed current (reviewed and reaffirmed 2025).

**ISO 9001:2026 (anticipated) — Quality Management Systems**
- URL: https://www.iso.org/standards.html
- Governs quality management processes in software development and service delivery. Relevant for accounting software providers seeking enterprise procurement approval; buyers in regulated sectors (government, healthcare) commonly require ISO 9001 certification of vendors. Publication expected late 2026.

---

### W3C & IETF Standards

**XBRL (eXtensible Business Reporting Language) — FASB 2026 Taxonomies**
- URLs: https://fasb.org/xbrl · https://xbrl.us/xbrl-taxonomy/2026-us-gaap/ · https://www.sec.gov/newsroom/whats-new/2603-2026-xbrl-taxonomies-update
- The SEC-mandated structured reporting format for public company financial filings. The 2026 GAAP Financial Reporting Taxonomy (GRT) was accepted by the SEC on 17 March 2026 and is now live in EDGAR Release 26.1. The 2026 taxonomy incorporates all FASB standards published through December 2025. Any accounting software targeting US public companies or aiming to automate SEC filing must be able to produce XBRL-tagged output conforming to the current taxonomy. FASB hosts a webinar in April 2026 covering AI and data quality impacts on future taxonomy design.

**RFC 7231 — Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content**
- URL: https://www.rfc-editor.org/rfc/rfc7231
- The foundational IETF standard governing HTTP semantics. Directly applicable to any REST API layer built on top of an AI-native accounting engine. All major accounting APIs (QuickBooks, Xero, Rillet) are HTTP/REST APIs and conform to this specification.

**RFC 8288 — Web Linking**
- URL: https://www.rfc-editor.org/rfc/rfc8288
- Defines the Link header field and relation types for paginated API responses and HATEOAS patterns. Relevant for designing the pagination model of a financial data API where GL queries may return large result sets.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- The foundational standard for delegated authorization. Required for any multi-tenant accounting API that allows accountants, third-party apps, or bank feed providers to access a company's financial data on behalf of the owner. All production accounting APIs (QuickBooks, Xero, Plaid) mandate OAuth 2.0.

**OpenID Connect 1.0**
- URL: https://openid.net/connect/
- Identity layer on top of OAuth 2.0; provides standardized user authentication in addition to API authorization. Required for multi-user accounting platforms where auditable identity claims (who approved this entry, who logged in) are part of the audit trail.

---

### Data Model & API Specifications

**OpenAPI Specification 3.2.0**
- URL: https://spec.openapis.org/oas/v3.2.0.html · https://www.openapis.org/
- The industry-standard machine-readable description format for REST APIs, released September 2025. Rillet, Xero (via auto-generated SDKs), and Codat all publish OpenAPI-compliant API descriptions. Adopting OAS 3.2.0 at design time enables automatic SDK generation, interactive documentation (Swagger UI, Redoc), and API gateway configuration without additional tooling.

**OASIS UBL 2.1 — Universal Business Language**
- URL: https://docs.oasis-open.org/ubl/UBL-2.1.html · https://docs.peppol.eu/poacc/billing/3.0/
- XML-based standard for structured business documents (invoices, credit notes, order responses). UBL 2.1 is the technical syntax underlying the PEPPOL BIS Billing 3.0 e-invoicing profile, which is the dominant standard across Europe. Required for any accounting software generating or consuming PEPPOL-compliant e-invoices.

**EN 16931-1:2026 — Semantic Data Model of the Core Elements of an Electronic Invoice**
- URL: https://ec.europa.eu/digital-building-blocks/sites/spaces/DIGITAL/pages/467108950/EN+16931+compliance
- The European Committee for Standardization (CEN) published the updated version of this standard on 18 March 2026. EN 16931 defines the semantic data model for e-invoices used across all EU member states. It is the normative basis for PEPPOL BIS Billing 3.0 and national CIUS specifications (XRechnung for Germany, ChorusPro for France, NLCIUS for Netherlands). Any accounting software targeting EU B2G or B2B mandatory e-invoicing compliance must implement this standard.

**PEPPOL BIS Billing 3.0 — November 2025 Release**
- URL: https://docs.peppol.eu/poacc/billing/3.0/
- The most widely deployed e-invoicing profile in Europe, built on UBL 2.1 and EN 16931. Adds three validation layers: XML Schema (structural), Schematron (EN 16931 business rules), and PEPPOL-specific artifacts. Mandatory for EU public procurement and increasingly required for B2B in France, Germany, and Belgium. The November 2025 release is the current production specification.

**UN/CEFACT CII — Cross Industry Invoice**
- URL: https://www.unece.org/cefact/xml_schemas/index
- The alternative syntax to UBL 2.1 that is also conformant with EN 16931. Used in Germany's ZUGFeRD / Factur-X hybrid PDF+XML format (PDF invoice with embedded CII XML). Required for generating Factur-X documents, which are the dominant e-invoice format in France and Germany for SMBs.

**FDX API v6.5 — Financial Data Exchange**
- URL: https://financialdataexchange.org/
- The CFPB-recognized open banking standard for the United States (formally recognized January 2025 as a standards body under Section 1033 of Dodd-Frank). FDX v6.5 (released late 2025) is the current stable release; v6.4 added consumer consent management refinements. Over 100 million consumer accounts have transitioned to FDX APIs as of 2026. The definitive US standard for secure, consumer-permissioned financial data sharing between banks and accounting tools; aggregators such as Plaid and MX implement FDX alongside proprietary interfaces.

---

### Security & Authentication Standards

**OAuth 2.0 with PKCE (RFC 7636)**
- URL: https://www.rfc-editor.org/rfc/rfc7636 · https://cheatsheetseries.owasp.org/cheatsheets/OAuth2_Cheat_Sheet.html
- Proof Key for Code Exchange is the OWASP-recommended extension to OAuth 2.0 for public clients (mobile apps, SPAs). OWASP's cheat sheet documents current best practices for avoiding authorization code interception attacks. PKCE is mandatory for any OAuth 2.0 integration with an accounting app that has a browser or mobile component.

**OWASP Application Security Verification Standard (ASVS) 5.0**
- URL: https://github.com/OWASP/ASVS/blob/master/5.0/en/0x19-V10-OAuth-and-OIDC.md
- Chapter V10 covers OAuth 2.0 and OIDC security requirements. Given that accounting software stores sensitive financial records, ASVS Level 2 compliance is the expected baseline for enterprise sales. Version 5.0 includes updated OAuth/OIDC requirements.

**SOC 2 Type II (AICPA Trust Services Criteria)**
- URL: https://www.aicpa-cima.com/
- American Institute of CPAs framework for evaluating security, availability, processing integrity, confidentiality, and privacy of service providers. Required by enterprise buyers before connecting financial data to any SaaS accounting tool. Codat (SOC 2 Type II), Plaid (SOC 2 Type II), and all major commercial accounting platforms carry this certification. An open-source AI-native accounting tool seeking enterprise or accounting-firm adoption will need a hosted version that achieves SOC 2 Type II.

**GDPR (General Data Protection Regulation)**
- URL: https://gdpr.eu/
- EU regulation governing collection, processing, and storage of personal data. Relevant to any accounting software storing transaction data of EU residents or accessing EU bank accounts via open banking. Key requirements: data minimisation, right of erasure, data processing agreements with sub-processors (e.g., LLM providers), and explicit consent for AI processing of financial data.

**PCI DSS 4.0 (Payment Card Industry Data Security Standard)**
- URL: https://www.pcisecuritystandards.org/
- Applies to any accounting software that directly processes, stores, or transmits payment card data. Version 4.0 (effective March 2025) introduces customized implementation options. Relevant if the accounting platform includes payment collection or bill payment features that touch card numbers; not required if the system delegates card processing entirely to Stripe or similar.

---

### IRS & Tax Filing Standards

**IRS Modernized e-File (MeF)**
- URL: https://www.irs.gov/e-file-providers/modernized-e-file-overview · https://www.irs.gov/e-file-providers/modernized-e-file-mef-schemas-and-business-rules
- The IRS web-based system for electronic filing of US federal tax returns using XML schemas and WS-I security standards. XML schema definitions and business rules (reject criteria) are published for software developers. Required for any accounting software that performs direct US tax filing (corporate, individual, partnership, exempt organization returns). Access requires IRS-approved transmitter credentials and EFIN enrollment.

---

### MCP Server Specifications

**Model Context Protocol (MCP) — 2025-11-25 Specification**
- URL: https://modelcontextprotocol.io/specification/2025-11-25 · https://github.com/modelcontextprotocol/modelcontextprotocol
- Open protocol introduced by Anthropic (November 2024) standardizing how AI agents integrate with external data sources and tools. MCP operates on a client-server model: the accounting system exposes an MCP server that LLM clients can invoke to query the GL, create journal entries, run reports, or retrieve bank feed data. Rillet shipped an MCP server in early 2026 as part of its API offering. Microsoft Dynamics 365 Finance also implements MCP for agent-driven data operations. An AI-native accounting OSS project should implement an MCP server as its primary AI integration surface, enabling use with Claude, GPT-4o, and other LLM clients without bespoke tool integration.

---

## Similar Products — Developer Documentation & APIs

### QuickBooks Online (Intuit)

- **Description:** Dominant US SMB accounting platform with 62% market share; REST API enables third-party app integrations across accounting, tax, payroll, and banking domains.
- **API Documentation:** https://developer.intuit.com/app/developer/qbo/docs/develop
- **SDKs/Libraries:** Official SDKs for Java, PHP, Python, Node.js, and .NET at https://developer.intuit.com/app/developer/qbo/docs/learn/explore-the-quickbooks-online-api
- **Developer Guide:** https://developer.intuit.com/app/developer/qbo/docs/get-started
- **Standards:** REST/JSON; OpenAPI; webhooks migrated to CloudEvents format (May 2026)
- **Authentication:** OAuth 2.0 (mandatory); rate-limited at 10 req/sec per realm ID
- **API Notes:** Postman collection available at https://documenter.getpostman.com/view/3967924/RW1dEx9d. Intuit App Partner Program (2026) provides 500,000 CorePlus API calls/month on the free Builder Tier.

---

### Xero

- **Description:** Leading global cloud accounting platform; strong multi-currency and international tax support; open API with 1,000+ third-party integrations.
- **API Documentation:** https://developer.xero.com/documentation/
- **SDKs/Libraries:** Auto-generated SDKs from OpenAPI spec; Node.js, Python, Java, C#, PHP, Ruby at https://developer.xero.com/documentation/sdks-and-tools/
- **Developer Guide:** https://developer.xero.com/documentation/guides/oauth2/overview/
- **Standards:** REST/JSON; OpenAPI (Xero auto-generates SDKs from its OAS files); granular OAuth 2.0 scopes (new default for all apps created after March 2026)
- **Authentication:** OAuth 2.0 (Authorization Code flow or PKCE); OAuth 1.0a being deprecated
- **API Notes:** Xero HQ API provides multi-client accountant portal access at https://developer.xero.com/documentation/xero-hq/overview-xero-hq-api. Rate limits documented at https://developer.xero.com/documentation/guides/oauth2/limits/.

---

### Rillet

- **Description:** AI-native ERP for venture-backed startups; REST API and MCP server for AI coding tool integration; claims 93% journal entry automation.
- **API Documentation:** https://docs.api.rillet.com/docs/getting-started
- **SDKs/Libraries:** No official SDK libraries identified; API adheres to OpenAPI Specification enabling client code generation
- **Developer Guide:** https://www.rillet.com/product/api
- **Standards:** REST/JSON; OpenAPI Specification; MCP server (shipped early 2026)
- **Authentication:** API key authentication for server-to-server; OAuth 2.0 for user-facing integrations
- **API Notes:** Production endpoint at https://api.rillet.com; Sandbox at https://sandbox.api.rillet.com. Supports idempotency keys, webhooks, and incremental sync via `updated_at` filters (added February 2026). MCP server makes Rillet's GL directly queryable from AI coding agents.

---

### Puzzle

- **Description:** AI-native bookkeeping for startups with embedded accounting API; designed for both end-user startups and fintech developers building accounting products.
- **API Documentation:** https://puzzle.io/api · https://puzzle.io/embedded-accounting
- **SDKs/Libraries:** Not publicly documented; REST API with developer documentation
- **Developer Guide:** https://puzzle.io/solutions/api
- **Standards:** REST/JSON; webhook-based event notifications
- **Authentication:** API key / OAuth (specifics in developer documentation)
- **API Notes:** Embedded Accounting API allows fintechs to spin up a configured accounting environment per user at sign-up, with real-time transaction sync and report generation from day one. Mercury API integration documented at https://puzzle.io/blog/how-puzzle-uses-the-mercury-api-to-automate-your-startup-accounting-metrics.

---

### Stripe Revenue Recognition

- **Description:** Stripe's ASC 606 / IFRS 15-compliant revenue recognition module; API-driven recognition rules, performance obligations, and revenue contracts for SaaS billing models.
- **API Documentation:** https://docs.stripe.com/revenue-recognition/api
- **SDKs/Libraries:** Stripe SDKs (Node.js, Python, Ruby, PHP, Java, Go, .NET) at https://stripe.com/docs/libraries
- **Developer Guide:** https://docs.stripe.com/revenue-recognition/get-started · https://docs.stripe.com/revenue-recognition/performance-obligations-api
- **Standards:** REST/JSON; OpenAPI; event-driven via Stripe webhooks
- **Authentication:** API keys; restricted keys for scoped access
- **API Notes:** Revenue recognition is calculated to the second on all Stripe events (subscriptions, invoices, one-time payments, refunds, disputes). Revenue contracts API supports enterprise sales-led deals with custom contract periods. Usage-based billing integration is in private preview with current limitations on meter aggregation types.

---

### Plaid

- **Description:** US market-leading financial data aggregation platform; provides bank connectivity, transaction data, balance verification, and income analysis via REST API.
- **API Documentation:** https://plaid.com/docs/api/
- **SDKs/Libraries:** Official SDKs for Python, Node.js, Ruby, Java, Go at https://plaid.com/docs/libraries/
- **Developer Guide:** https://plaid.com/docs/
- **Standards:** REST/JSON (POST requests with JSON responses); SOC 2 Type II certified; PSD2-compliant for EU; FDX API v6.5-aligned for US open banking
- **Authentication:** API keys (client_id + secret); Plaid Link for consumer-facing OAuth flows
- **API Notes:** Key products relevant to accounting: Transactions (categorized transaction history), Balance (real-time account balances), Auth (ACH account and routing number verification), and Identity. Sandbox environment available for free testing with simulated institutions. QuickBooks, Xero, and FreshBooks use Plaid for bank feeds.

---

### Codat

- **Description:** Unified API platform providing normalized, bidirectional access to 30+ accounting platforms (QuickBooks, Xero, Sage, NetSuite, FreshBooks) and banking data aggregators; enables build-once access to all major accounting systems.
- **API Documentation:** https://docs.codat.io/ · https://docs.codat.io/accounting-api
- **SDKs/Libraries:** REST-based; language SDKs documented at https://docs.codat.io
- **Developer Guide:** https://docs.codat.io/integrations/accounting/overview
- **Standards:** REST/JSON; OpenAPI; webhooks; SOC 2 Type II certified; GDPR-compliant; OAuth 2.0 for data connections (no credential storage)
- **Authentication:** API key for Codat API; OAuth/open banking protocols for downstream platform connections
- **API Notes:** Supports real-time bidirectional sync — pull financial data for analysis and push transaction data back to customers' accounting systems. Partnered with Plaid and TrueLayer for banking data. Primary use case is lending, credit underwriting, and B2B financial software; relevant as a unified integration layer for an OSS accounting tool that needs to interoperate with incumbent systems.

---

### GnuCash / Beancount (Open-Source Reference Implementations)

- **Description:** GnuCash is the most widely deployed GPL-2.0 desktop accounting system; Beancount is a plain-text double-entry accounting DSL with an emerging LLM-native data layer.
- **API Documentation:** GnuCash: https://api.gnucash.org/ (Python bindings only, no REST API). Beancount: https://beancount.github.io/docs/ · https://fava.pythonanywhere.com/ (Fava web UI)
- **SDKs/Libraries:** GnuCash Python bindings (GnuCash 4.x+); Beancount Python library (pip install beancount); Fava (MIT-licensed web UI for Beancount)
- **Developer Guide:** https://beancount.github.io/docs/beancount_api.html
- **Standards:** GnuCash: XML or SQLite storage; OFX/QIF import. Beancount: plain-text DSL; no formal standard but de-facto format for plain-text accounting
- **Authentication:** Desktop applications; no API authentication model
- **API Notes:** Neither system exposes a production REST API. GnuCash's Python bindings allow scripted book manipulation but are not network-accessible. Beancount's plain-text format is directly readable by LLMs without any API, making it the most LLM-native format in the OSS accounting space. Fava (MIT) adds a read-only web dashboard. An AI-native OSS project could treat Beancount format as an import/export standard for interoperability with the developer community.

---

## Notes

**Gaps and emerging areas:**

- **No unified AI accounting API standard**: Unlike banking (FDX) or e-invoicing (EN 16931), there is no emerging ISO or W3C standard specifically defining how AI agents should interact with accounting systems. MCP is the closest candidate but is application-protocol-level, not domain-specific.

- **MCP as the de facto AI integration layer**: Both Rillet (early 2026) and Microsoft Dynamics 365 Finance have shipped MCP servers for their accounting data. This is converging as the practical standard for AI agent access to financial data, ahead of any formal standardization effort.

- **ASC 606 / IFRS 15 has no API schema standard**: The five-step revenue recognition model (FASB ASC 606, IFRS 15) is an accounting standard with no corresponding machine-readable data model or API specification. Stripe's Revenue Recognition API and Rillet's GL are the closest practical reference implementations; any OSS system must derive its own data model from the accounting literature.

- **PEPPOL mandate expansion**: E-invoicing mandates are expanding rapidly — France (B2B mandate effective September 2026), Germany (B2B mandate effective January 2025 for large companies), Belgium (2026). An OSS accounting tool targeting Europe must plan for PEPPOL BIS Billing 3.0 compliance from the outset, not as a backlog item.

- **Section 1033 / FDX regulatory uncertainty**: The CFPB's Section 1033 open banking rule (which underpins FDX's authority in the US) is under a judicial stay as of 2025-2026. FDX adoption continues voluntarily via industry consensus, but mandatory compliance timelines for US banks remain in flux. Plaid and MX continue operating under existing bilateral agreements alongside FDX compliance.
