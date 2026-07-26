- **Volatile keyword**
	uses the cache coherence protocol (e.g. MESI "Modified, Exclusive, Shared, Invalid")
	[Medium Blog](https://medium.com/@pravvich/how-does-volatile-work-in-java-131f9c8b38fc)
	
* **PremGen vs MetaSpace** (JVM memory spaces, java 8 feature)

* **Deque** 
	 Used when required to add/remove from head and tail (modern implementation of the stack)
* **Collection views**

* **Navigable map:**
	A `NavigableMap` is a **sorted map with navigation capabilities**. It allows you to efficiently find:
	- nearest lower key (`lowerKey`)
	- nearest lower-or-equal key (`floorKey`)
	- nearest higher-or-equal key (`ceilingKey`)
	- nearest higher key (`higherKey`)
	- range views (`subMap`, `headMap`, `tailMap`)
	- reverse views (`descendingMap`)
	The most commonly used implementation is Tree Map, which stores entries in sorted order and performs navigation operations in **O(log n)** time.

* **Final Keyword** 
	The `final` keyword can be used as a **performance optimization technique**. When a variable is declared as final, the **Java compile**r may optimize the code by replacing the variable with a constant value. This optimization is known as **constant folding**.

* **Design Principle**
	- SOLID
	- DRY (Don't Repeat Yourself)
	- KISS (Keep it simple, Stupid!)
	- YANGI (You aren’t going to need it)
	- Favor composition over inheritance (IS-A vs HAS-A)
	
* **Method Reference Types**

| Type                    | Method Reference    | Lambda Equivalent           |
| ----------------------- | ------------------- | --------------------------- |
| Static Method           | `Integer::parseInt` | `s -> Integer.parseInt(s)`  |
| Object Instance Method  | `obj::method`       | `x -> obj.method(x)`        |
| Arbitrary Object Method | `String::concat`    | `(s1, s2) -> s1.concat(s2)` |
| Constructor             | `Class::new`        | `args -> new Class(args)`   |


- **HashMap Types**
![[Pasted image 20260624153958.png|478]]

* **Thread Lifecycle**
![[Pasted image 20260624161753.png|550]]

* **Executer Framework**
	[Refer to this article](https://dzone.com/articles/deep-dive-into-java-executorservice)

| Feature                | `execute()`                                        | `submit()`                                                         |
| ---------------------- | -------------------------------------------------- | ------------------------------------------------------------------ |
| **Return Type**        | `void`                                             | `Future<?>`                                                        |
| **Interface Origin**   | Introduced in `Executor`                           | Introduced in `ExecutorService`                                    |
| **Accepted Tasks**     | Only `Runnable`                                    | Both `Runnable` and `Callable`                                     |
| **Exception Handling** | Uncaught exceptions are printed to the stack trace | Exceptions are captured and rethrown when `Future.get()` is called |
| **Task Control**       | Cannot cancel the task after execution starts      | Can cancel the task using `Future.cancel()`                        |
| **Result Retrieval**   | No way to retrieve a result                        | Can retrieve results via `Future.get()`                            |
| **Use Case**           | Fire-and-forget tasks                              | Tasks where you need a result, status, or cancellation support     |
* **Lock Types**

| Lock Type                         | Reentrant | Multiple Readers | Timeout | Interruptible | Fair Option (FIFO Order, prevent starvation) |
| --------------------------------- | --------- | ---------------- | ------- | ------------- | -------------------------------------------- |
| synchronized (use intrinsic lock) | Yes       | No               | No      | No            | No                                           |
| ReentrantLock                     | Yes       | No               | Yes     | Yes           | Yes                                          |
| ReadWriteLock                     | Yes       | Yes              | Yes     | Yes           | Yes                                          |
| StampedLock                       | No        | Yes              | Limited | No            | No                                           |
| Semaphore                         | N/A       | Multiple permits | Yes     | Yes           | Fair option                                  |

* **Thread Failure** (deadlock, livelock, race condition, starvation)
	* **Deadlock Conditions**
		M H N C → Mutual Exclusion, Hold and Wait, No Preemption, Circular Wait
	
	* **Difference between deadlock and livelock**
		The difference between deadlock and livelock is, In the deadlock situation, two threads wait for another resource forever without releasing the acquired lock. 
		
		Whereas, in the livelock, both the thread release their lock at some point, however, both will not acquire all the resources they needed. They end up in keep trying forever and wasting the compute resources.

* **Lock-Free Programming (Atomic Variables)**
	Core operation is `	compareAndSet()` uses **Compare-And-Swap (CAS)** mechanism

| Scenario / Method Used      | Does it automatically retry? | What happens if the CAS step fails?                          |
| --------------------------- | ---------------------------- | ------------------------------------------------------------ |
| `compareAndSet()`           | **No**                       | Returns `false` instantly; changes nothing.                  |
| `incrementAndGet()`         | **Yes**                      | Loops internally, reads the fresh value, and tries again.    |
| `Extreme Thread Contention` | **Yes** (in high-level APIs) | High CPU usage, performance degradation, and latency spikes. |

* **ThreadLocal Design**
	Thread -> Map<ThreadLocal, Value>
