---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Kotlin Delegates"
date: 2024-12-23T08:00:00+01:00
description: "Kotlin avanzado - Delegates"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/koin-delegate.png
draft: false
tags: 
- kotlin
- android
- advanced
---

# ✨ Entendiendo los Kotlin Delegates: La magia detrás de código más limpio ✨

Los Kotlin **delegates** son una funcionalidad muy útil que te permite delegar el comportamiento de una propiedad o incluso una implementación de una interfaz a otro objecto. En lugar de escribir lógica repetitiva o manejar el estado directamente, puedes delegar esta responsabilidad a clases especializadas y reusables.

---

## Como funcionan los Delegates
Delegates en Kotlin funcionan usando la palabra reservada `by`, que redirecciona el comportamiento de una propiedad o interfaz al objeto delegado. Para propiedades, el objeto delegado provee una implementación personalizada de los métodos `get` y o `set`. Para la delegación de interfaces, la implementación de esa interfaz es delegada al objecto.

Esto es un ejemplo de una propiedad delegada:

```kotlin  
class StringDelegate {  
    private var value: String = ""  

    operator fun getValue(thisRef: Any?, property: kotlin.reflect.KProperty<*>): String {  
        println("Getting value for \${property.name}")  
        return value  
    }  

    operator fun setValue(thisRef: Any?, property: kotlin.reflect.KProperty<*>, newValue: String) {  
        println("Setting value for \${property.name} to \$newValue")  
        value = newValue  
    }  
}  

class Example {  
    var text: String by StringDelegate()  
}  

fun main() {  
    val example = Example()  
    example.text = "Hello, Kotlin!"  
    println(example.text)  
}  
```

### Output
```
Setting value for text to Hello, Kotlin!  
Getting value for text  
Hello, Kotlin!  
```
En este ejemplo:
- La clase `StringDelegate` define un comportamiento personalizado del acceso a la propiedad usando los operadores `getValue`y `setValue`.
- La propiedad `text` en la clase `Example`delega su comportamiento a la instancia de `StringDelegate`.

---

## Aplicaciones reales de Kotlin Delegates

### 1️⃣ Inyección de dependencias con Koin
En #Koin, puedes usar el delegado `by inject()` para inyectar dependencias directamente en tus clases. Esto elimina la necesidad de instanciar manualmente:

```kotlin  
class DelegatesFragment : Fragment() {  
    private val tracker: AnalyticsTracker by inject()  
}

inline fun <reified T : Any> KoinComponent.inject(
    qualifier: Qualifier? = null,
    mode: LazyThreadSafetyMode = KoinPlatformTools.defaultLazyMode(),
    noinline parameters: ParametersDefinition? = null,
): Lazy<T> =
    lazy(mode) { get<T>(qualifier, parameters) }
```

El delegado `by inject()` automáticamente resuelve la dependencia usando el contenedor de Koin. Esto abstrae la lógica, resultando en código más limpio y testeable.

---

### 2️⃣ Manejo de estados en Jetpack Compose
En Jetpack Compose, la función `remember` junto con `mutableStateOf` es un gran ejemplo de delegación. Esto ayuda a manejar el estado de forma eficiente dentro de los composables:

```kotlin  
@Composable  
fun Counter() {  
    var count by remember { mutableStateOf(0) }  

    Column {  
        Text("Count: $count")  
        Button(onClick = { count++ }) {  
            Text("Increment")  
        }  
    }  
}  
```

---

### 3️⃣ Inicialización Lazy
El delegado `lazy` es perfecto para propiedades que necesitan ser inicializadas solo cuando se acceden por primera vez:

```kotlin  
val greeting: String by lazy {  
    println("Initializing...")  
    "Hello, Kotlin!"  
}  

fun main() {  
    println(greeting) // Initializes here  
    println(greeting) // Uses cached value  
}  
```

### Output
```
Initializing...  
Hello, Kotlin!  
Hello, Kotlin!  
```

---

### 4️⃣ Delegación de interfaces
Kotlin permite delegar la implementación de una interfaz a otro objeto.

```kotlin  
interface Logger {  
    fun log(message: String)  
}  

class ConsoleLogger : Logger {  
    override fun log(message: String) {  
        println("Log: $message")  
    }  
}  

class FileLogger : Logger {  
    override fun log(message: String) {  
        println("Writing log to file: $message")  
    }  
}  

class Application(logger: Logger) : Logger by logger  

fun main() {  
    val consoleApp = Application(ConsoleLogger())  
    consoleApp.log("Starting console application")  

    val fileApp = Application(FileLogger())  
    fileApp.log("Starting file application")  
}  
```

### Output
```
Log: Starting console application  
Writing log to file: Starting file application  
```

Esto es lo que está pasando:
- La clase `Application` no tiene que implementar los métodos `Logger` de forma directa.
- En su lugar, delega la implementación de `Logger` al objeto pasado por constructor usando `by`.
- Esto hace más sencillo cambiar las implementaciones sin necesidad de cambiar la clase `Application`.

---

## Por qué usar Kotlin Delegates?
Los Delegates encapsulan la lógica que de otra manera cargarían y desordenarían tus clases. Ayudan a:
- Simplificar el código al reutilizar lógica, por ejemplo con la inicialización `lazy`.
- Abstraen patrones repetitivos, por ejemplo con la inyección de dependencias con #koin.
- Mejoran el manejo de los estados con la función `mutableStateOf` de Compose.
- Provee implementaciones modulares y reutilizables de interfaces.

---

## Conclusion
El mecanismo de los delegados en Kotlin es un ejemplo de como este lenguaje combina simplicidad con funcionalidad. Los delegados están en todas partes en el desarrollo de Kotlin. En qué otros casos los utilizas en tus proyectos?
