# ADVANCED SYSTEM DESIGN INTERVIEW GUIDE
## For Platform Engineers with GenAI, Kubernetes & Multi-Tenant Expertise

---

## PART 1: SYSTEM DESIGN FRAMEWORK (RESHADED)

### The RESHADED Framework (8 Steps)

Used by Meta, Microsoft, and top tech companies.

```
R - Requirements (Clarify what you're building)
E - Estimation (How big is this? Scale it)
S - Schema/Storage (Database design, data model)
H - High-level design (Architecture, components)
A - API design (Contract, endpoints)
D - Deep dive (Pick 1-2 components and detail them)
E - Evaluation (Tradeoffs, improvements)
D - Discuss (Scale, security, costs)
```

---

## PART 2: YOUR PLATFORM ENGINEER ADVANTAGE

### What Interviewers Expect From Your Level (14 Years + Certs)

**For Senior Platform Engineer / Infrastructure Architect:**

✅ **Must demonstrate:**
- Multi-tenant isolation concerns (not single-tenant thinking)
- Cost-aware design (₹/request, not just technical metrics)
- Operational concerns (monitoring, alerting, debugging)
- Trade-offs (consistency vs availability, latency vs cost)
- Real-world constraints (compliance, SLA, regulations)

✅ **Should mention:**
- Infrastructure as code (Terraform, Helm)
- GitOps (Argo CD, Flux)
- Observability from day 1 (not afterthought)
- Security by design (RBAC, encryption, audit)
- Cost optimization strategies

✅ **Nice to have:**
- Load testing methodology
- Disaster recovery plans
- Migration strategies (from old to new)
- Vendor lock-in concerns

**What NOT to do:**
- ❌ Design like a junior (single machine, no scaling)
- ❌ Ignore operational complexity
- ❌ Forget about costs
- ❌ Assume infinite resources
- ❌ Ignore security until end

---

## PART 3: 15 SYSTEM DESIGN QUESTIONS (TAILORED FOR YOU)

### Q1: Design a Multi-Tenant GenAI PaaS Platform (Your Actual Job)

**The Question:**
> "Design a platform where customers can integrate GenAI capabilities into their applications. The platform should handle multi-tenancy, cost tracking, observability, and support both synchronous and asynchronous GenAI operations."

**What They're Really Asking:**
- Can you design for multiple customers safely?
- How do you track costs and bill customers?
- How do you handle failures in external APIs (Claude)?
- How do you scale?
- How do you operate this in production?

**Your RESHADED Approach:**

#### **R - Requirements (Clarify)**

**Functional:**
```
1. Customers call GenAI API via our platform
2. We call Claude API on their behalf
3. We track usage per customer
4. We provide observability (logs, metrics)
5. Support both sync (wait for response) and async (queue)
6. Support multiple GenAI models (not just Claude)
```

**Non-Functional:**
```
- Scale: 1M requests/day, 100+ customers
- Latency: P95 < 1000ms for sync, async < 60s processing
- Availability: 99.9% SLA
- Cost: ₹0.05-0.10 per request (track accurately)
- Security: Tenant isolation, API keys, audit logs
```

**Constraints:**
```
- Claude API rate limit: 10K req/min
- Claude API cost: ₹0.0005-0.005 per token
- Kubernetes cluster: 5 nodes, 500GB storage
- Budget: ₹1M/month operations
```

#### **E - Estimation**

```
Daily traffic: 1M requests
- Peak: 100K req/hour = 28 req/sec
- Average: 1K req/hour = 0.3 req/sec
- Gen AI calls: 80% of traffic = 800K/day

Per request:
- Input tokens: ~500 avg
- Output tokens: ~200 avg
- Total: 700 tokens/request
- Daily tokens: 700M tokens = ₹350K in API costs

Storage:
- Request logs: 1M req/day × 500 bytes = 500GB/day
- After 30 days: 15TB (need retention policy)
- Audit logs: 10GB/day

Database:
- Customer data: 100 customers × 10MB = 1GB
- Usage tracking: 1M requests/day × 1KB = 1GB/month
- Request cache: 100K entries × 1KB = 100MB
```

#### **S - Schema/Storage**

