# Lab 14 — Capstone: Multi-Chart Application

## Objectives
This capstone pulls together every prior lab into one realistic exercise: build, template, package, distribute, harden, break, and recover an umbrella chart representing a small microservice application.

By the end you will have exercised: chart anatomy (Lab 02), values layering (Lab 03), templating (Labs 04–06), subcharts (Lab 07), OCI distribution via ECR (Lab 09), hooks and tests (Labs 10–11), schema validation (Lab 12), and failure recovery (Lab 13) — end to end, in one project.

## Prerequisites
- Labs 01–13 completed
- Helm 3.x, `kubectl`, AWS CLI with ECR access
- A running cluster (EKS or local)

---

## 1. Application Layout

Build an umbrella chart `myapp` with three subcharts, mirroring a realistic microservice layout:

```
myapp/
├── Chart.yaml                 # dependencies: frontend, backend, postgresql (Bitnami)
├── values.yaml                # shared defaults + global section
├── values-dev.yaml
├── values-staging.yaml
├── values-prod.yaml
├── values.schema.json
├── charts/
│   ├── frontend/               # your own chart
│   └── backend/                 # your own chart, owns the DB migration hook
└── templates/
    └── tests/
        └── test-backend-health.yaml
```

`postgresql` comes from the Bitnami repo as a real dependency (Lab 07); `frontend` and `backend` are charts you write yourself.

## 2. Chart.yaml Dependencies

```yaml
apiVersion: v2
name: myapp
version: 0.1.0
appVersion: "1.0.0"
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
  - name: frontend
    version: "0.1.0"
    repository: "file://../frontend"
  - name: backend
    version: "0.1.0"
    repository: "file://../backend"
```

## 3. Environment-Specific Values

```yaml
# values.yaml (shared base)
global:
  imageRegistry: <account-id>.dkr.ecr.us-east-1.amazonaws.com
postgresql:
  enabled: true
backend:
  replicaCount: 1
frontend:
  replicaCount: 1
```
```yaml
# values-prod.yaml (layered on top via -f)
backend:
  replicaCount: 3
frontend:
  replicaCount: 3
postgresql:
  primary:
    persistence:
      size: 50Gi
```
```bash
helm template myapp -f myapp/values.yaml -f myapp/values-prod.yaml
```
This is the pattern from Lab 03 applied for real: base + environment overlay, deep-merged.

## 4. The Backend's Migration Hook and Test

In `charts/backend/templates/`, include the `pre-upgrade`/`pre-install` migration Job pattern from Lab 10 (with `before-hook-creation,hook-succeeded` deletion policy), and a `helm.sh/hook: test` Pod hitting a real `/healthz` endpoint per Lab 11's corrected pattern (not just a bare `wget`).

## 5. Schema Validation

Write a `values.schema.json` at the umbrella level enforcing at minimum:
- `global.imageRegistry` required, string
- `backend.replicaCount` and `frontend.replicaCount`: integer, minimum 1
- `postgresql.enabled`: boolean

Confirm invalid input is rejected before rendering:
```bash
helm template myapp --set backend.replicaCount=0
```

## 6. Full Lifecycle: Package → ECR → Deploy → Break → Recover

```bash
# Package and push the umbrella chart (Lab 08 + Lab 09)
helm dependency build myapp
helm package myapp
helm push myapp-0.1.0.tgz oci://<account-id>.dkr.ecr.us-east-1.amazonaws.com/helm

# Install with atomic safety (Lab 13)
helm install myapp-release oci://<account-id>.dkr.ecr.us-east-1.amazonaws.com/helm/myapp \
  --version 0.1.0 -f values-prod.yaml -n myapp-prod --atomic --cleanup-on-fail

# Run the test suite (Lab 11)
helm test myapp-release -n myapp-prod

# Introduce a breaking change: bump backend image tag to a broken build
helm upgrade myapp-release oci://.../myapp --version 0.2.0 -f values-prod.yaml -n myapp-prod
# With --atomic on this upgrade too, a failed migration hook or failed test
# (if wired as a post-upgrade gate in your pipeline) should trigger automatic rollback

# Verify recovery
helm history myapp-release -n myapp-prod
helm status myapp-release -n myapp-prod
```

This step deliberately exercises Lab 13's recovery model for real: a bad upgrade should either auto-rollback via `--atomic`, or — if you disable `--atomic` to observe the failure state first — require a manual `helm rollback` to the last good revision, exactly as in Lab 13's troubleshooting exercises.

## 7. Bridge to ArgoCD

This same umbrella chart, once pushed to ECR as an OCI artifact, is exactly what an ArgoCD `Application` resource would point at as its Helm source — `repoURL: oci://<account-id>.dkr.ecr.us-east-1.amazonaws.com/helm`, `chart: myapp`, `targetRevision: 0.1.0`, with `values-prod.yaml` supplied via ArgoCD's own values override mechanism. This hand-off is the natural starting point for the ArgoCD curriculum.

---

## 8. Capstone Troubleshooting — Combined Scenario

**Scenario:** You attempt `helm upgrade myapp-release ... --version 0.2.0` and it fails. Three things are wrong at once:

1. The `backend` subchart's values reference `backend.database.host`, but `values-prod.yaml` sets `database.host` at the wrong nesting level (a Lab 07-style subchart values mistake)
2. The migration hook Job from the *previous* failed attempt was never cleaned up, blocking the new hook from being created (a Lab 10-style deletion-policy mistake)
3. `--atomic` wasn't used on this particular upgrade, so the release is now sitting in `failed` state and the target you'd naturally roll back to isn't actually the last fully-correct one, because someone manually patched a ConfigMap on the live cluster after that revision was deployed (a Lab 13-style drift mistake)

**Task:**
1. Diagnose issue 1: run `helm get values myapp-release --all -n myapp-prod` and `helm template` locally with the same values file; confirm `backend.database.host` isn't landing where the subchart expects it. Fix the values file nesting.
2. Diagnose issue 2: `kubectl get jobs -n myapp-prod`, find the stuck hook Job from the prior attempt, and either delete it manually to unblock now, or confirm the hook's `hook-delete-policy` already includes `before-hook-creation` and figure out why it still didn't clean up (check if the previous failure happened *before* the hook resource was even created vs. after — deletion policies only act on hooks Helm actually tracked)
3. Diagnose issue 3: compare `helm get manifest myapp-release --revision <last-good>` against live cluster state for the resource in question; identify the drifted ConfigMap; decide whether to restore it manually or accept the drift as the new intended state and reconcile it back into the chart's values instead
4. Fix all three, re-run the upgrade with `--atomic` this time, and confirm `helm test myapp-release -n myapp-prod` passes cleanly afterward

**This is deliberately messy** — real production incidents are rarely one clean root cause. Practicing the discipline of isolating and fixing each contributing issue separately, rather than guessing at one fix and hoping it resolves everything, is the actual skill being tested here.

---

## Cleanup
```bash
helm uninstall myapp-release -n myapp-prod
kubectl delete ns myapp-prod
rm -f myapp-0.1.0.tgz
```

---

**Curriculum complete.** All 14 labs across 5 phases are done — Helm fundamentals through production-grade multi-chart operations. This capstone's OCI-published chart is the natural entry point for the ArgoCD hands-on curriculum next.
