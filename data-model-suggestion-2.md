# Data Model Suggestion 2: Event-Sourced / CQRS Model

## Overview

This model applies Event Sourcing (ES) and Command Query Responsibility Segregation (CQRS) to the Book Publishing Platform. Every state change -- a manuscript submission, a contract execution, a royalty calculation, a rights grant -- is captured as an immutable domain event in an append-only event store. The current state of any aggregate (a Work, a Contract, a Royalty Account) is derived by replaying its event stream.

This architecture is particularly well-suited to book publishing because:

1. **Royalty accounting demands an immutable audit trail.** Regulators, auditors, and authors need to see exactly how every royalty figure was derived. Event sourcing makes the audit trail the system of record, not an afterthought.
2. **Contract and rights lifecycles span decades.** A single book contract may be active for 20+ years with amendments, reversions, and sub-license additions. Event sourcing preserves the full history of every change without temporal columns or soft deletes.
3. **Royalty calculations must be reproducible.** If an author disputes a statement, the publisher must be able to replay the exact calculation with the exact data and rates that existed at the time. Event sourcing makes this deterministic.
4. **Editorial workflows are inherently event-driven.** Manuscript submitted, review assigned, edits requested, proofs approved -- these are natural domain events.

---

## Core Architecture

```
                    ┌──────────────┐
                    │   Commands   │
                    │  (Write API) │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │   Command    │
                    │   Handlers   │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  Aggregates  │  ◄── Business rules / invariants
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  Event Store │  ◄── Append-only, immutable
                    │ (PostgreSQL) │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
       ┌──────▼──────┐ ┌──▼─────┐ ┌───▼──────┐
       │  Projection  │ │Catalog │ │ Royalty   │
       │  (Title View)│ │Search  │ │Dashboard │
       │  PostgreSQL  │ │Elastic │ │PostgreSQL│
       └─────────────┘ └────────┘ └──────────┘
              Read Models (Query Side)
```

---

## Event Store Schema (PostgreSQL)

```sql
-- The event store: the single source of truth
CREATE TABLE events (
    event_id            BIGSERIAL PRIMARY KEY,
    stream_id           UUID NOT NULL,              -- Aggregate ID (e.g., work_id, contract_id)
    stream_type         VARCHAR(100) NOT NULL,      -- Aggregate type (e.g., 'Work', 'Contract', 'RoyaltyAccount')
    event_type          VARCHAR(200) NOT NULL,      -- e.g., 'ManuscriptSubmitted', 'ContractExecuted'
    event_version       INTEGER NOT NULL,           -- Version within this stream (for optimistic concurrency)
    event_data          JSONB NOT NULL,             -- The event payload
    metadata            JSONB NOT NULL DEFAULT '{}',-- Correlation IDs, user info, causation
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by          UUID,                       -- User who triggered the event
    UNIQUE(stream_id, event_version)                -- Optimistic concurrency control
);

-- Indexes for efficient stream replay and event type queries
CREATE INDEX idx_events_stream ON events(stream_id, event_version);
CREATE INDEX idx_events_type ON events(event_type);
CREATE INDEX idx_events_created ON events(created_at);
CREATE INDEX idx_events_stream_type ON events(stream_type);

-- Aggregate version tracking for optimistic concurrency
CREATE TABLE aggregates (
    aggregate_id        UUID PRIMARY KEY,
    aggregate_type      VARCHAR(100) NOT NULL,
    current_version     INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Snapshot store for performance (avoid replaying long event streams)
CREATE TABLE snapshots (
    snapshot_id         BIGSERIAL PRIMARY KEY,
    stream_id           UUID NOT NULL,
    stream_type         VARCHAR(100) NOT NULL,
    snapshot_version    INTEGER NOT NULL,           -- The event version this snapshot represents
    snapshot_data       JSONB NOT NULL,             -- Serialized aggregate state
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(stream_id, snapshot_version)
);

CREATE INDEX idx_snapshots_stream ON snapshots(stream_id, snapshot_version DESC);

-- Subscription checkpoints for projections (idempotent replay)
CREATE TABLE projection_checkpoints (
    projection_name     VARCHAR(200) PRIMARY KEY,
    last_event_id       BIGINT NOT NULL DEFAULT 0,
    last_processed_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    status              VARCHAR(30) NOT NULL DEFAULT 'running' CHECK (status IN (
        'running', 'paused', 'rebuilding', 'error'
    )),
    error_message       TEXT
);

-- Dead letter queue for failed event processing
CREATE TABLE dead_letter_events (
    dead_letter_id      BIGSERIAL PRIMARY KEY,
    event_id            BIGINT NOT NULL REFERENCES events(event_id),
    projection_name     VARCHAR(200) NOT NULL,
    error_message       TEXT NOT NULL,
    retry_count         INTEGER NOT NULL DEFAULT 0,
    max_retries         INTEGER NOT NULL DEFAULT 5,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_retry_at       TIMESTAMPTZ
);
```

