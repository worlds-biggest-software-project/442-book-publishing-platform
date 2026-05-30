# Data Model Suggestion 4: Graph Database (Neo4j) with Relational Financial Core

## Overview

This model uses a **graph database (Neo4j)** as the primary store for the relationship-rich domains of the Book Publishing Platform -- rights management, contributor networks, catalog relationships, and editorial workflows -- paired with a **relational database (PostgreSQL)** for the financial domains that demand strict ACID compliance: royalty calculations, advance balances, payment records, and sales transactions.

The book publishing domain is fundamentally a network of relationships: works connect to contributors through roles, contributors connect to agents through representation agreements, works connect to contracts through rights grants that span territories and languages, contracts connect to sub-licenses that branch into foreign markets, and products relate to each other across formats and editions. These relationship networks are exactly what graph databases excel at traversing.

The key question this model answers that relational models struggle with is: **"For this work, in this territory, in this language, for this right type, who holds the rights, through which contract chain, and what sub-licenses exist?"** In a relational model, this requires recursive CTEs or multiple self-joins. In a graph model, it is a simple path traversal.

---

## Architecture: Dual-Database Design

```
┌─────────────────────────────────────────────────────┐
│                  Application Layer                   │
│                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │  Rights &     │  │  Editorial    │  │  Royalty   │ │
│  │  Catalog      │  │  Workflow     │  │  Engine    │ │
│  │  Service      │  │  Service     │  │  Service   │ │
│  └──────┬───────┘  └──────┬───────┘  └─────┬─────┘ │
│         │                  │                │       │
│         ▼                  ▼                ▼       │
│  ┌─────────────────────────────┐  ┌──────────────┐ │
│  │         Neo4j               │  │  PostgreSQL  │ │
│  │  (Graph: Rights, Catalog,   │  │  (Financial: │ │
│  │   Contributors, Workflow)   │  │   Royalties, │ │
│  │                             │  │   Sales,     │ │
│  │                             │  │   Payments)  │ │
│  └─────────────────────────────┘  └──────────────┘ │
│                                                     │
│  ┌─────────────────────────────────────────────────┐│
│  │  Synchronization Layer (Change Data Capture)    ││
│  │  Keeps graph IDs consistent with relational IDs ││
│  └─────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────┘
```

---

## Neo4j Graph Model

### Node Types and Properties

#### Work Node

```cypher
CREATE (w:Work {
    work_id: 'uuid-work-001',
    title: 'The Midnight Library',
    subtitle: null,
    work_type: 'monograph',
    original_language: 'eng',
    publication_status: 'published',
    acquisition_date: date('2024-03-15'),
    synopsis: 'Between life and death there is a library...',
    edition_number: 1,
    edition_statement: 'First edition',
    created_at: datetime(),
    updated_at: datetime()
})
```

#### Product Node

```cypher
CREATE (p:Product {
    product_id: 'uuid-prod-001',
    isbn_13: '9780525559474',
    isbn_10: '0525559477',
    product_form: 'hardback',
    list_price: 26.00,
    price_currency: 'USD',
    publication_date: date('2020-09-29'),
    on_sale_date: date('2020-09-29'),
    page_count: 304,
    trim_width_mm: 140,
    trim_height_mm: 210,
    weight_grams: 380,
    is_active: true,
    created_at: datetime()
})

CREATE (pa:Product {
    product_id: 'uuid-prod-002',
    isbn_13: '9780525559481',
    product_form: 'audiobook_download',
    list_price: 29.99,
    price_currency: 'USD',
    publication_date: date('2020-09-29'),
    duration_minutes: 527,
    narrator: 'Carey Mulligan',
    audio_format: 'unabridged',
    is_active: true,
    created_at: datetime()
})
```

#### Person Node

```cypher
CREATE (per:Person {
    person_id: 'uuid-person-001',
    display_name: 'Matt Haig',
    sort_name: 'Haig, Matt',
    email: 'matt@example.com',
    nationality: 'GB',
    biography: 'Matt Haig is a British author...',
    date_of_birth: date('1975-07-03'),
    is_active: true
})

CREATE (per:Person:Agent {
    person_id: 'uuid-agent-001',
    display_name: 'Clare Conville',
    sort_name: 'Conville, Clare',
    agency_name: 'C+W Agency'
})

CREATE (per:Person:Editor {
    person_id: 'uuid-editor-001',
    display_name: 'Sarah McGrath',
    sort_name: 'McGrath, Sarah',
    department: 'Fiction'
})
```

