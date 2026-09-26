# Helm — Complete Interview Prep & Command Cheatsheet (5 YOE Level)

Part 1 covers concepts you should be able to explain out loud. Part 2 is the command reference.

---

# PART 1 — Concepts

## 1. What Helm Is, In One Breath

Helm is the package manager for Kubernetes: it templates raw YAML manifests using Go templating,
bundles them into a versioned unit called a **chart**, and tracks each install as a **release**
with revision history, so deployments become repeatable, parameterized, and rollback-able instead
of hand-edited `kubectl apply` files.

## 2. Helm 2 vs Helm 3 (classic interview question)

| | Helm 2 | Helm 3 |
|---|---|---|
| Server component | **Tiller** (in-cluster, big security concern — cluster-admin by default) | No Tiller — client talks directly to K8s API using your kubeconfig/RBAC |
| Release storage | ConfigMaps in `kube-system` (Tiller's namespace) | Secrets, stored **in the release's own namespace** |
| Security model | Tiller = shared attack surface | Uses your own RBAC — much better multi-tenant story |
| CRD handling | Installed via hooks | Dedicated `crds/` directory, installed once, not upgraded automatically |
| 3-way merge | No (2-way: chart vs live) | Yes — chart, live state, and previous release all diffed, so manual `kubectl edit` drift is respected/reconciled better |
| `helm template` | Less central | First-class, used heavily for CI validation and by ArgoCD internally |

**Why it matters:** if asked "why was Tiller removed," the answer is security — Tiller ran with
broad in-cluster privileges, so anyone who could talk to Tiller could effectively act as
cluster-admin.

## 3. Chart Anatomy

```
mychart/
├── Chart.yaml          # Metadata: name, version, appVersion, dependencies
├── values.yaml          # Default configuration values
├── charts/              # Subcharts / vendored dependencies
├── templates/           # Go-templated K8s manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── _helpers.tpl     # Reusable named templates (helper functions)
│   ├── NOTES.txt        # Post-install message shown to user
│   └── tests/           # helm test hooks
├── crds/                # CRDs — installed once, never upgraded by helm
└── .helmignore          # Files to exclude when packaging
```

Key distinction to know: **`Chart.yaml` `version`** = chart's own version (semver, bump on any
chart change). **`appVersion`** = version of the application it deploys. They're independent and
people conflate them in interviews.

## 4. Templating Engine

- Helm templates are Go templates (`text/template`) plus the **Sprig** function library
  (string, list, math, date helpers) plus a few Helm-specific functions (`include`, `required`,
  `lookup`, `tpl`).
- `{{ .Values.foo }}` — pull from values.yaml (or `-f`/`--set` overrides).
- `{{ .Release.Name }}`, `{{ .Release.Namespace }}` — release-scoped built-ins.
- `{{ .Chart.Name }}`, `{{ .Chart.Version }}` — chart metadata.
- `{{- include "mychart.labels" . | nindent 4 }}` — call a named template from `_helpers.tpl`
  and control whitespace (`-` trims, `nindent` re-indents). Whitespace control trips people up
  constantly — worth being fluent in `-` placement.
- `{{ required "X is required" .Values.X }}` — fail fast if a value is missing, instead of
  deploying broken manifests.
- `lookup` function — can query live cluster state from within a template (e.g. check if a
  Secret already exists) — one of the few ways templates can be "live-aware."

## 5. Values Precedence (always comes up)

Lowest → highest priority:
1. Chart's own `values.yaml` defaults
2. Parent chart's values (when this chart is a subchart/dependency)
3. `-f/--values` files, **in the order passed** (later file wins on conflict)
4. `--set`, `--set-string`, `--set-file` (highest — always wins)

## 6. Release Lifecycle & Hooks

Hooks let you run jobs at specific points: `pre-install`, `post-install`, `pre-upgrade`,
`post-upgrade`, `pre-delete`, `post-delete`, `pre-rollback`, `post-rollback`, `test`.

