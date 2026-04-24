# Java Interview Practice Notes (Day 1 & Day 2)

---

## 1. Stack vs Heap

**Q: What is the difference between Stack and Heap?**

**Answer:**
Stack is a thread-local memory used for method execution, storing stack frames, local variables, and references. Heap is a shared memory area used to store objects and instance data. Stack is fast and automatically managed, while heap is managed by the garbage collector.

---

## 2. Object Creation Flow

**Q: What happens when you create an object in Java?**

**Answer:**
When an object is created, the JVM allocates memory in the heap, initializes fields with default values, executes the constructor, and assigns the reference to a variable in the stack.

---

## 3. Garbage Collection Eligibility

**Q: When does an object become eligible for garbage collection?**

**Answer:**
An object becomes eligible for garbage collection when there are no references pointing to it from any reachable context (GC roots).

---

## 4. hashCode and equals

**Q: What is hashCode and why is it used?**

**Answer:**
hashCode returns an integer representation of an object and is used to determine the bucket location in hash-based collections like HashMap. It enables fast lookup by reducing search space.

**Q: What is the relationship between hashCode and equals?**

**Answer:**
If two objects are equal according to equals(), they must have the same hashCode. hashCode is used to locate the bucket, and equals is used to find the exact object.

---

## 5. HashMap Internals

**Q: How does HashMap work internally?**

**Answer:**
HashMap uses an array of buckets. It calculates the hash of the key to find the bucket index. If multiple keys map to the same bucket, it handles collisions using a linked list or a balanced tree (Java 8+). It resizes when the load factor threshold is exceeded.

**Q: What is the time complexity of HashMap operations?**

**Answer:**
Average case is O(1), but worst case can be O(n) or O(log n) depending on collisions.

---

## 6. LRU Cache

**Q: What is LRU Cache?**

**Answer:**
LRU (Least Recently Used) cache removes the least recently accessed item when the cache reaches its limit. It helps control memory usage and keeps frequently used data.

---

## 7. ArrayList vs LinkedList

**Q: Why is ArrayList faster for reading?**

**Answer:**
ArrayList uses a continuous array structure, allowing direct index-based access in O(1) time.

**Q: Why does LinkedList use more memory?**

**Answer:**
Each node in LinkedList stores data along with references to next and previous nodes, increasing memory overhead.

**Q: Why is LinkedList rarely used?**

**Answer:**
LinkedList has O(n) access time due to traversal, making it inefficient for read-heavy applications.

---

## 8. Time Complexity (Big-O)

**Q: What is O(1)?**

**Answer:**
Constant time complexity where execution time does not depend on input size.

**Q: What is O(n)?**

**Answer:**
Linear time complexity where execution time grows proportionally with input size.

---

## 9. ConcurrentHashMap

**Q: Why is HashMap not thread-safe?**

**Answer:**
Multiple threads can modify the internal structure simultaneously, leading to data inconsistency and corruption.

**Q: Why is ConcurrentHashMap better than synchronizedMap?**

**Answer:**
ConcurrentHashMap uses fine-grained locking or CAS, allowing multiple threads to operate concurrently, whereas synchronizedMap uses a single lock and reduces performance.

---

## 10. Fail-Fast vs Fail-Safe

**Q: What is Fail-Fast?**

**Answer:**
Fail-fast iterators throw ConcurrentModificationException when a collection is modified during iteration to prevent inconsistent behavior.

**Q: What is Fail-Safe?**

**Answer:**
Fail-safe iterators work on a copy or allow safe iteration without throwing exceptions, though they may not reflect real-time changes.

---

## 11. CopyOnWriteArrayList

**Q: When should CopyOnWriteArrayList be used?**

**Answer:**
It should be used when reads are frequent and modifications are rare, and safe iteration without exceptions is required.

---

## 12. Memory Leak in Java

**Q: Does Java have memory leaks?**

**Answer:**
Java does not have traditional memory leaks, but logical leaks can occur when objects are unnecessarily retained due to lingering references.

---

## 13. GC Concepts

**Q: What is Minor GC vs Major GC?**

**Answer:**
Minor GC cleans the young generation and is fast, while Major GC cleans the old generation and is slower with longer pause times.

---

## 14. Best Practice Answer Pattern

Always structure answers as:

1. Definition
2. How it works
3. Why it is used
4. Trade-offs

---

## 15. When would you use Comparable instead of Comparator?

Comparable is used when a class has a natural ordering that should be used consistently across the application.

---

## 16. Can we use both Comparable and Comparator together?

