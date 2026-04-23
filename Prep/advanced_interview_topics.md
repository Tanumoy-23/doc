# ADVANCED PLATFORM ENGINEERING INTERVIEW QUESTIONS
## KEDA, SPOT, Rate Limiting, Resiliency, Rollback, Observability, eBPF

---

## TOPIC 1: KEDA (Kubernetes Event-Driven Autoscaling)

### What is KEDA and why use it?

**Q1: "What's the difference between HPA and KEDA? When would you use each?"**

**Model Answer:**

> "HPA (Horizontal Pod Autoscaler) scales based on metrics:
> - CPU: 'If CPU > 80%, add pods'
> - Memory: 'If memory > 70%, add pods'
> - Custom metrics: 'If request latency > 500ms, scale up'
>
> Limitation: HPA only understands metrics (Prometheus), not events.
>
> KEDA (Kubernetes Event-Driven Autoscaling) scales based on external events:
> - Kafka queue depth: 'If queue > 1000 messages, scale up'
> - RabbitMQ queue: 'If queue > 500, scale up'
> - AWS SQS: 'If queue has messages, scale up'
> - External HTTP endpoint: Custom event trigger
> - Database queries: 'If pending_jobs > 100, scale up'
> - Cloud provider events: Pub/Sub, Kinesis, etc.
>
> **When to use HPA:**
> - Synchronous APIs (request-response)
> - Predictable scaling (CPU/memory based)
> - Example: REST API, GraphQL server
>
> **When to use KEDA:**
> - Asynchronous workloads (event processing)
> - Queue-based architectures
> - External event triggers
> - Example: Message processing, batch jobs, stream processing
>
> **At your platform (Ai-Next):**
> We could use KEDA for:
> - Scale document analysis based on upload queue
> - Scale report generation based on pending_reports table
> - Scale batch processing based on job queue depth
> - Scale webhook handlers based on incoming events
>
> Combination: HPA for API servers + KEDA for workers"

---

### Q2: "Design KEDA scaler for multi-tenant document processing system"

**Scenario:**
```
Your platform:
- Customers upload documents
- Documents queued for processing (PostgreSQL)
- Workers process documents asynchronously
- Goal: Scale workers based on queue depth
```

**Model Answer:**

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: document-processor-scaler
  namespace: platform
spec:
  # What to scale
  scaleTargetRef:
    name: document-processor
    kind: Deployment
  
  # Min and max replicas
  minReplicaCount: 1
  maxReplicaCount: 100
  
  # How aggressively to scale
  pollingInterval: 15  # Check every 15 seconds
  cooldownPeriod: 300  # Wait 5 min before scaling down
  
  # Scale-up more aggressively than scale-down
  fallback:
    failureThreshold: 3
    replicas: 1
  
  triggers:
  # Trigger 1: PostgreSQL queue depth (primary)
  - type: postgresql
    metadata:
      query: "SELECT COUNT(*) as pending FROM documents WHERE status='pending' AND tenant_id=$1"
      targetQueryValue: "100"  # Scale up if > 100 pending
      connectionStringRef:
        name: postgres-conn
        key: connection-string
  
  # Trigger 2: CPU (fallback if queue query fails)
  - type: cpu
    metricType: Utilization
    metadata:
      value: "70"  # Scale at 70% CPU
      
advanced_scaler_config:
  # Don't scale down too fast (avoid thrashing)
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50  # Max 50% reduction per scale-down
        periodSeconds: 60
    
    scaleUp:
      stabilizationWindowSeconds: 0  # Scale up immediately
      policies:
      - type: Percent
        value: 100  # Can double the pods
        periodSeconds: 15
      - type: Pods
        value: 10  # Or add 10 pods per scale event
        periodSeconds: 15
      selectPolicy: Max  # Use whichever adds more pods

# Multi-tenant consideration:
# - Query filters by tenant_id
# - Per-tenant queue depth
# - Could create separate ScaledObject per tenant
# - Or use single scaler that handles all tenants
```

**Implementation:**

```python
# In document processor (worker):
import os
import logging
from keda_client import get_scaling_metrics

logger = logging.getLogger(__name__)

class DocumentProcessor:
    def __init__(self):
        self.tenant_id = os.getenv('TENANT_ID')
        self.db = init_database()
    
    def process_queue(self):
        """Process documents from queue"""
        while True:
            # Get pending document
            doc = self.db.query(
                "SELECT * FROM documents "
                "WHERE status='pending' AND tenant_id=$1 "
                "LIMIT 1",
                self.tenant_id
            )
            
            if not doc:
                logger.info(f"No pending documents for {self.tenant_id}")
                sleep(5)
                continue
            
            try:
                # Process document
                result = self.process_document(doc)
                
                # Mark as done
                self.db.execute(
                    "UPDATE documents SET status='done', result=$1 WHERE id=$2",
                    result, doc.id
                )
                
                # Log for KEDA metrics
                logger.info(f"Processed document {doc.id}, queue_depth={self.get_queue_depth()}")
                
            except Exception as e:
                logger.error(f"Error processing {doc.id}: {e}")
                # Increment retry count, mark as failed if too many retries
    
    def get_queue_depth(self):
        """Get current queue depth (what KEDA queries)"""
        count = self.db.query_scalar(
            "SELECT COUNT(*) FROM documents WHERE status='pending' AND tenant_id=$1",
            self.tenant_id
        )
        return count
    
    def process_document(self, doc):
        """Actual processing logic"""
        # Call GenAI for analysis
        # Generate PDF report
        # etc.
        pass

# KEDA queries this endpoint every 15 seconds:
@app.get('/metrics/queue-depth')
def get_queue_depth():
    # KEDA calls this to decide scaling
    depth = db.query_scalar(
        "SELECT COUNT(*) FROM documents WHERE status='pending'"
    )
    return {
        'queue_depth': depth,
        'timestamp': now()
    }
```

**Monitoring KEDA scaling:**

```yaml
# Prometheus metrics KEDA exposes:
# keda_scaler_active (is scaler active?)
# keda_scaler_metrics_value (current metric value)
# keda_scaler_metrics_latency (time to query metric)

# Grafana dashboard:
Dashboard: "KEDA Scaling Activity"

Panels:
- Pod count over time (min, max, current)
- Queue depth vs pod count (correlation)
- Scale events (when did we scale up/down?)
- Scaler latency (how long to query PostgreSQL?)
- Scale-up lag (time from queue spike to scale complete)
- Cost impact (more pods = more cost)
```

---

### Q3: "Your KEDA scaler is scaling too slowly. Latency increases before pods are ready. How do you fix?"

**Model Answer:**

> "KEDA scaling latency has multiple components:
>
> **Problem breakdown:**
> ```
> Event happens (document uploaded)
>   ↓ (0-15s: wait for next KEDA poll)
> KEDA polls queue depth
>   ↓ (1-2s: PostgreSQL query)
> KEDA receives metric value
>   ↓ (0.1s: calculate scale decision)
> KEDA triggers scale action
>   ↓ (5-30s: Kubernetes scheduler)
> Pod scheduled on node
>   ↓ (5-15s: pull image, start container)
> Pod ready
>   ↓ (Total: 20-65 seconds from event to handling!)
> ```
>
> **Solutions (in order of impact):**
>
> **1. Faster polling (quick win):**
> ```yaml
> pollingInterval: 15  # Was 30s, now 15s
> ```
> Impact: Reduces idle time by 7.5s
> Cost: Slightly higher PostgreSQL load
>
> **2. Reduce cooldownPeriod (scale faster):**
> ```yaml
> cooldownPeriod: 60  # Was 300s (5 min)
> ```
> Impact: Don't wait so long before scaling up
> Cost: Might scale too aggressively
>
> **3. Immediate scale-up behavior:**
> ```yaml
> scaleUp:
>   stabilizationWindowSeconds: 0  # No waiting
>   policies:
>   - type: Percent
>     value: 100  # Double pods immediately
>     periodSeconds: 15
> ```
> Impact: Pods scale instantly when queue spikes
> Cost: Might overprovision
>
> **4. Pre-warm pods (prevent cold start):**
> ```yaml
> minReplicaCount: 10  # Always keep 10 pods running
> ```
> Impact: Queue processed immediately by existing pods
> Cost: ₹500/month for 10 idle pods
>
> **5. Predict and pre-scale (advanced):**
> Use ML to predict queue spike before it happens
> - If uploads spike at certain times, pre-scale before
> - Example: Every Monday 9 AM, scale to 50 pods
> - Then KEDA scales down if not needed
>
> **6. Cache-based fastpath:**
> ```python
> # KEDA still scales normally
> # But handle queue immediately with in-memory cache
> 
> # When document uploaded:
> # 1. Add to fast cache (Redis)
> # 2. Return to user immediately
> # 3. Process from cache asynchronously
> # 4. KEDA scales pods normally in background
> # 5. Pods pull from cache
> ```
>
> **Recommended (your situation):**
> Combine 1 + 3 + 4:
> ```yaml
> pollingInterval: 10        # Check more frequently
> minReplicaCount: 5         # Pre-warm 5 pods
> cooldownPeriod: 60         # Quick scale-up
> scaleUp.stabilizationWindowSeconds: 0  # Immediate
> ```
> Impact: Latency 20-65s → 5-15s
> Cost: +5 pods = ₹250/month
> Payback: Faster processing = happier customers"

---

### Q4: "Design KEDA for multi-tenant GenAI platform with isolation requirements"

**Scenario:**
```
Multiple tenants, separate budgets:
- Tenant A: ₹10K/month, max 50 pods
- Tenant B: ₹20K/month, max 100 pods
- Tenant C: ₹5K/month, max 20 pods

