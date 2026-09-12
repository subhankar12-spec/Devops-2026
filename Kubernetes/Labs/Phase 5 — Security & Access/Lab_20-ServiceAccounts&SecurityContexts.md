# Kubernetes Hands-On Lab 20 — ServiceAccounts & Security Contexts

## 20.1 Objectives

By the end of this lab, you should be able to:

- Understand how a Pod gets its ServiceAccount token automatically
- Control token automounting with `automountServiceAccountToken`
- Attach an `imagePullSecrets` to a ServiceAccount for private registries
- Set Pod-level and container-level Security Contexts
- Use `runAsUser`, `runAsGroup`, `runAsNonRoot`, and `fsGroup`
- Drop and add Linux capabilities
- Use `readOnlyRootFilesystem` and understand what breaks when you enable it
- Understand `allowPrivilegeEscalation` and `privileged`
- Troubleshoot a container that crashes due to `runAsNonRoot` enforcement
- Perform common ServiceAccount/SecurityContext tasks quickly for the CKA

## 20.2 Architecture

```
                        Pod
                         |
        +----------------+----------------+
        |                                 |
        v                                 v
   ServiceAccount                  SecurityContext
   (WHO the Pod authenticates      (WHAT the container is
    as, to the API server —        ALLOWED to do at the
    ties into RBAC from Lab 19)    OS/kernel level — user ID,
                                    capabilities, filesystem
        |                          writability, privilege
        v                          escalation)
   auto-mounted token at
   /var/run/secrets/kubernetes.io/
   serviceaccount/token
```

Key concept: a ServiceAccount governs a Pod's identity **against the Kubernetes API** (what `kubectl`/API calls it can make). A SecurityContext governs what the container process can do **at the OS/kernel level** on the Node (which user ID it runs as, which Linux capabilities it has, whether it can write to its own root filesystem). They solve completely different problems and are frequently confused.

## 20.3 Prerequisites

```bash
kubectl version --client
kubectl get nodes
```

## 20.4 Lab 1 — Default ServiceAccount Token Mounting

Every namespace has a `default` ServiceAccount, and every Pod gets one unless told otherwise:

```bash
kubectl run default-sa-pod --image=busybox:1.36 --command -- sleep 3600
kubectl get pod default-sa-pod -o jsonpath='{.spec.serviceAccountName}'
```

Confirm the token is mounted inside the container:

```bash
kubectl exec default-sa-pod -- ls /var/run/secrets/kubernetes.io/serviceaccount/
kubectl exec default-sa-pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

This token is what any in-cluster tooling (like a client library or `kubectl` running inside a Pod) uses to authenticate to the API server — and its permissions are exactly whatever RBAC grants the `default` ServiceAccount (usually nothing, by design).

## 20.5 Disable Auto-Mounting

For Pods that never need to talk to the API server, disabling the token mount reduces attack surface:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-token-pod
spec:
  automountServiceAccountToken: false
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
```

Apply it and confirm no token directory exists:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: no-token-pod
spec:
  automountServiceAccountToken: false
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
EOF

kubectl exec no-token-pod -- ls /var/run/secrets/kubernetes.io/serviceaccount/
```

Expected: `No such file or directory`.

## 20.6 Lab 2 — imagePullSecrets on a ServiceAccount

Rather than adding `imagePullSecrets` to every Pod spec individually, attach it once to a ServiceAccount so every Pod using it inherits the credential automatically:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<user> \
  --docker-password=<password> \
  --docker-email=<email>
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: private-registry-sa
imagePullSecrets:
  - name: regcred
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: private-registry-sa
imagePullSecrets:
  - name: regcred
EOF
```

Any Pod using `serviceAccountName: private-registry-sa` automatically gets pull credentials without repeating them per-Pod.

## 20.7 Lab 3 — Pod-Level Security Context

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "id; sleep 3600"]
```

Apply it and check the effective identity:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "id; sleep 3600"]
EOF

kubectl logs secure-pod
```

Expect output like `uid=1000 gid=3000 groups=3000,2000` — the container runs as a non-root user even though the image itself might default to root, and any volume it writes to gets group ownership `2000` (`fsGroup`) automatically.

## 20.8 runAsNonRoot Enforcement

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nonroot-enforced-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  containers:
    - name: app
      image: nginx:1.27
```

Apply it — this will actually FAIL to start, because the stock `nginx` image's entrypoint expects to run as root (to bind port 80 and write PID files) even though we told it to run as UID 1000:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nonroot-enforced-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  containers:
    - name: app
      image: nginx:1.27
EOF

kubectl get pod nonroot-enforced-pod
```

Keep this Pod around — it's the basis for the troubleshooting exercise in 20.12.

## 20.9 Lab 4 — Container-Level Security Context (Capabilities)

Security contexts can be set at the container level too, overriding Pod-level defaults for that one container:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: capabilities-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      securityContext:
        capabilities:
          drop:
            - ALL
          add:
            - NET_BIND_SERVICE
        allowPrivilegeEscalation: false
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: capabilities-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      securityContext:
        capabilities:
          drop:
            - ALL
          add:
            - NET_BIND_SERVICE
        allowPrivilegeEscalation: false
EOF

