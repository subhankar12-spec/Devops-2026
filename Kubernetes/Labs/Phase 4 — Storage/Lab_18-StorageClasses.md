# Kubernetes Hands-On Lab 18 — StorageClasses & Dynamic Provisioning

## 18.1 Objectives

By the end of this lab, you should be able to:

- Understand what a StorageClass is and how it enables dynamic provisioning
- Inspect existing StorageClasses in a cluster and identify the default
- Create a PVC that dynamically provisions its own PV (no manual PV creation)
- Set and change the default StorageClass
- Understand `volumeBindingMode`: `Immediate` vs `WaitForFirstConsumer`
- Understand `allowVolumeExpansion` and expand a PVC in place
- Create a custom StorageClass with specific parameters
- Troubleshoot a PVC stuck `Pending` due to no default StorageClass or a missing provisioner
- Perform common StorageClass tasks quickly for the CKA

## 18.2 Architecture

```
                  StorageClass
        (a TEMPLATE for how to dynamically
         provision storage: which provisioner,
         what parameters, what reclaim policy)
                       |
              referenced by
                       |
                      PVC
             (storageClassName: <name>)
                       |
        provisioner automatically creates
        a matching PV AND the underlying
        cloud/storage resource (EBS volume,
        GCE PD, etc.) — no admin pre-creates
        anything manually
                       |
                       v
                 PV (auto-created)
                       |
                    Bound
```

Key concept: everything you did manually in Lab 17 (create a PV, hope it matches a PVC) is what a StorageClass **automates**. In real clusters — EKS, GKE, AKS — you almost never hand-write a PV; you request a PVC with a `storageClassName`, and the CSI driver behind that class does the rest.

## 18.3 Prerequisites

```bash
kubectl version --client
kubectl get storageclass
```

Identify the default StorageClass (marked with the `(default)` suffix in `kubectl get sc` output, or check the annotation directly):

```bash
kubectl get storageclass -o jsonpath='{range .items[*]}{.metadata.name}{": default="}{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}{"\n"}{end}'
```

