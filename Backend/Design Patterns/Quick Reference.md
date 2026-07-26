
## Table of Contents

- [[#Creational Patterns]]
- [[#Structural Patterns]]
- [[#Behavioral Patterns]]
- [[#One-Page Cheat Sheet]]

---

## Creational Patterns

_Concerned with **how objects get created**._

### Singleton

**Intent:** Ensure a class has only **one instance**, with a global access point to it.

```java
public class ConfigManager {
    private static final ConfigManager INSTANCE = new ConfigManager();

    private ConfigManager() {}                 // private constructor

    public static ConfigManager getInstance() {
        return INSTANCE;
    }
}
```

|||
|---|---|
|**Use when**|Exactly one instance must coordinate actions across the system (config, connection pools, loggers)|
|**Watch out for**|Global mutable state, hidden dependencies, hard to unit test/mock|
|**In Spring**|Every bean is a Singleton **by scope**, by default — but managed by the container, not a private static field. Prefer letting Spring manage it over hand-rolling this pattern.|

---

### Factory (Factory Method)

**Intent:** Delegate object **creation** to a method/class, so the caller doesn't need to know the concrete type being instantiated.

```java
interface PaymentGateway { void pay(); }
class StripeGateway implements PaymentGateway { public void pay() {} }
class PaypalGateway implements PaymentGateway { public void pay() {} }

class PaymentGatewayFactory {
    static PaymentGateway create(String type) {
        return switch (type) {
            case "stripe" -> new StripeGateway();
            case "paypal" -> new PaypalGateway();
            default -> throw new IllegalArgumentException(type);
        };
    }
}
```

```java
PaymentGateway gateway = PaymentGatewayFactory.create("stripe");
```

|||
|---|---|
|**Use when**|The exact class to instantiate is decided at runtime (config, input, environment)|
|**Related**|**Abstract Factory** — a factory of factories, producing families of related objects|
|**In Spring**|`BeanFactory`/`ApplicationContext` itself is a giant Factory; `FactoryBean<T>` lets you plug in custom creation logic for a bean|

---

### Builder

**Intent:** Construct a complex object **step by step**, avoiding a constructor with a huge parameter list ("telescoping constructors").

```java
public class Pizza {
    private final String size;
    private final boolean cheese;
    private final boolean pepperoni;

    private Pizza(Builder b) {
        this.size = b.size;
        this.cheese = b.cheese;
        this.pepperoni = b.pepperoni;
    }

    public static class Builder {
        private String size;
        private boolean cheese;
        private boolean pepperoni;

        public Builder size(String size) { this.size = size; return this; }
        public Builder cheese(boolean v) { this.cheese = v; return this; }
        public Builder pepperoni(boolean v) { this.pepperoni = v; return this; }
        public Pizza build() { return new Pizza(this); }
    }
}
```

```java
Pizza pizza = new Pizza.Builder()
        .size("Large")
        .cheese(true)
        .pepperoni(true)
        .build();
```

|||
|---|---|
|**Use when**|An object has many optional fields/parameters, and you want a readable, chainable construction API|
|**In Spring / Java**|`UriComponentsBuilder`, `ResponseEntity.ok().header(...).body(...)`, Lombok's `@Builder`, Java records with a builder wrapper|

---

## Structural Patterns

_Concerned with **how objects/classes are composed** into larger structures._

### Adapter

**Intent:** Convert the interface of an existing class into another interface clients expect — lets incompatible interfaces work together.

```
Client ──► TargetInterface ◄── Adapter ──► Adaptee (existing/incompatible class)
```

```java
// Existing class with an incompatible interface
class LegacyPrinter {
    void printLegacy(String text) { System.out.println("[legacy] " + text); }
}

// Target interface the client expects
interface ModernPrinter { void print(String text); }

// Adapter bridges the two
class PrinterAdapter implements ModernPrinter {
    private final LegacyPrinter legacy = new LegacyPrinter();
    public void print(String text) { legacy.printLegacy(text); }
}
```

|||
|---|---|
|**Use when**|You need to use an existing class but its interface doesn't match what your code expects (often 3rd-party/legacy code you can't modify)|
|**In Spring**|`HandlerAdapter` in Spring MVC —  adapts different handler types to a uniform invocation interface|

---

### Facade

**Intent:** Provide a single, simplified interface over a complex set of subsystems/classes.

```
Client ──► Facade ──► SubsystemA
                  ├──► SubsystemB
                  └──► SubsystemC
```

```java
class OrderFacade {
    private final InventoryService inventory = new InventoryService();
    private final PaymentService payment = new PaymentService();
    private final ShippingService shipping = new ShippingService();

    public void placeOrder(Order order) {
        inventory.reserve(order);
        payment.charge(order);
        shipping.schedule(order);
    }
}
```

```java
new OrderFacade().placeOrder(order); // client doesn't touch 3 subsystems directly
```

|||
|---|---|
|**Use when**|A subsystem is complex, and most clients only need a simple, common-case entry point|
|**Difference from Adapter**|Adapter makes **one** interface compatible with another; Facade **simplifies access to many** classes at once|
|**In Spring**|A `@Service` orchestrating multiple repositories/other services is effectively a Facade over the data layer|

---

### Decorator

**Intent:** Attach additional responsibilities to an object **dynamically**, without altering its class or affecting other instances.

```
Client ──► Decorator ──► Decorator ──► Component
```

```java
interface Coffee { double cost(); }

class SimpleCoffee implements Coffee {
    public double cost() { return 2.0; }
}

abstract class CoffeeDecorator implements Coffee {
    protected final Coffee delegate;
    CoffeeDecorator(Coffee delegate) { this.delegate = delegate; }
}

class MilkDecorator extends CoffeeDecorator {
    MilkDecorator(Coffee c) { super(c); }
    public double cost() { return delegate.cost() + 0.5; }
}

class SugarDecorator extends CoffeeDecorator {
    SugarDecorator(Coffee c) { super(c); }
    public double cost() { return delegate.cost() + 0.2; }
}
```

```java
Coffee order = new SugarDecorator(new MilkDecorator(new SimpleCoffee()));
order.cost(); // 2.0 + 0.5 + 0.2 = 2.7 — layers stack at runtime
```

|||
|---|---|
|**Use when**|You need to add behavior to individual objects at runtime, and subclassing every combination would explode (`MilkSugarCoffee`, `MilkCoffee`, `SugarCoffee`, ...)|
|**In Spring**|AOP proxies are conceptually decorators — they wrap a target bean and add behavior (logging, transactions) transparently. `BufferedReader(new FileReader(...))` in the JDK is the classic textbook example.|

---

## Behavioral Patterns

_Concerned with **how objects interact and distribute responsibility**._

### Strategy

**Intent:** Define a **family of interchangeable algorithms**, encapsulate each one, and select between them at runtime.

```java
interface DiscountStrategy { double apply(double price); }

class NoDiscount implements DiscountStrategy {
    public double apply(double price) { return price; }
}
class PercentageDiscount implements DiscountStrategy {
    private final double percent;
    PercentageDiscount(double percent) { this.percent = percent; }
    public double apply(double price) { return price * (1 - percent / 100); }
}

class Checkout {
    private DiscountStrategy strategy;
    Checkout(DiscountStrategy strategy) { this.strategy = strategy; }
    double total(double price) { return strategy.apply(price); }
}
```

```java
Checkout checkout = new Checkout(new PercentageDiscount(10));
checkout.total(100); // 90.0 — swap the strategy object to change behavior entirely
```

|||
|---|---|
|**Use when**|You have several ways to do the same job (sorting, pricing, validation) and want to pick/swap the algorithm without `if`/`switch` sprawl|
|**In Spring**|Injecting different implementations of an interface via Autowiring & @Qualifier / @Primary|

---

### Observer

**Intent:** Define a one-to-many dependency so that when one object (the **Subject**) changes state, all its dependents (**Observers**) are notified automatically.

```
Subject ──notifies──► Observer 1
        ├─notifies──► Observer 2
        └─notifies──► Observer 3
```

```java
interface OrderObserver { void onOrderPlaced(Order order); }

class OrderSubject {
    private final List<OrderObserver> observers = new ArrayList<>();
    void subscribe(OrderObserver o) { observers.add(o); }
    void placeOrder(Order order) {
        observers.forEach(o -> o.onOrderPlaced(order));
    }
}
```

|||
|---|---|
|**Use when**|Multiple parts of the system need to react to an event without the event source knowing about them directly|
|**In Spring**|`ApplicationEventPublisher` + `@EventListener` is Observer, built into the container

```java
@Component
class OrderPlacedListener {
    @EventListener
    public void handle(OrderPlacedEvent event) { ... }
}
```

---

### Command

**Intent:** Encapsulate a request/action as a standalone **object**, so it can be passed around, queued, logged, or undone.

```java
interface Command { void execute(); }

class PlaceOrderCommand implements Command {
    private final OrderService service;
    private final Order order;
    PlaceOrderCommand(OrderService service, Order order) {
        this.service = service; this.order = order;
    }
    public void execute() { service.place(order); }
}

class CommandInvoker {
    private final Deque<Command> history = new ArrayDeque<>();
    void run(Command cmd) { cmd.execute(); history.push(cmd); }
}
```

|||
|---|---|
|**Use when**|You need to queue actions, support undo/redo, log requests, or decouple the invoker from the object that performs the action|
|**In Spring**|Spring's `Runnable`/`Callable` tasks submitted to a `TaskExecutor` (`@Async`) are Commands in spirit; Spring Batch `Tasklet`s follow the same idea|

---

### Template Method

**Intent:** Define the **skeleton** of an algorithm in a base class, letting subclasses override specific steps without changing the overall structure.

```java
abstract class DataExporter {
    // Template method — the fixed algorithm skeleton
    public final void export() {
        fetchData();
        transformData();
        writeOutput();
    }

    protected abstract void fetchData();
    protected abstract void transformData();
    protected void writeOutput() { System.out.println("Writing output..."); } // default step
}

class CsvExporter extends DataExporter {
    protected void fetchData() { System.out.println("Fetching for CSV"); }
    protected void transformData() { System.out.println("Transform to CSV rows"); }
}
```

|||
|---|---|
|**Use when**|Several classes follow the **same overall process**, but differ in specific steps|
|**In Spring**|`JdbcTemplate`, `RestTemplate`, `TransactionTemplate` — the whole "`XxxTemplate`" naming convention in Spring is a direct nod to this pattern: fixed workflow (open connection → execute → handle exceptions → close), with the variable part (your SQL/callback) plugged in|

---

## One-Page Cheat Sheet

|Pattern|Category|One-Line Intent|Spring Example|
|---|---|---|---|
|Singleton|Creational|Only one instance, globally accessible|Default bean scope|
|Factory|Creational|Delegate object creation to hide concrete types|`BeanFactory`, `FactoryBean<T>`|
|Builder|Creational|Step-by-step construction of complex objects|`UriComponentsBuilder`, Lombok `@Builder`|
|Adapter|Structural|Make an incompatible interface fit what's expected|`HandlerAdapter`|
|Facade|Structural|One simple interface over a complex subsystem|A `@Service` orchestrating several repos|
|Decorator|Structural|Add behavior dynamically by wrapping an object|AOP proxies|
|Strategy|Behavioral|Swap interchangeable algorithms at runtime|`@Qualifier`-selected bean implementations|
|Observer|Behavioral|Notify many dependents when one subject changes|`ApplicationEventPublisher` + `@EventListener`|
|Command|Behavioral|Encapsulate a request as an object|`@Async` tasks, Spring Batch `Tasklet`|
|Template Method|Behavioral|Fixed algorithm skeleton, pluggable steps|`JdbcTemplate`, `RestTemplate`|

---

## Related Notes

- [[Spring-Framework-Notes|Spring Framework (Core)]]
- [[Spring-MVC-Notes|Spring MVC]]
- [[Spring-AOP-Notes|Spring AOP]]
- [[Microservices-Notes|Microservices Architecture]]