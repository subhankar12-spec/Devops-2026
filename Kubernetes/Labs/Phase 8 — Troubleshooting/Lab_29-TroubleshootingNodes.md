# Kubernetes Hands-On Lab 29 — Troubleshooting Node Issues

## 29.1 Objectives

By the end of this lab, you should be able to:

- Read and interpret all standard Node `Conditions` (`Ready`, `MemoryPressure`, `DiskPressure`, `PIDPressure`, `NetworkUnavailable`)
- Diagnose a Node stuck `NotReady` systematically (kubelet, container runtime, network)
- Use `journalctl` and `systemctl` to inspect and restart the kubelet directly on a Node
- Diagnose a Node reporting `DiskPressure` and understand what triggers it
- Understand what happens to existing Pods when a Node goes `NotReady` (eviction timing)
- Safely cordon, drain, and return a problem Node to service
- Diagnose a Node that's `Ready` but silently failing to schedule new Pods
- Perform common Node troubleshooting tasks quickly for the CKA

## 29.2 The Systematic Node Diagnostic Method

```
1. kubectl get nodes                        -> STATUS column: Ready / NotReady / SchedulingDisabled?
2. kubectl describe node <name>             -> Conditions (bottom-up), Allocated resources, Events
3. SSH/shell into the Node itself:
   systemctl status kubelet                 -> is the kubelet process even running?
   journalctl -u kubelet -f                 -> live kubelet logs — the PRIMARY source of truth
   systemctl status containerd              -> is the container runtime running?
   df -h                                    -> disk space (DiskPressure root cause)
   free -h                                  -> memory (MemoryPressure root cause)
4. kubectl get pods -o wide --all-namespaces | grep <node-name>  -> what was scheduled here?
```

