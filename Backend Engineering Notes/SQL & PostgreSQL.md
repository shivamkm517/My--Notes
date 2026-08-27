
> Your end-to-end reference: Core SQL → Intermediate → Advanced → PostgreSQL Deep Dive → Real-world Use Cases

---

# PART 1: SQL FUNDAMENTALS

## 1.1 What is SQL & RDBMS

**SQL (Structured Query Language)** is the standard language to talk to relational databases — to define, manipulate, control, and query data.

**RDBMS (Relational Database Management System)** stores data in **tables** (rows + columns), where relationships between tables are maintained using **keys**.

Popular RDBMS: PostgreSQL, MySQL, Oracle, SQL Server, SQLite.

### Categories of SQL Commands
| Category | Full Form | Commands |
|---|---|---|
| DDL | Data Definition Language | CREATE, ALTER, DROP, TRUNCATE, RENAME |
| DML | Data Manipulation Language | INSERT, UPDATE, DELETE |
| DQL | Data Query Language | SELECT |
| DCL | Data Control Language | GRANT, REVOKE |
| TCL | Transaction Control Language | COMMIT, ROLLBACK, SAVEPOINT |

---

## 1.2 Databases & Tables

```sql
CREATE DATABASE company;
DROP DATABASE company;

CREATE TABLE employees (
    emp_id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    salary DECIMAL(10,2),
    dept_id INT
);

DROP TABLE employees;
TRUNCATE TABLE employees;   -- removes all rows, keeps structure (faster than DELETE)
ALTER TABLE employees ADD COLUMN email VARCHAR(150);
ALTER TABLE employees DROP COLUMN email;
ALTER TABLE employees RENAME COLUMN name TO full_name;
ALTER TABLE employees RENAME TO staff;
```

**TRUNCATE vs DELETE vs DROP**
- `DELETE` — removes rows one by one, can use WHERE, logged, slower, rollback possible.
- `TRUNCATE` — removes all rows at once, resets identity, minimal logging, can't use WHERE.
- `DROP` — removes the entire table (structure + data).

---

## 1.3 Data Types (Standard SQL)

| Type | Use |
|---|---|
| INT / INTEGER | Whole numbers |
| BIGINT | Large whole numbers |
| DECIMAL(p,s) / NUMERIC | Exact decimal (money) |
| FLOAT / REAL | Approximate decimal |
| VARCHAR(n) | Variable-length string |
| CHAR(n) | Fixed-length string |
| TEXT | Long text |
| DATE | Date only |
| TIME | Time only |
| TIMESTAMP | Date + time |
| BOOLEAN | True/False |

---

## 1.4 Constraints

```sql
CREATE TABLE departments (
    dept_id INT PRIMARY KEY,
    dept_name VARCHAR(50) UNIQUE NOT NULL
);

CREATE TABLE employees (
    emp_id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    age INT CHECK (age >= 18),
    dept_id INT,
    salary DECIMAL(10,2) DEFAULT 0,
    FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
);
```

- **PRIMARY KEY** = UNIQUE + NOT NULL, uniquely identifies a row.
- **FOREIGN KEY** = enforces referential integrity between tables.
- **UNIQUE** = no duplicate values.
- **NOT NULL** = column must have a value.
- **CHECK** = custom condition.
- **DEFAULT** = fallback value.

---

## 1.5 Inserting, Updating, Deleting Data (DML)

```sql
INSERT INTO employees (emp_id, name, salary, dept_id)
VALUES (1, 'Aman', 55000, 1);

INSERT INTO employees VALUES
(2, 'Riya', 60000, 2),
(3, 'Kabir', 48000, 1);

UPDATE employees SET salary = salary * 1.10 WHERE dept_id = 1;

DELETE FROM employees WHERE emp_id = 3;
```

---

## 1.6 SELECT & Filtering

