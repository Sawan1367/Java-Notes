# 14 — Spring Security & Testing

## Why Spring Security exists
Any backend exposing data over the internet needs to answer two recurring questions (file 10): WHO is making this request (authentication), and WHAT are they allowed to do (authorization). Implementing this correctly by hand (password hashing, session management, protecting every endpoint consistently) is easy to get WRONG in security-critical, exploitable ways. Spring Security exists as a battle-tested, standard layer that intercepts every request and enforces these rules consistently, so individual developers aren't reinventing (and potentially breaking) security logic per project.

```java
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(Long id) { ... }
```

### The filter chain — why security is implemented as a series of interceptors, not inline code
Spring Security works by inserting a CHAIN of servlet filters in front of your actual controllers — each filter handles one specific concern (parsing/validating a JWT, checking CSRF tokens, enforcing HTTPS) and either lets the request continue to the next filter or rejects it early. This mirrors the same "single responsibility, cross-cutting concern" reasoning behind AOP (file 11): security checks apply UNIFORMLY across every endpoint automatically, without each controller method needing to remember to call security logic itself — you configure the chain once, centrally, and it applies everywhere.

### Password encoding — why plaintext or naive hashing isn't enough
Storing passwords in plaintext means a single database breach instantly exposes every user's real password (and, since people reuse passwords, likely their accounts on OTHER unrelated sites too) — an enormous, historically very real class of damage. Even naive hashing (a single, fast hash like plain SHA-256) is vulnerable to modern brute-force/rainbow-table attacks, since fast hashes can be tried billions of times per second on cheap hardware. Spring Security's `BCryptPasswordEncoder` uses a hashing algorithm DELIBERATELY designed to be slow and to include a random per-password "salt" — slow enough to make brute-forcing impractical at scale, and salted so identical passwords don't produce identical stored hashes (defeating precomputed rainbow-table attacks). This is why you should never write your own password hashing scheme — using a well-vetted, purpose-built encoder is a security fundamental, not a preference.
```java
PasswordEncoder encoder = new BCryptPasswordEncoder();
String hashed = encoder.encode(rawPassword);          // store this, never the raw password
encoder.matches(rawPassword, hashed);                   // verify at login, never by decrypting/comparing raw values
```

### CSRF — why it's a separate concern from authentication itself
Cross-Site Request Forgery is an attack where a malicious site tricks a victim's BROWSER into submitting a request (using the victim's already-authenticated session cookie) to a different site the victim is logged into — the attacker never sees the victim's credentials, but can still trigger unwanted actions AS the victim. This is specifically a risk for traditional cookie/session-based authentication (a cookie is sent automatically by the browser on every request to that domain, including ones triggered by another site); Spring Security's CSRF protection requires an additional, unguessable token that a malicious third-party site can't supply. Stateless, token-based auth (JWT, below) sidesteps this particular vulnerability differently — since the token must be explicitly attached by your own frontend code to an `Authorization` header, a foreign malicious page can't make the browser attach it automatically the way a cookie would be.

## Why JWT (JSON Web Token) exists

**Problem with traditional session-based auth**: server creates a "session" after login, stores session data SERVER-SIDE, gives client a session ID (cookie) to reference it on future requests. This means the server must REMEMBER every logged-in client — directly conflicts with REST's statelessness principle (file 10), and makes scaling to multiple server instances harder (session data must be shared/synced across all servers, or all requests from one client must always hit the SAME server).

**JWT's solution**: instead of the server storing session state, encode the user's identity/claims directly INTO a signed token, given to the client after login. Client sends this token with every subsequent request (typically in an `Authorization` header). Server VERIFIES the token's signature (proving it wasn't tampered with) without needing to look anything up in server-side storage — the token itself carries everything needed. This achieves genuinely stateless authentication, matching REST's core statelessness principle directly.

### JWT's actual structure — why it has three distinct parts
A JWT is three Base64-encoded segments separated by dots: `header.payload.signature`.
- **Header**: metadata about the token itself (which signing algorithm was used).
- **Payload**: the actual "claims" — user ID, roles, expiry time, any other data the server wants to embed. Important: this part is only ENCODED, not encrypted — anyone can decode and read it (never put secret data like a raw password in here).
- **Signature**: computed from the header+payload using a secret key only the server knows — this is what makes tampering detectable: change even one character of the payload, and the signature no longer matches, so the server immediately rejects the token.

Trade-off worth mentioning: since the server doesn't track tokens, immediately REVOKING a single token before its natural expiry (e.g. on logout, or a compromised account) is harder than with server-side sessions — real systems handle this with short token lifespans plus refresh-token strategies, a deeper topic for later.

