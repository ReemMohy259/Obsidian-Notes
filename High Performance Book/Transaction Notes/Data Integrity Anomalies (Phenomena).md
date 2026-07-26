# Transaction Phenomena (Data Anomalies)

## Overview

To improve **performance & scalability**, databases relax **serializability** using different isolation levels.

Trade-off:

- **Higher isolation** → fewer anomalies, lower concurrency
    
- **Lower isolation** → more anomalies, higher scalability
    

---

## Key Idea

- Lower isolation = less locking / fewer MVCC aborts
    
- BUT → more **data integrity risks**

> Responsibility shifts to **application logic** to handle anomalies

---

## Types of Phenomena

### 1. Dirty Write (NEVER ALLOWED)

 📍*Two transactions update the same row simultaneously*

![[Pasted image 20260421220543.png]]

#### Problem:

- One update overwrites another (**lost update**)
    
- Rollback becomes impossible to handle correctly
    

#### Important:

- Breaks **atomicity**
    
- Prevented by **ALL databases**, even at lowest isolation
    

---

### 2. Dirty Read

📍 _Reading uncommitted data from another transaction_

![[Pasted image 20260421220819.png]]

#### Problem:

- Data may be rolled back → invalid decisions
    

#### Prevention:

- 2PL → write locks block reads
    
- MVCC → use **previous version (undo log)**
    

#### Notes:

- Only allowed in **Read Uncommitted**
    
- Rarely used in practice
    

---

### 3. Non-Repeatable Read (Fuzzy Read)

📍 _Same row read twice → different values_

![[Pasted image 20260421220950.png]]

#### Cause:

- Another transaction updates the row between reads
    

#### Problem:

- Decisions based on outdated data
    

#### Prevention:

- **Repeatable Read / Serializable**
    
- Or explicit locks:
    

```sql
SELECT ... FOR SHARE
```

#### MVCC behavior:

- Detects version change → may **abort transaction**

#### ORM Trick:

- **Hibernate** caches entity → simulates repeatable read
    

---

### 4. Phantom Read

📍 _Same query returns different set of rows_

![[Pasted image 20260421221114.png]]

#### Cause:

- Another transaction **inserts/deletes** rows **matching condition**
    

#### Example:

- Query: `WHERE price < 100`
    
- New row inserted → appears later
    

#### Prevention:

- 2PL → **predicate locks**
    
- MVCC → **snapshot**, but:
    
    - Inserts may still bypass detection in some cases
        

#### Important:

- Harder to prevent than **non-repeatable reads**
    

---

### 5. Read Skew

📍 _Inconsistent reads across related data_

![[Pasted image 20260421221302.png]]

#### Cause:

- Transaction reads **multiple rows/tables**
    
- Another transaction updates them **in between**
    

#### Example:

- Reads:
    
    - `post` (old)
        
    - `post_details` (new)
        

#### Result:

- Inconsistent state → wrong assumptions
    

#### Prevention:

- **Shared locks**
    
- MVCC → **abort on commit validation**
    

---

### 6. Write Skew

📍 _Concurrent writes violate a constraint_

![[Pasted image 20260421221425.png]]

#### Cause:

- Two transactions **read same data**
    
- Each updates **different parts** independently

#### Result:

- Constraint is **broken**

#### Prevention:

- **Shared locks**
    
- MVCC → **conflict detection + rollback**

---

### 7. Lost Update (Very Common)

📍 _One update overwrites another silently_

![[Pasted image 20260421221559.png]]

#### Flow:

1. **T1** reads value
2. **T2** updates value
3. **T1** updates → overwrites **T2**

#### Problem:

- **Data inconsistency** (e.g., wrong price)
#### Prevention:

- 2PL → **locks prevent update**
    
- MVCC → **detects version mismatch then abort**

#### ORM Solution:

- **Optimistic Locking (Hibernate)**
    
    - Uses **version** column
        
    - If update count = 0 → rollback