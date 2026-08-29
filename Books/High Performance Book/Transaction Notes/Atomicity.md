
**Definition**

- Atomicity = _all-or-nothing_ execution
- If any operation **fails** → entire transaction is **rolled back**

---

### Write-Write Conflicts

- Only **one transaction can modify a record at a time**
- DB prevents conflicts automatically (**unlike version control like Git**)
- Changes are applied to real data structures but:
    - Become permanent **only at commit**
    - Can be reverted before commit

---

### Rollback Mechanism

- DB must restore **before image** of modified data
- Different databases implement this differently

---
### Oracle

- Uses **undo tablespace**
- Stores old versions in **undo segments**
- Rollback = reconstruct old data from undo

---

### SQL Server

- Uses **transaction log**
- Rollback = scan log _backward_ to undo changes

> [!NOTE]
>      Log must be truncated regularly
>      Long transactions → delay truncation → log growth

---

### PostgreSQL

- Uses **MVCC (multi-version storage)** (no undo log)
- Each row keeps its own versions
- Rollback = switch to previous version

**Important concepts:**

- **VACUUM** = cleans old versions
- Uses **XID (transaction ID)** (32-bit)

**Risks:**

- XID wraparound (~4B transactions)
- Without vacuum → serious data corruption risk

---

### MySQL

- Uses **undo log in rollback segments**

Undo log has two parts:

1. **Rollback** (short-lived)
2. **Versioning** (kept for other transactions)

- Background **purge process** cleans old data

Long transactions:

- Delay purge
- Cause large undo logs

---

### Key Takeaways

- Atomicity depends on the ability to **undo changes**
- All databases store **previous versions**, but differently:
    - Oracle / MySQL → undo logs
    - SQL Server → transaction log
    - PostgreSQL → MVCC (row versions)
- **Long-running transactions are dangerous**:
    - Block cleanup
    - Increase storage
    - Risk system issues