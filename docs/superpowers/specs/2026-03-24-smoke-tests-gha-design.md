# Smoke Tests GHA Improvements — Design Spec

**Date:** 2026-03-24
**Branch:** smoke
**Status:** Approved

---

## Problem Statement

The open-source smoke test GitHub Actions workflow (`system-tests-opensource.yml`) has four issues:

1. **Missing Unknown label on commit push** — When a commit is pushed to a PR, the `Smoke tests: Unknown` label is not set. The `label-oss-system-test-unknown` job in `ci.yaml` only runs on `pull_request` events and cannot set labels on `push` events due to GitHub's permission model (push-triggered workflows do not receive `pull-requests: write`).

2. **Marker filter can be gamed** — `pytest_markers` is a free-text `workflow_dispatch` input. A user can run the workflow with custom markers and still receive the official `Smoke tests: Pass` label, misrepresenting which tests were actually run.

3. **No PR comment with run link** — When smoke tests complete, there is no comment on the PR pointing to the workflow run, making it hard to navigate to results.

4. **No distinction between upstream and PR code** — There is no way for a user to explicitly declare they are testing from PR code vs upstream, and the resulting label does not reflect this distinction. Additionally, the test runner always checks out the workflow's default branch (`smoke`), meaning modified smoke tests inside a PR are never exercised — only the Docker images are built from PR code.

---

## Solution Overview

- **New `smoke-tests.yml`** — Official, locked entry point for smoke tests. Only this workflow sets `Smoke tests: Pass/Fail` labels and posts PR comments. `pytest_markers` is not an input; it is hardcoded.
- **New `set-smoke-label-unknown.yml`** — Sets `Smoke tests: Unknown` via a `workflow_run` trigger (on CI `requested`), which grants `pull-requests: write` regardless of push event restrictions.
- **Modify `system-tests-opensource.yml`** — Add `workflow_call` trigger; remove all label and comment logic; add `build_from_pr` input; expose `test_outcome` as a workflow output; check out PR code when `build_from_pr=true`.
- **Modify `ci.yaml`** — Remove the `label-oss-system-test-unknown` job (replaced by `set-smoke-label-unknown.yml`).
- **Fix `build-internal.yaml`** — Replace `github.event.inputs.pr_number` with `inputs.pr_number` in checkout conditions so the PR branch checkout is controlled by the explicit input, not by event context leakage.

---

## Files Changed

| File | Action | Summary |
|------|--------|---------|
| `.github/workflows/system-tests-opensource.yml` | Modify | Add `workflow_call` trigger; remove labeling/comments; add `build_from_pr` input; expose `test_outcome` output; checkout PR code when `build_from_pr=true` |
| `.github/workflows/smoke-tests.yml` | **New** | Official smoke entry point with locked markers, labeling, PR comment |
| `.github/workflows/set-smoke-label-unknown.yml` | **New** | Sets Unknown label via `workflow_run: [CI]: [requested]` |
| `.github/workflows/ci.yaml` | Modify | Remove `label-oss-system-test-unknown` job |
| `.github/workflows/build-internal.yaml` | Fix | Use `inputs.pr_number` instead of `github.event.inputs.pr_number` in checkout conditions |

---

## Trigger Map

```
1. Developer pushes commit to PR branch
   └─> CI workflow starts (event: pull_request or push)
         └─> set-smoke-label-unknown.yml [workflow_run: CI, requested]
               if: github.event.workflow_run.event == 'pull_request'
               └─> sets "Smoke tests: Unknown" on PR

2. User triggers smoke-tests.yml (workflow_dispatch)
   inputs: pr_number (optional), build_from_pr (bool), clean_resources_in_teardown, debug_enabled
   └─> job: run-smoke-tests
         calls system-tests-opensource.yml [workflow_call]
         with: pytest_markers="not enterprise and smoke" (hardcoded)
               pr_number, build_from_pr, clean_resources_in_teardown, debug_enabled
         output: test_outcome
   └─> job: post-results (if: always(), needs pr_number)
         sets Pass/Fail label (with "tests from pr" suffix if build_from_pr=true)
         removes opposite label + "Smoke tests: Unknown"
         posts PR comment with outcome + link to run

3. User triggers system-tests-opensource.yml directly (debugging)
   inputs: pr_number (optional), pytest_markers (free text), build_from_pr, ...
   └─> runs tests only — no labels, no PR comments
```

