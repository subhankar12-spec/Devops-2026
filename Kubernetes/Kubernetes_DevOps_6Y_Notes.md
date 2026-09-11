# Kubernetes --- 6-Year DevOps Engineer Notes

> Practical, production-focused Kubernetes notes for a DevOps engineer
> with \~6 years of experience.
>
> **Scope:** Everything you should be comfortable with in day-to-day
> engineering, troubleshooting, deployments, security, operations, and
> interviews --- without going unnecessarily deep into Kubernetes
> internals or obscure features.
>
> **Keep separate:** Helm and Argo CD/GitOps deserve their own notes.
> This document explains where they fit.

------------------------------------------------------------------------

# 1. Kubernetes Fundamentals

## What is Kubernetes?

Kubernetes (K8s) is a container orchestration platform used to:

-   Run containers across multiple machines
-   Maintain desired application state
-   Scale workloads
-   Provide service discovery and networking
-   Perform rolling deployments and rollbacks
-   Restart failed containers
-   Distribute workloads across nodes
-   Manage configuration and secrets
-   Automate application operations

### Core idea

You declare:

> "I want 3 replicas of my application running."

Kubernetes continuously works to make reality match that desired state.

``` text
Desired State
     |
     v
Kubernetes API
     |
     v
Controllers / Scheduler
     |
     v
Actual Cluster State
```

------------------------------------------------------------------------

# 2. Kubernetes Architecture

A cluster has:

``` text
                 Kubernetes Cluster
                        |
          +-------------+-------------+
          |                           |
     Control Plane                 Worker Nodes
          |                           |
   +------+------+             +------+------+
   |      |      |             |      |      |
 API   etcd  Scheduler       kubelet kube-proxy
Server         Controller          |
               Manager          Container Runtime
                                      |
                                   Pods
```

## Control Plane

Responsible for managing the cluster.

Main components:

-   kube-apiserver
-   etcd
-   kube-scheduler
-   kube-controller-manager
-   cloud-controller-manager (when applicable)

## Worker Node

Runs workloads.

Main components:

-   kubelet
-   container runtime
-   kube-proxy (commonly present, but service routing may also be
    implemented by alternatives such as CNI/eBPF datapaths)
-   Pods

------------------------------------------------------------------------

# 3. Kubernetes API Server

The API server is the front door of Kubernetes.

Almost everything goes through the API server:

``` text
kubectl
   |
   v
API Server
   |
   +--> Authentication
   +--> Authorization
   +--> Admission
   +--> etcd
```

Example:

``` bash
kubectl get pods
```

The request goes to the API server.

### Important concepts

-   Authentication = Who are you?
-   Authorization = What are you allowed to do?
-   Admission = Is this request allowed/modified before persistence?
-   API server = exposes Kubernetes API

------------------------------------------------------------------------

# 4. etcd

`etcd` is Kubernetes' distributed key-value store.

It stores cluster state such as:

-   Pods
-   Deployments
-   Services
-   ConfigMaps
-   Secrets
-   RBAC objects
-   Node information
-   Desired state

### Important

etcd is critical cluster data.

Production considerations:

-   Regular backups
-   Secure access
-   Encryption at rest where required
-   Monitor disk latency
-   Monitor database health
-   Test restore procedures

### Mental model

``` text
Kubernetes objects
       |
       v
   API Server
       |
       v
      etcd
```

Do not normally modify etcd directly.

------------------------------------------------------------------------

# 5. kube-scheduler

The scheduler decides **which node should run a newly created Pod**.

It considers things such as:

-   CPU/memory requests
-   Node availability
-   Node selectors
-   Affinity
-   Anti-affinity
-   Taints/tolerations
-   Topology constraints
-   Scheduling policies

The scheduler does not itself run the Pod.

``` text
Pod created
    |
    v
API Server
    |
    v
Scheduler
    |
    v
Select node
    |
    v
kubelet on selected node
```

------------------------------------------------------------------------

# 6. kube-controller-manager

Controllers continuously compare:

``` text
Desired state
      vs
Current state
```

and take action.

Examples:

-   Deployment controller
-   ReplicaSet controller
-   Node controller
-   Job controller
-   EndpointSlice-related controllers
-   Namespace controller

### Example

Desired:

``` yaml
replicas: 3
```

Current:

``` text
2 Pods
```

Controller creates another Pod.

------------------------------------------------------------------------

# 7. kubelet

`kubelet` runs on worker nodes.

Responsibilities include:

-   Watching Pod specifications assigned to the node
-   Starting/stopping containers through the container runtime
-   Reporting node and Pod status
-   Running health checks
-   Managing Pod lifecycle

Kubelet talks to the container runtime through the CRI.

``` text
kubelet
   |
  CRI
   |
containerd / CRI-compatible runtime
   |
OCI runtime
   |
container
```

------------------------------------------------------------------------

# 8. Container Runtime

Kubernetes needs a CRI-compatible container runtime.

Common example:

-   containerd

The runtime is responsible for actually running containers.

### Important distinction

``` text
Kubernetes
= orchestration

containerd
= container runtime

Pod
= Kubernetes workload unit containing one or more containers
```

Kubernetes does not require Docker Engine to run containers.

------------------------------------------------------------------------

# 9. Pods

A Pod is the smallest deployable unit in Kubernetes.

A Pod can contain:

-   One container --- most common
-   Multiple tightly coupled containers

Containers inside the same Pod share:

-   Network namespace
-   Pod IP
-   localhost
-   Volumes mounted into the Pod

Example:

``` text
Pod
 |
 +-- app container
 |
 +-- sidecar container
```

### Multi-container Pod

Use when containers need tightly coupled lifecycle/network/storage.

Example:

``` text
Application container
        |
        +---- shared volume ----+
        |                       |
        v                       v
     app logs                log agent
```

Do not put unrelated applications into one Pod.

------------------------------------------------------------------------

# 10. Pod Lifecycle

Common phases/states:

-   Pending
-   Running
-   Succeeded
-   Failed
-   Unknown

A Pod is generally disposable.

Do not treat a Pod as a permanent server.

If a Pod dies:

-   A Deployment may create a replacement
-   A StatefulSet may recreate the corresponding Pod
-   A Job may create/retry Pods according to its configuration

------------------------------------------------------------------------

# 11. Namespaces

Namespaces logically isolate resources inside a cluster.

Example:

``` bash
kubectl get pods -n dev
kubectl get pods -n prod
```

Common namespaces:

-   default
-   kube-system
-   kube-public
-   kube-node-lease

Create:

``` bash
kubectl create namespace dev
```

Use namespaces for:

-   Environment separation
-   Team/application organization
-   RBAC boundaries
-   ResourceQuota
-   NetworkPolicy scope

### Important

Namespaces are **not a complete security boundary** by themselves.

------------------------------------------------------------------------

# 12. Labels and Selectors

Labels identify Kubernetes objects.

``` yaml
labels:
  app: nginx
  environment: prod
```

Selectors find matching objects.

Example:

``` yaml
selector:
  matchLabels:
    app: nginx
```

### Mental model

``` text
Label = tag attached to object

Selector = query used to find matching objects
```

