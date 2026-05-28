# AI-Native Accounting Software — Phased Development Plan

> Project: 061-ai-native-accounting-software · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions & Rationale

### Language & Runtime
- **Backend: Python 3.12+ with FastAPI** — Rationale: Python dominates in ML/AI tooling (direct access to LLM SDKs, pandas, scikit-learn); FastAPI provides async REST with automatic OpenAPI 3.2.0 spec generation; Beancount and many accounting libraries are Python-native. Type hints with Pydantic provide runtime validation without a compilation step.
- **Frontend: TypeScript + React 19 with Next.js 15** — Rationale: industry-standard for SaaS dashboards; Server Components reduce client bundle for data-heavy financial views; TanStack Table for ledger grids.
- **CLI: Python (Click)** — for admin operations, migrations, and developer tooling.

### Data Model
- **Hybrid Relational + JSONB (Data Model Suggestion 3) as the primary model, with select immutability patterns from Suggestion 4.** Rationale:
  - The Hybrid model (Suggestion 3) provides the fastest path to MVP: typed columns for financial aggregation and constraints, JSONB for jurisdiction-specific data, AI metadata, and e-invoicing fields without schema migrations.
  - From Suggestion 4, we adopt: (a) immutable `posting_group` / `posting` tables with database triggers preventing UPDATE/DELETE, (b) content hashing for tamper detection, (c) a materialised reporting layer for financial statements and metrics.
  - We do NOT adopt full event sourcing (Suggestion 2) because the added CQRS complexity delays MVP and most accounting queries are current-state, not temporal replays.
  - We do NOT adopt pure normalized relational (Suggestion 1) because jurisdiction-specific nullable columns would proliferate unmanageably across 50+ countries.

### Database
- **PostgreSQL 16** — Rationale: JSONB with GIN indexes, Row Level Security for multi-tenancy, ltree extension for chart-of-accounts hierarchy, partitioning for audit logs, triggers for immutability enforcement. All four data model suggestions are PostgreSQL-native.

### AI / LLM Layer
- **Model-agnostic via an LLM abstraction layer** — support Claude (Anthropic SDK), GPT-4o (OpenAI SDK), and local models via Ollama. Rationale: avoids vendor lock-in; allows self-hosted deployments to run without cloud LLM dependencies.
- **MCP Server** — expose GL, bank transactions, and reports as MCP tools, following the pattern established by Rillet (early 2026) and Microsoft Dynamics 365 Finance.

### Authentication & Authorization
- **OAuth 2.0 with PKCE + OpenID Connect** — Rationale: required by every bank feed provider (Plaid, Open Banking), mandated by OWASP ASVS 5.0 for financial applications.
- **PostgreSQL Row Level Security** — defence-in-depth multi-tenancy beyond application-layer filtering.

### Bank Feeds
- **Plaid (US)** + **Open Banking / PSD2 (EU/UK)** + **OFX/CSV fallback** — Rationale: covers the three major banking data access patterns. Plaid is used by QuickBooks, Xero, and FreshBooks. FDX v6.5 alignment for future US open banking compliance.

### E-Invoicing
- **PEPPOL BIS Billing 3.0 / UBL 2.1 / EN 16931** — Rationale: EU B2B e-invoicing mandates are expanding (France Sept 2026, Germany Jan 2025, Belgium 2026). Must be designed in from the start, not bolted on.

### Deployment
- **Docker Compose for development, Kubernetes for production** — self-hostable as a core requirement.
- **S3-compatible object storage** for attachments and document PDFs.