```yaml
metadata:
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "1"          # Lower runs first, controls ordering among hooks
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

Common real use: a `pre-upgrade` Job that runs a DB migration before the new app pods roll out,
or a `post-install` Job that seeds initial data.

`helm test` runs any template annotated `helm.sh/hook: test` — used for post-deploy smoke tests
in CI pipelines.

## 7. Dependencies / Subcharts

- Declared in `Chart.yaml` under `dependencies:` (name, version, repository).
- `helm dependency update` pulls them into `charts/` and writes `Chart.lock` (pin exact versions
  — commit this for reproducible builds, same idea as a lockfile in npm/pip).
- Subchart values are namespaced under the subchart's name in the parent's `values.yaml`:
  ```yaml
  postgresql:
    auth:
      password: "..."
  ```
- `condition:` and `tags:` in `Chart.yaml` dependencies let you conditionally enable/disable
  subcharts (e.g. toggle a bundled Redis on/off per environment).

## 8. Chart Distribution

- **Classic repo**: an index.yaml + packaged `.tgz` charts served over HTTP (ChartMuseum, Artifact
  Hub, Nexus/Artifactory).
- **OCI registries** (current standard, Helm 3.8+): charts pushed/pulled like container images —
  `helm push`, `helm pull oci://...`. Same registry can host both your app images and its chart —
  simplifies auth and tooling (ECR, GHCR, Harbor all support this now).

## 9. Helm vs Kustomize (frequently asked to compare)

