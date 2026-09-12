# Kubernetes Hands-On Lab 27 — etcd Backup & Restore

## 27.1 Objectives

By the end of this lab, you should be able to:

- Take a consistent etcd snapshot using `etcdctl snapshot save`
- Verify a snapshot's integrity with `etcdctl snapshot status`
- Understand that `snapshot restore` creates a NEW data directory, not an in-place restore
- Restore a snapshot and repoint etcd's static Pod manifest at the restored data directory
- Verify a full cluster recovery after restore
- Understand backup frequency/retention as an operational practice, not just a one-off command
- Perform a full simulated disaster-recovery drill: break the cluster, restore from backup, verify
- Troubleshoot a restore that leaves the API server unable to start
- Perform this entire workflow quickly and correctly under CKA time pressure — this is one of the single most commonly tested CKA tasks

## 27.2 Architecture

```
   snapshot save                    snapshot restore
        |                                  |
        v                                  v
  etcd-snapshot.db              NEW empty data directory
  (a point-in-time file,        (e.g. /var/lib/etcd-restored)
   NOT a live backup —                     |
   doesn't touch the           populated entirely from the
   running etcd process)       snapshot file's contents
                                            |
                                            v
                              Update etcd's static Pod manifest:
                              --data-dir now points at the
                              NEW restored directory
                                            |
                                            v
                              kubelet detects the manifest
                              change and restarts the etcd
                              static Pod automatically
                                            |
                                            v
                              etcd comes up serving the
                              RESTORED data — cluster state
                              is exactly as of snapshot time
```

Key concept — the single most important thing to internalize about etcd restore: **you never restore "in place."** `etcdctl snapshot restore` always writes to a brand-new directory. The actual "restore" happens when you edit the etcd static Pod manifest to point `--data-dir` at that new directory — only then does etcd actually start serving the restored data. Skipping this step (or pointing at the wrong path) is the single most common mistake under exam time pressure.

## 27.3 Prerequisites

```bash
kubectl version --client
kubectl get nodes
```

This lab requires shell access to a control-plane Node. Set up your `etcdctl` environment as in Lab 26:

```bash
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key
```

## 27.4 Lab 1 — Create a Baseline Workload (to Prove Restore Later)

```bash
kubectl create namespace before-backup
kubectl run canary-pod --image=nginx:1.27 -n before-backup
kubectl get pods -n before-backup
```

Remember this — after the restore drill, this namespace/Pod should reappear exactly as it is now, and anything created *after* the backup should disappear.

## 27.5 Lab 2 — Take an etcd Snapshot

```bash
sudo etcdctl snapshot save /opt/etcd-backup-$(date +%Y%m%d-%H%M%S).db
```

Watch the output — it reports the snapshot's hash and size on success.

List it:

```bash
ls -lh /opt/etcd-backup-*.db
```

## 27.6 Verify Snapshot Integrity

```bash
sudo etcdctl snapshot status /opt/etcd-backup-*.db -w table
```

This shows the hash, revision, total keys, and size — always verify status immediately after taking a snapshot, before you actually need it, so you're not discovering a corrupt backup during an actual incident.

## 27.7 Lab 3 — Simulate Disaster (Data Created AFTER the Backup)

```bash
kubectl create namespace after-backup
kubectl run should-disappear --image=nginx:1.27 -n after-backup
kubectl get pods -n after-backup
```

Now simulate real disaster — corrupt/delete etcd's actual data:

```bash
sudo systemctl stop kubelet
sudo mv /var/lib/etcd /var/lib/etcd-corrupted
```

Confirm the cluster is now non-functional:

```bash
kubectl get pods
```

Expected: connection refused / timeout — with etcd's data gone and the static Pod unable to start, the entire API server is down.

## 27.8 Lab 4 — Restore From Snapshot

Restore the snapshot into a **new** directory (never reuse the old path directly):

