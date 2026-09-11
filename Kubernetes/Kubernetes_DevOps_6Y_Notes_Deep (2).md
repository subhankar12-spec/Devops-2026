# Kubernetes --- 6-Year DevOps Engineer Notes

> A practical Kubernetes reference for a DevOps engineer around the 5--6
> year level.
>
> **Goal:** understand Kubernetes deeply enough to design, deploy,
> operate, troubleshoot, secure, scale, and explain production workloads
> --- without getting lost in Kubernetes source-code internals.

------------------------------------------------------------------------

# 1. Kubernetes Mental Model

## 1.1 What Kubernetes actually does

Kubernetes is a platform for running containerized workloads and
continuously keeping them in the state you declared.

The most important idea is:

``` text
You declare desired state
        ↓
Kubernetes stores that state
        ↓
Controllers observe current state
        ↓
Controllers take actions
        ↓
Current state moves toward desired state
        ↓
Reconciliation continues continuously
```

For example:

``` yaml
spec:
  replicas: 3
```

does not mean:

> "Create three Pods once."

It means:

> "The desired state is three replicas. Keep trying to maintain that
> state."

If one Pod dies:

``` text
Desired = 3
Current = 2
        ↓
Deployment/ReplicaSet controller notices
        ↓
New Pod is created
        ↓
Current = 3
```

This is the **reconciliation model**.

------------------------------------------------------------------------

## 1.2 Desired state vs current state

Kubernetes objects generally contain:

``` yaml
metadata:
spec:
status:
```

### `metadata`

Identifies and describes the object.

Common fields:

-   `name`
-   `namespace`
-   `labels`
-   `annotations`
-   `uid`
-   `ownerReferences`
-   `resourceVersion`
-   `generation`

### `spec`

The desired configuration.

Example:

``` yaml
spec:
  replicas: 3
```

### `status`

What Kubernetes currently observes.

Example:

``` yaml
status:
  availableReplicas: 3
  readyReplicas: 3
```

A useful mental model:

``` text
spec   = what you want
status = what Kubernetes sees
```

------------------------------------------------------------------------

## 1.3 Declarative vs imperative

Imperative:

``` bash
kubectl create deployment nginx --image=nginx
```

You tell Kubernetes **what command/action to perform**.

Declarative:

``` bash
kubectl apply -f deployment.yaml
```

You provide the desired configuration and Kubernetes reconciles toward
it.

For production:

``` text
Git
 ↓
YAML/Helm
 ↓
review
 ↓
apply / GitOps
 ↓
Kubernetes
```

Declarative management is especially valuable because configuration
becomes reviewable, repeatable, and version-controlled.

------------------------------------------------------------------------

## 1.4 Kubernetes API objects

Most things you operate are API objects:

``` text
Pod
Deployment
Service
ConfigMap
Secret
PVC
Job
Node
Namespace
Ingress
...
```

You can inspect API resources with:

``` bash
kubectl api-resources
kubectl explain pod
kubectl explain deployment.spec
kubectl explain service.spec
```

This is extremely useful when learning YAML without memorizing every
field.

------------------------------------------------------------------------

# 2. Kubernetes Architecture

A Kubernetes cluster has two major areas:

``` text
Control Plane
    |
    +-- kube-apiserver
    +-- etcd
    +-- kube-scheduler
    +-- kube-controller-manager
    +-- cloud-controller-manager (where applicable)

Worker Node
    |
    +-- kubelet
    +-- container runtime
    +-- kube-proxy / equivalent datapath
    +-- Pods
```

The control plane makes decisions.

Worker nodes run workloads.

------------------------------------------------------------------------

# 3. API Server

The API server is the main entry point into the Kubernetes control
plane.

Typical requests:

``` text
kubectl
CI/CD
Argo CD
controllers
operators
cloud integrations
```

communicate with:

``` text
kube-apiserver
```

The API server validates requests and coordinates access to cluster
state.

------------------------------------------------------------------------

## 3.1 Simplified request path

A useful operational model is:

``` text
Client
  ↓
Authentication
  ↓
Authorization
  ↓
Admission
  ↓
Validation / processing
  ↓
Persistence in etcd
```

### Authentication

"Who are you?"

Examples:

-   certificate
-   OIDC
-   ServiceAccount token
-   cloud identity integration

### Authorization

"What is this identity allowed to do?"

Examples:

-   RBAC
-   other authorization mechanisms

### Admission

"Even if you are authenticated and authorized, should this object be
accepted?"

Examples:

-   Pod Security admission
-   validating webhooks
-   mutating webhooks
-   policy engines

This distinction is critical in interviews and troubleshooting.

------------------------------------------------------------------------

# 4. etcd

`etcd` is the distributed key-value store containing Kubernetes cluster
state.

Conceptually:

``` text
API Server
    ↓
  etcd
    ↓
cluster state
```

Examples of state include:

-   Kubernetes objects
-   configuration
-   metadata
-   desired state

Do not think of etcd as the place where container logs or application
data normally live.

------------------------------------------------------------------------

## 4.1 etcd quorum

For a highly available etcd cluster, quorum matters.

For:

``` text
3 members → quorum = 2
5 members → quorum = 3
```

Formula:

``` text
quorum = floor(N/2) + 1
```

The purpose is to prevent split-brain and preserve consistency.

For production HA control planes, etcd health and latency are critical.

------------------------------------------------------------------------

## 4.2 etcd backup

A Kubernetes backup strategy must include more than application YAML.

At minimum, understand:

``` text
etcd snapshot
+
application data
+
persistent volumes
+
cluster configuration
+
external dependencies
```

An etcd snapshot can restore Kubernetes control-plane state, but it does
not magically restore the contents of every application's persistent
storage.

------------------------------------------------------------------------

## 4.3 Do not edit etcd directly

Normal operational flow:

``` text
kubectl / API client
       ↓
API server
       ↓
etcd
```

Do not modify etcd directly as a normal administration technique.

------------------------------------------------------------------------

# 5. kube-scheduler

The scheduler decides which node should run an unscheduled Pod.

Simplified flow:

``` text
Pod created
   ↓
scheduler sees Pod without node assignment
   ↓
find feasible nodes
   ↓
score suitable nodes
   ↓
select node
   ↓
bind Pod to node
```

Scheduling considers things such as:

-   resource requests
-   node selectors
-   node affinity
-   pod affinity/anti-affinity
-   taints/tolerations
-   topology constraints
-   volumes
-   node availability
-   scheduling policies

------------------------------------------------------------------------

## 5.1 Why a Pod can remain Pending

Common reasons:

``` text
insufficient CPU
insufficient memory
node selector mismatch
required affinity mismatch
taint without matching toleration
volume topology issue
resource quota
```

Start with:

``` bash
kubectl describe pod <pod>
kubectl get events --sort-by=.lastTimestamp
```

The scheduler's events often tell you why no node is suitable.

------------------------------------------------------------------------

# 6. Controller Manager

Kubernetes uses controllers to continuously reconcile desired and
observed state.

Examples:

-   Deployment controller
-   ReplicaSet controller
-   Job controller
-   Node controller
-   Namespace controller
-   StatefulSet controller
-   DaemonSet controller

Example:

``` text
Deployment
   ↓
ReplicaSet
   ↓
Pods
```

The Deployment controller does not manually "run" your application. It
manages the objects needed to maintain the requested state.

------------------------------------------------------------------------

# 7. kubelet

`kubelet` is the primary node agent.

It runs on worker nodes.

Conceptually:

``` text
API Server
    ↓
kubelet
    ↓
CRI
    ↓
containerd
    ↓
OCI runtime
    ↓
container
```

The kubelet:

-   watches Pod specifications assigned to its node
-   works with the container runtime
-   manages containers
-   performs health checks
-   reports node/Pod status
-   mounts volumes
-   handles Pod lifecycle

------------------------------------------------------------------------

# 8. Container Runtime, CRI and containerd

Kubernetes does not directly call Docker to start every container.

The important abstraction is **CRI --- Container Runtime Interface**.

Common runtime stack:

``` text
Kubernetes
    ↓
kubelet
    ↓
CRI
    ↓
containerd
    ↓
runc / another OCI runtime
    ↓
container
```

Docker can still be part of a developer workflow:

``` text
Docker build
    ↓
image
    ↓
registry
    ↓
Kubernetes pulls image
```

But Docker Engine itself is not required for a modern Kubernetes node.

------------------------------------------------------------------------

# 9. Pods

A Pod is the smallest deployable unit in Kubernetes.

A Pod can contain:

``` text
one container
```

or:

``` text
multiple tightly coupled containers
```

Containers inside one Pod share:

-   network namespace
-   Pod IP
-   localhost
-   optionally volumes

Example:

``` text
Pod
 |
 +-- application container
 |
 +-- sidecar container
```

Containers can communicate using:

``` text
localhost:<port>
```

because they share the Pod network namespace.

------------------------------------------------------------------------

## 9.1 Why Pods are not usually permanent

Pod IPs are ephemeral.

If a Pod is recreated:

``` text
old Pod IP = 10.244.1.10
new Pod IP = 10.244.1.17
```

Therefore clients should normally not depend directly on Pod IPs.

Instead:

``` text
Client
  ↓
Service
  ↓
Pod
```

------------------------------------------------------------------------

# 10. Pod Lifecycle

Typical phases include:

``` text
Pending
Running
Succeeded
Failed
Unknown
```

A Pod can also be restarting containers even while the Pod remains in
`Running`.

Important distinction:

``` text
Pod phase
≠
container state
```

Inspect:

``` bash
kubectl get pod <pod>
kubectl describe pod <pod>
kubectl get pod <pod> -o yaml
```

------------------------------------------------------------------------

# 11. Restart Policy

Pod restart policies include:

-   `Always`
-   `OnFailure`
-   `Never`

Deployments normally use:

``` yaml
restartPolicy: Always
```

Jobs typically use:

``` yaml
restartPolicy: Never
```

or:

``` yaml
restartPolicy: OnFailure
```

Do not confuse:

``` text
container restart
```

with:

``` text
Pod replacement
```

A controller may replace a Pod entirely when required.

------------------------------------------------------------------------

# 12. Init Containers

Init containers run before application containers.

Example:

``` yaml
initContainers:
  - name: init
    image: busybox
    command: ["sh", "-c", "echo preparing"]
```

Useful for:

-   initialization
-   dependency preparation
-   generating configuration
-   permission preparation
-   waiting for a prerequisite

They must complete successfully before normal containers start.

------------------------------------------------------------------------

# 13. Sidecars

A sidecar is a helper container running alongside the main application
container.

Common concepts:

``` text
application
+
logging agent
```

or:

``` text
application
+
proxy
```

or:

``` text
application
+
configuration helper
```

Sidecars are useful when the helper must share the application's Pod
context.

Do not automatically add sidecars. They increase operational complexity.

------------------------------------------------------------------------

# 14. Deployment

A Deployment manages stateless application replicas.

Relationship:

``` text
Deployment
    ↓
ReplicaSet
    ↓
Pods
```

Example:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

------------------------------------------------------------------------

# 15. Deployment Rolling Updates

When you change the Pod template:

``` yaml
image: nginx:1.28
```

the Deployment creates a new ReplicaSet.

Conceptually:

``` text
Old ReplicaSet
   ↓
old Pods

New ReplicaSet
   ↓
new Pods
```

Kubernetes gradually replaces old Pods according to rollout strategy.

Check:

``` bash
kubectl rollout status deployment/nginx
kubectl rollout history deployment/nginx
kubectl get rs
```

------------------------------------------------------------------------

## 15.1 maxSurge and maxUnavailable