#### Organization Node

```cypher
CREATE (org:Organization:Publisher {
    organization_id: 'uuid-pub-001',
    name: 'Viking Press',
    country_code: 'US',
    default_currency: 'USD',
    is_active: true
})

CREATE (org:Organization:Imprint {
    organization_id: 'uuid-imp-001',
    name: 'Viking',
    parent_organization: 'Penguin Random House'
})

CREATE (org:Organization:Distributor {
    organization_id: 'uuid-dist-001',
    name: 'Ingram Content Group',
    country_code: 'US'
})
```

#### Contract Node

```cypher
CREATE (c:Contract {
    contract_id: 'uuid-contract-001',
    contract_number: 'CTR-2024-00142',
    contract_type: 'author_agreement',
    execution_date: date('2024-03-20'),
    effective_date: date('2024-04-01'),
    expiration_date: null,
    contract_currency: 'USD',
    status: 'active',
    total_advance: 150000.00,
    agent_commission_pct: 15.0,
    document_url: 's3://contracts/CTR-2024-00142.pdf',
    accounting_period: 'semi_annual',
    payment_timing_days: 90,
    minimum_payment_threshold: 25.00
})
```

#### Territory Node

```cypher
// Countries as nodes -- stable reference data
CREATE (t:Territory:Country {
    code: 'US',
    name: 'United States',
    region: 'North America',
    language_primary: 'eng',
    currency: 'USD'
})

CREATE (t:Territory:Country {
    code: 'GB',
    name: 'United Kingdom',
    region: 'Europe',
    language_primary: 'eng',
    currency: 'GBP'
})

// Territory groups as nodes for common groupings
CREATE (tg:TerritoryGroup {
    group_id: 'world_english',
    name: 'World English Language',
    description: 'All English-speaking territories'
})
```

#### Rights Grant Node

```cypher
CREATE (rg:RightsGrant {
    grant_id: 'uuid-grant-001',
    right_type: 'all_print',
    is_exclusive: true,
    grant_start: date('2024-04-01'),
    grant_end: null,
    status: 'active',
    reversion_trigger: 'out_of_print_12_months',
    minimum_in_print_threshold: 100
})
```

#### Series Node

```cypher
CREATE (s:Series {
    series_id: 'uuid-series-001',
    series_name: 'The Comfort Book Series',
    issn: null,
    description: 'Non-fiction reflections on life'
})
```

#### Subject Node

```cypher
CREATE (subj:Subject {
    code: 'FIC019000',
    scheme: 'BISAC',
    heading: 'FICTION / Literary'
})

CREATE (subj:Subject {
    code: 'FB',
    scheme: 'THEMA',
    heading: 'Fiction: general & literary'
})
```

#### Submission Node

```cypher
CREATE (sub:Submission {
    submission_id: 'uuid-sub-001',
    title: 'The Glass Garden',
    author_name: 'Elena Marchetti',
    author_email: 'elena@example.com',
    word_count: 82000,
    genre: 'literary fiction',
    status: 'under_review',
    submitted_at: datetime('2026-03-15T10:00:00Z'),
    ai_quality_score: 78.5,
    ai_genre_tags: ['literary_fiction', 'family_saga']
})
```

#### Pipeline and Stage Nodes

```cypher
CREATE (pt:PipelineTemplate {
    template_id: 'uuid-template-001',
    template_name: 'Standard Fiction Pipeline',
    is_default: true
})

CREATE (stage:PipelineStage {
    stage_id: 'uuid-stage-001',
    stage_name: 'Acquisition Review',
    stage_order: 1,
    default_duration_days: 14,
    is_milestone: false
})

CREATE (stage:PipelineStage {
    stage_id: 'uuid-stage-003',
    stage_name: 'Manuscript Delivery',
    stage_order: 3,
    default_duration_days: 0,
    is_milestone: true,
    triggers_payment: 'on_delivery'
})
```

### Relationship Types

