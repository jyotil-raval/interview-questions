<!-- Generated: 2026-03-12 | Docker 27.x | Kubernetes 1.33 -->

# Container Infrastructure Interview Preparation — Tech Lead / Architect Level

> Docker 27.x · Kubernetes 1.33 | Generated: 2026-03-12 | Career OS

---

# SECTION 1 — Docker 27.x

---

**L1–L5 Topics identified:**

- What is Docker / Containers vs VMs
- Images — Dockerfile, layers, build cache, multi-stage
- Container Runtime — namespaces, cgroups, OCI
- Networking — bridge / host / overlay
- Volumes & Storage
- Security — rootless, capabilities, read-only filesystem
- Docker Compose & production patterns

---

### Core Concepts

#### Containers vs VMs / What Docker Provides

- **Question:** What is a container, how does it differ from a virtual machine, and what kernel primitives underpin Docker?
- **Answer:**
  - A **VM** virtualize hardware — each VM runs a full OS kernel on top of a hypervisor. VMs are isolated at the hardware level. Boot time: seconds to minutes. Memory overhead: full OS per VM (hundreds of MB to GB).
  - A **container** virtualize at the OS level — shares the host kernel, isolates at the process level. No guest OS. Boot time: milliseconds. Memory overhead: only the process and its dependencies.
  - **Docker's kernel primitives:**
    (1) **Namespaces** — provide isolation: `pid` (process IDs), `net` (network interfaces), `mnt` (mount points), `uts` (hostname), `ipc` (inter-process communication), `user` (user IDs). Each container sees only its own namespace.
    (2) **cgroups (control groups)** — resource limits: CPU, memory, I/O, network. Prevent one container from consuming all host resources.
    (3) **Union filesystem (OverlayFS)** — layers container images on top of each other; only the diff layer is written per container.
  - Docker itself is not the only container runtime — it uses **containerd** (OCI-compliant runtime) under the hood. Kubernetes does not use Docker at all (since K8s 1.24) — it uses containerd or CRI-O directly.
- **Example:**

```bash
# Namespace isolation — container sees its own PID namespace
docker run --rm alpine ps aux
# PID 1 is the container's process — not the host's

# cgroup resource limits
docker run --memory="512m" --cpus="1.5" nginx
# Container cannot exceed 512MB RAM or 1.5 CPU cores

# What Docker adds over raw namespaces/cgroups:
# - Image management (pull, push, layer caching)
# - Dockerfile — reproducible image builds
# - Container lifecycle management (start, stop, restart)
# - Networking (bridge networks, DNS between containers)
# - Volume management
```

- **Technical Terms to Include:** namespace, cgroup, OverlayFS, containerd, OCI (Open Container Initiative), CRI (Container Runtime Interface), image layer, Union filesystem, PID 1, hypervisor vs kernel, rootless containers
- **Gotcha:** PID 1 in a container is your application process — not an init system. PID 1 is responsible for handling signals (SIGTERM for graceful shutdown) and reaping zombie processes. If your application doesn't handle SIGTERM, `docker stop` will wait `--stop-timeout` seconds then SIGKILL — causing ungraceful shutdown. Use `tini` as an init process in containers, or handle SIGTERM explicitly in your application.
- **Follow-Up:** "What is the difference between Docker and containerd?" → Docker is a developer tool — CLI, Dockerfile builder, image registry integration, and it uses containerd as its runtime. Containerd is the low-level container runtime (OCI-compliant) that actually creates and manages containers via runc. Kubernetes talks directly to containerd (or CRI-O) via the CRI interface — Docker is not in the path. Docker removed its own runtime (dockershim) from Kubernetes in 1.24. → They're testing runtime architecture depth.
- **Conclusion:** Docker's value is the developer experience layer (Dockerfile, image registry, CLI) built on top of kernel primitives (namespaces, cgroups) that have existed in Linux since 2007 — understanding both layers is what separates architects who design containers correctly from those who treat them as VMs.

---

#### Dockerfile — Layers, Build Cache, Multi-Stage

