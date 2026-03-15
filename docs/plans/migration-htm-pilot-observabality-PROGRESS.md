# Progress: Add Prometheus Metrics Instrumentation to htm-gestor-documentos

> Full plan: [migration-htm-pilot-observabality.md](./migration-htm-pilot-observabality.md)

## Progress Tracker

| Phase | Step | Description | Status | Commit |
|-------|------|-------------|--------|--------|
| **Phase 1** | Step 1 | Install prom-client | NOT STARTED | — |
| | Step 2 | Create MetricsService | NOT STARTED | — |
| **Phase 2** | Step 3 | Create HTTP metrics middleware | NOT STARTED | — |
| **Phase 3** | Step 4 | Create Prisma metrics extension | NOT STARTED | — |
| | Step 5 | Wire extension into Prisma chain | NOT STARTED | — |
| | Step 6 | Create pool metrics collector | NOT STARTED | — |
| **Phase 4** | Step 7 | Create metrics feature (vertical slice) + wire into app | NOT STARTED | — |
| **Phase 5** | Step 8 | Config Reference (docs) | NOT STARTED | — |
| | Step 9 | Config Reference (wiki) | NOT STARTED | — |
| | Step 10 | Grafana Dashboards (wiki) | NOT STARTED | — |
| | Step 11 | Onboarding Guide (docs) | NOT STARTED | — |

## Phase Acceptance Criteria

### Phase 1: Core Metrics Infrastructure
- [ ] `prom-client` installed and importable
- [ ] All 6 metrics defined matching the Grafana dashboard PromQL contract
- [ ] `getMetrics()` returns valid Prometheus text format output
- [ ] Unit tests for MetricsService pass
- [ ] Code reviewed by `code-reviewer` agent
- [ ] Committed

### Phase 2: HTTP Metrics
- [ ] HTTP metrics middleware created and tested
- [ ] Path normalization prevents label explosion (UUIDs, numeric IDs)
- [ ] `/metrics` and `/assets/*` excluded from tracking
- [ ] All unit tests pass
- [ ] Code reviewed by `code-reviewer` agent
- [ ] Committed

### Phase 3: Database Metrics
- [ ] Every Prisma query records duration with operation and success labels
- [ ] Pool gauges updated periodically from `pg_stat_activity`
- [ ] `basePrisma` exported for pool collector use
- [ ] Both `prisma` and `prismaUnfiltered` wrapped with metrics extension
- [ ] All unit tests pass (extension + pool collector)
- [ ] Code reviewed by `code-reviewer` agent
- [ ] Committed

### Phase 4: Integration
- [ ] `GET /metrics` returns valid Prometheus text format
- [ ] `/metrics` endpoint responds independently of CORS and `/api` content-type middleware
- [ ] HTTP metrics middleware captures all API requests
- [ ] Pool collector starts on server boot and interval cleared on shutdown
- [ ] App starts normally — no regressions to existing functionality
- [ ] All existing tests still pass (`npm test`)
- [ ] Code reviewed by `code-reviewer` agent
- [ ] Committed

### Phase 5: Documentation
- [ ] Config reference (docs + wiki) documents required metrics, labels, and example code
- [ ] Grafana Dashboards wiki documents panel-to-metric mapping and troubleshooting
- [ ] Onboarding guide warns about metrics requirements when `metrics.enabled: true`
- [ ] Demo-app referenced as canonical implementation example
- [ ] Code reviewed by `code-reviewer` agent
- [ ] Committed

## Verification Checklist (after all phases complete)

- [ ] `curl -s http://localhost:3000/metrics` returns Prometheus text format
- [ ] `http_requests_total` increments on API requests
- [ ] `/metrics` itself is NOT counted in `http_requests_total`
- [ ] `db_pool_connections_total` > 0
- [ ] `db_query_duration_seconds` has observations after DB-hitting requests
- [ ] All tests pass (`npm test`)
- [ ] Grafana dashboard shows App Status = UP, DB Connected = YES