Example:

``` yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

Meaning:

-   `maxSurge`: how many extra Pods can temporarily exist
-   `maxUnavailable`: how many desired Pods may be unavailable during
    rollout

For a critical application:

``` text
maxUnavailable = 0
```

may reduce disruption, but requires enough capacity.

------------------------------------------------------------------------

## 15.2 Rollback

If a new version is bad:

``` bash
kubectl rollout undo deployment/nginx
```

Then:

``` bash
kubectl rollout status deployment/nginx
```

Rollback is one reason Deployment revisions matter operationally.

------------------------------------------------------------------------

# 16. ReplicaSet

A ReplicaSet maintains a requested number of matching Pods.

Example:

``` text
replicas = 3
```

If one disappears:

``` text
desired = 3
current = 2
```

ReplicaSet creates another.

In normal application management, you usually manage:

``` text
Deployment
```

rather than creating ReplicaSets manually.

------------------------------------------------------------------------

# 17. StatefulSet

StatefulSet is designed for workloads needing stable identity.

Important properties:

-   stable ordinal identity
-   stable network identity when paired with suitable Services
-   persistent volume templates
-   ordered lifecycle behavior depending on configuration

Example identities:

``` text
db-0
db-1
db-2
```

This is different from Deployment Pods:

``` text
nginx-7d8...
nginx-6f2...
```

StatefulSets are common for:

-   databases
-   clustered systems
-   quorum-based systems
-   systems where identity matters

A StatefulSet does **not** magically make an application stateful or
distributed correctly. The application itself must support the required
storage/replication semantics.

------------------------------------------------------------------------

# 18. Headless Service + StatefulSet

A common pattern:

``` text
StatefulSet
   +
headless Service
```

A headless Service uses:

``` yaml
clusterIP: None
```

This allows DNS records to represent individual Pods.

Conceptually:

``` text
db-0
db-1
db-2
```

can be discoverable individually through DNS.

This is important for clustered applications where nodes must know each
other's stable identities.

------------------------------------------------------------------------

# 19. DaemonSet

DaemonSet ensures a Pod runs on appropriate nodes.

Common examples:

-   node log collectors
-   monitoring agents
-   security agents
-   networking components

Conceptually:

``` text
Node 1 → agent
Node 2 → agent
Node 3 → agent
```

When a new eligible node joins:

``` text
new node → DaemonSet Pod
```

------------------------------------------------------------------------

# 20. Jobs

A Job is for finite work.

Example:

``` text
run migration
process batch
generate report
```

Unlike a Deployment:

``` text
Deployment → continuously maintain replicas
Job        → complete a task
```

Important fields include:

-   `completions`
-   `parallelism`
-   `backoffLimit`
-   `activeDeadlineSeconds`
-   `ttlSecondsAfterFinished`

------------------------------------------------------------------------

# 21. CronJobs

CronJob creates Jobs on a schedule.

Example:

``` yaml
schedule: "0 2 * * *"
```

Operational concerns:

-   what happens if a previous Job is still running?
-   how many historical Jobs should remain?
-   what happens after missed schedules?
-   is the task idempotent?

Concurrency policy is especially important:

``` yaml
concurrencyPolicy: Forbid
```

can prevent overlapping Jobs.

------------------------------------------------------------------------

# 22. Labels and Selectors

Labels are key-value metadata:

``` yaml
labels:
  app: payment
  environment: production
```

Selectors identify objects.

Example Service:

``` yaml
selector:
  app: payment
```

The Service finds matching Pods.

A broken selector can cause:

``` text
Service exists
Pods exist
but Service has no endpoints
```

This is one of the most common Kubernetes operational mistakes.

------------------------------------------------------------------------

# 23. Services

Pods are ephemeral, so clients need a stable endpoint.

A Service provides a stable virtual networking abstraction over a set of
Pods.

Basic model:

``` text
Client
  ↓
Service
  ↓
selected Pods
```

------------------------------------------------------------------------

# 24. Service Ports

Understand these three fields:

``` yaml
ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
```

### `port`

The Service port.

### `targetPort`

The port on the selected Pod.

### `nodePort`

The node-level port used by a NodePort Service.

Example:

``` text
client
  ↓
Service:80
  ↓
Pod:8080
```

------------------------------------------------------------------------

# 25. ClusterIP

Default Service type:

``` yaml
type: ClusterIP
```

Accessible inside the cluster.

Example:

``` text
frontend Pod
   ↓
http://backend:8080
   ↓
backend Service
   ↓
backend Pods
```

------------------------------------------------------------------------

# 26. NodePort

NodePort exposes a Service on a port on cluster nodes.

Conceptually:

``` text
client
  ↓
NodeIP:NodePort
  ↓
Service
  ↓
Pod
```

Example:

``` yaml
type: NodePort
ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

Useful for labs and some architectures, but normally a production
external entry point is often an Ingress/Gateway or cloud LoadBalancer.

------------------------------------------------------------------------

# 27. LoadBalancer

On cloud platforms, a LoadBalancer Service can integrate with cloud
load-balancing infrastructure.

Conceptually:

``` text
Internet
   ↓
Cloud Load Balancer
   ↓
Kubernetes Service
   ↓
Pods
```

The exact implementation depends on the cloud/provider integration.

------------------------------------------------------------------------

# 28. Headless Services

A headless Service:

``` yaml
clusterIP: None
```

does not provide the usual virtual ClusterIP behavior.

Instead it is commonly used for direct endpoint discovery through DNS.

Typical use:

``` text
StatefulSet
+
headless Service
```

------------------------------------------------------------------------

# 29. EndpointSlices

Kubernetes represents Service backends using EndpointSlices.

Conceptually:

``` text
Service
  ↓
EndpointSlices
  ↓
Pod IPs / endpoints
```

This scales better than a single giant endpoint object.

Inspect:

``` bash
kubectl get endpointslice
kubectl describe endpointslice <name>
```

When debugging Services, checking EndpointSlices is often more useful
than only checking the Service object.

------------------------------------------------------------------------

# 30. Service Traffic Flow

A simplified path:

``` text
Client Pod
   ↓
Service DNS
   ↓
Service virtual IP
   ↓
node networking datapath
   ↓
selected Pod IP
```

The exact implementation depends on the Kubernetes networking stack.

Traditional clusters may use kube-proxy with iptables or IPVS.

Some modern environments use eBPF-based datapaths.

As a DevOps engineer, understand the purpose and troubleshooting
implications rather than memorizing kernel rules.

------------------------------------------------------------------------

# 31. kube-proxy

kube-proxy historically implements Service networking on nodes.

It watches Service and endpoint information and programs the node's
networking rules.

Common modes include:

``` text
iptables
IPVS
```

Modern Kubernetes networking solutions may replace or supplement this
behavior with eBPF.

Important point:

``` text
Service virtual IP
≠
real Pod IP
```

The datapath translates/routes Service traffic toward backend endpoints.

------------------------------------------------------------------------

# 32. Session Affinity

Services can optionally prefer sending traffic from a client to the same
backend.

Example:

``` yaml
sessionAffinity: ClientIP
```

This can help applications that incorrectly depend on session locality,
but it should not replace proper stateless design or external session
storage when appropriate.

------------------------------------------------------------------------

# 33. externalTrafficPolicy

For NodePort/LoadBalancer scenarios:

``` yaml
externalTrafficPolicy: Local
```

can preserve source client IP in situations where traffic would
otherwise be source-NATed.

Trade-offs include:

-   source IP preservation
-   uneven traffic
-   node-local endpoint requirements

Understand the traffic path before selecting it.

------------------------------------------------------------------------

# 34. Kubernetes DNS

CoreDNS normally provides cluster DNS.

Typical Service DNS:

``` text
<service>.<namespace>.svc.cluster.local
```

Example:

``` text
backend.default.svc.cluster.local
```

Within the same namespace, applications often use:

``` text
backend
```

DNS debugging:

``` bash
kubectl get pods -n kube-system
kubectl get svc -n kube-system
kubectl logs -n kube-system -l k8s-app=kube-dns
```

From a test Pod:

``` bash
nslookup backend
```

or:

``` bash
getent hosts backend
```

------------------------------------------------------------------------

# 35. Kubernetes Networking Model

A healthy Kubernetes networking implementation generally provides:

``` text
Pod → Pod
Pod → Service
Pod → external destination
Node → Pod
```

The exact implementation is supplied by the cluster's networking stack.

------------------------------------------------------------------------

# 36. CNI

CNI stands for Container Network Interface.

The CNI layer is responsible for configuring networking for Pods.

A simplified view:

``` text
kubelet
  ↓
container runtime
  ↓
CNI
  ↓
Pod network setup
```

Typical CNI implementations include networking solutions such as:

-   Calico
-   Cilium
-   Flannel
-   cloud-provider networking implementations

Do not assume every cluster uses the same datapath.

------------------------------------------------------------------------

# 37. Pod Network Namespaces

At a practical Linux level, Pods use Linux networking primitives.

A simplified model is:

``` text
Pod network namespace
        |
       veth
        |
host networking
```

The exact topology depends on the CNI.

You do not need kernel-source knowledge for most DevOps work, but
understanding namespaces, interfaces, routes, and IP addresses makes
Kubernetes troubleshooting much easier.

------------------------------------------------------------------------

# 38. NetworkPolicy

NetworkPolicy controls allowed network communication for Pods.

Example idea:

``` text
frontend → backend = allowed
frontend → database = denied
```

A policy commonly selects Pods and defines allowed ingress/egress.

Important:

**NetworkPolicy only works if the network implementation
supports/enforces it.**

A policy object existing in the API does not guarantee that traffic is
actually being filtered if the underlying network stack does not enforce
it.

------------------------------------------------------------------------

# 39. Ingress

Ingress provides HTTP/HTTPS routing into cluster Services.

Typical model:

``` text
Internet
   ↓
Load Balancer
   ↓
Ingress Controller
   ↓
Ingress rules
   ↓
Service
   ↓
Pods
```

Example routing:

``` text
example.com/api → api-service
example.com/web → web-service
```

Ingress is an API object; the actual traffic handling requires an
Ingress Controller.

------------------------------------------------------------------------

# 40. Ingress TLS

A common configuration:

``` yaml
tls:
  - hosts:
      - example.com
    secretName: example-tls
```

The controller terminates TLS using the certificate stored in the
referenced Secret.

Operationally verify:

-   DNS
-   LoadBalancer
-   controller
-   certificate
-   Secret
-   Ingress rule
-   backend Service
-   backend endpoints

------------------------------------------------------------------------

# 41. Gateway API

Gateway API is a newer, more expressive Kubernetes networking API.

Conceptually:

``` text
GatewayClass
    ↓
Gateway
    ↓
HTTPRoute / other routes
    ↓
Service
    ↓
Pods
```

Ingress remains important, but Gateway API is increasingly relevant for
more structured traffic management.

------------------------------------------------------------------------

# 42. ConfigMaps

ConfigMap stores non-sensitive configuration.

Example:

``` yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
  APP_MODE: "production"
```

Can be consumed as:

-   environment variables
-   files
-   configuration data

Do not use ConfigMaps for passwords or credentials.

------------------------------------------------------------------------

# 43. Secrets

Secrets are intended for sensitive configuration.

Example:

``` yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: <base64-value>
  password: <base64-value>
```

Critical point:

``` text
base64 encoding ≠ encryption
```

Treat Kubernetes Secrets as sensitive.

For production, understand:

-   encryption at rest
-   RBAC restrictions
-   external secret managers
-   secret rotation
-   avoiding secrets in Git

------------------------------------------------------------------------

