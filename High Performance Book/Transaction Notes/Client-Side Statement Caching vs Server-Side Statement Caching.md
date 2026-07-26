The sources distinguish between **server-side** and **client-side statement caching** based on where the cache is located, what exactly is being stored, and whether the JDBC driver or the database engine manages it.

**Server-side Statement Caching**

- **What is being cached:** The database caches **execution plans**. Because finding an optimal data-fetching algorithm is resource-intensive for the Parser and Optimizer, the database saves the resulting plan to eliminate this overhead for future identical statements.
- **Where it happens:** This occurs within the **database server itself** (e.g., Oracle’s "Shared Pool" or SQL Server’s "procedure cache").
- **How it works:** The database uses the SQL statement string as an input to a hashing function to create a cache key. If the string matches exactly, the database reuses the cached execution plan.
- **JDBC vs. Database:** This is a **database feature**. While you often use the JDBC `PreparedStatement` to ensure the statement strings remain identical for hashing, the actual cache is managed by the database engine. Some databases also offer "Forced Parameterization" to help the cache recognize identical statements even if they use different literals.

**Client-side Statement Caching**

- **What is being cached:** The cache stores **constructed statement objects** along with their associated **database-specific metadata**. This avoids the need to repeatedly rebuild these objects in the application's memory.
- **Where it happens:** This occurs on the **client-side**, within the Java Virtual Machine (JVM) where the application is running.
- **How it works:** The cache is typically confined to a specific database connection. When an application "closes" a cached statement, the driver doesn't destroy it; instead, it returns it to the cache to be recycled for the next request.
- **JDBC vs. Database:** This is a feature of the **JDBC driver**. In most cases, it is **not enabled by default** and must be activated by setting specific connection properties in the JDBC configuration (e.g., `cachePrepStmts` for MySQL or `implicitStatementCacheSize` for Oracle).

**Summary Table**

|Feature|Server-side Caching|Client-side Caching|
|---|---|---|
|**Location**|Database Server|JDBC Driver (Client JVM)|
|**Content**|Execution Plans|Statement Objects & Metadata|
|**Goal**|Sparing Database CPU/IO for planning|Sparing Application CPU/Memory for object creation|
|**Control**|Database Engine (Automatic or DB Settings)|JDBC Driver (Enabled via Connection Properties)|

Most database systems benefit significantly from **server-side caching**, particularly in high-performance OLTP systems where transactions are short. **Client-side caching** provides an additional layer of optimization by reducing JVM garbage and lowering the total transaction response time