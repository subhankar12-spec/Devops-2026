# Kubernetes Hands-On Lab 16 — Volumes (emptyDir, hostPath, ConfigMap/Secret)

## 16.1 Objectives

By the end of this lab, you should be able to:

- Understand that a Volume's lifecycle is tied to the **Pod**, not the container
- Use `emptyDir` to share data between containers in a Pod
- Use `emptyDir` with `medium: Memory` (tmpfs, RAM-backed)
- Use `hostPath` to mount a path from the Node's filesystem
- Understand the security and portability risks of `hostPath`
- Recap ConfigMap/Secret volumes in the context of the broader Volumes API (built in Lab 06)
- Understand what survives and what doesn't across container restarts vs Pod deletion
- Troubleshoot a Pod stuck `Pending`/`ContainerCreating` due to a bad `hostPath`
- Perform common Volume tasks quickly for the CKA

## 16.2 Architecture

```
                        Pod
                         |
              (Volumes are defined at
               the Pod level, mounted
               into 0+ containers)
                         |
        +----------------+----------------+----------------+
        |                |                |                |
        v                v                v                v
    emptyDir         emptyDir          hostPath      configMap/secret
   (disk-backed,    (medium: Memory,  (path on the    (recap: see
    survives         tmpfs, RAM-      NODE's own      Lab 06)
    container        backed, lost     filesystem —
    restarts,        on Pod deletion  NOT the
    deleted with     even faster)     container's)
    the Pod)
```

Key concept: **container restart** (crash, liveness probe failure) does NOT lose `emptyDir` data — the kubelet keeps the same Pod sandbox. **Pod deletion** (rescheduling to a new Node, scale-down, manual delete) DOES lose `emptyDir` data — it's tied to that specific Pod instance on that specific Node, not to the container inside it.

## 16.3 Prerequisites

```bash
kubectl version --client
kubectl get nodes
```

## 16.4 Lab 1 — emptyDir Shared Between Containers

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-pod
spec:
  volumes:
    - name: shared-data
      emptyDir: {}
  containers:
    - name: writer
      image: busybox:1.36
      command: ["sh", "-c", "while true; do date >> /data/log.txt; sleep 5; done"]
      volumeMounts:
        - name: shared-data
          mountPath: /data
    - name: reader
      image: busybox:1.36
      command: ["sh", "-c", "tail -f /data/log.txt"]
      volumeMounts:
        - name: shared-data
          mountPath: /data
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-pod
spec:
  volumes:
    - name: shared-data
      emptyDir: {}
  containers:
    - name: writer
      image: busybox:1.36
      command: ["sh", "-c", "while true; do date >> /data/log.txt; sleep 5; done"]
      volumeMounts:
        - name: shared-data
          mountPath: /data
    - name: reader
      image: busybox:1.36
      command: ["sh", "-c", "tail -f /data/log.txt"]
      volumeMounts:
        - name: shared-data
          mountPath: /data
EOF
```

Confirm the reader sees what the writer wrote:

```bash
kubectl logs emptydir-pod -c reader
```

## 16.5 emptyDir Survives Container Restart, Not Pod Deletion

Crash the writer container to force a restart:

```bash
kubectl exec emptydir-pod -c writer -- sh -c "kill 1"
```

Wait for the restart, then confirm the file still has old entries plus new ones (data survived):

```bash
kubectl get pod emptydir-pod
kubectl exec emptydir-pod -c reader -- cat /data/log.txt
```

Now delete the whole Pod and recreate it — this time the volume is a brand-new, empty one:

```bash
kubectl delete pod emptydir-pod
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-pod
spec:
  volumes:
    - name: shared-data
      emptyDir: {}
  containers:
    - name: writer
      image: busybox:1.36
      command: ["sh", "-c", "while true; do date >> /data/log.txt; sleep 5; done"]
      volumeMounts:
        - name: shared-data
          mountPath: /data
    - name: reader
      image: busybox:1.36
      command: ["sh", "-c", "tail -f /data/log.txt"]
      volumeMounts:
        - name: shared-data
          mountPath: /data
EOF

sleep 8
kubectl exec emptydir-pod -c reader -- cat /data/log.txt
```

Only the new entries are there — this confirms `emptyDir`'s lifecycle is bound to the Pod instance, not persisted anywhere durable.

## 16.6 Lab 2 — emptyDir Backed by Memory (tmpfs)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-emptydir-pod
spec:
  volumes:
    - name: cache-vol
      emptyDir:
        medium: Memory
        sizeLimit: 100Mi
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: cache-vol
          mountPath: /cache
```

Apply it and confirm it's mounted as tmpfs:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: memory-emptydir-pod
spec:
  volumes:
    - name: cache-vol
      emptyDir:
        medium: Memory
        sizeLimit: 100Mi
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: cache-vol
          mountPath: /cache
EOF

kubectl exec memory-emptydir-pod -- mount | grep /cache
```

You should see `tmpfs` in the output. Use cases: fast scratch space, avoiding disk I/O for temporary/sensitive data (RAM-backed data disappears completely on Pod deletion, never touches disk) — but it counts against the container's memory limit.

## 16.7 Lab 3 — hostPath

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  volumes:
    - name: host-vol
      hostPath:
        path: /tmp/k8s-hostpath-demo
        type: DirectoryOrCreate
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo Hello from Pod >> /data/from-pod.txt; sleep 3600"]
      volumeMounts:
        - name: host-vol
          mountPath: /data
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  volumes:
    - name: host-vol
      hostPath:
        path: /tmp/k8s-hostpath-demo
        type: DirectoryOrCreate
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo Hello from Pod >> /data/from-pod.txt; sleep 3600"]
      volumeMounts:
        - name: host-vol
          mountPath: /data
EOF
```

