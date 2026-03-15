# Progress: Add Prometheus Metrics Instrumentation to htm-gestor-documentos

> Full plan: [migration-htm-pilot-observabality.md](./migration-htm-pilot-observabality.md)

## Progress Tracker

| Phase | Step | Description | Status | Commit |
|-------|------|-------------|--------|--------|
| **Phase 1** | Step 1 | Install prom-client | DONE | ef59c7857 |
| | Step 2 | Create MetricsService | DONE | 4924b74d4 |
| **Phase 2** | Step 3 | Create HTTP metrics middleware | DONE | f20936ee0 |
| **Phase 3** | Step 4 | Create Prisma metrics extension | DONE | 9abea29d0 |
| | Step 5 | Wire extension into Prisma chain | DONE | 9ecc27f59 |
| | Step 6 | Create pool metrics collector | DONE | 796d3e918 |
| **Phase 4** | Step 7 | Create metrics feature (vertical slice) + wire into app | DONE | e44547fc7 |
| **Phase 5** | Step 8 | Config Reference (docs) | DONE | f53a3bd |
| | Step 9 | Config Reference (wiki) | DONE | f737374 |
| | Step 10 | Grafana Dashboards (wiki) | DONE | f737374 |
| | Step 11 | Onboarding Guide (docs) | DONE | f53a3bd |

## Phase Acceptance Criteria

### Phase 1: Core Metrics Infrastructure
- [x] `prom-client` installed and importable
- [x] All 6 metrics defined matching the Grafana dashboard PromQL contract
- [x] `getMetrics()` returns valid Prometheus text format output
- [x] Unit tests for MetricsService pass
- [x] Code reviewed by `code-reviewer` agent
- [x] Committed

### Phase 2: HTTP Metrics
- [x] HTTP metrics middleware created and tested
- [x] Path normalization prevents label explosion (UUIDs, numeric IDs)
- [x] `/metrics` and `/assets/*` excluded from tracking
- [x] All unit tests pass
- [x] Code reviewed by `code-reviewer` agent
- [x] Committed

### Phase 3: Database Metrics
- [x] Every Prisma query records duration with operation and success labels
- [x] Pool gauges updated periodically from `pg_stat_activity`
- [x] `basePrisma` exported for pool collector use
- [x] Both `prisma` and `prismaUnfiltered` wrapped with metrics extension
- [x] All unit tests pass (extension + pool collector)
- [x] Code reviewed by `code-reviewer` agent
- [x] Committed

### Phase 4: Integration
- [x] `GET /metrics` returns valid Prometheus text format
- [x] `/metrics` endpoint responds independently of CORS and `/api` content-type middleware
- [x] HTTP metrics middleware captures all API requests
- [x] Pool collector starts on server boot and interval cleared on shutdown
- [x] App starts normally — no regressions to existing functionality
- [x] All existing tests still pass (`npm test`)
- [x] Code reviewed by `code-reviewer` agent
- [x] Committed

### Phase 5: Documentation
- [x] Config reference (docs + wiki) documents required metrics, labels, and example code
- [x] Grafana Dashboards wiki documents panel-to-metric mapping and troubleshooting
- [x] Onboarding guide warns about metrics requirements when `metrics.enabled: true`
- [x] Demo-app referenced as canonical implementation example
- [x] Code reviewed by `code-reviewer` agent
- [x] Committed

## Verification Checklist (after all phases complete)

- [x] `curl -s http://localhost:3000/metrics` returns Prometheus text format
- [x] `http_requests_total` increments on API requests
- [x] `/metrics` itself is NOT counted in `http_requests_total`
- [x] `db_pool_connections_total` > 0
- [x] `db_query_duration_seconds` has observations after DB-hitting requests
- [x] All tests pass (`npm test`)
- [x] Grafana dashboard shows App Status = UP, DB Connected = YES
