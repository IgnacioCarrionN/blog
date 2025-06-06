---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Casos de Uso: Mejorando la Arquitectura de tu Proyecto"
date: 2025-06-06T08:00:00+01:00
description: "Una guía completa para entender los Casos de Uso y cómo pueden mejorar la arquitectura de tus proyectos de software con ejemplos prácticos."
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

### Casos de Uso: Mejorando la Arquitectura de tu Proyecto

En el desarrollo de software moderno, crear aplicaciones mantenibles, testables y escalables es un desafío constante. Un patrón arquitectónico que ha ganado una tracción significativa es el uso de Casos de Uso (también conocidos como Interactores en algunos contextos). Este artículo explora qué son los Casos de Uso, por qué mejoran la arquitectura de tu proyecto y cómo implementarlos de manera efectiva con ejemplos sencillos.

---

#### ¿Qué Son los Casos de Uso?

Los Casos de Uso representan la lógica de negocio o las reglas específicas de la aplicación en tu software. Encapsulan una acción única y específica que se puede realizar en tu aplicación. Piensa en un Caso de Uso como la respuesta a la pregunta: "¿Qué puede hacer el usuario con esta aplicación?"

Algunas características clave de los Casos de Uso:

1. **Responsabilidad Única**: Cada Caso de Uso debe hacer una cosa y hacerla bien
2. **Independencia**: Los Casos de Uso deben ser independientes de la UI, frameworks y agencias externas
3. **Testabilidad**: Deben ser fáciles de probar de forma aislada
4. **Reutilizables**: El mismo Caso de Uso puede ser activado desde diferentes partes de tu aplicación

```kotlin
// Una clase simple de Caso de Uso
class LoginUseCase(private val userRepository: UserRepository) {
    suspend operator fun invoke(username: String, password: String): Result<User> {
        // Detalles de implementación
        return Result.success(User("ejemplo", "Usuario Ejemplo"))
    }
}

// Un Caso de Uso que no requiere parámetros
class GetCurrentUserUseCase(private val userRepository: UserRepository) {
    suspend operator fun invoke(): User? {
        // Detalles de implementación
        return User("actual", "Usuario Actual")
    }
}
```

---

#### Por Qué los Casos de Uso Mejoran Tu Arquitectura

##### 1. Separación de Responsabilidades

Los Casos de Uso crean un límite claro entre tu lógica de negocio y otras capas de tu aplicación. Esta separación hace que tu código sea más organizado y fácil de entender.

```kotlin
// Sin Casos de Uso - Lógica de negocio mezclada con lógica de UI
class UserViewModel(private val userRepository: UserRepository) {
    fun loginUser(username: String, password: String) {
        viewModelScope.launch {
            try {
                // Lógica de validación
                if (username.isEmpty() || password.isEmpty()) {
                    _uiState.value = UiState.Error("El nombre de usuario y la contraseña no pueden estar vacíos")
                    return@launch
                }

                // Lógica de negocio
                val user = userRepository.login(username, password)
                if (user != null) {
                    userRepository.saveUserLocally(user)
                    _uiState.value = UiState.Success(user)
                } else {
                    _uiState.value = UiState.Error("Credenciales inválidas")
                }
            } catch (e: Exception) {
                _uiState.value = UiState.Error("Error de inicio de sesión: ${e.message}")
            }
        }
    }
}

// Con Casos de Uso - Separación limpia
class LoginUseCase(private val userRepository: UserRepository) {
    suspend operator fun invoke(username: String, password: String): Result<User> {
        if (username.isEmpty() || password.isEmpty()) {
            return Result.failure(IllegalArgumentException("El nombre de usuario y la contraseña no pueden estar vacíos"))
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
                onFailure = { UiState.Error(it.message ?: "Error desconocido") }
            )
        }
    }
}
```

##### 2. Mejor Testabilidad

Los Casos de Uso hacen que tu lógica de negocio sea altamente testable porque están aislados de dependencias externas.

```kotlin
// Probando un Caso de Uso
class LoginUseCaseTest {
    @Test
    fun `login con credenciales válidas devuelve éxito`() = runTest {
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
    fun `login con credenciales vacías devuelve fallo`() = runTest {
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

##### 3. Reutilización

Los Casos de Uso pueden ser reutilizados en diferentes partes de tu aplicación, promoviendo la reutilización de código y la consistencia.

```kotlin
// Reutilizando el mismo Caso de Uso en diferentes ViewModels
class LoginViewModel(private val loginUseCase: LoginUseCase) {
    fun login(username: String, password: String) {
        viewModelScope.launch {
            val result = loginUseCase(username, password)
            // Manejar resultado
        }
    }
}

