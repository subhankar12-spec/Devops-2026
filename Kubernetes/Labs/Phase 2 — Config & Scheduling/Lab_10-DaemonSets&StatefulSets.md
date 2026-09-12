# Kubernetes Hands-On Lab 10 — DaemonSets & StatefulSets

## 10.1 Objectives

By the end of this lab, you should be able to:

- Create a DaemonSet and understand "exactly one Pod per Node"
- Use tolerations on a DaemonSet to run on control-plane/tainted Nodes
- Perform a rolling update of a DaemonSet
- Create a StatefulSet and understand stable, ordered Pod identity
- Understand StatefulSet Pod naming (`<name>-0`, `<name>-1`, ...) and stable network identity via a Headless Service
- Use `volumeClaimTemplates` for per-replica persistent storage
- Observe ordered (not parallel) creation, scaling, and deletion of StatefulSet Pods
- Troubleshoot a StatefulSet stuck because a Pod won't terminate cleanly
- Perform common DaemonSet/StatefulSet tasks quickly for the CKA

## 10.2 Architecture

```
DaemonSet                              StatefulSet (replicas: 3)
    |                                       |
one Pod per Node,                  Headless Service (clusterIP: None)
automatically added/removed               |
as Nodes join/leave                +-------+-------+-------+
    |                              |       |       |
+---+---+---+                      v       v       v
v       v   v                  web-0   web-1   web-2
Node-1 Node-2 Node-3          (stable  (stable  (stable
(1 Pod) (1 Pod)(1 Pod)         name +   name +   name +
                                own PVC) own PVC) own PVC)

Creation order: web-0 -> web-1 -> web-2 (sequential, each must be Ready first)
Deletion order: web-2 -> web-1 -> web-0 (reverse sequential)
```

Key concept: a Deployment's Pods are interchangeable clones — any replica can be replaced by any other. A StatefulSet's Pods have a **persistent identity** (name, network address, and storage) that survives rescheduling — Pod `web-1` is always `web-1`, even after being deleted and recreated.

## 10.3 Prerequisites

```bash
kubectl version --client
kubectl get nodes
```

## 10.4 Lab 1 — Create a DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-monitor
  labels:
    app: node-monitor
spec:
  selector:
    matchLabels:
      app: node-monitor
  template:
    metadata:
      labels:
        app: node-monitor
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      containers:
        - name: monitor
          image: busybox:1.36
          command: ["sh", "-c", "while true; do echo Monitoring $(hostname); sleep 30; done"]
          resources:
            requests:
              cpu: "50m"
              memory: "32Mi"
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-monitor
  labels:
    app: node-monitor
spec:
  selector:
    matchLabels:
      app: node-monitor
  template:
    metadata:
      labels:
        app: node-monitor
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      containers:
        - name: monitor
          image: busybox:1.36
          command: ["sh", "-c", "while true; do echo Monitoring \$(hostname); sleep 30; done"]
          resources:
            requests:
              cpu: "50m"
              memory: "32Mi"
EOF
```

## 10.5 Verify One Pod Per Node

```bash
kubectl get daemonset node-monitor
kubectl get pods -l app=node-monitor -o wide
```

`DESIRED`, `CURRENT`, and `READY` should all equal your Node count. Note the toleration for `node-role.kubernetes.io/control-plane` — without it, the DaemonSet would skip a tainted control-plane Node (this is exactly how `kube-proxy` and CNI plugins run on every Node, including control-plane, in real clusters).

## 10.6 DaemonSet Rolling Update

Check the current update strategy:

```bash
kubectl get daemonset node-monitor -o jsonpath='{.spec.updateStrategy}'
```

Default is `RollingUpdate`. Update the image/command:

```bash
kubectl set image daemonset/node-monitor monitor=busybox:1.36
kubectl rollout status daemonset/node-monitor
kubectl rollout history daemonset/node-monitor
```

DaemonSets support `rollout status`/`history`/`undo` just like Deployments — but there is no scaling concept (`kubectl scale` doesn't apply; the Node count controls replica count).

## 10.7 Lab 2 — Create a Headless Service

StatefulSets need a Headless Service (`clusterIP: None`) to provide each Pod a stable DNS name:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-headless
spec:
  clusterIP: None
  selector:
    app: web-sts
  ports:
    - port: 80
      name: web
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: web-headless
spec:
  clusterIP: None
  selector:
    app: web-sts
  ports:
    - port: 80
      name: web
EOF
```

## 10.8 Create a StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web-headless
  replicas: 3
  selector:
    matchLabels:
      app: web-sts
  template:
    metadata:
      labels:
        app: web-sts
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
              name: web
          volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: www
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web-headless
  replicas: 3
  selector:
    matchLabels:
      app: web-sts
  template:
    metadata:
      labels:
        app: web-sts
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
              name: web
          volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: www
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
EOF
```

## 10.9 Observe Ordered Creation

Watch Pods come up **one at a time**, in order, each becoming Ready before the next starts:

```bash
kubectl get pods -l app=web-sts -w
```

You should see `web-0` reach `Running`/`Ready` before `web-1` is even created, then `web-1` before `web-2`. Compare this to a Deployment, where all replicas are created in parallel.

Verify PVCs were created per-replica:

```bash
kubectl get pvc
```

You'll see `www-web-0`, `www-web-1`, `www-web-2` — each Pod gets its own dedicated volume, unlike a Deployment where replicas would share or conflict over a single PVC.

## 10.10 Stable Network Identity

Each Pod gets a predictable DNS name via the headless Service:

```
<pod-name>.<service-name>.<namespace>.svc.cluster.local
```

Test DNS resolution from inside the cluster:

```bash
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- \
  nslookup web-0.web-headless.default.svc.cluster.local
