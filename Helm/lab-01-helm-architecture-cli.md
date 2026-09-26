# Lab 01 — Helm Architecture & CLI

## Objectives
- Understand Helm 3's client-only architecture (no Tiller)
- Understand how release state is stored and tracked
- Perform install, upgrade, rollback, and inspect release history
- Practice debugging two common real-world failures

## Prerequisites
- `kubectl` access to a cluster (minikube/kind/EKS)
- Helm 3.x installed (`helm version` to confirm)
- A namespace to work in, e.g. `kubectl create ns helm-lab01`

---

## 1. Helm 3 Architecture

Helm 3 removed Tiller entirely — there's no in-cluster server component. The Helm CLI talks directly to the Kubernetes API server using your existing `kubeconfig` context and RBAC permissions. This means:

- No Tiller service account = no cluster-wide privilege escalation risk that plagued Helm 2
- Release state is stored **as a Kubernetes Secret** in the same namespace as the release (by default, type `helm.sh/release.v1`)
- Multiple revisions of a release = multiple Secrets, one per revision

Check this yourself:
```bash
helm install demo-nginx bitnami/nginx --namespace helm-lab01
kubectl get secrets -n helm-lab01 -l owner=helm
```
You'll see something like `sh.helm.release.v1.demo-nginx.v1` — that Secret's data is the compressed, base64-encoded release manifest + metadata.

## 2. Install / Upgrade / Rollback / History

```bash
# Install
helm install demo-nginx bitnami/nginx -n helm-lab01

# Upgrade (bump replica count via --set)
helm upgrade demo-nginx bitnami/nginx -n helm-lab01 --set replicaCount=2

# View history — note it's append-only
helm history demo-nginx -n helm-lab01
```

Key behavior: **history is append-only**. Every `install`/`upgrade`/`rollback` creates a new revision Secret — nothing is overwritten. A `rollback` to revision 1 doesn't delete revisions 2 and 3; it creates a new revision 4 whose content matches revision 1.

```bash
helm rollback demo-nginx 1 -n helm-lab01
helm history demo-nginx -n helm-lab01
# You'll now see revision 4, content-identical to revision 1
```

## 3. Inspecting Releases

```bash
# The fully rendered manifest that was actually applied
helm get manifest demo-nginx -n helm-lab01

# The values used for the current release (merged, not just what you passed)
helm get values demo-nginx -n helm-lab01

# All user-supplied values across the whole history, revision by revision
helm get values demo-nginx -n helm-lab01 --revision 2
```

`helm get values` without `--all` only shows values you explicitly set — not the chart's defaults. Use `--all` to see the fully computed values.

---

## 4. Troubleshooting Exercise 1 — Release Name Collision

**Scenario:** A teammate already installed a release called `demo-nginx` in the same namespace under a *different* chart. You try to install your own:

```bash
helm install demo-nginx bitnami/apache -n helm-lab01
```

You get:
```
Error: INSTALLATION FAILED: cannot re-use a name that is still in use
```

**Task:**
1. Confirm the existing release and chart with `helm list -n helm-lab01`
2. Decide: uninstall the existing release, pick a different release name, or use a different namespace
3. Document why Helm enforces this (release name is the primary key tying the Secret history, not the chart)

**Fix:**
```bash
helm install demo-apache bitnami/apache -n helm-lab01
```

---

## 5. Troubleshooting Exercise 2 — Stale Repo Index

**Scenario:** A new chart version was published to a repo, but your local Helm client doesn't see it.

```bash
helm search repo bitnami/nginx --versions
# Latest version shown is older than what's actually published upstream
```

**Task:**
1. Understand that `helm repo add` only fetches the index once — it does not auto-refresh
2. Run `helm repo update` to pull the latest `index.yaml`
3. Re-run `helm search repo bitnami/nginx --versions` and confirm the new version now appears

**Root cause takeaway:** Helm caches the repo index locally (`~/.cache/helm/repository/`). Any pipeline or script that installs charts by "latest" without running `helm repo update` first risks deploying a stale version silently.

---

## Cleanup
```bash
helm uninstall demo-nginx demo-apache -n helm-lab01
kubectl delete ns helm-lab01
```
