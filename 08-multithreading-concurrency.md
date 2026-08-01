# 08 — Multithreading & Concurrency

## Why multithreading came to exist

Early computers had a single CPU core — programs ran strictly one instruction after another. As hardware evolved to have MULTIPLE CPU cores (early-to-mid 2000s onward), a program running as a single sequential thread could only ever use ONE core, wasting the rest of the available hardware. Also, some tasks (waiting for a slow file read, network call) would freeze the ENTIRE program if done sequentially, even though the CPU was actually free to do other useful work meanwhile.

**Solution**: threads — independent sequences of execution WITHIN one program, that can run concurrently (interleaved on one core) or in true parallel (across multiple cores), letting programs use hardware fully and stay responsive during slow operations.

## Process vs Thread
- **Process**: an independently running program, with its OWN isolated memory space — one process cannot directly access another's memory (OS-enforced isolation, for stability/security).
- **Thread**: a lightweight execution unit WITHIN a process, SHARING that process's memory with other threads of the same process.

This SHARED memory is exactly what makes threads useful (fast communication, no copying data between them) AND exactly what makes them dangerous (multiple threads touching the same data simultaneously causes bugs — see race conditions below).

```java
Runnable task = () -> System.out.println("Running via Runnable");
Thread t = new Thread(task);
t.start();  // starts a NEW thread; calling task.run() directly would NOT create a new thread, just run it on the current one — common beginner mistake
```
Runnable preferred over extending `Thread` directly because Java only allows single class inheritance (file 03) — if your class already extends something else, you can't also extend Thread, but you CAN always implement the Runnable interface.

### Thread lifecycle — why threads pass through defined states
A thread isn't simply "running or not" — the JVM tracks several distinct states, each existing to represent a genuinely different real condition:
```
NEW → RUNNABLE → (BLOCKED / WAITING / TIMED_WAITING) → TERMINATED
```
- **NEW**: created (`new Thread(...)`) but `.start()` not yet called.
- **RUNNABLE**: eligible to run, either actively executing or waiting its turn for CPU time (the OS scheduler decides).
- **BLOCKED**: waiting to acquire a lock (`synchronized`, below) currently held by another thread.
- **WAITING / TIMED_WAITING**: paused deliberately (`wait()`, `join()`, `sleep()`) until another thread signals it or time elapses.
- **TERMINATED**: `run()` has completed (normally or via an uncaught exception).

Knowing these states matters practically: a thread stuck in BLOCKED forever (waiting on a lock nobody ever releases) is exactly what a deadlock looks like from the outside.

## Race conditions — why they happen (the exact problem synchronization solves)
```java
class Counter {
    int count = 0;
    void increment() { count++; }  // NOT one atomic step! actually: read count, add 1, write count back — 3 separate steps
}
```
If two threads call `increment()` at nearly the same time, they can both READ the same old value before either WRITES their updated value back — one thread's update gets silently lost. Expected 2000 after 1000 increments each from two threads, actual result unpredictable and LESS than 2000. This is a classic, very real bug class unique to concurrent programming, invisible in single-threaded code.

## `synchronized` — why it exists
Directly solves the race condition above: it's a LOCK — only one thread may execute a synchronized method/block on a given object at a time; any other thread trying to enter must WAIT until the first is done.
```java
synchronized void increment() { count++; }
```
Trade-off: correctness gained, but performance cost (threads waiting = wasted time) and RISK of "deadlock" (two threads each waiting on a lock the other holds, both stuck forever) — a real, harder problem that comes WITH the solution, worth mentioning even at intro level so it isn't a surprise later.

### Deadlock — why it happens, made concrete
```java
// Thread A: synchronized(lockX) { ... synchronized(lockY) { ... } }
// Thread B: synchronized(lockY) { ... synchronized(lockX) { ... } }
```
If Thread A grabs `lockX` and Thread B grabs `lockY` at nearly the same moment, then each tries to grab the OTHER's already-held lock — both wait forever, neither can proceed, neither will ever release what it's holding. This happens specifically when locks are acquired in INCONSISTENT order across different threads. Simplest practical prevention: always acquire multiple locks in the same, consistent, agreed-upon order everywhere in the codebase, removing the possibility of this circular-wait pattern entirely.

### `volatile` — why it exists, and how it differs from `synchronized`
Modern CPUs and the JVM aggressively cache values for speed — a thread reading a shared variable might actually be reading a stale, cached copy rather than another thread's most recent write, unless something forces "visibility" of the true, current value. `volatile` guarantees any read of that variable always sees the latest write from ANY thread — solving a VISIBILITY problem. It does NOT solve the race-condition ATOMICITY problem above (`volatile int count; count++;` is still unsafe — the read-modify-write is still three separate steps) — `synchronized` (or atomic classes, below) is still needed when an operation must be all-or-nothing across multiple steps. Use `volatile` for simple flags read/written as a single, whole value (e.g. a `volatile boolean running` flag telling a worker thread to stop); use `synchronized`/atomics for compound operations like increment.

