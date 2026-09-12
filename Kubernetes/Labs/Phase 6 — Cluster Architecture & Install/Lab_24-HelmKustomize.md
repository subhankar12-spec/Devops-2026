# Kubernetes Hands-On Lab 24 — Helm & Kustomize

## 24.1 Objectives

By the end of this lab, you should be able to:

- Understand what problem Helm solves (templated, versioned, releasable packages of YAML)
- Install Helm and add a chart repository
- Install, upgrade, rollback, and uninstall a Helm release
- Override chart values at install time
- Understand a chart's structure (`Chart.yaml`, `values.yaml`, `templates/`)
- Understand what problem Kustomize solves (patch-based YAML composition, no templating language)
- Structure a `base` + `overlays` Kustomize layout for dev/prod variants
- Use `kubectl apply -k` to apply a Kustomization directly
- Understand when to reach for Helm vs Kustomize vs plain YAML
- Troubleshoot a failed Helm release stuck mid-upgrade
- Perform common Helm/Kustomize tasks quickly for the CKA

## 24.2 Architecture

```
                HELM                                KUSTOMIZE
        (templating + package                (patch-based composition,
         manager + release                    no templating language,
         history)                              built into kubectl)

   Chart (versioned package)              base/
     Chart.yaml                             ├── deployment.yaml
     values.yaml  <- defaults               ├── service.yaml
     templates/                             └── kustomization.yaml
       deployment.yaml  (Go templates)
       service.yaml                       overlays/
       _helpers.tpl                         ├── dev/
                                             │   ├── kustomization.yaml
   helm install my-app ./chart              │   └── patch-replicas.yaml
     -> renders templates + values          └── prod/
     -> tracks as a "release"                   ├── kustomization.yaml
     -> helm rollback if it breaks               └── patch-replicas.yaml

                                          kubectl apply -k overlays/prod
                                            -> merges base + prod patches
                                            -> no templating, pure YAML merge
```

Key concept: Helm uses a **templating language** (Go templates) to generate YAML from variables, and tracks installed "releases" with version history you can roll back. Kustomize has **no templating at all** — it takes complete, valid YAML (`base/`) and applies strategic merge patches on top (`overlays/`) to produce environment-specific variants, with nothing hidden behind template syntax.

## 24.3 Prerequisites

```bash
kubectl version --client
kubectl get nodes
```

Install Helm if not already present:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

Kustomize is built into `kubectl` (`kubectl apply -k`, `kubectl kustomize`) — no separate install needed for basic use.

## 24.4 Lab 1 — Add a Helm Repository and Install a Chart

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo bitnami/nginx
```

Install it as a named release:

```bash
helm install my-nginx bitnami/nginx --set service.type=ClusterIP
```

Check the release:

```bash
helm list
helm status my-nginx
kubectl get all -l app.kubernetes.io/instance=my-nginx
```

## 24.5 Override Values at Install Time

Two ways to override chart defaults:

```bash
# Inline, one-off overrides:
helm install my-nginx-2 bitnami/nginx --set replicaCount=3 --set service.type=NodePort
```

```yaml
# my-values.yaml — for larger, reusable overrides:
replicaCount: 2
service:
  type: ClusterIP
resources:
  requests:
    cpu: 100m
    memory: 128Mi
