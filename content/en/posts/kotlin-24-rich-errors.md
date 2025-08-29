---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Kotlin 2.4 Rich Errors: What They Are and How to Prepare"
date: 2025-08-29T08:00:00+01:00
description: "An overview of Kotlin’s Rich Errors in Kotlin 2.4: goals, mental model, examples, interop with exceptions and Result, and how to prepare your codebase today."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/rich-errors.png
draft: false
tags: 
- kotlin
- error-handling
- exceptions
- result
- kotlin-2-4
---

### Kotlin 2.4 Rich Errors: What They Are and How to Prepare

Kotlin 2.4 introduces “Rich errors” — a more expressive, structured way to represent and propagate failures. The goal is clear: make error flows visible and composable across your codebase and platforms, without losing Kotlin’s ergonomics or its great interop story.

This article explains the problems Rich errors solve, how they relate to today’s exceptions and Result, what to expect in terms of mental model and interop, and how to prepare your codebase to adopt it smoothly.

---

#### Why Rich Errors?

Traditional exception-based error handling is powerful but has some drawbacks:

- Hidden control flow: exceptions don’t appear in function signatures
- Mixed concerns: thrown types are not always explicit or structured
- Composition friction: composing results across layers often requires try/catch
- Multiplatform nuances: mapping exceptions across different platforms can be uneven

Kotlin already offers `Result<T>` and functional helpers that address part of this. Rich errors extend the idea: make failure channels explicit, typed, and composable — while keeping excellent interop with existing code.

---

#### Mental Model

At a high level, Rich errors aim to:

- Make error types explicit and first-class (e.g., domain-specific error hierarchies)
- Compose cleanly across suspend/async boundaries
- Interoperate with exceptions (e.g., map from/to exceptions where needed)
- Preserve structured typing across multiplatform modules

Think of it as bringing the clarity of sealed error types plus the ergonomics of Kotlin’s standard tooling to the language and its ecosystem. The following patterns illustrate how it looks in practice.

---


#### Actual Syntax: Union‑Typed Errors

Kotlin 2.4 introduces union‑typed error returns. A function can declare its success type and the set of error variants it may produce, all in the return type:

```text
// Basic API with a union return
fun fetchUser(id: UserId): User | NotFound | NetworkFailure

// Suspending API with several domain errors
suspend fun uploadAvatar(file: ImageFile): Url | UnsupportedFormat | QuotaExceeded | NetworkFailure

// Propagating unions across layers
fun loadProfile(id: UserId): Profile | NotFound | NetworkFailure | DecodeError
```

At the call site, you handle the union with a when expression and smart‑casts:

```kotlin
when (val r = fetchUser(currentId)) {
    is User -> showProfile(r)
    is NotFound -> showMissingUser()
    is NetworkFailure -> showOfflineMessage(r)
}
```

You can also map a union to UI state or another union, preserving exhaustiveness:

```text
fun toUi(result: AuthToken | InvalidCredentials | NetworkFailure): UiState = when (result) {
    is AuthToken -> UiState.Authenticated(result)
    is InvalidCredentials -> UiState.Error("Invalid email or password")
    is NetworkFailure -> UiState.Error("Check your connection and try again")
}
```

Note: Names in these examples are illustrative; the key idea is that the error channel is first‑class in the function type, enabling better tooling, exhaustiveness checking, and composition.

---

#### Result vs Rich Errors: Side-by-Side

Here are two small, realistic comparisons that show how you’d write the same flow with Kotlin’s Result today and with Rich errors.

- Example A — Read and parse a config file

```kotlin
// Result today
fun loadConfig(path: Path): Result<Config> =
    runCatching { fileSystem.readBytes(path) }
        .mapCatching { bytes -> parseConfig(bytes) }
        .recover { e ->
            if (e is NoSuchFileException) defaultConfig() else throw e
        }

fun useConfig(path: Path) {
    loadConfig(path)
        .onSuccess { cfg -> startApp(cfg) }
        .onFailure { e ->
            when (e) {
                is NoSuchFileException -> showMissingConfigWarning()
                is ConfigFormatException -> showConfigParseError(e)
                else -> showGenericError(e)
            }
        }
}
```

```text
// Rich errors
// Define error variants (could be data objects/classes in your domain)
data object IoFailure
data class ParseFailure(val details: String)

fun loadConfig(path: Path): Config | IoFailure | ParseFailure

fun useConfigRich(path: Path) {
    when (val r = loadConfig(path)) {
        is Config -> startApp(r)
        is IoFailure -> showMissingConfigWarning()
        is ParseFailure -> showConfigParseErrorMessage(r.details)
    }
}
```

- Example B — Compose two calls (session -> dashboard)

```kotlin
// Result today
suspend fun fetchSessionResult(userId: UserId): Result<Session>
suspend fun fetchDashboardResult(session: Session): Result<Dashboard>

inline fun <T, R> Result<T>.flatMap(transform: (T) -> Result<R>): Result<R> =
    fold(onSuccess = transform, onFailure = { Result.failure(it) })

suspend fun loadHomeResult(userId: UserId): Result<Dashboard> =
    fetchSessionResult(userId).flatMap { sess -> fetchDashboardResult(sess) }
```

```text
// Rich errors
// Domain error variants
data object AuthFailure
data object NetworkFailure
data object BackendFailure

suspend fun fetchSession(userId: UserId): Session | AuthFailure | NetworkFailure
suspend fun fetchDashboard(session: Session): Dashboard | NetworkFailure | BackendFailure

suspend fun loadHome(userId: UserId): Dashboard | AuthFailure | NetworkFailure | BackendFailure =
    when (val s = fetchSession(userId)) {
        is Session -> when (val d = fetchDashboard(s)) {
            is Dashboard -> d
            is NetworkFailure -> d
            is BackendFailure -> d
        }
        is AuthFailure -> s
        is NetworkFailure -> s
    }
```

---

#### Interop: Exceptions, Result, Coroutines, and Multiplatform

- Exceptions: Libraries and platform APIs that throw continue to work. Rich errors provide mapping in both directions so you can work with typed errors internally and convert to exceptions at the boundaries (or vice versa).
- Result: It remains straightforward to adapt between `Result<T>` and a richer typed error representation when crossing layers.
- Coroutines: Cancellation remains special and should not be swallowed; treat `CancellationException` as a non-error control flow signal.
- Multiplatform: Keep error domains in `commonMain` as sealed interfaces; provide platform-specific mappings when facing platform exceptions.

---

#### Migration Strategy You Can Start Today

- Model pragmatic sealed error types where they provide value (don’t overdo it)
- Keep throwing at boundaries only; internally prefer typed error channels
- Add lightweight mappers to convert exceptions <-> typed errors

---

#### Pitfalls and Gotchas

- Over-modeling: Keep error types pragmatic; don’t explode the hierarchy
- Boundary clarity: Decide where you convert between exceptions and typed errors
- Cancellation: Always rethrow `CancellationException`
- Logging: Centralize logging; don’t double-log at multiple layers

---

#### Takeaways
- Rich errors make failures explicit, typed, and composable
- You can get 80% of the benefits today using sealed error types + Result/Outcome
- Model errors in `commonMain` for KMP and convert at platform boundaries
