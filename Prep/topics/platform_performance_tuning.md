# PLATFORM PERFORMANCE MEASUREMENT & REAL-TIME TUNING
## Complete Guide for Multi-Tenant GenAI PaaS on Kubernetes

---

## PART 1: CORE PERFORMANCE CONCEPTS

### 1. The Performance Triangle

```
        LATENCY
         /    \
        /      \
    COST ---- THROUGHPUT
    
You can optimize 2, but the 3rd suffers (usually)
```

**Latency:** How fast does request complete? (100ms, 1s, 10s)
**Throughput:** How many requests/second? (1K req/s, 10K req/s)
**Cost:** How much money? (₹/month, $/request)

**Your platform goal:** Balance all three
- Fast (latency < 500ms for GenAI API)
- Scalable (handle 1M req/day)
- Cost-effective (₹3-5 per request, chargeable)

---

### 2. Four Dimensions of Performance

#### **Dimension 1: Latency (Speed)**
How long does request take?

**Types:**
- P50 (median) = 50% of requests faster than this
- P95 = 95% of requests faster than this
- P99 = 99% of requests faster than this
- P99.9 = 99.9% of requests faster than this (worst 1 in 1000)

**Example:**
```
GenAI API call latencies:
- P50: 200ms (median)
- P95: 800ms (bad outliers here)
- P99: 2000ms (really slow)
- P99.9: 5000ms (timeout territory)

Your SLA: P95 < 1000ms

If P95 exceeds 1000ms, you have a problem.
```

**What causes latency:**
- GenAI API call (200-500ms for Claude)
- Database query (10-100ms for MongoDB)
- Network (5-50ms between regions)
- Kubernetes scheduling (1-10ms)
- Connection overhead (5ms)

**How to fix:** See "Real-Time Tuning Examples" section

---

#### **Dimension 2: Throughput (Scale)**
How many requests can you handle per second?

**Example:**
```
Current: 100 req/sec = 8.6M req/day
Target: 1,000 req/sec = 86M req/day
Bottleneck: ? (could be CPU, memory, database, network)
```

**Common bottlenecks:**
- CPU (pod limited to 0.5 CPU)
- Memory (pod limited to 512Mi)
- Database connection pool (max 100 connections)
- Network bandwidth (1Gbps limit)
- External API rate limit (Claude API: 10K req/min)

**How to fix:**
1. Identify bottleneck (monitoring)
2. Remove it (increase CPU/memory, add replicas, connection pooling)
3. Measure again
4. Repeat until cost/benefit stops improving

---

#### **Dimension 3: Resource Utilization**
How much CPU/memory am I using?

**Healthy utilization:**
- CPU: 50-70% (room for spikes)
- Memory: 60-80% (OOMKill if > 100%)
- Network: 40-60% (burst capacity needed)
- Disk I/O: 40-60% (queue time matters)

**Under-utilization:** Money wasted
- CPU 20%, Memory 20% = throw out resources, save money
- Example: 10 pods with 50% CPU each = consolidate to 5 pods

**Over-utilization:** Performance degrades
- CPU 95%, Memory 95% = no room for spikes
- One burst = timeouts

**Your goal:** 60-70% utilization (leaves room for 2-3x spike)

---

#### **Dimension 4: Cost Efficiency**
Cost per unit of work

**Metrics:**
- Cost per request: ₹0.10-0.50 depending on complexity
- Cost per GB data: ₹500-2000/month for storage
- Cost per vCPU: ₹1500-3000/month
- Cost per GB memory: ₹200-400/month

**Formula:**
```
Cost per request = (Total monthly cost) / (Total requests in month)

Example:
- 1M requests/day = 30M requests/month
- Infrastructure cost = ₹1,50,000/month
- Cost per request = ₹1,50,000 / 30M = ₹0.005 per request

For your customers:
- Charge ₹0.10 per request
- Your cost: ₹0.005
- Profit: ₹0.095 per request
```

---

## PART 2: MONITORING ARCHITECTURE

### Your Platform's Monitoring Stack

```
Applications (Kubernetes)
    ↓ (logs, metrics, traces)
Fluent Bit (collect logs)
OpenSearch (store logs)
OTEL Collector (collect metrics, traces)
Prometheus (store metrics)
    ↓
Grafana Dashboards (visualize)
Alerting (when thresholds exceeded)
```

---

### Key Metrics to Collect

#### **Application Metrics** (What your app exposes)

```python
# Using Prometheus client library

from prometheus_client import Counter, Gauge, Histogram, Summary

# Counter: Increases only (total requests)
requests_total = Counter(
    'requests_total',
    'Total API requests',
    ['method', 'endpoint', 'status_code']
)

# Gauge: Can go up/down (active connections)
active_requests = Gauge(
    'active_requests',
    'Currently processing requests',
    ['endpoint']
)

# Histogram: Distribution of latencies
request_latency = Histogram(
    'request_latency_seconds',
    'Request latency in seconds',
    ['endpoint'],
    buckets=[0.01, 0.05, 0.1, 0.5, 1.0, 5.0]  # 10ms, 50ms, ..., 5s
)

# Summary: Like histogram but also calculates quantiles
response_size = Summary(
    'response_size_bytes',
    'Response size in bytes',
    ['endpoint']
)

# In your code:
with request_latency.labels(endpoint='/api/chat').time():
    response = claude_api.call(prompt)
    requests_total.labels(
        method='POST',
        endpoint='/api/chat',
        status_code=200
    ).inc()
```