### Licence
- **AGPL-3.0** for the core engine, **MIT** for client SDKs and the MCP server. Rationale: AGPL ensures open-source derivatives remain open (matching the project's OSS mission), while MIT SDKs allow commercial integrations without copyleft concerns.

---

## Project Structure

```
061-ai-native-accounting-software/
├── backend/
│   ├── alembic/                    # Database migrations
│   │   └── versions/
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py                 # FastAPI application entry
│   │   ├── config.py               # Settings (Pydantic BaseSettings)
│   │   ├── db/
│   │   │   ├── session.py          # SQLAlchemy async session
│   │   │   ├── base.py             # Declarative base
│   │   │   └── rls.py              # Row Level Security helpers
│   │   ├── models/                 # SQLAlchemy ORM models
│   │   │   ├── organisation.py
│   │   │   ├── account.py
│   │   │   ├── journal.py
│   │   │   ├── posting.py          # Immutable posting_group + posting
│   │   │   ├── invoice.py
│   │   │   ├── payment.py
│   │   │   ├── contact.py
│   │   │   ├── bank_feed.py
│   │   │   ├── revenue.py
│   │   │   ├── tax.py
│   │   │   ├── dimension.py
│   │   │   ├── audit.py
│   │   │   ├── ai_feedback.py
│   │   │   ├── metric.py
│   │   │   ├── attachment.py
│   │   │   └── materialised.py     # mat_* reporting tables
│   │   ├── schemas/                # Pydantic request/response schemas
│   │   │   ├── organisation.py
│   │   │   ├── account.py
│   │   │   ├── journal.py
│   │   │   ├── invoice.py
│   │   │   ├── bank_feed.py
│   │   │   └── ...
│   │   ├── api/                    # FastAPI routers
│   │   │   ├── v1/
│   │   │   │   ├── organisations.py
│   │   │   │   ├── accounts.py
│   │   │   │   ├── journal_entries.py
│   │   │   │   ├── invoices.py
│   │   │   │   ├── bank_feeds.py
│   │   │   │   ├── reports.py
│   │   │   │   ├── contacts.py
│   │   │   │   ├── payments.py
│   │   │   │   ├── revenue.py
│   │   │   │   └── metrics.py
│   │   │   └── router.py
│   │   ├── services/               # Business logic layer
│   │   │   ├── ledger.py           # Double-entry posting logic
│   │   │   ├── reconciliation.py
│   │   │   ├── invoicing.py
│   │   │   ├── revenue_recognition.py
│   │   │   ├── report_generator.py
│   │   │   ├── metric_calculator.py
│   │   │   ├── bank_sync.py
│   │   │   └── fiscal.py
│   │   ├── ai/                     # AI / LLM integration
│   │   │   ├── llm_client.py       # Model-agnostic LLM abstraction
│   │   │   ├── categoriser.py      # Transaction categorisation
│   │   │   ├── journal_drafter.py  # AI journal entry generation
│   │   │   ├── flux_analyst.py     # Variance analysis commentary
│   │   │   ├── nl_query.py         # Natural language GL queries
│   │   │   ├── anomaly.py          # Anomaly detection
│   │   │   └── feedback.py         # AI feedback loop
│   │   ├── integrations/           # External service adapters
│   │   │   ├── plaid_adapter.py
│   │   │   ├── open_banking.py
│   │   │   ├── peppol.py
│   │   │   ├── xbrl.py
│   │   │   └── exchange_rates.py
│   │   └── mcp/                    # MCP server
│   │       ├── server.py
│   │       └── tools.py
│   ├── tests/
│   │   ├── conftest.py
│   │   ├── unit/
│   │   ├── integration/
│   │   └── e2e/
│   ├── pyproject.toml
│   ├── Dockerfile
│   └── alembic.ini
├── frontend/
│   ├── src/
│   │   ├── app/                    # Next.js App Router
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── lib/
│   │   └── types/
│   ├── package.json
│   ├── tsconfig.json
│   └── Dockerfile
├── docker-compose.yml
├── docker-compose.prod.yml
├── .env.example
├── Makefile
└── README.md
```

---

## Phase Dependency Graph

```
Phase 1: Foundation
    │
    ├──▶ Phase 2: Chart of Accounts & General Ledger
    │        │
    │        ├──▶ Phase 3: Invoicing, AP/AR & Payments
    │        │        │
    │        │        └──▶ Phase 6: Revenue Recognition (ASC 606)
    │        │
    │        ├──▶ Phase 4: Bank Feeds & Reconciliation
    │        │        │
    │        │        └──▶ Phase 5: AI Categorisation & Journal Drafting
    │        │
    │        └──▶ Phase 7: Financial Statements & Reporting
    │                 │
    │                 └──▶ Phase 8: Startup Metrics & Intelligence
    │
    ├──▶ Phase 9: Frontend Application (can start after Phase 2, iterates with each phase)
    │
    └──▶ Phase 10: MCP Server & AI Query Interface
             │
             └──▶ Phase 11: E-Invoicing & International Compliance
                      │
                      └──▶ Phase 12: Production Hardening & Deployment
```

**Critical path:** Phase 1 → 2 → 4 → 5 (AI categorisation depends on bank feed data).
**Parallel work:** Phase 3 (invoicing) can proceed in parallel with Phase 4 (bank feeds) once Phase 2 is complete. Phase 9 (frontend) can begin after Phase 2 and iterate continuously.

---

## Phase 1: Foundation — Project Scaffolding, Database, Auth

### Definition of Done
- FastAPI application boots and serves a health endpoint.
- PostgreSQL database is provisioned with multi-tenancy via Row Level Security.
- OAuth 2.0 / OIDC authentication flow completes end-to-end.
- Organisation CRUD works with RLS enforced.
- Docker Compose runs the full stack locally.
- CI pipeline runs tests on every push.

### Task 1.1: Project Scaffolding & Configuration

**What:** Create the Python backend project with FastAPI, configure dependency management, and set up the development environment.

**Design:**

```python
# backend/app/config.py
from pydantic_settings import BaseSettings
from typing import Literal

class Settings(BaseSettings):
    app_name: str = "ai-native-accounting"
    environment: Literal["development", "staging", "production"] = "development"
    debug: bool = True

    # Database
    database_url: str = "postgresql+asyncpg://accounting:accounting@localhost:5432/accounting"
    database_pool_size: int = 20
    database_max_overflow: int = 10

    # Auth
    oidc_issuer_url: str = ""
    oidc_client_id: str = ""
    oidc_audience: str = ""

    # LLM
    llm_provider: Literal["anthropic", "openai", "ollama"] = "anthropic"
    llm_api_key: str = ""
    llm_model: str = "claude-sonnet-4-20250514"

    # Storage
    s3_bucket: str = ""
    s3_endpoint_url: str | None = None

    # Plaid
    plaid_client_id: str = ""
    plaid_secret: str = ""
    plaid_environment: Literal["sandbox", "development", "production"] = "sandbox"

    class Config:
        env_file = ".env"
        env_prefix = "ACCOUNTING_"

# backend/app/main.py
from fastapi import FastAPI
from contextlib import asynccontextmanager
from app.config import Settings
from app.db.session import init_db, close_db

settings = Settings()

@asynccontextmanager
async def lifespan(app: FastAPI):
    await init_db()
    yield
    await close_db()

app = FastAPI(
    title="AI-Native Accounting Software",
    version="0.1.0",
    lifespan=lifespan,
    docs_url="/api/docs",
    openapi_url="/api/openapi.json",
)

@app.get("/api/health")
async def health():
    return {"status": "ok", "version": "0.1.0"}
```

**Testing:**
- `test_health_endpoint_returns_ok` — GET /api/health returns 200 with `{"status": "ok"}`.
- `test_settings_load_from_env` — Settings correctly reads from environment variables.
- `test_openapi_spec_accessible` — GET /api/openapi.json returns valid OpenAPI 3.x document.

### Task 1.2: Database Session & Multi-Tenancy

**What:** Set up SQLAlchemy async session management with PostgreSQL, implement Row Level Security for multi-tenancy.

**Design:**

```python
# backend/app/db/session.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from sqlalchemy.orm import DeclarativeBase
from app.config import settings

engine = create_async_engine(
    settings.database_url,
    pool_size=settings.database_pool_size,
    max_overflow=settings.database_max_overflow,
    echo=settings.debug,
)

AsyncSessionLocal = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

class Base(DeclarativeBase):
    pass

async def init_db():
    async with engine.begin() as conn:
        await conn.execute(text("SELECT 1"))  # connectivity check

async def close_db():
    await engine.dispose()

async def get_db() -> AsyncSession:
    async with AsyncSessionLocal() as session:
        yield session

# backend/app/db/rls.py
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession

async def set_tenant_context(session: AsyncSession, organisation_id: str):
    """Set the current tenant for Row Level Security policies."""
    await session.execute(
        text("SET LOCAL app.current_organisation_id = :org_id"),
        {"org_id": str(organisation_id)},
    )
```

```sql
-- alembic/versions/001_foundation.py (migration SQL)

-- Enable RLS helper function
CREATE OR REPLACE FUNCTION current_org_id() RETURNS UUID AS $$
    SELECT NULLIF(current_setting('app.current_organisation_id', TRUE), '')::UUID;
$$ LANGUAGE SQL STABLE;

-- Organisation table
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    legal_name      TEXT,
    tax_id          TEXT,
    country_code    CHAR(2) NOT NULL,
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',
    reporting_standard TEXT NOT NULL DEFAULT 'gaap'
        CHECK (reporting_standard IN ('gaap', 'ifrs')),
    fiscal_year_end_month SMALLINT NOT NULL DEFAULT 12,
    jurisdiction_data JSONB NOT NULL DEFAULT '{}',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE organisation ENABLE ROW LEVEL SECURITY;
CREATE POLICY org_isolation ON organisation
    USING (id = current_org_id());

-- Org user table
CREATE TABLE org_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    user_id         UUID NOT NULL,
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    role            TEXT NOT NULL CHECK (role IN (
        'owner', 'admin', 'accountant', 'bookkeeper', 'viewer'
    )),
    permissions     JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, user_id)
);

ALTER TABLE org_user ENABLE ROW LEVEL SECURITY;
CREATE POLICY org_user_isolation ON org_user
    USING (organisation_id = current_org_id());
CREATE INDEX idx_org_user_org ON org_user(organisation_id);
```

**Testing:**
- `test_rls_prevents_cross_tenant_access` — User in org A cannot read org B's data.
- `test_set_tenant_context` — `current_org_id()` returns the correct UUID after SET LOCAL.
- `test_session_rollback_clears_tenant` — Tenant context is cleared after transaction rollback.
- `test_concurrent_sessions_isolated` — Two concurrent sessions with different tenants see different data.

### Task 1.3: Authentication & Authorization

**What:** Implement OAuth 2.0 / OIDC authentication middleware, role-based access control, and user management endpoints.

**Design:**

```python
# backend/app/auth/dependencies.py
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from jose import jwt, JWTError
from app.config import settings
from app.db.session import get_db
from app.models.organisation import OrgUser
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from dataclasses import dataclass
from typing import Literal

security = HTTPBearer()

@dataclass
class AuthenticatedUser:
    user_id: str
    email: str
    organisation_id: str
    role: Literal["owner", "admin", "accountant", "bookkeeper", "viewer"]
    org_user_id: str

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: AsyncSession = Depends(get_db),
) -> AuthenticatedUser:
    try:
        payload = jwt.decode(
            credentials.credentials,
            key=settings.oidc_jwks,  # cached JWKS
            algorithms=["RS256"],
            audience=settings.oidc_audience,
            issuer=settings.oidc_issuer_url,
        )
    except JWTError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)

    user_id = payload["sub"]
    org_id = payload.get("org_id")  # custom claim or header

    result = await db.execute(
        select(OrgUser).where(
            OrgUser.user_id == user_id,
            OrgUser.organisation_id == org_id,
            OrgUser.is_active == True,
        )
    )
    org_user = result.scalar_one_or_none()
    if not org_user:
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN)

    return AuthenticatedUser(
        user_id=user_id,
        email=org_user.email,
        organisation_id=str(org_id),
        role=org_user.role,
        org_user_id=str(org_user.id),
    )

def require_role(*roles: str):
    """Dependency that enforces role-based access."""
    async def check_role(user: AuthenticatedUser = Depends(get_current_user)):
        if user.role not in roles:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Role '{user.role}' insufficient. Required: {roles}",
            )
        return user
    return check_role
```

```python
# backend/app/api/v1/organisations.py
from fastapi import APIRouter, Depends
from app.auth.dependencies import get_current_user, require_role, AuthenticatedUser
from app.schemas.organisation import OrganisationCreate, OrganisationResponse

router = APIRouter(prefix="/organisations", tags=["organisations"])

@router.post("/", response_model=OrganisationResponse, status_code=201)
async def create_organisation(
    body: OrganisationCreate,
    user: AuthenticatedUser = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    ...

@router.get("/me", response_model=OrganisationResponse)
async def get_current_organisation(
    user: AuthenticatedUser = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    ...
```

**Testing:**
- `test_unauthenticated_request_returns_401` — Request without bearer token returns 401.
- `test_invalid_token_returns_401` — Malformed JWT returns 401.
- `test_valid_token_returns_user` — Valid JWT with matching org_user returns AuthenticatedUser.
- `test_role_enforcement_accountant_cannot_delete` — Accountant role blocked from owner-only endpoint.
- `test_viewer_read_only` — Viewer role can GET but not POST/PUT/DELETE.
- `test_inactive_user_returns_403` — User with `is_active=false` is denied.

### Task 1.4: Docker Compose & CI Setup

**What:** Create Docker Compose for local development (FastAPI, PostgreSQL, Redis for task queues), Makefile for common commands, and GitHub Actions CI pipeline.

**Design:**

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: accounting
      POSTGRES_USER: accounting
      POSTGRES_PASSWORD: accounting
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U accounting"]
      interval: 5s
      timeout: 5s
      retries: 5

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      ACCOUNTING_DATABASE_URL: postgresql+asyncpg://accounting:accounting@db:5432/accounting
      ACCOUNTING_ENVIRONMENT: development
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - ./backend:/app
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

volumes:
  pgdata:
```

```makefile
# Makefile
.PHONY: dev test migrate lint

dev:
	docker compose up --build

test:
	cd backend && python -m pytest tests/ -v --cov=app --cov-report=term-missing

migrate:
	cd backend && alembic upgrade head

migrate-create:
	cd backend && alembic revision --autogenerate -m "$(name)"

lint:
	cd backend && ruff check app/ tests/ && mypy app/

format:
	cd backend && ruff format app/ tests/
```

**Testing:**
- `test_docker_compose_starts` — `docker compose up` brings all services to healthy state.
- `test_database_connectivity` — Backend can reach PostgreSQL and execute a SELECT 1.
- `test_migrations_run_clean` — `alembic upgrade head` on an empty database completes without error.
- `test_migrations_reversible` — `alembic downgrade -1` and `upgrade head` succeeds.

---

## Phase 2: Chart of Accounts & General Ledger

### Definition of Done
- GAAP-compliant chart of accounts with hierarchy (adjacency list + optional ltree path).
- Fiscal year and period management with open/close controls.
- Immutable posting_group / posting tables with database-enforced immutability triggers.
- Double-entry balance constraint enforced (SUM(debits) = SUM(credits) per posting group).
- Journal entry draft/approve/post workflow.
- Audit log captures all mutations.
- Trial balance query returns correct results.

### Task 2.1: Chart of Accounts Model & API

**What:** Create the account table with GAAP-compliant type hierarchy, XBRL mapping, and CRUD endpoints. Seed a default chart of accounts template.

**Design:**

```python
# backend/app/models/account.py
from sqlalchemy import Column, String, Boolean, ForeignKey, Text, SmallInteger
from sqlalchemy.dialects.postgresql import UUID, JSONB
from app.db.session import Base
import uuid

class Account(Base):
    __tablename__ = "account"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    organisation_id = Column(UUID(as_uuid=True), ForeignKey("organisation.id"), nullable=False)
    code = Column(Text, nullable=False)
    name = Column(Text, nullable=False)
    account_type = Column(Text, nullable=False)  # asset, liability, equity, revenue, expense, contra_*
    account_subtype = Column(Text, nullable=True)
    parent_id = Column(UUID(as_uuid=True), ForeignKey("account.id"), nullable=True)
    currency_code = Column(String(3), nullable=False, default="USD")
    normal_balance = Column(Text, nullable=False)  # debit or credit
    is_active = Column(Boolean, nullable=False, default=True)
    is_system = Column(Boolean, nullable=False, default=False)
    xbrl_mapping = Column(JSONB, nullable=True)
    tax_config = Column(JSONB, nullable=False, server_default="{}")
    description = Column(Text, nullable=True)

# backend/app/schemas/account.py
from pydantic import BaseModel, Field
from uuid import UUID
from typing import Literal, Any

class AccountCreate(BaseModel):
    code: str = Field(..., min_length=1, max_length=20, examples=["1000"])
    name: str = Field(..., min_length=1, max_length=200, examples=["Cash and Cash Equivalents"])
    account_type: Literal[
        "asset", "liability", "equity", "revenue", "expense",
        "contra_asset", "contra_liability", "contra_equity",
        "contra_revenue", "contra_expense"
    ]
    account_subtype: str | None = None
    parent_id: UUID | None = None
    currency_code: str = "USD"
    normal_balance: Literal["debit", "credit"]
    description: str | None = None
    xbrl_mapping: dict[str, Any] | None = None
    tax_config: dict[str, Any] = {}

class AccountResponse(BaseModel):
    id: UUID
    code: str
    name: str
    account_type: str
    account_subtype: str | None
    parent_id: UUID | None
    currency_code: str
    normal_balance: str
    is_active: bool
    is_system: bool
    xbrl_mapping: dict | None
    tax_config: dict

class AccountTree(BaseModel):
    """Recursive tree representation for chart of accounts display."""
    id: UUID
    code: str
    name: str
    account_type: str
    normal_balance: str
    is_active: bool
    children: list["AccountTree"] = []
```

```sql
-- Migration: Chart of accounts
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
    description     TEXT,
    xbrl_mapping    JSONB,
    tax_config      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, code)
);

