# Week 5 — Advanced Concurrency & `java.util.concurrent`

**Theme:** Modern Java concurrency rarely uses raw threads. The `java.util.concurrent` (JUC) package provides production-grade abstractions: thread pools, futures, concurrent collections, atomics. This week moves you from "I know what a thread is" to "I can design concurrent code."

**Companion reading:** *Java Concurrency in Practice* — Chapters 5–8.

---

## Monday — Executor Framework & ThreadPoolExecutor

### 🎯 Objective
Choose the right thread pool configuration for CPU-bound and IO-bound workloads; handle task rejection.

### 📖 Core Concept

**The Executor framework decouples task submission from thread management.** Instead of `new Thread(task).start()`, you submit tasks to a pool.

**Interface hierarchy:**
```
Executor                    — single method: execute(Runnable)
└── ExecutorService         — submit() with Future, shutdown lifecycle
    └── ScheduledExecutorService — schedule() / scheduleAtFixedRate()
```

**Factory methods (`Executors`):**
```java
Executors.newFixedThreadPool(4);          // fixed size
Executors.newCachedThreadPool();          // elastic, 0..∞ threads, 60s idle timeout
Executors.newSingleThreadExecutor();      // single worker, serialized execution
Executors.newScheduledThreadPool(2);      // for delayed/periodic tasks
Executors.newWorkStealingPool();          // Java 8+, ForkJoinPool
Executors.newVirtualThreadPerTaskExecutor(); // Java 21+
```

**⚠️ Factory methods have traps:**
- `newFixedThreadPool` uses an **unbounded** `LinkedBlockingQueue` — memory leak if producers outpace consumers.
- `newCachedThreadPool` has **Integer.MAX_VALUE** max threads — can DoS the JVM.

**Production-safe: construct `ThreadPoolExecutor` explicitly:**
```java
ExecutorService pool = new ThreadPoolExecutor(
    4,                                          // corePoolSize
    8,                                          // maximumPoolSize
    60, TimeUnit.SECONDS,                       // keepAliveTime for extra threads
    new ArrayBlockingQueue<>(100),              // bounded queue
    new ThreadFactoryBuilder().setNameFormat("worker-%d").build(),
    new ThreadPoolExecutor.CallerRunsPolicy()   // backpressure
);
```

**Task-to-thread assignment:**
1. Threads < corePoolSize → create new thread.
2. Queue has space → enqueue.
3. Threads < maxPoolSize → create new thread.
4. Else → invoke `RejectedExecutionHandler`.

**Rejection handlers:**
- `AbortPolicy` (default) — throws `RejectedExecutionException`.
- `CallerRunsPolicy` — caller thread runs the task (natural backpressure).
- `DiscardPolicy` — silently drop.
- `DiscardOldestPolicy` — drop oldest queued.

**Sizing guidelines:**
- **CPU-bound**: `N + 1` threads where N = #cores.
- **IO-bound**: `N × (1 + wait/compute ratio)`. Usually 2N–100N depending on blocking.
- **Mixed**: break tasks into CPU and IO stages; use separate pools.

**Shutdown lifecycle:**
- `shutdown()` — no new tasks; finish in-flight.
- `shutdownNow()` — attempt to interrupt running tasks; return pending.
- `awaitTermination(timeout, unit)` — wait for pool to fully stop.
- **ALWAYS** shut down pools — otherwise non-daemon workers keep the JVM alive.

### 💻 Code Example

```java
ExecutorService pool = new ThreadPoolExecutor(
    4, 8,
    60, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(100),
    Executors.defaultThreadFactory(),
    new ThreadPoolExecutor.CallerRunsPolicy()
);

try {
    for (int i = 0; i < 1000; i++) {
        final int id = i;
        pool.submit(() -> process(id));
    }
} finally {
    pool.shutdown();
    if (!pool.awaitTermination(30, TimeUnit.SECONDS)) {
        pool.shutdownNow();
    }
}

// Java 19+ try-with-resources
try (ExecutorService ex = Executors.newFixedThreadPool(4)) {
    ex.submit(() -> ...);
}
```