---

## Domain Events by Aggregate

### Work Aggregate

The Work aggregate tracks the intellectual property from acquisition through publication. Its event stream records the full lifecycle.

```
Stream Type: "Work"
Stream ID: work_id (UUID)
```

| Event Type | Payload Fields | Description |
|-----------|---------------|-------------|
| `WorkProposed` | title, subtitle, work_type, synopsis, proposed_by | A new work enters the catalog |
| `WorkMetadataUpdated` | changed_fields (object of field:value) | Title, synopsis, or classification changed |
| `ContributorAdded` | person_id, role, sequence, contribution_pct | Author/editor/illustrator linked |
| `ContributorRemoved` | person_id, role, reason | Contributor removed from work |
| `SeriesAssigned` | series_id, series_name, position | Work placed in a series |
| `WorkAcquired` | acquisition_date, acquiring_editor_id, contract_id | Formal acquisition decision |
| `WorkAbandoned` | reason, abandoned_by | Work cancelled before publication |

**Event Payload Examples:**

```json
{
    "event_type": "WorkProposed",
    "event_data": {
        "title": "The Midnight Library",
        "subtitle": null,
        "work_type": "monograph",
        "original_language": "eng",
        "synopsis": "Between life and death there is a library...",
        "proposed_by": "a3f4e5d6-7890-1234-5678-abcdef012345",
        "genre_tags": ["literary_fiction", "speculative"]
    },
    "metadata": {
        "correlation_id": "cmd-2026-001234",
        "source": "editorial_dashboard",
        "ip_address": "10.0.1.42"
    }
}
```

```json
{
    "event_type": "ContributorAdded",
    "event_data": {
        "person_id": "b2c3d4e5-6789-0123-4567-bcdef0123456",
        "display_name": "Matt Haig",
        "role": "author",
        "sequence_number": 1,
        "contribution_pct": 100.00
    }
}
```

### Product Aggregate

Each product (ISBN-level format/edition) has its own event stream.

```
Stream Type: "Product"
Stream ID: product_id (UUID)
```

| Event Type | Payload Fields | Description |
|-----------|---------------|-------------|
| `ProductCreated` | work_id, product_form, isbn_13, publisher_id, imprint_id | New format/edition registered |
| `ISBNAssigned` | isbn_13, isbn_10, prefix_id | ISBN allocated from publisher prefix |
| `PricingSet` | list_price, currency, effective_date | Price established or changed |
| `PublicationDateSet` | publication_date, on_sale_date | Dates confirmed |
| `ProductPublished` | publication_date | Product officially published |
| `PhysicalSpecsSet` | page_count, trim_width_mm, trim_height_mm, weight_grams | Physical dimensions recorded |
| `ProductRetired` | out_of_print_date, reason | Product taken out of print |
| `SubjectClassificationSet` | scheme, code, heading_text, is_primary | BISAC/THEMA code assigned |
| `ONIXExported` | export_id, recipient, onix_version, timestamp | ONIX data sent to trading partner |

### Editorial Pipeline Aggregate

```
Stream Type: "EditorialPipeline"
Stream ID: work_id (UUID) -- one pipeline per work
```

| Event Type | Payload Fields | Description |
|-----------|---------------|-------------|
| `PipelineInitialized` | template_id, template_name, stages[] | Editorial workflow started |
| `StageEntered` | stage_id, stage_name, entered_at | Work moves to next stage |
| `TaskAssigned` | task_id, stage_id, assigned_to, task_type, due_date | Task given to person |
| `TaskCompleted` | task_id, completed_by, completed_at, notes | Task finished |
| `StageApproved` | stage_id, approved_by, approved_at | Stage signed off |
| `StageRejected` | stage_id, rejected_by, reason, return_to_stage_id | Stage sent back |
| `MilestoneReached` | stage_id, milestone_name, triggers_payment | Contractual milestone hit |
| `PipelineCompleted` | completed_at | All stages done |