This is where the graph model truly shines. Every relationship is a first-class entity with its own properties.

```cypher
// === Work Relationships ===

// Author/contributor relationships with rich role metadata
(person)-[:CONTRIBUTED_TO {
    role: 'author',
    sequence_number: 1,
    contribution_pct: 100.0,
    credited_as: 'Matt Haig',        // May differ from display_name (pseudonyms)
    contract_id: 'uuid-contract-001'
}]->(work)

(person)-[:CONTRIBUTED_TO {
    role: 'translator',
    sequence_number: 2,
    contribution_pct: null,
    source_language: 'eng',
    target_language: 'deu',
    contract_id: 'uuid-contract-translation-001'
}]->(work)

// Work to Product (format/edition)
(work)-[:HAS_EDITION {
    edition_type: 'original',
    created_at: datetime()
}]->(product)

// Series membership
(work)-[:BELONGS_TO_SERIES {
    position: 3,
    is_main_entry: true
}]->(series)

// Work-to-work relationships (crucial for publishing)
(work1)-[:RELATED_WORK {
    relation_type: 'translation_of',
    language: 'deu'
}]->(work2)

(work1)-[:RELATED_WORK {
    relation_type: 'abridgement_of'
}]->(work2)

(work1)-[:RELATED_WORK {
    relation_type: 'companion_to'
}]->(work2)

(work1)-[:RELATED_WORK {
    relation_type: 'sequel_to',
    sequence: 2
}]->(work2)

// Subject classification
(product)-[:CLASSIFIED_AS {
    is_primary: true
}]->(subject)

// === Organization Relationships ===

// Publisher/imprint hierarchy
(imprint)-[:IMPRINT_OF]->(publisher)
(publisher)-[:SUBSIDIARY_OF]->(parent_publisher)

// Publication
(product)-[:PUBLISHED_BY {
    publication_date: date('2020-09-29')
}]->(publisher)

(product)-[:UNDER_IMPRINT]->(imprint)

// === Contract Relationships ===

// Contract parties
(contract)-[:FOR_WORK]->(work)
(contract)-[:WITH_AUTHOR]->(person)
(contract)-[:THROUGH_AGENT {
    commission_pct: 15.0
}]->(agent)
(contract)-[:SIGNED_BY_PUBLISHER]->(organization)

// === Rights Relationships (the graph's greatest strength) ===

// Rights granted under a contract
(contract)-[:GRANTS_RIGHTS]->(rights_grant)

// Rights apply to territories (the key graph traversal)
(rights_grant)-[:COVERS_TERRITORY {
    is_included: true
}]->(territory)

// Territory group membership
(territory)-[:MEMBER_OF]->(territory_group)
(rights_grant)-[:COVERS_GROUP]->(territory_group)

// Sub-licensing chain (naturally a graph path)
(rights_grant)-[:SUB_LICENSED_TO {
    sub_license_id: 'uuid-sublicense-001',
    advance_amount: 25000.00,
    advance_currency: 'EUR',
    publisher_split_pct: 75.0,
    author_split_pct: 25.0,
    effective_date: date('2025-01-15'),
    expiration_date: date('2030-01-15'),
    status: 'active'
}]->(foreign_publisher)

// The sub-license creates a derivative rights grant
(sub_license_grant)-[:DERIVED_FROM]->(parent_grant)

// === Editorial Workflow Relationships ===

// Pipeline structure
(pipeline_template)-[:HAS_STAGE {order: 1}]->(stage1)
(pipeline_template)-[:HAS_STAGE {order: 2}]->(stage2)
(stage1)-[:FOLLOWED_BY]->(stage2)

// Work progress
(work)-[:IN_PIPELINE]->(pipeline_template)
(work)-[:AT_STAGE {
    entered_at: datetime(),
    status: 'in_progress',
    assigned_to: 'uuid-editor-001'
}]->(stage)

// Task assignment
(task)-[:FOR_STAGE]->(stage)
(task)-[:ASSIGNED_TO]->(person)
(task)-[:FOR_WORK]->(work)

// === Submission Relationships ===
(submission)-[:REVIEWED_BY {
    review_date: date('2026-03-20'),
    recommendation: 'request_full',
    notes: 'Strong voice. Request full.'
}]->(editor)

(submission)-[:ACCEPTED_AS]->(work)  // When submission becomes a work

// === Distribution Relationships ===
(product)-[:DISTRIBUTED_BY {
    territories: ['US', 'CA'],
    channel: 'wholesale',
    discount_schedule: 'standard_trade'
}]->(distributor)

// Agent representation
(person)-[:REPRESENTED_BY {
    representation_type: 'literary',
    start_date: date('2018-06-01'),
    territories: 'world',
    agency_name: 'C+W Agency'
}]->(agent)
```

