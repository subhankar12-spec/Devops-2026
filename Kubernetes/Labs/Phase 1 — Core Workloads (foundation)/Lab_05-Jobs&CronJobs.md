# Kubernetes Hands-On Lab 05 — Jobs & CronJobs

## 5.1 Objectives

By the end of this lab, you should be able to:

- Create a Job that runs to completion
- Understand `completions`, `parallelism`, and `backoffLimit`
- Run parallel Jobs
- Understand Job Pod failure/retry behavior
- Create a CronJob on a schedule
- Understand `concurrencyPolicy`, `startingDeadlineSeconds`, `successfulJobsHistoryLimit`, `failedJobsHistoryLimit`
- Manually trigger a CronJob run on demand
- Troubleshoot a Job stuck retrying due to a failing container
- Perform common Job/CronJob tasks quickly for the CKA

## 5.2 Architecture

```
                     Job
                (completions: 3)
                (parallelism: 1)
                      |
        runs Pods one at a time until
        3 have completed successfully
                      |
              +-------+-------+
              v       v       v
            Pod-1   Pod-2   Pod-3
           (runs   (runs   (runs
            once)   once)   once)

                   CronJob
              (schedule: "*/2 * * * *")
                      |
        creates a new Job at each
        scheduled tick
                      |
                  +---+---+
                  v       v
                Job-1   Job-2
             (from run  (from run
              at T+0)    at T+2m)
```

Key concept: a Job's Pods use `restartPolicy: Never` or `OnFailure` — never `Always`. A Job exists to run something **to completion**, not to keep it running forever like a Deployment does.

## 5.3 Prerequisites

```bash
kubectl version --client
kubectl get nodes
```

## 5.4 Lab 1 — Create a Basic Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: basic-job
spec:
  template:
    spec:
      containers:
        - name: worker
          image: busybox:1.36
          command: ["sh", "-c", "echo Processing started; sleep 10; echo Processing complete"]
      restartPolicy: Never
  backoffLimit: 4
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: basic-job
spec:
  template:
    spec:
      containers:
        - name: worker
          image: busybox:1.36
          command: ["sh", "-c", "echo Processing started; sleep 10; echo Processing complete"]
      restartPolicy: Never
  backoffLimit: 4
EOF
```

## 5.5 Verify the Job

```bash
kubectl get jobs
kubectl get pods -l job-name=basic-job
```

Watch it reach completion:

```bash
kubectl get job basic-job -w
```

Check logs:

```bash
kubectl logs -l job-name=basic-job
```

Inspect it:

```bash
kubectl describe job basic-job
```

Check `Completions:` and `Pods Statuses:`.

## 5.6 Completions and Parallelism

Create a Job that must complete 5 times, running 2 Pods in parallel at once:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-job
spec:
  completions: 5
  parallelism: 2
  template:
    spec:
      containers:
        - name: worker
          image: busybox:1.36
          command: ["sh", "-c", "echo Task running; sleep 5"]
      restartPolicy: Never
  backoffLimit: 4
```

Apply it and watch Pods come and go in waves of 2:

```bash
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-job
spec:
  completions: 5
  parallelism: 2
  template:
    spec:
      containers:
        - name: worker
          image: busybox:1.36
          command: ["sh", "-c", "echo Task running; sleep 5"]
      restartPolicy: Never
  backoffLimit: 4
EOF

kubectl get pods -l job-name=parallel-job -w
```

Verify total completions:

```bash
kubectl get job parallel-job
```

`COMPLETIONS` should read `5/5` once finished, even though only 2 ran at a time.

## 5.7 Job Failure and Retry Behavior

Create a Job that always fails:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: failing-job
spec:
  backoffLimit: 3
  template:
    spec:
      containers:
        - name: worker
          image: busybox:1.36
          command: ["sh", "-c", "echo This will fail; exit 1"]
      restartPolicy: Never
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: failing-job
spec:
  backoffLimit: 3
  template:
    spec:
      containers:
        - name: worker
          image: busybox:1.36
          command: ["sh", "-c", "echo This will fail; exit 1"]
      restartPolicy: Never
EOF
```

Watch it retry, then give up:

```bash
kubectl get pods -l job-name=failing-job -w
```

You'll see multiple Pods created — one per retry — until `backoffLimit` (3) is exceeded, then the Job reports `Failed`:

```bash
kubectl describe job failing-job
```

Look for `Status: Failed` and condition `type: Failed, reason: BackoffLimitExceeded`.

Key point: `restartPolicy: Never` means the **container** doesn't restart in place — instead the Job controller creates a brand-new Pod for each retry. With `restartPolicy: OnFailure`, the same Pod's container restarts in place instead.

## 5.8 Lab 2 — Create a CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cronjob
spec:
  schedule: "*/2 * * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          containers:
            - name: hello
              image: busybox:1.36
              command: ["sh", "-c", "date; echo Hello from the CronJob"]
          restartPolicy: OnFailure
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cronjob
spec:
  schedule: "*/2 * * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          containers:
            - name: hello
              image: busybox:1.36
              command: ["sh", "-c", "date; echo Hello from the CronJob"]
          restartPolicy: OnFailure
EOF
```

## 5.9 Observe the CronJob

```bash
kubectl get cronjob hello-cronjob
```

Check `SCHEDULE`, `LAST SCHEDULE`, and `ACTIVE` columns. Wait ~2 minutes and list the Jobs it created:

```bash
kubectl get jobs
kubectl get jobs -l job-name --show-labels
```

Check Pod logs from a run:

```bash
kubectl get pods -l job-name --sort-by=.metadata.creationTimestamp
kubectl logs <pod-name-from-a-cronjob-run>
```