### Submission Aggregate

```
Stream Type: "Submission"
Stream ID: submission_id (UUID)
```

| Event Type | Payload Fields | Description |
|-----------|---------------|-------------|
| `SubmissionReceived` | title, author_name, author_email, agent_name, word_count, synopsis, file_url | New manuscript arrives |
| `SubmissionAcknowledged` | acknowledged_at, auto_response_sent | Automated acknowledgement sent |
| `AITriageCompleted` | quality_score, genre_tags, confidence, model_version | AI scoring applied |
| `ReviewerAssigned` | reviewer_id, assigned_at | Editor assigned to review |
| `FullManuscriptRequested` | requested_by, requested_at, message | Partial submission prompts full request |
| `SubmissionAccepted` | accepted_by, work_id, notes | Manuscript accepted for acquisition |
| `SubmissionRejected` | rejected_by, reason, rejection_template | Manuscript declined |
| `SubmissionWithdrawn` | withdrawn_by, reason | Author/agent withdraws |

### Contract Aggregate

```
Stream Type: "Contract"
Stream ID: contract_id (UUID)
```

| Event Type | Payload Fields | Description |
|-----------|---------------|-------------|
| `ContractDrafted` | contract_number, work_id, contract_type, counterparty_person_id, counterparty_org_id | Contract initiated |
| `TermsNegotiated` | proposed_terms (advance, rates, territories, rights) | Terms under discussion |
| `ContractExecuted` | execution_date, effective_date, document_url | Contract signed |
| `AdvanceScheduled` | tranches[] (amount, currency, trigger_type, trigger_date) | Payment schedule established |
| `AdvanceTranchePaid` | tranche_number, amount, currency, paid_date, payment_ref | Advance installment paid |
| `RoyaltyRateSet` | rate_schedule (product_form, channel, territory, basis, tiers[]) | Rate schedule defined |
| `RoyaltyRateAmended` | old_schedule, new_schedule, effective_date, reason | Rate changed mid-contract |
| `AgentCommissionSet` | agent_person_id, commission_pct | Agent's cut defined |
| `ContractAmended` | amendment_number, changes, effective_date | General amendment |
| `ContractTerminated` | termination_date, reason | Contract ended early |
| `ContractExpired` | expiration_date | Natural expiry |

**Event Payload Example:**

```json
{
    "event_type": "RoyaltyRateSet",
    "event_data": {
        "rate_schedule_id": "rs-001",
        "product_form": "hardback",
        "sales_channel": null,
        "territory_code": null,
        "rate_basis": "list_price",
        "tiers": [
            {"floor": 0, "ceiling": 5000, "rate_pct": 10.0},
            {"floor": 5001, "ceiling": 10000, "rate_pct": 12.5},
            {"floor": 10001, "ceiling": null, "rate_pct": 15.0}
        ],
        "effective_from": "2026-01-01"
    }
}
```

### Rights Aggregate

```
Stream Type: "RightsPortfolio"
Stream ID: work_id (UUID) -- all rights for a work in one stream
```

| Event Type | Payload Fields | Description |
|-----------|---------------|-------------|
| `RightsGranted` | grant_id, contract_id, right_type, territory_type, countries[], language_code, is_exclusive, start, end | Rights granted to publisher |
| `RightsReverted` | grant_id, reversion_date, reason | Rights returned to author |
| `RightsExpired` | grant_id, expiration_date | Rights grant naturally expired |
| `SubLicenseGranted` | sub_license_id, parent_grant_id, licensee_org_id, territory, language, advance, splits | Sub-rights sold |
| `SubLicenseRevenuReceived` | sub_license_id, amount, currency, payment_date | Income from sub-licensee |
| `SubLicenseTerminated` | sub_license_id, termination_date, reason | Sub-license ended |
| `ReversionAlertTriggered` | grant_id, trigger_condition, alert_date | System detects approaching reversion |

### Royalty Account Aggregate

