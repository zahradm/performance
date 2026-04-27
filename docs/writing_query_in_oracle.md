# Query Optimization in Oracle: From Slow to Fast

## Part 1: Why Queries Get Slow

90% of database performance problems are **bad queries**. Before blaming hardware, hardware, fix your SQL.

```
Slow Query Reality:
- User complains: "Report takes 5 minutes"
- DBA says: "Hardware is fine, look at your query"
- Developer: "But it worked last year!"
- Truth: Data grew 10x, now query is scanning 10M rows instead of 1M
```

## Part 2: Query Optimization Process

### Step 1: Establish Baseline

Before optimizing, measure how slow it is.

```sql
-- Run query with timing
SET TIMING ON
SELECT * FROM large_table WHERE id = 123;

Output:
Elapsed time: 5432 ms

-- Get execution plan
EXPLAIN PLAN FOR
SELECT * FROM large_table WHERE id = 123;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

Output:
|--  Full Table Scan (large_table)
|    Filter: id = 123
Rows selected: 1,000,000
```

**Insight:** Full table scan of 1M rows to find 1 row = bad

### Step 2: Add Index

```sql
-- Create index on the WHERE clause column
CREATE INDEX idx_large_table_id ON large_table(id);

-- Run again
SELECT * FROM large_table WHERE id = 123;

Elapsed time: 5 ms (1000x faster!)

-- Check plan
|--  Index Range Scan (idx_large_table_id)
|    Access: id = 123
Rows selected: 1
```

### Step 3: Look for N+1 Queries

```sql
SLOW (10,001 database hits):
-- Get all orders
SELECT * FROM orders;  -- 10,000 rows

-- In your application (pseudo-code):
for each order:
    SELECT * FROM items WHERE order_id = order.id;  -- 10,000 separate queries

FAST (2 database hits):
-- Get orders with items in one query
SELECT o.*, i.*
FROM orders o
JOIN items i ON o.id = i.order_id;

-- Even better if possible: pre-computed count
SELECT o.*, COUNT(i.id) as item_count
FROM orders o
LEFT JOIN items i ON o.id = i.order_id
GROUP BY o.id;
```

### Step 4: Index Selection Columns

Indexes should be on columns you filter by (WHERE clause).

```sql
SLOW:
SELECT * FROM transactions
WHERE customer_id = 456
  AND amount > 100;
-- Full table scan of all transactions

FAST:
CREATE INDEX idx_transactions_customer ON transactions(customer_id);
CREATE INDEX idx_transactions_amount ON transactions(amount);

SELECT * FROM transactions
WHERE customer_id = 456
  AND amount > 100;
-- Uses customer_id index, then filters by amount
```

**Composite Index (multiple columns):**

```sql
CREATE INDEX idx_composite ON transactions(customer_id, amount);

-- This is better than two separate indexes
-- Oracle can: Find customer_id=456, then check amount>100 in same index scan
```

### Step 5: Avoid Functions on Columns

Oracle can't use index if you put function on column:

```sql
SLOW (no index):
SELECT * FROM users
WHERE UPPER(name) = 'JOHN';
-- Oracle can't use index because function is involved

FAST (with function-based index):
CREATE INDEX idx_users_upper_name ON users(UPPER(name));
SELECT * FROM users
WHERE UPPER(name) = 'JOHN';
-- Now uses index

BEST (no function):
SELECT * FROM users
WHERE name = 'JOHN';
-- If you always store uppercase, no function needed
```

### Step 6: Use BETWEEN for Ranges

```sql
SLOWER:
SELECT * FROM transactions
WHERE transaction_date >= '2024-01-01'
  AND transaction_date < '2024-01-31';

FASTER:
SELECT * FROM transactions
WHERE transaction_date BETWEEN '2024-01-01' AND '2024-01-30';
```

## Part 3: Real Example: Slow Report Query

```sql
ORIGINAL (12 seconds):
SELECT u.name, COUNT(o.id) as orders, SUM(o.amount) as total
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_date >= '2023-01-01'
GROUP BY u.id, u.name
ORDER BY total DESC;

Analysis:
- Joins 500K users with 50M orders = 25 trillion combinations (worst case)
- No index on created_date
- No index on foreign key
- Sorting 500K results by total
```

**Fix 1: Add Indexes**

```sql
CREATE INDEX idx_users_created ON users(created_date);
CREATE INDEX idx_orders_user_id ON orders(user_id);

Query time: 12 sec → 3 sec
```

**Fix 2: Filter Early**

```sql
SELECT u.name, COUNT(o.id) as orders, SUM(o.amount) as total
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_date >= '2023-01-01'
  AND o.created_date >= '2023-01-01'  -- ← Also filter orders
GROUP BY u.id, u.name
ORDER BY total DESC;

CREATE INDEX idx_orders_created ON orders(created_date);

Query time: 3 sec → 1 sec
```

**Fix 3: Materialize View**

```sql
-- Pre-compute daily (when load is low)
CREATE MATERIALIZED VIEW user_order_summary AS
SELECT u.id, u.name, COUNT(o.id) as orders, SUM(o.amount) as total
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_date >= '2023-01-01'
GROUP BY u.id, u.name;

-- Query materialized view (instant)
SELECT * FROM user_order_summary
ORDER BY total DESC;

Query time: 1 sec → 50 ms
```

