# Kubernetes Hands-On Lab 19 — RBAC (Roles, ClusterRoles, Bindings)

## 19.1 Objectives

By the end of this lab, you should be able to:

- Understand the four RBAC building blocks: `Role`, `ClusterRole`, `RoleBinding`, `ClusterRoleBinding`
- Understand namespaced (`Role`) vs cluster-scoped (`ClusterRole`) permissions
- Create a `Role` granting specific verbs on specific resources
- Bind a `Role` to a `ServiceAccount` via `RoleBinding`
- Use a `ClusterRole` + `RoleBinding` combination (grant a cluster-scoped Role, but only within one namespace)
- Use `kubectl auth can-i` to test permissions
- Use impersonation (`--as`) to test another identity's access
- Troubleshoot a Pod/ServiceAccount getting `Forbidden` errors against the API server
- Perform common RBAC tasks quickly for the CKA

## 19.2 Architecture

```
        WHO                    WHAT THEY CAN DO              WHERE

  ServiceAccount /       --->   Role (namespaced)      --->  one Namespace
  User / Group                  or
                                 ClusterRole             --->  whole cluster
                                 (cluster-scoped)              OR one Namespace
                                                                (if bound via
                                                                 RoleBinding)

  Binding = the "wire" connecting WHO to WHAT:
    RoleBinding          -> grants a Role OR ClusterRole, scoped to ONE namespace
    ClusterRoleBinding    -> grants a ClusterRole, cluster-wide
```

Key concept — the four combinations and what each actually means:

| Role/ClusterRole | Binding | Effective Scope |
|---|---|---|
| `Role` | `RoleBinding` | That Role's rules, in that Role's namespace only |
| `ClusterRole` | `RoleBinding` | That ClusterRole's rules, but restricted to the RoleBinding's namespace |
| `ClusterRole` | `ClusterRoleBinding` | That ClusterRole's rules, across the entire cluster |
| `Role` | `ClusterRoleBinding` | **Not possible** — a Role cannot be referenced by a ClusterRoleBinding |

The `ClusterRole` + `RoleBinding` combination is a very common real-world pattern: define reusable ClusterRoles once (e.g. a generic "view" or "edit" role), then grant them per-namespace without duplicating YAML.

## 19.3 Prerequisites

```bash
kubectl version --client
kubectl get nodes
```

## 19.4 Lab 1 — Create a Namespace and ServiceAccount

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lab19
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-reader-sa
  namespace: lab19
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: lab19
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-reader-sa
  namespace: lab19
EOF
```

## 19.5 Lab 2 — Create a Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: lab19
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: lab19
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
EOF
```

Field notes:

- `apiGroups: [""]` means the **core** API group (Pods, Services, ConfigMaps, Secrets, etc. — anything under `/api/v1`)
- Named/versioned groups like `apps`, `batch`, `rbac.authorization.k8s.io` are used for Deployments, Jobs, Roles, etc.
- `verbs` are the actions: `get`, `list`, `watch`, `create`, `update`, `patch`, `delete`, `deletecollection`

## 19.6 Lab 3 — Bind the Role to the ServiceAccount

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: lab19
subjects:
  - kind: ServiceAccount
    name: pod-reader-sa
    namespace: lab19
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: lab19
subjects:
  - kind: ServiceAccount
    name: pod-reader-sa
    namespace: lab19
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF
```

## 19.7 Test Permissions with kubectl auth can-i

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:lab19:pod-reader-sa -n lab19
```

Expected: `yes`.

Test something NOT granted:

```bash
kubectl auth can-i delete pods \
  --as=system:serviceaccount:lab19:pod-reader-sa -n lab19

kubectl auth can-i list secrets \
  --as=system:serviceaccount:lab19:pod-reader-sa -n lab19
```

Both should return `no`.

Test in a different namespace — Roles/RoleBindings are namespace-scoped, so this ServiceAccount has no permissions elsewhere:

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:lab19:pod-reader-sa -n default
```

Expected: `no`.

## 19.8 Prove It From Inside a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rbac-test-pod
  namespace: lab19
spec:
  serviceAccountName: pod-reader-sa
  containers:
    - name: kubectl
      image: bitnami/kubectl:latest
      command: ["sleep", "3600"]
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: rbac-test-pod
  namespace: lab19
spec:
  serviceAccountName: pod-reader-sa
  containers:
    - name: kubectl
      image: bitnami/kubectl:latest
      command: ["sleep", "3600"]
EOF
```

From inside, using the Pod's automatically-mounted ServiceAccount token:

```bash
kubectl exec -n lab19 rbac-test-pod -- kubectl get pods
kubectl exec -n lab19 rbac-test-pod -- kubectl get secrets
```

The first succeeds; the second returns a `Forbidden` error — confirming the Pod only has exactly the permissions its ServiceAccount was granted, nothing more (this is why every workload should run with a purpose-built, minimally-scoped ServiceAccount rather than `default`).

## 19.9 Lab 4 — ClusterRole + RoleBinding (Reusable Role, Namespace-Scoped Grant)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: configmap-reader
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: configmap-reader-binding
  namespace: lab19
subjects:
  - kind: ServiceAccount
    name: pod-reader-sa
    namespace: lab19
roleRef:
  kind: ClusterRole
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: configmap-reader
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: configmap-reader-binding
  namespace: lab19
subjects:
  - kind: ServiceAccount
    name: pod-reader-sa
    namespace: lab19
roleRef:
  kind: ClusterRole
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io
EOF
```

Test — granted in `lab19` only, despite `configmap-reader` being a ClusterRole:

```bash
kubectl auth can-i list configmaps \
  --as=system:serviceaccount:lab19:pod-reader-sa -n lab19

