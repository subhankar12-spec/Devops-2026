# Kubernetes Hands-On Lab 08 — Node Affinity & Pod Affinity/Anti-Affinity

## 8.1 Objectives

By the end of this lab, you should be able to:

- Label Nodes for scheduling decisions
- Use `nodeSelector` for simple node targeting
- Use `nodeAffinity` with `requiredDuringSchedulingIgnoredDuringExecution` (hard rule)
- Use `nodeAffinity` with `preferredDuringSchedulingIgnoredDuringExecution` (soft rule)
- Use `podAffinity` to co-locate Pods
- Use `podAntiAffinity` to spread Pods apart
- Understand `topologyKey` and how it defines "same/different location"
- Troubleshoot a Pod stuck `Pending` due to an affinity rule no Node can satisfy
- Perform common affinity tasks quickly for the CKA

## 8.2 Architecture

```
                     Scheduler
                         |
        which Node satisfies these rules?
                         |
        +----------------+----------------+
        |                |                |
        v                v                v
  nodeSelector      nodeAffinity      podAffinity /
  (exact label      (required = hard  podAntiAffinity
   match only)       preferred = soft) (relative to OTHER
                                        pods already
                                        placed, via
                                        topologyKey)

topologyKey examples:
  kubernetes.io/hostname        -> "same/different Node"
  topology.kubernetes.io/zone   -> "same/different AZ"
```

Key concept: `nodeAffinity` looks at **Node labels**. `podAffinity`/`podAntiAffinity` look at **labels on other Pods already running**, then use `topologyKey` to decide what "close" or "far" means (same Node? same zone?).

## 8.3 Prerequisites

```bash
kubectl version --client
kubectl get nodes
kubectl get nodes --show-labels
```

> This lab assumes at least 2 worker nodes for the affinity/anti-affinity behavior to be visible. On a single-node cluster (e.g. Minikube/kind default), some steps will show the rule being enforced via `Pending` status instead of actual spreading — that's still a valid, observable outcome.

## 8.4 Lab 1 — Label Nodes

List current nodes and pick two:

```bash
kubectl get nodes
```

Label them:

```bash
kubectl label node <node-1> disktype=ssd
kubectl label node <node-2> disktype=hdd
```

Verify:

```bash
kubectl get nodes --show-labels
kubectl get nodes -L disktype
```

## 8.5 nodeSelector — Simple Targeting

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nodeselector-pod
spec:
  nodeSelector:
    disktype: ssd
  containers:
    - name: nginx
      image: nginx:1.27
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nodeselector-pod
spec:
  nodeSelector:
    disktype: ssd
  containers:
    - name: nginx
      image: nginx:1.27
EOF
```

Confirm it landed on the labeled Node:

```bash
kubectl get pod nodeselector-pod -o wide
```

`nodeSelector` is exact-match-only and has no concept of "prefer but don't require" — that's what `nodeAffinity` adds.

## 8.6 nodeAffinity — Required (Hard Rule)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: required-affinity-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
  containers:
    - name: nginx
      image: nginx:1.27
```

Apply it and confirm placement:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: required-affinity-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
  containers:
    - name: nginx
      image: nginx:1.27
EOF

kubectl get pod required-affinity-pod -o wide
```

`requiredDuringSchedulingIgnoredDuringExecution` means: the rule **must** be true at scheduling time, but if the Node's label later changes, the already-running Pod is **not** evicted (that's the "IgnoredDuringExecution" part).

## 8.7 nodeAffinity — Preferred (Soft Rule)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: preferred-affinity-pod
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80
          preference:
            matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
  containers:
    - name: nginx
      image: nginx:1.27
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: preferred-affinity-pod
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80
          preference:
            matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
  containers:
    - name: nginx
      image: nginx:1.27
EOF

kubectl get pod preferred-affinity-pod -o wide
```

Unlike the required version, this Pod will still schedule even if no Node matches — it just prefers a matching Node when possible (higher `weight` = stronger preference among multiple soft rules).

## 8.8 Lab 2 — podAffinity (Co-locate Pods)

First, create a Pod that acts as the "anchor":

```bash
kubectl run cache-pod --image=redis:7 --labels="app=cache"
```

Now schedule a Pod that must land on the **same Node** as anything labeled `app=cache`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: colocated-pod
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - cache
          topologyKey: kubernetes.io/hostname
  containers:
    - name: app
      image: nginx:1.27
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: colocated-pod
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - cache
          topologyKey: kubernetes.io/hostname
  containers:
    - name: app
      image: nginx:1.27
