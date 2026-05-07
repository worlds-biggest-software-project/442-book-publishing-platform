# Book Publishing Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> A unified, cloud-native platform that replaces the patchwork of disconnected tools publishers use for manuscript intake, editorial production, ISBN management, rights tracking, and royalty accounting.

Book publishers -- from small independents to mid-size trade houses -- typically stitch together separate systems for submissions, editorial workflow, contracts, ISBN registration, and royalty payments. This fragmentation creates data silos, manual re-keying, and royalty calculation errors that damage author relationships and create financial liability. Book Publishing Platform consolidates these functions into a single system designed for the roughly 80% of publishing operations that are too small for enterprise solutions like Klopotek but too complex for spreadsheets.

---

## Why Book Publishing Platform?

- **Fragmented incumbent landscape.** Existing tools specialise in one slice of publishing operations. MetaComet handles royalties but has no editorial workflow or rights tracking. Consonance covers more ground but lacks robust manuscript submission intake and AI-assisted capabilities. No single affordable tool covers the full editorial-to-royalty lifecycle.
- **Enterprise pricing locks out independents.** Full-featured publishing ERPs are priced for the Big Five publishers and require lengthy implementation cycles. Independent and mid-size publishers -- the majority of the market by count -- cannot justify these costs.
- **Royalty complexity breeds errors.** Royalty calculations involve per-format rates, per-channel splits, reserve-against-returns logic, advance recoupment, escalating rate thresholds, and multi-currency sub-rights revenue. Spreadsheet-based approaches and limited tools produce calculation errors that trigger author disputes and audit exposure.
- **Dated user experiences.** Established players like MetaComet have 25 years of domain expertise but UIs that do not meet modern web expectations, creating friction for editorial and finance teams.
- **No open-source end-to-end option.** ERPNext offers an MIT-licensed foundation but requires significant custom development to function as a publishing platform. There is no purpose-built open-source alternative covering editorial workflow, ISBN management, rights, and royalties together.

---

## Key Features

### Title & ISBN Management

- Title database covering all formats (hardback, paperback, ebook, audiobook) with publication metadata
- ISBN allocation from registered prefixes with format-level assignment
- Export to national ISBN agencies (Nielsen, Bowker) for bulk registration
- ONIX product data generation and distribution to retailers and libraries

### Editorial Workflow

- Configurable stage pipeline: acquisition review, editorial pass, copy-edit, proofread, design, print-ready
- Task assignment and deadline tracking per title
- Manuscript submission portal with metadata capture, automated acknowledgement, and slush-pile queue management
- Editorial triage interface for assessing incoming manuscripts at volume

### Contracts & Rights

- Contract storage with advance amounts, payment tranches triggered by editorial milestones, and royalty rate schedules
- Territory and language rights grants with expiry alerts and reversion tracking
- Sub-rights management for translation, audio, film, and digital rights
- Sub-licencing revenue intake with split calculations

### Royalty Calculation Engine

- Per-format and per-channel rate tables with escalating royalty rates by sales threshold
- Reserve-against-returns logic with configurable reserve percentages and automated release schedules
- Advance recoupment: unearned balances automatically offset against earned royalties before payment
- Period-based royalty statements with line-item breakdown and full audit trail
- Override capability with documented justification for legal defensibility

### Sales Data & Author Portal

- Distributor sell-through report import from Ingram, Baker & Taylor, Amazon KDP, and major retailers via configurable parsers
- Author self-service portal for royalty statement access, sales charts, and contract document viewing
- Sales analytics with per-channel breakdown and title performance tracking

---

## AI-Native Advantage

AI capabilities address pain points that incumbents have not yet tackled. Manuscript quality scoring provides initial triage of unsolicited submissions against editorial criteria, reducing the time editors spend on the slush pile. Sales forecast modelling predicts first-year performance for acquisition candidates based on comparable title data, author track records, and category trends. Royalty anomaly detection flags distributor statements that appear inconsistent with data from other sources, catching errors before they become author disputes. Contract clause analysis uses NLP to review incoming sub-rights offers and identify unusual or unfavourable terms before negotiation begins.

---

## Tech Stack & Deployment

- **Deployment modes:** Cloud-hosted SaaS with a self-hosted option for publishers requiring complete data sovereignty
- **Royalty engine:** Rules-based calculation engine with per-contract parameterisation and immutable audit logging, avoiding hard-coded logic
- **Sales data ingestion:** Configurable ETL layer for parsing inconsistent distributor CSV formats from Ingram, Amazon KDP, Baker & Taylor, and retail chains
- **Standards:** ONIX for Books (EDItEUR XML standard for bibliographic data, freely implementable); ISBN assignment via national agency APIs
- **Integration:** REST API for connection to external production workflows, accounting systems (general ledger export), and payment gateways
- **Data model:** Designed for long-lived publishing titles (decades) with support for evolving rate structures and ownership changes without losing historical period data
- **Multi-currency:** Required for international sub-rights deals and foreign-territory royalty accounting

---

## Market Context

The global book publishing market generates revenues exceeding USD 120 billion annually, yet the software tools serving it remain fragmented and often dated. Independent and mid-size publishers -- roughly 80% of all publishing operations by count -- are the most underserved segment. Incumbent pricing ranges from MetaComet's enterprise-scale royalty software (not viable for small publishers) to Familiar's USD 1 per book per month freemium model (limited royalty sophistication). The growth of self-publishing and hybrid publishing models is creating additional demand for lightweight royalty and distribution tools that scale from single-author operations to catalogues of thousands of titles.

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

Licence to be determined. See [discussion](#) for context.
