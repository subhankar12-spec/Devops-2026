# Kubernetes Hands-On Lab 03 — ReplicaSets & ReplicationControllers

## 3.1 Objectives

By the end of this lab, you should be able to:

- Create a ReplicaSet using YAML
- Understand how a ReplicaSet's selector controls which Pods it manages
- Scale a ReplicaSet
- Understand what a ReplicationController is and why ReplicaSets replaced it
- Observe self-healing when a Pod is deleted
- Understand why editing a ReplicaSet's Pod template does NOT update existing Pods
- Adopt an existing, unmanaged Pod into a ReplicaSet via label matching
- Troubleshoot a ReplicaSet stuck at `0/3` ready replicas
- Perform common ReplicaSet tasks quickly for the CKA

## 3.2 Architecture

```
                  ReplicaSet
                 (desired: 3)
                      |
        selector: app=frontend
                      |
        +-------------+-------------+
        |             |             |
        v             v             v
      Pod-1         Pod-2         Pod-3
   (app=frontend) (app=frontend) (app=frontend)

The ReplicaSet controller constantly reconciles:
  actual Pods matching selector  vs.  spec.replicas
If actual < desired -> create Pods
If actual > desired -> delete Pods
```

Key concept: a ReplicaSet does **not** track Pods by name or ownership alone — it tracks them by **label selector**. Any Pod matching the selector counts toward the replica count, whether the ReplicaSet created it or not.

## 3.3 Prerequisites

```bash
kubectl version --client
kubectl get nodes
```

## 3.4 Lab 1 — Create a ReplicaSet

Create `replicaset.yaml`:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend-rs
  labels:
    app: frontend
spec:
  replicas: 3

  selector:
    matchLabels:
      app: frontend

  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

Apply it:

```bash
kubectl apply -f replicaset.yaml
```

## 3.5 Verify the ReplicaSet

```bash
kubectl get rs
kubectl get pods --show-labels
```

Or combined:

```bash
kubectl get rs,pods -l app=frontend
```

Inspect it:

```bash
kubectl describe rs frontend-rs
```

Check:

- `Replicas:` (desired / current / ready)
- `Selector:`
- `Pod Template:`
- `Events:` (Pod creation events)

## 3.6 Self-Healing — Delete a Pod

Pick one Pod and delete it:

```bash
kubectl get pods -l app=frontend
kubectl delete pod <one-of-the-pod-names>
```

Immediately check again:

```bash
kubectl get pods -l app=frontend
```

A replacement Pod appears within seconds — the ReplicaSet noticed actual (2) < desired (3) and reconciled. This is the core self-healing behavior every higher-level controller (Deployment, StatefulSet, DaemonSet) builds on.

## 3.7 Scale the ReplicaSet

Scale up:

```bash
kubectl scale rs frontend-rs --replicas=5
kubectl get pods -l app=frontend
```

Scale down:

```bash
kubectl scale rs frontend-rs --replicas=2
kubectl get pods -l app=frontend
```

Note: when scaling down, Kubernetes doesn't guarantee *which* Pods get deleted (it prefers newer/not-ready Pods first, but don't rely on a specific one surviving).

## 3.8 Editing the Template Does NOT Update Existing Pods

Edit the ReplicaSet's image:

```bash
kubectl edit rs frontend-rs
```

Change `image: nginx:1.27` to `image: nginx:1.28`, save and exit.

Check existing Pods — they are still running the OLD image:

```bash
kubectl get pods -l app=frontend -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.spec.containers[0].image}{"\n"}{end}'
```

This is the single biggest reason Deployments exist: a ReplicaSet has **no rollout mechanism**. To roll the new image out, you'd have to delete Pods manually and let the ReplicaSet recreate them with the new template:

```bash
kubectl delete pod -l app=frontend
kubectl get pods -l app=frontend -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.spec.containers[0].image}{"\n"}{end}'
```

Now they come back on `nginx:1.28` — but this was a manual, disruptive, all-at-once replacement, not a rolling update.

## 3.9 Adopt an Existing Pod

Create a standalone Pod with a matching label but not created by the ReplicaSet:

```bash
kubectl run adopted-pod --image=nginx:1.28 --labels="app=frontend"
```

Check the ReplicaSet — notice it now shows more current replicas than desired, and will delete one Pod to reconcile back down to `spec.replicas`:

```bash
kubectl get rs frontend-rs
kubectl get pods -l app=frontend
```

This demonstrates that ReplicaSets manage by **selector match**, not by creation history — a dangerous trap if a manually created Pod accidentally shares labels with a ReplicaSet's selector.

## 3.10 ReplicationController — What It Was and Why It's Gone

