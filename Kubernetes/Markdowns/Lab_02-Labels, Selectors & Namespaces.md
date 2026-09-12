# Kubernetes Hands-On Lab 02 — Labels, Selectors & Namespaces

## 2.1 Objectives

By the end of this lab, you should be able to:

- Create and organize Namespaces
- Set the default Namespace for a context
- Apply Labels to resources at creation and after the fact
- Query resources using Equality-based and Set-based label selectors
- Use `kubectl label` and `kubectl get --show-labels`
- Understand how Namespaces isolate resources but not all resource types
- Apply a ResourceQuota to a Namespace
- Troubleshoot a resource that a selector-based tool (Service/Deployment) can't find due to a label mismatch
- Perform common Label/Namespace tasks quickly for the CKA

## 2.2 Architecture

```
                        Cluster
                           |
        +------------------+------------------+
        |                                      |
        v                                      v
   Namespace: lab02-dev                Namespace: lab02-prod
        |
        +-- Pod (app=web, tier=frontend, env=dev, version=v1)
        +-- Pod (app=web, tier=frontend, env=dev, version=v2)
        +-- Pod (app=web, tier=backend,  env=dev, version=v1)
        +-- Pod (app=cache, tier=backend, env=staging)

Selectors query ACROSS labels, WITHIN a namespace (by default).
Namespaces do NOT isolate: Nodes, PersistentVolumes, StorageClasses,
or other cluster-scoped resources.
```

## 2.3 Prerequisites

```bash
kubectl version --client
kubectl get nodes
```

## 2.4 Lab 1 — Create Namespaces

Create the following file: [`namespaces.yaml`](./namespaces.yaml)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lab02-dev
  labels:
    environment: dev
---
apiVersion: v1
kind: Namespace
metadata:
  name: lab02-prod
  labels:
    environment: prod
```

Apply it:

```bash
kubectl apply -f namespaces.yaml
```

Verify:

```bash
kubectl get namespaces
kubectl get ns --show-labels
```

## 2.5 Working With Namespaces

Check which namespace your current context defaults to:

```bash
kubectl config view --minify | grep namespace
```

Run a one-off command against a specific namespace:

```bash
kubectl get pods -n lab02-dev
```

Set the default namespace for your current context (saves typing `-n` every time):

```bash
kubectl config set-context --current --namespace=lab02-dev
```

Verify it stuck:

```bash
kubectl config view --minify | grep namespace
kubectl get pods
```

List resources across **all** namespaces:

```bash
kubectl get pods --all-namespaces
kubectl get pods -A
```

## 2.6 Lab 2 — Create Labeled Pods

Create the following file: [`pods-with-labels.yaml`](./pods-with-labels.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-frontend-1
  namespace: lab02-dev
  labels:
    app: web
    tier: frontend
    env: dev
    version: v1
spec:
  containers:
    - name: nginx
      image: nginx:1.27
---
apiVersion: v1
kind: Pod
metadata:
  name: web-frontend-2
  namespace: lab02-dev
  labels:
    app: web
    tier: frontend
    env: dev
    version: v2
spec:
  containers:
    - name: nginx
      image: nginx:1.27
---
apiVersion: v1
kind: Pod
metadata:
  name: web-backend-1
  namespace: lab02-dev
  labels:
    app: web
    tier: backend
    env: dev
    version: v1
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
---
apiVersion: v1
kind: Pod
metadata:
  name: cache-1
  namespace: lab02-dev
  labels:
    app: cache
    tier: backend
    env: staging
spec:
  containers:
    - name: redis
      image: redis:7
```

Apply it:

```bash
kubectl apply -f pods-with-labels.yaml
```

View all labels:

```bash
kubectl get pods -n lab02-dev --show-labels
```

## 2.7 Equality-Based Selectors

Find Pods where `app=web`:

```bash
kubectl get pods -n lab02-dev -l app=web
```

Find Pods where `tier=frontend`:

```bash
kubectl get pods -n lab02-dev -l tier=frontend
```

Combine conditions (AND, comma-separated):

```bash
kubectl get pods -n lab02-dev -l app=web,tier=frontend
```

Find Pods where `tier` is **not** `backend`:

```bash
kubectl get pods -n lab02-dev -l tier!=backend
```

## 2.8 Set-Based Selectors

Find Pods where `tier` is `frontend` OR `backend`:

```bash
kubectl get pods -n lab02-dev -l 'tier in (frontend,backend)'
```

Find Pods where `env` is NOT `dev`:

```bash
kubectl get pods -n lab02-dev -l 'env notin (dev)'
```

Find Pods that have a `version` label at all (regardless of value):

```bash
kubectl get pods -n lab02-dev -l version
```

Find Pods that do **not** have a `version` label:

```bash
kubectl get pods -n lab02-dev -l '!version'
```

## 2.9 Label Management

Add a new label to an existing Pod:

```bash
kubectl label pod cache-1 -n lab02-dev tier=backend --overwrite
```

Add a label without overwriting (fails if it already exists):

```bash
kubectl label pod web-backend-1 -n lab02-dev criticality=high
```

Remove a label:

```bash
kubectl label pod cache-1 -n lab02-dev env-
```

Verify:

```bash
kubectl get pod cache-1 -n lab02-dev --show-labels
```

## 2.10 Selectors in Practice — Deployments and Services

Key concept: a Deployment's `spec.selector.matchLabels` must match `spec.template.metadata.labels`, and a Service's `spec.selector` must match the Pod labels it's meant to route to. Selectors are how these objects "find" their Pods — there's no other link.

Check what Pods a selector would match before wiring it into a Service or Deployment:

```bash
kubectl get pods -n lab02-dev -l app=web,tier=frontend -o wide
```