- **Question:** How do Dockerfile layers and build cache work, and what is the multi-stage build pattern and why is it mandatory for production images?
- **Answer:**
  - Every `RUN`, `COPY`, `ADD` instruction in a Dockerfile creates a new image layer. Layers are immutable and cached. If a layer and all preceding layers are unchanged, Docker uses the cached layer — dramatically accelerating rebuilds.
  - **Cache invalidation:** If any layer changes, all subsequent layers are invalidated. `COPY . .` (copying source code) invalidates everything after it. Dependency installation before code copy is the canonical pattern — dependencies change less often than code.
  - **Multi-stage builds:** Use multiple `FROM` statements in one Dockerfile. Each stage is independent. Final stage copies only what it needs from earlier stages — discards build tools, compilers, test dependencies. Results in a minimal production image.
  - **Production image principles:** (1) Use a minimal base (`alpine`, `distroless`). (2) Don't run as root — `USER nonroot`. (3) Multi-stage to exclude build tools. (4) `.dockerignore` to exclude `node_modules`, `.git`, test files from build context. (5) Pin base image versions — `node:20.10-alpine` not `node:latest`.
- **Example:**

```dockerfile
# WRONG — cache invalidated by any code change, reinstalls deps every time
FROM node:20-alpine
WORKDIR /app
COPY . .                     # copies everything including code — cache bust
RUN npm ci                   # reinstalled on every code change

# CORRECT — dependency layer cached separately from code
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./        # only package files — rarely changes
RUN npm ci --only=production # cached until package.json changes

FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build            # compile TypeScript, bundle, etc.

# Production stage — minimal image, no build tools
FROM node:20-alpine AS runner
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser                 # never run as root
COPY --from=deps    /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

- **Technical Terms to Include:** Dockerfile layer, build cache, cache invalidation, multi-stage build, `.dockerignore`, distroless, non-root user, `COPY --from`, build context, `ARG` vs `ENV`, layer squashing, `--no-cache`
- **Gotcha:** Secrets in `RUN` instructions are stored in the image layer — even if a subsequent `RUN rm` deletes the file, the secret is still visible in the layer history (`docker history`). Use Docker BuildKit's `--secret` mount for secrets: `RUN --mount=type=secret,id=api_key cat /run/secrets/api_key` — the secret is mounted in-memory only for that instruction and never written to a layer.
- **Follow-Up:** "What is a distroless image and when do you use it?" → Google's distroless images contain only the runtime (e.g., the JVM or Node.js binary) and no OS utilities (no shell, no package manager, no `ls`, no `curl`). The result: smaller image, dramatically reduced attack surface (no shell means no shell injection). Trade-off: no shell means debugging requires a separate debug image or ephemeral debug container (`kubectl debug`). Use distroless in production; use a full base in dev. → They're testing container security posture depth.
- **Conclusion:** Multi-stage Dockerfile design — separating dependency installation, build, and runtime stages — is the non-negotiable baseline for production container images: it minimises image size, attack surface, and rebuild time simultaneously.

---

#### Docker Networking

- **Question:** How does Docker networking work — bridge, host, and overlay modes — and what are the DNS and port-mapping mechanics?
- **Answer:**
  - **Bridge network (default):** Each container gets a virtual ethernet interface on a private bridge network (`172.17.0.0/16` by default). Containers on the same bridge can communicate by IP. Containers on user-defined bridge networks can also communicate by container name (DNS resolution). Traffic exits via NAT through the host's IP.
  - **Host network:** Container shares the host's network namespace — no network isolation. Container ports are directly the host ports. Eliminates NAT overhead — useful for high-throughput applications. No port mapping needed; no port isolation either.
  - **Overlay network (Swarm / multi-host):** Extends a bridge network across multiple Docker hosts using VXLAN. Containers on different hosts can communicate as if on the same network. Required for Docker Swarm; Kubernetes uses its own overlay (CNI plugin — Flannel, Calico, Cilium).
  - **DNS:** User-defined bridge networks get built-in DNS. Containers resolve each other by service name — `redis` resolves to the `redis` container's IP. Default bridge network has no built-in DNS — containers must use IP addresses or `--link` (legacy, deprecated).
  - **Port mapping (`-p`):** `docker run -p 8080:3000` maps host port 8080 to container port 3000. Docker adds an iptables rule to forward traffic. `EXPOSE` in Dockerfile is documentation — it doesn't publish the port.
- **Example:**

```bash
# User-defined bridge — DNS by container name
docker network create app-network
docker run -d --name redis --network app-network redis:7
docker run -d --name api --network app-network -e REDIS_URL=redis://redis:6379 api-image
# 'api' container resolves 'redis' by name — no hardcoded IP