Goal: Scale each tenant independently, respect budgets
```

**Model Answer:**

```yaml
# Approach 1: Per-tenant ScaledObject (recommended)

# Tenant A
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: genai-worker-tenant-a
  namespace: tenant-a
spec:
  scaleTargetRef:
    name: genai-worker
    kind: Deployment
  minReplicaCount: 1
  maxReplicaCount: 50  # Budget limit
  
  triggers:
  - type: postgresql
    metadata:
      query: "SELECT COUNT(*) FROM jobs WHERE status='pending' AND tenant_id='tenant-a'"
      targetQueryValue: "100"
      connectionStringRef:
        name: postgres-conn
        key: connection-string

# Tenant B (same, different limits)
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: genai-worker-tenant-b
  namespace: tenant-b
spec:
  scaleTargetRef:
    name: genai-worker
    kind: Deployment
  minReplicaCount: 1
  maxReplicaCount: 100  # Higher budget
  
  triggers:
  - type: postgresql
    metadata:
      query: "SELECT COUNT(*) FROM jobs WHERE status='pending' AND tenant_id='tenant-b'"
      targetQueryValue: "100"
      connectionStringRef:
        name: postgres-conn
        key: connection-string

# Tenant C (lower budget)
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: genai-worker-tenant-c
  namespace: tenant-c
spec:
  scaleTargetRef:
    name: genai-worker
    kind: Deployment
  minReplicaCount: 1
  maxReplicaCount: 20  # Budget limit
  
  triggers:
  - type: postgresql
    metadata:
      query: "SELECT COUNT(*) FROM jobs WHERE status='pending' AND tenant_id='tenant-c'"
      targetQueryValue: "50"
      connectionStringRef:
        name: postgres-conn
        key: connection-string
```

**Cost enforcement:**

```python
# Monitor to prevent overspending
class CostManager:
    def __init__(self, tenant_id, monthly_budget):
        self.tenant_id = tenant_id
        self.monthly_budget = monthly_budget  # e.g., ₹10,000
        self.cost_per_pod_hour = 100  # ₹100/hour
    
    def check_can_scale(self, requested_replicas):
        """Before scaling, check if within budget"""
        current_pods = get_current_pods(self.tenant_id)
        additional_pods = requested_replicas - current_pods
        
        if additional_pods <= 0:
            return True  # Scaling down is OK
        
        # Estimate cost of additional pods
        hours_remaining_month = 24 * (days_remaining_in_month())
        cost_of_additional = additional_pods * self.cost_per_pod_hour * hours_remaining_month
        spent_so_far = get_tenant_spend_to_date(self.tenant_id)
        
        if spent_so_far + cost_of_additional > self.monthly_budget:
            # Would exceed budget
            logger.warning(
                f"Tenant {self.tenant_id}: "
                f"Would exceed budget. "
                f"Spent: ₹{spent_so_far}, Budget: ₹{self.monthly_budget}, "
                f"Additional cost: ₹{cost_of_additional}"
            )
            return False
        
        return True
    
    def enforce_max_replicas(self, tenant_id):
        """Dynamically adjust maxReplicaCount based on budget"""
        spent = get_tenant_spend_to_date(tenant_id)
        remaining = self.monthly_budget - spent
        days_left = days_remaining_in_month()
        hours_left = days_left * 24
        
        max_pods = int(remaining / (self.cost_per_pod_hour * hours_left))
        
        # Update ScaledObject
        update_keda_max_replicas(tenant_id, max_pods)
        
        return max_pods

# Alerting:
alerts:
  - name: tenant_approaching_budget
    condition: tenant_spend > budget * 0.8
    action: notify_customer

  - name: tenant_exceeded_budget
    condition: tenant_spend > budget
    action: scale_down_to_minimum
```

**Multi-tenant isolation:**

```yaml
# ResourceQuota per tenant (in addition to KEDA)
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-a-quota
  namespace: tenant-a
spec:
  hard:
    pods: "50"  # Max 50 pods (matches KEDA maxReplicaCount)
    requests.cpu: "50"  # 50 cores max
    requests.memory: "100Gi"  # 100GB RAM max
    limits.cpu: "100"
    limits.memory: "200Gi"

# Pod disruption budget (don't evict all pods at once)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: genai-worker-pdb
  namespace: tenant-a
spec:
  minAvailable: 2  # At least 2 pods must stay running
  selector:
    matchLabels:
      app: genai-worker

# Network policy (multi-tenant isolation)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: tenant-a-isolation
  namespace: tenant-a
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: tenant-a
  - from:
    - namespaceSelector:
        matchLabels:
          name: platform  # Platform services can reach
