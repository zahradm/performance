# Important KPIs for Performance Testing

## Part 1: What is a KPI?

KPI = Key Performance Indicator. The metrics that actually matter for your business.

```
Bad KPI: "Average response time is 200ms"
- Hides outliers (1% of users see 5000ms)
- Not actionable

Good KPI: "p99 response time ≤ 2000ms AND error rate < 0.1%"
- Shows what users actually experience
- Tied to business requirements
- Actionable (know when you've failed)
```

## Part 2: Response Time KPIs

How fast is the system from user perspective?

### Response Time Percentiles

```
Metric: p50, p95, p99, p99.9

Example results:
p50 = 200ms   (median user sees 200ms)
p95 = 800ms   (95% of users see ≤ 800ms)
p99 = 2000ms  (99% of users see ≤ 2000ms)
p99.9 = 5000ms (99.9% of users see ≤ 5000ms)

Why percentiles matter:
Average = 400ms (misleading, hides slow users)
p99 = 2000ms (shows what slow users experience)

Slow users are:
- Frustrated (5 second wait feels infinite)
- Likely to abandon (go to competitor)
- More valuable (often repeat customers)

Therefore: p99 is more important than average
```

### Setting Response Time Targets

```
SLA (Service Level Agreement) targets:

E-commerce checkout:
p50: ≤ 500ms (normal user)
p99: ≤ 2000ms (slowest users, still acceptable)

API service:
p50: ≤ 100ms (fast response)
p99: ≤ 500ms (strict latency requirement)

Analytics dashboard:
p50: ≤ 10 seconds (background process, OK to wait)
p99: ≤ 60 seconds (user willing to wait for report)

Science: Response times > 1 second feel "slow" to users
         Response times > 5 seconds feel "broken"
         Choose targets accordingly
```

## Part 3: Throughput KPIs

How many requests can system handle?

### Requests Per Second (RPS)

```
Metric: Requests per second (req/sec)

Example:
- Current throughput: 5000 req/sec
- Peak throughput needed: 10000 req/sec
- Performance gap: 2x

What causes throughput limits?
1. CPU maxed at 100% (add more servers)
2. Database overloaded (optimize queries, add replicas)
3. Memory exhausted (GC pauses, heap too small)
4. Connection pool exhausted (increase pool size)

To increase throughput:
1. Identify bottleneck (CPU? DB? Memory? Connections?)
2. Fix that specific bottleneck
3. Measure improvement
4. Repeat
```

### Requests Until Saturation

```
Metric: At what load does system degrade?

Test results:
Load: 1000 users → p99 = 100ms (good)
Load: 5000 users → p99 = 300ms (acceptable)
Load: 10000 users → p99 = 1000ms (approaching limit)
Load: 15000 users → p99 = 5000ms (system struggling)
Load: 20000 users → error rate = 5% (system broken)

Saturation point: 15,000 concurrent users
Target: 20,000 users with p99 ≤ 2000ms

Action: Optimize or add capacity to handle 20,000 users
```

## Part 4: Error Rate KPI

System reliability.

### Error Rate Percentage

```
Metric: % of requests that fail (return error)

Example:
- Total requests: 1,000,000
- Successful: 999,900
- Failed: 100
- Error rate: 0.01%

Targets:
- Production: < 0.1% (SLA)
- Acceptable: < 1% (temporary degradation)
- Bad: > 1% (system is broken)

Common errors:
- 500 Internal Server Error: Application bug
- 503 Service Unavailable: Overloaded or down
- 408 Request Timeout: Taking too long
- 429 Too Many Requests: Rate limited
```

### Error Analysis

```
When errors happen, investigate:
- What error? (5xx, 4xx, timeout, connection refused)
- Which service? (Payment, Inventory, User)
- When? (Peak time? Random? After deployment?)
- Why? (Overload, bug, dependency down)

Example investigation:
T=10:00: Error rate jumps from 0% to 5%
Error: "503 Service Unavailable" from Payment Service
Action: Check Payment Service logs
Finding: Database connection pool exhausted
Solution: Increase pool size from 20 to 50
Result: Error rate back to 0%
```

## Part 5: Resource Utilization KPIs

How efficiently are you using hardware?

### CPU Usage

```
Metric: % of CPU utilized

Target: < 80% (headroom for spikes)

Scenarios:
- 30% CPU: Underutilized (waste money on unused servers)
- 60% CPU: Healthy (can handle 33% traffic spike)
- 85% CPU: Approaching limit
- 95% CPU: No headroom (any spike causes errors)
- 100% CPU: Maxed out (requests queued, performance degrades)

If CPU > 85%:
- Option 1: Add more servers (scale horizontally)
- Option 2: Optimize code (scale vertically)
- Usually both
```

