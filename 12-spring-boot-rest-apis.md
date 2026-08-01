# 12 — Spring Boot & Building REST APIs

## Why Spring Boot came to exist (2014)

Classic Spring (file 11), while a huge improvement over EJB, still required substantial manual configuration — deciding and wiring which libraries to use together, writing XML or Java config files for common setups (web server, database connection, JSON conversion), and manually assembling a working project skeleton before writing a single line of actual business logic. This "configuration overhead" became Spring's OWN complexity complaint, ironically similar (in kind, if lesser in degree) to the EJB problem Spring originally solved.

**Spring Boot's solution**: "convention over configuration" and "auto-configuration" — sensible, opinionated DEFAULTS provided automatically based on what dependencies are present on the classpath, an embedded web server (no separate server installation/deployment needed), and "starter" dependency bundles that pull in everything commonly needed for a task in one line. Goal: go from zero to a running application in minutes, not days.

```java
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```
`@SpringBootApplication` is itself a shorthand combining three annotations, each solving a distinct historical need:
- `@Configuration` — marks this class as a source of bean definitions (a Java-based alternative to old XML config)
- `@EnableAutoConfiguration` — the actual "magic": Spring Boot inspects what's on your classpath (e.g. sees a JPA + MySQL driver dependency) and automatically configures matching beans (a DataSource, connection pool, etc) with sensible defaults, unless you override them
- `@ComponentScan` — automatically scans your package for `@Component`/`@Service`/`@Repository`/`@Controller` classes, registering them as beans without manual listing

### Why an embedded server (Tomcat) instead of deploying to a separate one
Traditionally, Java web apps were packaged as a `.war` file and deployed onto a SEPARATELY installed application server (Tomcat, JBoss) already running on the machine — extra setup, extra version-compatibility concerns between your app and the server. Spring Boot instead EMBEDS a servlet container (Tomcat by default) directly inside your application's own executable `.jar` — `java -jar myapp.jar` just runs, with the web server bundled in, needing zero separate server installation. This "just runs" simplicity is a direct continuation of the same philosophy that gave Spring Boot its name and purpose.

## application.properties / application.yml — why external configuration exists
Hard-coding values (database URL, server port) directly in Java code means recompiling just to change an environment setting — bad practice, and impossible when the SAME compiled code needs to run differently in dev/test/production environments. External config files let you change behavior WITHOUT touching code.
```properties
server.port=8080
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=root
spring.datasource.password=secret
```
### Profile-specific configuration files — why `application-dev.properties` exists alongside `application.properties`
Directly implementing file 11's `@Profile` concept at the configuration-file level: `application.properties` holds defaults shared everywhere; `application-dev.properties`, `application-prod.properties` etc override/add values ONLY when that specific profile (`spring.profiles.active=dev`) is active — letting the same compiled application connect to a local test database in development and a real production database in production, without any code change.

## Why "starter" dependencies exist
Before starters, adding "web capability" to a Spring project meant manually finding and adding half a dozen individually-compatible library versions (a real historical pain point called "dependency hell"). A starter (`spring-boot-starter-web`) bundles a CURATED, tested-compatible set of dependencies for one purpose, in one line — directly targeting that pain point.

## Building REST APIs — connecting back to Backend Fundamentals (file 10)

