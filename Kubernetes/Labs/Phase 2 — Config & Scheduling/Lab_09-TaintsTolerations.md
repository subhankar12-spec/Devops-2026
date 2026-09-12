# Kubernetes Hands-On Lab 09 — Taints, Tolerations & Node Selectors

## 9.1 Objectives

By the end of this lab, you should be able to:

- Understand the difference between affinity (Pod chooses Node) and taints (Node repels Pods)
- Taint a Node with `NoSchedule`, `PreferNoSchedule`, and `NoExecute`
- Add matching Tolerations to a Pod so it can be scheduled onto a tainted Node
- Understand `tolerationSeconds` with `NoExecute`
- Observe a running Pod get evicted when a `NoExecute` taint is added live
- Recall the built-in taints Kubernetes adds automatically (e.g. `node.kubernetes.io/not-ready`)
- Combine taints/tolerations with `nodeSelector`/`nodeAffinity` for full control (dedicated Nodes pattern)
- Troubleshoot a Pod stuck `Pending` because it lacks a Toleration for a tainted Node
- Perform common taint/toleration tasks quickly for the CKA

## 9.2 Architecture

```
   Node (tainted: key=value:NoSchedule)
              |
     REPELS all Pods by default
              |
     unless the Pod has a matching
     Toleration for that exact
     key/value/effect
              |
        +-----+-----+
        |           |
        v           v
   Pod (no       Pod (has
   toleration)   matching
   -> stays      toleration)
   Pending       -> CAN schedule
   elsewhere     here (not
                 forced here —
                 toleration only
                 permits, it
                 doesn't attract)
```

Key concept — the direction of control is opposite to affinity:

- **Affinity** = the Pod expresses a preference/requirement about which Node it wants.
- **Taints + Tolerations** = the Node repels Pods; a Toleration only removes that repulsion, it does **not** attract the Pod there. If you want a Pod to *specifically* land on tainted Nodes (not just be allowed to), you combine a Toleration with `nodeAffinity`/`nodeSelector` — the classic "dedicated Node pool" pattern.

## 9.3 Prerequisites

```bash
kubectl version --client
kubectl get nodes
```

> This lab assumes at least 2 Nodes so you can compare tainted vs untainted scheduling behavior. On a single-node cluster, tainting your only Node will make ALL unmatched Pods stay Pending — useful for observing the effect, but remove the taint promptly afterward.

## 9.4 Lab 1 — Taint a Node

Pick a Node and taint it:

```bash
kubectl taint node <node-name> environment=production:NoSchedule
```

Verify:

```bash
kubectl describe node <node-name> | grep Taints
```

## 9.5 Confirm the Repulsion

Try scheduling a normal Pod (no toleration) — Kubernetes will avoid the tainted Node:

```bash
kubectl run untainted-test --image=nginx:1.27
kubectl get pod untainted-test -o wide
```

