# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-01-25)

**Core value:** Enable htm-gestor-documentos team to get fully functional PR preview environments (combined image + mock auth + PostgreSQL + MinIO) on AWS infrastructure in under 10 minutes
**Current focus:** Phase 4 - Pilot Project Integration

## Current Position

Phase: 4 of 4 (Pilot Project Integration)
Plan: 0 of 1 in current phase
Status: Ready to plan (Phases 1-3 done; Phases 4-5 removed — mock auth + combined image chosen)
Last activity: 2026-03-14 - Decided mock auth + combined image; Samba AD discarded; multi-container deferred

Progress: [=======---] 75% (Phases 1-3 done, Phase 4 remaining)

## Performance Metrics

**Velocity:**
- Total plans completed: 0
- Average duration: -
- Total execution time: 0 hours

**By Phase:**

| Phase | Plans | Total | Avg/Plan |
|-------|-------|-------|----------|
| - | - | - | - |

**Recent Trend:**
- Last 5 plans: -
- Trend: -

*Updated after each plan completion*

## Accumulated Context

### Decisions

Decisions are logged in PROJECT.md Key Decisions table.
Recent decisions affecting current work:

- VPS (k3s) over EKS for pilot: Faster setup, lower cost, proven architecture
- ECR over GHCR: **Decided ECR, implemented** — repo is private (must stay private), GHCR not viable. Registry-agnostic approach: same code supports both, selected by `registry-type` workflow input. No sync conflicts with upstream.
- **Combined single image** (not multi-container pod): Works today without multi-container support, simpler resource management
- **Mock authentication** (not real Samba AD): AUTH_BYPASS_LDAP + seeded test users; sufficient for preview environments
- Multi-container support: **Deferred** to main project roadmap (not needed for pilot)
- Samba AD chart: **Discarded** — mock auth chosen, not worth the complexity
- PostgreSQL only (no MariaDB): Client doesn't need MariaDB
- Domain changed: `*.k8s-ee.edge.net.br` (was `*.preview.edge.ufal.br`)
- TLS via Let's Encrypt + Route 53 DNS challenge (was ACM)

### Open Decisions

All decisions resolved as of 2026-03-14.

1. ~~**Authentication:**~~ **Decided: Mock auth** — AUTH_BYPASS_LDAP + seeded test users
2. ~~**Container strategy:**~~ **Decided: Combined single image** — one Dockerfile bundles backend + frontend
3. ~~**Image registry:**~~ **Decided: ECR** — repo must stay private, registry-agnostic approach (no fork divergence)

### Pending Todos

None yet.

### Blockers/Concerns

**From Research:**
- ~~Samba AD requires privileged container (P0 risk)~~ — **resolved**: Samba AD discarded, using mock auth
- ~~ECR OIDC trust policy must handle both pull_request and push events~~ — **resolved**: using static IAM credentials (org secrets), not OIDC
- ~~ARM64 availability for Samba base image uncertain~~ — **resolved**: EC2 cluster is x86_64 (moot point — Samba AD discarded)

## Session Continuity

Last session: 2026-03-14
Stopped at: All decisions resolved (mock auth + combined image); Phases 1-3 complete; ready to plan Phase 4 (Pilot Project Integration)
Resume file: None
