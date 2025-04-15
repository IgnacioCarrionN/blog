---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Leveraging Sealed Classes and Interfaces for Better Domain Modeling"
date: 2025-04-15T08:00:00+01:00
description: "How to use Kotlin's sealed classes and interfaces to create more robust and type-safe domain models."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/sealed-classes.png
draft: false
tags: 
- kotlin
- architecture
- domain modeling
- type safety
---

### Leveraging Sealed Classes and Interfaces for Better Domain Modeling

Domain modeling is a crucial aspect of software development, representing the core business concepts and rules in your application. Kotlin provides powerful language features that can help create more expressive, type-safe, and maintainable domain models. Among these features, sealed classes and interfaces stand out as particularly valuable tools. This blog post explores how to leverage these Kotlin features to build better domain models.

---

#### Understanding Sealed Classes and Interfaces

Sealed classes and interfaces in Kotlin are special constructs that restrict the hierarchy of a type. When a class or interface is marked as `sealed`, all of its subclasses must be defined within the same file (or, since Kotlin 1.5, within the same module as direct subclasses).

```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val message: String, val cause: Exception? = null) : Result<Nothing>()
    object Loading : Result<Nothing>()
}

fun main() {
    val result: Result<String> = Result.Success("Data loaded successfully")

    val message = when (result) {
        is Result.Success -> "Success: ${result.data}"
        is Result.Error -> "Error: ${result.message}"
        is Result.Loading -> "Loading..."
    }

    println(message) // Outputs: Success: Data loaded successfully
}
```

The key benefits of sealed classes include:

1. **Exhaustive when expressions**: The compiler ensures that all possible subclasses are handled in a `when` expression
2. **Restricted hierarchy**: All subclasses must be known at compile time
3. **Type safety**: The compiler can verify that all cases are handled
4. **Expressiveness**: They clearly communicate that a type has a limited set of subtypes

---

#### Domain Modeling with Sealed Classes

Let's explore how sealed classes can improve domain modeling through practical examples.

##### Example 1: Modeling Payment Methods

Consider an e-commerce application that needs to handle different payment methods:

```kotlin
sealed class PaymentMethod {
    data class CreditCard(
        val cardNumber: String,
        val expiryDate: String,
        val cvv: String
    ) : PaymentMethod()

    data class PayPal(val email: String) : PaymentMethod()

    data class BankTransfer(
        val accountNumber: String,
        val bankCode: String
    ) : PaymentMethod()

    object Cash : PaymentMethod()
}

class PaymentProcessor {
    fun process(payment: Payment) {
        val message = when (payment.method) {
            is PaymentMethod.CreditCard -> {
                val card = payment.method
                "Processing credit card payment with card ending with ${card.cardNumber.takeLast(4)}"
            }
            is PaymentMethod.PayPal -> {
                "Processing PayPal payment for ${payment.method.email}"
            }
            is PaymentMethod.BankTransfer -> {
                "Processing bank transfer from account ${payment.method.accountNumber}"
            }
            PaymentMethod.Cash -> {
                "Processing cash payment"
            }
        }
        println(message)
    }
}

data class Payment(
    val amount: Double,
    val currency: String,
    val method: PaymentMethod
)
```

This approach offers several advantages:

1. **Type safety**: The compiler ensures we handle all payment methods
2. **Self-documenting code**: The sealed class clearly shows all possible payment methods
3. **Extensibility**: Adding a new payment method is as simple as adding a new subclass
4. **Pattern matching**: The `when` expression provides a clean way to handle different payment methods

---

##### Example 2: Modeling API Responses

Sealed classes are particularly useful for modeling API responses:

```kotlin
sealed class ApiResponse<out T> {
    data class Success<T>(val data: T) : ApiResponse<T>()
    data class Error(val code: Int, val message: String) : ApiResponse<Nothing>()
    object Loading : ApiResponse<Nothing>()
    object Empty : ApiResponse<Nothing>()
}

class UserRepository {
    fun getUser(id: String): ApiResponse<User> {
        return try {
            // Simulate API call
            if (id == "123") {
                ApiResponse.Success(User("123", "John Doe"))
            } else {
                ApiResponse.Error(404, "User not found")
            }
        } catch (e: Exception) {
            ApiResponse.Error(500, e.message ?: "Unknown error")
        }
    }
}

data class User(val id: String, val name: String)

fun main() {
    val repository = UserRepository()

    val response = repository.getUser("123")

    val result = when (response) {
        is ApiResponse.Success -> "User found: ${response.data.name}"
        is ApiResponse.Error -> "Error: ${response.message} (${response.code})"
        ApiResponse.Loading -> "Loading..."
        ApiResponse.Empty -> "No user data available"
    }

    println(result) // Outputs: User found: John Doe
}
```

This pattern is widely used in Android development with architectures like MVI (Model-View-Intent) and helps create a clear separation between different states of data.

---

#### Sealed Interfaces for More Flexible Hierarchies

Kotlin 1.5 introduced sealed interfaces, which provide more flexibility than sealed classes because a class can implement multiple interfaces:

