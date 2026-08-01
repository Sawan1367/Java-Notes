# 13 — Spring Data JPA (ORM layer)

## Why ORM (Object-Relational Mapping) exists

Java code is naturally OBJECT-oriented (classes, objects, fields). Relational databases are naturally TABLE-oriented (rows, columns). This is called the "object-relational impedance mismatch" — a real, long-standing friction point: writing manual JDBC (file 09) means constantly hand-translating between Java objects and SQL rows/columns, writing large amounts of repetitive boilerplate mapping code for every single entity/table in an application.

**ORM's solution**: a library that automatically maps Java objects to database tables and back, letting you work primarily in Java object terms, generating the SQL underneath automatically for common operations.

## Why JPA and Hibernate are two separate names
- **JPA (Java Persistence API)**: a SPECIFICATION (a set of interfaces/rules) — defines WHAT an ORM for Java should provide, without dictating the actual implementation. Created so applications aren't locked into one specific ORM vendor.
- **Hibernate**: the most popular ACTUAL implementation of the JPA specification (predates JPA itself, historically — JPA later standardized ideas that Hibernate pioneered). When you use `@Entity`, `@Id` etc, you're using JPA's standard annotations; Hibernate (or another JPA-compliant library) does the real work underneath.
This spec-vs-implementation split mirrors JDBC's own driver model (file 09) — a recurring Java philosophy: standardize the CONTRACT, allow swappable IMPLEMENTATIONS.

## Why Spring Data JPA exists on top of plain JPA/Hibernate
Even with JPA/Hibernate, writing a full DAO (Data Access Object) class with basic CRUD methods for EVERY entity was still repetitive boilerplate (save, findById, findAll, delete — nearly identical code, entity after entity). Spring Data JPA's insight: GENERATE this repetitive code automatically from a simple interface declaration.

```java
@Entity                          // marks class as mapped to a database table
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // auto-increment, DB generates the value
    private Long id;

    private String name;
    private String email;
    // getters, setters, constructors
}

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    // save(), findById(), findAll(), deleteById() — provided FREE, zero code written
    List<User> findByName(String name);  // Spring parses the METHOD NAME itself and auto-generates matching SQL
}
```
The `findByName(String name)` "magic" isn't really magic: Spring Data JPA parses method names following a naming CONVENTION (`findBy` + field name) and builds the corresponding query automatically — a direct, practical example of "convention over configuration," the same philosophy behind Spring Boot itself (file 12). This same convention extends further, combining conditions and keywords directly in the method name:
```java
List<User> findByNameAndEmail(String name, String email);
List<User> findByAgeGreaterThan(int age);
List<User> findByNameContainingIgnoreCase(String partialName);
boolean existsByEmail(String email);
long countByActiveTrue();
```

### `@Query` and JPQL — why derived method names aren't always enough
Method-name-derived queries (above) get unwieldy or simply can't express more complex conditions (joins across multiple relationships, aggregate calculations). `@Query` lets you write a query explicitly, using **JPQL** (Java Persistence Query Language) — a query language that looks like SQL but operates on your ENTITY/FIELD names rather than raw table/column names, staying at the object-model level of abstraction consistent with the rest of JPA:
```java
@Query("SELECT u FROM User u WHERE u.email = :email")
User findByEmailCustom(@Param("email") String email);
```
For cases needing genuine native database-specific SQL features JPQL can't express, `@Query(nativeQuery = true)` allows dropping down to raw SQL directly — a deliberate escape hatch, used sparingly, when the higher-level abstraction genuinely can't do what's needed.

