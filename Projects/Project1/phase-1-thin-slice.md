# Phase 1: Thin Slice

## Goal

Get one real service (`order`) flowing through the entire pipeline end to end: code commit → Jenkins (test, scan, build, push) → GitOps repo update → ArgoCD sync → running behind HTTPS ingress. This proves the whole delivery chain before you add more services in Phase 2.

**Definition of done:** a code change to `order`, pushed to Bitbucket, results in a new version running in the cluster with no manual `kubectl` or `helm` commands — Jenkins and ArgoCD do it all.

---

## 1. The `order` service

### 1.1 Scope (deliberately minimal)

- `POST /orders` — accepts `{item, quantity}`, writes a row to PostgreSQL, publishes an `order.created` event to RabbitMQ
- `GET /orders/{id}` — reads from PostgreSQL
- `GET /health` — liveness/readiness target
- No auth, no business logic beyond this — the point is the platform, not the app

### 1.2 Structure (`mega-app/order/`)

```
order/
├── app/
│   ├── main.py
│   ├── db.py
│   ├── models.py
│   └── rabbitmq.py
├── tests/
│   └── test_orders.py
├── Dockerfile
├── requirements.txt
└── Jenkinsfile
```

### 1.3 `requirements.txt`

```
fastapi==0.115.0
uvicorn[standard]==0.30.6
sqlalchemy==2.0.35
psycopg2-binary==2.9.9
pika==1.3.2
pytest==8.3.3
httpx==0.27.2
```

### 1.4 `app/main.py` (minimal)

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from app.db import get_session, init_db
from app.models import Order
from app.rabbitmq import publish_event

app = FastAPI()

class OrderRequest(BaseModel):
    item: str
    quantity: int

@app.on_event("startup")
def startup():
    init_db()

@app.get("/health")
def health():
    return {"status": "ok"}

@app.post("/orders")
def create_order(req: OrderRequest):
    with get_session() as session:
        order = Order(item=req.item, quantity=req.quantity)
        session.add(order)
        session.commit()
        session.refresh(order)
    publish_event("order.created", {"order_id": order.id, "item": order.item})
    return {"id": order.id, "item": order.item, "quantity": order.quantity}

@app.get("/orders/{order_id}")
def get_order(order_id: int):
    with get_session() as session:
        order = session.get(Order, order_id)
        if not order:
            raise HTTPException(status_code=404, detail="not found")
        return {"id": order.id, "item": order.item, "quantity": order.quantity}
```

Fill in `db.py` (SQLAlchemy session + `DATABASE_URL` from env), `models.py` (simple `Order` table), and `rabbitmq.py` (a `pika` publisher using `RABBITMQ_URL` from env). Keep them under ~30 lines each — resist the urge to make this "real."

### 1.5 `Dockerfile`

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app ./app
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 1.6 `tests/test_orders.py`

A couple of basic tests (health check responds, order creation returns 200) — just enough for the Jenkins "test" stage to mean something.

---

## 2. PostgreSQL and RabbitMQ (minimal, non-HA for now)

Phase 1 keeps these as plain Deployments with a single replica and a PVC — StatefulSets, backups, and HA come in Phase 3. Don't over-build this yet.

### 2.1 Helm values sketch (`mega-gitops/apps/order/values-dev.yaml`)

```yaml
image:
  repository: localhost:5000/order
  tag: dev-latest
env:
  DATABASE_URL: postgresql://order:order@postgres:5432/orders
  RABBITMQ_URL: amqp://guest:guest@rabbitmq:5672/
replicaCount: 1
service:
  port: 8000
ingress:
  enabled: true
  host: order.mega.local
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 250m
    memory: 256Mi