### ⚠️ Common Pitfalls
- Using `newFixedThreadPool` / `newCachedThreadPool` in production without understanding the unbounded queue/thread count.
- Forgetting to shutdown — JVM hangs.
- Using the same pool for CPU-bound and IO-bound tasks — slow IO task blocks a thread that could be doing CPU work.

### 🎤 Interview Angle
> "Why is `Executors.newFixedThreadPool` dangerous in production?"

Unbounded `LinkedBlockingQueue` — if submissions outpace completion, queue grows until OOM. Always use a bounded queue.

### 🧪 Practice
1. Configure a `ThreadPoolExecutor` with a bounded queue and `CallerRunsPolicy`. Show backpressure by submitting 10k fast tasks.
2. Size a CPU-bound workload correctly; benchmark 1x, 2x, 4x core count.
3. Build a shutdown helper that shuts down → awaits → `shutdownNow` if stuck.

### 🔍 Self-check
1. What's the task-submission decision tree for `ThreadPoolExecutor`?
2. What's wrong with `Executors.newFixedThreadPool` in production?
3. When would you use `CallerRunsPolicy`?
4. What does `awaitTermination` actually wait for?

---

## Tuesday — Callable, Future, CompletableFuture Composition

### 🎯 Objective
Chain async operations using `CompletableFuture` without blocking; handle exceptions in pipelines.

### 📖 Core Concept

**`Future<V>`** — handle to an async computation.
- `V get()` — blocks until done.
- `V get(timeout, unit)` — blocks up to timeout.
- `cancel(boolean mayInterrupt)`.
- `isDone()`, `isCancelled()`.

**Limitations of `Future`:** can't compose; must `get()` to use the result (blocking); no callback.

**`CompletableFuture<V>` (Java 8+)** — fixes all of that. Represents a value that may be available later, plus a rich composition API.

**Creation:**
```java
CompletableFuture.completedFuture(42);                // already-complete
CompletableFuture.runAsync(() -> { /* void */ });     // Runnable
CompletableFuture.supplyAsync(() -> fetchUser(id));   // Supplier<T>
```

By default, `*Async` methods run on the common `ForkJoinPool`. Pass your own `Executor` for isolation.

**Transformation:**
```java
CompletableFuture<Integer> ageFuture = userFuture
    .thenApply(User::getAge);              // sync transform
    // .thenApplyAsync(...)                // on an executor
```

**Chaining (flat-map):**
```java
CompletableFuture<Profile> profileFuture = userFuture
    .thenCompose(user -> fetchProfileAsync(user.getId()));
    // thenCompose flattens — no nested CompletableFuture<CompletableFuture<...>>
```

**Combining multiple:**
```java
CompletableFuture<String> combined = userFuture
    .thenCombine(profileFuture, (user, profile) ->
        user.getName() + ": " + profile.getBio());

CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2, f3);
all.thenRun(() -> System.out.println("all done"));

CompletableFuture<Object> anyOne = CompletableFuture.anyOf(f1, f2, f3);
```

**Exception handling:**
```java
future
    .exceptionally(ex -> { log(ex); return fallback; })  // recover
    .handle((result, ex) -> ex != null ? fallback : result)  // both paths
    .whenComplete((result, ex) -> log(result, ex));      // observe, don't transform
```

**Timeouts (Java 9+):**
```java
future.orTimeout(5, TimeUnit.SECONDS);             // TimeoutException
future.completeOnTimeout(default, 5, TimeUnit.SECONDS);  // fallback value
```

### 💻 Code Example

```java
CompletableFuture<String> pipeline = CompletableFuture
    .supplyAsync(() -> fetchUserId())
    .thenCompose(id -> fetchUserAsync(id))             // async chain
    .thenCombine(
        CompletableFuture.supplyAsync(() -> fetchPermissions()),
        (user, perms) -> new UserWithPerms(user, perms))
    .thenApply(uwp -> "Hello " + uwp.name())
    .exceptionally(ex -> "Error: " + ex.getMessage())
    .orTimeout(5, TimeUnit.SECONDS);

String result = pipeline.join();    // join() doesn't throw checked exceptions
```

