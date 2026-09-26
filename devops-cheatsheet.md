# DevOps Engineer Cheatsheet (5 YOE Level)

Covers: Linux/Bash, Git, Docker, Kubernetes, Helm, Terraform, Jenkins, ArgoCD/GitOps, AWS, Prometheus/Grafana, Networking & Security fundamentals.

---

## 1. Linux / Bash Essentials

```bash
# Process & resource inspection
top / htop                          # Live process/resource view
ps aux | grep <name>                # Find a process
kill -9 <pid>                       # Force kill
df -h                               # Disk usage
du -sh *                            # Directory sizes
free -m                             # Memory usage
netstat -tulnp / ss -tulnp          # Listening ports & owning process
lsof -i :<port>                     # What's using a port

# File & text ops
grep -rn "pattern" .                # Recursive search with line numbers
find . -name "*.log" -mtime +7      # Files older than 7 days
tail -f app.log                     # Live log tail
sed -i 's/old/new/g' file.txt       # In-place replace
awk '{print $1,$3}' file.txt        # Column extraction
xargs                               # Build commands from stdin
chmod 750 / chown user:group        # Permissions & ownership

# Systemd
systemctl status <service>
systemctl restart <service>
journalctl -u <service> -f          # Follow service logs
```

## 2. Git

```bash
git status / git diff / git log --oneline --graph --all
git branch <name> && git checkout <name>       # or: git checkout -b <name>
git rebase -i HEAD~3                            # Interactive rebase (squash/edit history)
git cherry-pick <commit>                        # Apply a specific commit elsewhere
git stash / git stash pop                       # Shelve work-in-progress
git reset --soft/--mixed/--hard HEAD~1          # Undo commits (varying severity)
git revert <commit>                             # Safe undo (creates new commit)
git tag -a v1.2.0 -m "release"                  # Annotated tag for releases
git blame <file>                                # Who changed what, line by line
git log -p --follow <file>                      # Full history of a file across renames
```

## 3. Docker

```bash
docker build -t <image>:<tag> .
docker run -d -p 8080:8080 --name <n> <image>
docker ps -a / docker logs -f <container>
docker exec -it <container> /bin/sh             # Shell into running container
docker system prune -af                         # Clean unused images/containers/volumes
docker inspect <container>                      # Full metadata (network, mounts, env)
docker network ls / docker network inspect <n>
docker-compose up -d / docker-compose down -v
docker save/load                                # Export/import images as tar
docker history <image>                          # Layer-by-layer build history
```

## 4. Kubernetes (kubectl)

```bash
# Context & namespace
kubectl config get-contexts / use-context <ctx>
kubectl config set-context --current --namespace=<ns>

# Core resource ops
kubectl get pods -n <ns> -o wide
kubectl describe pod <pod> -n <ns>              # Events, mounts, conditions — first debug step
kubectl logs -f <pod> -c <container> -n <ns>
kubectl logs --previous <pod>                   # Logs from a crashed/previous container
kubectl exec -it <pod> -n <ns> -- /bin/sh
kubectl apply -f manifest.yaml
kubectl delete -f manifest.yaml
kubectl rollout status/history/undo deployment/<name> -n <ns>
kubectl scale deployment/<name> --replicas=3 -n <ns>
kubectl top pod / kubectl top node              # Requires metrics-server

# Debugging
kubectl get events -n <ns> --sort-by='.lastTimestamp'
kubectl get pods --field-selector=status.phase=Failed -A
kubectl debug <pod> -it --image=busybox         # Ephemeral debug container
kubectl port-forward svc/<svc> 8080:80 -n <ns>

# Config & secrets
kubectl create configmap <name> --from-file=path
kubectl create secret generic <name> --from-literal=key=value
kubectl get secret <name> -o jsonpath='{.data.key}' | base64 -d

# Misc
kubectl explain <resource>.<field>              # Inline API docs
kubectl api-resources                           # List all resource types
kubectl get crd                                 # List CRDs (relevant for ArgoCD/operators)
```

## 5. Helm

*(see companion `helm-cheatsheet.md` for the full command set)*

```bash
helm upgrade --install <release> <chart> -f values.yaml -n <ns>   # Idempotent, CI-friendly
helm template <chart> --debug                                     # Render locally before applying
helm rollback <release> <revision> -n <ns>
```

## 6. Terraform

```bash
terraform init                                  # Init backend + providers
terraform validate                              # Syntax check
terraform fmt -recursive                        # Format code
terraform plan -out=tfplan                      # Preview changes
terraform apply tfplan
terraform destroy -target=<resource>            # Destroy a specific resource
terraform state list                            # List resources in state
terraform state show <resource>
terraform state mv <old> <new>                  # Rename/move without recreation
terraform state rm <resource>                   # Remove from state (not from cloud)
terraform import <resource> <id>                # Bring existing infra under management
terraform workspace list/new/select <name>      # Manage environments (dev/stage/prod)
terraform taint <resource>                      # Force recreation on next apply (or -replace flag)
terraform output                                # Show output values
```

Best practices worth remembering:
- Remote state (S3 + DynamoDB lock table) for team workflows.
- `terraform plan` in CI on every PR; `apply` gated to merge/main.
- Modularize (network, compute, IAM) rather than one giant root module.

## 7. Jenkins

