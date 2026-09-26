# Kubernetes — Complete Concept & Command Cheatsheet (5 YOE Level)

Part 1 covers concepts, definitions, YAML, and scenarios. Part 2 is the full `kubectl` command reference.

---

# PART 1 — Concepts

## 1. What Kubernetes Is

Kubernetes (K8s) is a container orchestration platform: it schedules containerized workloads onto
a cluster of machines, keeps the actual running state converging toward a **declared desired
state** (via reconciliation control loops), and handles scaling, self-healing, networking, and
rollout of those workloads without manual intervention.

**Core idea to internalize:** you never tell K8s "restart this pod" — you declare "I want 3
replicas of this spec running" and a controller continuously reconciles reality to match that.
Everything else in K8s is a variation on this control-loop pattern.

## 2. Architecture

**Control Plane (brain of the cluster):**
| Component | Job |
|---|---|
| `kube-apiserver` | Front door — all reads/writes (including from `kubectl`) go through it; validates and persists to etcd |
| `etcd` | Distributed key-value store — the single source of truth for all cluster state |
| `kube-scheduler` | Decides which node a new Pod should run on, based on resources/affinity/taints |
| `kube-controller-manager` | Runs the core control loops (Node controller, ReplicaSet controller, Job controller, etc.) |
| `cloud-controller-manager` | Cloud-provider-specific glue (e.g. provisioning an AWS ELB for a `LoadBalancer` Service) |

**Node (worker) components:**
| Component | Job |
|---|---|
| `kubelet` | Agent on every node — talks to the API server, ensures containers described in PodSpecs are running and healthy |
| `kube-proxy` | Maintains network rules on each node so Services route traffic correctly (iptables/IPVS) |
| Container runtime | containerd/CRI-O — actually pulls images and runs containers, via the CRI (Container Runtime Interface) |

**Scenario prompt:** *"A pod is stuck Pending — walk me through where in this architecture you'd
look."* → Scheduler couldn't place it (check `kubectl describe pod` events for scheduling
failures — insufficient resources, unmatched affinity/taint) → if scheduled but not starting,
it's a kubelet/runtime problem on that node (image pull failure, volume mount failure).

## 2.1 Kubernetes Command Flow

The key rule:

> **`kubectl` talks to the `kube-apiserver`. It does not directly talk to the scheduler, controller-manager, or etcd.**

### Example: `kubectl apply -f pod.yaml`

```text
User
  kubectl apply -f pod.yaml
 
  Authorize (RBAC)
 
 
 
kube-scheduler
  Selects a suitable node
 
 
kubelet
  On the selected node
 
  Create container
 
 
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
kubelet
  
Container Runtime
  
Container terminated
```

If the Pod is managed by a Deployment:

```text
Deployment
   
ReplicaSet
   
Pod deleted
   
ReplicaSet creates replacement Pod
   
Scheduler selects node
   
kubelet
   
Container Runtime
```

---

### The Big Picture

```text
                        kube-apiserver
                       /      |       \
                      /       |        \
                   etcd   scheduler   controllers
                             
                             
                                   
                                   
                                   
                                   
API Server
   Desired State
  
API Server
  
Container Runtime
   kube-apiserver` is the starting point for Kubernetes API operations.

## 3. Pods — the atomic unit

A Pod is the smallest deployable unit — one or more containers that share network namespace
(same IP, can reach each other via `localhost`) and can share storage volumes. You almost never
create bare Pods directly in production; a controller (Deployment, StatefulSet, etc.) manages them.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  containers:
    - name: app
      image: myregistry/my-app:1.2.0
      ports:
        - containerPort: 8080
      resources:
        requests:
          cpu: "250m"
          memory: "256Mi"
        limits:
          cpu: "500m"
          memory: "512Mi"
      env:
        - name: ENVIRONMENT
          value: "production"
```

**Multi-container pod patterns (interview favorite):**
- **Sidecar** — helper container running alongside the main one for its whole lifetime (log
  shipper, service-mesh proxy like Envoy).
- **Init container** — runs to completion *before* app containers start (e.g. wait-for-db,
  run migrations, fetch config). Defined under `spec.initContainers`.
- **Ambassador** — proxies network calls out from the main container (less common now, service
  mesh usually replaces this).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  initContainers:
    - name: wait-for-db                  # Runs to completion FIRST; if it fails, the pod never starts main containers
      image: busybox:1.36
      command: ["sh", "-c", "until nc -z db-service 5432; do echo waiting for db; sleep 2; done"]
    - name: run-migrations                # Init containers run in order, one at a time — this runs only after wait-for-db succeeds
      image: myregistry/my-app-migrate:1.2.0
      command: ["./migrate.sh"]
  containers:
    - name: app                           # Main container — starts only after ALL init containers succeed
      image: myregistry/my-app:1.2.0
      ports: [{ containerPort: 8080 }]
    - name: log-shipper                   # Sidecar — starts alongside "app" and runs for the pod's whole lifetime
      image: fluent/fluent-bit:2.2
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
  volumes:
    - name: logs
      emptyDir: {}
```

**Native sidecar containers (K8s 1.28+, stable in newer versions):** a container listed under
`initContainers` with `restartPolicy: Always` is treated as a true sidecar by the kubelet — it
starts before the main container (like an init container) but keeps running for the pod's whole
life (like a sidecar), and the kubelet knows to terminate it *last* on pod shutdown so it can keep
shipping logs/metrics until the very end. This replaces older hacks (postStart/preStop scripting)
people used to fake sidecar ordering before this feature existed.

```yaml
  initContainers:
    - name: log-shipper
      image: fluent/fluent-bit:2.2
      restartPolicy: Always            # This one flag is what makes it a "native sidecar" instead of a one-shot init container
```

**Pod termination lifecycle (directly tied to "zero-downtime deploy" — know this cold):**
1. API server marks the Pod for deletion; kubelet is notified.
2. Pod is simultaneously removed from Service Endpoints (traffic stops routing to it) **and** sent
   `SIGTERM`.
3. If a `preStop` hook is defined, it runs now — commonly used to sleep a few seconds, covering the
   race between "endpoint removed" propagating to every kube-proxy/load balancer and the container
   actually still being able to finish in-flight requests.
4. Kubelet waits up to `terminationGracePeriodSeconds` (default 30s) for the container to exit
   cleanly after `SIGTERM`.
5. If it hasn't exited by then, kubelet sends `SIGKILL` — a hard kill, no further cleanup possible.

