# Data Model Suggestion 1: Normalized Relational Database (PostgreSQL)

## Overview

This model uses a fully normalized relational schema in PostgreSQL, designed around the distinct bounded contexts of the Book Publishing Platform: title/product management, editorial workflow, contracts and rights, royalty accounting, and sales data ingestion. Every entity is decomposed into third normal form (3NF) with explicit foreign key constraints, enabling referential integrity across the entire publishing lifecycle.

The normalized approach prioritizes data consistency, auditability, and the complex multi-table joins required by royalty calculations that span contracts, sales, returns, rates, and payment histories.

---

## Core Design Principles

1. **Work vs. Product separation**: A "work" (intellectual property) is distinct from a "product" (a specific format/edition with its own ISBN). One work can have many products. Royalties may be calculated at either level.
2. **Temporal validity**: Contract terms, royalty rates, rights grants, and exchange rates all carry `valid_from` / `valid_until` timestamps, supporting historical queries and preventing the loss of period data when terms change.
3. **Immutable financial records**: Royalty statements, payment records, and sales transactions are append-only. Corrections are recorded as adjustment rows, never as updates to existing records.
4. **ONIX alignment**: Product metadata tables mirror ONIX 3.0 composite structure (Descriptive Detail, Publishing Detail, Product Supply) to simplify export generation.
5. **Multi-currency by design**: All monetary amounts store both the original currency/amount and the reporting currency/amount with the exchange rate used.

---

## Schema Definition

### 1. Organization and People

```sql
-- Publishers, imprints, and organizational units
CREATE TABLE organizations (
    organization_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_type   VARCHAR(50) NOT NULL CHECK (organization_type IN (
        'publisher', 'imprint', 'distributor', 'retailer', 'agency', 'foreign_publisher'
    )),
    name                VARCHAR(500) NOT NULL,
    parent_org_id       UUID REFERENCES organizations(organization_id),
    country_code        CHAR(2),           -- ISO 3166-1 alpha-2
    default_currency    CHAR(3),           -- ISO 4217
    tax_id              VARCHAR(100),
    website             VARCHAR(500),
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_organizations_type ON organizations(organization_type);
CREATE INDEX idx_organizations_parent ON organizations(parent_org_id);

-- People: authors, editors, agents, illustrators, translators
CREATE TABLE persons (
    person_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    first_name          VARCHAR(200),
    last_name           VARCHAR(200),
    display_name        VARCHAR(500) NOT NULL,  -- The name used on covers / credits
    sort_name           VARCHAR(500),           -- Last, First for catalog ordering
    biography           TEXT,
    nationality         CHAR(2),                -- ISO 3166-1
    date_of_birth       DATE,
    date_of_death       DATE,
    email               VARCHAR(300),
    phone               VARCHAR(50),
    tax_id              VARCHAR(100),           -- For royalty payments
    payment_currency    CHAR(3) DEFAULT 'USD',  -- Preferred payment currency
    bank_account_info   TEXT,                   -- Encrypted in practice
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_persons_display_name ON persons(display_name);
CREATE INDEX idx_persons_sort_name ON persons(sort_name);

-- Person roles (a person can be author, agent, editor, etc.)
CREATE TABLE person_roles (
    person_role_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id           UUID NOT NULL REFERENCES persons(person_id),
    role_type           VARCHAR(50) NOT NULL CHECK (role_type IN (
        'author', 'co_author', 'editor', 'translator', 'illustrator',
        'foreword_by', 'introduction_by', 'narrator', 'agent', 'staff_editor',
        'copy_editor', 'proofreader', 'designer', 'photographer'
    )),
    organization_id     UUID REFERENCES organizations(organization_id),  -- e.g., which agency they belong to
    is_primary          BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_person_roles_person ON person_roles(person_id);
CREATE INDEX idx_person_roles_type ON person_roles(role_type);
```

### 2. Works and Products (Title & ISBN Management)

