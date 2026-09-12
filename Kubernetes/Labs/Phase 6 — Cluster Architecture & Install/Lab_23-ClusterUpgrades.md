# Kubernetes Hands-On Lab 23 — Cluster Upgrades (kubeadm upgrade workflow)

## 23.1 Objectives

By the end of this lab, you should be able to:

- Understand the version skew policy and why upgrades must go one minor version at a time
- Determine the correct upgrade order: control plane first, then workers
- Use `kubeadm upgrade plan` to preview an upgrade
- Upgrade the control-plane Node's `kubeadm`, then apply the upgrade, then `kubelet`/`kubectl`
- Safely drain and cordon a Node before upgrading it
- Upgrade a worker Node
- Uncordon a Node after upgrade to return it to service
- Troubleshoot an upgrade that fails partway through
- Perform common upgrade tasks quickly for the CKA

## 23.2 Architecture

```
Upgrade order (always this direction, one minor version per hop):

  1. Upgrade kubeadm ITSELF on the control-plane Node (apt/package upgrade)
                    |
                    v
  2. kubeadm upgrade plan          <- preview: what's compatible, what's available
                    |
                    v
  3. kubeadm upgrade apply v1.X.Y   <- upgrades control plane components + CoreDNS + kube-proxy
                    |
                    v
  4. drain control-plane Node -> upgrade kubelet + kubectl -> restart kubelet -> uncordon
                    |
                    v
  5. REPEAT per worker Node:
     drain -> kubeadm upgrade node -> upgrade kubelet + kubectl -> restart kubelet -> uncordon
```

Key concept: you cannot skip minor versions (1.30 → 1.32 directly is unsupported — you must go 1.30 → 1.31 → 1.32). The kubelet may lag up to **three** minor versions behind the API server (as of kubelet 1.25+), which is exactly why worker Nodes can be upgraded more gradually/later than the control plane, but the control plane itself must be upgraded first and one minor version at a time.

## 23.3 Prerequisites

```bash
kubectl version
kubectl get nodes
```

Check current versions across all Nodes:

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion
```

> This lab assumes a kubeadm-created cluster (from Lab 22) currently on some version 1.X, upgrading to 1.X+1. Substitute your actual current/target versions throughout.

## 23.4 Lab 1 — Upgrade kubeadm on the Control Plane

On the control-plane Node, update the package repo to the new minor version and upgrade `kubeadm` first (before anything else):

```bash
sudo apt-get update
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm='1.31.0-*'
sudo apt-mark hold kubeadm
```

Verify:

```bash
kubeadm version
```

## 23.5 Preview the Upgrade

```bash
sudo kubeadm upgrade plan
```

This shows:

- The current cluster version
- Available target versions
- Which components will be upgraded
- Any manual actions required before proceeding (e.g. CoreDNS/kube-proxy version bumps)

Read the output carefully — this is the single most useful command for confirming you're about to do exactly what you expect.

## 23.6 Lab 2 — Apply the Control-Plane Upgrade

```bash
sudo kubeadm upgrade apply v1.31.0
```

Confirm when prompted. This upgrades the control-plane static Pods (`kube-apiserver`, `kube-scheduler`, `kube-controller-manager`), and (on the first control-plane Node) also updates the CoreDNS and kube-proxy manifests cluster-wide.

Verify:

```bash
kubectl get nodes
kubectl -n kube-system get pods
```

Note: `kubectl get nodes` still shows the **old** kubelet version at this point — `kubeadm upgrade apply` only touches the control-plane components, not the kubelet itself.

## 23.7 Lab 3 — Drain and Upgrade the Control-Plane Node's kubelet

Cordon and drain the control-plane Node (mark unschedulable, evict/reschedule its workload Pods elsewhere):

```bash
kubectl drain <control-plane-node-name> --ignore-daemonsets --delete-emptydir-data
```

`--ignore-daemonsets` is required because DaemonSet Pods can't be "evicted" in the normal sense — they're meant to run on every Node. `--delete-emptydir-data` acknowledges that any `emptyDir`-backed data on this Node's Pods will be lost (expected, since we've covered in Lab 16 that `emptyDir` isn't meant to survive Pod rescheduling anyway).

Now upgrade `kubelet` and `kubectl`:

```bash
sudo apt-get update
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet='1.31.0-*' kubectl='1.31.0-*'
sudo apt-mark hold kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

## 23.8 Uncordon the Control-Plane Node

```bash
kubectl uncordon <control-plane-node-name>
```

Verify the version updated:

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion
```

## 23.9 Lab 4 — Upgrade a Worker Node

On the control plane, drain the worker first:

```bash
kubectl drain <worker-node-name> --ignore-daemonsets --delete-emptydir-data
```

On the **worker Node itself**, upgrade `kubeadm` and run the node-level upgrade command (note: workers use `kubeadm upgrade node`, not `kubeadm upgrade apply` — that command is control-plane-only):

```bash
sudo apt-get update
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm='1.31.0-*'
sudo apt-mark hold kubeadm

