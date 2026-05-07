# Standards & API Reference

> Project: Book Publishing Platform · Generated: 2026-05-07

## Industry Standards & Specifications

### ISO Standards

**ISO 2108 — International Standard Book Number (ISBN)**
- URL: https://www.isbn-international.org/
- The globally mandated identifier for books and distinct book formats. Every format variant (hardcover, paperback, EPUB, PDF, audiobook) requires a separate ISBN. Since 2007 the standard uses 13-digit identifiers (EAN-13 compatible). Integration with national agencies (ISBN.org in the US, Nielsen in the UK) is required for batch registration. The International ISBN Agency publishes assignment guidelines for e-books at https://www.isbn-international.org/content/guidelines-assignment-e-books/26.

**ISO/IEC 23078-2:2024 — Readium LCP (Licensed Content Protection)**
- URL: https://readium.org/lcp-specs/
- The only ebook DRM technology adopted as an ISO International Standard. Developed by EDRLab (European Digital Reading Lab) as a vendor-neutral, interoperable, distributed DRM solution for EPUB 2/3 and PDF. A book publishing platform that delivers protected digital content must support LCP or the older Adobe ADEPT system. Readium LCP is open source and increasingly preferred for its privacy-respecting design.

**ISO 14289 — PDF/UA (Universal Accessibility)**
- URL: https://www.iso.org/standard/64599.html
- Provides detailed technical requirements for creating accessible PDF files. Directly complementary to WCAG: WCAG governs how users access content, PDF/UA governs how they experience it inside the document. Relevant for any platform that generates or hosts PDF editions of books, particularly under EU accessibility obligations.

### W3C & IETF Standards

**EPUB 3.3 — W3C Recommendation (updated May 2025)**
- URL: https://www.w3.org/TR/epub-33/
- The authoritative standard for reflowable digital publications. EPUB 3.3 is based on HTML5/CSS3 and supports embedded media, scripting, and accessibility metadata. As of 2025 it is the required format for ebook distribution on all major platforms. EPUB Accessibility 1.1 is now an integral part of the EPUB 3.3 Recommendation — accessibility is no longer optional.

**EPUB Accessibility 1.1 — W3C Recommendation**
- URL: https://www.w3.org/TR/epub-a11y-11/
- Specifies discoverability and certification requirements for accessible EPUB publications. Mandates inclusion of Schema.org accessibility metadata and conformance to WCAG 2.1 AA. Directly mapped to EU Accessibility Act (EAA) requirements; any ebook sold in the EU after June 2025 must comply.

**EPUB 3.4 — W3C Working Draft (in development, chartered to February 2027)**
- URL: https://www.w3.org/TR/2025/WD-epub-34-20250327/
- The next revision of the EPUB standard, began February 2025. Platforms should monitor this working draft; implementations built now should be designed to migrate forward.

**EPUB Accessibility — EU Accessibility Act Mapping**
- URL: https://www.w3.org/TR/epub-a11y-eaa-mapping/
- W3C working note mapping EPUB Accessibility 1.1 requirements to European Accessibility Act obligations. Essential compliance reference for any publishing platform targeting EU markets.

**WCAG 2.2 — Web Content Accessibility Guidelines**
- URL: https://www.w3.org/TR/WCAG22/
- Current W3C Recommendation (October 2023). Level AA is the baseline compliance target for digital publishing platforms in the EU and US. Applies to platform UI, author portals, and any web-based reading interfaces. The EAA enforcement (June 2025) cites WCAG 2.1 AA as the minimum.

**WAI-ARIA 1.2 — Accessible Rich Internet Applications**
- URL: https://www.w3.org/TR/wai-aria-1.2/
- W3C Recommendation providing roles, states, and properties that make dynamic web content accessible to assistive technologies. Directly relevant to author dashboards, royalty portals, and any interactive editorial workflow UI.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- De-facto standard for delegated authorization. Required for third-party integrations (distributor feeds, payment providers, ISBN agencies). Lulu's Print API uses OpenID Connect (built on OAuth 2.0) for authentication.

