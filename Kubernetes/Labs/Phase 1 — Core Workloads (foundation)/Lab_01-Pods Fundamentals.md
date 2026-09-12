# Kubernetes Hands-On Lab 01 — Pods Fundamentals

## 1.1 Objectives

By the end of this lab, you should be able to:

- Understand the Pod lifecycle and its phases
- Create a single-container Pod using YAML
- Create and use multi-container Pods (sidecar pattern)
- Understand and use Init Containers
- Configure Liveness, Readiness, and Startup probes
- Read Pod status and container states
- Troubleshoot a Pod stuck in `CrashLoopBackOff` due to a failing probe
- Perform common Pod tasks quickly for the CKA

## 1.2 Architecture

```
                     Pod
                      |
        +-------------+-------------+
        |             |             |
        v             v             v
   initContainers   containers   volumes
  (run in order,    (run in       (shared
   must succeed      parallel,     storage
   before app        share pod     between
   containers        network &     containers)
   start)            can share
                      volumes)
```

Pod phases: `Pending` → `Running` → `Succeeded` / `Failed`
(a Pod can also show `Unknown` if the node is unreachable)

Container states inside a Pod: `Waiting` → `Running` → `Terminated`

## 1.3 Prerequisites

```bash
kubectl version --client
kubectl get nodes
```

Verify the cluster is available:

```bash
kubectl get nodes
```

## 1.4 Lab 1 — Create a Basic Pod