# 44. ConfigMap and Secret Update Behavior

When configuration is mounted as files, Kubernetes can update projected
data in the Pod over time.

Environment variables generally do not magically update inside an
already-running process.

Therefore applications may require:

``` text
Pod restart
```

to consume changed environment-based configuration.

GitOps systems may also trigger rollouts using checksum annotations when
configuration changes.

------------------------------------------------------------------------

# 45. Volumes

Pod containers have ephemeral writable layers.

If a container is recreated, data written only there may disappear.

Kubernetes volumes provide storage semantics beyond a single container
filesystem.

Common types/concepts:

``` text
emptyDir
hostPath
PersistentVolume
PersistentVolumeClaim
StorageClass
CSI
```

------------------------------------------------------------------------

# 46. emptyDir

`emptyDir` is created when the Pod is assigned to a node.

Containers in the Pod can share it.

Example:

``` yaml
volumes:
  - name: shared
    emptyDir: {}
```

Use cases:

-   temporary files
-   shared scratch space
-   container-to-container file exchange
-   temporary caches

When the Pod is removed from the node, the `emptyDir` data is lost.

------------------------------------------------------------------------

# 47. PersistentVolume

A PersistentVolume represents storage made available to the cluster.

Conceptually:

``` text
Storage backend
      ↓
PersistentVolume
      ↓
PersistentVolumeClaim
      ↓
Pod
```

In modern cloud environments, PVs are often dynamically provisioned
rather than manually created.

------------------------------------------------------------------------

# 48. PersistentVolumeClaim

A PVC is a workload's request for storage.

Example:

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

The application normally references the PVC, not the physical storage
backend directly.

------------------------------------------------------------------------

# 49. StorageClass

StorageClass defines how storage should be dynamically provisioned.

Conceptually:

``` text
PVC
 ↓
StorageClass
 ↓
CSI provisioner
 ↓
Storage backend
 ↓
PV
```

This is one of the most important production storage concepts.

------------------------------------------------------------------------

# 50. CSI

CSI stands for Container Storage Interface.

It standardizes storage integration.

A CSI implementation typically has:

``` text
controller components
+
node components
```

The controller handles operations such as provisioning/attachment, while
node components perform node-side volume operations.

The exact architecture varies by CSI driver.

------------------------------------------------------------------------

# 51. Access Modes

Common access modes:

``` text
ReadWriteOnce
ReadOnlyMany
ReadWriteMany
```

The actual supported modes depend on the storage implementation.

Do not assume:

``` text
ReadWriteOnce = one Pod globally
```

The precise semantics depend on the storage system and attachment
behavior.

------------------------------------------------------------------------

# 52. Reclaim Policy

PV reclaim behavior can include:

``` text
Retain
Delete
```

Understand what happens to the underlying storage after the PVC is
deleted.

For important production data, blindly using deletion semantics can be
dangerous.

------------------------------------------------------------------------

# 53. Storage Expansion

Some StorageClasses support volume expansion.

Typical workflow:

``` text
PVC 10Gi
   ↓
change request to 20Gi
   ↓
CSI/storage system expands
   ↓
filesystem may also require expansion
```

Always verify provider and filesystem support.

------------------------------------------------------------------------

# 54. Requests and Limits

Requests influence scheduling.

``` yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
```

Limits constrain resource usage.

``` yaml
resources:
  limits:
    cpu: "1"
    memory: "1Gi"
```

Important distinction:

``` text
request → scheduling reservation/accounting
limit   → upper bound enforced according to resource semantics
```

------------------------------------------------------------------------

# 55. CPU vs Memory Limits

CPU is compressible.

If a container reaches its CPU limit, it may be throttled.

Memory is not compressible in the same way.

If a container exceeds its memory limit, it can be killed, commonly
resulting in:

``` text
OOMKilled
```

This distinction is essential for troubleshooting.

------------------------------------------------------------------------

# 56. Kubernetes QoS Classes

Kubernetes can classify Pods into QoS classes such as:

``` text
Guaranteed
Burstable
BestEffort
```

A simplified understanding:

### Guaranteed

Appropriate requests and limits are specified consistently for
containers.

### Burstable

Some resources have requests/limits but not in the strict Guaranteed
pattern.

### BestEffort

No requests or limits are specified.

QoS influences eviction behavior under node resource pressure.

------------------------------------------------------------------------

# 57. LimitRange

LimitRange can establish default or maximum resource settings inside a
namespace.

Useful for preventing workloads from being deployed with completely
uncontrolled resource definitions.

------------------------------------------------------------------------

# 58. ResourceQuota

ResourceQuota limits aggregate namespace consumption.

Examples:

``` text
CPU
memory
Pods
Services
PVCs
Secrets
```

Useful in multi-team clusters.

------------------------------------------------------------------------

# 59. Scheduling --- nodeSelector

Simple scheduling constraint:

``` yaml
nodeSelector:
  disktype: ssd
```

Pod can only schedule on nodes matching the label.

Inspect labels:

``` bash
kubectl get nodes --show-labels
```

------------------------------------------------------------------------

# 60. Node Affinity

Node affinity provides more expressive placement rules.

Example concept:

``` yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      ...
```

Important distinction:

``` text
required → must satisfy
preferred → preference, not absolute requirement
```

The phrase:

``` text
IgnoredDuringExecution
```

means a change after scheduling does not automatically evict the
already-running Pod merely because the affinity condition changed.

------------------------------------------------------------------------

# 61. Pod Affinity and Anti-Affinity

These rules consider other Pods.

Example:

``` text
put frontend close to cache
```

or:

``` text
do not place replicas on the same failure domain
```

Anti-affinity can improve resilience but may make scheduling harder.

------------------------------------------------------------------------

# 62. Taints and Tolerations

Taint:

``` text
node says:
"don't schedule arbitrary Pods here"
```

Toleration:

``` text
Pod says:
"I can tolerate that taint"
```

Common taint effects:

``` text
NoSchedule
PreferNoSchedule
NoExecute
```

`NoExecute` can affect already-running Pods that do not tolerate the
taint.

Typical use:

``` text
dedicated nodes
control-plane nodes
special hardware
GPU nodes
```

------------------------------------------------------------------------

# 63. Topology Spread Constraints

Topology spread helps distribute replicas across failure domains.

Possible topology keys may represent:

``` text
zone
region
hostname
```

Goal:

``` text
replica 1 → node A / zone 1
replica 2 → node B / zone 2
replica 3 → node C / zone 3
```

This is often more predictable for resilience than relying only on
anti-affinity.

------------------------------------------------------------------------

# 64. Node Maintenance

Before maintenance:

``` bash
kubectl cordon <node>
```

This prevents new normal workload scheduling.

Then:

``` bash
kubectl drain <node>
```

Drain attempts to evict workloads safely.

After maintenance:

``` bash
kubectl uncordon <node>
```

Always understand exceptions involving:

-   DaemonSets
-   unmanaged Pods
-   local storage
-   PodDisruptionBudgets

Do not blindly run `--force` in production.

------------------------------------------------------------------------

# 65. Pod Disruption Budget

PDB controls voluntary disruptions.

Example:

``` yaml
spec:
  minAvailable: 2
```

If you have three replicas:

``` text
3 running
2 must remain available
```

PDB helps protect workloads during:

-   node drain
-   cluster maintenance
-   voluntary disruptions

PDB does not prevent every possible failure. It does not make a cluster
immune to node crashes or application bugs.

------------------------------------------------------------------------

# 66. Graceful Shutdown

When Kubernetes terminates a Pod:

``` text
termination requested
        ↓
Pod enters termination
        ↓
termination lifecycle hooks / shutdown handling
        ↓
SIGTERM to containers
        ↓
application cleanup
        ↓
grace period expires if still running
        ↓
SIGKILL
```

Relevant settings:

``` yaml
terminationGracePeriodSeconds: 30
```

Applications should handle `SIGTERM` correctly.

For HTTP applications, graceful shutdown should allow:

``` text
stop accepting new work
finish in-flight requests
close connections
exit
```

------------------------------------------------------------------------

# 67. Probes

Three major probe types:

``` text
startupProbe
readinessProbe
livenessProbe
```

------------------------------------------------------------------------

## 67.1 Startup Probe

Useful for slow-starting applications.

While startup probe has not succeeded, liveness/readiness behavior is
coordinated around startup handling.

Use it to prevent Kubernetes from killing a slow-starting application
too early.

------------------------------------------------------------------------

## 67.2 Readiness Probe

Answers:

> "Should this Pod receive traffic?"

If readiness fails:

``` text
Pod may remain Running
but is removed from Service endpoints
```

This is a traffic-routing concept, not merely a process-health concept.

------------------------------------------------------------------------

## 67.3 Liveness Probe

Answers:

> "Is the container stuck/unhealthy enough that it should be restarted?"

Bad liveness probes can cause restart loops.

Do not use liveness simply as:

``` text
application returned non-200 once
```

unless that behavior genuinely indicates unrecoverable application
health.

------------------------------------------------------------------------

# 68. HPA

Horizontal Pod Autoscaler changes replica count.

Example:

``` text
CPU rises
   ↓
HPA calculates desired replicas
   ↓
Deployment replicas increase
```

Common resource metric:

``` text
CPU utilization
```

HPA can also work with custom/external metrics depending on the metrics
pipeline.

------------------------------------------------------------------------

# 69. Metrics Server

Metrics Server provides resource metrics used by mechanisms such as HPA
and commands like:

``` bash
kubectl top nodes
kubectl top pods
```

It is not a full monitoring system.

For detailed historical observability, use a monitoring stack such as
Prometheus/Grafana or equivalent.

------------------------------------------------------------------------

# 70. HPA Operational Considerations

Autoscaling is not simply:

``` text
CPU > 70% → add Pod
```

There are calculations, timing, stabilization, and desired replica
behavior involved.

Consider:

-   correct resource requests
-   application startup time
-   readiness probes
-   workload burstiness
-   scale-up/scale-down behavior
-   stabilization
-   cluster capacity

If the cluster has no spare node capacity, HPA may request more Pods
while those Pods remain Pending.

That is why:

``` text
HPA
+
Cluster Autoscaler / node autoscaling
```

often work together.

------------------------------------------------------------------------

# 71. VPA

Vertical Pod Autoscaler adjusts resource requests/limits based on
observed usage and recommendations.

Concept:

``` text
workload
 ↓
observe usage
 ↓
recommend / adjust resources
```

VPA and HPA can conflict if both use the same resource signal
carelessly. Understand the scaling strategy before combining them.

------------------------------------------------------------------------

# 72. Cluster Autoscaling

Cluster autoscaling changes node capacity.

Conceptually:

``` text
Pending Pods due to insufficient capacity
          ↓
node autoscaler
          ↓
new node
          ↓
scheduler places Pods
```

On cloud platforms, this integrates with compute provisioning.

------------------------------------------------------------------------

# 73. Karpenter / Cloud Node Provisioning

On AWS and similar cloud environments, newer node-provisioning
approaches can dynamically provision compute based on workload
requirements.

For an EKS-focused engineer, understand the distinction:

``` text
Pod autoscaling
    vs
Node autoscaling
    vs
Node provisioning
```

These solve different problems.

------------------------------------------------------------------------

# 74. RBAC

RBAC = Role-Based Access Control.

Four core objects:

``` text
Role
ClusterRole
RoleBinding
ClusterRoleBinding
```

A useful mental model:

``` text
Identity
   +
Permissions
   +
Binding
```

------------------------------------------------------------------------

# 75. Authentication vs Authorization

Do not mix them.

Authentication:

``` text
Who are you?
```

Authorization:

