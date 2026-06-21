# Week 20 — Docker & Kubernetes Deep Dive

**Week of:** October 19, 2026
**Estimated study time:** ~2 hours
**Tags:** `kubernetes` `docker` `containers` `devops`

---

## Overview

Containers have fundamentally changed how software is packaged and operated, but the abstractions they provide often obscure what is actually happening at the kernel level. A senior engineer needs to see through those abstractions — understanding Linux namespaces and cgroups means you can reason about container isolation failures, resource contention, and security boundaries rather than treating the runtime as a black box. This week starts at that foundation before moving up the stack.

Docker's build system is deceptively simple on the surface but has significant performance and security implications in production. Multi-stage builds are the standard pattern for shipping minimal, auditable images, yet they are frequently misused in ways that negate their benefits. You will write a multi-stage Dockerfile modeled on the AESF middleware service, where the difference between a 1.2 GB development image and a 180 MB production image is both a deploy-time win and a meaningful reduction in attack surface.

Kubernetes architecture is covered next with emphasis on the control plane components that most engineers treat as magic. Understanding how etcd, the API server, the scheduler, and the controller manager interact explains why certain failure modes look the way they do — why a node going NotReady does not immediately evict pods, why a rolling update can stall, or why a misconfigured HPA keeps scaling up past its target. These are real operational questions in any GKE-hosted system.

The week closes with Helm chart authoring grounded in the `deployment-manifests` repository you already own. AESF runs three Kubernetes services — module-middleware, module-etl, and module-epic-sdk — each with environment-specific `values.*.yaml` files. Writing and reading Helm charts with that concrete context makes the templating concepts stick and surfaces the gaps that cause staging-to-production drift.

---

## 1. Container Internals: Namespaces and cgroups

Every container is a regular Linux process with a restricted view of the system. The kernel provides two orthogonal mechanisms: **namespaces** isolate what a process can *see*, and **cgroups** limit what it can *use*.

### Linux Namespaces

There are seven namespace types relevant to containers:

| Namespace | Isolates |
|-----------|----------|
| `pid` | Process IDs — container PID 1 is not host PID 1 |
| `net` | Network interfaces, routing tables, iptables rules |
| `mnt` | Mount points and the filesystem tree |
| `uts` | Hostname and domain name |
| `ipc` | System V IPC, POSIX message queues |
| `user` | UID/GID mappings (rootless containers) |
| `cgroup` | cgroup root (prevents container from seeing host hierarchy) |

When Docker runs a container, it calls `clone(2)` with flags like `CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWMNT`. You can inspect a running container's namespaces:

```bash
# Find the host PID of a container process
docker inspect --format '{{.State.Pid}}' <container_id>

# List its namespaces
ls -la /proc/<pid>/ns/
```

### cgroups v2

cgroups enforce resource limits. Key controllers for containers:

- `cpu` — CPU shares and hard quota (`cpu.max`)
- `memory` — `memory.max`, `memory.swap.max`
- `io` — block I/O throttling
- `pids` — maximum number of processes (prevents fork bombs)

Kubernetes translates `resources.requests` and `resources.limits` directly into cgroup settings. A container with `memory.limits: 512Mi` gets `memory.max = 536870912` in its cgroup. When it exceeds that limit the OOM killer fires — this is why you see `OOMKilled` as a container exit reason, not an application crash.

```bash
# Inspect cgroup limits for a Kubernetes pod (on the node)
cat /sys/fs/cgroup/kubepods/burstable/<pod-uid>/<container-id>/memory.max
```

**Common mistake:** Setting `requests` much lower than `limits` (wide QoS class `Burstable`) on AESF middleware pods. Under memory pressure the kubelet will evict Burstable pods before Guaranteed pods. The sync worker, which processes Epic webhook callbacks, should have `requests == limits` to be placed in the Guaranteed QoS class and avoid unexpected eviction during peak load.

---

## 2. Multi-Stage Dockerfiles

A multi-stage build uses multiple `FROM` statements in a single Dockerfile. Each stage is an independent layer graph; you selectively `COPY --from=<stage>` artifacts into the final image, discarding build tools, test dependencies, and intermediate files.

### AESF Middleware Multi-Stage Dockerfile