```sql
SELECT name, salary FROM employees;
SELECT * FROM employees WHERE salary > 50000;
SELECT DISTINCT dept_id FROM employees;

SELECT * FROM employees WHERE salary BETWEEN 40000 AND 60000;
SELECT * FROM employees WHERE dept_id IN (1, 2);
SELECT * FROM employees WHERE name LIKE 'A%';      -- starts with A
SELECT * FROM employees WHERE name LIKE '%an%';    -- contains 'an'
SELECT * FROM employees WHERE dept_id IS NULL;
SELECT * FROM employees WHERE salary > 40000 AND dept_id = 1;

SELECT * FROM employees ORDER BY salary DESC;
SELECT * FROM employees ORDER BY dept_id ASC, salary DESC;
SELECT * FROM employees LIMIT 5 OFFSET 10;   -- pagination
```

### Logical & Comparison Operators
`=`, `!=` / `<>`, `>`, `<`, `>=`, `<=`, `AND`, `OR`, `NOT`, `BETWEEN`, `IN`, `LIKE`, `IS NULL`

---

## 1.7 Aggregate Functions + GROUP BY + HAVING

```sql
SELECT COUNT(*) FROM employees;
SELECT AVG(salary), MAX(salary), MIN(salary), SUM(salary) FROM employees;

SELECT dept_id, COUNT(*) AS emp_count, AVG(salary) AS avg_sal
FROM employees
GROUP BY dept_id;

-- HAVING filters groups (WHERE filters rows before grouping)
SELECT dept_id, AVG(salary) AS avg_sal
FROM employees
GROUP BY dept_id
HAVING AVG(salary) > 50000;
```

**Order of execution (very important to memorize):**
```
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

---

## 1.8 Joins (the heart of SQL)

Sample tables: `employees(emp_id, name, dept_id)`, `departments(dept_id, dept_name)`

```sql
-- INNER JOIN: only matching rows in both tables
SELECT e.name, d.dept_name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id;

-- LEFT JOIN: all rows from left + matched from right (NULL if no match)
SELECT e.name, d.dept_name
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id;

-- RIGHT JOIN: all rows from right + matched from left
SELECT e.name, d.dept_name
FROM employees e
RIGHT JOIN departments d ON e.dept_id = d.dept_id;

-- FULL OUTER JOIN: all rows from both, NULLs where no match
SELECT e.name, d.dept_name
FROM employees e
FULL OUTER JOIN departments d ON e.dept_id = d.dept_id;

-- CROSS JOIN: cartesian product (every row x every row)
SELECT e.name, d.dept_name FROM employees e CROSS JOIN departments d;

-- SELF JOIN: table joined with itself (e.g., find employees with same manager)
SELECT a.name AS emp, b.name AS manager
FROM employees a
JOIN employees b ON a.manager_id = b.emp_id;
```

**Visual mental model:**
- INNER JOIN → intersection
- LEFT JOIN → all of left + overlap
- RIGHT JOIN → all of right + overlap
- FULL JOIN → union of both

---

## 1.9 Subqueries

```sql
-- Scalar subquery
SELECT name FROM employees WHERE salary > (SELECT AVG(salary) FROM employees);

-- Subquery with IN
SELECT name FROM employees WHERE dept_id IN (SELECT dept_id FROM departments WHERE dept_name = 'IT');

-- Correlated subquery (runs once per outer row)
SELECT e.name, e.salary
FROM employees e
WHERE e.salary > (SELECT AVG(salary) FROM employees WHERE dept_id = e.dept_id);

-- Subquery in FROM (derived table)
SELECT dept_id, avg_sal
FROM (SELECT dept_id, AVG(salary) AS avg_sal FROM employees GROUP BY dept_id) t
WHERE avg_sal > 50000;

-- EXISTS
SELECT name FROM departments d
WHERE EXISTS (SELECT 1 FROM employees e WHERE e.dept_id = d.dept_id);
```

---

## 1.10 Set Operations

```sql
SELECT name FROM employees_2023
UNION                      -- combines + removes duplicates
SELECT name FROM employees_2024;

SELECT name FROM employees_2023
UNION ALL                  -- combines, keeps duplicates (faster)
SELECT name FROM employees_2024;

SELECT name FROM employees_2023
INTERSECT                  -- common rows only
SELECT name FROM employees_2024;

