# Kubernetes Hands-On Lab 11 — Services (ClusterIP, NodePort, LoadBalancer)

## 11.1 Objectives

By the end of this lab, you should be able to:

- Create and use a `ClusterIP` Service (default, internal-only)
- Create and use a `NodePort` Service (external access via any Node's IP)
- Create and use a `LoadBalancer` Service and understand its cloud dependency
- Understand how a Service finds its Pods via label selector
- Inspect and understand Endpoints/EndpointSlices
- Understand `targetPort` vs `port` vs `nodePort`
- Understand kube-proxy's role in Service routing
- Troubleshoot a Service with 0 endpoints
- Perform common Service tasks quickly for the CKA

## 11.2 Architecture

```
                     Service
                (stable virtual IP)
                        |
              selector: app=web
                        |
        Endpoints / EndpointSlice
        (dynamically updated list
         of matching Pod IPs:ports)
                        |
              +---------+---------+
              |         |         |
              v         v         v
            Pod-1     Pod-2     Pod-3
          10.244.1.5 10.244.2.3 10.244.1.9

Service Types (increasing exposure):
  ClusterIP (default)  -> internal cluster traffic only
  NodePort              -> ClusterIP + a port opened on EVERY Node's IP
  LoadBalancer          -> NodePort + cloud provider provisions an external LB
```

Key concept: a Service is not a process or a Pod — it's a stable virtual IP plus iptables/IPVS rules that kube-proxy programs on every Node. Pods come and go and get new IPs constantly; the Service IP never changes, and its Endpoints list is what actually stays in sync with live, Ready Pods.

## 11.3 Prerequisites

```bash
kubectl version --client
kubectl get nodes
```

## 11.4 Lab 1 — Create Backing Pods

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
EOF

kubectl get pods -l app=web-app -o wide
```

## 11.5 Lab 2 — ClusterIP Service (Default)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-clusterip
spec:
  type: ClusterIP
  selector:
    app: web-app
  ports:
    - port: 80
      targetPort: 80
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: web-clusterip
spec:
  type: ClusterIP
  selector:
    app: web-app
  ports:
    - port: 80
      targetPort: 80
EOF
```

Check it:

```bash
kubectl get svc web-clusterip
kubectl describe svc web-clusterip
```

Note the `Endpoints:` line — it should list all 3 Pod IPs on port 80.

Test from inside the cluster (ClusterIP is not reachable from outside):

```bash
kubectl run curl-test --image=busybox:1.36 --rm -it --restart=Never -- \
  wget -qO- web-clusterip.default.svc.cluster.local
```

## 11.6 port vs targetPort vs nodePort

| Field | Meaning |
|---|---|
| `port` | Port the Service itself listens on (what other Pods/clients target) |
| `targetPort` | Port on the **container** that traffic gets forwarded to |
| `nodePort` | Port opened on **every Node's** external IP (NodePort/LoadBalancer only), range 30000–32767 by default |

If `targetPort` is omitted, it defaults to the same value as `port`.

## 11.7 Lab 3 — NodePort Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
EOF
```

Check it:

```bash
kubectl get svc web-nodeport
```

Get a Node's external/internal IP:

```bash
kubectl get nodes -o wide
```

Access it (from a machine that can reach the Node's IP, e.g. inside the same VPC/network):

```bash
curl http://<any-node-ip>:30080
```

Key point: a NodePort Service opens that port on **every** Node, not just the one running a matching Pod — kube-proxy routes the traffic to a healthy backend Pod wherever it lives.

## 11.8 Lab 4 — LoadBalancer Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
    - port: 80
      targetPort: 80
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: web-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
    - port: 80
      targetPort: 80
EOF
```

Check it:

```bash
kubectl get svc web-loadbalancer -w
```

On a real cloud cluster (EKS/GKE/AKS), `EXTERNAL-IP` transitions from `<pending>` to an actual IP/hostname once the cloud controller provisions a load balancer. On local clusters (kind/Minikube) with no cloud integration, it stays `<pending>` indefinitely — this is expected, not a bug, and important to recognize during troubleshooting so you don't chase a phantom issue.

## 11.9 Inspect Endpoints Directly

```bash
kubectl get endpoints web-clusterip
kubectl get endpointslices -l kubernetes.io/service-name=web-clusterip
```

Endpoints/EndpointSlices are the actual, continuously-updated list the Service uses for routing — this is the object to check whenever "the Service isn't working."

## 11.10 kube-proxy's Role

```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy
```

kube-proxy runs on every Node and programs iptables (or IPVS) rules that intercept traffic to a Service's ClusterIP and redirect it to one of the Endpoints — this is why Services work even though there's no single process actually "listening" on the Service IP.

## 11.11 Break It — Service With 0 Endpoints

```yaml
apiVersion: v1
kind: Service
metadata:
  name: broken-svc
spec:
  selector:
    app: web-app-typo
  ports:
    - port: 80
      targetPort: 80
```

Apply it (note the selector typo: `web-app-typo` instead of `web-app`):

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: broken-svc
spec:
  selector:
    app: web-app-typo
  ports:
    - port: 80
      targetPort: 80
EOF
```

## 11.12 Diagnose and Recover

```bash
kubectl get svc broken-svc
kubectl get endpoints broken-svc
```

`ENDPOINTS` column shows `<none>` — the Service exists but has nothing to route to.

```bash
kubectl describe svc broken-svc
```

Confirm the selector doesn't match any Pod's labels:

```bash
kubectl get pods --show-labels | grep web-app
```

Fix by correcting the selector:

```bash
kubectl patch svc broken-svc -p '{"spec":{"selector":{"app":"web-app"}}}'
kubectl get endpoints broken-svc
```

Endpoints should now populate with the 3 Pod IPs.

## 11.13 CKA Practice Task

**Task**

1. Create a Deployment named `api` with 2 replicas, image `nginx:1.28`, port `80`, label `app=api`.
2. Expose it with a ClusterIP Service named `api-svc` on port `8080` forwarding to container port `80`.
3. Verify `api-svc` has 2 endpoints.
4. Change `api-svc` to type `NodePort` with `nodePort: 30090` using `kubectl edit` or `kubectl patch`.
5. Create a broken Service `api-broken` with a mismatched selector, confirm it has 0 endpoints, then fix it.

**Target Time**

7 minutes

Try it without looking at previous commands.

## 11.14 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

What is the difference between `port`, `targetPort`, and `nodePort` on a Service spec?

**Question 2**

Why does a `LoadBalancer` type Service stay `<pending>` forever on a local cluster like kind or Minikube?

**Question 3**

A Service exists, Pods are Running, but `kubectl get endpoints <svc>` shows `<none>`. What's the most common cause and how do you confirm it?

**Question 4**

What component actually implements the traffic redirection from a Service's ClusterIP to a backend Pod?

**Question 5**

Is a NodePort opened only on the Node that's running a matching Pod, or on every Node? Why?

**Question 6**

What's the relationship between a Service and an EndpointSlice — which one is "live" and which is the stable abstraction?

## 11.15 Useful Commands

```bash
kubectl get svc
kubectl describe svc <name>
kubectl get endpoints <name>
kubectl get endpointslices -l kubernetes.io/service-name=<name>

kubectl expose deployment <name> --port=<port> --target-port=<port> --type=<type>

kubectl patch svc <name> -p '{"spec":{"selector":{"key":"value"}}}'
kubectl edit svc <name>

kubectl get nodes -o wide
```

## 11.16 Cleanup

```bash
kubectl delete deployment web-app api
kubectl delete svc web-clusterip web-nodeport web-loadbalancer broken-svc api-svc api-broken
```

## 11.17 Lab Checklist

- [ ] Created a ClusterIP Service and verified internal-only access
- [ ] Created a NodePort Service and understood every-Node port exposure
- [ ] Created a LoadBalancer Service and understood cloud-provider dependency
- [ ] Distinguished `port` vs `targetPort` vs `nodePort`
- [ ] Inspected Endpoints/EndpointSlices directly
- [ ] Understood kube-proxy's role in Service routing
- [ ] Reproduced a Service with 0 endpoints from a selector mismatch
- [ ] Diagnosed and fixed the selector
- [ ] Completed the CKA task within 7 minutes
