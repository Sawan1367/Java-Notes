# 11 — Spring Framework: History, IoC & Dependency Injection

## Why Spring came to exist (a real historical reaction)

Early-2000s Java enterprise development standard was **EJB (Enterprise JavaBeans)** — official, heavyweight, complex. Building even simple business applications with EJB required extensive boilerplate, verbose XML configuration, and classes tightly coupled to the EJB framework's own APIs (couldn't easily unit test business logic without deploying to a full EJB server). Developers were frustrated with the excessive complexity vs actual business value delivered.

Rod Johnson (2002-2004) published influential work criticizing this complexity and proposed a simpler alternative approach: **Plain Old Java Objects (POJOs)** — normal Java classes, framework-independent, wired together through **Dependency Injection** rather than complex framework inheritance/lookup APIs. This became the **Spring Framework** — deliberately positioned as the pragmatic, lightweight alternative to heavyweight EJB. Spring's massive adoption success eventually made it the DE FACTO standard for Java enterprise development, ironically becoming what EJB used to be, but built on genuinely better foundations.

## Why Inversion of Control (IoC) exists

**Traditional approach (tight coupling)**: a class creates its OWN dependencies directly.
```java
class Car {
    Engine engine = new Engine();  // Car controls creation of Engine — tightly coupled
}
```
**Real problems this causes**:
1. Hard to swap implementations (want an ElectricEngine instead? must edit Car's source code)
2. Hard to unit test (can't test Car in isolation with a FAKE/mock Engine — Car always creates a real one)
3. Car is responsible for BOTH its own logic AND managing its dependencies' lifecycle — violates separation of concerns

**IoC's core idea**: flip who's in CONTROL of creating and wiring objects together. Instead of your code controlling object creation, hand that control OVER to a framework/container — hence "inversion" (control flow inverted from your code to the framework).

## Why Dependency Injection (DI) exists (the specific technique implementing IoC)
```java
class Car {
    private final Engine engine;
    Car(Engine engine) {         // dependency INJECTED from outside, not created internally
        this.engine = engine;
    }
}
```
Now Car doesn't know or care HOW Engine was built, or WHICH specific Engine implementation it received — just that it received SOMETHING fulfilling the Engine contract. Solves all three problems above: swappable, testable (inject a mock Engine in tests), and Car's responsibility narrows to just its own logic.

Analogy: without DI, you cook your own food every time you're hungry (make your own dependency). With DI, you order food and someone else prepares/delivers it ready-to-use — you focus purely on eating (using the object), not on the labor of creating it.

## The Spring Container / ApplicationContext — why it exists
Someone still has to actually DO the wiring (create Engine, then create Car, passing Engine in) — doing this manually for a large application with hundreds of interdependent classes would itself become unmanageable. Spring's **container** (`ApplicationContext`) is that "someone" — it scans your code, discovers classes marked as manageable components, figures out their dependencies, creates them in the correct order, and wires them together automatically. These managed objects are called **beans**.

```java
@Component               // tells Spring: "manage this class as a bean"
class Engine { }

@Component
class Car {
    private final Engine engine;

    @Autowired            // tells Spring: "inject an Engine bean here automatically"
    Car(Engine engine) {
        this.engine = engine;
    }
}
```
`@Autowired` is technically OPTIONAL on a constructor when a class has exactly ONE constructor (Spring assumes that's the intended injection point automatically) — it becomes required only when a class has multiple constructors and you must tell Spring which one to use for injection.

### `@Component` and its specialized siblings — why several annotations for "this is a bean"
`@Service`, `@Repository`, and `@Controller` are all, underneath, `@Component` with extra semantic meaning attached — a direct application of the layered-architecture idea from file 10: the ANNOTATION NAME itself documents which layer a class belongs to, making the codebase self-describing at a glance, while `@Repository` additionally enables automatic translation of low-level database exceptions into Spring's own unified exception hierarchy (a real, practical extra behavior, not just a label), and `@Controller`/`@RestController` enables Spring's web-request-handling machinery (file 12).

### `@Qualifier` and `@Value` — why extra annotations are needed beyond `@Autowired`
Sometimes MULTIPLE beans satisfy the same type (two different `Engine` implementations both registered as beans) — Spring can't guess which one you want by type alone. `@Qualifier` names the SPECIFIC bean to inject, resolving that ambiguity explicitly rather than Spring guessing or throwing an error:
```java
@Component
class Car {
    @Autowired
    Car(@Qualifier("electricEngine") Engine engine) { ... }
}
```
`@Value` injects a simple external configuration value (from `application.properties`, file 12) directly into a field/parameter — useful for small, single settings that don't need a whole configuration class of their own:
```java
@Value("${app.max-users}")
private int maxUsers;
```

## Why constructor injection is preferred over field injection
```java
// Field injection (works, but discouraged):
@Component
class Car {
    @Autowired
    private Engine engine;
}
```
Problems with field injection: `engine` can't be `final` (mutable after construction, less safe), harder to unit test (must use reflection tricks or a Spring context just to set the field, can't just call `new Car(mockEngine)`), and dependencies are hidden (not visible in the class's public API/constructor signature — you must read the whole class body to know what it needs). Constructor injection makes dependencies explicit, immutable, and trivially testable with plain `new Car(mockEngine)` in a unit test — no framework needed for testing at all. This preference reflects a broader, deliberate Spring community shift toward "framework-light" POJOs, continuing Spring's founding philosophy.

## Bean scopes — why "one shared instance" isn't always right
By default, a Spring bean is a **singleton** — ONE shared instance for the entire application, reused everywhere it's injected. This default makes sense for genuinely stateless, shared services (a `UserService` has no reason to exist twice). But some things genuinely need a FRESH instance every time they're requested (a bean holding request-specific or user-specific mutable state) — the **prototype** scope creates a brand-new instance on every injection, trading Spring's default memory/performance efficiency for correctness in that specific scenario. (Web-specific scopes — `request`, `session` — also exist, tying a bean's lifetime to one HTTP request or one user's session, directly relevant once file 12's web layer is involved.)
```java
@Component
@Scope("prototype")
class ReportGenerator { ... }
```

## Profiles — why environment-specific bean configuration exists
The SAME application often needs genuinely different bean configurations across environments (a real database connection in production, an in-memory test database in tests, verbose debug logging only in development) — hard-coding environment checks throughout business logic would be messy and error-prone. `@Profile` lets a bean definition be registered ONLY when a specific named profile is active, letting Spring Boot's `application-{profile}.properties` file mechanism (file 12) cleanly separate environment-specific wiring from the application's core logic.

## AOP (Aspect-Oriented Programming) — why it exists as a complement to DI
Some concerns genuinely CUT ACROSS many unrelated classes/methods — logging every method call, checking security permissions before certain methods run, measuring performance timing, managing transactions (file 13). Implementing these by manually repeating the same boilerplate code inside every single method that needs it violates DRY (file 02) badly, and scatters a single "concern" across the entire codebase instead of keeping it in one place. **AOP** lets you define this cross-cutting behavior ONCE, as an "aspect," and declare WHERE it should automatically apply (e.g. "before every method in the service layer") — Spring weaves it in automatically, without touching each target method's own source code. `@Transactional` (file 13) is the most common real-world example a beginner will actually use: it's AOP quietly wrapping a method with transaction-management logic behind the scenes, based purely on that one annotation.

## Circular dependencies — a real, practical DI gotcha worth knowing about early
If Bean A's constructor needs Bean B, and Bean B's constructor needs Bean A, Spring can't determine which one to build FIRST — this is a genuine design smell (usually meaning two classes are too tightly coupled and one should be refactored/split), and with constructor injection specifically, Spring will fail fast at startup with a clear error rather than silently guessing — another concrete benefit of constructor injection's explicitness over field injection, where such cycles can sometimes be worked around less safely via lazy initialization.

## Bean lifecycle (brief)
Spring container starts → scans for `@Component` (and related) classes → creates beans in dependency order → injects dependencies → beans ready for use by the application → (on shutdown) beans destroyed, cleanup hooks run if defined. Entirely managed automatically — this automation is precisely WHY IoC/DI is valuable at scale, not just a theoretical nicety.

Lifecycle callback hooks (`@PostConstruct`, `@PreDestroy`) exist for the same reason a constructor alone (file 03) sometimes isn't enough: a bean's dependencies are only guaranteed fully injected AFTER construction completes, so any initialization logic that NEEDS those injected dependencies must run in a separate post-construction hook, not the constructor itself; `@PreDestroy` symmetrically guarantees cleanup logic (closing a connection pool, releasing a resource) runs during an orderly container shutdown.
