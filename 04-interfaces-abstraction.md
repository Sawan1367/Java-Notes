# 04 — Interfaces vs Abstract Classes

## Why interfaces exist

After Java banned multiple class inheritance (file 03, Diamond Problem), a real gap remained: unrelated classes often need to guarantee they support the SAME set of behaviors, without sharing an actual code lineage. Example: `Bird` and `Airplane` are completely unrelated as classes, but both can "fly." Forcing them into one artificial shared parent class would be a bad, unnatural design.

**Interface's solution**: a pure CONTRACT — defines WHAT methods a class must provide, without saying HOW (originally, pre-Java-8, interfaces could have ZERO implementation, only method signatures). A class can implement MULTIPLE interfaces, since there's no code/state to conflict over (no Diamond Problem risk).

```java
interface Flyable {
    void fly();  // no body — implicitly public and abstract
}
class Bird implements Flyable {
    public void fly() { System.out.println("Bird flaps wings"); }
}
class Airplane implements Flyable {
    public void fly() { System.out.println("Airplane uses engines"); }
}
```

### Interface constants — why fields in an interface behave differently
A field declared inside an interface is implicitly `public static final` — always a constant, never per-instance state. This is deliberate: interfaces define behavior contracts, not object state; allowing genuine mutable instance fields in an interface would blur the line between "contract" and "implementation," and would reopen exactly the kind of multiple-inheritance data conflicts interfaces were designed to avoid.
```java
interface Constants {
    int MAX_USERS = 100;   // implicitly public static final, same as writing it explicitly
}
```

### Implementing multiple interfaces — why no conflict, unlike class inheritance
```java
interface Swimmer { void swim(); }
interface Runner { void run(); }
class Triathlete implements Swimmer, Runner {
    public void swim() { System.out.println("Swimming"); }
    public void run() { System.out.println("Running"); }
}
```
Since (pre-Java-8) interfaces had no method bodies, there was nothing ambiguous to inherit — the Diamond Problem specifically arises from conflicting IMPLEMENTATIONS, not conflicting method signatures. This is precisely why "multiple inheritance of type" (via interfaces) was considered safe to allow, while "multiple inheritance of implementation" (via classes) was not.

## Why default and static methods were added to interfaces (Java 8)

**Real problem Java's designers faced**: imagine `List` interface, used by millions of existing classes across the world. Java wanted to add a new method (e.g. `forEach`) to the `List` interface for the new Streams feature (file 07). But adding an abstract method to an interface would BREAK every existing class that implements `List` (they'd suddenly fail to compile — missing the new method).

**Solution**: `default` methods — interface methods WITH a body, optional to override. Existing implementing classes automatically inherit the default behavior, nothing breaks, but classes CAN override it if they want custom behavior.
```java
interface Drivable {
    void drive();
    default void honk() { System.out.println("Beep beep"); }  // has a body, backward-compatible
    static void info() { System.out.println("A drivable thing"); }  // utility method tied to interface itself
}
```
This single design decision is what enabled Java to evolve its core Collections API (adding Stream support) without breaking decades of existing code — a direct, real-world example of language evolution constraints shaping a feature's existence.

### The Diamond Problem's return, and how default methods resolve it
Once interfaces could carry actual behavior (default methods), the Diamond Problem became possible again in a limited form: if a class implements two interfaces that both define the SAME default method, which wins? Java's rule: it's a COMPILE ERROR — the implementing class is forced to explicitly override the method itself and choose (or combine) the behavior, rather than Java silently guessing. This is a deliberate, safer resolution than C++'s more permissive/ambiguous approach — consistent with Java's broader "force the programmer to make ambiguity explicit" philosophy (also seen in narrowing casts, file 02).
```java
interface A { default void greet() { System.out.println("A"); } }
interface B { default void greet() { System.out.println("B"); } }
class C implements A, B {
    public void greet() {              // MUST override — compiler forces the conflict to be resolved
        A.super.greet();                // explicitly choose (or combine) a specific parent's version
    }
}
```