### ⚠️ Common Pitfalls
- Using `get()` inside a transformation (e.g., `thenApply(u -> fetchAsync(u).get())`) — blocks the thread. Use `thenCompose` instead.
- Forgetting that the common ForkJoinPool is shared across the JVM — slow tasks starve other parallel operations. Pass your own `Executor` for non-trivial workloads.
- Losing exceptions — a future that's never observed (no `get`/`join`/`whenComplete`) can silently swallow errors.

### 🎤 Interview Angle
> "Difference between `thenApply` and `thenCompose`?"

`thenApply` — takes `Function<T, R>`, returns `CompletableFuture<R>`.
`thenCompose` — takes `Function<T, CompletableFuture<R>>`, returns `CompletableFuture<R>` (flattened).

If your function itself returns a CompletableFuture, use `thenCompose` to avoid nested `CompletableFuture<CompletableFuture<R>>`.

### 🧪 Practice
1. Build a service that fetches a user, then their orders, then their order details — all async, no blocking between stages.
2. Add timeout and fallback with `orTimeout` + `exceptionally`.
3. Fan-out to 10 parallel async calls with `allOf`, aggregate results.

### 🔍 Self-check
1. What's the default executor for `*Async` methods without an Executor argument?
2. When do you use `thenCompose` vs `thenApply`?
3. Difference between `exceptionally`, `handle`, and `whenComplete`?
4. What's the risk of using the common ForkJoinPool for IO-heavy work?

---

## Wednesday — ForkJoinPool, Work-Stealing, Parallel Streams

### 🎯 Objective
Implement a divide-and-conquer task with ForkJoin; understand when parallel streams help (and when they hurt).

### 📖 Core Concept

**`ForkJoinPool`** — specialized pool for divide-and-conquer parallelism with **work-stealing** scheduling.

**Work-stealing:** each worker has its own deque of tasks. When a task splits (`fork`), children go on the local deque. When a worker's deque empties, it **steals** from another worker's deque tail. Balances load automatically.

**Base classes:**
- `RecursiveTask<V>` — returns V.
- `RecursiveAction` — void.

**Pattern:**
```java
class SumTask extends RecursiveTask<Long> {
    final long[] arr; final int lo, hi;
    static final int THRESHOLD = 10_000;

    public SumTask(long[] arr, int lo, int hi) {
        this.arr = arr; this.lo = lo; this.hi = hi;
    }

    @Override
    protected Long compute() {
        if (hi - lo <= THRESHOLD) {
            long sum = 0;
            for (int i = lo; i < hi; i++) sum += arr[i];
            return sum;
        }
        int mid = (lo + hi) >>> 1;
        SumTask left  = new SumTask(arr, lo, mid);
        SumTask right = new SumTask(arr, mid, hi);
        left.fork();                       // run async on another worker
        long rightResult = right.compute();// compute current thread
        long leftResult  = left.join();    // wait for left
        return leftResult + rightResult;
    }
}

long total = ForkJoinPool.commonPool().invoke(new SumTask(data, 0, data.length));
```

**The common pool** — `ForkJoinPool.commonPool()`. Size defaults to `#cores - 1`. Used by `parallelStream()`, default CompletableFuture executor, and any code that doesn't provide one. **A slow task in common pool affects the whole JVM** — use your own pool for isolation.

**Parallel streams:**
```java
long sum = LongStream.of(data).parallel().sum();
```

Backed by the common ForkJoinPool. Benefits from:
- Large datasets (millions of elements).
- CPU-bound, stateless operations.
- Work that's embarrassingly parallel.

Hurts when:
- Small datasets (fork/join overhead dominates).
- IO-bound ops (thread blocking starves the pool).
- Stateful/ordered operations (synchronization costs).
- Inside already-parallel code (common-pool contention).

### 💻 Code Example