`ReplicationController` (RC) is the original, older controller for identical Pods (`apiVersion: v1, kind: ReplicationController`). It's functionally similar to a ReplicaSet but:

| | ReplicationController | ReplicaSet |
|---|---|---|
| API version | `v1` | `apps/v1` |
| Selector type | Equality-based only (`matchLabels`-style key=value) | Equality-based **and** set-based (`matchExpressions`) |
| Used directly today | Rarely — legacy only | Rarely directly — almost always managed by a Deployment |
| Recommended | No | Use a Deployment, which manages ReplicaSets for you |

You will not create an RC or a standalone ReplicaSet in real production work — you'll always go through a Deployment (Lab 04) so you get rolling updates, rollback, and revision history for free. The CKA exam still expects you to *know* the RC → ReplicaSet → Deployment lineage and be able to spot/manage a bare ReplicaSet if one shows up in a task.

## 3.11 Break It — ReplicaSet Stuck at 0 Ready

Create a broken ReplicaSet with a bad image:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: broken-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: broken-demo
  template:
    metadata:
      labels:
        app: broken-demo
    spec:
      containers:
        - name: app
          image: nginx:does-not-exist-tag
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: broken-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: broken-demo
  template:
    metadata:
      labels:
        app: broken-demo
    spec:
      containers:
        - name: app
          image: nginx:does-not-exist-tag
EOF
```

Check status:

```bash
kubectl get rs broken-rs
kubectl get pods -l app=broken-demo
```

You'll see `DESIRED 3`, `CURRENT 3`, `READY 0` — the ReplicaSet successfully created Pods, but none of them are Ready.

## 3.12 Diagnose and Recover

```bash
kubectl describe rs broken-rs
kubectl describe pod -l app=broken-demo
```

Look for `ImagePullBackOff` / `ErrImagePull` in `Events:`. Fix it:

```bash
kubectl set image rs/broken-rs app=nginx:1.28
```

> Note: unlike a Deployment, `kubectl set image` on a bare ReplicaSet only updates the template — existing broken Pods won't self-correct. You still need to delete them:

```bash
kubectl delete pod -l app=broken-demo
kubectl get pods -l app=broken-demo
```

## 3.13 CKA Practice Task

**Task**

1. Create a ReplicaSet named `cka-rs` with 4 replicas, image `nginx:1.28`, selector/labels `app=cka-demo`.
2. Verify all 4 Pods reach `Running` and `Ready`.
3. Delete one Pod and confirm the ReplicaSet replaces it within seconds.
4. Scale the ReplicaSet down to 2 replicas.
5. Manually create a standalone Pod with label `app=cka-demo` and observe the ReplicaSet delete a Pod to reconcile back to 2.

**Target Time**

5 minutes

Try it without looking at previous commands.

## 3.14 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

Why does deleting a Pod managed by a ReplicaSet result in a new Pod appearing almost immediately?

**Question 2**

If you edit a ReplicaSet's `spec.template.spec.containers[0].image`, what happens to Pods that already exist?

**Question 3**

What could cause a ReplicaSet to show `DESIRED 3 / CURRENT 3 / READY 0`?

**Question 4**

Why is it risky for a manually created Pod to share labels with an existing ReplicaSet's selector?

**Question 5**

What capability does a Deployment add on top of a ReplicaSet that makes ReplicaSets rarely used directly?

**Question 6**

What is the main functional difference between a ReplicationController and a ReplicaSet?

## 3.15 Useful Commands

```bash
kubectl get rs
kubectl get rs -o wide
kubectl describe rs <name>

kubectl scale rs <name> --replicas=<n>

kubectl set image rs/<name> <container>=<image>

kubectl get pods -l <selector>
kubectl delete pod -l <selector>
kubectl delete pod <name>

kubectl run <pod-name> --image=<image> --labels="<key>=<value>"

kubectl edit rs <name>
kubectl delete rs <name>
```

## 3.16 Cleanup

```bash
kubectl delete rs frontend-rs
kubectl delete rs broken-rs
kubectl delete rs cka-rs
kubectl delete pod adopted-pod
```

## 3.17 Lab Checklist

- [ ] Created a ReplicaSet using YAML
- [ ] Verified desired/current/ready replica counts
- [ ] Observed self-healing after deleting a Pod
- [ ] Scaled a ReplicaSet up and down
- [ ] Confirmed editing the template does not update existing Pods
- [ ] Manually replaced Pods to pick up a template change
- [ ] Adopted an unmanaged Pod into a ReplicaSet via matching labels
- [ ] Understood the ReplicationController → ReplicaSet → Deployment lineage
- [ ] Reproduced a ReplicaSet stuck at 0 ready replicas
- [ ] Diagnosed and fixed the broken image
- [ ] Completed the CKA task within 5 minutes
