---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Mastering Kotlin Coroutines: Dispatchers, Jobs, and Structured Concurrency"
date: 2025-11-07T08:00:00+01:00
description: "A practical, in-depth guide to Kotlin Coroutines, Dispatchers, and Jobs with mental models, code recipes, pitfalls, and testing tips."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/coroutines.png
draft: false
tags: 
- kotlin
- coroutines
- concurrency
- async
- structured-concurrency
---

### Mastering Kotlin Coroutines: Dispatchers, Jobs, and Structured Concurrency

Kotlin Coroutines provide a lightweight, structured way to write asynchronous and concurrent code. This guide focuses on the three pillars you use every day: Dispatchers (where coroutines run), Jobs (what you are running and how it is supervised), and structured concurrency (the rules that keep your async code sane).

You’ll find mental models, small runnable snippets, gotchas, and production-ready patterns.

---

#### TL;DR

- Dispatcher: decides the thread or thread pool where a coroutine executes (`Dispatchers.Default`, `IO`, `Main`, `Unconfined`).
- Job: a handle to a running coroutine; supports lifecycle, cancellation, parent/child relations, and structured concurrency.
- Structured concurrency: launch coroutines in a scope; children are cancelled when the scope is cancelled; errors don’t leak silently.
- Prefer `withContext` for switching threads in a suspend function and `launch` for fire-and-forget in a lifecycle-aware scope.
- Always cancel scopes you create; prefer scope-provided dispatchers over global ones.

---

#### A Simple Mental Model

- Coroutine: a task that can suspend without blocking a thread.
- Dispatcher: the execution context (thread pool) the task uses.
- Job: the lifecycle controller for the task.
- Scope: binds a `CoroutineContext` (dispatcher + Job + extras) to a lifecycle.

```kotlin
class Repository(
    private val externalApi: ExternalApi,
    private val dispatcher: CoroutineDispatcher = Dispatchers.IO
) {
    suspend fun loadUser(id: String): User = withContext(dispatcher) {
        externalApi.fetchUser(id) // suspends without blocking a thread
    }
}
```

---

#### Dispatchers Deep Dive

- `Dispatchers.Default`: CPU-bound work (shared pool sized to CPU cores).
- `Dispatchers.IO`: blocking IO (files, sockets, JDBC). Uses a larger pool.
- `Dispatchers.Main`: UI thread on Android/desktop; only for light, quick work.
- `Dispatchers.Unconfined`: advanced/rare; starts where called and may resume elsewhere.

Switching dispatchers inside suspend functions:

```kotlin
suspend fun resizeAndSave(bitmap: Bitmap, file: File) {
    val resized = withContext(Dispatchers.Default) { resize(bitmap) } // CPU-bound
    withContext(Dispatchers.IO) { file.outputStream().use { save(resized, it) } } // IO-bound
}
```

Custom dispatcher (limited parallelism):

```kotlin
val singleIO = Dispatchers.IO.limitedParallelism(1)
```

---

#### Jobs and Lifecycle

A `Job` represents a cancellable unit of work. Every coroutine has one. Jobs can form hierarchies: cancelling a parent cancels all children.

```kotlin
val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)
val job: Job = scope.launch {
    // Do work
}

// Later
job.cancel("no longer needed")
```

Key types and helpers:

- `Job`: normal job; failure cancels parent unless supervised appropriately.
- `SupervisorJob`: children fail independently; parent is not cancelled on child failure.
- `coroutineContext[Job]`: access current job.

---

#### Structured Concurrency in Practice

- Launch coroutines within a scope you control (e.g., Android `viewModelScope`, `lifecycleScope`).
- Prefer `coroutineScope {}` or `supervisorScope {}` inside suspend functions to launch children.

```kotlin
suspend fun loadScreenData(repo: Repo): ScreenData = coroutineScope {
    val user = async { repo.user() }
    val posts = async { repo.posts() }
    ScreenData(user.await(), posts.await())
}
```

If one child must not cancel others, use `supervisorScope`:

```kotlin
suspend fun loadPartial(repo: Repo): Partial = supervisorScope {
    val user = async { repo.user() }
    val comments = async { repo.comments() } // might fail
    Partial(user.await(), runCatching { comments.await() }.getOrNull())
}
```

---

#### Cancellation and Timeouts