SELECT name FROM employees_2023
EXCEPT                     -- rows in first, not in second (MINUS in Oracle)
SELECT name FROM employees_2024;
```
Rule: same number of columns, compatible data types, in the same order.

---

## 1.11 String, Date, Numeric & Conditional Functions

```sql
-- String
SELECT UPPER(name), LOWER(name), LENGTH(name), TRIM(name) FROM employees;
SELECT CONCAT(name, ' - ', dept_id) FROM employees;
SELECT SUBSTRING(name, 1, 3) FROM employees;
SELECT REPLACE(name, 'a', '@') FROM employees;

-- Date
SELECT CURRENT_DATE, CURRENT_TIMESTAMP;
SELECT EXTRACT(YEAR FROM hire_date) FROM employees;
SELECT AGE(CURRENT_DATE, hire_date) FROM employees;   -- PostgreSQL

-- Numeric
SELECT ROUND(salary, 2), CEIL(salary), FLOOR(salary), ABS(-5) FROM employees;

-- Conditional
SELECT name,
    CASE
        WHEN salary >= 60000 THEN 'High'
        WHEN salary >= 40000 THEN 'Medium'
        ELSE 'Low'
    END AS salary_band
FROM employees;

SELECT COALESCE(commission, 0) FROM employees;  -- first non-null value
SELECT NULLIF(a, b) FROM some_table;            -- NULL if a = b else a
```

---

## 1.12 Views

A **view** is a saved, reusable virtual table based on a query.

```sql
CREATE VIEW high_earners AS
SELECT name, salary, dept_id FROM employees WHERE salary > 60000;

SELECT * FROM high_earners;

CREATE OR REPLACE VIEW high_earners AS
SELECT name, salary FROM employees WHERE salary > 70000;

DROP VIEW high_earners;
```
Use case: simplify complex/repetitive queries, restrict column-level access, present a clean interface to reporting tools.

---

## 1.13 Indexes

An **index** speeds up data retrieval (like a book's index) at the cost of extra storage and slightly slower writes.

```sql
CREATE INDEX idx_emp_dept ON employees(dept_id);
CREATE UNIQUE INDEX idx_emp_email ON employees(email);
DROP INDEX idx_emp_dept;
```
Use indexes on columns frequently used in `WHERE`, `JOIN`, `ORDER BY`. Avoid over-indexing — it slows down `INSERT`/`UPDATE`/`DELETE`.

---

## 1.14 Transactions & ACID

**ACID:**
- **Atomicity** — all or nothing
- **Consistency** — DB moves from one valid state to another
- **Isolation** — concurrent transactions don't interfere
- **Durability** — once committed, changes survive crashes

```sql
BEGIN;                      -- start transaction
UPDATE accounts SET balance = balance - 500 WHERE id = 1;
UPDATE accounts SET balance = balance + 500 WHERE id = 2;
COMMIT;                     -- save permanently

-- If something goes wrong:
ROLLBACK;

-- Partial rollback using savepoints
BEGIN;
UPDATE accounts SET balance = balance - 500 WHERE id = 1;
SAVEPOINT sp1;
UPDATE accounts SET balance = balance + 500 WHERE id = 2;
ROLLBACK TO sp1;
COMMIT;
```
Classic use case: bank transfers — debit and credit must both happen, or neither.

---

## 1.15 Normalization (Database Design)

Reduces redundancy and avoids anomalies.

- **1NF** — atomic values, no repeating groups.
- **2NF** — 1NF + no partial dependency on part of a composite key.
- **3NF** — 2NF + no transitive dependency (non-key columns depend only on the key).
- **BCNF** — stricter version of 3NF.

**Denormalization** = intentionally adding redundancy for read performance (common in reporting/analytics/data warehouses).

---

## 1.16 Window Functions (Advanced but essential)

Unlike GROUP BY, window functions **don't collapse rows** — they compute across a "window" of rows related to the current row.

```sql
SELECT name, dept_id, salary,
    RANK()       OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk,
    DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS dense_rnk,
    ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS row_num,
    SUM(salary)  OVER (PARTITION BY dept_id) AS dept_total,
    AVG(salary)  OVER (PARTITION BY dept_id) AS dept_avg,
    LAG(salary)  OVER (ORDER BY salary) AS prev_salary,
    LEAD(salary) OVER (ORDER BY salary) AS next_salary