```dockerfile
# ── Stage 1: dependency resolver ─────────────────────────────────────────────
FROM python:3.13-slim AS deps

WORKDIR /build

# Copy only dependency manifests first — Docker cache invalidation is per-layer.
# If only app code changes, this layer is served from cache.
COPY pyproject.toml uv.lock ./

# Install uv for fast dependency resolution, then export a flat requirements file
RUN pip install --no-cache-dir uv && \
    uv export --frozen --no-dev --output-file /build/requirements.txt

# ── Stage 2: builder ──────────────────────────────────────────────────────────
FROM python:3.13-slim AS builder

WORKDIR /app

COPY --from=deps /build/requirements.txt .

# Install into a prefix directory so we can copy just the site-packages
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# ── Stage 3: production runtime ───────────────────────────────────────────────
FROM python:3.13-slim AS runtime

# Create a non-root user — never run application code as root in production
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser

WORKDIR /app

# Copy installed packages from builder
COPY --from=builder /install /usr/local

# Copy application source
COPY --chown=appuser:appgroup app/ ./app/

USER appuser

# Metadata
ARG GIT_SHA=unknown
LABEL org.opencontainers.image.revision="${GIT_SHA}" \
      org.opencontainers.image.source="github.com/your-org/aesf-py-middleware"

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", \
     "--workers", "2", "--no-access-log"]
```

### Image Size Comparison

| Stage exported | Approximate size |
|---------------|-----------------|
| Builder (with pip cache) | ~1.1 GB |
| Runtime (production) | ~180 MB |
| Runtime + debug tools | ~260 MB |

Use `docker build --target deps .` during CI to run linting/tests in the deps stage, then build to `runtime` for the pushed image.

**Common mistake:** Copying the entire source tree in stage 1 then re-copying in stage 3. This creates two copies of source in the layer graph and, more critically, invalidates the dependency layer cache on every source change. Always copy dependency manifests first, install, then copy source code as the final step.

---

## 3. Kubernetes Control Plane Architecture

Understanding how the control plane works makes cluster behavior predictable rather than mysterious.

```
┌─────────────────────────────────────────────┐
│               Control Plane (GKE managed)    │
│                                             │
│  ┌──────────┐   ┌──────────────────────┐   │
│  │  etcd    │◄──│   kube-apiserver      │   │
│  │ (source  │   │  (auth, admission,    │   │
│  │ of truth)│   │   validation, REST)   │   │
│  └──────────┘   └──────────┬───────────┘   │
│                             │               │
│  ┌──────────────┐   ┌───────▼───────────┐  │
│  │  controller- │   │  kube-scheduler   │  │
│  │  manager     │   │  (assigns pods    │  │
│  │  (reconcile) │   │   to nodes)       │  │
│  └──────────────┘   └───────────────────┘  │
└─────────────────────────────────────────────┘
         │ watch / list                 │ watch
         ▼                             ▼
┌─────────────────────────────────────────────┐
│                Worker Nodes                  │
│                                             │
│  ┌──────────┐  ┌───────────┐  ┌─────────┐ │
│  │ kubelet  │  │ kube-proxy│  │container│ │
│  │(pod life)│  │(iptables/ │  │ runtime │ │
│  │          │  │ ipvs)     │  │(containd│ │
│  └──────────┘  └───────────┘  └─────────┘ │
└─────────────────────────────────────────────┘
```

### Key Components

**etcd** is a distributed key-value store that holds all cluster state. Every Kubernetes object — Pod, Deployment, ConfigMap — is stored as a protobuf-serialized entry in etcd. The API server is the only component that writes to etcd directly; all other components communicate through the API server.

**kube-apiserver** is the single entry point for all control plane operations. It validates and mutates resources (via admission webhooks), enforces RBAC, and serves the watch API that other components use to react to state changes.

**controller-manager** runs a set of controllers — Deployment controller, ReplicaSet controller, StatefulSet controller, etc. — each running a reconcile loop: observe actual state, compare to desired state, issue API calls to close the gap.

**kube-scheduler** watches for Pods with no `nodeName` set and assigns them to nodes based on resource requests, taints/tolerations, affinity rules, and custom scoring.

**kubelet** runs on every node. It watches the API server for Pods scheduled to its node, pulls images, starts containers via the container runtime interface (CRI), and reports Pod status back.

