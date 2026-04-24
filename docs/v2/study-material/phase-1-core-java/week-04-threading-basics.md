# Week 4 — Multithreading & Concurrency Basics

**Theme:** Concurrency is where senior-level Java interviews sharpen their knives. You must be fluent with thread lifecycle, the synchronized keyword, volatile, and the Java Memory Model. This week lays that foundation — next week builds the high-level abstractions (Executors, CompletableFuture).

**Companion reading:** *Java Concurrency in Practice* (Brian Goetz) — Chapters 1–3. This book is THE canonical reference.

---

## Monday — Thread Lifecycle, Thread vs Runnable vs Callable

### 🎯 Objective
Draw the thread state machine and explain each transition; choose between Thread/Runnable/Callable correctly.

### 📖 Core Concept

**Thread states (`Thread.State`):**

```
NEW ──start()──> RUNNABLE ──run() completes──> TERMINATED
                    │
                    ├── blocked waiting for monitor lock → BLOCKED
                    ├── wait() / join() / park() → WAITING
                    └── sleep(n) / wait(n) → TIMED_WAITING
```

- **NEW** — thread created but `start()` not yet called.
- **RUNNABLE** — eligible to run; may be actively running or waiting for CPU (JVM doesn't distinguish).
- **BLOCKED** — waiting to acquire a monitor lock (e.g., entering a `synchronized` block).
- **WAITING** — waiting indefinitely for another thread (via `wait()`, `join()`, `LockSupport.park()`).
- **TIMED_WAITING** — waiting with a timeout (`sleep`, `wait(ms)`, `join(ms)`).
- **TERMINATED** — `run()` has completed (normally or via exception).

**Three ways to create a thread:**

**1. Extend `Thread`** — old-school, couples task to thread.
```java
class MyThread extends Thread {
    public void run() { System.out.println("hi"); }
}
new MyThread().start();
```

**2. Implement `Runnable`** — preferred; decouples task from thread.
```java
Runnable task = () -> System.out.println("hi");
new Thread(task).start();
```

**3. Implement `Callable<V>`** — like Runnable but returns a value AND can throw checked exceptions.
```java
Callable<Integer> compute = () -> {
    Thread.sleep(1000);
    return 42;
};
Future<Integer> f = Executors.newSingleThreadExecutor().submit(compute);
Integer result = f.get();
```

**Daemon threads** — background threads that don't prevent JVM exit. Set via `t.setDaemon(true)` **before** `start()`. GC runs on daemon threads. User threads block JVM shutdown; daemon threads don't.

**Java 21 virtual threads** — lightweight threads (millions per JVM possible). `Thread.ofVirtual().start(task)` or `Executors.newVirtualThreadPerTaskExecutor()`. Game-changer for IO-heavy workloads.

### 💻 Code Example

```java
// Daemon worker that logs every second until JVM exit
Thread logger = new Thread(() -> {
    while (true) {
        System.out.println("heartbeat");
        try { Thread.sleep(1000); } catch (InterruptedException e) { return; }
    }
});
logger.setDaemon(true);
logger.start();

// Callable + Future
ExecutorService pool = Executors.newFixedThreadPool(2);
Future<Integer> f = pool.submit(() -> {
    Thread.sleep(500);
    return 42;
});
System.out.println(f.get());     // blocks until result
pool.shutdown();
```

### ⚠️ Common Pitfalls
- Calling `thread.run()` directly instead of `start()` — runs on the CURRENT thread (no new thread created).
- Calling `start()` twice — throws `IllegalThreadStateException`.
- Forgetting that `Runnable.run()` cannot throw checked exceptions — must catch or wrap.
- Setting `setDaemon(true)` AFTER `start()` — throws `IllegalThreadStateException`.

### 🎤 Interview Angle
> "What's the difference between `Runnable` and `Callable`?"

`Runnable` — `void run()`, no return, no checked exceptions. Fire-and-forget.
`Callable<V>` — `V call() throws Exception`. Returns a result and can throw.
Use `Callable` when you need the result (via `Future`) or need to propagate checked exceptions. Otherwise `Runnable`.

### 🧪 Practice
1. Write each thread-creation method (Thread subclass, Runnable, Callable) and time 10,000 tasks.
2. Observe daemon vs user thread behavior: spawn both, let the main thread exit, see which survives.
3. Log thread state transitions using `Thread.getState()` across various operations.

### 🔍 Self-check
1. What state is a thread in while waiting for `synchronized`?
2. Why is `extends Thread` less flexible than `implements Runnable`?
3. What happens if a thread's `run()` throws an uncaught exception?
4. How do you set a thread as daemon, and what's the deadline?

---

## Tuesday — `synchronized`: Method, Block, Intrinsic Locks

### 🎯 Objective
Use `synchronized` correctly on methods and blocks; explain intrinsic locks and reentrancy.

### 📖 Core Concept

**`synchronized`** acquires the **intrinsic monitor lock** of an object. While held, no other thread can enter **any** synchronized method or block that locks on the same object.

**Three syntaxes:**

```java
// 1. Synchronized instance method → locks on 'this'
public synchronized void m() { ... }

// 2. Synchronized static method → locks on Class object
public static synchronized void m() { ... }

// 3. Synchronized block → locks on any specified object
public void m() {
    synchronized (this) { ... }             // same as (1)
    synchronized (MyClass.class) { ... }     // same as (2)
    synchronized (someField) { ... }         // custom lock
}
```

**Reentrancy** — a thread holding a lock can re-enter the same lock without blocking:
```java
synchronized void a() { b(); }     // holds lock
synchronized void b() { ... }      // already holds, enters freely
```

**What synchronized guarantees:**
- **Mutual exclusion** — only one thread in the critical section at a time.
- **Visibility** — changes made before releasing the lock are visible to the next acquirer (happens-before).
- **Atomicity** — the entire block executes as an indivisible unit.

**Lock granularity:**
- Method-level: simple but coarse (locks the whole object).
- Block-level: tighter, only the critical region is synchronized.
- Separate lock object: independent locks for independent state (can be read/written concurrently).

### 💻 Code Example

```java
class Counter {
    private int value;
    private final Object lock = new Object();

    // Method-level — locks on 'this'
    public synchronized void incSynchronized() { value++; }

    // Block-level — locks on a separate object
    public void incWithLock() {
        synchronized (lock) { value++; }
    }

    public int get() {
        synchronized (lock) { return value; }
    }
}
```

**Separate locks for independent state:**
```java
class Account {
    private final Object depositLock = new Object();
    private final Object historyLock = new Object();
    private double balance;
    private List<Transaction> history = new ArrayList<>();

    public void deposit(double amt) {
        synchronized (depositLock) { balance += amt; }
        synchronized (historyLock) { history.add(new Transaction("DEPOSIT", amt)); }
    }
    // Two threads can deposit (contending on depositLock) while reading history freely.
}
```

### ⚠️ Common Pitfalls
- Locking on a **non-final** field — risk of the reference changing, creating two "different" locks.
- Locking on a **String literal** — literals are interned and shared across the JVM; anyone else locking on `"mylock"` shares your lock.
- Locking on a **boxed primitive** (`Integer.valueOf(1)`) — cached, shared.
- Using `this` as a lock in a public class — external callers can lock on your instance and block you.

**Best practice:** private final lock object:
```java
private final Object lock = new Object();
```

### 🎤 Interview Angle
> "What's the difference between `synchronized(this)` and `synchronized(lock)`?"

`this` exposes the lock to external callers who can synchronize on your object. A private final lock object encapsulates the lock, preventing interference. Always prefer the dedicated lock unless you have a specific reason to use `this`.

### 🧪 Practice
1. Write a thread-safe counter using three approaches: synchronized method, synchronized block on `this`, synchronized block on private lock. Benchmark contention.
2. Demonstrate reentrancy with a synchronized method calling itself recursively.
3. Cause a deadlock by locking on a String literal from two threads.

### 🔍 Self-check
1. What does `synchronized` on a static method lock on?
2. What does reentrancy mean? Is `synchronized` reentrant?
3. Why is locking on `String` literals dangerous?
4. What's the happens-before relationship with synchronized?

---

## Wednesday — `volatile`, Java Memory Model, Visibility vs Atomicity

### 🎯 Objective
Distinguish visibility from atomicity; know when `volatile` is sufficient and when it's not.

### 📖 Core Concept

**The problem:** modern CPUs cache memory aggressively. A write by thread A to a field may sit in A's CPU cache for an arbitrary time before propagating to RAM. Thread B reading the field may see a stale value — **indefinitely** without synchronization.

**Compiler reordering:** the JIT and CPU can reorder instructions for performance. `x = 1; flag = true;` may execute as `flag = true; x = 1;` if reorderings are locally safe but visible in multi-threaded contexts.

**`volatile` provides:**
1. **Visibility** — writes go through directly to main memory; reads don't use stale CPU cache.
2. **Ordering** — prevents certain reorderings (writes before a volatile write are not reordered past it; reads after a volatile read are not reordered before it).

**`volatile` does NOT provide:**
- **Atomicity of compound operations.** `x++` is read-modify-write — three steps. Two threads can both read x=5, both write x=6, losing an update.

**When `volatile` is enough:**
- Single-writer, multi-reader scenarios.
- Flag variables (e.g., `volatile boolean stop`).
- Published-once fields (e.g., double-checked locking uses `volatile` for the instance field).

**When `volatile` is NOT enough:**
- Compound state updates (`x++`, `map.put(key, newValue)` with dependency on old value).
- Need for `AtomicInteger`, `AtomicReference`, or synchronized.

**Happens-before** — the JMM formal rule. If action A happens-before action B, then B sees A's effects. Established by:
- Program order within a single thread.
- `volatile` write → subsequent `volatile` read on the same variable.
- `synchronized` release → `synchronized` acquire on the same lock.
- `Thread.start()` → first action of the started thread.
- Thread's last action → another thread's successful `join()`.

### 💻 Code Example

**Good — volatile flag:**
```java
class Worker {
    private volatile boolean stop = false;
    public void shutdown() { stop = true; }
    public void run() {
        while (!stop) { /* work */ }    // sees update eventually without volatile? MAYBE NEVER.
    }
}
// Without volatile, the JIT may hoist the read out of the loop and never recheck.
```

**Bad — volatile is not enough for compound ops:**
```java
class BrokenCounter {
    private volatile int count;
    public void inc() { count++; }   // NOT atomic! read-increment-write
}
// Use AtomicInteger or synchronized instead.
```

**Correct — AtomicInteger:**
```java
class Counter {
    private final AtomicInteger count = new AtomicInteger();
    public void inc() { count.incrementAndGet(); }
    public int get() { return count.get(); }
}
```

**Double-checked locking — volatile is mandatory here:**
```java
class Singleton {
    private static volatile Singleton instance;  // MUST be volatile
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) instance = new Singleton();
            }
        }
        return instance;
    }
}
// Without volatile: another thread could see a partially-constructed instance
// due to reordering of instance-assignment vs constructor-completion.
```

### ⚠️ Common Pitfalls
- Using `volatile` as a substitute for synchronization — fails for compound operations.
- Forgetting that `long`/`double` are NOT atomic for reads/writes on 32-bit JVMs — without `volatile`, you can see a "torn" value.
- DCL without `volatile` on the field — classic broken pattern.

### 🎤 Interview Angle
> "Is `x++` atomic? Why not?"

No. It's read-modify-write — three steps. Even on modern CPUs, the compiler generates three separate instructions (on x86 you can use `LOCK INC` for atomic increment, but Java doesn't emit that for a plain `x++`).