FROM employees;
```
- `RANK()` — skips ranks after ties (1,1,3)
- `DENSE_RANK()` — no skipping (1,1,2)
- `ROW_NUMBER()` — unique sequential number, ignores ties
- `LAG`/`LEAD` — access previous/next row's value
- Common use cases: top-N per group, running totals, moving averages, year-over-year comparison.

```sql
-- Top 2 earners per department
SELECT * FROM (
    SELECT name, dept_id, salary,
           ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rn
    FROM employees
) t
WHERE rn <= 2;
```

---

## 1.17 Common Table Expressions (CTE) — `WITH` clause

```sql
WITH dept_avg AS (
    SELECT dept_id, AVG(salary) AS avg_sal
    FROM employees
    GROUP BY dept_id
)
SELECT e.name, e.salary, d.avg_sal
FROM employees e
JOIN dept_avg d ON e.dept_id = d.dept_id
WHERE e.salary > d.avg_sal;
```
CTEs improve readability vs nested subqueries, and can be **recursive** (great for hierarchies like org charts, category trees):

```sql
WITH RECURSIVE org_chart AS (
    SELECT emp_id, name, manager_id, 1 AS level
    FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.emp_id, e.name, e.manager_id, oc.level + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.emp_id
)
SELECT * FROM org_chart;
```

---

## 1.18 Stored Procedures, Functions & Triggers (concept — standard SQL)

- **Function** — returns a value, can be used inside a query.
- **Stored Procedure** — performs actions (may or may not return a value), called explicitly.
- **Trigger** — code that auto-runs on INSERT/UPDATE/DELETE events.

(Full syntax with real code is in the PostgreSQL PL/pgSQL section below.)

---

## 1.19 Keys — Summary

| Key | Meaning |
|---|---|
| Primary Key | Uniquely identifies each row |
| Foreign Key | Links to another table's primary key |
| Composite Key | Primary key made of 2+ columns |
| Candidate Key | Any column(s) that could be a primary key |
| Super Key | Any set of columns that uniquely identifies a row |
| Alternate Key | Candidate key not chosen as primary |

---

# PART 2: POSTGRESQL — DEEP DIVE

## 2.1 Why PostgreSQL

PostgreSQL is a free, open-source **object-relational** database known for standards compliance, extensibility, and advanced features: JSONB, arrays, full-text search, custom types, powerful indexing (GIN/GiST/BRIN), MVCC concurrency, and a rich extension ecosystem (PostGIS, TimescaleDB, pgvector).

---

## 2.2 Getting Started — psql (command-line client)

```bash
psql -U postgres -d mydb -h localhost -p 5432
```

Useful `psql` meta-commands:
```
\l              -- list databases
\c dbname       -- connect to database
\dt             -- list tables
\d tablename    -- describe table structure
\du             -- list roles/users
\dn             -- list schemas
\df             -- list functions
\di             -- list indexes
\x              -- toggle expanded display (great for wide rows)
\timing         -- show query execution time
\q              -- quit
```

---

## 2.3 PostgreSQL-Specific Data Types

| Type | Description | Example |
|---|---|---|
| `SERIAL` / `BIGSERIAL` | Auto-incrementing integer | `id SERIAL PRIMARY KEY` |
| `UUID` | Universally unique identifier | via `gen_random_uuid()` |
| `TEXT` | Unlimited-length string | — |
| `JSON` | Text-stored JSON | — |
| `JSONB` | Binary JSON — indexable, faster | preferred over JSON |
| `ARRAY` | Array of any type | `INT[]`, `TEXT[]` |
| `BOOLEAN` | true/false | — |
| `INTERVAL` | Time span | `'2 days'::interval` |
| `INET` / `CIDR` | IP addresses | networking data |
| `HSTORE` | Key-value pairs (extension) | — |
| `ENUM` | Custom fixed set of values | `CREATE TYPE mood AS ENUM (...)` |
| `TSVECTOR` | Text search document | full-text search |

```sql
CREATE TABLE users (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    name TEXT,
    tags TEXT[],
    metadata JSONB,
    mood mood_enum
);
```

---

## 2.4 Sequences & Identity Columns

```sql
CREATE SEQUENCE order_seq START 1000 INCREMENT 1;
SELECT nextval('order_seq');
SELECT currval('order_seq');