class AutoLoginViewModel(private val loginUseCase: LoginUseCase) {
    fun attemptAutoLogin(savedCredentials: SavedCredentials) {
        viewModelScope.launch {
            val result = loginUseCase(savedCredentials.username, savedCredentials.password)
            // Manejar resultado
        }
    }
}
```

##### 4. Lógica de Negocio Más Fácil de Entender

Los Casos de Uso hacen que tu lógica de negocio sea explícita y más fácil de entender. Al mirar tus clases de Caso de Uso, cualquiera puede comprender rápidamente lo que hace tu aplicación.

```kotlin
// Una lista de Casos de Uso describe claramente las capacidades de la aplicación
class GetUserProfileUseCase(private val userRepository: UserRepository)
class UpdateUserProfileUseCase(private val userRepository: UserRepository)
class ChangePasswordUseCase(private val userRepository: UserRepository)
class LogoutUserUseCase(private val userRepository: UserRepository)
class GetUserPostsUseCase(private val postRepository: PostRepository)
class CreatePostUseCase(private val postRepository: PostRepository)
```

##### 5. Inyección de Dependencias Simplificada

Los Casos de Uso simplifican la inyección de dependencias al reducir el número de dependencias que cada componente necesita.

```kotlin
// Sin Casos de Uso - ViewModel necesita múltiples repositorios
class UserProfileViewModel(
    private val userRepository: UserRepository,
    private val postRepository: PostRepository,
    private val notificationRepository: NotificationRepository
) {
    // Métodos usando todos los repositorios
}

// Con Casos de Uso - ViewModel solo necesita los Casos de Uso relevantes
class UserProfileViewModel(
    private val getUserProfileUseCase: GetUserProfileUseCase,
    private val getUserPostsUseCase: GetUserPostsUseCase,
    private val updateProfileUseCase: UpdateProfileUseCase
) {
    // Métodos usando Casos de Uso
}
```

---

#### Implementando Casos de Uso en Tu Proyecto

Veamos un ejemplo práctico de implementación de Casos de Uso en una aplicación simple:

##### 1. Implementa Casos de Uso Concretos

```kotlin
// Modelos de dominio
data class User(val id: String, val name: String, val email: String)
data class Post(val id: String, val userId: String, val title: String, val content: String)

// Interfaces de repositorio
interface UserRepository {
    suspend fun getUser(id: String): User?
    suspend fun updateUser(user: User): Boolean
}

interface PostRepository {
    suspend fun getPostsByUser(userId: String): List<Post>
    suspend fun createPost(post: Post): Post
}

// Implementaciones de Casos de Uso
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

##### 2. Usa los Casos de Uso en Tu Aplicación

```kotlin
// En un ViewModel
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
                    _userState.value = UserState.Error("Usuario no encontrado")
                }
            } catch (e: Exception) {
                _userState.value = UserState.Error("Error al cargar usuario: ${e.message}")
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
                _postsState.value = PostsState.Error("Error al cargar publicaciones: ${e.message}")
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
                    _userState.value = UserState.Error("Error al actualizar usuario")
                }
            } catch (e: Exception) {
                _userState.value = UserState.Error("Error al actualizar usuario: ${e.message}")
            }
        }
    }
}

// Clases de estado
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

#### Mejores Prácticas para Casos de Uso

1. **Mantén los Casos de Uso Enfocados**: Cada Caso de Uso debe hacer solo una cosa
2. **Usa Nombres Significativos**: Nombra tus Casos de Uso basándote en la acción que realizan (por ejemplo, `GetUserUseCase`, `UpdateProfileUseCase`)
3. **Devuelve Modelos de Dominio**: Los Casos de Uso deben devolver modelos de dominio, no modelos de capa de datos o UI
4. **Maneja Errores Apropiadamente**: Considera usar tipos Result o excepciones para el manejo de errores
5. **Haz que los Casos de Uso sean Testables**: Asegúrate de que tus Casos de Uso puedan ser fácilmente probados de forma aislada
6. **Considera el Rendimiento**: Para operaciones críticas en rendimiento, considera implementar Casos de Uso síncronos
7. **Evita Dependencias Circulares**: Los Casos de Uso no deben depender directamente unos de otros

---

### Conclusión

Los Casos de Uso proporcionan una forma poderosa de organizar tu lógica de negocio y mejorar la arquitectura de tu aplicación. Al separar responsabilidades, mejorar la testabilidad y hacer que tu código sea más mantenible, los Casos de Uso te ayudan a construir mejor software que puede evolucionar con el tiempo.

Los ejemplos en este artículo demuestran cómo los Casos de Uso pueden ser implementados en una aplicación simple, pero los principios se aplican a proyectos de cualquier tamaño. Ya sea que estés construyendo una pequeña aplicación o un gran sistema empresarial, incorporar Casos de Uso en tu arquitectura puede llevar a un código más limpio y mantenible.

Al centrarte en lo que hace tu aplicación en lugar de cómo lo hace, los Casos de Uso te ayudan a crear software que es más fácil de entender, probar y modificar, lo que en última instancia conduce a un proyecto más exitoso.
