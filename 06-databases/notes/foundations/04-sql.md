# SQL — The Language of Relational Databases

## What SQL Actually Is

SQL is a **declarative language** rooted in relational algebra and
relational calculus. You describe **what** data you want, not **how**
to get it. The database's query planner figures out the "how" — which
indexes to use, which join algorithm, which scan method.

This means the same SQL query can execute completely differently on
two databases, or even on the same database with different indexes or
statistics. Understanding SQL means understanding both the language
semantics (what does the query mean?) and how those semantics map to
execution (what will the database actually do?).

SQL operates on **relations** (tables) — unordered sets of tuples
(rows). Every SQL operation takes one or more relations as input and
produces a relation as output. This closure property is what makes
SQL composable — you can nest queries, join results, and chain
operations because everything is a relation.

---

## Logical Execution Order

The SQL statement is written in one order but **executed in a
completely different order**. Understanding this is fundamental to
understanding scoping, aliasing, and what's available where.

```
Written order:          Logical execution order:

SELECT                  1. FROM / JOIN      ← build the working set
FROM                    2. WHERE            ← filter rows
WHERE                   3. GROUP BY         ← collapse into groups
GROUP BY                4. HAVING           ← filter groups
HAVING                  5. SELECT           ← compute output columns
ORDER BY                6. DISTINCT         ← deduplicate
LIMIT                   7. ORDER BY         ← sort results
                        8. LIMIT / OFFSET   ← truncate output
```

**Why this matters:**

```sql
-- This fails:
SELECT name, salary * 1.1 AS new_salary
FROM employees
WHERE new_salary > 100000;
-- ERROR: column "new_salary" does not exist

-- Why: WHERE executes before SELECT. The alias "new_salary"
-- doesn't exist yet when WHERE runs.

-- Fix: repeat the expression or use a subquery
SELECT name, salary * 1.1 AS new_salary
FROM employees
WHERE salary * 1.1 > 100000;

-- This works:
SELECT name, salary * 1.1 AS new_salary
FROM employees
ORDER BY new_salary;
-- ORDER BY executes AFTER SELECT — the alias exists.
```

```sql
-- You can't use window functions in WHERE:
SELECT name, ROW_NUMBER() OVER (ORDER BY salary DESC) AS rank
FROM employees
WHERE rank <= 10;
-- ERROR: window functions not allowed in WHERE

-- Why: WHERE runs before SELECT (where window functions are computed).
-- Fix: use a subquery or CTE
SELECT * FROM (
    SELECT name, ROW_NUMBER() OVER (ORDER BY salary DESC) AS rank
    FROM employees
) sub
WHERE rank <= 10;
```

---

## The Relational Algebra Behind SQL

Every SQL operation maps to a relational algebra operation. The planner
converts SQL into these operations before execution.

### Selection (σ) — WHERE

Filter rows matching a predicate. Does NOT remove columns.

```
σ(age > 30)(users) → all rows where age > 30, all columns

SQL: SELECT * FROM users WHERE age > 30;
```

### Projection (π) — SELECT

