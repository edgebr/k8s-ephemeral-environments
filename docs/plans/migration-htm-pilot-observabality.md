# Plan: Add Prometheus Metrics Instrumentation to htm-gestor-documentos

## Context

The Grafana "PR Developer Insights" dashboard shows app health, HTTP performance, and database status for each PR environment. The pilot app `htm-gestor-documentos` has `metrics.enabled: true` in `k8s-ee.yaml`, which deploys a ServiceMonitor telling Prometheus to scrape `GET /metrics` on the app. But the app doesn't expose `/metrics` or any Prometheus metrics — the dashboard shows App Status = DOWN, DB Connected = NO, and all other panels empty.

**Platform side is complete** — ServiceMonitor, Prometheus scraping, Grafana dashboard all work (proven by demo-app). Only the **pilot app** needs changes.

**Documentation must be updated alongside each code step** — config reference (docs + wiki), Grafana dashboard wiki, and onboarding guide all need updates to document the metrics instrumentation requirements.

---

## Workflow Per Step

Every step follows this process:

1. **Write code** — implementation files
2. **Write tests** — unit tests for the new code (Jest 30.2.0 + node-mocks-http for middleware)
3. **Run tests** — `npm test` (or `yarn test`) to verify all pass
4. **Run code-reviewer agent** — use the custom `code-reviewer` subagent (NOT coderabbit or superpowers variants) to review changes before committing
5. **Fix any issues** from the review
6. **Commit** — conventional commits, **no co-authors in commit messages**

**Project conventions** (must follow):
- Vertical slice architecture: each endpoint lives in `features/{feature}/` with controller, route, and tests
- Business API controllers must have `@swagger` JSDoc documentation (Brazilian Portuguese descriptions)
- Infrastructure endpoints (like `/metrics`) are exempt from swagger — the server base URL is `/api` so root-level paths resolve incorrectly in swagger-jsdoc
- Regenerate `swagger-output.json` with `yarn swagger` after modifying business API controllers

**Test conventions** (from existing codebase):
- Test files: `*.test.ts` colocated with source files (same directory)
- Framework: Jest 30.2.0, ts-jest 29.4.6
- Middleware testing: `node-mocks-http` for mock Request/Response
- Path aliases: `@shared/*`, `@features/*`
- Pattern: `jest.mock()` for module mocking, `beforeEach` for mock reset
- Coverage: `npm test` runs with `--coverage`
- Existing middleware test examples: `authorize.middleware.test.ts` (282 lines), `branch-context.middleware.test.ts` (94 lines)

---

## Dashboard Contract (what the app must expose)

| Dashboard Panel | PromQL Metric | Type | Labels |
|----------------|---------------|------|--------|
| App Status | `up` | Auto (just needs `/metrics` responding) | — |
| Error Rate | `http_requests_total{status_code=~"5.."}` | Counter | `method`, `route`, `status_code` |
| P95 Latency | `http_request_duration_seconds_bucket` | Histogram | `method`, `route`, `status_code` |
| Request Rate | `http_requests_total` | Counter | `method`, `route`, `status_code` |
| DB Connected | `db_pool_connections_total` | Gauge | — |
| 5xx by Endpoint | `http_requests_total{status_code=~"5.."}` by `route` | Counter | `method`, `route`, `status_code` |
| P95 by Endpoint | `http_request_duration_seconds_bucket` by `route` | Histogram | `method`, `route`, `status_code` |
| Requests by Status | `http_requests_total` by `status_code` | Counter | `method`, `route`, `status_code` |
| Slowest Endpoints | `http_request_duration_seconds_bucket` by `route` | Histogram | `method`, `route`, `status_code` |
| Connection Pool | `db_pool_connections_total`, `_idle`, `_waiting` | Gauge | — |
| Query Duration | `db_query_duration_seconds_bucket` by `operation` | Histogram | `operation`, `success` |
| Failed Queries | `db_query_duration_seconds_count{success="false"}` | Histogram | `operation`, `success` |
| Pods Running | `kube_pod_status_phase` | Auto (kube-state-metrics) | — |
| Error Logs | Loki query | Auto (promtail) | — |

---

## Phase 1: Core Metrics Infrastructure (Steps 1-2)

