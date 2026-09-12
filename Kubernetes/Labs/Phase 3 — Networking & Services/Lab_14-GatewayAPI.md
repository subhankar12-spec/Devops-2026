# Kubernetes Hands-On Lab 14 — Gateway API

## 14.1 Objectives

By the end of this lab, you should be able to:

- Understand why the Gateway API exists and how it differs from Ingress
- Install Gateway API CRDs and a Gateway controller implementation
- Create a `GatewayClass` and a `Gateway`
- Create an `HTTPRoute` for path-based and host-based routing
- Understand the role-oriented design (infra admin vs application developer resources)
- Use a `ReferenceGrant` to allow cross-namespace routing
- Troubleshoot a Route that isn't attaching to its Gateway
- Perform common Gateway API tasks quickly for the CKA

## 14.2 Architecture

```
                 GatewayClass
              (cluster-scoped, defines
               WHICH controller implements
               Gateways of this class —
               like IngressClass, but for Gateway API)
                       |
                    Gateway
         (namespaced, the actual listener:
          which ports/protocols/hostnames
          it accepts — created by
          infra/platform team)
                       |
        +--------------+--------------+
        |                             |
        v                             v
    HTTPRoute                    HTTPRoute
 (namespaced, created by       (namespaced, created by
  app team A — routing          app team B — routing
  rules for their Service)      rules for their Service)
        |                             |
        v                             v
   Service: team-a-svc          Service: team-b-svc
```

Key concept: Gateway API deliberately **splits roles** that Ingress mashed into one resource. A platform/infra team owns the `GatewayClass` and `Gateway` (the shared listener/load balancer). Application teams each own their own `HTTPRoute` objects that attach to that shared Gateway — no more fighting over annotations on one giant Ingress object, and much cleaner multi-tenancy.

## 14.3 Gateway API vs Ingress

| | Ingress | Gateway API |
|---|---|---|
| Role separation | One object, one owner | Split: GatewayClass/Gateway (infra) vs HTTPRoute (app team) |
| Protocol support | HTTP/HTTPS only (natively) | HTTP, HTTPS, TCP, UDP, gRPC, TLS passthrough |
| Cross-namespace routing | Awkward/controller-specific | Native, via `ReferenceGrant` |
| Traffic splitting / weighting | Controller-specific annotations | Native (`backendRefs` with `weight`) |
| Extensibility | Annotations (non-portable across controllers) | Typed fields + policy attachment (portable) |
| Status | Older, simpler use cases | GA (`v1`) for GatewayClass/Gateway/HTTPRoute/GRPCRoute; the modern direction for new work |

## 14.4 Prerequisites

```bash
kubectl version --client
kubectl get nodes
```

Check whether Gateway API CRDs are already installed:

```bash
kubectl get crd | grep gateway.networking.k8s.io
```

## 14.5 Lab 1 — Install Gateway API CRDs

Install the standard channel CRDs:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml
```

Verify:

```bash
kubectl get crd | grep gateway.networking.k8s.io
```

Expected CRDs include `gatewayclasses`, `gateways`, `httproutes`, `referencegrants`.

## 14.6 Install a Gateway Controller

The CRDs alone define the API shape — like Ingress, you still need a controller implementation to actually do anything. Common options: Envoy Gateway, Istio, Cilium, NGINX Gateway Fabric, or your cloud provider's implementation (e.g. GKE Gateway).

Example using Envoy Gateway:

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.1.0 -n envoy-gateway-system --create-namespace

kubectl wait --timeout=5m -n envoy-gateway-system \
  deployment/envoy-gateway --for=condition=Available
```

Verify the GatewayClass it registers:

```bash
kubectl get gatewayclass
```

## 14.7 Lab 2 — Create Backing Services

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: team-a-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: team-a-app
  template:
    metadata:
      labels:
        app: team-a-app
    spec:
      containers:
        - name: app
          image: hashicorp/http-echo
          args: ["-text=Response from team-a"]
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: team-a-svc
spec:
  selector:
    app: team-a-app
  ports:
    - port: 80
      targetPort: 5678
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: team-a-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: team-a-app
  template:
    metadata:
      labels:
        app: team-a-app
    spec:
      containers:
        - name: app
          image: hashicorp/http-echo
          args: ["-text=Response from team-a"]
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: team-a-svc
spec:
  selector:
    app: team-a-app
  ports:
    - port: 80
      targetPort: 5678
EOF
```

## 14.8 Lab 3 — Create a Gateway

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gateway
spec:
  gatewayClassName: eg
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
```

Apply it (adjust `gatewayClassName` to whatever your controller registered in 14.6):

```bash
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gateway
spec:
  gatewayClassName: eg
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
EOF
```

Check its status:

```bash
kubectl get gateway shared-gateway
kubectl describe gateway shared-gateway
```

Wait for `PROGRAMMED: True` in the conditions before continuing.

## 14.9 Lab 4 — Create an HTTPRoute

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: team-a-route
spec:
  parentRefs:
    - name: shared-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /team-a
      backendRefs:
        - name: team-a-svc
          port: 80
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: team-a-route
spec:
  parentRefs:
    - name: shared-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /team-a
      backendRefs:
        - name: team-a-svc
          port: 80
EOF
```

Check it attached successfully:

```bash
kubectl get httproute team-a-route
kubectl describe httproute team-a-route
```

Look for a `Accepted: True` and `ResolvedRefs: True` condition under `Parents:`.

Get the Gateway's external address and test:

```bash
kubectl get gateway shared-gateway -o jsonpath='{.status.addresses[0].value}'
curl http://<gateway-address>/team-a
```

## 14.10 Host-Based Routing with HTTPRoute

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: team-a-host-route
spec:
  parentRefs:
    - name: shared-gateway
  hostnames:
    - "team-a.example.local"
  rules:
    - backendRefs:
        - name: team-a-svc
          port: 80
```