# Host network — no NAT, direct host ports
docker run --network host nginx
# nginx binds directly to host port 80

# Port mapping
docker run -p 8080:80 nginx
# Host:8080 → Container:80
# EXPOSE 80 in Dockerfile is just metadata — -p does the actual binding

# Inspect container network
docker inspect <container> | jq '.[0].NetworkSettings.Networks'
```

- **Technical Terms to Include:** bridge network, host network, overlay network, VXLAN, CNI, iptables, NAT, port mapping, `EXPOSE`, DNS resolution, user-defined bridge, `docker network create`, container IP
- **Gotcha:** `EXPOSE` in a Dockerfile does **not** publish a port to the host. It is documentation for the image consumer and enables automatic port mapping in Docker Compose (with `--expose-all`), but `docker run` without `-p` means the port is only accessible from within the same Docker network. Many engineers assume `EXPOSE` means the port is accessible from the host.
- **Follow-Up:** "How does Docker Compose handle service networking?" → Docker Compose creates a default user-defined bridge network for each Compose project. All services in the Compose file are automatically attached to this network and can resolve each other by service name. No manual network creation needed. `ports:` publishes to the host; `expose:` only makes it accessible to other services (not the host). → They're testing Compose networking precision.
- **Conclusion:** Docker networking's user-defined bridge DNS resolution is the mechanism that enables service discovery in Docker Compose and single-host microservice deployments — understanding that DNS is name-based on user networks (not default bridge) and that `EXPOSE` is documentation-only prevents the most common networking misconfigurations.

---

# SECTION 2 — Kubernetes 1.33

---

**L1–L5 Topics identified:**

- What is Kubernetes / Control Plane vs Data Plane
- Core Objects — Pod / Deployment / Service / ConfigMap / Secret
- Scheduling — node affinity / taints / tolerations / resource requests/limits
- Networking — CNI / Services (ClusterIP / NodePort / LoadBalancer) / Ingress
- Storage — PV / PVC / StorageClass
- Health checks — liveness / readiness / startup probes
- RBAC & Security — ServiceAccount / Role / ClusterRole
- Horizontal Pod Autoscaler & Vertical Pod Autoscaler
- Helm & GitOps patterns

---

### Core Architecture

#### Control Plane vs Data Plane

- **Question:** What is Kubernetes, how does its control plane work, and what is the role of each control plane component?
- **Answer:**
  - Kubernetes is a container orchestration platform — it automates deployment, scaling, self-healing, and networking of containerised workloads.
  - **Control plane (master nodes):**
    (1) **API Server (`kube-apiserver`)** — the single entry point for all cluster operations. All components communicate via the API server. Validates and persists resource objects to etcd.
    (2) **etcd** — distributed key-value store. Single source of truth for all cluster state. Loss of etcd = loss of cluster state. Must be backed up.
    (3) **Scheduler (`kube-scheduler`)** — watches for unscheduled pods and assigns them to nodes based on resource requirements, affinity rules, taints/tolerations.
    (4) **Controller Manager (`kube-controller-manager`)** — runs control loops: ReplicaSet controller (ensures desired pod count), Deployment controller, Job controller, Node controller (detects node failures).
  - **Data plane (worker nodes):**
    (1) **kubelet** — agent on each node. Watches API server for pods assigned to the node, starts/stops containers via the CRI. Reports pod status back.
    (2) **kube-proxy** — maintains iptables/IPVS rules for Service load balancing.
    (3) **Container runtime** — containerd or CRI-O — actually runs containers.
  - **Kubernetes 1.33:** Sidecar containers (KEP-753) stable — init containers with `restartPolicy: Always` that run alongside main containers for the full pod lifetime (proxies, log shippers). Improved node memory manager.
- **Example:**

```bash
# Control loop model — declarative reconciliation
kubectl apply -f deployment.yaml
# API server validates and stores in etcd
# Deployment controller sees desired replicas ≠ actual → creates ReplicaSet
# ReplicaSet controller sees desired pods ≠ actual → creates Pod objects
# Scheduler assigns Pods to nodes
# kubelet on assigned nodes starts containers

