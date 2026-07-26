Entire inheritance hierarchy is stored in **one database table**. This is the **default JPA inheritance strategy**.
* **Used Annotations:**
	* `@Inheritance`
	* `@Inheritance(strategy = InheritanceType.SINGLE_TABLE)`
### Table Structure

**All fields from:**
- base class (`Topic`)
- subclasses (`Post`, `Announcement`)
are stored in a single table with **extra column for differentiation** called `DTYPE` (Discriminator column)
##### Example
![[Pasted image 20260513170727.png|147]]

> [!NOTE] Discriminator Column
> Hibernate uses a discriminator column (DTYPE) to determine the actual entity type.
> 
> **Expected Values (Subclass name):**
> * Post
> * Announcement

---
### Persisting Entities

```java
Post post = new Post();
post.setContent("Best practices");

entityManager.persist(post);

---------------------------------------------------
INSERT INTO topic
(board_id, createdOn, owner, title, content, DTYPE, id)
VALUES (..., 'Best practices', 'Post', 2)
```

---
### Polymorphic Queries

> [!NOTE] Major advantage of inheritance mapping.
> One advantage of using inheritance in the Domain Model is the support for **polymorphic queries** -> ==A query on a parent entity that automatically returns objects of its subclasses.==

#### In Single Table Inheritance Strategy:
* Query Parent Entity `SELECT t FROM Topic t`
* **Hibernate**:
	1. queries `topic` table
	2. checks `DTYPE`
	3. instantiates correct subclass `Post` or `Announcement`
---
###  Polymorphic Associations

 `TopicStatistics -> Topic` using `@OneToOne` relation:

```java
@OneToOne
private Topic topic;
```

Can reference:
- `Post`
- `Announcement`
Hibernate resolves actual subclass using `DTYPE`.

Using `@OneToMany` relation:

```java
@OneToMany(mappedBy = "board")
private List<Topic> topics;
```
Collection may contain:
- `Post`
- `Announcement`
---
### Performance Characteristics

#### Advantages:
* **Very Fast Reads/Writes** (Because only one table, no inheritance joins and no unions) 
* **Efficient Associations** (Even polymorphic associations need only single join)
* **Simple Query Execution** (Hibernate only checks discriminator value)
#### Disadvantages:
* **Subclass-specific columns cannot be** `NOT NULL`, for example:
	* `Announcement` rows have `content = NULL`
	- `Post` rows have `validUntil = NULL`


> [!NOTE] Data Integrity Problem
> Database cannot enforce subclass constraints naturally.
> **This weakens:**
> * Schema consistency
> * ACID integrity guarantees

---
### Solutions for Data Integrity

#### 1. CHECK Constraints (Best Option)
Validate subclass columns based on `DTYPE`.
##### Example
```sql
ALTER TABLE topic
ADD CONSTRAINT post_content_check
CHECK (
    CASE WHEN DTYPE = 'Post'
    THEN CASE WHEN content IS NOT NULL THEN 1 ELSE 0 END
    ELSE 1
    END = 1
)
```

> [!NOTE] Note
> All Databases Supporting CHECK Constraints, except Older MySQL versions ignore CHECK constraints.

#### 2. Database Triggers
Alternative for older databases.
* **BEFORE INSERT Trigger**
* **Need UPDATE Triggers Too**
##### Example
```sql
CREATE TRIGGER post_content_check
BEFORE INSERT ON topic
```

#### 3. Application-Level Validation
* **Bean Validation** `@NotNull`
*  **JPA Lifecycle Callbacks** `@PrePersist` and `@PreUpdate`