ALTER TABLE account ENABLE ROW LEVEL SECURITY;
CREATE POLICY account_isolation ON account
    USING (organisation_id = current_org_id());

CREATE INDEX idx_account_org ON account(organisation_id);
CREATE INDEX idx_account_parent ON account(parent_id);
CREATE INDEX idx_account_type ON account(organisation_id, account_type);
CREATE INDEX idx_account_xbrl ON account USING gin (xbrl_mapping) WHERE xbrl_mapping IS NOT NULL;
```

**Testing:**
- `test_create_account_validates_type` — Invalid account_type is rejected with 422.
- `test_account_code_unique_per_org` — Duplicate code within same org returns 409; different org allows same code.
- `test_chart_of_accounts_tree` — GET /accounts/tree returns nested hierarchy matching parent_id relationships.
- `test_deactivate_account_with_postings` — Account with existing postings can be deactivated but not deleted.
- `test_system_account_protected` — System accounts (is_system=true) cannot be deleted or renamed.
- `test_default_chart_seeded` — New organisation gets a default GAAP chart of accounts with standard categories.

### Task 2.2: Fiscal Year & Period Management

**What:** Create fiscal_year and fiscal_period tables with create/close operations. Support standard 12-month and 13-period (adjustment) configurations.

**Design:**

```python
# backend/app/services/fiscal.py
from dataclasses import dataclass
from datetime import date
from dateutil.relativedelta import relativedelta
from uuid import UUID

@dataclass
class FiscalPeriodSpec:
    period_number: int
    start_date: date
    end_date: date
    is_adjustment: bool

def generate_fiscal_periods(
    start_date: date,
    end_date: date,
    include_adjustment: bool = True,
) -> list[FiscalPeriodSpec]:
    """Generate monthly fiscal periods for a fiscal year.

    Args:
        start_date: First day of the fiscal year.
        end_date: Last day of the fiscal year.
        include_adjustment: If True, add a 13th adjustment period.

    Returns:
        List of FiscalPeriodSpec, one per month plus optional adjustment.
    """
    periods = []
    current = start_date
    period_num = 1
    while current <= end_date:
        period_end = min(current + relativedelta(months=1) - relativedelta(days=1), end_date)
        periods.append(FiscalPeriodSpec(
            period_number=period_num,
            start_date=current,
            end_date=period_end,
            is_adjustment=False,
        ))
        current = period_end + relativedelta(days=1)
        period_num += 1

    if include_adjustment:
        periods.append(FiscalPeriodSpec(
            period_number=period_num,
            start_date=end_date,
            end_date=end_date,
            is_adjustment=True,
        ))
    return periods

async def close_fiscal_period(
    db: AsyncSession,
    period_id: UUID,
    closed_by: UUID,
) -> None:
    """Close a fiscal period. Prevents new postings to this period.

    Raises:
        ValueError: If period has unposted draft entries.
        ValueError: If previous periods are still open.
    """
    ...
```

```sql
-- Migration: Fiscal periods
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

ALTER TABLE fiscal_year ENABLE ROW LEVEL SECURITY;
ALTER TABLE fiscal_period ENABLE ROW LEVEL SECURITY;
CREATE POLICY fy_isolation ON fiscal_year USING (organisation_id = current_org_id());
CREATE POLICY fp_isolation ON fiscal_period USING (organisation_id = current_org_id());
```

**Testing:**
- `test_create_fiscal_year_generates_12_periods` — Creating FY2026 (Jan-Dec) produces 12 monthly periods.
- `test_13th_adjustment_period` — With `include_adjustment=True`, a 13th period is created on the last day.
- `test_close_period_blocks_new_postings` — Attempting to post to a closed period raises error.
- `test_cannot_close_period_with_drafts` — Closing a period with draft entries raises ValueError.
- `test_must_close_periods_in_order` — Closing period 3 when period 2 is open raises error.
- `test_non_calendar_fiscal_year` — FY starting April 1 generates correct date ranges.

### Task 2.3: Immutable Posting Engine

**What:** Create the immutable posting_group and posting tables with database triggers preventing UPDATE/DELETE. Implement the double-entry posting service that enforces debit=credit balance.

**Design:**

```python
# backend/app/services/ledger.py
from dataclasses import dataclass
from decimal import Decimal
from uuid import UUID
from datetime import date
from typing import Literal
import hashlib
import json

@dataclass
class PostingLine:
    account_id: UUID
    description: str | None
    debit_amount: Decimal
    credit_amount: Decimal
    department_id: UUID | None = None
    project_id: UUID | None = None
    contact_id: UUID | None = None
    tax_code: str | None = None

@dataclass
class PostingGroupRequest:
    posting_date: date
    description: str
    source_type: Literal[
        "manual", "bank_feed", "invoice", "bill", "payment",
        "payroll", "depreciation", "accrual", "adjustment",
        "closing", "reversal", "ai_generated", "revenue_recognition"
    ]
    source_document_id: UUID | None
    lines: list[PostingLine]
    currency_code: str = "USD"
    exchange_rate: Decimal = Decimal("1.0")
    ai_generated: bool = False
    ai_confidence: Decimal | None = None
    ai_model_id: str | None = None
    ai_explanation: str | None = None

class LedgerService:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def create_posting(
        self,
        org_id: UUID,
        request: PostingGroupRequest,
        created_by: UUID,
    ) -> PostingGroup:
        """Create an immutable posting group with balanced debit/credit lines.

        Validates:
            1. SUM(debits) == SUM(credits)
            2. All account_ids exist and are active
            3. Fiscal period for posting_date is open
            4. At least 2 lines (one debit, one credit)

        Returns:
            The created PostingGroup with its sequence_number.

        Raises:
            ValueError: If debits != credits.
            ValueError: If fiscal period is closed.
            ValueError: If any account is inactive.
        """
        # Validate balance
        total_debit = sum(line.debit_amount for line in request.lines)
        total_credit = sum(line.credit_amount for line in request.lines)
        if total_debit != total_credit:
            raise ValueError(
                f"Posting group does not balance: "
                f"debits={total_debit}, credits={total_credit}"
            )

        # Compute content hash for tamper detection
        content_hash = self._compute_hash(org_id, request)

        # Get next sequence number
        sequence_number = await self._next_sequence(org_id)

        # Resolve fiscal period
        fiscal_period = await self._resolve_period(org_id, request.posting_date)
        if fiscal_period.is_closed:
            raise ValueError(f"Fiscal period {fiscal_period.period_number} is closed")

        # Insert posting_group and postings in a single transaction
        ...
        return posting_group

    async def reverse_posting(
        self,
        org_id: UUID,
        posting_group_id: UUID,
        reversal_date: date,
        reason: str,
        created_by: UUID,
    ) -> PostingGroup:
        """Create a reversing entry for an existing posting group.

        Creates a new posting group with all debit/credit amounts swapped.
        Updates the original posting group's reversed_by reference.
        """
        ...

    def _compute_hash(self, org_id: UUID, request: PostingGroupRequest) -> str:
        content = json.dumps({
            "org_id": str(org_id),
            "date": request.posting_date.isoformat(),
            "description": request.description,
            "lines": [
                {
                    "account_id": str(l.account_id),
                    "debit": str(l.debit_amount),
                    "credit": str(l.credit_amount),
                }
                for l in request.lines
            ],
        }, sort_keys=True)
        return hashlib.sha256(content.encode()).hexdigest()
```

```sql
-- Migration: Immutable posting tables
CREATE TABLE posting_group (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    sequence_number BIGINT NOT NULL,
    posting_date    DATE NOT NULL,
    fiscal_period_id UUID NOT NULL REFERENCES fiscal_period(id),
    description     TEXT NOT NULL,
    source_type     TEXT NOT NULL CHECK (source_type IN (
        'manual', 'bank_feed', 'invoice', 'bill', 'payment',
        'payroll', 'depreciation', 'accrual', 'adjustment',
        'closing', 'reversal', 'ai_generated', 'revenue_recognition'
    )),
    source_document_id UUID,
    reversal_of     UUID REFERENCES posting_group(id),
    reversed_by     UUID,
    is_reversal     BOOLEAN NOT NULL DEFAULT false,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    exchange_rate   NUMERIC(18,8) DEFAULT 1.0,
    ai_generated    BOOLEAN NOT NULL DEFAULT false,
    ai_confidence   NUMERIC(5,4),
    ai_model_id     TEXT,
    ai_explanation  TEXT,
    status          TEXT NOT NULL DEFAULT 'posted' CHECK (status IN (
        'pending_review', 'approved', 'posted'
    )),
    approved_by     UUID REFERENCES org_user(id),
    approved_at     TIMESTAMPTZ,
    created_by      UUID NOT NULL REFERENCES org_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    content_hash    TEXT NOT NULL,
    UNIQUE (organisation_id, sequence_number)
);

-- Immutability trigger
CREATE OR REPLACE FUNCTION prevent_posting_group_mutation()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'UPDATE' THEN
        IF NEW.reversed_by IS NOT NULL AND OLD.reversed_by IS NULL
           AND NEW.id = OLD.id THEN
            RETURN NEW;
        END IF;
        RAISE EXCEPTION 'posting_group is immutable';
    ELSIF TG_OP = 'DELETE' THEN
        RAISE EXCEPTION 'posting_group is immutable';
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_posting_group_immutable
    BEFORE UPDATE OR DELETE ON posting_group
    FOR EACH ROW EXECUTE FUNCTION prevent_posting_group_mutation();

CREATE TABLE posting (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    posting_group_id UUID NOT NULL REFERENCES posting_group(id),
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT chk_posting_debit_or_credit CHECK (
        (debit_amount > 0 AND credit_amount = 0) OR
        (credit_amount > 0 AND debit_amount = 0)
    )
);

CREATE OR REPLACE FUNCTION prevent_posting_mutation()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'posting is immutable';
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_posting_immutable
    BEFORE UPDATE OR DELETE ON posting
    FOR EACH ROW EXECUTE FUNCTION prevent_posting_mutation();

ALTER TABLE posting_group ENABLE ROW LEVEL SECURITY;
CREATE POLICY pg_isolation ON posting_group USING (organisation_id = current_org_id());