# Inspect control plane components (kubeadm cluster)
kubectl get pods -n kube-system
# kube-apiserver, etcd, kube-scheduler, kube-controller-manager, coredns

# Direct etcd access (emergency)
etcdctl get /registry/deployments/default/my-app --prefix
```

- **Technical Terms to Include:** control plane, data plane, API server, etcd, scheduler, controller manager, kubelet, kube-proxy, CRI, CNI, control loop, reconciliation, declarative model, watch mechanism
- **Gotcha:** All control plane components communicate through the API server — no direct component-to-component communication. This means the API server is the critical availability bottleneck. For HA control planes: 3 or 5 API server replicas behind a load balancer, 3 or 5 etcd members in a raft quorum. A single API server + single etcd member is not production-ready.
- **Follow-Up:** "What happens when the control plane goes down while workloads are running?" → Already-running pods continue running — kubelet operates independently of the control plane for existing pods. No new scheduling occurs, no self-healing (failed pods don't restart), no scaling. The cluster is frozen — it runs what it had but cannot change. This is why control plane HA is operational continuity, not just availability. → They're testing control plane failure mode understanding.
- **Conclusion:** Kubernetes's control plane implements a distributed reconciliation loop — each controller continuously compares desired state (etcd) to actual state (API server) and acts to converge them — making the platform self-healing by design, not by explicit failure handling.

---

#### Core Objects — Pod, Deployment, Service

- **Question:** What is a Pod, how does a Deployment manage Pods, and what does a Service do that the Pod's IP doesn't?
- **Answer:**
  - **Pod:** The smallest deployable unit in Kubernetes. A Pod contains one or more containers that share a network namespace (same IP, same localhost) and can share volumes. Pods are ephemeral — when a node fails or a pod crashes, it's gone. Do not run pods directly — use a controller.
  - **Deployment:** Manages a ReplicaSet, which manages Pods. Declares desired state (replicas, container image, resource requests). Handles rolling updates — gradually replaces old pods with new ones. Supports rollback (`kubectl rollout undo`). `strategy: RollingUpdate` with `maxSurge` and `maxUnavailable` controls update pace.
  - **Service:** A stable network identity for a set of pods. Pods get new IPs on restart — Services provide a consistent DNS name and IP (ClusterIP) that load-balances traffic to matching pods via label selector. Types: `ClusterIP` (internal only), `NodePort` (external via node IP), `LoadBalancer` (cloud LB provisioning), `ExternalName` (DNS alias).
  - **Label selector:** Services select pods by label — `app: api`. Any pod with that label receives traffic. This enables blue/green deployments (switch the selector) and canary (add a version label).
- **Example:**

```yaml
# Deployment — manages pods declaratively
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1 # at most 4 pods during update
      maxUnavailable: 0 # never reduce below 3 — zero-downtime
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: api-service:v2.1.0
          resources:
            requests: { cpu: '250m', memory: '256Mi' }
            limits: { cpu: '500m', memory: '512Mi' }
          readinessProbe:
            httpGet: { path: /health, port: 8080 }
            initialDelaySeconds: 5
            periodSeconds: 10
