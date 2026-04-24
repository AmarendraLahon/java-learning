# Week 2 — Exception Handling & Generics

**Theme:** Two orthogonal topics that both appear on nearly every senior-level Java interview. Exception-handling questions probe your sense of code correctness and resource safety. Generics questions test whether you actually understand what the compiler is doing.

---

## Monday — Exception Hierarchy, Checked vs Unchecked, Error vs Exception

### 🎯 Objective
Draw the `Throwable` hierarchy from memory and explain when to use checked vs unchecked exceptions.

### 📖 Core Concept

```
Throwable
├── Error                         (serious, usually unrecoverable)
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   └── VirtualMachineError
└── Exception
    ├── IOException               (checked)
    ├── SQLException              (checked)
    ├── ClassNotFoundException    (checked)
    └── RuntimeException          (UNCHECKED)
        ├── NullPointerException
        ├── IllegalArgumentException
        ├── ClassCastException
        └── IndexOutOfBoundsException
```

**Checked exceptions** — subclass of `Exception` but NOT `RuntimeException`. Compiler enforces handling (catch or declare). Model **recoverable** conditions the caller could reasonably handle (file missing, network down).

**Unchecked exceptions** — subclasses of `RuntimeException`. No compiler enforcement. Model **programming errors** (null passed where not allowed, wrong type cast).

**Errors** — subclass of `Error`. JVM-level problems. Don't catch them — the JVM is likely broken.

**Joshua Bloch's guidance (Effective Java, Item 70):**
- Use **checked** for conditions that a well-written caller should handle.
- Use **runtime** for programming errors (preconditions violated).
- Never catch `Error` in application code.

### 💻 Code Example

```java
// Checked — caller must handle
public Customer loadCustomer(String id) throws IOException, SQLException {
    ...
}

// Unchecked — programming error
public Customer findById(String id) {
    Objects.requireNonNull(id, "id");  // throws NullPointerException
    if (id.isBlank())
        throw new IllegalArgumentException("id is blank");
    return repo.get(id);
}

// Application domain exception — usually unchecked in modern code
public class InsufficientBalanceException extends RuntimeException {
    private final BigDecimal available;
    private final BigDecimal requested;
    public InsufficientBalanceException(BigDecimal available, BigDecimal requested) {
        super("Need " + requested + ", have " + available);
        this.available = available;
        this.requested = requested;
    }
    public BigDecimal available() { return available; }
    public BigDecimal requested() { return requested; }
}
```

### ⚠️ Common Pitfalls
- Catching `Throwable` or `Exception` broadly — hides bugs, makes diagnosis impossible.
- Wrapping a checked exception in a `RuntimeException` without preserving the cause: `throw new RuntimeException("oops")` → lost stack trace. Always pass the cause: `throw new RuntimeException("oops", e)`.
- Swallowing exceptions with an empty `catch` block — leads to silent failures.

### 🎤 Interview Angle
> "Why do modern frameworks (Spring, Hibernate) throw unchecked exceptions?"

Checked exceptions don't compose well with lambdas and streams. They also pollute method signatures across layers, encouraging either declaring `throws Exception` everywhere or catch-and-swallow. Unchecked exceptions give callers the choice to handle at the layer where handling makes sense, typically a controller advice/global handler.

### 🧪 Practice
1. Draw the `Throwable` hierarchy on paper including 5 subclasses of each branch.
2. Write a `BankAccount.withdraw()` that throws `InsufficientBalanceException` (unchecked) carrying context.
3. Demonstrate wrapping: catch an `IOException`, wrap as a domain `DataAccessException`, preserve the cause.

### 🔍 Self-check
1. Is `NullPointerException` checked or unchecked?
2. If a superclass method throws `IOException`, can a subclass override it to throw `Exception`?
3. Why is it wrong to catch `Throwable`?
4. What happens if you don't preserve `cause` when wrapping an exception?

---

## Tuesday — try-catch-finally, Multi-catch, `throw` vs `throws`

### 🎯 Objective
Predict the output of tricky try-catch-finally snippets (return overrides, exception masking); know multi-catch semantics.

### 📖 Core Concept

**Execution order:**
1. `try` block runs until it completes OR throws.
2. On throw, the JVM looks for the first matching `catch` (ordered **specific-first**).
3. Whether or not a catch matched, `finally` runs.
4. If neither `try` nor `catch` threw, control leaves normally.

