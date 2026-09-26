# Lab 05 — Conditionals, Loops & Named Templates

## Objectives
- Use `if`/`else`, `with`, and `range` correctly in templates
- Understand scope changes inside `with`/`range` and how to escape scope with `$`
- Write and use named templates (`define`/`template`/`include`) via `_helpers.tpl`
- Know when to use `include` vs `template`
- Practice debugging two common scope-related bugs

## Prerequisites
- Labs 01–04 completed
- Helm 3.x installed

---

## 1. if / else

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- else }}
# Ingress disabled
{{- end }}
```

Truthiness rules matter: empty string, `0`, `nil`, and `false` are all falsy. An empty map `{}` or empty list `[]` is also falsy in a boolean context.

## 2. with — Scoping Into a Value

`with` changes `.` (the current context) to the value you point it at — useful to avoid repeating a long path:

```yaml
{{- with .Values.service }}
type: {{ .type }}
port: {{ .port }}
{{- end }}
```
Inside the block, `.type` means `.Values.service.type`. **This is the #1 source of scope bugs** — see Troubleshooting Exercise 1.

`with` skips the block entirely if the value is empty/nil (like an implicit `if`).

## 3. range — Looping

```yaml
env:
{{- range .Values.envVars }}
  - name: {{ .name }}
    value: {{ .value | quote }}
{{- end }}
```

Ranging over a map instead of a list gives you `$key, $value`:
```yaml
{{- range $key, $value := .Values.labels }}
{{ $key }}: {{ $value | quote }}
{{- end }}
```

## 4. Escaping Scope with $

`$` always refers to the **root context** (the original top-level `.` passed into the template), no matter how deeply you're nested inside `with`/`range`. This is essential once you're inside a `range` or `with` but still need `.Release.Name` or `.Chart.Version`:

```yaml
{{- range .Values.services }}
  name: {{ .name }}-{{ $.Release.Name }}
{{- end }}
```
Without `$`, `.Release.Name` would fail inside the loop because `.` has been reassigned to the current service item, which has no `Release` field.

## 5. Named Templates: define / template / include

`_helpers.tpl` conventionally holds reusable named templates:

```yaml
{{- define "mychart.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}
```

Use it with either `template` or `include`:
```yaml
{{ template "mychart.labels" . }}
```
```yaml
{{- include "mychart.labels" . | nindent 4 }}
```

**`include` is almost always preferred over `template`** because `include` returns its output as a string that can be piped through functions like `nindent` — `template` cannot be piped at all (it writes directly to output). This is why every chart you'll encounter in the wild uses `include`, not `template`.

---

## 6. Troubleshooting Exercise 1 — with Silently Hides a Value

**Scenario:**
```yaml
{{- with .Values.service }}
type: {{ .type }}
release: {{ .Release.Name }}
{{- end }}
```
```
Error: nil pointer evaluating interface {}.Name
```

**Task:**
1. Recognize that inside `with .Values.service`, `.` is now `.Values.service` — there is no `.Release` at that scope anymore
2. Fix by using `$.Release.Name` instead of `.Release.Name`:
```yaml
{{- with .Values.service }}
type: {{ .type }}
release: {{ $.Release.Name }}
{{- end }}
```
3. Re-render with `helm template` and confirm it resolves correctly

## 7. Troubleshooting Exercise 2 — Named Template Output Not Indented Downstream

**Scenario:** A named template that returns multiple label lines is inserted at different indentation levels in two different manifests, and one of them breaks:

```yaml
# In deployment.yaml — works
metadata:
  labels:
    {{- include "mychart.labels" . | nindent 4 }}

# In service.yaml — someone copy-pasted without adjusting
spec:
  selector:
    {{- include "mychart.labels" . | nindent 4 }}
```
If `service.yaml`'s `selector:` is actually nested one level deeper (e.g. under a different parent key) than `deployment.yaml`'s `labels:`, the copy-pasted `nindent 4` produces mis-indented, invalid YAML there.

**Task:**
1. Run `helm template mychart` and diff the indentation level expected at each insertion point against what's produced
2. Understand that `nindent N` is **not chart-aware** — the number must be manually matched to wherever the `include` call sits in that specific file
3. Fix by adjusting the `nindent` value for the `service.yaml` insertion point to match its actual nesting level
4. Re-render and confirm both files produce valid YAML

**Takeaway:** Named templates make output reusable, but the *indentation* of that output is still the calling template's responsibility every single time — it's never inherited automatically.

---

## Cleanup
```bash
rm -rf mychart
```
