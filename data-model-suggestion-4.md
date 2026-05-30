# Data Model Suggestion 4: Time-Series Optimized (PostgreSQL + TimescaleDB)

> Project: Code Quality Metrics Dashboard (Candidate #287)
> Approach: PostgreSQL for relational entities + TimescaleDB hypertables for metric time-series with continuous aggregates

---

## Summary

A specialty architecture that recognizes that a code quality metrics dashboard is fundamentally a **time-series analytics application**. The primary user need -- tracking how metrics evolve over commits, sprints, and releases -- is a time-series problem. This design uses standard PostgreSQL tables for organizational and configuration entities, and TimescaleDB hypertables for all metric, finding, and event data that has a temporal dimension.

TimescaleDB is a PostgreSQL extension (not a separate database) that adds automatic time-based partitioning (hypertables), columnar compression (90%+ typical), continuous aggregates (incrementally maintained materialized views), and time-bucketing functions -- all accessible via standard SQL. This means no operational split between "the relational database" and "the time-series database"; it is a single PostgreSQL instance with superpowers for temporal data.

---

## Key Entities and Relationships

### Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                     PostgreSQL + TimescaleDB                     │
│                                                                  │
│  ┌─────────────────────┐    ┌──────────────────────────────────┐│
│  │  Standard Tables     │    │  TimescaleDB Hypertables         ││
│  │  (Relational)        │    │  (Time-Series)                   ││
│  │                      │    │                                  ││
│  │  organizations       │    │  metric_samples      ←─ raw     ││
│  │  repositories        │    │  finding_events      ←─ raw     ││
│  │  components          │    │  coverage_samples    ←─ raw     ││
│  │  rules               │    │  gate_evaluations    ←─ raw     ││
│  │  quality_gates       │    │                                  ││
│  │  quality_profiles    │    │  ┌───────────────────────────┐  ││
│  │  users               │    │  │ Continuous Aggregates     │  ││
│  │                      │    │  │ (auto-maintained views)   │  ││
│  │                      │    │  │                           │  ││
│  │                      │    │  │ metrics_hourly            │  ││
│  │                      │    │  │ metrics_daily             │  ││
│  │                      │    │  │ metrics_weekly            │  ││
│  │                      │    │  │ findings_daily_summary    │  ││
│  │                      │    │  │ repo_health_daily         │  ││
│  │                      │    │  └───────────────────────────┘  ││
│  └─────────────────────┘    └──────────────────────────────────┘│
└──────────────────────────────────────────────────────────────────┘
```

### Standard Relational Tables

```sql
-- === ORGANIZATIONAL STRUCTURE (standard PostgreSQL tables) ===

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(128) NOT NULL UNIQUE,
    settings        JSONB NOT NULL DEFAULT '{}',
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
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, name)
);

CREATE TABLE components (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id),
    path            TEXT NOT NULL,
    component_type  VARCHAR(32) NOT NULL,
    language        VARCHAR(64),
    lines_of_code   INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (repository_id, path)
);

CREATE TABLE rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    key             VARCHAR(255) NOT NULL UNIQUE,
    name            VARCHAR(512) NOT NULL,
    description     TEXT,
    language        VARCHAR(64),
    severity        VARCHAR(32) NOT NULL,
    rule_type       VARCHAR(32) NOT NULL,
    cwe_ids         INTEGER[],
    tags            TEXT[],
    remediation_minutes INTEGER,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE analyses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id),
    branch          VARCHAR(255) NOT NULL,
    commit_sha      VARCHAR(40) NOT NULL,
    status          VARCHAR(32) NOT NULL DEFAULT 'pending',
    triggered_by    VARCHAR(64),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    scanner_version VARCHAR(32),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE quality_gates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    conditions      JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255),
    role            VARCHAR(32) NOT NULL DEFAULT 'member',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### TimescaleDB Hypertables (Time-Series Data)