```java
// Parallel file processing with a dedicated pool
ForkJoinPool pool = new ForkJoinPool(8);
try {
    List<Path> files = Files.walk(root).filter(Files::isRegularFile).toList();
    long total = pool.submit(() ->
        files.parallelStream()
            .mapToLong(this::processFile)
            .sum()
    ).get();
} finally {
    pool.shutdown();
}
```

### ⚠️ Common Pitfalls
- Using `parallelStream()` on small collections or IO-heavy operations — usually slower.
- Shared mutable state in parallel stream pipelines — race conditions.
- Using `forEach` with side effects in parallel — ordering lost, races possible. Use `collect` or `reduce` instead.

### 🎤 Interview Angle
> "What's work-stealing and why does it help?"

Each worker has its own task deque. When a worker splits a task, the subtasks go on its local deque (fast, no contention). When a worker is idle, it steals from a busy worker's deque tail. Result: near-automatic load balancing even for irregular workloads.

### 🧪 Practice
1. Implement parallel merge sort with ForkJoin.
2. Benchmark `stream().sum()` vs `parallelStream().sum()` on arrays of size 1K, 100K, 10M.
3. Find a case where parallel stream is slower — explain why.

### 🔍 Self-check
1. What's the threshold rule of thumb in ForkJoin divide-and-conquer?
2. Why is the common ForkJoinPool dangerous for IO work?
3. When does `parallelStream()` actually speed things up?
4. How do you submit a parallel stream to a non-default pool?

---

## Thursday — Concurrent Collections

### 🎯 Objective
Select the right concurrent collection; explain ConcurrentHashMap's internals.

### 📖 Core Concept

**`ConcurrentHashMap`** — thread-safe hash map for high concurrency.

Java 8+ implementation:
- Reads are lock-free (volatile reads).
- Writes acquire `synchronized` on the head node of the affected bucket.
- Resizing is performed concurrently in chunks across threads.
- Treeification applied same as `HashMap` (>8 entries, capacity ≥64).
- **Does NOT permit null keys or null values** (unlike HashMap).

Atomic bulk ops: `computeIfAbsent`, `compute`, `merge` are atomic per key.
```java
ConcurrentHashMap<String, Integer> counts = new ConcurrentHashMap<>();
counts.merge("apple", 1, Integer::sum);   // atomic increment
```

**`CopyOnWriteArrayList`** — every mutation creates a new copy of the backing array. Reads are lock-free. Writes are expensive.
- Best for read-mostly lists (event listeners, config).
- **Iteration sees a snapshot** — never throws `ConcurrentModificationException`.

**`BlockingQueue<E>`** — thread-safe queue with blocking semantics.

| Operation | Throws | Returns special | Blocks | Times out |
|-----------|--------|-----------------|--------|-----------|
| Insert | `add(e)` | `offer(e)` | `put(e)` | `offer(e, time, unit)` |
| Remove | `remove()` | `poll()` | `take()` | `poll(time, unit)` |

Implementations:
- `ArrayBlockingQueue` — bounded, single-lock, predictable.
- `LinkedBlockingQueue` — optionally bounded, two locks (put/take) → higher throughput.
- `SynchronousQueue` — zero capacity, direct handoff. Used by `newCachedThreadPool`.
- `PriorityBlockingQueue` — unbounded, priority-ordered.
- `DelayQueue` — elements available only after their delay expires.
- `LinkedTransferQueue` — SynchronousQueue + LinkedBlockingQueue hybrid.

**`ConcurrentLinkedQueue`** — lock-free unbounded queue using CAS. Non-blocking. Use when `BlockingQueue`'s blocking semantics aren't needed.

**`ConcurrentSkipListMap` / `ConcurrentSkipListSet`** — concurrent sorted map/set (SortedMap/SortedSet). Concurrent analog of TreeMap/TreeSet.

### 💻 Code Example

