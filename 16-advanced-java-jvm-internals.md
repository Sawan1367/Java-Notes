# 16 — Advanced Java & JVM Internals

## `int` vs `Integer` — the full picture (why this distinction runs through the whole language)

This single distinction (primitive vs wrapper) is the root of a huge number of "why does Java behave this way" questions elsewhere in these notes (autoboxing in file 02, Collections only holding objects in file 06, `Optional` in file 07). Worth nailing down completely, once, here.

| | `int` | `Integer` |
|---|---|---|
| Kind | primitive | object (wrapper class, file 02) |
| Stored | raw 4-byte value, on the stack (or inline inside an object on the heap) | reference to an `Integer` object living on the heap; the reference itself sits on the stack |
| Default value (as a field) | `0` | `null` |
| Can be `null`? | never — always holds SOME value | yes — represents "no value present" |
| Memory overhead | none beyond the 4 bytes | object header (~12-16 bytes JVM overhead) + the 4-byte value |
| Usable in Collections/Generics? | no — `List<int>` does not compile | yes — `List<Integer>` |
| `==` compares | actual value | object reference (identity), UNLESS both operands come from the Integer cache (below) |
| Has methods? | no | yes (`.compareTo()`, `.toString()`, `Integer.parseInt()`, etc) |

```java
int a = 5;
int b = 5;
System.out.println(a == b); // true — comparing raw values, always correct

Integer x = new Integer(5);   // deprecated since Java 9 — forces a NEW heap object
Integer y = new Integer(5);
System.out.println(x == y);   // false — two different objects, comparing references

Integer p = 5;   // autoboxing — compiler calls Integer.valueOf(5), not `new Integer(5)`
Integer q = 5;
System.out.println(p == q);   // true — BUT only because of the Integer cache below
```

### The Integer cache trap — why `==` "works" for small numbers and silently breaks for large ones
`Integer.valueOf(int)` — which is what autoboxing calls under the hood — caches and reuses `Integer` objects for values **-128 to 127** (a JVM-internal optimization: these are overwhelmingly the most common small integers used in real programs, so reusing them avoids constant re-allocation). Outside that range, `valueOf()` always allocates a brand-new object.
```java
Integer m = 127, n = 127;
System.out.println(m == n); // true — both hit the cache, same object

Integer big1 = 128, big2 = 128;
System.out.println(big1 == big2); // false — outside cache range, two distinct objects
```
This is exactly why the rule "always use `.equals()`, never `==`, to compare wrapper/object values" exists (same root reason `.equals()` beats `==` for `String`, file 02) — `==` on objects is testing "is this the literal same object in memory," not "do these represent the same value," and the cache makes that distinction invisible for small numbers and very visible for large ones. A bug that only reproduces above 127 is a classic real-world consequence of not understanding this.

### Why unboxing a `null` Integer crashes
```java
Integer count = null;
int total = count + 5; // NullPointerException — JVM tries count.intValue() on a null reference
```
Autoboxing/unboxing is compiler *sugar*, not magic — `count + 5` where `count` is an `Integer` literally compiles to `count.intValue() + 5`. If `count` is `null`, that's a method call on `null`, same as any other NPE. This is the single most common source of "surprise" NPEs in code that mixes primitives and wrappers carelessly — e.g. a `Map<String,Integer>` counter where `map.get(key)` returns `null` for a missing key, then gets used in arithmetic.

