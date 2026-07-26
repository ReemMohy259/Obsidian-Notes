---
share_link: https://share.note.sx/or8ik1ms#JS0ucUKKUI3OsDQl+fp0HQ
share_updated: 2026-07-02T18:25:24+03:00
---

- **Behavioral design pattern:**
	Behavioral design patterns are **software solutions that focus on communication, collaboration, and the assignment of responsibilities between objects**. By defining how software components interact without tight coupling, they make systems more flexible, scalable, and easier to maintain.

### Problem with Naive Solutions
###### i- Use case #1 
Suppose you want to implement a **text-editor** application, and you want to control the **buttons**, these buttons may be (e.g. copy, paste, cut, highlight)

While all of these buttons **look similar**, ==they’re all supposed to do different things==

![[Pasted image 20260702155811.png|429]]

###### ii- Use case #2
You are developing a **smart home mobile application**, which provides the following functionalities: 
- turn lights on and off
- lock and unlock front door
- turn TV on and off. 

These functionalities need to be provided through the **mobile application as well as voice assistant and some shortcuts.**

![[Pasted image 20260702161559.png|658]]

* **SOLID Violation**

---
### Enhanced Solution
Good software design is often based on the **principle of separation of concerns**, which usually results in breaking an app into **layers**.

![[Pasted image 20260702160555.png|428]]

##### Definition 
**Command** is a behavioral design pattern that ==turns a request into a stand-alone object== that contains all information about the request. This transformation lets you pass requests as a method arguments, delay or queue a request’s execution, and support undoable operations.

* A good indicator is when your application has **many actions that share a common execution mechanism**.

* In all of these cases, the **action itself** can be represented as a command object, making the code easier to extend and keeping the component that triggers the action independent from the component that performs it.

#### Structure

![[Pasted image 20260702163336.png|659]]

###### Participants
* Command
	declares an interface for executing an operation. 
	
* ConcreteCommand (PasteCommand, OpenCommand) 
	- defines a binding between a Receiver object and an action. 
	- implements Execute by invoking the corresponding operation(s) on Receiver.
	
* Client (Application)
	creates a ConcreteCommand object and sets its receiver. 

* Invoker (Menultem)
	asks the command to carry out the request.  

* Receiver (Document ,Application) 
	Knows how to perform the operations associated with carrying out a request. Any class may serve as a Receiver.

##### Smart Home Solution

![[Pasted image 20260702172205.png|654]]

---
### Real Examples

#### 1. Java `Runnable` (Classic Command Pattern Example)
`Runnable` encapsulates a task (command) as an object. The `Thread` (invoker) executes the task without knowing what it does.

##### Roles

| Pattern Role         | Java Class                         |
| -------------------- | ---------------------------------- |
| **Command**          | `Runnable`                         |
| **Concrete Command** | Lambda / `Runnable` implementation |
| **Invoker**          | `Thread`                           |
| **Receiver**         | Business logic inside `run()`      |

##### Example
```java
// Concrete Command
Runnable task = () -> {
    // Receiver's business logic
    System.out.println("Sending welcome email...");
};

// Invoker
Thread thread = new Thread(task);

// Invoker calls task.run() internally
thread.start();
```

> **Key Idea:** `Thread` doesn't know _what_ the task does, it simply executes the `Runnable` command.


#### 2. Spring `TransactionTemplate`
`TransactionTemplate` encapsulates business logic (command) and executes it within a transaction.

##### Roles

| Pattern Role         | Spring Class                   |
| -------------------- | ------------------------------ |
| **Command**          | `TransactionCallback` (Lambda) |
| **Concrete Command** | Lambda passed to `execute()`   |
| **Invoker**          | `TransactionTemplate`          |
| **Receiver**         | Repository/Service methods     |

##### Example
```java
TransactionTemplate transactionTemplate =
	new TransactionTemplate(transactionManager);

// Invoker
transactionTemplate.execute(              // <-- Invoker executes the command

    status -> {                           // <-- Concrete Command

        accountRepository.save(account);  // <-- Receiver (business logic)
        auditRepository.save(log);

        return null;
    }
);
```

> **Key Idea:** `TransactionTemplate` manages the transaction lifecycle (begin, commit, rollback). It doesn't know the business logic, it simply executes the supplied command inside a transaction.

---
### When to use
- When **multiple actions share the same execution mechanism** (e.g., smart home remote, toolbar buttons).
- When you want to **decouple the invoker** (button, controller, scheduler) **from the receiver** (service, device, business logic).
- When actions need to be **queued** for later execution.
- When actions need to be **scheduled** (e.g., cron jobs, backups).
- When you need **Undo/Redo** functionality (reversable).
- When you need **macro commands** (one command executes multiple commands).
---
### PROS and CONS
##### Pros
- Loose coupling between invoker and receiver.
- Follows the **Open/Closed Principle (OCP)**—add new commands without modifying existing code.
- Follows the **Single Responsibility Principle (SRP)**—each command encapsulates one action.
- Supports **Undo/Redo**.
- Supports **task queues** and **asynchronous execution**.
- Supports **scheduling**.
- Enables **macro (composite) commands**,  assemble a set of simple commands into a complex one.
- Improves code organization and testability.

##### Cons
- Increases the number of classes.
- Adds an extra layer of abstraction (more indirection).
- Can be verbose for simple operations.
- Makes navigation/debugging slightly more complex.

---
### Related Patterns

#### 1. Chain of Responsibility vs Command
```text
Chain of Responsibility

Request
   │
   ▼
Handler A ──► Handler B ──► Handler C
                 │
                 ▼
              Handles Request
```

```text
Command

Invoker
   │
   ▼
Command
   │
   ▼
Receiver
```

##### Relationship
- **Chain of Responsibility (CoR)** passes a request through a chain of handlers until one handles it.
- **Command** encapsulates a request/action as an object and sends it directly to a receiver.
##### Notes
- A **handler in a CoR** can itself be implemented as a **Command**.
- Alternatively, the **request passed through the chain can be a Command object**, allowing the same command to be executed in different contexts.

---
#### 2. Command + Memento (Undo)

```text
Execute Command
       │
       ▼
Save State (Memento)
       │
       ▼
Perform Action
       │
       ▼
Undo
       │
       ▼
Restore Memento
```

##### Notes
- **Command** performs an action.
- **Memento** stores the object's previous state.
- Used together to implement **Undo/Redo** functionality.

---
#### 3. Command vs Strategy

```text
Strategy

Context
   │
   ▼
Strategy
 ├── PayPal
 ├── Stripe
 └── Credit Card
```

```text
Command

Invoker
   │
   ▼
Command
 ├── DeleteFile
 ├── SendEmail
 └── BackupDatabase
```

##### Notes
- **Command:** Different actions. -> Focuses on **what to do**
- **Strategy:** Different ways of performing the same action. -> Focuses on **how to do it**
