# On-Demand Trigger Mode Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Allow external repos to choose between automatic (every PR) and on-demand (`/deploy-preview` command) environment creation via a `trigger` field in `k8s-ee.yaml`.

**Architecture:** New reusable workflow `deploy-preview-reusable.yml` handles comment commands and auto-redeploy. Existing `pr-environment-reusable.yml` stays untouched. The `validate-config` action gains a `trigger` output. External repos use a different caller workflow that listens on `issue_comment` + `pull_request` events.

**Tech Stack:** GitHub Actions (reusable workflows, `issue_comment` events), JSON Schema (ajv), bash, jq, kubectl

**Design doc:** `docs/plans/2026-03-01-on-demand-trigger-mode-design.md`

---

## Pre-work: Create Feature Branch

**Step 1: Create and push feature branch**

```bash
git checkout -b feat/on-demand-trigger-mode
```

---

### Task 1: Add `trigger` field to JSON Schema

**Files:**
- Modify: `.github/actions/validate-config/schema.json:263` (after `metrics` property)

**Step 1: Add the trigger field to the schema**

Insert the `trigger` property after the `metrics` block (line 263) and before the closing `additionalProperties` (line 264):

```json
    },
    "trigger": {
      "type": "string",
      "description": "Environment trigger mode: 'automatic' creates on every PR, 'on-demand' requires /deploy-preview comment",
      "enum": ["automatic", "on-demand"],
      "default": "automatic"
    }
  },
  "additionalProperties": false
}
```

**Step 2: Validate the schema is valid JSON**

Run: `cat .github/actions/validate-config/schema.json | jq . > /dev/null && echo "Valid JSON"`
Expected: `Valid JSON`

**Step 3: Commit**

```bash
git add .github/actions/validate-config/schema.json
git commit -m "feat(schema): add trigger field for on-demand environments"
```

---

### Task 2: Add `trigger` output to validate-config action

**Files:**
- Modify: `.github/actions/validate-config/action.yml:80` (after metrics-config output)
- Modify: `.github/actions/validate-config/action.yml:439` (after metrics config extraction in parse step)

**Step 1: Add the output declaration**

After `metrics-config` output (line 80), add:

```yaml
  trigger:
    description: 'Environment trigger mode (automatic or on-demand)'
    value: ${{ steps.parse.outputs.trigger }}
```

**Step 2: Extract and output the trigger value in the parse step**

After the metrics config output block (line 439), add:

```bash
        # Extract trigger mode (default: automatic)
        echo "trigger=$(echo "$FINAL_CONFIG" | jq -r '.trigger // "automatic"')" >> "$GITHUB_OUTPUT"
```

Also add trigger to the FINAL_CONFIG jq expression (around line 379) by appending before the closing `'`):

```
          .trigger = (.trigger // "automatic")
```

**Step 3: Add trigger to the Configuration Summary echo block**

After line 450 (`echo "Metrics Enabled: ..."`), add:

```bash
        echo "Trigger Mode: $(echo "$FINAL_CONFIG" | jq -r '.trigger // "automatic"')"
```

**Step 4: Commit**

```bash
git add .github/actions/validate-config/action.yml
git commit -m "feat(validate-config): output trigger mode for downstream jobs"
```

---

### Task 3: Add `trigger` output to reusable workflow

**Files:**
- Modify: `.github/workflows/pr-environment-reusable.yml:122` (after k8s-ee-ref output in validate-config job)

**Step 1: Add trigger to validate-config job outputs**

After line 122 (`k8s-ee-ref: ${{ steps.resolve-k8s-ee-ref.outputs.value }}`), add:

```yaml
      trigger: ${{ steps.validate.outputs.trigger }}
```

**Step 2: Commit**

```bash
git add .github/workflows/pr-environment-reusable.yml
git commit -m "feat(reusable-workflow): expose trigger output from validate-config"
```

---

### Task 4: Create the on-demand reusable workflow

**Files:**
- Create: `.github/workflows/deploy-preview-reusable.yml`

**Step 1: Write the reusable workflow**

This workflow handles three scenarios:
1. **`issue_comment`** — `/deploy-preview` or `/destroy-preview` commands
2. **`pull_request: synchronize`** — auto-redeploy if namespace exists
3. **`pull_request: closed`** — cleanup (delegates to pr-environment-reusable.yml from caller)

The workflow accepts two sets of inputs — one for comment events, one for PR events. Comment events need to extract PR details via the GitHub API.