CREATE INDEX idx_pg_org_date ON posting_group(organisation_id, posting_date);
CREATE INDEX idx_pg_period ON posting_group(fiscal_period_id);
CREATE INDEX idx_posting_group_fk ON posting(posting_group_id);
CREATE INDEX idx_posting_account ON posting(account_id);
```

**Testing:**
- `test_balanced_posting_succeeds` — Posting with equal debits and credits is accepted.
- `test_unbalanced_posting_rejected` — Posting with debits != credits raises ValueError.
- `test_immutability_trigger_blocks_update` — Direct SQL UPDATE on posting_group raises exception.
- `test_immutability_trigger_blocks_delete` — Direct SQL DELETE on posting raises exception.
- `test_reversal_allowed_update` — Setting reversed_by on posting_group is the only permitted update.
- `test_reverse_posting_creates_mirror` — Reversal swaps all debit/credit amounts.
- `test_content_hash_deterministic` — Same inputs produce the same SHA-256 hash.
- `test_posting_to_closed_period_rejected` — Posting to a closed fiscal period raises ValueError.
- `test_sequence_number_monotonic` — Sequential postings get incrementing sequence numbers.
- `test_posting_to_inactive_account_rejected` — Posting to a deactivated account raises error.

### Task 2.4: Journal Entry Workflow

**What:** Create the journal entry draft/review/approve/post workflow on top of the immutable posting engine. Journal entries in draft status are mutable; once posted, they become immutable posting groups.

**Design:**

```python
# backend/app/models/journal.py
class JournalEntry(Base):
    __tablename__ = "journal_entry"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    organisation_id = Column(UUID(as_uuid=True), ForeignKey("organisation.id"), nullable=False)
    entry_number = Column(BigInteger, nullable=False)
    entry_date = Column(Date, nullable=False)
    fiscal_period_id = Column(UUID(as_uuid=True), nullable=False)
    description = Column(Text, nullable=False)
    source_type = Column(Text, nullable=False)
    source_id = Column(UUID(as_uuid=True), nullable=True)
    status = Column(Text, nullable=False, default="draft")
    # ... currency, reversal, AI fields ...
    posting_group_id = Column(UUID(as_uuid=True), ForeignKey("posting_group.id"), nullable=True)

# backend/app/services/journal.py
class JournalService:
    async def create_draft(self, org_id: UUID, data: JournalEntryCreate, user_id: UUID) -> JournalEntry:
        """Create a mutable draft journal entry."""
        ...

    async def update_draft(self, org_id: UUID, entry_id: UUID, data: JournalEntryUpdate) -> JournalEntry:
        """Update a draft entry. Only drafts can be modified."""
        ...

    async def submit_for_review(self, org_id: UUID, entry_id: UUID) -> JournalEntry:
        """Move draft to pending_review status."""
        ...

    async def approve(self, org_id: UUID, entry_id: UUID, approved_by: UUID) -> JournalEntry:
        """Approve a pending entry."""
        ...

    async def post(self, org_id: UUID, entry_id: UUID, posted_by: UUID) -> JournalEntry:
        """Post an approved entry. Creates an immutable posting_group.
        After posting, the journal entry becomes read-only."""
        ...
```

**Testing:**
- `test_draft_is_mutable` — Draft entries can be updated (lines added/removed).
- `test_posted_entry_is_immutable` — Attempting to update a posted entry raises error.
- `test_post_creates_posting_group` — Posting a journal entry creates a corresponding posting_group.
- `test_workflow_sequence` — draft → pending_review → approved → posted sequence enforced.
- `test_cannot_skip_approval` — Direct draft → posted transition is rejected (requires approval).
- `test_reversal_creates_new_entry` — Reversing a posted entry creates a new journal entry with opposite amounts.

### Task 2.5: Audit Log

**What:** Create the audit_log table and middleware that automatically captures all create/update/delete operations with before/after change diffs.

**Design:**

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    user_id         UUID,
    action          TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,
    context         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

-- Create partitions per quarter
CREATE TABLE audit_log_2026_q1 PARTITION OF audit_log
    FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');
CREATE TABLE audit_log_2026_q2 PARTITION OF audit_log
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');

CREATE INDEX idx_audit_org_date ON audit_log(organisation_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

```python
# backend/app/services/audit.py
from typing import Any

class AuditService:
    async def log(
        self,
        db: AsyncSession,
        org_id: UUID,
        user_id: UUID | None,
        action: str,
        entity_type: str,
        entity_id: UUID,
        changes: dict[str, Any] | None = None,
        context: dict[str, Any] | None = None,
    ) -> None:
        """Write an audit log entry."""
        ...

    def compute_changes(
        self,
        old: dict[str, Any],
        new: dict[str, Any],
    ) -> dict[str, dict[str, Any]]:
        """Compute field-level change diff: {field: {old: x, new: y}}."""
        ...
```

**Testing:**
- `test_create_generates_audit_log` — Creating an account generates an audit entry with action='create'.
- `test_update_captures_before_after` — Updating an account captures `{field: {old: x, new: y}}` diff.
- `test_audit_log_includes_user_id` — Every audit entry has the acting user's ID.
- `test_audit_log_partitioned` — Entries are stored in the correct quarterly partition.
- `test_posting_logged_as_immutable` — Posting a journal entry generates an audit entry; subsequent mutation attempts do not generate additional logs.

---

## Phase 3: Invoicing, AP/AR & Payments

### Definition of Done
- Unified invoice table supports sales invoices, purchase bills, credit notes, and debit notes.
- Invoice CRUD with line items, tax calculation, and status workflow.
- Payments can be created and allocated against invoices.
- Posting to GL: issuing an invoice creates the AR/AP journal entry automatically.
- Contacts (customers, vendors) with CRUD and balance tracking.
- Aging report for AR and AP.

### Task 3.1: Contact Management

**What:** Create the contact model supporting customers, vendors, employees, and dual-role contacts (customer+vendor). Addresses and bank details stored as JSONB.

**Design:**

```python
# backend/app/schemas/contact.py
from pydantic import BaseModel, Field
from uuid import UUID
from typing import Literal

class Address(BaseModel):
    type: Literal["billing", "shipping", "registered"]
    line1: str
    line2: str | None = None
    city: str
    state_province: str | None = None
    postal_code: str | None = None
    country_code: str = Field(..., min_length=2, max_length=2)
    is_primary: bool = False

class BankDetails(BaseModel):
    account_name: str | None = None
    iban: str | None = None
    bic: str | None = None
    routing_number: str | None = None
    account_number: str | None = None

class ContactCreate(BaseModel):
    contact_types: list[Literal["customer", "vendor", "employee", "other"]]
    name: str
    legal_name: str | None = None
    email: str | None = None
    phone: str | None = None
    tax_id: str | None = None
    currency_code: str = "USD"
    payment_terms_days: int = 30
    credit_limit: float | None = None
    addresses: list[Address] = []
    bank_details: BankDetails | None = None
    jurisdiction_data: dict = {}
