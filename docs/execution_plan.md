# Test Execution Plans: From Strategy to Results

## Part 1: What is a Test Execution Plan?

Before you can run a performance test, you need a **test execution plan**. This is your blueprint—it tells you:
- **Who** will run the test
- **What** scenarios you'll test
- **When** you'll run them
- **Where** (which environment)
- **How** to collect results
- **What happens if something goes wrong**

Think of it like a cooking recipe. You don't just throw ingredients in a pot and hope. You plan:
1. What ingredients you need (tools, environments)
2. The exact steps (test scenarios)
3. The temperature and timing (load profile)
4. What to do if it burns (escalation procedures)
5. How to know when it's done (success criteria)

Without an execution plan, your test will be chaotic, results will be unreliable, and you won't know if you actually tested what you needed to test.

## Part 2: Components of a Test Execution Plan

### 2.1 Test Objectives (Why Are We Testing?)

You must answer: **What question does this test answer?**

**Bad objective:** "Run performance test on the checkout flow"
- Too vague. What do you actually want to know?

**Good objective:** "Verify that checkout can handle 1000 concurrent users with response time ≤ 2 seconds at p99 during peak holiday traffic"
- Specific number (1000 concurrent)
- Specific metric (response time)
- Specific percentile (p99, not average)
- Specific time (peak traffic pattern)
- Success criteria (≤ 2 seconds)

### 2.2 Test Scope (What Are We Testing?)

Define **exactly** which features and scenarios are included:

**Example scope for payment service:**
```
✓ In Scope:
  - Credit card payments
  - Debit card payments
  - PayPal integration
  - Refund processing
  - Concurrency up to 1000 users

✗ Out of Scope:
  - Bank transfer payments (separate service)
  - Admin dashboard (different tier)
  - Cryptocurrency (future phase)
  - Mobile app (tested separately)
```

Why? Because if you test everything at once, you won't know what caused the failure.

### 2.3 Load Profile (How Do We Ramp Up?)

A load profile defines **how traffic increases over time**. This is critical because:
- If you instantly hit 1000 users, you're testing "connection setup under extreme load" (not realistic)
- If you ramp up slowly, you're testing "system behavior during peak ramp" (realistic)

**Example load profile for checkout:**

```
Phase 1: Ramp (0-5 minutes)
  - Start at 100 users
  - Increase by 200 users every 60 seconds
  - End at 1000 users after 5 minutes
  
Phase 2: Sustain (5-10 minutes)
  - Hold at 1000 concurrent users
  - Normal behavior (some users finishing, new ones arriving)
  
Phase 3: Cool Down (10-12 minutes)
  - Reduce by 200 users every 60 seconds
  - Back to 100 users
  
Total: 12 minutes
```

**Why this matters:**
- **Ramp phase tests:** Can the system handle traffic ramping? Do connection pools initialize correctly?
- **Sustain phase tests:** Can the system maintain steady state? Are there memory leaks?
- **Cool down phase tests:** Does the system gracefully shut down? Are there resource leaks?

### 2.4 Test Scenarios (What User Journeys?)

Define the **realistic user behaviors** you're testing:

**Example for e-commerce checkout:**

```
Scenario 1: Browse + Checkout (70% of traffic)
  1. GET /homepage
  2. GET /products?category=shoes
  3. GET /product/123 (product details)
  4. POST /cart/add (add to cart)
  5. GET /cart (view cart)
  6. POST /checkout/start
  7. POST /checkout/payment
  8. GET /order/confirmation

Scenario 2: Repeat Purchase (20% of traffic)
  1. POST /login
  2. GET /orders/history
  3. GET /order/456/reorder
  4. POST /checkout/payment
  5. GET /order/confirmation

Scenario 3: Cart Abandonment (10% of traffic)
  1. GET /products
  2. POST /cart/add
  3. GET /cart
  4. [Abandon - user leaves]
```

**Why percentages matter:**
- If you test equal traffic on all scenarios, you won't see real problems
- Real users have patterns: most browse then checkout; few repeat purchase; many abandon carts
- Your test should mirror reality

### 2.5 Environment Configuration

Be explicit about **which environment** you're testing:

```
Environment: Staging (Production-like)

Infrastructure:
  - Database: PostgreSQL 13 (8 CPU, 32 GB RAM)
  - Cache: Redis 6.0 (2 GB)
  - Web Servers: 4 instances × (2 CPU, 4 GB RAM)
  - Load Balancer: HAProxy

Network:
  - Simulated latency: 50ms (realistic internet)
  - Bandwidth: 1 Gbps (matches production)
  - Geographic origin: US East (matches expected users)

Data:
  - 1 million products in database
  - 100,000 existing users (realistic size)
  - Cache pre-warmed with 80% hit rate (typical production state)
```

