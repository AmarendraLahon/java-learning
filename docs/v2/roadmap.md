# 🗺️ Senior Java Developer Interview Prep — 20-Week Roadmap

**Goal:** Master Java in depth + ecosystem frameworks to crack senior-level (5+ YoE) interviews at product-based and service-based companies.

**Schedule:** 2 hours daily (Mon–Sat) · Sunday reserved for revision + Q&A.
**Total effort:** ~240 hours over 20 weeks (~5 months).

---

## 📊 Phase Overview

| Phase | Weeks | Focus |
|-------|-------|-------|
| **Phase 1 — Core Java Mastery** | 1–7 | Language fundamentals, collections, concurrency, JVM, Java 8+, design |
| **Phase 2 — Spring Ecosystem** | 8–10 | Spring Core, Boot, Data JPA, Hibernate |
| **Phase 3 — Data + APIs** | 11–12 | SQL, NoSQL, REST, API design |
| **Phase 4 — Security & Microservices** | 13–15 | Spring Security, Spring Cloud, messaging |
| **Phase 5 — Testing & DevOps** | 16–17 | JUnit, Mockito, Docker, CI/CD, observability |
| **Phase 6 — System Design & Interview Prep** | 18–20 | System design, DSA, mocks, behavioral |

---

# 🔷 Phase 1 — Core Java Mastery (Weeks 1–7)

## Week 1 — OOP Fundamentals & Language Basics

| Day | Topic | Deliverable |
|-----|-------|-------------|
| Mon | Classes, objects, constructors, `this`/`super`, constructor chaining | Sketch class + run 3 constructor variants |
| Tue | Inheritance, polymorphism (compile-time vs runtime), method overloading vs overriding | Bank/Account hierarchy with polymorphic calls |
| Wed | Abstraction, encapsulation, `abstract` classes, interfaces (default/static/private methods) | Payment gateway interface + 2 impls |
| Thu | Access modifiers, packages, inner classes, anonymous classes, nested types | Iterator impl as inner class |
| Fri | `final`, `static`, `this`, `super`, `instanceof` (pattern matching Java 16+) | Static utility class + final immutable value |
| Sat | String, StringBuilder, StringBuffer, String pool, `intern()` | Benchmark: `+` vs StringBuilder for 10k concat |
| Sun | **Revision + Q&A** — write 20 interview questions & answers from the week |

## Week 2 — Exception Handling & Generics

| Day | Topic | Deliverable |
|-----|-------|-------------|
| Mon | Exception hierarchy, checked vs unchecked, `Error` vs `Exception` | Diagram of Throwable tree |
| Tue | `try-catch-finally`, multi-catch, `throw` vs `throws`, exception chaining | Layered service with wrapped exceptions |
| Wed | Custom exceptions, `try-with-resources`, suppressed exceptions | Custom `InsufficientBalanceException` + try-with-resources |
| Thu | Generics: generic classes, methods, type parameters, diamond operator | Generic `Pair<K,V>` and `Stack<T>` |
| Fri | Type erasure, bridge methods, consequences (no `new T()`, no generic arrays) | Examples of each limitation |
| Sat | Bounded types, wildcards (`? extends`, `? super`), PECS | `Collections.copy`-style generic method |
| Sun | **Revision + Q&A** |

## Week 3 — Collections Deep Dive

| Day | Topic | Deliverable |
|-----|-------|-------------|
| Mon | Collection hierarchy, `Iterable`/`Iterator`/`ListIterator` | Custom iterator implementation |
| Tue | `ArrayList` vs `LinkedList` — internals, complexity, when to use | Benchmark comparing ops |
| Wed | `HashSet`, `LinkedHashSet`, `TreeSet`, `equals`/`hashCode` contract | Break `HashMap` with bad hashCode |
| Thu | `HashMap` internals: buckets, load factor, resizing, treeification (8/6/64) | Implement mini HashMap |
| Fri | `LinkedHashMap` (access-order → LRU), `TreeMap`, `WeakHashMap`, `IdentityHashMap` | LRU cache in 20 lines |
| Sat | `Queue`, `Deque`, `PriorityQueue`, `ArrayDeque`; `Comparable` vs `Comparator` | Task scheduler using PriorityQueue |
| Sun | **Revision + Q&A** |