```sql
-- A "work" is the intellectual property, independent of format
CREATE TABLE works (
    work_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title               VARCHAR(1000) NOT NULL,
    subtitle            VARCHAR(1000),
    original_language   CHAR(3),                -- ISO 639-2/B (e.g., 'eng')
    work_type           VARCHAR(50) NOT NULL CHECK (work_type IN (
        'monograph', 'anthology', 'serial', 'textbook', 'reference',
        'graphic_novel', 'poetry', 'drama', 'children', 'young_adult'
    )),
    edition_number      INTEGER DEFAULT 1,
    edition_statement   VARCHAR(200),
    series_id           UUID REFERENCES series(series_id),
    series_position     DECIMAL(5,1),           -- e.g., 1.5 for interstitial entries
    synopsis            TEXT,
    internal_notes      TEXT,
    acquisition_date    DATE,
    publication_status  VARCHAR(50) NOT NULL DEFAULT 'in_development' CHECK (publication_status IN (
        'proposed', 'in_development', 'in_production', 'published',
        'out_of_print', 'abandoned', 'rights_reverted'
    )),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_works_title ON works USING gin(to_tsvector('english', title));
CREATE INDEX idx_works_status ON works(publication_status);

-- Series grouping
CREATE TABLE series (
    series_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    series_name         VARCHAR(500) NOT NULL,
    issn                VARCHAR(20),            -- International Standard Serial Number
    description         TEXT,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Work-to-person relationships (contributors)
CREATE TABLE work_contributors (
    work_contributor_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_id             UUID NOT NULL REFERENCES works(work_id),
    person_id           UUID NOT NULL REFERENCES persons(person_id),
    contributor_role    VARCHAR(50) NOT NULL CHECK (contributor_role IN (
        'author', 'co_author', 'editor', 'translator', 'illustrator',
        'foreword_by', 'introduction_by', 'narrator', 'photographer',
        'compiled_by', 'adapted_by'
    )),
    sequence_number     INTEGER NOT NULL DEFAULT 1,   -- Display order
    contribution_pct    DECIMAL(5,2),                  -- Revenue split percentage
    UNIQUE(work_id, person_id, contributor_role)
);

CREATE INDEX idx_work_contributors_work ON work_contributors(work_id);
CREATE INDEX idx_work_contributors_person ON work_contributors(person_id);

-- Products: specific editions/formats of a work, each with its own ISBN
CREATE TABLE products (
    product_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_id             UUID NOT NULL REFERENCES works(work_id),
    isbn_13             CHAR(13) UNIQUE,
    isbn_10             CHAR(10),
    ean                 VARCHAR(20),
    product_form        VARCHAR(50) NOT NULL CHECK (product_form IN (
        'hardback', 'trade_paperback', 'mass_market_paperback',
        'ebook_epub', 'ebook_pdf', 'ebook_kindle',
        'audiobook_download', 'audiobook_cd',
        'large_print', 'board_book', 'spiral_bound'
    )),
    list_price          DECIMAL(10,2),
    price_currency      CHAR(3) DEFAULT 'USD',
    page_count          INTEGER,
    word_count          INTEGER,
    trim_width_mm       DECIMAL(6,1),
    trim_height_mm      DECIMAL(6,1),
    spine_width_mm      DECIMAL(6,1),
    weight_grams        INTEGER,
    duration_minutes    INTEGER,                -- For audiobooks
    publication_date    DATE,
    on_sale_date        DATE,
    out_of_print_date   DATE,
    print_run_size      INTEGER,
    publisher_id        UUID NOT NULL REFERENCES organizations(organization_id),
    imprint_id          UUID REFERENCES organizations(organization_id),
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_products_work ON products(work_id);
CREATE INDEX idx_products_isbn ON products(isbn_13);
CREATE INDEX idx_products_form ON products(product_form);
CREATE INDEX idx_products_pub_date ON products(publication_date);

-- ISBN prefix management
CREATE TABLE isbn_prefixes (
    prefix_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(organization_id),
    prefix              VARCHAR(20) NOT NULL,      -- e.g., '978-1-234567'
    total_isbns         INTEGER NOT NULL,
    assigned_count      INTEGER NOT NULL DEFAULT 0,
    agency              VARCHAR(100),              -- e.g., 'Bowker', 'Nielsen'
    registration_date   DATE,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Subject classifications (BISAC and THEMA)
CREATE TABLE product_subjects (
    product_subject_id  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id          UUID NOT NULL REFERENCES products(product_id),
    scheme              VARCHAR(20) NOT NULL CHECK (scheme IN ('BISAC', 'THEMA', 'BIC', 'custom')),
    code                VARCHAR(50) NOT NULL,
    heading_text        VARCHAR(500),
    is_primary          BOOLEAN NOT NULL DEFAULT FALSE,
    UNIQUE(product_id, scheme, code)
);

CREATE INDEX idx_product_subjects_product ON product_subjects(product_id);
CREATE INDEX idx_product_subjects_code ON product_subjects(scheme, code);
```

### 3. Editorial Workflow

