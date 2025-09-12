---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "De Retrofit/OkHttp a Ktor en Kotlin Multiplatform: una primera migración práctica"
date: 2025-09-12T08:00:00+01:00
description: "Guía paso a paso para migrar tu capa de red de Android desde Retrofit/OkHttp a Ktor en un proyecto Kotlin Multiplatform, con motores CIO u OkHttp y mínimo impacto fuera de la capa remota."
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

### De Retrofit/OkHttp a Ktor en Kotlin Multiplatform: una primera migración práctica

Si quieres comenzar a migrar una app Android existente a Kotlin Multiplatform (KMP), la capa de red es un excelente primer paso. Ktor Client funciona en múltiples plataformas y te permite mantener una sola pila HTTP para Android, iOS, Desktop y más. Esta guía muestra cómo migrar de Retrofit/OkHttp a Ktor con motores CIO u OkHttp, manteniendo el impacto limitado a la capa remota cuando tu arquitectura está bien diseñada.

---

#### ¿Por qué empezar por red? ¿Y qué debería cambiar?

Con una arquitectura modular limpia (por ejemplo, Clean Architecture + Repository/DataSources), solo deberían cambiar tus implementaciones de DataSources remotas:

- La capa de dominio (entidades, casos de uso) permanece intacta
- Los contratos de repositorio permanecen iguales
- Las fuentes de datos locales (por ejemplo, Room/SQLDelight) permanecen iguales
- Solo la implementación remota cambia de Retrofit → Ktor

Esto reduce el riesgo y permite una migración incremental.

---

#### Dependencias y configuración (Módulo compartido KMP)

Añade Ktor Client y Kotlinx Serialization a tu módulo compartido. Usa el motor CIO u OkHttp en Android; usa CIO o Darwin en iOS (CIO permite compartir el mismo motor en móviles). Mantén las versiones alineadas con tu configuración de Kotlin/AGP.

```kotlin
// build.gradle.kts del módulo compartido
plugins {
    kotlin("multiplatform")
    id("com.android.library")
    kotlin("plugin.serialization") // para kotlinx.serialization
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
                // Opcional: Resources, Auth, etc.
                // implementation("io.ktor:ktor-client-auth:$ktorVersion")
            }
        }
        val androidMain by getting {
            dependencies {
                // Elige UNO: OkHttp o CIO
                implementation("io.ktor:ktor-client-okhttp:$ktorVersion")
                // implementation("io.ktor:ktor-client-cio:$ktorVersion")
            }
        }
        val iosMain by getting {
            dependencies {
                implementation("io.ktor:ktor-client-cio:$ktorVersion") // o usa Darwin para la pila nativa
                // implementation("io.ktor:ktor-client-darwin:$ktorVersion")
            }
        }
    }
}
```

Consejo: Prefiere CIO tanto en Android como en iOS para compartir el mismo motor: usa `ktor-client-cio` en ambos. Alternativamente, mantén OkHttp en Android (`ktor-client-okhttp`) y Darwin en iOS (`ktor-client-darwin`).

---

#### Factoría de HttpClient con motores (CIO u OkHttp)

Crea un único lugar donde configuras tu HttpClient. Usa expect/actual para proporcionar el motor por plataforma y mantener el resto del código en común.

```kotlin
// commonMain (imports omitidos en el snippet)
expect fun provideEngine(): HttpClientEngineFactory<*>

fun createHttpClient(baseUrl: String, enableLogs: Boolean = true): HttpClient =
    HttpClient(provideEngine()) {
        expectSuccess = false // gestionaremos manualmente las respuestas no 2xx

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

        // Cabeceras por defecto o base URL se pueden manejar por petición o con un helper
    }
```

Motores por plataforma:

```kotlin
// androidMain (imports omitidos)
actual fun provideEngine(): HttpClientEngineFactory<*> = OkHttp
```

```kotlin
// jvmMain (si aplica) (imports omitidos)
actual fun provideEngine(): HttpClientEngineFactory<*> = CIO
```

```kotlin
// iosMain (imports omitidos)
actual fun provideEngine(): HttpClientEngineFactory<*> = CIO // o Darwin si prefieres la pila nativa
```

