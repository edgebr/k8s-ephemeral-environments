# Pilot Project Integration - Progress Tracker

**Project:** htm-gestor-documentos | **Branch:** `feat/k8s-ee-integration` (from `gd-sprint-17`)
**Cluster:** `ubuntu@13.58.99.235` | **Domain:** `*.k8s-ee.edge.net.br` | **Registry:** ECR (us-east-2)

## Completed Steps

- [x] **Step 1:** Create branch (`feat/k8s-ee-integration` from `gd-sprint-17`) ✓
- [x] **Step 2:** AUTH_BYPASS_LDAP — bypass logic + 9 unit tests (added NODE_ENV production guard) ✓
- [x] **Step 3:** Express static file serving — SPA fallback middleware + 11 unit tests ✓
- [x] **Step 4:** Dockerfile.k8s-ee + entrypoint + .dockerignore update ✓
- [x] **Steps 5-7:** k8s-ee.yaml + workflow ✓

## In Progress

- [ ] **End-to-end verification:** Waiting for workflow with CORS fix to complete

## Pending

- [ ] Revert `trigger: automatic` back to `trigger: on-demand` after confirming deployment works
- [ ] Merge workflow file to default branch so `/deploy-preview` command works
- [ ] Clean up empty trigger commits on the feature branch
- [ ] End-to-end login test: `admin` / `Senh@Valida123`

## Commits (htm-gestor-documentos)

| # | Message | Status |
|---|---------|--------|
| 1 | `feat(auth): add AUTH_BYPASS_LDAP for ephemeral environments` | ✅ Done (00e16287e) |
| 2 | `feat(backend): add static file serving for combined image mode` | ✅ Done (60c21ca65) |
| 3 | `feat(docker): add Dockerfile.k8s-ee and entrypoint` | ✅ Done (e1bfcc95b) |
| 4 | `feat: add k8s-ee platform configuration and workflow` | ✅ Done (ee5dc3d48) |
| 5 | `fix(config): use NODE_ENV=staging to avoid AUTH_BYPASS_LDAP conflict` | ✅ Done (43a5d6770) |
| 6 | `fix(docker): create upload directory in k8s-ee image` | ✅ Done (d8f9674b6) |
| 7 | `fix(cors): allow preview URL origin for ephemeral environments` | ✅ Done (e1bc200cc) |

## Platform Fixes (k8s-ephemeral-environments)

Issues discovered and fixed during pilot deployment:

| # | Commit | Issue | Root Cause |
|---|--------|-------|------------|
| 1 | `1995124` | ECR login failed ("security token is invalid") | `docker/login-action` overrides username/password for ECR URLs; replaced with `aws-actions/configure-aws-credentials` + `aws-actions/amazon-ecr-login` |
| 2 | `f9147ee` | SARIF upload failed on private repos | Private repos need GitHub Advanced Security for code scanning; added `continue-on-error: true` |
| 3 | `24e8605` | SARIF failure had no user-friendly message | Added `::warning::` step explaining the limitation |
| 4 | `8872a1e` | User-defined env vars not reaching pods | `env` section parsed by validate-config but never passed to deploy-app → Helm; threaded `env-json` through the entire pipeline |
| 5 | `d3a9d2d` | Health check rejected valid responses | Health check required `"status": "ok"` but apps use various formats; now accepts any HTTP 200 response |

## Deployment Issues Found & Fixed

### 1. ECR Login (platform fix #1)
`docker/login-action` detects ECR registry URLs and overrides username/password with AWS SDK auth, ignoring the provided credentials. Replaced with the AWS-recommended pattern: `aws-actions/configure-aws-credentials` + `aws-actions/amazon-ecr-login`.

### 2. SARIF Upload on Private Repos (platform fixes #2-3)
Private repos without GitHub Advanced Security cannot upload SARIF to the Security tab. Added `continue-on-error: true` with a `::warning::` message explaining the limitation. Trivy scan results remain available as downloadable artifacts.

### 3. Missing User-Defined Environment Variables (platform fix #4)
The `env` section in `k8s-ee.yaml` was parsed correctly by `validate-config` but never passed through the pipeline to Helm. The ConfigMap only contained platform metadata (PORT, PR_NUMBER, etc.), causing the app to crash on missing JWT_SECRET, NODE_ENV, etc.