---

## Key Graph Queries (Cypher)

### Rights Availability Check

This is the flagship query that justifies the graph model. In a relational database, determining rights availability requires joining contracts, rights_grants, rights_grant_territories, and sub_licenses with complex WHERE clauses. In Neo4j, it is a natural path traversal.

```cypher
// Find all available rights for a work in a specific territory
MATCH (w:Work {work_id: 'uuid-work-001'})
OPTIONAL MATCH (w)<-[:FOR_WORK]-(c:Contract)-[:GRANTS_RIGHTS]->(rg:RightsGrant)
                -[:COVERS_TERRITORY]->(t:Territory {code: 'DE'})
WHERE rg.status = 'active'
OPTIONAL MATCH (rg)-[sl:SUB_LICENSED_TO]->(fp:Organization)
RETURN w.title, rg.right_type, rg.is_exclusive, rg.grant_start, rg.grant_end,
       CASE WHEN sl IS NULL THEN 'available_for_sublicense' ELSE 'sub_licensed' END AS availability,
       fp.name AS licensee
ORDER BY rg.right_type
```

### Full Rights Chain Traversal

```cypher
// Trace the complete rights chain for a work across all territories
MATCH (w:Work {work_id: 'uuid-work-001'})
MATCH path = (w)<-[:FOR_WORK]-(c:Contract)-[:GRANTS_RIGHTS]->(rg:RightsGrant)
              -[:COVERS_TERRITORY|COVERS_GROUP*1..2]->(t)
WHERE rg.status IN ['active', 'sub_licensed']
OPTIONAL MATCH (rg)-[sl:SUB_LICENSED_TO]->(fp:Organization)
RETURN w.title,
       rg.right_type,
       rg.is_exclusive,
       CASE WHEN t:Country THEN t.code ELSE t.name END AS territory,
       sl.advance_amount,
       sl.publisher_split_pct,
       fp.name AS sub_licensee,
       rg.grant_end AS expiry
ORDER BY rg.right_type, territory
```

### Contributor Network Discovery

```cypher
// Find all people connected to a work and their roles
MATCH (w:Work {work_id: 'uuid-work-001'})<-[ct:CONTRIBUTED_TO]-(p:Person)
OPTIONAL MATCH (p)-[rep:REPRESENTED_BY]->(agent:Person:Agent)
RETURN p.display_name, ct.role, ct.contribution_pct,
       agent.display_name AS agent_name, rep.agency_name
ORDER BY ct.sequence_number
```

### Similar Works Discovery (for AI Acquisition Analysis)

```cypher
// Find works that share subjects, contributors, or series with a given work
MATCH (w:Work {work_id: 'uuid-work-001'})
MATCH (w)-[:HAS_EDITION]->(p:Product)-[:CLASSIFIED_AS]->(subj:Subject)
         <-[:CLASSIFIED_AS]-(p2:Product)<-[:HAS_EDITION]-(w2:Work)
WHERE w2 <> w
WITH w2, COUNT(DISTINCT subj) AS shared_subjects
OPTIONAL MATCH (w)-[:CONTRIBUTED_TO]-(shared_author)-[:CONTRIBUTED_TO]->(w2)
RETURN w2.title, shared_subjects,
       COLLECT(DISTINCT shared_author.display_name) AS common_contributors,
       w2.publication_status
ORDER BY shared_subjects DESC
LIMIT 20
```

### Reversion Alert Detection

