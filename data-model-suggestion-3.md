# Data Model Suggestion 3: Hybrid Relational + Document (PostgreSQL with JSONB)

## Overview

This model takes a pragmatic middle ground: use normalized relational tables for entities with stable structure and referential integrity requirements (contracts, royalty calculations, financial records, rights grants), and JSONB columns for data that is inherently variable, semi-structured, or schema-volatile (product metadata, ONIX attributes, distributor-specific sales data formats, editorial workflow configurations).

The core insight for book publishing is that **financial and legal data** (contracts, royalty rates, advance balances, rights territories) is highly structured and must enforce strict constraints, while **product metadata** (book descriptions, format-specific attributes, contributor details, subject classifications, marketing collateral) varies enormously between formats, genres, and publishing traditions. A hardback novel, an audiobook, and an interactive children's ebook have fundamentally different metadata profiles. Forcing all of these into rigid normalized tables either creates sparse columns or an explosion of EAV-pattern lookup tables. JSONB handles this variability natively.

---

## Design Principles

1. **JSONB for what varies; columns for what is queried and constrained.** If a field participates in foreign keys, financial calculations, or WHERE clauses in critical queries, it gets a proper column. If it is descriptive metadata that might differ by product form or change as ONIX standards evolve, it goes in JSONB.
2. **GIN indexes on JSONB for searchability.** Key JSONB fields are indexed so that metadata queries perform well without sacrificing flexibility.
3. **Validation at the application layer for JSONB, at the database layer for columns.** CHECK constraints and foreign keys protect financial integrity. JSON Schema validation in the application layer protects metadata quality.
4. **ONIX-native storage.** Product metadata JSONB structure mirrors ONIX 3.0 composite groupings so that export generation is a straightforward mapping rather than a complex multi-table assembly.
5. **Immutable financial appendix.** Royalty calculations and payments use append-only patterns with proper columns, not JSONB, because financial auditability requires explicit schema.

---

## Schema Definition

### 1. Organizations and People

```sql
-- Publishers, imprints, distributors -- stable structure, normalized
CREATE TABLE organizations (
    organization_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_type   VARCHAR(50) NOT NULL CHECK (organization_type IN (
        'publisher', 'imprint', 'distributor', 'retailer', 'agency', 'foreign_publisher'
    )),
    name                VARCHAR(500) NOT NULL,
    parent_org_id       UUID REFERENCES organizations(organization_id),
    country_code        CHAR(2),
    default_currency    CHAR(3),
    -- Variable contact and business info in JSONB
    contact_info        JSONB NOT NULL DEFAULT '{}',
    /*  contact_info example:
        {
            "address": {"street": "...", "city": "...", "state": "...", "postal": "...", "country": "US"},
            "phone": "+1-212-555-0100",
            "email": "info@publisher.com",
            "website": "https://publisher.com",
            "tax_ids": {"us_ein": "12-3456789", "uk_vat": "GB123456789"},
            "payment_details": {"bank": "...", "routing": "...", "account": "..."},
            "onix_sender_info": {"sender_name": "...", "sender_contact": "...", "sender_email": "..."}
        }
    */
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_organizations_type ON organizations(organization_type);
CREATE INDEX idx_organizations_parent ON organizations(parent_org_id);

-- People: authors, editors, agents -- core fields as columns, variable info as JSONB
CREATE TABLE persons (
    person_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    display_name        VARCHAR(500) NOT NULL,
    sort_name           VARCHAR(500),
    email               VARCHAR(300),
    -- Variable personal and professional details
    profile             JSONB NOT NULL DEFAULT '{}',
    /*  profile example:
        {
            "first_name": "Margaret",
            "last_name": "Atwood",
            "pseudonyms": ["M.E. Atwood"],
            "biography": "Margaret Atwood is a Canadian poet...",
            "biography_short": "Award-winning author of The Handmaid's Tale",
            "nationality": "CA",
            "languages": ["eng", "fra"],
            "date_of_birth": "1939-11-18",
            "website": "https://margaretatwood.ca",
            "social_media": {"twitter": "@MargaretAtwood"},
            "photo_url": "https://...",
            "awards": ["Booker Prize 2000", "Arthur C. Clarke Award 1987"],
            "agent": {"name": "...", "agency": "...", "email": "..."},
            "contributor_codes": {"ONIX": "A01", "MARC": "100"}
        }
    */
    -- Payment info as separate JSONB (sensitive, could be encrypted at column level)
    payment_info        JSONB NOT NULL DEFAULT '{}',
    /*  payment_info example:
        {
            "tax_id": "123-45-6789",
            "payment_currency": "USD",
            "payment_method": "wire_transfer",
            "bank_details": {"bank": "...", "routing": "...", "account": "..."},
            "w9_on_file": true,
            "w9_date": "2024-03-15"
        }
    */
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_persons_display_name ON persons(display_name);
CREATE INDEX idx_persons_sort_name ON persons(sort_name);
CREATE INDEX idx_persons_profile ON persons USING gin(profile);
```

### 2. Works and Products (Hybrid: Columns + JSONB)

