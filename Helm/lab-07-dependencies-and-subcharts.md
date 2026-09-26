# Lab 07 — Dependencies & Subcharts

## Objectives
- Declare and manage chart dependencies via `Chart.yaml`
- Understand values propagation between parent and child charts, including `global`
- Use conditions and tags to toggle subcharts on/off
- Alias a dependency to deploy multiple instances of the same subchart
- Practice debugging two common dependency mistakes

## Prerequisites
- Labs 01–06 completed
- Helm 3.x installed

---

## 1. Declaring Dependencies

In Helm 3, dependencies live directly in `Chart.yaml` (no separate `requirements.yaml` like Helm 2):

```yaml
# Chart.yaml
apiVersion: v2
name: myapp
version: 0.1.0
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
  - name: redis
    version: "17.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
    tags:
      - caching
```

Fetch the actual chart archives into `charts/`:
```bash
helm dependency update myapp
```
This creates `Chart.lock` (pins exact resolved versions) and downloads `.tgz` files into `myapp/charts/`.

`helm dependency build` re-downloads based on the **existing** `Chart.lock` (reproducible), whereas `helm dependency update` re-resolves versions against the constraints in `Chart.yaml` (can pick up new versions).

## 2. Values Propagation

A subchart only sees the values under its own key in the parent's `values.yaml`:

```yaml
# parent values.yaml
postgresql:
  auth:
    username: myapp_user
  primary:
    persistence:
      size: 10Gi
```
Inside the `postgresql` subchart's own templates, this is accessible simply as `.Values.auth.username` (the subchart doesn't see the `postgresql:` prefix — that's stripped by Helm when it scopes values down to the subchart).

## 3. global Values

Values under a top-level `global` key are visible to **every** chart in the dependency tree — parent and all subcharts — without any key-scoping:

```yaml
global:
  imageRegistry: my-private-registry.example.com
```
Every chart (parent or subchart) accesses this identically as `.Values.global.imageRegistry`. This is the standard way to share things like a private registry prefix or a shared environment name across an entire umbrella chart.

## 4. Conditions and Tags

- `condition`: points to a specific boolean values key that enables/disables that exact dependency
- `tags`: group multiple dependencies under one flag

```yaml
# values.yaml
postgresql:
  enabled: false
tags:
  caching: false   # disables everything tagged "caching", e.g. redis
```
`condition` on a specific dependency takes precedence over its `tags` setting if both are present.

## 5. Aliasing a Dependency

To deploy the same chart twice under different names (e.g. two Redis instances — one for caching, one for sessions):

```yaml
dependencies:
  - name: redis
    version: "17.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    alias: redis-cache
  - name: redis
    version: "17.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    alias: redis-sessions
```
Each aliased instance gets its own values scope in the parent's `values.yaml`, keyed by the alias:
```yaml
redis-cache:
  auth:
    enabled: false
redis-sessions:
  auth:
    enabled: true
```

---

## 6. Troubleshooting Exercise 1 — Subchart Values Not Picked Up

**Scenario:** You set a value intending to configure the `postgresql` subchart, but it has no effect:
```yaml
# values.yaml — WRONG
auth:
  username: myapp_user
```
Deploy and inspect — the subchart's default username is still used.

**Task:**
1. Run `helm template myapp` and search the rendered PostgreSQL manifest for the username value
2. Recognize the values were placed at the parent's top level instead of nested under the subchart's key (`postgresql:`) — the parent's own top-level `auth` key means nothing to the subchart
3. Fix:
```yaml
postgresql:
  auth:
    username: myapp_user
```
4. Re-render and confirm the value now appears correctly in the subchart's manifest

## 7. Troubleshooting Exercise 2 — Chart.lock Drift Between Environments

**Scenario:** A teammate runs `helm dependency update` locally, which bumps the resolved `redis` version in `Chart.lock` because the `Chart.yaml` constraint (`17.x.x`) allows a newer patch release than what's currently in `charts/`. They don't commit the updated `charts/*.tgz` files, and CI runs `helm dependency build` (not `update`) — but since `Chart.lock` was committed with the new version and the corresponding `.tgz` wasn't, the build fails or fetches a version nobody tested against.

**Task:**
1. Understand the difference again in this context: `build` trusts `Chart.lock` as the source of truth and fetches exactly what it specifies; it does not regenerate `Chart.lock`
2. Establish the correct workflow: whenever `Chart.lock` changes, it must be committed together with awareness that CI will fetch that exact version fresh — the `.tgz` files themselves generally should NOT be committed (they're reproducible from `Chart.lock`), but the lock file itself always must be
3. Fix the immediate issue by running `helm dependency build` locally against the committed `Chart.lock` and confirming it resolves cleanly
4. Document this as a required CI step: always run `helm dependency build` (not `update`) in pipelines, so nothing silently drifts to a newer version than what was tested

**Takeaway:** `Chart.lock` is your reproducibility guarantee — pipelines should always `build` against it, and only a deliberate `update` (with review of the diff) should ever change it.

---

## Cleanup
```bash
rm -rf myapp
```
