# Week 3 — Collections Deep Dive

**Theme:** Every senior Java interview probes collection internals — especially `HashMap` and `ConcurrentHashMap`. You should be able to explain the data structures, the complexity trade-offs, and why `equals`/`hashCode` contracts matter.

---

## Monday — Collection Hierarchy, Iterator, Iterable

### 🎯 Objective
Draw the `java.util.Collection` hierarchy; implement a custom iterator that supports `remove()`.

### 📖 Core Concept

```
Iterable<E>
└── Collection<E>
    ├── List<E>      — ordered, indexed, duplicates allowed
    ├── Set<E>       — no duplicates (by equals)
    │   └── SortedSet<E>
    │       └── NavigableSet<E>
    ├── Queue<E>     — FIFO/priority
    │   └── Deque<E> — double-ended
    └── (no Map here! Map is NOT a Collection)

Map<K, V>            — separate hierarchy
└── SortedMap<K, V>
    └── NavigableMap<K, V>
```

**`Iterable<E>`** — any class with a `iterator()` method, usable in enhanced-for.

**`Iterator<E>`** — forward-only cursor with `hasNext()`, `next()`, and (optional) `remove()`.

**`ListIterator<E>`** — bidirectional, supports `add`, `set`, `hasPrevious`.

**Enhanced for loop sugar:**
```java
for (String s : list) { ... }
// is equivalent to:
for (Iterator<String> it = list.iterator(); it.hasNext(); ) {
    String s = it.next();
    ...
}
```

### 💻 Code Example

```java
public class Range implements Iterable<Integer> {
    private final int from, to;
    public Range(int from, int to) { this.from = from; this.to = to; }

    @Override
    public Iterator<Integer> iterator() {
        return new Iterator<>() {
            int cursor = from;
            public boolean hasNext() { return cursor < to; }
            public Integer next() {
                if (!hasNext()) throw new NoSuchElementException();
                return cursor++;
            }
        };
    }
}

for (int i : new Range(0, 5)) System.out.println(i);   // 0 1 2 3 4
```

### ⚠️ Common Pitfalls
- Forgetting that `Map` is not a `Collection` — it's iterated via `entrySet()`, `keySet()`, or `values()`.
- Calling `iterator.remove()` without first calling `next()` — throws `IllegalStateException`.
- Modifying the underlying collection during iteration without using `iterator.remove()` — throws `ConcurrentModificationException`.

### 🎤 Interview Angle
> "How do you remove items from a list while iterating?"

Correct: use `iterator.remove()` or `list.removeIf(predicate)`. Incorrect: `list.remove(obj)` inside a for-each loop — `ConcurrentModificationException`.

### 🧪 Practice
1. Implement the `Range` class above and iterate it.
2. Extend it with a bidirectional `ListIterator`-like interface.
3. Write a method that removes every even number from a `List<Integer>` using an explicit iterator.

### 🔍 Self-check
1. Why is `Map` not a `Collection`?
2. What's the difference between `Iterator` and `ListIterator`?
3. How does the enhanced-for loop desugar?
4. When would you implement `Iterable<E>` on your own class?

---

## Tuesday — ArrayList vs LinkedList

### 🎯 Objective
Pick the right list based on the operations you need; explain the memory and complexity trade-offs.

### 📖 Core Concept

| Operation | ArrayList | LinkedList |
|-----------|-----------|-----------|
| `get(index)` | O(1) | O(n) |
| `add(E)` (append) | O(1) amortized | O(1) |
| `add(index, E)` (insert) | O(n) | O(n) walk + O(1) link |
| `remove(index)` | O(n) | O(n) walk + O(1) unlink |
| `contains(E)` | O(n) | O(n) |
| Memory per element | minimal (just ref) | ref + 2 pointers |
| Cache locality | excellent | poor |

**`ArrayList`** — resizable array.
- Default capacity 10. When full, grows by 50% (`newCap = oldCap + (oldCap >> 1)`).
- Backing `Object[]` — primitive lists (`int[]` etc.) aren't supported; use IntList libraries or `IntStream`.

**`LinkedList`** — doubly-linked list. Implements `List`, `Deque`, and `Queue`.
- Each node holds value + `prev` + `next` pointers.
- O(1) insert/remove at either end.
- Poor cache locality — nodes scattered in memory, each dereference is a cache miss.

**Rule of thumb:** use `ArrayList` unless you have a specific reason for `LinkedList` (frequent add/remove at BOTH ends, or queue/deque semantics — and even then, `ArrayDeque` is usually faster).

### 💻 Code Example

