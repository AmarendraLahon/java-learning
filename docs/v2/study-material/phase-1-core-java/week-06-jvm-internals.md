# Week 6 — JVM Internals & Memory

**Theme:** Understanding the JVM is the line between mid-level and senior Java engineers. This week covers class loading, runtime memory, garbage collection, and the tooling you need to diagnose production problems.

---

## Monday — JVM Architecture & Class Loaders

### 🎯 Objective
Draw the JVM architecture from memory; explain parent delegation and why it matters.

### 📖 Core Concept

**JVM subsystems:**

```
┌────────────────────────────────────────────────┐
│ Class Loader Subsystem                         │
│ (Loading → Linking → Initialization)           │
└────────────────────────────────────────────────┘
                    ↓
┌────────────────────────────────────────────────┐
│ Runtime Data Areas                             │
│ ┌──────────┐ ┌───────┐ ┌──────────────────┐    │
│ │ Heap     │ │ Meta- │ │ Stack (per       │    │
│ │          │ │ space │ │   thread)        │    │
│ └──────────┘ └───────┘ └──────────────────┘    │
│ ┌──────────────────┐ ┌─────────────────────┐   │
│ │ PC Register      │ │ Native Method Stack │   │
│ │ (per thread)     │ │ (per thread)        │   │
│ └──────────────────┘ └─────────────────────┘   │
└────────────────────────────────────────────────┘
                    ↓
┌────────────────────────────────────────────────┐
│ Execution Engine                               │
│ (Interpreter + JIT + Garbage Collector)        │
└────────────────────────────────────────────────┘
                    ↓
┌────────────────────────────────────────────────┐
│ JNI → Native Libraries                         │
└────────────────────────────────────────────────┘
```

**Class loader hierarchy (parent delegation):**

1. **Bootstrap ClassLoader** — native C/C++ code inside the JVM. Loads core `java.*` classes from `$JAVA_HOME/lib` (pre-Java 9 `rt.jar`, Java 9+ modular runtime). Its parent is `null`. `Object.class.getClassLoader()` returns `null`.
2. **Platform ClassLoader** (pre-Java 9: Extension) — loads platform modules like `java.sql`, `java.xml`.
3. **Application (System) ClassLoader** — loads user classes from the classpath.

**Parent delegation model:**
When a class loader is asked to load class X:
1. Delegate to **parent** first.
2. If parent can't find X, the child tries.

**Why delegation matters:**
- **Security** — you can't spoof `java.lang.String` by writing your own. Bootstrap always gets first dibs.
- **Uniqueness** — core classes loaded exactly once.
- **Namespaces** — the same class name loaded by different loaders produces **different runtime types** (how app servers isolate web apps).

**Custom class loaders** — useful for hot reloading, plugin systems, loading from databases/network. Override `findClass(String name)`, keep delegation intact.

### 💻 Code Example

```java
// Inspect the class loader chain
ClassLoader cl = Example.class.getClassLoader();
while (cl != null) {
    System.out.println(cl);            // AppClassLoader, PlatformClassLoader, ...
    cl = cl.getParent();
}
// Object.class.getClassLoader() returns null — Bootstrap loaded it

// Custom class loader loading from bytes
class MemoryClassLoader extends ClassLoader {
    private final Map<String, byte[]> classBytes;
    MemoryClassLoader(Map<String, byte[]> classBytes) { this.classBytes = classBytes; }
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] bytes = classBytes.get(name);
        if (bytes == null) throw new ClassNotFoundException(name);
        return defineClass(name, bytes, 0, bytes.length);
    }
}
```

### ⚠️ Common Pitfalls
- Calling `Class.forName(name)` instead of `loadClass(name)` — `forName` also **initializes** the class (runs static blocks). For mere loading, use `loadClass`.
- Writing a custom class loader that overrides `loadClass` instead of `findClass` — you break parent delegation and may reload core classes.
- Confusing `ClassNotFoundException` (thrown by explicit load attempts) with `NoClassDefFoundError` (thrown by JVM when a class present at compile is missing at runtime).

### 🎤 Interview Angle
> "Can two classes with the same fully-qualified name coexist in one JVM?"

