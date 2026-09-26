# Helm Hands-On Lab Curriculum
**Target audience:** DevOps engineers with ~5 years of experience, comfortable with Kubernetes fundamentals, looking to go deep on Helm for production use and interviews.

**Format:** Each lab is self-contained, GitHub-ready Markdown with inline YAML, a hands-on exercise, and a break/fix troubleshooting section. Labs progress from fundamentals to production-grade patterns, ending in a capstone.

**Prerequisites:** Working `kubectl` access to a cluster (minikube/kind/EKS all fine), Helm 3.x installed, basic Kubernetes object familiarity (Deployments, Services, ConfigMaps, Secrets).

---

## Phase 1 — Helm Fundamentals

### Lab 01 — Helm Architecture & CLI ✅ *(done)*
- Helm 3 client-only architecture (no Tiller)
- Release state stored as a Kubernetes Secret
- `install` / `upgrade` / `rollback` / `history` append-only behavior
- `helm get manifest` / `helm get values` inspection
- **Troubleshooting:** release-name collision, stale repo index

### Lab 02 — Chart Anatomy
- `Chart.yaml`, `values.yaml`, `templates/`, `charts/`, `_helpers.tpl`, `NOTES.txt`
- `helm create` walkthrough vs. hand-rolling a chart
- Chart versioning (`version` vs `appVersion`) and semver rules
- `.helmignore`
- **Troubleshooting:** chart fails to package due to malformed `Chart.yaml`; app breaks after a version bump because `appVersion` was confused with `version`

### Lab 03 — Values & Overrides
- Values precedence order: `values.yaml` → `-f custom.yaml` → `--set` → `--set-string`/`--set-json`
- Nested values and dot notation with `--set`
- `values.schema.json` intro (deferred deep dive to Phase 4)
- Multiple `-f` files and merge behavior
- **Troubleshooting:** a value silently gets overridden due to file ordering; a `--set` array value doesn't behave as expected

---

## Phase 2 — Templating

### Lab 04 — Go Template Basics
- `{{ .Values }}`, `{{ .Release }}`, `{{ .Chart }}`, `{{ .Files }}`, `{{ .Capabilities }}` objects
- Pipelines and basic functions (`quote`, `default`, `upper`, `indent`)
- Whitespace control (`{{-` / `-}}`)
- `helm template` vs `helm install --dry-run` for debugging
- **Troubleshooting:** YAML indentation broken by template whitespace; a nil value crashes rendering

### Lab 05 — Conditionals, Loops & Named Templates
- `if/else`, `with`, `range`
- Scope changes inside `with`/`range` and using `$` to escape scope
- Named templates (`define`/`template`/`include`) and `_helpers.tpl` conventions
- `include` vs `template` (why `include` is preferred for piping output)
- **Troubleshooting:** a `with` block silently hides a value because scope changed; a named template output isn't indented correctly downstream

### Lab 06 — Functions & Lookup
- Sprig function library overview (string, list, dict, math, date functions used in real charts)
- `tpl` function for rendering strings as templates
- `lookup` function to query live cluster state from within a template
- Using `lookup` safely (behavior during `helm template`/`--dry-run` vs `helm install`)
- **Troubleshooting:** a chart that relies on `lookup` fails in a CI dry-run pipeline; a Sprig function typo fails silently

---

## Phase 3 — Packaging & Distribution

### Lab 07 — Dependencies & Subcharts
- `Chart.yaml` `dependencies` block, `helm dependency update`/`build`
- Parent-child values propagation, `global` values
- Conditions and tags to toggle subcharts on/off
- Alias-ing a dependency for multiple instances of the same subchart
- **Troubleshooting:** subchart values not being picked up due to missing parent key namespace; `Chart.lock` drift between environments

### Lab 08 — Packaging & Repositories
- `helm package`, `helm repo index`, hosting a chart repo (e.g., via a static site or artifact store)
- `helm push` to a traditional chart museum-style repo
- Signing charts with `helm package --sign` and provenance files (`.prov`)
- **Troubleshooting:** repo index goes stale after publishing a new chart version; signature verification failure

### Lab 09 — OCI Registries
- Pushing/pulling charts as OCI artifacts (`helm push`/`helm pull` with `oci://`)
- Using ECR as an OCI-compliant Helm registry (directly relevant to your AWS stack)
- Authentication flow (`helm registry login`) and tagging conventions
- **Troubleshooting:** OCI push fails due to repository not pre-created in ECR; version tag collision

---

## Phase 4 — Production Patterns

### Lab 10 — Hooks
- Hook types: `pre-install`, `post-install`, `pre-upgrade`, `post-upgrade`, `pre-delete`, `post-delete`, `pre-rollback`, `post-rollback`
- Hook weights and deletion policies (`before-hook-creation`, `hook-succeeded`, `hook-failed`)
- Common real-world use: DB migration Jobs as hooks
- **Troubleshooting:** a failed hook blocks all future releases because its Job wasn't cleaned up; hook execution order bug from incorrect weights

### Lab 11 — Helm Test
- Writing `helm.sh/hook: test` Pods
- `helm test` execution and exit-code semantics
- Structuring tests for a multi-service chart
- **Troubleshooting:** test Pod hangs because it lacks a restart policy; false-positive pass due to a misconfigured test container

### Lab 12 — Linting & Schema Validation
- `helm lint` — what it catches and what it doesn't
- `values.schema.json` deep dive: enforcing required fields, types, enums
- Integrating `helm lint` + `helm template` into a CI pipeline (Jenkins-relevant)
- Third-party tools: `kubeval`/`kubeconform` against rendered output
- **Troubleshooting:** `helm lint` passes but `kubectl apply` fails on rendered output; schema validation blocks a legitimate override due to an overly strict schema

---

## Phase 5 — Troubleshooting & Capstone

### Lab 13 — Failed Releases & Recovery
- Understanding release states: `deployed`, `failed`, `pending-install`, `pending-upgrade`, `superseded`
- `helm rollback` deep dive, including rollback of a rollback
- Recovering from a stuck `pending-upgrade` state (manual Secret intervention as last resort)
- `--atomic` and `--cleanup-on-fail` flags
- **Troubleshooting:** a stuck release blocks all future upgrades; a rollback target no longer matches live cluster state (drift)

### Lab 14 — Capstone: Multi-Chart Application
- Umbrella chart tying together 3+ subcharts (e.g., frontend, backend, database) mirroring a realistic microservice layout
- Environment-specific values files (`values-dev.yaml`, `values-staging.yaml`, `values-prod.yaml`)
- End-to-end promotion flow: package → push to ECR (OCI) → deploy via ArgoCD (bridges into your ArgoCD curriculum)
- Full lifecycle exercise: install → upgrade with a breaking change → detect failure via `helm test` → rollback
- **Capstone troubleshooting:** multi-part scenario combining a subchart values bug, a failed hook, and a bad rollback target — diagnose and fix all three

---

## Notes
- Each lab should be deliverable as its own `.md` file (`lab-01-helm-architecture-cli.md`, etc.) for a GitHub repo.
- Difficulty ramps from CLI/conceptual (Phase 1) → templating logic (Phase 2) → distribution/ops (Phase 3) → production hardening (Phase 4) → integration and recovery under pressure (Phase 5).
- Lab 14 is designed to hand off directly into the ArgoCD curriculum (GitOps deployment of the same umbrella chart).
