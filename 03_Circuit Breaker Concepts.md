# The Circuit Breaker Pattern: A Complete Guide

> A thorough, interview-ready and production-minded guide to the **circuit breaker** resilience pattern — what problem it solves, the three-state machine (with a verified simulation), every tuning knob, how it differs from and combines with retries/timeouts/bulkheads/fallbacks, real library configs (Resilience4j), pitfalls, and testing. Written for a backend engineer who runs services that call other services at scale. The state machine is verified with a working implementation.

---

## Table of Contents
1. [The Problem: Cascading Failure](#1-the-problem-cascading-failure)
2. [The Core Idea (the electrical analogy)](#2-the-core-idea-the-electrical-analogy)
3. [The Three States (the state machine)](#3-the-three-states-the-state-machine)
4. [The State Transitions, Verified](#4-the-state-transitions-verified)
5. [How "Failure" Is Measured (count vs rate, sliding windows)](#5-how-failure-is-measured-count-vs-rate-sliding-windows)
6. [Every Tuning Knob](#6-every-tuning-knob)
7. [What Counts as a Failure (and what shouldn't)](#7-what-counts-as-a-failure-and-what-shouldnt)
8. [Fallbacks: What to Do When the Circuit Is Open](#8-fallbacks-what-to-do-when-the-circuit-is-open)
9. [Circuit Breaker vs Retry vs Timeout vs Bulkhead](#9-circuit-breaker-vs-retry-vs-timeout-vs-bulkhead)
10. [Combining Them (the resilience stack order)](#10-combining-them-the-resilience-stack-order)
11. [A Real Implementation (Resilience4j + Spring Boot)](#11-a-real-implementation-resilience4j--spring-boot)
12. [Where to Put the Breaker (per-dependency, distributed concerns)](#12-where-to-put-the-breaker-per-dependency-distributed-concerns)
13. [Observability: Metrics & Alerts](#13-observability-metrics--alerts)
14. [Common Pitfalls](#14-common-pitfalls)
15. [Testing a Circuit Breaker](#15-testing-a-circuit-breaker)
16. [Senior Interview Q&A](#16-senior-interview-qa)
17. [Cheat Sheet](#17-cheat-sheet)
18. [Glossary](#18-glossary)

---

## 1. The Problem: Cascading Failure

In a microservice system, service A calls service B calls service C. Suppose C gets slow or starts failing. Without protection, here's the chain reaction:

```
C slows down (say each call now hangs for 30s instead of 50ms)
   → B's threads all block waiting on C
   → B's thread pool / connection pool exhausts
   → B stops responding to A (even for requests that don't need C)
   → A's threads block waiting on B
   → A exhausts → the whole system is down
```
This is a **cascading failure**: one struggling dependency drags down everything that depends on it, directly or transitively. The cruelty is that **retrying makes it worse** — piling more requests onto an already-drowning C, and holding your own threads hostage waiting for calls that are going to fail anyway.

The circuit breaker's job is to **stop calling a dependency that's clearly failing**, so that (a) you fail fast instead of hanging, freeing your own resources, and (b) you give the sick dependency room to recover instead of hammering it.

> 💡 **The one-sentence problem statement:** *when a downstream dependency is failing, continuing to call it wastes your resources (threads/connections hang), worsens the downstream's overload, and cascades the outage upward — so you need something that detects the failure and stops calling until it recovers.*

---

## 2. The Core Idea (the electrical analogy)

The pattern is named after the **electrical circuit breaker** in your house. When there's a fault (a short, an overload), the breaker **trips** — it cuts the circuit to prevent a fire. Once the problem is fixed, you **reset** it and power flows again.

A software circuit breaker wraps a call to a dependency and does the same:
- Normally, calls pass through (circuit **closed** — current flows).
- If failures cross a threshold, the breaker **trips open** — further calls are **rejected instantly** (short-circuited) without even attempting the dependency.
- After a cooldown, it cautiously **tests** whether the dependency recovered, and either resets (closes) or trips again.

The critical behavioral win: when the circuit is **open**, a call **fails immediately** (microseconds) instead of hanging for a timeout (seconds) — this is what frees your threads and stops the cascade.

> 💡 **"Fail fast, don't hang."** A closed breaker lets calls through; an open breaker rejects them instantly so your service stays responsive and the sick dependency gets breathing room.

---

## 3. The Three States (the state machine)

A circuit breaker is a state machine with three states:

```
        failures cross threshold
  ┌────────────────────────────────────────┐
  │                                         ▼
┌─────────┐   cooldown elapsed      ┌──────────────┐
│ CLOSED  │ ─────────────────────►  │     OPEN     │
│ (normal)│ ◄─────────────────────  │ (reject fast)│
└─────────┘   probes succeed        └──────────────┘
     ▲                                     │ cooldown elapsed
     │  probe(s) succeed                   ▼
     │                              ┌──────────────┐
     └──────────────────────────── │  HALF_OPEN   │
                                    │ (trial calls)│ ── probe fails ──► back to OPEN
                                    └──────────────┘
```

**CLOSED (normal operation).** Calls pass through to the dependency. The breaker counts failures. As long as failures stay under the threshold, it stays closed. A success resets/decays the failure count.

**OPEN (tripped — failing fast).** The failure threshold was crossed, so the breaker is tripped. **Every call is rejected immediately** without touching the dependency (a "short-circuit" — often throwing `CallNotPermittedException` or invoking a fallback). It stays open for a configured **cooldown / reset timeout**. This is the state that protects you.

**HALF_OPEN (testing recovery).** After the cooldown, the breaker lets a **limited number of trial calls** through to probe whether the dependency has recovered:
- If those probes **succeed** (enough of them), it concludes recovery and transitions back to **CLOSED**.
- If a probe **fails**, it concludes the dependency is still sick and trips back to **OPEN** (restarting the cooldown).

HALF_OPEN is the safety valve that prevents flapping straight back to full traffic before the dependency is actually healthy.

> 💡 **The three states in one line:** *CLOSED = let calls through and watch; OPEN = reject instantly for a cooldown; HALF_OPEN = let a few probes through to decide whether to close (recovered) or re-open (still sick).*

---

## 4. The State Transitions, Verified

Here's the behavior confirmed by a working implementation (thresholds: trip after 5 failures, 0.3s cooldown, close after 2 successful probes):

```
1) after 3 healthy calls          → state = CLOSED         (normal traffic passes)
2) after 5 consecutive failures   → state = OPEN           (tripped)
3) while OPEN, 4 calls            → all 4 SHORT-CIRCUITED  (downstream NOT called — fail fast)
4) after cooldown + 2 good probes → state = CLOSED         (recovered via HALF_OPEN)
5) a probe fails in HALF_OPEN     → state = OPEN           (re-tripped, cooldown restarts)

Transition trace: →OPEN →HALF_OPEN →CLOSED →OPEN →HALF_OPEN →OPEN
```

The two things to notice: in step 3 the breaker **saved four calls to a failing dependency** (fail-fast, no hanging), and in step 5 a single failed probe **immediately re-opened** the circuit rather than letting bad traffic back in. That re-open-on-probe-failure is what makes recovery cautious.

The essential control flow of `call()`:
```
if state == OPEN:
    if cooldown elapsed → move to HALF_OPEN
    else                → reject immediately (short-circuit)      # the protective path
if state == HALF_OPEN and trial quota exhausted → reject
try:
    result = dependency()          # only reached in CLOSED or an allowed HALF_OPEN probe
    on_success()                   # HALF_OPEN: count toward closing; CLOSED: reset failures
    return result
except:
    on_failure()                   # HALF_OPEN: re-open; CLOSED: increment, trip if over threshold
    throw
```

---

## 5. How "Failure" Is Measured (count vs rate, sliding windows)

*When* does the breaker trip? There are two common strategies, and real libraries use windows, not a naive global counter:

**Count-based trip:** trip after N consecutive failures (simple, what the demo used). Downside: doesn't account for volume — 5 failures out of 5 is very different from 5 out of 5000.

**Rate-based trip (better for production):** trip when the **failure percentage** over a window exceeds a threshold — e.g., "open if ≥50% of the last 100 calls failed." This scales with traffic and is the default in mature libraries.

**Sliding windows** — the window over which the rate is computed:
- **Count-based window** — the last N calls (e.g., last 100 requests).
- **Time-based window** — all calls in the last T seconds (e.g., last 10 seconds), usually bucketed into sub-windows that roll over.

**Minimum number of calls** — a crucial guard: don't evaluate the failure rate until you've seen enough calls (e.g., at least 10). Otherwise 1 failure out of 1 call = 100% failure rate would trip the breaker on almost no evidence.

> 💡 **Rate + minimum-calls beats a raw count.** "Open if ≥50% of the last 100 calls fail, but only once at least 20 calls have been observed" is far more robust than "open after 5 failures" — it won't trip on a tiny sample or ignore volume.

---

## 6. Every Tuning Knob

The parameters you'll actually configure, and how to think about each:

| Knob | What it controls | Typical / guidance |
|:--|:--|:--|
| **failureRateThreshold** | % of failures in the window that trips OPEN | 50% is a common default; lower = more sensitive |
| **slowCallRateThreshold** | % of *slow* calls (over a duration) that trips OPEN | treat slow calls as failures too (a hung dep is as bad as a failing one) |
| **slowCallDurationThreshold** | what counts as "slow" | set near your acceptable latency (e.g., 2s) |
| **slidingWindowType** | COUNT_BASED or TIME_BASED | time-based for steady traffic, count-based for bursty |
| **slidingWindowSize** | N calls or T seconds evaluated | 100 calls / 10s are common starting points |
| **minimumNumberOfCalls** | calls required before evaluating rate | 10–20; prevents tripping on tiny samples |
| **waitDurationInOpenState** | cooldown before HALF_OPEN | 5–60s; long enough for the dep to recover, short enough to retry recovery |
| **permittedNumberOfCallsInHalfOpenState** | probe calls allowed while HALF_OPEN | small (3–10) |
| **(success criteria to close)** | how many probes must succeed | close when the probe window's failure rate is under threshold |

The core trade-off across all of them: **sensitive vs stable.** Trip too eagerly (low threshold, small window, tiny min-calls) → the breaker opens on transient blips and needlessly denies service. Trip too reluctantly → it opens too late and you've already suffered the cascade. Tune with real latency/error data, not guesses.

---

## 7. What Counts as a Failure (and what shouldn't)

A subtle but important design decision: **not every exception should count toward tripping the breaker.**

- **Should count** (the dependency is unhealthy): timeouts, connection refused, 5xx responses, slow calls over the threshold.
- **Should NOT count** (the dependency is fine — the problem is the request): 4xx client errors like `400 Bad Request`, `404 Not Found`, `422 Validation`. These mean *your request* was wrong, not that the dependency is down. If you count them, a burst of bad user input could trip the breaker and deny service to everyone — a self-inflicted outage.

Libraries let you configure this with `recordExceptions` / `ignoreExceptions` (or a predicate). Get this right: **only "the dependency is sick" signals should open the circuit.**

> 💡 **Rule:** count server-side/transport failures and slow calls; ignore client errors (4xx). A breaker that trips on validation errors is worse than no breaker.

---

## 8. Fallbacks: What to Do When the Circuit Is Open

When the breaker rejects a call (open state), you shouldn't just throw an error at the user if you can avoid it. Options, best-effort first:

- **Serve stale/cached data** — return the last known good value (e.g., a cached product price). Great for reads.
- **Return a sensible default** — an empty list, a "recommendations unavailable" placeholder, a degraded but functional response.
- **Queue for later** — for writes that can be deferred, enqueue and process when the dependency recovers (async).
- **Call a secondary source** — a backup service or region.
- **Fail gracefully** — a clear, fast error ("try again shortly") beats a 30-second hang.

This is **graceful degradation**: the feature that depends on the sick service is reduced or disabled, but the rest of your service keeps working. The circuit breaker is what *enables* the fallback to kick in instantly instead of after a timeout.

> 💡 The breaker + fallback together turn "the whole page hangs because recommendations are down" into "the page loads fine, recommendations show a placeholder."

---

## 9. Circuit Breaker vs Retry vs Timeout vs Bulkhead

These are distinct resilience patterns that people conflate. Each solves a different part of the problem:

| Pattern | What it does | Solves |
|:--|:--|:--|
| **Timeout** | give up on a call after T | stops a single call from hanging forever |
| **Retry** | re-attempt a failed call (with backoff) | rides out *transient* blips (a dropped packet) |
| **Circuit breaker** | stop calling a dep that's *persistently* failing | stops cascading failure; gives the dep room to recover |
| **Bulkhead** | isolate resources per dependency (separate thread/connection pools) | one sick dep can't exhaust the resources of others |
| **Rate limiter** | cap the request rate | protects a dep (and you) from overload |
| **Fallback** | alternative response when the call fails/short-circuits | graceful degradation |

**Retry vs circuit breaker** is the pairing people most often get wrong: **retry is for transient failures; the breaker is for sustained ones.** Retrying against a dependency that's *down* just piles on load — so the breaker must sit **outside** the retry (once open, it stops the retries entirely). And retries should always have **exponential backoff + jitter**, never tight loops.

**Bulkhead** is the underappreciated one: even with a breaker, give each downstream its own thread/connection pool so a slow dependency can't consume all your threads *before* the breaker trips. Named after ship bulkheads — a breach in one compartment doesn't sink the ship.

---

## 10. Combining Them (the resilience stack order)

In production you use several together, and **the order they wrap the call matters.** A common, sensible ordering (outermost → innermost):

```
Fallback( Retry( CircuitBreaker( Bulkhead( TimeLimiter( actual call ) ) ) ) )
```
Read inside-out:
1. **TimeLimiter/Timeout** — bound each individual attempt so it can't hang.
2. **Bulkhead** — cap concurrent calls to this dependency (isolate its resources).
3. **CircuitBreaker** — if the dependency is persistently failing, short-circuit.
4. **Retry** — retry transient failures *with backoff* — but because the breaker is inside it, once the breaker is open the retry immediately gets a fast rejection and stops hammering.
5. **Fallback** — whatever bubbles out (retries exhausted or circuit open) → serve the degraded response.

The key relationship again: **breaker inside retry** so that an open circuit halts retries rather than the retry defeating the breaker's protection. (Resilience4j's documented decoration order reflects this.)

> 💡 **Order matters:** timeout each attempt → isolate with a bulkhead → break on sustained failure → retry transient blips (outside the breaker) → fall back if all else fails.

---

## 11. A Real Implementation (Resilience4j + Spring Boot)

**Resilience4j** is the standard circuit-breaker library for modern Java/Spring Boot (Netflix Hystrix, the old default, is in maintenance mode). Config lives in `application.yml`:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentsService:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 100
        minimum-number-of-calls: 20            # don't evaluate rate below 20 calls
        failure-rate-threshold: 50             # open if >= 50% fail
        slow-call-rate-threshold: 80           # open if >= 80% are slow
        slow-call-duration-threshold: 2s       # "slow" = over 2 seconds
        wait-duration-in-open-state: 10s       # cooldown before HALF_OPEN
        permitted-number-of-calls-in-half-open-state: 5
        automatic-transition-from-open-to-half-open-enabled: true
        record-exceptions:                     # these count as failures
          - java.io.IOException
          - java.util.concurrent.TimeoutException
        ignore-exceptions:                     # these do NOT (client errors)
          - com.example.BadRequestException
```

Usage with annotations (with a fallback method):
```java
@Service
public class PaymentClient {

    @CircuitBreaker(name = "paymentsService", fallbackMethod = "chargeFallback")
    @Retry(name = "paymentsService")           // retry sits OUTSIDE the breaker
    @TimeLimiter(name = "paymentsService")
    public CompletableFuture<Receipt> charge(Order order) {
        return CompletableFuture.supplyAsync(() -> psp.charge(order));   // the real call
    }

    // fallback must match the signature + a Throwable param
    private CompletableFuture<Receipt> chargeFallback(Order order, Throwable t) {
        // e.g. queue for later, or return a "pending" receipt — graceful degradation
        return CompletableFuture.completedFuture(Receipt.pending(order));
    }
}
```
The breaker for `paymentsService` tracks the last 100 calls; once ≥50% fail (or ≥80% are slow), it opens for 10s, rejecting calls fast (routing to `chargeFallback`), then lets 5 probes through in HALF_OPEN to decide whether to close.

---

## 12. Where to Put the Breaker (per-dependency, distributed concerns)

- **One breaker per dependency (per downstream), not one global breaker.** If payments and inventory are separate services, they need separate breakers — inventory being down shouldn't stop payment calls. Often even **per endpoint/operation** if they have different reliability.
- **State is usually per-instance (in-memory).** Each pod/instance of your service keeps its own breaker state. That's normally fine (each instance independently learns the dependency is down) and avoids a network hop on every call. Some setups share state (e.g., via Redis) for faster cluster-wide reaction, at the cost of complexity and a dependency on Redis.
- **Client-side vs at the gateway/mesh.** Breakers can live in your service (library like Resilience4j), or in a **service mesh** (Istio/Envoy) / API gateway as infrastructure — no code change, applied uniformly. Many orgs use both: mesh-level for blanket protection, library-level for fine-grained per-call fallbacks.

---

## 13. Observability: Metrics & Alerts

A circuit breaker is only useful if you can *see* it. Emit and watch:
- **State transitions** — every CLOSED→OPEN is a signal a dependency is failing; alert on it.
- **Current state per breaker** — dashboards showing which breakers are open right now.
- **Failure rate, slow-call rate, call volume** per breaker.
- **Number of short-circuited (rejected) calls** — how much traffic the breaker is currently shedding.

Resilience4j publishes these to **Micrometer → Prometheus/Grafana** and emits events (`onStateTransition`, `onCallNotPermitted`). **Alert on OPEN transitions** — a breaker opening is often your earliest warning of a downstream outage, sometimes before the downstream's own alerts fire.

---

## 14. Common Pitfalls

- **Counting 4xx client errors as failures** → a burst of bad requests trips the breaker and denies everyone (self-inflicted outage). Ignore client errors.
- **`minimumNumberOfCalls` too low** → trips on a tiny sample (1/1 = 100%). Set it high enough to be statistically meaningful.
- **Retry *outside* the breaker but without backoff** → retries hammer a dying dependency. Always exponential backoff + jitter, and keep the breaker inside the retry.
- **One global breaker for all dependencies** → one sick dep stops calls to healthy ones. One breaker per dependency.
- **No fallback** → the breaker opens and users just get errors; you got fail-fast but not graceful degradation. Pair with a fallback.
- **Cooldown too short** → flaps open/closed rapidly, sending bursts at a not-yet-recovered dependency. Too long → slow to recover after the dep is healthy again.
- **Ignoring slow calls** → a dependency that's *slow* (not erroring) still exhausts your threads; count slow calls, not just errors.
- **No bulkhead** → the breaker trips based on failures, but a slow dep can exhaust your thread pool *before* enough failures accumulate. Isolate resources too.
- **No observability** → you can't tell a breaker is open, so you debug the wrong thing. Emit metrics and alert on transitions.

---

## 15. Testing a Circuit Breaker

- **Unit-test the state machine** — force failures and assert it trips to OPEN at the threshold; assert calls are rejected while OPEN; advance the clock past the cooldown and assert HALF_OPEN; feed successes and assert it closes; feed a probe failure and assert it re-opens. (Exactly the five checks verified in §4.)
- **Test failure classification** — a 4xx should NOT trip; a 5xx/timeout SHOULD.
- **Test the fallback** — when the circuit is open, assert the fallback path returns the degraded response.
- **Integration / chaos testing** — use a fault-injection tool or a stubbed downstream that returns errors/latency, and verify the breaker opens, your service stays responsive (fails fast), and recovers when the stub heals. Fault injection in a service mesh (Istio) is handy here.
- **Load test the open path** — confirm that when open, rejected calls are genuinely fast (microseconds) and don't consume threads.

---

## 16. Senior Interview Q&A

**Q1. What problem does a circuit breaker solve?** Cascading failure: when a downstream is failing/slow, continuing to call it hangs your threads, worsens the downstream's overload, and propagates the outage upward. The breaker stops calling a failing dependency so you fail fast and give it room to recover.

**Q2. Explain the three states.** CLOSED (calls pass, count failures), OPEN (trip crossed → reject instantly for a cooldown), HALF_OPEN (after cooldown, allow a few probe calls → close if they succeed, re-open if any fail).

**Q3. When does it trip?** When the failure (or slow-call) rate over a sliding window exceeds a threshold, once a minimum number of calls has been observed — e.g., ≥50% of the last 100 calls fail, min 20 calls.

**Q4. Circuit breaker vs retry?** Retry is for *transient* failures (backoff + jitter); the breaker is for *sustained* ones. The breaker must sit inside the retry so an open circuit stops the retries rather than retries hammering a dead dependency.

**Q5. What should NOT count as a failure?** Client errors (4xx) — the request was bad, not the dependency. Counting them lets bad input trip the breaker and cause a self-inflicted outage. Count 5xx, timeouts, and slow calls.

**Q6. What's HALF_OPEN for?** To test recovery cautiously — let a limited number of trial calls through instead of slamming full traffic back onto a possibly-still-sick dependency; close only if the probes succeed.

**Q7. Where does breaker state live?** Usually per service instance, in memory (no per-call network hop). Optionally shared (Redis) for cluster-wide reaction, at the cost of complexity.

**Q8. How does it combine with other patterns?** Timeout (bound each attempt) + bulkhead (isolate resources) + breaker (stop sustained failures) + retry with backoff (transient) + fallback (degrade gracefully). Order: fallback(retry(breaker(bulkhead(timeout(call))))).

**Q9. What library would you use?** Resilience4j (Hystrix is in maintenance mode), or mesh-level breakers via Istio/Envoy. Configure window, thresholds, cooldown, half-open probes, and exception classification; export metrics to Micrometer/Prometheus and alert on OPEN transitions.

**Q10. What's a slow-call threshold and why include it?** A dependency that's slow (not erroring) still exhausts your threads. Counting calls over a duration threshold (e.g., 2s) as failures lets the breaker trip on latency, not just errors.

---

## 17. Cheat Sheet

**What/why:** stop calling a persistently-failing dependency → fail fast (don't hang) + give it room to recover → prevent cascading failure.

**Three states:**
```
CLOSED    → calls pass, count failures; trip when rate over window > threshold
OPEN      → reject instantly (short-circuit) for a cooldown; downstream NOT called
HALF_OPEN → after cooldown, allow N probes → succeed: CLOSED · any fail: OPEN
```
**Verified transitions:** 5 failures→OPEN · open calls short-circuit · cooldown+2 probes→CLOSED · probe fail→OPEN.

**Trip on:** failure-rate (+ slow-call-rate) over a sliding window, after minimum-number-of-calls. Not a raw count.

**Count as failure:** 5xx, timeouts, slow calls. **Ignore:** 4xx client errors.

**Knobs:** failureRateThreshold · slowCallRate/Duration · slidingWindow (type/size) · minimumNumberOfCalls · waitDurationInOpenState (cooldown) · permittedCallsInHalfOpen.

**Combine (outer→inner):** Fallback( Retry( CircuitBreaker( Bulkhead( Timeout( call ))))) — breaker INSIDE retry.

**Library:** Resilience4j (`@CircuitBreaker(name, fallbackMethod)`), metrics → Micrometer/Prometheus, alert on OPEN.

**Don't:** count 4xx · set min-calls too low · retry without backoff · one global breaker · skip the fallback · ignore slow calls · forget the bulkhead.

> **One sentence:** *A circuit breaker wraps calls to a dependency and trips OPEN when its failure/slow-call rate over a sliding window crosses a threshold, rejecting calls instantly so you fail fast instead of hanging and the sick dependency gets room to recover; after a cooldown it probes in HALF_OPEN and either closes (recovered) or re-opens (still sick) — and in production you pair it with timeouts, bulkheads, backoff-retries, and fallbacks, count only real dependency failures (not 4xx), and alert on every OPEN transition.*

---

## 18. Glossary
- **Circuit breaker** — a wrapper that stops calls to a failing dependency to prevent cascading failure.
- **Cascading failure** — one failing dependency dragging down everything that depends on it.
- **CLOSED / OPEN / HALF_OPEN** — pass calls / reject fast / allow trial probes.
- **Trip** — transition to OPEN when the failure threshold is crossed.
- **Short-circuit** — rejecting a call instantly without attempting the dependency (the OPEN behavior).
- **Cooldown / waitDurationInOpenState** — how long the breaker stays OPEN before probing.
- **Sliding window** — the recent calls (count- or time-based) over which the failure rate is computed.
- **Failure-rate threshold** — the % of failures that trips the breaker.
- **Slow-call threshold** — calls over a duration counted as failures (latency, not errors).
- **minimumNumberOfCalls** — calls required before the rate is evaluated (avoids tiny-sample trips).
- **Probe / trial call** — a call allowed in HALF_OPEN to test recovery.
- **Fallback** — the degraded response served when a call fails or is short-circuited.
- **Graceful degradation** — reducing/disabling one feature while the rest of the service works.
- **Bulkhead** — isolating resources (pools) per dependency so one can't starve others.
- **Retry (with backoff + jitter)** — re-attempting transient failures without hammering.
- **Timeout / TimeLimiter** — bounding how long a single call may take.
- **Resilience4j** — the standard modern Java circuit-breaker library (Hystrix is legacy).
- **Service mesh (Istio/Envoy)** — infra layer that can provide breakers without app code.
