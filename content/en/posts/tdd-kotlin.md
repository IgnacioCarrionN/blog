---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Test-Driven Development (TDD) in Kotlin for Android"
date: 2025-01-20T08:00:00+01:00
description: "Test-Driven Development (TDD) in Kotlin for Android with real-world examples using JUnit, MockK, and Coroutines"
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

# Test-Driven Development (TDD) in Kotlin for Android

Test-Driven Development (TDD) is a software development practice that emphasizes writing tests before implementing functionality. It follows a **Red-Green-Refactor** cycle: first, you write a failing test (**Red**), then implement just enough code to make it pass (**Green**), and finally, refactor the code while keeping the test green (**Refactor**). In this post, we'll explore how to apply TDD in Kotlin for Android development using **JUnit, MockK, and Coroutines** with a real-world example.

![TDD Cycle](/images/kotlin/tdd-cycle.png)

## Why Use TDD in Android Development?

- **Better Code Quality**: Writing tests first ensures better design decisions and maintainability.
- **Faster Debugging**: Bugs are caught early before they become complex.
- **Refactoring Confidence**: Tests act as a safety net when modifying code.
- **Improved Productivity**: Although writing tests first might seem slower initially, it speeds up development in the long run.

## Setting Up the Test Environment

Before we begin, let's add the necessary dependencies to our **Gradle** file:

```kotlin
// Unit Testing
testImplementation("junit:junit:4.13.2")
testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.10.1")
testImplementation("io.mockk:mockk:1.13.16")
```

Now, let's create a real-world example demonstrating TDD.

---

## Real-World Example: Fetching Data in a UseCase

We'll implement a **UseCase** that fetches data from a **Repository** and runs it on the **IO Dispatcher**. We'll follow the TDD approach.

### Step 1: Write a Failing Test (Red)

First, let's define a test for our `FetchUserUseCase`. This use case fetches user details from a repository.

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
        useCase = FetchUserUseCase(repository, testDispatcher) // Inject test dispatcher
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

### Understanding Given-When-Then

- **Given** – Set up the initial conditions or dependencies required for the test.

  ```kotlin
  val expectedUser = User(id = 1, name = "John Doe")
  coEvery { repository.getUser(1) } returns expectedUser
  ```

    - This prepares a mock response for `repository.getUser(1)` so that it returns `expectedUser`.

- **When** – Execute the actual function or use case being tested.

  ```kotlin
  val result = useCase(1)
  ```

    - This calls the `FetchUserUseCase` with a user ID of `1`, triggering the behavior we want to test.

- **Then** – Verify that the expected outcome matches the actual outcome.

  ```kotlin
  assertEquals(expectedUser, result)
  coVerify { repository.getUser(1) }
  ```

    - This checks that the function returned the expected user and that the repository’s `getUser` method was called.

### Step 2: Implement Minimal Code to Pass the Test (Green)

Now, let's implement the **FetchUserUseCase** class.

```kotlin
import kotlinx.coroutines.CoroutineDispatcher
import kotlinx.coroutines.withContext

class FetchUserUseCase(
    private val repository: UserRepository,
    private val dispatcher: CoroutineDispatcher = Dispatchers.IO // Injected dispatcher
) {
    suspend operator fun invoke(userId: Int): User {
        return withContext(dispatcher) {
            repository.getUser(userId)
        }
    }
}
```

### Step 3: Refactor

Since our test is passing, we can clean up or improve our implementation if necessary. Here, the implementation is already clean, so no major refactoring is needed.

---

## Understanding Key Parts

### 1. **Mocking with MockK**

We use **MockK** to mock our repository:

```kotlin
coEvery { repository.getUser(1) } returns expectedUser
```

This simulates a function call returning a predefined value.

### 2. **Using Coroutines with Test Dispatchers**

We replace `Dispatchers.IO` with a **Test Dispatcher** to control coroutine execution.

### 3. **Verifying Function Calls**

We ensure that our repository function was called:

```kotlin
coVerify { repository.getUser(1) }
```

This confirms our code behaves as expected.

---

## Best Practices for TDD in Kotlin

- **Write Small, Focused Tests**: Each test should verify one thing.
- **Use Mocks Wisely**: Avoid over-mocking; only mock dependencies.
- **Prefer Deterministic Tests**: Avoid flaky or time-dependent tests.
- **Leverage Coroutines Test Utilities**: Use `StandardTestDispatcher` and `runTest`.
- **Keep Tests Fast**: Unit tests should run in milliseconds.

## Conclusion

TDD improves code quality and development efficiency. By writing tests first, we ensure reliable and maintainable code. In this post, we built a **UseCase that fetches data from a repository** while running on the **IO Dispatcher**, following TDD principles. With **MockK and Coroutines**, we created a robust testing setup.

Start applying TDD in your Kotlin projects today and see the benefits firsthand!

