---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Mocks, Fakes, and More"
date: 2025-02-06T08:00:00+01:00
description: "Mocks, Fakes, and More: Understanding Test Doubles in Kotlin"
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

# Mocks, Fakes, and More: Understanding Test Doubles in Kotlin

When writing tests in Kotlin, especially for Android development, we often need to replace real dependencies with test doubles. However, not all test doubles are the same—terms like mocks, fakes, stubs, spies, and dummies often come up. In this post, we’ll break down their differences with Kotlin examples using only plain Kotlin (no third-party libraries).

---

## 1. Understanding Test Doubles

Test doubles are objects that stand in for real dependencies in tests. They help us isolate the system under test (SUT) and make tests more reliable. Here are the main types:

- **Dummy** – A placeholder object that is never actually used.
- **Stub** – Provides predefined responses but doesn’t contain logic.
- **Fake** – A lightweight implementation with in-memory logic.
- **Mock** – A test double that verifies interactions.
- **Spy** – Wraps a real object while allowing selective behavior modification.

---

## 2. Dummy Objects

A **dummy** is an object that exists only to satisfy a method signature but is never actually used.

### Example:
```kotlin
interface EmailSender {
    fun sendEmail(email: String, message: String)
}

class UserService(private val emailSender: EmailSender) {
    fun registerUser(user: User) {
        // User is registered, but we don't actually send an email
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

A **stub** returns predefined responses to method calls but doesn’t track interactions.

### Example:
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

A **fake** is a simplified but functional version of a real class, often using in-memory storage.

### Example:
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

A **mock** is a test double that verifies interactions. Without a mocking framework, we must manually track calls.

### Example:
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

A **spy** wraps a real object while allowing selective behavior modification. Without a library, we must extend the real class and override specific behavior.

### Example:
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
        return super.add(a, b) // Calls real implementation
    }
}
```

---

## 7. Using MockK for Mocks and Spies

While manually creating mocks and spies is possible, using a library like **MockK** simplifies the process.

### Example using MockK:
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

MockK provides powerful features like automatic spies, relaxed mocks, and argument capturing, making testing easier and more maintainable.

---

## Conclusion

Understanding test doubles helps you write better tests by isolating dependencies. Use:

✅ **Dummies** when an argument is required but unused.
✅ **Stubs** for returning predefined values.
✅ **Fakes** for lightweight implementations.
✅ **Mocks** when verifying interactions.
✅ **Spies** when you need partial mocking.
✅ **MockK** for easier and more powerful mocking.

By choosing the right type, you can make your tests more reliable and maintainable.
