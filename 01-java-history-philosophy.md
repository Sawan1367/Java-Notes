# 01 — Java History & Philosophy

## Why Java came to exist

Early 1990s, Sun Microsystems team (James Gosling and others) worked on project for embedded consumer devices (set-top boxes, appliances). Existing language C++ caused problems:
- Manual memory management → frequent crashes, memory leaks
- Platform-dependent compiled code → program compiled for one device's chip wouldn't run on another
- Complex features (multiple inheritance, pointers) → more bugs, harder to secure

Team designed new language, originally called "Oak," goal: **write once, run on any device**, safer (no raw pointers), simpler than C++. Internet grew explosively mid-90s — Java's portability made it perfect for web applets too. Renamed "Java" (1995), released publicly, became dominant enterprise language due to reliability + portability + huge standard library.

**Core design philosophy carried forward into every Java concept you'll learn**:
1. Portability first (bytecode + JVM)
2. Safety over raw power (no manual pointers, automatic garbage collection)
3. Simplicity over cleverness (single inheritance for classes, explicit typing)
4. "Batteries included" (huge standard library, avoid reinventing wheels)

Keep referring back to this philosophy — almost every Java design decision (why no multiple class inheritance, why strong typing, why checked exceptions) traces back to these four principles.

Java is a **high-level, object-oriented programming language** developed by Sun Microsystems in 1995, spearheaded by James Gosling. It is widely used for building **desktop applications, web applications, mobile apps (especially Android), enterprise systems, and more**.

Java is renowned for its **platform independence**, encapsulated in the phrase **"Write Once, Run Anywhere" (WORA)**. This is achieved through the use of **bytecode**, which is executed by the **Java Virtual Machine (JVM)**, making Java programs portable across different operating systems.

Key Features of Java : -
- **Object-Oriented**: Everything in Java is treated as an object, promoting clean and reusable code.
- **Platform Independent**: Java programs can run on any device with a JVM installed.
- **Simple and Secure**: Java eliminates complex features like pointers and includes built-in security mechanisms.
- **Multithreading**: Java supports concurrent execution of tasks, making it suitable for complex applications.
- **Robust and Reliable**: Java emphasizes error checking at both compile-time and runtime.
- **Dynamic and Distributed**: Java adapts to evolving environments and supports distributed computing.

## Java's version history — why it matters, not just trivia

Understanding the timeline explains why certain features exist in certain files of these notes, and clears up "why does this tutorial online look different from what I'm learning":

| Version     | Year | Why it mattered                                                                                       |
| ----------- | ---- | ----------------------------------------------------------------------------------------------------- |
| Java 1.0    | 1996 | first public release                                                                                  |
| Java 5      | 2004 | Generics, enums, annotations, autoboxing, enhanced for-loop — biggest syntax shift until Java 8       |
| Java 7      | 2011 | try-with-resources, diamond operator `<>`, multi-catch                                                |
| Java 8      | 2014 | Lambdas, Streams, Optional, default interface methods — huge functional-programming shift (file 07)   |
| Java 9      | 2017 | Module system (Project Jigsaw), private interface methods, JShell (REPL)                              |
| Java 10     | 2018 | `var` local type inference                                                                            |
| **Java 11** | 2018 | **LTS** (Long-Term Support) — first LTS after 8, `var` in lambdas, HTTP Client API                    |
| Java 14/15  | 2020 | Switch expressions, text blocks (`"""`), records preview                                              |
| Java 16/17  | 2021 | Records finalized, sealed classes, pattern matching for `instanceof` — **17 is LTS**                  |
| Java 21     | 2023 | Virtual threads (Project Loom, file 08), pattern matching for switch, sequenced collections — **LTS** |

**Why LTS (Long Term Support) releases matter**: Oracle/OpenJDK release a new Java version every 6 months, but most only get security patches for a short window. LTS versions (8, 11, 17, 21) get years of patches — this is why production systems and companies almost always standardize on an LTS version, not the newest release. When starting out, install 17 or 21.

**Java SE vs EE (now Jakarta EE) vs ME** — why the split exists: Java began targeting many different contexts (desktop, servers, mobile/embedded) with different needs. **Java SE** (Standard Edition) = the core language + libraries everyone uses (what these notes teach). **Java EE** (Enterprise Edition, since donated to the Eclipse Foundation and renamed **Jakarta EE**) = extra specifications for large enterprise servers (this is the EJB world Spring reacted against, file 11). **Java ME** (Micro Edition) = stripped-down version for old feature-phones/embedded devices, largely obsolete today (Android uses its own separate runtime, not Java ME).

---

## Why JVM, JDK, JRE exist as separate things

**Problem before Java**: compile C++ code on Windows → binary only runs on Windows. Need separate compiled binary per OS. Expensive, slow, error-prone for companies shipping cross-platform software.

**Java's solution**: two-step translation.
```
Source code (.java) --[compiler: javac]--> Bytecode (.class) --[JVM, per-OS]--> Native machine code
```
Compiler produces ONE universal bytecode format, regardless of OS. Each OS gets its OWN JVM implementation that knows how to run that same bytecode on that specific machine. So: compile once, JVM per platform handles rest. "Write once, run anywhere" (WORA) — Java's most famous selling point, direct answer to the portability problem above.

