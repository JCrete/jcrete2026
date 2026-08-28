# Project Loom API Evolution: Java 24 → 27

> Based on a JCrete 2026 discussion on July 28, 2026.  
> The technical structure below follows José Paumard's slides. The discussion notes at the end preserve the substance of the original session notes, with speaker attribution kept deliberately cautious where the recording does not provide reliable labels.

**Session contributors:** [José Paumard](https://fr.linkedin.com/in/jos%C3%A9-paumard-2458ba5), [Cay Horstmann](https://de.linkedin.com/in/cay-horstmann-659a4b), [Cliff Click](https://www.linkedin.com/in/clifford-click), Simon Valter, [Ben Evans](https://es.linkedin.com/in/kittylyst), and Borja Bravo Alférez.

## References

**Slides:** [José Paumard — Virtual Threads, Structured Concurrency and Scoped Values in Java 25 and beyond](https://speakerdeck.com/josepaumard/virtual-threads-structured-concurrency-et-scoped-values-en-java-25-et-au-dela)

**JEPs referenced in this session:**

| JEP | Title | Java |
|-----|-------|------|
| [491](https://openjdk.org/jeps/491) | Synchronize Virtual Threads without Pinning | 24 |
| [487](https://openjdk.org/jeps/487) | Scoped Values, Fourth Preview | 24 |
| [499](https://openjdk.org/jeps/499) | Structured Concurrency, Fourth Preview | 24 |
| [505](https://openjdk.org/jeps/505) | Structured Concurrency, Fifth Preview | 25 |
| [506](https://openjdk.org/jeps/506) | Scoped Values (Final) | 25 |
| [525](https://openjdk.org/jeps/525) | Structured Concurrency, Sixth Preview | 26 |

## The evolution in one sentence

**Java 24 strengthens the foundations → Java 25 replaces the Structured Concurrency programming model → Java 26 fixes and refines timeout and result semantics → Java 27 moves toward making the exception type part of the policy itself.**

## Java 24 — the old API baseline becomes more practical

### Virtual threads: `synchronized` stops causing pinning

Virtual threads have been final since Java 21. The important change in Java 24 is therefore not a new source-level API, but a JVM improvement: when a virtual thread blocks inside a `synchronized` block or method, it releases its carrier thread.

Before Java 24, a blocking call inside `synchronized` could pin a virtual thread to its platform thread. That undermined exactly the kind of scalability virtual threads are intended to provide. Java 24 addresses this with [JEP 491 — Synchronize Virtual Threads without Pinning](https://openjdk.org/jeps/491).

**Practical impact:** existing libraries no longer need to be proactively rewritten from `synchronized` to `ReentrantLock` merely to work well with virtual threads. Native frames and some JVM edge cases can still cause pinning.

### Scoped Values: the API becomes fully fluent

Scoped Values are still in their fourth preview in Java 24. The main API change is the removal of the standalone `ScopedValue.callWhere(...)` and `runWhere(...)` methods. Bindings now flow exclusively through a `Carrier`:

```java
ScopedValue.where(USER, user)
        .where(TRACE_ID, traceId)
        .run(() -> handleRequest());
```

This makes the lexical scope of a binding explicit in the code. See [JEP 487 — Scoped Values, Fourth Preview](https://openjdk.org/jeps/487).

### Structured Concurrency: still the constructor-and-subclass model

Structured Concurrency itself does not materially change in Java 24. It remains in its fourth preview via [JEP 499](https://openjdk.org/jeps/499). Policies are still embedded in classes such as:

- `StructuredTaskScope.ShutdownOnFailure`
- `StructuredTaskScope.ShutdownOnSuccess<T>`
- a directly constructed `StructuredTaskScope<T>`

A typical Java 24 fragment looks conceptually like this:

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var image = scope.fork(service::readImage);
    var links = scope.fork(service::readLinks);

    scope.join().throwIfFailed();
    return new Page(image.get(), links.get());
}
```

Timeouts are still specified while waiting, using `joinUntil(Instant deadline)`.

**This is the baseline that Java 25 largely replaces.**

## Java 25 — the major Structured Concurrency redesign

### Scoped Values become final

With [JEP 506](https://openjdk.org/jeps/506), `ScopedValue` becomes a permanent Java API. The fluent style introduced in Java 24 remains. Scoped-value bindings are also automatically inherited by virtual threads started inside a `StructuredTaskScope`, provided the lexical structure remains intact.

### Constructors and policy subclasses give way to `open(...)`

The biggest change discussed in the session comes from [JEP 505 — Structured Concurrency, Fifth Preview](https://openjdk.org/jeps/505).

Instead of public constructors and specialized scope subclasses, code now opens a scope through static factory methods:

```java
try (var scope = StructuredTaskScope.open()) {
    var image = scope.fork(service::readImage);
    var links = scope.fork(service::readLinks);

    scope.join();
    return new Page(image.get(), links.get());
}
```

The default policy of `open()` is:

- wait until all subtasks succeed, or one subtask fails;
- cancel the scope on failure;
- return `null` from `join()` when everything succeeds;
- throw `FailedException` from `join()` when a subtask fails.

### Policy becomes an object: `Joiner<T, R>`

For different behavior, `StructuredTaskScope.open(...)` accepts a `Joiner`. The Joiner determines two things that, before Java 25, were spread across subclasses and custom result-processing code:

1. **When the scope should be cancelled.**
2. **What `scope.join()` ultimately returns or throws.**

The Java 25 shape shown in the slides is:

```java
interface Joiner<T, R> {
    boolean onFork(Subtask<? extends T> subtask);
    boolean onComplete(Subtask<? extends T> subtask);
    R result() throws Throwable;
}
```

- `onFork(...)` is called by the owner thread before the virtual thread for the task is started.
- `onComplete(...)` is called by the executing virtual thread when the task succeeds or fails.
- Returning `true` from either callback means: cancel the scope.
- `result()` produces the value returned by `join()`, or throws the underlying failure.

Usage:

```java
Joiner<Result, List<Result>> joiner = ...;

try (var scope = StructuredTaskScope.open(joiner)) {
    tasks.forEach(scope::fork);
    return scope.join();
}
```

### Configuration moves to scope creation

The scope name, thread factory, and timeout become part of scope configuration in Java 25:

```java
try (var scope = StructuredTaskScope.open(
        joiner,
        config -> config
                .withName("page-builder")
                .withThreadFactory(Thread.ofVirtual().factory())
                .withTimeout(Duration.ofSeconds(1)))) {
    // fork + join
}
```

This is a fundamental shift from `joinUntil(...)`: the timeout now belongs to the **lifetime and policy of the entire scope**, rather than to one particular wait operation.

## Java 26 — same architecture, refined edges

Structured Concurrency remains a preview feature via [JEP 525](https://openjdk.org/jeps/525). The Java 25 architecture remains intact, but four important details are refined.

### 1. `Joiner.onTimeout()` is added

The Joiner is now explicitly informed when the configured timeout expires:

```java
interface Joiner<T, R> {
    boolean onFork(Subtask<T> subtask);
    boolean onComplete(Subtask<T> subtask);
    void onTimeout();
    R result() throws Throwable;
}
```

The default implementation of `onTimeout()` throws a `TimeoutException`. A custom Joiner may choose **not to throw an exception**. In that case, `join()` proceeds to call `result()`, allowing the Joiner to construct, for example, a partial result from subtasks that completed in time.

This timeout mechanism appeared in José's slides but had not yet been reached in the earlier transcript fragment.

### 2. `allSuccessfulOrThrow()` returns results instead of subtasks

In Java 25, this Joiner returned a stream of `Subtask` objects. In Java 26, it returns a `List<T>` containing the actual results:

```java
List<Result> results = StructuredTaskScope
        .open(Joiner.<Result>allSuccessfulOrThrow())
        // fork tasks
        .join();
```

This removes boilerplate such as `.map(Subtask::get)` from the normal happy path.

### 3. Naming becomes more consistent

`anySuccessfulResultOrThrow()` is renamed to `anySuccessfulOrThrow()`.

The supplied strategies are roughly:

- `allSuccessfulOrThrow()` — require all results; fail fast on an error;
- `anySuccessfulOrThrow()` — one successful result is sufficient;
- `awaitAllSuccessfulOrThrow()` — heterogeneous or side-effecting tasks; wait for success without returning a result list;
- `awaitAll()` — wait for everything and leave failures available for inspection;
- `allUntil(predicate)` — collect subtasks until a condition causes the scope to cancel.

### 4. Small generics and configuration cleanups

- `Subtask<? extends T>` becomes the simpler `Subtask<T>` in Joiner callbacks.
- The configuration function changes from a general `Function` to a `UnaryOperator<Configuration>`, because its input and output are the same configuration type.

## What the `onComplete()` discussion was really about

This was not a tangent; it follows directly from the Java 25/26 Joiner contract.

The lifecycle is:

1. `fork(task)` first creates a `Subtask` in state `UNAVAILABLE`.
2. `joiner.onFork(subtask)` is called.
3. The virtual thread starts.
4. On normal completion, the state changes to `SUCCESS` or `FAILED`.
5. The result or exception is stored in the `Subtask`.
6. `joiner.onComplete(subtask)` is called.

Once the scope is cancelled, however:

- unfinished virtual threads are interrupted;
- `onComplete()` is no longer called for tasks that complete only *after* that cancellation;
- some of those subtasks may nevertheless transition to `SUCCESS` or `FAILED` before the interrupt takes effect;
- `result()` is eventually called on the Joiner.

That led to the key point of contention:

> **Collecting state only in `onComplete()` does not necessarily give you a complete history of all subtasks.**

A Joiner that needs to inspect every created subtask later can retain the subtask references in `onFork()`. Its `result()` method can then inspect their current states.

This explains the disagreement in the room: José was correct that `onComplete()` is no longer guaranteed after cancellation; the counterargument was also valid that this does not automatically mean the information is hidden or lost.

A custom Joiner also needs to be thread-safe: `onComplete()` may be called concurrently from multiple virtual threads. Mutating a plain `ArrayList` from `onComplete()` is therefore unsafe; use synchronization or a concurrent collection.

## Java 27 — exceptions become part of the type contract

At the time of this session, Java 27 was still in the future and Structured Concurrency remained a preview feature. The direction was to make not only task type `T` and result type `R`, but also the exception type of `join()` statically explicit:

```java
interface Joiner<T, R, R_X extends Throwable> {
    boolean onFork(Subtask<T> subtask);
    boolean onComplete(Subtask<T> subtask);
    R result() throws R_X;
    // timeout handling is also tied to R_X
}
```

The April 2026 slides show a separate `timeout() throws R_X`. The subsequently evolving Java 27 preview API continued in the same general direction — typed exceptions and an explicit timeout cause — but the exact method shape was not final. The preview at the time introduced, among other things, `CancelledByTimeoutException` and exception-supplying functions for standard Joiners.

**Important:** treat Java 27 code from these slides as a design snapshot, not as a stable contract on which production code should already depend.

## The design arc from Java 24 to 27

- **Java 24:** policy is represented by scope subclasses, and timeout lives in `joinUntil(...)`.
- **Java 25:** policy becomes a `Joiner`; scope creation moves to `open(...)`; timeout becomes scope configuration.
- **Java 26:** timeout becomes visible to the Joiner and standard result handling becomes simpler.
- **Java 27:** the exception type becomes part of the generic policy, reducing `join()`'s dependence on generic wrapper exceptions.

That is the real through-line:

> **Structured Concurrency moves from a class hierarchy of specialized scopes toward a composable policy object that owns cancellation, results, timeout behavior, and eventually exception typing.**

## Participants and tentative speaker attribution

José Paumard led the technical walkthrough. According to the session notes, Cay Horstmann, Cliff Click, Simon Valter, and Ben Evans contributed most of the discussion; Borja Bravo Alférez intervened once.

The recording does not contain reliable speaker labels, so the safest tentative attribution is:

- **José Paumard:** version-by-version explanation, callback lifecycle, and cancellation semantics.
- **Person X — possibly Cliff Click:** the pointed question about why task state should be hidden when the scope knows all of its subtasks.
- **Person Y — possibly Cay Horstmann, Simon Valter, or Ben Evans:** the counterargument that missing `onComplete()` callbacks do not imply that the Joiner cannot retain subtask references or state.
- **Person Z:** additional remarks about `result()`, custom state, and race conditions.
- **Borja Bravo Alférez:** one intervention; the exact fragment cannot be attributed reliably. 

These names identify participants in the discussion, not definitive transcript attribution.

## Technical sources

- [José Paumard — Loom in JDK 25–26–27](https://speakerdeck.com/josepaumard/virtual-threads-structured-concurrency-et-scoped-values-en-java-25-et-au-dela)
- [JEP 491 — Synchronize Virtual Threads without Pinning](https://openjdk.org/jeps/491)
- [JEP 487 — Scoped Values, Fourth Preview](https://openjdk.org/jeps/487)
- [JEP 499 — Structured Concurrency, Fourth Preview](https://openjdk.org/jeps/499)
- [JEP 506 — Scoped Values, Final](https://openjdk.org/jeps/506)
- [JEP 505 — Structured Concurrency, Fifth Preview](https://openjdk.org/jeps/505)
- [JEP 525 — Structured Concurrency, Sixth Preview](https://openjdk.org/jeps/525)

---

## Discussion notes

### Business logic in a Joiner vs. a library-level abstraction

- Participant: “We should be worried about business developers. They shouldn’t use this directly. This should be a library-level thing, not something a business developer implements.”
- Ben: “Should there be a level in between?”
- Participant: “The implementer of the Joiner needs to understand the intricacies of the API well enough to express a policy. You gave an example involving a price and minimising something. If you leave the Joiner to the business programmer, they will intermix that policy with the concrete way of computing it.”
- Another participant: “In my business program I can see this use case, especially with AI. You have different providers. One may be slow or time out. You may want to say: if I already have a price, I can return despite a timeout. That is business logic, even though it is related to threads.”
- Response: “Figure out the general policy and make a Joiner that expresses that policy, with lambdas where people plug in the business-specific parts.”
- Response: “Human programmers doing business-rule work are not typically good at managing Joiner states.”
- Question: “Are you proposing removing the result callback?”
- Answer: “No. A class called `WeatherReportJoiner` is a code smell. The concurrency policy should be generic; weather-specific logic should be supplied separately.”

### Predicates, business-defined success, and scope configuration

- Participant: “There is also a Joiner that accepts predicates. That may fit: I can define when a result counts as successful, rather than success meaning only ‘no exception’.”
- Question: “Could we go back to `open`? It has two parameters, and the second is configuration. One configuration option is the thread factory, so I can run a structured-concurrency block with good old-fashioned platform threads, not just virtual threads.”
- Answer: “Exactly.”
- Question: “What is the intended use? We always discuss virtual threads, but the API allows this.”
- Answer: “The published use for the factory is mainly to label the threads [Note: you can also label the StructuredTaskScope itself] so they show up meaningfully in diagnostics.”
- Participant: “In practice there are two common things you want: label the threads and set a timeout.”
- Participant: “The API is crazily complex for doing those simple things. If you asked people to design an API for naming threads and setting a timeout, you would get many simpler designs.”

### Unit testing and where business logic belongs

- Participant: “That is Victor’s response, and I strongly disagree because I want a unit test.”
- Response: “I don’t care about unit-testing the Joiner. I want to unit-test my business logic.”
- Another participant: “This kind of threading code is difficult to unit-test.”
- Question: “Which code do you want to unit-test?”
- Answer: “The business logic.”
- Response: “If you cannot put the business logic there, and you do not want it in the Joiner, where do you put it?”
- Slide context: after `scope.join()`, a `Page` was constructed from `imagesSubtask.get()` and `linksSubtask.get()`.
- Underlying design question: the Joiner decides when enough results are available, but domain logic should preferably remain a pure, independently testable function or policy outside the concurrent state machine.

### Interrupting mounted and unmounted virtual threads

- Slide: when a `StructuredTaskScope` is cancelled, all virtual threads are interrupted and `onComplete()` is no longer invoked afterwards.
- Question: “Most virtual threads are not running. Only virtual threads currently mounted on carriers are running. How can you interrupt an unmounted thread that is dormant?”
- Answer: “It is an object. You call `.interrupt()` on it.”
- Question: “Does the pool then cycle through and remount every virtual thread?”
- Response: “I assume it has to become runnable to process the interrupt.”
- Counterpoint: “Historically interrupt was just a bit. You set the bit on the thread object. When it is scheduled again, it sees the bit and reacts. If it never needs to wake again, the stored bit costs almost nothing.”
- Response: “For interruptible I/O, the I/O operation may have to be woken up or aborted.”
- Example from the room: with one million suspended threads, cancellation does not necessarily mean one million threads must immediately be mounted. At the simplest level, interrupt status is recorded and only relevant work is resumed later to unwind.
- Nuance from the continued discussion: there is no hard kill. Cancellation is an interrupt request; a task stops only when its blocking operation or its code responds to that request.
- An answer came later in a discussion after the session. A virtual thread that is running a task that is currently blocking, and that is interrupted needs to be mounted again on a carrier thread, as the stack trace needs to be populated, and some application code in the catching of this exception needs to be executed.

### How does the caller know why `join()` returned?

- Question: “After `join()` you can execute code and then the scope closes. How do you know that it was cancelled?”
- Answer: “You cancelled it.”
- Response: “I did not cancel it. The Joiner may have done that, and perhaps I do not control the Joiner.”
- Answer: “You provide the Joiner when creating the `StructuredTaskScope`; the Joiner is part of your dependency and behavioural contract.”
- Response: “I could receive it from an external source. Suppose I am a framework that runs tasks and does not know the Joiner’s semantics.”
- Counterpoint: “Why do you need to know that cancellation happened? The Joiner decided the operation was complete. The meaningful public outcome is the Joiner’s result or exception, not the internal fact that sibling tasks were interrupted.”

### Is `onComplete()` the complete source of task state?

- Question: “Why hide task information from users? You know how many tasks were forked and whether they completed, failed or were still running.”
- Response: “No, that information is not all delivered through `onComplete()` after cancellation.”
- Counterpoint: “`onFork()` is always called, so the Joiner knows exactly which subtasks were created. It can retain those `Subtask` references and inspect them in `result()`.”
- Objection: “But `onComplete()` is the normal callback for knowing that a task completed. Why should I need a second mechanism when one callback returns `true` and cancels the scope?”
- Counterpoint: “The information is not necessarily lost merely because `onComplete()` is no longer called. The Joiner can keep the subtasks registered during `onFork()` and later inspect their states.”
- Core disagreement: one side treats `onComplete()` as the normative completion stream; the other sees it as a policy callback that stops once the scope outcome has been decided, while full inventory remains possible through retained `Subtask` handles.

### `close()` and tasks that ignore interruption

- Slide: `close()` waits until all threads have finished.
- Slide: if a task does not stop after interruption, `close()` does not return.
- Observation: this preserves the structured-concurrency guarantee that child tasks cannot outlive the scope, but cancellation remains cooperative.
- Positive diagnostic point from the slide: if a task remains stuck, there is at least a stack trace showing where it is stuck.

### TL;DR

- The central question in the discussion: why should task status be hidden from users if the scope knows how many subtasks were started and which completed or failed?
- `onComplete()` is not called for every still-running task once the scope has been cancelled.
- That does not automatically make all information inaccessible: the Joiner or its state can retain references or metadata during execution.
- The accumulated state can be analyzed in `result()`.

## Not covered in the live discussion

- The timeout mechanism was not reached during the live session and discussion.

## Open questions

- Which information does the API guarantee publicly, and which information is only available through retained `Subtask` handles?
- Which callbacks are invoked exactly on success, failure, cancellation, and timeout?
- How do you design a custom Joiner without races or hidden lifecycle assumptions?

## Provenance

These notes are based on a personal transcript fragment and session notes from JCrete 2026. Technical wording should be checked against the final JDK API before being treated as authoritative.