```java
// Concurrent cache with atomic load-if-absent
ConcurrentHashMap<String, User> cache = new ConcurrentHashMap<>();

public User getUser(String id) {
    return cache.computeIfAbsent(id, this::loadFromDb);
    // If 100 threads call with same id concurrently,
    // loadFromDb runs ONCE; others wait and get the result.
}

// Listener list — read-heavy
CopyOnWriteArrayList<Listener> listeners = new CopyOnWriteArrayList<>();
public void addListener(Listener l) { listeners.add(l); }
public void fireEvent(Event e) {
    for (Listener l : listeners) l.on(e);  // iterator is a snapshot, safe
}

// Producer-consumer with BlockingQueue
BlockingQueue<Task> queue = new ArrayBlockingQueue<>(100);
Runnable producer = () -> { while (running) queue.put(generate()); };
Runnable consumer = () -> { while (running) process(queue.take()); };
```

### ⚠️ Common Pitfalls
- Using `Collections.synchronizedMap(new HashMap<>())` expecting high concurrency — it uses a single lock on all ops. `ConcurrentHashMap` is far more scalable.
- Using `ConcurrentHashMap.get() != null` as a "contains" check — race if another thread puts null... wait, CHM doesn't allow null. But `map.get(k) != null` is still wrong for "exists and is non-null" without care.
- `CopyOnWriteArrayList` for write-heavy workloads — O(n) copy per write is brutal.
- Iterating a `ConcurrentHashMap` and expecting a consistent snapshot — iteration is weakly consistent: reflects state at some point during iteration, may see concurrent modifications.

### 🎤 Interview Angle
> "How is ConcurrentHashMap thread-safe without synchronizing everything?"

Java 8+: fine-grained locking — `synchronized` on the **head node** of each bucket during writes. Reads use `volatile` reads, no locking. CAS for contention-free inserts on empty buckets. Resizing is cooperative (multiple threads help move chunks).

> "Why doesn't ConcurrentHashMap allow null values?"

To avoid ambiguity in `map.get(k) == null` — did the key not exist, or was the value null? In a concurrent context, you can't disambiguate without the value being in-band (null means absent). `containsKey` would race with `get`.

### 🧪 Practice
1. Build a concurrent counter map using `merge`.
2. Measure throughput: `Hashtable` vs `Collections.synchronizedMap(new HashMap<>())` vs `ConcurrentHashMap` at 1/10/50 threads.
3. Build a producer-consumer with `BlockingQueue` and run 4 producers × 4 consumers.

### 🔍 Self-check
1. Why doesn't `ConcurrentHashMap` allow null keys or values?
2. When is `CopyOnWriteArrayList` a good choice?
3. What's the difference between `SynchronousQueue` and `ArrayBlockingQueue`?
4. What's "weakly consistent" iteration?

---

## Friday — Atomic Classes, CAS, LongAdder

### 🎯 Objective
Use atomic classes for lock-free counters; understand CAS and its retry cost; know when `LongAdder` beats `AtomicLong`.

### 📖 Core Concept

**Compare-And-Swap (CAS)** — a CPU instruction that atomically compares a memory location with an expected value and, only if they match, writes a new value. Returns success/failure.

```
CAS(address, expected, new):
    if *address == expected:
        *address = new
        return true
    else:
        return false
```

**Atomic classes** in `java.util.concurrent.atomic` use CAS + `volatile` internally to provide lock-free updates:

| Class | Purpose |
|-------|---------|
| `AtomicInteger` / `AtomicLong` | integer counters |
| `AtomicBoolean` | atomic flag |
| `AtomicReference<V>` | atomic reference |
| `AtomicIntegerArray` | each element CAS-able |
| `AtomicStampedReference` | version-aware (avoids ABA problem) |
| `LongAdder` / `DoubleAdder` | high-contention counters |
| `LongAccumulator` | general associative accumulation |

**Canonical use:**
```java
AtomicInteger counter = new AtomicInteger();
counter.incrementAndGet();                 // atomic ++
counter.compareAndSet(5, 10);              // if 5, set to 10

AtomicReference<List<String>> listRef = new AtomicReference<>(List.of());
listRef.updateAndGet(old -> {
    List<String> updated = new ArrayList<>(old);
    updated.add("x");
    return List.copyOf(updated);
});
```