---

## Detailed Design

### `system-tests-opensource.yml` changes

**Add `workflow_call` trigger:**

```yaml
on:
  workflow_dispatch:
    inputs: { ... }  # unchanged
  workflow_call:
    inputs:
      pr_number:
        type: string
        required: false
        default: ''
      pytest_markers:
        type: string
        required: false
        default: 'not enterprise and smoke'
      clean_resources_in_teardown:
        type: string
        required: false
        default: 'true'
      debug_enabled:
        type: string
        required: false
        default: 'false'
      build_from_pr:
        type: boolean
        required: false
        default: false
    outputs:
      test_outcome:
        description: 'Outcome of the system tests job (success|failure)'
        value: ${{ jobs.run-system-tests-opensource-ci.outputs.test_outcome }}
```

**Expose test outcome from job:**

```yaml
jobs:
  run-system-tests-opensource-ci:
    outputs:
      test_outcome: ${{ steps.system-tests.outcome }}
```

**`build_from_pr` semantics:**

| | `build_from_pr=false` (default) | `build_from_pr=true` |
|---|---|---|
| Docker images | Skip build job; use upstream pre-built images | Build from `refs/pull/{PR_NUM}/merge` |
| Test code checkout | Workflow branch (`smoke`) | `refs/pull/{PR_NUM}/merge` |
| `mlrun_version_specifier` | Upstream commit hash | `refs/pull/{PR_NUM}/merge` |

The `build-mlrun` job condition changes from:
```yaml
if: github.event_name == 'pull_request' || github.event.inputs.pr_number != ''
```
to:
```yaml
if: inputs.build_from_pr == true
```

The `run-system-tests-opensource-ci` job adds a conditional checkout:
```yaml
- uses: actions/checkout@v6
  if: inputs.build_from_pr != true

- uses: actions/checkout@v6
  if: inputs.build_from_pr == true
  with:
    ref: refs/pull/${{ inputs.pr_number }}/merge
    fetch-depth: 0
```

The `prepare-inputs` job's `mlrun_version_specifier` logic:
- `build_from_pr=true` + `pr_number` set → `refs/pull/{PR_NUM}/merge`
- `build_from_pr=false` → upstream commit hash (no PR ref)

**Remove entirely:** The `Label PR with system test result` step.

**Pass `pr_number` to `build-internal.yaml`:** Add `pr_number: ${{ inputs.pr_number }}` to the `with:` block of the `build-mlrun` job call.

---

### `smoke-tests.yml` (new)