**Goal:** Install `prom-client` and create the central metrics registry with all metric definitions.

**Phase acceptance criteria:**
- [ ] `prom-client` installed and importable
- [ ] All 6 metrics defined matching the Grafana dashboard PromQL contract
- [ ] `getMetrics()` returns valid Prometheus text format output
- [ ] Unit tests for MetricsService pass
- [ ] Code reviewed by `code-reviewer` agent
- [ ] Committed

---

### Step 1: Install prom-client dependency

**Repo:** `htm-gestor-documentos`
**Action:** Run `npm install prom-client` (or `yarn add prom-client`) in `backend/`

**File modified:**
- `backend/package.json` — `prom-client` added to `dependencies`

**Acceptance criteria:**
- [ ] `prom-client` appears in `dependencies` in `package.json`
- [ ] Lock file updated
- [ ] `npm ls prom-client` (or `yarn why prom-client`) shows it installed

---

### Step 2: Create MetricsService singleton module

**Repo:** `htm-gestor-documentos`
**New file:** `backend/src/shared/infrastructure/metrics/metrics.service.ts`

Singleton module (NOT an Awilix class — it must be available at import time before DI container initializes). Creates a custom `prom-client` Registry with all metric definitions.

**Metric definitions:**

| Metric Name | Type | Labels | Buckets |
|-------------|------|--------|---------|
| `http_requests_total` | Counter | `method`, `route`, `status_code` | — |
| `http_request_duration_seconds` | Histogram | `method`, `route`, `status_code` | `[0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]` |
| `db_pool_connections_total` | Gauge | — | — |
| `db_pool_connections_idle` | Gauge | — | — |
| `db_pool_connections_waiting` | Gauge | — | — |
| `db_query_duration_seconds` | Histogram | `operation`, `success` | `[0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1]` |

**Implementation details:**
- Custom registry with default labels: `{ app: 'htm-gestor-docs', pr: process.env.PR_NUMBER || 'unknown' }`
- Call `collectDefaultMetrics({ register: registry })` for Node.js runtime metrics (heap, event loop, etc.)
- Export: individual metric instances, `getMetrics()` (async, returns `registry.metrics()`), `getContentType()` (returns `registry.contentType`)

**Reference file:** `k8s-ephemeral-environments/demo-app/apps/api/src/metrics/metrics.service.ts`

**Tests:** `backend/src/shared/infrastructure/metrics/metrics.service.test.ts`
- [ ] Test that registry is created with correct default labels (`app`, `pr`)
- [ ] Test that all 6 metrics are defined with correct names and types
- [ ] Test that `getMetrics()` returns a string containing expected metric names
- [ ] Test that `getContentType()` returns Prometheus content type
- [ ] Test that `collectDefaultMetrics` populates `nodejs_*` metrics
- [ ] Test that `PR_NUMBER` env var is used when set, defaults to `'unknown'`
- [ ] Use `jest.resetModules()` in `beforeEach` to avoid "metric already registered" errors between tests (follows existing pattern from `authorize.middleware.test.ts`)

**Acceptance criteria:**
- [ ] File compiles without errors
- [ ] All 6 metrics defined with correct names, types, labels, and buckets matching the dashboard PromQL
- [ ] Default labels include `app` and `pr`
- [ ] `getMetrics()` returns valid Prometheus text format
- [ ] `getContentType()` returns the Prometheus content type string
- [ ] All tests pass

---

## Phase 2: HTTP Metrics (Steps 3)

**Goal:** Instrument all HTTP requests with counter and histogram metrics, enabling 7 of 14 dashboard panels (Error Rate, P95 Latency, Request Rate, 5xx by Endpoint, P95 by Endpoint, Requests by Status, Slowest Endpoints).

**Phase acceptance criteria:**
- [ ] HTTP metrics middleware created and tested
- [ ] Path normalization prevents label explosion (UUIDs, numeric IDs)
- [ ] `/metrics` and `/assets/*` excluded from tracking
- [ ] All unit tests pass
- [ ] Code reviewed by `code-reviewer` agent
- [ ] Committed

---

### Step 3: Create HTTP metrics middleware

**Repo:** `htm-gestor-documentos`
**New file:** `backend/src/shared/middleware/metrics.middleware.ts`