### Role-based vs permission-based authorization — why both models exist
`@PreAuthorize("hasRole('ADMIN')")` checks a coarse-grained ROLE — simple, works well when access rules naturally group by job function (admin vs regular user). Larger/more nuanced systems sometimes need finer-grained PERMISSIONS (`CAN_DELETE_USERS`, `CAN_VIEW_REPORTS`) that don't map neatly onto a small set of roles — e.g. two different roles might both need one specific shared permission, without being otherwise identical. Spring Security supports both models; the choice is a genuine design decision based on how naturally an application's real access rules group into roles versus needing more granular, mix-and-match permissions.

## Why testing exists as a discipline, and why Spring makes it easy

**Problem it targets**: manually re-testing an application by hand (clicking through, sending Postman requests) after every code change doesn't scale — slow, easy to forget edge cases, no record that a past bug won't silently reappear (regression). Automated tests codify expected behavior as CODE itself, run in seconds, repeatable forever.

```java
@SpringBootTest
class UserServiceTest {

    @Autowired
    UserService userService;

    @Test
    void testGetAllUsers() {
        List<User> users = userService.getAllUsers();
        assertNotNull(users);
    }
}
```

## Why unit tests and integration tests are BOTH needed, not just one
- **Unit test**: tests ONE piece (e.g. one Service method) in complete ISOLATION, faking/mocking its dependencies (using Mockito) — fast, pinpoints exactly which piece broke, but doesn't prove the pieces work correctly TOGETHER.
- **Integration test**: tests MULTIPLE real pieces together (real or realistic in-memory database, real Spring context) — slower, but catches problems that only appear from actual interaction between components (e.g. a wrong SQL query generated by a repository method, invisible to a mocked unit test).
Real projects need both: unit tests for fast, precise feedback on logic correctness; integration tests for confidence the whole system actually works end-to-end.

### Test "slices" — why `@SpringBootTest` isn't always the right annotation
Loading the ENTIRE application context (every bean, full auto-configuration) for every single test — as plain `@SpringBootTest` does — is slow when you only actually need to test ONE layer. Spring Boot provides narrower "slice" test annotations that load only the relevant portion of the context, directly mirroring the layered architecture from file 10:
- `@WebMvcTest` — loads only the web/controller layer (with a mocked service layer beneath it) — fast tests focused purely on HTTP request/response handling, status codes, routing, validation.
- `@DataJpaTest` — loads only the JPA/repository layer against a real (often in-memory) test database — fast tests focused purely on query correctness.
- `@SpringBootTest` — loads the FULL context — reserved for genuine end-to-end integration tests where you need everything wired together realistically.

Choosing the narrowest slice that still exercises what you're actually testing keeps a test suite fast enough to run constantly during development, rather than becoming slow enough that developers start skipping it — a very real, practical concern in growing codebases.

### MockMvc — why controller tests don't need a real running server
Testing a `@RestController` by actually starting a full embedded server and firing real HTTP requests at it (over a real network socket) for every test would be slow and heavier than necessary. `MockMvc` simulates the Spring MVC request-handling pipeline directly in-process — no real server/socket involved — letting you assert on status codes, response bodies, and headers with the same API shape as a real request, at a fraction of the cost:
```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired MockMvc mockMvc;

    @Test
    void getUser_returns200() throws Exception {
        mockMvc.perform(get("/api/users/1"))
               .andExpect(status().isOk());
    }
}
```

## Mockito — why it exists
To unit test a Service that depends on a Repository, you don't want to hit a REAL database in every fast unit test run. Mockito lets you create a FAKE Repository object that returns pre-programmed responses, isolating the Service's own logic from the real Repository's behavior entirely — a direct practical benefit of the Dependency Injection pattern from file 11 (since dependencies are injected, not hard-created, they're trivially swappable for fakes in tests).
```java
@Mock
UserRepository userRepository;   // fake, controlled by the test itself

@InjectMocks
UserService userService;          // real service, with the fake repository injected automatically

@Test
void testGetUser() {
    when(userRepository.findById(1L)).thenReturn(Optional.of(new User("John")));  // pre-program the fake's behavior
    User result = userService.getUserById(1L);
    assertEquals("John", result.getName());
}
```

## Testcontainers — a brief mention for realistic integration testing
An in-memory database (H2, commonly used in `@DataJpaTest`) is fast but isn't ALWAYS 100% behaviorally identical to your real production database (e.g. subtle SQL dialect differences from PostgreSQL/MySQL) — sometimes a bug only shows up against the REAL database engine. Testcontainers is a library that spins up a real, throwaway database (in an actual Docker container) just for the duration of a test run, giving genuinely realistic integration tests without needing a permanently-running shared test database — worth knowing this tool exists once testing needs grow beyond the basics covered here.