Key concept: `kubectl describe node` tells you WHAT is wrong (a Condition is `True`/`Unknown` when it shouldn't be) — but `journalctl -u kubelet` on the Node itself tells you WHY. Many people get stuck because they only ever run `kubectl` commands and never actually get a shell on the affected Node — but the kubelet isn't a Pod, so `kubectl logs` cannot see its logs.

## 29.3 Node Conditions Reference

| Condition | `True` means... |
|---|---|
| `Ready` | kubelet is healthy and ready to accept Pods (this is the one everyone checks first) |
| `MemoryPressure` | Node is running low on available memory |
| `DiskPressure` | Node is running low on disk space (or inode count) |
| `PIDPressure` | Node is running low on available process IDs |
| `NetworkUnavailable` | Node's network is not correctly configured (common right after join, before CNI settles) |

`Ready` can also show `Unknown` — this specifically means the control plane hasn't heard from this Node's kubelet recently (network partition, kubelet crashed, Node powered off) — distinct from `False`, which means the kubelet actively reported itself unhealthy.

## 29.4 Prerequisites

```bash
kubectl version --client
kubectl get nodes
```

This lab requires shell access to at least one worker Node for the deeper diagnostic steps.

## 29.5 Lab 1 — Baseline Node Health Check

```bash
kubectl get nodes
kubectl describe node <worker-node-name>
```

Read through the entire `Conditions:` block — under healthy operation, `Ready` is `True` and all pressure conditions are `False`.

Check current resource allocation:

```bash
kubectl describe node <worker-node-name> | grep -A 10 "Allocated resources"
```

## 29.6 Scenario 1 — kubelet Service Stopped

On the target worker Node:

```bash
sudo systemctl stop kubelet
```

From the control plane, watch the Node's status:

```bash
kubectl get nodes -w
```

## 29.7 Diagnose the Stopped kubelet

Initially you'll see `Ready` transition to `Unknown` (not `False`) — the control plane hasn't heard from this kubelet in a while, but doesn't yet know FOR CERTAIN it's down versus just network-partitioned:

```bash
kubectl describe node <worker-node-name>
```

Look at the `Ready` condition's `Reason:` — likely `NodeStatusUnknown`, with a message about not receiving updates.

On the Node itself, confirm directly:

```bash
sudo systemctl status kubelet
```

`inactive (dead)` confirms the exact cause.

## 29.8 Recover From Scenario 1

```bash
sudo systemctl start kubelet
sudo systemctl status kubelet
```

Watch recovery from the control plane:

```bash
kubectl get nodes -w
```

Confirm via kubelet's own logs that it re-registered cleanly:

```bash
sudo journalctl -u kubelet -n 50 --no-pager
```

## 29.9 Scenario 2 — Container Runtime Down

On the worker Node:

```bash
sudo systemctl stop containerd
```

From the control plane:

```bash
kubectl get nodes
kubectl describe node <worker-node-name>
```

## 29.10 Diagnose the Stopped Runtime

The kubelet process is still running (unlike Scenario 1), but it can't do anything useful without a container runtime:

```bash
sudo systemctl status kubelet
```

`active (running)` — but check its logs:

```bash
sudo journalctl -u kubelet -n 50 --no-pager
```

Look for repeated errors like:

```
Failed to get status for pod ...: rpc error: code = Unavailable desc = connection error
```

or

```
container runtime is down
```

This is exactly why step 3 in the diagnostic method (29.2) checks BOTH kubelet and containerd separately — a `Ready: True` kubelet with a dead runtime produces different symptoms than a fully-dead kubelet.

## 29.11 Recover From Scenario 2

```bash
sudo systemctl start containerd
sudo systemctl status containerd
```

The kubelet should self-heal once the runtime is back (no kubelet restart needed, though restarting it is harmless if recovery seems slow):

```bash
kubectl get nodes -w
```

## 29.12 Scenario 3 — Simulated DiskPressure

Fill up disk space on the Node to trigger `DiskPressure` (⚠️ use a disposable/test Node, and a small dummy file — don't actually fill a real disk to 100% carelessly):

```bash
df -h /
sudo fallocate -l $(($(df --output=avail / | tail -1) * 90 / 100))K /tmp/pressure-test-file
df -h /
```

Wait a minute or two for the kubelet's periodic Node status update, then check:

```bash
kubectl describe node <worker-node-name> | grep -A 2 DiskPressure
```

## 29.13 Diagnose DiskPressure

```bash
kubectl get nodes
```

Under real `DiskPressure`, the Node's scheduler eligibility changes and the kubelet begins garbage-collecting unused images/containers to reclaim space — and new Pods will be **evicted or refused scheduling** here in real conditions, since the kubelet actively protects itself from a full disk.

```bash
kubectl describe node <worker-node-name>
```

Check `Events:` for any eviction activity if existing Pods were already running here.

## 29.14 Recover From DiskPressure

```bash
sudo rm /tmp/pressure-test-file
df -h /
```

Confirm the condition clears (may take a minute for the next kubelet status report):

```bash
kubectl describe node <worker-node-name> | grep -A 2 DiskPressure
```

## 29.15 Pod Eviction Timing When a Node Goes NotReady

When a Node's `Ready` condition becomes `False` or `Unknown`, Kubernetes doesn't evict Pods immediately — there's a grace period (`--pod-eviction-timeout` on the controller-manager, default 5 minutes) to avoid mass rescheduling from brief blips:

```bash
kubectl get pods -o wide --all-namespaces | grep <worker-node-name>
```

If the Node stays unreachable past that window, affected Pods are marked for deletion and (if managed by a Deployment/ReplicaSet/etc.) recreated elsewhere. This is exactly why a brief Node hiccup usually self-heals invisibly, while a genuinely dead Node eventually results in workload migration — understanding this timing prevents both false alarms and impatient manual intervention during transient issues.

## 29.16 Safely Take a Node Out of Service for Maintenance

The correct sequence — never just stop the kubelet on a Node with live traffic without doing this first:

```bash
kubectl cordon <worker-node-name>
kubectl get nodes
```

`SchedulingDisabled` appears next to the Node — existing Pods keep running, but no NEW Pods will land here.

```bash
kubectl drain <worker-node-name> --ignore-daemonsets --delete-emptydir-data
```

Perform your maintenance (patching, reboot, hardware work), then return it to service:

```bash
kubectl uncordon <worker-node-name>
kubectl get nodes
```

## 29.17 Break It — Node Ready But Pods Won't Schedule (Taint Left Behind)

Simulate a Node that looks perfectly healthy but silently refuses new Pods:

```bash
kubectl taint node <worker-node-name> maintenance-leftover=true:NoSchedule
```

```bash
kubectl get nodes
```

Status shows `Ready` — no obvious problem from `get nodes` alone.

```bash
kubectl run stuck-test-pod --image=nginx:1.27
kubectl get pod stuck-test-pod -o wide
```

If this is your only Node, it stays `Pending`.

## 29.18 Diagnose and Recover

```bash
kubectl describe pod stuck-test-pod
```

Look for:

```
0/1 nodes are available: 1 node(s) had untolerated taint {maintenance-leftover: true}.
```

Cross-check the Node directly:

```bash
kubectl describe node <worker-node-name> | grep Taints
```

This is a very common real-world scenario: someone cordoned/tainted a Node for maintenance, finished the work, but forgot to remove the taint (as opposed to `uncordon`, which only removes the auto-applied `node.kubernetes.io/unschedulable` taint — a manually-applied custom taint like this one needs manual removal). Fix:

```bash
kubectl taint node <worker-node-name> maintenance-leftover=true:NoSchedule-
kubectl get pod stuck-test-pod -o wide
```

## 29.19 CKA Practice Task

**Task**

(Mix of live and command-recall.)

1. Run `kubectl describe node <any-node>` and identify the current state of all 5 standard Conditions.
2. Write the exact `journalctl` command to tail live kubelet logs.
3. Write the sequence of 3 commands to safely take a Node out of service, do maintenance, and return it.
4. A Node shows `Ready` but a new Pod stays `Pending` with an untolerated-taint event. Write the two commands you'd use to identify and remove the offending taint.
5. Explain the difference between a Node's `Ready` condition showing `False` versus `Unknown`.

**Target Time**

8 minutes

## 29.20 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

Why can't `kubectl logs` show you kubelet errors, and what command do you use instead?

**Question 2**

What's the practical difference between a Node's kubelet being stopped versus its container runtime being stopped, in terms of symptoms?

**Question 3**

What does `DiskPressure: True` cause the kubelet to actively do, beyond just reporting the condition?

**Question 4**

Why doesn't Kubernetes immediately reschedule Pods the instant a Node's `Ready` condition flips to `False`?

**Question 5**

Does `kubectl uncordon` remove ALL taints from a Node? What's the distinction that matters here?

**Question 6**

A Node shows `Ready: Unknown` rather than `Ready: False`. What does this specific distinction tell you about what the control plane actually knows?

## 29.21 Useful Commands

```bash
kubectl get nodes
kubectl get nodes -o wide
kubectl describe node <name>

kubectl cordon <name>
kubectl drain <name> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <name>

kubectl taint node <name> <key>=<value>:<effect>
kubectl taint node <name> <key>=<value>:<effect>-

# On the Node itself:
sudo systemctl status kubelet
sudo systemctl restart kubelet
sudo journalctl -u kubelet -f
sudo journalctl -u kubelet -n 50 --no-pager
sudo systemctl status containerd
sudo systemctl restart containerd
df -h
free -h
```

## 29.22 Cleanup

```bash
kubectl delete pod stuck-test-pod
kubectl taint node <worker-node-name> maintenance-leftover=true:NoSchedule- 2>/dev/null || true
sudo rm -f /tmp/pressure-test-file
```

## 29.23 Lab Checklist

- [ ] Read and interpreted all 5 Node Conditions on a healthy Node
- [ ] Simulated a stopped kubelet and observed `Ready: Unknown`
- [ ] Diagnosed via `journalctl -u kubelet` and recovered
- [ ] Simulated a stopped container runtime and distinguished its symptoms from a stopped kubelet
- [ ] Simulated and diagnosed `DiskPressure`
- [ ] Understood Pod eviction timing after a Node goes `NotReady`
- [ ] Safely cordoned, drained, and uncordoned a Node for maintenance
- [ ] Reproduced a Ready-but-tainted Node silently refusing new Pods
- [ ] Diagnosed via `describe pod`/`describe node` and removed the leftover taint
- [ ] Completed the CKA task within 8 minutes
