---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Elegant Error Handling in Kotlin: Using runCatching and Result"
date: 2025-06-20T08:00:00+01:00
description: "A comprehensive guide to using Kotlin's runCatching and Result for more elegant error handling without try/catch blocks, with practical examples and best practices."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/result-pattern.png
draft: false
tags: 
- kotlin
- error-handling
- functional-programming
- result-pattern
- exception-handling
---

### Elegant Error Handling in Kotlin: Using runCatching and Result

Exception handling is a critical aspect of writing robust applications, but traditional try/catch blocks can lead to verbose, nested code that's difficult to read and maintain. Kotlin offers a more elegant approach with the `runCatching` function and `Result` type, which allow you to handle exceptions in a functional way while maintaining code readability and preventing crashes. This blog post explores how to effectively use these features to improve your error handling strategy.

---

#### Understanding Result and runCatching

The `Result` class in Kotlin is a discriminated union that encapsulates a successful outcome with a value of type `T` or a failure with an exception. It's similar to the Either pattern found in functional programming languages.

`runCatching` is a standard library function that executes a given block of code and wraps the outcome in a `Result` object, catching any exceptions that might occur during execution.

```kotlin
// Traditional try/catch approach
fun getUserData(userId: String): UserData {
    try {
        val response = api.fetchUser(userId)
        return response.toUserData()
    } catch (e: NetworkException) {
        logger.error("Network error", e)
    }
}

// Using runCatching
fun getUserData(userId: String): Result<UserData> {
    return runCatching {
        val response = api.fetchUser(userId)
        response.toUserData()
    }
}
```

While this example doesn't fully showcase the benefits yet, we'll explore more powerful patterns as we proceed.

---

#### The Power of Result: Beyond try/catch

The real power of `Result` comes from its ability to be passed around and transformed, enabling a more functional approach to error handling.

##### Key Benefits of Using Result

1. **Explicit Error Types**: Makes error handling visible in function signatures
2. **Composition**: Easily chain operations that might fail
3. **Deferred Error Handling**: Separate the logic of what to do from error handling
4. **Predictable Control Flow**: Avoid exceptions interrupting the normal flow

```kotlin
// Function that returns a Result
fun fetchUserResult(userId: String): Result<UserData> {
    return runCatching {
        val response = api.fetchUser(userId)
        response.toUserData()
    }
}

// Using the Result
fun processUser(userId: String) {
    val userResult = fetchUserResult(userId)
    
    userResult.onSuccess { userData ->
        displayUserProfile(userData)
        analyticsTracker.logUserFetch(userData.id)
    }.onFailure { exception ->
        when (exception) {
            is NetworkException -> showOfflineMessage()
            is UserNotFoundException -> showUserNotFoundMessage()
            else -> showGenericErrorMessage()
        }
    }
}
```

---

#### Transforming and Chaining Results

One of the most powerful aspects of the `Result` type is the ability to transform and chain operations, similar to how you would work with other monadic types like `Optional` or `Stream` in Java.

```kotlin
fun getUserSettings(userId: String): Result<UserSettings> {
    return fetchUserResult(userId)
        .map { userData -> 
            userData.settings 
        }
        .recover { exception ->
            when (exception) {
                is UserNotFoundException -> UserSettings.createDefault()
                else -> throw exception
            }
        }
}

fun synchronizeUserData(userId: String): Result<SyncStatus> {
    return fetchUserResult(userId)
        .flatMap { userData ->
            runCatching { 
                val cloudData = cloudService.fetchUserData(userId)
                syncService.merge(userData, cloudData)
            }
        }
        .map { mergedData ->
            saveUserData(mergedData)
            SyncStatus.Success(timestamp = System.currentTimeMillis())
        }
        .recoverCatching { exception ->
            logger.warn("Sync failed", exception)
            SyncStatus.Failed(reason = exception.message ?: "Unknown error")
        }
}
```

The `map`, `flatMap`, and `recover` functions allow you to transform the success value or handle specific exceptions without breaking the chain.

---

#### Practical Patterns with Result

Let's explore some practical patterns for using `Result` in real-world scenarios.

##### 1. Repository Pattern with Result

```kotlin
interface UserRepository {
    fun getUser(id: String): Result<User>
    fun saveUser(user: User): Result<Unit>
    fun deleteUser(id: String): Result<Boolean>
}

class UserRepositoryImpl(
    private val remoteDataSource: UserRemoteDataSource,
    private val localDataSource: UserLocalDataSource
) : UserRepository {
    
    override fun getUser(id: String): Result<User> {
        return runCatching {
            val localUser = localDataSource.getUser(id)
            if (localUser != null) {
                return@runCatching localUser
            }
            
            val remoteUser = remoteDataSource.getUser(id)
            localDataSource.saveUser(remoteUser)
            remoteUser
        }
    }
    
    override fun saveUser(user: User): Result<Unit> {
        return runCatching {
            localDataSource.saveUser(user)
            remoteDataSource.saveUser(user)
        }
    }
    
    override fun deleteUser(id: String): Result<Boolean> {
        return runCatching {
            val localResult = localDataSource.deleteUser(id)
            val remoteResult = remoteDataSource.deleteUser(id)
            localResult && remoteResult
        }
    }
}
```

