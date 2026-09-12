# Kubernetes Hands-On Lab 28 — Troubleshooting Pods & Workloads

## 28.1 Objectives

By the end of this lab, you should be able to:

- Apply a systematic, repeatable diagnostic method to any broken Pod, rather than guessing
- Read and interpret exit codes and container states precisely
- Diagnose and fix `ImagePullBackOff`/`ErrImagePull` from several distinct root causes
- Diagnose and fix `CrashLoopBackOff` from an application-level failure (not infra)
- Diagnose and fix a Pod stuck in `Init:Error`/`Init:CrashLoopBackOff`
- Diagnose a container that starts but immediately exits (`Completed` when it shouldn't be)
- Diagnose a misconfigured `command`/`args` causing `CreateContainerError`
- Distinguish a Pod-level problem from a workload-controller-level problem (Deployment/ReplicaSet)
- Practice the full method end-to-end under CKA time pressure

## 28.2 The Systematic Diagnostic Method

This is the sequence to run through, in order, for **any** broken Pod — don't skip steps even when you think you already know the answer:

```
1. kubectl get pods                      -> STATUS column: what phase/state?
2. kubectl get pods -o wide              -> which Node? any obvious IP/scheduling issue?
3. kubectl describe pod <name>           -> Events (bottom), container States, restart count
4. kubectl logs <name> [-c container]    -> current container's stdout/stderr
5. kubectl logs <name> --previous        -> CRASHED container's last output (before restart wiped it)
6. kubectl get events --sort-by=.lastTimestamp -> cluster-wide recent events, useful when
                                                    the Pod itself has few Events
7. Check the CONTROLLER (Deployment/ReplicaSet/Job) if the Pod keeps getting recreated —
   the problem may be in the Pod TEMPLATE, not this one instance
```

The single most common exam-time mistake is jumping straight to `logs` and skipping `describe` — many failures (scheduling, image pull, volume mount) never produce application logs at all because the container never actually started.

## 28.3 Container State & Exit Code Reference

| Symptom | Likely Cause |
|---|---|
| `Pending` | Scheduling problem: resources, affinity/taints, no matching PV (Labs 07, 08, 09, 17) |
| `ImagePullBackOff` / `ErrImagePull` | Wrong image name/tag, private registry auth missing, registry unreachable |
| `CreateContainerConfigError` | Missing ConfigMap/Secret key referenced by the Pod (Lab 06) |
| `CreateContainerError` | Invalid `command`/`args`, bad volume mount, runtime-level config issue |
| `CrashLoopBackOff` | Container starts then exits — check exit code + `logs --previous` |
| `Completed` (unexpectedly) | Main process exited 0 — command ran once and finished instead of running as a daemon |
| `OOMKilled` (exit 137) | Memory limit exceeded (Lab 07) |
| Exit code 1 | Generic application error — check logs |
| Exit code 126 | Command found but not executable (permissions) |
| Exit code 127 | Command not found (typo in `command`, or binary not in image) |
| Exit code 137 | SIGKILL — OOMKilled OR manually killed (`kill -9`) |
| Exit code 143 | SIGTERM — graceful shutdown, often from `kubectl delete` honoring grace period |

## 28.4 Prerequisites

```bash
kubectl version --client
kubectl get nodes
```

## 28.5 Scenario 1 — ImagePullBackOff From a Typo'd Tag

```bash
kubectl run scenario1 --image=nginx:1.27.99-does-not-exist
```

Diagnose:

```bash
kubectl get pod scenario1
kubectl describe pod scenario1
```

Look for the exact Event message — it distinguishes a missing **tag** from a missing **repository** from an **auth** failure, each phrased slightly differently:

```
Failed to pull image "nginx:1.27.99-does-not-exist": rpc error: ... manifest unknown
```

Fix:

```bash
kubectl delete pod scenario1
kubectl run scenario1 --image=nginx:1.27
kubectl get pod scenario1
```

## 28.6 Scenario 2 — ImagePullBackOff From a Private Registry With No Credentials

```bash
kubectl run scenario2 --image=myprivateregistry.example.com/app:1.0
```

Diagnose:

```bash
kubectl describe pod scenario2
```

Look for:

```
Failed to pull image ...: unauthorized: authentication required
```

This is a **completely different root cause** than Scenario 1 despite the same `ImagePullBackOff` status — reinforcing why reading the exact Event text matters more than the status label alone. Fix (conceptually — requires a real registry credential):

```bash
kubectl create secret docker-registry regcred \
  --docker-server=myprivateregistry.example.com \
  --docker-username=<user> --docker-password=<pass>

kubectl patch pod scenario2 -p '{"spec":{"imagePullSecrets":[{"name":"regcred"}]}}' 2>/dev/null || \
  echo "Pod spec is immutable — delete and recreate with imagePullSecrets set"
```

## 28.7 Scenario 3 — CrashLoopBackOff From an Application Error

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: scenario3
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo 'Starting app...'; echo 'FATAL: config file not found' >&2; exit 1"]
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: scenario3
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo 'Starting app...'; echo 'FATAL: config file not found' >&2; exit 1"]
EOF
```

Diagnose — note the restart count climbing:

```bash
kubectl get pod scenario3
```

Check the CURRENT attempt's logs (may be a fresh crash with nothing useful yet):

```bash
kubectl logs scenario3
```

Check the PREVIOUS attempt's logs — this is where the actual error usually is:

```bash
kubectl logs scenario3 --previous
```

Check the exact exit code:

```bash
kubectl get pod scenario3 -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'
```

This is exit code `1` — a generic application-level failure, confirmed by the log message itself (`FATAL: config file not found`) rather than any Kubernetes-level infrastructure problem. The fix here would be application-specific (in this case, providing the missing config via a ConfigMap — Lab 06), not a `kubectl`/YAML fix.

## 28.8 Scenario 4 — Init Container Failure

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: scenario4
spec:
  initContainers:
    - name: init-check
      image: busybox:1.36
      command: ["sh", "-c", "echo 'Checking dependency...'; exit 1"]
  containers:
    - name: main
      image: nginx:1.27
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: scenario4
spec:
  initContainers:
    - name: init-check
      image: busybox:1.36
      command: ["sh", "-c", "echo 'Checking dependency...'; exit 1"]
  containers:
    - name: main
      image: nginx:1.27
EOF
```

Diagnose:

```bash
kubectl get pod scenario4
```

Status shows `Init:Error` or `Init:CrashLoopBackOff` — the main container (`nginx`) never even attempts to start, because init containers must succeed in order first (Lab 01).

```bash
kubectl logs scenario4 -c init-check
```

Note: for init container logs you MUST specify `-c <init-container-name>` — omitting it defaults to attempting the main container, which hasn't started and has no logs yet.

## 28.9 Scenario 5 — Unexpectedly "Completed" (Wrong Command Design)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: scenario5
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["echo", "This runs once and exits"]
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: scenario5
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["echo", "This runs once and exits"]
EOF
```

Diagnose:

```bash
kubectl get pod scenario5
```

Status: `Completed` — this is often mistaken for a bug when someone actually wanted a long-running Pod, but the `command` was written to do one thing and exit (exit code 0, no error at all). Kubernetes doesn't consider this a failure; a bare Pod simply won't restart a successfully-completed container without a controller like a Job (Lab 05) or a corrected long-running `command`.

```bash
kubectl logs scenario5
```

The log confirms exactly what happened — the command ran successfully and finished, which is precisely what was asked for. The "fix" here is recognizing this isn't a crash at all — the manifest's `command` needs a long-running process (e.g. `sleep infinity`, or the actual application binary) if a persistent Pod was intended.

## 28.10 Scenario 6 — Bad command Causes CreateContainerError

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: scenario6
spec:
  containers:
    - name: app
      image: nginx:1.27
      command: ["/this/binary/does/not/exist"]
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: scenario6
spec:
  containers:
    - name: app
      image: nginx:1.27
      command: ["/this/binary/does/not/exist"]
EOF
```

Diagnose:

```bash
kubectl get pod scenario6
kubectl describe pod scenario6
```

Look for:

```
Error: failed to create containerd task: ... executable file not found
```

This fails BEFORE the container process even starts, so there are no application logs — it's a container-runtime-level failure, not an app-level one. Fix by removing the incorrect `command` override (letting the image's own default `ENTRYPOINT` run) or providing a valid path.