---
# Service — stable ClusterIP, load balances to app: api pods
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api # routes to any pod with this label
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP # internal only — use Ingress for external
```

- **Technical Terms to Include:** Pod, Deployment, ReplicaSet, Service, ClusterIP, NodePort, LoadBalancer, label selector, rolling update, `maxSurge`, `maxUnavailable`, `kubectl rollout`, readiness probe, liveness probe
- **Gotcha:** A Deployment with `maxUnavailable: 1` and `maxSurge: 0` during a rolling update will terminate a pod before starting a new one — causing momentary capacity reduction. For zero-downtime updates: `maxUnavailable: 0` and `maxSurge: 1` ensures capacity is always maintained. Combine with a proper `readinessProbe` — traffic only routes to the new pod after it's ready.
- **Follow-Up:** "What is the difference between a Deployment and a StatefulSet?" → A Deployment manages stateless pods — pods are interchangeable, any pod can handle any request. A StatefulSet manages stateful pods — each pod has a stable hostname (`pod-0`, `pod-1`), stable storage (PVC per pod), and ordered startup/shutdown (pod-0 starts before pod-1). Use StatefulSet for databases, Kafka brokers, Elasticsearch nodes — anything that requires stable identity. → They're testing workload type selection.
- **Conclusion:** Pod → ReplicaSet → Deployment is the fundamental hierarchy for stateless workloads; Services decouple consumers from the ephemeral nature of pod IPs — together they implement the basic pattern of scalable, self-healing, zero-downtime deployable services in Kubernetes.

---

#### Health Probes — Liveness, Readiness, Startup

- **Question:** What are the three Kubernetes health probes, what does each control, and what are the misconfiguration consequences?
- **Answer:**
  - **Readiness probe:** Controls whether a pod receives traffic from a Service. Failing readiness → pod removed from Service endpoints. Passing readiness → pod added back. Use for: checking database connectivity, cache warmup completion, initial data loading. A pod can be running but not ready — correctly preventing traffic before it's capable of serving.
  - **Liveness probe:** Controls whether Kubernetes restarts a pod. Failing liveness → kubelet kills and restarts the container. Use for: detecting deadlocks, infinite loops, or stuck states that the process cannot recover from on its own. Should only fail if the process is truly unrecoverable.
  - **Startup probe:** Protects slow-starting containers. While the startup probe is failing, liveness/readiness probes are disabled. Once startup probe succeeds (or fails `failureThreshold` times → pod restarts), normal probing begins. Use for: JVM warmup, database migration on startup, large cache initialization.
  - **Probe types:** `httpGet` (GET request, 200–399 is success), `tcpSocket` (port open is success), `exec` (command exit code 0 is success), `grpc` (gRPC health check protocol).
- **Example:**

```yaml
containers:
  - name: api
    image: api-service:latest
    startupProbe: # protects slow startup
      httpGet: { path: /health, port: 8080 }
      failureThreshold: 30 # 30 * 10s = 5 minutes to start
      periodSeconds: 10

    readinessProbe: # controls traffic routing
      httpGet: { path: /ready, port: 8080 }
      initialDelaySeconds: 0 # check immediately after startup probe passes
      periodSeconds: 5
      failureThreshold: 3 # 3 consecutive failures → remove from LB

    livenessProbe: # controls restart
      httpGet: { path: /health, port: 8080 }
      initialDelaySeconds: 0
      periodSeconds: 10
      failureThreshold: 5 # 5 failures → restart container
```

```go
// /ready endpoint — checks real dependencies
func readyHandler(w http.ResponseWriter, r *http.Request) {
    if err := db.Ping(); err != nil {
        w.WriteHeader(http.StatusServiceUnavailable) // 503 → not ready
        return
    }
    w.WriteHeader(http.StatusOK) // 200 → ready
}

