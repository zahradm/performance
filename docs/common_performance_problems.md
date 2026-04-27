# Common Performance Problems: Diagnosis & Solutions

## Part 1: Why Do Applications Get Slow?

Performance problems don't appear randomly. They follow patterns. Learn these patterns, and you can:
- **Diagnose** problems 10x faster (know where to look)
- **Prevent** problems before they happen
- **Fix** them with targeted solutions (not random tuning)

Every slow application falls into one of these categories:
1. **CPU bottleneck** - computation is slow
2. **Memory bottleneck** - garbage collection, memory leaks
3. **I/O bottleneck** - database, disk, network are slow
4. **Lock bottleneck** - threads waiting on each other
5. **Misconfiguration** - settings don't match workload

Let's go through each.

## Part 2: CPU Bottleneck

### What It Looks Like

```
Symptoms:
- CPU usage: 85-99% (maxed out)
- Response time: Increasing with traffic
- Throughput: Hits ceiling (can't go higher)
- Heap memory: Stable (not the problem)
- Database: Fast responses (not the problem)

Real example:
- At 500 concurrent users: p99 = 300ms
- At 700 concurrent users: p99 = 800ms
- At 1000 concurrent users: p99 = 2500ms
- Meanwhile, database is idle. Why? CPU is maxed.
```

### Root Causes

**1. Algorithm is inefficient**

```java
// SLOW: O(n²) algorithm
public List<User> findDuplicates(List<User> users) {
    List<User> duplicates = new ArrayList<>();
    for (int i = 0; i < users.size(); i++) {
        for (int j = i + 1; j < users.size(); j++) {
            if (users.get(i).getEmail().equals(users.get(j).getEmail())) {
                duplicates.add(users.get(j));
            }
        }
    }
    return duplicates;
}

// Problem:
// 1000 users: 1000 * 1000 = 1,000,000 comparisons
// Takes 100ms CPU per request
// At 100 req/s, CPU = 10,000 ms/s = 100% of 1 CPU
// System can only handle 10 req/s total
```

**Fix: Use efficient data structure (O(n log n))**

```java
public List<User> findDuplicates(List<User> users) {
    Set<String> seen = new HashSet<>();
    List<User> duplicates = new ArrayList<>();
    for (User user : users) {
        if (!seen.add(user.getEmail())) {
            duplicates.add(user);
        }
    }
    return duplicates;
}

// Better:
// 1000 users: 1000 comparisons (hash lookup is O(1))
// Takes 1ms CPU per request
// At 100 req/s, CPU = 100 ms/s = 10% of 1 CPU
// System can handle 1000+ req/s
```

**2. Unoptimized loops in hot path**

```java
// SLOW: Allocating objects in loop
List<Product> products = getProductList();
List<Integer> prices = new ArrayList<>();
for (Product p : products) {
    prices.add(p.getPrice());  // ← ArrayList.add allocates memory
}

// This runs 1000 times per request, 100 users = 100K allocations/sec
// GC runs constantly → CPU waste
```

**Fix: Pre-allocate or stream**

```java
// FAST: Pre-allocate size
List<Integer> prices = new ArrayList<>(products.size());
for (Product p : products) {
    prices.add(p.getPrice());
}

// Or use streams (better):
List<Integer> prices = products.stream()
    .map(Product::getPrice)
    .collect(Collectors.toList());
```

**3. Missing CPU cache optimization**

```java
// SLOW: Cache misses (random memory access)
int[][] matrix = new int[10000][10000];
for (int i = 0; i < 10000; i++) {
    for (int j = 0; j < 10000; j++) {
        sum += matrix[i][j];  // ← Jumping all over memory
    }
}

// Each access misses CPU cache
// CPU can be 10-50x slower on cache miss
// Actual latency: 200 CPU cycles per value
```

**Fix: Sequential access (cache-friendly)**

```java
// FAST: Sequential memory access (cache hits)
for (int j = 0; j < 10000; j++) {
    for (int i = 0; i < 10000; i++) {
        sum += matrix[i][j];  // ← Walking through memory sequentially
    }
}

// CPU cache predicts next access
// Latency: 3 CPU cycles per value
// 50x+ speedup just from cache efficiency
```