```sql
-- Configurable editorial pipeline templates
CREATE TABLE pipeline_templates (
    template_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_name       VARCHAR(200) NOT NULL,
    description         TEXT,
    is_default          BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE pipeline_stages (
    stage_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id         UUID NOT NULL REFERENCES pipeline_templates(template_id),
    stage_name          VARCHAR(200) NOT NULL,
    stage_order         INTEGER NOT NULL,
    default_duration_days INTEGER,
    is_milestone        BOOLEAN NOT NULL DEFAULT FALSE,  -- Triggers contract payments
    description         TEXT,
    UNIQUE(template_id, stage_order)
);

-- Work progress through editorial pipeline
CREATE TABLE work_pipeline (
    work_pipeline_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_id             UUID NOT NULL REFERENCES works(work_id),
    template_id         UUID NOT NULL REFERENCES pipeline_templates(template_id),
    current_stage_id    UUID REFERENCES pipeline_stages(stage_id),
    started_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at        TIMESTAMPTZ
);

CREATE TABLE work_stage_history (
    history_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_pipeline_id    UUID NOT NULL REFERENCES work_pipeline(work_pipeline_id),
    stage_id            UUID NOT NULL REFERENCES pipeline_stages(stage_id),
    assigned_to         UUID REFERENCES persons(person_id),
    status              VARCHAR(30) NOT NULL CHECK (status IN (
        'pending', 'in_progress', 'review', 'approved', 'rejected', 'skipped'
    )),
    deadline            DATE,
    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_work_stage_history_pipeline ON work_stage_history(work_pipeline_id);

-- Manuscript submissions (slush pile)
CREATE TABLE submissions (
    submission_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title               VARCHAR(1000) NOT NULL,
    author_name         VARCHAR(500) NOT NULL,
    author_email        VARCHAR(300),
    agent_name          VARCHAR(500),
    agent_email         VARCHAR(300),
    genre               VARCHAR(200),
    word_count          INTEGER,
    synopsis            TEXT,
    cover_letter        TEXT,
    manuscript_file_url VARCHAR(1000),          -- S3 or object storage reference
    submission_status   VARCHAR(50) NOT NULL DEFAULT 'received' CHECK (submission_status IN (
        'received', 'acknowledged', 'under_review', 'requested_full',
        'accepted', 'rejected', 'withdrawn'
    )),
    ai_quality_score    DECIMAL(5,2),           -- AI triage score (0-100)
    ai_genre_tags       TEXT[],                 -- AI-detected genre tags
    reviewer_id         UUID REFERENCES persons(person_id),
    reviewer_notes      TEXT,
    work_id             UUID REFERENCES works(work_id),  -- Link if accepted
    submitted_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    acknowledged_at     TIMESTAMPTZ,
    reviewed_at         TIMESTAMPTZ,
    decided_at          TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_submissions_status ON submissions(submission_status);
CREATE INDEX idx_submissions_date ON submissions(submitted_at);

-- Editorial tasks and assignments
CREATE TABLE editorial_tasks (
    task_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_id             UUID NOT NULL REFERENCES works(work_id),
    stage_id            UUID REFERENCES pipeline_stages(stage_id),
    task_type           VARCHAR(100) NOT NULL,
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    assigned_to         UUID REFERENCES persons(person_id),
    priority            VARCHAR(20) DEFAULT 'normal' CHECK (priority IN (
        'low', 'normal', 'high', 'urgent'
    )),
    status              VARCHAR(30) NOT NULL DEFAULT 'open' CHECK (status IN (
        'open', 'in_progress', 'blocked', 'complete', 'cancelled'
    )),
    due_date            DATE,
    completed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_editorial_tasks_work ON editorial_tasks(work_id);
CREATE INDEX idx_editorial_tasks_assigned ON editorial_tasks(assigned_to);
CREATE INDEX idx_editorial_tasks_status ON editorial_tasks(status);
```

### 4. Contracts and Rights

