# Kubernetes Hands-On Lab 17 — PersistentVolumes & PersistentVolumeClaims

## 17.1 Objectives

By the end of this lab, you should be able to:

- Understand the PV/PVC abstraction and why it decouples storage from Pods
- Create a static PersistentVolume (PV) manually
- Create a PersistentVolumeClaim (PVC) that binds to a matching PV
- Understand access modes: `ReadWriteOnce`, `ReadOnlyMany`, `ReadWriteMany`, `ReadWriteOncePod`
- Understand the PV lifecycle: `Available` → `Bound` → `Released` → (`Retain`/`Recycle`/`Delete`)
- Mount a PVC into a Pod
- Understand what happens when a PVC-backed Pod is deleted vs when the PVC itself is deleted
- Troubleshoot a PVC stuck `Pending` due to no matching PV
- Perform common PV/PVC tasks quickly for the CKA

## 17.2 Architecture

```
      Cluster-scoped                        Namespaced
      PersistentVolume (PV)                 PersistentVolumeClaim (PVC)
     (the actual storage,                  (a NAMESPACE's REQUEST
      provisioned by an admin              for storage matching
      or a StorageClass —                  certain criteria:
      see Lab 18)                          size, accessMode,
            |                              storageClassName)
            |          binds to                    |
            +-----------------------------------------+
                              |
                              v
                        Pod's volumeMount
                    (references the PVC by
                     name, never the PV directly)

Binding is 1:1 — once a PVC binds to a PV, that PV is
exclusively reserved for that PVC, even if the PV's
capacity is larger than requested.
```

Key concept: a Pod never references a PV directly — it references a PVC. This indirection is the whole point: developers request abstract storage ("I need 5Gi, ReadWriteOnce"), and administrators (or dynamic provisioning) worry about which actual disk/NFS-share/cloud volume satisfies that request.

## 17.3 Prerequisites

```bash
kubectl version --client
kubectl get nodes
kubectl get storageclass
```

> This lab uses `hostPath`-backed PVs for simplicity and portability across any cluster (including local ones like kind/Minikube). In real production clusters, PVs are almost always dynamically provisioned via a StorageClass (Lab 18) rather than created manually like this — but understanding the static/manual flow first is essential for the CKA and for understanding what dynamic provisioning is actually automating.

## 17.4 Lab 1 — Create a Static PersistentVolume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-manual-1
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/pv-data
    type: DirectoryOrCreate
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-manual-1
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/pv-data
    type: DirectoryOrCreate
EOF
```

## 17.5 Verify the PV

```bash
kubectl get pv
kubectl describe pv pv-manual-1
```

`STATUS` should show `Available` — it exists but nothing has claimed it yet. PVs are **cluster-scoped** (no namespace).

## 17.6 Lab 2 — Create a Matching PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-manual-1
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-manual-1
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
```

## 17.7 Verify the Binding

```bash
kubectl get pvc pvc-manual-1
kubectl get pv pv-manual-1
```

Both should now show `STATUS: Bound`, and `pv-manual-1`'s `CLAIM` column should read `default/pvc-manual-1`. The PVC bound because its `accessModes` and requested `storage` were compatible with the PV (note: a PVC can bind to a PV with *more* capacity than requested, but never less).

## 17.8 Access Modes Explained

| Access Mode | Meaning |
|---|---|
| `ReadWriteOnce` (RWO) | Mounted read-write by a single Node at a time (not single Pod — multiple Pods on the same Node can share it) |
| `ReadOnlyMany` (ROX) | Mounted read-only by many Nodes simultaneously |
| `ReadWriteMany` (RWX) | Mounted read-write by many Nodes simultaneously (needs a storage backend that supports it, e.g. NFS, EFS — not typical block storage like EBS) |
| `ReadWriteOncePod` (RWOP) | Mounted read-write by a single **Pod** only (stricter than RWO — newer addition for workloads that truly need exclusive single-Pod access) |

Most cloud block storage (AWS EBS, GCP PD, Azure Disk) only supports `ReadWriteOnce`. Getting `ReadWriteMany` typically requires file-based storage (NFS, EFS, Azure Files) or a specialized CSI driver.

## 17.9 Lab 3 — Mount the PVC in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
    - name: app
      image: nginx:1.27
      volumeMounts:
        - name: storage
          mountPath: /usr/share/nginx/html
  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: pvc-manual-1
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
    - name: app
      image: nginx:1.27
      volumeMounts:
        - name: storage
          mountPath: /usr/share/nginx/html
  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: pvc-manual-1
EOF
```

Write some data and confirm it lands on the underlying `hostPath`:

```bash
kubectl exec pvc-pod -- sh -c "echo 'Hello from PVC' > /usr/share/nginx/html/index.html"
kubectl exec pvc-pod -- cat /usr/share/nginx/html/index.html
```

## 17.10 Data Survives Pod Deletion (Unlike emptyDir)

Delete just the Pod:

```bash
kubectl delete pod pvc-pod
```

Recreate it (same YAML) and confirm the data is still there:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
    - name: app
      image: nginx:1.27
      volumeMounts:
        - name: storage
          mountPath: /usr/share/nginx/html
  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: pvc-manual-1
EOF

kubectl exec pvc-pod -- cat /usr/share/nginx/html/index.html
```