kubectl get pod capabilities-pod
```

`drop: [ALL]` removes every Linux capability the container would otherwise have by default, then `add: [NET_BIND_SERVICE]` grants back only the one needed (binding to ports below 1024) — the principle of least privilege in practice. `allowPrivilegeEscalation: false` additionally blocks a process from gaining more privileges than its parent (e.g. via setuid binaries).

## 20.10 readOnlyRootFilesystem

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readonly-fs-pod
spec:
  containers:
    - name: app
      image: nginx:1.27
      securityContext:
        readOnlyRootFilesystem: true
      volumeMounts:
        - name: cache-vol
          mountPath: /var/cache/nginx
        - name: run-vol
          mountPath: /var/run
  volumes:
    - name: cache-vol
      emptyDir: {}
    - name: run-vol
      emptyDir: {}
```

Apply it — note the two `emptyDir` mounts for the specific paths NGINX needs to write to even in normal operation (cache and PID file):

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: readonly-fs-pod
spec:
  containers:
    - name: app
      image: nginx:1.27
      securityContext:
        readOnlyRootFilesystem: true
      volumeMounts:
        - name: cache-vol
          mountPath: /var/cache/nginx
        - name: run-vol
          mountPath: /var/run
  volumes:
    - name: cache-vol
      emptyDir: {}
    - name: run-vol
      emptyDir: {}
EOF

kubectl get pod readonly-fs-pod
```

Confirm the root filesystem really is read-only:

```bash
kubectl exec readonly-fs-pod -- sh -c "echo test > /test-file.txt"
```

Expected: `Read-only file system` error — this is exactly the intended hardening, and exactly why you must identify and explicitly mount every path the application legitimately needs to write to.

## 20.11 privileged Containers (Understand, Don't Overuse)

```yaml
securityContext:
  privileged: true
```

`privileged: true` gives a container almost all the same access as processes on the Node itself — full device access, ability to modify kernel parameters, effectively a container escape by design. Legitimate uses are rare and specific: certain CNI plugins, storage drivers, or monitoring agents that genuinely need Node-level access. You will not create a `privileged: true` Pod in this lab — recognize it as a major red flag in any manifest you review, and know that Pod Security Standards / Pod Security Admission (the modern replacement for PodSecurityPolicy) can block it cluster-wide at the `restricted` level.

## 20.12 Diagnose and Recover — runAsNonRoot Failure

Revisit the Pod from 20.8:

```bash
kubectl describe pod nonroot-enforced-pod
```

Look for an Event/message like:

```
Error: container has runAsNonRoot and image will run as root
```

Fix it — either use an image built to run as non-root, or (for `nginx` specifically) use the `nginxinc/nginx-unprivileged` image designed for exactly this case:

```bash
kubectl delete pod nonroot-enforced-pod

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: fixed-nonroot-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  containers:
    - name: app
      image: nginxinc/nginx-unprivileged:1.27
      ports:
        - containerPort: 8080
EOF

kubectl get pod fixed-nonroot-pod
```

## 20.13 CKA Practice Task

**Task**

1. Create a ServiceAccount `cka-sa` with `automountServiceAccountToken: false`.
2. Create a Pod `cka-secure-pod` using `cka-sa`, with Pod-level `runAsUser: 2000`, `fsGroup: 3000`.
3. On the same Pod's single container, set `securityContext.capabilities.drop: [ALL]` and `allowPrivilegeEscalation: false`.
4. Confirm via `kubectl exec ... -- id` that the container runs as UID 2000.
5. Confirm no ServiceAccount token is mounted in the container.

**Target Time**

7 minutes

Try it without looking at previous commands.

## 20.14 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

What's the fundamental difference in scope between a ServiceAccount and a SecurityContext?

**Question 2**

Why would you set `automountServiceAccountToken: false` on a Pod that never calls the Kubernetes API?

**Question 3**

A Pod with `runAsNonRoot: true` fails to start with an error about the image running as root. What are your two options to fix it?

**Question 4**

What's the security benefit of `capabilities.drop: [ALL]` followed by adding back only specific capabilities, versus leaving the default capability set?

**Question 5**

You set `readOnlyRootFilesystem: true` on an application container and it now crashes on startup. What's the most likely cause and how do you fix it without disabling the setting?

**Question 6**

Why is `privileged: true` considered dangerous, and what modern Kubernetes feature can prevent it from being used at all in a namespace?

## 20.15 Useful Commands

```bash
kubectl get sa -n <namespace>
kubectl describe sa <name> -n <namespace>

kubectl exec <pod> -- ls /var/run/secrets/kubernetes.io/serviceaccount/
kubectl exec <pod> -- id

kubectl create secret docker-registry <name> \
  --docker-server=<server> --docker-username=<user> \
  --docker-password=<pass> --docker-email=<email>

kubectl describe pod <name>
```

## 20.16 Cleanup

```bash
kubectl delete pod default-sa-pod no-token-pod secure-pod nonroot-enforced-pod \
  capabilities-pod readonly-fs-pod fixed-nonroot-pod cka-secure-pod

kubectl delete sa private-registry-sa cka-sa
kubectl delete secret regcred
```

## 20.17 Lab Checklist

- [ ] Confirmed the default ServiceAccount token is auto-mounted into Pods
- [ ] Disabled auto-mounting with `automountServiceAccountToken: false`
- [ ] Attached `imagePullSecrets` to a ServiceAccount
- [ ] Set Pod-level `runAsUser`, `runAsGroup`, and `fsGroup`
- [ ] Reproduced a `runAsNonRoot` startup failure
- [ ] Used container-level `capabilities.drop`/`add` and `allowPrivilegeEscalation: false`
- [ ] Enabled `readOnlyRootFilesystem` and mounted writable exceptions where needed
- [ ] Understood the risk profile of `privileged: true`
- [ ] Diagnosed and fixed the `runAsNonRoot` failure with an appropriate image
- [ ] Completed the CKA task within 7 minutes