```sql
-- Master contract record
CREATE TABLE contracts (
    contract_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_number     VARCHAR(100) UNIQUE NOT NULL,
    work_id             UUID NOT NULL REFERENCES works(work_id),
    contract_type       VARCHAR(50) NOT NULL CHECK (contract_type IN (
        'author_agreement', 'co_publishing', 'distribution',
        'sub_license_out', 'sub_license_in', 'work_for_hire',
        'translation_license', 'audio_license', 'film_option'
    )),
    counterparty_person_id  UUID REFERENCES persons(person_id),
    counterparty_org_id     UUID REFERENCES organizations(organization_id),
    agent_person_id     UUID REFERENCES persons(person_id),
    agent_commission_pct DECIMAL(5,2),         -- Agent's percentage of author earnings
    execution_date      DATE,
    effective_date      DATE NOT NULL,
    expiration_date     DATE,
    termination_date    DATE,
    termination_reason  TEXT,
    contract_currency   CHAR(3) NOT NULL DEFAULT 'USD',
    status              VARCHAR(30) NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'negotiation', 'executed', 'active', 'expired',
        'terminated', 'reverted'
    )),
    document_url        VARCHAR(1000),         -- Signed contract scan / PDF
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_contracts_work ON contracts(work_id);
CREATE INDEX idx_contracts_status ON contracts(status);
CREATE INDEX idx_contracts_expiration ON contracts(expiration_date);

-- Advance payment schedule (tranches)
CREATE TABLE contract_advances (
    advance_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id         UUID NOT NULL REFERENCES contracts(contract_id),
    tranche_number      INTEGER NOT NULL,
    tranche_description VARCHAR(200),          -- e.g., 'On signing', 'On delivery', 'On publication'
    trigger_type        VARCHAR(50) CHECK (trigger_type IN (
        'on_signing', 'on_delivery', 'on_acceptance',
        'on_publication', 'on_paperback', 'on_date', 'on_milestone'
    )),
    trigger_stage_id    UUID REFERENCES pipeline_stages(stage_id),
    trigger_date        DATE,
    amount              DECIMAL(12,2) NOT NULL,
    currency            CHAR(3) NOT NULL DEFAULT 'USD',
    status              VARCHAR(30) NOT NULL DEFAULT 'scheduled' CHECK (status IN (
        'scheduled', 'due', 'paid', 'cancelled'
    )),
    paid_date           DATE,
    payment_reference   VARCHAR(200),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(contract_id, tranche_number)
);

CREATE INDEX idx_contract_advances_contract ON contract_advances(contract_id);
CREATE INDEX idx_contract_advances_status ON contract_advances(status);

-- Royalty rate schedules (with escalators)
CREATE TABLE royalty_rate_schedules (
    rate_schedule_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id         UUID NOT NULL REFERENCES contracts(contract_id),
    product_form        VARCHAR(50),           -- NULL means all forms under this contract
    sales_channel       VARCHAR(100),          -- NULL means all channels
    territory_code      VARCHAR(10),           -- NULL means all territories in grant
    rate_basis          VARCHAR(30) NOT NULL CHECK (rate_basis IN (
        'list_price', 'net_receipts', 'net_revenue', 'retail_price'
    )),
    valid_from          DATE NOT NULL,
    valid_until         DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_royalty_rates_contract ON royalty_rate_schedules(contract_id);

-- Escalating rate tiers within a schedule
CREATE TABLE royalty_rate_tiers (
    tier_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rate_schedule_id    UUID NOT NULL REFERENCES royalty_rate_schedules(rate_schedule_id),
    tier_floor          INTEGER NOT NULL DEFAULT 0,    -- Units sold threshold (cumulative)
    tier_ceiling        INTEGER,                        -- NULL means unlimited
    royalty_pct         DECIMAL(6,3) NOT NULL,          -- e.g., 10.000, 12.500, 15.000
    UNIQUE(rate_schedule_id, tier_floor)
);

CREATE INDEX idx_royalty_rate_tiers_schedule ON royalty_rate_tiers(rate_schedule_id);

-- Territory and language rights grants
CREATE TABLE rights_grants (
    grant_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id         UUID NOT NULL REFERENCES contracts(contract_id),
    right_type          VARCHAR(50) NOT NULL CHECK (right_type IN (
        'print_hardback', 'print_paperback', 'ebook', 'audiobook',
        'translation', 'film_tv', 'dramatic', 'merchandising',
        'serial_first', 'serial_second', 'book_club',
        'large_print', 'braille', 'anthology', 'digital_audio',
        'all_print', 'all_digital', 'all_rights', 'world_rights'
    )),
    territory_type      VARCHAR(20) NOT NULL CHECK (territory_type IN (
        'world', 'world_english', 'country_list', 'region_list', 'exclusion_list'
    )),
    language_code       CHAR(3),               -- ISO 639-2/B; NULL means all languages
    is_exclusive        BOOLEAN NOT NULL DEFAULT TRUE,
    grant_start         DATE NOT NULL,
    grant_end           DATE,
    reversion_clause    TEXT,
    reversion_trigger   VARCHAR(200),          -- e.g., 'Out of print > 12 months'
    status              VARCHAR(30) NOT NULL DEFAULT 'active' CHECK (status IN (
        'active', 'expired', 'reverted', 'sub_licensed'
    )),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rights_grants_contract ON rights_grants(contract_id);
CREATE INDEX idx_rights_grants_status ON rights_grants(status);
CREATE INDEX idx_rights_grants_expiry ON rights_grants(grant_end);

-- Territories associated with a rights grant
CREATE TABLE rights_grant_territories (
    grant_territory_id  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    grant_id            UUID NOT NULL REFERENCES rights_grants(grant_id),
    country_code        CHAR(2) NOT NULL,      -- ISO 3166-1 alpha-2
    is_included         BOOLEAN NOT NULL DEFAULT TRUE,  -- FALSE for exclusion lists
    UNIQUE(grant_id, country_code)
);

CREATE INDEX idx_rgt_grant ON rights_grant_territories(grant_id);

-- Sub-licensing deals (rights-out)
CREATE TABLE sub_licenses (
    sub_license_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_grant_id     UUID NOT NULL REFERENCES rights_grants(grant_id),
    licensee_org_id     UUID NOT NULL REFERENCES organizations(organization_id),
    contract_id         UUID NOT NULL REFERENCES contracts(contract_id),  -- The sub-license contract
    advance_amount      DECIMAL(12,2),
    advance_currency    CHAR(3),
    royalty_split_pct   DECIMAL(5,2),          -- Publisher's share of sub-license revenue
    author_split_pct    DECIMAL(5,2),          -- Author's share of sub-license revenue
    effective_date      DATE NOT NULL,
    expiration_date     DATE,
    status              VARCHAR(30) NOT NULL DEFAULT 'active',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sub_licenses_grant ON sub_licenses(parent_grant_id);
```

