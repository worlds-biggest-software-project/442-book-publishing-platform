# Book Publishing Platform — Feature & Functionality Survey

> Candidate #442 · Researched: 2026-05-02

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Consonance | SaaS | Commercial (per-user subscription) | https://www.consonance.app |
| MetaComet | SaaS | Commercial | https://metacomet.com |
| Familiar | SaaS | Commercial (freemium) | https://familiar.pub |
| PublishDrive | SaaS | Commercial (distribution-focused) | https://publishdrive.com |
| ERPNext for Publishing | Open source / SaaS | MIT (self-host); commercial cloud | https://erpnext.com |

## Feature Analysis by Solution

### Consonance

**Core features**
- Web-based publishing management platform covering project management, contracts, rights, and royalties
- Editorial workflow: configurable stage pipeline (acquisition, editorial, copy-edit, proof, design, print-ready) with task assignment and deadline tracking per title
- Contract management: advance amounts, payment tranches triggered by editorial milestones, and recoupment tracking against earned royalties
- Rights tracking: territory and language rights grants, reversions, and sub-licencing revenue intake with expiry alerts
- Royalty calculation engine: per-format and per-channel rate tables with reserve-against-returns logic and period-based statements

**Differentiating features**
- True end-to-end platform combining editorial project management with financial royalty accounting — rare in the mid-market
- Sub-rights tracking: translation, audio, film, and digital rights managed within the same platform
- Author portal: self-service view of royalty statements, sales charts, and contract documents for authors

**UX patterns**
- Title management grid with configurable columns for editorial stage, publication date, and financial status
- Royalty statement generator producing author-ready PDFs for each accounting period
- Rights dashboard showing granted, available, and expired rights by territory and language

**Integration points**
- Distributor sales data import from Ingram, Nielsen, and Amazon KDP in standard formats
- REST API for integration with production workflows and external systems
- ISBN agency export for bulk registration

**Known gaps**
- Manuscript submission portal is basic; not designed for high-volume unsolicited submission intake
- AI-assisted content analysis or editorial workflow automation not yet available
- ISBN management and agency registration require manual steps

**Licence / IP notes**
- Proprietary SaaS. Per-user subscription model with no per-title fees at volume.

---

### MetaComet

**Core features**
- Royalty-focused software managing calculations and payments for over 300,000 book titles
- Refined over 25 years with major publishers; configurable royalty rule engine handling complex multi-format, multi-channel, and multi-territory rate structures
- Reserve-against-returns management: configurable reserve percentages by channel with automated release on schedule
- Sub-rights and co-publishing revenue intake with split calculations
- Advance recoupment tracking: unearned advance balances automatically offset against earned royalties before payment

**Differentiating features**
- Deepest royalty calculation engine in the market — handles edge cases (reserves, escalating rates, non-returnable sales, digital returns) that simpler tools cannot
- Battle-tested at major publisher scale: used by publishers managing hundreds of thousands of titles
- Royalty audit trail: immutable record of every calculation step supporting author audit rights under publishing agreements

**UX patterns**
- Royalty statement preview with line-item breakdown before generating final statements
- Override capability: publishers can manually adjust individual royalty line items with documented justification
- Contract record with rate table editor supporting escalating royalty rates by sales threshold

**Integration points**
- Distributor sell-through reports from Ingram, Baker & Taylor, Amazon, and major retailers via configurable import
- General ledger export to accounting systems for royalty liability posting
- Author portal access (optional) for statement self-service

**Known gaps**
- No editorial project management, rights tracking, or submission management — royalty-only focus
- UI is functional but dated; not designed for modern web expectations
- Not suitable for self-publishers or very small independent publishers; minimum viable use case requires meaningful title volume

**Licence / IP notes**
- Proprietary SaaS. No open-source components. 25-year royalty engine expertise is a competitive moat.

---

### Familiar

**Core features**
- All-in-one platform covering royalties, contracts, submissions, and editorial analysis
- Freemium entry point: first 10 books free, then USD 1 per book per month — accessible pricing for small and independent publishers
- Manuscript submission portal: online intake with metadata capture, automated acknowledgement, and slush-pile queue management
- Contract generation: templates with configurable advance amounts, royalty rates, and territorial grants
- Sales analytics: title performance charts with per-channel breakdown

**Differentiating features**
- Lowest price point in the market for a multi-feature publishing platform
- Submission management alongside royalties and contracts is unusually complete for an affordable tool
- Analytics module identifying which acquisition sources and editorial categories perform best commercially

**UX patterns**
- Simple dashboard showing active submissions, upcoming royalty statement dates, and contract renewal alerts
- One-click royalty statement generation from uploaded sales data
- Submission reader interface for editorial triage of incoming manuscripts

**Integration points**
- CSV import for distributor and retailer sales data
- Email notifications for submission acknowledgements and royalty statement delivery
- Stripe for author advance payment processing

**Known gaps**
- Royalty calculation sophistication is limited compared with MetaComet; not suitable for major publisher complexity
- Rights tracking for international sub-rights is basic
- No ISBN management or agency registration workflow

**Licence / IP notes**
- Proprietary SaaS. Freemium model with per-book pricing above free tier threshold.

---

### ERPNext for Publishing

