# Kubernetes Hands-On Lab 25 — CRDs & Operators (Intro)

## 25.1 Objectives

By the end of this lab, you should be able to:

- Understand what a CustomResourceDefinition (CRD) is and why it extends the Kubernetes API
- Create a CRD and define its schema with OpenAPI validation
- Create, read, and manage Custom Resources (CRs) of your new type using standard `kubectl`
- Understand the relationship between a CRD (the schema) and an Operator (the controller that acts on it)
- Understand the reconciliation loop pattern that every controller/operator follows
- Install a real-world Operator and observe it manage resources on your behalf
- Troubleshoot a Custom Resource that's "stuck" because no controller is watching it
- Perform common CRD/Operator tasks quickly for the CKA

## 25.2 Architecture

```
                  CustomResourceDefinition (CRD)
              (registers a NEW resource type with
               the API server — e.g. "Website")
                            |
              kubectl can now do:
              kubectl get websites
              kubectl apply -f my-website.yaml
                            |
                            v
                  Custom Resource (CR)
              (an actual instance of your new type —
               just data in etcd, like any other object,
               UNLESS something is watching it)
                            |
                            v
                       Operator
              (a controller, usually running as a
               Deployment, that watches CRs of your
               type and RECONCILES real cluster state
               to match what the CR declares)

Reconciliation loop (the pattern EVERY controller follows,
including built-in ones like the Deployment controller):

    watch for changes -> compare desired vs actual state
         ^                                    |
         |                                    v
         +------ take action to converge -----+
```

Key concept: a CRD alone does **nothing** except let you store structured custom data in Kubernetes and validate its shape — `kubectl get pods` and `kubectl get websites` (a hypothetical CRD) behave identically at the API level. It's the **Operator** (a controller you write or install) that gives a Custom Resource actual behavior, the same way the built-in Deployment controller is what makes a Deployment object actually create Pods.

## 25.3 Prerequisites

```bash
kubectl version --client
kubectl get nodes
kubectl get crd
```

## 25.4 Lab 1 — Create a CRD

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: websites.example.com
spec:
  group: example.com
  names:
    kind: Website
    plural: websites
    singular: website
    shortNames:
      - ws
  scope: Namespaced
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                domain:
                  type: string
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 10
                image:
                  type: string
              required:
                - domain
                - image
      subresources:
        status: {}
      additionalPrinterColumns:
        - name: Domain
          type: string
          jsonPath: .spec.domain
        - name: Replicas
          type: integer
          jsonPath: .spec.replicas
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: websites.example.com
spec:
  group: example.com
  names:
    kind: Website
    plural: websites
    singular: website
    shortNames:
      - ws
  scope: Namespaced
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                domain:
                  type: string
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 10
                image:
                  type: string
              required:
                - domain
                - image
      subresources:
        status: {}
      additionalPrinterColumns:
        - name: Domain
          type: string
          jsonPath: .spec.domain
        - name: Replicas
          type: integer
          jsonPath: .spec.replicas
EOF
```

Verify:

```bash
kubectl get crd websites.example.com
kubectl explain website.spec
```

`kubectl explain` works on your custom type exactly like it does on built-in ones, because it reads the schema you just registered.

## 25.5 Lab 2 — Create Custom Resources

```yaml
apiVersion: example.com/v1
kind: Website
metadata:
  name: my-blog
spec:
  domain: blog.example.com
  replicas: 3
  image: nginx:1.27
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: example.com/v1
kind: Website
metadata:
  name: my-blog
spec:
  domain: blog.example.com
  replicas: 3
  image: nginx:1.27
EOF
```

Use it exactly like any built-in resource:

```bash
kubectl get websites
kubectl get ws
kubectl describe website my-blog
kubectl get website my-blog -o yaml
```

Notice the custom `Domain`/`Replicas` columns from `additionalPrinterColumns` showing in `kubectl get`.

## 25.6 Schema Validation in Action

Try creating an invalid Website (missing required `image`, and `replicas` out of range):

```bash
kubectl apply -f - <<EOF
apiVersion: example.com/v1
kind: Website
metadata:
  name: invalid-site
spec:
  domain: bad.example.com
  replicas: 50
EOF
```

Expected: the API server itself rejects this before it's even stored, citing the missing required field and the `maximum: 10` violation — this is the OpenAPI schema validation working exactly like built-in resource validation.

## 25.7 The Missing Piece — No Controller Yet

```bash
kubectl get pods -l app=my-blog
```

Nothing exists. Creating the `my-blog` Website CR did **not** create any Pods, Deployments, or Services — because we've only defined the *shape* of the data (the CRD), not any *behavior*. This is the critical distinction: without a controller/operator watching `Website` resources, they're just structured data sitting in etcd.

## 25.8 Lab 3 — Install a Real Operator

Cert-manager is a widely-used, production-grade Operator that manages TLS certificates via its own CRDs (`Certificate`, `Issuer`, `ClusterIssuer`):

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.3/cert-manager.yaml
```

Wait for its controller Pods to be ready:

```bash
kubectl get pods -n cert-manager -w
```

Confirm its CRDs were registered:

```bash
kubectl get crd | grep cert-manager.io
```

## 25.9 Observe the Reconciliation Loop in Action