### How to Diagnose CPU Bottleneck

**Step 1: Confirm it's CPU**

```bash
# During test, check CPU usage
top

# Output:
%CPU  PROCESS
85%   java -Xmx4g MyApp
10%   postgres
 3%   redis

# Yes, Java is maxed at 85% - this is CPU bottleneck
```

**Step 2: Where in the code?**

Use profiler (shows which methods consume CPU):

```
Top CPU consumers:
1. findDuplicates()  - 25% of CPU (the O(n²) algorithm)
2. encryptPassword() - 18% of CPU (slow crypto)
3. serialializeJSON() - 15% of CPU (inefficient JSON encoding)
4. calculateDiscount() - 12% of CPU (complex math loop)

Action: Fix the top 3, get 60% CPU back
```

**Step 3: Can you parallelize?**

```java
// Currently running on 1 core
// Can you use multiple cores?

// BEFORE: Single-threaded
List<User> users = getAllUsers();  // 10,000 users
List<String> processed = new ArrayList<>();
for (User u : users) {
    processed.add(expensiveComputation(u));
}

// AFTER: Parallel
List<String> processed = users.parallelStream()
    .map(this::expensiveComputation)
    .collect(Collectors.toList());

// On 4-core server:
// Single-threaded: 4000ms
// Parallel: 1000ms (4x speedup)
```

### Solutions for CPU Bottleneck

| Problem | Solution | Speedup |
|---------|----------|---------|
| O(n²) algorithm | Use hash set | 100x |
| Frequent allocations | Pre-allocate or reuse | 10x |
| Inefficient serialization | Use binary format | 5x |
| Single-threaded | Parallelize | 4x (on 4 cores) |
| Slow cryptography | Pre-compute or cache results | 50x |
| Unoptimized loops | Cache-friendly access pattern | 10x |

## Part 3: Memory Bottleneck

### What It Looks Like

```
Symptoms:
- GC pauses: 100-500ms happening every 30 seconds
- Response time: Stable until GC pause, then spike to 5000ms
- Memory usage: Heap fills to 95%, then drops to 50% (GC)
- Throughput: Smooth except during GC (drops by 80%)
- CPU: Normal (40%), but GC threads max out (90%)

Real example:
- p99 response time without GC: 200ms
- p99 response time including GC: 5000ms
- GC is causing 25x slowness 1% of the time
```

### Root Causes

**1. Heap too small**

```
Scenario: Application allocates 1GB/sec of garbage
Current heap: 2GB
GC cycle: Must collect 2GB before heap fills
Time to collect 2GB: 300ms pause
Frequency: Every 2 seconds

Problem: 300ms pause every 2 seconds = 15% of request time lost to GC
```

**Fix: Increase heap size**

```
New heap: 8GB
GC cycle: Must collect 8GB before heap fills
Time to collect 8GB: 300ms pause (same)
Frequency: Every 8 seconds

Result: 300ms pause every 8 seconds = 3.75% of time lost to GC
       (4x improvement)
```

**2. Too many short-lived objects**

```java
// WRONG: 1000 objects created per request
String result = "";
for (int i = 0; i < 1000; i++) {
    result = result + "item-" + i + ",";  // ← Creates new String each iteration
}

// At 100 req/s: 100,000 objects/sec of garbage
// GC must clean this constantly
// Pause time: 50-100ms every second
```

**Fix: Use StringBuilder**

```java
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++) {
    sb.append("item-").append(i).append(",");
}
String result = sb.toString();  // ← Only 1 String created

// At 100 req/s: 100 objects/sec of garbage
// GC barely notices this
// Pause time: near zero
```

**3. Memory leak (objects never freed)**

```java
// WRONG: Objects stay in memory forever
Map<String, byte[]> cache = new HashMap<>();
public void cacheData(String key, byte[] data) {
    cache.put(key, data);
    // Never remove old entries
}

// Week 1: 10 MB cache
// Week 2: 100 MB cache (20x growth)
// Week 3: 500 MB cache
// Week 4: Out of memory crash

// No GC can help because objects are still referenced
```

**Fix: Bounds on cache**

