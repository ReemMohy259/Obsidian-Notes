### Transactions (Database Systems)

#### 🔹 Why Transactions Matter

* Databases allow **concurrent access** → multiple operations at once
* Must ensure **data integrity** despite shared data
* Requires **synchronization** of **reads/writes**

---

#### 🔹 Concurrency Control (Java Analogy)

* `synchronized` keyword:

  * Restricts access → only **one thread at a time**
  * Ensures **memory visibility** (updates shared globally)

> Similar concept applies in databases

---

#### 🔹 What is a Transaction?

* A group of **read/write operations**

* Executes as a **single unit**

* Either:

  * **All succeed**
  * **All fail (rollback)**

* Every DB operation runs inside a **transaction**
  *(even if not explicitly defined)*

---

#### 🔹 ACID Properties

**1. [[Atomicity]]**

* All or nothing execution

**2. [[Consistency]]**

* Database stays in a **valid state**

**3. [[Isolation]]**

* Transactions don’t **interfere** with each other

**4. [[Durability]]**

* Changes persist after commit (**even after crash**)

---

#### 🔹 Historical Context

* **1981 – Jim Gray**
  * Defined: Atomicity, Consistency, Durability

* **SQL-92**
  * Added: **[[Isolation Levels]]**
  * Result: **ACID model**

---

#### 🔹 Why ACID Transactions are Important

* **Effective access**
  * Maintains data integrity
* **Efficient access**
  * Reduces contention
  * Improves performance reduce **response time** & **throughput**

#### 🔹 Related Topics (Deep Dive)
* [[Read-Only Transactions]]  
* [[Transaction Boundaries]]