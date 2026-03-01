# MongoDB Query Reference — CRUD, Aggregation & Joins

## Sample Collections Used Throughout

```js
// users collection
db.users.find()
```
| _id | name    | city    | age |
|-----|---------|---------|-----|
| 1   | Alice   | NYC     | 30  |
| 2   | Bob     | Austin  | 25  |
| 3   | Charlie | Chicago | 35  |
| 4   | Diana   | NYC     | 28  |
| 5   | Eve     | Austin  | 22  |

```js
// orders collection
db.orders.find()
```
| _id | user_id | amount  | status    | product   |
|-----|---------|---------|-----------|-----------|
| 1   | 1       | 999.99  | shipped   | Laptop    |
| 2   | 1       | 499.99  | shipped   | Phone     |
| 3   | 2       | 199.99  | pending   | Chair     |
| 4   | 2       | 299.99  | shipped   | Monitor   |
| 5   | 3       | 999.99  | cancelled | Laptop    |
| 6   | 3       | 239.97  | shipped   | Keyboard  |
| 7   | 4       | 499.99  | pending   | Phone     |
| 8   | 4       | 599.98  | shipped   | Monitor   |

> Note: User 5 (Eve) has **no orders** — useful for lookup demos.

---

## 1. INSERT

```js
// insert one
db.users.insertOne({
  _id: 6,
  name: "Frank",
  city: "LA",
  age: 31
})
```
```
{ acknowledged: true, insertedId: 6 }
```

```js
// insert many
db.users.insertMany([
  { name: "Grace", city: "NYC", age: 27 },
  { name: "Hank",  city: "LA",  age: 40 }
])
```
```
{ acknowledged: true, insertedIds: { 0: ObjectId(...), 1: ObjectId(...) } }
```

---

## 2. FIND — Basic Queries

```js
// find all
db.users.find()

// find with filter
db.users.find({ city: "NYC" })
```
| _id | name  | city | age |
|-----|-------|------|-----|
| 1   | Alice | NYC  | 30  |
| 4   | Diana | NYC  | 28  |

```js
// find specific fields only (projection)
db.users.find({ city: "NYC" }, { name: 1, _id: 0 })
```
| name  |
|-------|
| Alice |
| Diana |

---

## 3. COMPARISON OPERATORS

```js
// age greater than 28
db.users.find({ age: { $gt: 28 } })
```
| _id | name    | city    | age |
|-----|---------|---------|-----|
| 1   | Alice   | NYC     | 30  |
| 3   | Charlie | Chicago | 35  |

```js
// amount between 200 and 600
db.orders.find({ amount: { $gte: 200, $lte: 600 } })
```
| _id | user_id | amount | status  |
|-----|---------|--------|---------|
| 4   | 2       | 299.99 | shipped |
| 7   | 4       | 499.99 | pending |
| 8   | 4       | 599.98 | shipped |

| Operator | Meaning               |
|----------|-----------------------|
| `$eq`    | equal                 |
| `$ne`    | not equal             |
| `$gt`    | greater than          |
| `$gte`   | greater than or equal |
| `$lt`    | less than             |
| `$lte`   | less than or equal    |
| `$in`    | in array              |
| `$nin`   | not in array          |

---

## 4. LOGICAL OPERATORS

```js
// AND — shipped orders over $500
db.orders.find({
  $and: [
    { status: "shipped" },
    { amount: { $gt: 500 } }
  ]
})
```
| _id | user_id | amount | status  |
|-----|---------|--------|---------|
| 1   | 1       | 999.99 | shipped |
| 8   | 4       | 599.98 | shipped |

```js
// OR — pending OR cancelled
db.orders.find({
  $or: [
    { status: "pending" },
    { status: "cancelled" }
  ]
})
```
| _id | status    | amount |
|-----|-----------|--------|
| 3   | pending   | 199.99 |
| 5   | cancelled | 999.99 |
| 7   | pending   | 499.99 |

---

