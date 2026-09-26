# Lab 11 — Helm Test

## Objectives
- Write test Pods using the `helm.sh/hook: test` annotation
- Run and interpret `helm test` results
- Structure tests for a multi-service chart
- Practice debugging two common test setup mistakes

## Prerequisites
- Labs 01–10 completed (test resources are technically a special case of hooks from Lab 10)
- Helm 3.x installed

---

## 1. What Helm Test Is For

`helm test` runs one or more Pods that exist purely to validate a release actually works after install/upgrade — e.g., can the app respond to a health check, can it reach its database. Unlike regular hooks, test Pods don't run automatically on install/upgrade; they run only when you explicitly invoke `helm test`.

## 2. Writing a Test Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ .Release.Name }}-test-connection
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: wget
      image: busybox
      command: ["wget"]
      args: ["{{ .Release.Name }}-myapp:{{ .Values.service.port }}"]
  restartPolicy: Never
```
This is exactly what `helm create`'s scaffold puts in `templates/tests/test-connection.yaml` — a minimal `wget` against the Service to confirm it's reachable.

## 3. Running Tests

```bash
helm test demo -n <namespace>
```
```
Pod demo-test-connection pending
Pod demo-test-connection succeeded
NOTES:
...
```

Exit code semantics: `helm test` exits non-zero if **any** test Pod fails (non-zero container exit code), making it directly usable as a CI pipeline gate after a deploy:
```bash
helm upgrade demo mychart -n prod && helm test demo -n prod
```

Get full logs from a specific test Pod after the fact:
```bash
kubectl logs demo-test-connection -n <namespace>
```

## 4. Structuring Tests for a Multi-Service Chart

For an umbrella chart with multiple services, give each test a clear, distinct name and keep them focused — one concern per test Pod, rather than one giant test script:

```
templates/tests/
├── test-frontend-connection.yaml
├── test-backend-connection.yaml
└── test-db-migration-applied.yaml
```
All test Pods tagged `helm.sh/hook: test` run together on a single `helm test` invocation; Helm reports pass/fail per Pod and an overall result.

---

## 5. Troubleshooting Exercise 1 — Test Pod Hangs

**Scenario:**
```bash
helm test demo -n <namespace>
```
The command hangs indefinitely, never returning pass or fail.

**Task:**
1. Run `kubectl get pods -n <namespace> -l helm.sh/hook=test` in another terminal and check the Pod's status — likely `CrashLoopBackOff` or repeatedly `Running` → restarting
2. Recognize the cause: the test Pod spec was missing `restartPolicy: Never` (or had the default `Always`), so a failing container keeps getting restarted by the kubelet instead of settling into a terminal `Failed`/`Succeeded` state that Helm can observe
3. Fix by adding `restartPolicy: Never` to the Pod spec (test Pods should always set this, same as Job-based hooks)
4. Re-run `helm test demo` and confirm it now returns a clear pass/fail instead of hanging

## 6. Troubleshooting Exercise 2 — False-Positive Pass

**Scenario:** `helm test` reports success, but the app is actually broken — a manual check shows the service isn't responding correctly.

**Task:**
1. Inspect the test Pod's actual command/logic — e.g. it might be doing `wget -q -O- <service>` without checking the response content or status code, so it "succeeds" as long as *any* TCP connection is made, even to a service returning a 500 error page
2. Recognize the gap: a test that only checks reachability, not correctness, gives false confidence
3. Fix by tightening the test — e.g. use `wget --spider` with explicit exit-code checking against a real health endpoint, or curl with `-f` (fail on HTTP error codes) rather than just confirming a TCP connection succeeded:
```yaml
command: ["sh", "-c"]
args: ["curl -f http://{{ .Release.Name }}-myapp:{{ .Values.service.port }}/healthz"]
```
4. Re-run against both a healthy and a deliberately broken deployment to confirm the test now correctly distinguishes the two

**Takeaway:** a test hook is only as good as what it actually asserts — "the port is open" and "the application is healthy" are very different claims, and interviewers will probe this distinction.

---

## Cleanup
```bash
helm uninstall demo -n <namespace>
```

---

Next: Lab 12 — Linting & Schema Validation, the last lab in Phase 4.
