

# 📘 Java Interview Definitions – Quick Reference

---

## 🔹 Memory & Object Creation

### Stack

Memory area that stores method calls, local variables, and references. It is thread-specific and follows a LIFO structure.

### Heap

Shared memory area used for storing objects and instance variables. Managed by the Garbage Collector.

### Java Object Creation

Process involving class loading, memory allocation in heap, initialization of variables, constructor execution, and assignment of reference.

---

## 🔹 Garbage Collection

### Minor GC

Garbage collection process that cleans the Young Generation space, occurring frequently with low pause time.

### Major GC (Full GC)

Garbage collection process that cleans the Old Generation space, less frequent but more expensive in terms of performance.

---

## 🔹 Hashing & Equality

### hashCode()

Method used to generate an integer hash value for an object, primarily for bucket placement in hash-based collections.

### equals()

Method used to compare logical equality between two objects.

### Hash Collision

Condition where multiple keys produce the same hash value and map to the same bucket.

---

## 🔹 Collections Core

### List

Ordered collection that allows duplicate elements and maintains insertion order.

### Set

Collection that does not allow duplicate elements.

---

## 🔹 List Implementations

### ArrayList

Resizable array implementation of List that provides fast random access but slower insertions and deletions.

### LinkedList

Doubly linked list implementation that provides efficient insertions and deletions but slower access.

---

## 🔹 Set Implementations

### HashSet

Set implementation backed by a hash table with no guarantee of order.

### LinkedHashSet

HashSet variant that maintains insertion order.

---

## 🔹 Map Implementations

### HashMap

Key-value data structure that allows one null key, is not thread-safe, and provides constant-time operations on average.

### Treeification in HashMap

Process of converting a bucket’s linked list into a balanced tree structure when a threshold is exceeded to improve performance.

### LinkedHashMap

HashMap variant that maintains insertion or access order.

---

## 🔹 LRU Cache

### LRU Cache

Caching mechanism that removes the least recently used entry when capacity is exceeded.

---

## 🔹 Time Complexity

Measure of algorithm performance in terms of input size, expressed using Big-O notation.

---

## 🔹 Concurrency Collections

### ConcurrentHashMap

Thread-safe map designed for high concurrency using internal partitioning or non-blocking techniques.

### SynchronizedMap

Thread-safe wrapper over a map where all operations are synchronized on a single lock.

---

## 🔹 Iteration Behavior

### Fail-Fast

Iterator behavior that throws an exception if the collection is structurally modified during iteration.

### Fail-Safe

Iterator behavior that operates on a copy of the collection and does not throw exceptions on modification.

---

### CopyOnWriteArrayList

Thread-safe list where modifications result in a new copy, ensuring safe iteration.

---

## 🔹 Memory Management

### Memory Leak in Java

Situation where objects are no longer needed but remain referenced, preventing garbage collection.

---

## 🔹 Sorting & Comparison

### Comparable

Interface that defines natural ordering of objects through a single comparison method.

### Comparator

Interface used to define custom ordering separate from the object's natural ordering.

### compareTo()

Method used for natural ordering comparison between objects.

---

## 🔹 String Handling

### String

Immutable sequence of characters stored in a special memory pool.

### String Pool

Memory area that stores unique string literals to optimize memory usage.

### StringBuilder

Mutable sequence of characters used for efficient string manipulation.

---

## 🔹 Wrapper Classes

### Wrapper Class

Object representation of primitive data types.

### Autoboxing

Automatic conversion of primitive types to their corresponding wrapper objects.

### Unboxing

Automatic conversion of wrapper objects to primitive types.

---

## 🔹 OOP Concepts

### Abstraction

Concept of hiding implementation details and exposing only essential functionality.

### Encapsulation

Binding data and methods together while restricting direct access to internal state.

### Inheritance

Mechanism where one class acquires properties and behavior of another.

### Composition

Design principle where a class contains objects of other classes to achieve functionality.

---

### IS-A Relationship

Represents inheritance where one class is a subtype of another.

### HAS-A Relationship

Represents composition where a class contains another class as a member.

---

### List vs ArrayList Reference

Practice of using interface type references for flexibility and loose coupling.

---

## 🔹 Static & Interfaces

### Static Method

Method that belongs to the class rather than an instance and can be accessed without object creation.

### Abstract Class

Class that may contain abstract and concrete methods and cannot be instantiated.

### Interface

Contract that defines abstract methods (and optionally default/static methods) to be implemented by classes.

---

## 🔹 Multithreading Basics

### Thread

Smallest unit of execution within a process.

### Runnable

Functional interface representing a task to be executed by a thread.

---

### start()

Method that initiates a new thread and invokes run() internally.

### run()

Method containing the code executed by a thread.

---

### Thread Execution Order

Order in which threads execute, determined by the scheduler and not guaranteed.

---

### Thread-safe

Property of code that behaves correctly when accessed by multiple threads concurrently.

---

### join()

Method that causes the current thread to wait until another thread completes execution.

---

### volatile

Keyword ensuring visibility of variable changes across threads but not guaranteeing atomicity.

---

## 🔹 Concurrency Issues

### Race Condition

Condition where multiple threads access and modify shared data leading to unpredictable results.

### Critical Section

Section of code that accesses shared resources and must be executed by only one thread at a time.

---

### Synchronized Keyword

Mechanism for acquiring intrinsic lock to ensure mutual exclusion.

---

### tryLock()

Method that attempts to acquire a lock without blocking indefinitely.

---

### Visibility

Guarantee that changes made by one thread are visible to others.

### Atomicity

Guarantee that operations are completed entirely without interruption.

---

### ReentrantLock

Advanced locking mechanism providing explicit control over locking with additional features.

---

### ReentrantLock vs Synchronized

Comparison between explicit lock mechanism and intrinsic locking with differences in flexibility and features.

---

### Deadlock

Situation where two or more threads are blocked indefinitely waiting for each other to release resources.

---