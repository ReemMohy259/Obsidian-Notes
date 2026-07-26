The **transfer rate** is controlled by the **Statement fetch size**.

The **default value** of $0$ leaves each database **choose** its driver specific **fetching policy**.

### Oracle
#### In 10i/11g:

- Eager allocation (**At Statement Creation**)
- Worst-case sizing
- Duplicate storage (`byte[]` + `char[]`)
- High memory footprint (**Fetch Size * Max Column Size**)

#### In 12c:

- Lazy allocation (**Allocated When the Result Set is Ready for Fetching**)
- Actual data size (**Actual Extracted Data Size**)
- Byte-based storage (**Decoded When needed** $2$ * `byte[]` )
- Less duplication

Result: **significantly lower memory usage and better scalability**

The Oracle JDBC Driver specification recommends limiting the **fetch size** to at most **100** records.

> [!NOTE]
> Avoiding memory allocation, by reusing existing buffers, is a very solid reason for employing [[Client-Side Statement Caching]]

---
### SQL Server

The **SQL Server** JDBC driver uses **adaptive buffering**, so the result set is fetched in batches, **as needed**.

The size of a batch is therefore **automatically** controlled by the driver.

---
### Postgresql

The entire result set is fetched **at once** into client memory. The default fetch size **Large Enough To** require only one database roundtrip.

By changing **fetch size**, the result set is associated with a **database cursor**, allowing data to be fetched on demand.

---

### MySQL

Because of the **network protocol design consideration**, fetching the whole result set is the **most efficient** data retrieval strategy

 You basically have **two extremes**:
#### 1) Default mode

- Fast
- Few roundtrips
- High memory usage

#### 2) Streaming mode

-  Low memory
-  Slower
-  More network overhead ([[Full Fetch vs Streaming]])

# Performance Difference
The following graph captures the **response time** of four database systems when fetching ==10 000 rows== while varying the ==fetch size== of the ==forward-only and read-only== **ResultSet**.

![[Pasted image 20260420015942.png]]

Also Check:
-  [[ResultSet Size]]

