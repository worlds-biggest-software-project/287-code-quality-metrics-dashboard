# Data Model Suggestion 3: Hybrid Relational + JSONB/Document Approach

> Project: Code Quality Metrics Dashboard (Candidate #287)
> Approach: PostgreSQL with relational core tables and JSONB columns for flexible/variable data

---

## Summary

A pragmatic hybrid design that uses normalized relational tables for high-query entities (organizations, repositories, analyses, rules) and PostgreSQL JSONB columns for semi-structured, variable, or rapidly evolving data (metric payloads, finding details, SARIF import data, scanner configurations, AI-generated insights). This balances the referential integrity and query performance of relational modeling with the schema flexibility of document stores -- all within a single PostgreSQL database, avoiding the operational overhead of a separate document database.

This approach reflects the "relational core + JSONB extras" pattern that has become the dominant pragmatic choice for analytics-oriented applications built on PostgreSQL 17+.

---

## Key Entities and Relationships

### Design Principles

1. **Relational columns** for data that is frequently filtered, joined, indexed, or aggregated (IDs, timestamps, statuses, numeric metric values).
2. **JSONB columns** for data that varies by context, is schema-flexible, or is consumed as a unit (scanner output details, rule parameters, SARIF fragments, AI explanations).
3. **GIN indexes** on JSONB columns that support query patterns.
4. **Generated columns** to extract frequently-queried JSONB fields into indexable relational columns.

### Core Schema

```sql
-- === ORGANIZATIONAL STRUCTURE (fully relational) ===

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(128) NOT NULL UNIQUE,
    settings        JSONB NOT NULL DEFAULT '{}',  -- org-level config: default gates, notification prefs
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE repositories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    url             TEXT,
    default_branch  VARCHAR(255) DEFAULT 'main',
    primary_language VARCHAR(64),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- JSONB for variable repository metadata
    metadata        JSONB NOT NULL DEFAULT '{}',  -- languages[], framework, build_tool, ci_provider, etc.
    config          JSONB NOT NULL DEFAULT '{}',  -- scanner config, exclusion patterns, custom thresholds
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, name)
);

-- Extract primary_language from metadata for indexing if populated there
CREATE INDEX idx_repos_org ON repositories (organization_id);
CREATE INDEX idx_repos_metadata ON repositories USING GIN (metadata jsonb_path_ops);

-- === ANALYSES (relational + JSONB details) ===

CREATE TABLE analyses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id),
    branch          VARCHAR(255) NOT NULL,
    commit_sha      VARCHAR(40) NOT NULL,
    status          VARCHAR(32) NOT NULL DEFAULT 'pending',
    triggered_by    VARCHAR(64),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    -- Aggregated summary metrics (relational for fast dashboard queries)
    total_files     INTEGER,
    total_lines     INTEGER,
    findings_count  INTEGER DEFAULT 0,
    coverage_pct    DOUBLE PRECISION,
    avg_complexity  DOUBLE PRECISION,
    duplication_pct DOUBLE PRECISION,
    maintainability_idx DOUBLE PRECISION,
    debt_minutes    INTEGER DEFAULT 0,
    quality_gate_status VARCHAR(32),
    -- JSONB for detailed breakdown and scanner output
    metrics_detail  JSONB NOT NULL DEFAULT '{}',  -- full metric breakdown by domain
    quality_gate_detail JSONB,                     -- gate conditions and results
    scanner_info    JSONB,                         -- scanner version, duration, warnings
    sarif_summary   JSONB,                         -- compressed SARIF metadata
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_analyses_repo ON analyses (repository_id, created_at DESC);
CREATE INDEX idx_analyses_status ON analyses (status);
CREATE INDEX idx_analyses_branch ON analyses (repository_id, branch, created_at DESC);

-- Example of what metrics_detail JSONB contains:
-- {
--   "complexity": {
--     "cyclomatic_avg": 8.3,
--     "cyclomatic_max": 47,
--     "cyclomatic_p90": 15,
--     "high_complexity_files": 23,
--     "cognitive_complexity_avg": 6.1
--   },
--   "duplication": {
--     "duplicated_lines_pct": 4.2,
--     "duplicated_blocks": 18,
--     "duplicated_files": 7
--   },
--   "coverage": {
--     "line_coverage_pct": 78.3,
--     "branch_coverage_pct": 62.1,
--     "lines_to_cover": 14200,
--     "lines_covered": 11118
--   },
--   "security": {
--     "vulnerabilities": 3,
--     "security_hotspots": 12,
--     "security_rating": "C"
--   },
--   "maintainability": {
--     "index": 72.4,
--     "rating": "B",
--     "code_smells": 156,
--     "debt_ratio": 2.1
--   }
-- }

-- === COMPONENTS (relational + JSONB metrics) ===

CREATE TABLE components (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id),
    path            TEXT NOT NULL,
    component_type  VARCHAR(32) NOT NULL,
    language        VARCHAR(64),
    lines_of_code   INTEGER,
    -- Current metrics snapshot (updated on each analysis)
    current_metrics JSONB NOT NULL DEFAULT '{}',
    -- Change metadata
    last_modified_at TIMESTAMPTZ,
    change_frequency INTEGER DEFAULT 0,  -- commits in last 90 days
    contributor_count INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (repository_id, path)
);

CREATE INDEX idx_components_repo ON components (repository_id);
CREATE INDEX idx_components_metrics ON components USING GIN (current_metrics jsonb_path_ops);

-- Generated column for fast complexity filtering
ALTER TABLE components ADD COLUMN cyclomatic_complexity DOUBLE PRECISION
    GENERATED ALWAYS AS ((current_metrics->>'cyclomatic_complexity')::DOUBLE PRECISION) STORED;
CREATE INDEX idx_components_complexity ON components (repository_id, cyclomatic_complexity DESC NULLS LAST);

-- === FINDINGS (relational core + JSONB detail) ===

CREATE TABLE rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    key             VARCHAR(255) NOT NULL UNIQUE,
    name            VARCHAR(512) NOT NULL,
    description     TEXT,
    language        VARCHAR(64),
    severity        VARCHAR(32) NOT NULL,
    rule_type       VARCHAR(32) NOT NULL,
    -- JSONB for variable rule metadata
    tags            JSONB NOT NULL DEFAULT '[]',
    cwe_ids         JSONB NOT NULL DEFAULT '[]',
    owasp_ids       JSONB NOT NULL DEFAULT '[]',
    config          JSONB NOT NULL DEFAULT '{}',  -- default parameters, thresholds
    remediation     JSONB,                         -- effort function, examples
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rules_type ON rules (rule_type, severity);
CREATE INDEX idx_rules_tags ON rules USING GIN (tags);

CREATE TABLE findings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    analysis_id     UUID NOT NULL REFERENCES analyses(id),
    component_id    UUID NOT NULL REFERENCES components(id),
    rule_id         UUID NOT NULL REFERENCES rules(id),
    -- Relational for filtering and aggregation
    severity        VARCHAR(32) NOT NULL,
    finding_type    VARCHAR(32) NOT NULL,
    status          VARCHAR(32) NOT NULL DEFAULT 'open',
    resolution      VARCHAR(32),
    line_number     INTEGER,
    effort_minutes  INTEGER,
    assignee_id     UUID,
    -- JSONB for rich context
    location        JSONB NOT NULL DEFAULT '{}',  -- line, column, end_line, end_column, snippet
    message         TEXT NOT NULL,
    context         JSONB,                         -- data flow paths, related locations, call chain
    ai_explanation  JSONB,                         -- AI-generated explanation and fix suggestion
    sarif_data      JSONB,                         -- original SARIF result fragment
    -- Lifecycle
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_findings_analysis ON findings (analysis_id);
CREATE INDEX idx_findings_component ON findings (component_id, status);
CREATE INDEX idx_findings_status ON findings (status, severity, finding_type);
CREATE INDEX idx_findings_context ON findings USING GIN (context jsonb_path_ops);

-- === METRIC HISTORY (relational for time-series queries) ===

CREATE TABLE metric_history (
    id              BIGSERIAL PRIMARY KEY,
    repository_id   UUID NOT NULL REFERENCES repositories(id),
    analysis_id     UUID NOT NULL REFERENCES analyses(id),
    analysis_date   DATE NOT NULL,
    -- Key aggregate metrics as relational columns for efficient trend queries
    coverage_pct    DOUBLE PRECISION,
    avg_complexity  DOUBLE PRECISION,
    max_complexity  DOUBLE PRECISION,
    duplication_pct DOUBLE PRECISION,
    maintainability_idx DOUBLE PRECISION,
    open_findings   INTEGER,
    open_bugs       INTEGER,
    open_vulnerabilities INTEGER,
    open_code_smells INTEGER,
    debt_minutes    INTEGER,
    total_lines     INTEGER,
    -- JSONB for the full metric breakdown (domain-specific details)
    full_metrics    JSONB NOT NULL DEFAULT '{}',
    UNIQUE (repository_id, analysis_id)
);

CREATE INDEX idx_metric_history_repo ON metric_history (repository_id, analysis_date DESC);

-- === COMPONENT-LEVEL METRIC HISTORY ===

CREATE TABLE component_metric_history (
    id              BIGSERIAL PRIMARY KEY,
    component_id    UUID NOT NULL REFERENCES components(id),
    analysis_id     UUID NOT NULL REFERENCES analyses(id),
    analysis_date   DATE NOT NULL,
    -- Key metrics as columns
    cyclomatic_complexity DOUBLE PRECISION,
    cognitive_complexity  DOUBLE PRECISION,
    lines_of_code   INTEGER,
    coverage_pct    DOUBLE PRECISION,
    finding_count   INTEGER DEFAULT 0,
    -- Everything else in JSONB
    metrics         JSONB NOT NULL DEFAULT '{}',
    UNIQUE (component_id, analysis_id)
);

CREATE INDEX idx_comp_hist_component ON component_metric_history (component_id, analysis_date DESC);

-- === QUALITY GATES ===

CREATE TABLE quality_gates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    -- Conditions stored as JSONB array (flexible, easy to extend)
    conditions      JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"metric": "coverage_pct", "operator": "LT", "error": 80, "warn": 85},
    --   {"metric": "open_vulnerabilities", "operator": "GT", "error": 0},
    --   {"metric": "duplication_pct", "operator": "GT", "error": 5, "warn": 3}
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- === AI INSIGHTS (fully JSONB — schema changes frequently) ===

CREATE TABLE ai_insights (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id),
    analysis_id     UUID REFERENCES analyses(id),
    insight_type    VARCHAR(64) NOT NULL,  -- 'explanation', 'prioritization', 'prediction', 'summary'
    target_type     VARCHAR(64),           -- 'component', 'finding', 'repository'
    target_id       UUID,
    -- AI-generated content (schema evolves with model capabilities)
    content         JSONB NOT NULL,
    -- Example for 'explanation':
    -- {
    --   "summary": "This file has high complexity because...",
    --   "risk_factors": ["frequent changes", "low coverage", "deep nesting"],
    --   "recommended_actions": [
    --     {"action": "Extract method", "target": "processData()", "impact": "high"},
    --     {"action": "Add tests", "target": "edge cases in parseInput()", "impact": "medium"}
    --   ],
    --   "confidence": 0.87,
    --   "model_version": "claude-4-sonnet"
    -- }
    confidence      DOUBLE PRECISION,
    model_version   VARCHAR(64),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_insights_repo ON ai_insights (repository_id, insight_type);
CREATE INDEX idx_ai_insights_target ON ai_insights (target_type, target_id);

-- === SARIF IMPORTS (raw document storage) ===

CREATE TABLE sarif_imports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    analysis_id     UUID NOT NULL REFERENCES analyses(id),
    tool_name       VARCHAR(255) NOT NULL,
    tool_version    VARCHAR(64),
    -- Store the full SARIF run as JSONB for traceability
    sarif_run       JSONB NOT NULL,
    -- Extracted summary for quick queries
    results_count   INTEGER GENERATED ALWAYS AS (
        jsonb_array_length(COALESCE(sarif_run->'results', '[]'::JSONB))
    ) STORED,
    imported_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sarif_imports_analysis ON sarif_imports (analysis_id);
```

### Example Queries

```sql
-- Dashboard: Repository health overview with JSONB metric detail
SELECT
    r.name,
    a.coverage_pct,
    a.avg_complexity,
    a.duplication_pct,
    a.quality_gate_status,
    a.metrics_detail->'security'->>'security_rating' AS security_rating,
    a.metrics_detail->'maintainability'->>'rating' AS maintainability_rating,
    a.findings_count
FROM repositories r
JOIN LATERAL (
    SELECT * FROM analyses
    WHERE repository_id = r.id AND status = 'completed'
    ORDER BY created_at DESC LIMIT 1
) a ON true
WHERE r.organization_id = $1 AND r.is_active = true;

-- Trend: Coverage over time with full JSONB details when drilling down
SELECT
    mh.analysis_date,
    mh.coverage_pct,
    mh.avg_complexity,
    mh.open_findings,
    mh.full_metrics->'coverage'->'branch_coverage_pct' AS branch_coverage
FROM metric_history mh
WHERE mh.repository_id = $1
  AND mh.analysis_date >= CURRENT_DATE - INTERVAL '90 days'
ORDER BY mh.analysis_date;

-- Find components with AI-generated code patterns flagged by AI
SELECT
    c.path,
    c.current_metrics->>'cyclomatic_complexity' AS complexity,
    ai.content->>'summary' AS ai_explanation,
    ai.content->'risk_factors' AS risk_factors,
    ai.confidence
FROM components c
JOIN ai_insights ai ON ai.target_id = c.id AND ai.target_type = 'component'
WHERE c.repository_id = $1
  AND ai.insight_type = 'explanation'
  AND ai.confidence > 0.8
ORDER BY (c.current_metrics->>'cyclomatic_complexity')::FLOAT DESC;
```

---

## Pros

- **Best of both worlds**: Relational columns for the 80% of queries that need fast filtering, aggregation, and joins. JSONB columns for the 20% that need flexibility.
- **Single database**: No need to operate PostgreSQL AND MongoDB/Elasticsearch. Reduces operational complexity, backup strategy, and team knowledge requirements.
- **Schema evolution without migrations**: New metrics, new AI insight types, new scanner output fields, and new SARIF extensions can be stored immediately in JSONB without ALTER TABLE migrations. Only promote to relational columns when query patterns stabilize.
- **SARIF native storage**: Full SARIF documents can be stored as JSONB, preserving the original scanner output for traceability and compliance while extracted findings populate the relational finding tables.
- **Generated columns bridge the gap**: PostgreSQL generated columns allow extracting frequently-queried JSONB fields into indexed relational columns without application-layer denormalization.
- **AI-insight friendly**: AI-generated explanations, predictions, and fix suggestions have rapidly evolving schemas. JSONB accommodates this without constant migrations.
- **GIN index performance**: PostgreSQL GIN indexes on JSONB support efficient containment queries (`@>`), existence checks (`?`), and path queries, enabling ad-hoc exploration of metric details.

## Cons

- **Discipline required**: Without clear guidelines on what goes in relational columns vs. JSONB, the schema can devolve into "everything is JSONB" -- losing query performance and type safety. The team must enforce the boundary.
- **JSONB query performance**: Complex JSONB queries (deep nesting, array operations) are slower than equivalent relational joins. JSONB is excellent for lookups and containment checks but poor for analytical aggregation across large JSONB arrays.
- **No schema validation in DB**: JSONB columns accept any valid JSON. Without application-layer validation (JSON Schema, Zod, etc.), data quality in JSONB fields can degrade silently.
- **ORM limitations**: Some ORMs handle JSONB queries awkwardly. Drizzle and Prisma have improving JSONB support, but complex JSONB queries often require raw SQL.
- **Testing complexity**: Unit tests must validate both relational constraints and JSONB structure. Missing or malformed JSONB won't trigger a database constraint violation.
- **Backup/restore granularity**: Large JSONB fields (especially sarif_run in sarif_imports) increase row sizes and can slow pg_dump/pg_restore.

---

## Technology Recommendations

| Layer | Technology |
|-------|-----------|
| Database | PostgreSQL 17+ (JSONB improvements, generated columns) |
| ORM | Drizzle ORM (best JSONB support in TypeScript) or Prisma with raw queries |
| Validation | Zod schemas for JSONB field validation at the application layer |
| SARIF Parsing | @microsoft/sarif-multitool or custom parser |
| API | REST with OpenAPI 3.1 (JSON Schema for JSONB response types) |
| Search | PostgreSQL full-text search (tsvector) for finding messages; Elasticsearch only if needed |
| Cache | Redis for dashboard query caching |

---

## Migration and Scaling Considerations

- **Promote-to-column pattern**: Start by storing new data in JSONB. When a JSONB field is queried frequently enough to need an index, promote it to a generated column or a real column. This avoids premature schema commitments.
- **JSONB size monitoring**: Set alerts for JSONB column average sizes. If `sarif_run` exceeds 1MB per row, consider compressing or moving to object storage with a reference pointer.
- **Partitioning**: Partition `metric_history`, `component_metric_history`, and `findings` by date range. JSONB columns do not interfere with PostgreSQL declarative partitioning.
- **TOAST tuning**: Large JSONB values are automatically TOASTed (compressed and stored out-of-line). Tune `toast_tuple_target` for tables with large JSONB columns to control when TOAST kicks in.
- **JSON Schema registry**: Maintain versioned JSON Schema documents for each JSONB column (e.g., `metrics_detail` v1, v2). Validate incoming data against these schemas at the application layer before insertion.
- **Migration from pure relational**: This is the easiest migration path from Suggestion 1. Add JSONB columns to existing tables, backfill from relational data, and gradually shift new feature data into JSONB.
- **Scaling ceiling**: This approach scales well to 1000+ repositories with daily analysis. For larger deployments, consider adding TimescaleDB (Suggestion 4) specifically for the metric_history and component_metric_history tables.
