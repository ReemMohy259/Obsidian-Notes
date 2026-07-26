## Table of contents

- 1- [[#Iterators in Java]]
- 2- [[#Iterator Vs List Iterator]]
- 3- [[#Fail-Safe Iterator vs Fail-Fast Iterator]]

---

# Iterators in Java

An **Iterator** traverses a `Collection` without exposing its internal structure — obtained via `iterable.iterator()`.

```java
Iterator<Integer> it = numbers.iterator();
while (it.hasNext()) {
    Integer n = it.next();
    if (n == 3) it.remove();   // safe removal during iteration
}
```

|Method|Purpose|
|---|---|
|`hasNext()`|More elements left?|
|`next()`|Return next element, advance cursor|
|`remove()`|Remove the last element `next()` returned — safe, no `ConcurrentModificationException`|
|`forEachRemaining(action)`|Java 8+ — process all remaining with a lambda|

- Forward-only, no guaranteed order.
- `remove()` is a `default` method — throws `UnsupportedOperationException` on immutable collections.

---

# Iterator Vs List Iterator

||`Iterator`|`ListIterator`|
|---|---|---|
|Works on|Any `Collection`|Only `List`|
|Direction|Forward only|Forward **and** backward (`hasPrevious()`/`previous()`)|
|Extra methods|—|`add()`, `set()`, `nextIndex()`, `previousIndex()`|
|Start position|Beginning only|Anywhere — `listIterator(int index)`|
|Relationship|Base interface|`ListIterator extends Iterator`|

```java
ListIterator<String> lit = list.listIterator();
lit.next(); lit.next();
lit.set("X");        // replaces last next()/previous() result
lit.add("Y");         // inserts before the cursor
```

> [!tip] Cursor model The cursor sits **between** elements (`n` elements → `n+1` positions) — alternating `next()`/`previous()` returns the **same** element each time.

---

# Fail-Safe Iterator vs Fail-Fast Iterator

## Core Idea

- **Fail-Fast**: aborts immediately on failure, exposing it right away.
- **Fail-Safe**: avoids aborting — tries to keep going despite a concurrent change.

## Fail-Fast Iterators

- Default for `java.util` collections (`ArrayList`, `HashMap`, etc.).
- Collections track a `modCount` (incremented on structural add/remove). Each `next()` compares current `modCount` to the value captured at iterator creation — mismatch → `ConcurrentModificationException`.
- Thrown on a **best-effort basis only** — not guaranteed in every concurrent scenario.

```java
Iterator<Integer> it = numbers.iterator();
while (it.hasNext()) {
    Integer n = it.next();
    numbers.add(50);       // structural change via the COLLECTION → CME on next next()
}
```

> [!important] The one safe exception Removing via the **iterator's own** `remove()` is fine — it updates `modCount` in sync, so no exception:
> 
> ```java
> if (it.next() == 30) it.remove();   // OK
> numbers.remove(2);                    // NOT OK — throws CME
> ```

## Fail-Safe Iterators

- Default for `java.util.concurrent` collections (`ConcurrentHashMap`, `CopyOnWriteArrayList`, etc.).
- Iterate over a **clone/snapshot** (or an internally synchronized view) instead of the live structure — modifications during iteration don't throw.
- **Not truly "fail-safe"** — the accurate term is **weakly consistent**: what the iterator sees during a concurrent modification is only weakly guaranteed, and varies by collection (documented per-class in the Javadoc).

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.put("A", 1); map.put("B", 2);

Iterator<String> it = map.keySet().iterator();
while (it.hasNext()) {
    String key = it.next();
    map.put("C", 3);       // no exception — may or may not be seen by this iteration
}
```

### Trade-offs

||Fail-Fast|Fail-Safe|
|---|---|---|
|On concurrent modification|Throws `ConcurrentModificationException`|Continues, no exception|
|Data seen|Always current (or fails)|May be stale — works off a snapshot/weakly consistent view|
|Overhead|None extra|Cloning/synchronization cost (time + memory)|
|Example collections|`ArrayList`, `HashMap`, `HashSet`|`ConcurrentHashMap`, `CopyOnWriteArrayList`|
