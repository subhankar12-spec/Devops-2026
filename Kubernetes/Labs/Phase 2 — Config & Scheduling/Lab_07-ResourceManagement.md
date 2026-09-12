# Kubernetes Hands-On Lab 07 — Resource Requests, Limits, LimitRanges & ResourceQuotas

## 7.1 Objectives

By the end of this lab, you should be able to:

- Set CPU/memory `requests` and `limits` on a container
- Understand how `requests` drive scheduling and `limits` drive enforcement
- Understand Kubernetes QoS classes: `Guaranteed`, `Burstable`, `BestEffort`
- Apply a `LimitRange` to set namespace-wide defaults and min/max bounds
- Apply a `ResourceQuota` to cap total namespace consumption
- Trigger and diagnose an `OOMKilled` container
- Trigger and diagnose a Pod stuck `Pending` due to insufficient cluster resources
- Perform common resource-management tasks quickly for the CKA

## 7.2 Architecture

```
Namespace
  |
  +-- ResourceQuota (hard cap on TOTAL cpu/memory/pods in the namespace)
  |
  +-- LimitRange (default + min/max requests/limits PER container/pod)
  |
  +-- Pod
        |
        +-- Container
              requests: { cpu: 250m, memory: 128Mi }  <- used by scheduler
              limits:   { cpu: 500m, memory: 256Mi }  <- enforced at runtime

Scheduler decision: "does any Node have >= requests available?"
Runtime enforcement:
  - CPU limit exceeded  -> container is THROTTLED (not killed)
  - Memory limit exceeded -> container is OOMKilled (terminated)
```

Key concept: `requests` are a **scheduling promise** (the Node must have this much free) — `limits` are a **runtime ceiling** enforced by the kubelet/container runtime. CPU is compressible (throttled); memory is not (OOMKilled).

## 7.3 Prerequisites

```bash
kubectl version --client
kubectl get nodes
kubectl top nodes
```

> `kubectl top` requires metrics-server. If it's not installed, `describe node` still shows allocatable capacity.

## 7.4 Lab 1 — Requests and Limits

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resourced-pod
spec:
  containers:
    - name: app
      image: nginx:1.27
      resources:
        requests:
          cpu: "250m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: resourced-pod
spec:
  containers:
    - name: app
      image: nginx:1.27
      resources:
        requests:
          cpu: "250m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
EOF
```

## 7.5 Inspect Resource Allocation

```bash
kubectl describe pod resourced-pod
```

Look for the `Requests:` and `Limits:` lines under the container, and `QoS Class:` at the bottom.

Check live usage (needs metrics-server):

```bash
kubectl top pod resourced-pod
```

Check what a Node has allocated vs allocatable:

```bash
kubectl describe node <node-name>
```

Look for the `Allocated resources:` section near the bottom.

## 7.6 QoS Classes

Kubernetes assigns one of three Quality of Service classes automatically based on how requests/limits are set — you never set QoS directly.

| QoS Class | How it's assigned | Eviction priority under Node pressure |
|---|---|---|
| `Guaranteed` | Every container has `requests == limits` for both CPU and memory | Evicted last (safest) |
| `Burstable` | At least one container has a request or limit set, but not equal on both | Evicted after BestEffort, before Guaranteed |
| `BestEffort` | No requests or limits set at all | Evicted first |

Create one Pod of each class and confirm:

```bash
# Guaranteed
kubectl run qos-guaranteed --image=nginx:1.27 \
  --requests=cpu=200m,memory=128Mi --limits=cpu=200m,memory=128Mi

# Burstable
kubectl run qos-burstable --image=nginx:1.27 \
  --requests=cpu=100m,memory=64Mi --limits=cpu=200m,memory=128Mi

# BestEffort
kubectl run qos-besteffort --image=nginx:1.27
```

Check each:

```bash
kubectl get pod qos-guaranteed -o jsonpath='{.status.qosClass}'
kubectl get pod qos-burstable -o jsonpath='{.status.qosClass}'
kubectl get pod qos-besteffort -o jsonpath='{.status.qosClass}'
```

## 7.7 Lab 2 — LimitRange (Namespace Defaults)

Create a namespace and a LimitRange for it:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lab07-limits
---
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: lab07-limits
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "250m"
        memory: "128Mi"
      max:
        cpu: "1"
        memory: "512Mi"
      min:
        cpu: "100m"
        memory: "64Mi"
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: lab07-limits
---
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: lab07-limits
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "250m"
        memory: "128Mi"
      max:
        cpu: "1"
        memory: "512Mi"
      min:
        cpu: "100m"
        memory: "64Mi"
EOF
```

Create a Pod with **no** resources specified in this namespace, and see the LimitRange fill in defaults automatically:

```bash
kubectl run auto-limited --image=nginx:1.27 -n lab07-limits
kubectl get pod auto-limited -n lab07-limits -o jsonpath='{.spec.containers[0].resources}'
```

Try to create a Pod that violates the `max` — this should be rejected:

```bash
kubectl run oversized --image=nginx:1.27 -n lab07-limits \
  --requests=cpu=2,memory=1Gi --limits=cpu=2,memory=1Gi
```

Expect an error like `maximum cpu usage per Container is 1, but limit is 2`.

## 7.8 Lab 3 — ResourceQuota (Namespace Totals)

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: lab07-quota
  namespace: lab07-limits
spec:
  hard:
    pods: "5"
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: lab07-quota
  namespace: lab07-limits