```yaml
      terminationGracePeriodSeconds: 45
      containers:
        - name: app
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 10"]   # Buys time for in-flight requests / LB deregistration before SIGTERM lands
```

**Scenario prompt:** *"Users see a handful of 502s on every deploy despite readinessProbe passing —
why?"* — almost always this exact race: the pod is removed from Endpoints and killed at roughly the
same instant, but the upstream load balancer/ingress hasn't finished propagating the removal yet
and sends one more request to a pod that's already gone. A short `preStop` sleep is the standard
fix — it delays the actual shutdown just long enough for that propagation to catch up.

## 4. Workload Controllers

### Deployment — stateless apps

Manages a ReplicaSet, which manages Pods. Supports rolling updates and rollback history.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # Extra pods allowed above `replicas` during rollout
      maxUnavailable: 0    # Pods allowed to be unavailable during rollout
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: myregistry/my-app:1.2.0
          readinessProbe:
            httpGet: { path: /healthz, port: 8080 }
            initialDelaySeconds: 5
```

**Rollout mechanics:** Deployment → creates a new ReplicaSet → scales new RS up and old RS down
according to `maxSurge`/`maxUnavailable` → old ReplicaSet kept (scaled to 0) for `helm
rollback`/`kubectl rollout undo` history.

### StatefulSet — stateful apps (databases, queues)

Like a Deployment, but pods get **stable network identity** (`<name>-0`, `<name>-1`, ...) and
**stable storage** (each pod keeps its own PVC across restarts/rescheduling). Pods are created
and terminated in order.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless      # Requires a headless Service for stable DNS per pod
  replicas: 3
  selector:
    matchLabels: { app: postgres }
  template:
    metadata:
      labels: { app: postgres }
    spec:
      containers:
        - name: postgres
          image: postgres:15
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:                # Each replica gets its own PVC, auto-generated
    - metadata: { name: data }
      spec:
        accessModes: ["ReadWriteOnce"]
        resources: { requests: { storage: 10Gi } }
```

### DaemonSet — one pod per node

Ensures every (or a filtered subset of) node runs a copy — used for node-level agents: log
collectors (Fluentd), monitoring agents (node-exporter), CNI plugins.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels: { app: node-exporter }
  template:
    metadata:
      labels: { app: node-exporter }
    spec:
      containers:
        - name: node-exporter
          image: prom/node-exporter
```

### Job & CronJob — run-to-completion workloads

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  backoffLimit: 3            # Retries before marking Job failed
  template:
    spec:
      restartPolicy: Never   # Jobs require Never or OnFailure, not Always
      containers:
        - name: migrate
          image: myregistry/migrator:1.0
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *"                # Standard cron syntax
  concurrencyPolicy: Forbid            # Forbid | Allow | Replace overlapping runs
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: myregistry/backup-tool:1.0
```

## 5. Services & Networking

**Service** = a stable virtual IP + DNS name that load-balances traffic to a dynamic set of Pods
selected by label. Solves the problem of pods being ephemeral (new IP every restart).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app                # Routes to any pod with this label
  ports:
    - port: 80                 # Port the Service listens on
      targetPort: 8080          # Port the container actually listens on
  type: ClusterIP               # ClusterIP | NodePort | LoadBalancer | ExternalName
```

| Type | Use case |
|---|---|
| `ClusterIP` (default) | Internal-only, reachable within the cluster |
| `NodePort` | Exposes a static port on every node's IP — mostly for dev/on-prem, rarely prod |
| `LoadBalancer` | Provisions a cloud LB (ELB/NLB on AWS) pointing at the service — standard for public-facing prod services |
| `ExternalName` | Pure DNS CNAME alias to an external service — no proxying |
| `Headless` (`clusterIP: None`) | No virtual IP — DNS returns individual pod IPs directly; required for StatefulSets |

**Ingress** — HTTP(S) routing layer *in front of* Services: host/path-based routing, TLS
termination, all from one entry point instead of one LoadBalancer per service.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts: ["app.example.com"]
      secretName: my-app-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port: { number: 80 }
```

Needs an **Ingress Controller** actually running in the cluster (nginx-ingress, ALB Ingress
Controller, Traefik) — the Ingress object alone is just a routing rule, it does nothing without
a controller watching it.

**NetworkPolicy** — firewall rules between pods. Default in K8s is **all pods can talk to all
pods** — NetworkPolicy is how you lock that down (default-deny + explicit allow is best practice).

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-then-allow-frontend
spec:
  podSelector:
    matchLabels: { app: backend }
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - podSelector:
            matchLabels: { app: frontend }
      ports:
        - port: 8080
```

**DNS:** every Service gets `<service>.<namespace>.svc.cluster.local`; every Pod (with a headless
Service) gets `<pod-name>.<service>.<namespace>.svc.cluster.local`. CoreDNS is what resolves this.

**EndpointSlices** — the modern backing object behind a Service's routing, replacing the older
single `Endpoints` object at scale. A Service with thousands of backing pods produces one giant
`Endpoints` object that gets fully rewritten on every pod change (expensive); `EndpointSlices`
shard that into multiple smaller objects instead, so kube-proxy/CoreDNS only need to process the
slice that actually changed. You won't hand-write these, but `kubectl get endpointslices` is worth
knowing as the more scalable equivalent of `kubectl get endpoints` on a large cluster.

## 6. Storage

| Object | Definition |
|---|---|
| **Volume** | Storage attached to a Pod's lifecycle — types include `emptyDir` (scratch space, dies with pod), `hostPath` (node's filesystem, rarely safe in prod), `configMap`/`secret` (mount config as files), `persistentVolumeClaim` |
| **PersistentVolume (PV)** | Cluster-wide storage resource, provisioned either statically by an admin or dynamically via a StorageClass |
| **PersistentVolumeClaim (PVC)** | A Pod's *request* for storage — binds to a matching PV. Pods reference PVCs, never PVs directly |
| **StorageClass** | Defines *how* to dynamically provision a PV on demand (which cloud disk type, reclaim policy) |

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes: ["ReadWriteOnce"]     # RWO | ROX | RWX — RWX support depends on the storage backend
  storageClassName: gp3
  resources:
    requests:
      storage: 20Gi
```

`reclaimPolicy: Retain` vs `Delete` on the StorageClass/PV — `Delete` (common cloud default)
destroys the underlying disk when the PVC is deleted; `Retain` keeps it around for manual recovery
— important distinction to know before touching a prod StatefulSet's storage.