Why all this detail? Because:
- Testing on a single tiny instance won't show real bottlenecks
- Testing with empty cache won't show real behavior
- Testing from data center (0ms latency) won't show real user experience

### 2.6 Thresholds & Acceptance Criteria

Define **when the test passes or fails**:

```
PASS Criteria (All must be true):
  - p99 response time ≤ 2000 ms
  - p95 response time ≤ 1000 ms
  - p50 response time ≤ 500 ms
  - Error rate < 0.1%
  - CPU utilization < 80%
  - Memory utilization < 85%
  - No OutOfMemory exceptions

FAIL Criteria (Any one triggers failure):
  - p99 response time > 3000 ms
  - Error rate > 1%
  - Any 5xx server errors
  - Heap grows beyond 90% of max
  - Database connection pool exhausted
```

**Important:** These are not arbitrary numbers. They come from:
- SLA requirements (what you promised customers)
- Competitor benchmarks
- Cost constraints (more servers = more money)
- User expectations (users abandon slow sites)

## Part 3: Pre-Test Checklist

Before you run the test, verify:

### Infrastructure Ready
```
☐ All servers are healthy and running
☐ Database is at expected size (not aged data)
☐ Cache is warmed up (or explicitly empty if testing cache miss)
☐ Load balancer is configured correctly
☐ Monitoring and alerting are enabled
☐ Disk space is > 20% free (room for logs)
☐ No background jobs running (would interfere)
```

### Test Tools Ready
```
☐ Load testing tool is installed and working
☐ Test scripts are written and validated (test your load tester!)
☐ All test data is prepared
☐ Think-time is configured (not too fast, not too slow)
☐ Ramp-up curves are correct
☐ Report generation is tested
```

### Monitoring Ready
```
☐ Metrics collection is enabled (CPU, memory, I/O, network)
☐ Application logs are streaming to log aggregator
☐ GC logging is enabled (for Java apps)
☐ Slow query logging is enabled (for databases)
☐ Alert thresholds are set
☐ You have runbook access (how to respond to alerts)
```

### Team Ready
```
☐ Test owner is identified
☐ On-call person is available (if test fails, who fixes it?)
☐ Escalation path is clear (who do you call if bad things happen?)
☐ All team members understand success criteria
☐ Database backups are fresh (ready to restore if test corrupts data)
```

## Part 4: Running the Test

### Start the Test
1. **Record baseline metrics** (before load)
   - CPU: 5%
   - Memory: 40%
   - Response time: 100ms
   
2. **Start monitoring** - watch in real-time:
   ```
   Every 30 seconds, check:
   - Are requests completing?
   - Are error rates increasing?
   - Is CPU rising normally (not spiking)?
   - Are database connections increasing (not stuck)?
   ```

3. **Start load generation** - ramp according to plan

### During the Test
```
Minute 0-2: Ramp phase
  - CPU rises from 5% to 25% (normal)
  - Response time rises from 100ms to 300ms (expected)
  - Some connections being established (normal)
  - Error rate: 0% (good)
  → Continue ramp
  
Minute 2-5: Ramp continues
  - CPU rises from 25% to 60% (normal)
  - Response time: 400-600ms (good)
  - Database connections stable at 45/100 (good)
  - Error rate: 0% (good)
  → Continue ramp
  
Minute 5: Full load reached (1000 users)
  - CPU: 72%
  - Response time: p99 = 1200ms (under 2000ms threshold)
  - Memory: 68% (under 85% threshold)
  - Error rate: 0.02% (under 0.1% threshold)
  - All looks good
  → Hold for sustain phase
  
Minute 5-10: Sustain phase
  - Monitor for memory leaks (heap growing?)
  - Monitor for deadlocks (requests hanging?)
  - Monitor for degradation (response times increasing?)
  → Everything stable
  
Minute 10-12: Cool down phase
  - Users disconnecting
  - CPU dropping
  - Memory releasing (is heap dropping? or leak?)
  → Test ends
```

### If Something Goes Wrong

**If error rate spikes to 5% at minute 7:**
- Immediately look at logs: What error?
- Is it timeout errors? Database errors? Application errors?
- Decision: Stop test and investigate? Or continue?
- Document: "At minute 7, error rate increased. Root cause: connection pool exhausted"

**If CPU suddenly jumps to 95% at minute 6:**
- Look at process profiling: What's consuming CPU?
- Is it garbage collection? User code? Database?
- Decision: Stop and investigate? Or throttle load?
- Document: Exact time, behavior, what you did

## Part 5: Collecting Results

### Metrics to Collect

**Response Time Distribution:**
```
Percentile    Time (ms)    Interpretation
50th (p50)    450          Median user experience
90th (p90)    850          Most users see this or better
95th (p95)    1100         Only 5% of users are slower
99th (p99)    1850         Only 1% of users are slower
99.9th        2300         Only 0.1% of users are slower
Max           5200         Slowest user
```

