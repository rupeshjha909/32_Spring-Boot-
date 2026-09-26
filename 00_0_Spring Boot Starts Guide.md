# How Spring Boot Starts and Runs: A Detailed Guide

> A thorough, interview-ready walkthrough of what actually happens from `main()` to a running Spring Boot application — `SpringApplication.run()` step by step, `@SpringBootApplication` decomposed, the auto-configuration mechanism, the `ApplicationContext` refresh lifecycle, the embedded Tomcat startup, property/profile resolution, the executable fat-JAR, and how a request is served once up. The two "magic" mechanisms — conditional auto-configuration and property precedence — are illustrated with verified simulations.

---

## Table of Contents
1. [What "Spring Boot Starts" Even Means](#1-what-spring-boot-starts-even-means)
2. [The Entry Point: `main()` → `SpringApplication.run()`](#2-the-entry-point-main--springapplicationrun)
3. [`@SpringBootApplication` Decomposed](#3-springbootapplication-decomposed)
4. [The Startup Sequence of `SpringApplication.run()`](#4-the-startup-sequence-of-springapplicationrun)
5. [The Environment: Properties & Profiles](#5-the-environment-properties--profiles)
6. [Auto-Configuration — The Heart of the Magic](#6-auto-configuration--the-heart-of-the-magic)
7. [Component Scanning & the IoC Container](#7-component-scanning--the-ioc-container)
8. [The `ApplicationContext.refresh()` Lifecycle](#8-the-applicationcontextrefresh-lifecycle)
9. [The Embedded Server (Tomcat) Startup](#9-the-embedded-server-tomcat-startup)
10. [Runners: Code That Runs After Startup](#10-runners-code-that-runs-after-startup)
11. [The Executable Fat JAR — Structure & Launch](#11-the-executable-fat-jar--structure--launch)
12. [How a Request Is Served Once Running](#12-how-a-request-is-served-once-running)
13. [Startup Failures & Debugging](#13-startup-failures--debugging)
14. [Senior Interview Q&A](#14-senior-interview-qa)
15. [Cheat Sheet](#15-cheat-sheet)
16. [Glossary](#16-glossary)

---

## 1. What "Spring Boot Starts" Even Means

Before Spring Boot, running a Java web app meant: configure a mountain of XML/Java beans by hand, build a WAR file, install and configure an external servlet container (Tomcat), and deploy the WAR into it. Spring Boot collapses all of that into: **one `main()` method, one `run()` call, and a self-contained executable JAR with the server embedded inside it.**

Three things Spring Boot does that make this possible, and that this guide explains:
1. **Auto-configuration** — it looks at what's on your classpath and **automatically configures sensible default beans** (a DataSource if a JDBC driver is present, an embedded Tomcat if a web starter is present, a Jackson JSON mapper, etc.) so you write almost no config.
2. **Embedded server** — Tomcat (or Jetty/Undertow) runs **inside your app**, started by your `main()`. No external container to install; you run a plain JAR.
3. **Starters + opinionated defaults** — dependency bundles (`spring-boot-starter-web`, etc.) bring a coherent, version-aligned set of libraries, and Boot picks reasonable defaults you can override.

> 💡 **The one idea:** Spring Boot turns "configure everything and deploy a WAR to an external server" into "run a JAR whose `main()` boots an embedded server and auto-configures beans by inspecting the classpath." The rest of this doc is *how* that happens.

---

## 2. The Entry Point: `main()` → `SpringApplication.run()`

Every Spring Boot app starts from an ordinary Java `main` method:

```java
@SpringBootApplication
public class PaymentsApplication {
    public static void main(String[] args) {
        SpringApplication.run(PaymentsApplication.class, args);
    }
}
```

Two lines carry the entire framework:
- **`@SpringBootApplication`** — the annotation that turns on component scanning and auto-configuration (§3).
- **`SpringApplication.run(...)`** — the single call that bootstraps everything: builds the Spring container, runs auto-configuration, starts the embedded server, and returns a fully-initialized `ApplicationContext`. Everything in §4 happens inside this call.

`SpringApplication.run` is a static convenience that creates a `SpringApplication` instance and calls its instance `run(args)`. You can also build it manually (`new SpringApplicationBuilder(...)`) to customize banners, listeners, profiles, etc.

---

## 3. `@SpringBootApplication` Decomposed

`@SpringBootApplication` is a **meta-annotation** — a convenience that combines three annotations. Knowing the three is a classic interview question:

```
@SpringBootApplication  =  @SpringBootConfiguration   (this class is a source of bean definitions;
                                                        a specialized @Configuration)
                        +  @EnableAutoConfiguration    (turn ON auto-configuration — §6)
                        +  @ComponentScan              (scan THIS package and sub-packages for
                                                        @Component/@Service/@Repository/@Controller)
```

- **`@SpringBootConfiguration`** — marks the class as a configuration class (holds `@Bean` methods); a Boot-specific flavor of `@Configuration`.
- **`@EnableAutoConfiguration`** — the switch that triggers Spring Boot to auto-configure beans based on the classpath (the mechanism in §6).
- **`@ComponentScan`** — tells Spring to discover your own `@Component`-annotated classes, **starting from the package of the annotated class**. (This is why your main class should sit in a **root/top package** — everything below it is scanned; anything in a sibling/parent package is missed. A very common beginner bug.)

> 💡 **What to say:** *`@SpringBootApplication` = `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan`. Auto-config configures library beans from the classpath; component-scan discovers my own beans from the main class's package downward — so the main class must live in the root package.*

---

## 4. The Startup Sequence of `SpringApplication.run()`

This is the core of the topic — the ordered steps `run()` executes. Know this sequence:

```
1. Create the SpringApplication:
   - deduce the APPLICATION TYPE from the classpath: SERVLET (Spring MVC), REACTIVE (WebFlux),
     or NONE (no web) → decides which ApplicationContext & server to create.
   - load ApplicationContextInitializers and ApplicationListeners from META-INF/spring.factories.

2. Get SpringApplicationRunListeners and fire "starting".

3. Prepare the ENVIRONMENT:
   - read property sources (application.yml/properties, env vars, command-line args, ...) — §5
   - activate PROFILES (dev/prod/...).

4. Print the BANNER (the Spring ASCII art; customizable/disable-able).

5. Create the ApplicationContext of the right type:
   - servlet web → AnnotationConfigServletWebServerApplicationContext
   - reactive    → AnnotationConfigReactiveWebServerApplicationContext
   - none        → AnnotationConfigApplicationContext

6. Prepare the context:
   - apply the initializers, register the primary source (your @SpringBootApplication class),
     wire the environment.

7. REFRESH the context  ◄── the big one (§8):
   - process @Configuration classes, run COMPONENT SCAN (§7),
   - run AUTO-CONFIGURATION (§6) → register conditional beans,
   - instantiate & wire all singleton beans (dependency injection),
   - for web apps: CREATE & START the EMBEDDED SERVER (Tomcat) (§9).

8. Fire "started" listeners.

9. Call RUNNERS: ApplicationRunner / CommandLineRunner beans (§10) — your post-startup code.

10. Fire "running"; run() returns the fully-initialized ApplicationContext.
```

The single most important step is **#7, `refresh()`** — that's where beans are actually created and the server starts. Steps 1–6 set the stage; refresh does the work; 8–10 are post-startup hooks.

> 💡 **The sequence in one line:** *create the app (deduce web type, load listeners) → prepare the environment (properties + profiles) → create the right ApplicationContext → **refresh it** (scan, auto-configure, instantiate beans, start Tomcat) → run your runners → return the context.*

---

## 5. The Environment: Properties & Profiles

Early in startup (step 3), Spring builds the **`Environment`** — the unified view of all configuration. Two parts:

**Property sources, in precedence order.** The same key can be set in many places; **higher-priority sources override lower ones.** The order (high → low, simplified) is roughly:
```
command-line args (--server.port=7000)   ← highest, wins
OS environment variables / system properties
application-{profile}.yml (profile-specific)
application.yml / application.properties  ← lowest of the common ones
```
Verified with a resolver simulation:
```
server.port = 7000  (won by command-line --arg, over profile-yml's 9090 and application.yml's 8080)
log.level   = WARN  (won by OS environment variable, over application.yml's INFO)
```
This precedence is *why* you can bake defaults into `application.yml` and override them per environment via env vars or command-line flags without rebuilding — the 12-factor config model.

**Profiles.** A **profile** (`dev`, `prod`, `test`) selects a set of beans/config. Activate with `spring.profiles.active=prod`. Then `application-prod.yml` is layered on top, and beans annotated `@Profile("prod")` are included. This is how one build runs differently per environment.

---

## 6. Auto-Configuration — The Heart of the Magic

This is *the* Spring Boot mechanism to understand. Triggered by `@EnableAutoConfiguration`, it asks: **"given what's on the classpath and what the user has already defined, what beans should I create automatically?"**

**How it finds candidates.** `@EnableAutoConfiguration` imports a special selector that reads a list of **auto-configuration classes** from a well-known file in every starter jar: `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (in older Boot, the `EnableAutoConfiguration` key in `META-INF/spring.factories`). Each listed class (e.g. `DataSourceAutoConfiguration`, `KafkaAutoConfiguration`) is a `@Configuration` full of `@Bean` methods — but each is **guarded by conditions.**

**How it decides — `@Conditional` annotations.** Each auto-config bean only activates if its conditions hold. The key ones:
- **`@ConditionalOnClass(X)`** — only if class `X` is on the classpath (i.e., the relevant library is present).
- **`@ConditionalOnMissingBean`** — only if *you* haven't already defined that bean (so **your bean always wins** — Boot "backs off").
- `@ConditionalOnProperty` — only if a config property has a certain value.
- `@ConditionalOnBean`, `@ConditionalOnMissingClass`, `@ConditionalOnWebApplication`, etc.

Verified simulation of exactly this decision logic:
```
Classpath has HikariCP + RedisLib, user defined no beans:
  CREATE dataSource     — default connection pool          (class present, no user bean)
  SKIP   kafkaTemplate  — @ConditionalOnClass(KafkaLib) not on classpath
  CREATE redisTemplate  — default Redis client

Same classpath, but user defined their OWN dataSource:
  SKIP   dataSource     — @ConditionalOnMissingBean (your bean wins)
```
So auto-configuration is not magic — it's **"if the library is on the classpath AND you didn't configure it yourself, create a sensible default."** Adding `spring-boot-starter-data-redis` puts RedisLib on the classpath → `@ConditionalOnClass` passes → you get a `RedisTemplate` for free. Define your own `RedisTemplate` → `@ConditionalOnMissingBean` makes Boot back off.

**Ordering & override.** Auto-config runs **after** your own configuration (so `@ConditionalOnMissingBean` sees your beans), and auto-config classes can be ordered relative to each other. You can exclude one with `@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)`.

> 💡 **What to say:** *Auto-configuration reads a list of guarded @Configuration classes from every starter's `AutoConfiguration.imports`. Each bean is gated by `@Conditional` — chiefly `@ConditionalOnClass` (library present?) and `@ConditionalOnMissingBean` (you didn't define it?). So Boot creates a default only when the library is on the classpath and you haven't overridden it — your bean always wins.*

---

## 7. Component Scanning & the IoC Container

Alongside auto-config, `@ComponentScan` discovers **your** beans. During refresh, Spring scans the main class's package (and sub-packages) for stereotype annotations — `@Component`, `@Service`, `@Repository`, `@Controller`/`@RestController`, `@Configuration` — and registers each as a **bean definition** in the container.

The container (the **`ApplicationContext`**, an IoC/DI container) then **instantiates** these beans and **injects their dependencies** (constructor injection preferred), resolving the dependency graph in the right order. (This is the DI mechanism covered in the SDE2 guide — Spring Boot just wires it up automatically for scanned + auto-configured beans together.) The end state of startup is a context full of ready, interconnected singleton beans.

---

## 8. The `ApplicationContext.refresh()` Lifecycle

Step 7 of §4 — `refresh()` — is where the container actually comes alive. Its key sub-steps (from Spring core, the `AbstractApplicationContext.refresh()` template):

```
1. prepareBeanFactory        — set up the bean factory (the registry of bean definitions).
2. invokeBeanFactoryPostProcessors
      — process @Configuration classes, run COMPONENT SCAN + AUTO-CONFIGURATION here,
        registering all bean DEFINITIONS. (ConfigurationClassPostProcessor does this.)
3. registerBeanPostProcessors — register BeanPostProcessors (these later create AOP proxies,
        e.g. for @Transactional/@Async — see the AOP guide).
4. finishBeanFactoryInitialization
      — INSTANTIATE all non-lazy singleton beans, run dependency injection,
        apply BeanPostProcessors (proxy creation), call @PostConstruct/init methods.
5. finishRefresh
      — for web apps, START THE EMBEDDED SERVER (Tomcat) here; publish ContextRefreshedEvent.
```

The distinction that matters: **step 2 registers bean *definitions*** (the recipes — from your scan and from auto-config's conditions), while **step 4 creates the *instances*** and wires them. Auto-configuration's conditions are evaluated in step 2, so by the time beans are instantiated, the container knows exactly which beans to build. The embedded server starts near the very end (step 5), once all beans are ready to serve requests.

> 💡 **The bean lifecycle within refresh:** *register all bean definitions (scan + auto-config, evaluating conditions) → register BeanPostProcessors → instantiate singletons + inject dependencies + create proxies + run @PostConstruct → start the embedded server.*

---

## 9. The Embedded Server (Tomcat) Startup

For a web app (`spring-boot-starter-web`), Spring Boot runs an **embedded servlet container** — by default **Tomcat** — inside your process. There's no external Tomcat to install; your `main()` starts it.

How it happens:
- Because a servlet web app type was detected (§4 step 1), Spring created a **`ServletWebServerApplicationContext`**.
- During `refresh()`'s `finishRefresh` (§8 step 5), that context finds a **`ServletWebServerFactory`** bean (auto-configured — `TomcatServletWebServerFactory` because Tomcat is on the classpath) and uses it to **create and start the embedded Tomcat** on the configured port (default 8080).
- It registers the **`DispatcherServlet`** (Spring MVC's front controller) with Tomcat, so incoming HTTP requests are routed into your controllers.

Swapping servers is just a dependency change: exclude Tomcat and add `spring-boot-starter-jetty` or `-undertow`, and auto-config picks that factory instead (`@ConditionalOnClass` again). Once started, the app blocks (the server's threads keep the JVM alive) and serves requests until shut down.

> 💡 **Embedded, not deployed:** Spring Boot doesn't build a WAR for an external Tomcat — it embeds Tomcat as a library, and `refresh()` starts it near the end of bootstrapping. That's why you run a plain `java -jar app.jar`.

---

## 10. Runners: Code That Runs After Startup

Sometimes you want code to run **once, right after the app is fully up** (seed data, warm a cache, kick off a job). Spring Boot calls any beans implementing:
- **`CommandLineRunner`** — `run(String... args)` (raw args).
- **`ApplicationRunner`** — `run(ApplicationArguments args)` (parsed args).

```java
@Component
public class Warmup implements CommandLineRunner {
    public void run(String... args) {
        // runs after the context is refreshed and the server has started
        System.out.println("App is up — warming caches...");
    }
}
```
These run at step 9 of §4 — after the context is refreshed and the server started, before `run()` returns. Use `@Order` to sequence multiple runners.

---

## 11. The Executable Fat JAR — Structure & Launch

Spring Boot packages your app as a single **executable "fat JAR"** (a.k.a. uber-jar) containing your code **and all its dependencies** — so `java -jar app.jar` just works, no classpath juggling. But a normal JAR can't contain other JARs on its classpath, so Boot uses a special layout and launcher:

```
app.jar
├── META-INF/MANIFEST.MF        Main-Class: org.springframework.boot.loader.JarLauncher
│                               Start-Class: com.example.PaymentsApplication   ← YOUR main
├── org/springframework/boot/loader/...   ← Spring Boot's launcher classes
├── BOOT-INF/
│   ├── classes/                ← YOUR compiled classes + resources (application.yml)
│   └── lib/                    ← all dependency JARs, nested as-is
└── ...
```
How it launches:
1. `java -jar app.jar` reads the manifest → runs **`JarLauncher`** (not your class directly).
2. `JarLauncher` sets up a **special class loader** that can load classes from the **nested JARs** in `BOOT-INF/lib/` (standard Java can't do nested jars — this is Boot's `LaunchedURLClassLoader`).
3. It then invokes your real **`Start-Class`** (`main`), and from there §2–§10 proceed as normal.

So the fat JAR is self-contained and portable — exactly what you push to a container image and run on EKS.

> 💡 **Why a special launcher:** a fat JAR nests dependency JARs inside `BOOT-INF/lib/`, which the standard JVM class loader can't read — so Boot's `JarLauncher` installs a custom class loader, then hands off to your `main`.

---

## 12. How a Request Is Served Once Running

Once startup finishes, the app sits in the embedded Tomcat's request loop. A request flows:
```
HTTP request → embedded Tomcat → DispatcherServlet (front controller)
   → HandlerMapping finds your @RestController method
   → arguments bound (@RequestBody via Jackson), method runs
   → return value serialized to JSON (HttpMessageConverter) → HTTP response
```
(This is the Spring MVC request lifecycle covered in the SDE2 guide — the point here is that **startup's job was to get to this ready state**: Tomcat listening, `DispatcherServlet` registered, all controller/service/repository beans instantiated and wired.)

---

## 13. Startup Failures & Debugging

- **`--debug` / the Condition Evaluation Report** — run with `--debug` to see the **auto-configuration report**: exactly which auto-configs matched, which didn't, and *why* (which `@Conditional` passed/failed). Invaluable for "why isn't my bean being created?"
- **`NoSuchBeanDefinitionException`** — a dependency wasn't found: usually the bean isn't component-scanned (main class not in a root package — §3) or an auto-config condition didn't match.
- **`Field ... required a bean ... that could not be found`** — Boot's friendly startup failure message; it also suggests fixes.
- **Port already in use** — embedded Tomcat can't bind `server.port`; change it or free the port.
- **Bean definition conflicts / circular dependencies** — two beans of the same type without `@Primary`/`@Qualifier`, or A↔B constructor cycles (constructor injection surfaces these at startup — a good thing).
- **`spring-boot-starter-actuator` `/actuator/conditions`** — at runtime, inspect the same condition report over HTTP.

---

## 14. Senior Interview Q&A

**Q1. What does `SpringApplication.run()` do?** Creates the SpringApplication (deduces web type, loads listeners), prepares the Environment (properties + profiles), creates the right ApplicationContext, **refreshes** it (component scan + auto-configuration register bean definitions, then singletons are instantiated and wired, and the embedded server starts), calls runners, and returns the context.

**Q2. What is `@SpringBootApplication`?** A meta-annotation = `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan`. Config source + auto-config switch + scan-my-package-downward.

**Q3. How does auto-configuration work?** `@EnableAutoConfiguration` loads guarded `@Configuration` classes listed in each starter's `AutoConfiguration.imports`. Each bean is gated by `@Conditional` — mainly `@ConditionalOnClass` (library present) and `@ConditionalOnMissingBean` (you didn't define it). So Boot creates a default only when the library is on the classpath and you haven't overridden it.

**Q4. Why does my bean override Spring's default?** Because auto-config beans use `@ConditionalOnMissingBean` and run *after* your config — if you defined the bean, Boot backs off. Your bean always wins.

**Q5. Where does the embedded Tomcat come from and when does it start?** A `spring-boot-starter-web` puts Tomcat on the classpath → auto-config provides a `TomcatServletWebServerFactory` → during `refresh()`'s `finishRefresh`, the servlet context creates and starts embedded Tomcat and registers the `DispatcherServlet`.

**Q6. Why must the main class be in a root package?** `@ComponentScan` scans from the main class's package **downward**. Beans in sibling/parent packages aren't found → `NoSuchBeanDefinitionException`.

**Q7. What's the property precedence order?** Command-line args > OS env/system properties > profile-specific `application-{profile}.yml` > `application.yml`. Higher sources override lower, enabling per-environment overrides without rebuilding.

**Q8. What's inside a Spring Boot fat JAR and how does it run?** `BOOT-INF/classes` (your code) + `BOOT-INF/lib` (nested dependency jars) + Boot's loader. The manifest's Main-Class is `JarLauncher`, which installs a class loader that reads nested jars, then invokes your `Start-Class` main.

**Q9. What's the difference between the two `run()` phases — definitions vs instances?** `invokeBeanFactoryPostProcessors` registers bean **definitions** (scan + auto-config, evaluating conditions); `finishBeanFactoryInitialization` **instantiates** the singletons and injects dependencies. Conditions are decided before any instance is built.

**Q10. How do you run code right after startup?** Implement `CommandLineRunner` or `ApplicationRunner` — Boot calls them after the context refreshes and the server starts, before `run()` returns.

---

## 15. Cheat Sheet

**Startup in one flow:**
```
main() → SpringApplication.run(App.class, args)
  → deduce web type + load listeners
  → build Environment (properties [cmdline>env>profile-yml>application.yml] + profiles)
  → create ApplicationContext (servlet/reactive/none)
  → refresh():  scan + AUTO-CONFIGURE (register bean DEFINITIONS, eval @Conditional)
                → instantiate singletons + inject deps + create proxies + @PostConstruct
                → START embedded Tomcat + register DispatcherServlet
  → run CommandLineRunner/ApplicationRunner
  → return ready ApplicationContext
```
**@SpringBootApplication =** `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan`.

**Auto-config (verified):** reads guarded @Configuration from `AutoConfiguration.imports`; `@ConditionalOnClass` (library present?) + `@ConditionalOnMissingBean` (you didn't define it?) → create default only if both hold; **your bean always wins**.

**Property precedence (verified):** command-line > env/system > `application-{profile}.yml` > `application.yml`.

**Embedded server:** Tomcat is a library on the classpath; started during `refresh()`'s finishRefresh; swap via dependency (Jetty/Undertow). Runs `java -jar app.jar` — no external container.

**Fat JAR:** `BOOT-INF/classes` + `BOOT-INF/lib` (nested jars) + Boot loader; manifest Main-Class = `JarLauncher` → custom class loader → your `Start-Class`.

**Debug:** `--debug` → condition evaluation report (why a bean was/wasn't auto-configured); `/actuator/conditions` at runtime.

> **One sentence:** *`SpringApplication.run()` deduces the web type, builds the Environment from ordered property sources, creates the right ApplicationContext, and refreshes it — component-scanning your beans and running classpath-conditional auto-configuration to register bean definitions, then instantiating and wiring the singletons and starting an embedded Tomcat with the DispatcherServlet — after which it runs your CommandLineRunners and hands back a ready-to-serve context, all packaged as a self-contained fat JAR launched by Boot's JarLauncher.*

---

## 16. Glossary
- **SpringApplication.run()** — the call that bootstraps the entire app and returns the ApplicationContext.
- **@SpringBootApplication** — meta-annotation = @SpringBootConfiguration + @EnableAutoConfiguration + @ComponentScan.
- **ApplicationContext** — Spring's IoC/DI container holding all beans; the running app.
- **refresh()** — the context lifecycle method that registers definitions, instantiates beans, and starts the server.
- **Auto-configuration** — automatic bean setup based on the classpath, via guarded @Configuration classes.
- **@EnableAutoConfiguration** — the switch that triggers auto-configuration.
- **AutoConfiguration.imports** — the file in each starter listing auto-config classes.
- **@ConditionalOnClass / @ConditionalOnMissingBean** — the main conditions gating auto-config beans.
- **Starter** — a curated dependency bundle (spring-boot-starter-web, -data-jpa, ...).
- **Component scan** — discovering your @Component/@Service/etc. beans from the main class's package down.
- **Environment / property source / profile** — unified config; ordered config origins; per-environment bean/config set.
- **BeanFactoryPostProcessor / BeanPostProcessor** — hooks that process bean definitions / bean instances (proxies).
- **Embedded server** — Tomcat/Jetty/Undertow running inside the app process.
- **ServletWebServerFactory** — the bean that creates & starts the embedded server.
- **DispatcherServlet** — Spring MVC's front controller registered with the embedded server.
- **CommandLineRunner / ApplicationRunner** — beans run once after startup completes.
- **Fat JAR / JarLauncher / BOOT-INF** — the executable self-contained JAR, its launcher, and its layout.
- **Condition evaluation report** — the `--debug` output explaining which auto-configs matched and why.
