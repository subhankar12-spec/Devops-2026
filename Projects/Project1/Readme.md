## Architecture

This is a **microservices architecture**, not a monolithic 3-tier application. There is no frontend — it's an API-first platform, tested via `curl`/Postman/k6, focused on the deployment, reliability, and operations layer rather than UI.

### Services

| Service | Responsibility | Pattern |
|---|---|---|
| `order` | Accepts orders via REST, persists to its own schema, publishes an `order.created` event | Sync API + async producer |
| `payment` | Consumes `order.created`, processes payment idempotently, publishes `payment.processed` | Async consumer/producer |
| `notification` | Consumes `payment.processed`, simulates notification delivery (can fail intentionally for DLQ/retry testing) | Async consumer |

Each service:
- Owns its own database schema (no shared tables across services)
- Is independently deployable via its own Helm chart and CI pipeline
- Communicates with others only through RabbitMQ events — never direct service-to-service calls

### Why microservices instead of 3-tier

A 3-tier app (UI → single backend → single database) doesn't exercise the problems this project is meant to teach: independent deployability, async messaging failure modes, per-service scaling, service-to-service network policy, and distributed tracing/observability. Those require multiple independently-owned services, which is why this project is structured as microservices rather than a layered monolith.

### Infrastructure

| Layer | Tools |
|---|---|
| Orchestration | Kubernetes (kind, locally) |
| CI/CD | Jenkins → Trivy scan → local registry |
| GitOps / CD | ArgoCD (app-of-apps) |
| Messaging | RabbitMQ (event-driven, with DLQ/retry) |
| Data | PostgreSQL (per-service schema) |
| Ingress/TLS | NGINX Ingress + cert-manager |
| Observability | Prometheus, Grafana, Alertmanager, Loki |
| Security | RBAC, NetworkPolicy, Sealed Secrets |
