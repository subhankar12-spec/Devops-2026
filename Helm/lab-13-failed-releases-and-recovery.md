# Lab 13 — Failed Releases & Recovery

## Objectives
- Understand every Helm release state and what causes each
- Use `helm rollback` deeply, including rolling back a rollback
- Recover a release stuck in `pending-upgrade`
- Use `--atomic` and `--cleanup-on-fail` to prevent stuck states in the first place
- Practice debugging two real recovery scenarios

## Prerequisites
- Labs 01–12 completed
- Helm 3.x installed

---

## 1. Release States

```bash
helm status demo -n <namespace>
```

| State | Meaning |
|---|---|
| `deployed` | Current, successfully applied release |
| `failed` | Install/upgrade completed its attempt but failed (e.g. a hook failed, or a resource was rejected) |
| `pending-install` | An install is in progress / was interrupted mid-flight |
| `pending-upgrade` | An upgrade is in progress / was interrupted mid-flight |
| `pending-rollback` | A rollback is in progress / was interrupted mid-flight |
| `superseded` | A previous revision, no longer active — normal, not an error state |
| `uninstalled` | Release was removed (only visible with `helm history --show-desc` if history retention allows it) |

The dangerous ones are the `pending-*` states — they mean Helm's own bookkeeping thinks an operation is still running, usually because the Helm client process was killed (CI timeout, network drop, manual Ctrl+C) mid-operation, not because Kubernetes itself failed anything.

## 2. helm rollback Deep Dive

```bash
helm rollback demo 3 -n <namespace>
```
Rolling back doesn't delete history — it creates a **new revision** whose manifest content matches the target revision (same append-only model from Lab 01).

**Rolling back a rollback:** if revision 5 was a rollback to revision 3's content, and you want to undo *that*, you don't "un-rollback" — you just roll back again to whatever revision had the state you actually want (e.g. `helm rollback demo 4` to return to what was active before the rollback):
```bash
helm history demo -n <namespace>
# REVISION  STATUS      DESCRIPTION
# 3         superseded  Upgrade complete
# 4         superseded  Upgrade complete
# 5         deployed    Rollback to 3
helm rollback demo 4 -n <namespace>
# Creates revision 6, content matching revision 4
```

## 3. Recovering a Stuck pending-upgrade

**This is a genuine last-resort procedure** — always try `helm rollback` to a known-good prior revision first, since it works even from a `pending-*` state in most cases:
```bash
helm rollback demo <last-good-revision> -n <namespace>
```

If rollback itself refuses to run (rare, but possible if the stored release Secret is in a genuinely inconsistent state), the manual fallback is editing the release Secret's status directly:
```bash
kubectl get secret -n <namespace> -l owner=helm,name=demo
# Find the Secret for the stuck revision, decode it, and inspect/patch its status field
```
This is invasive and should be a documented, rare, break-glass procedure — not a routine fix. Always prefer `helm rollback` first.

## 4. --atomic and --cleanup-on-fail

Both flags exist specifically to prevent ever reaching a stuck state in the first place:

```bash
helm upgrade demo mychart -n <namespace> --atomic --cleanup-on-fail
```
- `--atomic`: if the upgrade fails, Helm automatically rolls back to the previous successful revision — you never end up sitting in a `failed` state waiting for manual intervention
- `--cleanup-on-fail`: on a failed operation, deletes any new resources that were already created before the failure, rather than leaving orphaned partial resources behind

**Recommendation for any production pipeline (directly relevant to your Jenkins-based deploys): always use `--atomic` on `helm upgrade`.** It converts "silent partial failure requiring manual cleanup" into "automatic safe rollback," which is a meaningfully different operational posture.

---

## 5. Troubleshooting Exercise 1 — Stuck pending-upgrade Blocks All Future Upgrades

**Scenario:** A Jenkins job running `helm upgrade` gets killed by a pipeline timeout mid-deploy. The next pipeline run fails immediately:
```
Error: UPGRADE FAILED: another operation (install/upgrade/rollback) is in progress
```

**Task:**
1. Run `helm status demo -n <namespace>` and confirm it shows `pending-upgrade`
2. Run `helm history demo -n <namespace>` to identify the last revision that was actually `deployed` successfully
3. Recover with `helm rollback demo <last-good-revision> -n <namespace>`
4. Confirm `helm status demo` now shows `deployed` again and the next upgrade attempt proceeds normally
5. Going forward, add `--atomic` to the pipeline's `helm upgrade` invocation and increase the pipeline timeout to comfortably exceed the slowest expected deploy, so this class of interruption stops happening

## 6. Troubleshooting Exercise 2 — Rollback Target Drift

**Scenario:** You roll back to revision 3, but the running Pods immediately crash — revision 3's manifest references a ConfigMap key that a *later* manual `kubectl edit` change removed from the live cluster (someone edited a resource directly, bypassing Helm).

**Task:**
1. Run `helm get manifest demo --revision 3 -n <namespace>` and compare it against `kubectl get configmap <name> -o yaml` in the live cluster
2. Identify the missing key that revision 3 expects but the live ConfigMap no longer has
3. Recognize the root cause: manual `kubectl` edits that bypass Helm create drift between what Helm *thinks* a past revision looked like (which is fixed and historical) and what's *actually* live now (which can be changed by anyone with cluster access) — a rollback assumes the rest of the cluster state hasn't moved out from under it
4. Fix immediately by restoring the missing ConfigMap key manually, or by re-applying via Helm rather than `kubectl edit` going forward
5. Document as a policy: all changes to Helm-managed resources should go through Helm (values/chart changes), never direct `kubectl edit`/`patch` — direct edits are exactly what makes rollback assumptions unreliable

**Takeaway:** `helm rollback` restores what Helm applied — it has no way to know about or restore state that was changed outside of Helm's own control.

---

## Cleanup
```bash
helm uninstall demo -n <namespace>
```

---

Next: Lab 14 — Capstone: Multi-Chart Application, the final lab.