```sql
-- === METRIC SAMPLES (the core time-series table) ===

CREATE TABLE metric_samples (
    time            TIMESTAMPTZ NOT NULL,
    repository_id   UUID NOT NULL,
    analysis_id     UUID NOT NULL,
    component_id    UUID,               -- NULL = repository-level metric
    metric_key      VARCHAR(128) NOT NULL,
    value           DOUBLE PRECISION NOT NULL,
    commit_sha      VARCHAR(40)
);

-- Convert to hypertable with automatic time-based partitioning (7-day chunks)
SELECT create_hypertable('metric_samples', 'time',
    chunk_time_interval => INTERVAL '7 days'
);

-- Indexes optimized for dashboard query patterns
CREATE INDEX idx_ms_repo_metric ON metric_samples (repository_id, metric_key, time DESC);
CREATE INDEX idx_ms_component ON metric_samples (component_id, metric_key, time DESC)
    WHERE component_id IS NOT NULL;

-- Enable columnar compression for chunks older than 2 weeks
ALTER TABLE metric_samples SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'repository_id, metric_key',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('metric_samples', INTERVAL '14 days');

-- === FINDING EVENTS (time-series of finding lifecycle) ===

CREATE TABLE finding_events (
    time            TIMESTAMPTZ NOT NULL,
    repository_id   UUID NOT NULL,
    analysis_id     UUID NOT NULL,
    component_id    UUID NOT NULL,
    rule_id         UUID NOT NULL,
    finding_hash    VARCHAR(64) NOT NULL,  -- deterministic hash for deduplication across analyses
    event_type      VARCHAR(32) NOT NULL,  -- 'detected', 'resolved', 'reopened', 'suppressed'
    severity        VARCHAR(32) NOT NULL,
    finding_type    VARCHAR(32) NOT NULL,  -- 'bug', 'vulnerability', 'code_smell', 'security_hotspot'
    line_number     INTEGER,
    message         TEXT,
    effort_minutes  INTEGER,
    detail          JSONB                  -- location, snippet, data flow, AI explanation
);

SELECT create_hypertable('finding_events', 'time',
    chunk_time_interval => INTERVAL '7 days'
);

CREATE INDEX idx_fe_repo ON finding_events (repository_id, time DESC);
CREATE INDEX idx_fe_type ON finding_events (finding_type, severity, time DESC);
CREATE INDEX idx_fe_hash ON finding_events (finding_hash, time DESC);

ALTER TABLE finding_events SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'repository_id, finding_type',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('finding_events', INTERVAL '14 days');

-- === COVERAGE SAMPLES ===

CREATE TABLE coverage_samples (
    time            TIMESTAMPTZ NOT NULL,
    repository_id   UUID NOT NULL,
    analysis_id     UUID NOT NULL,
    component_id    UUID,
    lines_to_cover  INTEGER NOT NULL,
    lines_covered   INTEGER NOT NULL,
    branches_to_cover INTEGER DEFAULT 0,
    branches_covered INTEGER DEFAULT 0,
    line_coverage_pct DOUBLE PRECISION GENERATED ALWAYS AS (
        CASE WHEN lines_to_cover > 0 THEN (lines_covered::DOUBLE PRECISION / lines_to_cover) * 100.0 ELSE 0 END
    ) STORED,
    commit_sha      VARCHAR(40)
);

SELECT create_hypertable('coverage_samples', 'time',
    chunk_time_interval => INTERVAL '7 days'
);

CREATE INDEX idx_cs_repo ON coverage_samples (repository_id, time DESC);

ALTER TABLE coverage_samples SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'repository_id',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('coverage_samples', INTERVAL '14 days');

-- === QUALITY GATE EVALUATIONS ===

CREATE TABLE gate_evaluations (
    time            TIMESTAMPTZ NOT NULL,
    repository_id   UUID NOT NULL,
    analysis_id     UUID NOT NULL,
    gate_id         UUID NOT NULL,
    status          VARCHAR(32) NOT NULL,  -- 'passed', 'warned', 'failed'
    conditions      JSONB NOT NULL,        -- evaluated conditions with actual values
    commit_sha      VARCHAR(40)
);

SELECT create_hypertable('gate_evaluations', 'time',
    chunk_time_interval => INTERVAL '30 days'
);

CREATE INDEX idx_ge_repo ON gate_evaluations (repository_id, time DESC);
```

### Continuous Aggregates (Auto-Maintained Rollups)

