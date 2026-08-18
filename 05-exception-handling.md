# 05 — Exception Handling

## Why exceptions came to exist (replacing error codes)

Older languages (C) had no built-in error-handling mechanism — functions returned special "error codes" (e.g. `-1`, `NULL`) that the CALLER had to remember to check every single time. Real-world consequence: developers routinely FORGOT to check return codes, errors silently ignored, programs crashed or behaved unpredictably far from the actual bug's source, very hard to trace.

**Java's solution**: exceptions — a separate, structured "error channel" that CANNOT be silently ignored. When something goes wrong, code "throws" an exception, which immediately interrupts normal flow and propagates UP the call stack until some code explicitly "catches" and handles it (or the program terminates with a clear stack trace pointing at the exact failure point).

## The exception class hierarchy — why it's shaped this way

Everything throwable in Java descends from one root class, `Throwable`, which splits into two very differently-intentioned branches:
```
Throwable
├── Error            — serious problems the JVM itself encounters (OutOfMemoryError, StackOverflowError).
│                       NOT meant to be caught/handled by normal application code — if the JVM itself is in trouble,
│                       your catch block usually can't meaningfully fix it.
└── Exception         — problems application code IS expected to potentially handle
    ├── checked exceptions (e.g. IOException, SQLException) — direct subclasses of Exception
    └── RuntimeException (unchecked) — NullPointerException, ArrayIndexOutOfBoundsException, IllegalArgumentException, etc
```
Knowing this shape explains a lot of behavior at once: why catching generic `Exception` also silently catches `RuntimeException` (it's a subclass), why catching `Throwable` is almost always wrong (it would also swallow serious `Error`s you can't actually recover from), and why the checked/unchecked split (below) is really just "is this class `Exception` directly/checked-subclass, or is it under `RuntimeException`."

## Why checked vs unchecked exceptions — a deliberate two-tier design

Java's designers noticed two very different categories of "things going wrong":
1. **External, expected-possible failures** — file might not exist, network might be down, database might be unreachable. These are OUTSIDE your code's control, and a responsible caller SHOULD be forced to consider handling them.
2. **Programmer bugs** — null reference used, array index out of bounds. These are INTERNAL logic errors, not external conditions — forcing every method to declare "might throw NullPointerException" everywhere would be absurd noise.

**Checked exceptions** (`IOException`, `SQLException`) — compiler FORCES you to either catch or declare (`throws`) them. Represents category 1.
**Unchecked exceptions** (`RuntimeException` and subclasses like `NullPointerException`, `ArrayIndexOutOfBoundsException`) — no compiler enforcement. Represents category 2 — expected to be fixed as bugs, not routinely "handled."

This distinction is controversial even among Java experts (some say checked exceptions became overused/annoying in practice) but the ORIGINAL reasoning was sound: force handling of genuinely expected external failure modes.

**Why the controversy exists, in a bit more depth**: in large codebases, checked exceptions can "leak" upward through many layers of methods that don't actually know how to handle them, forcing every intermediate method to add a `throws` clause purely to satisfy the compiler — bloating signatures without adding real safety. This is a big part of WHY many modern frameworks (Spring itself, file 11-14) deliberately convert checked exceptions into UNCHECKED ones at their API boundaries (e.g. Spring's `DataAccessException` hierarchy wraps checked `SQLException`s as unchecked exceptions) — a real, practical design reaction you'll see directly once you reach the Spring files.

## Practical usage
```java
try {
    int[] arr = {1,2,3};
    System.out.println(arr[5]);
} catch (ArrayIndexOutOfBoundsException e) {
    System.out.println("Error: " + e.getMessage());
} finally {
    System.out.println("Always runs — cleanup code goes here");
}
```
`finally` — why it exists: guarantees cleanup code (closing files, releasing locks, closing DB connections) runs REGARDLESS of whether an exception occurred or not, and regardless of whether the try block succeeded or an exception was caught — critical for resource management (a file left open due to a skipped cleanup step is a real, damaging bug class).

### Multi-catch (Java 7+) — why added
Before it, handling several DIFFERENT exception types with IDENTICAL handling logic meant either duplicating the same catch-block body multiple times, or catching an overly-broad common parent type (losing precision about what actually went wrong). Multi-catch lets one block handle several unrelated exception types explicitly, without either downside:
```java
try {
    // risky code
} catch (IOException | SQLException e) {
    System.out.println("Failed: " + e.getMessage());   // shared handling, but still precise about types caught
}
```