If you only have one Node (and it's now tainted), this Pod will stay `Pending` — confirming the taint is working.

## 9.6 Lab 2 — Add a Matching Toleration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tolerant-pod
spec:
  tolerations:
    - key: "environment"
      operator: "Equal"
      value: "production"
      effect: "NoSchedule"
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
  name: tolerant-pod
spec:
  tolerations:
    - key: "environment"
      operator: "Equal"
      value: "production"
      effect: "NoSchedule"
  containers:
    - name: nginx
      image: nginx:1.27
EOF

kubectl get pod tolerant-pod -o wide
```

This Pod **can** land on the tainted Node — but so can any other Node without a taint. The toleration removed the repulsion; it did not pin the Pod there. Combine with `nodeSelector` to force it there specifically:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tolerant-and-targeted-pod
spec:
  tolerations:
    - key: "environment"
      operator: "Equal"
      value: "production"
      effect: "NoSchedule"
  nodeSelector:
    kubernetes.io/hostname: <node-name>
  containers:
    - name: nginx
      image: nginx:1.27
```

## 9.7 Taint Effects Explained

| Effect | Behavior on new Pods | Behavior on already-running Pods |
|---|---|---|
| `NoSchedule` | Blocked unless tolerated | Unaffected — stays running |
| `PreferNoSchedule` | Scheduler tries to avoid, but will place if no better option | Unaffected |
| `NoExecute` | Blocked unless tolerated | **Evicted** unless tolerated (optionally with a grace period via `tolerationSeconds`) |

## 9.8 Lab 3 — PreferNoSchedule (Soft Repulsion)

```bash
kubectl taint node <node-name> workload=batch:PreferNoSchedule
```

Create a Pod with no toleration — it will still be allowed here if no better Node exists, unlike `NoSchedule`:

```bash
kubectl run soft-repel-test --image=nginx:1.27
kubectl get pod soft-repel-test -o wide
```

Remove this taint before continuing:

```bash
kubectl taint node <node-name> workload=batch:PreferNoSchedule-
```

## 9.9 Lab 4 — NoExecute and Live Eviction

Start a Pod on an **untainted** Node with a toleration ready to go (but the taint doesn't exist yet):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: noexecute-pod
spec:
  tolerations:
    - key: "maintenance"
      operator: "Equal"
      value: "true"
      effect: "NoExecute"
      tolerationSeconds: 30
  nodeSelector:
    kubernetes.io/hostname: <node-name>
  containers:
    - name: nginx
      image: nginx:1.27
```

Apply it and confirm it's Running on `<node-name>`:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: noexecute-pod
spec:
  tolerations:
    - key: "maintenance"
      operator: "Equal"
      value: "true"
      effect: "NoExecute"
      tolerationSeconds: 30
  nodeSelector:
    kubernetes.io/hostname: <node-name>
  containers:
    - name: nginx
      image: nginx:1.27
EOF

kubectl get pod noexecute-pod -o wide
```

Now apply the matching `NoExecute` taint live to that Node:

```bash
kubectl taint node <node-name> maintenance=true:NoExecute
```

Watch the Pod — because `tolerationSeconds: 30` is set, it tolerates the taint for 30 seconds, then gets evicted:

```bash
kubectl get pod noexecute-pod -w
```

Without `tolerationSeconds`, a matching toleration would tolerate the taint **indefinitely**. Without any toleration at all, a `NoExecute` taint evicts immediately.

Remove the taint afterward:

```bash
kubectl taint node <node-name> maintenance=true:NoExecute-
```

## 9.10 Built-In Taints You'll See in Real Clusters

Kubernetes automatically taints Nodes under certain conditions — you don't apply these yourself, but you need to recognize them when troubleshooting:

```
node.kubernetes.io/not-ready:NoExecute
node.kubernetes.io/unreachable:NoExecute
node.kubernetes.io/memory-pressure:NoSchedule
node.kubernetes.io/disk-pressure:NoSchedule
node.kubernetes.io/network-unavailable:NoSchedule
node.kubernetes.io/unschedulable:NoSchedule
```

Check for these on any Node:

```bash
kubectl describe node <node-name> | grep Taints
```

This is exactly why a Pod can suddenly go `Pending` or get evicted with no YAML change on your end — the Node itself became unhealthy and Kubernetes auto-tainted it.

## 9.11 Break It — Missing Toleration

```bash
kubectl taint node <node-name> dedicated=gpu-only:NoSchedule
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-toleration-pod
spec:
  nodeSelector:
    kubernetes.io/hostname: <node-name>
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
  name: no-toleration-pod
spec:
  nodeSelector:
    kubernetes.io/hostname: <node-name>
  containers:
    - name: nginx
      image: nginx:1.27
EOF
```

## 9.12 Diagnose and Recover

```bash
kubectl get pod no-toleration-pod
```

Status: `Pending`.

```bash
kubectl describe pod no-toleration-pod
```

Look for:

```
0/2 nodes are available: 1 node(s) had untolerated taint {dedicated: gpu-only}.
```

Fix by adding the matching toleration and reapplying:

```bash
kubectl delete pod no-toleration-pod

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: fixed-toleration-pod
spec:
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "gpu-only"
      effect: "NoSchedule"
  nodeSelector:
    kubernetes.io/hostname: <node-name>
  containers:
    - name: nginx
      image: nginx:1.27
EOF

kubectl get pod fixed-toleration-pod -o wide
```

Clean up the taint:

```bash
kubectl taint node <node-name> dedicated=gpu-only:NoSchedule-
```

## 9.13 CKA Practice Task

**Task**

1. Taint a Node with `team=ml:NoSchedule`.
2. Create a Pod named `no-tol-pod` with no toleration and confirm it avoids/stays Pending relative to that Node.
3. Create a Pod named `ml-workload` with a matching toleration AND a `nodeSelector` targeting that specific Node.
4. Verify `ml-workload` is Running on the tainted Node.
5. Remove the taint using the `-` suffix syntax.

**Target Time**

6 minutes

Try it without looking at previous commands.

## 9.14 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

What is the fundamental difference in "direction" between affinity and taints/tolerations?

**Question 2**

Does a matching Toleration guarantee a Pod will be scheduled on the tainted Node? Why or why not?

**Question 3**

What's the difference between `NoSchedule` and `NoExecute` for a Pod that is already running on the Node when the taint is applied?

**Question 4**

What does `tolerationSeconds` control, and what happens if it's omitted from a `NoExecute` toleration?

**Question 5**

Name two built-in taints Kubernetes applies automatically, and what condition triggers each.

**Question 6**

A Pod is `Pending` with the event `had untolerated taint`. What are your first two diagnostic commands, and what's the fix?

## 9.15 Useful Commands

```bash
kubectl taint node <name> <key>=<value>:<effect>
kubectl taint node <name> <key>=<value>:<effect>-   # remove

kubectl describe node <name> | grep Taints

kubectl get pod <name> -o wide
kubectl describe pod <name>
```

## 9.16 Cleanup

```bash
kubectl delete pod untainted-test tolerant-pod tolerant-and-targeted-pod \
  soft-repel-test noexecute-pod no-toleration-pod fixed-toleration-pod \
  no-tol-pod ml-workload

kubectl taint node <node-name> environment=production:NoSchedule-
kubectl taint node <node-name> maintenance=true:NoExecute-
kubectl taint node <node-name> dedicated=gpu-only:NoSchedule-
kubectl taint node <node-name> team=ml:NoSchedule-
```

## 9.17 Lab Checklist

- [ ] Tainted a Node with `NoSchedule`
- [ ] Confirmed an untolerated Pod avoids the tainted Node
- [ ] Added a matching Toleration and confirmed the Pod CAN (not must) schedule there
- [ ] Combined a Toleration with `nodeSelector` to force placement (dedicated Node pattern)
- [ ] Tested `PreferNoSchedule` soft repulsion
- [ ] Tested `NoExecute` with `tolerationSeconds` and observed live eviction
- [ ] Reviewed built-in taints (`not-ready`, `unreachable`, `memory-pressure`, etc.)
- [ ] Reproduced a Pod stuck `Pending` from an untolerated taint
- [ ] Diagnosed and fixed it with a matching Toleration
- [ ] Completed the CKA task within 6 minutes
