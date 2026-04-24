# Phase 1 — Core Java Mastery (Weeks 1–7)

This phase builds your Java fundamentals from language basics to JVM internals. Everything after this (Spring, microservices, system design) sits on this foundation — don't rush through it.

## 📁 Week-by-week

| Week | File | Focus |
|------|------|-------|
| 1 | [week-01-oop-fundamentals.md](./week-01-oop-fundamentals.md) | OOP, classes, inheritance, interfaces, Strings |
| 2 | [week-02-exceptions-generics.md](./week-02-exceptions-generics.md) | Exception handling, generics, type erasure, wildcards |
| 3 | [week-03-collections.md](./week-03-collections.md) | List/Set/Map internals, iterators, comparators |
| 4 | [week-04-threading-basics.md](./week-04-threading-basics.md) | Threads, synchronized, volatile, locks |
| 5 | [week-05-concurrency-advanced.md](./week-05-concurrency-advanced.md) | Executors, CompletableFuture, concurrent collections |
| 6 | [week-06-jvm-internals.md](./week-06-jvm-internals.md) | ClassLoader, GC, memory model, tooling |
| 7 | [week-07-java8-design.md](./week-07-java8-design.md) | Streams, Optional, time API, SOLID, records |

## 🧭 How to use these docs

Each week has **6 day-sections (Mon–Sat)** + **Sunday revision prompts**.

Each day follows the same structure:
1. **Objective** — what you'll be able to explain by the end.
2. **Core concept** — the theory, with diagrams/tables where useful.
3. **Code example** — runnable snippets you should type and execute.
4. **Common pitfalls** — the bugs interviewers love to ask about.
5. **Interview angle** — how this gets asked + the nuance to show depth.
6. **Practice** — 2–3 exercises. Push them to a GitHub repo.
7. **Self-check** — 3–5 questions. Answer out loud before moving on.

## ✅ Completion criteria

Before moving to Phase 2, you should be able to (without notes):
- Explain the JVM's runtime memory layout and the lifecycle of an object from `new` to GC.
- Justify why `HashMap` is O(1) on average and O(log n) in the worst case.
- Describe 3 ways a `ConcurrentModificationException` can occur and how to avoid it.
- Write thread-safe code using 3 different mechanisms and defend the trade-offs.
- Refactor a "God class" applying SOLID without being told which principle to use.

If you can't yet — revisit the weak week before proceeding.

## 🔁 Sunday protocol (every week)

- Spend 90 minutes revising the week's topics.
- Ask me (Claude) to generate an interview Q&A doc for that week — you write answers, I review and rate.
- Spend 30 minutes tidying notes + committing practice code.

## 📚 Suggested companion resources

- **Book:** *Effective Java* (3rd edition) — read items matching the week's topic.
- **Book (Week 4–5 only):** *Java Concurrency in Practice* (Goetz) — Ch. 1–4.
- **Blog:** [Baeldung](https://www.baeldung.com/) for quick refreshers.
- **Docs:** Official Java API docs for the classes you touch that day.