```java
// Preferred random-access pattern
List<String> names = new ArrayList<>();
names.add("A"); names.add("B");
String first = names.get(0);               // O(1)

// Deque semantics — LinkedList or ArrayDeque
Deque<Integer> queue = new ArrayDeque<>();
queue.offerLast(1);    // add to tail
queue.offerFirst(0);   // add to head
queue.pollFirst();     // remove from head

// Iteration benchmark — ArrayList wins due to cache locality
// Insertion at head — LinkedList wins
```

### ⚠️ Common Pitfalls
- Choosing `LinkedList` "because I'll be inserting a lot" — you still need an O(n) walk to find the insertion point unless you have the node reference.
- `ArrayList.remove(int)` vs `ArrayList.remove(Object)` — for `List<Integer>`, `list.remove(1)` removes index 1; `list.remove(Integer.valueOf(1))` removes value 1.
- Not pre-sizing `new ArrayList<>(expectedSize)` — repeated re-allocation during bulk adds.

### 🎤 Interview Angle
> "Why is `ArrayList` almost always faster than `LinkedList` in practice even though `LinkedList` has better Big-O for inserts?"

CPU cache locality. `ArrayList` stores references contiguously in an array — when the CPU loads one element, the next N are prefetched. `LinkedList` nodes are scattered in heap memory, causing cache misses on every hop. For typical list sizes (<10k), this dominates over Big-O differences.

### 🧪 Practice
1. Benchmark `ArrayList.add(0, x)` vs `LinkedList.addFirst(x)` for 100k inserts.
2. Benchmark `get(random index)` for both — 1M reads.
3. Implement a simple LRU queue using `ArrayDeque`.

### 🔍 Self-check
1. What's the amortized complexity of `ArrayList.add(E)` and why isn't it O(n) due to resizing?
2. When does `LinkedList` actually outperform `ArrayList`?
3. Why might `new ArrayList<>(1000)` be faster than `new ArrayList<>()`?
4. What's the difference between `ArrayDeque` and `LinkedList` as a deque?

---

## Wednesday — Sets & the equals/hashCode Contract

### 🎯 Objective
Write a correct `equals`/`hashCode` pair; explain how bad implementations break hash-based collections.

### 📖 Core Concept

**Set implementations:**

| Set | Underlying | Order | Null |
|-----|-----------|-------|------|
| `HashSet` | `HashMap` | no order | 1 null |
| `LinkedHashSet` | `LinkedHashMap` | insertion order | 1 null |
| `TreeSet` | `TreeMap` (red-black) | sorted | no null |
| `EnumSet` | bit vector | natural enum order | no null |
| `CopyOnWriteArraySet` | `CopyOnWriteArrayList` | insertion order | 1 null |

**The `equals` contract:**
- **Reflexive:** `x.equals(x)` → true.
- **Symmetric:** `x.equals(y) == y.equals(x)`.
- **Transitive:** if `x.equals(y)` and `y.equals(z)`, then `x.equals(z)`.
- **Consistent:** repeated calls return the same result (unless mutated).
- **Non-null:** `x.equals(null)` → false.

**The `hashCode` contract:**
- If `x.equals(y)`, then `x.hashCode() == y.hashCode()` (MANDATORY).
- If NOT equal, hash codes don't HAVE to differ but should (for distribution).
- Consistent within a program execution.

**Violations break hash-based collections** — `HashMap.get(key)` first locates the bucket via `hashCode()`, then confirms with `equals()`. Bad hashCode → key disappears.

### 💻 Code Example

```java
public class Employee {
    private final long id;
    private final String name;
    private final String email;

    public Employee(long id, String name, String email) {
        this.id = id; this.name = name; this.email = email;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Employee e)) return false;     // pattern matching
        return id == e.id && Objects.equals(email, e.email);
        // equality based on id + email (business identity)
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, email);
    }
}
```

**Demonstrating breakage:**
```java
class BadKey {
    int id;
    BadKey(int id) { this.id = id; }
    public boolean equals(Object o) { return o instanceof BadKey b && b.id == id; }
    // forgot hashCode — uses Object's default (identity)
}

Map<BadKey, String> m = new HashMap<>();
m.put(new BadKey(1), "Alice");
m.get(new BadKey(1));     // null — different hash buckets, never finds the entry
```

### ⚠️ Common Pitfalls
- Override `equals` without `hashCode` (or vice versa) — the #1 collection bug.
- Basing `equals`/`hashCode` on **mutable** fields that change after the object is inserted into a hash-based collection — key becomes lost.
- Using `==` instead of `.equals()` for object comparison inside your own `equals` (defeating the purpose).

