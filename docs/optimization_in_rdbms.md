# RDBMS Optimization: Beyond Individual Queries

## Part 1: RDBMS Performance vs Database Performance

You've learned to optimize **individual queries**. But what about **the whole system**?

```
Optimized Query (runs in 100ms): Good
But:
- Runs 10,000 times per second: Bad (database overloaded)
- Blocks all other queries: Bad (contention)
- Locks tables for minutes: Bad (cascade failure)

True performance optimization is about:
1. Fast queries (query tuning)
2. + Efficient resource usage (connection pools, memory)
3. + Minimal contention (locking, transactions)
4. + Scalability (replication, sharding)
```

## Part 2: Connection Pool Sizing

Connections are expensive resources.

### Too Few Connections

```
Config: Pool size = 5
Users: 100 concurrent
Result:
  - First 5 users: Get connection, execute query
  - Users 6-100: Wait in queue (blocked)
  - Queue time: 5 seconds per user
  - Perceived latency: 5000ms + query time

Problem: Application is slow because connection queue is huge
```

### Too Many Connections

```
Config: Pool size = 1000
Users: 100 concurrent
Result:
  - All 100 users: Get connection immediately
  - But database maximum connections = 500
  - 500 connections idle in pool, consuming RAM
  - Other applications can't connect

Problem: Waste of resources, connections sitting idle
```

### Right Size

```
Formula: Pool size = (2 × CPU cores) + effective spindle count

Example on 8-core server with 1 SSD:
  Pool size = (2 × 8) + 1 = 17

Reasoning:
  - 2 × cores: Enough concurrent queries
  - + 1 for spindle: Account for I/O operations
  - Total: Not too many (waste), not too few (queue)
```

## Part 3: Transaction Length

Long transactions lock resources.

### Long Transaction (Bad)

```java
@Transactional
public void complexProcess() {
    // Start transaction
    User user = db.getUser(userId);  // Lock acquired
    user.setBalance(user.getBalance() - 100);
    
    // Call external API (might take 30 seconds!)
    Payment payment = externalPaymentService.charge(user, 100);
    
    // Still holding lock while external API runs
    user.setPaymentId(payment.id);
    db.save(user);  // Lock released
}

Problems:
- Lock held for 30 seconds
- Other transactions wanting user row must wait
- Cascade: 100 other users waiting on this one
- Database queue fills up
```

### Short Transaction (Good)

```java
public void complexProcess() {
    // Short transaction 1: Read and update
    @Transactional
    void updateUserBalance() {
        User user = db.getUser(userId);
        user.setBalance(user.getBalance() - 100);
        db.save(user);  // Lock released after 10ms
    }
    
    updateUserBalance();
    
    // No lock while calling external API
    Payment payment = externalPaymentService.charge(user, 100);
    
    // Short transaction 2: Update with payment ID
    @Transactional
    void updateWithPayment(Payment p) {
        User user = db.getUser(userId);
        user.setPaymentId(p.id);
        db.save(user);  // Lock released after 10ms
    }
    
    updateWithPayment(payment);
}

Benefits:
- Each transaction: 10ms lock
- No queue buildup
- Other transactions can proceed
- System stays responsive
```

## Part 4: Deadlock Prevention

When two transactions wait on each other's locks.

### Deadlock Scenario

```
Transaction A:
  1. Lock user row 123
  2. Update user balance
  3. Wait for order row 456 (held by B)
  
Transaction B:
  1. Lock order row 456
  2. Update order amount
  3. Wait for user row 123 (held by A)

Result: Both wait forever (deadlock)
```

### Solution: Lock Ordering

Always acquire locks in same order:

```java
// Transaction A (correct order)
lock(user 123)  // ← Always user first
lock(order 456) // ← Then order

// Transaction B (same order)
lock(user 123)  // ← Always user first
lock(order 456) // ← Then order

// No deadlock possible
```

### Detection and Recovery

```sql
-- Check for deadlocks (Oracle)
SELECT * FROM v$deadlock_graph;

-- If found, abort one transaction and retry
-- Database automatically handles in most cases
-- But better to prevent with lock ordering
```

## Part 5: Index Maintenance

Indexes degrade over time.

### Index Fragmentation

