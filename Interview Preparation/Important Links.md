# 1- Intake 44 GitHub Repo

https://github.com/Omar-Ashraf9/Interview-Resources/tree/main

# 2- Comparator Vs Comparable

https://www.baeldung.com/java-comparator-comparable

# 3- Atomic Class

https://www.baeldung.com/java-atomic-variables

**Lock-Free Programming (Atomic Variables)**
	Core operation is `compareAndSet()` uses **Compare-And-Swap (CAS)** mechanism

| Scenario / Method Used      | Does it automatically retry? | What happens if the CAS step fails?                          |
| --------------------------- | ---------------------------- | ------------------------------------------------------------ |
| `compareAndSet()`           | **No**                       | Returns `false` instantly; changes nothing.                  |
| `incrementAndGet()`         | **Yes**                      | Loops internally, reads the fresh value, and tries again.    |

# 4- Streams

* **Streams internals** [Medium Blog](https://medium.com/@akhtar.jahan.qureshi/java-streams-internal-working-145f28243f67)

* **Map Vs FlatMap**

	![[Pasted image 20260704014211.png|584]]