**CAS retry cost:** on contention, the CAS fails and the operation retries. Under heavy contention, this is a spin-loop — wastes CPU. For high-contention counters, use `LongAdder` instead.

**`LongAdder`** — maintains multiple internal cells, each updated independently with CAS. `sum()` aggregates across cells. Trade-off: faster writes but slower reads. Perfect for counters (stats, metrics) where writes dominate.

**ABA problem:** CAS(addr, A, B) succeeds if someone else went A→X→A between your read and your CAS. Usually fine for counters; dangerous for reference-based structures (e.g., node recycling in a lock-free stack). Solution: `AtomicStampedReference` adds a version counter.

### 💻 Code Example

```java
// Simple atomic counter
AtomicLong totalRequests = new AtomicLong();
public void onRequest() { totalRequests.incrementAndGet(); }

// High-contention counter
LongAdder hits = new LongAdder();
public void onHit() { hits.increment(); }
public long totalHits() { return hits.sum(); }   // only when needed

// CAS-based lock-free stack
class Stack<T> {
    private final AtomicReference<Node<T>> head = new AtomicReference<>();
    private static class Node<T> { T value; Node<T> next; }

    public void push(T value) {
        Node<T> n = new Node<>();
        n.value = value;
        Node<T> old;
        do {
            old = head.get();
            n.next = old;
        } while (!head.compareAndSet(old, n));
    }

    public T pop() {
        Node<T> old, next;
        do {
            old = head.get();
            if (old == null) return null;
            next = old.next;
        } while (!head.compareAndSet(old, next));
        return old.value;
    }
}
```

### ⚠️ Common Pitfalls
- Using `AtomicLong` for hot counters — contention makes it slower than `LongAdder`.
- Forgetting CAS is weak — needs retry loop for correctness.
- Writing multi-variable "atomic" updates with atomics — you need `AtomicReference<State>` holding an immutable state record, or use a lock.
- The ABA problem — use `AtomicStampedReference` when the value can cycle.

### 🎤 Interview Angle
> "When would you use LongAdder over AtomicLong?"

For write-heavy counters (metrics, request counts, stats) where you increment often and read rarely. `LongAdder` distributes writes across cells to avoid CAS contention. Slightly more memory; much better write throughput.

> "What's the ABA problem?"

Thread A reads value X. Thread B changes X→Y→X. Thread A's CAS(X, new) succeeds even though the state changed. Usually benign for counters; deadly for pointer-based structures where the "same" pointer can point to recycled memory.

### 🧪 Practice
1. Benchmark `AtomicLong` vs `LongAdder` under 1/10/50 thread contention.
2. Implement a lock-free stack using `AtomicReference`.
3. Construct an ABA scenario using `AtomicReference` and solve it with `AtomicStampedReference`.

### 🔍 Self-check
1. What does CAS stand for and what does it guarantee?
2. Why is `LongAdder` faster than `AtomicLong` under contention?
3. What's the ABA problem?
4. What's the return type of `compareAndSet`?

---

## Saturday — Synchronizers: CountDownLatch, CyclicBarrier, Semaphore, Phaser

### 🎯 Objective
Know which synchronizer fits which coordination pattern.

### 📖 Core Concept

**`CountDownLatch(n)`** — one-shot gate. Threads call `await()` and block until `countDown()` has been called n times. Cannot be reset.

Use: wait for N tasks to complete before proceeding.

```java
CountDownLatch startGate = new CountDownLatch(1);
CountDownLatch doneGate  = new CountDownLatch(n);
for (int i = 0; i < n; i++) {
    new Thread(() -> {
        try {
            startGate.await();             // wait for the "go" signal
            work();
        } finally { doneGate.countDown(); }
    }).start();
}
startGate.countDown();     // everyone starts simultaneously
doneGate.await();          // wait for all to finish
```

**`CyclicBarrier(n, action)`** — reusable rendezvous. N threads call `await()`; when all arrive, optional action runs, and the barrier resets.

