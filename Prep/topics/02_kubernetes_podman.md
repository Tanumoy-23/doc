# TOPIC 2: KUBERNETES & PODMAN FOR INTERVIEWS
## Interview Preparation with Deep Questions & Answers

---

## 📚 CORE CONCEPTS

### 1. Kubernetes Architecture

**Master Components:**
- API Server: RESTful API for all operations
- etcd: Distributed key-value store (cluster state)
- Scheduler: Assigns pods to nodes
- Controller Manager: Manages controllers

**Node Components:**
- kubelet: Agent on each node, manages pods
- Container runtime: Docker/Podman/containerd
- kube-proxy: Network proxy, manages services

**Your platform:** You manage multi-tenant clusters - understand these deeply.

### 2. Pod, Deployment, StatefulSet, DaemonSet

**Pod:**
- Smallest Kubernetes unit
- One or more containers (usually one)
- Shared network namespace
- Ephemeral (replaced frequently)

**Deployment:**
- Manages ReplicaSets
- Rolling updates
- Desired state management
- Use for stateless apps

**StatefulSet:**
- For stateful apps
- Stable network identity
- Persistent storage
- Ordered scaling

**DaemonSet:**
- One pod per node
- Monitoring, logging, networking agents
- Always running

**For multi-tenancy:**
- Deployments for customer apps
- StatefulSet if they need persistence
- DaemonSet for platform monitoring

### 3. Namespaces for Multi-Tenancy

**What:**
- Virtual cluster within cluster
- Resource quotas per namespace
- RBAC per namespace
- Network policies between namespaces

**For your platform:**
```
Per-tenant namespace:
- Namespace: tenant-1, tenant-2, tenant-3
- ResourceQuota: CPU, memory limits
- NetworkPolicy: Prevent cross-tenant traffic
- RBAC: Tenant can only access their namespace
```

### 4. Services and Ingress

**Service:**
- Stable IP + DNS for pods
- Load balancing
- ClusterIP, NodePort, LoadBalancer types

**Ingress:**
- External HTTP(S) routing
- Host-based routing
- Path-based routing
- TLS termination

**For platform:**
- ClusterIP for internal services
- Ingress for customer access
- TLS via cert-manager

### 5. ConfigMaps and Secrets

**ConfigMap:**
- Non-sensitive configuration
- Environment variables
- Config files

**Secrets:**
- Sensitive data (passwords, tokens, API keys)
- Encrypted at rest (depends on storage)
- Never log or display secrets

**Critical:** Never put secrets in ConfigMaps!

### 6. Persistent Volumes (PV) and PVC

**PersistentVolume (PV):**
- Cluster-wide storage resource
- Created by admin or dynamic provisioning
- Independent of pod lifecycle

**PersistentVolumeClaim (PVC):**
- Request for storage
- Pod uses PVC, which uses PV
- Storage survives pod deletion

**For multi-tenant:**
- Each tenant gets their own PVC
- Different storage classes (fast vs cheap)
- Storage quotas per tenant

### 7. Podman vs Docker

**Container Runtime Comparison:**

Docker:
- Industry standard
- Has daemon (always running)
- Container is owned by Docker daemon

Podman:
- Daemonless (no always-running process)
- Same CLI as Docker
- Rootless containers (better security)
- Can run as non-root user

**Key difference:**
```
Docker: docker run image
Podman: podman run image
```
(Same CLI, different backend)

**For your platform:**
- Either works with Kubernetes
- Podman emerging alternative
- Both supported by Kubernetes (container runtime interface)

---

## 🎤 INTERVIEW QUESTIONS & ANSWERS

### Q1: "What is Kubernetes and why use it?"

**Answer:**
> "Kubernetes is a container orchestration platform. Manages containerized applications at scale.
>
> **Why:**
> 1. **Automated deployment:** Place containers optimally
> 2. **Scaling:** Scale up/down on demand
> 3. **Self-healing:** Restart failed containers
> 4. **Rolling updates:** Deploy new versions without downtime
> 5. **Resource management:** CPu/memory requests & limits
> 6. **Multi-tenant:** Isolation via namespaces
>
> **At our platform:**
> We use Kubernetes to:
> - Host customer applications (in separate namespaces)
> - Manage scale (auto-scaling)
> - Ensure isolation (network policies, RBAC)
> - Automate deployments (GitOps)
> - Track costs (resource limits per tenant)"