## 7. Configuration — ConfigMap, Secret, Downward API

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
  config.yaml: |
    feature_flags:
      new_ui: true
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQ=      # base64-encoded — NOT encrypted, just encoded. Treat as sensitive.
```

Consuming them in a Pod:

```yaml
      envFrom:
        - configMapRef: { name: app-config }
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef: { name: app-secret, key: DB_PASSWORD }
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap: { name: app-config }
```

**Critical fact:** `Secret` data is base64-**encoded**, not encrypted. At rest in etcd it's only
encrypted if you've explicitly enabled [encryption at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/).
Anyone with `get secrets` RBAC access can trivially decode it. Real secret management usually
layers External Secrets Operator / Sealed Secrets / Vault on top.

**Downward API** — expose Pod/container metadata (name, namespace, labels, resource limits) to
the container as env vars or files, without hardcoding:

```yaml
      env:
        - name: POD_NAME
          valueFrom: { fieldRef: { fieldPath: metadata.name } }
        - name: POD_NAMESPACE
          valueFrom: { fieldRef: { fieldPath: metadata.namespace } }
```

## 8. Scheduling Controls

**Resource requests & limits:**
- `requests` = what the scheduler guarantees/reserves when placing the pod on a node — the
  scheduler will only place a pod on a node that has at least this much *unreserved* capacity left.
- `limits` = hard ceiling; CPU gets throttled past this, memory past this gets the container
  **OOMKilled**.
- Omitting limits risks a single pod starving a node ("noisy neighbor"); omitting requests makes
  scheduling unpredictable.

**Quality of Service (QoS) classes** — derived automatically from how requests/limits are set,
never set directly. This matters because it decides **eviction order** when a node runs low on
resources.

| Class | How you get it | Eviction priority |
|---|---|---|
| `Guaranteed` | Every container has `requests == limits` for both CPU and memory | Evicted **last** |
| `Burstable` | At least one container has requests/limits set, but they don't all match | Evicted **before** Guaranteed |
| `BestEffort` | No requests or limits set at all, on any container | Evicted **first** |

```yaml
      resources:
        requests: { cpu: "500m", memory: "512Mi" }
        limits:   { cpu: "500m", memory: "512Mi" }   # Identical to requests → this pod is Guaranteed QoS
```

**Node-pressure eviction** — a *separate mechanism* from OOMKill, and one of the most commonly
conflated pairs in interviews:
- **OOMKilled** = a single container exceeded *its own* memory `limit` → that container is killed
  and (usually) restarted by the kubelet, in place, on the same node.
- **Node-pressure eviction** = the *node itself* is running low on memory/disk/PIDs overall → the
  kubelet proactively evicts whole pods (lowest QoS class first — BestEffort, then Burstable, then
  Guaranteed only as a last resort) to protect the node, and those pods get rescheduled elsewhere
  by their controller.

`kubectl describe node` shows `MemoryPressure`/`DiskPressure`/`PIDPressure` conditions — that's
where you'd see this happening before it shows up as pods disappearing and reappearing on other
nodes.

**Node affinity / anti-affinity — what the fields actually mean:**

Node affinity attracts pods *toward* nodes matching a label; pod (anti-)affinity attracts or repels
pods based on *other pods already running*, not node labels directly.

```yaml
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:   # "required" = hard rule, pod won't schedule at all if unmet
            nodeSelectorTerms:
              - matchExpressions:
                  - key: node-type
                    operator: In               # In | NotIn | Exists | DoesNotExist | Gt | Lt
                    values: ["compute-optimized"]
        podAntiAffinity:                # Spread replicas across nodes/zones — HA best practice
          preferredDuringSchedulingIgnoredDuringExecution:  # "preferred" = soft rule, scheduler tries but won't refuse to schedule
            - weight: 100                                   # Relative weight among multiple "preferred" terms (1-100)
              podAffinityTerm:
                labelSelector:
                  matchLabels: { app: my-app }   # Match against OTHER PODS' labels, not node labels
                topologyKey: "kubernetes.io/hostname"   # The "unit" of spreading — see below
```

Field-by-field, in plain terms:
- **`required...` vs `preferred...`** — required is a hard constraint (like an `AND` the scheduler
  must satisfy, or the pod stays `Pending`); preferred is a scoring hint (scheduler tries, but will
  still place the pod somewhere if it can't be satisfied).
- **`...IgnoredDuringExecution`** — the suffix on every current affinity rule; means the rule is
  only checked *at scheduling time*. If a node's labels change afterward (or pods it was scheduled
  near move away), an already-running pod is **not** evicted to re-satisfy the rule.
- **`operator`** — `In`/`NotIn` compare against a list of values; `Exists`/`DoesNotExist` just check
  the label key is present (no value comparison); `Gt`/`Lt` for numeric label comparisons.
- **`topologyKey`** — this is the part people usually can't explain clearly: it defines what
  "spread" even means, by picking a node label to group nodes into buckets. `kubernetes.io/hostname`
  means "one bucket per node" (spread across individual machines); `topology.kubernetes.io/zone`
  means "one bucket per AZ" (spread across availability zones) — same affinity rule, completely
  different blast-radius guarantee depending on which topologyKey you pick.
- **`podAffinity` vs `podAntiAffinity`** — affinity pulls a pod *toward* nodes already running pods
  matching the selector (e.g. co-locate a cache next to the app that uses it, same zone, lower
  latency); anti-affinity pushes it *away* from them (the HA use case — don't stack all replicas of
  the same app on one node/zone).

**`topologySpreadConstraints`** — the more precise, purpose-built alternative to `podAntiAffinity`
for even distribution, and usually the better answer when asked how to spread replicas across
zones in a modern cluster:

```yaml
      topologySpreadConstraints:
        - maxSkew: 1                                    # Max allowed difference in pod count between the busiest and quietest zone
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule               # DoNotSchedule (hard) | ScheduleAnyway (soft, like "preferred")
          labelSelector:
            matchLabels: { app: my-app }
```

Why prefer this over `podAntiAffinity` in practice: anti-affinity only expresses "avoid" — it
doesn't guarantee *even* distribution, just that pods aren't co-located past what the rule says.
`topologySpreadConstraints` directly targets an even count per zone/node via `maxSkew`, which is
what you actually want for HA, and is easier to reason about at scale (dozens of replicas across
several zones).

**Taints & tolerations** — inverse of affinity: a **taint on a node** repels pods unless the pod
has a matching **toleration**. Used to dedicate nodes (e.g. GPU nodes, spot instances) to specific
workloads.

```bash
kubectl taint nodes node1 gpu=true:NoSchedule
```
```yaml
      tolerations:
        - key: "gpu"
          operator: "Equal"
          value: "true"
          effect: "NoSchedule"