``` text
What can you do?
```

Example:

``` text
User authenticates successfully
        ↓
RBAC checks permission
        ↓
allowed / denied
```

A valid kubeconfig does not mean the identity has permission to perform
every action.

------------------------------------------------------------------------

# 76. Role

Role grants permissions inside one namespace.

Example:

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

------------------------------------------------------------------------

# 77. RoleBinding

RoleBinding connects an identity to a Role.

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev
subjects:
  - kind: ServiceAccount
    name: app-sa
    namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

------------------------------------------------------------------------

# 78. ClusterRole

ClusterRole can represent cluster-scoped permissions or reusable
permissions.

A ClusterRole can also be bound into a namespace using a RoleBinding.

This distinction is commonly misunderstood.

------------------------------------------------------------------------

# 79. ClusterRoleBinding

ClusterRoleBinding grants the referenced ClusterRole at cluster scope.

Use carefully.

Example:

``` text
ClusterRoleBinding
   ↓
broad permissions
   ↓
identity
```

A careless ClusterRoleBinding can effectively create cluster-admin-like
exposure.

------------------------------------------------------------------------

# 80. RBAC Verbs

Common verbs:

``` text
get
list
watch
create
update
patch
delete
deletecollection
```

A read-only application might need:

``` text
get/list/watch
```

It usually should not receive:

``` text
create/update/delete
```

unless necessary.

------------------------------------------------------------------------

# 81. kubectl auth can-i

One of the most useful RBAC troubleshooting commands:

``` bash
kubectl auth can-i get pods
kubectl auth can-i create deployment
kubectl auth can-i get secrets
```

For another identity:

``` bash
kubectl auth can-i get pods \
  --as=system:serviceaccount:dev:app-sa \
  -n dev
```

Use this before guessing about RBAC.

------------------------------------------------------------------------

# 82. ServiceAccounts

Pods can run under a ServiceAccount.

Example:

``` yaml
serviceAccountName: app-sa
```

This identity can be granted RBAC permissions.

Do not give an application:

``` text
cluster-admin
```

just because it needs access to one API resource.

Use least privilege.

------------------------------------------------------------------------

# 83. ServiceAccount Tokens

Modern Kubernetes environments use projected/short-lived ServiceAccount
tokens rather than treating long-lived static tokens as the default
pattern.

Applications should receive only the permissions they need.

If an application does not need Kubernetes API access, consider:

``` yaml
automountServiceAccountToken: false
```

where appropriate.

------------------------------------------------------------------------

# 84. SecurityContext

SecurityContext controls security-related settings for Pods/containers.

Examples:

``` yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
```

These controls reduce attack surface.

------------------------------------------------------------------------

# 85. Linux Capabilities

Linux capabilities split traditionally powerful root privileges into
smaller capability sets.

Avoid unnecessary capabilities.

A common hardened pattern is:

``` yaml
capabilities:
  drop:
    - ALL
```

and add back only what is actually required.

------------------------------------------------------------------------

# 86. Seccomp

Seccomp can restrict system calls available to a process.

A modern baseline often uses a runtime/default seccomp profile where
supported.

The purpose is:

``` text
reduce kernel attack surface
```

You do not need to memorize individual syscalls for normal DevOps work.

------------------------------------------------------------------------

# 87. Pod Security Standards

Pod Security Standards define security profiles commonly described as:

``` text
Privileged
Baseline
Restricted
```

The Restricted profile is the strongest of these standard profiles.

Use Pod Security admission/policy mechanisms to prevent insecure
workload configurations from entering namespaces.

------------------------------------------------------------------------

# 88. Admission Controllers and Webhooks

Admission happens after authentication/authorization and before
persistence.

Two important webhook types:

``` text
MutatingWebhook
ValidatingWebhook
```

Mutating:

``` text
modify object
```

Validating:

``` text
allow / reject object
```

Examples of real-world use:

-   inject sidecars
-   enforce security rules
-   validate image policies
-   add labels
-   enforce organization standards

Webhook failures can block deployments, so treat admission components as
critical infrastructure.

------------------------------------------------------------------------

# 89. Finalizers

Finalizers delay deletion until required cleanup has happened.

Conceptually:

``` text
delete requested
    ↓
finalizer exists
    ↓
cleanup controller performs work
    ↓
finalizer removed
    ↓
object fully deleted
```

A stuck finalizer can cause:

``` text
Terminating
```

objects to remain indefinitely.

Do not remove finalizers blindly. First understand what cleanup they are
protecting.

------------------------------------------------------------------------

# 90. ownerReferences

Kubernetes objects can declare ownership relationships.

Example:

``` text
Deployment
   ↓ owner
ReplicaSet
   ↓ owner
Pod
```

This helps Kubernetes understand which objects belong together.

It also supports garbage collection.

------------------------------------------------------------------------

# 91. Garbage Collection

When an owner is deleted, dependent objects may be garbage-collected
depending on ownership and deletion policy.

This is why deleting a Deployment can eventually result in its
ReplicaSet/Pods disappearing.

Be careful when manipulating owner references manually.

------------------------------------------------------------------------

# 92. Custom Resource Definitions

CRDs extend the Kubernetes API with custom resource types.

Example conceptual resource:

``` text
Database
```

could become a Kubernetes API object:

``` yaml
apiVersion: example.com/v1
kind: Database
```

CRDs allow platforms to model domain-specific resources using Kubernetes
API conventions.

------------------------------------------------------------------------

# 93. Operators

An Operator combines:

``` text
CRD
+
controller/reconciliation logic
```

Example:

``` text
Database CR
    ↓
Operator
    ↓
Pods / Services / PVCs / configuration
```

The Operator continuously reconciles the custom resource.

This is the same core Kubernetes pattern:

``` text
desired state → reconciliation
```

------------------------------------------------------------------------

# 94. Helm

Helm is a Kubernetes package/release management tool.

A Helm chart packages Kubernetes manifests using templates and values.

Conceptually:

``` text
Chart
 +
values
 ↓
templates
 ↓
rendered manifests
 ↓
Kubernetes
```

Useful commands:

``` bash
helm repo add <repo> <url>
helm search repo <name>
helm install <release> <chart>
helm upgrade <release> <chart>
helm rollback <release> <revision>
helm list
helm uninstall <release>
helm show values <chart>
helm get values <release>
helm get manifest <release>
helm history <release>
```

## 94.1 Real chart structure

```text
mychart/
├── Chart.yaml
├── values.yaml
├── values-dev.yaml
├── values-prod.yaml
├── charts/                # subchart dependencies (.tgz or unpacked)
├── templates/
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── serviceaccount.yaml
│   ├── NOTES.txt
│   └── tests/
│       └── test-connection.yaml
└── .helmignore
```

`Chart.yaml`:

``` yaml
apiVersion: v2
name: mychart
description: A production microservice chart
type: application
version: 1.4.0          # chart version
appVersion: "2.3.1"     # application version
dependencies:
  - name: redis
    version: "18.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

`templates/_helpers.tpl` (shared naming logic, avoids repetition):

``` yaml
{{- define "mychart.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "mychart.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion }}
{{- end -}}
```

`templates/deployment.yaml` using the helper and values:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app.kubernetes.io/instance: {{ .Release.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

## 94.2 Hooks

Helm hooks run at specific points in a release lifecycle
(`pre-install`, `post-install`, `pre-upgrade`, `post-upgrade`,
`pre-delete`, `post-delete`, `pre-rollback`, `post-rollback`):

``` yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: myapp:{{ .Values.image.tag }}
          command: ["./migrate.sh"]
      restartPolicy: Never
```

Common real use: running a DB migration Job before the new app version
rolls out. If the migration Job fails, `helm upgrade` reports failure
and the release stops before Pods are updated — this is a common
interview question ("how do you handle schema migrations with Helm?").

------------------------------------------------------------------------

# 95. Helm Values

A chart often exposes configurable values:

``` yaml
replicaCount: 3

image:
  repository: nginx
  tag: "1.27"
```

Render before applying:

``` bash
helm template myapp ./chart
```

Debug:

``` bash
helm lint ./chart
helm template myapp ./chart -f values-prod.yaml --debug
```

## 95.1 Values layering and precedence

Helm merges values in this order, **later wins**:

``` text
chart's values.yaml (defaults)
   ↓ overridden by
-f values-prod.yaml (environment file, can pass multiple -f)
   ↓ overridden by
--set key=value (highest precedence, good for CI-injected values like image tag)
```

Real deployment command in a CI/CD pipeline:

``` bash
helm upgrade --install myapp ./chart \
  -f values.yaml \
  -f values-prod.yaml \
  --set image.tag=$CI_COMMIT_SHA \
  --namespace prod \
  --atomic \
  --timeout 5m \
  --wait
```

`--atomic` rolls back automatically on failure. `--wait` blocks until
Pods are Ready. `--timeout` prevents CI pipelines hanging forever on a
stuck rollout.

## 95.2 Diffing before applying

The `helm-diff` plugin shows exactly what will change against the live
cluster, which is critical before touching production:

``` bash
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade myapp ./chart -f values-prod.yaml
```

Treat an unreviewed `helm upgrade` in production the same way you'd
treat an unreviewed `terraform apply` — always diff first.

------------------------------------------------------------------------

# 96. GitOps and Argo CD

A typical GitOps model:

``` text
Developer
   ↓
Git
   ↓
Argo CD
   ↓
Kubernetes API
   ↓
Cluster
```

Argo CD continuously compares:

``` text
Git desired state
vs
cluster live state
```

and reconciles drift according to its configuration.

Important distinction:

``` text
CI → builds/tests/packages
CD/GitOps → delivers/reconciles
```

The exact split depends on the organization's workflow.

## 96.1 Argo CD Application manifest

``` yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://bitbucket.org/myorg/myapp-charts.git
    targetRevision: main
    path: charts/myapp
    helm:
      valueFiles:
        - values-prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true       # deletes resources removed from Git
      selfHeal: true     # reverts manual cluster drift back to Git state
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 10s
        factor: 2
        maxDuration: 3m
```

## 96.2 App of Apps pattern

For managing many services from one root Application:

``` text
root Application (argocd/apps.git)
   ├── Application: myapp-frontend
   ├── Application: myapp-backend
   ├── Application: myapp-worker
   └── Application: shared-infra (ingress-nginx, cert-manager, etc.)
```

Each child Application points to its own chart/path, but Argo CD
manages the whole tree from a single Git source of truth.

## 96.3 Sync waves (ordering dependencies)

``` yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"   # applies before wave "0"
```

Use this when, for example, a `Namespace` or `CustomResourceDefinition`
must exist before the Deployment that depends on it is applied.

## 96.4 Common Argo CD failure modes

``` text
OutOfSync forever
  → often caused by a mutating webhook or controller (e.g. HPA
    writing back .spec.replicas) fighting with Git's declared value
  → fix: ignoreDifferences on that field, or let HPA own replicas

Stuck in Progressing
  → readiness probe failing, so the Deployment never reports healthy
  → check kubectl rollout status + kubectl describe pod

Sync succeeds but app still serves old version
  → image tag not actually changed in Git (mutable tag like "latest")
  → fix: use immutable tags (commit SHA) so Git diff always reflects reality

selfHeal reverting an emergency manual kubectl fix
  → expected behavior; the emergency fix must go into Git, not kubectl,
    or selfHeal must be paused first with:
    argocd app set myapp-prod --sync-policy none
```

------------------------------------------------------------------------

# 97. Terraform + Kubernetes

Terraform is commonly used for infrastructure.

Example:

``` text
Terraform
   ↓