## 2.11 Apply a ResourceQuota to the Namespace

Create the following file: [`resourcequota.yaml`](./resourcequota.yaml)

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: lab02-dev-quota
  namespace: lab02-dev
spec:
  hard:
    pods: "10"
    requests.cpu: "2"
    requests.memory: 2Gi
    limits.cpu: "4"
    limits.memory: 4Gi
```

Apply it:

```bash
kubectl apply -f resourcequota.yaml
```

Check usage against the quota:

```bash
kubectl describe resourcequota lab02-dev-quota -n lab02-dev
```

## 2.12 Break It — Label Mismatch

Create the following file: [`pod-unlabeled.yaml`](./pod-unlabeled.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: unlabeled-pod
  namespace: lab02-dev
spec:
  containers:
    - name: nginx
      image: nginx:1.27
```

Apply it:

```bash
kubectl apply -f pod-unlabeled.yaml
```

Now try to select it as if it were part of the `web` app:

```bash
kubectl get pods -n lab02-dev -l app=web
```

Notice `unlabeled-pod` does **not** appear — this is exactly what happens in production when a Service or Deployment selector silently matches zero Pods because someone forgot a label, or a typo slipped into the YAML (`app: wbe` instead of `app: web`).

Simulate the typo scenario:

```bash
kubectl label pod unlabeled-pod -n lab02-dev app=wbe
kubectl get pods -n lab02-dev -l app=web
```

Still missing. This is the same failure mode as a Service with 0 endpoints.

## 2.13 Diagnose and Recover

Check what labels the Pod actually has:

```bash
kubectl get pod unlabeled-pod -n lab02-dev --show-labels
```

Fix the label:

```bash
kubectl label pod unlabeled-pod -n lab02-dev app=web --overwrite
```

Verify it's now selectable:

```bash
kubectl get pods -n lab02-dev -l app=web
```

## 2.14 Namespace & Label YAML Modification Exercise

Modify `pods-with-labels.yaml` to:

- add a new Pod named `web-frontend-3` in `lab02-dev`
- give it labels `app=web`, `tier=frontend`, `env=dev`, `version=v3`
- image `nginx:1.28`

Then confirm it's picked up by the same selector as the other frontend Pods:

```bash
kubectl apply -f pods-with-labels.yaml
kubectl get pods -n lab02-dev -l app=web,tier=frontend
```

## 2.15 CKA Practice Task

**Task**

1. Create a namespace named `cka-lab02`.
2. Create a Pod named `api-server` in that namespace with image `nginx:1.28` and labels `app=api`, `tier=backend`.
3. Using a single `kubectl get` command with a label selector, list only Pods where `tier=backend` in `cka-lab02`.
4. Add a new label `owner=devops-team` to `api-server` without removing existing labels.
5. Set `cka-lab02` as your current context's default namespace.

**Target Time**

5 minutes

Try it without looking at previous commands.

## 2.16 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

Why do Deployments and Services rely on label selectors instead of Pod names?

**Question 2**

A Service has 0 endpoints even though matching Pods appear to exist and are Running. What are the first two things you check?

**Question 3**

What is the difference between an equality-based selector (`app=web`) and a set-based selector (`app in (web,cache)`)?

**Question 4**

Which of these are namespaced, and which are cluster-scoped: Pods, Nodes, Services, PersistentVolumes, Deployments, StorageClasses?

**Question 5**

If you delete a Namespace, what happens to all the resources inside it?

**Question 6**

What's the difference between `kubectl label pod x key=value` and `kubectl label pod x key=value --overwrite`?

## 2.17 Useful Commands

```bash
kubectl get namespaces
kubectl get ns --show-labels
kubectl create namespace <name>
kubectl delete namespace <name>

kubectl config view --minify | grep namespace
kubectl config set-context --current --namespace=<name>

kubectl get pods -n <namespace>
kubectl get pods --all-namespaces
kubectl get pods -A

kubectl get pods -l key=value
kubectl get pods -l key!=value
kubectl get pods -l 'key in (v1,v2)'
kubectl get pods -l 'key notin (v1,v2)'
kubectl get pods -l key
kubectl get pods -l '!key'

kubectl label pod <name> key=value
kubectl label pod <name> key=value --overwrite
kubectl label pod <name> key-

kubectl get pods --show-labels

kubectl describe resourcequota <name> -n <namespace>
```

## 2.18 Cleanup

```bash
kubectl delete namespace lab02-dev
kubectl delete namespace lab02-prod
kubectl delete namespace cka-lab02
```

Or:

```bash
kubectl delete -f namespaces.yaml
kubectl delete -f pods-with-labels.yaml
kubectl delete -f resourcequota.yaml
kubectl delete -f pod-unlabeled.yaml
```

> Deleting a Namespace deletes every namespaced resource inside it — this is usually simpler than deleting Pods one by one.

## 2.19 Lab Checklist

- [ ] Created Namespaces via YAML
- [ ] Set a default Namespace for the current context
- [ ] Created Pods with multiple labels
- [ ] Queried Pods with equality-based selectors
- [ ] Queried Pods with set-based selectors
- [ ] Added, overwrote, and removed labels on a live Pod
- [ ] Understood how Services/Deployments use selectors to find Pods
- [ ] Applied a ResourceQuota to a Namespace
- [ ] Reproduced a label-mismatch failure (the "0 endpoints" scenario)
- [ ] Diagnosed and fixed the label mismatch
- [ ] Completed the CKA task within 5 minutes

## Files for this lab

```
02-Labels-Selectors-Namespaces/
├── README.md
├── namespaces.yaml
├── pods-with-labels.yaml
├── resourcequota.yaml
└── pod-unlabeled.yaml
```
