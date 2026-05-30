# Data Model Suggestion 1: Normalized Relational (PostgreSQL)

> Project: Code Quality Metrics Dashboard (Candidate #287)
> Approach: Fully normalized relational schema using PostgreSQL

---

## Summary

A traditional, fully normalized relational schema in PostgreSQL (3NF+) where every entity has its own table with strict foreign key relationships. This is the approach used by SonarQube, Code Climate, and most incumbent code quality tools. It provides strong consistency guarantees, well-understood query patterns, and excellent tooling support.

---

## Key Entities and Relationships

### Entity-Relationship Overview

```
Organization (1) ──── (N) Repository (1) ──── (N) Branch
                                 │
                                 ├──── (N) Analysis (snapshot of a scan)
                                 │            │
                                 │            ├──── (N) MetricSnapshot (metric values at this point)
                                 │            ├──── (N) Finding (issues discovered)
                                 │            └──── (N) CoverageReport
                                 │
                                 └──── (N) Component (files, directories, modules)
                                              │
                                              └──── (N) ComponentMetric

Rule (1) ──── (N) Finding
QualityProfile (1) ──── (N) QualityProfileRule ──── (1) Rule
QualityGate (1) ──── (N) QualityGateCondition
```

### Core Tables

```sql
-- === ORGANIZATIONAL STRUCTURE ===

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(128) NOT NULL UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255),
    role            VARCHAR(32) NOT NULL DEFAULT 'member',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- === SOURCE CODE STRUCTURE ===

CREATE TABLE repositories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    url             TEXT,
    default_branch  VARCHAR(255) DEFAULT 'main',
    language        VARCHAR(64),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, name)
);

CREATE TABLE branches (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id),
    name            VARCHAR(255) NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    UNIQUE (repository_id, name)
);

CREATE TABLE components (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id),
    path            TEXT NOT NULL,
    component_type  VARCHAR(32) NOT NULL, -- 'file', 'directory', 'module'
    language        VARCHAR(64),
    lines_of_code   INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (repository_id, path)
);

-- === ANALYSIS ENGINE ===

CREATE TABLE analyses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id),
    branch_id       UUID NOT NULL REFERENCES branches(id),
    commit_sha      VARCHAR(40) NOT NULL,
    status          VARCHAR(32) NOT NULL DEFAULT 'pending', -- pending, running, completed, failed
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    triggered_by    VARCHAR(64), -- 'ci_pipeline', 'manual', 'schedule'
    scanner_version VARCHAR(32),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_analyses_repo_branch ON analyses (repository_id, branch_id, created_at DESC);

-- === METRICS ===

CREATE TABLE metric_definitions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    key             VARCHAR(128) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    domain          VARCHAR(64) NOT NULL, -- 'complexity', 'coverage', 'duplication', 'security', 'maintainability'
    value_type      VARCHAR(32) NOT NULL, -- 'integer', 'float', 'percent', 'rating', 'boolean'
    direction       SMALLINT DEFAULT 0,   -- -1 = lower is better, 0 = neutral, 1 = higher is better
    is_builtin      BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE metric_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    analysis_id     UUID NOT NULL REFERENCES analyses(id) ON DELETE CASCADE,
    component_id    UUID NOT NULL REFERENCES components(id),
    metric_id       UUID NOT NULL REFERENCES metric_definitions(id),
    value           DOUBLE PRECISION,
    text_value      VARCHAR(512),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (analysis_id, component_id, metric_id)
);

CREATE INDEX idx_metric_snapshots_component ON metric_snapshots (component_id, metric_id, created_at DESC);

-- === FINDINGS (issues, code smells, vulnerabilities) ===

CREATE TABLE rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    key             VARCHAR(255) NOT NULL UNIQUE,
    name            VARCHAR(512) NOT NULL,
    description     TEXT,
    language        VARCHAR(64),
    severity        VARCHAR(32) NOT NULL, -- 'info', 'minor', 'major', 'critical', 'blocker'
    rule_type       VARCHAR(32) NOT NULL, -- 'bug', 'vulnerability', 'code_smell', 'security_hotspot'
    cwe_ids         INTEGER[],            -- CWE references
    tags            VARCHAR(64)[],
    remediation_fn  VARCHAR(32),          -- 'constant', 'linear'
    remediation_val INTEGER,              -- minutes
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE findings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    analysis_id     UUID NOT NULL REFERENCES analyses(id),
    component_id    UUID NOT NULL REFERENCES components(id),
    rule_id         UUID NOT NULL REFERENCES rules(id),
    severity        VARCHAR(32) NOT NULL,
    status          VARCHAR(32) NOT NULL DEFAULT 'open', -- open, confirmed, resolved, false_positive, wont_fix
    resolution      VARCHAR(32),
    line_number     INTEGER,
    column_number   INTEGER,
    end_line        INTEGER,
    message         TEXT NOT NULL,
    effort_minutes  INTEGER,
    assignee_id     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_findings_analysis ON findings (analysis_id);
CREATE INDEX idx_findings_component ON findings (component_id, status);
CREATE INDEX idx_findings_rule ON findings (rule_id);
CREATE INDEX idx_findings_status ON findings (status, severity);

-- === COVERAGE ===

CREATE TABLE coverage_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    analysis_id     UUID NOT NULL REFERENCES analyses(id),
    component_id    UUID NOT NULL REFERENCES components(id),
    lines_to_cover  INTEGER NOT NULL DEFAULT 0,
    lines_covered   INTEGER NOT NULL DEFAULT 0,
    branches_to_cover INTEGER NOT NULL DEFAULT 0,
    branches_covered INTEGER NOT NULL DEFAULT 0,
    coverage_pct    DOUBLE PRECISION GENERATED ALWAYS AS (
        CASE WHEN lines_to_cover > 0 THEN (lines_covered::DOUBLE PRECISION / lines_to_cover) * 100.0 ELSE 0 END
    ) STORED,
    UNIQUE (analysis_id, component_id)
);

-- === QUALITY GATES ===

CREATE TABLE quality_gates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE quality_gate_conditions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    quality_gate_id UUID NOT NULL REFERENCES quality_gates(id) ON DELETE CASCADE,
    metric_id       UUID NOT NULL REFERENCES metric_definitions(id),
    operator        VARCHAR(8) NOT NULL, -- 'LT', 'GT', 'EQ', 'NE'
    error_threshold DOUBLE PRECISION NOT NULL,
    warn_threshold  DOUBLE PRECISION
);

-- === QUALITY PROFILES ===

CREATE TABLE quality_profiles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    language        VARCHAR(64) NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    parent_id       UUID REFERENCES quality_profiles(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE quality_profile_rules (
    quality_profile_id UUID NOT NULL REFERENCES quality_profiles(id) ON DELETE CASCADE,
    rule_id            UUID NOT NULL REFERENCES rules(id),
    severity_override  VARCHAR(32),
    PRIMARY KEY (quality_profile_id, rule_id)
);

-- === TECHNICAL DEBT ===

CREATE TABLE technical_debt_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id),
    component_id    UUID REFERENCES components(id),
    category        VARCHAR(64) NOT NULL, -- 'complexity', 'duplication', 'coverage_gap', 'security'
    description     TEXT NOT NULL,
    estimated_effort_minutes INTEGER,
    priority        VARCHAR(32) DEFAULT 'medium',
    status          VARCHAR(32) NOT NULL DEFAULT 'open',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);
```

### Trend Queries (Historical Analysis)

```sql
-- Cyclomatic complexity trend for a repository over time
SELECT
    a.commit_sha,
    a.completed_at,
    AVG(ms.value) AS avg_complexity,
    MAX(ms.value) AS max_complexity,
    COUNT(CASE WHEN ms.value > 10 THEN 1 END) AS high_complexity_count
FROM analyses a
JOIN metric_snapshots ms ON ms.analysis_id = a.id
JOIN metric_definitions md ON md.id = ms.metric_id
WHERE a.repository_id = $1
  AND md.key = 'cyclomatic_complexity'
  AND a.status = 'completed'
GROUP BY a.id, a.commit_sha, a.completed_at
ORDER BY a.completed_at DESC
LIMIT 50;
```

---

## Pros

- **Strong consistency**: Foreign key constraints enforce referential integrity across all entities; no orphaned findings or metric snapshots.
- **Mature tooling**: PostgreSQL is the default database for SonarQube, Code Climate, and most incumbents. ORM support (Prisma, Drizzle, SQLAlchemy, TypeORM) is excellent.
- **Well-understood query patterns**: JOINs across metric_snapshots, analyses, and components produce the exact data needed for dashboards. No denormalization guesswork.
- **ACID transactions**: Analysis results can be written atomically -- all metric snapshots and findings for an analysis commit together or roll back.
- **Standards alignment**: Maps directly to SARIF entities (runs -> analyses, results -> findings, rules -> rules) and ISO/IEC 25010 quality characteristics (via metric_definitions.domain).
- **Reporting simplicity**: Aggregate queries for executive dashboards are straightforward SQL with GROUP BY on time, repository, and team dimensions.

## Cons

- **Trend query performance**: Historical metric queries spanning months of data across many repositories require scanning large metric_snapshots tables. Without partitioning or materialized views, these queries become slow as data grows.
- **Metric cardinality explosion**: With N components x M metrics x P analyses, the metric_snapshots table grows rapidly (e.g., 10K files x 15 metrics x 365 daily analyses = 54.75M rows/year for one repository).
- **Schema rigidity**: Adding a new metric type, a new finding attribute, or a new coverage dimension requires ALTER TABLE migrations. This can be operationally disruptive on large tables.
- **Read/write contention**: Concurrent analysis ingestion (writes) and dashboard queries (reads) compete for the same tables and indexes, potentially requiring read replicas.
- **No native time-series optimization**: PostgreSQL lacks built-in time-bucketing, automatic partitioning by time, or compression for metric history. These must be layered manually (table partitioning, materialized views).

---

## Technology Recommendations

| Layer | Technology |
|-------|-----------|
| Database | PostgreSQL 17+ |
| ORM / Query Builder | Drizzle ORM or Prisma (TypeScript) / SQLAlchemy (Python) |
| Migrations | Built-in ORM migrations or Flyway |
| Connection Pool | PgBouncer for connection pooling |
| Caching | Redis for dashboard query caching |
| Read Scaling | PostgreSQL streaming replicas for dashboard reads |

---

## Migration and Scaling Considerations

- **Table partitioning**: Partition `metric_snapshots` and `findings` by `created_at` (range partitioning, monthly) to keep query scans bounded and enable efficient archival.
- **Materialized views**: Pre-compute daily/weekly/monthly aggregates of key metrics per repository using `REFRESH MATERIALIZED VIEW CONCURRENTLY` on a schedule.
- **Archival strategy**: Move metric_snapshots older than 2 years to cold storage (S3 + Parquet) while keeping summary aggregates in the primary database.
- **Index strategy**: Partial indexes on `findings(status)` WHERE status = 'open' to accelerate dashboard queries that only show active issues.
- **Read replicas**: Route all dashboard and API read queries to streaming replicas; direct analysis ingestion writes to the primary.
- **Vertical scaling path**: PostgreSQL handles the first 100M-500M rows well on modern hardware. Beyond that, consider upgrading to the time-series approach (Suggestion 4) or adding partitioning.
- **Data volume estimate**: For 500 repositories with daily scans, expect approximately 27B metric_snapshot rows/year before aggregation. Partitioning and materialized views are essential at this scale.