### Reading a stack trace — why it's shaped the way it is
When an exception isn't caught anywhere, the JVM prints a **stack trace**: the exception type and message, followed by a list of "at ClassName.methodName(File.java:line)" entries, ordered from where the exception was actually thrown UP through every method call that led there. This ordering exists because it mirrors the call stack itself (file 01/02) — reading top to bottom shows you the most specific failure point first, then progressively "zooms out" to how execution got there, letting you jump straight to your OWN code's line number even inside deep library call chains.

### Exception chaining — why a "cause" matters
Sometimes a low-level exception (a raw `SQLException`) needs to be reported as a higher-level, more meaningful one (a custom `UserNotFoundException`) without losing the ORIGINAL technical detail — useful for debugging while still exposing a clean, domain-relevant exception to callers. Java's exception constructors support wrapping an original "cause":
```java
try {
    // some JDBC call throws SQLException
} catch (SQLException e) {
    throw new RuntimeException("Failed to load user", e);   // 'e' preserved as the cause, visible in the full stack trace
}
```

### try-with-resources (Java 7+) — why added
Before this, closing a resource properly required verbose, error-prone manual `finally` blocks (and a subtle historical bug: if BOTH the try block AND the finally-close threw exceptions, the original real error could get silently swallowed/replaced). try-with-resources auto-closes any resource implementing `AutoCloseable`, correctly, concisely, and fixes that swallowing bug.
```java
try (FileReader fr = new FileReader("file.txt")) {
    // use fr
} catch (IOException e) {
    e.printStackTrace();
}
// fr auto-closed here, even on exception
```
Multiple resources can be declared in the same try-with-resources (semicolon-separated) — they're closed automatically in REVERSE order of declaration, mirroring how you'd manually close nested resources by hand (close the thing that depends on another BEFORE closing what it depends on).

### Custom exceptions — why you'd create your own
Built-in exceptions are generic. Real applications benefit from exceptions that describe DOMAIN-SPECIFIC problems clearly (`InsufficientFundsException` communicates intent far better than a generic `RuntimeException` with a message buried inside).
```java
class InsufficientFundsException extends Exception {
    InsufficientFundsException(String message) { super(message); }
}
void withdraw(double amount, double balance) throws InsufficientFundsException {
    if (amount > balance) throw new InsufficientFundsException("Not enough balance");
}
```
**Choosing checked vs unchecked for a custom exception**: extend `Exception` when the failure is genuinely an expected, recoverable, external condition the caller should be forced to consider (matches category 1 above). Extend `RuntimeException` when it represents a programming/business-rule violation that's more of a bug signal than an expected everyday occurrence — this is also why most modern Spring/web-layer custom exceptions (file 12's `@ControllerAdvice` pattern) are unchecked: the framework wants them to propagate freely up to one central handler without every intermediate layer needing a `throws` declaration.

### Anti-pattern to explicitly warn against: empty catch blocks
```java
catch (Exception e) { }  // NEVER — silently swallows the exact problem exceptions were designed to prevent
```
This recreates the EXACT problem exceptions were invented to solve (silently ignored errors) — explain this irony directly, makes the lesson stick. At an absolute minimum, log the exception (`e.printStackTrace()` for learning purposes; a proper logging framework in real projects) so the failure is at least visible somewhere.

Exceptions were invented to **make failures visible** and force developers to deal with them. But when you write the above code. You’re doing the exact opposite: you’re **silently swallowing the error**. The program continues as if nothing happened, even though something went wrong. This recreates the very problem exceptions were designed to solve — hidden failures.
###### 🔎 Why It’s Harmful
- **Debugging nightmare**: The program misbehaves, but you have no clue why, because the error vanished.
- **Data corruption risk**: If the exception was about I/O, transactions, or state changes, ignoring it can leave your system in an inconsistent state.
- **Security issues**: Silently ignoring exceptions can mask authentication/authorization failures or input validation errors.
- **Maintenance hazard**: Future developers (including you) won’t know an error occurred at all.
### Rethrowing and `throws` declarations — why a method signature can advertise failure
A method that can't itself meaningfully handle a checked exception (e.g. a low-level data-access method has no sensible way to "recover" from a database being down) is allowed to simply declare `throws SQLException` and let the caller — who may have more context — decide how to handle it. This is a deliberate design choice: not every layer of a program is the RIGHT layer to handle every error, and Java's checked-exception system lets that responsibility be explicitly passed upward rather than forcing a premature, poorly-informed catch too early.
