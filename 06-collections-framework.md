# 06 — Collections Framework

## Why Collections came to exist (replacing raw arrays)

Arrays (file 02) have hard limitations that become real problems in practical programs:
- **Fixed size** — must know max size UPFRONT; resizing means manually creating a new bigger array and copying everything over
- **No built-in rich operations** — no built-in sort-by-custom-rule, no easy remove-by-value, no built-in duplicate prevention, no key-value pairing
- Every program that needed dynamic lists, unique-value sets, or key-value lookups had to hand-roll this logic repeatedly — massive duplicated effort across the whole Java ecosystem before Java 1.2 (1998) introduced the unified Collections Framework.

**Solution**: a standard, well-tested, interoperable set of data structure interfaces and implementations, covering the recurring real-world patterns: ordered lists, unique sets, key-value maps — each with multiple implementation choices trading off speed vs order vs memory.

### The overall shape of the framework — why it's an interface hierarchy
```
Collection (interface)
├── List    — ordered, duplicates allowed
├── Set     — unique elements
└── Queue   — processing order (FIFO, priority, etc)
Map (separate hierarchy — key/value pairs, not a "collection of single elements")
```
Deliberately mirrors the interface-vs-implementation philosophy from file 04: your code should generally be written against the INTERFACE (`List<String> names = ...`), so the concrete implementation (`ArrayList` vs `LinkedList`) can be swapped later based on performance needs without touching the code that USES the collection.

## List — why it exists, and why multiple implementations
```java
List<String> names = new ArrayList<>();
names.add("John");
names.add("Jane");
names.add("John");  // duplicates ALLOWED — a List models an ORDERED SEQUENCE, matching real ordered data
```
- **ArrayList** — internally backed by a resizable array. Chosen when reading by index is the common operation (`get(i)` is O(1), constant time — direct memory offset calculation). Inserting/removing in the MIDDLE is slow (O(n)) because remaining elements must shift.
- **LinkedList** — internally a chain of nodes, each pointing to the next/previous. Insert/remove is fast (O(1)) once you're AT the position (no shifting), but reaching that position (random access) is slow (O(n), must walk the chain). Also directly implements the `Deque` interface (below), letting it double as a stack or queue.

This existence of TWO implementations behind one `List` interface is a direct lesson in a core software design principle: program against the INTERFACE, choose the IMPLEMENTATION based on actual usage pattern (read-heavy vs write-heavy).

### How ArrayList resizing actually works, and why it matters
An `ArrayList` starts with some initial internal array capacity; when a new element is added beyond that capacity, Java allocates a NEW, larger internal array (commonly ~1.5x the old size) and copies every existing element over — an O(n) operation, but one that happens INFREQUENTLY (not on every single `add()`), so the AVERAGE cost per add stays close to O(1) ("amortized constant time"). Knowing this explains a real practical tip: if you know roughly how many elements you'll store upfront, constructing with `new ArrayList<>(expectedSize)` avoids several unnecessary resize-and-copy cycles.

