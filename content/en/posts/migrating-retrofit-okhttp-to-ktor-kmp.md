---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "From Retrofit/OkHttp to Ktor in Kotlin Multiplatform: A Practical First Migration"
date: 2025-09-12T08:00:00+01:00
description: "Step-by-step guide to migrate your Android networking layer from Retrofit/OkHttp to Ktor in a Kotlin Multiplatform project, with CIO or OkHttp engines and minimal impact beyond the remote layer."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/ktor.png
draft: false
tags: 
- kotlin
- multiplatform
- kmp
- ktor
- retrofit
- okhttp
- android
- networking
- migration
---

### From Retrofit/OkHttp to Ktor in Kotlin Multiplatform: A Practical First Migration

If you want to start migrating an existing Android app to Kotlin Multiplatform (KMP), the networking layer is an excellent first step. Ktor Client works across platforms and lets you keep a single HTTP stack for Android, iOS, Desktop, and more. This guide shows how to migrate from Retrofit/OkHttp to Ktor with either CIO or OkHttp engines — while keeping the impact limited to the remote layer when your architecture is clean.

---

#### Why Networking First? And What Should Change?

With good modular architecture (e.g., Clean Architecture + Repository/DataSources), only your remote DataSources implementations should change:

- Domain layer (entities, use cases) remains untouched
- Repository contracts remain the same
- Local data sources (e.g., Room/SQLDelight) remain the same
- Only Remote data source implementation swaps Retrofit → Ktor

This keeps risk small and the migration incremental.

---

#### Dependencies and Setup (KMP Shared Module)

Add Ktor Client and Kotlinx Serialization to your shared module. Use CIO or OkHttp engine on Android; use CIO or Darwin on iOS (CIO lets both mobile platforms share the same engine). Keep versions aligned with your Kotlin/AGP setup.

```kotlin
// build.gradle.kts of shared module
plugins {
    kotlin("multiplatform")
    id("com.android.library")
    kotlin("plugin.serialization") // for kotlinx.serialization
}

kotlin {
    androidTarget()
    iosX64(); iosArm64(); iosSimulatorArm64()

    sourceSets {
        val ktorVersion = "2.3.12"
        val commonMain by getting {
            dependencies {
                implementation("io.ktor:ktor-client-core:$ktorVersion")
                implementation("io.ktor:ktor-client-content-negotiation:$ktorVersion")
                implementation("io.ktor:ktor-serialization-kotlinx-json:$ktorVersion")
                implementation("io.ktor:ktor-client-logging:$ktorVersion")
                // Optional: Resources, Auth, etc.
                // implementation("io.ktor:ktor-client-auth:$ktorVersion")
            }
        }
        val androidMain by getting {
            dependencies {
                // Choose ONE: OkHttp or CIO
                implementation("io.ktor:ktor-client-okhttp:$ktorVersion")
                // implementation("io.ktor:ktor-client-cio:$ktorVersion")
            }
        }
        val iosMain by getting {
            dependencies {
                implementation("io.ktor:ktor-client-cio:$ktorVersion") // or use Darwin for the native stack
                // implementation("io.ktor:ktor-client-darwin:$ktorVersion")
            }
        }
    }
}
```

Tip: Prefer CIO on both Android and iOS to share the same engine: use `ktor-client-cio` on both. Alternatively, keep OkHttp on Android (`ktor-client-okhttp`) and Darwin on iOS (`ktor-client-darwin`).

---

#### HttpClient Factory with Engines (CIO or OkHttp)

Create a single place to configure your HttpClient. Use expect/actual to provide the engine per platform and to keep the rest of your code common.

```kotlin
// commonMain (imports omitted in snippet)
expect fun provideEngine(): HttpClientEngineFactory<*>

fun createHttpClient(baseUrl: String, enableLogs: Boolean = true): HttpClient =
    HttpClient(provideEngine()) {
        expectSuccess = false // we will handle non-2xx manually

        install(ContentNegotiation) {
            json(
                Json {
                    ignoreUnknownKeys = true
                    isLenient = true
                    encodeDefaults = true
                }
            )
        }

        install(HttpTimeout) {
            requestTimeoutMillis = 15_000
            connectTimeoutMillis = 10_000
            socketTimeoutMillis = 15_000
        }

        if (enableLogs) {
            install(Logging) {
                logger = Logger.DEFAULT
                level = LogLevel.INFO
            }
        }

        // Default headers or base URL can be handled per request or via a helper
    }
```

Provide platform engines:

```kotlin
// androidMain (imports omitted in snippet)
actual fun provideEngine(): HttpClientEngineFactory<*> = OkHttp
```

```kotlin
// jvmMain (if used) (imports omitted in snippet)
actual fun provideEngine(): HttpClientEngineFactory<*> = CIO
```

```kotlin
// iosMain (imports omitted in snippet)
actual fun provideEngine(): HttpClientEngineFactory<*> = CIO // or Darwin if you prefer the native stack
```

OkHttp-specific engine configuration (optional):

