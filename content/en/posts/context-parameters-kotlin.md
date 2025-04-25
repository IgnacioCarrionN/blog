---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Understanding Context Parameters in Kotlin 2.2.0"
date: 2025-04-25T08:00:00+01:00
description: "Exploring Kotlin 2.2.0's new context parameters feature and how it can improve your code's readability and maintainability."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/context-parameters.png
draft: false
tags: 
- kotlin
- language-features
---

### Understanding Context Parameters in Kotlin 2.2.0

Kotlin 2.2.0 introduces an exciting new language feature called "context parameters" that promises to make your code more concise, readable, and maintainable. This feature addresses the common challenge of passing contextual information through deep call hierarchies without cluttering function signatures. In this blog post, we'll explore what context parameters are, how they work, and how you can leverage them in your Kotlin projects.

---

#### What Are Context Parameters?

Context parameters are a new way to declare dependencies in function signatures that are implicitly passed from callers to callees. They serve as an alternative to explicitly passing parameters through every function in a call chain, reducing boilerplate while maintaining type safety.

```kotlin
// Function with a context parameter
context(logger: Logger)
fun processData(data: Data) {
    // Use the Logger context directly
    logger.info("Processing data: $data")

    // Process the data...
    val result = data.transform()

    logger.debug("Processing complete, result: $result")
}

// Calling the function with a context
with(ConsoleLogger()) {
    processData(myData)
}
```

In this example, `Logger` is a context parameter for the `processData` function. The function can directly use methods from `Logger` without explicitly receiving it as a parameter. The caller provides the context using standard Kotlin scope functions like `with`, `run`, or `apply`.

---

#### How Context Parameters Differ from Extension Receivers

Kotlin developers might initially confuse context parameters with extension receivers, but they serve different purposes and have distinct capabilities:

```kotlin
// Extension function
fun Logger.processDataAsExtension(data: Data) {
    info("Processing data: $data")
    // ...
}

// Function with context parameter
context(logger: Logger)
fun processDataWithContext(data: Data) {
    logger.info("Processing data: $data")
    // ...
}
```

Key differences include:

1. **Multiple Contexts**: You can specify multiple context parameters, unlike extension receivers.
   ```kotlin
   context(logger: Logger, txManager: TransactionManager, auth: UserAuthorization)
   fun performComplexOperation(data: Data) {
       // Use methods from all three contexts
   }
   ```

2. **Composition**: Context parameters compose better with extension functions.
   ```kotlin
   context(txManager: TransactionManager)
   fun List<Transaction>.processAll() {
       // Both TransactionManager context and List<Transaction> receiver available
   }
   ```

3. **Implicit Propagation**: Context parameters are implicitly passed down the call chain.

---

#### Context Receivers vs. Context Parameters: Important Update

It's important to note that context parameters are the evolution of an earlier experimental feature called "context receivers." Context receivers are being deprecated in favor of context parameters. Here are the key differences:

1. **Named Parameters**: Context parameters require a name (`context(logger: Logger)`), while context receivers only specified the type (`context(Logger)`).
2. **Explicit Usage**: With context parameters, you must use the parameter name to access methods (`logger.info()`), whereas context receivers allowed direct method access (`info()`).
3. **Clarity and Maintainability**: Named parameters provide better clarity about which context is being used, especially when multiple contexts are involved.
4. **IDE Support**: Named parameters enable better IDE support, including code completion and navigation.

This change aligns with Kotlin's philosophy of explicit over implicit when it improves code clarity and maintainability. If you've been using context receivers in experimental code, you should migrate to context parameters as they are the officially supported feature moving forward.

---

#### Advantages of Using Context Parameters

Context parameters offer several benefits that can significantly improve your codebase:

1. **Reduced Boilerplate**
   - Eliminate the need to pass the same parameters through multiple layers of function calls
   - Make function signatures cleaner and more focused on their primary purpose
   - Reduce the verbosity of code that deals with cross-cutting concerns

2. **Improved Readability**
   - Function calls focus on the essential parameters
   - Context-dependent operations become more intuitive
   - Code reads more like natural language with fewer interruptions

3. **Better Maintainability**
   - Changes to contextual requirements don't cascade through the entire call hierarchy
   - Adding new contextual dependencies has minimal impact on existing code
   - Testing becomes easier with explicit context boundaries

4. **Type Safety**
   - Unlike global variables or singletons, context parameters maintain compile-time type safety
   - The compiler ensures that required contexts are provided
   - IDE support for code completion and navigation works with context parameters

---

#### Practical Examples of Context Parameters

Let's explore some real-world scenarios where context parameters shine:

##### Example 1: Logging Framework