Choose which columns to output. Removes duplicates in pure relational
algebra (SQL doesn't by default — use DISTINCT).

```
π(name, email)(users) → only name and email columns

SQL: SELECT DISTINCT name, email FROM users;
```

### Cartesian Product (×) — CROSS JOIN

Every row of A paired with every row of B. If A has N rows and B has
M rows, the result has N × M rows. Almost never what you want.

```
users × orders → every user paired with every order

SQL: SELECT * FROM users CROSS JOIN orders;
     SELECT * FROM users, orders;  -- implicit cross join
```

### Join (⋈) — JOIN ... ON

Cartesian product filtered by a join condition. The most fundamental
multi-table operation.

```
users ⋈(users.id = orders.user_id) orders

SQL: SELECT * FROM users JOIN orders ON users.id = orders.user_id;
```

### Set Operations (∪, ∩, −) — UNION, INTERSECT, EXCEPT

Combine results of two queries with the same column structure.

```
π(email)(customers) ∪ π(email)(employees) → all emails from both

SQL: SELECT email FROM customers UNION SELECT email FROM employees;
```

### Division (÷) — No direct SQL equivalent

"Find all X that are related to every Y." The hardest relational algebra
operation to express in SQL.

```
-- Find students enrolled in ALL courses:

-- Relational algebra: enrollments ÷ courses

-- SQL (double negation: "no course exists that this student isn't enrolled in"):
SELECT s.student_id
FROM students s
WHERE NOT EXISTS (
    SELECT c.course_id
    FROM courses c
    WHERE NOT EXISTS (
        SELECT 1
        FROM enrollments e
        WHERE e.student_id = s.student_id
          AND e.course_id = c.course_id
    )
);
```

---

## JOIN Types — Complete Reference

### INNER JOIN

Returns only rows where the join condition matches in **both** tables.
Non-matching rows are discarded from both sides.

```sql
SELECT u.name, o.amount
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- Users with no orders: excluded
-- Orders with no matching user (orphaned FK): excluded
```

### LEFT (OUTER) JOIN

Returns all rows from the left table. If no match exists in the right
table, the right side columns are NULL.

```sql
SELECT u.name, o.amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;

-- Users with no orders: included (o.amount = NULL)
-- Every row from users appears at least once
```

### RIGHT (OUTER) JOIN

Mirror of LEFT JOIN. Returns all rows from the right table. Rarely used
— rewrite as LEFT JOIN with tables swapped for readability.

### FULL (OUTER) JOIN

Returns all rows from both tables. NULLs on whichever side has no match.

```sql
SELECT u.name, o.amount
FROM users u
FULL OUTER JOIN orders o ON u.id = o.user_id;

-- Users with no orders: included (o.amount = NULL)
-- Orders with no matching user: included (u.name = NULL)
```

### CROSS JOIN

Cartesian product. Every row paired with every row. No ON clause.

```sql
SELECT * FROM sizes CROSS JOIN colors;

-- 3 sizes × 4 colors = 12 rows (all combinations)
-- Use case: generating all combinations for a product matrix
```

### LATERAL JOIN

The right side of the join can reference columns from the left side.
This turns the right side into a **correlated subquery that returns
a set** — evaluated once per left row.

```sql
-- For each user, get their 3 most recent orders:
SELECT u.name, recent.amount, recent.created_at
FROM users u
CROSS JOIN LATERAL (
    SELECT o.amount, o.created_at
    FROM orders o
    WHERE o.user_id = u.id
    ORDER BY o.created_at DESC
    LIMIT 3
) recent;

-- Without LATERAL, you'd need a window function + filter:
SELECT name, amount, created_at
FROM (
    SELECT u.name, o.amount, o.created_at,
           ROW_NUMBER() OVER (PARTITION BY u.id ORDER BY o.created_at DESC) rn
    FROM users u
    JOIN orders o ON u.id = o.user_id
) sub
WHERE rn <= 3;
```

LATERAL is essential for **top-N-per-group** queries. The subquery
inside LATERAL can use LIMIT, aggregates, or any per-group logic that
references the outer row. Under the hood, the planner can use a nested
loop with an index scan on the inner — very efficient with the right
index.

### Self Join

Join a table to itself. Common for hierarchical data or comparing rows
within the same table.

```sql
-- Find employees who earn more than their manager:
SELECT e.name AS employee, m.name AS manager, e.salary, m.salary
FROM employees e
JOIN employees m ON e.manager_id = m.id
WHERE e.salary > m.salary;
```

### Anti Join (NOT EXISTS / NOT IN / LEFT JOIN WHERE NULL)

Find rows in the left table that have **no match** in the right table.
Three syntaxes, different performance characteristics:

```sql
-- 1. NOT EXISTS (usually best — can short-circuit)
SELECT u.name
FROM users u
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);

-- 2. LEFT JOIN + IS NULL (same semantics, planner often converts to anti join)
SELECT u.name
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.id IS NULL;

-- 3. NOT IN (DANGER: different NULL semantics!)
SELECT u.name
FROM users u
WHERE u.id NOT IN (SELECT user_id FROM orders);
-- If ANY user_id in orders is NULL, the entire NOT IN returns
-- no rows (NULL comparison propagates). This is a common bug.
-- Always prefer NOT EXISTS over NOT IN.
```

### Semi Join (EXISTS)

Return rows from the left table where a match **exists** in the right
table. Like INNER JOIN but doesn't duplicate left rows when multiple
right rows match.

```sql
-- Find users who have at least one order:
SELECT u.name
FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);

-- Different from INNER JOIN: if a user has 5 orders, EXISTS returns
-- the user once. INNER JOIN returns them 5 times.
-- The planner converts EXISTS into a semi join operator internally.
```

---

## Aggregation

### Aggregate Functions

Aggregate functions collapse multiple rows into a single value:

```sql
SELECT
    COUNT(*)           AS total_rows,        -- count all rows (includes NULLs)
    COUNT(email)       AS non_null_emails,   -- count non-NULL values
    COUNT(DISTINCT city) AS unique_cities,   -- count distinct non-NULL values
    SUM(amount)        AS total_amount,
    AVG(amount)        AS avg_amount,        -- SUM / COUNT (ignores NULLs)
    MIN(created_at)    AS first_order,
    MAX(created_at)    AS last_order,
    BOOL_OR(active)    AS any_active,        -- true if any value is true
    BOOL_AND(active)   AS all_active,        -- true only if all values are true
    STRING_AGG(name, ', ' ORDER BY name) AS names,  -- concatenate with separator
    ARRAY_AGG(tag ORDER BY tag) AS tags,             -- collect into array
    JSONB_AGG(data)    AS json_array          -- collect into JSON array
FROM orders;
```

**NULL handling is critical**: All aggregate functions except `COUNT(*)`
**ignore NULL values**. `AVG(column)` computes the average of non-NULL
values, not the average treating NULL as 0. This is mathematically
correct (NULL means "unknown," not "zero") but trips up many developers.

```sql
-- These are NOT equivalent:
SELECT AVG(rating) FROM reviews;           -- average of non-NULL ratings
SELECT SUM(rating) / COUNT(*) FROM reviews; -- includes NULLs in denominator
                                            -- (and SUM ignores NULLs)

-- NULL-safe average treating NULL as 0:
SELECT AVG(COALESCE(rating, 0)) FROM reviews;
```

### GROUP BY

Partitions rows into groups. Each group becomes one output row.
Every column in SELECT must either be in GROUP BY or inside an
aggregate function.

```sql
SELECT
    department,
    COUNT(*) AS headcount,
    AVG(salary) AS avg_salary,
    MAX(salary) - MIN(salary) AS salary_spread
FROM employees
GROUP BY department;
```

**HAVING** filters groups (runs after GROUP BY). WHERE filters
individual rows (runs before GROUP BY).

```sql
-- Departments with more than 10 employees AND average salary > 80K:
SELECT department, COUNT(*), AVG(salary)
FROM employees
WHERE active = true              -- filters individual rows first
GROUP BY department
HAVING COUNT(*) > 10             -- then filters resulting groups
   AND AVG(salary) > 80000;
```

### GROUPING SETS, CUBE, ROLLUP

Generate multiple levels of aggregation in a single query.

```sql
-- GROUPING SETS: explicit list of grouping combinations
SELECT region, department, SUM(salary)
FROM employees
GROUP BY GROUPING SETS (
    (region, department),   -- per region+department
    (region),               -- per region total
    (department),           -- per department total
    ()                      -- grand total
);

-- ROLLUP: hierarchical aggregation (progressively removes rightmost columns)
SELECT region, department, SUM(salary)
FROM employees
GROUP BY ROLLUP (region, department);
-- Produces: (region, department), (region), ()

-- CUBE: all possible combinations
SELECT region, department, SUM(salary)
FROM employees
GROUP BY CUBE (region, department);
-- Produces: (region, department), (region), (department), ()

-- GROUPING() function: tells you which columns are aggregated
SELECT
    CASE WHEN GROUPING(region) = 1 THEN 'All Regions' ELSE region END,
    CASE WHEN GROUPING(department) = 1 THEN 'All Depts' ELSE department END,
    SUM(salary)
FROM employees
GROUP BY ROLLUP (region, department);
```

The planner executes grouping sets as a single pass through the data
with multiple aggregate states — not as N separate queries. This is
significantly more efficient than UNIONing separate GROUP BY queries.

### FILTER Clause

Apply different predicates to different aggregates in the same query:

```sql
SELECT
    COUNT(*) AS total,
    COUNT(*) FILTER (WHERE status = 'active') AS active_count,
    COUNT(*) FILTER (WHERE status = 'inactive') AS inactive_count,
    AVG(salary) FILTER (WHERE department = 'engineering') AS eng_avg_salary,
    AVG(salary) FILTER (WHERE department = 'sales') AS sales_avg_salary
FROM employees;

-- Without FILTER, you'd use CASE:
SELECT
    COUNT(CASE WHEN status = 'active' THEN 1 END) AS active_count
FROM employees;
-- FILTER is cleaner and the planner can optimize it better.
```

---

## Window Functions

Window functions compute values across a set of rows **related to the
current row** without collapsing them into a single output row like
aggregate functions do. They operate on a "window" of rows defined by
PARTITION BY and ORDER BY.

### Anatomy of a Window Function

```sql
function_name(args) OVER (
    [PARTITION BY expr, ...]    -- divide into groups (like GROUP BY, but no collapsing)
    [ORDER BY expr [ASC|DESC], ...]  -- sort within partition
    [frame_clause]              -- which rows relative to current row
)
```

### Ranking Functions

```sql
SELECT
    name, department, salary,
    ROW_NUMBER() OVER w AS row_num,      -- 1,2,3,4,5 (always unique)
    RANK()       OVER w AS rank,         -- 1,2,2,4,5 (gaps after ties)
    DENSE_RANK() OVER w AS dense_rank,   -- 1,2,2,3,4 (no gaps after ties)
    NTILE(4)     OVER w AS quartile,     -- divide into 4 equal groups
    PERCENT_RANK() OVER w AS pct_rank,   -- (rank - 1) / (total_rows - 1)
    CUME_DIST()    OVER w AS cume_dist   -- fraction of rows <= current row
FROM employees
WINDOW w AS (PARTITION BY department ORDER BY salary DESC);
```

**ROW_NUMBER vs RANK vs DENSE_RANK** — the critical difference:

```
Salary:     100, 90, 90, 80, 70

ROW_NUMBER: 1, 2, 3, 4, 5    (arbitrary tiebreaker for 90s — nondeterministic!)
RANK:       1, 2, 2, 4, 5    (skip 3 because two items share rank 2)
DENSE_RANK: 1, 2, 2, 3, 4    (no skip — next rank after tie is 3)
```

ROW_NUMBER with ties is **nondeterministic** — two runs can produce
different results. Add a tiebreaker column to ORDER BY:

```sql
ROW_NUMBER() OVER (ORDER BY salary DESC, id ASC)  -- deterministic
```

### Offset Functions

Access values from other rows in the partition:

```sql
SELECT
    date, revenue,
    LAG(revenue, 1)  OVER w AS prev_revenue,     -- previous row's value
    LEAD(revenue, 1) OVER w AS next_revenue,     -- next row's value
    LAG(revenue, 7)  OVER w AS week_ago_revenue, -- 7 rows back
    FIRST_VALUE(revenue) OVER w AS first_revenue, -- first in partition
    LAST_VALUE(revenue)  OVER w AS last_revenue,  -- last in partition
    NTH_VALUE(revenue, 3) OVER w AS third_revenue -- Nth row in partition
FROM daily_metrics
WINDOW w AS (ORDER BY date);
```

**LAST_VALUE trap**: The default frame is `RANGE BETWEEN UNBOUNDED
PRECEDING AND CURRENT ROW`. This means LAST_VALUE returns the current
row, not the actual last row. Fix:

```sql
LAST_VALUE(revenue) OVER (
    ORDER BY date
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
) AS actual_last_revenue
```

### Running Aggregates

```sql
SELECT
    date, revenue,
    SUM(revenue) OVER (ORDER BY date) AS running_total,
    AVG(revenue) OVER (ORDER BY date) AS running_avg,
    SUM(revenue) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS rolling_7day_sum,
    AVG(revenue) OVER (
        ORDER BY date
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ) AS rolling_30day_avg
FROM daily_metrics;
```

### Frame Specifications

The frame defines which rows are included relative to the current row:

```
ROWS BETWEEN <start> AND <end>
RANGE BETWEEN <start> AND <end>
GROUPS BETWEEN <start> AND <end>

Bounds:
  UNBOUNDED PRECEDING    ← first row of partition
  N PRECEDING            ← N rows/values before current
  CURRENT ROW            ← the current row
  N FOLLOWING            ← N rows/values after current
  UNBOUNDED FOLLOWING    ← last row of partition
```

**ROWS vs RANGE vs GROUPS:**

```sql
-- Data: date=Mon, Tue, Tue, Wed, Thu
-- Value: 10, 20, 30, 40, 50

-- ROWS BETWEEN 1 PRECEDING AND CURRENT ROW
-- For second Tue (value=30): includes previous row (Tue, 20) + current (Tue, 30)
-- → Based on physical row position

-- RANGE BETWEEN 1 PRECEDING AND CURRENT ROW (with ORDER BY date)
-- For second Tue (value=30): includes ALL rows with same date ± range
-- Both Tues are "current row" peers. Also includes Mon (1 day preceding)
-- → Based on value proximity

-- GROUPS BETWEEN 1 PRECEDING AND CURRENT ROW
-- For second Tue (value=30): includes the Tue group + Mon group
-- → Based on peer group position
```

### Window Function Execution Internals

The planner adds a **WindowAgg** node to the execution plan. How it
works internally:

```
Plan:
  WindowAgg (partition by department, order by salary)
    → Sort (department, salary)
      → SeqScan (employees)

Execution:
  1. Sort all rows by (department, salary)
  2. Scan sorted input:
     - Track current partition (when department changes, reset)
     - Maintain frame boundaries (two pointers into sorted rows)
     - For each row: compute the window function using the frame
     - Emit the row with the computed value appended

Memory: O(partition_size) worst case (when frame is UNBOUNDED)
  Some functions (ROW_NUMBER, RANK) need only O(1) state
  Running SUM needs O(1) state (add new, subtract old)
  ARRAY_AGG needs O(frame_size) state
```

Multiple window functions with the **same** PARTITION BY and ORDER BY
share one WindowAgg pass. Different windows require separate sorts:

```sql
-- One sort, one WindowAgg pass:
SELECT
    ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary),
    SUM(salary)  OVER (PARTITION BY dept ORDER BY salary)
FROM employees;

-- Two sorts, two WindowAgg passes:
SELECT
    ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary),
    ROW_NUMBER() OVER (PARTITION BY dept ORDER BY hire_date)
FROM employees;
```

Use the `WINDOW` clause to define reusable window specifications:

```sql
SELECT
    ROW_NUMBER() OVER w,
    SUM(salary) OVER w,
    AVG(salary) OVER w
FROM employees
WINDOW w AS (PARTITION BY department ORDER BY salary DESC);
```

---

## Subqueries

### Scalar Subquery

Returns exactly one value. Can be used anywhere a single value is
expected.

```sql
SELECT name, salary,
       salary - (SELECT AVG(salary) FROM employees) AS diff_from_avg
FROM employees;

-- If the subquery returns more than one row: ERROR
-- If the subquery returns zero rows: NULL
```

### Correlated Subquery

References the outer query. Conceptually re-executed for every outer
row (though the planner often optimizes this into a join).

```sql
-- Employees who earn more than their department's average:
SELECT e.name, e.salary
FROM employees e
WHERE e.salary > (
    SELECT AVG(e2.salary)
    FROM employees e2
    WHERE e2.department = e.department   -- references outer query
);

-- The planner may convert this to:
-- HashAggregate(employees by department) → Hash Join → Filter
-- Much more efficient than executing the subquery per row.
```

### EXISTS / NOT EXISTS

Tests for the existence of rows. Efficient because it can **short-
circuit** — returns TRUE as soon as one matching row is found, without
scanning the entire subquery.

```sql
-- Customers who have placed at least one order this year:
SELECT c.name
FROM customers c
WHERE EXISTS (
    SELECT 1        -- the value doesn't matter, only existence
    FROM orders o
    WHERE o.customer_id = c.id
      AND o.created_at >= '2024-01-01'
);
```

### IN / ANY / ALL

```sql
-- IN: matches any value in the list/subquery
SELECT * FROM users WHERE id IN (1, 2, 3);
SELECT * FROM users WHERE id IN (SELECT user_id FROM active_users);

-- ANY/SOME: matches if the comparison is true for any value
SELECT * FROM products WHERE price > ANY (SELECT price FROM discounted);
-- Equivalent to: price > MIN(discounted prices)

-- ALL: matches if the comparison is true for all values
SELECT * FROM products WHERE price > ALL (SELECT price FROM discounted);
-- Equivalent to: price > MAX(discounted prices)

-- NULL trap with IN/NOT IN:
SELECT * FROM users WHERE id NOT IN (1, 2, NULL);
-- This returns NOTHING. Because id != NULL is UNKNOWN for every row,
-- and NOT IN requires ALL comparisons to be true.
-- The NULL poisons the entire NOT IN.
-- Always use NOT EXISTS instead of NOT IN when NULLs are possible.
```

---

## Common Table Expressions (CTEs)

### Non-Recursive CTEs

Named temporary result sets that exist for the duration of the query.
Improve readability and allow referencing the same subquery multiple
times.

```sql
WITH monthly_revenue AS (
    SELECT
        DATE_TRUNC('month', created_at) AS month,
        SUM(amount) AS revenue
    FROM orders
    WHERE status = 'completed'
    GROUP BY DATE_TRUNC('month', created_at)
),
growth AS (
    SELECT
        month,
        revenue,
        LAG(revenue) OVER (ORDER BY month) AS prev_revenue,
        ROUND(100.0 * (revenue - LAG(revenue) OVER (ORDER BY month))
              / LAG(revenue) OVER (ORDER BY month), 1) AS growth_pct
    FROM monthly_revenue
)
SELECT * FROM growth WHERE growth_pct > 20;
```

**CTE materialization behavior (PostgreSQL 12+):**

Before PostgreSQL 12, CTEs were **always materialized** — the CTE was
executed once, results stored in a temp tuplestore, and any references
read from that store. This was an **optimization fence**: the planner
couldn't push predicates from the outer query into the CTE.

```sql
-- Pre-PG12: this scans ALL users, materializes them, then filters
WITH all_users AS (
    SELECT * FROM users
)
SELECT * FROM all_users WHERE id = 42;

-- Post-PG12: the planner inlines the CTE and pushes the predicate down
-- Equivalent to: SELECT * FROM users WHERE id = 42
```

PostgreSQL 12+ **inlines** CTEs that are referenced only once and have
no side effects. To force materialization (if the CTE is referenced
multiple times and you want to avoid recomputation):

```sql
WITH expensive_query AS MATERIALIZED (
    SELECT ... complex computation ...
)
SELECT * FROM expensive_query a
JOIN expensive_query b ON ...;
```

### Recursive CTEs

Execute a base query, then repeatedly execute a recursive query that
references the CTE's own output, until no new rows are produced.

```sql
WITH RECURSIVE cte AS (
    -- Base case (non-recursive term):
    SELECT ...
    UNION [ALL]
    -- Recursive term (references cte):
    SELECT ... FROM cte JOIN ...
)
SELECT * FROM cte;
```

**Execution model:**

```
1. Execute base case → put results in working table
2. While working table is not empty:
   a. Execute recursive term using working table as input
   b. New results become the next working table
   c. Append new results to the final output
3. Return all accumulated results
```

**Example: organizational hierarchy**

```sql
WITH RECURSIVE org_tree AS (
    -- Base: start from the CEO (no manager)
    SELECT id, name, manager_id, 1 AS depth, ARRAY[id] AS path
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive: find all direct reports of the current level
    SELECT e.id, e.name, e.manager_id, t.depth + 1, t.path || e.id
    FROM employees e
    JOIN org_tree t ON e.manager_id = t.id
    WHERE NOT (e.id = ANY(t.path))   -- cycle prevention
)
SELECT
    REPEAT('  ', depth - 1) || name AS org_chart,
    depth
FROM org_tree
ORDER BY path;
```

```
org_chart         depth
─────────────────────────
Alice (CEO)         1
  Bob (VP Eng)      2
    Carol (Lead)    3
      Dave (Dev)    4
    Eve (Lead)      3
  Frank (VP Sales)  2
    Grace (Rep)     3
```

**Cycle detection**: Without the `WHERE NOT (e.id = ANY(t.path))`
guard, a cycle in the data (employee reports to their own report)
causes infinite recursion. PostgreSQL 14+ has built-in cycle detection:

```sql
WITH RECURSIVE org_tree AS (
    SELECT id, name, manager_id
    FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.id, e.name, e.manager_id
    FROM employees e JOIN org_tree t ON e.manager_id = t.id
)
CYCLE id SET is_cycle USING path   -- PG14+ syntax
SELECT * FROM org_tree WHERE NOT is_cycle;
```

**Example: generating a series (how generate_series works conceptually)**

```sql
WITH RECURSIVE nums AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM nums WHERE n < 100
)
SELECT n FROM nums;
-- Returns: 1, 2, 3, ..., 100
```

**Example: graph traversal (shortest path)**

```sql
-- Find all nodes reachable from node 1 with distance:
WITH RECURSIVE reachable AS (
    SELECT id AS node, 0 AS distance
    FROM graph_nodes
    WHERE id = 1

    UNION  -- not UNION ALL: deduplicates to avoid revisiting nodes

    SELECT e.target, r.distance + 1
    FROM reachable r
    JOIN edges e ON r.node = e.source
)
SELECT node, MIN(distance) AS shortest_distance
FROM reachable
GROUP BY node;
```

---

## Data Types

### Numeric Types

```
Type              Storage    Range                          Notes
──────────────────────────────────────────────────────────────────────
smallint (int2)   2 bytes    -32,768 to 32,767              Rarely used
integer (int4)    4 bytes    -2.1B to 2.1B                  Default integer
bigint (int8)     8 bytes    -9.2×10¹⁸ to 9.2×10¹⁸        Use for IDs, counters
numeric(p,s)      variable   up to 131,072 digits before    Exact decimal. Use for
                             decimal point                  money. SLOW for math.
real (float4)     4 bytes    ~6 decimal digits precision    IEEE 754. Fast, inexact
double (float8)   8 bytes    ~15 decimal digits precision   IEEE 754. Fast, inexact
serial            4 bytes    auto-increment integer         Actually an int + sequence
bigserial         8 bytes    auto-increment bigint          Prefer GENERATED ALWAYS
```

**The money trap:**

```sql
-- NEVER use float for money:
SELECT 0.1::float + 0.2::float;        -- 0.30000000000000004
SELECT 0.1::float + 0.2::float = 0.3;  -- false!

-- Use numeric (exact decimal arithmetic):
SELECT 0.1::numeric + 0.2::numeric = 0.3;  -- true
SELECT 10.99::numeric(10,2) * 3;            -- 32.97 (exact)

-- Or use integer cents:
-- Store $10.99 as 1099 (integer). Divide by 100 for display.
-- Fastest option. No rounding issues.
```

**serial vs GENERATED ALWAYS:**

```sql
-- Old way (serial = implicit sequence):
CREATE TABLE users (
    id serial PRIMARY KEY  -- creates a sequence, hides it behind sugar
);

-- Modern way (SQL standard, explicit):
CREATE TABLE users (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY
);
-- Cleaner: the identity is visible in the table definition
-- Safer: GENERATED ALWAYS prevents manual ID insertion (unlike serial)
-- Can use OVERRIDING SYSTEM VALUE to force a specific ID if needed
```

### Text Types

```
Type           Storage              Max Size        Notes
──────────────────────────────────────────────────────────────────────
char(n)        padded to n bytes    up to 10 MB     Right-pads with spaces.
                                                    Almost never use.
varchar(n)     up to n + header     up to 10 MB     Length-limited text
text           unlimited + header   up to 1 GB      Unlimited length
```

**In PostgreSQL, there is NO performance difference between varchar(n)
and text.** Both use the same internal varlena format:

```
Storage format for text/varchar:
  [4 bytes: varlena header (includes length)]
  [N bytes: actual string data]

If > ~2 KB: compressed and/or TOASTed automatically
  (see 06-postgresql.md TOAST section)
```

Use `text` for most columns. Use `varchar(n)` only if you need to
enforce a **business constraint** on maximum length (e.g., country
code must be 2 characters). The length check has negligible overhead.

### Timestamps

```sql
-- Without timezone (stores the literal value as-is):
timestamp without time zone
'2024-06-15 14:30:00'  -- stored exactly as written

-- With timezone (stores internally as UTC, displays in session timezone):
timestamp with time zone (timestamptz)
'2024-06-15 14:30:00 America/New_York'  -- stored as UTC: 18:30:00+00

-- ALWAYS use timestamptz. Always. Without exception.
-- timestamp without time zone is a bug waiting to happen:
SET timezone = 'America/New_York';
SELECT '2024-06-15 14:30:00'::timestamptz;  -- 2024-06-15 14:30:00-04
SET timezone = 'Europe/London';
SELECT '2024-06-15 14:30:00'::timestamptz;  -- 2024-06-15 14:30:00+01
-- Same query, different interpretation. The timezone context matters.

-- Internal storage: both types are 8 bytes (microseconds since 2000-01-01 UTC)
-- timestamptz converts to UTC on input and to session timezone on output
-- timestamp stores the literal microseconds with no conversion
```

### UUID

```sql
-- 128-bit universally unique identifier. 16 bytes storage.
-- Better than serial for distributed systems (no central sequence needed)

-- Generation (PostgreSQL 13+):
SELECT gen_random_uuid();  -- v4 (random)
-- Output: a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11

-- As primary key:
CREATE TABLE events (
    id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
    data jsonb
);

-- Indexing consideration: UUIDs are random → B-tree inserts are random
-- across the index (no sequential append). This causes more page splits
-- and worse cache behavior than sequential IDs. For write-heavy tables,
-- consider UUIDv7 (timestamp-ordered) or use bigserial.
```

### JSONB

```sql
-- Binary JSON. Supports indexing, containment, path operators.
-- Stored in a decomposed binary format (not as a JSON text string).

CREATE TABLE documents (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    data jsonb NOT NULL
);

-- Operators:
SELECT data->>'name'             FROM documents;  -- extract as text
SELECT data->'address'->>'city'  FROM documents;  -- nested extraction
SELECT data @> '{"role":"admin"}'FROM documents;  -- containment
SELECT data ? 'email'            FROM documents;  -- key exists
SELECT data ?| ARRAY['email','phone'] FROM documents; -- any key exists
SELECT data ?& ARRAY['email','phone'] FROM documents; -- all keys exist
SELECT data #>> '{address,city}' FROM documents;  -- path extraction as text

-- Modification (immutable — returns new JSONB):
SELECT data || '{"verified": true}'::jsonb    FROM documents;  -- merge
SELECT data - 'temporary_field'               FROM documents;  -- remove key
SELECT jsonb_set(data, '{address,zip}', '"10001"') FROM documents; -- set path

-- Indexing:
CREATE INDEX idx_data ON documents USING GIN (data);
-- Supports: @>, ?, ?|, ?& operators

CREATE INDEX idx_data_path ON documents USING GIN (data jsonb_path_ops);
-- Only supports @> (containment) but smaller and faster index

-- When JSONB vs relational columns:
-- Use JSONB for: truly schemaless data, rarely queried metadata,
--   external API payloads, varying document structures
-- Use columns for: frequently filtered/joined data, data with
--   constraints, data that participates in foreign keys
-- JSONB can't enforce NOT NULL on individual fields or foreign keys
```

### Arrays

```sql
-- PostgreSQL native array type. Any type can be arrayed.

CREATE TABLE posts (
    id bigint PRIMARY KEY,
    tags text[] NOT NULL DEFAULT '{}'
);

INSERT INTO posts (id, tags) VALUES (1, ARRAY['python', 'web', 'api']);
INSERT INTO posts (id, tags) VALUES (2, '{go,grpc,microservices}');

-- Operators:
SELECT * FROM posts WHERE 'python' = ANY(tags);      -- contains element
SELECT * FROM posts WHERE tags @> ARRAY['web','api']; -- contains all
SELECT * FROM posts WHERE tags && ARRAY['python','go']; -- overlap (any in common)
SELECT * FROM posts WHERE array_length(tags, 1) > 2;  -- length

-- Index:
CREATE INDEX idx_tags ON posts USING GIN (tags);
-- Supports @>, &&, = operators

-- Unnest: expand array to rows (inverse of ARRAY_AGG)
SELECT id, unnest(tags) AS tag FROM posts;
-- id=1, 'python'; id=1, 'web'; id=1, 'api'; id=2, 'go'; ...
```

---

## Advanced Patterns

### UPSERT (INSERT ... ON CONFLICT)

Atomic insert-or-update. Solves the race condition between
"check if exists" and "insert or update."

```sql
-- Insert if not exists, update if conflict on unique constraint:
INSERT INTO user_settings (user_id, theme, language)
VALUES (42, 'dark', 'en')
ON CONFLICT (user_id)
DO UPDATE SET
    theme = EXCLUDED.theme,         -- EXCLUDED refers to the proposed row
    language = EXCLUDED.language,
    updated_at = NOW();

-- Insert if not exists, do nothing on conflict:
INSERT INTO unique_events (event_id, data)
VALUES ('evt_123', '{"type":"click"}')
ON CONFLICT (event_id) DO NOTHING;

-- Conditional update (only update if the new data is "newer"):
INSERT INTO inventory (product_id, quantity, last_updated)
VALUES (100, 50, NOW())
ON CONFLICT (product_id)
DO UPDATE SET
    quantity = EXCLUDED.quantity,
    last_updated = EXCLUDED.last_updated
WHERE inventory.last_updated < EXCLUDED.last_updated;
-- Only updates if the incoming data is more recent
```

Under the hood: PostgreSQL attempts the INSERT. If a unique constraint
violation occurs, it catches the error internally and executes the
DO UPDATE instead. The entire operation is atomic (no race condition
between check and insert).

### RETURNING

Get back the affected rows from INSERT, UPDATE, or DELETE:

```sql
-- Get the auto-generated ID:
INSERT INTO users (name, email)
VALUES ('alice', 'alice@example.com')
RETURNING id;

-- Get old values before update:
UPDATE accounts
SET balance = balance - 100
WHERE id = 42
RETURNING id, balance AS new_balance;

-- Use in a CTE to chain DML:
WITH deleted AS (
    DELETE FROM expired_sessions
    WHERE expires_at < NOW()
    RETURNING user_id, session_id
)
INSERT INTO session_audit (user_id, session_id, action)
SELECT user_id, session_id, 'expired'
FROM deleted;
```

### SELECT FOR UPDATE (Row Locking)

```sql
-- Lock rows during a read-modify-write cycle:
BEGIN;
SELECT balance FROM accounts WHERE id = 42 FOR UPDATE;
-- Row is now locked. Other transactions trying to UPDATE or
-- SELECT FOR UPDATE this row will WAIT until we commit.

UPDATE accounts SET balance = balance - 100 WHERE id = 42;
COMMIT;

-- SKIP LOCKED: skip rows that are already locked (job queues):
SELECT id, payload
FROM jobs
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
-- Returns the next available (unlocked) job.
-- Multiple workers can run this concurrently without blocking.
-- This is how you build a simple, reliable job queue in PostgreSQL.

-- NOWAIT: fail immediately if the row is locked:
SELECT * FROM accounts WHERE id = 42 FOR UPDATE NOWAIT;
-- ERROR: could not obtain lock on row in relation "accounts"
-- (instead of waiting indefinitely)
```

### Batch Operations

```sql
-- Multi-row INSERT:
INSERT INTO events (type, data) VALUES
    ('click', '{"page":"/home"}'),
    ('view',  '{"page":"/about"}'),
    ('click', '{"page":"/pricing"}');
-- One round trip, one WAL flush for the whole batch.
-- 10-100x faster than individual INSERTs.

-- Batch UPDATE with VALUES:
UPDATE products AS p
SET price = v.new_price
FROM (VALUES (1, 19.99), (2, 29.99), (3, 9.99)) AS v(id, new_price)
WHERE p.id = v.id;

-- Batch DELETE:
DELETE FROM logs
WHERE created_at < NOW() - INTERVAL '90 days';
-- For very large deletes, do it in batches to avoid long-held locks:
DELETE FROM logs
WHERE id IN (
    SELECT id FROM logs
    WHERE created_at < NOW() - INTERVAL '90 days'
    LIMIT 10000
);
-- Run in a loop until 0 rows affected.
```

### COPY — Bulk Loading

The fastest way to load data. Bypasses the SQL parser and executor
per-row overhead.

```sql
-- From CSV file:
COPY users (name, email, created_at) FROM '/path/to/users.csv'
WITH (FORMAT csv, HEADER true, DELIMITER ',');

-- From STDIN (application sends data via wire protocol):
COPY users (name, email) FROM STDIN WITH (FORMAT csv);

-- To CSV (export):
COPY (SELECT * FROM users WHERE active = true) TO '/path/to/export.csv'
WITH (FORMAT csv, HEADER true);
```

Performance comparison for 1 million rows:
```
Individual INSERTs:            ~300 seconds
Multi-row INSERT (1000/batch): ~30 seconds
COPY:                          ~3 seconds
```

COPY streams data directly into heap pages without per-row SQL parsing.
It writes WAL records in bulk. Index updates are batched. This is why
COPY is 100x faster.

### Lateral Subquery Patterns

```sql
-- Top-N per group (most efficient with LATERAL):
SELECT u.name, r.title, r.amount
FROM users u
CROSS JOIN LATERAL (
    SELECT title, amount
    FROM orders
    WHERE user_id = u.id
    ORDER BY amount DESC
    LIMIT 5
) r;
-- With index on orders(user_id, amount DESC): nested loop + index scan
-- Each user triggers one index scan that stops after 5 rows.
-- Far more efficient than window function for small N and large table.

-- Unpivot columns to rows:
SELECT id, key, value
FROM measurements m,
LATERAL (VALUES
    ('temperature', m.temperature),
    ('humidity', m.humidity),
    ('pressure', m.pressure)
) AS unpivoted(key, value);

-- Call a set-returning function per row:
SELECT u.name, s.tag
FROM users u,
LATERAL unnest(u.tags) AS s(tag);
```

### Conditional Expressions

```sql
-- CASE: SQL's if/else
SELECT name,
    CASE
        WHEN salary > 200000 THEN 'executive'
        WHEN salary > 100000 THEN 'senior'
        WHEN salary > 50000  THEN 'mid'
        ELSE 'junior'
    END AS level
FROM employees;

-- Simple CASE (equality):
SELECT name,
    CASE department
        WHEN 'engineering' THEN 'E'
        WHEN 'sales'       THEN 'S'
        ELSE 'O'
    END AS dept_code
FROM employees;

-- COALESCE: first non-NULL value
SELECT COALESCE(nickname, first_name, 'Anonymous') AS display_name
FROM users;

-- NULLIF: return NULL if two values are equal (prevents division by zero)
SELECT revenue / NULLIF(cost, 0) AS margin
FROM financials;
-- If cost = 0: NULLIF returns NULL, division returns NULL (not error)

-- GREATEST / LEAST:
SELECT GREATEST(a, b, c) AS max_val, LEAST(a, b, c) AS min_val
FROM measurements;
```

---

## NULL Semantics

NULL is the most misunderstood concept in SQL. NULL means **unknown**,
not zero, not empty string, not false. Any operation with NULL propagates
the unknown.

### Three-Valued Logic

SQL uses **three-valued logic**: TRUE, FALSE, UNKNOWN.

```
NULL = NULL      → UNKNOWN (not TRUE!)
NULL != NULL     → UNKNOWN (not TRUE!)
NULL > 5         → UNKNOWN
NULL AND TRUE    → UNKNOWN
NULL AND FALSE   → FALSE (short-circuit: FALSE AND anything = FALSE)
NULL OR TRUE     → TRUE  (short-circuit: TRUE OR anything = TRUE)
NULL OR FALSE    → UNKNOWN
NOT NULL         → UNKNOWN
```

**WHERE clause** returns rows where the expression evaluates to TRUE.
UNKNOWN is treated as FALSE — the row is excluded:

```sql
-- Table: users with name = ['alice', 'bob', NULL]
SELECT * FROM users WHERE name = 'alice';      -- returns: alice
SELECT * FROM users WHERE name != 'alice';     -- returns: bob (NOT the NULL row!)
SELECT * FROM users WHERE name IS NULL;        -- returns: the NULL row
SELECT * FROM users WHERE name IS NOT NULL;    -- returns: alice, bob
```

### NULL in Aggregates

```sql
-- Values: [10, 20, NULL, 40]

COUNT(*) = 4          -- counts rows, not values
COUNT(value) = 3      -- counts non-NULL values
SUM(value) = 70       -- ignores NULL
AVG(value) = 23.33    -- 70 / 3 (not 70 / 4)
MIN(value) = 10       -- ignores NULL
MAX(value) = 40       -- ignores NULL
```

### NULL in DISTINCT and GROUP BY

NULL values are considered **equal** for DISTINCT and GROUP BY
(despite NULL = NULL being UNKNOWN). This is a deliberate exception
to the three-valued logic rules:

```sql
SELECT DISTINCT status FROM orders;
-- Returns: 'active', 'inactive', NULL (one NULL, not duplicated)

SELECT status, COUNT(*)
FROM orders
GROUP BY status;
-- Groups: 'active': 50, 'inactive': 30, NULL: 20
```

### NULL in ORDER BY

NULLs sort last by default in ascending order (PostgreSQL). Controllable:

```sql
ORDER BY column ASC NULLS FIRST     -- NULLs before all values
ORDER BY column ASC NULLS LAST      -- NULLs after all values (PG default for ASC)
ORDER BY column DESC NULLS FIRST    -- NULLs before all values (PG default for DESC)
ORDER BY column DESC NULLS LAST     -- NULLs after all values
```

### NULL-Safe Operators

```sql
-- IS DISTINCT FROM: NULL-safe inequality (treats NULL = NULL as TRUE)
SELECT * FROM users WHERE name IS DISTINCT FROM 'alice';
-- Returns bob AND the NULL row (unlike !=, which excludes NULL)

-- IS NOT DISTINCT FROM: NULL-safe equality
SELECT * FROM users WHERE name IS NOT DISTINCT FROM NULL;
-- Equivalent to IS NULL, but works with any value on right side

-- COALESCE for comparisons:
WHERE COALESCE(column, 'default') = 'default'
-- Maps NULL to a sentinel value for comparison
```

---

## Transaction Control

```sql
-- Explicit transactions:
BEGIN;                    -- or START TRANSACTION
-- ... statements ...
COMMIT;                   -- make changes permanent
-- or
ROLLBACK;                 -- undo all changes

-- Savepoints (partial rollback):
BEGIN;
INSERT INTO users (name) VALUES ('alice');
SAVEPOINT sp1;
INSERT INTO users (name) VALUES ('bob');
-- Oops, bob was wrong:
ROLLBACK TO SAVEPOINT sp1;    -- undoes bob, keeps alice
INSERT INTO users (name) VALUES ('carol');
COMMIT;  -- alice and carol are committed, bob is not

-- Set isolation level:
BEGIN ISOLATION LEVEL SERIALIZABLE;
-- ... strongly isolated statements ...
COMMIT;

-- Read-only transactions (optimizer can be more aggressive):
BEGIN READ ONLY;
SELECT ... ;  -- INSERT/UPDATE/DELETE will ERROR
COMMIT;

-- Deferred constraints (check at commit time, not statement time):
SET CONSTRAINTS ALL DEFERRED;
-- Useful for circular foreign keys or bulk operations where
-- intermediate states temporarily violate constraints
```

---

## DDL — Data Definition Internals

### What ALTER TABLE Actually Does

Not all ALTER TABLE operations are equal. Some are instant, some
rewrite the entire table.

```
Operation                          Lock Level              What Happens
──────────────────────────────────────────────────────────────────────────
ADD COLUMN (no default)            AccessExclusiveLock     Metadata-only change.
                                                          Existing rows read NULL.
                                                          Instant.

ADD COLUMN (with default, PG11+)   AccessExclusiveLock     Metadata-only. Default
                                                          stored in pg_attribute.
                                                          Existing rows lazily
                                                          filled on read. Instant.

ADD COLUMN (with default, <PG11)   AccessExclusiveLock     FULL TABLE REWRITE.
                                                          Every row written with
                                                          the default value.

DROP COLUMN                        AccessExclusiveLock     Metadata-only. Column
                                                          marked as dropped.
                                                          Physical data remains
                                                          until VACUUM FULL.

ALTER COLUMN TYPE                  AccessExclusiveLock     FULL TABLE REWRITE
                                                          (must convert every
                                                          value to new type).

ADD CONSTRAINT (CHECK)             AccessExclusiveLock     Scans entire table to
                                                          validate. Blocks all
                                                          access during scan.

ADD CONSTRAINT (CHECK) NOT VALID   ShareUpdateExclusiveLock  Metadata-only. No
                                                            scan. Enforced for
                                                            new rows only.
VALIDATE CONSTRAINT                ShareUpdateExclusiveLock  Scans table but
                                                            allows concurrent
                                                            reads and writes.

ADD CONSTRAINT (FK)                ShareRowExclusiveLock   Scans table to
                                                          validate references.

ADD CONSTRAINT (FK) NOT VALID      ShareRowExclusiveLock   Metadata-only.
VALIDATE CONSTRAINT                ShareRowExclusiveLock   Concurrent-safe scan.

CREATE INDEX                       ShareLock               Blocks writes.
                                                          Table readable.

CREATE INDEX CONCURRENTLY          ShareUpdateExclusiveLock  Two passes. No lock
                                                            on writes. Slower
                                                            but zero downtime.

SET NOT NULL (PG12+)               AccessExclusiveLock     If a valid CHECK
                                                          constraint exists,
                                                          metadata-only.
```

**Safe migration pattern for adding a NOT NULL column with default:**

```sql
-- Step 1: Add column with default (instant in PG11+)
ALTER TABLE users ADD COLUMN verified boolean DEFAULT false;

-- Step 2: Add NOT NULL constraint as NOT VALID (instant)
ALTER TABLE users ADD CONSTRAINT users_verified_not_null
    CHECK (verified IS NOT NULL) NOT VALID;

-- Step 3: Validate concurrently (allows reads + writes)
ALTER TABLE users VALIDATE CONSTRAINT users_verified_not_null;

-- Step 4: Set actual NOT NULL (instant because valid CHECK exists, PG12+)
ALTER TABLE users ALTER COLUMN verified SET NOT NULL;

-- Step 5: Drop the redundant CHECK constraint
ALTER TABLE users DROP CONSTRAINT users_verified_not_null;
```

This pattern avoids any full table locks that block writes. Each step
is either instant or allows concurrent access. This is how you run
migrations on production databases with zero downtime.

**CREATE INDEX CONCURRENTLY internals:**

```
Normal CREATE INDEX:
  1. Take ShareLock on table (blocks writes, allows reads)
  2. Scan entire table, build index
  3. Release lock

CREATE INDEX CONCURRENTLY:
  1. Take ShareUpdateExclusiveLock (allows reads AND writes)
  2. First pass: scan table, build initial index from snapshot
     (concurrent writes create tuples not in the index)
  3. Wait for all transactions that started before pass 1 to finish
  4. Second pass: scan for tuples added/modified during pass 1
     Add them to the index
  5. Wait for all transactions that started before pass 2 to finish
  6. Mark index as valid

  If it fails partway through: leaves an INVALID index.
  Must DROP INDEX CONCURRENTLY and retry.
```

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| SQL execution order: FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT | Aliases from SELECT not available in WHERE. Window functions not in WHERE |
| JOINs: INNER, LEFT, FULL, CROSS, LATERAL, self, anti (NOT EXISTS), semi (EXISTS) | LATERAL enables per-row subqueries. NOT EXISTS is NULL-safe unlike NOT IN |
| Window functions: OVER (PARTITION BY ... ORDER BY ... frame) | ROW_NUMBER is nondeterministic with ties. LAST_VALUE needs UNBOUNDED FOLLOWING frame |
| ROWS vs RANGE vs GROUPS frame types | ROWS = physical position. RANGE = value proximity. GROUPS = peer groups |
| CTEs: materialized by default pre-PG12, inlined post-PG12 | Recursive CTEs: base case UNION ALL recursive term. Cycle detection with path array |
| NULL = UNKNOWN, not zero or empty | NULL = NULL is UNKNOWN. NOT IN with NULLs returns nothing. Use NOT EXISTS instead |
| Three-valued logic: TRUE, FALSE, UNKNOWN | WHERE excludes UNKNOWN rows. NULL is equal for DISTINCT and GROUP BY |
| UPSERT: INSERT ... ON CONFLICT DO UPDATE | Atomic insert-or-update. EXCLUDED references the proposed row |
| RETURNING gets back affected rows | Chain DML operations: DELETE ... RETURNING in a CTE feeding INSERT |
| COPY is 100x faster than individual INSERTs | Bypasses SQL parser per-row. Streams directly into heap pages |
| SELECT FOR UPDATE SKIP LOCKED builds job queues | Multiple workers grab unlocked rows concurrently without blocking |
| ALTER TABLE: not all operations are equal | ADD COLUMN instant (PG11+). ALTER TYPE = full rewrite. NOT VALID + VALIDATE = zero downtime |
| CREATE INDEX CONCURRENTLY: two-pass, no write lock | Allows concurrent writes. Fails → leaves INVALID index, must retry |
| Always use timestamptz, never timestamp without timezone | Both 8 bytes internally. timestamptz converts to/from UTC automatically |
| numeric for money, never float | 0.1 + 0.2 != 0.3 in float. Use numeric or integer cents |
| text vs varchar(n): no performance difference in PostgreSQL | Both use same varlena format. varchar(n) only for business constraints |
