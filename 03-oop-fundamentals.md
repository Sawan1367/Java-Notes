# 03 — Object-Oriented Programming Fundamentals

## Why OOP came to exist (the historical problem)

Before OOP, "procedural" languages (C, early BASIC) organized programs as: global data + functions that operate on that data, largely disconnected from each other. As programs grew (thousands, then millions of lines), this caused real problems:
- Any function could modify ANY global data — hard to trace bugs, no ownership boundaries
- Duplicated logic across the codebase — no natural way to say "this data + behavior belong together"
- Hard to model real-world complex systems (a bank has accounts, customers, transactions — all interrelated) using just flat functions and data

**OOP's core idea**: bundle DATA and the BEHAVIOR that operates on that data together into one unit — an "object" — mirroring how real-world things naturally work (a car HAS properties like speed/color and BEHAVIORS like accelerate/brake, bundled together, not separate). Java was designed OOP-first from day one (unlike C++ which bolted OOP onto C) — object orientation is central to the language philosophy, not optional.

---

## Class & Object

**Class** = blueprint/template, defines what data (fields) and behavior (methods) objects of this type will have.
**Object** = actual instance created from that blueprint, with its own real data.

```java
class Car {
    String color;
    String model;
    int speed;

    void accelerate() {
        speed += 10;
        System.out.println(model + " speed is now " + speed);
    }
}
Car car1 = new Car();
car1.model = "Tesla";
car1.accelerate(); // Tesla speed is now 10
```
Why separate class and object: reusability. Write the blueprint ONCE, create as many independent objects as needed, each with its own separate data (`car1` and `car2` don't interfere with each other).

### Constructor — why it exists
Problem: objects created with `new` start with default/empty values (0, null, false) — often invalid/unusable state. Constructor is a special method that runs AUTOMATICALLY at creation time, guaranteeing the object starts in a valid, properly-initialized state — impossible to forget, unlike calling a separate "init()" method manually.
```java
class Car {
    String model;
    Car(String model) {         // constructor
        this.model = model;     // 'this' disambiguates field from parameter of same name
    }
}
```

### `this` keyword — why it's needed beyond disambiguation
Beyond resolving name clashes (above), `this` refers to "the current object instance" generally — useful for passing the current object to another method/constructor (`someMethod(this)`), or for **constructor chaining**: one constructor calling ANOTHER constructor of the same class, to avoid duplicating initialization logic across overloaded constructors.
```java
class Car {
    String model;
    int speed;
    Car(String model) {
        this(model, 0);       // calls the two-arg constructor below, must be the FIRST line
    }
    Car(String model, int speed) {
        this.model = model;
        this.speed = speed;
    }
}
```

### Instance vs Static — why both exist
- **Instance members**: each object needs its OWN independent copy (every car has its own speed).
- **Static members**: sometimes data/behavior belongs to the CONCEPT itself, not any one instance (e.g. "how many cars have been created total" — doesn't belong to any single car, belongs to the Car class as a whole).
```java
class Counter {
    static int count = 0;   // ONE shared copy across all objects
    Counter() { count++; }
}
```
Static exists to avoid the awkwardness of needing an arbitrary "extra" object just to hold shared/class-wide data.

### Nested & inner classes — why they exist
Sometimes a class only makes conceptual sense as a HELPER tied tightly to one outer class (e.g. a `Node` class that only ever exists inside a `LinkedList` implementation). Nesting it inside the outer class communicates that tight coupling directly in the code's structure, and keeps the outer class's namespace uncluttered.
```java
class LinkedList {
    static class Node {           // static nested class: doesn't need an outer instance to exist
        int value;
        Node next;
    }
}
```
A **non-static inner class** (no `static` keyword) additionally holds an implicit reference to its enclosing instance — used when the inner class genuinely needs access to the outer object's instance data, at the cost of that extra hidden coupling.

---

## The Four Pillars — why each exists (real historical motivation)

### 1. Encapsulation
**Problem it solves**: in procedural code, any part of the program could directly read/modify any data, in any invalid way (e.g. setting a bank balance to -9999 directly). No way to enforce rules around data changes.
**Solution**: hide fields (`private`), expose controlled access via methods (getters/setters) that can validate/enforce rules.
```java
class BankAccount {
    private double balance;
    public void deposit(double amount) {
        if (amount > 0) balance += amount;   // rule enforced HERE, impossible to bypass
    }
    public double getBalance() { return balance; }
}
```
Real benefit beyond "hiding for hiding's sake": you can change the INTERNAL implementation later (e.g. switch from a `double` to a more precise `BigDecimal`) without breaking any code that USES the class through its public methods — internal changes stay internal.

### 2. Inheritance
**Problem it solves**: without it, related classes (Dog, Cat, Bird — all Animals) would duplicate identical code (name, age, eat() method) in every single class. Copy-pasted code = a bug fixed in one place but forgotten in five others.
**Solution**: define shared structure ONCE in a parent class, child classes automatically inherit it, add only what's DIFFERENT.
```java
class Animal {
    String name;
    void eat() { System.out.println(name + " is eating"); }
}
class Dog extends Animal {
    void bark() { System.out.println(name + " is barking"); }
}
```
**Why Java allows only SINGLE class inheritance** (unlike C++'s multiple inheritance): C++ suffered the "Diamond Problem" — if a class inherits from two parents that both define the same method, which version wins? Ambiguous, caused real bugs and confusion. Java's designers deliberately banned multiple CLASS inheritance to avoid this entirely, choosing simplicity/safety over C++'s flexibility (fits Java's founding philosophy). Multiple inheritance of TYPE was later allowed via interfaces (file 04) — a safer, compromise solution, since interfaces (until Java 8) had no method bodies to conflict over.

### `super` keyword — why it exists
A child class often needs to explicitly reach INTO its parent — to call the parent's constructor (required as the very first line of any child constructor, implicitly or explicitly, so the parent's state is always fully initialized before the child adds its own), or to call a parent method the child has overridden but still wants to reuse part of.
```java
class Animal {
    String name;
    Animal(String name) { this.name = name; }
    void eat() { System.out.println(name + " eats"); }
}
class Dog extends Animal {
    Dog(String name) {
        super(name);              // MUST run parent constructor first
    }
    @Override
    void eat() {
        super.eat();               // reuse parent's version...
        System.out.println(name + " wags tail while eating"); // ...then add extra behavior
    }
}
```

### Method overriding rules — why they're strict
Overriding replaces a parent's method behavior in a child class — but Java enforces rules so overriding can't silently break the CONTRACT callers rely on: same method signature (name + parameter types), return type must be the same or a subtype ("covariant return type" — a method returning `Animal` can be overridden to return `Dog` specifically, since a `Dog` IS an `Animal`, so nothing calling the parent version is surprised), and access level can't be MORE restrictive than the parent's (can't override a `public` method as `private` — would break code that legitimately calls it as `public` through a parent reference). `@Override` isn't required but should always be used — it makes the compiler verify you're ACTUALLY overriding something (catches typos in method signatures that would otherwise silently create an unrelated new method instead of overriding).

### 3. Polymorphism
**Problem it solves**: without it, code handling different-but-related types needs explicit type-checking branches everywhere (`if (animal is Dog) ... else if (animal is Cat) ...`) — brittle, grows unmanageably as new types are added.
**Solution**: write code against the PARENT type/interface, let the ACTUAL object's own method implementation run automatically at runtime — no branching needed, and new subclasses "just work" with existing code, without modifying it.
```java
Animal myPet = new Dog();  // parent-type reference, child object
myPet.makeSound();          // Java decides at RUNTIME which version to run, based on actual object type

void feedAnimal(Animal a) { a.eat(); }  // works for ANY current or future Animal subclass
```
This ability to add new subclasses without touching existing code is a foundational software design principle (Open/Closed Principle — open for extension, closed for modification) — polymorphism is HOW Java achieves it.

**Compile-time vs runtime polymorphism — the two forms, why both are called "polymorphism"**: overloading (file 02) is resolved at COMPILE time based on argument types (the compiler figures out which version to call before the program even runs) — hence "static"/compile-time polymorphism. Overriding is resolved at RUNTIME based on the actual object's real type (the JVM checks the object at the moment the method is actually called) — hence "dynamic"/runtime polymorphism. Both let "the same call mean different things," just resolved at different times, for different reasons.

### `instanceof` and pattern matching — why needed occasionally, and why it's normally avoided
Polymorphism handles MOST type-varying logic cleanly, but sometimes code genuinely needs to check an object's real type (e.g. deserializing unknown data). `instanceof` performs that check; pre-Java-16 code then needed an extra manual cast to actually use the object as that specific type — an easy-to-forget, ceremony-heavy step. Pattern matching for `instanceof` (Java 16+) merges the check and cast into one line:
```java
// Old style:
if (animal instanceof Dog) {
    Dog d = (Dog) animal;   // manual cast, easy to forget or get wrong
    d.bark();
}
// Modern style (Java 16+):
if (animal instanceof Dog d) {   // 'd' is automatically cast and usable inside this block
    d.bark();
}
```

### 4. Abstraction
**Problem it solves**: complex systems have too much internal detail for a USER of that system to reasonably need or want to know. Forcing every caller to understand full internals creates tight coupling and cognitive overload.
**Solution**: expose a simple, essential interface; hide implementation complexity behind it.
```java
abstract class Shape {
    abstract double area();  // WHAT must be done, not HOW
}
class Circle extends Shape {
    double radius;
    double area() { return Math.PI * radius * radius; }  // HOW, hidden inside implementation
}
```
Abstraction and Encapsulation are related but distinct: Encapsulation hides DATA (internal state). Abstraction hides IMPLEMENTATION COMPLEXITY (internal logic/steps), exposing only the essential "what."

---

## Access Modifiers — why four levels, not just public/private
Real-world codebases have layered trust levels: your own class, your package (module of related classes), subclasses elsewhere, and truly external code. Java's four modifiers (`private`, default/package, `protected`, `public`) map exactly to these four real trust boundaries, giving fine-grained control instead of an all-or-nothing choice.

| Modifier | Same class | Same package | Subclass elsewhere | Everywhere |
|---|---|---|---|---|
| `private` | yes | no | no | no |
| default | yes | yes | no | no |
| `protected` | yes | yes | yes | no |
| `public` | yes | yes | yes | yes |

Practical rule: default to most restrictive (`private`), open up only as genuinely needed — minimizes accidental coupling, a direct application of encapsulation's philosophy.

---

## Object class fundamentals — `equals()`, `hashCode()`, `toString()`

**Why every class secretly extends `Object`**: Java needed some baseline guaranteed behavior for ANY object (default string representation, default comparison, etc), so it made `Object` the implicit root of every class hierarchy — guarantees these methods always exist, even if not overridden.

- `getClass()` → Returns runtime class info.    
- `finalize()` → Cleanup before garbage collection (deprecated now).
- `toString()` — default prints an unreadable memory-address-based string; override for meaningful debug output.
- `equals()` — default compares memory address (identity), same as `==`; override when you need CONTENT-based comparison (e.g. two different Person objects with same name/age should be "equal").
- `hashCode()` — MUST be overridden alongside `equals()`: Java's hashing-based collections (HashMap, HashSet — file 06) rely on a strict contract: if two objects are `equals()`, they MUST return the same `hashCode()`. Breaking this contract silently corrupts HashMap/HashSet behavior — a very real, hard-to-diagnose bug if forgotten.
```java
@Override
public boolean equals(Object obj) {
    if (this == obj) return true;
    if (!(obj instanceof Person)) return false;
    Person p = (Person) obj;
    return name.equals(p.name) && age == p.age;
}
@Override
public int hashCode() {
    return Objects.hash(name, age);
}
```

## `clone()` — why it exists, and why it's mostly avoided today
Sometimes code needs an independent COPY of an object, not just another reference to the same one. `Object.clone()` (via the `Cloneable` marker interface) was Java's original answer — but it's widely considered a design misstep: it does a shallow copy by default (copies field VALUES, but reference fields still point to the SAME nested objects), throws a checked exception, and requires implementing an empty marker interface with no methods of its own for it to even work. Modern Java code generally prefers a copy constructor or a static factory method instead, which are explicit, type-safe, and let you clearly choose shallow vs deep copy semantics yourself:
```java
class Point {
    int x, y;
    Point(Point other) { this.x = other.x; this.y = other.y; }  // copy constructor, explicit and clear
}
```

## Enums — why they exist
Some values are naturally a FIXED, KNOWN set (days of the week, order status: PENDING/SHIPPED/DELIVERED) — representing them as raw Strings or ints invites typos (`"Pendign"`) and invalid values (`status = 99`) that the compiler can't catch. An `enum` is a special class where the compiler guarantees only the listed constants can ever exist as a value of that type — invalid values become impossible, not just discouraged.
```java
enum OrderStatus { PENDING, SHIPPED, DELIVERED }
OrderStatus status = OrderStatus.PENDING;
switch (status) {
    case PENDING -> System.out.println("Waiting");
    case SHIPPED -> System.out.println("On the way");
    case DELIVERED -> System.out.println("Arrived");
}
```
Enums can also hold fields/methods of their own (each constant is technically a singleton instance of the enum class) — useful when each fixed value also carries associated data (e.g. each `Planet` enum constant carrying its own gravity value).

## Records (Java 16+) — why added
A huge share of everyday classes are pure, simple DATA carriers (a `Point(x, y)`, a `UserDTO(name, email)` — file 12) with no real behavior beyond holding fields, generating an equals/hashCode/toString, and exposing getters. Writing all of that boilerplate by hand for every such class was repetitive and error-prone (easy to forget updating `equals()` after adding a field). A `record` generates all of it automatically from a single declaration — immutable by design (fields are implicitly `final`), matching the same thread-safety/security reasoning behind String's immutability earlier in this file.
```java
record Point(int x, int y) { }
Point p = new Point(3, 4);
p.x();                 // auto-generated accessor (not getX(), a deliberate naming difference)
p.equals(new Point(3, 4));   // true — auto-generated, field-by-field
```

## Sealed classes/interfaces (Java 17+) — why added
Normal inheritance lets ANY class extend a public parent, anywhere, anytime — sometimes a design deliberately wants to allow only a SPECIFIC, KNOWN, closed set of subclasses (modeling something like "a Shape is EXACTLY a Circle, Square, or Triangle — nothing else"), so that code handling all cases (e.g. a modern switch expression, file 02) can be verified EXHAUSTIVE by the compiler, with no `default` case needed or possible surprise unknown subtype. `sealed` declares exactly which classes are permitted to extend it.
```java
sealed interface Shape permits Circle, Square, Triangle { }
final class Circle implements Shape { }
final class Square implements Shape { }
final class Triangle implements Shape { }
```
