Setting the appropriate fetching size can ==undoubtedly speed up== the result set **retrieval**, as long as a **statement** fetches ==only the data required== by the **current** business logic.

# Too Many Rows

It is **inefficient** to fetch a whole result set if it cannot fit into the **user interface**.

**Pagination** or **dynamic scrolling** are common ways of addressing this issue, and ==partitioning data sets becomes unavoidable==.

==Without placing upperbounds==, the **resultsets** grow **proportionally** with the underlying table data.

**Limiting queries** can, therefore, ensure predictable ==response times and database resource== utilization.

### There are basically two ways of limiting a result set

#### 1) The most efficient strategy is to include the **row restriction** clause in the SQL statement 
`LIMIT`
`TOP`
#### 2) Configure a **maximum row count** at the JDBC **Statement** level 
`statement.setMaxRows(maxRows)`

According to the **JDBC documentation**, the driver is expected to ==discard the extra rows== when the ==maximum threshold== is reached.
###### Oracle

- Uses **fetch size batching** → data is retrieved in chunks while iterating.
- `maxRows` stops fetching early → saves network + client memory.
- Small `maxRows` can hurt performance → optimizer may **avoid indexes** and choose full scans.

---

###### SQL Server

- `setMaxRows()` translates to `SET ROWCOUNT N`.
- Affects **execution phase only**, not query planning.
- Can lead to **suboptimal plans** (e.g., table scan instead of index) prefer `TOP` / `FETCH`.

---

###### PostgreSQL

- `maxRows` is sent to the server → optimizer can **adjust execution plan**.
- Can skip expensive work (e.g., full sorting) based on limit.
- Cursor closes early after reaching limit → saves DB + network resources.

---

###### MySQL

- `maxRows` is **not sent to the server** → optimizer can’t use it.
- Driver still fetches normally, then limits on client side.
- Only benefit → **reduced client/network usage**, no server-side optimization.

---
###### Performance Difference

This test shows how limiting the result size **improves** **performance** on a ==dataset of 100k posts and 1M comments==.

Fetching all records scales with data size, while restricting results to ==100 rows== (via SQL or JDBC `maxRows`) ==greatly reduces response time==.

![[Pasted image 20260420022032.png]]

# Too Many Columns

- Selecting unnecessary columns (`SELECT *`) increases **processing time and resource usage**, even if row count is small.
- Fetching fewer columns (==only what you need==) can significantly **improve performance**.
- Common issue with **ORM tools** → they often load **entire entities**, leading to over-fetching.
- Extra columns **waste resources** across the board → **CPU**, **memory**, **I/O**, and **network bandwidth**.

### Performance Gain
Selecting **100** **posts** with details and their associated **1000** **comments**, using one of following two **statements**:

```
SELECT * 
FROM post_comment pc
INNER JOIN post p ON p.id = pc.post_id
INNER JOIN post_details pd ON p.id = pd.id
```

```
SELECT pc.version 
FROM post_comment pc
INNER JOIN post p ON p.id = pc.post_id 
INNER JOIN post_details pd ON p.id = pd.id
```

![[Pasted image 20260420023548.png]]