### 5. Royalty Calculation Engine

```sql
-- Royalty accounting periods
CREATE TABLE royalty_periods (
    period_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    period_name         VARCHAR(100) NOT NULL,     -- e.g., 'H1 2026', 'Q3 2025'
    period_start        DATE NOT NULL,
    period_end          DATE NOT NULL,
    period_type         VARCHAR(20) NOT NULL CHECK (period_type IN (
        'monthly', 'quarterly', 'semi_annual', 'annual'
    )),
    status              VARCHAR(30) NOT NULL DEFAULT 'open' CHECK (status IN (
        'open', 'closed', 'calculating', 'reviewed', 'finalized', 'paid'
    )),
    finalized_at        TIMESTAMPTZ,
    finalized_by        UUID REFERENCES persons(person_id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(period_start, period_end)
);

-- Sales data imported from distributors
CREATE TABLE sales_transactions (
    transaction_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id          UUID NOT NULL REFERENCES products(product_id),
    period_id           UUID NOT NULL REFERENCES royalty_periods(period_id),
    distributor_id      UUID NOT NULL REFERENCES organizations(organization_id),
    sales_channel       VARCHAR(100),              -- e.g., 'Amazon', 'Barnes & Noble', 'Ingram wholesale'
    territory_code      CHAR(2),                   -- Country of sale
    transaction_type    VARCHAR(30) NOT NULL CHECK (transaction_type IN (
        'sale', 'return', 'adjustment', 'free_copy', 'damaged', 'remainder'
    )),
    quantity            INTEGER NOT NULL,
    unit_price          DECIMAL(10,2),             -- Price per unit in sale currency
    sale_currency       CHAR(3) NOT NULL,
    gross_amount        DECIMAL(12,2) NOT NULL,    -- quantity * unit_price in sale currency
    discount_pct        DECIMAL(5,2),
    net_amount          DECIMAL(12,2) NOT NULL,    -- After discounts in sale currency
    reporting_amount    DECIMAL(12,2),             -- Converted to reporting currency
    reporting_currency  CHAR(3),
    exchange_rate       DECIMAL(12,6),
    import_batch_id     UUID NOT NULL REFERENCES sales_import_batches(batch_id),
    sale_date           DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sales_product ON sales_transactions(product_id);
CREATE INDEX idx_sales_period ON sales_transactions(period_id);
CREATE INDEX idx_sales_distributor ON sales_transactions(distributor_id);
CREATE INDEX idx_sales_batch ON sales_transactions(import_batch_id);

-- Sales import batch tracking (ETL provenance)
CREATE TABLE sales_import_batches (
    batch_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    distributor_id      UUID NOT NULL REFERENCES organizations(organization_id),
    file_name           VARCHAR(500),
    file_format         VARCHAR(50),               -- 'ingram_csv', 'amazon_kdp', 'baker_taylor', etc.
    period_id           UUID REFERENCES royalty_periods(period_id),
    row_count           INTEGER,
    total_units         INTEGER,
    total_gross         DECIMAL(14,2),
    status              VARCHAR(30) NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'validating', 'validated', 'imported', 'failed', 'rejected'
    )),
    error_log           TEXT,
    imported_at         TIMESTAMPTZ,
    imported_by         UUID REFERENCES persons(person_id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Royalty calculation results per contract per period
CREATE TABLE royalty_calculations (
    calculation_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id         UUID NOT NULL REFERENCES contracts(contract_id),
    period_id           UUID NOT NULL REFERENCES royalty_periods(period_id),
    product_id          UUID REFERENCES products(product_id),    -- NULL if at work level
    total_units_sold    INTEGER NOT NULL DEFAULT 0,
    total_units_returned INTEGER NOT NULL DEFAULT 0,
    net_units           INTEGER NOT NULL DEFAULT 0,
    cumulative_units    INTEGER NOT NULL DEFAULT 0,              -- For escalator threshold
    gross_revenue       DECIMAL(14,2) NOT NULL DEFAULT 0,
    net_revenue         DECIMAL(14,2) NOT NULL DEFAULT 0,
    royalty_basis_amount DECIMAL(14,2) NOT NULL DEFAULT 0,       -- The amount rates apply to
    calculated_royalty  DECIMAL(14,2) NOT NULL DEFAULT 0,        -- Before reserves/recoupment
    reserve_held        DECIMAL(14,2) NOT NULL DEFAULT 0,        -- Reserve against returns
    reserve_released    DECIMAL(14,2) NOT NULL DEFAULT 0,        -- From prior period reserves
    advance_recoupment  DECIMAL(14,2) NOT NULL DEFAULT 0,        -- Applied against unearned advance
    agent_commission    DECIMAL(14,2) NOT NULL DEFAULT 0,
    net_payable         DECIMAL(14,2) NOT NULL DEFAULT 0,        -- Final amount due
    calculation_currency CHAR(3) NOT NULL DEFAULT 'USD',
    calculated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    calculated_by       VARCHAR(100) DEFAULT 'system',
    is_override         BOOLEAN NOT NULL DEFAULT FALSE,
    override_reason     TEXT,
    UNIQUE(contract_id, period_id, product_id)
);

CREATE INDEX idx_royalty_calc_contract ON royalty_calculations(contract_id);
CREATE INDEX idx_royalty_calc_period ON royalty_calculations(period_id);

-- Reserve against returns tracking
CREATE TABLE reserves_against_returns (
    reserve_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id         UUID NOT NULL REFERENCES contracts(contract_id),
    period_id           UUID NOT NULL REFERENCES royalty_periods(period_id),
    reserve_pct         DECIMAL(5,2) NOT NULL,         -- Configured reserve percentage
    amount_held         DECIMAL(12,2) NOT NULL,
    amount_released     DECIMAL(12,2) NOT NULL DEFAULT 0,
    release_period_id   UUID REFERENCES royalty_periods(period_id),  -- When released
    status              VARCHAR(30) NOT NULL DEFAULT 'held' CHECK (status IN (
        'held', 'partially_released', 'fully_released'
    )),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Advance balance tracking (running unearned balance)
CREATE TABLE advance_balances (
    balance_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id         UUID NOT NULL REFERENCES contracts(contract_id),
    period_id           UUID NOT NULL REFERENCES royalty_periods(period_id),
    opening_balance     DECIMAL(12,2) NOT NULL,        -- Unearned advance at period start
    advance_paid        DECIMAL(12,2) NOT NULL DEFAULT 0,  -- New advances in period
    royalties_earned    DECIMAL(12,2) NOT NULL DEFAULT 0,  -- Earned against advance
    closing_balance     DECIMAL(12,2) NOT NULL,        -- Remaining unearned
    is_earned_out       BOOLEAN NOT NULL DEFAULT FALSE, -- TRUE when advance fully recouped
    earned_out_date     DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(contract_id, period_id)
);

CREATE INDEX idx_advance_balances_contract ON advance_balances(contract_id);

-- Royalty statements (author-facing documents)
CREATE TABLE royalty_statements (
    statement_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id         UUID NOT NULL REFERENCES contracts(contract_id),
    period_id           UUID NOT NULL REFERENCES royalty_periods(period_id),
    person_id           UUID NOT NULL REFERENCES persons(person_id),   -- The payee
    statement_number    VARCHAR(50) UNIQUE NOT NULL,
    statement_date      DATE NOT NULL,
    total_earned        DECIMAL(14,2) NOT NULL,
    total_reserves      DECIMAL(14,2) NOT NULL,
    total_recoupment    DECIMAL(14,2) NOT NULL,
    total_agent_comm    DECIMAL(14,2) NOT NULL DEFAULT 0,
    net_payable         DECIMAL(14,2) NOT NULL,
    payment_currency    CHAR(3) NOT NULL,
    minimum_payment_threshold DECIMAL(10,2) DEFAULT 25.00,
    is_below_threshold  BOOLEAN NOT NULL DEFAULT FALSE,
    status              VARCHAR(30) NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'reviewed', 'approved', 'sent', 'paid'
    )),
    sent_at             TIMESTAMPTZ,
    paid_at             TIMESTAMPTZ,
    payment_reference   VARCHAR(200),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_statements_contract ON royalty_statements(contract_id);
CREATE INDEX idx_statements_person ON royalty_statements(person_id);
CREATE INDEX idx_statements_period ON royalty_statements(period_id);

-- Statement line items (detailed breakdown)
CREATE TABLE royalty_statement_lines (
    line_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    statement_id        UUID NOT NULL REFERENCES royalty_statements(statement_id),
    product_id          UUID NOT NULL REFERENCES products(product_id),
    line_type           VARCHAR(30) NOT NULL CHECK (line_type IN (
        'sales', 'returns', 'reserve_held', 'reserve_released',
        'sub_license_income', 'advance_recoupment', 'adjustment',
        'agent_commission'
    )),
    description         VARCHAR(500),
    units               INTEGER,
    unit_price          DECIMAL(10,2),
    royalty_rate_pct    DECIMAL(6,3),
    amount              DECIMAL(12,2) NOT NULL,
    currency            CHAR(3) NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_statement_lines_stmt ON royalty_statement_lines(statement_id);
```