```bash
sudo etcdctl snapshot restore /opt/etcd-backup-*.db \
  --data-dir=/var/lib/etcd-restored \
  --name=<etcd-member-name> \
  --initial-cluster=<etcd-member-name>=https://127.0.0.1:2380 \
  --initial-advertise-peer-urls=https://127.0.0.1:2380
```

> Find your actual `--name`, `--initial-cluster`, and `--initial-advertise-peer-urls` values from the existing `/etc/kubernetes/manifests/etcd.yaml` (they must match exactly for a single-node control plane; multi-control-plane setups require each member's own values). For a typical kubeadm single-control-plane lab, the member name is usually the Node's hostname.

Verify the new directory was populated:

```bash
sudo ls /var/lib/etcd-restored/member/
```

## 27.9 Repoint etcd at the Restored Data

Edit the static Pod manifest:

```bash
sudo cp /etc/kubernetes/manifests/etcd.yaml /tmp/etcd.yaml.orig
sudo vi /etc/kubernetes/manifests/etcd.yaml
```

Find and update **both** of these to point at the new directory:

```yaml
    - --data-dir=/var/lib/etcd-restored
    ...
    volumeMounts:
      - mountPath: /var/lib/etcd
        name: etcd-data
...
  volumes:
    - hostPath:
        path: /var/lib/etcd-restored    # <-- change from /var/lib/etcd
        type: DirectoryOrCreate
      name: etcd-data
```

Save and exit. Restart the kubelet so it re-reads the manifest:

```bash
sudo systemctl start kubelet
```

## 27.10 Verify the Restore

Watch etcd's static Pod come back up:

```bash
sudo crictl ps --name etcd
```

Check etcd health:

```bash
etcdctl endpoint health
```

Confirm the cluster is functional again:

```bash
kubectl get nodes
kubectl get namespaces
```

## 27.11 Confirm the Restore Point Is Correct

```bash
kubectl get pods -n before-backup
```

Expected: `canary-pod` is back — this data existed at snapshot time.

```bash
kubectl get pods -n after-backup
```

Expected: `namespace "after-backup" not found` — this was created AFTER the snapshot, so the restore correctly does not contain it. This is the definitive proof the restore worked and rolled back to exactly the right point in time — not before it, not after it.

## 27.12 Break It — Wrong Data Directory Reference

Simulate the single most common real mistake: editing the manifest but forgetting to update the `volumes.hostPath.path` (leaving `--data-dir` changed but the actual mounted path unchanged):

```bash
sudo cp /etc/kubernetes/manifests/etcd.yaml /tmp/etcd-broken-attempt.yaml
```

Manually create this exact mismatch by editing only the `--data-dir` flag but NOT the corresponding `hostPath.path`:

```bash
sudo sed -i 's|--data-dir=/var/lib/etcd-restored|--data-dir=/var/lib/etcd-restored-v2|' /etc/kubernetes/manifests/etcd.yaml
```

(Note: `/var/lib/etcd-restored-v2` was never actually created by a restore — this deliberately reproduces a path mismatch.)

## 27.13 Diagnose and Recover

```bash
kubectl get nodes
```

Times out — etcd never came back up.

```bash
sudo crictl ps -a --name etcd
sudo crictl logs $(sudo crictl ps -a --name etcd -q | head -1)
```

Look for errors about the data directory not existing or being empty/unrecognized.

Fix by reverting to the known-good manifest:

```bash
sudo cp /tmp/etcd.yaml.orig /etc/kubernetes/manifests/etcd.yaml
```

Confirm recovery:

```bash
sudo crictl ps --name etcd
kubectl get nodes
```

**The core lesson**: the `--data-dir` flag and the `volumes.hostPath.path` **must always match** — they are two separate lines in the same manifest, and it's extremely easy under time pressure to update one and forget the other.

## 27.14 Automating Backups (Operational Practice)

A one-off snapshot is not a backup strategy. In real production, this runs on a schedule — typically via a CronJob (Lab 05) or an external tool (Velero, etcd-backup operators):

```bash
# Example crontab entry on the control-plane Node (not a Kubernetes CronJob,
# since etcdctl needs direct Node-level access to etcd's certs):
0 * * * * root ETCDCTL_API=3 etcdctl snapshot save /opt/etcd-backups/etcd-$(date +\%Y\%m\%d-\%H\%M).db --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key
```

Retention and off-Node/off-cluster storage (S3, another datacenter) matter just as much as the snapshot command itself — a backup that lives only on the same disk as the live data doesn't protect against Node loss.

## 27.15 CKA Practice Task

**Task**

(This mirrors the actual CKA etcd task almost exactly — practice it end-to-end under time pressure.)

1. Take an etcd snapshot to `/opt/cka-snapshot.db`.
2. Verify its status.
3. Create a namespace `cka-marker` with a test Pod.
4. "Simulate disaster" by stopping the kubelet and moving `/var/lib/etcd` aside.
5. Restore the snapshot to `/var/lib/etcd-cka-restored`.
6. Update the static Pod manifest's `--data-dir` AND `hostPath.path` to match.
7. Restart the kubelet and verify the cluster recovers with `cka-marker` intact.

**Target Time**

12 minutes (the real CKA gives roughly this much for the equivalent task)

## 27.16 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

Why does `etcdctl snapshot restore` never restore "in place" into the existing data directory?

**Question 2**

Name the two separate places in the etcd static Pod manifest that must both be updated to point at a new data directory, and explain what happens if you only update one.

**Question 3**

Why do you restart the kubelet (or wait for it) after editing the etcd manifest, rather than restarting the etcd Pod with `kubectl`?

**Question 4**

If you restore a snapshot taken at 2:00 PM, and data was created at 2:15 PM, what happens to that data after the restore?

**Question 5**

Why is `etcdctl snapshot status` worth running immediately after every backup, rather than only when you need to restore?

**Question 6**

Why is a Kubernetes CronJob generally not used to run etcd backups directly, and what runs them instead?

## 27.17 Useful Commands

```bash
export ETCDCTL_API=3

etcdctl snapshot save <path>.db \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

etcdctl snapshot status <path>.db -w table

etcdctl snapshot restore <path>.db \
  --data-dir=<new-directory> \
  --name=<member-name> \
  --initial-cluster=<member-name>=https://127.0.0.1:2380 \
  --initial-advertise-peer-urls=https://127.0.0.1:2380

sudo vi /etc/kubernetes/manifests/etcd.yaml
sudo systemctl restart kubelet

sudo crictl ps --name etcd
sudo crictl logs <container-id>
```

## 27.18 Cleanup

```bash
kubectl delete namespace before-backup cka-marker
sudo rm -rf /opt/etcd-backup-*.db /opt/cka-snapshot.db
sudo rm -rf /var/lib/etcd-corrupted /var/lib/etcd-restored /var/lib/etcd-cka-restored
sudo rm -f /tmp/etcd.yaml.orig /tmp/etcd-broken-attempt.yaml
```

## 27.19 Lab Checklist

- [ ] Took an etcd snapshot with `etcdctl snapshot save`
- [ ] Verified the snapshot with `etcdctl snapshot status`
- [ ] Created data before the backup and more data after it
- [ ] Simulated total etcd data loss
- [ ] Restored the snapshot into a NEW data directory
- [ ] Updated both `--data-dir` and `hostPath.path` in the manifest to match
- [ ] Restarted the kubelet and confirmed etcd came back up
- [ ] Verified pre-backup data was restored and post-backup data was correctly absent
- [ ] Reproduced a data-dir/hostPath mismatch and diagnosed it via `crictl logs`
- [ ] Understood backup automation and off-Node retention as real operational practice
- [ ] Completed the full CKA-style drill within 12 minutes
