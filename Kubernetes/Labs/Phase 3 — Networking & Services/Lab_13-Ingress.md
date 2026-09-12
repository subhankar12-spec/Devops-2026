# Kubernetes Hands-On Lab 13 — Ingress Controllers & Ingress Resources

## 13.1 Objectives

By the end of this lab, you should be able to:

- Understand why Ingress exists (Layer 7 routing on top of Services)
- Install an Ingress Controller (NGINX Ingress Controller)
- Create an Ingress resource for path-based routing
- Create an Ingress resource for host-based (name-based virtual hosting) routing
- Understand the relationship: Ingress resource (rules) vs Ingress Controller (the actual proxy that implements them)
- Configure a default backend for unmatched requests
- Add TLS termination to an Ingress
- Troubleshoot an Ingress that returns 404/502 despite correct-looking rules
- Perform common Ingress tasks quickly for the CKA

## 13.2 Architecture

```
                     Client
                       |
                       v
              Ingress Controller
             (e.g. ingress-nginx,
              itself exposed via a
              NodePort/LoadBalancer
              Service)
                       |
        reads Ingress resources from
        the API server and configures
        its own routing rules
                       |
        +--------------+--------------+
        |                             |
        v                             v
   Ingress rule:                Ingress rule:
   host: shop.example.com       host: api.example.com
   path: /                      path: /v1
        |                             |
        v                             v
   Service: shop-svc             Service: api-svc
        |                             |
        v                             v
      Pods                          Pods
```

Key concept: an **Ingress resource** is just a set of routing rules (YAML) — it does nothing by itself. An **Ingress Controller** is the actual running proxy (NGINX, Traefik, HAProxy, cloud ALB, etc.) that watches Ingress resources and configures itself accordingly. Without a controller installed, creating Ingress YAML has zero effect — a very common CKA/interview trap.

## 13.3 Prerequisites

```bash
kubectl version --client
kubectl get nodes
```

Check whether an Ingress Controller is already installed:

```bash
kubectl get pods -n ingress-nginx
kubectl get ingressclass
```

## 13.4 Lab 1 — Install the NGINX Ingress Controller

For a cloud cluster:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

For kind:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

Wait for it to be ready:

```bash
kubectl get pods -n ingress-nginx -w
```

Verify the IngressClass was registered:

```bash
kubectl get ingressclass
```

You should see `nginx` listed — every Ingress resource must reference an `ingressClassName` so the right controller picks it up (useful if multiple controllers run in one cluster).

## 13.5 Lab 2 — Backing Services

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shop-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: shop-app
  template:
    metadata:
      labels:
        app: shop-app
    spec:
      containers:
        - name: app
          image: hashicorp/http-echo
          args: ["-text=Response from shop-app"]
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: shop-svc
spec:
  selector:
    app: shop-app
  ports:
    - port: 80
      targetPort: 5678
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-app
  template:
    metadata:
      labels:
        app: api-app
    spec:
      containers:
        - name: app
          image: hashicorp/http-echo
          args: ["-text=Response from api-app"]
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: api-svc
spec:
  selector:
    app: api-app
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
  name: shop-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: shop-app
  template:
    metadata:
      labels:
        app: shop-app
    spec:
      containers:
        - name: app
          image: hashicorp/http-echo
          args: ["-text=Response from shop-app"]
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: shop-svc
spec:
  selector:
    app: shop-app
  ports:
    - port: 80
      targetPort: 5678
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-app
  template:
    metadata:
      labels:
        app: api-app
    spec:
      containers:
        - name: app
          image: hashicorp/http-echo
          args: ["-text=Response from api-app"]
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: api-svc
spec:
  selector:
    app: api-app
  ports:
    - port: 80
      targetPort: 5678
EOF
```

## 13.6 Lab 3 — Path-Based Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /shop
            pathType: Prefix
            backend:
              service:
                name: shop-svc
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 80
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /shop
            pathType: Prefix
            backend:
              service:
                name: shop-svc
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 80
EOF
```

Check it:

```bash
kubectl get ingress path-based-ingress
kubectl describe ingress path-based-ingress
```

Get the Ingress Controller's external address:

```bash
kubectl get svc -n ingress-nginx
```

Test both paths (replace `<ingress-ip>` accordingly):

```bash
curl http://<ingress-ip>/shop
curl http://<ingress-ip>/api
```

## 13.7 Lab 4 — Host-Based Routing (Virtual Hosting)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: shop.example.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: shop-svc
                port:
                  number: 80
    - host: api.example.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 80
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: shop.example.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: shop-svc
                port:
                  number: 80
    - host: api.example.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 80
