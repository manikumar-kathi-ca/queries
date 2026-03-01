# PostgreSQL SQL Reference — Joins, Aggregates & Window Functions

## Sample Tables Used Throughout

```sql
-- USERS
SELECT * FROM users;
```
| id | name    | city    |
|----|---------|---------|
| 1  | Alice   | NYC     |
| 2  | Bob     | Austin  |
| 3  | Charlie | Chicago |
| 4  | Diana   | NYC     |
| 5  | Eve     | Austin  |

```sql
-- ORDERS
SELECT * FROM orders;
```
| id | user_id | amount  | status    |
|----|---------|---------|-----------|
| 1  | 1       | 999.99  | shipped   |
| 2  | 1       | 499.99  | shipped   |
| 3  | 2       | 199.99  | pending   |
| 4  | 2       | 299.99  | shipped   |
| 5  | 3       | 999.99  | cancelled |
| 6  | 3       | 239.97  | shipped   |
| 7  | 4       | 499.99  | pending   |
| 8  | 4       | 599.98  | shipped   |

> Note: User 5 (Eve) has **no orders** — useful for LEFT JOIN demos.

---

## 1. INNER JOIN

> Returns only rows that have a match in **both** tables.

```sql
SELECT u.name, o.amount, o.status
FROM users u
JOIN orders o ON u.id = o.user_id;
```

| name    | amount  | status    |
|---------|---------|-----------|
| Alice   | 999.99  | shipped   |
| Alice   | 499.99  | shipped   |
| Bob     | 199.99  | pending   |
| Bob     | 299.99  | shipped   |
| Charlie | 999.99  | cancelled |
| Charlie | 239.97  | shipped   |
| Diana   | 499.99  | pending   |
| Diana   | 599.98  | shipped   |

> ⚠️ Eve is **missing** — she has no orders, so INNER JOIN drops her.

---

## 2. LEFT JOIN

> Returns **all rows from left table**, with NULLs where no match exists in right table.

```sql
SELECT u.name, o.amount, o.status
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;
```

| name    | amount  | status    |
|---------|---------|-----------|
| Alice   | 999.99  | shipped   |
| Alice   | 499.99  | shipped   |
| Bob     | 199.99  | pending   |
| Bob     | 299.99  | shipped   |
| Charlie | 999.99  | cancelled |
| Charlie | 239.97  | shipped   |
| Diana   | 499.99  | pending   |
| Diana   | 599.98  | shipped   |
| Eve     | NULL    | NULL      |

> ✅ Eve is **included** with NULLs — she has no orders but still appears.

---

## 3. Find Users With NO Orders

> Classic interview question — LEFT JOIN + IS NULL.

```sql
SELECT u.name
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.id IS NULL;
```

| name |
|------|
| Eve  |

---

## 4. GROUP BY — Total Per User

> Collapses rows into one summary row per group.

```sql
SELECT u.name,
       COUNT(o.id)   AS total_orders,
       SUM(o.amount) AS total_spent
FROM users u
JOIN orders o ON u.id = o.user_id
GROUP BY u.name
ORDER BY total_spent DESC;
```

| name    | total_orders | total_spent |
|---------|--------------|-------------|
| Alice   | 2            | 1499.98     |
| Diana   | 2            | 1099.97     |
| Charlie | 2            | 1239.96     |
| Bob     | 2            | 499.98      |

---

## 5. HAVING — Filter After GROUP BY

> Like WHERE but runs **after** aggregation.

```sql
SELECT u.name, SUM(o.amount) AS total_spent
FROM users u
JOIN orders o ON u.id = o.user_id
GROUP BY u.name
HAVING SUM(o.amount) > 1000
ORDER BY total_spent DESC;
```

| name    | total_spent |
|---------|-------------|
| Alice   | 1499.98     |
| Charlie | 1239.96     |
| Diana   | 1099.97     |

> Only users who spent **more than $1000** appear.

---

## 6. Subquery

> Query inside a query — find users who ordered the most expensive product.

```sql
SELECT u.name
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE o.amount = (SELECT MAX(amount) FROM orders);
```

| name    |
|---------|
| Alice   |
| Charlie |

---

## 7. CTE (Common Table Expression)

> Cleaner alternative to subqueries — define named temp results with WITH.

