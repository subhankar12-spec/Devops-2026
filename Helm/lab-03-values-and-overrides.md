# Lab 03 — Values & Overrides

## Objectives
- Understand Helm's values precedence order end-to-end
- Work confidently with nested values and dot notation via `--set`
- Get a working intro to `values.schema.json` (deep dive comes in Phase 4)
- Practice debugging two common override mistakes

## Prerequisites
- Labs 01–02 completed
- Helm 3.x installed

---

## 1. Values Precedence Order

Helm merges values from multiple sources, highest precedence last (i.e., last one wins):

1. Chart's own `values.yaml` (the base defaults)
2. Parent chart's `values.yaml` (if this is a subchart — covered in Lab 07)
3. `-f`/`--values` file(s), **in the order given on the command line** — later files override earlier ones
4. `--set`
5. `--set-string`
6. `--set-json` / `--set-file`

Within `-f` files, the merge is a **deep merge** — an override file only needs to specify the keys it changes, not the whole values tree.

```bash
helm show values bitnami/nginx > default-values.yaml
```
```yaml
# custom.yaml
service:
  type: NodePort
```
```bash
helm install demo bitnami/nginx -f custom.yaml
helm get values demo
# Only shows service.type: NodePort — the rest is defaulted, not duplicated
```

## 2. Multiple -f Files and Merge Order

```bash
helm install demo bitnami/nginx -f base-values.yaml -f env-prod-values.yaml
```

`env-prod-values.yaml` wins on any overlapping key. This is the standard pattern for per-environment overrides (`values-dev.yaml`, `values-staging.yaml`, `values-prod.yaml`) layered on top of a shared base — you'll use this directly in the Lab 14 capstone.

## 3. --set and Nested Values

```bash
helm install demo bitnami/nginx --set service.type=NodePort
helm install demo bitnami/nginx --set image.tag=1.25.3,service.port=8080
```

Dot notation walks into nested maps. For arrays/lists:
```bash
helm install demo mychart --set ingress.hosts[0].host=example.com
```

`--set-string` forces the value to be treated as a string (important when a value looks numeric but must stay a string, e.g. a version tag `"1.20"` that would otherwise be parsed as the float `1.2`):
```bash
helm install demo mychart --set-string image.tag=1.20
```

`--set-json` lets you pass a JSON blob for complex nested structures in one shot:
```bash
helm install demo mychart --set-json 'resources={"limits":{"cpu":"500m","memory":"512Mi"}}'
```

## 4. values.schema.json — Quick Intro

A chart can ship a `values.schema.json` (JSON Schema) alongside `values.yaml`. Helm validates user-supplied values against it before rendering.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "replicaCount": { "type": "integer", "minimum": 1 }
  },
  "required": ["replicaCount"]
}
```
```bash
helm install demo mychart --set replicaCount=0
# Error: values don't meet the specifications of the schema(s)
```
Full schema design (required fields, enums, conditional schemas) is covered in depth in Lab 12.

---

## 5. Troubleshooting Exercise 1 — Silent Override Due to File Ordering

**Scenario:** You expect `replicaCount: 3` from `prod-values.yaml`, but the deployed release shows `replicaCount: 1`.

```bash
helm install demo mychart -f prod-values.yaml -f debug-values.yaml
```

**Task:**
1. Run `helm get values demo --all` and check the effective merged value
2. Inspect `debug-values.yaml` — it was left over from local testing and also sets `replicaCount: 1`
3. Recognize that the **last `-f` file wins**, regardless of which one "looks more important" by name
4. Fix by reordering the flags or removing the stray debug file, then re-run and verify with `helm get values demo --all`

## 6. Troubleshooting Exercise 2 — --set Array Behavior Surprise

**Scenario:** You want to add a second host to an ingress list:
```bash
helm upgrade demo mychart --set ingress.hosts[1].host=second.example.com
```
The upgrade succeeds, but `helm get values demo` shows `hosts[0]` is now **empty/null**, not the original first host.

**Task:**
1. Understand that `--set` does **not** merge arrays element-by-element against a base array in `values.yaml` the way it merges maps — assigning `hosts[1]` without `hosts[0]` can leave a hole or unexpected result depending on what's already set
2. Recognize the safer pattern for modifying array-heavy values: use a `-f` override file with the full array, rather than `--set` with index notation, for anything beyond a trivial single-element change
3. Fix by supplying a `custom-hosts.yaml` file with the complete `ingress.hosts` list and re-upgrading with `-f`

**Takeaway:** `--set` is convenient for scalar overrides and quick testing, but for lists/arrays of any complexity, a values file is safer and more auditable — which also matters for interview answers about "why we standardized on values files over --set in our pipelines."

---

## Cleanup
```bash
helm uninstall demo
rm -f default-values.yaml custom.yaml base-values.yaml env-prod-values.yaml prod-values.yaml debug-values.yaml custom-hosts.yaml
```