**In Prometheus query:**
```
# What's my P95 latency?
histogram_quantile(0.95, request_latency_seconds)

# How many requests per second?
rate(requests_total[1m])

# Error rate?
rate(requests_total{status_code="500"}[1m]) / rate(requests_total[1m])
```

---

#### **Infrastructure Metrics** (Kubernetes auto-collects)

**Node metrics:**
- `node_cpu_seconds_total` - CPU usage
- `node_memory_bytes` - Memory usage
- `node_disk_bytes` - Disk usage
- `node_network_bytes` - Network in/out

**Pod metrics:**
- `container_cpu_usage_seconds_total` - Per-pod CPU
- `container_memory_usage_bytes` - Per-pod memory
- `pod_network_in_bytes` - Network per pod

**Kubernetes metrics:**
- `kube_pod_status_phase` - Pod state (Running, Pending, Failed)
- `kube_deployment_status_replicas_ready` - Healthy replicas
- `kube_persistentvolumeclaim_status_phase` - Storage health

---

#### **Observability (from your stack)**

**Logs (OpenSearch):**
- Request/response payloads (for debugging)
- Errors, stack traces
- Tenant ID (for multi-tenancy debugging)
- Timestamp, latency, status

**Traces (OTEL):**
- Request flow through services
- Which service is slow?
- Database query timing
- External API call timing

**Metrics (Prometheus):**
- Counter (requests, errors, costs)
- Gauge (active users, queue depth)
- Histogram (latency distribution)
- Summary (percentiles)

---

## PART 3: REAL-TIME TUNING EXAMPLES

### SCENARIO 1: Latency Spike During Peak Hours

**Situation:**
```
10 AM: Traffic increases
P95 latency jumps from 200ms to 2000ms
Customer complaints: "API is slow"
```

**Detection:**
```
Grafana dashboard shows:
- P95 latency: 200ms → 2000ms (10x increase)
- CPU: 40% → 90%
- Memory: 50% → 75%
- Error rate: 0.1% → 5%
```

**Root cause analysis:**

1. **Check Prometheus:**
```
# Is it our code or external?
- Request latency WITHOUT Claude call: 50ms (normal)
- Claude API latency: 1500ms (SLOW!)
- Our code: 50ms (fine)

Conclusion: Claude API is slow, not our code
```

2. **Check Claude API status:**
```
- Visit status.anthropic.com
- Check if there's degradation
- Check rate limits (are we hitting 10K req/min limit?)
```

3. **Check our database:**
```
Prometheus query:
rate(db_query_duration_seconds_sum) / rate(db_query_duration_seconds_count)

Result: 10ms average (fine)
```

**Solution:**

```
Option 1: Cache Claude responses
- Cache query + response for 5 minutes
- If tenant asks same question twice, use cache
- Reduces Claude API calls by 40%

Option 2: Fallback to faster model
- Use Claude Haiku for simple queries
- Use Claude Opus for complex queries
- Haiku: 100ms latency, Opus: 500ms

Option 3: Timeout + queue
- If Claude API takes > 2 seconds, timeout
- Queue request for async processing
- Return: "Your request is processing, we'll notify you"

Option 4: Increase concurrency
- Add more pod replicas
- Better load distribution
- Cost increase, but improves P95
```

**Implementation (choose based on cost/benefit):**

```python
# Monitoring-driven tuning:
if p95_latency > 1000ms:
    if claude_api_latency > 1000ms:
        # It's Claude API
        enable_caching = True
        use_fallback_haiku = True
    elif db_latency > 100ms:
        # It's database
        increase_connection_pool()
        scale_database_replicas()
    else:
        # It's our code
        profile_code()
        optimize_hot_paths()
    
    # Notify on-call engineer
    slack.notify("P95 latency spike detected")
```

**Results (after tuning):**
```
Before:
- P95: 2000ms, P99: 5000ms
- Customer SLA: VIOLATED

After caching + Haiku fallback:
- P95: 300ms, P99: 800ms
- Customer SLA: MET
- Cost: ₹5K/month saved (fewer Claude Opus calls)
```

---

### SCENARIO 2: Memory Leak in One Tenant's Workload

**Situation:**
```
Tuesday 3 PM: Tenant-A's pod crashes
Alert: Pod OOMKilled (Out of Memory)
Tenant-A: "My app was running fine yesterday"
```

**Detection:**
```
OpenSearch logs show:
- Pod tenant-a-app-12345 OOMKilled
- Memory usage: 0 MB → 512 MB (limit) in 2 hours
- No code changes in last week
```

**Investigation:**

1. **Check container memory:**
```
kubectl describe pod tenant-a-app-12345 -n tenant-a

Status: OOMKilled
Memory limit: 512Mi
Memory request: 256Mi
```

