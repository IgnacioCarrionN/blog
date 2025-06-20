---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Manejo Elegante de Errores en Kotlin: Usando runCatching y Result"
date: 2025-06-20T08:00:00+01:00
description: "Una guía completa sobre el uso de runCatching y Result de Kotlin para un manejo de errores más elegante sin bloques try/catch, con ejemplos prácticos y mejores prácticas."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/result-pattern.png
draft: false
tags: 
- kotlin
- manejo-de-errores
- programacion-funcional
- patron-result
- manejo-de-excepciones
---

### Manejo Elegante de Errores en Kotlin: Usando runCatching y Result

El manejo de excepciones es un aspecto crítico para escribir aplicaciones robustas, pero los bloques tradicionales try/catch pueden llevar a código verboso y anidado que es difícil de leer y mantener. Kotlin ofrece un enfoque más elegante con la función `runCatching` y el tipo `Result`, que permiten manejar excepciones de manera funcional mientras se mantiene la legibilidad del código y se previenen fallos. Este artículo explora cómo utilizar efectivamente estas características para mejorar tu estrategia de manejo de errores.

---

#### Entendiendo Result y runCatching

La clase `Result` en Kotlin es una unión discriminada que encapsula un resultado exitoso con un valor de tipo `T` o un fallo con una excepción. Es similar al patrón Either que se encuentra en lenguajes de programación funcional.

`runCatching` es una función de la biblioteca estándar que ejecuta un bloque de código dado y envuelve el resultado en un objeto `Result`, capturando cualquier excepción que pueda ocurrir durante la ejecución.

```kotlin
// Enfoque tradicional con try/catch
fun getUserData(userId: String): UserData {
    try {
        val response = api.fetchUser(userId)
        return response.toUserData()
    } catch (e: NetworkException) {
        logger.error("Error de red", e)
    }
}

// Usando runCatching
fun getUserData(userId: String): Result<UserData> {
    return runCatching {
        val response = api.fetchUser(userId)
        response.toUserData()
    }
}
```

Aunque este ejemplo no muestra completamente los beneficios todavía, exploraremos patrones más poderosos a medida que avancemos.

---

#### El Poder de Result: Más Allá de try/catch

El verdadero poder de `Result` proviene de su capacidad para ser pasado y transformado, permitiendo un enfoque más funcional para el manejo de errores.

##### Beneficios Clave de Usar Result

1. **Tipos de Error Explícitos**: Hace visible el manejo de errores en las firmas de funciones
2. **Composición**: Encadena fácilmente operaciones que podrían fallar
3. **Manejo de Errores Diferido**: Separa la lógica de qué hacer del manejo de errores
4. **Flujo de Control Predecible**: Evita que las excepciones interrumpan el flujo normal

```kotlin
// Función que devuelve un Result
fun fetchUserResult(userId: String): Result<UserData> {
    return runCatching {
        val response = api.fetchUser(userId)
        response.toUserData()
    }
}

// Usando el Result
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

#### Transformando y Encadenando Results

Uno de los aspectos más poderosos del tipo `Result` es la capacidad de transformar y encadenar operaciones, similar a cómo trabajarías con otros tipos monádicos como `Optional` o `Stream` en Java.

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
            logger.warn("Sincronización fallida", exception)
            SyncStatus.Failed(reason = exception.message ?: "Error desconocido")
        }
}
```

Las funciones `map`, `flatMap` y `recover` te permiten transformar el valor de éxito o manejar excepciones específicas sin romper la cadena.

---

#### Patrones Prácticos con Result

Exploremos algunos patrones prácticos para usar `Result` en escenarios del mundo real.

##### 1. Patrón Repositorio con Result

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

##### 2. Capa de Servicio API con Result

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
            throw ParseException("Error al analizar la respuesta", e)
        }
    }
}
```

##### 3. Combinando Múltiples Results

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

#### Técnicas Avanzadas

##### 1. Extensiones Personalizadas de Result

Puedes extender la clase `Result` con tus propias funciones de utilidad:

```kotlin
// Extensión para convertir Result a un tipo Either personalizado
fun <T> Result<T>.toEither(): Either<Throwable, T> {
    return fold(
        onSuccess = { Either.Right(it) },
        onFailure = { Either.Left(it) }
    )
}

// Extensión para manejar tipos de error específicos
inline fun <T, reified E : Throwable> Result<T>.onSpecificError(
    crossinline action: (E) -> Unit
): Result<T> {
    return onFailure {
        if (it is E) {
            action(it)
        }
    }
}

// Uso
fetchUserResult(userId)
    .onSpecificError<User, NetworkException> { 
        connectivityManager.retryConnection() 
    }
    .onSuccess { user ->
        // Procesar usuario
    }
```

##### 2. Integración con Coroutines

`Result` funciona perfectamente con coroutines:

```kotlin
suspend fun fetchUserDataAsync(userId: String): Result<UserData> {
    return runCatching {
        val response = api.fetchUserAsync(userId).await()
        response.toUserData()
    }
}

// En un ámbito de coroutine
viewModelScope.launch {
    val result = fetchUserDataAsync(userId)
    result.onSuccess { userData ->
        _uiState.value = SuccessState(userData)
    }.onFailure { error ->
        _uiState.value = ErrorState(error.message ?: "Error desconocido")
    }
}
```

---

#### Mejores Prácticas para Usar Result

1. **Sé Consistente**: Elige si usar `Result` o excepciones en todo tu código
2. **Documenta los Casos de Error**: Deja claro qué tipos de errores pueden ser devueltos
3. **No Mezcles Enfoques**: Evita mezclar `Result` con el manejo tradicional de excepciones
4. **Usa Transformaciones Significativas**: Aprovecha `map`, `flatMap` y `recover` para código limpio
5. **Maneja Todos los Casos**: Siempre maneja tanto los casos de éxito como los de fallo
6. **Evita el Anidamiento**: Usa `flatMap` en lugar de anidar llamadas a `runCatching`
7. **Considera el Rendimiento**: `Result` crea objetos, así que úsalo juiciosamente en código crítico para el rendimiento
8. **Pruebas**: Escribe pruebas tanto para escenarios de éxito como de fallo

```kotlin
// Buena práctica: Manejo claro de errores con transformación
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

// Mala práctica: Mezclando enfoques
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

### Conclusión

La función `runCatching` y el tipo `Result` de Kotlin proporcionan un enfoque funcional y poderoso para el manejo de errores que puede mejorar significativamente la legibilidad y mantenibilidad del código. Al hacer explícitos los posibles fallos y permitir transformaciones funcionales, te permiten escribir código más robusto con un manejo de errores más limpio.

Aunque este enfoque puede no ser adecuado para todas las situaciones, es particularmente valioso en escenarios donde necesitas encadenar operaciones que podrían fallar o cuando quieres diferir el manejo de errores a un nivel superior en tu aplicación. Siguiendo los patrones y mejores prácticas descritos en este artículo, puedes aprovechar estas características para crear aplicaciones más elegantes, confiables y libres de fallos.

Ya sea que estés construyendo una nueva aplicación o refactorizando una existente, considera incorporar `Result` y `runCatching` en tu estrategia de manejo de errores para crear código que sea tanto más funcional como más resistente.