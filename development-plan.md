# Code Quality Metrics Dashboard — Phased Development Plan

> Project: 287-code-quality-metrics-dashboard · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesizes `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files into concrete technology decisions, an architecture, and a sequenced set of implementation phases. Each task carries a **Design** section (real types, signatures, SQL, wire formats) and a **Testing** section (named scenarios with inputs and expected outcomes).

---

## Product Summary

An AI-native, open-source dashboard that measures code health — cyclomatic complexity, duplication, test coverage, code smells, security findings — and tracks how those metrics **evolve over time** across commits, sprints, and releases. The differentiators versus SonarQube/Codacy/Code Climate are: (1) trend tracking as a first-class concern rather than an afterthought, (2) SARIF ingestion as table stakes so the dashboard aggregates findings from any SAST/quality tool, (3) an MCP server exposing metrics to AI assistants, and (4) LLM-generated explanations, technical-debt prioritisation, and executive trend summaries.

**Primary personas:** engineering managers (technical-debt tracking), VPs of Engineering (executive code-health reporting), platform teams (CI/CD quality gates), security engineers (SAST findings alongside quality).

**Deployment model:** Self-hosted-first via Docker Compose, with a clean path to SaaS. API + web dashboard + CLI scanner + MCP server.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language (backend) | **TypeScript on Node.js 22 LTS** | The dashboard is API/integration/frontend-heavy, not ML-heavy. A single language across API, CLI, MCP server, and React frontend minimises context-switching and lets the SARIF/finding types be shared end-to-end. Octokit, the official MCP SDK, and the strongest SARIF tooling all live in the TS ecosystem. |
| Analysis engine language | **TypeScript + tree-sitter** | `tree-sitter` has official WASM grammars for 40+ languages with one consistent API, enabling multi-language complexity/duplication analysis from a Node process without per-language toolchains. McCabe complexity and duplication are AST-walk problems that tree-sitter handles cleanly. |
| API framework | **Fastify 5** | Fastest mainstream Node framework, first-class JSON Schema validation per route (we need JSON Schema Draft 2020-12 anyway per `standards.md`), and `@fastify/swagger` auto-generates the **OpenAPI 3.1** spec the standards doc requires. |
| Database | **PostgreSQL 17 + TimescaleDB 2.x** | Trend tracking is the #1 buying requirement (research.md). TimescaleDB makes time-bucketed queries, continuous aggregates, and 90%+ compression first-class while staying ordinary PostgreSQL for config/state. This adopts **data-model-suggestion-4** as the spine, blended with JSONB detail columns from **suggestion-3** for findings and AI insights. |
| ORM / query layer | **Drizzle ORM** for relational tables; **raw SQL via `postgres` (postgres.js)** for hypertable/continuous-aggregate queries | Drizzle gives typed schema + migrations for standard tables; TimescaleDB-specific SQL (`time_bucket`, `regr_slope`, continuous aggregates) is cleaner as raw parameterised SQL. |
| Task queue | **BullMQ on Redis** | Analyses, SARIF ingestion, and LLM calls are async and long-running. BullMQ gives retries, concurrency control, scheduled jobs, and a UI — exactly what webhook-triggered scans need. |
| Cache / pub-sub | **Redis 7** | Doubles as BullMQ backing store and dashboard query cache. |
| Frontend | **Next.js 15 (App Router) + React 19 + Tailwind + Tremor/Recharts** | Server components for fast dashboard loads; Tremor/Recharts for the trend charts that are the product's core. TypeScript shared types with the backend. |
| LLM integration | **Anthropic SDK (`@anthropic-ai/sdk`) with a provider abstraction** | AI-native features (explanations, prioritisation, summaries). Provider abstraction keeps OpenAI/local models swappable. Prompt caching used for repeated repo context. |
| MCP server | **`@modelcontextprotocol/sdk`** | `standards.md` calls out MCP (spec 2025-11-25) as a differentiator; DeepSource already ships one. Exposes metrics/trends/hotspots as tools over JSON-RPC 2.0. |
| Auth | **OAuth 2.0 (GitHub/GitLab) + JWT sessions; API tokens for CI** | RFC 6749 per standards.md; CI pipelines need long-lived project tokens like Codacy/SonarQube. |
| Containerisation | **Docker + Docker Compose** | Self-hosted-first deployment; one `docker compose up` brings up Postgres+Timescale, Redis, API, worker, web, MCP. |
| Testing | **Vitest** (unit/integration), **Testcontainers** (real Postgres/Redis), **Playwright** (e2e web) | Vitest is fast and native to the Vite/TS ecosystem; Testcontainers gives real TimescaleDB for integration tests; Playwright drives the dashboard. |
| Lint / format / types | **ESLint (flat config) + Prettier + `tsc --noEmit` + Biome optional** | Standard TS quality gate; the project must hold itself to the bar it measures. |
| Package manager / monorepo | **pnpm workspaces + Turborepo** | Multiple deployable apps (api, worker, web, cli, mcp) sharing packages (db, core, sarif, types). pnpm + Turbo give fast, cached builds. |
| Standards implemented | **SARIF 2.1.0** (import/export), **OpenAPI 3.1**, **JSON Schema 2020-12**, **CWE** mapping, **ISO/IEC 25010 + 5055** metric taxonomy, **McCabe** complexity, **Maintainability Index**, **MCP 2025-11-25** | Directly from `standards.md`; these are table stakes and differentiators rather than optional. |

### Project Structure

```
code-quality-dashboard/
├── package.json                      # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── tsconfig.base.json
├── docker-compose.yml                # postgres+timescale, redis, api, worker, web, mcp
├── docker-compose.dev.yml
├── Dockerfile.api
├── Dockerfile.worker
├── Dockerfile.web
├── .env.example
├── eslint.config.mjs
├── packages/
│   ├── types/                        # shared TS types (metrics, findings, SARIF, API DTOs)
│   │   └── src/
│   ├── db/                           # Drizzle schema, migrations, raw-SQL helpers, seed
│   │   ├── src/schema/               # relational tables (organizations, repositories, ...)
│   │   ├── src/timescale/            # hypertable + continuous-aggregate DDL
│   │   ├── migrations/
│   │   └── src/seed/                 # builtin metric_definitions, builtin rules
│   ├── core/                         # analysis engine (language-agnostic)
│   │   └── src/
│   │       ├── analyzers/            # complexity, duplication, smells, coverage parsers
│   │       ├── languages/            # tree-sitter grammar adapters
│   │       ├── metrics/              # McCabe, Halstead, Maintainability Index
│   │       └── quality-gate/         # gate evaluation engine
│   ├── sarif/                        # SARIF 2.1.0 import/export + validation
│   │   └── src/
│   ├── ai/                           # LLM provider abstraction + prompt templates
│   │   └── src/
│   └── config/                       # config loading + validation (env + project files)
│       └── src/
├── apps/
│   ├── api/                          # Fastify REST API + OpenAPI + auth + webhooks
│   │   └── src/
│   │       ├── routes/
│   │       ├── plugins/
│   │       └── server.ts
│   ├── worker/                       # BullMQ workers (analysis, sarif, ai, aggregates)
│   │   └── src/
│   │       ├── queues/
│   │       └── processors/
│   ├── web/                          # Next.js dashboard
│   │   └── src/app/
│   ├── cli/                          # `cqd` scanner CLI (runs locally / in CI)
│   │   └── src/
│   └── mcp/                          # MCP server exposing metrics/trends/hotspots
│       └── src/
└── tests/
    ├── fixtures/                     # sample repos, SARIF files, coverage reports
    └── e2e/                          # Playwright specs
