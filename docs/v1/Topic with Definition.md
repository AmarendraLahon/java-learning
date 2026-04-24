# 📘 Java Interview Definitions – Quick Reference

---

## 🔹 Memory & Object Creation

### Stack

A thread-specific, LIFO memory region that stores **stack frames** — one per method call. Each frame holds local variables, method parameters, partial results, and return addresses. Allocation and deallocation happen automatically when methods are entered and exited. Fixed size per thread (`-Xss`); exceeding it throws `StackOverflowError`.

```java
void outer() {
    int x = 10;          // x lives in outer()'s stack frame
    inner(x);            // a new stack frame is pushed for inner()
}
void inner(int n) {
    int result = n * 2;  // result lives in inner()'s stack frame
}                         // frame popped when inner() returns
```

### Heap

Shared memory region (across all threads) where **all objects and arrays** are allocated. Managed by the Garbage Collector. Divided into generations (Young, Old) for GC efficiency. Exceeding available heap throws `OutOfMemoryError: Java heap space`.

```java
Person p = new Person("Alice");  // 'p' reference sits on stack, Person object on heap
int[] arr = new int[1000];       // array object on heap
```

### Java Object Creation

Full sequence when `new` is executed:
1. **Class loading** — JVM loads the class if not already loaded.
2. **Class initialization** — static blocks/fields run once per class.
3. **Memory allocation** — space reserved in heap (Eden for young objects).
4. **Default initialization** — fields set to defaults (null, 0, false).
5. **Explicit initialization** — instance initializer blocks + field initializers.
6. **Constructor execution** — chained from `Object` down through superclasses.
7. **Reference assignment** — the reference is returned to the caller.

```java
Person p = new Person("Alice"); // triggers all 7 steps above
```

---

## 🔹 Garbage Collection

### Minor GC

Reclaims unreachable objects from the **Young Generation** (Eden + Survivor spaces). Runs frequently because most objects die young ("generational hypothesis"). Short stop-the-world pause. Surviving objects are promoted through Survivor spaces and eventually to Old Generation.

### Major GC

Collects the **Old Generation** only. Runs much less often than Minor GC but is significantly more expensive. Often triggered when Old Gen fills up after repeated promotions from Young Gen.

### Full GC

Collects **the entire heap — Young + Old Generation (and Metaspace)**. More expensive than Major GC. Triggered by `System.gc()`, Metaspace exhaustion, or when Old Gen can't accommodate promotions. Should be rare in a well-tuned JVM.

> ⚠️ **Common myth:** Major GC ≠ Full GC. Major GC = Old Gen only; Full GC = everything.

---

## 🔹 Hashing & Equality

### hashCode()

Method on `Object` that returns an `int` hash representing the object. Used by hash-based collections (`HashMap`, `HashSet`) to determine the bucket where the object should be placed. Contract: if `a.equals(b)` then `a.hashCode() == b.hashCode()`.

```java
@Override
public int hashCode() {
    return Objects.hash(id, name);
}
```

### equals()