```sql
-- Works: the intellectual property. Core fields as columns, descriptive metadata as JSONB
CREATE TABLE works (
    work_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title               VARCHAR(1000) NOT NULL,
    subtitle            VARCHAR(1000),
    work_type           VARCHAR(50) NOT NULL,
    original_language   CHAR(3),
    publication_status  VARCHAR(50) NOT NULL DEFAULT 'in_development',
    acquisition_date    DATE,
    -- Flexible metadata: series info, editorial notes, AI analysis, marketing info
    metadata            JSONB NOT NULL DEFAULT '{}',
    /*  metadata example:
        {
            "series": {
                "series_id": "...",
                "series_name": "Discworld",
                "position": 15,
                "issn": null
            },
            "edition": {
                "number": 1,
                "statement": "First edition",
                "revision_date": null
            },
            "synopsis": "In a world where...",
            "internal_notes": "Comparable to The Night Circus...",
            "ai_analysis": {
                "comparable_titles": ["978-0-385-33348-1", "978-0-06-112008-4"],
                "predicted_first_year_sales": 12000,
                "confidence": 0.72,
                "model_version": "forecast-v3.2",
                "analysis_date": "2026-04-15"
            },
            "marketing": {
                "tagline": "A spellbinding debut...",
                "selling_points": ["#1 bestseller in Germany", "Film rights optioned"],
                "keywords": ["magical realism", "family saga", "debut"],
                "comp_titles": ["The House in the Cerulean Sea", "Piranesi"]
            }
        }
    */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_works_title ON works USING gin(to_tsvector('english', title));
CREATE INDEX idx_works_status ON works(publication_status);
CREATE INDEX idx_works_metadata ON works USING gin(metadata);

-- Work contributors: normalized junction for queryability
CREATE TABLE work_contributors (
    work_contributor_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_id             UUID NOT NULL REFERENCES works(work_id),
    person_id           UUID NOT NULL REFERENCES persons(person_id),
    contributor_role    VARCHAR(50) NOT NULL,
    sequence_number     INTEGER NOT NULL DEFAULT 1,
    contribution_pct    DECIMAL(5,2),
    -- Role-specific details that vary by contributor type
    role_details        JSONB NOT NULL DEFAULT '{}',
    /*  role_details examples:
        For translator: {"source_language": "deu", "target_language": "eng"}
        For narrator:   {"studio": "Audible Studios", "voice_type": "male"}
        For illustrator: {"illustration_type": "full_color", "page_count": 32}
    */
    UNIQUE(work_id, person_id, contributor_role)
);

CREATE INDEX idx_work_contributors_work ON work_contributors(work_id);
CREATE INDEX idx_work_contributors_person ON work_contributors(person_id);

-- Products: ISBN-level format/edition. This is where JSONB shines most.
CREATE TABLE products (
    product_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_id             UUID NOT NULL REFERENCES works(work_id),
    isbn_13             CHAR(13) UNIQUE,
    isbn_10             CHAR(10),
    product_form        VARCHAR(50) NOT NULL,
    list_price          DECIMAL(10,2),
    price_currency      CHAR(3) DEFAULT 'USD',
    publication_date    DATE,
    on_sale_date        DATE,
    out_of_print_date   DATE,
    publisher_id        UUID NOT NULL REFERENCES organizations(organization_id),
    imprint_id          UUID REFERENCES organizations(organization_id),
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,

    -- ONIX-aligned descriptive detail (varies dramatically by product form)
    descriptive_detail  JSONB NOT NULL DEFAULT '{}',
    /*  descriptive_detail for a hardback:
        {
            "page_count": 384,
            "word_count": 95000,
            "physical": {
                "trim_width_mm": 152,
                "trim_height_mm": 229,
                "spine_width_mm": 25,
                "weight_grams": 520
            },
            "binding": "case_bound",
            "paper_type": "acid_free",
            "jacket": true,
            "printed_endpapers": false,
            "illustrations": {"count": 0, "type": null},
            "country_of_manufacture": "US"
        }

        descriptive_detail for an audiobook:
        {
            "duration_minutes": 720,
            "narrators": [
                {"person_id": "...", "name": "Stephen Fry", "role": "narrator"}
            ],
            "audio_format": "unabridged",
            "file_format": "mp3",
            "bitrate_kbps": 128,
            "studio": "Penguin Audio",
            "recording_date": "2025-11-01",
            "sample_url": "https://..."
        }

        descriptive_detail for an ebook:
        {
            "file_formats": ["epub3", "kindle_kf8"],
            "drm": "adobe_acs4",
            "fixed_layout": false,
            "reflowable": true,
            "enhanced_content": false,
            "word_count": 95000,
            "file_size_mb": 2.4,
            "accessibility": {
                "screen_reader_compatible": true,
                "alt_text_images": true,
                "wcag_level": "AA"
            }
        }
    */

    -- Subject classifications (BISAC, THEMA, BIC) -- array of objects
    classifications     JSONB NOT NULL DEFAULT '[]',
    /*  classifications example:
        [
            {"scheme": "BISAC", "code": "FIC019000", "heading": "FICTION / Literary", "is_primary": true},
            {"scheme": "BISAC", "code": "FIC028000", "heading": "FICTION / Science Fiction / General", "is_primary": false},
            {"scheme": "THEMA", "code": "FB", "heading": "Fiction: general & literary", "is_primary": true},
            {"scheme": "THEMA", "code": "1KBB", "heading": "United Kingdom", "qualifier_type": "place"}
        ]
    */

    -- ONIX-specific metadata for export (notification type, product composition, audience, etc.)
    onix_metadata       JSONB NOT NULL DEFAULT '{}',
    /*  onix_metadata example:
        {
            "notification_type": "03",
            "product_composition": "00",
            "product_form_code": "BB",
            "product_form_detail": "B305",
            "audience_codes": [{"type": "01", "value": "01"}],
            "edition_type": "NED",
            "text_language": "eng",
            "countries_of_publication": ["US", "CA", "GB"],
            "sales_restrictions": [],
            "related_products": [
                {"relation": "13", "isbn": "978-0-123456-79-0", "form": "ebook"}
            ],
            "prizes": [
                {"name": "Booker Prize", "year": 2025, "achievement": "shortlisted"}
            ]
        }
    */

    -- Pricing by territory/channel (complex, variable structure)
    pricing             JSONB NOT NULL DEFAULT '[]',
    /*  pricing example:
        [
            {
                "territory": ["US", "CA"],
                "currency": "USD",
                "price_type": "retail",
                "amount": 27.99,
                "tax_included": false,
                "effective_date": "2026-06-01",
                "end_date": null
            },
            {
                "territory": ["GB"],
                "currency": "GBP",
                "price_type": "retail",
                "amount": 16.99,
                "tax_included": true,
                "tax_rate_pct": 0,
                "effective_date": "2026-07-01"
            },
            {
                "territory": ["AU"],
                "currency": "AUD",
                "price_type": "retail",
                "amount": 32.99,
                "tax_included": true,
                "tax_rate_pct": 10,
                "effective_date": "2026-07-15"
            }
        ]
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_products_work ON products(work_id);
CREATE INDEX idx_products_isbn ON products(isbn_13);
CREATE INDEX idx_products_form ON products(product_form);
CREATE INDEX idx_products_pub_date ON products(publication_date);
CREATE INDEX idx_products_descriptive ON products USING gin(descriptive_detail);
CREATE INDEX idx_products_classifications ON products USING gin(classifications);
CREATE INDEX idx_products_onix ON products USING gin(onix_metadata);

-- ISBN prefix management (normalized -- small, stable data)
CREATE TABLE isbn_prefixes (
    prefix_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(organization_id),
    prefix              VARCHAR(20) NOT NULL,
    total_isbns         INTEGER NOT NULL,
    assigned_count      INTEGER NOT NULL DEFAULT 0,
    agency              VARCHAR(100),
    registration_date   DATE,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 3. Editorial Workflow (Hybrid: Schema + JSONB for Flexibility)

```sql
-- Pipeline templates: defined structure, but stage configurations vary
CREATE TABLE pipeline_templates (
    template_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_name       VARCHAR(200) NOT NULL,
    description         TEXT,
    -- Stage definitions as JSONB array (order, names, durations, milestone flags)
    stages              JSONB NOT NULL DEFAULT '[]',
    /*  stages example:
        [
            {"order": 1, "name": "Acquisition Review", "default_days": 14, "is_milestone": false,
             "required_approvers": 2, "checklist": ["Market analysis", "P&L projection"]},
            {"order": 2, "name": "Structural Edit", "default_days": 30, "is_milestone": false,
             "checklist": ["Chapter structure", "Character arcs", "Plot consistency"]},
            {"order": 3, "name": "Manuscript Delivery", "default_days": 0, "is_milestone": true,
             "triggers_payment": "on_delivery"},
            {"order": 4, "name": "Copy Edit", "default_days": 21, "is_milestone": false},
            {"order": 5, "name": "Proofread", "default_days": 14, "is_milestone": false},
            {"order": 6, "name": "Design & Layout", "default_days": 28, "is_milestone": false,
             "checklist": ["Cover design", "Interior layout", "Index"]},
            {"order": 7, "name": "Print Ready", "default_days": 7, "is_milestone": true,
             "triggers_payment": "on_acceptance"}
        ]
    */
    is_default          BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Work pipeline tracking
CREATE TABLE work_pipelines (
    pipeline_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_id             UUID NOT NULL REFERENCES works(work_id),
    template_id         UUID NOT NULL REFERENCES pipeline_templates(template_id),
    current_stage_order INTEGER NOT NULL DEFAULT 1,
    current_stage_name  VARCHAR(200),
    status              VARCHAR(30) NOT NULL DEFAULT 'active' CHECK (status IN (
        'active', 'paused', 'completed', 'cancelled'
    )),
    started_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at        TIMESTAMPTZ,
    -- Full stage history as JSONB (append-only within the JSON array)
    stage_history       JSONB NOT NULL DEFAULT '[]',
    /*  stage_history example:
        [
            {
                "stage_order": 1,
                "stage_name": "Acquisition Review",
                "assigned_to": "person-uuid-123",
                "assigned_name": "Sarah Chen",
                "started_at": "2026-01-15T10:00:00Z",
                "completed_at": "2026-01-28T16:30:00Z",
                "status": "approved",
                "notes": "Strong debut. Comparable to recent Picador acquisitions.",
                "checklist_results": {"Market analysis": true, "P&L projection": true}
            },
            {
                "stage_order": 2,
                "stage_name": "Structural Edit",
                "assigned_to": "person-uuid-456",
                "assigned_name": "James Wright",
                "started_at": "2026-02-01T09:00:00Z",
                "completed_at": null,
                "status": "in_progress",
                "notes": null
            }
        ]
    */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_work_pipelines_work ON work_pipelines(work_id);
CREATE INDEX idx_work_pipelines_status ON work_pipelines(status);

-- Submissions: mostly variable metadata, stored as JSONB
CREATE TABLE submissions (
    submission_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    status              VARCHAR(50) NOT NULL DEFAULT 'received' CHECK (status IN (
        'received', 'acknowledged', 'under_review', 'requested_full',
        'accepted', 'rejected', 'withdrawn'
    )),
    submitted_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    decided_at          TIMESTAMPTZ,
    work_id             UUID REFERENCES works(work_id),  -- Linked if accepted
    -- All submission content and metadata as JSONB
    submission_data     JSONB NOT NULL,
    /*  submission_data example:
        {
            "title": "The Glass Garden",
            "author": {
                "name": "Elena Marchetti",
                "email": "elena@example.com",
                "previous_publications": ["Short story in Granta 142"],
                "bio": "Elena Marchetti is an Italian-American writer..."
            },
            "agent": {
                "name": "David Park",
                "agency": "Park Literary",
                "email": "david@parkliterary.com"
            },
            "manuscript": {
                "genre": "literary fiction",
                "word_count": 82000,
                "synopsis": "A multi-generational story...",
                "cover_letter": "Dear Editor, I am submitting...",
                "sample_chapters": 3,
                "file_url": "s3://submissions/2026/03/glass-garden-marchetti.docx",
                "file_format": "docx"
            },
            "ai_triage": {
                "quality_score": 78.5,
                "genre_tags": ["literary_fiction", "family_saga", "immigrant_story"],
                "comparable_works": ["Pachinko", "The Namesake"],
                "strengths": ["Strong prose", "Compelling narrator voice"],
                "concerns": ["Slow pacing in chapters 4-6"],
                "confidence": 0.81,
                "model_version": "triage-v2.1",
                "scored_at": "2026-03-15T14:22:00Z"
            },
            "review": {
                "reviewer_id": "person-uuid-789",
                "reviewer_name": "Emma Sullivan",
                "review_date": "2026-03-20",
                "recommendation": "request_full",
                "notes": "Exceptional voice. Request full manuscript and P&L."
            }
        }
    */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_submissions_status ON submissions(status);
CREATE INDEX idx_submissions_date ON submissions(submitted_at);
CREATE INDEX idx_submissions_data ON submissions USING gin(submission_data);

-- Editorial tasks
CREATE TABLE editorial_tasks (
    task_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_id             UUID NOT NULL REFERENCES works(work_id),
    task_type           VARCHAR(100) NOT NULL,
    title               VARCHAR(500) NOT NULL,
    assigned_to         UUID REFERENCES persons(person_id),
    priority            VARCHAR(20) DEFAULT 'normal',
    status              VARCHAR(30) NOT NULL DEFAULT 'open',
    due_date            DATE,
    completed_at        TIMESTAMPTZ,
    -- Task-specific details vary by task type
    task_details        JSONB NOT NULL DEFAULT '{}',
    /*  task_details examples:
        For copy_edit: {"chapter_range": "1-15", "style_guide": "Chicago 17th", "track_changes": true}
        For cover_design: {"dimensions": "6x9", "spine_width_mm": 25, "reference_covers": [...]}
        For index: {"index_type": "subject", "page_count": 384, "turnaround_days": 14}
    */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_editorial_tasks_work ON editorial_tasks(work_id);
CREATE INDEX idx_editorial_tasks_assigned ON editorial_tasks(assigned_to);
CREATE INDEX idx_editorial_tasks_status ON editorial_tasks(status);
```

### 4. Contracts and Rights (Normalized -- Financial/Legal Integrity)

```sql
-- Contracts: fully normalized, financial integrity is paramount
CREATE TABLE contracts (
    contract_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_number     VARCHAR(100) UNIQUE NOT NULL,
    work_id             UUID NOT NULL REFERENCES works(work_id),
    contract_type       VARCHAR(50) NOT NULL,
    counterparty_person_id  UUID REFERENCES persons(person_id),
    counterparty_org_id     UUID REFERENCES organizations(organization_id),
    agent_person_id     UUID REFERENCES persons(person_id),
    agent_commission_pct DECIMAL(5,2),
    execution_date      DATE,
    effective_date      DATE NOT NULL,
    expiration_date     DATE,
    termination_date    DATE,
    contract_currency   CHAR(3) NOT NULL DEFAULT 'USD',
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
    document_url        VARCHAR(1000),
    -- Contract-specific clauses and terms that vary significantly
    contract_terms      JSONB NOT NULL DEFAULT '{}',
    /*  contract_terms example:
        {
            "option_clause": {
                "type": "next_work",
                "terms": "Publisher has right of first refusal on author's next work",
                "response_period_days": 60
            },
            "non_compete": {
                "duration_months": 12,
                "scope": "same genre and format"
            },
            "reversion_conditions": {
                "out_of_print_definition": "fewer than 100 copies sold in trailing 12 months across all formats",
                "reversion_notice_period_days": 90,
                "cure_period_days": 180
            },
            "accounting_period": "semi_annual",
            "payment_timing_days": 90,
            "minimum_payment_threshold": 25.00,
            "audit_rights": "Author may audit once per 12 months with 30 days notice",
            "territory_notes": "World English excluding Indian subcontinent",
            "special_provisions": [
                "Author retains approval over cover design",
                "Publisher will produce audiobook within 18 months of hardback publication"
            ],
            "ai_clause_analysis": {
                "flagged_terms": ["Non-compete scope unusually broad"],
                "model_version": "contract-v1.3",
                "analysis_date": "2026-04-01"
            }
        }
    */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_contracts_work ON contracts(work_id);
CREATE INDEX idx_contracts_status ON contracts(status);
CREATE INDEX idx_contracts_expiration ON contracts(expiration_date);

-- Advances: normalized (financial data, explicit columns)
CREATE TABLE contract_advances (
    advance_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id         UUID NOT NULL REFERENCES contracts(contract_id),
    tranche_number      INTEGER NOT NULL,
    tranche_description VARCHAR(200),
    trigger_type        VARCHAR(50),
    amount              DECIMAL(12,2) NOT NULL,
    currency            CHAR(3) NOT NULL DEFAULT 'USD',
    status              VARCHAR(30) NOT NULL DEFAULT 'scheduled',
    scheduled_date      DATE,
    paid_date           DATE,
    payment_reference   VARCHAR(200),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(contract_id, tranche_number)
);

CREATE INDEX idx_advances_contract ON contract_advances(contract_id);

-- Royalty rate schedules: normalized (calculation-critical data)
CREATE TABLE royalty_rate_schedules (
    rate_schedule_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id         UUID NOT NULL REFERENCES contracts(contract_id),
    product_form        VARCHAR(50),
    sales_channel       VARCHAR(100),
    territory_code      VARCHAR(10),
    rate_basis          VARCHAR(30) NOT NULL,
    valid_from          DATE NOT NULL,
    valid_until         DATE,
    -- Tier structure as JSONB (more natural than a separate tiers table)
    tiers               JSONB NOT NULL,
    /*  tiers example:
        [
            {"floor": 0, "ceiling": 5000, "rate_pct": 10.0},
            {"floor": 5001, "ceiling": 10000, "rate_pct": 12.5},
            {"floor": 10001, "ceiling": null, "rate_pct": 15.0}
        ]
    */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rate_schedules_contract ON royalty_rate_schedules(contract_id);

-- Rights grants: normalized for territory lookups and expiry alerts
CREATE TABLE rights_grants (
    grant_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id         UUID NOT NULL REFERENCES contracts(contract_id),
    right_type          VARCHAR(50) NOT NULL,
    territory_type      VARCHAR(20) NOT NULL,
    language_code       CHAR(3),
    is_exclusive        BOOLEAN NOT NULL DEFAULT TRUE,
    grant_start         DATE NOT NULL,
    grant_end           DATE,
    status              VARCHAR(30) NOT NULL DEFAULT 'active',
    -- Variable reversion and sub-rights terms
    grant_details       JSONB NOT NULL DEFAULT '{}',
    /*  grant_details example:
        {
            "reversion_clause": "If out of print for 12 consecutive months...",
            "reversion_trigger": "out_of_print_12_months",
            "minimum_in_print_threshold": 100,
            "option_for_renewal": true,
            "renewal_terms": "Auto-renew for 3 years if sales exceed threshold",
            "restrictions": ["No digital-first publication without author consent"]
        }
    */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rights_grants_contract ON rights_grants(contract_id);
CREATE INDEX idx_rights_grants_status ON rights_grants(status);
CREATE INDEX idx_rights_grants_expiry ON rights_grants(grant_end);

-- Rights grant territories (normalized for lookup queries)
CREATE TABLE rights_grant_territories (
    grant_territory_id  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    grant_id            UUID NOT NULL REFERENCES rights_grants(grant_id),
    country_code        CHAR(2) NOT NULL,
    is_included         BOOLEAN NOT NULL DEFAULT TRUE,
    UNIQUE(grant_id, country_code)
);

CREATE INDEX idx_rgt_grant ON rights_grant_territories(grant_id);
CREATE INDEX idx_rgt_country ON rights_grant_territories(country_code);

-- Sub-licenses (normalized)
CREATE TABLE sub_licenses (
    sub_license_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_grant_id     UUID NOT NULL REFERENCES rights_grants(grant_id),
    licensee_org_id     UUID NOT NULL REFERENCES organizations(organization_id),
    contract_id         UUID NOT NULL REFERENCES contracts(contract_id),
    advance_amount      DECIMAL(12,2),
    advance_currency    CHAR(3),
    royalty_split_pct   DECIMAL(5,2),
    author_split_pct    DECIMAL(5,2),
    effective_date      DATE NOT NULL,
    expiration_date     DATE,
    status              VARCHAR(30) NOT NULL DEFAULT 'active',
    -- Variable license terms
    license_terms       JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 5. Royalty Engine (Normalized -- Financial Calculations)

```sql
-- Royalty periods (normalized)
CREATE TABLE royalty_periods (
    period_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    period_name         VARCHAR(100) NOT NULL,
    period_start        DATE NOT NULL,
    period_end          DATE NOT NULL,
    period_type         VARCHAR(20) NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'open',
    finalized_at        TIMESTAMPTZ,
    finalized_by        UUID REFERENCES persons(person_id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(period_start, period_end)
);

-- Sales data import batches
CREATE TABLE sales_import_batches (
    batch_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    distributor_id      UUID NOT NULL REFERENCES organizations(organization_id),
    file_name           VARCHAR(500),
    period_id           UUID REFERENCES royalty_periods(period_id),
    status              VARCHAR(30) NOT NULL DEFAULT 'pending',
    -- Import stats and error details as JSONB (variable per distributor format)
    import_metadata     JSONB NOT NULL DEFAULT '{}',
    /*  import_metadata example:
        {
            "file_format": "ingram_csv_v3",
            "parser_version": "2.1.0",
            "row_count": 15234,
            "valid_rows": 15200,
            "invalid_rows": 34,
            "total_units": 45678,
            "total_gross": 567890.12,
            "currency": "USD",
            "unmatched_isbns": ["978-0-000000-00-0"],
            "warnings": ["3 rows with negative quantities treated as returns"],
            "column_mapping": {"isbn": "A", "qty": "D", "price": "F"},
            "errors": []
        }
    */
    imported_at         TIMESTAMPTZ,
    imported_by         UUID REFERENCES persons(person_id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Sales transactions: normalized for calculation joins
CREATE TABLE sales_transactions (
    transaction_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id          UUID NOT NULL REFERENCES products(product_id),
    period_id           UUID NOT NULL REFERENCES royalty_periods(period_id),
    distributor_id      UUID NOT NULL REFERENCES organizations(organization_id),
    sales_channel       VARCHAR(100),
    territory_code      CHAR(2),
    transaction_type    VARCHAR(30) NOT NULL,
    quantity            INTEGER NOT NULL,
    unit_price          DECIMAL(10,2),
    sale_currency       CHAR(3) NOT NULL,
    gross_amount        DECIMAL(12,2) NOT NULL,
    discount_pct        DECIMAL(5,2),
    net_amount          DECIMAL(12,2) NOT NULL,
    reporting_amount    DECIMAL(12,2),
    reporting_currency  CHAR(3),
    exchange_rate       DECIMAL(12,6),
    import_batch_id     UUID NOT NULL REFERENCES sales_import_batches(batch_id),
    sale_date           DATE,
    -- Distributor-specific raw data preserved as JSONB
    raw_data            JSONB,
    /*  raw_data preserves the original row from the distributor:
        {
            "source_row": 1542,
            "original_fields": {
                "isbn": "9780123456789",
                "title": "The Glass Garden",
                "qty_shipped": 150,
                "qty_returned": 12,
                "list_price": 27.99,
                "discount": 0.48,
                "net_price": 14.55,
                "account": "B&N National"
            }
        }
    */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (sale_date);

CREATE INDEX idx_sales_product ON sales_transactions(product_id);
CREATE INDEX idx_sales_period ON sales_transactions(period_id);
CREATE INDEX idx_sales_distributor ON sales_transactions(distributor_id);

-- Create initial partitions
CREATE TABLE sales_transactions_2024 PARTITION OF sales_transactions
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
CREATE TABLE sales_transactions_2025 PARTITION OF sales_transactions
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
CREATE TABLE sales_transactions_2026 PARTITION OF sales_transactions
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');

-- Royalty calculations: normalized, append-only
CREATE TABLE royalty_calculations (
    calculation_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id         UUID NOT NULL REFERENCES contracts(contract_id),
    period_id           UUID NOT NULL REFERENCES royalty_periods(period_id),
    product_id          UUID REFERENCES products(product_id),
    total_units_sold    INTEGER NOT NULL DEFAULT 0,
    total_units_returned INTEGER NOT NULL DEFAULT 0,
    net_units           INTEGER NOT NULL DEFAULT 0,
    cumulative_units    INTEGER NOT NULL DEFAULT 0,
    gross_revenue       DECIMAL(14,2) NOT NULL DEFAULT 0,
    net_revenue         DECIMAL(14,2) NOT NULL DEFAULT 0,
    royalty_basis_amount DECIMAL(14,2) NOT NULL DEFAULT 0,
    calculated_royalty  DECIMAL(14,2) NOT NULL DEFAULT 0,
    reserve_held        DECIMAL(14,2) NOT NULL DEFAULT 0,
    reserve_released    DECIMAL(14,2) NOT NULL DEFAULT 0,
    advance_recoupment  DECIMAL(14,2) NOT NULL DEFAULT 0,
    agent_commission    DECIMAL(14,2) NOT NULL DEFAULT 0,
    net_payable         DECIMAL(14,2) NOT NULL DEFAULT 0,
    calculation_currency CHAR(3) NOT NULL DEFAULT 'USD',
    -- Calculation details preserved for audit
    calculation_details JSONB NOT NULL DEFAULT '{}',
    /*  calculation_details example:
        {
            "rate_schedule_used": "rs-uuid-123",
            "rate_basis": "list_price",
            "tiers_applied": [
                {"floor": 0, "ceiling": 10000, "rate_pct": 12.5, "units": 1500, "amount": 4685.63},
                {"floor": 10001, "ceiling": null, "rate_pct": 15.0, "units": 1700, "amount": 6372.45}
            ],
            "exchange_rate_used": 1.0,
            "reserve_pct": 25,
            "recoupment_source": "unearned_advance",
            "prior_cumulative_units": 8500
        }
    */
    is_override         BOOLEAN NOT NULL DEFAULT FALSE,
    override_reason     TEXT,
    calculated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(contract_id, period_id, product_id)
);

CREATE INDEX idx_royalty_calc_contract ON royalty_calculations(contract_id);
CREATE INDEX idx_royalty_calc_period ON royalty_calculations(period_id);

-- Advance balances: normalized, append-only
CREATE TABLE advance_balances (
    balance_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id         UUID NOT NULL REFERENCES contracts(contract_id),
    period_id           UUID NOT NULL REFERENCES royalty_periods(period_id),
    opening_balance     DECIMAL(12,2) NOT NULL,
    advance_paid        DECIMAL(12,2) NOT NULL DEFAULT 0,
    royalties_earned    DECIMAL(12,2) NOT NULL DEFAULT 0,
    closing_balance     DECIMAL(12,2) NOT NULL,
    is_earned_out       BOOLEAN NOT NULL DEFAULT FALSE,
    earned_out_date     DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(contract_id, period_id)
);

-- Royalty statements and line items
CREATE TABLE royalty_statements (
    statement_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id         UUID NOT NULL REFERENCES contracts(contract_id),
    period_id           UUID NOT NULL REFERENCES royalty_periods(period_id),
    person_id           UUID NOT NULL REFERENCES persons(person_id),
    statement_number    VARCHAR(50) UNIQUE NOT NULL,
    statement_date      DATE NOT NULL,
    total_earned        DECIMAL(14,2) NOT NULL,
    total_reserves      DECIMAL(14,2) NOT NULL,
    total_recoupment    DECIMAL(14,2) NOT NULL,
    total_agent_comm    DECIMAL(14,2) NOT NULL DEFAULT 0,
    net_payable         DECIMAL(14,2) NOT NULL,
    payment_currency    CHAR(3) NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
    sent_at             TIMESTAMPTZ,
    paid_at             TIMESTAMPTZ,
    payment_reference   VARCHAR(200),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE royalty_statement_lines (
    line_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    statement_id        UUID NOT NULL REFERENCES royalty_statements(statement_id),
    product_id          UUID NOT NULL REFERENCES products(product_id),
    line_type           VARCHAR(30) NOT NULL,
    description         VARCHAR(500),
    units               INTEGER,
    unit_price          DECIMAL(10,2),
    royalty_rate_pct    DECIMAL(6,3),
    amount              DECIMAL(12,2) NOT NULL,
    currency            CHAR(3) NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 6. Exchange Rates and Audit

```sql
-- Exchange rates: normalized
CREATE TABLE exchange_rates (
    rate_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    from_currency       CHAR(3) NOT NULL,
    to_currency         CHAR(3) NOT NULL,
    rate                DECIMAL(14,8) NOT NULL,
    rate_date           DATE NOT NULL,
    rate_type           VARCHAR(20) NOT NULL,
    source              VARCHAR(100),
    UNIQUE(from_currency, to_currency, rate_date, rate_type)
);

-- Audit log: JSONB for old/new values (natural fit for variable schemas)
CREATE TABLE audit_log (
    audit_id            BIGSERIAL PRIMARY KEY,
    table_name          VARCHAR(100) NOT NULL,
    record_id           UUID NOT NULL,
    action              VARCHAR(20) NOT NULL,
    old_values          JSONB,
    new_values          JSONB,
    changed_by          UUID,
    changed_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (changed_at);
```

---

## JSONB Query Examples

The power of the hybrid model is that JSONB data remains queryable:

```sql
-- Find all hardback products with page count > 300
SELECT p.isbn_13, w.title, p.descriptive_detail->>'page_count' AS pages
FROM products p
JOIN works w ON w.work_id = p.work_id
WHERE p.product_form = 'hardback'
  AND (p.descriptive_detail->>'page_count')::int > 300;

-- Find products classified under a specific BISAC code
SELECT p.isbn_13, w.title
FROM products p
JOIN works w ON w.work_id = p.work_id
WHERE p.classifications @> '[{"scheme": "BISAC", "code": "FIC019000"}]'::jsonb;

-- Find all audiobooks with duration over 10 hours
SELECT p.isbn_13, w.title,
       p.descriptive_detail->>'duration_minutes' AS duration
FROM products p
JOIN works w ON w.work_id = p.work_id
WHERE p.product_form LIKE 'audiobook%'
  AND (p.descriptive_detail->>'duration_minutes')::int > 600;

-- Find submissions with AI quality score above 80
SELECT submission_id,
       submission_data->'manuscript'->>'title' AS title,
       submission_data->'author'->>'name' AS author,
       (submission_data->'ai_triage'->>'quality_score')::numeric AS score
FROM submissions
WHERE (submission_data->'ai_triage'->>'quality_score')::numeric > 80
ORDER BY score DESC;

-- Find contracts with option clauses
SELECT c.contract_number, w.title,
       c.contract_terms->'option_clause'->>'terms' AS option_terms
FROM contracts c
JOIN works w ON w.work_id = c.work_id
WHERE c.contract_terms ? 'option_clause';

-- Find products with US pricing above $25
SELECT p.isbn_13, w.title, price_entry->>'amount' AS price
FROM products p
JOIN works w ON w.work_id = p.work_id,
     jsonb_array_elements(p.pricing) AS price_entry
WHERE price_entry->'territory' ? 'US'
  AND (price_entry->>'amount')::numeric > 25;

-- ONIX export: gather all product data in a single query
SELECT
    p.isbn_13,
    w.title,
    p.product_form,
    p.descriptive_detail,
    p.classifications,
    p.onix_metadata,
    p.pricing,
    jsonb_agg(jsonb_build_object(
        'name', per.display_name,
        'role', wc.contributor_role,
        'sequence', wc.sequence_number,
        'role_details', wc.role_details
    )) AS contributors
FROM products p
JOIN works w ON w.work_id = p.work_id
LEFT JOIN work_contributors wc ON wc.work_id = w.work_id
LEFT JOIN persons per ON per.person_id = wc.person_id
WHERE p.is_active = TRUE
GROUP BY p.product_id, w.work_id;
```

---

## Pros and Cons

### Pros

1. **Best of both worlds**: Financial data (contracts, royalties, advances) retains full referential integrity and constraint enforcement. Product metadata (ONIX attributes, format-specific specs, classifications) enjoys schema flexibility without EAV pattern complexity.
2. **ONIX export simplification**: Product metadata stored in JSONB that mirrors ONIX composite structure means export generation is a near-direct mapping, not a 12-table join reassembly.
3. **Schema evolution without migrations for metadata**: Adding support for a new ebook DRM type, a new audiobook attribute, or a new ONIX code does not require a database migration -- just update the application validation.
4. **Single-query product views**: All product metadata (descriptive, classification, pricing, ONIX) lives on a single row. No joins needed to render a product detail page.
5. **Distributor format flexibility**: The `raw_data` JSONB on sales transactions preserves the original distributor data exactly as received, enabling debugging without maintaining format-specific archive tables.
6. **Reduced table count**: The hybrid model eliminates approximately 8-10 tables that would exist in a fully normalized schema (pipeline_stages, product_subjects, product_onix_metadata, royalty_rate_tiers, etc.), reducing schema complexity.
7. **GIN indexing**: PostgreSQL GIN indexes on JSONB columns enable fast containment queries (`@>`) and existence checks (`?`), maintaining query performance despite the flexible schema.
8. **Natural for editorial workflow**: Checklist items, stage configurations, and task-specific details vary enormously and change frequently. JSONB handles this without schema changes.

### Cons

1. **No foreign key constraints in JSONB**: A `person_id` referenced inside a JSONB `submission_data` or `stage_history` object cannot be validated by the database. Referential integrity for JSONB-embedded references depends entirely on application logic.
2. **Type safety in JSONB requires casting**: Queries like `(descriptive_detail->>'page_count')::int` are verbose and will fail at runtime if the value is not numeric. Application-layer validation must be rigorous.
3. **Partial indexing limitations**: While GIN indexes support containment queries, range queries on JSONB values (e.g., "all products with price between 10 and 20 in USD") require expression indexes or are slower than equivalent column queries.
4. **JSONB update granularity**: Updating a single field inside a JSONB column rewrites the entire JSONB value. For frequently-updated JSONB columns (like `stage_history` with appended entries), this creates write amplification and potential for lost updates under concurrency.
5. **Tooling and ORM gaps**: ORMs handle JSONB with varying levels of sophistication. Complex JSONB queries often require raw SQL, reducing the benefit of ORM abstractions.
6. **Documentation burden**: With JSONB, the schema is implicit. Developers must maintain separate documentation (or JSON Schema definitions) describing the expected structure of each JSONB column. Without discipline, JSONB columns become dumping grounds.
7. **Reporting complexity**: BI tools and report generators work best with flat relational tables. JSONB data must be unnested or extracted into views before it is usable in standard reporting tools.

---

## Technology Recommendations

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| **Database** | PostgreSQL 16+ | Native JSONB, GIN indexes, table partitioning, full-text search |
| **JSONB validation** | JSON Schema (AJV for Node.js, jsonschema for Python) | Validate JSONB structure at the application layer before writes |
| **ORM** | Prisma (typed JSON support) or SQLAlchemy (with JSONB column type) | Both support JSONB queries; Prisma has improving JSONB support |
| **ONIX export** | Direct JSONB-to-XML mapping library | Leverage the ONIX-mirrored JSONB structure for efficient export |
| **Migration** | Flyway or Alembic | For relational schema changes; JSONB schema changes handled by application code |
| **Monitoring** | pg_stat_user_tables + custom JSONB depth metrics | Monitor JSONB column sizes and query performance |

---

## Migration and Scaling Considerations

### Data Migration Strategy

1. **Relational data**: Standard ETL from legacy systems into normalized tables (contracts, advances, royalty calculations, rights grants).
2. **Product metadata**: Transform legacy data into the JSONB structure during import. For ONIX-compliant sources, parse existing ONIX XML directly into the `onix_metadata` and `descriptive_detail` JSONB columns.
3. **Historical submissions**: Import as JSONB documents with best-effort field mapping. Missing fields are naturally handled (they simply are not present in the JSONB).

### JSONB Column Management

```sql
-- Create a view to monitor JSONB column sizes
CREATE VIEW jsonb_column_stats AS
SELECT
    'products.descriptive_detail' AS column_path,
    COUNT(*) AS row_count,
    pg_size_pretty(AVG(pg_column_size(descriptive_detail))::bigint) AS avg_size,
    pg_size_pretty(MAX(pg_column_size(descriptive_detail))::bigint) AS max_size
FROM products
UNION ALL
SELECT
    'submissions.submission_data',
    COUNT(*),
    pg_size_pretty(AVG(pg_column_size(submission_data))::bigint),
    pg_size_pretty(MAX(pg_column_size(submission_data))::bigint)
FROM submissions;
```

### Scaling Path

1. **Phase 1**: Single PostgreSQL instance. JSONB columns keep the schema manageable during rapid development. GIN indexes on key JSONB columns.
2. **Phase 2**: Read replicas for portal and reporting queries. Table partitioning for `sales_transactions` and `audit_log`. Materialized views for royalty dashboards extracting JSONB into flat columns.
3. **Phase 3**: If JSONB query performance degrades on specific columns, promote high-traffic JSONB fields to proper columns via migration (e.g., if `page_count` is queried in 80% of product searches, add it as a column while keeping it in JSONB for backward compatibility).
4. **Archival**: Same as relational model -- old sales partitions to cold storage, JSONB documents naturally compress well.
