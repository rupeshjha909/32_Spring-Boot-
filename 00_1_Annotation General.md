# Important Spring Boot Annotations: A Detailed Reference

> A practical, interview-ready reference to the Spring Boot & Spring annotations you use every day — grouped by purpose, each with **what it does, how it works under the hood, a code example, and the gotchas.** Written for a backend developer building real services. Examples are syntax-checked.

---

## Table of Contents
1. [Bootstrapping](#1-bootstrapping) — `@SpringBootApplication`, `@EnableAutoConfiguration`, `@ComponentScan`, `@SpringBootConfiguration`
2. [Stereotypes (declaring beans)](#2-stereotypes-declaring-beans) — `@Component`, `@Service`, `@Repository`, `@Controller`, `@RestController`
3. [Dependency Injection](#3-dependency-injection) — `@Autowired`, `@Qualifier`, `@Primary`, `@Value`, `@Lazy`
4. [Java Config](#4-java-config) — `@Configuration`, `@Bean`, `@Import`
5. [Web / REST](#5-web--rest) — `@RequestMapping`, `@GetMapping`…, `@RequestBody`, `@PathVariable`, `@RequestParam`, `@ResponseStatus`
6. [Configuration & Properties](#6-configuration--properties) — `@ConfigurationProperties`, `@Value`, `@Profile`, `@PropertySource`
7. [Data / JPA](#7-data--jpa) — `@Entity`, `@Id`, `@GeneratedValue`, `@Column`, `@Table`, `@Query`
8. [Transactions](#8-transactions) — `@Transactional`
9. [Exception Handling](#9-exception-handling) — `@ExceptionHandler`, `@ControllerAdvice`, `@RestControllerAdvice`, `@ResponseStatus`
10. [Conditional / Auto-config](#10-conditional--auto-config) — `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`
11. [Validation](#11-validation) — `@Valid`, `@NotNull`, `@Size`…
12. [Scheduling & Async](#12-scheduling--async) — `@EnableScheduling`, `@Scheduled`, `@EnableAsync`, `@Async`
13. [Lifecycle](#13-lifecycle) — `@PostConstruct`, `@PreDestroy`, `@Scope`
14. [Testing](#14-testing) — `@SpringBootTest`, `@WebMvcTest`, `@MockBean`, `@DataJpaTest`
15. [Cheat Sheet](#15-cheat-sheet)

---

## 1. Bootstrapping

### `@SpringBootApplication`
**What:** the single annotation on your main class that boots the whole app. **It's a meta-annotation** combining three:
```
@SpringBootApplication = @SpringBootConfiguration + @EnableAutoConfiguration + @ComponentScan
```
```java
@SpringBootApplication
public class PaymentsApplication {
    public static void main(String[] args) {
        SpringApplication.run(PaymentsApplication.class, args);
    }
}
```
**Gotcha:** because it enables `@ComponentScan` starting from *this* class's package, the main class must sit in a **root package** above all your other code — otherwise your beans won't be found.

### `@EnableAutoConfiguration`
**What:** switches on Spring Boot's auto-configuration — it inspects the classpath and auto-creates sensible default beans (a DataSource if a JDBC driver is present, embedded Tomcat if web is present, etc.). **How:** loads guarded `@Configuration` classes from each starter's `AutoConfiguration.imports` file. Exclude one with `@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)`.

### `@ComponentScan`
**What:** tells Spring where to scan for your `@Component`-family beans. Defaults to the annotated class's package and below. You can widen it: `@ComponentScan(basePackages = "com.example")`.

### `@SpringBootConfiguration`
**What:** a Boot-specific `@Configuration` marking your main class as a source of bean definitions. You rarely use it directly — it's inside `@SpringBootApplication`.

> 💡 **Interview line:** *`@SpringBootApplication` = config source + auto-config switch + component scan; it must be on a root-package class so scanning reaches all your beans.*

---

## 2. Stereotypes (declaring beans)

These mark a class as a **Spring-managed bean** (discovered by component scan). They're functionally similar — the different names document *intent* and some add behavior.

| Annotation | Use for | Adds |
|:--|:--|:--|
| **`@Component`** | any generic Spring bean | the base stereotype |
| **`@Service`** | business-logic classes | semantic only (readability) |
| **`@Repository`** | data-access classes | **translates DB exceptions** into Spring's `DataAccessException` |
| **`@Controller`** | web MVC controllers (return views) | request mapping support |
| **`@RestController`** | REST controllers (return data/JSON) | `@Controller` + `@ResponseBody` (auto-serializes returns to JSON) |

```java
@Service                       // business logic
public class PaymentService { ... }

@Repository                    // data access — DB exceptions become DataAccessException
public class PaymentDao { ... }

@RestController                // REST endpoints — returns become JSON
public class PaymentController { ... }
```
**Gotcha:** `@RestController` = `@Controller` + `@ResponseBody`, so return values are written straight to the response body as JSON. Use plain `@Controller` only when returning view names (Thymeleaf/JSP).

> 💡 All are `@Component` under the hood; pick the one that names the layer. `@Repository` uniquely adds exception translation.

---

## 3. Dependency Injection

### `@Autowired`
**What:** asks Spring to inject a dependency by type. Works on constructors, setters, or fields.
```java
@Service
public class PaymentService {
    private final PaymentRepository repo;
    public PaymentService(PaymentRepository repo) {   // @Autowired optional on a single constructor
        this.repo = repo;
    }
}
```
**Gotcha / best practice:** prefer **constructor injection** (final fields, testable, fails fast, exposes circular deps) over field injection (`@Autowired` on a field). Since Spring 4.3, `@Autowired` is optional if the class has a single constructor.

### `@Qualifier` and `@Primary`
When **multiple beans of the same type** exist, Spring can't pick one — disambiguate:
- **`@Primary`** on one bean → the default choice.
- **`@Qualifier("name")`** at the injection point → pick a specific one.
```java
@Bean @Primary public PaymentGateway razorpay() { ... }
@Bean @Qualifier("stripe") public PaymentGateway stripe() { ... }

public PaymentService(@Qualifier("stripe") PaymentGateway gw) { ... }   // force Stripe
```
**Gotcha:** without one of these and multiple candidates → `NoUniqueBeanDefinitionException` at startup.

### `@Value`
Injects a single config value (with a default): `@Value("${payments.retries:3}")` (`:3` is the fallback). For many related properties, prefer `@ConfigurationProperties` (§6).

### `@Lazy`
Defers a bean's creation until first use (instead of eagerly at startup). Useful to break a circular dependency or speed startup — but usually a design smell if overused.

> 💡 **Interview line:** *Inject by type via constructor; disambiguate multiple candidates with `@Primary` (default) or `@Qualifier` (specific); `@Value` for single properties.*

---

## 4. Java Config

### `@Configuration` + `@Bean`
**What:** a class that defines beans programmatically. Each `@Bean` method returns an object Spring registers and manages — used for third-party classes you can't annotate.
```java
@Configuration
public class AppConfig {
    @Bean
    public RestTemplate restTemplate() { return new RestTemplate(); }   // a managed bean named "restTemplate"
}
```
**How (subtle):** `@Configuration` classes are **CGLIB-proxied** so that calling one `@Bean` method from another returns the *same* singleton, not a new object. (A plain `@Component` with `@Bean` methods — "lite mode" — doesn't get this.)

### `@Import`
Pulls in another `@Configuration` class's beans: `@Import(SecurityConfig.class)`.

> 💡 Use `@Bean` in a `@Configuration` for beans you can't annotate (library classes); use stereotypes (`@Service` etc.) for your own classes.

---

## 5. Web / REST

### `@RequestMapping` and the shortcuts
`@RequestMapping` maps HTTP requests to handler methods (by path, method, params, headers). The common shortcuts are specializations: **`@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, `@PatchMapping`.**

### `@RequestBody`, `@PathVariable`, `@RequestParam`
Bind parts of the HTTP request to method parameters:
- **`@PathVariable`** — a URL path segment (`/payments/{id}` → `id`).
- **`@RequestParam`** — a query parameter (`?page=2`), with `defaultValue`/`required`.
- **`@RequestBody`** — the request body deserialized (JSON → object via Jackson).

```java
@RestController
@RequestMapping("/api/payments")
public class PaymentController {

    @GetMapping("/{id}")
    public Payment get(@PathVariable Long id) { return service.find(id); }

    @PostMapping
    public ResponseEntity<Payment> create(@RequestBody @Valid PaymentRequest req) {
        return ResponseEntity.status(HttpStatus.CREATED).body(service.create(req));
    }

    @GetMapping
    public List<Payment> search(@RequestParam(defaultValue = "0") int page) {
        return service.page(page);
    }
}
```
**Gotcha:** `@RequestBody` needs valid JSON matching your DTO; pair it with `@Valid` (§11) to validate. `@PathVariable` vs `@RequestParam` is a common mix-up — path segment vs query string.

### `@ResponseStatus`
Sets the HTTP status for a handler or exception: `@ResponseStatus(HttpStatus.CREATED)` on a method, or on a custom exception class so throwing it returns that status.

> 💡 **Interview line:** *`@RestController` + `@GetMapping`/`@PostMapping`; bind with `@PathVariable` (path), `@RequestParam` (query), `@RequestBody` (JSON body); return `ResponseEntity` for full control of status/headers.*

---

## 6. Configuration & Properties

### `@ConfigurationProperties`
**What:** binds a whole group of properties (by prefix) to a typed object — cleaner than many `@Value`s.
```java
@ConfigurationProperties(prefix = "payments")
public class PaymentProps {
    private int retries;            // binds payments.retries
    private String gatewayUrl;      // binds payments.gateway-url
    // getters & setters (or use a record / constructor binding)
    public int getRetries() { return retries; }
    public void setRetries(int r) { this.retries = r; }
    public String getGatewayUrl() { return gatewayUrl; }
    public void setGatewayUrl(String u) { this.gatewayUrl = u; }
}
```
Enable with `@EnableConfigurationProperties(PaymentProps.class)` or annotate the class `@Component`. **Gotcha:** relaxed binding — `payments.gatewayUrl`, `payments.gateway-url`, and `PAYMENTS_GATEWAYURL` all map to `gatewayUrl`.

### `@Value`
Single value with SpEL/placeholder: `@Value("${server.port:8080}")`. Prefer `@ConfigurationProperties` for grouped/typed config.

### `@Profile`
Includes a bean/config only when a profile is active: `@Profile("prod")`. Activate via `spring.profiles.active=prod`. This is how one build behaves differently per environment.

### `@PropertySource`
Loads an additional properties file into the Environment: `@PropertySource("classpath:extra.properties")`.

> 💡 **Interview line:** *`@ConfigurationProperties` for typed, grouped config with relaxed binding; `@Value` for one-offs; `@Profile` to include beans per environment.*

---

## 7. Data / JPA

### `@Entity`, `@Table`, `@Id`, `@GeneratedValue`, `@Column`
Map a class to a database table (JPA/Hibernate):
```java
@Entity
@Table(name = "payments")
public class Payment {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)   // DB auto-increment PK
    private Long id;

    @Column(nullable = false)
    private BigDecimal amount;
}
```
- **`@Entity`** — this class is a persistent entity. **`@Table`** — the table name (optional).
- **`@Id`** — the primary key. **`@GeneratedValue`** — how the PK is generated (`IDENTITY`, `SEQUENCE`, `AUTO`).
- **`@Column`** — column mapping/constraints; **`@Transient`** excludes a field from persistence.

### `@Query` (Spring Data)
On a repository method, supply a custom JPQL/SQL query when derived queries aren't enough:
```java
public interface PaymentRepository extends JpaRepository<Payment, Long> {
    Optional<Payment> findByOrderId(String orderId);              // derived query (from method name)
    @Query("SELECT p FROM Payment p WHERE p.amount > :min")
    List<Payment> findLargerThan(@Param("min") BigDecimal min);  // explicit JPQL
}
```
**Gotcha:** entity classes need a no-arg constructor; `@Transactional` matters for lazy loading (accessing a lazy relation outside a transaction → `LazyInitializationException`).

> 💡 **Interview line:** *`@Entity`/`@Id`/`@GeneratedValue`/`@Column` map class→table; Spring Data derives queries from method names and `@Query` supplies custom JPQL.*

---

## 8. Transactions

### `@Transactional`
**What:** wraps a method in a database transaction — commit on success, roll back on failure. **How:** proxy-based AOP (Spring wraps the bean; the proxy opens/commits/rolls back around the call).
```java
@Service
public class TransferService {
    @Transactional(rollbackFor = Exception.class)
    public void transfer(Long from, Long to, BigDecimal amt) {
        accounts.debit(from, amt);
        accounts.credit(to, amt);       // if this throws, the debit rolls back too
    }
}
```
**The classic gotchas (interviewers love these):**
- **Self-invocation** — calling another `@Transactional` method via `this.method()` bypasses the proxy → no transaction. Fix: call from another bean or self-inject the proxy.
- **Rollback only on unchecked** — by default rolls back on `RuntimeException`/`Error`, **not checked exceptions**; add `rollbackFor = Exception.class`.
- **No `private`/`final` methods** — proxies can't intercept them.
- Also: `propagation` (REQUIRED default, REQUIRES_NEW, NESTED…) and `readOnly = true` for read paths.

> 💡 **Interview line:** *`@Transactional` is proxy-based, so self-invocation bypasses it and it only rolls back on unchecked exceptions unless I set `rollbackFor`.*

---

## 9. Exception Handling

### `@ExceptionHandler`, `@RestControllerAdvice`, `@ResponseStatus`
Turn exceptions into clean HTTP responses:
- **`@ExceptionHandler`** — a method that handles a specific exception (in a controller, or globally in advice).
- **`@ControllerAdvice` / `@RestControllerAdvice`** — apply exception handlers **across all controllers** (the REST one adds `@ResponseBody`). The standard place for global error handling.
- **`@ResponseStatus`** — map a custom exception to an HTTP status automatically.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(PaymentNotFoundException.class)
    public ResponseEntity<ApiError> handle(PaymentNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                             .body(new ApiError("NOT_FOUND", ex.getMessage()));
    }
    @ExceptionHandler(MethodArgumentNotValidException.class)   // @Valid failures → 400
    public ResponseEntity<ApiError> handleValidation(MethodArgumentNotValidException ex) {
        return ResponseEntity.badRequest().body(new ApiError("VALIDATION_ERROR", ex.getMessage()));
    }
}
```
**Gotcha:** never leak stack traces to clients; map each exception type to the right status; one global `@RestControllerAdvice` beats scattered try/catch.

> 💡 **Interview line:** *Centralize errors in a `@RestControllerAdvice` with `@ExceptionHandler` methods mapping each exception to a status + consistent DTO; `@ResponseStatus` for simple custom-exception mapping.*

---

## 10. Conditional / Auto-config

These power auto-configuration and your own conditional beans — a bean is created **only if a condition holds**:
- **`@ConditionalOnClass(X)`** — only if class `X` is on the classpath (library present).
- **`@ConditionalOnMissingBean`** — only if you haven't defined that bean (so **your bean wins**).
- **`@ConditionalOnProperty`** — only if a property has a given value.
- Others: `@ConditionalOnBean`, `@ConditionalOnMissingClass`, `@ConditionalOnWebApplication`.

```java
@Configuration
public class RedisAutoConfig {
    @Bean
    @ConditionalOnClass(RedisClient.class)     // only if Redis library is present
    @ConditionalOnMissingBean                  // only if the user didn't define their own
    public RedisTemplate redisTemplate() { return new RedisTemplate(); }
}
```
**Gotcha:** this is exactly how Spring Boot "backs off" when you define your own bean — very useful in your own starter/config modules too.

> 💡 **Interview line:** *`@ConditionalOnClass` + `@ConditionalOnMissingBean` = the core of auto-config: create a default only when the library is present and the user hasn't overridden it.*

---

## 11. Validation

Bean Validation (JSR-380) annotations declare constraints on DTO fields; `@Valid` triggers validation of a request body:
```java
public class PaymentRequest {
    @NotNull private String orderId;
    @Positive private BigDecimal amount;
    @Size(min = 3, max = 3) private String currency;
    @Email private String receiptEmail;
}

@PostMapping
public Payment create(@RequestBody @Valid PaymentRequest req) { ... }   // @Valid enforces the constraints
```
Common constraints: `@NotNull`, `@NotBlank`, `@NotEmpty`, `@Size`, `@Min`/`@Max`, `@Positive`, `@Email`, `@Pattern`. **Gotcha:** a `@Valid` failure throws `MethodArgumentNotValidException` — handle it in your `@RestControllerAdvice` (§9) to return a clean 400. Needs `spring-boot-starter-validation`.

> 💡 **Interview line:** *Annotate DTO fields with constraints (`@NotNull`, `@Positive`…) and trigger with `@Valid` on `@RequestBody`; catch `MethodArgumentNotValidException` globally for a 400.*

---

## 12. Scheduling & Async

- **`@EnableScheduling`** (on a config class) + **`@Scheduled`** — run a method on a schedule.
- **`@EnableAsync`** + **`@Async`** — run a method on a **separate thread** (returns `void` or `CompletableFuture`).
```java
@Service
public class Jobs {
    @Scheduled(fixedRate = 60000)              // every 60s; also fixedDelay / cron = "0 0 * * * *"
    public void everyMinute() { /* poll, cleanup, ... */ }

    @Async                                      // runs on a background thread pool
    public CompletableFuture<Report> build() {
        return CompletableFuture.completedFuture(new Report());
    }
}
```
**Gotchas:** both are **proxy-based**, so (like `@Transactional`) `@Async`/`@Scheduled` on a self-invoked or `private` method won't work. `@Async` uses a default `SimpleAsyncTaskExecutor` unless you define a proper `Executor` bean — configure a bounded pool for production. A `@Scheduled` method that throws stops future runs unless caught.

> 💡 **Interview line:** *`@Scheduled` (fixedRate/fixedDelay/cron) for periodic jobs, `@Async` for background execution — both proxy-based (no self-invocation), and `@Async` needs a configured thread pool.*

---

## 13. Lifecycle

- **`@PostConstruct`** — a method run **after** the bean is constructed and dependencies injected (initialization work).
- **`@PreDestroy`** — a method run **before** the bean is destroyed (cleanup).
- **`@Scope`** — the bean's scope: `singleton` (default, one per container), `prototype` (new each time), `request`/`session` (web).
```java
@Component
public class CacheWarmer {
    @PostConstruct public void init()   { /* warm the cache after DI */ }
    @PreDestroy   public void cleanup() { /* release resources on shutdown */ }
}
```
**Gotcha:** a `prototype` bean injected into a `singleton` is created **once** (at injection), not per use — use a provider if you need a fresh instance each time.

> 💡 **Interview line:** *`@PostConstruct` runs init after injection; `@PreDestroy` runs cleanup on shutdown; `@Scope` controls instance lifecycle (singleton default).*

---

## 14. Testing

- **`@SpringBootTest`** — loads the full application context for integration tests.
- **`@WebMvcTest(XController.class)`** — loads only the web layer (fast controller tests) with `MockMvc`.
- **`@DataJpaTest`** — loads only the JPA layer (repositories) against an in-memory DB.
- **`@MockBean`** — replaces a bean in the context with a Mockito mock (isolate the unit under test).
```java
@WebMvcTest(PaymentController.class)
class PaymentControllerTest {
    @Autowired MockMvc mvc;
    @MockBean PaymentService service;          // mock the service layer
    @Test void getsPayment() throws Exception {
        when(service.find(1L)).thenReturn(new Payment(...));
        mvc.perform(get("/api/payments/1")).andExpect(status().isOk());
    }
}
```
**Gotcha:** `@SpringBootTest` is heavy (full context) — prefer the sliced tests (`@WebMvcTest`/`@DataJpaTest`) for speed when you only need one layer.

> 💡 **Interview line:** *`@SpringBootTest` for full integration; sliced tests (`@WebMvcTest`, `@DataJpaTest`) for fast layer tests; `@MockBean` to swap a dependency with a mock.*

---

## 15. Cheat Sheet

```
BOOTSTRAP    @SpringBootApplication = @SpringBootConfiguration + @EnableAutoConfiguration + @ComponentScan
STEREOTYPES  @Component (base) · @Service · @Repository (+exception translation) · @Controller · @RestController (+@ResponseBody)
DI           constructor inject; @Autowired (optional on 1 ctor) · @Qualifier/@Primary (disambiguate) · @Value · @Lazy
JAVA CONFIG  @Configuration (+CGLIB proxy) · @Bean (library beans) · @Import
WEB/REST     @RequestMapping / @GetMapping @PostMapping … · @PathVariable (path) · @RequestParam (query) · @RequestBody (JSON) · @ResponseStatus
CONFIG       @ConfigurationProperties (typed, relaxed binding) · @Value (one-off) · @Profile · @PropertySource
JPA          @Entity @Table @Id @GeneratedValue @Column · @Query / derived queries
TX           @Transactional (proxy: no self-invocation; rollback only on unchecked unless rollbackFor)
ERRORS       @RestControllerAdvice + @ExceptionHandler · @ResponseStatus
CONDITIONAL  @ConditionalOnClass · @ConditionalOnMissingBean · @ConditionalOnProperty  (auto-config core)
VALIDATION   @Valid on @RequestBody + @NotNull/@Positive/@Size/@Email…  → MethodArgumentNotValidException
SCHED/ASYNC  @EnableScheduling+@Scheduled · @EnableAsync+@Async  (proxy-based; @Async needs a pool)
LIFECYCLE    @PostConstruct · @PreDestroy · @Scope (singleton default)
TESTING      @SpringBootTest · @WebMvcTest · @DataJpaTest · @MockBean
```

**Three annotations that are secretly proxies (so self-invocation breaks them):** `@Transactional`, `@Async`, `@Cacheable`. Remember they only work when called *through* the Spring proxy.

> **One sentence:** *Spring Boot annotations fall into a few buckets — bootstrap (`@SpringBootApplication`), declare beans (`@Service`/`@RestController`), wire them (`@Autowired`/constructor + `@Qualifier`/`@Primary`), define config beans (`@Configuration`/`@Bean`), map HTTP (`@GetMapping`/`@RequestBody`), bind config (`@ConfigurationProperties`/`@Value`/`@Profile`), map data (`@Entity`/`@Query`), manage transactions (`@Transactional`), handle errors (`@RestControllerAdvice`), and gate auto-config (`@ConditionalOn…`) — and the proxy-based ones (`@Transactional`/`@Async`) break on self-invocation.*
