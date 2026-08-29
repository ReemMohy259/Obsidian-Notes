> See this video for more visualization [Watch me](https://www.youtube.com/watch?v=o_2psWN8k_c&pp=ygUHQisgdHJlZQ%3D%3D)
## Why B+ Trees?

- Databases store data on **disk pages/blocks** (typically 4–16 KB). Disk I/O is the bottleneck, not CPU.
- Goal: minimize the number of **disk reads** needed to find a record.
- A B+ Tree is a **self-balancing, disk-friendly tree** optimized for exactly this — high fanout keeps the tree short (low height), so few I/O operations are needed even for millions of rows.

---

## Core Structure

A B+ Tree has two kinds of nodes:

### 1. Internal (Index) Nodes

- Contain only **keys** and **child pointers** — used purely for routing/navigation.
- Do **not** store actual row data.
- Each node holds up to `n-1` keys and `n` pointers (n = "order" of the tree).

### 2. Leaf Nodes

- Contain the **actual data** or a **pointer to the row** (depends on index type — see below).
- All leaf nodes sit at the **same depth** → tree is always balanced.
- Leaf nodes are **linked together** (singly or doubly linked list) → enables fast **range scans** without traversing back up the tree.
![[Pasted image 20260829205526.png|617]]

```
                [ 30 | 60 ]
               /     |     \
        [10|20]   [30|40|50]  [60|70|80]
          |  |       |  |  |      |  |  |
        (leaf nodes, linked left→right)
```

> [!tip] Key idea Internal nodes = signposts. Leaf nodes = the actual destination + a highway connecting all destinations in order.

---

## B+ Tree vs B-Tree

||B-Tree|B+ Tree|
|---|---|---|
|Data storage|In internal **and** leaf nodes|Only in **leaf** nodes|
|Leaf nodes linked?|No|Yes (linked list)|
|Range queries|Slower (requires traversal/backtracking)|Fast (walk the linked leaves)|
|Fanout per node|Lower (space used by data too)|Higher (internal nodes = keys only)|
|Used in practice by DBs?|Rarely|**Almost universally** (MySQL InnoDB, PostgreSQL, Oracle, SQL Server)|

> **Fanout** = the number of children an internal B+ tree node can have
> With fanout 1,000, a B+ tree can cover **huge amounts of data with very few levels**.

> [!note] Why higher fanout matters Since internal nodes only store keys (no data), more keys fit per disk page → more children per node → shorter tree → fewer I/Os per lookup.

---

## Node Size & Disk Pages

- Each **node = one disk page** (or block).
- Node size chosen to match the OS/DB page size (e.g. 8KB in PostgreSQL, 16KB in InnoDB) → **one node = one I/O read**.
- **Order (fanout)** is determined by: `page_size / (key_size + pointer_size)`
- Higher fanout → shorter tree. Example: fanout of 100 → a tree of height 4 can index ~100 million rows.

---
## Operations

### Search
- Start at root, compare search key to keys in node, follow the correct child pointer.
- Repeat until reaching a leaf node.
- **Time complexity:** O(log_n N) — where n = fanout, N = total records.
- Very shallow tree (often 3–4 levels) → 3–4 disk reads even for huge tables.

### Range Query (`WHERE age BETWEEN 20 AND 40`)
- Traverse down to the first leaf matching the lower bound.
- Then just **walk the linked list of leaves** sequentially — no need to go back up the tree.
- This is the single biggest reason DBs prefer B+ over B-Trees.

### Insert
1. Search for the correct leaf.
2. Insert key in sorted order within the leaf.
3. If leaf **overflows** (exceeds max keys) → **split** into two leaves, push the middle key up to the parent.
4. If parent overflows too → split propagates upward (can reach the root → tree grows one level taller).

### Delete
1. Search for the leaf containing the key, remove it.
2. If leaf **underflows** (below minimum occupancy) →
    - **Borrow** a key from a sibling, or
    - **Merge** with a sibling.
3. Merges can propagate upward, potentially shrinking tree height.

> [!warning] Balance guarantee Splits/merges are what keep the tree balanced — height only grows/shrinks from the **root**, never becomes uneven across branches.

---
## How Databases Actually Use B+ Trees

### Clustered Index
- The leaf nodes **are** the actual table rows (data physically stored in key order).
- A table can have **only one** clustered index (data can only be sorted one way on disk).
- E.g., InnoDB (MySQL) always stores tables as a clustered index on the primary key.

### Secondary (Non-Clustered) Index
- Leaf nodes store the **indexed column value + a pointer/reference** to the actual row (either a row ID or the primary key value, depending on engine).
- A lookup via a secondary index = search secondary B+ Tree → get pointer → **second lookup** in the clustered index (called a "bookmark lookup" / "double lookup").
- A table can have **many** secondary indexes.

```
Secondary Index B+Tree          Clustered Index B+Tree (PK)
   [ leaf: name → PK ]   --->     [ leaf: PK → full row ]
```

### Why B+ Trees fit relational DBs so well
- **Sorted order maintained** → great for `ORDER BY`, range filters, `BETWEEN`, `<`, `>`.
- **Logarithmic search** → fast point lookups (`WHERE id = 5`).
- **Sequential leaf scanning** → efficient range scans and full index scans.
- **Balanced by design** → consistent performance regardless of insert order (no worst-case degeneration like a naive BST).
- **High fanout** → shallow tree despite huge data volume → few disk I/Os.

---
