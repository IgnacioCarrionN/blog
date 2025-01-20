---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Test-Driven Development (TDD) en Kotlin para Android"
date: 2025-01-20T08:00:00+01:00
description: "Test-Driven Development (TDD) en Kotlin para Android con ejemplos reales usando JUnit, MockK y Coroutines"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/tdd-cycle.png
draft: false
tags: 
- kotlin
- architecture
- TDD
- testing
---

# Test-Driven Development (TDD) en Kotlin para Android

El **Test-Driven Development (TDD)** es una práctica de desarrollo de software que enfatiza escribir pruebas antes de implementar la funcionalidad. Sigue un ciclo **Rojo-Verde-Refactorización**: primero, escribes una prueba que falla (**Rojo**), luego implementas el código mínimo para que pase (**Verde**), y finalmente, refactorizas el código manteniendo la prueba en verde (**Refactorización**). En esta publicación, exploraremos cómo aplicar TDD en Kotlin para el desarrollo de Android usando **JUnit, MockK y Coroutines**, con un ejemplo del mundo real.

![Ciclo TDD](/images/kotlin/tdd-cycle.png)

## ¿Por qué usar Test-Driven Development en el desarrollo de Android?

- **Mejor calidad del código**: Escribir pruebas primero garantiza mejores decisiones de diseño y mantenibilidad.
- **Depuración más rápida**: Los errores se detectan temprano antes de volverse complejos.
- **Confianza al refactorizar**: Las pruebas actúan como una red de seguridad al modificar código.
- **Mayor productividad**: Aunque escribir pruebas primero puede parecer más lento al principio, acelera el desarrollo a largo plazo.

## Configuración del entorno de prueba

Antes de comenzar, agreguemos las dependencias necesarias a nuestro archivo **Gradle**:

```kotlin
// Pruebas unitarias
testImplementation("junit:junit:4.13.2")
testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.10.1")
testImplementation("io.mockk:mockk:1.13.16")
```

Ahora, crearemos un ejemplo del mundo real para demostrar Test-Driven Development.

---

## Ejemplo del mundo real: Obtener datos en un UseCase

Implementaremos un **UseCase** que obtiene datos de un **Repositorio** y los ejecuta en el **Dispatcher IO**. Seguiremos el enfoque Test-Driven Development.

### Paso 1: Escribir una prueba que falle (Rojo)

Primero, definamos una prueba para nuestro `FetchUserUseCase`. Este caso de uso obtiene los detalles de un usuario desde un repositorio.

```kotlin
import io.mockk.*
import kotlinx.coroutines.CoroutineDispatcher
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.*
import kotlinx.coroutines.runBlocking
import org.junit.Before
import org.junit.Test
import kotlin.test.assertEquals

@ExperimentalCoroutinesApi
class FetchUserUseCaseTest {
    private val repository: UserRepository = mockk()
    private lateinit var useCase: FetchUserUseCase
    private val testDispatcher = StandardTestDispatcher()

    @Before
    fun setup() {
        useCase = FetchUserUseCase(repository, testDispatcher) // Inyectar dispatcher de prueba
    }

    @Test
    fun `fetch user returns expected user`() = runTest {
        // Given
        val expectedUser = User(id = 1, name = "John Doe")
        coEvery { repository.getUser(1) } returns expectedUser

        // When
        val result = useCase(1)

        // Then
        assertEquals(expectedUser, result)
        coVerify { repository.getUser(1) }
    }
}
```

### Entendiendo Given-When-Then

- **Given (Dado)** – Configurar las condiciones o dependencias necesarias para la prueba.

  ```kotlin
  val expectedUser = User(id = 1, name = "John Doe")
  coEvery { repository.getUser(1) } returns expectedUser
  ```

    - Esto prepara una respuesta simulada para `repository.getUser(1)`, de modo que devuelva `expectedUser`.

- **When (Cuando)** – Ejecutar la función o caso de uso que se está probando.

  ```kotlin
  val result = useCase(1)
  ```

    - Esto llama a `FetchUserUseCase` con un ID de usuario `1`, activando el comportamiento que queremos probar.

- **Then (Entonces)** – Verificar que el resultado esperado coincida con el resultado real.

  ```kotlin
  assertEquals(expectedUser, result)
  coVerify { repository.getUser(1) }
  ```

    - Esto comprueba que la función devolvió el usuario esperado y que el método `getUser` del repositorio fue llamado.

### Paso 2: Implementar el código mínimo para que pase la prueba (Verde)

Ahora, implementemos la clase **FetchUserUseCase**.

```kotlin
import kotlinx.coroutines.CoroutineDispatcher
import kotlinx.coroutines.withContext

class FetchUserUseCase(
    private val repository: UserRepository,
    private val dispatcher: CoroutineDispatcher = Dispatchers.IO // Dispatcher inyectado
) {
    suspend operator fun invoke(userId: Int): User {
        return withContext(dispatcher) {
            repository.getUser(userId)
        }
    }
}
```

### Paso 3: Refactorizar

Dado que nuestra prueba está pasando, podemos limpiar o mejorar nuestra implementación si es necesario. Aquí, la implementación ya es clara, por lo que no se requieren grandes refactorizaciones.

---

## Entendiendo las partes clave

### 1. **Simulación con MockK**

Usamos **MockK** para simular nuestro repositorio:

```kotlin
coEvery { repository.getUser(1) } returns expectedUser
```

Esto simula una llamada a una función que devuelve un valor predefinido.

### 2. **Uso de Coroutines con Test Dispatchers**

Reemplazamos `Dispatchers.IO` con un **Test Dispatcher** para controlar la ejecución de las corrutinas.

### 3. **Verificación de llamadas a funciones**

Nos aseguramos de que la función del repositorio haya sido llamada:

```kotlin
coVerify { repository.getUser(1) }
```

Esto confirma que nuestro código se comporta como se espera.

---

## Mejores prácticas para Test-Driven Development en Kotlin

- **Escribir pruebas pequeñas y enfocadas**: Cada prueba debe verificar una sola cosa.
- **Usar mocks con prudencia**: Evita el exceso de mocks; solo simula dependencias reales.
- **Preferir pruebas deterministas**: Evita pruebas inestables o dependientes del tiempo.
- **Aprovechar las utilidades de prueba de Coroutines**: Usa `StandardTestDispatcher` y `runTest`.
- **Mantener pruebas rápidas**: Las pruebas unitarias deben ejecutarse en milisegundos.

## Conclusión

Test-Driven Development mejora la calidad del código y la eficiencia en el desarrollo. Al escribir pruebas primero, garantizamos código confiable y mantenible. En esta publicación, construimos un **UseCase que obtiene datos de un repositorio** ejecutándolo en el **Dispatcher IO**, siguiendo los principios de Test-Driven Development. Con **MockK y Coroutines**, creamos una configuración de pruebas robusta.

¡Comienza a aplicar Test-Driven Development en tus proyectos de Kotlin hoy mismo y experimenta los beneficios de primera mano!

---

### 🚀 ¿Qué sigue?

¿Te gustaría que ampliemos este contenido con pruebas de **ViewModel** o **pruebas de UI con Jetpack Compose**? ¡Déjamelo saber en los comentarios!