```
New index (0% fragmented):
[1] [2] [3] [4] [5]
Organized, sequential reads are fast

After many deletes:
[1]     [3]     [5]
Gaps left behind, reads are slow

Solution: Rebuild index
CREATE INDEX idx_rebuild;  -- Re-organize all data
```

### Too Many Indexes

```
Table has 20 indexes
Benefits:
  - Queries run faster (more index options)
  
Costs:
  - Every INSERT/UPDATE/DELETE must update 20 indexes
  - Maintenance is slow
  - Memory is used for 20 indexes
  
Better: 3-5 indexes (cover most common queries)
```

**Index Strategy:**
- Index columns that are in WHERE clause (filtering)
- Index columns that are in JOIN condition
- Don't index columns that are rarely used
- Monitor unused indexes: delete them

## Part 6: Replication and Failover

Read from replicas, write to primary.

```
Primary Database (Write)
  ↓ (Replication log)
  ├─ Replica 1 (Read)
  ├─ Replica 2 (Read)
  └─ Replica 3 (Read)

Benefits:
- Reads scale: 4x read capacity (3 replicas)
- HA: If primary fails, promote replica
- Backup: Replica is live backup

Example:
- Primary: 1000 writes/sec, 100 reads/sec
- With 3 replicas: 1000 writes/sec, 400 reads/sec (4x read capacity)
```

### Replication Lag

Problem: Replica is slightly behind primary.

```
T=0:  Write "John" to primary
T=1:  Replica hasn't replicated yet
      Read from replica: returns stale "Jane"
      User sees inconsistency

Solution: 
- Critical reads from primary
- Non-critical reads from replica
- Or accept eventual consistency
```

## Part 7: Sharding (Horizontal Scaling)

When database is too big, split into pieces.

```
Before sharding (1 database, 100 million users):
- Reads: 10,000 req/sec
- Database is at 95% capacity
- Can't add more

After sharding (4 databases, 25 million users each):
- Shard 1: Users 1-25M
- Shard 2: Users 25M-50M
- Shard 3: Users 50M-75M
- Shard 4: Users 75M-100M

- Each database: 2,500 req/sec (25% load)
- Total capacity: 40,000 req/sec (4x improvement)
```

**Sharding key (how to divide):**

```
Good: User ID
- Distribute evenly
- Queries typically by user
- Load balanced

Bad: Country
- USA: 90% of traffic (one shard overloaded)
- Nepal: 1% of traffic (one shard underutilized)
- Imbalanced
```

## Part 8: Monitoring RDBMS Health

Metrics that matter:

```
Connection Pool:
- Active connections: < 80% of max (good)
- Queue wait time: < 100ms (good)
- Connections exhausted: Alert

Query Performance:
- Slow query log: Monitor queries > 1000ms
- Query throughput: Consistent or trending down?
- Lock waits: < 1% of query time

Resource Utilization:
- CPU: < 80% (headroom for spikes)
- Memory: < 85% (can cache more)
- Disk I/O: < 80% (can handle more queries)
- Disk space: > 20% free (growing?)

Transaction Health:
- Uncommitted transactions: < 100 (long running?)
- Rollbacks: Should be rare
- Deadlocks: Should be 0
```

## Part 9: Interview Q&A

**Q: What's the right connection pool size?**

A: Formula: (2 × CPU cores) + effective spindle count
Example on 8-core server with 1 SSD: (2 × 8) + 1 = 17 connections
Too few: Queue forms, app is slow
Too many: Waste memory, still can't go faster

**Q: Why are long transactions bad?**

A: Because:
- Locks held longer = other queries wait longer
- Cascade: One slow transaction slows entire system
- Solution: Break into multiple short transactions

**Q: How do you prevent deadlocks?**

A: Acquire locks in consistent order:
- All transactions: lock user first, then order
- No circular waits = no deadlock

**Q: When should you use replication vs sharding?**

A:
- Replication: Scale reads (same data, multiple copies)
- Sharding: Scale writes (split data, multiple servers)
- Can use both: Multiple shards, each with replicas

**Q: How do you handle replication lag?**

A:
- Critical reads (login, payment): Read from primary
- Non-critical reads (analytics): Read from replica
- Or: Accept eventual consistency and retry

**Q: What's the most important database metric?**

A: Connection pool queue wait time.
If queue is growing, everything else becomes slow.
Optimize that first, before worrying about query speed.