```
Database Design:

Customers table:
- id, name, tier, api_key, created_at
- Indexed: id, api_key

Usage table:
- customer_id, timestamp, tokens_input, tokens_output, cost
- Indexed: customer_id, timestamp
- Partitioned by month (for old data retention)

Requests table (audit log):
- id, customer_id, request_payload, response_payload, tokens, cost, timestamp
- Indexed: customer_id, timestamp
- Sharded by customer_id (multi-tenant)

Cache (Redis):
- Key: hash(customer_id + prompt)
- Value: response
- TTL: 3600 seconds
- Max size: 100MB

Storage Choice:
- Transactional data: PostgreSQL
  ├─ ACID guarantees
  ├─ Complex queries (billing calculations)
  ├─ Relatively small data
  
- Time-series data: ClickHouse or TimescaleDB
  ├─ Optimize for time-series (usage, metrics)
  ├─ Fast aggregations
  ├─ Compression
  
- Request logs: OpenSearch
  ├─ Full-text search
  ├─ Real-time queries
  ├─ Easy retention policies
  
- Cache: Redis
  ├─ Sub-millisecond latency
  ├─ TTL support
  ├─ Simple KV access
```

#### **H - High-Level Architecture**

```
┌─────────────────────────────────────────────────────────────┐
│                     Customer Applications                     │
│                     (Using our API)                          │
└──────────────────────────┬──────────────────────────────────┘
                           │
                    ┌──────▼────────┐
                    │ API Gateway   │ (Rate limiting, auth)
                    └──────┬────────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
    ┌─────▼──────┐  ┌─────▼──────┐  ┌────▼────────┐
    │  Sync API  │  │ Async API  │  │ Management  │
    │  (immediate)  │  (queued)   │  │ API         │
    └─────┬──────┘  └─────┬──────┘  └────┬────────┘
          │                │              │
    ┌─────┴────────────────┴──────────────┴─────┐
    │         GenAI Service (Pods)              │
    │  ┌──────────────────────────────────────┐ │
    │  │ 1. Token counter                     │ │
    │  │ 2. Cache check (Redis)               │ │
    │  │ 3. Circuit breaker                   │ │
    │  │ 4. Claude API call                   │ │
    │  │ 5. Log response (OpenSearch)         │ │
    │  │ 6. Update cost (PostgreSQL)          │ │
    │  └──────────────────────────────────────┘ │
    └─────────────────────────────────────────────┘
          │
    ┌─────┴──────────────────────────────┐
    │   Claude API (external)             │
    │   (Call on customer's behalf)       │
    └────────────────────────────────────┘

Async path:
┌─────────────────────┐
│  Async API Request  │
└─────────┬───────────┘
          │
    ┌─────▼─────────────┐
    │ Kafka/RabbitMQ    │ (Queue request)
    └─────┬─────────────┘
          │
    ┌─────▼──────────────────┐
    │ Background Worker      │ (Process queued requests)
    │ (scale independently)  │
    └─────┬──────────────────┘
          │
    ┌─────▼──────────────────┐
    │ Notify Customer        │ (webhook, long-polling)
    │ (Result is ready)      │
    └───────────────────────┘

Infrastructure:
┌───────────────────────────────────────────┐
│         Kubernetes Cluster                │
│  ┌─────────────────────────────────────┐  │
│  │ Namespace: platform                 │  │
│  │  - API Gateway                      │  │
│  │  - GenAI Service (3-5 replicas)     │  │
│  │  - Background Workers               │  │
│  │                                     │  │
│  │ Namespace: data                     │  │
│  │  - PostgreSQL (StatefulSet)         │  │
│  │  - Redis (StatefulSet)              │  │
│  │  - OpenSearch (3-node cluster)      │  │
│  │                                     │  │
│  │ Namespace: observability            │  │
│  │  - Prometheus                       │  │
│  │  - Grafana                          │  │
│  │  - Fluent Bit (logging)             │  │
│  └─────────────────────────────────────┘  │
└───────────────────────────────────────────┘
```

#### **A - API Design**