## Part 4: Partition Large Tables

When table is massive (100M+ rows), partitioning is essential.

```sql
CREATE TABLE transactions (
    id NUMBER,
    customer_id NUMBER,
    amount NUMBER,
    transaction_date DATE
)
PARTITION BY RANGE (transaction_date) (
    PARTITION trans_2023 VALUES LESS THAN (TO_DATE('2024-01-01')),
    PARTITION trans_2024 VALUES LESS THAN (TO_DATE('2025-01-01')),
    PARTITION trans_2025 VALUES LESS THAN (MAXVALUE)
);

-- Oracle automatically partitions data by year
-- Queries on 2024 only search 2024 partition (100x faster)

SELECT * FROM transactions
WHERE transaction_date >= '2024-01-01'
  AND transaction_date < '2024-12-31';
-- Searches only trans_2024 partition (much smaller)
```

## Part 5: Hints (Forcing Oracle's Hand)

Sometimes Oracle's optimizer makes wrong choice. Use hints to override:

```sql
-- Force using specific index
SELECT /*+ INDEX(large_table idx_large_table_id) */ *
FROM large_table
WHERE id = 123;

-- Force parallel execution
SELECT /*+ PARALLEL(4) */ *
FROM huge_table;

-- Force hash join
SELECT /*+ USE_HASH(u o) */ u.name, o.amount
FROM users u
JOIN orders o ON u.id = o.user_id;

-- Force nested loop join
SELECT /*+ USE_NL(u o) */ u.name, o.amount
FROM users u
JOIN orders o ON u.id = o.user_id;
```

**Caution:** Hints are dangerous. If data distribution changes, hint might make query slow again. Use only when absolutely necessary.

## Part 6: Statistics

Oracle optimizer needs statistics to make good decisions.

```
Problem: Statistics are stale
- Table has 1M rows
- Oracle thinks it has 10K rows
- Optimizer chooses wrong plan

Solution: Gather statistics

EXEC DBMS_STATS.GATHER_TABLE_STATS('SCHEMA_NAME', 'TABLE_NAME');

// Or automatic (daily)
EXEC DBMS_SCHEDULER.CREATE_JOB(job_name => 'gather_stats',
    job_type => 'PLSQL_BLOCK',
    job_action => 'BEGIN DBMS_STATS.GATHER_DICTIONARY_STATS; END;',
    repeat_interval => 'FREQ=DAILY;BYHOUR=2');
```

## Part 7: Common Mistakes

### Mistake 1: SELECT * (get all columns)

```sql
WRONG:
SELECT *
FROM large_table;
-- Gets 50 columns, but you only need 3

RIGHT:
SELECT id, name, email
FROM large_table;
-- Much less data transferred, faster
```

### Mistake 2: DISTINCT without WHERE

```sql
WRONG:
SELECT DISTINCT name
FROM transactions
WHERE amount > 100;
-- Scans all rows, then removes duplicates

RIGHT:
SELECT DISTINCT name
FROM transactions
WHERE amount > 100
GROUP BY name;
-- Groups first, then returns unique names
```

### Mistake 3: Implicit Type Conversion

```sql
WRONG:
SELECT * FROM users
WHERE id = '456';
-- Oracle converts string to number (can't use index efficiently)

RIGHT:
SELECT * FROM users
WHERE id = 456;
-- Direct number comparison, index works
```

### Mistake 4: Subqueries Instead of Join

```sql
SLOW (10,001 subqueries):
SELECT *
FROM orders o
WHERE o.user_id IN (
    SELECT id FROM users WHERE country = 'USA'
);

FAST (1 join):
SELECT o.*
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.country = 'USA';
```

## Part 8: Monitoring Query Performance

```sql
-- Find slowest queries
SELECT sql_text, elapsed_time_secs, executions
FROM v$sql
WHERE elapsed_time_secs > 10
ORDER BY elapsed_time_secs DESC;

-- Monitor real-time query
SELECT * FROM v$session
WHERE sql_exec_start IS NOT NULL;

-- Check wait events for specific query
SELECT event, count(*)
FROM v$session_wait
WHERE sid = 123
GROUP BY event;
```

## Part 9: Interview Q&A

**Q: Why is index on WHERE clause important?**

A: Because Oracle can go directly to rows matching WHERE condition, instead of scanning all rows. Index is like a book index - you don't read every page to find "Chapter 5", you look it up in the index.

**Q: What's the difference between UNIQUE index and PRIMARY KEY?**

A: 
- PRIMARY KEY: Unique + NOT NULL, identifies each row
- UNIQUE index: Allows one NULL, but all other values must be unique
- Both prevent duplicate values

**Q: Why would query performance change without code changes?**

A: 
- Data grew 10x (old indexes less effective)
- New complex query competing for resources
- Statistics are stale (optimizer makes wrong choice)
- Parameter change (buffer cache size reduced)

**Q: How do you optimize a slow JOIN?**

A: 
1. Add index on foreign key column
2. Make sure statistics are fresh
3. Use EXPLAIN PLAN to see join method
4. Consider partitioning large tables
5. Pre-compute if possible (materialized view)

**Q: When should you use partitioning?**

A: When table is huge (100M+ rows) and most queries access subset:
- Queries filter by date range
- Queries need only recent data
- Table is growing rapidly
- Full table scans take too long

