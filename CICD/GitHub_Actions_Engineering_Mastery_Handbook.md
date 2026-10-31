# GitHub Actions Engineering Handbook — Fundamentals to Production Mastery

> Practical, sequential, repository-ready learning path. Each lab provides **complete workflow YAML**, setup, execution, verification, failure exercise, and cleanup where relevant. Labs build on one common FastAPI repository; do not run all workflows concurrently unless you want duplicate CI runs.

## Navigation

**Advanced modules:** [Engineering foundations](#part-ii--engineering-foundations) · [Labs 16–27](#lab-16--advanced-job-outputs-and-expressions) · [Capstone](#lab-28--reference-ci-to-gitops-capstone) · [Runbooks](#operations-runbooks-and-interview-assessment)

- [Concepts and Jenkins comparison](#1-introduction)
- [Repository and prerequisites](#lab-00--shared-repository-setup)
- [Labs 01–15](#lab-01--first-workflow)
- [Production design principles](#production-design-principles)
- [Troubleshooting and references](#troubleshooting)

**Conventions:** Replace `OWNER`, `REPO`, `ACCOUNT_ID`, and other uppercase placeholders. Workflows are independently complete for their lab, but shared application files are created only once in Lab 00. For production, pin third-party actions to reviewed immutable commit SHAs rather than the readable major version tags used in these exercises. Commands assume Bash unless noted. Use a separate practice repository.

# Part I — Fundamentals and Labs 00–15

## 1. Introduction

GitHub Actions automates tasks triggered by repository events: pushes, pull requests, releases, schedules, and manual invocations. Typical pipeline: developer push → test/lint → build/scan → publish image → update deployment configuration → Argo CD sync → Kubernetes rollout.

### GitHub Actions vs Jenkins

| Feature | GitHub Actions | Jenkins |
|---|---|---|
| Definition | YAML under `.github/workflows/` | Jenkinsfile |
| Execution | Hosted or self-hosted runners | Agents |
| Setup | Minimal with hosted runners | Controller and agents required |
| Integration | Native GitHub events | Plugins/webhooks |
| Reuse | Actions/reusable workflows | Shared libraries/plugins |
| Operations | Less for hosted runners | More when self-managed |

## 2. Architecture and core concepts

- **Workflow:** YAML automation definition.
- **Event:** Trigger such as `push`, `pull_request`, `workflow_dispatch`.
- **Job:** Set of steps on one runner; jobs run concurrently unless constrained by `needs`.
- **Step:** Sequential command (`run`) or reusable action (`uses`) within a job.
- **Runner:** Hosted or self-hosted execution machine.
- **Action:** Reusable unit, e.g. checkout or language setup.

```text
Repository
└── .github/workflows/ci.yml
    ├── on: push / pull_request
    ├── job: test
    │   ├── checkout
    │   ├── install dependencies
    │   └── pytest
    └── job: build
        └── docker build
```

Each job has its own filesystem; use artifacts, a registry, or another shared store to transfer outputs between jobs.


## Lab 00 — Shared repository setup

**Goal:** Prepare the application used by Labs 05–15. Earlier labs can run without it.

```bash
mkdir -p gha-labs/{app,tests,.github/workflows,deploy/k8s}
cd gha-labs
touch app/__init__.py
cat > app/main.py <<'EOF'
from fastapi import FastAPI
app = FastAPI()

@app.get('/health')
def health():
    return {'status': 'healthy'}
EOF
cat > tests/test_api.py <<'EOF'
from fastapi.testclient import TestClient
from app.main import app

def test_health():
    response = TestClient(app).get('/health')
    assert response.status_code == 200
    assert response.json() == {'status': 'healthy'}
EOF
cat > requirements.txt <<'EOF'
fastapi>=0.115,<1
uvicorn>=0.30,<1
httpx>=0.27,<1
pytest>=8,<10
EOF
cat > Dockerfile <<'EOF'
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app/ ./app/
USER 10001
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
EOF
cat > .dockerignore <<'EOF'
.git
.github
.venv
__pycache__
.pytest_cache
reports
EOF
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python -m pytest -q
```

Create a GitHub repository, commit these files, and push to `main`. Use a dependency lockfile and pinned image digest in real deployments. To avoid accidental duplicate triggers, disable or remove a previous lab's workflow when progressing to the next.

## Lab 01 — First workflow

**Goal:** Learn runner execution, shell steps, and manual dispatch.

**Setup:** No application files needed.

Create `.github/workflows/lab-01.yml`:

```yaml
name: Lab 01 - Hello
on:
  push:
    branches: [main]
  workflow_dispatch:
permissions:
  contents: read
jobs:
  hello:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - name: Print system details
        run: |
          echo "Hello GitHub Actions"
          uname -a
          pwd
          date -u
      - name: Print repository metadata
        env:
          REPO: ${{ github.repository }}
          SHA: ${{ github.sha }}
        run: printf 'repo=%s\nsha=%s\n' "$REPO" "$SHA"
```

**Run:** Push to `main`; then open **Actions → Lab 01 - Hello → Run workflow** for a manual run.

**Verify:** Both runs succeed; metadata matches the commit.

**Failure experiment:** Change `run: date -u` to `run: exit 1`, observe a red step, and restore it.



---

## Lab 02 — Triggers and filters

**Goal:** Test push path filtering, PR events and typed manual inputs.

**Setup:** Create `src/example.txt` and commit it.

Create `.github/workflows/lab-02.yml`:

```yaml
name: Lab 02 - Triggers
on:
  push:
    branches: [main]
    paths: ['src/**']
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      target:
        description: Target environment
        type: choice
        options: [dev, staging]
        required: true
        default: dev
permissions:
  contents: read
jobs:
  inspect:
    runs-on: ubuntu-latest
    steps:
      - name: Display event
        env:
          EVENT: ${{ github.event_name }}
          REF: ${{ github.ref }}
          TARGET: ${{ inputs.target }}
        run: |
          echo "event=$EVENT"
          echo "ref=$REF"
          echo "target=${TARGET:-not-applicable}"
```

**Run:** Commit changes to `src/example.txt` on `main`, then `README.md` only; open a PR; finally dispatch manually with `staging`.

**Verify:** The src push runs, README-only push does not trigger this workflow, PR runs, and manual logs show staging.

**Failure experiment:** Change PR target to a different branch and verify it does not match. Note: a skipped required check can leave a PR pending; plan branch protection accordingly.



---

## Lab 03 — Parallel jobs and dependencies

**Goal:** Implement a test/lint/build DAG and failure propagation.

**Setup:** No application needed.

Create `.github/workflows/lab-03.yml`:

```yaml
name: Lab 03 - Job DAG
on: [workflow_dispatch]
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
      - run: echo 'building after both checks'
  summary:
    if: always()
    needs: [test, lint, build]
    runs-on: ubuntu-latest
    steps:
      - name: Show statuses
        env:
          TEST: ${{ needs.test.result }}
          LINT: ${{ needs.lint.result }}
          BUILD: ${{ needs.build.result }}
        run: echo "test=$TEST lint=$LINT build=$BUILD"
```

**Run:** Manually dispatch. Open the visualization graph.

**Verify:** `test` and `lint` are independent; `build` starts after both pass; `summary` always executes.

**Failure experiment:** Change test command to `exit 1`; `build` is skipped and summary reports results. Restore afterward.



---

## Lab 04 — Secrets, variables and conditions

**Goal:** Use GitHub contexts without leaking credentials.

**Setup:** Add repository variable `APP_NAME=order-service` and a dummy repository secret `LAB_TOKEN=example-only-not-real`.

Create `.github/workflows/lab-04.yml`:

```yaml
name: Lab 04 - Configuration
on: [workflow_dispatch]
permissions:
  contents: read
env:
  APP_NAME: ${{ vars.APP_NAME }}
jobs:
  config:
    runs-on: ubuntu-latest
    env:
      STAGE: dev
    steps:
      - name: Validate secret exists without printing
        env:
          LAB_TOKEN: ${{ secrets.LAB_TOKEN }}
        run: |
          test -n "$LAB_TOKEN"
          echo 'secret present'
      - name: Display nonsecret config
        env:
          BRANCH: ${{ github.ref_name }}
        run: echo "app=$APP_NAME stage=$STAGE branch=$BRANCH"
      - name: Only main
        if: github.ref == 'refs/heads/main'
        run: echo 'main branch action'
```

**Run:** Dispatch from `main` and inspect step outputs.

**Verify:** Only non-sensitive metadata is printed and secret-presence test passes.

**Failure experiment:** Delete the dummy secret, rerun, and confirm failure without exposing a value; recreate it.



---

## Lab 05 — Full FastAPI CI

**Goal:** Run lint, tests, JUnit reports and Docker build in dependent jobs.

**Setup:** Complete Lab 00; add `ruff` to your local tooling if desired.

Create `.github/workflows/lab-05.yml`:

```yaml
name: Lab 05 - Python CI
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
permissions:
  contents: read
jobs:
  quality:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: pip
      - name: Install
        run: |
          python -m pip install -r requirements.txt
          python -m pip install ruff
      - name: Lint
        run: ruff check app tests
      - name: Test and report
        run: |
          mkdir -p reports
          python -m pytest -v --junitxml=reports/junit.xml
      - name: Save report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: junit
          path: reports/junit.xml
          if-no-files-found: ignore
  docker-build:
    needs: quality
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t order-service:ci .
```

**Run:** Push a PR and review both jobs. Download `junit` from the run summary.

**Verify:** Test results are present and Docker build starts only after quality passes.

**Failure experiment:** Change expected test status to 201; verify quality fails, Docker build skips, and JUnit artifact remains available. Revert.



---

## Lab 06 — Docker smoke test

**Goal:** Build and run the container and verify HTTP health within CI.

**Setup:** Lab 00 Dockerfile and application required.

Create `.github/workflows/lab-06.yml`:

```yaml
name: Lab 06 - Docker Smoke
on: [workflow_dispatch]
permissions:
  contents: read
jobs:
  smoke:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - name: Build image
        run: docker build -t order-service:smoke .
      - name: Start container
        run: docker run -d --name order-service -p 127.0.0.1:8000:8000 order-service:smoke
      - name: Wait and probe
        run: |
          for i in $(seq 1 30); do
            if curl --fail --silent http://127.0.0.1:8000/health | grep -q healthy; then exit 0; fi
            sleep 1
          done
          docker logs order-service
          exit 1
      - name: Logs on failure
        if: failure()
        run: docker logs order-service
      - name: Cleanup
        if: always()
        run: docker rm -f order-service || true
```

**Run:** Dispatch and expand the probe step.

**Verify:** Probe succeeds, container is removed, and no registry is required.

**Failure experiment:** Change the endpoint in the probe to `/missing`, confirm failure/logs, then restore.



---

## Lab 07 — Publish to GitHub Container Registry

**Goal:** Authenticate, build, and push a SHA-tagged image.

**Setup:** Repository must permit package publication; `GITHUB_TOKEN` is automatic.

Create `.github/workflows/lab-07.yml`:

```yaml
name: Lab 07 - Publish GHCR
on:
  push:
    branches: [main]
  workflow_dispatch:
permissions:
  contents: read
jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - name: Normalize image name
        id: image
        env:
          REPO: ${{ github.repository }}
        run: echo "name=ghcr.io/${REPO,,}" >> "$GITHUB_OUTPUT"
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        id: build
        with:
          context: .
          push: true
          tags: ${{ steps.image.outputs.name }}:${{ github.sha }}
      - name: Record digest
        env:
          DIGEST: ${{ steps.build.outputs.digest }}
        run: echo "Published digest=$DIGEST"
```

**Run:** Dispatch or push; find the image under GitHub Packages. Pull with `docker pull ghcr.io/owner/repo:FULL_SHA` (lowercase owner/repo).

**Verify:** The package exists and the digest is logged.

**Failure experiment:** Remove `packages: write` temporarily and observe permission failure; restore. For private packages, authenticate before pulling.



---

## Lab 08 — Matrix compatibility

**Goal:** Test supported Python versions and OS combinations.

**Setup:** Lab 00 files present; use Python dependencies compatible with each target.

Create `.github/workflows/lab-08.yml`:

```yaml
name: Lab 08 - Matrix
on: [workflow_dispatch]
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
      - name: Install and test
        run: |
          python -m pip install -r requirements.txt
          python -m pytest -q
```

**Run:** Dispatch and inspect all four matrix children.

**Verify:** All four combinations pass.

**Failure experiment:** Temporarily add a Python version unsupported by a dependency and observe one failing matrix child. Restore.



---

## Lab 09 — Cross-job artifacts

**Goal:** Generate test reports in one job and consume them in another.

**Setup:** Lab 00 application present.

Create `.github/workflows/lab-09.yml`:

```yaml
name: Lab 09 - Artifact Transfer
on: [workflow_dispatch]
permissions:
  contents: read
jobs:
  produce:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: |
          pip install -r requirements.txt
          mkdir -p reports
          python -m pytest --junitxml=reports/junit.xml
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: junit-results
          path: reports/junit.xml
          retention-days: 7
          if-no-files-found: ignore
  consume:
    needs: produce
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: junit-results
          path: reports
      - run: |
          test -s reports/junit.xml
          head -c 250 reports/junit.xml
```

**Run:** Dispatch and inspect `consume` logs.

**Verify:** The second job reads the uploaded XML; no filesystem sharing is assumed.

**Failure experiment:** Change download name to `wrong-name` and verify the consumer fails; restore.



---

## Lab 10 — Security gates

**Goal:** Run Python lint, dependency audit and container vulnerability scan.

**Setup:** Lab 00 present. Scanner results depend on changing vulnerability databases; review findings before deciding policy.

Create `.github/workflows/lab-10.yml`:

```yaml
name: Lab 10 - Security
on:
  pull_request:
    branches: [main]
  workflow_dispatch:
permissions:
  contents: read
jobs:
  scan:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - name: Python checks
        run: |
          pip install -r requirements.txt ruff pip-audit
          ruff check app tests
          pip-audit -r requirements.txt
      - name: Build image
        run: docker build -t order-service:scan .
      - name: Install Trivy
        run: |
          sudo apt-get update
          sudo apt-get install -y wget
          wget -qO /tmp/trivy.deb https://github.com/aquasecurity/trivy/releases/download/v0.67.2/trivy_0.67.2_Linux-64bit.deb
          sudo dpkg -i /tmp/trivy.deb
      - name: Scan container
        run: trivy image --exit-code 1 --severity CRITICAL --ignore-unfixed order-service:scan
```

**Run:** Dispatch and inspect audit and scanner logs. Trivy installation example is version-specific; update and verify package/checksum for production, or use a pinned trusted Trivy action.

**Verify:** Pipeline succeeds only if checks pass; failures require investigation, not blanket suppression.

**Failure experiment:** Add an unused variable or invalid Python syntax and verify lint catches it; restore. Scanner may legitimately fail because of base-image CVEs.



---

## Lab 11 — Reusable workflows

**Goal:** Define a callable workflow and a separate caller.

**Setup:** Two YAML files are required; the caller shown below is the complete pipeline, and the reusable definition is included immediately afterward.

Create `.github/workflows/lab-11.yml`:

```yaml
name: Lab 11 - Caller
on: [workflow_dispatch]
permissions:
  contents: read
jobs:
  test:
    uses: ./.github/workflows/reusable-python.yml
    with:
      python-version: '3.12'
```

**Run:** Create the reusable file below, commit both, then manually dispatch the caller.

**Verify:** The reusable job executes and runs pytest.

**Failure experiment:** Change the caller input to `3.11` and verify the interpreter changes.

Create `.github/workflows/reusable-python.yml`:

```yaml
name: Reusable Python Tests
on:
  workflow_call:
    inputs:
      python-version:
        type: string
        required: true
permissions:
  contents: read
jobs:
  tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ inputs.python-version }}
      - run: |
          pip install -r requirements.txt
          python -m pytest -q
```

Reusable workflows are called at the **job** level; composite actions are called at the **step** level.


---

## Lab 12 — AWS OIDC role assumption

**Goal:** Exchange GitHub OIDC token for short-lived AWS role credentials.

**Setup:** Create an IAM OIDC provider for `https://token.actions.githubusercontent.com` with audience `sts.amazonaws.com`; create a role with the trust policy below. Set repository variable `AWS_ROLE_ARN` to its ARN.

Create `.github/workflows/lab-12.yml`:

```yaml
name: Lab 12 - AWS OIDC
on: [workflow_dispatch]
permissions:
  contents: read
jobs:
  verify-identity:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ap-south-1
      - name: Verify AWS identity
        run: aws sts get-caller-identity
```

**Run:** Dispatch from `main`.

**Verify:** Output shows the expected assumed IAM role, without static AWS access keys.

**Failure experiment:** Attempt to run from a branch not permitted by the role trust policy; it should fail.

**Example IAM role trust policy** (replace account ID, GitHub owner/repository; restrict to `main`):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"},
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
        "token.actions.githubusercontent.com:sub": "repo:OWNER/REPO:ref:refs/heads/main"
      }
    }
  }]
}
```

Attach only the AWS permissions needed by each lab. If you use GitHub environments, the OIDC subject format changes; update the trust policy accordingly.


---

## Lab 13 — Push image to Amazon ECR

**Goal:** Use OIDC and ECR credentials to publish the same application image.

**Setup:** Complete Lab 12. Create ECR repository `order-service` in `ap-south-1`. Add repository variable `AWS_ACCOUNT_ID`. Role needs `ecr:GetAuthorizationToken` on `*` and ECR upload permissions on the specific repository (see note).

Create `.github/workflows/lab-13.yml`:

```yaml
name: Lab 13 - ECR Publish
on: [workflow_dispatch]
permissions:
  contents: read
jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ap-south-1
      - name: Login to ECR
        env:
          ACCOUNT_ID: ${{ vars.AWS_ACCOUNT_ID }}
        run: |
          aws ecr get-login-password --region ap-south-1 |             docker login --username AWS --password-stdin             "$ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com"
      - name: Build and push
        env:
          ACCOUNT_ID: ${{ vars.AWS_ACCOUNT_ID }}
          SHA: ${{ github.sha }}
        run: |
          IMAGE="$ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/order-service:$SHA"
          docker build -t "$IMAGE" .
          docker push "$IMAGE"
          aws ecr describe-images --repository-name order-service             --image-ids imageTag="$SHA" --region ap-south-1
```

**Run:** Dispatch from the trusted branch.

**Verify:** Image digest is visible in ECR and `describe-images` succeeds.

**Failure experiment:** Temporarily remove ECR upload permissions and verify the push is denied; restore. Delete test images when finished.

**IAM permissions:** `ecr:BatchCheckLayerAvailability`, `ecr:CompleteLayerUpload`, `ecr:InitiateLayerUpload`, `ecr:PutImage`, and `ecr:UploadLayerPart` scoped to `arn:aws:ecr:ap-south-1:ACCOUNT_ID:repository/order-service`, plus `ecr:GetAuthorizationToken` on `*`. Review current AWS ECR guidance before deployment. ECR storage and scanning may incur charges.


---

## Lab 14 — GitOps with kind and Argo CD

**Goal:** Update a separate GitOps repository via PR, then let Argo CD reconcile the cluster.

**Setup:** Create a second repository `OWNER/order-gitops`; configure Argo CD to track its `main` branch. Store a GitHub fine-grained token with **contents and pull requests write** on that GitOps repo as `GITOPS_TOKEN` in the application repo; use a GitHub App token instead for stronger production controls.

Create `.github/workflows/lab-14.yml`:

```yaml
name: Lab 14 - GitOps PR
on: [workflow_dispatch]
permissions:
  contents: read
jobs:
  propose-deployment:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Checkout GitOps repository
        uses: actions/checkout@v4
        with:
          repository: OWNER/order-gitops
          token: ${{ secrets.GITOPS_TOKEN }}
          path: gitops
          persist-credentials: false
      - name: Update image tag
        env:
          IMAGE_TAG: ${{ github.sha }}
        run: |
          cd gitops
          sed -i "s|^  tag:.*|  tag: "$IMAGE_TAG"|" values.yaml
          git diff -- values.yaml
      - name: Create deployment PR
        env:
          GH_TOKEN: ${{ secrets.GITOPS_TOKEN }}
        run: |
          cd gitops
          BRANCH="deploy/${GITHUB_RUN_ID}"
          git config user.name 'github-actions[bot]'
          git config user.email '41898282+github-actions[bot]@users.noreply.github.com'
          git checkout -b "$BRANCH"
          git add values.yaml
          git commit -m "Promote image ${GITHUB_SHA}" || exit 0
          git -c http.extraheader="AUTHORIZATION: basic $(printf 'x-access-token:%s' "$GH_TOKEN" | base64 -w0)" push             https://github.com/OWNER/order-gitops.git "$BRANCH"
          gh pr create --repo OWNER/order-gitops --head "$BRANCH"             --base main --title "Deploy ${GITHUB_SHA}"             --body 'Automated image promotion; review before merge.'
```

**Run:** First publish an image tagged with this commit SHA using Lab 07, then dispatch Lab 14. Replace `OWNER` in the workflow with your account/org. Merge the resulting GitOps PR.

**Verify:** `kubectl get applications -n argocd` shows Synced/Healthy; `kubectl rollout status deployment/order-service -n order-service` succeeds.

**Failure experiment:** Revert the GitOps change to the prior known-good image and confirm rollback.

**GitOps repository contents:**

`values.yaml`:
```yaml
image:
  repository: ghcr.io/owner/gha-labs
  tag: "REPLACE_WITH_PUBLISHED_SHA"
```

`templates/deployment.yaml` (inside a minimal Helm chart with `Chart.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 1
  selector:
    matchLabels: {app: order-service}
  template:
    metadata:
      labels: {app: order-service}
    spec:
      containers:
        - name: api
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 8000
          readinessProbe:
            httpGet: {path: /health, port: 8000}
            initialDelaySeconds: 2
            periodSeconds: 5
```

`Chart.yaml`:
```yaml
apiVersion: v2
name: order-service
version: 0.1.0
```

Argo CD `Application` (apply after installing Argo CD; set `repoURL` to your actual GitOps repository):
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/OWNER/order-gitops.git
    targetRevision: main
    path: .
    helm:
      valueFiles: [values.yaml]
  destination:
    server: https://kubernetes.default.svc
    namespace: order-service
  syncPolicy:
    automated: {prune: true, selfHeal: true}
    syncOptions: [CreateNamespace=true]
```

**Important:** If GHCR package is private, configure Kubernetes imagePullSecrets. The GitOps PR updates a SHA **tag**; for immutable production deployments, update to a registry digest instead. The workflow deliberately uses a separate token for cross-repository PR creation. GitHub Actions must not be given direct cluster credentials for this model.


---

## Lab 15 — Protected release promotion

**Goal:** Build a gated workflow with checks, staging, and production approval.

**Setup:** Configure GitHub `staging` and `production` environments. Enable required reviewers for production where your plan/repository supports them. This lab models approval and promotion; actual deployment should update the GitOps repository using the reviewed mechanism in Lab 14.

Create `.github/workflows/lab-15.yml`:

```yaml
name: Lab 15 - Release Gate
on:
  workflow_dispatch:
permissions:
  contents: read
concurrency:
  group: release-${{ github.ref }}
  cancel-in-progress: false
jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: |
          pip install -r requirements.txt
          python -m pytest -q
      - run: docker build -t order-service:${{ github.sha }} .
  staging:
    needs: verify
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Stage verified release
        env:
          SHA: ${{ github.sha }}
        run: echo "Stage immutable release $SHA via GitOps PR"
  production:
    needs: staging
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Promote verified release
        env:
          SHA: ${{ github.sha }}
        run: echo "Promote same release $SHA via approved GitOps PR"
```

**Run:** Dispatch on `main`; inspect `verify`, staging, and the production environment approval gate.

**Verify:** Production job waits for the configured approval and executes only after upstream jobs pass.

**Failure experiment:** Make pytest fail; verify neither staging nor production executes. Note: the deployment commands above intentionally print only; connect the actual PR creation logic from Lab 14 after setting appropriate approvals.

**Production implementation checklist:** publish a single image once, record its digest, perform scans and provenance checks, deploy that exact digest to staging, smoke-test staging, require production approval, update the production GitOps overlay through reviewed PR, watch Argo CD health and rollout, and revert the GitOps commit for rollback. The workflow above demonstrates the **control gates**, not a fully automated production deployment.


---

## Production design principles

1. Keep CI and CD separate: GitHub Actions builds/tests/publishes; Argo CD reconciles Git state to Kubernetes.
2. Prefer OIDC over long-lived AWS keys; scope IAM roles to exact repository/branch or environment.
3. Restrict `GITHUB_TOKEN` at workflow and job level; avoid elevated tokens on untrusted PR code.
4. Pin action revisions and image digests; review third-party action maintainers and dependencies.
5. Prefer locked dependencies, SBOM, provenance/attestations, security scans and reviewable exceptions.
6. Use branch protections, required checks, environment reviewers and deployment concurrency controls.
7. Promote the *same image digest* between staging and production; never rebuild for promotion.
8. Set timeouts, retention policies and resource cleanup; watch GitHub Actions minutes and AWS costs.
9. Keep self-hosted runners isolated from untrusted contributions and privileged networks.
10. Test rollback, health checks and failure paths, not only successful deployment.

## Troubleshooting

| Symptom | Likely cause | Check |
|---|---|---|
| Workflow does not trigger | Wrong branch/path/event | YAML `on`, branch/path filters, default branch |
| Job skipped | `needs` failed or `if` false | Job graph and dependency results |
| `Resource not accessible by integration` | Missing token permission | `permissions` and repo/org policies |
| `pip` dependencies conflict | Unlocked versions | Pin compatible versions and inspect resolver |
| GHCR push denied | Missing `packages: write` or package policy | Token and package permissions |
| ECR login/push denied | OIDC trust or IAM ECR permissions | `sts get-caller-identity`, trust `sub`/`aud`, ECR repo |
| Argo CD Synced but old image | Tag not updated, cached tag, wrong overlay | Git revision, image tag/digest, live Deployment |
| Kubernetes image pull fails | Private registry or wrong image | `kubectl describe pod`, pull secrets |
| Production approval missing | Environment rules unavailable/not configured | Repo environment protection settings |
| Security scan red | Real CVE or scanner/db issue | Scanner output and policy, avoid blanket bypass |

## Official references

- [GitHub Actions documentation](https://docs.github.com/en/actions)
- [Workflow syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax)
- [Reusable workflows](https://docs.github.com/en/actions/how-tos/sharing-automations/reuse-workflows)
- [Security hardening](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions)
- [AWS OIDC](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws)
- [Argo CD](https://argo-cd.readthedocs.io/en/stable/)

## Completion checklist

- [ ] Lab 00
- [ ] Lab 01
- [ ] Lab 02
- [ ] Lab 03
- [ ] Lab 04
- [ ] Lab 05
- [ ] Lab 06
- [ ] Lab 07
- [ ] Lab 08
- [ ] Lab 09
- [ ] Lab 10
- [ ] Lab 11
- [ ] Lab 12
- [ ] Lab 13
- [ ] Lab 14
- [ ] Lab 15



---

# Part II — Engineering foundations

> **Scope and rigor.** Labs 00–15 form the introductory practical course. This second part is additive: it does not repeat their installation instructions. It concentrates on architectural choices, runnable advanced pipelines, adversarial failure tests, and operational acceptance criteria. Commands use Bash, GitHub CLI (`gh`), Docker, `kubectl`, Helm, and Python where applicable. GitHub features, runner images, action versions, billing, and security policies evolve: inspect the official links at the end before using these patterns in production. Replace `OWNER`, `REPO`, `ROLE_ARN`, `ACCOUNT_ID`, and other explicit placeholders. Never use production cloud credentials in a practice repository.

## A. Runtime mental model and limits

A workflow is an event-triggered DAG of jobs. The workflow YAML is *control-plane configuration*, not a shell script. Each job gets a runner; steps in a job share its workspace, but jobs do not share writable filesystems. A job's `needs` relationship governs scheduling and skip propagation. Outputs carry small strings; artifacts move files; registries move images; caches only accelerate repeated work and are not an integrity-preserving artifact store.

```text
Event -> Workflow evaluation -> Job graph -> Runner allocation
                                          ├── job test -> steps in order
                                          ├── job lint -> steps in order
                                          └── job package (needs test,lint)
                                                  -> upload artifact / publish image
```

| Mechanism | Best for | Do not use as |
|---|---|---|
| `GITHUB_OUTPUT` / `needs.*.outputs` | Small data: tag, digest, version | Secret storage or binary payload |
| Artifact | Test reports, small compiled outputs, SBOM | Long-term software registry |
| Cache | Dependency caches, compiler cache | Trusted cross-trust-boundary transfer |
| Registry | Versioned deployable images | Secrets |
| Environment | Deployment control, approvals, scoped secrets | Mutual exclusion without concurrency |
| Concurrency | Limiting overlap by group | FIFO deployment queue |

## B. Shell, expressions, inputs and injection boundaries

Default shell matters: on Linux/macOS `run` normally uses Bash; PowerShell is normal for Windows; specifying `shell: bash` improves predictability where available. Each `run` step starts a new process: `cd` and `export` do not persist to the next step. Use `GITHUB_ENV` to persist environment variables to *later steps in the same job* and `GITHUB_PATH` for PATH changes. Use outputs for deliberate data exchange between jobs.

**Do not splice attacker-controlled expressions into a shell script.** This is unsafe:

```yaml
# WRONG: PR title could become shell syntax
- run: echo "${{ github.event.pull_request.title }}"
```

Safer pattern: put potentially untrusted text in an environment variable, then quote it in Bash:

```yaml
- name: Print title safely
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: printf '%s\n' "$PR_TITLE"
```

This *avoids direct expression-to-script injection*, but never trust an untrusted value as a filename, command, URL, artifact, or deployment target without validation. Avoid `eval`; use arrays instead of constructing shell commands as strings.

## C. Runner trust classes and deployment boundaries

- **Untrusted code:** Fork PRs, public contributions, or unknown workflow dependencies. Run with minimal token rights and no privileged secrets or production network access.
- **Trusted CI:** Verified repository branch, read-only source access, bounded build rights, ephemeral runners. A main-branch commit is not automatically immune to supply-chain compromise.
- **Privileged publish:** Protected branch/tag, `packages: write`, registry credentials only where necessary.
- **Production promotion:** Approved environment, branch restrictions, narrow cloud IAM, digest-pinned artifact, audit trail.

**Self-hosted runner warning:** Treat repository code as code executing on the machine. A long-lived runner can accumulate state across jobs; never put an Internet-facing untrusted PR workload on a runner with cloud credentials, Docker host socket, accessible production networks or persistent personal home-directory secrets. Prefer ephemeral isolated runners for risky workloads.

## D. Practical verification commands

```bash
# Install GitHub CLI using official docs for your OS, authenticate locally:
gh auth login
# Validate a repo's workflow directory quickly:
find .github/workflows -name '*.yml' -o -name '*.yaml'
# List / inspect workflow runs:
gh workflow list
gh run list --limit 10
gh run view RUN_ID --log-failed
# Manual dispatch (workflow must support workflow_dispatch):
gh workflow run lab-16.yml
# Inspect action versions and PR diff in code review:
git diff -- .github/workflows
```

Use **actionlint** locally for static validation (download it from its official project with checksum verification), and use **yamllint** for style checks; neither substitutes for an actual GitHub execution. Use `gh run watch RUN_ID` and run annotations to diagnose failures.

---

# Part III — Advanced labs (16–28)

Each lab contains *objective, prerequisites, complete code, execution, verification, failure injection and cleanup*. Create these files one at a time in the same practice repository from Lab 00. To prevent duplicate runs, these labs use `workflow_dispatch` unless they specifically test events. Workflow names are unique. On the Actions page, manually dispatch the named workflow on a branch containing its file; for reusable workflows, ensure their files have been committed.

## Lab 16 — Advanced job outputs and expressions

**Objective:** Generate a deterministic release candidate, transmit it across jobs, and conditionally execute downstream verification.

**Create** `.github/workflows/lab-16.yml`:

```yaml
name: Lab 16 - Outputs and Decisions
on:
  workflow_dispatch:
    inputs:
      deploy:
        type: boolean
        description: 'Enable simulated deployment'
        required: true
        default: false
permissions:
  contents: read
jobs:
  calculate:
    runs-on: ubuntu-latest
    outputs:
      short_sha: ${{ steps.version.outputs.short_sha }}
      candidate: ${{ steps.version.outputs.candidate }}
    steps:
      - id: version
        shell: bash
        run: |
          short="${GITHUB_SHA::12}"
          echo "short_sha=$short" >> "$GITHUB_OUTPUT"
          echo "candidate=rc-$short" >> "$GITHUB_OUTPUT"
  verify:
    needs: calculate
    runs-on: ubuntu-latest
    steps:
      - name: Assert format
        env:
          CANDIDATE: ${{ needs.calculate.outputs.candidate }}
        run: |
          [[ "$CANDIDATE" =~ ^rc-[a-f0-9]{12}$ ]]
          echo "Candidate validated: $CANDIDATE"
  deploy-simulation:
    needs: [calculate, verify]
    if: ${{ inputs.deploy }}
    runs-on: ubuntu-latest
    steps:
      - env:
          CANDIDATE: ${{ needs.calculate.outputs.candidate }}
        run: echo "Would deploy $CANDIDATE"
```

**Run:** `gh workflow run lab-16.yml -f deploy=true`. **Verify:** Calculate → Verify → Simulated deploy; repeat with `deploy=false` and observe skip. **Break:** Change candidate prefix to `bad-`, confirm assertion fails and dependency-based skip. **Cleanup:** Restore prefix.

## Lab 17 — Dynamic matrix from validated JSON

**Objective:** Generate a runtime matrix safely and understand `fromJSON`.

**Create** `.github/workflows/lab-17.yml`:

```yaml
name: Lab 17 - Dynamic Matrix
on: workflow_dispatch
permissions:
  contents: read
jobs:
  plan:
    runs-on: ubuntu-latest
    outputs:
      versions: ${{ steps.generate.outputs.versions }}
    steps:
      - id: generate
        shell: bash
        run: |
          python3 - <<'PY' >> "$GITHUB_OUTPUT"
          import json
          versions=['3.11','3.12','3.13']
          assert all(v.startswith('3.') for v in versions)
          print('versions='+json.dumps(versions,separators=(',',':')))
          PY
  test:
    needs: plan
    strategy:
      fail-fast: false
      matrix:
        python-version: ${{ fromJSON(needs.plan.outputs.versions) }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: |
          python -m pip install -r requirements.txt
          python -m pytest -q
```

**Verify:** Three jobs, each with its own Python runtime. **Break:** Emit invalid JSON and confirm the matrix evaluation fails before scheduling; restore. **Production note:** Matrix explosion increases billable runner time; impose bounds on dynamic inputs.

## Lab 18 — Composite action with testable behavior

**Objective:** Factor a *sequence of steps* into one reusable composite action; contrast with Lab 11's reusable workflow (which factors complete jobs).

**Create** `.github/actions/python-tests/action.yml`:

```yaml
name: 'Run Python tests'
description: 'Set up Python, install dependencies and run tests'
inputs:
  python-version:
    description: 'Python runtime'
    required: false
    default: '3.12'
runs:
  using: composite
  steps:
    - uses: actions/setup-python@v5
      with:
        python-version: ${{ inputs.python-version }}
        cache: pip
    - shell: bash
      run: |
        python -m pip install -r requirements.txt
        python -m pytest -q
```

**Create** `.github/workflows/lab-18.yml`:

```yaml
name: Lab 18 - Composite Action
on: workflow_dispatch
permissions:
  contents: read
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/python-tests
        with:
          python-version: '3.12'
```

**Verify:** Checkout happens in caller; composite steps execute. **Break:** Move checkout after the composite and see missing local action; restore. **Design rule:** Composite actions are step-level reuse, not job runners or environment boundaries.

## Lab 19 — Reusable workflows with contracts and outputs

**Objective:** Build a reusable workflow with typed inputs and a tested output passed to another job.

**Create** `.github/workflows/reusable-quality.yml`:

```yaml
name: Reusable Quality Gate
on:
  workflow_call:
    inputs:
      python-version:
        required: true
        type: string
    outputs:
      quality_result:
        description: 'Result of quality gate'
        value: ${{ jobs.quality.outputs.result }}
permissions:
  contents: read
jobs:
  quality:
    runs-on: ubuntu-latest
    outputs:
      result: ${{ steps.status.outputs.result }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ inputs.python-version }}
      - run: |
          python -m pip install -r requirements.txt
          python -m pytest -q
      - id: status
        run: echo 'result=passed' >> "$GITHUB_OUTPUT"
```

**Create** `.github/workflows/lab-19.yml`:

```yaml
name: Lab 19 - Reusable Contract
on: workflow_dispatch
permissions:
  contents: read
jobs:
  quality:
    uses: ./.github/workflows/reusable-quality.yml
    with:
      python-version: '3.12'
  report:
    needs: quality
    runs-on: ubuntu-latest
    steps:
      - env:
          QUALITY: ${{ needs.quality.outputs.quality_result }}
        run: |
          test "$QUALITY" = passed
          echo 'Quality contract satisfied'
```

**Verify:** Output resolves in `report`. **Break:** Change output name only in caller; see failure. **Security:** A called workflow does not magically receive more privileges than the caller; explicitly design permissions and secrets at workflow boundaries. Prefer versioned, SHA-pinned cross-repository references in enterprise usage.

## Lab 20 — Artifact integrity across isolated jobs

**Objective:** Prove jobs cannot share their workspaces, and use artifacts with a checksum for a controlled handoff.

**Create** `.github/workflows/lab-20.yml`:

```yaml
name: Lab 20 - Artifact Handoff
on: workflow_dispatch
permissions:
  contents: read
jobs:
  produce:
    runs-on: ubuntu-latest
    steps:
      - run: |
          mkdir -p output
          printf '%s\n' "${GITHUB_SHA}" > output/revision.txt
          (cd output && sha256sum revision.txt > SHA256SUMS)
      - uses: actions/upload-artifact@v4
        with:
          name: revision-${{ github.run_id }}
          path: output/
          if-no-files-found: error
          retention-days: 3
  consume:
    needs: produce
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: revision-${{ github.run_id }}
          path: output/
      - shell: bash
        run: |
          cd output
          sha256sum -c SHA256SUMS
          test "$(cat revision.txt)" = "$GITHUB_SHA"
```

**Verify:** Digest check succeeds. **Break:** Add `sed -i 's/./x/' output/revision.txt` before verification; see checksum failure. Restore. **Caution:** A checksum stored alongside an artifact detects accidental mutation, but does not independently establish trusted provenance. Use signing/attestations and trusted builders for stronger assurance.

## Lab 21 — Concurrency and deployment locking

**Objective:** Observe that concurrency groups are not a durable FIFO queue and that environment rules are separate.

**Create** `.github/workflows/lab-21.yml`:

```yaml
name: Lab 21 - Concurrency
on:
  workflow_dispatch:
    inputs:
      target:
        description: 'Simulated environment'
        type: choice
        options: [staging, production]
        default: staging
permissions:
  contents: read
jobs:
  simulate:
    runs-on: ubuntu-latest
    concurrency:
      group: lab21-${{ inputs.target }}
      cancel-in-progress: false
    steps:
      - run: |
          echo "Target: ${{ inputs.target }}"
          sleep 45
          echo Done
```

**Run:** Dispatch three runs targeting staging in close succession. **Verify:** At most one running and one pending within the group; new pending runs may replace old pending runs. Dispatch production and see different group. **Break:** Set `cancel-in-progress: true` and inspect cancellation. **Important:** This is a *simulation*, not production protection. Required reviewers are configured in the repository environment settings, then referenced with `environment: production`.

## Lab 22 — Release metadata and immutable promotion

**Objective:** Normalize an image identity and generate a deployable manifest recording the image *digest* rather than a moving tag.

**Prerequisite:** Published image in GHCR, with public visibility for this lab. Supply exact digest `sha256:...` as manual input after inspecting the registry.

**Create** `.github/workflows/lab-22.yml`:

```yaml
name: Lab 22 - Immutable Release Manifest
on:
  workflow_dispatch:
    inputs:
      image:
        description: 'Full image reference, e.g. ghcr.io/owner/repo@sha256:...'
        type: string
        required: true
permissions:
  contents: read
jobs:
  manifest:
    runs-on: ubuntu-latest
    steps:
      - name: Validate immutable reference
        env:
          IMAGE: ${{ inputs.image }}
        shell: bash
        run: |
          [[ "$IMAGE" =~ ^ghcr\.io/[a-z0-9_.-]+/[a-z0-9_.-]+@sha256:[a-f0-9]{64}$ ]] || exit 1
          mkdir -p release
          printf 'image: "%s"\nsource_commit: "%s"\n' "$IMAGE" "$GITHUB_SHA" > release/manifest.yaml
      - uses: actions/upload-artifact@v4
        with:
          name: immutable-release
          path: release/manifest.yaml
```

**Verify:** Artifact contains digest and source commit. **Break:** Pass `:latest` instead of `@sha256:...` and confirm rejection. **Important:** This lab validates *syntax*, not whether the digest corresponds to an approved, scanned image. Promotion policy must verify provenance and authorization.

## Lab 23 — Container service integration tests (PostgreSQL)

**Objective:** Launch a database service container on the runner and test an actual client connection. This is independent from the minimal FastAPI code in Lab 00.

**Create** `.github/workflows/lab-23.yml`:

```yaml
name: Lab 23 - PostgreSQL Integration
on: workflow_dispatch
permissions:
  contents: read
jobs:
  integration:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: app
          POSTGRES_PASSWORD: integration-only
          POSTGRES_DB: appdb
        ports: ['5432:5432']
        options: >-
          --health-cmd 'pg_isready -U app -d appdb'
          --health-interval 5s
          --health-timeout 5s
          --health-retries 10
    env:
      PGPASSWORD: integration-only
      PGHOST: 127.0.0.1
      PGPORT: '5432'
      PGUSER: app
      PGDATABASE: appdb
    steps:
      - name: Install client
        run: |
          sudo apt-get update
          sudo apt-get install -y postgresql-client
      - name: Test SQL lifecycle
        run: |
          psql -v ON_ERROR_STOP=1 -c 'CREATE TABLE IF NOT EXISTS items (id INT PRIMARY KEY, name TEXT NOT NULL)'
          psql -v ON_ERROR_STOP=1 -c "INSERT INTO items VALUES (1, 'demo')"
          test "$(psql -Atqc 'SELECT name FROM items WHERE id=1')" = demo
```

**Verify:** DB healthy and `psql` query passes. **Break:** Change DB name in job `PGDATABASE` to missing database and observe connection failure. **Cleanup:** Restore original; services are cleaned up with hosted runner lifecycle. **Production:** Prefer migration tests and isolated test databases; never reuse a live production database.

## Lab 24 — Self-hosted runner on Debian: installation and containment

**Objective:** Understand runner lifecycle on your home Debian machine. This is an operational lab; it intentionally does **not** supply an automatically copied, possibly stale runner binary download URL or registration token. Obtain both from GitHub's UI for the specific repository.

**Prerequisites:** Dedicated low-privilege OS account, SSH access to your Debian host, and an isolated practice repository that you control. For sensitive workloads, use a disposable VM rather than the host running personal services, Jenkins, Docker, Argo CD and kind.

**Procedure:**

1. In GitHub: repository → **Settings → Actions → Runners → New self-hosted runner**. Choose Linux x64 and copy the generated, architecture-specific install commands and *short-lived* registration command. Do not publish registration tokens.
2. Create a dedicated runner VM or Unix user; download and verify the runner using GitHub's displayed checksum; execute the registration command there with a distinct runner name and labels such as `lab-runner`. Never register as root.
3. First run in the foreground (`./run.sh`) to observe it; then stop and optionally configure the systemd service using the runner's provided service script on an appropriate dedicated host.
4. Place it in an access-controlled runner group if your GitHub plan/organization supports it; restrict which repositories can dispatch to that group.
5. After the exercise, remove the runner from GitHub and uninstall the service on the host. Remove its work directory and any temporary state.

**Create** `.github/workflows/lab-24.yml` (only dispatch to your self-hosted runner after registration):

```yaml
name: Lab 24 - Self Hosted Diagnostics
on: workflow_dispatch
permissions:
  contents: read
jobs:
  diagnostics:
    runs-on: [self-hosted, linux, x64, lab-runner]
    timeout-minutes: 5
    steps:
      - run: |
          whoami
          uname -a
          df -h .
          free -h
          echo "RUNNER_NAME=$RUNNER_NAME"
```

**Verify:** Job lands on the expected runner; confirm OS identity is not root. **Break:** Disconnect the runner; the job queues rather than silently switching to hosted. Cancel the run and reconnect. **Critical:** Do not run arbitrary public PR code on this runner; do not expose the Docker socket or production kubeconfig. This lab does not prove strong isolation.

## Lab 25 — OIDC trust-policy testing and least privilege

**Objective:** Harden the basic AWS OIDC configuration from Lab 12 with a branch-bound trust policy. This is a *template* for one repository; configure it through audited IaC in serious use.

**AWS IAM trust policy** (`trust-policy.json`, replace uppercase placeholders):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:OWNER/REPO:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

**Create** `.github/workflows/lab-25.yml`:

```yaml
name: Lab 25 - OIDC Trust Verification
on: workflow_dispatch
permissions:
  contents: read
jobs:
  identity:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-region: ap-south-1
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
      - name: Assert role identity
        env:
          EXPECTED_ROLE: ${{ vars.AWS_ROLE_NAME }}
        run: |
          aws sts get-caller-identity
          test -n "$EXPECTED_ROLE"
          arn="$(aws sts get-caller-identity --query Arn --output text)"
          [[ "$arn" == *":assumed-role/${EXPECTED_ROLE}/"* ]]
```

**Run:** Dispatch from `main`. **Verify:** Role ARN matches. **Break:** Dispatch from a non-main branch with this workflow present; expect AWS STS `AccessDenied` due to `sub` restriction. **Cost:** STS identity check is usually negligible, but verify AWS account usage. **Caveat:** If you use GitHub environments, the `sub` format can change to environment-based claims; do not copy this branch-bound policy without reconciling claims and protection rules.

## Lab 26 — Terraform plan CI with no automatic apply

**Objective:** Validate Terraform in PR-like CI and keep write credentials out of untrusted jobs. This lab uses only local/offline configuration with no provider resources, so it does not incur cloud charges.

**Create** `infra/main.tf`:

```hcl
terraform {
  required_version = ">= 1.6.0"
}

variable "environment" {
  type    = string
  default = "dev"
}

output "environment" {
  value = var.environment
}
```

**Create** `.github/workflows/lab-26.yml`:

```yaml
name: Lab 26 - Terraform Validation
on:
  workflow_dispatch:
  pull_request:
    paths:
      - 'infra/**'
      - '.github/workflows/lab-26.yml'
permissions:
  contents: read
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: '1.10.5'
      - name: Format
        run: terraform -chdir=infra fmt -check
      - name: Init without backend
        run: terraform -chdir=infra init -backend=false -input=false
      - name: Validate
        run: terraform -chdir=infra validate
      - name: Plan local-only configuration
        run: terraform -chdir=infra plan -input=false -no-color
```

**Verify:** Run manually; open PR modifying `infra/main.tf`. **Break:** Format the HCL poorly or refer to undefined variables; see format/validation errors. **Production:** Plans for real cloud resources require constrained read permissions and careful handling of state and plan files, which can contain sensitive values; separate `apply` with gated workflow and state locking.

## Lab 27 — Failure diagnostics and retention

**Objective:** Capture a debug bundle *even when a step fails*, and surface a concise run summary.

**Create** `.github/workflows/lab-27.yml`:

```yaml
name: Lab 27 - Failure Diagnostics
on:
  workflow_dispatch:
    inputs:
      fail:
        type: boolean
        default: true
permissions:
  contents: read
jobs:
  diagnose:
    runs-on: ubuntu-latest
    steps:
      - name: Run controlled test
        env:
          SHOULD_FAIL: ${{ inputs.fail }}
        shell: bash
        run: |
          mkdir -p diagnostics
          echo 'test started' > diagnostics/run.log
          if [[ "$SHOULD_FAIL" == true ]]; then
            echo 'simulated failure' >> diagnostics/run.log
            exit 7
          fi
          echo 'test passed' >> diagnostics/run.log
      - name: Write summary
        if: always()
        run: |
          echo '## Diagnostics' >> "$GITHUB_STEP_SUMMARY"
          echo "- Run: $GITHUB_RUN_ID" >> "$GITHUB_STEP_SUMMARY"
          echo "- Job: $GITHUB_JOB" >> "$GITHUB_STEP_SUMMARY"
      - name: Upload diagnostics
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: diagnostics-${{ github.run_id }}
          path: diagnostics/
          retention-days: 2
          if-no-files-found: warn
```

**Verify:** Set `fail=true`; job fails yet summary and artifact appear. Set `fail=false`; job succeeds. **Production:** Never dump environment variables, tokens, kubeconfigs, credentials, core dumps, or complete HTTP auth headers into artifacts. Redact logs, control access and retention.

---

## Lab 28 — Reference CI-to-GitOps capstone

**Objective:** Build a single nonduplicated end-to-end design that validates code, publishes a testable Docker image, captures a canonical image digest, and opens a GitOps pull request for human-reviewed promotion. This capstone deliberately does **not** commit to the GitOps default branch and does **not** grant a cloud or Kubernetes admin token to CI.

**Architecture:**

```text
App repo --(PR)--> tests + Docker build smoke test (read-only)
        --(merge to main)--> tests --> GHCR image digest
                                         |
                                         v
                             GitOps branch + pull request
                                         |
                                  review / merge
                                         |
                           Argo CD (pulls from GitOps)
                                         |
                            Kubernetes rollout health
```

**Repository prerequisites:**

1. Application repository containing Lab 00's `app/`, `tests/`, `Dockerfile`, and `requirements.txt`.
2. Separate `OWNER/order-gitops` repository with a *single* `helm/order-service` chart defined below.
3. Create a GitHub App installation token or a fine-grained PAT in `GITOPS_TOKEN`, with only `contents:write` and `pull_requests:write` on the GitOps repository, and ensure the token can create PRs. Use protected branches and review policies there. For enterprise use, mint short-lived GitHub App installation tokens instead of PATs.
4. GHCR configured for package visibility/access; for this walkthrough use a public image (otherwise configure an image pull secret in Kubernetes). For registry publication, `GITHUB_TOKEN` must have `packages: write` and necessary package access.
5. Argo CD installed with access to GitOps repository and Kubernetes; local kind clusters are fine. Restrict application project destinations and source repositories in serious deployments.
6. Configure an `app-staging` or `production` GitHub environment only when you need GitHub-side approval. GitOps PR review remains the deployment gate in this example.

### Capstone A: One complete CI/build/publish workflow

**Create** `.github/workflows/platform-ci.yml`:

```yaml
name: Platform CI and Publish
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
  workflow_dispatch:
permissions:
  contents: read
concurrency:
  group: platform-ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}
jobs:
  quality:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: pip
      - name: Unit tests
        run: |
          python -m pip install -r requirements.txt
          python -m pytest -q
      - name: Docker smoke build
        run: |
          docker build -t order-service:ci .
          docker image inspect order-service:ci >/dev/null
  publish:
    if: ${{ github.event_name != 'pull_request' && github.ref == 'refs/heads/main' }}
    needs: quality
    runs-on: ubuntu-latest
    timeout-minutes: 25
    permissions:
      contents: read
      packages: write
    outputs:
      image: ${{ steps.meta.outputs.image }}
      digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false
      - name: Compute lowercase image name
        id: meta
        env:
          REPOSITORY: ${{ github.repository }}
        run: |
          echo "image=ghcr.io/${REPOSITORY,,}" >> "$GITHUB_OUTPUT"
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build and publish
        id: build
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.image }}:${{ github.sha }}
          provenance: true
      - name: Digest summary
        env:
          IMAGE: ${{ steps.meta.outputs.image }}
          DIGEST: ${{ steps.build.outputs.digest }}
        run: |
          test -n "$DIGEST"
          echo "Published ${IMAGE}@${DIGEST}" >> "$GITHUB_STEP_SUMMARY"
  propose:
    if: ${{ github.event_name != 'pull_request' && github.ref == 'refs/heads/main' }}
    needs: publish
    runs-on: ubuntu-latest
    timeout-minutes: 10
    permissions:
      contents: read
    steps:
      - name: Checkout GitOps repo
        uses: actions/checkout@v4
        with:
          repository: OWNER/order-gitops
          token: ${{ secrets.GITOPS_TOKEN }}
          path: gitops
          persist-credentials: false
      - name: Set image digest
        working-directory: gitops
        env:
          IMAGE: ${{ needs.publish.outputs.image }}
          DIGEST: ${{ needs.publish.outputs.digest }}
        shell: bash
        run: |
          test -n "$DIGEST"
          python3 - <<'PY'
          import os
          from pathlib import Path
          p=Path('helm/order-service/values.yaml')
          # Controlled single-key file, not arbitrary YAML substitution.
          p.write_text('image:\n  reference: "'+os.environ['IMAGE']+'@'+os.environ['DIGEST']+'"\n')
          PY
          git diff --check
      - name: Push change branch and open PR
        working-directory: gitops
        env:
          GH_TOKEN: ${{ secrets.GITOPS_TOKEN }}
          DIGEST: ${{ needs.publish.outputs.digest }}
        shell: bash
        run: |
          set -euo pipefail
          branch="promote/${GITHUB_RUN_ID}-${GITHUB_RUN_ATTEMPT}"
          git config user.name 'github-actions[bot]'
          git config user.email '41898282+github-actions[bot]@users.noreply.github.com'
          git switch -c "$branch"
          git add helm/order-service/values.yaml
          if git diff --cached --quiet; then
            echo 'No changes to propose' >> "$GITHUB_STEP_SUMMARY"
            exit 0
          fi
          git commit -m "Propose image digest ${DIGEST}"
          # gh provides authenticated transport without embedding token in URL or logs.
          gh auth setup-git
          git push origin "$branch"
          gh pr create --repo OWNER/order-gitops --base main --head "$branch" \
            --title "Promote image from run ${GITHUB_RUN_ID}" \
            --body "Review digest ${DIGEST} and test results before merge."
```

**Important dependencies:** The `gh` CLI is preinstalled on standard GitHub-hosted Ubuntu runners at the time of writing, but verify runner image tooling if using other runners. `gh auth setup-git` updates the Git credential helper; do not echo the token. The cross-repository token still has write privilege and must be scoped, rotated, and protected. Because the repository uses `actions/checkout` and third-party Docker actions with readable major tags, **pin every external action to audited full commit SHAs before production**, or use GitHub's workflow dependency lockfile/pinning support.

**Caveat about concurrency:** A new `main` workflow run can overlap an earlier main run only if configured to; the example group and `cancel-in-progress` logic avoid canceling an already executing main release. GitOps PRs are individually reviewable; have a promotion policy to close stale PRs and block older-digest promotions.

### Capstone B: Minimal GitOps Helm chart (complete file tree)

GitOps repository:

```text
order-gitops/
└── helm/
    └── order-service/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
            ├── deployment.yaml
            └── service.yaml
```

`helm/order-service/Chart.yaml`:

```yaml
apiVersion: v2
name: order-service
description: Minimal FastAPI service for GitOps training
type: application
version: 0.1.0
appVersion: '1.0'
```

`helm/order-service/values.yaml` (initial value must be a real published digest; template shown as placeholder):

```yaml
image:
  reference: 'ghcr.io/owner/repo@sha256:REPLACE_WITH_64_HEX_CHARS'
```

`helm/order-service/templates/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: api
          image: {{ .Values.image.reference | quote }}
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8000
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 15
            periodSeconds: 10
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: [ALL]
          resources:
            requests:
              cpu: 25m
              memory: 64Mi
            limits:
              cpu: 250m
              memory: 256Mi
```

`helm/order-service/templates/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
  ports:
    - name: http
      port: 80
      targetPort: 8000
```

`argocd-application.yaml` (store/administer outside the Helm chart, then apply to the cluster):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/OWNER/order-gitops.git
    targetRevision: main
    path: helm/order-service
    helm:
      releaseName: order-service
  destination:
    server: https://kubernetes.default.svc
    namespace: order-service
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**Render and validate locally:**

```bash
cd order-gitops
helm lint helm/order-service
helm template order-service helm/order-service --namespace order-service
# First replace placeholder with a real, published immutable digest.
# On a configured Kubernetes context:
kubectl apply -f argocd-application.yaml
kubectl get applications -n argocd
kubectl -n order-service rollout status deployment/order-service --timeout=180s
kubectl -n order-service port-forward svc/order-service 8088:80
# In another terminal: curl -fsS http://127.0.0.1:8088/health
```

**End-to-end test:**

1. Open an app PR with a changed test. Check the PR workflow runs `quality`, and `publish`/`propose` are skipped.
2. Merge to `main`; see `quality` succeed, then `publish` with an immutable digest in job summary.
3. Verify a GitOps PR was created with `image.reference` set to the exact digest.
4. Review and merge the PR. Verify Argo CD reconciliation and the Deployment's resolved image identity.
5. Run `kubectl get pods -n order-service`, `kubectl describe deploy -n order-service order-service`, and inspect events for image pull failures.
6. Simulate a broken rollout by deliberately proposing an invalid image digest to the GitOps repo; after failure, revert the GitOps commit to the previous known-good digest and confirm recovery. Do **not** do this in a shared cluster.
7. Collect evidence: run URLs, commit SHA, published image digest, approved GitOps PR, Argo CD application state, healthy endpoint response and rollback verification.

**Common faults and precise checks:**

| Symptom | First check | Typical correction |
|---|---|---|
| `403` publishing GHCR | `packages:write`, package/repository access | Grant only job-specific publish permissions |
| `checkout` fails for GitOps | Fine-grained token target repository | Fix installation/token permission, never log token |
| `gh pr create` fails | Token PR write scope and git push success | Check repo policies and head branch |
| Argo CD app OutOfSync | Destination repo branch/path, Application project | Correct GitOps source or investigate diff |
| `ImagePullBackOff` | Registry visibility, image reference, pull secret | Publish correct digest or configure `imagePullSecrets` |
| `CrashLoopBackOff` | Application logs and probes | Fix application startup/configuration |
| No new pods after merge | Did chart image reference actually change? | Compare GitOps diff and deployed spec |
| Wrong version deployed | Digest in approved PR vs live Deployment | Reconcile to approved immutable digest |

**Security note:** The Dockerfile in Lab 00 uses a non-root UID but is an instructional image, not a complete hardened supply chain. Real projects need build-base pinning, lockfile hygiene, image scanning, provenance, policy enforcement, and vulnerability remediation.

---

# Part IV — Production engineering reference

## 1. Events worth mastering

| Event | Typical purpose | Pitfall |
|---|---|---|
| `push` | Mainline CI/publish | Unintentionally publishing from every branch |
| `pull_request` | Untrusted-change checks | Fork secrets are not ordinary privileged secrets |
| `pull_request_target` | Limited trusted-side PR metadata operations | Dangerous if checking out/running attacker-controlled PR code |
| `workflow_dispatch` | Manual release / operations | Must authorize inputs and environments |
| `workflow_call` | Shared workflow contract | Called at job level; privileges are bounded |
| `workflow_run` | React to another workflow outcome | Untrusted artifact laundering / privilege crossing |
| `repository_dispatch` | Cross-repo automation | Validate source identity and payload |
| `schedule` | Maintenance | UTC schedules may be delayed; default-branch restriction |
| `release` | Publish on a release | Tag/release authorization and provenance |
| `merge_group` | Required checks in merge queue | Configure checks for merge queue event |

## 2. Privilege escalation and artifact trust

- Start with `permissions: {contents: read}` and override narrowly per job.
- Do not use a privileged `workflow_run` to execute code from an untrusted artifact generated by a fork PR.
- Do not use `pull_request_target` to checkout the incoming PR head then run its scripts with write tokens.
- Treat every action dependency as code running with runner privileges. Lock actions to verified SHAs or an audited dependency lockfile; review upgrades.
- Put deploy authorization in branch rules, protected environments, IAM trust conditions, registry policy and GitOps review—not only `if:` expressions.
- Require verified provenance for high-assurance workflows; GitHub artifact attestations can strengthen integrity and supply-chain visibility but need compatible organization/repository feature support and verification.
- Secrets are not perfectly safe just because logs mask recognized values; a malicious action or subprocess can exfiltrate them. Minimize their exposure.

## 3. Release and rollback policy

**Release evidence:** reviewed source commit, test status, scanned image digest, provenance, deployment PR approval, live Deployment observed digest, smoke-test result, monitoring and rollback runbook.

**Rollback is a Git change in GitOps:** revert GitOps to previously verified image digest. Always verify application health, schema compatibility, background job behavior, error rates, queue semantics, and data changes. Container rollback cannot automatically reverse database migrations. Use backward-compatible expand/contract migrations.

## 4. Organization-wide reuse design

A mature organization might keep `org/.github` workflow templates, a shared CI workflow repository, policy-as-code, a small set of standardized composite actions, and pinned versions. Consumers should pin a reviewed revision of shared workflows to avoid unreviewed breaking changes. Define contracts: inputs, output names, IAM permissions, runner labels, support policy, and deprecation timeline. A single enormous reusable workflow with dozens of conditionals is usually harder to secure than a few focused contracts.

## 5. Workflow performance and cost

Optimize bottlenecks in this order: trim unnecessary triggers, ensure useful path filters, right-size runner, parallelize independent tests, cache immutable dependencies keyed by lockfiles, reuse Docker layer cache appropriately, reduce artifact size/retention, and collect queue-time vs execution-time metrics. Remember GitHub-hosted runner minutes/storage and self-hosted infrastructure have different cost models. Confirm current plan billing and private-repo entitlements rather than relying on a tutorial snapshot.

## 6. Detailed PR-to-production threat model

| Trust boundary | Threat | Mitigation |
|---|---|---|
| PR code -> CI runner | Code execution by contributor | Unprivileged, ephemeral runner; no cloud tokens |
| Third-party action -> runner | Malicious action update | Verified commit SHA/lockfile, code review |
| CI job -> registry | Poisoned tag/overwriting | Scoped publish rights, immutable digest consumption |
| Publish -> GitOps PR | Unauthorized environment promotion | Separate repo permissions, required review, audit |
| GitOps -> cluster | Excessive controller privilege | Argo CD Projects, least-privilege Kubernetes RBAC |
| Cluster -> production services | Faulty application update | Progressive delivery, health checks, fast rollback |
| Build -> audit system | Missing provenance | Build attestations, SBOM, log retention |

---

# Operations runbooks and interview assessment

## Runbook 1 — GitHub Actions job stuck in queue

1. Verify trigger and branch filters, workflow file presence, and whether Actions is enabled.
2. For hosted runners: inspect GitHub status, repository concurrency, approvals and runner label availability.
3. For self-hosted: inspect runner Online/Idle status, labels, group access, service logs and outbound connectivity.
4. Inspect concurrency groups; a pending job can be superseded by a newer pending run.
5. Cancel obsolete runs, fix configuration, then rerun with an audit trail.

## Runbook 2 — `Resource not accessible by integration` / permission denied

1. Identify exact operation and API response; examine effective `permissions` at workflow and job levels.
2. Confirm event type (`pull_request` from fork, Dependabot, push) and token scope limitations.
3. Verify repository/org settings and environment protections.
4. For cross-repo operations, the original repo's `GITHUB_TOKEN` usually cannot write the target repo; use narrowly scoped GitHub App installation credentials or suitable approved mechanism.
5. Never 'fix' by setting `permissions: write-all` globally.

## Runbook 3 — AWS STS OIDC `AccessDenied`

1. Ensure job has `id-token: write` and correct role ARN and region.
2. Compare actual GitHub token claim context to `aud`/`sub` trust policy and role's federated provider ARN.
3. Remember GitHub environment jobs may have different `sub` claims from branch jobs.
4. Check org-level restrictions and IAM explicit denies; preserve least privilege.
5. Test `aws sts get-caller-identity`, then add only necessary service permissions.

## Runbook 4 — Cache seems stale or unsafe

1. Understand exact cache key and restore keys. Compare lockfile hashing.
2. Clear offending caches through supported repository controls if needed.
3. Do not cache sensitive material, credentials, mutable authorization state, or anything that must have strong provenance.
4. Rebuild without cache to distinguish cache corruption from genuine build failures.

## Runbook 5 — Deployment succeeded but code didn't change

1. Compare Git SHA -> registry digest -> GitOps PR -> chart values -> live Kubernetes `image` field.
2. Check Argo CD target repo, branch, application revision, sync and health.
3. Query `kubectl rollout history` and `kubectl get rs,pods` for the intended Deployment.
4. Examine imagePullPolicy and immutable tags/digests; do not assume `latest` means a new image.
5. Test HTTP endpoint and app version identifier; reconcile and restore the intended digest.

## Interview drills (answer aloud, then prove with a lab)

1. Why is GitHub Actions not just a hosted Jenkinsfile parser?
2. What persists between steps? Between jobs? Across runs?
3. Why does `needs` cause a downstream skip after failure?
4. How do `if: always()`, `failure()`, `cancelled()` and `success()` differ?
5. When should you choose composite action versus reusable workflow?
6. Why is `pull_request_target` more dangerous than ordinary PR CI?
7. Explain a secure workflow for fork PR tests and protected publishing.
8. What exact trust claims does AWS OIDC validate?
9. Why can't `GITHUB_TOKEN` automatically push to an unrelated private repository?
10. What do artifacts, caches and registries each guarantee and *not* guarantee?
11. How does matrix expansion work, and how would you avoid expensive accidental expansion?
12. Why is an image digest better than a mutable tag for promotion?
13. What are requirements for safe self-hosted runner access?
14. How would you make production deployments mutually exclusive without pretending to have a FIFO queue?
15. What audit evidence proves a specific source commit is now running in production?
16. What can go wrong with `workflow_run` and attacker-controlled artifacts?
17. How do you implement rollback when a database schema migration already ran?
18. How would you trace a `403` from GHCR and a corresponding `ImagePullBackOff`?
19. How should a monorepo's workflows avoid expensive unrelated builds?
20. When would you select Jenkins instead of GitHub Actions for an internal regulated workload?

## Competency rubric

| Level | Demonstrable evidence |
|---|---|
| Foundation | Correct simple workflow and event filters; explains job/runner boundaries |
| Intermediate | Tests, Docker publishing, artifacts, caches, matrix, reusable components |
| Advanced | OIDC, scoped tokens, environment approvals, GitOps, failure recovery |
| Production | Threat model, immutable provenance, runner isolation, protected release policy, rollback drills |
| Platform engineering | Standardized shared workflows, governance, observability, cost controls, multi-team adoption |

**Completion condition:** You are not done simply by reading. You are done when you have a clean repository, passing tests, one immutable published image, a reviewed GitOps PR, a healthy rollout, a successful rollback drill, and a written explanation of how each token, runner, and deployment permission is scoped.

## Reference links and version policy

- Workflow syntax: https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax
- Reusable workflows: https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows
- Deployment environments: https://docs.github.com/en/actions/concepts/workflows-and-actions/deployment-environments
- Security and dependency pinning: https://docs.github.com/en/actions/how-tos/secure-your-work
- OIDC cloud providers: https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-cloud-providers
- Secure `pull_request_target`: https://docs.github.com/en/actions/reference/security/securely-using-pull_request_target
- GitHub CLI: https://cli.github.com/manual/
- actionlint: https://github.com/rhysd/actionlint
- Docker build-push-action: https://github.com/docker/build-push-action
- Argo CD: https://argo-cd.readthedocs.io/
- Helm: https://helm.sh/docs/

**Maintenance policy:** Before reusing a complete workflow in an actual organization, (1) validate against the current GitHub docs and your plan's feature availability, (2) pin reviewed action revisions, (3) update runtime/container versions and dependencies, (4) audit repository, org, AWS and GitOps permissions, and (5) test it on a disposable staging target. This handbook is extensive but cannot replace your organization's security review or real operational experience.
