## Overview

Four core principles describe **every way** an index can be used by a query. Goal: build a mental model to test _"can this query use this index?"_ — this is the core skill for designing good indexes.

1. Fast Lookup
2. Scan in One Direction
3. From Left to Right (multi-column indexes)
4. Scan on Range Conditions

---
## Principle 1: Fast Lookup

- For an equality condition (`WHERE release_year = 2019`), the DB jumps almost directly to the matching offset in the sorted index — using the index summary (internal nodes) — **not** a linear scan.
- Myth: "bigger index = slower query." **False.** Indexes are built exactly to make lookups over huge datasets just as fast as small ones (logarithmic, not linear).

---
## Principle 2: Scan in One Direction

- After a fast lookup finds a starting offset, the DB can **continue scanning** the sorted list — either **ascending or descending**.
- Example: `WHERE age >= 35 ORDER BY age ASC LIMIT 3`
    1. Fast lookup to first entry where `age >= 35`
    2. Scan ascending
    3. Stop after 3 matches
- ⚠️ You can scan **one direction only per query** — never both ascending and descending simultaneously.

---
## Principle 3: From Left to Right (Without Skipping a Column)

Core rule for **multi-column (composite) indexes**.

- Index entries are sorted by column 1 first, then column 2 (for ties), then column 3, etc.
- Think of it as a **funnel**: each column narrows down the range further, but only if you enter the funnel from the **first** column and don't skip any.

**Index on `(firstname, lastname, country)` supports:**

```
WHERE firstname = 'James'
WHERE firstname = 'James' AND lastname = 'Walker'
WHERE firstname = 'James' AND lastname = 'Walker' AND country = 'US'
```

### Myth: "Order by selectivity"

- Common but **wrong** advice: put the most selective (most unique values) column first.
- Reordering columns doesn't reduce the number of funnel steps for a query using _all_ columns.
- **The real rule:** order columns so the index covers the **most common query patterns** across your app — not just one query.
- ⚠️ If most queries filter on `country` alone, `country` **must be first**, or the index is useless for those queries (can't enter the funnel mid-way).

### Skipping a Column

Condition `firstname = 'James' AND country = 'US'` on index `(firstname, lastname, country)`:

- Can't skip `lastname` → can't use full funnel.
- DB instead:
    1. Fast lookup on `firstname = 'James'` (1st column only)
    2. Scan ascending through all "James" entries
    3. Filter each entry against `country = 'US'` — discard non-matches
- **Still better than no multi-column index** (filtering happens inside the index, so only matching rows get loaded from the table) — but **worse than a proper index** like `(firstname, country, lastname)` or `(firstname, country)`, which would allow a clean fast lookup instead of scan-and-filter.

> [!tip] Golden rule **"From left to right, without skipping a column."** First part = mandatory. Second part = optional but needed for real performance.

### Overlapping / Redundant Indexes

- If you have `(country, lastname, firstname)`, a separate single-column index on `(country)` alone is **redundant** — drop it.
- Rule: index B is redundant if it's a **strict prefix subset** of index A with the same column order.
- `(country, lastname)` and `(country, lastname, telephone)` — NOT redundant vs `(country, lastname, email)`, since they diverge after the shared prefix.

---
## Principle 4: Scan on Range Conditions

- Range conditions (`< <= > >=`) behave differently from equality (`=`).
- Once a **range condition starts a scan**, the DB **cannot skip entries anymore** — every entry within that range must be scanned, even if a later funnel column could have filtered further.

**Example:** Index `(country, age, married)`, query:

```sql
WHERE country = 'US' AND married = 'yes' AND age > 28
```

- ❌ Naive expectation: fast lookup + skip via `married` column too.
- ✅ Reality: fast lookup on `country`, then **scan all entries** where `age > 28` (can't narrow further using the funnel), filtering `married` per-entry (used only to decide which _rows_ to load, not to reduce _scanned index entries_).

> [!important] Ordering rule Put **equality-check columns before range-condition columns** in a multi-column index. Range conditions should come **last**, since anything after a range column loses fast-lookup/funnel benefits. ✅ Better index for above query: `(country, married, age)`

### Exception: Loose Index Scan / Skip Scan

- Some DBs _can_ skip entries for specific `GROUP BY` queries needing min/max per group.
- MySQL: **"Loose Index Scan"**. SQL Server / Oracle: **"Skip Scan"**.
- **Not available in PostgreSQL.**

---

