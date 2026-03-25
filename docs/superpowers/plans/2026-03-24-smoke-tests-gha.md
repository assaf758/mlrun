# Smoke Tests GHA Improvements Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Harden the open-source smoke test GitHub Actions workflows so labels are unforgeable, set correctly at every stage, and trace back to their GHA run.

**Architecture:** Five changes across five files — fix `build-internal.yaml`'s input context leak, remove the broken label job from `ci.yaml`, extend `system-tests-opensource.yml` with a `workflow_call` trigger + `build_from_pr` support, and add two new workflows (`set-smoke-label-unknown.yml` for Unknown label on CI start, `smoke-tests.yml` as the locked smoke entry point).

**Tech Stack:** GitHub Actions YAML, `curl`/`jq` for GitHub API calls, `thollander/actions-comment-pull-request` for PR comments, `workflow_run` + `workflow_call` event types.

**Spec:** `docs/superpowers/specs/2026-03-24-smoke-tests-gha-design.md`

---

## File Map

| File | Action | Responsibility |
|------|--------|---------------|
| `.github/workflows/build-internal.yaml` | Modify | Fix 4 `github.event.inputs.pr_number` → `inputs.pr_number` references |
| `.github/workflows/ci.yaml` | Modify | Remove `label-oss-system-test-unknown` job |
| `.github/workflows/system-tests-opensource.yml` | Modify | Add `workflow_call` trigger; `build_from_pr` input; PR-code checkout; expose `test_outcome` output; remove label/comment step; fix `github.event.inputs.*` in step bodies |
| `.github/workflows/set-smoke-label-unknown.yml` | **Create** | Set `Smoke tests: Unknown` via `workflow_run` on CI start |
| `.github/workflows/smoke-tests.yml` | **Create** | Official locked smoke entry point; post-run labeling and PR comment |

---

## Validation approach

These are GitHub Actions YAML files — there are no unit tests. Validation for each task is:

1. **YAML syntax check** — run `python3 -c "import yaml; yaml.safe_load(open('FILE'))"` on the changed file. A parse error means something is wrong.
2. **Logic review** — read the changed section and confirm it matches the spec.

Functional end-to-end testing requires a live GitHub environment, so each task ends with a commit and a note on what to manually verify.

---

## Task 1: Fix `build-internal.yaml` — replace `github.event.inputs.pr_number` with `inputs.pr_number`

**Files:**
- Modify: `.github/workflows/build-internal.yaml` (lines 103, 106, 109, 176, 184)

**Background:** `build-internal.yaml` is called via `workflow_call` from `system-tests-opensource.yml`. Under `workflow_call`, `github.event.inputs` is the caller's event context, not the explicit `with:` inputs. This causes the PR checkout to fire incorrectly when called from a chain where the original dispatch had a `pr_number`. After this fix, the checkout and Docker login conditions are controlled by the `inputs.pr_number` that is explicitly passed.

- [ ] **Step 1: Open the file and locate the five lines**

  Run: `grep -n "github.event.inputs.pr_number" .github/workflows/build-internal.yaml`

  Expected output — five lines (four replacement *actions* cover five lines because Change 2 spans two lines):
  ```
  103:    if: github.event.inputs.pr_number == ''
  106:    if: github.event.inputs.pr_number != ''
  109:        ref: refs/pull/${{ github.event.inputs.pr_number }}/merge
  176:      if: github.event.inputs.pr_number == ''
  184:      if: github.event.inputs.pr_number == ''
  ```

