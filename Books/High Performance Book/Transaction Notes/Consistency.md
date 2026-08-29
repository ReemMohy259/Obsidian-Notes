
**Definition**

- A transaction transforms DB from **one valid state → another valid state**
- Validity is defined by **schema constraints**

---

### Constraint Enforcement

Database ensures all changes respect:

- Data types
- Column length
- `NULL` / `NOT NULL`
- Foreign keys
- Unique constraints
- Check constraints
- Triggers (secondary changes must also be valid)

---

### Core Rule

- If **any constraint is violated** → entire transaction is **rolled back**
- **No partial** commits allowed

---

### Role of Application vs Database

- Application should validate input **before sending queries**
- BUT:
    - Cannot handle **concurrent requests across systems**
- Database acts as the **final authority (single source of truth)**

> Strong schemas are critical in distributed / multi-service systems

---

### MySQL Behavior

By default, MySQL is **NOT strictly consistent**

Instead of **failing**, it silently fixes **invalid data**:

- **Out-of-range numbers** → `0` or max value
- **Strings** → truncated to max length
- Invalid dates allowed (e.g., `2015-02-30`)
- `NOT NULL`:
    - Enforced for single inserts
    - Ignored in multi-row inserts:
        - Numbers → `0`
        - Strings → `''`

---

### Enabling Strict Mode in MySQL

Strict consistency requires configuration:

`SET GLOBAL sql_mode = 'POSTGRESQL,STRICT_ALL_TABLES';`

---

### Consistency vs Consistency in [[CAP Theorem]]

Two **different meanings** of **consistency**

### ACID Consistency

- Ensures **data validity**
- Enforces constraints

### CAP Consistency

- Means **linearizability**
- All nodes see the **same data at the same time**

---

### Key Takeaways

- Consistency = **constraint enforcement**
- DB is the **ultimate validator**, not the app
- MySQL can **silently corrupt data** if not in strict mode
- ACID consistency ≠ CAP consistency (common confusion)