---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Kotlin 2.4 Rich Errors: Qué son y cómo prepararte"
date: 2025-08-29T08:00:00+01:00
description: "Una visión general de los Rich Errors en Kotlin 2.4: objetivos, modelo mental, ejemplos, interop con excepciones y Result, y cómo preparar tu base de código desde hoy."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/rich-errors.png
draft: false
tags: 
- kotlin
- manejo-de-errores
- excepciones
- result
- kotlin-2-4
---

### Kotlin 2.4 Rich Errors: Qué son y cómo prepararte

Kotlin 2.4 introduce “Rich errors” (errores enriquecidos): una forma más expresiva y estructurada de representar y propagar fallos. El objetivo es claro: hacer que los flujos de error sean visibles y componibles en toda tu base de código y en múltiples plataformas, sin perder la ergonomía de Kotlin ni su gran historia de interoperabilidad.

Este artículo explica los problemas que Rich errors resuelven, cómo se relacionan con las excepciones y `Result` actuales, qué implican en cuanto a modelo mental e interop, y cómo preparar tu base de código para adoptarlos sin fricciones.

---

#### ¿Por qué Rich Errors?

El manejo de errores basado en excepciones es potente, pero tiene inconvenientes:

- Flujo de control oculto: las excepciones no aparecen en las firmas de las funciones
- Mezcla de preocupaciones: los tipos lanzados no siempre son explícitos o estructurados
- Fricción en la composición: componer resultados entre capas suele requerir try/catch
- Matices multiplataforma: mapear excepciones entre plataformas puede ser desigual

Kotlin ya ofrece `Result<T>` y utilidades funcionales que cubren parte de esto. Rich errors amplían la idea: hacer explícito, tipado y componible el canal de fallo, manteniendo una excelente interop con el código existente.

---

#### Modelo mental

A alto nivel, Rich errors aspira a:

- Hacer explícitos y de primera clase los tipos de error (p. ej., jerarquías de error de dominio)
- Componer limpiamente a través de límites suspend/async
- Interoperar con excepciones (p. ej., mapear de/para excepciones cuando sea necesario)
- Preservar tipado estructurado en módulos multiplataforma

Piensa en ello como llevar la claridad de los tipos de error sellados junto con la ergonomía de las herramientas estándar de Kotlin al lenguaje y su ecosistema. Los siguientes patrones ilustran cómo se ve en la práctica.

---

#### Sintaxis real: Errores con tipo unión

Kotlin 2.4 introduce retornos con tipo unión. Una función puede declarar su tipo de éxito y el conjunto de variantes de error que puede producir, todo en el tipo de retorno:

```text
// API básica con retorno unión
fun obtenerUsuario(id: UserId): Usuario | NoEncontrado | FallaDeRed

// API suspend con varios errores de dominio
suspend fun subirAvatar(archivo: Imagen): Url | FormatoNoSoportado | CuotaExcedida | FallaDeRed

// Propagación de uniones entre capas
fun cargarPerfil(id: UserId): Perfil | NoEncontrado | FallaDeRed | ErrorDeDecodificacion
```

En el sitio de llamada, manejas la unión con when y smart‑casts:

```kotlin
when (val r = obtenerUsuario(idActual)) {
    is Usuario -> mostrarPerfil(r)
    is NoEncontrado -> mostrarUsuarioInexistente()
    is FallaDeRed -> mostrarMensajeSinConexion(r)
}
```

También puedes mapear una unión a estado de UI (u otra unión), preservando exhaustividad:

```text
fun aUi(resultado: Token | CredencialesInvalidas | FallaDeRed): Ui = when (resultado) {
    is Token -> Ui.Autenticado(resultado)
    is CredencialesInvalidas -> Ui.Error("Credenciales inválidas")
    is FallaDeRed -> Ui.Error("Revisa tu conexión")
}
```

Nota: Los nombres de estos ejemplos son ilustrativos; la idea clave es que el canal de error es de primera clase en el tipo de la función, habilitando mejor tooling, chequeo de exhaustividad y composición.

---

#### Comparativa: Result vs Rich Errors

Aquí tienes dos comparativas pequeñas y realistas que muestran el mismo flujo con Result de Kotlin hoy y con Rich errors.

- Ejemplo A — Leer y parsear configuración

