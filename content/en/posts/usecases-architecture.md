---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "UseCases: Improving Your Project Architecture"
date: 2025-06-06T08:00:00+01:00
description: "A comprehensive guide to understanding UseCases and how they can improve the architecture of your software projects with practical examples."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/usecase.png
draft: false
tags: 
- architecture
- clean-architecture
- usecases
- software-design
---

### UseCases: Improving Your Project Architecture

In modern software development, creating maintainable, testable, and scalable applications is a constant challenge. One architectural pattern that has gained significant traction is the use of UseCases. This blog post explores what UseCases are, why they improve your project's architecture, and how to implement them effectively with simple examples.

---

#### What Are UseCases?

UseCases represent the business logic or application-specific rules of your software. They encapsulate a single, specific action that can be performed in your application. Think of a UseCase as answering the question: "What can the user do with this application?"

Some key characteristics of UseCases:

1. **Single Responsibility**: Each UseCase should do one thing and do it well
2. **Independent**: UseCases should be independent of UI, frameworks, and external agencies
3. **Testable**: They should be easy to test in isolation
4. **Reusable**: The same UseCase can be triggered from different parts of your application

```kotlin
// A simple UseCase class
class LoginUseCase(private val userRepository: UserRepository) {
    suspend operator fun invoke(username: String, password: String): Result<User> {
        // Implementation details
        return Result.success(User("example", "Example User"))
    }
}

// A UseCase that doesn't require parameters
class GetCurrentUserUseCase(private val userRepository: UserRepository) {
    suspend operator fun invoke(): User? {
        // Implementation details
        return User("current", "Current User")
    }
}
```

---

#### Why UseCases Improve Your Architecture

##### 1. Separation of Concerns

UseCases create a clear boundary between your business logic and other layers of your application. This separation makes your codebase more organized and easier to understand.

```kotlin
// Without UseCases - Business logic mixed with UI logic
class UserViewModel(private val userRepository: UserRepository) {
    fun loginUser(username: String, password: String) {
        viewModelScope.launch {
            try {
                // Validation logic
                if (username.isEmpty() || password.isEmpty()) {
                    _uiState.value = UiState.Error("Username and password cannot be empty")
                    return@launch
                }

                // Business logic
                val user = userRepository.login(username, password)
                if (user != null) {
                    userRepository.saveUserLocally(user)
                    _uiState.value = UiState.Success(user)
                } else {
                    _uiState.value = UiState.Error("Invalid credentials")
                }
            } catch (e: Exception) {
                _uiState.value = UiState.Error("Login failed: ${e.message}")
            }
        }
    }
}

// With UseCases - Clean separation
class LoginUseCase(private val userRepository: UserRepository) {
    suspend operator fun invoke(username: String, password: String): Result<User> {
        if (username.isEmpty() || password.isEmpty()) {
            return Result.failure(IllegalArgumentException("Username and password cannot be empty"))
        }

        return try {
            val user = userRepository.login(username, password)
            if (user != null) {
                userRepository.saveUserLocally(user)
                Result.success(user)
            } else {
                Result.failure(InvalidCredentialsException())
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}

class UserViewModel(private val loginUseCase: LoginUseCase) {
    fun loginUser(username: String, password: String) {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            val result = loginUseCase(username, password)
            _uiState.value = result.fold(
                onSuccess = { UiState.Success(it) },
                onFailure = { UiState.Error(it.message ?: "Unknown error") }
            )
        }
    }
}
```

##### 2. Improved Testability

UseCases make your business logic highly testable because they're isolated from external dependencies.

```kotlin
// Testing a UseCase
class LoginUseCaseTest {
    @Test
    fun `login with valid credentials returns success`() = runTest {
        // Arrange
        val mockRepository = mockk<UserRepository>()
        val user = User("john", "John Doe")
        coEvery { mockRepository.login("john", "password123") } returns user
        coEvery { mockRepository.saveUserLocally(user) } just Runs

        val loginUseCase = LoginUseCase(mockRepository)

        // Act
        val result = loginUseCase("john", "password123")

        // Assert
        assertTrue(result.isSuccess)
        assertEquals(user, result.getOrNull())
    }

    @Test
    fun `login with empty credentials returns failure`() = runTest {
        // Arrange
        val mockRepository = mockk<UserRepository>()
        val loginUseCase = LoginUseCase(mockRepository)

        // Act
        val result = loginUseCase("", "")

        // Assert
        assertTrue(result.isFailure)
        assertTrue(result.exceptionOrNull() is IllegalArgumentException)
    }
}
```

##### 3. Reusability

UseCases can be reused across different parts of your application, promoting code reuse and consistency.

```kotlin
// Reusing the same UseCase in different ViewModels
class LoginViewModel(private val loginUseCase: LoginUseCase) {
    fun login(username: String, password: String) {
        viewModelScope.launch {
            val result = loginUseCase(username, password)
            // Handle result
        }
    }
}

class AutoLoginViewModel(private val loginUseCase: LoginUseCase) {
    fun attemptAutoLogin(savedCredentials: SavedCredentials) {
        viewModelScope.launch {
            val result = loginUseCase(savedCredentials.username, savedCredentials.password)
            // Handle result
        }
    }
}
```

##### 4. Easier to Understand Business Logic

UseCases make your business logic explicit and easier to understand. By looking at your UseCase classes, anyone can quickly grasp what your application does.