- [ ] **Step 2: Apply the four replacements (covering five lines)**

  Edit `.github/workflows/build-internal.yaml`:

  **Change 1** — Checkout step 1 condition (line ~103):
  ```yaml
  # Before
      if: github.event.inputs.pr_number == ''
  # After
      if: inputs.pr_number == ''
  ```

  **Change 2** — Checkout step 2 condition and ref (lines ~106, ~109):
  ```yaml
  # Before
      if: github.event.inputs.pr_number != ''
      with:
        fetch-depth: 0
        ref: refs/pull/${{ github.event.inputs.pr_number }}/merge
  # After
      if: inputs.pr_number != ''
      with:
        fetch-depth: 0
        ref: refs/pull/${{ inputs.pr_number }}/merge
  ```

  **Change 3** — Docker login (quay.io) condition (line ~176):
  ```yaml
  # Before
        if: github.event.inputs.pr_number == ''
  # After
        if: inputs.pr_number == ''
  ```

  **Change 4** — Docker login (docker.com) condition (line ~184):
  ```yaml
  # Before
        if: github.event.inputs.pr_number == ''
  # After
        if: inputs.pr_number == ''
  ```

- [ ] **Step 3: Verify no remaining stale references**

  Run: `grep -n "github.event.inputs.pr_number" .github/workflows/build-internal.yaml`

  Expected: no output (zero matches).

- [ ] **Step 4: YAML syntax check**

  Run: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/build-internal.yaml'))" && echo "OK"`

  Expected: `OK`

- [ ] **Step 5: Commit**

  ```bash
  git add .github/workflows/build-internal.yaml
  git commit -m "[CI] Fix build-internal.yaml to use inputs.pr_number over event context"
  ```

---

## Task 2: Remove `label-oss-system-test-unknown` from `ci.yaml`

**Files:**
- Modify: `.github/workflows/ci.yaml` (lines 446–488)

**Background:** This job sets the Unknown label but only works on `pull_request` events — it cannot write to PRs when CI is triggered by a `push` event. It is being replaced by `set-smoke-label-unknown.yml` (Task 4) which uses `workflow_run` and has the correct permissions.

- [ ] **Step 1: Locate the job block**

  Run: `grep -n "label-oss-system-test-unknown\|Label PR with Smoke Unknown" .github/workflows/ci.yaml`

  Expected: matches around lines 446–448.

- [ ] **Step 2: Delete the entire job block**

  Remove from `.github/workflows/ci.yaml` the complete `label-oss-system-test-unknown` job (lines 446–488 inclusive):

  ```yaml
  # Delete this entire block:
    label-oss-system-test-unknown:
      name: Label PR with Smoke Unknown
      if: github.event_name == 'pull_request'
      runs-on: ubuntu-latest
      steps:
        - name: Set Smoke tests Unknown label
          run: |
            ...
          env:
            GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  ```

- [ ] **Step 3: YAML syntax check**

  Run: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/ci.yaml'))" && echo "OK"`

  Expected: `OK`

- [ ] **Step 4: Confirm the job is gone**

  Run: `grep -n "label-oss-system-test-unknown" .github/workflows/ci.yaml`

  Expected: no output.

- [ ] **Step 5: Commit**

  ```bash
  git add .github/workflows/ci.yaml
  git commit -m "[CI] Remove label-oss-system-test-unknown job (replaced by set-smoke-label-unknown.yml)"
  ```

---

## Task 3: Modify `system-tests-opensource.yml` — part A: add `workflow_call` trigger and top-level outputs

**Files:**
- Modify: `.github/workflows/system-tests-opensource.yml` (lines 22–48 `on:` block, and the `run-system-tests-opensource-ci` job)

**Background:** Adding `workflow_call` lets `smoke-tests.yml` call this workflow as a sub-workflow. The `outputs:` block at the top level exposes the test result so the caller can decide what label to set. This is modelled after how `build.yaml` calls `build-internal.yaml`.

