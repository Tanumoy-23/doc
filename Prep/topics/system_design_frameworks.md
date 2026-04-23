# SYSTEM DESIGN INTERVIEW: BEST PRACTICES & FRAMEWORKS
## LinkedIn Top Insights + Interview-Ready Questions

---

## PART 1: THE 6-STEP FRAMEWORK (Based on Industry Best Practices)

Most successful candidates follow this exact structure:

### Step 1: CLARIFY & SCOPE (5 minutes)

**What the interviewer wants:**
- See if you ask good questions (not jump to design)
- Understand if you know how to scope projects
- Verify you won't design the wrong thing

**Questions you should ask:**

```
Scope questions:
□ "Are we designing the whole Instagram, or just the feed?"
□ "Is this for mobile, web, or both?"
□ "Are we starting from scratch or integrating with existing systems?"

Scale questions:
□ "How many daily active users? 1M? 100M?"
□ "What's the geographic distribution?"
□ "Peak QPS? Average QPS?"

Business questions:
□ "What does success mean? Speed? Cost? Reliability?"
□ "Are we optimizing for growth or profitability?"
□ "What's the revenue model?"

Technical constraints:
□ "Any existing technology we must integrate with?"
□ "Database is chosen, or we pick?"
□ "Budget constraints?"

Real example for your profile:
"If designing a GenAI platform:
 - How many tenants? 10? 1000?
 - What GenAI providers? Claude only or multiple?
 - Scale: 1M requests/day?
 - Cost-aware? We're billing customers?
 - Multi-region or single region?"
```

**Killer move:** Write down their answers. Shows you're serious.

---

### Step 2: ESTABLISH METRICS (5 minutes)

**What you're doing:** Turning requirements into numbers

```
Functional requirements:
├─ User can upload content
├─ User can view content
├─ Content is searchable
└─ Content expires after 30 days

Non-functional requirements (what matters):
├─ Latency: P95 < 100ms
├─ Availability: 99.99% uptime
├─ Consistency: Eventually consistent OK
└─ Cost: < $0.01 per operation

Then quantify:
├─ DAU: 100M
├─ Peak QPS: 100K req/sec
├─ Average QPS: 10K req/sec
├─ Storage: 1PB total, 1GB per day growth
├─ Bandwidth: 1Tbps
└─ Cost: $10M/month max

Example for GenAI platform:
├─ DAU: 100 customers
├─ QPS: 100 req/sec average, 500 at peak
├─ Tokens: 100K tokens/day average per customer
├─ Latency SLA: P95 < 500ms (GenAI is slow)
├─ Availability SLA: 99.5% (GenAI API sometimes fails)
├─ Cost per request: < ₹0.10 (charged at ₹0.20)
└─ Cost per month: < ₹500K (100 customers × ₹5K each)
```

**Pro tip:** Always mention cost. Shows business thinking.

---

### Step 3: HIGH-LEVEL ARCHITECTURE (10 minutes)

**What you're doing:** Showing the big picture

```
Rule: If you can't draw it, you don't understand it.

On whiteboard/document, show:
1. Entry point (load balancer, API gateway)
2. Main services (what do they do?)
3. Data stores (where is data?)
4. External integrations
5. Message queues (if async)
6. Cache layers

Example (GenAI Platform):

    ┌──────────────────────────┐
    │     API Gateway          │ (auth, rate-limit, route)
    └──────────┬───────────────┘
               │
         ┌─────┴──────┐
         │            │
    ┌────▼────┐  ┌───▼──────┐
    │Sync API │  │Async API │
    └────┬────┘  └───┬──────┘
         │           │
         └─────┬─────┘
               │
         ┌─────▼─────────────┐
         │ GenAI Service     │ (the core logic)
         └─────┬─────────────┘
               │
         ┌─────┴───────────┬────────────┐
         │                 │            │
    ┌────▼────┐       ┌───▼──┐    ┌───▼──────┐
    │PostgreSQL│       │Redis │    │Claude API│
    │(metadata)│       │Cache │    │(external)│
    └──────────┘       └──────┘    └──────────┘
    
    ┌──────────────────────────┐
    │ OpenSearch               │ (logs + audit)
    └──────────────────────────┘
    
    ┌──────────────────────────┐
    │ Prometheus               │ (metrics)
    └──────────────────────────┘
```

**Key points:**
- Every box should be labeled (explain its job)
- Show data flow (arrows)
- Identify bottlenecks (mark them)

---

### Step 4: DEEP DIVE (20 minutes)

**What you're doing:** Proving you know the details

**Strategy:** Pick 1-2 hardest components, go deep