### When to deliberately choose the wrapper over the primitive
- Field can legitimately be "unset" (e.g. an entity's DB column is nullable) → use `Integer`, not `int` — `int` cannot represent "no value," it would default to misleading `0`.
- Storing in any Collection/Generic → forced to use the wrapper.
- Everyday local variables, loop counters, math-heavy code → use the primitive: less memory, no null risk, faster (no boxing/unboxing overhead, no heap allocation/GC pressure).

The same table/reasoning applies identically to every primitive/wrapper pair: `long`/`Long`, `double`/`Double`, `boolean`/`Boolean`, `char`/`Character`, etc.

---

## JVM, JDK, JRE architecture — going deeper than file 01
File 01 introduced these three; here's the actual internal machinery.

```
Your .java source
   ↓  javac (compiler, part of JDK)
.class bytecode file  (platform-independent instructions, NOT machine code)
   ↓  loaded and run by the JVM
JVM: Class Loader → Runtime Data Areas → Execution Engine
```

### Class loading — why it happens in three distinct phases
A class isn't just "read off disk and ready to go" — the JVM deliberately splits loading into stages so it can share loaded classes across the app, verify safety, and delay work until actually needed:
1. **Loading** — finds the `.class` bytecode (from disk, a JAR, or network) and reads it into memory as a `Class` object. Done by a **ClassLoader** — and there's a hierarchy of these (Bootstrap → Platform → Application/System), each responsible for a different source of classes (JDK core classes vs your app's own classes), following a **parent-delegation model**: a class loader always asks its PARENT to try loading first, only loading it itself if the parent can't — this prevents a malicious or accidental app-level class from silently replacing a core JDK class like `java.lang.String`.
2. **Linking** — three sub-steps: *Verification* (bytecode is checked for structural/security correctness — you can't feed the JVM hand-crafted invalid bytecode and have it just run), *Preparation* (static fields allocated and set to default zero-values), *Resolution* (symbolic references to other classes resolved to actual references).
3. **Initialization** — static initializers and static field assignments actually run, in order, the FIRST time the class is genuinely used (not merely loaded) — this laziness is deliberate: no cost paid for classes that are loaded (e.g. present in a JAR) but never actually used in a given run.

### JVM memory areas — completing file 01's picture
| Area | What lives there | Why it's separate |
|---|---|---|
| **Heap** | all objects, instance fields | shared across all threads — needs GC (below), needs to be large/resizable |
| **Stack** (one per thread) | local variables, method call frames, primitive values | thread-private by design — no locking needed for local variables, which is exactly why local variables are inherently thread-safe while instance/static fields are not (file 08) |
| **Metaspace** (replaced PermGen in Java 8) | class metadata itself (method bytecode, static structure) | separated from heap so class-loading-heavy apps (many frameworks/libraries) don't exhaust the same memory pool user objects live in; PermGen had a FIXED size and was a notorious source of `OutOfMemoryError` in app servers that reloaded classes often — Metaspace uses native OS memory and grows dynamically, directly fixing that |
| **PC (Program Counter) Register** (one per thread) | address of the currently executing instruction for that thread | needed because each thread independently tracks its own execution point |
| **Native Method Stack** | state for native (non-Java, e.g. C) method calls | JVM sometimes calls into native code (JNI) — needs its own stack discipline, separate from Java's |

### Bytecode, JIT, and why Java is "compiled AND interpreted"
`.class` bytecode is not native machine code — it's an intermediate, platform-neutral instruction set. The JVM's **Execution Engine** initially *interprets* bytecode (reads and executes it instruction-by-instruction — flexible, portable, but slower). The **JIT (Just-In-Time) compiler** watches which methods run frequently ("hot" methods) and compiles JUST those to real native machine code at runtime, caching the result — giving Java near-native speed for the code that actually matters, without sacrificing bytecode's write-once-run-anywhere portability for everything else. This is the deliberate middle ground between pure interpretation (slow, portable) and pure ahead-of-time compilation (fast, NOT portable — different machine code per OS/CPU).

### Garbage Collection algorithms — file 02 explained WHAT GC is; here's HOW
The heap is split into **generations**, based on the empirical observation that most objects die young (a method's local temp objects) while a few live a very long time (caches, singletons) — treating all objects identically would waste effort repeatedly re-scanning long-lived objects that are never garbage.
- **Young Generation** (further split into Eden + two Survivor spaces): new objects allocated here. Minor GC runs frequently, is fast, because most objects here are already garbage by the time it runs.
- **Old Generation (Tenured)**: objects that survive several young-GC cycles get "promoted" here. Major/Full GC runs less often but is more expensive, since it must scan a much larger, longer-lived object set.

| Collector | Strategy | Best for |
|---|---|---|
| Serial GC | single-threaded, stops app entirely during GC | small apps, single-core environments |
| Parallel GC | multiple threads do GC work, still stops app | throughput-focused batch jobs, older Java default |
| CMS (Concurrent Mark Sweep) | does most work concurrently WITH the app running | low-latency needs (deprecated/removed in newer Java, superseded below) |
| **G1 (Garbage First)** | splits heap into many small regions, collects the ones with most garbage first; default since Java 9 | general-purpose default — balances throughput and pause time |
| **ZGC / Shenandoah** | designed for extremely large heaps with sub-millisecond pauses, almost fully concurrent | latency-critical services with huge heaps (multi-GB to TB) |

---

## Generics & type erasure — why `List<String>` isn't what it looks like at runtime
**Problem generics solve**: before Java 5, collections held raw `Object` — `List list = new ArrayList(); list.add("x");` compiled fine, but `list.add(42)` also compiled fine, and pulling items back out required an explicit, unsafe cast (`(String) list.get(0)`) that could fail at RUNTIME with a `ClassCastException` far from the actual bug.
**Generics' fix**: `List<String>` lets the COMPILER enforce type-correctness at compile time — `list.add(42)` on a `List<String>` is now a compile error, not a runtime surprise.

**Type erasure** — why generics vanish at runtime: for backward compatibility with pre-Java-5 bytecode/libraries, the JVM does NOT actually keep `List<String>` as a distinct runtime type — the compiler enforces the type rule, then "erases" the generic info, compiling `List<String>` down to plain `List` with inserted casts. This is why `list.getClass()` on a `List<String>` and a `List<Integer>` returns the identical `ArrayList` class, and why you can't do `new T[10]` or `instanceof List<String>` — the runtime genuinely doesn't have that information anymore.
```java
List<String> strings = new ArrayList<>();
List<Integer> ints = new ArrayList<>();
System.out.println(strings.getClass() == ints.getClass()); // true — erasure
```

### Bounded wildcards — completing file 06's generics mention
- `<? extends T>` (upper bound) — "some subtype of T, I only READ from this" (Producer). Can't safely `add()` to it (compiler doesn't know the EXACT subtype).
- `<? super T>` (lower bound) — "some supertype of T, I only WRITE into this" (Consumer).
- Mnemonic: **PECS** — Producer Extends, Consumer Super.

---

## Reflection — why Java lets code inspect/modify itself at runtime
**Problem**: some legitimate needs (frameworks like Spring building objects from config, testing tools, serialization libraries) require code that works with classes it doesn't know about at COMPILE time — a dependency-injection container can't hardcode `new UserService()` for every possible class a user might define.
**Reflection** lets code examine a class's structure (fields, methods, constructors, annotations) and even invoke/modify them, all at runtime, using only the `Class` object.
```java
Class<?> clazz = Class.forName("com.example.User");
Object instance = clazz.getDeclaredConstructor().newInstance();
Method method = clazz.getDeclaredMethod("getName");
Object result = method.invoke(instance);
```
This is EXACTLY the mechanism Spring's `@Autowired` (file 11), `@RestController` mapping (file 12), and JPA's entity mapping (file 13) all rely on internally — those annotations are inert metadata until a framework uses reflection at startup to read them and wire things together. Trade-off: reflection bypasses normal compile-time checks and is noticeably slower than direct calls — used deliberately by frameworks, rarely appropriate in everyday application code.

## Annotations — why metadata needed a first-class language feature
Before annotations (Java 5), frameworks configured behavior via separate XML files (`web.xml`, Hibernate mapping files) — correct but disconnected from the code it described, easy for code and config to drift out of sync. Annotations attach metadata DIRECTLY to the code element it describes, read via reflection (above) by whatever framework cares about them.
```java
@Retention(RetentionPolicy.RUNTIME)  // must survive to runtime to be reflectively readable
@Target(ElementType.METHOD)          // only valid on methods
public @interface LogExecutionTime { }

@LogExecutionTime
public void process() { ... }
```
`@Retention` matters because by default annotations are discarded after compilation (`SOURCE`) or kept in bytecode but invisible to reflection (`CLASS`) — a framework that needs to READ your custom annotation at runtime (like Spring reading `@Service`) requires `RUNTIME` retention, otherwise the annotation info simply isn't there to find.

---

## Java Memory Model & advanced concurrency — extending file 08

### `volatile` — why `synchronized` isn't always necessary for visibility
File 08 covers locks for mutual exclusion. A separate, narrower problem: each CPU core may cache a variable's value locally for speed — one thread updating a shared variable doesn't guarantee ANOTHER thread's cached copy sees the new value promptly. `volatile` tells the JVM: always read/write this variable directly from main memory, never from a per-core cache — guaranteeing VISIBILITY of updates across threads. It does NOT provide atomicity for compound operations (`count++` is still read-modify-write, still unsafe even if `count` is `volatile`) — for that, still need `synchronized` or `java.util.concurrent.atomic` classes (`AtomicInteger`, etc), which use CPU-level compare-and-swap instructions for lock-free atomic updates.

### Executor framework — why raw `Thread` creation doesn't scale
Creating a new OS thread per task (file 08's basic `new Thread()`) is expensive (thread creation/teardown overhead) and uncontrolled (nothing stops an app from spawning thousands of threads and exhausting memory). The **Executor framework** manages a reusable pool of worker threads, decoupling "submitting a task" from "how/when it actually runs":
```java
ExecutorService pool = Executors.newFixedThreadPool(4);
pool.submit(() -> doWork());
pool.shutdown(); // stop accepting new tasks, let queued ones finish
```

### `CompletableFuture` — why async code needed a better shape than raw callbacks
Returning a plain `Future` (older API) only lets you `.get()` and BLOCK waiting for a result — defeating the point of async work. `CompletableFuture` lets you chain what happens NEXT without blocking:
```java
CompletableFuture.supplyAsync(() -> fetchUser())
    .thenApply(user -> user.getName())
    .thenAccept(name -> System.out.println(name));
```
This directly underlies how modern Spring WebFlux and many async I/O patterns in real backend systems are structured.

### `ThreadLocal` — why some state should NOT be shared even though threads share memory
Occasionally you want a variable that's isolated PER THREAD despite threads normally sharing an object's memory — e.g. a per-request user context in a web server, where each request is handled by a different pooled thread and must not see another request's data. `ThreadLocal<T>` gives each thread its own independent copy.

### Virtual threads (Java 21) — why they exist
Traditional (platform) threads map 1:1 to a real OS thread — expensive, so pools cap concurrency (a few hundred/thousand threads max). Many backend workloads (a web server handling thousands of mostly-IDLE connections waiting on I/O) don't need that much real parallelism, just huge CONCURRENCY. **Virtual threads** are lightweight threads managed by the JVM itself (not the OS), letting an app run millions of them cheaply — a direct, deliberate answer to the popularity of async/reactive programming models (which existed largely to work around the cost of platform threads) by making blocking-style code cheap enough to use at that scale again.

---

## Common design patterns — why standard shapes for common problems exist
Patterns aren't rules but named, reusable solutions to recurring design problems — naming them lets teams communicate design intent quickly ("just make it a Factory") instead of re-explaining from scratch.

- **Singleton** — exactly one instance of a class ever exists (e.g. a single shared config object). Risk: naive implementations aren't thread-safe (file 08) — two threads could both see "not yet created" and both create one; fixed via `synchronized`, double-checked locking, or (cleanest) an `enum` singleton.
- **Factory** — a method/class dedicated to CREATING objects, hiding the concrete class chosen from the caller — lets you change which concrete implementation gets created without touching every call site.
- **Builder** — constructs a complex object step-by-step via chained calls, avoiding constructors with a huge number of optional parameters (the "telescoping constructor" problem).
- **Observer** — objects (observers) subscribe to be notified when another object's (subject's) state changes — the exact conceptual root of event listeners and reactive streams.
- **Strategy** — encapsulates an interchangeable ALGORITHM behind a common interface, so behavior can be swapped at runtime (e.g. different sorting/pricing/validation strategies plugged in without changing the code that uses them) — this is precisely what a functional interface + lambda (file 07) makes lightweight to implement in modern Java, versus needing a full class hierarchy pre-Java-8.
