# Standards & API Reference

> Project: Code Quality Metrics Dashboard · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 25010:2023 — Systems and Software Quality Requirements and Evaluation (SQuaRE) — Product Quality Model**
- URL: https://www.iso.org/standard/78176.html
- The primary international standard defining software product quality characteristics. The 2023 revision defines nine quality characteristics (Functional Suitability, Performance Efficiency, Compatibility, Interaction Capability, Reliability, Security, Maintainability, Flexibility, and Safety), each further subdivided into measurable sub-characteristics. A code quality dashboard must map its metrics to these characteristics — particularly Maintainability, Reliability, and Security — to align with the standard and enable benchmarking across organisations.

**ISO/IEC 5055:2021 — Automated Source Code Quality Measures**
- URL: https://www.iso.org/standard/80623.html
- Also known as the CISQ standard (Consortium for Information and Software Quality). This is the first ISO standard that measures software quality directly from internal source code structure via static analysis, covering four business-critical factors: Security, Reliability, Performance Efficiency, and Maintainability. Each quality factor is operationalised as a set of CWE (Common Weakness Enumeration) weaknesses, making this the definitive standard for automated code quality measurement and a key reference for any tool implementing quality scoring at the codebase level.

**ISO/IEC 25023 — Systems and Software Quality Requirements and Evaluation (SQuaRE) — Measurement of System and Software Product Quality**
- URL: https://www.iso.org/standard/35747.html
- Companion standard to ISO/IEC 25010 that defines specific measures for each quality characteristic. Relevant for dashboard implementations that expose numeric quality scores and need to justify how those scores are calculated and what they represent.

### IEEE Standards

**IEEE 730-2014 — Standard for Software Quality Assurance Processes**
- URL: https://standards.ieee.org/ieee/730/5284/
- Establishes minimum requirements for software quality assurance (SQA) processes across the SDLC. Relevant as the framework within which code quality metrics are gathered and reported; dashboards operating in regulated industries (defence, aerospace, medical devices) are frequently required to demonstrate compliance with IEEE 730.

**IEEE 1061-1998 — Standard for a Software Quality Metrics Methodology**
- URL: https://ieeexplore.ieee.org/document/749159/
- Defines a methodology for establishing quality requirements and identifying, implementing, analysing, and validating software quality metrics. While older, it remains the foundational IEEE reference for metric selection and validation, directly applicable to deciding which metrics a code quality dashboard should expose and how they should be validated.

### W3C & IETF Standards