```
Example: Deep dive on GenAI Service

"Let me detail the GenAI Service, which is the most complex part.

When a request comes in:
1. Validate API key (check PostgreSQL)
2. Estimate tokens (count before calling API)
3. Check cache (Redis) - if hit, return immediately
4. Enforce rate limits (token bucket algorithm)
5. Check budget (customer quota)
6. Call Claude API (with timeout)
7. Log response (OpenSearch for debugging)
8. Update costs (PostgreSQL for billing)
9. Return to user (or queue if async)

Here's pseudocode:

def process_genai_request(customer_id, prompt):
    # Validation
    customer = db.get(customer_id)
    if not customer:
        return error(401)
    
    # Token estimation
    tokens = estimate_tokens(prompt)
    cost = tokens * PRICE_PER_TOKEN
    
    # Cache check
    cached = cache.get(hash(customer_id, prompt))
    if cached:
        return cached
    
    # Rate limit
    if not rate_limit.allow(customer_id, tokens):
        return error(429)
    
    # Budget check
    if cost > customer.remaining_budget:
        return error(402)
    
    # Call Claude
    try:
        response = claude_api.call(prompt, timeout=2s)
    except Timeout:
        return fallback_response()
    
    # Log + bill
    log(customer_id, tokens, cost)
    db.update_cost(customer_id, cost)
    
    return response

Key design decisions:
- Token count BEFORE calling API (cost estimation)
- Cache to reduce API calls
- Rate limiting at 3 levels: API gateway, customer-level, token-level
- Timeout (2 seconds) to prevent hanging
- Fallback for failures
- Logging for debugging + compliance
"
```

**What impresses:**
- Show you've thought about edge cases
- Explain WHY each decision
- Handle failures (not just happy path)
- Mention scalability implications

---

### Step 5: SCALING & TRADEOFFS (15 minutes)

**What you're doing:** Proving you know how to handle growth

```
Scaling analysis:

Current system handles:
- 100 req/sec
- 100K tokens/day
- 1M requests/day

Question: How to scale to 10x (1M req/sec)?

Bottleneck analysis:
1. Claude API: Rate limit is 10K req/min
   - Current: 100 req/sec = 6K req/min (OK)
   - 10x: 1K req/sec = 60K req/min (EXCEEDS limit)
   - Solution: Use fallback model (Haiku), implement queueing

2. PostgreSQL: Can it handle 1K writes/sec?
   - Current: 100 writes/sec (OK)
   - 10x: 1K writes/sec (maybe, depends on CPU)
   - Solution: Add replicas for reads, master for writes

3. Redis cache: Can it handle 1K req/sec?
   - Current: 100 req/sec (OK)
   - 10x: 1K req/sec (easy, Redis can handle 100K/sec)
   - Solution: Add sharding if cache grows too large

4. API Gateway: Can it handle 1K req/sec?
   - Current: 100 req/sec (OK)
   - 10x: 1K req/sec (yes, horizontal scale)
   - Solution: Add more instances, load balancer

Scaling plan:
1. Claude API bottleneck is THE limiting factor
   - Can't scale beyond Claude's rate limit without fallback
   - Action: Implement Haiku fallback, reduce tokens

2. Database: Add read replicas
   - Primary handles writes
   - Replicas handle reads
   - Replication lag: acceptable (1 second)

3. Application: Horizontal scaling
   - Current: 5 pods
   - Target: 20 pods
   - Cost increase: ₹50K/month

4. Caching: Improve hit rate
   - Current: 30% hit rate
   - Target: 50% hit rate
   - Savings: 20% fewer API calls

Cost implications:
- Current: ₹500K/month
- 10x scale: ₹2.5M/month (mostly Claude API)
- Optimization potential: ₹1.5M/month (fallback model + caching)
- Final: ₹2M/month (15% higher efficiency at 10x scale)
```

**Killer move:** Show you've thought about cost growth

---

### Step 6: TRADEOFFS & DISCUSSION (10 minutes)

**What you're doing:** Showing mature thinking