## 28.11 Scenario 7 — Pod-Level Fix Doesn't Stick (Controller-Owned Pod)

Create a Deployment with the same broken command as Scenario 6:

```bash
kubectl create deployment scenario7 --image=nginx:1.27 -- /this/binary/does/not/exist
kubectl get pods -l app=scenario7
```

Try to fix the individual Pod directly:

```bash
kubectl delete pod -l app=scenario7
kubectl get pods -l app=scenario7
```

The ReplicaSet immediately creates a **new** Pod with the exact same broken `command` — deleting the Pod did nothing, because the problem lives in the Deployment's `spec.template`, not in any individual Pod instance. Fix at the correct level:

```bash
kubectl patch deployment scenario7 --type=json \
  -p='[{"op":"remove","path":"/spec/template/spec/containers/0/command"}]'

kubectl get pods -l app=scenario7 -w
```

This is a critical diagnostic instinct: **if a freshly-recreated Pod has the identical problem, stop touching Pods and go up a level** to the Deployment/ReplicaSet/Job/StatefulSet that owns it.

## 28.12 CKA Practice Task

**Task**

You'll be handed (simulate this yourself) 4 broken Pods/Deployments with different root causes. For each, using ONLY the diagnostic method from 28.2:

1. Identify whether the problem is: scheduling, image pull, container runtime, application-level, or controller-template-level.
2. State the exact command that revealed the root cause.
3. Apply the fix at the correct level (Pod vs controller).

