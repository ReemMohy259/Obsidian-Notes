### Why Explicit Locking?

- Implicit locking is sufficient for many concurrency scenarios.
- Explicit locking provides **finer-grained control** over data consistency.
- JPA offers a concurrency control API for implementing complex integrity rules.
- Supports both:
    - **Optimistic Locking**
    - **Pessimistic Locking**
---
### Pessimistic Locking

- JPA abstracts database-specific locking implementations.
- Allows acquiring:
    - **Shared Locks** → multiple readers allowed.
    - **Exclusive Locks** → only one transaction can access/update.
---
### Optimistic Locking

- Implicit optimistic locking automatically manages the `@Version` field.    
- Developers cannot manually modify the version attribute.
- Explicit optimistic lock modes allow, ==Forcing a version increment even when no entity state changed.==
- Useful for maintaining consistency between related entities.

> **Example:**
> Child entity is updated. Then parent entity version is incremented to signal a logical change.

---
### Where Lock Modes Can Be Applied

#### Direct Entity Operations

```java
entityManager.find(...)
entityManager.lock(...)
entityManager.refresh(...)
```

#### JPQL / Criteria Queries

```java
query.setLockMode(...)
```

---
### LockModeType Overview

| Lock Mode                              | Purpose                                                                                | Detailed Reference                               |
| -------------------------------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------ |
| `NONE`                                 | Uses default behavior (no explicit lock).                                              | ----                                             |
| `PESSIMISTIC_WRITE`                    | Acquires **exclusive lock**, prevents others from obtaining shared or exclusive locks. | [[Pessimistic Read and Pessimistic Write Modes]] |
| `PESSIMISTIC_READ`                     | Acquires **shared lock**, prevents others from obtaining exclusive lock.               | [[Pessimistic Read and Pessimistic Write Modes]] |
| `OPTIMISTIC` (`READ`)                  | Performs version check at **transaction commit.**                                      | [[Optimistic Mode]]                              |
| `OPTIMISTIC_FORCE_INCREMENT` (`WRITE`) | Forces version increment before commit.                                                | [[Optimistic Force Increment Mode]]              |
| `PESSIMISTIC_FORCE_INCREMENT`          | Acquires **exclusive DB lock** and immediately increments version.                     | [[Pessimistic Force Increment Mode]]             |

> [!NOTE] Where Lock Modes Can Be Acquired
> - **Direct Entity Operations** 
> 	entityManager.find(...)
> 	entityManager.lock(...)
> 	entityManager.refresh(...)
> 	
> - J**PQL / Criteria Queries**
> 	query.setLockMode(...)