```

---

## TOPIC 2: SPOT INSTANCES (Cost Optimization)

### Q1: "Design a hybrid strategy mixing on-demand and spot instances for cost optimization"

**Model Answer:**

> "Spot instances are 70-90% cheaper than on-demand, but can be terminated with 2-minute notice. We need a hybrid strategy.
>
> **Architecture:**
> ```
> Cluster:
> ├─ On-demand nodes (guaranteed)
> │  ├─ 3 nodes: API servers (stateless, critical)
> │  └─ 2 nodes: Databases (state, must stay running)
> │
> └─ Spot nodes (cheap, interruptible)
>    ├─ Up to 20 nodes: Workers (fault-tolerant)
>    ├─ Up to 10 nodes: Batch jobs
>    └─ Up to 5 nodes: GenAI processing
> ```
>
> **Characteristics:**
> - On-demand: High availability, predictable cost
> - Spot: Low cost, unpredictable, but fault-tolerant workloads
>
> **Cost analysis:**
> ```
> On-demand cost (m5.large): ₹100/hour
> Spot cost (m5.large): ₹25/hour (75% cheaper!)
>
> Cluster (5 on-demand + 20 spot):
> - 5 on-demand × ₹100 × 730 hours = ₹3,65,000/month
> - 20 spot × ₹25 × 730 hours = ₹3,65,000/month (!)
> - Total: ₹7,30,000/month
>
> If all on-demand:
> - 25 on-demand × ₹100 × 730 = ₹18,25,000/month
>
> Savings: 60% (₹11 lakh/month!)
> ```
>
> **Implementation:**
>
> 1. Node groups with different cost models
> ```yaml
> # On-demand node group (critical workloads)
> apiVersion: nodegroups/v1
> kind: NodeGroup
> metadata:
>   name: on-demand-critical
> spec:
>   machineType: m5.xlarge
>   desiredCount: 5
>   capacityType: on-demand
>   taints:
>   - key: workload-type
>     value: critical
>     effect: NoSchedule
> 
> # Spot node group (fault-tolerant workloads)
> ---
> apiVersion: nodegroups/v1
> kind: NodeGroup
> metadata:
>   name: spot-workers
> spec:
>   machineType: m5.xlarge
>   desiredCount: 20
>   capacityType: spot
>   spotPrice: ₹30/hour  # Max we'll pay
>   interruptionBehavior: terminate  # Or consolidate
> ```
>
> 2. Pod placement strategy
> ```yaml
> # Critical API servers → on-demand
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: api-server
> spec:
>   template:
>     spec:
>       nodeSelector:
>         workload-type: critical
>       tolerations:
>       - key: workload-type
>         operator: Equal
>         value: critical
>         effect: NoSchedule
>       affinity:
>         podAntiAffinity:
>           requiredDuringSchedulingIgnoredDuringExecution:
>           - labelSelector:
>               matchExpressions:
>               - key: app
>                 operator: In
>                 values:
>                 - api-server
>             topologyKey: kubernetes.io/hostname
> 
> # Workers → spot (with fallback to on-demand if needed)
> ---
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: genai-worker
> spec:
>   replicas: 50
>   template:
>     spec:
>       affinity:
>         nodeAffinity:
>           preferredDuringSchedulingIgnoredDuringExecution:
>           # Prefer spot (cheap)
>           - weight: 100
>             preference:
>               matchExpressions:
>               - key: karpenter.sh/capacity-type
>                 operator: In
>                 values: [spot]
>           # Fallback to on-demand if spot full
>           - weight: 10
>             preference:
>               matchExpressions:
>               - key: karpenter.sh/capacity-type
>                 operator: In
>                 values: [on-demand]
> ```
>
> 3. Handle spot terminations gracefully
> ```python
> import signal
> import threading
> from kubernetes import client, watch
> 
> class SpotTerminationHandler:
>     def __init__(self):
>         # AWS sends SIGTERM 2 minutes before termination
>         signal.signal(signal.SIGTERM, self.on_termination)
>         
>         # Also poll metadata service for termination notice
>         self.monitoring_thread = threading.Thread(
>             target=self.monitor_spot_termination,
>             daemon=True
>         )
>         self.monitoring_thread.start()
>     
>     def monitor_spot_termination(self):
>         \"\"\"Poll AWS metadata for termination notice\"\"\"
>         import requests
>         while True:
>             try:
>                 # AWS sends notice here
>                 response = requests.get(
>                     'http://169.254.169.254/latest/meta-data/spot/instance-action',
>                     timeout=1
>                 )
>                 if response.status_code == 200:
>                     # Spot termination notice received
>                     print('Spot termination notice received')
>                     self.graceful_shutdown()
>                     break
>             except requests.Timeout:
>                 pass  # No notice yet
>             time.sleep(5)
>     
>     def on_termination(self, sig, frame):
>         \"\"\"Handle SIGTERM from AWS\"\"\"
>         print('Received SIGTERM, graceful shutdown starting')
>         self.graceful_shutdown()
>     
>     def graceful_shutdown(self):
>         \"\"\"Drain work queue, finish current job, exit\"\"\"
>         logger.info('Graceful shutdown: stopping accepting new work')
>         self.stop_accepting_work = True
>         
>         # Give 90 seconds to finish current work
>         start = time.time()
>         while time.time() - start < 90:
>             if self.current_job_done:
>                 break
>             time.sleep(1)
>         
>         logger.info('Graceful shutdown: exiting')
>         sys.exit(0)
> 
> # In main processing loop:
> handler = SpotTerminationHandler()
> 
> while True:
>     if handler.stop_accepting_work:
>         # Don't take new jobs, wait for current to finish
>         time.sleep(1)
>         continue
>     
>     job = job_queue.get()
>     handler.current_job_done = False
>     try:
>         process_job(job)
>     finally:
>         handler.current_job_done = True
> ```
>
> 4. Automatic scaling with Karpenter (recommended)
> ```yaml
> # Karpenter automatically manages on-demand vs spot
> # Replaces KEDA for node-level scaling
>
> apiVersion: karpenter.sh/v1beta1
> kind: NodePool
> metadata:
>   name: default
> spec:
>   # Allow both capacity types, prefer spot
>   template:
>     spec:
>       requirements:
>       - key: karpenter.sh/capacity-type
>         operator: In
>         values: [spot, on-demand]
>       - key: kubernetes.io/arch
>         operator: In
>         values: [amd64]
>       - key: karpenter.sh/instance-family
>         operator: In
>         values: [m5, m6i, c5, c6i]
>   
>   # Limits
>   limits:
>     cpu: 1000
>     memory: 1000Gi
>   
>   # Consolidation (remove underutilized nodes)
>   consolidationPolicy:
>     nodes: \"underutilized\"
>     expireAfter: 720h  # One month
> ```
>
> **Monitoring & Cost:**
> ```
> Dashboard: \"Cost Breakdown\"
> 
> ├─ On-demand cost
> │  ├─ API servers: ₹3,65,000 (guaranteed)
> │  ├─ Databases: ₹1,46,000 (guaranteed)
> │
> ├─ Spot cost
> │  ├─ Workers: ₹2,55,000 (best effort)
> │  ├─ Batch: ₹1,27,500
> │
> ├─ Spot interruption rate: 2% (acceptable)
> ├─ Spot savings: ₹11 lakh/month
> └─ Total cost: ₹7,30,000/month (vs ₹18.25L all on-demand)
> ```"

---

### Q2: "Your spot instances are getting interrupted frequently (20% failure rate). How do you handle?"

**Model Answer:**

> "20% failure rate means 1 in 5 pods dies every 5 minutes. That's high and will cause problems.
>
> **Root causes:**
> 1. Spot price increased (close to on-demand)
> 2. AWS capacity constraint (limited spot capacity)
> 3. Instance type in low supply
>
> **Solutions (in order):**
>
> **1. Switch to different instance type (immediate)**
> ```yaml
> # Before (m5.large, expensive):
> - key: karpenter.sh/instance-family
>   operator: In
>   values: [m5]
> 
> # After (cheaper families):
> - key: karpenter.sh/instance-family
>   operator: In
>   values: [t3, t4g, m6i, m6a]  # More available
> ```
>
> **2. Increase on-demand ratio (cost trade-off)**
> ```
> Prefer spot, but fall back to on-demand:
> 
> affinity:
>   nodeAffinity:
>     preferredDuringSchedulingIgnoredDuringExecution:
>     - weight: 100  # Strongly prefer
>       preference: spot
>     - weight: 50
>       preference: on-demand  # But allow on-demand
> ```
> Cost: +₹100K/month, but stable
>
> **3. Fault tolerance (best solution)**
> ```
> If your workload already tolerates failures:
> - Multiple replicas (if one dies, others handle)
> - Distributed processing (each pod handles subset of data)
> - Async + retry (if pod dies, work requeued)
> 
> Then 20% failure rate is OK!
> ```
>
> **4. Spot price limit (force cheaper alternatives)**
> ```yaml
> # If price > threshold, don't provision
> spotPrice: ₹25/hour
> 
> # Karpenter won't provision if spot > ₹25
> # Will use on-demand instead
> ```
>
> **Monitoring:**
> ```
> Prometheus query:
> rate(pod_terminated_by_spot_interruption[5m])
>
> Alert: if interruption_rate > 10%, investigate
> 
> Dashboard: \"Spot Health\"
> - Interruption rate: 20% (TOO HIGH)
> - Average pod life: 5 minutes (should be hours/days)
> - Cost savings: Not worth the instability
> ```"

---

## TOPIC 3: REQUEST LIMITING & QUOTA MANAGEMENT

### Q1: "Design request rate limiting for multi-tenant SaaS platform"

**Model Answer:**

> "Multi-tenant rate limiting has multiple layers:
>
> **1. Global platform limit (protect yourself)**
> ```
> Total platform capacity: 10,000 req/sec
> 
> If traffic exceeds:
> - 80% (8K req/s): Alert ops team
> - 95% (9.5K req/s): Start shedding low-priority traffic
> - 100% (10K req/s): Return 503 Service Unavailable
> ```
>
> **2. Per-tenant quota (enforce SLA)**
> ```
> Tier 1 (Free): 100 req/sec = 8.6M req/day
> Tier 2 (Pro): 1,000 req/sec = 86M req/day
> Tier 3 (Enterprise): 10,000 req/sec = 864M req/day (custom)
> 
> If tenant exceeds quota:
> - Return 429 Too Many Requests
> - Tell them: 'You've exceeded your quota. Upgrade or wait.'
> ```
>
> **3. Per-user limit (prevent single user from dominating)**
> ```
> If tenant has 100 users, they all share 100 req/sec quota
>
> User rate limiting prevents:
> - One user hogging all bandwidth
> - Runaway scripts/loops
> - Accidental DDOS from buggy code
> ```
>
> **4. Per-endpoint limits (protect sensitive endpoints)**
> ```
> Endpoint: /api/authenticate
> Limit: 10 req/min (prevent brute force)
>
> Endpoint: /api/list-documents
> Limit: 100 req/min (normal browsing)
>
> Endpoint: /api/expensive-operation
> Limit: 1 req/sec (expensive resource)
> ```
>
> **Implementation (Python):**
> ```python
> from redis import Redis
> from datetime import datetime, timedelta
> 
> class RateLimiter:
>     def __init__(self, redis_url):
>         self.redis = Redis.from_url(redis_url)
>     
>     # Token bucket algorithm (industry standard)
>     def is_allowed(self, tenant_id, user_id, limit_per_sec, endpoint):
>         key = f\"rate_limit:{tenant_id}:{user_id}:{endpoint}\"
>         
>         # Get current bucket state
>         bucket = self.redis.hgetall(key)
>         tokens = float(bucket.get(b'tokens', 0))
>         last_refill = float(bucket.get(b'last_refill', time.time()))
>         
>         now = time.time()
>         time_passed = now - last_refill
>         
>         # Refill tokens (1 token per 1/limit_per_sec seconds)
>         refill_rate = limit_per_sec  # tokens per second
>         tokens_to_add = time_passed * refill_rate
>         tokens = min(tokens + tokens_to_add, limit_per_sec)  # Cap at limit
>         
>         if tokens >= 1:
>             # Allow request, consume 1 token
>             tokens -= 1
>             self.redis.hset(
>                 key,
>                 mapping={'tokens': tokens, 'last_refill': now},
>                 ex=3600  # Expire after 1 hour
>             )
>             return True, None
>         else:
>             # Request denied, calculate retry_after
>             tokens_needed = 1
>             time_to_next_token = (tokens_needed - tokens) / refill_rate
>             return False, int(time_to_next_token)
>     
>     # In API endpoint:
>     @app.post('/api/chat')
>     async def chat(request: ChatRequest, credentials: Credentials):
>         tenant_id = credentials.tenant_id
>         user_id = credentials.user_id
>         
>         # Check rate limit
>         allowed, retry_after = limiter.is_allowed(
>             tenant_id=tenant_id,
>             user_id=user_id,
>             limit_per_sec=get_tenant_quota(tenant_id),
>             endpoint='chat'
>         )
>         
>         if not allowed:
>             return JSONResponse(
>                 status_code=429,
>                 content={
>                     'error': 'Rate limit exceeded',
>                     'retry_after': retry_after
>                 },
>                 headers={'Retry-After': str(retry_after)}
>             )
>         
>         # Process request
>         response = await claude_api.call(request.prompt)
>         return response
> ```
>
> **API Gateway level (Nginx/Kong):**
> ```nginx
> # Rate limit per IP (prevent DDOS)
> limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
>
> # Rate limit per tenant header
> limit_req_zone $http_x_tenant_id zone=tenant:10m rate=100r/s;
>
> # Rate limit per endpoint
> limit_req_zone $binary_remote_addr zone=auth:10m rate=5r/m;
>
> server {
>     location /api/ {
>         limit_req zone=api burst=20 nodelay;
>         limit_req zone=tenant burst=200 nodelay;
>     }
>     
>     location /api/authenticate {
>         limit_req zone=auth burst=2 nodelay;
>     }
> }
> ```
>
> **Distributed rate limiting (if multi-server):**
> ```
> Single server: Use in-memory rate limiter (fast)
> Multiple servers: Must use Redis/Memcached (shared state)
> 
> Otherwise, rate limit enforced per-server:
> - 3 servers × 100 req/sec = 300 req/sec total (not 100!)
> - Defeats the purpose
> ```
>
> **Monitoring:**
> ```
> Prometheus metrics:
> - rate_limit_exceeded_total (counter, by tenant)
> - current_rate_limit_usage (gauge, by tenant)
>
> Alerts:
> - Tenant consistently hitting limit (upgrade plan?)
> - Spike in 429 errors (DDoS or broken client?)
> ```"

---

### Q2: "One tenant is complaining their requests are getting rate-limited. Investigation shows they're under quota, but still getting 429. Root cause?"

**Model Answer:**

> "Rate limiting failures usually due to:
>
> **1. Client IP blocking (not tenant ID)**
> ```
> API gateway rate limit $binary_remote_addr
> Shared IP (proxy, office, corporate firewall)?
> 
> Solution: Rate limit by tenant_id header, not IP
> ```
>
> **2. Cascading rate limits**
> ```
> API gateway: 100 req/sec
> Application layer: 100 req/sec
> Database connection pool: 50 concurrent connections
>
> Issue: Application getting 100 req/sec, but database can only handle 50
> Response: Database slow, requests queue up, eventually timeout
>
> Looks like rate limiting, but isn't!
>
> Solution: Ensure connection pool can handle burst
> ```
>
> **3. Request burst not allowed**
> ```
> Token bucket:
> - Limit: 10 req/sec average
> - Burst: 0 (no burst allowed)
>
> Client sends 20 requests at once:
> - First 10 succeed instantly
> - Next 10: Rate limited, even though under daily quota
>
> Solution: Allow burst
> ```
>
> **4. Per-user limit is stricter than per-tenant**
> ```
> Tenant quota: 100 req/sec
> Individual user limit: 10 req/sec
>
> Tenant has 20 users
> One user hammering API at 15 req/sec
> Gets rate limited by user limit (10 < 15)
> Even though tenant quota OK
>
> Solution: Check multi-level limiting
> ```
>
> **5. Rate limit key collision**
> ```
> Redis key: rate_limit:tenant_id:user_id:endpoint
>
> Bug: Key not including user_id
> Multiple users get same rate limit bucket
> 10 users × 10 req/sec = 100 req/sec, but they share one bucket
> So each user only gets 1 req/sec (100 / 10 users)
>
> Solution: Fix key generation
> ```
>
> **Debugging approach:**
> ```python
> # Simulate the request with logging
> def debug_rate_limit_issue(tenant_id, user_id):
>     key = f\"rate_limit:{tenant_id}:{user_id}:chat\"
>     bucket = redis.hgetall(key)
>     
>     print(f'Key: {key}')
>     print(f'Tokens available: {bucket.get(\"tokens\")}')
>     print(f'Last refill: {bucket.get(\"last_refill\")}')
>     print(f'Tenant quota: {get_tenant_quota(tenant_id)}')
>     print(f'User quota: {get_user_quota(tenant_id, user_id)}')
>     print(f'IP: {request.remote_addr}')
>     print(f'API gateway rate limit: ...')
>     
>     # Try request, see exactly where it fails
>     allowed, retry_after = limiter.is_allowed(...)
>     print(f'Result: {allowed}, Retry after: {retry_after}')
> ```"

---

## TOPIC 4: RESILIENCY (Fault Tolerance & High Availability)

### Q1: "Design resilient architecture for GenAI platform that can withstand component failures"

**Model Answer:**

> "Resilience means: If something breaks, system still works.
>
> **Failure modes to handle:**
> 1. Pod crashes (container OOMKill, segfault)
> 2. Node failure (hardware dies, AWS termination)
> 3. Database failure (primary down, replicas available)
> 4. External API failure (Claude API slow or down)
> 5. Network partition (service unreachable)
> 6. Resource exhaustion (CPU 100%, memory full)
> 7. Configuration error (broken config deployed)
>
> **Architecture (by component):**
>
> **API Layer (Request handling):**
> ```
> Load Balancer (external)
>   ├─ API Pod 1 (zone 1)
>   ├─ API Pod 2 (zone 1)
>   ├─ API Pod 3 (zone 2)
>   └─ API Pod 4 (zone 2)
>
> If one pod crashes:
> - Load balancer detects (health check fails)
> - Stops sending traffic to failed pod
> - Routes to other 3 pods
> - Kubernetes restarts the pod
> 
> Resilience: 1 pod failure → 75% capacity (3/4 pods working)
> ```
>
> **Liveness & Readiness Probes (health checks):**
> ```yaml
> spec:
>   containers:
>   - name: api
>     livenessProbe:
>       httpGet:
>         path: /health/live
>         port: 8080
>       initialDelaySeconds: 30
>       periodSeconds: 10
>       failureThreshold: 3
>       # If 3 consecutive failures, kill and restart
>     
>     readinessProbe:
>       httpGet:
>         path: /health/ready
>         port: 8080
>       initialDelaySeconds: 5
>       periodSeconds: 5
>       failureThreshold: 3
>       # If 3 failures, remove from load balancer (but don't restart)
> ```
>
> What to check:
> ```python
> @app.get('/health/live')
> def liveness():
>     # Is the process alive? (very basic check)
>     return {'status': 'alive'}
>
> @app.get('/health/ready')
> def readiness():
>     # Can I handle requests?
>     checks = {
>         'database': db.ping(),
>         'cache': redis.ping(),
>         'api_quota': api_quota_remaining() > 0,
>         'memory': psutil.virtual_memory().percent < 90,
>     }
>     
>     all_ready = all(checks.values())
>     if all_ready:
>         return {'status': 'ready', 'checks': checks}
>     else:
>         return {'status': 'not ready', 'checks': checks}, 503
> ```
>
> **Database Layer (Data persistence):**
> ```
> PostgreSQL Primary (writes)
>   ├─ Replica 1 (reads)
>   ├─ Replica 2 (reads)
>   └─ Replica 3 (async backup)
>
> If primary fails:
> 1. Automatic failover (Patroni or similar)
> 2. Replica 1 promoted to primary
> 3. App reconnects (connection string auto-resolves to new primary)
> 4. Replica 2 and 3 continue serving reads
>
> Resilience: Primary failure → No data loss (if synchronous replication)
> ```
>
> **Backup and Recovery:**
> ```
> Daily backup:
> - Full backup at 2 AM (take snapshot)
> - Replication lag: < 1 second
> - Recovery time: 1-5 minutes (failover)
> - Recovery point: < 10 seconds (RPO)
>
> Backup location: Different region (protect against region failure)
> ```
>
> **External API Failure (Claude API):**
> ```python
> class ResilientGenAIClient:
>     def __init__(self):
>         self.circuit_breaker = CircuitBreaker(
>             failure_threshold=5,  # 5 failures → open circuit
>             timeout=60  # Reset after 60 seconds
>         )
>     
>     def call_with_resilience(self, prompt, tenant_id):
>         # Fallback chain
>         fallbacks = [
>             lambda: self._call_claude_opus(prompt),
>             lambda: self._call_claude_haiku(prompt),  # Faster, cheaper
>             lambda: self._cache_lookup(prompt),  # Try cache
>             lambda: self._queue_for_async(prompt),  # Async processing
>         ]
>         
>         for fallback_fn in fallbacks:
>             try:
>                 # Try with circuit breaker
>                 result = self.circuit_breaker.call(fallback_fn)
>                 return result
>             except CircuitBreakerOpen:
>                 continue  # Try next fallback
>             except APIError as e:
>                 if e.retriable:
>                     # Retry with exponential backoff
>                     for attempt in range(3):
>                         time.sleep(2 ** attempt)
>                         try:
>                             return fallback_fn()
>                         except APIError:
>                             continue
>             except Exception as e:
>                 continue  # Try next fallback
>         
>         # All fallbacks failed
>         raise AllFallbacksFailed('All GenAI fallbacks exhausted')
>
>     def _call_claude_opus(self, prompt):
>         return claude_api.call(prompt, model='claude-opus')
>     
>     def _call_claude_haiku(self, prompt):
>         # Faster (100ms) but lower quality
>         return claude_api.call(prompt, model='claude-haiku')
>     
>     def _cache_lookup(self, prompt):
>         # Cached response from previous similar prompt
>         cached = cache.get(hash(prompt))
>         if cached:
>             return cached
>         raise CacheNotFound()
>     
>     def _queue_for_async(self, prompt):
>         # Queue for async processing (return immediately)
>         job_id = queue.enqueue(process_prompt, prompt)
>         return {'status': 'processing', 'job_id': job_id}
> ```
>
> **Network Resilience:**
> ```yaml
> Pod Disruption Budget (don't evict all pods during maintenance):
> apiVersion: policy/v1
> kind: PodDisruptionBudget
> metadata:
>   name: api-server-pdb
> spec:
>   minAvailable: 2  # Always keep at least 2 API pods running
>   selector:
>     matchLabels:
>       app: api-server
>
> Result:
> - Kubernetes maintenance won't evict all pods
> - At least 2 API pods always running
> - Request distribution: Never 100% down
> ```
>
> **Configuration Management:**
> ```
> Bad: Hard-coded configuration
>
> Good: ConfigMap + feature flags
>
> Better: Feature flags that can change without deployment:
> - Circuit breaker threshold: Increase if many failures
> - Fallback model: Switch to Haiku if Opus slow
> - Rate limits: Increase/decrease per tenant dynamically
> ```
>
> **Monitoring for Resilience:**
> ```
> Alerts to catch failures early:
> - Pod crash rate > 0.1%
> - Restart count increasing
> - Error rate spike
> - Latency spike (indicates struggling component)
> - Failed health checks
>
> Dashboard: \"Resilience Status\"
> - Pod health: 48/50 healthy (96%)
> - Database health: Primary + replicas healthy
> - External API: 99.9% availability
> - Failover time: Last failover 5 days ago
> ```"

---

### Q2: "During peak traffic, your service starts timing out. Users get 504 errors. How do you keep it resilient?"

**Model Answer:**

> "504 Gateway Timeout = upstream service too slow or not responding.
>
> **Immediate actions (next 2 minutes):**
> 1. Check alert dashboard - where is latency spike?
> 2. Scale up (KEDA trigger or manual kubectl scale)
> 3. Check if external API (Claude) is down
>
> **Resilience improvements:**
> ```
> Problem: API call to Claude takes 5 seconds, timeout is 3 seconds
> Result: All requests timeout
>
> Solution 1: Increase timeout
> - Change timeout from 3s → 10s
> - But doesn't solve underlying slowness
> - Users wait longer, might timeout at 10s too
>
> Solution 2: Async processing (better)
> - Return immediately with job ID: \"Your request is processing\"
> - Process in background
> - User polls status endpoint to get result
> - Result: No timeouts, user perceives faster service
>
> Solution 3: Fallback to cached result
> - If Claude slow, return previous similar response
> - Slightly less accurate, but no timeout
> - Mark response: \"This is a cached result\"
>
> Solution 4: Circuit breaker
> - If Claude API is slow (p95 > 2s), stop calling it
> - Use fallback immediately
> - No waiting for timeout
> ```
>
> **Implementation (async processing):**
> ```python
> # Endpoint that returns immediately
> @app.post('/api/chat')
> async def chat_async(request: ChatRequest):
>     # Queue the request
>     job = Job.create(
>         type='chat',
>         tenant_id=request.tenant_id,
>         prompt=request.prompt,
>         status='queued'
>     )
>     
>     # Return immediately with job ID
>     return {
>         'job_id': job.id,
>         'status': 'queued',
>         'message': 'Your request is processing. Check status at /api/jobs/{job_id}'
>     }
>
> # Background worker processes
> @app.post('/api/jobs/{job_id}/status')
> async def check_status(job_id: str):
>     job = Job.get(job_id)
>     if job.status == 'done':
>         return {'status': 'done', 'result': job.result}
>     elif job.status == 'error':
>         return {'status': 'error', 'error': job.error}, 500
>     else:
>         return {'status': job.status}
> ```
>
> **Load shedding (when overwhelmed):**
> ```python
> # If system is overloaded, reject lowest-priority requests
> class LoadSheddingMiddleware:
>     def __call__(self, request):
>         # Current load (0-100%)
>         load = get_system_load()
>         
>         # Priority (from JWT or tenant tier)
>         priority = request.get_priority()
>         
>         # Reject some requests based on load + priority
>         if load > 80:
>             # Overloaded
>             if priority < 5:
>                 # Low priority, reject
>                 return JSONResponse(
>                     status_code=503,
>                     content={
>                         'error': 'Service overloaded, try again in 30 seconds',
>                         'retry_after': 30
>                    },
>                    headers={'Retry-After': '30'}
>                 )
>         
>         # High priority or low load, allow request
>         return request
> ```
>
> **Resilience summary:**
> - Multi-replica deployment (if 1 fails, 3 others handle)
> - Health checks (catch failures, restart pods)
> - Async processing (avoid timeouts)
> - Circuit breakers (fail fast, don't wait)
> - Load shedding (drop lowest-priority requests when overloaded)
> - Proper timeout (not too short, not too long)
> ```"

---

## TOPIC 5: ROLLBACK STRATEGIES

### Q1: "Design comprehensive rollback strategy for multi-tenant platform with zero downtime requirement"

**Model Answer:**

> "Rollback needs to be automatic AND manual, fast AND safe.
>
> **Rollback triggers (automatic):**
> ```
> Monitor these metrics during deployment:
> 
> 1. Error rate:
>    - If error rate > 1% (was 0.1%), rollback
>    - Example: 1000 req/sec × 1% = 10 errors/sec = TOO MANY
>
> 2. Latency:
>    - If P95 latency > 1s (was 200ms), rollback
>    - Example: Users perceive 5x slower = bad
>
> 3. Pod crashes:
>    - If pod restart rate > 10%, rollback
>    - Example: OOMKill, segfault, panic loop
>
> 4. Traffic drops:
>    - If throughput < 50% expected, rollback
>    - Example: API broken, client traffic drops
> ```
>
> **Blue-Green Deployment (safest for critical services):**
> ```yaml
> # Current state: Blue (v1.0) handling all traffic
> Blue (v1.0):
> ├─ API Pod 1 (v1.0)
> ├─ API Pod 2 (v1.0)
> └─ API Pod 3 (v1.0)
>
> Green (v1.1) - NEW VERSION:
> ├─ API Pod 1 (v1.1)
> ├─ API Pod 2 (v1.1)
> └─ API Pod 3 (v1.1)
>
> Deploy flow:
> 1. Deploy v1.1 to Green (no traffic yet)
> 2. Run integration tests on Green
> 3. Run load tests on Green
> 4. When satisfied, switch load balancer: Blue → Green
> 5. Monitor Green for 5 minutes
> 6. If all good, decommission Blue
> 7. If problems, switch back: Green → Blue (instant rollback)
>
> Benefits:
> - Instant rollback (just switch load balancer)
> - Time to test: Limited only by how thorough you want
> - Zero downtime: Both versions running simultaneously
> - Full rollback: Old version still running
> ```
>
> **Implementation:**
> ```yaml
> # v1.0 (current, Blue)
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: api-server-blue
> spec:
>   replicas: 3
>   selector:
>     matchLabels:
>       app: api-server
>       version: v1.0
>       color: blue
>   template:
>     metadata:
>       labels:
>         app: api-server
>         version: v1.0
>         color: blue
>     spec:
>       containers:
>       - name: api
>         image: api-server:v1.0
>
> # v1.1 (new, Green)
> ---
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: api-server-green
> spec:
>   replicas: 3
>   selector:
>     matchLabels:
>       app: api-server
>       version: v1.1
>       color: green
>   template:
>     metadata:
>       labels:
>         app: api-server
>         version: v1.1
>         color: green
>     spec:
>       containers:
>       - name: api
>         image: api-server:v1.1
>
> # Service routes to Blue (currently)
> ---
> apiVersion: v1
> kind: Service
> metadata:
>   name: api-server
> spec:
>   selector:
>     app: api-server
>     color: blue  # Points to Blue right now
>   ports:
>   - port: 80
>     targetPort: 8080
>
> # To switch to Green:
> # kubectl patch service api-server -p '{\"spec\":{\"selector\":{\"color\":\"green\"}}}'
> 
> # To rollback to Blue:
> # kubectl patch service api-server -p '{\"spec\":{\"selector\":{\"color\":\"blue\"}}}'
> ```
>
> **Automated rollback:**
> ```python
> # Monitor during deployment, auto-rollback if metrics bad
>
> class DeploymentMonitor:
>     def monitor_after_switch(self, new_version, duration_seconds=300):
>         \"\"\"Monitor for 5 minutes after switching, rollback if needed\"\"\"
>         start = time.time()
>         
>         while time.time() - start < duration_seconds:
>             metrics = get_metrics(version=new_version)
>             
>             if self.check_health(metrics):
>                 logger.info(f'✅ {new_version} healthy after deployment')
>                 return True
>             
>             time.sleep(10)
>         
>         # Duration expired, still not healthy
>         logger.error(f'❌ {new_version} not healthy, rolling back')
>         self.rollback(previous_version)
>         return False
>     
>     def check_health(self, metrics):
>         \"\"\"All metrics must be within acceptable range\"\"\"
>         checks = {
>             'error_rate': metrics['error_rate'] < 0.01,  # < 1%
>             'latency_p95': metrics['latency_p95'] < 1000,  # < 1s
>             'pod_restart_rate': metrics['pod_restart_rate'] < 0.1,  # < 10%
>             'uptime': metrics['uptime'] > 60,  # Been up > 60 seconds
>         }
>         
>         return all(checks.values())
>     
>     def rollback(self, previous_version):
>         \"\"\"Immediate rollback to previous version\"\"\"
>         service = k8s.get_service('api-server')
>         service.selector['version'] = previous_version
>         k8s.update_service(service)
>         logger.info(f'Rolled back to {previous_version}')
> ```
>
> **Rolling Update (alternative, gradual):**
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: api-server
> spec:
>   replicas: 10
>   strategy:
>     type: RollingUpdate
>     rollingUpdate:
>       maxSurge: 2  # Max 2 extra pods during update
>       maxUnavailable: 0  # Never go below 10 pods
>   
>   template:
>     spec:
>       containers:
>       - name: api
>         image: api-server:v1.1
> 
> # Deployment flow:
> # 1. Start with 10 pods (v1.0)
> # 2. Create 1 pod (v1.1) → 11 pods (10 old, 1 new)
> # 3. Old pod dies → 10 pods (9 old, 1 new)
> # 4. Create 1 pod (v1.1) → 11 pods (8 old, 2 new)
> # 5. Continue until 10 pods (all v1.1)
> # 6. If problem detected at step 4, rollback by reversing process
>
> Rollback command:
> kubectl rollout undo deployment/api-server
> ```
>
> **For multi-tenant (tenant-specific rollback):**
> ```python
> # Some tenants might have feature flags for new version
> # Others stay on old version
>
> @app.get('/api/data')
> def get_data(request: Request):
>     tenant_id = request.get_tenant_id()
>     version = get_feature_flag(tenant_id, 'api_version')
>     
>     if version == 'v1.1':
>         return new_implementation()
>     else:
>         return old_implementation()
>
> # If v1.1 has bugs only for Tenant A:
> # Set feature flag: Tenant A → v1.0 (only for them)
> # Other tenants stay on v1.1
>
> # Result: Partial rollback (only affected tenant reverts)
> ```
>
> **Deployment checklist:**
> ```
> Before deployment:
> ☐ Code reviewed and approved
> ☐ Automated tests passing
> ☐ Load tests passing (expected latency/throughput)
> ☐ Staging deployment successful
> ☐ On-call engineer available
> ☐ Rollback plan documented
>
> During deployment:
> ☐ Deploy to non-prod first (staging)
> ☐ Run integration tests
> ☐ Deploy to prod (Blue-Green)
> ☐ Monitor metrics for 5 min
> ☐ Switch traffic if all good
> ☐ Monitor for 10 min
> ☐ Decommission old version
>
> After deployment:
> ☐ Verify logs for errors
> ☐ Customer feedback (any issues?)
> ☐ Document what changed
> ☐ Update runbooks if deployment process changed
> ```"

---

## TOPIC 6: OBSERVABILITY DESIGN & ALERT MANAGEMENT

### Q1: "Design observability system for multi-tenant platform to enable fast troubleshooting"

**Model Answer:**

> "Observability = See what's happening in production at any time.
>
> **Three pillars (logs + metrics + traces):**
>
> **1. Logs (What happened?)**
> ```
> Level 1 (Human readable):
> 2024-04-23 14:30:15 INFO api_server.py:123
> \"POST /api/chat from tenant=tenant-a user=user-123 latency=250ms\"
>
> Level 2 (Structured, searchable):
> {
>   \"timestamp\": \"2024-04-23T14:30:15Z\",
>   \"level\": \"INFO\",
>   \"service\": \"api_server\",
>   \"tenant_id\": \"tenant-a\",
>   \"user_id\": \"user-123\",
>   \"endpoint\": \"/api/chat\",
>   \"method\": \"POST\",
>   \"status_code\": 200,
>   \"latency_ms\": 250,
>   \"error\": null,
>   \"trace_id\": \"abc123def456\"  # Links to trace
> }
>
> Storage: OpenSearch (logs searchable by any field)
> Retention: 30 days (compliance), older logs to S3
> Query: OpenSearch query language (similar to SQL)
> ```
>
> **2. Metrics (How much and how many?)**
> ```
> Counter (increases only):
> - requests_total: 1,000,000
> - errors_total: 1000
> - genai_tokens_total: 50,000,000
>
> Gauge (can go up/down):
> - active_requests: 250
> - pod_memory_bytes: 512MB
> - queue_depth: 1000
>
> Histogram (distribution):
> - request_latency_ms: [100, 150, 200, 300, 500, 1000, 2000, 5000]
>
> Storage: Prometheus
> Query language: PromQL
> Retention: 15 days (more recent) + archival to long-term storage
> ```
>
> **3. Traces (How did request flow through system?)**
> ```
> Single request flow:
>
> Request arrives: API Gateway
>   → Authorization (Keycloak): 50ms
>   → Rate Limit check: 5ms
>   → Call GenAI (Claude API): 500ms
>     → Token counting: 10ms
>     → API call: 450ms
>     → Response parsing: 40ms
>   → Store in database: 100ms
>   → Return response: 5ms
>
> Total: 660ms (you can see where time spent!)
>
> If slow: Drill down to see which span is slow
>
> Storage: Jaeger or Tempo
> Query: Trace ID (e.g., \"abc123def456\")
> ```
>
> **Unified logging stack:**
> ```
> Application (Python/Java/Go)
>   ↓ (logs + metrics + traces)
> OpenTelemetry Collector (unified collection)
>   ├→ Logs: Fluent Bit → OpenSearch
>   ├→ Metrics: Prometheus
>   └→ Traces: Jaeger
>
> Frontend (Grafana):
>   - Explore logs by any field
>   - View metrics graphs
>   - Follow traces
>   - Correlate all three
> ```
>
> **Implementation (Python example):**
> ```python
> from opentelemetry import trace, metrics, logging as otel_logging
> from opentelemetry.exporter.jaeger.thrift import JaegerExporter
> from opentelemetry.exporter.prometheus import PrometheusMetricReader
> import logging
> import structlog
>
> # Setup structured logging
> structlog.configure(
>     processors=[
>         structlog.processors.JSONRenderer()
>     ]
> )
> logger = structlog.get_logger()
>
> # Setup tracing
> jaeger_exporter = JaegerExporter(
>     agent_host_name=\"localhost\",
>     agent_port=6831,
> )
> trace.set_tracer_provider(...)
> tracer = trace.get_tracer(__name__)
>
> # Setup metrics
> meter = metrics.get_meter(__name__)
> requests_counter = meter.create_counter(
>     \"requests_total\",
>     description=\"Total API requests\"
> )
>
> # In API endpoint:
> @app.post(\"/api/chat\")
> async def chat(request: ChatRequest):
>     with tracer.start_as_current_span(\"chat_endpoint\") as span:
>         span.set_attribute(\"tenant_id\", request.tenant_id)
>         span.set_attribute(\"user_id\", request.user_id)
>         
>         try:
>             # Log structured data
>             logger.info(
>                 \"chat_request_received\",
>                 tenant_id=request.tenant_id,
>                 user_id=request.user_id,
>                 prompt_length=len(request.prompt),
>                 trace_id=span.context.trace_id
>             )
>             
>             # Call GenAI with tracing
>             with tracer.start_as_current_span(\"genai_api_call\") as genai_span:
>                 response = await claude_api.call(request.prompt)
>             
>             # Store in database
>             with tracer.start_as_current_span(\"database_write\") as db_span:
>                 db.store_result(...)
>             
>             # Increment metrics
>             requests_counter.add(1, {\"status\": \"success\"})
>             
>             logger.info(
>                 \"chat_request_completed\",
>                 tenant_id=request.tenant_id,
>                 status=\"success\",
>                 trace_id=span.context.trace_id
>             )
>             
>             return response
>         
>         except Exception as e:
>             requests_counter.add(1, {\"status\": \"error\"})
>             logger.error(
>                 \"chat_request_failed\",
>                 tenant_id=request.tenant_id,
>                 error=str(e),
>                 trace_id=span.context.trace_id
>             )
>             raise
> ```
>
> **Grafana dashboards (for different roles):**
> ```
> Dashboard 1: \"Ops Health\" (for on-call)
> ├─ Error rate (red if > 0.5%)
> ├─ Latency P95 (red if > 1s)
> ├─ Pod health (how many healthy?)
> ├─ Database health (replication lag)
> └─ Alert status (which are firing?)
>
> Dashboard 2: \"Performance\" (for engineers)
> ├─ Request latency by endpoint
> ├─ Throughput by tenant
> ├─ Error rate by service
> └─ Resource usage by pod
>
> Dashboard 3: \"GenAI Metrics\" (for product)
> ├─ Token usage by tenant
> ├─ GenAI cost per request
> ├─ Queue depth
> └─ Processing time
>
> Dashboard 4: \"Customer View\" (for tenant/support)
> ├─ Your requests: 10K today
> ├─ Your errors: 5 (0.05%)
> ├─ Your latency: 200ms (good)
> └─ Tokens used: 500K (budget: 1M)
> ```"

---

### Q2: "Alert fatigue: You're getting 100 alerts per day. 95% are false positives. How do you fix?"

**Model Answer:**

> "Alert fatigue kills on-call. You ignore alerts if too many.
>
> **Problem analysis:**
> ```
> 100 alerts/day = 1 alert every 14 minutes
> 95% false positive = 95 noisy alerts/day
> Result: On-call engineer ignores all alerts
> Real incident happens: Ignored!
> ```
>
> **Root causes of false positives:**
>
> **1. Thresholds too sensitive**
> ```
> Alert: CPU > 50%
> Triggers constantly on normal load spike
> Not useful
>
> Solution: Raise threshold
> Alert: CPU > 80% AND duration > 5 min AND error_rate > 1%
> (More specific = fewer false positives)
> ```
>
> **2. Alerting on intermediate states**
> ```
> Alert: Pod restarting
> Kubernetes autoheal: Pod crashes, restarts, comes back up
> Alert fires even though system recovers automatically
>
> Solution: Only alert if doesn't recover in time
> Alert: Pod restarted 3+ times in 5 min (indicates loop)
> ```
>
> **3. Maintenance windows not respected**
> ```
> Alert: Database replica lag > 1s
> During maintenance: Database upgrade, expected lag
> Alert fires every 10 seconds
>
> Solution: Silence alerts during maintenance window
> Or: Alert only if replica lag > 10s (more tolerance during maint)
> ```
>
> **4. Cascading alerts**
> ```
> Database primary down
> ├─ Alert: \"Database unreachable\"
> ├─ Alert: \"API errors high\" (because no DB)
> ├─ Alert: \"Pod restarts increasing\" (apps crashing)
> ├─ Alert: \"CPU spike\" (kernel OOM killer)
> └─ Alert: \"Memory leak detected\" (actually out of memory)
>
> 5 alerts fired from 1 problem!
>
> Solution: Group related alerts
> Only alert on root cause (database unreachable)
> Suppress dependent alerts for 5 min
> ```
>
> **5. Alerting on symptoms, not problems**
> ```
> Alert: High latency
> This is a symptom. Root cause could be:
> - Database slow (check slow query log)
> - CPU maxed (check CPU)
> - Memory full (check memory)
> - Network congestion (check network)
>
> Solution: Alert on root causes
> Alert: \"Slow database queries detected (query_time > 500ms)\"
> Alert: \"CPU >80% for 5 min\"
> Alert: \"Memory >85% for 5 min\"
> ```
>
> **Implementation (better alerting):**
> ```yaml
> # Prometheus alert rules
>
> # Bad alert (too sensitive, false positives)
> - alert: HighCPU
>   expr: rate(container_cpu_usage_seconds[1m]) > 0.5
>   annotations:
>     summary: \"Pod CPU high\"
>
> # Better alert (specific, respects grace period)
> - alert: PodCrashLoop
>   expr: increase(kube_pod_container_status_restarts_total[30m]) >= 3
>   annotations:
>     summary: \"Pod restarting {{ $value }} times in 30 min\"
>     description: \"Pod {{ $labels.pod }} in {{ $labels.namespace }} crashing\"
>     runbook: \"https://wiki.company.com/pod-crash-loop\"
>
> # Grouped alerts (suppress dependent ones)
> - alert: DatabaseDown
>   expr: up{job=\"database\"} == 0
>   annotations:
>     summary: \"Database is down\"
>   for: 2m  # Wait 2 min before alerting (might be transient)
>
> # Suppress alerts that depend on database being up
> - alert: HighAPIErrorRate
>   expr: rate(api_errors_total[5m]) > 0.01
>   # Silence this alert if database is down (it's expected)
>   rules:
>   - exclude: up{job=\"database\"} == 0
> ```
>
> **Alerting best practices:**
> ```
> Rule 1: Only alert if actionable
> ├─ Good: \"Pod OOMKilled, scale memory\"
> ├─ Bad: \"Memory usage 50%\" (so what?)
>
> Rule 2: Include runbook in alert
> ├─ Alert message should say what to do
> ├─ Or link to runbook: /wiki/pod-crash-loop
>
> Rule 3: Route to right person
> ├─ Database alert → DBA on-call
> ├─ API alert → Backend on-call
> ├─ GenAI alert → Platform on-call
> ├─ Don't alert everyone
>
> Rule 4: Severity levels
> ├─ P1 (critical): Wake up on-call person
> ├─ P2 (warning): Create ticket, not urgent
> ├─ P3 (info): Log it, review in standup
>
> Rule 5: Grace period
> ├─ Don't alert on 1-off failures
> ├─ Alert if: Problem persists > 5 minutes
> ├─ Example: \"Alert if CPU > 80% for 5 min\"
> ```
>
> **Metrics to improve alerting:**
> ```
> Track for each alert:
> - How many times fired? (if < 1/month, maybe not needed)
> - How many true positives? (target > 80%)
> - Average resolution time (should be < 5 min for P1)
> - False positive rate (goal < 20%)
>
> Review quarterly:
> - Disable alerts with < 80% true positive rate
> - Adjust thresholds based on false positive rate
> - Consolidate related alerts
> - Add/remove alerts based on production issues
> ```"

---

## TOPIC 7: EBPF (Extended Berkeley Packet Filter) - PoC

### Q1: "What is eBPF and how would you use it for platform observability?"

**Model Answer:**

> "eBPF = Run small programs inside Linux kernel. Observe everything without modifying application code.
>
> **Traditional monitoring (application level):**
> ```
> Application (Python)
>   ↓ (log a message)
> Logging system (writes to disk/network)
>   ↓
> Observability backend (OpenSearch, Prometheus)
>
> Overhead: Application has to call logging function
> Latency: Extra function calls slow down app
> Missing data: What if application doesn't log it?
> ```
>
> **eBPF monitoring (kernel level):**
> ```
> Linux Kernel:
> ├─ System call: open()
> ├─ Network packet received
> ├─ Function called: malloc()
> ├─ Disk I/O complete
> └─ Process scheduled
>
> eBPF programs (in kernel):
> ├─ Track every syscall
> ├─ Capture network packets
> ├─ Track memory allocation
> ├─ Track I/O latency
> └─ Track CPU time
>
> Benefits: See EVERYTHING, 0 application changes needed, kernel efficiency
> ```
>
> **Real-world use cases:**
>
> **1. Network observability (without touching app)**
> ```
> Traditional: Add logging to app
> eBPF: Intercept all network packets at kernel level
>
> You can see:
> - Every TCP connection (source, destination, port)
> - Network latency (packet timing)
> - Packet loss (retransmissions)
> - DNS queries
> - TLS handshakes
>
> All without modifying application!
> 
> Tools: Cilium, Falco, Tetragon
> ```
>
> **2. System call tracing (for debugging)**
> ```
> App is slow, but why?
> eBPF can trace every syscall:
> - open() / close()
> - read() / write()
> - mmap() / brk()
> - futex() (locks)
> - epoll() (I/O multiplexing)
>
> Example output:
> write(fd=3, buf=0x123abc, size=1024) → 1000ms (SLOW!)
> (shows disk I/O is slow, not app logic)
> ```
>
> **3. Container/Pod network policy enforcement**
> ```
> NetworkPolicy (Kubernetes):
> \"Pod A can't talk to Pod B\"
>
> Enforcement:
> - iptables: Slow, complex rules
> - eBPF: Fast kernel-level filtering
>
> Cilium (eBPF-powered network policy):
> - Transparent traffic filtering
> - No proxy overhead
> - Full visibility into traffic
> ```
>
> **4. Real-time threat detection (Falco)**
> ```
> Detect suspicious activity in real-time:
> - Process opening sensitive file (/etc/passwd)
> - Shell spawned in container (shouldn't happen)
> - Unexpected network connection (cryptominer?)
> - Privilege escalation attempt
>
> All caught at kernel level before damage done
> ```
>
> **eBPF PoC (Proof of Concept) for your platform:**
>
> **Goal: Understand actual request latency breakdown**
>
> Currently you see: \"API request took 500ms\"
> But where? Breakdown:
> - Kernel scheduling: 50ms
> - Code execution: 200ms
> - System calls (I/O): 150ms
> - Database query: 100ms
>
> eBPF PoC would show this breakdown automatically.
>
> **Implementation:**
> ```c
> // eBPF program (in kernel, compiled to bytecode)
> // Runs whenever a system call is made
>
> #include <uapi/linux/ptrace.h>
> #include <linux/sched.h>
>
> // Map to store latency buckets
> BPF_HASH(start_times, u64, u64);
> BPF_HISTOGRAM(lat_hists, u64);
>
> // Trace system call entry
> TRACEPOINT_PROBE(raw_syscalls, sys_enter) {
>   u64 pid_tgid = bpf_get_current_pid_tgid();
>   u64 timestamp = bpf_ktime_get_ns();
>   start_times.update(&pid_tgid, &timestamp);
>   return 0;
> }
>
> // Trace system call exit
> TRACEPOINT_PROBE(raw_syscalls, sys_exit) {
>   u64 pid_tgid = bpf_get_current_pid_tgid();
>   u64 *start_time = start_times.lookup(&pid_tgid);
>   
>   if (start_time) {
>     u64 delta = bpf_ktime_get_ns() - *start_time;
>     lat_hists.increment(bpf_log2l(delta));
>     start_times.delete(&pid_tgid);
>   }
>   return 0;
> }
> ```
>
> **Tools for eBPF:**
>
> **BCC (BPF Compiler Collection):**
> ```python
> from bcc import BPF
>
> program = \"\"\"
> int trace_syscall_enter(struct tracepoint__raw_syscalls__sys_enter *ctx) {
>   bpf_printk(\"syscall: %ld\", ctx->id);
>   return 0;
> }
> \"\"\"
>
> b = BPF(text=program)
> b.attach_tracepoint(tp=\"raw_syscalls:sys_enter\", fn_name=\"trace_syscall_enter\")
> b.trace_print()
> ```
>
> **bpftrace (high-level):**
> ```
> // Trace HTTP request latency
> tracepoint:syscalls:sys_enter_write {\n   @latency[execname] = hist(arg3);
> }
>
> // Show distribution of write() system call sizes by application
> ```
>
> **For your platform (PoC goals):**
> 1. **Understand actual latency** (what causes slowness?)
> 2. **Capture network flows** (which tenants talk to which services?)
> 3. **Detect anomalies** (sudden traffic spike from one tenant?)
> 4. **Security** (any unexpected network activity?)
>
> **Limitations:**
> - eBPF kernel programs are tricky to write
> - Limited kernel context available
> - Can't access full application state
> - Requires kernel >= 4.8 (and BPF features vary by version)
>
> **Better approach for you:**
> Use existing tools instead of writing eBPF:
> - Cilium (network observability)
> - Falco (security monitoring)
> - tetragon (runtime security)
> - But understand eBPF under the hood!
> ```"

---

## TOPIC 8: COMPREHENSIVE ALERT MANAGEMENT

### Q1: "Design alert routing system for multi-team platform"

**Model Answer:**

> "Alert routing = Right alert → Right team → Right time
>
> **Alert types:**
> ```
> P1 (Critical): Wake someone up
> - Database completely down
> - API serving 50%+ errors
> - Platform entirely unreachable
>
> P2 (Warning): Create ticket
> - Disk usage > 80%
> - Error rate > 1%
> - Latency P95 > 2s
>
> P3 (Info): Log it
> - Pod restart
> - Minor latency increase
> - Non-critical service unhealthy
> ```
>
> **Teams:**
> ```
> Platform Team: K8s, infrastructure, general
> Database Team: PostgreSQL, replication, backups
> GenAI Team: Claude API, token limits, models
> Security Team: Intrusions, policy violations
> On-Call (rotation): Everyone on rotation for P1
> ```
>
> **Routing logic:**
> ```
> IF alert is database-related:
>   → Database team
> ELSE IF alert is network-related:
>   → Platform team
> ELSE IF alert is GenAI-related:
>   → GenAI team
> ELSE IF alert is security-related:
>   → Security team
> ELSE:
>   → On-call engineer
>
> AND:
> IF alert is P1:
>   → Page (call phone)
> ELSE IF alert is P2:
>   → Slack + PagerDuty ticket
> ELSE IF alert is P3:
>   → Slack only
> ```
>
> **Implementation (Prometheus AlertManager):**
> ```yaml
> # Alert routing rules
> global:
>   resolve_timeout: 5m  # Auto-resolve after 5m if no updates
>
> route:
>   # Default route
>   receiver: 'default'
>   group_by: ['alertname', 'cluster', 'service']
>   group_wait: 10s  # Wait 10s for more alerts before sending
>   group_interval: 10m  # Re-send every 10m if not resolved
>   repeat_interval: 4h  # Remind every 4h if still not resolved
>   
>   # Sub-routes (specific teams)
>   routes:
>   # Database alerts
>   - match:
>       alertname: 'Database.*'
>     receiver: 'database-team'
>     continue: true  # Also send to default
>   
>   # GenAI alerts
>   - match:
>       service: 'genai'
>     receiver: 'genai-team'
>   
>   # Critical alerts (always page on-call)
>   - match:
>       severity: 'critical'
>     receiver: 'oncall-pagerduty'
>     group_wait: 0  # Don't wait, send immediately
>
> # Receivers (where to send)
> receivers:
> - name: 'default'
>   slack_configs:
>   - channel: '#alerts'
>     title: '{{ .GroupLabels.alertname }}'
>     text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
>
> - name: 'database-team'
>   slack_configs:
>   - channel: '#database-alerts'
>   email_configs:
>   - to: 'dba-team@company.com'
>
> - name: 'genai-team'
>   slack_configs:
>   - channel: '#genai-alerts'
>   webhook_configs:
>   - url: 'http://genai-service/alerts'
>
> - name: 'oncall-pagerduty'
>   pagerduty_configs:
>   - service_key: 'YOUR_PAGERDUTY_KEY'
>     description: '{{ .GroupLabels.alertname }}: {{ .Alerts[0].Annotations.description }}'
> ```
>
> **Alert lifecycle:**
> ```
> 1. Alert fires (Prometheus detects condition)
> 2. AlertManager groups alert (combines similar alerts)
> 3. AlertManager routes (sends to appropriate receiver)
> 4. Notification sent (Slack, PagerDuty, email)
> 5. Engineer acknowledged (in PagerDuty)
> 6. Engineer investigates (checks logs, metrics)
> 7. Fix deployed
> 8. Alert resolves (condition no longer true)
> 9. AlertManager sends \"resolved\" notification
> 10. Incident closed
> ```
>
> **Alert quality metrics:**
> ```
> Track for each alert type:
> - Firing time: How long before alert fires (latency)
> - True positive rate: % of alerts that indicated real problems
> - Mean time to acknowledge (MTTA): How long before engineer responds
> - Mean time to resolve (MTTR): How long to fix problem
>
> Dashboard: \"Alert Quality\"
> - Alert: \"Pod OOMKilled\"
>   - TPR: 98% (good, almost all real problems)
>   - MTTA: 3 min (good)
>   - MTTR: 15 min (good)
>   - Action: Keep as is
>
> - Alert: \"High CPU\"
>   - TPR: 40% (bad, lots of false positives)
>   - Action: Adjust threshold or disable
> ```"

---

## FINAL INTERVIEW TIPS FOR THESE TOPICS

**When asked about KEDA:**
- Explain difference from HPA (events vs metrics)
- Real-world example from your platform
- Talk about scaling challenges (lag time, cost)

**When asked about SPOT:**
- Show you understand cost-benefit trade-off
- Talk about handling interruptions gracefully
- Mention hybrid on-demand + spot strategy

**When asked about rate limiting:**
- Token bucket algorithm
- Multi-level limits (global, tenant, user, endpoint)
- Demo Redis-based implementation

**When asked about resiliency:**
- Health checks (liveness + readiness)
- Fallback strategies
- Circuit breakers for external APIs
- Pod disruption budgets

**When asked about rollback:**
- Blue-green deployment (safest)
- Automated metrics-based rollback
- Feature flags for tenant-specific rollback
- Always emphasize zero downtime

**When asked about observability:**
- Logs + metrics + traces (three pillars)
- Structured logging (JSON, searchable)
- Trace ID correlation
- Multi-tenant dashboards

**When asked about alert management:**
- Alert fatigue and how to combat it
- Routing to right team
- Grace periods to reduce false positives
- Severity levels and escalation

**When asked about eBPF:**
- Kernel-level observability without app changes
- Use cases: network monitoring, syscall tracing
- Tools: Cilium, Falco, tetragon
- Limitations and when to use vs alternatives

---

*Generated: April 23, 2026 | Advanced Platform Engineering Interview Questions*