-- Modern preferred way (SQL standard, PG 10+)
CREATE TABLE orders (
    order_id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    order_date DATE DEFAULT CURRENT_DATE
);
```
`GENERATED ALWAYS AS IDENTITY` is preferred over `SERIAL` in modern PostgreSQL (more standards-compliant, safer with permissions).

---

## 2.5 Schemas

A schema is a namespace inside a database to organize tables logically (e.g., `sales`, `hr`).

```sql
CREATE SCHEMA sales;
CREATE TABLE sales.orders (id SERIAL PRIMARY KEY, amount NUMERIC);
SET search_path TO sales, public;   -- default schema lookup order
```
Use case: multi-tenant apps, separating modules within one database.

---

## 2.6 JSON / JSONB — Working with Semi-Structured Data

```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT,
    details JSONB
);

INSERT INTO products (name, details) VALUES
('Phone', '{"brand": "Samsung", "specs": {"ram": "8GB", "storage": "128GB"}}');

-- Query JSON fields
SELECT details->>'brand' AS brand FROM products;             -- text
SELECT details->'specs'->>'ram' AS ram FROM products;        -- nested
SELECT * FROM products WHERE details @> '{"brand": "Samsung"}';  -- contains

-- Indexing JSONB for fast lookups
CREATE INDEX idx_details ON products USING GIN (details);
```
Use case: storing flexible/variable attributes (e.g., product specs, event logs, API payloads) without rigid schema changes.

---

## 2.7 Arrays

```sql
CREATE TABLE posts (id SERIAL PRIMARY KEY, tags TEXT[]);
INSERT INTO posts (tags) VALUES (ARRAY['sql', 'postgres', 'database']);

SELECT * FROM posts WHERE 'sql' = ANY(tags);
SELECT unnest(tags) FROM posts;   -- expand array into rows
```

---

## 2.8 Full-Text Search

```sql
ALTER TABLE articles ADD COLUMN search_vector TSVECTOR;
UPDATE articles SET search_vector = to_tsvector('english', title || ' ' || body);
CREATE INDEX idx_search ON articles USING GIN(search_vector);

SELECT * FROM articles
WHERE search_vector @@ to_tsquery('english', 'database & performance');
```
Use case: search-engine-like functionality directly inside Postgres — no need for Elasticsearch in smaller apps.

---

## 2.9 Advanced Indexing

| Index Type | Best For |
|---|---|
| **B-Tree** (default) | Equality & range queries (`=`, `<`, `>`, `BETWEEN`) |
| **Hash** | Simple equality lookups |
| **GIN** (Generalized Inverted Index) | JSONB, arrays, full-text search |
| **GiST** | Geometric data, ranges, nearest-neighbor |
| **BRIN** (Block Range Index) | Very large, naturally ordered tables (e.g., timestamps in logs) |
| **Partial Index** | Index a subset of rows |
| **Composite/Multi-column Index** | Queries filtering on multiple columns together |

```sql
CREATE INDEX idx_partial ON orders(customer_id) WHERE status = 'pending';
CREATE INDEX idx_composite ON employees(dept_id, salary);
```

---

## 2.10 EXPLAIN / EXPLAIN ANALYZE — Query Tuning

```sql
EXPLAIN SELECT * FROM employees WHERE dept_id = 3;
EXPLAIN ANALYZE SELECT * FROM employees WHERE dept_id = 3;
```
- `EXPLAIN` shows the planned execution strategy (Seq Scan vs Index Scan, estimated cost).
- `EXPLAIN ANALYZE` actually runs the query and shows real timing.
- Look for: **Seq Scan** on large tables (often a red flag → consider an index), high **cost**, or nested loops on huge row counts.

---

## 2.11 MVCC & Locking

PostgreSQL uses **Multi-Version Concurrency Control (MVCC)**: readers never block writers and writers never block readers — each transaction sees a consistent snapshot.

**Isolation levels:**
```sql
BEGIN ISOLATION LEVEL READ COMMITTED;   -- default
BEGIN ISOLATION LEVEL REPEATABLE READ;
BEGIN ISOLATION LEVEL SERIALIZABLE;     -- strictest
```
- `READ COMMITTED` — sees only committed data at the start of each statement.
- `REPEATABLE READ` — consistent snapshot for the whole transaction.
- `SERIALIZABLE` — transactions behave as if executed one at a time.

**Row locking:**
```sql
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;  -- lock row until commit
```

**VACUUM** — reclaims space from dead rows (result of MVCC's row versioning):
```sql
VACUUM employees;
VACUUM ANALYZE employees;   -- also updates planner statistics
VACUUM FULL employees;      -- rewrites table, reclaims disk, locks table
```

---

## 2.12 PL/pgSQL — Functions, Procedures, Triggers

### Function
```sql
CREATE OR REPLACE FUNCTION get_dept_avg_salary(d_id INT)
RETURNS NUMERIC AS $$
DECLARE
    avg_sal NUMERIC;