```kotlin
// androidMain - inside HttpClient(OkHttp) { engine { ... } } if you inline it
engine {
    config {
        followRedirects(true)
        retryOnConnectionFailure(true)
    }
    addInterceptor { chain ->
        val request = chain.request().newBuilder()
            .header("X-App-Version", "1.0")
            .build()
        chain.proceed(request)
    }
}
```

CIO-specific tuning:

```kotlin
// jvmMain - with CIO
engine {
    requestTimeout = 15_000
    threadsCount = 4
    pipelining = true
}
```

---

#### From Retrofit Services to Ktor Calls

Typical Retrofit service:

```kotlin
// Retrofit
interface UserService {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: String): UserDto

    @POST("users")
    suspend fun createUser(@Body body: CreateUserRequest): UserDto
}
```

Ktor replacement in the remote implementation (keep the interface you already use in your repository):

```kotlin
// Common remote contract used by Repository
data class CreateUserRequest(val name: String)

interface UserRemote {
    suspend fun getUser(id: String): UserDto
    suspend fun createUser(body: CreateUserRequest): UserDto
}

class UserRemoteImpl(
    private val client: HttpClient,
    private val baseUrl: String,
) : UserRemote {
    override suspend fun getUser(id: String): UserDto =
        client.get("$baseUrl/users/$id").body()

    override suspend fun createUser(body: CreateUserRequest): UserDto =
        client.post("$baseUrl/users") {
            contentType(io.ktor.http.ContentType.Application.Json)
            setBody(body)
        }.body()
}
```

Query parameters and headers map directly:

```kotlin
client.get("$baseUrl/search") {
    url {
        parameters.append("q", "john")
        parameters.append("page", "1")
    }
    headers.append("Authorization", "Bearer $token")
}
```

---

#### Handling Errors and Mapping Them

Unlike Retrofit, Ktor won’t throw for non-2xx if `expectSuccess = false`. Handle responses explicitly or catch client/server exceptions.

```kotlin
suspend fun <T> HttpResponse.safeBody(): T {
    if (status.isSuccess()) return body()
    val raw = bodyAsText()
    // Parse domain error if your backend returns structured error JSON
    throw ApiException(status, raw)
}

class ApiException(val status: HttpStatusCode, message: String) : Exception(message)
```

Using it:

```kotlin
val response = client.get("$baseUrl/users/$id")
val user: UserDto = response.safeBody()
```

Alternatively, set `expectSuccess = true` and catch `ClientRequestException`/`ServerResponseException`.

---

#### Auth, Interceptors, and Logging

- DefaultRequest: set headers applied to every request
- Auth plugin: Bearer/Basic flows if you prefer
- Logging plugin: request/response logs (avoid PII in production)

```kotlin
HttpClient(provideEngine()) {
    install(DefaultRequest) {
        headers.append("Accept", "application/json")
        // headers.append("Authorization", "Bearer $token") // inject token via DI
    }
    install(Logging) {
        level = LogLevel.HEADERS
    }
}
```

With OkHttp engine, you can also reuse existing OkHttp Interceptors (e.g., network inspection, custom retries) as shown earlier.

---

#### Keep the Architecture Boundary Intact

- Keep your `UserRemote` (or similar) interface in commonMain
- Migrate only the implementation from Retrofit → Ktor
- Repositories depend on the interface, so the rest of the app doesn’t change

```kotlin
class UserRepository(private val remote: UserRemote) {
    suspend fun getUser(id: String) = remote.getUser(id)
}
```

---

#### Testing with Ktor MockEngine

You can test your remote implementation without hitting the network using MockEngine.

```kotlin
fun testClient(): HttpClient = HttpClient(MockEngine) {
    engine {
        addHandler { request ->
            if (request.url.fullPath == "/users/123") {
                respond(
                    content = """{"id": "123", "name": "Jane"}""",
                    headers = headersOf(HttpHeaders.ContentType, "application/json"),
                    status = HttpStatusCode.OK
                )
            } else respondError(HttpStatusCode.NotFound)
        }
    }
    install(ContentNegotiation) { json() }
}
```

---

#### Migration Checklist

- Add Ktor and kotlinx-serialization dependencies to shared module
- Create HttpClient factory with engine per platform (Android: OkHttp or CIO; iOS: CIO or Darwin)
- Install ContentNegotiation(Json), HttpTimeout, Logging
- Port your Retrofit services to a Ktor-based remote implementation
- Keep repository/domain unchanged; swap only the remote implementation in DI
- Handle errors consistently; consider a Result/Either wrapper
- Write MockEngine tests for happy/error paths

---

#### Frequently Asked Choices

- CIO vs OkHttp on Android?
  - OkHttp: reuse existing interceptors/TLS/pinning; familiar debugging
  - CIO: pure Ktor/JVM engine, fewer external deps
- Serialization: kotlinx-serialization is idiomatic in KMP; migrate DTOs to @Serializable
- Caching: implement at repository layer; Ktor doesn’t impose policy

---

#### Conclusion

Migrating Retrofit/OkHttp to Ktor is a focused, low-risk first step toward KMP. With a clean architecture, you only change remote interfaces/implementations while the rest of the app remains stable. Start with a single feature, validate with tests, and expand confidently across your codebase.
