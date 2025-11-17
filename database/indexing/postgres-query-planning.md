## **1. The PostgreSQL Query Planner (Optimizer)**

When you execute a query like:

```sql
SELECT * FROM users WHERE email = 'shawon@example.com';
```

PostgreSQL goes through these steps:

1. **Parser** – Converts SQL → parse tree.
2. **Planner/Optimizer** – Creates many possible query plans, estimates their cost, and picks the cheapest.
3. **Executor** – Runs the chosen plan.

Indexes come in **at the planning phase**. The optimizer decides:

* Should it use the index or do a sequential scan?
* Should it combine multiple indexes?
* Should it use bitmap scanning?

---

## **2. `EXPLAIN` and `EXPLAIN ANALYZE`**

Let’s use a simple example:

```sql
EXPLAIN SELECT * FROM users WHERE email = 'shawon@example.com';
```

### Output example:

```
Index Scan using idx_users_email on users  (cost=0.29..8.30 rows=1 width=72)
  Index Cond: (email = 'shawon@example.com')
```

Let’s break it down:

| Field             | Meaning                                                      |
| ----------------- | ------------------------------------------------------------ |
| `Index Scan`      | PostgreSQL used an index instead of scanning the whole table |
| `idx_users_email` | Name of the index used                                       |
| `cost=0.29..8.30` | Estimated startup cost and total cost                        |
| `rows=1`          | Estimated number of rows matching the condition              |
| `width=72`        | Estimated size (in bytes) of each row                        |
| `Index Cond`      | Condition that can be satisfied directly via the index       |

Now with:

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'shawon@example.com';
```

This *actually executes* the query and shows real execution time.

Example:

```
Index Scan using idx_users_email on users  (cost=0.29..8.30 rows=1 width=72)
  Actual time=0.040..0.043 rows=1 loops=1
  Index Cond: (email = 'shawon@example.com')
Planning Time: 0.065 ms
Execution Time: 0.082 ms
```

The difference between `cost` and `actual time` shows how accurate PostgreSQL’s statistics are.

---

## **3. How the Cost Is Estimated**

PostgreSQL stores **table and index statistics** via:

```sql
ANALYZE users;
```

It gathers:

* Number of rows (`n_tuples`)
* Number of distinct values per column (`n_distinct`)
* Null fraction
* Most common values and frequencies
* Correlation (ordering measure)

Then, during query planning:

* Sequential scan cost = proportional to table size.
* Index scan cost = proportional to tree depth + number of page fetches.
* If the planner thinks a large % of the table matches, it prefers a sequential scan.

👉 This is why **keeping statistics up-to-date is crucial**.
When data changes a lot, run:

```sql
VACUUM ANALYZE;
```

---

## **4. When PostgreSQL Chooses Index vs Seq Scan**

| Case                                                | What Happens                         |
| --------------------------------------------------- | ------------------------------------ |
| Small table                                         | Seq scan — cheaper to read all pages |
| Large table, low-selectivity column (like `gender`) | Seq scan — too many matches          |
| Highly selective condition (like `email`)           | Index scan — very efficient          |
| Condition with range (`BETWEEN`, `>`, `<`)          | May use index if selectivity is good |
| Function on column (`LOWER(name)`)                  | Needs functional index               |

---

## **5. Bitmap Index Scans**

When multiple rows match, PostgreSQL may use a **bitmap index scan**.

Example:

```sql
EXPLAIN SELECT * FROM orders WHERE total_amount > 1000;
```

Output:

```
Bitmap Heap Scan on orders  (cost=8.50..250.50 rows=300 width=84)
  Recheck Cond: (total_amount > 1000)
  ->  Bitmap Index Scan on idx_orders_total  (cost=0.00..8.00 rows=300 width=0)
        Index Cond: (total_amount > 1000)
```

🔹 The planner first collects all matching row locations into a bitmap
🔹 Then fetches table rows in physical order (reduces random I/O)
🔹 Useful for **range queries** or **multiple index conditions**

---

## **6. Hands-on Task for Week 3**

### **Setup**

Use your existing `users` and `orders` tables.

### **Tasks**

1. Run:

   ```sql
   EXPLAIN ANALYZE SELECT * FROM users WHERE id = 5000;
   ```

   Then:

   ```sql
   EXPLAIN ANALYZE SELECT * FROM users;
   ```

   → Compare index vs seq scan behavior.

2. Create an index on a low-selectivity column:

   ```sql
   CREATE INDEX idx_users_gender ON users(gender);
   EXPLAIN ANALYZE SELECT * FROM users WHERE gender = 'male';
   ```

   → Does PostgreSQL use it?

3. Try a range query:

   ```sql
   EXPLAIN ANALYZE SELECT * FROM orders WHERE total_amount > 2000;
   ```

   → Check if it uses a bitmap index scan.

4. Run:

   ```sql
   VACUUM ANALYZE;
   ```

   Then re-run your queries — notice any plan change?

---

## 🎯 **Learning Goals**

By the end of Week 3, you should:

* Read and interpret query plans confidently.
* Understand when PostgreSQL chooses index vs sequential scans.
* Know how statistics affect the optimizer’s decisions.
* Be comfortable experimenting with `EXPLAIN ANALYZE`.

---