BEGIN
    SELECT AVG(salary) INTO avg_sal FROM employees WHERE dept_id = d_id;
    RETURN avg_sal;
END;
$$ LANGUAGE plpgsql;

SELECT get_dept_avg_salary(1);
```

### Stored Procedure
```sql
CREATE OR REPLACE PROCEDURE give_raise(emp_id INT, amount NUMERIC)
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE employees SET salary = salary + amount WHERE emp_id = emp_id;
    COMMIT;
END;
$$;

CALL give_raise(1, 5000);
```

### Trigger
```sql
CREATE TABLE employee_audit (
    id SERIAL PRIMARY KEY,
    emp_id INT,
    old_salary NUMERIC,
    new_salary NUMERIC,
    changed_at TIMESTAMP DEFAULT NOW()
);

CREATE OR REPLACE FUNCTION log_salary_change()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.salary <> OLD.salary THEN
        INSERT INTO employee_audit(emp_id, old_salary, new_salary)
        VALUES (OLD.emp_id, OLD.salary, NEW.salary);
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_salary_audit
AFTER UPDATE ON employees
FOR EACH ROW
EXECUTE FUNCTION log_salary_change();
```
Use case: automatic audit trails, enforcing complex business rules, cascading updates, notifications.

---

## 2.13 Partitioning (for very large tables)

```sql
CREATE TABLE sales (
    id SERIAL,
    sale_date DATE NOT NULL,
    amount NUMERIC
) PARTITION BY RANGE (sale_date);

CREATE TABLE sales_2024 PARTITION OF sales
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE sales_2025 PARTITION OF sales
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
```
Use case: huge tables (billions of rows) like logs, transactions, IoT data — queries hitting one partition are much faster (partition pruning), and old partitions can be dropped instantly instead of slow DELETEs.

---

## 2.14 Roles & Permissions

```sql
CREATE ROLE analyst WITH LOGIN PASSWORD 'secret';
GRANT CONNECT ON DATABASE company TO analyst;
GRANT SELECT ON employees TO analyst;
GRANT INSERT, UPDATE ON employees TO analyst;
REVOKE INSERT ON employees FROM analyst;

CREATE ROLE readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;
GRANT readonly TO analyst;   -- role inheritance
```

---

## 2.15 Backup & Restore

```bash
pg_dump -U postgres -d company > company_backup.sql
psql -U postgres -d company < company_backup.sql

# Custom compressed format (recommended for large DBs)
pg_dump -U postgres -Fc company > company.dump
pg_restore -U postgres -d company company.dump