```

Modules group by concern, not by phase. Every later phase adds files inside these directories without restructuring.

---

## Phase 1: Foundation — Monorepo, Database, Shared Types

### Purpose
Stand up the monorepo, the Docker dev environment, the database schema (relational + TimescaleDB hypertables + continuous aggregates), and the shared type package. After this phase, `docker compose up` brings up Postgres+TimescaleDB and Redis, migrations apply cleanly, the schema is seeded with built-in metric definitions and rules, and every downstream package can import strongly typed domain models. This is foundational and additive-only.

### Tasks

#### 1.1 — Monorepo & tooling scaffold

**What**: Initialise pnpm + Turborepo workspace with TypeScript, ESLint flat config, Prettier, and Vitest wired across all packages/apps.

**Design**:
- Root `pnpm-workspace.yaml` lists `packages/*` and `apps/*`.
- `turbo.json` pipelines: `build` (depends on `^build`), `lint`, `test`, `typecheck`.
- `tsconfig.base.json`: `"target": "ES2023"`, `"module": "NodeNext"`, `"strict": true`, `"noUncheckedIndexedAccess": true`, path aliases `@cqd/*` → `packages/*/src`.
- Each package exposes a `package.json` with `"exports"` map and `scripts`: `build`, `test`, `lint`, `typecheck`.
- `.env.example` documents: `DATABASE_URL`, `REDIS_URL`, `ANTHROPIC_API_KEY`, `JWT_SECRET`, `GITHUB_OAUTH_CLIENT_ID/SECRET`, `PUBLIC_BASE_URL`.

**Testing**:
- `Unit: pnpm install + turbo build` → all empty packages build, exit 0.
- `Unit: turbo lint` on a deliberately mis-formatted file → non-zero exit, ESLint reports the rule.
- `Unit: turbo typecheck` with a type error introduced → fails with the expected TS error code.

#### 1.2 — Docker dev environment

**What**: `docker-compose.dev.yml` bringing up PostgreSQL 17 + TimescaleDB and Redis 7 with healthchecks.

**Design**:
- `db` service uses `timescale/timescaledb:latest-pg17`, volume-mounted data, `POSTGRES_DB=cqd`, healthcheck `pg_isready`.
- `redis` service `redis:7-alpine`, healthcheck `redis-cli ping`.
- Ports exposed: 5432, 6379. App services (api/worker/web/mcp) added in later phases but stubbed here with `profiles: [full]`.

**Testing**:
- `Integration (real): docker compose -f docker-compose.dev.yml up -d` → both services reach `healthy` within 30s.
- `Integration (real): psql $DATABASE_URL -c "CREATE EXTENSION IF NOT EXISTS timescaledb; SELECT extversion FROM pg_extension WHERE extname='timescaledb';"` → returns a 2.x version string.

#### 1.3 — Shared types package (`packages/types`)

**What**: Canonical TypeScript types/Zod schemas for the domain, reused by API, worker, CLI, web, and MCP.

**Design**:
```typescript
// metric.ts
export const MetricDomain = ['complexity','coverage','duplication','security','maintainability','size'] as const;
export type MetricDomain = (typeof MetricDomain)[number];

export interface MetricDefinition {
  key: string;                 // e.g. 'cyclomatic_complexity'
  name: string;
  domain: MetricDomain;
  valueType: 'integer'|'float'|'percent'|'rating'|'boolean';
  direction: -1 | 0 | 1;       // -1 lower-is-better, 1 higher-is-better
}

// finding.ts
export type Severity = 'info'|'minor'|'major'|'critical'|'blocker';
export type FindingType = 'bug'|'vulnerability'|'code_smell'|'security_hotspot';
export type FindingStatus = 'open'|'confirmed'|'resolved'|'false_positive'|'wont_fix';

export interface Finding {
  id: string;
  ruleKey: string;
  severity: Severity;
  type: FindingType;
  status: FindingStatus;
  componentPath: string;
  line?: number; endLine?: number; column?: number;
  message: string;
  effortMinutes?: number;
  cweIds: number[];
  findingHash: string;         // deterministic dedup hash across analyses
}

