---
share_link: https://share.note.sx/wnz97imf#FGxYMdIkj3NZBMNh3pYNZQ
share_updated: 2026-06-27T15:40:53+03:00
---
a **structural** design pattern:
* Structural design patterns explain how to assemble objects and classes into larger structures, while keeping these structures flexible and efficient.

#### The problem:
1. imagine you have an API and you want to call it and use its result (Remote call) -> Time consuming
2. you have an expensive object and you want to use it -> Slow startup
3. you want only specific clients to be able to use the service object
4. you want to add logging feature to all requests and responses on that class
---
#### Naive Solution

![[Pasted image 20260626200732.png|334]]

> This solution is a valid solution, but it's violate SOLID principles

What if i told you that we can use another light weight object instead of the real one and delegate the job to the real one is this solves the issue ? 

---
#### Enhanced Solution
**Proxy** is a structural design pattern that lets you provide a substitute or placeholder for another object. A proxy controls access to the original object, allowing you to perform something either before or after the request gets through to the original object.

![[Pasted image 20260626201118.png|609]]

![[Pasted image 20260626201152.png|614]]

> One of the most important conditions in this pattern that the proxy and the real subject should be **interchangeable**, as they both implement the same interface 

---
#### Decorator vs Proxy

| Aspect                | Proxy                                         | Decorator                                   |
| --------------------- | --------------------------------------------- | ------------------------------------------- |
| **Purpose**           | Control access to the target                  | Add behavior dynamically                    |
| **Lifecycle Control** | Managed by the **proxy itself**.              | Managed by the **client** (passed in).      |
| **Object Dependency** | Target can be **lazy-loaded** on demand.      | Target **must exist** before decoration.    |
| **Composition**       | Typically wraps a **single, specific class**. | Supports **recursive wrapping** (stacking). |
| **Interface Impact**  | Maintains the **exact same interface**.       | Provides an **enhanced interface**.         |

---
#### Proxy Implementation Approaches
A proxy and the real service must be **interchangeable**. There are two ways to achieve this:

##### 1. Interface-Based Proxy (Preferred)
Both the proxy and the real service implement the same interface.

```text
	    Service   Interface
           ▲          ▲
           │          │
    Real Service    Proxy
```

**Advantages**
- Loose coupling
- Follows the Dependency Inversion Principle (DIP)
- Easy to swap implementations
- Preferred approach in Java and Spring

**Example**
```java
public interface PaymentService {}

public class PaymentServiceImpl implements PaymentService {}

public class PaymentProxy implements PaymentService {}
```

##### 2. Class-Based Proxy (Inheritance)
If no interface exists, the proxy extends the real service.

```text
Real Service
      ▲
      │
    Proxy
```

**When to use**
- Legacy code without interfaces
- Refactoring to interfaces is impractical
- Need a proxy without changing existing clients

**Example**
```java
public class PaymentService {}

public class PaymentProxy extends PaymentService {}
```

##### Spring Connection
Spring supports both approaches:

* JDK Dynamic Proxy
	- Uses **interfaces**	    
	- Proxy **implements** the target interface
	- Requires at least one interface
 * CGLIB Proxy
	- Uses **inheritance**
	- Proxy **extends** the target class
	- Used when no suitable interface exists


> [!NOTE] Fail Inheritance-based Proxy Case
> Inheritance-based proxies are excellent for intercepting method calls, but they are a poor choice for lazy initialization of heavyweight objects because the **superclass constructor executes before the proxy can intervene**.

---
#### Proxy Types
##### 1- Virtual Proxy (Lazy Initialization)
* Problem -> Creating the real object is expensive.

Examples:
- Loading a 2GB video
- Loading an AI model
- Opening a database connection

**Instead of:**
```
Application Starts
        │
        ▼
Create Video (10 seconds)
        │
        ▼
User may never watch it!
```

**Use a proxy:**
```
Application Starts
       │
       ▼
Create VideoProxy (very cheap)
       │
       ▼
User clicks Play
       │
       ▼
Proxy creates RealVideo
       │
       ▼
Play Video
```

##### 2-  Access control (protection proxy)
This is when you want only specific clients to be able to use the service object. The proxy can pass the request to the service object only if the client’s credentials match some criteria.

**Spring Example**
```java
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser() {
	// Do some logic
}
```

**Internally**
```
Client
	│
    ▼
Security Proxy
	│
    ▼
Check Permission
	│
    ▼
deleteUser() // If valid credintials
```

##### 3- Remote Proxy
The object is on another machine, remote server.

**Without proxy**, Client would need to know
- HTTP
- TCP
- Serialization
- Authentication
- Retries
Too complicated.

**Instead** -> The proxy hides networking completely.
```
Client
	↓
Remote Proxy
	↓
Network
	↓
Remote Object
```


**Spring Example**

==Feign Client==
```java
@FeignClient(name = "inventory-service")
public interface InventoryClient {   
	@GetMapping("/inventory/{id}")  
	InventoryResponse getInventory(Long id);
}
```

==Spring generates==
```
InventoryClient Proxy
  ↓
HTTP
  ↓
Inventory Service
```

##### 4- Logging Proxy
You want to record every request. Without changing the original class.

```
Client
	↓
Logging Proxy
	↓
Write Log
	↓
Real Object
```

**Spring Example**
Spring AOP `@Before`

##### 5- Caching Proxy
This is when you need to cache results of client requests and manage the life cycle of this cache, especially if **results are quite large**. The proxy can implement caching for recurring requests that always **yield the same results**. 

Example
`getProduct(15)` this Database query takes `500 ms` and if `10,000` users request product 15

![[Pasted image 20260626212945.png|465]]

**Java example** `productService.getProduct(10);`
* First call -> `Database`
* Second call -> `Cache`

**Spring Example**
```java
@Cacheable("products")
public Product getProduct(Long id)
```

##### 6- Smart Reference Proxy
This is when you need to be able to dismiss a heavyweight object once there are no clients that use it.

The proxy wants to know
- Who is using the object?
- Can it be deleted?
- Has it changed?

Imagine a large document 100 MB. Many users open it.
```
User A, User B, User C
	↓
Proxy
	↓
Document
```
Proxy keeps a count `3 references` and decrement when a user close, when the counter is `Zero` the proxy decide to free up the memory.

> The proxy also can track object modifications, if the object is not modified, so it reuse the same object instead of creating new instance.

**Example**
**HikariCP Connection Pool** (tracks connection usage and returns connections to the pool instead of destroying them)

---
#### When to use proxy ?
* Use the **Proxy pattern** when you need to control access to an original object to manage its creation, restrict access, or add supplementary operations without altering the original code. 

* Skip it when direct access to the object is sufficient, as it introduces unnecessary complexity and execution overhead. 
---
#### Summary

| Ask Yourself...                                                         | Pattern       |
| ----------------------------------------------------------------------- | ------------- |
| Do I need to control access to an object?                           | Proxy     |
| Do I need to add new behavior to an object?                         | Decorator |
| Do I need to make incompatible interfaces work together?            | Adapter   |
| Do I need to provide one simple entry point to a complex subsystem? | Facade    |
