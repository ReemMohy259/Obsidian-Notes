#### Purpose
- Forces an entity version increment even if the entity itself was not modified.
- Useful when a child entity change should be reflected in the parent entity version.
- Allows coordinating changes across related entities.

> [!NOTE] Note
> The `@Version` attribute **should never have a setter method** because this attribute is managed **automatically** by Hibernate. To increment the version of a given entity, one of the two `FORCE_INCREMENT` lock strategies must be used instead

#### Example

![[Pasted image 20260605092620.png]]
Every new `Commit` should increment the `Repository` version.
##### Usage
```java
Repository repository = entityManager.find(Repository.class, 1L,
	LockModeType.OPTIMISTIC_FORCE_INCREMENT
);
    
Commit commit = new Commit(repository); 
commit.getChanges().add(new Change("FrontMatter.md", "0a1,5...")); commit.getChanges().add(new Change("HibernateIntro.md", "17c17...")); 

entityManager.persist(commit);
```
##### Generated SQL
```sql
SELECT * FROM repository WHERE id = 1

INSERT INTO commit (repository_id, id) VALUES (1, 2)

INSERT INTO commit_change (commit_id, diff, path) 
VALUES (2, '0a1,5...', 'FrontMatter.md') 
INSERT INTO commit_change (commit_id, diff, path) 
VALUES (2, '17c17...', 'HibernateIntro.md')

UPDATE repository
SET version = version + 1
WHERE id = 1 AND version = 0
```

Even though `Repository` was not modified directly, its version is incremented.

---
#### Concurrent Example

* **Initial State** `Repository version = 0`
* **Alice** Load Repository (v0) with `OPTIMISTIC_FORCE_INCREMENT` and try to insert a new comment
* **Bob** Load Repository (v0) with `OPTIMISTIC_FORCE_INCREMENT` and try to insert a new comment
* **Bob Commits First** `UPDATE Repository` Success ✅
* **Alice Commits Later** `OptimisticLockException` Fails ❌

![[Pasted image 20260605093443.png]]