## Table of Contents

- [[#1. Why a Framework? (Interface vs Implementation)]]
- [[#2. The Interface Hierarchy]]
- [[#3. The Collection Interface]]
- [[#4. List — ArrayList, LinkedList, Vector]]
- [[#5. Set — HashSet, LinkedHashSet, TreeSet]]
- [[#6. Queue & Deque — PriorityQueue, ArrayDeque]]
- [[#7. Other Map Implementations]]
- [[#8. Map Operations & Updating Entries]]
- [[#9. Collection Views]]
- [[#10. Algorithms (the `Collections` Utility Class)]]
- [[#11. Legacy Collections]]

---

## 1. Why a Framework? (Interface vs Implementation)

The Collections Framework's core design decision: **separate the interface (what a structure can do) from its implementation (how it does it).**

```java
interface Queue<E> {
    void add(E element);
    E remove();
    int size();
}
```

Two implementations satisfy the same contract very differently:

||Circular Array|Linked List|
|---|---|---|
|Storage|Fixed-capacity array, wraps around at the end|Chain of separately-allocated links|
|Capacity|Bounded|Unbounded|
|Efficiency|Generally faster|Slightly more overhead per element|

```java
Queue<Customer> expressLane = new CircularArrayQueue<>(100);  // concrete class ONLY at construction
expressLane.add(new Customer("Harry"));                          // interface type used everywhere else
```

> [!important] The one rule this produces **Declare variables by interface type, construct by concrete type.** Code written against `Queue<E>` never needs to change if you later swap `CircularArrayQueue` for a linked-list-backed one — this is the entire payoff of the separation.

---

## 2. The Interface Hierarchy

![[Pasted image 20260725021822.png]]
![[Pasted image 20260725020433.png]]

|Interface|Represents|
|---|---|
|`Collection`|Root interface for single-element containers|
|`List`|**Ordered**, index-based, allows duplicates|
|`Set`|No duplicates — `add()` rejects them|
|`SortedSet` / `NavigableSet`|Sorted set + comparator access / extra search & traversal methods (Java 6+)|
|`Queue` / `Deque`|FIFO access / double-ended access|
|`Map`|Key → value associations — **not** a `Collection` at all, has its own hierarchy|
|`SortedMap` / `NavigableMap`|Sorted map + comparator access / extra search & traversal methods|

`TreeSet`/`TreeMap` implement the `Navigable*` interfaces.

---

## 3. The Collection Interface

```java
public interface Collection<E> {
    boolean add(E element);      // returns true if the collection actually changed
    Iterator<E> iterator();
    // + many utility methods below
}
```

### Utility Methods Every Collection Provides

```java
int size();  boolean isEmpty();  boolean contains(Object o);
boolean containsAll(Collection<?> c);  boolean addAll(Collection<? extends E> c);
boolean remove(Object o);  boolean removeAll(Collection<?> c);  void clear();
boolean retainAll(Collection<?> c);  Object[] toArray();  <T> T[] toArray(T[] a);
default boolean removeIf(Predicate<? super E> filter);
```

`AbstractCollection` implements all of these in terms of just `size()` and `iterator()`, so a new concrete collection class only needs to supply those two — most of this predates default methods; only a few (`removeIf`, stream-related methods) were later added as `default` directly on `Collection`.

---

## 4. List — ArrayList, LinkedList, Vector

||`ArrayList`|`LinkedList`|`Vector`|
|---|---|---|---|
|Backing structure|Resizable array|Doubly-linked list|Resizable array|
|Random access (`get(i)`)|O(1) — fast|O(n) — must walk from an end|O(1) — fast|
|Insert/remove at start/middle|O(n) — shifts elements|O(1) once positioned (via `ListIterator`)|O(n)|
|Thread-safe?|❌|❌|✅ (synchronized methods)|
|Modern status|**Default choice**|Use when heavy middle insert/remove dominates|**Legacy** — see [[#11. Legacy Collections]]|

### Array Resizing & Capacity Growth

`ArrayList`/`Vector` store elements in an internal array. When `add()` would exceed the array's current capacity:

1. A **new, larger** internal array is allocated.
2. All existing elements are **copied** into it.
3. The old array is discarded.

```
capacity=4: [A][B][C][D]           add("E")
                 │
                 ▼
capacity=6 (grown): [A][B][C][D][E][ ]     ← new array, old one copied over
```

- `ArrayList` typically grows by **~50%** of current capacity when full; `Vector` traditionally **doubles**.
- This resize is an **amortized O(1)** operation per `add()` — most calls are cheap O(1) appends, with occasional O(n) copies, averaging out over many additions.
- Pre-sizing avoids the copies entirely when the eventual size is known:

```java
List<String> list = new ArrayList<>(1000);   // initial capacity — no growth needed until 1000 elements
```

### Vector vs ArrayList — Thread Safety

- `Vector`'s methods are `synchronized` — every `add()`/`get()`/etc. acquires a lock, making single-operation calls thread-safe.
- **This does NOT make compound operations safe** — `if (!vector.contains(x)) vector.add(x);` can still race between two threads, since the check and the act aren't atomic together.
- The synchronization overhead makes `Vector` **slower** than `ArrayList` even in single-threaded code — this is why it's considered legacy.
- **Modern preference for actual thread-safety**: `Collections.synchronizedList(new ArrayList<>())` (external locking, same caveat as `Vector`) or `CopyOnWriteArrayList` (fail-safe iteration, good for read-heavy/write-rare scenarios).

### LinkedList — Position-Dependent Insertion via ListIterator

Since `Iterator` has no `add()` method (position-dependent insertion only makes sense for **ordered** collections), `ListIterator` supplies it:

```java
List<String> numbers = new LinkedList<>();
numbers.add("First"); 
numbers.add("Second"); 
numbers.add("Third");

ListIterator<String> iter = numbers.listIterator();
iter.next();                     // skip past "First"
iter.add("Before Second");       // inserted between "First" and "Second"
```

### Linked List Efficiency Summary

| Operation                              | Efficiency                                       |
| -------------------------------------- | ------------------------------------------------ |
| Insert/remove a node (once positioned) | ✅ Efficient — only neighboring links are updated |
| Visiting all elements in order         | ✅ Efficient                                      |
| Random access (`get(index)`)           | ❌ Inefficient — must traverse from an end        |

### Concurrent Modification (Recap)

Two independent iterators over the same mutable list — one writing, one reading — trigger `ConcurrentModificationException` on the reader, since the writer invalidates the reader's position. 

---

## 5. Set — HashSet, LinkedHashSet, TreeSet

||`HashSet`|`LinkedHashSet`|`TreeSet`|
|---|---|---|---|
|Ordering|None guaranteed|**Insertion order** preserved|**Sorted** (natural or via `Comparator`)|
|Backing structure|Hash table|Hash table + linked list (tracks insertion order)|Red-black tree|
|`add`/`contains`|O(1) average|O(1) average|O(log n)|
|Extra capability|—|—|`NavigableSet` methods (`floor`, `ceiling`, `first`, `last`, range views)|

- `Set.add()` **rejects duplicates** — this is the one behavioral difference from plain `Collection`; everything else about the interface is identical.
- `EnumSet` — a specialized, highly efficient `Set` implementation restricted to values of a single `enum` type, internally backed by a bitvector.

---

## 6. Queue & Deque — PriorityQueue, ArrayDeque

||`PriorityQueue`|`ArrayDeque`|
|---|---|---|
|Ordering|Smallest element (by natural order/`Comparator`) always removed first|FIFO/LIFO — double-ended|
|Backing structure|Binary heap|Resizable circular array|
|Use case|Efficiently repeatedly remove the "smallest"/"highest priority" item|Stack **or** queue — more efficient than legacy `Stack`/`LinkedList` for either role|

```java
Queue<Integer> pq = new PriorityQueue<>();
pq.add(5); pq.add(1); pq.add(3);
pq.poll();   // 1 — smallest first, regardless of insertion order
```

```java
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1); stack.push(2);
stack.pop();     // 2 — LIFO

Deque<Integer> queue = new ArrayDeque<>();
queue.offer(1); queue.offer(2);
queue.poll();     // 1 — FIFO
```

---

## 7. Other Map Implementations

| Map                 | Ordering / Behavior                                                                                                                   |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `HashMap`           | No guaranteed order                                                                                                                   |
| `LinkedHashMap`     | Remembers **insertion order** (or optionally access order, for LRU-style caches)                                                      |
| `TreeMap`           | Keys kept **sorted**; implements `NavigableMap`                                                                                       |
| `EnumMap`           | Keys restricted to one `enum` type; very efficient array-backed implementation                                                        |
| `ConcurrentHashMap` | Thread-safe, high-concurrency; **fail-safe/weakly-consistent** iterators                                                              |
| `WeakHashMap`       | Values become eligible for garbage collection once no strong reference exists elsewhere — useful for caches that shouldn't prevent GC |
| `IdentityHashMap`   | Compares keys with `==`, not `.equals()` — rare, niche use cases (e.g. object-graph traversal bookkeeping)                            |

---

## 8. Map Operations & Updating Entries

### Core Methods

```java
V get(Object key);
default V getOrDefault(Object key, V defaultValue);
V put(K key, V value);              // returns the PREVIOUS value, or null
void putAll(Map<? extends K, ? extends V> entries);
boolean containsKey(Object key);
boolean containsValue(Object value);
V remove(Object key);
default void forEach(BiConsumer<? super K, ? super V> action);
```

```java
staff.forEach((k, v) -> System.out.println("key=" + k + ", value=" + v));
```

### The "First Occurrence" Problem

Naive counting logic:

```java
counts.put(word, counts.get(word) + 1);   // NullPointerException the first time `word` appears
```

Three progressively better fixes:

```java
counts.put(word, counts.getOrDefault(word, 0) + 1);  // OK — supplies a fallback

counts.putIfAbsent(word, 0);              // OK — ensures a value exists first
counts.put(word, counts.get(word) + 1);

counts.merge(word, 1, Integer::sum);  // BEST — one call, no null-handling needed
```

`merge(key, value, remappingFunction)`: if the key is absent, associates it with `value` directly; if present, combines the existing value and `value` via the given function.

---

## 9. Collection Views

A **collection view** in Java is ==a lightweight, alternative interface to an existing collection or data structure that does not store its own copy of the elements==. Instead, it **directly references the backing collection**, meaning any modifications made to the view write through to the original collection, and vice versa.
### Map Views — the Three Ways to See a Map as a Collection

`Map` deliberately does **not** extend `Collection` — but you can obtain three collection-like views over it:

```java
Set<K> keySet();                     // view of just the keys
Collection<V> values();               // view of just the values
Set<Map.Entry<K, V>> entrySet();     // view of key/value pairs together
```

```java
for (Map.Entry<String, Employee> entry : staff.entrySet()) {
    System.out.println(entry.getKey() + " -> " + entry.getValue());
}
```

> [!tip] `Map.Entry` is itself a nested interface inside `Map` — the `entrySet()` view is where you'll actually encounter it in practice.

### `subList` — a List View

```java
List<Integer> full = new ArrayList<>(List.of(1, 2, 3, 4, 5));
List<Integer> view = full.subList(1, 4);   // view over indices 1..3 — [2, 3, 4]

view.set(0, 99);
System.out.println(full);                    // [1, 99, 3, 4, 5] — the BACKING list changed too
```

### Unmodifiable Views

```java
List<String> readOnly = Collections.unmodifiableList(original);
```

Wraps the original — reads pass through, but any mutating call (`add`, `remove`, `set`) throws `UnsupportedOperationException`. **Important**: as covered in **the immutability checklist**, this does **not** protect against changes made directly through the _original_ reference — it's a view, not a copy.

### Synchronized Views

```java
List<String> threadSafe = Collections.synchronizedList(new ArrayList<>());
```

Wraps every method with synchronization on a common lock — same "individual calls are safe, compound operations aren't" caveat as `Vector` (see [[#4. List — ArrayList, LinkedList, Vector]]); manual `synchronized` blocks are still needed around multi-step operations like iteration.

### Why Views Matter

Views let you expose a **restricted or synchronized perspective** on your actual data structure without copying it — directly the same principle behind defensive copying, just the opposite tool: sometimes you deliberately _want_ a live, linked view rather than an independent copy.

---

## 10. Algorithms (the `Collections` Utility Class)

### Why Generic Algorithms Matter

Writing a "find the max" loop separately for arrays, `ArrayList`, and `LinkedList` is repetitive and error-prone (off-by-one bugs, empty-collection edge cases). Since finding a max only requires **iteration** — no random access — it can be written **once**, generically, against the `Collection` interface:

```java
public static <T extends Comparable<T>> T max(Collection<T> c) {
    if (c.isEmpty()) throw new NoSuchElementException();
    Iterator<T> iter = c.iterator();
    T largest = iter.next();
    while (iter.hasNext()) {
        T next = iter.next();
        if (largest.compareTo(next) < 0) largest = next;
    }
    return largest;
}
```

This single method now works identically on an array-backed list, a linked list, or any future `Collection` implementation.

### Sorting

```java
Collections.sort(staff);                                            // needs Comparable elements
staff.sort(Comparator.comparingDouble(Employee::getSalary));          // custom order
staff.sort(Comparator.reverseOrder());                                 // descending, natural order
staff.sort(Comparator.comparingDouble(Employee::getSalary).reversed()); // descending, custom order
```

### Shuffling & Selecting

```java
List<Integer> numbers = IntStream.rangeClosed(1, 49).boxed().collect(Collectors.toList());
Collections.shuffle(numbers);
List<Integer> winningCombination = numbers.subList(0, 6);   // a VIEW — see section 10
Collections.sort(winningCombination);
```

### Binary Search

Requires a **pre-sorted** collection — an unsorted input silently produces a wrong answer, since the algorithm assumes it can discard half the remaining range on each comparison.

```java
int i = Collections.binarySearch(c, element);
int i = Collections.binarySearch(c, element, comparator);   // when not naturally Comparable
```

||Linear Search|Binary Search|
|---|---|---|
|1024 elements, average case|~512 steps|~10 steps|
|Requires sorted input?|No|**Yes**|

### Other `Collections` Static Methods

```java
static <T> T min(Collection<T> c, Comparator<? super T> cmp);
static <T> T max(Collection<T> c, Comparator<? super T> cmp);
static <T> void copy(List<? super T> to, List<T> from);
static <T> void fill(List<? super T> l, T value);
static void swap(List<?> l, int i, int j);
static void reverse(List<?> l);
static void rotate(List<?> l, int distance);   // element at index i moves to (i + distance) mod size
static int frequency(Collection<?> c, Object o);
```

### `removeIf` / `replaceAll` (Java 8+, on `Collection`/`List` directly)

```java
words.removeIf(w -> w.length() <= 3);
words.replaceAll(String::toLowerCase);
```

---

## 11. Legacy Collections

Present since Java's first release, **before** the Collections Framework existed (Java 1.2) — later retrofitted to implement the modern interfaces so they interoperate with everything else:

| Legacy Class  | Modern Replacement                                                         |
| ------------- | -------------------------------------------------------------------------- |
| `Vector`      | `ArrayList` (or `CopyOnWriteArrayList` if genuine thread safety is needed) |
| `Stack`       | `ArrayDeque` (as a stack, via `push`/`pop`)                                |
| `Hashtable`   | `HashMap` (or `ConcurrentHashMap` for thread safety)                       |
| `Enumeration` | `Iterator`                                                                 |

They still work and still implement `List`/`Map`/etc., but offer no advantage over their modern counterparts today — mainly encountered when maintaining older codebases.

---
## Extra Notes

|Feature|EnumSet|HashSet|
|---|---|---|
|Only enums|✅|❌|
|Internal structure|Bit vector|Hash table|
|Faster|✅|Good|
|Less memory|✅|More|
|Allows null|❌|One null allowed|

|Feature|EnumMap|HashMap|
|---|---|---|
|Keys|Enum only|Any object|
|Internal structure|Array|Hash table|
|Ordered|Natural enum order|No guarantee|
|Faster|Usually|General-purpose|
|Memory|Less|More|

| Class/Interface | Type                 | Purpose                                                       | Typical Use                 |
| --------------- | -------------------- | ------------------------------------------------------------- | --------------------------- |
| `Enumeration`   | Legacy interface     | Iterate over legacy collections                               | `Vector`, `Hashtable`       |
| `WeakHashMap`   | `Map` implementation | Entries disappear when keys are no longer strongly referenced | Caches, metadata            |
| `EnumSet`       | `Set` implementation | High-performance set for enum constants                       | Sets of flags or states     |
| `EnumMap`       | `Map` implementation | High-performance map with enum keys                           | Mapping enum values to data |
