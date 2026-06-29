# MySQL Query Execution Lifecycle — Lab

This lab demonstrates how MySQL processes a SQL statement from the moment you hit Enter to the moment results appear on screen. You will observe each layer of the execution pipeline hands-on, using a realistic 1 million-row dataset.

## Learning Objectives

By the end of this lab you will be able to:

- Explain what happens when a query is executed
- Understand the role of the Parser, Optimizer, Executor, and InnoDB
- Understand how the optimizer selects an execution plan
- Understand why the optimizer sometimes makes surprising choices
- Observe the impact of single-column and composite indexes

---

## The Query Execution Lifecycle

Every query follows the same pipeline before a single row is returned:

```
Client
  ↓  sends SQL text over the connection
Parser
  ↓  tokenises and builds a parse tree
Authorization & Preprocessing
  ↓  checks privileges, resolves names
Optimizer
  ↓  evaluates access paths and picks the cheapest plan
Executor
  ↓  drives the plan step by step
Storage Engine API  (InnoDB)
  ↓  reads/writes actual data pages
Result returned to client
```

> **Key question to keep in mind throughout:** which stage is usually responsible for the most elapsed time?

---

## Setup

The database and tables are already created. Run this final step to populate 1 million rows:

```sql
USE mysql_lab;

CREATE TABLE IF NOT EXISTS numbers (n INT PRIMARY KEY);

INSERT INTO numbers (n)
WITH RECURSIVE seq(n) AS (
  SELECT 0
  UNION ALL
  SELECT n + 1 FROM seq WHERE n < 999
)
SELECT n FROM seq;

CREATE TABLE IF NOT EXISTS customers (
    customer_id INT AUTO_INCREMENT PRIMARY KEY,
    customer_name VARCHAR(100),
    email VARCHAR(150),
    country VARCHAR(50),
    status ENUM('ACTIVE','INACTIVE','SUSPENDED','PENDING','DELETED'),
    created_at DATE
);

INSERT INTO customers (
    customer_name, email, country, status, created_at
)
SELECT
    CONCAT('Customer_', a.n, '_', b.n),
    CONCAT('customer_', a.n, '_', b.n, '@example.com'),
    CASE MOD(a.n, 10)
        WHEN 0 THEN 'Kenya'       WHEN 1 THEN 'Uganda'
        WHEN 2 THEN 'Tanzania'    WHEN 3 THEN 'Rwanda'
        WHEN 4 THEN 'South Africa' WHEN 5 THEN 'Nigeria'
        WHEN 6 THEN 'Ghana'       WHEN 7 THEN 'Egypt'
        WHEN 8 THEN 'Ethiopia'    ELSE 'Zambia'
    END,
    CASE MOD(b.n, 5)
        WHEN 0 THEN 'ACTIVE'    WHEN 1 THEN 'INACTIVE'
        WHEN 2 THEN 'SUSPENDED' WHEN 3 THEN 'PENDING'
        ELSE 'DELETED'
    END,
    DATE_ADD('2020-01-01', INTERVAL FLOOR(RAND() * 2200) DAY)
FROM numbers a
CROSS JOIN numbers b;

SELECT COUNT(*) FROM customers;
-- Expected: 1,000,000
```

---

## Part 1 — The Parser

### What the parser does

The parser is the first component to see your SQL. It:

1. Runs the **lexer** — breaks the text into tokens (`SELECT`, `*`, `FROM`, `customers`, …)
2. Validates the token stream against MySQL's grammar rules
3. Builds a **parse tree** — an in-memory structure representing the query

If any step fails, MySQL returns `ERROR 1064` and stops. The optimizer, executor, and InnoDB are never called.

### Exercise

Run a valid query:

```sql
SELECT * FROM customers LIMIT 5;
```

Now introduce a deliberate typo:

```sql
SELECTE * FROM customers LIMIT 5;
```

Expected output:
```
ERROR 1064 (42000): You have an error in your SQL syntax ...
```

