# Oracle Architecture: Understanding the Database Engine

## Part 1: What is Oracle?

Oracle is a **relational database** that stores data in tables and enforces relationships between them.

But Oracle is also a **complex engine** with many moving parts:
- Storage engine (how data is physically stored on disk)
- Query optimizer (how to execute queries efficiently)
- Transaction engine (ACID guarantees)
- Memory caches (buffers, shared pool)

This guide explains **how Oracle works inside**, so you understand:
- Why queries are slow (and how to fix them)
- How to configure Oracle for your workload
- What metrics matter (and what to ignore)

## Part 2: Oracle Architecture Overview

```
Client Application (e.g., Java, Python)
         ↓
    SQL Query: "SELECT * FROM users WHERE id = 123"
         ↓
    ─────────────────────────────────────
    Oracle Database Instance
    ─────────────────────────────────────
    
    [Parser] - Is the SQL valid?
         ↓
    [Optimizer] - What's the best way to execute this?
         ↓
    [Executor] - Run the optimal plan
         ↓
    [Buffer Cache] - Data from disk (fast RAM)
    [Redo Log] - Write-ahead logging (durability)
    [Data Files] - Physical storage on disk
         ↓
    Result Set: [User record]
         ↓
    Client receives: {"id": 123, "name": "John"}
```

## Part 3: Oracle Memory Architecture (SGA and PGA)

Oracle uses two main memory areas to manage performance and transactions:

### 3.0 SGA (System Global Area)

**SGA** is the shared memory used by the entire Oracle instance. All processes connect to it.

```
SGA (Shared, allocated at Oracle startup):
├── Database Buffer Cache (DB_CACHE_SIZE)
├── Shared Pool
│   ├── Library Cache (SQL statements, execution plans)
│   ├── Dictionary Cache (table/column metadata)
│   └── Result Cache (query results)
├── Redo Log Buffer (REDO_LOG_BUFFER_SIZE)
├── Large Pool (optional, for parallel queries)
└── Streams Pool (optional, for Oracle Streams)

Total SGA = SGA_TARGET (typical: 25-75% of available RAM)

Example on 32GB server:
  SGA_TARGET = 16 GB
  └── DB_CACHE_SIZE = 8 GB (caches data blocks)
  └── SHARED_POOL_SIZE = 4 GB (SQL, plans, metadata)
  └── REDO_LOG_BUFFER = 512 MB
  └── LARGE_POOL = 256 MB
  └── STREAMS_POOL = 256 MB
```

**Database Buffer Cache (Most Important)**

The buffer cache is where Oracle keeps frequently accessed data blocks in RAM.

```
Query: "SELECT * FROM users WHERE id = 123"

Step 1: Check if block is in buffer cache
  ├─ HIT: Return from RAM (0.1 ms) ✓ Fast
  └─ MISS: Read from disk (8 ms), then cache it

Buffer Cache Hit Ratio (measure of performance):
  = (Blocks read from cache) / (Total blocks read)
  
Target: > 90% (means 90% of reads avoid disk)
Bad: < 80% (too much disk I/O)

If hit ratio is low:
  → Increase DB_CACHE_SIZE
  → Or optimize queries to read fewer blocks
```

**Shared Pool**

The shared pool stores SQL statements and execution plans.

```
SQL Statement: "SELECT * FROM users WHERE id = ?"

Step 1: Parse query (check syntax, permissions)
  └─ Stored in Shared Pool → next time, reuse
  
Step 2: Generate execution plan
  └─ Stored in Shared Pool → next time, reuse

Benefits:
  - Don't re-parse same SQL (CPU savings)
  - Don't re-optimize (plan cache)
  
Problem: Shared Pool can fill up
  └─ Increase SHARED_POOL_SIZE
  └─ Or use bind variables (reuse plans)

Example - WITHOUT bind variables (bad):
  Query 1: SELECT * FROM users WHERE id = 1;
  Query 2: SELECT * FROM users WHERE id = 2;
  → 2 different SQL statements → 2 parsing operations
  
Example - WITH bind variables (good):
  Query: SELECT * FROM users WHERE id = ?;
  Execute with param 1
  Execute with param 2
  → Same SQL → parse once, reuse plan twice
```

**Redo Log Buffer**

Temporary storage for transaction changes before writing to disk.

```
When you INSERT a row:
  1. Change is written to Redo Log Buffer
  2. COMMIT occurs
  3. Buffer flushed to disk (Redo Log Files)
  4. Change is permanent

Without Redo Log Buffer:
  - Every INSERT = 1 disk write (slow)
  
With Redo Log Buffer:
  - Multiple INSERTs batched in RAM
  - One flush to disk = multiple commits (fast)
  
Typical size: REDO_LOG_BUFFER_SIZE = 10-100 MB
```

