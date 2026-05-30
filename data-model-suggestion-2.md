# Data Model Suggestion 2: Event-Sourced / CQRS Approach

> Project: Code Quality Metrics Dashboard (Candidate #287)
> Approach: Event Sourcing with Command Query Responsibility Segregation (CQRS)

---

## Summary

An event-sourced architecture where every code quality event (scan started, metric computed, finding detected, quality gate evaluated) is stored as an immutable event in an append-only event store. Read-side projections are materialized into query-optimized views tailored for each dashboard use case. This approach provides a complete audit trail, enables retroactive metric computation, and naturally supports the temporal analysis that buyers increasingly demand.

---

## Key Entities and Relationships

### Architecture Overview

```
                    ┌──────────────────────┐
                    │   Command Side       │
                    │  (Write Model)       │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │   Event Store        │
                    │  (Append-Only Log)   │
                    │  - AnalysisEvents    │
                    │  - MetricEvents      │
                    │  - FindingEvents     │
                    │  - GateEvents        │
                    └──────────┬───────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
    ┌─────────▼──────┐ ┌──────▼───────┐ ┌──────▼───────┐
    │  Dashboard     │ │  Trend       │ │  Alert       │
    │  Projection    │ │  Projection  │ │  Projection  │
    │  (Read Model)  │ │  (Read Model)│ │  (Read Model)│
    └────────────────┘ └──────────────┘ └──────────────┘
```

### Event Store Schema

```sql
-- === EVENT STORE (the source of truth) ===

CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       VARCHAR(255) NOT NULL,  -- e.g., 'repository:abc123' or 'analysis:xyz789'
    stream_type     VARCHAR(64) NOT NULL,   -- 'Repository', 'Analysis', 'Finding'
    event_type      VARCHAR(128) NOT NULL,  -- e.g., 'AnalysisStarted', 'MetricComputed'
    event_version   BIGINT NOT NULL,        -- monotonic sequence per stream
    payload         JSONB NOT NULL,         -- event-specific data
    metadata        JSONB,                  -- correlation_id, causation_id, actor, etc.
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, event_version)
);

CREATE INDEX idx_event_store_stream ON event_store (stream_id, event_version);
CREATE INDEX idx_event_store_type ON event_store (event_type, occurred_at);
CREATE INDEX idx_event_store_occurred ON event_store (occurred_at);

-- === STREAM POSITION TRACKING (for projections) ===

CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(128) PRIMARY KEY,
    last_event_id   UUID REFERENCES event_store(event_id),
    last_position   BIGINT NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Event Types and Payloads

```jsonc
// AnalysisStarted
{
  "event_type": "AnalysisStarted",
  "stream_id": "analysis:a1b2c3",
  "payload": {
    "repository_id": "repo:xyz",
    "branch": "main",
    "commit_sha": "abc123def456",
    "triggered_by": "ci_pipeline",
    "scanner_version": "2.1.0"
  }
}

// MetricComputed
{
  "event_type": "MetricComputed",
  "stream_id": "analysis:a1b2c3",
  "payload": {
    "component_path": "src/services/auth.ts",
    "metric_key": "cyclomatic_complexity",
    "value": 24.0,
    "previous_value": 18.0,
    "threshold": 10.0,
    "exceeded": true
  }
}

// FindingDetected
{
  "event_type": "FindingDetected",
  "stream_id": "analysis:a1b2c3",
  "payload": {
    "rule_key": "typescript:S1192",
    "severity": "major",
    "type": "code_smell",
    "component_path": "src/utils/parser.ts",
    "line": 142,
    "message": "String 'SELECT' appears 7 times; define it as a constant.",
    "effort_minutes": 5,
    "cwe_ids": []
  }
}

// FindingResolved
{
  "event_type": "FindingResolved",
  "stream_id": "finding:f1a2b3",
  "payload": {
    "resolution": "fixed",
    "resolved_by": "user:u123",
    "resolved_in_analysis": "analysis:a9b8c7"
  }
}

// QualityGateEvaluated
{
  "event_type": "QualityGateEvaluated",
  "stream_id": "analysis:a1b2c3",
  "payload": {
    "gate_name": "Production Gate",
    "status": "failed",
    "conditions": [
      {"metric": "coverage_pct", "operator": "LT", "threshold": 80.0, "actual": 72.3, "passed": false},
      {"metric": "critical_findings", "operator": "GT", "threshold": 0, "actual": 3, "passed": false}
    ]
  }
}

// TechnicalDebtEstimated
{
  "event_type": "TechnicalDebtEstimated",
  "stream_id": "analysis:a1b2c3",
  "payload": {
    "total_effort_minutes": 4320,
    "by_category": {
      "complexity": 1800,
      "duplication": 960,
      "coverage_gap": 720,
      "code_smell": 840
    }
  }
}
```

### Read-Side Projections (Materialized Views)

```sql
-- === PROJECTION: Current Repository Health (Dashboard) ===

CREATE TABLE proj_repository_health (
    repository_id       UUID PRIMARY KEY,
    organization_id     UUID NOT NULL,
    repository_name     VARCHAR(255) NOT NULL,
    last_analysis_id    UUID,
    last_analysis_at    TIMESTAMPTZ,
    commit_sha          VARCHAR(40),
    -- Current metric values (latest analysis)
    cyclomatic_avg      DOUBLE PRECISION,
    cyclomatic_max      DOUBLE PRECISION,
    coverage_pct        DOUBLE PRECISION,
    duplication_pct     DOUBLE PRECISION,
    maintainability_idx DOUBLE PRECISION,
    -- Finding counts
    open_bugs           INTEGER DEFAULT 0,
    open_vulnerabilities INTEGER DEFAULT 0,
    open_code_smells    INTEGER DEFAULT 0,
    open_security_hotspots INTEGER DEFAULT 0,
    -- Quality gate
    quality_gate_status VARCHAR(32),
    -- Debt
    total_debt_minutes  INTEGER DEFAULT 0,
    -- Timestamps
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_repo_health_org ON proj_repository_health (organization_id);

-- === PROJECTION: Metric Trends (Charts) ===

CREATE TABLE proj_metric_trends (
    id                  BIGSERIAL PRIMARY KEY,
    repository_id       UUID NOT NULL,
    component_path      TEXT,          -- NULL = repository-level aggregate
    metric_key          VARCHAR(128) NOT NULL,
    analysis_date       DATE NOT NULL,
    value               DOUBLE PRECISION NOT NULL,
    delta               DOUBLE PRECISION, -- change from previous analysis
    commit_sha          VARCHAR(40),
    UNIQUE (repository_id, component_path, metric_key, analysis_date)
);

CREATE INDEX idx_proj_trends_repo_metric ON proj_metric_trends (repository_id, metric_key, analysis_date DESC);

-- === PROJECTION: Active Findings (Issue Tracker) ===

CREATE TABLE proj_active_findings (
    finding_id          UUID PRIMARY KEY,
    repository_id       UUID NOT NULL,
    component_path      TEXT NOT NULL,
    rule_key            VARCHAR(255) NOT NULL,
    rule_name           VARCHAR(512),
    severity            VARCHAR(32) NOT NULL,
    finding_type        VARCHAR(32) NOT NULL,
    line_number         INTEGER,
    message             TEXT,
    effort_minutes      INTEGER,
    status              VARCHAR(32) NOT NULL DEFAULT 'open',
    assignee_id         UUID,
    first_detected_at   TIMESTAMPTZ NOT NULL,
    last_seen_at        TIMESTAMPTZ NOT NULL,
    analysis_count      INTEGER DEFAULT 1,  -- how many analyses have seen this finding
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_findings_repo ON proj_active_findings (repository_id, status, severity);
CREATE INDEX idx_proj_findings_type ON proj_active_findings (finding_type, severity);

-- === PROJECTION: Team Performance (Management Dashboard) ===

CREATE TABLE proj_team_metrics (
    id                  BIGSERIAL PRIMARY KEY,
    organization_id     UUID NOT NULL,
    team_id             UUID,
    period_start        DATE NOT NULL,
    period_end          DATE NOT NULL,
    repos_analyzed      INTEGER DEFAULT 0,
    findings_introduced INTEGER DEFAULT 0,
    findings_resolved   INTEGER DEFAULT 0,
    avg_resolution_days DOUBLE PRECISION,
    debt_added_minutes  INTEGER DEFAULT 0,
    debt_removed_minutes INTEGER DEFAULT 0,
    coverage_change_pct DOUBLE PRECISION,
    UNIQUE (organization_id, team_id, period_start)
);

-- === PROJECTION: Hotspot Analysis (AI-Powered) ===

CREATE TABLE proj_code_hotspots (
    id                  BIGSERIAL PRIMARY KEY,
    repository_id       UUID NOT NULL,
    component_path      TEXT NOT NULL,
    risk_score          DOUBLE PRECISION NOT NULL, -- 0-100
    change_frequency    INTEGER NOT NULL,          -- commits in last 90 days
    complexity_current  DOUBLE PRECISION,
    complexity_trend    DOUBLE PRECISION,          -- slope of complexity over time
    finding_density     DOUBLE PRECISION,          -- findings per 1000 LOC
    contributor_count   INTEGER,                   -- ownership spread
    last_modified       TIMESTAMPTZ,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (repository_id, component_path)
);

CREATE INDEX idx_proj_hotspots_risk ON proj_code_hotspots (repository_id, risk_score DESC);
```

### Event Processing (Projection Builder)

```python
# Pseudocode for a projection event handler

class DashboardProjectionHandler:
    """Processes events to update the repository health projection."""

    async def handle(self, event: Event):
        match event.event_type:
            case "AnalysisCompleted":
                # Update last analysis metadata
                await self.update_repo_health(
                    repo_id=event.payload["repository_id"],
                    analysis_id=event.stream_id,
                    commit_sha=event.payload["commit_sha"],
                    completed_at=event.occurred_at
                )

            case "MetricComputed":
                # Update current metric value and append to trend
                await self.update_metric(event)
                await self.append_trend(event)

            case "FindingDetected":
                # Increment finding count, upsert active finding
                await self.upsert_finding(event)

            case "FindingResolved":
                # Decrement finding count, update status
                await self.resolve_finding(event)

            case "QualityGateEvaluated":
                # Update gate status
                await self.update_gate_status(event)
```

---

## Pros

- **Complete audit trail**: Every metric computation, finding detection, and status change is permanently recorded. This enables answering questions like "when did coverage first drop below 80%?" or "who resolved this finding and when?" without additional logging infrastructure.
- **Retroactive analysis**: When new metrics or AI models are added, the entire event history can be replayed to compute new projections. For example, adding a "hotspot risk score" metric later can be back-calculated across all historical events.
- **Natural fit for trend tracking**: The append-only event log IS the trend data. No separate mechanism is needed to capture metric evolution over time -- it emerges directly from the event stream.
- **Independent read model scaling**: Each projection (dashboard, trends, alerts) can be scaled independently. A slow trend query does not affect dashboard responsiveness because they read from different tables.
- **CI/CD integration**: Analysis events naturally map to webhook/event-driven CI/CD integration. Quality gate events can trigger pipeline pass/fail decisions without polling.
- **Temporal queries**: "What was the state of repository X at date Y?" is answered by replaying events up to that date -- no point-in-time snapshots needed.

## Cons

- **Complexity overhead**: Event sourcing introduces significant architectural complexity (event store, projections, projection rebuilds, idempotency, ordering guarantees). For a team unfamiliar with the pattern, this is a steep learning curve.
- **Eventual consistency**: Read-side projections are eventually consistent with the event store. Dashboard data may lag behind the latest analysis by seconds to minutes depending on projection processing speed.
- **Projection rebuilds**: When projection logic changes (e.g., a new dashboard view or a metric calculation correction), the entire event history must be replayed. With millions of events, this can take hours.
- **Storage growth**: The event store grows without bound since events are never deleted. A repository with 365 daily analyses, 10K files, and 15 metrics per file produces ~54.75M events/year. JSONB payloads amplify storage requirements versus normalized columns.
- **Debugging difficulty**: Tracing a dashboard value back to the source events requires understanding the projection logic and event sequence. Standard SQL debugging skills do not transfer directly.
- **Query complexity for ad-hoc analysis**: Ad-hoc queries that were simple JOINs in a normalized model now require either querying projections (which may not have the right shape) or scanning the event store (which is slow for analytical queries).
- **Tooling maturity**: Event sourcing tooling in the Node.js/TypeScript ecosystem is less mature than relational ORMs. Libraries like EventStoreDB or Marten (.NET) are solid but niche.

---

## Technology Recommendations

| Layer | Technology |
|-------|-----------|
| Event Store | PostgreSQL (with JSONB events) or EventStoreDB |
| Event Bus | Apache Kafka or NATS JetStream for event distribution |
| Projection Store | PostgreSQL for materialized projections |
| Projection Framework | Custom handlers or Marten (if .NET) |
| Cache | Redis for hot projection caching |
| Search | Elasticsearch for full-text finding search |
| API | GraphQL (ideal for multiple projection shapes) |

---

## Migration and Scaling Considerations

- **Event store partitioning**: Partition the event_store table by `occurred_at` (monthly ranges) to bound scan sizes for projection rebuilds and enable efficient archival.
- **Snapshotting**: For streams with thousands of events (e.g., a frequently-scanned repository), store periodic snapshots to avoid replaying the entire stream on every read. Snapshot every 100-500 events per stream.
- **Projection versioning**: Maintain a version number for each projection. When the projection logic changes, create a new version and rebuild from scratch while the old version continues serving reads. Swap atomically when the rebuild completes.
- **Event archival**: Archive events older than N years to cold storage (S3, Glacier) but keep a compressed summary projection for long-term trend charts.
- **Kafka for scale**: At high volume (1000+ repositories, continuous scanning), replace direct PostgreSQL event store writes with Kafka as the primary event bus, and use a consumer to persist events to PostgreSQL for durability and queryability.
- **Idempotency**: All projection handlers must be idempotent (replayable without duplication). Use the event_id as a deduplication key.
- **CQRS boundary**: Start with a single write database and separate read projections. Split into physically separate databases only when read/write contention becomes measurable.
- **Migration from relational**: If starting with Suggestion 1 (normalized relational) and migrating later, the relational tables can be converted into initial events via a one-time "event hydration" migration script.
