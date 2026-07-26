---
share_link: https://share.note.sx/um0shvit#M5+UMdbbPeX2GLF3FNLn5Q
share_updated: 2026-07-23T20:01:36+03:00
---
## What Is an Escaping Reference?

**Escaping reference** — when a class hands out a direct reference to its internal mutable state (a field, array, or collection), letting outside code silently mutate the object's "private" data without going through any of its methods.

```
Class A (internal mutable state)          Outside Code
   private Date hireDate ─────getHireDate()────> Date ref
                                                     |
                                                  ref.setYear(1999)
                                                     |
   A's hireDate is now changed <─────────────────────┘
   (encapsulation broken — A never touched it!)
```

---

## Setup — The Bug

```java
public class Employee {
    private Date hireDate;

    public Employee(Date hireDate) {
        this.hireDate = hireDate;   // ⚠️ stores the caller's reference directly
    }

    public Date getHireDate() {
        return hireDate;         // ⚠️ hands out the internal reference directly
    }
}
```

```java
Date d = new Date();
Employee e = new Employee(d);
d.setYear(1999);          // mutates Employee's internal state from outside!

Date leaked = e.getHireDate();
leaked.setYear(1800);     // mutates Employee's internal state again!
```

Two escape points: the **constructor/setter** (reference comes *in*) and the **getter** (reference goes *out*). Both must be fixed.

> [!important] Interview trap Q: "Is this class immutable?" (shows a class with only a getter, no setters, but the getter returns a mutable field directly) A: No — it's *not truly immutable* if the getter returns a direct reference to a mutable object. The field can't be reassigned, but the object it points to can still be mutated externally.

---
## Defensive Copy — Copy at the *Boundary*, Not the Whole Object

- The fix for escaping references: copy mutable data **on the way in** (constructor/setter) and **on the way out** (getter), so no one outside the class ever touches the real internal object.
- Doesn't require copying the *entire* object graph like deep copy — only the mutable fields that cross the class boundary.

```
        constructor/setter                    getter
Caller ───────copy in────────► Employee ───────copy out────────► Caller
 (owns d)   (Employee owns    (internal      (Employee still    (owns a
             its own copy)     hireDate)       owns hireDate)     new copy)
```

```java
public final class Employee {
    private final Date hireDate;

    public Employee(Date hireDate) {
        this.hireDate = new Date(hireDate.getTime());   // defensive copy IN
    }

    public Date getHireDate() {
        return new Date(hireDate.getTime());            // defensive copy OUT
    }
}
```

> [!tip] Analogy Escaping reference = giving someone the key to your actual house. Defensive copy = giving them a photo of your house instead — they can look at it, even "modify" the photo, but your real house is untouched.

---
## Shallow Copy — Copies the Container, Not the Contents

- Duplicates the top-level object and its fields.
- If a field is a **reference type** (array, object, collection), the copy gets the **same reference** — original and copy still share that nested object.
- Primitives (`int`, `boolean`, etc.) are safely duplicated since they're copied by value.

```
Original Object                 Shallow Copy
┌──────────────┐                ┌──────────────┐
│ name: "Bob"  │  (copied)      │ name: "Bob"  │
│ dates: ref A │ ──────┬───────►│ dates: ref A │
└──────────────┘       │        └──────────────┘
                       ▼
                 [Date Object A]   ← BOTH point here
                 mutate it once → both "copies" see the change
```

```java
public class Employee implements Cloneable {
    private String name;
    private Date hireDate;

    @Override
    public Employee clone() throws CloneNotSupportedException {
      return (Employee) super.clone(); // shallow — hireDate reference is shared
    }
}
```

---

## Deep Copy — Copies the Container *and* the Contents

- Recursively duplicates every mutable object reachable from the original.
- Original and copy share **nothing** mutable — fully independent.

```
Original Object                 Deep Copy
┌──────────────┐                ┌──────────────┐
│ name: "Bob"  │  (copied)      │ name: "Bob"  │
│ dates: ref A │                │ dates: ref B │  ← new, independent object
└──────────────┘                └──────────────┘
       │                              │
       ▼                              ▼
 [Date Object A]                [Date Object B]   (identical value, separate memory)
```