| | Helm | Kustomize |
|---|---|---|
| Approach | Templating (text substitution before it's even YAML) | Overlays / patches on valid YAML (structural) |
| Packaging | Versioned chart, distributable artifact | No packaging concept — just directories of YAML + overlays |
| Logic | Full templating language (conditionals, loops, functions) | Declarative patches only, no logic |
| Built into kubectl | No (separate binary) | Yes (`kubectl apply -k`) |
| Best fit | Distributing reusable apps to many consumers/environments | In-house apps with a handful of environment variants |

Many shops use both: Helm to package/distribute, Kustomize (or `helm template | kustomize`) to
apply last-mile environment patches without forking the chart.

## 10. Production Patterns (things that trip up experienced engineers)

**`checksum/config` annotation** — the single most common Helm pattern in real production use.
K8s won't restart pods when a mounted ConfigMap/Secret changes, since the Deployment's pod spec
itself is unchanged. Fix: hash the config file into a pod annotation so the spec *does* change
on config drift, forcing a rolling restart.

```yaml
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

**`.Values.global`** — values under `global:` are visible to the parent chart *and every subchart*,
without needing to be namespaced per-subchart. This is how you pass shared config (image registry,
environment name, common labels) down through subcharts in one place instead of repeating it in
each subchart's values block.

```yaml
global:
  imageRegistry: myregistry.io
  environment: prod
```

**`values.schema.json`** — a JSON Schema file at the chart root that validates `values.yaml` at
`lint`/`template`/`install`/`upgrade` time. Catches malformed config (wrong type, missing required
field) before a broken manifest ever reaches the cluster, instead of failing deep inside a template.

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["image", "service"],
  "properties": {
    "replicaCount": { "type": "integer", "minimum": 1, "maximum": 50 },
    "image": {
      "type": "object",
      "required": ["repository"],
      "properties": { "repository": { "type": "string", "minLength": 1 } }
    }
  }
}
```

**`.Capabilities`** — lets a template branch on what the target cluster actually supports, so one
chart can work across clusters running different K8s versions or with different CRDs installed.

```yaml
{{- if .Capabilities.APIVersions.Has "batch/v1" }}
apiVersion: batch/v1
{{- else }}
apiVersion: batch/v1beta1
{{- end }}
# .Capabilities.KubeVersion.Version is also available for version-gated logic
```

**Library charts** (`type: library` in `Chart.yaml`) — a chart that produces *no* K8s resources of
its own; it exists only to hold reusable named templates (`_helpers.tpl`-style logic) that other
charts `include`. Prevents duplicating the same labels/selectors/annotations boilerplate across
every microservice's chart when you're running 10-20+ services off similar templates.

```yaml
# Chart.yaml
apiVersion: v2
name: mylib
type: library
version: 0.1.0
```

**Post-renderer / Helm+Kustomize wiring** — the actual mechanism behind "many shops use both"
(mentioned above): either pipe `helm template` output into `kustomize build -`, or use Helm's
native `--post-renderer` flag to run any executable (often a Kustomize wrapper script) against
the rendered manifests before they're applied — lets you patch a vendored chart's output without
forking it.

```bash
helm template <chart> | kustomize build - -o rendered.yaml     # manual pipe approach
helm install <release> <chart> --post-renderer ./kustomize-wrapper.sh   # native flag
```

## 11. CRDs — the gotcha everyone forgets

Helm **installs** CRDs found in a chart's `crds/` directory on first install, but will **not**
upgrade or delete them on `helm upgrade`/`helm uninstall` — this is intentional (CRDs are cluster-
wide and risky to auto-mutate). If a chart bumps a CRD's schema, you must `kubectl apply` the new
CRD yourself before upgrading the release.

## 12. Security Considerations

- Never bake secrets into `values.yaml` committed to Git — use `helm-secrets` (SOPS-encrypted
  values), External Secrets Operator, or inject via AWS Secrets Manager/Vault at deploy time.
- `--set` values can leak into shell history / CI logs — prefer `-f` with a file sourced from a
  secrets manager in pipelines.
- Verify chart provenance for third-party charts: `helm verify`, checking `.prov` signature files
  when charts are signed.
- RBAC: since Helm 3 has no Tiller, the identity actually applying changes is whatever
  ServiceAccount/kubeconfig your CI runner uses — scope that tightly.

## 13. Common Interview Questions (rapid recall)

- **"How does Helm decide what changed between revisions?"** — 3-way merge: compares the chart's
  new manifest, the last applied release manifest, and the live cluster state, so manual
  `kubectl edit` changes aren't silently clobbered.
- **"What happens if `helm upgrade` fails halfway?"** — Release marked `failed`; use `--atomic`
  to have Helm auto-rollback to the last good revision on failure, or `helm rollback` manually.
- **"How do you debug a template rendering issue without touching the cluster?"** —
  `helm template <chart> --debug` or `helm install --dry-run --debug`.
- **"How do you manage the same chart across dev/stage/prod?"** — one chart, environment-specific
  values files (`values-dev.yaml`, `values-prod.yaml`), same chart version promoted forward.
- **"Why would `helm upgrade --install` be preferred in a CI pipeline?"** — idempotent: works
  whether the release exists yet or not, so first deploy and 100th deploy use identical pipeline
  code.
- **"How does ArgoCD use Helm?"** — internally runs `helm template` to render manifests, then
  applies/diffs them like any other manifest source — Helm's release/rollback machinery is
  bypassed in favor of Argo's own sync history.

---

# PART 2 — Command Reference

## Repo Management

```bash
helm repo add <name> <url>              # Add a chart repo
helm repo update                        # Refresh local cache of all repos
helm repo list                          # List added repos
helm repo remove <name>                 # Remove a repo
helm search repo <keyword>              # Search charts in added repos
helm search hub <keyword>               # Search Artifact Hub
```

## Installing / Uninstalling Releases

```bash
helm install <release-name> <chart> -n <namespace>              # Install chart
helm install <release-name> <chart> -f values.yaml -n <ns>      # Install with custom values file
helm install <release-name> <chart> --set key=value -n <ns>     # Install with inline overrides
helm install <release-name> <chart> --dry-run --debug -n <ns>   # Simulate install, render output only
helm install <release-name> <chart> --create-namespace -n <ns>  # Auto-create namespace if missing
helm install <release-name> <chart> --wait --timeout 5m         # Wait for resources to be ready

helm uninstall <release-name> -n <namespace>                    # Remove a release
helm uninstall <release-name> --keep-history -n <namespace>     # Remove but keep release history
```

## Upgrade & Rollback

```bash
helm upgrade <release-name> <chart> -n <namespace>              # Upgrade existing release
helm upgrade --install <release-name> <chart> -n <namespace>    # Upgrade or install if not present (idempotent, great for CI/CD)
helm upgrade <release-name> <chart> -f values.yaml --atomic      # Rollback automatically on failed upgrade
helm upgrade <release-name> <chart> --force                      # Force resource recreation
helm upgrade <release-name> <chart> --reuse-values                # Reuse previously set values, override new ones

helm rollback <release-name> <revision> -n <namespace>           # Rollback to a specific revision
helm rollback <release-name> 0 -n <namespace>                    # Rollback to previous revision (0 = last)
```

## Release Info & History

```bash
helm list -n <namespace>                # List releases in a namespace
helm list -A                            # List releases across all namespaces
helm list --failed                      # List only failed releases
helm status <release-name> -n <ns>      # Show current status of a release
helm history <release-name> -n <ns>     # Show revision history
helm get values <release-name> -n <ns>  # Show values currently in use
helm get values <release-name> --all    # Show computed values (defaults + overrides)
helm get manifest <release-name> -n <ns># Show rendered K8s manifests for a release
helm get hooks <release-name> -n <ns>   # Show hooks defined in a release
helm get notes <release-name> -n <ns>   # Show post-install NOTES.txt output
```

## Chart Authoring & Local Dev

```bash
helm create <chart-name>                # Scaffold a new chart
helm lint <chart-dir>                   # Validate chart syntax/best practices
helm template <chart-dir> -f values.yaml# Render templates locally without installing (great for debugging)
helm template <chart-dir> --show-only templates/deployment.yaml  # Render a single template
helm package <chart-dir>                # Package chart into a .tgz
helm package <chart-dir> --sign --key '<key>' --keyring '<keyring>'  # Sign a packaged chart
helm dependency update <chart-dir>      # Pull/update subchart dependencies (per Chart.yaml)
helm dependency build <chart-dir>       # Rebuild dependencies from Chart.lock
helm dependency list <chart-dir>        # List chart dependencies
```

## Chart Inspection

```bash
helm show values <chart>                # Print a chart's default values.yaml without installing
helm show chart <chart>                 # Print Chart.yaml metadata
helm show readme <chart>                # Print the chart's README
helm show all <chart>                   # All of the above at once
```

## OCI Registry (modern chart distribution)

```bash
helm registry login <registry>
helm push <chart>.tgz oci://<registry>/<repo>
helm pull oci://<registry>/<repo>/<chart> --version <ver>
helm install <release> oci://<registry>/<repo>/<chart> --version <ver>
```

## Debugging

```bash
helm install <release> <chart> --dry-run --debug   # Preview rendered manifests + values before install
helm get manifest <release> -n <ns> | less          # Inspect what's actually deployed
helm diff upgrade <release> <chart> -f values.yaml   # (requires helm-diff plugin) Show diff before upgrading
kubectl describe pod <pod> -n <ns>                   # When a release "succeeds" but pods crash
helm template <chart> --debug                        # Verbose render, useful for template errors
helm test <release> -n <ns>                          # Run test hooks against a live release
helm install <release> <chart> --post-renderer ./script.sh   # Run rendered manifests through a post-renderer (e.g. Kustomize)
```

## Plugins

```bash
helm plugin list                        # List installed plugins
helm plugin install <git-url>           # Install a plugin (e.g., helm-diff, helm-secrets)
helm plugin uninstall <name>            # Remove a plugin
helm plugin update <name>               # Update a plugin
```

Common plugins worth knowing:
- **helm-diff** — preview changes before `upgrade`
- **helm-secrets** — manage encrypted values (pairs well with SOPS/AWS Secrets Manager workflows)
- **helm-unittest** — unit testing for chart templates

## Values & Overrides — Quick Reference

```
1. Chart's default values.yaml
2. Parent chart's values (for subcharts)
3. -f/--values files (later files override earlier ones)
4. --set / --set-string / --set-file (highest precedence)
```

```bash
helm install <release> <chart> -f base.yaml -f prod.yaml --set replicaCount=3
```

## GitOps / ArgoCD-Adjacent Notes

- ArgoCD renders charts via `helm template` internally — validate locally with the same command before pushing, to avoid drift-detection surprises.
- Prefer `helm upgrade --install` semantics in CI pipelines (idempotent); ArgoCD's Helm integration follows the same pattern declaratively.
- Keep `Chart.lock` committed so `helm dependency build` is reproducible in CI runners.
- Use `helm template ... | kubectl diff -f -` locally to sanity-check before ArgoCD syncs, especially for CRDs (Helm doesn't upgrade CRDs on `helm upgrade` by default).

## Useful Flags Cheat Sheet

| Flag | Purpose |
|---|---|
| `-n, --namespace` | Target namespace |
| `--create-namespace` | Create namespace if missing |
| `-f, --values` | Specify values file(s) |
| `--set` | Inline value override |
| `--dry-run` | Simulate without applying |
| `--debug` | Verbose output |
| `--atomic` | Auto-rollback on failure |
| `--wait` | Wait until resources are ready |
| `--timeout` | Max wait duration |
| `--version` | Pin chart version |
| `--force` | Force resource replacement |
| `-A, --all-namespaces` | Operate across all namespaces |

## Version & Env Info

```bash
helm version                            # Show Helm client/server version
helm env                                # Show Helm environment variables (cache, config, plugin paths)
```
