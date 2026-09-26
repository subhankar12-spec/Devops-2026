# Lab 12 — Linting & Schema Validation

## Objectives
- Use `helm lint` and understand exactly what it does and doesn't catch
- Build a `values.schema.json` in depth: required fields, types, enums, conditional rules
- Wire `helm lint` + `helm template` into a CI pipeline
- Use third-party validators (`kubeconform`) against rendered output for a deeper check
- Practice debugging two common validation gaps

## Prerequisites
- Labs 01–11 completed
- Helm 3.x installed
- `kubeconform` installed for this lab (or `kubeval`, its predecessor)

---

## 1. helm lint — What It Actually Checks

```bash
helm lint mychart
```
`helm lint` checks:
- Chart structure is valid (required files present, `Chart.yaml` well-formed)
- Templates render without errors (essentially a `helm template` pass internally)
- Basic YAML syntax of the rendered output
- Some opinionated best-practice warnings (e.g. missing `app.kubernetes.io` labels)

**What it does NOT check:**
- Whether the rendered YAML is actually a *valid Kubernetes object* per the Kubernetes API schema (e.g. a typo'd field name, wrong type for a field, invalid enum value for something like `restartPolicy`)
- Whether the manifest would actually be *accepted* by a specific Kubernetes API server version

This gap is exactly why Lab 04's `helm install --dry-run --debug` (which validates against a live API server) catches things `helm lint` misses — and why this lab introduces `kubeconform` as a way to get that same class of validation **without needing a live cluster**, which matters for CI.

## 2. values.schema.json Deep Dive

Building on the intro from Lab 03:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["replicaCount", "image"],
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1,
      "maximum": 20
    },
    "image": {
      "type": "object",
      "required": ["repository", "tag"],
      "properties": {
        "repository": { "type": "string" },
        "tag": { "type": "string" },
        "pullPolicy": {
          "type": "string",
          "enum": ["Always", "IfNotPresent", "Never"]
        }
      }
    },
    "service": {
      "type": "object",
      "properties": {
        "type": {
          "type": "string",
          "enum": ["ClusterIP", "NodePort", "LoadBalancer"]
        },
        "port": { "type": "integer", "minimum": 1, "maximum": 65535 }
      }
    }
  }
}
```

Validation runs automatically on `helm install`/`upgrade`/`template` whenever `values.schema.json` is present in the chart root — no extra flag needed:
```bash
helm install demo mychart --set replicaCount=0
# Error: values don't meet the specifications of the schema(s) in the following chart(s):
# mychart:
# - replicaCount: Must be greater than or equal to 1
```

This is valuable specifically because it catches bad input **before** any templates even render — faster feedback than waiting for a Kubernetes API rejection.

## 3. CI Pipeline Integration

A solid Jenkins-relevant pipeline stage:
```bash
helm lint mychart
helm template mychart --set replicaCount=3 > rendered.yaml
kubeconform -summary -kubernetes-version 1.29.0 rendered.yaml
```
Three layers, each catching a different class of problem:
1. `helm lint` — chart structure and template rendering sanity
2. `values.schema.json` (implicit, runs during `helm template`) — bad input values
3. `kubeconform` against rendered output — actual Kubernetes API schema validity, without needing a live cluster or the slower `--dry-run` round-trip

---

## 4. Troubleshooting Exercise 1 — Lint Passes, kubectl apply Fails

**Scenario:**
```bash
helm lint mychart   # passes cleanly
kubectl apply -f <(helm template mychart)
```
```
error validating data: ValidationError(Deployment.spec.template.spec.containers[0]): unknown field "imagePullPolicy2"
```

**Task:**
1. Confirm `helm lint` truly reported no errors on this chart — it won't catch this
2. Run `kubeconform` against the rendered output and confirm it *does* catch the same typo'd field
3. Fix the typo (`imagePullPolicy2` → `imagePullPolicy`) in the template
4. Add `kubeconform` as a required CI step going forward so this class of error is caught before merge, not at deploy time

## 5. Troubleshooting Exercise 2 — Overly Strict Schema Blocks a Legitimate Override

**Scenario:** A schema enforces `"additionalProperties": false` at the top level to be strict. A user tries a perfectly reasonable override:
```bash
helm install demo mychart --set podAnnotations."vault\.hashicorp\.com/agent-inject"=true
```
```
Error: values don't meet the specifications of the schema(s): additional properties 'podAnnotations' not allowed
```

**Task:**
1. Inspect the schema and find that `podAnnotations` was never declared as a valid property at all, and `additionalProperties: false` rejects anything not explicitly listed
2. Recognize the tradeoff: `additionalProperties: false` is great for catching typos in known fields, but it also blocks legitimately open-ended fields like annotations/labels maps, which by nature accept arbitrary keys
3. Fix by declaring `podAnnotations` explicitly as an open object:
```json
"podAnnotations": {
  "type": "object",
  "additionalProperties": true
}
```
4. Re-run the install and confirm the override now succeeds

**Takeaway:** schema strictness is a judgment call per-field — lock down fields with a known fixed shape (enums, specific required keys), but leave genuinely open-ended fields (annotations, labels, arbitrary env var maps) permissive.

---

## Cleanup
```bash
rm -f rendered.yaml
helm uninstall demo -n <namespace> 2>/dev/null || true
```

---

**Phase 4 (Production Patterns) complete.** Next: Phase 5 — Troubleshooting & Capstone, starting with Lab 13 — Failed Releases & Recovery.
