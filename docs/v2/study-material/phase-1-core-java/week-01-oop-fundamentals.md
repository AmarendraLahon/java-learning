# Week 1 — OOP Fundamentals & Language Basics

**Theme:** Build a concrete mental model of how Java classes, objects, and inheritance actually behave. Most interview OOP questions aren't about definitions — they're about edge cases (constructor chaining order, overload resolution, final variable semantics).

---

## Monday — Classes, Objects, Constructors, `this` / `super`

### 🎯 Objective
Explain what happens in memory from `new Person("Alice")` until the reference is returned, including constructor chaining.

### 📖 Core Concept

A **class** is a blueprint — it describes fields (state) and methods (behavior). An **object** is an instance of a class, living in the heap. A **reference** (what you usually call "variable") sits on the stack and points to the heap object.

**Constructors** initialize an object after memory is allocated. Every class has at least one — if you don't declare any, the compiler adds a public no-arg constructor. The moment you declare **any** constructor, that implicit one disappears.

**Constructor chaining** works two ways:
- `this(...)` — call another constructor in **the same class** (must be first statement).
- `super(...)` — call a superclass constructor (must be first statement; inserted implicitly as `super()` if omitted).

Execution order when `new Child()` runs:
1. Memory allocated, fields set to default values (0, null, false).
2. Control jumps to `Child`'s constructor.
3. `super(...)` runs first → goes all the way up to `Object`.
4. On the way back down, each class's **instance initializer blocks** and **field initializers** run, followed by the **constructor body**.

### 💻 Code Example

```java
class Animal {
    String name;
    Animal() {
        System.out.println("Animal no-arg");
        name = "unknown";
    }
    Animal(String name) {
        System.out.println("Animal(name)");
        this.name = name;
    }
}

class Dog extends Animal {
    int age;
    { System.out.println("Dog instance initializer"); }  // runs after super() returns
    Dog() {
        // implicit super() here
        System.out.println("Dog no-arg");
    }
    Dog(String name, int age) {
        super(name);                    // explicit — skips Animal()
        this.age = age;
        System.out.println("Dog(name, age)");
    }
}

public class Demo {
    public static void main(String[] args) {
        new Dog("Rex", 3);
        // Prints:
        // Animal(name)
        // Dog instance initializer
        // Dog(name, age)
    }
}
```

### ⚠️ Common Pitfalls
- Calling a method from a constructor that the subclass overrides — the subclass version runs before its own fields are initialized. Classic NPE trap.
- `this(...)` and `super(...)` cannot coexist in the same constructor (both must be first statement).
- Forgetting that `final` instance fields must be assigned exactly once before the constructor finishes.

### 🎤 Interview Angle
> "What's printed if a subclass constructor calls `super()` that calls an overridden method?"

You must show awareness that polymorphism happens during construction — the overridden method runs on a partially-initialized subclass, often seeing default values for subclass fields. This is why Joshua Bloch says: **never call overridable methods from constructors**.

### 🧪 Practice
1. Create a `Vehicle` → `Car` → `ElectricCar` chain. Log the constructor order when creating an `ElectricCar`.
2. Write a class with 3 overloaded constructors that all delegate to a single "primary" constructor via `this(...)`.
3. Force a NullPointerException by calling an overridable method from a superclass constructor.

### 🔍 Self-check
1. What does the implicit `super()` do if the parent has no no-arg constructor?
2. Why can't `this(...)` and `super(...)` appear in the same constructor?
3. What's the difference between an instance initializer block `{ ... }` and a static initializer block `static { ... }`?
4. Where does a local variable live vs an instance field?

---

## Tuesday — Inheritance, Polymorphism, Overloading vs Overriding

### 🎯 Objective
Distinguish compile-time vs runtime polymorphism with a concrete example; predict which method resolves in tricky dispatch cases.

### 📖 Core Concept

**Inheritance** uses `extends` to acquire parent fields and methods. Java supports **single inheritance** of classes but **multiple inheritance of types** via interfaces.

Two flavors of polymorphism:

**1. Compile-time polymorphism (method overloading)** — multiple methods with the same name in the same class, differing by parameter list. Resolved by the compiler based on **static (declared) types** of arguments.

