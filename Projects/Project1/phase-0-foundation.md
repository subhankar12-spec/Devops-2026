# Phase 0: Foundation

## Goal

Stand up a local Kubernetes platform foundation that mirrors a production bootstrap: a cluster, ingress with TLS, an image registry, GitOps tooling, and a CI entry point — all reproducible from a single command, with no Terraform (local-only, no cloud cost).

**Definition of done:** running `make cluster` from a clean machine gives you a kind cluster with working ingress, TLS, a local registry, and ArgoCD installed and reachable, in under 10 minutes.

---

## 0. Prerequisites

Install on your machine (WSL2 recommended if on Windows):

- Docker Desktop (or Docker Engine) — with at least 20 GB RAM allocated
- `kind` (Kubernetes in Docker)
- `kubectl`
- `helm`
- `mkcert` (optional, for trusted local TLS certs)
- Jenkins will run as a container, no local install needed

Verify:

```bash
docker version
kind version
kubectl version --client
helm version
```

---

## 1. Local Cluster (kind)

### 1.1 Cluster topology

- 1 control-plane node
- 2 worker nodes
- Port mappings for 80/443 so ingress is reachable from the host

### 1.2 `kind-config.yaml`

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: mega-local
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
  - role: worker
  - role: worker
```

### 1.3 Makefile targets

```makefile
CLUSTER_NAME=mega-local

.PHONY: cluster destroy

cluster:
	kind create cluster --name $(CLUSTER_NAME) --config kind-config.yaml
	kubectl cluster-info --context kind-$(CLUSTER_NAME)

destroy:
	kind delete cluster --name $(CLUSTER_NAME)
```

### 1.4 Verify

```bash
make cluster
kubectl get nodes -o wide
```

You should see 1 control-plane + 2 workers, all `Ready`.

---

## 2. Ingress + TLS

### 2.1 Install NGINX Ingress Controller

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.hostPort.enabled=true \
  --set controller.service.type=NodePort \
  --set controller.nodeSelector."ingress-ready"="true" \
  --set controller.tolerations[0].key="node-role.kubernetes.io/control-plane" \
  --set controller.tolerations[0].operator="Exists" \
  --set controller.tolerations[0].effect="NoSchedule"
```

Wait for the controller pod to be `Running`:

```bash
kubectl get pods -n ingress-nginx --watch
```

### 2.2 Install cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true
```

### 2.3 Local CA / self-signed issuer

**Option A — self-signed ClusterIssuer (simplest):**

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
```

**Option B — mkcert (trusted in your browser, no cert warnings):**

```bash
mkcert -install
mkcert "*.mega.local"
kubectl create secret tls mega-local-tls \
  --cert=_wildcard.mega.local.pem \
  --key=_wildcard.mega.local-key.pem \
  -n default
```

Add `127.0.0.1 mega.local` and any subdomains you use to `/etc/hosts`.

Recommendation: start with Option A (self-signed) for speed; switch to mkcert once you're tired of curl `-k` flags.

### 2.4 Test ingress + TLS end to end

```bash
kubectl create deployment hello --image=nginxdemos/hello
kubectl expose deployment hello --port=80
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-ingress
  annotations:
    cert-manager.io/cluster-issuer: selfsigned-issuer
spec:
  ingressClassName: nginx
  tls:
    - hosts: ["hello.mega.local"]
      secretName: hello-tls
  rules:
    - host: hello.mega.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: hello
                port:
                  number: 80
```

Add `127.0.0.1 hello.mega.local` to `/etc/hosts`, then:

```bash
curl -k https://hello.mega.local
```

You should see the nginx demo response. Delete the test deployment once confirmed.

---

## 3. Local Image Registry

### 3.1 Run the registry container

```bash
docker run -d --restart=always -p 5000:5000 --name local-registry registry:2
docker network connect kind local-registry
```

### 3.2 Configure containerd on kind nodes to use it

kind documents a "local registry" pattern that patches each node's containerd config to redirect `localhost:5000` pulls to the `local-registry` container. Apply the standard kind local-registry manifest (from kind's official docs) after cluster creation — this creates a `ConfigMap` documenting the registry and patches containerd on each node.

### 3.3 Test push/pull

```bash
docker pull alpine:3.19
docker tag alpine:3.19 localhost:5000/alpine:3.19
docker push localhost:5000/alpine:3.19

kubectl run test-pull --image=localhost:5000/alpine:3.19 --command -- sleep 3600
kubectl get pod test-pull
```

