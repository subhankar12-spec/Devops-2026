# Kubernetes Hands-On Lab 15 — NetworkPolicies

## 15.1 Objectives

By the end of this lab, you should be able to:

- Understand that all Pod traffic is allowed by default until a NetworkPolicy says otherwise
- Understand that NetworkPolicies require a CNI plugin that enforces them
- Create a default-deny-all ingress policy for a namespace
- Allow traffic from specific Pods using `podSelector`
- Allow traffic from specific namespaces using `namespaceSelector`
- Combine `podSelector` and `namespaceSelector` in one rule (AND vs OR semantics)
- Restrict egress traffic from a Pod
- Allow DNS egress explicitly (a near-universal requirement once egress is restricted)
- Troubleshoot a Pod that can't be reached due to an overly strict NetworkPolicy
- Perform common NetworkPolicy tasks quickly for the CKA

## 15.2 Architecture

```
                Namespace: lab15
                       |
        default-deny-all NetworkPolicy
        (blocks ALL ingress unless
         explicitly allowed)
                       |
        +--------------+--------------+
        |                             |
        v                             v
   allow-from-frontend            allow-from-monitoring-ns
   (podSelector: tier=frontend)   (namespaceSelector: purpose=monitoring)
        |                             |
        v                             v
   backend Pods now reachable    backend Pods now reachable
   ONLY from frontend Pods       ONLY from Pods in the
   in the same namespace         monitoring namespace
```