**RFC 7519 — JSON Web Tokens (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Widely used token format for API authentication and inter-service authorization in publishing platform APIs.

### Data Model & API Specifications

**ONIX for Books 3.1 (EDItEUR)**
- URL: https://www.editeur.org/12/About-Release-3.0-and-3.1/
- The international XML standard for communicating book product information between publishers, aggregators, and retailers. Covers 200+ data elements: ISBN, title, contributors, descriptions, prices, territorial rights, availability, and — as of ONIX 3 — accessibility metadata. Amazon set a March 2026 deadline for full ONIX 3 compliance. The specification and schemas (RNG, XSD, DTD) are freely available from EDItEUR.

**Schema.org — Book Type**
- URL: https://schema.org/Book
- The W3C-endorsed vocabulary for structured data markup. The `Book` type and its properties (isbn, author, publisher, copyrightYear, numberOfPages, accessibilityFeature) are the standard for embedding book metadata in HTML pages and REST API responses as JSON-LD. Essential for SEO and interoperability with Google, library catalogues, and aggregators.

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- The industry standard for describing RESTful APIs. Any publishing platform exposing a developer API should publish an OpenAPI 3.1 document. Enables automatic SDK generation, interactive documentation, and partner integrations.

**BISAC Subject Headings (Book Industry Study Group — BISG)**
- URL: https://www.bisg.org/BISAC-Subject-Codes-main
- The US standard for topical categorisation of books: 54 major sections with detailed sub-categories, each a 9-character alphanumeric code. Required by all major distributors (IngramSpark, Amazon KDP) for catalogue ingestion. Free to use; updated annually by BISG.

**MARC 21 / MARC XML**
- URL: https://www.loc.gov/marc/
- The Machine-Readable Cataloguing standard used by libraries globally. Relevant for any platform that integrates with library channels (OverDrive, BiblioBoard) or exports catalogue data to OCLC WorldCat.

### Security & Authentication Standards

**OAuth 2.0 / OpenID Connect (OIDC)**
- URL: https://openid.net/connect/
- OIDC (built on OAuth 2.0) is the authentication standard used by major publishing APIs including Lulu Print API. Required for author portal login, third-party integrations, and any SSO functionality.

**OWASP Top Ten**
- URL: https://owasp.org/www-project-top-ten/
- The foundational web application security checklist. Particularly relevant for an author portal handling contracts, royalty statements, and personal financial data. SQL injection, broken access control, and insecure deserialization are the highest-risk categories.

**GDPR (Regulation (EU) 2016/679)**
- URL: https://gdpr-info.eu/
- European data protection regulation. Any platform handling EU author or reader personal data must implement lawful basis for processing, consent mechanisms, right-to-erasure flows, and data processing agreements with sub-processors. Cumulative GDPR fines reached €5.88 billion as of 2025. The EDPB's 2026 coordinated action focuses on Articles 12–14 transparency obligations.

**PCI DSS v4.0**
- URL: https://www.pcisecuritystandards.org/
- Payment Card Industry Data Security Standard. Applies if the platform processes card payments directly; using Stripe Connect or similar tokenised payment providers reduces scope significantly but does not eliminate compliance obligations.

### MCP Server Specifications

Model Context Protocol (MCP) is relevant to this project for AI-native features such as intelligent royalty calculation, manuscript analysis, rights suggestion, and metadata enrichment.

**Model Context Protocol**
- URL: https://modelcontextprotocol.io/
- Anthropic's open standard for connecting AI models to external data sources and tools. Publishing platforms can expose MCP servers to allow AI agents to query book metadata, generate royalty reports, draft contract language, and interact with editorial workflow states. The protocol uses JSON-RPC 2.0 over HTTP/SSE or STDIO transports.

---

## Similar Products — Developer Documentation & APIs

### Lulu Print API

- **Description:** Lulu is a print-on-demand and global fulfilment platform. Its API enables programmatic ordering of physical books (hardcover, paperback, coil-bound) and handles printing, shipping, and tracking worldwide.
- **API Documentation:** https://api.lulu.com/docs/
- **Developer Portal:** https://developers.lulu.com/
- **SDKs/Libraries:** Community client available at https://github.com/minireference/lulu-api-client (Python); official SDKs not published at time of research.
- **Developer Guide:** https://blog.lulu.com/building-your-book-business-with-apis/
- **Standards:** REST/JSON; webhooks for order status and shipping events; 27-character Pod Package ID (SKU) encodes trim size, colour, binding, paper, and finish.
- **Authentication:** OpenID Connect (OAuth 2.0). Both production and sandbox environments available.

### PublishDrive

- **Description:** An ebook and audiobook distribution platform serving 400+ stores. Offers a publisher-facing API for automated book uploads, metadata management, sales reporting, and royalty tracking.
- **API Documentation:** https://publishdrive.com/software-for-book-publishers.html (API access via enterprise/publisher tiers)
- **SDKs/Libraries:** Not publicly documented; contact required for API credentials.
- **Developer Guide:** Described in platform documentation; supports BISAC codes and ONIX metadata ingestion.
- **Standards:** REST/JSON; ONIX-compatible metadata; distribution covers Kindle, Apple Books, Google Play, Kobo, OverDrive.
- **Authentication:** API key / OAuth; details available on request.

### Amazon Kindle Direct Publishing (KDP)

- **Description:** Amazon's self-publishing platform for ebooks and print-on-demand paperbacks. Dominant in US and UK markets.
- **API Documentation:** No official public API for publishing or royalty data. Amazon Selling Partner API (SP-API) does not cover KDP content or royalties.
- **SDKs/Libraries:** No official SDK. Third-party tools (e.g., KDP Wizard via Airtable) fill the gap.
- **Developer Guide:** https://kdp.amazon.com/en_US/ (manual web interface only)
- **Standards:** Accepts EPUB 3 and DOCX; ONIX-compatible metadata for distribution side. AZW/KFX proprietary format for delivery.
- **Authentication:** N/A — no developer API available.
- **Note:** The absence of a KDP API is a documented platform gap. Sales data ingestion for royalty calculation must rely on CSV export or screen-scraping workarounds.

### IngramSpark / Ingram Content Group

- **Description:** The world's largest book distributor, operating IngramSpark for independent publishers and Lightning Source for larger houses. Provides print-on-demand and access to 39,000+ wholesale channels globally.
- **API Documentation:** https://www.ingramspark.com/hubfs/downloads/user-guide.pdf (IngramSpark User Guide v3.0, August 2025); no public REST API documented.
- **SDKs/Libraries:** None publicly available.
- **Developer Guide:** EDI-based integration for larger publishers via Ingram iPage; direct API not available for IngramSpark tier.
- **Standards:** ONIX for metadata submission; distribution reports in proprietary CSV formats; EPUB 3 and PDF for content.
- **Authentication:** Account-based; no OAuth API.

### Stripe Connect

- **Description:** Stripe's multi-party payments solution. Used by publishing platforms and marketplaces to route author royalty payouts, handle advance disbursements, and manage platform fees.
- **API Documentation:** https://docs.stripe.com/connect
- **SDKs/Libraries:** Official SDKs for JavaScript/Node.js, Python, Ruby, PHP, Java, Go, .NET — https://docs.stripe.com/development
- **Developer Guide:** https://docs.stripe.com/connect/end-to-end-marketplace
- **Standards:** REST/JSON; OpenAPI-described; 500+ endpoints across v1 and v2 namespaces; supports 135+ currencies and 40+ payment methods.
- **Authentication:** API keys; OAuth 2.0 for connected account onboarding.

### Draft2Digital

- **Description:** An ebook aggregator and distributor (owner of legacy Smashwords) offering wide distribution to Apple Books, Barnes & Noble, Kobo, libraries, and more. Author-facing; no public developer API documented.
- **API Documentation:** Not publicly available. Smashwords distribution migrated to Draft2Digital in 2025; API access not listed.
- **SDKs/Libraries:** None documented.
- **Developer Guide:** https://draft2digital.com/
- **Standards:** Accepts EPUB and DOCX; BISAC/ONIX metadata on submission.
- **Authentication:** N/A (no developer API available).

### ISBN International / Bowker / Nielsen

- **Description:** National ISBN agencies that issue publisher prefixes and accept batch ISBN registration. Bowker (US) and Nielsen (UK) are the primary registrars.
- **API Documentation:**
  - Bowker: https://www.myidentifiers.com/ (no public REST API; web-based registration only)
  - Nielsen: https://www.nielsenisbnstore.com/ (EDI/FTP batch delivery for large publishers)
- **SDKs/Libraries:** None publicly available.
- **Standards:** ISBN-13 (EAN-13 compatible); ONIX for metadata submission; national agency integrations are typically bespoke.
- **Authentication:** Account-based; no OAuth.

### EditionGuard / Readium LCP

- **Description:** EditionGuard is a hosted DRM-as-a-service platform implementing both Readium LCP and Adobe ADEPT DRM for ebook protection and fulfilment.
- **API Documentation:** https://www.editionguard.com/ (API documentation available after account registration)
- **SDKs/Libraries:** Readium open-source SDK: https://readium.org/
- **Developer Guide:** https://readium.org/lcp-specs/
- **Standards:** ISO/IEC 23078-2:2024 (Readium LCP); EPUB 3; REST/JSON API for licence issuance and revocation.
- **Authentication:** API key.

---

## Notes

**ONIX 3 migration pressure:** Amazon's March 2026 deadline for ONIX 3 compliance, combined with the EU Accessibility Act's June 2025 enforcement date, means any new book publishing platform must implement ONIX 3.1 from day one. ONIX 2.1 is end-of-life for practical purposes.

**Lack of public distributor APIs:** Unlike music streaming (Spotify, Apple Music with OAuth APIs), book distribution remains predominantly batch/EDI based. IngramSpark, Draft2Digital, and KDP do not expose developer APIs. This is a structural gap that an AI-native publishing platform could address by building a normalised, multi-source sales data ingestion layer.

**EU Accessibility Act (EAA) compliance is now mandatory:** The EAA has been in force since June 2025. All new ebooks sold in the EU must embed EPUB Accessibility 1.1 metadata and meet WCAG 2.1 AA criteria. Any platform generating or distributing EPUB files must automate accessibility validation.

**Schema.org JSON-LD for interoperability:** Structuring all book metadata as Schema.org JSON-LD (Book type) maximises interoperability with Google, library catalogues, and downstream aggregators without requiring proprietary format converters.