### 🎤 Interview Angle
> "What happens if two keys have the same `hashCode` but `equals` returns false?"

They collide — both live in the same bucket as a linked list (or red-black tree if >8 entries & capacity ≥ 64 since Java 8). `get()` still works correctly; performance degrades.

> "What happens if they're equal but have different hashCodes?"

Catastrophic: `HashMap.put` and `get` look in different buckets. The second put creates a duplicate entry; `get` returns null for one of them.

### 🧪 Practice
1. Write an `Employee` class with correct `equals`/`hashCode` and demonstrate use in a `HashSet`.
2. Write a deliberately broken `hashCode` (always returns 0) and measure the `HashMap` lookup degradation.
3. Show a "lost key" by mutating a field used in hashCode after inserting into a `HashSet`.

### 🔍 Self-check
1. If `x.equals(y)` but `x.hashCode() != y.hashCode()`, which collections are broken?
2. Why does `TreeSet` not allow null?
3. What's the difference between `HashSet` and `LinkedHashSet`?
4. Why is using mutable objects as HashMap keys dangerous?

---

## Thursday — HashMap Internals & Treeification

### 🎯 Objective
Explain step-by-step what happens on `map.put(key, value)` including resize and treeification.

### 📖 Core Concept

**Internal structure:**
- `HashMap` has a backing array `Node<K,V>[] table`.
- Each bucket is either `null`, a single `Node`, a linked list of nodes, or a red-black `TreeNode`.

**On `put(key, value)`:**
1. Compute `hash(key)` — uses `key.hashCode()` XOR'd with `hash >>> 16` (distributes high bits into the index).
2. Compute index: `(table.length - 1) & hash` (equivalent to modulo because length is power of 2).
3. If bucket is empty → create a Node.
4. If bucket has an entry, walk the chain:
   - If a node with `equals`-matching key exists → update value, return old.
   - Else → append at end (Java 8+; pre-Java 8 was head-insertion).
5. Count the number of entries. If chain length > **TREEIFY_THRESHOLD (8)** AND `table.length` ≥ **MIN_TREEIFY_CAPACITY (64)** → convert the bucket's linked list to a red-black tree.
6. If total size > `capacity × loadFactor (0.75)` → **resize** (double capacity, rehash all entries).

**On `get(key)`:**
1. Compute hash and index.
2. Walk the bucket (list or tree). Return the node whose key `.equals` the search key.

**Resize cost:** O(n) — allocates a new array and redistributes all entries. Rare due to 0.75 factor, but visible in hot-insert paths.

**Java 8 treeification** protects against **hash-collision DoS attacks** (adversary sends many keys with the same hash → O(n) bucket → use as DoS). Trees cap it at O(log n).

**Java 8 chain order:** when entries rehash, their relative order is preserved (not reversed as in Java 7). Prevents a subtle concurrency bug that made pre-Java-8 `HashMap` resize a known infinite-loop hazard.

### 💻 Code Example

```java
// Demonstrate resize trigger
Map<Integer, Integer> map = new HashMap<>();   // default cap 16, loadFactor 0.75
// Resize at size > 16 * 0.75 = 12
for (int i = 0; i < 13; i++) map.put(i, i);
// capacity is now 32

// Pre-size for known bulk loads to avoid resizes
Map<String, String> bulk = new HashMap<>((int)(expectedSize / 0.75f) + 1);
```

**Chain length exploration:**
```java
// Bad hashCode forces all entries into one bucket — triggers treeification
class BadKey {
    int id;
    BadKey(int id) { this.id = id; }
    public int hashCode() { return 0; }    // all collide
    public boolean equals(Object o) { return o instanceof BadKey b && b.id == id; }
}
```

### ⚠️ Common Pitfalls
- Assuming iteration order is insertion order — `HashMap` gives NO order guarantee. Use `LinkedHashMap` if you need one.
- Sharing a `HashMap` across threads without synchronization — can produce `ConcurrentModificationException`, data loss, or in Java 7, infinite loops.
- Mutating a key's hash-relevant state after insertion — key becomes unreachable.

### 🎤 Interview Angle
> "Why is `HashMap` capacity always a power of 2?"

So that `index = (capacity - 1) & hash` is cheap — bitwise AND instead of modulo. If capacity were 15, you'd need `hash % 15` — much slower.

> "What's the treeify threshold and why?"

8 for treeify, 6 for untreeify, minimum table capacity 64 before treeifying. The hysteresis avoids thrashing. The threshold is derived from a Poisson distribution analysis: with a good hash, the probability of a bucket having ≥ 8 entries is ~10⁻⁸.