Yes, if loaded by different class loaders. They're considered distinct types — casting one to the other fails. This is how Tomcat isolates web applications: each WAR has its own class loader.

### 🧪 Practice
1. Print the class loader hierarchy for `String`, `java.sql.Connection`, and one of your own classes.
2. Write a custom class loader that loads `.class` bytes from a `Map<String, byte[]>`.
3. Trigger `NoClassDefFoundError` by compiling with one classpath and running with another.

### 🔍 Self-check
1. Why does `Object.class.getClassLoader()` return null?
2. What's the difference between `Class.forName` and `ClassLoader.loadClass`?
3. What's the purpose of parent delegation?
4. How do app servers isolate web applications?

---

## Tuesday — Runtime Data Areas

### 🎯 Objective
Explain what each memory region stores; predict where a given variable or object lives.

### 📖 Core Concept

**Heap** — shared across all threads. All objects and arrays live here.
- Divided into **Young Generation** (Eden + 2 Survivor spaces) and **Old Generation (Tenured)**.
- New objects go to Eden. Survive a few Minor GCs → promoted to Old.
- `-Xms` (initial size), `-Xmx` (max size). OOM: `java.lang.OutOfMemoryError: Java heap space`.

**Method Area / Metaspace** — shared.
- Pre-Java 8: **PermGen** (part of heap, fixed size, caused OOM in app servers).
- Java 8+: **Metaspace** — in native memory, auto-grows.
- Holds class metadata (fields, methods, bytecode), runtime constant pool, static fields, JIT-compiled code.

**JVM Stack** — one per thread.
- Stores **stack frames**: one per method call, holds local variables, operand stack, return address.
- `-Xss` controls size. OOM: `StackOverflowError`.
- Primitive locals and object references live here, NOT the objects themselves.

**PC Register** — one per thread. Holds the address of the currently executing bytecode instruction.

**Native Method Stack** — one per thread. Used for native (JNI) method calls.

**Code Cache** (not a JVM spec area, but HotSpot): stores JIT-compiled machine code.

### 📐 Memory location of a variable

```java
public class Example {
    private static int sharedCount;        // Metaspace (static field)
    private String name;                    // Heap (inside the Example instance)

    public void method() {
        int local = 10;                     // Stack (local primitive)
        Example other = new Example();      // Object on Heap; reference on Stack
        String constant = "hello";          // Pooled String in Heap (since Java 7)
    }
}
```

**Object header layout (approximate):**
```
+-----------------------+
| Mark Word (8 bytes)   | — hash, GC age, lock state
+-----------------------+
| Class Pointer (4/8)   | — points to class metadata
+-----------------------+
| Fields (aligned)      |
+-----------------------+
```

Typical small object: 16–24 bytes overhead, even for an empty class.

### 💻 Code Example

```java
public class MemoryDemo {
    static int shared = 1;          // Metaspace

    public static void main(String[] args) {
        int x = 5;                   // Stack
        String s = "hello";          // Literal pooled in heap
        int[] arr = new int[100];    // Array on heap; reference on stack
        recurse(0);
    }

    static void recurse(int n) {
        recurse(n + 1);              // StackOverflowError eventually
    }
}
```

