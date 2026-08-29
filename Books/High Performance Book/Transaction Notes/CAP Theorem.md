# What is the CAP Theorem?

The **CAP theorem** applies to **distributed systems** (multiple servers/nodes working together).

It says:

> When there is a **network problem (partition)**, a system can guarantee only **2 out of 3**

- **C — Consistency (CAP meaning)**
- **A — Availability**
- **P — Partition tolerance**

---
## The 3 Properties

### 1. Consistency (CAP version)

- Every read gets the **latest write**
- All nodes show **the same data at the same time**

> _“No stale data ever”_

---

### 2. Availability

- Every request gets a response
- Even if it’s **not the latest data**

> _“System never says no”_

---

### 3. Partition Tolerance

- System keeps working **despite network failures**

> This is **not optional** in real systems

---
### CAP Theorem Summary

In distributed systems, under network partition:

You must choose between:

- **Consistency (CAP)** → same data across nodes
- **Availability** → system always responds

Cannot guarantee both simultaneously