# 02 — Core Java Basics

## Variables & Data Types

### Why static typing exists
Java requires declaring type upfront (`int age = 25;`) unlike dynamically typed languages (Python/JS decide type at runtime). Why: Java designers prioritized catching errors AT COMPILE TIME, before program even runs — safer for large enterprise systems where a runtime crash in production is expensive. Trade-off: more verbose code, but far fewer "surprise" bugs.

### Why 8 separate primitive types instead of one generic "number"
Different types = different memory footprint and value range, chosen deliberately for performance/memory efficiency. A counter that never exceeds 100 doesn't need 8 bytes (long) — using `byte` saves memory at scale (matters when you have millions of objects).

| Type      | Size               | Why it exists                                                                                                                                                                       |
| --------- | ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `byte`    | 1B                 | tiny values, memory-critical contexts (large arrays, embedded)                                                                                                                      |
| `short`   | 2B                 | rarely used now, legacy from C-era memory constraints                                                                                                                               |
| `int`     | 4B                 | default whole-number choice, balance of range and memory                                                                                                                            |
| `long`    | 8B                 | when int's ~2 billion range insufficient (timestamps, big IDs)                                                                                                                      |
| `float`   | 4B                 | decimal, less precision, used when memory matters more than precision                                                                                                               |
| `double`  | 8B                 | decimal, default choice — most decimal literals in Java are double by default                                                                                                       |
| `char`    | 2B                 | single character — 2 bytes because Java uses Unicode (supports all world languages), not just ASCII (1 byte, English-only) — a portability decision matching Java's WORA philosophy |
| `boolean` | 1 bit conceptually | true/false only, exists for readable conditional logic                                                                                                                              |

### Why reference types are different from primitives
**Problem**: objects (String, custom classes, arrays) can be large, complex, variable-sized. Storing a full copy every time you pass them around would be slow and wasteful.
**Solution**: reference types store a memory ADDRESS (pointing to actual data in heap), not the data itself. Passing a reference = passing a small address, not copying a huge object.

```java
int x = 10;              // primitive: value directly in stack
String name = "John";    // reference: address in stack, "John" text lives in heap
```

**Stack vs Heap** — why two separate memory areas exist:
- **Stack**: fast, small, automatically managed (LIFO — last in first out, tied to method calls). Method ends → its stack frame vanishes automatically. No cleanup cost.
- **Heap**: bigger, holds objects that may need to live LONGER than the method that created them (e.g. an object returned from a method, or shared across methods). Needs a smarter cleanup mechanism — this is why Garbage Collector exists (see below).

### Why Garbage Collection exists
**Problem in C/C++**: programmer manually allocates and frees memory (`malloc`/`free`). Forget to free → memory leak. Free too early/twice → crash (dangling pointer). This was THE most common source of serious bugs and security vulnerabilities in older languages.
**Java's solution**: Garbage Collector (GC) — background process that automatically detects objects no longer reachable/used, reclaims their memory. Programmer never manually frees memory. Trade-off: slight unpredictable pause (GC pause) vs eliminated entire category of bugs. Direct continuation of Java's "safety over raw performance" philosophy.

**How GC decides what's "garbage"**: an object is eligible for collection once nothing in the program can reach it anymore (no live reference chain from any running thread's stack down to that object). Setting a reference to `null`, or letting a local variable go out of scope, are the two most common ways an object becomes unreachable. You can never force immediate collection — `System.gc()` only *suggests* the JVM run GC soon, it's not a guarantee, which is deliberate: giving programmers manual control over collection timing would reopen the exact unpredictability problems GC exists to remove.

### Wrapper classes & autoboxing — why they exist
Primitives aren't objects — they can't be stored in Collections (file 06, which store `Object` references only), can't be `null` (a primitive `int` always has SOME value, never "no value"), and have no methods of their own. Java provides a wrapper CLASS for each primitive (`int`→`Integer`, `double`→`Double`, `boolean`→`Boolean`, etc) — full objects, usable anywhere an `Object` is required, and capable of being `null` (representing "no value present," useful for optional/missing data from a database column, for example).

```java
int primitive = 5;
Integer wrapped = 5;          // autoboxing: compiler auto-wraps primitive into Integer
int backAgain = wrapped;      // auto-unboxing: compiler auto-unwraps back to primitive
List<Integer> nums = new ArrayList<>();  // Collections can ONLY hold objects, hence Integer not int
```
**Autoboxing pitfall worth knowing**: `Integer` caches small values (-128 to 127) for efficiency, so `==` comparison between two boxed `Integer` objects can misleadingly return `true` for small numbers and `false` for larger ones — another reason (alongside String, file 02 above) to prefer `.equals()` over `==` for object comparison.