- Cancellation is cooperative: check `isActive` or call suspending functions that are cancellable.
- Use `withTimeout`/`withTimeoutOrNull` for upper bounds.

```kotlin
suspend fun poll(api: Api): Data? = withTimeoutOrNull(1_000) {
    while (isActive) {
        val data = api.tryFetch()
        if (data != null) return@withTimeoutOrNull data
        delay(50)
    }
    null
}
```

Always close or clean up in `finally`:

```kotlin
scope.launch(Dispatchers.IO) {
    val socket = openSocket()
    try {
        readLoop(socket)
    } finally {
        socket.close() // called on cancellation too
    }
}
```

---

#### Exception Handling

- Exceptions in a child cancel the parent in regular scopes.
- In a `SupervisorJob`/`supervisorScope`, siblings are isolated.
- Use `CoroutineExceptionHandler` only for top-level, uncaught exceptions.

```kotlin
val handler = CoroutineExceptionHandler { _, e ->
    log("Unhandled: ${e.message}")
}

val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main + handler)
```

For child failures you anticipate, catch locally:

```kotlin
scope.launch {
    try {
        fetch()
    } catch (e: IOException) {
        showError(e)
    }
}
```

---

#### Context Elements and `withContext`

`CoroutineContext` is a set of elements: `Dispatcher`, `Job`, `CoroutineName`, etc.

- Combine with `+`.
- Replace an element by adding another of the same key.

```kotlin
val scope = CoroutineScope(Dispatchers.Default + SupervisorJob() + CoroutineName("sync"))

suspend fun syncAll() = withContext(CoroutineName("sync-step")) {
    // Only name changed; dispatcher/job inherited
}
```

---

#### Choosing the Right Dispatcher

- CPU-bound: `Default` (or a dedicated limited pool if you need isolation).
- IO-bound: `IO`.
- UI updates: `Main`.
- Avoid `Unconfined` unless you know why you need it.

Anti-patterns:

- Spawning your own `newFixedThreadPoolContext` without a strong reason.
- Calling `withContext(Dispatchers.Main)` for long work.
- Using `GlobalScope` in app code (hard to cancel and test).

---

#### Common Patterns and Recipes

Fire-and-forget tied to lifecycle:

```kotlin
// Android ViewModel
class MyViewModel(/* ... */) : ViewModel() {
    fun refresh() = viewModelScope.launch { repository.refresh() }
}
```

Pipelining:

```kotlin
suspend fun process(input: Input): Output = withContext(Dispatchers.Default) {
    val parsed = parse(input)
    val enriched = withContext(Dispatchers.IO) { enrich(parsed) }
    transform(enriched)
}
```

Backpressure with `limitedParallelism`:

```kotlin
suspend fun fetchAll(ids: List<String>): List<Item> = coroutineScope {
    val gate = Dispatchers.IO.limitedParallelism(8)
    ids.map { id -> async(gate) { fetch(id) } }.awaitAll()
}
```

---

#### Testing Coroutines

- Use `kotlinx-coroutines-test`.
- Replace dispatchers and control virtual time.

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class RepoTest {
    private val dispatcher = StandardTestDispatcher()
    private val scope = TestScope(dispatcher)

    @Before fun setUp() { Dispatchers.setMain(dispatcher) }
    @After fun tearDown() { Dispatchers.resetMain() }

    @Test fun loadsUser() = scope.runTest {
        val repo = Repo(/* fakes */)
        val user = repo.user()
        assertEquals("42", user.id)
    }
}
```

---

#### Performance Tips and Pitfalls

- Keep coroutines coarse-grained; too many tiny coroutines add overhead.
- Avoid unnecessary context switches.
- Prefer structured scopes over ad-hoc `GlobalScope.launch`.
- Remember that `async` without `await` leaks errors; use `launch` for fire-and-forget.

---

#### Quick Reference

- Create a scope: `CoroutineScope(Job() + Dispatchers.Default)`
- Switch threads: `withContext(Dispatchers.IO) { ... }`
- Start child tasks: `coroutineScope { val a = async { ... }; a.await() }`
- Timeouts: `withTimeout(…)`, `withTimeoutOrNull(…)`
- Supervision: `SupervisorJob`, `supervisorScope { … }`
- Exception handler: `CoroutineExceptionHandler { ctx, e -> … }`