Configuración específica de OkHttp (opcional):

```kotlin
// androidMain - dentro de HttpClient(OkHttp) { engine { ... } } si lo incrustas
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

Ajustes de CIO:

```kotlin
// jvmMain - con CIO
engine {
    requestTimeout = 15_000
    threadsCount = 4
    pipelining = true
}
```

---

#### De servicios de Retrofit a llamadas con Ktor

Servicio típico de Retrofit:

```kotlin
// Retrofit
interface UserService {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: String): UserDto

    @POST("users")
    suspend fun createUser(@Body body: CreateUserRequest): UserDto
}
```

Sustitución con Ktor en la implementación remota (mantén la interfaz que ya usa tu repositorio):

```kotlin
// Contrato remoto común usado por el Repositorio
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

Parámetros de query y cabeceras mapean directamente:

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

#### Gestión de errores y mapeo

A diferencia de Retrofit, Ktor no lanza excepción para códigos no-2xx si `expectSuccess = false`. Maneja las respuestas explícitamente o captura excepciones de cliente/servidor.

```kotlin
suspend fun <T> HttpResponse.safeBody(): T {
    if (status.isSuccess()) return body()
    val raw = bodyAsText()
    // Parsea el error de dominio si tu backend devuelve JSON estructurado
    throw ApiException(status, raw)
}

class ApiException(val status: HttpStatusCode, message: String) : Exception(message)
```

Uso:

```kotlin
val response = client.get("$baseUrl/users/$id")
val user: UserDto = response.safeBody()
```

Alternativamente, configura `expectSuccess = true` y captura `ClientRequestException`/`ServerResponseException`.

---

#### Autenticación, interceptores y logging

- DefaultRequest: cabeceras aplicadas a cada petición
- Auth plugin: flujos Bearer/Basic si lo prefieres
- Logging plugin: logs de request/response (evita PII en producción)

```kotlin
HttpClient(provideEngine()) {
    install(DefaultRequest) {
        headers.append("Accept", "application/json")
        // headers.append("Authorization", "Bearer $token") // inyecta el token vía DI
    }
    install(Logging) {
        level = LogLevel.HEADERS
    }
}
```

Con el motor OkHttp, también puedes reutilizar Interceptors existentes (p. ej., inspección de red, reintentos personalizados), como se mostró antes.

---

#### Mantén intactos los límites de arquitectura

- Mantén tu interfaz `UserRemote` (o similar) en commonMain
- Migra solo la implementación de Retrofit → Ktor
- Los repositorios dependen de la interfaz, por lo que el resto de la app no cambia

```kotlin
class UserRepository(private val remote: UserRemote) {
    suspend fun getUser(id: String) = remote.getUser(id)
}
```

---

#### Testear con MockEngine de Ktor

Puedes testear tu implementación remota sin tocar la red usando MockEngine.

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

#### Checklist de migración

- Añade dependencias de Ktor y kotlinx-serialization al módulo compartido
- Crea una factoría de HttpClient con motor por plataforma (Android: OkHttp o CIO; iOS: CIO o Darwin)
- Instala ContentNegotiation(Json), HttpTimeout, Logging
- Porta tus servicios de Retrofit a una implementación remota basada en Ktor
- Mantén repositorio/dominio sin cambios; intercambia solo la implementación remota en el DI
- Maneja errores de forma consistente; considera un wrapper Result/Either
- Escribe tests con MockEngine para casos de éxito y error

---

#### Decisiones frecuentes

- ¿CIO vs OkHttp en Android?
  - OkHttp: reutiliza interceptors/TLS/pinning existentes; depuración familiar
  - CIO: motor Ktor/JVM puro, menos dependencias externas
- Serialización: kotlinx-serialization es idiomática en KMP; migra tus DTOs a @Serializable
- Caché: impleméntala en la capa de repositorio; Ktor no impone políticas

---

#### Conclusión

Migrar de Retrofit/OkHttp a Ktor es un primer paso enfocado y de bajo riesgo hacia KMP. Con una arquitectura limpia, solo cambias las interfaces/implementaciones remotas mientras el resto de la app permanece estable. Empieza con una funcionalidad, valida con tests y expande con confianza en tu base de código.
