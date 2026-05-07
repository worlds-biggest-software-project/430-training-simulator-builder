# Standards & API Reference

> Project: Training Simulator Builder · Generated: 2026-05-07

---

## Industry Standards & Specifications

### eLearning Content & Tracking Standards

**SCORM 1.2 and SCORM 2004 (Sharable Content Object Reference Model)**
- Standard body: Advanced Distributed Learning (ADL) Initiative
- URL: https://adlnet.gov/projects/scorm/
- The dominant eLearning packaging and tracking standard used by the majority of corporate LMS platforms. SCORM defines how content packages are structured (ZIP with manifest), how they communicate runtime data (completion, score, time) to an LMS via a JavaScript API, and how learners' progress is tracked. SCORM 1.2 remains the most widely supported version; SCORM 2004 (4th Edition) adds sequencing and navigation. Any training simulator must export SCORM-compliant packages to be deployable to Cornerstone, SAP SuccessFactors, Docebo, Moodle, and other LMS platforms.

**xAPI / Experience API (formerly Tin Can API) — Version 1.0.3**
- Standard body: ADL Initiative
- Spec repository: https://github.com/adlnet/xAPI-Spec
- Data model reference: https://github.com/adlnet/xAPI-Spec/blob/master/xAPI-Data.md
- Communication reference: https://github.com/adlnet/xAPI-Spec/blob/master/xAPI-Communication.md
- xAPI extends SCORM's tracking capabilities beyond LMS boundaries. Statements follow a subject-verb-object format (e.g., "learner answered decision-point correctly") and are sent over HTTP REST to a Learning Record Store (LRS). For simulation training, xAPI enables granular tracking of every branch decision, response attempt, and scenario path — data unavailable in SCORM. Mandatory for compliance audit trails and advanced analytics dashboards.

**cmi5 — AICC/ADL cmi5 Specification**
- Standard body: ADL Initiative (inherited from the dissolved AICC)
- Spec: https://aicc.github.io/CMI-5_Spec_Current/
- Comparison with SCORM: https://aicc.github.io/CMI-5_Spec_Current/SCORM/
- cmi5 is an xAPI profile that adds a precise deployment protocol to xAPI's flexible tracking — defining mandatory initialization, completion, and session-close statements that SCORM structured but xAPI left open. It is the active evolution of SCORM, backed by the U.S. Department of Defense as the official successor. Moodle 4.x, Docebo, TalentLMS, and Cornerstone have all strengthened cmi5 support. A simulator builder targeting future-proof LMS compatibility should support cmi5 export alongside SCORM and xAPI.

**AICC (Aviation Industry Computer-Based Training Committee) — HACP Protocol**
- Standard body: AICC (dissolved 2014; documentation archived at ADL)
- Reference: https://elearningindustry.com/elearning-standards-scorm-aicc-xapi-cmi5-ims-cartridge
- The predecessor to SCORM, still supported by legacy enterprise LMS installations. AICC's HACP (HTTP-based AICC/CMI Protocol) uses HTTP POST to communicate between content and LMS, rather than JavaScript API calls. Some older Cornerstone and SAP SuccessFactors deployments require AICC support. Low priority for new development but worth tracking for enterprise compatibility.

---

### Learning Interoperability Standards

**IMS Learning Tools Interoperability (LTI) 1.3**
- Standard body: 1EdTech (formerly IMS Global Learning Consortium)
- Spec: https://www.imsglobal.org/spec/lti/v1p3
- Assignment and Grade Services (AGS) OpenAPI: https://www.imsglobal.org/spec/lti-ags/v2p0/openapi
- Overview: https://www.imsglobal.org/activity/learning-tools-interoperability
- LTI enables a simulation tool to be launched natively within an LMS (Canvas, Moodle, Brightspace, Blackboard) without SCORM packaging, using OpenID Connect and signed JWTs for authentication. LTI 1.3's Assignment and Grade Services allow the simulator to post scores and completion status directly back to the LMS gradebook. This is increasingly preferred over SCORM in higher education and modern enterprise LMS deployments.

**IEEE 1484.12.1 — Learning Object Metadata (LOM)**
- Standard body: IEEE Learning Technology Standards Committee (LTSC)
- URL: https://www.ieeeltsc.org/working-groups/wg12LOM/lomDescription/
- LOM defines the metadata schema for describing eLearning content objects — including title, description, keywords, learning objectives, educational context, and technical requirements. Compliance training simulations that are catalogued in a content library (e.g., Cornerstone Content Subscriptions) use LOM-compliant metadata for discoverability. Relevant for any catalogue/marketplace distribution model.