This is the financial heart of the system. One royalty account per contract tracks cumulative sales, advance balances, reserves, and payments.

```
Stream Type: "RoyaltyAccount"
Stream ID: contract_id (UUID)
```

| Event Type | Payload Fields | Description |
|-----------|---------------|-------------|
| `RoyaltyAccountOpened` | contract_id, work_id, payee_person_id, currency | Account created when contract activates |
| `AdvanceRecorded` | tranche_number, amount, currency, date | Advance payment recorded to balance |
| `SalesDataImported` | batch_id, period_id, distributor_id, line_count, total_units, total_gross | Sales data batch imported |
| `SalesLineRecorded` | product_id, channel, territory, type (sale/return), quantity, unit_price, gross, net, currency | Individual sales line |
| `RoyaltyCalculated` | period_id, product_id, net_units, cumulative_units, royalty_basis, rate_applied, calculated_amount | Per-product royalty computed |
| `ReserveHeld` | period_id, reserve_pct, amount_held | Reserve against returns withheld |
| `ReserveReleased` | period_id, original_period_id, amount_released | Prior reserve released |
| `AdvanceRecouped` | period_id, amount_recouped, remaining_balance | Earned royalties offset advance |
| `AdvanceEarnedOut` | earned_out_date, total_advance, total_earned | Advance fully recouped |
| `AgentCommissionCalculated` | period_id, commission_pct, commission_amount | Agent's share computed |
| `RoyaltyStatementGenerated` | statement_id, period_id, total_earned, reserves, recoupment, net_payable | Statement created |
| `RoyaltyStatementSent` | statement_id, sent_to, sent_at, delivery_method | Statement delivered to author/agent |
| `RoyaltyPaymentMade` | statement_id, amount, currency, payment_date, payment_ref | Payment executed |
| `RoyaltyAdjustment` | period_id, reason, adjustment_type, amount, authorized_by | Manual correction |
| `RoyaltyOverrideApplied` | period_id, original_amount, override_amount, reason, authorized_by | Manual override with justification |
| `AnomalyDetected` | period_id, anomaly_type, description, severity, suggested_action | AI flags inconsistency |

**Event Payload Example (Royalty Calculation):**

```json
{
    "event_type": "RoyaltyCalculated",
    "event_data": {
        "period_id": "period-2026-h1",
        "product_id": "prod-hardback-001",
        "product_form": "hardback",
        "net_units": 3200,
        "cumulative_units_before": 8500,
        "cumulative_units_after": 11700,
        "rate_tiers_applied": [
            {
                "tier": {"floor": 0, "ceiling": 10000, "rate_pct": 12.5},
                "units_in_tier": 1500,
                "basis_amount_per_unit": 24.99,
                "tier_royalty": 4685.63
            },
            {
                "tier": {"floor": 10001, "ceiling": null, "rate_pct": 15.0},
                "units_in_tier": 1700,
                "basis_amount_per_unit": 24.99,
                "tier_royalty": 6372.45
            }
        ],
        "total_calculated_royalty": 11058.08,
        "calculation_currency": "USD",
        "rate_basis": "list_price"
    }
}
```

### Sales Import Aggregate

```
Stream Type: "SalesImport"
Stream ID: batch_id (UUID)
```

| Event Type | Payload Fields | Description |
|-----------|---------------|-------------|
| `ImportBatchCreated` | distributor_id, file_name, file_format, period_id | New import started |
| `ImportValidationStarted` | row_count, validation_rules | Validation begun |
| `ImportValidationCompleted` | valid_rows, invalid_rows, warnings[] | Validation results |
| `ImportValidationFailed` | errors[], rejected_rows | Validation failed |
| `ImportCompleted` | total_units, total_gross, matched_isbns, unmatched_isbns | Import finished |
| `ISBNMappingResolved` | unmatched_isbn, resolved_product_id, resolution_method | Manual or automated ISBN match |

---

## Read Model Projections

Read models are built by subscribing to the event stream and maintaining denormalized query-optimized tables. Each projection can be rebuilt from scratch by replaying events from the beginning.

### Projection 1: Title Catalog View