### 3.1 PGA (Program Global Area)

**PGA** is private memory for each Oracle process (session). Not shared.

```
Each user connection gets its own PGA:

User 1 Connection (PGA)
├── Sort Area (for ORDER BY)
├── Hash Area (for hash joins)
├── Session Variables
└── Stack Space

User 2 Connection (PGA)
├── Sort Area
├── Hash Area
├── Session Variables
└── Stack Space

Total PGA across all users = PGA_AGGREGATE_TARGET

Example on 32GB server:
  PGA_AGGREGATE_TARGET = 4 GB
  
  If 100 users connected:
  Average per user = 4 GB / 100 = 40 MB each

  If user sorts 100 MB data:
  └─ Uses 100 MB from their PGA allocation
  └─ When finished, memory returned to pool
```

**When PGA is Used**

```
Scenario 1: Sorting (ORDER BY)
  SELECT * FROM users ORDER BY name;
  
  Oracle needs to:
  1. Read all rows into PGA Sort Area
  2. Sort in memory
  3. Return sorted rows
  
  If data < PGA available:
    └─ Sort in memory (fast, 100 ms)
  
  If data > PGA available:
    └─ Spill to disk (slow, 5000 ms)
    └─ Reduce by: Add index on ORDER BY column

Scenario 2: Hash Join
  SELECT * FROM users u
  JOIN orders o ON u.id = o.user_id;
  
  Oracle needs to:
  1. Load one table into Hash Area (PGA)
  2. Scan second table, match using hash
  
  If both tables fit in PGA:
    └─ Fast (10x faster than nested loop)
  
  If tables > PGA available:
    └─ Spill to disk (slow)
    └─ Reduce by: Increase PGA_AGGREGATE_TARGET
```

**Difference: SGA vs PGA**

```
Feature          | SGA                  | PGA
─────────────────────────────────────────────────
Allocation       | At instance startup  | Per connection
Visibility       | Shared by all users  | Private per user
Size             | SGA_TARGET           | PGA_AGGREGATE_TARGET
Contains         | Data blocks, SQL     | Sorts, joins
Connection dies  | SGA stays            | PGA returned to pool

Example: 100 users, 32GB RAM
  SGA_TARGET = 16 GB (shared)
  PGA_AGGREGATE_TARGET = 4 GB (all 100 users share this)
  Remaining = 12 GB (OS, processes)
```

## Part 3: Key Oracle Concepts

### 3.1 Tablespaces

Physical division of storage. Think of it as "partitions of the hard drive."

```
Oracle Disk Storage:
├── SYSTEM tablespace (Oracle's own tables)
├── SYSAUX tablespace (indexes, statistics)
├── USERS tablespace (your tables)
└── TEMP tablespace (sorting, temporary data)

Example:
CREATE TABLESPACE user_data
  DATAFILE '/oracle/data/user_data.dbf' SIZE 1G;

CREATE TABLE users (
    id NUMBER,
    name VARCHAR2(100)
) TABLESPACE user_data;
```

### 3.2 Segments, Extents, Blocks

How Oracle physically stores tables:

```
Tablespace
  └─ Segment (e.g., "users" table)
      └─ Extent (chunk of space, e.g., 1 MB)
          └─ Block (smallest unit, e.g., 8 KB)

Example:
Table "users" = 50 MB
  = 50 Extents (1 MB each)
  = 6400 Blocks (8 KB each)

When you read row:
Oracle reads entire block (8 KB) into memory
Even if you only want 100 bytes of data
```

### 3.3 Buffer Cache

RAM used to cache frequently accessed data.

```
Without Buffer Cache:
Query: "SELECT * FROM users WHERE id = 123"
  → Read block from disk: 8 ms
  → Return row: 8 ms
  
Query runs again: Same 8 ms (no caching)
At 1000 queries/sec: 8,000 ms of disk I/O

With Buffer Cache (hit):
First query: Read from disk: 8 ms
Second query: Read from RAM: 0.1 ms (80x faster!)

Buffer Cache Configuration:
- SGA_TARGET = 4G (total memory to use)
- DB_CACHE_SIZE = 2G (for data blocks)
- SHARED_POOL_SIZE = 1G (for SQL statements, execution plans)
- REDO_LOG_BUFFER = 50M (for transaction log)
```

