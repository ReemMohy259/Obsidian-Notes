## What Is RMI?

**Remote Method Invocation** — lets an object in one JVM call methods on an object in another JVM (possibly another machine) as if it were local.

```
Client JVM              Server JVM
Client Object --RMI--> Remote Object
     |  add(5,3)              |
     |----------------------->|
     |<-----------------------|
     |          8             |
```

---

## Setup

```java
public interface Calculator extends Remote {
    int add(int a, int b) throws RemoteException;
}
```

```java
public class CalculatorImpl extends UnicastRemoteObject implements Calculator {
    @Override
    public int add(int a, int b) { return a + b; }
}
```

Client just calls `calculator.add(5, 3)` — networking is fully hidden.

---

## Internal Flow

```
Client
  │  calculator.add(5,3)
  ▼
Stub  → serialize args → TCP/IP → Server
                                    │
                                 Skeleton (obsolete since JDK 1.2)
                                    │
                            CalculatorImpl.add()
                                    │
                          serialize result → back to client
```

---

## Stub — Client-Side Proxy

- What the client actually calls (`calculator` is really the stub).
- Serializes arguments, sends over network, waits, deserializes result, returns it.
- Conceptually a **Proxy** hiding all networking.

## Skeleton — Server-Side Dispatcher (Obsolete)

- Old job: receive request → deserialize args → call real object → serialize result → respond.
- **Since JDK 1.2**: no longer generated — RMI runtime dispatches server-side automatically.

> [!important] Interview trap Q: "Does Java still generate skeleton classes?" A: No — obsolete since JDK 1.2. RMI runtime handles server-side dispatch automatically. Stub still exists (often generated dynamically).

---

## Serialization Requirement

Arguments/return types crossing JVMs must implement `Serializable`, or:

```
java.io.NotSerializableException
```

---

## RMI Registry — Naming/Discovery Service

Maps a **logical name → remote reference (stub)**, like DNS. Stores the **stub**, never the actual object.

```
Server: new CalculatorImpl() → registry.rebind("CalculatorService", calculator)
                                        │
                              ┌────────────────────────────┐
                              │  RMI Registry              │
                              │ "CalculatorService" → Stub │
                              └────────────────────────────┘
                                        │
Client: registry.lookup("CalculatorService") → gets Stub → calculator.add(5,3)
```

### Key Methods

|Method|Behavior|
|---|---|
|`bind(name, obj)`|Register — **fails** if name exists|
|`rebind(name, obj)`|Register **or replace** — most common|
|`lookup(name)`|Returns the remote reference (stub)|
|`unbind(name)`|Removes the binding|

- **Default port**: `1099`
- Only involved at **lookup time** — after that, client talks directly to the object via the stub.

### Registry vs Stub

||Registry|Stub|
|---|---|---|
|Role|Naming, registration, discovery|Sending requests, receiving responses|
|Used|Once, at lookup|Every remote method call|

> [!tip] Analogy Registry = hotel reception ("here's the room number"). Stub = you walking to the room yourself, every time.

---

## Advantages / Disadvantages

| Advantages                       | Disadvantages                              |
| -------------------------------- | ------------------------------------------ |
| Simple remote calls, feels local | Java-only (client + server both need Java) |
| Pure Java                        | Slower than local calls                    |
| Hides networking                 | Firewall/network issues                    |
| Automatic serialization          | Tight coupling                             |
| Native distributed-app support   | Largely superseded by REST/gRPC            |

---

## RMI vs REST vs gRPC

||RMI|REST|gRPC|
|---|---|---|---|
|Language|Java only|Language-independent|Multi-language|
|Transport/format|Java serialization (binary)|HTTP + JSON/XML|HTTP/2 + Protocol Buffers|
|Coupling|Tight|Loose|Loose|
|Style|Method calls|Resource-based requests|Modern high-perf RPC|
|Typical use today|Legacy Java-to-Java|Public/web APIs|Cloud-native microservices|

---