**Core features**
- Open-source ERP adapted for publishing workflows including manuscript submission, editorial review, contract generation, and royalty schedules
- Fully self-hostable with complete source code available under MIT licence
- Publishing-specific customisations: title master, author record, royalty schedule linked to contract, and ISBN assignment
- Integrated accounting module: royalty payments posted directly to the general ledger without a separate finance system
- Manufacturing module adaptable for print-run management and production cost tracking

**Differentiating features**
- Complete data sovereignty: all publishing and financial data hosted by the publisher organisation
- No per-user or per-title fees beyond hosting costs; significant cost advantage at scale
- Full ERP capabilities available alongside publishing-specific modules: HR, procurement, inventory, and project management

**UX patterns**
- Form-based UI consistent with ERPNext's general-purpose design
- Workflow state machine for configurable editorial pipeline stages
- Reporting module generating standard financial and operational reports

**Integration points**
- Standard Python/REST API for integration with any external system
- ISBN agency integration via custom development
- Payment gateways for author advance disbursement

**Known gaps**
- Requires significant technical customisation to configure publishing-specific workflows; not plug-and-play
- No purpose-built author portal; would require custom development
- Statement generation and royalty calculation sophistication requires customisation to match dedicated tools like MetaComet

**Licence / IP notes**
- ERPNext core: MIT licence — fully permissive for commercial use. No copyleft obligations for the application layer.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Title management database covering all formats (hardback, paperback, ebook, audiobook) with publication metadata
- Royalty calculation engine: per-format and per-channel rate tables with reserve-against-returns logic
- Distributor sales data import from Ingram, Amazon KDP, and major retail chains
- Advance recoupment tracking: unearned balance automatically offset against earned royalties
- Contract storage with advance payment tranches, royalty rates, and territorial grants
- Author royalty statements: period-based PDF generation with line-item breakdown

### Differentiating Features
- Sub-rights management: tracking translation, audio, film, and digital rights alongside primary publishing rights
- Manuscript submission portal for managing unsolicited submissions at volume
- Author self-service portal for statement and contract document access
- Editorial project management pipeline integrated with financial royalty accounting
- Open-source self-hosting option providing complete data sovereignty (ERPNext)

### Underserved Areas / Opportunities
- Modern cloud-native platform combining editorial workflow, ISBN management, sub-rights tracking, and royalty accounting in a single affordable subscription for independent and mid-size publishers
- Self-publishing and hybrid publishing tools at scale: managing thousands of titles from individual authors with lightweight per-title pricing
- AI-assisted acquisitions: comparing submitted manuscript signals against commercial performance of comparable published titles
- ONIX data management: automated generation and distribution of ONIX product information to retailers and libraries

### AI-Augmentation Candidates
- Manuscript quality scoring: AI assessment of submitted manuscripts against editorial criteria for initial triage
- Sales forecast modelling: predicting first-year sales based on comparable title performance, author track record, and category trends
- Royalty anomaly detection: flagging distributor statements that appear inconsistent with streaming and sales data from other sources
- Contract clause analysis: NLP review of received sub-rights offers identifying unusual or unfavourable terms before negotiation

## Legal & IP Summary

Book publishing royalty calculation methodology is based on publicly documented accounting practices (reserve-against-returns, advance recoupment, escalating royalty rates) with no patent encumbrances. ERPNext (MIT licence) provides a legally clear open-source foundation for a self-hosted publishing platform. ONIX for Books (XML standard for bibliographic data) is published by EDItEUR under open terms and freely implementable. ISBN assignment requires a fee-based agreement with the national ISBN agency (ISBN.org or Nielsen BookData); the assignment process itself is straightforward. Distributor sales data arrives in proprietary CSV formats; independently implementing parsers for each major distributor is required. Author royalty statements create contractual obligations under the publishing agreement; calculation errors create liability to authors — robust audit logging and an override-with-justification capability are essential for legal defensibility.

## Recommended Feature Scope

**Must-have (MVP)**:
- Title database covering all formats with publication metadata, ISBN assignment, and editorial stage tracking
- Royalty calculation engine: per-format, per-channel rate tables with reserve-against-returns and advance recoupment
- Distributor sales data import for Ingram, Amazon KDP, and major retail chain CSV formats
- Contract management: advance amounts, payment tranches, royalty rate schedules, and territorial rights grants
- Author royalty statements: period-based PDF generation with line-item breakdown and audit trail
- Basic rights register: territory and language rights grants with expiry date alerts

**Should-have (v1.1)**:
- Manuscript submission portal: online intake with metadata capture, automated acknowledgement, and editorial triage queue
- Author self-service portal: royalty statement access, sales charts, and contract document viewing
- Sub-rights management: translation, audio, film, and digital rights with sub-licencing revenue intake
- Editorial workflow: configurable pipeline stages with task assignment and deadline tracking per title
- ONIX product data generation and distribution to retailers and libraries

**Nice-to-have (backlog)**:
- AI manuscript triage scoring against editorial criteria for initial acquisition assessment
- Sales forecasting model for new acquisitions based on comparable title and category performance
- AI-powered royalty anomaly detection comparing distributor statements against aggregated streaming data
- Print-run management module tracking physical production costs and inventory
- Hybrid publishing / self-publishing mode with per-title pricing for author-direct use cases