```sql
-- Denormalized view of works with all products, optimized for browsing
CREATE TABLE rm_title_catalog (
    work_id             UUID PRIMARY KEY,
    title               VARCHAR(1000) NOT NULL,
    subtitle            VARCHAR(1000),
    work_type           VARCHAR(50),
    publication_status  VARCHAR(50),
    primary_author      VARCHAR(500),
    all_contributors    JSONB,          -- [{name, role, sequence}]
    series_name         VARCHAR(500),
    series_position     DECIMAL(5,1),
    synopsis            TEXT,
    acquisition_date    DATE,
    first_publication   DATE,
    formats_available   JSONB,          -- [{form, isbn, price, currency, pub_date}]
    primary_bisac       VARCHAR(50),
    primary_thema       VARCHAR(50),
    all_subjects        JSONB,          -- [{scheme, code, heading}]
    cover_image_url     VARCHAR(1000),
    publisher_name      VARCHAR(500),
    imprint_name        VARCHAR(500),
    last_updated        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rm_catalog_title ON rm_title_catalog USING gin(to_tsvector('english', title));
CREATE INDEX idx_rm_catalog_author ON rm_title_catalog(primary_author);
CREATE INDEX idx_rm_catalog_status ON rm_title_catalog(publication_status);
CREATE INDEX idx_rm_catalog_pub_date ON rm_title_catalog(first_publication);
```

### Projection 2: Contract Dashboard

```sql
CREATE TABLE rm_contract_dashboard (
    contract_id         UUID PRIMARY KEY,
    contract_number     VARCHAR(100),
    work_id             UUID,
    work_title          VARCHAR(1000),
    contract_type       VARCHAR(50),
    counterparty_name   VARCHAR(500),
    agent_name          VARCHAR(500),
    status              VARCHAR(30),
    execution_date      DATE,
    expiration_date     DATE,
    total_advance       DECIMAL(12,2),
    advance_paid        DECIMAL(12,2),
    advance_remaining   DECIMAL(12,2),
    is_earned_out       BOOLEAN DEFAULT FALSE,
    current_unearned    DECIMAL(12,2),
    total_royalties_paid DECIMAL(14,2) DEFAULT 0,
    rate_summary        JSONB,          -- Summary of rate schedules
    rights_summary      JSONB,          -- Summary of rights grants
    upcoming_payments   JSONB,          -- Next scheduled advance tranches
    contract_currency   CHAR(3),
    last_updated        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rm_contracts_status ON rm_contract_dashboard(status);
CREATE INDEX idx_rm_contracts_expiry ON rm_contract_dashboard(expiration_date);
```

### Projection 3: Royalty Statement View

```sql
CREATE TABLE rm_royalty_statements (
    statement_id        UUID PRIMARY KEY,
    contract_id         UUID NOT NULL,
    contract_number     VARCHAR(100),
    period_name         VARCHAR(100),
    period_start        DATE,
    period_end          DATE,
    payee_name          VARCHAR(500),
    payee_person_id     UUID,
    work_title          VARCHAR(1000),
    statement_date      DATE,
    line_items          JSONB,          -- [{product, form, channel, units, price, rate, amount}]
    total_earned        DECIMAL(14,2),
    reserves_held       DECIMAL(14,2),
    reserves_released   DECIMAL(14,2),
    advance_recoupment  DECIMAL(14,2),
    agent_commission    DECIMAL(14,2),
    net_payable         DECIMAL(14,2),
    payment_currency    CHAR(3),
    status              VARCHAR(30),
    payment_date        DATE,
    payment_reference   VARCHAR(200),
    last_updated        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rm_statements_payee ON rm_royalty_statements(payee_person_id);
CREATE INDEX idx_rm_statements_contract ON rm_royalty_statements(contract_id);
CREATE INDEX idx_rm_statements_period ON rm_royalty_statements(period_start);
```

### Projection 4: Rights Availability Matrix

```sql
CREATE TABLE rm_rights_matrix (
    work_id             UUID NOT NULL,
    right_type          VARCHAR(50) NOT NULL,
    country_code        CHAR(2) NOT NULL,
    language_code       CHAR(3),
    status              VARCHAR(30) NOT NULL,  -- 'available', 'granted', 'sub_licensed', 'expired'
    grant_id            UUID,
    contract_id         UUID,
    licensee_name       VARCHAR(500),
    is_exclusive        BOOLEAN,
    grant_start         DATE,
    grant_end           DATE,
    sub_license_id      UUID,
    sub_licensee_name   VARCHAR(500),
    last_updated        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (work_id, right_type, country_code, COALESCE(language_code, '---'))
);

CREATE INDEX idx_rm_rights_status ON rm_rights_matrix(status);
CREATE INDEX idx_rm_rights_work ON rm_rights_matrix(work_id);
```