Labels are fundamental to:

-   Services
-   Deployments
-   ReplicaSets
-   Scheduling
-   Monitoring
-   Organization

------------------------------------------------------------------------

# 13. Kubernetes YAML

Typical structure:

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
```

Important fields:

``` text
apiVersion
kind
metadata
spec
```

### `spec`

Defines desired state.

### `status`

Represents current observed state and is normally populated by
Kubernetes.

------------------------------------------------------------------------

# 14. Imperative vs Declarative

## Imperative

Tell Kubernetes what command to execute.

``` bash
kubectl create deployment nginx --image=nginx
```

## Declarative

Describe desired state in YAML.

``` bash
kubectl apply -f deployment.yaml
```

Production workflows generally favor declarative configuration.

------------------------------------------------------------------------

# 15. Deployments

A Deployment manages stateless application Pods.

It provides:

-   Replica management
-   Rolling updates
-   Rollbacks
-   ReplicaSets
-   Desired-state management

Example:

``` yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: web

spec:
  replicas: 3

  selector:
    matchLabels:
      app: web

  template:
    metadata:
      labels:
        app: web

    spec:
      containers:
        - name: web
          image: nginx:1.27
          ports:
            - containerPort: 80
```

Apply:

``` bash
kubectl apply -f deployment.yaml
```

Check:

``` bash
kubectl get deployment
kubectl get rs
kubectl get pods
```

------------------------------------------------------------------------

# 16. ReplicaSet

A ReplicaSet maintains the requested number of matching Pods.

``` text
Deployment
    |
    v
ReplicaSet
    |
    +-- Pod
    +-- Pod
    +-- Pod
```

Normally you manage Deployments rather than creating ReplicaSets
directly.

------------------------------------------------------------------------

# 17. StatefulSets

StatefulSet is used for workloads requiring stable identity and/or
persistent storage.

Examples:

-   Databases
-   Kafka-like systems
-   Stateful clustered applications

Pods have stable names:

``` text
db-0
db-1
db-2
```

StatefulSets can provide:

-   Stable network identity
-   Ordered creation/deletion behavior
-   Stable persistent storage association

Stateful does **not** automatically mean the application is safe to run
as a cluster. The application itself must support the required
clustering/replication behavior.

------------------------------------------------------------------------

# 18. DaemonSets

A DaemonSet ensures a Pod runs on selected nodes, commonly one Pod per
eligible node.

Typical uses:

-   Node log collectors
-   Monitoring agents
-   Security agents
-   Network components

Example concept:

``` text
Node 1 -> agent
Node 2 -> agent
Node 3 -> agent
```

------------------------------------------------------------------------

# 19. Jobs

A Job runs a task to completion.

Examples:

-   Database migration
-   Batch processing
-   One-time scripts

``` yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: migration
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migration
          image: myapp:migrate
```

------------------------------------------------------------------------

# 20. CronJobs

CronJob creates Jobs on a schedule.

``` yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: backup
              image: backup:1.0
```

Production considerations:

-   Concurrency policy
-   History limits
-   Job retries
-   Idempotency
-   Time zone requirements
-   Resource requests/limits

------------------------------------------------------------------------

# 21. Services

Pods are ephemeral and their IPs can change.

A Service provides a stable network endpoint for a group of Pods.

``` text
Client
  |
  v
Service
  |
  +--> Pod
  +--> Pod
  +--> Pod
```

Service selects Pods using labels.

------------------------------------------------------------------------

# 22. Service Types

## ClusterIP

Default.

Accessible inside the cluster.

``` yaml
type: ClusterIP
```

Use for internal communication.

------------------------------------------------------------------------

## NodePort

Exposes a Service through a port on nodes.

``` yaml
type: NodePort
```

Example:

``` text
NodeIP:30080
     |
     v
 Service
     |
     v
 Pods
```

Useful for labs and simple exposure.

------------------------------------------------------------------------

## LoadBalancer

Requests external load-balancing integration, commonly through a cloud
provider.

Example in cloud:

``` text
Internet
   |
Cloud Load Balancer
   |
Kubernetes Service
   |
Pods
```

------------------------------------------------------------------------

# 23. Service Discovery and DNS

Kubernetes provides DNS for Services.

Example:

``` text
my-service
```

or:

``` text
my-service.my-namespace.svc.cluster.local
```

Pods should generally communicate using Service DNS names rather than
hard-coded Pod IPs.

------------------------------------------------------------------------

# 24. Endpoints and EndpointSlices

A Service selects Pods, and Kubernetes maintains backend information for
those selected endpoints.

Modern Kubernetes uses **EndpointSlices** for scalable endpoint
tracking.

Troubleshooting:

``` bash
kubectl get endpoints
kubectl get endpointslices
```

If a Service has no backends, check:

-   Selector
-   Pod labels
-   Pod readiness
-   Namespace
-   EndpointSlices

------------------------------------------------------------------------

# 25. Ingress

Ingress provides HTTP/HTTPS routing into cluster Services.

Example:

``` text
                 Internet
                    |
                    v
             Ingress Controller
                    |
          +---------+---------+
          |                   |
        /api                /web
          |                   |
       api-svc             web-svc
          |                   |
        Pods                Pods
```

Important:

> An Ingress resource by itself does not necessarily implement traffic
> handling. You need an ingress controller or an implementation provided
> by your platform.

Typical features:

-   Host-based routing
-   Path-based routing
-   TLS termination
-   HTTP/HTTPS routing

------------------------------------------------------------------------

# 26. Gateway API

Gateway API is a newer Kubernetes networking API for expressing traffic
routing.

Core concepts include:

-   GatewayClass
-   Gateway
-   HTTPRoute
-   GRPCRoute
-   Other route types depending on implementation

Know the concept and why it exists.

You do not need to memorize every Gateway API field for normal DevOps
work.

------------------------------------------------------------------------

# 27. ConfigMaps

ConfigMap stores non-sensitive configuration.

Example:

``` yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_MODE: production
  LOG_LEVEL: info
```

Use as:

-   Environment variables
-   Mounted files

Do not store passwords or tokens in ConfigMaps.

------------------------------------------------------------------------

# 28. Secrets

Secrets are intended for sensitive configuration.

Examples:

-   Passwords
-   API tokens
-   Certificates
-   Registry credentials

Example:

``` yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  DB_PASSWORD: change-me
```

### Important security reality

Kubernetes Secrets are not automatically equivalent to a fully secure
external secrets system.

Production considerations:

-   Encryption at rest
-   RBAC
-   Least privilege
-   Secret rotation
-   External secret managers where appropriate
-   Avoid committing plaintext secrets to Git

------------------------------------------------------------------------

# 29. Volumes

Containers are ephemeral.

Kubernetes provides storage mechanisms for Pods.

Common volume concepts:

-   emptyDir
-   configMap
-   secret
-   projected
-   persistentVolumeClaim

------------------------------------------------------------------------

# 30. emptyDir

Temporary storage tied to the Pod lifecycle.

``` yaml
volumes:
  - name: data
    emptyDir: {}