kubectl auth can-i list configmaps \
  --as=system:serviceaccount:lab19:pod-reader-sa -n default
```

First: `yes`. Second: `no` — proving the `ClusterRole` + `RoleBinding` combo stays namespace-scoped.

## 19.10 Lab 5 — ClusterRole + ClusterRoleBinding (True Cluster-Wide)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: configmap-reader-cluster-binding
subjects:
  - kind: ServiceAccount
    name: pod-reader-sa
    namespace: lab19
roleRef:
  kind: ClusterRole
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: configmap-reader-cluster-binding
subjects:
  - kind: ServiceAccount
    name: pod-reader-sa
    namespace: lab19
roleRef:
  kind: ClusterRole
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io
EOF
```

Test again in `default` — now it works cluster-wide:

```bash
kubectl auth can-i list configmaps \
  --as=system:serviceaccount:lab19:pod-reader-sa -n default
```

Expected: `yes`.

## 19.11 Built-In ClusterRoles

Kubernetes ships several pre-built, commonly-used ClusterRoles:

```bash
kubectl get clusterrole | grep -E "^(view|edit|admin|cluster-admin)\s"
```

| ClusterRole | Typical use |
|---|---|
| `view` | Read-only access to most objects, no Secrets |
| `edit` | Read/write to most objects, no RBAC changes |
| `admin` | Full control within a namespace, including RBAC for that namespace |
| `cluster-admin` | Full control over everything, cluster-wide — the "root" of Kubernetes RBAC |

## 19.12 Break It — Missing Verb

Simulate a workload that needs to `delete` Pods but its Role only grants `get`/`list`/`watch` (from 19.5):

```bash
kubectl exec -n lab19 rbac-test-pod -- kubectl delete pod some-other-pod
```

## 19.13 Diagnose and Recover

```bash
kubectl auth can-i delete pods \
  --as=system:serviceaccount:lab19:pod-reader-sa -n lab19
```

Returns `no` — confirming this is an RBAC gap, not a typo or a networking issue. Fix by adding the verb to the existing Role:

```bash
kubectl patch role pod-reader -n lab19 --type=json \
  -p='[{"op":"add","path":"/rules/0/verbs/-","value":"delete"}]'

kubectl auth can-i delete pods \
  --as=system:serviceaccount:lab19:pod-reader-sa -n lab19
```

Expected: `yes`.

## 19.14 CKA Practice Task

**Task**

1. Create a namespace `cka-rbac` and a ServiceAccount `deployer-sa` in it.
2. Create a Role `deployment-manager` allowing `get`, `list`, `create`, `update`, `patch` on `deployments` (apiGroup `apps`).
3. Bind it to `deployer-sa` via a RoleBinding.
4. Use `kubectl auth can-i` to confirm `deployer-sa` can create Deployments in `cka-rbac` but not in `default`, and cannot delete Deployments anywhere.
5. Create a ClusterRole `secret-viewer` (get/list on `secrets`), bind it only in `cka-rbac` via RoleBinding, and confirm the scope is namespace-limited.

**Target Time**

8 minutes

Try it without looking at previous commands.

## 19.15 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

Why can't a `Role` be referenced by a `ClusterRoleBinding`?

**Question 2**

What's the practical outcome of binding a `ClusterRole` via a `RoleBinding` versus via a `ClusterRoleBinding`?

**Question 3**

A Pod's application logs show a Kubernetes API `403 Forbidden` error. What's the fastest way to confirm whether this is an RBAC issue and pinpoint the missing permission?

**Question 4**

Why is it bad practice for every Pod to run under the `default` ServiceAccount with broad permissions?

**Question 5**

What does `apiGroups: [""]` mean in a Role's rules, and what resources fall under it?

**Question 6**

What is the difference between the built-in `edit` and `admin` ClusterRoles?

## 19.16 Useful Commands

```bash
kubectl get roles -n <namespace>
kubectl get clusterroles
kubectl get rolebindings -n <namespace>
kubectl get clusterrolebindings

kubectl describe role <name> -n <namespace>
kubectl describe clusterrole <name>
kubectl describe rolebinding <name> -n <namespace>

kubectl auth can-i <verb> <resource> --as=<user-or-sa> -n <namespace>
kubectl auth can-i <verb> <resource> --as=system:serviceaccount:<ns>:<sa-name>

kubectl create role <name> --verb=get,list --resource=pods -n <namespace>
kubectl create rolebinding <name> --role=<role> --serviceaccount=<ns>:<sa> -n <namespace>
```

## 19.17 Cleanup

```bash
kubectl delete namespace lab19 cka-rbac
kubectl delete clusterrole configmap-reader secret-viewer
kubectl delete clusterrolebinding configmap-reader-cluster-binding
```

## 19.18 Lab Checklist

- [ ] Created a ServiceAccount
- [ ] Created a Role scoped to a namespace
- [ ] Bound the Role to the ServiceAccount via RoleBinding
- [ ] Tested permissions with `kubectl auth can-i` and `--as`
- [ ] Proved scoped access from inside a Pod using its mounted ServiceAccount token
- [ ] Used a ClusterRole with a RoleBinding to keep it namespace-scoped
- [ ] Used the same ClusterRole with a ClusterRoleBinding for true cluster-wide access
- [ ] Reviewed built-in `view`/`edit`/`admin`/`cluster-admin` ClusterRoles
- [ ] Reproduced a `Forbidden` error from a missing verb
- [ ] Diagnosed via `auth can-i` and fixed by patching the Role
- [ ] Completed the CKA task within 8 minutes