2. **Check memory trend:**
```
Prometheus query:
container_memory_usage_bytes{pod="tenant-a-app-12345"}

Result:
- 10 AM: 100 MB
- 11 AM: 150 MB
- 12 PM: 250 MB
- 1 PM: 400 MB
- 2 PM: 512 MB (CRASHED)

Pattern: Steady increase = memory leak
```

3. **Check application logs:**
```
OpenSearch query:
tenant_id: "tenant-a" AND level: "ERROR"

Results:
- "Connection pool: 100/100 (full)"
- "Cache size: 50K entries, 400MB"
- "Database connections not closing"

Root cause: Database connections not being returned to pool
```

**Solution:**

```python
# Before (buggy code):
def process_request(tenant_id):
    connection = db.get_connection()  # Get from pool
    result = connection.query("SELECT * FROM data")
    # BUG: Never close connection!
    return result

# After (fixed):
def process_request(tenant_id):
    connection = db.get_connection()
    try:
        result = connection.query("SELECT * FROM data")
        return result
    finally:
        connection.close()  # Always return to pool

# Or use context manager:
def process_request(tenant_id):
    with db.get_connection() as connection:
        result = connection.query("SELECT * FROM data")
        return result  # Auto-closes
```

**Implementation:**

1. **Fix code** (deploy new version)
2. **Increase memory limit** (temporary, while deploying)
   ```yaml
   resources:
     limits:
       memory: 1Gi  # Was 512Mi
     requests:
       memory: 512Mi
   ```
3. **Monitor memory after fix**
   ```
   Prometheus alert:
   container_memory_usage_bytes{pod="tenant-a-app-*"} > 400Mi
   -> Page on-call engineer
   ```
4. **Add automated tests**
   ```python
   # Test that connections are properly closed
   def test_connection_pool_cleanup():
       initial_connections = db.pool.active_connections
       for _ in range(100):
           process_request("test-tenant")
       final_connections = db.pool.active_connections
       assert final_connections == initial_connections  # No leak
   ```

**Results:**
```
Before:
- Pod crashes every 2 hours
- Tenant-A loses requests
- Manual restart required
- Customer satisfaction: LOW

After:
- Pod stable for days
- Memory usage: 100-150 MB (steady)
- No more OOMKills
- Customer satisfaction: HIGH
```

---

### SCENARIO 3: Database Query Slowdown Under Load

**Situation:**
```
Morning: 1000 requests/sec
- Average query: 10ms
- P95 query: 50ms
- Throughput: GOOD

Evening: 5000 requests/sec (5x increase)
- Average query: 50ms (5x slower!)
- P95 query: 500ms (10x slower!)
- Throughput: BOTTLENECK
```

**Root cause:** Database is CPU-bound

**Detection:**

```
Prometheus query:
rate(db_query_duration_seconds_sum) / rate(db_query_duration_seconds_count)

Morning (low load):
- Query latency: 10ms
- CPU: 20%, Disk I/O: 10%

Evening (high load):
- Query latency: 50ms
- CPU: 95%, Disk I/O: 40%

Bottleneck: CPU, not disk
```

**Solution:**

Option 1: Add database replicas (read scaling)
```
Before:
- Single database instance
- All reads compete for CPU
- CPU 95%

After:
- 1 primary (writes)
- 2 replicas (reads)
- Distribute reads across 3 machines
- CPU per machine: 30%
```

Option 2: Add caching layer
```
Before:
- Every request queries database
- 5000 queries/sec to database
- Database CPU: 95%

After:
- Add Redis cache
- Cache hot queries (tenant config, user data)
- 5000 req/sec, but 3000 served from cache
- 2000 queries/sec to database
- Database CPU: 40%
```

Option 3: Query optimization
```
Before query:
SELECT * FROM users WHERE tenant_id = ?
  JOIN orders ON orders.user_id = users.id
  JOIN products ON products.id = orders.product_id
(3 table join, no index on tenant_id)

After:
SELECT u.id, u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.tenant_id = ? AND o.created_at > ?
GROUP BY u.id
(indexes: tenant_id, created_at)
(remove unused product table)

Latency improvement: 50ms → 15ms
```

**Implementation (choose based on cost/benefit):**

| Solution | Cost | Time | Latency | Throughput |
|----------|------|------|---------|-----------|
| Add read replicas | ₹30K/mo | 1 day | 50ms→20ms | ↑2x |
| Add Redis cache | ₹5K/mo | 2 days | 50ms→5ms | ↑5x |
| Query optimization | ₹0 | 1 day | 50ms→15ms | ↑2x |
| All 3 combined | ₹35K/mo | 3 days | 50ms→2ms | ↑10x |

**Your choice:**
```
Cost-benefit analysis:
- Revenue from 5x throughput: ₹500K/month extra
- Cost of improvements: ₹35K/month
- Payback: < 1 month
- Decision: DO IT

Always measure revenue impact, not just technical metrics
```

**Monitoring after tuning:**