```

Useful for:

-   Temporary files
-   Shared scratch space between containers
-   Cache

When the Pod is removed, the `emptyDir` data is normally removed.

------------------------------------------------------------------------

# 31. Persistent Volumes

A PersistentVolume (PV) represents storage available to Kubernetes.

``` text
Storage system
      |
      v
PersistentVolume
      |
      v
PersistentVolumeClaim
      |
      v
Pod
```

------------------------------------------------------------------------

# 32. PersistentVolumeClaim

A PVC requests storage.

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
      storage: 5Gi
```

Pods mount the PVC.

------------------------------------------------------------------------

# 33. StorageClass

StorageClass enables dynamic provisioning.

``` text
PVC
 |
 v
StorageClass
 |
 v
Provisioner
 |
 v
Actual storage
```

This is the normal production pattern in many cloud environments.

------------------------------------------------------------------------

# 34. Access Modes

Know the concepts:

-   ReadWriteOnce (RWO)
-   ReadOnlyMany (ROX)
-   ReadWriteMany (RWX)
-   ReadWriteOncePod (RWOP)

Actual support depends on the storage implementation.

Do not assume every storage backend supports every mode.

------------------------------------------------------------------------

# 35. Probes

Probes are essential for production workloads.

## Liveness Probe

Answers:

> Is the container still alive?

Failure can cause the container to be restarted.

## Readiness Probe

Answers:

> Can this Pod receive traffic?

If readiness fails, the Pod can be removed from Service endpoints while
remaining running.

## Startup Probe

Useful for slow-starting applications.

It gives the application time to start before liveness/readiness checks
take over.

### Mental model

``` text
Startup
   |
   v
Application ready?
   |
   +-- No --> don't treat slow startup as failure
   |
   +-- Yes --> normal readiness/liveness behavior
```

------------------------------------------------------------------------

# 36. Resources: Requests and Limits

## Requests

Used for scheduling.

``` yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
```

Meaning:

> This Pod needs at least this much resource for scheduling purposes.

## Limits

Maximum resource boundary.

``` yaml
resources:
  limits:
    cpu: "1"
    memory: "512Mi"
```

### CPU

``` text
1000m = 1 CPU
250m = 0.25 CPU
```

### Memory

Common units:

``` text
Mi
Gi
```

### Important

Poorly configured requests/limits can cause:

-   Scheduling problems
-   OOMKills
-   CPU throttling
-   Inefficient cluster utilization

------------------------------------------------------------------------

# 37. QoS Classes

Kubernetes can classify Pods into:

-   Guaranteed
-   Burstable
-   BestEffort

Understand the operational impact, especially during node memory
pressure.

------------------------------------------------------------------------

# 38. Scheduling

The scheduler decides where Pods run.

Important scheduling mechanisms:

-   Resource requests
-   nodeSelector
-   node affinity
-   pod affinity
-   pod anti-affinity
-   taints/tolerations
-   topology spread constraints

------------------------------------------------------------------------

# 39. nodeSelector

Simple node selection.

``` yaml
nodeSelector:
  disktype: ssd
```

Pod only schedules to nodes with that label.

------------------------------------------------------------------------

# 40. Node Affinity

More expressive than nodeSelector.

Types include:

-   requiredDuringSchedulingIgnoredDuringExecution
-   preferredDuringSchedulingIgnoredDuringExecution

Use when workload placement needs rules or preferences.

------------------------------------------------------------------------

# 41. Pod Affinity / Anti-Affinity

Affinity:

> Place workloads near each other.

Anti-affinity:

> Prefer or require workloads to be separated.

Useful for:

-   High availability
-   Avoiding single-node concentration
-   Latency-sensitive workloads

------------------------------------------------------------------------

# 42. Taints and Tolerations

Taint is applied to a node.

``` text
Node
  |
  +-- taint: dedicated=gpu:NoSchedule
```

A Pod needs a matching toleration to schedule there.

### Common effects

-   NoSchedule
-   PreferNoSchedule
-   NoExecute

Mental model:

``` text
Taint = Node says "keep away"

Toleration = Pod says "I am allowed"
```

A toleration does not automatically force the Pod onto that node.

------------------------------------------------------------------------

# 43. Topology Spread Constraints

Used to distribute Pods across topology domains such as:

-   Nodes
-   Zones
-   Regions

Useful for high availability.

Example goal:

``` text
zone-a -> 2 Pods
zone-b -> 2 Pods
zone-c -> 2 Pods
```

instead of putting everything into one zone.

------------------------------------------------------------------------

# 44. Node Management

Useful commands:

``` bash
kubectl get nodes
kubectl describe node <node>
kubectl top nodes
```

Cordon:

``` bash
kubectl cordon <node>
```

Drain:

``` bash
kubectl drain <node> --ignore-daemonsets
```

Uncordon:

``` bash
kubectl uncordon <node>
```

### Cordon

Stops new Pods from being scheduled.

### Drain

Evicts workloads so the node can be safely maintained, subject to
workload and policy constraints.

------------------------------------------------------------------------

# 45. Kubernetes Networking

At minimum, understand these layers:

``` text
Pod networking
     |
Service networking
     |
Ingress / Gateway
     |
External network
```

Kubernetes networking commonly relies on a CNI implementation.

Examples of CNI/networking technologies include:

-   Cilium
-   Calico
-   Flannel

You should understand what the CNI does rather than memorize
implementation internals.

------------------------------------------------------------------------

# 46. CNI

Container Network Interface provides the networking integration used by
Kubernetes/container runtimes.

A CNI implementation commonly handles:

-   Pod network connectivity
-   Pod IP allocation
-   Network configuration
-   Network policy features depending on implementation

If Pods cannot communicate, CNI health should be part of your
troubleshooting checklist.

------------------------------------------------------------------------

# 47. NetworkPolicy

NetworkPolicy controls allowed network traffic for selected Pods when
supported by the cluster's network implementation.

Default mindset:

``` text
Who can talk to whom?
```

Typical policy dimensions:

-   Namespace
-   Pod labels
-   IP blocks
-   Ports
-   Ingress
-   Egress

Example concept:

``` text
frontend ---> backend ---> database
   allowed       allowed

frontend -X-> database
```

Do not assume NetworkPolicy works merely because the YAML was accepted;
the networking implementation must enforce it.

------------------------------------------------------------------------

# 48. RBAC

RBAC = Role-Based Access Control.

Four important objects:

-   Role
-   ClusterRole
-   RoleBinding
-   ClusterRoleBinding

## Role

Permissions inside one namespace.

## ClusterRole

Cluster-scoped permissions or reusable permission rules.

## RoleBinding

Binds permissions to users/groups/service accounts in a namespace.

## ClusterRoleBinding

Binds a ClusterRole at cluster scope.

### Mental model

``` text
Subject
   |
Binding
   |
Role / ClusterRole
   |
Permissions
```

------------------------------------------------------------------------

# 49. ServiceAccounts

Pods can use ServiceAccounts for identity when interacting with the
Kubernetes API or integrated systems.

Check:

``` bash
kubectl get serviceaccounts
```

Production principle:

> Give workloads only the permissions they actually need.