Method on `Object` for **logical equality** comparison. Default implementation uses `==` (reference equality). Must be overridden for value semantics. Must be reflexive, symmetric, transitive, consistent, and `x.equals(null)` must return false.

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Employee e)) return false;
    return id == e.id && Objects.equals(name, e.name);
}
```

### Hash Collision

When two distinct keys produce the same hash bucket index. Resolved by chaining within the bucket — in `HashMap`, the bucket holds a linked list (or red-black tree if too many entries collide).

```java
// Two different strings happening to hash to the same bucket
map.put("FB", 1);
map.put("Ea", 2);  // "FB".hashCode() == "Ea".hashCode() — classic collision
```

---

## 🔹 Collections Core

### List

An **ordered** collection (indexed) that allows duplicates and preserves insertion order. Supports positional access via index.

```java
List<String> names = new ArrayList<>();
names.add("A"); names.add("B"); names.add("A");  // duplicates OK
names.get(0);  // "A"
```

### Set

A collection that contains **no duplicate elements** (per `equals`). Most implementations don't guarantee order (`HashSet`); some do (`LinkedHashSet`, `TreeSet`).

```java
Set<String> unique = new HashSet<>();
unique.add("A"); unique.add("A");  // second add ignored
unique.size();  // 1
```

---

## 🔹 List Implementations

### ArrayList

Resizable-array implementation of `List`. Default capacity 10, grows by ~50% when full. O(1) random access, O(1) amortized append, O(n) insert/delete in the middle (shifting required).

```java
List<Integer> al = new ArrayList<>();
al.add(1); al.add(2); al.add(3);
al.get(1);           // O(1)
al.add(0, 99);       // O(n) — shifts elements right
```

### LinkedList

**Doubly-linked list** that implements `List`, `Deque`, and `Queue`. Each node holds a value plus `prev` and `next` pointers. O(1) add/remove at either end, O(1) when you already have the node, but O(n) random access by index. Higher memory overhead per element (two pointers).

```java
LinkedList<Integer> ll = new LinkedList<>();
ll.addFirst(1); ll.addLast(2);    // O(1) both ends — Deque behavior
ll.peekFirst(); ll.pollLast();    // used as queue/stack/deque
ll.get(5);                        // O(n) — walks from head or tail
```

**When to use:** frequent insertions/removals at both ends, queue/deque semantics. For most list use cases, `ArrayList` is faster due to CPU cache locality.

---

## 🔹 Set Implementations

### HashSet

Backed by a `HashMap` internally. O(1) average add/contains/remove. No ordering guarantee; the iteration order can change as the set grows. Permits one `null` element.

```java
Set<String> s = new HashSet<>();
s.add("B"); s.add("A"); s.add("C");
// Iteration order is NOT A, B, C — no guarantee
```

### LinkedHashSet

`HashSet` variant backed by a `LinkedHashMap` — maintains a doubly-linked list through entries in **insertion order**. Slightly slower than `HashSet` due to link maintenance, but predictable iteration.

```java
Set<String> s = new LinkedHashSet<>();
s.add("B"); s.add("A"); s.add("C");
// Iterates as B, A, C — insertion order preserved
```

---

## 🔹 Map Implementations

### HashMap

Key-value structure backed by a hash table (array of buckets). Allows **one null key and multiple null values**. **Not thread-safe**. Average O(1) for `get`/`put`/`remove`, O(log n) worst case (treeified bucket), O(n) if hashing is pathological.

```java
Map<String, Integer> m = new HashMap<>();
m.put("a", 1); m.put(null, 0);   // one null key allowed
m.put("b", null);                // null values allowed
m.getOrDefault("c", -1);         // -1
```

### Treeification in HashMap

Java 8+ optimization: when a single bucket's linked list grows beyond **TREEIFY_THRESHOLD (8)** entries **AND** the table capacity is ≥ 64, that bucket converts from a linked list into a **red-black tree**, improving worst-case lookup from O(n) to O(log n). Reverts to a linked list when entries drop below **UNTREEIFY_THRESHOLD (6)**. Protects against hash-collision DoS attacks.

### LinkedHashMap

`HashMap` that additionally maintains a doubly-linked list across entries. Two modes:
- **Insertion-order** (default) — iterates in insertion sequence.
- **Access-order** (constructor flag) — moves accessed entries to the tail; enables easy LRU caches.

```java
Map<String, Integer> lru = new LinkedHashMap<>(16, 0.75f, true) {
    protected boolean removeEldestEntry(Map.Entry<String, Integer> e) {
        return size() > 100;   // LRU cap
    }
};
```

---

## 🔹 LRU Cache

### LRU Cache

**Least Recently Used** cache — when capacity is exceeded, evicts the entry that hasn't been accessed for the longest time. Commonly implemented via `LinkedHashMap` with access-order + `removeEldestEntry` override, or a HashMap + doubly linked list.

```java
class LRU<K,V> extends LinkedHashMap<K,V> {
    private final int cap;
    LRU(int cap) { super(cap, 0.75f, true); this.cap = cap; }
    protected boolean removeEldestEntry(Map.Entry<K,V> e) { return size() > cap; }
}
```

---

## 🔹 Collection Time Complexity

Quick reference — average case:

| Operation | ArrayList | LinkedList | HashMap/HashSet | TreeMap/TreeSet | LinkedHashMap |
|-----------|-----------|-----------|-----------------|-----------------|---------------|
| `get`/`contains` | O(1) | O(n) | O(1) | O(log n) | O(1) |
| `add`/`put` | O(1) amortized | O(1) at ends | O(1) | O(log n) | O(1) |
| `remove` | O(n) | O(1) with ref | O(1) | O(log n) | O(1) |

---

## 🔹 Concurrency Collections

### ConcurrentHashMap

Thread-safe `Map` designed for high concurrent throughput. **Java 8+** implementation uses **CAS (Compare-And-Swap) operations** and **synchronized blocks on individual bucket heads** (not segments as in Java 7). Reads are generally lock-free; writes lock only the affected bucket. Does **not** allow null keys or null values.

```java
ConcurrentHashMap<String, Integer> counts = new ConcurrentHashMap<>();
counts.merge("apple", 1, Integer::sum);   // atomic increment
counts.computeIfAbsent("x", k -> load(k));
```

### SynchronizedMap

`Collections.synchronizedMap(map)` — wraps a map so every method acquires a **single intrinsic lock**. Simple but coarse-grained; poor scalability under contention. Iteration still requires external synchronization on the wrapper.

```java
Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());
synchronized (syncMap) {   // required for safe iteration
    for (var e : syncMap.entrySet()) { ... }
}
```

---

## 🔹 Iteration Behavior

### Fail-Fast

Iterator that throws `ConcurrentModificationException` **immediately** if the underlying collection is structurally modified during iteration (except via the iterator's own `remove`). Implemented via a `modCount` counter. Used by most non-concurrent collections (`ArrayList`, `HashMap`, `HashSet`).

```java
List<Integer> list = new ArrayList<>(List.of(1, 2, 3));
for (int x : list) {
    list.add(4);   // throws ConcurrentModificationException
}
```

### Fail-Safe

Iterator that operates on a **snapshot or copy** of the collection's state, so concurrent modifications don't cause exceptions — but iteration may not reflect the latest changes. Used by `CopyOnWriteArrayList`, `ConcurrentHashMap`.

```java
CopyOnWriteArrayList<Integer> list = new CopyOnWriteArrayList<>(List.of(1, 2, 3));
for (int x : list) {
    list.add(4);   // no exception — iterator sees original snapshot
}
```

---

### CopyOnWriteArrayList

Thread-safe `List` where **every mutating operation creates a fresh copy of the internal array**. Reads are lock-free and fast; writes are expensive. Best for read-heavy, rarely-mutated lists (event listener lists, configuration).

```java
CopyOnWriteArrayList<Listener> listeners = new CopyOnWriteArrayList<>();
listeners.add(l);                  // copy-on-write: allocates new array
for (var l : listeners) l.fire();  // safe even if modified concurrently
```

---

## 🔹 Memory Management

### Memory Leak in Java

Objects that are no longer logically needed but remain reachable from GC roots, preventing reclamation. Common causes: unbounded caches, static collections, unremoved listeners, `ThreadLocal` misuse, inner-class references.

```java
class LeakExample {
    static List<byte[]> cache = new ArrayList<>();   // grows forever
    void add() { cache.add(new byte[1024 * 1024]); } // eventual OOM
}
```

---

## 🔹 Sorting & Comparison

### Comparable

Interface implemented by a class to define its **natural ordering**. Single method `compareTo(T o)` returning negative / zero / positive. Used by `Collections.sort`, `TreeSet`, `TreeMap`, sorted streams.

```java
class Person implements Comparable<Person> {
    int age;
    public int compareTo(Person o) { return Integer.compare(this.age, o.age); }
}
```

### Comparator

Interface defining an **external** ordering, separate from the class itself. Useful when the class has no natural order, or when you need a different order than the natural one.

```java
Comparator<Person> byName = Comparator.comparing(p -> p.name);
list.sort(byName.thenComparing(p -> p.age).reversed());
```

### compareTo()

The single method in `Comparable`. Returns:
- negative int → `this` < `other`
- zero → equal ordering
- positive int → `this` > `other`

Should be consistent with `equals` (recommended, not enforced).

---

## 🔹 String Handling

### String

Immutable sequence of characters. **Stored on the heap**, not all in the string pool. Only **string literals** and explicitly `intern()`-ed strings live in the pool; `new String("...")` creates a fresh heap object outside the pool.

```java
String a = "hello";                      // pooled literal
String b = "hello";                      // same pooled reference as a
String c = new String("hello");          // new heap object, NOT pooled
String d = c.intern();                   // gets the pooled reference
a == b;  // true
a == c;  // false
a == d;  // true
```

### String Pool (String Constant Pool)

A special area in the **heap** (moved from PermGen in Java 7) that stores unique string literals. When you write a literal, the JVM checks the pool: reuses the existing instance if present, otherwise adds it. Saves memory for commonly repeated strings.

### StringBuilder

Mutable, resizable sequence of characters for efficient string manipulation. **Not thread-safe** (use `StringBuffer` for that, but its synchronization overhead is rarely worth it). Prefer over repeated `String` concatenation in loops.

```java
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++) sb.append(i).append(',');
String result = sb.toString();
```

---

## 🔹 Wrapper Classes

### Wrapper Class

Object representation of a primitive: `Integer`, `Long`, `Double`, `Boolean`, `Character`, `Byte`, `Short`, `Float`. Needed because generics and collections work only with reference types. All wrappers are **immutable** and **final**.

```java
Integer i = Integer.valueOf(42);
int unboxed = i.intValue();
List<Integer> list = new ArrayList<>();   // primitives not allowed in generics
```

### Autoboxing

Automatic compiler-inserted conversion from primitive to wrapper.

```java
List<Integer> list = new ArrayList<>();
list.add(5);           // autoboxes int 5 → Integer.valueOf(5)
Integer x = 10;        // autoboxes
```

### Unboxing

Automatic conversion from wrapper to primitive. **Can throw `NullPointerException`** if the wrapper is null.

```java
Integer x = null;
int y = x;             // NullPointerException at unboxing
```

> ⚠️ Integer cache: `Integer.valueOf` caches values from -128 to 127. So `Integer.valueOf(100) == Integer.valueOf(100)` is true, but `Integer.valueOf(200) == Integer.valueOf(200)` is false. Always use `.equals()` for wrapper comparison.

---

## 🔹 OOP Concepts

### Abstraction

Hiding internal implementation details while exposing only the essential behavior through a simplified interface. Achieved via abstract classes and interfaces.

```java
interface PaymentGateway { void charge(double amount); }  // WHAT, not HOW
class StripeGateway implements PaymentGateway {
    public void charge(double amount) { /* Stripe-specific logic */ }
}
```

### Encapsulation

Bundling data and methods that operate on it together, while restricting direct access to the internal state. Typically: `private` fields + `public` getters/setters + validation.

```java
class Account {
    private double balance;   // hidden state
    public void deposit(double amt) {
        if (amt <= 0) throw new IllegalArgumentException();
        balance += amt;
    }
    public double getBalance() { return balance; }
}
```

### Inheritance

Mechanism where a subclass acquires the fields and methods of a superclass using `extends`. Enables code reuse and polymorphism. Java supports single inheritance of classes, multiple inheritance of interfaces.

```java
class Animal { void eat() { ... } }
class Dog extends Animal { void bark() { ... } }
Dog d = new Dog();
d.eat(); d.bark();   // inherits eat()
```

### Composition

Design where a class holds references to other classes' instances to delegate functionality ("has-a"). Often preferred over inheritance ("favor composition over inheritance") because it's more flexible and avoids tight coupling.

```java
class Engine { void start() { ... } }
class Car {
    private final Engine engine = new Engine();   // Car HAS-A Engine
    void drive() { engine.start(); }
}
```

---

### IS-A Relationship

Expresses **inheritance** — one class is a specialized version of another. Tested by `instanceof`.

```java
class Dog extends Animal { }    // Dog IS-A Animal
Dog d = new Dog();
boolean b = d instanceof Animal; // true
```

### HAS-A Relationship

Expresses **composition** — one class contains another as a field. Not tested by `instanceof`; inspected by looking at the class's fields.

```java
class Car { private Engine engine; }   // Car HAS-A Engine
```

---

### Program to Interface

Using the interface (not the concrete type) as the declared reference type. Enables swapping implementations without changing client code — reduces coupling, increases flexibility.

```java
List<String> list = new ArrayList<>();   // interface reference, concrete impl
// Later, swap impl without touching callers:
List<String> list2 = new LinkedList<>();
```

---

## 🔹 Static & Interfaces

### Static Method

A method belonging to the class itself rather than any instance. Cannot access instance fields directly. Called via `ClassName.method()`. Cannot be overridden (only hidden).

```java
class MathUtils {
    public static int square(int x) { return x * x; }
}
int r = MathUtils.square(5);   // no object needed
```

### Abstract Class

A class declared with `abstract` that cannot be instantiated. May contain both abstract (unimplemented) and concrete (implemented) methods, state (fields), and constructors. Intended as a partial implementation to be completed by subclasses.

```java
abstract class Shape {
    abstract double area();               // must override
    void describe() { System.out.println("Shape"); }  // inherited as-is
}
class Circle extends Shape {
    double r;
    double area() { return Math.PI * r * r; }
}
```

### Interface

A contract of abstract methods (and since Java 8, optional `default` and `static` methods; since Java 9, `private` helper methods). Classes implement interfaces with `implements`. Supports multiple inheritance of type.

```java
interface Runnable { void run(); }
interface Comparable<T> {
    int compareTo(T o);
    default boolean isLessThan(T o) { return compareTo(o) < 0; }
}
```

---

## 🔹 Multithreading Basics

### Thread

The smallest unit of independent execution within a process. Each thread has its own stack and program counter but shares the heap with other threads in the same process. Created by extending `Thread` or implementing `Runnable`.

```java
Thread t = new Thread(() -> System.out.println("running"));
t.start();
```

### Runnable

Functional interface with a single `void run()` method. Represents a task. Preferred over extending `Thread` because it decouples the task from thread management and leaves the class free to extend something else.

```java
Runnable task = () -> System.out.println("hello from " + Thread.currentThread().getName());
new Thread(task).start();
```

---

### start()

Method on `Thread` that registers the thread with the OS scheduler and invokes `run()` on a new thread of execution. Calling `start()` twice throws `IllegalThreadStateException`.

### run()

Method containing the code to be executed. Calling `run()` directly executes the code on the **current** thread — not a new one. Always call `start()` to actually create a new thread.

```java
new Thread(() -> System.out.println(Thread.currentThread().getName())).start();  // "Thread-0"
new Thread(() -> System.out.println(Thread.currentThread().getName())).run();    // "main"
```

---

### Thread-safe

Code that produces correct results regardless of the timing or interleaving of multiple threads accessing it. Achieved via immutability, synchronization, atomic operations, thread confinement, or thread-safe data structures.

```java
// Thread-safe — uses atomic operation
AtomicInteger count = new AtomicInteger();
count.incrementAndGet();
```

---

### join()

Causes the caller to wait until the target thread has finished execution. Useful for coordinating completion.

```java
Thread t = new Thread(task);
t.start();
t.join();           // main waits here until t completes
t.join(1000);       // wait at most 1 second
```

---

### volatile

Keyword that ensures **visibility** — writes by one thread are immediately visible to reads on other threads (no CPU cache staleness). Also prevents certain compiler reorderings. Does NOT provide atomicity for compound operations like `x++`.

```java
class Worker {
    private volatile boolean stop = false;
    void shutdown() { stop = true; }        // visible immediately in run()
    void run() { while (!stop) { /* work */ } }
}
```

---

## 🔹 Concurrency Issues

### Race Condition

A bug where the correctness of the computation depends on the unpredictable interleaving of multiple threads accessing shared mutable state.

```java
int count = 0;
// Thread 1 and Thread 2 both execute:
count++;  // read, increment, write — can interleave, losing updates
```

### Critical Section

A code region accessing shared mutable resources that must execute atomically (by at most one thread at a time). Protected by synchronization.

```java
synchronized (lock) {
    // critical section — only one thread here at a time
    balance += amount;
}
```

---

### Synchronized Keyword

Acquires the **intrinsic monitor lock** of an object. Can be applied to methods (locks `this` for instance, `Class` for static) or blocks (explicit object). Guarantees mutual exclusion AND visibility (establishes happens-before relationships).

```java
public synchronized void increment() { count++; }          // locks this