## 5. UPDATE

```js
// update one — set status
db.orders.updateOne(
  { _id: 3 },
  { $set: { status: "shipped" } }
)
```
```
{ matchedCount: 1, modifiedCount: 1 }
```

```js
// update many — increase all amounts by 10%
db.orders.updateMany(
  { status: "shipped" },
  { $mul: { amount: 1.10 } }
)
```
```
{ matchedCount: 5, modifiedCount: 5 }
```

| Operator  | What it does                  |
|-----------|-------------------------------|
| `$set`    | set a field value             |
| `$unset`  | remove a field                |
| `$inc`    | increment by number           |
| `$mul`    | multiply by number            |
| `$push`   | add item to array             |
| `$pull`   | remove item from array        |
| `$rename` | rename a field                |

---

## 6. DELETE

```js
// delete one
db.orders.deleteOne({ _id: 5 })
// { deletedCount: 1 }

// delete many
db.orders.deleteMany({ status: "cancelled" })
// { deletedCount: 1 }
```

---

## 7. SORT, LIMIT, SKIP

```js
// top 3 most expensive orders
db.orders.find()
  .sort({ amount: -1 })
  .limit(3)
```
| _id | amount  | status  |
|-----|---------|---------|
| 1   | 999.99  | shipped |
| 5   | 999.99  | cancelled |
| 8   | 599.98  | shipped |

```js
// pagination — page 2, 3 items per page
db.orders.find()
  .sort({ _id: 1 })
  .skip(3)
  .limit(3)
```
| _id | amount | status  |
|-----|--------|---------|
| 4   | 299.99 | shipped |
| 5   | 999.99 | cancelled |
| 6   | 239.97 | shipped |

---

## 8. AGGREGATION PIPELINE

> MongoDB's equivalent of SQL GROUP BY + JOINs + Window Functions.
> Each stage feeds into the next like a pipe.

```
collection → [$match] → [$group] → [$sort] → result
```

---

## 9. $match — Filter (like WHERE)

```js
db.orders.aggregate([
  { $match: { status: "shipped" } }
])
```
| _id | user_id | amount  | status  |
|-----|---------|---------|---------|
| 1   | 1       | 999.99  | shipped |
| 2   | 1       | 499.99  | shipped |
| 4   | 2       | 299.99  | shipped |
| 6   | 3       | 239.97  | shipped |
| 8   | 4       | 599.98  | shipped |

---

## 10. $group — Aggregate (like GROUP BY)

```js
// total spent per user
db.orders.aggregate([
  {
    $group: {
      _id: "$user_id",
      total_spent:   { $sum: "$amount" },
      total_orders:  { $count: {} },
      avg_order:     { $avg: "$amount" }
    }
  },
  { $sort: { total_spent: -1 } }
])
```
| _id | total_spent | total_orders | avg_order |
|-----|-------------|--------------|-----------|
| 1   | 1499.98     | 2            | 749.99    |
| 4   | 1099.97     | 2            | 549.99    |
| 3   | 1239.96     | 2            | 619.98    |
| 2   | 499.98      | 2            | 249.99    |

| Accumulator | Meaning             |
|-------------|---------------------|
| `$sum`      | total               |
| `$avg`      | average             |
| `$min`      | minimum value       |
| `$max`      | maximum value       |
| `$count`    | count of documents  |
| `$push`     | collect into array  |
| `$first`    | first value in group|
| `$last`     | last value in group |

---

## 11. $project — Shape Output (like SELECT)

```js
db.orders.aggregate([
  {
    $project: {
      _id: 0,
      user_id: 1,
      amount: 1,
      status: 1,
      is_expensive: { $gt: ["$amount", 500] }  // computed field
    }
  }
])
```
| user_id | amount  | status    | is_expensive |
|---------|---------|-----------|--------------|
| 1       | 999.99  | shipped   | true         |
| 1       | 499.99  | shipped   | false        |
| 2       | 199.99  | pending   | false        |
| 2       | 299.99  | shipped   | false        |
| 3       | 999.99  | cancelled | true         |
| 4       | 599.98  | shipped   | true         |