```

**Affinity vs. taints/tolerations — the distinction interviewers probe:** affinity/anti-affinity is
the *pod* expressing a preference about where it wants to go; taints/tolerations is the *node*
expressing a restriction about what's allowed to land on it. They're often combined: taint the GPU
nodes so nothing schedules there by accident (toleration required to even be considered), **then**
add node affinity on the GPU workload so it actively prefers those nodes (rather than merely being
allowed to land elsewhere too) — toleration alone doesn't attract a pod, it only permits it.

**PodDisruptionBudget (PDB)** — guarantees a minimum number/percentage of pods stay up during
*voluntary* disruptions (node drains, cluster upgrades) — does not protect against node crashes.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2            # or maxUnavailable: 1
  selector:
    matchLabels: { app: my-app }
```

## 9. Scaling

| Mechanism | What it scales | Trigger |
|---|---|---|
| **HPA** (Horizontal Pod Autoscaler) | Number of pod replicas | CPU/memory %, or custom metrics (via Prometheus adapter) |
| **VPA** (Vertical Pod Autoscaler) | A pod's resource requests/limits | Historical usage — recommends/auto-applies right-sizing |
| **Cluster Autoscaler / Karpenter** | Number of *nodes* | Unschedulable pods (scale up) or underutilized nodes (scale down) |

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 70 }
```

**Scenario prompt:** *"HPA isn't scaling up despite high CPU — what do you check?"* → metrics-server
installed and reachable → `kubectl top pods` returning data → the Deployment actually has resource
`requests` set (HPA percentage math needs a baseline to calculate against) → `kubectl describe hpa`
for scaling-decision events.

## 10. RBAC & Security

**ServiceAccount** — identity for a *process* (a pod), as opposed to a User (a human). Every pod
runs as some ServiceAccount (`default` if unspecified).

```yaml
apiVersion: v1
kind: ServiceAccount
metadata: { name: my-app-sa, namespace: prod }
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role                       # Namespace-scoped. Use ClusterRole for cluster-wide permissions.
metadata: { name: pod-reader, namespace: prod }
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding                # Binds a Role to a subject. ClusterRoleBinding for cluster-wide.
metadata: { name: read-pods, namespace: prod }
subjects:
  - kind: ServiceAccount
    name: my-app-sa
    namespace: prod
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**`Role` vs `ClusterRole`, `RoleBinding` vs `ClusterRoleBinding`** — the classic RBAC interview
question. Role/RoleBinding = scoped to one namespace. ClusterRole/ClusterRoleBinding = cluster-wide
(or reusable across namespaces via a RoleBinding referencing a ClusterRole — common pattern to
avoid duplicating the same permission set as a Role in every namespace).

**SecurityContext** — pod/container-level hardening:

```yaml
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        readOnlyRootFilesystem: true
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
```

**Pod Security Standards** (replaced the deprecated PodSecurityPolicy) — enforced via namespace
labels at three levels: `privileged`, `baseline`, `restricted`.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

## 11. Health Checks — Probes

| Probe | Purpose | Failure behavior |
|---|---|---|
| `livenessProbe` | Is the container alive/healthy right now? | Kubelet **restarts** the container |
| `readinessProbe` | Is the container ready to receive traffic? | Pod **removed from Service endpoints** (not restarted) — traffic stops routing to it |
| `startupProbe` | Has a slow-starting app finished booting? | Disables liveness/readiness checks until this passes, preventing premature kills of slow-starting apps |

```yaml
      livenessProbe:
        httpGet: { path: /healthz, port: 8080 }
        initialDelaySeconds: 10
        periodSeconds: 10
        failureThreshold: 3
      readinessProbe:
        httpGet: { path: /ready, port: 8080 }
        periodSeconds: 5
      startupProbe:
        httpGet: { path: /healthz, port: 8080 }
        failureThreshold: 30
        periodSeconds: 10
```

**Common mistake to flag in an interview:** using the same endpoint/logic for liveness and
readiness. If a pod is temporarily overloaded and fails readiness (correctly pulled from traffic),
it shouldn't *also* fail liveness and get restarted — that turns a transient slowdown into a
crash-loop.

## 12. Namespaces, Quotas & Limits

```yaml
apiVersion: v1
kind: ResourceQuota
metadata: { name: prod-quota, namespace: prod }
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    pods: "50"
---
apiVersion: v1
kind: LimitRange
metadata: { name: default-limits, namespace: prod }
spec:
  limits:
    - default: { cpu: "500m", memory: "512Mi" }        # Applied if a pod doesn't specify limits
      defaultRequest: { cpu: "250m", memory: "256Mi" }  # Applied if a pod doesn't specify requests
      type: Container
```

## 13. CRDs & Operators

A **CustomResourceDefinition** extends the K8s API with your own resource kind (e.g.
`kind: PostgresCluster`). An **Operator** is a custom controller that watches those custom
resources and reconciles real infrastructure to match — encoding operational knowledge (backup,
failover, upgrade procedure) into code instead of a runbook. This is exactly the mechanism ArgoCD,
cert-manager, and Prometheus Operator are built on.

## 14. Admission Control & Policy Enforcement

Admission controllers intercept requests to the API server **after** auth/RBAC but **before**
the object is persisted to etcd — this is how you enforce cluster-wide policy beyond what RBAC
covers (RBAC says *who* can act; admission control says *what* the object they create is allowed
to look like).

| Type | Job |
|---|---|
| `ValidatingWebhookConfiguration` | Can only accept/reject a request — no mutation |
| `MutatingWebhookConfiguration` | Can rewrite the object before it's stored (e.g. inject a sidecar, set defaults) |

Mutating webhooks run before validating ones, since a mutation might be needed to pass validation.

**OPA Gatekeeper** and **Kyverno** are the two dominant policy engines built on this mechanism —
you write policy-as-code instead of a custom webhook server yourself.

```yaml
# Kyverno example: block any Pod that doesn't set resource limits
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
spec:
  validationFailureAction: Enforce      # Enforce (block) vs Audit (log only, don't block)
  rules:
    - name: check-limits
      match:
        resources:
          kinds: ["Pod"]
      validate:
        message: "CPU and memory limits are required"
        pattern:
          spec:
            containers:
              - resources:
                  limits:
                    cpu: "?*"
                    memory: "?*"
```