Avoid unnecessarily mounting powerful credentials.

------------------------------------------------------------------------

# 50. Security Context

Security context controls security-related settings for Pods/containers.

Important concepts:

-   runAsUser
-   runAsGroup
-   runAsNonRoot
-   readOnlyRootFilesystem
-   allowPrivilegeEscalation
-   capabilities
-   seccompProfile

Example:

``` yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
```

Prefer non-root workloads when possible.

------------------------------------------------------------------------

# 51. Pod Security

Understand:

-   Pod Security Standards
-   Namespace-level Pod Security Admission
-   Privileged
-   Baseline
-   Restricted

Production clusters should avoid unnecessarily privileged workloads.

------------------------------------------------------------------------

# 52. Admission Controllers

Admission happens after authentication/authorization and before the
object is persisted.

They can:

-   Validate requests
-   Mutate requests
-   Enforce policies

Examples in the ecosystem include:

-   Pod Security Admission
-   Policy engines such as Kyverno or Gatekeeper

You should understand the concept even if you do not administer every
admission plugin.

------------------------------------------------------------------------

# 53. Rolling Updates

Deployment updates can be performed without stopping all application
Pods.

``` text
Old Pods
  |
  v
New Pods gradually created
  |
  v
Old Pods gradually removed
```

Important Deployment settings:

-   maxUnavailable
-   maxSurge

Example:

``` yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```

------------------------------------------------------------------------

# 54. Rollback

Check rollout:

``` bash
kubectl rollout status deployment/web
```

History:

``` bash
kubectl rollout history deployment/web
```

Rollback:

``` bash
kubectl rollout undo deployment/web
```

Production engineers should know how to safely roll back a bad
deployment.

------------------------------------------------------------------------

# 55. Recreate Strategy

Another Deployment strategy is:

``` yaml
strategy:
  type: Recreate
```

Old Pods are terminated before new Pods are created.

Useful when multiple versions cannot safely run simultaneously.

------------------------------------------------------------------------

# 56. Horizontal Pod Autoscaler

HPA changes replica count based on metrics.

``` text
CPU / Memory / custom metrics
             |
             v
            HPA
             |
             v
       Replica count
```

Example:

``` bash
kubectl autoscale deployment web \
  --cpu-percent=70 \
  --min=2 \
  --max=10
```

Modern production environments may use metrics beyond CPU/memory.

------------------------------------------------------------------------

# 57. Vertical Pod Autoscaler

VPA adjusts resource requests/limits based on observed usage, depending
on configuration and implementation.

Know the concept and trade-offs.

Do not assume HPA and VPA can always be combined without careful design.

------------------------------------------------------------------------

# 58. Cluster Autoscaling

Cluster autoscaling changes the number of nodes based on
scheduling/resource demand.

Concept:

``` text
Pending Pods
    |
    v
Cluster Autoscaler / platform autoscaler
    |
    v
New node
    |
    v
Pods schedule
```

In managed Kubernetes, node autoscaling may be implemented through
platform-specific systems.

------------------------------------------------------------------------

# 59. kubectl Essentials

Basic:

``` bash
kubectl version
kubectl cluster-info
kubectl get nodes
kubectl get namespaces
kubectl get pods
```

All namespaces:

``` bash
kubectl get pods -A
```

Detailed:

``` bash
kubectl get pods -o wide
kubectl describe pod <pod>
```

YAML:

``` bash
kubectl get pod <pod> -o yaml
```

------------------------------------------------------------------------

# 60. Working With Deployments

``` bash
kubectl get deploy
kubectl describe deploy <name>
kubectl rollout status deploy/<name>
kubectl rollout history deploy/<name>
kubectl rollout undo deploy/<name>
```

Scale:

``` bash
kubectl scale deployment web --replicas=5
```

------------------------------------------------------------------------

# 61. Logs

Basic:

``` bash
kubectl logs <pod>
```

Follow:

``` bash
kubectl logs -f <pod>
```

Previous crashed container:

``` bash
kubectl logs <pod> --previous
```

Specific container:

``` bash
kubectl logs <pod> -c <container>
```

Logs are one of the first places to investigate application failures.

------------------------------------------------------------------------

# 62. Exec

Run a command inside a container:

``` bash
kubectl exec -it <pod> -- sh
```

For multiple containers:

``` bash
kubectl exec -it <pod> -c <container> -- sh
```

Use `exec` for troubleshooting, not as a substitute for proper
application operations.

------------------------------------------------------------------------

# 63. Events

Events are extremely useful during troubleshooting.

``` bash
kubectl get events
```

Namespace-specific:

``` bash
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

Look for:

-   FailedScheduling
-   FailedMount
-   BackOff
-   ImagePullBackOff
-   Unhealthy
-   FailedCreate

------------------------------------------------------------------------

# 64. Debugging Pods

Start with:

``` bash
kubectl get pods -o wide
kubectl describe pod <pod>
kubectl logs <pod>
kubectl get events
```

Then inspect:

-   Image
-   Command/args
-   Environment
-   Volumes
-   Mounts
-   Probes
-   Requests/limits
-   ServiceAccount
-   Security context
-   Node assignment

------------------------------------------------------------------------

# 65. CrashLoopBackOff

Meaning:

> The container repeatedly starts and crashes, and Kubernetes is backing
> off before restarting it.

Common causes:

-   Application crash
-   Bad command/args
-   Missing configuration
-   Missing secret
-   Dependency unavailable
-   Wrong permissions
-   Probe failures

Investigate:

``` bash
kubectl logs <pod> --previous
kubectl describe pod <pod>
```

------------------------------------------------------------------------

# 66. ImagePullBackOff

Meaning:

> Kubernetes cannot successfully pull the container image and is backing
> off retries.

Check:

-   Image name
-   Image tag
-   Registry connectivity
-   Registry credentials
-   ImagePullSecrets
-   Network/DNS
-   Architecture compatibility
-   Registry availability

------------------------------------------------------------------------

# 67. Pending Pod

Common reasons:

-   Insufficient CPU/memory
-   Taints
-   Affinity rules
-   Node selector
-   PVC not available
-   Topology constraints

Start with:

``` bash
kubectl describe pod <pod>
```

Look at scheduler events.

------------------------------------------------------------------------

# 68. OOMKilled

Usually means the container exceeded its memory limit or the node
experienced memory pressure, depending on the situation.

Check:

``` bash
kubectl describe pod <pod>
kubectl get pod <pod> -o json
kubectl top pod
```

Investigate:

-   Memory limit
-   Application memory growth
-   JVM/runtime configuration
-   Node memory pressure
-   Actual working set

------------------------------------------------------------------------

# 69. Service Not Working

Use this troubleshooting flow:

``` text
Pod running?
    |
    v
Pod Ready?
    |
    v
Service selector correct?
    |
    v
Endpoints/EndpointSlices populated?
    |
    v
Service port correct?
    |
    v
targetPort correct?
    |
    v
Application listening?
    |
    v
NetworkPolicy?
    |
    v