**Questions:**
- Which component rejected the second query?
- Did InnoDB ever see the second query?

**Answers:** The Parser. No — execution never reached InnoDB.

> **Takeaway:** A query must successfully parse before any other stage runs. Parse errors are cheap — no data is touched.

---

## Part 2 — The Optimizer and `EXPLAIN`

### What the optimizer does

After parsing and authorization, the optimizer receives the parse tree and must decide *how* to execute it — not *what* to return (that is fixed by SQL semantics). It:

- Looks at available indexes on every referenced table
- Estimates the **cost** of each candidate access path using table statistics
- Picks the plan with the lowest estimated cost
- Hands a concrete **query execution plan (QEP)** to the executor

`EXPLAIN` lets you see the plan the optimizer chose, without actually running the query.

### Exercise — primary key lookup

```sql
EXPLAIN SELECT * FROM customers WHERE customer_id = 500000;
```

Expected output:
```
+----+-------------+-----------+-------+---------------+---------+---------+-------+------+-------+
| id | select_type | table     | type  | possible_keys | key     | key_len | rows  | filt | Extra |
+----+-------------+-----------+-------+---------------+---------+---------+-------+------+-------+
|  1 | SIMPLE      | customers | const | PRIMARY       | PRIMARY | 8       |     1 |  100 | NULL  |
+----+-------------+-----------+-------+---------------+---------+---------+-------+------+-------+
```

### Reading the EXPLAIN columns

| Column | What it means |
|--------|--------------|
| `type` | The access method. `const` = at most one row (best possible). `ref` = index lookup. `range` = index range scan. `ALL` = full table scan (usually bad). |
| `possible_keys` | Indexes the optimizer considered |
| `key` | Index the optimizer actually chose |
| `key_len` | Bytes of the index used — longer is not always better |
| `rows` | Optimizer's *estimate* of rows to examine. Not exact — it is based on statistics. |
| `filtered` | Estimated % of examined rows that pass the `WHERE` filter |
| `Extra` | Important notes: `Using filesort`, `Using temporary`, `Using index` (covering index) |

**Questions:**
- Which index was used?
- How many rows does MySQL expect to examine?
- Which component generated this plan?

**Answers:** `PRIMARY`. `1`. The Optimizer.

> **Takeaway:** The optimizer chooses the access path before any row is read. Understanding its output is the core DBA skill for diagnosing slow queries.

---

## Part 3 — `EXPLAIN ANALYZE`: Estimated vs Actual

### What EXPLAIN ANALYZE adds

`EXPLAIN` shows the *plan*. `EXPLAIN ANALYZE` actually **executes** the query and annotates each plan node with real timing and row counts. This exposes the gap between what the optimizer predicted and what actually happened.

### Reading EXPLAIN ANALYZE output

```sql
EXPLAIN ANALYZE SELECT * FROM customers WHERE customer_id = 500000;
```

Output:
```
-> Rows fetched before execution  (cost=0..0 rows=1)
   (actual time=258e-6..476e-6 rows=1 loops=1)
```

Each node reports:

```
(cost=<startup>..<total>  rows=<estimated>)
(actual time=<first_row_ms>..<last_row_ms>  rows=<actual>  loops=<N>)
```

| Field | Meaning |
|-------|---------|
| `cost=0..0` | Optimizer's estimated cost units. `startup..total` — startup is cost to return the first row, total is cost for all rows. Lower is better. |
| `rows=1` (in cost) | Estimated rows — what the optimizer predicted |
| `actual time=0.000258..0.000476` | Real elapsed time in milliseconds. Format is `first_row..last_row`. |
| `rows=1` (in actual) | Real rows returned |
| `loops=1` | How many times this node ran. In a nested-loop join the inner node may loop thousands of times. |

> **Large gap between estimated and actual rows** means stale statistics. Fix with `ANALYZE TABLE customers;`.

---

## Part 4 — Full Table Scan (No Index)