```cypher
// Find rights grants approaching expiry or meeting reversion triggers
MATCH (rg:RightsGrant)
WHERE rg.status = 'active'
  AND (
    (rg.grant_end IS NOT NULL AND rg.grant_end < date() + duration({months: 6}))
    OR rg.reversion_trigger IS NOT NULL
  )
MATCH (rg)<-[:GRANTS_RIGHTS]-(c:Contract)-[:FOR_WORK]->(w:Work)
MATCH (c)-[:WITH_AUTHOR]->(author:Person)
OPTIONAL MATCH (rg)-[:COVERS_TERRITORY]->(t:Territory)
RETURN w.title, author.display_name, rg.right_type,
       COLLECT(t.code) AS territories,
       rg.grant_end, rg.reversion_trigger, c.contract_number
ORDER BY rg.grant_end ASC
```

### Editorial Pipeline Status Dashboard

```cypher
// All works currently in editorial pipeline with status
MATCH (w:Work)-[:AT_STAGE]->(stage:PipelineStage)
MATCH (w)<-[:FOR_WORK]-(c:Contract)-[:WITH_AUTHOR]->(author:Person)
OPTIONAL MATCH (stage)<-[:FOR_STAGE]-(task:Task)-[:ASSIGNED_TO]->(editor:Person)
WHERE task.status <> 'complete'
RETURN w.title, author.display_name, stage.stage_name, stage.stage_order,
       COLLECT({task: task.title, editor: editor.display_name, due: task.due_date, status: task.status}) AS open_tasks
ORDER BY stage.stage_order
```

### Sub-License Revenue Path

```cypher
// Trace revenue flow from sub-licenses back to original rights holder
MATCH path = (author:Person)<-[:WITH_AUTHOR]-(c:Contract)-[:GRANTS_RIGHTS]->(rg:RightsGrant)
              -[sl:SUB_LICENSED_TO]->(fp:Organization)
WHERE author.person_id = 'uuid-person-001'
RETURN author.display_name,
       c.contract_number,
       rg.right_type,
       sl.advance_amount, sl.advance_currency,
       sl.publisher_split_pct, sl.author_split_pct,
       fp.name AS licensee,
       sl.status
```

---

## PostgreSQL Financial Core

The financial side of the platform -- royalty calculations, sales data, payments -- remains in PostgreSQL for ACID compliance and complex aggregate queries.

```sql
-- Royalty periods
CREATE TABLE royalty_periods (
    period_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    period_name         VARCHAR(100) NOT NULL,
    period_start        DATE NOT NULL,
    period_end          DATE NOT NULL,
    period_type         VARCHAR(20) NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'open',
    finalized_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(period_start, period_end)
);

-- Sales import batches
CREATE TABLE sales_import_batches (
    batch_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    distributor_id      UUID NOT NULL,    -- References Neo4j Organization node
    file_name           VARCHAR(500),
    file_format         VARCHAR(50),
    period_id           UUID REFERENCES royalty_periods(period_id),
    row_count           INTEGER,
    total_units         INTEGER,
    total_gross         DECIMAL(14,2),
    status              VARCHAR(30) NOT NULL DEFAULT 'pending',
    error_log           TEXT,
    imported_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Sales transactions (partitioned by date)
CREATE TABLE sales_transactions (
    transaction_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id          UUID NOT NULL,      -- References Neo4j Product node
    period_id           UUID NOT NULL REFERENCES royalty_periods(period_id),
    distributor_id      UUID NOT NULL,      -- References Neo4j Organization node
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
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (sale_date);

CREATE INDEX idx_sales_product ON sales_transactions(product_id);
CREATE INDEX idx_sales_period ON sales_transactions(period_id);

-- Royalty calculations
CREATE TABLE royalty_calculations (
    calculation_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id         UUID NOT NULL,      -- References Neo4j Contract node
    period_id           UUID NOT NULL REFERENCES royalty_periods(period_id),
    product_id          UUID,               -- References Neo4j Product node
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
    calculation_details JSONB NOT NULL DEFAULT '{}',
    is_override         BOOLEAN NOT NULL DEFAULT FALSE,
    override_reason     TEXT,
    calculated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(contract_id, period_id, product_id)
);

-- Advance balances
CREATE TABLE advance_balances (
    balance_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id         UUID NOT NULL,      -- References Neo4j Contract node
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

-- Reserves against returns
CREATE TABLE reserves_against_returns (
    reserve_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id         UUID NOT NULL,
    period_id           UUID NOT NULL REFERENCES royalty_periods(period_id),
    reserve_pct         DECIMAL(5,2) NOT NULL,
    amount_held         DECIMAL(12,2) NOT NULL,
    amount_released     DECIMAL(12,2) NOT NULL DEFAULT 0,
    release_period_id   UUID REFERENCES royalty_periods(period_id),
    status              VARCHAR(30) NOT NULL DEFAULT 'held',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Royalty statements
CREATE TABLE royalty_statements (
    statement_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id         UUID NOT NULL,
    period_id           UUID NOT NULL REFERENCES royalty_periods(period_id),
    person_id           UUID NOT NULL,      -- References Neo4j Person node
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

-- Statement line items
CREATE TABLE royalty_statement_lines (
    line_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    statement_id        UUID NOT NULL REFERENCES royalty_statements(statement_id),
    product_id          UUID NOT NULL,
    line_type           VARCHAR(30) NOT NULL,
    description         VARCHAR(500),
    units               INTEGER,
    unit_price          DECIMAL(10,2),
    royalty_rate_pct    DECIMAL(6,3),
    amount              DECIMAL(12,2) NOT NULL,
    currency            CHAR(3) NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Exchange rates
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

-- Audit log (both databases feed into this for complete traceability)
CREATE TABLE audit_log (
    audit_id            BIGSERIAL PRIMARY KEY,
    source_database     VARCHAR(20) NOT NULL CHECK (source_database IN ('neo4j', 'postgresql')),
    entity_type         VARCHAR(100) NOT NULL,
    entity_id           VARCHAR(200) NOT NULL,
    action              VARCHAR(20) NOT NULL,
    old_values          JSONB,
    new_values          JSONB,
    changed_by          UUID,
    changed_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (changed_at);
```