### 6. Exchange Rates

```sql
CREATE TABLE exchange_rates (
    rate_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    from_currency       CHAR(3) NOT NULL,
    to_currency         CHAR(3) NOT NULL,
    rate                DECIMAL(14,8) NOT NULL,
    rate_date           DATE NOT NULL,
    rate_type           VARCHAR(20) NOT NULL CHECK (rate_type IN (
        'daily', 'period_average', 'period_end', 'contractual'
    )),
    source              VARCHAR(100),          -- e.g., 'ECB', 'manual', 'API'
    UNIQUE(from_currency, to_currency, rate_date, rate_type)
);

CREATE INDEX idx_exchange_rates_pair ON exchange_rates(from_currency, to_currency, rate_date);
```

### 7. ONIX Export Support

```sql
-- ONIX export jobs and history
CREATE TABLE onix_exports (
    export_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    export_name         VARCHAR(200),
    recipient_org_id    UUID REFERENCES organizations(organization_id),
    onix_version        VARCHAR(10) NOT NULL DEFAULT '3.0',
    product_count       INTEGER,
    file_url            VARCHAR(1000),
    status              VARCHAR(30) NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'generating', 'completed', 'failed', 'sent'
    )),
    generated_at        TIMESTAMPTZ,
    sent_at             TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Product-level ONIX metadata overrides (beyond standard product fields)
CREATE TABLE product_onix_metadata (
    metadata_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id          UUID NOT NULL REFERENCES products(product_id),
    notification_type   VARCHAR(10) DEFAULT '03',  -- ONIX notification type code
    product_composition VARCHAR(10),               -- ONIX code
    product_form_code   VARCHAR(10),               -- ONIX product form codelist value
    product_form_detail VARCHAR(10),
    country_of_manufacture CHAR(2),
    text_language       CHAR(3),
    audience_code       VARCHAR(10),
    edition_type        VARCHAR(10),
    UNIQUE(product_id)
);
```

