# 07 — Java 8+ Functional Features

## Why these features were added (2014, a major turning point for Java)

By early 2010s, other languages (Python, JavaScript, Scala) popularized a more DECLARATIVE style of processing data — "what to do" rather than "how to loop through it step by step." Traditional Java code for processing collections was verbose, full of manual loops and temporary variables, and hard to parallelize safely. Java risked feeling outdated for modern data-processing needs. Java 8 introduced Lambdas + Streams + Optional specifically to close this gap, while staying backward-compatible with existing code (enabled by default interface methods, file 04).

## Lambda expressions — why they exist
**Problem before**: passing a small piece of BEHAVIOR (not data) as an argument required verbose anonymous inner classes.
```java
// Before Java 8:
Runnable r1 = new Runnable() {
    @Override
    public void run() { System.out.println("Running"); }
};
```
**Lambda solution**: compact syntax for the same thing, removing ceremonial boilerplate (class name, `@Override`, method signature repetition) while keeping full type safety underneath (still compiled to a real object implementing a functional interface — no "magic," just less typing).
```java
Runnable r2 = () -> System.out.println("Running");
```

### Variable capture in lambdas — why "effectively final" is required
A lambda can reference a local variable from its enclosing scope, but that variable must be `final` or "effectively final" (never reassigned after initialization, even without the explicit keyword). Why: the lambda might be executed LATER, potentially on a different thread, after the original method has already returned and its local stack frame no longer exists (file 01/02). Java captures a snapshot COPY of the variable's value at creation time rather than a live reference to the (soon-to-vanish) local variable — allowing reassignment afterward would create a confusing mismatch between the captured snapshot and a variable the lambda can't legally see change anyway, so Java forbids it outright at compile time instead of allowing subtly incorrect behavior.

## Functional interface — why the "exactly one abstract method" rule
For a lambda to work, the JVM needs to know EXACTLY which method the lambda's body is implementing — with more than one abstract method, it would be ambiguous which one the lambda refers to. The `@FunctionalInterface` annotation isn't required but documents intent and lets the compiler warn you if you accidentally break the single-method rule.
```java
@FunctionalInterface
interface Calculator {
    int operate(int a, int b);
}
Calculator add = (a, b) -> a + b;
```
Common built-ins (each exists for a specific recurring pattern):
- `Predicate<T>` — takes T, returns boolean (a "test" — used constantly in filtering)
- `Function<T,R>` — takes T, returns R (a "transform")
- `Consumer<T>` — takes T, returns nothing (a "do something with it" action)
- `Supplier<T>` — takes nothing, returns T (a "provide/generate a value" action)
- `BiFunction<T,U,R>` — takes two inputs of possibly different types, returns R (e.g. combining two values)
- `UnaryOperator<T>` — a `Function<T,T>` specialization where input and output are the same type (e.g. "double this number")