```

```sql
CREATE TABLE contact (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
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
    jurisdiction_data JSONB NOT NULL DEFAULT '{}',
    bank_details    JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE contact ENABLE ROW LEVEL SECURITY;
CREATE POLICY contact_isolation ON contact USING (organisation_id = current_org_id());
CREATE INDEX idx_contact_org ON contact(organisation_id);
CREATE INDEX idx_contact_types ON contact USING gin (contact_types);
```

**Testing:**
- `test_create_customer` — Create contact with type=['customer'] succeeds.
- `test_dual_role_contact` — Contact with types=['customer','vendor'] is valid.
- `test_addresses_stored_as_jsonb` — Multiple addresses are persisted and retrieved correctly.
- `test_search_by_contact_type` — Filtering contacts by type uses GIN index.
- `test_contact_rls_enforced` — Contact from org A not visible to org B.

### Task 3.2: Unified Invoice Model

**What:** Create a unified invoice table handling both sales invoices and purchase bills with line items, tax calculation, and PEPPOL e-invoice metadata.

**Design:**

```python
# backend/app/schemas/invoice.py
class InvoiceLineCreate(BaseModel):
    description: str
    account_id: UUID
    quantity: Decimal = Decimal("1")
    unit_price: Decimal
    discount_pct: Decimal = Decimal("0")
    tax_rate_id: UUID | None = None

class InvoiceCreate(BaseModel):
    direction: Literal["sales", "purchase"]
    document_type: Literal["invoice", "credit_note", "debit_note"] = "invoice"
    contact_id: UUID
    issue_date: date
    due_date: date
    currency_code: str = "USD"
    reference: str | None = None
    notes: str | None = None
    lines: list[InvoiceLineCreate] = Field(..., min_length=1)
```

```python
# backend/app/services/invoicing.py
class InvoicingService:
    async def create_invoice(self, org_id: UUID, data: InvoiceCreate, user_id: UUID) -> Invoice:
        """Create a draft invoice with computed line totals and tax."""
        ...

    async def issue_invoice(self, org_id: UUID, invoice_id: UUID, user_id: UUID) -> Invoice:
        """Issue the invoice: assign invoice number, create AR/AP journal entry.

        For sales invoices:
            DR Accounts Receivable  (total)
                CR Revenue          (subtotal)
                CR Tax Payable      (tax_total)

        For purchase bills:
            DR Expense/Asset        (subtotal)
            DR Tax Receivable       (tax_total)
                CR Accounts Payable (total)
        """
        ...

    async def void_invoice(self, org_id: UUID, invoice_id: UUID, user_id: UUID) -> Invoice:
        """Void an invoice: create reversing journal entry."""
        ...

    def _compute_line_totals(self, line: InvoiceLineCreate, tax_rate: Decimal | None) -> dict:
        """Compute line_total and tax_amount from quantity, unit_price, discount, tax_rate."""
        subtotal = line.quantity * line.unit_price * (1 - line.discount_pct / 100)
        tax_amount = subtotal * (tax_rate / 100) if tax_rate else Decimal("0")
        return {"line_total": subtotal, "tax_amount": tax_amount}
```

**Testing:**
- `test_create_sales_invoice` — Draft sales invoice with 2 lines computes correct subtotal, tax, total.
- `test_issue_invoice_creates_journal_entry` — Issuing creates a balanced AR journal entry.
- `test_purchase_bill_creates_ap_entry` — Issuing a purchase bill creates AP journal entry.
- `test_credit_note_reverses` — Credit note creates opposite journal entry.
- `test_invoice_number_sequential` — Invoice numbers are sequential per org and direction.
- `test_void_invoice_creates_reversal` — Voiding creates a reversing posting group.
- `test_cannot_modify_issued_invoice` — Issued invoice cannot be edited (only voided).
- `test_tax_calculation_accuracy` — Tax computed at line level matches expected values to 4 decimal places.

### Task 3.3: Payment Processing & Allocation

**What:** Create payment records, allocate payments against invoices, and generate the corresponding GL postings.

**Design:**

```python
# backend/app/services/payment.py
class PaymentService:
    async def create_payment(
        self, org_id: UUID, data: PaymentCreate, user_id: UUID
    ) -> Payment:
        """Record a payment and allocate against invoices.

        For payment received (sales):
            DR Cash/Bank Account    (amount)
                CR Accounts Receivable (amount)

        For payment made (purchase):
            DR Accounts Payable     (amount)
                CR Cash/Bank Account (amount)

        Updates invoice.amount_paid and invoice.status accordingly.
        """
        ...

    async def allocate_payment(
        self, payment_id: UUID, allocations: list[PaymentAllocation]
    ) -> None:
        """Allocate a payment across one or more invoices.

        Validates:
            - Sum of allocations <= payment.amount
            - Each allocation <= invoice.amount_due
            - Invoice and payment are same direction
        """
        ...
```

**Testing:**
- `test_payment_received_creates_journal` — Receiving $1000 creates DR Cash / CR AR posting.
- `test_payment_fully_allocates_invoice` — Allocating full amount marks invoice as 'paid'.
- `test_partial_payment` — Partial allocation marks invoice as 'partially_paid'.
- `test_overpayment_rejected` — Allocating more than invoice.amount_due is rejected.
- `test_payment_currency_mismatch` — Payment in different currency uses exchange rate for base conversion.

### Task 3.4: Tax Rate Configuration

**What:** Create tax_rate table with jurisdiction-specific configuration in JSONB.

**Design:**

```sql
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
    jurisdiction_config JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Testing:**
- `test_create_vat_rate` — Create 20% UK VAT with PEPPOL category code 'S'.
- `test_create_sales_tax` — Create CA sales tax at 7.25% with nexus states.
- `test_tax_applied_to_invoice_line` — Tax rate correctly applied during invoice calculation.
- `test_exempt_and_zero_rated` — Exempt and zero-rated taxes produce correct amounts (0).

---

## Phase 4: Bank Feeds & Reconciliation

### Definition of Done
- Plaid integration: link bank account, import transactions, handle webhooks.
- OFX/CSV manual import as fallback.
- Bank transaction storage with full provider data in JSONB.
- Reconciliation workflow: match bank transactions to journal entries (manual, rule-based, AI-suggested).
- Bank reconciliation report showing matched/unmatched/excluded transactions.

### Task 4.1: Bank Connection & Account Linking

**What:** Implement Plaid Link integration to connect bank accounts, store connection metadata, and map bank accounts to GL accounts.

**Design:**

```python
# backend/app/integrations/plaid_adapter.py
from plaid.api import plaid_api
from plaid.model import *

class PlaidAdapter:
    def __init__(self, client_id: str, secret: str, environment: str):
        ...

    async def create_link_token(self, org_id: str, user_id: str) -> str:
        """Create a Plaid Link token for the frontend."""
        ...

    async def exchange_public_token(self, public_token: str) -> str:
        """Exchange public token for access token after user completes Link."""
        ...

    async def get_accounts(self, access_token: str) -> list[PlaidBankAccount]:
        """Fetch bank accounts for a linked item."""
        ...

    async def sync_transactions(
        self, access_token: str, cursor: str | None = None
    ) -> TransactionSyncResult:
        """Fetch new, modified, and removed transactions since cursor."""
        ...
```

```sql
CREATE TABLE bank_connection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    provider        TEXT NOT NULL CHECK (provider IN ('plaid', 'open_banking', 'fdx', 'manual')),
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
    ledger_account_id UUID REFERENCES account(id),
    current_balance NUMERIC(19,4),
    available_balance NUMERIC(19,4),
    balance_updated_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Testing:**
- `test_create_link_token` — Plaid sandbox returns valid link token.
- `test_exchange_public_token` — Public token exchange returns access token.
- `test_fetch_accounts` — Account listing returns checking/savings with balances.
- `test_map_bank_to_ledger_account` — Mapping a bank account to GL account persists correctly.
- `test_reconnect_expired_connection` — Re-authentication flow updates provider_config.

### Task 4.2: Transaction Import & Storage

**What:** Import bank transactions via Plaid sync API and OFX/CSV fallback, storing raw provider data in JSONB alongside extracted fields.

**Design:**

```python
# backend/app/services/bank_sync.py
class BankSyncService:
    async def sync_transactions(
        self, org_id: UUID, bank_connection_id: UUID
    ) -> SyncResult:
        """Sync transactions for a bank connection.

        Uses Plaid's transaction sync API with cursor-based pagination.
        Stores new transactions, updates modified ones, removes deleted ones.
        """
        ...

    async def import_ofx(
        self, org_id: UUID, bank_account_id: UUID, file_content: bytes
    ) -> int:
        """Import transactions from OFX file. Returns count imported."""
        ...

    async def import_csv(
        self, org_id: UUID, bank_account_id: UUID,
        file_content: bytes, mapping: CsvColumnMapping
    ) -> int:
        """Import transactions from CSV with user-defined column mapping."""
        ...
```

```sql
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
    provider_data   JSONB NOT NULL DEFAULT '{}',
    ai_categorisation JSONB,
    reconciliation_status TEXT NOT NULL DEFAULT 'unmatched' CHECK (reconciliation_status IN (
        'unmatched', 'ai_suggested', 'matched', 'reconciled', 'excluded'
    )),
    matched_posting_group_id UUID REFERENCES posting_group(id),
    reconciled_at   TIMESTAMPTZ,
    reconciled_by   UUID REFERENCES org_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (bank_account_id, provider_txn_id)
);

ALTER TABLE bank_transaction ENABLE ROW LEVEL SECURITY;
CREATE POLICY bt_isolation ON bank_transaction USING (organisation_id = current_org_id());
CREATE INDEX idx_bank_txn_org_date ON bank_transaction(organisation_id, transaction_date);
CREATE INDEX idx_bank_txn_status ON bank_transaction(organisation_id, reconciliation_status);
```

**Testing:**
- `test_plaid_sync_imports_transactions` — Sandbox sync returns transactions with Plaid categories.
- `test_duplicate_transaction_deduped` — Re-importing same provider_txn_id does not create duplicate.
- `test_ofx_import_parses_correctly` — OFX file parsed into correct date, amount, description.
- `test_csv_import_with_mapping` — CSV with custom columns mapped correctly.
- `test_provider_data_preserved` — Full Plaid response stored in provider_data JSONB.

### Task 4.3: Reconciliation Workflow

**What:** Implement bank reconciliation: manual matching, rule-based matching, and a reconciliation report showing the bank-to-book difference.

**Design:**

```python
# backend/app/services/reconciliation.py
class ReconciliationService:
    async def match_transaction(
        self,
        org_id: UUID,
        bank_txn_id: UUID,
        posting_group_id: UUID,
        user_id: UUID,
    ) -> None:
        """Manually match a bank transaction to an existing posting group."""
        ...

    async def create_and_match(
        self,
        org_id: UUID,
        bank_txn_id: UUID,
        account_id: UUID,
        user_id: UUID,
    ) -> PostingGroup:
        """Create a new posting from a bank transaction and reconcile it.

        DR/CR Bank Account (per bank_account.ledger_account_id)
        CR/DR Target Account (per account_id parameter)
        """
        ...

    async def exclude_transaction(
        self, org_id: UUID, bank_txn_id: UUID, reason: str, user_id: UUID
    ) -> None:
        """Mark a bank transaction as excluded from reconciliation."""
        ...

    async def auto_match(self, org_id: UUID, bank_account_id: UUID) -> AutoMatchResult:
        """Rule-based auto-matching: match by amount+date+description to unreconciled postings."""
        ...

    async def get_reconciliation_report(
        self, org_id: UUID, bank_account_id: UUID, as_of_date: date
    ) -> ReconciliationReport:
        """Bank reconciliation report: bank balance, book balance, unreconciled items, difference."""
        ...
```

**Testing:**
- `test_manual_match_updates_status` — Matching sets reconciliation_status to 'reconciled'.
- `test_create_and_match_creates_posting` — Creates a balanced posting and links to bank transaction.
- `test_exclude_transaction` — Excluded transaction no longer appears in unmatched list.
- `test_auto_match_exact_amount_date` — Exact amount+date match reconciles automatically.
- `test_reconciliation_report_balances` — Report shows correct bank balance, book balance, and difference.
- `test_cannot_reconcile_to_wrong_bank_account` — Matching to a posting for a different bank account is rejected.

---

## Phase 5: AI Categorisation & Journal Drafting

### Definition of Done
- LLM abstraction layer supports Claude, GPT-4o, and Ollama.
- Transaction categorisation: AI suggests GL account for unmatched bank transactions.
- Feedback loop: accepted/corrected/rejected feedback stored for model improvement.
- AI journal entry drafting from invoices and structured inputs.
- Categorisation rules: user-defined and AI-learned.
- AI confidence thresholds: high-confidence suggestions can auto-post; low-confidence requires review.

### Task 5.1: LLM Abstraction Layer

**What:** Create a model-agnostic LLM client supporting multiple providers with a unified interface for categorisation and generation tasks.

**Design:**

```python
# backend/app/ai/llm_client.py
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Literal

@dataclass
class LLMResponse:
    content: str
    model_id: str
    model_version: str
    input_tokens: int
    output_tokens: int
    latency_ms: float

class LLMClient(ABC):
    @abstractmethod
    async def complete(
        self,
        system_prompt: str,
        user_prompt: str,
        temperature: float = 0.0,
        max_tokens: int = 1024,
        response_format: type | None = None,  # Pydantic model for structured output
    ) -> LLMResponse:
        ...

class AnthropicClient(LLMClient):
    def __init__(self, api_key: str, model: str = "claude-sonnet-4-20250514"):
        import anthropic
        self.client = anthropic.AsyncAnthropic(api_key=api_key)
        self.model = model

    async def complete(self, system_prompt, user_prompt, **kwargs) -> LLMResponse:
        ...

class OpenAIClient(LLMClient):
    def __init__(self, api_key: str, model: str = "gpt-4o"):
        ...

class OllamaClient(LLMClient):
    def __init__(self, base_url: str = "http://localhost:11434", model: str = "llama3"):
        ...

def create_llm_client(provider: str, **kwargs) -> LLMClient:
    """Factory function to create an LLM client based on provider setting."""
    clients = {
        "anthropic": AnthropicClient,
        "openai": OpenAIClient,
        "ollama": OllamaClient,
    }
    return clients[provider](**kwargs)
