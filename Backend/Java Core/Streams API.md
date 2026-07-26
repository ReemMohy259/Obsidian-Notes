---
share_link: https://share.note.sx/feirgn1p#3vs0+rcaQKlho7S2loylYA
share_updated: 2026-07-25T06:49:52+03:00
---
## Table of Contents

1. [[#What Is a Stream]]
2. [[#How Streams Actually Work Internally]]
3. [[#Creating Streams]]
4. [[#Intermediate vs Terminal Operations — Full Reference]]
5. [[#map / filter / flatMap — Deep Dive]]
6. [[#reduce — Deep Dive]]
7. [[#collect — Deep Dive]]
8. [[#groupingBy / partitioningBy — Deep Dive]]
9. [[#Collectors — Full Reference]]
10. [[#Optional]]
11. [[#Parallel Streams]]
12. [[#Sequential vs Parallel Streams]]
13. [[#Advantages / Disadvantages]]
14. [[#Streams vs Traditional Loops]]

---

## What Is a Stream

**Stream** — a sequence of elements supporting functional-style operations (map, filter, reduce...) computed **lazily**, in a **pipeline**, without modifying the underlying source. A stream is **not a data structure** — it doesn't store elements; it's a pipeline of computation over a source.

```
Source (List, Array, File...)
        │
        ▼
  Intermediate Ops (lazy — build a pipeline, don't run yet)
   filter → map → sorted → distinct ...
        │
        ▼
  Terminal Op (triggers execution — runs the WHOLE pipeline, once)
   collect / forEach / reduce / count ...
        │
        ▼
     Result
```

> [!important] Interview trap Q: "Does calling `.filter()` on a stream do any work?" A: No — intermediate operations are **lazy**; nothing executes until a terminal operation is called. Also: a stream can only be **consumed once** — calling a terminal op twice throws `IllegalStateException: stream has already been operated upon or closed`.

---

## How Streams Actually Work Internally

This is the part most people skip — and the part interviewers love to probe.

### 1. Every stream wraps a `Spliterator`

A `Spliterator` ("splittable iterator") is the low-level engine behind every stream. It knows how to:

- `tryAdvance()` — process one element at a time (like `Iterator.next()`)
- `trySplit()` — split itself into two spliterators, each covering half the elements (this is what makes **parallel streams** possible)
- `estimateSize()` — estimate how many elements remain

```
List.stream() → Spliterator over the List's internal array/nodes
```

Different sources split differently: an `ArrayList`'s spliterator splits cheaply (it's backed by an array — just cut the index range in half). A `LinkedList` or `Files.lines()` spliterator splits poorly (no random access), which is why parallelizing those sources gives little/no benefit.

### 2. Intermediate operations don't "run" — they build a chain of `Sink`s

Internally, each intermediate operation (`filter`, `map`, ...) is compiled into a `Sink` — a tiny object with an `accept(element)` method. Calling `.filter(p).map(f)` doesn't loop over anything; it **wraps** one sink around another, forming a chain:

```
Sink chain built (nothing executed yet):

  filterSink.accept(x) {
      if (predicate.test(x)) mapSink.accept(x);
  }

  mapSink.accept(x) {
      downstream.accept(function.apply(x));
  }
```

### 3. The terminal operation "pulls" — and drives everything

When a terminal operation runs, it asks the spliterator to push elements through the sink chain **one at a time**. This is why execution is "vertical" (one element flows through _every_ stage before the next element starts) rather than "horizontal" (one stage finishing for _all_ elements before the next stage begins):

```
element 1 → filter → map → sorted-buffer... 
element 2 → filter → map → sorted-buffer...
element 3 → filter → REJECTED, stops here for this element
...
(only once ALL elements are pulled does a terminal op like collect() finalize the result)
```

Stateless ops (`filter`, `map`) process purely element-by-element and can short-circuit immediately. Stateful ops (`sorted`, `distinct`) must **buffer all elements first** before they can emit anything — so `sorted()` on an infinite stream will hang forever, but `filter().limit(5)` on an infinite stream works fine.

```java
Stream.iterate(1, n -> n + 1)
    .filter(n -> n % 2 == 0)
    .limit(3)
    .toList();
// filter is stateless → fine
// limit is short-circuiting → stops the whole pipeline once 3 found
// Result: [2, 4, 6]

Stream.iterate(1, n -> n + 1)
    .sorted()          // ⚠️ stateful — needs ALL elements before it can emit even one
    .limit(3)
    .toList();
// HANGS FOREVER — sorted() can never receive "all" elements of an infinite stream
```

> [!important] Interview trap Q: "Why does `filter().limit()` work on an infinite stream but `sorted().limit()` doesn't?" A: `filter` is **stateless** and processes/short-circuits one element at a time. `sorted` is **stateful** — it must consume the entire stream before producing any output, which is impossible for an infinite source.

---

## Creating Streams

```java
// From a Collection
List<String> names = List.of("Ali", "Sara", "Omar");
Stream<String> s1 = names.stream();
Stream<String> parallel = names.parallelStream();

// From values / arrays
Stream<Integer> s2 = Stream.of(1, 2, 3);
Stream<Integer> s3 = Arrays.stream(new int[]{1, 2, 3}).boxed();

// Infinite streams (must limit or short-circuit!)
Stream<Integer> s4 = Stream.iterate(0, n -> n + 2).limit(5);   // 0,2,4,6,8
Stream<Double> s5 = Stream.generate(Math::random).limit(3);

// Primitive streams (avoid autoboxing overhead — IntStream/LongStream/DoubleStream)
IntStream s6 = IntStream.range(1, 5);       // 1,2,3,4   (exclusive end)
IntStream s7 = IntStream.rangeClosed(1, 5); // 1,2,3,4,5 (inclusive end)

// From a file (must be closed — try-with-resources)
try (Stream<String> lines = Files.lines(Path.of("data.txt"))) { ... }
```

---

## Intermediate vs Terminal Operations — Full Reference

### Intermediate Operations (lazy, return a `Stream`, can be chained)

|Category|Method|What it does internally|
|---|---|---|
|Stateless|`filter(Predicate)`|Tests each element; passes it downstream only if `true`|
|Stateless|`map(Function)`|Transforms each element 1-to-1, passes result downstream|
|Stateless|`mapToInt/Long/Double(...)`|Same as `map` but produces a primitive stream (avoids boxing)|
|Stateless|`flatMap(Function)`|Maps each element to a _stream_, then flattens all those streams into one|
|Stateless|`peek(Consumer)`|Passes elements through unchanged; side-effect only (debugging)|
|Stateful|`sorted()` / `sorted(Comparator)`|Buffers **all** elements, sorts, then emits in order|
|Stateful|`distinct()`|Buffers seen elements (via `hashCode`/`equals`) to filter duplicates|
|Stateful, short-circuiting|`limit(n)`|Emits only the first `n`, then signals upstream to stop|
|Stateful|`skip(n)`|Discards the first `n` elements before emitting|

### Terminal Operations (eager, consume the stream, produce a result or side effect)

|Category|Method|Returns|What it does internally|
|---|---|---|---|
|Reduction|`reduce(...)`|`Optional<T>` / `T`|Folds all elements into a single value|
|Mutable reduction|`collect(...)`|`R` (any container)|Accumulates elements into a mutable container|
|Reduction|`count()`|`long`|Counts elements (can sometimes skip iteration entirely if size is known)|
|Reduction|`min(Comparator)` / `max(Comparator)`|`Optional<T>`|Single pass, tracks current extreme|
|Short-circuiting|`anyMatch(Predicate)`|`boolean`|Stops at first `true`|
|Short-circuiting|`allMatch(Predicate)`|`boolean`|Stops at first `false`|
|Short-circuiting|`noneMatch(Predicate)`|`boolean`|Stops at first `true` (inverted)|
|Short-circuiting|`findFirst()`|`Optional<T>`|Stops at first element (respects encounter order)|
|Short-circuiting|`findAny()`|`Optional<T>`|Stops at _any_ element (faster in parallel — no ordering constraint)|
|Side-effect|`forEach(Consumer)`|`void`|Iterates, order **not guaranteed** in parallel streams|
|Side-effect|`forEachOrdered(Consumer)`|`void`|Iterates, order guaranteed even in parallel streams|
|Conversion|`toArray()`|`Object[]` / `T[]`|Collects into an array|
|Conversion|`toList()`|`List<T>` (Java 16+)|Shortcut for `collect(Collectors.toList())`, returns unmodifiable list|

---

## `map` / `filter` / `flatMap` — Deep Dive

### `filter` — Selection

Takes a `Predicate<T>` (a function `T → boolean`). Each element is tested; only elements returning `true` continue down the pipeline. Nothing is transformed — the type stays `Stream<T>`.

```java
List<Integer> nums = List.of(1, 2, 3, 4, 5, 6);

List<Integer> evens = nums.stream()
    .filter(n -> n % 2 == 0)   // predicate runs on EVERY element
    .toList();
// [2, 4, 6]
```

### `map` — 1-to-1 Transformation

Takes a `Function<T, R>`. Every input element produces **exactly one** output element (possibly of a different type). The stream's generic type changes from `Stream<T>` to `Stream<R>`.

```java
List<String> names = List.of("ali", "sara", "omar");

List<Integer> lengths = names.stream()
    .map(String::length)       // Stream<String> → Stream<Integer>
    .toList();
// [3, 4, 4]
```

```
map:  T ──function──▶ R      (cardinality: 1 input → 1 output, always)
```

### `flatMap` — 1-to-Many, Then Flatten

Takes a `Function<T, Stream<R>>` — each input maps to a **whole stream** of outputs, and `flatMap` merges (flattens) all those inner streams into a single continuous outer stream. This is the tool for "list of lists → single list" or "one word → its characters" style problems.

```java
List<List<Integer>> nested = List.of(List.of(1, 2), List.of(3, 4), List.of(5));

List<Integer> flat = nested.stream()
    .flatMap(List::stream)     // each List<Integer> → Stream<Integer>, then merged
    .toList();
// [1, 2, 3, 4, 5]
```

```
map alone:      List<List<Integer>>  →  Stream<Stream<Integer>>   (still nested — wrong!)
map + flatMap:  List<List<Integer>>  →  Stream<Integer>            (flattened — correct)

Visually:
  [ [1,2], [3,4], [5] ]
        │        │      │
     stream    stream  stream     ← map produces THESE separate streams
        └────────┴──────┘
              merged                ← flatMap concatenates them into ONE stream
        [1, 2, 3, 4, 5]
```

```java
// classic flatMap use case: words → all their individual letters, deduplicated
List<String> words = List.of("hello", "world");

List<Character> letters = words.stream()
    .flatMap(w -> w.chars().mapToObj(c -> (char) c))
    .distinct()
    .toList();
// [h, e, l, o, w, r, d]
```

> [!tip] Analogy `map` is a vending machine: one coin in, one snack out — always. `flatMap` is a box-opening machine: one box in, but the box might contain 0, 1, or 100 items — and it dumps all of them straight onto the belt.

---

## `reduce` — Deep Dive

`reduce` folds a stream down into a **single value** by repeatedly applying a combining function. There are three overloads, each building on the last:

### Overload 1 — No identity

```java
Optional<T> reduce(BinaryOperator<T> accumulator)
```

Returns `Optional` because an **empty stream has no result** to return.

```java
List<Integer> nums = List.of(4, 2, 7, 1);
Optional<Integer> max = nums.stream().reduce((a, b) -> a > b ? a : b);   // Optional[7]
```

### Overload 2 — With identity (starting value)

```java
T reduce(T identity, BinaryOperator<T> accumulator)
```

`identity` is both the starting value **and** the result for an empty stream — so this overload never needs `Optional`.

```java
int sum = nums.stream().reduce(0, (acc, x) -> acc + x);   // 14
```

**How it actually runs (sequential):**

```
acc = 0
acc = accumulator(0, 4)  = 4
acc = accumulator(4, 2)  = 6
acc = accumulator(6, 7)  = 13
acc = accumulator(13, 1) = 14
result = 14
```

### Overload 3 — With identity + combiner (needed for parallel streams)

```java
<U> U reduce(U identity, BiFunction<U,T,U> accumulator, BinaryOperator<U> combiner)
```

Used when the result type `U` differs from the element type `T` (e.g., reducing a `Stream<String>` down to an `int` total length). The **combiner** merges partial results from different threads/chunks — it's ignored in sequential streams but essential in parallel streams.

```java
List<String> words = List.of("a", "bb", "ccc");

int totalLength = words.stream().reduce(
    0,                                  // identity: U
    (partialSum, word) -> partialSum + word.length(),  // accumulator: (U, T) → U
    (sum1, sum2) -> sum1 + sum2         // combiner: (U, U) → U — merges chunks
);
// 6
```

```
Parallel execution with 2 threads:
Thread A: chunk ["a","bb"]   → accumulator → partial result 3
Thread B: chunk ["ccc"]      → accumulator → partial result 3
                                     │
                          combiner(3, 3) = 6   ← merge step
```

> [!important] Interview trap Q: "Why does `reduce` need a _combiner_ argument at all if the accumulator already combines values?" A: In a **parallel** stream, the source is split into chunks, and each chunk is reduced independently (in a different thread) using the accumulator. The **combiner** is what merges those independent partial results back into one final value — the accumulator alone never sees across chunks.

---

## `collect` — Deep Dive

`collect` performs a **mutable reduction**: instead of building a new immutable result at every step (like `reduce` does), it accumulates elements into one **mutable container** (like a `List` or `StringBuilder`), which is far more efficient for building collections.

### The Raw 3-Argument Form (what `Collectors` do under the hood)

```java
<R> R collect(Supplier<R> supplier,
              BiConsumer<R, ? super T> accumulator,
              BiConsumer<R, R> combiner)
```

|Part|Role|
|---|---|
|`supplier`|Creates a new, empty mutable container (called once per thread/chunk)|
|`accumulator`|Adds one element into the container (mutates it in place)|
|`combiner`|Merges two containers into one (used when parallel chunks must be joined)|

```java
List<String> names = List.of("Ali", "Sara", "Omar");

List<String> result = names.stream().collect(
    ArrayList::new,          // supplier: new empty ArrayList
    ArrayList::add,          // accumulator: list.add(element)
    ArrayList::addAll        // combiner: list1.addAll(list2)
);
```

### The Common Form: `collect(Collector)`

In practice, you almost never write the 3-arg form — you pass a pre-built `Collector` from the `Collectors` utility class, which packages up the supplier/accumulator/combiner (plus an optional final transformation step called the **finisher**) for you:

```java
List<String> result = names.stream().collect(Collectors.toList());
```

```
The Collector<T, A, R> interface bundles 4 things:
  supplier()      → A          (empty mutable container, e.g. new ArrayList<>())
  accumulator()   → (A, T)→void (container.add(element))
  combiner()      → (A, A)→A    (merge two containers — for parallel streams)
  finisher()      → A → R       (optional final transform, e.g. wrap as unmodifiable)
```

> [!tip] Analogy `reduce` is like folding origami — every step produces a brand-new folded shape. `collect` is like filling a basket — you keep dropping items into the _same_ basket instead of creating a new basket each time. That's why `collect` is preferred for building collections: far less object churn.

> [!important] Interview trap Q: "Can't you just use `reduce` to build a `List`?" A: Technically yes, but it's an anti-pattern — `reduce`'s accumulator must return a **new** immutable result each call, so reducing into a `List` means creating a new list on every single element (O(n²) copying). `collect` mutates one shared container instead, which is O(n).

---

## `groupingBy` / `partitioningBy` — Deep Dive

Both are **downstream-composable** `Collectors` — meaning you can nest another collector inside them to control exactly how each group's elements get combined.

### `groupingBy` — Classify into a `Map<K, List<T>>` (or custom downstream)

```java
Collector<T, ?, Map<K, List<T>>> groupingBy(Function<T, K> classifier)
```

Internally, `groupingBy` is implemented as: for each element, compute `classifier.apply(element)` to get a key, look up (or create) the group's bucket in an internal `HashMap`, and add the element to that bucket — by default using an internal `Collectors.toList()` as the "downstream" collector for each bucket.

```java
record Employee(String name, String dept, double salary) {}

List<Employee> employees = List.of(
    new Employee("Ali", "IT", 6000),
    new Employee("Sara", "IT", 7000),
    new Employee("Omar", "HR", 5000)
);

Map<String, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::dept));
// {IT=[Ali, Sara], HR=[Omar]}
```

```
For each employee:
  Ali  → key "IT" → bucket["IT"].add(Ali)
  Sara → key "IT" → bucket["IT"].add(Sara)
  Omar → key "HR" → bucket["HR"].add(Omar)

Result map:
  "IT" → [Ali, Sara]
  "HR" → [Omar]
```

#### Downstream Collectors — Changing What Goes in Each Bucket

```java
// Instead of a List per group, get a COUNT per group
Map<String, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::dept, Collectors.counting()));
// {IT=2, HR=1}

// Sum of salaries per group
Map<String, Double> totalSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::dept, Collectors.summingDouble(Employee::salary)));
// {IT=13000.0, HR=5000.0}

// Just the names per group, not the whole Employee object
Map<String, List<String>> namesByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::dept,
              Collectors.mapping(Employee::name, Collectors.toList())));
// {IT=[Ali, Sara], HR=[Omar]}
```

#### Multi-Level Grouping (nesting `groupingBy` inside `groupingBy`)

```java
Map<String, Map<Boolean, List<Employee>>> byDeptThenHighEarner = employees.stream()
    .collect(Collectors.groupingBy(Employee::dept,
              Collectors.groupingBy(e -> e.salary() > 5500)));
// {IT={true=[Ali, Sara]}, HR={false=[Omar]}}
```

### `partitioningBy` — Special Case: Split into Exactly Two Groups

```java
Collector<T, ?, Map<Boolean, List<T>>> partitioningBy(Predicate<T> predicate)
```

`partitioningBy` is a specialized (and slightly more efficient) version of `groupingBy` where the classifier is always a `boolean` predicate. The key difference: the resulting map **always has exactly two keys — `true` and `false`** — even if one group is empty (unlike `groupingBy`, which only creates keys that actually occur).

```java
Map<Boolean, List<Employee>> highVsLow = employees.stream()
    .collect(Collectors.partitioningBy(e -> e.salary() > 5500));
// {false=[Omar], true=[Ali, Sara]}
```

```
partitioningBy(predicate)          groupingBy(classifier)
┌─────────────────────┐            ┌─────────────────────┐
│ Always 2 keys:      │            │ N keys — only ones  │
│ true / false        │            │ that actually appear│
│ (backed by array,   │            │ (backed by HashMap) │
│  slightly faster)   │            │                     │
└─────────────────────┘            └─────────────────────┘
```

> [!important] Interview trap Q: "What's the real difference between `partitioningBy` and `groupingBy(booleanPredicate)`?" A: Functionally similar, but `partitioningBy` **guarantees both `true` and `false` keys exist** in the result map (even with empty lists), and is implemented with a lighter-weight structure internally (essentially a 2-slot array) instead of a general-purpose `HashMap`.

---

## `Collectors` — Full Reference

|Collector|Produces|Notes|
|---|---|---|
|`toList()`|`List<T>`|Mutable `ArrayList` in practice (implementation detail, not guaranteed)|
|`toUnmodifiableList()`|Immutable `List<T>`|Throws if you try to modify the result|
|`toSet()`|`Set<T>`|No guaranteed order, duplicates removed|
|`toMap(keyFn, valueFn)`|`Map<K,V>`|**Throws `IllegalStateException`** if two elements map to the same key|
|`toMap(keyFn, valueFn, mergeFn)`|`Map<K,V>`|`mergeFn` resolves key collisions, e.g. `(v1, v2) -> v1 + v2`|
|`joining()` / `joining(sep)` / `joining(sep, prefix, suffix)`|`String`|Concatenates elements (must be `CharSequence`)|
|`counting()`|`Long`|Count of elements — usually used as a downstream collector|
|`summingInt/Long/Double(fn)`|`Integer`/`Long`/`Double`|Sum of a mapped numeric field|
|`averagingInt/Long/Double(fn)`|`Double`|Average of a mapped numeric field|
|`summarizingInt/Long/Double(fn)`|`IntSummaryStatistics`, etc.|One pass gives min/max/sum/avg/count all at once|
|`minBy(comparator)` / `maxBy(comparator)`|`Optional<T>`|Collector form of `Stream.min`/`max` (useful as downstream)|
|`groupingBy(...)`|`Map<K, List<T>>` (or custom)|See deep dive above|
|`partitioningBy(...)`|`Map<Boolean, List<T>>`|See deep dive above|
|`mapping(fn, downstream)`|Depends on downstream|Transforms elements _before_ they reach a downstream collector|
|`filtering(predicate, downstream)`|Depends on downstream|Filters elements _before_ a downstream collector (Java 9+)|
|`reducing(...)`|Depends|Collector form of `reduce`, usable as a downstream collector|
|`collectingAndThen(collector, finisher)`|Depends|Runs a collector, then applies one more transform to the result|
|`teeing(c1, c2, merger)`|Depends|Runs **two** collectors over the same stream in one pass, then merges (Java 12+)|

```java
// toMap with a merge function — handling duplicate keys
List<Employee> list = List.of(
    new Employee("Ali", "IT", 6000),
    new Employee("Sara", "IT", 7000)
);
Map<String, Double> deptTotalSalary = list.stream()
    .collect(Collectors.toMap(Employee::dept, Employee::salary, Double::sum));
// {IT=13000.0}   ← without the merge function, this line throws IllegalStateException

// summarizing — one pass, five stats at once
IntSummaryStatistics stats = list.stream()
    .collect(Collectors.summarizingInt(e -> (int) e.salary()));
stats.getMin(); stats.getMax(); stats.getAverage(); stats.getSum(); stats.getCount();

// collectingAndThen — collect then lock it down as immutable
List<String> immutableNames = list.stream()
    .map(Employee::name)
    .collect(Collectors.collectingAndThen(Collectors.toList(), Collections::unmodifiableList));

// teeing — compute two aggregates in a SINGLE pass over the stream
record MinMax(double min, double max) {}
MinMax range = list.stream()
    .collect(Collectors.teeing(
        Collectors.minBy(Comparator.comparingDouble(Employee::salary)),
        Collectors.maxBy(Comparator.comparingDouble(Employee::salary)),
        (min, max) -> new MinMax(min.get().salary(), max.get().salary())
    ));
```

---

## Optional

`Optional<T>` is what stream terminal operations return instead of `null` when a result might not exist (`findFirst`, `reduce` without identity, `min`/`max`).

```java
Optional<String> first = names.stream()
    .filter(n -> n.startsWith("Z"))
    .findFirst();

String value      = first.orElse("Not found");                 // default value
String value2     = first.orElseGet(() -> computeDefault());    // lazy default
first.ifPresent(System.out::println);                           // run only if present
String value3      = first.orElseThrow(() -> new NoSuchElementException());
```

---

## Parallel Streams

- `.parallelStream()` (from a Collection) or `.stream().parallel()` splits the source into chunks and processes them **concurrently** across multiple threads, then combines the results.
- Powered by the **common `ForkJoinPool`** (shared JVM-wide — same pool used by `CompletableFuture.supplyAsync`), sized to `Runtime.getRuntime().availableProcessors() - 1` by default.
- Uses a **fork/join (divide-and-conquer)** model: recursively split via the source's `Spliterator.trySplit()` → process leaf chunks → merge (`combiner`) results back up the tree.

```
                     parallelStream()
                            │
                 ┌──────────┼──────────┐
                 ▼          ▼          ▼
             chunk 1    chunk 2    chunk 3      ← split via Spliterator.trySplit() (fork)
                 │          │          │
            Thread A   Thread B   Thread C      ← each runs the SAME pipeline
                 │          │          │
                 └──────────┼──────────┘
                            ▼
              combine results via combiner (join)
                            │
                            ▼
                          Result
```

```java
List<Integer> nums = IntStream.rangeClosed(1, 1_000_000).boxed().toList();

long sum = nums.parallelStream()
    .mapToLong(Integer::longValue)
    .sum();
```

### When Parallel Streams Help

|Good fit|Poor fit|
|---|---|
|Large data sets (thousands+ elements)|Small collections (thread coordination overhead > benefit)|
|CPU-heavy, stateless operations|I/O-bound work (network, disk, DB calls — threads just block)|
|`ArrayList`, arrays, `IntStream` ranges (cheap to split, random access)|`LinkedList`, `Iterator`-based sources (expensive to split)|
|Operations with no shared mutable state|Operations that mutate shared state|
|Associative, order-independent reduces|Order-sensitive logic without `forEachOrdered`|

### Common Pitfalls

```java
// ❌ BAD: shared mutable state — race condition, wrong/inconsistent results
List<Integer> results = new ArrayList<>();
nums.parallelStream().forEach(results::add);   // NOT thread-safe!

// ✅ GOOD: let collect() handle thread-safety internally via supplier/accumulator/combiner
List<Integer> results = nums.parallelStream()
    .collect(Collectors.toList());
```

```java
// ❌ forEach does NOT guarantee order in parallel streams
nums.parallelStream().forEach(System.out::println);   // scrambled output

// ✅ use forEachOrdered if order matters (but this reduces parallel benefit)
nums.parallelStream().forEachOrdered(System.out::println);
```

> [!important] Interview trap Q: "Will `parallelStream()` always be faster than `stream()`?" A: No — for small collections or I/O-bound/blocking operations, thread coordination overhead often makes it **slower**. Parallel streams shine on large, CPU-bound, splittable, stateless workloads.

> [!important] Interview trap Q: "What thread pool do parallel streams use?" A: The shared **common `ForkJoinPool`** by default — heavy parallel-stream usage elsewhere in the app (or blocking I/O inside one) can starve other parallel streams and `CompletableFuture` tasks running concurrently on the same pool.

---

## Sequential vs Parallel Streams

||Sequential Stream|Parallel Stream|
|---|---|---|
|Execution|Single thread, in order|Multiple threads, out of order (by default)|
|Order guarantee|Preserved|Not guaranteed unless `forEachOrdered`/ordered collector used|
|Best for|Small/medium data, I/O-bound, order-sensitive|Large data, CPU-bound, stateless|
|Thread safety needed?|No|Yes — avoid shared mutable state|
|Overhead|Minimal|Thread management, splitting, merging|
|Underlying mechanism|Simple `Spliterator.tryAdvance()` loop|Fork/Join framework, common `ForkJoinPool`, recursive `trySplit()`|

---

## Advantages / Disadvantages

|Advantages|Disadvantages|
|---|---|
|Declarative, readable functional-style code|Debugging pipelines can be harder than loops|
|Lazy evaluation — only computes what's needed|Streams are single-use (consumed once)|
|Easy parallelism with one method call|Parallel streams misused → race conditions/perf loss|
|Built-in short-circuiting (`findFirst`, `anyMatch`)|Checked exceptions are awkward inside lambdas|
|Composable pipeline of small, testable operations|Overusing streams for simple loops can hurt readability|

---

## Streams vs Traditional Loops

||Traditional Loop|Stream|
|---|---|---|
|Style|Imperative (how)|Declarative (what)|
|Mutability|Often mutates external variables|Encourages immutability|
|Laziness|N/A — runs immediately, line by line|Lazy until terminal op|
|Parallelism|Manual (threads, executors)|One method call: `.parallel()`|
|Reusability|Loop body can run any number of times|Stream can only be consumed once|