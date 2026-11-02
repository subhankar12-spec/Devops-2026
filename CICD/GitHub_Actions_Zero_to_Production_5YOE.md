# GitHub Actions: Zero to Production — 5 YOE DevOps Engineer Handbook

**Audience:** An engineer with ~5 years of operations/Linux experience who knows some Jenkins, Docker, Kubernetes, and GitOps but is new to GitHub Actions.  
**Goal:** Design, implement, secure, troubleshoot, and explain production-quality CI/CD pipelines.  
**Method:** Concepts → working YAML → hands-on labs → capstone → interview scenarios.  
**Last reviewed:** November 2026. GitHub Actions evolves: check the official documentation and action release notes before adopting examples in production.

> **Important:** Example `@v4` / `@v5` tags below are readable teaching references, not an assertion that they are the newest versions. In a hardened production repository, use verified full commit SHAs for third-party actions, and keep them updated with automation. Example image versions, role ARNs, repo names, and URLs are placeholders.

## Contents

1. [Mental model and Jenkins comparison](#1-mental-model-and-jenkins-comparison)
2. [Your first workflow](#2-your-first-workflow)
3. [Workflow YAML anatomy](#3-workflow-yaml-anatomy)
4. [Events and triggers](#4-events-and-triggers)
5. [Runners, jobs, steps and DAGs](#5-runners-jobs-steps-and-dags)
6. [Contexts, expressions, conditions and outputs](#6-contexts-expressions-conditions-and-outputs)
7. [Variables, secrets and environments](#7-variables-secrets-and-environments)
8. [Matrix, artifacts, caching and service containers](#8-matrix-artifacts-caching-and-service-containers)
9. [Reusable workflows and composite actions](#9-reusable-workflows-and-composite-actions)
10. [Permissions, security and supply chain](#10-permissions-security-and-supply-chain)
11. [Docker, registry and cloud authentication](#11-docker-registry-and-cloud-authentication)
12. [Deployment strategies and GitOps](#12-deployment-strategies-and-gitops)
13. [Observability, debugging and optimization](#13-observability-debugging-and-optimization)
14. [Hands-on labs 1–16](#14-hands-on-labs-116)
15. [Production capstone: FastAPI → GHCR → GitOps](#15-production-capstone-fastapi--ghcr--gitops)
16. [Interviews, pitfalls and mastery checklist](#16-interviews-pitfalls-and-mastery-checklist)

---

## 1. Mental model and Jenkins comparison

GitHub Actions is an event-driven automation platform integrated with GitHub repositories. Its configuration is YAML under `.github/workflows/`. A **workflow** is triggered by an **event**, contains one or more **jobs**, and each job contains **steps** that execute commands (`run`) or packaged actions (`uses`). Jobs run on **runners**. Steps in the same job share the runner's filesystem; separate jobs generally start with separate runner environments. [Official overview](https://docs.github.com/en/actions/concepts/workflows-and-actions/workflows).

| Jenkins | GitHub Actions | Important distinction |
|---|---|---|
| `Jenkinsfile` | `.github/workflows/*.yml` | Multiple workflows per repository are normal |
| Pipeline | Workflow | Triggered by GitHub events, manual requests or schedules |
| Stage | Job (rough analogy) | Jobs appear as graph nodes; steps appear inside job logs |
| Agent/node | Runner | GitHub-hosted or self-hosted |
| `sh '...'` | `run: ...` | Commands execute in the selected shell |
| Shared library | Reusable workflow / composite action | Workflow reuse versus step reuse |
| Credentials | Secrets / OIDC | Prefer temporary federated cloud credentials |
| `parallel` | Independent jobs / matrix | Jobs without `needs` can run concurrently |
| `when` | `if` | Expression-controlled execution |
| `post { always }` | `if: ${{ always() }}` | Useful for summaries or cleanup |
| Build parameters | `workflow_dispatch.inputs` | Manual trigger parameters |

**Question: Are steps visible in the pipeline UI?** Yes. The main visual graph emphasizes **jobs**; click a job to view the individual **steps**, their timing, and expandable logs. A step is *not* its own graph node. Use separate jobs when you need distinct scheduling, isolation, permissions or approvals—not merely to make every shell command look like a Jenkins stage.

Typical application path:

```text
Developer PR → lint + unit tests + security checks → merge to main
             → image build → image scan → registry push
             → GitOps repository image digest update → Argo CD reconciliation
             → Kubernetes rollout and monitoring
```

## 2. Your first workflow

Create any GitHub repository; add `.github/workflows/hello.yml`:

```yaml
name: 01 Hello GitHub Actions

on:
  workflow_dispatch:

permissions:
  contents: read

jobs:
  hello:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - name: Print a message
        run: echo 'Hello from GitHub Actions'

      - name: Print runner details
        run: |
          echo "Repository: $GITHUB_REPOSITORY"
          echo "Commit: $GITHUB_SHA"
          uname -a
```

Commit the file to the default branch. In GitHub, open **Actions → 01 Hello GitHub Actions → Run workflow**. Open the run, click `hello`, inspect the two steps and their logs. `workflow_dispatch` workflows need to be on the default branch to be manually runnable from the UI.

Line by line:

- `name`: human-readable workflow title.
- `on`: events that start a run; here the manual event.
- `permissions`: least-privilege capabilities for `GITHUB_TOKEN`.
- `jobs.hello`: job ID; `hello` is a label chosen by us.
- `runs-on`: runner selection; `ubuntu-latest` is a moving image label.
- `steps`: sequential tasks within the runner.
- `run`: shell code; `|` creates a multiline YAML string.

`uses` **invokes an existing action**; `run` **executes commands**. Two actions you will repeatedly see:

```yaml
- uses: actions/checkout@v4       # Download repository code into runner workspace
- uses: actions/setup-python@v5   # Install/select a Python version
  with:
    python-version: '3.12'
```

Without checkout, a fresh runner does not automatically have your repository's source files.

## 3. Workflow YAML anatomy

```yaml
name: Example CI
run-name: CI for ${{ github.ref_name }} by @${{ github.actor }}

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

env:
  APP_NAME: order-api

defaults:
  run:
    shell: bash

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    name: Unit tests
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - name: Inspect application
        run: echo "Testing $APP_NAME at $GITHUB_SHA"
```

Key levels: workflow (`name`, `on`, `permissions`, `env`, `concurrency`), job (`runs-on`, `needs`, `strategy`, `environment`, `permissions`, `steps`), step (`name`, `id`, `if`, `uses`/`run`, `with`, `env`). Indentation is semantic. Use spaces, not tabs.

**Common syntax trap:** `on:` should remain a YAML mapping key. Some general-purpose YAML tools based on YAML 1.1 interpret `on` as Boolean; GitHub's workflow parser treats it as the event key. Use GitHub-aware linting such as `actionlint`.

## 4. Events and triggers

```yaml
on:
  push:
    branches: [main, 'release/**']
    paths:
      - 'app/**'
      - '.github/workflows/ci.yml'
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      target:
        description: Deployment target
        required: true
        type: choice
        options: [dev, staging]
  schedule:
    - cron: '17 2 * * 1'
```

| Event | Typical purpose | Watch out |
|---|---|---|
| `push` | Test builds, publish on `main` | Branch/path filters can skip expected runs |
| `pull_request` | Validate PR changes | Fork PRs have restricted tokens/secrets |
| `workflow_dispatch` | Manual jobs, controlled maintenance | Input values need validation |
| `schedule` | Regular scans | Cron is UTC and scheduling can be delayed |
| `release` | Package/release publishing | Validate release source and tag |
| `workflow_call` | Reusable workflow | Called as a job, not a step |
| `workflow_run` | Chain workflows | Privilege boundary requires care |
| `repository_dispatch` | External event integration | Requires authenticated API trigger |

`branches` filters the target branch for `pull_request`. For a `push`, it filters the pushed ref. `paths` and `branches` combine: both must match. A skipped required PR workflow may block branch protection; choose checks carefully. `schedule` only executes on the default branch.

**Security warning:** `pull_request_target` is not a drop-in replacement for `pull_request`. It runs in the base repository's trust context. Never check out and execute untrusted PR code in that context with write tokens or secrets. [GitHub security guide](https://docs.github.com/en/actions/reference/security/securely-using-pull_request_target).

## 5. Runners, jobs, steps and DAGs

```yaml
name: DAG demonstration
on: workflow_dispatch
permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: echo 'tests passed'

  lint:
    runs-on: ubuntu-latest
    steps:
      - run: echo 'lint passed'

  build:
    needs: [test, lint]
    runs-on: ubuntu-latest
    steps:
      - run: echo 'build after BOTH prerequisites'

  summary:
    needs: [test, lint, build]
    if: ${{ always() }}
    runs-on: ubuntu-latest
    env:
      TEST_RESULT: ${{ needs.test.result }}
      LINT_RESULT: ${{ needs.lint.result }}
      BUILD_RESULT: ${{ needs.build.result }}
    steps:
      - run: echo "test=$TEST_RESULT lint=$LINT_RESULT build=$BUILD_RESULT"
```

`test` and `lint` start independently. `build` waits for both success. `summary` evaluates after dependencies finish even if they failed/skipped; it reports outcomes. By default, a failed prerequisite skips dependent jobs.

**Isolation:** Steps of *one* job share the same runner workspace and can see files created earlier in that job. Jobs run in distinct environments; transfer needed files using artifacts or external storage, and small metadata using job outputs. Installing a tool in Job A does not install it in Job B.

GitHub-hosted runners are convenient and ephemeral. Self-hosted runners provide custom hardware/network access, but you own patching, credentials, isolation, cleanup and cost. Never expose a sensitive persistent runner to arbitrary untrusted PR code.

## 6. Contexts, expressions, conditions and outputs

Expressions use `${{ ... }}`. Shell expansion uses `$NAME` or `${NAME}`. They happen at different times.

| Context | Example | Meaning |
|---|---|---|
| `github` | `${{ github.sha }}` | Triggering commit |
| `github` | `${{ github.event_name }}` | `push`, `pull_request`, etc. |
| `runner` | `${{ runner.os }}` | Runner operating system |
| `matrix` | `${{ matrix.python }}` | Current matrix entry |
| `vars` | `${{ vars.APP_ENV }}` | Configuration variable |
| `secrets` | `${{ secrets.DEPLOY_TOKEN }}` | Encrypted secret |
| `inputs` | `${{ inputs.target }}` | Manual/reusable workflow input |
| `steps` | `${{ steps.meta.outputs.version }}` | Step output |
| `needs` | `${{ needs.build.outputs.image }}` | Upstream job output |

```yaml
jobs:
  generate:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.make.outputs.version }}
    steps:
      - id: make
        run: echo "version=1.2.${GITHUB_RUN_NUMBER}" >> "$GITHUB_OUTPUT"

  consume:
    needs: generate
    runs-on: ubuntu-latest
    env:
      VERSION: ${{ needs.generate.outputs.version }}
    steps:
      - run: echo "Version=$VERSION"
```

Important runner command files:

- `$GITHUB_ENV`: sets environment variables for **later steps in the same job**.
- `$GITHUB_OUTPUT`: sets named step outputs; map to job outputs when crossing jobs.
- `$GITHUB_PATH`: prepends executable search paths for later steps.
- `$GITHUB_STEP_SUMMARY`: writes a Markdown run summary.

```yaml
- name: Set build flag
  run: echo 'BUILD_MODE=release' >> "$GITHUB_ENV"
- name: Use build flag
  run: echo "$BUILD_MODE"
```

Use `if: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}` for conditions. Functions include `success()`, `failure()`, `cancelled()`, `always()` and `contains()`. For straightforward cleanup, prefer `if: ${{ !cancelled() }}` where appropriate to avoid delaying cancellation. **Do not embed untrusted `${{ github.event.* }}` values directly into shell code**; pass them via `env` and quote shell variables.

## 7. Variables, secrets and environments

Repository-level settings: **Settings → Secrets and variables → Actions**. Create values on either the **Variables** or **Secrets** tab. Organization-level values can be shared with selected repositories. Environment-level values belong to a named deployment environment.

| Feature | `env` | `vars` | `secrets` |
|---|---|---|---|
| Created in | YAML | Repo/org/environment settings | Repo/org/environment settings |
| Example | `env.APP: api` | `${{ vars.REGION }}` | `${{ secrets.API_TOKEN }}` |
| Intended for | Job/step runtime config | Non-sensitive managed config | Sensitive credentials |
| Encrypted secret storage | No | No | Yes |
| Masked in logs by GitHub | No | No | Known secret values, best effort |

To create an environment: **Settings → Environments → New environment → production**; configure allowed branches, required reviewers where available for your plan/repository visibility, and environment-specific secrets/variables. A job refers to it:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    env:
      REGION: ${{ vars.AWS_REGION }}
      API_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
    steps:
      - run: echo "Deploying to $REGION"
```

Environment controls protect **jobs**, not all workflows globally. An environment secret is made available to an eligible job only after required protections pass. Repository variables and secrets are not automatically interchangeable. A secret absent from a fork PR generally evaluates to an empty string; do not build a test suite that requires secret access for untrusted PRs.

**Principle:** Prefer OIDC for cloud access; prefer scoped `GITHUB_TOKEN` for GitHub API and package operations; use stored long-lived secrets only if unavoidable. Do not print secrets, place them in artifact files, or expose them via debugging traces.

## 8. Matrix, artifacts, caching and service containers

### Matrix: one job definition, many independent runs

```yaml
name: Matrix tests
on: workflow_dispatch
permissions:
  contents: read
jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest]
        python: ['3.11', '3.12']
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python }}
          cache: pip
      - run: |
          python -m pip install -r requirements.txt
          python -m pytest -q
```

This creates **four** combinations: Ubuntu/Python 3.11, Ubuntu/3.12, Windows/3.11, Windows/3.12. Matrix lets you verify cross-platform compatibility, version support and independent test partitions. `fail-fast: false` means a failure does not cancel the other matrix entries. Use `include` for special configurations and `exclude` to remove combinations. Avoid combinatorial explosion (e.g., 4 OS × 5 versions × 3 databases = 60 jobs).

### Artifacts versus cache

- **Artifact:** a specific run's output, e.g., coverage report, compiled binary, test results. Transfer between jobs or download for diagnosis.
- **Cache:** reusable downloaded dependencies or build intermediates across runs, e.g., pip packages. It is an optimization, not a correctness dependency or secret store.

```yaml
- uses: actions/upload-artifact@v4
  if: ${{ always() }}
  with:
    name: test-report
    path: reports/
    if-no-files-found: warn
    retention-days: 7
```

Another job can use `actions/download-artifact@v4` with `name: test-report` after `needs:` ensures the producer finished. For pip, `setup-python` with `cache: pip` plus a committed requirements/lock file gives a simple dependency cache. Do not cache mutable deployment results as if they were immutable release artifacts.

### Service containers

Useful for integration tests with PostgreSQL, Redis, etc.:

```yaml
jobs:
  integration:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: testpassword
          POSTGRES_DB: orders_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U postgres"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 10
    env:
      DATABASE_URL: postgresql://postgres:testpassword@localhost:5432/orders_test
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: python -m pip install -r requirements.txt
      - run: python -m pytest -q tests/integration
```

Those credentials are deliberately disposable test-only values, not production secrets. If the job itself runs in a container, service hostnames/networking differ from a runner-host job; consult container networking documentation.

## 9. Reusable workflows and composite actions

### Reusable workflow: share whole jobs

`.github/workflows/reusable-test.yml`:

```yaml
name: Reusable Python tests
on:
  workflow_call:
    inputs:
      python-version:
        required: true
        type: string
permissions:
  contents: read
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ inputs.python-version }}
      - run: python -m pip install -r requirements.txt
      - run: python -m pytest -q
```

Caller `.github/workflows/caller.yml`:

```yaml
name: CI calling reusable workflow
on: workflow_dispatch
permissions:
  contents: read
jobs:
  python-tests:
    uses: ./.github/workflows/reusable-test.yml
    with:
      python-version: '3.12'
```

Reusable workflows are referenced at **job** level. You cannot add `runs-on` and arbitrary `steps` to that calling job. For cross-repo usage, reference a trusted repository, path and immutable ref; set explicit permissions, and pass only necessary secrets. Called workflows cannot elevate the permissions provided by the caller.

### Composite action: share a bundle of steps

`.github/actions/python-check/action.yml`:

```yaml
name: Python checks
description: Install dependencies and execute pytest
runs:
  using: composite
  steps:
    - name: Install
      shell: bash
      run: python -m pip install -r requirements.txt
    - name: Test
      shell: bash
      run: python -m pytest -q
```

Use it inside an existing job after checkout and Python setup:

```yaml
- uses: ./.github/actions/python-check
```

**Decision rule:** Need multiple reusable jobs, runner selection, permissions or matrix? Reusable workflow. Need a convenient sequence inside one job? Composite action. Keep a business-specific one-off step inline.

## 10. Permissions, security and supply chain

Every pipeline is part of your attack surface. Secure GitHub Actions is a **5-YOE/production-level topic**, not an optional bonus.

1. **Least-privilege token:** start at `permissions: contents: read`; grant write permissions only to a specific job that needs them.
2. **Action immutability:** pin externally maintained actions to verified full commit SHAs in production (tags like `@v4` are convenient for learning but movable).
3. **Cloud federation:** use short-lived OIDC credentials with tightly constrained cloud-side trust policy; avoid permanent AWS access keys.
4. **Untrusted PRs:** do not run fork code with deployment secrets, write-enabled tokens or privileged self-hosted runners.
5. **Script injection:** treat PR titles, branch names, issue bodies and other event fields as untrusted input; use environment variables and careful quoting.
6. **Dependency security:** use dependency scanning, SAST, container scanning, lockfiles, review and actionable policy gates.
7. **Artifact security:** avoid persisting credentials; define minimal retention and control who consumes artifacts.
8. **Deploy gates:** protect production environments, constrain branches and enforce approvals where supported.
9. **Supply chain provenance:** consider image digests, artifact attestations, SBOMs and signatures according to the organization's risk profile.
10. **Runner hardening:** patch and isolate self-hosted runners; favor ephemeral runners, network segmentation and no shared admin credentials.

For sensitive write operations, add only needed permissions:

```yaml
permissions:
  contents: read
  packages: write
```

AWS OIDC job also typically needs:

```yaml
permissions:
  contents: read
  id-token: write
```

`id-token: write` permits requesting an OIDC identity token; it does **not** automatically grant AWS permissions. AWS IAM trust and role policy determine what the exchanged credentials can do. Restrict the role to the intended repo, branch/environment and required resources. [Official security guidance](https://docs.github.com/en/actions/reference/security/secure-use) and [OIDC guide](https://docs.github.com/en/actions/concepts/security/openid-connect).

## 11. Docker, registry and cloud authentication

### Build and push image to GitHub Container Registry (GHCR)

The workflow below is a **learning-grade** pattern that publishes on `main`. Require PR checks and branch protection before using this repository in a real delivery process.

```yaml
name: Build and publish container
on:
  push:
    branches: [main]
permissions:
  contents: read
  packages: write
jobs:
  image:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
```

GHCR package naming/access policies must match your repository and organization settings. Depending on package visibility/permissions, deployments may need separate read access. A commit-SHA tag helps traceability but remains a **tag** and can be moved; the **content digest** (`sha256:...`) is the immutable deployment identifier. Prefer promoting the exact same built digest from staging to production instead of rebuilding production separately.

### AWS with OIDC (replace placeholders)

Prerequisites: Configure GitHub's OIDC provider in AWS IAM, create a scoped role, set its trust policy to the exact repository and expected ref/environment, and set `AWS_ROLE_ARN` as a repository variable.

```yaml
name: AWS Identity Check
on: workflow_dispatch
permissions:
  contents: read
  id-token: write
jobs:
  identity:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ap-south-1
      - run: aws sts get-caller-identity
```

This check performs no AWS resource creation. Avoid putting an administrative role behind this workflow. For an ECR pipeline, additionally obtain an ECR login token, build/push the image, and use the resulting digest. For EKS, prefer GitOps and short-lived cluster test environments to contain expense.

## 12. Deployment strategies and GitOps

Understand four deployment patterns:

| Pattern | How it works | Tradeoff |
|---|---|---|
| Push deployment | CI runs `kubectl`/Helm against cluster | Simple, but CI needs cluster credentials and network access |
| GitOps | CI changes desired state in Git; Argo CD/Flux reconciles | Better separation and audit trail, extra repo/controller setup |
| Rolling deployment | Replace replicas gradually | Default Kubernetes behavior; needs readiness probes |
| Blue/green or canary | Shift traffic between versions/increments | Safer releases but requires routing, telemetry and rollback design |

Your existing **mega-local** lab already uses the right conceptual split:

```text
Application repository
  PR → GitHub Actions test/scan → merge
  main → build image once → push to registry → capture digest
                                        ↓
GitOps repository                      commit digest to environment overlay
                                        ↓
Argo CD                                sync/reconcile
                                        ↓
Kubernetes                             rollout + readiness + monitoring
```

The GitOps repository update needs **separate credentials** with permission to that repository (such as a narrowly scoped GitHub App installation token or fine-grained token). The source repository's default `GITHUB_TOKEN` does not automatically have cross-repository write access. Protect the GitOps main branch and consider having CI create a PR for production, while dev can auto-merge under policy. Never place a PAT directly in YAML.

Avoid assuming an Argo CD application is healthy just because CI published an image. Verify all three: image digest in registry, desired digest committed in GitOps, and deployed image digest in Kubernetes. This directly addresses the common “pipeline succeeded but image not reflecting” failure.

## 13. Observability, debugging and optimization

When a run fails, diagnose in this order:

1. **Trigger:** Did the event/branch/path filter match? Is the workflow on the right branch?
2. **Graph:** Was the job skipped because a dependency failed or `if` evaluated false?
3. **Runner:** Does OS/shell/architecture match? Is disk space/network adequate?
4. **Checkout:** Was correct commit/ref checked out? Is checkout actually present?
5. **Dependency install:** Lockfile mismatch, rate limit, package repository outage, cache confusion?
6. **Permissions:** Does `GITHUB_TOKEN` have `packages: write` or `contents: write` if needed? Is an environment gate blocking?
7. **Secrets/OIDC:** Is the secret available for this event? Does IAM trust policy match the token subject?
8. **Artifacts:** Does producing job upload expected path? Is consuming job dependent on producer?
9. **Deployment:** Correct image digest? Correct namespace/overlay? Argo CD sync and Kubernetes rollout healthy?

Useful observability patterns:

```yaml
- name: Summary
  if: ${{ !cancelled() }}
  run: |
    echo '## Build summary' >> "$GITHUB_STEP_SUMMARY"
    echo "Commit: $GITHUB_SHA" >> "$GITHUB_STEP_SUMMARY"
```

Set `timeout-minutes` to prevent indefinite jobs. Use cache for dependencies and BuildKit layer cache only after validating correctness. Use `concurrency` to cancel redundant **CI** runs per branch, but for production deploys avoid thoughtlessly canceling an in-flight release. Set `max-parallel` for expensive matrices. Track run duration, failure causes, time-to-restore and actual hosted-runner usage.

CLI: `gh run list`, `gh run view RUN_ID --log-failed`, `gh run watch RUN_ID`, `gh workflow list`, `gh workflow run NAME`. Authenticated `gh` CLI and correct repository context are required. Validate YAML with `actionlint`, plus branch-protected PR checks.

## 14. Hands-on labs 1–16

**Lab setup:** Create `github-actions-labs`, clone it locally, create `.github/workflows/`, enable Actions in repository settings and commit each lab as a separate file. In examples using Python, create a real `requirements.txt` (e.g., `pytest==8.3.5` for a pinned tutorial baseline) and `tests/test_math.py` containing a trivial test. Pin dependencies to verified supported versions for production. Each lab specifies an expected result; don't proceed before observing it.

### Lab 01 — Manual workflow and log inspection

**Objective:** Understand workflow, job, step, runner and `run`.  
**Procedure:** Use the complete workflow from Section 2; commit, manually run, open job and expand logs.  
**Verify:** One job and two visible step logs.  
**Challenge:** Add a third step printing `pwd`, `whoami` and `date -u`.

### Lab 02 — Push and pull request CI

**Objective:** Learn event filters and checkout.

```yaml
name: 02 Push and PR
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
permissions:
  contents: read
jobs:
  inspect:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Event details
        env:
          EVENT_NAME: ${{ github.event_name }}
          REF_NAME: ${{ github.ref_name }}
        run: |
          echo "event=$EVENT_NAME ref=$REF_NAME"
          git log -1 --oneline
```

**Procedure:** Push to `main`, create a feature branch and open PR to `main`.  
**Verify:** Push run and PR run have different `github.event_name`.  
**Challenge:** Restrict the push trigger to `app/**`, then prove a docs-only commit does not trigger it.

### Lab 03 — Job dependency graph

**Objective:** Parallel checks and dependent build.  
**Procedure:** Run the complete DAG workflow in Section 5.  
**Verify:** `test` and `lint` are independent, `build` waits, `summary` reports statuses.  
**Failure test:** Change `test` to `exit 1`; see `build` skipped and `summary` still executed.

### Lab 04 — Step condition and environment propagation

```yaml
name: 04 Conditions and env
on: workflow_dispatch
permissions:
  contents: read
jobs:
  demo:
    runs-on: ubuntu-latest
    steps:
      - name: Set variable
        run: echo 'COLOR=blue' >> "$GITHUB_ENV"
      - name: Read variable
        run: test "$COLOR" = blue
      - name: Conditional step
        if: ${{ github.ref == 'refs/heads/main' }}
        run: echo 'Running on main'
```

**Verify:** Second step sees `COLOR`; conditional step behavior follows selected branch.  
**Challenge:** Add `if: ${{ success() }}` and intentionally fail the preceding step.

### Lab 05 — Step outputs and job outputs

**Objective:** Exchange small metadata across jobs.  
**Procedure:** Run Section 6's `generate`/`consume` workflow (add top-level `name`, `on`, `permissions`).  
**Verify:** Consumer prints a version generated in producer.  
**Challenge:** Generate `image_tag=${GITHUB_SHA:0:12}` using bash and map it as an output.

### Lab 06 — Repository variable, secret and environment

```yaml
name: 06 Configuration
on: workflow_dispatch
permissions:
  contents: read
jobs:
  config:
    runs-on: ubuntu-latest
    environment: dev
    env:
      APP_REGION: ${{ vars.APP_REGION }}
      DEMO_SECRET: ${{ secrets.DEMO_SECRET }}
    steps:
      - run: |
          test -n "$APP_REGION"
          test -n "$DEMO_SECRET"
          echo 'Configuration provided successfully'
```

**Procedure:** Create `dev` environment; add `APP_REGION=ap-south-1` and a disposable `DEMO_SECRET` in that environment's Variables/Secrets; run workflow.  
**Verify:** Success without printing secret.  
**Challenge:** Remove the variable and observe failure, then restore it. Compare repository versus environment configuration.

### Lab 07 — Python unit-test workflow

Create `requirements.txt`: `pytest==8.3.5`, and `tests/test_math.py`:

```python
def test_sum():
    assert 2 + 3 == 5
```

Workflow:

```yaml
name: 07 Python CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
permissions:
  contents: read
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: pip
      - run: python -m pip install -r requirements.txt
      - run: python -m pytest -q
```

**Verify:** One successful test.  
**Failure test:** Change assertion to `== 6`, open PR and inspect failing job.

### Lab 08 — Matrix across OS and Python versions

**Procedure:** Run the complete matrix workflow in Section 8 with Lab 07 repository files.  
**Verify:** Four independently visible job instances, one per OS/version pair.  
**Challenge:** Add `python: '3.13'` after confirming your dependencies support it; evaluate runtime impact. Set `max-parallel: 2`.

### Lab 09 — Test artifacts

```yaml
name: 09 Artifacts
on: workflow_dispatch
permissions:
  contents: read
jobs:
  produce:
    runs-on: ubuntu-latest
    steps:
      - run: |
          mkdir -p reports
          echo 'sample report' > reports/report.txt
      - uses: actions/upload-artifact@v4
        with:
          name: report
          path: reports/
  consume:
    needs: produce
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: report
          path: downloaded
      - run: test -s downloaded/report.txt
```

**Verify:** The downloaded artifact exists in the second job; find the artifact on the run page.  
**Challenge:** Change the producing path and inspect upload behavior.

### Lab 10 — Security checks

```yaml
name: 10 Security gates
on:
  pull_request:
    branches: [main]
  workflow_dispatch:
permissions:
  contents: read
jobs:
  audit:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - name: Install dependency scanner
        run: python -m pip install pip-audit
      - name: Check Python dependencies
        run: python -m pip_audit -r requirements.txt
```

**Verify:** Clean dependencies pass; vulnerable dependencies cause an actionable failure.  
**Challenge:** Add CodeQL code scanning for supported languages using GitHub's maintained setup instructions. Add a Docker image scan only after a Dockerfile exists. Do not indiscriminately suppress scanner findings to get a green pipeline.

### Lab 11 — Docker build and GHCR publish

Create a `Dockerfile` suitable for your app.  
**Procedure:** Run the full Section 11 GHCR workflow on a protected `main`; confirm the package is visible and inspect tag.  
**Verify:** New image exists at GHCR; `docker pull` succeeds with appropriate package access.  
**Challenge:** Capture and print the build's digest (not just tag), then add Trivy/another approved scanner as a policy gate before promotion. Note: if the scanner runs *after* publishing, the image is already published; enforce scan-before-publish or gate promotion explicitly.

### Lab 12 — PostgreSQL integration tests

**Procedure:** Use the full Section 8 service-container workflow with a real integration test that connects to `orders_test`, creates and queries a table, then closes the connection. Add `psycopg` or your supported DB client to test requirements.  
**Verify:** Service readiness and test success.  
**Failure test:** Intentionally use wrong DB password and inspect authentication failure.

### Lab 13 — Reusable workflow

**Procedure:** Add both files from Section 9 and invoke `caller.yml` manually.  
**Verify:** Caller contains a reusable workflow job, and its internal test job/steps can be inspected.  
**Challenge:** Call the same test workflow with a matrix of two Python versions.

### Lab 14 — Environment approval and concurrency

```yaml
name: 14 Deployment gate
on: workflow_dispatch
permissions:
  contents: read
concurrency:
  group: production-release
  cancel-in-progress: false
jobs:
  deploy:
    environment: production
    runs-on: ubuntu-latest
    steps:
      - run: echo 'Approved production simulation'
```

**Procedure:** Create `production` and configure required reviewers if available on your plan/repo. Run twice.  
**Verify:** Approval gate appears if configured; concurrency controls overlapping runs.  
**Challenge:** Explain why `cancel-in-progress: true` can be unsafe for real production deployment jobs.

### Lab 15 — AWS OIDC without deployment

**Procedure:** With least-privilege AWS role and exact IAM trust conditions, run the identity-only workflow in Section 11.  
**Verify:** `aws sts get-caller-identity` returns the intended assumed role.  
**Failure test:** Narrow trust policy to a different branch and verify assumption is denied, then restore.  
**Cost note:** STS identity verification itself does not provision EKS/EC2. No permanent access keys are needed in GitHub.

### Lab 16 — Self-hosted runner investigation

**Objective:** Understand labels, lifecycle, network and threats.  
**Procedure:** Read GitHub's self-hosted runner setup docs; register a **dedicated, disposable** lab runner with only necessary permissions. Run a harmless manual `echo` workflow using a matching custom label.  
**Verify:** Job runs on your selected host; GitHub shows runner online/offline status.  
**Challenge:** Explain how you would make runners ephemeral in AWS, prohibit untrusted PRs, patch runner images and destroy the execution environment after each job. Do not place unrestricted public PR workloads on your home Debian machine.

---

## 15. Production capstone: FastAPI → GHCR → GitOps

This project is designed around your **order FastAPI service** and Argo CD learning. It is intentionally split into *three workflows/files* to separate PR validation, trusted mainline publishing, and desired-state promotion.

### 15.1 Repository contract

```text
order-api/
├── .github/workflows/
│   ├── pr.yml
│   └── image.yml
├── app/                         # FastAPI application
├── tests/                       # unit/integration tests
├── requirements.txt
└── Dockerfile

order-gitops/
└── environments/
    ├── dev/                     # manifests or Kustomize overlay
    ├── staging/
    └── production/
```

Your service must have genuinely passing tests, a health endpoint, and a Dockerfile that builds successfully. The generic examples below cannot infer your actual import paths, Kubernetes manifests or GitOps file shape.

### 15.2 PR quality gate (`.github/workflows/pr.yml`)

```yaml
name: Pull request quality gate
on:
  pull_request:
    branches: [main]
permissions:
  contents: read
concurrency:
  group: pr-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
jobs:
  tests:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: pip
      - run: python -m pip install -r requirements.txt
      - run: python -m pytest -q

  docker-build:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: false
          tags: order-api:pr-check
```

Both jobs are independent; if either fails, branch protection should prevent merging. Add lint, SAST, SCA and integration test jobs when project assets support them. To scan the actual built image, use a build-to-local-Docker or OCI artifact workflow so scanner and builder inspect the **same image**; a `push: false` build alone does not automatically create a Docker daemon image for a later scanner to inspect.

### 15.3 Trusted publish (`.github/workflows/image.yml`)

```yaml
name: Publish trusted image
on:
  push:
    branches: [main]
permissions:
  contents: read
  packages: write
concurrency:
  group: publish-${{ github.ref }}
  cancel-in-progress: false
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: pip
      - run: python -m pip install -r requirements.txt
      - run: python -m pytest -q

  publish:
    needs: test
    runs-on: ubuntu-latest
    outputs:
      digest: ${{ steps.push.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
      - name: Record immutable reference
        env:
          DIGEST: ${{ steps.push.outputs.digest }}
        run: echo "Image digest: $DIGEST" >> "$GITHUB_STEP_SUMMARY"
```

This creates a test-to-publish dependency and captures digest in `publish.outputs.digest`. **Production hardening still required:** pin external actions to SHAs; scan and gate the exact produced image **before deployment promotion**; generate SBOM/provenance as policy dictates; keep registries and deploy credentials scoped; enforce code review. For stricter publish policies, build once to an OCI artifact, scan it, then push that exact artifact, or publish to a quarantined location and promote only on passing scan.

### 15.4 GitOps promotion: controlled implementation contract

Do not blindly append YAML with `sed` to an unknown manifest. Implement promotion in a separate GitOps repository with these precise requirements:

1. A trusted publish job obtains `ghcr.io/OWNER/REPO@sha256:...` from the image build output.
2. An automation identity with **write access only to the relevant GitOps repository** checks out the targeted environment overlay.
3. A format-aware update changes only the intended container's image digest in `environments/dev` (e.g., `kustomize edit set image` or an explicit YAML parser).
4. It validates the rendered manifests (`kustomize build`/Helm render and schema/policy checks).
5. It commits a digest update, preferably as a reviewed PR for staging/production; record source commit SHA in the PR description.
6. Argo CD observes the GitOps repository; successful sync and health are separately verified in the cluster.
7. Rollback reverts the GitOps commit to a previously known-good digest; monitor actual service health.

**Why no copy-paste cross-repo mutation step here?** The correct code depends on your GitOps manifest format and automation identity. Inventing a universal cross-repository write token or `sed` replacement would be a security and correctness trap. Build the promotion PR after defining those two facts.

### 15.5 Acceptance tests

| Scenario | Expected outcome |
|---|---|
| PR with failing test | Merge blocked; no image published |
| Trusted merge to main | Tests pass; image published with commit tag and digest |
| Image scan fails | Promotion blocked; deployment not advanced |
| Update dev overlay | GitOps commit/PR contains expected immutable digest |
| Argo CD syncs | Deployment uses same digest as GitOps |
| Pod readiness fails | Rollout unhealthy; alert and rollback process invoked |
| Retry workflow | No uncontrolled duplicate production mutation |
| Stale workflow | No unintended deployment due to concurrency and protection |
| Secret missing | Clear failure; no secret values leaked into logs |

## 16. Interviews, pitfalls and mastery checklist

### 16.1 Questions a 5-YOE DevOps engineer should answer

1. How do workflow, job, step, action and runner differ?
2. Why does a downstream job not see the filesystem of an upstream job?
3. How do `needs` and `if: always()` interact when a job fails/skips?
4. Why is a 2×2 matrix four jobs? How do `include`, `exclude`, `fail-fast` and `max-parallel` differ?
5. What are the differences between repository variables, environment variables, secrets and `env`?
6. When do you use `$GITHUB_OUTPUT`, `$GITHUB_ENV`, artifacts or caches?
7. How do `push`, `pull_request` and `pull_request_target` differ in trust and permissions?
8. How do you grant `GITHUB_TOKEN` only what is required per job?
9. How do GitHub OIDC claims and AWS IAM trust conditions prevent a different repository from assuming your role?
10. Why pin third-party actions to full SHAs? What updates them?
11. Reusable workflow versus composite action versus Docker action?
12. How do you avoid deploying the wrong commit when several merges happen quickly?
13. What is an image digest and why is it stronger than a mutable tag?
14. What is the safest way for CI to update a GitOps repo owned by another team?
15. How can you prove CI succeeded **and** the running Kubernetes workloads changed?
16. Why is a long-lived self-hosted runner dangerous for untrusted PRs?
17. How do you use branch protection, environments and approvals together?
18. What are your recovery actions when a container scan service or registry is unavailable?
19. How do you reduce runner minutes without compromising deterministic builds?
20. What telemetry tells you a CD pipeline is reliable?

### 16.2 Common mistakes

| Mistake | Correction |
|---|---|
| Expect each step to appear as a graph stage | Graph displays jobs; expand job for steps |
| Forget `actions/checkout` | Explicitly check out repo before reading source |
| Treat jobs as same machine | Use outputs/artifacts to transfer data |
| Use `secrets` for normal config | Use `vars` for non-sensitive config |
| Put secret in `echo` or workflow output | Do not expose it; use scoped environment variables |
| Use `pull_request_target` to run PR tests | Use `pull_request` with restricted permissions |
| Give every job `contents: write` | Least privilege per job |
| Build image separately for each environment | Promote the same verified digest |
| Assume image tag automatically changes Kubernetes | Update desired image and check actual rollout |
| Cache build output and assume it is deployable | Artifact/registry digest is authoritative |
| Blindly cancel production deployments | Model concurrency and safe rollback explicitly |
| Store AWS access key in repository secret | Prefer OIDC and scoped role |

### 16.3 Readiness rubric

**Foundation:** You can create triggers, jobs/steps, expressions, `needs`, variables and secrets; diagnose a failing job.  
**Intermediate:** You can write matrices, integration tests, artifact passing, caches, concurrency and reusable workflows.  
**Advanced:** You understand token permissions, fork PR trust, OIDC, immutable action refs, scanning, self-hosted runner risk and environment approvals.  
**Production-ready:** You can design a reviewed PR gate, reproducible image build, immutable digest promotion, GitOps deployment, health verification, rollback, observability, least privilege and documented incident handling.

Suggested pace for hands-on mastery:

- **Week 1:** Sections 1–7 + Labs 1–7.
- **Week 2:** Sections 8–10 + Labs 8–10, 13–14.
- **Week 3:** Sections 11–13 + Labs 11–12, 15–16.
- **Week 4:** Capstone and failure drills; explain every design decision aloud as if in a senior interview.

### Official reference links

- [Workflows and basic concepts](https://docs.github.com/en/actions/concepts/workflows-and-actions/workflows)
- [Complete workflow syntax reference](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax)
- [Contexts reference](https://docs.github.com/en/actions/reference/workflows-and-actions/contexts)
- [Jobs, dependencies and matrices](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-jobs)
- [Reusable workflow configurations](https://docs.github.com/en/actions/reference/workflows-and-actions/reusing-workflow-configurations)
- [Deployment environments and protections](https://docs.github.com/en/actions/concepts/workflows-and-actions/deployment-environments)
- [Security hardening](https://docs.github.com/en/actions/reference/security/secure-use)
- [OpenID Connect](https://docs.github.com/en/actions/concepts/security/openid-connect)
- [Using `pull_request_target` safely](https://docs.github.com/en/actions/reference/security/securely-using-pull_request_target)

---

**Final exercise:** Design a pipeline from scratch for a FastAPI service with Postgres and RabbitMQ: PR lint/test/integration/security → protected merge → build and scan one image → push and capture digest → GitOps promotion PR → Argo CD sync → Kubernetes smoke checks → rollback by digest. If you can implement it safely and explain *why* each boundary exists, you have a credible GitHub Actions skill set for a 5-YOE DevOps interview.
