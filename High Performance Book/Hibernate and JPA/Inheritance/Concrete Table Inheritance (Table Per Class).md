`@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)` Each **concrete** subclass has its own **independent table** containing:
- inherited parent fields
- subclass-specific fields
No shared inheritance table joins. ==If the class is declared as **abstract** no generated table for it== 
![[Pasted image 20260515163855.png|552]]

> [!NOTE] Notes
> * **No Shared Parent Table** -> Unlike `JOINED` parent fields are duplicated in subclass tables
> * **Tables are independent** -> No FK relationships between inheritance tables

---
### Persisting Entities
Each subclass entity requires only **1 INSERT statement** unlike `JOINED` that required 2 insert statements
##### Persist Post Entity
```sql
INSERT INTO post (...)
VALUES (...)
```

---
### Polymorphic Queries

Querying Parent Entity `SELECT t FROM Topic t` 

* Hibernate must **combine all subclass** tables using, `UNION ALL`
##### Generated SQL Concept
```SQL
SELECT
    t.id AS id1_3_, t.board_id AS board_id5_3_, t.createdOn AS createdO2_3_,
    t.owner AS owner3_3_, t.title AS title4_3_, t.content AS content1_2_,
    t.validUntil AS validUnt1_0_, t.clazz_ AS clazz_
FROM (
    SELECT
        id, createdOn, owner, title, board_id,
        CAST(NULL AS VARCHAR(100)) AS content,
        CAST(NULL AS TIMESTAMP) AS validUntil,
        0 AS clazz_
    FROM topic

    UNION ALL
    
    SELECT
        id, createdOn, owner, title, board_id, content,
        CAST(NULL AS TIMESTAMP) AS validUntil,
        1 AS clazz_
    FROM post

    UNION ALL

    SELECT
        id, createdOn, owner, title, board_id,
        CAST(NULL AS VARCHAR(100)) AS content,
        validUntil,
        2 AS clazz_
    FROM announcement
) t
WHERE t.board_id = 1;
```


> [!NOTE] Note
> `CAST(NULL AS VARCHAR(100)) AS content` **Needed because** all UNION queries must return:
> * same columns
> * same types
> * same order
> 
> so we have to make `content` column to be null in `Announcement` entity, same as `validUntil` in `Post` entity

---
### Identity Generator Limitation

The **identity generator is not allowed** with this strategy because rows belonging to **different subclasses** would share the **same identifier**, therefore **conflicting** in polymorphic `@ManyToOne` or `@OneToOne` associations.
###### Example:
```java
@ManyToOne
private Topic topic;
```

**If FK = 1:**
- is it `Post(1)`?
- or `Announcement(1)`?
**Ambiguous identity.**

---
### Performance Characteristics

#### Advantages

* **Fast Writes** -> Only one insert per subclass entity. Faster than `JOINED`.
* **Efficient Subclass Queries** -> `SELECT p FROM Post p` Touches only `post` table. No joins/unions.
* **No Nullable Subclass Columns** -> Each table stores only relevant fields
#### Disadvantages

 * **Expensive Polymorphic Queries** -> Parent queries require `UNION ALL` across all subclass tables.
 * **Scalability Problem** More subclasses ⇒ larger UNION queries.
 * **Data Duplication** -> Inherited fields repeated in every subclass table.
 * **Schema Maintenance Cost** -> Changing parent fields requires modifying all subclass tables.

Rule of thumb:
`More subclass tables = slower polymorphic queries`