```java
@RestController                     // = @Controller + @ResponseBody, auto-converts return values to JSON
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    @Autowired
    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping
    public List<User> getAllUsers() {
        return userService.getAllUsers();
    }

    @GetMapping("/{id}")
    public User getUserById(@PathVariable Long id) {
        return userService.getUserById(id);
    }

    @PostMapping
    public User createUser(@RequestBody User user) {
        return userService.createUser(user);
    }

    @PutMapping("/{id}")
    public User updateUser(@PathVariable Long id, @RequestBody User user) {
        return userService.updateUser(id, user);
    }

    @DeleteMapping("/{id}")
    public void deleteUser(@PathVariable Long id) {
        userService.deleteUser(id);
    }
}
```
Why `@ResponseBody` (bundled in `@RestController`) exists: before it, Spring's default assumption for a controller method's return value was a VIEW NAME (an HTML page to render, from Spring's older MVC-for-webpages roots). REST APIs need to return raw DATA (JSON), not a rendered page — `@ResponseBody` tells Spring "convert this return value directly to the response body (JSON via the Jackson library), don't try to render it as a view."

- `@PathVariable` — extracts a value embedded IN the URL path itself (`/users/5` → `id=5`) — used for identifying a specific resource
- `@RequestBody` — deserializes the incoming JSON request body into a Java object automatically (Jackson library does this conversion) — used for receiving data to create/update
- `@RequestParam` — extracts query string parameters (`/users?age=25`) — used for optional filters/options

### `ResponseEntity` — why returning a raw object isn't always enough
The examples above return the domain object directly, and Spring maps it to a `200 OK` JSON response automatically — but real APIs often need to control the STATUS CODE explicitly too (`201 Created` after a successful POST, `404 Not Found` when a lookup fails, custom headers). `ResponseEntity<T>` wraps both the body AND full control over status code/headers in one return type:
```java
@PostMapping
public ResponseEntity<User> createUser(@RequestBody User user) {
    User saved = userService.createUser(user);
    return ResponseEntity.status(HttpStatus.CREATED).body(saved);   // explicit 201, not the default 200
}

@GetMapping("/{id}")
public ResponseEntity<User> getUserById(@PathVariable Long id) {
    return userService.findById(id)
            .map(ResponseEntity::ok)                    // 200 if present
            .orElse(ResponseEntity.notFound().build());  // 404 if absent — a real practical use of Optional (file 07)
}
```

### Global exception handling with `@ControllerAdvice` — why centralizing error handling matters
Without it, every single controller method would need its own repetitive try-catch blocks to convert exceptions into proper HTTP error responses — violating DRY (file 02) badly across a whole codebase, and risking inconsistent error response shapes between different endpoints. `@ControllerAdvice` (combined with `@ExceptionHandler`) lets you define exception-to-HTTP-response mapping ONCE, globally, applied automatically across every controller — a direct, practical real-world use of the AOP idea (file 11): a cross-cutting concern factored out of individual methods.
```java
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<String> handleNotFound(UserNotFoundException e) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(e.getMessage());
    }
}
```

### Request validation with `@Valid` — why manual if-checks in every controller don't scale
Real APIs need to reject malformed input (missing required fields, an email in the wrong format, a negative age) before that bad data ever reaches business logic or the database. Writing manual if-checks for every field, in every endpoint, is repetitive and error-prone. Bean Validation (`@Valid` plus annotations like `@NotNull`, `@Email`, `@Size` directly on the DTO's fields, below) lets validation rules live declaratively on the DATA CLASS itself, enforced automatically the moment a `@RequestBody` is bound:
```java
class UserDTO {
    @NotBlank
    String name;
    @Email
    String email;
}
@PostMapping
public User createUser(@Valid @RequestBody UserDTO dto) { ... }  // invalid input -> automatic 400 response, before this method body even runs
```

## Why the DTO pattern exists
Directly exposing your database `@Entity` class as the API's input/output shape couples your PUBLIC API contract to your INTERNAL database structure — any DB schema change breaks the API, and sensitive/internal fields (password hashes, internal flags) risk accidental exposure. A separate DTO (Data Transfer Object) class explicitly defines what the API exposes, decoupled from internal storage.
```java
class UserDTO {
    String name;
    String email;
    // deliberately no password field, no internal DB-only fields
}
```

## Actuator — why production monitoring needs its own dedicated tooling
Once an application is actually running in production, developers/operators need visibility INTO it — is it healthy, what's its memory usage, which beans are loaded — without needing to attach a debugger or add custom monitoring endpoints by hand for every project. `spring-boot-starter-actuator` exposes a standard, ready-made set of operational endpoints (`/actuator/health`, `/actuator/metrics`, etc) purely by adding the dependency — another instance of Spring Boot's "convention over configuration, sensible defaults out of the box" philosophy applied to operations, not just development.

## Swagger / OpenAPI — why API documentation is generated, not hand-written
A REST API's contract (which endpoints exist, what they accept/return) needs to be communicated clearly to consumers (frontend developers, third-party integrators) — hand-maintained documentation drifts out of sync with the actual code almost immediately as the API evolves. Tools like springdoc-openapi inspect your actual `@RestController` annotations at runtime and generate accurate, always-up-to-date interactive API documentation (a Swagger UI page) directly from the real code — solving the "docs go stale" problem structurally, by generating docs FROM the source of truth rather than maintaining a separate description by hand.

## Testing with Postman — why build this habit
A REST API has no visual interface by itself — before any frontend exists, you need SOME way to manually send HTTP requests and inspect responses to verify your API actually works as intended. Postman (or similar tools) fills exactly this gap, letting backend development and testing proceed independently of frontend development.