- [ ] **Step 1: Add `workflow_call` to the `on:` block**

  In `.github/workflows/system-tests-opensource.yml`, the `on:` block currently has only `workflow_dispatch`. Add `workflow_call` after it:

  ```yaml
  on:
    workflow_dispatch:
      inputs:
        pr_number:
          description: 'PR number to run the system tests against (default: run on the latest commit in the development branch)'
          required: false
          default: ''
        pytest_markers:
          description: 'Pytest markers to run'
          required: false
          default: 'not enterprise and smoke'
        clean_resources_in_teardown:
          description: 'Clean test resources upon test teardown'
          required: true
          default: 'true'
          type: choice
          options:
            - 'true'
            - 'false'
        debug_enabled:
          description: 'Allow SSH debugging'
          required: false
          default: 'false'
          type: choice
          options:
            - 'true'
            - 'false'

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

- [ ] **Step 2: Add `outputs:` block to the `run-system-tests-opensource-ci` job**

  Find the `run-system-tests-opensource-ci:` job definition (line ~172). Add an `outputs:` key directly under `needs:`:

  ```yaml
    run-system-tests-opensource-ci:
      name: Run System Tests Open Source
      runs-on:
        group: mlrun-ce
      needs: [prepare-inputs, build-mlrun]
      outputs:
        test_outcome: ${{ steps.system-tests.outcome }}
  ```

- [ ] **Step 3: YAML syntax check**

  Run: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/system-tests-opensource.yml'))" && echo "OK"`

  Expected: `OK`

- [ ] **Step 4: Commit**

  ```bash
  git add .github/workflows/system-tests-opensource.yml
  git commit -m "[CI] Add workflow_call trigger and test_outcome output to system-tests-opensource"
  ```

---

## Task 4: Modify `system-tests-opensource.yml` — part B: fix `github.event.inputs.*` in step bodies

**Files:**
- Modify: `.github/workflows/system-tests-opensource.yml` (lines in `prepare-inputs` job and `run-system-tests-opensource-ci` job env block)

**Background:** Under `workflow_call`, `github.event.inputs` is empty — only `inputs.*` is populated. The `prepare-inputs` job uses `env:` vars to pass values to shell scripts; the env assignments must be updated.

- [ ] **Step 1: Find all `github.event.inputs` references**

  Run: `grep -n "github.event.inputs" .github/workflows/system-tests-opensource.yml`

  Expected — exactly these lines (note: lines containing `github.event_name` or shell variable refs like `$PR_NUMBER` do NOT match this grep):
  ```
  113:          PR_NUMBER: ${{ github.event.inputs.pr_number }}
  147:          INPUT_CLEAN_RESOURCES_IN_TEARDOWN: ${{ github.event.inputs.clean_resources_in_teardown || env.DEFAULT_CLEAN_RESOURCES_IN_TEARDOWN }}
  148:          PR_NUMBER: ${{ github.event.inputs.pr_number }}
  258:        if: ${{ (github.event.inputs.debug_enabled || env.DEFAULT_DEBUG_ENABLED) == 'true' }}
  547:        PYTEST_MARKERS: ${{ github.event.inputs.pytest_markers || env.DEFAULT_PYTEST_MARKERS }}
  ```

  Exact line numbers may shift if the file was modified; the content match is what matters.

- [ ] **Step 2: Update `env:` assignments in `prepare-inputs` to use `inputs.*`**

  In the `prepare-inputs` job, change every `github.event.inputs.*` env assignment:

  **In `Extract git hashes from upstream and latest version` step `env:` block:**
  ```yaml
  # Before
        env:
          PR_NUMBER: ${{ github.event.inputs.pr_number }}
  # After
        env:
          PR_NUMBER: ${{ inputs.pr_number }}
  ```

  **In `Set computed versions params` step `env:` block:**
  ```yaml
  # Before
        env:
          INPUT_CLEAN_RESOURCES_IN_TEARDOWN: ${{ github.event.inputs.clean_resources_in_teardown || env.DEFAULT_CLEAN_RESOURCES_IN_TEARDOWN }}
          PR_NUMBER: ${{ github.event.inputs.pr_number }}
  # After
        env:
          INPUT_CLEAN_RESOURCES_IN_TEARDOWN: ${{ inputs.clean_resources_in_teardown || env.DEFAULT_CLEAN_RESOURCES_IN_TEARDOWN }}
          PR_NUMBER: ${{ inputs.pr_number }}
  ```

  Note: `github.event_name` (without `.inputs`) comparisons like `if [ "${{ github.event_name }}" = "pull_request" ]` inside shell scripts are fine and unchanged — `github.event_name` works correctly under both `workflow_dispatch` and `workflow_call`.