```java
// Option 1: LRU Cache (evict oldest)
Map<String, byte[]> cache = new LinkedHashMap<String, byte[]>(16, 0.75f, true) {
    protected boolean removeEldestEntry(Map.Entry eldest) {
        return size() > 1000;  // Keep max 1000 entries
    }
};

// Option 2: Expiring cache
ConcurrentHashMap<String, CachedValue> cache = new ConcurrentHashMap<>();
// Evict entries older than 1 hour

// Result: Stable memory usage at 100 MB, no growth
```

### How to Diagnose Memory Bottleneck

**Step 1: Identify GC overhead**

```bash
# Enable GC logging
java -Xmx4g -Xlog:gc:gc.log MyApp

# Analyze logs
GC Log Output:
[GC (G1 Evacuation Pause) 1234 ms]
[GC (G1 Evacuation Pause) 1256 ms]
[GC (G1 Evacuation Pause) 1268 ms]

# If you see 1000+ ms pauses every 30 seconds: memory problem
```

**Step 2: Find memory leaks**

```bash
# Take heap dump during test
jcmd <pid> GC.heap_dump heapdump.hprof

# Analyze with heap analyzer
# Shows: Which objects are eating memory?
# Example finding:
# byte[][] array from cache - 2.5 GB (leak!)
# Fix: Remove cache or add bounds
```

**Step 3: Calculate allocation rate**

```
Allocation rate = Memory allocated per second
= (Heap size - free memory after GC) / Time between GCs
= (2GB - 0.2GB) / 2 seconds
= 900 MB/sec

This means: Application creates 900 MB of garbage every second
At 100 req/s: Each request creates 9 MB of garbage
This is excessive. Typical: 100-500 KB per request
```

### Solutions for Memory Bottleneck

| Problem | Solution | Improvement |
|---------|----------|-------------|
| Heap too small | Increase to match allocation rate | 4x less pause time |
| Short-lived objects | Use StringBuilder, pools | 100x less GC |
| Memory leak | Add cache bounds, listeners | Stable memory |
| Large allocations | Object pooling, streaming | 50x less GC |

## Part 4: I/O Bottleneck

### What It Looks Like

```
Symptoms:
- CPU usage: 10% (not working hard)
- Disk I/O: 95% (maxed)
- Database response time: 500-2000ms
- Application waiting for database: 90% of request time
- Memory: Fine
- Network: Fine

Example:
- Query takes 200ms (why?)
- Network roundtrip: 1ms
- Network transfer: 1ms
- Database processing: 198ms
- Disk I/O: 150ms (most of the time!)
```

### Root Causes

**1. Inefficient database queries**

```sql
-- SLOW: Full table scan on 1 million users
SELECT * FROM users WHERE email = 'john@example.com'

-- Database must read all 1 million rows to find one
-- Cost: 500ms (read from disk)
```

**Fix: Add index**

```sql
CREATE INDEX idx_users_email ON users(email);

-- Now: Database uses index, reads exactly 1 row
-- Cost: 1ms (random disk access to index, then data)
-- Speedup: 500x
```

**2. N+1 queries (1 query becomes 1000 queries)**

```java
// WRONG: 1 query + N additional queries
List<Orders> orders = db.query("SELECT * FROM orders LIMIT 100");

for (Order order : orders) {
    List<Items> items = db.query("SELECT * FROM items WHERE order_id = " + order.id);
    // ↑ This runs 100 times!
    order.setItems(items);
}

// Total: 101 database queries
// Time: 100 * 200ms = 20 seconds
```

**Fix: Join in one query**

```java
// RIGHT: 1 query with JOIN
List<Orders> orders = db.query(
    "SELECT o.*, i.* FROM orders o " +
    "LEFT JOIN items i ON o.id = i.order_id " +
    "LIMIT 100"
);

// Total: 1 database query
// Time: 200ms (10x faster)
```

**3. Inefficient data transfer**

```
WRONG: Request "SELECT * FROM users"
- Returns all 50 columns
- Each user: 2KB JSON
- 10,000 users: 20 MB transferred
- Network time: 2 seconds (on 10Mbps network)

RIGHT: Request "SELECT id, email FROM users"
- Returns only 2 columns
- Each user: 50 bytes JSON
- 10,000 users: 500 KB transferred
- Network time: 50ms (40x faster)
```