```

Add PostgreSQL and RabbitMQ as simple Bitnami Helm chart dependencies (or your own thin charts) in `mega-gitops/apps/postgres/` and `mega-gitops/apps/rabbitmq/`, each with 1 replica and a small PVC.

---

## 3. Helm chart for `order` (reusable template)

Since `payment` and `notification` will reuse this template in Phase 2, keep it generic:

```
mega-gitops/apps/order/
├── Chart.yaml
├── values-dev.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    └── hpa.yaml   # placeholder, enabled=false until Phase 3
```

Parameterize `image.repository`, `image.tag`, `env`, `resources`, and `ingress.host` — everything else stays fixed so `payment` and `notification` are just new `values-*.yaml` files against the same templates.

---

## 4. Real Jenkins Pipeline

Replace Phase 0's stub `Jenkinsfile` with the real stages:

```groovy
pipeline {
  agent any
  environment {
    IMAGE = "localhost:5000/order"
    TAG = "dev-${env.BUILD_NUMBER}"
  }
  stages {
    stage('Checkout') {
      steps { checkout scm }
    }
    stage('Unit Tests') {
      steps {
        sh 'pip install -r requirements.txt'
        sh 'pytest tests/'
      }
    }
    stage('Build Image') {
      steps {
        sh "docker build -t ${IMAGE}:${TAG} ."
      }
    }
    stage('Scan Image (Trivy)') {
      steps {
        sh "trivy image --severity HIGH,CRITICAL --exit-code 1 ${IMAGE}:${TAG}"
      }
    }
    stage('Push Image') {
      steps {
        sh "docker push ${IMAGE}:${TAG}"
      }
    }
    stage('Update GitOps Repo') {
      steps {
        sh '''
          git clone https://bitbucket.org/<you>/mega-gitops.git
          cd mega-gitops
          yq -i ".image.tag = \\"${TAG}\\"" apps/order/values-dev.yaml
          git commit -am "order: bump image to ${TAG}"
          git push
        '''
      }
    }
  }
}
```

Notes:
- Trivy needs to be installed in the Jenkins container (or run via a Trivy Docker image) — add it to the Jenkins Dockerfile/setup now rather than deferring.
- `yq` is used to patch the values file in place; install it in the Jenkins container too.
- This is the point where the pipeline actually deviates from a stub — treat the first successful run of this pipeline as a real milestone.

---

## 5. ArgoCD wiring

Add an `Application` for `order` (or fold it into the existing app-of-apps under `environments/dev`):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://bitbucket.org/<you>/mega-gitops.git
    targetRevision: main
    path: apps/order
    helm:
      valueFiles: ["values-dev.yaml"]
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Confirm in the ArgoCD UI that it detects the `values-dev.yaml` change from the Jenkins pipeline and auto-syncs.

---

## 6. End-to-End Verification

1. Make a trivial code change to `order` (e.g. change a response field)
2. Commit and push to Bitbucket
3. Watch Jenkins pick it up via SCM polling, run through all stages
4. Watch the GitOps repo receive the commit with the new tag
5. Watch ArgoCD auto-sync and roll out the new pod
6. Confirm via `curl -k https://order.mega.local/health` and `POST /orders`
7. Confirm a message actually lands in RabbitMQ (`rabbitmqctl list_queues` or the management UI)

If all seven steps work without a manual step, Phase 1 is done.

---

## 7. Phase 1 Deliverables Checklist

- [ ] `order` service running with `/health`, `POST /orders`, `GET /orders/{id}`
- [ ] PostgreSQL and RabbitMQ running (single replica, PVC-backed)
- [ ] Reusable Helm chart template for services
- [ ] Jenkins pipeline: test → Trivy scan → build → push → GitOps update, all real (no stubs)
- [ ] ArgoCD auto-syncing `order` from the GitOps repo
- [ ] End-to-end test passed: commit → running new version, zero manual `kubectl`/`helm`
- [ ] README updated with the pipeline flow diagram
- [ ] Short write-up: first real pipeline failure you hit and how you debugged it

## 8. Known Local-vs-Production Deviations (add to Phase 0's table)

| Local approach | Production equivalent |
|---|---|
| Single-replica PostgreSQL/RabbitMQ, no HA yet | Managed RDS / HA RabbitMQ cluster |
| `yq` + git push from Jenkins to update GitOps repo | Same pattern in real orgs, but usually via a bot account + PR, not direct push to `main` |
| No image signing | Cosign/Notary in a hardened pipeline |

## 9. Next: Phase 2

Add `payment` and `notification` using the same Helm template and Jenkins pipeline pattern, wire the async chain (`order` → `payment` → `notification`), and introduce basic DLQ/retry handling on RabbitMQ.