## Week 4 — Multithreading & Concurrency Basics

| Day | Topic | Deliverable |
|-----|-------|-------------|
| Mon | Thread lifecycle, `Thread` vs `Runnable` vs `Callable`, daemon threads | Thread state tracer |
| Tue | `synchronized` (method/block/class), intrinsic locks, reentrancy | Thread-safe counter 3 ways |
| Wed | `volatile`, Java Memory Model, happens-before, visibility vs atomicity | Debug a visibility bug |
| Thu | `wait`/`notify`/`notifyAll`, spurious wakeups, producer-consumer problem | Bounded buffer from scratch |
| Fri | `ReentrantLock`, `ReadWriteLock`, `StampedLock`, fairness, `tryLock` | Reader-writer demo |
| Sat | `ThreadLocal`, `InheritableThreadLocal`, thread interruption, `Thread.stop` dangers | ThreadLocal for request context |
| Sun | **Revision + Q&A** |

## Week 5 — Advanced Concurrency & `java.util.concurrent`

| Day | Topic | Deliverable |
|-----|-------|-------------|
| Mon | Executor framework: `Executor`, `ExecutorService`, `ThreadPoolExecutor` parameters | Tunable pool + rejection handler |
| Tue | `Callable`/`Future`, `CompletableFuture` composition (`thenApply`, `thenCompose`, `allOf`) | Orchestrate 3 async calls |
| Wed | `ForkJoinPool`, work-stealing, `RecursiveTask`, parallel streams gotchas | Parallel merge sort |
| Thu | Concurrent collections: `ConcurrentHashMap`, `CopyOnWriteArrayList`, `BlockingQueue` family | Pipeline using BlockingQueue |
| Fri | Atomic classes, CAS semantics, `AtomicInteger`, `LongAdder`, `AtomicReference` | Lock-free stack |
| Sat | Synchronizers: `CountDownLatch`, `CyclicBarrier`, `Semaphore`, `Phaser`, `Exchanger` | Race-start simulation |
| Sun | **Revision + Q&A** |

## Week 6 — JVM Internals & Memory

| Day | Topic | Deliverable |
|-----|-------|-------------|
| Mon | JVM architecture, class loaders (Bootstrap/Platform/App), parent delegation | Custom classloader loading from bytes |
| Tue | Runtime data areas: Stack, Heap, Metaspace, PC register, native stack | Diagram + object footprint calc |
| Wed | Bytecode basics (`javap -c`), constant pool, method area | Decompile a simple class |
| Thu | Garbage collection: generational hypothesis, Minor/Major/Full GC (distinct!) | GC log analysis |
| Fri | GC algorithms: Serial, Parallel, CMS (deprecated), G1, ZGC, Shenandoah | Compare GC flags on a workload |
| Sat | Memory leaks, heap dumps, tools: jstack, jmap, JFR, VisualVM, MAT | Find a leak in sample code |
| Sun | **Revision + Q&A** |

## Week 7 — Java 8+ Features & Design Principles

| Day | Topic | Deliverable |
|-----|-------|-------------|
| Mon | Lambdas, functional interfaces, method references, variable capture | Rewrite 5 anon classes as lambdas |
| Tue | Streams API: sources, intermediate/terminal ops, lazy evaluation | Analytics over 1M records |
| Wed | Collectors: `toMap`, `groupingBy`, `partitioningBy`, custom collector | Build a frequency histogram |
| Thu | `Optional` best practices, `java.time` API (LocalDate/ZonedDateTime/Instant/Duration) | Timezone-aware scheduler |
| Fri | Java 9–21: modules, records, sealed classes, pattern matching, virtual threads | Convert a DTO to a record |
| Sat | SOLID, DRY, KISS, cohesion/coupling, composition over inheritance, immutability | Refactor a "God class" |
| Sun | **Revision + Q&A** |

---

# 🌱 Phase 2 — Spring Ecosystem (Weeks 8–10)

## Week 8 — Spring Core & IoC

