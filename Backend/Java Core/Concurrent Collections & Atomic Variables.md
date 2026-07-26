## Table of Contents

- [[#1. Why Not Just Synchronize Everything?]]
- [[#2. Two Strategies: Fine-Grained Locking vs Lock-Free CAS]]
- [[#3. Concurrent Collections — Classification]]
- [[#4. Concurrent Maps]]
- [[#5. Concurrent Lists & Sets (Copy-on-Write)]]
- [[#6. Non-Blocking Queues & Deques]]
- [[#7. Blocking Queues — Producer/Consumer]]
- [[#8. Choosing the Right Collection]]
- [[#9. Atomic Variables]]
- [[#10. Compare-And-Swap (CAS) — The Mechanism Behind Atomics]]
- [[#11. Atomic Variable Classes & Methods]]
- [[#12. Locks vs Atomics vs Concurrent Collections]]

---

## 1. Why Not Just Synchronize Everything?

The legacy approach — `Hashtable`, `Vector`, or `Collections.synchronizedXxx(...)` (see [[Java-Collections-Framework-Notes#4. List — ArrayList, LinkedList, Vector|Vector vs ArrayList]]) — wraps **every single method** in a lock on one shared monitor. This works, but doesn't scale:

```
Thread A ──┐
Thread B ──┼──► ONE lock guarding the WHOLE structure ──► only ONE thread proceeds at a time
Thread C ──┘        (everyone else blocks, even for unrelated keys/indices)
```

- Every operation — read or write, regardless of which part of the structure it touches — competes for the **same single lock**.
- Under real concurrency, threads spend more time **blocked and context-switching** than doing actual work.
- Iterating one of these while another thread mutates it also throws `ConcurrentModificationException` — [[Java-Iterators-FailFast-FailSafe-Notes#Fail-Fast Iterators|fail-fast]], exactly like the plain `ArrayList`/`HashMap` case.

**`java.util.concurrent`** collections exist specifically to fix both problems at once.

---

## 2. Two Strategies: Fine-Grained Locking vs Lock-Free CAS

Concurrent collections use one of two fundamentally different approaches instead of a single global lock:

| Strategy                                 | Idea                                                                                                                                         | Used By                                                                                                                                     |
| ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| **Fine-grained locking / lock striping** | Split the structure into many independent segments, each with its **own** lock — threads touching different segments never contend at all    | `ConcurrentHashMap` (per-bucket-region locking)                                                                                             |
| **Lock-free CAS** (Compare-And-Swap)     | No locks at all — threads race to update shared state using an atomic hardware instruction; a losing thread just retries instead of blocking | `ConcurrentLinkedQueue`, `ConcurrentHashMap`'s reads, atomic variables (see [[#10. Compare-And-Swap (CAS) — The Mechanism Behind Atomics]]) |
| **Copy-on-write**                        | Every mutation copies the **entire** underlying array; readers always see a stable, unchanging snapshot                                      | `CopyOnWriteArrayList`, `CopyOnWriteArraySet`                                                                                               |

### The Other Shared Benefit: Fail-Safe Iteration

Every collection in this package provides **fail-safe / weakly-consistent** iterators — no `ConcurrentModificationException`, even while the collection is being modified concurrently — directly the [[Java-Iterators-FailFast-FailSafe-Notes#Fail-Safe Iterators|Fail-Safe Iterators]] behavior, just applied consistently across this whole package rather than a one-off exception.

---

## 3. Concurrent Collections — Classification

|Collection|Ordering|Blocking|Copy-on-Write|Sorted|
|---|---|---|---|---|
|`CopyOnWriteArrayList`|✅ Insertion order|❌|✅|❌|
|`CopyOnWriteArraySet`|❌|❌|✅|❌|
|`ConcurrentHashMap`|❌|❌|❌|❌|
|`ConcurrentSkipListMap`|✅|❌|❌|✅|
|`ConcurrentSkipListSet`|✅|❌|❌|✅|
|`ConcurrentLinkedQueue`|FIFO|❌|❌|❌|
|`ConcurrentLinkedDeque`|Double-ended|❌|❌|❌|
|`ArrayBlockingQueue`|FIFO|✅|❌|❌|
|`LinkedBlockingQueue`|FIFO|✅|❌|❌|
|`LinkedBlockingDeque`|Double-ended|✅|❌|❌|
|`PriorityBlockingQueue`|Priority|✅|❌|✅|
|`DelayQueue`|Delayed|✅|❌|Time-based|
|`SynchronousQueue`|Direct handoff|✅|❌|❌|
|`LinkedTransferQueue`|FIFO|✅|❌|❌|

---

## 4. Concurrent Maps

### `ConcurrentHashMap` — the `HashMap` Replacement

- Uses **CAS operations** plus **localized, bucket-region-level locking** — multiple threads can write to _different_ regions simultaneously, and reads are largely **non-blocking**.
- **Does not accept `null` keys or values** — unlike `HashMap`, which allows one `null` key. This is deliberate: in a concurrent context, `map.get(key) == null` is ambiguous (does it mean "absent" or "present with a null value?") in a way that's tolerable in single-threaded code but dangerous once other threads can be racing to insert.
- Directly builds on the [[Java-Collections-Framework-Notes#7. Map — HashMap Internals|HashMap internals]] you already know (buckets, hashing, treeification) — the concurrency comes from _how_ access to those buckets is coordinated, not from a different data layout.

```java
ConcurrentHashMap<String, Integer> scores = new ConcurrentHashMap<>();
scores.put("Alice", 10);
scores.computeIfPresent("Alice", (k, v) -> v + 5);   // atomic read-modify-write for that key
```

### `ConcurrentSkipListMap` — the `TreeMap` Replacement

- Thread-safe, **sorted** map, backed by a **skip list** (a probabilistic, layered linked-list structure) rather than the red-black tree `TreeMap` uses — because rebalancing a tree under concurrent modification is much harder to do without heavy locking, while a skip list supports lock-free/CAS-based concurrent insertion naturally.
- Guarantees expected **O(log n)** for `get`/`put`/`remove`, same asymptotic complexity as `TreeMap`, just concurrency-safe.

---

## 5. Concurrent Lists & Sets (Copy-on-Write)

### `CopyOnWriteArrayList`

- Every **mutating** call (`add`, `set`, `remove`) allocates a **brand-new underlying array**, copies existing elements into it, applies the change, and swaps the reference — readers already iterating hold a reference to the **old** array and are completely unaffected.

```
Write happens:
old array [A, B, C]  ──copy──►  new array [A, B, C, D]
     ▲                                  ▲
     readers already iterating    future reads see the new array
     keep reading THIS one safely
```

- **Ideal for read-heavy, write-rare workloads** — e.g. a list of event listeners that's read constantly but modified only occasionally. Every write is expensive (full array copy); every read is essentially free and never blocks.
- Iterators are inherently fail-safe here — they're iterating a frozen snapshot by construction, not merely "weakly consistent" by convention.

### `CopyOnWriteArraySet`

- A `Set` built **internally on top of `CopyOnWriteArrayList`** — same copy-on-write mechanics, plus the standard duplicate-rejection behavior of any `Set`.

### `ConcurrentSkipListSet`

- The sorted-set counterpart, internally backed by a `ConcurrentSkipListMap` — same relationship as `TreeSet`/`TreeMap` in the non-concurrent world (recap: [[Java-Collections-Framework-Notes#5. Set — HashSet, LinkedHashSet, TreeSet|Set implementations]]).

---

## 6. Non-Blocking Queues & Deques

### `ConcurrentLinkedQueue` / `ConcurrentLinkedDeque`

- **Unbounded**, lock-free, linked-node-based FIFO queue / double-ended queue.
- Uses CAS internally to update head/tail pointers — under contention, losing threads simply **retry** rather than block (see [[#10. Compare-And-Swap (CAS) — The Mechanism Behind Atomics]]).
- No blocking behavior at all: if you try to `poll()` an empty queue, you just get `null` back immediately — contrast with the blocking queues below, which would make the calling thread wait.

---

## 7. Blocking Queues — Producer/Consumer

These implement `BlockingQueue`/`BlockingDeque`: a producer thread adding to a **full** bounded queue **blocks** until space frees up; a consumer thread taking from an **empty** queue blocks until something arrives. This is the standard building block for the classic **producer-consumer** pattern.

```
Producer thread ──put()──► [ Queue: bounded capacity ] ──take()──► Consumer thread
                     blocks if FULL                      blocks if EMPTY
```

|Queue|Bounded?|Ordering|Notes|
|---|---|---|---|
|`ArrayBlockingQueue`|✅ Fixed capacity|FIFO|Backed by a plain array|
|`LinkedBlockingQueue`|Optionally bounded|FIFO|Linked-node based, generally higher throughput than the array-backed version|
|`LinkedBlockingDeque`|Optionally bounded|Double-ended|Blocking operations at **both** ends|
|`PriorityBlockingQueue`|Unbounded|By priority, not FIFO|Blocking version of [[Java-Collections-Framework-Notes#6. Queue & Deque — PriorityQueue, ArrayDeque|
|`DelayQueue`|Unbounded|By expiry time|Elements only become extractable once their individual delay has elapsed — useful for scheduling/retry-after logic|
|`SynchronousQueue`|Zero capacity|Direct handoff|Every `put()` must rendezvous with a matching `take()` on another thread — there's no actual storage, just a synchronization point|
|`LinkedTransferQueue`|Unbounded|FIFO|Implements `TransferQueue` — adds `transfer()`, letting a producer block until a consumer explicitly receives that specific item|

---

## 8. Choosing the Right Collection

|Concurrent Collection|Standard Equivalent|Primary Thread-Safety Mechanism|Best Use Case|
|---|---|---|---|
|`ConcurrentHashMap`|`HashMap`|CAS + bucket-region locks|High-throughput key-value caching|
|`CopyOnWriteArrayList`|`ArrayList`|Copy entire array on write|Read-heavy, write-rare (listener lists, config snapshots)|
|`ConcurrentSkipListMap`|`TreeMap`|Skip list + CAS|Sorted access under high thread counts|
|`LinkedBlockingQueue`|`LinkedList`/`Queue`|Separate put/take locks|Producer-consumer pipelines|

> [!tip] Decision checklist Ask: **(1)** key-value, list, or unique-set data? **(2)** read-to-write ratio — copy-on-write only pays off if reads dominate. **(3)** do you need sorted order, or strict FIFO/priority/delay semantics? Those three answers narrow the table above to essentially one row.

---

## 9. Atomic Variables

**Atomic variables** solve a narrower problem than concurrent collections: making **a single value** (not a whole collection) safely and efficiently updatable across threads, without locks.

### The Problem: `counter++` Isn't Actually One Operation

```java
public class Counter {
    int counter;
    public void increment() { counter++; }   // looks atomic — ISN'T
}
```

`counter++` is really **three** separate steps: **read** the current value, **increment** it, **write** it back. If two threads interleave those steps, one thread's update can be silently **lost**:

```
Thread A reads counter = 5
Thread B reads counter = 5
Thread A computes 6, writes counter = 6
Thread B computes 6, writes counter = 6      ← B's write LOSES one increment; should be 7
```

### The Traditional Fix: Locks

```java
public class SafeCounterWithLock {
    private int counter;
    public synchronized void increment() { counter++; }
}
```

- `synchronized` forces only **one thread at a time** through the method — correct, but every losing thread gets **blocked and suspended**, and later **resumed** — an expensive context-switch, especially costly relative to how little work `counter++` actually does. For something this small, lock overhead can dwarf the actual computation.

### The Modern Fix: Lock-Free Atomics

Instead of blocking, atomic variables use a hardware-level **Compare-And-Swap** instruction — covered next — so contending threads never actually block the OS scheduler; a "losing" thread just finds out immediately and retries.

---

## 10. Compare-And-Swap (CAS) — The Mechanism Behind Atomics

A CAS operation takes **three operands**:

- **M** — the memory location to update
- **A** — the value the caller _expects_ to currently be there
- **B** — the new value to set

**Behavior**: atomically, in one CPU instruction, if the current value at `M` equals `A`, set it to `B`; otherwise, do nothing. Either way, return the value that was actually in `M` at that instant.

```
CAS(M, A, B):
  if M == A:
      M = B
      return A        (success)
  else:
      return current M value    (failure — someone else already changed it)
```

### Why This Avoids Blocking

```
Thread A: CAS(counter, 5, 6)  → succeeds, counter is now 6
Thread B: CAS(counter, 5, 6)  → FAILS (counter is no longer 5) → B is simply told "no", not suspended
Thread B: reads counter = 6, retries CAS(counter, 6, 7)  → succeeds
```

- The "losing" thread is never suspended by the OS — it's just told immediately that the update didn't apply, and can **retry** with the new current value (typically in a tight loop inside the atomic class's own method).
- This eliminates the **context-switch cost** that made locks expensive for small, frequent operations — at the cost of slightly more complex logic internally (the retry loop), which the JDK's atomic classes handle for you.

> [!tip] CAS is a hardware instruction This isn't a Java-level trick — CAS corresponds to real CPU instructions (`CMPXCHG` on x86, load-link/store-conditional on ARM). That's _why_ it can combine "get, compare, and set" into one indivisible step with no possibility of another thread interleaving in the middle.

---

## 11. Atomic Variable Classes & Methods

### The Core Classes

|Class|Wraps|
|---|---|
|`AtomicInteger`|`int`|
|`AtomicLong`|`long`|
|`AtomicBoolean`|`boolean`|
|`AtomicReference<V>`|Any object reference|

### Key Methods (shared shape across all of them)

|Method|Behavior|
|---|---|
|`get()`|Reads the current value — visible across threads, equivalent to reading a `volatile` field|
|`set(value)`|Writes a new value — equivalent to writing a `volatile` field, visible to other threads immediately|
|`incrementAndGet()` / `getAndIncrement()`|Atomically increments by one (differ only in whether they return the value before or after)|
|`compareAndSet(expected, newValue)`|The CAS operation itself — returns `true` on success, `false` if the current value didn't match `expected`|
|`lazySet(value)`|Eventually writes the value — may be reordered with subsequent operations; useful when nulling out a reference purely for GC purposes and no thread needs to observe it immediately, trading strict visibility timing for better performance|
|`weakCompareAndSet(...)`|A weaker CAS that doesn't establish full happens-before ordering with other variables — deprecated since Java 9 in favor of more explicitly named variants (`weakCompareAndSetPlain()`, `weakCompareAndSetVolatile()`) to avoid the old name misleadingly implying `volatile`-level guarantees|

### Thread-Safe Counter, the Lock-Free Way

```java
public class SafeCounterWithoutLock {
    private final AtomicInteger counter = new AtomicInteger(0);

    int getValue() { return counter.get(); }
    void increment() { counter.incrementAndGet(); }
}
```

`incrementAndGet()` internally performs the same read-compute-write sequence as `counter++`, but as a **single atomic operation** (via a CAS retry loop under the hood) — no `synchronized`, no blocking, no lost updates.

---

## 12. Locks vs Atomics vs Concurrent Collections

||Locks (`synchronized`)|Atomic Variables|Concurrent Collections|
|---|---|---|---|
|Protects|A block of code / an object monitor|A **single** value|An entire **collection's** structure|
|Mechanism|Blocking — losing threads suspended|Lock-free CAS — losing threads retry|Mixed: fine-grained locks, CAS, or copy-on-write depending on the class|
|Best for|Coordinating multiple related steps/variables together|One counter/flag/reference updated frequently by many threads|Shared lists/maps/sets/queues accessed concurrently|
|Overhead under contention|High (context switches)|Low (retry loop, no suspension)|Varies by strategy — generally much lower than a single global lock|
|Complexity cost|Simple to reason about, easy to deadlock if used carelessly|Retry logic mostly hidden inside the class, but harder to compose multiple atomics together correctly|Just use the right class for the job — the hard part is choosing correctly, not implementing anything yourself|

---