```groovy
// Minimal declarative pipeline skeleton
pipeline {
  agent any
  stages {
    stage('Build') { steps { sh 'mvn clean package' } }
    stage('Test')  { steps { sh 'mvn test' } }
    stage('Docker') { steps { sh 'docker build -t app:$BUILD_NUMBER .' } }
    stage('Deploy') { steps { sh 'helm upgrade --install app ./chart -f values.yaml' } }
  }
  post {
    failure { echo 'Notify on failure (Slack/email)' }
  }
}
```

Useful concepts/commands:
- `Jenkinsfile` in repo root = pipeline as code.
- Shared Libraries for reusable pipeline steps across microservices.
- `parameters {}` block for manual/triggered inputs.
- Blue Ocean / classic UI for visualizing stage failures.
- CLI: `java -jar jenkins-cli.jar -s <url> build <job>` to trigger jobs remotely.
- Credentials binding: `withCredentials([usernamePassword(...)])`.

## 8. ArgoCD / GitOps

```bash
argocd login <server>
argocd app list
argocd app get <app-name>
argocd app sync <app-name>                      # Manual sync
argocd app diff <app-name>                      # Show drift between Git and live state
argocd app history <app-name>
argocd app rollback <app-name> <revision>
argocd repo add <url>
argocd cluster add <context>
```

Concepts:
- App-of-Apps pattern for managing many microservices declaratively.
- Sync policies: automated (with `selfHeal` + `prune`) vs manual.
- ArgoCD renders Helm charts via `helm template` internally — validate the same way locally first.
- Sync waves (`argocd.argoproj.io/sync-wave` annotation) to order resource application.

## 9. AWS (EKS-centric)

```bash
# EKS
aws eks update-kubeconfig --name <cluster> --region <region>
eksctl create cluster --name <n> --nodes 3
eksctl get nodegroup --cluster <cluster>
eksctl utils associate-iam-oidc-provider --cluster <cluster> --approve   # For IRSA

# IAM
aws iam list-attached-role-policies --role-name <role>
aws sts get-caller-identity                     # "Who am I" check — first debug step for auth issues

# Secrets Manager
aws secretsmanager get-secret-value --secret-id <id>
aws secretsmanager create-secret --name <n> --secret-string '<json>'

# S3 / general
aws s3 sync ./local s3://bucket/path
aws s3 ls s3://bucket --recursive
aws logs tail /aws/eks/<cluster>/cluster --follow    # CloudWatch log tailing

# ECR
aws ecr get-login-password | docker login --username AWS --password-stdin <account>.dkr.ecr.<region>.amazonaws.com
```

Concepts:
- IRSA (IAM Roles for Service Accounts) — how pods get scoped AWS permissions instead of node-wide roles.
- VPC CNI, security groups per pod vs per node.
- Cluster Autoscaler vs Karpenter for node scaling.

## 10. Observability (Prometheus / Grafana)

```promql
# PromQL basics
rate(http_requests_total[5m])                   # Per-second rate over 5m window
sum(rate(http_requests_total[5m])) by (service)
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
up == 0                                         # Targets currently down
increase(container_restarts_total[1h])
```

```bash
kubectl port-forward svc/prometheus-server 9090:80 -n monitoring
kubectl port-forward svc/grafana 3000:80 -n monitoring
```

Concepts:
- Alertmanager routing/grouping/silencing rules for on-call.
- ServiceMonitor / PodMonitor CRDs (Prometheus Operator) to auto-discover scrape targets.
- RED (Rate, Errors, Duration) and USE (Utilization, Saturation, Errors) methods for dashboard design.

## 11. Networking & Security Fundamentals

```bash
curl -v https://<host>                          # Verbose request/response, TLS handshake visible
dig <domain> / nslookup <domain>                 # DNS resolution
traceroute <host>                                # Hop-by-hop path
openssl s_client -connect <host>:443             # Inspect TLS cert chain
```

Concepts to keep sharp:
- OSI layers 3 (IP/routing) through 7 (app) — where a failure sits changes the fix.
- K8s NetworkPolicy — default-deny + explicit allow between namespaces/pods.
- Ingress vs Service (ClusterIP/NodePort/LoadBalancer) vs Gateway API.
- Secrets: never in Git — use Sealed Secrets, External Secrets Operator, or Secrets Manager/Vault injection.
- Least-privilege IAM: scope by resource + action, avoid `*:*`.

## 12. CI/CD Design Principles (the "why" behind the commands)

- Idempotent deploys: `helm upgrade --install`, `terraform apply` — safe to re-run.
- Immutable artifacts: build once, promote the same image/chart across environments (dev → stage → prod).
- Fail fast: lint/test/security-scan stages before build; build stages before deploy.
- Progressive delivery: canary/blue-green via Argo Rollouts or service-mesh traffic splitting.
- Rollback path defined before rollout: know your `helm rollback` / `argocd app rollback` / `kubectl rollout undo` in advance, not mid-incident.

## 13. Incident/Debug Checklist (quick recall under pressure)

1. `kubectl get pods -A | grep -v Running` — what's not healthy right now.
2. `kubectl describe pod <pod>` — events first, they usually say why.
3. `kubectl logs --previous` — if it crashed and restarted.
4. `argocd app diff` — did GitOps drift, or did someone `kubectl edit` directly.
5. `aws sts get-caller-identity` — rule out auth/IAM before blaming the app.
6. Check Grafana dashboards for the RED metrics around the incident window before diving into logs.
