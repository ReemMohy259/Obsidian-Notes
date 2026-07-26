In Java Object-Oriented Programming (OOP), relationships between classes ==describe **how classes interact, communicate, or share properties with one another**==. These relationships are broadly divided into three main categories: **Inheritance (IS-A)**, **Association (HAS-A)**, and **Dependency (USES-A)**.

---
### 1. Inheritance (IS-A Relationship)

Inheritance allows a child class (subclass) to inherit fields and methods from a parent class (superclass). It helps achieve **code reusability** and enables **polymorphism**.
- **Mechanism**: Achieved using the `extends` keyword (for classes) or `implements` keyword (for interfaces).
- **Example**: A `Dog` **is an** `Animal`. 

```java
class Animal {
    void eat() { System.out.println("Eating..."); }
}

// Dog inherits from Animal
class Dog extends Animal {
    void bark() { System.out.println("Barking..."); }
}
```
---
### 2. Association (HAS-A Relationship)

**Association** is a structural relationship where one class contains a reference to an instance of another class as a field. It has two specialized sub-forms based on how strictly the lifecycles of the objects are bound:

#### A. Aggregation (Weak Association) 

The child object **can exist independently** of the parent object. If the parent object is destroyed, the child object survives

- **Example**: A `Professor` belongs to a `Department`. If the department closes down, the professor still exists. 

```Java
class Professor {
    String name;
    Professor(String name) { this.name = name; }
}

class Department {
    // Weak connection: Professor is created outside and passed in
    private Professor professor; 
    Department(Professor professor) { this.professor = professor; }
}
```

#### B. Composition (Strong Association)

The child object **cannot exist independently** of the parent object. The parent class manages and controls the lifecycle of the child object.

- **Example**: A `House` has a `Room`. If the house is destroyed, the rooms are destroyed too.

```Java
class Room {
    String type;
    Room(String type) { this.type = type; }
}

class House {
    private Room room;
    House() {
        // Strong connection: Room is created inside the House constructor
        this.room = new Room("Bedroom"); 
    }
}
```

---

### 3. Dependency (USES-A Relationship)

**Dependency** is a temporary and transient relationship. A class uses another class briefly, typically by receiving it as a method parameter, using it inside a local method scope, or calling its static utility methods.

- **Mechanism**: No long-term reference or instance variable is saved inside the class.
- **Example**: A `Driver` **uses a** `Car` temporarily to complete an action.

```Java
class Car {
    void startEngine() { System.out.println("Engine started."); }
}

class Driver {
    // Temporary use via a method parameter
    void drive(Car car) { 
        car.startEngine();
    }
}
```

---

### Summary

| Relationship Type | Common Label     | Java Implementation                          | Lifecycle Bond          | Coupling Strength |
| ----------------- | ---------------- | -------------------------------------------- | ----------------------- | ----------------- |
| **Inheritance**   | IS-A             | `extends` / `implements`                     | Tightly Coupled         | High              |
| **Association**   | HAS-A            | Generic member reference                     | Independent Lifecycles  | Low               |
| **Aggregation**   | HAS-A (Weak)     | Member object passed via constructor/setter  | Independent Lifecycles  | Medium-Low        |
| **Composition**   | PART-OF (Strong) | Member object instantiated within the parent | Dependent Lifecycles    | Medium-High       |
| **Dependency**    | USES-A           | Method parameter or local variable           | Transient / Short-lived | Low               |