```yaml
# Prometheus alerts
alerts:
  - name: db_query_latency
    condition: db_query_latency_p95 > 100ms
    action: page_engineer

  - name: db_cpu
    condition: db_cpu_usage > 80%
    action: prepare_scaling

  - name: cache_hit_rate
    condition: cache_hit_rate < 60%
    action: investigate
```

---

### SCENARIO 4: GenAI Token Explosion (Cost Spike)

**Situation:**
```
Monday: ₹5000/day in GenAI API costs
Tuesday: ₹15000/day (3x increase!)
Wednesday: ₹25000/day (5x increase!)
Finance team: "WTF is happening?"
```

**Root cause investigation:**

```
OpenSearch query (logs):
genai_tokens_input_total - group by tenant_id, feature

Results:
- Tenant: "BigCorp"
- Feature: "document_analysis"
- Yesterday: 10K tokens
- Today: 500K tokens (50x increase!)

Tenant-A added new feature yesterday:
"Analyze all PDFs in folder"
Users uploading 100-page documents
Each page = 300 tokens
100 pages = 30K tokens
1000 users × 100 pages = 30M tokens per hour!
```

**Immediate actions:**

```python
# Real-time circuit breaker (deploy in 10 minutes)
def call_genai_api(prompt, tenant_id):
    # Check tenant's daily token budget
    tokens_today = get_tokens_used_today(tenant_id)
    tokens_budget = get_tenant_budget(tenant_id)  # e.g., 100K tokens
    tokens_in_request = estimate_tokens(prompt)
    
    if tokens_today + tokens_in_request > tokens_budget:
        # BLOCK: Would exceed budget
        raise QuotaExceededError(
            f"Daily token limit reached. "
            f"Used: {tokens_today}, Limit: {tokens_budget}"
        )
    
    # SAFE: Within budget
    response = claude_api.call(prompt)
    log_token_usage(tenant_id, tokens_in_request)
    return response
```

**Second action (1-hour fix):**

```python
# Add request caching for identical prompts
@cache(ttl=3600)  # Cache for 1 hour
def call_genai_with_cache(prompt, tenant_id):
    # If same prompt called twice = use cache
    # Reduces tokens by 80-90% for repeated requests
    return claude_api.call(prompt)

# Example:
# User A asks: "Summarize this PDF"
# Token cost: 10K tokens
# 
# User B asks: same question on same PDF
# Token cost: 0 (from cache)
# Savings: 10K tokens = ₹0.50
```

**Third action (1-day fix):**

```python
# Optimize prompts to use fewer tokens
def analyze_document_optimized(document_text):
    # BEFORE:
    # prompt = f"Analyze this document: {document_text}"
    # Tokens: 30K
    
    # AFTER (with preprocessing):
    # Extract key sections first
    key_sections = extract_key_sections(document_text)
    
    # Summarize before sending to Claude
    summary = quick_summarize(key_sections)  # Local, free
    
    # Send only summary to Claude
    prompt = f"Analyze this summary: {summary}"
    # Tokens: 3K (10x reduction!)
    
    return claude_api.call(prompt)
```

**Long-term solution (1-week fix):**

```python
# Implement token-based billing
class TenantBilling:
    def __init__(self, tenant_id, monthly_token_budget):
        self.tenant_id = tenant_id
        self.monthly_budget = monthly_token_budget  # e.g., 1M tokens
        
    def call_genai(self, prompt):
        tokens_used = estimate_tokens(prompt)
        cost = tokens_used * PRICE_PER_TOKEN  # ₹0.0005 per token
        
        # Check budget
        if cost > self.remaining_budget:
            raise BudgetExceededError(
                f"Cost: ₹{cost}, Remaining: ₹{self.remaining_budget}"
            )
        
        # Make API call
        response = claude_api.call(prompt)
        
        # Deduct from budget
        self.remaining_budget -= cost
        self.charge_customer(cost)
        
        return response
```

**Results:**

```
Before (out of control):
- Day 1: ₹5K
- Day 2: ₹15K
- Day 3: ₹25K
- Day 4: ₹40K (exponential!)
- Total: ₹85K wasted

After immediate fixes:
- Circuit breaker stops runaway requests
- Caching prevents duplicate processing
- Cost stabilizes at ₹2K/day (within budget)
- Revenue from tenant: ₹10K/day (5x ROI)
```

**Monitoring dashboard:**

```
Real-time dashboard (Grafana):
├─ Daily GenAI Cost (by feature)
│  ├─ document_analysis: ₹2000/day
│  ├─ code_generation: ₹500/day
│  └─ summarization: ₹300/day
├─ Token usage (by tenant)
│  ├─ BigCorp: 100K tokens (highest)
│  ├─ TechCorp: 50K tokens
│  └─ Others: 20K tokens
├─ Budget utilization
│  ├─ BigCorp: 100/100 (100%) - ALERT
│  ├─ TechCorp: 50/200 (25%) - healthy
│  └─ Others: 20/50 (40%) - healthy
└─ Token savings (from cache)
   └─ Saved: 500K tokens = ₹250/day
```

---

## PART 4: COMPREHENSIVE MONITORING DASHBOARD DESIGN