```kotlin
interface Logger {
    fun debug(message: String)
    fun info(message: String)
    fun warn(message: String)
    fun error(message: String, throwable: Throwable? = null)
}

class ConsoleLogger : Logger {
    override fun debug(message: String) = println("[DEBUG] $message")
    override fun info(message: String) = println("[INFO] $message")
    override fun warn(message: String) = println("[WARN] $message")
    override fun error(message: String, throwable: Throwable?) {
        println("[ERROR] $message")
        throwable?.printStackTrace()
    }
}

// Using context parameters for logging
context(logger: Logger)
fun processUserData(user: User) {
    logger.info("Processing data for user: ${user.id}")

    try {
        val result = user.processProfile()
        logger.debug("Profile processed: $result")

        val permissions = user.calculatePermissions()
        logger.debug("Permissions calculated: $permissions")
    } catch (e: Exception) {
        logger.error("Failed to process user data", e)
    }
}

// Usage
fun main() {
    val user = User(id = "12345", name = "John Doe")

    with(ConsoleLogger()) {
        processUserData(user)
    }
}
```

This example demonstrates how context parameters can simplify logging throughout a codebase without passing a logger instance explicitly to every function.

---

##### Example 2: Dependency Injection

```kotlin
class UserRepository {
    fun getUser(id: String): User = // implementation
    fun saveUser(user: User): Boolean = // implementation
}

class TransactionManager {
    fun beginTransaction() { /* implementation */ }
    fun commitTransaction() { /* implementation */ }
    fun rollbackTransaction() { /* implementation */ }
}

class NotificationService {
    fun sendNotification(userId: String, message: String) { /* implementation */ }
}

// Using multiple context parameters
context(repo: UserRepository, txManager: TransactionManager, notificationService: NotificationService)
fun updateUserProfile(userId: String, profileUpdate: ProfileUpdate): Boolean {
    txManager.beginTransaction()

    try {
        val user = repo.getUser(userId)
        user.applyProfileUpdate(profileUpdate)
        val success = repo.saveUser(user)

        if (success) {
            txManager.commitTransaction()
            notificationService.sendNotification(userId, "Your profile has been updated")
        } else {
            txManager.rollbackTransaction()
        }

        return success
    } catch (e: Exception) {
        txManager.rollbackTransaction()
        throw e
    }
}

// Usage
fun main() {
    val userRepo = UserRepository()
    val txManager = TransactionManager()
    val notificationService = NotificationService()

    with(userRepo) {
        with(txManager) {
            with(notificationService) {
                updateUserProfile("12345", ProfileUpdate(name = "Jane Doe"))
            }
        }
    }

    // Or more concisely with Kotlin's run function:
    run {
        context(userRepo, txManager, notificationService)
        updateUserProfile("12345", ProfileUpdate(name = "Jane Doe"))
    }
}
```

This example shows how context parameters can simplify dependency injection patterns by making dependencies available implicitly.

---

#### Best Practices for Using Context Parameters

To get the most out of context parameters, consider these best practices:

1. **Use for Cross-Cutting Concerns**
   - Logging, tracing, and monitoring
   - Transaction management
   - Security and authorization
   - Configuration and environment settings

2. **Keep Context Interfaces Focused**
   - Define small, cohesive interfaces for contexts
   - Avoid large contexts with many unrelated methods
   - Consider using composition of multiple contexts instead

3. **Be Mindful of Nesting and Scope**
   - Clearly define where contexts begin and end
   - Avoid deeply nested context blocks
   - Consider using extension functions on context parameters for better organization

4. **Document Context Requirements**
   - Clearly document what each context parameter is used for
   - Explain the expected behavior of context implementations
   - Provide examples of how to supply the required contexts

5. **Testing with Context Parameters**
   - Create test-specific implementations of context interfaces
   - Use mock frameworks that support context parameters
   - Consider creating test utilities to simplify providing test contexts

---

#### Combining with Extension Functions

```kotlin
// Extension function with context parameter
context(txManager: TransactionManager)
fun List<Transaction>.processAllInTransaction() {
    txManager.beginTransaction()
    try {
        forEach { it.process() }
        txManager.commitTransaction()
    } catch (e: Exception) {
        txManager.rollbackTransaction()
        throw e
    }
}

// Usage
with(transactionManager) {
    transactions.processAllInTransaction()
}
```


---

### Conclusion

Context parameters in Kotlin 2.2.0 represent a significant enhancement to the language, offering a powerful way to manage contextual dependencies with less boilerplate. By allowing implicit passing of parameters through call chains, they address a common pain point in software development while maintaining Kotlin's commitment to type safety and readability.

As you incorporate context parameters into your codebase, start with clear, focused use cases like logging or dependency injection. Over time, you'll discover more opportunities to leverage this feature to make your code more concise and maintainable.

Remember that context parameters are a tool in your Kotlin toolkit—use them judiciously alongside other language features to create clean, expressive, and maintainable code. With the right approach, context parameters can significantly improve the way you structure and organize your Kotlin applications.
