#### Purpose
- Used when an entity's state depends on another entity's version.
- Detects concurrent modifications without acquiring a database lock.
- Prevents application-level inconsistencies while preserving scalability.
#### Problem Scenario
![[Pasted image 20260605084113.png]]
A `PostComment` is meaningful only for the version of the `Post` that was reviewed.

##### Example
```java
Alice loads Post (version=0)

Bob modifies Post version -> 1

Alice adds comment based on old Post content
---------------------------------------------------
Without optimistic locking:

Post -> Chapter 17
Comment -> "Chapter 16 is about Caching"  // ❌ Logical inconsistency
```

**Acquiring an Optimistic Lock** `entityManager.lock(post, LockModeType.OPTIMISTIC)`

`OPTIMISTIC` does **NOT** acquire a database lock. Instead, it schedules a **version check** before transaction commit.
#### What Hibernate Does

At commit time:
```sql
INSERT INTO post_comment (post_id, review, version, id) 
VALUES (1, 'Chapter 16 is about Caching.', 0, 1)

SELECT version FROM post WHERE id = 1
```

Then compares
```text
Loaded version   = 0
Current version  = 1
```

Result `OptimisticLockException`, So **Transaction is rolled back.**

---
### Inconsistency Risk (Window of Opportunity)
Unfortunately, this kind of application-level check is always prone to inconsistencies due to **bad timing.**

**Problem:**
1. Hibernate checks version
2. Version matches
3. Another transaction updates Post
4. Current transaction commits
![[Pasted image 20260605085326.png]]
A concurrent update may occur in that window.
##### Stronger Protection
**To prevent** such an incident, the `LockModeType.OPTIMISTIC` should be accompanied by a **shared lock** acquisition:

Combine:
```java
entityManager.lock(post, LockModeType.OPTIMISTIC);
entityManager.lock(post, LockModeType.PESSIMISTIC_READ);
```


> [!NOTE] Note
> **OPTIMISTIC + PESSIMISTIC_READ**
> * PESSIMISTIC_READ already prevents concurrent updates, therefore the optimistic version check becomes mostly redundant.
> 
> The combination eliminates the small race window between the optimistic version check and transaction commit.
> 
> That's why optimistic locking is usually preferred for **user-think-time** workflows.
> 
> 

