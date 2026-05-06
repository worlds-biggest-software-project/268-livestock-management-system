# Standards & API Reference

> Project: Livestock Management System · Generated: 2026-05-04

## Industry Standards & Specifications

### ISO Standards

**ISO 11784 — Radio-Frequency Identification of Animals: Code Structure**
- URL: https://www.iso.org/standard/25881.html
- Defines the 64-bit code structure for RFID transponders used in animal identification, comprising a country code (ISO 3166), an animal flag, and a 12-digit national ID number. Mandatory for any livestock software that reads or writes EID ear-tag data.

**ISO 11785 — Radio-Frequency Identification of Animals: Technical Concept**
- URL: https://www.iso.org/standard/25882.html
- Specifies how transponders are activated and how data is transferred to a reader, including the 134.2 kHz activation frequency and ASK/FSK modulation protocols. Implementations integrating with RFID readers must conform to this standard.

**ISO 24631-5:2014 — Procedure for Testing RFID Transceivers**
- URL: https://www.iso.org/standard/53531.html
- Defines test procedures for verifying that RFID transceivers correctly read ISO 11784/11785 transponders. Relevant when certifying integrations with ear-tag readers.

**ISO 11788-1/2/3 — Electronic Data Interchange between Information Systems in Agriculture: Agricultural Data Element Dictionary**
- URLs: https://www.iso.org/standard/19984.html (Part 1), https://www.iso.org/standard/26400.html (Part 2, dairy), https://www.iso.org/standard/26401.html (Part 3, pig farming)
- Specifies how on-farm management computers exchange structured data. Part 2 covers dairy farming and Part 3 covers pig farming, both directly relevant to livestock management record interchange.

**ISO 22005:2007 — Traceability in the Feed and Food Chain**
- URL: https://www.iso.org/standard/36297.html
- Gives general principles and basic requirements for designing a traceability system that spans feed and food production chains. Essential for livestock software that supports end-to-end supply chain traceability from farm to slaughter.

### EU Regulatory Standards

**EU Regulation (EC) No 1760/2000 — Bovine Animal Identification and Labelling of Beef**
- URL: https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32000R1760
- Establishes the EU system for identifying and registering bovine animals via double ear tags, holding registers, cattle passports, and a computerised national database. Any cattle management software operating in the EU must support the data elements and reporting requirements of this regulation.

**EU Regulation (EU) No 653/2014 — Electronic Identification of Bovine Animals**
- URL: https://www.legislation.gov.uk/eur/2014/653/introduction
- Amends Regulation 1760/2000 to introduce optional electronic identification (EID) for bovine animals and updates beef labelling rules. Livestock software must be capable of recording and reporting EID data for EU-based producers who adopt electronic tagging.

**EU Regulation (EC) No 21/2004 — Identification and Registration of Ovine and Caprine Animals**
- Mandates electronic identification (RFID, ISO 11784/11785 compliant) for sheep and goats across the EU from January 2010. Any multi-species platform targeting EU flocks must support EID recording and movement reporting under this regulation.

### W3C & IETF Standards

**RFC 7231 — Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content**
- URL: https://www.rfc-editor.org/rfc/rfc7231
- Defines the semantics of HTTP methods (GET, POST, PUT, DELETE, PATCH) used in RESTful APIs. All livestock management APIs reviewed (ICAR ADE, Farmbrite, LIS UK) implement REST over HTTP.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- Defines the OAuth 2.0 framework for delegated API authorisation. The UK Livestock Information Service and other government-linked livestock APIs use OAuth 2.0 (via Azure B2C) to authenticate third-party software integrators.

**RFC 8288 — Web Linking**
- URL: https://www.rfc-editor.org/rfc/rfc8288
- Defines the `Link` header for pagination and resource relationships in REST APIs. Relevant for livestock data APIs that return large paged result sets (herd listings, movement records).

### Data Model & API Specifications

**ICAR Animal Data Exchange (ADE) Standard**
- URL: https://github.com/adewg/ICAR
- Licence: Apache 2.0
- The primary open standard for JSON-based exchange of individual animal data in livestock. Produced by the International Committee for Animal Recording (ICAR) Animal Data Exchange Working Group. Version 1.2 covers dairy cows, beef cattle, sheep, farmed buffalo, and deer. Defines REST endpoints, JSON schemas, and data models for animal identification, health events, reproduction, milk recording, weighing, and feeding. Already adopted by robotic milking vendors, farm software providers, and national milk recording organisations. This is the foundational interoperability specification for any livestock platform targeting multi-vendor data exchange.

