# Kubernetes Hands-On Lab 06 — ConfigMaps & Secrets

## 6.1 Objectives

By the end of this lab, you should be able to:

- Create ConfigMaps from literals, files, and YAML
- Consume ConfigMaps as environment variables and as mounted volumes
- Create Secrets (generic, and from literals)
- Understand Secret encoding (base64) vs encryption
- Consume Secrets as environment variables and as mounted volumes
- Understand the difference between `envFrom` and individual `env.valueFrom`
- Understand what happens to a running Pod when a mounted ConfigMap/Secret is updated
- Troubleshoot a Pod stuck in `CreateContainerConfigError` due to a missing ConfigMap/Secret key
- Perform common ConfigMap/Secret tasks quickly for the CKA

## 6.2 Architecture

```
        ConfigMap                      Secret
     (non-sensitive config)      (sensitive data, base64-encoded)
             |                            |
     +-------+-------+           +--------+--------+
     |               |           |                 |
     v               v           v                 v
env variable   mounted volume  env variable   mounted volume
(one-time,     (file per key,  (one-time,     (file per key,
 read at        can be          read at        can be
 container      updated live)   container      updated live)
 start)                         start)
```

Key concept: env vars sourced from a ConfigMap/Secret are injected **once**, at container start. Volume-mounted ConfigMaps/Secrets are periodically synced by the kubelet and **do** update the files inside the running container — but the application itself must notice the file changed (Kubernetes doesn't restart it for you).

## 6.3 Prerequisites

```bash
kubectl version --client
kubectl get nodes
```

## 6.4 Lab 1 — Create a ConfigMap (Imperative)

From literals:

```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info
```

Verify:

```bash
kubectl get configmap app-config -o yaml
```

From a file:

```bash
echo "server.port=8080" > app.properties
echo "server.timeout=30" >> app.properties

kubectl create configmap file-config --from-file=app.properties
kubectl get configmap file-config -o yaml
```

## 6.5 Create a ConfigMap (Declarative YAML)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-config
data:
  APP_ENV: staging
  LOG_LEVEL: debug
  nginx.conf: |
    server {
      listen 80;
      location / {
        return 200 'Hello from ConfigMap-mounted config';
      }
    }
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-config
data:
  APP_ENV: staging
  LOG_LEVEL: debug
  nginx.conf: |
    server {
      listen 80;
      location / {
        return 200 'Hello from ConfigMap-mounted config';
      }
    }
EOF
```

## 6.6 Consume a ConfigMap as Environment Variables

Individual keys via `valueFrom`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-env-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo APP_ENV=$APP_ENV LOG_LEVEL=$LOG_LEVEL; sleep 3600"]
      env:
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: web-config
              key: APP_ENV
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: web-config
              key: LOG_LEVEL
```

Apply and check:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: config-env-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo APP_ENV=\$APP_ENV LOG_LEVEL=\$LOG_LEVEL; sleep 3600"]
      env:
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: web-config
              key: APP_ENV
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: web-config
              key: LOG_LEVEL
EOF

kubectl logs config-env-pod
```

All keys at once via `envFrom`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-envfrom-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "env | grep -E 'APP_ENV|LOG_LEVEL'; sleep 3600"]
      envFrom:
        - configMapRef:
            name: web-config
```

Apply and check:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: config-envfrom-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "env | grep -E 'APP_ENV|LOG_LEVEL'; sleep 3600"]
      envFrom:
        - configMapRef:
            name: web-config
EOF

kubectl logs config-envfrom-pod
```

Note: `envFrom` pulls in **every** key from the ConfigMap as an env var (including `nginx.conf`, which becomes a messy multi-line env var) — use individual `valueFrom` when you only need specific keys.

## 6.7 Consume a ConfigMap as a Mounted Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-volume-pod
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      volumeMounts:
        - name: config-vol
          mountPath: /etc/nginx/conf.d
  volumes:
    - name: config-vol
      configMap:
        name: web-config
        items:
          - key: nginx.conf
            path: default.conf
```

Apply and verify the file exists inside the container:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: config-volume-pod
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      volumeMounts:
        - name: config-vol
          mountPath: /etc/nginx/conf.d
  volumes:
    - name: config-vol
      configMap:
        name: web-config
        items:
          - key: nginx.conf
            path: default.conf
EOF

kubectl exec config-volume-pod -- cat /etc/nginx/conf.d/default.conf
```

## 6.8 Lab 2 — Create a Secret

Generic Secret from literals:

```bash
kubectl create secret generic db-credentials \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=SuperSecret123
```

Verify — note the values are hidden by default:

```bash
kubectl get secret db-credentials -o yaml
```

Decode a value manually to confirm it's only base64, **not encrypted**:

```bash
kubectl get secret db-credentials -o jsonpath='{.data.DB_PASSWORD}' | base64 --decode
```

> Important CKA/interview point: base64 is **encoding**, not encryption. Anyone with API access to read the Secret object can trivially decode it. Real protection comes from RBAC restricting who can `get`/`list` Secrets, plus (in production) encryption at rest for etcd and tools like Sealed Secrets or an external secrets manager.

## 6.9 Consume a Secret as Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo DB_USER=$DB_USER; sleep 3600"]
      env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: DB_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: DB_PASSWORD
```

Apply and check:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo DB_USER=\$DB_USER; sleep 3600"]
      env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: DB_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: DB_PASSWORD
EOF