```kotlin
// Result hoy
fun cargarConfig(ruta: Path): Result<Config> =
    runCatching { fs.readBytes(ruta) }
        .mapCatching { bytes -> parsearConfig(bytes) }
        .recover { e ->
            if (e is NoSuchFileException) configPorDefecto() else throw e
        }

fun usarConfig(ruta: Path) {
    cargarConfig(ruta)
        .onSuccess { cfg -> iniciarApp(cfg) }
        .onFailure { e ->
            when (e) {
                is NoSuchFileException -> mostrarAvisoConfigInexistente()
                is ConfigFormatException -> mostrarErrorParseoConfig(e)
                else -> mostrarErrorGenerico(e)
            }
        }
}
```

```text
// Rich errors
// Variantes de error (objetos/clases de dominio)
data object FallaIO
data class FallaParseo(val detalle: String)

fun cargarConfig(ruta: Path): Config | FallaIO | FallaParseo

fun usarConfigRico(ruta: Path) {
    when (val r = cargarConfig(ruta)) {
        is Config -> iniciarApp(r)
        is FallaIO -> mostrarAvisoConfigInexistente()
        is FallaParseo -> mostrarErrorParseoConfigMensaje(r.detalle)
    }
}
```

- Ejemplo B — Componer dos llamadas (sesión -> panel)

```kotlin
// Result hoy
suspend fun obtenerSesionResult(usuario: UserId): Result<Sesion>
suspend fun obtenerPanelResult(sesion: Sesion): Result<Panel>

inline fun <T, R> Result<T>.flatMap(transform: (T) -> Result<R>): Result<R> =
    fold(onSuccess = transform, onFailure = { Result.failure(it) })

suspend fun cargarInicioResult(usuario: UserId): Result<Panel> =
    obtenerSesionResult(usuario).flatMap { s -> obtenerPanelResult(s) }
```

```text
// Rich errors
// Variantes de error de dominio
data object FallaAutenticacion
data object FallaDeRed
data object FallaBackend

suspend fun obtenerSesion(usuario: UserId): Sesion | FallaAutenticacion | FallaDeRed
suspend fun obtenerPanel(sesion: Sesion): Panel | FallaDeRed | FallaBackend

suspend fun cargarInicio(usuario: UserId): Panel | FallaAutenticacion | FallaDeRed | FallaBackend =
    when (val s = obtenerSesion(usuario)) {
        is Sesion -> when (val p = obtenerPanel(s)) {
            is Panel -> p
            is FallaDeRed -> p
            is FallaBackend -> p
        }
        is FallaAutenticacion -> s
        is FallaDeRed -> s
    }
```

---

#### Interop: Excepciones, Result, Coroutines y Multiplatform

- Excepciones: Las librerías y APIs de plataforma que lanzan siguen funcionando. Rich errors provee mapeo en ambas direcciones, para trabajar con errores tipados internamente y convertir a excepciones en los bordes (o viceversa).
- Result: Es sencillo adaptar entre `Result<T>` y una representación de error tipado más rica al cruzar capas.
- Coroutines: La cancelación es especial y no debe tragarse; trata `CancellationException` como una señal de flujo de control, no un error.
- Multiplatform: Mantén los dominios de error en `commonMain` como interfaces selladas; proporciona mapeos específicos de plataforma cuando te enfrentes a excepciones de plataforma.

---

#### Estrategia de migración que puedes empezar hoy

- Modela jerarquías de error pragmáticas donde aporten valor (sin sobre‑modelar)
- Lanza excepciones solo en los bordes; internamente prefiere canales de error tipados
- Añade mapeadores ligeros para convertir excepciones <-> errores tipados

---

#### Trampas y gotchas

- Sobre-modelado: Mantén los tipos de error pragmáticos; no explotes la jerarquía
- Claridad en los bordes: Decide dónde conviertes entre excepciones y errores tipados
- Cancelación: Vuelve a lanzar siempre `CancellationException`
- Logging: Centraliza el logging; evita el doble log en múltiples capas

---

#### Conclusiones

- Rich errors buscan hacer los fallos explícitos, tipados y componibles
- Hoy puedes obtener el 80% del beneficio usando tipos de error sellados + Result/Outcome
- Modela errores en `commonMain` para KMP y convierte en los bordes de plataforma
