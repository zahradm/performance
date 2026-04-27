# Types of Performance Tests (Senior Engineer Guide)

The senior question is not "load test vs stress test" but:
- *What risk are we trying to reduce?*
- *What decision will this test enable?*

Each test type is a tool. Choose based on the business decision you need.

## Test Type → Decision Map

| Test | Primary Question | Business Decision | Success Criteria |
|---|---|---|---|
| Load | Can we meet SLO at forecast load? | Safe to scale traffic? | p99 ≤ SLO at 1× peak |
| Capacity | What is max sustainable throughput? | How much traffic can we handle? | p99 ≤ SLO at highest sustainable RPS |
| Stress | How does it fail and recover? | Safe failure mode? | No data loss, recovery when load drops |
| Spike | Can it absorb sudden bursts? | Can we survive traffic surges? | No cascading failures |
| Soak | Does performance degrade over time? | Safe for 24/7 operation? | Metrics stable after 48 hours |
| Volume | Does data growth break performance? | When will we hit scaling limits? | p99 acceptable at 2× data volume |

## 1) Load Testing

**Goal:** Validate SLOs at expected load.

**Method:**
```
Start at baseline (e.g., 100 RPS)
Ramp to target load (e.g., 2000 RPS) over 5-10 minutes
Sustain for 30+ minutes (until steady-state)
```

**Steady-state means:**
- Metrics have stabilized (no climbing/falling trends)
- GC pauses predictable
- Memory usage stable
- No new errors appearing

**Success criteria:**
```
✓ p95 latency ≤ X ms
✓ p99 latency ≤ Y ms
✓ Error rate ≤ Z%
✓ No cascading failures
```

## 2) Capacity Testing (Find the Knee)

**Goal:** Find maximum sustainable throughput under an SLO.

**Method:**
```
Step load up gradually (e.g., +200 RPS every 10 minutes)
Observe where p95/p99 bends sharply (queueing starts)
That's your capacity limit
```

**Graphical pattern:**

```
Latency vs Load

        p99
        ↑
        |     /
        |    /
        |   /
        |  / (flat region - healthy)
        | /
     100|/ (knee - capacity boundary)
        |\
        | \___________
        |
        +---+---+---+--- RPS →
          100 200 300 400
            (knee at ~350 RPS)
```

**Why it matters:**
```
If knee is at 350 RPS with 30% safety margin:
Safe operating point = 350 × 0.7 = 245 RPS
Beyond this → latency explodes
```

## 3) Stress Testing

**Goal:** Force failure to learn failure mode and recovery.

**Method:**
```
1. Find capacity (say, 350 RPS)
2. Push 20-30% beyond (400-450 RPS)
3. Keep load there for 10-15 minutes
4. Drop load back to normal
5. Observe recovery
```

**Senior focus:**
- What does "break" mean? (timeouts? errors? data loss?)
- Does it recover automatically when load drops?
- Does it fail safely (protect DB, avoid retry storms)?
- Are there cascading failures?

**Example failure modes:**

```
BAD: Under stress, DB connections exhaust, app hangs, doesn't recover
GOOD: Under stress, app returns 503 errors gracefully, circuit breaker opens, 
      when load drops circuit breaker closes and system recovers
```

## 4) Spike Testing

**Goal:** Validate burst handling (sudden traffic surges).

**Method:**
```
Steady state at 100 RPS
Sudden spike: jump to 500 RPS
Maintain spike for 2-5 minutes
Drop back to 100 RPS
```

**Measures:**
```
- Queue buildup: how fast does queue grow?
- Queue drain time: after spike ends, how long to clear queue?
- User impact: if queue = 100 requests, slowest users wait longest
```

**Real example:**
```
Social media: celebrity tweets, 50,000 → 500,000 RPS instantly
If system has:
- Slow queue drain (10 minutes to clear backlog)
- Users waiting in queue see old data (stale)
- Timeouts after 30 seconds in queue

Result: Cascading timeouts, retry storm, worse load
```

## 5) Soak (Endurance) Testing

**Goal:** Detect time-based degradation.

**Method:**
```
Moderate load (say, 200 RPS)
Run continuously for 24-48 hours
Monitor for trends
```

**What you're hunting:**
- Memory leaks (heap slope trending up)
- Connection leaks (DB connections growing)
- Cache churn and eviction patterns
- Periodic batch jobs creating spikes
- Log growth and disk fill

**Example leak discovery:**

```
Time   Heap       Old Gen     Trend
00:00  1.0 GB     200 MB      ↑
04:00  1.5 GB     400 MB      ↑↑ (climbing)
08:00  2.0 GB     650 MB      ↑↑↑
12:00  2.5 GB     900 MB      ↑↑↑↑ (clear leak!)
16:00  3.0 GB     1200 MB     OOM risk
```

## 6) Volume Testing

**Goal:** Ensure performance at future data sizes.

**Method:**
```
Define target data volume (e.g., 1 year of data, 10× current size)
Load database with that volume
Run load test with realistic query patterns
```

**What changes with data size:**

```
Small table (1 million rows):
- Index fits in cache
- Query: 1 disk read
- Latency: 1 ms

Large table (1 billion rows):
- Index partially cached, query needs multiple reads
- Query: 50 disk reads  
- Latency: 50 ms

Even larger table (10 billion rows, not indexed):
- Query does full table scan
- Query: 1,000,000 disk reads
- Latency: seconds

Same query, different scale → different bottlenecks!
```

## 7) Scalability Testing

**Goal:** Understand how system scales with resources.

**Method:**
```
Run load test on different configurations:
- 1 server, 4 CPU
- 2 servers, 4 CPU each
- 4 servers, 4 CPU each
- 8 servers, 4 CPU each

Measure throughput at same latency SLO
```

**Expected vs Actual:**

```
Expected (perfect linear scaling):
1 server @ 400 RPS
2 servers @ 800 RPS
4 servers @ 1600 RPS
8 servers @ 3200 RPS

Actual (always degrades):
1 server @ 400 RPS
2 servers @ 750 RPS (87.5% of expected)
4 servers @ 1400 RPS (87.5% of expected)
8 servers @ 2700 RPS (84% of expected)

Degradation means:
- Contention for shared resources
- DB becomes bottleneck
- Network saturation
- Coordination overhead
```

## Interview Q&A

**"Which test type would you run first?"**

"A load test for the top revenue journey using production-like data. Then capacity to find limits, then stress/spike/soak based on risk."

**"What makes a performance test invalid?"**

"Unrealistic workload mix, wrong data volume, uncontrolled environment (config changes between runs), missing critical dependencies (always test with actual DB/cache)."

**"When do you scale?"**

"Only after I confirm the bottleneck is resource limits (CPU maxed, memory full), not contention or dependencies. Scaling the wrong dimension is waste."