```sql
WITH top_spenders AS (
    SELECT user_id, SUM(amount) AS total
    FROM orders
    GROUP BY user_id
    HAVING SUM(amount) > 1000
)
SELECT u.name, t.total
FROM users u
JOIN top_spenders t ON u.id = t.user_id
ORDER BY t.total DESC;
```

| name    | total   |
|---------|---------|
| Alice   | 1499.98 |
| Charlie | 1239.96 |
| Diana   | 1099.97 |

---

## 8. WINDOW FUNCTION — SUM() OVER PARTITION BY

> Calculate totals **per group** without collapsing rows.

```sql
SELECT
    u.name,
    o.amount,
    SUM(o.amount) OVER (PARTITION BY u.name) AS user_total
FROM users u
JOIN orders o ON u.id = o.user_id;
```

| name    | amount  | user_total |
|---------|---------|------------|
| Alice   | 999.99  | 1499.98    |
| Alice   | 499.99  | 1499.98    |
| Bob     | 199.99  | 499.98     |
| Bob     | 299.99  | 499.98     |
| Charlie | 999.99  | 1239.96    |
| Charlie | 239.97  | 1239.96    |
| Diana   | 499.99  | 1099.97    |
| Diana   | 599.98  | 1099.97    |

> ✅ Individual rows are **kept** — unlike GROUP BY which collapses them.

---

## 9. RANK() — Rank All Users by Spending

```sql
SELECT
    u.name,
    SUM(o.amount)                             AS total_spent,
    RANK() OVER (ORDER BY SUM(o.amount) DESC) AS rank
FROM users u
JOIN orders o ON u.id = o.user_id
GROUP BY u.name;
```

| name    | total_spent | rank |
|---------|-------------|------|
| Alice   | 1499.98     | 1    |
| Charlie | 1239.96     | 2    |
| Diana   | 1099.97     | 3    |
| Bob     | 499.98      | 4    |

---

## 10. RANK() with PARTITION BY — Rank Within City

```sql
SELECT
    u.name,
    u.city,
    SUM(o.amount) AS total_spent,
    RANK() OVER (
        PARTITION BY u.city
        ORDER BY SUM(o.amount) DESC
    ) AS city_rank
FROM users u
JOIN orders o ON u.id = o.user_id
GROUP BY u.name, u.city;
```

| name    | city    | total_spent | city_rank |
|---------|---------|-------------|-----------|
| Alice   | NYC     | 1499.98     | 1         |
| Diana   | NYC     | 1099.97     | 2         |
| Bob     | Austin  | 499.98      | 1         |
| Charlie | Chicago | 1239.96     | 1         |

> Rank **resets** for each city — Alice and Bob are both rank 1 in their cities.

---

## 11. Running Total — SUM() OVER ORDER BY

```sql
SELECT
    id,
    amount,
    SUM(amount) OVER (ORDER BY id) AS running_total
FROM orders;
```

| id | amount  | running_total |
|----|---------|---------------|
| 1  | 999.99  | 999.99        |
| 2  | 499.99  | 1499.98       |
| 3  | 199.99  | 1699.97       |
| 4  | 299.99  | 1999.96       |
| 5  | 999.99  | 2999.95       |
| 6  | 239.97  | 3239.92       |
| 7  | 499.99  | 3739.91       |
| 8  | 599.98  | 4339.89       |

---

## 12. LAG() — Compare With Previous Row

```sql
SELECT
    id,
    amount,
    LAG(amount) OVER (ORDER BY id)              AS prev_amount,
    amount - LAG(amount) OVER (ORDER BY id)     AS diff
FROM orders;
```

| id | amount  | prev_amount | diff    |
|----|---------|-------------|---------|
| 1  | 999.99  | NULL        | NULL    |
| 2  | 499.99  | 999.99      | -500.00 |
| 3  | 199.99  | 499.99      | -300.00 |
| 4  | 299.99  | 199.99      | +100.00 |
| 5  | 999.99  | 299.99      | +700.00 |

---

## 13. ROW_NUMBER vs RANK vs DENSE_RANK

> Show the difference when there are **ties**.

```sql
SELECT
    u.name,
    SUM(o.amount)                                     AS total,
    ROW_NUMBER() OVER (ORDER BY SUM(o.amount) DESC)   AS row_num,
    RANK()       OVER (ORDER BY SUM(o.amount) DESC)   AS rank,
    DENSE_RANK() OVER (ORDER BY SUM(o.amount) DESC)   AS dense_rank
FROM users u
JOIN orders o ON u.id = o.user_id
GROUP BY u.name;
```