### Dashboard 1: Operational Health (For Ops Team)

```
Grafana Dashboard: "Platform Health"

Top row (RED ALERTS):
├─ Error rate: 0.1% (should be < 0.5%)
├─ P95 latency: 300ms (should be < 1000ms)
├─ CPU usage: 65% (should be < 80%)
├─ Memory usage: 70% (should be < 85%)

Middle row (KUBERNETES):
├─ Pod health: 48/50 healthy
├─ Node health: 3/3 healthy
├─ PVC usage: 40% full
├─ Storage growth rate: +2GB/day

Bottom row (EXTERNAL):
├─ Claude API availability: 99.9%
├─ Claude API latency: P95 = 200ms
├─ Database availability: 99.99%
├─ Database connections: 45/100 used
```

---

### Dashboard 2: Performance Analytics (For Platform Team)

```
Grafana Dashboard: "Performance Metrics"

Left side (LATENCY):
├─ Request latency distribution
│  ├─ P50: 100ms
│  ├─ P95: 300ms
│  ├─ P99: 500ms
├─ By endpoint
│  ├─ /api/chat: 200ms
│  ├─ /api/analyze: 300ms
│  ├─ /api/generate: 400ms

Right side (THROUGHPUT):
├─ Requests per second
│  ├─ Trend: ↑ (growing)
│  ├─ Current: 5K req/sec
│  ├─ Capacity: 10K req/sec
├─ By feature
│  ├─ GenAI: 3K req/sec (60%)
│  ├─ Analytics: 1.5K req/sec (30%)
│  ├─ Other: 0.5K req/sec (10%)
```

---

### Dashboard 3: Cost Analysis (For Finance Team)

```
Grafana Dashboard: "Cost Breakdown"

Total daily cost: ₹50,000

Breakdown:
├─ GenAI API: ₹25,000 (50%)
│  ├─ Claude Opus: ₹20,000
│  ├─ Claude Haiku: ₹5,000
├─ Kubernetes: ₹15,000 (30%)
│  ├─ Compute: ₹10,000
│  ├─ Storage: ₹3,000
│  ├─ Networking: ₹2,000
├─ Database: ₹8,000 (16%)
│  ├─ Primary: ₹5,000
│  ├─ Replicas: ₹3,000
├─ Other: ₹2,000 (4%)
│  ├─ Monitoring: ₹500
│  ├─ Backups: ₹1,000
│  ├─ Tools: ₹500

Cost per request: ₹50,000 / 1M req = ₹0.05
Revenue per request: ₹0.10
Profit per request: ₹0.05
Daily profit: ₹50,000
```

---

### Dashboard 4: Tenant Performance (Multi-Tenant View)

```
Grafana Dashboard: "Tenant Analytics"

Per-tenant table (sortable):
Tenant       Requests/day  Latency P95  Errors  Tokens  Cost/day
------------ ------------- ------------ ------- ------- ----------
BigCorp      500K          150ms        0.01%   100K    ₹5,000
TechCorp     300K          200ms        0.05%   50K     ₹2,500
StartupXYZ   100K          180ms        0.02%   20K     ₹1,000
SmallCo      50K           120ms        0.00%   10K     ₹500

Alert: BigCorp latency increased 2x in last hour
Action: Check if they're overloaded, scale if needed
```

---

## PART 5: PERFORMANCE TUNING CHECKLIST

### Before Any Change

```
☐ Baseline measurement
  ☐ Current latency (P50, P95, P99)
  ☐ Current throughput (req/sec)
  ☐ Current resource usage (CPU, memory)
  ☐ Current cost (₹/request)

☐ Define success criteria
  ☐ Target latency: ? ms
  ☐ Target throughput: ? req/sec
  ☐ Cost increase acceptable: ? %
```

### Quick Wins (Do First)

```
☐ Add caching
  ☐ Cache query results (5-60 min TTL)
  ☐ Cache external API responses
  ☐ Saves: 30-50% latency, 20-30% API calls

☐ Optimize database queries
  ☐ Add missing indexes
  ☐ Remove unnecessary columns
  ☐ Use LIMIT for large result sets
  ☐ Saves: 50% query latency

☐ Adjust Kubernetes requests/limits
  ☐ Set correct resource requests (not too low)
  ☐ Set reasonable limits (prevent waste)
  ☐ Saves: 20-40% wasted resources

☐ Enable connection pooling
  ☐ Database connections
  ☐ HTTP connection reuse
  ☐ Saves: 30% latency from overhead
```

### Medium Effort (Worth It)

```
☐ Add monitoring
  ☐ Metrics: P50, P95, P99, error rate
  ☐ Logs: Request details for debugging
  ☐ Traces: End-to-end flow analysis
  ☐ Saves: Hours of debugging time

☐ Load testing
  ☐ Simulate peak load
  ☐ Find breaking point
  ☐ Determine scaling strategy
  ☐ Saves: Production outages

☐ Multi-region deployment
  ☐ Reduce latency for geo-distributed users
  ☐ Redundancy (if one region fails)
  ☐ Saves: 100-500ms latency

☐ Async processing
  ☐ Queue slow operations
  ☐ Process in background
  ☐ Improves: Response time, UX
```