Create a self-signed `ClusterIssuer` (cert-manager's simplest issuer type) and a `Certificate` CR:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: demo-cert
  namespace: default
spec:
  secretName: demo-cert-tls
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
  dnsNames:
    - demo.example.local
```

Apply it:

```bash
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: demo-cert
  namespace: default
spec:
  secretName: demo-cert-tls
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
  dnsNames:
    - demo.example.local
EOF
```

Unlike our `Website` CRD, cert-manager's controller IS watching — within seconds, it reconciles this `Certificate` CR into an actual Secret:

```bash
kubectl get certificate demo-cert -w
kubectl get secret demo-cert-tls
kubectl describe certificate demo-cert
```

Check the `Status:` block on the Certificate — this is the reconciliation loop's output: the controller continuously compares "what does this Certificate CR want" against "does a valid, non-expired cert Secret already exist," and creates/renews as needed, forever, without you doing anything further.

## 25.10 CRD vs Operator — The Complete Picture

| | CRD | Operator |
|---|---|---|
| What it is | A schema registration (API extension) | A controller (usually a running Deployment) |
| What it does alone | Lets you store/validate/query custom structured data | Nothing — it needs CRs to watch |
| What it needs to be useful | An Operator watching it | One or more CRDs defining what it manages |
| Analogy | A new database table + validation rules | The application code that reads/writes that table and takes action |

## 25.11 Break It — CR With a Missing/Uninstalled Operator

Simulate the scenario from 25.7 more explicitly — delete cert-manager's controller but leave its CRDs and CRs behind:

```bash
kubectl delete deployment -n cert-manager cert-manager cert-manager-webhook cert-manager-cainjector
```

Create a new Certificate CR:

```bash
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: orphaned-cert
  namespace: default
spec:
  secretName: orphaned-cert-tls
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
  dnsNames:
    - orphaned.example.local
EOF
```

## 25.12 Diagnose and Recover

```bash
kubectl get certificate orphaned-cert
kubectl get secret orphaned-cert-tls
```

The Certificate CR exists (it's just data — the API server happily stored it), but no Secret is ever created, and `Status:` never populates:

```bash
kubectl describe certificate orphaned-cert
```

Confirm the root cause — no controller Pods exist to do the reconciling:

```bash
kubectl get pods -n cert-manager
```

Fix by reinstalling the operator's controller components:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.3/cert-manager.yaml
kubectl get pods -n cert-manager -w
```

Once controllers are back, watch the previously-orphaned CR get reconciled automatically — no need to recreate it, since it was already sitting there as valid data waiting for a controller to notice it:

```bash
kubectl get certificate orphaned-cert -w
kubectl get secret orphaned-cert-tls
```

## 25.13 CKA Practice Task

**Task**

1. Create a CRD `backups.example.com` (kind `Backup`, scope `Namespaced`) with a schema requiring `spec.target` (string) and `spec.schedule` (string).
2. Create a `Backup` CR named `nightly-backup` with `target: postgres-db` and `schedule: "0 2 * * *"`.
3. Attempt to create an invalid `Backup` missing `target`, and confirm the API server rejects it.
4. Confirm `kubectl get backups` shows your resource, and explain (without applying anything else) why no actual backup job runs.
5. Check whether `cert-manager` is installed and confirm its CRDs are distinct from your own.

**Target Time**

8 minutes

Try it without looking at previous commands.

## 25.14 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

What does a CRD actually register with the API server, and what does creating one NOT give you?

**Question 2**

What is the reconciliation loop pattern, and can you name a built-in Kubernetes controller (from earlier labs) that follows it?

**Question 3**

If you `kubectl apply` a Custom Resource and nothing happens (no Pods, no Secrets, no side effects), what's the single most likely explanation?

**Question 4**

What validates that a Custom Resource's fields match the expected types/constraints, and at what point does that validation happen?

**Question 5**

Why does deleting an Operator's controller Deployment not delete the Custom Resources it was managing?

**Question 6**

What happens to orphaned Custom Resources (whose controller was deleted) once the controller is reinstalled — do you need to recreate the CRs?

## 25.15 Useful Commands

```bash
kubectl get crd
kubectl describe crd <name>
kubectl explain <kind>.spec

kubectl get <plural-name>
kubectl describe <kind> <name>
kubectl get <kind> <name> -o yaml

kubectl get pods -n <operator-namespace>
kubectl logs -n <operator-namespace> -l <operator-label-selector>
```

## 25.16 Cleanup

```bash
kubectl delete website my-blog invalid-site
kubectl delete crd websites.example.com backups.example.com

kubectl delete certificate demo-cert orphaned-cert
kubectl delete clusterissuer selfsigned-issuer
kubectl delete -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.3/cert-manager.yaml
```

## 25.17 Lab Checklist

- [ ] Created a CRD with OpenAPI schema validation
- [ ] Created, listed, and described a Custom Resource
- [ ] Confirmed the API server rejects a CR violating the schema
- [ ] Confirmed a CRD alone produces no behavior without a controller
- [ ] Installed a real-world Operator (cert-manager)
- [ ] Watched the Operator reconcile a Custom Resource into real cluster state (a Secret)
- [ ] Understood the CRD-vs-Operator relationship (schema vs behavior)
- [ ] Reproduced an orphaned CR with no controller watching it
- [ ] Diagnosed and fixed it by reinstalling the operator, without recreating the CR
- [ ] Completed the CKA task within 8 minutes
