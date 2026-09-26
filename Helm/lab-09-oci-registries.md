# Lab 09 — OCI Registries

## Objectives
- Push and pull Helm charts as OCI artifacts
- Use AWS ECR as an OCI-compliant Helm registry
- Understand the authentication flow and tagging conventions
- Practice debugging two common OCI push/pull mistakes

## Prerequisites
- Labs 01–08 completed
- Helm 3.8+ (OCI support is stable by default from 3.8 onward)
- AWS CLI configured with ECR permissions

---

## 1. Why OCI Registries

Since Helm 3.8, charts can be stored as OCI artifacts in any OCI-compliant registry — the same kind of registry that stores container images (ECR, GHCR, Docker Hub, ACR, GCR, Harbor). This eliminates the need for the traditional `index.yaml`-based chart repository model from Lab 08 entirely: no separate hosting, no manual index merging, no drift risk.

This is directly relevant to your stack — ECR isn't just for container images anymore; it can host your Helm charts too, in the same registry infrastructure you already manage.

## 2. Authenticating to ECR

```bash
aws ecr get-login-password --region us-east-1 | \
  helm registry login --username AWS --password-stdin \
  <account-id>.dkr.ecr.us-east-1.amazonaws.com
```
This is nearly identical to the `docker login` flow for ECR — same short-lived token mechanism (12 hours), same `aws ecr get-login-password` command.

## 3. Creating the Repository

Unlike traditional chart repos, ECR requires the repository to exist **before** you can push to it — `helm push` will not auto-create it:
```bash
aws ecr create-repository --repository-name helm/mychart --region us-east-1
```

## 4. Pushing a Chart

```bash
helm package mychart
helm push mychart-0.1.0.tgz oci://<account-id>.dkr.ecr.us-east-1.amazonaws.com/helm
```
Note the URL: it points to the **path/namespace** (`helm`), not including the chart name or version — Helm derives the repository name and tag from the packaged chart's `Chart.yaml` (`mychart:0.1.0`).

## 5. Pulling and Installing

```bash
helm pull oci://<account-id>.dkr.ecr.us-east-1.amazonaws.com/helm/mychart --version 0.1.0

# Or install directly without a separate pull step
helm install demo oci://<account-id>.dkr.ecr.us-east-1.amazonaws.com/helm/mychart --version 0.1.0
```
Note: `--version` is **required** with OCI references for `install`/`upgrade`/`pull` — there's no `helm search repo`-style discovery against an OCI registry the way there is with `index.yaml`; you need to already know (or look up via `aws ecr describe-images`) which tags/versions exist.

---

## 6. Troubleshooting Exercise 1 — Push Fails, Repository Not Pre-Created

**Scenario:**
```bash
helm push mychart-0.1.0.tgz oci://<account-id>.dkr.ecr.us-east-1.amazonaws.com/helm
```
```
Error: unexpected status: 404 Not Found
```

**Task:**
1. Recognize that unlike Docker Hub (which can auto-create repos on first push depending on settings) or a traditional chart repo (just files on a server), ECR repositories must exist beforehand
2. Run `aws ecr describe-repositories --repository-names helm/mychart` to confirm it doesn't exist yet
3. Create it: `aws ecr create-repository --repository-name helm/mychart`
4. Re-run `helm push` and confirm success
5. Document this as a required Terraform/IaC step (relevant to your stack) — chart repositories should be provisioned alongside application infrastructure, not created ad hoc by whoever happens to run the first push

## 7. Troubleshooting Exercise 2 — Version Tag Collision

**Scenario:** CI re-runs a pipeline without bumping `Chart.yaml`'s `version`, and tries to push again:
```bash
helm push mychart-0.1.0.tgz oci://.../helm
```
```
Error: PUT ... 400 Bad Request ... tag already exists
```
(Behavior here depends on the registry's tag immutability setting — ECR repositories can be configured as immutable, which is the safer production default and is what causes this hard failure.)

**Task:**
1. Check the repository's tag mutability setting: `aws ecr describe-repositories --repository-names helm/mychart --query 'repositories[0].imageTagMutability'`
2. If `IMMUTABLE` (recommended for any shared/production chart repo), understand that this is working as intended — it's preventing an accidental overwrite of a version someone may already have deployed
3. Fix the actual root cause: bump `version` in `Chart.yaml` before re-packaging and pushing, per the versioning discipline from Lab 02
4. Re-package and push with the new version, confirm success

**Takeaway:** immutable tags on a chart registry are a deliberate safety net, not a bug — the correct fix is always "bump the version," never "force overwrite," for anything beyond local scratch testing.

---

## Cleanup
```bash
rm -f mychart-0.1.0.tgz
helm registry logout <account-id>.dkr.ecr.us-east-1.amazonaws.com
```

---

**Phase 3 (Packaging & Distribution) complete.** Next: Phase 4 — Production Patterns, starting with Lab 10 — Hooks.