Use: multi-phase parallel algorithms (simulation step, matrix multiply, MapReduce).

```java
CyclicBarrier barrier = new CyclicBarrier(4, () -> System.out.println("phase complete"));
// Each thread does a chunk of phase-1 work, then:
barrier.await();           // blocks until all 4 arrive
// Then phase-2 work, barrier.await() again, etc.
```

**`Semaphore(n)`** — classical counting semaphore. `acquire()` decrements; `release()` increments. `acquire()` blocks if count is 0.

Use: bound concurrent access to a resource (connection pool, rate limiter).

```java
Semaphore perms = new Semaphore(10);     // max 10 concurrent calls
public Result callApi() throws InterruptedException {
    perms.acquire();
    try { return api.call(); }
    finally { perms.release(); }
}
```

**`Phaser`** — like `CyclicBarrier` but with dynamic registration (threads can join/leave phases). Good for dynamic parallelism.

**`Exchanger<V>`** — two threads meet and swap objects.

### 💻 Code Example

```java
// Race simulation — all start at the same instant
int runners = 10;
CountDownLatch go = new CountDownLatch(1);
CountDownLatch finished = new CountDownLatch(runners);
long[] times = new long[runners];

for (int i = 0; i < runners; i++) {
    final int id = i;
    new Thread(() -> {
        try {
            go.await();
            long start = System.nanoTime();
            run();
            times[id] = System.nanoTime() - start;
        } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        finally { finished.countDown(); }
    }).start();
}

Thread.sleep(1000);  // let all threads park at the gate
go.countDown();      // start!
finished.await();    // wait for all
```

### ⚠️ Common Pitfalls
- Using `CountDownLatch` when you want to coordinate repeatedly — you can't reset it. Use `CyclicBarrier` or `Phaser`.
- Forgetting to release a semaphore in `finally` — permanent permit leak.
- Using `Semaphore` to implement mutual exclusion for mutable state — prefer a lock (semaphore doesn't track owner, can't detect misuse).

### 🎤 Interview Angle
> "What's the difference between CountDownLatch and CyclicBarrier?"

| | CountDownLatch | CyclicBarrier |
|---|---|---|
| Reusable | No | Yes |
| Threads awaiting | Different counts (any number) | Exactly N |
| Who counts down | Anyone | The threads themselves by calling await() |
| Action on completion | No built-in | Optional Runnable |

### 🧪 Practice
1. Use `CountDownLatch` to orchestrate a race-start (above).
2. Use `CyclicBarrier` for a multi-phase parallel computation (e.g., matrix step).
3. Use `Semaphore` to bound concurrent HTTP calls to 10.

### 🔍 Self-check
1. Can `CountDownLatch` be reused?
2. What happens if one thread in a `CyclicBarrier` is interrupted?
3. Why is `Semaphore` different from a `ReentrantLock` with multi-permit?
4. When would you use `Phaser` over `CyclicBarrier`?

---

## Sunday — Revision & Q&A

### 🧭 Revision checklist
- [ ] Draw the `ThreadPoolExecutor` submission decision tree.
- [ ] Chain 3 async operations with CompletableFuture (no blocking).
- [ ] Implement divide-and-conquer with ForkJoin.
- [ ] Compare AtomicLong vs LongAdder under contention.
- [ ] List the 4 main synchronizers and one use case for each.

### 📝 Interview Q&A
Ask Claude: **"Prepare a Phase 1 Week 5 interview Q&A doc"** — answer, review, rate.

### 📂 Commit your work
```
phase-1/week-05/
├── exercises/
│   ├── TunedThreadPool.java
│   ├── AsyncPipeline.java       (CompletableFuture chain)
│   ├── ParallelMergeSort.java   (ForkJoin)
│   ├── ConcurrentCache.java     (ConcurrentHashMap.computeIfAbsent)
│   ├── LockFreeStack.java       (CAS)
│   ├── RaceStart.java           (CountDownLatch)
│   └── BoundedApiCaller.java    (Semaphore)
└── notes.md
```
