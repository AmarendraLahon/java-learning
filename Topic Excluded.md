# Java Interview-Ready Notes — Full Detailed Descriptions

---

# 🔹 Exception Handling

## 1. Exception Hierarchy

Every exception in Java is an object. The root of the hierarchy is `java.lang.Throwable`. Only objects that are instances of `Throwable` (or its subclasses) can be thrown by the JVM or by a `throw` statement, and only they can be caught by a `catch` clause.

```
Throwable
 ├── Error           (serious, usually unrecoverable)
 │    ├── OutOfMemoryError
 │    ├── StackOverflowError
 │    └── VirtualMachineError
 └── Exception
      ├── IOException        (checked)
      ├── SQLException       (checked)
      ├── ClassNotFoundException (checked)
      └── RuntimeException   (unchecked)
           ├── NullPointerException
           ├── ArithmeticException
           ├── ArrayIndexOutOfBoundsException
           ├── ClassCastException
           └── IllegalArgumentException
```

**Why the hierarchy matters:**
- The JVM walks the call stack looking for a `catch` block whose declared type is the same class or a superclass of the thrown exception. That's why you can write `catch (Exception e)` to catch almost anything.
- Catch blocks must be arranged **most-specific-first**; if `catch (Exception e)` appears before `catch (IOException e)`, the compiler rejects the code as unreachable.
- Never catch `Throwable` or `Error` in application code — the JVM may be in an invalid state (e.g., `OutOfMemoryError` means you cannot even allocate the message string for logging).

**Interview-level insight:** exception types should model **what went wrong** (a business/technical condition), not **where** it went wrong. A well-designed hierarchy lets callers decide granularity — catching the abstract parent for generic handling or a specific child for targeted recovery.

---

## 2. Checked vs Unchecked Exceptions

**Checked exceptions** extend `Exception` (but NOT `RuntimeException`). The compiler forces you to either:
- Handle them with `try-catch`, or
- Declare them with `throws` on the method signature.

Examples: `IOException`, `SQLException`, `ClassNotFoundException`, `InterruptedException`.

**Unchecked exceptions** extend `RuntimeException`. The compiler does NOT enforce handling. They typically signal **programming errors**.

Examples: `NullPointerException`, `ArrayIndexOutOfBoundsException`, `IllegalStateException`, `ClassCastException`.

**Design philosophy (per Joshua Bloch):**
- Use **checked** exceptions for recoverable conditions the caller can reasonably be expected to handle (e.g., file missing — caller might prompt user, retry, or fallback).
- Use **unchecked** exceptions for programming errors that should be fixed in code, not handled at runtime (e.g., passing null where not allowed, index out of bounds).