```yaml
name: Deploy Preview (Reusable)

# Reusable workflow for on-demand PR environments
# Handles /deploy-preview and /destroy-preview comment commands
# Also handles auto-redeploy on push when namespace already exists

on:
  workflow_call:
    inputs:
      # --- Comment event inputs (issue_comment trigger) ---
      comment-body:
        description: 'Comment body text (for issue_comment events)'
        required: false
        type: string
        default: ''
      comment-user:
        description: 'Comment author login (for issue_comment events)'
        required: false
        type: string
        default: ''
      issue-number:
        description: 'Issue/PR number (for issue_comment events)'
        required: false
        type: number
        default: 0
      # --- PR event inputs (pull_request trigger) ---
      pr-number:
        description: 'Pull request number (for pull_request events)'
        required: false
        type: number
        default: 0
      pr-action:
        description: 'PR event action (synchronize)'
        required: false
        type: string
        default: ''
      head-sha:
        description: 'Full commit SHA (for pull_request events)'
        required: false
        type: string
        default: ''
      head-ref:
        description: 'Branch name (for pull_request events)'
        required: false
        type: string
        default: ''
      # --- Shared inputs ---
      repository:
        description: 'GitHub repository (owner/repo)'
        required: true
        type: string
      config-path:
        description: 'Path to k8s-ee.yaml configuration file'
        required: false
        type: string
        default: 'k8s-ee.yaml'
      preview-domain:
        description: 'Base domain for preview URLs'
        required: false
        type: string
        default: 'k8s-ee.genesluna.dev'
      kubectl-version:
        description: 'kubectl version'
        required: false
        type: string
        default: 'v1.31.0'
      helm-version:
        description: 'Helm version'
        required: false
        type: string
        default: 'v3.16.0'
      k8s-api-ip:
        description: 'Kubernetes API server IP for NetworkPolicy egress'
        required: false
        type: string
        default: '10.0.0.39'
      chart-version:
        description: 'k8s-ee-app chart version'
        required: false
        type: string
        default: '1.0.0'
      platforms:
        description: 'Target platform for container image build'
        required: false
        type: string
        default: 'linux/arm64'
      k8s-ee-repo:
        description: 'Repository containing k8s-ee reusable actions and Helm charts'
        required: false
        type: string
        default: 'koder-cat/k8s-ephemeral-environments'
      k8s-ee-ref:
        description: 'Git ref to check out from k8s-ee-repo'
        required: false
        type: string
        default: ''

concurrency:
  group: deploy-preview-${{ inputs.repository }}-${{ inputs.issue-number || inputs.pr-number }}
  cancel-in-progress: false
```

**Step 2: Add the `parse-command` job**

This job determines what action to take — deploy, destroy, auto-redeploy, or skip.

```yaml
jobs:
  # ============================================================================
  # PARSE COMMAND
  # Determines the action: deploy, destroy, redeploy, or skip
  # ============================================================================
  parse-command:
    name: Parse Command
    runs-on: ubuntu-latest
    timeout-minutes: 2

    permissions:
      contents: read
      pull-requests: read

    outputs:
      action: ${{ steps.parse.outputs.action }}
      pr-number: ${{ steps.parse.outputs.pr-number }}
      head-sha: ${{ steps.parse.outputs.head-sha }}
      head-ref: ${{ steps.parse.outputs.head-ref }}

    steps:
      - name: Parse comment or PR event
        id: parse
        uses: actions/github-script@ed597411d8f924073f98dfc5c65a23a2325f34cd # v8.0.0
        with:
          script: |
            const commentBody = '${{ inputs.comment-body }}'.trim();
            const prNumber = ${{ inputs.pr-number }} || ${{ inputs.issue-number }} || 0;
            const prAction = '${{ inputs.pr-action }}';

            // Determine action
            let action = 'skip';

            if (commentBody.startsWith('/deploy-preview')) {
              action = 'deploy';
            } else if (commentBody.startsWith('/destroy-preview')) {
              action = 'destroy';
            } else if (prAction === 'synchronize') {
              action = 'redeploy';
            }

            if (action === 'skip') {
              core.info('No matching command or action, skipping');
              core.setOutput('action', 'skip');
              core.setOutput('pr-number', '0');
              core.setOutput('head-sha', '');
              core.setOutput('head-ref', '');
              return;
            }

            // For comment events, fetch PR details from API
            if (commentBody && prNumber > 0) {
              const { data: pr } = await github.rest.pulls.get({
                owner: context.repo.owner,
                repo: context.repo.repo,
                pull_number: prNumber,
              });

              if (pr.state !== 'open') {
                core.setFailed(`PR #${prNumber} is not open (state: ${pr.state})`);
                return;
              }

              core.setOutput('action', action);
              core.setOutput('pr-number', String(pr.number));
              core.setOutput('head-sha', pr.head.sha);
              core.setOutput('head-ref', pr.head.ref);
            } else {
              // For PR events, use inputs directly
              core.setOutput('action', action);
              core.setOutput('pr-number', String(prNumber));
              core.setOutput('head-sha', '${{ inputs.head-sha }}');
              core.setOutput('head-ref', '${{ inputs.head-ref }}');
            }

            core.info(`Action: ${action}, PR: #${prNumber}`);

      - name: Add eyes reaction to comment
        if: steps.parse.outputs.action != 'skip' && inputs.comment-body != ''
        uses: actions/github-script@ed597411d8f924073f98dfc5c65a23a2325f34cd # v8.0.0
        with:
          script: |
            // We need the comment ID — it's available via the REST API
            // The caller passes comment-body but not comment-id, so we find it
            const { data: comments } = await github.rest.issues.listComments({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: ${{ inputs.issue-number }},
              per_page: 5,
              direction: 'desc',
            });
            const comment = comments.find(c => c.body.trim().startsWith('${{ inputs.comment-body }}'.trim().split('\n')[0]));
            if (comment) {
              await github.rest.reactions.createForIssueComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                comment_id: comment.id,
                content: 'eyes',
              });
            }