```sql
-- === HOURLY METRIC AGGREGATES ===

CREATE MATERIALIZED VIEW metrics_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    repository_id,
    metric_key,
    AVG(value)   AS avg_value,
    MIN(value)   AS min_value,
    MAX(value)   AS max_value,
    LAST(value, time) AS latest_value,
    COUNT(*)     AS sample_count
FROM metric_samples
GROUP BY bucket, repository_id, metric_key
WITH NO DATA;

SELECT add_continuous_aggregate_policy('metrics_hourly',
    start_offset    => INTERVAL '3 hours',
    end_offset      => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);

-- === DAILY METRIC AGGREGATES ===

CREATE MATERIALIZED VIEW metrics_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    repository_id,
    metric_key,
    AVG(value)   AS avg_value,
    MIN(value)   AS min_value,
    MAX(value)   AS max_value,
    LAST(value, time) AS latest_value,
    COUNT(*)     AS sample_count
FROM metric_samples
GROUP BY bucket, repository_id, metric_key
WITH NO DATA;

SELECT add_continuous_aggregate_policy('metrics_daily',
    start_offset    => INTERVAL '3 days',
    end_offset      => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day'
);

-- === WEEKLY METRIC AGGREGATES (for long-range trend charts) ===

CREATE MATERIALIZED VIEW metrics_weekly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('7 days', time) AS bucket,
    repository_id,
    metric_key,
    AVG(value)   AS avg_value,
    MIN(value)   AS min_value,
    MAX(value)   AS max_value,
    LAST(value, time) AS latest_value,
    COUNT(*)     AS sample_count
FROM metric_samples
GROUP BY bucket, repository_id, metric_key
WITH NO DATA;

SELECT add_continuous_aggregate_policy('metrics_weekly',
    start_offset    => INTERVAL '14 days',
    end_offset      => INTERVAL '7 days',
    schedule_interval => INTERVAL '1 day'
);

-- === DAILY FINDING SUMMARY ===

CREATE MATERIALIZED VIEW findings_daily_summary
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    repository_id,
    finding_type,
    severity,
    event_type,
    COUNT(*) AS event_count,
    SUM(effort_minutes) AS total_effort_minutes
FROM finding_events
GROUP BY bucket, repository_id, finding_type, severity, event_type
WITH NO DATA;

SELECT add_continuous_aggregate_policy('findings_daily_summary',
    start_offset    => INTERVAL '3 days',
    end_offset      => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day'
);

-- === DAILY REPOSITORY HEALTH COMPOSITE ===

CREATE MATERIALIZED VIEW repo_health_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    repository_id,
    -- Coverage (latest in bucket)
    LAST(
        CASE WHEN metric_key = 'coverage_pct' THEN value END,
        time
    ) AS coverage_pct,
    -- Complexity
    LAST(
        CASE WHEN metric_key = 'cyclomatic_complexity_avg' THEN value END,
        time
    ) AS avg_complexity,
    LAST(
        CASE WHEN metric_key = 'cyclomatic_complexity_max' THEN value END,
        time
    ) AS max_complexity,
    -- Duplication
    LAST(
        CASE WHEN metric_key = 'duplication_pct' THEN value END,
        time
    ) AS duplication_pct,
    -- Maintainability
    LAST(
        CASE WHEN metric_key = 'maintainability_index' THEN value END,
        time
    ) AS maintainability_idx,
    -- Debt
    LAST(
        CASE WHEN metric_key = 'technical_debt_minutes' THEN value END,
        time
    ) AS debt_minutes
FROM metric_samples
WHERE component_id IS NULL  -- repository-level metrics only
GROUP BY bucket, repository_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('repo_health_daily',
    start_offset    => INTERVAL '3 days',
    end_offset      => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day'
);
```

### Data Retention Policies

```sql
-- Raw metric samples: retain 6 months (compressed after 2 weeks)
SELECT add_retention_policy('metric_samples', INTERVAL '180 days');

-- Raw finding events: retain 1 year
SELECT add_retention_policy('finding_events', INTERVAL '365 days');

-- Hourly aggregates: retain 90 days
SELECT add_retention_policy('metrics_hourly', INTERVAL '90 days');

-- Daily aggregates: retain 3 years
SELECT add_retention_policy('metrics_daily', INTERVAL '1095 days');

-- Weekly aggregates: retain 10 years
SELECT add_retention_policy('metrics_weekly', INTERVAL '3650 days');
```

### Example Dashboard Queries