### How to Diagnose I/O Bottleneck

**Step 1: Check what's slow**

```
During load test, measure:
- Response time: 500ms
- Network latency: 5ms
- Database response: 300ms
- Application processing: 195ms

Conclusion: Database is the bottleneck (300ms of 500ms)
```

**Step 2: Profile database queries**

```sql
-- Enable slow query log
SET slow_query_log = 1;
SET long_query_time = 0.1;  -- log queries > 100ms

-- Results show:
-- Query 1: "SELECT * FROM users WHERE id = ?" - 2ms (good)
-- Query 2: "SELECT * FROM orders" - 450ms (bad! no WHERE clause)
-- Query 3: "SELECT * FROM items" - 1200ms (very bad! full table scan)

Action: Add WHERE clause to Query 2 and 3
```

**Step 3: Check query execution plans**

```sql
EXPLAIN SELECT * FROM items WHERE order_id = 123;

Output:
→ Index Scan using idx_items_order_id
  Index Cond: (order_id = 123)
  Actual rows returned: 15

Good: Using index, returns only 15 rows

vs. Bad query:

EXPLAIN SELECT * FROM items;

Output:
→ Seq Scan on items
  Actual rows returned: 5,000,000

Bad: Full table scan of 5 million rows
```

### Solutions for I/O Bottleneck

| Problem | Solution | Speedup |
|---------|----------|---------|
| No database index | Add index on WHERE column | 100x |
| N+1 queries | Use JOIN instead of loop | 10x |
| Full table scan | Add index, use WHERE | 50x |
| Transferring too much data | Select only needed columns | 5x |
| Repeated same query | Add caching | 100x |
| Slow network | Use compression, reduce payload | 3x |

## Part 5: Lock Bottleneck

### What It Looks Like

```
Symptoms:
- CPU usage: 40% (not maxed)
- Threads: Many stuck (waiting)
- Response time: Increases dramatically with concurrency
- Throughput: Doesn't scale (10 threads = 10x fast, but 100 threads = 5x faster)
- Deadlocks or lock timeouts appearing

Example:
- 10 concurrent users: p99 = 200ms
- 100 concurrent users: p99 = 5000ms
- 1000 concurrent users: p99 = 20000ms
- Threads are waiting on locks, not running
```

### Root Causes

**1. Global lock on hot resource**

```java
// WRONG: All threads queue for this lock
synchronized Map<String, User> userCache = new HashMap<>();

public User getUser(String id) {
    synchronized (userCache) {  // ← All threads wait here
        if (userCache.containsKey(id)) {
            return userCache.get(id);
        }
        User user = db.getUser(id);
        userCache.put(id, user);
        return user;
    }
}

// 100 concurrent users:
// Average wait time for lock: 50ms (if lock held 200ms)
// Request time: 200ms actual work + 50ms waiting = 250ms
```

**Fix: Fine-grained locks or concurrent structure**

```java
// RIGHT: Use ConcurrentHashMap (no global lock)
ConcurrentHashMap<String, User> userCache = new ConcurrentHashMap<>();

public User getUser(String id) {
    // No explicit lock needed; ConcurrentHashMap handles it
    // Only locks the bucket being accessed (multiple threads can read different buckets)
    if (userCache.containsKey(id)) {
        return userCache.get(id);
    }
    User user = db.getUser(id);
    userCache.put(id, user);
    return user;
}

// 100 concurrent users:
// Lock contention: minimal (each thread accesses different bucket)
// Request time: 200ms actual work + 0-1ms waiting
```

**2. Long-held locks**

```java
// WRONG: Hold lock while doing I/O (database query)
synchronized void processOrder(Order order) {
    order.validate();  // 1ms (fast)
    order.setStatus("PROCESSING");  // 1ms (fast)
    db.saveOrder(order);  // ← 200ms database I/O (while lock held!)
    order.notify(customer);  // ← 500ms email send (while lock held!)
}

// If 10 threads try this:
// Thread 1: Gets lock, holds for 701ms
// Threads 2-10: Wait in queue
// Total queue time: 9 * 701ms = 6309ms of wasted time!
```

**Fix: Hold lock only for shared data**