Yes, both Comparable and Comparator can be used together.

Collections.sort(users); // uses Comparable

Collections.sort(users, byName); // uses Comparator

---

## 17. What happens if compareTo is inconsistent with equals?

If compareTo is inconsistent with equals, it can lead to incorrect behavior in sorted collections like TreeSet, where duplicates may be allowed or elements may be lost.

---

30. When should you use List instead of Set?

Answer:

Use List when order matters, duplicates are allowed, and index-based access is required.

31. Why does Set not allow duplicates?

Answer:

Set does not allow duplicates because it uses hashCode to find the bucket and equals to check if an element already exists before inserting.

32. Why does HashSet depend on hashCode and equals?

Answer:

HashSet uses hashCode to determine the bucket location and equals to ensure uniqueness of elements.

33. What happens if hashCode is same but equals returns false?

Answer:

Both elements are stored in the same bucket as separate entries because equals determines uniqueness.

34. How is HashSet implemented internally?

Answer:

HashSet internally uses a HashMap where elements are stored as keys and a constant dummy value is used.

35. Which collection maintains uniqueness and insertion order?

Answer:

LinkedHashSet maintains uniqueness and preserves insertion order.

Q1. Why must equals() and hashCode() be overridden together?

They must be overridden together to maintain consistency in hash-based collections. If equals() returns true but hashCode() differs, objects go to different buckets, equals() is never called, leading to duplicates and retrieval failures.

Q2. What issue occurs in HashMap.get() if hashCode() is not overridden?

Logically equal objects may have different hashCodes and be stored in different buckets. During get(), HashMap searches a specific bucket, so it may fail to find the object even if it exists.

Q3. What happens internally in HashMap.get()?

1. Compute hashCode of key
2. Determine bucket index
3. Traverse bucket (linked list or tree)
4. Use equals() to find matching key
5. Return value if found, else null

Q4. Must unequal objects have different hashCode()?

No. Unequal objects can have the same hashCode (hash collision). In such cases, equals() is used to differentiate them within the same bucket.

Q5. How do you choose fields for equals() and hashCode()?

Use fields that represent logical identity, typically immutable and unique fields like primary key. Avoid mutable fields. In some cases, use business keys before persistence. Ensure consistency across system lifecycle.

Q1. What is the String Pool and why is it used?

String Pool is a special memory area in heap that stores unique string literals. It avoids duplicate objects and improves memory efficiency by reusing existing strings.

Q2. Difference between String a = "hello" and new String("hello")?

String a = "hello" uses the String Pool and reuses the object if it exists.
String b = new String("hello") creates a new object in heap even if the value exists in the pool, resulting in separate objects.

Q3. Why is String immutable in Java?

1. Security – prevents modification of sensitive data
2. Thread safety – immutable objects are inherently thread-safe
3. String Pooling – reuse works only if values don’t change
4. HashCode caching – improves performance in hash-based collections

Q4. What happens when s = s.concat(" world")?

A new String object "hello world" is created because String is immutable. The original string remains unchanged, and the reference is updated.

Q5. Why use StringBuilder in loops?

String is immutable and creates new objects on each modification, causing performance overhead. StringBuilder is mutable and modifies the same object, making it more efficient.

Q1. Why do we need wrapper classes?

Wrapper classes convert primitives into objects so they can be used in collections, generics, and APIs that require objects. They also provide utility methods.

Q2. What is autoboxing and unboxing?

Autoboxing is automatic conversion from primitive to wrapper (e.g., Integer a = 10 → Integer.valueOf(10)).
Unboxing is conversion from wrapper to primitive (e.g., int b = a → a.intValue()).

Q3. Why does Integer behave differently for 100 and 200?

Integer values between -128 to 127 are cached. Values within this range reuse the same object, while values outside create new objects.

Q4. What is the problem with using == for wrapper classes?

== compares references, not values. It may return true for cached values and false otherwise. equals() should be used for value comparison.

Q5. How can unboxing cause NullPointerException?

If a wrapper object is null, unboxing calls methods like intValue(), causing NullPointerException.
Example:
Integer a = null;
int b = a; // NPE

Q1. What problem did Java 8 solve in HashMap?
Before Java 8, collisions were handled using linked lists, leading to O(n) worst-case time. Java 8 introduced treeification to convert buckets into trees, improving performance to O(log n).

Q2. What is treeification in HashMap?
Treeification is the process of converting a bucket from a linked list to a balanced tree when collisions exceed a threshold.

