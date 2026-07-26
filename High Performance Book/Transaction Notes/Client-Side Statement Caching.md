## Client-Side Statement Caching (JDBC)

### What is it?

Client-side statement caching allows the **JDBC driver** to reuse already created `PreparedStatement` / `CallableStatement` objects instead of **recreating** them every time along with their associated **database-specific metadata** (*SQL string & parsed structure*).

### Goals

- Reduce statement creation overhead → **faster response time**
    
- Reuse statement objects + metadata → **save resources**
    
- Improve throughput in **short OLTP transactions**
    
[[Client-Side Statement Caching vs Server-Side Statement Caching]]

---

## Key Characteristics

- Cache is **per database connection** (not shared globally)
    
- Cache key = **SQL string**
    
- Works best with:
    
    - `PreparedStatement`
        
    - `CallableStatement`
        

---

## Oracle JDBC

### 1. Implicit Statement Caching

- Enabled via connection/DataSource config
    
- Caches **only metadata**
    
- Automatically caches statements
    

```java
connectionProperties.put(
"oracle.jdbc.implicitStatementCacheSize", "50");
```

#### Control caching

```java
statement.setPoolable(false); // prevent caching
```

#### Limitation

- May cache rarely used queries → evict important ones
    

---

### 2. Explicit Statement Caching

- Full control using Oracle API
    
- Stores:
    
    - Metadata
        
    - Execution state 
	    - **risky** because it saves the **parameters** from the **last execution** (might reuse old state)

#### Enable:

```java
oracleConnection.setExplicitCachingEnabled(true);
oracleConnection.setStatementCacheSize(50);
```

#### Usage:

```java
PreparedStatement stmt =
    oracleConnection.getStatementWithKey("KEY");

if (stmt == null) {
    stmt = connection.prepareStatement(SQL);
}

try {
    stmt.execute();
} finally {
    ((OraclePreparedStatement) stmt).closeWithKey("KEY");
}
```

#### Key Operations

- `getStatementWithKey()` → fetch from cache
    
- `closeWithKey()` → return to cache
    

#### Downsides

- Tied to Oracle API → **not portable**
    
- Risk of mixing execution states
    

---

## SQL Server

- Official driver → no statement caching
    
- jTDS driver:
    
    - Supports caching (default: 500)
        

```java
dataSource.setMaxStatements(500);
```

---

## PostgreSQL

- **Automatic caching** (since driver 9.4)
    
- Cannot disable per statement
    

### Config:

- `preparedStatementCacheQueries` (default: 256)
    
- `preparedStatementCacheSizeMiB` (default: 5MB)
    

```java
dataSource.setPreparedStatementCacheQueries(256);
dataSource.setPreparedStatementCacheSizeMiB(5);
```

---

## MySQL (Connector/J)

### Enable caching:

```java
dataSource.setCachePrepStmts(true);
```

### Config:

- `prepStmtCacheSize` (default: 25)
    
- `prepStmtCacheSqlLimit` (default: 256 chars)
    

```java
dataSource.setPreparedStatementCacheSize(50);
dataSource.setPreparedStatementCacheSqlLimit(256);
```

### Notes

- Cache is per connection
    
- `setPoolable(false)` affects **server-side only**
    

---

## Summary

- Client-side caching reduces JDBC overhead significantly
    
- Best for **high-frequency, short queries**
    
- Always tune cache size carefully
    
- Beware of:
    
    - Cache pollution (implicit caching)
        
    - Portability issues (explicit caching)
        
    - State reuse bugs (advanced caching)