Standard Express middleware function (same pattern as `date-formatter.middleware.ts`).

**Logic:**
1. Skip if `req.path === '/metrics'` or `req.path.startsWith('/assets/')` (prevents recursion and high cardinality)
2. Record `startTime = process.hrtime.bigint()`
3. Listen on `res.on('finish', callback)`
4. In callback: calculate duration, normalize route, increment counter + observe histogram
5. Call `next()`

**Path normalization** (prevents label explosion):
- Strip query strings first: `.split('?')[0]`
- UUIDs → `:uuid` (regex: `/[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}/gi`)
- Numeric IDs → `/:id` (regex: `/\/\d+(?=\/|$)/g`)

**Labels recorded:** `{ method: req.method, route: normalizedPath, status_code: res.statusCode.toString() }`

**Reference file:** `k8s-ephemeral-environments/demo-app/apps/api/src/middleware/metrics.middleware.ts`

**Tests:** `backend/src/shared/middleware/metrics.middleware.test.ts`

Following existing patterns from `authorize.middleware.test.ts` and `branch-context.middleware.test.ts`:
- [ ] Test that `/metrics` requests are skipped (not tracked)
- [ ] Test that `/assets/*` requests are skipped
- [ ] Test that normal requests increment `http_requests_total` with correct labels
- [ ] Test that normal requests observe `http_request_duration_seconds`
- [ ] Test path normalization: `/api/users/123` → `/api/users/:id`
- [ ] Test path normalization: `/api/docs/550e8400-e29b-41d4-a716-446655440000` → `/api/docs/:uuid`
- [ ] Test that `req.route.path` is preferred when available (in `res.on('finish')` callback)
- [ ] Test path normalization strips query strings: `/api/users?page=1` → `/api/users`
- [ ] Test that errors in metric recording don't break the request (next() still called)
- [ ] Test labels: method, route, status_code are correctly set

**Acceptance criteria:**
- [ ] Middleware intercepts all requests except `/metrics` and `/assets/*`
- [ ] Each request increments `http_requests_total` with correct labels
- [ ] Each request observes `http_request_duration_seconds` with duration in seconds
- [ ] UUIDs in paths normalized to `:uuid`
- [ ] Numeric IDs in paths normalized to `:id`
- [ ] Errors in metric recording are caught and logged (never break the request)
- [ ] All tests pass

---

## Phase 3: Database Metrics (Steps 4-6)

**Goal:** Instrument Prisma queries and database pool connections, enabling 4 dashboard panels (DB Connected, Connection Pool, Query Duration, Failed Queries).

**Phase acceptance criteria:**
- [ ] Every Prisma query records duration with operation and success labels
- [ ] Pool gauges updated periodically from `pg_stat_activity`
- [ ] `basePrisma` exported for pool collector use
- [ ] Both `prisma` and `prismaUnfiltered` wrapped with metrics extension
- [ ] All unit tests pass (extension + pool collector)
- [ ] Code reviewed by `code-reviewer` agent
- [ ] Committed

---

### Step 4: Create Prisma metrics extension

**Repo:** `htm-gestor-documentos`
**New file:** `backend/src/shared/infrastructure/database/prisma-metrics-extension.ts`

Prisma `$extends` extension following the exact same pattern as `prisma-audit-extension.ts` (lines 28-73).

**Logic:**
```
query.$allModels.$allOperations → time the query, observe db_query_duration_seconds
```

- `operation` label = Prisma operation name (`findUnique`, `findMany`, `create`, `update`, `delete`, `count`, etc.)
- `success` label = `"true"` or `"false"` (string, matching demo-app convention)

**Reference files:**
- `htm-gestor-documentos/backend/src/shared/infrastructure/database/prisma-audit-extension.ts` (pattern to follow)
- `k8s-ephemeral-environments/demo-app/apps/api/src/database.service.ts` (metric usage pattern)

**Tests:** `backend/src/shared/infrastructure/database/prisma-metrics-extension.test.ts`
- [ ] Test that successful queries observe `db_query_duration_seconds` with `success: "true"`
- [ ] Test that failed queries observe `db_query_duration_seconds` with `success: "false"` and re-throw the error
- [ ] Test that `operation` label matches the Prisma operation name passed (e.g., `findMany`, `create`)
- [ ] Test that duration is recorded as a positive number in seconds
- [ ] Test that the extension passes through query results unchanged

