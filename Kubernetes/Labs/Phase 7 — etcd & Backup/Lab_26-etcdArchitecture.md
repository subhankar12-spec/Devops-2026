# Kubernetes Hands-On Lab 26 — etcd Architecture & Internals

## 26.1 Objectives

By the end of this lab, you should be able to:

- Understand what etcd is and why Kubernetes depends on it entirely
- Understand the Raft consensus algorithm at a conceptual level (leader election, quorum)
- Locate etcd's data directory, certificates, and static Pod manifest on a kubeadm cluster
- Use `etcdctl` directly to inspect cluster health and membership
- Understand how Kubernetes objects are actually stored as keys in etcd
- Understand quorum math and why etcd cluster size should be odd
- Observe what happens to the cluster when etcd loses quorum
- Troubleshoot a kube-apiserver that can't reach etcd
- Perform common etcd inspection tasks quickly for the CKA

## 26.2 Architecture

```
              kube-apiserver
        (the ONLY component that
         talks to etcd directly —
         everything else goes
         through the API server)
                    |
                    v
              etcd cluster
        (distributed key-value store,
         using RAFT consensus)

        +-----------+-----------+
        |           |           |
        v           v           v
    etcd-1       etcd-2       etcd-3
    (LEADER)    (follower)   (follower)

Raft consensus:
  - One node is elected LEADER at a time
  - ALL writes go through the leader
  - The leader replicates writes to followers
  - A write is only "committed" once a QUORUM
    (majority) of nodes have it
  - Quorum = floor(N/2) + 1
      3 nodes -> quorum 2 -> tolerates 1 node down
      5 nodes -> quorum 3 -> tolerates 2 nodes down
      4 nodes -> quorum 3 -> tolerates only 1 down
                 (WORSE than 3 nodes — never use even numbers)
```

Key concept: etcd is the **single source of truth** for the entire cluster's state — every Pod, Deployment, Secret, ConfigMap, and Node object lives there as a key. The kube-apiserver is stateless; if etcd is lost with no backup, the cluster's state is lost, even if every Node and every running container is perfectly healthy. This is exactly why etcd backup (Lab 27) is one of the highest-weighted, most operationally critical skills in the entire CKA.

## 26.3 Prerequisites

```bash
kubectl version --client
kubectl get nodes
```