**`finally` runs except:**
- `System.exit(n)` was called.
- The JVM crashed (`Runtime.halt`, native crash).
- The thread was killed (`Thread.stop`, deprecated).
- Infinite loop / deadlock in try or catch.

**Multi-catch (Java 7+):**
```java
try { ... }
catch (IOException | SQLException e) { log(e); }
```
- The variable `e` is **effectively final** (can't reassign).
- Compiler treats `e` as the common supertype.

**`throw`** — statement that raises an exception. `throws` — method-signature declaration that the method may propagate the listed checked exceptions.

### 💻 Code Example — the classics

```java
// (1) return in finally overrides try's return
int m1() {
    try { return 1; }
    finally { return 2; }
}
// m1() → 2

// (2) throw in finally masks original exception
void m2() {
    try { throw new RuntimeException("A"); }
    finally { throw new RuntimeException("B"); }
}
// caller sees "B"; "A" is LOST (not even suppressed)

// (3) primitive return captured before finally modifies
int m3() {
    int x = 1;
    try { return x; }
    finally { x = 99; }       // too late — return value was snapshotted
}
// m3() → 1

// (4) mutable return value CAN be modified
StringBuilder m4() {
    StringBuilder sb = new StringBuilder("a");
    try { return sb; }
    finally { sb.append("b"); }
}
// m4().toString() → "ab"
```

### ⚠️ Common Pitfalls
- Never put `return` or `throw` in `finally` — masks exceptions and makes code impossible to reason about.
- Ordering catch blocks wrong: `catch (Exception e)` before `catch (IOException e)` → compile error (unreachable).
- Using multi-catch to reassign `e` — compile error (it's effectively final in this form).

### 🎤 Interview Angle
> "What prints?" with a combination of try/catch/finally and returns. The trick is knowing primitive-value snapshot vs reference mutability, and that finally's return/throw overrides.

### 🧪 Practice
1. Write every one of the 4 classics above and confirm the output.
2. Create a method that logs but doesn't swallow: `catch(Exception e) { log.error(e); throw e; }` — then explain why the `throw e;` type-checks even if `e` is declared as `Exception`.
3. Implement retry-with-backoff that catches only transient exceptions, re-throws the rest.

### 🔍 Self-check
1. Does `finally` run if `System.exit(0)` is called in `try`?
2. What happens to the primary exception if `finally` throws?
3. Why is `e` effectively final in multi-catch?
4. What's the difference between `throw` and `throws`?

---

## Wednesday — Custom Exceptions, try-with-resources, Suppressed Exceptions

### 🎯 Objective
Write a production-quality custom exception; use try-with-resources correctly and understand suppressed exceptions.

### 📖 Core Concept

**Custom exceptions** should:
- Extend `RuntimeException` by default (unless handling is forced-recovery).
- Provide the standard 4 constructors (empty, message, message+cause, cause).
- Include domain fields when useful (an `orderId` on `OrderNotFoundException`).
- Be serializable (`Exception` already is) with `serialVersionUID`.

**`try-with-resources`** (Java 7+) auto-closes resources implementing `AutoCloseable`. Resources are closed in **reverse order of declaration**. If multiple resources throw on close, the first (from the try body) is the primary; later ones attach as **suppressed exceptions**.

### 💻 Code Example

```java
public class OrderNotFoundException extends RuntimeException {
    private final String orderId;

    public OrderNotFoundException(String orderId) {
        super("Order not found: " + orderId);
        this.orderId = orderId;
    }
    public OrderNotFoundException(String orderId, Throwable cause) {
        super("Order not found: " + orderId, cause);
        this.orderId = orderId;
    }
    public String getOrderId() { return orderId; }
}
```

**try-with-resources:**
```java
try (BufferedReader in = Files.newBufferedReader(Path.of("in.txt"));
     BufferedWriter out = Files.newBufferedWriter(Path.of("out.txt"))) {
    String line;
    while ((line = in.readLine()) != null) {
        out.write(line);
        out.newLine();
    }
}   // 'out' closed first, then 'in' — reverse of declaration
```

**Suppressed exceptions:**
```java
try {
    ...
} catch (Exception e) {
    for (Throwable s : e.getSuppressed()) {
        log.warn("suppressed", s);
    }
}
```

Java 9 extension — you can use an **effectively final variable** already declared:
```java
BufferedReader reader = Files.newBufferedReader(path);
try (reader) {
    ...
}
```

### ⚠️ Common Pitfalls
- Declaring a custom exception that takes only a message and dropping the `cause` overload — you lose diagnostic context forever.
- Try-finally with a close() that throws masks the original exception. Always prefer try-with-resources.
- Inheriting from `Throwable` or `Error` — never do that.

### 🎤 Interview Angle
> "What's the difference between a suppressed exception and a cause?"

**Cause** — a lower-level exception that triggered the one you're holding (set via `initCause` or constructor). Shown as `Caused by:` in stack traces.
**Suppressed** — a **secondary** exception that occurred during cleanup/close() while a **primary** exception was already in flight. Shown as `Suppressed:`.

### 🧪 Practice
1. Write a `ConfigLoadException` with `filename`, message, and cause.
2. Rewrite an old try-finally file-copy using try-with-resources.
3. Create a custom `AutoCloseable` that throws on close; confirm the suppression mechanism attaches it.

### 🔍 Self-check
1. What interface must a resource implement to be used in try-with-resources?
2. In what order are multiple resources closed?
3. Where can you see a suppressed exception in a stack trace?
4. Should your custom exception extend `Exception` or `RuntimeException` by default?

---

## Thursday — Generics Basics: Classes, Methods, Type Parameters

### 🎯 Objective
Write generic classes and methods; explain why generics existed before Java 5 couldn't exist naturally.

### 📖 Core Concept

**Generics** parameterize types over other types, giving compile-time type safety and removing casts.

**Pre-generics (Java 1.4) — unsafe:**
```java
List list = new ArrayList();
list.add("hello");
Integer i = (Integer) list.get(0);   // ClassCastException at runtime
```

**With generics — safe:**
```java
List<String> list = new ArrayList<>();
list.add("hello");
Integer i = list.get(0);              // COMPILE ERROR
```

**Generic class:**
```java
public class Box<T> {
    private T value;
    public void set(T v) { this.value = v; }
    public T get() { return value; }
}
```

**Multiple type parameters:**
```java
public class Pair<K, V> {
    private final K key;
    private final V value;
    public Pair(K key, V value) { this.key = key; this.value = value; }
    public K key() { return key; }
    public V value() { return value; }
}
```

**Generic method:**
```java
public static <T> T firstOrDefault(List<T> list, T defaultValue) {
    return list.isEmpty() ? defaultValue : list.get(0);
}
// Usage — type often inferred:
String s = firstOrDefault(List.of("a", "b"), "none");
```

**Naming conventions:** `T` (Type), `E` (Element), `K/V` (Key/Value), `N` (Number), `R` (Return), additional `S/U`.

### 💻 Code Example

```java
public class Result<T, E extends Exception> {
    private final T value;
    private final E error;

    private Result(T value, E error) { this.value = value; this.error = error; }

    public static <T, E extends Exception> Result<T, E> ok(T value) {
        return new Result<>(value, null);
    }
    public static <T, E extends Exception> Result<T, E> err(E error) {
        return new Result<>(null, error);
    }

    public boolean isOk() { return error == null; }
    public T value() { return value; }
    public E error() { return error; }
}

// Usage:
Result<Integer, IOException> r = Result.ok(42);
```

### ⚠️ Common Pitfalls
- Using a raw type (`List` instead of `List<String>`) — compiler issues warnings and you lose all type safety.
- Confusing type parameters with method parameters. `<T>` is the "argument" to the class/method at the type level.
- Declaring `<T>` on the **method** versus on the **class** — they're different scopes.

### 🎤 Interview Angle
> "Why can't you overload methods that differ only by generic type?"

Because of type erasure — see tomorrow. `void m(List<String>)` and `void m(List<Integer>)` both erase to `void m(List)` and clash.

### 🧪 Practice
1. Rewrite an `ObjectStack` you might have written pre-generics as a `Stack<T>`.
2. Write a generic `Pair<K, V>` with `equals`/`hashCode`.
3. Write a generic method `<T> List<T> repeat(T item, int times)` that returns a list of `times` copies of `item`.

### 🔍 Self-check
1. Where does the `<T>` go: on the class, the method, or both?
2. Can you instantiate a generic class without specifying the type? (Raw type — legal but warned.)
3. What does the diamond `<>` operator do?
4. Can you declare `<T>` twice in the same class at different scopes?

---

## Friday — Type Erasure & Its Consequences

### 🎯 Objective
Explain how generics work at runtime (or rather, don't), and list the concrete limitations that result.

### 📖 Core Concept

**Type erasure:** generics are **compile-time only**. After compilation, type parameters are replaced by their upper bound (or `Object` if unbounded). This is for **backward compatibility** with pre-generics Java.

```java
// Source:
class Box<T> { T item; }
// After erasure:
class Box { Object item; }

// Source:
class NumberBox<T extends Number> { T item; }
// After erasure:
class NumberBox { Number item; }
```

**Consequences you must know:**

**1. No `new T()` — can't instantiate a type parameter:**
```java
class Factory<T> {
    T create() { return new T(); }   // COMPILE ERROR
}
// Workaround — pass a Supplier or Class<T>:
class Factory<T> {
    T create(Supplier<T> s) { return s.get(); }
}
```

**2. No generic arrays:**
```java
T[] arr = new T[10];                 // COMPILE ERROR
List<String>[] lists = new List<String>[10]; // COMPILE ERROR
```

**3. `instanceof` with parameterized types forbidden:**
```java
if (obj instanceof List<String>)    // COMPILE ERROR
if (obj instanceof List<?>)         // OK (unbounded wildcard)
if (obj instanceof List)            // OK (raw type, with warning)
```

**4. Overloads that differ only by generic parameter conflict:**
```java
void m(List<String> s) {}
void m(List<Integer> i) {}           // COMPILE ERROR — both erase to m(List)
```

**5. Static fields are shared across all type parameterizations:**
```java
class Box<T> { static int counter; }
Box<String>.counter == Box<Integer>.counter;  // same variable
```

**6. Generic info IS retained at the class/method/field level** (for tools and reflection):
```java
Field f = Foo.class.getDeclaredField("list");
Type t = f.getGenericType();    // "java.util.List<java.lang.String>"
```

### 💻 Code Example

```java
// Workaround for "no new T()"
class Cache<T> {
    private final Supplier<T> factory;
    public Cache(Supplier<T> factory) { this.factory = factory; }
    public T getOrCreate(/* key */) { return factory.get(); }
}
Cache<ArrayList<String>> c = new Cache<>(ArrayList::new);

// Workaround for reading generic class at runtime
class TypeToken<T> {
    public Type type() {
        return ((ParameterizedType) getClass().getGenericSuperclass())
            .getActualTypeArguments()[0];
    }
}
Type t = new TypeToken<List<String>>() {}.type();   // List<String>
// Jackson's TypeReference and Gson's TypeToken use this exact trick.
```

### ⚠️ Common Pitfalls
- Assuming you can check a type's generic parameter at runtime — you can't, except via reflected metadata.
- Unchecked casts `(T) obj` compile with a warning but blow up at runtime if wrong. The cast is actually inserted where the value is **used**, not where you wrote it.
- Declaring `static <T>` fields — they're shared, not per-type.

### 🎤 Interview Angle
> "Can you do `if (list instanceof List<String>)`? Why not?"

No. After erasure, the runtime has no clue what the generic parameter is. The only check possible is `instanceof List` or `instanceof List<?>`.

### 🧪 Practice
1. Try to create each forbidden construct above and read the exact compile error.
2. Implement a tiny type-token class like Gson's `TypeToken<T>` using an anonymous subclass.
3. Write a `class Repository<T>` with a "find by example" that requires a `Class<T>` to work around erasure.

### 🔍 Self-check
1. Why is type erasure a thing at all?
2. Why can't you have a static generic field per type?
3. How does Jackson parse `List<MyObj>` from JSON if generics are erased? (`TypeReference` trick.)
4. What warning do you get when you use a raw type?

---

## Saturday — Bounded Types & Wildcards (PECS)

### 🎯 Objective
Read and write method signatures using `extends`/`super` wildcards correctly — apply PECS.

### 📖 Core Concept

**Bounded type parameters:**
```java
<T extends Number>             // T must be Number or subclass
<T extends Number & Comparable<T>>    // multiple bounds — class first, then interfaces
```

**Invariance** — by default, `List<Integer>` is NOT a subtype of `List<Number>` even though `Integer extends Number`.

**Wildcards provide variance:**

**`? extends T`** — **Producer**: T or any subtype. You can READ (elements are at least T) but NOT write (exact subtype unknown).
```java
double sum(List<? extends Number> list) {
    double total = 0;
    for (Number n : list) total += n.doubleValue();   // read OK
    // list.add(1);  // COMPILE ERROR — type unknown
    return total;
}
sum(List.of(1, 2, 3));     // List<Integer> accepted
sum(List.of(1.5, 2.5));    // List<Double> accepted
```

**`? super T`** — **Consumer**: T or any supertype. You can WRITE (T fits into any supertype) but reads come out as `Object`.
```java
void addIntegers(List<? super Integer> list) {
    list.add(1);                 // OK — any supertype holds Integer
    list.add(42);
    // Integer i = list.get(0);  // COMPILE ERROR — could be List<Object>
}
addIntegers(new ArrayList<Integer>());  // OK
addIntegers(new ArrayList<Number>());   // OK
addIntegers(new ArrayList<Object>());   // OK
```

**`?` unbounded** — read as `Object`, cannot write (except null).

### 📐 PECS Mnemonic (Joshua Bloch)

> **P**roducer **E**xtends, **C**onsumer **S**uper.

If the parameter produces T → `? extends T`.
If the parameter consumes T → `? super T`.
If both → use exact `T`.

**Canonical example — `Collections.copy`:**
```java
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
    for (T t : src) dest.add(t);
}
// src produces T  → extends
// dest consumes T → super
```

### 💻 Code Example

```java
// Producer — read values out
public static double sumOfList(List<? extends Number> list) {
    double total = 0;
    for (Number n : list) total += n.doubleValue();
    return total;
}

// Consumer — write values in
public static void addAll(List<? super Integer> dest, int... values) {
    for (int v : values) dest.add(v);
}

// Both producer and consumer — use exact T
public static <T> void swap(List<T> list, int i, int j) {
    T tmp = list.get(i);
    list.set(i, list.get(j));
    list.set(j, tmp);
}
```

### ⚠️ Common Pitfalls
- Trying to add to a `List<? extends T>` — compile error.
- Reading from `List<? super T>` and using it as T — you only get `Object`.
- Writing `<T super X>` on a type parameter — lower bounds only exist for **wildcards**.

### 🎤 Interview Angle
> "Why does `List<? extends Number>` reject `.add(1)`?"

Because the compiler knows the list is *some* subtype of Number — maybe `List<Double>`. Adding an `Integer` would violate that list's type. `? extends` means "read-only from the caller's perspective."

> "What does `Comparator<? super T>` mean on `Collections.sort(List<T>, Comparator<? super T>)`?"

It means a comparator for T OR any supertype of T. This lets you sort `List<Employee>` using a `Comparator<Person>`. Demonstrates real-world PECS (consumer super).

### 🧪 Practice
1. Fix this signature so it compiles and works correctly:
   ```java
   <T> void copyAll(List<T> src, List<T> dest)   // too strict
   ```
2. Write a `min` method that takes `List<? extends T>` and `Comparator<? super T>`.
3. Explain in writing why `Collections.addAll(Collection<? super T>, T...)` uses `super`.

### 🔍 Self-check
1. Is `List<Integer>` a subtype of `List<Number>`?
2. What's the difference between `List<?>` and `List<Object>`?
3. In PECS, which wildcard do you use for a parameter that you only read from?
4. Can you write `<T super Foo>` on a type parameter?

---

## Sunday — Revision & Q&A

### 🧭 Revision checklist
- [ ] Draw the `Throwable` hierarchy, list 3 checked + 3 unchecked examples.
- [ ] Predict output for 5 tricky try-catch-finally snippets.
- [ ] Write a polished custom exception with 4 constructors.
- [ ] List 5 consequences of type erasure from memory.
- [ ] Rewrite a bad `<T>` signature with correct PECS wildcards.

### 📝 Interview Q&A
Ask Claude: **"Prepare a Phase 1 Week 2 interview Q&A doc"** — you answer, Claude reviews and rates.

### 📂 Commit your work
```
phase-1/week-02/
├── exercises/
│   ├── BankAccount.java + InsufficientBalanceException.java
│   ├── FinallyClassics.java
│   ├── CustomException.java
│   ├── Box.java / Pair.java / Result.java
│   └── PECSDemo.java
└── notes.md
```
