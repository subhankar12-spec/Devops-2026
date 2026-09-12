# Kubernetes Hands-On Lab 21 — Authentication/Authorization Deep Dive

## 21.1 Objectives

By the end of this lab, you should be able to:

- Understand the full API server request pipeline: Authentication → Authorization → Admission Control
- Understand the authentication methods Kubernetes supports (client certs, bearer tokens, OIDC, webhook)
- Create a new human user identity using a Certificate Signing Request (CSR)
- Grant that user permissions via RBAC and build them a kubeconfig context
- Understand authorization modes: `RBAC`, `Node`, `ABAC`, `Webhook`
- Understand what Admission Controllers do and see a few in action
- Distinguish a `401 Unauthorized` from a `403 Forbidden` and know what each implies
- Troubleshoot a `kubectl` command failing with `Unable to connect to the server: x509` or `Forbidden`
- Perform common auth-related tasks quickly for the CKA

## 21.2 Architecture

```
   kubectl / client
         |
         v
  +--------------------------------------------------------------+
  |                        API Server                             |
  |                                                                |
  |   1. AUTHENTICATION        2. AUTHORIZATION      3. ADMISSION |
  |   "Who are you?"           "Are you allowed      "Should this  |
  |                              to do THIS?"          request be   |
  |   - Client certificates                            mutated or   |
  |   - Bearer tokens (SA)     - RBAC (default)        rejected?"   |
  |   - OIDC                   - Node                              |
  |   - Webhook                - Webhook              - Mutating   |
  |                             - ABAC (legacy)          webhooks   |
  |   Failure -> 401            Failure -> 403        - Validating |
  |                                                      webhooks   |
  |                                                    - Built-ins  |
  |                                                      (e.g.      |
  |                                                    ResourceQuota,|
  |                                                    LimitRanger)  |
  +--------------------------------------------------------------+
         |
         v
     etcd (if the request passes all three stages)
```

Key concept: these three stages run **in order**, and each can reject a request independently. A `401` means the API server doesn't know who you are at all (bad/expired/missing credentials). A `403` means it knows exactly who you are, but RBAC (or another authorizer) says you're not allowed to do that specific thing. This distinction alone resolves a huge fraction of real-world "kubectl isn't working" tickets.

## 21.3 Prerequisites

```bash
kubectl version --client
kubectl get nodes
```

Check which authorization mode(s) the API server is running with (requires access to the control-plane Node or its static pod manifest):

```bash
kubectl -n kube-system get pod -l component=kube-apiserver -o yaml | grep authorization-mode
```

Most modern clusters run `Node,RBAC`.

## 21.4 Lab 1 — Create a New User via Certificate Signing Request

Kubernetes has no built-in "User" object — human users are identified purely by the Common Name (CN) on a client certificate that the API server trusts. Generate a key and CSR for a new user `jane`:

```bash
openssl genrsa -out jane.key 2048
openssl req -new -key jane.key -out jane.csr -subj "/CN=jane/O=developers"
```

The `O=developers` sets jane's group membership, which RBAC can reference directly.

## 21.5 Submit the CSR to Kubernetes

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane-csr
spec:
  request: <base64-encoded-csr>
  signerName: kubernetes.io/kube-apiserver-client
  usages:
    - client auth
```

Generate the base64 CSR and submit it:

```bash
CSR_BASE64=$(cat jane.csr | base64 | tr -d '\n')

kubectl apply -f - <<EOF
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane-csr
spec:
  request: ${CSR_BASE64}
  signerName: kubernetes.io/kube-apiserver-client
  usages:
    - client auth
EOF
```

Check its status:

```bash
kubectl get csr jane-csr
```

## 21.6 Approve the CSR

CSRs require explicit administrator approval — this is a security checkpoint:

```bash
kubectl certificate approve jane-csr
kubectl get csr jane-csr
```

Extract the signed certificate:

```bash
kubectl get csr jane-csr -o jsonpath='{.status.certificate}' | base64 --decode > jane.crt
```

## 21.7 Build a kubeconfig for the New User

```bash
kubectl config set-credentials jane --client-certificate=jane.crt --client-key=jane.key --embed-certs=true

kubectl config set-context jane-context --cluster=$(kubectl config view -o jsonpath='{.clusters[0].name}') --user=jane

kubectl config get-contexts
```

Test — jane currently has NO RBAC permissions at all, so everything should fail with `403`:

```bash
kubectl --context=jane-context get pods
```

Expected: `Error from server (Forbidden): pods is forbidden: User "jane" cannot list resource "pods"...`

This confirms authentication succeeded (the API server knows this is "jane") but authorization failed (RBAC grants her nothing yet).

## 21.8 Grant jane Permissions via RBAC

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer-read
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jane-developer-binding
subjects:
  - kind: User
    name: jane
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: developer-read
  apiGroup: rbac.authorization.k8s.io
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer-read
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jane-developer-binding
subjects:
  - kind: User
    name: jane
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: developer-read
  apiGroup: rbac.authorization.k8s.io
EOF
```

Test again:

```bash
kubectl --context=jane-context get pods
kubectl --context=jane-context delete pod some-pod
```

First succeeds; second fails with `403` — jane's identity (certificate CN) never changed, only her RBAC grant did. Note: you could alternatively bind by `Group: developers` (from the CSR's `O=developers`) instead of by individual `User: jane` — this is the standard pattern for managing many users at once.

## 21.9 Authorization Modes Overview

| Mode | Summary |
|---|---|
| `RBAC` | The default and recommended mode — Roles/ClusterRoles/Bindings, as used throughout Lab 19-21 |
| `Node` | Special-purpose authorizer that grants kubelets permissions scoped to their own Node's objects only |
| `ABAC` | Attribute-Based Access Control — static policy file, legacy, rarely used in new clusters |
| `Webhook` | Delegates the authorization decision to an external HTTP service — used for custom/enterprise policy engines |