## Why entity relationships (`@OneToMany` etc) exist
Real data is relational (file 10) — a User has many Orders. JPA needs an explicit way to express these same relationships that exist at the database level (foreign keys) in OBJECT terms, so navigating `user.getOrders()` in Java automatically triggers the correct underlying SQL join/query.
```java
@Entity
class User {
    @OneToMany(mappedBy = "user")
    List<Order> orders;
}
@Entity
class Order {
    @ManyToOne
    @JoinColumn(name = "user_id")
    User user;
}
```
`mappedBy` — exists to tell JPA which side "owns" the relationship (avoids creating TWO redundant foreign key columns, one per side, for what's really a single relationship).

### Lazy vs Eager fetching — why loading strategy is a deliberate, explicit choice
Loading a `User` doesn't necessarily mean you ALSO want every one of their (potentially thousands of) `Order` records loaded immediately — doing so unconditionally would waste memory/time whenever you just need basic user info. **Lazy** fetching (the default for `@OneToMany`/`@ManyToMany`) defers loading the related data until it's actually ACCESSED (`user.getOrders()` triggers a separate query only at that moment). **Eager** fetching (the default for `@ManyToOne`/`@OneToOne`) loads the related data immediately, in the same query. Choosing wrong in either direction causes real, common problems:
- Lazy fetching accessed AFTER the database session/transaction has already closed throws `LazyInitializationException` — a very common early Spring Data JPA mistake.
- Eager fetching on a relationship you rarely need wastes performance loading data you don't use most of the time.

### The N+1 query problem — why it's a famous, real ORM performance trap
A naive loop like `for (User u : userRepository.findAll()) { u.getOrders().size(); }` looks innocent, but if `orders` is lazily fetched, it silently triggers: 1 query to fetch all users, PLUS one ADDITIONAL query per user to fetch that user's orders — "N+1" queries total instead of one efficient join. This is a classic hidden-performance-cost of ORMs precisely because the object-oriented syntax HIDES how many actual database round-trips are happening underneath. The fix is to explicitly request a JOIN FETCH (via `@Query` with `JOIN FETCH`, or Spring Data's `@EntityGraph`) when you know upfront you'll need the related data, collapsing it back into one efficient query.

## Pagination and sorting — why `findAll()` alone doesn't scale
Returning EVERY row of a large table in one response is both slow and wasteful — most UIs only display one "page" of results at a time anyway. Spring Data JPA's `Pageable` parameter lets a repository method automatically apply `LIMIT`/`OFFSET`-style pagination and sorting, generated from simple request parameters, without writing manual SQL paging logic yourself:
```java
Page<User> findAll(Pageable pageable);
// controller: Pageable pageable = PageRequest.of(0, 20, Sort.by("name"));  // page 0, 20 per page, sorted by name
```

## Cascading — why some operations should automatically ripple to related entities
Deleting a `User` who has `Order`s might need those orders deleted too (an order can't meaningfully exist without its user) — without cascading, you'd need to manually delete every related child entity yourself, in the right order, every time. `cascade = CascadeType.ALL` (or more targeted options like `CascadeType.REMOVE`) tells JPA to automatically propagate certain operations (persist, remove) from a parent entity to its related children — a convenience that also carries real risk if applied carelessly (accidentally cascading a delete much further than intended), so it's chosen deliberately per relationship, not applied blindly everywhere.

## `@Transactional` — why service methods need explicit transaction boundaries
Multiple repository calls within one business operation (debit one account, credit another — file 09's transaction example, now at the JPA level) still need to succeed or fail TOGETHER as one unit. `@Transactional` on a service method is Spring's AOP (file 11) automatically wrapping that method in a database transaction — commit if it completes normally, rollback automatically if any (unchecked, by default) exception is thrown partway through — removing the need to manually manage `commit()`/`rollback()` calls as file 09's raw JDBC example did by hand.
```java
@Transactional
public void transferFunds(Long fromId, Long toId, double amount) {
    // debit fromId, credit toId — both succeed or both roll back automatically
}
```

## Entity lifecycle states — why JPA tracks more than just "saved or not"
JPA internally tracks each entity object's relationship to the persistence context (roughly, the current "unit of work"):
- **Transient**: a plain new object (`new User()`), not yet associated with JPA at all.
- **Managed/Persistent**: JPA is actively tracking this object; any field changes are automatically detected and will be written to the database when the transaction commits ("dirty checking") — notably, you often don't even need to call an explicit `save()` again for an already-managed entity's field changes to persist.
- **Detached**: was managed, but the persistence context/transaction has since closed — changes no longer auto-tracked, would need `merge()` to reattach and persist further changes.
- **Removed**: marked for deletion, will be deleted from the database at commit.

Knowing this explains a genuinely common beginner confusion: why modifying a fetched entity's field sometimes seems to "just save itself" without an explicit call — it's the managed state's automatic dirty-checking behavior, not accidental magic.

## Practical connection back to JDBC (file 09)
Everything Spring Data JPA does automatically is fundamentally the SAME operations you'd write manually with `PreparedStatement` and `ResultSet` — JPA just generates that code for you based on your entity/repository definitions. Understanding manual JDBC first is exactly why this layer doesn't feel like unexplained magic — it's automation of a process already understood.