| Day | Topic | Deliverable |
|-----|-------|-------------|
| Mon | IoC container, `ApplicationContext` vs `BeanFactory`, bean lifecycle | Bean lifecycle logger |
| Tue | Dependency Injection: constructor vs setter vs field, `@Autowired`, `@Qualifier`, `@Primary` | Refactor service to constructor injection |
| Wed | Bean scopes (singleton, prototype, request, session), stereotypes (`@Component`, `@Service`, `@Repository`) | Prototype vs singleton demo |
| Thu | Configuration: `@Configuration`, `@Bean`, `@Import`, `@PropertySource`, profiles | Multi-profile config |
| Fri | AOP: aspects, pointcuts, advice types, JDK vs CGLIB proxies, `@Aspect` | Logging aspect for service methods |
| Sat | Events (`ApplicationEvent`, `@EventListener`), SpEL, `@ConditionalOn*` | Audit event listener |
| Sun | **Revision + Q&A** |

## Week 9 — Spring Boot

| Day | Topic | Deliverable |
|-----|-------|-------------|
| Mon | Spring Boot fundamentals, auto-configuration, `@SpringBootApplication`, starters | Explore `spring.factories` |
| Tue | Configuration: `application.yml`, profiles, `@ConfigurationProperties`, validation | Typed config class with validation |
| Wed | REST controllers: `@RestController`, `@RequestMapping`, content negotiation | CRUD REST API |
| Thu | Exception handling: `@ControllerAdvice`, `@ExceptionHandler`, problem details (RFC 7807) | Global error handler |
| Fri | Validation: Bean Validation, `@Valid`, custom `ConstraintValidator` | Custom validator for email domain |
| Sat | Actuator, metrics (Micrometer), health checks, info contributors | Add Prometheus endpoint |
| Sun | **Revision + Q&A** |

## Week 10 — Spring Data JPA & Hibernate

| Day | Topic | Deliverable |
|-----|-------|-------------|
| Mon | JPA essentials, `EntityManager`, persistence context, entity states (new/managed/detached/removed) | Entity lifecycle tracer |
| Tue | Relationships: `@OneToOne`, `@OneToMany`, `@ManyToMany`, cascade, fetch types | Order-Customer-Item schema |
| Wed | JPQL, Criteria API, native queries, `@Query`, projections, pagination | Complex search query |
| Thu | Spring Data repositories, method derivation, specifications, `@EntityGraph` | Dynamic search with Specifications |
| Fri | Transactions: `@Transactional`, propagation (REQUIRED/REQUIRES_NEW), isolation levels | Nested tx demo with rollback |
| Sat | First/second-level cache, N+1 problem, fetch strategies, `BatchSize`, lazy loading | Fix N+1 in 3 different ways |
| Sun | **Revision + Q&A** |

---

# 💾 Phase 3 — Data + APIs (Weeks 11–12)

## Week 11 — Databases Deep Dive

| Day | Topic | Deliverable |
|-----|-------|-------------|
| Mon | SQL advanced: joins, subqueries, CTEs, window functions, ranking | 5 analytical queries |
| Tue | Indexes: B-tree, hash, covering, composite; when indexes hurt | EXPLAIN plan analysis |
| Wed | ACID, isolation levels (Read Uncommitted → Serializable), phantom/dirty/non-repeatable reads | Reproduce each anomaly |
| Thu | Locking: optimistic vs pessimistic, row/table locks, deadlock detection | Simulate a deadlock |
| Fri | NoSQL overview: document, key-value, column-family, graph; CAP theorem | Choose NoSQL for 3 scenarios |
| Sat | Redis (data structures, persistence, pub/sub) + MongoDB basics (aggregation, indexing) | Caching + doc-store demo |
| Sun | **Revision + Q&A** |

## Week 12 — REST API Design

| Day | Topic | Deliverable |
|-----|-------|-------------|
| Mon | REST principles: stateless, resources, URIs, HATEOAS, Richardson Maturity | Score 3 APIs on maturity |
| Tue | HTTP methods, status codes, idempotency, safety, caching headers | Cheat sheet + examples |
| Wed | Pagination (offset vs cursor), filtering, sorting, field selection, API versioning | Paginated endpoint 2 ways |
| Thu | OpenAPI/Swagger, `springdoc-openapi`, API-first workflow | Generated docs for CRUD API |
| Fri | Rate limiting, throttling, idempotency keys, retries with backoff | Bucket4j limiter |
| Sat | GraphQL vs REST, gRPC basics, WebSockets, Server-Sent Events | REST vs GraphQL trade-off write-up |
| Sun | **Revision + Q&A** |

