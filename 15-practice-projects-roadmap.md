# 15 — Practice Projects & Teaching Roadmap

## Why project-based practice matters (not just theory)
Concepts learned without application fade fast and stay shallow ("I know the definition" vs "I can use this to solve a problem"). Each project below deliberately targets the concepts from specific files above, in increasing difficulty, so every new topic gets applied almost immediately after learning it — spaced practice reinforces retention far better than passive reading/listening alone.

## Project sequence (build in order)
1. **Console Calculator** — targets file 02 (variables, operators, control flow, methods). Stretch goal once basic version works: support a running history of past calculations using an `ArrayList<String>` (early, informal preview of file 06, before collections are formally taught).
2. **Student Record Manager** — targets file 03 (OOP), file 06 (collections), file 09 (file I/O — save/load records). Stretch goal: add a custom `equals()`/`hashCode()` so duplicate student records can be detected via a `HashSet` (directly exercises file 03's Object-class section).
3. **Simple Bank App** — targets file 05 (exception handling), file 03 (encapsulation), custom exceptions. Stretch goal: add a simple transaction history log using `try-with-resources` file writes (file 05 + file 09 combined).
4. **Library Management System** — targets file 03-04 (inheritance, polymorphism, interfaces), file 06 (collections), file 09 (JDBC). Stretch goal: introduce a `Fine`/`Overdue` calculation using Java 8 date/time or Streams (early file 07 exposure, informally, before its dedicated lesson).
5. **REST API — Todo App** — targets file 12 (Spring Boot, full CRUD), file 13 (database persistence), DTO pattern. Stretch goal: add pagination (file 13) and basic input validation (`@Valid`, file 12) once the plain CRUD version works end-to-end.
6. **REST API — User Auth System** — targets file 14 (Spring Security, JWT), file 10 (layered architecture), validation. Stretch goal: add role-based authorization (`@PreAuthorize`) distinguishing an ADMIN role from a regular USER role.
7. **Capstone: Full Backend (e.g. E-commerce or Blog API)** — combines everything: entities with relationships (file 13), service layer (file 10/12), validation, global exception handling (file 05/12), security (file 14), testing (file 14). A strong capstone should also include: API documentation (Swagger/OpenAPI, file 12), a Postman collection demonstrating every endpoint, and at least one unit test plus one integration/slice test (file 14) per layer, to reinforce that testing is a habit applied throughout, not a bolt-on at the very end.

### Additional stretch project ideas (optional, once the core sequence is solid)
- **Multithreaded file processor** — targets file 08 directly: process a folder of files concurrently using an `ExecutorService`, compare timing against a single-threaded version to make the "why concurrency" motivation tangible rather than theoretical.
- **Weather/News aggregator console app** — targets file 07 (Streams for filtering/transforming API results) and Java's HTTP Client, consuming a real external REST API as a CLIENT (a useful complementary perspective to always building the SERVER side in files 12-14).
- **Second capstone variant, same requirements, different domain** — deliberately rebuilding the SAME architectural pattern (layered REST API + JPA + security + tests) in a different business domain (e.g. a bookstore instead of a blog) is one of the highest-value exercises for cementing the whole stack as a repeatable, transferable pattern rather than memorized steps tied to one specific example.

## Trial session plan (90 min, single friend, use files 01-03 as source material)
- 10 min — Phase 0 concept: what is programming, why Java (file 01 history section) — verbal + hook questions only, no code yet
- 15 min — file 01 (JVM/JDK/JRE) + live Hello World setup
- 20 min — file 02 (variables, types, operators) hands-on, predict-before-run exercises
- 20 min — file 02 (control flow) — build FizzBuzz together live
- 15 min — file 03 intro only (class vs object, blueprint/house analogy) — do NOT rush into all four pillars in one session
- 10 min — recap quiz: he explains the whole session back in his own words; assign homework (modify FizzBuzz rules + write one overloaded method)

## Full multi-week course timing (once trial validated, for future real batches)
- Week 1-2: files 01-04 (Java core + OOP + interfaces)
- Week 3-4: files 05-08 (exceptions, collections, functional features, concurrency)
- Week 5: file 09-10 (JDBC + backend fundamentals)
- Week 6-9: files 11-14 (Spring: DI, REST APIs, JPA, Security/Testing)
- Week 10-12: capstone project (item 7 above)

### Per-week checkpoint questions — a lightweight way to verify real understanding, not just coverage
At the end of each week block above, a good comprehension check is asking the student to answer WITHOUT notes:
- Weeks 1-2: "Why does Java only allow single class inheritance, and how do interfaces work around that limitation?"
- Weeks 3-4: "Why would you choose a `HashMap` over a `TreeMap`, and when would checked vs unchecked exceptions matter for a method you're writing?"
- Week 5: "Why does `PreparedStatement` prevent SQL injection when a plain `Statement` doesn't?"
- Weeks 6-9: "Trace what happens, layer by layer, from an incoming HTTP request to a database row and back, naming which annotation is responsible at each step."
- Weeks 10-12: capstone project itself IS the checkpoint — a working, tested, documented API is the proof of integrated understanding.

If a checkpoint answer is shaky, treat it as a signal to revisit that file's material with fresh examples before moving forward — the schedule above is a guideline, not a fixed deadline to push through regardless of comprehension.

## Teaching method — apply to EVERY topic, every file above
1. **Hook question first** — before defining a term, ask "what problem do you think this solves?" Let him guess before revealing the historical answer.
2. **Analogy before syntax** — map the concept to something real-world familiar first.
3. **Code live, with an intentional bug at least once per session** — ask him to spot it. Builds debugging instinct early, more memorable than a clean demo.
4. **He types, you narrate** — after your demo, hand him the keyboard for the next similar example.
5. **Micro-quiz every 15 minutes** — one verbal question, low-pressure, checks retention live rather than assuming it happened.
6. **End each topic with "explain it back to me in your own words"** — if he can't, the topic isn't actually done, regardless of how much content was "covered."

## Trainer-level notes (for scaling to real batches later)
- Keep a running "misconception log" across students — patterns of wrong answers reveal which explanations need reworking for future cohorts, this document evolves as YOU teach it more
- Record trial sessions (with consent) — review your OWN explanation clarity afterward, not just the student's understanding
- Every phase ends with build-something, never just study-something
- Use spaced repetition deliberately — when introducing a later file's concept, casually re-ask something from an earlier file that connects to it (files above are cross-referenced for exactly this purpose)
- As the notes themselves grow more detailed (files 01-14 now cover considerably more ground than a first pass), resist the urge to teach EVERY sub-point in a single sitting for any one file — use the "why → what → how" backbone of each section as the required core, and treat the deeper/edge-case subsections (marked by their own "why X exists" headers) as optional depth to bring in only once the core of that file is solid, or when a student's own question naturally leads there