**Scenario prompt:** *"How do you stop teams from deploying `:latest` image tags or containers
running as root, cluster-wide, without reviewing every PR?"* → a Kyverno/Gatekeeper policy in
`Enforce` mode is the answer — codify it once, applies to every namespace automatically, and you
can run it in `Audit` mode first to see what would break before flipping to enforce.

## 15. Service Mesh Basics

A service mesh (Istio, Linkerd) adds a sidecar proxy (usually Envoy) next to every pod that
transparently handles service-to-service traffic — without changing application code. What it
buys you that plain K8s Services/NetworkPolicy don't:

- **mTLS between services automatically** — every pod-to-pod call encrypted and authenticated,
  policy-enforced, without the app knowing.
- **Fine-grained traffic control** — canary/weighted routing (e.g. 5% of traffic to a new
  version), retries, timeouts, circuit-breaking — all declarative, outside app code.
- **Deep observability** — golden-signal metrics (latency, traffic, errors) for every service call
  for free, since every call passes through the sidecar proxy.

Trade-off to be honest about in an interview: real added latency per hop and real operational
complexity (another control plane to run and upgrade) — it's justified once you have enough
services that manual mTLS/retry logic per-service becomes unmanageable, not from day one.

## 16. Cluster Upgrades & Control Plane Operations

**Self-managed cluster upgrade order (kubeadm-style clusters):** control plane first, one node at
a time, then worker nodes — never skip more than one minor version at a time (K8s only supports
upgrading one minor version per step, e.g. 1.28 → 1.29, not 1.28 → 1.30 directly).

```bash
kubeadm upgrade plan                    # Check what versions are available/safe
kubeadm upgrade apply v1.29.0           # Upgrade the first control-plane node
kubectl drain <node> --ignore-daemonsets  # Drain before upgrading kubelet on that node
kubeadm upgrade node                    # Run on remaining control-plane nodes
# then upgrade kubelet + kubectl package on the node, restart kubelet, uncordon
```

**etcd backup & restore** — etcd *is* your cluster's state; losing it without a backup means
losing every object definition (though not the running containers themselves, briefly).

```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored
```

**On managed control planes (EKS/GKE/AKS)** you don't do any of the above — the cloud provider
upgrades and backs up the control plane/etcd for you. What you *do* still own:
1. Bumping the EKS control-plane version (one minor version at a time, same constraint applies).
2. Upgrading node groups to match — a control plane can run up to a few minor versions ahead of
   its nodes for a limited window, but drift too far and pods can fail to schedule/run.
3. Checking add-on compatibility (VPC CNI, CoreDNS, kube-proxy versions) against the new control
   plane version before/after the bump.
4. Testing workloads against deprecated/removed API versions **before** upgrading — this is the
   step people skip and then get paged for (e.g. a manifest still using a `apiVersion` removed in
   the target version).

```bash
kubectl convert -f old-manifest.yaml --output-version apps/v1   # Requires kubectl-convert plugin
```

**Scenario prompt:** *"You're told to upgrade a prod EKS cluster two minor versions behind — what's
your plan?"* → check deprecated API usage across all manifests first (`pluto` or `kubent` tools,
or `kubectl deprecations`) → upgrade one minor version at a time, control plane before nodes →
upgrade managed add-ons (VPC CNI, CoreDNS, kube-proxy) to versions compatible with the new control
plane after each step → validate workloads on a staging cluster of the target version before
touching prod, repeated per minor-version hop.

## 17. AWS EKS-Specific Concepts

**IRSA (IAM Roles for Service Accounts)** — how pods get scoped AWS permissions instead of every
pod on a node sharing the node's IAM role. EKS runs an OIDC identity provider; a ServiceAccount is
annotated with an IAM role ARN, and the AWS SDK inside the pod automatically exchanges a projected
service-account token for temporary AWS credentials for *that specific role* — least-privilege,
per-workload, no shared node-wide permissions.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: prod
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/my-app-role
```

```bash
eksctl utils associate-iam-oidc-provider --cluster <cluster> --approve   # One-time cluster setup for IRSA
eksctl create iamserviceaccount \
  --cluster <cluster> --namespace prod --name my-app-sa \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
```

**EKS Pod Identity** — the newer, AWS-recommended alternative to IRSA (EKS 1.24+). Same goal
(scoped per-pod AWS credentials), simpler setup: no OIDC provider association needed, no
per-role trust-policy conditions to hand-write — just install the `eks-pod-identity-agent`
add-on and associate a role directly with a ServiceAccount via the API.

```bash
aws eks create-addon --cluster-name <cluster> --addon-name eks-pod-identity-agent   # One-time, replaces OIDC association
aws eks create-pod-identity-association \
  --cluster-name <cluster> --namespace prod --service-account my-app-sa \
  --role-arn arn:aws:iam::123456789012:role/my-app-role
```

Practical takeaway if asked to compare: **IRSA** is the pattern to know for existing/older
clusters and for understanding *why* OIDC federation works the way it does; **Pod Identity** is
what you'd reach for by default on a newer cluster because it removes a whole category of trust-
policy misconfiguration. Both can coexist in the same cluster during a migration.

**EBS/EFS CSI Drivers** — CSI (Container Storage Interface) is the standard plugin interface
letting Kubernetes talk to any storage backend without built-in code per vendor. On EKS:
- **EBS CSI driver** → dynamic `ReadWriteOnce` block storage (most stateful workloads, databases).
- **EFS CSI driver** → `ReadWriteMany` shared filesystem (multiple pods, multiple AZs, same volume).

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: { name: ebs-gp3 }
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
volumeBindingMode: WaitForFirstConsumer   # Delays provisioning until a pod is actually scheduled — avoids AZ mismatch
```

The CSI driver itself needs its own IRSA role to call the EC2/EFS APIs — a common first EKS setup
mistake is forgetting this and getting PVCs stuck in `Pending` with an opaque provisioning error.

**VPC CNI** — EKS's default networking plugin. Distinct from most K8s distros in that **pods get
real VPC IP addresses**, drawn from ENIs (Elastic Network Interfaces) attached to each node — not
an overlay network. This means pod IP capacity is bounded by instance type (ENI/IP limits per EC2
instance size) — a real, frequently-hit limit at scale, and the reason `prefix delegation` mode
exists (assigns IP prefixes instead of individual IPs, raising the pods-per-node ceiling).