### 3.4 Redo Logs

Write-ahead logging ensures durability (ACID compliance).

```
When you INSERT a row:
1. Write to redo log: "INSERT user id=123"
2. Write to buffer cache: [user record]
3. Confirm to client: "Done"

Benefits:
- If database crashes, redo log has the change
- Can recover: Restart and replay redo log
- Customer doesn't lose data

Performance impact:
- Every INSERT/UPDATE requires disk write to redo log
- 5 ms disk I/O per transaction
- But concurrent inserts are batched (efficient)
```

## Part 4: Execution Plan

When you run a query, Oracle creates an **execution plan** (how to retrieve data).

### 4.1 Index vs Full Table Scan

```sql
-- Query: Find user by ID
SELECT * FROM users WHERE id = 123;

Execution Plan Option 1: Full Table Scan
  - Read all 1 million rows (block by block)
  - Check each row: "Is id = 123?"
  - Return matching row
  - Time: 500 ms

Execution Plan Option 2: Index Scan
  - Index on "id" has sorted: 1, 2, 3, ..., 123, ...
  - Go directly to block containing id=123
  - Return matching row
  - Time: 1 ms

Oracle chooses: Index Scan (500x faster!)
```

**When does Oracle use index?**

```sql
-- Index will be used (simple lookup)
SELECT * FROM users WHERE id = 123;

-- Index WILL NOT be used (OR forces scan)
SELECT * FROM users WHERE id = 123 OR id = 124 OR id = 125;
-- Solution: Use IN instead
SELECT * FROM users WHERE id IN (123, 124, 125);

-- Index WILL NOT be used (function on column)
SELECT * FROM users WHERE UPPER(name) = 'JOHN';
-- Solution: Create function-based index
CREATE INDEX idx_users_upper_name ON users(UPPER(name));

-- Index WILL NOT be used (comparison with function)
SELECT * FROM users WHERE id + 1 = 124;
-- Solution: Rewrite as: WHERE id = 123
SELECT * FROM users WHERE id = 123;
```

### 4.2 Join Strategies

When joining two tables, Oracle chooses a join method:

```sql
SELECT u.name, o.amount
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.country = 'USA';

Execution Plan Option 1: Nested Loop Join
  - For each user in "USA" (1000 rows)
    - Find matching orders (1 to 10 per user)
    - Return row
  - Time: 1000 index lookups = 1000 ms

Execution Plan Option 2: Hash Join
  - Load all USA users into memory hash table
  - Scan orders, for each order look up user in hash
  - Return row
  - Time: 1 full table scan = 100 ms

Execution Plan Option 3: Merge Join (if both sorted)
  - Both tables sorted by user_id
  - Walk through both simultaneously
  - Match rows as you go
  - Time: 50 ms (most efficient for large joins)

Oracle chooses based on:
- Table sizes
- Available memory
- Indexes available
- Data distribution
```

## Part 5: Wait Events

Queries are slow because Oracle waits for something. Identify what:

### 5.1 Common Wait Events

```
db file sequential read
- Oracle is reading a single block from disk
- Typical wait: 8 ms per read
- Problem: No index, full table scan
- Fix: Create index on WHERE clause

db file scattered read
- Oracle is reading multiple blocks from disk
- Typical wait: 5-50 ms
- Problem: Full table scan or index range scan
- Fix: Better index or filter earlier

lock
- Another transaction is blocking this one
- Typical wait: Until other transaction commits
- Problem: Long-running transaction
- Fix: Commit more frequently

direct path read
- Reading large amount of data bypassing buffer cache
- Typical: 1-10 ms
- Problem: Sorting huge result set
- Fix: Add index, reduce result set size

CPU time
- Oracle is computing something (not waiting)
- Problem: No I/O issue, just slow algorithm
- Fix: Optimize query or add indexes
```

### 5.2 AWR Report (Oracle Performance Diagnostic)

```
AWR = Automatic Workload Repository
Captures database statistics every 60 minutes

Key metrics:

Top Wait Events:
  1. db file sequential read - 45% of time
     Interpretation: Reading blocks from disk for indexed reads
  2. lock - 30% of time
     Interpretation: Two processes blocking each other
  3. CPU time - 25% of time
     Interpretation: Actual query processing

Top SQL Statements (by elapsed time):
  1. "SELECT * FROM transactions WHERE..." - 5000 ms
  2. "SELECT * FROM audit_log WHERE..." - 3000 ms
  3. "SELECT * FROM users WHERE..." - 2000 ms

Action:
- SQL 1: Create index or filter more
- SQL 2: Archive old audit data (don't scan millions of rows)
- SQL 3: Use cache instead of querying
```

