# Performance Testing: Deep Fundamentals (Senior Interview Level)

Performance testing is applied systems engineering. The goal is not "run JMeter and show graphs," but to answer **business-capacity questions** with **defensible evidence**:

- *Can we meet our SLOs at the forecast load?*
- *Where does the system saturate and why?*
- *How will it fail, and does it recover safely?*
- *What change will produce the best improvement per effort?*

## 1) What Performance Testing Actually Measures

You test the *end-to-end system* under a modeled workload and observe:

- **Latency distribution** (p50/p95/p99; tail behavior matters)
- **Throughput** (requests/sec, transactions/sec)
- **Errors** (functional + timeouts + partial failures)
- **Saturation signals** (CPU, memory, disk, network, thread pools, DB waits, queue depth)

**Senior mindset:** You're measuring a **queueing system**. When demand approaches capacity, queues form and tail latency increases sharply.

### Little's Law: The Interview Cheat-Code

For stable systems:

```
L = λ × W

L: average requests in system (in-flight)
λ: throughput (requests/sec)  
W: average time in system (sec)
```

**Senior usage:**

If throughput stays flat but W rises, then L must rise ⇒ **queues are growing**.

**Real example:**

```
Metric snapshot at time T:
- Throughput: 1000 RPS (stable)
- Average response time: 50ms (stable)
- In-flight requests: 1000 × 0.050 = 50 (okay)

One hour later:
- Throughput: 1000 RPS (still stable)
- Average response time: 200ms (increased!)
- In-flight requests: 1000 × 0.200 = 200 (queue building!)

→ Queues forming somewhere. Either thread pool saturation or dependency slow-down.
```

## 2) The Performance Testing Lifecycle

```
1. NFR / SLO discovery
   ↓
2. Workload modeling (open/closed, mix, seasonality)
   ↓
3. Test design (scenarios, data, pass/fail criteria)
   ↓
4. Environment readiness (parity + observability)
   ↓
5. Script + data engineering
   ↓
6. Baseline + calibrate
   ↓
7. Execute (ramp + steady + step)
   ↓
8. Analyze (correlate, attribute bottlenecks)
   ↓
9. Recommend + retest
```

## 3) NFR / SLO Discovery

Instead of "fast," get specific:

```
VAGUE:   "The system should be fast under load"
SPECIFIC: "p95 latency for checkout ≤ 800ms at 2,000 RPS"
          "error rate ≤ 0.1% under expected peak"
          "system must degrade gracefully at 2× peak (no cascading failures)"
```

**Key question:** What are the **business-critical transactions**?
- Top revenue paths
- Top endpoints by traffic
- Top support tickets

## 4) Workload Modeling

### 4.1 Workload Mix

Real systems are mixtures:

```
Example e-commerce system:
- Browse products (60%)
- Search (20%)
- Add to cart (10%)
- Checkout (8%)
- Admin operations (2%)

Each has different:
- Resource consumption
- Latency requirements
- Error handling needs
```

**Performance varies dramatically by mix.**

Example latencies:
```
Browse: p99 = 100ms (caching helps, mostly reads)
Search: p99 = 300ms (DB query, more work)
Checkout: p99 = 800ms (DB writes, payments, complex)
```

If you test "browse-only," you'll get false confidence. **Test the actual mix.**

### 4.2 Data Realism

Performance is data-dependent:

```
- Small tables: queries fast (fit in cache)
- Large tables: queries slow (disk IO)
- Skewed distributions (80% of traffic hits 20% of data)
- Index selectivity (how much data does query touch?)
```

**Example impact:**

```
Query: SELECT * FROM users WHERE user_id = ?

Small table (1 million rows):
- Index lookup: 1 disk IO
- Latency: 1ms

Large table (1 billion rows, not indexed properly):
- Index scan: 100,000+ disk IOs
- Latency: 100+ ms

Same query, same code, different data → 100x latency difference!
```

## 5) Environment Parity

Before trusting any numbers, confirm:

```
☐ Hardware class (CPU cores, memory, storage type)
☐ CPU/memory limits (if container)
☐ Network topology and latency
☐ Database size + stats + indexes + partitions
☐ Caches enabled/disabled as in production
☐ Config (timeouts, pool sizes, GC options, logging level)
☐ Load balancer / routing configuration
```

**If parity is impossible**, do *capacity modeling* explicitly and communicate uncertainty.