**IMS Content Packaging**
- Standard body: 1EdTech
- Reference: https://aristeksystems.com/blog/elearning-standards/
- Defines the ZIP-based manifest structure (imsmanifest.xml) used by SCORM content packages. Required for any tool that generates SCORM output. Well-understood and stable specification.

---

### Accessibility Standards

**WCAG 2.2 — Web Content Accessibility Guidelines**
- Standard body: W3C Web Accessibility Initiative (WAI)
- URL: https://www.w3.org/WAI/standards-guidelines/
- WCAG 2.2 is now ISO/IEC 40500:2025 and is the current compliance target for accessible eLearning content. Level AA compliance is required for US federal contracts (Section 508) and EU public sector deployments (EN 301 549). Training simulations deployed in regulated industries (healthcare, financial services, government) must meet WCAG 2.2 AA to satisfy procurement requirements and legal obligations. Authored output must have accessible keyboard navigation, screen reader compatibility, sufficient colour contrast, and captions for audio content.

**WAI-ARIA 1.3 — Accessible Rich Internet Applications**
- Standard body: W3C
- URL: https://www.w3.org/WAI/standards-guidelines/aria/
- ARIA provides semantic roles and properties that make dynamic web interfaces (such as interactive simulation elements) accessible to assistive technologies. Branching scenario UIs, dialogue choice buttons, and progress indicators must use ARIA roles and states to be navigable by screen readers. Required for WCAG 2.2 AA compliance on complex interactive content.

---

### Security & Authentication Standards

**OAuth 2.0 — RFC 6749**
- Standard body: IETF
- URL: https://oauth.net/2/
- The standard authorization framework for API access delegation. The simulator builder's backend API should use OAuth 2.0 for programmatic integrations (LMS webhooks, content library access, LRS posting) and for SaaS multi-tenant authentication. LTI 1.3 is itself built on OAuth 2.0 and OpenID Connect.

**OpenID Connect (OIDC) 1.0**
- Standard body: OpenID Foundation
- URL: https://openid.net/connect/
- OpenID Connect is the identity layer built on OAuth 2.0, providing user authentication in addition to authorization. Required for LTI 1.3 integration and for enterprise SSO (SAML 2.0 alternative). Most enterprise LMS platforms support OIDC-based SSO, and a SaaS training authoring tool should support it for seamless organizational login.

**JWT — RFC 7519 / RFC 9068 (JWT Profile for OAuth 2.0 Access Tokens)**
- Standard body: IETF
- URL: https://datatracker.ietf.org/doc/rfc9068/
- JSON Web Tokens are used throughout LTI 1.3, xAPI authentication, and modern API security. RFC 9068 standardizes the JWT format for OAuth access tokens, enabling interoperability across identity providers and resource servers. Short-lived tokens (≤15 minutes) with refresh token rotation are security best practice for simulation delivery APIs.

---

### Data & Content Format Standards

**OpenAPI Specification 3.1**
- Standard body: OpenAPI Initiative (Linux Foundation)
- URL: https://spec.openapis.org/oas/v3.1.0
- The de-facto standard for documenting REST APIs. The simulator builder's public API (for LMS integration, content export, analytics retrieval) should be documented with an OpenAPI 3.1 specification to enable LMS vendors, customers, and integrators to build against it without hand-holding.

**JSON Schema (Draft 2020-12)**
- Standard body: IETF / json-schema.org
- URL: https://json-schema.org/
- Used for validating xAPI statement structure, API request/response bodies, and scenario data interchange formats. A simulator builder that exposes a public API or stores scenario content in a portable format should use JSON Schema to define and validate data structures.

**ISO 29990:2010 — Learning Services for Non-Formal Education and Training**
- Standard body: ISO
- URL: https://www.iso.org/standard/53392.html
- Defines basic requirements for quality management of learning service providers in non-formal education and training contexts. Relevant for enterprise customers in regulated industries who require their training vendors (including simulation platform providers) to demonstrate process quality and service standards. Not a technical implementation standard but a quality framework for sales and procurement contexts.

---

## Similar Products — Developer Documentation & APIs

### Rustici SCORM Cloud / Rustici Engine

