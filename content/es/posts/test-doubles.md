---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Mocks, Fakes y Más"
date: 2025-02-06T08:00:00+01:00
description: "Mocks, Fakes y Más: Entendiendo los Test Doubles en Kotlin"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/mock.png
draft: false
tags:
- kotlin
- testing
- mock
- tdd
---

# Mocks, Fakes y Más: Entendiendo los Test Doubles en Kotlin

Al escribir tests en Kotlin, especialmente para el desarrollo de Android, a menudo necesitamos reemplazar dependencias reales con test doubles. Sin embargo, no todos los test doubles son iguales: términos como mocks, fakes, stubs, spies y dummies aparecen con frecuencia. En esta publicación, desglosaremos sus diferencias con ejemplos en Kotlin utilizando solo Kotlin puro (sin bibliotecas de terceros).

---

## 1. Entendiendo los Test Doubles

Los test doubles son objetos que sustituyen dependencias reales en las tests. Ayudan a aislar el sistema bajo prueba (SUT) y hacen que las tests sean más confiables. Aquí están los principales tipos:

- **Dummy** – Un objeto de relleno que se pasa como argumento pero no se utiliza en la ejecución del test.
- **Stub** – Proporciona respuestas predefinidas pero no contiene lógica.
- **Fake** – Una implementación liviana con lógica en memoria.
- **Mock** – Un test double que verifica interacciones.
- **Spy** – Envuelve un objeto real permitiendo la modificación selectiva de su comportamiento.

---

## 2. Objetos Dummy

Un **dummy** es un objeto que solo existe para satisfacer la firma de un método, pero nunca se usa realmente.

### Ejemplo:
```kotlin
interface EmailSender {
    fun sendEmail(email: String, message: String)
}

class UserService(private val emailSender: EmailSender) {
    fun registerUser(user: User) {
        // The user registers, but we don't actually send an email
    }
}

data class User(val name: String, val email: String)

// Test
fun testRegisterUser() {
    val dummyEmailSender = object : EmailSender {
        override fun sendEmail(email: String, message: String) {
            // This will never be called in the test
        }
    }
    val userService = UserService(dummyEmailSender)

    userService.registerUser(User("John Doe", "john@example.com"))
}
```

---

## 3. Stubs

Un **stub** devuelve respuestas predefinidas a las llamadas a métodos, pero no rastrea interacciones.

### Ejemplo:
```kotlin
interface UserRepository {
    fun getUser(id: Int): User?
}

class StubUserRepository : UserRepository {
    override fun getUser(id: Int): User? {
        return if (id == 1) User("John Doe", "john@example.com") else null
    }
}

// Test
fun testGetUser() {
    val stubRepo = StubUserRepository()
    val user = stubRepo.getUser(1)

    assert(user?.name == "John Doe")
}
```

---

## 4. Fakes

Un **fake** es una versión simplificada pero funcional de una clase real, a menudo utilizando almacenamiento en memoria.

### Ejemplo:
```kotlin
class FakeUserRepository : UserRepository {
    private val users = mutableMapOf<Int, User>()

    override fun getUser(id: Int): User? = users[id]

    fun addUser(id: Int, user: User) {
        users[id] = user
    }
}

// Test
fun testFakeUserRepository() {
    val fakeRepo = FakeUserRepository()
    fakeRepo.addUser(1, User("Jane Doe", "jane@example.com"))

    assert(fakeRepo.getUser(1)?.name == "Jane Doe")
}
```

---

## 5. Mocks

Un **mock** es un test double que verifica interacciones. Sin un framework de mocking, debemos rastrear manualmente las llamadas.

### Ejemplo:
```kotlin
class MockEmailSender : EmailSender {
    var wasSendEmailCalled = false
    var sentTo: String? = null
    var sentMessage: String? = null

    override fun sendEmail(email: String, message: String) {
        wasSendEmailCalled = true
        sentTo = email
        sentMessage = message
    }
}

// Test
fun testSendWelcomeEmail() {
    val mockEmailSender = MockEmailSender()
    val service = NotificationService(mockEmailSender)

    service.sendWelcomeEmail(User("test@example.com", "test@example.com"))

    assert(mockEmailSender.wasSendEmailCalled)
    assert(mockEmailSender.sentTo == "test@example.com")
    assert(mockEmailSender.sentMessage == "Welcome!")
}

class NotificationService(private val emailSender: EmailSender) {
    fun sendWelcomeEmail(user: User) {
        emailSender.sendEmail(user.email, "Welcome!")
    }
}
```

---

## 6. Spies

Un **spy** envuelve un objeto real mientras permite la modificación selectiva de su comportamiento. Sin una biblioteca, debemos extender la clase real y sobrescribir comportamientos específicos.

### Ejemplo:
```kotlin
open class MathService {
    open fun add(a: Int, b: Int) = a + b
}

class SpyMathService : MathService() {
    var wasAddCalled = false
    var lastA: Int? = null
    var lastB: Int? = null

    override fun add(a: Int, b: Int): Int {
        wasAddCalled = true
        lastA = a
        lastB = b
        return super.add(a, b) // Calls the real implementation
    }
}
```

---

## 7. Uso de MockK para Mocks y Spies

Si bien es posible crear mocks y spies manualmente, usar una biblioteca como **MockK** simplifica el proceso.

### Ejemplo usando MockK:
```kotlin
import io.mockk.*

fun testMockKExample() {
    val mockEmailSender = mockk<EmailSender>()
    every { mockEmailSender.sendEmail(any(), any()) } just Runs

    val service = NotificationService(mockEmailSender)
    service.sendWelcomeEmail(User("test@example.com", "test@example.com"))

    verify { mockEmailSender.sendEmail("test@example.com", "Welcome!") }
}
```

MockK proporciona características avanzadas como spies automáticos, mocks relajados y captura de argumentos, lo que facilita la escritura de tests mantenibles.

---

## Conclusión

Comprender los test doubles te ayuda a escribir mejores tests al aislar dependencias. Usa:

✅ **Dummies** cuando se requiere un argumento pero no se usa.
✅ **Stubs** para devolver valores predefinidos.
✅ **Fakes** para implementaciones livianas.
✅ **Mocks** para verificar interacciones.
✅ **Spies** cuando necesitas mockeo parcial.
✅ **MockK** para un mockeo más fácil y potente.

Al elegir el tipo correcto, puedes hacer que tus tests sean más confiables y mantenibles.
