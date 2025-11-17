
---

## Overview — Index Optimization & Maintenance

**Goal:**
You’ll learn to **measure**, **tune**, and **maintain** indexes for high-performance query-heavy systems.

---

### 📘 Topics Overview

| Topic                                   | Core Concepts                                             |
| --------------------------------------- | --------------------------------------------------------- |
| 1. Measuring Index Size & Bloat         | How to inspect index growth and space waste               |
| 2. Fillfactor Tuning                    | Controlling page utilization for update-heavy workloads   |
| 3. CLUSTER & REINDEX                    | Physically reorganizing data for locality and performance |
| 4. Index-only scans                     | Making indexes return results without touching the heap   |
| 5. VACUUM and autovacuum impact         | How to keep indexes healthy automatically                 |
| 6. Detecting and fixing bloated indexes | Hands-on diagnostics                                      |
| 7. Maintenance best practices           | Real-world tuning checklist                               |

---

## 🧩 1. Measuring Index Size & Bloat

Indexes can silently **grow larger than necessary** due to dead tuples and page splits.
You can inspect index sizes with:

```sql
SELECT
    relname AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE schemaname = 'public';
```

You can also measure *how much of that space is wasted* (bloat):

```sql
CREATE EXTENSION pgstattuple;

SELECT
    indexrelid::regclass AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size,
    ROUND(100 * stats.avg_leaf_density, 2) AS leaf_density
FROM pgstatindex('your_index_name');
```

🧠 Interpretation:

* **Low leaf density (<70%)** → means fragmentation.
* **High density (~90%)** → healthy and compact.

---

## 🧩 2. Fillfactor — Balancing Inserts vs Updates

When PostgreSQL creates an index, each page is filled close to **90%** by default.

```sql
CREATE INDEX idx_orders_user ON orders(user_id) WITH (fillfactor = 90);
```

🧠 Why it matters:

* **Lower fillfactor (e.g., 70)** leaves free space in each page → fewer splits during future inserts/updates.
* **Higher fillfactor (e.g., 95)** maximizes density → smaller index, but more page splits later.

📊 **Guideline:**

| Workload                    | Recommended Fillfactor |
| --------------------------- | ---------------------- |
| Read-heavy (mostly SELECTs) | 95–100                 |
| Insert/update-heavy         | 70–85                  |
| Random inserts              | 60–70                  |

You can check a table/index’s setting:

```sql
SELECT relname, reloptions FROM pg_class WHERE relname='idx_orders_user';
```

---

## 🧩 3. CLUSTER and REINDEX

### 🧱 REINDEX

Rebuilds a corrupted or bloated index:

```sql
REINDEX INDEX idx_orders_user;
```

or for all indexes in a table:

```sql
REINDEX TABLE orders;
```

This creates a fresh, compact version of the index — useful after mass deletes.

---

### 📦 CLUSTER

Physically **reorders table rows** on disk based on an index order:

```sql
CLUSTER orders USING idx_orders_user;
```

Benefits:

* Improves locality for queries filtering on that index.
* Reduces random I/O.
* Makes index-only scans more likely.

🧠 But: It locks the table exclusively while running.

---

## 🧩 4. Index-Only Scans

PostgreSQL can sometimes **serve a query entirely from the index** — skipping the heap lookup altogether.
This is called an **Index-Only Scan**.

Example:

```sql
CREATE INDEX idx_users_email ON users(email);
ANALYZE users;

EXPLAIN ANALYZE SELECT email FROM users WHERE email LIKE 'a%';
```

Output:

```
Index Only Scan using idx_users_email on users
```

✅ The planner uses only the index because:

* The query needs only `email` (which is stored in the index).
* All index entries are visible to the current snapshot (thanks to visibility map).

Visibility Map tracks which heap pages have *no dead tuples*.

You can inspect it:

```sql
SELECT * FROM pg_visibility_map('users');
```

If visibility map bits are not set → PostgreSQL must still touch heap pages, falling back to **Index Scan**.

---

## 🧩 5. VACUUM, Autovacuum & Index Health

Remember from earlier weeks:
When rows are updated/deleted → old tuples remain until vacuumed.

### Manual cleanup:

```sql
VACUUM ANALYZE;
```

### Automated (default):

Autovacuum runs in the background, cleaning up dead tuples.

You can monitor it:

```sql
SELECT relname, n_dead_tup, vacuum_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

💡 Indexes don’t shrink automatically — `VACUUM FULL` or `REINDEX` is required for that.

---

## 🧩 6. Detecting and Fixing Bloated Indexes (Hands-On)

Install diagnostic tools:

```sql
CREATE EXTENSION pgstattuple;
```

Then:

```sql
SELECT * FROM pgstattuple('your_index_name');
```

Example output:

```
table_len | tuple_count | dead_tuple_count | free_space
-----------+--------------+------------------+------------
327680    | 10000        | 1200             | 2048
```

🧠 **Interpretation:**

* `dead_tuple_count > 0` → indicates index bloat.
* High `free_space` → wasted storage.

Fix it:

```sql
REINDEX INDEX your_index_name;
```

---

## 🧩 7. Maintenance Best Practices (Production Level)

| Practice                                    | Why                                              |
| ------------------------------------------- | ------------------------------------------------ |
| Run `ANALYZE` regularly                     | Keeps planner’s stats fresh                      |
| Tune autovacuum thresholds                  | Prevents index bloat                             |
| Use lower fillfactor for write-heavy tables | Reduces page splits                              |
| Monitor index sizes monthly                 | Detect slow-growing bloat early                  |
| Drop unused indexes                         | Reduces write overhead                           |
| REINDEX periodically                        | Reclaims disk and rebuilds structure             |
| Consider `pg_repack`                        | Rebuilds tables/indexes **online** (no downtime) |

Check unused indexes:

```sql
SELECT
    relname AS index_name,
    idx_scan AS scans,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0;
```

---

## 🧠 Key Insights Recap

| Concept                       | Meaning                                 |
| ----------------------------- | --------------------------------------- |
| **Fillfactor**                | How full each page gets when created    |
| **Index-only scan**           | Avoids heap lookups entirely            |
| **Vacuum**                    | Cleans up dead tuples                   |
| **Reindex**                   | Physically rebuilds index               |
| **Cluster**                   | Reorders table by index                 |
| **Bloat**                     | Wasted space from dead tuples or splits |
| **pgstattuple / pgstatindex** | Tools to inspect health                 |

---

## 🧩 Hands-on Exercise

1. Create a large test table:

   ```sql
   CREATE TABLE orders AS
   SELECT i, (i % 1000)::int AS user_id, (random() * 1000)::int AS amount
   FROM generate_series(1, 100000) i;
   CREATE INDEX idx_orders_user ON orders(user_id);
   ANALYZE orders;
   ```

2. Measure index size:

   ```sql
   SELECT pg_size_pretty(pg_relation_size('idx_orders_user'));
   ```

3. Perform updates and deletes:

   ```sql
   UPDATE orders SET amount = amount + 1 WHERE user_id < 500;
   DELETE FROM orders WHERE user_id < 200;
   ```

4. Check bloat:

   ```sql
   SELECT * FROM pgstattuple('idx_orders_user');
   ```

5. Run:

   ```sql
   VACUUM ANALYZE;
   REINDEX INDEX idx_orders_user;
   ```

6. Compare sizes before and after.

---

## 🧩 Reflection Questions

1. Why doesn’t `VACUUM` immediately shrink index files?
2. When should you prefer `REINDEX` over `CLUSTER`?
3. What’s the trade-off of lowering fillfactor?
4. Why are index-only scans not always possible?

---

