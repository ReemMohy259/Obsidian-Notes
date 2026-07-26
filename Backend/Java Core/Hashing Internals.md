#### [Refer to this Blog](https://medium.com/@ashwinikunju27/hashmap-internals-in-java-a-deep-dive-into-hashing-buckets-bc36d15ecde9)

# 1. What is Hashing?

Hashing is a technique that maps a key to an array index using a mathematical function called a **hash function**.

Instead of searching sequentially:

```
Array:
[A][B][C][D][E]
```

we calculate

```
index = hash(key)
```

and directly access the location.

Average complexity:

```
Search  : O(1)
Insert  : O(1)
Delete  : O(1)
```

---

# 2. Basic Terminologies

### Key

The object used to identify a value.

```
"John" -> Employee object
```

---

### Value

The data associated with the key.

```
Key: "John"
Value: Employee(id=10)
```

---

### Entry (Node)

One key-value pair.

```
("John", Employee)
```

Internally Java stores:

```java
class Node<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
}
```

---

### Hash Function

A function that converts an object into an integer.

```
hash(key) -> integer
```

Example:

```
hash("ABC") = 78452
hash("XYZ") = 123441
```

The hash function should:

- deterministic
- fast
- uniformly distribute values
---

# 3. hashCode()

Every Java object inherits

```java
public int hashCode()
```

Example:

```java
String s = "Hello";

int h = s.hashCode();
```

returns

```
69609650
```

HashMap uses this value.

---

# 4. Bucket

A bucket is simply one position in the internal array.

```
table

index

0
1
2
3
4
5
6
7
```

Each bucket can contain:

- nothing
- one node
- multiple nodes
---
# 5. Capacity

Capacity = number of buckets.

Example:

```
new HashMap<>(16)

capacity = 16
```

Indexes:

```
0..15
```

---

# 6. Load Factor

Load factor determines when HashMap should resize.

Formula:

```
load factor = size / capacity
```

Default:

```
0.75
```

Example

initial capacity

```
16
```

threshold

```
16 × 0.75 = 12
```

After inserting the 13th element,

HashMap resizes.

---

# 7. Threshold

Threshold =

```
capacity × loadFactor
```

Example

```
capacity = 32

load factor = 0.75

threshold = 24
```

Once size exceeds 24,

resize occurs.

---

# 8. Hash Collision

Collision occurs when two keys produce the same bucket index.

Example

```
hash("ABC") -> bucket 5

hash("XYZ") -> bucket 5
```

Now both must exist together.

```
bucket 5

ABC

XYZ
```

---

# 9. Why collisions happen?

Because

```
hashCode()

returns int

-2^31 ... 2^31-1
```

while

```
capacity

16
32
64
128
```

Many integers must map to the same bucket.

Collisions are unavoidable.

---

# 10. Compression Function

After hashCode(),

we still need an array index.

Conceptually:

```
bucket = hash % capacity
```

Example

```
hash = 150

capacity = 16

bucket = 150 % 16 = 6
```

This is called compression.

---

# 11. Does Java really use modulo?

No.

Because capacity is always a power of 2.

Instead Java uses:

```
index = (capacity - 1) & hash
```

instead of

```
hash % capacity
```

because bitwise AND is much faster.

---

Example

capacity

```
16

binary

10000

capacity-1

1111
```

Suppose

```
hash

10110110

AND

00001111

=

00000110

= 6
```

---

# 12. Why capacity is always power of 2?

Suppose

capacity = 16

```
10000

capacity-1

1111
```

Bitwise AND perfectly extracts lower bits.

Fast.

Efficient.

Even distribution.

---

# 13. Java Hash Spreading

Java does NOT directly use hashCode().

Instead:

```
hash = h ^ (h >>> 16)
```

Why?

Because many classes produce similar lower bits.

Java mixes upper bits into lower bits.

This improves bucket distribution.

---

Example

```
original

11001100101010101100111100001111

shift

00000000000000001100110010101010

xor

11001100101010100000001110100101
```

Better randomness.

---

# 14. HashMap Internal Structure

```
table[]

index 0

null

index 1

Node

index 2

Node -> Node -> Node

index 3

TreeNode

index 4

null
```

Internally

```
Node<K,V>[] table
```

---
# 17. Why equals() is needed?

Different objects can have identical hashCode().

Example

```
A.hashCode() = 100

B.hashCode() = 100
```

HashMap must verify

```
A.equals(B)
```

before returning the value.

---

# 18. Importance of hashCode() and equals()

Rule:

If

```
a.equals(b)
```

then

```
a.hashCode() == b.hashCode()
```

must be true.

Otherwise HashMap breaks.

---

# 19. Collision Resolution Strategies

There are two major families.

```
Collision Resolution

├── Open Addressing
└── Closed Addressing (Separate Chaining)
```

---

# 20. Closed Addressing (Separate Chaining)