**GS1 EPCIS 2.0 — Electronic Product Code Information Services**
- URL: https://www.gs1.org/standards/epcis
- Developer reference: https://openepcis.io/
- Defines event-based supply chain visibility across the food chain ("what, when, where, why, how"). Version 2.0 introduces a REST/JSON-LD Web API alongside the legacy SOAP/XML interface. Applicable to livestock traceability from property-of-birth through to slaughter and processing. The open-source OpenEPCIS implementation provides a fully compliant reference server.

**GS1 Global Traceability Standard (GTS)**
- URL: https://www.gs1.org/standards/gs1-global-traceability-standard/current-standard
- Provides a framework for implementing end-to-end traceability combining GS1 identifiers (GTIN, GLN, SSCC) with EPCIS events. Supports hybrid implementations that combine GS1 standards with regulatory livestock IDs (e.g., NLIS, BCMS).

**OpenAPI Specification (OAS) 3.x**
- URL: https://www.openapis.org/
- The industry standard for describing REST APIs. Farmbrite publishes an OpenAPI specification for their livestock API. The ICAR ADE standard and UK LIS Developer Hub also provide OpenAPI-compatible specifications. Any new livestock management platform should publish an OAS 3.x definition for third-party integration.

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) and OpenID Connect 1.0**
- URL: https://openid.net/connect/
- OAuth 2.0 is the base authorisation framework; OpenID Connect adds identity federation on top. The UK Livestock Information Service uses Azure B2C (OAuth 2.0/OIDC) to authenticate third-party farm software integrators. Any livestock platform with a developer API should implement OAuth 2.0 with PKCE for user-delegated access.

**GDPR (EU) 2016/679 — General Data Protection Regulation**
- URL: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A32016R0679
- Applies where livestock management data can be linked to an identifiable individual (farm operator, employee). EU-deployed platforms must address lawful basis for data processing, data minimisation, portability, and right to erasure. Geo-located farm data and records tied to a registered holding may constitute personal data in some jurisdictions.

**EU Code of Conduct on Agricultural Data Sharing (2018)**
- URL: https://copa-cogeca.eu/app/uploads/2020/11/Code_of_conduct_agricultural_data.pdf
- Industry voluntary framework governing how farm data collected by third-party software and machinery vendors may be used, shared, and transferred. Livestock management platforms should align with its principles on data portability and transparency.

### MCP Server Specifications

**Leaf Agriculture MCP Server — AI-Ready Farm Data**
- URL: https://withleaf.io/en/whats-new/leaf-mcp-launch/
- Leaf Agriculture has launched an MCP (Model Context Protocol) server enabling AI agents to query unified farm data via natural language. This is directly relevant to an AI-native livestock platform: an MCP server interface would allow language model agents to query animal records, health events, and performance data without custom tool integrations.

---

## Similar Products — Developer Documentation & APIs

### ICAR Animal Data Exchange (ADE) — Open Standard

- **Description:** The canonical open-standard API for individual animal data exchange across the livestock industry, developed by the ICAR ADE Working Group. Covers dairy, beef, sheep, and farmed deer.
- **API Documentation:** https://github.com/adewg/ICAR/blob/ADE-1/docs/generic-data-exchange-api.md
- **OpenAPI / JSON Schema:** https://github.com/adewg/ICAR/blob/ADE-1/resources/icarResource.json
- **Wiki:** https://github.com/adewg/ICAR/wiki
- **Standards:** REST/JSON, Apache 2.0 licence
- **Authentication:** Not prescribed by the standard; implementers use OAuth 2.0 or API keys per their platform

### Farmbrite — Livestock & Farm Management API

- **Description:** Multi-species SaaS platform covering cattle, sheep, goats, pigs, poultry, and horses. Provides a published REST API for livestock records, health events, feeding, measurements, and inventory.
- **API Documentation:** https://developers.farmbrite.com/docs/
- **Developer Reference:** https://developers.farmbrite.com/
- **Standards:** REST/JSON, OpenAPI specification available
- **Authentication:** API key; Zapier integration for no-code connectivity

### UK Livestock Information Service (LIS) — Government API

- **Description:** The UK's national livestock traceability platform (replacing BCMS) for recording and reporting cattle, sheep, and goat movements. Provides a developer API for third-party farm management software to submit and retrieve movement records electronically.
- **Developer Hub:** https://developers.livestockinformation.org.uk/
- **APIs Overview:** https://developer.service.livestockinformation.org.uk/apis/overview
- **Third-party Lifecycle Guide:** https://developers.livestockinformation.org.uk/documentation/third-party-application-lifecycle
- **Authentication Documentation:** https://developers.livestockinformation.org.uk/documentation/api-authentication
- **Standards:** REST/JSON, OpenAPI specification
- **Authentication:** OAuth 2.0 via Azure B2C; subscription key required