Apply and test with a Host header:

```bash
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: team-a-host-route
spec:
  parentRefs:
    - name: shared-gateway
  hostnames:
    - "team-a.example.local"
  rules:
    - backendRefs:
        - name: team-a-svc
          port: 80
EOF

curl -H "Host: team-a.example.local" http://<gateway-address>/
```

## 14.11 Cross-Namespace Routing with ReferenceGrant

By default, a Route in Namespace A cannot reference a Service in Namespace B unless explicitly permitted. Create the target namespace and Service:

```bash
kubectl create namespace team-b

kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: team-b-app
  namespace: team-b
spec:
  replicas: 1
  selector:
    matchLabels:
      app: team-b-app
  template:
    metadata:
      labels:
        app: team-b-app
    spec:
      containers:
        - name: app
          image: hashicorp/http-echo
          args: ["-text=Response from team-b"]
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: team-b-svc
  namespace: team-b
spec:
  selector:
    app: team-b-app
  ports:
    - port: 80
      targetPort: 5678
EOF
```

Allow the default namespace's HTTPRoutes to reference Services in `team-b`:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-default-to-team-b
  namespace: team-b
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      namespace: default
  to:
    - group: ""
      kind: Service
```

Apply it, then create a Route in `default` pointing across the namespace boundary:

```bash
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-default-to-team-b
  namespace: team-b
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      namespace: default
  to:
    - group: ""
      kind: Service
EOF

kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: team-b-cross-ns-route
spec:
  parentRefs:
    - name: shared-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /team-b
      backendRefs:
        - name: team-b-svc
          namespace: team-b
          port: 80
EOF

curl http://<gateway-address>/team-b
```

## 14.12 Break It — Route Not Attaching

Create an HTTPRoute pointing at a Gateway that doesn't exist:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: orphan-route
spec:
  parentRefs:
    - name: nonexistent-gateway
  rules:
    - backendRefs:
        - name: team-a-svc
          port: 80
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: orphan-route
spec:
  parentRefs:
    - name: nonexistent-gateway
  rules:
    - backendRefs:
        - name: team-a-svc
          port: 80
EOF
```

## 14.13 Diagnose and Recover

```bash
kubectl get httproute orphan-route
kubectl describe httproute orphan-route
```

Look for a condition like:

```
Type: Accepted   Status: False   Reason: NoMatchingParent
```

Fix by pointing at the real Gateway:

```bash
kubectl patch httproute orphan-route --type=json \
  -p='[{"op":"replace","path":"/spec/parentRefs/0/name","value":"shared-gateway"}]'

kubectl describe httproute orphan-route
```

Confirm `Accepted: True` and `ResolvedRefs: True` now appear.

## 14.14 CKA Practice Task

**Task**

1. Confirm Gateway API CRDs and a controller are installed, and identify the `GatewayClass` name.
2. Create a `Gateway` named `cka-gateway` listening on HTTP port 80.
3. Create a Deployment/Service `checkout-app`/`checkout-svc` on port 80.
4. Create an `HTTPRoute` named `checkout-route` attaching to `cka-gateway`, matching path prefix `/checkout`.
5. Confirm `Accepted: True` in the HTTPRoute's status conditions.

**Target Time**

8 minutes

Try it without looking at previous commands.

## 14.15 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

What is the core role-separation idea behind Gateway API that Ingress doesn't have?

**Question 2**

What object plays the same conceptual role for Gateway API that `IngressClass` plays for Ingress?

**Question 3**

Why would an `HTTPRoute` fail to attach to a `Gateway` even though both objects exist and look syntactically correct?

**Question 4**

What problem does `ReferenceGrant` solve, and why does it exist?

**Question 5**

Name two protocols or capabilities Gateway API supports natively that Ingress does not.

**Question 6**

What two status conditions on an `HTTPRoute` tell you whether it successfully attached and whether its backend reference is valid?

## 14.16 Useful Commands

```bash
kubectl get crd | grep gateway.networking.k8s.io
kubectl get gatewayclass
kubectl get gateway
kubectl describe gateway <name>
kubectl get httproute
kubectl describe httproute <name>
kubectl get referencegrant -n <namespace>

kubectl get gateway <name> -o jsonpath='{.status.addresses[0].value}'
```

## 14.17 Cleanup

```bash
kubectl delete httproute team-a-route team-a-host-route team-b-cross-ns-route \
  orphan-route checkout-route
kubectl delete gateway shared-gateway cka-gateway
kubectl delete deployment team-a-app checkout-app
kubectl delete svc team-a-svc checkout-svc
kubectl delete namespace team-b
```

## 14.18 Lab Checklist

- [ ] Installed Gateway API CRDs
- [ ] Installed a Gateway controller and confirmed its GatewayClass
- [ ] Created a Gateway and confirmed `PROGRAMMED: True`
- [ ] Created an HTTPRoute for path-based routing
- [ ] Created an HTTPRoute for host-based routing
- [ ] Understood the infra-team vs app-team role split
- [ ] Used a ReferenceGrant to allow cross-namespace routing
- [ ] Reproduced an HTTPRoute failing to attach to a nonexistent Gateway
- [ ] Diagnosed via `Accepted`/`ResolvedRefs` conditions and fixed it
- [ ] Completed the CKA task within 8 minutes