> "Why is volatile needed in double-checked locking?"

To prevent reordering of constructor completion vs. field assignment. Without volatile, another thread could see a non-null reference to a half-constructed object.

### 🧪 Practice
1. Run two threads incrementing a plain `int` counter 1M times each. Show the result is < 2M (race).
2. Fix with `volatile` — show it's STILL < 2M (volatile doesn't help compound ops).
3. Fix with `synchronized` or `AtomicInteger` — show correct 2M result.

### 🔍 Self-check
1. What two things does `volatile` guarantee?
2. Why is `x++` not atomic even with volatile?
3. List 3 scenarios where volatile is sufficient on its own.
4. Why does double-checked locking REQUIRE volatile?

---

## Thursday — wait / notify / notifyAll, Producer-Consumer

### 🎯 Objective
Use `wait`/`notify` correctly inside `synchronized`; implement a bounded buffer; understand spurious wakeups.

### 📖 Core Concept

**`wait()` / `notify()` / `notifyAll()`** are methods on `Object` (not Thread) used for thread coordination.

**Rules:**
- ALL three MUST be called while holding the **monitor of the object** they're called on — otherwise `IllegalMonitorStateException`.
- `wait()` **releases** the monitor and puts the thread into WAITING state; it reacquires the monitor before returning.
- `notify()` wakes up ONE arbitrary waiting thread on the same monitor.
- `notifyAll()` wakes up ALL waiting threads (they compete to re-acquire).