- [ ] **Step 3: Update `PYTEST_MARKERS` env in `Run system tests` step**

  ```yaml
  # Before
        env:
          PYTEST_MARKERS: ${{ github.event.inputs.pytest_markers || env.DEFAULT_PYTEST_MARKERS }}
  # After
        env:
          PYTEST_MARKERS: ${{ inputs.pytest_markers || env.DEFAULT_PYTEST_MARKERS }}
  ```

- [ ] **Step 4: Update `debug_enabled` step condition**

  ```yaml
  # Before
        if: ${{ (github.event.inputs.debug_enabled || env.DEFAULT_DEBUG_ENABLED) == 'true' }}
  # After
        if: ${{ (inputs.debug_enabled || env.DEFAULT_DEBUG_ENABLED) == 'true' }}
  ```

- [ ] **Step 5: Verify no remaining `github.event.inputs` references**

  Run: `grep -n "github.event.inputs" .github/workflows/system-tests-opensource.yml`

  Expected: no output. (References to `github.event_name` and `github.event.pull_request.*` are fine — only `github.event.inputs.*` must be gone.)

- [ ] **Step 6: YAML syntax check**

  Run: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/system-tests-opensource.yml'))" && echo "OK"`

  Expected: `OK`

- [ ] **Step 7: Commit**

  ```bash
  git add .github/workflows/system-tests-opensource.yml
  git commit -m "[CI] Replace github.event.inputs.* with inputs.* for workflow_call compatibility"
  ```

---

## Task 5: Modify `system-tests-opensource.yml` — part C: `build_from_pr` support

**Files:**
- Modify: `.github/workflows/system-tests-opensource.yml` (build-mlrun job, prepare-inputs job, run-system-tests-opensource-ci job)

**Background:** `build_from_pr=true` means: build Docker images from the PR merge ref, check out PR code for tests, and point `mlrun_version_specifier` at the PR. `build_from_pr=false` (default) skips the build job and uses upstream images. The test job's `needs:` must handle `build-mlrun` being skipped.

