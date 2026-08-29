# Introduction

JPA provides two row-level pessimistic lock types:
* `PESSIMISTIC_READ`   -> Shared lock (read lock)
* `PESSIMISTIC_WRITE` -> Exclusive lock (write lock)

> Database vendors implement shared/exclusive locks differently, so JPA abstracts these differences, and each database system defines its own syntax.

##### Oracle
- Supports only **exclusive locks**.
- Uses `FOR UPDATE` select clause
- Locked rows cannot be modified or locked by other transactions until commit/rollback.
##### SQL Server
- Supports both **exclusive locks** and **shared locks**.
- Uses **table hints**
- `WITH (HOLDLOCK, ROWLOCK)` for shared locks 
- `WITH (UPDLOCK, HOLDLOCK, ROWLOCK)` for exclusive locks
##### PostgreSQL
- Supports both **exclusive locks** and **shared locks**.
- Uses **select clause** with multiple locking clauses
- `FOR SHARE` for shared locks 
- `FOR UPDATE` for exclusive locks
##### MySQL
- Supports both **exclusive locks** and **shared locks**.
- Uses **select clause** with multiple locking clauses
- `LOCK IN SHARE MODE` for shared locks 
- `FOR UPDATE` for exclusive locks
---
### Hibernate Abstraction
- Developers use JPA lock modes.
- Hibernate **automatically** generates the correct SQL for the current database dialect.
##### Example -> Acquiring an Exclusive Lock
```java
Post post = entityManager.find(
    Post.class,
    1L,
    LockModeType.PESSIMISTIC_WRITE
);
------------------------- PostgreSQL SQL ---------------------------
SELECT *
FROM post
WHERE id = 1
FOR UPDATE
```

##### Example -> Acquiring an Shared Lock
```java
Post post = entityManager.find(
    Post.class,
    1L,
    LockModeType.PESSIMISTIC_READ
);
------------------------- PostgreSQL SQL ---------------------------
SELECT *
FROM post
WHERE id = 1
FOR SHARE
```

**Important Note**
> If the database does not support shared locks (e.g., Oracle), Hibernate upgrades `PESSIMISTIC_READ` to an exclusive lock (`FOR UPDATE`).
---
### Locking an Already Loaded Entity
Although it is much more convenient to lock entities at the moment they are fetched from the database, entities can also be locked even **after they are loaded** in the currently running Persistence context.
```java
Post post = entityManager.find(Post.class, 1L);

entityManager.lock(
    post,
    LockModeType.PESSIMISTIC_WRITE
);
```

Hibernate executes:
1. **Load entity**    
```sql
SELECT * FROM post WHERE id = 1
```
2. **Acquire lock**
```sql
SELECT id
FROM post
WHERE id = 1
  AND version = 0
FOR UPDATE
```


> [!NOTE] Requirement: Entity Must Be Managed
> **The JPA** `lock()` method accepts only a **managed entity**. Otherwise, an `IllegalArgumentException` is being thrown indicating that the entity is **detached**.
> 
> On the other hand, the **Hibernate native API** offers entity **reattachment** upon locking
> **Steps:**
> 1- Acquires database lock.
> 2- Makes entity managed again.

##### Example:
```Java
Post post = doInJPA(entityManager -> { 
	return entityManager.find(Post.class, 1L); 
}); // Detached entity

doInJPA(entityManager -> { 
	Session session = entityManager.unwrap(Session.class);
	session.buildLockRequest( new LockOptions(LockMode.PESSIMISTIC_WRITE))
	.lock(post); 
	post.setTitle("High-Performance Hibernate"); 
});
```

```SQL
SELECT *
FROM post 
WHERE id = 1

-- Lock and reattach 
SELECT id 
FROM post 
WHERE id = 1 AND version = 0 
FOR UPDATE 

UPDATE post 
SET title = 'High-Performance Hibernate' 
WHERE id = 1 AND version = 0 
```
---
# Lock Scope
#### Default Lock Scope
- By default, a lock applies **only to the target entity**.
- Child associations are **not automatically locked**.

![[Pasted image 20260605011305.png]]

> Locking `Post` does not necessarily lock `PostDetails` or `PostComments`.
##### Preferred Approach: Lock Using Entity Queries
When a query fetches an entire entity graph and applies a lock:

```java
Post post = entityManager.createQuery(
		"select p " + 
		"from Post p " + 
		"join fetch p.details " + 
		"join fetch p.comments " + 
		"where p.id = :id", Post.class)
	.setParameter("id", 1L)
    .setLockMode(LockModeType.PESSIMISTIC_WRITE)
    .getSingleResult();
```

Hibernate generates:

```sql
SELECT *
FROM post p
INNER JOIN post_details pd ON p.id = pd.id 
INNER JOIN post_comment pc ON p.id = pc.post_id
WHERE p.id = 1
FOR UPDATE
```

**Result**
- `FOR UPDATE` applies to the whole result set.
- The lock scope depends on the query filtering criteria.

✅ Recommended way to lock an entity graph.

---
### Cascading Lock Requests
Aside from entity queries, Hibernate can also **propagate** a lock acquisition request from a parent entity to its children when using direct fetching. For this purpose, the child associations must be **annotated** with the Hibernate specific `CascadeType.Lock` attribute.