### Type casting — why needed
Sometimes you need to convert between types (int to double, double to int). 
- **Implicit/widening** (small → big): safe, automatic, no data lost — compiler does it silently: `int i = 10; double d = i;`
- **Explicit/narrowing** (big → small): risky, possible data loss — compiler FORCES you to write it explicitly (`(int) d`), so YOU consciously accept the risk, not an accident:
```java
double d = 9.7;
int i = (int) d; // 9, decimal truncated — YOU chose this, compiler warned you by requiring the cast
```

### `final` keyword — why it exists
Sometimes a value or reference should be LOCKED after initial assignment — a config constant, a value that must never accidentally change mid-program. `final` on a variable makes reassignment a COMPILE error, catching an entire class of accidental-overwrite bugs before the program ever runs.
```java
final double PI_APPROX = 3.14159;
// PI_APPROX = 3.14; // COMPILE ERROR — can't reassign a final variable
```
`final` also applies to methods (cannot be overridden by subclasses, file 03) and classes (cannot be extended at all — `String` itself is a `final` class, precisely so no one can create a rogue subclass that breaks the immutability/security guarantees described later in this file).

### `var` keyword — why added later (Java 10)
Reduces boilerplate for obvious types: `var list = new ArrayList<String>();` instead of repeating `ArrayList<String>` twice. Java stayed statically typed underneath (type still fixed at compile time) — this is just SYNTAX sugar, not a shift to dynamic typing, keeping safety while reducing verbosity complaint that pushed some developers toward other languages. `var` can only be used for LOCAL variables with an immediately-obvious initializer (never for fields, method parameters, or return types) — a deliberate restriction so type inference never obscures a public API's contract, only reduces noise in method-internal code.

---

## Operators & Control Flow

### Why so many operator categories
Arithmetic (math), relational (compare, produce boolean), logical (combine booleans), assignment (store/update) — each maps to a distinct kind of real-world decision-making need. Ternary (`cond ? a : b`) exists purely as a compact shorthand for simple if-else, reduces verbose code for trivial decisions.

### Full operator reference
| Category | Operators | Why |
|---|---|---|
| Arithmetic | `+ - * / % ++ --` | basic math; `%` (modulo) exists for "remainder" problems (even/odd checks, cyclic indexing) |
| Relational | `== != > < >= <=` | comparisons, always produce `boolean` |
| Logical | `&& \|\| !` | combine boolean expressions; `&&`/`\|\|` are "short-circuit" — stop evaluating as soon as the result is certain, both a performance optimization and a safety feature (e.g. `obj != null && obj.value > 0` never touches `.value` if `obj` is null) |
| Bitwise | `& \| ^ ~ << >> >>>` | operate on raw bits directly — rare in everyday app code, but essential for flags/masks, low-level performance code, and some legacy/embedded contexts |
| Assignment | `= += -= *= /= %=` | compound forms exist purely as shorthand (`x += 5` same as `x = x + 5`) |

### Integer division trap — why it happens
```java
System.out.println(5 / 2); // 2, not 2.5
```
Because both operands are `int`, Java assumes you want INTEGER result (matches primitive type rules — no automatic promotion unless one operand is already decimal). This isn't a "bug" — it's a deliberate consistency rule: operation between two ints always stays int, no silent implicit widening.

### Why `if/else`, `switch`, and loops all separately exist
- `if/else` — flexible, arbitrary boolean conditions
- `switch` — cleaner when comparing ONE variable against MANY discrete values (readability over chained if-else-if)
- Loops — needed because programs must repeat actions without rewriting code N times. `for` when iteration count known upfront, `while` when condition-based/unknown count, `do-while` when action must run at least once regardless of condition (e.g. "show menu once, then check if user wants to continue").

Modern switch (Java 14+, arrow syntax) added specifically to fix a long-standing complaint: classic switch's fall-through behavior (forgetting `break`) was a notorious bug source. New syntax removes fall-through by default, and can also be used as an EXPRESSION (returns a value directly), not just a statement:
```java
// Classic switch — fall-through risk if 'break' forgotten:
switch (day) {
    case 1: System.out.println("Mon"); break;
    case 2: System.out.println("Tue"); break;
    default: System.out.println("Other");
}
// Modern switch expression (Java 14+) — no fall-through, can return a value directly:
String name = switch (day) {
    case 1 -> "Mon";
    case 2 -> "Tue";
    default -> "Other";
};
```