```sql
EXPLAIN ANALYZE
SELECT * FROM customers WHERE country = 'Kenya';
```

Output:
```
-> Filter: (customers.country = 'Kenya')
     (cost=100964  rows=99482)
     (actual time=0.191..799  rows=100000  loops=1)
   -> Table scan on customers
        (cost=100964  rows=994824)
        (actual time=0.169..678  rows=1000000  loops=1)
```

### How to read this

The plan has two nodes — read bottom-up (inner node first):

1. **Table scan on customers** — InnoDB reads every row in the table. Cost `100964`, examined `1,000,000` rows, took ~678 ms to deliver the last row.
2. **Filter** — the executor applies `country = 'Kenya'` to each row. Only 100,000 pass. Cost unchanged (filtering is cheap), but we wasted effort on 900,000 rows.

**Questions:**
- What access type is used?
- Why didn't MySQL use an index?

**Answers:** `ALL` (full table scan). No index exists on `country`.

> **Takeaway:** Without an index, MySQL has no choice but to read every row and test the filter on each one.

---

## Part 5 — Single-Column Index

### Add the index

```sql
CREATE INDEX idx_country ON customers(country);

ANALYZE TABLE customers;
```

`ANALYZE TABLE` recomputes the index statistics that the optimizer uses for cost estimation. Always run it after creating an index or after a large bulk load.

### Re-run the same query

```sql
EXPLAIN ANALYZE
SELECT * FROM customers WHERE country = 'Kenya';
```

Output:
```
-> Index lookup on customers using idx_country (country='Kenya')
     (cost=24364  rows=199182)
     (actual time=4.46..656  rows=100000  loops=1)
```

### Before vs After

| Metric | Before (no index) | After (idx_country) |
|--------|-------------------|---------------------|
| Access type | `ALL` | `ref` (index lookup) |
| Estimated cost | 100,964 | 24,364 |
| Rows examined | 1,000,000 | ~100,000 |
| Actual time | ~799 ms | ~656 ms |

### Why the time saving is modest here

The index reduced the cost by **4×** — but wall-clock time dropped only modestly. Reason: `country = 'Kenya'` matches 100,000 rows (10% of the table). For each index entry, InnoDB must do a **bookmark lookup** — follow the secondary index entry back to the clustered (primary key) index to fetch the full row. With 100,000 bookmark lookups, the random I/O adds up. The index helps most when selectivity is high (few rows match).

> **Takeaway:** The optimizer evaluates cost, not just "is there an index?" For low-selectivity columns, a full scan can be cheaper than 100,000 random index lookups.

---

## Part 6 — Composite Index

### The problem: filter + sort

```sql
EXPLAIN ANALYZE
SELECT * FROM customers
WHERE status = 'ACTIVE'
ORDER BY created_at;
```

Output:
```
-> Sort: customers.created_at
     (cost=100964  rows=994824)
     (actual time=984..1014  rows=200000  loops=1)
   -> Filter: (customers.status = 'ACTIVE')
        (cost=100964  rows=994824)
        (actual time=0.121..800  rows=200000  loops=1)
      -> Table scan on customers
           (cost=100964  rows=994824)
           (actual time=0.114..673  rows=1000000  loops=1)
```

Three nodes: full scan → filter → sort. The sort node alone added ~30 ms *after* 1 second of scanning.

### Add a composite index

```sql
CREATE INDEX idx_status_created ON customers(status, created_at);

ANALYZE TABLE customers;
```

### Re-run

```sql
EXPLAIN ANALYZE
SELECT * FROM customers
WHERE status = 'ACTIVE'
ORDER BY created_at;
```

### Why column order matters in a composite index

`(status, created_at)` means InnoDB stores index entries sorted first by `status`, then by `created_at` within each status group. A query that filters on `status = 'ACTIVE'` and orders by `created_at` can:

1. Jump directly to the `ACTIVE` section of the index (no full scan)
2. Read those entries in `created_at` order (no sort step)