**Trade-off:** checked exceptions pollute method signatures and often get swallowed or wrapped. Modern frameworks (Spring, Hibernate) lean heavily toward unchecked because excessive `throws` declarations harm maintainability and composability (especially with lambdas, which don't accept checked exceptions in most functional interfaces).

**Common interview question:** Can you override a method that throws a checked exception with one that throws no exception? **Yes** — a subclass method can throw fewer/narrower checked exceptions but not broader ones (Liskov Substitution).

---

## 3. Error vs Exception

Both extend `Throwable`, but they model fundamentally different categories:

**`Error`** represents **serious problems** that a reasonable application should NOT try to catch:
- `OutOfMemoryError` — heap/stack exhausted
- `StackOverflowError` — infinite recursion or deeply nested calls
- `NoClassDefFoundError` — required class absent at runtime
- `VirtualMachineError` — JVM internal failure

These generally indicate the JVM is broken, not your code. Catching them is usually pointless because the JVM cannot guarantee continued execution.

**`Exception`** represents **application-level problems** that programs should handle:
- File not found, database connection lost, invalid user input, etc.

**Practical difference:**
- Errors bubble up and typically crash the thread or JVM.
- Exceptions are part of normal program flow handling.

**Tricky angle:** `Error` is **unchecked** (you don't have to declare it), even though it's not a `RuntimeException`. The compile-time distinction is: checked = subclasses of `Exception` minus `RuntimeException` and its subclasses.

---

## 4. try-catch-finally

**Basic form:**
```java
try {
    // risky code
} catch (SpecificException e) {
    // handle
} catch (AnotherException e) {
    // handle
} finally {
    // cleanup — always runs
}
```

**Rules:**
1. `try` is mandatory; it must be followed by at least one `catch` OR `finally` (or both).
2. Multiple `catch` blocks are allowed but must be ordered **specific → general**.
3. At most one `catch` block runs per exception.
4. `finally` runs regardless of whether an exception was thrown or caught — even if `return` or `throw` appears inside `try` or `catch`.

**Multi-catch (Java 7+):**
```java
try { ... }
catch (IOException | SQLException e) {
    log.error("IO or DB failure", e);
}
```
- The variable `e` is **effectively final** in multi-catch — you can't reassign it.
- Reduces code duplication for common handling.

**Exception chaining:**
```java
catch (LowLevelException e) {
    throw new HighLevelException("context", e); // preserves stack trace
}
```

**Interview trap:** A `try` without `catch` is valid ONLY if followed by `finally`. This is common in try-with-resources.

---

## 5. throw vs throws

These look alike but serve different purposes.

**`throw`** is a **statement** used inside a method body to explicitly raise an exception:
```java
if (balance < amount) {
    throw new InsufficientFundsException("Balance too low");
}
```
- You can only throw a `Throwable` (or subclass).
- Control flow transfers immediately — code after `throw` is unreachable.

**`throws`** is a **declaration** in the method signature indicating which checked exceptions the method may propagate:
```java
public void readFile(String path) throws IOException, FileNotFoundException {
    ...
}
```
- Only required for checked exceptions (unchecked exceptions don't need declaration).
- Callers must handle them or declare them themselves.
- Subclass overrides can declare FEWER or NARROWER checked exceptions, never broader ones.

**Mnemonic:** `throw` = "I am throwing this one thing right now." `throws` = "This method might throw one of these types — caller beware."

---

## 6. Custom Exception

Creating your own exception types makes error handling more expressive and aligns exceptions with your domain model.

**Rules of thumb:**
- Extend `Exception` if you want callers forced to handle it (checked).
- Extend `RuntimeException` if the error is a programming bug or purely optional to handle.
- Provide the standard four constructors:

```java
public class InsufficientBalanceException extends RuntimeException {
    public InsufficientBalanceException() { super(); }
    public InsufficientBalanceException(String msg) { super(msg); }
    public InsufficientBalanceException(String msg, Throwable cause) { super(msg, cause); }
    public InsufficientBalanceException(Throwable cause) { super(cause); }
}
```

**Add domain-specific fields** when useful:
```java
public class InsufficientBalanceException extends RuntimeException {
    private final BigDecimal available;
    private final BigDecimal requested;
    public InsufficientBalanceException(BigDecimal available, BigDecimal requested) {
        super("Need " + requested + ", have " + available);
        this.available = available;
        this.requested = requested;
    }
    public BigDecimal getAvailable() { return available; }
    public BigDecimal getRequested() { return requested; }
}
```

**Best practices:**
- Name should end in `Exception`.
- Make the exception serializable (most inherited `Exception` already is).
- Include meaningful context in the message — future-you debugging a production log will thank you.
- Prefer unchecked (`RuntimeException`) unless you genuinely want to force the caller to handle it.

---

## 7. try-with-resources (Java 7+)

Automatically closes resources — files, streams, DB connections — even if exceptions occur. Replaces verbose try-finally-close patterns.

**Requirement:** The resource must implement `java.lang.AutoCloseable` (whose single method is `close()`), or its subinterface `java.io.Closeable`.

**Syntax:**
```java
try (BufferedReader br = new BufferedReader(new FileReader("file.txt"));
     FileWriter fw = new FileWriter("out.txt")) {
    // use br and fw
} catch (IOException e) {
    // handle
}
// both br and fw are auto-closed, in REVERSE order of declaration
```

**Java 9+ enhancement:** You can use effectively final resources declared outside the try:
```java
BufferedReader br = new BufferedReader(new FileReader("x.txt"));
try (br) {
    // use br
}
```

**Suppressed exceptions:**
If `close()` throws while the try block already threw, the original exception is preserved as the primary, and the close-time exception is added to it via `addSuppressed()`. Retrieve with `Throwable.getSuppressed()`.

**Why it matters:** Without try-with-resources, handling the close() exception properly while still reporting the primary exception requires 10+ lines of boilerplate. It also prevents resource leaks — the #1 cause of server degradation.

---

## 8. finally Block Behavior (tricky)

`finally` is designed for cleanup — close resources, release locks, log exit.

**Always executes:**
- After normal completion of try.
- After an exception caught by catch.
- After an exception NOT caught (re-thrown automatically).
- After a `return` in try or catch.

**Does NOT execute when:**
- `System.exit(n)` is called — JVM terminates before finally.
- The JVM crashes (e.g., fatal native code, `Runtime.halt`).
- The thread is killed via `Thread.stop` (deprecated) or the OS kills the process.
- An infinite loop or deadlock occurs in try/catch.

**Dangerous patterns:**

1. **`return` inside finally** overrides any earlier return — silently discards it.
```java
int m() {
    try { return 1; }
    finally { return 2; } // returns 2, hides 1
}
```

2. **Throwing inside finally** masks the original exception:
```java
try { throw new RuntimeException("primary"); }
finally { throw new RuntimeException("secondary"); }
// caller sees "secondary", "primary" is LOST
```

3. **Modifying returned primitive** has NO effect — the return value was already copied:
```java
int m() {
    int x = 1;
    try { return x; }
    finally { x = 99; } // still returns 1 (primitive copied)
}
```

4. **Modifying returned object reference's fields** DOES take effect, because the reference was returned but the object's state is still mutable:
```java
StringBuilder m() {
    StringBuilder sb = new StringBuilder("a");
    try { return sb; }
    finally { sb.append("b"); } // returned object shows "ab"
}
```

**Interview takeaway:** Never put `return` or `throw` inside `finally` — it masks bugs and makes code harder to reason about. Use finally ONLY for cleanup.

---

# 🔹 JVM Internals

## 9. JVM Architecture

The JVM is the runtime engine that executes Java bytecode. Its architecture breaks into three main subsystems:

```
┌────────────────────────────────────────────────────┐
│  Class Loader Subsystem                            │
│  (Loading → Linking → Initialization)              │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│  Runtime Data Areas                                │
│  ┌─────────────┐  ┌─────────┐  ┌──────────────┐    │
│  │ Method Area │  │  Heap   │  │ Stack (per   │    │
│  │ (Metaspace) │  │         │  │   thread)    │    │
│  └─────────────┘  └─────────┘  └──────────────┘    │
│  ┌─────────────────────┐  ┌───────────────────┐    │
│  │ PC Register         │  │ Native Method     │    │
│  │ (per thread)        │  │ Stack (per thread)│    │
│  └─────────────────────┘  └───────────────────┘    │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│  Execution Engine                                  │
│  (Interpreter + JIT + Garbage Collector)           │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│  Native Method Interface (JNI) → Native Libraries  │
└────────────────────────────────────────────────────┘
```

**What happens when `java MyApp` runs:**
1. ClassLoader finds `MyApp.class`, verifies bytecode, resolves references, initializes static fields.
2. JVM allocates runtime data areas.
3. Execution engine starts interpreting `main()`.
4. Hot code paths get JIT-compiled to native machine code for speed.
5. GC runs periodically to reclaim unreachable objects.

---

## 10. ClassLoader

Class loaders load `.class` files into the JVM on demand (lazy loading). Java uses a **hierarchical parent-delegation model**:

**1. Bootstrap ClassLoader**
- Written in native code (C/C++) as part of the JVM itself.
- Parent of all other class loaders; its own parent is `null`.
- Loads core Java classes from `$JAVA_HOME/lib` (`rt.jar` pre-Java 9; modules like `java.base` in Java 9+).
- When you query `Object.class.getClassLoader()`, you get `null` because Bootstrap isn't a Java object.

**2. Extension / Platform ClassLoader**
- Pre-Java 9: **Extension ClassLoader** — loaded JARs from `$JAVA_HOME/lib/ext`.
- Java 9+: renamed **Platform ClassLoader** — loads non-core platform modules (e.g., `java.sql`, `java.xml`).

**3. Application / System ClassLoader**
- Loads user classes from the classpath (`-cp` or `CLASSPATH` env var).
- Default for user-written code.

**Parent Delegation Model:**
When asked to load a class, a class loader first delegates to its **parent**. Only if the parent cannot find the class does the child attempt to load it.

**Why delegation matters:**
- **Security** — prevents malicious classes from impersonating core Java classes. If you write `java.lang.String`, Bootstrap still loads the real one.
- **Uniqueness** — guarantees core classes are loaded exactly once.
- **Namespacing** — two classes with the same fully qualified name but loaded by different loaders are considered **distinct types** by the JVM (this is how app servers isolate web apps).

**Custom class loaders:**
Extend `ClassLoader` to load classes from network, encrypted JARs, database, etc. Override `findClass(String name)` (not `loadClass` — you want to preserve delegation).

**Common interview questions:**
- "Can two classes with the same name coexist in one JVM?" — Yes, if loaded by different class loaders.
- "What's `ClassNotFoundException` vs `NoClassDefFoundError`?" — `ClassNotFoundException` is thrown by explicit `Class.forName`/`loadClass` when class is missing; `NoClassDefFoundError` is thrown by JVM when a class that was present at compile time is missing at runtime.

---

## 11. Method Area / Metaspace

The **Method Area** is a JVM memory region (shared across all threads) that stores class-level metadata:
- Class structure (fields, methods, parent class, interfaces)
- Method bytecode
- Runtime constant pool (string literals, symbolic references)
- Static variables
- JIT-compiled native code (sometimes)

**Pre-Java 8 — PermGen:**
- Part of the heap with a fixed size, controlled by `-XX:PermSize` and `-XX:MaxPermSize`.
- Notorious for `java.lang.OutOfMemoryError: PermGen space` — common in app servers that deploy/redeploy apps (class metadata leaked).
- String literal interning happened in PermGen until Java 7.

**Java 8+ — Metaspace:**
- Moved to **native memory** (outside the JVM heap).
- Auto-grows by default (limited by `-XX:MaxMetaspaceSize`, default unlimited).
- Eliminates most PermGen OOM scenarios.
- String pool moved to the heap, so interned strings are GC-able.
- Still possible to get `OutOfMemoryError: Metaspace` if you generate endless classes at runtime (bytecode libraries, class reloading bugs).

**Why the change:** PermGen's fixed size caused tuning headaches. Moving to native memory means JVM can resize without heap pressure.

---

## 12. JIT Compiler (Just-In-Time)

The JIT compiler translates bytecode to native machine code at runtime, dramatically boosting performance after "warmup."

**Execution pipeline:**
1. JVM starts executing bytecode through the **interpreter** (slow but starts immediately).
2. HotSpot profiler counts method invocations and loop iterations.
3. When a method crosses a **threshold** (`-XX:CompileThreshold`, default ~10,000 invocations), JIT compiles it to native code.
4. Subsequent calls run the native version — often 10-100× faster.

**Tiered Compilation (default since Java 8):**
- **Tier 0:** interpreter.
- **Tier 1-3:** **C1 compiler** — fast compilation, basic optimizations, starts profiling.
- **Tier 4:** **C2 compiler** — slow compilation but aggressive optimization, only for very hot code.

**Optimizations JIT performs:**
- **Method inlining** — replaces method call with the method body (huge performance win, enables further optimizations).
- **Escape analysis** — if an object never "escapes" the method, allocate on stack (no GC).
- **Dead code elimination** — removes unreachable branches.
- **Loop unrolling** — reduces loop overhead.
- **Null-check elision** — skips checks when the compiler proves they're unnecessary.
- **Speculative optimization** — optimizes assuming the common case; **deoptimizes** if assumption violated.

**AOT (Ahead-Of-Time):**
Java 9 introduced AOT compilation (`jaotc`), but it was removed in Java 17. GraalVM Native Image is the modern AOT solution — compiles Java to a native executable with minimal startup time (great for serverless/CLI, but no JIT adaptivity).

---

## 13. Execution Engine

Executes the bytecode loaded by the class loader. Has three main components:

**1. Interpreter**
- Reads and executes bytecode one instruction at a time.
- Portable across platforms but slow.
- Used at JVM startup and for cold code.

**2. JIT Compiler** (see above)
- Compiles hot methods to native code for speed.
- Stores compiled code in the **code cache**.

**3. Garbage Collector**
- Runs automatically to reclaim memory used by unreachable objects.
- Pluggable — several GC algorithms available (see GC section).

**4. Native Method Interface (JNI)**
- Allows Java code to call native C/C++ libraries.
- Used by JDK internals (file I/O, threading) and third-party libraries.

---

## 14. Garbage Collection

GC frees memory held by objects that are no longer reachable from any GC root (thread stack variables, static fields, JNI references, etc.).

**Generational Hypothesis:**
"Most objects die young." Java divides the heap into generations, treating young and old objects differently.

**Heap layout:**
```
Young Generation
├── Eden
└── Survivor spaces (S0 and S1)
Old Generation (Tenured)
Metaspace (native memory, not heap)
```

**How allocation works:**
1. New object → Eden.
2. When Eden fills → **Minor GC** (stop-the-world):
   - Live objects copied from Eden to one Survivor space.
   - Older survivors are also copied between S0/S1, incrementing an "age" counter.
   - When age exceeds `MaxTenuringThreshold` (default ~15), object is **promoted** to Old gen.
3. When Old gen fills → **Major GC** (or **Full GC** — includes Metaspace cleanup).

**Major GC algorithms:**

**Serial GC** (`-XX:+UseSerialGC`)
- Single-threaded, stop-the-world.
- Good for small single-CPU environments (embedded, desktop).

**Parallel GC** (`-XX:+UseParallelGC`)
- Multi-threaded young and old generation collectors.
- Was default through Java 8 for server class machines.
- Best throughput, worst pause times.

**CMS — Concurrent Mark Sweep** (`-XX:+UseConcMarkSweepGC`) — DEPRECATED in Java 9, removed in Java 14
- Concurrent marking (application keeps running during most of the GC cycle).
- Low pause times but suffered from heap fragmentation.

**G1 — Garbage First** (`-XX:+UseG1GC`) — DEFAULT since Java 9
- Divides heap into ~2048 **regions** of equal size (typically 1-32 MB each).
- Each region can be Eden, Survivor, Old, or Humongous.
- Tracks which regions have the most garbage and collects those **first** (hence "Garbage First").
- Aims for **predictable pause times** (target via `-XX:MaxGCPauseMillis`).
- Does concurrent marking + copying/compaction in parallel.

**ZGC** (`-XX:+UseZGC`) — production-ready since Java 15
- **Sub-10ms pause times** regardless of heap size (tested up to 16TB heaps).
- Uses **colored pointers** and **load barriers** for concurrent compaction.
- Ideal for low-latency, large-heap workloads.

**Shenandoah** (`-XX:+UseShenandoahGC`) — similar goals to ZGC, developed by Red Hat.

**GC tuning flags:**
- `-Xms` / `-Xmx` — initial/max heap size.
- `-Xmn` — young gen size.
- `-XX:MaxGCPauseMillis` — target pause time for G1.
- `-XX:NewRatio` — ratio of old to young gen.
- `-Xlog:gc*` (Java 9+) — GC logging.

**Key interview concepts:**
- **Stop-the-world (STW):** pause of all application threads during GC.
- **GC Roots:** starting points from which reachability is traced (local variables, active thread stacks, static fields, JNI references).
- **Memory leaks in Java:** caused by lingering references (e.g., unbounded caches, listeners not unregistered, static Maps).

---

## 15. Object Lifecycle

1. **Allocation** — `new Foo()` allocates memory in Eden (fast bump-the-pointer allocation).
2. **Initialization** — constructor chain runs (from `Object` down).
3. **Usage** — object used by application.
4. **Unreachability** — all references to the object are gone.
5. **Finalization** (legacy) — if class overrides `finalize()`, it's queued for the finalizer thread.
6. **Reclamation** — GC reclaims the memory.

During step 2, the JVM may also perform:
- **Class loading** (if Foo's class isn't loaded yet).
- **Class initialization** (static blocks, static fields — happens ONCE per class).
- **Object header setup** (mark word, klass pointer).

**Memory footprint of an object:**
- Object header (~12-16 bytes on 64-bit JVM with compressed oops).
- Fields (aligned to 8-byte boundaries by default).
- Padding for alignment.

---

# 🔹 Java 8+ Features

## 16. Lambda Expressions

A lambda is an **anonymous function** — a concise way to represent a single method interface (functional interface). Introduced to support functional programming idioms and simplify APIs that take behavior as parameters.

**Syntax forms:**
```java
// No parameters
() -> System.out.println("Hello")

// Single parameter (parentheses optional)
x -> x * 2

// Multiple parameters
(x, y) -> x + y

// With type declarations
(int x, int y) -> x + y

// Block body
(x, y) -> {
    int sum = x + y;
    return sum;
}
```

**Key properties:**

1. **Target typing** — the lambda's type is inferred from the context. The compiler looks at the functional interface expected and matches the lambda signature to it.
```java
Runnable r = () -> System.out.println("run"); // target type: Runnable
Comparator<String> c = (a, b) -> a.compareTo(b); // target type: Comparator<String>
```

2. **Variable capture** — lambdas can access variables from the enclosing scope, but **captured local variables must be effectively final** (not reassigned after capture).
```java
int x = 10;
Runnable r = () -> System.out.println(x); // OK
x = 20; // ERROR: x is no longer effectively final
```

3. **`this` refers to enclosing class** — unlike anonymous inner classes where `this` refers to the anonymous class itself.

4. **No shadowing** — lambda parameters can't have the same name as enclosing scope variables (unlike inner classes).

**Under the hood:**
Lambdas are NOT compiled to inner classes. They use the `invokedynamic` bytecode instruction + `LambdaMetafactory` to lazily create an implementation class at runtime — much more efficient than anonymous inner classes (no separate .class file, lighter objects, often cached/reused).

**Benefits:**
- Less boilerplate.
- Enables pass-behavior-as-parameter idioms (strategy, callbacks).
- Required for Streams, `CompletableFuture`, reactive programming.

---

## 17. Functional Interfaces

An interface with **exactly one abstract method** (SAM — Single Abstract Method). It's the target type for lambdas and method references.

**Rules:**
- Can have any number of `default` and `static` methods.
- Can override methods from `Object` (`equals`, `hashCode`, `toString`) — these don't count as abstract.
- `@FunctionalInterface` annotation is optional but recommended — the compiler enforces the single-abstract-method rule.

**Built-in functional interfaces (`java.util.function` package):**

| Interface | Signature | Purpose |
|-----------|-----------|---------|
| `Function<T, R>` | `R apply(T t)` | Transform T into R |
| `Predicate<T>` | `boolean test(T t)` | Test a condition |
| `Consumer<T>` | `void accept(T t)` | Consume T (side effect) |
| `Supplier<T>` | `T get()` | Produce T (factory) |
| `BiFunction<T,U,R>` | `R apply(T t, U u)` | Two-arg Function |
| `BinaryOperator<T>` | `T apply(T, T)` | Combine two T's into T |
| `UnaryOperator<T>` | `T apply(T)` | Function<T,T> |
| `Runnable` | `void run()` | No-arg, no-return |
| `Callable<V>` | `V call() throws Exception` | No-arg, returns V, can throw |
| `Comparator<T>` | `int compare(T, T)` | Comparison |

**Primitive specializations** avoid autoboxing (big perf win):
- `IntFunction<R>`, `IntPredicate`, `IntConsumer`, `IntSupplier`, `IntUnaryOperator`
- Same for `Long`, `Double`
- `ToIntFunction<T>`, `ToLongFunction<T>`, `ToDoubleFunction<T>`

**Custom example:**
```java
@FunctionalInterface
public interface TriFunction<A, B, C, R> {
    R apply(A a, B b, C c);
}
```

---

## 18. Streams API

A **stream** is a sequence of elements supporting aggregate operations in a declarative, functional style. Unlike collections, streams **don't store data** — they describe a computation pipeline.

**Pipeline anatomy:**
```
Source → Intermediate Ops → Terminal Op
```

**Example:**
```java
List<String> names = List.of("Alice", "Bob", "Charlie", "Dave");
List<String> result = names.stream()               // source
    .filter(n -> n.length() > 3)                   // intermediate
    .map(String::toUpperCase)                      // intermediate
    .sorted()                                      // intermediate
    .collect(Collectors.toList());                 // terminal
// result: [ALICE, CHARLIE, DAVE]
```

**Key properties:**

1. **Lazy evaluation** — intermediate operations do nothing until a terminal operation is invoked. This enables optimizations like short-circuiting (`findFirst`, `anyMatch`).

2. **Single-use** — a stream can be traversed once. Reusing throws `IllegalStateException`.

3. **Non-mutating** — operations produce new streams; the source is untouched.

4. **Parallel support** — `.parallelStream()` or `.stream().parallel()` splits work across ForkJoinPool (default pool size = number of cores). Only helps with large, CPU-bound workloads with stateless operations.

**Creating streams:**
```java
Stream.of("a", "b", "c")
Arrays.stream(arr)
list.stream()
Files.lines(Path.of("file.txt"))
Stream.iterate(1, i -> i * 2).limit(10) // infinite, capped
Stream.generate(Math::random).limit(5)
IntStream.range(0, 10)      // 0..9
IntStream.rangeClosed(0, 10) // 0..10
```

**Gotchas:**
- Operations inside streams should be **stateless** and **side-effect-free** (especially for parallel streams).
- Don't mutate shared state in `forEach` — use `collect` instead.
- Parallel streams use the common ForkJoinPool, which is shared across the entire JVM. One slow task can starve others.

---

## 19. Stream vs Collection

| Aspect | Collection | Stream |
|--------|-----------|--------|
| **Purpose** | Data storage | Data computation |
| **Storage** | In-memory | Nothing stored |
| **Traversal** | Multiple times | Once only |
| **Mutability** | Can add/remove | Immutable pipeline |
| **Evaluation** | Eager | Lazy |
| **Iteration** | External (for-each) | Internal (API controls) |
| **Size** | Finite | Can be infinite (with limit) |
| **Parallelism** | Manual | Built-in (`.parallel()`) |

**External vs internal iteration:**
```java
// External — you control the loop
for (String s : list) { if (s.startsWith("A")) result.add(s); }

// Internal — API controls iteration, you declare what to do
list.stream().filter(s -> s.startsWith("A")).collect(toList());
```

Internal iteration is more declarative, enables parallel execution, and lets the JVM optimize the traversal.

---

## 20. Intermediate vs Terminal Operations

**Intermediate operations:**
- Return a new Stream — chainable.
- **Lazy** — don't execute until a terminal op triggers the pipeline.
- Can be **stateless** (`filter`, `map`, `peek`) or **stateful** (`sorted`, `distinct`, `limit`, `skip`).
- Stateful ops may require buffering the entire input before producing output.

| Operation | Purpose | Type |
|-----------|---------|------|
| `filter(Predicate)` | Keep matching elements | Stateless |
| `map(Function)` | Transform each element | Stateless |
| `flatMap(Function)` | Flatten nested structures | Stateless |
| `distinct()` | Remove duplicates | Stateful |
| `sorted()` / `sorted(Comparator)` | Sort | Stateful |
| `peek(Consumer)` | Side effect (debugging) | Stateless |
| `limit(n)` | Take first n | Short-circuiting stateful |
| `skip(n)` | Drop first n | Stateful |
| `takeWhile(Predicate)` (Java 9+) | Take until false | Short-circuiting stateful |
| `dropWhile(Predicate)` (Java 9+) | Drop until false | Stateful |

**Terminal operations:**
- Trigger pipeline execution.
- Produce a non-Stream result (collection, primitive, Optional, or void).
- After terminal op, the stream is consumed.

| Operation | Purpose |
|-----------|---------|
| `collect(Collector)` | Accumulate into a collection/map |
| `forEach(Consumer)` | Perform action on each |
| `forEachOrdered` | Like forEach but preserves order in parallel |
| `reduce(...)` | Fold to single value |
| `count()` | Number of elements |
| `min()` / `max()` | Extremes |
| `findFirst()` / `findAny()` | Return Optional, short-circuits |
| `anyMatch` / `allMatch` / `noneMatch` | Boolean tests, short-circuit |
| `toArray()` | Convert to array |
| `toList()` (Java 16+) | Convenience for `collect(toList())` |

**Short-circuiting** means the operation can complete without processing every element (e.g., `findFirst` stops at first match; `anyMatch` stops at first true). Critical for infinite streams.

---

## 21. Optional

`Optional<T>` is a container that may or may not hold a non-null value. Designed to reduce `NullPointerException` and to make nullability **explicit in the API**.

**Creation:**
```java
Optional<String> empty = Optional.empty();
Optional<String> present = Optional.of("hello");           // NPE if null
Optional<String> maybe = Optional.ofNullable(someString);  // empty if null
```

**Inspection:**
```java
opt.isPresent()    // true if value
opt.isEmpty()      // Java 11+
opt.ifPresent(System.out::println)
opt.ifPresentOrElse(val -> ..., () -> ...)  // Java 9+
```

**Retrieval:**
```java
opt.get()                             // throws NoSuchElementException if empty — avoid
opt.orElse("default")                 // returns default if empty
opt.orElseGet(() -> computeDefault()) // lazy default
opt.orElseThrow()                     // throws NoSuchElementException (Java 10+)
opt.orElseThrow(() -> new MyException())
```

**Transformation:**
```java
opt.map(String::toUpperCase)    // Optional<String>
opt.flatMap(s -> Optional.of(s.trim())) // Avoid nested Optional
opt.filter(s -> s.length() > 3) // Optional, empty if predicate fails
```

**`orElse` vs `orElseGet`:** `orElse` **always evaluates** its argument, even when Optional is present. `orElseGet` evaluates lazily via Supplier. Use `orElseGet` when the default is expensive.

**Best practices (Stuart Marks, JDK architect):**
- ✅ Use as **return type** from a method where absence is a valid outcome.
- ❌ Don't use as a **field** in domain classes.
- ❌ Don't use as a **method parameter**.
- ❌ Don't use in **collections** — prefer empty collections or streams.
- ❌ Don't call `get()` without checking `isPresent()` — defeats the purpose.
- ❌ Don't use for primitive-specific cases — use `OptionalInt`, `OptionalLong`, `OptionalDouble`.

---

## 22. Method References

Shorthand syntax for lambdas that simply invoke a single method. Uses `::` operator.

**Four forms:**

1. **Static method reference:** `ClassName::staticMethod`
```java
Function<String, Integer> parser = Integer::parseInt;
// equivalent to: s -> Integer.parseInt(s)
```

2. **Bound instance method reference:** `instance::method`
```java
String prefix = "Hello ";
Function<String, String> greet = prefix::concat;
// equivalent to: s -> prefix.concat(s)
```

3. **Unbound instance method reference:** `ClassName::instanceMethod`
```java
Function<String, Integer> len = String::length;
// equivalent to: s -> s.length()
```

4. **Constructor reference:** `ClassName::new`
```java
Supplier<ArrayList<String>> factory = ArrayList::new;
Function<Integer, ArrayList<String>> sized = ArrayList::new;
```

**Benefits:**
- More readable when the lambda just forwards its args to a method.
- Slightly faster (JVM can cache the reference).

**When NOT to use:** if transformations happen inline (`x -> x * 2` can't be a method reference).

---

## 23. Default & Static Methods in Interface

**Problem solved:** Before Java 8, adding a method to an interface broke every implementing class. This made interface evolution painful (`Collection.stream()` couldn't be added without breaking the world).

**Default methods:**
- Methods with a **body** defined in an interface using `default` keyword.
- Implementing classes inherit the default but can override.
- Enable **backward-compatible API evolution**.

```java
public interface List<E> {
    default void forEach(Consumer<? super E> action) {
        for (E e : this) action.accept(e);
    }
}
```

**Static methods in interfaces:**
- Belong to the interface itself (not inherited by implementers).
- Used for utility methods related to the interface.
```java
public interface Comparator<T> {
    static <T> Comparator<T> naturalOrder() { ... }
    static <T> Comparator<T> reverseOrder() { ... }
}
```

**Private methods (Java 9+):**
- Helper methods within an interface that default/static methods can share.
- Not visible to implementers.

**Diamond problem & resolution rules:**
When a class inherits the same default method from multiple interfaces:

```java
interface A { default void hi() { System.out.println("A"); } }
interface B { default void hi() { System.out.println("B"); } }
class C implements A, B { /* COMPILE ERROR */ }
```

Resolution rules (in priority order):
1. **Class wins** — if a superclass provides a concrete method, it overrides any interface default.
2. **More specific interface wins** — if one interface extends another, the child wins.
3. **Otherwise, the class MUST override** and can explicitly call either with `A.super.hi()` or `B.super.hi()`.

**Limitation:** cannot override methods from `Object` (like `toString`) with default methods.

---

# 🔹 Serialization

## 24. Serialization & Deserialization

**Serialization** is the process of converting an object's state into a byte stream so it can be:
- Persisted to disk.
- Sent over a network (RMI, message queues).
- Cached (e.g., Redis serializers, session replication).

**Deserialization** reverses the process — byte stream back into an object.

**Core API:**
```java
// Serialize
try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("obj.ser"))) {
    oos.writeObject(myObj);
}

// Deserialize
try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("obj.ser"))) {
    MyClass obj = (MyClass) ois.readObject();
}
```

**How it works under the hood:**
1. JVM checks the class implements `Serializable`.
2. JVM walks the object graph, serializing each reachable non-transient field.
3. For each object, JVM writes class metadata (name, `serialVersionUID`, field descriptors) followed by field values.
4. Circular references and shared references are handled via a reference table — shared objects are serialized once, referenced by handle.

**What gets serialized:**
- All instance fields NOT marked `transient` and NOT `static`.
- The entire object graph (deep serialization by default).
- Class metadata (name, version ID, field types).

**What does NOT get serialized:**
- `transient` fields.
- `static` fields (they belong to the class, not the instance).
- Constructor does NOT run during deserialization — fields are set directly via reflection.

**Security concerns:**
- Deserialization is a **major attack vector** (remote code execution via crafted streams). Never deserialize untrusted data.
- Alternatives: JSON (Jackson), Protocol Buffers, Avro — schema-based, safer, language-agnostic.

---

## 25. Serializable Interface

A **marker interface** (no methods) signaling to the JVM that instances of this class may be serialized.

```java
public class Employee implements Serializable {
    private String name;
    private int id;
    // ...
}
```

**Rules:**
- All non-transient, non-static fields must themselves be serializable (or be primitives).
- If a field's type is not serializable, the field must be `transient` or the class must use `Externalizable`.
- If a superclass is not serializable but a subclass is, the superclass must have a **no-arg constructor** — deserialization will invoke it to set superclass fields to default values.
- Inner classes capture a reference to the enclosing instance — avoid serializing non-static inner classes.

**Customization hooks (NOT methods of Serializable, but recognized by the JVM):**
```java
private void writeObject(ObjectOutputStream oos) throws IOException {
    oos.defaultWriteObject();   // handle default fields first
    oos.writeInt(customValue);  // then add custom
}

private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
    ois.defaultReadObject();
    customValue = ois.readInt();
}

private Object writeReplace() { ... }   // substitute object on serialization
private Object readResolve() { ... }    // substitute object on deserialization (e.g., for singletons/enums)
```

---

## 26. transient Keyword

Marks a field to be **excluded** from default serialization. Upon deserialization, the field will have its default value (null for objects, 0 for numbers, false for booleans).

**Common uses:**
- Sensitive data (passwords, API keys).
- Derived / cached fields that can be recomputed.
- Fields referencing non-serializable resources (DB connections, file handles, threads).

```java
public class User implements Serializable {
    private String username;
    private transient String password;  // don't persist
    private transient Logger logger = Logger.getLogger(User.class); // recreate
}
```

**Recreating transient state on deserialization:**
Use `readObject` to re-initialize transient fields:
```java
private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
    ois.defaultReadObject();
    this.logger = Logger.getLogger(User.class);
}
```

---

## 27. serialVersionUID

A **version identifier** for serialized classes. Used to verify that the serialized data is compatible with the class definition during deserialization.

```java
private static final long serialVersionUID = 1L;
```

**What happens without it:**
- The JVM computes one based on the class structure (fields, methods, interfaces, modifiers).
- ANY change to the class (adding a field, changing a method signature) changes the computed UID → existing serialized data becomes unreadable with `InvalidClassException`.

**Best practice:** Always declare `serialVersionUID` explicitly. Set it to `1L` initially; **increment only when making incompatible changes**.

**Compatible changes (same UID OK):**
- Adding new fields (old data deserializes with defaults for new fields).
- Adding methods.
- Changing `transient` status.

**Incompatible changes (should bump UID):**
- Changing field types.
- Removing fields.
- Changing class hierarchy.
- Changing Serializable/Externalizable.

**Interview trap:** `serialVersionUID` must be **`private static final long`**. Any other modifier combination is ignored.

---

## 28. Externalizable (advanced)

A sub-interface of `Serializable` giving the developer **full control** over the serialization format.

```java
public interface Externalizable extends Serializable {
    void writeExternal(ObjectOutput out) throws IOException;
    void readExternal(ObjectInput in) throws IOException, ClassNotFoundException;
}
```

**Key differences from Serializable:**

| Aspect | Serializable | Externalizable |
|--------|--------------|----------------|
| Mechanism | Reflection | Manual `write`/`read` methods |
| Transient fields | Skipped automatically | Developer decides |
| Constructor on deserialize | NOT called | **Public no-arg constructor IS called** |
| Performance | Slower (reflection) | Faster (direct writes) |
| Format control | Limited | Full |

**Use case:** Performance-critical systems, custom binary formats, interop with non-Java systems.

**Example:**
```java
public class Point implements Externalizable {
    private int x, y;

    public Point() { }  // MANDATORY public no-arg constructor

    public Point(int x, int y) { this.x = x; this.y = y; }

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeInt(x);
        out.writeInt(y);
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException {
        x = in.readInt();
        y = in.readInt();
    }
}
```

**Important:** Deserialization of `Externalizable` first calls the **public no-arg constructor**, then `readExternal`. This means your constructor's logic WILL run (unlike with `Serializable`).

---

# 🔹 Enum

## 29. Enum Basics

An `enum` is a special class that represents a **fixed, finite set of constants**. Introduced in Java 5.

```java
public enum Day {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY
}
```

**Key properties:**
- Each enum value is an **instance of the enum type** — fully typed.
- Enum values are implicitly `public`, `static`, `final`.
- Enum types implicitly extend `java.lang.Enum<E>` — so they cannot extend any other class (but CAN implement interfaces).
- You cannot instantiate an enum with `new` — values are fixed at class load time.
- Enums are **singletons per value** (guaranteed by JVM, immune to reflection attacks).

**Built-in methods (inherited from `Enum`):**
```java
Day d = Day.MONDAY;
d.name()       // "MONDAY"
d.ordinal()    // 0 (position)
Day.values()   // Day[] of all values (compiler-generated)
Day.valueOf("MONDAY")  // throws IllegalArgumentException if not found
```

**Enums in switch:**
```java
switch (day) {
    case MONDAY: ...     // no need for Day.MONDAY
    case TUESDAY: ...
}
```

Switch expressions (Java 14+):
```java
String type = switch (day) {
    case SATURDAY, SUNDAY -> "weekend";
    default -> "weekday";
};
```

---

## 30. Enum vs Constants

**Old way — int constants:**
```java
public class Days {
    public static final int MONDAY = 1;
    public static final int TUESDAY = 2;
    // ...
}
```

**Problems:**
- **Not type-safe** — any int can be passed where a day is expected.
- **No namespace** — can clash with other constants.
- **Printed as numbers** — not meaningful in logs.
- **Can't iterate** — no way to get all values.
- **Hard to add behavior** — no methods attached.

**Enum way solves all of it:**
- Type-safe (compiler rejects non-enum values).
- Self-documenting (`MONDAY` prints as "MONDAY").
- `values()` to iterate.
- Can carry data and behavior.
- Implicit singleton — safe to use `==` for equality.

**Result:** Enums should replace constant groups whenever the set is closed and known at compile time.

---

## 31. Enum with Methods and Fields

Enums can have fields, constructors, and methods — they're essentially full classes with a fixed set of instances.

```java
public enum Planet {
    MERCURY(3.303e+23, 2.4397e6),
    VENUS(4.869e+24, 6.0518e6),
    EARTH(5.976e+24, 6.37814e6);

    private final double mass;   // kg
    private final double radius; // m

    Planet(double mass, double radius) {   // constructor is implicitly private
        this.mass = mass;
        this.radius = radius;
    }

    public double surfaceGravity() {
        return G * mass / (radius * radius);
    }

    public static final double G = 6.67300E-11;
}
```

**Constant-specific method bodies** — override per constant:
```java
public enum Operation {
    PLUS  { public double apply(double x, double y) { return x + y; } },
    MINUS { public double apply(double x, double y) { return x - y; } },
    TIMES { public double apply(double x, double y) { return x * y; } },
    DIV   { public double apply(double x, double y) { return x / y; } };

    public abstract double apply(double x, double y);
}
```

**Enums can implement interfaces:**
```java
public enum Status implements Printable {
    ACTIVE {
        public void print() { System.out.println("Active"); }
    },
    INACTIVE {
        public void print() { System.out.println("Inactive"); }
    };
}
```

**Enum Singleton Pattern (Joshua Bloch):**
```java
public enum DatabaseConnection {
    INSTANCE;

    public void connect() { ... }
}

// Usage:
DatabaseConnection.INSTANCE.connect();
```

This is the **best singleton** because:
- **Serialization-safe** (Enum serialization guarantees singleton).
- **Reflection-safe** (JVM forbids `Enum.newInstance` via reflection).
- **Thread-safe** by default.
- **Lazy initialization** by JVM (class loading rules).

**Advanced enums:**
- `EnumSet` — high-performance Set implementation backed by a bit vector.
- `EnumMap` — high-performance Map where keys are enum values (backed by an array indexed by ordinal).

---

# 🔹 Reflection API

## 32. What is Reflection

**Reflection** is the ability of a Java program to **inspect and modify** its own structure (classes, methods, fields, annotations) **at runtime** — without knowing class names at compile time.

Core classes in `java.lang.reflect`:
- `Class<?>` — the entry point, represents a loaded class.
- `Method` — represents a method.
- `Field` — represents a field.
- `Constructor<?>` — represents a constructor.
- `Modifier` — utility for decoding access flags.
- `Array` — dynamic array creation/access.
- `Proxy` — create dynamic proxies.

**Getting a Class object:**
```java
Class<?> c1 = String.class;                    // class literal
Class<?> c2 = "abc".getClass();                // from instance
Class<?> c3 = Class.forName("java.lang.String"); // by name (throws ClassNotFoundException)
```

**Inspecting a class:**
```java
Class<?> c = MyClass.class;

c.getName();             // "com.example.MyClass"
c.getSimpleName();       // "MyClass"
c.getSuperclass();       // Class<?> for parent
c.getInterfaces();       // Class<?>[] implemented
c.getModifiers();        // int — decode with Modifier
c.getPackage();          // Package

c.getFields();           // public fields only (including inherited)
c.getDeclaredFields();   // ALL fields declared in this class (any access)

c.getMethods();          // public methods (including inherited)
c.getDeclaredMethods();  // ALL methods declared in this class

c.getConstructors();     // public constructors
c.getDeclaredConstructors(); // ALL constructors
```

**Dynamic instantiation and invocation:**
```java
Class<?> c = Class.forName("com.example.MyClass");

Constructor<?> ctor = c.getDeclaredConstructor(String.class, int.class);
ctor.setAccessible(true);  // bypass access checks
Object instance = ctor.newInstance("hello", 42);

Method m = c.getDeclaredMethod("doSomething", String.class);
m.setAccessible(true);
Object result = m.invoke(instance, "arg");

Field f = c.getDeclaredField("name");
f.setAccessible(true);
f.set(instance, "new value");
String value = (String) f.get(instance);
```

---

## 33. Use Cases of Reflection

**1. Frameworks**
- **Spring** — dependency injection scans for `@Component`, `@Autowired`, then instantiates and wires via reflection.
- **Hibernate / JPA** — maps entity fields to DB columns via reflection.
- **Jackson / Gson** — serialize/deserialize objects to JSON by reflecting on fields.
- **JUnit** — finds `@Test` methods, invokes them reflectively.

**2. Dynamic Proxies**
```java
Object proxy = Proxy.newProxyInstance(
    classLoader,
    new Class<?>[] { MyService.class },
    (p, method, args) -> {
        // pre-processing (logging, security)
        Object result = method.invoke(realTarget, args);
        // post-processing
        return result;
    }
);
```
- Used by Spring AOP, Hibernate lazy loading, RPC frameworks.

**3. IDE Features**
- Auto-complete, refactoring, debugging.

**4. Plugin Architectures**
- Load classes from external JARs / network, discover services.
- `ServiceLoader` uses reflection under the hood.

**5. Serialization Libraries**
- Reflect on all fields to serialize/deserialize.

**6. Testing Tools**
- Mockito reflects to create mocks; JUnit reflects to find tests and assertions.

---

## 34. Performance Impact of Reflection

Reflection is **much slower** than direct method calls due to:
1. **Access checks** — security checks on every call unless `setAccessible(true)`.
2. **Method lookup** — symbolic resolution of method by name.
3. **No JIT inlining** — the JIT can't easily inline reflective calls.
4. **Boxing** — primitives must be boxed to `Object` for `invoke`.
5. **Varargs array allocation** — arguments wrapped in Object[] every call.

**Typical overhead:** 10-100× slower than direct call for small methods. Less noticeable for large methods.

**Mitigation strategies:**
1. **Cache `Method`/`Field`/`Constructor` objects** — don't call `getMethod` on every invocation.
2. **Call `setAccessible(true)`** once — skips access checks on subsequent calls.
3. **Use `MethodHandle`** (Java 7+, `java.lang.invoke`) — JIT-friendlier alternative.
4. **Use `VarHandle`** (Java 9+) — replaces `sun.misc.Unsafe` for many low-level ops.
5. **LambdaMetafactory** — generate functional interfaces from method handles for near-direct-call performance.

**`MethodHandle` is preferred** for new code — faster, safer, JVM-optimized.

---

## 35. Security Concerns with Reflection

Reflection can **bypass standard access controls**:
- Access `private` fields/methods.
- Modify `final` fields (in some cases).
- Instantiate classes without constructors (`Unsafe.allocateInstance`).

**Security risks:**
- **Leak sensitive data** — read private fields containing credentials.
- **Tamper with immutable state** — change a `final` field.
- **Bypass business logic** — invoke validation-less methods.
- **Break singletons** — create "second" instances.
- **Deserialization gadgets** — chain reflection calls in deserialized data for RCE.

**Historical safeguard — SecurityManager:**
- `ReflectPermission("suppressAccessChecks")` required for `setAccessible(true)`.
- **Deprecated in Java 17, removed in future releases** — the design was flawed.

**Modern safeguards (Java 9+ Module System):**
- **Strong encapsulation** — modules export/open packages explicitly.
- By default, reflection CANNOT access non-exported package internals, even with `setAccessible`.
- Use `--add-opens java.base/java.lang=ALL-UNNAMED` to open specific packages.
- `Module.isOpen(String packageName, Module otherModule)` — checks access permission.

**Best practices:**
- Avoid reflection unless absolutely necessary.
- When using it, **validate inputs** that drive reflective calls.
- Never deserialize untrusted data.
- Use the narrowest possible API (e.g., `MethodHandle` with lookup scope) instead of full reflection.

---

# 🔹 Annotations

## 36. Built-in Annotations

**`@Override`** (marker annotation)
- Indicates the annotated method overrides a superclass/interface method.
- Compile-time check — if the method doesn't actually override, compilation fails.
- Prevents bugs from typos and parameter mismatches.
```java
@Override
public String toString() { return "..."; }
```

**`@Deprecated`** (Java 9+ with elements)
- Marks API as outdated; generates compile warnings for callers.
- Java 9+ elements: `since` (version when deprecated), `forRemoval` (will be removed in future).
```java
@Deprecated(since = "5.1", forRemoval = true)
public void oldMethod() { ... }
```

**`@SuppressWarnings`**
- Tells compiler to ignore specific warnings.
- Common values: `"unchecked"`, `"rawtypes"`, `"deprecation"`, `"all"`.
- Should be applied as narrowly as possible.
```java
@SuppressWarnings("unchecked")
List<String> list = (List<String>) rawList;
```

**`@FunctionalInterface`**
- Declares intent that interface has exactly one abstract method.
- Compile-time check enforces single-abstract-method rule.

**`@SafeVarargs`**
- Suppresses unchecked warnings for methods/constructors with generic varargs.
- Promise to the compiler that you won't perform unsafe operations on the varargs array.
- Only valid on `final`, `static`, or `private` methods + constructors.

**Less common built-ins:**
- `@Native` — marks constants referenced by native code.
- `@SuppressFBWarnings` — for FindBugs/SpotBugs (third-party but common).

---

## 37. Custom Annotations

Annotations are **type declarations** you can attach metadata to program elements (classes, methods, fields, parameters, local variables, packages).

**Syntax:**
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Loggable {
    String level() default "INFO";
    boolean includeArgs() default true;
    String[] tags() default {};
}
```

**Usage:**
```java
@Loggable(level = "DEBUG", tags = {"perf", "audit"})
public void doSomething(String x) { ... }
```

**Allowed annotation element types:**
- Primitives and `String`.
- `Class<?>`.
- Enums.
- Other annotations.
- Arrays of the above.

**Default values** are optional.

**Special shortcuts:**
- If element is named `value`, callers can omit the name: `@Loggable("DEBUG")`.
- Single-element array: `@Loggable(tags = "x")` same as `{"x"}`.

**Meta-annotations (annotations on annotations):**

- **`@Retention`** — how long the annotation is retained:
  - `SOURCE` — discarded after compilation (e.g., `@Override`).
  - `CLASS` — in .class file but not loaded into JVM at runtime (default).
  - `RUNTIME` — available at runtime via reflection (required for frameworks).

- **`@Target`** — what program elements can be annotated:
  - `TYPE` (class/interface/enum), `FIELD`, `METHOD`, `PARAMETER`, `CONSTRUCTOR`, `LOCAL_VARIABLE`, `ANNOTATION_TYPE`, `PACKAGE`, `TYPE_PARAMETER`, `TYPE_USE` (Java 8+).

- **`@Inherited`** — if class A is annotated, subclasses inherit the annotation. Only applies to TYPE-level annotations.

- **`@Documented`** — include in Javadoc.

- **`@Repeatable`** — allow the annotation to be applied multiple times to the same element (Java 8+).

**Reading annotations at runtime:**
```java
Method m = clazz.getMethod("doSomething", String.class);
if (m.isAnnotationPresent(Loggable.class)) {
    Loggable ann = m.getAnnotation(Loggable.class);
    String level = ann.level();
    ...
}
```

---

## 38. Runtime vs Compile-time Annotations

Decides **when** the annotation is available.

**SOURCE retention:**
- Present in source code only.
- Discarded by compiler — not in .class file.
- Used by **annotation processors** to generate code or issue warnings at compile time.
- Examples: `@Override`, `@SuppressWarnings`, Lombok's `@Data`.

**CLASS retention (default):**
- Included in .class file.
- NOT loaded into memory at runtime — invisible to reflection.
- Used by bytecode manipulation tools, some static analyzers.

**RUNTIME retention:**
- Loaded into JVM and queryable via reflection.
- Drives most framework magic: Spring, JUnit, JPA, Jackson.

**Annotation processors (compile-time):**
`javax.annotation.processing.Processor` implementations run during compilation and can:
- Generate source/class files.
- Emit warnings/errors.
- Inspect the source code AST.

Examples: Lombok, Dagger (DI), MapStruct (mappers), AutoValue.

**Runtime annotations + reflection:**
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Cached { long ttlSeconds() default 60; }

// Framework side:
for (Method m : clazz.getDeclaredMethods()) {
    if (m.isAnnotationPresent(Cached.class)) {
        long ttl = m.getAnnotation(Cached.class).ttlSeconds();
        // wrap with caching logic via proxy
    }
}
```

---

# 🔹 IO & NIO

## 39. File Handling

Java has **two** file APIs:

**Legacy `java.io.File`:**
- Represents a path — may or may not refer to an actual file/directory.
- Limited: no proper error handling (methods return boolean), no symbolic link support, poor cross-platform behavior.
```java
File f = new File("data.txt");
f.exists(); f.canRead(); f.delete(); f.mkdirs();
```

**Modern `java.nio.file`** (Java 7+):
- **`Path`** — location (richer than File).
- **`Files`** — utility class with dozens of static methods.
- **`FileSystem`** — allows plugging in alternate filesystems (zip/jar as FS).
- Full exception-based error reporting.
- Symbolic link awareness.
- File attributes (POSIX permissions, creation time, etc.).

```java
Path p = Path.of("data.txt");
Files.exists(p);
Files.createDirectories(p);
Files.readAllLines(p, StandardCharsets.UTF_8);
Files.write(p, List.of("line1", "line2"));
Files.copy(src, dst, StandardCopyOption.REPLACE_EXISTING);
Files.walk(dir).filter(Files::isRegularFile).forEach(System.out::println);
```

**Use `java.nio.file` for new code.**

---

## 40. InputStream / OutputStream

**Byte streams** — the base abstraction for binary data. Abstract classes.

**Class hierarchy:**
```
InputStream                    OutputStream
├── FileInputStream            ├── FileOutputStream
├── ByteArrayInputStream       ├── ByteArrayOutputStream
├── FilterInputStream          ├── FilterOutputStream
│   ├── BufferedInputStream    │   ├── BufferedOutputStream
│   ├── DataInputStream        │   ├── DataOutputStream
│   └── PushbackInputStream    │   └── PrintStream
├── ObjectInputStream          ├── ObjectOutputStream
└── PipedInputStream           └── PipedOutputStream
```

**Key methods:**
- `InputStream`: `int read()`, `int read(byte[] b)`, `int read(byte[] b, int off, int len)`, `close()`.
- `OutputStream`: `write(int b)`, `write(byte[] b)`, `write(byte[] b, int off, int len)`, `flush()`, `close()`.

**Example:**
```java
try (InputStream in = new FileInputStream("in.bin");
     OutputStream out = new FileOutputStream("out.bin")) {
    byte[] buf = new byte[4096];
    int n;
    while ((n = in.read(buf)) != -1) {
        out.write(buf, 0, n);
    }
}
```

**Decorator pattern** — these streams illustrate the classic decorator pattern: you wrap a basic stream with decorators to add functionality (buffering, data typing, etc.).

---

## 41. Reader / Writer

**Character streams** — designed for text, handling encoding (bytes → characters) automatically.

**Why separate from byte streams?**
- Bytes are platform-independent, but **text is not** — each character is encoded as 1+ bytes (UTF-8, UTF-16, ISO-8859-1, etc.).
- Reader/Writer handle encoding transparently.

**Class hierarchy:**
```
Reader                         Writer
├── FileReader                 ├── FileWriter
├── CharArrayReader            ├── CharArrayWriter
├── StringReader               ├── StringWriter
├── BufferedReader             ├── BufferedWriter
├── InputStreamReader          ├── OutputStreamWriter
└── PrintWriter (Writer+)      └── PrintWriter
```

**Bridging byte & char streams:**
- **`InputStreamReader`** — wraps InputStream + specifies charset → Reader.
- **`OutputStreamWriter`** — wraps OutputStream + charset → Writer.

```java
Reader r = new InputStreamReader(new FileInputStream("in.txt"), StandardCharsets.UTF_8);
```

**Always specify charset explicitly** — defaulting to platform charset causes portability bugs.

**`FileReader` / `FileWriter` pre-Java 11 use the platform default charset** — prefer `Files.newBufferedReader(path, UTF_8)` instead. Java 11 added `FileReader(File, Charset)` constructors.

---

## 42. Buffered Streams

Wrap another stream to add an internal buffer. Reading/writing in chunks dramatically reduces the number of system calls.

**Without buffering:**
```java
FileInputStream in = new FileInputStream("big.txt"); // slow — reads byte by byte
int c;
while ((c = in.read()) != -1) { ... }
```

**With buffering:**
```java
BufferedInputStream in = new BufferedInputStream(new FileInputStream("big.txt"));
```
- Internal 8KB buffer by default (tunable).
- Underlying OS call happens once per buffer fill instead of once per byte.

**BufferedReader / BufferedWriter add features:**
- `BufferedReader.readLine()` — reads a line at a time.
- `BufferedWriter.newLine()` — platform-appropriate line separator.

```java
try (BufferedReader br = new BufferedReader(new FileReader("f.txt"))) {
    String line;
    while ((line = br.readLine()) != null) { System.out.println(line); }
}
```

**Modern (Files utility):**
```java
try (Stream<String> lines = Files.lines(Path.of("f.txt"))) {
    lines.forEach(System.out::println);
}
```

---

## 43. NIO Basics (Channels, Buffers)

**NIO (New I/O)** introduced in Java 1.4, upgraded in 7 (NIO.2). Core differences from java.io:

| Aspect | IO | NIO |
|--------|-----|-----|
| Orientation | Stream (one element at a time) | Buffer (chunks) |
| Blocking | Always blocking | Blocking or non-blocking |
| Operations | Read OR write per stream | Read AND write (channels are bidirectional) |
| Selectors | No | Yes (one thread, many channels) |

**Buffers:**
A Buffer is a block of memory (typically in native memory for direct buffers). Main types:
- `ByteBuffer`, `CharBuffer`, `ShortBuffer`, `IntBuffer`, `LongBuffer`, `FloatBuffer`, `DoubleBuffer`.

**Buffer state:**
- `capacity` — total size.
- `position` — next index to read/write.
- `limit` — end of data (for read) or writable space (for write).
- `mark` — remembered position (for `reset()`).

Invariant: `0 <= mark <= position <= limit <= capacity`.

**Typical buffer lifecycle:**
```java
ByteBuffer buf = ByteBuffer.allocate(1024); // position=0, limit=1024

channel.read(buf);   // writes into buffer; position advances
buf.flip();          // sets limit=position, position=0 — ready to read

while (buf.hasRemaining()) {
    process(buf.get());
}

buf.clear();         // reset for next write (position=0, limit=capacity)
```

**Direct vs Heap Buffers:**
- `ByteBuffer.allocate(n)` — heap buffer (backed by byte[]).
- `ByteBuffer.allocateDirect(n)` — allocated in native memory, faster for I/O (avoids JVM→OS copy) but more expensive to allocate.

**Channels:**
Bidirectional data conduits. Common implementations:
- `FileChannel` — for files.
- `SocketChannel`, `ServerSocketChannel` — TCP.
- `DatagramChannel` — UDP.
- `AsynchronousFileChannel`, `AsynchronousSocketChannel` — truly async NIO.2.

**Example:**
```java
try (FileChannel ch = FileChannel.open(Path.of("f.txt"), READ)) {
    ByteBuffer buf = ByteBuffer.allocate(1024);
    while (ch.read(buf) > 0) {
        buf.flip();
        // process
        buf.clear();
    }
}
```

**Selectors (non-blocking I/O):**
One thread can manage many channels:
```java
Selector selector = Selector.open();
channel.configureBlocking(false);
channel.register(selector, SelectionKey.OP_READ);

while (true) {
    selector.select();   // blocks until some channel is ready
    for (SelectionKey key : selector.selectedKeys()) {
        if (key.isReadable()) { /* read */ }
    }
    selector.selectedKeys().clear();
}
```

**Use case:** high-concurrency servers — Netty and Tomcat NIO connector are built on this.

**Memory-mapped files (mmap):**
```java
MappedByteBuffer map = fc.map(MapMode.READ_WRITE, 0, fc.size());
```
Maps file directly into memory — OS pages in/out automatically. Very fast for large files, used by Kafka and many DBs.

---

# 🔹 Generics

## 44. Generic Classes and Methods

**Generics** parameterize types over other types, giving compile-time type safety and eliminating casts.

**Before generics (Java 1.4):**
```java
List list = new ArrayList();
list.add("hello");
String s = (String) list.get(0);  // cast required; ClassCastException at runtime if wrong
```

**With generics:**
```java
List<String> list = new ArrayList<>();
list.add("hello");
String s = list.get(0);  // no cast; type checked at compile time
```

**Generic class:**
```java
public class Box<T> {
    private T value;
    public void set(T value) { this.value = value; }
    public T get() { return value; }
}
```

**Multiple type parameters:**
```java
public class Pair<K, V> {
    private final K key;
    private final V value;
    public Pair(K k, V v) { key = k; value = v; }
    public K getKey() { return key; }
    public V getValue() { return value; }
}
```

**Generic method:**
```java
public static <T> T firstOrNull(List<T> list) {
    return list.isEmpty() ? null : list.get(0);
}

// Usage (type usually inferred):
String s = firstOrNull(List.of("a", "b"));
```

**Diamond operator (Java 7+):**
```java
Map<String, List<Integer>> map = new HashMap<>();   // infers <String, List<Integer>>
```

**Naming conventions:**
- `T` — Type (generic)
- `E` — Element (in collections)
- `K`, `V` — Key, Value
- `N` — Number
- `R` — Return type
- `S`, `U`, `V` — additional type params

---

## 45. Type Erasure

**Core concept:** Java generics are a **compile-time** feature. At runtime, all type parameters are **erased** — replaced with their **upper bound** (or `Object` if unbounded).

```java
// What you write:
public class Box<T> { T item; }

// What the JVM sees (after compilation):
public class Box { Object item; }
```

**Bounded erasure:**
```java
public class NumberBox<T extends Number> { T item; }
// Erased to:
public class NumberBox { Number item; }
```

**Why type erasure?** **Backward compatibility.** Pre-generics `List` and generic `List<T>` must interoperate, which requires them to be the same runtime class.

**Consequences (important for interviews):**

1. **No new T()** — the type is unknown at runtime.
```java
public class Box<T> {
    T create() { return new T(); }  // COMPILE ERROR
}
```
**Workaround:** pass a `Class<T>` or `Supplier<T>`:
```java
T create(Class<T> type) throws Exception { return type.getDeclaredConstructor().newInstance(); }
```

2. **No generic arrays:**
```java
T[] arr = new T[10];  // COMPILE ERROR
List<String>[] lists = new List<String>[10];  // COMPILE ERROR
```
**Workaround:** use `Object[]` + unchecked cast, or `List<List<String>>`.

3. **`instanceof` with parameterized types forbidden:**
```java
if (obj instanceof List<String>)  // COMPILE ERROR
if (obj instanceof List<?>)       // OK (unbounded wildcard)
```

4. **Overloads by generic type DON'T work:**
```java
void m(List<String> x) { }
void m(List<Integer> x) { }  // COMPILE ERROR — both erase to List
```

5. **Bridge methods** — compiler synthesizes bridge methods to preserve polymorphism when generic subclass overrides a generic method. (You won't usually see them but they appear in stack traces.)

6. **Runtime reflection can SOMETIMES recover generic info** — generic info at class/method/field level is retained in metadata (but not for local variables). `Field.getGenericType()`, `Method.getGenericParameterTypes()`.

---

## 46. Bounded Types

Restrict type parameters to specific types or their subtypes.

**Upper bound with `extends`:**
```java
public class NumberBox<T extends Number> {
    T value;
    double asDouble() { return value.doubleValue(); } // Number methods available
}

NumberBox<Integer> ok1 = new NumberBox<>();   // OK
NumberBox<Double>  ok2 = new NumberBox<>();   // OK
NumberBox<String>  err = new NumberBox<>();   // COMPILE ERROR
```

**Multiple bounds:**
```java
public <T extends Number & Comparable<T>> T max(List<T> list) { ... }
```
- At most **one class** (must come first).
- Any number of **interfaces**.
- `& Comparable<T>` is a **type intersection** — T must be a subtype of BOTH Number AND Comparable.

**Why use bounded types?**
- Call specific methods on T (e.g., `Number.doubleValue()`).
- Restrict API to meaningful types.

**No lower bound on type parameters (`<T super Foo>` does NOT exist)** — lower bounds only on wildcards.

---

## 47. Wildcards (? extends, ? super)

Wildcards let you express **flexibility in generic types** — especially around variance.

**Problem motivating wildcards:**
```java
List<Integer> ints = new ArrayList<>();
List<Number> nums = ints;  // COMPILE ERROR
```
Even though `Integer extends Number`, `List<Integer>` is NOT a subtype of `List<Number>`. Generics are **invariant** by default.

**Wildcards provide variance:**

**Unbounded wildcard `<?>`** — unknown type, allows any:
```java
void printAll(List<?> list) { list.forEach(System.out::println); }
```
You can read from `List<?>` (elements come out as `Object`) but you cannot add anything (except `null`).

**Upper bounded `<? extends T>`** — T or any subtype. **Producer.**
```java
double sum(List<? extends Number> list) {
    double total = 0;
    for (Number n : list) total += n.doubleValue();  // read Numbers — OK
    return total;
}

sum(List.of(1, 2, 3));          // List<Integer> — OK
sum(List.of(1.5, 2.5));         // List<Double>  — OK
```
- You can **read** from it (each element is a `T`).
- You **cannot add** to it (compiler doesn't know the exact subtype) — except `null`.

**Lower bounded `<? super T>`** — T or any supertype. **Consumer.**
```java
void addIntegers(List<? super Integer> list) {
    list.add(1);  // OK — any supertype can hold an Integer
    list.add(42);
}

addIntegers(new ArrayList<Integer>()); // OK
addIntegers(new ArrayList<Number>());  // OK
addIntegers(new ArrayList<Object>());  // OK
```
- You can **add** a T (or its subtypes).
- You **cannot read** meaningfully (elements come out as `Object`).

**PECS mnemonic (Joshua Bloch):**
> **P**roducer **E**xtends, **C**onsumer **S**uper.

If a type parameter only **produces** values, use `? extends T`.
If it only **consumes** values, use `? super T`.
If it does both, use **exact** T.

**Real-world example — `Collections.copy`:**
```java
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
    for (T t : src) dest.add(t);
}
```
`src` produces T → `? extends T`. `dest` consumes T → `? super T`.

---

# 🔹 Immutable Objects

## 48. What Makes a Class Immutable

An **immutable object's state cannot change** after construction.

**Rules (Joshua Bloch's checklist):**

1. **Don't provide methods that mutate state** — no setters, no state-modifying methods.

2. **Make the class `final`** — prevents subclasses from adding mutability.
   (Alternative: make all constructors private and use static factories.)

3. **Make all fields `final`** — enforces single assignment, also provides JMM guarantees (safe publication across threads).

4. **Make all fields `private`** — prevents direct external access.

5. **Prevent exposure of mutable internal state** — for mutable field types (`Date`, collection, arrays), make **defensive copies** in the constructor AND in any getter.

**Example:**
```java
public final class Period {
    private final Date start;
    private final Date end;

    public Period(Date start, Date end) {
        // Defensive copy in constructor — caller can't mutate via retained reference
        this.start = new Date(start.getTime());
        this.end = new Date(end.getTime());
        if (this.start.after(this.end))
            throw new IllegalArgumentException();
    }

    public Date getStart() {
        // Defensive copy in getter — caller can't mutate internal state
        return new Date(start.getTime());
    }

    public Date getEnd() { return new Date(end.getTime()); }
}
```

**Do the validation AFTER the defensive copy** — otherwise a malicious caller could mutate the original between the check and the copy (TOCTOU bug).

**Modern alternatives:**
- **`java.time`** types (`LocalDate`, `Instant`) — already immutable, no defensive copy needed.
- **Collection.copyOf / List.copyOf** (Java 10+) — returns immutable copies.
- **Records (Java 16+)** — concise immutable data carriers:
```java
public record Period(LocalDate start, LocalDate end) {}
```
(Auto-generates constructor, getters, equals, hashCode, toString.)

---

## 49. Benefits of Immutability

**1. Thread-safety for free.**
Immutable objects need NO synchronization — multiple threads can share them without locks. Major concurrency win.

**2. Safe to share and cache.**
Since state never changes, you can:
- Cache instances (e.g., `Integer.valueOf()` caches -128 to 127).
- Reuse a single instance.
- Pass them around without defensive copies.

**3. Safe as HashMap keys and Set elements.**
A mutable key whose hashCode changes after being added to a map becomes undiscoverable. Immutable keys are guaranteed stable.

**4. Failure atomicity.**
An operation can either succeed fully or fail without leaving the object in a partial/broken state. Since immutable objects return NEW objects on "modification," the original remains unchanged on failure.

**5. Easier to reason about.**
No "who modified this when" debugging. Immutable state = pure functional style = simpler mental model.

**6. Security.**
Immutable objects can't be maliciously modified after validation. Critical for things like security tokens.

**Costs:**
- Creating new objects for every "change" — GC pressure.
- Mitigated by JIT (escape analysis) and persistent data structures (e.g., `PersistentHashMap` from Clojure, structural sharing).

**Famous immutable classes:**
- `String`, all boxed primitives (`Integer`, `Long`, etc.), `BigInteger`, `BigDecimal`.
- All `java.time` classes.
- `java.util.UUID`.
- Enum values.

---

# 🔹 Design Principles

## 50. SOLID Principles

**S — Single Responsibility Principle (SRP)**
> A class should have ONE reason to change.

Each class should focus on a single concern. If a class handles both user authentication AND database persistence AND email sending, any of those responsibilities changing forces modifications to the same class.

**Bad:**
```java
class User {
    void save() { /* DB logic */ }
    void sendEmail() { /* SMTP logic */ }
    void validate() { /* validation */ }
}
```

**Good:** split into `User`, `UserRepository`, `EmailService`, `UserValidator`.

---

**O — Open/Closed Principle (OCP)**
> Software entities should be OPEN for extension, CLOSED for modification.

You should be able to extend behavior without modifying existing code (which might break other clients). Typically achieved via polymorphism/interfaces.

**Bad:**
```java
double area(Shape s) {
    if (s instanceof Circle) return ...;
    if (s instanceof Square) return ...;
    // adding Triangle requires modifying this method
}
```

**Good:** `Shape` interface with `area()` method; each shape implements it. Adding a new shape doesn't touch existing code.

---

**L — Liskov Substitution Principle (LSP)**
> Subtypes must be substitutable for their base types without breaking the program's correctness.

A subclass must honor the contract of its superclass:
- Preconditions of overridden methods cannot be stronger.
- Postconditions cannot be weaker.
- Invariants must be preserved.
- No new exceptions should be thrown (unless superclass declared them).

**Classic violation — Rectangle/Square:**
```java
class Rectangle {
    void setWidth(int w) { ... }
    void setHeight(int h) { ... }
}
class Square extends Rectangle {
    void setWidth(int w) { super.setWidth(w); super.setHeight(w); }
    void setHeight(int h) { super.setWidth(h); super.setHeight(h); }
}
// A method `area(Rectangle r)` expecting width*height breaks if given a Square.
```

---

**I — Interface Segregation Principle (ISP)**
> Clients should not be forced to depend on interfaces they don't use.

Prefer many small, role-specific interfaces over one "fat" interface. Implementers shouldn't have to stub methods they don't need.

**Bad:**
```java
interface Worker {
    void work();
    void eat();
    void sleep();
}
class Robot implements Worker {
    void work() { ... }
    void eat() { throw new UnsupportedOperationException(); }  // BAD
    void sleep() { throw new UnsupportedOperationException(); }
}
```

**Good:** split into `Workable`, `Feedable`, `Sleepable` — robots implement only `Workable`.

---

**D — Dependency Inversion Principle (DIP)**
> Depend on abstractions, not concretions. High-level modules should not depend on low-level modules; both should depend on abstractions.

**Bad:**
```java
class OrderService {
    private MySqlOrderRepository repo = new MySqlOrderRepository();  // tightly coupled
}
```

**Good:**
```java
class OrderService {
    private final OrderRepository repo;  // interface
    public OrderService(OrderRepository repo) { this.repo = repo; }  // inject
}
```

DIP + interfaces → testability (inject mocks), flexibility (swap implementations), decoupling. Spring's DI is DIP in action.

---

## 51. DRY, KISS

**DRY — Don't Repeat Yourself**
> Every piece of knowledge should have a single, unambiguous, authoritative representation within a system.

Duplicated code means duplicated bugs and multiple places to update. Apply DRY to:
- **Code** — extract methods/classes.
- **Configuration** — single source of truth.
- **Documentation** — avoid duplicating info that can be generated.

**Careful:** premature DRY can be worse than duplication. If two similar methods serve **different conceptual purposes**, they should remain separate. "Rule of three" — wait until you see the pattern three times before abstracting.

**KISS — Keep It Simple, Stupid**
> The simplest solution is usually the best.

Avoid:
- Unnecessary abstractions.
- Over-engineered patterns.
- Speculative generality (building for imaginary future needs).

**Related — YAGNI (You Aren't Gonna Need It):** don't add functionality until you actually need it. Reduces code bloat and maintenance.

**In practice:** start with the simplest design that satisfies today's requirements. Refactor when real requirements pull you toward complexity.

---

## 52. High Cohesion vs Low Coupling

**Cohesion** — how closely related a module's responsibilities are.
- **High cohesion:** module does ONE thing well.
- **Low cohesion:** module is a grab-bag of unrelated features.

**Coupling** — degree of interdependence between modules.
- **Tight coupling:** changing one module ripples through many.
- **Loose coupling:** modules interact through well-defined, minimal interfaces.

**Goal: high cohesion + low coupling.** Results in:
- **Maintainability** — changes are localized.
- **Testability** — isolated modules are easy to test.
- **Reusability** — cohesive modules can be lifted into other projects.
- **Readability** — clear purpose for each module.

**Techniques to improve them:**
- **Dependency Injection** → lowers coupling.
- **Single Responsibility** → raises cohesion.
- **Interfaces/abstractions** → lowers coupling.
- **Event-driven / pub-sub** → lowers coupling further.

**Indicators of bad design:**
- Many classes know each other's internals (tight coupling).
- A single class has 20 fields spanning 5 unrelated concerns (low cohesion).
- Any small change touches a dozen files (tight coupling).

---

# 🔹 Object Class Methods

All classes in Java implicitly extend `java.lang.Object` (unless they explicitly extend something that does, which transitively does). Object defines 11 methods; these appear in every object.

## 53. toString()

Returns a String representation of the object. Default: `ClassName@hexHashCode`.

```java
public String toString() {
    return getClass().getName() + "@" + Integer.toHexString(hashCode());
}
```

**Why override:**
- Automatically called by `println`, `+` (string concat), `String.format`, debuggers, loggers.
- A good `toString` includes the class's most important fields.
- Makes debugging enormously easier.

```java
@Override
public String toString() {
    return "Employee{id=" + id + ", name='" + name + "', salary=" + salary + '}';
}
```

---

## 54. equals() & hashCode() (the contract)

Object's default `equals` returns true only for the SAME reference (==). Most classes override it for **logical equality**.

**`equals` contract (reflexive, symmetric, transitive, consistent, null-return):**
- `x.equals(x)` is true (reflexive).
- `x.equals(y) == y.equals(x)` (symmetric).
- If `x.equals(y)` and `y.equals(z)`, then `x.equals(z)` (transitive).
- Repeated calls return the same result (consistent, assuming no state changes).
- `x.equals(null)` is always false.

**`hashCode` contract:**
- If `x.equals(y)` then `x.hashCode() == y.hashCode()` (MANDATORY).
- If NOT equal, the hash codes don't HAVE to differ (but should, for good distribution).
- Consistent across calls (within one program execution).

**Standard implementation pattern:**
```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Employee)) return false;
    Employee e = (Employee) o;
    return id == e.id && Objects.equals(name, e.name);
}

@Override
public int hashCode() {
    return Objects.hash(id, name);  // helper since Java 7
}
```

**Violations cause real bugs:**
- Breaks `HashMap`, `HashSet` — entries become unfindable if hash differs from equal objects' hash.
- Breaks `List.contains`, `List.remove`, many collection algorithms.

**Important:** if you override `equals`, you MUST override `hashCode`. (IDE generators do both together.)

---

## 55. clone()

Creates a copy of the object. Requires the class to implement `Cloneable` (marker interface), else throws `CloneNotSupportedException`.

```java
class Foo implements Cloneable {
    int x; int[] arr;
    @Override
    public Foo clone() throws CloneNotSupportedException {
        return (Foo) super.clone();
    }
}
```

**What `super.clone()` does:**
- Field-by-field shallow copy.
- Does NOT call a constructor.
- Returns `Object` — must cast.

**Shallow vs Deep Copy:**
- **Shallow:** primitives copied by value, object references copied as-is (both original and clone point to same sub-objects).
- **Deep:** copies the entire object graph (recursively clones referenced objects).

```java
@Override
public Foo clone() throws CloneNotSupportedException {
    Foo copy = (Foo) super.clone();
    copy.arr = arr.clone();   // deep copy the array
    return copy;
}
```

**Problems with clone():**
- Breaks encapsulation (creates objects without constructors, potentially bypassing invariants).
- Cloneable is a broken marker interface (it doesn't declare clone()).
- Subclasses inherit `CloneNotSupportedException` awkwardness.
- Joshua Bloch: **"A fine approach to object copying is to provide a copy constructor or copy factory."**

**Preferred alternatives:**
```java
public Foo(Foo other) {          // copy constructor
    this.x = other.x;
    this.arr = other.arr.clone();
}

public static Foo copyOf(Foo other) { return new Foo(other); }  // copy factory
```

---

## 56. finalize() (deprecated since Java 9)

Called by the garbage collector before reclaiming an object's memory. Used historically for cleanup of non-memory resources.

**Why it's deprecated/removed:**
- **Not guaranteed to run** — GC may never collect an object before JVM exits.
- **Timing is unpredictable** — can run long after object is unreachable.
- **Performance cost** — adds a step to GC, slows down allocation AND reclamation.
- **Safety hazards** — can resurrect objects by storing `this` somewhere.
- **Security risks** — malicious subclasses can override finalize for attacks.

**Modern replacements:**
- **`try-with-resources`** — explicit, deterministic cleanup (preferred).
- **`AutoCloseable`** interface — implement `close()`.
- **`java.lang.ref.Cleaner`** (Java 9+) — a safer, last-ditch cleanup mechanism.

**Finalize may still appear in:**
- Legacy APIs.
- Interview trivia (know why it's bad).

---

## 57. getClass()

Returns the **runtime** `Class<?>` of this object. Entry point to reflection.

```java
Class<?> c = "hello".getClass();  // Class<String>
c.getName();                      // "java.lang.String"
c.getMethods();                   // all public methods
```

**Difference from `.class` literal:**
- `String.class` — compile-time, evaluates to `Class<String>`.
- `obj.getClass()` — runtime, returns actual class (important for polymorphism).

```java
Object o = new Integer(5);
o.getClass();   // class java.lang.Integer
Integer.class;  // class java.lang.Integer
```

**`final` in Object** — cannot be overridden.

---

## 58. wait(), notify(), notifyAll()

Low-level thread coordination. All three methods MUST be called while the thread holds the **monitor** of the object (i.e., inside a `synchronized` block on that object). Otherwise `IllegalMonitorStateException`.

**`wait()` / `wait(long ms)`** — current thread:
- Releases the monitor of this object.
- Enters the object's **wait set** (goes to WAITING or TIMED_WAITING state).
- Resumes when notified (and re-acquires the monitor).

**`notify()`** — wakes up ONE arbitrary thread waiting on this object's monitor. Awakened thread must still re-acquire the lock.

**`notifyAll()`** — wakes up ALL waiting threads. Each competes to re-acquire the lock.

**Classic producer-consumer skeleton:**
```java
synchronized void produce() throws InterruptedException {
    while (queue.isFull()) wait();  // always in a LOOP (spurious wakeups + condition changes)
    queue.add(item);
    notifyAll();
}
synchronized T consume() throws InterruptedException {
    while (queue.isEmpty()) wait();
    T item = queue.remove();
    notifyAll();
    return item;
}
```

**Why `while` not `if`?** Spurious wakeups are permitted by the JVM spec — a thread may wake up without a notify. And between wakeup and re-acquiring the lock, another thread could change the condition. Always re-check.

**Modern alternatives (preferred):**
- `java.util.concurrent.locks.Lock` + `Condition` — multiple condition variables per lock.
- `BlockingQueue` — handles producer-consumer without manual locking.
- `CountDownLatch`, `CyclicBarrier`, `Semaphore`, `Phaser`.

---

# 🔹 Thread Advanced Topics

## 59. Executor Framework

Introduced in Java 5 (`java.util.concurrent`), this framework **decouples task submission from thread management** — huge step up from manually creating `new Thread()`.

**Core interfaces:**
- **`Executor`** — single method: `execute(Runnable)`. Fires tasks.
- **`ExecutorService`** — extends `Executor`, adds `submit()`, `shutdown()`, `invokeAll()`, `invokeAny()`.
- **`ScheduledExecutorService`** — for delayed/periodic tasks.

**Standard factories (Executors class):**
```java
ExecutorService fixed   = Executors.newFixedThreadPool(8);
ExecutorService cached  = Executors.newCachedThreadPool();
ExecutorService single  = Executors.newSingleThreadExecutor();
ScheduledExecutorService sched = Executors.newScheduledThreadPool(2);
ExecutorService work    = Executors.newWorkStealingPool();  // Java 8+, ForkJoinPool
```

**Usage:**
```java
ExecutorService ex = Executors.newFixedThreadPool(4);
Future<String> f = ex.submit(() -> {
    Thread.sleep(1000);
    return "done";
});
String result = f.get();  // blocks until task finishes
ex.shutdown();           // orderly shutdown: no new tasks, existing finish
// ex.shutdownNow();     // attempts to interrupt running tasks
ex.awaitTermination(10, TimeUnit.SECONDS);
```

**Shutdown lifecycle:**
- `shutdown()` — stops accepting new tasks, finishes in-flight ones.
- `shutdownNow()` — attempts to interrupt running tasks; returns tasks that never started.
- `isShutdown()`, `isTerminated()` — state queries.
- **Always shutdown** — otherwise threads prevent JVM exit.

**Best practice (Java 8+):**
```java
try (ExecutorService ex = Executors.newFixedThreadPool(4)) {  // Java 19+ with try-with-resources
    // submit tasks
}
```

**Java 21+ Virtual Threads:** `Executors.newVirtualThreadPerTaskExecutor()` — create a fresh virtual thread per task, enabling massive concurrency.

---

## 60. Thread Pool

Reuses a limited set of worker threads for many tasks instead of creating a thread per task. Key benefits:
- Avoids thread creation overhead.
- Controls memory (each thread ~1MB stack).
- Bounds concurrency.
- Enables graceful shutdown.

**`ThreadPoolExecutor`** is the concrete class. Main parameters:
```java
new ThreadPoolExecutor(
    int corePoolSize,        // always-on threads
    int maximumPoolSize,     // max threads allowed
    long keepAliveTime,      // idle time before extra threads die
    TimeUnit unit,
    BlockingQueue<Runnable> workQueue,  // where tasks wait
    ThreadFactory threadFactory,        // how to create threads
    RejectedExecutionHandler handler    // what to do when saturated
);
```

**Execution policy:**
1. If fewer than `corePoolSize` threads running → create a new thread.
2. Else if queue has space → enqueue.
3. Else if less than `maximumPoolSize` threads → create a new thread.
4. Else → invoke `RejectedExecutionHandler`.

**Rejection policies (`RejectedExecutionHandler`):**
- `AbortPolicy` (default) — throws `RejectedExecutionException`.
- `CallerRunsPolicy` — caller thread runs the task (backpressure).
- `DiscardPolicy` — silently drops.
- `DiscardOldestPolicy` — drops the oldest queued task, retries.

**Queue choices:**
- `LinkedBlockingQueue` — unbounded (careful, can OOM).
- `ArrayBlockingQueue` — bounded, FIFO.
- `SynchronousQueue` — no storage; producer must meet consumer (used by `newCachedThreadPool`).

**Sizing rules:**
- **CPU-bound tasks:** `N+1` threads (N = cores), so one can run while others swap.
- **IO-bound tasks:** `N × (1 + wait/compute)`. Often 2N to 100N+ for blocking-heavy code.

**Pitfalls:**
- `newFixedThreadPool` uses unbounded `LinkedBlockingQueue` — memory leak if tasks produced faster than consumed.
- `newCachedThreadPool` uses unbounded thread count — can DoS the JVM.
- Custom `ThreadPoolExecutor` with bounded queue is usually safer.

---

## 61. Callable & Future

**`Runnable`** — `void run()`, no return, cannot throw checked exceptions.

**`Callable<V>`** — `V call() throws Exception`, returns V, can throw checked exceptions.

```java
Callable<Integer> task = () -> {
    Thread.sleep(1000);
    return 42;
};
Future<Integer> f = executor.submit(task);
Integer result = f.get();  // blocks until complete
```

**`Future<V>`** — handle to an asynchronous computation:
- `V get()` — blocks until done.
- `V get(long timeout, TimeUnit unit)` — blocks up to timeout.
- `boolean cancel(boolean mayInterruptIfRunning)`.
- `boolean isDone()`.
- `boolean isCancelled()`.

**Limitations of Future:**
- Can't **compose** multiple Futures.
- Can't react to completion (must block or poll).
- No exception handling callbacks.

**`CompletableFuture` (Java 8+)** — solves those limitations:
```java
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> fetchUser(id))
    .thenApply(user -> user.getEmail())
    .thenCompose(email -> fetchMessages(email))
    .thenAccept(messages -> display(messages))
    .exceptionally(ex -> { log(ex); return null; });
```

**Composition methods:**
- `thenApply(Function)` — transform value.
- `thenAccept(Consumer)` — consume value.
- `thenRun(Runnable)` — no-arg follow-up.
- `thenCompose(Function)` — flat-map (chain async).
- `thenCombine(CompletableFuture, BiFunction)` — combine two.
- `allOf(...)`, `anyOf(...)` — aggregate many.
- `exceptionally(Function)`, `handle(BiFunction)` — error handling.

**CompletableFuture is the modern way** to do async in Java 8+.

---

## 62. ForkJoinPool

A specialized pool for **divide-and-conquer** parallelism. Key innovation: **work-stealing**.

**Model:**
- Each worker thread has its own deque of tasks.
- When a task splits into sub-tasks, children go on the local deque.
- When a worker's deque is empty, it **steals** tasks from another worker's deque.
- This balances load automatically and minimizes contention.

**Core classes:**
- `ForkJoinPool` — the pool.
- `RecursiveTask<V>` — divide-and-conquer task returning V.
- `RecursiveAction` — same but void return.
- `ForkJoinTask<V>` — common superclass.

**Example — parallel sum:**
```java
class SumTask extends RecursiveTask<Long> {
    final long[] arr; final int lo, hi;
    static final int THRESHOLD = 1000;
    SumTask(long[] arr, int lo, int hi) { ... }

    @Override
    protected Long compute() {
        if (hi - lo <= THRESHOLD) {
            long s = 0;
            for (int i = lo; i < hi; i++) s += arr[i];
            return s;
        }
        int mid = (lo + hi) >>> 1;
        SumTask left = new SumTask(arr, lo, mid);
        SumTask right = new SumTask(arr, mid, hi);
        left.fork();            // async compute on another thread
        long rightResult = right.compute();  // compute locally
        long leftResult = left.join();        // wait for left result
        return leftResult + rightResult;
    }
}

long total = ForkJoinPool.commonPool().invoke(new SumTask(data, 0, data.length));
```

**Common Pool** — `ForkJoinPool.commonPool()` is a shared pool used by `parallelStream()`, `CompletableFuture` with default executor, and any code that doesn't provide one. Size = `#cores - 1` by default.

**Beware:** one long-running task in common pool can starve other parallel operations JVM-wide. Use your own `ForkJoinPool` for isolation.

---

## 63. BlockingQueue

A **thread-safe queue** that blocks callers when operations can't proceed.

**Operations (4 flavors each):**

| Operation | Throws | Returns special | Blocks | Times out |
|-----------|--------|-----------------|--------|-----------|
| Insert | `add(e)` | `offer(e)` | `put(e)` | `offer(e, time, unit)` |
| Remove | `remove()` | `poll()` | `take()` | `poll(time, unit)` |
| Examine | `element()` | `peek()` | N/A | N/A |

**Implementations:**
- **`ArrayBlockingQueue`** — bounded, array-backed, FIFO. Single lock (fair option). Predictable memory.
- **`LinkedBlockingQueue`** — optionally bounded, linked-node-backed. Two locks (put/take), higher throughput.
- **`PriorityBlockingQueue`** — unbounded, priority-ordered (via Comparator).
- **`DelayQueue`** — unbounded, elements available only when delay expires.
- **`SynchronousQueue`** — zero capacity; each put waits for a take. Used for handoff.
- **`LinkedTransferQueue`** — like SynchronousQueue + LinkedBlockingQueue hybrid.
- **`LinkedBlockingDeque`** — double-ended.

**Classic producer-consumer:**
```java
BlockingQueue<Task> queue = new ArrayBlockingQueue<>(100);

// Producer
Runnable producer = () -> {
    while (true) queue.put(generateTask());  // blocks if full
};
// Consumer
Runnable consumer = () -> {
    while (true) process(queue.take());       // blocks if empty
};
```

Eliminates `wait/notify` boilerplate and handles backpressure automatically.

---

## 64. Producer-Consumer Problem

**Setup:** One or more producers generate items; one or more consumers process them. Producers and consumers run at different rates. Solution must handle **bounded buffer** (prevent OOM) and **coordination** (don't consume empty, don't produce when full).

**Solution 1 — BlockingQueue (idiomatic):**
```java
BlockingQueue<Task> queue = new LinkedBlockingQueue<>(1000);

// Producer
for (int i = 0; i < N; i++) {
    new Thread(() -> {
        while (!done) queue.put(produce());
    }).start();
}
// Consumer
for (int i = 0; i < M; i++) {
    new Thread(() -> {
        while (!done) consume(queue.take());
    }).start();
}
```

**Solution 2 — wait/notify (educational):**
```java
class BoundedBuffer<T> {
    private final Queue<T> buf = new LinkedList<>();
    private final int capacity;
    public BoundedBuffer(int cap) { this.capacity = cap; }

    public synchronized void put(T item) throws InterruptedException {
        while (buf.size() == capacity) wait();
        buf.add(item);
        notifyAll();
    }
    public synchronized T take() throws InterruptedException {
        while (buf.isEmpty()) wait();
        T item = buf.poll();
        notifyAll();
        return item;
    }
}
```

**Solution 3 — Lock + Condition (more control):**
```java
Lock lock = new ReentrantLock();
Condition notFull = lock.newCondition();
Condition notEmpty = lock.newCondition();

void put(T t) throws InterruptedException {
    lock.lock();
    try {
        while (buf.size() == capacity) notFull.await();
        buf.add(t);
        notEmpty.signal();
    } finally { lock.unlock(); }
}
```
Benefit: separate waiting conditions (producers wake only on notFull, consumers only on notEmpty) → fewer spurious wakeups.

**Real-world use:** message queues, logging, async tasks, event buses.

---

# 🔹 Date & Time API

## 65. LocalDate, LocalDateTime

Java 8's `java.time` package (JSR-310) replaced the broken legacy `Date`/`Calendar` with a clean, immutable, type-safe API inspired by Joda-Time.

**Main types:**

| Class | Represents | Example |
|-------|-----------|---------|
| `LocalDate` | Date (no time, no zone) | 2024-11-23 |
| `LocalTime` | Time (no date, no zone) | 14:30:15.123 |
| `LocalDateTime` | Date + Time (no zone) | 2024-11-23T14:30:15 |
| `ZonedDateTime` | Date + Time + Zone | 2024-11-23T14:30+05:30 Asia/Kolkata |
| `OffsetDateTime` | Date + Time + Offset (no region) | 2024-11-23T14:30+05:30 |
| `Instant` | Nanos since epoch (UTC) | 1700749815.123456789 |
| `Duration` | Time-based amount | PT2H30M (2h 30m) |
| `Period` | Date-based amount | P1Y2M3D (1y 2m 3d) |
| `Year`, `YearMonth`, `MonthDay` | Partial dates | 2024, 2024-11, --11-23 |

**Key features:**
- **All immutable** — thread-safe, every "modification" returns a new instance.
- **Fluent API** — readable method chaining.
- **No nulls** — uses `Optional` or throws explicit exceptions.
- **Type safety** — you can't accidentally mix dates and times.

**Common operations:**
```java
LocalDate today = LocalDate.now();
LocalDate birthday = LocalDate.of(1990, 6, 15);
LocalDate parsed = LocalDate.parse("2024-11-23");

LocalDate tomorrow = today.plusDays(1);
LocalDate lastMonth = today.minusMonths(1);
LocalDate firstOfMonth = today.withDayOfMonth(1);

boolean isLeap = today.isLeapYear();
Month month = today.getMonth();
DayOfWeek dow = today.getDayOfWeek();

Period between = Period.between(birthday, today);  // age
int years = between.getYears();
```

**Formatting/parsing:**
```java
DateTimeFormatter fmt = DateTimeFormatter.ofPattern("dd/MM/yyyy");
String s = today.format(fmt);                  // "23/11/2024"
LocalDate d = LocalDate.parse("23/11/2024", fmt);

// Locale-aware
DateTimeFormatter full = DateTimeFormatter.ofLocalizedDate(FormatStyle.FULL)
    .withLocale(Locale.FRENCH);
```

---

## 66. Old Date vs New API

| Aspect | `java.util.Date` / `Calendar` | `java.time` |
|--------|-----------|-------------|
| **Mutability** | Mutable → not thread-safe | Immutable → thread-safe |
| **Nullability** | nulls common | Avoided |
| **Month indexing** | 0-based (January = 0) 😱 | 1-based (January = 1) |
| **Year** | `Date.getYear()` returns year - 1900 | Actual year |
| **Timezone** | Poor, scattered across classes | First-class `ZoneId`/`ZoneOffset` |
| **Design** | One class does everything | Type per concept |
| **Arithmetic** | Cumbersome (`Calendar.add()`) | Fluent (`date.plusDays(3)`) |
| **Parsing** | `SimpleDateFormat` (not thread-safe!) | `DateTimeFormatter` (immutable, thread-safe) |
| **Formatting** | Limited | Rich, localized |
| **Epoch support** | Basic `getTime()` | `Instant` with nanosecond precision |

**Legacy gotchas to know for interviews:**

1. **`SimpleDateFormat` is NOT thread-safe.** Common bug: shared static formatter causes weird parsing results under load.

2. **`Date` represents an instant, NOT a date.** The name lies — it's milliseconds since epoch.

3. **`Calendar.getInstance()` is locale-sensitive** — first day of week, month naming differ.

**Interop:**
```java
Date legacyDate = Date.from(instant);
Instant i = legacyDate.toInstant();

LocalDateTime ldt = LocalDateTime.ofInstant(instant, ZoneId.systemDefault());
Instant i2 = ldt.atZone(ZoneId.systemDefault()).toInstant();
```

**Rule:** for new code, use `java.time`. Wrap legacy APIs at boundaries.

---

## 67. Time Zones

**`ZoneId`** — a region like `Asia/Kolkata`, `America/New_York`. Handles DST.
**`ZoneOffset`** — a fixed offset from UTC like `+05:30`. No DST.

```java
ZoneId kolkata = ZoneId.of("Asia/Kolkata");
ZoneOffset utc = ZoneOffset.UTC;
ZoneId system = ZoneId.systemDefault();
Set<String> allZones = ZoneId.getAvailableZoneIds();
```

**`ZonedDateTime`** — full moment in time:
```java
ZonedDateTime now = ZonedDateTime.now(kolkata);
ZonedDateTime sameInstantNY = now.withZoneSameInstant(ZoneId.of("America/New_York"));
ZonedDateTime sameLocal = now.withZoneSameLocal(ZoneId.of("America/New_York"));
```
- `withZoneSameInstant` — same absolute moment, different display zone.
- `withZoneSameLocal` — same wall-clock time, different zone (shifts the instant).

**`Instant`** — machine-readable timestamp in UTC. Best format for storage/interchange.
```java
Instant now = Instant.now();
long epochSec = now.getEpochSecond();
```

**Best practices for application design:**
- **Store as UTC `Instant`** (or `TIMESTAMP WITH TIME ZONE` in DB).
- **Convert to user's zone** only for display.
- **Don't mix** — always know whether a variable is a zoned datetime or an instant.
- **Be explicit about zone in logs** — "3 PM" is ambiguous; "2024-11-23T15:00+05:30" is not.

**DST issues:**
- Times "spring forward" can be non-existent (e.g., 2:30 AM on DST day).
- Times "fall back" can be ambiguous (same wall time appears twice).
- `ZonedDateTime` handles these via resolution strategy; `Instant` sidesteps entirely.

---

# 🔹 Class Design Concepts

## 68. Immutable vs Mutable Classes

**Mutable** — state can change after construction. Default for most domain classes.
```java
class Counter { private int count = 0; public void inc() { count++; } }
```

**Immutable** — state fixed at construction. See section 48 for full details.

**When to choose immutable:**
- Value objects (`Money`, `Address`, `Point`).
- Small objects used as keys or in concurrent code.
- DTOs / API response objects.

**When mutable is OK:**
- Entities with identity (e.g., JPA `User` with DB id).
- Large objects where "mutation" means expensive field updates.
- Stateful services (connection pools, caches).

**Java practice:**
- Make classes immutable unless there's a concrete reason not to (Joshua Bloch, Item 17).
- Use Java records (Java 16+) for quick immutable data classes.

**Hybrid — builder pattern for constructing immutable objects:**
```java
Pizza p = new Pizza.Builder()
    .addTopping(CHEESE)
    .size(LARGE)
    .build();
```

---

## 69. Singleton Pattern

A class that guarantees **exactly one instance** per JVM, with global access.

**Use cases:** logging, configuration, connection pool, thread pool.

**Implementation approaches (ranked best to worst):**

**1. Enum Singleton (Joshua Bloch's recommendation):**
```java
public enum Logger {
    INSTANCE;
    public void log(String msg) { ... }
}
// Usage: Logger.INSTANCE.log("hi");
```
- ✅ Thread-safe (JVM-guaranteed).
- ✅ Serialization-safe (Enum handling).
- ✅ Reflection-safe (can't newInstance an enum).
- ✅ Simplest code.
- ❌ Can't lazy-init (rarely an issue).

**2. Bill Pugh / Holder Idiom (lazy + thread-safe without synchronization):**
```java
public class Singleton {
    private Singleton() { }
    private static class Holder {
        static final Singleton INSTANCE = new Singleton();
    }
    public static Singleton getInstance() { return Holder.INSTANCE; }
}
```
- ✅ Lazy (Holder class not loaded until getInstance() called).
- ✅ Thread-safe (class loading is atomic).
- ✅ No synchronization overhead.
- ❌ Vulnerable to reflection and serialization attacks.

**3. Double-Checked Locking (DCL):**
```java
public class Singleton {
    private static volatile Singleton instance;   // volatile CRITICAL
    private Singleton() { }
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) instance = new Singleton();
            }
        }
        return instance;
    }
}
```
- ✅ Lazy and thread-safe (with volatile since Java 5).
- ❌ Verbose, easy to get wrong, less safe than Bill Pugh.

**4. Eager Initialization:**
```java
public class Singleton {
    private static final Singleton INSTANCE = new Singleton();
    private Singleton() { }
    public static Singleton getInstance() { return INSTANCE; }
}
```
- ✅ Simple, thread-safe.
- ❌ Not lazy — instance created at class load even if unused.

**5. Synchronized method (simple but slow):**
```java
public static synchronized Singleton getInstance() {
    if (instance == null) instance = new Singleton();
    return instance;
}
```
- ❌ Every call takes the lock, even after init — performance hit.

**Pitfalls across most implementations:**
- **Reflection attack** — `setAccessible(true)` + call private constructor. Guard by throwing from constructor if INSTANCE already set.
- **Serialization** — default behavior creates a new instance. Add `readResolve` returning the singleton.
- **Multiple class loaders** — each loader has its own "singleton." App servers can create multiple instances.
- **Cloning** — override `clone()` to throw.

**Enum avoids ALL of these automatically.**

---

## 70. Factory Pattern

Encapsulates object creation, returning objects of some type without exposing instantiation logic.

**Variants:**

**1. Simple Factory (method):**
```java
public class ShapeFactory {
    public static Shape create(String type) {
        return switch (type) {
            case "CIRCLE" -> new Circle();
            case "SQUARE" -> new Square();
            default -> throw new IllegalArgumentException();
        };
    }
}
```

**2. Factory Method Pattern — subclasses decide:**
```java
abstract class Creator {
    public abstract Product createProduct();  // factory method
    public void use() {
        Product p = createProduct();
        p.doSomething();
    }
}
class ConcreteCreator extends Creator {
    public Product createProduct() { return new ConcreteProduct(); }
}
```

**3. Abstract Factory — families of related objects:**
```java
interface UIFactory {
    Button createButton();
    Checkbox createCheckbox();
}
class MacUIFactory implements UIFactory { ... }
class WindowsUIFactory implements UIFactory { ... }
```

**Benefits:**
- **Encapsulates complex creation logic** (constructors can't have meaningful names).
- **Hides concrete types** — client depends on interface.
- **Enables caching** (factory can return a cached instance).
- **Enables subtype selection** based on context.
- **Simplifies testing** (inject mock factory).

**Related idioms:**
- **Static factory methods** — `Integer.valueOf`, `List.of`, `Optional.of`. Benefits:
  - Have names (unlike constructors).
  - Can return subtypes.
  - Can return cached/existing objects.
  - Can return any subtype determined at runtime.
- **Builder** — for objects with many optional parameters.
- **Prototype** — clone an existing instance.

---

# 🔹 Misc Frequently Asked

## 71. final, finally, finalize

Three unrelated things that share a name stem.

**`final`** (keyword):
- **Final class** — cannot be subclassed. `final class String {}`.
- **Final method** — cannot be overridden.
- **Final variable** — cannot be reassigned after initialization (for object references, the reference is fixed, but the object's state may still mutate unless the object itself is immutable).
- **Effectively final** — local variable that could be `final` based on usage (lambdas require effectively final captures).

```java
final int x = 5;  // reassignment forbidden
final int[] arr = {1, 2};
arr[0] = 99;      // OK — reference is final, array contents are not
arr = new int[5]; // ERROR
```

**`finally`** (block):
- Always executes after try/catch (except in System.exit or JVM crash cases).
- Used for resource cleanup, logging, state restoration.
- See section 8 for quirks.

**`finalize()`** (method):
- From `Object`, called by GC before reclaiming object memory.
- Deprecated since Java 9, removed in future versions.
- See section 56.

**Memory aid:** final → variable/method/class modifier; finally → try-catch-finally block; finalize → method on Object.

---

## 72. transient vs volatile

Completely different purposes — both modify how a field is treated.

**`transient`** — serialization modifier:
- Marks a field to be **excluded** from default serialization.
- Upon deserialization, field has its default value (null, 0, false).
- Common for sensitive data, derived data, non-serializable references.
- See section 26.

**`volatile`** — concurrency modifier:
- Guarantees **visibility** of writes across threads — a write on one thread is immediately visible to reads on others.
- Prevents the compiler/CPU from caching the field in a register or CPU cache.
- Establishes **happens-before** relationships (Java Memory Model).
- Does NOT guarantee atomicity of compound operations (`count++` is read + increment + write).

**volatile example — common use:**
```java
class StopFlag {
    private volatile boolean stop = false;  // volatile ensures visibility
    public void stop() { stop = true; }
    public void run() {
        while (!stop) { /* work */ }        // sees update promptly
    }
}
```

**When volatile is NOT enough — atomicity needed:**
```java
private volatile int counter;
// counter++ is NOT atomic — use AtomicInteger instead
```

**When volatile IS sufficient:**
- Boolean flags.
- Single reads/writes where visibility is the only concern.
- Pointer/reference swapping where the referenced object is already immutable.

**Key distinction:**
- `transient` → "don't persist this."
- `volatile` → "don't cache this across threads."

---

## 73. instanceof

Operator that tests whether an object is an instance of a type.

```java
Object o = "hello";
if (o instanceof String) {
    String s = (String) o;
    // use s
}
```

**Key properties:**
- Returns `false` for `null` (no NullPointerException).
- Works with interface types: `o instanceof Comparable`.
- Works with superclass types: `o instanceof Object` (true for any non-null).
- Parameterized types NOT allowed: `o instanceof List<String>` → compile error. Use `o instanceof List<?>`.

**Java 16+ Pattern Matching for instanceof:**
```java
if (o instanceof String s) {   // binds s with correct type
    System.out.println(s.length());
}
```
- Eliminates the cast.
- `s` is in scope in the `if` body (and optionally the else if the compiler can prove the type — flow-sensitive scoping).

**Java 21 Pattern Matching in switch:**
```java
String description = switch (obj) {
    case Integer i -> "int " + i;
    case String s when s.isEmpty() -> "empty string";
    case String s -> "string " + s;
    case null -> "null";
    default -> "unknown";
};
```

**Overuse is a smell:** if you find yourself using many `instanceof` checks, consider polymorphism instead — call a virtual method on the object rather than switching on its type.

---

## 74. Marker Interfaces

An interface with **no methods or constants**, used purely to **signal/tag** that a class has a property the JVM or a framework should check.

**Built-in examples:**
- `java.io.Serializable` — "this class can be serialized."
- `java.lang.Cloneable` — "Object.clone() should work on this."
- `java.util.RandomAccess` — "this List supports constant-time random access."
- `java.rmi.Remote` — "this object is remotely invocable."

**Use:**
```java
public class Foo implements Serializable { }

// Later:
if (obj instanceof Serializable) { ... }
```

**How the JVM or frameworks use them:**
- `ObjectOutputStream.writeObject` checks if the object implements `Serializable`, else `NotSerializableException`.
- `Collections.shuffle` uses `RandomAccess` to pick a different algorithm.

**Modern alternative: annotations.**
Marker interfaces are a pre-Java 5 technique. Today, an annotation is usually more appropriate:
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Loggable { }
```

**When to prefer marker interfaces over annotations:**
1. You want to define a **type** that polymorphism can exploit (can declare a method parameter `Serializable x`).
2. You want **compile-time checks** tied to a type hierarchy.
3. You want `instanceof` compatibility.

**When to prefer annotations:**
- Need to target non-class elements (methods, fields).
- Want to carry **parameters** (marker interfaces can't).
- Want **finer-grained** scope than a type-wide marker.

**Interview trap:** `Serializable` is a marker, but `Externalizable` is NOT — it has two real methods (`readExternal`, `writeExternal`).

---

# End of Notes