---

## Synchronization Between Neo4j and PostgreSQL

The dual-database architecture requires careful synchronization. UUIDs are shared between both databases as the common identifier.

### Synchronization Strategy

```
Write Operations:
1. Application writes to the PRIMARY database for the entity type
   - Rights, Catalog, Contributors, Workflow → Neo4j first
   - Sales, Royalties, Payments → PostgreSQL first
2. Synchronization service propagates relevant IDs and summary data
   to the SECONDARY database

Read Operations:
- Rights queries → Neo4j
- Financial calculations → PostgreSQL
- Combined views (e.g., author portal) → Application joins data from both
```

### Change Data Capture (CDC)

```cypher
// Neo4j triggers (using APOC triggers or Neo4j Streams)
// When a new Product is created in Neo4j, publish to Kafka/RabbitMQ
// so that PostgreSQL sales_transactions can reference the product_id

// When a contract's rate schedule changes in Neo4j, notify the
// royalty calculation service to use updated rates
```

```sql
-- PostgreSQL side: listen for Neo4j entity changes
-- Use a sync_queue table for processing
CREATE TABLE sync_queue (
    sync_id         BIGSERIAL PRIMARY KEY,
    source          VARCHAR(20) NOT NULL,
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL,
    payload         JSONB,
    processed       BOOLEAN NOT NULL DEFAULT FALSE,
    processed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sync_queue_pending ON sync_queue(processed) WHERE processed = FALSE;
```

---

## Neo4j Constraints and Indexes