### Projection 5: Author Portal View

```sql
CREATE TABLE rm_author_portal (
    person_id           UUID NOT NULL,
    contract_id         UUID NOT NULL,
    work_title          VARCHAR(1000),
    work_id             UUID,
    contract_number     VARCHAR(100),
    total_advance       DECIMAL(12,2),
    advance_remaining   DECIMAL(12,2),
    is_earned_out       BOOLEAN,
    latest_statement    JSONB,          -- Most recent statement summary
    all_statements      JSONB,          -- [{period, earned, net_payable, status}]
    sales_summary       JSONB,          -- Aggregated sales by format/channel
    total_units_sold    INTEGER DEFAULT 0,
    total_royalties     DECIMAL(14,2) DEFAULT 0,
    contract_documents  JSONB,          -- [{doc_type, url, date}]
    last_updated        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (person_id, contract_id)
);

CREATE INDEX idx_rm_author_portal_person ON rm_author_portal(person_id);
```

### Projection 6: Editorial Pipeline Status

```sql
CREATE TABLE rm_editorial_pipeline (
    work_id             UUID PRIMARY KEY,
    title               VARCHAR(1000),
    primary_author      VARCHAR(500),
    pipeline_template   VARCHAR(200),
    current_stage       VARCHAR(200),
    current_stage_order INTEGER,
    total_stages        INTEGER,
    assigned_to         VARCHAR(500),
    deadline            DATE,
    days_in_stage       INTEGER,
    is_overdue          BOOLEAN DEFAULT FALSE,
    stage_history       JSONB,          -- [{stage, entered, completed, assigned_to}]
    open_tasks          JSONB,          -- [{task_id, title, assigned_to, due_date, status}]
    milestones_reached  JSONB,          -- [{milestone, date, triggered_payment}]
    last_updated        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rm_pipeline_stage ON rm_editorial_pipeline(current_stage);
CREATE INDEX idx_rm_pipeline_overdue ON rm_editorial_pipeline(is_overdue) WHERE is_overdue = TRUE;
```

### Projection 7: Submission Triage Queue

```sql
CREATE TABLE rm_submission_queue (
    submission_id       UUID PRIMARY KEY,
    title               VARCHAR(1000),
    author_name         VARCHAR(500),
    agent_name          VARCHAR(500),
    genre               VARCHAR(200),
    word_count          INTEGER,
    submitted_at        TIMESTAMPTZ,
    status              VARCHAR(50),
    ai_quality_score    DECIMAL(5,2),
    ai_genre_tags       TEXT[],
    reviewer_name       VARCHAR(500),
    reviewer_id         UUID,
    days_since_submission INTEGER,
    last_updated        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rm_submissions_status ON rm_submission_queue(status);
CREATE INDEX idx_rm_submissions_score ON rm_submission_queue(ai_quality_score DESC);
```

### Projection 8: Sales Analytics

```sql
CREATE TABLE rm_sales_analytics (
    product_id          UUID NOT NULL,
    period_id           UUID NOT NULL,
    work_id             UUID,
    work_title          VARCHAR(1000),
    product_form        VARCHAR(50),
    isbn_13             CHAR(13),
    channel             VARCHAR(100),
    territory           CHAR(2),
    units_sold          INTEGER DEFAULT 0,
    units_returned      INTEGER DEFAULT 0,
    net_units           INTEGER DEFAULT 0,
    gross_revenue       DECIMAL(14,2) DEFAULT 0,
    net_revenue         DECIMAL(14,2) DEFAULT 0,
    reporting_currency  CHAR(3),
    period_start        DATE,
    period_end          DATE,
    last_updated        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (product_id, period_id, channel, territory)
);

CREATE INDEX idx_rm_sales_work ON rm_sales_analytics(work_id);
CREATE INDEX idx_rm_sales_period ON rm_sales_analytics(period_start);
```

---

## Command Handlers (Application Layer)

Commands validate business rules before emitting events. Here are the key command flows:

