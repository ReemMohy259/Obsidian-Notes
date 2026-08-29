## Definition

**Durability** guarantees that once a transaction is **committed**, its changes are:

-  **Permanent**
    
-  **Survive crashes / restarts**

---

## Intuition

Example:

- You buy a flight ticket
    
- Money is deducted + seat reserved
    
- System crashes immediately
    

After restart:

- The transaction **must still exist**
    

Otherwise:

- Money lost
    
- No ticket → **data inconsistency**

---

## Purpose

- Ensure **system recoverability**
    
- Guarantee that committed data is **never lost**
    

---

## Relation to Rollback

- **Undo log** → used for rollback (uncommitted changes)
    
- **Durability** → needs **committed changes only**
    

> So undo logs alone are **NOT enough**

---

## Key Mechanism: Redo Log

When a transaction commits:

- Changes are written to a **Redo Log**
    
- This log is:
    
    - **Append-only**
        
    - **Sequential (fast for disk I/O)**

> Ensures changes can be **reapplied after crash**

---

## Important Distinction

|Mechanism|Purpose|
|---|---|
|Undo Log|Rollback / MVCC|
|Redo Log|Recovery / Durability|

---

## How It Works

1. Transaction modifies data (in memory)
    
2. Changes recorded in logs
    
3. On **commit**:
    
    - Redo log is **flushed to disk**
        
4. If crash happens:
    
    - Database **replays redo log**

# Database Implementations

## Oracle

### Structure:

- Uses **Redo Log**
    
- Contains:
    
    - **Redo Records**
        
    - Each has **change vectors** (actual data block changes)

### Flow:

- Changes first go to **log buffer (memory)**
    
- **Log Writer (LGWR)** flushes to disk

### On Commit:

- Buffer is flushed → changes become durable

### Important:

- Buffer may flush even with **uncommitted changes** (only when its full)
    
    - These are later removed if rollback happens
        

### Files:

- At least **2 redo log files**
	- write to one while the other is being **processed/archived**
    
- Only **one active at a time**

---

## SQL Server

### Structure:

- Uses a single **Transaction Log**
    
    - Combines:
        
        - **Undo log**
            
        - **Redo log**

### Default Behavior:

- On commit → log is **flushed to disk synchronously**
    

### Configurable Durability:

- Can delay flushing (**asynchronous**)
    

#### Pros:

- Better performance
    
- Lower latency
    

#### Cons:

- Crash → **data loss possible**

 Use only if:

- **Data loss is acceptable**

---

## PostgreSQL

### Structure:

- Uses **WAL (Write-Ahead Log)**
	- WAL = a **sequential log of all changes** written to disk before data pages

### Key Idea:

- Log is written **before data pages**
    

### Flow:

- Changes written to WAL (in memory)
    
- Flushed on commit
    

### Optimization:

- Data pages **don’t need immediate flush**
    
- Can be rebuilt from WAL
    

> Improves **I/O performance**

### Configurable:

- Supports **asynchronous flushing** 

---

## MySQL (InnoDB)

### Structure:

- Uses **Redo Log**

### Flow:

1. Changes go to:
    
    - **Mini-transaction buffer**
        
2. Then to:
    
    - **Global redo buffer**
        
3. On commit:
    
    - Flushed to disk

### Files:

- Typically **2 redo log files** (alternating)

### Config:

- `innodb_flush_log_at_trx_commit`

#### Modes:

- **Synchronous (default)** → safe
    
- **Asynchronous** → faster but unsafe

---

## Synchronous vs Asynchronous Durability

### Synchronous (Safe)

- Flush on every commit
    
- No data loss
    
- Slower

### Asynchronous (Fast)

- Delayed flush
    
- Better performance
    
- Risk of losing recent transactions

---

## Key Takeaways

- Durability relies on **logging, not data pages**
    
- Logs are:
    
    - Sequential → fast
        
    - Reliable → used for recovery

---

## Practical Insight

> If the system crashes:

- Data pages may be inconsistent
    
- But logs allow **recovery to last committed state**

---

## Final Rule

> Only a transaction whose **log is safely written to disk** is truly committed

---

## Best Practice

- Use **synchronous durability** for:
    
    - Banking
        
    - Payments
        
    - Critical systems
        
- Use **asynchronous durability** only when:
    
    - Some data loss is acceptable
        
    - Logging is a performance bottleneck


> [!NOTE]
>1. Data pages in memory are the **current working data used for reads and writes**
>2. WAL logs store **changes for durability**, not the actual data
>3. Later, data pages are flushed to disk to **update the real stored database**