```sql
-- 1. Coverage trend for a repository over the last 90 days (< 1ms with continuous aggregates)
SELECT
    bucket AS date,
    latest_value AS coverage_pct
FROM metrics_daily
WHERE repository_id = $1
  AND metric_key = 'coverage_pct'
  AND bucket >= NOW() - INTERVAL '90 days'
ORDER BY bucket;

-- 2. Multi-metric trend chart (complexity, coverage, duplication)
SELECT
    bucket AS date,
    coverage_pct,
    avg_complexity,
    duplication_pct,
    maintainability_idx
FROM repo_health_daily
WHERE repository_id = $1
  AND bucket >= NOW() - INTERVAL '180 days'
ORDER BY bucket;

-- 3. Finding burn-down: new findings vs. resolved findings per day
SELECT
    bucket AS date,
    SUM(CASE WHEN event_type = 'detected' THEN event_count ELSE 0 END) AS new_findings,
    SUM(CASE WHEN event_type = 'resolved' THEN event_count ELSE 0 END) AS resolved_findings,
    SUM(CASE WHEN event_type = 'detected' THEN event_count ELSE 0 END)
      - SUM(CASE WHEN event_type = 'resolved' THEN event_count ELSE 0 END) AS net_change
FROM findings_daily_summary
WHERE repository_id = $1
  AND bucket >= NOW() - INTERVAL '90 days'
GROUP BY bucket
ORDER BY bucket;

-- 4. Sprint-over-sprint comparison (2-week buckets)
SELECT
    time_bucket('14 days', bucket) AS sprint,
    AVG(coverage_pct) AS avg_coverage,
    AVG(avg_complexity) AS avg_complexity,
    AVG(duplication_pct) AS avg_duplication
FROM repo_health_daily
WHERE repository_id = $1
  AND bucket >= NOW() - INTERVAL '365 days'
GROUP BY sprint
ORDER BY sprint;

-- 5. Cross-repository benchmarking (organization-wide)
SELECT
    r.name AS repository,
    rhd.coverage_pct,
    rhd.avg_complexity,
    rhd.duplication_pct,
    rhd.maintainability_idx
FROM repo_health_daily rhd
JOIN repositories r ON r.id = rhd.repository_id
WHERE r.organization_id = $1
  AND rhd.bucket = (
      SELECT MAX(bucket) FROM repo_health_daily
      WHERE repository_id = rhd.repository_id
  )
ORDER BY rhd.maintainability_idx ASC;

-- 6. Hotspot detection: files with rising complexity
SELECT
    c.path,
    c.language,
    regr_slope(ms.value, EXTRACT(EPOCH FROM ms.time)) AS complexity_trend_slope,
    LAST(ms.value, ms.time) AS current_complexity,
    FIRST(ms.value, ms.time) AS complexity_90d_ago
FROM metric_samples ms
JOIN components c ON c.id = ms.component_id
WHERE ms.repository_id = $1
  AND ms.metric_key = 'cyclomatic_complexity'
  AND ms.time >= NOW() - INTERVAL '90 days'
GROUP BY c.id, c.path, c.language
HAVING regr_slope(ms.value, EXTRACT(EPOCH FROM ms.time)) > 0
ORDER BY regr_slope(ms.value, EXTRACT(EPOCH FROM ms.time)) DESC
LIMIT 20;
```

---

## Pros

- **Purpose-built for the core use case**: Trend tracking is the #1 buying requirement identified in market research. TimescaleDB makes time-range queries, bucketed aggregations, and multi-resolution rollups first-class operations rather than afterthoughts.
- **Continuous aggregates eliminate dashboard latency**: Pre-computed hourly/daily/weekly rollups update incrementally. Dashboard queries hit small, pre-aggregated tables and return in under 1ms instead of scanning millions of raw rows.
- **90%+ compression**: TimescaleDB columnar compression reduces storage for metric history by 10-20x. A dataset that would require 500GB uncompressed fits in 25-50GB.
- **Automatic data lifecycle**: Built-in retention policies drop old raw data while keeping aggregates for long-term trends. No custom archival scripts or cron jobs.
- **Still PostgreSQL**: All standard PostgreSQL features (foreign keys, transactions, JSONB, full-text search, CTEs, window functions) remain available. ORMs, migration tools, and connection pools work unchanged.
- **Multi-resolution analytics**: Hourly, daily, weekly, and sprint-level views are served from their respective continuous aggregates. Users can zoom from a 2-year weekly trend into a daily view into raw hourly data seamlessly.
- **Linear regression built-in**: PostgreSQL's `regr_slope()` function on time-series data enables native hotspot detection (files with rising complexity trend) without external ML tooling.
- **Grafana native support**: TimescaleDB has first-class Grafana data source support, enabling rapid prototyping of dashboards before building a custom UI.

