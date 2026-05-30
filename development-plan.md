# Book Publishing Platform — Phased Development Plan

> Project: 442-book-publishing-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files. The database design adopts **Suggestion 1 (Normalized PostgreSQL)** as the system of record, augmented with the **append-only financial-record discipline** and selective **JSONB** use from Suggestion 3 for ONIX/metadata flexibility. Event-sourcing (Suggestion 2) and graph storage (Suggestion 4) are explicitly rejected for the MVP: the immutable `audit_log` plus append-only financial tables deliver the required auditability without the operational complexity of a full ES/CQRS or polyglot-persistence stack.

The core value proposition — a unified platform covering title/ISBN management, editorial workflow, contracts/rights, and a correct royalty engine for independent and mid-size publishers — ships across Phases 1–6. AI-native features and advanced integrations follow in Phases 9–11.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | Python 3.12 | Royalty correctness is the central risk; Python's `decimal` module gives exact fixed-point arithmetic, and the domain leans on data parsing (distributor CSVs), XML generation (ONIX), and LLM calls (AI triage) where Python's library ecosystem is strongest. |
| API framework | FastAPI | Native Pydantic v2 validation for financial inputs, automatic OpenAPI 3.1 generation (a `standards.md` requirement), async support for webhook/LLM I/O, and first-class dependency injection for auth and tenancy. |
| ORM / DB toolkit | SQLAlchemy 2.0 (async) + Alembic | Explicit mapping suits the normalized 8-context schema; Alembic gives version-controlled migrations for a decade-lived data model. |
| Database | PostgreSQL 16 | Chosen per Suggestion 1: `NUMERIC` for exact money, `JSONB` for audit log and ONIX overrides, GIN full-text search for catalogue, range partitioning for `sales_transactions`, ACID for financial transactions. |
| Money type | `decimal.Decimal` (Python) ↔ `NUMERIC(14,2)` (PG) | Floating point is forbidden in royalty math; all currency arithmetic uses `Decimal` with `ROUND_HALF_UP` quantisation to the statement currency's minor unit. |
| Task queue | Celery + Redis | Royalty calculation runs, ONIX export generation, CSV imports, statement PDF rendering, and LLM scoring are long-running and must not block API requests. |
| Cache / broker | Redis 7 | Doubles as Celery broker/result backend and a short-TTL cache for exchange rates and rights-availability lookups. |
| Object storage | S3-compatible (MinIO self-host / AWS S3 SaaS) | Manuscript files, signed contract PDFs, generated statements, and ONIX exports are large binaries kept out of the database. |
| Auth | OAuth 2.0 / OIDC via Authlib, JWT (RFC 7519) sessions | `standards.md` mandates OAuth 2.0/OIDC; staff use SSO, authors use the portal. JWT bearer tokens for the REST API. |
| Frontend | React 18 + TypeScript + Vite, TanStack Query, shadcn/ui | Two SPAs (staff console + author portal) consuming the REST API. shadcn/ui + Radix gives WCAG 2.2 AA / WAI-ARIA 1.2 accessible primitives required by `standards.md`. |
| PDF generation | WeasyPrint | Renders royalty statements from HTML/CSS templates; supports PDF/UA tagging for accessible statements. |
| XML / ONIX | lxml + custom builder validated against EDItEUR ONIX 3.1 XSD | ONIX 3.1 is mandatory (Amazon's March 2026 deadline); validate generated XML against the official schema. |
| LLM provider | Anthropic Claude via the official SDK, abstracted behind a provider interface | Manuscript scoring, sales forecasting, anomaly detection, and contract-clause analysis. Provider interface allows swap/self-host. |
| Payments | Stripe Connect | `standards.md` and `features.md` cite Stripe for advance disbursement and royalty payouts; tokenised flow keeps PCI DSS scope minimal. |
| Containerisation | Docker + docker-compose | Both deployment modes (SaaS, self-hosted) ship as containers; compose orchestrates api, worker, db, redis, minio. |
| Testing | pytest + pytest-asyncio + Schemathesis + Playwright | Unit/integration in pytest; Schemathesis fuzzes the OpenAPI spec; Playwright drives the SPAs for E2E and accessibility audits. |
| Code quality | Ruff (lint+format), mypy (strict), pre-commit | Single fast linter/formatter; strict typing catches money/currency mistakes at compile time. |
| Package manager | uv | Fast, reproducible locking for the Python backend; pnpm for the frontend workspaces. |
| Multi-tenancy | PostgreSQL Row-Level Security keyed on `publisher_id` | Per-publisher isolation in the SaaS deployment without schema-per-tenant overhead at MVP scale. |

### Project Structure

```
book-publishing-platform/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── .pre-commit-config.yaml
├── README.md
├── migrations/                      # Alembic revisions
│   └── versions/
├── src/
│   └── bpp/
│       ├── __init__.py
│       ├── main.py                  # FastAPI app factory
│       ├── config.py                # Pydantic Settings (env-driven)
│       ├── db.py                    # Async engine, session, RLS context
│       ├── deps.py                  # Auth, tenancy, pagination dependencies
│       ├── money.py                 # Decimal/Money + Currency helpers
│       ├── audit.py                 # Audit-log writing (SQLAlchemy events)
│       ├── models/                  # SQLAlchemy ORM models per context
│       │   ├── org.py               # organizations, persons, person_roles
│       │   ├── catalog.py           # works, series, products, isbn_prefixes, subjects
│       │   ├── editorial.py         # pipelines, stages, submissions, tasks
│       │   ├── contracts.py         # contracts, advances, rate_schedules, rights
│       │   ├── royalty.py           # periods, sales, calculations, statements
│       │   └── audit.py             # audit_log, portal_users
│       ├── schemas/                 # Pydantic request/response models
│       ├── routers/                 # FastAPI routers per context
│       ├── services/                # Business logic
│       │   ├── isbn.py              # ISBN-13 validation/allocation
│       │   ├── royalty_engine.py    # The calculation engine
│       │   ├── imports/             # Distributor CSV parsers
│       │   │   ├── base.py
│       │   │   ├── ingram.py
│       │   │   ├── amazon_kdp.py
│       │   │   └── baker_taylor.py
│       │   ├── onix.py              # ONIX 3.1 export builder
│       │   ├── statements.py        # PDF statement rendering
│       │   ├── fx.py                # Exchange-rate resolution
│       │   └── ai/                  # LLM-backed features
│       │       ├── provider.py      # LLM provider interface
│       │       ├── triage.py
│       │       ├── forecast.py
│       │       ├── anomaly.py
│       │       └── clauses.py
│       ├── tasks/                   # Celery tasks
│       └── templates/               # Jinja2/HTML for statement PDFs
├── frontend/
│   ├── pnpm-workspace.yaml
│   ├── packages/
│   │   ├── api-client/              # Generated from OpenAPI spec
│   │   └── ui/                      # Shared shadcn/ui components
│   └── apps/
│       ├── console/                 # Staff SPA
│       └── portal/                  # Author SPA
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── e2e/
│   └── fixtures/                    # Sample distributor CSVs, ONIX golden files
└── schemas/
    └── onix/                        # EDItEUR ONIX 3.1 XSD/RNG (vendored)
```

---

## Phase 1: Foundation & Platform Skeleton

### Purpose
Establish the runnable application skeleton: configuration, database connectivity, migrations, auth scaffolding, multi-tenancy, audit logging, and the money primitives that every later phase depends on. After this phase the app boots, serves a health check and an OpenAPI doc, runs migrations, and records audit entries — but exposes no domain features yet.

### Tasks

#### 1.1 — Project bootstrap, config, and containerisation

**What**: Scaffold the repo, dependency management, Docker stack, and environment-driven settings.

**Design**:
- `pyproject.toml` declares dependencies and tool config (Ruff, mypy strict, pytest).
- `docker-compose.yml` services: `api`, `worker`, `db` (postgres:16), `redis`, `minio`.
- `config.py` using `pydantic-settings`:
```python
class Settings(BaseSettings):
    database_url: PostgresDsn
    redis_url: RedisDsn
    s3_endpoint: str; s3_bucket: str; s3_key: str; s3_secret: str
    jwt_secret: SecretStr
    oidc_issuer: str | None = None
    default_currency: str = "USD"
    statement_minimum_payment: Decimal = Decimal("25.00")
    environment: Literal["dev", "test", "staging", "prod"] = "dev"
    model_config = SettingsConfigDict(env_prefix="BPP_", env_file=".env")
```
- `main.py` exposes `create_app() -> FastAPI` with `/health` and `/openapi.json`.
- Error-handling strategy: a global exception handler maps `DomainError` subclasses to RFC 9457 problem+json responses with stable `type` URIs.

**Testing**:
- `Unit: Settings loads from env vars with BPP_ prefix → correct typed values`
- `Unit: missing required DATABASE_URL → ValidationError naming the field`
- `Integration: GET /health → 200 {"status":"ok"}`
- `Integration: GET /openapi.json → valid OpenAPI 3.1 document (schema-validated)`
- `E2E: docker compose up → api container healthy within 30s`

#### 1.2 — Database layer, async sessions, and RLS tenancy

**What**: Async SQLAlchemy engine/session, per-request publisher tenancy via Postgres RLS.

**Design**:
- `db.py`: async engine, `async_sessionmaker`, `get_session()` dependency.
- Every tenant-scoped table carries `publisher_id UUID NOT NULL`. A session-level `SET app.current_publisher = :pid` drives RLS policies:
```sql
ALTER TABLE works ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON works
  USING (publisher_id = current_setting('app.current_publisher')::uuid);
```
- `deps.py` `tenant_context(session, publisher_id)` sets the GUC at the start of each request transaction.

**Testing**:
- `Integration (real PG): session for publisher A cannot SELECT rows owned by publisher B`
- `Integration (real PG): INSERT without a current_publisher set → policy error`
- `Unit: get_session yields and closes session, rolls back on exception`

#### 1.3 — Money, currency, and exchange-rate primitives

**What**: A `Money` value type and FX resolution used by every financial computation.

**Design**:
```python
@dataclass(frozen=True)
class Money:
    amount: Decimal          # exact; never float
    currency: str            # ISO 4217
    def quantize(self) -> "Money": ...   # ROUND_HALF_UP to minor units
    def __add__(self, other: "Money") -> "Money": ...  # raises on currency mismatch
```
- `exchange_rates` table per Suggestion 1 §6. `fx.convert(money, to_currency, on_date, rate_type)` resolves the applicable rate; raises `MissingExchangeRate` if none.
- Rounding policy documented and centralised: bankers' rounding rejected; `ROUND_HALF_UP` chosen to match standard royalty-accounting convention.

**Testing**:
- `Unit: Money(10.005, USD).quantize() → 10.01 (ROUND_HALF_UP)`
- `Unit: Money(USD) + Money(GBP) → CurrencyMismatchError`
- `Unit: convert(100 GBP → USD @1.27 period_end) → 127.00 USD`
- `Unit: convert with no rate on date → MissingExchangeRate`

#### 1.4 — Audit log & append-only enforcement

**What**: Automatic audit trail on every mutation, plus DB-level immutability for financial tables.

**Design**:
- `audit_log` table per Suggestion 1 §8 (`old_values`/`new_values` JSONB, `changed_by`, `changed_at`, `ip_address`).
- SQLAlchemy `after_insert/after_update/after_delete` events write audit rows within the same transaction.
- Append-only financial tables (`sales_transactions`, `royalty_calculations`, `royalty_statements`, `royalty_statement_lines`) get a Postgres trigger rejecting `UPDATE`/`DELETE`; corrections are new adjustment rows.

**Testing**:
- `Integration: UPDATE a work → audit_log row with correct old/new JSONB diff`
- `Integration: UPDATE a finalized royalty_statement → trigger raises, row unchanged`
- `Integration: changed_by populated from request auth context`

#### 1.5 — Auth scaffolding (OIDC staff, JWT API)

**What**: Authentication middleware, role model, and protected-route dependency.

**Design**:
- `User`/role model: roles `admin`, `editor`, `finance`, `rights_manager`, `author`.
- `require_role(*roles)` FastAPI dependency; OIDC code flow for staff, password+JWT for portal users (`portal_users` table).
- OWASP Top Ten alignment: bcrypt/argon2 password hashing, JWT expiry + refresh, broken-access-control tests on every protected route.

**Testing**:
- `Unit: require_role('finance') with editor token → 403`
- `Integration: valid OIDC callback → session JWT issued`
- `Integration: expired JWT → 401`
- `Integration: author token cannot reach /console/* endpoints → 403`

---

## Phase 2: Title & ISBN Management (Catalogue Core)

### Purpose
Build the work/product model that everything else references. Implements the Work-vs-Product separation (one intellectual work, many ISBN-bearing format products), ISBN allocation/validation, contributors, and subject classification (BISAC). After this phase a publisher can register titles, allocate ISBNs per format, and search the catalogue.

### Tasks

#### 2.1 — Works, series, and contributors

**What**: CRUD for works, series grouping, and contributor links.

**Design**: ORM models mirror Suggestion 1 §2 (`works`, `series`, `work_contributors`, `persons`, `person_roles`). `publication_status` state machine: `proposed → in_development → in_production → published → out_of_print` plus `abandoned`/`rights_reverted`. `contribution_pct` per contributor feeds royalty splits later.
- Endpoints: `POST /works`, `GET /works/{id}`, `PATCH /works/{id}`, `GET /works?q=&status=&series_id=` (GIN full-text on title).

**Testing**:
- `Unit: status transition published→proposed → InvalidTransition`
- `Unit: contributor pcts summing >100 → ValidationError`
- `Integration: POST /works then GET → contributors echoed in sequence order`
- `Integration: GET /works?q=midnight → full-text match returned`

#### 2.2 — Products & ISBN allocation

**What**: Format-level products each carrying a validated ISBN-13, drawn from a registered prefix.

**Design**: `products`, `isbn_prefixes` per Suggestion 1 §2. `isbn.py`:
```python
def validate_isbn13(isbn: str) -> bool: ...      # mod-10 check digit
def allocate_isbn(prefix: IsbnPrefix) -> str:    # next sequential + check digit; increments assigned_count
```
- `product_form` enum covers hardback, trade/mass-market paperback, ebook (epub/pdf/kindle), audiobook (download/cd), large_print, board_book. Every format requires a distinct ISBN (ISO 2108).
- Endpoint `POST /works/{id}/products` allocates from the publisher's active prefix unless an ISBN is supplied (then validated).

**Testing**:
- `Unit: validate_isbn13('9780306406157') → True; bad check digit → False`
- `Unit: allocate_isbn exhausted prefix → PrefixExhausted`
- `Integration: create epub + paperback → two distinct ISBNs, assigned_count incremented by 2`
- `Integration: duplicate ISBN insert → 409 (unique constraint)`

#### 2.3 — Subject classification (BISAC) & metadata

**What**: BISAC/THEMA subject codes and ONIX-relevant metadata fields per product.

**Design**: `product_subjects` (scheme, code, heading, is_primary) and `product_onix_metadata` per Suggestion 1 §2/§7. Validate BISAC codes against a vendored BISAC code list; exactly one `is_primary` per product per scheme.

**Testing**:
- `Unit: invalid BISAC code 'ZZZ999' → ValidationError`
- `Unit: two primary subjects same scheme → ValidationError`
- `Integration: assign BISAC FIC000000 → stored with heading text`

---

## Phase 3: Royalty Calculation Engine (Core Value)

### Purpose
The financial heart of the product and its hardest correctness problem. Implements escalating per-tier rates, reserve-against-returns, advance recoupment, agent commission, and multi-currency — all as a deterministic, fully audited, override-capable engine. This phase delivers the differentiator versus lightweight competitors (Familiar) while matching incumbent depth (MetaComet). The engine is pure and testable in isolation before any sales data or statements exist.

### Tasks

#### 3.1 — Contract, advance, and rate-schedule model

**What**: Persist contracts with advance tranches and escalating royalty rate schedules.

**Design**: ORM per Suggestion 1 §4 (`contracts`, `contract_advances`, `royalty_rate_schedules`, `royalty_rate_tiers`). Rate schedules are temporal (`valid_from`/`valid_until`) so historical periods always use the rate then in effect. Tiers are cumulative-unit thresholds:
```python
@dataclass
class RateTier:
    floor: int           # cumulative units (inclusive)
    ceiling: int | None  # None = unbounded
    rate_pct: Decimal
```
- `rate_basis` ∈ {list_price, net_receipts, net_revenue, retail_price}.

**Testing**:
- `Unit: tiers with overlapping floors → ValidationError`
- `Unit: tier ceiling < floor → ValidationError`
- `Integration: create contract with 3-tier hardback schedule → persisted with valid_from`

#### 3.2 — The calculation engine (pure functions)

**What**: A pure module computing per-contract, per-period royalties from sales facts.

**Design**: `royalty_engine.py` operates on plain inputs (no DB), enabling exhaustive unit testing:
```python
@dataclass
class SalesFact:
    product_id: UUID; product_form: str; channel: str | None
    territory: str | None; net_units: int; basis_amount: Money

@dataclass
class CalcContext:
    schedules: list[RateSchedule]      # resolved as-of period
    cumulative_units_before: dict[UUID, int]
    reserve_pct: Decimal
    opening_advance_balance: Money
    agent_commission_pct: Decimal

@dataclass
class CalcResult:
    per_product: list[ProductRoyalty]  # tier breakdown each
    gross_royalty: Money
    reserve_held: Money
    reserve_released: Money
    advance_recouped: Money
    closing_advance_balance: Money
    agent_commission: Money
    net_payable: Money

def calculate(facts: list[SalesFact], ctx: CalcContext) -> CalcResult: ...
```
- Algorithm, in order: (1) for each product, walk cumulative units through tiers, multiplying units-in-tier × basis × rate; (2) sum gross; (3) hold reserve = gross × reserve_pct; (4) release matured reserves from prior periods; (5) recoup advance up to remaining balance; (6) agent commission on net-of-recoupment; (7) net_payable. All steps emit line records.
- Returns negative `net_payable` as zero-carried (unrecouped balance carries forward; never claws back paid advances).

**Testing** (the highest-density test target in the project):
- `Unit: single tier, 100 units @ $20 list, 10% → $200.00`
- `Unit: escalator crossing 5000-unit boundary → split across two rates, exact Decimal`
- `Unit: returns exceed sales → negative net units handled, reserve logic correct`
- `Unit: reserve 20% held → gross minus reserve, ReserveHeld recorded`
- `Unit: prior reserve matures this period → released and added to payable`
- `Unit: advance balance $5000, earned $3000 → recoup $3000, closing $2000, payable $0`
- `Unit: advance earned out mid-period → partial recoup, remainder paid, earned_out flagged`
- `Unit: agent 15% commission applied after recoupment`
- `Unit: multi-currency facts → all converted to contract currency before summation`
- `Property test: sum(line_items) == net_payable for randomised inputs`

#### 3.3 — Periods, persistence, and override-with-justification

**What**: Wire the pure engine to periods, persist results immutably, and support documented overrides.

**Design**: `royalty_periods`, `royalty_calculations`, `reserves_against_returns`, `advance_balances` per Suggestion 1 §5. A Celery task `run_royalty_calculation(period_id)` loads facts, calls `calculate`, and persists results in one transaction. Override: `is_override=TRUE` + mandatory `override_reason`; original computed values retained in audit_log for legal defensibility.

**Testing**:
- `Integration: run period → royalty_calculations rows match engine output`
- `Integration: override without reason → 422`
- `Integration: re-running a finalized period → blocked (immutability trigger)`
- `Integration: override recorded in audit_log with prior value`

---

## Phase 4: Sales Data Ingestion (ETL)

### Purpose
Feed the royalty engine. Distributor reports arrive in inconsistent proprietary CSVs with no public APIs (a documented structural gap in `standards.md`). This phase builds a pluggable parser framework normalising Ingram, Amazon KDP, and Baker & Taylor formats into `sales_transactions`, with ISBN matching, validation, and full import provenance.

### Tasks

#### 4.1 — Parser framework & import batches

**What**: A base parser interface and batch-tracking lifecycle.

**Design**: `sales_import_batches` per Suggestion 1 §5; lifecycle `pending → validating → validated → imported` (or `failed`/`rejected`).
```python
class SalesParser(Protocol):
    format_id: str
    def parse(self, raw: IO[bytes]) -> Iterator[ParsedSalesRow]: ...

@dataclass
class ParsedSalesRow:
    isbn: str; channel: str; territory: str | None
    txn_type: Literal["sale","return","adjustment"]
    quantity: int; unit_price: Decimal; currency: str; sale_date: date
```
- Registry maps `file_format` → parser. Unknown ISBNs queued as `unmatched` for manual resolution rather than failing the batch.

**Testing**:
- `Fixture: ingram_sample.csv → N normalised rows, units/gross totals match header`
- `Unit: malformed row → row error logged, batch continues`
- `Integration: unmatched ISBN → row flagged, resolvable via PATCH /imports/{id}/mappings`

#### 4.2 — Per-distributor parsers (Ingram, Amazon KDP, Baker & Taylor)

**What**: Concrete parsers for the three priority formats.

**Design**: Each handles its column names, return sign conventions (KDP reports returns as negatives; Ingram as separate return rows), date formats, and currency columns. Committed golden fixtures per distributor in `tests/fixtures/`.

**Testing**:
- `Fixture: each parser vs a hand-verified golden output JSON`
- `Unit: KDP negative-quantity row → txn_type='return', positive quantity`
- `Unit: Ingram multi-currency report → currency per row preserved`

#### 4.3 — Validation, dedup, and commit to sales_transactions

**What**: Validate parsed rows, detect duplicate batches, and persist immutably.

**Design**: ISBN→product resolution, currency validation, duplicate-file detection (hash of contents + distributor + period). On commit, write `sales_transactions` with both sale-currency and reporting-currency amounts (FX at period rate). Range-partition `sales_transactions` by `sale_date` year (Suggestion 1 partitioning strategy).

**Testing**:
- `Integration: re-import identical file → rejected as duplicate`
- `Integration: committed rows carry reporting_amount + exchange_rate`
- `Integration: validated batch feeds run_royalty_calculation correctly`

---

## Phase 5: Royalty Statements & Author-Facing Output

### Purpose
Turn calculations into the contractual artefacts authors receive. Generates period statements with line-item breakdowns and an audit trail, rendered as accessible (PDF/UA) PDFs, with a minimum-payment threshold and payment-status tracking. This closes the editorial-to-royalty loop that no single affordable incumbent fully covers.

### Tasks

#### 5.1 — Statement assembly & line items

**What**: Build statement + line records from a finalized calculation.

**Design**: `royalty_statements`, `royalty_statement_lines` per Suggestion 1 §5. Line types: sales, returns, reserve_held, reserve_released, sub_license_income, advance_recoupment, agent_commission, adjustment. `is_below_threshold` set when `net_payable < statement_minimum_payment`; sub-threshold balances carry forward.

**Testing**:
- `Unit: lines reconcile to header totals (sum invariant)`
- `Unit: net_payable $12 with $25 threshold → below_threshold, carried forward`
- `Integration: generate statement from a calc → immutable, audit-logged`

#### 5.2 — PDF rendering (PDF/UA accessible)

**What**: Render statements to tagged, accessible PDFs.

**Design**: WeasyPrint over a Jinja2 HTML template; emit PDF/UA structure tags (ISO 14289) per `standards.md`. Store in S3; `document_url` on the statement.

**Testing**:
- `Integration: rendered PDF passes veraPDF PDF/UA structural check`
- `Integration: PDF line items match statement_lines`
- `Snapshot: HTML template output stable for a fixed fixture`

#### 5.3 — Statement lifecycle & GL export

**What**: Statement status flow and general-ledger export for accounting integration.

**Design**: status `draft → reviewed → approved → sent → paid`. CSV/JSON GL export posting royalty liability per period (feature parity with MetaComet's GL export). `payment_reference` recorded on payment.

**Testing**:
- `Unit: approve before review → InvalidTransition`
- `Integration: GL export totals equal sum of approved statements for period`

---

## Phase 6: REST API Surface & Staff Console

### Purpose
Make the platform usable by humans. Consolidates Phases 2–5 behind a coherent, OpenAPI-3.1-described REST API and a staff React console covering the title grid, contract editor, royalty run/preview, and statement review. After this phase a publisher's editorial and finance teams can operate the platform end-to-end.

### Tasks

#### 6.1 — API consolidation, pagination, OpenAPI 3.1

**What**: Uniform list/pagination/filtering conventions and a published spec.

**Design**: Cursor pagination, `?fields=` sparse responses, consistent problem+json errors. The generated `/openapi.json` is the source for the frontend `api-client` package (codegen). Schema.org `Book` JSON-LD available on `GET /works/{id}?format=jsonld`.

**Testing**:
- `Integration: Schemathesis fuzz over the full spec → no 500s, schema-conformant responses`
- `Integration: GET /works/{id}?format=jsonld → valid Schema.org Book`

#### 6.2 — Staff console SPA

**What**: React console for catalogue, contracts, royalty runs, and statements.

**Design**: Title grid with configurable columns (editorial stage, pub date, financial status — mirrors Consonance UX). Rate-table editor for escalating tiers. Royalty preview before finalisation (mirrors MetaComet). shadcn/ui components meet WCAG 2.2 AA.

**Testing**:
- `E2E (Playwright): create work → allocate ISBNs → create contract → run royalties → preview statement`
- `E2E: axe-core accessibility scan on each major view → no WCAG AA violations`

---

## Phase 7: Editorial Workflow & Submissions

### Purpose
Add the editorial project-management layer that, combined with royalty accounting, forms the unified platform. Configurable pipelines, task/deadline tracking, and a manuscript submission portal with slush-pile triage — the v1.1 scope from `features.md`.

### Tasks

#### 7.1 — Configurable pipeline engine

**What**: Template-driven editorial stage pipelines, configurable without code.

**Design**: `pipeline_templates`, `pipeline_stages`, `work_pipeline`, `work_stage_history`, `editorial_tasks` per Suggestion 1 §3. Stages flagged `is_milestone` can trigger contract advance tranches (links to `contract_advances.trigger_stage_id`).

**Testing**:
- `Integration: advancing to a milestone stage → linked advance tranche marked 'due'`
- `Unit: skip mandatory stage → blocked`

#### 7.2 — Submission portal & triage queue

**What**: Public intake form, automated acknowledgement, and editor triage queue.

**Design**: `submissions` per Suggestion 1 §3; statuses `received → acknowledged → under_review → (requested_full) → accepted/rejected/withdrawn`. Manuscript files to S3. Accepted submissions link to a new `work`. Email acknowledgement via transactional provider.

**Testing**:
- `Integration: submit manuscript → stored, acknowledged email enqueued, appears in queue`
- `Integration: accept submission → work created and linked`

---

## Phase 8: Rights, Sub-Rights & ONIX Distribution

### Purpose
Complete the rights lifecycle and outbound metadata distribution. Territory/language grants with reversion alerts, sub-licensing revenue intake feeding the royalty engine, and ONIX 3.1 export — mandatory for retailer/library distribution.

### Tasks

#### 8.1 — Rights register & reversion alerts

**What**: Territory/language rights grants with expiry and reversion tracking.

**Design**: `rights_grants`, `rights_grant_territories`, `sub_licenses` per Suggestion 1 §4. A scheduled Celery beat job scans for approaching `grant_end`/reversion triggers and raises alerts.

**Testing**:
- `Unit: world grant minus exclusion list → availability matrix correct`
- `Integration: grant 30 days from expiry → alert generated`

#### 8.2 — Sub-licensing revenue intake

**What**: Record sub-license deals and split revenue between publisher and author.

**Design**: Sub-license income enters the royalty engine as a statement line (`sub_license_income`) split per `royalty_split_pct`/`author_split_pct`.

**Testing**:
- `Unit: $10k sub-license, 50/50 split → $5k author line`
- `Integration: sub-license income appears on next statement`

#### 8.3 — ONIX 3.1 export

**What**: Generate and validate ONIX 3.1 product messages.

**Design**: `onix.py` builds ONIX 3.1 XML (Descriptive Detail, Publishing Detail, Product Supply) from products + metadata + subjects + prices, validated against the vendored EDItEUR XSD. `onix_exports` tracks jobs. Includes EPUB Accessibility 1.1 metadata for digital products (EAA compliance).

**Testing**:
- `Integration: export product → XML validates against ONIX 3.1 XSD`
- `Fixture: golden ONIX file comparison for a known product`
- `Unit: ebook product → accessibility metadata block present`

---

## Phase 9: Author Portal

### Purpose
Self-service access for authors and agents — royalty statements, sales charts, and contract documents — reducing publisher admin load and differentiating against royalty-only incumbents.

### Tasks

#### 9.1 — Portal API & data scoping

**What**: Author-scoped read endpoints over statements, sales, and documents.

**Design**: `portal_users` (Phase 1) authenticate via JWT; endpoints filter strictly to the author's contracts (OWASP broken-access-control hardening). Aggregated sales summary by format/channel.

**Testing**:
- `Integration: author A requesting author B's statement → 403`
- `Integration: portal sales summary matches sales_transactions aggregate`

#### 9.2 — Portal SPA

**What**: React portal: statement list/download, sales charts, contract viewer.

**Design**: Separate SPA, shared `ui` package, WCAG 2.2 AA.

**Testing**:
- `E2E: author logs in → views statement → downloads PDF`
- `E2E: axe-core scan → no AA violations`

---

## Phase 10: AI-Native Features

### Purpose
Deliver the AI-native advantages that incumbents lack: manuscript triage scoring, sales forecasting, royalty anomaly detection, and contract-clause analysis. All sit behind a provider interface and write results back as advisory fields (never silently altering financial records).

### Tasks

#### 10.1 — LLM provider interface

**What**: Pluggable LLM abstraction with structured output and cost tracking.

**Design**:
```python
class LLMProvider(Protocol):
    async def complete(self, system: str, user: str,
                       response_schema: type[BaseModel]) -> BaseModel: ...
```
- Default Anthropic Claude implementation; prompts versioned (`model_version` stored alongside results).

**Testing**:
- `Unit (mocked): malformed LLM JSON → retry then ProviderError`
- `Integration (recorded): triage prompt → schema-valid score`

#### 10.2 — Manuscript triage scoring

**What**: Score slush-pile submissions against editorial criteria.

**Design**: Writes `ai_quality_score` (0–100) + `ai_genre_tags` to `submissions`; advisory only, surfaced in the triage queue sorted by score. System prompt template captured in `ai/triage.py`.

**Testing**:
- `Unit (mocked): score persisted, queue re-sorts by score`
- `Integration: scoring never changes submission decision fields`

#### 10.3 — Sales forecast, anomaly detection, clause analysis

**What**: First-year forecast for acquisitions; statement anomaly flags; sub-rights clause review.

**Design**: Forecast uses comparable-title sales features. Anomaly detection compares a distributor batch against historical baselines and flags outliers as advisory notes (no auto-adjustment of royalties). Clause analysis extracts and flags unusual sub-rights terms from uploaded contract text.

**Testing**:
- `Unit: anomaly flag is advisory — royalty_calculations untouched`
- `Integration (mocked): forecast returns ranged estimate with confidence`
- `Unit: clause analysis flags below-market royalty term`

---

## Phase 11: Integrations, Payments & Hardening

### Purpose
Production-readiness: Stripe Connect payouts, ISBN-agency export, an MCP server for AI-agent access, and security/compliance hardening (GDPR, PCI scope, OWASP). Final phase before GA.

### Tasks

#### 11.1 — Stripe Connect payouts

**What**: Disburse advances and royalty payments via Stripe Connect.

**Design**: Author onboarding via Connect OAuth; payouts triggered from approved statements; webhook updates `paid_at`/`payment_reference`. Tokenised — no card data stored (minimises PCI DSS scope).

**Testing**:
- `Integration (Stripe test mode): approved statement → payout created, webhook marks paid`
- `Integration: webhook signature invalid → 401, no state change`

#### 11.2 — ISBN-agency export & MCP server

**What**: Batch ISBN registration export (Nielsen/Bowker) and an MCP server exposing read tools.

**Design**: Agency export produces the agency-specific batch file (no public APIs exist — file/EDI based per `standards.md`). MCP server (JSON-RPC 2.0 over HTTP/SSE) exposes read-only tools: query metadata, fetch royalty summary, list rights availability.

**Testing**:
- `Integration: export batch matches agency template`
- `Integration: MCP tool call returns scoped, tenant-isolated data`

#### 11.3 — GDPR, security & compliance hardening

**What**: Right-to-erasure flows, transparency, and an OWASP pass.

**Design**: Author data export + erasure (anonymise while preserving immutable financial records as required for audit). Security review against OWASP Top Ten; rate limiting; dependency scanning in CI.

**Testing**:
- `Integration: erasure request anonymises PII but retains statement integrity`
- `Integration: Schemathesis + ZAP baseline → no high-severity findings`
- `Integration: API rate limit enforced (429 after threshold)`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                  ─── required by everything
    │
Phase 2: Title & ISBN                ─── requires 1
    │
Phase 3: Royalty Engine              ─── requires 1, 2
    │
Phase 4: Sales Ingestion             ─── requires 2, 3
    │
Phase 5: Statements                  ─── requires 3, 4
    │
Phase 6: API + Staff Console         ─── requires 2-5
    ├── Phase 7: Editorial/Submissions ─── requires 2, 6 · parallel with 8
    ├── Phase 8: Rights & ONIX         ─── requires 2, 3, 6 · parallel with 7
    └── Phase 9: Author Portal         ─── requires 5, 6 · parallel with 7, 8
         │
Phase 10: AI Features                ─── requires 4, 5, 7 (submissions)
    │
Phase 11: Integrations & Hardening   ─── requires 5, 6, 9
```

**Parallelism opportunities:**
- Phases 7, 8, and 9 can be developed concurrently once Phase 6 ships the API + console foundation.
- Within Phase 4, the three distributor parsers (4.2) can be built in parallel.
- Frontend (`console`, `portal`) work can begin against the OpenAPI mock as soon as 6.1 publishes the spec.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks implemented as specified.
2. All unit and integration tests pass; new test scenarios listed in the phase are covered.
3. Ruff lint + format pass with zero warnings.
4. mypy (strict) passes with no `type: ignore` added without justification.
5. `docker compose build` succeeds and the stack starts healthy.
6. The phase's feature works end-to-end (demonstrated by its E2E or integration test).
7. New configuration options documented in `README.md` and `.env.example`.
8. New/changed API endpoints appear correctly in the generated OpenAPI 3.1 spec.
9. Alembic migration(s) created, reversible, and applied cleanly to a fresh database.
10. For financial tasks: append-only immutability enforced and audit-log coverage verified.
11. For UI tasks: axe-core accessibility scan shows no WCAG 2.2 AA violations.