```yaml
name: Smoke Tests

permissions:
  contents: read
  pull-requests: write
  packages: write

on:
  workflow_dispatch:
    inputs:
      pr_number:
        description: 'PR number to target for labeling (and optionally building from)'
        required: false
        default: ''
      build_from_pr:
        description: 'Build images and run tests from the PR branch (vs upstream)'
        required: false
        default: false
        type: boolean
      clean_resources_in_teardown:
        description: 'Clean test resources upon teardown'
        required: true
        default: 'true'
        type: choice
        options: ['true', 'false']
      debug_enabled:
        description: 'Allow SSH debugging'
        required: false
        default: 'false'
        type: choice
        options: ['true', 'false']

jobs:
  run-smoke-tests:
    uses: ./.github/workflows/system-tests-opensource.yml
    with:
      pr_number: ${{ inputs.pr_number }}
      pytest_markers: 'not enterprise and smoke'   # locked — not exposed as input
      build_from_pr: ${{ inputs.build_from_pr }}
      clean_resources_in_teardown: ${{ inputs.clean_resources_in_teardown }}
      debug_enabled: ${{ inputs.debug_enabled }}
    secrets: inherit

  post-results:
    name: Post smoke test results
    runs-on: ubuntu-latest
    if: always() && inputs.pr_number != ''
    needs: run-smoke-tests
    permissions:
      pull-requests: write
    steps:
      - name: Determine label and outcome
        id: outcome
        run: |
          OUTCOME="${{ needs.run-smoke-tests.outputs.test_outcome }}"
          BUILD_FROM_PR="${{ inputs.build_from_pr }}"

          if [ "$OUTCOME" = "success" ]; then
            STATUS="Pass"
            EMOJI="✅"
            COLOR="0e8a16"
            REMOVE_STATUS="Fail"
          else
            STATUS="Fail"
            EMOJI="❌"
            COLOR="d93f0b"
            REMOVE_STATUS="Pass"
          fi

          if [ "$BUILD_FROM_PR" = "true" ]; then
            LABEL="Smoke tests: ${STATUS} (tests from pr)"
            REMOVE_LABEL="Smoke tests: ${REMOVE_STATUS} (tests from pr)"
            SOURCE="PR branch"
          else
            LABEL="Smoke tests: ${STATUS}"
            REMOVE_LABEL="Smoke tests: ${REMOVE_STATUS}"
            SOURCE="upstream"
          fi

          echo "label=$LABEL"            >> $GITHUB_OUTPUT
          echo "remove_label=$REMOVE_LABEL" >> $GITHUB_OUTPUT
          echo "color=$COLOR"            >> $GITHUB_OUTPUT
          echo "emoji=$EMOJI"            >> $GITHUB_OUTPUT
          echo "status=$STATUS"          >> $GITHUB_OUTPUT
          echo "source=$SOURCE"          >> $GITHUB_OUTPUT

      - name: Set label on PR
        run: |
          PR="${{ inputs.pr_number }}"
          API_URL="https://api.github.com/repos/${{ github.repository }}/issues/${PR}/labels"
          REPO_LABELS_URL="https://api.github.com/repos/${{ github.repository }}/labels"
          LABEL="${{ steps.outcome.outputs.label }}"
          COLOR="${{ steps.outcome.outputs.color }}"
          REMOVE_LABEL="${{ steps.outcome.outputs.remove_label }}"

          ENCODED_LABEL=$(echo "${LABEL}" | sed 's/ /%20/g; s/:/%3A/g; s/(/%28/g; s/)/%29/g')
          ENCODED_REMOVE=$(echo "${REMOVE_LABEL}" | sed 's/ /%20/g; s/:/%3A/g; s/(/%28/g; s/)/%29/g')

          # Ensure label exists
          curl -sS -X PATCH \
            -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            -H "Accept: application/vnd.github+json" \
            -d "{\"color\":\"${COLOR}\"}" \
            "${REPO_LABELS_URL}/${ENCODED_LABEL}" || \
          curl -sS -X POST \
            -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            -H "Accept: application/vnd.github+json" \
            -d "{\"name\":\"${LABEL}\",\"color\":\"${COLOR}\"}" \
            "${REPO_LABELS_URL}"

          # Remove opposite label, Unknown label
          curl -sS -X DELETE \
            -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            -H "Accept: application/vnd.github+json" \
            "${API_URL}/${ENCODED_REMOVE}" || true
          curl -sS -X DELETE \
            -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            -H "Accept: application/vnd.github+json" \
            "${API_URL}/Smoke%20tests%3A%20Unknown" || true

          # Add correct label
          curl -sS -X POST \
            -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            -H "Accept: application/vnd.github+json" \
            -d "{\"labels\":[\"${LABEL}\"]}" \
            "${API_URL}"
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Build comment body
        id: comment
        run: |
          RUN_URL="${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
          cat > comment.md <<EOF
          ## Smoke Tests: ${{ steps.outcome.outputs.emoji }} ${{ steps.outcome.outputs.status }}
          **Source:** ${{ steps.outcome.outputs.source }}
          **Run:** [View workflow run](${RUN_URL})
          EOF

      - name: Post comment to PR
        uses: thollander/actions-comment-pull-request@24bffb9b452ba05a4f3f77933840a6a841d1b32b
        with:
          pr-number: ${{ inputs.pr_number }}
          file-path: comment.md
          comment-tag: smoke_test_result
          mode: recreate
```

---

### `set-smoke-label-unknown.yml` (new)