Used by Java HashMap.

Each bucket stores multiple elements.

```
bucket

A -> B -> C -> D
```

Advantages

- simple
    
- deletion easy
    
- resizing easy
    

Disadvantages

- extra memory
    
- linked list traversal
    

---

# 21. Open Addressing

No linked list.

Every element stays inside the array.

Collision?

Search another empty slot.

```
Array

A

B

empty

C

D
```

Popular in C/C++ implementations.

---

# 22. Linear Probing

Collision?

Move one step.

```
index

5 occupied

↓

6

↓

7

↓

8
```

Formula

```
(h+i)%capacity
```

Problems:

Primary clustering.

---

# 23. Quadratic Probing

Instead of

```
+1
```

use

```
+1²

+2²

+3²
```

Formula

```
(h+i²)%capacity
```

Better distribution.

---

# 24. Double Hashing

Use another hash function.

```
index

h1(key)+i*h2(key)
```

Best distribution among open addressing techniques.

---

# 25. Primary Clustering

Linear probing creates long consecutive occupied blocks.

```
XXXXX_____
```

New elements continue joining this cluster.

Performance decreases.

---

# 26. Secondary Clustering

Quadratic probing reduces primary clustering but keys with identical initial hash still follow identical probe sequences.

---

# 27. Why Java does NOT use open addressing?

Deletion becomes complicated.

High load factor reduces performance.

Separate chaining is easier to resize.

Supports tree conversion.

---

# 28. Linked List before Java 8

Before Java 8

```
bucket

A

↓

B

↓

C

↓

D
```

Worst case

```
O(n)
```

---

# 29. Treeification (Java 8+)

If collisions become excessive,

linked list converts into a Red-Black Tree.

```
Before

A

↓

B

↓

C

↓

D

↓

E

↓

F

↓

G

↓

H

↓

I


After

        D
      /   \
     B     G
    / \   / \
   A  C F  H
         \
          I
```

---

# 30. Treeify Threshold

If bucket size >=

```
8
```

Java converts linked list to tree.

```
TREEIFY_THRESHOLD = 8
```

---

# 31. Untreeify Threshold

If elements decrease below

```
6
```

tree becomes linked list again.

```
UNTREEIFY_THRESHOLD = 6
```

---

# 32. Minimum Treeify Capacity

Treeification happens only if

```
capacity >= 64
```

Otherwise,

Java prefers resizing instead.

```
MIN_TREEIFY_CAPACITY = 64
```

Reason:

Resizing often distributes elements and removes collisions more efficiently than creating trees.

---

# 33. Resize (Rehashing)

Suppose

```
capacity = 16

threshold = 12
```

Insert 13th element.

New capacity:

```
32
```

Every node is redistributed into new buckets.

---

# 34. Why resize doubles capacity?

Because

```
16 -> 32 -> 64 -> 128
```

maintains power-of-two optimization.

---

# 35. Rehashing Cost

Resize complexity

```
O(n)
```

because every node must be relocated.

But resizing happens infrequently.

Therefore average insertion remains

```
Amortized O(1)
```

---

# 36. Complexity Summary

|Operation|Average|Worst|
|---|---|---|
|put()|O(1)|O(n)|
|get()|O(1)|O(n)|
|remove()|O(1)|O(n)|
|get() with tree|O(log n)|O(log n)|

---

# 37. Internal put() Flow (Simplified)

```
put(key,value)

↓

hashCode()

↓

hash = h ^ (h >>> 16)

↓

index = (n-1)&hash

↓

bucket empty?

├── yes → insert
│
└── no
      │
      key exists?
      │
      ├── yes → replace value
      │
      └── collision
             │
             linked list/tree traversal
             │
             insert node
             │
             bucket size >=8 ?
             │
             ├── yes + capacity>=64
             │        ↓
             │    treeify
             │
             └── no
                    ↓
                 linked list

↓

size++

↓

size > threshold ?

↓

resize()
```

---

# 38. Common Interview Questions

### Why must equal objects have the same hashCode?

Because `HashMap` first locates a bucket using the hash code and only then uses `equals()`. If equal objects produce different hashes, they end up in different buckets and lookups fail.

---

### Why is the default load factor `0.75`?

It provides a practical balance:

- Lower load factor (e.g., 0.5): fewer collisions, more memory usage.
    
- Higher load factor (e.g., 0.9): better memory utilization, more collisions.
    

`0.75` is an empirically good trade-off.

---

### Why are collisions unavoidable?

A 32-bit `hashCode()` can produce about 4.3 billion values, but an internal table typically has only 16, 32, 64, or 128 buckets. Multiple hash values must therefore map to the same bucket.

---

### Why is `HashMap` not synchronized?

To avoid synchronization overhead and maximize performance. Concurrent access should use `ConcurrentHashMap`, which is designed for thread-safe operations with much better scalability than synchronizing an entire `HashMap`.

---