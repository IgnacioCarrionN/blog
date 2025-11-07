---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Dominando Kotlin Coroutines: Dispatchers, Jobs y Concurrencia Estructurada"
date: 2025-11-07T08:00:00+01:00
description: "Guía práctica sobre Kotlin Coroutines, Dispatchers y Jobs con modelos mentales, recetas de código, trampas y consejos de testing."
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

### Dominando Kotlin Coroutines: Dispatchers, Jobs y Concurrencia Estructurada

Kotlin Coroutines ofrecen una forma ligera y estructurada de escribir código asíncrono y concurrente. Esta guía se centra en tres pilares que usarás a diario: Dispatchers (dónde corren las coroutines), Jobs (qué corre y cómo se supervisa), y la concurrencia estructurada (las reglas que mantienen tu código asíncrono bajo control).

Encontrarás modelos mentales, snippets ejecutables, gotchas y patrones listos para producción.

---

#### TL;DR

- Dispatcher: decide el hilo o pool donde ejecuta una coroutine (`Dispatchers.Default`, `IO`, `Main`, `Unconfined`).
- Job: un handle a una coroutine en ejecución; soporta lifecycle, cancelación, relaciones padre/hijo y concurrencia estructurada.
- Concurrencia estructurada: lanza coroutines en un scope; los hijos se cancelan cuando el scope se cancela; los errores no se filtran silenciosamente.
- Prefiere `withContext` para cambiar de hilo dentro de una función `suspend` y `launch` para fire-and-forget en un scope con lifecycle.
- Cancela siempre los scopes que crees; prefiere dispatchers provistos por el scope en lugar de globales.

---

#### Un modelo mental simple

- Coroutine: una tarea que puede suspenderse sin bloquear un hilo.
- Dispatcher: el contexto de ejecución (thread pool) que usa la tarea.
- Job: el controlador del ciclo de vida de la tarea.
- Scope: une un `CoroutineContext` (dispatcher + Job + extras) a un lifecycle.

```kotlin
class Repository(
    private val externalApi: ExternalApi,
    private val dispatcher: CoroutineDispatcher = Dispatchers.IO
) {
    suspend fun loadUser(id: String): User = withContext(dispatcher) {
        externalApi.fetchUser(id) // suspende sin bloquear un hilo
    }
}
```

---

#### Dispatchers a fondo

- `Dispatchers.Default`: trabajo CPU-bound (pool compartido ajustado a núcleos).
- `Dispatchers.IO`: IO bloqueante (archivos, sockets, JDBC). Usa un pool más grande.
- `Dispatchers.Main`: hilo de UI en Android/desktop; solo para trabajo ligero y rápido.
- `Dispatchers.Unconfined`: avanzado/raro; empieza donde se llama y puede reanudar en otro lugar.

Cambiar de dispatcher dentro de funciones `suspend`:

```kotlin
suspend fun resizeAndSave(bitmap: Bitmap, file: File) {
    val resized = withContext(Dispatchers.Default) { resize(bitmap) } // CPU-bound
    withContext(Dispatchers.IO) { file.outputStream().use { save(resized, it) } } // IO-bound
}
```

Dispatcher personalizado (paralelismo limitado):

```kotlin
val singleIO = Dispatchers.IO.limitedParallelism(1)
```

---

#### Jobs y ciclo de vida

Un `Job` representa una unidad de trabajo cancelable. Toda coroutine tiene uno. Los Jobs pueden formar jerarquías: cancelar un padre cancela a todos sus hijos.

```kotlin
val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)
val job: Job = scope.launch {
    // Trabajo
}

// Más tarde
job.cancel("no longer needed")
```

Tipos y utilidades clave:

- `Job`: job normal; un fallo cancela al padre salvo que haya supervisión adecuada.
- `SupervisorJob`: los hijos fallan de forma independiente; el padre no se cancela por fallos de un hijo.
- `coroutineContext[Job]`: accede al job actual.

---

#### Concurrencia estructurada en la práctica

- Lanza coroutines dentro de un scope que controlas (por ejemplo, en Android `viewModelScope`, `lifecycleScope`).
- Prefiere `coroutineScope {}` o `supervisorScope {}` dentro de funciones `suspend` para lanzar hijos.