EOF
```

Test using the `Host` header (no real DNS needed for testing):

```bash
curl -H "Host: shop.example.local" http://<ingress-ip>/
curl -H "Host: api.example.local" http://<ingress-ip>/
```

## 13.8 TLS Termination

Generate a self-signed cert for testing:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=shop.example.local"

kubectl create secret tls shop-tls --cert=tls.crt --key=tls.key
```

Add TLS to the Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - shop.example.local
      secretName: shop-tls
  rules:
    - host: shop.example.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: shop-svc
                port:
                  number: 80
```

Apply and test:

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - shop.example.local
      secretName: shop-tls
  rules:
    - host: shop.example.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: shop-svc
                port:
                  number: 80
EOF

curl -k -H "Host: shop.example.local" https://<ingress-ip>/
```

## 13.9 Break It — Ingress With No Controller Reference

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: broken-ingress-no-class
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: shop-svc
                port:
                  number: 80
```

Apply it (note: no `ingressClassName`, and assume multiple controllers or a strict cluster policy where a default class isn't set):

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: broken-ingress-no-class
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: shop-svc
                port:
                  number: 80
EOF
```

## 13.10 Diagnose and Recover

```bash
kubectl get ingress broken-ingress-no-class
```

`ADDRESS` column may stay empty — no controller has claimed this Ingress.

```bash
kubectl describe ingress broken-ingress-no-class
```

Check if a default IngressClass exists:

```bash
kubectl get ingressclass -o jsonpath='{range .items[*]}{.metadata.name}{": default="}{.metadata.annotations.ingressclass\.kubernetes\.io/is-default-class}{"\n"}{end}'
```

Fix by explicitly setting the class:

```bash
kubectl patch ingress broken-ingress-no-class --type=merge -p '{"spec":{"ingressClassName":"nginx"}}'
kubectl get ingress broken-ingress-no-class
```

Also check the backend Service itself is healthy (a 502 often means the Ingress→Service wiring is fine but the Service has 0 endpoints — see Lab 11):

```bash
kubectl get endpoints shop-svc
```

## 13.11 CKA Practice Task

**Task**

1. Confirm an Ingress Controller and its `IngressClass` are present in the cluster.
2. Create Deployment/Service pair `blog-app` / `blog-svc` on port 80.
3. Create an Ingress named `blog-ingress` routing path `/blog` (Prefix) to `blog-svc`, explicitly setting `ingressClassName`.
4. Verify with `kubectl describe ingress` that the backend and rules are correct.
5. Curl the Ingress Controller's address with path `/blog` and confirm a response.

**Target Time**

8 minutes

Try it without looking at previous commands.

## 13.12 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

What is the difference between an Ingress resource and an Ingress Controller? What happens if you create Ingress YAML with no controller installed?

**Question 2**

What does `ingressClassName` do, and why does it matter in a cluster with multiple Ingress Controllers?

**Question 3**

A request to an Ingress path returns `502 Bad Gateway`. Is this more likely an Ingress Controller problem or a backend Service problem? How do you tell?

**Question 4**

What's the difference between path-based routing and host-based (virtual hosting) routing?

**Question 5**

What Kubernetes object type provides the certificate for TLS termination on an Ingress?

**Question 6**

If two Ingress resources define overlapping rules for the same host/path, which one wins, and why should you avoid this in practice?

## 13.13 Useful Commands

```bash
kubectl get ingressclass
kubectl get ingress
kubectl describe ingress <name>

kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx

kubectl get endpoints <backend-service>

kubectl create secret tls <name> --cert=<cert-file> --key=<key-file>

kubectl patch ingress <name> --type=merge -p '{"spec":{"ingressClassName":"<class>"}}'
```

## 13.14 Cleanup

```bash
kubectl delete ingress path-based-ingress host-based-ingress tls-ingress \
  broken-ingress-no-class blog-ingress

kubectl delete deployment shop-app api-app blog-app
kubectl delete svc shop-svc api-svc blog-svc
kubectl delete secret shop-tls

# Only if you installed it solely for this lab:
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

## 13.15 Lab Checklist

- [ ] Installed an Ingress Controller and confirmed its IngressClass
- [ ] Created a path-based routing Ingress
- [ ] Created a host-based (virtual hosting) Ingress
- [ ] Understood the Ingress resource vs Ingress Controller distinction
- [ ] Added TLS termination with a Secret
- [ ] Reproduced an Ingress with no effective controller/class
- [ ] Diagnosed and fixed the missing `ingressClassName`
- [ ] Distinguished an Ingress-level problem from a backend Service problem
- [ ] Completed the CKA task within 8 minutes