VPC
subnets
IAM
EKS
load balancer infrastructure
```

Kubernetes/Helm/GitOps then manages workloads.

Avoid unnecessary overlap where two systems continuously fight over the
same resource.

A clean ownership model is important.

## 97.1 Remote state backend (real setup)

``` hcl
terraform {
  backend "s3" {
    bucket         = "myorg-terraform-state"
    key            = "eks/prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"   # prevents concurrent apply
    encrypt        = true
  }
}
```

The DynamoDB table provides state locking — without it, two engineers
(or two CI pipelines) running `terraform apply` at once can corrupt
state. Losing/forgetting the lock table is a classic real incident.

## 97.2 EKS module usage

``` hcl
module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  version         = "~> 20.0"
  cluster_name    = "prod-cluster"
  cluster_version = "1.30"
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnets

  eks_managed_node_groups = {
    general = {
      instance_types = ["m5.large"]
      min_size       = 2
      max_size       = 6
      desired_size   = 3
    }
  }

  enable_irsa = true
}

provider "kubernetes" {
  host                   = module.eks.cluster_endpoint
  cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)
  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    args        = ["eks", "get-token", "--cluster-name", module.eks.cluster_name]
  }
}
```

## 97.3 Common Terraform + Kubernetes failure modes

``` text
State drift after manual kubectl/helm changes
  → terraform plan shows unexpected diffs; someone changed the live
    object outside Terraform. Fix by importing the real state or
    reverting the manual change, not by forcing apply blindly.

terraform destroy hangs on EKS
  → often a LoadBalancer Service or PVC created finalizers/dependent
    AWS resources (ELB, EBS volume) that Terraform doesn't know about.
    Delete the Kubernetes objects first, then destroy infra.

Provider version mismatch after a team member upgrades
  → lock provider versions in a committed .terraform.lock.hcl file.

Two systems fighting over the same object
  → e.g. Terraform manages a Kubernetes Deployment AND Argo CD also
    manages it from Git. Pick one owner: infra (VPC/EKS/IAM) via
    Terraform, workloads via GitOps. Do not let both manage workloads.

Local state used in a team (no backend configured)
  → guaranteed to eventually corrupt/conflict; always use a remote
    backend with locking for anything beyond a personal sandbox.
```

------------------------------------------------------------------------

# 98. CI/CD + Kubernetes

A mature deployment flow may look like:

``` text
Developer
   ↓
Git
   ↓
CI
   ├── test
   ├── build
   ├── scan
   └── push image
          ↓
     container registry
          ↓
      Git update
          ↓
       Argo CD
          ↓
      Kubernetes
```

This creates separation between:

``` text
artifact creation
```

and:

``` text
deployment reconciliation
```

## 98.1 Example Jenkins pipeline (build → scan → push → update Git)

``` groovy
pipeline {
  agent any
  environment {
    ECR_REPO = "123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp"
    IMAGE_TAG = "${env.GIT_COMMIT.take(7)}"
  }
  stages {
    stage('Build') {
      steps { sh "docker build -t $ECR_REPO:$IMAGE_TAG ." }
    }
    stage('Scan') {
      steps { sh "trivy image --exit-code 1 --severity CRITICAL $ECR_REPO:$IMAGE_TAG" }
    }
    stage('Push') {
      steps {
        sh "aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_REPO"
        sh "docker push $ECR_REPO:$IMAGE_TAG"
      }
    }
    stage('Update GitOps repo') {
      steps {
        sh """
          git clone https://bitbucket.org/myorg/myapp-charts.git
          cd myapp-charts
          yq -i '.image.tag = "${IMAGE_TAG}"' values-prod.yaml
          git commit -am "Deploy ${IMAGE_TAG}"
          git push
        """
      }
    }
  }
}
```

Note the pipeline never runs `kubectl apply` or `helm upgrade` itself —
it only updates the Git repo that Argo CD watches. This is the
practical meaning of "CI builds, GitOps deploys."

## 98.2 Immutable image tagging

``` text
BAD:  image: myapp:latest
GOOD: image: myapp:a1b2c3d      (git commit SHA)
GOOD: image: myapp:2024-06-01-3 (date + build number)
```

Mutable tags break rollback (`kubectl rollout undo` and Argo CD diffing
both rely on the tag actually changing between versions) and break
reproducibility during incident investigation.

------------------------------------------------------------------------

# 99. Docker vs Kubernetes

Docker and Kubernetes solve different layers.

Docker commonly handles:

``` text
build images
run containers
local development
```

Kubernetes handles:

``` text
orchestration
scheduling
service discovery
scaling
rollouts
self-healing
configuration
secrets
storage abstractions
```

A Kubernetes cluster can use containerd or another CRI-compatible
runtime without Docker Engine.

------------------------------------------------------------------------

# 100. Kubernetes on AWS / EKS

EKS is AWS's managed Kubernetes offering.

AWS can manage much of the Kubernetes control-plane infrastructure while
you operate workloads and node/compute configuration according to the
chosen architecture.

Common AWS integrations include:

``` text
VPC
IAM
Load Balancers
EBS
EFS
Route 53
CloudWatch
ECR
KMS
```

For a DevOps engineer, understand where responsibility sits:

``` text
AWS-managed
vs
customer-managed
```

This is the core managed-Kubernetes mindset.

## 100.1 IRSA (IAM Roles for Service Accounts)

Lets a Pod assume an AWS IAM role without static credentials.

``` text
ServiceAccount (annotated with IAM role ARN)
        ↓
EKS OIDC provider issues a projected token for the Pod
        ↓
AWS STS AssumeRoleWithWebIdentity
        ↓
Pod gets temporary AWS credentials
```

``` yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: prod
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/myapp-s3-access
```

``` yaml
spec:
  serviceAccountName: myapp-sa
  containers:
    - name: myapp
      image: myapp:a1b2c3d
```

Common real failure: token expires and isn't refreshed because an
old SDK version doesn't support the projected service account token
volume — the Pod gets `AccessDenied` after running fine for ~1 hour.
Check `aws-sdk` version and that
`AWS_WEB_IDENTITY_TOKEN_FILE`/`AWS_ROLE_ARN` env vars are actually
injected (`kubectl exec <pod> -- env | grep AWS`).

## 100.2 VPC CNI and IP exhaustion

The AWS VPC CNI assigns each Pod a real VPC IP from the subnet.

``` text
Symptom: new Pods stuck in ContainerCreating
kubectl describe pod shows:
  "failed to assign an IP address to container"
```

Root causes:

``` text
subnet ran out of free IPs (common in small /24 subnets with many
  small nodes, since ENI IP allocation is per-node, not per-cluster)
ENI IP warm pool exhausted on that instance type
too many small Pods packed onto instances with low max-IPs-per-ENI
```

Fixes:

``` text
use larger subnets (/20 or bigger) for node subnets
enable prefix delegation (ENABLE_PREFIX_DELEGATION=true) to get many
  more IPs per ENI on nitro instance types
spread nodes across more subnets/AZs
right-size instance types vs Pod density
```

## 100.3 AWS Load Balancer Controller (ALB/NLB Ingress)

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...:certificate/xxxx
    alb.ingress.kubernetes.io/healthcheck-path: /healthz
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80
```

Common real issue: ALB provisions but returns 502s. Sequence to
check: `target-type: ip` requires the CNI to allow direct Pod IP
targeting; targets show "unhealthy" in the AWS console when the
`healthcheck-path` doesn't match an actual app route, or when a
`NetworkPolicy` blocks traffic from the ALB's ENIs.

## 100.4 Managed node groups vs Fargate vs Karpenter

``` text
EKS Managed Node Groups
  + simplest, AWS handles AMI/lifecycle rolling updates
  - slower to scale (ASG launch time), coarse-grained sizing

Fargate profiles
  + no node management at all, pay per Pod
  - no DaemonSets, limited to specific launch configs, higher per-Pod cost
    at sustained load, some CNI/networking features unavailable

Karpenter
  + fast, right-sized, bin-packing aware provisioning; reacts to
    unschedulable Pods directly instead of waiting on ASG scaling
  - more setup complexity, needs careful NodePool/disruption config
```

A realistic production pattern: Karpenter or managed node groups for
general workloads, Fargate for isolated/low-traffic namespaces (e.g.
internal tooling) where operational simplicity matters more than cost
efficiency.

## 100.5 EKS add-ons and upgrades

``` bash
aws eks list-addons --cluster-name prod-cluster
aws eks describe-addon --cluster-name prod-cluster --addon-name vpc-cni
aws eks update-addon --cluster-name prod-cluster --addon-name vpc-cni --addon-version v1.18.0-eksbuild.1
```

Upgrade order that avoids version-skew incidents:

``` text
1. Upgrade control plane (aws eks update-cluster-version)
2. Upgrade core add-ons (vpc-cni, kube-proxy, coredns) to versions
   compatible with the new control-plane version
3. Upgrade node groups (new AMI/launch template)
4. Validate workloads, then repeat for the next minor version
```

Kubernetes only supports skipping one minor version at a time for
in-place upgrades — never jump multiple minors in one step.

------------------------------------------------------------------------

# 101. Kubernetes Observability

Think in three major signals:

``` text
Metrics
Logs
Traces
```

Also use:

``` text
Events
```

and Kubernetes object state.

------------------------------------------------------------------------

## 101.1 Metrics

Metrics answer:

``` text
How much?
How often?
How many?
How long?
```

Examples:

-   CPU
-   memory
-   request rate
-   error rate
-   latency
-   Pod restarts
-   node utilization

Common ecosystem:

``` text
Prometheus
Grafana
kube-state-metrics
Metrics Server
```

These solve different problems.

------------------------------------------------------------------------

## 101.2 Logs

Logs answer:

``` text
What happened?
```

Kubernetes commonly exposes container logs through:

``` bash
kubectl logs
```

Production platforms often aggregate logs into systems such as:

``` text
Loki
Elasticsearch/OpenSearch
Cloud logging services
```

------------------------------------------------------------------------

## 101.3 Traces

Distributed tracing answers:

``` text
Where did this request spend time?
Which service caused the delay?
```

OpenTelemetry is an important modern observability standard/ecosystem.

------------------------------------------------------------------------

# 102. Events

Events are extremely useful during incidents.

Commands:

``` bash
kubectl get events --sort-by=.lastTimestamp
```

or:

``` bash
kubectl describe pod <pod>
```

Events can reveal:

``` text
FailedScheduling
FailedMount
BackOff
Failed
Unhealthy
Pulling
Pulled
```

Treat events as a timeline clue, not a permanent application log.

------------------------------------------------------------------------

# 103. Kubernetes Troubleshooting Method

Do not randomly run commands.

Use a hierarchy:

``` text
1. Is the object present?
2. Is it in the expected state?
3. What does describe say?
4. What do events say?
5. Are dependencies healthy?
6. Are endpoints correct?
7. Are networking/DNS paths working?
8. Are node resources healthy?
9. Are control-plane components healthy?
10. What changed recently?
```

This prevents command memorization from replacing diagnosis.

------------------------------------------------------------------------

# 104. Troubleshooting: Pod Pending

Start:

``` bash
kubectl get pod <pod>
kubectl describe pod <pod>
kubectl get events --sort-by=.lastTimestamp
```

Look for:

``` text
Insufficient cpu
Insufficient memory
node selector mismatch
taint
affinity
quota
volume constraints
```

Then:

``` bash
kubectl get nodes
kubectl describe node <node>
```

A senior diagnosis should explain **why no node is feasible**, not just
say "Pod is Pending."

------------------------------------------------------------------------

# 105. Troubleshooting: CrashLoopBackOff