```cypher
// Uniqueness constraints
CREATE CONSTRAINT work_id_unique FOR (w:Work) REQUIRE w.work_id IS UNIQUE;
CREATE CONSTRAINT product_isbn_unique FOR (p:Product) REQUIRE p.isbn_13 IS UNIQUE;
CREATE CONSTRAINT product_id_unique FOR (p:Product) REQUIRE p.product_id IS UNIQUE;
CREATE CONSTRAINT person_id_unique FOR (p:Person) REQUIRE p.person_id IS UNIQUE;
CREATE CONSTRAINT org_id_unique FOR (o:Organization) REQUIRE o.organization_id IS UNIQUE;
CREATE CONSTRAINT contract_id_unique FOR (c:Contract) REQUIRE c.contract_id IS UNIQUE;
CREATE CONSTRAINT contract_number_unique FOR (c:Contract) REQUIRE c.contract_number IS UNIQUE;
CREATE CONSTRAINT territory_code_unique FOR (t:Territory) REQUIRE t.code IS UNIQUE;
CREATE CONSTRAINT subject_code_unique FOR (s:Subject) REQUIRE (s.scheme, s.code) IS UNIQUE;
CREATE CONSTRAINT grant_id_unique FOR (rg:RightsGrant) REQUIRE rg.grant_id IS UNIQUE;
CREATE CONSTRAINT series_id_unique FOR (s:Series) REQUIRE s.series_id IS UNIQUE;
CREATE CONSTRAINT submission_id_unique FOR (s:Submission) REQUIRE s.submission_id IS UNIQUE;
CREATE CONSTRAINT pipeline_template_id FOR (pt:PipelineTemplate) REQUIRE pt.template_id IS UNIQUE;
CREATE CONSTRAINT stage_id_unique FOR (s:PipelineStage) REQUIRE s.stage_id IS UNIQUE;

// Indexes for common lookups
CREATE INDEX work_title_index FOR (w:Work) ON (w.title);
CREATE INDEX work_status_index FOR (w:Work) ON (w.publication_status);
CREATE INDEX product_form_index FOR (p:Product) ON (p.product_form);
CREATE INDEX product_pub_date_index FOR (p:Product) ON (p.publication_date);
CREATE INDEX person_name_index FOR (p:Person) ON (p.display_name);
CREATE INDEX contract_status_index FOR (c:Contract) ON (c.status);
CREATE INDEX rights_status_index FOR (rg:RightsGrant) ON (rg.status);
CREATE INDEX rights_expiry_index FOR (rg:RightsGrant) ON (rg.grant_end);
CREATE INDEX submission_status_index FOR (s:Submission) ON (s.status);
CREATE INDEX territory_region_index FOR (t:Territory) ON (t.region);

// Full-text search index
CREATE FULLTEXT INDEX work_search FOR (w:Work) ON EACH [w.title, w.subtitle, w.synopsis];
CREATE FULLTEXT INDEX person_search FOR (p:Person) ON EACH [p.display_name, p.biography];
```

---

## Pros and Cons

### Pros

1. **Rights management becomes intuitive**: The core challenge of "who holds what rights, in which territory, through which contract chain, with what sub-licenses" is a natural graph traversal. No recursive CTEs, no multi-table joins, no temporary tables.
2. **Relationship-rich queries are fast**: Finding all works by contributors who share an agent, or all sub-licenses that chain through a specific rights grant, or all products distributed in a territory -- these are O(relationships) in a graph database, not O(table_size) as in relational joins.
3. **Flexible schema for catalog data**: Neo4j nodes can have arbitrary properties without schema migrations. Adding a new attribute to audiobook products does not affect hardback products.
4. **Work-to-work relationships**: Publishing has rich inter-work relationships (translations, abridgements, companion volumes, sequels, tie-in editions) that are awkward in relational models but natural in graphs.
5. **Contributor networks**: Finding "all works by authors represented by this agent" or "all editors who have worked with this author" is a single-hop traversal.
6. **Visual data exploration**: Neo4j's browser and Bloom visualization tools allow rights managers to visually explore the rights landscape for a work, seeing the full territory map and sub-license chain as a graph.
7. **AI/ML integration**: Graph embeddings from the catalog network can feed into the AI sales forecasting and manuscript triage features described in the README.
8. **ACID compliance where needed**: PostgreSQL handles all financial calculations with full transactional guarantees.

### Cons

1. **Operational complexity**: Running two databases (Neo4j + PostgreSQL) doubles the operational burden: separate backup strategies, monitoring, scaling, and failover configurations.
2. **Data synchronization challenges**: Keeping entity IDs and summary data consistent between Neo4j and PostgreSQL is a significant source of potential bugs and eventual consistency issues. Cross-database transactions are not possible.
3. **No foreign keys across databases**: A `contract_id` in PostgreSQL's `royalty_calculations` table cannot enforce referential integrity against the Neo4j Contract node. Application-layer validation is the only safeguard.
4. **Team skill requirements**: Neo4j and Cypher are less widely known than SQL. Hiring, training, and maintaining expertise in both databases increases organizational cost.
5. **Aggregate queries in Neo4j are weaker**: "Total revenue by territory for Q3 2025" requires PostgreSQL. Neo4j is not designed for the large-scale aggregation queries that royalty calculations demand.
6. **Licensing cost**: Neo4j Enterprise Edition (needed for production features like clustering, backup, and monitoring) carries significant licensing costs. The Community Edition has limitations.
7. **Write throughput**: Neo4j's write throughput is lower than PostgreSQL for bulk operations like importing millions of sales transactions.
8. **Query plan predictability**: Cypher query performance can be less predictable than SQL, especially for complex traversals with many optional matches. Query tuning requires Neo4j-specific expertise.
9. **Testing complexity**: Integration tests must exercise both databases and the synchronization layer, increasing test infrastructure requirements.