```java
public class Employee implements Cloneable {
    private String name;
    private Date hireDate;

    @Override
    public Employee clone() throws CloneNotSupportedException {
        Employee copy = (Employee) super.clone();
        copy.hireDate = (Date) hireDate.clone();   // deep — new Date object
        return copy;
    }
}
```

For nested collections, clone each element too (shallow `clone()` on a `List` only copies the list itself, not its elements):

```java
List<Date> deepCopy = new ArrayList<>();
for (Date d : original) {
    deepCopy.add((Date) d.clone());
}
```

---
## How to Implement Copies in Java

| Technique | Copy type | Notes |
|---|---|---|
| `Object.clone()` + `Cloneable` | Shallow by default | Override to deep-clone specific fields |
| Copy constructor `new Employee(other)` | Shallow or deep (your choice) | Most readable; no `Cloneable` quirks |
| Static factory `Employee.copyOf(other)` | Shallow or deep | Same idea, static form |
| `Arrays.copyOf(arr, arr.length)` | Shallow (array itself) | Elements still shared if they're objects |
| Serialization (write then read back) | Deep | Slow; requires every nested class `Serializable` |
| Manual field-by-field copy | Deep | Full control, most boilerplate |
| Immutable objects (e.g. `String`, `LocalDate`) | No copy needed | Safe to share freely — can't be mutated |

### Copy Constructor Pattern (recommended)

```java
public class Employee {
    private final String name;
    private final Date hireDate;

    public Employee(String name, Date hireDate) {
        this.name = name;
        this.hireDate = new Date(hireDate.getTime());   // defensive copy
    }

    // Copy constructor
    public Employee(Employee other) {
        this.name = other.name;
        this.hireDate = new Date(other.hireDate.getTime());  // deep copy of mutable field
    }

    public Date getHireDate() {
        return new Date(hireDate.getTime());            // defensive copy
    }
}
```

### Using Immutable Types to Avoid the Problem Entirely

```java
import java.time.LocalDate;

public class Employee {
    private final LocalDate hireDate;   // immutable — no copying needed anywhere

    public Employee(LocalDate hireDate) {
        this.hireDate = hireDate;       // safe: caller can't mutate a LocalDate
    }

    public LocalDate getHireDate() {
        return hireDate;                // safe: this reference can't be mutated
    }
}
```

> [!important] Interview trap Q: "When should you use defensive copying vs just making the field immutable?" A: If you control the type (e.g. it's your own class), prefer making it **immutable** — no copying needed, no performance cost. Defensive copying is the fallback when the field's type is inherently mutable and outside your control (e.g. legacy `Date`, arrays, collections).

---

## Shallow vs Deep vs Defensive — Comparison

| | Shallow Copy | Deep Copy | Defensive Copy |
|---|---|---|---|
| What's copied | Top-level object only | Entire object graph, recursively | Only the specific mutable field(s) crossing a boundary |
| Nested mutable objects | Shared (same reference) | Duplicated (independent) | Duplicated at the boundary only |
| Typical use | Cheap duplication when nested state won't change | Full independent snapshot of an object | Protecting encapsulation in constructors/getters/setters |
| Cost | Low | Can be high (recursive) | Low (only touches exposed fields) |
| Fixes escaping references? | ❌ No — nested refs still leak | ✅ Yes, if applied everywhere | ✅ Yes, precisely where needed |

---

## Advantages / Disadvantages

| Advantages | Disadvantages |
|---|---|
| Defensive/deep copies preserve encapsulation | Extra allocation on every call |
| Prevents subtle "spooky action at a distance" bugs | Easy to forget a nested field (partial deep copy) |
| Works well alongside immutability | `Object.clone()` API is famously awkward/error-prone |
| Copy constructors are simple and explicit | Deep copy of large graphs can be expensive |

---

## Escaping Reference vs Shallow Copy vs Deep Copy vs Defensive Copy

|| Escaping Reference (the bug) | Shallow Copy | Deep Copy | Defensive Copy |
|---|---|---|---|---|
| Role | Anti-pattern — leaks internal state | Copy technique | Copy technique | Copy *strategy* applied at API boundaries |
| Where it happens | Any getter/setter/constructor | Anywhere an object is duplicated | Anywhere a full independent copy is needed | Specifically at points where objects enter/leave a class |
| Goal | (none — it's the problem) | Fast duplication | Full isolation | Encapsulation safety |