`CrashLoopBackOff` generally means:

``` text
container starts
 ↓
container exits/fails
 ↓
Kubernetes restarts it
 ↓
backoff increases
```

Commands:

``` bash
kubectl logs <pod>
kubectl logs <pod> --previous
kubectl describe pod <pod>
```

Check:

-   application error
-   wrong command/args
-   missing environment variable
-   missing Secret/ConfigMap
-   permission issue
-   dependency failure
-   failed probe
-   OOMKilled

`--previous` is especially valuable because the current container may
have restarted already.

------------------------------------------------------------------------

# 106. Troubleshooting: ImagePullBackOff

Check:

``` bash
kubectl describe pod <pod>
```

Typical causes:

``` text
wrong image name
wrong tag
private registry authentication
network/DNS problem
registry outage
image unavailable
```

Check:

``` bash
kubectl get secret
```

and:

``` yaml
imagePullSecrets:
```

when private registry authentication is required.

------------------------------------------------------------------------

# 107. Troubleshooting: OOMKilled

Check:

``` bash
kubectl describe pod <pod>
kubectl get pod <pod> -o json
```

Look for:

``` text
reason: OOMKilled
```

Then investigate:

``` text
memory request
memory limit
actual application usage
heap configuration
traffic/load
memory leak
node memory pressure
```

Do not solve every OOM by blindly increasing the limit. Determine
whether the application has abnormal memory behavior.

------------------------------------------------------------------------

# 108. Troubleshooting: Service Not Working

Use this order:

``` bash
kubectl get svc
kubectl describe svc <service>
kubectl get endpointslice
kubectl get pods --show-labels
```

Most common issue:

``` text
Service selector
        ≠
Pod labels
```

If EndpointSlices have no healthy endpoints, the Service cannot route
traffic to the expected Pods.

Then test from inside the cluster:

``` bash
kubectl run tmp --rm -it --image=curlimages/curl -- sh
```

and:

``` bash
curl http://service-name:port
```

------------------------------------------------------------------------

# 109. Troubleshooting: DNS

Check CoreDNS:

``` bash
kubectl get pods -n kube-system
kubectl logs -n kube-system -l k8s-app=kube-dns
```

From a test Pod:

``` bash
nslookup kubernetes.default
nslookup <service>.<namespace>
```

Then distinguish:

``` text
DNS failure
vs
Service failure
vs
application failure
```

A DNS name resolving successfully does not prove the application is
healthy.

------------------------------------------------------------------------

# 110. Troubleshooting: Storage

If a Pod is stuck because of a volume:

``` bash
kubectl describe pod <pod>
kubectl get pvc
kubectl describe pvc <pvc>
kubectl get pv
kubectl get storageclass
```

Look for:

``` text
Pending PVC
failed provisioning
attachment failure
mount failure
access mode mismatch
storage topology
CSI driver issue
```

Then inspect CSI controller/node components.

------------------------------------------------------------------------

# 111. Troubleshooting: Node NotReady

Start:

``` bash
kubectl get nodes
kubectl describe node <node>
```

Then on the node:

``` bash
systemctl status kubelet
journalctl -u kubelet --no-pager
```

Check:

``` text
CPU
memory
disk
network
container runtime
kubelet
certificates
CNI
```

Node conditions such as:

``` text
MemoryPressure
DiskPressure
PIDPressure
Ready
```

can explain why scheduling or workload behavior changed.

------------------------------------------------------------------------

# 112. Troubleshooting: Container Runtime

Check:

``` bash
systemctl status containerd
journalctl -u containerd --no-pager
```

Then:

``` bash
crictl info
crictl ps
crictl images
```

if `crictl` is configured.

Understand the chain:

``` text
Pod spec
 ↓
kubelet
 ↓
CRI
 ↓
containerd
 ↓
OCI runtime
```

This lets you identify which layer is failing.

------------------------------------------------------------------------

# 113. Troubleshooting: kubelet

Useful commands:

``` bash
systemctl status kubelet
journalctl -u kubelet -f
```

Check:

-   API connectivity
-   certificates
-   container runtime
-   CNI
-   disk
-   memory
-   static Pods
-   kubelet configuration

Do not restart kubelet repeatedly without reading its logs.

------------------------------------------------------------------------

# 114. Troubleshooting: Control Plane

For kubeadm-style clusters, inspect:

``` bash
kubectl get nodes
kubectl get pods -n kube-system
```

Control-plane components may run as static Pods.

Common checks:

``` bash
crictl ps
journalctl -u kubelet
```

and inspect static Pod manifests where appropriate.

If API server is unavailable, `kubectl` itself may not work, so
node-level diagnostics become important.

------------------------------------------------------------------------

# 115. Static Pods

Static Pods are managed directly by kubelet from configured manifests
rather than through the normal API-server-driven workload controller
flow.

In kubeadm clusters, control-plane components are commonly deployed as
static Pods.

This is why:

``` text
kube-apiserver
etcd
kube-scheduler
kube-controller-manager
```

may appear as Pods in:

``` text
kube-system
```

while their definitions are maintained on the control-plane node.

------------------------------------------------------------------------

# 116. kubeconfig

A kubeconfig contains information needed to connect to clusters.

Conceptually:

``` text
cluster
+
user/credentials
+
context
```

Inspect:

``` bash
kubectl config view
kubectl config get-contexts
kubectl config current-context
```

Switch:

``` bash
kubectl config use-context <context>
```

Avoid accidentally running production commands against the wrong
context.

A useful habit:

``` bash
kubectl config current-context
```

before destructive work.

------------------------------------------------------------------------

# 117. Authentication

Kubernetes authentication identifies callers.

Common approaches include:

``` text
client certificates
OIDC
ServiceAccounts
cloud IAM integrations
webhook authentication
```

Authentication answers:

``` text
Who is this?
```

It does not grant permission by itself.

------------------------------------------------------------------------

# 118. Authorization

Authorization decides whether an authenticated identity can perform an
operation.

Example:

``` text
GET pods in namespace dev
```

may be allowed while:

``` text
DELETE secrets in namespace dev
```

is denied.

RBAC is the primary authorization mechanism to master for DevOps work.

------------------------------------------------------------------------

# 119. Production Security Model

A useful security stack:

``` text
Identity
  ↓
Authentication
  ↓
RBAC
  ↓
Admission policy
  ↓
Pod security
  ↓
NetworkPolicy
  ↓
Image security
  ↓
Runtime security
  ↓
Secrets management
```

Security is not one Kubernetes YAML field.

------------------------------------------------------------------------

# 120. Image Security

Production image security should include:

``` text
trusted base images
small images where practical
versioned tags
vulnerability scanning
SBOM
signature/provenance where adopted
non-root execution
regular patching
```

Avoid relying only on:

``` text
latest
```

because it is mutable and weak for reproducibility.

------------------------------------------------------------------------

# 121. Resource Planning

A cluster should not normally run at 100% capacity.

Plan headroom for:

``` text
rolling deployments
node failure
DaemonSets
system components
traffic spikes
autoscaling
maintenance
```

Example:

``` text
3 replicas require capacity
+
capacity for at least one additional replica
+
failure/maintenance headroom
```

Capacity planning is part of reliability.

------------------------------------------------------------------------

# 122. High Availability

A production Kubernetes control plane should avoid a single point of
failure.

Typical HA concepts:

``` text
multiple control-plane nodes
multiple API server instances
load-balanced API endpoint
highly available etcd
multiple worker nodes
multiple availability zones
```

For workload HA:

``` text
replicas
+
topology spread
+
PDB
+
multi-zone placement
```

These solve different failure modes.

------------------------------------------------------------------------

# 123. Multi-Zone Design

A production application should not accidentally place every replica in
one failure domain.

Example:

``` text
Zone A → replica 1
Zone B → replica 2
Zone C → replica 3
```

Use:

-   topology spread
-   affinity/anti-affinity
-   zone-aware storage
-   load balancing

Also remember that a stateful workload may have storage constraints that
affect placement.

------------------------------------------------------------------------

# 124. Cluster Upgrades

Never treat a Kubernetes upgrade as:

``` text
apt upgrade
```

A production upgrade requires planning around:

``` text
version compatibility
API removals/deprecations
control plane
worker nodes
CNI
CSI
Ingress/Gateway
admission webhooks
Helm charts
operators
applications
backup
rollback/recovery plan
```

Typical sequence:

``` text
read release notes
 ↓
check compatibility
 ↓
backup
 ↓
upgrade control plane
 ↓
upgrade nodes
 ↓
validate workloads
```

Exact procedure depends on the installation method and platform.

------------------------------------------------------------------------

# 125. Version Skew

Kubernetes components are not all required to have identical versions at
every moment.

There are supported version-skew rules between:

``` text
kube-apiserver
kubelet
kubectl
kube-controller-manager
kube-scheduler
```

Before an upgrade, always check the official version-skew policy for the
target release.

------------------------------------------------------------------------

# 126. Backup and Disaster Recovery

Separate:

``` text
control-plane state
```

from:

``` text
application data
```

Example:

``` text
etcd backup
+
PV backup
+
external database backup
+
Git/IaC
+
secret recovery
```

Define:

### RPO

How much data can you afford to lose?

### RTO

How long can recovery take?

Example:

``` text
RPO = 15 minutes
RTO = 1 hour
```

These are business requirements translated into technical design.

------------------------------------------------------------------------

# 127. Production Deployment Checklist

Before production:

## Workload

-   replicas configured
-   resource requests/limits defined
-   readiness probe
-   liveness probe where justified
-   startup probe where needed
-   graceful shutdown
-   correct termination grace period

## Networking

-   Service selector verified
-   DNS verified
-   ingress/gateway configured
-   TLS configured
-   NetworkPolicy reviewed

## Security

-   non-root where possible
-   least-privilege RBAC
-   restricted ServiceAccount
-   secrets protected
-   image scanned
-   security context configured

## Reliability

-   PDB
-   topology spread
-   multiple replicas
-   multi-zone placement where appropriate
-   enough cluster capacity

## Observability

-   logs
-   metrics
-   alerts
-   dashboards
-   useful events

------------------------------------------------------------------------

# 128. Production Cluster Checklist

## Control plane

-   HA
-   etcd backup
-   API availability
-   certificate lifecycle
-   upgrade process

## Nodes

-   capacity
-   autoscaling
-   OS patching
-   kubelet health
-   runtime health
-   disk monitoring

## Networking

-   CNI
-   DNS
-   ingress/gateway
-   load balancing
-   NetworkPolicy

## Storage

-   CSI health
-   backup
-   expansion
-   reclaim policy
-   zone awareness

## Security

-   RBAC
-   admission policy
-   Pod Security
-   image security
-   secret management

## Operations

-   monitoring
-   logging
-   alerting
-   incident process
-   DR testing

------------------------------------------------------------------------

# 129. kubectl Core Commands

``` bash
kubectl get pods
kubectl get pods -A
kubectl get nodes
kubectl get svc
kubectl get deploy
kubectl get rs
kubectl get events --sort-by=.lastTimestamp
```

Detailed:

``` bash
kubectl describe pod <pod>
kubectl describe node <node>
kubectl describe svc <service>
```

YAML:

``` bash
kubectl get pod <pod> -o yaml
kubectl get deploy <deploy> -o yaml
```

------------------------------------------------------------------------

# 130. Logs and Debugging

``` bash
kubectl logs <pod>
kubectl logs <pod> -c <container>
kubectl logs <pod> --previous
kubectl logs -f <pod>
```

Execute:

``` bash
kubectl exec -it <pod> -- sh
```

Port forwarding:

