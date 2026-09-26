# Lab 10 — Hooks

## Objectives
- Understand every Helm hook type and when each fires
- Control hook execution order with weights
- Control hook cleanup with deletion policies
- Implement a real-world pattern: a DB migration Job as a hook
- Practice debugging two common hook failures

## Prerequisites
- Labs 01–09 completed
- Helm 3.x installed

---

## 1. Hook Types

A hook is just a normal Kubernetes manifest (usually a `Job` or `Pod`) annotated to run at a specific point in a release's lifecycle, instead of being tracked as a normal part of the release:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    "helm.sh/hook": pre-upgrade
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: myapp-migrator:1.0
      restartPolicy: Never
```

| Hook | Fires |
|---|---|
| `pre-install` | Before any templates are rendered/applied on install |
| `post-install` | After all resources are installed |
| `pre-upgrade` | Before templates are applied on upgrade |
| `post-upgrade` | After all resources are upgraded |
| `pre-delete` | Before any resources are deleted on uninstall |
| `post-delete` | After all resources are deleted |
| `pre-rollback` | Before rollback applies previous manifests |
| `post-rollback` | After rollback completes |

A resource can declare multiple hook types in a comma-separated list if it should run at more than one lifecycle point.

## 2. Hook Weights

When multiple hooks share the same type, `helm.sh/hook-weight` controls execution order — **ascending**, lowest first:

```yaml
metadata:
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "1"
```
Hooks with no weight default to `0`. Ties are broken by resource name, alphabetically — but never rely on that; always set explicit weights when order matters.

## 3. Deletion Policies

`helm.sh/hook-delete-policy` controls when a hook resource gets cleaned up:

| Policy | Behavior |
|---|---|
| `before-hook-creation` (default) | Delete the previous hook resource right before the new one is created — this is what allows a hook Job to run again on the next upgrade, since Job names/specs are otherwise immutable |
| `hook-succeeded` | Delete immediately after the hook succeeds |
| `hook-failed` | Delete if the hook fails (rarely used alone — you usually want to keep failed hook resources around for debugging) |

Without `before-hook-creation` (or an equivalent policy), a second upgrade attempting to recreate the same hook Job name will fail, because completed Jobs aren't automatically cleaned up by Kubernetes and Job specs are immutable — this is Troubleshooting Exercise 1.

## 4. Real-World Pattern: DB Migration Hook

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-db-migrate
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: ["./migrate.sh"]
```
This runs the migration **before** the app's own Deployment is updated, on both fresh installs and upgrades, and cleans itself up on success while also clearing the way for the next run.

---

## 5. Troubleshooting Exercise 1 — Failed Hook Blocks All Future Releases

**Scenario:** A migration hook Job fails once (e.g. bad SQL). The next `helm upgrade` attempt now also fails immediately, before even reaching the migration step:
```
Error: pre-upgrade hooks failed: job "myapp-db-migrate" already exists
```

**Task:**
1. Run `kubectl get jobs -n <namespace>` and confirm the failed Job from the previous attempt is still sitting there
2. Recognize the missing piece: the hook had no `hook-delete-policy` set (or only had `hook-succeeded`), so a **failed** hook Job was never cleaned up, and its immutable spec blocks Helm from creating a new one with the same name on retry
3. Manually clean up to unblock right now: `kubectl delete job myapp-db-migrate -n <namespace>`
4. Fix the chart going forward by adding `before-hook-creation` to the delete policy (as shown in the pattern above), so this can never happen again regardless of success/failure
5. Re-run the upgrade and confirm it proceeds past the hook step

## 6. Troubleshooting Exercise 2 — Hook Execution Order Bug

**Scenario:** Two `pre-upgrade` hooks exist — one creates a ConfigMap the other reads from — but neither has an explicit weight, and on some upgrades they run in the wrong order, causing an intermittent failure.

**Task:**
1. Inspect both hook manifests and confirm neither has `helm.sh/hook-weight` set
2. Understand why this is intermittent rather than consistently broken: with no explicit weight, ordering falls back to name-based tie-breaking, which can still produce a "working" order by coincidence until something (a rename, a new hook added alphabetically earlier) shifts it
3. Fix by adding explicit weights: the ConfigMap-creating hook gets `"0"`, the consumer hook gets `"1"`
4. Re-run the upgrade multiple times and confirm deterministic, correct ordering every time

**Takeaway:** never depend on default/alphabetical hook ordering for anything with a real dependency between hooks — always set explicit weights.

---

## Cleanup
```bash
kubectl delete job -l app.kubernetes.io/instance=demo -n <namespace>
helm uninstall demo -n <namespace>
```