##### 2. API Service Layer with Result

```kotlin
class ApiService(private val httpClient: HttpClient) {
    fun fetchData(endpoint: String): Result<ApiResponse> {
        return runCatching {
            val response = httpClient.get(endpoint)
            if (response.isSuccessful) {
                parseResponse(response.body)
            } else {
                throw HttpException(response.code, response.message)
            }
        }
    }
    
    private fun parseResponse(body: String): ApiResponse {
        return runCatching {
            jsonParser.fromJson(body, ApiResponse::class.java)
        }.getOrElse { e ->
            throw ParseException("Failed to parse response", e)
        }
    }
}
```

##### 3. Combining Multiple Results

```kotlin
fun loadDashboardData(userId: String): Result<DashboardData> {
    val userResult = userRepository.getUser(userId)
    val statsResult = statsRepository.getUserStats(userId)
    val notificationsResult = notificationService.getNotifications(userId)
    
    return runCatching {
        val user = userResult.getOrThrow()
        val stats = statsResult.getOrThrow()
        val notifications = notificationsResult.getOrNull() ?: emptyList()
        
        DashboardData(
            user = user,
            stats = stats,
            notifications = notifications
        )
    }
}
```

---

#### Advanced Techniques

##### 1. Custom Result Extensions

You can extend the `Result` class with your own utility functions:

```kotlin
// Extension to convert Result to a custom Either type
fun <T> Result<T>.toEither(): Either<Throwable, T> {
    return fold(
        onSuccess = { Either.Right(it) },
        onFailure = { Either.Left(it) }
    )
}

// Extension for handling specific error types
inline fun <T, reified E : Throwable> Result<T>.onSpecificError(
    crossinline action: (E) -> Unit
): Result<T> {
    return onFailure {
        if (it is E) {
            action(it)
        }
    }
}

// Usage
fetchUserResult(userId)
    .onSpecificError<User, NetworkException> { 
        connectivityManager.retryConnection() 
    }
    .onSuccess { user ->
        // Process user
    }
```

##### 2. Coroutine Integration

`Result` works seamlessly with coroutines:

```kotlin
suspend fun fetchUserDataAsync(userId: String): Result<UserData> {
    return runCatching {
        val response = api.fetchUserAsync(userId).await()
        response.toUserData()
    }
}

// In a coroutine scope
viewModelScope.launch {
    val result = fetchUserDataAsync(userId)
    result.onSuccess { userData ->
        _uiState.value = SuccessState(userData)
    }.onFailure { error ->
        _uiState.value = ErrorState(error.message ?: "Unknown error")
    }
}
```

---

#### Best Practices for Using Result

1. **Be Consistent**: Choose whether to use `Result` or exceptions throughout your codebase
2. **Document Error Cases**: Make it clear what types of errors can be returned
3. **Don't Mix Approaches**: Avoid mixing `Result` with traditional exception handling
4. **Use Meaningful Transformations**: Leverage `map`, `flatMap`, and `recover` for clean code
5. **Handle All Cases**: Always handle both success and failure cases
6. **Avoid Nesting**: Use `flatMap` instead of nesting `runCatching` calls
7. **Consider Performance**: `Result` creates objects, so use it judiciously in performance-critical code
8. **Testing**: Write tests for both success and failure scenarios

```kotlin
// Good practice: Clear error handling with transformation
fun getUserProfile(userId: String): Result<UserProfile> {
    return userRepository.getUser(userId)
        .map { user -> 
            profileMapper.toProfile(user) 
        }
        .recover { error ->
            when (error) {
                is UserNotFoundException -> UserProfile.createGuestProfile()
                else -> throw error
            }
        }
}

// Bad practice: Mixing approaches
fun getUserProfile(userId: String): UserProfile {
    val result = userRepository.getUser(userId)
    if (result.isSuccess) {
        return profileMapper.toProfile(result.getOrNull()!!)
    } else {
        try {
            throw result.exceptionOrNull()!!
        } catch (e: UserNotFoundException) {
            return UserProfile.createGuestProfile()
        }
    }
}
```

---

### Conclusion

Kotlin's `runCatching` and `Result` type provide a powerful, functional approach to error handling that can significantly improve code readability and maintainability. By making potential failures explicit and enabling functional transformations, they allow you to write more robust code with cleaner error handling.

While this approach may not be suitable for every situation, it's particularly valuable in scenarios where you need to chain operations that might fail or when you want to defer error handling to a higher level in your application. By following the patterns and best practices outlined in this post, you can leverage these features to create more elegant, reliable, and crash-free applications.

Whether you're building a new application or refactoring an existing one, consider incorporating `Result` and `runCatching` into your error handling strategy to create code that's both more functional and more resilient.