``` bash
kubectl port-forward svc/<service> 8080:80
```

This is useful for testing without exposing a Service externally.

------------------------------------------------------------------------

# 131. JSONPath and Custom Output

Examples:

``` bash
kubectl get pods -o wide
kubectl get pods -o yaml
kubectl get pods -o json
```

JSONPath:

``` bash
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
```

Sorting:

``` bash
kubectl get pods --sort-by=.status.startTime
```

These become useful for automation and troubleshooting.

------------------------------------------------------------------------

# 132. Useful kubectl Patterns

Get all objects in a namespace:

``` bash
kubectl get all -n <namespace>
```

Watch:

``` bash
kubectl get pods -w
```

Labels:

``` bash
kubectl get pods --show-labels
kubectl get pods -l app=nginx
```

Namespaces:

``` bash
kubectl get ns
kubectl get pods -n kube-system
```

------------------------------------------------------------------------

# 133. YAML Validation Workflow

Do not immediately apply every YAML file.

Use:

``` bash
kubectl apply --dry-run=client -f file.yaml
```

Then inspect:

``` bash
kubectl diff -f file.yaml
```

Then:

``` bash
kubectl apply -f file.yaml
```

Afterward:

``` bash
kubectl get
kubectl describe
kubectl get events
```

For Helm:

``` bash
helm lint
helm template
```

------------------------------------------------------------------------

# 134. Common Kubernetes Failure Patterns

## Pod Pending

Usually scheduling/resource/storage constraints.

## CrashLoopBackOff

Usually application/container/probe/config failure.

## ImagePullBackOff

Usually image/registry/authentication/network issue.

## OOMKilled

Usually memory limit or application memory behavior.

## Service has no traffic

Usually selector/endpoints/network/application issue.

## DNS fails

Usually CoreDNS/network/configuration issue.

## Node NotReady

Usually kubelet/runtime/network/resource/control-plane connectivity
issue.

The status is a clue, not the root cause.

------------------------------------------------------------------------

# 135. Practical Lab --- Deployment

Create:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

Apply:

``` bash
kubectl apply -f deployment.yaml
```

Verify:

``` bash
kubectl get deploy
kubectl get rs
kubectl get pods -o wide
```

Scale:

``` bash
kubectl scale deployment nginx --replicas=5
```

------------------------------------------------------------------------

# 136. Practical Lab --- Rolling Update

Change:

``` yaml
image: nginx:1.28
```

Apply:

``` bash
kubectl apply -f deployment.yaml
```

Watch:

``` bash
kubectl rollout status deployment/nginx
kubectl get pods -w
```

Inspect:

``` bash
kubectl rollout history deployment/nginx
```

Rollback:

``` bash
kubectl rollout undo deployment/nginx
```

------------------------------------------------------------------------

# 137. Practical Lab --- Service

Create:

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
  type: NodePort
```

Check:

``` bash
kubectl get svc nginx
kubectl get endpointslice
```

Then test from a node/client using the appropriate node IP and assigned
NodePort.

------------------------------------------------------------------------

# 138. Practical Lab --- Break a Service Selector

Change the Service selector:

``` yaml
selector:
  app: wrong
```

Apply it.

Then:

``` bash
kubectl get svc
kubectl get endpointslice
kubectl get pods --show-labels
```

Observe that:

``` text
Pods exist
but no matching endpoints
```

Restore:

``` yaml
selector:
  app: nginx
```

This is a valuable troubleshooting lab.

------------------------------------------------------------------------

# 139. Practical Lab --- CrashLoopBackOff

Create a Pod that exits:

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: crash-test
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "echo failing; exit 1"]
```

Observe:

``` bash
kubectl get pod crash-test -w
kubectl logs crash-test
kubectl describe pod crash-test
```

Learn to connect:

``` text
process exit
→ restart
→ backoff
→ CrashLoopBackOff
```

------------------------------------------------------------------------

# 140. Practical Lab --- Pending Pod

Create an intentionally unschedulable Pod using an impossible node
selector.

Then:

``` bash
kubectl describe pod <pod>
kubectl get events --sort-by=.lastTimestamp
```

Your objective is not merely to fix it.

Your objective is to explain:

``` text
Why scheduler rejected every node
```

------------------------------------------------------------------------

# 141. Practical Lab --- Probes

Deploy nginx with:

``` yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
```

Break the probe path intentionally.

Observe:

``` bash
kubectl get pod
kubectl get endpointslice
```

Understand:

``` text
container may be Running
but Pod may not receive Service traffic
```

------------------------------------------------------------------------

# 142. Practical Lab --- ConfigMap

Create:

``` yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_MODE: "dev"
```

Consume it through environment variables.

Then modify the ConfigMap and observe that the running process does not
automatically receive a new environment variable value.

Restart the Pod and verify the new value.

------------------------------------------------------------------------

# 143. Practical Lab --- Secret

Create a Secret and consume it as a mounted file.

Then inspect:

``` bash
kubectl get secret
kubectl describe secret <secret>
```

Remember:

``` text
Secret data being base64 encoded does not make it encrypted.
```

------------------------------------------------------------------------

# 144. Practical Lab --- PVC

Create:

``` text
StorageClass
PVC
Pod
```

Observe:

``` bash
kubectl get pvc
kubectl get pv
kubectl describe pvc <pvc>
```

Understand the binding/provisioning flow.

------------------------------------------------------------------------

# 145. Practical Lab --- RBAC

Create:

``` text
ServiceAccount
Role
RoleBinding
```

Give permission only to:

``` text
get/list/watch pods
```

Test:

``` bash
kubectl auth can-i get pods \
  --as=system:serviceaccount:dev:app-sa \
  -n dev
```

Then:

``` bash
kubectl auth can-i delete pods \
  --as=system:serviceaccount:dev:app-sa \
  -n dev
```

Expected result:

``` text
get → yes
delete → no
```

This teaches least privilege directly.

------------------------------------------------------------------------

# 146. Practical Lab --- Taints and Tolerations

Taint a node:

``` bash
kubectl taint nodes <node> dedicated=devops:NoSchedule
```

Create a Pod without a matching toleration.

Observe:

``` text
Pending
```

Add:

``` yaml
tolerations:
  - key: dedicated
    operator: Equal
    value: devops
    effect: NoSchedule
```

Observe scheduling.

Remove taint:

``` bash
kubectl taint nodes <node> dedicated=devops:NoSchedule-
```

------------------------------------------------------------------------

# 147. Practical Lab --- Node Maintenance

Use a non-critical workload.

``` bash
kubectl cordon <node>
kubectl get nodes
```

Then:

``` bash
kubectl drain <node> --ignore-daemonsets
```

Observe Pod movement.

After maintenance:

``` bash
kubectl uncordon <node>
```

Do this in your lab before ever performing node maintenance in
production.

------------------------------------------------------------------------

# 148. Practical Lab --- DNS

Launch a temporary diagnostic Pod.

Then test:

``` bash
nslookup kubernetes.default
nslookup <service>.<namespace>
```

Compare:

``` text
Service DNS
vs
Pod IP
```

Then inspect CoreDNS.

------------------------------------------------------------------------

# 149. Practical Lab --- NetworkPolicy

Create:

``` text
frontend
backend
```

Allow:

``` text
frontend → backend
```

and deny other traffic.

Test connectivity before and after applying the policy.

This is one of the best ways to understand that NetworkPolicy is an
enforcement model rather than just YAML syntax.

------------------------------------------------------------------------

# 150. Practical Lab --- Troubleshooting Challenge

Build:

``` text
Deployment
Service
ConfigMap
Secret
PVC
Ingress/NodePort
```

Then intentionally introduce:

1.  wrong Service selector
2.  wrong image tag
3.  invalid environment variable
4.  failed readiness probe
5.  insufficient resource request
6.  incorrect RBAC permission
7.  broken DNS test
8.  PVC problem

For every incident write:

``` text
Symptom
↓
Evidence
↓
Root cause
↓
Fix
↓
Prevention
```

This is much more valuable than simply completing YAML labs.

------------------------------------------------------------------------

# 151. Production Mental Model

When an application is deployed:

``` text
Git
 ↓
CI
 ↓
image registry
 ↓
Helm/YAML/GitOps
 ↓
API Server
 ↓
Admission
 ↓
etcd
 ↓
Controller
 ↓
Scheduler
 ↓
kubelet
 ↓
CRI
 ↓
container runtime
 ↓
Pod
 ↓
Service
 ↓
Ingress/Gateway
 ↓
user
```

This end-to-end chain is one of the best mental models for Kubernetes
interviews.

------------------------------------------------------------------------

# 152. How to Think During an Incident

Suppose:

``` text
User says:
"Application is down."
```

Do not start with:

``` bash
kubectl delete pod
```

Start:

``` text
Is DNS working?
        ↓
Is external entry point working?
        ↓
Is Service present?
        ↓
Does Service have endpoints?
        ↓
Are Pods Ready?
        ↓
Are containers healthy?
        ↓
Are nodes healthy?
        ↓
Are dependencies healthy?
        ↓
What changed?
```

This is the difference between:

``` text
command execution
```

and:

``` text
incident diagnosis
```

------------------------------------------------------------------------

# 153. 6-Year DevOps Kubernetes Expectations

At around six years of DevOps experience, you should be comfortable
explaining and operating:

## Fundamentals

-   Pods
-   namespaces
-   labels/selectors
-   API objects
-   declarative model
-   reconciliation

## Architecture

-   API server
-   etcd
-   scheduler
-   controllers
-   kubelet
-   CRI
-   containerd

## Workloads

-   Deployment
-   ReplicaSet
-   StatefulSet
-   DaemonSet
-   Job
-   CronJob

## Networking

-   Service
-   ClusterIP
-   NodePort
-   LoadBalancer
-   headless Service
-   EndpointSlice
-   DNS
-   CNI
-   NetworkPolicy
-   Ingress
-   Gateway API

## Storage

-   volume
-   emptyDir
-   PV
-   PVC
-   StorageClass
-   CSI
-   reclaim policies

## Scheduling

-   requests/limits
-   QoS
-   nodeSelector
-   affinity
-   anti-affinity
-   taints/tolerations
-   topology spread

## Security

-   authentication
-   authorization
-   RBAC
-   ServiceAccounts
-   SecurityContext
-   capabilities
-   seccomp
-   Pod Security
-   admission
-   NetworkPolicy
-   image security

## Reliability

-   probes
-   graceful shutdown
-   PDB
-   rolling updates
-   rollback
-   multi-zone design

## Scaling

-   HPA
-   Metrics Server
-   VPA concept
-   node autoscaling
-   cloud node provisioning

## Operations

-   kubectl
-   logs
-   events
-   node maintenance
-   upgrades
-   certificates
-   kubeconfig
-   backup/DR

## Ecosystem

-   Helm
-   Terraform integration
-   CI/CD
-   GitOps
-   Argo CD
-   EKS

------------------------------------------------------------------------

# 154. What You Should Be Able to Explain Without Notes

A 6-year engineer should be able to answer questions such as:

### Why does Kubernetes need a Service?

Because Pod IPs are ephemeral and workload membership changes. Service
provides a stable abstraction and routes traffic to matching endpoints.

### Why did my Pod remain Pending?

Because the scheduler could not find a feasible node, commonly due to
resources, taints, affinity, selectors, quota, or storage constraints.

### Why is a Pod Running but not receiving traffic?

Readiness may be failing, causing it to be removed from Service
endpoints.

### Why does CrashLoopBackOff happen?

The container repeatedly starts and exits/fails, and Kubernetes applies
increasing restart backoff.

### Why was my Pod OOMKilled?