```

**Testing:**
- `test_anthropic_client_returns_response` — AnthropicClient returns structured LLMResponse.
- `test_openai_client_returns_response` — OpenAIClient returns structured LLMResponse.
- `test_factory_creates_correct_client` — `create_llm_client("anthropic")` returns AnthropicClient.
- `test_structured_output_parsing` — Response parsed into Pydantic model when response_format specified.
- `test_client_retries_on_rate_limit` — Rate limit errors trigger exponential backoff retry.

### Task 5.2: Transaction Categoriser

**What:** AI-powered transaction categorisation that suggests a GL account for each unmatched bank transaction based on the organisation's chart of accounts, transaction history, and categorisation rules.

**Design:**

```python
# backend/app/ai/categoriser.py
from pydantic import BaseModel
from decimal import Decimal
from uuid import UUID

class CategorisationSuggestion(BaseModel):
    account_id: UUID
    account_code: str
    account_name: str
    confidence: Decimal
    reasoning: str
    alternatives: list[dict]  # [{account_id, code, confidence}]

class TransactionCategoriser:
    def __init__(self, llm: LLMClient, db: AsyncSession):
        self.llm = llm
        self.db = db

    async def categorise(
        self,
        org_id: UUID,
        bank_transaction: BankTransaction,
    ) -> CategorisationSuggestion:
        """Categorise a bank transaction using few-shot LLM reasoning.

        Strategy:
            1. Check user-defined rules first (exact match, pattern match).
            2. If no rule matches, query similar historical categorisations.
            3. Build few-shot prompt with chart of accounts + similar examples.
            4. Call LLM for categorisation with structured output.
            5. Store suggestion on bank_transaction.ai_categorisation.
        """
        # Step 1: Rule-based check
        rule_match = await self._check_rules(org_id, bank_transaction)
        if rule_match:
            return rule_match

        # Step 2: Find similar historical categorisations
        examples = await self._find_similar_transactions(org_id, bank_transaction)

        # Step 3: Build prompt
        chart_of_accounts = await self._get_active_accounts(org_id)
        prompt = self._build_categorisation_prompt(
            bank_transaction, chart_of_accounts, examples
        )

        # Step 4: Call LLM
        response = await self.llm.complete(
            system_prompt=CATEGORISATION_SYSTEM_PROMPT,
            user_prompt=prompt,
            response_format=CategorisationSuggestion,
        )

        return CategorisationSuggestion.model_validate_json(response.content)

    async def batch_categorise(
        self,
        org_id: UUID,
        transaction_ids: list[UUID],
    ) -> list[CategorisationSuggestion]:
        """Categorise multiple transactions efficiently."""
        ...

CATEGORISATION_SYSTEM_PROMPT = """You are an expert bookkeeper categorising bank transactions
for a business. Given a transaction description, amount, merchant, and the company's chart of
accounts, determine the most appropriate GL account.

Rules:
- Always prefer the most specific account available.
- Consider the transaction amount and direction (positive=inflow, negative=outflow).
- Use historical categorisation examples as guidance for consistency.
- Provide confidence as a decimal 0.0-1.0.
- Explain your reasoning briefly.
- Suggest up to 2 alternatives with their confidence scores.
"""
```

**Testing:**
- `test_categorise_rent_payment` — "WEWORK RENT MAY" categorised to Office Rent (6100) with >0.8 confidence.
- `test_categorise_with_rule_match` — User-defined rule for "STRIPE" bypasses LLM call.
- `test_batch_categorise_efficiency` — Batch of 50 transactions completes with shared context.
- `test_low_confidence_flagged` — Ambiguous transaction gets <0.5 confidence and alternatives.
- `test_categorisation_stored_on_transaction` — Suggestion persisted in ai_categorisation JSONB.

### Task 5.3: AI Feedback Loop

**What:** Capture user feedback on AI categorisations (accepted, corrected, rejected) and use it to improve future suggestions. Store feedback as immutable training data.

**Design:**

```python
# backend/app/ai/feedback.py
class AIFeedbackService:
    async def record_feedback(
        self,
        org_id: UUID,
        bank_txn_id: UUID,
        feedback_type: Literal["accepted", "corrected", "rejected"],
        correct_account_id: UUID | None,
        user_id: UUID,
    ) -> None:
        """Record user feedback on an AI categorisation suggestion.

        For 'accepted': the AI suggestion was correct.
        For 'corrected': user selected a different account.
        For 'rejected': user rejected the suggestion entirely.

        Updates categorisation_rule table for AI-learned rules when patterns emerge.
        """
        ...

    async def get_accuracy_metrics(
        self,
        org_id: UUID,
        model_id: str | None = None,
        period_days: int = 30,
    ) -> AccuracyMetrics:
        """Calculate AI categorisation accuracy over a period."""
        ...
```

```sql
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
    feedback_data   JSONB NOT NULL,
    user_id         UUID NOT NULL REFERENCES org_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE categorisation_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    rule_type       TEXT NOT NULL CHECK (rule_type IN ('user_defined', 'ai_learned')),
    match_field     TEXT NOT NULL,
    match_pattern   TEXT NOT NULL,
    target_account_id UUID NOT NULL REFERENCES account(id),
    priority        SMALLINT NOT NULL DEFAULT 100,
    hit_count       INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Testing:**
- `test_accept_feedback_recorded` — Accepting a suggestion creates ai_feedback record.
- `test_correction_updates_rule` — Correcting the same merchant 3 times creates an AI-learned rule.
- `test_accuracy_metrics` — 80/100 accepted, 15 corrected, 5 rejected = 80% accuracy.
- `test_learned_rule_used_in_next_categorisation` — After learning a rule, next similar transaction uses it.
- `test_feedback_is_immutable` — ai_feedback records cannot be updated or deleted.

### Task 5.4: AI Journal Entry Drafting

**What:** AI generates draft journal entries from structured inputs (invoices, payroll exports, contracts) with explanations.

**Design:**

```python
# backend/app/ai/journal_drafter.py
class JournalDrafter:
    async def draft_from_document(
        self,
        org_id: UUID,
        document_text: str,
        document_type: str,  # "invoice", "payroll", "contract"
    ) -> JournalEntryDraft:
        """Generate a draft journal entry from a document using LLM.

        The LLM reads the document, identifies the transaction type,
        selects appropriate accounts, and produces balanced debit/credit lines.
        """
        ...

    async def draft_accrual(
        self,
        org_id: UUID,
        expense_description: str,
        amount: Decimal,
        period: FiscalPeriod,
    ) -> JournalEntryDraft:
        """Generate a standard accrual entry with reversal."""
        ...
```

**Testing:**
- `test_draft_from_invoice` — Invoice document produces DR AR / CR Revenue entry.
- `test_draft_from_payroll` — Payroll summary produces multi-line entry (salary, taxes, benefits).
- `test_draft_accrual_with_reversal` — Accrual generates both the accrual and the reversing entry.
- `test_draft_includes_explanation` — Every AI-drafted entry includes ai_explanation text.
- `test_high_confidence_auto_post` — Entries with confidence >= 0.95 auto-post per org settings.
- `test_low_confidence_requires_review` — Entries with confidence < 0.80 go to pending_review.

---

## Phase 6: Revenue Recognition (ASC 606 / IFRS 15)

### Definition of Done
- Revenue contracts with performance obligations and standalone selling prices.
- Transaction price allocation (relative standalone selling price method).
- Revenue recognition schedules: point-in-time and over-time (straight-line, input, output methods).
- Automated GL postings: DR Deferred Revenue / CR Revenue for each recognition event.
- Contract modification handling (prospective and cumulative catch-up).
- Revenue waterfall report.

### Task 6.1: Revenue Contract & Obligation Model

**What:** Create revenue_contract and performance_obligation tables with the ASC 606 five-step framework.

**Design:**

```python
# backend/app/services/revenue_recognition.py
class RevenueRecognitionService:
    async def create_contract(
        self,
        org_id: UUID,
        data: RevenueContractCreate,
        user_id: UUID,
    ) -> RevenueContract:
        """Create a revenue contract with performance obligations.

        Step 1 (Identify contract): Validate contract criteria.
        Step 2 (Identify obligations): Create performance_obligation records.
        Step 3 (Determine transaction price): Set total_value.
        Step 4 (Allocate price): Allocate to obligations using relative SSP method.
        Step 5 (Recognise revenue): Generate recognition schedule.
        """
        ...

    def allocate_transaction_price(
        self,
        total_value: Decimal,
        obligations: list[PerformanceObligation],
    ) -> dict[UUID, Decimal]:
        """Allocate transaction price to obligations using relative SSP.

        allocated_price_i = total_value * (ssp_i / sum(all_ssp))
        """
        total_ssp = sum(o.standalone_selling_price for o in obligations)
        return {
            o.id: (total_value * o.standalone_selling_price / total_ssp).quantize(Decimal("0.0001"))
            for o in obligations
        }

    async def generate_schedule(
        self,
        obligation: PerformanceObligation,
        fiscal_periods: list[FiscalPeriod],
    ) -> list[RevenueScheduleEntry]:
        """Generate recognition schedule entries for an obligation.

        For over_time/straight_line:
            Spread allocated_price evenly across periods from start to end.
        For point_in_time:
            Recognise full amount on satisfaction date.
        """
        ...

    async def recognise_revenue(
        self,
        org_id: UUID,
        schedule_entry_id: UUID,
        user_id: UUID,
    ) -> PostingGroup:
        """Create the GL posting for a revenue recognition event.

        DR Deferred Revenue (allocated from contract)
        CR Revenue (recognised amount)
        """
        ...
```

**Testing:**
- `test_price_allocation_relative_ssp` — $120K contract with $96K and $24K SSP allocates 80/20.
- `test_straight_line_schedule` — $96K over 12 months = $8K per month.
- `test_point_in_time_recognition` — Full amount recognised on single satisfaction date.
- `test_recognition_creates_posting` — Recognition event creates DR Deferred Revenue / CR Revenue.
- `test_contract_modification_prospective` — Scope change creates new version with adjusted schedule.
- `test_contract_modification_cumulative_catchup` — Price change triggers cumulative catch-up adjustment.
- `test_revenue_waterfall_report` — Report shows recognised vs deferred per period.

---

## Phase 7: Financial Statements & Reporting

### Definition of Done
- Materialised reporting layer: mat_account_balance, mat_financial_statement.
- Trial balance, P&L, balance sheet, cash flow statement generation.
- Period-over-period comparison with variance calculation.
- XBRL-tagged output for SEC filing preparation.
- Beancount-format export for plain-text workflows.
- Report refresh orchestration (scheduled and on-demand).

### Task 7.1: Materialised Account Balances

**What:** Create and maintain mat_account_balance table with opening/closing balances per account per fiscal period. Refresh via trigger or scheduled job.