**Common mistake:** Assuming that deleting a Pod terminates the workload. The Deployment controller's reconcile loop immediately creates a replacement. To stop a workload you must scale the Deployment to zero or delete the Deployment itself.

---

## 4. Workload Controllers: Deployments, StatefulSets, DaemonSets

### Deployment

A Deployment manages a ReplicaSet, which manages Pods. The Deployment controller handles rolling updates by creating a new ReplicaSet and scaling it up while scaling the old one down.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aesf-middleware
  namespace: aesf-production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: aesf-middleware
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Allow 1 extra pod during rollout
      maxUnavailable: 0  # Never go below desired replica count (zero-downtime)
  template:
    metadata:
      labels:
        app: aesf-middleware
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: middleware
          image: gcr.io/your-project/aesf-middleware:{{ .Values.image.tag }}
          ports:
            - containerPort: 8000
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 15
            periodSeconds: 20
```

`maxUnavailable: 0` with `maxSurge: 1` is the zero-downtime pattern for AESF middleware — critical because Epic callback webhooks arrive continuously and a brief gap in the receiver causes missed sync events that require manual reconciliation.

### StatefulSet

StatefulSets are for workloads with stable identity requirements: ordered pod names (`pod-0`, `pod-1`), stable network identity via a headless service, and persistent volume claim templates that survive pod rescheduling.

In the AESF ecosystem, PostgreSQL (if self-hosted) or a Redis queue would use a StatefulSet. The key difference from a Deployment: pods are created and deleted in order, and `kubectl rollout restart` on a StatefulSet terminates `pod-N` and waits for it to become Ready before proceeding to `pod-(N-1)`.

### DaemonSet

A DaemonSet ensures exactly one pod runs on every node (or on nodes matching a selector). Common uses: log shippers (Fluentd), node-level metrics exporters (Datadog agent), and CNI plugins. In GKE the Datadog DaemonSet that monitors AESF node-level metrics is managed this way.

**Common mistake:** Using a Deployment with `replicas: N` where N equals the node count when a DaemonSet is the correct primitive. If a new node is added, the Deployment does not automatically place a pod there; the DaemonSet does.

---

## 5. Services, Ingress, and NetworkPolicy

### Services

A Service provides a stable virtual IP (ClusterIP) and DNS name for a set of Pods selected by label. kube-proxy maintains iptables/ipvs rules that load-balance traffic to matching pod IPs.

| Service Type | Use Case |
|-------------|----------|
| `ClusterIP` | Internal cluster communication (default) |
| `NodePort` | Expose on each node's IP at a static port |
| `LoadBalancer` | Provision a GCP L4 load balancer (GKE) |
| `ExternalName` | DNS alias to an external hostname |
| `Headless` (`clusterIP: None`) | StatefulSet stable DNS, direct pod addressing |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: aesf-middleware
  namespace: aesf-production
spec:
  selector:
    app: aesf-middleware
  ports:
    - name: http
      port: 80
      targetPort: 8000
  type: ClusterIP
```

### Ingress

An Ingress resource configures an L7 load balancer (in GKE, the GCP HTTP(S) Load Balancer via the GKE Ingress controller or nginx-ingress). It routes based on hostname and path.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: aesf-middleware-ingress
  namespace: aesf-production
  annotations:
    kubernetes.io/ingress.class: "gce"
    kubernetes.io/ingress.allow-http: "false"
spec:
  tls:
    - secretName: aesf-tls-cert
  rules:
    - host: api.aesf.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: aesf-middleware
                port:
                  number: 80
```

### NetworkPolicy

NetworkPolicy is a namespace-scoped firewall implemented by the CNI plugin (Calico, Cilium). Without any NetworkPolicy, all pods can reach all other pods across namespaces — a significant blast radius for a compromised container.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: middleware-ingress-policy
  namespace: aesf-production
spec:
  podSelector:
    matchLabels:
      app: aesf-middleware
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - port: 8000
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: aesf-postgres
      ports:
        - port: 5432
    - to: []          # Allow DNS
      ports:
        - port: 53
          protocol: UDP
```

**Common mistake:** Applying a NetworkPolicy with only an `ingress` section and assuming egress is implicitly allowed. Once a NetworkPolicy selects a pod, *all unmatched traffic in the specified policyTypes is denied*. Missing a `policyTypes: [Ingress]`-only declaration means egress is unaffected; but adding `policyTypes: [Ingress, Egress]` without explicit egress rules silently blocks database connections.