If you reversed the order to `(created_at, status)`, the `status` filter would not be able to use the leading edge of the index efficiently.

**Questions:**
- Is the `Sort` node still present?
- Why is one composite index better than two separate indexes here?

**Answers:** No — the index already delivers rows in the required order. Two separate indexes can only satisfy one operation each; the composite index satisfies both the `WHERE` filter and `ORDER BY` in a single pass.

> **Takeaway:** Design indexes around your query patterns, not just your columns. The column order in a composite index determines which queries benefit.

---

## Part 7 — Working With the Optimizer (Sargability)

### The problem: function on a column

```sql
EXPLAIN ANALYZE
SELECT * FROM customers WHERE YEAR(created_at) = 2025;
```

Output:
```
-> Filter: (year(customers.created_at) = 2025)
     (cost=100964  rows=994824)
     (actual time=0.0739..521  rows=165799  loops=1)
   -> Table scan on customers  ...
```

Full scan. Even though `created_at` is part of `idx_status_created`, the optimizer cannot use a range scan because `YEAR()` wraps the column. MySQL cannot invert `YEAR(x) = 2025` into a range on `x` without evaluating the function for every row.

This is called a **non-sargable** predicate (Search ARGument ABLE). A predicate is sargable if MySQL can translate it into an index range operation.

### The fix: isolate the bare column

```sql
EXPLAIN ANALYZE
SELECT * FROM customers
WHERE created_at >= '2025-01-01'
  AND created_at  < '2026-01-01';
```

Output:
```
-> Filter: (customers.created_at >= '2025-01-01' and customers.created_at < '2026-01-01')
     (cost=100964  rows=110514)
     (actual time=0.128..440  rows=165799  loops=1)
   -> Table scan on customers  ...
```

The time drops from 521 ms to 440 ms even with a full scan, because the filter is evaluated earlier. With an index on `created_at` alone the gain would be dramatic.

### Common non-sargable patterns to avoid

| Non-sargable (avoid) | Sargable rewrite |
|----------------------|-----------------|
| `WHERE YEAR(created_at) = 2025` | `WHERE created_at >= '2025-01-01' AND created_at < '2026-01-01'` |
| `WHERE DATE(created_at) = '2025-06-01'` | `WHERE created_at >= '2025-06-01' AND created_at < '2025-06-02'` |
| `WHERE LOWER(email) = 'a@b.com'` | Store email lowercased, or use a functional index |
| `WHERE status + 0 = 1` | `WHERE status = '1'` |

> **Takeaway:** Never apply a function to an indexed column in a `WHERE` clause. Rewrite the predicate so the bare column sits on one side of the operator — this is the single most impactful query rewrite pattern.

---

## Summary

| Part | Concept | Key observation |
|------|---------|-----------------|
| 1 | Parser | Typo → ERROR 1064 before InnoDB is involved |
| 2 | EXPLAIN | Optimizer picks `PRIMARY`, estimates 1 row |
| 3 | EXPLAIN ANALYZE | Adds real timing; gap = stale statistics |
| 4 | Full table scan | No index → 1M rows examined |
| 5 | Single-column index | Cost drops 4×; selectivity limits the gain |
| 6 | Composite index | Eliminates both the scan and the sort in one index |
| 7 | Sargability | Function on column defeats index; bare column rewrite fixes it |

---

## Quick Reference

```sql
-- See the execution plan (no execution)
EXPLAIN SELECT ...;
EXPLAIN FORMAT=TREE SELECT ...;
EXPLAIN FORMAT=JSON SELECT ...\G

-- See the plan with real timing (executes the query)
EXPLAIN ANALYZE SELECT ...\G

-- Refresh optimizer statistics after bulk loads or index creation
ANALYZE TABLE table_name;

-- Check indexes on a table
SHOW INDEX FROM table_name;

-- Force or skip an index (for testing)
SELECT ... FROM t USE INDEX (idx_name) WHERE ...;
SELECT ... FROM t IGNORE INDEX (idx_name) WHERE ...;
```