---
share_link: https://share.note.sx/rfsuvirm#symXnFPW34cZfCDMKhB5rA
share_updated: 2026-07-25T16:47:57+03:00
---
## Table of Contents

- [[#1. JVM Runtime Memory — Overview]]
- [[#2. Stack vs Heap]]
- [[#3. The Heap — Young Gen vs Old Gen]]
- [[#4. Metaspace vs PermGen (Class Metadata)]]
- [[#5. Other Per-Thread Regions]]
- [[#6. Garbage Collection — Quick Recap]]
- [[#7. Common JVM Memory Flags]]
- [[#8. Quick Revision Cheat Sheet]]
## 1. JVM Runtime Memory — Overview

```
                           JVM MEMORY

┌────────────────────────────────────────────────────────────────────┐
│                         Shared Memory                              │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  HEAP                                                              │
│  ┌──────────────────────────┬───────────────────────────────────┐  │
│  │ Young Generation         │ Old Generation                    │  │
│  │ ┌─────┬─────┬─────┐      │ Long-lived objects                │  │
│  │ │Eden │ S0  │ S1  │      │                                   │  │
│  │ └─────┴─────┴─────┘      │                                   │  │
│  └──────────────────────────┴───────────────────────────────────┘  │
│                                                                    │
│  Metaspace (Java 8+) / PermGen (Java 7 and earlier)                │
│  • Class metadata                                                  │
│  • Method metadata                                                 │
│  • Runtime Constant Pool                                           │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘


            Per Thread (Each Thread Has Its Own)

        ┌──────────────────────────────┐
        │        Java Stack            │
        │ • Stack Frames               │
        │ • Local Variables            │
        │ • Operand Stack              │
        ├──────────────────────────────┤
        │       PC Register            │
        │ • Current Bytecode Address   │
        ├──────────────────────────────┤
        │    Native Method Stack       │
        │ • JNI / Native Calls         │
        └──────────────────────────────┘
```

| Region                                     | Shared or Per-Thread? | Stores                                                      |
| ------------------------------------------ | --------------------- | ----------------------------------------------------------- |
| **Heap**                                   | Shared                | All objects, arrays                                         |
| **Metaspace** (Java 8+) / **PermGen** (≤7) | Shared                | Class/method metadata                                       |
| **Java Stack**                             | Per-thread            | Method frames: local variables, partial results, references |
| **PC Register**                            | Per-thread            | Address of the currently executing JVM instruction          |
| **Native Method Stack**                    | Per-thread            | State for native (JNI/non-Java) method calls                |
### Different Layout

```
                    JVM Memory (Before Java 8)
+------------------------------------------------------+
|                      Heap                            |
|  +----------------+  +----------------------------+  |
|  | Young Gen      |  | Old Gen                    |  |
|  +----------------+  +----------------------------+  |
+------------------------------------------------------+

+-----------------------------------+
| PermGen (Managed by JVM) -> Fixed |
|-----------------------------------|
| Class Metadata                    |
| Method Metadata                   |
| Runtime Constant Pool             |
| Static Variables (*)              |
+-----------------------------------+

+----------------------+
|      Java Stack      |
+----------------------+

+----------------------+
|      PC Register     |
+----------------------+

+----------------------+
| Native Method Stack  |
+----------------------+
```

```
                    JVM Memory (From Java 8 and onwards)
+------------------------------------------------------+
|                      Heap                            |
|  +----------------+  +----------------------------+  |
|  | Young Gen      |  | Old Gen                    |  |
|  +----------------+  +----------------------------+  |
+------------------------------------------------------+

+-------------------------------------------+
|  Metaspace (Managed by OS) -> Dynamic     |
|-------------------------------------------|
| Class Metadata                            |
| Method Metadata                           |
| Runtime Constant Pool                     |
+-------------------------------------------+
        (Stored in Native Memory)

+----------------------+
| Java Stack           |
+----------------------+
```

---

## 2. Stack vs Heap

||Stack|Heap|
|---|---|---|
|**Stores**|Primitives, and **references** to objects; method call frames|The actual objects/arrays themselves|
|**Scope**|Per-thread — each thread has its own stack|Shared across all threads|
|**Lifetime**|Freed automatically when the method returns (frame popped)|Lives until no reference points to it → eligible for GC|
|**Allocation speed**|Very fast (simple pointer bump)|Slower (GC-managed, more bookkeeping)|
|**Size**|Small, fixed per thread (`-Xss`)|Large, configurable (`-Xms`/`-Xmx`)|
|**Overflow error**|`StackOverflowError` (e.g. infinite/deep recursion)|`OutOfMemoryError: Java heap space`|

```java
void method() {
    int x = 5;              // x lives on the STACK
    Person p = new Person(); // reference 'p' → STACK, the Person OBJECT itself → HEAP
}
```

```
Stack Frame               Heap
┌─────────┐               ┌───────────────┐
│ x = 5   │               │ Person object │
│ p = ●───┼──────────────►│ name: null    │
└─────────┘               └───────────────┘
```

> [!tip] One-line rule **Primitives and references live on the stack. Objects always live on the heap** — even if the reference to that object is a local variable.

---

## 3. The Heap — Young Gen vs Old Gen

The heap is divided by **object age**, because most objects die young (short-lived temporaries) while a few live a long time — generational GC optimizes for this pattern.

```
┌──────────────────────────────────┐   ┌──────────────────────┐
│           YOUNG GENERATION       │   │     OLD GENERATION   │
│  ┌───────┐  ┌──────┐  ┌──────┐   │   │   (Tenured Space)    │
│  │ Eden  │  │  S0  │  │  S1  │   │   │                      │
│  └───────┘  └──────┘  └──────┘   │   │  long-lived objects  │
└──────────────────────────────────┘   └──────────────────────┘
     ▲                    ▲
     │ new objects        │ survivors promoted after
     │ allocated here     │ surviving enough Minor GCs
```

||Young Generation|Old Generation|
|---|---|---|
|**Holds**|Newly created, short-lived objects|Long-lived objects that survived multiple GC cycles|
|**Sub-regions**|Eden + two Survivor spaces (S0, S1)|Single contiguous region (Tenured)|
|**GC type**|**Minor GC** — frequent, fast|**Major/Full GC** — less frequent, slower, scans more memory|
|**Typical size**|Smaller|Larger (most of the heap)|
|**Object flow**|New objects → Eden → survive a Minor GC → moved to a Survivor space → after enough survived cycles → **promoted** to Old Gen|Stays until unreferenced, then reclaimed by a Major/Full GC|

### Minor GC Flow (Simplified)

```
1. New object allocated in Eden
2. Eden fills up → Minor GC triggered
3. Live objects in Eden copied to Survivor space (S0)
4. Eden cleared entirely
5. Next Minor GC: live objects in Eden + S0 copied to S1; S0 cleared
6. Objects that survive enough of these cycles → PROMOTED to Old Gen
```

> [!important] Why Minor GC is cheap and Major GC is expensive Young Gen is small and mostly garbage on every cycle (most objects there are already dead) — so a Minor GC has very little **live** data to copy. Old Gen is large and mostly **live** data, so a Full GC must scan much more memory and typically causes longer pauses — this is the direct motivation behind low-pause collectors like G1, ZGC, and Shenandoah.

---

## 4. Metaspace vs PermGen (Class Metadata)

||PermGen (≤ Java 7)|Metaspace (Java 8+)|
|---|---|---|
|**Location**|Fixed-size JVM-managed region, separate from heap|**Native memory** (OS-allocated, off-heap)|
|**Stores**|Class metadata, method metadata, runtime constant pool, (pre-Java 7) interned Strings|Class metadata, method metadata, runtime constant pool|
|**Size**|Fixed — must configure `-XX:MaxPermSize` upfront|Grows dynamically as needed|
|**OOM error**|`OutOfMemoryError: PermGen space`|`OutOfMemoryError: Metaspace`|
|**JVM flags**|`-XX:PermSize`, `-XX:MaxPermSize`|`-XX:MetaspaceSize`, `-XX:MaxMetaspaceSize`|
|**Why it changed**|Fixed size caused frequent OOMs with dynamic-class-generating frameworks (Spring, Hibernate, CGLIB proxies, ByteBuddy) even with plenty of free RAM|Dynamic growth removes that artificial ceiling — bounded only by native memory or an explicit configured max|

```
Person class metadata → Metaspace (native memory)
Person object instances → Heap (managed by GC)
static fields' referenced objects → Heap (metadata about the field lives in Metaspace)
```

> [!tip] Recap The **String Pool** made the same PermGen → Heap move, one version earlier (Java 7) — same root motivation: get frequently-churned, GC-relevant data out of a fixed-size, hard-to-tune region.

---

## 5. Other Per-Thread Regions

| Region                            | Purpose                                                                                                                                                        |
| --------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Java Stack**                    | One per thread; holds a stack of **frames**, one per active method call — each frame holds that call's local variables, method parameters, and partial results |
| **PC (Program Counter) Register** | Tracks the address of the JVM instruction currently executing for that thread — lets each thread resume correctly after context switches                       |
| **Native Method Stack**           | Separate stack used when a thread calls native (non-Java, e.g. JNI/C) code                                                                                     |

---

## 6. Garbage Collection — Quick Recap

- An object becomes **eligible for GC** once nothing reachable from a GC root (active thread stacks, static fields, etc.) references it anymore.
- **Minor GC**: cleans Young Gen — frequent, fast, small pauses.
- **Major/Full GC**: cleans Old Gen (often the whole heap) — rarer, larger pauses.
- Modern collectors (G1 — default since Java 9, ZGC, Shenandoah) aim to minimize pause times by collecting incrementally/concurrently rather than stopping the world for a full sweep.

---

## 7. Common JVM Memory Flags

|Flag|Controls|
|---|---|
|`-Xms`|Initial heap size|
|`-Xmx`|Maximum heap size|
|`-Xss`|Thread stack size|
|`-XX:MetaspaceSize`|Initial threshold that triggers class-metadata GC|
|`-XX:MaxMetaspaceSize`|Upper limit on Metaspace (unbounded by default!)|
|`-XX:+PrintFlagsFinal`|Inspect effective JVM memory settings|

---

## 8. Quick Revision Cheat Sheet

|Concept|Remember|
|---|---|
|Stack|Per-thread; primitives + object **references**; fast; freed on method return|
|Heap|Shared; actual **objects**; GC-managed|
|`StackOverflowError`|Stack region exhausted (deep/infinite recursion)|
|`OutOfMemoryError: Java heap space`|Heap exhausted|
|Young Gen|Eden + 2 Survivor spaces; short-lived objects; cleaned by **Minor GC**|
|Old Gen|Long-lived, promoted objects; cleaned by **Major/Full GC**|
|Promotion|Object survives enough Minor GC cycles → moved from Young → Old Gen|
|PermGen (≤ Java 7)|Fixed-size, JVM-managed, held class metadata — removed in Java 8|
|Metaspace (Java 8+)|Native memory, grows dynamically, replaces PermGen|
|`OutOfMemoryError: Metaspace`|Native memory exhausted or `MaxMetaspaceSize` hit|
|Only class metadata|Lives in Metaspace/PermGen — ordinary objects are **always** on the heap|

---