#### Purpose
- Combines:
    - **Pessimistic Locking** (`FOR UPDATE`)
    - **Version Increment**
- Increments the entity version **immediately** after acquiring the database lock.
- Used when child changes must be coordinated through a parent entity version.

| Lock Mode                     | Version Increment                  |
| ----------------------------- | ---------------------------------- |
| `OPTIMISTIC_FORCE_INCREMENT`  | At transaction commit              |
| `PESSIMISTIC_FORCE_INCREMENT` | Immediately after lock acquisition |

---
### Flow

```java
entityManager.find(Repository.class, 1L,
    LockModeType.PESSIMISTIC_FORCE_INCREMENT
);

Commit commit = new Commit(repository); 
commit.getChanges().add(new Change("FrontMatter.md", "0a1,5...")); commit.getChanges().add(new Change("HibernateIntro.md", "17c17...")); 

entityManager.persist(commit);
```

Hibernate executes:

```sql
SELECT *
FROM repository
WHERE id = 1
FOR UPDATE

UPDATE repository
SET version = version + 1
WHERE id = 1 AND version = 0

INSERT INTO commit (repository_id, id) VALUES (1, 2) 
INSERT INTO commit_change (commit_id, diff, path) 
VALUES (2, '0a1,5...', 'FrontMatter.md') 
INSERT INTO commit_change (commit_id, diff, path) 
VALUES (2, '17c17...', 'HibernateIntro.md')
```

**Result**
```text
1. Acquire row lock
2. Increment version immediately
3. Continue transaction
```

---
### Fail-Fast Behavior

```Java
Repository repository = entityManager.find(Repository.class, 1L);  

executeSync(() -> {  
    doInJPA(_entityManager -> {  
        Repository _repository = _entityManager.find(Repository.class, 1L,  
                LockModeType.PESSIMISTIC_FORCE_INCREMENT);  
        Commit _commit = new Commit(_repository);  
        _commit.getChanges().add(new Change("Intro.md", "0a1,2..."));  
        _entityManager.persist(_commit);  
    });  
});  

entityManager.lock(repository, LockModeType.PESSIMISTIC_FORCE_INCREMENT);
```

![[Pasted image 20260605100836.png]]

==Alice fails immediately instead of discovering the conflict at commit time.==

---
### Coordinating Concurrent Transactions
Once a transaction acquires the `PESSIMISTIC_FORCE_INCREMENT` lock and increments the entity version, **no other transaction** can acquire a `PESSIMISTIC_FORCE_INCREMENT` lock because the second select statement is **blocked** until the first transaction releases the row-level physical lock.

![[Pasted image 20260605101703.png]]
##### Result

Transactions execute sequentially `Alice` -> `Bob` instead of competing and producing optimistic lock failures, therefore, **reducing** the likelihood of getting an `OptimisticLockingException`

---

## Comparison

|Feature|OPTIMISTIC_FORCE_INCREMENT|PESSIMISTIC_FORCE_INCREMENT|
|---|---|---|
|Version increment|Commit time|Immediately|
|Database lock|❌|✅ (`FOR UPDATE`)|
|Conflict detection|Usually at commit|Immediate|
|Blocks concurrent writers|❌|✅|
|Likelihood of OptimisticLockException|Higher|Lower|

---

## Typical Use Case

```text
Repository
    ├── Commit A
    ├── Commit B
    └── Commit C
```

Each commit must be applied sequentially.

Use parent entity (`Repository`) as a coordination point:

```java
PESSIMISTIC_FORCE_INCREMENT
```

so only one transaction can advance the repository version at a time.