`Hello from PVC` is still there — this is the entire point of PV/PVC versus `emptyDir`: the storage outlives the Pod.

## 17.11 The Reclaim Policy — What Happens When the PVC Is Deleted

```bash
kubectl delete pod pvc-pod
kubectl delete pvc pvc-manual-1
kubectl get pv pv-manual-1
```

With `persistentVolumeReclaimPolicy: Retain` (set in 17.4), the PV's `STATUS` becomes `Released`, **not** `Available` — the underlying data is preserved, but the PV can't be automatically reused by a new PVC until an administrator manually intervenes (clean the data and reset the PV, or delete and recreate it).

| Reclaim Policy | Behavior when PVC is deleted |
|---|---|
| `Retain` | PV becomes `Released`; data preserved; requires manual cleanup before reuse |
| `Delete` | PV and its underlying storage are automatically deleted (default for most dynamically-provisioned PVs) |
| `Recycle` | Deprecated — basic scrub (`rm -rf`) then made `Available` again; don't use in new clusters |

Clean up the released PV so it can be reused:

```bash
kubectl delete pv pv-manual-1
```

## 17.12 Break It — PVC Stuck Pending (No Matching PV)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-too-big
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
```

Apply it (with no matching PV — either wrong access mode, wrong size, or none available at all):

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-too-big
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
EOF
```

## 17.13 Diagnose and Recover

```bash
kubectl get pvc pvc-too-big
```

`STATUS` stays `Pending` indefinitely. Investigate:

```bash
kubectl describe pvc pvc-too-big
```

Look for an Event like:

```
no persistent volumes available for this claim and no storage class is set
```

Check what PVs actually exist and their specs:

```bash
kubectl get pv
```

Fix by either creating a matching PV, or (more realistically in production) requesting something an available `StorageClass` can dynamically provision (Lab 18):

```bash
kubectl delete pvc pvc-too-big

kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-large
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/pv-large-data
    type: DirectoryOrCreate
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-fixed
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
EOF

kubectl get pvc pvc-fixed
```

> Note: `hostPath` doesn't actually support true multi-Node `ReadWriteMany` semantics in a real cluster — this fix is illustrative for the binding mechanics, not a production-valid RWX solution (real RWX needs NFS/EFS/etc.).

## 17.14 CKA Practice Task

**Task**

1. Create a PV named `cka-pv` with 2Gi capacity, `ReadWriteOnce`, `hostPath` at `/tmp/cka-pv-data`, reclaim policy `Retain`.
2. Create a PVC named `cka-pvc` requesting 1Gi, `ReadWriteOnce`, and confirm it binds to `cka-pv`.
3. Mount `cka-pvc` into a Pod named `cka-pv-pod` and write a test file into it.
4. Delete the Pod, recreate it, and confirm the file still exists.
5. Delete the PVC and confirm the PV transitions to `Released` (not `Available`).

**Target Time**

7 minutes

Try it without looking at previous commands.

## 17.15 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

Why does a Pod reference a PVC instead of referencing a PV directly?

**Question 2**

What is the difference between `ReadWriteOnce` and `ReadWriteOncePod`?

**Question 3**

A PVC is stuck `Pending`. What are the two most common causes?

**Question 4**

If a PV's reclaim policy is `Retain` and its bound PVC is deleted, what status does the PV move to, and what must happen before it can be reused?

**Question 5**

Does deleting a Pod that mounts a PVC delete the underlying data? What about deleting the PVC itself?

**Question 6**

Can a single PV be bound to two different PVCs simultaneously to share storage across namespaces?

## 17.16 Useful Commands

```bash
kubectl get pv
kubectl get pvc
kubectl describe pv <name>
kubectl describe pvc <name>

kubectl get pv,pvc

kubectl delete pvc <name>
kubectl delete pv <name>
```

## 17.17 Cleanup

```bash
kubectl delete pod pvc-pod cka-pv-pod
kubectl delete pvc pvc-manual-1 pvc-too-big pvc-fixed cka-pvc
kubectl delete pv pv-manual-1 pv-large cka-pv
```

## 17.18 Lab Checklist

- [ ] Created a static PersistentVolume
- [ ] Created a PersistentVolumeClaim that bound to it
- [ ] Understood the four access modes and their real-world storage backend implications
- [ ] Mounted a PVC into a Pod
- [ ] Confirmed data survives Pod deletion/recreation
- [ ] Observed a PV move to `Released` after its PVC was deleted (with `Retain` policy)
- [ ] Understood `Retain` vs `Delete` vs (deprecated) `Recycle`
- [ ] Reproduced a PVC stuck `Pending` from no matching PV
- [ ] Diagnosed and fixed it by creating a matching PV
- [ ] Completed the CKA task within 7 minutes