**AWS Load Balancer Controller** — watches Ingress/Service objects and provisions real AWS ALBs
(for Ingress) or NLBs (for `type: LoadBalancer` Services) — the AWS-native alternative/successor
to the older in-tree "Classic" ELB provisioning and to community nginx-ingress for L7 use cases.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: my-app, port: { number: 80 } }
```

**Node provisioning: Managed Node Groups vs Self-Managed vs Fargate:**

| Option | Who patches/manages nodes | Notes |
|---|---|---|
| Managed Node Group | AWS handles node provisioning/lifecycle, you handle AMI/upgrade timing | Most common default choice |
| Self-managed nodes | You handle everything (ASG, AMI, patching) | More control, more ops overhead |
| Fargate profile | No nodes at all — AWS runs each pod in its own micro-VM | No DaemonSets, no privileged containers, no `hostNetwork`/`hostPort`, higher per-pod cost, zero node patching |

**Karpenter vs Cluster Autoscaler** — Karpenter (AWS's newer autoscaler) provisions nodes directly
from EC2 APIs based on unschedulable pod requirements (right-sized instance chosen per workload,
nodes ready in roughly a minute), vs Cluster Autoscaler scaling predefined ASGs/node groups up and
down (typically several minutes per scale-up, since it goes through the ASG lifecycle) — Karpenter
is the current AWS-recommended default for new EKS clusters.

```yaml
apiVersion: karpenter.sh/v1                  # Current stable API — the older v1alpha5 Provisioner CRD is gone
kind: NodePool
metadata: { name: default }
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]        # Mix spot + on-demand in one pool
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized   # Actively bin-packs and removes underused nodes, not just empty ones
    consolidateAfter: 30s
  limits:
    cpu: "100"          # Hard ceiling on total capacity this NodePool can provision — a safety rail
---
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass                             # Defines the actual AWS-side details: AMI, subnets, security groups
metadata: { name: default }
spec:
  amiFamily: AL2023
  role: KarpenterNodeRole                      # IAM role for the nodes themselves (not the Karpenter controller)
  subnetSelectorTerms:
    - tags: { karpenter.sh/discovery: <cluster-name> }
  securityGroupSelectorTerms:
    - tags: { karpenter.sh/discovery: <cluster-name> }
