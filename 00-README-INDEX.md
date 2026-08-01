# Java + Spring Boot — Concept Notes (Index)

Each file below = one topic area. Each concept inside answers three questions always:
**Why did this come to exist (problem it solved)** → **What it is** → **How to use it practically (code)**.

Read in order for a course. Use individually as reference after. Every file now also includes a deeper "extra depth" layer beyond the original core lesson — additional sub-topics, edge cases, gotchas, and newer-Java-version features — so each file can serve as a genuinely complete reference on its topic, not just an intro. Core concepts and extra-depth sections are structured the same way (why → what → how) so the deeper material never feels bolted-on; treat the extra sections as optional-but-available depth to pull in once the core of a file is solid (see file 15's teaching-method notes for pacing guidance).

1. `01-java-history-philosophy.md` — Java origin, JVM/JDK/JRE, why Java designed this way, full version-history timeline (LTS releases), JIT compilation, JVM memory areas, SE/EE/ME editions
2. `02-core-basics.md` — variables, types, operators, control flow, arrays, strings, methods, plus wrapper classes/autoboxing, `final`, `var`, text blocks, Scanner, varargs, labeled break/continue
3. `03-oop-fundamentals.md` — class/object, four pillars, why OOP replaced procedural style, plus `this`/`super`, constructor chaining, nested classes, overriding rules, `instanceof` pattern matching, enums, records, sealed classes
4. `04-interfaces-abstraction.md` — interface vs abstract class, why multiple inheritance banned then reintroduced, plus interface constants, default-method diamond resolution, private interface methods, marker interfaces, sealed interfaces, module system
5. `05-exception-handling.md` — why exceptions replaced error codes, plus the full Throwable hierarchy, multi-catch, stack traces, exception chaining, choosing checked vs unchecked for custom exceptions
6. `06-collections-framework.md` — why collections replaced raw arrays, internal structure, plus Queue/Deque/PriorityQueue, useful Map convenience methods, utility classes, immutable collection factories, generics wildcards/bounds
7. `07-java8-functional-features.md` — why lambdas/streams added, functional programming shift, plus variable capture rules, method references, Collectors deep dive, intermediate-vs-terminal/laziness, Optional chaining
8. `08-multithreading-concurrency.md` — why concurrency needed, problems it causes, plus thread lifecycle, deadlock example, `volatile`/atomics, `CompletableFuture`, concurrent collections, virtual threads (Java 21)
9. `09-jdbc-file-io.md` — connecting Java to files and databases, plus NIO.2, serialization, Statement variants, batch updates, transactions, connection pooling
10. `10-backend-fundamentals.md` — HTTP, REST, JSON, SQL, architecture — history and reasoning, plus idempotency, headers/cookies/CORS, API versioning, HATEOAS, normalization, indexing, ACID
11. `11-spring-framework-history-di.md` — why Spring was born (reaction against EJB), IoC/DI deep dive, plus `@Qualifier`/`@Value`, bean scopes, profiles, AOP, circular dependencies, lifecycle hooks
12. `12-spring-boot-rest-apis.md` — why Spring Boot was created, building REST APIs, plus embedded server rationale, `ResponseEntity`, `@ControllerAdvice`, `@Valid` validation, Actuator, OpenAPI/Swagger
13. `13-spring-data-jpa.md` — why ORM exists, JPA/Hibernate reasoning, plus JPQL/`@Query`, lazy vs eager fetching, the N+1 problem, pagination, cascading, `@Transactional`, entity lifecycle states
14. `14-spring-security-testing.md` — auth history, JWT reasoning, testing philosophy, plus filter chain, password encoding, CSRF, role vs permission auth, test slices, MockMvc, Testcontainers
15. `15-practice-projects-roadmap.md` — project sequence + full teaching roadmap timing, plus stretch goals per project, additional project ideas, per-week checkpoint questions
16. `16-advanced-java-jvm-internals.md` — deep `int` vs `Integer` (autoboxing, Integer cache, unboxing NPEs), JVM/classloading/JIT/Metaspace internals, GC algorithms (G1/ZGC), generics & type erasure, reflection, annotations, `volatile`/Executor framework/`CompletableFuture`/`ThreadLocal`/virtual threads, core design patterns
17. `17-system-design-scalability.md` — vertical vs horizontal scaling, load balancing (algorithms, L4 vs L7), caching (strategies, eviction, Redis, invalidation), API Gateway & rate limiting, message queues (Kafka vs RabbitMQ, queue vs pub/sub), CDN, DB scaling (replication, sharding, consistent hashing, CAP theorem), microservices (service discovery, circuit breaker, saga pattern)