### 🧪 Practice
1. Implement a mini `HashMap` with just an array + linked list (no treeification).
2. Benchmark your mini vs `java.util.HashMap` for 100k puts & gets.
3. Trigger treeification with a deliberately bad hashCode — observe behavior.

### 🔍 Self-check
1. What's the default capacity and load factor?
2. When exactly does treeification happen?
3. Why did Java 7's `HashMap` resize have a concurrency bug?
4. Why is pre-sizing a `HashMap` valuable for bulk inserts?

---

## Friday — LinkedHashMap, TreeMap, WeakHashMap, LRU Cache

### 🎯 Objective
Build an LRU cache in 20 lines; explain when TreeMap's O(log n) beats HashMap's O(1).

### 📖 Core Concept

**`LinkedHashMap`** — `HashMap` + a doubly-linked list through entries.
- **Insertion-order** mode (default): iterates in the order entries were added.
- **Access-order** mode (constructor flag): on each `get`, moves the entry to the tail. Combined with `removeEldestEntry` override — this is the canonical LRU cache.

**`TreeMap`** — red-black tree. Sorted by keys (natural order or `Comparator`).
- All ops O(log n).
- NavigableMap: `firstKey`, `lastKey`, `floorKey`, `ceilingKey`, `headMap`, `tailMap`.
- No null keys allowed (comparison on null).

**`WeakHashMap`** — keys held via `WeakReference`. When no strong references to a key remain, GC reclaims the key AND the map entry. Useful for metadata caches keyed on objects.

**`IdentityHashMap`** — uses `==` for key comparison (not `equals`). Used by serialization frameworks tracking "objects I've already seen by reference."

**`EnumMap`** — keys are enum constants. Backed by an array indexed by ordinal. Extremely fast.

### 💻 Code Example — LRU Cache

```java
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public LRUCache(int capacity) {
        super(capacity, 0.75f, true);   // access-order = true
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
}

// Usage:
LRUCache<String, String> cache = new LRUCache<>(3);
cache.put("a", "1");
cache.put("b", "2");
cache.put("c", "3");
cache.get("a");              // access 'a' → moves to tail
cache.put("d", "4");          // evicts 'b' (least recently used)
System.out.println(cache);    // {c=3, a=1, d=4}
```

**TreeMap range queries:**
```java
TreeMap<LocalDate, Order> orders = new TreeMap<>();
// All orders in a date range:
SortedMap<LocalDate, Order> jan = orders.subMap(
    LocalDate.of(2024, 1, 1),
    LocalDate.of(2024, 2, 1));
```

### ⚠️ Common Pitfalls
- Using `LinkedHashMap` in access-order mode but forgetting to override `removeEldestEntry` — no eviction, just a memory leak with nicer iteration order.
- Assuming `TreeMap` is "always slower than HashMap" — if you need sorted access or range queries, HashMap can't help.
- `WeakHashMap` values held with strong references defeat its purpose if they reference the keys.

### 🎤 Interview Angle
> "Implement an LRU cache."

Many candidates go straight to a HashMap + doubly linked list. That's the "from scratch" version. The elegant answer: `LinkedHashMap` with access-order. Show both, because some interviewers want the LL+HashMap implementation as a DSA exercise.

### 🧪 Practice
1. Implement LRU cache using `LinkedHashMap` (above) AND from scratch with HashMap + doubly linked list.
2. Use `TreeMap` to find all keys in a range.
3. Use `WeakHashMap` to build a metadata cache that doesn't prevent GC.

### 🔍 Self-check
1. What's the difference between insertion-order and access-order in `LinkedHashMap`?
2. When is `TreeMap` preferable to `HashMap`?
3. What happens to a `WeakHashMap` entry when the key is GC'd?
4. What operation does `LinkedHashMap` override to implement LRU eviction?

---

## Saturday — Queue, Deque, PriorityQueue, Comparable vs Comparator

### 🎯 Objective
Use the right queue/deque implementation per scenario; sort with natural order AND custom comparators.

### 📖 Core Concept

**Queue hierarchy:**
```
Queue<E>
├── PriorityQueue<E>    — priority heap (min-heap by default)
├── ArrayDeque<E>       — resizable array deque (ALSO a Deque)
├── LinkedList<E>       — also List, also Deque
└── BlockingQueue<E>    — thread-safe (next week)
    ├── ArrayBlockingQueue
    ├── LinkedBlockingQueue
    └── SynchronousQueue
```