```

## 10.11 Ordered Scaling and Deletion

Scale up — new Pods are added in order (`web-3` after `web-2` is Ready):

```bash
kubectl scale statefulset web --replicas=5
kubectl get pods -l app=web-sts -w
```

Scale down — Pods are removed in **reverse** order (`web-4` first, then `web-3`, etc.):

```bash
kubectl scale statefulset web --replicas=2
kubectl get pods -l app=web-sts -w
```

Note: scaling down does **not** delete the PVCs — `www-web-3` and `www-web-4` still exist, so if you scale back up, the same Pods reattach to their original storage:

```bash
kubectl get pvc
```

## 10.12 Persistent Identity After Pod Deletion

Delete `web-0` directly:

```bash
kubectl delete pod web-0
```

Watch it come back with the **same name** and reattach to the **same PVC**:

```bash
kubectl get pods -l app=web-sts -w
kubectl get pvc
```

This is the core StatefulSet guarantee: identity (name + storage + network) survives Pod rescheduling, which a Deployment/ReplicaSet does not provide (a replaced Pod there gets a brand-new random name and a fresh, unrelated volume unless you engineer that separately).

## 10.13 Break It — Stuck Terminating

Delete the whole StatefulSet but leave a stuck scenario by force-deleting incorrectly is uncommon; instead simulate a Pod that won't terminate cleanly with a `preStop` hook that hangs:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: stuck-sts
spec:
  serviceName: stuck-headless
  replicas: 1
  selector:
    matchLabels:
      app: stuck-sts
  template:
    metadata:
      labels:
        app: stuck-sts
    spec:
      terminationGracePeriodSeconds: 300
      containers:
        - name: app
          image: nginx:1.27
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 290"]
```

Apply it, then try to scale it to 0:

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: stuck-sts
spec:
  serviceName: stuck-headless
  replicas: 1
  selector:
    matchLabels:
      app: stuck-sts
  template:
    metadata:
      labels:
        app: stuck-sts
    spec:
      terminationGracePeriodSeconds: 300
      containers:
        - name: app
          image: nginx:1.27
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 290"]
EOF

kubectl scale statefulset stuck-sts --replicas=0
kubectl get pods -l app=stuck-sts -w
```

## 10.14 Diagnose and Recover

The Pod sits in `Terminating` for a long time:

```bash
kubectl get pod -l app=stuck-sts
```

Check what's holding it up:

```bash
kubectl describe pod -l app=stuck-sts
```

You'll see `terminationGracePeriodSeconds: 300` — Kubernetes is honoring the full grace period while the `preStop` hook runs.

Force-delete only when you understand the consequences (data loss risk, potential split-brain if the process is actually still running elsewhere):

```bash
kubectl delete pod -l app=stuck-sts --grace-period=0 --force
```

Confirm cleanup:

```bash
kubectl get pods -l app=stuck-sts
kubectl delete statefulset stuck-sts
```

## 10.15 CKA Practice Task

**Task**

1. Create a DaemonSet named `log-agent` (image `busybox:1.36`, command `sleep 3600`) and verify it has exactly one Pod per Node.
2. Create a Headless Service `db-headless` and a StatefulSet `db` with 3 replicas, image `nginx:1.28`, using `volumeClaimTemplates` for a 1Gi volume named `data`.
3. Confirm Pods are named `db-0`, `db-1`, `db-2` and came up in order.
4. Scale `db` down to 1 replica and confirm PVCs for `db-1` and `db-2` still exist.
5. Delete `db-0` and confirm it comes back with the same name and reattaches to the same PVC.

**Target Time**

8 minutes

Try it without looking at previous commands.

## 10.16 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

Why does a DaemonSet not have a `replicas` field, and what determines how many Pods it runs?

**Question 2**

Why do DaemonSets typically need tolerations that regular Deployments don't?

**Question 3**

What is the purpose of a Headless Service in a StatefulSet, specifically?

**Question 4**

If you scale a StatefulSet from 3 to 1, what happens to the PVCs for the removed replicas?

**Question 5**

What guarantee does a StatefulSet provide that a Deployment does not, when a Pod is deleted and recreated?

**Question 6**

A Pod stays `Terminating` for several minutes. What two things would you check, and what's the risk of force-deleting it?

## 10.17 Useful Commands

```bash
kubectl get daemonset
kubectl get ds
kubectl describe ds <name>
kubectl set image daemonset/<name> <container>=<image>
kubectl rollout status daemonset/<name>
kubectl rollout history daemonset/<name>

kubectl get statefulset
kubectl get sts
kubectl describe sts <name>
kubectl scale statefulset <name> --replicas=<n>

kubectl get pvc
kubectl describe pvc <name>

kubectl delete pod <name> --grace-period=0 --force
```

## 10.18 Cleanup

```bash
kubectl delete daemonset node-monitor log-agent
kubectl delete statefulset web db stuck-sts
kubectl delete service web-headless db-headless
kubectl delete pvc -l app=web-sts
kubectl delete pvc -l app=db
```

## 10.19 Lab Checklist

- [ ] Created a DaemonSet and confirmed one Pod per Node
- [ ] Used a toleration to run a DaemonSet Pod on a control-plane Node
- [ ] Performed a DaemonSet rolling update
- [ ] Created a Headless Service
- [ ] Created a StatefulSet with `volumeClaimTemplates`
- [ ] Observed sequential (not parallel) Pod creation
- [ ] Verified per-replica PVCs
- [ ] Tested stable DNS naming via the headless Service
- [ ] Observed ordered scale-up and reverse-order scale-down
- [ ] Confirmed identity/storage persistence after deleting a StatefulSet Pod
- [ ] Reproduced and resolved a stuck-`Terminating` Pod
- [ ] Completed the CKA task within 8 minutes