### 8. Author Portal and Audit

```sql
-- Author portal user accounts
CREATE TABLE portal_users (
    portal_user_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id           UUID NOT NULL REFERENCES persons(person_id),
    email               VARCHAR(300) NOT NULL UNIQUE,
    password_hash       VARCHAR(500) NOT NULL,
    last_login          TIMESTAMPTZ,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Comprehensive audit log
CREATE TABLE audit_log (
    audit_id            BIGSERIAL PRIMARY KEY,
    table_name          VARCHAR(100) NOT NULL,
    record_id           UUID NOT NULL,
    action              VARCHAR(20) NOT NULL CHECK (action IN ('INSERT', 'UPDATE', 'DELETE')),
    old_values          JSONB,
    new_values          JSONB,
    changed_by          UUID,
    changed_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    ip_address          INET,
    user_agent          VARCHAR(500)
);

CREATE INDEX idx_audit_log_table ON audit_log(table_name, record_id);
CREATE INDEX idx_audit_log_time ON audit_log(changed_at);
```

---

## Key Relationships Diagram (Conceptual)

```
Organization (publisher/imprint/distributor)
    |
    +-- ISBN Prefixes
    |
    +-- Contracts ---- Work ---- Series
    |       |            |
    |       |            +-- Work Contributors -- Person
    |       |            |
    |       |            +-- Products (ISBN per format)
    |       |                    |
    |       |                    +-- Product Subjects (BISAC/THEMA)
    |       |                    +-- Sales Transactions
    |       |                    +-- ONIX Metadata
    |       |
    |       +-- Contract Advances (payment tranches)
    |       +-- Royalty Rate Schedules -- Rate Tiers (escalators)
    |       +-- Rights Grants -- Territories
    |       |       +-- Sub-Licenses
    |       |
    |       +-- Royalty Calculations (per period)
    |       +-- Advance Balances (per period)
    |       +-- Royalty Statements -- Statement Lines
    |
    +-- Sales Import Batches

Work Pipeline -- Pipeline Template -- Pipeline Stages
    +-- Work Stage History

Submissions (slush pile, may link to Work if accepted)
```

---

## Pros and Cons

### Pros

1. **Referential integrity guarantees**: Foreign key constraints ensure that every royalty calculation references a valid contract, product, and period. Orphaned records are impossible.
2. **Complex multi-table joins for royalty calculation**: The normalized structure makes it straightforward to write SQL that joins contracts to rate schedules to sales transactions to compute royalties with escalating thresholds -- the core business logic.
3. **Temporal accuracy**: `valid_from`/`valid_until` on rate schedules and rights grants ensures that historical royalty calculations always use the rates that were in effect during that period, even after renegotiation.
4. **Standard tooling**: PostgreSQL has mature tooling for backups, replication, monitoring, and migration. ORMs like SQLAlchemy, Prisma, and TypeORM all support it well.
5. **ACID compliance**: Financial calculations (advance recoupment, reserve accounting) require transactional consistency that PostgreSQL provides natively.
6. **Audit trail**: The separate audit_log table with JSONB old/new values captures every change without polluting business tables.
7. **Query flexibility**: Ad hoc reporting, analytics, and the complex WHERE clauses needed for territory-specific rights lookups are natural in SQL.
8. **Well-understood scaling**: Read replicas, connection pooling (PgBouncer), and table partitioning (e.g., sales_transactions by period) are all proven PostgreSQL capabilities.

