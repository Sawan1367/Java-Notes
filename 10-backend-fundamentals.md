# 10 — Backend Fundamentals (why the web works the way it does)

## Client-Server model — why it exists
Early computing had single standalone programs, no networked interaction. As networks (then the internet) grew, a NATURAL split emerged: one machine (server) holds data/logic and waits to respond to requests; other machines (clients) ask for things. Centralizing logic/data on a server means updates happen in ONE place, not on every individual user's device.
Analogy: restaurant — client = customer ordering, server = kitchen preparing, request = order slip, response = food delivered.

## Why HTTP exists
Early internet needed a STANDARD, universal language for clients and servers to exchange documents/data, regardless of what OS or software either side ran — otherwise every pair of systems would need custom, incompatible communication logic. HTTP (HyperText Transfer Protocol, created early 1990s alongside the web itself) became that universal standard: a simple, text-based request/response protocol.

**HTTP Methods** — why several exist, not just one generic "do something":
Mapping cleanly onto CRUD (Create Read Update Delete) operations makes API intent self-documenting and lets infrastructure (caches, browsers, proxies) apply sensible universal rules (e.g. GET requests are assumed safe/repeatable/cacheable, so browsers cache them; POST is assumed to have side effects, so browsers warn on accidental resubmission).
| Method | Purpose | CRUD |
|---|---|---|
| GET | fetch data | Read |
| POST | create new data | Create |
| PUT | replace entire resource | Update |
| PATCH | partial update | Update |
| DELETE | remove data | Delete |

### Idempotency — why it's a distinct concept from "safe"
An operation is **idempotent** if performing it multiple times has the SAME effect as performing it once — this matters enormously for real-world reliability, because networks are unreliable: a client might not receive a response and retry a request that actually already succeeded server-side. `GET`, `PUT`, and `DELETE` are (by convention/design) idempotent — retrying a `PUT` that sets a resource to a specific state, or a `DELETE` on something already deleted, causes no additional harm. `POST` is generally NOT idempotent — retrying a "create new order" POST could create a SECOND, duplicate order. This is exactly why real payment/checkout systems need extra explicit safeguards (idempotency keys) around POST-style operations, a direct practical consequence of this HTTP-method design distinction.

**HTTP Status codes** — why grouped in ranges: gives any client a QUICK way to categorize a response's general nature (success vs client mistake vs server mistake) even without reading the specific code.
- 2xx success (200 OK, 201 Created, 204 No Content)
- 3xx redirection (301 Moved Permanently, 304 Not Modified)
- 4xx client's fault (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict)
- 5xx server's fault (500 Internal Server Error, 503 Service Unavailable)

### HTTP headers — why metadata travels separately from the actual request/response body
A request/response needs to communicate things ABOUT the data (its format, its size, caching rules, authentication credentials) without mixing that metadata into the actual content itself. Headers are key-value pairs sent alongside the body specifically for this purpose:
```
Content-Type: application/json        — tells the receiver how to interpret/parse the body
Authorization: Bearer eyJhbGc...       — carries credentials/identity (file 14's JWT)
Content-Length: 348                    — lets the receiver know exactly how much data to expect
Cache-Control: no-cache                — instructs caching behavior
```

### Cookies and sessions — why they exist, and the statelessness tension they create
HTTP itself is stateless (below) — by default, a server has no memory of a previous request from the same client. But real applications often need SOME continuity (e.g. "stay logged in" across multiple page visits). A **cookie** is a small piece of data the server asks the client's browser to store and automatically resend on every subsequent request to that same domain — commonly used to carry a session identifier, letting the server look up server-side "session" state associated with that ID. This traditional session-cookie approach is precisely the "server must remember every client" model that JWT-based authentication (file 14) was designed to avoid, for better horizontal scalability.

### CORS (Cross-Origin Resource Sharing) — why browsers need an explicit exception mechanism
By default, browsers enforce the "Same-Origin Policy": a webpage loaded from one origin (domain+port+protocol) cannot make requests to a DIFFERENT origin via JavaScript — a security measure preventing a malicious site from silently making authenticated requests to, say, your bank's site using your logged-in browser session. This becomes a real practical problem the moment a frontend (e.g. `myapp.com`) and backend API (e.g. `api.myapp.com`) live on different origins, which is extremely common. CORS is the standard mechanism letting a SERVER explicitly declare (via response headers like `Access-Control-Allow-Origin`) which other origins are permitted to call it — Spring Boot (file 12) provides `@CrossOrigin` and global CORS configuration specifically to manage this declaration cleanly.

## Why JSON became the standard data format
Before JSON, XML was the dominant data-exchange format — verbose, heavier to parse, more ceremony (opening/closing tags for everything). JSON emerged from JavaScript's own native object syntax — lightweight, human-readable, and (crucially) trivially parseable in almost any language, not just JavaScript. It won out over XML for APIs mainly on simplicity and smaller payload size.
```json
{
  "id": 1,
  "name": "John",
  "isActive": true,
  "courses": ["Java", "Spring Boot"]
}
```

## Why REST principles exist
Early web APIs were inconsistent — every team invented its own conventions (`/getUser?id=5`, `/fetchUserData`, etc), making APIs hard to learn/predict. Roy Fielding's REST (REpresentational State Transfer, 2000 doctoral dissertation) proposed a consistent, resource-oriented style:
- **Resource-based URLs**: nouns, not verbs (`/users/5`, not `/getUser?id=5`) — the HTTP METHOD already conveys the verb/action
- **Statelessness**: server does NOT remember anything about the client between requests — every request must carry all info needed to process it. Why: makes servers trivially scalable (any server instance can handle any request, no "sticky" per-client memory needed) — critical for the internet's scale.
- Proper use of HTTP methods and status codes (above)

