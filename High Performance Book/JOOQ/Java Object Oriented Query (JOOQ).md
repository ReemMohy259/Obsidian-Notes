#### Why Use jOOQ?

- Enterprise applications cannot rely only on Hibernate/JPA only.
- Many use cases require **native SQL**:
    - Reporting
    - Analytics
    - ETL (Extract, Transform, Load)
    - Database-specific features (Window Functions, CTEs, UPSERT/MERGE)
- JPA/Hibernate native queries are mostly suitable for **static SQL**.
- Building **dynamic SQL** becomes difficult with JPA/Hibernate alone.

> jOOQ is a **query builder framework**, that provides a type-safe Java API for building dynamic, database-specific SQL queries.

---
# How jOOQ Works

The entry point is:
```java
DSLContext sql = DSL.using(connection, SQLDialect.POSTGRES_9_5);
```

`DSLContext` requires:
1. JDBC Connection
2. Database Dialect

```text
Java Query
  ↓
jOOQ
  ↓
Database-specific SQL
```

---
# DML Support

jOOQ supports all common SQL operations:
#### DELETE
```java
sql.deleteFrom(table("post")).execute();
```

```sql
DELETE FROM post
```

#### INSERT
```java
sql.insertInto(table("post"))
   .columns(field("id"), field("title"))
   .values(1L, "Book")
   .execute();
```

```sql
INSERT INTO post(id, title)
VALUES (1, 'Book')
```

#### UPDATE
```java
sql.update(table("post"))
   .set(field("title"), "New Title")
   .where(field("id").eq(1))
   .execute();
```

```sql
UPDATE post
SET title = 'New Title'
WHERE id = 1
```

#### SELECT
```java
sql.select(field("title"))
   .from(table("post"))
   .where(field("id").eq(1))
   .fetch();
```

```sql
SELECT title
FROM post
WHERE id = 1
```

---
### execute() vs fetch()

|Method|Purpose|
|---|---|
|`execute()`|INSERT, UPDATE, DELETE|
|`fetch()`|SELECT|

#### execute()
Returns affected row count.
```java
int rows = query.execute();
```

#### fetch()
Returns query results.
```java
query.fetch();
```

---
# Java-Based Schema (Type-Safe Queries)

All the previous queries were referencing the database schema explicitly. However, just like JPA defines a Metamodel API for Criteria queries, jOOQ allows generating a Java-based schema that mirrors the one in the database.

Instead of:
```java
table("post")
field("title")
```

jOOQ can generate Java classes from the database schema.
```java
POST
POST.ID
POST.TITLE
```

##### Type-Safe Version
```java
sql.insertInto(POST)
   .columns(POST.ID, POST.TITLE)
   .values(1L, "Book")
   .execute();
```

#### Benefits of Java-Based Schema

##### Compile-Time Safety
Renaming a column:
```text
Database column renamed
       ↓
Compilation error
```

instead of:
```text
Runtime SQL failure
```

##### IDE Autocomplete
Improves productivity and reduces typos.
##### Strong Typing
jOOQ knows:
- Table types
- Column types
- Procedure parameters
- Query result types
**at compile time.**
---
# Upsert

==Upsert = INSERT + UPDATE==

Algorithm:
```text
Try INSERT
    ↓
Success? → Done
    ↓ No
Row already exists
    ↓
Execute UPDATE
```

Used when a row may or may not already exist.
#### Why Upsert Matters
Without upsert:
```java
if(exists(id)) {
    update(...);
} else {
    insert(...);
}
```

**Problems:**
- Requires multiple queries.
- Race conditions under concurrency.

Example:
```text
Alice checks → row doesn't exist

Bob checks → row doesn't exist

Alice INSERT

Bob INSERT → Duplicate Key Exception
```

==Upsert solves this atomically inside the database.==

### jOOQ Upsert API

```java
public void upsertPostDetails(DSLContext sql, BigInteger id, String owner, Timestamp timestamp) {  
    sql  
		.insertInto(POST_DETAILS)  
		.columns(POST_DETAILS.ID, POST_DETAILS.CREATED_BY,POST_DETAILS.CREATED_ON) 
		.values(id, owner, timestamp)  
		.onDuplicateKeyUpdate()  
		.set(POST_DETAILS.UPDATED_BY, owner)  
		.set(POST_DETAILS.UPDATED_ON, timestamp)  
		.execute();  
}
```

Intent:
```text
Insert if missing
Update if existing
```

#### Concurrent Example

Two users execute:
```java
upsertPostDetails(..., "Alice");
upsertPostDetails(..., "Bob");
```

Possible outcome:
```text
Alice → INSERT
Bob → UPDATE
```

==No exceptions. No race conditions.==


> [!NOTE] Database Implementations
> JOOQ is going to **translate** the upsert Java-based operation to the specific syntax employed by the underlying relational database.
> 
> **Portability** is jOOQ's responsibility

---
# Batch Updates

==Reduce database roundtrips.==

Instead of:
```text
INSERT row 1
INSERT row 2
INSERT row 3
```
(3 network trips)

Use:
```text
Single batch
```
(1 network trip)
### jOOQ Batch API

```java
BatchBindStep batch =
    sql.batch(
        sql.insertInto(POST, POST.TITLE)
           .values("?")
    );

for(int i = 0; i < 3; i++) {
    batch.bind("Post " + i);
}

batch.execute();
```

#### Generated SQL (MySQL)

```sql
INSERT INTO post(title)
VALUES
    ('Post 0'),
    ('Post 1'),
    ('Post 2')
```
Single roundtrip.
### Hibernate vs jOOQ

#### Hibernate
JDBC batching works for `SEQUENCE` and `TABLE` generators only. But not effectively for `IDENTITY` 
because Hibernate must execute INSERT immediately to obtain generated ID.
#### jOOQ

* No entity state management.
* Pure SQL execution.
* Therefore batching works even with **MySQL** `AUTO_INCREMENT`

> [!NOTE] Practical Recommendation
> **Lots of INSERTs on MySQL?**
> Prefer jOOQ batching over Hibernate.

---

# Complex Queries

#### Main Idea
One of jOOQ's biggest strengths is the ability to express **very complex SQL queries** using a type-safe Java DSL.

Supports:
- Derived Tables
- Window Functions
- CTEs (Common Table Expressions)
- Recursive CTEs
- Subqueries
- Database-specific SQL features

```text
jOOQ excels at complex SQL.

Supports:
    - Recursive CTEs
    - Window Functions
    - Derived Tables
    - Complex ranking queries

Large queries can be split into reusable methods, improving readability and maintainability.
```

---
