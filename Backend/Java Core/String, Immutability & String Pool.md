## Table of Contents

- [[#1. What Does "Immutable" Actually Mean?]]
- [[#2. Why Is String Immutable in Java?]]
- [[#3. The Java String Pool]]
- [[#4. String Literal vs `new String(...)`]]
- [[#5. Manual Interning]]
- [[#6. String Pool Location & Garbage Collection History]]
- [[#7. Compact Strings (Java 9+)]]
- [[#8. Mutable Alternatives: StringBuilder vs StringBuffer]]
- [[#9. How to Make Any Class Immutable — The Full Checklist]]
- [[#10. Immutability Edge Cases]]
---

## 1. What Does "Immutable" Actually Mean?

An **immutable object** is one whose internal state can never change after construction — you can't reassign its fields, and you can't mutate whatever those fields point to, once the object exists.

> [!tip] A famous quote worth knowing for interviews James Gosling (Java's creator), when asked when to prefer immutable objects, reportedly said he'd use them whenever possible — citing caching, safety, and easy reuse without duplication as the payoff.

Once a variable holds a reference to an immutable object, that object's observable state is frozen forever — only the **reference itself** can later be pointed at a _different_ object.

---

## 2. Why Is String Immutable in Java?

`String` is arguably the most heavily used type in Java, so the design choice to make it immutable pays off across the entire language. The four commonly cited reasons:

### 2.1. Enables the String Pool (Caching)

Because a `String`'s value can never change after creation, the JVM can safely let many variables share **one single copy** of an identical literal, instead of allocating a new object per occurrence — see [[#3. The Java String Pool]] for the full mechanism. This is only safe _because_ immutability guarantees none of the sharers can ever corrupt the shared value for the others.

### 2.2. Security

`String` is routinely used to carry sensitive values — usernames, passwords, URLs, file paths — and is used internally by the JVM's class-loading machinery. Immutability closes a specific class of bug/attack: if `String` were mutable, a method could validate an input string (e.g. checking it's alphanumeric to prevent injection) and then — _before_ actually using that validated value — have the underlying value changed out from under it by whoever else still holds a reference (the original caller, or another thread). Since the object can't change after that validation, the checked value is guaranteed to still be safe by the time it's actually used.

### 2.3. Thread Safety (Synchronization)

An object that can never change is automatically **safe to share across threads** with zero synchronization — there's no "modify" operation for two threads to race on. Any operation that looks like "changing" a `String` (`concat`, `replace`, `toUpperCase`, ...) actually just returns a **brand-new** `String` object, leaving the original untouched.

### 2.4. Safe & Fast Hash Code Caching

`String` overrides `hashCode()` to compute the hash **once**, on first call, and cache the result in a field — safe to do only because the content can never change afterward, so the cached hash can never go stale. This matters enormously because `String` is the most common key type in `HashMap`/`HashSet`/`Hashtable`, all of which call `hashCode()` on every insert and lookup for bucketing.

> [!warning] Why mutability would break hashed collections If a `String` key's content changed _after_ being inserted into a `HashMap`, its hash code would change too — but the map wouldn't know to re-bucket it. The entry would become unreachable via `get()`, effectively "lost" inside the map even though it's technically still there.

### 2.5. Performance (a Consequence of the Above)

The String Pool reduces heap usage, and cached hash codes speed up every hash-based lookup involving Strings — since `String` usage is so pervasive, these savings compound across the whole application.

> [!important] The unifying theme All of this boils down to one idea: because a `String` reference can be safely passed between methods and threads without anyone needing to worry the underlying value might shift underneath them, `String` behaves like a simple value — not an object you need to defensively copy or guard.

---

## 3. The Java String Pool

The **String Pool** (a.k.a. String Intern Pool) is a dedicated memory region where the JVM stores **one canonical copy** of each distinct string literal, so identical literals across the program can share the same object instead of each allocating its own.

```java
String s1 = "Hello World";
String s2 = "Hello World";

s1 == s2   // true — both point to the SAME pooled object
```

### How Interning Works

When the compiler encounters a string literal:

1. It searches the pool for an existing entry with that exact value.
2. If found → returns a reference to the **existing** pooled object (no new allocation).
3. If not found → creates a new entry in the pool and returns a reference to it.

This process — reusing or adding to the pool — is called **interning**.

---

## 4. String Literal vs `new String(...)`

```java
String constantString = "Baeldung";
String newString = new String("Baeldung");

constantString == newString   // false — different objects
```

||String Literal (`"..."`)|`new String("...")`|
|---|---|---|
|Where it lives|The String Pool (interned automatically)|A distinct object on the heap, outside the pool|
|Reuse|Shares the pooled instance with equal literals|Always allocates a **new** object, even for identical content|
|Recommended?|✅ Preferred — lets the JVM optimize automatically|Rarely needed — mainly historical/edge cases where a guaranteed-distinct instance matters|

```java
String third  = new String("Baeldung");
String fourth = new String("Baeldung");

third == fourth   // false — two separate heap objects, neither interned automatically
```

```java
String fifth = "Baeldung";
String sixth = new String("Baeldung");

fifth == sixth   // false — literal (pool) vs explicit heap object
```

> [!tip] Always use `.equals()` for value comparison `==` on `String` compares **references**, not content. The pool makes `==` "accidentally" work for literal-to-literal comparisons, but relying on that is fragile and considered bad practice — always use `.equals()` (or `.equalsIgnoreCase()`) to compare String _values_.

---

## 5. Manual Interning

You can force a heap-allocated `String` into the pool explicitly using `.intern()`:

```java
String constantString = "interned Baeldung";
String newString = new String("interned Baeldung");

constantString == newString              // false

String internedString = newString.intern();

constantString == internedString          // true — now shares the pooled instance
```

`intern()` checks the pool for an equal value; if present, it returns that pooled reference, otherwise it adds the current value to the pool and returns a reference to it.

---

## 6. String Pool Location & Garbage Collection History

|Java Version|Pool Location|Consequence|
|---|---|---|
|**Before Java 7**|**PermGen** space (fixed size, not garbage collected)|Interning too many Strings could exhaust PermGen and trigger `OutOfMemoryError`|
|**Java 7 onward**|**Heap** space (garbage collected)|Unreferenced pooled Strings can now be collected, substantially lowering OOM risk|

### Tuning the Pool

|JVM Option|Purpose|Era|
|---|---|---|
|`-XX:MaxPermSize=1G`|Increase PermGen size|Java 6 and earlier (obsolete once the pool moved to Heap)|
|`-XX:+PrintFlagsFinal` / `-XX:+PrintStringTableStatistics`|Inspect current pool size/stats|Java 7+|
|`-XX:StringTableSize=<n>`|Set the number of hash buckets in the pool|Java 7+|

Default bucket counts have grown over time: **1009** (pre-7u40) → **60013** (7u40 through Java 11) → **65536** (current). A larger table uses more memory but speeds up insertion since there are fewer collisions per bucket.

---

## 7. Compact Strings (Java 9+)

- **Before Java 9**: every `String` was backed internally by a `char[]`, using **UTF-16** — 2 bytes per character, _even for plain ASCII/Latin-1 text_ that only ever needs 1 byte per character.
- **Java 9+ ("Compact Strings")**: the internal representation automatically switches between a `byte[]` (for content that fits in Latin-1) and the old UTF-16 form, choosing whichever is sufficient for the actual content.
- **Effect**: significantly less heap consumed by typical (mostly-ASCII) text, which in turn reduces GC pressure — a transparent, zero-code-change performance win.

---

## 8. Mutable Alternatives: StringBuilder vs StringBuffer

Since `String` can't be modified in place, repeated concatenation actually creates a **new object every single time**:

```java
String s = "abc";
s = s + "def";   // NOT modifying "abc" — creates an entirely new String
```

For heavy or repeated text-building, Java provides two **mutable** sequence-of-characters classes that modify their own internal buffer instead of allocating a new object per change:

```java
StringBuilder sb = new StringBuilder("abc");
sb.append("def");   // modifies sb's internal buffer directly — no new object
```

### `StringBuilder` vs `StringBuffer`

||`StringBuilder`|`StringBuffer`|
|---|---|---|
|Introduced|Java 1.5, as a faster replacement for `StringBuffer`|Original mutable string class (since Java 1.0)|
|Thread safety|**Not synchronized** — not thread-safe|**Synchronized** — thread-safe|
|Performance|Faster (no synchronization overhead)|Slower, due to lock acquisition on every mutating call|
|API|Identical method surface to `StringBuffer`|Identical method surface to `StringBuilder`|
|When to use|Default choice — single-threaded use, or you already control synchronization externally|Only when multiple threads truly mutate the **same instance** concurrently without external synchronization|

> [!tip] Rule of thumb Default to `StringBuilder`. Reach for `StringBuffer` only in the rare case where a single mutable buffer is genuinely shared and mutated by multiple threads directly, and you're not already synchronizing access another way. In small-scale usage the performance gap is negligible; at large iteration counts `StringBuilder` measurably outperforms `StringBuffer` purely due to skipped synchronization — but always profile before optimizing on this basis alone.

---

## 9. How to Make Any Class Immutable — The Full Checklist

Beyond `String`, you'll often need to design your **own** immutable classes (value objects, DTOs, domain concepts like `Money`, `DateRange`) — this is the same discipline referenced in DDD Value Objects.

### The Checklist

1. **Declare the class `final`** — prevents subclasses from adding mutating behavior or overriding methods in ways that break the immutability contract.
2. **Make all fields `private final`** — `final` ensures each field is assigned exactly once (in the constructor) and never reassigned afterward.
3. **No setters** — don't expose any method that modifies state after construction.
4. **Initialize all fields via the constructor**, and validate there.
5. **If a field is a mutable type** (array, `List`, `Date`, another mutable class), **defensively copy it**:
    - **On the way in** (constructor) — copy what's passed in, don't store the caller's reference directly.
    - **On the way out** (getter) — return a copy (or an unmodifiable wrapper) instead of the internal reference directly.
6. **Don't leak `this` during construction** — avoid passing `this` to another object mid-constructor, before the object is fully initialized.

### Example — Getting It Right

```java
public final class Point {                 // ① final class

    private final int x;                    // ② private final fields
    private final int y;

    public Point(int x, int y) {             // ④ set only via constructor
        this.x = x;
        this.y = y;
    }

    public int getX() { return x; }          // ③ getters only, no setters
    public int getY() { return y; }
}
```

### Example — Defensive Copying With a Mutable Field

```java
public final class Schedule {

    private final Date meetingTime;          // Date is MUTABLE (has setTime(), etc.)

    public Schedule(Date meetingTime) {
        this.meetingTime = new Date(meetingTime.getTime());   // ⑤ copy IN
    }

    public Date getMeetingTime() {
        return new Date(meetingTime.getTime());                // ⑤ copy OUT
    }
}
```

Without both defensive copies:

- **Missing copy-in**: the caller keeps their original `Date` reference and can mutate it after construction — silently changing your "immutable" object's internal state from the outside.
- **Missing copy-out**: any caller of `getMeetingTime()` gets your **actual internal reference** and can mutate it directly — again breaking immutability, just from the other direction.

### Example — Collections

```java
public final class Team {

    private final List<String> members;

    public Team(List<String> members) {
        this.members = List.copyOf(members);        // immutable copy-in (Java 10+)
    }

    public List<String> getMembers() {
        return members;                              // already unmodifiable — safe to return directly
    }
}
```

`List.copyOf(...)` (or wrapping with `Collections.unmodifiableList(new ArrayList<>(members))` pre–Java 10) does **both** jobs at once: it copies the contents so future changes to the caller's original list don't affect you, **and** the result itself can't be mutated by anyone holding a reference to it.

> [!danger] `Collections.unmodifiableList(list)` alone is NOT enough `Collections.unmodifiableList(originalList)` only wraps the **same underlying list** — it prevents modification _through the wrapper_, but if the original caller still holds a reference to `originalList` directly, they can still mutate it, and your "immutable" field changes along with it. You must **copy**, not just wrap, unless you're certain no one else retains a reference to the source collection.

---

## 10. Immutability Edge Cases

### Records (Java 16+) — Immutability Built In

```java
public record Point(int x, int y) { }
```

Records automatically generate `final` fields, a canonical constructor, accessor methods (`x()`, `y()` — not `getX()`), `equals()`, `hashCode()`, and `toString()`. 
**Caveat**: records give you _shallow_ immutability for free — if a record field holds a mutable type, you still need to defensively copy it yourself inside a custom canonical constructor/accessor:

```java
public record Team(List<String> members) {
    public Team(List<String> members) {
        this.members = List.copyOf(members);   // still needed for a mutable field type
    }
}
```

### Inheritance & `final`

- If the class **isn't** `final`, a subclass could add mutable state or override a method to expose/mutate the parent's internals — this is exactly why step ① (`final` class) matters, even though every _field_ might individually be correct.
- Alternative to a `final` class: make the **constructor private** and expose only static factory methods, so no subclass can be created at all.

### Enum Fields

- `enum` constants are inherently singleton-like and effectively immutable by convention, but an enum **can** technically have mutable fields if you're not careful — the same defensive-copy discipline applies if an enum constant holds, say, a `List`.

### Lazy Initialization Inside an Immutable Class

- Caching a **derived** value the first time it's needed (exactly like `String.hashCode()` in [[#2.4. Safe & Fast Hash Code Caching]]) doesn't violate immutability, **as long as**:
    - the cached value is always computed purely from the object's own immutable fields (so it's always the same result), and
    - the write to the cache field is safe under concurrent access (commonly handled via `volatile` or by accepting redundant recomputation across threads rather than synchronizing).

### Arrays Are Always Mutable — Even `final` Ones

```java
private final int[] values;   // final only prevents reassigning the ARRAY REFERENCE

// values = new int[]{1,2,3};  // NOT allowed (final)
values[0] = 999;                // ALLOWED — the reference is fixed, but array contents are NOT
```

`final` on an array field only stops you from pointing the field at a _different_ array — it does nothing to prevent mutating the _contents_ of the array it already points to. Arrays **always** need the defensive-copy treatment (`Arrays.copyOf(...)`) both in and out — `List.copyOf(...)`-style immutability doesn't exist for raw arrays.