### Atomic classes — why they exist as an alternative to `synchronized`
Wrapping a single counter increment in a full `synchronized` lock works but carries lock-acquisition overhead for what's fundamentally a simple, small operation. `java.util.concurrent.atomic` classes (`AtomicInteger`, `AtomicLong`, etc) provide lock-free, hardware-supported atomic operations (compare-and-swap, a CPU-level instruction) for exactly these common simple cases — same correctness guarantee as `synchronized` for a single value, generally better performance under contention.
```java
AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();   // atomic — safe across threads with no explicit lock needed
```

## ExecutorService — why introduced (replacing manual Thread creation)
**Problem with manually creating Threads**: creating and destroying a raw OS thread is relatively expensive; naive code creating thousands of threads for thousands of small tasks would overwhelm the system. Also, manually tracking/coordinating many raw threads is error-prone.
**Solution**: a managed THREAD POOL — a fixed set of reusable worker threads, and a queue of tasks submitted to them. Reuses threads instead of constantly creating/destroying them, and provides higher-level coordination tools.
```java
ExecutorService executor = Executors.newFixedThreadPool(4);
executor.submit(() -> System.out.println("Task running"));
executor.shutdown();
```
Why `.shutdown()` matters: an `ExecutorService`'s pool threads don't automatically stop just because your `main` method logic is done — without explicitly shutting the pool down, the JVM process can hang around indefinitely waiting on those still-alive pool threads instead of exiting cleanly.

### `Future` and `CompletableFuture` — why a "result that arrives later" needs its own type
Submitting a task to an `ExecutorService` doesn't return the RESULT immediately (the task hasn't finished yet) — it returns a `Future`, a placeholder representing "a result that will exist eventually." `Future.get()` blocks (waits) until that result is ready. `CompletableFuture` (Java 8+) improves on this by letting you CHAIN what happens once a result arrives, without manually blocking and waiting — directly borrowing the same declarative-pipeline mindset as Streams (file 07), applied to asynchronous work instead of collections:
```java
CompletableFuture.supplyAsync(() -> fetchDataFromServer())
    .thenApply(data -> process(data))
    .thenAccept(result -> System.out.println("Done: " + result));
```

## `wait()`/`notify()` — why they exist, and why higher-level tools usually replace them today
Sometimes a thread must PAUSE until some specific condition becomes true (a producer/consumer queue: consumer waits until the queue isn't empty). `Object.wait()` pauses a thread and releases its lock so others can proceed; `notify()`/`notifyAll()` wakes waiting thread(s) up. This low-level mechanism is notoriously easy to get subtly wrong (missed signals, spurious wakeups) — in modern code, higher-level `java.util.concurrent` tools (`BlockingQueue`, `CountDownLatch`, `Semaphore`) solve the same coordination problems with far less room for error, and are strongly preferred in real applications; `wait`/`notify` remain worth understanding conceptually since they're the foundation those higher-level tools are themselves built on.

## Concurrent collections — brief cross-reference to file 06
Regular `ArrayList`/`HashMap` are NOT safe for simultaneous multi-threaded modification. `ConcurrentHashMap` and `CopyOnWriteArrayList` are purpose-built thread-safe alternatives — internally using finer-grained locking or copy-on-write strategies (rather than one big lock around the whole structure) specifically so multiple threads can operate concurrently with good performance, not just correctness.

## Virtual threads (Java 21, Project Loom) — why they were introduced
Traditional Java threads map directly to a real, relatively heavyweight OS thread — meaning a server handling thousands of simultaneous slow, blocking operations (e.g. one thread per incoming web request, each waiting on a slow database call) needs thousands of expensive OS threads, an inherent scalability ceiling. Virtual threads are extremely lightweight threads MANAGED BY THE JVM ITSELF (not the OS) — a program can create millions of them cheaply, and the JVM efficiently maps many virtual threads onto a much smaller number of real OS threads, automatically "parking" a virtual thread while it's blocked/waiting so the underlying OS thread can serve someone else in the meantime. This directly targets the traditional-thread scalability ceiling for exactly the kind of high-concurrency, I/O-heavy workload a typical Spring Boot REST API (files 12-14) represents — worth knowing it exists even though deep usage is a more advanced topic for later.

## A note on learning pace
Keep this topic light for a first-time learner — real depth (deadlock debugging, concurrent collections internals, `CompletableFuture` composition, virtual threads in practice) comes later, once single-threaded fundamentals are fully solid. The goal at this stage is recognizing WHY concurrency exists and WHAT can go wrong, not yet mastering every tool that manages it.