Why percentiles instead of average?
- Average = 520ms (misleading - hides slow users)
- p99 = 1850ms (shows what slowest 1% experience)

**Throughput:**
```
Requests per second: 890 req/s
This means: With 1000 concurrent users, the system processes 890 requests per second
```

**Resource Utilization:**
```
CPU:     72% average (stayed under 80% threshold)
Memory:  68% average (stayed under 85% threshold)
Disk I/O: 300 IOPS (normal for database)
Network:  45 Mbps (out of 1000 Mbps available)
```

**Error Analysis:**
```
Total requests: 534,000
Successful: 533,900 (99.98%)
Failed: 100 (0.02%)

Error breakdown:
- 408 Request Timeout: 45 (database slow at minute 7)
- 503 Service Unavailable: 30 (brief deployment at minute 8)
- 500 Internal Error: 25 (unknown - needs investigation)
```

### Real Example Report

```
TEST REPORT: Checkout Service - Holiday Load Test

Objective: Verify checkout can handle 1000 concurrent users
Environment: Staging (production-like)
Date: December 20, 2024
Duration: 12 minutes

RESULTS:
✓ PASS - All acceptance criteria met

Response Times:
  p50:   450ms  (target: any)
  p95:  1100ms  (target: any)
  p99:  1850ms  (target: ≤ 2000ms) ✓
  
Throughput:
  890 req/s sustained

Errors:
  0.02% error rate (target: <0.1%) ✓
  
Resources:
  CPU:    72% (target: <80%) ✓
  Memory: 68% (target: <85%) ✓

Conclusion:
System is ready for 1000 concurrent users during peak holiday traffic.
Recommend: Proceed to production deployment.

Recommendations for future:
1. Response time at p99 is close to threshold (1850ms of 2000ms limit)
   → Consider caching frequently accessed products
2. At minute 7, database showed slow queries (100ms+ response)
   → Need query optimization on product search
```

## Part 6: Common Mistakes in Test Execution

### Mistake 1: Not Validating the Load Tester Itself

You test the application, but who tests the load tester?

```
WRONG:
- Run load tester with 5000 threads
- Monitor application metrics
- Application looks fine
- Conclusion: System is fine

PROBLEM:
- Load tester was on same network as application
- Load tester was slow to connect
- Application never got 5000 actual requests
- You tested at 2000 req/s, not 5000 req/s

RIGHT:
- First, test the load tester in isolation
- Verify it can generate target load
- Check network bandwidth
- Verify application actually receives all requests
- Only then draw conclusions
```

### Mistake 2: Testing with Unrealistic Data

```
WRONG:
- Test with 100 products in database
- Response time: 50ms (great!)
- Deploy to production with 1M products
- Response time: 500ms (disaster!)

RIGHT:
- Test with production-scale data (1M products)
- Response time: 500ms (shows real behavior)
- Now you know what to optimize
```

### Mistake 3: Wrong Think Time

```
WRONG:
- Real users think for 30 seconds between requests
- Your test sends requests 100ms apart
- Application never sees real connection patterns
- Results are meaningless

RIGHT:
- Use realistic think time (30 seconds)
- Or adjust test to account for it
- Match what real users actually do
```

### Mistake 4: Not Baselining

```
WRONG:
- Run test, get p99 = 1500ms
- Claim victory ("under 2000ms!")
- No context on whether this is good or bad

RIGHT:
- Establish baseline first (maybe p99 = 300ms)
- Run test, get p99 = 1500ms
- Calculate degradation: 5x slower
- Now decide: is 5x acceptable?
```

## Part 7: Interview Q&A

**Q: How do you design a test execution plan?**
A: Start with the business question: What do we need to know? Then design the test backward from success criteria. Define load profile, scenarios, environment, thresholds. Validate with stakeholders before running. A good test execution plan prevents wasted time on tests that don't answer real questions.

**Q: What's the difference between load profile and test scenario?**
A: Load profile is the timeline—how traffic increases (ramp, sustain, cool down). Test scenario is the user journey—what buttons they click. You need both. A load profile without scenarios is empty. A scenario without a load profile is unmeasurable.

**Q: How do you know if your load test actually loaded the system?**
A: Validate:
1. Did the application receive the target load? Check request counts.
2. Did CPU/memory rise appropriately? Low CPU = system wasn't stressed.
3. Did response times increase? If constant, you didn't reach saturation.
4. Can you reproduce with half the load? If results are same, load wasn't real.

**Q: What do you do if the system fails during test?**
A: Stop, investigate, document, fix, retest. Don't ignore failures. If the test was valid and the system failed, that's real information. But first verify your test was actually valid (load tester working, data realistic, environment correct).

**Q: Why do percentiles matter more than average?**
A: Because 1% of users experiencing 10x slowness is a real business problem, even if average is great. P99 response time tells you what the slow users see. Average hides them.