### Pure Farming (Map of Ag) — Agri Data Platform API

- **Description:** A multi-farm, multi-species data platform that standardises and provisions permissioned farm data from multiple sources including livestock records, field boundaries, weather, and satellite imagery. Supports ICAR breed coding schemes.
- **Developer Hub:** https://developer.purefarming.com/connected-data/
- **Data API Documentation:** https://developer.purefarming.com/documentation/data/index.html
- **Breed Reference API:** https://developer.purefarming.com/documentation/reference/breed.html
- **API Explorer:** https://developer.purefarming.com/api-explorer/index.html
- **Standards:** REST/JSON, OGC API, streaming API
- **Authentication:** OAuth 2.0 with farmer-controlled data consent

### Integrity Systems Company (ISC) — NLIS Australia API

- **Description:** The national NLIS (National Livestock Identification System) database for Australian cattle, sheep, and goats. ISC provides a developer portal and API for third-party software to submit and retrieve livestock movement, LPA (Livestock Production Assurance), and eNVD (electronic National Vendor Declaration) records.
- **Developer Portal:** https://www.integritysystems.com.au/about/api/
- **Third-party Integrators:** https://www.integritysystems.com.au/identification--traceability/NLIS-Database-Uplift-Project/third-party-integrators/
- **Standards:** REST/JSON, OpenAPI specifications; mock endpoints available in sandbox
- **Authentication:** Registered developer access; sandbox environment for integration testing

### Leaf Agriculture — Unified Farm Data API

- **Description:** A unified farm data aggregation API that normalises data from major precision agriculture providers (John Deere, Ag Leader, Climate FieldView, and others) into a single schema. Provides field operations, boundary, weather, and satellite data. Recently launched an MCP server for AI agent access.
- **Developer Documentation:** https://leaf-agriculture.github.io/docs/docs/
- **MCP Server:** https://withleaf.io/en/whats-new/leaf-mcp-launch/
- **Standards:** REST/JSON; language examples in cURL, Python, Node.js
- **Authentication:** API key

### Herdwatch Enterprise — Livestock Data Platform

- **Description:** European livestock management platform (22,000+ producers) with an enterprise-grade data layer that normalises animal-level records (movements, treatments, weights, carbon, performance) into XML/API-ready feeds for ERP, processor, and sustainability platform integration.
- **Enterprise Documentation:** https://herdwatch.com/enterprise/
- **Standards:** XML and REST API feeds; compatible with ERP and iPaaS platforms
- **Authentication:** Partner-gated access; contact enterprise@herdwatch.com

### OpenEPCIS — Open-Source EPCIS 2.0 Reference Implementation

- **Description:** A fully GS1 EPCIS 2.0-compliant open-source platform for supply chain event visibility. Provides a ready-to-deploy server for capturing livestock supply chain events (birth, movement, slaughter, processing) with a REST/JSON-LD API.
- **Project Site:** https://openepcis.io/
- **GitHub:** https://github.com/openepcis
- **Standards:** GS1 EPCIS 2.0, JSON-LD, REST Web API
- **Authentication:** Configurable; standard OAuth 2.0 patterns supported

---

## Notes

- **ICAR ADE as the integration backbone:** For a new AI-native livestock platform, implementing the ICAR ADE standard as the primary data exchange format maximises interoperability with existing farm software vendors, national databases, and sensor hardware suppliers. The Apache 2.0 licence permits commercial implementation without restriction.

- **Dual API strategy (government + commercial):** The UK LIS and Australian NLIS APIs represent mandatory government integrations for regulatory compliance in their respective markets. These are non-negotiable for producers operating in those jurisdictions and should be treated as first-class integration targets.

- **EPCIS 2.0 for supply chain use cases:** Producers selling into supermarket supply chains or export markets increasingly face requirements for EPCIS-formatted traceability events. Building an EPCIS event export function on top of core livestock records positions the platform for supply chain integrations without re-architecting core data models.

- **MCP server opportunity:** No incumbent livestock management platform has yet shipped a production MCP server. Building one early—allowing AI agents to query herd records via natural language—would be a meaningful differentiator for an AI-native platform targeting tech-forward producers and veterinary advisors.

- **Emerging standards gap:** There is no single ISO or W3C standard for livestock health vocabulary or veterinary event classification. The veterinary informatics community has proposed adapting SNOMED CT and HL7 to animal health (AVMA guidelines), but adoption in farm software is minimal. This gap represents an opportunity to define a de-facto schema within an open-source platform.