```

```bash
helm install my-nginx-3 bitnami/nginx -f my-values.yaml
```

Inspect exactly what values a release is actually running with:

```bash
helm get values my-nginx-3
```

## 24.6 Lab 2 — Upgrade and Rollback a Release

Upgrade `my-nginx` to change replica count:

```bash
helm upgrade my-nginx bitnami/nginx --set replicaCount=3
```

Check revision history:

```bash
helm history my-nginx
```

Each `helm upgrade` creates a new numbered revision — this history is what makes rollback possible.

Roll back to the previous revision:

```bash
helm rollback my-nginx 1
kubectl get pods -l app.kubernetes.io/instance=my-nginx
```

Compare this to a bare `kubectl apply` workflow (Lab 04's Deployments) — Helm's rollback is chart-release-aware (it reverts the ENTIRE rendered manifest set back to that revision's state), whereas `kubectl rollout undo` only works within a single Deployment's own revision history.

## 24.7 Uninstall a Release

```bash
helm uninstall my-nginx-2
helm list
```

By default `helm uninstall` deletes all resources it created — check `--keep-history` if you want to retain the release record for future rollback/audit purposes.

## 24.8 Anatomy of a Chart

```bash
helm pull bitnami/nginx --untar
ls nginx/
```

Expected structure:

```
nginx/
├── Chart.yaml       <- name, version, description, dependencies
├── values.yaml      <- default configuration values
├── templates/       <- Go-templated Kubernetes manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   └── _helpers.tpl <- reusable template snippets
└── charts/          <- any subchart dependencies
```

Render the templates locally without installing anything (extremely useful for debugging):

```bash
helm template my-test ./nginx --set replicaCount=5 | less
```

## 24.9 Lab 3 — Kustomize base/overlay Structure

Create a base:

```bash
mkdir -p k8s/base k8s/overlays/dev k8s/overlays/prod
```

```yaml
# k8s/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

```yaml
# k8s/base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-svc
spec:
  selector:
    app: web-app
  ports:
    - port: 80
      targetPort: 80
```

```yaml
# k8s/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

Write all three files:

```bash
cat > k8s/base/deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
EOF

cat > k8s/base/service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: web-app-svc
spec:
  selector:
    app: web-app
  ports:
    - port: 80
      targetPort: 80
EOF

cat > k8s/base/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
EOF
```

Preview the rendered base (no cluster changes yet):

```bash
kubectl kustomize k8s/base
```

## 24.10 Create dev and prod Overlays

```yaml
# k8s/overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namePrefix: dev-
resources:
  - ../../base
patches:
  - target:
      kind: Deployment
      name: web-app
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 1
```

```yaml
# k8s/overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namePrefix: prod-
resources:
  - ../../base
patches:
  - target:
      kind: Deployment
      name: web-app
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 5
      - op: replace
        path: /spec/template/spec/containers/0/image
        value: nginx:1.28
```

Write both:

```bash
cat > k8s/overlays/dev/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namePrefix: dev-
resources:
  - ../../base
patches:
  - target:
      kind: Deployment
      name: web-app
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 1
EOF

cat > k8s/overlays/prod/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namePrefix: prod-
resources:
  - ../../base
patches:
  - target:
      kind: Deployment
      name: web-app
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 5
      - op: replace
        path: /spec/template/spec/containers/0/image
        value: nginx:1.28
EOF
```

## 24.11 Preview and Apply Each Overlay

```bash
kubectl kustomize k8s/overlays/dev
kubectl kustomize k8s/overlays/prod
```

Notice: same base, two completely different rendered outputs — `dev-web-app` with 1 replica on `nginx:1.27`, `prod-web-app` with 5 replicas on `nginx:1.28`. No templating syntax anywhere; it's all valid, readable YAML at every layer.

Apply directly:

```bash
kubectl apply -k k8s/overlays/dev
kubectl apply -k k8s/overlays/prod