---

## 12. $lookup — JOIN Two Collections

> MongoDB's equivalent of SQL JOIN.

```js
// join orders with users
db.orders.aggregate([
  {
    $lookup: {
      from:         "users",       // collection to join
      localField:   "user_id",     // field in orders
      foreignField: "_id",         // field in users
      as:           "user_info"    // output array name
    }
  },
  { $unwind: "$user_info" },       // flatten the array
  {
    $project: {
      _id: 1,
      amount: 1,
      status: 1,
      "user_info.name": 1,
      "user_info.city": 1
    }
  }
])
```
| _id | amount  | status    | user_info.name | user_info.city |
|-----|---------|-----------|----------------|----------------|
| 1   | 999.99  | shipped   | Alice          | NYC            |
| 2   | 499.99  | shipped   | Alice          | NYC            |
| 3   | 199.99  | pending   | Bob            | Austin         |
| 4   | 299.99  | shipped   | Bob            | Austin         |
| 5   | 999.99  | cancelled | Charlie        | Chicago        |

---

## 13. LEFT JOIN — Keep Users With No Orders

```js
// users who have NO orders (like SQL LEFT JOIN + IS NULL)
db.users.aggregate([
  {
    $lookup: {
      from:         "orders",
      localField:   "_id",
      foreignField: "user_id",
      as:           "orders"
    }
  },
  {
    $match: { orders: { $size: 0 } }   // empty array = no orders
  },
  {
    $project: { name: 1, city: 1, _id: 0 }
  }
])
```
| name | city   |
|------|--------|
| Eve  | Austin |

---

## 14. Multi-Stage Pipeline — Full Example

```js
// total shipped revenue per city, only cities > $1000
db.orders.aggregate([
  { $match: { status: "shipped" } },               // stage 1: filter shipped
  {
    $lookup: {                                       // stage 2: join users
      from: "users",
      localField: "user_id",
      foreignField: "_id",
      as: "user"
    }
  },
  { $unwind: "$user" },                            // stage 3: flatten
  {
    $group: {                                       // stage 4: group by city
      _id: "$user.city",
      revenue: { $sum: "$amount" },
      orders:  { $count: {} }
    }
  },
  { $match: { revenue: { $gt: 1000 } } },          // stage 5: filter groups
  { $sort: { revenue: -1 } }                       // stage 6: sort
])
```
| _id     | revenue | orders |
|---------|---------|--------|
| NYC     | 1599.97 | 3      |
| Chicago | 239.97  | 1      |

---

## 15. $addFields — Add Computed Fields

```js
db.orders.aggregate([
  {
    $addFields: {
      tax:         { $multiply: ["$amount", 0.08] },
      total_with_tax: { $multiply: ["$amount", 1.08] }
    }
  },
  { $project: { amount: 1, tax: 1, total_with_tax: 1, _id: 0 } }
])
```
| amount  | tax   | total_with_tax |
|---------|-------|----------------|
| 999.99  | 80.00 | 1079.99        |
| 499.99  | 40.00 | 539.99         |
| 199.99  | 16.00 | 215.99         |

---

## 16. $setWindowFields — Window Functions (MongoDB 5.0+)

> MongoDB's equivalent of SQL RANK(), SUM() OVER(), LAG()

```js
// rank users by total spending
db.orders.aggregate([
  {
    $group: {
      _id: "$user_id",
      total: { $sum: "$amount" }
    }
  },
  {
    $setWindowFields: {
      sortBy: { total: -1 },
      output: {
        rank: { $rank: {} }
      }
    }
  }
])
```
| _id | total   | rank |
|-----|---------|------|
| 1   | 1499.98 | 1    |
| 3   | 1239.96 | 2    |
| 4   | 1099.97 | 3    |
| 2   | 499.98  | 4    |