Create the following file: [`pod-basic.yaml`](./pod-basic.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: basic-pod
  labels:
    app: basic
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
```

Apply it:

```bash
kubectl apply -f pod-basic.yaml
```

## 1.5 Observe the Pod Lifecycle

Watch the Pod move through phases:

```bash
kubectl get pod basic-pod -w
```

Check the phase directly:

```bash
kubectl get pod basic-pod -o jsonpath='{.status.phase}'
```

Inspect full status, including conditions and container states:

```bash
kubectl describe pod basic-pod
```

Look at:

- `Status:` (Pending / Running / Succeeded / Failed)
- `Conditions:` (PodScheduled, Initialized, ContainersReady, Ready)
- `Containers: ... State:` (Waiting / Running / Terminated)

## 1.6 Lab 2 — Multi-Container Pod (Sidecar Pattern)

A sidecar is a secondary container in the same Kubernetes Pod as the main application container, used to provide supporting or auxiliary functionality to the main application.

```
Sidecar ≠ simply "two containers in one Pod."
Main container + helper container = Sidecar pattern.
```

Create the following file: [`pod-multicontainer.yaml`](./pod-multicontainer.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
  labels:
    app: multi-container
spec:
  volumes:
    - name: shared-logs
      emptyDir: {}

  containers:
    - name: app
      image: busybox:1.36
      command:
        - sh
        - -c
        - >
          while true; do
            echo "$(date) - app is running" >> /var/log/app.log;
            sleep 5;
          done
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log

    - name: log-sidecar
      image: busybox:1.36
      command:
        - sh
        - -c
        - "tail -f /var/log/app.log"
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log
```

Apply it:

```bash
kubectl apply -f pod-multicontainer.yaml
```

Verify both containers are running:

```bash
kubectl get pod multi-container-pod
```

Expected `READY` column shows `2/2`.

View logs from a specific container:

```bash
kubectl logs multi-container-pod -c app
kubectl logs multi-container-pod -c log-sidecar
```

Exec into a specific container:

```bash
kubectl exec -it multi-container-pod -c app -- sh
```

Key point: containers in the same Pod share the network namespace (same IP, `localhost` between them) and can share storage via volumes, but each has its own filesystem otherwise.

## 1.7 Lab 3 — Init Containers

Create the following file: [`pod-init-container.yaml`](./pod-init-container.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-container-pod
  labels:
    app: init-demo
spec:
  volumes:
    - name: shared-data
      emptyDir: {}

  initContainers:
    - name: init-setup
      image: busybox:1.36
      command:
        - sh
        - -c
        - "echo 'Initialization complete' > /work-dir/status.txt && sleep 5"
      volumeMounts:
        - name: shared-data
          mountPath: /work-dir

  containers:
    - name: main-app
      image: nginx:1.27
      volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html
      ports:
        - containerPort: 80
```

Apply it and immediately watch the Pod:

```bash
kubectl apply -f pod-init-container.yaml
kubectl get pod init-container-pod -w
```

You should briefly see status `Init:0/1` before it moves to `Running`.

Check the init container's logs:

```bash
kubectl logs init-container-pod -c init-setup
```

Describe the Pod and note the two separate sections:

```bash
kubectl describe pod init-container-pod
```

Look for:

- `Init Containers:` (ran once, to completion, in order)
- `Containers:` (started only after all init containers succeeded)

Verify the init container's output was passed via the shared volume:

```bash
kubectl exec -it init-container-pod -c main-app -- cat /usr/share/nginx/html/status.txt
```

Key point: init containers run sequentially and **must exit successfully (exit code 0)** before any app container starts. If an init container fails, the Pod retries it with backoff and stays stuck in `Init:Error` / `Init:CrashLoopBackOff`.

## 1.8 Lab 4 — Probes (Startup, Liveness, Readiness)

Create the following file: [`pod-probes.yaml`](./pod-probes.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probes-pod
  labels:
    app: probes-demo
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80

      startupProbe:
        httpGet:
          path: /
          port: 80
        failureThreshold: 30
        periodSeconds: 2

      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 10
        failureThreshold: 3

      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 5
        failureThreshold: 3
```

Apply it:

```bash
kubectl apply -f pod-probes.yaml
```

Watch readiness flip to `True`:

```bash
kubectl get pod probes-pod -w
```

Inspect probe results and events:

```bash
kubectl describe pod probes-pod
```

**What each probe is for:**

| Probe | Purpose | On failure |
|---|---|---|
| `startupProbe` | Gives slow-starting apps time to boot before liveness/readiness kick in | Kubelet keeps waiting (up to `failureThreshold × periodSeconds`); liveness/readiness are disabled until it succeeds |
| `livenessProbe` | Detects a container that is running but stuck/deadlocked | Kubelet **restarts** the container |
| `readinessProbe` | Detects a container that is running but not ready to serve traffic | Pod is **removed from Service endpoints** (not restarted) |

## 1.9 Break the Pod — Failing Liveness Probe

Create the following file: [`pod-broken-probe.yaml`](./pod-broken-probe.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: broken-probe-pod
  labels:
    app: broken-probe
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80

      livenessProbe:
        httpGet:
          path: /wrong-path-that-does-not-exist
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 5
        failureThreshold: 2
```

Apply it:

```bash
kubectl apply -f pod-broken-probe.yaml
```

Watch the restart count climb:

```bash
kubectl get pod broken-probe-pod -w
```

You should see `RESTARTS` increasing and the Pod cycling through `Running` → back-off.

Investigate:

```bash
kubectl describe pod broken-probe-pod
```

Check the `Events:` section for:

```
Liveness probe failed: HTTP probe failed with statuscode: 404
```

Check restart count and last state:

```bash
kubectl get pod broken-probe-pod -o wide
kubectl get pod broken-probe-pod -o jsonpath='{.status.containerStatuses[0].restartCount}'
```

## 1.10 Recover the Pod

Fix it by editing the probe path back to `/`:

```bash
kubectl edit pod broken-probe-pod
```

> Note: most fields on a running Pod are immutable. In practice you would delete and re-apply rather than edit live — this is expected CKA behavior, not a mistake:

```bash
kubectl delete -f pod-broken-probe.yaml
```

Then apply the corrected `pod-probes.yaml` instead, and verify:

```bash
kubectl apply -f pod-probes.yaml
kubectl get pod probes-pod
```

## 1.11 Pod YAML Modification Exercise

Modify `pod-basic.yaml` to:

- add a second container named `sidecar` using image `busybox:1.36` running `sleep 3600`
- add a `readinessProbe` on the `nginx` container checking path `/` on port `80`
- add a `resources` block on the `nginx` container requesting `100m` CPU and `128Mi` memory

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: basic-pod
  labels:
    app: basic
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 5

    - name: sidecar
      image: busybox:1.36
      command: ["sleep", "3600"]
```

Apply and verify:

```bash
kubectl apply -f pod-basic.yaml
kubectl get pod basic-pod
```

## 1.12 CKA Practice Task

**Task**

Create a Pod named:

```
webserver
```

Requirements:

- namespace: `default`
- image: `nginx:1.28`
- container name: `nginx`
- container port: `80`
- add an `initContainer` named `init-check` (image `busybox:1.36`) that runs `sleep 3` before the main container starts
- add a `livenessProbe` on `/` port `80` with `initialDelaySeconds: 5`

Then:

1. Verify the init container completed before `nginx` started.
2. Verify the Pod reaches `Running` and `1/1 Ready`.
3. Describe the Pod and confirm the liveness probe is configured.

**Target Time**

5 minutes

Try it without looking at previous commands.

## 1.13 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

What are the possible values of `.status.phase` for a Pod, and what does each mean?

**Question 2**

What is the difference between a container in `CrashLoopBackOff` versus one stuck in `Waiting` with reason `ContainerCreating`?

**Question 3**

Why do init containers run sequentially instead of in parallel like regular containers?

**Question 4**

A Pod shows `READY 0/1` but `STATUS Running`. What's the most likely cause, and which probe is responsible?

**Question 5**

What is the practical difference between a failing `livenessProbe` and a failing `readinessProbe` from a user's perspective hitting a Service?

**Question 6**

Why can't you edit most fields (like the container image) on a live Pod directly, and what do you do instead?

## 1.14 Useful Commands

```bash
kubectl get pods
kubectl get pods -o wide
kubectl get pod <name> -w

kubectl describe pod <name>

kubectl logs <pod-name>
kubectl logs <pod-name> -c <container-name>
kubectl logs <pod-name> --previous

kubectl exec -it <pod-name> -- sh
kubectl exec -it <pod-name> -c <container-name> -- sh

kubectl get pod <name> -o jsonpath='{.status.phase}'
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[0].restartCount}'

kubectl delete pod <name>
kubectl delete -f <file>.yaml
```

## 1.15 Cleanup

```bash
kubectl delete pod basic-pod
kubectl delete pod multi-container-pod
kubectl delete pod init-container-pod
kubectl delete pod probes-pod
kubectl delete pod broken-probe-pod
kubectl delete pod webserver
```

Or:

```bash
kubectl delete -f pod-basic.yaml
kubectl delete -f pod-multicontainer.yaml
kubectl delete -f pod-init-container.yaml
kubectl delete -f pod-probes.yaml
kubectl delete -f pod-broken-probe.yaml
```

## 1.16 Lab Checklist

- [ ] Created a basic single-container Pod
- [ ] Observed Pod phases and container states
- [ ] Created a multi-container Pod and viewed per-container logs
- [ ] Understood shared network/volume between containers in a Pod
- [ ] Created a Pod with an Init Container
- [ ] Confirmed init containers run to completion before app containers start
- [ ] Configured startup, liveness, and readiness probes
- [ ] Understood the difference between liveness and readiness failure behavior
- [ ] Created an intentional liveness probe failure
- [ ] Diagnosed the failure via `describe` and `Events`
- [ ] Recovered by replacing the broken Pod
- [ ] Completed the CKA task within 5 minutes

## Files for this lab

```
01-Pods-Fundamentals/
├── README.md
├── pod-basic.yaml
├── pod-multicontainer.yaml
├── pod-init-container.yaml
├── pod-probes.yaml
└── pod-broken-probe.yaml
```