---

# 🔒 Phase 4 — Security & Microservices (Weeks 13–15)

## Week 13 — Spring Security

| Day | Topic | Deliverable |
|-----|-------|-------------|
| Mon | Security architecture, filter chain, `SecurityContext`, `Authentication` object | Filter chain diagram |
| Tue | Authentication: Basic, Form, JWT, OAuth2 Resource Server | JWT-secured API |
| Wed | Authorization: role-based, method security (`@PreAuthorize`, `@PostAuthorize`) | Fine-grained permission demo |
| Thu | Password encoding (BCrypt, Argon2), credential storage, brute-force protection | Secure user registration flow |
| Fri | CSRF, CORS, clickjacking, security headers | Correct CORS for SPA |
| Sat | OAuth 2.0 flows (Auth Code + PKCE, Client Creds), OIDC, SAML intro | Demo auth-code with Keycloak |
| Sun | **Revision + Q&A** |

## Week 14 — Microservices & Spring Cloud

| Day | Topic | Deliverable |
|-----|-------|-------------|
| Mon | Microservices principles, 12-factor app, bounded contexts (DDD basics) | Decompose a monolith on paper |
| Tue | Service discovery: Eureka/Consul, client-side load balancing | Register-and-discover demo |
| Wed | API Gateway (Spring Cloud Gateway), routing, filters, rewriting | Gateway with auth filter |
| Thu | Resilience: Circuit Breaker, Retry, Bulkhead, Rate Limiter (Resilience4j) | Fallback + circuit break demo |
| Fri | Distributed config (Spring Cloud Config), secret management (Vault) | Centralized config service |
| Sat | Distributed tracing (Micrometer Tracing, Zipkin), correlation IDs | Trace a request across 3 services |
| Sun | **Revision + Q&A** |

## Week 15 — Messaging & Event-Driven

| Day | Topic | Deliverable |
|-----|-------|-------------|
| Mon | Kafka fundamentals: broker, topic, partition, offset, replication | Local Kafka + produce/consume |
| Tue | Kafka advanced: consumer groups, rebalancing, exactly-once semantics, idempotent producer | Transactional producer demo |
| Wed | RabbitMQ: exchanges (direct, topic, fanout), queues, bindings, DLQ | DLQ + retry strategy |
| Thu | Event-driven patterns: event sourcing, CQRS, outbox pattern | Outbox for reliable publish |
| Fri | Distributed transactions: 2PC (bad), Saga pattern (orchestration vs choreography) | Order saga diagram + code |
| Sat | Spring integration: `spring-kafka`, `spring-amqp`, Spring Cloud Stream | Producer/consumer with Spring |
| Sun | **Revision + Q&A** |

---

# 🧪 Phase 5 — Testing & DevOps (Weeks 16–17)

## Week 16 — Testing

| Day | Topic | Deliverable |
|-----|-------|-------------|
| Mon | JUnit 5: lifecycle, assertions, assumptions, parameterized tests, dynamic tests | 20 parameterized cases |
| Tue | Mockito: mocks, spies, stubs, argument captors, verification modes | Mock a 3-level service |
| Wed | Spring Boot Test: `@SpringBootTest`, slice tests (`@WebMvcTest`, `@DataJpaTest`) | Slice test each layer |
| Thu | Integration testing with Testcontainers (Postgres, Kafka, Redis) | Full stack test with Testcontainers |
| Fri | Contract testing (Spring Cloud Contract / Pact), WireMock for external APIs | Consumer-driven contract |
| Sat | Mutation testing (PIT), coverage (JaCoCo), performance testing (JMeter basics) | PIT report on a module |
| Sun | **Revision + Q&A** |

## Week 17 — DevOps Essentials

| Day | Topic | Deliverable |
|-----|-------|-------------|
| Mon | Git advanced: rebase vs merge, cherry-pick, interactive rebase, conflict resolution | Clean up a messy branch |
| Tue | Maven / Gradle: dependency scopes, multi-module builds, BOMs, plugins | Multi-module Spring Boot project |
| Wed | Docker: Dockerfile, multi-stage builds, `docker-compose`, layer caching | Containerize Spring Boot app |
| Thu | Kubernetes: pods, deployments, services, ingress, ConfigMap/Secret, probes | Deploy to kind/minikube |
| Fri | CI/CD: GitHub Actions or Jenkins pipeline (build → test → image → deploy) | End-to-end pipeline |
| Sat | Observability: logging (structured, ELK), metrics (Prometheus + Grafana), alerts | Dashboard with 4 golden signals |
| Sun | **Revision + Q&A** |