### The enhanced for-loop ("for-each") — why added
Classic indexed loops (`for (int i = 0; i < arr.length; i++)`) require manually managing an index variable — an easy source of off-by-one bugs (`<=` instead of `<`, wrong start value). When you only need to VISIT each element (not track/use its index), the enhanced for-loop removes the index entirely, eliminating that bug class:
```java
int[] marks = {90, 85, 70};
for (int m : marks) {           // "for each int m in marks"
    System.out.println(m);
}
```
Trade-off: no access to the index itself and no way to modify the underlying collection structure mid-loop — use the classic form when you genuinely need the index or must remove/insert during iteration.

### Labeled break/continue — why they exist
A plain `break`/`continue` only affects the INNERMOST enclosing loop. Nested loops (a grid search, matrix processing) sometimes need to break out of an OUTER loop from inside an inner one — without a label, you'd need an awkward extra boolean flag checked in every iteration. A label removes that workaround:
```java
outer:
for (int i = 0; i < 5; i++) {
    for (int j = 0; j < 5; j++) {
        if (i * j > 6) break outer;   // exits BOTH loops directly
    }
}
```

---

## Arrays & Strings

### Why arrays exist
Before arrays, storing 100 related values meant 100 separate variables — unmanageable. Array = single named block of contiguous memory, indexed access, fixed size decided upfront (this fixed-size constraint is exactly why Collections framework was created later — see file 06).

```java
int[] marks = {90, 85, 70, 60};
System.out.println(marks[0]); // 90, 0-based indexing (inherited convention from C)
```
Zero-based indexing exists because index technically represents "offset from start address" — element 0 is AT the start (0 steps away), not "1st item." This is a low-level memory-layout convention Java inherited from C.