```java
class Printer {
    void print(Object o) { System.out.println("Object"); }
    void print(String s) { System.out.println("String"); }
}

Printer p = new Printer();
Object o = "hello";
p.print(o);   // prints "Object" — static type is Object
```

**2. Runtime polymorphism (method overriding)** — subclass provides its own version of an inherited method. Resolved by the JVM based on the **runtime (actual) type** of the object.

```java
class Animal { void speak() { System.out.println("..."); } }
class Dog extends Animal { void speak() { System.out.println("Woof"); } }

Animal a = new Dog();
a.speak();   // "Woof" — runtime type is Dog
```

**Rules for overriding:**
- Same method signature (name + parameter types).
- Return type: same or a **covariant** subtype.
- Access: same or **more permissive** (`protected` → `public` OK, not reverse).
- Exceptions: cannot throw broader checked exceptions than the parent.
- Can't override `static`, `final`, or `private` methods (private = hidden, not overridden).

### 💻 Code Example

```java
class Shape {
    public Shape clone() { return new Shape(); }   // covariant return
}

class Circle extends Shape {
    @Override
    public Circle clone() { return new Circle(); }  // returns subtype — valid
}

// Dispatch example:
Shape s = new Circle();
Shape copy = s.clone();                            // copy is actually a Circle
System.out.println(copy.getClass().getSimpleName()); // "Circle"
```