### Major Changes (Plan Carefully)

```
☐ Scale database (vertical or horizontal)
  ☐ Vertical: Bigger machine (more expensive, quicker)
  ☐ Horizontal: Read replicas, sharding (complex)
  ☐ Saves: 50-80% query latency

☐ Switch to faster component
  ☐ Change database (PostgreSQL → Vitess)
  ☐ Change cache (Redis vs Memcached)
  ☐ Use CDN for static content
  ☐ Saves: 30-60% latency

☐ Refactor architecture
  ☐ Microservices instead of monolith
  ☐ Async instead of sync
  ☐ Event-driven instead of request-response
  ☐ Savings: Varies wildly, high risk
```

---

## PART 6: PERFORMANCE TUNING WORKFLOW

### Step 1: Monitor (Always On)

```
Every service must have:
├─ Request latency histogram
├─ Error rate counter
├─ Resource usage gauge
└─ Business metrics (requests/day, cost)

Minimum 1 week of baseline data before optimizing
```

### Step 2: Identify Problem

```
Using monitoring, answer:
□ What is slow? (latency > target)
□ What is failing? (error rate > target)
□ What is expensive? (cost > budget)
□ What is overloaded? (resource usage > target)

Focus on biggest impact first
```

### Step 3: Root Cause Analysis

```
For each problem:
□ Is it your code? (profile, logs)
□ Is it dependency? (external API, database)
□ Is it infrastructure? (CPU, memory, network)
□ Is it load? (do 10x traffic and measure)

Verify with data, not guesses
```

### Step 4: Solution Brainstorm

```
For each root cause, list solutions:
Example (slow GenAI calls):
├─ Cache responses (low cost, moderate benefit)
├─ Use faster model (lower cost, lower quality)
├─ Timeout + queue (moderate cost, good UX)
├─ Optimize prompt (free, moderate benefit)
└─ Nothing (accept slowness)

Rank by cost/benefit ratio
```

### Step 5: Plan Change

```
Before any change:
□ Design on whiteboard
□ Estimate impact
□ Calculate cost (time + money)
□ Plan rollback strategy
□ Schedule change window (if risky)
□ Notify stakeholders
```

### Step 6: Implement & Measure

```
□ Implement in staging first
□ Measure (same metrics as prod)
□ Verify improvement
□ Deploy to prod (canary or blue-green)
□ Monitor for 1 hour
□ Compare new latency vs old
□ Calculate actual cost/benefit
```

### Step 7: Document & Repeat

```
□ Document what changed and why
□ Document the improvement achieved
□ Add alerting for new issues
□ Add monitoring for metric
□ Set new target (don't stop improving)
□ Repeat step 2
```

---

## PART 7: REAL-TIME TUNING EXAMPLE - COMPLETE WALKTHROUGH

### The Scenario

```
Your GenAI platform (Ai-Next):
- 1M requests/day
- 11 tenants
- Running on Kubernetes (5 nodes)
- Using Claude API
- Multi-tenant isolation (namespace per tenant)

Tuesday 2 PM:
Alert: "P95 latency increased from 300ms to 1500ms"
Alert: "Error rate increased from 0.1% to 2%"
Alert: "Pod CPU: 40% → 95%"

Customer complaints:
- "API is slow"
- "Getting timeouts"
- "What's wrong?"

Your on-call engineer (YOU) needs to fix this in 15 minutes.
```

### Minute 0-2: Initial Triage

**Actions:**
```
1. Open Grafana real-time dashboard
2. Check if it's platform-wide or specific tenant
3. Check if it's all endpoints or specific feature

Findings:
- Error rate spike: 2%
- Latency spike: P95 1500ms (was 300ms)
- CPU: 95% (was 40%)
- Memory: 70% (normal)
- Network: 30% (normal)

Conclusion: CPU-bound, not memory or network limited
```

**Slack alert:**
```
:warning: Production incident: API latency 5x, errors 20x
Root cause: CPU spike
Status: Investigating
Action: Scaling pods
```

### Minute 2-5: Root Cause

**Check Prometheus queries:**

```
1. Which service is CPU-heavy?

Query: sum(rate(container_cpu_usage_seconds[1m])) by (pod_label_app)

Results:
- api-server: 3000m (75% of CPU)
- genai-worker: 2000m (50% of CPU)
- database-proxy: 500m (12% of CPU)

Bottleneck: api-server pod is CPU-bound
```

```
2. What changed in api-server?

Check recent deployments:
- 2:00 PM: Deployed new GenAI integration
- 1:50 PM: Shipped batch processing feature
- 1:30 PM: Increased GenAI model calls by 3x

Suspect: New GenAI integration causing high CPU
```

```
3. What are the slow requests?

Query logs (OpenSearch):
status: "error" AND timestamp > 2PM

Results:
- 80% of errors: timeout after 30 seconds
- Feature: "document_analysis"
- Tenant: "BigCorp"

Something is making document analysis hang
```

### Minute 5-8: Confirmation

**Check GenAI-specific metrics:**

