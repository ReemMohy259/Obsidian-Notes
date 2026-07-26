## Why Isolation Exists

- Single user → **no conflicts**
- Multiple users (concurrent transactions) → **possible conflicts**

From the **Universal Scalability Law**:

- If some work is parallelizable → system **scales with concurrency**
- More concurrent transactions → **higher throughput**

Problem:

- Parallel execution → **interleaving operations**
- Interleaving → can break **data integrity**

---

## Goal of Isolation

- Ensure concurrent transactions behave **as if executed one by one** this property is called:
### Serializable Execution

- Final result = same as **sequential execution**

# Concurrency Control

To handle conflicts, databases use 2 strategies:

## 1. Avoid Conflicts (Locking)

- Example: **[[Two-Phase Locking (2PL)]]**
- Prevents conflicts before they happen

✔ Strong correctness  
❌ Lower concurrency (blocking)

---

## 2. Detect Conflicts (MVCC)

- Example: **[[Multi-Version Concurrency Control (MVCC)]]**
- Allows conflicts, then resolves them

✔ Better performance  
❌ May allow anomalies (weaker than full serializability)

# Transaction Schedule & Strictness

## Transaction Schedule

- The database engine determines the transaction schedule
- Order of all interleaved operations

## Strict Schedule

Rules:

- A transaction must **commit before others use its changes**
- Locks released **only after commit/rollback**

### Benefits:

- Prevents **cascading aborts**

---

## Cascading Abort

- One rollback → **forces** other **dependent** transactions to **rollback**

> Strict schedules prevent this


# Deadlocks

## What is a Deadlock?

- **Two transactions** waiting on each other **forever**

Example:

- T1 holds Lock A → wants Lock B
- T2 holds Lock B → wants Lock A

> Neither can proceed

---

## Why Deadlocks Happen

- Both transactions are in **expanding phase**
- Neither releases locks early (by design in 2PL)

---

## How Databases Handle Deadlocks

- DB detects **lock cycles**
- Chooses one transaction to **abort**
- Releases its locks
- Other transaction continues

---

## Important Insight

- Deadlocks **cannot be fully prevented** by the database alone
- Application should:
    - Access resources in consistent order
    - Keep transactions short