---

## Technology Recommendations

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| **Graph database** | Neo4j 5.x Enterprise | Production-grade graph database with ACID support, full-text search, and clustering |
| **Relational database** | PostgreSQL 16+ | Financial calculations, sales data, payments with full ACID compliance |
| **Synchronization** | Apache Kafka or Debezium CDC | Change Data Capture from both databases, published to a shared event stream |
| **Application framework** | Spring Boot (Java) or NestJS (Node.js) | Both have mature Neo4j drivers and PostgreSQL support |
| **Neo4j driver** | Official Neo4j Driver (Java/JavaScript/Python) | Reactive and imperative APIs, connection pooling, transaction management |
| **Graph visualization** | Neo4j Bloom or custom D3.js | Rights managers need visual exploration of territory/rights graphs |
| **Monitoring** | Neo4j Ops Manager + PostgreSQL pg_stat | Unified dashboard via Prometheus/Grafana |

### Alternative: AuraDB (Managed Neo4j)

For the SaaS deployment mode, Neo4j AuraDB provides a fully managed graph database service, eliminating operational overhead. This is recommended for the initial launch phase.

---

## Migration and Scaling Considerations

### Data Migration

1. **Reference data first**: Load Territory nodes (all ISO 3166-1 countries), Subject nodes (BISAC and THEMA codes), and Organization nodes.
2. **Catalog data**: Import Works, Products, and Contributors with their relationships.
3. **Contracts and Rights**: Import the rights graph with full territory and sub-license chains.
4. **Financial data to PostgreSQL**: Import sales history, royalty calculations, and advance balances.
5. **Verification**: Run parallel queries against both databases to verify consistency.

### Scaling Strategy

1. **Phase 1 (0-100 publishers)**: Single Neo4j instance + single PostgreSQL instance. Neo4j Community Edition may suffice if clustering is not needed.
2. **Phase 2 (100-500 publishers)**: Neo4j Enterprise with read replicas. PostgreSQL read replicas. Kafka for synchronization.
3. **Phase 3 (500+ publishers)**: Neo4j causal clustering for high availability. PostgreSQL with Citus or schema-per-tenant. Consider whether the graph model is still justified at this scale, or whether a fully relational model with materialized views for rights queries would be simpler to operate.

### When to Consider Simplification

If the platform's primary usage pattern is royalty calculation (write-heavy, aggregate-heavy) rather than rights exploration (traversal-heavy), the graph model may be over-engineered. The graph database is most valuable when:

- Rights managers spend significant time exploring territory availability
- Sub-licensing chains are deep (3+ levels)
- Contributor networks are a valuable discovery feature
- Work-to-work relationships (translations, adaptations) are a key navigation path

If the platform's users primarily interact through forms, reports, and dashboards (rather than graph exploration), a relational model with good indexing may serve equally well with less operational complexity.

### Cost Projection

| Component | Phase 1 (Cloud) | Phase 2 (Cloud) | Notes |
|-----------|-----------------|-----------------|-------|
| Neo4j AuraDB | ~$65/month (Professional) | ~$800/month (Enterprise) | Based on database size and queries |
| PostgreSQL (RDS) | ~$100/month (db.r6g.large) | ~$400/month (Multi-AZ) | Standard AWS RDS pricing |
| Kafka (MSK) | $0 (sync via polling) | ~$300/month (managed) | Only needed at scale |
| **Total infrastructure** | **~$165/month** | **~$1,500/month** | Excluding compute |

This is higher than a single-database approach (~$100-400/month total), which is the primary trade-off for the relationship query advantages.
