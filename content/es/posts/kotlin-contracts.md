---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Kotlin contracts"
date: 2024-12-20T08:00:00+01:00
description: "Kotlin avanzado - Contracts"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/kotlin-contract.png
draft: false
tags: 
- kotlin
- android
- advanced
---

### **Kotlin Avanzado - Contracts: Cómo volver al compilador de Kotlin más inteligente**

Kotlin nunca deja de impresionarme con sus funcionalidades. Una función avanzada pero poco utilizada en el arsenal de Kotlin son los **Contracts**. Los contratos te permiten guiar al compilador de Kotlin para que tome mejores decisiones acerca de tu código, resultando en mejor seguridad ante nulos, mejor rendimiento o incluso menores errores en tiempo de ejecución.

---

### **Qué son los contratos de Kotlin?**

Los contratos de Kotlin te permiten definir **reglas** acerca de como se comporta tu código, ayudando al compilador a hacer un análisis estático más avanzado. Los contratos habilitan funcionalidades como smart-casts y comprobaciones teniendo en cuenta el contexto, superando las capacidades básicas de Kotlin.

---

### **Por qué usar contratos?**

1. **Mejora la seguridad ante nulos**: Elimina las comprobaciones de nulos redundantes ayudando al compilador a saber cuando algo está garantizado que no sea nulo.
2. **Smart-casts optimizados**: Hace que el compilador conozca el tipo de las variables en casos específicos.
3. **Reduce la repeticón de código**: Escribe código más limpio e intuitivo delegando las comprobaciones repetitivas al compilador.

---

### **Ejemplos de contratos en Kotlin**

#### 1. **Simplificar las comprobaciones de nuloss**

Vamos a crear una función para validar valores no nulos:

```kotlin
@OptIn(ExperimentalContracts::class)
inline fun <T> requireNotNull(value: T?, message: String): T {
    contract {
        returns() implies (value != null)
    }
    if (value == null) {
        throw IllegalArgumentException(message)
    }
    return value
}

fun processName(name: String?) {
    val nonNullName = requireNotNull(name, "Name cannot be null")
    // No need for additional null checks; compiler knows 'nonNullName' is not null!
    println("Processing name: $nonNullName")
}

fun main() {
    processName("John")  // Works fine
    // processName(null) // Throws an IllegalArgumentException
}
```

##### **Cómo los contratos nos ayudan aquí?**

- La parte del contrato **`returns() implies (value != null)`** le dice al compilador:
  > Si la función retorna de forma satisfactoria, entonces `value` está garantizado que no es nulo.
- Esto habilita smart-casts, de manera que no tienes que volver a comprobar si es nulo manualmente una vez llamada esta función.

Algo muy similar se hace en las funciones `require` y `requireNotNull` de la librería estandar de Kotlin:

```kotlin
/**
 * Throws an [IllegalArgumentException] with the result of calling [lazyMessage] if the [value] is false.
 *
 * @sample samples.misc.Preconditions.failRequireWithLazyMessage
 */
@kotlin.internal.InlineOnly
public inline fun require(value: Boolean, lazyMessage: () -> Any): Unit {
    contract {
        returns() implies value
    }
    if (!value) {
        val message = lazyMessage()
        throw IllegalArgumentException(message.toString())
    }
}

/**
 * Throws an [IllegalArgumentException] with the result of calling [lazyMessage] if the [value] is null. Otherwise
 * returns the not null value.
 *
 * @sample samples.misc.Preconditions.failRequireNotNullWithLazyMessage
 */
@kotlin.internal.InlineOnly
public inline fun <T : Any> requireNotNull(value: T?, lazyMessage: () -> Any): T {
    contract {
        returns() implies (value != null)
    }

    if (value == null) {
        val message = lazyMessage()
        throw IllegalArgumentException(message.toString())
    } else {
        return value
    }
}
```

---

#### 2. **Afirmaciones personalizadas**

Aquí se ve como los contratos pueden ser usados para definir afirmaciones personalizadas:

```kotlin
@OptIn(ExperimentalContracts::class)
fun assertValidState(condition: Boolean, message: String) {
    contract {
        returns() implies condition
    }
    if (!condition) {
        throw IllegalStateException(message)
    }
}

fun performOperation(state: Boolean) {
    val state: Any? = "Hello"
    assertValidState(state is String, "Is String")
    // Here the compiler knows that the state val is of type String so no need to other cast checks
    println("String length: ${assertion.length}")
}

fun main() {
    performOperation(true)   // Prints success
    // performOperation(false) // Throws IllegalStateException
}
```

---

#### 3. **Smart-Casts con condiciones personalizadas**

Vamos a crear una funcionalidad custom que comprueba si una valor coincide con un tipo específico. Esto demostrará como los contratos pueden ayudar a mejorar las comprobaciones:

```kotlin
@OptIn(ExperimentalContracts::class)
inline fun <reified T> isOfType(value: Any?): Boolean {
    contract {
        returns(true) implies (value is T)
    }
    return value is T
}

fun main() {
    val input: Any? = "Hello, Kotlin!"

    if (isOfType<String>(input)) {
        println("String length: ${input.length}")
    }

    val inputInt: Any? = 10
    if (isOfType<Int>(inputInt)) {
        println("The value is an integer ${input.toUInt()}")
    }
}
```

Con esta implementación, el compilador sabe que dentro del bloque `if`, `input` es un `String`, gracias al contrato definido en `isOfType`. The la misma manera, el compilador sabe que `inputInt` es de tipo `Int` y no hace falta comprobar el tipo de nuevo.

---

#### 4. **Optimizando el control del flujo**

Los contratos pueden simplificar el control del flujo habilitando al compilador para entender las invariantes o condiciones. Por ejemplo:

```kotlin
inline fun isNotEmpty(list: List<*>?): Boolean {
    contract {
        returns(true) implies (list != null && list.isNotEmpty())
    }
    return list != null && list.isNotEmpty()
}

fun processItems(items: List<String>?) {
    if (isNotEmpty(items)) {
        // Compiler knows items is non-null and not empty
        println("Processing ${items.size} items")
    } else {
        println("No items to process")
    }
}

fun main() {
    processItems(listOf("A", "B", "C"))
    processItems(null)
    processItems(emptyList())
}
```

##### **Salida**

```
Processing 3 items
No items to process
No items to process
```

---

### **Cuando usar contratos**

Los contratos son ideales para:

1. **Desarrollo de librerías**: Proteger APIs públicas forzando condiciones pre existentes.
2. **DSLs y Frameworks**: Simplificando la comprobación de tipos y validación de estados en DSLs de Kotlin.
3. **Optimizaciones en tiempo de ejecución**: Reduce las comprobaciones en tiempo de ejecución al permitir al compilador inferir las condiciones en tiempo de compilación.

---

### **Conclusion**

Los contratos de Kotlin son una gema oculta que pueden perfeccionar tu código mejorando la seguridad, reduciendo la repetición de código, y permitiendo un análisis por parte del compilador más inteligente. Tanto si estás creando librerías, escribiendo complejos DSLs, o simplemente optimizando código del día a día, los contratos proveen una herramienta muy poderosa para guiar al compilador de Kotlin y asegurando un código correcto.

Tener en cuenta que los contratos están anotados como funcionalidad experimental pero están implementados en Kotlin desde la versión 1.3 y se usan extensamente en la librería estandar de Kotlin así que son lo suficiente estables como para utilizarlos.