### ⚠️ Common Pitfalls
- **Static methods are NOT overridden — they're hidden.** Calling a static method through a reference uses the declared type, not runtime type.
- Overloading resolution uses **static** types — changing a variable's declared type can change which overload runs with no change to the object.
- Adding an `@Override` annotation is optional but catches typos (`toSring()` won't compile).

### 🎤 Interview Angle
> "Is this overloading or overriding?" — given a snippet.

Also: "What prints?" with a `Parent p = new Child();` and overloaded/overridden combos. Always check: static method? Field access? Private? Those aren't dispatched dynamically.

### 🧪 Practice
1. Write a class hierarchy demonstrating covariant return types.
2. Create 3 overloads of `log(...)` and predict which runs for `log(null)`. (Compile error — ambiguous — you'll need a cast.)
3. Demonstrate field hiding: child class declares a field with the same name as parent. Show that field access is NOT polymorphic.

### 🔍 Self-check
1. Why doesn't `p.print(o)` above call the `String` overload even though the object is actually a String?
2. Can a subclass override a method and change the return type? Under what condition?
3. What's the difference between method hiding (static) and method overriding (instance)?
4. Why is calling an overridable method from a constructor dangerous?

---

## Wednesday — Abstraction, Encapsulation, Abstract Classes, Interfaces

### 🎯 Objective
Design a class hierarchy using both abstract classes and interfaces, choosing correctly between them.

### 📖 Core Concept

**Encapsulation** — bundling data + operations while restricting direct access to internal state. Achieved by making fields `private`, exposing behavior via methods, validating input.

**Abstraction** — presenting a clean interface while hiding implementation. Achieved via abstract classes and interfaces.

**Abstract class**:
- Declared with `abstract` keyword.
- May have abstract methods (no body) AND concrete methods (with body).
- Can have state (fields), constructors, static members.
- Cannot be instantiated directly — subclass must provide implementations for abstract methods.
- A class extends **one** abstract class.

**Interface**:
- A pure contract (originally). Since Java 8: can have `default` and `static` methods with bodies. Since Java 9: `private` helper methods.
- Fields are implicitly `public static final` (constants only — no instance state).
- A class can implement **many** interfaces.
- Methods are implicitly `public abstract` (unless `default`/`static`/`private`).

### 💻 Code Example

```java
// Abstract class — partial implementation with shared state
abstract class Shape {
    protected String color;
    Shape(String color) { this.color = color; }
    abstract double area();                          // must override
    void describe() {                                // shared implementation
        System.out.println(color + " shape, area=" + area());
    }
}

// Interfaces — capabilities
interface Drawable { void draw(); }
interface Resizable {
    void resize(double factor);
    default void doubleSize() { resize(2.0); }        // default method
}

class Circle extends Shape implements Drawable, Resizable {
    double radius;
    Circle(String color, double r) { super(color); this.radius = r; }
    double area() { return Math.PI * radius * radius; }
    public void draw() { System.out.println("drawing " + color + " circle"); }
    public void resize(double f) { radius *= f; }
}
```

### ⚠️ Common Pitfalls
- **Diamond problem with default methods:** if two interfaces define the same default method, the implementing class must override and disambiguate with `InterfaceName.super.method()`.
- Forgetting that interface fields are `public static final` — any "field" in an interface is a shared constant.
- Using `abstract` + `final` together — compile error (nothing could implement it).

### 🎤 Interview Angle
> "When would you use an abstract class vs an interface?"

Abstract class when you have **shared state or implementation** and want a true IS-A hierarchy (`LivingThing → Animal → Dog`). Interface when you want to express **capability** that can appear across unrelated types (`Comparable`, `Iterable`, `Serializable`). Since Java 8, this line has blurred — most modern designs prefer interfaces + composition.

### 🧪 Practice
1. Design a `PaymentGateway` interface with 3 implementations (`Stripe`, `PayPal`, `Razorpay`). Include a `default` retry method.
2. Create a `Vehicle` abstract class with shared fuel tracking + abstract `moveOneUnit()` method; implement `Car`, `Bike`.
3. Force the diamond problem with two interfaces defining the same `default` method; resolve it.

### 🔍 Self-check
1. Can you instantiate an abstract class? Why or why not?
2. Can an interface have a constructor?
3. How does Java avoid the traditional multiple-inheritance diamond problem?
4. Can a `default` method call a `private` helper method in the same interface? (Yes, since Java 9.)

---

## Thursday — Access Modifiers, Packages, Inner Classes, Anonymous Classes

### 🎯 Objective
Pick the correct access level for class members; know the 4 inner-class variants and when to use each.

### 📖 Core Concept

**Access modifiers** (most to least restrictive):

| Modifier | Same class | Same package | Subclass (diff pkg) | Anywhere |
|----------|:---------:|:------------:|:-------------------:|:--------:|
| `private` | ✅ | ❌ | ❌ | ❌ |
| (default, package-private) | ✅ | ✅ | ❌ | ❌ |
| `protected` | ✅ | ✅ | ✅ | ❌ |
| `public` | ✅ | ✅ | ✅ | ✅ |

**Rule of thumb:** start with the most restrictive, open up only as needed.

**Inner classes** (four kinds):

**1. Static nested class** — a class declared inside another with the `static` keyword. Has no reference to outer instance. Essentially a top-level class in the enclosing class's namespace.
```java
class Outer {
    static class Nested { }
}
Outer.Nested n = new Outer.Nested();
```

**2. Inner (non-static nested) class** — captures a reference to the enclosing instance. Cannot have static members (except constants).
```java
class Outer {
    class Inner { }
}
Outer.Inner i = new Outer().new Inner();
```

**3. Local class** — declared inside a method body. Can capture effectively final local variables.
```java
void method() {
    class Local { }
    Local l = new Local();
}
```

**4. Anonymous class** — unnamed class declared and instantiated in a single expression. Pre-lambda idiom.
```java
Runnable r = new Runnable() {
    public void run() { System.out.println("running"); }
};
```

### 💻 Code Example

```java
public class LinkedList<E> {
    private Node<E> head;

    // Static nested — no reference to enclosing instance
    private static class Node<E> {
        E value;
        Node<E> next;
        Node(E v) { value = v; }
    }

    // Inner class — holds implicit reference to the enclosing LinkedList
    private class Iter implements Iterator<E> {
        Node<E> current = head;   // accesses outer 'head'
        public boolean hasNext() { return current != null; }
        public E next() { E v = current.value; current = current.next; return v; }
    }

    public Iterator<E> iterator() { return new Iter(); }
}
```

### ⚠️ Common Pitfalls
- **Non-static inner classes prevent GC of the outer instance** — common memory-leak cause, especially in Android.
- Anonymous classes capture variables from the enclosing scope — those variables must be **effectively final** (not reassigned after capture).
- Forgetting that `this` inside an anonymous class refers to the anonymous class, not the enclosing class — use `OuterClass.this` to access the outer.

### 🎤 Interview Angle
> "Why prefer a static nested class over an inner class?"

Answer: if the nested class doesn't need access to the outer instance, static avoids the hidden reference (saves memory, prevents accidental leaks, easier to reason about). Joshua Bloch's Effective Java: **always make nested classes static unless you specifically need access to enclosing-instance members**.

### 🧪 Practice
1. Implement a mini linked list with an internal static `Node<E>` class.
2. Create an iterator as an inner class; observe that it can access outer `private` state.
3. Capture a loop variable in an anonymous `Runnable` — see the "effectively final" rule fire.

### 🔍 Self-check
1. Why would making `Node` non-static in the linked list above be a bad idea?
2. What's the difference between `Outer.this` and `this` inside an inner class?
3. Can a local class access a non-final local variable? (Only if it's effectively final.)
4. Is `protected` more or less restrictive than package-private for classes in the same package?

---

## Friday — `final`, `static`, `instanceof`, Pattern Matching

### 🎯 Objective
Use `final` correctly across variables, methods, classes; use `static` deliberately; apply modern `instanceof` pattern matching.

### 📖 Core Concept

**`final` variable** — assigned exactly once.
- **Final primitive** — value can't change.
- **Final reference** — reference can't be reassigned, BUT the referenced object can still mutate (unless the object itself is immutable).
- **Final local variable** — can be initialized once anywhere before use.
- **Final instance field** — must be initialized by the end of every constructor. Gains safe-publication guarantees in the JMM.

**`final` method** — cannot be overridden by subclasses.

**`final` class** — cannot be subclassed (e.g., `String`, `Integer`).

**`static` variable** — one copy per class, shared by all instances. Lives in the Method Area / Metaspace.

**`static` method** — belongs to the class; no implicit `this`. Can't access instance state directly.

**`static` block** — runs once when the class is initialized.

**`instanceof`** — checks runtime type. Returns `false` for `null` (safe).

**Pattern matching for `instanceof`** (Java 16+) — binds a variable of the tested type on success, eliminating the cast:
```java
if (obj instanceof String s) {
    // s is a String; no cast needed
}
```

### 💻 Code Example

```java
final class Money {                    // final class — can't subclass
    private final String currency;     // final field — set once
    private final long cents;

    public Money(String currency, long cents) {
        this.currency = currency;      // must assign in constructor
        this.cents = cents;
    }

    public Money add(Money other) {
        if (!currency.equals(other.currency))
            throw new IllegalArgumentException("currency mismatch");
        return new Money(currency, cents + other.cents);  // returns NEW instance
    }
}

// Static utility class pattern
final class Strings {
    private Strings() {}               // prevent instantiation
    public static boolean isBlank(String s) { return s == null || s.trim().isEmpty(); }
}

// Pattern matching
Object obj = fetch();
String desc = switch (obj) {            // Java 21 pattern matching in switch
    case Integer i  -> "int: " + i;
    case String s when !s.isEmpty() -> "str: " + s;
    case null       -> "null";
    default         -> "other";
};
```

### ⚠️ Common Pitfalls
- `final int[] arr = {1,2,3}; arr[0] = 99;` — legal! The array reference is final, the contents aren't.
- `static` fields initialized with `new SomethingHeavy()` run when the class loads — can trigger surprise initialization.
- Using `instanceof` to branch on types is often a design smell — prefer polymorphism unless you can't modify the types.

### 🎤 Interview Angle
> "How does `final` help with thread safety?"

`final` fields are safely published by the JMM: once a constructor completes, any thread that sees the object reference is **guaranteed** to see the final fields' fully-initialized values. No `volatile` needed. This is why immutable objects are thread-safe for free.

### 🧪 Practice
1. Write an immutable `Money` class with add/subtract/multiply returning new instances.
2. Create a `DatabaseConfig` utility using a private constructor + static methods.
3. Refactor a chain of `if (obj instanceof X) { X x = (X) obj; ... }` to pattern matching.

### 🔍 Self-check
1. Is a `final` collection immutable? (No — only the reference is final.)
2. Why is `String` marked `final`?
3. Can a `static` method access `this`?
4. When does a `static` initializer block run?

---

## Saturday — String, StringBuilder, StringBuffer, String Pool

### 🎯 Objective
Explain string immutability, the constant pool, and why `StringBuilder` is 100× faster for loop concatenation.

### 📖 Core Concept

**`String` is immutable.** Every "modification" (concat, substring, replace) returns a NEW `String` object.

**The String Constant Pool (SCP)** is a special region in the heap (since Java 7; used to be in PermGen). String **literals** go here automatically. `new String("abc")` creates a heap object **outside** the pool. `intern()` adds a string to the pool (or returns the existing one).

```java
String a = "hello";                   // in pool
String b = "hello";                   // returns SAME pooled instance
String c = new String("hello");       // new heap object, NOT in pool
String d = c.intern();                // returns pooled instance

a == b;       // true
a == c;       // false (different objects)
a == d;       // true
a.equals(c);  // true (content equality)
```

**`StringBuilder`** — mutable, **not thread-safe**, fast. Default buffer 16 chars, doubles + 2 when full.

**`StringBuffer`** — mutable, thread-safe (synchronized methods). Slower than StringBuilder. Rarely needed — thread safety is usually wrong at this layer.

### 💻 Code Example

```java
// Bad — creates N intermediate String objects
String result = "";
for (int i = 0; i < 10_000; i++) {
    result = result + i;              // O(n²) — copies full string each iteration
}

// Good — single buffer, amortized O(n)
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10_000; i++) {
    sb.append(i);
}
String result = sb.toString();

// Java 8+ — compiler optimizes simple + to StringBuilder automatically,
// but NOT across loop iterations or method boundaries.
```

**Text blocks** (Java 15+):
```java
String json = """
        {
          "name": "Alice",
          "age": 30
        }
        """;
// Preserves intent, handles indentation, no escaping quotes.
```

### ⚠️ Common Pitfalls
- Using `==` to compare strings — works for literals by coincidence, breaks for `new String(...)` or user input.
- Repeated `+=` on String in a loop — quadratic performance.
- `String.intern()` overuse — fills the pool; interned strings compete for the fixed hash table inside.
- `substring()` behavior changed in Java 7 — pre-7 it shared the char array (could leak memory); post-7 it copies. Trivia, but asked.

### 🎤 Interview Angle
> "How many String objects are created by `String s = new String(\"abc\");`?"

One or two:
- If `"abc"` was **not** already in the pool, the literal creates a pooled String → then `new String(...)` creates a second object in the heap. Total: **2**.
- If `"abc"` was already in the pool, only the heap object is new. Total: **1**.

### 🧪 Practice
1. Write a benchmark comparing `+` vs `StringBuilder` for 100,000 concatenations.
2. Demonstrate `intern()` — create two strings, confirm `==` is false, intern them, confirm `==` is true.
3. Build a CSV row with `StringBuilder` handling escaping (embedded commas and quotes).

### 🔍 Self-check
1. Where does the string pool live since Java 7?
2. What's the difference between `equals()` and `==` for Strings?
3. Why is `StringBuffer` rarely the right choice today?
4. How many objects does `String s1 = "a" + "b" + "c"` create at runtime? (One — compiler folds literals.)

---

## Sunday — Revision & Q&A

### 🧭 Revision checklist

Spend 30–45 minutes on each:
- [ ] Constructor chaining: draw the call order for a 3-level hierarchy.
- [ ] Overloading vs overriding: write 5 snippets and predict output.
- [ ] Abstract class vs interface: list 5 differences.
- [ ] Inner class types: when to use each.
- [ ] `final` semantics across variable/method/class.
- [ ] String pool mechanics + `intern()`.

### 📝 Preparing for the interview Q&A

Ask Claude: **"Prepare a Phase 1 Week 1 interview Q&A doc"** — it will generate 20 questions covering all 6 days. Answer in your own words, then ask for a review and rating.

### 📂 Commit your work

```
phase-1/
└── week-01/
    ├── exercises/
    │   ├── ConstructorChainDemo.java
    │   ├── PaymentGateway/... (interface + 3 impls)
    │   ├── Money.java
    │   └── StringBenchmark.java
    └── notes.md
```