```
Query: genai_api_latency_seconds

Results:
- 2:00 PM: 200ms (normal)
- 2:05 PM: 800ms
- 2:10 PM: 2000ms
- 2:15 PM: ERROR (Claude API timeout)

Conclusion: Claude API is overloaded
                (or we're making too many calls)
```

**Check token usage:**

```
Query: genai_tokens_input_total (rate, last 15 minutes)

Results:
- Before 2 PM: 100K tokens/min
- After 2 PM: 500K tokens/min (5x!)

Root cause confirmed: New feature sending 5x tokens
```

**Specific issue:**

```
New "document_analysis" feature:
1. User uploads PDF
2. Code extracts text from PDF (200K chars)
3. Sends entire text to Claude: "Analyze this:"

Problem: 200K characters = ~50K tokens per request
If 100 users upload → 5M tokens = ₹25K in minutes!
```

### Minute 8-10: Solution Design

```
Option A (immediate, 5 min): Disable new feature
├─ Risk: Customers lose feature
├─ Benefit: System stable immediately
├─ Cost: Low
└─ Decision: Last resort

Option B (immediate, 10 min): Add circuit breaker
├─ Limit tokens per tenant per minute
├─ Block if exceeded
├─ Risk: Some requests fail, but API stable
├─ Benefit: System stable, feature still works
└─ Decision: GOOD

Option C (5 min + ongoing): Optimize prompts
├─ Extract key sections first
├─ Summarize locally before sending
├─ Reduce tokens by 80%
├─ Risk: Might lose context
├─ Benefit: Feature works, costs 80% less
└─ Decision: BEST, but takes time

Option D (30 min): Add caching
├─ Cache document analyses
├─ Don't re-analyze same PDF
├─ Benefit: Reductions 50-90%
├─ Risk: Stale results if doc updated
└─ Decision: Good medium-term
```

**Decision: Do B immediately (circuit breaker), then C (optimization)**

### Minute 10-12: Implement Circuit Breaker

**Deploy (5 minute hotfix):**

```python
# File: genai_limiter.py
# Deploy to all api-server pods immediately

class TokenLimiter:
    def __init__(self, tenant_id, max_tokens_per_min=50000):
        self.tenant_id = tenant_id
        self.max_tokens = max_tokens_per_min
        self.minute_start = now()
        self.tokens_this_minute = 0
    
    def can_call(self, estimated_tokens):
        if now() > self.minute_start + 60:
            # New minute, reset
            self.minute_start = now()
            self.tokens_this_minute = 0
        
        if self.tokens_this_minute + estimated_tokens > self.max_tokens:
            # Would exceed budget
            return False, f"Token limit: {self.tokens_this_minute}/{self.max_tokens}"
        
        return True, "OK"
    
    def call_genai(self, prompt):
        tokens = estimate_tokens(prompt)
        can_call, msg = self.can_call(tokens)
        
        if not can_call:
            raise TokenLimitExceeded(msg)
        
        response = claude_api.call(prompt)
        self.tokens_this_minute += tokens
        return response

# In main API handler:
limiter = TokenLimiter(tenant_id="BigCorp", max_tokens_per_min=50000)
try:
    response = limiter.call_genai(prompt)
except TokenLimitExceeded as e:
    return error_response(429, "Rate limited: " + str(e))
```

**Deploy command:**
```bash
# Deploy hotfix to all pods
kubectl set image deployment/api-server \
  api-server=api-server:v1.2.1-hotfix \
  -n platform

# Watch rollout (should take 2 minutes)
kubectl rollout status deployment/api-server -n platform --timeout=5m

# Check metrics immediately
watch kubectl top pods -n platform
```

### Minute 12-15: Verify Fix

**Monitor system:**

```
Grafana (real-time):
- CPU: 95% → 60% (improving)
- Error rate: 2% → 0.5% (improving)
- P95 latency: 1500ms → 500ms (improving)
- GenAI tokens: 500K → 100K tokens/min (circuit breaker working!)

Logs (OpenSearch):
- New errors: "Token limit exceeded for BigCorp"
- Status: 429 Too Many Requests
- Frequency: 50 req/sec hitting limit

Conclusion: System stable, but BigCorp can't use feature
```

**Slack update:**
```
:white_check_mark: INCIDENT RESOLVED
- Root cause: New GenAI feature sending 5x tokens
- Immediate fix: Token rate limiter deployed
- Status: System stable
- Next: Optimize feature (15 min) to remove limiter
- ETA: 2:30 PM fully resolved

BigCorp: Your feature is rate-limited while we optimize.
         Will notify when fixed. Apologize for inconvenience.
```

### Minute 15-30: Optimize Feature

**Analyze and fix:**