```yaml
name: Set Smoke Tests Unknown Label

on:
  workflow_run:
    workflows: ["CI"]
    types: [requested]

permissions:
  pull-requests: write
  actions: read

jobs:
  set-unknown:
    name: Set Smoke tests Unknown label
    runs-on: ubuntu-latest
    if: github.event.workflow_run.event == 'pull_request'
    steps:
      - name: Set Smoke tests Unknown label
        run: |
          PR="${{ github.event.workflow_run.pull_requests[0].number }}"
          if [ -z "$PR" ] || [ "$PR" = "null" ]; then
            echo "No PR number found, skipping"
            exit 0
          fi

          API_URL="https://api.github.com/repos/${{ github.repository }}/issues/${PR}/labels"
          REPO_LABELS_URL="https://api.github.com/repos/${{ github.repository }}/labels"

          # Ensure Unknown label exists with gray color
          curl -sS -X PATCH \
            -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            -H "Accept: application/vnd.github+json" \
            -d '{"color":"c5def5"}' \
            "${REPO_LABELS_URL}/Smoke%20tests%3A%20Unknown" || \
          curl -sS -X POST \
            -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            -H "Accept: application/vnd.github+json" \
            -d '{"name":"Smoke tests: Unknown","color":"c5def5"}' \
            "${REPO_LABELS_URL}"

          # Remove Pass and Fail labels (all variants)
          for label in \
            "Smoke%20tests%3A%20Pass" \
            "Smoke%20tests%3A%20Fail" \
            "Smoke%20tests%3A%20Pass%20%28tests%20from%20pr%29" \
            "Smoke%20tests%3A%20Fail%20%28tests%20from%20pr%29"; do
            curl -sS -X DELETE \
              -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
              -H "Accept: application/vnd.github+json" \
              "${API_URL}/${label}" || true
          done

          # Add Unknown label
          curl -sS -X POST \
            -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            -H "Accept: application/vnd.github+json" \
            -d '{"labels":["Smoke tests: Unknown"]}' \
            "${API_URL}"
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

### `ci.yaml` change

Remove the `label-oss-system-test-unknown` job entirely. It is replaced by `set-smoke-label-unknown.yml`.

---

### `build-internal.yaml` fix

Replace event-level input references with workflow_call input references in both checkout steps:

```yaml
# Before
- uses: actions/checkout@v6
  if: github.event.inputs.pr_number == ''

- uses: actions/checkout@v6
  if: github.event.inputs.pr_number != ''
  with:
    fetch-depth: 0
    ref: refs/pull/${{ github.event.inputs.pr_number }}/merge

# After
- uses: actions/checkout@v6
  if: inputs.pr_number == ''

- uses: actions/checkout@v6
  if: inputs.pr_number != ''
  with:
    fetch-depth: 0
    ref: refs/pull/${{ inputs.pr_number }}/merge
```

Also remove the `quay.io` Docker login step condition fix — it currently uses `github.event.inputs.pr_number == ''`; update to `inputs.pr_number == ''`.

---

## Label Reference

| Condition | Label |
|-----------|-------|
| Commit pushed to PR / CI starts | `Smoke tests: Unknown` (gray `c5def5`) |
| Upstream code passes | `Smoke tests: Pass` (green `0e8a16`) |
| Upstream code fails | `Smoke tests: Fail` (red `d93f0b`) |
| PR code passes | `Smoke tests: Pass (tests from pr)` (green `0e8a16`) |
| PR code fails | `Smoke tests: Fail (tests from pr)` (red `d93f0b`) |

---

## Edge Cases

- **`pr_number` not set in `smoke-tests.yml`:** `post-results` job is skipped (`if: always() && inputs.pr_number != ''`). No label, no comment. Tests still run.
- **`build_from_pr=true` without `pr_number`:** Should be treated as a configuration error. `prepare-inputs` will produce an upstream hash for `mlrun_hash` (no PR to reference) — the build step will run but check out the default branch (since `inputs.pr_number` is empty). A warning or explicit guard can be added.
- **CI triggered by `push` (not `pull_request`):** `set-smoke-label-unknown.yml` guards with `if: github.event.workflow_run.event == 'pull_request'` — label is not set for push-only CI runs.
- **Stale Unknown label:** If smoke tests are never run after a commit push, the PR will permanently show `Unknown`. This is acceptable — it correctly signals that smoke tests have not been run for the latest code.
- **`workflow_run.pull_requests` empty (fork PRs):** GitHub does not populate `pull_requests` for forks. In that case, `set-smoke-label-unknown.yml` exits early with no error.

---

## Out of Scope

- Automatic triggering of `smoke-tests.yml` from CI (remains manual).
- Smoke test result on `push` events to release branches (not applicable, no PR to label).