```kotlin
suspend fun loadScreenData(repo: Repo): ScreenData = coroutineScope {
    val user = async { repo.user() }
    val posts = async { repo.posts() }
    ScreenData(user.await(), posts.await())
}
```

Si un hijo no debe cancelar a los demás, usa `supervisorScope`:

```kotlin
suspend fun loadPartial(repo: Repo): Partial = supervisorScope {
    val user = async { repo.user() }
    val comments = async { repo.comments() } // podría fallar
    Partial(user.await(), runCatching { comments.await() }.getOrNull())
}
```

---

#### Cancelación y timeouts

- La cancelación es cooperativa: revisa `isActive` o llama funciones `suspend` cancelables.
- Usa `withTimeout`/`withTimeoutOrNull` para límites superiores.

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

Cierra o limpia siempre en `finally`:

```kotlin
scope.launch(Dispatchers.IO) {
    val socket = openSocket()
    try {
        readLoop(socket)
    } finally {
        socket.close() // también se llama en cancelación
    }
}
```

---

#### Manejo de excepciones

- Las excepciones en un hijo cancelan al padre en scopes regulares.
- En un `SupervisorJob`/`supervisorScope`, los hermanos quedan aislados.
- Usa `CoroutineExceptionHandler` solo para excepciones de nivel superior no capturadas.

```kotlin
val handler = CoroutineExceptionHandler { _, e ->
    log("Unhandled: ${e.message}")
}

val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main + handler)
```

Para fallos esperados en hijos, captura localmente:

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

#### Elementos de contexto y `withContext`

`CoroutineContext` es un conjunto de elementos: `Dispatcher`, `Job`, `CoroutineName`, etc.

- Combina con `+`.
- Reemplaza un elemento añadiendo otro con la misma clave.

```kotlin
val scope = CoroutineScope(Dispatchers.Default + SupervisorJob() + CoroutineName("sync"))

suspend fun syncAll() = withContext(CoroutineName("sync-step")) {
    // Solo cambia el nombre; dispatcher/job heredados
}
```

---

#### Elegir el Dispatcher correcto

- CPU-bound: `Default` (o un pool dedicado limitado si necesitas aislamiento).
- IO-bound: `IO`.
- Actualizaciones de UI: `Main`.
- Evita `Unconfined` salvo que sepas por qué lo necesitas.

Anti-patrones:

- Crear tu propio `newFixedThreadPoolContext` sin una razón fuerte.
- Llamar `withContext(Dispatchers.Main)` para trabajo largo.
- Usar `GlobalScope` en código de app (difícil de cancelar y testear).

---

#### Patrones y recetas comunes

Fire-and-forget ligado al lifecycle:

```kotlin
// Android ViewModel
class MyViewModel(/* ... */) : ViewModel() {
    fun refresh() = viewModelScope.launch { repository.refresh() }
}
```

Pipeline:

```kotlin
suspend fun process(input: Input): Output = withContext(Dispatchers.Default) {
    val parsed = parse(input)
    val enriched = withContext(Dispatchers.IO) { enrich(parsed) }
    transform(enriched)
}
```

Backpressure con `limitedParallelism`:

```kotlin
suspend fun fetchAll(ids: List<String>): List<Item> = coroutineScope {
    val gate = Dispatchers.IO.limitedParallelism(8)
    ids.map { id -> async(gate) { fetch(id) } }.awaitAll()
}
```

---

#### Testing de coroutines

- Usa `kotlinx-coroutines-test`.
- Reemplaza dispatchers y controla el tiempo virtual.

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

#### Consejos de rendimiento y trampas

- Mantén las coroutines de grano grueso; demasiadas coroutines pequeñas añaden overhead.
- Evita cambios de contexto innecesarios.
- Prefiere scopes estructurados frente a `GlobalScope.launch` ad-hoc.
- Recuerda que `async` sin `await` fuga errores; usa `launch` para fire-and-forget.

---

#### Chuleta rápida

- Crear un scope: `CoroutineScope(Job() + Dispatchers.Default)`
- Cambiar de hilo: `withContext(Dispatchers.IO) { ... }`
- Lanzar tareas hijas: `coroutineScope { val a = async { ... }; a.await() }`
- Timeouts: `withTimeout(…)`, `withTimeoutOrNull(…)`
- Supervisión: `SupervisorJob`, `supervisorScope { … }`
- Manejador de excepciones: `CoroutineExceptionHandler { ctx, e -> … }`