```kotlin
// A list of UseCases clearly describes application capabilities
class GetUserProfileUseCase(private val userRepository: UserRepository)
class UpdateUserProfileUseCase(private val userRepository: UserRepository)
class ChangePasswordUseCase(private val userRepository: UserRepository)
class LogoutUserUseCase(private val userRepository: UserRepository)
class GetUserPostsUseCase(private val postRepository: PostRepository)
class CreatePostUseCase(private val postRepository: PostRepository)
```

##### 5. Simplified Dependency Injection

UseCases simplify dependency injection by reducing the number of dependencies each component needs.

```kotlin
// Without UseCases - ViewModel needs multiple repositories
class UserProfileViewModel(
    private val userRepository: UserRepository,
    private val postRepository: PostRepository,
    private val notificationRepository: NotificationRepository
) {
    // Methods using all repositories
}

// With UseCases - ViewModel only needs relevant UseCases
class UserProfileViewModel(
    private val getUserProfileUseCase: GetUserProfileUseCase,
    private val getUserPostsUseCase: GetUserPostsUseCase,
    private val updateProfileUseCase: UpdateProfileUseCase
) {
    // Methods using UseCases
}
```

---

#### Implementing UseCases in Your Project

Let's look at a practical example of implementing UseCases in a simple application:

##### 1. Implement Concrete UseCases

```kotlin
// Domain models
data class User(val id: String, val name: String, val email: String)
data class Post(val id: String, val userId: String, val title: String, val content: String)

// Repository interfaces
interface UserRepository {
    suspend fun getUser(id: String): User?
    suspend fun updateUser(user: User): Boolean
}

interface PostRepository {
    suspend fun getPostsByUser(userId: String): List<Post>
    suspend fun createPost(post: Post): Post
}

// UseCase implementations
class GetUserUseCase(private val userRepository: UserRepository) {
    suspend fun invoke(userId: String): User? {
        return userRepository.getUser(userId)
    }
}

class GetUserPostsUseCase(private val postRepository: PostRepository) {
    suspend fun invoke(userId: String): List<Post> {
        return postRepository.getPostsByUser(userId)
    }
}

class CreatePostUseCase(private val postRepository: PostRepository) {
    suspend fun invoke(post: Post): Post {
        return postRepository.createPost(post)
    }
}

class UpdateUserUseCase(private val userRepository: UserRepository) {
    suspend fun invoke(user: User): Boolean {
        return userRepository.updateUser(user)
    }
}
```

##### 2. Use the UseCases in Your Application

```kotlin
// In a ViewModel
class UserProfileViewModel(
    private val getUserUseCase: GetUserUseCase,
    private val getUserPostsUseCase: GetUserPostsUseCase,
    private val updateUserUseCase: UpdateUserUseCase
) : ViewModel() {

    private val _userState = MutableStateFlow<UserState>(UserState.Loading)
    val userState: StateFlow<UserState> = _userState

    private val _postsState = MutableStateFlow<PostsState>(PostsState.Loading)
    val postsState: StateFlow<PostsState> = _postsState

    fun loadUserProfile(userId: String) {
        viewModelScope.launch {
            _userState.value = UserState.Loading
            try {
                val user = getUserUseCase(userId)
                if (user != null) {
                    _userState.value = UserState.Success(user)
                    loadUserPosts(userId)
                } else {
                    _userState.value = UserState.Error("User not found")
                }
            } catch (e: Exception) {
                _userState.value = UserState.Error("Failed to load user: ${e.message}")
            }
        }
    }

    private fun loadUserPosts(userId: String) {
        viewModelScope.launch {
            _postsState.value = PostsState.Loading
            try {
                val posts = getUserPostsUseCase(userId)
                _postsState.value = PostsState.Success(posts)
            } catch (e: Exception) {
                _postsState.value = PostsState.Error("Failed to load posts: ${e.message}")
            }
        }
    }

    fun updateUserProfile(user: User) {
        viewModelScope.launch {
            _userState.value = UserState.Loading
            try {
                val success = updateUserUseCase(user)
                if (success) {
                    _userState.value = UserState.Success(user)
                } else {
                    _userState.value = UserState.Error("Failed to update user")
                }
            } catch (e: Exception) {
                _userState.value = UserState.Error("Failed to update user: ${e.message}")
            }
        }
    }
}

// State classes
sealed class UserState {
    object Loading : UserState()
    data class Success(val user: User) : UserState()
    data class Error(val message: String) : UserState()
}

sealed class PostsState {
    object Loading : PostsState()
    data class Success(val posts: List<Post>) : PostsState()
    data class Error(val message: String) : PostsState()
}
```

---

#### Best Practices for UseCases

1. **Keep UseCases Focused**: Each UseCase should do one thing only
2. **Use Meaningful Names**: Name your UseCases based on the action they perform (e.g., `GetUserUseCase`, `UpdateProfileUseCase`)
3. **Return Domain Models**: UseCases should return domain models, not data layer or UI models
4. **Handle Errors Appropriately**: Consider using Result types or exceptions for error handling
5. **Make UseCases Testable**: Ensure your UseCases can be easily tested in isolation
6. **Consider Performance**: For performance-critical operations, consider implementing synchronous UseCases
7. **Avoid Circular Dependencies**: UseCases should not depend on each other directly

---

### Conclusion

UseCases provide a powerful way to organize your business logic and improve your application's architecture. By separating concerns, enhancing testability, and making your code more maintainable, UseCases help you build better software that can evolve over time.

The examples in this post demonstrate how UseCases can be implemented in a simple application, but the principles apply to projects of any size. Whether you're building a small app or a large enterprise system, incorporating UseCases into your architecture can lead to cleaner, more maintainable code.

By focusing on what your application does rather than how it does it, UseCases help you create software that's easier to understand, test, and modify—ultimately leading to a more successful project.