### Cons

1. **Schema rigidity**: Adding a new product form, a new rights type, or a new metadata field for a specific format requires a migration. For a domain where publishing conventions evolve (e.g., new digital formats, new distribution channels), this creates ongoing maintenance.
2. **ONIX mapping complexity**: ONIX 3.0 is deeply nested XML with repeating composites. Mapping it to flat normalized tables requires significant application-layer transformation logic. Some ONIX elements (like multiple price conditions with different validity dates per territory) are awkward to normalize fully.
3. **Performance at scale for royalty runs**: A semi-annual royalty calculation touching millions of sales rows across hundreds of contracts with multi-tier escalators and cross-product aggregation requires careful query optimization, potentially materialized views or pre-aggregation.
4. **Many joins for common operations**: Displaying a single title's full details (work + products + contributors + contracts + rights + subjects) requires joining 8+ tables. This creates complex queries and potential performance concerns.
5. **Rigid contributor model**: The publishing world has edge cases (pseudonyms, corporate authors, estates, anonymous works) that the normalized person/contributor model handles awkwardly.
6. **Migration burden**: As the platform evolves over years (the README notes titles have decade-long lifespans), the accumulated migration history becomes a maintenance concern.

---

## Technology Recommendations

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| **Database** | PostgreSQL 16+ | Native UUID generation, JSONB for audit log, excellent full-text search, table partitioning |
| **Connection pooling** | PgBouncer | Essential for multi-tenant SaaS; handles connection limits gracefully |
| **Migrations** | Flyway or Alembic | Version-controlled SQL migrations; Flyway for Java stacks, Alembic for Python |
| **ORM** | SQLAlchemy (Python) or Prisma (Node.js) | Both handle complex joins well; SQLAlchemy's explicit mapping suits this domain |
| **Search** | PostgreSQL full-text search (GIN indexes) | Sufficient for title/author search; avoid adding Elasticsearch unless catalog exceeds 500K titles |
| **Reporting** | Materialized views + pg_cron | Pre-aggregate royalty summaries for dashboard performance |
| **Backup** | pg_basebackup + WAL archiving | Point-in-time recovery is essential for financial data |
| **Monitoring** | pg_stat_statements + Prometheus | Track slow queries, especially during royalty calculation runs |

---

## Migration and Scaling Considerations

### Partitioning Strategy

```sql
-- Partition sales_transactions by period for query performance
CREATE TABLE sales_transactions (
    -- ... columns as above ...
) PARTITION BY RANGE (sale_date);

-- Create partitions per year
CREATE TABLE sales_transactions_2024 PARTITION OF sales_transactions
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
CREATE TABLE sales_transactions_2025 PARTITION OF sales_transactions
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
CREATE TABLE sales_transactions_2026 PARTITION OF sales_transactions
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');

-- Partition audit_log by month (high write volume)
CREATE TABLE audit_log (
    -- ... columns as above ...
) PARTITION BY RANGE (changed_at);
```

### Scaling Path

1. **Phase 1 (0-100 publishers)**: Single PostgreSQL instance with read replica. PgBouncer for connection management. Adequate for millions of sales rows per period.
2. **Phase 2 (100-1000 publishers)**: Row-level security for multi-tenant isolation. Partition sales and audit tables. Add materialized views for royalty dashboards.
3. **Phase 3 (1000+ publishers)**: Schema-per-tenant or separate databases per large publisher. Consider Citus for distributed PostgreSQL if single-node capacity is exceeded.
4. **Archival**: Sales data older than 7 years can be moved to cold storage (S3 + Parquet) while keeping summary records in the live database. Rights and contract data must be retained indefinitely due to legal requirements.

### Data Migration from Legacy Systems

Publishers adopting this platform will need to migrate from spreadsheets, MetaComet, Consonance, or custom systems. Key considerations:

- **ISBN mapping**: Validate all ISBNs against check-digit algorithm on import; flag duplicates.
- **Historical royalty data**: Import as pre-calculated statement records rather than attempting to re-derive from raw sales. Store legacy statement PDFs as document references.
- **Contract back-dating**: Allow `effective_date` to be in the past. Populate `advance_balances` with the current unearned balance as an opening entry.
- **Rights reconstruction**: Require manual review of rights grants during migration; automated extraction from contract PDFs via AI can assist but should not be trusted without human verification.
