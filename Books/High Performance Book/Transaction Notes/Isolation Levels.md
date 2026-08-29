## Definition

**Isolation levels** define how **visible the effects** of one transaction are to other concurrent transactions.

They control:

- What data you can see
    
- When you can see it
    
- How **safe** concurrent execution is
    

---

## Why Isolation Levels Exist

- Multiple transactions run **concurrently**
    
- Without control → **[[Data Integrity Anomalies (Phenomena)]]**
	
- Isolation levels define the trade-off between **performance (concurrency & scalability) and correctness (data integrity)**.

---

## Key Insight

- **Serializable** = only level that guarantees **full ACID isolation**
    
- Lower levels:
    
    - Improve performance
        
    - Allow anomalies

> Database shifts responsibility → **application must handle inconsistencies**

---

## Standard Isolation Levels (SQL-92)

| Isolation Level  | Dirty Read | Non-Repeatable Read | Phantom Read |
| ---------------- | ---------- | ------------------- | ------------ |
| Read Uncommitted | ✔          | ✔                   | ✔            |
| Read Committed   | ❌          | ✔                   | ✔            |
| Repeatable Read  | ❌          | ❌                   | ✔            |
| Serializable     | ❌          | ❌                   | ❌            |

> [!NOTE]
> This is the **minimum guarantee** → real DBs may behave differently.

---

## JDBC Control

### Get default level:

```java
int level = connection.getMetaData().getDefaultTransactionIsolation();
```

### Set isolation level:

```java
connection.setTransactionIsolation(
    Connection.TRANSACTION_SERIALIZABLE
);
```

---

## Default Isolation Levels (Common DBs)

- MySQL → **Repeatable Read**
    
- Oracle → **Read Committed**
    
- SQL Server → **Read Committed**
    
- PostgreSQL → **Read Committed**

# The Isolation Levels

## 1. Serializable (Strongest)

> Transactions behave as if executed **sequentially**

### Prevents ALL anomalies:

- Dirty read
    
- Non-repeatable read
    
- Phantom read
    
- Lost update
    
- Read skew
    
- Write skew

### Trade-offs:

- Lowest concurrency
    
- High contention (locks or aborts in MVCC)
    

---

### Important (Real-world behavior)

- Not all DBs are truly serializable:
    
    - Oracle (MVCC) → allows **write skew**
        
    - SQL Server (2PL) → fully serializable
        
    - PostgreSQL → uses **SSI (Serializable Snapshot Isolation)**
        
        - May abort transactions (even false positives)

---

## 2. Repeatable Read

> Same row read multiple times → **same value**

### Prevents:

- Dirty read
    
- Non-repeatable read
    

### Depends on DB:

- **PostgreSQL (MVCC / Snapshot Isolation):**
    
    - Prevents phantom reads too
        
    - Aborts conflicting transactions
        
- **SQL Server (2PL):**
    
    - Uses **shared locks** → blocks updates
        
- **MySQL (InnoDB):**
    
    - Prevents non-repeatable reads
        
    -  Allows:
        
        - Lost update
            
        - Write skew
            
- **Oracle:**
    
    -  Not supported


---

## 3. Read Committed (Most Common)

> Each query sees **latest committed data at execution time**

### Prevents:

- Dirty read
    
- Dirty write
    

### Allows:

- Non-repeatable read
    
- Phantom read
    
- Lost update
    
- Read skew
    
- Write skew

---

### Behavior by DB

#### Oracle / PostgreSQL

- Each query gets **snapshot at query start**
    
- No shared locks for reads
    
- Writers can still modify data

#### SQL Server

- Default:
    
    - Uses **shared locks (released after query)**
        
- Optional:
    
    - MVCC mode (Read Committed Snapshot)

#### MySQL

- Uses **query-time snapshots**
    
- Locks applied on:
    
    - UPDATE / DELETE
        
    - Explicit locking queries

---

### Important Risk

- Lost updates are common:
    
    - Second transaction may overwrite first

---

## 4. Read Uncommitted (Weakest)

> Can read **uncommitted data**

### Prevents:

- Only dirty writes
    

### Allows:

- Dirty read
    
- Non-repeatable read
    
- Phantom read
    
- Lost update
    
- Read skew
    
- Write skew

---

### Behavior by DB

- **Oracle / PostgreSQL:**
    
    -  Not supported → fallback to Read Committed
        
- **SQL Server:**
    
    - No shared/exclusive locks → high concurrency
        
    - Useful for large reporting queries
        
- **MySQL:**
    
    - Allows dirty reads
        
    - Skips rebuilding previous versions (optimization)

---

## Key Takeaways

- Isolation level = **business decision**
    
- Default (Read Committed) is:
    
    - Fast
        
    - But **unsafe for critical logic**
        

---

## Practical Guidelines

- Use **Serializable**:
    
    - Critical financial operations
        
- Use **Repeatable Read**:
    
    - When consistency across reads matters
        
- Use **Read Committed**:
    
    - General-purpose apps (most common)
        
- Avoid **Read Uncommitted**:
    
    - Unless doing non-critical reporting
        

---

## Final Insight

> Higher isolation = safer but slower  
> Lower isolation = faster but riskier

 Always design your application assuming:  
**anomalies CAN happen unless explicitly prevented**