```

**Step 3: Add the `validate-config` job**

Reuses the same validate-config pattern from `pr-environment-reusable.yml`.

```yaml
  # ============================================================================
  # VALIDATE CONFIG
  # ============================================================================
  validate-config:
    name: Validate Config
    if: needs.parse-command.outputs.action != 'skip'
    needs: parse-command
    runs-on: ubuntu-latest
    timeout-minutes: 3

    permissions:
      contents: read

    outputs:
      project-id: ${{ steps.validate.outputs.project-id }}
      namespace: ${{ steps.validate.outputs.namespace }}
      preview-url: ${{ steps.validate.outputs.preview-url }}
      config-json: ${{ steps.validate.outputs.config-json }}
      app-port: ${{ steps.validate.outputs.app-port }}
      health-path: ${{ steps.validate.outputs.health-path }}
      image-context: ${{ steps.validate.outputs.image-context }}
      image-dockerfile: ${{ steps.validate.outputs.image-dockerfile }}
      postgresql-enabled: ${{ steps.validate.outputs.postgresql-enabled }}
      mongodb-enabled: ${{ steps.validate.outputs.mongodb-enabled }}
      redis-enabled: ${{ steps.validate.outputs.redis-enabled }}
      minio-enabled: ${{ steps.validate.outputs.minio-enabled }}
      mariadb-enabled: ${{ steps.validate.outputs.mariadb-enabled }}
      postgresql-config: ${{ steps.validate.outputs.postgresql-config }}
      mongodb-config: ${{ steps.validate.outputs.mongodb-config }}
      redis-config: ${{ steps.validate.outputs.redis-config }}
      minio-config: ${{ steps.validate.outputs.minio-config }}
      mariadb-config: ${{ steps.validate.outputs.mariadb-config }}
      metrics-enabled: ${{ steps.validate.outputs.metrics-enabled }}
      metrics-config: ${{ steps.validate.outputs.metrics-config }}
      trigger: ${{ steps.validate.outputs.trigger }}
      k8s-ee-ref: ${{ steps.resolve-k8s-ee-ref.outputs.value }}

    steps:
      - name: Checkout calling repository
        uses: actions/checkout@0c366fd6a839edf440554fa01a7085ccba70ac98 # v4.3.1
        with:
          repository: ${{ inputs.repository }}
          ref: ${{ needs.parse-command.outputs.head-sha }}

      - name: Resolve k8s-ee ref
        id: resolve-k8s-ee-ref
        shell: bash
        env:
          K8S_EE_REF_INPUT: ${{ inputs.k8s-ee-ref }}
          WORKFLOW_REF: ${{ github.workflow_ref }}
        run: |
          set -euo pipefail
          if [[ -n "${K8S_EE_REF_INPUT}" ]]; then
            echo "value=${K8S_EE_REF_INPUT}" >> "$GITHUB_OUTPUT"
          else
            REF="${WORKFLOW_REF##*@}"
            REF="${REF#refs/heads/}"
            REF="${REF#refs/tags/}"
            if [[ "${REF}" == refs/* ]]; then
              echo "::warning::k8s-ee-ref auto-inference got '${REF}'; falling back to 'main'"
              REF="main"
            fi
            echo "value=${REF}" >> "$GITHUB_OUTPUT"
          fi

      - name: Checkout k8s-ee for action
        uses: actions/checkout@0c366fd6a839edf440554fa01a7085ccba70ac98 # v4.3.1
        with:
          repository: ${{ inputs.k8s-ee-repo }}
          ref: ${{ steps.resolve-k8s-ee-ref.outputs.value }}
          path: _k8s-ee
          sparse-checkout: |
            .github/actions
            .github/config
          sparse-checkout-cone-mode: false

      - name: Validate configuration
        id: validate
        uses: ./_k8s-ee/.github/actions/validate-config
        with:
          config-path: ${{ inputs.config-path }}
          pr-number: ${{ needs.parse-command.outputs.pr-number }}
          repository: ${{ inputs.repository }}
          preview-domain: ${{ inputs.preview-domain }}
```

**Step 4: Add the `check-namespace` job**

For `redeploy` action (synchronize events), check if namespace exists before proceeding.

```yaml
  # ============================================================================
  # CHECK NAMESPACE (for auto-redeploy)
  # Only redeploy if the namespace already exists (meaning /deploy-preview was run)
  # ============================================================================
  check-namespace:
    name: Check Namespace Exists
    if: needs.parse-command.outputs.action == 'redeploy'
    needs: [parse-command, validate-config]
    runs-on: arc-runner-set-k8s-ee
    timeout-minutes: 2

    permissions:
      contents: read

    outputs:
      exists: ${{ steps.check.outputs.exists }}

    steps:
      - name: Checkout k8s-ee for actions
        uses: actions/checkout@0c366fd6a839edf440554fa01a7085ccba70ac98 # v4.3.1
        with:
          repository: ${{ inputs.k8s-ee-repo }}
          ref: ${{ needs.validate-config.outputs.k8s-ee-ref }}
          sparse-checkout: .github/actions
          sparse-checkout-cone-mode: false

      - name: Setup tools
        uses: ./.github/actions/setup-tools
        with:
          kubectl-version: ${{ inputs.kubectl-version }}
          install-helm: 'false'

      - name: Check if namespace exists
        id: check
        env:
          NAMESPACE: ${{ needs.validate-config.outputs.namespace }}
        run: |
          set -euo pipefail
          if kubectl get namespace "$NAMESPACE" >/dev/null 2>&1; then
            echo "Namespace $NAMESPACE exists, will redeploy"
            echo "exists=true" >> "$GITHUB_OUTPUT"
          else
            echo "Namespace $NAMESPACE does not exist, skipping redeploy"
            echo "exists=false" >> "$GITHUB_OUTPUT"
          fi
```

**Step 5: Add the deploy jobs**

These mirror the `create-namespace`, `build-image`, `deploy-app`, and `pr-comment-deploy` jobs from `pr-environment-reusable.yml`, but with conditions based on the parsed action.

The deploy condition is: `(action == 'deploy') || (action == 'redeploy' && namespace exists)`

```yaml
  # ============================================================================
  # CREATE NAMESPACE
  # ============================================================================
  create-namespace:
    name: Create Namespace
    if: |
      always() &&
      needs.validate-config.result == 'success' &&
      (needs.parse-command.outputs.action == 'deploy' ||
       (needs.parse-command.outputs.action == 'redeploy' && needs.check-namespace.outputs.exists == 'true'))
    needs: [parse-command, validate-config, check-namespace]
    runs-on: arc-runner-set-k8s-ee
    timeout-minutes: 5

    permissions:
      contents: read

    outputs:
      created: ${{ steps.create.outputs.created }}
      namespace: ${{ steps.create.outputs.namespace }}

    steps:
      - name: Checkout k8s-ee for actions
        uses: actions/checkout@0c366fd6a839edf440554fa01a7085ccba70ac98 # v4.3.1
        with:
          repository: ${{ inputs.k8s-ee-repo }}
          ref: ${{ needs.validate-config.outputs.k8s-ee-ref }}
          sparse-checkout: .github/actions
          sparse-checkout-cone-mode: false

      - name: Setup tools
        uses: ./.github/actions/setup-tools
        with:
          kubectl-version: ${{ inputs.kubectl-version }}
          install-helm: 'false'

      - name: Create namespace
        id: create
        uses: ./.github/actions/create-namespace
        with:
          namespace: ${{ needs.validate-config.outputs.namespace }}
          project-id: ${{ needs.validate-config.outputs.project-id }}
          pr-number: ${{ needs.parse-command.outputs.pr-number }}
          branch-name: ${{ needs.parse-command.outputs.head-ref }}
          commit-sha: ${{ needs.parse-command.outputs.head-sha }}
          repository: ${{ inputs.repository }}
          k8s-api-ip: ${{ inputs.k8s-api-ip }}
          postgresql-enabled: ${{ needs.validate-config.outputs.postgresql-enabled }}
          mongodb-enabled: ${{ needs.validate-config.outputs.mongodb-enabled }}
          redis-enabled: ${{ needs.validate-config.outputs.redis-enabled }}
          minio-enabled: ${{ needs.validate-config.outputs.minio-enabled }}
          mariadb-enabled: ${{ needs.validate-config.outputs.mariadb-enabled }}
          app-port: ${{ needs.validate-config.outputs.app-port }}

  # ============================================================================
  # BUILD IMAGE
  # ============================================================================
  build-image:
    name: Build Image
    if: |
      always() &&
      needs.validate-config.result == 'success' &&
      (needs.parse-command.outputs.action == 'deploy' ||
       (needs.parse-command.outputs.action == 'redeploy' && needs.check-namespace.outputs.exists == 'true'))
    needs: [parse-command, validate-config, check-namespace]
    runs-on: ubuntu-latest
    timeout-minutes: 15

    permissions:
      contents: read
      packages: write
      security-events: write

    outputs:
      image-tag: ${{ steps.build.outputs.image-tag }}
      image-digest: ${{ steps.build.outputs.image-digest }}
      image-ref: ${{ steps.build.outputs.image-ref }}

    steps:
      - name: Checkout calling repository
        uses: actions/checkout@0c366fd6a839edf440554fa01a7085ccba70ac98 # v4.3.1
        with:
          repository: ${{ inputs.repository }}
          ref: ${{ needs.parse-command.outputs.head-sha }}

      - name: Checkout k8s-ee for action
        uses: actions/checkout@0c366fd6a839edf440554fa01a7085ccba70ac98 # v4.3.1
        with:
          repository: ${{ inputs.k8s-ee-repo }}
          ref: ${{ needs.validate-config.outputs.k8s-ee-ref }}
          path: _k8s-ee
          sparse-checkout: .github/actions
          sparse-checkout-cone-mode: false

      - name: Build and push image
        id: build
        uses: ./_k8s-ee/.github/actions/build-image
        with:
          context: ${{ needs.validate-config.outputs.image-context }}
          dockerfile: ${{ needs.validate-config.outputs.image-dockerfile }}
          image-name: ${{ inputs.repository }}/${{ needs.validate-config.outputs.project-id }}
          pr-number: ${{ needs.parse-command.outputs.pr-number }}
          commit-sha: ${{ needs.parse-command.outputs.head-sha }}
          github-token: ${{ secrets.GITHUB_TOKEN }}
          platforms: ${{ inputs.platforms }}

  # ============================================================================
  # DEPLOY APP
  # ============================================================================
  deploy-app:
    name: Deploy App
    if: |
      always() &&
      needs.validate-config.result == 'success' &&
      needs.create-namespace.result == 'success' &&
      needs.build-image.result == 'success'
    needs: [parse-command, validate-config, create-namespace, build-image]
    runs-on: arc-runner-set-k8s-ee
    timeout-minutes: 10

    permissions:
      contents: read
      packages: read

    outputs:
      preview-url: ${{ steps.deploy.outputs.preview-url }}
      release-status: ${{ steps.deploy.outputs.release-status }}

    steps:
      - name: Checkout k8s-ee for actions and charts
        uses: actions/checkout@0c366fd6a839edf440554fa01a7085ccba70ac98 # v4.3.1
        with:
          repository: ${{ inputs.k8s-ee-repo }}
          ref: ${{ needs.validate-config.outputs.k8s-ee-ref }}
          sparse-checkout: |
            .github/actions/*
            charts/k8s-ee-app/*
            charts/postgresql/*
            charts/mongodb/*
            charts/redis/*
            charts/minio/*
            charts/mariadb/*
          sparse-checkout-cone-mode: false

      - name: Setup tools
        uses: ./.github/actions/setup-tools
        with:
          kubectl-version: ${{ inputs.kubectl-version }}
          helm-version: ${{ inputs.helm-version }}

      - name: Deploy application
        id: deploy
        uses: ./.github/actions/deploy-app
        with:
          namespace: ${{ needs.validate-config.outputs.namespace }}
          project-id: ${{ needs.validate-config.outputs.project-id }}
          pr-number: ${{ needs.parse-command.outputs.pr-number }}
          image-repository: ghcr.io/${{ inputs.repository }}/${{ needs.validate-config.outputs.project-id }}
          image-tag: ${{ needs.build-image.outputs.image-tag }}
          image-digest: ${{ needs.build-image.outputs.image-digest }}
          commit-sha: ${{ needs.parse-command.outputs.head-sha }}
          branch-name: ${{ needs.parse-command.outputs.head-ref }}
          preview-domain: ${{ inputs.preview-domain }}
          chart-version: ${{ inputs.chart-version }}
          use-local-charts: 'true'
          app-port: ${{ needs.validate-config.outputs.app-port }}
          health-path: ${{ needs.validate-config.outputs.health-path }}
          postgresql-enabled: ${{ needs.validate-config.outputs.postgresql-enabled }}
          mongodb-enabled: ${{ needs.validate-config.outputs.mongodb-enabled }}
          redis-enabled: ${{ needs.validate-config.outputs.redis-enabled }}
          minio-enabled: ${{ needs.validate-config.outputs.minio-enabled }}
          mariadb-enabled: ${{ needs.validate-config.outputs.mariadb-enabled }}
          postgresql-config: ${{ needs.validate-config.outputs.postgresql-config }}
          mongodb-config: ${{ needs.validate-config.outputs.mongodb-config }}
          redis-config: ${{ needs.validate-config.outputs.redis-config }}
          minio-config: ${{ needs.validate-config.outputs.minio-config }}
          mariadb-config: ${{ needs.validate-config.outputs.mariadb-config }}
          metrics-enabled: ${{ needs.validate-config.outputs.metrics-enabled }}
          metrics-config: ${{ needs.validate-config.outputs.metrics-config }}
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

**Step 6: Add the `destroy-namespace` job (for `/destroy-preview` command)**

```yaml
  # ============================================================================
  # DESTROY NAMESPACE (for /destroy-preview command)
  # ============================================================================
  destroy-namespace:
    name: Destroy Namespace
    if: needs.parse-command.outputs.action == 'destroy' && needs.validate-config.result == 'success'
    needs: [parse-command, validate-config]
    runs-on: arc-runner-set-k8s-ee
    timeout-minutes: 5

    permissions:
      contents: read

    outputs:
      deleted: ${{ steps.destroy.outputs.deleted }}

    steps:
      - name: Checkout k8s-ee for actions
        uses: actions/checkout@0c366fd6a839edf440554fa01a7085ccba70ac98 # v4.3.1
        with:
          repository: ${{ inputs.k8s-ee-repo }}
          ref: ${{ needs.validate-config.outputs.k8s-ee-ref }}
          sparse-checkout: .github/actions
          sparse-checkout-cone-mode: false

      - name: Setup tools
        uses: ./.github/actions/setup-tools
        with:
          kubectl-version: ${{ inputs.kubectl-version }}
          install-helm: 'false'

      - name: Destroy namespace
        id: destroy
        uses: ./.github/actions/destroy-namespace
        with:
          namespace: ${{ needs.validate-config.outputs.namespace }}
          repository: ${{ inputs.repository }}
```

**Step 7: Add the PR comment jobs**

```yaml
  # ============================================================================
  # PR COMMENT (DEPLOY)
  # ============================================================================
  pr-comment-deploy:
    name: PR Comment (Deploy)
    if: |
      always() &&
      needs.validate-config.result == 'success' &&
      (needs.parse-command.outputs.action == 'deploy' ||
       (needs.parse-command.outputs.action == 'redeploy' && needs.check-namespace.outputs.exists == 'true'))
    needs: [parse-command, validate-config, check-namespace, build-image, deploy-app]
    runs-on: ubuntu-latest
    timeout-minutes: 2

    permissions:
      pull-requests: write

    steps:
      - name: Checkout k8s-ee for action
        uses: actions/checkout@0c366fd6a839edf440554fa01a7085ccba70ac98 # v4.3.1
        with:
          repository: ${{ inputs.k8s-ee-repo }}
          ref: ${{ needs.validate-config.outputs.k8s-ee-ref }}
          sparse-checkout: .github/actions
          sparse-checkout-cone-mode: false

      - name: Comment on PR (success)
        if: needs.deploy-app.result == 'success'
        uses: ./.github/actions/pr-comment
        with:
          status: deployed
          preview-url: ${{ needs.deploy-app.outputs.preview-url }}
          namespace: ${{ needs.validate-config.outputs.namespace }}
          commit-sha: ${{ needs.parse-command.outputs.head-sha }}
          branch-name: ${{ needs.parse-command.outputs.head-ref }}
          github-token: ${{ secrets.GITHUB_TOKEN }}
          pr-number: ${{ needs.parse-command.outputs.pr-number }}

      - name: Comment on PR (failure)
        if: needs.deploy-app.result != 'success'
        uses: ./.github/actions/pr-comment
        with:
          status: failed
          namespace: ${{ needs.validate-config.outputs.namespace }}
          commit-sha: ${{ needs.parse-command.outputs.head-sha }}
          branch-name: ${{ needs.parse-command.outputs.head-ref }}
          github-token: ${{ secrets.GITHUB_TOKEN }}
          pr-number: ${{ needs.parse-command.outputs.pr-number }}

      - name: Add rocket reaction to comment
        if: needs.deploy-app.result == 'success' && inputs.comment-body != ''
        continue-on-error: true
        uses: actions/github-script@ed597411d8f924073f98dfc5c65a23a2325f34cd # v8.0.0
        with:
          script: |
            const { data: comments } = await github.rest.issues.listComments({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: ${{ inputs.issue-number }},
              per_page: 5,
              direction: 'desc',
            });
            const commentBody = '${{ inputs.comment-body }}'.trim().split('\n')[0];
            const comment = comments.find(c => c.body.trim().startsWith(commentBody));
            if (comment) {
              await github.rest.reactions.createForIssueComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                comment_id: comment.id,
                content: 'rocket',
              });
            }

  # ============================================================================
  # PR COMMENT (DESTROY)
  # ============================================================================
  pr-comment-destroy:
    name: PR Comment (Destroy)
    if: always() && needs.parse-command.outputs.action == 'destroy' && needs.validate-config.result == 'success'
    needs: [parse-command, validate-config, destroy-namespace]
    runs-on: ubuntu-latest
    timeout-minutes: 2

    permissions:
      pull-requests: write

    steps:
      - name: Checkout k8s-ee for action
        uses: actions/checkout@0c366fd6a839edf440554fa01a7085ccba70ac98 # v4.3.1
        with:
          repository: ${{ inputs.k8s-ee-repo }}
          ref: ${{ needs.validate-config.outputs.k8s-ee-ref }}
          sparse-checkout: .github/actions
          sparse-checkout-cone-mode: false

      - name: Comment on PR (destroyed)
        if: needs.destroy-namespace.result == 'success'
        uses: ./.github/actions/pr-comment
        with:
          status: destroyed
          namespace: ${{ needs.validate-config.outputs.namespace }}
          commit-sha: ${{ needs.parse-command.outputs.head-sha }}
          branch-name: ${{ needs.parse-command.outputs.head-ref }}
          github-token: ${{ secrets.GITHUB_TOKEN }}
          pr-number: ${{ needs.parse-command.outputs.pr-number }}

      - name: Comment on PR (destroy failed)
        if: needs.destroy-namespace.result != 'success'
        uses: ./.github/actions/pr-comment
        with:
          status: failed
          namespace: ${{ needs.validate-config.outputs.namespace }}
          commit-sha: ${{ needs.parse-command.outputs.head-sha }}
          branch-name: ${{ needs.parse-command.outputs.head-ref }}
          github-token: ${{ secrets.GITHUB_TOKEN }}
          pr-number: ${{ needs.parse-command.outputs.pr-number }}

      - name: Add reaction to comment
        if: inputs.comment-body != ''
        continue-on-error: true
        uses: actions/github-script@ed597411d8f924073f98dfc5c65a23a2325f34cd # v8.0.0
        with:
          script: |
            const success = '${{ needs.destroy-namespace.result }}' === 'success';
            const { data: comments } = await github.rest.issues.listComments({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: ${{ inputs.issue-number }},
              per_page: 5,
              direction: 'desc',
            });
            const commentBody = '${{ inputs.comment-body }}'.trim().split('\n')[0];
            const comment = comments.find(c => c.body.trim().startsWith(commentBody));
            if (comment) {
              await github.rest.reactions.createForIssueComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                comment_id: comment.id,
                content: success ? '+1' : '-1',
              });
            }
```

**Step 8: Commit**

```bash
git add .github/workflows/deploy-preview-reusable.yml
git commit -m "feat(workflow): add on-demand deploy-preview reusable workflow"
```

---

### Task 5: Create the dogfooding workflow

**Files:**
- Create: `.github/workflows/deploy-preview.yml`

**Step 1: Write the dogfooding workflow**

```yaml
# Deploy Preview Workflow - Dogfooding Edition
#
# This workflow uses the on-demand reusable workflow from this same repository,
# demonstrating how external repositories should integrate with on-demand mode.
# Requires trigger: on-demand in k8s-ee.yaml.
#
# For external repositories, change the workflow reference to:
#   uses: koder-cat/k8s-ephemeral-environments/.github/workflows/deploy-preview-reusable.yml@main

name: Deploy Preview

on:
  issue_comment:
    types: [created]
  pull_request:
    types: [synchronize, closed]

concurrency:
  group: deploy-preview-${{ github.event.pull_request.number || github.event.issue.number }}
  cancel-in-progress: false

permissions:
  contents: read
  packages: write
  pull-requests: write
  security-events: write

jobs:
  deploy-preview:
    # Only handle /deploy-preview and /destroy-preview comments on PRs
    if: |
      github.event_name == 'issue_comment' &&
      github.event.issue.pull_request &&
      (startsWith(github.event.comment.body, '/deploy-preview') ||
       startsWith(github.event.comment.body, '/destroy-preview'))
    uses: ./.github/workflows/deploy-preview-reusable.yml
    with:
      comment-body: ${{ github.event.comment.body }}
      comment-user: ${{ github.event.comment.user.login }}
      issue-number: ${{ github.event.issue.number }}
      repository: ${{ github.repository }}
      k8s-ee-ref: ${{ github.base_ref || 'main' }}
    secrets: inherit

  auto-redeploy:
    if: github.event_name == 'pull_request' && github.event.action == 'synchronize'
    uses: ./.github/workflows/deploy-preview-reusable.yml
    with:
      pr-number: ${{ github.event.pull_request.number }}
      pr-action: synchronize
      head-sha: ${{ github.event.pull_request.head.sha }}
      head-ref: ${{ github.head_ref }}
      repository: ${{ github.repository }}
      k8s-ee-ref: ${{ github.base_ref || 'main' }}
    secrets: inherit

  cleanup:
    if: github.event_name == 'pull_request' && github.event.action == 'closed'
    uses: ./.github/workflows/pr-environment-reusable.yml
    with:
      pr-number: ${{ github.event.pull_request.number }}
      pr-action: closed
      head-sha: ${{ github.event.pull_request.head.sha }}
      head-ref: ${{ github.head_ref }}
      repository: ${{ github.repository }}
      k8s-ee-ref: ${{ github.base_ref || 'main' }}
    secrets: inherit
```

**Step 2: Commit**

```bash
git add .github/workflows/deploy-preview.yml
git commit -m "feat(workflow): add deploy-preview dogfooding workflow"
```

---

### Task 6: Update repo documentation — k8s-ee-config-reference.md

**Files:**
- Modify: `docs/guides/k8s-ee-config-reference.md`

**Step 1: Add `trigger` to the full config example**

After `projectId: myapp` (line 23), add:

```yaml
trigger: automatic    # or "on-demand" for /deploy-preview command
```

**Step 2: Add `trigger` field reference section**

After the `projectId` field reference section (after line ~98), add a new section:

```markdown
### trigger

Controls how PR environments are created.

| Property | Value |
|----------|-------|
| Type | string |
| Required | No |
| Default | `automatic` |
| Values | `automatic`, `on-demand` |

**`automatic`** (default): Environment is created automatically when a PR is opened or updated. This is the standard behavior.

**`on-demand`**: Environment is only created when someone comments `/deploy-preview` on the PR. After creation, subsequent pushes auto-redeploy. Use `/destroy-preview` to tear down the environment early.

```yaml
trigger: on-demand
```

> **Note:** On-demand mode requires a different workflow file. See [On-Demand Environments](../../wiki/On-Demand-Environments) for setup instructions.
```

**Step 3: Add an on-demand scenario to Common Scenarios section**

Add a new scenario:

```markdown
### On-Demand Environments

Save resources by creating environments only when needed:

\`\`\`yaml
projectId: myapp
trigger: on-demand

app:
  port: 3000
  healthPath: /health

databases:
  postgresql: true
\`\`\`

With this configuration, environments are only created when someone comments `/deploy-preview` on the PR. Use `/destroy-preview` to tear down the environment early. See [Onboarding Guide](./onboarding-new-repo.md) for the required workflow file.
```

**Step 4: Commit**

```bash
git add docs/guides/k8s-ee-config-reference.md
git commit -m "docs(config-reference): add trigger field documentation"
```

---

### Task 7: Update repo documentation — onboarding-new-repo.md

**Files:**
- Modify: `docs/guides/onboarding-new-repo.md`

**Step 1: Add trigger mode to Step 1 config example**

After the existing config example in Step 1 (line ~97), add a note:

```markdown
> **On-demand mode:** To create environments only when needed (via `/deploy-preview` comment), add `trigger: on-demand` to your configuration. See Step 2 for the corresponding workflow file.
```

**Step 2: Add on-demand workflow template to Step 2**

After the existing workflow template (line ~132), add:

```markdown
#### On-Demand Workflow (Optional)

If you set `trigger: on-demand` in your `k8s-ee.yaml`, use this workflow file instead:

\`\`\`yaml
name: PR Environment (On-Demand)

on:
  issue_comment:
    types: [created]
  pull_request:
    types: [synchronize, closed]

concurrency:
  group: deploy-preview-${{ github.event.pull_request.number || github.event.issue.number }}
  cancel-in-progress: false

permissions:
  contents: read
  packages: write
  pull-requests: write
  security-events: write

jobs:
  deploy-preview:
    if: |
      github.event_name == 'issue_comment' &&
      github.event.issue.pull_request &&
      (startsWith(github.event.comment.body, '/deploy-preview') ||
       startsWith(github.event.comment.body, '/destroy-preview'))
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
\`\`\`

**Available commands:**

| Command | Action |
|---------|--------|
| `/deploy-preview` | Creates or redeploys the PR environment |
| `/destroy-preview` | Destroys the environment (PR stays open) |

After the first `/deploy-preview`, subsequent pushes to the PR automatically redeploy.
```

**Step 3: Commit**

```bash
git add docs/guides/onboarding-new-repo.md
git commit -m "docs(onboarding): add on-demand trigger mode workflow template"
```

---

### Task 8: Update wiki — Configuration-Reference.md

**Files:**
- Modify: `/home/genesluna/repos/k8s-ephemeral-environments.wiki/Configuration-Reference.md`

**Step 1: Add `trigger` field documentation**

Mirror the changes from Task 6 — add the `trigger` field to the full example and add a field reference section. Check the wiki file structure first and add in the corresponding locations.

**Step 2: Commit (in wiki repo)**

```bash
cd /home/genesluna/repos/k8s-ephemeral-environments.wiki
git add Configuration-Reference.md
git commit -m "docs: add trigger field to configuration reference"
```

---

### Task 9: Create wiki page — On-Demand-Environments.md

**Files:**
- Create: `/home/genesluna/repos/k8s-ephemeral-environments.wiki/On-Demand-Environments.md`

**Step 1: Write the full guide page**

Content should cover:
- Overview: what on-demand mode is and why you'd use it
- Setup: k8s-ee.yaml config + workflow file
- Commands: `/deploy-preview` and `/destroy-preview` with examples
- Auto-redeploy behavior
- How it interacts with `/preserve`
- Troubleshooting common issues

**Step 2: Commit**

```bash
cd /home/genesluna/repos/k8s-ephemeral-environments.wiki
git add On-Demand-Environments.md
git commit -m "docs: add on-demand environments guide"
```

---

### Task 10: Update wiki — _Sidebar.md, Home.md, Quick-Start.md

**Files:**
- Modify: `/home/genesluna/repos/k8s-ephemeral-environments.wiki/_Sidebar.md`
- Modify: `/home/genesluna/repos/k8s-ephemeral-environments.wiki/Home.md`
- Modify: `/home/genesluna/repos/k8s-ephemeral-environments.wiki/Quick-Start.md`

**Step 1: Add On-Demand Environments to sidebar**

Add `[[On-Demand Environments]]` link under the User Guides section.

**Step 2: Update Home.md**

Update the "How It Works" section to mention both automatic and on-demand trigger modes.

**Step 3: Update Quick-Start.md**

In Step 2 (workflow file), add a note about the on-demand alternative with a link to the On-Demand Environments page.

**Step 4: Commit**

```bash
cd /home/genesluna/repos/k8s-ephemeral-environments.wiki
git add _Sidebar.md Home.md Quick-Start.md
git commit -m "docs: add on-demand environments to sidebar, home, and quick start"
```

---

### Task 11: Update wiki — System-Overview.md, Developer-Onboarding.md, User-Guides.md

**Files:**
- Modify: `/home/genesluna/repos/k8s-ephemeral-environments.wiki/System-Overview.md`
- Modify: `/home/genesluna/repos/k8s-ephemeral-environments.wiki/Developer-Onboarding.md`
- Modify: `/home/genesluna/repos/k8s-ephemeral-environments.wiki/User-Guides.md`

**Step 1: Update System-Overview.md**

Add on-demand trigger mode to the PR Environment Lifecycle section, showing the alternative path.

**Step 2: Update Developer-Onboarding.md**

Mention trigger modes in the Development Workflow and Quick Reference sections.

**Step 3: Update User-Guides.md**

Add a link to the On-Demand Environments page.

**Step 4: Commit**

```bash
cd /home/genesluna/repos/k8s-ephemeral-environments.wiki
git add System-Overview.md Developer-Onboarding.md User-Guides.md
git commit -m "docs: add on-demand trigger mode to system overview, onboarding, and user guides"
```

---

### Task 12: Push branch and create PR

**Step 1: Push the feature branch**

```bash
cd /home/genesluna/repos/k8s-ephemeral-environments
git push -u origin feat/on-demand-trigger-mode
```

**Step 2: Push wiki changes**

```bash
cd /home/genesluna/repos/k8s-ephemeral-environments.wiki
git push
```

**Step 3: Create PR**

```bash
cd /home/genesluna/repos/k8s-ephemeral-environments
gh pr create --title "feat: add on-demand trigger mode for PR environments" --body "$(cat <<'EOF'
## Summary

- Adds `trigger: on-demand` option to `k8s-ee.yaml` for repos that want environments created only via `/deploy-preview` comment
- New reusable workflow `deploy-preview-reusable.yml` handles comment commands and auto-redeploy
- `/destroy-preview` command allows early teardown
- Existing automatic mode is unchanged and remains the default

## Changes

- `.github/actions/validate-config/schema.json` — added `trigger` enum field
- `.github/actions/validate-config/action.yml` — outputs trigger mode
- `.github/workflows/pr-environment-reusable.yml` — exposes trigger output
- `.github/workflows/deploy-preview-reusable.yml` — new on-demand reusable workflow
- `.github/workflows/deploy-preview.yml` — dogfooding workflow
- `docs/guides/k8s-ee-config-reference.md` — trigger field docs
- `docs/guides/onboarding-new-repo.md` — on-demand workflow template

## Test plan

- [ ] Verify automatic mode still works (no regression)
- [ ] Test `/deploy-preview` creates environment
- [ ] Test `/destroy-preview` destroys environment
- [ ] Test push after `/deploy-preview` auto-redeploys
- [ ] Test push without prior `/deploy-preview` is a no-op
- [ ] Test PR close destroys environment
- [ ] Verify schema validation accepts `trigger: automatic` and `trigger: on-demand`
- [ ] Verify schema validation rejects invalid trigger values
EOF
)"
```