```

**Scenario prompt:** *"Pods are stuck Pending, Karpenter is installed, but no new node appears —
what do you check?"* → `kubectl describe pod` first, to confirm it's actually a capacity problem
and not an affinity/taint mismatch → check the NodePool's `requirements` actually permit an
instance type that fits the pod (a NodePool scoped too narrowly silently can't satisfy the
request) → check `limits` on the NodePool hasn't already been hit → check the Karpenter controller
pod's own logs for EC2 API errors (commonly an IAM permission or subnet/security-group tagging
issue on the `EC2NodeClass`).

**`aws-auth` ConfigMap** (or the newer EKS Access Entries API) — maps IAM users/roles to Kubernetes
RBAC identities; this is *the* thing that breaks "I can't see any resources, `kubectl get pods`
returns nothing/unauthorized" right after cluster creation, since IAM authentication and K8s RBAC
are two separate systems that this bridges.

```bash
aws eks update-kubeconfig --name <cluster> --region <region>   # Get IAM-authenticated kubeconfig
kubectl get configmap aws-auth -n kube-system -o yaml           # Legacy mapping mechanism
aws eks list-access-entries --cluster-name <cluster>            # Newer Access Entries API
aws eks create-access-entry --cluster-name <cluster> --principal-arn <iam-role-arn>
```

**EKS Add-ons** — VPC CNI, CoreDNS, kube-proxy, EBS CSI driver can all be managed as first-class
"EKS add-ons" (versioned, upgraded by AWS) instead of self-managed manifests — check add-on
version compatibility explicitly before/after any control-plane upgrade (section 16).

```bash
aws eks list-addons --cluster-name <cluster>
aws eks describe-addon-versions --addon-name vpc-cni --kubernetes-version 1.29
aws eks update-addon --cluster-name <cluster> --addon-name vpc-cni --addon-version <version>
```

**EKS-specific scenario prompt:** *"A pod's PVC is stuck Pending, and it's specifically on EKS —
what's different about your debug path vs vanilla K8s?"* → check the EBS CSI driver pods are
actually running (`kubectl get pods -n kube-system -l app=ebs-csi-controller`) → check the CSI
driver's own IRSA role has the needed EC2 permissions (a silent auth failure looks identical to a
generic provisioning failure) → check `volumeBindingMode` isn't causing an AZ mismatch between the
node the pod's scheduled to and the EBS volume's AZ.

## 18. Troubleshooting Playbook (the part interviewers actually probe hardest)

**Pod stuck `Pending`:**
- `kubectl describe pod` → check Events for `FailedScheduling`.
- Common causes: insufficient CPU/memory on all nodes, unmatched node affinity/taint, PVC not
  bound.

**Pod stuck `CrashLoopBackOff`:**
- `kubectl logs <pod> --previous` — see why the *last* attempt crashed (the current container may
  not have logs yet).
- Check `livenessProbe` isn't failing due to slow startup (add/tune a `startupProbe`).
- Check for `OOMKilled` in `kubectl describe pod` → container exceeded memory `limit`.

**Pod `Running` but Service not reaching it:**
- `kubectl get endpoints <service>` — empty means the Service's label `selector` doesn't match
  any pod's labels, or the pod is failing its `readinessProbe`.
- `kubectl get pods --show-labels` to confirm label match.

**`ImagePullBackOff`:**
- Wrong image tag/typo, private registry auth missing (`imagePullSecrets` not set/wrong), or
  network egress blocked to the registry.

**Node `NotReady`:**
- `kubectl describe node` → check kubelet conditions (disk pressure, memory pressure, network
  unavailable).
- If `kubectl` itself can't reach the node's kubelet at all, drop a level below `kubectl` — SSH to
  the node and use `crictl` (talks directly to containerd, the same interface the kubelet uses)
  rather than `docker`, which isn't present on modern nodes:
  ```bash
  crictl ps                          # List containers as containerd sees them, independent of kubelet/API server state
  crictl logs <container-id>
  crictl inspect <container-id>
  systemctl status kubelet           # Is the kubelet process itself even running/healthy on this node
  journalctl -u kubelet -f
  ```

**Pod stuck `Terminating` forever:**
- Almost always a **finalizer** that never got removed — a finalizer is a marker on an object
  telling the API server "don't actually delete this until some controller finishes cleanup work
  and removes this finalizer itself." If that controller is down, crashed, or was uninstalled
  while resources still reference it, the object hangs in `Terminating` indefinitely.
- Very common with PVCs/PVs (CSI driver has to detach/delete the underlying cloud volume before
  its finalizer clears) — if the CSI driver pod itself is unhealthy, every PVC deletion behind it
  will hang.
- Diagnose: `kubectl get pod <pod> -o yaml | grep -A3 finalizers` to see what's still attached.
- Fix the root cause (get the owning controller healthy again) rather than reaching straight for
  force-removal — `kubectl patch pod <pod> -p '{"metadata":{"finalizers":null}}'` deletes the
  object immediately but skips whatever cleanup that finalizer existed to guarantee (e.g. it can
  leak an orphaned cloud disk that Kubernetes no longer knows about). Treat it as a last resort,
  not a first move.

## 19. Common Interview Questions (rapid recall)

- **"What actually happens when you run `kubectl apply -f deployment.yaml`?"** — client sends the
  manifest to the API server → validated & persisted to etcd → Deployment controller notices the
  diff → creates/updates a ReplicaSet → ReplicaSet controller creates Pods → scheduler assigns
  nodes → kubelet on each node pulls the image and starts containers → kube-proxy/CoreDNS wire up
  networking once the pod is Ready.
- **"Difference between a Deployment and a StatefulSet?"** — identity and storage. Deployment pods
  are interchangeable/stateless; StatefulSet pods have stable names, stable per-pod storage, and
  ordered start/stop.
- **"How does a Service find its pods?"** — label selector match, continuously watched; the set
  of matching, *ready* pod IPs becomes the Service's Endpoints object.
- **"Liveness vs readiness — what's the practical difference in failure behavior?"** — restart vs
  remove-from-traffic. Conflating them causes unnecessary restarts during transient slowness.
- **"How would you zero-downtime deploy a breaking config change?"** — `checksum/config`-style
  annotation (or equivalent) to force a rolling restart in step with the config change, combined
  with `readinessProbe` gating so old pods keep serving until new ones are actually ready.
- **"What's the blast radius difference between Role and ClusterRole misconfiguration?"** — a
  Role mistake is namespace-scoped; a ClusterRole/ClusterRoleBinding mistake can leak permissions
  cluster-wide — much higher stakes to get RBAC review right at the cluster level.
- **"How do you enforce that no pod runs as root, cluster-wide, without relying on every engineer
  remembering to set `securityContext`?"** — admission control (Kyverno/OPA Gatekeeper) in
  `Enforce` mode, tested in `Audit` mode first.
- **"Why would pod IPs run out on an EKS cluster even though the node has plenty of CPU/memory
  headroom?"** — VPC CNI assigns real ENI-backed IPs; each instance type has a hard ENI/IP limit
  independent of compute capacity — the fix is `prefix delegation` mode or a larger instance type,
  not just "add more nodes."
- **"What breaks first if you upgrade an EKS control plane two minor versions without checking
  anything first?"** — workloads using an `apiVersion` removed in the target version fail outright,
  and add-ons (VPC CNI, CoreDNS, kube-proxy) can become incompatible with the new control plane —
  both must be checked before upgrading, not after.

---

# PART 2 — Command Reference

## Cluster & Context

```bash
kubectl cluster-info                              # Control plane / service endpoints
kubectl config get-contexts                       # List available contexts
kubectl config use-context <ctx>                  # Switch context
kubectl config set-context --current --namespace=<ns>   # Set default namespace for current context
kubectl version --short                           # Client/server version
kubectl api-resources                             # List all resource types the API server knows
kubectl explain <resource>.<field>                 # Inline docs for any field, e.g. `deployment.spec.strategy`
```

## Get / Describe / Inspect

```bash
kubectl get pods -n <ns> -o wide
kubectl get all -n <ns>                           # All common resource types in a namespace
kubectl get pods -A                               # Across all namespaces
kubectl get pods --show-labels
kubectl get pods -l app=my-app                    # Filter by label
kubectl get pods --field-selector=status.phase=Running
kubectl describe pod <pod> -n <ns>                # Events, conditions, mounts — first debug step
kubectl get events -n <ns> --sort-by='.lastTimestamp'
kubectl get pod <pod> -o yaml                     # Full manifest as currently applied/running
kubectl get pod <pod> -o jsonpath='{.status.podIP}'
```

## Logs & Exec

```bash
kubectl logs <pod> -n <ns>
kubectl logs -f <pod> -c <container>              # Follow, specific container in a multi-container pod
kubectl logs --previous <pod>                     # Logs from the last crashed instance
kubectl logs -l app=my-app --all-containers=true --prefix   # Aggregate logs across matching pods
kubectl exec -it <pod> -n <ns> -- /bin/sh
kubectl exec <pod> -c <container> -- env
kubectl cp <pod>:/path/in/container ./local/path  # Copy files out of a pod
kubectl debug <pod> -it --image=busybox --target=<container>  # Ephemeral debug container, shares namespaces
kubectl port-forward svc/<svc> 8080:80 -n <ns>
kubectl port-forward pod/<pod> 8080:8080
```

## Create / Apply / Edit / Delete

```bash
kubectl apply -f manifest.yaml                    # Declarative — creates or updates
kubectl apply -f ./manifests/                     # Apply a whole directory
kubectl create -f manifest.yaml                   # Imperative create — fails if it already exists
kubectl delete -f manifest.yaml
kubectl delete pod <pod> --grace-period=0 --force # Force delete a stuck pod
kubectl edit deployment <name> -n <ns>            # Opens live resource in $EDITOR
kubectl patch deployment <name> -p '{"spec":{"replicas":5}}'
kubectl diff -f manifest.yaml                     # Show what apply would change, without applying
kubectl apply -f manifest.yaml --server-side       # Server-side apply — API server owns the merge/conflict resolution instead of the client; increasingly the recommended default over classic client-side apply
kubectl apply -f manifest.yaml --server-side --force-conflicts   # Override another field manager's ownership when conflicts are expected (e.g. taking over a field ArgoCD also manages)
kubectl label pod <pod> tier=frontend
kubectl annotate pod <pod> note="manually patched"
kubectl set image deployment/<name> <container>=<image>:<tag>   # Quick image bump without editing YAML
```

## Deployments & Rollouts

```bash
kubectl rollout status deployment/<name> -n <ns>
kubectl rollout history deployment/<name> -n <ns>
kubectl rollout history deployment/<name> --revision=3
kubectl rollout undo deployment/<name>                    # Rollback to previous revision
kubectl rollout undo deployment/<name> --to-revision=2
kubectl rollout restart deployment/<name>                 # Force a rolling restart (e.g. after Secret change)
kubectl rollout pause deployment/<name>                   # Pause mid-rollout
kubectl rollout resume deployment/<name>
kubectl scale deployment/<name> --replicas=5
kubectl autoscale deployment/<name> --min=2 --max=10 --cpu-percent=70
```

## Nodes & Cluster Maintenance

```bash
kubectl get nodes -o wide
kubectl describe node <node>                       # Conditions, capacity, allocatable, taints
kubectl top nodes                                   # Requires metrics-server
kubectl top pods -n <ns>
kubectl cordon <node>                               # Mark unschedulable (no new pods placed)
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data   # Evict pods before maintenance
kubectl uncordon <node>                             # Mark schedulable again
kubectl taint nodes <node> key=value:NoSchedule
kubectl taint nodes <node> key=value:NoSchedule-    # Trailing `-` removes a taint
```

## RBAC

```bash
kubectl get sa -n <ns>
kubectl get roles,rolebindings -n <ns>
kubectl get clusterroles,clusterrolebindings
kubectl auth can-i create pods --as=system:serviceaccount:<ns>:<sa>    # Test what a ServiceAccount can do
kubectl auth can-i '*' '*' --as=<user>              # Check for cluster-admin-equivalent access
kubectl create rolebinding <name> --clusterrole=view --serviceaccount=<ns>:<sa> -n <ns>
```

## Namespaces, Config, Storage

```bash
kubectl get namespaces
kubectl create namespace <ns>
kubectl create configmap <name> --from-literal=key=value
kubectl create configmap <name> --from-file=path/to/file
kubectl create secret generic <name> --from-literal=key=value
kubectl get secret <name> -o jsonpath='{.data.key}' | base64 -d    # Decode a secret value
kubectl get pvc -n <ns>
kubectl get pv
kubectl get storageclass
kubectl get resourcequota -n <ns>
kubectl describe resourcequota -n <ns>
```

## AWS EKS-Specific Commands

```bash
aws eks update-kubeconfig --name <cluster> --region <region>       # IAM-authenticated kubeconfig
aws eks describe-cluster --name <cluster>
aws eks list-clusters
eksctl create cluster --name <n> --nodes 3
eksctl create nodegroup --cluster <cluster> --name <ng> --nodes 3
eksctl get nodegroup --cluster <cluster>
eksctl utils associate-iam-oidc-provider --cluster <cluster> --approve   # Required once, for IRSA
eksctl create iamserviceaccount --cluster <cluster> --namespace <ns> --name <sa> \
  --attach-policy-arn <policy-arn> --approve