```java
void processOrder(Order order) {
    // Shared data
    synchronized (order) {
        order.validate();
        order.setStatus("PROCESSING");
    }
    
    // No lock needed for I/O
    db.saveOrder(order);  // 200ms, no lock
    order.notify(customer);  // 500ms, no lock
}

// If 10 threads:
// Thread 1: Lock 10ms, then I/O 700ms (not holding lock)
// Threads 2-10: While 1 is doing I/O, they can each get lock and do their I/O
// Total time: 10 * 710ms = 7100ms
// With lock: 7100ms
// Without lock: 710ms (10x faster!)
```

**3. Deadlock (threads waiting on each other)**

```java
// WRONG: Lock order not consistent
Thread A:
    synchronized(resource1) {
        synchronized(resource2) {
            // do work
        }
    }

Thread B:
    synchronized(resource2) {  // ← Different order!
        synchronized(resource1) {
            // do work
        }
    }

// Scenario:
// Thread A: Gets lock on resource1
// Thread B: Gets lock on resource2
// Thread A: Waits for resource2 (held by B)
// Thread B: Waits for resource1 (held by A)
// Deadlock: Both wait forever
```

**Fix: Always acquire in same order**

```java
Thread A:
    synchronized(resource1) {
        synchronized(resource2) {
            // do work
        }
    }

Thread B:
    synchronized(resource1) {  // ← Same order
        synchronized(resource2) {
            // do work
        }
    }

// No deadlock possible
```

### Solutions for Lock Bottleneck

| Problem | Solution | Impact |
|---------|----------|--------|
| Global lock | Use ConcurrentHashMap or fine-grained locks | 10x-100x |
| Long-held locks | Lock only critical section | 10x |
| Deadlock | Consistent lock ordering | Stability |
| Lock contention | Reduce contention (sharding, replication) | 5x-50x |

## Part 6: Misconfiguration Bottleneck

### Connection Pools Too Small

```
Configuration: Connection pool size = 10

At 100 concurrent users:
- First 10 users: Get database connections
- Users 11-100: Queue for connection (wait 100-500ms per request)

Fix: Increase to 100 connections
- All 100 users: Get immediate connection
- No queue
- Response time drops from 500ms to 100ms
```

### GC Configuration Wrong

```
Wrong: Heap = 512MB, Application needs 2GB of live objects
- GC pauses constantly (memory pressure)
- Pause time: 500ms every 10 seconds
- Effective throughput: 50% (half time in GC)

Right: Heap = 4GB
- GC pauses occasionally
- Pause time: 200ms every 60 seconds
- Effective throughput: 99.7%
```

### Thread Pool Too Small

```
Web server config: Thread pool = 5

At 100 concurrent users:
- First 5 users: Get request handled immediately
- Users 6-100: Queue in web server
- Even with instant processing: 5000ms wait per request

Fix: Thread pool = 100
- All 100 users handled concurrently
- No queue
- Response time is just processing time
```

## Part 7: Interview Q&A

**Q: How do you diagnose which bottleneck a slow system has?**

A: Follow this checklist in order:
1. CPU usage - if 85%+, CPU bottleneck
2. GC logs - if frequent long pauses, memory bottleneck
3. Database response time - if slow, I/O bottleneck
4. Thread wait analysis - if many waiting, lock bottleneck
5. Configuration - if connections/threads are limited, misconfiguration

Once identified, measure the bottleneck to quantify impact, then target that specific area.

**Q: What's the difference between optimizing for throughput vs latency?**

A: 
- Throughput: Can you handle more requests per second? (Parallel processing, batching)
- Latency: How fast does each request complete? (Caching, algorithm optimization)

Sometimes they conflict. A cache improves latency but might reduce throughput if serialized. A thread pool improves throughput but can increase latency (queueing).

**Q: Why did increasing heap size fix the performance problem?**

A: Larger heap means:
- GC happens less frequently (more space to allocate)
- Each GC pause is same duration (must collect same amount)
- But overall pause time is lower (lower frequency)

Example: 
- Small heap (2GB): 500ms pause every 5 seconds = 10% pause time
- Large heap (8GB): 500ms pause every 20 seconds = 2.5% pause time
- Same pause duration, but happens less often
