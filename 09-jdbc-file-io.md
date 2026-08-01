# 09 — File I/O & JDBC (Java's bridge to the outside world)

## Why File I/O exists
A program that only keeps data in memory (RAM) loses everything when it stops running. Real programs need PERSISTENCE — saving data that survives after the program ends. File I/O is the most basic form of persistence: reading and writing to the filesystem directly.
```java
try (BufferedWriter bw = new BufferedWriter(new FileWriter("data.txt"))) {
    bw.write("Hello File");
} catch (IOException e) {
    e.printStackTrace();
}
```
Why `BufferedWriter` wraps `FileWriter`: writing directly to disk on every single small write call is slow (disk I/O is orders of magnitude slower than memory operations). Buffering collects writes in memory and flushes to disk in efficient larger batches — a performance-driven design layered on top of the basic capability.

### Streams vs Readers/Writers — why Java has two parallel I/O hierarchies
Java's I/O classes split into **byte streams** (`InputStream`/`OutputStream` — raw binary data: images, executables, any non-text file) and **character streams** (`Reader`/`Writer` — text, automatically handling character encoding). This split exists because text has an extra concern binary data doesn't: CHARACTER ENCODING (how bytes map to actual human-readable characters — UTF-8, ASCII, etc, tying back to file 02's reasoning for `char` being 2-byte Unicode). Using a byte stream for text risks corrupting non-ASCII characters if the encoding is mishandled manually; Reader/Writer classes handle that translation correctly and consistently, so text-specific code should generally prefer them over raw byte streams.
```java
FileReader fr = new FileReader("data.txt");        // character stream, for text
FileInputStream fis = new FileInputStream("photo.jpg");  // byte stream, for binary data
```

### NIO.2 (`java.nio.file`, Java 7+) — why a newer file API was added
The original `java.io.File` class (used above) had real, long-standing limitations: poor error reporting (many failed operations just silently returned `false` instead of throwing a descriptive exception), no built-in support for symbolic links, and clunky directory-tree operations. NIO.2 introduced `Path` and the `Files` utility class as a more modern, capable replacement:
```java
Path path = Path.of("data.txt");
List<String> lines = Files.readAllLines(path);           // one line — no manual buffering/loop needed
Files.writeString(path, "Hello NIO");
boolean exists = Files.exists(path);
```
For straightforward everyday file reading/writing tasks, `Files` utility methods are generally the simplest modern choice; the older `Reader`/`Writer`/`Stream` classes remain relevant for more fine-grained, custom, or performance-sensitive I/O.

### Object Serialization — why it exists
Sometimes you want to persist an entire Java OBJECT'S state directly (not manually reformat its fields into text yourself). Serialization converts an object implementing `Serializable` (file 04's marker-interface pattern) into a byte stream that can be written to a file (or sent over a network) and later reconstructed ("deserialized") back into an equivalent object.
```java
class User implements Serializable {
    String name;
    int age;
}
try (ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("user.dat"))) {
    out.writeObject(new User());
}
```
Worth knowing as a caveat: raw Java serialization has real historical security weaknesses (a maliciously crafted byte stream can trigger unexpected code execution during deserialization) and poor cross-language compatibility (a serialized Java object can't easily be read by a non-Java system) — this is exactly why JSON (file 10) is the dominant choice for modern APIs and cross-system data exchange, with Java serialization now mostly reserved for narrow, trusted, same-JVM use cases.

---

## Why JDBC exists
Files are fine for simple persistence, but real applications need: structured data, fast searching/filtering across huge datasets, relationships between different kinds of data, and safe concurrent access from multiple users at once. This is exactly what relational DATABASES were built for (decades of dedicated engineering). JDBC (Java Database Connectivity) is the STANDARD bridge letting Java code talk to ANY relational database (MySQL, PostgreSQL, Oracle, etc) through one consistent API — the DATABASE VENDOR provides a "driver" implementing this standard interface, so your Java code doesn't need to change if you switch databases (mirrors Java's broader "write once" philosophy, applied to the database layer).

