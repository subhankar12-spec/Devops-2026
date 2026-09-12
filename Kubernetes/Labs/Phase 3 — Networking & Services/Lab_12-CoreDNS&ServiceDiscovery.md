# Kubernetes Hands-On Lab 12 — CoreDNS & Service Discovery

## 12.1 Objectives

By the end of this lab, you should be able to:

- Understand how CoreDNS enables service discovery inside a cluster
- Resolve a Service by short name, namespace-qualified name, and FQDN
- Understand the DNS naming pattern for Services and Pods
- Inspect and read the CoreDNS `Corefile` configuration
- Understand `dnsPolicy` and `dnsConfig` on a Pod
- Resolve a Pod's own DNS name via a Headless Service (recap/extension of Lab 10)
- Troubleshoot DNS resolution failures (CoreDNS down, wrong namespace, `ndots` confusion)
- Perform common DNS/service-discovery tasks quickly for the CKA

## 12.2 Architecture

```
                     Pod
                      |
           /etc/resolv.conf points to
           CoreDNS's ClusterIP (kube-dns Service)
                      |
                      v
                  CoreDNS Pods
                (kube-system namespace)
                      |
        watches the API server for
        Services & Endpoints, serves:
                      |
        <service>.<namespace>.svc.cluster.local  -> Service ClusterIP
        <pod-ip-dashed>.<namespace>.pod.cluster.local -> Pod IP
        <pod-hostname>.<headless-svc>.<namespace>.svc.cluster.local -> Pod IP
                                                       (only for Headless Services)
```

Key concept: every Pod's `/etc/resolv.conf` is automatically configured by the kubelet to point at CoreDNS, with a search list of namespace-qualified suffixes — this is *why* `curl my-service` (short name, no domain) works from within the same namespace, but resolving a Service in a different namespace needs at least `my-service.other-namespace`.

## 12.3 Prerequisites

```bash
kubectl version --client
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

## 12.4 Lab 1 — Set Up Test Resources

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lab12-a
---
apiVersion: v1
kind: Namespace
metadata:
  name: lab12-b
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: lab12-a
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: lab12-a
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 80
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: lab12-a
---
apiVersion: v1
kind: Namespace
metadata:
  name: lab12-b
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: lab12-a
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: lab12-a
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 80
EOF
```

## 12.5 Resolve a Service — Same Namespace

Run a debug Pod in `lab12-a`:

```bash
kubectl run dns-test-a -n lab12-a --image=busybox:1.36 --rm -it --restart=Never -- sh
```

Inside the shell, try all three forms:

```sh
nslookup backend-svc
nslookup backend-svc.lab12-a
nslookup backend-svc.lab12-a.svc.cluster.local
exit
```

All three should resolve to the same ClusterIP — because the Pod's search domains automatically expand the short name.

## 12.6 Resolve a Service — Cross Namespace

From a Pod in `lab12-b`, the short name will **not** work:

```bash
kubectl run dns-test-b -n lab12-b --image=busybox:1.36 --rm -it --restart=Never -- sh
```

```sh
nslookup backend-svc
```

Expected: `server can't find backend-svc: NXDOMAIN` — because `lab12-b`'s search list expands to `backend-svc.lab12-b.svc.cluster.local` first, which doesn't exist.

Now try the namespace-qualified form:

```sh
nslookup backend-svc.lab12-a
nslookup backend-svc.lab12-a.svc.cluster.local
exit
```

Both work — this is the #1 real-world DNS mistake: forgetting the namespace when calling a Service cross-namespace.

## 12.7 Inspect a Pod's resolv.conf

```bash
kubectl run resolv-check -n lab12-a --image=busybox:1.36 --rm -it --restart=Never -- \
  cat /etc/resolv.conf
```

You'll see something like:

```
nameserver 10.96.0.10
search lab12-a.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

`ndots:5` means: any name with fewer than 5 dots is tried against each `search` suffix, in order, before being tried as an absolute name — this is exactly why `backend-svc` (0 dots) gets expanded, while a fully qualified external domain like `example.com.` (with a trailing dot) is looked up directly.

## 12.8 Inspect CoreDNS Itself

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get svc -n kube-system kube-dns
```

View the CoreDNS configuration:

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

You'll see the `Corefile` — the plugin chain (`kubernetes`, `forward`, `cache`, `loop`, `reload`, etc.) that defines how CoreDNS resolves cluster-internal names and forwards everything else upstream.

Check CoreDNS logs (useful when troubleshooting resolution failures):

```bash
kubectl logs -n kube-system -l k8s-app=kube-dns
```

## 12.9 Pod DNS via Headless Service (Recap)

If you still have the `web`/`web-headless` StatefulSet from Lab 10, resolve an individual Pod:

```bash
kubectl run dns-test-pod --image=busybox:1.36 --rm -it --restart=Never -- \
  nslookup web-0.web-headless.default.svc.cluster.local
```

If not, recreate a minimal version:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: quick-headless
spec:
  clusterIP: None
  selector:
    app: quick-sts
  ports:
    - port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: quick
spec:
  serviceName: quick-headless
  replicas: 2
  selector:
    matchLabels:
      app: quick-sts
  template:
    metadata:
      labels:
        app: quick-sts
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
EOF

