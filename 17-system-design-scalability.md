# 17 — Backend System Design & Scalability (why large systems are built the way they are)

File 10 covered a single server talking to a single database. Real systems serving many users need to survive traffic spikes, hardware failure, and geographic distance — this file covers the patterns that emerge once "one server, one database" stops being enough.

## Vertical vs Horizontal scaling — why horizontal eventually wins
**Vertical scaling** (scale UP): give the single server more CPU/RAM/disk. Simple, no architecture change needed — but has a hard ceiling (a single machine's max hardware), a single point of failure (that one box dies, everything's down), and cost grows non-linearly (a machine twice as powerful often costs far more than twice as much).
**Horizontal scaling** (scale OUT): add MORE servers, spread the load across them. No hard ceiling (keep adding machines), naturally more fault-tolerant (one dies, others keep serving) — but requires the application to be designed for it: this is exactly WHY REST's statelessness (file 10) matters so much in practice — a stateless server means ANY instance can handle ANY request, which is a prerequisite for horizontal scaling to work at all. Most real large-scale systems (and everything below in this file) exist specifically to make horizontal scaling practical.

---

## Load Balancing — why you can't just point users at "the server" anymore
Once there are multiple server instances, something must decide WHICH instance handles each incoming request — a **load balancer** sits in front of the server pool and distributes traffic, so from the outside the whole pool looks like one address.

### Load balancing algorithms — why several exist, not just one
| Algorithm | How it decides | Why used |
|---|---|---|
| Round Robin | cycles through servers in order | simplest, works well when all servers are roughly equal capacity and requests are roughly equal cost |
| Least Connections | sends to whichever server currently has fewest active connections | better when request durations vary a lot — avoids piling more work onto an already-busy server |
| IP Hash | same client IP always routed to same server | needed when a server holds per-client state in memory (session affinity/"sticky sessions") — a workaround for NOT being fully stateless |
| Weighted variants | some servers get proportionally more traffic | useful when the pool has mixed hardware (a newer, more powerful box can fairly take more load) |

### Layer 4 vs Layer 7 load balancing — why the distinction matters
- **L4 (Transport layer)**: routes based on IP/port only, doesn't look at the actual HTTP request content — fast, low overhead, "dumb" in a good way.
- **L7 (Application layer)**: understands HTTP itself — can route based on URL path, headers, cookies (e.g. `/api/*` to one pool, `/static/*` to another) — more flexible, slightly more overhead since it must parse the request.
Real systems (e.g. NGINX, AWS ALB) commonly use L7 for this exact routing flexibility, reserving L4 for cases where raw throughput matters more than routing intelligence.

### Health checks — why load balancers must actively monitor servers
Blindly sending traffic to a server assumes it's alive — but servers crash, hang, or run out of resources. Load balancers periodically ping each server (a lightweight `/health` endpoint) and automatically stop routing to any that fail to respond — this is precisely why Spring Boot Actuator's `/actuator/health` endpoint (file 12) exists: it's built specifically to be that health-check target in real deployments, not just a debugging convenience.

---