```
GET    /users       — list all
GET    /users/5     — get one
POST   /users       — create new
PUT    /users/5     — update
DELETE /users/5     — delete
```

### API versioning — why real APIs eventually need it
Once an API has real external consumers (mobile apps, third-party integrations), you can't simply CHANGE its shape (renaming/removing a field, changing behavior) without breaking every existing client that relies on the old shape — unlike internal code, you often can't force every consumer to update simultaneously. Versioning (`/api/v1/users` vs `/api/v2/users`, or a version header) lets a breaking change be introduced as a NEW version while the old version keeps working unchanged for clients not yet migrated — a practical, deliberate trade-off between evolving an API and not breaking its existing users.

### HATEOAS — why it's part of REST's original vision (even though often skipped in practice)
Fielding's original REST concept included **HATEOAS** (Hypermedia As The Engine Of Application State) — the idea that a response shouldn't just return raw data, but also include LINKS describing what related actions/resources are available next (much like clicking through a website via links, rather than needing to already know every URL upfront). Most real-world "REST" APIs today are actually simpler "RESTful" APIs that skip this — worth knowing the term exists and what "true" REST originally envisioned, versus the more pragmatic subset most APIs (including what file 12 teaches) actually implement.

## Why relational databases and SQL exist
Businesses needed to store structured, related data (customers, orders, products) with strong guarantees: no duplicate/conflicting data, safe concurrent access from many users, ability to query complex relationships efficiently. Relational model (tables, rows, columns, keys — formalized by Edgar Codd, 1970) plus SQL (Structured Query Language) as its standard query language became the dominant solution because of these strong consistency guarantees, still the backbone of most backend systems today.

- **Primary Key** — uniquely identifies each row, exists so other tables/rows can reliably REFER to a specific record
- **Foreign Key** — a column in one table referencing another table's primary key, exists to model REAL relationships between different kinds of data without duplicating that data everywhere
```sql
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100),
    email VARCHAR(100)
);
SELECT * FROM users WHERE id = 5;
INSERT INTO users (name, email) VALUES ('John', 'john@mail.com');
UPDATE users SET name = 'Jane' WHERE id = 5;
DELETE FROM users WHERE id = 5;
```
Relationship types exist to model real-world associations accurately: One-to-One, One-to-Many (one user, many orders), Many-to-Many (students and courses — each has many of the other, needs a junction table).

### Normalization — why data is deliberately split across multiple tables
Storing a customer's full name/address repeated on EVERY one of their order rows wastes space and creates a real update-consistency risk (fix a typo'd address on one order row, forget the other nine — now the data disagrees with itself). **Normalization** is the discipline of structuring tables so each real-world fact is stored in exactly ONE place, and related facts are linked via foreign keys instead of duplicated — trading a bit of query complexity (needing `JOIN`s to reassemble related data) for much stronger data consistency guarantees.

### Indexes — why some queries are fast and others are slow
Without an index, finding matching rows means the database scanning EVERY row in a table ("full table scan") — fine for a few hundred rows, ruinously slow for millions. An **index** is an auxiliary, sorted data structure (commonly a B-tree) built on one or more columns, letting the database jump almost directly to matching rows instead of scanning everything — conceptually similar to how a book's index lets you jump to a topic instead of reading every page. The trade-off: indexes speed up READS but add overhead to WRITES (every insert/update must also update the index), and consume extra storage — so indexes are added deliberately on columns actually queried/filtered frequently, not blindly on everything.

### ACID transactions — why relational databases guarantee this
Briefly introduced in file 09; formally, relational databases promise four properties for any transaction: **Atomicity** (all-or-nothing, no partial results), **Consistency** (a transaction only moves the database from one valid state to another, never violating defined rules like foreign key constraints), **Isolation** (concurrent transactions don't see each other's uncommitted, in-progress changes), **Durability** (once committed, data survives even a crash immediately after). These exist because real applications (banking, inventory, bookings) cannot tolerate the alternative — silent partial updates, dirty reads of half-finished work, or "successfully confirmed" data vanishing on a crash — so the database engine itself is engineered to guarantee them, rather than leaving every application to reinvent this correctness by hand.

## Why layered/N-tier architecture exists
As backend applications grew complex, putting ALL logic (HTTP handling, business rules, database queries) into one giant undifferentiated block of code became unmaintainable — any small change risked breaking unrelated things, and testing was difficult (couldn't test business logic without a real HTTP request or real database). Layering enforces SEPARATION OF CONCERNS: each layer has ONE job, layers only talk to their immediate neighbor, each layer independently testable/replaceable.
```
Client → Controller (HTTP concerns only)
       → Service (business logic/rules)
       → Repository/DAO (database access only)
       → Database
```
This exact pattern is what Spring Boot's `@RestController` / `@Service` / `@Repository` annotations directly encode (file 12-13) — understanding WHY the separation exists makes those annotations meaningful, not just required boilerplate.

## Authentication vs Authorization — why both exist as separate concepts
Two genuinely different questions arise for any protected system: "who are you" (identity) and "what are you allowed to do" (permissions) — a system could correctly verify WHO you are, yet still need a SEPARATE rule set determining what THAT identity is permitted to access. Conflating the two leads to security design mistakes.
Analogy: authentication = showing ID at the entrance. Authorization = your ID/badge determines which FLOORS you can access once inside.