```
Synchronous API:

POST /api/v1/chat
Headers:
  X-API-Key: customer_api_key
  Content-Type: application/json

Request:
{
  "messages": [
    {"role": "user", "content": "Explain quantum computing"}
  ],
  "model": "claude-opus",
  "temperature": 0.7,
  "max_tokens": 500
}

Response (200 OK):
{
  "id": "req_123456",
  "content": "Quantum computing is...",
  "tokens_used": {
    "input": 15,
    "output": 120
  },
  "cost": "₹0.05"
}

Error Responses:
- 401: Invalid API key
- 429: Rate limit exceeded
- 500: Claude API error (with fallback response or queue option)
- 503: Service unavailable

---

Asynchronous API:

POST /api/v1/chat/async
{
  "messages": [...],
  "model": "claude-opus",
  "webhook_url": "https://customer.com/webhook"  (optional)
}

Response (202 Accepted):
{
  "request_id": "async_789012",
  "status": "queued",
  "estimated_wait": "30 seconds"
}

Customer polls:
GET /api/v1/chat/async/{request_id}

Response (when ready):
{
  "request_id": "async_789012",
  "status": "completed",
  "content": "...",
  "tokens_used": {...},
  "cost": "₹0.05"
}

Or webhook callback to customer with result.

---

Management API:

GET /api/v1/customers/me
- Get current customer info, quota, billing

GET /api/v1/usage
- Get usage metrics, cost breakdown

POST /api/v1/models
- List available models, pricing

GET /api/v1/billing/invoices
- Download invoices
```

#### **D - Deep Dive (2 Components)**

**Component 1: GenAI Service Pod (The Core)**

```python
# Pseudocode of single request processing

class GenAIService:
    def __init__(self):
        self.cache = RedisClient()
        self.db = PostgreSQLClient()
        self.limiter = TokenLimiter()
        self.circuit_breaker = CircuitBreaker()
        self.logger = OpenSearchLogger()
    
    def process_request(self, customer_id, prompt):
        request_id = generate_id()
        start_time = now()
        
        try:
            # Step 1: Validate
            customer = self.db.get_customer(customer_id)
            if not customer:
                raise InvalidCustomerError()
            
            # Step 2: Estimate tokens
            estimated_tokens = estimate_tokens(prompt)
            estimated_cost = estimated_tokens * PRICE_PER_TOKEN
            
            # Step 3: Check budget
            remaining_budget = self.db.get_remaining_budget(customer_id)
            if estimated_cost > remaining_budget:
                raise BudgetExceededError()
            
            # Step 4: Check rate limit (tokens per minute)
            if not self.limiter.allow(customer_id, estimated_tokens):
                raise RateLimitError()
            
            # Step 5: Check cache (don't call Claude if already have response)
            cache_key = hash(customer_id + prompt)
            cached_response = self.cache.get(cache_key)
            if cached_response:
                self.log_cache_hit(request_id, customer_id)
                return cached_response
            
            # Step 6: Call Claude API (with circuit breaker)
            if self.circuit_breaker.is_open():
                # Claude API having issues, fallback
                return self.fallback_response(prompt)
            
            try:
                response = self.call_claude_api(prompt)
            except TimeoutError:
                # Timeout after 2 seconds, return partial or fallback
                return self.fallback_response(prompt)
            
            # Step 7: Cache response (for identical future queries)
            self.cache.set(cache_key, response, ttl=3600)
            
            # Step 8: Log request and response
            actual_tokens = response.tokens_used
            actual_cost = actual_tokens * PRICE_PER_TOKEN
            
            self.logger.log({
                'request_id': request_id,
                'customer_id': customer_id,
                'prompt': prompt,
                'response': response.content,
                'tokens_input': response.tokens.input,
                'tokens_output': response.tokens.output,
                'cost': actual_cost,
                'latency_ms': (now() - start_time) * 1000,
                'cache_hit': False,
                'timestamp': now()
            })
            
            # Step 9: Update database (cost tracking, usage)
            self.db.add_usage_entry({
                'customer_id': customer_id,
                'tokens_input': response.tokens.input,
                'tokens_output': response.tokens.output,
                'cost': actual_cost,
                'timestamp': now()
            })
            
            # Step 10: Update Prometheus metrics
            self.metrics.request_latency.observe((now() - start_time) * 1000)
            self.metrics.tokens_used.inc(actual_tokens)
            self.metrics.api_cost.inc(actual_cost)
            
            return response
            
        except BudgetExceededError as e:
            self.logger.log_error(request_id, customer_id, e)
            raise
        except RateLimitError as e:
            self.logger.log_error(request_id, customer_id, e)
            raise
        except Exception as e:
            # Unknown error
            self.logger.log_error(request_id, customer_id, e)
            # Try fallback
            return self.fallback_response(prompt)
```