Key concept: NetworkPolicies are **additive/allow-only** — you cannot write a policy that explicitly denies something; you can only restrict a Pod to a default-deny state and then allow specific exceptions. Multiple policies selecting the same Pod are combined with OR (if ANY policy allows a connection, it's allowed). Also critical: **NetworkPolicies do nothing at all unless your CNI plugin implements them** (Calico, Cilium, and others do; the basic bridge/kubenet setups in some minimal environments do not).

## 15.3 Prerequisites

```bash
kubectl version --client
kubectl get nodes
```

Check your CNI plugin — this determines whether this entire lab will actually have any effect:

```bash
kubectl get pods -n kube-system | grep -Ei 'calico|cilium|weave|flannel'
```

> If your cluster uses a CNI that doesn't enforce NetworkPolicy (e.g. plain Flannel), every policy below will be accepted by the API server but silently have **no effect** on traffic. This is one of the most common CKA/production confusions — always verify enforcement before assuming a "working" policy actually failed.

## 15.4 Lab 1 — Set Up Test Resources

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lab15
  labels:
    purpose: netpol-lab
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: lab15
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
        tier: backend
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: lab15
spec:
  selector:
    app: backend
  ports:
    - port: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: lab15
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        tier: frontend
    spec:
      containers:
        - name: busybox
          image: busybox:1.36
          command: ["sleep", "3600"]
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: lab15
  labels:
    purpose: netpol-lab
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: lab15
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
        tier: backend
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: lab15
spec:
  selector:
    app: backend
  ports:
    - port: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: lab15
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        tier: frontend
    spec:
      containers:
        - name: busybox
          image: busybox:1.36
          command: ["sleep", "3600"]
EOF
```

## 15.5 Baseline — Everything Can Talk to Everything

Before any policy exists, confirm the default-open behavior:

```bash
kubectl exec -n lab15 deploy/frontend -- wget -qO- --timeout=3 backend-svc
```

This should succeed — Kubernetes has no network isolation by default.

## 15.6 Lab 2 — Default Deny All Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: lab15
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: lab15
spec:
  podSelector: {}
  policyTypes:
    - Ingress
EOF
```

`podSelector: {}` (empty) means "select every Pod in this namespace." With only `Ingress` in `policyTypes` and no `ingress:` rules listed, this blocks **all** incoming traffic to every Pod in `lab15`.

Confirm the previous connection now fails:

```bash
kubectl exec -n lab15 deploy/frontend -- wget -qO- --timeout=3 backend-svc
```

Expect a timeout.

## 15.7 Lab 3 — Allow From a Specific Pod (podSelector)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend
  namespace: lab15
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: frontend
      ports:
        - protocol: TCP
          port: 80
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend
  namespace: lab15
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: frontend
      ports:
        - protocol: TCP
          port: 80
EOF
```

Test again — this should now succeed, since `frontend` Pods have `tier: frontend`:

```bash
kubectl exec -n lab15 deploy/frontend -- wget -qO- --timeout=3 backend-svc
```

Confirm an **unlabeled** Pod is still blocked:

```bash
kubectl run stray-pod -n lab15 --image=busybox:1.36 --command -- sleep 3600
kubectl exec -n lab15 stray-pod -- wget -qO- --timeout=3 backend-svc
```

Expect a timeout — `stray-pod` doesn't have `tier: frontend`.

## 15.8 Lab 4 — Allow From a Specific Namespace (namespaceSelector)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lab15-monitoring
  labels:
    purpose: monitoring
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-monitoring-ns
  namespace: lab15
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              purpose: monitoring
      ports:
        - protocol: TCP
          port: 80
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: lab15-monitoring
  labels:
    purpose: monitoring
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-monitoring-ns
  namespace: lab15
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              purpose: monitoring
      ports:
        - protocol: TCP
          port: 80
EOF
```

Test from that namespace:

```bash
kubectl run monitor-pod -n lab15-monitoring --image=busybox:1.36 --command -- sleep 3600
kubectl exec -n lab15-monitoring monitor-pod -- wget -qO- --timeout=3 backend-svc.lab15
```

This succeeds because this policy is independent of (and additive to) `allow-from-frontend` — remember, multiple policies combine with OR.

## 15.9 podSelector + namespaceSelector Together: AND, Not OR

A common point of confusion — when both are in the **same** `from` entry, they combine with AND (must match both); when they're **separate entries** in the `from` list, they combine with OR (match either).

AND example — only Pods labeled `role: trusted` **from within** the monitoring namespace:

```yaml
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            purpose: monitoring
        podSelector:
          matchLabels:
            role: trusted
```

OR example — Pods labeled `tier: frontend` (any namespace) OR any Pod in the monitoring namespace:

```yaml
ingress:
  - from:
      - podSelector:
          matchLabels:
            tier: frontend
      - namespaceSelector:
          matchLabels:
            purpose: monitoring
```

## 15.10 Lab 5 — Restrict Egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-frontend-egress
  namespace: lab15
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 80
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-frontend-egress
  namespace: lab15
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 80
EOF
```

Confirm `frontend` can still reach `backend`:

```bash
kubectl exec -n lab15 deploy/frontend -- wget -qO- --timeout=3 backend-svc
```

Confirm `frontend` can no longer resolve/reach anything else — including DNS:

```bash
kubectl exec -n lab15 deploy/frontend -- nslookup kubernetes.default
```

This will likely fail or hang — restricting egress without an explicit DNS exception is a classic self-inflicted outage.

## 15.11 Allow DNS Egress Explicitly

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: lab15
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

Apply it (this is additive to `restrict-frontend-egress` since both select `tier: frontend`):

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: lab15
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
EOF

kubectl exec -n lab15 deploy/frontend -- nslookup kubernetes.default
```

DNS resolution should now succeed.

## 15.12 Break It — Overly Strict Policy Blocks a Legit Client

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: too-strict
  namespace: lab15
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: frontend-typo
      ports:
        - protocol: TCP
          port: 80
```

Apply it (note: `tier: frontend-typo` doesn't match any real Pod label):

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: too-strict
  namespace: lab15
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: frontend-typo
      ports:
        - protocol: TCP
          port: 80
EOF
```

> Note: this policy alone doesn't break anything, since `allow-from-frontend` (15.7) still independently allows the real frontend traffic (policies are additive/OR). To actually observe a break, delete the working policy first:

```bash
kubectl delete networkpolicy allow-from-frontend -n lab15
kubectl exec -n lab15 deploy/frontend -- wget -qO- --timeout=3 backend-svc
```

Now the connection fails — `too-strict` is the only policy selecting `backend` Pods for ingress, and its selector has a typo.

## 15.13 Diagnose and Recover

```bash
kubectl get networkpolicy -n lab15
kubectl describe networkpolicy too-strict -n lab15
```

Compare the policy's selector against actual Pod labels:

```bash
kubectl get pods -n lab15 --show-labels
```

Spot the mismatch (`frontend-typo` vs `frontend`). Fix it:

```bash
kubectl patch networkpolicy too-strict -n lab15 --type=json \
  -p='[{"op":"replace","path":"/spec/ingress/0/from/0/podSelector/matchLabels/tier","value":"frontend"}]'

kubectl exec -n lab15 deploy/frontend -- wget -qO- --timeout=3 backend-svc
```

## 15.14 CKA Practice Task

**Task**

1. In a new namespace `cka-netpol`, create `web` (image `nginx:1.28`) and `client` (image `busybox:1.36`, `sleep 3600`) Deployments, plus a Service `web-svc` for `web`.
2. Apply a default-deny-all ingress policy to `cka-netpol`.
3. Confirm `client` can no longer reach `web-svc`.
4. Create a policy allowing ingress to `web` only from Pods labeled `role=allowed`.
5. Label `client`'s Pods with `role=allowed` and confirm access is restored.

**Target Time**

8 minutes

Try it without looking at previous commands.

## 15.15 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

If no NetworkPolicy exists in a namespace, what is the default traffic behavior between Pods?

**Question 2**

What happens if you apply a well-formed NetworkPolicy on a cluster whose CNI doesn't support enforcement?

**Question 3**

Two NetworkPolicies both select the same Pod for ingress: one allows from `tier=frontend`, the other allows from `namespace=monitoring`. Does a Pod need to satisfy both, or either?

**Question 4**

Inside a single `from` entry, if you specify both `podSelector` and `namespaceSelector`, are they combined with AND or OR?

**Question 5**

You applied an egress-restricting NetworkPolicy and now the Pod can't resolve any DNS names. What did you forget?

**Question 6**

A Pod that should be reachable isn't. What are the first three things you'd check, in order?

## 15.16 Useful Commands

```bash
kubectl get networkpolicy
kubectl get netpol
kubectl describe networkpolicy <name> -n <namespace>

kubectl get pods --show-labels
kubectl get pods -n kube-system | grep -Ei 'calico|cilium|weave|flannel'

kubectl exec <pod> -- wget -qO- --timeout=3 <target>
kubectl exec <pod> -- nslookup <name>

kubectl patch networkpolicy <name> -n <namespace> --type=json -p='[...]'
```

## 15.17 Cleanup

```bash
kubectl delete namespace lab15 lab15-monitoring cka-netpol
```

## 15.18 Lab Checklist

- [ ] Confirmed default-open traffic behavior before any policy exists
- [ ] Applied a default-deny-all ingress policy
- [ ] Allowed ingress from a specific Pod via `podSelector`
- [ ] Allowed ingress from a specific namespace via `namespaceSelector`
- [ ] Understood AND (same `from` entry) vs OR (separate entries / separate policies)
- [ ] Restricted egress and observed DNS breakage
- [ ] Added an explicit DNS egress allow rule
- [ ] Reproduced an overly strict policy blocking a legitimate client
- [ ] Diagnosed the label mismatch and fixed it
- [ ] Completed the CKA task within 8 minutes