- **JVM (Java Virtual Machine)**: the actual engine that loads bytecode, verifies it (security — checks it's not malicious/corrupt), interprets/compiles it to native instructions (using JIT — Just-In-Time compiler, converts hot/frequently-run bytecode to native code for speed), and manages memory (garbage collection).
- **JRE (Java Runtime Environment)**: JVM + core class libraries (String, collections, I/O, etc) needed to RUN a compiled Java program. Ship this if you're only distributing an app to END USERS who just run it. (Note: as a standalone separate download, JRE was discontinued after Java 8 — modern JDKs let you build a stripped-down custom runtime with `jlink` instead, but the JRE *concept* — "just enough to run, not to build" — still applies.)
- **JDK (Java Development Kit)**: JRE + development tools (`javac` compiler, debugger, documentation generator `javadoc`, packaging tool `jar`, etc). Needed if you're WRITING and BUILDING Java code.

Analogy: JDK = full workshop (build + use tool). JRE = just the finished tool (use only). JVM = the actual mechanism inside the tool doing the work.

## Why bytecode, not direct machine code

Bytecode is intermediate — not tied to any specific CPU architecture. JVM also VERIFIES bytecode before running (checks for illegal memory access, stack corruption) — a security layer impossible if compiling straight to native machine code. This verification step is part of why early Java Applets (running inside browsers) were considered safer than native plugins.

### JIT compilation — why bytecode isn't just slowly interpreted forever
Pure interpretation (reading bytecode instruction-by-instruction every single time) is slow. Compiling everything to native code upfront (ahead-of-time) loses portability and startup speed. JVM's compromise: **interpret bytecode initially** (fast startup, stays portable) while a background **JIT (Just-In-Time) compiler** watches for "hot" code — methods/loops run very frequently — and compiles JUST those into optimized native machine code on the fly, replacing the interpreted version mid-execution. This is why long-running Java programs (servers) actually get FASTER the longer they run — the JIT has had time to identify and optimize hot paths. (Modern JVMs use tiered compilation: a quick, less-optimizing C1 compiler first, then a slower, more-optimizing C2 compiler for the hottest code.)

### JVM memory areas — why more than one region exists
- **Heap**: where all objects live (file 02's reference types); shared across all threads; this is what the Garbage Collector manages.
- **Stack** (one per thread): stores method call frames, local variables, primitives — fast, automatically cleaned up when a method returns.
- **Metaspace** (replaced "PermGen" since Java 8): stores class metadata itself (the loaded `.class` structure info), grows into native OS memory instead of a fixed-size heap region — this change specifically fixed a common `OutOfMemoryError: PermGen space` problem from older Java versions caused by loading too many classes (common in app servers reloading web apps repeatedly).

## Practical setup (do this, no external search needed)
1. Download JDK (version 17 or 21 — these are LTS: Long Term Support releases, meaning Oracle/OpenJDK guarantee years of security patches, safest choice for learning/production)
2. Install IntelliJ IDEA (Community Edition, free) — an IDE (Integrated Development Environment): editor + compiler integration + debugger + project management in one tool. Exists because writing/compiling/running/debugging as SEPARATE manual steps (like early programmers did with plain text editors + command line) was slow and error-prone. (Alternatives worth knowing exist: Eclipse — older, still widely used in some enterprises; VS Code with the Java extension pack — lighter weight, popular for polyglot developers.)
3. First program:
```java
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello World");
    }
}
```
- `main` method — JVM specifically looks for this EXACT signature as program's entry point. Why this exact form: `public` (JVM from outside must access it), `static` (JVM calls it without needing to create an object first — object creation happens later, chicken-and-egg problem avoided), `void` (program start doesn't need to return a value to JVM), `String[] args` (allows passing command-line arguments into the program, a carryover convention from C).

Common first mistake (show live): public class name MUST exactly match filename (`HelloWorld.java`). Java enforces this so the compiler/JVM can locate the right file for a given public class without extra configuration — simplicity/convention over configuration, a philosophy repeated later in Spring Boot.

### What actually happens when you run it, command line (no IDE) — worth showing once
```bash
javac HelloWorld.java   # compiler: source -> HelloWorld.class (bytecode)
java HelloWorld          # JVM: loads HelloWorld.class, runs main()
```
Seeing this manual two-step process once demystifies everything the IDE's "Run" button silently automates — a small but valuable "lift the hood" moment for a beginner.

### Comments and Javadoc — why documentation is baked into the language
Code explains HOW; comments explain WHY (mirroring this whole notes philosophy). Java has three comment forms: `// single line`, `/* multi line */`, and `/** Javadoc */` — the last is special: tools like `javadoc` parse these specially-formatted comments and auto-generate browsable HTML API documentation, which is exactly how official Java API documentation itself is produced. This exists because large teams/libraries need documentation that stays physically attached to (and versioned with) the code it describes, rather than a separate document that silently goes stale.