**Design:**

```sql
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
```

```python
# backend/app/services/report_generator.py
class ReportGenerator:
    async def refresh_account_balances(
        self, org_id: UUID, fiscal_period_id: UUID
    ) -> int:
        """Recompute mat_account_balance from immutable postings.

        For each account:
            opening_balance = previous period closing_balance (or 0 for first period)
            period_debits = SUM(posting.debit_amount) WHERE period matches
            period_credits = SUM(posting.credit_amount) WHERE period matches
            closing_balance = opening_balance + period_debits - period_credits
                (adjusted for normal_balance direction)
        """
        ...

    async def generate_trial_balance(
        self, org_id: UUID, fiscal_period_id: UUID
    ) -> list[TrialBalanceLine]:
        """Generate trial balance from mat_account_balance."""
        ...

    async def generate_income_statement(
        self, org_id: UUID, fiscal_period_id: UUID, comparative_period_id: UUID | None = None
    ) -> FinancialStatement:
        """Generate P&L statement with optional period comparison."""
        ...

    async def generate_balance_sheet(
        self, org_id: UUID, as_of_period_id: UUID
    ) -> FinancialStatement:
        """Generate balance sheet as of period end."""
        ...

    async def generate_cash_flow(
        self, org_id: UUID, fiscal_period_id: UUID
    ) -> FinancialStatement:
        """Generate cash flow statement (indirect method)."""
        ...
```

**Testing:**
- `test_balance_refresh_correct` — After posting 3 entries, balances match expected values.
- `test_opening_balance_carries_forward` — Period 2 opening = Period 1 closing.
- `test_trial_balance_debits_equal_credits` — Total debits == total credits.
- `test_income_statement_revenue_minus_expenses` — Net income = revenue - expenses.
- `test_balance_sheet_assets_equal_liabilities_plus_equity` — A = L + E.
- `test_comparative_period_variance` — P&L shows prior period and variance %.

### Task 7.2: XBRL Export

**What:** Generate XBRL-tagged output from financial statements using account.xbrl_mapping for element tagging.

**Design:**

```python
# backend/app/integrations/xbrl.py
class XBRLExporter:
    def export_instance(
        self,
        statement: FinancialStatement,
        taxonomy: str = "us-gaap-2026",
    ) -> str:
        """Generate XBRL instance document from a financial statement.

        Maps each statement line to its XBRL element via account.xbrl_mapping.
        Returns XML string conforming to the specified taxonomy.
        """
        ...
```

**Testing:**
- `test_xbrl_export_valid_xml` — Output is well-formed XML.
- `test_xbrl_elements_mapped` — Accounts with xbrl_mapping produce correct element tags.
- `test_xbrl_period_context` — XBRL contexts include correct period dates.

### Task 7.3: Beancount Export

**What:** Export the general ledger in Beancount plain-text format for Git-versionable workflows.

**Design:**

```python
# backend/app/integrations/beancount_export.py
class BeancountExporter:
    async def export_ledger(
        self,
        org_id: UUID,
        start_date: date,
        end_date: date,
    ) -> str:
        """Export postings as Beancount-formatted text.

        Format:
            2026-05-10 * "Monthly office rent"
              Expenses:Rent    5000.00 USD
              Liabilities:RentPayable    -5000.00 USD
        """
        ...

    def _format_account_path(self, account: Account) -> str:
        """Convert account hierarchy to Beancount account path.
        e.g., Assets:Current:CashAndEquivalents
        """
        ...
```

**Testing:**
- `test_beancount_export_parses` — Exported text can be parsed by Beancount's parser.
- `test_beancount_accounts_mapped` — Account hierarchy maps to Beancount account paths.
- `test_beancount_balances_match` — Beancount `bean-check` validates the exported file.

---

## Phase 8: Startup Metrics & Intelligence

### Definition of Done
- Automated computation of ARR, MRR, burn rate, runway, headcount cost, gross margin, NRR, customer count, ARPU.
- Metrics derived from GL data (not manual input).
- Daily/weekly metric snapshot storage.
- Metrics dashboard API endpoints.
- AI-generated flux commentary for period-over-period variance.
- Anomaly detection on transaction patterns.

### Task 8.1: Metric Calculator

**What:** Compute startup financial metrics from GL data. Each metric has a defined computation method traceable to specific accounts and posting types.

**Design:**

```python
# backend/app/services/metric_calculator.py
from dataclasses import dataclass
from decimal import Decimal
from datetime import date

@dataclass
class MetricResult:
    metric_type: str
    value: Decimal
    currency_code: str
    computation_details: dict  # how it was derived, for auditability

class MetricCalculator:
    async def compute_burn_rate(
        self, org_id: UUID, as_of: date, lookback_months: int = 3
    ) -> MetricResult:
        """Average monthly operating expenses over lookback period.

        burn_rate = AVG(monthly operating expenses for last N months)
        Operating expenses = SUM(postings to expense accounts) per month.
        Excludes COGS, depreciation, and one-time items.
        """
        ...

    async def compute_runway(
        self, org_id: UUID, as_of: date
    ) -> MetricResult:
        """Months of cash remaining at current burn rate.

        runway = cash_balance / burn_rate
        cash_balance = current balance of cash & cash equivalent accounts
        """
        ...

    async def compute_arr(self, org_id: UUID, as_of: date) -> MetricResult:
        """Annual Recurring Revenue from revenue recognition data.

        ARR = SUM(active contract annual values) for recurring obligations.
        """
        ...

    async def compute_mrr(self, org_id: UUID, as_of: date) -> MetricResult:
        """Monthly Recurring Revenue = ARR / 12."""
        ...

    async def compute_all(self, org_id: UUID, as_of: date) -> list[MetricResult]:
        """Compute all metrics and store as a snapshot."""
        ...
```

**Testing:**
- `test_burn_rate_3_month_average` — $90K total expenses over 3 months = $30K/month burn.
- `test_runway_calculation` — $300K cash / $30K burn = 10 months runway.
- `test_arr_from_contracts` — Two $60K annual contracts = $120K ARR.
- `test_mrr_equals_arr_divided_by_12` — $120K ARR = $10K MRR.
- `test_metrics_stored_as_snapshot` — Computed metrics persisted to mat_startup_metric.
- `test_metrics_exclude_one_time_items` — One-time legal fee excluded from burn rate.

### Task 8.2: AI Flux Commentary & Anomaly Detection

**What:** AI-generated variance analysis commentary for period-over-period changes, and anomaly detection on transaction patterns.

**Design:**

```python
# backend/app/ai/flux_analyst.py
class FluxAnalyst:
    async def generate_commentary(
        self,
        org_id: UUID,
        current_period_id: UUID,
        prior_period_id: UUID,
    ) -> list[FluxComment]:
        """Generate variance analysis commentary for significant account changes.

        For each account with >10% or >$1K variance:
            1. Identify the variance amount and percentage.
            2. Query the underlying transactions causing the change.
            3. Use LLM to generate a natural-language explanation.
        """
        ...

# backend/app/ai/anomaly.py
class AnomalyDetector:
    async def detect_anomalies(
        self, org_id: UUID, period_id: UUID
    ) -> list[Anomaly]:
        """Detect unusual transaction patterns.

        Checks:
            - Transactions outside 2 standard deviations of historical amounts per account.
            - Duplicate payments (same vendor, same amount, same period).
            - Weekend/holiday postings (unusual for the org).
            - Round-number bias (suspiciously round amounts).
        """
        ...
```

**Testing:**
- `test_flux_commentary_generated` — 50% increase in marketing spend produces commentary.
- `test_anomaly_duplicate_payment` — Two identical vendor payments flagged.
- `test_anomaly_outlier_amount` — $50K expense on an account that averages $5K flagged.
- `test_anomaly_round_number` — Multiple $10,000.00 expenses to same vendor flagged.

---

## Phase 9: Frontend Application

### Definition of Done
- Next.js application with authentication (OIDC flow).
- Dashboard with key financial metrics (cash balance, AR/AP, burn rate, runway).
- Chart of accounts management UI.
- Journal entry creation and approval workflow.
- Bank feed connection (Plaid Link) and reconciliation interface.
- Invoice creation and management.
- Financial statement views (P&L, balance sheet).
- Responsive design for desktop and tablet.

### Task 9.1: Authentication & Layout Shell

**What:** Next.js app with OIDC login, authenticated layout, organisation switcher, and navigation sidebar.

**Design:**

```typescript
// frontend/src/types/auth.ts
export interface AuthenticatedUser {
  userId: string;
  email: string;
  organisationId: string;
  role: 'owner' | 'admin' | 'accountant' | 'bookkeeper' | 'viewer';
  displayName: string;
}

// frontend/src/app/layout.tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <AuthProvider>
          <OrganisationProvider>
            {children}
          </OrganisationProvider>
        </AuthProvider>
      </body>
    </html>
  );
}

// frontend/src/components/sidebar.tsx
const navigation = [
  { name: 'Dashboard', href: '/', icon: HomeIcon },
  { name: 'Accounts', href: '/accounts', icon: BookOpenIcon },
  { name: 'Journal Entries', href: '/journals', icon: DocumentTextIcon },
  { name: 'Invoices', href: '/invoices', icon: DocumentIcon },
  { name: 'Bank Feeds', href: '/bank-feeds', icon: BanknotesIcon },
  { name: 'Reconciliation', href: '/reconciliation', icon: ArrowsRightLeftIcon },
  { name: 'Reports', href: '/reports', icon: ChartBarIcon },
  { name: 'Contacts', href: '/contacts', icon: UsersIcon },
  { name: 'Settings', href: '/settings', icon: CogIcon },
];
```

**Testing:**
- `test_unauthenticated_redirects_to_login` — Accessing /dashboard without auth redirects.
- `test_sidebar_renders_all_navigation` — All navigation items render correctly.
- `test_organisation_switcher` — Switching orgs reloads data for new context.
- `test_role_based_menu_visibility` — Viewer role does not see Settings or Journal Entry creation.

### Task 9.2: Dashboard & Metrics Display

**What:** Financial dashboard with cash balance, AR/AP summaries, burn rate, runway, and recent transactions.

**Testing:**
- `test_dashboard_loads_metrics` — Dashboard displays burn rate, runway, cash balance.
- `test_dashboard_chart_renders` — Revenue trend chart renders with correct data points.
- `test_dashboard_recent_transactions` — Last 10 bank transactions displayed.

### Task 9.3: Chart of Accounts, Journals, Invoices, Reconciliation UI

**What:** CRUD interfaces for chart of accounts (tree view), journal entry workflow (create, review, approve, post), invoice management, and bank reconciliation with AI suggestions.

