## Definition

A **concurrency control mechanism** where **multiple versions of data** are maintained so that:

- Reads don’t block writes
    
- Writes don’t block reads
	
- Writers **DO block other writers** (to preserve atomicity & rollback)

Instead of **preventing conflicts (like 2PL)**, MVCC **detects conflicts**.

---

## Core Idea

- Database keeps **multiple versions of each row**
    
- Each transaction sees a **snapshot (point-in-time view)**
    

---

## How It Works

- On UPDATE/DELETE:
    
    - Old version is **retained**
        
    - New version is **created**
        
- Each transaction reads data based on:
    
    - **Transaction start time** or
        
    - **Statement start time** (depends on isolation level)
        

---

## Example

- T1 starts → sees version V1
    
- T2 updates → creates V2
    

Result:

- T1 reads **V1**
    
- New transactions read **V2**
    

> No blocking between readers and writers

---
## Why MVCC?

**Locking (2PL)**:

- Causes **contention**
    
- Slows down **throughput & scalability**

**MVCC**:

- Avoids blocking
    
- Improves **concurrency**
    
- Still needs **conflict detection**

---

## Trade-offs

#### Advantages

- High concurrency
    
- Better performance in **OLTP systems**
    
- Reduced lock contention
    

#### Disadvantages

- Extra storage (multiple versions)
    
- Cleanup required
    
- Harder to guarantee **serializability**
    

---

## Compared to Locking (2PL)

|Feature|2PL (Locking)|MVCC|
|---|---|---|
|Conflict handling|Prevents conflicts|Detects conflicts|
|Reads vs Writes|Blocking|Non-blocking|
|Writers conflict|Yes|Yes|
|Performance|Lower|Higher|
|Serializability|Easier|Harder|

---

## Snapshot Concept

- Each transaction sees a **consistent snapshot**
    
- Based on a **logical timestamp**
    
- Often referred to as **Snapshot Isolation**
    

---

## Database Implementations

### Oracle

- Uses **MVCC only** (no 2PL)
    
- Snapshot based on **SCN (System Change Number)** → logical timestamp
    
- Old versions stored in **Undo Segments**
    
- Supports explicit locking:
    

```sql
SELECT ... FOR UPDATE
```

---

### SQL Server

- Default → **locking (2PL)**
    
- MVCC (row versioning) must be enabled:
    

```sql
ALTER DATABASE db SET READ_COMMITTED_SNAPSHOT ON;
ALTER DATABASE db SET ALLOW_SNAPSHOT_ISOLATION ON;
```

- Snapshot isolation must be set per transaction:
    

```sql
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
```

#### Internals

- Old versions stored in **tempdb (version store)**
    
- Uses **linked versions (chains)**
    

#### Important

- Long transactions:
    
    - Keep old versions alive
        
    - Increase memory & lookup cost
        
- Cleanup via **Ghost Cleanup Task**
    

---

### PostgreSQL

- Pure MVCC (no 2PL)
    
- Stores versions **inside the table itself**
    

#### Row metadata:

- `xmin` → creating transaction (**when this version was born**)
    
- `xmax` → deleting/updating transaction (**when this version died**)
    

#### Cleanup:

- **VACUUM** removes old versions
    

#### Locking still exists for:

```sql
SELECT ... FOR UPDATE
SELECT ... FOR SHARE
```

---

### MySQL (InnoDB)

- MVCC similar to Oracle
    
- Old versions stored in **Rollback Segments**
    

#### Behavior:

- DELETE → marks row (not immediate removal)
    
- Cleanup via **Purge Thread**
    

#### Notes:

- Reconstructs old versions when needed
    
- Explicit locking supported:
    

```sql
SELECT ... FOR UPDATE
SELECT ... LOCK IN SHARE MODE
```

---

## Important Considerations

- MVCC does NOT eliminate all contention:
    
    - Write-write conflicts still exist
        
- Long-running transactions:
    
    - Delay cleanup
        
    - Increase storage usage
        
    - Slow down version reconstruction
        

---

## Final Summary

- MVCC = **non-blocking reads + versioned data**
    
- Improves performance under concurrency
    
- Requires:
    
    - Careful isolation level choice
        
    - Monitoring of long transactions
        
    - Proper cleanup (VACUUM / purge / GC)


> [!NOTE] Title
> MVCC controls **how data versions are stored and accessed**,  
> while isolation levels control **which version a transaction can see**