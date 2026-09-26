# Pipeline Audit & Failure Documentation

This document contains the audit findings for `.github/workflows/pipeline.yml` in the `multi-job-pipeline-drill` repository. It details what each job in the initial pipeline was intended to do, the misconfigurations present, and the required fixes.

---

## Audit Summary

| Job | Purpose | Identified Issues | Required Fix |
|---|---|---|---|
| `lint` | Code formatting & syntax checking | Missing job execution timeout; no dependency sequencing. | Add `timeout-minutes: 10`; serve as root job. |
| `unit-tests` | Isolated unit testing (`npm test`) | Missing `needs: lint` dependency; missing timeout. | Add `needs: lint`; add `timeout-minutes: 15`. |
| `build` | Builds production artifacts (`dist/`) | Missing `needs: lint`; build output lost on job completion; missing timeout. | Add `needs: lint`; add `upload-artifact@v4` for `dist/` named `app-build`; set `timeout-minutes: 20`. |
| `integration-tests` | Integration testing on built bundle | Missing `needs: build`; missing artifact download step; missing timeout. | Add `needs: build`; add `download-artifact@v4` for `app-build`; set `timeout-minutes: 30`. |
| `deploy-staging` | Staging environment deployment | Missing `needs: [unit-tests, integration-tests]`; runs on feature branches; missing timeout. | Add `needs: [unit-tests, integration-tests]`; add `if: github.ref == 'refs/heads/main'`; set `timeout-minutes: 15`. |
| `deploy-production` | Gated production deployment | Missing `needs: deploy-staging`; fires on feature branches; missing timeout. | Add `needs: deploy-staging`; add `if: github.ref == 'refs/heads/main'`; set `timeout-minutes: 15`. |
| `notify` | Pipeline status notification | Missing `if: always()`; missing `needs:` on prior jobs; missing timeout. | Add `needs` for prior jobs; add `if: always()`; set `timeout-minutes: 10`. |

---

## Detailed Job Analysis

### 1. Job: `lint`
* **Intended Behavior**: Run ESLint checks against source files (`src/`) to catch syntax and formatting errors before running test suites or building assets.
* **Current Failures / Misconfigurations**:
  1. No `timeout-minutes` specified, risking infinite hanging on runner machines.
  2. Runs without prerequisite checks, but because there are no job dependencies, it runs concurrently with unit tests and builds.
* **Required Fix**:
  * Set `timeout-minutes: 10`.
  * Position as the primary prerequisite job at the top of the workflow graph.

### 2. Job: `unit-tests`
* **Intended Behavior**: Execute unit tests using Jest (`npm test`) to verify core application logic in isolation.
* **Current Failures / Misconfigurations**:
  1. Missing `needs: lint` declaration. Starts immediately upon workflow trigger, wasting compute resources if code fails basic linting rules.
  2. Missing `timeout-minutes` limit.
* **Required Fix**:
  * Add `needs: lint` dependency with an explanatory comment.
  * Set `timeout-minutes: 15`.

### 3. Job: `build`
* **Intended Behavior**: Compile source files into `dist/` (`npm run build`).
* **Current Failures / Misconfigurations**:
  1. Missing `needs: lint` declaration. Runs in parallel with linting and unit testing.
  2. Does not upload the generated `dist/` directory. GitHub Actions runners execute in isolated ephemeral virtual environments; files generated in one runner are lost when the job finishes.
  3. Missing `timeout-minutes` limit.
* **Required Fix**:
  * Add `needs: lint` dependency with comment.
  * Add step `actions/upload-artifact@v4` configured with `name: app-build` and `path: dist`.
  * Set `timeout-minutes: 20`.

### 4. Job: `integration-tests`
* **Intended Behavior**: Run integration tests (`npm run test:integration`), which verify the presence and execution of `dist/api.js`.
* **Current Failures / Misconfigurations**:
  1. Missing `needs: build` dependency. Executes concurrently with `build`, so `dist/` is not ready.
  2. Missing `actions/download-artifact@v4` step. Without fetching `app-build`, `dist/api.js` does not exist on the runner machine, throwing `Error: Build output not found.`.
  3. Missing `timeout-minutes` limit.
* **Required Fix**:
  * Add `needs: build` dependency with comment.
  * Add step `actions/download-artifact@v4` with `name: app-build` and `path: dist`.
  * Set `timeout-minutes: 30`.

### 5. Job: `deploy-staging`
* **Intended Behavior**: Deploy application to staging after unit and integration tests succeed on the `main` branch.
* **Current Failures / Misconfigurations**:
  1. Missing `needs: [unit-tests, integration-tests]` dependencies. Triggers concurrently with testing and building, deploying untested or failing code.
  2. Missing `if:` condition to restrict deployment to `main` branch, triggering staging deploys on PRs and feature branches.
  3. Missing `timeout-minutes` limit.
* **Required Fix**:
  * Add `needs: [unit-tests, integration-tests]` dependency with comment.
  * Add `if: github.ref == 'refs/heads/main'`.
  * Set `timeout-minutes: 15`.

### 6. Job: `deploy-production`
* **Intended Behavior**: Deploy application to production only after staging deployment succeeds on the `main` branch.
* **Current Failures / Misconfigurations**:
  1. Missing `needs: deploy-staging` dependency. Triggers immediately on push without staging verification or test checks.
  2. Missing `if:` condition to restrict deployment to `main` branch, risking production releases from feature branches.
  3. Missing `timeout-minutes` limit.
* **Required Fix**:
  * Add `needs: deploy-staging` dependency with comment.
  * Add `if: github.ref == 'refs/heads/main'`.
  * Set `timeout-minutes: 15`.

### 7. Job: `notify`
* **Intended Behavior**: Send notifications about pipeline completion regardless of whether jobs succeeded or failed.
* **Current Failures / Misconfigurations**:
  1. Missing `if: always()` condition. Defaults to running only when upstream succeeds, skipping notification when jobs fail.
  2. Missing explicit `needs:` dependencies on pipeline jobs, causing it to run prematurely or get skipped.
  3. Missing `timeout-minutes` limit.
* **Required Fix**:
  * Add `needs: [lint, unit-tests, build, integration-tests, deploy-staging, deploy-production]` dependency.
  * Add `if: always()` condition.
  * Set `timeout-minutes: 10`.