```
CAP Theorem tradeoff:

In my design, I chose:
- Consistency (C): Moderate
  └─ PostgreSQL has immediate consistency for cost tracking
- Availability (A): High
  └─ If Claude API fails, use fallback (still respond)
- Partition tolerance (P): High
  └─ Multi-region ready

Trade-offs made:
1. Caching for speed, but risk of stale data
   → Acceptable because TTL = 1 hour (usually accurate)
   → If problem: validation in next request

2. Async processing for scale, but delayed notification
   → Acceptable because billing delay is OK
   → Not acceptable for payment processing (would make sync)

3. Replication lag in read replicas (2-5 seconds)
   → Acceptable for analytics queries
   → Not acceptable for exact balances (query primary)

4. GenAI cost spike risk if customer abuses API
   → Mitigated by rate limiting + budget enforcement
   → Worst case: lose 1 customer, protect others

Consistency options I considered:
A) Strong consistency (primary key check before debit)
   - Cost: Higher latency, lower throughput
   - Benefit: No double-charges
   - Decision: Too expensive

B) Eventual consistency (debit, then verify)
   - Cost: Low, high throughput
   - Risk: Possible double-charge
   - Decision: Mitigate with idempotency keys + reconciliation

C) My choice (eventual + validation)
   - Primary handles writes (atomic)
   - Replicas handle reads (may be slightly stale)
   - Nightly reconciliation catches errors
   - Best of both worlds

Availability vs Cost tradeoff:
- 99.9% availability: ₹500K/month (current design)
- 99.99% availability: ₹1.2M/month (multi-region, hot standby)
- 99.999% availability: ₹3M/month (complex, diminishing returns)

For GenAI platform, I chose 99.5% because:
- Cloud APIs (Claude) are 99.9%, so we can't be better
- Customers can implement retries
- Cost doubles for minimal benefit
- Business decision: Good enough at 99.5%
```

---

## PART 2: 10 SYSTEM DESIGN QUESTIONS (TAILORED TO YOUR PROFILE)

### Q1: Design a Cost Allocation System for Multi-Tenant PaaS
```
Problem: You have 100 shared Kubernetes nodes running 1000 pods from 100 customers.
How do you fairly allocate costs to each customer?

Considerations:
- CPU/memory allocated vs actual used
- Storage costs (shared vs dedicated)
- Bandwidth costs
- External API costs (charged directly)
- Overhead allocation (platform ops, monitoring)

Answer outline:
1. Metered resources (Prometheus metrics per pod)
2. Cost calculation per resource type
3. Overhead allocation (per-tenant overhead)
4. Billing tier (monthly, pay-as-you-go)
5. Dispute resolution (customer questions cost)
```

### Q2: Design a Request Deduplication System
```
Problem: Async GenAI requests can timeout and retry. How do you prevent double-processing?

Example:
- Customer sends: Process this document
- Timeout after 2 seconds
- Automatic retry
- But first request succeeds after 5 seconds
- Result: Document processed twice

Answer outline:
1. Idempotency key (customer provides)
2. Store in cache (first 10 min)
3. Detect retry, return cached result
4. Webhook called only once
```

### Q3: Design a Feature Flag System
```
Problem: How do you roll out new GenAI models to 20% of customers first?

Answer outline:
1. Feature flag service (returns true/false)
2. Store in Redis (fast)
3. Rules: by customer ID, by tenant, by percentage
4. No deployment needed (change flag at runtime)
5. Observability: track which customers enabled
```

### Q4: Design a Distributed Tracing System
```
Problem: Customer reports slow request. How do you find which service is slow?

Answer outline:
1. Jaeger/Zipkin for tracing
2. Trace ID propagated through all services
3. Each service records duration
4. Span tree shows: API → GenAI → Claude API → DB
5. Visualize latency breakdown
```

### Q5: Design an Audit Log System
```
Problem: Compliance requires audit trail. Who accessed what, when?

Answer outline:
1. Log all actions (create, read, update, delete)
2. Store in OpenSearch (append-only)
3. Include: user_id, action, resource, timestamp, IP
4. Immutable (can't delete logs)
5. Query for investigations
```

### Q6: Design a Backup and Disaster Recovery System
```
Problem: Database corruption or data center fire. How do you recover?

Answer outline:
1. RPO (Recovery Point Objective): how much data loss acceptable?
2. RTO (Recovery Time Objective): how long to recover?
3. Daily snapshots to different region
4. Test recovery monthly
5. Runbook for actual recovery
```

### Q7: Design a Cache Invalidation System
```
Problem: If document is updated, all caches referencing it must update.
How do you do this at scale?

Answer outline:
1. Event-driven (document updated → event to Kafka)
2. Cache service listens, invalidates
3. Or TTL-based (cache expires automatically)
4. Or explicit (API call to invalidate)
5. Testing: verify cache actually updated
```

### Q8: Design a Load Testing Framework
```
Problem: Before peak season, can your system handle 10x load?
How do you test this safely?

Answer outline:
1. Load test in staging (not production)
2. Simulate realistic traffic (not just hammering)
3. Gradual ramp-up (detect breaking point)
4. Measure: latency, errors, resource usage
5. Find bottleneck, optimize, repeat
```