kubectl run dns-test-pod --image=busybox:1.36 --rm -it --restart=Never -- \
  nslookup quick-0.quick-headless.default.svc.cluster.local
```

Only Pods backed by a **Headless** Service get individual DNS names this way — a regular ClusterIP Service only resolves to the Service's own virtual IP, never to individual Pod IPs by name.

## 12.10 dnsPolicy and dnsConfig

Most Pods use the default:

```yaml
spec:
  dnsPolicy: ClusterFirst
```

| `dnsPolicy` | Behavior |
|---|---|
| `ClusterFirst` (default) | Use CoreDNS; non-cluster names fall through to upstream resolvers CoreDNS is configured with |
| `Default` | Use the Node's own `/etc/resolv.conf` — bypasses CoreDNS entirely |
| `None` | Ignore all of the above; you must fully specify `dnsConfig` yourself |
| `ClusterFirstWithHostNet` | Same as ClusterFirst, for Pods using `hostNetwork: true` |

Custom `dnsConfig` example (e.g. to add a search domain or a custom nameserver):

```yaml
spec:
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 10.96.0.10
    searches:
      - lab12-a.svc.cluster.local
    options:
      - name: ndots
        value: "2"
```

## 12.11 Break It — CoreDNS Scaled to Zero

Simulate a cluster-wide DNS outage (⚠️ disruptive — only do this on a lab/test cluster):

```bash
kubectl scale deployment coredns -n kube-system --replicas=0
```

Try resolving anything:

```bash
kubectl run dns-outage-test -n lab12-a --image=busybox:1.36 --rm -it --restart=Never -- \
  nslookup backend-svc
```

Expected: the lookup times out or fails entirely — every Service-name-based communication in the cluster breaks, even though the Services and Pods themselves are perfectly healthy. This is a critical production lesson: DNS failure looks like "everything is broken" even when nothing but DNS actually is.

## 12.12 Diagnose and Recover

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

Zero Pods — that's the smoking gun. Restore it:

```bash
kubectl scale deployment coredns -n kube-system --replicas=2
kubectl get pods -n kube-system -l k8s-app=kube-dns -w
```

Once Ready, confirm resolution works again:

```bash
kubectl run dns-recovery-test -n lab12-a --image=busybox:1.36 --rm -it --restart=Never -- \
  nslookup backend-svc
```

## 12.13 CKA Practice Task

**Task**

1. Create two namespaces: `cka-ns-a` and `cka-ns-b`.
2. In `cka-ns-a`, create a Deployment `payments` (image `nginx:1.28`) and a ClusterIP Service `payments-svc`.
3. From a debug Pod in `cka-ns-b`, demonstrate that `payments-svc` alone fails to resolve, but `payments-svc.cka-ns-a` succeeds.
4. Print the `search` line from `/etc/resolv.conf` inside a Pod in `cka-ns-a`.
5. Check how many CoreDNS Pods are currently running in `kube-system`.

**Target Time**

6 minutes

Try it without looking at previous commands.

## 12.14 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

Why does `nslookup my-svc` work from the same namespace but fail from a different one?

**Question 2**

What does `ndots:5` in `/etc/resolv.conf` actually control?

**Question 3**

What's the difference between how a regular ClusterIP Service and a Headless Service handle individual Pod DNS names?

**Question 4**

If every Service-based connection in the cluster suddenly starts timing out, but `kubectl get pods` shows everything Running, what's the first thing you check?

**Question 5**

What does `dnsPolicy: Default` actually mean, and why is it a confusing name?

**Question 6**

Where does CoreDNS forward DNS queries for names outside the cluster domain (e.g. `google.com`)?

## 12.15 Useful Commands

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get svc -n kube-system kube-dns
kubectl get configmap coredns -n kube-system -o yaml
kubectl logs -n kube-system -l k8s-app=kube-dns

kubectl run <name> --image=busybox:1.36 --rm -it --restart=Never -- nslookup <target>
kubectl run <name> --image=busybox:1.36 --rm -it --restart=Never -- cat /etc/resolv.conf

kubectl scale deployment coredns -n kube-system --replicas=<n>
```

## 12.16 Cleanup

```bash
kubectl delete namespace lab12-a lab12-b cka-ns-a cka-ns-b
kubectl delete statefulset quick
kubectl delete service quick-headless
```

## 12.17 Lab Checklist

- [ ] Resolved a Service by short name within the same namespace
- [ ] Confirmed short-name resolution fails cross-namespace
- [ ] Resolved a Service using the namespace-qualified and FQDN forms
- [ ] Inspected a Pod's `/etc/resolv.conf` and understood `search` and `ndots`
- [ ] Viewed the CoreDNS `Corefile` via its ConfigMap
- [ ] Resolved an individual Pod's DNS name via a Headless Service
- [ ] Understood the four `dnsPolicy` values and when to use `dnsConfig`
- [ ] Simulated a full CoreDNS outage and observed cluster-wide DNS failure
- [ ] Diagnosed the outage back to zero CoreDNS Pods
- [ ] Restored CoreDNS and confirmed resolution recovered
- [ ] Completed the CKA task within 6 minutes
