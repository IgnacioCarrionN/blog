---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Clean Architecture in Kotlin & Android"
date: 2025-01-18T08:00:00+01:00
description: "Clean Architecture in Kotlin & Android with practical examples"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/domain-layer.png
draft: false
tags: 
- kotlin
- architecture
---

# Clean Architecture in Kotlin & Android

## Introduction

When building Android applications, maintaining scalability and readability is crucial. Without a clear architectural approach, projects can become difficult to maintain as they grow. This is where **Clean Architecture**, introduced by **Uncle Bob (Robert C. Martin)**, becomes invaluable. It emphasizes separation of concerns, making code more modular, testable, and maintainable.

## Understanding Clean Architecture

Clean Architecture is structured into **three main layers**, each with a specific role:

- **Presentation Layer**: Handles UI and user interactions.
- **Domain Layer**: Contains business logic, use cases, and repository interfaces.
- **Data Layer**: Implements repositories, manages API calls, and handles database operations.

The core principle of Clean Architecture is **dependency direction**—each layer should only depend on the layers closer to the core (domain). This ensures flexibility and scalability.

## Project Structure

A Clean Architecture project in Kotlin typically follows this structure:

```
com.example.app
│── presentation (ViewModels, UI, State)
│── domain (UseCases, Repository Interfaces, Models)
│── data (Repository Implementations, Data Sources, APIs, DB)
```

Each layer should be in a separate module or package, ensuring proper separation of concerns.

## Modularization

To further enhance maintainability and scalability, consider structuring your project into separate **Gradle modules**. This ensures clear separation between different layers and promotes reusability.

A modularized Clean Architecture project could follow this structure:

```
com.example.app
│── app (Main application module)
│── feature-user
│   │── domain (UseCases, Repository Interfaces, Models)
│   │── data (Repository Implementations, Data Sources, APIs, DB)
│   │── presentation (UI and ViewModels for user features)
│── core (Common utilities, networking, database helpers)
```

Benefits of modularization:

- Faster build times due to isolated module compilation.
- Improved code encapsulation and separation of concerns.
- Easier feature development and maintenance.
- Better testability by allowing independent testing of modules.

## Implementing Clean Architecture with Kotlin

### 1. **Domain Layer (Core Business Logic)**

The **domain layer** defines the business logic and use cases. It does not depend on any framework or external library, making it the most stable part of the application.

#### Example: Defining a Repository Interface

```kotlin
interface UserRepository {
    suspend fun getUserById(id: String): User
}
```

#### Example: Use Case

```kotlin
class GetUserByIdUseCase(private val userRepository: UserRepository) {
    suspend operator fun invoke(id: String): User {
        return userRepository.getUserById(id)
    }
}
```

### 2. **Data Layer (Implementing Repositories and Data Sources)**

The **data layer** provides concrete implementations of the repository interfaces. It interacts with APIs, databases, or local storage.

#### Example: Data Source

```kotlin
interface UserRemoteDataSource {
    suspend fun fetchUserById(id: String): User
}

class UserRemoteDataSourceImpl(private val api: UserApi) : UserRemoteDataSource {
    override suspend fun fetchUserById(id: String): User {
        return api.fetchUserById(id)
    }
}
```

#### Example: Repository Implementation

```kotlin
class UserRepositoryImpl(private val remoteDataSource: UserRemoteDataSource) : UserRepository {
    override suspend fun getUserById(id: String): User {
        return remoteDataSource.fetchUserById(id)
    }
}
```

### 3. **Presentation Layer (UI & ViewModel)**

The **presentation layer** is responsible for UI logic and state management. It depends on the **domain layer** but does not interact directly with the **data layer**.

#### Example: ViewModel

```kotlin
class UserViewModel(private val getUserByIdUseCase: GetUserByIdUseCase) : ViewModel() {

    private val _user = MutableStateFlow<User?>(null)
    val user: StateFlow<User?> get() = _user.asStateFlow()

    fun loadUser(id: String) {
        viewModelScope.launch {
            _user.value = getUserByIdUseCase(id)
        }
    }
}
```

## Best Practices

1. **Keep the Domain Layer Pure**: It should have no dependencies on Android frameworks.
2. **Use Dependency Injection**: Koin helps in managing dependencies cleanly.
3. **Follow the Dependency Rule**: The inner layers should not depend on the outer layers.
4. **Separate Repository Interfaces and Implementations**: Interfaces go in the domain layer, and implementations stay in the data layer.
5. **Use Data Sources**: Encapsulate API and database calls in dedicated data source classes.
6. **Modularize Your Code**: Use Gradle modules to separate concerns and improve build times.

## Conclusion

Clean Architecture provides a robust way to structure Android applications. By separating concerns and enforcing clear dependencies, it makes code more testable and scalable. Using **Koin** for dependency injection further enhances maintainability. Adopting this architecture, along with **modularization**, will result in a more modular and resilient codebase for your Kotlin projects.