### Calculate Royalties Command

```
Command: CalculateRoyalties
Input: contract_id, period_id

Flow:
1. Load RoyaltyAccount aggregate (replay events or load snapshot)
2. Load Contract aggregate to get current rate schedules
3. Query SalesDataImported events for this period
4. For each product under the contract:
   a. Sum net units (sales - returns)
   b. Determine cumulative units (from aggregate state)
   c. Apply escalating rate tiers based on cumulative position
   d. Emit RoyaltyCalculated event with full tier breakdown
5. Calculate reserve against returns:
   a. Apply reserve percentage to gross royalties
   b. Emit ReserveHeld event
   c. Check for releasable reserves from prior periods
   d. Emit ReserveReleased events
6. Check advance balance:
   a. If unearned advance remains, apply recoupment
   b. Emit AdvanceRecouped event
   c. If balance reaches zero, emit AdvanceEarnedOut
7. Calculate agent commission on net payable
   a. Emit AgentCommissionCalculated
8. Generate statement
   a. Emit RoyaltyStatementGenerated with all line items
```

### Import Sales Data Command

```
Command: ImportSalesData
Input: distributor_id, file_path, file_format, period_id

Flow:
1. Create SalesImport aggregate
2. Emit ImportBatchCreated
3. Parse file using format-specific parser (Ingram CSV, Amazon KDP, etc.)
4. Validate each row:
   a. ISBN lookup against Product aggregate
   b. Currency validation
   c. Duplicate detection
5. Emit ImportValidationCompleted or ImportValidationFailed
6. For each valid row, emit SalesLineRecorded on the RoyaltyAccount aggregate
7. Emit ImportCompleted with summary
```

---

## Snapshot Strategy

Long-lived aggregates (contracts active for decades, royalty accounts with years of sales data) will accumulate thousands of events. Snapshots prevent replay from becoming slow.

```
Snapshot Policy:
- RoyaltyAccount: Snapshot every 100 events
- Contract: Snapshot every 50 events
- Work: Snapshot every 30 events
- EditorialPipeline: No snapshots needed (short-lived, few events)
- Submission: No snapshots needed (short-lived)

Snapshot Content (RoyaltyAccount):
{
    "contract_id": "...",
    "work_id": "...",
    "payee_person_id": "...",
    "currency": "USD",
    "total_advance": 50000.00,
    "advance_paid": 50000.00,
    "advance_recouped": 42000.00,
    "advance_remaining": 8000.00,
    "is_earned_out": false,
    "cumulative_units_by_product": {
        "prod-hb-001": 11700,
        "prod-pb-001": 28400,
        "prod-eb-001": 45200
    },
    "reserves_held": [
        {"period_id": "p-2025-h2", "amount": 1200.00, "release_period": "p-2026-h2"}
    ],
    "total_royalties_earned": 42000.00,
    "total_royalties_paid": 0.00,
    "last_statement_period": "p-2025-h2",
    "snapshot_version": 847
}
```

---

## Pros and Cons

### Pros

1. **Perfect audit trail**: Every state change is recorded permanently. For royalty disputes, you can replay the exact sequence of events that produced any historical statement. This is the gold standard for financial auditability.
2. **Temporal queries are free**: "What was the contract's rate schedule on March 15, 2024?" -- replay events up to that date. No temporal columns, no bi-temporal joins, no SCD Type 2 complexity.
3. **Deterministic reproduction**: Royalty calculations can be re-run against the same event stream to verify correctness. If a bug is found in the calculation logic, you can fix the code and replay to produce corrected statements with full traceability.
4. **Natural fit for domain events**: The publishing lifecycle (submission received, manuscript accepted, contract signed, advance paid, book published, sales reported, royalty calculated) is inherently event-driven. The code reads like the business process.
5. **Independent read model scaling**: The author portal, editorial dashboard, and royalty reports each have their own projection, independently scalable and rebuildable without affecting the write side.
6. **Retroactive projections**: Need a new report that did not exist when the system launched? Build a new projection and replay all events. No data loss from not having anticipated the requirement.
7. **Conflict resolution**: Optimistic concurrency on the event stream prevents conflicting writes (e.g., two editors approving the same stage simultaneously).
8. **Compliance-friendly**: Immutable event logs satisfy regulatory requirements for financial record-keeping without additional audit infrastructure.

