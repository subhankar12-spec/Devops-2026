# Lab 02 — Chart Anatomy

## Objectives
- Understand every file/folder in a Helm chart and its purpose
- Know the difference between `helm create` scaffolding and a hand-rolled chart
- Understand chart versioning rules (`version` vs `appVersion`)
- Practice debugging two common packaging/versioning mistakes

## Prerequisites
- Lab 01 completed
- Helm 3.x installed

---

## 1. Scaffolding a Chart

```bash
helm create mychart
```

This generates:
```
mychart/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default configuration values
├── charts/             # Subcharts / dependencies live here
├── templates/          # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── serviceaccount.yaml
│   ├── hpa.yaml
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   └── tests/
│       └── test-connection.yaml
└── .helmignore
```

Walk through each generated template file and identify which `values.yaml` keys feed into it. This mapping is the single most useful mental model for reading any unfamiliar chart later.

## 2. Chart.yaml Deep Dive

```yaml
apiVersion: v2
name: mychart
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "1.16.0"
```

| Field | Meaning |
|---|---|
| `apiVersion` | `v2` for Helm 3 charts (v1 is legacy, no subchart `dependencies` block support) |
| `type` | `application` (deployable) or `library` (shared templates only, not deployable) |
| `version` | **The chart's own version** — bump this whenever the chart (templates/values structure) changes |
| `appVersion` | **The version of the application/image the chart deploys** — informational only, doesn't affect Helm's behavior |

**This distinction is a very common interview trip-up.** `version` follows SemVer strictly and is what `helm search`/`helm repo index` key off of. `appVersion` is just a label — Helm doesn't validate or use it for anything except display and the default image tag in some chart templates.

## 3. values.yaml and .helmignore

- `values.yaml` — the default configuration. Anything referenced in templates via `.Values.x` should have a sane default here.
- `.helmignore` — same idea as `.gitignore`; controls what gets excluded when the chart is packaged with `helm package` (keeps `.git/`, editor swap files, etc. out of the `.tgz`).

## 4. _helpers.tpl and NOTES.txt

- `_helpers.tpl` — holds named templates (functions), prefixed with `_` so Helm knows not to render it as a standalone manifest.
- `NOTES.txt` — rendered and printed to the terminal after a successful `install`/`upgrade`. Supports the same templating as any other file — commonly used to print a `kubectl port-forward` command or the app's URL.

---

## 5. Troubleshooting Exercise 1 — Malformed Chart.yaml

**Scenario:** A teammate hand-edited `Chart.yaml` and packaging now fails:

```bash
helm package mychart
```
```
Error: validation: chart.metadata.version "1.0" is invalid
```

**Task:**
1. Open `Chart.yaml` and find the offending `version` field
2. Understand why `1.0` fails but `1.0.0` doesn't (Helm enforces strict SemVer — major.minor.patch)
3. Fix it and re-run `helm package mychart`

## 6. Troubleshooting Exercise 2 — Confusing version and appVersion

**Scenario:** Someone bumped `appVersion` to `2.0.0` to "release the new version" but didn't touch `version`. They push it to the chart repo and re-run `helm repo index`.

```bash
helm upgrade myapp ./mychart
```
The upgrade appears to succeed, but consumers pulling from the repo still get the **old chart** because the repo index keys releases by `version`, not `appVersion` — and since `version` never changed, tooling that checks "is there a new chart available" sees no change.

**Task:**
1. Identify that `version` must be bumped for the chart repo/index to recognize a new publishable artifact
2. Bump `version` (e.g. `0.1.0` → `0.2.0`), keep `appVersion` at `2.0.0`
3. Re-run `helm package` + `helm repo index --merge index.yaml .` and confirm the new version is now discoverable

**Takeaway:** `appVersion` communicates "what's inside," `version` communicates "is this artifact new." Bumping only one of them independently is valid and common — but forgetting to bump `version` when you need consumers to pick up a change is a frequent real-world mistake.

---

## Cleanup
```bash
rm -rf mychart *.tgz index.yaml
```