EOF
```

Verify both Pods landed on the same Node:

```bash
kubectl get pods -o wide | grep -E "cache-pod|colocated-pod"
```

## 8.9 Lab 3 — podAntiAffinity (Spread Pods Apart)

Delete the previous Pods to start clean:

```bash
kubectl delete pod cache-pod colocated-pod
```

Create a Deployment where replicas must NOT share a Node with each other (classic HA pattern):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spread-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spread-app
  template:
    metadata:
      labels:
        app: spread-app
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - spread-app
              topologyKey: kubernetes.io/hostname
      containers:
        - name: nginx
          image: nginx:1.27
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spread-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spread-app
  template:
    metadata:
      labels:
        app: spread-app
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - spread-app
              topologyKey: kubernetes.io/hostname
      containers:
        - name: nginx
          image: nginx:1.27
EOF
```

Check placement:

```bash
kubectl get pods -l app=spread-app -o wide
```

On a cluster with fewer Nodes than replicas, some Pods will stay `Pending` — this is expected and demonstrates the "required" rule being strictly enforced (see 8.10).

## 8.10 Break It — Impossible Affinity Rule

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: impossible-affinity-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values:
                  - nvme-that-does-not-exist
  containers:
    - name: nginx
      image: nginx:1.27
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: impossible-affinity-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values:
                  - nvme-that-does-not-exist
  containers:
    - name: nginx
      image: nginx:1.27
EOF
```

## 8.11 Diagnose and Recover

```bash
kubectl get pod impossible-affinity-pod
```

Status stays `Pending`. Investigate:

```bash
kubectl describe pod impossible-affinity-pod
```

Look for a scheduler Event like:

```
0/2 nodes are available: 2 node(s) didn't match Pod's node affinity/selector.
```

Fix by deleting and recreating with a label that actually exists on a Node:

```bash
kubectl delete pod impossible-affinity-pod

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: fixed-affinity-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
  containers:
    - name: nginx
      image: nginx:1.27
EOF

kubectl get pod fixed-affinity-pod -o wide
```

## 8.12 CKA Practice Task

**Task**

1. Label one Node with `zone=us-east`.
2. Create a Pod named `zone-pod` using `nodeSelector` to require `zone=us-east`.
3. Create a Deployment named `web-ha` with 2 replicas and a `podAntiAffinity` rule (using `topologyKey: kubernetes.io/hostname`) so no two replicas share a Node.
4. Create a Pod named `sidecar-colocate` with a `podAffinity` rule requiring it to run on the same Node as any Pod labeled `app=web-ha`.
5. Verify placement of all Pods with `-o wide`.

**Target Time**

8 minutes

Try it without looking at previous commands.

## 8.13 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

What's the practical difference between `nodeSelector` and `nodeAffinity` with a `required` rule?

**Question 2**

What does "IgnoredDuringExecution" mean for a running Pod if the Node's label changes after scheduling?

**Question 3**

What does `topologyKey: kubernetes.io/hostname` mean versus `topologyKey: topology.kubernetes.io/zone` in a `podAntiAffinity` rule?

**Question 4**

A Deployment has 5 replicas and a `required` `podAntiAffinity` rule keyed on `kubernetes.io/hostname`, but the cluster only has 3 Nodes. What happens?

**Question 5**

What's the difference in scheduler behavior between `requiredDuringSchedulingIgnoredDuringExecution` and `preferredDuringSchedulingIgnoredDuringExecution`?

**Question 6**

A Pod is stuck `Pending` with the event `didn't match Pod's node affinity/selector`. What are your first two diagnostic commands?

## 8.14 Useful Commands

```bash
kubectl get nodes --show-labels
kubectl get nodes -L <label-key>
kubectl label node <name> <key>=<value>
kubectl label node <name> <key>-

kubectl get pod <name> -o wide
kubectl describe pod <name>

kubectl get pods -o wide -l <selector>
```

## 8.15 Cleanup

```bash
kubectl delete pod nodeselector-pod required-affinity-pod preferred-affinity-pod \
  cache-pod colocated-pod impossible-affinity-pod fixed-affinity-pod \
  zone-pod sidecar-colocate

kubectl delete deployment spread-app web-ha

kubectl label node <node-1> disktype-
kubectl label node <node-2> disktype-
```

## 8.16 Lab Checklist

- [ ] Labeled Nodes for scheduling
- [ ] Used `nodeSelector` for exact-match placement
- [ ] Used `nodeAffinity` with a required (hard) rule
- [ ] Used `nodeAffinity` with a preferred (soft) rule
- [ ] Used `podAffinity` to co-locate a Pod with another
- [ ] Used `podAntiAffinity` to spread Deployment replicas across Nodes
- [ ] Understood `topologyKey` and how it defines "same/different location"
- [ ] Reproduced a Pod stuck `Pending` from an impossible affinity rule
- [ ] Diagnosed the failure via scheduler events
- [ ] Completed the CKA task within 8 minutes