### Cons

1. **Complexity overhead**: Event sourcing is significantly more complex than CRUD. Every operation requires defining commands, events, aggregates, and projections. The learning curve is steep for teams unfamiliar with the pattern.
2. **Eventual consistency in read models**: Projections update asynchronously after events are written. The author portal might show a stale royalty statement for a few seconds after calculation. This requires careful UX design (loading states, explicit refresh).
3. **Event schema evolution**: Over years, event payloads will need to change (new fields, renamed fields, deprecated fields). Upcasting (transforming old events to match new schemas) adds ongoing maintenance complexity.
4. **Query complexity for ad hoc reporting**: You cannot run a simple SQL query against the event store to answer "which contracts expire in the next 90 days?" -- you must query the appropriate read model projection. If the projection does not exist, you must build one.
5. **Debugging difficulty**: When a royalty statement is wrong, debugging requires tracing through event replay rather than inspecting a row in a table. This requires specialized tooling.
6. **Storage growth**: The event store grows monotonically. A busy platform might generate millions of SalesLineRecorded events per period. Unlike CRUD, you never delete or compact production data.
7. **Snapshot management**: Long-lived aggregates require snapshot infrastructure to maintain acceptable replay performance. This is additional code and operational complexity.
8. **Team skill requirements**: Most developers are more comfortable with CRUD. Recruiting and training for event sourcing adds organizational cost.

---

## Technology Recommendations

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| **Event store** | PostgreSQL with the schema above | Avoids introducing a new database technology; leverages ACID for event writes; JSONB for flexible event payloads |
| **Alternative event store** | EventStoreDB | Purpose-built for event sourcing with built-in projections, subscriptions, and catch-up semantics. Consider if the team has experience with it |
| **Message broker** | Apache Kafka or RabbitMQ | For publishing events to external consumers (ONIX export service, accounting system integration). Kafka if high throughput; RabbitMQ if simpler setup |
| **Projection engine** | Custom projection service (Node.js or Python) | Subscribes to PostgreSQL LISTEN/NOTIFY or polls the events table; updates read model tables |
| **Search projection** | Elasticsearch or OpenSearch | For full-text catalog search if the read model in PostgreSQL is insufficient |
| **Framework** | Axon Framework (Java), Marten (C#/.NET), or custom (Node.js/Python) | Axon and Marten provide CQRS/ES infrastructure out of the box. Custom is viable for smaller teams |
| **Monitoring** | Projection lag dashboard | Critical to monitor the gap between the latest event and each projection's checkpoint |

---

## Migration and Scaling Considerations

### Event Store Partitioning

```sql
-- Partition events by creation date for archival and query performance
CREATE TABLE events (
    -- ... columns as above ...
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2024 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
CREATE TABLE events_2025 PARTITION OF events
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
CREATE TABLE events_2026 PARTITION OF events
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');
```

### Migration from Legacy Systems

Migrating to event sourcing from existing systems requires a different approach than traditional data migration:

1. **Import historical data as "backdated" events**: For each existing contract, generate a sequence of synthetic events (ContractDrafted, ContractExecuted, AdvanceScheduled, etc.) with timestamps matching the original dates. This preserves the event-sourced invariant that all state derives from events.
2. **Bulk-import sales history**: Generate SalesDataImported and SalesLineRecorded events for each historical period. Mark these with metadata indicating they are synthetic imports.
3. **Snapshot after import**: Take snapshots of all aggregates immediately after historical import to avoid expensive replays of synthetic events.
4. **Parallel-run period**: Run the new system alongside the legacy system for at least two royalty periods, comparing statement outputs to verify correctness before cutover.

### Scaling Strategy

1. **Phase 1**: Single PostgreSQL instance for both event store and read models. Adequate for up to 50 publishers with typical catalog sizes.
2. **Phase 2**: Separate read replicas for projections. Kafka for event distribution to external systems. Introduce snapshotting for high-volume aggregates.
3. **Phase 3**: Dedicated event store database separated from read model databases. Shard event store by aggregate type if write throughput demands it. Multiple projection instances for horizontal read scaling.
4. **Archival**: Events are never deleted, but old partitions can be moved to cheaper storage (S3-backed). Read models for historical periods can be materialized and frozen.