## Caching — why re-computing/re-fetching the same thing repeatedly is wasteful
**Problem**: many requests ask for the SAME data repeatedly (a popular product page, a user's profile), and re-running the same expensive DB query or computation every single time wastes resources and adds latency the data didn't need to re-earn. **Caching** stores a copy of the result somewhere FASTER to access than the original source, serving repeat requests from that copy instead.

### Where caching happens — multiple layers, each solving a different distance problem
```
Client (browser cache)
  ↓
CDN (geographically distributed cache, below)
  ↓
API Gateway / reverse-proxy cache
  ↓
Application-level cache (in-memory, or Redis/Memcached)
  ↓
Database (query cache / buffer pool)
```
Each layer exists because it's progressively CLOSER to the requester — a browser cache hit costs zero network time; a database query cache hit still costs a network round trip but skips disk I/O. Real systems layer several of these deliberately rather than picking just one.

### Cache strategies — why different write patterns exist
| Strategy | How it works | Trade-off |
|---|---|---|
| **Cache-aside (lazy loading)** | app checks cache first; on miss, reads DB, then WRITES result into cache itself | simple, only caches what's actually requested; risk of stale data if DB changes without updating cache |
| **Write-through** | every write goes to cache AND database together, synchronously | cache always fresh; write is slightly slower (two writes) |
| **Write-behind (write-back)** | write goes to cache immediately, database updated ASYNCHRONOUSLY later | fastest writes; risk of data loss if cache fails before the async DB write completes |
| **Read-through** | cache itself is responsible for loading from DB on a miss (app just always talks to cache) | simplifies app code, logic centralized in the caching layer |

### Cache eviction — why a cache can't just grow forever
Memory is finite — a cache must eventually decide what to REMOVE to make room for new entries.
- **LRU (Least Recently Used)** — evict whatever hasn't been accessed in the longest time; assumes "recently used = likely used again soon," true for most real access patterns.
- **LFU (Least Frequently Used)** — evict whatever's been accessed the fewest total times; better when some items are consistently popular and others rarely touched, regardless of recency.
- **TTL (Time To Live)** — entries expire automatically after a fixed duration, regardless of usage — necessary whenever staleness itself (not just memory pressure) is the real risk, e.g. caching a stock price.

### Redis / Memcached — why a dedicated caching layer instead of just in-process memory
An in-process cache (a plain `HashMap` inside your app) is fast but has two real problems at scale: it's NOT shared across multiple server instances (defeats horizontal scaling — each instance would cache separately, wasting memory and giving inconsistent answers), and it vanishes on restart. **Redis** (and similar tools like Memcached) run as a separate, shared, in-memory data store that every app instance talks to over the network — slightly slower than truly in-process memory, but shared, and (Redis specifically) can optionally persist to disk and offers richer data structures (lists, sets, sorted sets) beyond simple key-value.

### Cache invalidation — the famously "hard problem"
The classic joke ("there are only two hard things in computer science: cache invalidation and naming things") exists because staleness is genuinely tricky: invalidate too aggressively and you lose most of caching's benefit; invalidate too lazily and users see outdated data. Common real approaches: TTL expiry (simplest, accept some staleness), explicit invalidation on write (delete/update the cache entry the moment the underlying data changes), and versioned/keyed cache entries (e.g. include a data version in the cache key, so old versions simply age out naturally).

---

## API Gateway — why a single entry point in front of many services
Once a backend is split into multiple services (see Microservices below), clients calling each service DIRECTLY creates real problems: clients must know every service's address, each service must independently implement auth/rate-limiting/logging, and there's no single place to apply cross-cutting policy changes.
**API Gateway**: a single entry point ALL client traffic passes through first, which then routes each request to the appropriate backend service.
- Centralizes **authentication/authorization** (verify identity ONCE at the gateway, not separately in every service — file 14's JWT is commonly validated right here).
- Centralizes **rate limiting** (below) and **logging/monitoring** in one place instead of duplicated per-service.
- Can do **request routing** (`/orders/*` → Order Service, `/users/*` → User Service) and even **protocol translation** (public REST in, internal gRPC out, for example).
- Shields internal architecture from clients — services can be split, merged, or moved without clients needing to know, since they only ever talk to the gateway's stable public contract.

### Rate limiting — why unrestricted traffic is dangerous even from legitimate clients
A single buggy client (or a genuine traffic spike, or a malicious one) sending requests without limit can overwhelm a backend regardless of intent. Rate limiting caps how many requests a client can make in a given window.
| Algorithm | How it works |
|---|---|
| **Token bucket** | bucket holds tokens, refilled at a steady rate; each request consumes one token; empty bucket = request rejected/delayed. Allows brief bursts up to the bucket's capacity, which is often the realistic traffic pattern worth tolerating. |
| **Leaky bucket** | requests queue up and are processed at a strictly constant rate, like water leaking from a hole — smooths bursts out completely rather than tolerating them. |
| **Fixed window** | simple counter reset every N seconds | simplest, but can allow a burst right at a window boundary (2x the intended rate for a moment) |
| **Sliding window** | counts requests in a continuously moving time window | fixes the fixed-window boundary problem, at the cost of a bit more bookkeeping |

---

## Message Queues & async processing — why not everything can be synchronous request/response
**Problem**: some work is genuinely slow (sending an email, processing a video, generating a report) — making the CLIENT wait synchronously for it wastes a connection and gives a bad user experience; and some work needs to survive the CALLER crashing before it's picked up.
**Solution**: a **message queue** (Kafka, RabbitMQ, SQS) sits between a producer (submits work) and one or more consumers (process it) — the producer publishes a message and moves on immediately; consumers process messages independently, at their own pace, and the queue durably holds messages until they're actually processed, surviving a consumer crash mid-way.

### Why this decouples systems, not just "delays" work
Beyond just async speed, a queue breaks a DIRECT dependency between services — the order service doesn't need to know or care HOW MANY email-sending consumers exist, or whether one is temporarily down; it just publishes "order placed" and trusts the queue. This is a foundational building block for microservices communicating without every service needing to know every other service's address and availability directly.

### Point-to-point vs Publish/Subscribe — two different delivery models
- **Point-to-point (queue)**: each message consumed by exactly ONE consumer — used for distributing WORK across a pool of workers (e.g. multiple order-processing workers sharing one queue, each order handled once).
- **Publish/Subscribe (pub/sub, topics)**: each message delivered to EVERY subscriber — used when multiple independent systems all need to REACT to the same event (e.g. "order placed" triggers email service, analytics service, and inventory service, independently, all from one published event).

### Kafka vs RabbitMQ — why both exist rather than one universal choice
- **RabbitMQ**: traditional message BROKER — good at complex routing rules, message acknowledgment/retry semantics, moderate throughput. Fits classic task-queue use cases.
- **Kafka**: a distributed, append-only, persistent LOG — built for very high throughput and for consumers to be able to replay/re-read history (each consumer independently tracks its own read position/offset) — fits event-streaming and analytics-heavy use cases where message history itself has lasting value, not just momentary delivery.

---

## CDN (Content Delivery Network) — why static content shouldn't travel from one origin server every time
A user in Tokyo requesting a static image/CSS/JS file from a server physically located in Virginia pays real network latency for that distance, every single request. A **CDN** is a network of servers distributed GEOGRAPHICALLY, each caching a copy of static content close to end users — the request gets served from whichever CDN node is physically nearest, cutting latency dramatically, and simultaneously taking load off the origin server (which now only needs to serve each piece of content once per region, not once per user).

---

## Database scaling — extending file 10's single-database model

### Replication — why a database needs backup copies beyond just "for safety"
A **primary/replica** setup keeps one or more READ-ONLY copies of the database in sync with the primary. Two real benefits: fault tolerance (a replica can be promoted if the primary fails) and READ scaling (route read-heavy traffic — often the vast majority of real traffic — to replicas, keeping the primary free for writes). Real trade-off: replication is rarely perfectly instantaneous, so replicas can serve slightly STALE data (**replication lag**) — acceptable for many reads (a product listing), not acceptable for others (checking your own just-placed order immediately after placing it), which is exactly why some systems route certain specific reads back to the primary deliberately.

### Sharding (horizontal partitioning) — why one database eventually isn't enough even with replication
Replication solves READ scaling and fault tolerance, but every replica still holds the FULL dataset and the PRIMARY still takes ALL writes — eventually write volume or total data size outgrows what one machine can hold. **Sharding** splits the data itself across multiple independent database instances (shards), each holding only a SUBSET of the data (e.g. users A-M on shard 1, N-Z on shard 2; or split by user ID hash) — this is what actually scales WRITE throughput and total storage beyond a single machine's limits, at the cost of real complexity: queries that need data across shards (e.g. "count all users") become much harder, and choosing a good shard key (to avoid one shard becoming a hot spot) is a genuinely hard design decision.

### Consistent hashing — why naive sharding breaks when you add/remove a shard
A naive shard assignment (`hash(key) % numberOfShards`) means adding or removing even ONE shard changes the modulo result for almost EVERY key — forcing a massive, disruptive data reshuffle. **Consistent hashing** maps both shards and keys onto a conceptual ring; each key belongs to the next shard clockwise on the ring — adding/removing a shard then only affects the small portion of keys immediately adjacent to that change on the ring, not virtually all of them. This same technique underlies how distributed caches (Redis Cluster) and load balancers distribute load with minimal disruption on scaling events.

### CAP theorem — why you can't have everything in a distributed database
For any distributed data system, during a network partition (some nodes can't talk to others — an inevitability at scale, not a rare edge case), you must choose between:
- **Consistency** — every read gets the most recent write, or an error (never stale data).
- **Availability** — every request gets SOME response, even if it might be stale.
(**Partition tolerance** is essentially mandatory for any real distributed system — networks WILL partition eventually — so in practice CAP is really a CP vs AP choice under partition.)
This is exactly why different databases make different deliberate trade-offs: traditional relational databases (file 10) lean CP (favor correctness, will refuse/delay a request rather than serve stale data); many NoSQL systems (Cassandra, DynamoDB) lean AP (favor always responding, accept **eventual consistency** — replicas converge to the same value eventually, just not instantly).

---

## Microservices — why (and when) to split one backend into many
File 10's layered architecture is still ONE deployable application. **Microservices** splits a system into multiple INDEPENDENTLY deployable services, each owning its own data and a specific business capability (Order Service, User Service, Payment Service), communicating over the network (often via the message queues and API gateway above).

**Why**: a large single ("monolithic") codebase eventually becomes hard for many teams to work on simultaneously without stepping on each other, hard to deploy incrementally (any change requires redeploying the WHOLE app), and hard to scale selectively (if only the "search" feature is under heavy load, a monolith forces you to scale the entire app, not just that piece).
**Real cost, not a free upgrade**: distributed systems are genuinely harder to reason about (network calls can fail in ways in-process calls can't), require solving problems a monolith gets for free (service discovery, distributed transactions, below), and add real operational overhead (many more things to deploy/monitor). This is why most systems reasonably start as a monolith (file 10-13's approach) and split out microservices later, deliberately, once a specific real scaling or team-organization need justifies the added complexity — not as a default starting architecture.

### Service discovery — why services can't just hardcode each other's addresses
In a cloud/container environment, service instances are created and destroyed dynamically (scaling up/down, redeploys) — a hardcoded IP address would constantly go stale. A **service registry** (e.g. Eureka, Consul) lets each service instance register itself on startup and de-register on shutdown; other services look UP the current address(es) by service NAME rather than a fixed address, resolving this exact problem.

### Circuit breaker pattern — why one failing service shouldn't cascade into everything
If Service A calls Service B and B is down/slow, A's requests to B will pile up waiting/timing out — if this happens enough, A itself can become overwhelmed and start failing too, and this can CASCADE across a whole system. A **circuit breaker** wraps calls to a downstream service and tracks failure rate — after too many failures, it "trips open" and starts failing FAST (without even attempting the call) for a cooldown period, giving the struggling downstream service room to recover instead of being hit with a continuous flood of retries, and protecting the CALLING service from piling up its own resources waiting on a service that's clearly not responding.

### Saga pattern — why distributed transactions can't just use ACID (file 10) across services
File 10's ACID transactions work within ONE database. When a single business operation spans multiple services with SEPARATE databases (e.g. "place order" touches Order, Payment, and Inventory services), a traditional all-or-nothing transaction across all of them isn't practically available. A **saga** instead breaks the operation into a sequence of local transactions, each with a defined COMPENSATING action to undo it if a later step fails (e.g. if Payment fails after Inventory already reserved stock, a compensating "release reserved stock" step runs) — trading true atomicity for an eventually-consistent, explicitly-recoverable sequence, which is the realistic trade-off distributed systems accept in exchange for independent per-service databases.

---

## Putting it together — a realistic request's journey
```
Client
  → CDN (static assets served here directly, never reach origin)
  → Load Balancer (picks a healthy instance)
  → API Gateway (auth check, rate limit check, routes by path)
  → Application server (checks cache first)
       cache hit  → return cached response
       cache miss → query database (possibly a read replica)
                  → write result to cache
                  → maybe publish an event to a message queue for async side-effects
  → Response back to client
```
Every layer in this chain exists to answer one of the same recurring questions this whole file covers: what happens when there's too much traffic for one machine, what happens when a machine fails, and what happens when work is too slow to make a client wait for it synchronously.