aws eks list-addons --cluster-name <cluster>
aws eks describe-addon-versions --addon-name vpc-cni --kubernetes-version <ver>
aws eks update-addon --cluster-name <cluster> --addon-name vpc-cni --addon-version <ver>

aws eks list-access-entries --cluster-name <cluster>                # Newer IAM<->RBAC mapping
aws eks create-access-entry --cluster-name <cluster> --principal-arn <iam-role-arn>
kubectl get configmap aws-auth -n kube-system -o yaml                # Legacy IAM<->RBAC mapping

aws ecr get-login-password | docker login --username AWS --password-stdin <acct>.dkr.ecr.<region>.amazonaws.com
aws logs tail /aws/eks/<cluster>/cluster --follow                    # Control-plane log tailing
```

## Cluster Upgrade & etcd (self-managed / kubeadm)

```bash
kubeadm upgrade plan
kubeadm upgrade apply v1.29.0
kubeadm upgrade node

ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 --cacert=<ca> --cert=<cert> --key=<key>
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db --data-dir=<new-data-dir>
ETCDCTL_API=3 etcdctl endpoint health --endpoints=https://127.0.0.1:2379
```

## Admission Control / Policy

```bash
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations
kubectl get clusterpolicy                     # Kyverno policies
kubectl get constrainttemplates                # OPA Gatekeeper templates
kubectl get constraints                        # OPA Gatekeeper active constraints
```

## Handy Third-Party Tools (not core kubectl, but daily-driver in practice)

| Tool | What it's for |
|---|---|
| `k9s` | Terminal UI for browsing/acting on cluster resources interactively instead of chained `kubectl` calls |
| `stern` | Tail logs from multiple pods/containers matching a pattern at once, with per-line pod labels |
| `kubectx` / `kubens` | Fast context/namespace switching (shortcuts around `kubectl config use-context`) |
| `pluto` / `kubent` | Scan manifests/cluster for deprecated or removed API versions before an upgrade |
| `kubefwd` | Bulk port-forwarding — local DNS entries for whole namespaces of services at once |

## Debugging Quick Reference

```bash
kubectl get pods -A | grep -v Running               # What's unhealthy cluster-wide, right now
kubectl get events -A --sort-by='.lastTimestamp' | tail -30
kubectl describe pod <pod> -n <ns>                   # Events section explains almost everything
kubectl logs <pod> --previous                        # For CrashLoopBackOff
kubectl get endpoints <service> -n <ns>              # Empty = selector mismatch or no ready pods
kubectl get endpointslices -n <ns>                    # Scalable equivalent of the above on large clusters
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
kubectl get pod <pod> -o jsonpath='{.metadata.finalizers}'   # What's blocking a pod stuck Terminating
kubectl describe node <node>                          # MemoryPressure/DiskPressure/PIDPressure conditions
crictl ps                                             # Below-kubelet view: what containerd itself sees running on a node
crictl logs <container-id>
```

## Useful Output & Filtering Flags

| Flag | Purpose |
|---|---|
| `-o wide` | Extra columns (node, IP) |
| `-o yaml` / `-o json` | Full structured output |
| `-o jsonpath='{...}'` | Extract a specific field |
| `-o name` | Just resource names — good for piping into other commands |
| `-w` / `--watch` | Stream live updates |
| `-l <key>=<value>` | Filter by label |
| `--field-selector` | Filter by a resource field (e.g. `status.phase=Running`) |
| `-A` / `--all-namespaces` | Across every namespace |
| `--context <ctx>` | Run one command against a non-default context, without switching |
| `--dry-run=client -o yaml` | Generate manifest YAML without creating anything — great for scaffolding |