This lab requires SSH/shell access to a control-plane Node (etcd internals aren't visible through `kubectl` alone).

## 26.4 Lab 1 — Locate etcd on the Control-Plane Node

```bash
sudo cat /etc/kubernetes/manifests/etcd.yaml
```

This is etcd running as a **static Pod** (same pattern as Lab 22) — the kubelet starts it directly from this manifest, bypassing the scheduler entirely, because etcd must be running before the rest of the control plane can even function.

Note the key flags inside:

- `--data-dir=/var/lib/etcd` — where the actual key-value data lives on disk
- `--cert-file` / `--key-file` — etcd's own server certificate (client-facing)
- `--peer-cert-file` / `--peer-key-file` — certificates used for etcd-to-etcd communication
- `--trusted-ca-file` — the CA used to validate clients (including kube-apiserver)
- `--initial-cluster` — the list of etcd member peer URLs

## 26.5 Confirm etcd's Data Directory

```bash
sudo ls -la /var/lib/etcd/member/
```

This is where the actual Raft log and key-value snapshots persist to disk — this directory is exactly what a proper etcd backup captures (via snapshot, not by copying these files directly — see Lab 27).

## 26.6 Lab 2 — Install etcdctl and Connect

`etcdctl` is usually already present on control-plane Nodes provisioned by kubeadm, or install it directly:

```bash
ETCD_VER=v3.5.15
curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd.tar.gz
tar xzf /tmp/etcd.tar.gz -C /tmp
sudo mv /tmp/etcd-${ETCD_VER}-linux-amd64/etcdctl /usr/local/bin/
etcdctl version
```

Set the API version (v3 is standard for all modern Kubernetes):

```bash
export ETCDCTL_API=3
```

## 26.7 Check etcd Cluster Health

```bash
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health
```

Expected: `https://127.0.0.1:2379 is healthy: successfully committed proposal: took = ...`

Check cluster membership (all members in a multi-control-plane setup):

```bash
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list -w table
```

Check which member is currently the leader:

```bash
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status -w table
```

The `IS LEADER` column shows `true` for exactly one member.

> To avoid retyping these long flags every time, export them once: `export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379; export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt; export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt; export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key` — then just run `etcdctl endpoint health`, etc.

## 26.8 Lab 3 — See Kubernetes Objects as Raw etcd Keys

Every Kubernetes object is stored under the `/registry/` prefix in etcd. List all keys for Pods:

```bash
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/pods --prefix --keys-only
```

Create a test Pod from `kubectl`, then find its exact key in etcd:

```bash
kubectl run etcd-demo-pod --image=nginx:1.27

sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/pods/default/etcd-demo-pod
```

The value is protobuf-encoded (not plain JSON) — you'll see mostly binary-looking output, but this concretely demonstrates: **every single object you've created in this entire curriculum has been a key in this store**. `kubectl get`/`create`/`delete` are really just a friendly interface over reads and writes to etcd, mediated by the API server.

## 26.9 Quorum Math and Fault Tolerance

| Cluster Size | Quorum Needed | Tolerable Failures |
|---|---|---|
| 1 | 1 | 0 (no fault tolerance at all) |
| 3 | 2 | 1 |
| 4 | 3 | 1 (same as 3, but needs one more node — never do this) |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

This is why production etcd clusters are always **odd-numbered** — an even-numbered cluster gets zero additional fault tolerance over the next-lower odd number while costing you an extra Node to run and replicate to.

## 26.10 Break It — Simulate Quorum Loss (Conceptual/Multi-Node Only)

> ⚠️ Only perform this on a disposable multi-control-plane lab cluster (3+ etcd members) — this will make the cluster briefly unusable by design.

Stop etcd on enough members to break quorum (e.g. 2 of 3):

```bash
# On etcd member 2:
sudo mv /etc/kubernetes/manifests/etcd.yaml /tmp/etcd.yaml.bak
# On etcd member 3:
sudo mv /etc/kubernetes/manifests/etcd.yaml /tmp/etcd.yaml.bak
```

Try any `kubectl` command from the control plane:

```bash
kubectl get pods
```

Expected: this hangs or times out entirely — with quorum lost, etcd cannot commit **any** read or write, and the API server has nothing to serve from, even though the underlying Pods/Nodes are still physically running fine.

## 26.11 Diagnose and Recover

```bash
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health
```

This itself may time out or report unhealthy — confirming the diagnosis. Check kube-apiserver logs for the telltale symptom:

```bash
sudo crictl logs $(sudo crictl ps --name kube-apiserver -q)
```

Look for repeated `etcdserver: request timed out` or `context deadline exceeded` errors.

Restore quorum by restarting the stopped etcd members:

```bash
# On the affected members:
sudo mv /tmp/etcd.yaml.bak /etc/kubernetes/manifests/etcd.yaml
```

Confirm recovery:

```bash
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

kubectl get pods
```

## 26.12 CKA Practice Task

**Task**

(Command-recall plus live execution if you have a real control-plane Node available.)

1. Write the command to view etcd's static Pod manifest.
2. Write the full `etcdctl endpoint health` command with all four required flags (endpoint, cacert, cert, key).
3. Write the command to list etcd cluster membership in table format.
4. Explain, using the quorum table, why a 5-node etcd cluster tolerates 2 failures but a 4-node cluster only tolerates 1.
5. Identify which single Kubernetes component is the only one allowed to talk to etcd directly.

**Target Time**

8 minutes

## 26.13 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

Why does Kubernetes lose all cluster state if etcd is lost with no backup, even if every Node is still running?

**Question 2**

What is quorum, and why is an even-numbered etcd cluster never a good idea?

**Question 3**

If etcd loses quorum, can the API server still serve `kubectl get` requests from any local cache? Why or why not?

**Question 4**

What prefix are all Kubernetes objects stored under in etcd's key space?

**Question 5**

Which component is the only one that communicates with etcd directly, and what does that imply about troubleshooting an "etcd problem" that's actually visible through `kubectl` errors?

**Question 6**

What's the difference between etcd's `--cert-file`/`--key-file` and its `--peer-cert-file`/`--peer-key-file`?

## 26.14 Useful Commands

```bash
sudo cat /etc/kubernetes/manifests/etcd.yaml
sudo ls /var/lib/etcd/member/

export ETCDCTL_API=3
etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

etcdctl ... member list -w table
etcdctl ... endpoint status -w table
etcdctl ... get /registry/pods --prefix --keys-only

sudo crictl ps --name kube-apiserver
sudo crictl logs <container-id>
```

## 26.15 Cleanup

```bash
kubectl delete pod etcd-demo-pod
unset ETCDCTL_API ETCDCTL_ENDPOINTS ETCDCTL_CACERT ETCDCTL_CERT ETCDCTL_KEY
```

## 26.16 Lab Checklist

- [ ] Located etcd's static Pod manifest and data directory
- [ ] Installed and connected `etcdctl` with the correct TLS flags
- [ ] Checked etcd cluster health and membership
- [ ] Identified the current Raft leader
- [ ] Viewed a real Kubernetes object as a raw key under `/registry/`
- [ ] Understood quorum math and why odd-numbered clusters are used
- [ ] (If multi-node lab available) Simulated quorum loss and observed cluster-wide API unavailability
- [ ] Diagnosed the failure via `etcdctl endpoint health` and apiserver logs
- [ ] Restored quorum and confirmed recovery
- [ ] Completed the CKA command-recall task within 8 minutes
