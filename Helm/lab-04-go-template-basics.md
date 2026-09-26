# Lab 04 — Go Template Basics

## Objectives
- Understand the core built-in objects available in Helm templates
- Use pipelines and common template functions
- Control whitespace correctly in rendered YAML
- Debug templates with `helm template` and `helm install --dry-run`
- Practice debugging two common rendering failures

## Prerequisites
- Labs 01–03 completed
- Helm 3.x installed

---

## 1. Core Built-In Objects

Every template has access to these objects:

| Object | Contains |
|---|---|
| `.Values` | Merged values (defaults + overrides) |
| `.Release` | `.Release.Name`, `.Release.Namespace`, `.Release.IsInstall`, `.Release.IsUpgrade`, `.Release.Revision` |
| `.Chart` | Contents of `Chart.yaml` — `.Chart.Name`, `.Chart.Version`, `.Chart.AppVersion` |
| `.Files` | Access to non-template files in the chart (e.g. reading a config file to embed as a ConfigMap) |
| `.Capabilities` | Info about the target cluster — `.Capabilities.KubeVersion`, `.Capabilities.APIVersions.Has "batch/v1"` |

Example:
```yaml
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
  labels:
    app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
```

## 2. Pipelines and Functions

Go templates use `|` to pipe a value through functions, left to right:

```yaml
name: {{ .Values.name | default "app" | upper | quote }}
```
Reads as: take `.Values.name`, fall back to `"app"` if empty, uppercase it, then quote it.

Common functions you'll see constantly in real charts:
- `default "value"` — fallback if the piped value is empty/nil
- `quote` / `squote` — wrap in double/single quotes
- `upper` / `lower` / `trim`
- `indent N` / `nindent N` — indent a block by N spaces (`nindent` also adds a leading newline — critical for inserting multi-line blocks correctly)
- `toYaml` — convert a map/list value into YAML (used constantly for `resources:`, `nodeSelector:`, `tolerations:`)

```yaml
resources:
{{ toYaml .Values.resources | nindent 2 }}
```

## 3. Whitespace Control

`{{-` trims whitespace/newline **before** the action; `-}}` trims **after**.

```yaml
{{- if .Values.enabled }}
enabled: true
{{- end }}
```
Without the `-`, `if`/`end` lines leave behind blank lines in the rendered output — usually harmless, but can break strict YAML parsers or produce ugly diffs. Get in the habit of using `{{-`/`-}}` on control structure lines by default.

## 4. Debugging Templates

```bash
# Render locally without touching the cluster
helm template mychart

# Render exactly as Helm would install it (includes hooks, does a dry submission to API server for validation)
helm install demo mychart --dry-run --debug
```

`helm template` is faster and cluster-independent — use it first. `--dry-run --debug` on `helm install`/`upgrade` also validates against the live API server's OpenAPI schema, which `helm template` does not, so it catches things like invalid field names that `helm template` would silently render.

---

## 5. Troubleshooting Exercise 1 — Whitespace Breaks YAML Indentation

**Scenario:** A named template used inside `_helpers.tpl` renders extra content that shifts everything below it in the manifest, and `kubectl apply` fails with a YAML parse error.

```yaml
metadata:
  labels:
    {{ include "mychart.labels" . }}
```

**Task:**
1. Run `helm template mychart` and inspect the raw output around `labels:`
2. Notice the `include` output isn't indented to match its surrounding context — `include` returns exactly what the named template produced, with no awareness of where it's being inserted
3. Fix using `nindent`:
```yaml
metadata:
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
```
4. Re-render and confirm valid YAML

## 6. Troubleshooting Exercise 2 — Nil Value Crashes Rendering

**Scenario:**
```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```
```bash
helm template mychart --set image=nginx
```
```
Error: template: mychart/templates/deployment.yaml:12:24: executing... nil pointer evaluating interface {}.repository
```

**Task:**
1. Recognize that `--set image=nginx` overwrote the entire `image` map with a plain string, so `.Values.image.repository` is now trying to look up a field on a string, not a map
2. Understand the fix is either correcting the `--set` syntax (`--set image.repository=nginx`) or making the template defensive
3. Make the template resilient using `default`:
```yaml
image: "{{ .Values.image.repository | default "nginx" }}:{{ .Values.image.tag | default "latest" }}"
```
4. Re-render both the broken `--set` and the corrected `--set` and compare output

**Takeaway:** Go templates fail loudly (and sometimes unhelpfully) on nil/type mismatches. Defensive use of `default` and validating input shape via `values.schema.json` (Lab 12) both address this — from different angles.

---

## Cleanup
```bash
rm -rf mychart
```
