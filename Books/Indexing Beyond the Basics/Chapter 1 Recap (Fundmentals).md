## 1.1 A Different (Simplified) View on B+ Trees

- Indexes are fast **because of the B+ tree data structure** underneath them.
- You don't need to memorize internal B+ tree mechanics (splitting, rebalancing, etc.) to use indexes well — the DB handles that automatically.
- Simplified mental model, a B+ tree gives you:
    - A **sorted list of values** (leaf nodes) = essentially "the table sorted by indexed column(s)".
    - **Multiple levels of index summaries** (internal nodes) sitting above it, like a table of contents, letting you skip straight to the right range instead of scanning.
- Even with millions of rows, only a handful of steps are needed to reach the right leaf entries.

### Automatic Index Maintenance

The database keeps every index in sync with the table automatically:

| Table event                          | Index effect                        |
| ------------------------------------ | ----------------------------------- |
| Row **added**                        | New index entry created             |
| Row **removed**                      | Index entry removed                 |
| Row **changed** (indexed column)     | Old entry removed + new entry added |
| Row **changed** (non-indexed column) | No index change needed              |

> [!warning] Tradeoff Every index = extra write work on every insert/update/delete. More indexes → slower writes, faster reads. In practice this is rarely a real concern since most workloads are read-heavy and tables have relatively few indexes.

### Sorted vs Unsorted Insertion

- B+ trees can insert anywhere in the sorted order, but **appending to the end is always fastest**.
- This is _why_ people say random UUIDs make bad keys vs sequential ones (UUIDv7, auto-increment ints) — random values force insertion at random positions in the tree.
- ⚠️ Impact **depends on the database**. Matters a lot for **MySQL** (see clustered index section below). Only worth worrying about at large scale (millions/billions of rows, high insert rate).

---
## 1.2 The Interaction of Indexes and Tables

### Table Scan (default, no index)

- DB reads **every row**, checks condition, keeps matches, discards rest.
- Gets slower linearly as table grows.

### How an Index Query Actually Works

1. DB finds all **matching index entries** for the indexed conditions.
2. DB then **loads each referenced row** from the table one by one.
3. Any condition **not covered by the index** is evaluated only after the row is loaded.

> [!danger] Common trap — "I have an index but it's still slow" If a query has a condition on a **non-indexed column** that filters out most rows, the DB still had to load _all_ rows matched by the index first, only to discard most of them afterward. Fix: build a better index that also covers that filtering column.

### Table Storage Models

Two totally different ways DBs physically store table data:

#### 🔹 Heap Tables (e.g. **PostgreSQL** always, SQL Server/Oracle optionally)

- New rows just get **appended to the end** of the table — fast, regardless of PK order.
- Index entries store a **pointer to the row's physical location**.
- No distinction between PK / index / unique index — they're all just pointers to physical location.

![[Pasted image 20260830233747.png|510]]

#### 🔹 Clustered Index (e.g. **MySQL/InnoDB** always, SQL Server/Oracle optionally)

- Table **IS** the primary key index — no separate table storage. Row data lives directly in the PK's leaf nodes.
- Secondary indexes don't point to a physical location — they store a **copy of the primary key** to look the row up.
- **PK lookups are very fast**: finding the index entry = you already have the full row (no second fetch).
- Secondary index lookups: no real difference from heap tables — still need a second lookup step.

![[Pasted image 20260830233758.png|516]]

==Secondary Index Example== `SELECT * FROM users WHERE name = 'Carol'`
requires **two B-tree traversals**:
```
        Secondary Index
              │
              ▼
       name = "Carol"
              │
              ▼
         PK = 30
              │
              │  second lookup
              ▼
     Clustered PK Index
              │
              ▼
    ┌─────────────────┐
    │ 30 | Carol | 28 │
    └─────────────────┘
          actual row
```


> [!danger] Clustered Index Pitfalls
> 
> 1. **Don't use random PKs** (e.g. UUIDv4) with clustered-index databases like MySQL — every insert lands at a random position → massive slowdown vs. sequential ints or UUIDv7.
> 2. **PK size matters more than you think** — the PK is duplicated into _every_ secondary index. A large PK type (e.g. 26-byte ULID) multiplied across several indexes × millions of rows = significant storage bloat (tens/hundreds of MB extra).
> 3. Default safe choice: **incrementing integer PK**.

---