// /health endpoint — lightweight, no dependencies
func healthHandler(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK) // just confirms process is alive
}
```

- **Technical Terms to Include:** liveness probe, readiness probe, startup probe, `failureThreshold`, `successThreshold`, `periodSeconds`, `initialDelaySeconds`, endpoint removal, pod restart, probe types (httpGet/tcpSocket/exec/grpc)
- **Gotcha:** **Liveness probes that check external dependencies (DB, cache) are an anti-pattern.** If the database is down, every pod's liveness probe fails → every pod restarts → you now have a pod restart storm on top of a database outage. Liveness should only detect unrecoverable process-internal states. Readiness handles dependency checks — it takes pods out of traffic rotation without restarting them.
- **Follow-Up:** "What happens if you don't configure a readiness probe?" → Without a readiness probe, Kubernetes considers a pod ready immediately when the container starts — before your application has actually initialised. Traffic is routed to the pod before it can serve requests → connection errors and 5xx during rolling deployments. Always configure a readiness probe for any service that handles HTTP traffic. → They're testing zero-downtime deployment prerequisites.
- **Conclusion:** The three probes serve distinct purposes — startup gates the other probes during slow initialisation, readiness controls traffic eligibility, liveness controls restart — misconfiguring liveness to check external dependencies is one of the most common and most damaging Kubernetes anti-patterns in production.

---

#### RBAC & Security

- **Question:** How does Kubernetes RBAC work, and what is the principle of least privilege applied to service accounts and workloads?
- **Answer:**
  - **RBAC (Role-Based Access Control):** Controls which subjects (users, groups, service accounts) can perform which actions (verbs: get, list, watch, create, update, patch, delete) on which resources (pods, deployments, secrets) in which namespaces.
  - **Objects:** `Role` (namespace-scoped) + `RoleBinding` (binds role to subject in namespace). `ClusterRole` (cluster-wide) + `ClusterRoleBinding` (binds cluster-wide). A `RoleBinding` can bind a `ClusterRole` to a subject in a specific namespace — gives cluster-wide role definition with namespace-scoped binding.
  - **ServiceAccount:** An identity for pods to authenticate to the API server. Every pod runs as a ServiceAccount (default: `default` in the namespace). API server tokens are automatically mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`. Pods that need API access (operators, CI agents, monitoring) use dedicated ServiceAccounts with the minimum required RBAC.
  - **Least privilege:** Default `default` ServiceAccount has no permissions beyond basic discovery. Grant permissions explicitly. Never use `ClusterRole: cluster-admin` for application workloads. Audit with `kubectl auth can-i --list --as system:serviceaccount:namespace:name`.
  - **Pod Security Standards (K8s 1.25+):** Enforced via `PodSecurity` admission: `privileged`, `baseline`, `restricted`. `restricted` is the target for production — no root, no host namespaces, no privileged containers, seccomp required.
- **Example:**

```yaml
# ServiceAccount for a metrics scraper
apiVersion: v1
kind: ServiceAccount
metadata:
  name: metrics-reader
  namespace: monitoring
---
# Role — only what's needed
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-metrics-reader
rules:
  - apiGroups: ['']
    resources: ['pods', 'nodes']
    verbs: ['get', 'list', 'watch'] # read-only — no create/delete/update
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: metrics-reader-binding
subjects:
  - kind: ServiceAccount
    name: metrics-reader
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: pod-metrics-reader
  apiGroup: rbac.authorization.k8s.io
---
# Pod using dedicated ServiceAccount
spec:
  serviceAccountName: metrics-reader
  automountServiceAccountToken: false # opt out if API access not needed
```

- **Technical Terms to Include:** RBAC, Role, ClusterRole, RoleBinding, ClusterRoleBinding, ServiceAccount, subject, verb, resource, namespace-scoped, cluster-scoped, Pod Security Standards, `automountServiceAccountToken`, audit log
- **Gotcha:** The `default` ServiceAccount token is automounted into every pod unless `automountServiceAccountToken: false` is set. This means any process running in a pod can authenticate to the Kubernetes API server with the pod's identity — even if that pod has no legitimate reason to access the API. Explicitly disable automounting for all application pods that don't need API access.
- **Follow-Up:** "How do you audit what permissions a ServiceAccount has?" → `kubectl auth can-i --list --as=system:serviceaccount:<namespace>:<name>` lists all allowed actions for the ServiceAccount. For deeper audit: `kubectl get rolebinding,clusterrolebinding -A -o json | jq` to find all bindings referencing the ServiceAccount. Tools like `rbac-tool` or `rakkess` provide visual RBAC analysis. → They're testing operational RBAC awareness.
- **Conclusion:** Kubernetes RBAC's power is in its granularity — verb-level control on specific resources, namespace-scoped bindings, and per-pod ServiceAccount identities — but its default permissiveness (automounted token, broad default ClusterRole bindings) means least privilege requires explicit, deliberate configuration, not assumption.

