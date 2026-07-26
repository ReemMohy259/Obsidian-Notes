Introduced by:

- Jim Gray and others (1976)

## Core Idea

Transactions must follow **two phases**:

### 1. Expanding Phase

- Acquire locks
- Cannot release locks

### 2. Shrinking Phase

- Release locks
- Cannot acquire new locks

> Ensures **serializability**

---

## Lock Types

- **Shared Lock (Read)**
    - Allows **multiple reads**
    - Blocks writes
- **Exclusive Lock (Write)**
    - Blocks both **reads** and **writes**

---

## Lock Granularity

**Locks** can be applied at different levels:

- **Tablespace** → large logical storage area (multiple files), blocking access to a big portion of the database
- **File** → An entire database file
- **Page** → fixed-size block of rows, affecting multiple nearby records
- **Row** → single row

### Trade-off:

- More Granularity (row-level):
    - ✔ More concurrency
    - ❌ More overhead
- Less Granularity (table-level):
    - ✔ Less overhead
    - ❌ More contention

---

## Lock Escalation

- DB may replace many small locks with **one large lock**

**Trades:**

- Less resource usage
- More contention (**less concurrency**)

# Example Flow (2PL Behavior)

1. Two transactions read same row → both get **shared lock**
2. One tries to write → **blocked**
3. After first transaction commits → locks released
4. Second transaction:
    - Upgrades to **exclusive lock**
    - Performs update
5. Others trying to read/write → blocked until commit