**Testing:**
- `test_account_tree_view` — Chart of accounts renders as an expandable tree.
- `test_journal_entry_form_validates_balance` — Form prevents submission when debits != credits.
- `test_invoice_line_item_calculation` — Adding a line recalculates subtotal, tax, total in real time.
- `test_reconciliation_ai_suggestion_display` — AI suggestions shown with confidence score and accept/correct/reject buttons.
- `test_reconciliation_drag_and_drop` — Bank transactions can be dragged to match with GL entries.

---

## Phase 10: MCP Server & AI Query Interface

### Definition of Done
- MCP server exposes GL, accounts, bank transactions, reports, and metrics as tools.
- Natural language query interface: ask questions about financial data in plain English.
- MCP server tested with Claude Desktop and other MCP clients.
- API endpoints for NL query via REST (for the frontend chat interface).

### Task 10.1: MCP Server Implementation

**What:** Implement an MCP server following the 2025-11-25 specification, exposing accounting data and operations as tools for LLM clients.

**Design:**

```python
# backend/app/mcp/server.py
from mcp.server import Server
from mcp.types import Tool, TextContent

server = Server("ai-native-accounting")

@server.tool("get_trial_balance")
async def get_trial_balance(period: str, org_id: str) -> str:
    """Get the trial balance for a fiscal period."""
    ...

@server.tool("get_account_balance")
async def get_account_balance(account_code: str, as_of_date: str) -> str:
    """Get the current balance of a specific account."""
    ...

@server.tool("search_transactions")
async def search_transactions(query: str, start_date: str, end_date: str) -> str:
    """Search bank transactions and journal entries by description or amount."""
    ...

@server.tool("create_journal_entry")
async def create_journal_entry(
    date: str, description: str, lines: list[dict]
) -> str:
    """Create a draft journal entry for review."""
    ...

@server.tool("get_burn_rate")
async def get_burn_rate() -> str:
    """Get the current monthly burn rate and runway."""
    ...

@server.tool("get_financial_statement")
async def get_financial_statement(
    statement_type: str, period: str
) -> str:
    """Get a financial statement (income_statement, balance_sheet, cash_flow)."""
    ...
```

**Testing:**
- `test_mcp_server_lists_tools` — Server returns complete tool catalogue.
- `test_mcp_trial_balance_tool` — `get_trial_balance` returns correct data.
- `test_mcp_create_journal_entry` — Tool creates a draft entry and returns confirmation.
- `test_mcp_authentication` — Unauthenticated MCP requests are rejected.

### Task 10.2: Natural Language Query Interface

**What:** Allow users to ask questions about their financial data in plain English, powered by LLM with tool use over the GL.

**Design:**

```python
# backend/app/ai/nl_query.py
class NaturalLanguageQuery:
    async def query(self, org_id: UUID, question: str, user_id: UUID) -> NLQueryResponse:
        """Answer a natural-language financial question.

        Examples:
            "What was our revenue last month?"
            "Which vendor did we spend the most with in Q1?"
            "How much do we owe in accounts payable?"
            "What's our current burn rate?"

        Strategy:
            1. Parse the question to identify intent and parameters.
            2. Use tool-use (MCP tools) to fetch the relevant data.
            3. Generate a natural-language answer with data citations.
        """
        ...
```

**Testing:**
- `test_nl_query_revenue` — "What was our revenue last month?" returns correct revenue figure.
- `test_nl_query_top_vendor` — "Which vendor did we pay the most?" returns correct vendor name and amount.
- `test_nl_query_burn_rate` — "What's our burn rate?" returns the computed metric.
- `test_nl_query_ambiguous` — Ambiguous question gets a clarification request, not a wrong answer.

---

## Phase 11: E-Invoicing & International Compliance

### Definition of Done
- PEPPOL BIS Billing 3.0 invoice generation (UBL 2.1 XML).
- EN 16931 validation of generated e-invoices.
- ZUGFeRD / Factur-X hybrid PDF generation.
- Multi-currency improvements: automatic ECB rate fetching, unrealized gain/loss.
- VAT return calculation for EU jurisdictions.
- Exchange rate source configuration (ECB, Open Exchange Rates).

### Task 11.1: PEPPOL Invoice Generation

**What:** Generate PEPPOL BIS Billing 3.0 compliant UBL 2.1 XML from invoice data.

**Design:**

```python
# backend/app/integrations/peppol.py
class PEPPOLGenerator:
    def generate_ubl_invoice(self, invoice: Invoice, seller: Organisation, buyer: Contact) -> str:
        """Generate UBL 2.1 XML conforming to PEPPOL BIS Billing 3.0.

        Validates:
            - Seller and buyer PEPPOL endpoint IDs present.
            - Tax category codes conform to EN 16931.
            - All mandatory BIS Billing fields populated.

        Returns:
            UBL 2.1 XML string.
        """
        ...

    def validate_en16931(self, ubl_xml: str) -> list[ValidationError]:
        """Validate UBL XML against EN 16931 Schematron rules."""
        ...

    def generate_facturx_pdf(self, invoice: Invoice, ubl_xml: str, pdf_template: bytes) -> bytes:
        """Embed CII XML into a PDF to create a Factur-X/ZUGFeRD hybrid document."""
        ...
```

**Testing:**
- `test_ubl_xml_valid` — Generated XML validates against UBL 2.1 schema.
- `test_en16931_validation_passes` — Invoice with correct fields passes EN 16931 Schematron.
- `test_en16931_validation_catches_missing_field` — Missing buyer reference fails validation.
- `test_facturx_pdf_embeds_xml` — Generated PDF contains extractable CII XML.

### Task 11.2: Multi-Currency & Exchange Rates

**What:** Automatic exchange rate fetching, unrealized foreign exchange gain/loss calculation, and multi-currency reporting.

**Design:**

```python
# backend/app/integrations/exchange_rates.py
class ExchangeRateService:
    async def fetch_rates(self, base_currency: str, date: date, source: str = "ecb") -> dict[str, Decimal]:
        """Fetch exchange rates from ECB or Open Exchange Rates."""
        ...

    async def compute_unrealized_fx_gain_loss(
        self, org_id: UUID, as_of_date: date
    ) -> list[FXGainLoss]:
        """Calculate unrealized FX gain/loss on open foreign-currency positions.

        For each foreign-currency AR/AP balance:
            gain_loss = balance * (current_rate - original_rate)
        """
        ...
```

**Testing:**
- `test_ecb_rate_fetch` — Fetches EUR/USD rate from ECB API.
- `test_fx_gain_loss_calculation` — EUR invoice at 1.10, current rate 1.15 = gain.
- `test_multi_currency_balance_sheet` — Foreign balances converted at period-end rate.

---

## Phase 12: Production Hardening & Deployment

### Definition of Done
- Kubernetes deployment manifests (Helm chart or Kustomize).
- Database migration strategy for zero-downtime deployments.
- Rate limiting, request validation, and security headers.
- Structured logging with correlation IDs.
- Health check, readiness probe, and liveness probe.
- Backup and restore procedures for PostgreSQL.
- Performance testing: 1000 concurrent users, 100K transactions/org.
- Security audit checklist (OWASP ASVS Level 2).
- Documentation: API reference (auto-generated from OpenAPI), deployment guide, admin guide.

### Task 12.1: Kubernetes Deployment

**What:** Create Kubernetes manifests for production deployment: backend, frontend, PostgreSQL (with operator), Redis, and ingress.

**Design:**

```yaml
# k8s/backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: accounting-backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: accounting-backend
  template:
    spec:
      containers:
        - name: backend
          image: accounting-backend:latest
          ports:
            - containerPort: 8000
          env:
            - name: ACCOUNTING_DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: accounting-secrets
                  key: database-url
          livenessProbe:
            httpGet:
              path: /api/health
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /api/health
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 2Gi
```

**Testing:**
- `test_health_endpoint_under_load` — /api/health responds <100ms under 1000 concurrent requests.
- `test_zero_downtime_deployment` — Rolling update completes with no 5xx errors during transition.
- `test_database_migration_backward_compatible` — Migration can run while old version is still serving.
- `test_rate_limiting` — More than 100 requests/sec from a single IP returns 429.

### Task 12.2: Security Hardening

**What:** Implement OWASP ASVS Level 2 controls: input validation, output encoding, CSRF protection, security headers, secrets management.

**Testing:**
- `test_sql_injection_prevented` — Parameterised queries prevent SQL injection on all endpoints.
- `test_xss_prevention` — HTML entities are escaped in all API responses.
- `test_security_headers` — Response includes X-Content-Type-Options, X-Frame-Options, CSP.
- `test_secrets_not_in_logs` — API keys and tokens are redacted from structured logs.
- `test_audit_log_integrity` — Audit log entries cannot be deleted via API.

### Task 12.3: Performance & Load Testing

**What:** Verify system performance at target scale: 1000 concurrent users, 100K transactions per organisation, sub-second report generation.

**Testing:**
- `test_trial_balance_100k_transactions` — Trial balance for org with 100K postings returns in <2s.
- `test_bank_sync_1000_transactions` — Importing 1000 bank transactions completes in <30s.
- `test_concurrent_posting` — 100 concurrent posting requests complete without deadlocks.
- `test_materialised_refresh_performance` — Full balance refresh for 100K postings completes in <60s.

---

## Summary

| Phase | Name | Tasks | Key Deliverable |
|-------|------|-------|-----------------|
| 1 | Foundation | 4 | FastAPI app, PostgreSQL with RLS, OAuth, Docker |
| 2 | Chart of Accounts & GL | 5 | GAAP chart of accounts, immutable posting engine, journal workflow |
| 3 | Invoicing, AP/AR & Payments | 4 | Unified invoice model, payment allocation, tax rates |
| 4 | Bank Feeds & Reconciliation | 3 | Plaid integration, transaction import, reconciliation workflow |
| 5 | AI Categorisation & Journal Drafting | 4 | LLM abstraction, categoriser, feedback loop, journal drafter |
| 6 | Revenue Recognition | 1 | ASC 606 five-step framework, recognition schedules |
| 7 | Financial Statements & Reporting | 3 | Materialised balances, P&L/BS/CF, XBRL & Beancount export |
| 8 | Startup Metrics & Intelligence | 2 | Burn rate, runway, ARR, flux commentary, anomaly detection |
| 9 | Frontend Application | 3 | Next.js dashboard, account/journal/invoice/reconciliation UI |
| 10 | MCP Server & AI Query | 2 | MCP tools, natural language financial queries |
| 11 | E-Invoicing & International | 2 | PEPPOL/UBL, Factur-X, multi-currency, VAT |
| 12 | Production Hardening | 3 | Kubernetes, security, performance testing |
| **Total** | | **36 tasks** | |