// analysis.ts
export type AnalysisStatus = 'pending'|'running'|'completed'|'failed';
```
- Zod schema mirrors for each (`MetricDefinitionSchema`, `FindingSchema`, …) used for runtime validation at API and queue boundaries.
- `findingHash` algorithm specified here (see 3.4) so producers and consumers agree.

**Testing**:
- `Unit: FindingSchema.parse(valid finding)` → returns typed object.
- `Unit: FindingSchema.parse({severity:'fatal'})` → ZodError naming `severity`.
- `Unit: exhaustive switch over FindingType` compiles (no `default`), proving the union is closed.

#### 1.4 — Database schema: relational tables (`packages/db`)

**What**: Drizzle schema + migrations for the standard relational tables.

**Design** (adapted from suggestion-4 relational core + suggestion-3 JSONB):
- Tables: `organizations`, `users`, `repositories`, `branches`, `components`, `analyses`, `rules`, `quality_gates`, `quality_profiles`, `quality_profile_rules`, `metric_definitions`, `api_tokens`.
- `repositories` carries `metadata JSONB`, `config JSONB` (scanner config, exclusion globs, custom thresholds).
- `analyses` carries relational summary columns (`coverage_pct`, `avg_complexity`, `duplication_pct`, `maintainability_idx`, `findings_count`, `quality_gate_status`) plus `metrics_detail JSONB`.
- `rules` carries `cwe_ids INTEGER[]`, `owasp_ids TEXT[]`, `severity`, `rule_type`, `remediation_minutes`.
- All ids `UUID DEFAULT gen_random_uuid()`; `created_at/updated_at TIMESTAMPTZ DEFAULT now()`.
- Drizzle migration generator wired (`drizzle-kit generate` → `packages/db/migrations`).

**Testing**:
- `Integration (Testcontainers): run all migrations on fresh TimescaleDB` → exits 0, `\dt` lists all expected tables.
- `Integration: insert org → repo → branch → analysis` with valid FKs → succeeds; inserting an analysis with a non-existent `repository_id` → FK violation.
- `Unit: drizzle schema typecheck` → inferred row types match `packages/types`.

#### 1.5 — Database schema: TimescaleDB hypertables + continuous aggregates

**What**: Hypertables and continuous aggregates for time-series metric, finding, coverage, and gate data.

**Design** (from suggestion-4, verbatim intent):
- Hypertables: `metric_samples`, `finding_events`, `coverage_samples`, `gate_evaluations`, each `create_hypertable(..., chunk_time_interval => INTERVAL '7 days')` (gate evals 30 days).
- Compression policies at 14 days; retention: metric_samples 180d, finding_events 365d.
- Continuous aggregates: `metrics_hourly`, `metrics_daily`, `metrics_weekly`, `findings_daily_summary`, `repo_health_daily` with refresh policies exactly as specified in suggestion-4.
- These DDL statements live in `packages/db/src/timescale/*.sql` and run after Drizzle relational migrations via a `pnpm db:timescale` step (Drizzle does not manage hypertables).

**Testing**:
- `Integration (real): apply timescale DDL` → `SELECT * FROM timescaledb_information.hypertables` lists 4 hypertables.
- `Integration: insert 1000 metric_samples across 30 days, refresh metrics_daily` → `metrics_daily` row count matches distinct (day, repo, metric) tuples.
- `Integration: query repo_health_daily for a repo` → returns one row per day with coverage/complexity/duplication populated.

#### 1.6 — Seed data: built-in metric definitions and starter rules

**What**: Idempotent seed inserting ISO/IEC-25010-aligned metric definitions and an initial rule catalogue.

**Design**:
- `metric_definitions` seeded with: `cyclomatic_complexity` (complexity, -1), `cognitive_complexity` (complexity, -1), `coverage_pct` (coverage, +1), `duplication_pct` (duplication, -1), `maintainability_index` (maintainability, +1), `code_smells` (maintainability, -1), `vulnerabilities` (security, -1), `security_hotspots` (security, 0), `lines_of_code` (size, 0), `technical_debt_minutes` (maintainability, -1).
- Each metric maps to an ISO/IEC 25010 characteristic stored in a `iso25010_characteristic` column.
- Starter rules cover a cross-language smell/security set, each mapped to CWE ids where applicable (e.g. duplicated-string → CWE-1041; hardcoded-secret → CWE-798).
- Seed is upsert-by-`key` so re-running is safe.

**Testing**:
- `Integration: run seed twice` → second run inserts 0 new rows (idempotent), counts unchanged.
- `Unit: every seeded metric has a valid domain and direction` per `packages/types`.
- `Unit: every security rule has at least one cweId`.

---

## Phase 2: Analysis Engine (Core Value)

### Purpose
Build the language-agnostic engine that turns source code into metrics and findings: cyclomatic and cognitive complexity, Halstead volume, the Maintainability Index, duplication detection, basic code smells, and coverage-report parsing. This is the heart of the product and ships early. After this phase, a function can take a directory of source files and produce a complete in-memory `AnalysisResult` without any database or API.

### Tasks

#### 2.1 — Language adapters via tree-sitter

**What**: A `LanguageAdapter` registry that parses source files into ASTs for a set of MVP languages (TypeScript/JavaScript, Python, Java, Go).

**Design**:
```typescript
export interface LanguageAdapter {
  id: string;                          // 'typescript'
  extensions: string[];                // ['.ts','.tsx']
  parse(source: string): Tree;         // web-tree-sitter Tree
  functionNodes(tree: Tree): SyntaxNode[];   // function/method declarations
  decisionPoints(node: SyntaxNode): SyntaxNode[]; // if/for/while/case/&&/||/?: /catch
  operatorsAndOperands(node: SyntaxNode): { operators: string[]; operands: string[] };
}
export function adapterForPath(path: string): LanguageAdapter | undefined;
```
- Grammars loaded from `web-tree-sitter` WASM bundles shipped in `packages/core/grammars/`.
- `decisionPoints` enumerates the McCabe-relevant constructs per language (mapping documented in code comments referencing McCabe 1976).

**Testing**:
- `Unit: adapterForPath('a.py')` → Python adapter; `adapterForPath('a.unknown')` → undefined.
- `Fixture: parse a known TS file` → `functionNodes` count equals hand-counted functions in the fixture.
- `Unit: decisionPoints on 'if (a && b) {}'` → 2 (the `if` + the `&&`).

#### 2.2 — Cyclomatic & cognitive complexity

**What**: McCabe cyclomatic complexity and cognitive complexity per function and aggregated per file.

**Design**:
- `cyclomatic(fnNode, adapter) = 1 + adapter.decisionPoints(fnNode).length`.
- Cognitive complexity follows the SonarSource increment+nesting model (increment on each control-flow break, extra increment per nesting level).
```typescript
export interface ComplexityResult {
  perFunction: { name: string; line: number; cyclomatic: number; cognitive: number }[];
  fileCyclomaticAvg: number; fileCyclomaticMax: number;
  highComplexityCount: number; // functions with cyclomatic > 10 (configurable threshold)
}
export function analyzeComplexity(tree: Tree, adapter: LanguageAdapter, opts?: { threshold?: number }): ComplexityResult;
```
- Default high-risk threshold 10 (McCabe), overridable via repo config.

**Testing**:
- `Unit: a function with one if/else` → cyclomatic 2.
- `Unit: nested if inside if` → cognitive > cyclomatic (nesting penalty applied).
- `Unit: switch with 4 cases` → cyclomatic 5.
- `Fixture: file with 3 functions, one over threshold` → `highComplexityCount === 1`.

#### 2.3 — Halstead metrics & Maintainability Index

**What**: Halstead volume/difficulty/effort and the composite Maintainability Index (0–100).

**Design**:
- Halstead from `operatorsAndOperands`: n1 (distinct operators), n2 (distinct operands), N1, N2; `volume = (N1+N2) * log2(n1+n2)`.
- Maintainability Index (Oman & Hagemeister / Visual Studio normalised form):
  `MI = max(0, (171 - 5.2*ln(volume) - 0.23*cyclomatic - 16.2*ln(LOC)) * 100 / 171)`.
```typescript
export function halstead(tokens: { operators: string[]; operands: string[] }): { volume: number; difficulty: number; effort: number };
export function maintainabilityIndex(volume: number, cyclomatic: number, loc: number): number; // 0..100
```

**Testing**:
- `Unit: empty file` → MI clamps to a valid 0..100 number, no NaN/Infinity.
- `Unit: volume monotonic` → adding distinct operands increases volume.
- `Unit: high complexity + high LOC` → MI lower than a small simple file (ordering assertion).

#### 2.4 — Duplication detection

**What**: Token-based clone detection producing duplicated-line percentage and duplicated blocks.

**Design**:
- Rolling-hash (Rabin–Karp) over normalised token windows (min block size default 100 tokens / ~10 lines, configurable). Identical windows across files form duplication blocks.
```typescript
export interface DuplicationResult {
  duplicatedLines: number; totalLines: number; duplicatedLinesPct: number;
  blocks: { hash: string; locations: { path: string; startLine: number; endLine: number }[] }[];
}
export function detectDuplication(files: { path: string; tokens: string[]; lineMap: number[] }[], opts?: { minBlockTokens?: number }): DuplicationResult;
```
- Normalisation strips whitespace/comments; identifiers optionally normalised (Type-2 clones) behind a config flag.

**Testing**:
- `Unit: two files with an identical 12-line block` → one block, two locations.
- `Unit: block below minBlockTokens` → not reported.
- `Fixture: repo with known 6% duplication` → `duplicatedLinesPct` within ±0.5%.

#### 2.5 — Code smell rules engine

**What**: A pluggable rule engine running AST-pattern rules to produce `Finding`s, seeded with a starter rule set.

**Design**:
```typescript
export interface Rule {
  key: string; type: FindingType; severity: Severity; cweIds: number[];
  appliesTo(adapterId: string): boolean;
  check(ctx: { tree: Tree; adapter: LanguageAdapter; path: string; source: string }): RawFinding[];
}
export interface RawFinding { ruleKey: string; line: number; endLine?: number; column?: number; message: string; effortMinutes?: number; }
export function runRules(rules: Rule[], ctx): Finding[]; // attaches severity/type/cwe + computes findingHash
```
- Starter rules: long-method (LOC > N), too-many-params, duplicated-string-literal (CWE-1041), empty-catch (CWE-390-ish), hardcoded-secret regex (CWE-798), TODO/FIXME density.
- `findingHash = sha256(ruleKey + ':' + normalizedPath + ':' + normalizedSnippet)` — stable across line shifts (snippet-based, not line-based) so the same issue dedups across analyses.

**Testing**:
- `Unit: file with a 200-line function` → long-method finding at the right line.
- `Unit: hardcoded AWS key string` → hardcoded-secret finding with cweIds including 798.
- `Unit: findingHash stable` → same issue at a shifted line produces the same hash.
- `Unit: rule.appliesTo` filters Python-only rules out of a TS file.

#### 2.6 — Coverage report parsers

**What**: Parse standard coverage formats into per-component coverage.

**Design**:
- Support Cobertura XML, LCOV, and JaCoCo XML (covering JS/Py/Java/Go ecosystems).
```typescript
export interface CoverageReport { components: { path: string; linesToCover: number; linesCovered: number; branchesToCover: number; branchesCovered: number; }[]; }
export function parseCoverage(format: 'cobertura'|'lcov'|'jacoco', content: string): CoverageReport;
export function detectCoverageFormat(content: string): 'cobertura'|'lcov'|'jacoco'|undefined;
```

**Testing**:
- `Fixture: sample LCOV` → component count and line totals match the file.
- `Fixture: Cobertura with branch data` → branch coverage parsed.
- `Unit: malformed XML` → throws a typed `CoverageParseError`, not a raw exception.

#### 2.7 — Analysis orchestrator

**What**: Compose the analyzers into a single `analyzeRepository` producing a complete `AnalysisResult`.

**Design**:
```typescript
export interface AnalysisResult {
  components: ComponentResult[];
  repositoryMetrics: Record<string, number>; // metric_key -> value (averages/maxes/pcts)
  findings: Finding[];
  coverage?: CoverageReport;
}
export async function analyzeRepository(opts: {
  rootDir: string; include?: string[]; exclude?: string[];
  coverage?: { format: string; path: string };
  thresholds?: { complexity?: number };
}): Promise<AnalysisResult>;
```
- Walks files honouring include/exclude globs; per file runs complexity → halstead/MI → smells; runs duplication across all files; merges coverage by path; aggregates repository-level metrics (avg/max complexity, duplication %, coverage %, MI, debt minutes from summed `effortMinutes`).

**Testing**:
- `Integration: analyzeRepository on tests/fixtures/sample-ts-repo` → expected file count, findings count, and aggregate metrics within tolerance (golden snapshot).
- `Integration: exclude glob 'node_modules/**'` → excluded files absent from components.
- `Integration: no coverage provided` → `coverage` undefined, `coverage_pct` not emitted.

---

## Phase 3: Persistence & SARIF Interchange

### Purpose
Persist analysis results into the relational + time-series schema and make SARIF a first-class import/export format so the dashboard can aggregate findings from any external SAST tool. After this phase, an `AnalysisResult` (from Phase 2) or an external SARIF file can be ingested into the database, and the same data can be exported back out as SARIF 2.1.0.

### Tasks

#### 3.1 — Analysis ingestion repository

**What**: Transactional writer that persists an `AnalysisResult` across relational and hypertable storage.

**Design**:
```typescript
export async function ingestAnalysis(input: {
  repositoryId: string; branch: string; commitSha: string; triggeredBy: string;
  result: AnalysisResult; analyzedAt: Date;
}): Promise<{ analysisId: string }>;
```
- In one Drizzle transaction: upsert `components` by `(repository_id, path)`; insert `analyses` row with summary columns + `metrics_detail`; bulk-insert `metric_samples` (one row per component+metric and per repo-level metric), `coverage_samples`, and `finding_events` (`event_type='detected'`).
- Finding lifecycle: compare current `findingHash` set against the previous completed analysis for the branch; emit `resolved` events for hashes absent now, `reopened` for previously-resolved hashes that returned.
- Hypertable inserts use raw parameterised batched SQL (`postgres.js` `.unsafe`/multi-row insert) for throughput.

**Testing**:
- `Integration (Testcontainers): ingest a result` → `analyses` row present; `metric_samples` count = components×metrics + repo metrics.
- `Integration: ingest twice with one finding removed` → a `resolved` finding_event emitted for the missing hash.
- `Integration: transaction rolls back` if a hypertable insert fails → no partial `analyses` row remains.

#### 3.2 — SARIF import

**What**: Parse and validate SARIF 2.1.0, map results to `Finding`s, and ingest.

**Design**:
- Validate against the SARIF 2.1.0 JSON Schema (Draft 2020-12) before processing; reject with descriptive errors otherwise.
```typescript
export function parseSarif(doc: unknown): { tool: string; toolVersion?: string; findings: Finding[]; raw: object };
```
- Map `run.results[]` → `Finding`: `ruleId`→`ruleKey`, `level`(error/warning/note)→`severity`, `locations[0].physicalLocation`→ path/line, `message.text`→message; pull CWE ids from `rule.relationships`/`taxa` where present.
- Store the full SARIF run in a `sarif_imports` table (`sarif_run JSONB`) for traceability, then route findings through the same finding-event path as 3.1.

**Testing**:
- `Fixture: CodeQL SARIF sample` → findings count equals `results` length; severities mapped correctly.
- `Unit: SARIF missing 'version'` → validation error naming the field.
- `Integration: import SARIF then export` (3.3) → round-trip preserves ruleId, level, location.

#### 3.3 — SARIF export

**What**: Emit a repository's findings for an analysis as a valid SARIF 2.1.0 document.

**Design**:
```typescript
export function toSarif(input: { toolName: string; toolVersion: string; rules: Rule[]; findings: Finding[] }): SarifLog;
```
- Build `tool.driver.rules[]` from referenced rules (with `helpUri`, `properties.cwe`), `results[]` with `ruleId`, `level`, `message`, `locations`, `partialFingerprints.findingHash` for stable cross-run identity.
- Validate output against the SARIF schema before returning.

**Testing**:
- `Unit: toSarif output validates` against the SARIF 2.1.0 schema.
- `Unit: each result references a rule present in tool.driver.rules`.
- `Unit: partialFingerprints.findingHash === finding.findingHash`.

#### 3.4 — Trend & query repository

**What**: Typed read functions over continuous aggregates powering the dashboard and API.

**Design**:
```typescript
export async function metricTrend(p: { repositoryId: string; metricKey: string; from: Date; to: Date; resolution: 'daily'|'weekly' }): Promise<{ bucket: string; value: number }[]>;
export async function repoHealthLatest(repositoryId: string): Promise<RepoHealth>;
export async function findingBurndown(p: { repositoryId: string; from: Date; to: Date }): Promise<{ bucket: string; detected: number; resolved: number; net: number }[]>;
export async function crossRepoBenchmark(organizationId: string): Promise<RepoHealth[]>;
```
- Implemented as parameterised raw SQL against `metrics_daily`/`metrics_weekly`/`repo_health_daily`/`findings_daily_summary` (queries adapted from suggestion-4).

**Testing**:
- `Integration: seed 90 days of metric_samples, refresh aggregates, call metricTrend` → 90 (or 13 weekly) ordered buckets.
- `Integration: findingBurndown` → detected/resolved/net reconcile to raw event counts.
- `Unit: from > to` → throws a validation error.

---

## Phase 4: REST API, Auth & OpenAPI

### Purpose
Expose the persisted data and ingestion capabilities over an HTTP API conforming to RFC 7231 semantics, with OAuth 2.0 + API-token auth and an auto-generated OpenAPI 3.1 spec. After this phase, external clients (CI pipelines, the web app, partner integrations) can authenticate, submit analyses/SARIF, and read metrics and trends.

### Tasks

#### 4.1 — Fastify server, error model, OpenAPI

**What**: Bootstrap the Fastify app with JSON Schema validation, RFC 7807 problem-details errors, request logging, and `@fastify/swagger` OpenAPI 3.1 generation at `/openapi.json` + Swagger UI at `/docs`.

**Design**:
- Per-route `schema: { body, querystring, params, response }` using JSON Schema Draft 2020-12.
- Error handler returns `application/problem+json` (`{ type, title, status, detail, instance }`).
- Pagination via RFC 8288 `Link` headers (`rel="next"/"prev"`) plus `?limit&cursor`.

**Testing**:
- `Integration: GET /openapi.json` → valid OpenAPI 3.1 document (validated with an OAS validator).
- `Integration: POST with invalid body` → 400 problem+json naming the offending field.
- `Integration: unknown route` → 404 problem+json.

#### 4.2 — Authentication & authorization

**What**: OAuth 2.0 login (GitHub + GitLab), JWT session cookies, and scoped API tokens for CI.

**Design**:
- `GET /auth/:provider/start` → redirect; `GET /auth/:provider/callback` → exchange code, upsert `users`, set signed JWT cookie.
- API tokens: `POST /orgs/:id/tokens` creates a token (stored hashed, shown once), scopes `analysis:write`, `read`. Sent as `Authorization: Bearer <token>` (per standards.md convention used by SonarQube/CodeScene).
- RBAC roles: `owner`, `admin`, `member`, `viewer`; route guards check org membership + role.

**Testing**:
- `Integration (mocked OAuth): callback with valid code` → user upserted, JWT set, 302.
- `Integration: request with revoked token` → 401.
- `Integration: member hits an owner-only route` → 403.
- `Unit: tokens stored hashed` → raw token never persisted.

#### 4.3 — Repository & analysis endpoints

**What**: CRUD for repositories and analysis submission/read endpoints.

**Design** (selected routes; all documented in OpenAPI):
```
POST   /orgs/:orgId/repositories            -> create repo
GET    /repositories/:id                     -> repo + latest health
POST   /repositories/:id/analyses            -> submit AnalysisResult (enqueues ingest)  [analysis:write]
POST   /repositories/:id/analyses/sarif      -> upload SARIF file (multipart)            [analysis:write]
GET    /repositories/:id/analyses            -> paginated analysis list
GET    /analyses/:id                          -> analysis detail + metrics_detail
GET    /analyses/:id/sarif                     -> export findings as SARIF 2.1.0
GET    /repositories/:id/metrics/:metricKey/trend?from&to&resolution
GET    /repositories/:id/findings?status&severity&type   (paginated)
PATCH  /findings/:id                           -> update status/assignee/resolution
GET    /orgs/:orgId/benchmark                  -> cross-repo benchmark
```
- Submission endpoints enqueue a BullMQ job and return `202 Accepted` with a `Location` to the analysis resource.

**Testing**:
- `Integration: POST analysis` → 202, job enqueued, analysis row in `pending`.
- `Integration: GET trend` → ordered buckets, correct `Link` pagination when applicable.
- `Integration: PATCH finding to 'false_positive'` → status persisted, finding_event emitted.
- `E2E: submit SARIF → poll analysis → GET /analyses/:id/sarif` round-trips findings.

#### 4.4 — Webhook receiver

**What**: Endpoint receiving CI/SCM webhooks (GitHub/GitLab push & PR) to trigger analyses.

**Design**:
- `POST /webhooks/:provider` verifies HMAC signature (`X-Hub-Signature-256` for GitHub), then enqueues an analysis job for the pushed commit.
- Idempotency by delivery id; duplicate deliveries are no-ops.

**Testing**:
- `Integration (mocked): valid signature` → 200, job enqueued.
- `Integration: invalid signature` → 401, no job enqueued.
- `Integration: duplicate delivery id` → 200, single job only.

---

## Phase 5: Async Workers & Quality Gates

### Purpose
Move long-running work (analysis ingestion, SARIF processing, AI calls, aggregate refresh) onto BullMQ workers, and implement the quality-gate engine that produces pass/warn/fail decisions for CI. After this phase, submitted analyses are processed asynchronously, gate results are computed and persisted, and CI can block merges on gate failure.

### Tasks

#### 5.1 — Queue infrastructure

**What**: BullMQ queues, a worker app, and a job-status surface.

**Design**:
- Queues: `analysis-ingest`, `sarif-import`, `ai-insights`, `aggregate-refresh`.
- `apps/worker` registers processors; concurrency configurable per queue; exponential backoff retries (max 3) with dead-letter handling.
- Job payloads are Zod-validated `packages/types` DTOs.

**Testing**:
- `Integration (Testcontainers Redis): enqueue + process a job` → state transitions waiting→active→completed.
- `Integration: processor throws` → retried per backoff, lands in failed after max attempts.
- `Unit: invalid job payload` → rejected before processing.

#### 5.2 — Analysis & SARIF processors

**What**: Workers that run `analyzeRepository` (clone/checkout) or `parseSarif`, then `ingestAnalysis`, updating analysis status.

**Design**:
- `analysis-ingest` processor: set `analyses.status='running'`, clone repo at `commitSha` (shallow), run engine, ingest, set `completed`/`failed`, then enqueue `ai-insights` + `aggregate-refresh`.
- `sarif-import` processor: parse/validate, ingest findings.
- Repo cloning isolated to a temp dir, cleaned up in `finally`.

**Testing**:
- `Integration: end-to-end on a local fixture repo` → analysis reaches `completed`, metrics queryable.
- `Integration: clone failure` → analysis `failed` with error captured, no orphaned temp dir.

#### 5.3 — Quality gate engine

**What**: Evaluate configurable conditions against an analysis and record the result.

**Design** (conditions as JSONB per suggestion-3/4):
```typescript
export interface GateCondition { metric: string; operator: 'LT'|'GT'|'EQ'|'NE'; error: number; warn?: number; onNewCodeOnly?: boolean; }
export function evaluateGate(conditions: GateCondition[], metrics: Record<string, number>): { status: 'passed'|'warned'|'failed'; conditions: EvaluatedCondition[] };
```
- Writes a `gate_evaluations` row and sets `analyses.quality_gate_status`.
- Default gate: coverage ≥ 80, vulnerabilities = 0, duplication ≤ 5%, maintainability rating ≥ B.

**Testing**:
- `Unit: coverage 72 vs LT 80 error` → failed condition.
- `Unit: one warn + zero error` → overall `warned`.
- `Integration: gate result persisted` → `gate_evaluations` row + `analyses.quality_gate_status` set.

#### 5.4 — Aggregate refresh & scheduled jobs

**What**: Trigger continuous-aggregate refresh and run scheduled hotspot recomputation.

**Design**:
- `aggregate-refresh` calls `CALL refresh_continuous_aggregate(...)` for affected windows after ingest (real-time aggregation otherwise covers gaps).
- Repeatable BullMQ job nightly recomputes hotspot risk scores (uses `regr_slope` complexity trend, change frequency, finding density) into a `proj_code_hotspots` table.

**Testing**:
- `Integration: after ingest, daily aggregate reflects new analysis` within the refresh window.
- `Integration: nightly hotspot job` → `proj_code_hotspots` ranked by `risk_score` desc.

---

## Phase 6: Web Dashboard

### Purpose
Deliver the user-facing dashboard — the product's primary surface — covering repository health overview, trend charts, finding/issue management, quality-gate status, and cross-repo benchmarking. After this phase, users can log in, connect repositories, and see code-health trends over time.

### Tasks

#### 6.1 — App shell, auth, and org/repo navigation

**What**: Next.js App Router shell with OAuth login, org switcher, and repository list.

**Design**:
- Server components fetch via the API using the session JWT; middleware redirects unauthenticated users to `/login`.
- Routes: `/`, `/orgs/:org`, `/repos/:id`, `/repos/:id/findings`, `/repos/:id/trends`, `/benchmark`.

**Testing**:
- `E2E (Playwright): unauthenticated visit to /repos/x` → redirected to /login.
- `E2E: login flow (mocked OAuth)` → lands on org dashboard with repo list.

#### 6.2 — Repository health overview

**What**: Per-repo cards showing current coverage, complexity, duplication, maintainability rating, finding counts, gate status, and tech-debt estimate.

**Design**: Consumes `GET /repositories/:id` (latest health). Rating badges A–E derived from thresholds. Gate status banner (green/amber/red).

**Testing**:
- `E2E: repo with seeded analysis` → all metric tiles render with expected values.
- `E2E: failing gate` → red banner shown.

#### 6.3 — Trend charts

**What**: Interactive multi-metric time-series charts (coverage, complexity, duplication, MI) with daily/weekly resolution and sprint comparison.

**Design**: Tremor/Recharts line charts fed by `GET /metrics/:key/trend`; date-range and resolution selectors; finding burn-down chart from `findingBurndown`.

**Testing**:
- `E2E: select 90-day range` → chart renders correct number of points.
- `E2E: switch daily→weekly` → fewer points, axis updates.

#### 6.4 — Findings/issue management

**What**: Filterable findings table with assignment and status changes.

**Design**: `GET /repositories/:id/findings` with filters; row actions call `PATCH /findings/:id`; severity/type facets; CWE links to MITRE.

**Testing**:
- `E2E: filter to critical vulnerabilities` → table shows only matching rows.
- `E2E: mark finding false_positive` → row updates, count decrements.

#### 6.5 — Cross-repo benchmark view

**What**: Organisation-wide table/heatmap ranking repos by maintainability and coverage.

**Design**: `GET /orgs/:orgId/benchmark`; sortable columns; colour-coded cells.

**Testing**:
- `E2E: benchmark page` → one row per active repo, sortable by maintainability.

---

## Phase 7: AI-Native Features

### Purpose
Implement the differentiating AI layer: plain-English explanations of high-complexity/duplicated files, automatic technical-debt prioritisation, executive trend summaries, and flagging of AI-generated code patterns correlated with defects. After this phase, the dashboard turns raw metrics into prioritised, explained action.

### Tasks

#### 7.1 — LLM provider abstraction & prompt library

**What**: A provider-agnostic LLM client with typed prompt templates and prompt caching.

**Design**:
```typescript
export interface LlmProvider { complete(p: { system: string; messages: Msg[]; maxTokens: number; cacheKey?: string }): Promise<{ text: string; usage: Usage }>; }
export const explainComplexityPrompt: (ctx: { path: string; metrics: object; changeHistory: object; snippet: string }) => { system: string; user: string };
```
- Anthropic implementation uses prompt caching for stable repo context blocks. Outputs validated against Zod schemas; stored in `ai_insights` (JSONB content, `confidence`, `model_version`) per suggestion-3.

**Testing**:
- `Unit (mocked LLM): explanation returned` → parsed into the `ai_insights` content shape.
- `Unit: malformed LLM JSON` → repaired-or-rejected path, no crash.

#### 7.2 — Complexity/hotspot explanations

**What**: Generate explanations for why a file is high-risk, contextualised by purpose and change history.

**Design**: `ai-insights` worker selects top-risk components, builds prompt with metrics + git change frequency + snippet, stores `insight_type='explanation'` with `risk_factors[]` and `recommended_actions[]`.

**Testing**:
- `Integration (mocked LLM): hotspot file` → insight row with ≥1 recommended action and a confidence in [0,1].

#### 7.3 — Technical-debt prioritisation

**What**: Rank debt items by correlating quality scores with finding density, change frequency, and severity.

**Design**: Compute a debt score per component; LLM produces a sequenced remediation plan (`insight_type='prioritization'`). Surfaced via `GET /repositories/:id/debt-plan`.

**Testing**:
- `Integration: two files, one hot+complex` → ranked first in the plan.
- `Unit: scoring deterministic` for fixed inputs (pre-LLM ranking).

#### 7.4 — Executive trend summaries & AI-code flagging

**What**: Generate sprint-over-sprint narrative summaries; flag components matching AI-generated-code patterns statistically linked to defects.

**Design**: Summary worker reads `repo_health_daily` deltas → `insight_type='summary'`. AI-code flag heuristic (duplication spike + churn + low coverage on recently-added code) tagged on components; surfaced as a finding type.

**Testing**:
- `Integration (mocked LLM): two sprints of data` → summary references the largest metric delta.
- `Unit: AI-code heuristic` → component meeting all three criteria is flagged; one missing a criterion is not.

---

## Phase 8: CLI Scanner & CI/CD Integration

### Purpose
Provide the local/CI scanner (`cqd`) that engineering teams run in pipelines to submit analyses and enforce quality gates, plus ready-made CI integrations. After this phase, a `cqd scan` step in CI can fail a build on gate breach.

### Tasks

#### 8.1 — `cqd` CLI

**What**: A CLI that runs the analysis engine locally and submits results to the API.

**Design**:
```
cqd scan [--dir .] [--coverage cobertura:cov.xml] [--token $CQD_TOKEN] [--server URL] [--branch] [--commit]
cqd gate [--wait]    # poll analysis, exit non-zero on gate failure
cqd login            # device-flow OAuth for local use
```
- Reads `.cqd.yml` (include/exclude globs, thresholds, gate name); validated with Zod.
- `scan` runs `analyzeRepository`, POSTs to `/repositories/:id/analyses`, prints a summary table.

**Testing**:
- `E2E: cqd scan on fixture repo (mocked API)` → posts correct payload, exit 0.
- `Unit: .cqd.yml invalid` → descriptive error, non-zero exit.

#### 8.2 — Gate enforcement & exit codes

**What**: `cqd gate` waits for analysis completion and returns CI-friendly exit codes.

**Design**: Polls `GET /analyses/:id` until terminal; exit 0 passed, 1 failed, 2 warned-as-error (configurable), 3 error. Emits a GitHub Actions annotation format when `GITHUB_ACTIONS=true`.

**Testing**:
- `Integration (mocked API): gate failed` → exit 1.
- `Integration: warned with --warn-as-error` → exit 2.

#### 8.3 — CI templates & SARIF upload to GitHub

**What**: Ship a GitHub Action and GitLab CI snippet; optionally upload SARIF to GitHub code scanning.

**Design**: `action.yml` wrapping `cqd scan && cqd gate`; optional `github/codeql-action/upload-sarif` step using the exported SARIF from `/analyses/:id/sarif`.

**Testing**:
- `Fixture: rendered workflow YAML` validates against the Actions schema.
- `Integration (mocked GitHub API): SARIF upload` → 202 from the code-scanning endpoint.

---

## Phase 9: MCP Server (AI Assistant Access)

### Purpose
Expose code-quality data to AI assistants over the Model Context Protocol (spec 2025-11-25), a differentiator called out in `standards.md`. After this phase, an AI assistant can answer "which files are most at risk this sprint?" directly against the dashboard.

### Tasks

#### 9.1 — MCP server & tools

**What**: An MCP server exposing read tools over the metrics/trends/hotspots/findings APIs.

**Design** (`@modelcontextprotocol/sdk`, JSON-RPC 2.0):
- Tools: `list_repositories`, `get_repository_health(repoId)`, `get_metric_trend(repoId, metricKey, range)`, `list_hotspots(repoId)`, `list_findings(repoId, filters)`, `get_debt_plan(repoId)`.
- Auth via an API token passed in server config; tools call the REST API internally.
- Tool schemas defined with Zod → JSON Schema.

**Testing**:
- `Integration: MCP client lists tools` → all six present with valid input schemas.
- `Integration: call get_repository_health` → returns latest health JSON.
- `Integration: call with bad repoId` → MCP error response, not a crash.

#### 9.2 — Resources & prompts

**What**: Expose recent analyses as MCP resources and ship a `quality-review` prompt template.

**Design**: Resource URIs `cqd://repo/{id}/latest`; a prompt that instructs the assistant to review hotspots + failing gate conditions and propose fixes.

**Testing**:
- `Integration: read resource cqd://repo/{id}/latest` → latest analysis summary.
- `Integration: get prompt 'quality-review'` → returns templated messages.

---

## Phase 10: Hardening, Observability & Release

### Purpose
Make the system production-ready: OpenTelemetry instrumentation (DORA-metric-friendly per standards.md), security hardening aligned to OWASP/NIST SSDF, full Docker Compose deployment, and release packaging. After this phase, the project is self-hostable and documented end-to-end.

### Tasks

#### 10.1 — Observability

**What**: OpenTelemetry traces/metrics/logs across API and workers.

**Design**: OTEL SDK auto-instrumentation; custom spans around analysis ingest and LLM calls; metrics exported to Prometheus; optional DORA-metric panel (change failure rate from gate failures).

**Testing**:
- `Integration: an API request emits a trace` with expected span names.
- `Integration: /metrics` Prometheus endpoint returns counters.

#### 10.2 — Security hardening

**What**: Secrets management, rate limiting, input hardening, dependency scanning.

**Design**: `@fastify/rate-limit`, helmet headers, token hashing (already in 4.2), `npm audit`/Trivy in CI. Map the project's own rules to OWASP SCP and NIST SSDF practice IDs in docs.

**Testing**:
- `Integration: exceed rate limit` → 429 with `Retry-After`.
- `Integration: security headers present` on responses.
- `CI: dependency scan` gate fails on a known critical CVE fixture.

#### 10.3 — Full deployment & docs

**What**: Complete `docker-compose.yml` (db, redis, api, worker, web, mcp) and operator/user documentation.

**Design**: Production compose with healthchecks, restart policies, and a one-command bootstrap that runs migrations + timescale DDL + seed. `README` quickstart, `.cqd.yml` reference, API guide pointing to `/docs`.

**Testing**:
- `E2E (real): docker compose up` from clean → all services healthy; submit a fixture analysis via CLI → appears in the web dashboard.
- `Smoke: fresh DB bootstrap` → migrations + timescale DDL + seed all succeed in order.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (monorepo, DB, types)        ─── required by everything
    │
Phase 2: Analysis Engine (core value)            ─── requires Phase 1 (types only; no DB)
    │
Phase 3: Persistence & SARIF                      ─── requires Phases 1 + 2
    │
Phase 4: REST API, Auth, OpenAPI                  ─── requires Phase 3
    │
Phase 5: Workers & Quality Gates                  ─── requires Phase 4
    │
    ├── Phase 6: Web Dashboard                    ─── requires Phase 4 (reads), 5 (gates) ─ parallel with 8, 9
    ├── Phase 7: AI-Native Features               ─── requires Phase 5 ──────────────────── parallel with 6
    ├── Phase 8: CLI & CI/CD                       ─── requires Phase 4 ──────────────────── parallel with 6, 7
    └── Phase 9: MCP Server                        ─── requires Phase 4 ──────────────────── parallel with 6, 7, 8
         │
Phase 10: Hardening, Observability & Release      ─── requires all prior phases
```

**Parallelism opportunities:**
- Phase 2 needs only Phase 1's `types` package — the analysis engine can be built before or alongside the DB work in Phase 1.
- Phases 6, 7, 8, and 9 all depend only on the API (Phase 4) plus gates (Phase 5 for 6/7) and can be developed concurrently by separate contributors.

---

## Definition of Done (per phase)

Every phase must satisfy all of the following before it is considered complete:

1. All tasks in the phase implemented.
2. All unit and integration tests for the phase pass (`turbo test`).
3. Linting and formatting pass (`turbo lint`, Prettier check).
4. Type checking passes (`tsc --noEmit` across affected packages, `turbo typecheck`).
5. Relevant Testcontainers integration tests pass against real PostgreSQL+TimescaleDB / Redis.
6. New or changed API endpoints appear in the auto-generated OpenAPI 3.1 spec and validate.
7. New config options (env vars, `.cqd.yml`, repo `config` JSONB) are documented in `.env.example` / docs.
8. Database changes ship as Drizzle migrations (relational) and ordered Timescale DDL (hypertables/aggregates); a fresh-DB bootstrap applies them cleanly.
9. Docker build succeeds for any app touched in the phase.
10. The phase's headline capability works end-to-end (demonstrated by an integration or e2e test using committed fixtures).
11. SARIF/OpenAPI/MCP outputs validate against their respective schemas where the phase produces them.