## 6) Observability: You Can't Tune What You Can't See

Minimum instrumentation:

```
Application:
  - Latency percentiles per endpoint (p50/p95/p99)
  - Error breakdown (type, endpoint)
  - Queue times (time waiting vs time working)

JVM:
  - GC pauses and frequency
  - Heap usage and old gen growth
  - Thread counts

Database:
  - Wait events (IO reads, locks, log sync)
  - Top SQL by elapsed time
  - Connection count and waits

Infrastructure:
  - CPU, memory, disk latency
  - Network I/O
  - Container throttling (if applicable)

Application tracing:
  - Critical path for slow requests
  - Fan-out and dependency latency
```

## 7) Test Execution: Senior Patterns

### 7.1 Baseline First

Start with low load to verify:

```
☐ Correctness of load test scripts (no functional errors)
☐ Load generator has headroom (not CPU-bound)
☐ Instrumentation works (metrics flowing)
☐ No client-side bottlenecks
```

### 7.2 Step-load to Find the Knee

Run steps (e.g., +200 RPS every 5–10 min) to observe:

```
Load progression:
100 RPS:  p99 = 50ms
200 RPS:  p99 = 55ms
300 RPS:  p99 = 60ms
400 RPS:  p99 = 70ms  ← latency rising
500 RPS:  p99 = 150ms ← sharp inflection (the "knee")
600 RPS:  p99 = 500ms (queues building)
700 RPS:  errors rising, throughput dropping
```

The "knee" at 500 RPS is your capacity limit. Beyond this, queues form and tail latency explodes.

### 7.3 Soak to Expose Leaks

Many failures only show after hours:

```
Monitor over 24 hours for:
- Memory growth (leaks)
- Connection pool exhaustion
- Disk growth (logs)
- Cache behavior changes
```

## 8) Analysis: Turn Graphs Into Causality

Senior analysis is correlation + attribution:

```
1. Identify when latency changed
2. Identify which component saturated at that time
3. Explain the mechanism (queue, lock, GC, IO)
4. Propose smallest change to remove bottleneck
```

**Example walkthrough:**

```
Timeline:
09:00 - Latency p99 = 100ms, normal
09:10 - Latency p99 = 500ms (spike!)

Check correlation:
- CPU: 95% at 09:10 (✓ correlates)
- GC: no pause at 09:10 (✗ not GC)
- DB wait: high log file sync at 09:10 (✓ correlates)

Hypothesis: DB commit path is slow

Verification:
- DB logs show redo log fsync time increased (✓ confirms)
- Storage team reports disk latency spike at 09:10 (✓ confirms)

Conclusion: Storage latency increased → DB commits slower → app thread waits → p99 increases
```

## 9) Reporting Template

```
Performance Test Report: [System] [Date]

OBJECTIVE:
  Validate p99 latency ≤ 500ms at 2000 RPS

WORKLOAD:
  - 2000 RPS
  - Mix: Browse 60%, Search 20%, Checkout 20%
  - Data volume: Production snapshot (1 billion users)

RESULTS:
  PASS / FAIL (against SLO)
  
  p50 latency: 45ms
  p95 latency: 200ms
  p99 latency: 480ms (✓ ≤ 500ms)
  Error rate: 0.05% (✓ < 0.1%)

BOTTLENECK:
  Primary: Database (SELECT queries taking 300+ ms)
  Evidence: DB wait time dominates request time in traces
  
RECOMMENDED FIXES:
  1. Add index on user_id (expected improvement: 50ms p99 reduction)
  2. Increase DB query cache (expected: 30ms reduction)
  
RISK:
  - Test did not include write-heavy scenarios
  - Cache behavior not validated at 24-hour mark (soak test needed)
```

## 10) Common Senior Pitfalls

- "We tested 1000 users" without specifying arrival model
- Trusting averages instead of tail latency (users feel p99)
- Ignoring warm-up (JIT, caches, connections)
- Load generator saturation mistaken for server saturation
- Not controlling variables between runs (noise)

## 11) Interview Q&A

**"How do you find capacity?"**

"Step-load to the knee, confirm steady-state, validate saturation signals via traces/waits, then build a capacity curve."

**"How do you defend results?"**

"Versioned configs, controlled runs, baseline, and causal evidence via correlation + traces."

**"What's your first action when p99 explodes?"**

"Check saturation + queue depth, then find critical path dependency via traces and wait analysis."