Create these four yourself to practice against:

```bash
kubectl run task-a --image=redis:1.0.0-nonexistent
kubectl run task-b --image=busybox:1.36 --command -- sh -c "exit 3"
kubectl create deployment task-c --image=nginx:1.27 -- /bin/false
kubectl run task-d --image=nginx:1.27 --requests=cpu=64
```

**Target Time**

10 minutes for all four

## 28.13 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

Why does `kubectl logs` sometimes return nothing at all for a clearly broken Pod, and what should you check instead?

**Question 2**

What's the specific difference between `kubectl logs <pod>` and `kubectl logs <pod> --previous`, and when is `--previous` essential?

**Question 3**

A Pod shows `Init:Error`. Why won't `kubectl logs <pod>` (without `-c`) show you anything useful here?

**Question 4**

A Pod shows `STATUS: Completed` and the user expected it to keep running. Is this a Kubernetes failure? What actually happened?

**Question 5**

You delete a broken Pod and an identical, equally broken Pod reappears seconds later. What does this tell you about where to actually apply the fix?

**Question 6**

What's the practical difference in troubleshooting approach between exit code 1, exit code 127, and exit code 137?

## 28.14 Useful Commands

```bash
kubectl get pods
kubectl get pods -o wide
kubectl describe pod <name>
kubectl logs <name> [-c <container>]
kubectl logs <name> --previous
kubectl get events --sort-by=.lastTimestamp

kubectl get pod <name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[0].restartCount}'

kubectl patch deployment <name> --type=json -p='[...]'
```

## 28.15 Cleanup

```bash
kubectl delete pod scenario1 scenario2 scenario3 scenario4 scenario5 scenario6 \
  task-a task-b task-d
kubectl delete deployment scenario7 task-c
kubectl delete secret regcred
```

## 28.16 Lab Checklist

- [ ] Internalized the systematic diagnostic sequence (get → describe → logs → previous → events → controller)
- [ ] Diagnosed two distinct `ImagePullBackOff` root causes (bad tag vs missing registry auth)
- [ ] Diagnosed `CrashLoopBackOff` from an application-level error using `logs --previous`
- [ ] Diagnosed an `Init:Error` and correctly targeted `-c <init-container>` for logs
- [ ] Recognized an unexpected `Completed` state as a command-design issue, not a crash
- [ ] Diagnosed a `CreateContainerError` from a bad `command` path
- [ ] Recognized when a fix must go to the Deployment/controller, not the Pod
- [ ] Read and correctly interpreted exit codes 1, 126, 127, 137, 143
- [ ] Completed the CKA-style multi-scenario task within 10 minutes