spec:
  hard:
    pods: "5"
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
EOF
```

Check current usage against the quota:

```bash
kubectl describe resourcequota lab07-quota -n lab07-limits
```

> Important: once a ResourceQuota exists in a namespace, **every** new Pod in that namespace must explicitly specify requests/limits for the resources the quota covers — the LimitRange defaults from 7.7 are what make this work seamlessly together.

Try to exceed the quota by creating enough Pods to cross `requests.cpu: 1` (each auto-limited Pod requests `250m`, so a 5th would push past):

```bash
for i in 1 2 3 4 5; do
  kubectl run quota-test-$i --image=nginx:1.27 -n lab07-limits
done

kubectl get pods -n lab07-limits
```

Watch one get rejected once the quota is exceeded — check the event:

```bash
kubectl describe resourcequota lab07-quota -n lab07-limits
```

## 7.9 Trigger an OOMKilled Container

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: oom-demo
spec:
  containers:
    - name: memory-hog
      image: polinux/stress
      resources:
        limits:
          memory: "50Mi"
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: oom-demo
spec:
  containers:
    - name: memory-hog
      image: polinux/stress
      resources:
        limits:
          memory: "50Mi"
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
EOF
```

Watch it get killed and restarted:

```bash
kubectl get pod oom-demo -w
```

## 7.10 Diagnose the OOMKilled Container

```bash
kubectl describe pod oom-demo
```

Look for:

```
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
```

Check restart count:

```bash
kubectl get pod oom-demo -o jsonpath='{.status.containerStatuses[0].restartCount}'
```

Fix it by raising the memory limit above what the container actually needs:

```bash
kubectl delete pod oom-demo
kubectl run oom-demo --image=polinux/stress --limits=memory=300Mi \
  --command -- stress --vm 1 --vm-bytes 150M --vm-hang 1
kubectl get pod oom-demo
```

## 7.11 Trigger a Pod Stuck Pending (Insufficient Resources)

Request far more CPU than any Node has allocatable:

```bash
kubectl run impossible-pod --image=nginx:1.27 --requests=cpu=100
```

Check status:

```bash
kubectl get pod impossible-pod
```

It stays `Pending` indefinitely.

## 7.12 Diagnose the Pending Pod

```bash
kubectl describe pod impossible-pod
```

Look for an Event from the scheduler:

```
0/2 nodes are available: 2 Insufficient cpu.
```

Check what each Node actually has available:

```bash
kubectl describe nodes | grep -A 5 "Allocated resources"
```

Fix by deleting and requesting something realistic:

```bash
kubectl delete pod impossible-pod
kubectl run realistic-pod --image=nginx:1.27 --requests=cpu=100m
kubectl get pod realistic-pod
```

## 7.13 CKA Practice Task

**Task**

1. Create a namespace `cka-resources`.
2. Apply a LimitRange to it with `defaultRequest` cpu=`200m`/memory=`128Mi` and `max` cpu=`800m`/memory=`512Mi`.
3. Apply a ResourceQuota to it capping `pods: "3"` and `requests.cpu: "600m"`.
4. Create a Pod with no resources specified and confirm the LimitRange defaults were applied.
5. Attempt to create a 4th Pod and confirm it's rejected by the quota.
6. Identify the QoS class of the Pod from step 4.

**Target Time**

7 minutes

Try it without looking at previous commands.

## 7.14 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

What is the practical difference between a container exceeding its CPU limit versus exceeding its memory limit?

**Question 2**

Which field — `requests` or `limits` — does the scheduler use to decide which Node a Pod can run on?

**Question 3**

A Pod is stuck `Pending`. What are the two most common resource-related causes, and how do you tell them apart?

**Question 4**

What QoS class does a Pod get if it sets `requests.cpu=200m` and `limits.cpu=500m` but no memory values at all?

**Question 5**

If a namespace has a ResourceQuota but a new Pod is submitted without specifying any resources, what happens, and why does a LimitRange matter here?

**Question 6**

What exit code corresponds to `OOMKilled`, and where do you find it?

## 7.15 Useful Commands

```bash
kubectl top nodes
kubectl top pod <name>

kubectl describe node <name>
kubectl describe pod <name>

kubectl get pod <name> -o jsonpath='{.status.qosClass}'

kubectl describe limitrange <name> -n <namespace>
kubectl describe resourcequota <name> -n <namespace>

kubectl run <name> --image=<image> \
  --requests=cpu=<val>,memory=<val> \
  --limits=cpu=<val>,memory=<val>
```

## 7.16 Cleanup

```bash
kubectl delete pod resourced-pod qos-guaranteed qos-burstable qos-besteffort \
  oom-demo impossible-pod realistic-pod

kubectl delete namespace lab07-limits
kubectl delete namespace cka-resources
```

## 7.17 Lab Checklist

- [ ] Set requests and limits on a container
- [ ] Understood requests (scheduling) vs limits (runtime enforcement)
- [ ] Identified Guaranteed, Burstable, and BestEffort QoS classes
- [ ] Applied a LimitRange and saw defaults auto-injected
- [ ] Confirmed a LimitRange `max` rejects an oversized Pod
- [ ] Applied a ResourceQuota and viewed usage against it
- [ ] Confirmed a ResourceQuota rejects a Pod once exceeded
- [ ] Reproduced and diagnosed an `OOMKilled` container
- [ ] Reproduced and diagnosed a `Pending` Pod from insufficient CPU
- [ ] Completed the CKA task within 7 minutes