**Always use `while` (not `if`) to check the condition:**
```java
synchronized (obj) {
    while (!condition) {
        obj.wait();
    }
    // proceed
}
```
Three reasons:
1. **Spurious wakeups** — the JVM is allowed to wake threads without a notify.
2. **Multiple waiters** — another waiter might consume the condition before you.
3. **Condition may have changed** — between wakeup and lock re-acquisition.

**Producer-consumer with a bounded buffer:**

```java
class BoundedBuffer<T> {
    private final Queue<T> buf = new LinkedList<>();
    private final int capacity;

    public BoundedBuffer(int capacity) { this.capacity = capacity; }

    public synchronized void put(T item) throws InterruptedException {
        while (buf.size() == capacity) wait();
        buf.add(item);
        notifyAll();                         // wake up consumers (and other producers)
    }

    public synchronized T take() throws InterruptedException {
        while (buf.isEmpty()) wait();
        T item = buf.poll();
        notifyAll();
        return item;
    }
}
```

### 💻 Code Example (complete demo)

```java
public class ProducerConsumerDemo {
    public static void main(String[] args) throws InterruptedException {
        BoundedBuffer<Integer> buf = new BoundedBuffer<>(5);

        Thread producer = new Thread(() -> {
            try {
                for (int i = 0; i < 20; i++) {
                    buf.put(i);
                    System.out.println("produced " + i);
                    Thread.sleep(50);
                }
            } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        });

        Thread consumer = new Thread(() -> {
            try {
                for (int i = 0; i < 20; i++) {
                    int v = buf.take();
                    System.out.println("consumed " + v);
                    Thread.sleep(100);
                }
            } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        });

        producer.start(); consumer.start();
        producer.join(); consumer.join();
    }
}
```