Q3. When does HashMap convert a LinkedList into a tree?
When the number of elements in a bucket exceeds 8 and the overall capacity is at least 64.

Q4. Why not always use a tree instead of a linked list?
Trees have higher overhead. For small data, linked lists are more efficient, so trees are used only when collisions are high.

Q5. What happens if capacity < 64 but collisions exceed threshold?
HashMap resizes (increases capacity) and rehashes entries instead of treeifying.

Q1. Difference between HashMap and LinkedHashMap?
HashMap does not maintain order, while LinkedHashMap maintains insertion or access order using a doubly linked list.

Q2. How does LinkedHashMap maintain order?
It uses a doubly linked list of entries, maintaining references to previous and next nodes for predictable iteration.

Q3. Types of ordering in LinkedHashMap?
1. Insertion order
2. Access order

Q4. When to use LinkedHashMap?
When order matters, such as LRU cache, logging, or predictable iteration.

Q5. How to implement LRU cache using LinkedHashMap?
Use access-order LinkedHashMap and override removeEldestEntry() to remove least recently used entries when capacity is exceeded.

Q1. Why is encapsulation important?
Encapsulation protects internal state and enforces business rules through controlled access, ensuring objects remain in a valid state.

Q2. Real-world example of abstraction?
A messaging system where NotificationService interface has implementations like Email, SMS, etc., allowing flexibility without changing client code.

Q3. Abstraction vs Encapsulation?
Abstraction hides complexity and exposes behavior, while encapsulation hides data and protects internal state.

Q4. Why use interfaces in service layer?
Interfaces provide loose coupling, flexibility, and support multiple implementations, enabling better design and dependency injection.

Q5. What is the risk of over-abstraction?
It can make systems complex, harder to read, debug, and maintain due to unnecessary layers and indirection.

Q1. What is the main problem with inheritance?
Inheritance leads to tight coupling and rigid design. Changes in the parent class can impact all subclasses and make the system harder to modify.

Q2. Why is composition preferred over inheritance?
Composition provides loose coupling and flexibility, allowing behavior to change dynamically without affecting class hierarchy.

Q3. Example where inheritance fails but composition works?
Bird example: defining fly() in base class breaks subclasses like Ostrich. Composition allows only appropriate classes to implement flying behavior.

Q4. Difference between IS-A and HAS-A?
IS-A represents inheritance (Dog is an Animal), while HAS-A represents composition (Car has an Engine).

Q5. Can inheritance and composition be combined?
Yes. Use inheritance for common behavior and composition for flexible or optional behavior, such as using interfaces for capabilities.

Q2. Can private methods be overridden?
No, because private methods are not inherited and hence cannot be overridden.

Q3. Difference between List and ArrayList reference?
Reference type determines accessible methods. List allows only List methods (abstraction), while ArrayList allows implementation-specific methods.

Q4. Why composition over inheritance?
Composition provides loose coupling and flexibility, while inheritance leads to rigid and tightly coupled hierarchies.

Q5. Static method behavior?
Static methods are not overridden but hidden. They are resolved at compile time based on reference type.
Output: A

Q6. Abstract class vs Interface?
Abstract class can have state and behavior; interface defines contract. Use abstract class for shared logic, interface for flexibility and multiple implementations.

Q1. Why is Runnable preferred over Thread?
Runnable separates task from thread, provides flexibility, and allows extending other classes.

Q2. Difference between start() and run()?
start() creates a new thread and executes run() concurrently, while run() executes in the current thread like a normal method call.

Q3. Why is thread execution order unpredictable?
Because thread scheduling is handled by JVM and OS, and threads execute independently and concurrently.

Q1. Why is count++ not thread-safe?
count++ is not thread-safe because it is a non-atomic operation involving read, modify, and write steps. Multiple threads can concurrently modify the same variable, leading to inconsistent results due to lost updates.

Q2. What is the role of join()?
join() ensures that one thread waits for another to complete execution, guaranteeing that results are consistent and complete before proceeding.

Q1. Why does volatile not solve race conditions?
volatile ensures visibility but not atomicity. Multiple threads can concurrently modify a variable, leading to inconsistent results.

Q2. When to use volatile?
volatile is used for status flags or signals, where visibility is required but no compound operations are involved.

Q1. Why use unlock() in finally?
To ensure the lock is always released, even if an exception occurs, preventing threads from being blocked indefinitely.

Q2. How does tryLock() help prevent deadlock?
tryLock() attempts to acquire a lock without waiting indefinitely, allowing threads to avoid circular wait conditions.

**End of Document**
