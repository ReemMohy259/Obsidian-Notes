Each entity in the hierarchy has its **own table**. **Subclass table PK is also FK to parent table.**

![[Pasted image 20260513181223.png|495]]

* **Used Annotations:**
	* `@Inheritance(strategy = InheritanceType.JOINED)`
	* `@PrimaryKeyJoinColumn` -> Child table primary key is ALSO a foreign key to parent table.
---
### Persisting Entities
Persisting subclass entity requires:
1. insert into parent table
2. insert into subclass table

**Example:**
```sql
INSERT INTO topic ...
INSERT INTO post ...
```

---
### Polymorphic Queries

Querying parent entity `SELECT t FROM Topic t`

* Requires joins **with all subclass** tables. Hibernate determines subtype based on which join matches.
```sql
FROM topic t
LEFT JOIN post p
LEFT JOIN announcement a
```

---
### Performance
#### Advantages
* Proper normalization  
* No unnecessary nullable columns  
* Better DB integrity (`NOT NULL` works naturally)
#### Disadvantages
* More joins  
* Slower polymorphic queries  
* More inserts per entity  
* Larger index footprint

---
### Important Performance Rule
If there are `N subclasses` Hibernate usually needs `N + 1 joins` for polymorphic queries.