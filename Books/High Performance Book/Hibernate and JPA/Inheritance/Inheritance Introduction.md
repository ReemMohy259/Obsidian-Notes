Java, like any other **object-oriented programming language**, makes heavy use of inheritance and polymorphism. Inheritance allows defining class hierarchies that offer different implementations of a common interface.

Conceptually, the Domain Model defines both data (e.g. persisted entities) and behavior (business logic). Nevertheless, **inheritance** is more useful for **varying behavior rather than reusing data** (**composition** is much more suitable for sharing structures). Even if the data (persisted entities) and the business logic (transactional services) are decoupled, inheritance can still help varying business logic.
#### Example
![[Pasted image 20260513164149.png|605]]

* **Base Class** `Topic`, where all subclasses share common fields
* **Subclasses**:
	* `Post` Adds "content"
	* `Announcement` Adds "validUntil"
---
### Object-Relational Impedance Mismatch
Inheritance is **natural** in **OOP**. **Relational databases** do **not** naturally support inheritance.
Most databases **rely on:**
- tables 
- rows
- joins
instead of object hierarchies. So inheritance must be emulated using table relationships.
---
### Inheritance Mapping Strategies
**Three** inheritance mapping models.
#### 1. [[Single Table Inheritance]]
which uses a **single database table** to represent all classes in a given inheritance hierarchy.
#### 2.  [[Class Table Inheritance (Joined Inheritance)]]
which maps **each class to a table**, and the inheritance association is resolved through **table joins.**
#### 3. [[Concrete Table Inheritance (Table Per Class)]]
where each table defines all fields that are either defined in the subclass or inherited from a superclass.
Each **concrete** subclass has a full table containing:
- **subclass fields**
- **inherited fields**

> [!NOTE] Note
> The **JPA** specification defines all these three inheritance mapping models through the following **strategies**:
> * InheritanceType.**SINGLE_TABLE** 
> * InheritanceType.**JOINED** 
> * InheritanceType.**TABLE_PER_CLASS**

---
### Additional JPA Option

### `@MappedSuperclass`

* Inheritance exists **only** in **Java**.
* Superclass is **not** mapped as an entity/table.
* Used only to **reuse** mappings.
#### When to Use it
Best for reusable technical/common fields:
##### Examples
- id
- audit fields
    - createdAt
    - updatedAt
    - createdBy
- versioning
- soft delete flags

```java
@MappedSuperclass  
public abstract class AuditableEntity {  
	@Id  
	@GeneratedValue  
	protected Long id;  
	  
	@CreationTimestamp  
	protected Instant createdAt;  
	  
	@UpdateTimestamp  
	protected Instant updatedAt;  
	  
	@Version  
	protected Long version;  
}
```

---
### Performance Perspective
Inheritance support is implemented at the **ORM layer** because databases lack native inheritance support.

**Therefore:**
- ORM may generate expensive SQL
- joins/unions can hurt performance
- inheritance strategy directly impacts query efficiency

**Main concern:**
> Understanding inheritance trade-offs and their performance implications.

---
### Rule of Thumb Summary
| **Strategy**     | **Best At**             | **Main Problem**         |
| ---------------- | ----------------------- | ------------------------ |
| SINGLE_TABLE     | Performance             | Nullable columns         |
| JOINED           | Normalization           | Many joins               |
| TABLE_PER_CLASS  | Simple subclass queries | UNION-heavy polymorphism |
| MappedSuperclass | Mapping reuse           | No polymorphism          |