Ingress/Gateway?
```

Commands:

``` bash
kubectl get svc
kubectl describe svc <svc>
kubectl get endpoints <svc>
kubectl get endpointslices
kubectl get pods --show-labels
```

------------------------------------------------------------------------

# 70. DNS Troubleshooting

Check DNS components:

``` bash
kubectl get pods -n kube-system
```

Test from a temporary Pod:

``` bash
kubectl run dns-test \
  --image=busybox:1.36 \
  -it --rm --restart=Never -- sh
```

Then:

``` sh
nslookup my-service
```

Investigate:

-   Service name
-   Namespace
-   CoreDNS health
-   Pod DNS configuration
-   CNI/networking
-   NetworkPolicy

------------------------------------------------------------------------

# 71. Node Troubleshooting

Check:

``` bash
kubectl get nodes
kubectl describe node <node>
kubectl top node
```

On the node:

``` bash
systemctl status kubelet
journalctl -u kubelet
```

Check runtime:

``` bash
systemctl status containerd
journalctl -u containerd
```

Also inspect:

-   CPU
-   Memory
-   Disk
-   Inodes
-   Network
-   Runtime health
-   Kubelet configuration
-   Certificates

------------------------------------------------------------------------

# 72. Control Plane Troubleshooting

Check:

``` bash
kubectl get pods -n kube-system
kubectl get nodes
```

Investigate:

-   API server
-   etcd
-   scheduler
-   controller manager
-   kubelet
-   Certificates
-   Network connectivity
-   Disk availability

For kubeadm clusters, control-plane components may run as static Pods.

------------------------------------------------------------------------

# 73. Static Pods

Static Pods are managed directly by kubelet rather than through a normal
Deployment/ReplicaSet controller.

In kubeadm clusters, control-plane components commonly use static Pods.

Typical manifest directory:

``` text
/etc/kubernetes/manifests/
```

Common files:

``` text
etcd.yaml
kube-apiserver.yaml
kube-controller-manager.yaml
kube-scheduler.yaml
```

Do not casually edit production control-plane manifests without
understanding the consequences.

------------------------------------------------------------------------

# 74. kubeconfig

`kubectl` uses kubeconfig to know:

-   Cluster
-   API server
-   Credentials
-   Context

Common location:

``` bash
~/.kube/config
```

Check contexts:

``` bash
kubectl config get-contexts
kubectl config current-context
```

Switch:

``` bash
kubectl config use-context <context>
```

------------------------------------------------------------------------

# 75. Authentication

Kubernetes supports multiple authentication mechanisms depending on
environment.

Common production patterns:

-   Client certificates
-   OIDC
-   Cloud IAM integration
-   ServiceAccount identities

Authentication answers:

> Who is this?

RBAC answers:

> What can this identity do?

------------------------------------------------------------------------

# 76. Authorization

Common authorization approaches include:

-   RBAC
-   Node authorization
-   Other configured authorization mechanisms

For most DevOps work, RBAC is the key concept.

------------------------------------------------------------------------

# 77. Secrets and External Secret Management

For production:

``` text
Application
    |
    v
Secret integration
    |
    v
Vault / Cloud Secret Manager / external system
```

Understand:

-   Secret rotation
-   Least privilege
-   Short-lived credentials where possible
-   Encryption
-   Auditability

------------------------------------------------------------------------

# 78. Container Image Security

Before deploying an image, care about:

-   Trusted base images
-   Image scanning
-   CVEs
-   Minimal images
-   Non-root execution
-   Image provenance
-   Signed/verified artifacts where applicable
-   Immutable versioning

Prefer:

``` text
myapp:git-sha
```

over relying on:

``` text
myapp:latest
```

for production traceability.

------------------------------------------------------------------------

# 79. ResourceQuota

Limits aggregate resource usage within a namespace.

Example concepts:

-   CPU requests
-   CPU limits
-   Memory requests
-   Memory limits
-   Pod count

Useful for preventing one namespace/team from consuming the entire
cluster.

------------------------------------------------------------------------

# 80. LimitRange

Defines default/min/max resource constraints within a namespace.

Useful for:

-   Default requests/limits
-   Minimum resource requirements
-   Maximum resource limits

------------------------------------------------------------------------

# 81. Pod Disruption Budget

PDB helps protect application availability during voluntary disruptions.

Example concept:

``` text
3 replicas
minimum available = 2
```

This helps during:

-   Node drains
-   Cluster maintenance
-   Voluntary disruptions

PDB does not protect against every failure, such as a sudden hardware
outage.

------------------------------------------------------------------------

# 82. Graceful Shutdown

Applications should handle termination correctly.

Kubernetes generally:

``` text
Pod termination
     |
     v
SIGTERM
     |
graceful shutdown period
     |
     v
SIGKILL if still running
```

Important application settings:

-   terminationGracePeriodSeconds
-   preStop hooks where justified
-   Correct signal handling
-   Readiness behavior during shutdown

------------------------------------------------------------------------

# 83. Init Containers

Init containers run before application containers.

Useful for:

-   Initialization
-   Dependency checks
-   Preparing files
-   Configuration generation

Example:

``` text
Init container
      |
      v
App container
```

Init containers must complete successfully before the main containers
start.

------------------------------------------------------------------------

# 84. Sidecars

A sidecar is a helper container in the same Pod as the main application.

Possible uses:

-   Proxy
-   Log processing
-   Local helper service
-   Configuration synchronization

Use sidecars when the lifecycle/network/storage coupling makes sense.

Do not add sidecars automatically just because they are available.

------------------------------------------------------------------------

# 85. Service Mesh --- Know the Concept

A service mesh can provide features such as:

-   Service-to-service traffic management
-   mTLS
-   Retries
-   Traffic splitting
-   Observability

Examples in the ecosystem:

-   Istio
-   Linkerd
-   Ambient/service-mesh architectures

For a 6-year DevOps engineer:

> Understand why a service mesh exists and its operational trade-offs.
> You do not need to memorize every configuration option.

------------------------------------------------------------------------

# 86. Observability

Production Kubernetes requires:

``` text
Metrics
Logs
Traces
```

### Metrics

Examples:

-   CPU
-   Memory
-   Request rate
-   Error rate
-   Latency
-   Pod restarts
-   Node health

Common ecosystem:

-   Prometheus
-   Grafana

### Logs

Common ecosystem:

-   Fluent Bit
-   OpenTelemetry Collector
-   Loki
-   Elasticsearch/OpenSearch

### Traces

Common ecosystem:

-   OpenTelemetry
-   Jaeger
-   Tempo

Know the concepts and how telemetry flows.

------------------------------------------------------------------------

# 87. Metrics Server

Metrics Server provides resource metrics commonly used by:

-   `kubectl top`
-   HPA resource metrics

Check:

``` bash
kubectl top pods
kubectl top nodes
```

If `kubectl top` fails, investigate Metrics Server and API aggregation
configuration.

------------------------------------------------------------------------

# 88. Kubernetes API Resources

Discover resources:

``` bash
kubectl api-resources
```

API versions:

``` bash
kubectl api-versions
```

Explain a resource:

``` bash
kubectl explain deployment
kubectl explain deployment.spec
```

These are extremely useful when working without memorizing YAML.

------------------------------------------------------------------------

# 89. Labels, Annotations, and Finalizers

## Labels

Used for selection and organization.

## Annotations

Store non-identifying metadata/configuration consumed by
tools/controllers.

## Finalizers

Prevent an object from being fully deleted until cleanup logic
completes.

This is important when troubleshooting objects stuck in `Terminating`.

------------------------------------------------------------------------

# 90. Object Ownership

Kubernetes uses owner references for relationships between resources.

Example:

``` text
Deployment
    |
    v
