# Week 7 — Java 8+ Features & Design Principles

**Theme:** Modern Java (8+) changed the language more than any release since Java 5 generics. This week makes you fluent in functional-style Java and grounds you in the design principles senior interviews probe. You'll finish Phase 1 ready to move into frameworks.

---

## Monday — Lambdas, Functional Interfaces, Method References

### 🎯 Objective
Write idiomatic lambdas; know the built-in functional interfaces; explain variable capture rules.

### 📖 Core Concept

**Lambda** — an anonymous function. Target type: any functional interface.

**Syntax:**
```java
()         -> expression
x          -> x * 2
(x, y)     -> x + y
(int x)    -> x + 1        // with explicit type
(x, y)     -> { return x + y; }   // block body
```

**Functional interface** — interface with exactly ONE abstract method. Optionally annotated `@FunctionalInterface` (compiler-enforced).

**Variable capture:**
- Lambdas capture **effectively final** local variables (assigned once, never reassigned).
- Lambdas capture instance and class fields freely (these aren't "local").
- `this` inside a lambda refers to the **enclosing class** (unlike anonymous inner classes, where `this` refers to the anonymous instance itself).

**Under the hood:** Lambdas are NOT compiled to inner classes. They use `invokedynamic` + `LambdaMetafactory` to lazily create lightweight implementation objects. More efficient than the pre-Java 8 anonymous-class idiom.

**Built-in functional interfaces (`java.util.function`):**

| Interface | Signature | Typical use |
|-----------|-----------|-------------|
| `Function<T, R>` | `R apply(T)` | Transform |
| `Predicate<T>` | `boolean test(T)` | Filter condition |
| `Consumer<T>` | `void accept(T)` | Side effect |
| `Supplier<T>` | `T get()` | Factory/lazy value |
| `BiFunction<T, U, R>` | `R apply(T, U)` | Two-arg transform |
| `BinaryOperator<T>` | `T apply(T, T)` | Reduce/combine |
| `UnaryOperator<T>` | `T apply(T)` | Function<T,T> |

**Primitive specializations** avoid autoboxing: `IntFunction<R>`, `IntPredicate`, `IntToLongFunction`, `ToIntFunction<T>`, etc.

**Method references** — shorthand for lambdas that just call one method.

| Form | Example | Equivalent lambda |
|------|---------|-------------------|
| Static | `Integer::parseInt` | `s -> Integer.parseInt(s)` |
| Bound instance | `"hi"::concat` | `s -> "hi".concat(s)` |
| Unbound instance | `String::length` | `s -> s.length()` |
| Constructor | `ArrayList::new` | `() -> new ArrayList<>()` |

### 💻 Code Example

```java
// Traditional
button.addActionListener(new ActionListener() {
    public void actionPerformed(ActionEvent e) { System.out.println("clicked"); }
});

// Lambda
button.addActionListener(e -> System.out.println("clicked"));

// Method reference
button.addActionListener(System.out::println);

// Function composition
Function<String, String> trim = String::trim;
Function<String, String> upper = String::toUpperCase;
Function<String, String> pipeline = trim.andThen(upper);    // trim → upper
Function<String, Integer> length = pipeline.andThen(String::length);

// Predicate composition
Predicate<String> notEmpty = s -> !s.isEmpty();
Predicate<String> shortEnough = s -> s.length() < 20;
Predicate<String> valid = notEmpty.and(shortEnough);
```

### ⚠️ Common Pitfalls
- Trying to reassign a captured local variable — compile error (must be effectively final).
- Assuming `this` inside a lambda refers to the lambda — it refers to the enclosing class.
- Overusing method references when inline logic is clearer (`list.forEach(x -> log.info("item: {}", x))` vs `list.forEach(log::info)` — the first is clearer).

### 🎤 Interview Angle
> "What's the difference between a lambda and an anonymous inner class?"

- Lambdas require a functional interface; anon classes can implement any interface or extend any class.
- Lambdas are compiled to `invokedynamic` (lighter); anon classes to separate .class files.
- Lambda `this` → enclosing class; anon class `this` → the anon instance.
- Lambdas cannot have state (no fields); anon classes can.

### 🧪 Practice
1. Rewrite 5 anonymous inner classes from a Swing app as lambdas or method references.
2. Compose 3 `Predicate`s using `and`/`or`/`negate`.
3. Write a generic `Function<T, R>` pipeline that trims, uppercases, and counts characters of a string.

### 🔍 Self-check
1. Why must captured locals be effectively final?
2. What does `@FunctionalInterface` do?
3. What's the difference between `Function.andThen` and `Function.compose`?
4. When would you prefer a lambda over a method reference?

---

## Tuesday — Streams API

### 🎯 Objective
Build stream pipelines fluently; know intermediate vs terminal ops; understand laziness and short-circuiting.

### 📖 Core Concept

**Stream** — a sequence of elements supporting aggregate operations. NOT a data structure. Streams **do not store data**; they describe a computation.

**Pipeline shape:**
```
Source → Intermediate ops (lazy) → Terminal op (triggers execution)
```

**Source examples:**
```java
list.stream()
Arrays.stream(arr)
Stream.of("a", "b", "c")
Stream.iterate(1, i -> i * 2).limit(10)
Stream.generate(Math::random).limit(5)
IntStream.range(0, 10)
Files.lines(Path.of("file.txt"))
```

**Intermediate ops (return Stream, lazy):**

| Op | Stateful? | Short-circuit? |
|----|-----------|----------------|
| `filter(Predicate)` | No | No |
| `map(Function)` | No | No |
| `flatMap(Function)` | No | No |
| `distinct()` | Yes (buffer) | No |
| `sorted()` | Yes (buffer) | No |
| `peek(Consumer)` | No | No |
| `limit(n)` | Yes | Yes |
| `skip(n)` | Yes | No |
| `takeWhile(Predicate)` (Java 9+) | No | Yes |
| `dropWhile(Predicate)` (Java 9+) | Yes | No |

**Terminal ops (trigger pipeline):**

| Op | Result |
|----|--------|
| `collect(Collector)` | Collection / Map |
| `forEach(Consumer)` | void |
| `forEachOrdered(Consumer)` | void, order-preserving |
| `reduce(...)` | Single value |
| `count()` | long |
| `min(Comparator)` / `max(Comparator)` | Optional<T> |
| `findFirst()` / `findAny()` | Optional<T>, short-circuit |
| `anyMatch` / `allMatch` / `noneMatch` | boolean, short-circuit |
| `toArray()` | array |
| `toList()` (Java 16+) | immutable List |

**Laziness** — no work happens until the terminal op runs. Filter → map → limit → findFirst might process only a handful of elements even on a million-element source.

**Short-circuiting** — ops like `findFirst`, `anyMatch`, `limit` can stop processing early. Essential for infinite streams.

**Parallel streams** — `.parallelStream()` uses the common ForkJoinPool. Helps for large, CPU-bound, stateless operations. Hurts for small streams, IO-bound work, stateful/ordered operations.

### 💻 Code Example

```java
// Group employees by department, average salary per dept
Map<String, Double> avgByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.averagingDouble(Employee::getSalary)));

// Top 5 earners
List<Employee> top5 = employees.stream()
    .sorted(Comparator.comparingDouble(Employee::getSalary).reversed())
    .limit(5)
    .toList();

// FlatMap — flatten lists of tags
List<String> allTags = posts.stream()
    .flatMap(p -> p.getTags().stream())
    .distinct()
    .sorted()
    .toList();

// Reduce — total salary
double total = employees.stream()
    .mapToDouble(Employee::getSalary)
    .sum();    // using primitive specialization (no boxing)

// Infinite stream + short-circuit
int firstSquareOver1000 = Stream.iterate(1, i -> i + 1)
    .map(i -> i * i)
    .filter(n -> n > 1000)
    .findFirst()
    .orElseThrow();
```

### ⚠️ Common Pitfalls
- Calling a terminal op twice on the same stream — `IllegalStateException` (streams are single-use).
- Mutating shared state inside `forEach` — race conditions in parallel streams.
- Using `parallelStream()` for small collections — overhead dominates.
- Performance hit from unnecessary boxing: `Stream<Integer>` is much slower than `IntStream`.

### 🎤 Interview Angle
> "Why is `Stream` lazy?"

To enable optimizations like short-circuiting (`findFirst` stops at first match) and fusion (consecutive `filter`/`map` ops are performed in a single pass without intermediate collections). Laziness means a 1M-element stream pipeline might process only 10 elements if the terminal op finds what it needs.

### 🧪 Practice
1. Write a pipeline that groups `Transaction` by `year`, then by `month`, summing amounts.
2. Process a 100MB text file with `Files.lines` and extract top-10 most common words.
3. Build an `IntStream`-based primality test and find the first 50 primes.

### 🔍 Self-check
1. What's the difference between a stream and a collection?
2. Name 3 stateful intermediate ops.
3. What does `flatMap` do?
4. Why is `IntStream` faster than `Stream<Integer>`?

---

## Wednesday — Collectors, Grouping, Partitioning

### 🎯 Objective
Use the Collectors toolbox to produce complex aggregations; write a custom collector.

### 📖 Core Concept

**`Collectors.toList()`**, **`toSet()`**, **`toMap(keyFn, valueFn)`**, **`toUnmodifiableList()`** (Java 10+).

**Java 16+ shortcut:** `stream.toList()` — returns an **immutable** list.

**Aggregation:**
```java
Collectors.counting()                      // long
Collectors.summingInt(Employee::getAge)    // int
Collectors.averagingDouble(Employee::getSalary)
Collectors.summarizingInt(Employee::getAge) // IntSummaryStatistics (count, sum, avg, min, max)
Collectors.minBy(comparator) / maxBy(comparator)
Collectors.reducing(identity, binaryOp)
```

**Grouping:**
```java
Map<Dept, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDept));

// Grouping with downstream collector
Map<Dept, Double> avgSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDept,
        Collectors.averagingDouble(Employee::getSalary)));

// Nested grouping
Map<Dept, Map<String, List<Employee>>> byDeptAndRole = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDept,
        Collectors.groupingBy(Employee::getRole)));
```

**Partitioning** — grouping into exactly two buckets by a predicate:
```java
Map<Boolean, List<Employee>> highLow = employees.stream()
    .collect(Collectors.partitioningBy(e -> e.getSalary() > 100_000));
```

**joining:**
```java
String names = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.joining(", ", "[", "]"));
// "[Alice, Bob, Carol]"
```

**mapping (downstream):**
```java
Map<Dept, List<String>> namesByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDept,
        Collectors.mapping(Employee::getName, Collectors.toList())));
```

**Custom collector** — usually done via `Collector.of(supplier, accumulator, combiner, [finisher])`.

### 💻 Code Example

```java
// toMap with merge function to handle duplicate keys
Map<String, List<Employee>> byName = employees.stream()
    .collect(Collectors.toMap(
        Employee::getName,
        e -> List.of(e),
        (list1, list2) -> {
            List<Employee> merged = new ArrayList<>(list1);
            merged.addAll(list2);
            return merged;
        }));

// Multi-level statistics
Map<String, DoubleSummaryStatistics> statsByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDept,
        Collectors.summarizingDouble(Employee::getSalary)));

// Custom collector — accumulate into a comma-separated string (like joining)
Collector<String, StringBuilder, String> joinCollector = Collector.of(
    StringBuilder::new,
    (sb, s) -> sb.append(s).append(','),
    (sb1, sb2) -> sb1.append(sb2),
    sb -> sb.length() == 0 ? "" : sb.substring(0, sb.length() - 1)
);
```

### ⚠️ Common Pitfalls
- `Collectors.toMap` with duplicate keys and no merge function → `IllegalStateException`. Always supply a merge function when duplicates are possible.
- Nested `groupingBy` with huge cardinality can explode memory.
- Mutable accumulators in a custom collector must be thread-safe if used with parallel streams, OR marked with `Collector.Characteristics.CONCURRENT`.

### 🎤 Interview Angle
> "What's the difference between `groupingBy` and `partitioningBy`?"

`groupingBy` — arbitrary number of groups (one per classifier value).
`partitioningBy` — exactly two groups (predicate true/false). Faster and the returned Map always has both keys.

### 🧪 Practice
1. Group employees by department, compute average salary AND count per department.
2. Partition orders into "shipped" vs "pending," then format as two comma-separated lists.
3. Write a custom collector that computes median of a stream.

### 🔍 Self-check
1. What's the difference between `toList()` and `Collectors.toList()`?
2. How do you handle duplicate keys in `Collectors.toMap`?
3. What does the "downstream collector" in `groupingBy` do?
4. When would you write a custom collector?

---

## Thursday — Optional, java.time API

### 🎯 Objective
Use `Optional` idiomatically (not as a null replacement); use `java.time` correctly with time zones.

### 📖 Core Concept — Optional

`Optional<T>` — container that may or may not hold a non-null value. Makes nullability **explicit in the API**.

**Creation:**
```java
Optional.of(value)             // NPE if value is null
Optional.ofNullable(value)     // empty if null
Optional.empty()
```

**Inspection:**
```java
opt.isPresent()                // boolean
opt.isEmpty()                  // Java 11+
opt.ifPresent(action)
opt.ifPresentOrElse(action, emptyAction)   // Java 9+
```

**Retrieval:**
```java
opt.orElse(default)              // always evaluates default
opt.orElseGet(() -> compute())   // lazy default (preferred for expensive defaults)
opt.orElseThrow()                // throws NoSuchElementException (Java 10+)
opt.orElseThrow(ExceptionSupplier)
```

**Transformation:**
```java
opt.map(String::toUpperCase)
opt.flatMap(s -> Optional.of(s.trim()))   // avoid nested Optional
opt.filter(s -> s.length() > 3)
```

**Best practices (Stuart Marks, JDK architect):**
- ✅ Use as **return type** where absence is a valid outcome.
- ❌ Don't use as field — adds indirection, doesn't serialize well.
- ❌ Don't use as method parameter — callers already know whether they have a value.
- ❌ Don't put inside collections — use empty collections instead.
- ❌ Never call `.get()` without checking presence — defeats the purpose.
- ❌ Never use `Optional.ofNullable(x).orElse(default)` when `x != null ? x : default` would do.

### 📖 Core Concept — java.time

JSR-310 (Java 8) replaced the broken `Date`/`Calendar` APIs. All types are **immutable** and thread-safe.

| Class | Represents | Example |
|-------|-----------|---------|
| `LocalDate` | date, no time, no zone | 2026-04-24 |
| `LocalTime` | time, no date, no zone | 14:30:15 |
| `LocalDateTime` | date + time, no zone | 2026-04-24T14:30 |
| `ZonedDateTime` | date + time + zone | 2026-04-24T14:30+05:30 Asia/Kolkata |
| `OffsetDateTime` | date + time + offset (no region) | 2026-04-24T14:30+05:30 |
| `Instant` | nanoseconds since UTC epoch | (machine timestamp) |
| `Duration` | time-based amount | PT2H30M |
| `Period` | date-based amount | P1Y2M3D |

**Common operations:**
```java
LocalDate today = LocalDate.now();
LocalDate birthday = LocalDate.of(1990, 6, 15);
LocalDate tomorrow = today.plusDays(1);
Period age = Period.between(birthday, today);

ZonedDateTime kolkata = ZonedDateTime.now(ZoneId.of("Asia/Kolkata"));
ZonedDateTime ny = kolkata.withZoneSameInstant(ZoneId.of("America/New_York"));

Instant now = Instant.now();
Duration elapsed = Duration.between(start, end);

// Formatting
String s = today.format(DateTimeFormatter.ofPattern("dd/MM/yyyy"));
LocalDate parsed = LocalDate.parse("24/04/2026", DateTimeFormatter.ofPattern("dd/MM/yyyy"));
```

**Best practices:**
- Store timestamps as `Instant` / UTC in databases.
- Convert to user's `ZoneId` only for display.
- Never mix legacy `Date` / `Calendar` with `java.time` beyond interop boundaries.
- `SimpleDateFormat` (legacy) is NOT thread-safe; `DateTimeFormatter` IS.

### 💻 Code Example

```java
// Optional — fluent chain
public Optional<String> findUserEmail(String userId) {
    return userRepo.findById(userId)
        .map(User::getEmail)
        .filter(e -> !e.isBlank());
}

// Caller:
String email = findUserEmail("u1").orElseThrow(() -> new NoSuchUserException("u1"));

// Time zones
ZonedDateTime bookingKolkata = ZonedDateTime.of(
    LocalDate.of(2026, 5, 1), LocalTime.of(10, 0),
    ZoneId.of("Asia/Kolkata"));
Instant utcTimestamp = bookingKolkata.toInstant();   // store this
```

### ⚠️ Common Pitfalls
- `opt.isPresent() { opt.get() }` — same pattern as null-check; `ifPresent` or `map` is cleaner.
- Using `orElse(expensiveCall())` instead of `orElseGet(() -> expensiveCall())` — `orElse` always evaluates.
- Using `LocalDateTime` when you actually need a specific time zone — no TZ info, ambiguous.
- Hardcoding time zone as "server local" — breaks when deployed to a different region.

### 🎤 Interview Angle
> "What's the difference between `Optional.orElse` and `orElseGet`?"

`orElse(x)` ALWAYS evaluates `x`, even if Optional is non-empty. `orElseGet(supplier)` only calls the supplier when empty. Use `orElseGet` for any non-trivial default.

> "Why was `java.util.Date` replaced?"

Mutable (not thread-safe), months were 0-indexed (January = 0), poor timezone support, mixed "date" and "instant" concepts, and `SimpleDateFormat` wasn't thread-safe.

### 🧪 Practice
1. Refactor a null-returning API to return `Optional<T>`; update 3 callers.
2. Compute age from a birthday using `Period.between`.
3. Write a function that takes a `ZonedDateTime` and converts to any number of target time zones.

### 🔍 Self-check
1. When should an API return `Optional` vs null?
2. Why is `java.time` thread-safe while `java.util.Date` isn't?
3. Difference between `Instant` and `LocalDateTime`?
4. When would you use `Period` vs `Duration`?

---

## Friday — Java 9–21: Modules, Records, Sealed Classes, Pattern Matching, Virtual Threads

### 🎯 Objective
Recognize modern language features and know when to use them.

### 📖 Core Concept

**Records (Java 16+)** — immutable data carriers. Auto-generates constructor, accessors, `equals`/`hashCode`/`toString`.
```java
public record Point(int x, int y) { }
// Equivalent to 40+ lines of a proper immutable class.

// Compact canonical constructor for validation
public record PositiveRange(int lo, int hi) {
    public PositiveRange {
        if (lo < 0 || hi < lo) throw new IllegalArgumentException();
    }
}
```
Records CAN implement interfaces, have additional methods, and static factories. They CANNOT extend classes (implicitly extend `Record`).

**Sealed classes (Java 17+)** — restrict which classes can extend/implement.
```java
public sealed interface Shape permits Circle, Square, Triangle { }
public final class Circle implements Shape { ... }
public final class Square implements Shape { ... }
public non-sealed class Triangle implements Shape { ... }  // opens extension
```
Enables exhaustive pattern matching (compiler knows all possibilities).

**Pattern matching for `instanceof` (Java 16+):**
```java
if (obj instanceof String s && s.length() > 5) {
    System.out.println(s.toUpperCase());   // s already typed as String
}
```

**Pattern matching in `switch` (Java 21):**
```java
String describe(Shape s) {
    return switch (s) {
        case Circle c    -> "circle r=" + c.radius();
        case Square sq   -> "square side=" + sq.side();
        case Triangle t  -> "triangle";
    };   // exhaustive (sealed Shape); no default needed
}
```

**Text blocks (Java 15+):**
```java
String json = """
        {
          "name": "Alice"
        }
        """;
```

**Modules (Java 9+, JPMS)** — strong encapsulation above the package level. A module declares what packages it `exports` and what modules it `requires`.
```
module com.example.app {
    requires java.sql;
    exports com.example.app.api;
}
```
Most applications don't use modules directly (Spring Boot, Maven projects typically ignore JPMS). Library authors might.

**Virtual threads (Java 21+)** — lightweight threads. Created per task, not backed 1:1 by an OS thread. Millions concurrent. Game-changer for IO-heavy workloads.
```java
try (ExecutorService ex = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 1_000_000).forEach(i -> ex.submit(() -> httpCall()));
}
```

**Other notable additions:**
- `var` keyword (Java 10) — local variable type inference.
- Enhanced `switch` expressions (Java 14).
- Helpful NullPointerException messages (Java 14).
- Unnamed variables and patterns `_` (Java 21).

### 💻 Code Example

```java
// Combining records + sealed + pattern matching
public sealed interface Event permits UserRegistered, OrderPlaced, PaymentFailed { }
public record UserRegistered(String userId, Instant at) implements Event { }
public record OrderPlaced(String orderId, BigDecimal amount) implements Event { }
public record PaymentFailed(String orderId, String reason) implements Event { }

public String describe(Event e) {
    return switch (e) {
        case UserRegistered(var userId, var at) -> "User " + userId + " at " + at;
        case OrderPlaced(var id, var amt)       -> "Order " + id + " for " + amt;
        case PaymentFailed(var id, var reason)  -> "Failed " + id + ": " + reason;
    };
}
```

### ⚠️ Common Pitfalls
- Using records when you need mutability or inheritance — they don't fit.
- Overusing `var` — readability suffers when the type isn't obvious.
- Assuming virtual threads are faster for CPU-bound work — they're not; they help IO-bound scenarios.

### 🎤 Interview Angle
> "What problem do virtual threads solve?"

Thread-per-request web servers used to be limited to a few thousand concurrent requests because each OS thread costs ~1MB of stack + context-switch overhead. Virtual threads are mounted on a small carrier thread pool; when blocked on IO, they unmount, freeing the carrier for another VT. Millions of in-flight requests become feasible with simple blocking code — no reactive framework needed.

### 🧪 Practice
1. Refactor a DTO class into a record — observe the line count.
2. Design a small state machine using sealed interfaces + pattern matching.
3. Benchmark a 10K-request HTTP client using platform threads vs virtual threads.

### 🔍 Self-check
1. Can a record have setters?
2. What's the purpose of `sealed` classes?
3. What's the difference between platform threads and virtual threads?
4. When is `var` a bad idea?

---

## Saturday — SOLID, DRY, KISS, YAGNI, Composition over Inheritance

### 🎯 Objective
Recognize violations in code; refactor toward clean design.

### 📖 Core Concept

**SOLID** — object-oriented design principles.

**S — Single Responsibility** — a class should have one reason to change.
```java
// BAD — User handles DB, email, validation
class User {
    void save();
    void sendEmail();
    boolean isValid();
}
// GOOD
class User { /* state only */ }
class UserRepository { void save(User u); }
class EmailService { void sendTo(User u); }
class UserValidator { boolean validate(User u); }
```

**O — Open/Closed** — open for extension, closed for modification.
Use polymorphism. Adding a new `Shape` shouldn't require modifying existing `area()` code.

**L — Liskov Substitution** — subtypes must be substitutable for their base type without breaking correctness.
Classic violation: `Square extends Rectangle` where setWidth also sets height → breaks callers expecting rectangle semantics.

**I — Interface Segregation** — many narrow interfaces > one wide interface.
`Worker { work(); eat(); sleep(); }` forces `Robot` to stub `eat()`. Split into `Workable`, `Feedable`, `Sleepable`.

**D — Dependency Inversion** — depend on abstractions, not concretions.
```java
// BAD
class OrderService { MySqlOrderRepo repo = new MySqlOrderRepo(); }
// GOOD
class OrderService {
    final OrderRepo repo;
    OrderService(OrderRepo repo) { this.repo = repo; }  // inject
}
```

**DRY** — Don't Repeat Yourself. Same knowledge in one place.
**Caution:** premature DRY can hurt — rule of three, wait until the pattern is clearly repeated before abstracting.

**KISS** — Keep It Simple, Stupid. Simplest solution wins.

**YAGNI** — You Aren't Gonna Need It. Don't build for imaginary future requirements.

**Composition over inheritance** — prefer "has-a" over "is-a" where possible. Inheritance is rigid, exposes internals; composition is flexible, loosely coupled.
```java
// Inheritance (tight coupling)
class Stack<E> extends Vector<E> { ... }

// Composition (flexible)
class Stack<E> {
    private final List<E> items = new ArrayList<>();
    public void push(E e) { items.add(e); }
    public E pop() { return items.remove(items.size() - 1); }
}
```

**Cohesion vs Coupling:**
- **High cohesion** — class members are closely related (good).
- **Low coupling** — classes depend minimally on each other (good).

### 💻 Code Example — refactor a "God class"

```java
// Before — ReportGenerator does EVERYTHING
class ReportGenerator {
    void fetchData() { /* DB */ }
    void transformData() { /* calc */ }
    void formatAsPdf() { /* PDF */ }
    void formatAsExcel() { /* Excel */ }
    void emailTo(String to) { /* SMTP */ }
    void save(String path) { /* file IO */ }
}

// After — each responsibility in its own class, injected
interface DataSource { List<Record> fetch(); }
interface Formatter { byte[] format(List<Record> data); }
interface Publisher { void publish(byte[] content); }

class Report {
    private final DataSource src;
    private final Formatter fmt;
    private final Publisher pub;

    Report(DataSource s, Formatter f, Publisher p) {
        this.src = s; this.fmt = f; this.pub = p;
    }

    public void generate() {
        List<Record> data = src.fetch();
        byte[] bytes = fmt.format(data);
        pub.publish(bytes);
    }
}
// Add PDF formatter, Email publisher, S3 publisher — no Report change (OCP + DIP)
```

### ⚠️ Common Pitfalls
- Taking SOLID too literally — extracting every conditional into a strategy class ("pattern heavy") is its own smell.
- Deep inheritance hierarchies — usually composition would be cleaner.
- Excessive mocking in tests often signals high coupling — refactor the production code.

### 🎤 Interview Angle
> "Why is Liskov Substitution tricky with Rectangle/Square?"

If `Square extends Rectangle` and overrides `setWidth(w)` to also set height, callers using `Rectangle.setWidth(5); assertEquals(5, r.getWidth())` pass for Rectangle but fail for Square. The subtype is not substitutable — it narrows the contract. The fix: neither `IS-A` the other; both are Shapes.

### 🧪 Practice
1. Refactor a God class in your codebase applying SOLID. List which principle each change addresses.
2. Find a violation of LSP in a codebase (or create one). Fix it.
3. Convert an inheritance hierarchy to composition — compare before/after.

### 🔍 Self-check
1. Give an example of an Open/Closed violation and its fix.
2. What's the LSP and why does Square/Rectangle violate it?
3. Why prefer composition over inheritance?
4. When does DRY become harmful?

---

## Sunday — Revision & Q&A

### 🧭 Revision checklist
- [ ] Write lambda, method reference, and anonymous class for the same task.
- [ ] Chain a stream with filter + map + flatMap + collect.
- [ ] Use Optional correctly in a return type; refactor a null-returning method.
- [ ] Use java.time to convert between zones + compute age.
- [ ] Build a small type hierarchy with sealed interfaces + records + pattern matching.
- [ ] Refactor a code smell applying one SOLID principle.

### 📝 Interview Q&A — Final Phase 1 Session
Ask Claude: **"Prepare a Phase 1 Week 7 interview Q&A doc"** — answer, review, rate.

Then: **"Prepare a full Phase 1 cumulative Q&A doc"** — 40 questions covering all 7 weeks as the graduation test.

### 📂 Commit your work
```
phase-1/week-07/
├── exercises/
│   ├── LambdaComposition.java
│   ├── StreamPipelines.java
│   ├── CollectorsDemo.java
│   ├── OptionalCorrectUsage.java
│   ├── TimeZoneConverter.java
│   ├── EventBus.java          (records + sealed + pattern matching)
│   └── RefactoredGodClass.java
└── notes.md
```

---

## 🎓 Phase 1 Graduation

You've completed Core Java. Before moving to Phase 2 (Spring), verify:

- [ ] Can explain the JVM memory layout end-to-end.
- [ ] Can write thread-safe code using 3 different mechanisms and defend the trade-offs.
- [ ] Can explain HashMap internals including treeification.
- [ ] Can use CompletableFuture for composed async work without blocking.
- [ ] Can write a correctly immutable class.
- [ ] Can read bytecode with `javap -c` and recognize common instructions.
- [ ] Can design a small system applying SOLID without being told which principle to use.

If any of these feel shaky — **go back** before starting Spring. The framework abstractions will only hide the gaps, not fix them.
