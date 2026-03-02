# On-Demand Trigger Mode Design

**Date:** 2026-03-01
**Status:** Approved

## Problem

All external repos using k8s-ephemeral-environments get an environment created on every PR automatically. For repos with high PR volume or limited cluster resources, this wastes compute and namespace capacity. Repo maintainers need a way to opt into on-demand environment creation via a comment command.

## Design

### Configuration

A new top-level `trigger` field in `k8s-ee.yaml`:

```yaml
projectId: myapp
trigger: on-demand    # "automatic" (default) or "on-demand"
```

- **`automatic`** (default): Current behavior. Environment created on every PR event. Fully backwards compatible.
- **`on-demand`**: Environment created only when someone comments `/deploy-preview` on the PR. After creation, pushes auto-redeploy. `/destroy-preview` tears it down early.

### Commands

| Command | Action |
|---------|--------|
| `/deploy-preview` | Creates or redeploys the PR environment |
| `/destroy-preview` | Destroys the environment (PR stays open) |

### Workflow Architecture

**Approach:** Separate reusable workflow (Approach A). The existing `pr-environment-reusable.yml` stays untouched.

**New files:**

1. **`.github/workflows/deploy-preview-reusable.yml`** — On-demand reusable workflow handling comment commands, auto-redeploy on push, and cleanup on PR close.
2. **`.github/workflows/deploy-preview.yml`** — Dogfooding workflow for this repo.

**External repo workflow for on-demand mode:**

```yaml
name: PR Environment (On-Demand)

on:
  issue_comment:
    types: [created]
  pull_request:
    types: [synchronize, closed]

concurrency:
  group: pr-env-${{ github.event.pull_request.number || github.event.issue.number }}
  cancel-in-progress: false

permissions:
  contents: read
  packages: write
  pull-requests: write
  security-events: write

jobs:
  deploy-preview:
    if: github.event_name == 'issue_comment'
    uses: koder-cat/k8s-ephemeral-environments/.github/workflows/deploy-preview-reusable.yml@main
    with:
      comment-body: ${{ github.event.comment.body }}
      comment-user: ${{ github.event.comment.user.login }}
      issue-number: ${{ github.event.issue.number }}
      repository: ${{ github.repository }}
    secrets: inherit

  auto-redeploy:
    if: github.event_name == 'pull_request' && github.event.action == 'synchronize'
    uses: koder-cat/k8s-ephemeral-environments/.github/workflows/deploy-preview-reusable.yml@main
    with:
      pr-number: ${{ github.event.pull_request.number }}
      pr-action: synchronize
      head-sha: ${{ github.event.pull_request.head.sha }}
      head-ref: ${{ github.head_ref }}
      repository: ${{ github.repository }}
    secrets: inherit

  cleanup:
    if: github.event_name == 'pull_request' && github.event.action == 'closed'
    uses: koder-cat/k8s-ephemeral-environments/.github/workflows/pr-environment-reusable.yml@main
    with:
      pr-number: ${{ github.event.pull_request.number }}
      pr-action: closed
      head-sha: ${{ github.event.pull_request.head.sha }}
      head-ref: ${{ github.head_ref }}
      repository: ${{ github.repository }}
    secrets: inherit
```

### Comment Command Handling

- Only triggers on PR comments (not issues) — checked via `github.event.issue.pull_request`
- Validates commenter belongs to an allowed organization (reuses `validate-config` allowlist)
- Adds reactions: `:eyes:` while processing, `:rocket:` on success, `:-1:` on failure
- Concurrency group prevents parallel deployments per PR

**Edge cases:**

| Scenario | Behavior |
|----------|----------|
| Multiple `/deploy-preview` in quick succession | Concurrency group queues them |
| `/deploy-preview` on closed PR | Rejected with message |
| `/destroy-preview` with no environment | Posts "No environment found" |
| `/destroy-preview` then `/deploy-preview` | Works — environment recreated |
| Push after `/deploy-preview` | Auto-redeploys (namespace exists check) |
| Push without prior `/deploy-preview` | No-op (namespace doesn't exist) |

### Auto-Redeploy Logic

On `pull_request: synchronize`, the workflow checks if the namespace already exists. If it does (meaning `/deploy-preview` was previously run), it redeploys. If not, it exits cleanly. This avoids accidental environment creation from pushes.

### Changes to Existing Files

| File | Change |
|------|--------|
| `.github/actions/validate-config/schema.json` | Add `trigger` enum field |
| `.github/actions/validate-config/action.yml` | Output `trigger` value |
| `.github/workflows/pr-environment-reusable.yml` | No changes |
| `.github/workflows/preserve-environment.yml` | No changes |

### Documentation Updates

**Repo docs:**

| File | Changes |
|------|---------|
| `docs/guides/k8s-ee-config-reference.md` | Add `trigger` field, on-demand scenario |
| `docs/guides/onboarding-new-repo.md` | Add trigger mode choice, on-demand workflow template |
| `CLAUDE.local.md` | Add `/deploy-preview` and `/destroy-preview` to lifecycle |

**Wiki (`k8s-ephemeral-environments.wiki/`):**

| File | Changes |
|------|---------|
| `Home.md` | Update flowchart for both paths |
| `_Sidebar.md` | Add link to On-Demand Environments page |
| `Quick-Start.md` | Add on-demand workflow variant |
| `Configuration-Reference.md` | Add `trigger` field |
| `System-Overview.md` | Update lifecycle for both paths |
| `Developer-Onboarding.md` | Mention trigger modes |
| `User-Guides.md` | Add link to new page |
| **New: `On-Demand-Environments.md`** | Full feature guide: commands, setup, examples, troubleshooting |

### Default Behavior

`trigger: automatic` is the default. Existing repos without the field continue working without any changes.