ReplicaSet
    |
    v
Pod
```

This helps Kubernetes understand dependent resources and garbage
collection.

------------------------------------------------------------------------

# 91. Kubernetes Garbage Collection

Kubernetes can remove dependent resources based on ownership
relationships.

Understand:

-   Owner references
-   Cascading deletion
-   Orphaning behavior
-   Finalizers

This matters when resources appear unexpectedly deleted or stuck
terminating.

------------------------------------------------------------------------

# 92. CRDs

CRD = CustomResourceDefinition.

It allows Kubernetes APIs to be extended with custom resource types.

Example:

``` text
Kubernetes API
      |
      +-- Deployment
      +-- Service
      +-- Pod
      |
      +-- Custom Resource
```

Commonly used by operators and platforms.

------------------------------------------------------------------------

# 93. Operators

An Operator uses Kubernetes APIs/controllers to automate operational
knowledge.

Concept:

``` text
Custom Resource
      |
      v
Operator / Controller
      |
      v
Application state
```

Examples of use cases:

-   Databases
-   Messaging systems
-   Certificates
-   Monitoring platforms

Know the pattern; you do not need to write an Operator to be a good
DevOps engineer.

------------------------------------------------------------------------

# 94. Helm --- Where It Fits

Keep Helm as a separate topic.

Kubernetes:

> Runs and manages workloads.

Helm:

> Packages and templates Kubernetes applications.

Typical flow:

``` text
Helm Chart
    |
    v
Rendered Kubernetes YAML
    |
    v
Kubernetes API
```

Learn separately:

-   Charts
-   Templates
-   Values
-   Releases
-   Repositories
-   `helm upgrade`
-   `helm rollback`

------------------------------------------------------------------------

# 95. Argo CD / GitOps --- Where It Fits

Keep Argo CD as a separate topic.

Concept:

``` text
Git
 |
 | desired state
 v
Argo CD
 |
 v
Kubernetes
```

Argo CD continuously compares Git desired state with cluster state.

Learn separately:

-   Applications
-   Projects
-   Sync
-   Automated sync
-   Health
-   Drift
-   Rollback
-   Multi-cluster management

------------------------------------------------------------------------

# 96. CI/CD + Kubernetes

Typical modern flow:

``` text
Developer
    |
    v
Git
    |
    v
CI
    |
    +--> Test
    +--> Build image
    +--> Scan
    +--> Push image
    |
    v
GitOps repository
    |
    v
Argo CD
    |
    v
Kubernetes
```

Important distinction:

``` text
CI = Build/test/package

CD = Deliver/deploy

GitOps = Desired deployment state stored in Git
```

------------------------------------------------------------------------

# 97. Kubernetes and Docker

Modern Kubernetes does not require Docker Engine.

Common flow:

``` text
Dockerfile
   |
   v
Container Image
   |
   v
Registry
   |
   v
containerd
   |
   v
Kubernetes Pod
```

Docker remains extremely useful for:

-   Building images
-   Local development
-   Container workflows

Kubernetes is responsible for orchestration.

------------------------------------------------------------------------

# 98. Kubernetes and AWS/EKS

In AWS, EKS is managed Kubernetes.

AWS can manage/control parts of the control-plane infrastructure while
you manage workloads and selected infrastructure components.

Important concepts to understand:

-   EKS cluster
-   Worker nodes
-   Managed node groups
-   Fargate
-   IAM
-   Load balancers
-   VPC/CNI
-   EBS/EFS storage
-   Security groups
-   CloudWatch/observability
-   Autoscaling
-   IAM Roles for Service Accounts / workload identity patterns
-   EKS access/RBAC integration

The exact responsibilities depend on the architecture.

------------------------------------------------------------------------

# 99. Production Deployment Checklist

Before deploying an application, check:

## Image

-   Trusted image
-   Versioned tag
-   Vulnerability scanning
-   Non-root where possible

## Deployment

-   Correct replicas
-   Rolling strategy
-   Resource requests
-   Resource limits
-   Probes
-   Graceful shutdown

## Networking

-   Service
-   Correct ports
-   DNS
-   NetworkPolicy
-   Ingress/Gateway if required

## Security

-   RBAC
-   ServiceAccount
-   Secrets
-   SecurityContext
-   Pod Security
-   Least privilege

## Availability

-   Multiple replicas
-   Anti-affinity/topology spread where appropriate
-   PDB
-   Multi-zone placement where appropriate

## Observability

-   Logs
-   Metrics
-   Alerts
-   Traces where useful

## Operations

-   Rollback plan
-   Deployment history
-   Runbook
-   Backup/recovery plan for stateful systems

------------------------------------------------------------------------

# 100. Production Kubernetes Checklist

A production cluster should have appropriate controls around:

### Cluster

-   Control-plane availability
-   etcd backups
-   Certificates
-   Upgrade strategy
-   Node lifecycle management
-   Capacity planning

### Security

-   RBAC
-   Pod Security
-   NetworkPolicy
-   Secret management
-   Image scanning
-   Least privilege
-   Audit logging

### Networking

-   CNI
-   DNS
-   Ingress/Gateway
-   Load balancing
-   Network policies

### Storage

-   StorageClasses
-   PV/PVC
-   Backup
-   Restore testing
-   Capacity monitoring

### Observability

-   Metrics
-   Logs
-   Traces where useful
-   Alerts
-   Kubernetes events

### Reliability

-   Replicas
-   Probes
-   PDB
-   Topology spread
-   Graceful shutdown
-   Autoscaling

### Delivery

-   CI/CD
-   Image promotion
-   GitOps where appropriate
-   Rollbacks
-   Immutable artifacts

------------------------------------------------------------------------

# 101. Kubernetes Upgrade Concepts

Do not treat upgrades as:

``` bash
apt upgrade
```

A production Kubernetes upgrade requires planning.

Understand:

-   Kubernetes version skew rules
-   Control-plane upgrade
-   Worker-node upgrade
-   kubelet version
-   kubectl compatibility
-   CNI compatibility
-   CSI compatibility
-   Ingress/controller compatibility
-   CRD compatibility
-   Application compatibility
-   Backup before upgrade
-   Drain/uncordon strategy

Typical concept:

``` text
Backup
  |
Compatibility check
  |
Control plane
  |
Workers
  |
Add-ons
  |
Application validation
```

Always consult the version-specific Kubernetes documentation before a
real upgrade.

------------------------------------------------------------------------

# 102. Kubernetes Backup and Disaster Recovery

Know what needs protection.

Important cluster state includes:

-   etcd
-   Kubernetes manifests/configuration
-   Persistent application data
-   Secrets
-   External dependencies

A backup is not enough.

You need:

``` text
Backup
  +