### Multidimensional arrays — why they exist
Real data is often naturally grid-shaped (a chessboard, a spreadsheet, an image's pixels). A 2D array is Java's direct model for this: technically, "an array of arrays" — each row is itself a separate array object, which also means rows can (unusually) have different lengths ("jagged arrays") if needed.
```java
int[][] grid = {
    {1, 2, 3},
    {4, 5, 6}
};
System.out.println(grid[1][2]); // 6 — row 1, column 2
```

### Why String is a class, not a primitive
Text is variable-length and complex — doesn't fit the fixed-size primitive model. String needed to be an object (reference type) holding character data on the heap.

### Why String is immutable — deliberate design decision, not accident
Reasons Java designers chose immutability:
1. **Security**: Strings used for class names, file paths, network hosts, DB URLs. If mutable, malicious code could alter a String after a security check but before actual use (a real attack vector in early Java). Immutability closes this gap.
2. **Thread safety**: immutable object can be freely shared across threads with zero synchronization needed (nothing can change, no race condition possible) — matters heavily once Java's multithreading model is used (file 08).
3. **String Pool / memory efficiency**: since Strings can't change, Java can safely cache and REUSE identical String literals instead of creating duplicates.

```java
String a = "hello";
String b = "hello";
System.out.println(a == b);   // true — both reused from same pooled object

String c = new String("hello");
System.out.println(a == c);   // false — new forces a fresh heap object, bypassing pool
System.out.println(a.equals(c)); // true — content comparison
```
**Golden practical rule**: never use `==` for String content comparison — it compares memory addresses (a leftover consequence of String being a reference type), always use `.equals()`.

### Common String methods — why the API is this shape
Every one of these exists because it maps to a genuinely common real-world text operation, and (because String is immutable) EVERY one of them returns a NEW String rather than modifying the original:
```java
String s = "  Hello World  ";
s.trim();                 // remove leading/trailing whitespace — cleaning user input
s.toUpperCase();           // case-insensitive comparisons, display formatting
s.length();                // loop bounds, validation (e.g. password min length)
s.substring(2, 7);          // extract a portion — parsing fixed-format text
s.split(" ");               // break into an array — CSV parsing, tokenizing
s.replace("World", "Java"); // find-and-replace
s.contains("Hello");         // simple membership check, e.g. search filters
s.indexOf("World");          // position of a substring, or -1 if absent
```

### Text blocks (Java 15+) — why added
Before this, multi-line strings (JSON payloads, SQL queries, HTML snippets embedded in Java) required awkward manual `\n` and `+` concatenation across lines — hard to read, easy to introduce whitespace bugs. Text blocks let you write literal multi-line text directly:
```java
String json = """
    {
        "name": "John",
        "age": 25
    }
    """;
```

### Why StringBuilder exists
Since String is immutable, every concatenation in a loop (`s = s + "x"`) secretly creates a BRAND NEW String object each time — wasteful for many operations. StringBuilder was created as a MUTABLE companion class specifically for scenarios needing lots of modifications (loops), avoiding the repeated-object-creation cost.
```java
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++) sb.append(i); // one mutable buffer, not 1000 new String objects
```
(`StringBuffer` is an older, synchronized/thread-safe equivalent, predating `StringBuilder` — slower due to that synchronization overhead. Use `StringBuilder` by default unless the SAME builder instance is genuinely shared/mutated across multiple threads.)

---

## Reading input — Scanner class

### Why Scanner exists
A program that can only display fixed output isn't interactive. `Scanner` wraps a raw input source (commonly `System.in`, the keyboard) and provides convenient typed-parsing methods, so you don't have to manually read and parse raw bytes/characters yourself.
```java
import java.util.Scanner;
Scanner sc = new Scanner(System.in);
System.out.print("Enter your age: ");
int age = sc.nextInt();        // reads and parses an int directly
String line = sc.nextLine();   // reads a full line of text
sc.close();
```
Common gotcha worth demonstrating live: calling `nextInt()` then `nextLine()` back-to-back — `nextInt()` doesn't consume the trailing newline character, so the very next `nextLine()` call reads an empty leftover line instead of the next real input. Fix: add an extra `sc.nextLine()` to consume it, or use `nextLine()` consistently and parse manually (`Integer.parseInt(...)`).

---

## Methods

### Why methods exist
Code reuse (DRY — Don't Repeat Yourself) and organization — breaking large logic into named, testable, reusable chunks. Directly enables the OOP model covered next (methods become an object's "behaviors").

### Method overloading — why allowed
Lets the SAME logical operation ("add") work naturally across different input types/counts, without inventing awkward different names (`addInts`, `addDoubles`). Compiler resolves which version to call based on argument types at COMPILE time — this is why it's called "compile-time polymorphism."

### Varargs — why they exist
Sometimes the NUMBER of arguments a method should accept isn't known upfront (a `sum` method that might take 2 numbers or 10). Before varargs, you'd either overload dozens of versions or force callers to pass an array. Varargs (`...`) let a method accept a variable number of arguments while still calling it as if they were separate parameters:
```java
int sum(int... nums) {                 // treated internally as int[]
    int total = 0;
    for (int n : nums) total += n;
    return total;
}
sum(1, 2);        // works
sum(1, 2, 3, 4);  // also works, same method
```

### Why Java is strictly pass-by-value (a frequently misunderstood design choice)
Java NEVER passes a variable itself into a method — always a COPY of its value. For primitives, that copy is the actual number. For objects/arrays, that copy is the REFERENCE (address) — so the method can't reassign your original variable, but CAN modify the object the address points to.
```java
void changeValue(int x) { x = 100; }
int num = 5;
changeValue(num);
System.out.println(num); // 5 — untouched, copy was modified, not original

void changeArray(int[] arr) { arr[0] = 100; }
int[] myArr = {1,2,3};
changeArray(myArr);
System.out.println(myArr[0]); // 100 — the COPIED reference still points to the SAME heap object
```
Why designed this way: predictability. A method can never secretly reassign your variable out from under you — only mutate shared object contents if you explicitly allow access to that object. Reduces a class of subtle bugs common in pass-by-reference languages.

### Recursion — why it exists as an alternative to loops
Some problems are naturally SELF-similar/self-referential (tree traversal, factorial, Fibonacci, divide-and-conquer algorithms) — expressing them recursively often matches the problem's natural structure better than a loop, at the cost of extra memory (call stack) per recursive call.
```java
int factorial(int n) {
    if (n == 0) return 1;         // base case: MUST exist, else infinite recursion, StackOverflowError
    return n * factorial(n - 1);
}
```
Why `StackOverflowError` specifically: each recursive call adds a new frame to the (fixed-size, file 01) thread stack; a missing/incorrect base case means frames pile up forever until that fixed space runs out — a direct, practical consequence of the stack-memory model described earlier in this file.