- [ ] **Step 1: Update `build-mlrun` job condition**

  Find the `build-mlrun` job (line ~154). Change its `if:` condition:

  ```yaml
  # Before
    build-mlrun:
      if: github.event_name == 'pull_request' || github.event.inputs.pr_number != ''
  # After
    build-mlrun:
      if: inputs.build_from_pr == true
  ```

  Also add `pr_number` to the `with:` block (so `build-internal.yaml` can checkout the correct ref after Task 1's fix):

  ```yaml
      with:
        docker_registries: ${{ needs.prepare-inputs.outputs.mlrun_docker_registry }}
        docker_repo: ${{ needs.prepare-inputs.outputs.mlrun_docker_repo }}
        version: ${{ needs.prepare-inputs.outputs.mlrun_version }}
        cache_tag_suffix: ${{ needs.prepare-inputs.outputs.mlrun_docker_tag }}
        skip_images: test,mlrun-gpu,jupyter
        pr_number: ${{ inputs.pr_number }}   # <-- add this line
  ```

- [ ] **Step 2: Update `run-system-tests-opensource-ci` `needs:` condition**

  The job must run even when `build-mlrun` is skipped. Add an `if:` to the job:

  ```yaml
    run-system-tests-opensource-ci:
      name: Run System Tests Open Source
      runs-on:
        group: mlrun-ce
      needs: [prepare-inputs, build-mlrun]
      outputs:
        test_outcome: ${{ steps.system-tests.outcome }}
      if: >-
        always() &&
        needs.prepare-inputs.result == 'success' &&
        (needs.build-mlrun.result == 'success' || needs.build-mlrun.result == 'skipped')
  ```

- [ ] **Step 3: Update `mlrun_version_specifier` logic in `prepare-inputs`**

  In the `Set computed versions params` step, find the `mlrun_version_specifier` block. Update it to also require `BUILD_FROM_PR`:

  ```bash
  # Before:
          export mlrun_version_specifier=$mlrun_hash
          if [ -n "$PR_NUM" ]; then
            mlrun_version_specifier="refs/pull/${PR_NUM}/merge"
          fi

  # After:
          export mlrun_version_specifier=$mlrun_hash
          if [ -n "$PR_NUM" ] && [ "$BUILD_FROM_PR" = "true" ]; then
            mlrun_version_specifier="refs/pull/${PR_NUM}/merge"
          fi
  ```

  Add `BUILD_FROM_PR` to the step's `env:` block:

  ```yaml
        env:
          INPUT_CLEAN_RESOURCES_IN_TEARDOWN: ${{ inputs.clean_resources_in_teardown || env.DEFAULT_CLEAN_RESOURCES_IN_TEARDOWN }}
          PR_NUMBER: ${{ inputs.pr_number }}
          BUILD_FROM_PR: ${{ inputs.build_from_pr }}   # <-- add this line
  ```

- [ ] **Step 4: Add conditional checkout in `run-system-tests-opensource-ci`**

  Find the existing `- uses: actions/checkout@v6` step (line ~178, the first step in `run-system-tests-opensource-ci`). Replace it with two conditional checkouts:

  ```yaml
      - uses: actions/checkout@v6
        if: inputs.build_from_pr != true

      - uses: actions/checkout@v6
        if: inputs.build_from_pr == true
        with:
          ref: refs/pull/${{ inputs.pr_number }}/merge
          fetch-depth: 0
  ```

- [ ] **Step 5: YAML syntax check**

  Run: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/system-tests-opensource.yml'))" && echo "OK"`

  Expected: `OK`

- [ ] **Step 6: Commit**

  ```bash
  git add .github/workflows/system-tests-opensource.yml
  git commit -m "[CI] Add build_from_pr support to system-tests-opensource"
  ```

---

## Task 6: Modify `system-tests-opensource.yml` — part D: remove label/comment step

**Files:**
- Modify: `.github/workflows/system-tests-opensource.yml` (lines 546–601 — `Label PR with system test result` step)

**Background:** Labeling moves exclusively to `smoke-tests.yml`. Direct dispatch of `system-tests-opensource.yml` is now debugging-only and never touches PR labels.

- [ ] **Step 1: Locate the step**

  Run: `grep -n "Label PR with system test result" .github/workflows/system-tests-opensource.yml`

  Expected: one match around line 546.

- [ ] **Step 2: Delete the entire step block**

  Remove from `.github/workflows/system-tests-opensource.yml` the complete step:

  ```yaml
      - name: Label PR with system test result
        if: >-
          always() &&
          (github.event_name == 'pull_request' || github.event.inputs.pr_number != '')
        run: |
          ...
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  ```

  The step starts at the `- name: Label PR with system test result` line and ends at the closing `GITHUB_TOKEN: ...` line under its `env:`.

- [ ] **Step 3: Verify it is gone**

  Run: `grep -n "Label PR with system test result" .github/workflows/system-tests-opensource.yml`

  Expected: no output.

- [ ] **Step 4: YAML syntax check**

  Run: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/system-tests-opensource.yml'))" && echo "OK"`

  Expected: `OK`

- [ ] **Step 5: Commit**

  ```bash
  git add .github/workflows/system-tests-opensource.yml
  git commit -m "[CI] Remove label-setting step from system-tests-opensource (moves to smoke-tests.yml)"
  ```

---

## Task 7: Create `set-smoke-label-unknown.yml`

**Files:**
- Create: `.github/workflows/set-smoke-label-unknown.yml`

**Background:** This workflow fires when CI is *requested* (i.e., just started) for a PR. Using `workflow_run` gives it `pull-requests: write` permission even on push events — which `ci.yaml` itself cannot have. It replaces the deleted `label-oss-system-test-unknown` job.

The key field for the PR number is `github.event.workflow_run.pull_requests[0].number` — this is a JSON array populated by GitHub for non-fork PRs. For fork PRs it is empty; the script handles that with an early exit.

- [ ] **Step 1: Create the file**

  Create `.github/workflows/set-smoke-label-unknown.yml` with this content:

  ```yaml
  # Copyright 2026 Iguazio
  #
  # Licensed under the Apache License, Version 2.0 (the "License");
  # you may not use this file except in compliance with the License.
  # You may obtain a copy of the License at
  #
  #   http://www.apache.org/licenses/LICENSE-2.0
  #
  # Unless required by applicable law or agreed to in writing, software
  # distributed under the License is distributed on an "AS IS" BASIS,
  # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  # See the License for the specific language governing permissions and
  # limitations under the License.

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
              echo "No PR number found in workflow_run context (fork PR?), skipping"
              exit 0
            fi

            API_URL="https://api.github.com/repos/${{ github.repository }}/issues/${PR}/labels"
            REPO_LABELS_URL="https://api.github.com/repos/${{ github.repository }}/labels"

            # Ensure Unknown label exists with gray color (create if absent, update color if present)
            curl -sS -X PATCH \
              -H "Authorization: token ${GITHUB_TOKEN}" \
              -H "Accept: application/vnd.github+json" \
              -d '{"color":"c5def5"}' \
              "${REPO_LABELS_URL}/Smoke%20tests%3A%20Unknown" || \
            curl -sS -X POST \
              -H "Authorization: token ${GITHUB_TOKEN}" \
              -H "Accept: application/vnd.github+json" \
              -d '{"name":"Smoke tests: Unknown","color":"c5def5"}' \
              "${REPO_LABELS_URL}"

            # Remove all Pass/Fail smoke labels (both upstream and "tests from pr" variants)
            for label in \
              "Smoke%20tests%3A%20Pass" \
              "Smoke%20tests%3A%20Fail" \
              "Smoke%20tests%3A%20Pass%20%28tests%20from%20pr%29" \
              "Smoke%20tests%3A%20Fail%20%28tests%20from%20pr%29"; do
              curl -sS -X DELETE \
                -H "Authorization: token ${GITHUB_TOKEN}" \
                -H "Accept: application/vnd.github+json" \
                "${API_URL}/${label}" || true
            done

            # Add Unknown label
            curl -sS -X POST \
              -H "Authorization: token ${GITHUB_TOKEN}" \
              -H "Accept: application/vnd.github+json" \
              -d '{"labels":["Smoke tests: Unknown"]}' \
              "${API_URL}"
          env:
            GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  ```

- [ ] **Step 2: YAML syntax check**

  Run: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/set-smoke-label-unknown.yml'))" && echo "OK"`

  Expected: `OK`

- [ ] **Step 3: Commit**

  ```bash
  git add .github/workflows/set-smoke-label-unknown.yml
  git commit -m "[CI] Add set-smoke-label-unknown workflow (workflow_run on CI start)"
  ```

  **Manual verification after merge:** Open a PR, push a commit — within seconds the `Smoke tests: Unknown` label should appear. Check that the label is also removed from the PR after `smoke-tests.yml` posts a final result.

---

## Task 8: Create `smoke-tests.yml`

**Files:**
- Create: `.github/workflows/smoke-tests.yml`

**Background:** This is the official, locked smoke test entry point. `pytest_markers` is hardcoded — it is not an input, preventing marker gaming. The `post-results` job owns all label logic and posts a PR comment. It runs `if: always()` to post results even when tests fail, and skips entirely if no `pr_number` is provided.

The label removal loop deletes ALL four Pass/Fail smoke variants before adding the correct one — this prevents stale labels when a user switches between `build_from_pr=true` and `build_from_pr=false` across runs. This is a deliberate simplification over the spec's targeted `ENCODED_REMOVE` single-delete: the bulk loop achieves the same result more robustly (handles cross-variant stale labels) with less code.

The PR comment uses a single-quote heredoc (`<<'HEREDOC'`) to avoid shell expansion of `${{ }}` expressions (which are already pre-processed by GHA before the shell runs), and a `sed` placeholder swap for `RUN_URL` (which is a shell variable, not a GHA expression).

- [ ] **Step 1: Create the file**

  Create `.github/workflows/smoke-tests.yml` with this content:

  ```yaml
  # Copyright 2026 Iguazio
  #
  # Licensed under the Apache License, Version 2.0 (the "License");
  # you may not use this file except in compliance with the License.
  # You may obtain a copy of the License at
  #
  #   http://www.apache.org/licenses/LICENSE-2.0
  #
  # Unless required by applicable law or agreed to in writing, software
  # distributed under the License is distributed on an "AS IS" BASIS,
  # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  # See the License for the specific language governing permissions and
  # limitations under the License.

  name: Smoke Tests

  permissions:
    contents: read
    pull-requests: write
    packages: write

  on:
    workflow_dispatch:
      inputs:
        pr_number:
          description: 'PR number to target for labeling (and optionally to build from)'
          required: false
          default: ''
        build_from_pr:
          description: 'Build Docker images and run tests from the PR branch (vs upstream). Requires pr_number.'
          required: false
          default: false
          type: boolean
        clean_resources_in_teardown:
          description: 'Clean test resources upon teardown'
          required: true
          default: 'true'
          type: choice
          options:
            - 'true'
            - 'false'
        debug_enabled:
          description: 'Allow SSH debugging'
          required: false
          default: 'false'
          type: choice
          options:
            - 'true'
            - 'false'

  jobs:
    run-smoke-tests:
      name: Run Smoke Tests
      uses: ./.github/workflows/system-tests-opensource.yml
      with:
        pr_number: ${{ inputs.pr_number }}
        pytest_markers: 'not enterprise and smoke'
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
            else
              STATUS="Fail"
              EMOJI="❌"
              COLOR="d93f0b"
            fi

            if [ "$BUILD_FROM_PR" = "true" ]; then
              LABEL="Smoke tests: ${STATUS} (tests from pr)"
              SOURCE="PR branch"
            else
              LABEL="Smoke tests: ${STATUS}"
              SOURCE="upstream"
            fi

            echo "label=${LABEL}"   >> $GITHUB_OUTPUT
            echo "color=${COLOR}"   >> $GITHUB_OUTPUT
            echo "emoji=${EMOJI}"   >> $GITHUB_OUTPUT
            echo "status=${STATUS}" >> $GITHUB_OUTPUT
            echo "source=${SOURCE}" >> $GITHUB_OUTPUT

        - name: Set label on PR
          run: |
            PR="${{ inputs.pr_number }}"
            API_URL="https://api.github.com/repos/${{ github.repository }}/issues/${PR}/labels"
            REPO_LABELS_URL="https://api.github.com/repos/${{ github.repository }}/labels"
            LABEL="${{ steps.outcome.outputs.label }}"
            COLOR="${{ steps.outcome.outputs.color }}"

            ENCODED_LABEL=$(echo "${LABEL}" | sed 's/ /%20/g; s/:/%3A/g; s/(/%28/g; s/)/%29/g')

            # Ensure label exists in the repo (create if absent, update color if present)
            curl -sS -X PATCH \
              -H "Authorization: token ${GITHUB_TOKEN}" \
              -H "Accept: application/vnd.github+json" \
              -d "{\"color\":\"${COLOR}\"}" \
              "${REPO_LABELS_URL}/${ENCODED_LABEL}" || \
            curl -sS -X POST \
              -H "Authorization: token ${GITHUB_TOKEN}" \
              -H "Accept: application/vnd.github+json" \
              -d "{\"name\":\"${LABEL}\",\"color\":\"${COLOR}\"}" \
              "${REPO_LABELS_URL}"

            # Remove ALL smoke Pass/Fail labels (both variants) and Unknown before adding the correct one.
            # This prevents stale labels when build_from_pr changes between runs.
            for stale in \
              "Smoke%20tests%3A%20Pass" \
              "Smoke%20tests%3A%20Fail" \
              "Smoke%20tests%3A%20Pass%20%28tests%20from%20pr%29" \
              "Smoke%20tests%3A%20Fail%20%28tests%20from%20pr%29" \
              "Smoke%20tests%3A%20Unknown"; do
              curl -sS -X DELETE \
                -H "Authorization: token ${GITHUB_TOKEN}" \
                -H "Accept: application/vnd.github+json" \
                "${API_URL}/${stale}" || true
            done

            # Add the correct label
            curl -sS -X POST \
              -H "Authorization: token ${GITHUB_TOKEN}" \
              -H "Accept: application/vnd.github+json" \
              -d "{\"labels\":[\"${LABEL}\"]}" \
              "${API_URL}"
          env:
            GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

        - name: Build comment body
          run: |
            RUN_URL="${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
            # Single-quote heredoc prevents shell from expanding ${{ }} (already resolved by GHA).
            # RUN_URL is a shell variable so it must be injected via sed.
            cat > comment.md <<'HEREDOC'
  ## Smoke Tests: ${{ steps.outcome.outputs.emoji }} ${{ steps.outcome.outputs.status }}
  **Source:** ${{ steps.outcome.outputs.source }}
  **Run:** [View workflow run](RUN_URL_PLACEHOLDER)
  HEREDOC
            sed -i "s|RUN_URL_PLACEHOLDER|${RUN_URL}|g" comment.md

        - name: Post comment to PR
          uses: thollander/actions-comment-pull-request@24bffb9b452ba05a4f3f77933840a6a841d1b32b
          with:
            pr-number: ${{ inputs.pr_number }}
            file-path: comment.md
            comment-tag: smoke_test_result
            mode: recreate
  ```

- [ ] **Step 2: YAML syntax check**

  Run: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/smoke-tests.yml'))" && echo "OK"`

  Expected: `OK`

- [ ] **Step 3: Commit**

  ```bash
  git add .github/workflows/smoke-tests.yml
  git commit -m "[CI] Add smoke-tests.yml — official locked smoke test entry point"
  ```

  **Manual verification after merge:**
  - Trigger `smoke-tests.yml` via the GitHub UI with a valid `pr_number` (upstream run, `build_from_pr=false`)
  - Confirm: `Smoke tests: Pass` or `Smoke tests: Fail` label appears on the PR
  - Confirm: a comment appears on the PR with the outcome and link to the run
  - Re-trigger with `build_from_pr=true` — confirm label changes to `(tests from pr)` variant and old label is removed

---

## Task 9: Final review and push

- [ ] **Step 1: Confirm all expected files are modified/created**

  Run:
  ```bash
  git log --oneline -9
  ```

  Expected — 8 commits since the start of this work:
  1. Fix build-internal.yaml inputs
  2. Remove label job from ci.yaml
  3. Add workflow_call trigger and output
  4. Fix github.event.inputs.* in step bodies
  5. Add build_from_pr support
  6. Remove label step
  7. Create set-smoke-label-unknown.yml
  8. Create smoke-tests.yml

- [ ] **Step 2: YAML check all changed files**

  ```bash
  for f in \
    .github/workflows/build-internal.yaml \
    .github/workflows/ci.yaml \
    .github/workflows/system-tests-opensource.yml \
    .github/workflows/set-smoke-label-unknown.yml \
    .github/workflows/smoke-tests.yml; do
    echo -n "Checking $f ... "
    python3 -c "import yaml; yaml.safe_load(open('$f'))" && echo "OK" || echo "FAILED"
  done
  ```

  Expected: all `OK`

- [ ] **Step 3: Confirm no remaining stale `github.event.inputs.*` in modified files**

  ```bash
  grep -rn "github.event.inputs" \
    .github/workflows/build-internal.yaml \
    .github/workflows/system-tests-opensource.yml
  ```

  Expected: no output.

- [ ] **Step 4: Confirm `label-oss-system-test-unknown` is gone from ci.yaml**

  Run: `grep -n "label-oss-system-test-unknown" .github/workflows/ci.yaml`

  Expected: no output.

- [ ] **Step 5: Confirm `Label PR with system test result` is gone from system-tests-opensource.yml**

  Run: `grep -n "Label PR with system test result" .github/workflows/system-tests-opensource.yml`

  Expected: no output.

- [ ] **Step 6: Push branch**

  ```bash
  git push origin smoke
  ```