If the pod reaches `Running`, the registry is wired correctly. Clean up:

```bash
kubectl delete pod test-pull
```

---

## 4. GitOps Repos + ArgoCD

### 4.1 Repo layout

Two repos on Bitbucket:

- **`mega-app`** — service source code (order, payment, notification), Dockerfiles, Jenkinsfile
- **`mega-gitops`** — Helm charts + per-environment values, ArgoCD Application manifests

```
mega-gitops/
├── apps/
│   ├── order/
│   │   ├── Chart.yaml
│   │   ├── values-dev.yaml
│   │   └── templates/
│   ├── payment/
│   └── notification/
├── argocd/
│   ├── root-app.yaml        # app-of-apps
│   └── projects/
└── environments/
    ├── dev/
    ├── stage/
    └── prod/
```

### 4.2 Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for pods:

```bash
kubectl get pods -n argocd --watch
```

### 4.3 Expose ArgoCD via ingress + TLS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: selfsigned-issuer
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  tls:
    - hosts: ["argocd.mega.local"]
      secretName: argocd-tls
  rules:
    - host: argocd.mega.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 443
```

Add `127.0.0.1 argocd.mega.local` to `/etc/hosts`.

### 4.4 Get initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

Log in at `https://argocd.mega.local` as `admin`.

### 4.5 Root app-of-apps

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://bitbucket.org/<you>/mega-gitops.git
    targetRevision: main
    path: environments/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Apply it, then confirm the root app appears and syncs (even with an empty `environments/dev` to start) in the ArgoCD UI.

---

## 5. Bitbucket + Jenkins (pipeline stub)

### 5.1 Bitbucket

- Create `mega-app` and `mega-gitops` repos on Bitbucket Cloud
- Push local scaffolding to both

### 5.2 Run Jenkins as a container

```bash
docker network create jenkins-net 2>/dev/null || true

docker run -d --name jenkins \
  --network jenkins-net \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts
```

Get the initial admin password:

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Install suggested plugins, plus: Docker Pipeline, Bitbucket, Git.

### 5.3 SCM polling (webhooks aren't practical locally)

Since Jenkins runs locally and Bitbucket Cloud can't reach it, configure the pipeline job with **"Poll SCM"** (e.g. `H/5 * * * *` — every 5 minutes) instead of a webhook. Note this as a known deviation from production (which would use a real webhook).

### 5.4 Pipeline stub (`Jenkinsfile` in `mega-app`)

```groovy
pipeline {
  agent any
  stages {
    stage('Checkout') {
      steps { checkout scm }
    }
    stage('Build (stub)') {
      steps { echo 'build step placeholder — real build comes in Phase 1' }
    }
    stage('Push (stub)') {
      steps { echo 'push step placeholder — real push comes in Phase 1' }
    }
  }
}
```

Confirm the job runs end to end (even as a stub) triggered by SCM polling after a commit.

---

## 6. Phase 0 Deliverables Checklist

- [ ] `make cluster` brings up kind cluster (1 control-plane + 2 workers) in one command
- [ ] NGINX ingress installed and verified with a test app over HTTPS
- [ ] cert-manager installed with a working ClusterIssuer
- [ ] Local registry running and reachable from kind nodes (push/pull test passed)
- [ ] `mega-app` and `mega-gitops` repos created on Bitbucket
- [ ] ArgoCD installed, reachable via ingress, root app-of-apps syncing
- [ ] Jenkins running as a container, SCM-polling pipeline stub executes successfully
- [ ] `/etc/hosts` entries documented for all local domains used
- [ ] README with an architecture diagram of what's running
- [ ] Short write-up: what broke during setup and how you fixed it

## 7. Known Local-vs-Production Deviations (log these — useful for interviews)

| Local approach | Production equivalent |
|---|---|
| kind cluster | Amazon EKS |
| Self-signed / mkcert TLS | ACM-issued certificates |
| `registry:2` container | Amazon ECR |
| SCM polling | Real Bitbucket webhook to Jenkins |
| Jenkins as a single Docker container | Jenkins on dedicated infra / EC2 / controller-agent setup |
| No Terraform (Makefile + scripts) | Terraform-managed VPC, EKS, IAM |

## 8. Next: Phase 1

Once every box above is checked, move to **Phase 1: Thin Slice** — building the `order` service, wiring the real Jenkins pipeline (test → scan → build → push → update GitOps repo), and getting ArgoCD to deploy it end to end.
