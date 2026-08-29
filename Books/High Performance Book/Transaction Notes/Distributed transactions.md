### 🔹 Local vs Global Transactions

- **Local Transaction**
    - Uses **single resource manager** (e.g., one DB).
    - Managed via:
        - JDBC
        - JPA “Java Transaction API” (`RESOURCE_LOCAL`)
- **Global (Distributed) Transaction**
    - Spans **multiple heterogeneous resource managers**.
    - Requires a **Global Transaction Manager (JTA)**.
    - Ensures **ACID per resource**, coordinated globally.

---

### 🔹 Role of Transaction Manager

- Registers all **transactional resources**.
- Controls:
    - `commit`
    - `rollback`
- Coordinates outcome across all participants.
- Resources accessed via:
    - **JNDI (Java Naming and Directory Interface) → “go manually get the resource”**
    - **CDI (Contexts and Dependency Injection) → “resource is given to you automatically”**

---

### 🔹 Spring Transaction Abstraction

- Provides unified API over:
    - **Local transactions** (JDBC / JPA `RESOURCE_LOCAL`)
    - **Global transactions** (JTA)
- Uses **Dependency Injection** to wire resources into beans.

---

## **Two-Phase Commit (2PC)**

> JTA makes use of 2PC, which guarantees **atomic commit across multiple resources**.

---

### 🔹 Phase 1: Prepare

- Each resource:
    - Executes required operations.
    - Prepares to commit.
    - Responds with:
        - ✅ _Ready_ → continue
        - ❌ _Fail_ → rollback all

---

### 🔹 Phase 2: Commit

- If **all resources are ready**:
    - Transaction Manager issues **commit**
- If **any resource fails**:
    - Transaction Manager issues **rollback**
![[Pasted image 20260426234535.png]]

---

### 🔹 Failure Handling (Even after Ack)

- If a resource:
    - Fails during commit OR
        
    - Times out
        
        → Transaction Manager:
        
    - Retries in background
        
    - May require **manual intervention**
        

---

## One-Phase Commit (1PC) Optimization

### When it applies

> Only **one resource manager** involved (too many round trips redundant)

### 🔹 Behavior

- Skips **prepare phase**
- Directly executes:
    - `commit` OR `rollback`

### 🔹 Benefit

- Reduces:
    - Network overhead
    - Latency

```java
XAResource.commit(Xid xid, boolean onePhase)
```

`onePhase = true` → enables 1PC optimization

---