- **Description:** Rustici Software's SCORM Cloud is the industry-standard hosted test-and-distribution environment for SCORM, xAPI, cmi5, and AICC content. Rustici Engine (formerly SCORM Engine) is the embeddable SDK that adds multi-format content playback and tracking to any LMS or application.
- **API Documentation:** https://cloud.scorm.com/docs/v2/reference/swagger/
- **Engine Documentation:** https://docs.rusticisoftware.com/engine/23.x/index.html
- **SDKs/Libraries:** REST API (language-agnostic); JavaScript player integration
- **Developer Guide:** https://rusticisoftware.com/technical-documentation/
- **Standards:** SCORM 1.1, 1.2, 2004 (all editions), xAPI, cmi5, AICC, LTI 1.1 and 1.3
- **Authentication:** API key (SCORM Cloud); application-level secrets (Engine)
- **Notes:** SCORM Cloud's hosted LRS (xAPI statements) is widely used as a reference implementation for testing. Integrating with Rustici Engine is the fastest path to certified SCORM/xAPI compatibility without implementing the runtime layer from scratch.

### ADL xAPI LRS (Reference Implementation)

- **Description:** The ADL's open-source reference Learning Record Store, used for conformance testing xAPI implementations.
- **API Documentation:** https://github.com/adlnet/xAPI-Spec/blob/master/xAPI-Communication.md
- **LRS Conformance Tests:** https://adl.gitbooks.io/xapi-lrs-conformance-requirements/content/
- **Hosted LRS:** https://lrs.adlnet.gov/
- **Standards:** xAPI 1.0.3
- **Authentication:** HTTP Basic Auth (reference implementation); production LRS implementations use OAuth 2.0
- **Notes:** All xAPI statements produced by the simulation tool should be validated against the ADL conformance test suite before release.

### iSpring TalkMaster API / iSpring Suite

- **Description:** iSpring's dialogue simulation module that produces SCORM-compliant branching conversation training. The platform provides a developer API for LMS integration and content distribution.
- **API Documentation:** https://www.ispringsolutions.com/ispring-talkmaster/features
- **Developer Resources:** https://www.ispringsolutions.com/ispring-suite/api
- **Standards:** SCORM 1.2, SCORM 2004, HTML5
- **Authentication:** API key for iSpring Learn LMS integration
- **Notes:** iSpring's PowerPoint plugin architecture is not extensible, but the platform demonstrates how a dialogue tree editor can be simplified for non-technical authors.

### Articulate 360 / Storyline API