---

# 🏛️ Phase 6 — System Design & Interview Prep (Weeks 18–20)

## Week 18 — System Design Fundamentals

| Day | Topic | Deliverable |
|-----|-------|-------------|
| Mon | Scalability: vertical vs horizontal, stateless design, load balancing (L4/L7) | Sketch a scalable web tier |
| Tue | Caching strategies: cache-aside, write-through, write-back, TTL, eviction policies | Compare Redis caching modes |
| Wed | Database scaling: read replicas, sharding, partitioning, CAP/PACELC | Sharding key design exercise |
| Thu | Async processing, message queues, pub/sub, event streaming trade-offs | When to choose Kafka vs RabbitMQ |
| Fri | CDN, DNS, reverse proxy, rate limiting, API gateway patterns | High-level architecture for news site |
| Sat | Consistent hashing, bloom filters, quorum reads/writes, leader election | Implement consistent hashing |
| Sun | **Revision + Q&A** |

## Week 19 — System Design Practice

Focus: 1 full design per day — requirements → estimations → high-level → components → deep-dives → trade-offs.

| Day | Design Problem |
|-----|---------------|
| Mon | URL shortener (TinyURL) |
| Tue | News feed (Twitter/Facebook timeline) |
| Wed | Chat application (WhatsApp/Slack) |
| Thu | Video streaming (YouTube/Netflix) |
| Fri | Ride-sharing matching (Uber) |
| Sat | Distributed cache + payment processing system |
| Sun | **Revision** — compile your personal design template + cheat sheet |

## Week 20 — DSA, Behavioral, Mocks

| Day | Topic | Deliverable |
|-----|-------|-------------|
| Mon | Arrays, strings, two pointers, sliding window (top 15 LeetCode) | Solve 5, note patterns |
| Tue | Linked lists, stacks, queues (reverse, cycle detect, min-stack) | Solve 5 |
| Wed | Trees (BST, traversals, LCA), heaps (top-K, merge-K) | Solve 5 |
| Thu | Hash maps, graphs (BFS/DFS, topological sort, Dijkstra intro) | Solve 5 |
| Fri | Dynamic programming essentials (1D/2D, memoization vs tabulation) | Solve 3 classics |
| Sat | Behavioral: STAR method, leadership, conflict, ambiguity; Q&A for the interviewer | Write out 10 STAR stories |
| Sun | **Final mock interview** — full loop: coding + design + behavioral |

---

## 📌 Study Rules

1. **2 hours daily, non-negotiable.** Block the slot on your calendar.
2. **60 minutes reading/watching · 60 minutes coding.** No passive learning.
3. **Push every exercise to GitHub** — builds your public portfolio.
4. **Sunday is sacred:** no new topics, only revision + writing Q&A for the week.
5. **Weekly retrospective (Sunday night):** what stuck, what didn't, adjust next week.

## 📚 Recommended Resources

- **Books:** *Effective Java* (Bloch), *Java Concurrency in Practice* (Goetz), *Designing Data-Intensive Applications* (Kleppmann), *Clean Architecture* (Martin), *System Design Interview Vol. 1 & 2* (Alex Xu).
- **Courses:** Baeldung (Spring), Vlad Mihalcea's blog (JPA/Hibernate), "Grokking the System Design Interview."
- **Practice:** LeetCode (Top Interview 150), HackerRank (Java track), excalidraw for design diagrams.
- **Watch:** Devoxx/JavaOne talks on JVM, GC, virtual threads; Martin Fowler on microservices.

## ✅ Progress Tracking

- Maintain a `progress.md` in this folder — tick off each day as you complete it.
- End of each phase → write a summary doc of what you learned + gaps.
- End of week 10, 15, 20 → do a full mock interview (friend, paid service, or record yourself).

---

**Start date:** _____________
**Target end date:** _____________ (~20 weeks from start)

*"Slow is smooth, smooth is fast. Consistency beats intensity."*