| name    | total   | row_num | rank | dense_rank |
|---------|---------|---------|------|------------|
| Alice   | 1499.98 | 1       | 1    | 1          |
| Charlie | 1239.96 | 2       | 2    | 2          |
| Diana   | 1099.97 | 3       | 3    | 3          |
| Bob     | 499.98  | 4       | 4    | 4          |

| Function       | On Tie          | Example    |
|----------------|-----------------|------------|
| `ROW_NUMBER()` | Always unique   | 1, 2, 3, 4 |
| `RANK()`       | Skips after tie | 1, 1, 3, 4 |
| `DENSE_RANK()` | No skip on tie  | 1, 1, 2, 3 |

---

## 14. NTILE() — Split Into Buckets (Quartiles)

```sql
SELECT
    u.name,
    SUM(o.amount)                              AS total_spent,
    NTILE(4) OVER (ORDER BY SUM(o.amount))     AS quartile
FROM users u
JOIN orders o ON u.id = o.user_id
GROUP BY u.name;
```

| name    | total_spent | quartile |
|---------|-------------|----------|
| Bob     | 499.98      | 1        |
| Diana   | 1099.97     | 2        |
| Charlie | 1239.96     | 3        |
| Alice   | 1499.98     | 4        |

---

## 15. EXPLAIN ANALYZE — See Query Cost

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE user_id = 1;
```

```
Seq Scan on orders  (cost=0.00..1.10 rows=2 width=56)
                     actual time=0.012..0.015 rows=2 loops=1
  Filter: (user_id = 1)
  Rows Removed by Filter: 6
  Buffers: shared hit=1
Planning Time:  0.05 ms
Execution Time: 0.03 ms
```

```sql
-- Add index then re-run
CREATE INDEX idx_orders_user_id ON orders(user_id);

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE user_id = 1;
```

```
Index Scan using idx_orders_user_id on orders
                     (cost=0.13..8.15 rows=2 width=56)
                      actual time=0.020..0.022 rows=2 loops=1
  Buffers: shared hit=2
Planning Time:  0.2 ms
Execution Time: 0.04 ms
```

---

## Quick Reference — All Concepts

| Concept          | Query Pattern                                      | When to Use                              |
|------------------|----------------------------------------------------|------------------------------------------|
| INNER JOIN       | `JOIN t ON a.id = t.fk`                            | Only matched rows from both tables       |
| LEFT JOIN        | `LEFT JOIN t ON a.id = t.fk`                       | All left rows, NULLs where no match      |
| No match filter  | `LEFT JOIN ... WHERE t.id IS NULL`                 | Find orphan rows (users with no orders)  |
| GROUP BY         | `GROUP BY col`                                     | Collapse rows into summary               |
| HAVING           | `HAVING SUM(...) > n`                              | Filter after aggregation                 |
| Subquery         | `WHERE col = (SELECT ...)`                         | Single value lookup inside query         |
| CTE              | `WITH name AS (SELECT ...) SELECT ...`             | Named reusable temp result               |
| SUM OVER         | `SUM(col) OVER (PARTITION BY x)`                   | Group totals without collapsing rows     |
| Running total    | `SUM(col) OVER (ORDER BY id)`                      | Cumulative sum row by row                |
| RANK             | `RANK() OVER (ORDER BY col DESC)`                  | Rank with gaps on ties                   |
| DENSE_RANK       | `DENSE_RANK() OVER (ORDER BY col DESC)`            | Rank without gaps on ties                |
| ROW_NUMBER       | `ROW_NUMBER() OVER (ORDER BY col)`                 | Always unique sequential number          |
| PARTITION + RANK | `RANK() OVER (PARTITION BY x ORDER BY y)`          | Rank within each group separately        |
| LAG              | `LAG(col) OVER (ORDER BY id)`                      | Access previous row value                |
| LEAD             | `LEAD(col) OVER (ORDER BY id)`                     | Access next row value                    |
| NTILE            | `NTILE(4) OVER (ORDER BY col)`                     | Split rows into N equal buckets          |
| EXPLAIN ANALYZE  | `EXPLAIN (ANALYZE, BUFFERS) SELECT ...`            | See actual cost, time, cache hits        |