### Memory Usage

```
Metric: % of heap used (Java apps)

Target: < 80% (headroom before GC)

Scenarios:
- 40% heap: Healthy
- 70% heap: OK
- 85% heap: Approaching limit
- 95% heap: GC runs constantly (pauses)
- 100% heap: OutOfMemory crash

Heap too small:
- Frequent GC pauses (system stalls)
- Solution: Increase -Xmx parameter

Heap too large:
- GC pauses longer (more to clean)
- Wasted money on unused memory
- Solution: Size appropriately
```

### Disk I/O

```
Metric: Disk operations per second (IOPS)

Target: < 80% of disk capacity

Example SSD specifications:
- SSD: 10,000 IOPS (fast, expensive)
- HDD: 100 IOPS (slow, cheap)

If at 80% capacity:
- 8000 IOPS out of 10,000
- Any spike hits ceiling
- Response time degradation
- Solution: Add disk, optimize queries
```

## Part 6: Availability KPIs

How often is the system up and working?

### Uptime Percentage

```
Metric: % of time system is available

Examples:
- 99.9% uptime = 43 minutes downtime per month
- 99.99% uptime = 4 minutes downtime per month
- 99.999% uptime = 26 seconds downtime per month (Five Nines)

Business impact of downtime:
- E-commerce: $1000+ per minute lost
- SaaS: $10+ per customer per minute lost
- Social media: Reputation damage

For critical systems: Target 99.99% or higher
For internal tools: 99% is acceptable
```

### Mean Time To Recovery (MTTR)

```
Metric: How long until service is restored after failure?

Example:
Incident: Payment Service goes down
T=10:00: Alert fires
T=10:02: Team notices alert
T=10:05: Root cause identified
T=10:10: Fix deployed
T=10:11: Service restored

MTTR = 11 minutes

To improve MTTR:
1. Faster alerting (automated, not email)
2. Better runbooks (documented steps)
3. Automated recovery (restart service automatically)
4. Good monitoring (know what's happening)
```

## Part 7: Business KPIs

Metrics that matter to product and business teams.

### Conversion Rate

```
Metric: % of visitors that make a purchase

Example:
- Visitors: 100,000
- Purchases: 2,000
- Conversion: 2%

Performance impact:
- If checkout takes 5 seconds: Conversion = 2%
- If checkout takes 2 seconds: Conversion = 2.5% (25% improvement)
- If checkout takes 10 seconds: Conversion = 1.5% (25% decrease)

Every 100ms slower checkout = -0.5% conversion
Link performance to revenue:
- $1M revenue × 0.5% improvement = $5,000 more revenue
- This justifies performance optimization budget
```

### User Retention

```
Metric: % of users that return next day/week/month

Example:
- Users day 1: 100,000
- Return day 2: 60,000
- Retention: 60%

Performance impact:
Slow system → Frustrated users → Less likely to return

Improve retention:
1. Fast response times (users stay engaged)
2. Reliable (no errors, users trust you)
3. Smooth (no lag, no freezes)
```

## Part 8: Interview Q&A

**Q: What's more important: average response time or p99?**

A: P99 is more important.
- Average hides outliers (some users see 10x slowness)
- P99 shows worst-case (what slow users experience)
- Slow users are frustrated and likely to leave
- Slow users are often your best customers

**Q: How do you set response time targets?**

A: Based on:
1. User expectations (e-commerce: 2-3 sec, API: 100-500ms)
2. Business impact (every 100ms = revenue impact)
3. Technical feasibility (can we achieve it?)
4. Competitive benchmark (what do competitors do?)

**Q: Why is p99 latency more actionable than average?**

A: Because:
- Average = 200ms (sounds good)
- P99 = 5000ms (tells real story)
- You fix p99 (make worst case acceptable)
- Average improves automatically

**Q: How do you tie performance to business metrics?**

A: Connect the dots:
- Performance improvement: Checkout time 5s → 2s
- User behavior: Conversion rate +0.5%
- Business impact: Revenue +$50K/month
- Justifies performance team budget

**Q: What's acceptable error rate for production?**

A: Depends on system:
- Financial transactions: 0% (zero tolerance)
- E-commerce: < 0.1% (SLA)
- Internal tools: < 1%
- Batch jobs: < 5% (retryable)

**Q: How often should you measure KPIs?**

A: Real-time monitoring:
- Response time: Every request (or sampled)
- Error rate: Every request
- CPU/Memory: Every 10 seconds
- Business KPIs: Daily/Weekly
- Uptime: Always (automated monitoring)

Continuous measurement enables quick response to problems.