```kotlin
sealed interface Error {
    val message: String
}

sealed interface NetworkError : Error

data class ServerError(override val message: String) : NetworkError
data class ConnectionError(override val message: String) : NetworkError

sealed interface DatabaseError : Error

data class QueryError(override val message: String) : DatabaseError
data class TransactionError(override val message: String) : DatabaseError

class ErrorHandler {
    fun handle(error: Error) {
        val action = when (error) {
            is ServerError -> "Retry server request"
            is ConnectionError -> "Check internet connection"
            is QueryError -> "Fix database query"
            is TransactionError -> "Rollback transaction"
        }

        println("Error: ${error.message}. Action: $action")
    }
}
```

Sealed interfaces allow for more complex hierarchies while maintaining the benefits of exhaustiveness and type safety.

---

#### Advanced Domain Modeling Patterns

Let's explore some advanced patterns using sealed classes and interfaces.

##### State Machines with Sealed Classes

Sealed classes are excellent for implementing state machines:

```kotlin
sealed class OrderState {
    object Created : OrderState()
    data class Processing(val startTime: Long) : OrderState()
    data class Shipped(val trackingNumber: String) : OrderState()
    data class Delivered(val deliveryTime: Long) : OrderState()
    data class Cancelled(val reason: String) : OrderState()
}

class OrderStateMachine {
    fun transition(currentState: OrderState, event: OrderEvent): OrderState {
        return when (currentState) {
            is OrderState.Created -> handleCreatedState(event)
            is OrderState.Processing -> handleProcessingState(event)
            is OrderState.Shipped -> handleShippedState(event)
            is OrderState.Delivered -> currentState // Terminal state
            is OrderState.Cancelled -> currentState // Terminal state
        }
    }

    private fun handleCreatedState(event: OrderEvent): OrderState {
        return when (event) {
            is OrderEvent.StartProcessing -> OrderState.Processing(System.currentTimeMillis())
            is OrderEvent.CancelOrder -> OrderState.Cancelled(event.reason)
            else -> throw IllegalStateException("Invalid event $event for state Created")
        }
    }

    private fun handleProcessingState(event: OrderEvent): OrderState {
        return when (event) {
            is OrderEvent.ShipOrder -> OrderState.Shipped(event.trackingNumber)
            is OrderEvent.CancelOrder -> OrderState.Cancelled(event.reason)
            else -> throw IllegalStateException("Invalid event $event for state Processing")
        }
    }

    private fun handleShippedState(event: OrderEvent): OrderState {
        return when (event) {
            is OrderEvent.DeliverOrder -> OrderState.Delivered(System.currentTimeMillis())
            else -> throw IllegalStateException("Invalid event $event for state Shipped")
        }
    }
}

sealed class OrderEvent {
    object StartProcessing : OrderEvent()
    data class ShipOrder(val trackingNumber: String) : OrderEvent()
    object DeliverOrder : OrderEvent()
    data class CancelOrder(val reason: String) : OrderEvent()
}
```

This pattern ensures that:
1. All possible states are explicitly defined
2. State transitions are controlled and validated
3. The compiler helps ensure all states are handled
4. The code is self-documenting regarding possible states and transitions

---

#### Best Practices for Using Sealed Classes in Domain Modeling

1. **Use sealed classes for representing finite sets of possibilities**
   - API responses (Success, Error, Loading)
   - State machines (Created, Processing, Completed)
   - Command patterns (Add, Remove, Update)

2. **Prefer sealed interfaces when classes need to implement multiple interfaces**
   - Error hierarchies
   - Feature capabilities
   - Cross-cutting concerns

3. **Combine with data classes for immutable value objects**
   - Makes your domain model more predictable
   - Provides equals(), hashCode(), and toString() implementations
   - Enables destructuring declarations

4. **Leverage exhaustive when expressions**
   - Let the compiler ensure all cases are handled
   - Use the `when` statement without an `else` branch to force handling all cases

5. **Keep the hierarchy shallow**
   - Deep hierarchies can become difficult to understand
   - Consider composition over inheritance for complex behaviors

6. **Use nested sealed classes for related concepts**
   - Helps organize code and maintain context
   - Reduces namespace pollution

---

### Conclusion

Sealed classes and interfaces are powerful tools for domain modeling in Kotlin. They provide type safety, exhaustiveness checking, and clear expression of business concepts. By leveraging these features, you can create more robust, maintainable, and self-documenting domain models.

Remember that good domain modeling is about clearly expressing the business concepts and rules in your code. Sealed classes and interfaces help achieve this goal by providing a way to model finite sets of possibilities in a type-safe manner. Whether you're building an e-commerce platform, a content management system, or a mobile app, these Kotlin features can significantly improve the quality of your domain model.

As with any tool, the key is knowing when and how to apply it. Use sealed classes and interfaces when you need to represent a closed set of possibilities, and combine them with other Kotlin features like data classes and extension functions to create expressive and maintainable domain models.