public void deposit(double amount) {
    synchronized (this) {                                   // explicit block
        balance += amount;
    }
}
```

---

### tryLock()

Method on `Lock` (`ReentrantLock`) that attempts to acquire the lock without blocking forever. Returns immediately with `true`/`false`, or can wait up to a timeout. Useful for avoiding deadlocks.

```java
Lock lock = new ReentrantLock();
if (lock.tryLock(500, TimeUnit.MILLISECONDS)) {
    try { /* critical section */ }
    finally { lock.unlock(); }
} else {
    // fallback — didn't get the lock, avoid deadlock
}
```

---

### Visibility

A memory guarantee: a write performed by one thread is observable by other threads. Java ensures visibility via `volatile`, `synchronized`, `final` field semantics, and `java.util.concurrent` utilities. Without these, another thread might see a stale, cached value indefinitely.

### Atomicity

Operations that execute as an indivisible unit — either all effects are visible or none. In Java, reads and writes of `references`, `int`, `boolean`, etc. are atomic, but `long`/`double` are NOT (unless `volatile`). Compound operations (`x++`) are not atomic without help (`AtomicInteger`, `synchronized`).

```java
AtomicInteger counter = new AtomicInteger();
counter.incrementAndGet();   // atomic compound operation
```

---

### ReentrantLock

Explicit lock from `java.util.concurrent.locks` that provides the same mutual exclusion as `synchronized` plus: `tryLock`, interruptible acquisition, fairness option, multiple `Condition` variables, and the ability to query the lock state. **Reentrant** = the holding thread can re-acquire it safely.

```java
Lock lock = new ReentrantLock(true);     // fair ordering
lock.lock();
try { /* work */ }
finally { lock.unlock(); }               // MUST unlock in finally
```

---

### ReentrantLock vs Synchronized

| Aspect | synchronized | ReentrantLock |
|--------|-------------|---------------|
| Acquisition | Implicit (block/method) | Explicit `lock()`/`unlock()` |
| Unlock | Automatic | Manual (in `finally`) |
| Try-acquire | Not supported | `tryLock()`, `tryLock(timeout)` |
| Interruptible | No | Yes (`lockInterruptibly()`) |
| Fairness | No control | Optional fairness |
| Conditions | One wait-set per object | Multiple `Condition` per lock |
| Readability | Simpler | More verbose |

**Rule of thumb:** use `synchronized` by default; reach for `ReentrantLock` when you need its extra capabilities.

---

### Deadlock

Two or more threads are blocked forever, each waiting for a lock held by another. Classic example: thread A holds lock1 and waits for lock2; thread B holds lock2 and waits for lock1.

```java
// Deadlock risk:
Thread t1 = new Thread(() -> {
    synchronized (lockA) { synchronized (lockB) { /* ... */ } }
});
Thread t2 = new Thread(() -> {
    synchronized (lockB) { synchronized (lockA) { /* ... */ } }
});
// Fix: always acquire locks in a consistent global order (e.g., by hashCode).
```

**Prevention:** consistent lock ordering, `tryLock` with timeouts, avoiding nested locks, using higher-level concurrency utilities (`BlockingQueue`, `ConcurrentHashMap`).

---