```java
String url = "jdbc:mysql://localhost:3306/mydb";
Connection conn = DriverManager.getConnection(url, "username", "password");

String sql = "SELECT * FROM users WHERE id = ?";
PreparedStatement ps = conn.prepareStatement(sql);
ps.setInt(1, 5);
ResultSet rs = ps.executeQuery();
while (rs.next()) {
    System.out.println(rs.getString("name"));
}
conn.close();
```

### `Statement` vs `PreparedStatement` vs `CallableStatement` — why three variants exist
- **`Statement`**: executes a raw SQL string as-is. Simplest, but unsafe for any query involving user input (SQL injection risk, below), and re-parses/re-plans the SQL fresh on the database server EVERY execution.
- **`PreparedStatement`**: SQL structure is pre-compiled once by the database, with placeholders (`?`) bound separately as data (below) — both safer AND, for repeated executions of the same query shape, faster (the database can reuse its query execution plan).
- **`CallableStatement`**: specifically for invoking a database's own STORED PROCEDURES (pre-written SQL routines living inside the database itself) — exists because calling a stored procedure needs slightly different syntax (`{call procedureName(?, ?)}`) and support for OUTPUT parameters that plain SQL statements don't have.

### Why PreparedStatement exists, and the SQL Injection problem it prevents
**The vulnerability**: if you build SQL by directly concatenating user input into a query string, a malicious user can inject their OWN SQL logic into your query.
```java
// DANGEROUS — never do this:
String sql = "SELECT * FROM users WHERE username = '" + userInput + "'";
// if userInput = "' OR '1'='1", final query becomes:
// SELECT * FROM users WHERE username = '' OR '1'='1'   -- returns EVERY row, check bypassed entirely
```
This was (and remains) one of the most damaging and common real-world security vulnerabilities in software history.
**PreparedStatement's solution**: the SQL structure is compiled/fixed FIRST, with placeholders (`?`); user input is then bound SEPARATELY as pure DATA, never interpreted as part of the SQL syntax itself — structurally impossible to inject SQL logic through it. This is why PreparedStatement should be the DEFAULT choice, essentially always, over raw Statement.

### Batch updates — why they exist
Inserting/updating many rows one-at-a-time, each as a separate round-trip to the database, wastes time on repeated network/communication overhead per statement. Batching groups multiple statements together and sends them to the database in one round trip:
```java
PreparedStatement ps = conn.prepareStatement("INSERT INTO users (name) VALUES (?)");
for (String name : names) {
    ps.setString(1, name);
    ps.addBatch();          // queue this set of parameters, don't execute yet
}
ps.executeBatch();          // send all queued inserts together, one round trip
```

### Transactions — why "all or nothing" matters
Some operations genuinely need MULTIPLE database changes to succeed or fail TOGETHER, as one indivisible unit — the classic example: transferring money means debiting one account AND crediting another; if the program crashes after the debit but before the credit, money would simply vanish. A transaction groups statements so they either ALL commit (become permanent) or ALL roll back (as if nothing happened) if any part fails.
```java
conn.setAutoCommit(false);         // JDBC defaults to auto-committing every statement individually — turn that off
try {
    // debit statement...
    // credit statement...
    conn.commit();                  // both succeed together
} catch (SQLException e) {
    conn.rollback();                // either failed -> undo both, no partial state left behind
}
```
This "all-or-nothing" guarantee is part of what's formally called **ACID** properties (Atomicity, Consistency, Isolation, Durability) — a foundational reliability guarantee relational databases were specifically engineered to provide, covered in a bit more depth in file 10's backend fundamentals.

### Connection pooling — why raw `DriverManager.getConnection()` doesn't scale
Opening a new physical database connection is a genuinely expensive operation (network handshake, authentication, resource setup on the database server) — doing this fresh for every single request in a busy web application would be slow and would quickly exhaust the database's own limited connection capacity. A **connection pool** (e.g. HikariCP, the default in Spring Boot, file 12) pre-opens and maintains a reusable set of live connections; application code "borrows" one, uses it briefly, and returns it to the pool instead of closing it outright — the same underlying idea as `ExecutorService`'s thread pool (file 08), applied to database connections instead of threads.

This exact JDBC + PreparedStatement pattern, done manually, is precisely what Spring Data JPA (file 13) later automates — understanding the manual version first makes the automated version feel like a natural evolution, not unexplained magic.