**RFC 7231 — Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Defines the semantics of the HTTP/1.1 protocol used by all REST APIs in the code quality tools space. Dashboard API implementations should conform to HTTP semantics for status codes, methods, and content negotiation.

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Defines the `Link` header and link relations, used by pagination patterns in REST APIs (e.g., Code Climate's JSON API pagination). Relevant for dashboard APIs that expose paginated metric histories.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The authentication standard underlying all major code quality tool APIs (SonarQube v2, Codacy v3, Semgrep, GitHub Code Scanning). A dashboard aggregating multiple upstream sources must implement OAuth 2.0 flows for each.

### Data Model & API Specifications

**OASIS SARIF v2.1.0 — Static Analysis Results Interchange Format**
- URL: https://docs.oasis-open.org/sarif/sarif/v2.1.0/sarif-v2.1.0.html
- OASIS-approved standard defining a common JSON-based format for the output of static analysis tools. SARIF is the interchange format used by GitHub Advanced Security, Visual Studio, and many CI/CD platforms to ingest and display code scanning results. Any code quality dashboard aggregating results from multiple SAST tools should support SARIF as both an import and export format.

**OpenAPI Specification v3.1.0 (OAS)**
- URL: https://spec.openapis.org/oas/v3.1.0.html
- The de facto standard for describing REST APIs. Semgrep, Qodana, and Codacy all publish OpenAPI specifications for their APIs. A code quality dashboard should expose an OAS 3.1-compliant API description to enable SDK generation, partner integrations, and MCP tool exposure.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/specification.html
- Used for validating API request and response bodies, webhook payloads, and configuration files (e.g., `.codeclimate.yml`, `sonar-project.properties`). A dashboard accepting metric data from multiple upstream tools should define JSON Schema documents for all inbound data models.

**JSON:API Specification v1.1**
- URL: https://jsonapi.org/format/
- Used by Code Climate's REST API for structuring resource collections, pagination, and relationships. Relevant if the dashboard adopts a similar compound-document API style.

**OpenTelemetry (OTEL) — Traces, Metrics, Logs Specification**
- URL: https://opentelemetry.io/docs/specs/otel/
- CNCF-graduated standard for observability telemetry, with stable SDKs across all major languages. In 2026, DORA metrics (deployment frequency, lead time, change failure rate, MTTR) are increasingly tracked alongside application metrics via OpenTelemetry pipelines. A code quality dashboard that surfaces engineering performance alongside code health metrics should integrate with OTEL-compatible backends (Prometheus, Grafana, Datadog).

**Model Context Protocol (MCP) — Specification 2025-11-25**
- URL: https://modelcontextprotocol.io/specification/2025-11-25
- Anthropic's open protocol for exposing tool capabilities to AI models via JSON-RPC 2.0. By 2026, the MCP ecosystem includes servers for GitHub, static analysis, and code graph analysis (e.g., Code Pathfinder). An AI-native code quality dashboard should expose an MCP server to allow AI assistants to query metrics, trends, and hotspot data on demand.

### Security & Authentication Standards

**OWASP Secure Code Review Guide**
- URL: https://owasp.org/www-project-code-review-guide/
- OWASP's authoritative guide for manual and automated code review, defining vulnerability patterns, review checklists, and the relationship between code quality and security. Relevant for dashboards that incorporate security-oriented metrics alongside quality metrics.

**OWASP Secure Coding Practices — Quick Reference Guide**
- URL: https://owasp.org/www-project-secure-coding-practices-quick-reference-guide/
- Defines actionable coding practices across six domains. A code quality dashboard implementing rule-based checks should align its security rules with OWASP SCP to ensure industry-recognised coverage.

**NIST SP 800-218 v1.1 — Secure Software Development Framework (SSDF)**
- URL: https://csrc.nist.gov/pubs/sp/800/218/final
- NIST's framework for secure software development, structured around four practices: Prepare the Organisation (PO), Protect Software (PS), Produce Well-Secured Software (PW), and Respond to Vulnerabilities (RV). Organisations in regulated sectors increasingly require SSDF compliance; a dashboard that maps its findings to SSDF practice IDs adds compliance-reporting value.

**NIST Common Weakness Enumeration (CWE)**
- URL: https://cwe.mitre.org/
- The industry-standard taxonomy for software weaknesses, referenced directly by ISO/IEC 5055 and CISQ standards. Code quality rules should be mapped to CWE identifiers so findings can be cross-referenced with CVE databases, OWASP Top 10, and vulnerability management tooling.

---

## Foundational Code Metrics References

These are not formal standards but are the de facto mathematical definitions underpinning every code quality tool in the market.

**McCabe Cyclomatic Complexity (1976)**
- Reference: McCabe, T.J. (1976). "A Complexity Measure." IEEE Transactions on Software Engineering, 2(4):308–320.
- NIST guidance: https://www.mccabe.com/pdf/mccabe-nist235r.pdf
- Quantifies the number of linearly independent paths through code. Values above 10 are universally treated as a quality gate threshold. Every code quality dashboard must implement cyclomatic complexity as a table-stakes metric.

**Halstead Software Science Metrics (1977)**
- Reference: Halstead, M.H. (1977). Elements of Software Science. Elsevier.
- Derives program volume, difficulty, and effort from operator/operand counts. Combined with cyclomatic complexity, these form the Maintainability Index used by Visual Studio and SonarQube.

**Maintainability Index**
- Reference: Oman, P. & Hagemeister, J. (1992). "Metrics for Assessing a Software System's Maintainability." Proceedings of the International Conference on Software Maintenance.
- Composite score (0–100) combining cyclomatic complexity, Halstead volume, and lines of code. Scores below 10 indicate unmaintainable code; Visual Studio and SonarQube both expose this metric.

---

## Similar Products — Developer Documentation & APIs

### SonarQube (Sonar)

- **Description:** The market-leading static analysis and code quality platform, used by 400K+ organisations across 30+ languages. Available as SonarQube Server (self-hosted) and SonarQube Cloud (SaaS).
- **API Documentation (v1):** https://docs.sonarsource.com/sonarqube-server/latest/extension-guide/web-api/
- **API Documentation (v2 — current):** https://next.sonarqube.com/sonarqube/web_api
- **Blog post introducing API v2:** https://www.sonarsource.com/blog/new-web-api-v2
- **SonarQube Cloud API:** https://docs.sonarsource.com/sonarqube-cloud/appendices/web-api
- **SDKs/Libraries:** No official SDK; community wrappers exist in Python and Java. Official CLI: `sonar-scanner`.
- **Standards:** REST (v2 adopts REST architectural style); Bearer token authentication.
- **Authentication:** Bearer token (recommended); X-Sonar-Passcode (legacy).
- **Key endpoints:** `/api/measures` (metric values and histories), `/api/metrics` (metric key catalogue), `/api/projects/search`.

### Code Climate

- **Description:** Code quality and test coverage platform targeting small-to-mid engineering teams, with a GitHub-native integration and a JSON API-compliant REST interface.
- **API Documentation:** https://developer.codeclimate.com/
- **Standards:** REST, JSON:API v1.1 specification.
- **Authentication:** Personal access tokens; Bearer scheme.
- **Rate Limiting:** 5,000 requests per token per hour.
- **Pagination:** Cursor-based via `page[]` query parameters per JSON:API.

### Codacy

- **Description:** Automated code review platform offering complexity, duplication, coverage, and security metrics with CI/CD-native integration.
- **API Documentation:** https://docs.codacy.com/codacy-api/using-the-codacy-api/
- **API Version:** v3 (actively developed; v2 deprecated).
- **OpenAPI Import:** OpenAPI 2.0 definition available for tooling integration.
- **SDKs/Libraries:** Java client: https://github.com/codacy/codacy-api-java
- **Plugins API (for custom analysers):** https://github.com/codacy/codacy-plugins-api
- **Standards:** REST/JSON; OpenAPI 2.0 spec published.
- **Authentication:** `api-token` header (account token) or `project-token` header (repository token).
- **Pagination:** Cursor-based; `limit` parameter (max 1000, default 100).

### DeepSource

- **Description:** AI-augmented static analysis platform with autofix capabilities across 20+ languages and a developer-friendly feedback loop.
- **API Documentation:** https://docs.deepsource.com/docs/api
- **API Type:** GraphQL (not REST).
- **GraphQL Endpoint:** `https://api.deepsource.io/graphql/`
- **API Playground:** Postman collection available.
- **MCP Server:** Available — exposes scan results and code health data to AI assistants.
- **Standards:** GraphQL; Bearer token authentication.
- **Authentication:** Personal access token in `Authorization: Bearer <token>` header.
- **Rate Limiting:** Per-user limits; HTTP 429 on exceedance.
- **Webhook Events:** Real-time webhook events for scan completion and issue lifecycle.

### Semgrep

- **Description:** Pattern-based static analysis tool with a highly extensible custom rule engine, available as open-source CLI and SaaS AppSec Platform.
- **API Documentation:** https://semgrep.dev/docs/semgrep-appsec-platform/semgrep-api
- **API ReDoc (interactive):** https://semgrep.dev/api/v1/docs/
- **GitHub Repository (OSS):** https://github.com/semgrep/semgrep
- **Integration Guide:** https://semgrep.dev/docs/integrating
- **Standards:** REST/JSON; OpenAPI format documentation.
- **Authentication:** Bearer token (Team/Enterprise accounts required for API).
- **Capabilities:** List deployments, gather findings, list projects; integrates via Pipedream, GitHub Actions, CI/CD webhooks.

### Qodana (JetBrains)

- **Description:** JetBrains code quality platform built on IntelliJ inspection engine, with deep IDE integration and support for JVM, Python, JavaScript, and PHP.
- **API Documentation:** https://www.jetbrains.com/help/qodana/cloud-api.html
- **OpenAPI Specification:** Published as `openapi.yaml` at https://github.com/jetbrains-qodana/public-api
- **CLI Tool:** https://github.com/JetBrains/qodana-cli
- **Standards:** REST; OpenAPI 3.x spec; Bearer token authentication.
- **Authentication:** Organisation API token; Owner/Admin role required.
- **Licence:** Ultimate Plus plan required for public API access.
- **Key Capabilities:** Create teams/projects, list users, retrieve analysis results programmatically.

### CodeScene

- **Description:** Behavioural code analysis platform combining Git history with complexity metrics to identify high-risk hotspots and knowledge silos.
- **API Documentation (Cloud):** https://codescene.io/docs/integrations/rest-api.html
- **API Documentation (Enterprise):** https://docs.enterprise.codescene.io/latest/integrations/rest-api.html
- **Standards:** REST v2; Swagger/OpenAPI documented; Bearer token authentication.
- **Authentication:** `Authorization: Bearer <token>` header; tokens generated in CodeScene UI.
- **Key Endpoints:** Project management, developer settings, analysis data, KPI trend series.
- **Pagination:** Filter parameters supported on all paged endpoints.

### GitHub Advanced Security (Code Scanning)

- **Description:** GitHub's integrated SAST platform (powered by CodeQL and third-party SARIF uploads) exposing code scanning alerts via the GitHub REST API.
- **API Documentation:** https://docs.github.com/en/rest/code-scanning/code-scanning
- **SDKs/Libraries:** Octokit (JavaScript, Ruby, .NET, Go): https://github.com/octokit
- **Standards:** REST/JSON; SARIF v2.1.0 for results upload; OAuth 2.0 / personal access token authentication.
- **Authentication:** `security_events` scope (private repos) or `public_repo` scope (public repos).
- **Key Endpoints:** List alerts for organisation, list alerts for repository, get single alert, update alert status.
- **API Version:** Current as of 2026-03-10.

---

## Notes

**Emerging convergence:** By 2026, SARIF has become the lingua franca for passing static analysis results between tools, CI/CD platforms, and dashboards. Any new tool aggregating from multiple sources should treat SARIF ingestion as table stakes.

**GraphQL vs REST split:** The market is divided — Semgrep and DeepSource use GraphQL while SonarQube, Code Climate, Codacy, and Qodana use REST. An aggregation dashboard must support both paradigms.

**MCP opportunity:** DeepSource already exposes an MCP server for its findings. This is a meaningful differentiator for AI-native tools; a new dashboard that exposes code quality data over MCP enables AI assistants to answer questions like "which files are most at risk this sprint?" without custom integrations.

**DORA metrics gap:** No current code quality dashboard natively unifies DORA metrics (deployment frequency, lead time, MTTR, change failure rate) with code quality scores. The OpenTelemetry ecosystem provides the infrastructure; the integration opportunity remains open.