sudo kubeadm upgrade node
```

Then upgrade kubelet/kubectl exactly as in 23.7:

```bash
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet='1.31.0-*' kubectl='1.31.0-*'
sudo apt-mark hold kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

Back on the control plane, uncordon it:

```bash
kubectl uncordon <worker-node-name>
```

Repeat 23.9 for each remaining worker Node, one at a time — never drain multiple Nodes simultaneously in a small cluster, or you may exhaust capacity for rescheduled Pods.

## 23.10 Verify the Full Cluster Is Upgraded

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion
```

All Nodes should now show the new version, and all should be `Ready`.

## 23.11 Break It — Upgrade Skipping a Minor Version

Attempt an unsupported jump (e.g. from 1.31 straight to 1.33):

```bash
sudo kubeadm upgrade plan
```

kubeadm will refuse or warn heavily, because it explicitly enforces the one-minor-version-at-a-time rule to protect API compatibility. Attempting `kubeadm upgrade apply v1.33.0` directly from 1.31 typically errors out with a version-skew validation failure before touching anything.

## 23.12 Diagnose an Upgrade That Fails Partway

If `kubeadm upgrade apply` fails mid-way (e.g. due to a failed health check or an interrupted network connection):

```bash
kubectl -n kube-system get pods
kubectl get nodes
sudo kubeadm upgrade plan
```

`kubeadm upgrade apply` is designed to be **safely re-runnable** — running it again with the same target version will resume/retry rather than corrupt state, since it checks the current state of static Pod manifests before making changes. Check static Pod manifest timestamps if you suspect a partial apply:

```bash
sudo ls -la /etc/kubernetes/manifests/
```

If a specific static Pod is crash-looping post-upgrade, check its logs the same way you would in Lab 27 (control-plane troubleshooting):

```bash
sudo crictl ps -a | grep -E "apiserver|scheduler|controller-manager"
sudo crictl logs <container-id>
```

## 23.13 CKA Practice Task

**Task**

(Command-recall — the real exam gives you a live multi-node cluster one minor version behind target.)

1. Write the command to check what upgrade kubeadm would perform, without applying it.
2. Write the full command sequence to upgrade `kubeadm` itself on a control-plane Node to `1.31.2`.
3. Write the command to apply the control-plane upgrade to `v1.31.2`.
4. Write the drain command you'd run before upgrading a Node's kubelet, and explain why `--ignore-daemonsets` is required.
5. Write the single command used to upgrade a **worker** Node's kubeadm-managed configuration (as opposed to the control-plane-only apply command).
6. Write the command to return a Node to schedulable status after maintenance.

**Target Time**

8 minutes

## 23.14 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

Why must Kubernetes minor version upgrades happen one version at a time instead of jumping several versions at once?

**Question 2**

What's the difference between `kubeadm upgrade apply` and `kubeadm upgrade node`, and when do you use each?

**Question 3**

Why is `kubeadm upgrade plan` a safe command to run at any time, while `kubeadm upgrade apply` is not?

**Question 4**

Why is `--ignore-daemonsets` required when draining a Node, and what would happen without it?

**Question 5**

According to the kubelet version skew policy (kubelet 1.25+), how many minor versions behind the API server can a kubelet be?

**Question 6**

If `kubeadm upgrade apply` is interrupted partway through, is it safe to simply run it again? Why?

## 23.15 Useful Commands

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion

sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v<version>
sudo kubeadm upgrade node

kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl cordon <node>
kubectl uncordon <node>

sudo apt-mark unhold kubeadm kubelet kubectl
sudo apt-mark hold kubeadm kubelet kubectl
sudo apt-get install -y kubeadm=<version> kubelet=<version> kubectl=<version>

sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

## 23.16 Lab Checklist

- [ ] Understood the one-minor-version-at-a-time upgrade rule
- [ ] Upgraded `kubeadm` on the control-plane Node before anything else
- [ ] Ran `kubeadm upgrade plan` to preview the upgrade
- [ ] Ran `kubeadm upgrade apply` to upgrade control-plane components
- [ ] Drained the control-plane Node before touching its kubelet
- [ ] Upgraded kubelet/kubectl and restarted kubelet
- [ ] Uncordoned the control-plane Node
- [ ] Upgraded a worker Node using `kubeadm upgrade node` (not `apply`)
- [ ] Verified all Nodes report the new version and are `Ready`
- [ ] Understood why an interrupted `kubeadm upgrade apply` is safe to re-run
- [ ] Completed the CKA command-recall task within 8 minutes