**Queue methods (FIFO):**
| Operation | Throws | Returns special |
|-----------|--------|-----------------|
| insert | `add(e)` | `offer(e)` |
| remove | `remove()` | `poll()` |
| examine | `element()` | `peek()` |

**Deque methods — same but with First/Last:**
`offerFirst`, `offerLast`, `pollFirst`, `pollLast`, `peekFirst`, `peekLast`.

**`ArrayDeque`** — the recommended general-purpose queue/stack/deque. Faster than `LinkedList`, faster than `Stack` (legacy, synchronized).

**`PriorityQueue`** — binary min-heap. Elements come out in priority order, NOT insertion order. Iteration order is NOT sorted (only the head is guaranteed smallest).

**`Comparable<T>`** — natural ordering, defined by the class itself.
```java
public int compareTo(T other) {
    // negative → this < other
    // zero     → equal ordering
    // positive → this > other
}
```

**`Comparator<T>`** — external ordering, defined outside the class. Lambdas and method references make this one-liners.

### 💻 Code Example

```java
// Comparable
public class Person implements Comparable<Person> {
    private final String name;
    private final int age;
    public int compareTo(Person o) {
        return Integer.compare(this.age, o.age);   // null-safe, overflow-safe
    }
}

// Comparator — rich composition
Comparator<Person> byName = Comparator.comparing(Person::getName);
Comparator<Person> byAgeDesc = Comparator.comparingInt(Person::getAge).reversed();
Comparator<Person> byAgeThenName = Comparator
    .comparingInt(Person::getAge)
    .thenComparing(Person::getName);

// Handling nulls
Comparator<Person> nullsLast = Comparator.nullsLast(byAgeThenName);

// PriorityQueue with custom order — max-heap of integers
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
```

**Common DSA pattern — top-K:**
```java
public static List<Integer> topK(int[] nums, int k) {
    PriorityQueue<Integer> minHeap = new PriorityQueue<>();
    for (int n : nums) {
        minHeap.offer(n);
        if (minHeap.size() > k) minHeap.poll();
    }
    return new ArrayList<>(minHeap);
}
```

### ⚠️ Common Pitfalls
- Iterating a `PriorityQueue` and expecting sorted order — you won't get it. Only `poll()` produces sorted extraction.
- Writing `compareTo` as `return this.x - other.x` — integer overflow if values have opposite signs. Use `Integer.compare`.
- `compareTo` inconsistent with `equals` — not illegal but violates contract (`TreeSet` uses `compareTo`, `HashSet` uses `equals`; you'll get surprising behavior).
- Using `Stack` — legacy, synchronized, slow. Use `ArrayDeque` as a stack.

### 🎤 Interview Angle
> "Why prefer `Comparator.comparing` over a custom comparator class?"

Composability and clarity: `Comparator.comparing(X::age).thenComparing(X::name).reversed()` reads like the intent. Also handles nulls and primitives cleanly.

> "Is `compareTo` consistent with `equals` required?"

Strongly recommended. If inconsistent, `TreeSet`/`TreeMap` treat elements as equal when `compareTo == 0`, while `HashSet`/`HashMap` use `equals`. BigDecimal is a famous example: `new BigDecimal("1.0").compareTo(new BigDecimal("1.00")) == 0` but `.equals()` is false — prepared buckets of bugs.

### 🧪 Practice
1. Sort a `List<Person>` by age desc, then name asc using `Comparator` composition.
2. Implement top-K with `PriorityQueue` (above).
3. Use `ArrayDeque` as both a stack and a queue; benchmark against `Stack` and `LinkedList`.

### 🔍 Self-check
1. Is `PriorityQueue` sorted when you iterate?
2. What's the difference between `add` and `offer` for queues?
3. Why is `Stack` discouraged?
4. How do you reverse a comparator?

---

## Sunday — Revision & Q&A

### 🧭 Revision checklist
- [ ] Draw the Collection hierarchy from memory.
- [ ] Explain `HashMap.put` step by step including resize + treeify.
- [ ] Implement LRU cache two ways.
- [ ] List 5 mistakes that break `equals`/`hashCode`.
- [ ] Use `Comparator` composition to sort by 3 fields.

### 📝 Interview Q&A
Ask Claude: **"Prepare a Phase 1 Week 3 interview Q&A doc"** — answer in your own words, get reviewed.

### 📂 Commit your work
```
phase-1/week-03/
├── exercises/
│   ├── Range.java
│   ├── ListBenchmark.java
│   ├── Employee.java        (with proper equals/hashCode)
│   ├── MiniHashMap.java
│   ├── LRUCache.java        (both implementations)
│   └── TopK.java
└── notes.md
```