---

## 6. Horizontal Pod Autoscaler (HPA)

HPA scales a Deployment (or StatefulSet) based on observed metrics versus a target. The most common metric is CPU utilization, but AESF's sync worker is a better candidate for a custom metric like queue depth.

### HPA with CPU Metric

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: aesf-sync-worker-hpa
  namespace: aesf-production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: aesf-sync-worker
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
```

### HPA with Custom Metric (Queue Depth)

For the AESF ETL sync worker, scaling on CPU is a lagging indicator — the queue fills up before CPU climbs. A better approach is to expose a `/metrics` endpoint from the worker with a Prometheus gauge for pending job count, and configure the HPA to target that metric via the Prometheus Adapter.

```yaml
  metrics:
    - type: External
      external:
        metric:
          name: aesf_sync_queue_depth
        target:
          type: AverageValue
          averageValue: "50"   # Scale when avg queue depth per pod exceeds 50
```

The HPA reconcile loop runs every 15 seconds by default. It fetches current metric values from the metrics API, computes `desiredReplicas = ceil(currentReplicas * currentMetricValue / targetMetricValue)`, and updates the Deployment's `spec.replicas`.

**Common mistake:** Setting `minReplicas: 1` for a stateless HTTP service in production. A single pod means any rollout, node drain, or OOM event causes a brief outage. For AESF middleware, `minReplicas: 2` across two availability zones is the correct baseline.

---

## 7. Helm Chart Authoring

Helm packages Kubernetes manifests as charts with Go-template-based parameterization. A chart has this structure:

```
charts/module-middleware/
├── Chart.yaml
├── values.yaml
├── values.dev.yaml
├── values.staging.yaml
├── values.production.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── hpa.yaml
    ├── ingress.yaml
    ├── configmap.yaml
    ├── serviceaccount.yaml
    └── _helpers.tpl
```

### Chart.yaml

```yaml
apiVersion: v2
name: module-middleware
description: AESF Python FastAPI middleware
type: application
version: 0.5.0          # Chart version — bump on chart changes
appVersion: "2.14.0"    # Application version — informational
dependencies:
  - name: common
    version: "~1.0"
    repository: "oci://registry.yourdomain.com/helm-charts"
```

### values.yaml (defaults)

```yaml
replicaCount: 2

image:
  repository: gcr.io/your-project/aesf-middleware
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8000

ingress:
  enabled: false
  host: ""
  tlsSecret: ""

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 60

env:
  LOG_LEVEL: "info"
  EPIC_SERVER: ""
  DATABASE_URL: ""

vaultSecrets:
  enabled: true
  role: "aesf-middleware"
```

### values.staging.yaml (overrides)

```yaml
replicaCount: 2

image:
  tag: "2.14.0-rc1"

ingress:
  enabled: true
  host: api-staging.aesf.yourdomain.com
  tlsSecret: aesf-staging-tls

resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1Gi

autoscaling:
  enabled: true
  maxReplicas: 8

env:
  LOG_LEVEL: "debug"
  EPIC_SERVER: "de21web"
