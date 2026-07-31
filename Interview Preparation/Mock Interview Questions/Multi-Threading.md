# Q1: What are the types of threads in java

* System thread
* User defined thread 
* Daemon thread

> By default all java applications are multi-threaded at least system thread and main thread

==Follow Up: What is the Daemon thread and what will be the output of the following code?==

```Java
public class Zoo {  
    public static void pause() {  
        try {  
            Thread.sleep(10_000);  
        } catch (InterruptedException e) {}  
        System.out.println("Thread finished!");  
    }  
  
    public static void main(String[] unused) {  
        var job = new Thread(() -> pause());  
        job.setDaemon(true);  
        job.start();  
        System.out.println("Main method finished!");  
    }  
}
```

> setDaemon() method must called before start() method otherwise will throw exception

# Q2: A multithreaded application produces intermittently incorrect results without crashing. What is the root cause and how do you fix it ?

Intermittent incorrect results without a crash are the signature of a race condition. The application is not broken in a way that throws exceptions or corrupts memory visibly, it is broken in a way that only manifests when thread scheduling produces a specific interleaving of operations. Because the bug depends on timing, it disappears under a debugger, passes in low-load tests, and surfaces unpredictably in production.

**Understanding the root cause**
The underlying cause is shared mutable state being accessed by multiple threads without synchronization. A read-modify-write sequence like reading a counter, incrementing it, and writing it back appears atomic in source code but compiles into multiple instructions. Two threads executing this sequence simultaneously can both read the same value, both compute the same incremented result, and both write it back, losing one update entirely. The program continues running with a silently wrong value.

**Why these bugs are hard to reproduce**
Race conditions are timing-sensitive. The window where two threads can interleave incorrectly is often measured in nanoseconds. Running under a debugger serializes execution enough to close that window, making the bug disappear. Higher load, a different CPU core count, or a different OS scheduler can open or close the window unpredictably, which is why the bug appears intermittently rather than consistently.

The happens-before relationship in the Java memory model defines when a write by one thread is guaranteed to be visible to a read by another. Without a synchronization action establishing happens-before between the write and the read, the JVM and CPU are free to reorder and cache operations in ways that make a write invisible to another thread even after it has completed.

**Fixes**
The right fix depends on what the shared state needs to do. For simple counters and flags, atomic variables eliminate the race without any locking

# Q3: What is the difference between Synchronized block and Synchronized method ?
 
 ==Answer about lock/monitor freedom and also about block scope==

---
# Another Questions

1. What is the difference between the start() and run() method?
2. Sleep vs wait
3. What is context switching in multithreading?
4. What is The Executor Framework ?
5. What is atomic operation?
6. What is the difference between a Process and a Thread?
7. What is a Race Condition?
8. What does synchronized do?
9. What is the difference between Runnable and Callable?
10. What is ExecutorService and why is it better than creating threads manually?
11. What is Deadlock?
12. what is the four reasons to deadlock ?
13. What is volatile?
14. Difference Between Concurrency and Parallelism
15. What is Thread Starvation?
16. Thread Lifecycle in Java
17. What is ThreadLocal?
18. What is the Fork/Join framework?
19. What is the difference between Deadlock, Livelock, and Starvation?
20. What is the happens-before relationship?
21. What are the differences between mutex and semaphore?