## Definition

A **read-only transaction** is a transaction that:

- **Only reads** data
    
- Does **NOT modify** data

In JDBC:

```java
connection.setReadOnly(true);
```

---

## Important Rules

- Must be set **before transaction starts**
    
- Cannot **convert** a running transaction to **read-only**
    
- It is often just a **hint**, not enforcement

---

## Purpose

- Enable **database optimizations**
    
- Reduce overhead (locks, metadata, etc.)
    
- Improve performance for **read-heavy workloads**

---

## Behavior by Database

### Oracle

- `setReadOnly(true)` is **NOT enforced**
    
- Modifications are still allowed
    

To enforce:

```sql
SET TRANSACTION READ ONLY;
```

#### Requirements:

- Must disable auto-commit first
    
- Applies to the **whole transaction**
    

---

### SQL Server

- Read-only flag NOT enforced

 To enforce:

- Use **restricted user (read-only account)**

#### Note:

- `ApplicationIntent=ReadOnly`
    
    - Used for **routing**, NOT enforcement

---

### PostgreSQL

- Enforced:
    
    - Modifying statements → **Exception**
        

### Optimization:

- Reduces **false-positive aborts** in Serializable

### Advanced Feature:

**Deferrable Snapshot**

```sql
SET TRANSACTION SERIALIZABLE READ ONLY DEFERRABLE;
```

#### Behavior:

- Waits for a **safe snapshot**
    
- **Avoids serialization failures**

 Best for:

- Long-running read queries

---

### MySQL (InnoDB)

- Enforced:
    
    - Modifications → **Exception**
        

### Optimization:

- Skips **transaction ID generation**
    
- Improves performance

---

## Key Takeaways

- Read-only = **optimization hint or enforcement (DB dependent)**
    
- Useful for:
    
    - Reporting
        
    - Analytics
        
    - Long-running reads

# Read-Only Transaction Routing

## Idea

In replication setups:

- **Master** → read + write
    
- **Slave/Replica** → read-only

 Route transactions based on:

- Read-only vs read-write

---

## Benefits

- **Load balancing**
    
- **High availability**
    
- **Better scalability**

---

## Database Support

### Oracle (Active Data Guard)

- Routes:
    
    - Write → Primary (**Master**)
        
    - Read → Standby (**Slave**)
- Supports:
    
    - Failover
        
    - Load balancing

---

### SQL Server

- Uses **Availability Groups**

 Routing based on:

- `ApplicationIntent=ReadOnly`
    

Requires:

- Separate DataSources

---

### PostgreSQL

- Driver supports:
    
    - `loadBalanceHosts`
        
    - `targetServerType` (**master / preferSlave**)

 Routing:

- Must be handled **in application**

---

### MySQL

- `ReplicationDriver` supports routing
    

Decision based on:

- `connection.setReadOnly(true)`

---

## Important Design Note

- Routing must happen **before getting connection**
    
- Cannot rely on connection state after creation

---
## Best Practice

- Use separate:
    
    - **Read** DataSource
        
    - **Write** DataSource

![[Pasted image 20260423213151.png|697]]