### Private interface methods (Java 9) — why added
Once interfaces could have several `default` methods, those default methods sometimes shared common internal logic — but that shared helper logic had no way to stay truly PRIVATE/hidden (any method in an interface was at least `public`). Java 9 allowed `private` methods inside interfaces purely as internal helpers for other default/static methods to reuse, without exposing that helper as part of the interface's actual public contract.

## Interface vs Abstract class — when to use which, and why the distinction matters

**Abstract class**: partial implementation — some methods have bodies (shared code), some don't (`abstract`, must be implemented by subclass). Still follows single-inheritance rule (one abstract parent only).
```java
abstract class Shape {
    abstract double area();          // must implement
    void display() {                 // shared, inherited as-is
        System.out.println("This is a shape");
    }
}
```
Unlike an interface, an abstract class CAN have normal instance fields (real per-object state, not just constants), non-public members, and a constructor (run via `super()` from subclasses, file 03) — because it models a genuine "is-a" relationship with real shared state, not just a behavioral contract.

**Decision rule** (real design reasoning, not arbitrary):
- Use **interface** when: unrelated classes need a shared CAPABILITY/contract (Flyable, Comparable, Runnable) — behavior-based grouping, need multiple inheritance of type
- Use **abstract class** when: classes are naturally related (all ARE a Shape), and share actual reusable CODE or STATE, not just a method signature — "is-a" hierarchy with shared implementation

| | Interface | Abstract class |
|---|---|---|
| Inheritance | multiple allowed | single only |
| Fields | constants only (`public static final`) | any instance fields |
| Constructors | none | yes |
| Method bodies | default/static/private methods (Java 8+) | any method can have a body |
| Use case | "CAN do" capability contract | "IS a" shared-code hierarchy |

## Marker interfaces — why an interface with NO methods at all is still useful
Some interfaces (`Serializable`, `Cloneable`) declare zero methods — their sole purpose is to TAG a class with metadata the JVM or a library can check at runtime via `instanceof`, without requiring any actual method implementation. This predates Java's modern `@Annotation` system (which now largely replaces this pattern for new code), but marker interfaces remain in core/legacy APIs for backward-compatibility reasons.

## Sealed interfaces (Java 17+) — brief cross-reference
Just like sealed classes (file 03), an interface can also be `sealed`, restricting which classes/interfaces may implement it — useful when a capability should only ever be provided by a specific, closed, known set of types, enabling exhaustive `switch` handling over all implementers.

## Packages — why they exist
As programs grow to thousands of classes, name collisions become likely (two different libraries both define a class called `Utils`). Packages = namespaces, group related classes AND prevent naming collisions (`com.mycompany.utils.Utils` vs `com.otherlibrary.utils.Utils` — fully distinct despite same simple class name).
```java
package com.mycompany.myapp;
import java.util.List;
```
### Why the reverse-domain-name convention (`com.company.project`)
Since anyone can name a package anything, a GLOBAL naming collision is still possible between two unrelated organizations. Reversing a domain name you already own (`mycompany.com` → `com.mycompany`) as the package root guarantees global uniqueness for free — no central registry needed, since domain ownership is already globally unique and enforced elsewhere (DNS registrars).

### The Java Platform Module System (Java 9+) — why it exists on top of packages
Packages group classes, but historically had no enforced boundary at a larger scale: any public class in any package was accessible from anywhere on the classpath, even internal implementation classes never meant for outside use. The module system (`module-info.java`, "Project Jigsaw") lets a whole set of packages be bundled as a **module** that explicitly declares which packages it `exports` (usable by others) versus keeps fully internal — a stronger, JVM-enforced encapsulation boundary above the class/package level. Most everyday application code (especially typical Spring Boot projects) still runs on the traditional, simpler classpath rather than adopting the module system directly, but it's worth knowing it exists and why (it also enabled the JDK itself to be split into smaller, composable pieces via `jlink`, file 01).