**Acceptance criteria:**
- [ ] Every Prisma query (any model, any operation) records duration in `db_query_duration_seconds`
- [ ] Failed queries record with `success: "false"`
- [ ] `operation` label matches Prisma operation names (findMany, create, update, delete, count, etc.)
- [ ] Extension follows same `ExtendableClient` interface pattern as audit extension
- [ ] All tests pass

---

### Step 5: Wire metrics extension into Prisma client chain

**Repo:** `htm-gestor-documentos`
**Modified file:** `backend/src/shared/infrastructure/database/prisma.ts`

**Changes:**
1. Import `createMetricsExtension` from `./prisma-metrics-extension`
2. Add metrics as outermost extension in both client chains:
   - `prisma = createMetricsExtension(createAuditExtension(createBranchFilteredClient(basePrisma)))` (was: `createAuditExtension(createBranchFilteredClient(basePrisma))`)
   - `prismaUnfiltered = createMetricsExtension(createAuditExtension(basePrisma))` (was: `createAuditExtension(basePrisma)`)
3. Export `basePrisma` for use by the pool collector (currently `const`, not exported). Add a JSDoc warning: `/** @internal Used only by db-pool-collector for pg_stat_activity queries. Do not use for application queries — use prisma or prismaUnfiltered instead. */`

**Why outermost:** Measures total query time including branch filtering + audit extension overhead.

**Acceptance criteria:**
- [ ] Both `prisma` and `prismaUnfiltered` clients wrapped with metrics extension
- [ ] `basePrisma` exported (e.g., `export { basePrisma }`)
- [ ] Existing functionality unchanged — audit and branch filtering still work
- [ ] Dev-mode global cache updated to use the new extended clients

---

### Step 6: Create database pool metrics collector

**Repo:** `htm-gestor-documentos`
**New file:** `backend/src/shared/infrastructure/metrics/db-pool-collector.ts`