```js
// running total of orders by date
db.orders.aggregate([
  {
    $setWindowFields: {
      sortBy: { _id: 1 },
      output: {
        running_total: {
          $sum: "$amount",
          window: { documents: ["unbounded", "current"] }
        }
      }
    }
  },
  { $project: { amount: 1, running_total: 1, _id: 1 } }
])
```
| _id | amount  | running_total |
|-----|---------|---------------|
| 1   | 999.99  | 999.99        |
| 2   | 499.99  | 1499.98       |
| 3   | 199.99  | 1699.97       |
| 4   | 299.99  | 1999.96       |
| 5   | 999.99  | 2999.95       |

---

## 17. INDEXES

```js
// create index
db.orders.createIndex({ user_id: 1 })

// compound index
db.orders.createIndex({ user_id: 1, status: 1 })

// see all indexes
db.orders.getIndexes()

// drop index
db.orders.dropIndex("user_id_1")
```

```js
// check if index is used — like EXPLAIN ANALYZE
db.orders.find({ user_id: 1 }).explain("executionStats")

// look for:
// "winningPlan.stage": "IXSCAN"   ← index used ✅
// "winningPlan.stage": "COLLSCAN" ← full scan ❌
```

---

## 18. TEXT SEARCH

```js
// create text index
db.users.createIndex({ name: "text", city: "text" })

// search
db.users.find({ $text: { $search: "Alice" } })
```
| _id | name  | city | age |
|-----|-------|------|-----|
| 1   | Alice | NYC  | 30  |

---

## Quick Reference — All Concepts

| Concept              | MongoDB                                              | SQL Equivalent              |
|----------------------|------------------------------------------------------|-----------------------------|
| Filter rows          | `db.col.find({ field: val })`                        | `WHERE`                     |
| Greater than         | `{ amount: { $gt: 500 } }`                           | `amount > 500`              |
| AND condition        | `{ $and: [{...}, {...}] }`                           | `AND`                       |
| OR condition         | `{ $or: [{...}, {...}] }`                            | `OR`                        |
| Select fields        | `find({}, { name: 1, _id: 0 })`                     | `SELECT name`               |
| Sort                 | `.sort({ amount: -1 })`                              | `ORDER BY amount DESC`      |
| Limit                | `.limit(5)`                                          | `LIMIT 5`                   |
| Skip (pagination)    | `.skip(10).limit(5)`                                 | `OFFSET 10 LIMIT 5`         |
| Filter stage         | `{ $match: { status: "shipped" } }`                 | `WHERE`                     |
| Group & aggregate    | `{ $group: { _id: "$city", total: { $sum: "$amount" } } }` | `GROUP BY + SUM()`  |
| Filter after group   | `{ $match: { total: { $gt: 1000 } } }` after $group | `HAVING`                    |
| Shape output         | `{ $project: { name: 1 } }`                         | `SELECT col`                |
| Add computed field   | `{ $addFields: { tax: { $multiply: [...] } } }`     | computed column             |
| JOIN collections     | `{ $lookup: { from, localField, foreignField, as } }`| `JOIN`                     |
| Flatten join array   | `{ $unwind: "$fieldname" }`                          | flattens 1-to-many join     |
| No match (LEFT JOIN) | `$lookup` + `{ $match: { orders: { $size: 0 } } }`  | `LEFT JOIN ... IS NULL`     |
| Rank                 | `$setWindowFields` + `$rank`                         | `RANK() OVER()`             |
| Running total        | `$setWindowFields` + `$sum` + `window: unbounded`    | `SUM() OVER(ORDER BY)`      |
| Index                | `createIndex({ field: 1 })`                          | `CREATE INDEX`              |
| Query plan           | `.explain("executionStats")`                         | `EXPLAIN ANALYZE`           |
| IXSCAN in plan       | index was used ✅                                    | Index Scan                  |
| COLLSCAN in plan     | full collection scan ❌                              | Seq Scan                    |
