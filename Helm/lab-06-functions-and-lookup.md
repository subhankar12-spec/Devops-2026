# Lab 06 — Functions & Lookup

## Objectives
- Get comfortable with the Sprig function library as used in real charts
- Use `tpl` to render a string value as a template
- Use `lookup` to query live cluster state from within a template
- Understand how `lookup` behaves differently across `helm template`, `--dry-run`, and a real `install`
- Practice debugging two common function/lookup mistakes

## Prerequisites
- Labs 01–05 completed
- Helm 3.x installed

---

## 1. Sprig Function Library

Helm bundles [Sprig](http://masterminds.github.io/sprig/) — a large library of template functions beyond Go's built-ins. You won't memorize all ~100+, but these show up constantly in real charts:

**Strings:** `trim`, `trunc 63`, `replace "-" "_"`, `contains`, `hasPrefix`, `hasSuffix`, `printf`
**Lists:** `first`, `last`, `join ","`, `uniq`, `sortAlpha`
**Dicts:** `dict`, `merge`, `keys`, `hasKey`
**Math:** `add`, `sub`, `mul`, `div`, `max`, `min`
**Dates:** `now`, `date "2006-01-02"`
**Encoding:** `b64enc`, `b64dec`, `sha256sum`

Real-world example — generating a DNS-safe, length-limited resource name:
```yaml
name: {{ printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" }}
```
This pattern (`trunc 63 | trimSuffix "-"`) appears in almost every `helm create` scaffold's `_helpers.tpl` because Kubernetes object names are limited to 63 characters (DNS label limit).

## 2. The tpl Function

`tpl` renders a **string value** as if it were a template, with a given context. This is how charts let users pass templated strings through `values.yaml`:

```yaml
# values.yaml
podAnnotations:
  configHash: "{{ .Release.Revision }}"
```
```yaml
# template
annotations:
  {{- tpl (toYaml .Values.podAnnotations) . | nindent 4 }}
```
Without `tpl`, the `{{ .Release.Revision }}` string in `values.yaml` would be inserted literally as text — `tpl` is what makes it actually evaluate.

## 3. The lookup Function

`lookup` queries the live Kubernetes API from inside a template:
```yaml
{{- $existingSecret := lookup "v1" "Secret" .Release.Namespace "my-secret" }}
{{- if $existingSecret }}
# Secret already exists — reuse it, don't regenerate
{{- else }}
# Generate a new one
{{- end }}
```
Signature: `lookup apiVersion, kind, namespace, name` (empty string for namespace/name = list all in scope).

**Critical behavior:** `lookup` only works when Helm has an actual connection to a live cluster.
- `helm install`/`upgrade` (real run): works normally
- `helm template`: **`lookup` always returns an empty result** (no cluster context available) — this can silently produce different output than a real install
- `helm install --dry-run`: as of Helm 3.13+, `lookup` **can** query the live cluster during a client-side dry run, unlike `helm template`

This distinction is a common interview and real-world gotcha — see Troubleshooting Exercise 1.

---

## 4. Troubleshooting Exercise 1 — lookup Fails Silently in CI

**Scenario:** A chart uses `lookup` to check for an existing auto-generated password Secret and reuse it across upgrades (a common pattern to avoid rotating passwords on every deploy). A CI pipeline runs `helm template` to validate charts before merging, and everything looks fine there — but on the actual `helm upgrade` against the cluster, a teammate reports the password keeps getting regenerated every deploy in a *different* pipeline that only ever uses `helm template` to render manifests it then applies via a separate `kubectl apply` step (not `helm install`/`upgrade`).

**Task:**
1. Run `helm template mychart` and confirm `lookup` returns empty/nil in this mode, regardless of what actually exists in the cluster
2. Understand why: `helm template` never contacts the API server, so `lookup` has nothing to query
3. Recognize the architectural problem: any pipeline that renders with `helm template` and applies separately loses all `lookup`-based logic — this only works correctly through `helm install`/`upgrade` (or dry-run in newer Helm versions)
4. Document this as a hard constraint when advising a team on `helm template` + `kubectl apply` vs. native `helm upgrade` workflows

## 5. Troubleshooting Exercise 2 — Sprig Function Typo Fails Silently

**Scenario:**
```yaml
name: {{ .Values.name | trimSuffi "-" }}
```
```
Error: function "trimSuffi" not defined
```
This one isn't silent — it errors clearly. But a subtler version is:
```yaml
replicas: {{ .Values.replicaCount | default 1 }}
```
where `.Values.replicaCount` is actually set to the string `"0"` (not the integer `0`, not empty) — `default` only falls back on empty/nil/zero-value inputs, and depending on how the value was set (e.g. via `--set-string replicaCount=0`), the behavior may not be what's expected.

**Task:**
1. Reproduce both: the typo'd function name and the string-vs-int `"0"` case
2. For the typo: fix `trimSuffi` → `trimSuffix`
3. For the `"0"` case: run `helm template mychart --set-string replicaCount=0` and `--set replicaCount=0` and compare rendered output to confirm how type coercion affects `default`'s behavior
4. Takeaway: always sanity-check numeric-looking values with `helm template` when there's any ambiguity about string vs. int handling, especially combined with `default`

---

## Cleanup
```bash
rm -rf mychart
```

---

**Phase 2 (Templating) complete.** Next: Phase 3 — Packaging & Distribution, starting with Lab 07 — Dependencies & Subcharts.