```

### _helpers.tpl

```
{{- define "module-middleware.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "module-middleware.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```

### templates/deployment.yaml (excerpt)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "module-middleware.fullname" . }}
  labels:
    {{ include "module-middleware.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ .Chart.Name }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    spec:
      containers:
        - name: middleware
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          env:
            {{- range $key, $val := .Values.env }}
            - name: {{ $key }}
              value: {{ $val | quote }}
            {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

### Deploying with Environment Overrides

```bash
# Deploy to staging
helm upgrade --install aesf-middleware ./charts/module-middleware \
  -f charts/module-middleware/values.yaml \
  -f charts/module-middleware/values.staging.yaml \
  --namespace aesf-staging \
  --set image.tag=$(git rev-parse --short HEAD)

# Dry-run to preview rendered manifests
helm template aesf-middleware ./charts/module-middleware \
  -f charts/module-middleware/values.staging.yaml | kubectl diff -f -
```

**Common mistake:** Hardcoding secret values in `values.*.yaml` files committed to git. Secrets must come from Vault (via the Vault Agent Injector or Secrets Store CSI Driver), Kubernetes Secrets created out-of-band, or environment-specific `--set` flags passed by the CI pipeline at deploy time. The `vaultSecrets.enabled` pattern in the values above is the correct hook for the Vault injector annotation.

---

## 8. Zero-Downtime Deployments and Pod Disruption Budgets

A zero-downtime deployment in Kubernetes requires coordination between the rollout strategy, readiness probes, and termination grace period.

### Lifecycle of a Safe Rollout

1. New ReplicaSet is created; new pods start.
2. Kubelet runs the container's `readinessProbe`. Pod remains `NotReady` — kube-proxy does not route traffic to it.
3. Once the readiness probe passes, the pod enters `Ready` state and kube-proxy adds it to the Service endpoints.
4. The old ReplicaSet is scaled down by one pod.
5. The old pod receives `SIGTERM`. The application should stop accepting new connections and finish in-flight requests within `terminationGracePeriodSeconds`.
6. After the grace period, `SIGKILL` is sent.

### PodDisruptionBudget

A PDB limits voluntary disruptions (node drains, cluster upgrades) to ensure a minimum number of pods remain available:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: aesf-middleware-pdb
  namespace: aesf-production
spec:
  minAvailable: 2   # At least 2 pods must be available during disruptions
  selector:
    matchLabels:
      app: aesf-middleware
```

In GKE, cluster auto-upgrade drains nodes. Without a PDB, all middleware pods can be evicted simultaneously if they happen to land on the same node being drained. With `minAvailable: 2`, the drain operation blocks until it can evict a pod without violating the budget.

**Common mistake:** Setting `minAvailable` equal to `replicaCount`. This makes the PDB impossible to satisfy during a drain — the node drain will hang indefinitely. A common safe value is `replicaCount - 1` or a percentage like `minAvailable: "66%"`.

---

## 9. Observability: Logs, Metrics, and Health Probes

### Health Probe Types

| Probe | Failure Action | Use For |
|-------|---------------|---------|
| `livenessProbe` | Restart the container | Deadlock detection |
| `readinessProbe` | Remove from Service endpoints | Traffic gating |
| `startupProbe` | Delay liveness checks | Slow-starting apps |

For the AESF middleware (FastAPI), the `/health` endpoint should return fast — it should not query the database. Use a separate `/health/ready` endpoint that checks the database connection for the readiness probe, and `/health/live` (a simple 200) for the liveness probe. This prevents a slow database query from triggering a liveness restart.

### Structured Logging for GKE

GKE's Cloud Logging agent parses JSON logs automatically. Configure uvicorn/FastAPI to emit JSON:

```python
# app/core/logging.py
import logging, json, sys

class JsonFormatter(logging.Formatter):
    def format(self, record):
        return json.dumps({
            "severity": record.levelname,
            "message": record.getMessage(),
            "logger": record.name,
            "timestamp": self.formatTime(record),
        })

handler = logging.StreamHandler(sys.stdout)
handler.setFormatter(JsonFormatter())
logging.root.addHandler(handler)
```

**Common mistake:** Using `livenessProbe` to check external dependencies (database, Epic API). If Epic's API is down, every pod restarts in a crash loop — turning an external dependency outage into a full service outage. Liveness probes should only detect internal deadlocks; readiness probes gate traffic based on dependency health.

---

## 10. Key Concepts Summary

```
Docker & Kubernetes Stack
│
├── Container Primitives
│   ├── Linux Namespaces (pid, net, mnt, uts, ipc, user, cgroup)
│   ├── cgroups v2 (cpu, memory, io, pids)
│   └── OCI Image: layers + config + manifest
│
├── Docker Build
│   ├── Multi-stage Dockerfile
│   │   ├── Stage: deps   → resolves dependencies
│   │   ├── Stage: builder → installs packages
│   │   └── Stage: runtime → minimal production image
│   └── Layer caching: copy manifests before source
│
├── Kubernetes Control Plane
│   ├── etcd           → source of truth (all objects)
│   ├── kube-apiserver → single entry point, RBAC, admission
│   ├── controller-manager → reconcile loops (Deployment, RS, etc.)
│   └── kube-scheduler → assigns pods to nodes
│
├── Workload Controllers
│   ├── Deployment     → stateless, rolling updates
│   ├── StatefulSet    → stable identity, ordered ops
│   └── DaemonSet      → one pod per node
│
├── Networking
│   ├── Service (ClusterIP / LB / NodePort / Headless)
│   ├── Ingress → L7 routing, TLS termination
│   └── NetworkPolicy → CNI-enforced firewall
│
├── Autoscaling
│   └── HPA → CPU / custom metrics → scales Deployment replicas
│
├── Helm
│   ├── Chart: Chart.yaml + templates/ + values.yaml
│   ├── values.*.yaml → per-environment overrides
│   └── _helpers.tpl  → reusable template functions
│
└── Reliability
    ├── PodDisruptionBudget → drain safety
    ├── RollingUpdate strategy → zero-downtime
    ├── readinessProbe → traffic gating
    └── livenessProbe  → deadlock recovery
```

---

## Quiz — 20 Questions

### Questions

**1.** What is the Linux kernel mechanism that prevents a container process from seeing processes in other containers or on the host?

**2.** A Kubernetes pod is OOMKilled repeatedly despite the application reporting low memory usage. What is the most likely cause and where would you verify it?

**3.** Explain what `maxUnavailable: 0` and `maxSurge: 1` mean in a Deployment rolling update strategy.

**4.** In a multi-stage Dockerfile, why should you copy `pyproject.toml` and `uv.lock` *before* copying the full application source?

**5.** What is the role of `etcd` in the Kubernetes control plane, and which component is the only one that writes to it directly?

**6.** What is the difference between a `StatefulSet` and a `Deployment`? Give one example use case for each.

**7.** A `NetworkPolicy` is applied to the `aesf-middleware` pods with `policyTypes: [Ingress, Egress]` but the egress rules only allow port 5432 to the database. What happens to outbound DNS queries from those pods?

**8.** What does the HPA stabilization window (`scaleDown.stabilizationWindowSeconds`) control, and why is a higher value typically used for scale-down than scale-up?

**9.** You add a new node to a GKE cluster. Which controller type ensures the Datadog monitoring agent is automatically scheduled onto the new node?

**10.** What is a `PodDisruptionBudget` and when would it cause a `kubectl drain` to block indefinitely?

**11.** In Helm, what is the difference between `Chart.version` and `Chart.appVersion`?

**12.** Describe the sequence of events from when a Pod receives `SIGTERM` to when the process is forcibly killed.

**13.** Why is it dangerous to set a `livenessProbe` that checks an external dependency like a database or an upstream API?

**14.** You run `kubectl delete pod aesf-middleware-abc123` in the production namespace. A new pod appears 5 seconds later. Why?

**15.** What Kubernetes QoS class is assigned to a pod where `requests.memory == limits.memory` and `requests.cpu == limits.cpu`? Why does this matter for eviction?

**16.** In the AESF Helm chart, `values.staging.yaml` sets `autoscaling.enabled: true` but `values.yaml` sets it to `false`. The deployment template has `{{- if not .Values.autoscaling.enabled }} replicas: {{ .Values.replicaCount }} {{- end }}`. What happens to `spec.replicas` in the staging deployment?

**17.** What does a headless Service (`clusterIP: None`) provide that a regular ClusterIP Service does not?

**18.** Explain the difference between `readinessProbe` and `startupProbe`. In what scenario do you need both?

**19.** Why should secret values never appear in `values.*.yaml` files committed to a git repository, and what is the correct alternative for AESF?

**20.** A Helm deployment to staging succeeds but produces different behavior than production despite ostensibly the same chart. What are two likely causes related to Helm values, and how would you diagnose them?

---

### Answers

??? note "Reveal Answers"

    **1.** The `pid` namespace is the mechanism that isolates process visibility. Each container gets its own PID namespace so it sees only its own processes, with its init process appearing as PID 1. Processes in other containers or on the host are invisible within the namespace. You can verify this by running `ps aux` inside a container and observing only its own process tree, then checking `/proc/<host-pid>/ns/pid` on the host to confirm a different inode than the host's PID namespace.

    **2.** The most likely cause is that the container's cgroup `memory.max` limit is lower than the application believes it has available. The application may be reading `/proc/meminfo` (which shows host memory) rather than its cgroup limit. Verify by inspecting `/sys/fs/cgroup/memory/kubepods/.../memory.max` on the node, or by checking `kubectl describe pod <name>` for the OOMKilled exit reason and comparing the `resources.limits.memory` value against the actual working set observed before the kill.

    **3.** `maxUnavailable: 0` means zero pods from the current desired count may be in an unavailable state during the rollout — no existing pod is removed until its replacement is healthy. `maxSurge: 1` means one extra pod above the desired count may be created. Together they ensure at least `replicaCount` pods are always serving traffic; the rollout temporarily runs `replicaCount + 1` pods. This is the zero-downtime pattern used for AESF middleware where missing a single Epic webhook callback requires manual reconciliation.

    **4.** Docker builds each `COPY` instruction as a separate layer. If application source files change but `pyproject.toml` and `uv.lock` do not, Docker serves the dependency installation layer from cache, making builds significantly faster. Copying the full source tree first would invalidate the cache on every code change, forcing a full `pip install` on every build. This separation is especially valuable in CI pipelines where dependency installation can take 2-5 minutes.

    **5.** etcd is a distributed, strongly consistent key-value store that holds the entire cluster state — every Kubernetes object serialized as protobuf. It is the single source of truth; if etcd is lost and has no backup, the cluster state is gone. The `kube-apiserver` is the only component that reads from and writes to etcd directly. All other components (scheduler, controller-manager, kubelet) communicate with the cluster state exclusively through the API server's REST/watch interface.

    **6.** A `Deployment` manages stateless pods with interchangeable identity — pods can be replaced in any order and with any name suffix. A `StatefulSet` provides stable, ordered pod identity: predictable names (`pod-0`, `pod-1`), stable DNS entries via a headless service, and per-pod PersistentVolumeClaims that survive rescheduling. Use a Deployment for the AESF middleware API servers; use a StatefulSet for PostgreSQL or Redis where data locality and stable network identity are required.

    **7.** The DNS queries will be silently dropped, causing all outbound connections from those pods to fail with DNS resolution errors. `policyTypes: [Ingress, Egress]` with no matching egress rule for UDP port 53 blocks DNS. The fix is to add an explicit egress rule allowing UDP and TCP port 53 to the cluster DNS service (`kube-dns` in the `kube-system` namespace), or using an open `- {}` egress rule to allow all outbound traffic while only restricting ingress.

    **8.** The stabilization window prevents flapping by requiring the desired replica count to be stable for the specified number of seconds before acting. A higher scale-down window (e.g., 300 seconds) prevents the HPA from prematurely removing pods during a brief lull in traffic, which would then require immediately scaling back up. Scale-up uses a shorter window because the cost of being under-provisioned (dropped requests, latency spikes) is higher than briefly over-provisioning. For AESF's sync worker, a 5-minute scale-down window prevents repeatedly cycling pods during bursty Epic sync events.

    **9.** A `DaemonSet` ensures exactly one pod per node automatically. When a new node joins the cluster, the DaemonSet controller's reconcile loop detects a node without the matching daemon pod and schedules one. A Deployment with replicas equal to node count would not react to node additions; you would have to manually update the replica count. This is why all cluster-level monitoring agents, log shippers, and CNI components are deployed as DaemonSets.

    **10.** A PodDisruptionBudget defines the minimum number (or percentage) of pods that must remain available during voluntary disruptions like node drains. `kubectl drain` evicts pods one at a time, checking the PDB before each eviction. If evicting a pod would violate `minAvailable`, the drain blocks. It blocks indefinitely if `minAvailable` equals `replicaCount` — satisfying the budget would require keeping all pods running, which is impossible when trying to drain the node they are on. The correct value is `replicaCount - 1` or a percentage below 100%.

    **11.** `Chart.version` is the semantic version of the Helm chart itself — it changes whenever the chart's templates, values schema, or metadata change. `Chart.appVersion` is informational and typically tracks the version of the application the chart deploys. Bumping `appVersion` from `2.14.0` to `2.15.0` does not require bumping `Chart.version` unless the chart templates also changed. In practice, CI pipelines should bump both together to maintain a clear audit trail of which chart version deployed which application version.

    **12.** When a pod is terminated, Kubernetes simultaneously removes the pod from all Service endpoints (stopping new traffic) and sends `SIGTERM` to PID 1 of each container. The application should catch `SIGTERM` and enter a graceful shutdown: stop accepting connections, drain in-flight requests, and close database connections. After `terminationGracePeriodSeconds` (default 30, set to 60 for AESF middleware to allow long Epic API calls to complete), Kubernetes sends `SIGKILL`, which immediately terminates the process. If the application ignores `SIGTERM`, it is killed after the grace period regardless.

    **13.** If a liveness probe checks an external dependency and that dependency becomes unavailable, every pod will fail its liveness check and be restarted. Kubernetes will restart them, they will fail again, enter `CrashLoopBackOff`, and the service becomes completely unavailable — even though the application itself is healthy and could serve requests that do not require that dependency. A liveness probe should only detect that the application process itself is deadlocked or stuck. External dependency health belongs in readiness probes, which remove the pod from load balancing without restarting it.

    **14.** The Deployment controller's reconcile loop runs continuously and observes that the actual replica count (2) is below the desired count (3). It immediately creates a replacement pod via the ReplicaSet to restore the desired state. Deleting a pod managed by a Deployment never reduces the running count permanently — to do that you must edit `spec.replicas` on the Deployment or delete the Deployment itself. This is fundamental to the controller model: desired state always wins over observed state.

    **15.** A pod where requests equal limits for all resources is assigned the `Guaranteed` QoS class. This is the highest priority class; the kubelet will only evict Guaranteed pods as a last resort, after all `BestEffort` and `Burstable` pods have been evicted. For AESF sync workers that hold in-memory state about Epic webhook callbacks being processed, unexpected eviction could cause duplicate processing or data loss. Setting requests equal to limits on those pods prevents eviction under node memory pressure.

    **16.** The `spec.replicas` field is omitted from the staging deployment manifest entirely. The `{{- if not .Values.autoscaling.enabled }}` condition evaluates to false (since `autoscaling.enabled` is `true` in staging), so the `replicas:` line is not rendered. When HPA is enabled, this is correct behavior — the HPA takes ownership of `spec.replicas` and having a static value in the manifest would conflict with HPA-controlled scaling. If you set `replicas` in the manifest alongside a functioning HPA, Helm upgrades will reset the replica count on every deploy.

    **17.** A headless Service (clusterIP: None) does not allocate a virtual IP. Instead, DNS queries for the service name return A records for each individual pod IP rather than a single stable VIP. This allows clients to enumerate and connect directly to individual pods — essential for StatefulSets where `pod-0.service.namespace.svc.cluster.local` must resolve to a specific, stable pod. A regular ClusterIP Service is appropriate for stateless load balancing; a headless Service is required when the client needs to know or control which specific pod instance it connects to.

    **18.** A `startupProbe` gates liveness and readiness probes — while the startup probe has not succeeded, neither liveness nor readiness probes are checked. This prevents the liveness probe from killing a slow-starting container before it has had time to initialize. A `readinessProbe` runs throughout the pod's lifetime and gates traffic routing. You need both when the application has a long initialization phase (database migrations, cache warming) AND may deadlock later during normal operation. For AESF middleware with Alembic migrations that can take 30-60 seconds, a startupProbe with `failureThreshold: 12, periodSeconds: 10` gives 2 minutes for startup before liveness probes begin.

    **19.** Git history is permanent and widely accessible — a secret committed to a repository may be leaked even after deletion via `git filter-branch` if the history has been cloned. For AESF, the correct alternative is the Vault Agent Injector: pods annotated with `vault.hashicorp.com/agent-inject-secret-*` have secrets written to the pod filesystem by a sidecar container, retrieved from Vault using the pod's Kubernetes service account JWT for authentication. Secrets never appear in Helm values, Kubernetes Secrets manifests, or CI logs. The `vaultSecrets.enabled` flag in the Helm chart controls whether the Vault injector annotations are added.

    **20.** Two likely causes: (1) **Values file precedence** — if the staging deploy passes `-f values.yaml -f values.staging.yaml`, overrides are applied correctly, but if the order is reversed or a values file is missing, the wrong defaults apply. Diagnose with `helm get values <release> -n aesf-staging` to see the effective computed values. (2) **Default values in templates** — a template may use `{{ .Values.someKey | default "prod-default" }}` where the default was written assuming production config. In staging, if the key is absent from `values.staging.yaml`, it falls through to the template default rather than `values.yaml`. Diagnose with `helm template <release> ./chart -f values.staging.yaml` and diff the rendered output against the production render.