Inspect scheduling details and history limits:

```bash
kubectl describe cronjob hello-cronjob
```

**Key CronJob fields:**

| Field | Purpose |
|---|---|
| `schedule` | Standard cron syntax (`min hour day month weekday`) |
| `concurrencyPolicy` | `Allow` (default, runs overlap) / `Forbid` (skip new run if previous still active) / `Replace` (cancel current, start new) |
| `startingDeadlineSeconds` | How late a missed run can start before being skipped entirely |
| `successfulJobsHistoryLimit` | How many completed Jobs to keep (default 3) |
| `failedJobsHistoryLimit` | How many failed Jobs to keep (default 1) |
| `suspend` | Set `true` to pause scheduling without deleting the CronJob |

## 5.10 Manually Trigger a CronJob Run

You don't have to wait for the schedule — create a one-off Job from the CronJob's template:

```bash
kubectl create job manual-run-1 --from=cronjob/hello-cronjob
```

Verify:

```bash
kubectl get jobs
kubectl logs -l job-name=manual-run-1
```

This is the standard way to test a CronJob's logic without waiting for its schedule, and a common CKA/production task.

## 5.11 Suspend and Resume a CronJob

Pause future scheduled runs without deleting anything:

```bash
kubectl patch cronjob hello-cronjob -p '{"spec":{"suspend":true}}'
kubectl get cronjob hello-cronjob
```

Resume:

```bash
kubectl patch cronjob hello-cronjob -p '{"spec":{"suspend":false}}'
```

## 5.12 Break It — Job Stuck Retrying

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: broken-job
spec:
  backoffLimit: 6
  template:
    spec:
      containers:
        - name: worker
          image: busybox:1.36
          command: ["sh", "-c", "exit 1"]
      restartPolicy: OnFailure
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: broken-job
spec:
  backoffLimit: 6
  template:
    spec:
      containers:
        - name: worker
          image: busybox:1.36
          command: ["sh", "-c", "exit 1"]
      restartPolicy: OnFailure
EOF
```

Watch the single Pod cycle through restarts (note: with `OnFailure`, it's the **same Pod** restarting, not new Pods each time):

```bash
kubectl get pods -l job-name=broken-job -w
```

## 5.13 Diagnose and Recover

```bash
kubectl describe job broken-job
kubectl logs -l job-name=broken-job
```

Check the restart count and exit code:

```bash
kubectl get pod -l job-name=broken-job -o jsonpath='{.items[0].status.containerStatuses[0].restartCount}'
kubectl get pod -l job-name=broken-job -o jsonpath='{.items[0].status.containerStatuses[0].lastState.terminated.exitCode}'
```

Since a Job's `spec.template` is immutable once created, you can't patch the command in place — delete and recreate with a fixed command:

```bash
kubectl delete job broken-job
kubectl create job fixed-job --image=busybox:1.36 -- sh -c "echo Fixed run; exit 0"
kubectl get jobs
```

## 5.14 CKA Practice Task

**Task**

1. Create a Job named `cka-job` that runs `busybox` and executes `echo done`, with `completions: 3` and `parallelism: 1`.
2. Verify all 3 completions succeed.
3. Create a CronJob named `cka-cron` with schedule `*/1 * * * *` running the same command, `concurrencyPolicy: Forbid`.
4. Manually trigger one run of `cka-cron` without waiting for the schedule.
5. Suspend `cka-cron`.

**Target Time**

5 minutes

Try it without looking at previous commands.

## 5.15 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

What's the difference in Pod behavior between `restartPolicy: Never` and `restartPolicy: OnFailure` on a failing Job?

**Question 2**

If `completions: 5` and `parallelism: 2`, how many Pods run at any given moment, and how many total Pods complete?

**Question 3**

What happens to a Job's Pods once `backoffLimit` is exceeded?

**Question 4**

What does `concurrencyPolicy: Forbid` do if a CronJob's previous run is still active when the next scheduled time arrives?

**Question 5**

Why can't you edit a Job's `spec.template` after creation, and what do you do instead?

**Question 6**

How would you run a CronJob's logic immediately, right now, without changing its schedule?

## 5.16 Useful Commands

```bash
kubectl get jobs
kubectl describe job <name>
kubectl delete job <name>

kubectl get cronjobs
kubectl get cj
kubectl describe cronjob <name>

kubectl create job <name> --image=<image> -- <command>
kubectl create job <name> --from=cronjob/<cronjob-name>

kubectl patch cronjob <name> -p '{"spec":{"suspend":true}}'
kubectl patch cronjob <name> -p '{"spec":{"suspend":false}}'

kubectl get pods -l job-name=<job-name>
kubectl logs -l job-name=<job-name>
```

## 5.17 Cleanup

```bash
kubectl delete job basic-job parallel-job failing-job broken-job fixed-job manual-run-1 cka-job
kubectl delete cronjob hello-cronjob cka-cron
```

## 5.18 Lab Checklist

- [ ] Created a basic Job and watched it reach completion
- [ ] Used `completions` and `parallelism` together
- [ ] Observed a Job retry and fail after exceeding `backoffLimit`
- [ ] Understood `restartPolicy: Never` vs `OnFailure` for Job Pods
- [ ] Created a CronJob with a schedule
- [ ] Understood `concurrencyPolicy`, history limits, and `suspend`
- [ ] Manually triggered a Job from a CronJob's template
- [ ] Suspended and resumed a CronJob
- [ ] Reproduced a Job stuck retrying on failure
- [ ] Diagnosed and replaced the broken Job
- [ ] Completed the CKA task within 5 minutes