Multiple authorizers can be chained (`--authorization-mode=Node,RBAC` is the common production setting) — a request is allowed if **any** authorizer in the chain allows it.

## 21.10 Admission Controllers — What They Do

Once a request passes authentication and authorization, it goes through admission control before being persisted. Two categories:

- **Mutating** admission webhooks/controllers can modify the request (e.g. injecting a sidecar container, setting default resource requests)
- **Validating** admission webhooks/controllers can only accept or reject (e.g. blocking a Pod that violates a Pod Security Standard)

Check which admission plugins are enabled (requires control-plane access):

```bash
kubectl -n kube-system get pod -l component=kube-apiserver -o yaml | grep enable-admission-plugins
```

Common built-in admission controllers you'll recognize from earlier labs:

- `ResourceQuota` (Lab 07) — enforces namespace resource caps
- `LimitRanger` (Lab 07) — applies default requests/limits
- `NamespaceLifecycle` — prevents creating resources in a namespace that's being deleted
- `DefaultStorageClass` (Lab 18) — attaches the default StorageClass to PVCs missing one
- `PodSecurity` — enforces Pod Security Standards (`privileged`/`baseline`/`restricted`)

## 21.11 Break It — Expired/Revoked Access Simulation

Deny jane's ClusterRoleBinding to simulate access revocation (a very common real "why did my access suddenly break" scenario):

```bash
kubectl delete clusterrolebinding jane-developer-binding
```

```bash
kubectl --context=jane-context get pods
```

## 21.12 Diagnose and Recover

```bash
kubectl auth can-i list pods --as=jane
```

Returns `no` — confirming this is an authorization gap, not a broken certificate (if it were a cert/auth problem, you'd see a `401`/connection-level error instead of a clean `no`).

Distinguish the two failure types precisely:

```bash
# Authorization failure (403) — identity is fine, permission is missing:
kubectl --context=jane-context get pods
# Error from server (Forbidden): ...

# Authentication failure (401) — simulate with a corrupted cert:
kubectl --context=jane-context --client-certificate=/dev/null get pods
# error: x509: ... OR "Unable to connect to the server"
```

Restore jane's access:

```bash
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jane-developer-binding
subjects:
  - kind: User
    name: jane
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: developer-read
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl --context=jane-context get pods
```

## 21.13 CKA Practice Task

**Task**

1. Generate a key and CSR for a user named `bob` with group `qa-team`.
2. Submit and approve the CSR, extract the signed certificate.
3. Build a kubeconfig context `bob-context` for bob.
4. Confirm bob currently gets `Forbidden` on any `kubectl` command.
5. Create a ClusterRole allowing `get`/`list` on `pods` and `services`, bound to the `qa-team` Group.
6. Confirm bob can now list Pods and Services, but not create/delete anything.

**Target Time**

10 minutes

Try it without looking at previous commands.

## 21.14 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

What is the exact difference between what causes a `401 Unauthorized` versus a `403 Forbidden`?

**Question 2**

Why does Kubernetes have no built-in "User" object, and how is a user's identity actually determined?

**Question 3**

What is the purpose of requiring explicit `kubectl certificate approve` for a CSR rather than auto-approving?

**Question 4**

What does the `Node` authorization mode specifically restrict, and why is it needed in addition to RBAC?

**Question 5**

What's the difference between a mutating and a validating admission webhook, and which class does `ResourceQuota` fall into?

**Question 6**

If a user's access suddenly stops working with `Forbidden` errors but their kubeconfig/certificate hasn't changed, what's the most likely explanation?

## 21.15 Useful Commands

```bash
kubectl get csr
kubectl certificate approve <csr-name>
kubectl certificate deny <csr-name>
kubectl get csr <name> -o jsonpath='{.status.certificate}' | base64 --decode

kubectl config set-credentials <user> --client-certificate=<cert> --client-key=<key> --embed-certs=true
kubectl config set-context <context-name> --cluster=<cluster> --user=<user>
kubectl config get-contexts
kubectl config use-context <context-name>

kubectl auth can-i <verb> <resource> --as=<user>
kubectl auth can-i <verb> <resource> --as=<user> --as-group=<group>

kubectl --context=<context> get pods
```

## 21.16 Cleanup

```bash
kubectl delete csr jane-csr bob-csr
kubectl delete clusterrole developer-read qa-team-role
kubectl delete clusterrolebinding jane-developer-binding qa-team-binding

kubectl config delete-context jane-context bob-context
kubectl config delete-user jane bob

rm -f jane.key jane.csr jane.crt bob.key bob.csr bob.crt
```

## 21.17 Lab Checklist

- [ ] Understood the Authentication → Authorization → Admission Control pipeline
- [ ] Generated a client key/CSR and submitted it as a CertificateSigningRequest
- [ ] Approved the CSR and extracted the signed certificate
- [ ] Built a kubeconfig context for the new user identity
- [ ] Confirmed a brand-new user gets `403 Forbidden` with zero RBAC grants
- [ ] Granted permissions via ClusterRole + ClusterRoleBinding to a User and/or Group
- [ ] Reviewed the four authorization modes (RBAC, Node, ABAC, Webhook)
- [ ] Reviewed mutating vs validating admission controllers and named a few built-ins
- [ ] Distinguished a `401` (authentication) failure from a `403` (authorization) failure precisely
- [ ] Reproduced and restored a revoked ClusterRoleBinding
- [ ] Completed the CKA task within 10 minutes