##### Example:
```java
@OneToMany(cascade = CascadeType.ALL)
private List<PostComment> comments;

@OneToOne(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
private PostDetails details;
```

Even with `CascadeType.LOCK`, Hibernate does **not** cascade locks unless **scope attribute** must be set to **true**:

```java
session.buildLockRequest(
    new LockOptions(LockMode.PESSIMISTIC_WRITE)
)
.setScope(true)
.lock(post);
```

#### Managed Entity Limitation
For a **managed** entity:
```java
Post post = entityManager.find(Post.class, 1L);

entityManager.unwrap(Session.class)
.buildLockRequest( 
	new LockOptions(LockMode.PESSIMISTIC_WRITE))
.setScope(true)
.lock(post);
```

Hibernate locks only the `Post` entity:
```sql
SELECT *
FROM post p
WHERE p.id = 1

SELECT id 
FROM post 
WHERE id = 1 AND version = 0 
FOR UPDATE
```


> [!IMPORTANT] Note
> **For managed entities**, Hibernate **does not** cascade the lock acquisition request even if the **scope attribute is provided**, therefore, the **entity query alternative is preferred**.

#### Detached Entity Graph Locking
When locking a detached entity graph, Hibernate is going to **reattach** every entity that enabled cascade propagation while also propagating the lock request.

```java
Post post = doInJPA(entityManager -> {  
    return entityManager.createQuery(  
                    "select p " +  
					"from Post p " +  
					"join fetch p.details " +  
					"join fetch p.comments " +  
					"where p.id = :id", Post.class)  
            .setParameter("id", 1L)  
            .getSingleResult();  
});  // Detached

doInJPA(entityManager -> {  
    entityManager.unwrap(Session.class)  
            .buildLockRequest(  
                    new LockOptions(LockMode.PESSIMISTIC_WRITE))  
            .setScope(true)  
            .lock(post);  
});
```

Hibernate locks:
```sql
SELECT *
FROM post p  
INNER JOIN post_details pd ON p.id = pd.id  
INNER JOIN post_comment pc ON p.id = pc.post_id  
WHERE p.id = 1  

SELECT id FROM post_comment WHERE id = 2 AND version = 0 FOR UPDATE  
SELECT id FROM post_comment WHERE id = 3 AND version = 0 FOR UPDATE 
 
SELECT id FROM post_details WHERE id = 1 AND version = 0 FOR UPDATE  

SELECT id FROM post WHERE id = 1 AND version = 0 FOR UPDATE
```

All previously loaded child entities are locked:
- Post
- PostDetails
- Every PostComment

#### Lazy Loading Impact
The lock acquisition request is cascaded to child collections only if the collection is **already initialized**
##### Example
```Java
Post post = doInJPA(entityManager -> (Post) entityManager.find(Post.class, 1L));

--------------------------------------------------------------------------
SELECT *
FROM post_details
WHERE p.id = 1  // Reattach Post details only

SELECT id from post_details WHERE id =1 AND version = 0 FOR UPDATE 
SELECT id from post WHERE id =1 AND version = 0 FOR UPDATE
```

##### Key Point
```Java
Detached Entity Lock Cascading

@OneToOne / @ManyToOne
    -> can be locked through proxies
    -> Hibernate knows target identifier
    -> lock propagation works

@OneToMany / @ManyToMany
    -> lock propagation only if collection is initialized
    -> Hibernate must know child entity identifiers
    -> uninitialized collections are not fetched just for locking
```

==This is exactly why Vlad recommends query-level locking: the query result already contains every row that should be locked, so Hibernate doesn't need to guess or initialize lazy associations.==
##### Scalability Warning
Locking more rows than necessary reduces concurrency.
**Effects**:
- Longer blocking periods
- More waiting transactions
- Lower scalability
- Increased contention

> Always lock the smallest set of rows required.
---
# Lock Timeout

- Prevent waiting indefinitely when acquiring a pessimistic lock.    
- Define how long a transaction waits before giving up.
##### Timeout Configuration
**JPA**
```java
Collections.singletonMap("javax.persistence.lock.timeout", TimeUnit.SECONDS.toMillis(3))
```

**Hibernate**
```java
.setTimeOut((int) TimeUnit.SECONDS.toMillis(3))
```

**SQL**
```SQL
SELECT id 
FROM post 
WHERE id = 1 AND version = 0 
FOR UPDATE WAIT 3
```
---
### Timeout Modes

| Value                | Behavior                                           |
| -------------------- | -------------------------------------------------- |
| `> 0`                | Wait up to specified time, fail with **exception** |
| `0` (`NO_WAIT`)      | Fail immediately if row is locked                  |
| `-2` (`SKIP_LOCKED`) | Skip locked rows                                   |

#### NO_WAIT
```sql
FOR UPDATE NOWAIT
```

- No waiting.
- Throws exception if row is already locked.
- Supported only by some databases (Oracle, PostgreSQL).
#### SKIP_LOCKED
```sql
FOR UPDATE SKIP LOCKED
```

- Ignores already locked rows.
- No waiting.
- No exception.
- Returns only unlocked rows.

**Ideal for:**
- Job queues
- Batch processing
- Moderation systems
- Multiple workers processing tasks concurrently
##### Example

```text
Posts: [0,1,2,3]

Alice locks: [0,1]

Bob using SKIP_LOCKED:
    skips [0,1]
    locks [2,3]
```

Workers never block each other.