The process exceeded its memory constraint or the node experienced
relevant memory pressure; inspect container termination state and node
conditions.

### Why is my Service not routing?

Check selectors → EndpointSlices → Pod readiness → network path →
application listener.

### What is the difference between authentication and authorization?

Authentication identifies the caller; authorization determines what that
caller may do.

### What is the difference between Role and ClusterRole?

Role is namespace-scoped; ClusterRole can represent cluster-scoped
permissions and can also be referenced by bindings in namespaces.

### Why do we need readiness and liveness separately?

Readiness controls traffic eligibility; liveness determines whether a
stuck/unhealthy container should be restarted.

### Why use StatefulSet instead of Deployment?

When stable identity and/or stable persistent storage association is
part of the workload's requirements.

------------------------------------------------------------------------

# 155. Kubernetes Interview Troubleshooting Framework

For almost any scenario, use:

``` text
1. State the symptom
2. Identify the Kubernetes object
3. Inspect current state
4. Inspect events
5. Inspect dependencies
6. Inspect node/control-plane health
7. Form a hypothesis
8. Test the hypothesis
9. Fix
10. Verify
11. Prevent recurrence
```

Example:

``` text
Service unavailable
```

Answer:

``` text
Check Service
→ selector
→ EndpointSlices
→ Pod readiness
→ Pod logs
→ DNS
→ network policy
→ node/CNI
→ application listener
```

That is much stronger than listing twenty commands without reasoning.

------------------------------------------------------------------------

# 156. Learning Order

A practical learning sequence:

``` text
1. Linux networking + processes
2. Containers/Docker
3. Kubernetes architecture
4. Pods
5. Deployments
6. Services
7. ConfigMaps/Secrets
8. Storage
9. Probes
10. Requests/limits
11. Scheduling
12. Networking/CNI
13. NetworkPolicy
14. RBAC/security
15. Ingress/Gateway
16. StatefulSets
17. Jobs/CronJobs
18. HPA/autoscaling
19. Observability
20. Troubleshooting
21. Helm
22. GitOps/Argo CD
23. Upgrades/backup/HA
24. EKS/cloud integration
25. Production design
```

Do not spend months memorizing API fields before you can troubleshoot a
broken Service or Pod.

------------------------------------------------------------------------

# 157. Recommended Hands-On Project

Build a small production-like platform:

``` text
                         Internet
                            |
                     NodePort / Ingress
                            |
                         frontend
                            |
                         backend
                       /         \
                 ConfigMap      Secret
                     |             |
                  config        credentials
                            |
                         database
                            |
                           PVC
```

Add:

``` text
Deployment
Service
ConfigMap
Secret
StatefulSet
PVC
readiness/liveness
resource requests/limits
HPA
PDB
NetworkPolicy
RBAC
Helm
Git
Argo CD
monitoring
logging
```

Then deliberately break it.

Your goal is not:

> "Can I deploy it?"

Your goal is:

> "Can I operate and recover it?"

------------------------------------------------------------------------

# 158. Final Kubernetes Checklist

Before calling yourself comfortable with Kubernetes at the 6-year DevOps
level, make sure you can:

-   explain the reconciliation model
-   explain the API server request path
-   explain etcd and quorum
-   explain scheduler decisions
-   explain controller reconciliation
-   explain kubelet/CRI/containerd
-   troubleshoot Pod lifecycle
-   deploy and roll back Deployments
-   explain StatefulSet identity
-   use DaemonSets correctly
-   use Jobs/CronJobs
-   explain Services deeply
-   troubleshoot EndpointSlices
-   troubleshoot DNS
-   explain CNI at a practical level
-   write NetworkPolicies
-   use Ingress/Gateway
-   understand PV/PVC/StorageClass/CSI
-   size resource requests/limits
-   understand QoS and OOM behavior
-   use affinity/anti-affinity
-   use taints/tolerations
-   use topology spread
-   configure probes correctly
-   implement graceful shutdown
-   understand PDBs
-   configure HPA
-   understand node autoscaling
-   implement least-privilege RBAC
-   use ServiceAccounts safely
-   harden Pods with SecurityContext
-   understand Pod Security
-   understand admission webhooks
-   understand finalizers/ownership
-   understand CRDs/operators
-   package workloads with Helm
-   understand GitOps
-   integrate Kubernetes with CI/CD
-   maintain kubeconfig/context safety
-   perform node maintenance
-   understand upgrades/version skew
-   plan HA
-   plan backup/DR
-   diagnose cluster failures
-   diagnose application failures
-   explain production trade-offs

------------------------------------------------------------------------

# 159. Final Principle

Kubernetes expertise is not:

``` text
memorizing kubectl commands
```

It is:

``` text
understanding desired state
        +
understanding reconciliation
        +
understanding networking
        +
understanding storage
        +
understanding security
        +
understanding scheduling
        +
understanding observability
        +
understanding failure modes
        +
being able to recover production workloads
```

If you can look at:

``` text
Pod
Deployment
Service
Node
PVC
RBAC
Ingress
```

and reason about:

``` text
what should happen
what is actually happening
where the mismatch is
why the mismatch exists
how to prove the root cause
how to fix it safely
how to prevent it
```

you are operating Kubernetes at the level expected from a strong senior
DevOps engineer.

------------------------------------------------------------------------

# 160. Real-World Incident Scenarios (Worked Examples)

These follow the format: symptom → commands run → what the output
actually showed → root cause → fix → prevention.

## 160.1 Deployment stuck mid-rollout, half old / half new Pods

``` bash
kubectl rollout status deployment/myapp
# Waiting for deployment "myapp" rollout to finish: 2 out of 5 new replicas have been updated...

kubectl get pods -l app=myapp
# 3 pods Running (old), 2 pods CrashLoopBackOff (new)

kubectl logs <new-pod> --previous
# panic: failed to connect to database: connection refused
```

Root cause: new version changed a required env var name
(`DB_HOST` → `DATABASE_HOST`) but the ConfigMap wasn't updated in the
same change. Old Pods keep running fine because Kubernetes never kills
working replicas until enough new ones are healthy — this is the
`maxUnavailable`/`maxSurge` rolling update strategy protecting you.

Fix: `kubectl rollout undo deployment/myapp`, correct the ConfigMap key
in Git, redeploy.

Prevention: readiness probes tied to an actual DB connectivity check
(not just an HTTP 200 on a static route), so bad rollouts fail fast
and visibly instead of degrading only partially.

## 160.2 Service intermittently returns 503

``` bash
kubectl get endpointslices -l kubernetes.io/service-name=myapp
# only 1 of 3 expected pod IPs listed

kubectl get pods -l app=myapp -o wide
kubectl describe pod <pod-missing-from-endpoints>
# Readiness probe failed: HTTP probe failed with statuscode: 500
```

Root cause: one replica's readiness probe hits `/health`, which checks
a downstream cache dependency. That cache pod was itself unhealthy,
so this Pod correctly reports not-ready and is pulled from Service
endpoints — the intermittent 503s were actually load balancing
correctly working around 1-of-3 unhealthy Pods, not a Service bug.

Fix: fix the cache dependency; separately, evaluate whether `/health`
should fail on a non-critical dependency at all (readiness vs a
narrower liveness check).

## 160.3 Node shows NotReady, Pods evicted

``` bash
kubectl get nodes
# ip-10-0-3-21   NotReady   <none>   45d

kubectl describe node ip-10-0-3-21
# Conditions: MemoryPressure=True, DiskPressure=False
# Events: "kubelet has disk pressure" ... "Failed to garbage collect"

kubectl top node ip-10-0-3-21
# (fails - kubelet not reporting metrics, consistent with node distress)
```

Root cause: node's root volume filled up from unrotated container logs
(app was logging at DEBUG level in production and writing several GB
per hour to stdout, which containerd was persisting to disk).

Fix: cordon/drain the node, clear disk or replace it via ASG/Karpenter
node replacement, fix the app's log level.

Prevention: `--log-opt max-size`/log rotation on the container runtime,
node disk usage alerts before it becomes eviction-triggering, and
shipping logs to CloudWatch/ELK instead of relying on local disk
retention.

## 160.4 HPA not scaling despite high CPU

``` bash
kubectl get hpa
# myapp   Deployment/myapp   <unknown>/70%   2   10   2   45d

kubectl describe hpa myapp
# Warning FailedGetResourceMetric  unable to get metrics for resource cpu

kubectl get apiservice v1beta1.metrics.k8s.io
# False (not available)

kubectl get pods -n kube-system -l k8s-app=metrics-server
# CrashLoopBackOff
```

Root cause: metrics-server crashed after a cluster upgrade because its
`--kubelet-insecure-tls` flag/cert trust changed with the new kubelet
version, so it can't scrape kubelet metrics, so HPA has no data to
act on — a control-plane upgrade caused a workload-visible scaling
outage days later during a traffic spike.

Fix: restore metrics-server config for the new kubelet TLS setup,
verify with `kubectl top pods` before trusting HPA again.

Prevention: metrics-server health is a first-class thing to check
immediately after any cluster upgrade, not just node/Pod status.

## 160.5 ArgoCD shows Synced but users see the old version

``` bash
kubectl get pods -l app=myapp -o jsonpath='{.items[*].spec.containers[*].image}'
# myapp:latest   (all pods)

kubectl get deployment myapp -o jsonpath='{.metadata.annotations}'
# no change in resourceVersion despite a "new" Argo sync
```

Root cause: the chart used a mutable `latest` tag. Git technically
"changed" (a comment or unrelated value updated), Argo CD reconciled
and saw no actual diff in the rendered manifest since the tag string
was unchanged, so it never triggered a new rollout — the image
running in the cluster was still the old `latest` pull from whenever
that node last pulled it.

Fix: force a new rollout (`kubectl rollout restart deployment/myapp`)
as an immediate mitigation; switch to immutable commit-SHA tags as
the actual fix.

------------------------------------------------------------------------

# Quick Command Cheat Sheet

``` bash
# Cluster
kubectl cluster-info
kubectl get nodes
kubectl get nodes -o wide

# Workloads
kubectl get pods -A
kubectl get deploy
kubectl get rs
kubectl get sts
kubectl get ds
kubectl get jobs
kubectl get cronjobs

# Services
kubectl get svc
kubectl get endpointslice

# Details
kubectl describe pod <pod>
kubectl describe node <node>
kubectl describe svc <service>

# Logs
kubectl logs <pod>
kubectl logs <pod> --previous
kubectl logs -f <pod>

# Exec
kubectl exec -it <pod> -- sh

# Events
kubectl get events --sort-by=.lastTimestamp

# Rollouts
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>

# Scaling
kubectl scale deployment <name> --replicas=5
kubectl get hpa

# Resources
kubectl top nodes
kubectl top pods

# Scheduling
kubectl get nodes --show-labels
kubectl describe pod <pod>

# RBAC
kubectl auth can-i get pods
kubectl auth can-i --list

# Config
kubectl config get-contexts
kubectl config current-context
kubectl config use-context <context>

# Maintenance
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets
kubectl uncordon <node>

# API discovery
kubectl api-resources
kubectl explain pod
kubectl explain deployment.spec

# YAML
kubectl apply --dry-run=client -f file.yaml
kubectl diff -f file.yaml
kubectl apply -f file.yaml
kubectl get <resource> -o yaml
```

------------------------------------------------------------------------

# End

**Use these notes as a reference, but make the labs the real source of
skill.**

For a 6-year DevOps engineer, the strongest combination is:

``` text
Deep concepts
+
hands-on labs
+
failure injection
+
troubleshooting
+
production design
+
clear explanation
```