## Part 6: Query Tuning Example

Real-world example: Slow query reporting system.

```sql
-- Slow Query (5 seconds)
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;

-- Analysis with EXPLAIN PLAN:
-- Full table scan of users (1M rows)
-- For each user, find orders (N+1 queries pattern)
-- Group by and count

-- Execution Plan:
|--  Sort (order by u.id)
|    |-- Hash Aggregate (group by)
|         |-- Hash Join
|              |-- Full Table Scan on users (1M rows)
|              |-- Full Table Scan on orders (50M rows)
```

**Problem:** Joining 1M users with 50M orders = massive intermediate result set.

**Solution 1: Pre-compute**

```sql
-- Create materialized view (cached result)
CREATE MATERIALIZED VIEW user_order_count AS
  SELECT u.id, u.name, COUNT(o.id) as order_count
  FROM users u
  LEFT JOIN orders o ON u.id = o.user_id
  GROUP BY u.id, u.name;

-- Refresh nightly when nobody is using it
EXEC DBMS_MVIEW.REFRESH('user_order_count');

-- Now query is instant (reads pre-computed view)
SELECT * FROM user_order_count;  -- 10 ms
```

**Solution 2: Add indexes**

```sql
-- Index on foreign key (speeds up join)
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Now Oracle can:
-- 1. Scan users table
-- 2. For each user, use index to find orders quickly
-- Time drops: 5000 ms → 800 ms (6x faster)
```

**Solution 3: Filter early**

```sql
-- Only show users with recent orders
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.created_date >= SYSDATE - 30  -- Last 30 days only
GROUP BY u.id, u.name;

-- Now:
-- Filter to 1M orders (instead of 50M)
-- Join 1M users with 1M orders (not 50M)
-- Time: 5000 ms → 200 ms (25x faster)
```

## Part 7: Configuration Tuning

### 7.1 Memory Configuration

```
Example: Poorly configured Oracle on 32 GB server

Current settings:
  SGA_TARGET = 4 GB (way too small!)
  PGA_AGGREGATE_TARGET = 500 MB

Result:
  - Buffer cache is tiny (constant misses)
  - Frequently read data is flushed from cache
  - Database hits disk 10x more than necessary
  - Slow queries everywhere

Better settings for 32 GB server:
  SGA_TARGET = 16 GB
  PGA_AGGREGATE_TARGET = 4 GB
  
Result:
  - Buffer cache holds frequently accessed data
  - Less disk I/O
  - Queries run 5-10x faster
```

### 7.2 Parallel Execution

```
Large query scanning 1 billion rows:

Serial (1 process):
  - Process reads all rows sequentially
  - Time: 30 minutes

Parallel (4 processes):
  - Process 1: Rows 1-250M
  - Process 2: Rows 250M-500M
  - Process 3: Rows 500M-750M
  - Process 4: Rows 750M-1B
  - Time: 8 minutes (4x faster)

Configuration:
  PARALLEL_MAX_SERVERS = 8 (allow up to 8 parallel processes)
  PARALLEL_MIN_TIME_THRESHOLD = 10 (use parallel for queries > 10 sec)
```

## Part 8: Interview Q&A

**Q: How do you diagnose why an Oracle query is slow?**

A: Follow this process:
1. Run EXPLAIN PLAN - see if using index or full scan
2. Check AWR report - what wait events are happening?
3. If db file sequential read - add indexes
4. If lock - check for blocking transactions
5. If CPU time - optimize SQL or add indexes
6. Measure before/after to confirm fix

**Q: What's the difference between EXPLAIN PLAN and actual execution?**

A: 
- EXPLAIN PLAN: Oracle's estimate of how to run query (may be wrong)
- Actual execution: What really happened (measured)
- Use DBMS_XPLAN to see actual vs estimated rows

**Q: Why would you use a hash join over a nested loop?**

A: 
- Nested loop: Fast for small result sets (< 1000 rows)
- Hash join: Fast for large result sets (millions of rows)
- Depends on table size and available memory

**Q: How do you prevent index fragmentation?**

A:
- Indexes fragment over time (deletions leave gaps)
- Rebuild index: ALTER INDEX idx_rebuild;
- Or let Oracle rebuild automatically during maintenance window

**Q: What's the difference between COMMIT and ROLLBACK?**

A:
- COMMIT: Persist all changes (write to redo log, permanent)
- ROLLBACK: Undo all changes since last commit
- Both are important for data consistency