- **Description:** Articulate's eLearning authoring suite is the market-dominant platform; Storyline 360 supports custom xAPI statement emission via JavaScript API for custom tracking beyond standard completion data.
- **API Documentation:** https://articulate.com/support/article/Storyline-360-Custom-xAPI-Statements
- **xAPI JavaScript Library (tincan.js):** https://rusticisoftware.github.io/TinCanJS/
- **Review 360 API:** Limited (private API for Articulate's own products)
- **Standards:** SCORM 1.2, SCORM 2004, xAPI, HTML5
- **Authentication:** Articulate account OAuth; LRS credentials configured per-publish
- **Notes:** Storyline's JavaScript API allows custom xAPI statement injection — a pattern a new simulator should replicate, allowing authors to define custom verb/object pairs beyond standard completion tracking.

### ElevenLabs Voice Synthesis API

- **Description:** ElevenLabs provides high-quality AI text-to-speech with voice cloning, voice design, multilingual synthesis, and a Conversational AI Agents SDK — enabling realistic voice characters for simulation scenarios.
- **API Documentation:** https://elevenlabs.io/docs/api-reference/introduction
- **Text-to-Speech Guide:** https://elevenlabs.io/docs/overview/capabilities/text-to-speech
- **Voice Design API:** https://elevenlabs.io/docs/api-reference/text-to-voice/design
- **Quickstart:** https://elevenlabs.io/docs/eleven-api/quickstart
- **SDKs/Libraries:** Python SDK, Node.js SDK, REST API
- **Standards:** REST/JSON; WebSocket for streaming real-time audio
- **Authentication:** API key (Bearer token)
- **Notes:** ElevenLabs' Flash v2.5 model achieves ~75ms latency — suitable for real-time conversational simulation. The voice design API (generate voice from text description) enables character voice creation without pre-recorded audio. Multilingual v2 model provides emotional nuance for realistic character voices.

### OpenAI API (GPT-4o / Realtime API)

- **Description:** OpenAI's API provides LLM-powered dialogue generation, scenario content creation, and the Realtime API for low-latency voice agent interactions — the primary AI backbone for AI-native simulation authoring.
- **API Documentation:** https://platform.openai.com/docs/
- **Audio and Speech Guide:** https://developers.openai.com/api/docs/guides/audio
- **Realtime API:** https://platform.openai.com/docs/guides/realtime
- **SDKs/Libraries:** Python SDK, Node.js SDK, REST API; Agents SDK with voice integration
- **Standards:** REST/JSON; WebSocket for Realtime API
- **Authentication:** API key (Bearer token); organization and project scoping
- **Notes:** The Realtime API supports production-ready voice agents with MCP server integration, SIP phone calling, and remote tool use — enabling open-ended conversational simulations where learners speak freely and the AI responds as a simulated character. GPT-4o's context window and instruction-following enable policy-document-grounded scenario generation with high accuracy.

### Cornerstone OnDemand / Talespin API

- **Description:** Cornerstone's enterprise LMS platform, now including Talespin's XR simulation authoring capabilities, with a REST API for content management, learner data, and simulation analytics.
- **API Documentation:** https://developer.csod.com/
- **Talespin (by Cornerstone) Integration:** https://www.cornerstoneondemand.com/company/news-room/press-releases/cornerstone-becomes-end-to-end-learning-content-solution-with-spatial-learning-acquisition/
- **Standards:** REST/JSON; SCORM, xAPI, LTI via Rustici Engine; Skills data via Cornerstone Skills Graph
- **Authentication:** OAuth 2.0
- **Notes:** Cornerstone's acquisition of Talespin (2024) means XR simulation content now flows through the Cornerstone content delivery infrastructure. Their developer API is the integration target for any enterprise simulation tool targeting Cornerstone LMS customers.

### H5P API / Content Hub

- **Description:** H5P's open-source branching scenario content type provides a reference implementation for scenario authoring and LTI-based delivery, with xAPI statement emission. The H5P.com hosted service offers a REST API for content management.
- **API Documentation:** https://h5p.com/developer
- **Open-Source Repository:** https://github.com/h5p/h5p-branching-scenario
- **LTI Integration Guide:** https://h5p.org/lti
- **SDKs/Libraries:** PHP library (h5p-php-library), JavaScript (h5p-core)
- **Standards:** xAPI (statement emission), LTI 1.1/1.3 for LMS embedding
- **Authentication:** API key for H5P.com; OAuth for LTI 1.3
- **Notes:** H5P's open-source branching scenario implementation is a useful reference for the node graph data model and xAPI statement patterns for simulation tracking, without IP encumbrances.

### Moodle REST API

- **Description:** Moodle is the dominant open-source LMS globally. Its REST API enables external content tools to enrol users, track completion, post grades, and retrieve learner data — critical for simulator integration with self-hosted LMS environments.
- **API Documentation:** https://moodledev.io/docs/apis/subsystems/external/
- **Web Services Reference:** https://docs.moodle.org/dev/Creating_a_web_service_client
- **Standards:** REST/JSON, SCORM player via Moodle core, xAPI via Moodle xAPI plugin, LTI 1.3
- **Authentication:** Token-based (user tokens); OAuth 2.0 for external services
- **Notes:** Moodle's SCORM activity module and H5P plugin are the most common deployment targets for open-source scenario training content. Understanding Moodle's completion tracking API is valuable for building LMS-native simulator integrations beyond standard SCORM packaging.

---

## Notes

**Emerging: MCP (Model Context Protocol) for AI authoring workflows**
Anthropic's Model Context Protocol (MCP) is emerging as a standard for AI agents to interact with external tools and data sources. For a Training Simulator Builder, MCP server integration could enable LLM-powered authoring assistants to pull policy documents from SharePoint, HR systems, or compliance knowledge bases to generate contextually accurate scenario content. This is not yet an eLearning industry standard but is relevant to the AI-native authoring architecture of a new tool.

**WCAG 3.0 timeline**
WCAG 3.0 is a Working Draft in 2026, projected to reach Candidate Recommendation in 2026–2027 and W3C Recommendation in 2027–2028. Build WCAG 2.2 Level AA compliance now; design the accessibility architecture to accommodate WCAG 3.0's scoring model when it reaches Recommendation status.

**cmi5 adoption momentum**
cmi5 adoption is accelerating, driven by U.S. Department of Defense mandates and major LMS platform updates (Moodle 4.x, Docebo, Cornerstone). New development should implement cmi5 as a first-class export format alongside SCORM, not as an afterthought.

**xAPI statement design**
The xAPI specification is intentionally flexible; interoperability requires choosing or defining an xAPI Profile (a curated set of verbs, activity types, and statement patterns for a specific domain). ADL maintains a registry of community profiles at https://profiles.adlnet.gov/ — a simulator builder should either adopt an existing profile (e.g., the cmi5 profile) or publish its own for transparency.
