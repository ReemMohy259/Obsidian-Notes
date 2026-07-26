## Table of Contents

- [[#1. What Is an Interface?]]
- [[#2. Rules for Creating Interfaces]]
- [[#3. What Interfaces Achieve]]
- [[#4. Default Methods]]
- [[#5. Static Methods in Interfaces]]
- [[#6. Private Methods in Interfaces (Java 9+)]]
- [[#7. Interface Inheritance Rules]]
- [[#8. Interface (with default methods) vs Abstract Class]]
- [[#9. When to Use Interface vs Abstract Class]]
- [[#10. Marker Interfaces]]
- [[#11. Inner Interfaces]]
- [[#12. Functional Interfaces (Brief)]]
- [[#13. Java Packages]]
- [[#14. Quick Revision Cheat Sheet]]

---

## 1. What Is an Interface?

An **interface** is an abstract type — a contract of method signatures (and constants) that implementing classes must fulfill. It's Java's primary mechanism for **abstraction**, **polymorphism**, and (since classes can implement multiple interfaces) a form of **multiple inheritance**.

```java
public interface Electronic {

    String LED = "LED";                 // constant (implicitly public static final)

    int getElectricityUse();             // abstract method

    static boolean isEnergyEfficient(String type) {   // static method
        return type.equals(LED);
    }

    default void printDescription() {    // default method
        System.out.println("Electronic Description");
    }
}
```

```java
public class Computer implements Electronic {
    @Override
    public int getElectricityUse() { return 1000; }
}
```

A class opts in via `implements`.

---

## 2. Rules for Creating Interfaces

An interface may contain:

- Constant variables
- Abstract methods
- Static methods
- Default methods

And the constraints to remember:

|Rule|Detail|
|---|---|
|Can't instantiate directly|No `new SomeInterface()`|
|Can be empty|Zero methods/variables is valid|
|No `final` on the interface itself|Compiler error — contradicts the idea of something meant to be implemented|
|Declared `public` or package-private|The compiler auto-adds `abstract` to the interface declaration|
|Methods can't be `protected` or `final`|Would conflict with the override contract|
|Private methods|**Not allowed before Java 9** — introduced in Java 9, see [[#6. Private Methods in Interfaces (Java 9+)]]|
|Interface variables|Always implicitly `public static final` — visibility can't be changed|

---

## 3. What Interfaces Achieve

### 3.1. Behavioral Contracts for Unrelated Classes

Interfaces let unrelated classes share a common capability. `Comparable`, `Comparator`, and `Cloneable` are classic JDK examples:

```java
public class EmployeeSalaryComparator implements Comparator<Employee> {
    @Override
    public int compare(Employee a, Employee b) {
        return Double.compare(a.getSalary(), b.getSalary());
    }
}
```

### 3.2. Multiple Inheritance

Java classes support only **single** class inheritance, but a class can `implement` **several** interfaces, inheriting each one's abstract method contracts:

```java
public interface Fly { void fly(); }
public interface Transform { void transform(); }

public class Car implements Fly, Transform {
    @Override public void fly() { System.out.println("I can Fly!!"); }
    @Override public void transform() { System.out.println("I can Transform!!"); }
}
```

### 3.3. Polymorphism

An interface type reference can point to any implementing class, and the correct overridden method runs at runtime based on the actual object type:

```java
public interface Shape { String name(); }
public class Circle implements Shape { public String name() { return "Circle"; } }
public class Square implements Shape { public String name() { return "Square"; } }
```

```java
List<Shape> shapes = List.of(new Circle(), new Square());
for (Shape shape : shapes) {
    System.out.println(shape.name());   // resolves to the ACTUAL object's implementation
}
```

---

## 4. Default Methods

### Why They Exist

Before Java 8, adding a new abstract method to an existing interface would **break every implementing class** — none of them would compile until updated. **Default methods** solve this: they carry a body, so existing implementers don't need to change anything, keeping the design backward-compatible. This was the direct motivation for retrofitting lambda-friendly methods onto the Collections Framework without breaking the Java ecosystem.

```java
public interface Vehicle {
    String getBrand();
    String speedUp();
    String slowDown();

    default String turnAlarmOn() { return "Turning the vehicle alarm on."; }
    default String turnAlarmOff() { return "Turning the vehicle alarm off."; }
}
```

A class implementing `Vehicle` automatically gets `turnAlarmOn()`/`turnAlarmOff()` for free, without providing its own implementation — unless it chooses to override them.

### Default Methods Can't Access Object State

This is the **most important distinction** from abstract classes: a default method has no fields to read, because interfaces have no instance state. Any logic inside a default method must be expressed purely in terms of the interface's **other methods**:

```java
public interface CircleInterface {
    List<String> allowedColors = List.of("RED", "GREEN", "BLUE");

    String getColor();                                 // abstract — supplies the "state"

    default boolean isValid() {
        return allowedColors.contains(getColor());       // default method calls the abstract method
    }
}
```

The implementing class supplies actual state by overriding `getColor()`; the default method `isValid()` never touches a field directly — it only ever calls other interface methods.

---

## 5. Static Methods in Interfaces

Static interface methods are utility methods that don't belong to any particular instance, called via the **interface name**, not through an object reference:

```java
public interface Vehicle {
    static int getHorsePower(int rpm, int torque) {
        return (rpm * torque) / 5252;
    }
}
```

```java
Vehicle.getHorsePower(2500, 480);
```

**Purpose**: group related utility logic directly on the type it's about, instead of creating an artificial `VehicleUtils` class that exists purely to hold static methods.

---

## 6. Private Methods in Interfaces (Java 9+)

Before Java 9, every method in an interface had to be effectively public (abstract, static, or default). **Java 9 added private interface methods**, letting default/static methods **share common code** without exposing that code as part of the public API.

### Private Instance Method (usable from default methods)

```java
public interface Foo {
    default void bar() {
        System.out.print("Hello");
        baz();                    // calling the private helper
    }

    private void baz() {
        System.out.println(" world!");
    }
}
```

### Private Static Method (usable from static methods)

```java
public interface Foo {
    static void buzz() {
        System.out.print("Hello");
        staticBaz();
    }

    private static void staticBaz() {
        System.out.println(" static world!");
    }
}
```

### Benefits

- **Encapsulation** — hides implementation detail from implementing classes.
- **Reduces duplication** — shared logic used by multiple default/static methods lives in one private method instead of being copy-pasted.

---

## 7. Interface Inheritance Rules

### 7.1. Interface Extending Another Interface

```java
public interface HasColor {
    String getColor();
}

public interface Box extends HasColor {
    int getHeight();
}
```

`Box` inherits `getColor()` from `HasColor`, so it now effectively declares **two** abstract methods. Any concrete class implementing `Box` must provide **both**.

### 7.2. Abstract Class Implementing an Interface

```java
public interface Transform {
    void transform();
    default void printSpecs() { System.out.println("Transform Specification"); }
}

public abstract class Vehicle implements Transform { }
```

`Vehicle` inherits both the abstract `transform()` and the default `printSpecs()` — and, being abstract itself, isn't forced to implement `transform()` right away; that obligation passes down to its own concrete subclasses.

### 7.3. The Diamond Problem — Multiple Default Method Conflicts

When a class implements two interfaces that define the **same default method signature**, the compiler can't pick one for you — it's ambiguous:

```java
public interface Alarm {
    default String turnAlarmOn() { return "Turning the alarm on."; }
}
// Vehicle ALSO defines turnAlarmOn() as a default method (from section 4)

public class Car implements Vehicle, Alarm {
    // WON'T COMPILE without an explicit override — ambiguous inheritance
}
```

You **must** resolve it explicitly:

```java
public class Car implements Vehicle, Alarm {

    @Override
    public String turnAlarmOn() {
        return Vehicle.super.turnAlarmOn();       // explicitly pick Vehicle's version
    }
    // or Alarm.super.turnAlarmOn();
    // or combine both: Vehicle.super.turnAlarmOn() + " " + Alarm.super.turnAlarmOn();
    // or write a brand-new implementation entirely
}
```

The `InterfaceName.super.methodName()` syntax is the **only** way to explicitly call a specific interface's default method from within an overriding implementation.

---

## 8. Interface (with Default Methods) vs Abstract Class

Default methods narrowed the gap between interfaces and abstract classes, but real differences remain.

||Interface (with default methods)|Abstract Class|
|---|---|---|
|**State**|No instance fields; default methods can't read any object state directly — only call other interface methods|Can hold fields; non-abstract methods **can** read/use that state directly|
|**Constructors**|None — can't be instantiated even partially|Can have constructors, run when a subclass is instantiated|
|**Overriding `Object` methods** (`equals`, `toString`, ...)|Not possible|Possible|
|**Field access modifiers**|Fields are always `public static final`|Any access modifier, any combination|
|**Instance/static initializer blocks**|Not allowed|Allowed|
|**Lambda expressions**|A **functional interface** (single abstract method) can be implemented via a lambda|Cannot be implemented via a lambda, regardless of abstract method count|
|**Multiple inheritance**|A class can implement many|A class can extend only one|

> [!important] The single clearest distinguishing test Try to write a `default` method whose logic depends on "the current value of this object's field." **You can't** — because there is no field to read. If you find yourself wanting a method to read/use instance state directly, that's exactly the situation calling for an **abstract class**, not an interface. See the `CircleClass`/`CircleInterface` comparison in [[#4. Default Methods]] for exactly this contrast worked through in code.

### Practical Guidance

Given the choice, **prefer an interface with default methods when possible** — a class can only extend one abstract class, but it can implement many interfaces, so choosing an interface keeps that door open for the implementing class to also extend something else.

---

## 9. When to Use Interface vs Abstract Class

### Use an Interface When…

- The problem needs multiple inheritance across unrelated class hierarchies.
- Unrelated classes need to share a capability contract (`Comparable`, `Cloneable`) without being related by "is-a".
- You're defining a contract for third parties to implement, without caring how they implement it.
- **Test**: the relationship reads as **"A is capable of [doing this]."** — e.g. _"Sender is capable of sending a file."_

```java
public interface Sender {
    void send(File fileToBeSent);
}

public class ImageSender implements Sender {
    @Override
    public void send(File fileToBeSent) { /* image-sending logic */ }
}
```

### Use an Abstract Class When…

- You want to **share code** across several closely related classes via a common base.
- You have a partial implementation plus some behavior every subclass must still supply.
- Subclasses share several common fields/methods that need non-public visibility.
- You need mutable/non-static/non-final methods that modify object state.
- **Test**: the relationship reads as **"A is a B."** — e.g. _"Car is a Vehicle."_

```java
public abstract class Vehicle {
    protected abstract void start();
    protected abstract void stop();
    protected abstract void drive();
    // shared getters/setters/fields could live here too
}

public class Car extends Vehicle {
    @Override protected void start() { /* car-specific starting logic */ }
    @Override protected void stop() { /* car-specific stopping logic */ }
    @Override protected void drive() { /* car-specific driving logic */ }
}
```

### Shared Similarity Between Both

- Neither can be instantiated directly (`new SomeInterface()`/`new AbstractClass()` fail) — both require either a concrete subclass or an anonymous class.
- Both can mix declared-but-undefined methods (abstract methods) with defined ones (default/static methods in interfaces; regular instance methods in abstract classes).

---

## 10. Marker Interfaces

A **marker interface** (a.k.a. tagging interface) declares **no methods or constants at all** — its only purpose is to give the JVM/compiler run-time type information about objects that implement it.

```java
public interface Deletable {
}
```

### Well-Known JDK Examples

|Marker Interface|What Implementing It Signals|
|---|---|
|`Serializable`|`ObjectOutputStream.writeObject()` will actually serialize the object; without it, a `NotSerializableException` is thrown|
|`Cloneable`|`Object.clone()` is permitted on this type; without it, `CloneNotSupportedException` is thrown|
|`Remote`|Used in RMI to mark objects that can be invoked remotely|

### Custom Example

```java
public class Entity implements Deletable { /* ... */ }

public class ShapeDao {
    public boolean delete(Object object) {
        if (!(object instanceof Deletable)) {
            return false;
        }
        // deletion logic
        return true;
    }
}
```

Only objects implementing `Deletable` pass the check — the interface acts purely as a runtime tag checked via `instanceof`.

### Marker Interfaces vs Annotations

Both can flag a class for special handling, but there's one meaningful difference: because a marker **interface** participates in the normal Java type system, you can build **additional type restrictions** on top of it that annotations alone can't express:

```java
public interface Shape {
    double getArea();
    double getCircumference();
}

public interface DeletableShape extends Shape { }   // must ALSO be a Shape

public class Rectangle implements DeletableShape { /* ... */ }
```

Now every `DeletableShape` is _guaranteed_ to also be a `Shape` — a compile-time guarantee an annotation can't provide on its own. The trade-off cuts both ways though: this same polymorphism means **every subclass of `Rectangle` automatically inherits `DeletableShape` too**, whether or not that's actually appropriate — a hidden coupling annotations don't create.

> [!warning] Marker interfaces are a mild code smell today They blur what an "interface" conceptually represents (a behavior contract) since they define no behavior at all. Modern Java code generally favors **annotations** for pure metadata/tagging use cases, reserving marker interfaces mainly for cases (like `Serializable`/`Cloneable`) baked deep into the JDK's own type-checking, or where the type-restriction trick above is specifically useful.

### Why Not Just Use a Regular (Non-Marker) Interface as the Check?

You could check `instanceof Shape` directly instead of introducing `Deletable`. This breaks down once you need to allow **multiple, type-unrelated** things to be deletable (e.g. both `Shape` and `Person`): you'd either have to keep expanding an `if` chain (`instanceof Shape || instanceof Person || ...`) for every new type, or force unrelated types to implement a shared interface that doesn't semantically describe them (is a `Person` really a `Shape`? No) — both are worse designs than a dedicated, purpose-built marker interface.

---

## 11. Inner Interfaces

An **inner interface** is declared inside the body of another interface or class. Main motivations:

- **Namespacing** — avoids collisions when a natural interface name (like `List`) is already taken/common.
- **Encapsulation** — keeps two tightly related interfaces physically together.
- **Readability** — groups related types in one place.

The canonical JDK example is `Map.Entry`, declared inside `Map` so it doesn't pollute the global namespace and its relationship to `Map` is obvious from the name itself.

### Inside Another Interface

```java
public interface Customer {
    interface List {     // implicitly public and static
        // ...
    }
}
```

### Inside a Class

```java
public class Customer {
    public interface List {     // still implicitly static; access modifier is explicit here
        void add(Customer customer);
        String getCustomerNames();
    }
}
```

```java
public class CommaSeparatedCustomers implements Customer.List {
    // ...
}
```

Inner interfaces declared inside a **class** are also implicitly `static`, but (unlike inner interfaces declared inside another _interface_) they can carry an explicit access modifier constraining where they may be implemented.

---

## 12. Functional Interfaces (Brief)

A **functional interface** is any interface with **exactly one** abstract method (default/static methods don't count toward that limit) — this single-method constraint is what makes it usable as the target of a **lambda expression**.

Long-standing JDK examples predate Java 8 entirely: `Comparable` (since Java 1.2), `Runnable` (since Java 1.0). Java 8 added a standard library of general-purpose ones — `Predicate<T>`, `Consumer<T>`, `Function<T,R>`, and more — covered in depth in a dedicated functional-interfaces note. The `@FunctionalInterface` annotation is optional but recommended: it makes the compiler enforce the single-abstract-method constraint, catching accidental violations at compile time.

---

## 13. Java Packages

### Why Packages Exist

Packages group related classes, interfaces, and sub-packages together, giving three concrete benefits:

- **Discoverability** — logically related types live together.
- **Avoiding naming conflicts** — `com.baeldung.Application` and `com.example.Application` can coexist without clashing.
- **Access control** — combined with access modifiers, packages let you expose some types/members only within the package (package-private) rather than to the whole world.

### Declaring a Package

Must be the **very first line** of code in the file:

```java
package com.baeldung.packages;
```

### Naming Conventions

- All **lower case**.
- Period-delimited, mirroring a reversed company domain: `www.baeldung.com` → `com.baeldung` → further refined into `com.baeldung.packages`, `com.baeldung.packages.domain`, etc.
- Each package/sub-package corresponds to an actual **directory** on disk (`com/baeldung/packages/`).

### The Default (Unnamed) Package — Avoid It

Types not explicitly placed in a package land in the anonymous default package. This has real costs:

- No sub-packages possible, and no package structure to organize by.
- Can't be **imported** by classes in any named package.
- `protected` and package-private access scopes become meaningless, since there's no package boundary to enforce them.

The default package exists mainly for trivial/throwaway scripts — real applications should give every type an explicit package.

### Imports

```java
import com.baeldung.packages.domain.TodoItem;      // single type
import com.baeldung.packages.domain.*;               // whole package (wildcard)
```

JDK types are imported the same way:

```java
import java.util.ArrayList;
import java.util.List;
```

### Fully Qualified Names — Resolving Naming Collisions

When two imported types share a simple name (classically `java.sql.Date` vs `java.util.Date`), at least one usage needs its **fully qualified name**:

```java
private List<com.baeldung.packages.domain.TodoItem> todoItems;
```

### Compiling & Running Packaged Classes

```bash
javac com/baeldung/packages/domain/TodoItem.java
javac -classpath . com/baeldung/packages/*.java     # -classpath tells javac where dependent compiled classes are
java com.baeldung.packages.TodoApp                    # run using the fully qualified class name
```

Directory structure must exactly mirror the package declaration — `com.baeldung.packages.domain` → `com/baeldung/packages/domain/`.

---

## 14. Quick Revision Cheat Sheet

|Concept|Remember|
|---|---|
|Interface|Abstract type — method contract + constants, enables abstraction/polymorphism/multiple inheritance|
|Interface fields|Always implicitly `public static final`|
|Default method|Has a body, can't access object state, added in Java 8 for backward compatibility|
|Static interface method|Called via interface name, not an instance|
|Private interface method (Java 9+)|Shares code between default/static methods without exposing it publicly|
|Diamond problem|Two interfaces with the same default method → must resolve with `Interface.super.method()`|
|Interface vs Abstract Class (key test)|Can the method body read instance state? Interface: no. Abstract class: yes|
|"A is capable of X"|→ Interface|
|"A is a B"|→ Abstract class|
|Marker interface|No methods/constants — pure runtime type tag (`Serializable`, `Cloneable`)|
|Marker interface vs annotation|Interface gives compile-time type guarantees via polymorphism; annotation doesn't|
|Inner interface|Declared inside a class/interface for namespacing + encapsulation (`Map.Entry`)|
|Functional interface|Exactly one abstract method → usable as a lambda target|
|Package|Groups related types; avoids naming collisions; enables access control|
|Default package|Avoid in real projects — no imports, no sub-packages, breaks package-private/protected|
|Fully qualified name|Needed when two imported types share a simple name|

---

## Related Notes

- [[Java-String-Immutability-Notes|Java Core: String, Immutability & String Pool]]
- [[Design-Patterns-Quick-Reference|Design Patterns Quick Reference]] — Strategy/Observer rely heavily on interfaces
- [[Spring-Framework-Notes|Spring Framework (Core)]] — Spring's stereotype annotations are themselves meta-annotated interfaces-of-a-sort (annotation interfaces)