## Cons

- **Extension dependency**: TimescaleDB is a PostgreSQL extension, not a core feature. Managed PostgreSQL providers (AWS RDS, Azure) support it, but some (e.g., Neon, Supabase) have limitations. Self-hosted deployments need to install and maintain the extension.
- **Hypertable constraints**: Hypertables have restrictions: no foreign keys pointing TO a hypertable, unique constraints must include the time column, and some DDL operations behave differently. This shapes the schema design.
- **Learning curve**: The team must learn hypertable chunking, compression policies, continuous aggregate semantics (real-time vs. materialized-only), and retention policies. This is additional operational knowledge.
- **Continuous aggregate limitations**: Continuous aggregates have restrictions on supported SQL (no joins, limited function support in older versions). Complex aggregations may require post-processing at the application layer.
- **Write amplification**: Each metric sample is a separate row (denormalized from the analysis). An analysis that measures 15 metrics across 10K files generates 150K rows per analysis, versus a single row per analysis in a normalized model.
- **Finding tracking complexity**: Tracking finding lifecycle (open/resolved/reopened) across analyses requires computing diffs between consecutive finding_events, which is more complex than updating a status column in a relational findings table.
- **Cost at scale**: TimescaleDB Community is free. TimescaleDB Cloud pricing is based on storage and compute, which can become significant for large deployments with high cardinality metrics.

---

## Technology Recommendations

| Layer | Technology |
|-------|-----------|
| Database | PostgreSQL 17+ with TimescaleDB 2.x extension |
| Managed Option | Timescale Cloud or AWS RDS with TimescaleDB |
| ORM (relational tables) | Drizzle ORM or Prisma |
| Time-series queries | Raw SQL via pg driver (best control over TimescaleDB functions) |
| Dashboard prototype | Grafana with TimescaleDB data source |
| Production dashboard | Custom React/Next.js with chart library (Recharts, Tremor) |
| API | REST with OpenAPI 3.1 |
| Cache | Redis for current-state caching; continuous aggregates eliminate most cache needs for trends |

---

## Migration and Scaling Considerations

- **Chunk interval tuning**: Start with 7-day chunks for metric_samples. Monitor chunk sizes; if chunks exceed 1-2GB, reduce the interval to 1 day. If chunks are too small (< 25MB), increase to 14 days.
- **Compression policy tuning**: The 14-day compression offset means the most recent 2 weeks of data remain uncompressed for fast insert and point queries. Adjust based on how frequently users query recent vs. historical data.
- **Space-partitioning for multi-tenancy**: For large deployments with many organizations, add space partitioning by `repository_id` to distribute hypertable chunks across tablespaces: `SELECT create_hypertable('metric_samples', 'time', partitioning_column => 'repository_id', number_partitions => 4)`.
- **Continuous aggregate refresh tuning**: The default policies refresh hourly/daily. For real-time dashboards, enable real-time aggregation mode (combines materialized data with recent unmaterialized data at query time).
- **Migration from normalized relational (Suggestion 1)**: Extract metric_snapshots and findings into the time-series tables with a one-time migration. Keep the relational tables for config and state, replacing only the historical data storage.
- **Horizontal scaling**: TimescaleDB supports distributed hypertables (multi-node) for deployments exceeding single-node capacity. This requires TimescaleDB 2.x and careful planning.
- **Backup strategy**: Use `pg_dump` for relational tables and TimescaleDB's native backup tooling for hypertables. Continuous aggregates are automatically rebuilt from base hypertables on restore.
- **Estimated storage**: For 500 repositories with daily scans (15 metrics x 10K files): ~2.7B raw rows/year. With 95% compression on data older than 2 weeks, actual storage is approximately 50-100GB/year.