---

#### Resource Management & HPA

- **Question:** How do resource requests and limits work in Kubernetes, what is the QoS class system, and how does HPA scale pods?
- **Answer:**
  - **Resource requests:** The minimum guaranteed resources for a pod. The scheduler uses requests to place pods on nodes with sufficient available capacity. A pod always gets at least its requested CPU/memory.
  - **Resource limits:** The maximum resources a pod can consume. CPU: throttled when limit is reached (no kill). Memory: OOMKilled when limit is exceeded (kernel kills the process). No limit = pod can consume all node resources (noisy neighbour).
  - **QoS classes (auto-assigned by Kubernetes):** (1) **Guaranteed** — request == limit for all containers. Highest priority, last to be evicted under pressure. (2) **Burstable** — request < limit, or only some containers have requests/limits. Middle priority. (3) **BestEffort** — no requests or limits at all. First to be evicted. Never use BestEffort in production.
  - **HPA (Horizontal Pod Autoscaler):** Scales pod replicas based on metrics. Default: CPU utilisation (`targetCPUUtilizationPercentage`). Custom: any metric via the Metrics API (memory, custom app metrics via Prometheus Adapter, external metrics). Scales up fast (every 15s), scales down slowly (stabilisation window default: 5 minutes) to prevent thrashing.
- **Example:**

```yaml
# Resource requests and limits — Guaranteed QoS
containers:
  - name: api
    resources:
      requests:
        cpu: '250m' # 0.25 CPU cores guaranteed
        memory: '256Mi' # 256MB guaranteed
      limits:
        cpu: '250m' # same as request → Guaranteed QoS
        memory: '256Mi' # OOMKilled if exceeded

---
# HPA — scale 2–10 replicas based on CPU
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70 # scale up when avg CPU > 70%
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300 # wait 5 min before scaling down
```

- **Technical Terms to Include:** resource requests, resource limits, CPU throttling, OOMKilled, QoS class (Guaranteed/Burstable/BestEffort), HPA, VPA (Vertical Pod Autoscaler), Metrics API, stabilisation window, KEDA (event-driven autoscaler)
- **Gotcha:** Setting a CPU limit lower than the actual usage of the application causes **CPU throttling** — the container is artificially slowed, increasing latency without the container appearing unhealthy. Unlike memory OOM (visible via OOMKilled), CPU throttling is silent. Monitor `container_cpu_cfs_throttled_periods_total / container_cpu_cfs_periods_total > 0.25` — more than 25% of CPU scheduling periods throttled indicates a too-low CPU limit.
- **Follow-Up:** "What is KEDA and when do you use it over HPA?" → KEDA (Kubernetes Event-Driven Autoscaling) extends HPA with event-source scaling — scale based on Kafka consumer group lag, Redis queue length, SQS queue depth, HTTP requests per second. HPA scales on resource metrics; KEDA scales on business/event metrics. For queue-based workloads, KEDA enables scaling to zero (no queue = 0 replicas) which HPA cannot do. → They're testing advanced autoscaling architecture.
- **Conclusion:** Resource requests and limits define the Kubernetes scheduling and eviction contract — QoS class Guaranteed prevents eviction under pressure, and HPA combined with correct resource configuration provides the self-scaling foundation for production workloads under variable load.

---

_— End of ContainerInfra_Interview_QA.md —_
_Coverage: Docker (L1–L4, 3 QA units) | Kubernetes (L1–L5, 5 QA units)_
_Calibrated for: Tech Lead / Architect level_