## Set — why it exists
Real-world need: "give me only unique values" (unique usernames, unique tags) — with an array/List, preventing/checking duplicates manually means writing a loop-and-check every single insert (O(n) per insert, slow at scale). Set formalizes "no duplicates" as a GUARANTEE of the data structure itself, and implementations use hashing internally to check/reject duplicates in near-constant time.
```java
Set<String> uniqueNames = new HashSet<>();
uniqueNames.add("John");
uniqueNames.add("John");  // silently ignored, duplicate
```
- **HashSet** — fastest, no order guaranteed (order determined by internal hash bucket placement, not insertion order)
- **LinkedHashSet** — maintains insertion order, small extra overhead
- **TreeSet** — maintains SORTED order automatically, using a self-balancing tree internally, slightly slower (O(log n) operations vs HashSet's O(1))

## Map — why it exists
Real-world need: "look up a value by a key" (username → user profile, product ID → product details) — without Map, you'd need a parallel-arrays hack (one array of keys, one of values, manually keeping them in sync — error prone) or a linear search through a List of pairs (slow).
```java
Map<String, Integer> ages = new HashMap<>();
ages.put("John", 25);
ages.put("John", 26);  // overwrites — keys are unique by definition
System.out.println(ages.get("John")); // 26
```

### How HashMap actually works internally (the "why" behind its speed)
Each key's `hashCode()` (file 03) is used to compute WHICH internal "bucket" (small sub-list) it belongs in — so instead of scanning ALL entries to find a key, HashMap jumps almost directly to the right bucket, then does a small local search using `equals()` to confirm exact match. This is WHY the `equals()`/`hashCode()` contract (file 03) is not optional — a broken hashCode() breaks the bucket placement, and HashMap silently starts behaving incorrectly (lookups fail even though the key visually looks present).

- **HashMap** — fastest, no order
- **LinkedHashMap** — insertion order preserved
- **TreeMap** — sorted by key, uses a tree internally

### Useful Map methods that avoid manual null-checking boilerplate
Before these existed, common patterns ("get a value, or a default if absent," "only insert if not already present," "update a running total for a key") required verbose manual `containsKey`/`get`/`put` sequences. These methods (added mainly Java 8) collapse that boilerplate into one call:
```java
ages.getOrDefault("Mike", 0);                         // avoid manual null-check after get()
ages.putIfAbsent("John", 30);                          // won't overwrite existing "John" entry
ages.merge("John", 1, Integer::sum);                    // "increment, or insert 1 if absent" in one call
ages.forEach((name, age) -> System.out.println(name + ":" + age));
```

## Queue & Deque — why they exist
Some real problems are naturally about PROCESSING ORDER, not arbitrary indexed access — a print job queue (first submitted, first printed — FIFO), an "undo" history (last action, undone first — LIFO), or task scheduling by priority. Modeling these with a plain `List` works but doesn't communicate INTENT (nothing stops you from grabbing the middle element of a "queue" by accident) and isn't optimized for the specific add/remove-at-the-ends pattern these use cases need.
```java
Queue<String> printQueue = new LinkedList<>();
printQueue.offer("doc1.pdf");   // add to back
printQueue.poll();               // remove and return from front — FIFO

Deque<String> undoStack = new ArrayDeque<>();  // "Deque" = double-ended queue, usable as a Stack too
undoStack.push("typed 'hello'");
undoStack.pop();                                // remove and return from front — LIFO when used this way
```
`ArrayDeque` is generally preferred over the older `Stack` class today — `Stack` predates the Collections Framework, extends `Vector` (a legacy, synchronized-by-default class carrying unnecessary locking overhead for typical single-threaded use), a good example of "the newer, purpose-built type is usually the better default even when an older type technically still works."

### PriorityQueue — why order isn't insertion order
A `PriorityQueue` always returns elements in a defined PRIORITY order (smallest first, by default, via natural ordering or a custom `Comparator`) rather than insertion order — useful for scheduling algorithms, "process the most urgent task first" style problems. Internally backed by a binary heap, giving O(log n) insert/remove-highest-priority — far better than manually re-sorting a List after every insertion.

## Iterator — why it exists
Directly looping over a collection with an index (like an array) doesn't work cleanly for structures without meaningful indices (Set, Map) or when SAFELY removing elements mid-loop (removing directly during a for-each loop throws `ConcurrentModificationException` — a deliberate fail-fast safety check). Iterator provides a UNIFORM, structure-agnostic way to traverse ANY collection, with a safe `remove()` method built in.
```java
Iterator<String> it = names.iterator();
while (it.hasNext()) {
    if (it.next().equals("John")) it.remove(); // safe mid-loop removal
}
```
**Why `ConcurrentModificationException` exists at all**: most collections keep an internal "modification count"; the iterator checks that count hasn't changed unexpectedly on each step. This is a deliberate FAIL-FAST design — better to throw a clear, immediate exception the moment unsafe concurrent structural modification is detected, than to silently produce corrupted/undefined iteration results that might not surface as a bug until much later.

## Comparable vs Comparator — why both exist, not just one
Real need: sometimes a class has ONE obvious natural sort order (numbers ascending, dates chronological) — but sometimes you need MULTIPLE different valid sort orders for the SAME objects depending on context (sort students by marks sometimes, by name other times), without permanently baking one choice into the class itself.
```java
class Student implements Comparable<Student> {
    int marks;
    public int compareTo(Student other) { return this.marks - other.marks; } // ONE natural order, inside the class
}
students.sort((a, b) -> b.marks - a.marks);           // Comparator: alternate order, defined OUTSIDE, as needed
students.sort(Comparator.comparing(s -> s.name));      // another alternate order
```
`Comparator` also supports chaining multiple sort criteria cleanly — useful when a single field alone doesn't fully determine order (e.g. sort by last name, then by first name as a tiebreaker):
```java
students.sort(Comparator.comparing((Student s) -> s.lastName)
                         .thenComparing(s -> s.firstName));
```

## Utility classes: `Collections` and `Arrays` — why they exist separately from the data structures themselves
Following the same "small, focused responsibility" philosophy as the layered architecture discussed in file 10, common OPERATIONS on collections/arrays (sorting, searching, reversing, finding min/max, creating read-only wrappers) were deliberately placed in separate static-method utility classes rather than bloating every collection class itself with dozens of extra methods.
```java
Collections.sort(names);                       // sort any List using natural order
Collections.reverse(names);
Collections.max(numbers);
List<String> readOnly = Collections.unmodifiableList(names);  // wraps a list, blocks further mutation

int[] arr = {5, 2, 8};
Arrays.sort(arr);
System.out.println(Arrays.toString(arr));       // arrays don't override toString() usefully by default (file 03) — this utility fixes that for debugging
```

### Immutable collection factories (Java 9+) — why added
Before Java 9, creating a genuinely read-only collection required the slightly awkward `Collections.unmodifiableList(new ArrayList<>(...))` wrapping pattern. Since immutability has real, recurring benefits (thread-safety, safety against accidental mutation — same reasoning as String's immutability, file 02), Java 9 added direct factory methods for the common case of "I just want a fixed, unmodifiable collection of known values":
```java
List<String> names = List.of("John", "Jane");   // immutable — .add() throws UnsupportedOperationException
Map<String, Integer> ages = Map.of("John", 25, "Jane", 30);
```

## A brief word on thread-safety (fuller detail in file 08)
None of `ArrayList`, `HashMap`, `HashSet` etc are safe for simultaneous modification by multiple threads — using them that way risks the same kind of corruption `ConcurrentModificationException` guards against, or worse, silent data corruption with no exception at all. Java provides purpose-built concurrent alternatives (`ConcurrentHashMap`, `CopyOnWriteArrayList`) for exactly that situation — worth knowing they exist now, covered properly once file 08's concurrency fundamentals are in place.

## Generics — why introduced (Java 5, 2004)
**Before generics**: collections stored plain `Object` type (no type info). Retrieving required manual casting, and WRONG casts only failed at RUNTIME (`ClassCastException`), often far from the actual mistake.
```java
List names = new ArrayList();   // pre-generics style
names.add("John");
names.add(123);                 // allowed! no compile-time check
String s = (String) names.get(1); // crashes at RUNTIME — bug found too late
```
**Generics' solution**: declare the intended type upfront (`List<String>`), compiler enforces it at COMPILE time, catching type mistakes immediately instead of at runtime, in production. Direct continuation of Java's core philosophy: catch errors as early as possible.
```java
List<String> names = new ArrayList<>();
names.add(123); // COMPILE ERROR now, caught immediately
```

### Bounded type parameters and wildcards — why they exist beyond simple `<T>`
Sometimes a generic method needs to guarantee the type parameter supports SOME capability (e.g. being compared), not just be "any type." A bounded type parameter (`<T extends Comparable<T>>`) restricts what `T` can be, while still keeping compile-time type safety:
```java
<T extends Comparable<T>> T max(List<T> list) {
    T maxVal = list.get(0);
    for (T item : list) if (item.compareTo(maxVal) > 0) maxVal = item;
    return maxVal;
}
```
Wildcards (`List<? extends Number>`) solve a related but different problem: accepting a collection of an UNKNOWN-but-bounded type, useful when writing a method that should work with `List<Integer>`, `List<Double>`, etc, without needing a separate overload for each.
