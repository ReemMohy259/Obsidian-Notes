## Table of Contents

- [[#1. What Is a Nested Class?]]
- [[#2. The Four Types at a Glance]]
- [[#3. Static Nested Classes]]
- [[#4. Inner Classes (Non-Static / Member Classes)]]
- [[#5. Local Classes]]
- [[#6. Anonymous Classes]]
- [[#7. Rules Comparison Table]]
- [[#8. How the Compiler Actually Represents Them]]
- [[#9. Real-World Applications]]
- [[#10. Common Pitfalls]]
---

## 1. What Is a Nested Class?

A **nested class** is any class defined **inside** another class or interface. The enclosing class is called the **outer class**.

```java
class Outer {
    class Nested {
        // nested class body
    }
}
```

### Why Nest a Class at All?

- **Logical grouping** — a helper class only makes sense in the context of its outer class (e.g. a linked list's `Node`), so nesting keeps that relationship explicit in the code's structure.
- **Increased encapsulation** — a nested class can be made `private`, hiding it entirely from the rest of the codebase; even a package-private top-level class is visible to the whole package, but a private nested class is visible **only** inside its outer class.
- **More readable, more maintainable code** — related classes live together instead of being scattered across separate files.
- **Access to the outer class's members** — (for non-static nested classes) direct access to the enclosing instance's fields and methods, including `private` ones, without needing getters.

---

## 2. The Four Types at a Glance

**Nested Classes**
- Static Nested Class        (declared with `static`, inside a class)
- Inner Class (non-static)   (declared without `static`, tied to an outer INSTANCE)
	 - Member Inner Class     (declared directly in the class body, like a field)
	 - Local Class            (declared inside a method/block body)
	 - Anonymous Class        (no name at all — declared and instantiated in one expression)


|Type|Tied To|Has a Name?|Typical Use|
|---|---|---|---|
|**Static Nested Class**|The outer **class** itself (no outer instance needed)|Yes|Helper/builder classes logically scoped to the outer class|
|**Inner Class (member)**|A specific outer **instance**|Yes|Needs live access to the enclosing object's state|
|**Local Class**|A method/constructor/block|Yes (but scoped to that block)|One-off helper used only within a single method|
|**Anonymous Class**|A single instantiation expression|No|Quick one-time implementation of an interface/abstract class|

---

## 3. Static Nested Classes

Declared with the `static` modifier, directly inside the outer class body.

```java
public class Outer {

    private static int outerStaticField = 10;
    private int outerInstanceField = 20;

    static class StaticNested {
        void display() {
            System.out.println(outerStaticField);   // OK — static members only
            // System.out.println(outerInstanceField);  // COMPILE ERROR — no outer instance exists
        }
    }
}
```

### Key Rules

- Behaves essentially like a **top-level class** that happens to be namespaced inside another class — it does **not** hold an implicit reference to any outer instance.
- Can access the outer class's **static** members directly (including `private static` ones), but **cannot** access instance (non-static) members of the outer class at all — there's no enclosing instance to read them from.
- Can be instantiated **without** first creating an outer instance:
    
    ```java
    Outer.StaticNested nested = new Outer.StaticNested();
    ```
    
- Can itself declare static members, unlike a (non-static) inner class.
- Access modifiers (`private`, `protected`, `public`, package-private) all work exactly as they would on any class member.

### When to Use It

Use a static nested class whenever the nested class **doesn't need** access to the outer instance's state — this is the **default/preferred choice** among the nested class types, precisely because it avoids holding an unnecessary hidden reference to an outer instance (which can otherwise silently leak memory — see [[#10. Common Pitfalls]]).

---

## 4. Inner Classes (Non-Static / Member Classes)

Declared **without** `static`, directly inside the outer class body — each instance of the inner class is implicitly bound to **one specific instance** of the outer class.

```java
public class Outer {

    private int outerInstanceField = 42;

    class Inner {
        void display() {
            System.out.println(outerInstanceField);   // OK — direct access to the enclosing instance's field
        }
    }
}
```

### Key Rules

- Holds an **implicit reference** to the enclosing outer instance — this is what enables direct access to the outer instance's fields/methods, including `private` ones.
- **Cannot** be instantiated without an outer instance. Two ways to create one from outside the outer class:

```java
Outer outer = new Outer();
Outer.Inner inner = outer.new Inner();// explicit outer-instance-qualified syntax
```

Or, more commonly, instantiated **from inside** a non-static outer method:
```java
public class Outer {    
	class Inner { }    
	void makeInner() {        
		Inner inner = new Inner(); // implicit — uses `this` as the enclosing               instance    
	}}
```
    
- **Cannot declare `static` members** itself (with the narrow exception of compile-time-constant `static final` fields) — since it's tied to an instance, it can't simultaneously host class-level (static) state.
- Referring to the outer instance explicitly, when shadowed by a same-named inner field, uses `Outer.this`:
    
```java
class Outer {    
	int x = 1;    
	class Inner {       
	 int x = 2;        
	 void show() {            
		System.out.println(x);         // 2 — Inner's own field                              System.out.println(Outer.this.x); // 1 — explicitly the outer instance's field   }    
	}}
```


### When to Use It

Use a (non-static) inner class specifically when it **must** interact with a **particular** outer instance's live state — e.g. an iterator that walks a specific collection instance's internal data.

---

## 5. Local Classes

A class declared **inside a method body, constructor, or any block** (including inside a loop or `if` block) — scoped exactly like a local variable.

```java
public class Outer {

    void processOrders(List<Order> orders) {

        class OrderValidator {           // local class — only exists within processOrders()
            boolean isValid(Order order) {
                return order.getTotal() > 0;
            }
        }

        OrderValidator validator = new OrderValidator();
        for (Order order : orders) {
            if (validator.isValid(order)) {
                // ...
            }
        }
    }
}
```

### Key Rules

- Visible/usable **only within the block it's declared in** — it doesn't exist as far as any code outside that method is concerned.
- Can access the enclosing method's **local variables and parameters**, but only if they're **effectively final** (never reassigned after initialization) — the compiler enforces this because the local class instance might outlive the method call itself (e.g. if returned or stored elsewhere), so it captures a frozen **copy** of those values rather than a live reference to the stack frame.
- If declared in an **instance** method, it also implicitly has access to the outer class's instance members (same as a regular inner class) — if declared in a **static** method, it behaves like it's in a static context, with no outer instance available.
- Cannot be declared with access modifiers (`public`, `private`, etc.) — that would be meaningless, since its scope is already fully determined by the enclosing block.
- Cannot declare `static` members (same restriction as inner classes), except compile-time constants.

### When to Use It

Rare in modern code — mostly superseded by lambdas and anonymous classes for one-off implementations. Still useful when you need a **named, multi-method** helper type that's genuinely local to one method's logic and would be noise anywhere else in the class.

---

## 6. Anonymous Classes

A class with **no name**, declared and instantiated **in a single expression** — always simultaneously extending a class or implementing an interface.

```java
Runnable task = new Runnable() {
    @Override
    public void run() {
        System.out.println("Running...");
    }
};
```

### Key Rules

- Can extend **exactly one** class **or** implement **exactly one** interface — never both, and never more than one of either.
- Follows the **same variable-capture rule as local classes**: can read effectively-final local variables/parameters from the enclosing scope, but can't reassign them.
- Cannot declare a constructor (it has no name to give one), but **can** use an **instance initializer block** to run setup logic:
    
    ```java
    Map<String, Integer> map = new HashMap<>() {{    put("a", 1);    put("b", 2);}};   // "double brace initialization" — an anonymous subclass + instance initializer
    ```
    
    > [!warning] Double brace initialization is generally discouraged It silently creates a new (anonymous) subclass of the collection just to run an initializer block — this has real costs: it captures a hidden reference to the enclosing instance (a memory-leak risk, see [[#10. Common Pitfalls]]), and produces a class only good for one-time use. Prefer `Map.of(...)`, `List.of(...)`, or a plain builder-style initialization instead.
    
- Every distinct anonymous class expression compiles to its **own** compiler-generated `.class` file (see [[#8. How the Compiler Actually Represents Them]]).
- Cannot have `static` members other than compile-time constants — same restriction as inner/local classes.

### When to Use It

The classic use case — providing a throwaway, one-off implementation of a single-method interface — is now often better expressed as a **lambda expression** if the target type is a functional interface:

```java
Runnable lambdaTask = () -> System.out.println("Running...");   // equivalent, much shorter
```

Anonymous classes remain necessary when:

- The type has **more than one** abstract method (not a functional interface), or
- You need to **extend a concrete/abstract class**, not just implement an interface (lambdas can only target functional interfaces), or
- You need **instance state** (fields) inside the throwaway implementation, which a lambda can't hold.

---

## 7. Rules Comparison Table

|Rule|Static Nested|Inner (Member)|Local|Anonymous|
|---|---|---|---|---|
|Needs an outer instance to create?|No|Yes|Depends on enclosing context|Depends on enclosing context|
|Access to outer **instance** members|❌|✅|✅ (if in an instance method)|✅ (if in an instance method)|
|Access to outer **static** members|✅|✅|✅|✅|
|Can declare `static` members|✅|❌ (except constants)|❌ (except constants)|❌ (except constants)|
|Has a name|✅|✅|✅|❌|
|Can have its own constructor|✅|✅|✅|❌|
|Can have access modifiers|✅|✅|❌|❌|
|Can extend a class AND implement an interface|✅ (normal class rules)|✅ (normal class rules)|✅ (normal class rules)|❌ — one or the other, never both|
|Captures enclosing local variables|N/A (no enclosing method)|N/A (tied to instance, not a method call)|✅ (effectively final only)|✅ (effectively final only)|

---

## 8. How the Compiler Actually Represents Them

There's no such thing as a "nested class" at the JVM bytecode level — the compiler flattens every nested class into its **own separate `.class` file**, using a `$` to encode the nesting relationship in the filename.

```
Outer.java
  └── produces:
        Outer.class
        Outer$StaticNested.class
        Outer$Inner.class
        Outer$1OrderValidator.class      (local class — numbered + named)
        Outer$1.class                     (anonymous class — just numbered)
```

- Non-static inner/local/anonymous classes get a **synthetic field** injected by the compiler (conventionally named something like `this$0`) holding the reference back to the enclosing outer instance — this is the actual mechanism behind "implicit access to the outer instance."
- This is also exactly **why** an inner/local/anonymous class instance can keep an outer instance alive longer than expected — see [[#10. Common Pitfalls]].

---

## 9. Real-World Applications

### 9.1. `Map.Entry` — Static Nested Interface (Inner Interfaces Recap)

```java
public interface Map<K, V> {
    interface Entry<K, V> {
        K getKey();
        V getValue();
    }
}
```

Namespaced as `Map.Entry` rather than a free-floating `Entry` type

### 9.2. Iterators — Non-Static Inner Classes

A classic textbook example: a custom collection's iterator needs live access to the **specific instance's** internal data structure, so it's implemented as a non-static inner class:

```java
public class LinkedList<T> implements Iterable<T> {

    private Node<T> head;

    private class Node<T> {           // static nested would ALSO work here, since Node
        T value;                       // doesn't need to read LinkedList's instance fields directly
        Node<T> next;
    }

    private class LinkedListIterator implements Iterator<T> {
        private Node<T> current = head;    // reads the ENCLOSING LinkedList instance's `head` directly

        public boolean hasNext() { return current != null; }
        public T next() {
            T value = current.value;
            current = current.next;
            return value;
        }
    }

    @Override
    public Iterator<T> iterator() {
        return new LinkedListIterator();
    }
}
```

### 9.3. Builder Pattern — Static Nested Class

 Design Patterns: Builder — the `Builder` doesn't need any of the outer object's instance state (there isn't one yet, since it's still being built), so it's a **static** nested class:

```java
public class Pizza {
    private final String size;

    private Pizza(Builder b) { this.size = b.size; }

    public static class Builder {          // static — no outer instance exists yet
        private String size;
        public Builder size(String size) { this.size = size; return this; }
        public Pizza build() { return new Pizza(this); }
    }
}
```

### 9.4. Event Listeners / Callbacks — Anonymous Classes

```java
button.addActionListener(new ActionListener() {
    @Override
    public void actionPerformed(ActionEvent e) {
        System.out.println("Button clicked!");
    }
});
```

The modern equivalent for single-method listener interfaces is usually a lambda, but this pattern is still common in older codebases and GUI frameworks (Swing, older Android).

### 9.5. Private Helper Classes for Encapsulation

```java
public class OrderProcessor {

    public void process(Order order) {
        ValidationResult result = new Validator().validate(order);
        // ...
    }

    private static class Validator {         // completely hidden from the rest of the codebase
        ValidationResult validate(Order order) { /* ... */ }
    }
}
```

`Validator` is entirely invisible outside `OrderProcessor` — stronger encapsulation than even a package-private top-level class would give.

### 9.6. Comparators — Anonymous Classes or Lambdas

```java
Collections.sort(employees, new Comparator<Employee>() {
    @Override
    public int compare(Employee a, Employee b) {
        return Double.compare(a.getSalary(), b.getSalary());
    }
});
// modern equivalent:
employees.sort((a, b) -> Double.compare(a.getSalary(), b.getSalary()));
```

Directly connects to [[Java-Interfaces-Packaging-Notes#3. What Interfaces Achieve|Comparator/Comparable]] from the interfaces note — historically implemented via anonymous classes before lambdas existed.

---

## 10. Common Pitfalls

### 10.1. Memory Leaks via Hidden Outer References

Because every non-static inner/local/anonymous class instance secretly holds a reference to its enclosing outer instance, **storing** such an instance somewhere long-lived (a static collection, a cache, an event-listener registry) can accidentally keep the **entire outer object** alive far longer than intended — a classic, hard-to-spot memory leak, especially common with anonymous inner classes used as listeners in long-running applications (a well-known historical issue in Android development, for instance).

```java
public class Activity {
    void registerLongLivedListener() {
        SomeStaticRegistry.register(new Listener() {     // anonymous class captures `this` Activity
            public void onEvent() { /* ... */ }
        });
        // This Activity instance can now never be garbage collected
        // as long as SomeStaticRegistry holds the listener!
    }
}
```

**Fix**: prefer a **static** nested class (or a static nested class taking an explicit weak reference to the outer instance) whenever the nested class doesn't actually need outer-instance access.

### 10.2. Forgetting "Effectively Final" for Captured Locals

```java
void process() {
    int counter = 0;
    Runnable r = () -> {
        counter++;              // COMPILE ERROR — counter isn't effectively final (it's reassigned above/below)
    };
    counter = 5;
}
```

Any local variable captured by a local/anonymous class or lambda must never be reassigned anywhere in the enclosing scope — not just "not reassigned after this point," but never reassigned at all after initialization.

### 10.3. Confusing `this` Inside a Nested Class

Inside a non-static inner class, unqualified `this` refers to the **inner** instance, not the outer one — a frequent source of confusion when a method needs the enclosing instance specifically. Use `Outer.this` (see [[#4. Inner Classes (Non-Static / Member Classes)]]) whenever you need to be explicit.

### 10.4. Trying to Give a Static Nested Class Instance-Only Behavior

```java
public class Outer {
    static class Nested {
        void show() {
            System.out.println(instanceField);   // COMPILE ERROR — no outer instance in scope
        }
    }
    int instanceField = 5;
}
```

If a "static nested class" needs outer instance data, it's not actually a candidate for `static` — either make it a proper (non-static) inner class, or explicitly pass the needed data in via its constructor instead of relying on implicit outer access.

---