### Q9: Design a Rollback Strategy
```
Problem: You deploy code that crashes in production.
How do you quickly restore service?

Answer outline:
1. Canary deploy (10% traffic first)
2. Monitor error rate, latency
3. Auto-rollback if error rate exceeds threshold
4. Manual rollback available
5. Practice regularly (runbooks)
```

### Q10: Design API Versioning Strategy
```
Problem: You need to change API but customers depend on old format.
How do you maintain backward compatibility?

Answer outline:
1. URL versioning (/v1/api, /v2/api)
2. Header versioning (API-Version: 2)
3. Graceful deprecation (announce 6 months in advance)
4. Support 2-3 versions simultaneously
5. Sunset old versions (stop supporting)
```

---

## PART 3: FRAMEWORKS FROM INDUSTRY LEADERS

### Framework 1: Google's SRE Principles

```
In system design, consider SRE principles:

1. Embrace risk (100% availability is impossible, don't aim for it)
2. SLO (Service Level Objective): Define target (99.9%)
3. SLA (Service Level Agreement): Contract with customers (99.5%)
4. Error budget: If 99.5% SLA, 0.5% errors/month allowed
   └─ Use budget for: deployments, experiments, risks
5. Monitoring: Causes of SLA breach
6. Incident response: What to do when it breaks
7. Blameless post-mortems: Learn from failures
```

### Framework 2: Amazon's Design Principles

```
1. Decoupling: Services don't depend on each other tightly
2. Async everywhere: Reduce blocking
3. Scale the database: Hard problem, solve early
4. Multi-region: Plan for global from day 1
5. Cost: Always know per-request cost
6. Security: By default, not after-thought
7. Compliance: GDPR, SOC2, etc.
```

### Framework 3: Netflix's Chaos Engineering

```
You can't test failure modes without actually failing. So:

1. Chaos monkey: Random failures in production
2. Chaos gorilla: Entire region fails
3. Chaos kong: Multi-region failure
4. Always ask: "What breaks if X fails?"
5. Practice recovery
```

---

## PART 4: RED FLAGS THE INTERVIEWER NOTICES

❌ You jump to code/architecture without asking questions
→ Shows lack of scoping skills

❌ You design for theoretical scale (1B users) without asking
→ Shows inexperience (you over-engineer)

❌ You never mention cost
→ Shows lack of business thinking

❌ You ignore failure modes (what if X breaks?)
→ Shows lack of operational experience

❌ You can't explain WHY you chose X over Y
→ Shows you're memorizing, not thinking

❌ You design sync when async is better (or vice versa)
→ Shows you don't understand tradeoffs

❌ You forget monitoring/alerting
→ Shows you've never run production

❌ You can't calculate load (QPS, storage, cost)
→ Shows estimation skills are weak

---

## PART 5: GREEN LIGHTS THE INTERVIEWER LOVES

✅ You ask clarifying questions first
→ Shows professionalism, scoping skills

✅ You write down requirements/constraints
→ Shows you care about getting it right

✅ You mention cost at multiple points
→ Shows business awareness

✅ You discuss tradeoffs (not "this is best")
→ Shows mature thinking

✅ You handle edge cases ("What if customer hits rate limit?")
→ Shows operational thinking

✅ You mention monitoring/logging/alerting
→ Shows you understand production

✅ You explain your assumptions
→ Shows clear communication

✅ You can defend your choices
→ Shows confidence in reasoning

✅ You can pivot design if constraints change
→ Shows flexibility

---

## PART 6: FINAL INTERVIEW CHECKLIST

**Before the interview:**
- [ ] Sleep well (mental clarity matters)
- [ ] Test your setup (whiteboard, document sharing, etc.)
- [ ] Have water nearby
- [ ] Wear something comfortable
- [ ] Minimize distractions

**During the interview:**
- [ ] Listen carefully to the question
- [ ] Ask 3-5 clarifying questions
- [ ] Write down requirements/constraints
- [ ] State your assumptions explicitly
- [ ] Draw architecture (show your thinking)
- [ ] Explain decisions (not just list components)
- [ ] Discuss tradeoffs (show maturity)
- [ ] Handle follow-ups gracefully
- [ ] If stuck, say so ("I haven't solved this exact problem, but here's my approach...")
- [ ] Be conversational (not a monologue)

**After the interview:**
- [ ] Send thank you (shows professionalism)
- [ ] Note feedback (if given)
- [ ] Prepare for next round (they may ask similar questions)

---

*Generated: April 23, 2026 | System Design Frameworks & Best Practices*
