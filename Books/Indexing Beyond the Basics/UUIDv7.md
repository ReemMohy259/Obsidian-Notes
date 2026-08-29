**UUIDv7** is a UUID format designed to be **time-ordered** while still providing a very large amount of uniqueness.

- Based on the **Unix timestamp in milliseconds**.
- Contains randomness to ensure uniqueness.
- UUIDs generated around the same time are generally ordered by creation time.
- Particularly useful as **database primary keys**, because its ordering can improve index locality compared with completely random UUIDs.
- It does **not** guarantee a strict ordering when multiple UUIDs are generated within the same millisecond.

### UUIDv4 vs UUIDv7

||UUIDv4|UUIDv7|
|---|---|---|
|Main characteristic|Random|Time-ordered|
|Timestamp|❌ No|✅ Yes|
|Randomness|~122 bits|~74 bits|
|Sortable by creation time|❌ No|✅ Generally|
|Database index locality|Worse|Better|
|Common use|General-purpose unique IDs|Database IDs / distributed systems|
|Privacy|Doesn't reveal creation time|Reveals approximate creation time|

> **Index locality** refers to ==keeping related data physically close together on a storage disk or in memory to speed up database performance==

Example:

```text
UUIDv4:
550e8400-e29b-41d4-a716-446655440000

UUIDv7:
0198c5f2-7abc-7def-8123-456789abcdef
```

The **first part of a v7 UUID contains the timestamp**, which is why v7 values can be ordered approximately by creation time.

---

## Using UUIDv4 in Java

Java's standard `UUID` API provides UUIDv4 directly:

```java
UUID id = UUID.randomUUID();

System.out.println(id);
System.out.println(id.version()); // 4
```

For example:

```java
@Entity
public class User {

    @Id
    private UUID id;

    @PrePersist
    void generateId() {
        if (id == null) {
            id = UUID.randomUUID();
        }
    }
}
```

---

## Using UUIDv7 in Java

The approach depends on your **Java version and UUIDv7 library**.

With a UUIDv7 library, the usage is typically along these lines:

```java
UUID id = UuidCreator.getTimeOrderedEpoch();
```

Then:

```java
System.out.println(id.version()); // 7
```

For a JPA entity, you can generate it yourself:

```java
@Entity
public class User {

    @Id
    private UUID id;

    @PrePersist
    void generateId() {
        if (id == null) {
            id = UuidCreator.getTimeOrderedEpoch();
        }
    }
}
```

### In short

```text
UUIDv4 → random UUID
UUIDv7 → timestamp + random data
```

For **database primary keys**, UUIDv7 is often preferable when you want UUIDs while also getting better index locality and approximate chronological ordering.

---
### Trade-Offs of UUIDv7 

- **UUIDv7 generally improves database performance** compared with UUIDv4 because its time-ordered nature provides better index locality.
- This is especially beneficial for traditional relational databases such as **PostgreSQL and MySQL**.
- However, in **horizontally scaled distributed databases**, sequential IDs can cause **hotspots** by concentrating new writes in the same database partition/shard.
- This can become a bottleneck when many clients insert records simultaneously.

---
### UUIDv7 and Hotspots in Distributed Databases

A **hotspot** occurs when a large amount of read/write traffic is concentrated on a **single database node or partition**, while other nodes remain underutilized. This creates a performance bottleneck.

**Why can UUIDv7 cause hotspots?**
UUIDv7 is **time-sortable**, so newly generated IDs are close to each other. In some distributed databases that partition data based on the primary key, this can cause new writes to be directed to the **same partition/node**:

```text
UUIDv4 (Random)
─────────────────────────────
Server A  ████
Server B  ███
Server C  ████
Server D  ███
           ↑
    Writes are distributed


UUIDv7 (Time-ordered)
─────────────────────────────
Server A  ████████████████  ← New writes
Server B  ░░░
Server C  ░░░
Server D  ░░░
           ↑
        HOTSPOT
```

### UUIDv4 vs UUIDv7

||UUIDv4|UUIDv7|
|---|---|---|
|ID generation|Random|Time-ordered|
|Write distribution|More evenly distributed|Can cluster|
|Hotspot risk|**Low**|**Higher** in some distributed DBs|
|Recent-data ordering|❌|✅|
|Index locality|Lower|Better|

**Key takeaway:**
> UUIDv7 provides better ordering and index locality, but in highly distributed databases, its sequential nature can create write hotspots. Techniques such as **sharding, hash-based partitioning, or adding a distribution prefix** can help mitigate this problem.