kubectl logs secret-env-pod
kubectl exec secret-env-pod -- env | grep DB_
```

## 6.10 Consume a Secret as a Mounted Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: secret-vol
          mountPath: /etc/secrets
          readOnly: true
  volumes:
    - name: secret-vol
      secret:
        secretName: db-credentials
```

Apply and check the mounted files (one file per key, containing the decoded value):

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: secret-vol
          mountPath: /etc/secrets
          readOnly: true
  volumes:
    - name: secret-vol
      secret:
        secretName: db-credentials
EOF

kubectl exec secret-volume-pod -- ls /etc/secrets
kubectl exec secret-volume-pod -- cat /etc/secrets/DB_USER
```

## 6.11 Live Update Behavior

Update the ConfigMap:

```bash
kubectl patch configmap web-config --type merge -p '{"data":{"LOG_LEVEL":"warn"}}'
```

Check the **volume-mounted** Pod — the file updates within ~60 seconds (kubelet sync period), no restart needed:

```bash
kubectl exec config-volume-pod -- cat /etc/nginx/conf.d/default.conf
```

Check the **env-var** Pod — it does NOT update, because env vars are injected once at container start:

```bash
kubectl exec config-envfrom-pod -- env | grep LOG_LEVEL
```

This is a critical distinction: if an app needs to pick up config changes without a restart, it must read from a mounted file (and watch for changes) — env vars require the Pod to be recreated.

## 6.12 Break It — Missing ConfigMap Key

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: broken-config-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      env:
        - name: MISSING_VALUE
          valueFrom:
            configMapKeyRef:
              name: web-config
              key: KEY_THAT_DOES_NOT_EXIST
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: broken-config-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      env:
        - name: MISSING_VALUE
          valueFrom:
            configMapKeyRef:
              name: web-config
              key: KEY_THAT_DOES_NOT_EXIST
EOF
```

## 6.13 Diagnose and Recover

```bash
kubectl get pod broken-config-pod
```

Status shows `CreateContainerConfigError`. Investigate:

```bash
kubectl describe pod broken-config-pod
```

Look for an Event like:

```
Error: couldn't find key KEY_THAT_DOES_NOT_EXIST in ConfigMap default/web-config
```

Fix by pointing to a key that actually exists:

```bash
kubectl delete pod broken-config-pod
kubectl run broken-config-pod --image=busybox:1.36 --command -- sh -c "sleep 3600"
```

(In practice you'd correct the YAML's `key:` field and re-apply — the fix here is conceptual since Pod spec is immutable.)

## 6.14 CKA Practice Task

**Task**

1. Create a ConfigMap named `cka-config` with keys `GREETING=hello` and `TARGET=world`.
2. Create a Secret named `cka-secret` with key `API_KEY=abc123`.
3. Create a Pod named `cka-app` (image `busybox:1.36`, command `sleep 3600`) that:
   - injects both ConfigMap keys as env vars using `envFrom`
   - injects `API_KEY` as an individual env var using `valueFrom`
4. Exec into the Pod and confirm all three env vars are present.
5. Mount `cka-secret` as a volume at `/etc/api` in the same Pod and confirm the `API_KEY` file exists.

**Target Time**

7 minutes

Try it without looking at previous commands.

## 6.15 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

What causes a Pod to enter `CreateContainerConfigError`, and how do you diagnose which key/ConfigMap/Secret is missing?

**Question 2**

Why don't environment variables sourced from a ConfigMap update after you edit the ConfigMap, while a mounted volume's contents do?

**Question 3**

Is a Kubernetes Secret encrypted at rest by default? What actually protects Secret data?

**Question 4**

What's the difference between `envFrom` and `env: - valueFrom:` when consuming a ConfigMap?

**Question 5**

If you need an application to reload config without a Pod restart, should you use env vars or a mounted volume, and why?

**Question 6**

What command would you use to view the decoded (plaintext) value of a Secret key from the CLI?

## 6.16 Useful Commands

```bash
kubectl create configmap <name> --from-literal=KEY=value
kubectl create configmap <name> --from-file=<path>
kubectl get configmap <name> -o yaml
kubectl describe configmap <name>
kubectl patch configmap <name> --type merge -p '{"data":{"KEY":"newvalue"}}'

kubectl create secret generic <name> --from-literal=KEY=value
kubectl get secret <name> -o yaml
kubectl get secret <name> -o jsonpath='{.data.KEY}' | base64 --decode

kubectl exec <pod> -- env
kubectl exec <pod> -- cat <mounted-file-path>

kubectl describe pod <pod-name>
```

## 6.17 Cleanup

```bash
kubectl delete pod config-env-pod config-envfrom-pod config-volume-pod \
  secret-env-pod secret-volume-pod broken-config-pod cka-app

kubectl delete configmap app-config file-config web-config cka-config
kubectl delete secret db-credentials cka-secret
```

## 6.18 Lab Checklist

- [ ] Created ConfigMaps imperatively (literals and files) and declaratively (YAML)
- [ ] Consumed a ConfigMap via individual `valueFrom` env vars
- [ ] Consumed a ConfigMap via `envFrom`
- [ ] Mounted a ConfigMap as a volume and read the file inside a container
- [ ] Created a Secret and decoded its value from the CLI
- [ ] Consumed a Secret via env vars and via a mounted volume
- [ ] Confirmed volume-mounted config updates live; env-var config does not
- [ ] Reproduced `CreateContainerConfigError` from a missing key
- [ ] Diagnosed the missing key via `describe pod` events
- [ ] Completed the CKA task within 7 minutes