### ⚠️ Common Pitfalls
- Assuming primitives are always on the stack — they are when declared as local variables, but `int` fields of an object live in the heap inside the object.
- Forgetting that string literals are pooled in the heap since Java 7 (not PermGen).
- Confusing `StackOverflowError` (thread's stack) with `OutOfMemoryError` (heap or metaspace).

### 🎤 Interview Angle
> "Where does `new String(\"abc\")` live vs `\"abc\"`?"

`"abc"` literal → string pool (heap, dedicated region).
`new String("abc")` → a new String object in the heap, **outside** the pool, but its char array might still reference the pool's char data (since Java 8+, String is `char[]` based; since Java 9, `byte[]` with compact strings).

### 🧪 Practice
1. Write a program that triggers `StackOverflowError`. Change `-Xss` and observe.
2. Write a program that triggers `OutOfMemoryError: Java heap space`.
3. Compute the object header size using a library (JOL — Java Object Layout).

### 🔍 Self-check
1. Where does a local `int x = 5` live?
2. Where do static fields live (Java 8+)?
3. What does `-Xss` control?
4. Why did PermGen get replaced with Metaspace?

---

## Wednesday — Bytecode & Execution Engine

### 🎯 Objective
Read `javap -c` output; explain how the interpreter and JIT cooperate.

### 📖 Core Concept

Java source → **bytecode** (.class file) → JVM interprets or JIT-compiles.

**Bytecode** is a stack-based instruction set. Key categories:
- `iload_0`, `iload_1` — push local variable onto stack.
- `istore_0`, `istore_1` — pop stack into local variable.
- `iadd`, `imul`, `isub`, `idiv` — arithmetic on stack.
- `invokevirtual`, `invokestatic`, `invokespecial`, `invokeinterface`, `invokedynamic` — call methods.
- `new`, `dup`, `astore` — object creation.
- `ifeq`, `if_icmplt`, `goto` — branches.

**Execution engine:**

1. **Interpreter** — reads bytecode instruction-by-instruction. Starts immediately but slow.
2. **JIT compiler** — compiles hot methods to native code. HotSpot uses **tiered compilation**:
   - Tier 0: interpreter.
   - Tier 1–3: **C1 (client)** — fast compilation, basic optimizations.
   - Tier 4: **C2 (server)** — slower compilation, aggressive optimizations.
3. **Garbage collector** — reclaims unreachable objects.

**JIT optimizations:**
- **Method inlining** — replaces calls with the method body.
- **Escape analysis** — if an object doesn't escape its method, allocate on stack (no GC).
- **Dead code elimination.**
- **Loop unrolling, vectorization (SIMD).**
- **Null check elision** when safe.
- **Speculative optimization** — optimizes for the observed common case; **deoptimizes** if assumptions violated.

**AOT (Ahead-Of-Time) compilation** — GraalVM Native Image compiles Java to a native binary. Fast startup, low memory. Trade-off: no runtime JIT adaptivity.

### 💻 Code Example — read bytecode

```java
public class Demo {
    public int add(int a, int b) { return a + b; }
}
```

```bash
$ javap -c Demo.class
public int add(int, int);
  Code:
     0: iload_1         // push a
     1: iload_2         // push b
     2: iadd            // a + b
     3: ireturn         // return result
```

**A slightly more interesting one:**
```java
public int sum(int[] arr) {
    int s = 0;
    for (int x : arr) s += x;
    return s;
}
```
Produces ~20 bytecode instructions using `iaload`, `iadd`, `iinc`, `if_icmplt`.

### ⚠️ Common Pitfalls
- Assuming the interpreter always runs — JIT takes over for hot code after ~10k invocations.
- Misreading microbenchmarks because of JIT warmup. Use JMH (Java Microbenchmark Harness) for accurate measurements.
- Relying on escape analysis to eliminate allocations — the JIT may or may not do it. Measure.

### 🎤 Interview Angle
> "What's the difference between interpreter and JIT?"

Interpreter reads bytecode one instruction at a time — easy to start, slow. JIT compiles hot bytecode to native machine code — expensive to compile, fast to execute. HotSpot uses both: interpret first, promote hot methods to C1, then the hottest to C2.

### 🧪 Practice
1. Write `Demo.add` and inspect bytecode with `javap -c`.
2. Turn on JIT logging: `-XX:+PrintCompilation`. Run a hot loop and watch compilation events.
3. Use JMH to benchmark with and without warmup — see the difference.

### 🔍 Self-check
1. How many operands does `iadd` take, and from where?
2. What's tiered compilation?
3. What's escape analysis?
4. Why is JMH needed instead of System.nanoTime benchmarks?

---

## Thursday — Garbage Collection Generations & Types

### 🎯 Objective
Explain why generational GC works; distinguish Minor, Major, and Full GC.

### 📖 Core Concept

**Generational hypothesis:** most objects die young. So treat young and old objects differently — collect young frequently (cheap), collect old rarely (expensive).

**Heap layout:**
```
┌───────────────────────────────────────────────┐
│ Young Generation                              │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐          │
│  │  Eden   │ │  S0     │ │  S1     │          │
│  └─────────┘ └─────────┘ └─────────┘          │
├───────────────────────────────────────────────┤
│ Old Generation (Tenured)                      │
└───────────────────────────────────────────────┘
```

**Allocation flow:**
1. New object → Eden.
2. Eden fills → **Minor GC** (stop-the-world, but fast):
   - Live objects moved from Eden to one Survivor space.
   - Older survivors move between S0/S1, incrementing **age counter**.
   - When age > `MaxTenuringThreshold` (default ~15), object **promoted** to Old gen.
3. Old gen fills → **Major GC** (Old gen only).
4. Both fill, or Metaspace fills → **Full GC** (entire heap + Metaspace).

**Three GC types (interview must-know):**

| Type | Scope | Frequency | Pause |
|------|-------|-----------|-------|
| **Minor GC** | Young gen | High | Short |
| **Major GC** | Old gen | Low | Long |
| **Full GC** | Young + Old + Metaspace | Rare | Longest |

> ⚠️ **Major GC ≠ Full GC.** Many blog posts conflate them. Major = Old only; Full = everything.

**GC roots** — starting points for reachability tracing:
- Active thread stacks (local variables, method parameters).
- Static fields of loaded classes.
- JNI global references.
- Monitors held.

**Finalization + reference types:**
- **Strong reference** — default, prevents GC.
- **Soft** (`SoftReference`) — cleared only if memory is low; good for caches.
- **Weak** (`WeakReference`) — cleared at next GC; backs `WeakHashMap`, `ThreadLocal` values.
- **Phantom** (`PhantomReference`) — never returns the object; notifies after GC.

### 💻 Code Example

```java
// GC root trace — why a seemingly unreachable object isn't GC'd
class Holder {
    static List<Object> leaks = new ArrayList<>();   // static = GC root!
}

Object o = new Object();
Holder.leaks.add(o);
o = null;        // 'o' is gone, BUT still reachable via Holder.leaks
System.gc();     // won't reclaim — the object is alive
```

**Weak reference caching:**
```java
WeakHashMap<Key, Value> cache = new WeakHashMap<>();
// If no strong reference to Key remains, GC reclaims BOTH key and value.
```

### ⚠️ Common Pitfalls
- Calling `System.gc()` and expecting immediate cleanup — it's a **hint**. The JVM may ignore it.
- "Memory leaks in Java are impossible" — they are, if you keep references alive (static collections, listeners, ThreadLocals, caches).
- Assuming all unreachable objects are immediately GC'd — only when the collector runs.
- Using finalizers for cleanup — deprecated since Java 9. Use `try-with-resources` or `Cleaner`.

### 🎤 Interview Angle
> "How does GC know an object is eligible for collection?"

It's unreachable from all GC roots via a chain of references. "Unreachable" is determined by a mark-and-sweep or tracing algorithm starting from roots.

> "What's the difference between Major and Full GC?"

Major GC collects only the Old Generation. Full GC collects the entire heap — Young + Old — plus Metaspace. Full GCs are the ones that cause 10-second "stop the world" pauses in poorly-tuned apps.

### 🧪 Practice
1. Write a method that causes a memory leak via a static collection; monitor heap with VisualVM.
2. Demonstrate `SoftReference` — when memory is tight, it gets cleared before OOM.
3. Use `WeakHashMap` for a metadata cache; verify entries vanish after GC.

### 🔍 Self-check
1. Why is Minor GC fast while Full GC is slow?
2. What are GC roots?
3. When does a `WeakReference` get cleared?
4. Why was `finalize()` deprecated?

---

## Friday — GC Algorithms: Serial, Parallel, CMS, G1, ZGC, Shenandoah

### 🎯 Objective
Choose the right GC algorithm for a given workload; interpret GC logs.

### 📖 Core Concept

**Serial GC** (`-XX:+UseSerialGC`)
- Single-threaded, stop-the-world.
- Simple, low overhead. Good for small-heap / single-CPU apps (embedded, CLI tools).

**Parallel GC** (`-XX:+UseParallelGC`) — default Java 8.
- Multi-threaded. Still stop-the-world.
- Optimizes for **throughput**: maximize app CPU time, tolerate longer pauses.
- Batch jobs, data processing.

**CMS — Concurrent Mark Sweep** (`-XX:+UseConcMarkSweepGC`)
- **Deprecated in Java 9, removed in Java 14.**
- Did most work concurrently with the app. Lower pauses than Parallel.
- Problem: no compaction → heap fragmentation.

**G1 — Garbage First** (`-XX:+UseG1GC`) — **default since Java 9**.
- Divides heap into ~2048 **regions** (typically 1–32 MB each).
- Each region dynamically Eden / Survivor / Old / Humongous.
- Tracks region garbage; collects regions with most garbage first.
- **Predictable pause target**: `-XX:MaxGCPauseMillis=200`.
- Does concurrent marking and mostly-concurrent compaction.

**ZGC** (`-XX:+UseZGC`) — production-ready since Java 15.
- **Sub-10ms pause times** regardless of heap size (tested to 16 TB).
- Uses **colored pointers** and **load barriers** for concurrent relocation.
- Best for low-latency, large-heap workloads.

**Shenandoah** — similar low-pause goals, Red Hat implementation. Uses Brooks pointers.

**Epsilon** (`-XX:+UseEpsilonGC`) — a "no-op" GC. Allocates memory, never reclaims. Used for benchmarking and short-lived JVMs where OOM is acceptable.

**Choosing:**
- **Throughput over latency**: Parallel GC.
- **Balanced**: G1 (default, safe bet).
- **Lowest latency, large heap**: ZGC or Shenandoah.
- **Small apps / containers**: Serial.

### 💻 Code Example — GC logging

Java 9+ unified logging:
```bash
-Xlog:gc*:file=gc.log:time,uptime:filecount=5,filesize=10M
```

Example G1 log line:
```
[2.500s][info][gc] GC(3) Pause Young (Normal) (G1 Evacuation Pause)
    40M->5M(256M) 8.123ms
```

Translation: at 2.5 s uptime, GC event #3, a young-generation evacuation pause. Heap went from 40MB used to 5MB used (capacity 256MB). Took 8ms.

**Common flags:**
```bash
-Xms4g -Xmx4g                    # initial and max heap
-Xmn1g                           # young gen size (Parallel GC)
-XX:MaxGCPauseMillis=200         # G1 pause target
-XX:+UseG1GC
-XX:+PrintGCDetails              # legacy (pre-9)
-Xlog:gc*:file=gc.log            # Java 9+ unified logging
-XX:+HeapDumpOnOutOfMemoryError  # dump heap on OOM
```

### ⚠️ Common Pitfalls
- Using CMS on new code (removed).
- Setting `MaxGCPauseMillis` too aggressively — forces more frequent minor collections, hurts throughput.
- Overlooking `-XX:MaxMetaspaceSize` — metaspace grows by default, uncapped, can surprise you.

### 🎤 Interview Angle
> "Why is G1 the default in modern Java?"

Predictable pauses + reasonable throughput + works well across small and large heaps. For most apps, it's "good enough" without tuning. Parallel GC is better for pure throughput, ZGC for pure latency.

### 🧪 Practice
1. Generate GC logs with `-Xlog:gc*`. Run a workload, analyze with GCViewer or GCeasy.
2. Compare G1 vs Parallel on the same workload — measure throughput and p99 latency.
3. Trigger `OutOfMemoryError`, use `-XX:+HeapDumpOnOutOfMemoryError`, analyze the dump in MAT.

### 🔍 Self-check
1. What's the default GC in Java 9+?
2. Why was CMS deprecated?
3. What GC would you pick for a 32GB heap with <10ms latency requirement?
4. What does `MaxGCPauseMillis` do in G1?

---

## Saturday — Memory Leaks, Heap Dumps, Diagnostic Tools

### 🎯 Objective
Find a memory leak using a heap dump; use JFR/VisualVM/jstack for live diagnosis.

### 📖 Core Concept

**Common memory leak causes:**
1. **Unbounded caches** — `static Map` that grows forever.
2. **Listeners not unregistered** — subscriber holds publisher → publisher holds subscribers.
3. **ThreadLocal values in thread pools** — pool thread outlives task, ThreadLocal entries accumulate.
4. **Inner class references** — non-static inner class holds the outer, preventing its GC.
5. **Unclosed resources** — streams, connections, threads.
6. **Classloader leaks** — a class loaded by classloader A holds a reference to an object loaded by classloader B; B can't be GC'd.

**Diagnostic workflow:**

**Live process inspection:**
- `jps` — list JVMs.
- `jstack <pid>` — thread dump (what are threads doing?).
- `jmap -heap <pid>` — heap summary.
- `jmap -histo:live <pid>` — class histogram (top classes by instance count / size).
- `jstat -gc <pid> 1s` — GC stats.
- `jcmd <pid> GC.run` — trigger GC (gentle request).

**Heap dump:**
- On demand: `jmap -dump:live,file=heap.hprof <pid>`.
- On OOM automatically: `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/path/to/dumps`.
- Analyze with **Eclipse MAT** — the gold standard. Use **dominator tree** to find retained objects, **leak suspects** for automated analysis.

**JFR (Java Flight Recorder)** — low-overhead profiling built into the JVM. Enable with:
```bash
-XX:StartFlightRecording=duration=60s,filename=recording.jfr
```
Analyze with **JDK Mission Control (JMC)**.

**VisualVM** — GUI for live monitoring (CPU, memory, threads, GC). Free, bundled pre-Java 9.

### 💻 Code Example — reproducing a leak

```java
public class LeakDemo {
    static final List<byte[]> leaks = new ArrayList<>();   // leak sink

    public static void main(String[] args) throws InterruptedException {
        while (true) {
            leaks.add(new byte[1024 * 1024]);              // +1 MB each iteration
            Thread.sleep(10);
        }
    }
}
// Run with:
//   java -Xmx256m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=. LeakDemo
// Analyze the dump in MAT — dominators point straight to LeakDemo.leaks.
```

**Thread dump via `jstack`:**
```
"main" #1 prio=5 os_prio=0 tid=0x... nid=0x... WAITING (parking)
    at sun.misc.Unsafe.park(Native Method)
    - parking to wait for <0x...> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
    at ...
```

Look for:
- Many threads stuck on the same lock → contention hotspot.
- Deadlock reports (`Found one Java-level deadlock`).
- Threads in `RUNNABLE` stuck in the same code location → infinite loop.

### ⚠️ Common Pitfalls
- Running `jmap -dump` without `live,` — captures garbage too, noisy dump.
- Ignoring thread dumps in production — they're free, always capture 3+ dumps a few seconds apart to detect motion.
- Not enabling `HeapDumpOnOutOfMemoryError` — when OOM happens, you have no data.

### 🎤 Interview Angle
> "A production app is growing heap over time. How do you diagnose?"

1. Confirm with GC logs (`-Xlog:gc*`) — is Old gen growing?
2. Take a heap dump with `jmap -dump:live` (or wait for `HeapDumpOnOutOfMemoryError`).
3. Open in MAT. Check **leak suspects** → **dominator tree** → top retained objects.
4. Identify the owning class and the path to GC root.
5. Fix: unbounded cache, uncleared ThreadLocal, classloader leak, etc.

### 🧪 Practice
1. Write the `LeakDemo` above, generate a heap dump, analyze it in MAT.
2. Generate a JFR recording during a busy operation; find the CPU hotspot in JMC.
3. Capture 3 thread dumps with `jstack` during a slow operation; identify the bottleneck.

### 🔍 Self-check
1. Name 3 tools to capture a heap dump.
2. What's the dominator tree in MAT?
3. What JVM flag produces a heap dump on OOM?
4. What's the cost of running JFR continuously in production?

---

## Sunday — Revision & Q&A

### 🧭 Revision checklist
- [ ] Draw the JVM architecture and memory areas from memory.
- [ ] Explain parent delegation + one attack it prevents.
- [ ] List Minor vs Major vs Full GC differences.
- [ ] Choose GC algorithm for 3 workload types.
- [ ] List 5 common memory leak causes.
- [ ] Walk through a heap-dump analysis workflow.

### 📝 Interview Q&A
Ask Claude: **"Prepare a Phase 1 Week 6 interview Q&A doc"** — answer, review, rate.

### 📂 Commit your work
```
phase-1/week-06/
├── exercises/
│   ├── ClassLoaderChain.java
│   ├── BytecodeDemo.java   (+ javap output as comment)
│   ├── LeakDemo.java
│   ├── WeakReferenceCache.java
│   └── gc-logs/  (captured traces)
└── notes.md
```