Periodic `pg_stat_activity` query to approximate pool metrics (Prisma doesn't expose pool stats directly).

**Logic:**
- Uses `basePrisma.$queryRaw` to query `pg_stat_activity`
- Updates `db_pool_connections_total`, `db_pool_connections_idle`, `db_pool_connections_waiting` gauges
- Runs every 30 seconds (matches ServiceMonitor scrape interval — no point querying more often than Prometheus reads)
- Errors silently caught (pool metrics are best-effort)

**SQL query:**
```sql
SELECT
  count(*) AS total,
  count(*) FILTER (WHERE state = 'idle') AS idle
FROM pg_stat_activity
WHERE datname = current_database()
  AND pid != pg_backend_pid()
```

**Gauge mapping:**
- `db_pool_connections_total` ← `total`
- `db_pool_connections_idle` ← `idle`
- `db_pool_connections_waiting` ← always `0` (Prisma's internal Rust query engine manages its own connection pool and does not expose waiting queue metrics — `pg_stat_activity` has no concept of ORM-level pool queuing)

> **Note:** `pg_stat_activity` shows PostgreSQL backend processes, not Prisma pool connections directly. This is an approximation. For the "DB Connected" panel (checks `total > 0`), this is accurate. Add a code comment explaining this limitation.

**Exports:** `startDbPoolCollector(intervalMs?: number): NodeJS.Timeout`

**Tests:** `backend/src/shared/infrastructure/metrics/db-pool-collector.test.ts`
- [ ] Test that `startDbPoolCollector()` calls `$queryRaw` and updates all 3 gauges
- [ ] Test that errors in `$queryRaw` are caught silently (no throw)
- [ ] Test that the interval handle is returned for cleanup
- [ ] Test that gauge values reflect the query result (total, idle, active)
- [ ] Test with empty result (no connections) — gauges should not throw

**Acceptance criteria:**
- [ ] Pool gauges updated every 30 seconds
- [ ] "DB Connected" dashboard panel shows "YES" when DB has connections (total > 0)
- [ ] Errors in the collector do NOT crash the app
- [ ] Returns the interval handle (for cleanup if needed)
- [ ] All tests pass

---

## Phase 4: Integration (Step 7)

**Goal:** Wire all metrics components into the Express app. After this phase, the app exposes a working `/metrics` endpoint with all metrics.

**Phase acceptance criteria:**
- [ ] `GET /metrics` returns valid Prometheus text format
- [ ] `/metrics` endpoint responds independently of CORS and `/api` content-type middleware
- [ ] HTTP metrics middleware captures all API requests
- [ ] Pool collector starts on server boot and interval cleared on shutdown
- [ ] App starts normally — no regressions to existing functionality
- [ ] All existing tests still pass (`npm test`)
- [ ] Code reviewed by `code-reviewer` agent
- [ ] Committed

---

### Step 7: Create metrics feature (vertical slice) and wire into app

**Repo:** `htm-gestor-documentos`

The `/metrics` endpoint follows the vertical slice architecture with a feature directory, controller (with swagger docs), and route — same pattern as `health-check`. It is mounted at the **root level** in `index.ts` (like `/api-docs`), not through the `/api` router, because:
- The ServiceMonitor scrapes at `/metrics` (root), not `/api/metrics`
- It must be registered before ALL middleware (to avoid `dateFormatterMiddleware` parsing Prometheus text as JSON)
- It returns Prometheus text format, not the standard JSON response format

**New files:**

**`backend/src/features/metrics/get-metrics/get-metrics.controller.ts`**
- Does NOT extend `BaseController` (returns raw Prometheus text, not JSON `{ success, data, message }`)
- No `@swagger` docs — the `/metrics` endpoint is infrastructure (Prometheus scraping), not a business API. The swagger server base URL is `http://localhost:3000/api`, so a `/metrics` JSDoc path would incorrectly resolve to `/api/metrics`. Infrastructure endpoints don't belong in the API swagger docs.
- No auth required (Prometheus scrapes are unauthenticated, internal cluster)
- Imports `getMetrics()` and `getContentType()` from `metrics.service.ts`

**`backend/src/features/metrics/index.ts`**
- Creates Router with `GET /` handler wrapping controller in `asyncHandler()`
- Exports the router
- Follows the structure of `features/health-check/index.ts` but does NOT use Awilix DI (MetricsService is a module singleton, not an Awilix-registered class)

**Modified files:**

**`backend/src/index.ts`**

1. **Add imports:**
   ```typescript
   import metricsRoutes from './features/metrics/index';
   import { metricsMiddleware } from '@shared/middleware/metrics.middleware';
   import { startDbPoolCollector } from '@shared/infrastructure/metrics/db-pool-collector';
   ```

2. **Mount metrics feature immediately after `const app = express()`** (line ~31, before ALL `app.use()` calls):
   ```typescript
   // Prometheus metrics endpoint (before all middleware — returns text/plain, not JSON)
   app.use('/metrics', metricsRoutes);
   ```
   **Why before ALL middleware:** The `dateFormatterMiddleware` overrides `res.send()` and tries `JSON.parse()` on the body — Prometheus text format would trigger a `logger.warn()` fallback on every 30s scrape, creating unnecessary log noise. Placing `/metrics` before all `app.use()` calls avoids this.

3. **Register metrics middleware AFTER body parsers, BEFORE date formatter** (after line 59, before line 65):
   ```typescript
   app.use(metricsMiddleware);
   ```

4. **Start pool collector** in `startServer()` after `initializeServices()` call (after line 194), and store the handle for cleanup:
   ```typescript
   const poolCollectorInterval = startDbPoolCollector();
   ```

5. **Clear interval on shutdown** — in the `gracefulShutdown()` function inside `initializeServices()`, add `clearInterval(poolCollectorInterval)` before database disconnect. Pass the handle via a module-level variable or pass it to `initializeServices`.

**Updated middleware chain:**
```
const app = express()
app.use('/metrics', metricsRoutes)  ← NEW: vertical slice feature (before ALL middleware)
CORS
express.json()
express.urlencoded()
metricsMiddleware                   ← NEW: HTTP request tracking
Swagger UI (/api-docs)
dateFormatterMiddleware
Content-Type setter (/api)
Routes (/api)
Static files
SPA fallback
```

**Tests:** `backend/src/features/metrics/get-metrics/get-metrics.controller.test.ts`
- [ ] Test that `handle()` returns Prometheus text format
- [ ] Test that `Content-Type` header is set to Prometheus content type
- [ ] Test that response status is 200
- [ ] Test that response body contains expected metric names

**Acceptance criteria:**
- [ ] Feature follows vertical slice pattern (`features/metrics/` with controller, route, tests)
- [ ] `GET /metrics` returns Prometheus text format with correct content type
- [ ] `/metrics` endpoint NOT affected by CORS, body parsers, or `dateFormatterMiddleware`
- [ ] All HTTP requests (except `/metrics` and `/assets/*`) tracked by metrics middleware
- [ ] Pool collector starts after services initialize and interval cleared on shutdown
- [ ] App starts and serves normally — no regressions
- [ ] All tests pass

---

## Phase 5: Documentation (Steps 8-11)

**Goal:** Document the metrics instrumentation requirements so other developers know what their apps must expose for the Grafana dashboard to work.

**Phase acceptance criteria:**
- [ ] Config reference (docs + wiki) documents required metrics, labels, and example code
- [ ] Grafana Dashboards wiki documents panel-to-metric mapping and troubleshooting
- [ ] Onboarding guide warns about metrics requirements when `metrics.enabled: true`
- [ ] Demo-app referenced as canonical implementation example
- [ ] Code reviewed by `code-reviewer` agent
- [ ] Committed

---

### Step 8: Update documentation — Config Reference (docs)

**Repo:** `k8s-ephemeral-environments`
**Modified file:** `docs/guides/k8s-ee-config-reference.md`

**Changes:**
- In the `metrics` section (line ~564), add a subsection explaining **what the app must expose** for dashboard panels to work
- Add a table of required metrics with names, types, labels
- Add a "Metrics Instrumentation Guide" subsection with:
  - Link to demo-app as reference implementation
  - Minimum requirements: `/metrics` endpoint returning Prometheus format, `http_requests_total` counter, `http_request_duration_seconds` histogram
  - Optional DB metrics: pool gauges, query duration histogram
  - Note that `namespace` label is auto-injected by ServiceMonitor
- Add a Node.js/Express example snippet using `prom-client`

**Acceptance criteria:**
- [ ] Developer reading only this doc understands what metrics their app must expose
- [ ] Required metric names, types, labels, and buckets are documented
- [ ] Example code snippet included for Node.js/Express apps
- [ ] Demo-app referenced as full implementation example

---

### Step 9: Update documentation — Config Reference (wiki)

**Repo:** `k8s-ephemeral-environments.wiki`
**Modified file:** `Configuration-Reference.md`

**Changes:** Mirror the same metrics instrumentation content added to the docs guide in Step 8. The wiki `Configuration-Reference.md` tracks the docs version.

**Acceptance criteria:**
- [ ] Wiki metrics section matches docs guide content
- [ ] Same table of required metrics, example code, and dashboard panel mapping

---

### Step 10: Update documentation — Grafana Dashboards (wiki)

**Repo:** `k8s-ephemeral-environments.wiki`
**Modified file:** `Grafana-Dashboards.md`

**Changes:**
- Add a section **"PR Developer Insights — Required App Metrics"** documenting:
  - Which panels require app-side instrumentation vs auto-provided metrics
  - Table mapping each panel to its PromQL query and the metric the app must expose
  - Troubleshooting: "Dashboard shows DOWN/NO" → app doesn't expose `/metrics` → link to config reference

**Acceptance criteria:**
- [ ] Developer can look up which metrics feed which panels
- [ ] Clear troubleshooting path for "panels show no data"

---

### Step 11: Update documentation — Onboarding Guide

**Repo:** `k8s-ephemeral-environments`
**Modified file:** `docs/guides/onboarding-new-repo.md`

**Changes:**
- In the deployment/configuration section, add a note about metrics:
  - If `metrics.enabled: true`, the app **must** expose `/metrics` endpoint with Prometheus-format metrics
  - Link to config reference for required metric names
  - Note that without instrumentation, the Grafana dashboard will show misleading DOWN/NO indicators

**Acceptance criteria:**
- [ ] New repos being onboarded are warned about metrics requirements
- [ ] Clear link to detailed metrics documentation

---

## Files Summary

### Pilot repo (`htm-gestor-documentos`) — 6 new source, 5 new test, 3 modified

| Action | File |
|--------|------|
| Create | `backend/src/shared/infrastructure/metrics/metrics.service.ts` |
| Create | `backend/src/shared/infrastructure/metrics/metrics.service.test.ts` |
| Create | `backend/src/shared/middleware/metrics.middleware.ts` |
| Create | `backend/src/shared/middleware/metrics.middleware.test.ts` |
| Create | `backend/src/shared/infrastructure/database/prisma-metrics-extension.ts` |
| Create | `backend/src/shared/infrastructure/database/prisma-metrics-extension.test.ts` |
| Create | `backend/src/shared/infrastructure/metrics/db-pool-collector.ts` |
| Create | `backend/src/shared/infrastructure/metrics/db-pool-collector.test.ts` |
| Create | `backend/src/features/metrics/index.ts` |
| Create | `backend/src/features/metrics/get-metrics/get-metrics.controller.ts` |
| Create | `backend/src/features/metrics/get-metrics/get-metrics.controller.test.ts` |
| Modify | `backend/src/shared/infrastructure/database/prisma.ts` |
| Modify | `backend/src/index.ts` |
| Modify | `backend/package.json` (via npm/yarn) |

### Platform repo (`k8s-ephemeral-environments`) — 2 modified

| Action | File |
|--------|------|
| Modify | `docs/guides/k8s-ee-config-reference.md` |
| Modify | `docs/guides/onboarding-new-repo.md` |

### Wiki repo (`k8s-ephemeral-environments.wiki`) — 2 modified

| Action | File |
|--------|------|
| Modify | `Configuration-Reference.md` |
| Modify | `Grafana-Dashboards.md` |

---

## What is NOT needed

- No `schema.prisma` changes (no `previewFeatures`)
- No `k8s-ee.yaml` changes (already has `metrics.enabled: true`)
- No Grafana dashboard changes (PromQL already matches these metric names)
- No Awilix DI registration (singleton module avoids init-order issues)
- No Dockerfile changes (`prom-client` is pure JS, works on ARM64)
- No MinIO-specific metrics (not queried by any dashboard panel)

---

## Verification

### Local development verification

1. Start the backend: `yarn dev` (or `npm run dev`)
2. Verify `/metrics` endpoint:
   ```bash
   curl -s http://localhost:3000/metrics | head -50
   ```
   - Returns Prometheus text format (not JSON)
   - Contains `http_requests_total`, `http_request_duration_seconds`, `db_*` metrics, `nodejs_*` default metrics
3. Make API requests and re-check `/metrics`:
   ```bash
   curl http://localhost:3000/api/
   curl http://localhost:3000/api/health
   curl -s http://localhost:3000/metrics | grep http_requests_total
   ```
   - `http_requests_total` counter increments
   - `http_request_duration_seconds_bucket` has observations
4. Verify `/metrics` itself is NOT counted in `http_requests_total`
5. Verify `db_pool_connections_total` shows value > 0 (DB connected)
6. Make a DB-hitting request, verify `db_query_duration_seconds` has observations

### Live PR environment verification (on EC2 cluster)

1. Push changes, trigger workflow, wait for deployment
2. SSH to cluster and verify metrics endpoint:
   ```bash
   kubectl exec -n htm-gestor-docs-pr-{N} deploy/htm-gestor-docs-pr-{N}-app -- wget -qO- http://localhost:3000/metrics | head -30
   ```
3. Check Prometheus is scraping:
   ```bash
   kubectl exec prometheus-prometheus-prometheus-0 -n observability -c prometheus -- \
     wget -qO- 'http://localhost:9090/api/v1/query?query=up{namespace="htm-gestor-docs-pr-{N}"}'
   ```
   Should return `"value":[..., "1"]`
4. Open Grafana "PR Developer Insights" dashboard, select namespace `htm-gestor-docs-pr-{N}`:
   - **App Status** → UP (green)
   - **Error Rate** → 0% or a percentage
   - **P95 Latency** → time value
   - **Request Rate** → requests/sec
   - **DB Connected** → YES (green)
   - **Connection Pool** → Total/Idle/Waiting lines
   - **Query Duration** → per-operation latencies
   - **Requests by Status** → stacked areas by status code