### ⚠️ Common Pitfalls
- Calling `wait` without holding the monitor → `IllegalMonitorStateException`.
- Using `if` instead of `while` → breaks on spurious wakeup or concurrent condition changes.
- Using `notify()` with multiple condition types on same monitor (e.g., "not empty" and "not full") — you might wake the wrong thread. `notifyAll()` is safer but noisier. Better: use `Condition` from `ReentrantLock`.
- Forgetting to propagate `InterruptedException` — set the interrupt flag with `Thread.currentThread().interrupt()` or re-throw.

### 🎤 Interview Angle
> "Why `while` instead of `if` when checking the wait condition?"

Spurious wakeups are allowed by the JMM. Also, between `wait()` returning and your code executing, the condition might have changed (another thread consumed the item). `while` re-checks.

> "Difference between `notify` and `notifyAll`?"

`notify` wakes one arbitrary thread, `notifyAll` wakes all. Use `notifyAll` when waiters are waiting on different conditions, or when you're not sure which specific thread should proceed.

### 🧪 Practice
1. Implement the `BoundedBuffer` above.
2. Break it by using `if` instead of `while` — find a scenario where it produces wrong output.
3. Replace `wait/notify` with `ArrayBlockingQueue` — see how much cleaner it becomes (preview for next week).

### 🔍 Self-check
1. Where must `wait()` be called?
2. What's a spurious wakeup?
3. What does `wait()` do with the lock it's holding?
4. When would you use `notifyAll` instead of `notify`?

---

## Friday — ReentrantLock, ReadWriteLock, Fairness, tryLock

### 🎯 Objective
Use `ReentrantLock` and `ReadWriteLock` appropriately; know when to choose them over `synchronized`.

### 📖 Core Concept

**`ReentrantLock`** — explicit lock from `java.util.concurrent.locks`. Same mutual-exclusion semantics as `synchronized`, plus:

1. **`tryLock()`** — non-blocking attempt; optional timeout.
2. **`lockInterruptibly()`** — waiting threads can be interrupted.
3. **Fairness option** — FIFO ordering when constructed with `new ReentrantLock(true)`.
4. **Multiple `Condition` variables per lock** — separate wait queues for different conditions (can't be done with `wait/notify`).
5. **Queryable state** — `isLocked()`, `getHoldCount()`, `getQueueLength()`.

**Canonical usage pattern (ALWAYS unlock in finally):**
```java
Lock lock = new ReentrantLock();
lock.lock();
try {
    // critical section
} finally {
    lock.unlock();
}
```

**`ReadWriteLock`** — pair of locks: a shared **read lock** (multiple concurrent readers) and an exclusive **write lock**. Useful when reads vastly outnumber writes.

```java
ReadWriteLock rw = new ReentrantReadWriteLock();
rw.readLock().lock();
try { return data; } finally { rw.readLock().unlock(); }

rw.writeLock().lock();
try { data = newValue; } finally { rw.writeLock().unlock(); }
```

**`StampedLock` (Java 8+)** — advanced, supports **optimistic reads** (no locking at all, validated after the fact). Not reentrant. Higher performance for read-heavy workloads.

**`Condition` — multiple wait queues per lock:**
```java
Lock lock = new ReentrantLock();
Condition notFull = lock.newCondition();
Condition notEmpty = lock.newCondition();

void put(T t) throws InterruptedException {
    lock.lock();
    try {
        while (isFull()) notFull.await();
        buf.add(t);
        notEmpty.signal();      // signal consumers
    } finally { lock.unlock(); }
}
```
Advantage over `wait/notifyAll`: producers only wake consumers (not other producers) → fewer spurious wakeups.

### 💻 Code Example

**tryLock for deadlock-free multi-lock acquisition:**
```java
boolean transfer(Account from, Account to, double amount) throws InterruptedException {
    while (true) {
        if (from.lock.tryLock()) {
            try {
                if (to.lock.tryLock()) {
                    try {
                        if (from.balance >= amount) {
                            from.balance -= amount;
                            to.balance += amount;
                            return true;
                        }
                        return false;
                    } finally { to.lock.unlock(); }
                }
            } finally { from.lock.unlock(); }
        }
        Thread.sleep(1);       // back off before retry
    }
}
```

### ⚠️ Common Pitfalls
- Forgetting `unlock()` in `finally` — lock leaks, thread holds it forever, deadlock.
- Unlocking more times than locked — `IllegalMonitorStateException`.
- Using fairness unnecessarily — significant throughput cost (strict FIFO serializes more).
- `ReadWriteLock` write starvation under heavy read load — some implementations avoid this via fairness.

### 🎤 Interview Angle
> "When would you choose ReentrantLock over synchronized?"

- Need `tryLock` to avoid deadlocks.
- Need `lockInterruptibly` for responsive cancellation.
- Need multiple `Condition` variables.
- Need fairness guarantees.
- Need to query lock state.

Otherwise `synchronized` is simpler and has had significant JVM optimization (biased locking historically, now mostly lock coarsening and elision).

### 🧪 Practice
1. Rewrite the `BoundedBuffer` using `ReentrantLock` + two `Condition` variables.
2. Implement a cache using `ReadWriteLock` — benchmark read throughput vs `synchronized`.
3. Implement deadlock-free account transfer using `tryLock` with backoff.

### 🔍 Self-check
1. Why must `unlock()` go in `finally`?
2. What's the advantage of `Condition` over `wait/notify`?
3. What is a "fair" lock and what's the cost?
4. When would you use `ReadWriteLock` over `ReentrantLock`?

---

## Saturday — ThreadLocal, Interruption, Thread.stop Dangers

### 🎯 Objective
Use `ThreadLocal` correctly (and clean it up); cancel work via interruption; know why `Thread.stop` is banned.

### 📖 Core Concept

**`ThreadLocal<T>`** — per-thread storage. Each thread has its own copy of the value.

```java
private static final ThreadLocal<SimpleDateFormat> DATE_FORMAT =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

public String formatDate(Date d) {
    return DATE_FORMAT.get().format(d);     // thread-safe formatter
}
```

Common uses: per-request context in web apps, per-thread DB transactions, thread-confined mutable state.

**⚠️ Memory leak pitfall:** `ThreadLocal` in a thread pool (where threads are reused indefinitely) keeps values alive forever — GC can't reclaim them because the thread is still alive. ALWAYS call `remove()` in a finally:
```java
try {
    DATE_FORMAT.set(...);
    // use
} finally {
    DATE_FORMAT.remove();
}
```

**`InheritableThreadLocal<T>`** — child threads inherit the parent's value at creation time. Useful for propagating context across thread boundaries.

**Interruption** — the cooperative cancellation mechanism. Setting a thread's interrupt flag doesn't forcibly stop it; it signals the thread to stop at the next interruption-aware operation.

**Interrupt-aware operations** — `sleep`, `wait`, `join`, `BlockingQueue.take`, `Thread.sleep` — all throw `InterruptedException` when the flag is set.

**Proper cancellation pattern:**
```java
public void run() {
    while (!Thread.currentThread().isInterrupted()) {
        try {
            // work
            Thread.sleep(100);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();    // restore the flag
            return;                                  // exit cleanly
        }
    }
}
```

**Never use `Thread.stop()`** (deprecated). It forcibly terminates the thread, releasing all locks. Data structures get left in inconsistent states. Removed from new APIs entirely.

### 💻 Code Example

```java
// Per-thread request context
public class RequestContext {
    private static final ThreadLocal<String> USER_ID = new ThreadLocal<>();

    public static void setUserId(String id) { USER_ID.set(id); }
    public static String getUserId() { return USER_ID.get(); }
    public static void clear() { USER_ID.remove(); }
}

// Cancellable task
public class CancellableTask implements Runnable {
    @Override
    public void run() {
        try {
            while (!Thread.currentThread().isInterrupted()) {
                doWork();
                Thread.sleep(500);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();    // restore flag
            System.out.println("task interrupted — exiting");
        }
    }
}

Thread t = new Thread(new CancellableTask());
t.start();
Thread.sleep(2000);
t.interrupt();     // request cancellation
```

### ⚠️ Common Pitfalls
- Using `ThreadLocal` in a thread pool without `remove()` — memory leak.
- Catching `InterruptedException` and swallowing — the flag is cleared, cancellation is lost. ALWAYS either re-throw or restore with `Thread.currentThread().interrupt()`.
- Using boolean flag for cancellation (`volatile boolean stop`) — works for CPU-bound loops but not for threads blocked in `sleep/wait/take`. `interrupt()` wakes them up.
- `Thread.stop()` — don't. Ever.

### 🎤 Interview Angle
> "How do you safely cancel a running thread?"

Use `Thread.interrupt()`. The thread must cooperate by checking `isInterrupted()` regularly and propagating `InterruptedException` (or at minimum restoring the flag). Never use `Thread.stop()`.

> "What's the memory leak risk with ThreadLocal in a thread pool?"

The pool thread outlives the task. If the task sets a ThreadLocal and doesn't `remove()` it, the value lives as long as the thread does. In long-running pools this prevents the value (and anything it references) from being GC'd.

### 🧪 Practice
1. Build a request-scoped logger using `ThreadLocal` with proper cleanup.
2. Write a long-running loop that responds to interruption cleanly.
3. Demonstrate the `ThreadLocal` leak by spawning many tasks in a small pool without `remove()` — watch memory usage.

### 🔍 Self-check
1. What happens to the interrupt flag when `InterruptedException` is thrown?
2. Why is `Thread.stop()` deprecated?
3. How does `ThreadLocal` relate to thread pools and memory leaks?
4. What's the difference between `interrupted()` and `isInterrupted()`?

---

## Sunday — Revision & Q&A

### 🧭 Revision checklist
- [ ] Draw the Thread state diagram and list causes of each transition.
- [ ] Explain what `volatile` does and does NOT guarantee.
- [ ] Write a bounded buffer using wait/notify AND using ReentrantLock+Condition.
- [ ] List 3 memory leak causes with ThreadLocal and their fixes.
- [ ] Implement cancellable task with proper interrupt handling.

### 📝 Interview Q&A
Ask Claude: **"Prepare a Phase 1 Week 4 interview Q&A doc"** — answer, review, rate.

### 📂 Commit your work
```
phase-1/week-04/
├── exercises/
│   ├── ThreadCreationBenchmark.java
│   ├── Counter.java      (3 synchronization variants)
│   ├── VolatileDemo.java
│   ├── BoundedBuffer.java (wait/notify + ReentrantLock versions)
│   ├── TransferDeadlockFree.java
│   └── CancellableTask.java
└── notes.md
```