Check which Node it landed on, then verify the file actually exists on that Node's filesystem (not just inside the container):

```bash
kubectl get pod hostpath-pod -o wide
```

If you have Node shell access (e.g. `ssh` or `kubectl debug node/<name>`), you'd find `/tmp/k8s-hostpath-demo/from-pod.txt` sitting directly on that Node.

## 16.8 hostPath Risks

| Risk | Why it matters |
|---|---|
| Node-specific data | If the Pod reschedules to a different Node, the data is NOT there — `hostPath` is tied to one physical/virtual machine, not the cluster |
| Security | A Pod with `hostPath` access to sensitive paths (e.g. `/etc`, `/var/run/docker.sock`) can potentially compromise the Node itself |
| Not portable | Breaks the "Pods are interchangeable" model — makes workloads effectively pinned to specific Nodes without using proper affinity/taints |

`hostPath` `type` options worth knowing: `DirectoryOrCreate`, `Directory`, `FileOrCreate`, `File`, `Socket`, `CharDevice`, `BlockDevice` — each validates or creates the target differently and will error at Pod creation if the existing path doesn't match the expected type.

## 16.9 Lab 4 — Recap: ConfigMap/Secret Volumes

(Built in full in Lab 06 — quick recap here since they're part of the same Volumes API family.)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: recap-config-vol-pod
spec:
  volumes:
    - name: cm-vol
      configMap:
        name: app-config
        optional: true
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: cm-vol
          mountPath: /etc/config
```

Note the `optional: true` field here — without it, if the referenced ConfigMap doesn't exist, the Pod stays stuck (see 16.11). With `optional: true`, the mount is simply empty if the ConfigMap is missing, instead of blocking Pod startup.

## 16.10 Break It — Bad hostPath Type

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: broken-hostpath-pod
spec:
  volumes:
    - name: bad-vol
      hostPath:
        path: /tmp/k8s-hostpath-demo/from-pod.txt
        type: Directory
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: bad-vol
          mountPath: /data
```

Apply it (note: this path is actually a **file** from 16.7, but we're claiming `type: Directory`):

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: broken-hostpath-pod
spec:
  volumes:
    - name: bad-vol
      hostPath:
        path: /tmp/k8s-hostpath-demo/from-pod.txt
        type: Directory
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: bad-vol
          mountPath: /data
EOF
```

## 16.11 Diagnose and Recover

```bash
kubectl get pod broken-hostpath-pod
```

Status stuck at `ContainerCreating`. Investigate:

```bash
kubectl describe pod broken-hostpath-pod
```

Look for an Event like:

```
Warning  FailedMount  ... hostPath type check failed: /tmp/k8s-hostpath-demo/from-pod.txt is not a directory
```

Fix by correcting the type to match reality:

```bash
kubectl delete pod broken-hostpath-pod

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: fixed-hostpath-pod
spec:
  volumes:
    - name: good-vol
      hostPath:
        path: /tmp/k8s-hostpath-demo/from-pod.txt
        type: File
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: good-vol
          mountPath: /data/from-pod.txt
EOF

kubectl get pod fixed-hostpath-pod
```

## 16.12 CKA Practice Task

**Task**

1. Create a Pod `cka-vol-pod` with two containers sharing an `emptyDir` volume at `/shared` — one container writes a timestamp every 5 seconds, the other tails the file.
2. Confirm the shared file updates are visible in the second container's logs.
3. Add a second volume to the same Pod: `emptyDir` with `medium: Memory`, mounted at `/cache`, and confirm via `mount` that it's tmpfs.
4. Create a separate Pod `cka-hostpath-pod` using `hostPath` with `type: DirectoryOrCreate` at `/tmp/cka-demo`, and write a file into it from inside the container.

**Target Time**

7 minutes

Try it without looking at previous commands.

## 16.13 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

Does `emptyDir` data survive a container crash/restart? Does it survive Pod deletion? Why the difference?

**Question 2**

What's the tradeoff of using `emptyDir` with `medium: Memory` instead of the default disk-backed `emptyDir`?

**Question 3**

Why is `hostPath` considered risky in multi-tenant or security-sensitive clusters?

**Question 4**

If a Pod using `hostPath` gets rescheduled to a different Node, what happens to the data it wrote?

**Question 5**

A Pod is stuck in `ContainerCreating` with a `FailedMount` event mentioning a `hostPath type check failed`. What's the likely cause?

**Question 6**

What does setting `optional: true` on a ConfigMap volume source change about Pod startup behavior?

## 16.14 Useful Commands

```bash
kubectl exec <pod> -c <container> -- <command>
kubectl exec <pod> -- mount | grep <mount-path>

kubectl describe pod <name>
kubectl get pod <name> -o wide

kubectl delete pod <name>
```

## 16.15 Cleanup

```bash
kubectl delete pod emptydir-pod memory-emptydir-pod hostpath-pod \
  recap-config-vol-pod broken-hostpath-pod fixed-hostpath-pod \
  cka-vol-pod cka-hostpath-pod
```

## 16.16 Lab Checklist

- [ ] Shared data between two containers via `emptyDir`
- [ ] Confirmed `emptyDir` survives container restart
- [ ] Confirmed `emptyDir` does NOT survive Pod deletion
- [ ] Used `emptyDir` with `medium: Memory` and confirmed tmpfs
- [ ] Mounted a `hostPath` volume and understood its Node-specific risk
- [ ] Recapped ConfigMap volume mounting and the `optional` field
- [ ] Reproduced a `FailedMount` from a `hostPath` type mismatch
- [ ] Diagnosed and fixed the type mismatch
- [ ] Completed the CKA task within 7 minutes