**Key Design Decisions:**
1. **Token counting before API call** (cost estimation)
2. **Circuit breaker** (protect from Claude API cascading failures)
3. **Caching by (customer_id + prompt)** (reduce API calls by 30-50%)
4. **Logging everything** (for debugging, billing, compliance)
5. **Timeouts** (don't wait forever for Claude)
6. **Budget enforcement** (stop overruns)

---

**Component 2: Cost Tracking & Billing**

```
Multi-tenant cost attribution:

For each request:
├─ GenAI API cost (most expensive)
│  ├─ Tokens × price per token
│  └─ Example: 1000 tokens × ₹0.0005 = ₹0.50
│
├─ Compute cost (Kubernetes pod CPU/memory)
│  ├─ Pod CPU × duration × hourly_rate
│  ├─ Pod memory × duration × hourly_rate
│  └─ Example: 100m CPU × 1sec × ₹0.002 = ₹0.0002
│
├─ Storage cost (database, OpenSearch)
│  └─ Amortized across all requests
│
└─ Bandwidth cost (minimal)
   └─ Amortized across all requests

Daily cost breakdown (for finance team):

Customer: BigCorp
- Requests: 100K
- GenAI API cost: ₹5000 (80%)
- Compute cost: ₹1000 (16%)
- Storage cost: ₹250 (4%)
- Total: ₹6250 per day = ₹187.5K per month

Billing:
- We charge: ₹0.10 per request
- Revenue: 100K × ₹0.10 = ₹10K/day = ₹300K/month
- Cost: ₹6250/day = ₹187.5K/month
- Profit: ₹112.5K/month = 37.5% margin

Database schema for billing:

UsageTable:
├─ customer_id
├─ timestamp
├─ requests_count
├─ tokens_input
├─ tokens_output
├─ genai_api_cost
├─ compute_cost
├─ storage_cost_allocated
└─ total_cost

BillingTable:
├─ customer_id
├─ period (month)
├─ total_usage_cost
├─ discount_applied
├─ final_charge
└─ status (draft, sent, paid)
```

#### **E - Evaluation (Trade-offs)**

| Aspect | Choice | Trade-off |
|--------|--------|-----------|
| **Database** | PostgreSQL | ACID but not NoSQL (fine for small data) |
| **Caching** | Redis | Fast but volatile (data loss on restart) |
| **Queue** | Kafka | Reliable but complex (RabbitMQ is simpler) |
| **Logging** | OpenSearch | Real-time search but expensive |
| **GenAI calls** | Circuit breaker | Lose some requests but protect platform |
| **Scaling** | HPA (auto-scale) | Good but can't predict spikes |
| **Multi-tenancy** | Namespace per tenant | Good isolation but more Kubernetes resources |
| **Cost tracking** | Per-request | Accurate but slower than batch |

#### **D - Discuss (Scale, Security, Costs)**

**Scaling to 10x:**
```
Current: 1M req/day
Target: 10M req/day

Bottleneck analysis:
- API Gateway: Can handle, add load balancer
- GenAI Service: Scale from 5 to 20 replicas
- Database: Add read replicas (PostgreSQL)
- Redis: Add Redis cluster
- OpenSearch: Add more nodes
- Claude API: Rate-limited to 10K req/min
  → Need fallback model (Haiku instead of Opus)

Cost increase:
- Kubernetes compute: +₹50K/month
- Database: +₹10K/month
- OpenSearch: +₹5K/month
- Claude API: +₹2M/month (5x volume)
- Total: +₹2M/month for 10x revenue increase
```

**Security Concerns:**
```
1. API key leakage
   → Store as hashed keys, rotate regularly

2. Prompt injection
   → Validate input length, sanitize

3. Data privacy
   → Namespace isolation
   → Encryption at rest (for PostgreSQL, OpenSearch)
   → Audit logging

4. Compliance
   → GDPR: Right to be forgotten (delete customer data)
   → SOC2: Audit trails, access control
   → Data residency: EU customers = EU storage
```

**Cost Optimization:**
```
1. Use cheaper model when possible
   - Haiku for simple queries (₹0.0001/token)
   - Opus only when needed (₹0.005/token)
   - Savings: 30-50% on GenAI costs

2. Caching
   - Cache identical prompts
   - Savings: 30% of requests from cache

3. Batch processing
   - Combine multiple requests
   - Savings: 10% from batch efficiency

4. Reserved capacity (for steady-state load)
   - Commit to baseline with provider
   - Savings: 20-30% vs on-demand
```

---

### Q2: Design an Observability Platform

**The Question:**
> "Design an observability platform that collects logs, metrics, and traces from multiple sources, allows customers to query them, and provides alerting. Consider multi-tenancy and cost."

**Your answer structure:**

```
Requirements:
- Ingest logs from 100s of customer apps
- Real-time querying (< 1 second)
- Metrics storage and aggregation
- Distributed traces (end-to-end request flow)
- Multi-tenant isolation

Estimation:
- Daily volume: 10TB of logs (1M req/day × 10KB avg)
- 100 customers
- 90-day retention (900TB total storage)
- Query rate: 100 queries/sec

Architecture:
- Fluent Bit/OTEL Collector → Kafka → OpenSearch (logs)
- Prometheus → ClickHouse (metrics)
- Jaeger → Elasticsearch (traces)
- Grafana (query + visualize)

Data model:
- Logs: timestamp, tenant_id, service, message, level
- Metrics: timestamp, tenant_id, metric_name, value, labels
- Traces: trace_id, span_id, service, latency, status

Scaling:
- 10x: Add Kafka partitions, OpenSearch shards
- Multi-region: Replicate data to backup region
- Cost: Move old logs to S3 (cheaper)
```

---

### Q3: Design Multi-Region Failover for GenAI Platform

**The Question:**
> "Your GenAI platform is currently in one region (India). Design a multi-region deployment where if the India region goes down, traffic automatically fails over to another region with minimal data loss."

**Key considerations:**
```
1. Primary-Secondary setup
   - Primary: India (active)
   - Secondary: US or Europe (standby)

2. Data replication
   - Sync replication (slow but no data loss)
   - Async replication (fast but possible data loss)
   - Trade-off: RPO = 1 minute (max 1 min of data loss)

3. Failover detection
   - Active health checks (primary sends heartbeat)
   - If no heartbeat for 30 seconds → failover
   - Switch DNS/load balancer to secondary

4. Costs
   - Running 2 full clusters = 2x cost
   - But for critical services, necessary
   - Can optimize: standby region runs at 20% capacity, scales up on failover

5. Challenges
   - Customer sessions (JWT tokens valid in both regions?)
   - Database consistency (PostgreSQL replication lag)
   - Caching (Redis in secondary is cold after failover)
   - Webhooks (customer APIs might get called twice)
```

---

### Q4: Design API Rate Limiting & Quota System

**The Question:**
> "Design a rate limiting system for your GenAI platform where different customer tiers have different limits. How would you implement this globally (multi-region)?

**Answer approach:**
```
Tiers:
- Free: 10 req/min, 1K tokens/day
- Pro: 1000 req/min, 100K tokens/day
- Enterprise: Custom limits

Rate limiting logic:
1. Token bucket algorithm (not fixed window)
   - Each customer starts with N tokens
   - Each request costs 1 token
   - Tokens refill at rate_limit/60 tokens per second

2. Implementation:
   - Redis for fast checking
   - Key: customer_id:quota:minute
   - Value: tokens_remaining
   - Incr by refill rate every second (or check on request)

3. Global (multi-region):
   - Cannot use local state (each region has different counts)
   - Use central Redis cluster (replicated across regions)
   - Or use eventual consistency (each region tracks, sync hourly)
   - Trade-off: Small chance of going slightly over limit

4. Edge cases:
   - What if customer hits limit? (return 429, queue request)
   - Burst allowance? (allow 2x for 10 seconds)
   - Reset time? (per minute, per hour, or per day)
```

---

### Q5: Design a Real-time Notification System

**The Question:**
> "Design a notification system that sends alerts to customers when their spending exceeds thresholds. Must be real-time and guarantee delivery."

**Architecture:**
```
Data flow:
1. Usage event → Kafka topic (usage_events)
2. Aggregator service:
   - Calculates daily spend for each customer
   - Detects threshold breach (80%, 100%)
3. Alert generator:
   - Creates notification
   - Checks user preferences (email, Slack, webhook)
4. Multi-channel delivery:
   - Email: SES/SendGrid
   - Slack: Slack API
   - SMS: Twilio
   - Webhook: Call customer's endpoint
5. Retry logic:
   - Failed notifications retry with exponential backoff
   - Track delivery status
   - Manual retry if needed

Guarantee delivery:
- At-least-once: Store notification in DB before sending
- If send fails, retry
- Customer verifies they received it

Real-time:
- Process usage events within 10 seconds
- Send alert within 1 minute of threshold breach
```

---

## PART 4: MOST COMMON MISTAKES IN SYSTEM DESIGN INTERVIEWS

### 1. ❌ Designing like a junior (single server, no scaling)

```
Bad answer:
- "I'll put everything on one server"
- "MySQL will handle all data"
- "No caching needed"

Good answer:
- "Load balancer → multiple app servers → database"
- "PostgreSQL for transactional, Redis for cache, OpenSearch for logs"
- "Multi-region failover at high levels"
```

### 2. ❌ Forgetting about multi-tenancy concerns

```
Bad answer:
- Design for single customer, assume "we'll multi-tenant later"

Good answer:
- Isolation from day 1 (namespace per tenant)
- Cost tracking per tenant
- RBAC per tenant
- Audit logging per tenant
```

### 3. ❌ Ignoring the most expensive component

```
Bad answer:
- Focus on database performance when GenAI API is 10x more expensive

Good answer:
- Identify most expensive component (Claude API)
- Optimize there first (caching, cheaper model, better prompts)
- Don't over-engineer other parts
```

### 4. ❌ Not asking clarifying questions

```
Bad approach:
- Interviewer: "Design a chat system"
- You: Draw architecture for 1B users immediately

Good approach:
- "Is this just a chat app, or entire social network?"
- "What's the scale? 1M users or 1B users?"
- "Real-time or batch?"
- "Emphasize speed, cost, or reliability?"
```

### 5. ❌ Designing without constraints

```
Bad answer:
- "I'll use 100 microservices with Kubernetes everywhere"

Good answer:
- Know your constraints (budget: ₹1M/month, team size: 5, etc.)
- Design within constraints
- Mention what you'd change with different constraints
```

### 6. ❌ Forgetting operational aspects

```
Bad answer:
- Only design the happy path

Good answer:
- Monitoring (how do we know it's broken?)
- Alerting (how do we get paged?)
- Runbooks (what do we do when alerted?)
- Logging (how do we debug?)
- Cost tracking (how much are we spending?)
```

### 7. ❌ Making assumptions without stating them

```
Bad answer:
- "I'll use PostgreSQL" (without explaining why)

Good answer:
- "I'm using PostgreSQL because we need ACID guarantees for billing accuracy, and data volume is small enough that it handles it well. If data grows to PB scale, we'd shard or switch to a distributed database."
```

### 8. ❌ Not discussing tradeoffs

```
Bad answer:
- "I'll use eventual consistency"

Good answer:
- "I'll use eventual consistency because X (reduced latency).
  But this means Y (possible temporary inconsistency).
  Risk is Z (e.g., customer billed twice).
  Mitigation: A (idempotency checks, reconciliation)."
```

---

## PART 5: HOW TO STRUCTURE YOUR ANSWER (60-MINUTE INTERVIEW)

```
Minute 0-5: Clarifying Questions
- Scope? (whole system or just part)
- Scale? (100K users? 1M? 1B?)
- What matters most? (speed, cost, reliability)
- Constraints? (budget, team size)

Minute 5-10: Requirements & Estimation
- Functional: what system does
- Non-functional: performance targets
- Capacity: requests/day, users, storage
- Cost estimates

Minute 10-20: High-Level Architecture
- Major components (API, service, database, cache)
- Data flow (request → response)
- Draw on whiteboard/document
- Explain each component briefly

Minute 20-30: Deep Dive on 1-2 Components
- Pick the hardest parts
- Explain design decisions
- Show code or pseudocode
- Handle edge cases

Minute 30-45: Scaling & Tradeoffs
- How to scale to 10x
- Data consistency vs availability (CAP)
- Cost optimization
- Security concerns

Minute 45-60: Questions & Discussion
- Interviewer asks follow-ups
- Discuss alternative approaches
- Real-world production concerns
- Hiring signal: how you handle disagreement
```

---

## PART 6: KILLER INTERVIEW PHRASES

**These phrases signal you're senior:**

### 1. "Let me state my assumptions..."
```
Good: "Assuming this is a read-heavy system with 1M daily active users..."
Shows: You know how assumptions affect design
```

### 2. "Given current constraints, I'd optimize for X, but if Y changes, I'd reconsider..."
```
Good: "With budget of ₹1M/month, I'd use Postgres. But if budget doubled, I'd use distributed Cassandra."
Shows: Practical thinking, constraint-aware
```

### 3. "This component will be the bottleneck at scale..."
```
Good: "Database writes will bottleneck. Before that happens, I'd implement sharding by customer_id."
Shows: Thinking ahead, proactive scaling
```

### 4. "The tradeoff is between X and Y..."
```
Good: "Caching reduces latency by 50% but adds complexity. I'd measure impact first."
Shows: Balanced thinking, measurement-driven
```

### 5. "We need monitoring/alerting for this because..."
```
Good: "If cache miss rate exceeds 20%, queries will timeout. Alert on-call team at 15%."
Shows: Operational maturity
```

### 6. "Cost per request is important because..."
```
Good: "At ₹0.10 revenue per request, if cost exceeds ₹0.07, margin disappears. I'd optimize aggressively."
Shows: Business-aware thinking
```

---

## PART 7: QUICK DECISION TREE FOR COMPONENT CHOICES

```
Need fast reads?
├─ Yes: Use cache (Redis, Memcached)
└─ No: Direct database read is fine

Need to store logs at scale?
├─ Yes: Use OpenSearch or ClickHouse (not PostgreSQL)
└─ No: PostgreSQL fine

Need to process events in real-time?
├─ Yes: Use Kafka or event streaming
└─ No: Batch processing is fine

Need sub-millisecond latency?
├─ Yes: Use in-memory (Redis, cache)
└─ No: Database latency (10-100ms) acceptable

Need to handle 100K req/sec?
├─ Yes: Distributed database (Cassandra, ScyllaDB)
└─ No: Single-instance (PostgreSQL) fine

Need multi-region?
├─ Yes: Synchronize data (complex)
└─ No: Single region (simple)

Need exactly-once processing?
├─ Yes: Add deduplication logic
└─ No: At-least-once is simpler

Need compliance/audit?
├─ Yes: Log everything, encryption
└─ No: Standard security fine
```

---

## PART 8: PRACTICE SYSTEM DESIGN QUESTIONS

### Easy (For warm-up):

1. **Design TinyURL**
   - Shorten URLs, redirect to original
   - Estimate: 1M shortening/day, 100M redirects/day
   - Challenge: Handle collision, redirect instantly

2. **Design a Cache**
   - In-memory key-value store
   - Eviction policy (LRU, LFU)
   - Expiration (TTL)

3. **Design a User Authentication System**
   - Sign up, login, JWT tokens
   - Password hashing, salt
   - 2FA for security

### Medium (Realistic):

4. **Design an Analytics System**
   - Collect events from apps
   - Aggregate metrics
   - Dashboard + queries

5. **Design a Recommendation System**
   - Suggest products/content
   - Personalization
   - Real-time vs batch

6. **Design a File Storage System (Dropbox-like)**
   - Upload, download, sync
   - Multi-device
   - Conflict resolution

### Hard (Your level):

7. **Design Multi-Tenant SaaS Platform**
   - Multiple customers on shared infra
   - Cost tracking and billing
   - Isolation and security
   - ← This is YOUR level

8. **Design Real-Time Collaboration Tool** (Figma-like)
   - Simultaneous editing
   - Conflict resolution
   - OT/CRDT algorithm

9. **Design Distributed Transaction System**
   - ACID across multiple databases
   - Consensus algorithms
   - Failure handling

---

## ✅ FINAL CHECKLIST BEFORE INTERVIEW

```
☐ Understand RESHADED framework
☐ Know your platform (Ai-Next) architecture deeply
☐ Can explain multi-tenancy design decisions
☐ Understand GenAI-specific constraints (cost, latency, API limits)
☐ Know cost calculations (tokens, compute, storage)
☐ Can design monitoring/observability
☐ Know how to handle failures (circuit breaker, fallback, retry)
☐ Can discuss tradeoffs (consistency vs availability, latency vs cost)
☐ Know when to use which database/cache/queue
☐ Can explain operational concerns (alerting, logging, debugging)
☐ Ask clarifying questions first
☐ Draw architecture clearly
☐ Explain reasoning, not just "here's the design"
☐ Mention constraints and how they affect design
☐ Discuss scaling strategy
☐ Calculate estimates (QPS, storage, cost)
```

---

*Generated: April 23, 2026 | Advanced System Design for Platform Engineers*