> This lab assumes a cluster with a working dynamic provisioner (EKS/GKE/AKS's default class, or a local equivalent like `standard` on kind/Minikube with the `local-path-provisioner`). If no provisioner exists at all, dynamic provisioning steps will show PVCs stuck `Pending` — which is itself a useful, realistic troubleshooting scenario covered in 18.10.

## 18.4 Lab 1 — Inspect an Existing StorageClass

```bash
kubectl describe storageclass <default-class-name>
```

Note the fields:

- `Provisioner:` (e.g. `ebs.csi.aws.com`, `kubernetes.io/gce-pd`, `rancher.io/local-path`)
- `Parameters:` (backend-specific, e.g. disk type, filesystem type)
- `ReclaimPolicy:` (usually `Delete` by default for dynamic provisioning — the opposite default from manual PVs)
- `VolumeBindingMode:`
- `AllowVolumeExpansion:`

## 18.5 Lab 2 — Dynamic Provisioning via PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  storageClassName: <default-class-name>
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Apply it (substitute your actual default class name):

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  storageClassName: <default-class-name>
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
```

Watch it bind — notice no PV existed beforehand:

```bash
kubectl get pvc dynamic-pvc -w
```

Confirm a PV was auto-created:

```bash
kubectl get pv
```

You'll see a PV with an auto-generated name (e.g. `pvc-a1b2c3d4-...`) that didn't exist until the PVC triggered it.

## 18.6 Omitting storageClassName Entirely

If a cluster has a default StorageClass, you don't even need to specify one:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: default-class-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Apply and confirm it picked up the default automatically:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: default-class-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

kubectl get pvc default-class-pvc -o jsonpath='{.spec.storageClassName}'
```

## 18.7 volumeBindingMode: Immediate vs WaitForFirstConsumer

```bash
kubectl get storageclass <default-class-name> -o jsonpath='{.volumeBindingMode}'
```

| Mode | Behavior |
|---|---|
| `Immediate` | PV is provisioned as soon as the PVC is created, before any Pod uses it — risky in multi-zone clusters, since the volume might land in a zone with no available Node for the Pod |
| `WaitForFirstConsumer` | Provisioning is delayed until a Pod actually references the PVC — the scheduler can then provision storage in the **same zone** as the chosen Node (the default and recommended mode for most cloud CSI drivers) |

## 18.8 Lab 3 — Create a Custom StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-expandable
provisioner: <same-provisioner-as-default-class>
parameters:
  type: gp3
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

Apply it (substitute your cluster's actual provisioner from 18.4 — `parameters` are provisioner-specific and can be omitted/adjusted if not applicable):

```bash
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-expandable
provisioner: <same-provisioner-as-default-class>
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
EOF
```

Verify:

```bash
kubectl get storageclass fast-expandable
```

## 18.9 Volume Expansion

Create a PVC using the expandable class:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: expandable-pvc
spec:
  storageClassName: fast-expandable
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: expandable-pod
spec:
  containers:
    - name: app
      image: nginx:1.27
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: expandable-pvc
```

Apply it (remember: `WaitForFirstConsumer` means the PVC stays `Pending` until this Pod is scheduled — that's expected):

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: expandable-pvc
spec:
  storageClassName: fast-expandable
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: expandable-pod
spec:
  containers:
    - name: app
      image: nginx:1.27
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: expandable-pvc
EOF

kubectl get pvc expandable-pvc -w
```

Once `Bound`, expand it by editing the PVC's requested size upward (shrinking is not supported):

```bash
kubectl patch pvc expandable-pvc -p '{"spec":{"resources":{"requests":{"storage":"3Gi"}}}}'
kubectl get pvc expandable-pvc
```

Watch the `CAPACITY` field update — on most CSI drivers, filesystem expansion completes without needing to restart the Pod, though this depends on the driver.

## 18.10 Break It — PVC With No Provisioner

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nonexistent-provisioner-class
provisioner: fake.csi.driver.does.not.exist
volumeBindingMode: Immediate
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: broken-dynamic-pvc
spec:
  storageClassName: nonexistent-provisioner-class
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nonexistent-provisioner-class
provisioner: fake.csi.driver.does.not.exist
volumeBindingMode: Immediate
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: broken-dynamic-pvc
spec:
  storageClassName: nonexistent-provisioner-class
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
```

## 18.11 Diagnose and Recover

```bash
kubectl get pvc broken-dynamic-pvc
```

`STATUS` stuck `Pending`. Investigate:

```bash
kubectl describe pvc broken-dynamic-pvc
```

Look for an Event like:

```
waiting for a volume to be created, either by external provisioner "fake.csi.driver.does.not.exist" or manually created by system administrator
```

Confirm no such provisioner is actually running in the cluster:

```bash
kubectl get pods -A | grep -i provisioner
```

Fix by pointing the PVC at a real StorageClass:

```bash
kubectl delete pvc broken-dynamic-pvc
kubectl delete storageclass nonexistent-provisioner-class

kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fixed-dynamic-pvc
spec:
  storageClassName: <default-class-name>
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

kubectl get pvc fixed-dynamic-pvc
```

## 18.12 CKA Practice Task

**Task**

1. Identify the cluster's default StorageClass and its provisioner.
2. Create a custom StorageClass `cka-fast` using the same provisioner, `volumeBindingMode: WaitForFirstConsumer`, `allowVolumeExpansion: true`.
3. Create a PVC `cka-dyn-pvc` (1Gi, RWO) using `cka-fast`, and a Pod that mounts it.
4. Confirm the PVC only binds once the Pod is scheduled (not immediately at PVC creation).
5. Expand `cka-dyn-pvc` to 2Gi and confirm the new capacity.

**Target Time**

7 minutes

Try it without looking at previous commands.

## 18.13 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

What manual step from Lab 17 does a StorageClass eliminate?

**Question 2**

Why is `WaitForFirstConsumer` generally preferred over `Immediate` in multi-zone clusters?

**Question 3**

A PVC references a StorageClass that has no running provisioner. What status does it show, and what Event confirms the cause?

**Question 4**

Can you shrink a PVC's requested storage size after creation? Can you grow it?

**Question 5**

What's the default `reclaimPolicy` typically used by dynamically-provisioned StorageClasses, and how does that differ from the typical manual-PV convention?

**Question 6**

If a PVC doesn't specify `storageClassName` at all, what determines which class it gets?

## 18.14 Useful Commands

```bash
kubectl get storageclass
kubectl get sc
kubectl describe storageclass <name>

kubectl get pvc
kubectl describe pvc <name>
kubectl patch pvc <name> -p '{"spec":{"resources":{"requests":{"storage":"<new-size>"}}}}'

kubectl patch storageclass <name> -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

## 18.15 Cleanup

```bash
kubectl delete pod expandable-pod cka-pod
kubectl delete pvc dynamic-pvc default-class-pvc expandable-pvc \
  broken-dynamic-pvc fixed-dynamic-pvc cka-dyn-pvc
kubectl delete storageclass fast-expandable nonexistent-provisioner-class cka-fast
```

## 18.16 Lab Checklist

- [ ] Identified the cluster's default StorageClass and provisioner
- [ ] Created a PVC that dynamically provisioned its own PV
- [ ] Confirmed omitting `storageClassName` falls back to the default
- [ ] Understood `Immediate` vs `WaitForFirstConsumer` binding modes
- [ ] Created a custom StorageClass with specific parameters
- [ ] Expanded a PVC in place via `allowVolumeExpansion`
- [ ] Reproduced a PVC stuck `Pending` from a nonexistent provisioner
- [ ] Diagnosed via `describe pvc` events and fixed it
- [ ] Completed the CKA task within 7 minutes