---

### Q2: "Explain Kubernetes namespace and how you'd use them for multi-tenancy"

**Answer:**
> "Namespace: Virtual cluster within cluster. Logical isolation.
>
> **For multi-tenancy:**
>
> 1. **One namespace per tenant:**
> ```
> Namespace: tenant-1
>   - All their pods, services, storage
>   - ResourceQuota: CPU 2, Memory 4Gi
>   - NetworkPolicy: No traffic from other namespaces
>   - RBAC: Tenant can only access tenant-1
> ```
>
> 2. **Resource isolation:**
> ```
> apiVersion: v1
> kind: ResourceQuota
> metadata:
>   name: compute-quota
>   namespace: tenant-1
> spec:
>   hard:
>     requests.cpu: '2'
>     requests.memory: 4Gi
>     limits.cpu: '4'
>     limits.memory: 8Gi
> ```
> Tenant can't use more than this.
>
> 3. **Network isolation:**
> ```
> NetworkPolicy: Deny all ingress by default
> Then allow: only traffic from own namespace
> ```
> Other tenants can't reach your pods.
>
> 4. **RBAC:**
> ```
> Role: Limited permissions for tenant
> Binding: User can only do specific things
> ```
>
> 5. **Cost tracking:**
> ```
> Each namespace has quota
> Monitor usage: actual vs quota
> Charge tenant based on actual usage
> ```
>
> **Benefits:**
> - True isolation
> - Resource limits (one tenant can't starve others)
> - Cost attribution (know who used what)
> - Compliance (separate access)
>
> **At our platform:**
> This is exactly how we do multi-tenancy. Each customer gets namespace with strict quotas."

---

### Q3: "What's the difference between a Pod and a Deployment?"

**Answer:**
> "Pod:
> - Smallest K8s unit
> - One or more containers
> - Ephemeral (temporary, can be deleted anytime)
> - Usually created by Deployment, not manually
>
> Deployment:
> - Manages Pods (via ReplicaSet)
> - Maintains desired state
> - Rolling updates (zero downtime)
> - Creates new pods when old ones die
> - Scaling (increase/decrease replicas)
>
> **Analogy:**
> - Pod: Individual container instance
> - Deployment: 'Run 3 copies, replace if they crash'
>
> **When to use:**
> - Pod: Debugging, one-off jobs
> - Deployment: Production apps (stateless)
> - StatefulSet: Stateful apps (databases)
> - DaemonSet: System agents (monitoring)
>
> **Our platform:**
> Customer apps run as Deployments (managed by Helm).
> We handle pod lifecycle, scaling, updates."

---

### Q4: "How would you ensure one tenant's workload doesn't affect another's?"

**Answer:**
> "Multi-tenant isolation has multiple layers:
>
> 1. **Namespace isolation:**
>    - One namespace per tenant
>    - Kubernetes API enforces separation
>
> 2. **Resource quotas:**
> ```
> ResourceQuota per namespace:
> - CPU: tenant-1 max 2 cores
> - Memory: tenant-1 max 4Gi
> If tenant-1 tries to use more, request is rejected.
> Prevents: noisy neighbor problem
> ```
>
> 3. **Pod resource requests/limits:**
> ```
> spec:
>   containers:
>   - resources:
>       requests:
>         cpu: '100m'      # Need 0.1 CPU minimum
>         memory: '128Mi'  # Need 128Mi minimum
>       limits:
>         cpu: '500m'      # Max 0.5 CPU
>         memory: '512Mi'  # Max 512Mi
> ```
> If pod uses > 512Mi, it gets OOMKilled.
>
> 4. **Network policies:**
> ```
> Deny all ingress by default
> Allow only:
> - Traffic from same namespace
> - Traffic from specific services
> ```
> Tenant-2 pods can't reach tenant-1 pods.
>
> 5. **RBAC (Role-Based Access Control):**
> ```
> Tenant user can only:
> - View their namespace
> - Create pods in their namespace
> - Can't view other namespaces
> ```
>
> 6. **Storage isolation:**
>    - Each tenant's PVC is separate
>    - Different storage classes if needed
>    - Can't access other's data
>
> 7. **Monitoring & limits:**
>    - Monitor CPU/memory per namespace
>    - Alert if usage approaching quota
>    - Track costs per tenant
>
> **What happens if tenant exceeds quota:**
> ```
> Scenario: Tenant-1 tries to create 10 pods but hits quota
> Result: Pod stays pending, not scheduled
> Message: 'Insufficient quota'
> ```
>
> **At our platform:**
> All 7 layers implemented. One tenant absolutely cannot affect another."

---

### Q5: "Explain Kubernetes services and when you'd use each type"

**Answer:**
> "Service: Stable IP + DNS for pods. Abstracts pod ephemeral nature.
>
> **Three main types:**
>
> 1. **ClusterIP (default):**
>    - Internal IP only
>    - Accessible only within cluster
>    - Use: Internal services (database, cache)
>
> 2. **NodePort:**
>    - Opens port on every node
>    - External traffic reaches that port
>    - Routed to service
>    - Use: External access without load balancer
>    - Limitation: Ports 30000-32767 only
>
> 3. **LoadBalancer:**
>    - Provisions external load balancer (if cloud provider)
>    - External IP assigned
>    - Traffic routed to service
>    - Use: Production external services
>    - Cost: Expensive (separate LB per service)
>
> **For our platform:**
>
> ```
> Service type: ClusterIP
> - Platform services talk via Service DNS
> - Example: 'mongodb.platform.svc'
> - Not exposed externally
>
> Service type: LoadBalancer
> - Customer applications exposed via LoadBalancer
> - Each customer gets external IP
> - TLS via Ingress/cert-manager
> ```
>
> **Bonus: Ingress**
> - Higher level than Service
> - Route based on hostname/path
> - Example:
>   - customer1.platform.com → customer-1-service
>   - customer2.platform.com → customer-2-service
> - TLS termination at Ingress
> - More efficient than LoadBalancer per service"

---

### Q6: "What are Persistent Volumes and why do you need them?"

**Answer:**
> "Problem: Pods are ephemeral. When pod dies, all data inside is lost.
>
> Solution: PersistentVolume (PV) + PersistentVolumeClaim (PVC)
>
> **Flow:**
> ```
> 1. Admin creates PersistentVolume
>    (actual storage: NFS, EBS, local disk)
>
> 2. App makes PersistentVolumeClaim
>    (request: 'I need 10Gi storage')
>
> 3. K8s binds PVC to PV
>
> 4. Pod uses PVC
>    (mount point: /data/)
>
> 5. Pod dies, new pod created
>    (new pod mounts same PVC)
>    (data is preserved!)
> ```
>
> **For multi-tenant:**
> ```
> Each tenant gets:
> - PersistentVolumeClaim (their storage request)
> - Bound to separate PersistentVolume
> - Storage quota enforced
> - Data isolated
>
> Tenant-1 PVC → PV1 (10Gi NFS)
> Tenant-2 PVC → PV2 (10Gi NFS)
> One tenant's data doesn't leak to other
> ```
>
> **Storage classes:**
> ```
> Fast storage (SSD): For databases
> Slow storage (HDD): For backups, archives
>
> Tenant can choose:
> storage-class: fast → More expensive
> storage-class: slow → Cheaper
> ```
>
> **At our platform:**
> - Tenant databases use Persistent storage
> - StatefulSet + PVC for databases
> - Different storage classes available
> - Storage quotas enforced"

---

### Q7: "How would you handle pod failures?"

**Answer:**
> "Kubernetes handles pod failures automatically (with proper setup).
>
> **Pod failure scenarios:**
> 1. Container crash
> 2. Node failure (whole node dies)
> 3. Resource exhaustion (out of memory)
> 4. Bad deployment
>
> **Handling:**
>
> 1. **Restart policy:**
> ```
> spec:
>   restartPolicy: Always  # Restart crashed containers
> ```
> If container crashes, kubelet restarts it.
>
> 2. **Liveness probe:**
> ```
> livenessProbe:
>   httpGet:
>     path: /health
>     port: 8080
>   initialDelaySeconds: 30
>   periodSeconds: 10
> ```
> If probe fails, pod is killed & restarted.
> Detects: Hung process (not responding but running)
>
> 3. **Readiness probe:**
> ```
> readinessProbe:
>   httpGet:
>     path: /ready
>     port: 8080
>   periodSeconds: 5
> ```
> If probe fails, pod removed from service.
> Detects: Pod starting up (not ready to serve requests)
>
> 4. **Deployment replicas:**
> ```
> spec:
>   replicas: 3
> ```
> If 1 pod dies, K8s creates new one (always 3 running).
>
> 5. **Pod disruption budgets:**
> ```
> minAvailable: 2
> ```
> During cluster maintenance, at least 2 pods stay running.
>
> 6. **Node affinity:**
> ```
> Spread pods across nodes
> If node dies, other pods on other nodes still running
> ```
>
> **At our platform:**
> - All deployments have resource limits
> - Liveness + readiness probes
> - Multiple replicas (high availability)
> - Pod disruption budgets
> - Automatic rollout on failed deployments"

---

### Q8: "What's the difference between ConfigMap and Secret?"

**Answer:**
> "ConfigMap: Non-sensitive configuration (OK to commit to git)
> Secret: Sensitive data (never commit!)
>
> **ConfigMap example:**
> ```yaml
> kind: ConfigMap
> data:
>   DATABASE_HOST: 'mongodb.default.svc'
>   LOG_LEVEL: 'info'
>   FEATURE_FLAG_X: 'true'
> ```
> These are fine in git, available to containers as env vars.
>
> **Secret example:**
> ```yaml
> kind: Secret
> data:
>   API_KEY: 'eyJ...'  # base64 encoded
>   PASSWORD: 'dGVz...'
> ```
> Never commit to git!
>
> **How to use:**
> ```yaml
> # ConfigMap
> env:
> - name: DB_HOST
>   valueFrom:
>     configMapKeyRef:
>       name: app-config
>       key: DATABASE_HOST
>
> # Secret
> - name: API_KEY
>   valueFrom:
>     secretKeyRef:
>       name: api-secrets
>       key: API_KEY
> ```
>
> **Security:**
> - Secrets are base64 encoded (not encrypted by default!)
> - Enable encryption at rest in etcd
> - Use external secret manager (HashiCorp Vault)
> - Never log secrets
> - Limit who can read secrets (RBAC)
>
> **At our platform:**
> - ConfigMaps for app config (per tenant)
> - Secrets for API keys, credentials
> - Encryption at rest enabled
> - RBAC restricts secret access
> - Regular rotation of secrets"

---

### Q9: "How would you implement rolling updates with zero downtime?"

**Answer:**
> "Problem: Deploy new version without users seeing downtime.
>
> **Solution: Rolling update**
> ```
> Current state: 3 pods running v1
>
> 1. Create 1 pod v2 (now 4 pods: 3 v1, 1 v2)
> 2. Service routes traffic to both (gradually)
> 3. Old pod v1 is removed (now 3 pods: 2 v1, 1 v2)
> 4. Create another v2 (now 3 pods: 1 v1, 2 v2)
> 5. Remove v1 (now 3 pods: 3 v2)
> 6. Done! (All v2)
>
> During update: Always 3+ pods running
> No request loss if each pod takes time to drain
> ```
>
> **Kubernetes config:**
> ```yaml
> spec:
>   replicas: 3
>   strategy:
>     type: RollingUpdate
>     rollingUpdate:
>       maxSurge: 1        # Max 4 pods during update
>       maxUnavailable: 0  # Min 3 pods always available
>   
>   template:
>     spec:
>       containers:
>       - name: app
>         image: myapp:v2  # New version
>         
>       terminationGracePeriodSeconds: 30
>       # Give pods 30s to finish requests before killing
> ```
>
> **Key settings:**
> - maxSurge: How many extra pods during update
> - maxUnavailable: How many pods can be down
> - terminationGracePeriodSeconds: Time to drain requests
>
> **Blue-green alternative:**
>
> ```
> Blue (current): 3 pods v1
> Green (new): 3 pods v2
>
> Test green pods
> When ready: Switch service to green
> If problem: Switch back to blue
> ```
> Faster switchover, uses 2x resources during update.
>
> **At our platform:**
> - Rolling updates for stateless services
> - Blue-green for critical services
> - Monitoring during updates
> - Automatic rollback on errors
> - Helm handles deployment strategy"

---

### Q10: "Explain Podman and how it differs from Docker"

**Answer:**
> "Podman: Container runtime alternative to Docker.
>
> **Key differences:**
>
> | Feature | Docker | Podman |
> |---------|--------|--------|
> | Architecture | Daemon | Daemonless |
> | Root required | Yes | No (rootless) |
> | CLI | docker run | podman run |
> | Pod support | No | Yes (native) |
> | Kubernetes | Works | Works (native) |
> | Security | Daemon runs as root | Can run as user |
>
> **Daemonless means:**
> - No background docker daemon always running
> - Each podman command starts fresh
> - Lower resource usage
> - Better security (no privilege escalation through daemon)
>
> **Rootless means:**
> - Run containers as regular user
> - Not root
> - Better security (container escape = restricted user)
>
> **For your platform:**
>
> Both work with Kubernetes equally well.
> Choose based on:
> - Existing tooling (Docker ecosystem larger)
> - Security requirements (Podman better)
> - Resource constraints (Podman lighter)
>
> At EV: Either Docker or Podman works. Podman trending up for security."

---

### Q11: "Design a multi-tenant Kubernetes cluster"

**Answer:**
> "Architecture for 100+ customers sharing one cluster:
>
> **Design:**
> ```
> Cluster (shared)
>   ├─ kube-system (platform pods)
>   ├─ monitoring (Prometheus, Grafana)
>   ├─ ingress (Nginx Ingress Controller)
>   ├─ tenant-1 (namespace)
>   ├─ tenant-2 (namespace)
>   └─ tenant-N (namespace)
> ```
>
> **Per-tenant namespace includes:**
> ```
> Namespace: tenant-1
>   ├─ Deployments (customer apps)
>   ├─ Services (customer services)
>   ├─ PVC (customer storage)
>   ├─ ConfigMap (customer config)
>   ├─ Secrets (customer credentials)
>   └─ ResourceQuota (CPU 2, Memory 4Gi)
> ```
>
> **Isolation mechanisms:**
> 1. Namespace (API segregation)
> 2. ResourceQuota (CPU/memory limits)
> 3. NetworkPolicy (no cross-tenant traffic)
> 4. RBAC (access control)
> 5. PVC (storage isolation)
> 6. Pod security policy (runtime constraints)
>
> **Shared services:**
> - Ingress Controller (routes external traffic)
> - Prometheus (monitors all tenants)
> - Logging (centralized logs)
> - Certificate management
>
> **Cost tracking:**
> - Monitor actual CPU/memory per namespace
> - Track: requests, limits, actual usage
> - Calculate cost per tenant
> - Charge based on actual usage
>
> **High availability:**
> - Multi-node cluster (3+ nodes)
> - Pod disruption budgets
> - Storage replicated
> - Automatic failover
>
> **At our platform:**
> This is our setup at Ai-Next. 100+ customers, single cluster, isolated namespaces."

---

## ✅ FINAL CHECKLIST

- [ ] Understand K8s architecture (master + node components)
- [ ] Know pod vs deployment vs statefulset
- [ ] Can design multi-tenant namespace isolation
- [ ] Understand resource quotas + limits
- [ ] Can explain services (ClusterIP, NodePort, LoadBalancer)
- [ ] Know PV/PVC for persistent storage
- [ ] Understand probes (liveness, readiness)
- [ ] Can design rolling updates
- [ ] Know Podman vs Docker
- [ ] Can design RBAC for multi-tenant
- [ ] Understand network policies for isolation

---

*Generated: April 23, 2026 | Kubernetes Interview Preparation*