```python
# BEFORE (buggy):
def analyze_document(pdf_bytes, tenant_id):
    text = extract_text_from_pdf(pdf_bytes)  # 200K chars
    
    prompt = f"""
Analyze this document and provide:
1. Summary
2. Key insights
3. Action items

Document:
{text}
"""
    # 200K chars = ~50K tokens (expensive!)
    response = claude_api.call(prompt)
    return response

# AFTER (optimized):
def analyze_document(pdf_bytes, tenant_id):
    # Step 1: Extract text (free, local)
    full_text = extract_text_from_pdf(pdf_bytes)  # 200K chars
    
    # Step 2: Find key sections (free, local)
    # Don't send entire document, just key parts
    key_sections = extract_key_sections(full_text)  # 20K chars
    
    # Step 3: Summarize key sections (cheap, local)
    summary = quick_local_summarize(key_sections)  # 2K chars
    
    # Step 4: Ask Claude to analyze summary (cheap!)
    prompt = f"""
Analyze this document summary and provide:
1. Summary (already provided below)
2. Key insights
3. Action items

Summary:
{summary}

Full text available for reference if needed (but don't duplicate)
"""
    # 2K chars = ~500 tokens (10x cheaper!)
    response = claude_api.call(prompt)
    return response

# Token savings: 50K → 500 tokens per request (99% reduction!)
```

**Deploy optimization:**
```bash
# Deploy optimized version
kubectl set image deployment/api-server \
  api-server=api-server:v1.2.2-optimized \
  -n platform

# Monitor rollout
kubectl rollout status deployment/api-server -n platform

# After optimization, remove rate limiter
# (Token usage now 10x lower, circuit breaker not needed)
```

### Minute 30: Final Status

**Metrics after everything:**

```
Before incident (1 PM):
- P95 latency: 300ms
- Error rate: 0.1%
- CPU: 40%
- Tokens: 100K/min
- Cost: ₹500/hour

After circuit breaker (2:15 PM):
- P95 latency: 500ms (slightly worse, but stable)
- Error rate: 0.5% (higher, but not crashing)
- CPU: 60%
- Tokens: 100K/min (circuit breaker blocking excess)

After optimization (2:45 PM):
- P95 latency: 250ms (BETTER than before!)
- Error rate: 0.1% (back to normal)
- CPU: 35% (LOWER than before!)
- Tokens: 15K/min (10x reduction)
- Cost: ₹50/hour (10x savings!)
```

**Post-mortem:**

```
What happened:
- New feature sent entire PDF to Claude (inefficient)
- 100 customers = 5M tokens = API overload
- System crashed under load

What we did:
1. Circuit breaker (stop bleeding)
2. Optimization (permanent fix)
3. Saved 90% cost while improving performance

How to prevent:
- Add token estimation tests
- Load testing before deploy (simulate this scenario)
- Better defaults (summarize before sending to Claude)
- Cost alerts (alert if tokens jump 2x)

What we learned:
- Always profile GenAI features with realistic data
- Circuit breakers are lifesavers for external APIs
- Optimization beats scaling (cheaper, faster)
```

---

## PART 8: PERFORMANCE OPTIMIZATION CHECKLIST

```
☐ GenAI-specific
  ☐ Estimate tokens before calling API
  ☐ Cache responses for identical prompts
  ☐ Use faster models for simple tasks (Haiku vs Opus)
  ☐ Summarize before sending to Claude
  ☐ Implement circuit breakers for API calls
  ☐ Add timeout + fallback for slow responses

☐ Kubernetes-specific
  ☐ Set resource requests (enables better scheduling)
  ☐ Set resource limits (prevents runaway processes)
  ☐ Use HPA (horizontal pod autoscaler) for load spikes
  ☐ Use VPA (vertical pod autoscaler) for right-sizing
  ☐ Pod disruption budgets for high availability
  ☐ Affinity rules to spread load

☐ Database-specific
  ☐ Add indexes on frequently-queried columns
  ☐ Use connection pooling
  ☐ Cache query results
  ☐ Read replicas for scaling reads
  ☐ Partition tables for large datasets
  ☐ Monitor slow query log

☐ Monitoring-specific
  ☐ Track P50, P95, P99 latencies
  ☐ Track error rate (5xx, 4xx separately)
  ☐ Track resource utilization (CPU, memory)
  ☐ Track business metrics (requests/day, revenue)
  ☐ Set up alerting for anomalies
  ☐ Regular review of dashboards

☐ Cost-specific
  ☐ Track GenAI API costs (most expensive)
  ☐ Track infrastructure costs (Kubernetes, database)
  ☐ Cost per customer (are high-volume customers profitable?)
  ☐ Cost per feature (is new feature worth the cost?)
  ☐ Optimize (cache, cheaper models, better algorithms)
```

---

## FINAL THOUGHTS

**Performance tuning is continuous:**
- Measure baseline
- Identify problem
- Fix problem
- Measure new baseline
- Repeat

**Best order of optimizations:**
1. **Caching** (easiest, 30-50% improvement)
2. **Query optimization** (easy, 20-40% improvement)
3. **Algorithm changes** (medium, 50-80% improvement)
4. **Scaling** (expensive, 2-10x improvement)
5. **Architecture redesign** (hardest, 10x+ improvement)

**Remember:** Premature optimization is evil, but ignoring performance is worse.

**Monitor first, optimize second.**

---

*Generated: April 23, 2026 | Platform Performance Measurement & Real-Time Tuning Guide*