**Fix:** Added `env-json` output/input/passthrough across three files:
- `validate-config/action.yml` → outputs `env-json`
- `pr-environment-reusable.yml` → threads it to deploy-app
- `deploy-app/action.yml` → passes `--set-json env=$ENV_JSON` to Helm

### 4. Health Check Too Strict (platform fix #5)
The deploy-app health check required `"status": "ok"` or `"status": "healthy"` in the JSON response, but the pilot app returns `{"success":true,"message":"API rodando"}`. Since `curl -sf` already ensures HTTP 200, changed to accept any non-empty response.

### 5. Upload Directory Permission (app fix)
The app calls `mkdirSync('upload/')` at startup but runs as non-root user (`node`). Added `RUN mkdir -p /app/upload && chown node:node /app/upload` to `Dockerfile.k8s-ee` before `USER node`.

### 6. CORS Blocking Preview URL (app fix)
The app's CORS allowlist was hardcoded and didn't include the preview domain. Added `process.env.PREVIEW_URL` (platform-injected) to the allowed origins list.

## Verification Checklist

### Local (before pushing)

- [x] Unit tests pass: `yarn test --testPathPatterns "auth-ldap|spa-fallback"` (20 new tests)
- [x] Full test suite passes: `yarn test` (5238 tests, no regressions)
- [x] Docker build succeeds: `docker build -f Dockerfile.k8s-ee -t htm-gestor-docs:local .`
- [x] Image architecture: amd64
- [x] Static files present: `/app/public/index.html`
- [x] Entrypoint present: `/entrypoint.sh`

### End-to-end (after pushing)

- [x] PR opened against `gd-sprint-17` (PR #1433)
- [ ] ~~`/deploy-preview` comment triggers workflow~~ (requires workflow on default branch; using `trigger: automatic` instead)
- [x] Namespace created: `htm-gestor-docs-pr-1433`
- [x] Image built and pushed to ECR
- [x] Migrations run successfully (33 migrations applied)
- [x] Seed data created (42 users, 152 documents, etc.)
- [x] App starts without crashes
- [x] Env vars in ConfigMap: NODE_ENV, JWT_SECRET, AUTH_BYPASS_LDAP, etc.
- [ ] Preview URL resolves: `https://htm-gestor-docs-pr-1433.k8s-ee.edge.net.br`
- [ ] Login works: `admin` / `Senh@Valida123`
- [ ] Frontend loads (Vue SPA)
- [ ] API responds (`/api/`)
- [ ] PR close triggers namespace cleanup

## External Dependencies

- [x] EC2 cluster running with k3s (v1.34.4+k3s1)
- [x] ARC runner registered with GitHub
- [x] KUBECONFIG secret set on edgebr org
- [x] DNS wildcard `*.k8s-ee.edge.net.br` → EC2 Elastic IP
- [x] ECR credentials (org secrets: `ECR_AWS_ACCESS_KEY_ID`, `ECR_AWS_SECRET_ACCESS_KEY`)
- [x] TLS via Let's Encrypt + Route 53

## Log

| Date | Action | Result |
|------|--------|--------|
| 2026-03-14 | Plan created | `docs/plans/migration-htm-pilot.md` — 7 steps, 4 commits, 17 unit tests |
| 2026-03-14 | Steps 1-7 implemented | 5 commits on `feat/k8s-ee-integration`, 20 unit tests |
| 2026-03-14 | First deploy attempt | ECR login failed — `docker/login-action` incompatible with org secrets |
| 2026-03-14 | Platform fix: ECR login | Replaced with `aws-actions/configure-aws-credentials` + `amazon-ecr-login` |
| 2026-03-14 | Platform fix: SARIF upload | Added `continue-on-error: true` + `::warning::` for private repos |
| 2026-03-14 | Platform fix: env injection | Threaded `env-json` from validate-config → deploy-app → Helm |
| 2026-03-14 | Platform fix: health check | Accept any HTTP 200 response instead of requiring specific JSON format |
| 2026-03-14 | App fix: upload dir | Pre-create `/app/upload` with correct ownership in Dockerfile |
| 2026-03-14 | App fix: CORS | Add `PREVIEW_URL` to CORS allowed origins |
| 2026-03-14 | Documentation updated | Config reference, onboarding guide, troubleshooting guide |