Restore test
  =
Useful DR capability
```

For stateful applications, application-level backups may also be
required.

------------------------------------------------------------------------

# 103. High Availability

For production:

``` text
Multiple control-plane nodes
        +
Multiple worker nodes
        +
Reliable datastore
        +
Load-balanced API access
```

For applications:

``` text
Multiple replicas
        +
Distributed placement
        +
PDB
        +
Health probes
```

High availability is a system design, not simply "3 replicas."

------------------------------------------------------------------------

# 104. Capacity Planning

Understand:

``` text
Cluster capacity
   =
Node resources
   -
System reservations
   -
DaemonSets
   -
Platform overhead
```

Plan for:

-   CPU
-   Memory
-   Storage
-   Network
-   Pod density
-   Failure scenarios
-   Headroom

Do not schedule a cluster to 100% theoretical capacity.

------------------------------------------------------------------------

# 105. Troubleshooting Mental Model

Use this sequence:

``` text
1. What changed?
        |
2. Is the object created?
        |
3. Is the Pod scheduled?
        |
4. Is the container starting?
        |
5. Is the application healthy?
        |
6. Is the Pod Ready?
        |
7. Does the Service have endpoints?
        |
8. Does DNS work?
        |
9. Does network policy allow traffic?
        |
10. Does Ingress/Gateway work?
        |
11. Is the external infrastructure working?
```

This avoids randomly running commands.

------------------------------------------------------------------------

# 106. Essential Troubleshooting Commands

``` bash
kubectl get pods -A -o wide

kubectl describe pod <pod>

kubectl logs <pod>

kubectl logs <pod> --previous

kubectl get events -A --sort-by=.lastTimestamp

kubectl get svc

kubectl describe svc <service>

kubectl get endpoints

kubectl get endpointslices

kubectl get nodes

kubectl describe node <node>

kubectl top nodes

kubectl top pods

kubectl get pvc

kubectl describe pvc <pvc>

kubectl get ingress

kubectl describe ingress <name>

kubectl get networkpolicy -A
```

------------------------------------------------------------------------

# 107. Common Failure Patterns

  Symptom                   First checks
  ------------------------- ---------------------------------------
  Pending                   `describe pod`, scheduler events
  CrashLoopBackOff          logs, previous logs, probes
  ImagePullBackOff          image/tag/registry/auth
  OOMKilled                 memory usage and limits
  Service no traffic        selector, readiness, EndpointSlices
  DNS failure               CoreDNS, Pod DNS, CNI
  Mount failure             PVC, StorageClass, CSI
  Node NotReady             kubelet, runtime, node resources
  Ingress failure           controller, Service, DNS, TLS
  Pod cannot communicate    CNI, NetworkPolicy, routes
  Pod stuck Terminating     finalizers, kubelet, volume detach
  Deployment not updating   image/tag, ReplicaSet, rollout status
  HPA not scaling           metrics, requests, HPA config

------------------------------------------------------------------------

# 108. Must-Know Commands Cheat Sheet

## Cluster

``` bash
kubectl cluster-info
kubectl get nodes
kubectl get ns
kubectl get pods -A
```

## Workloads

``` bash
kubectl get deploy
kubectl get rs
kubectl get pods
kubectl get ds
kubectl get sts
kubectl get jobs
kubectl get cronjobs
```

## Services

``` bash
kubectl get svc
kubectl get endpoints
kubectl get endpointslices
kubectl get ingress
```

## Storage

``` bash
kubectl get pv
kubectl get pvc
kubectl get storageclass
```

## Security

``` bash
kubectl get sa
kubectl get role
kubectl get rolebinding
kubectl get clusterrole
kubectl get clusterrolebinding
kubectl auth can-i get pods
```

## Debugging

``` bash
kubectl describe <resource> <name>
kubectl logs <pod>
kubectl exec -it <pod> -- sh
kubectl get events -A
```

## YAML

``` bash
kubectl apply -f file.yaml
kubectl delete -f file.yaml
kubectl diff -f file.yaml
kubectl get <resource> <name> -o yaml
kubectl explain <resource>
```

------------------------------------------------------------------------

# 109. Useful `kubectl` Output Options

``` bash
kubectl get pods -o wide
kubectl get pods -o yaml
kubectl get pods -o json
kubectl get pods --show-labels
kubectl get pods -l app=web
```

JSONPath is useful for scripting:

``` bash
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
```

------------------------------------------------------------------------

# 110. Practical YAML Patterns You Should Be Able to Write

As a 6-year DevOps engineer, you should be comfortable writing and
modifying:

-   Namespace
-   Deployment
-   Service
-   ConfigMap
-   Secret
-   Ingress
-   ServiceAccount
-   Role
-   RoleBinding
-   NetworkPolicy
-   PVC
-   Job
-   CronJob
-   HPA
-   PDB

You do not need to memorize every field.

You should know how to find the correct API and validate the manifest.

------------------------------------------------------------------------

# 111. YAML Validation Workflow

Before applying:

``` bash
kubectl apply --dry-run=client -f app.yaml
```

For server-side validation when appropriate:

``` bash
kubectl apply --dry-run=server -f app.yaml
```

Then:

``` bash
kubectl diff -f app.yaml
kubectl apply -f app.yaml
```

------------------------------------------------------------------------

# 112. Declarative Management Rules

Prefer:

``` bash
kubectl apply -f ...
```

and Git-managed configuration.

Avoid manually changing production objects with imperative commands
unless there is a deliberate operational reason.

After emergency changes:

> Make sure the source of truth is updated.

Otherwise Git and the cluster can drift.

------------------------------------------------------------------------

# 113. GitOps Mental Model

Traditional:

``` text
Engineer
  |
kubectl apply
  |
Cluster
```

GitOps:

``` text
Engineer
  |
Git
  |
Argo CD
  |
Cluster
```

Git becomes the desired-state source of truth.

------------------------------------------------------------------------

# 114. What a 6-Year DevOps Engineer Should Be Able to Do

You should be able to:

### Build

-   Write Kubernetes YAML
-   Deploy applications
-   Create Services
-   Configure Ingress/Gateway
-   Manage ConfigMaps and Secrets
-   Manage PVCs
-   Configure probes

### Operate

-   Scale workloads
-   Perform rolling deployments
-   Roll back releases
-   Drain nodes
-   Investigate failed Pods
-   Troubleshoot Services
-   Troubleshoot DNS
-   Troubleshoot storage
-   Troubleshoot nodes

### Secure

-   Configure RBAC
-   Use ServiceAccounts
-   Apply security contexts
-   Understand Pod Security
-   Apply NetworkPolicies
-   Manage secrets securely
-   Use least privilege

### Optimize

-   Set requests/limits
-   Configure HPA
-   Understand cluster autoscaling
-   Improve scheduling
-   Control resource consumption

### Automate

-   Use Helm
-   Use CI/CD
-   Use GitOps/Argo CD
-   Integrate image build/scan/promotion

### Design

-   High availability
-   Multi-zone workloads
-   Stateful workloads
-   Disaster recovery
-   Upgrade strategy
-   Observability
-   Capacity planning

------------------------------------------------------------------------

# 115. What You Do NOT Need to Over-Study

For a 6-year DevOps role, do not spend excessive time memorizing:

-   Every Kubernetes API field
-   Every obscure controller
-   Kubernetes source code
-   etcd internals
-   Scheduler source code
-   Writing a CNI from scratch
-   Writing a CSI driver
-   Writing a custom Kubernetes runtime
-   Every CRD in the ecosystem
-   Every service mesh feature
-   Every Gateway API field
-   Rare networking implementations

You should understand the architecture and troubleshoot real systems.

------------------------------------------------------------------------

# 116. Kubernetes Learning Order

Recommended practical order:

``` text
1. Kubernetes architecture
        ↓