# Backup entire cluster (all databases)
pg_dumpall -U postgres > all_databases.sql
```

---

## 2.16 Replication & High Availability (concept overview)

- **Streaming Replication** — a primary server streams WAL (Write-Ahead Log) changes to one or more standby (replica) servers in near real-time.
- **Physical Replication** — byte-for-byte copy of the database.
- **Logical Replication** — replicates specific tables/changes, allows different Postgres versions or selective sync.
- Use case: read scaling (route SELECTs to replicas), disaster recovery, zero-downtime failover.

---

## 2.17 Extensions

PostgreSQL's superpower — install optional capabilities:
```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";     -- UUID generation
CREATE EXTENSION IF NOT EXISTS pgcrypto;        -- encryption, gen_random_uuid()
CREATE EXTENSION IF NOT EXISTS postgis;         -- geospatial data
CREATE EXTENSION IF NOT EXISTS pg_trgm;         -- fuzzy text search / similarity
CREATE EXTENSION IF NOT EXISTS vector;          -- pgvector: AI embeddings/similarity search
```

---

## 2.18 Common Real-World Use Cases (map of "what to use when")

| Requirement | PostgreSQL Feature |
|---|---|
| Flexible/variable schema (product catalogs, configs) | `JSONB` + GIN index |
| Search text like Google | Full-text search (`tsvector`/`tsquery`) or `pg_trgm` |
| Fast lookups on huge log tables ordered by time | `BRIN` index + partitioning |
| Org chart / category tree / comments hierarchy | `WITH RECURSIVE` CTE |
| Leaderboard / top-N per group | Window functions (`ROW_NUMBER`, `RANK`) |
| Financial transactions (money moving between accounts) | Transactions with `BEGIN/COMMIT/ROLLBACK`, `SERIALIZABLE` isolation |
| Audit trail of changes | Triggers + audit table |
| Multi-tenant SaaS app | Schemas per tenant, or `tenant_id` column + Row-Level Security |
| Geolocation (nearby stores, delivery zones) | PostGIS extension |
| AI similarity search (recommendations, embeddings) | `pgvector` extension |
| Reporting dashboards on huge historical data | Partitioning + materialized views |
| Read-heavy app needing to scale reads | Streaming replication + read replicas |
| Duplicate-safe imports | `INSERT ... ON CONFLICT DO NOTHING/UPDATE` (upsert) |

### Upsert (very common in real apps)
```sql
INSERT INTO users (id, email, login_count)
VALUES (1, 'a@x.com', 1)
ON CONFLICT (id)
DO UPDATE SET login_count = users.login_count + 1;
```

### Materialized Views (precomputed, cached query results)
```sql
CREATE MATERIALIZED VIEW dept_summary AS
SELECT dept_id, COUNT(*), AVG(salary) FROM employees GROUP BY dept_id;

REFRESH MATERIALIZED VIEW dept_summary;   -- manually refresh when data changes
```
Use case: expensive aggregations for dashboards that don't need to be real-time.

### Row-Level Security (RLS) — multi-tenant data isolation
```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.current_tenant')::INT);
```

---

## 2.19 Performance Best Practices Checklist

1. Index columns used in `WHERE`, `JOIN`, `ORDER BY` — but don't over-index.
2. Use `EXPLAIN ANALYZE` before optimizing blindly.
3. Prefer `JSONB` over `JSON`.
4. Use connection pooling (e.g., PgBouncer) for high-concurrency apps.
5. Run `VACUUM ANALYZE` regularly (or trust autovacuum, but monitor it).
6. Avoid `SELECT *` in production queries — fetch only needed columns.
7. Batch large `INSERT`/`UPDATE` operations instead of row-by-row.
8. Use partitioning for tables expected to grow into hundreds of millions of rows.
9. Use `LIMIT` + keyset pagination (`WHERE id > last_id`) instead of large `OFFSET` for deep pagination.
10. Watch out for N+1 query patterns from application code — use JOINs or batched IN queries instead.

---

## 2.20 Suggested Learning Path (Day-by-Day Style)

1. **Basics**: SELECT, WHERE, ORDER BY, DISTINCT, LIMIT
2. **Data manipulation**: INSERT, UPDATE, DELETE, constraints
3. **Aggregation**: GROUP BY, HAVING, aggregate functions
4. **Joins**: INNER, LEFT, RIGHT, FULL, SELF, CROSS
5. **Subqueries & Set operations**
6. **Views & Indexes**
7. **Transactions & ACID**
8. **Window functions & CTEs** (this is where "SQL for interviews/analytics" really lives)
9. **PostgreSQL specifics**: JSONB, arrays, schemas, sequences
10. **PL/pgSQL**: functions, procedures, triggers
11. **Performance**: EXPLAIN ANALYZE, indexing strategy, VACUUM, MVCC
12. **Scale topics**: partitioning, replication, extensions, RLS
13. **Practice**: solve problems on LeetCode SQL, HackerRank SQL, and rebuild a small real app schema (e.g., e-commerce, blog, bank) end-to-end.

---

*Keep this file as your master reference — update it with new things you learn (edge cases, gotchas, query patterns) as you go through practice problems and real projects.*