`Predicate` specifically also supplies default combinator methods (file 04's default methods in action) so filters can be composed without writing a brand-new lambda each time:
```java
Predicate<Integer> isEven = n -> n % 2 == 0;
Predicate<Integer> isPositive = n -> n > 0;
Predicate<Integer> isPositiveAndEven = isEven.and(isPositive);
```

## Method references — why added
Purely a further shorthand: when a lambda's ENTIRE body is just "call this one existing method," writing the lambda syntax at all is redundant ceremony.
```java
names.forEach(System.out::println); // equivalent to: names.forEach(n -> System.out.println(n));
```
Four forms exist because a method reference can point at different KINDS of existing methods, and Java needs slightly different syntax for each:
```java
Integer::parseInt;              // static method reference
str::toUpperCase;                // instance method reference on a PARTICULAR existing object
String::toUpperCase;             // instance method reference on an ARBITRARY future argument
ArrayList::new;                   // constructor reference — "make a new one of these"
```

## Stream API — why it exists
**Problem before**: processing a collection (filter some items, transform them, collect result) meant writing a manual loop with a temporary result list, mutating that list step by step — verbose, and hard to express "clearly" as a pipeline of transformations.
```java
// Old style:
List<Integer> result = new ArrayList<>();
for (int n : nums) {
    if (n % 2 == 0) result.add(n * n);
}
```
**Stream's solution**: express the SAME logic as a declarative pipeline — read almost like a sentence describing WHAT should happen, not the loop mechanics of HOW.
```java
List<Integer> evenSquares = nums.stream()
    .filter(n -> n % 2 == 0)
    .map(n -> n * n)
    .collect(Collectors.toList());
```
Bonus benefit this design enabled: Streams can be trivially switched to `.parallelStream()` to run across multiple CPU cores, because each pipeline stage is a pure, independent transformation — something much harder to safely retrofit onto old manual mutable-loop code (ties directly into file 08's concurrency concerns).

Mental model: think of it as an assembly line — items enter, each station does ONE transformation or filter, final station packages the result. `reduce()` — combines all elements into a single value (e.g. sum):
```java
int sum = nums.stream().reduce(0, (a, b) -> a + b);
```

### Intermediate vs terminal operations — why the distinction matters, and why Streams are "lazy"
`filter`, `map`, `sorted`, etc are **intermediate operations** — they don't actually DO anything by themselves, they just describe one stage of the pipeline and return another Stream. Only a **terminal operation** (`collect`, `forEach`, `reduce`, `count`, `sum`) actually triggers the entire pipeline to run, element by element, in one pass. This laziness exists deliberately for efficiency: without it, `filter` would need to fully process the ENTIRE collection before `map` could even start; with laziness, each element flows through the WHOLE pipeline (filter → map → collect) one at a time, and — crucially — a stream can also short-circuit (`findFirst`, `limit`) without processing elements that turn out to be unnecessary at all.
```java
nums.stream().filter(n -> n > 0);   // does NOTHING yet — no terminal operation called, nothing executes
```
A Stream can also only be consumed ONCE — calling a terminal operation a second time on the same stream throws `IllegalStateException`. This is a deliberate trade-off for the pipeline model: a Stream represents a one-time computation description, not a reusable, storable collection like a `List` — if you need to reprocess the same data again, you re-derive a fresh stream from the original source collection.

### Collectors — why a whole dedicated toolkit around `.collect()`
Turning a stream's processed elements back into a concrete, USEFUL result (not just "some elements") covers many genuinely different real needs — a plain list, a set, a grouped breakdown, a single joined string, a summary statistic. `Collectors` centralizes all these common "how do I package this stream's results" patterns so you don't hand-write the same accumulation logic repeatedly:
```java
List<String> names = people.stream().map(p -> p.name).collect(Collectors.toList());

Map<String, List<Person>> byCity = people.stream()
    .collect(Collectors.groupingBy(p -> p.city));   // "SQL GROUP BY," expressed in Java

Map<Boolean, List<Person>> adultsVsMinors = people.stream()
    .collect(Collectors.partitioningBy(p -> p.age >= 18));  // exactly two groups: true/false — a specialized, simpler case of grouping

String csv = names.stream().collect(Collectors.joining(", "));  // combine into one delimited String
```

## Optional — why it exists
**Problem it targets**: `NullPointerException` was famously called "the billion-dollar mistake" by Tony Hoare (inventor of the null reference concept) due to how common and costly null-related bugs were across the entire software industry. Java methods returning "maybe nothing" (e.g. a search that might not find a match) traditionally returned `null`, silently, and callers routinely FORGOT to check for it before using the result — instant crash.
**Optional's solution**: wrap a "might be absent" result in an explicit `Optional<T>` container, forcing the caller to consciously acknowledge and handle the absence case, rather than accidentally forgetting a null check.
```java
Optional<String> name = Optional.ofNullable(getName());
System.out.println(name.orElse("Default Name")); // explicit handling of the "missing" case
```

### Common Optional methods, and why chaining them avoids the null-check problem all over again
A naive misuse of `Optional` — calling `.get()` directly without checking `.isPresent()` first — just RECREATES `NullPointerException` under a new name (`NoSuchElementException`), missing the entire point. The intended usage style CHAINS operations, so the "absent" case is handled once, declaratively, without ever needing a manual if-check:
```java
Optional<String> maybeName = Optional.ofNullable(getName());

maybeName.map(String::toUpperCase)          // transform IF present, otherwise a no-op, still an Optional
         .filter(n -> n.length() > 2)         // further narrows, IF present
         .ifPresentOrElse(
             n -> System.out.println("Found: " + n),
             () -> System.out.println("Nothing found")
         );
```
`Optional` is intended for RETURN types (communicating "this might have nothing" to a caller) — it's generally discouraged as a field type or method parameter type, since those already have simpler, more idiomatic ways to express optionality (a nullable field with clear documentation, or method overloading), and wrapping every field in `Optional` adds ceremony without matching its original design intent.

## Java 9+ small but useful functional-adjacent additions worth knowing
- `Stream.of(...)`, `List.of(...)` — quick fixed-size stream/collection creation without a full collection literal.
- `takeWhile` / `dropWhile` — stream operations that stop (or start) processing as soon as a condition first fails/succeeds — useful for already-sorted data where you want elements "while a condition holds," distinct from `filter` which checks every single element regardless of position.
- `var` (Java 10, file 02) frequently appears in Stream-heavy code purely to reduce the visual noise of long generic types like `Map<String, List<Person>>`.