2. kubectl
        ↓
3. Pods
        ↓
4. YAML
        ↓
5. Deployments / ReplicaSets
        ↓
6. Services
        ↓
7. ConfigMaps / Secrets
        ↓
8. Volumes / PV / PVC
        ↓
9. Probes
        ↓
10. Requests / Limits
        ↓
11. Ingress
        ↓
12. Scheduling
        ↓
13. RBAC / Security
        ↓
14. NetworkPolicy
        ↓
15. HPA
        ↓
16. Troubleshooting
        ↓
17. Observability
        ↓
18. Production operations
        ↓
19. Helm
        ↓
20. Terraform + Kubernetes
        ↓
21. CI/CD
        ↓
22. Argo CD / GitOps
```

------------------------------------------------------------------------

# 117. Recommended Hands-On Labs

## Lab 1 --- Basic Pod

-   Create Pod
-   Inspect Pod
-   View logs
-   Exec into container
-   Delete Pod

## Lab 2 --- Deployment

-   Create Deployment
-   Scale
-   Update image
-   Roll back

## Lab 3 --- Service

-   Deploy nginx
-   Create ClusterIP
-   Test from another Pod
-   Create NodePort

## Lab 4 --- ConfigMap / Secret

-   Inject environment variables
-   Mount configuration files
-   Use Secret

## Lab 5 --- Storage

-   Create PVC
-   Mount it
-   Verify persistence

## Lab 6 --- Probes

-   Configure readiness
-   Configure liveness
-   Break the application intentionally
-   Observe behavior

## Lab 7 --- Scheduling

-   Labels
-   nodeSelector
-   Affinity
-   Taints/tolerations

## Lab 8 --- Security

-   ServiceAccount
-   Role
-   RoleBinding
-   `kubectl auth can-i`
-   SecurityContext

## Lab 9 --- NetworkPolicy

``` text
frontend -> backend -> database
```

Allow only required traffic.

## Lab 10 --- HPA

-   Install metrics support
-   Deploy workload
-   Configure HPA
-   Generate load
-   Observe scaling

## Lab 11 --- Troubleshooting

Intentionally create:

-   Bad image
-   Wrong Service selector
-   Broken probe
-   Missing ConfigMap
-   Missing PVC
-   DNS issue

Then diagnose using only:

``` bash
get
describe
logs
events
exec
```

## Lab 12 --- Production Deployment

Build one application with:

``` text
Deployment
Service
ConfigMap
Secret
PVC
Probes
Resources
Ingress
HPA
PDB
NetworkPolicy
RBAC
```

This should become a portfolio project.

------------------------------------------------------------------------

# 118. Mental Models to Remember

## Kubernetes

``` text
Declare desired state
        ↓
API Server
        ↓
Controllers/Scheduler
        ↓
Kubelet
        ↓
Container Runtime
        ↓
Containers
```

## Application traffic

``` text
User
 ↓
Load Balancer / Ingress / Gateway
 ↓
Service
 ↓
Ready Pods
 ↓
Container
```

## Storage

``` text
Pod
 ↓
PVC
 ↓
PV
 ↓
Storage backend
```

## Configuration

``` text
Application
 ↓
ConfigMap / Secret
```

## Security

``` text
Identity
 ↓
Authentication
 ↓
Authorization / RBAC
 ↓
Admission / Policy
 ↓
Pod Security
```

## GitOps

``` text
Git
 ↓
Argo CD
 ↓
Kubernetes
```

------------------------------------------------------------------------

# 119. Final 6-Year DevOps Kubernetes Checklist

Before considering Kubernetes "job ready", make sure you can confidently
explain and use:

-   [ ] Kubernetes architecture
-   [ ] API Server
-   [ ] etcd
-   [ ] Scheduler
-   [ ] Controller Manager
-   [ ] kubelet
-   [ ] Container runtime / CRI
-   [ ] Pods
-   [ ] Namespaces
-   [ ] Labels/selectors
-   [ ] Deployments
-   [ ] ReplicaSets
-   [ ] StatefulSets
-   [ ] DaemonSets
-   [ ] Jobs/CronJobs
-   [ ] Services
-   [ ] ClusterIP
-   [ ] NodePort
-   [ ] LoadBalancer
-   [ ] DNS
-   [ ] Ingress
-   [ ] Gateway API concept
-   [ ] ConfigMaps
-   [ ] Secrets
-   [ ] Volumes
-   [ ] PV/PVC
-   [ ] StorageClass
-   [ ] CNI
-   [ ] NetworkPolicy
-   [ ] Requests/limits
-   [ ] QoS
-   [ ] Probes
-   [ ] Scheduling
-   [ ] Affinity/anti-affinity
-   [ ] Taints/tolerations
-   [ ] Topology spread
-   [ ] RBAC
-   [ ] ServiceAccounts
-   [ ] Security contexts
-   [ ] Pod Security
-   [ ] Admission
-   [ ] Rolling updates
-   [ ] Rollbacks
-   [ ] HPA
-   [ ] VPA concept
-   [ ] Cluster autoscaling concept
-   [ ] ResourceQuota
-   [ ] LimitRange
-   [ ] PDB
-   [ ] Graceful shutdown
-   [ ] Init containers
-   [ ] Sidecars
-   [ ] CRDs
-   [ ] Operators
-   [ ] Observability
-   [ ] Metrics
-   [ ] Logs
-   [ ] Traces
-   [ ] kubeconfig
-   [ ] Node maintenance
-   [ ] Cluster upgrades
-   [ ] Backup/restore
-   [ ] HA
-   [ ] Capacity planning
-   [ ] Production troubleshooting
-   [ ] CI/CD integration
-   [ ] Helm integration
-   [ ] GitOps/Argo CD integration
-   [ ] Managed Kubernetes/EKS concepts

------------------------------------------------------------------------

# 120. The Most Important Principle

Do not learn Kubernetes as a list of YAML files.

Learn it as:

``` text
Desired State
      ↓
API
      ↓
Controllers
      ↓
Scheduling
      ↓
Pods
      ↓
Networking
      ↓
Storage
      ↓
Security
      ↓
Observability
      ↓
Operations
```

If you understand that flow, Kubernetes becomes much easier to
troubleshoot and design.

**Target outcome:** You should be able to deploy, secure, troubleshoot,
operate, scale, upgrade, and explain a production Kubernetes workload
--- not merely write YAML.