kubectl get deployments
```

## 24.12 Helm vs Kustomize — When to Use Which

| | Helm | Kustomize |
|---|---|---|
| Best for | Distributing reusable, third-party, or complex parameterized applications | Managing your own app's environment-specific variants (dev/staging/prod) |
| Templating | Yes (Go templates) — powerful but can obscure the final YAML | No — what you see in `base/` is always valid YAML |
| Release tracking / rollback | Built-in (`helm history`, `helm rollback`) | None — you rely on your own Git history / CI pipeline |
| Learning curve | Higher (templating syntax, chart structure, hooks) | Lower (just YAML + patches) |
| Native `kubectl` support | No (separate binary) | Yes (`kubectl apply -k` built in since 1.14) |
| Typical real-world use | Installing third-party software (ingress-nginx, cert-manager, Prometheus) | Managing your own team's app manifests across environments |

Many real organizations use **both**: Helm to install third-party infrastructure charts, Kustomize (or a GitOps tool built on it, like Argo CD) to manage their own application manifests per environment.

## 24.13 Break It — Failed Helm Upgrade

Simulate an upgrade with an invalid value that breaks the chart's rendering (e.g. a non-existent image tag causing an unschedulable Pod):

```bash
helm upgrade my-nginx-3 bitnami/nginx --set image.tag=this-tag-does-not-exist
```

Check the release and Pod status:

```bash
helm status my-nginx-3
kubectl get pods -l app.kubernetes.io/instance=my-nginx-3
```

Pods show `ImagePullBackOff`, and the Helm release may show as `deployed` even though the underlying Pods are unhealthy — Helm considers the manifest apply itself successful; it does **not** deeply verify application health by default.

## 24.14 Diagnose and Recover

```bash
helm history my-nginx-3
```

Roll back to the last known-good revision:

```bash
helm rollback my-nginx-3
kubectl get pods -l app.kubernetes.io/instance=my-nginx-3 -w
```

Confirm the release is healthy again:

```bash
helm status my-nginx-3
```

## 24.15 CKA Practice Task

**Task**

1. Add the `bitnami` Helm repo and install `bitnami/nginx` as release `cka-nginx` with `replicaCount=2`.
2. Upgrade it to `replicaCount=4`, then check `helm history` and roll back to the first revision.
3. Build a Kustomize `base/` with a Deployment (`nginx:1.27`, 2 replicas) and a Service.
4. Build a `staging` overlay that patches replicas to 3 and adds a `namePrefix: staging-`.
5. Render (don't apply) the staging overlay and confirm the output matches expectations.

**Target Time**

8 minutes

Try it without looking at previous commands.

## 24.16 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

What core problem does Helm's release history/rollback solve that plain `kubectl apply` does not?

**Question 2**

Why does Kustomize have no templating language, and what's the tradeoff versus Helm's Go templates?

**Question 3**

If `helm upgrade` succeeds but the underlying Pods immediately crash-loop, does Helm automatically roll back? Why or why not?

**Question 4**

What command lets you preview the exact YAML a Helm chart would generate, without installing anything?

**Question 5**

In a Kustomize overlay, what is the purpose of `resources: [../../base]`?

**Question 6**

Given the comparison table in 24.12, when would you specifically prefer Kustomize over Helm for your own team's application?

## 24.17 Useful Commands

```bash
helm repo add <name> <url>
helm repo update
helm search repo <term>

helm install <release> <chart> [--set key=value] [-f values.yaml]
helm upgrade <release> <chart> [--set key=value]
helm rollback <release> [revision]
helm uninstall <release>

helm list
helm status <release>
helm history <release>
helm get values <release>
helm template <release> <chart>
helm pull <chart> --untar

kubectl kustomize <directory>
kubectl apply -k <directory>
```

## 24.18 Cleanup

```bash
helm uninstall my-nginx my-nginx-3 cka-nginx

kubectl delete -k k8s/overlays/dev
kubectl delete -k k8s/overlays/prod

rm -rf k8s nginx my-values.yaml
```

## 24.19 Lab Checklist

- [ ] Added a Helm repo and installed a chart
- [ ] Overridden chart values via `--set` and a values file
- [ ] Upgraded a release and reviewed its history
- [ ] Rolled back a release to a previous revision
- [ ] Inspected a chart's `Chart.yaml`/`values.yaml`/`templates/` structure
- [ ] Rendered a chart locally with `helm template`
- [ ] Built a Kustomize `base/` with a Deployment and Service
- [ ] Built `dev`/`prod` overlays with patches and `namePrefix`
- [ ] Rendered and applied overlays with `kubectl kustomize`/`apply -k`
- [ ] Reproduced a Helm release that "succeeded" but left Pods unhealthy
- [ ] Diagnosed and rolled back the failed release
- [ ] Completed the CKA task within 8 minutes
