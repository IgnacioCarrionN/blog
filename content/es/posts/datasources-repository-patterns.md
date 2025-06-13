---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Patrones DataSource y Repository: Construyendo una Capa de Datos Robusta"
date: 2025-06-13T08:00:00+01:00
description: "Una guía completa para implementar los patrones DataSource y Repository para crear una capa de datos limpia, mantenible y testeable en tus aplicaciones."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/repository.png
draft: false
tags: 
- architecture
- clean-architecture
- repository-pattern
- datasources
- software-design
---

### Patrones DataSource y Repository: Construyendo una Capa de Datos Robusta

En el desarrollo de aplicaciones modernas, gestionar el acceso a datos de manera eficiente es crucial para crear software mantenible y escalable. Dos patrones arquitectónicos que mejoran significativamente la gestión de datos son los patrones DataSource y Repository. Este artículo explora qué son estos patrones, cómo funcionan juntos y cómo implementarlos de manera efectiva con ejemplos prácticos.

---

#### ¿Qué Son los DataSources?

Los DataSources son componentes responsables de manejar operaciones de datos con un origen específico. Abstraen los detalles de cómo se obtienen, almacenan o manipulan los datos de una fuente particular, como:

1. **DataSources Remotos**: Manejan llamadas a API, peticiones de red y almacenamiento en la nube
2. **DataSources Locales**: Gestionan operaciones de base de datos, acceso al sistema de archivos o caché en memoria
3. **DataSources de Servicios de Terceros**: Interactúan con servicios externos como procesadores de pago o plataformas de análisis

Algunas características clave de los DataSources:

1. **Responsabilidad Única**: Cada DataSource se enfoca en un solo origen de datos
2. **Detalles de Implementación**: Contienen los detalles técnicos del acceso a datos
3. **Operaciones de Bajo Nivel**: Realizan operaciones primitivas como CRUD (Crear, Leer, Actualizar, Eliminar)

```kotlin
// Ejemplo de un DataSource Remoto
class UserRemoteDataSource(private val apiService: ApiService) {
    suspend fun getUser(userId: String): UserDto {
        return apiService.getUser(userId)
    }

    suspend fun updateUser(userDto: UserDto): UserDto {
        return apiService.updateUser(userDto)
    }

    suspend fun deleteUser(userId: String): Boolean {
        return apiService.deleteUser(userId)
    }
}

// Ejemplo de un DataSource Local
class UserLocalDataSource(private val userDao: UserDao) {
    suspend fun getUser(userId: String): UserEntity? {
        return userDao.getUserById(userId)
    }

    suspend fun saveUser(userEntity: UserEntity) {
        userDao.insertOrUpdate(userEntity)
    }

    suspend fun deleteUser(userId: String) {
        userDao.deleteUserById(userId)
    }

    suspend fun getAllUsers(): List<UserEntity> {
        return userDao.getAllUsers()
    }
}
```

---

#### ¿Qué Es el Patrón Repository?

El patrón Repository actúa como una capa de abstracción entre tu lógica de negocio y las fuentes de datos. Proporciona una API limpia para el acceso a datos y oculta la complejidad de obtener, combinar y gestionar datos de múltiples fuentes.

Características clave de los Repositories:

1. **Abstracción**: Ocultan los detalles de las operaciones de datos
2. **Coordinación**: Orquestan el acceso a datos a través de múltiples DataSources
3. **Enfoque en el Dominio**: Trabajan con modelos de dominio en lugar de modelos de datos
4. **Reglas de Negocio**: Pueden aplicar reglas de negocio relacionadas con el acceso a datos

```kotlin
// Interfaz del Repository
interface UserRepository {
    suspend fun getUser(userId: String): User
    suspend fun updateUser(user: User): User
    suspend fun deleteUser(userId: String): Boolean
    suspend fun syncUserData(userId: String): User
}

// Implementación del Repository
class UserRepositoryImpl(
    private val remoteDataSource: UserRemoteDataSource,
    private val localDataSource: UserLocalDataSource,
    private val userMapper: UserMapper
) : UserRepository {

    override suspend fun getUser(userId: String): User {
        // Intentar obtener primero de la caché local
        val localUser = localDataSource.getUser(userId)

        // Si se encuentra localmente, devolverlo
        if (localUser != null) {
            return userMapper.mapEntityToDomain(localUser)
        }

        // De lo contrario, obtener del remoto y guardarlo en caché
        val remoteUser = remoteDataSource.getUser(userId)
        val userEntity = userMapper.mapDtoToEntity(remoteUser)
        localDataSource.saveUser(userEntity)

        return userMapper.mapDtoToDomain(remoteUser)
    }

    override suspend fun updateUser(user: User): User {
        // Actualizar primero en remoto
        val userDto = userMapper.mapDomainToDto(user)
        val updatedDto = remoteDataSource.updateUser(userDto)

        // Luego actualizar la caché local
        val userEntity = userMapper.mapDtoToEntity(updatedDto)
        localDataSource.saveUser(userEntity)

        return userMapper.mapDtoToDomain(updatedDto)
    }

    override suspend fun deleteUser(userId: String): Boolean {
        // Eliminar del remoto
        val success = remoteDataSource.deleteUser(userId)

        // Si tiene éxito, también eliminar del local
        if (success) {
            localDataSource.deleteUser(userId)
        }

        return success
    }

    override suspend fun syncUserData(userId: String): User {
        // Forzar una actualización desde el remoto
        val remoteUser = remoteDataSource.getUser(userId)
        val userEntity = userMapper.mapDtoToEntity(remoteUser)
        localDataSource.saveUser(userEntity)

        return userMapper.mapDtoToDomain(remoteUser)
    }
}
```

---

#### Cómo Funcionan Juntos los DataSources y Repositories

La relación entre DataSources y Repositories crea un sistema de gestión de datos potente:

1. **DataSources** manejan el "cómo" del acceso a datos (detalles de implementación)
2. **Repositories** manejan el "qué" del acceso a datos (requisitos de negocio)

Esta separación proporciona varios beneficios:

##### 1. Separación Limpia de Responsabilidades

```kotlin
// Sin separación adecuada - Todo mezclado
class UserViewModel(private val apiService: ApiService, private val userDao: UserDao) {
    fun getUser(userId: String) {
        viewModelScope.launch {
            try {
                // Intentar primero local
                var user = userDao.getUserById(userId)

                if (user == null) {
                    // Obtener de la red
                    val networkUser = apiService.getUser(userId)
                    // Convertir DTO a entidad
                    user = convertDtoToEntity(networkUser)
                    // Guardar en base de datos
                    userDao.insertOrUpdate(user)
                }

                // Convertir a modelo UI
                val uiModel = convertEntityToUiModel(user)
                _userState.value = UserState.Success(uiModel)
            } catch (e: Exception) {
                _userState.value = UserState.Error("Error al cargar usuario")
            }
        }
    }
}

// Con separación adecuada
class UserViewModel(private val userRepository: UserRepository) {
    fun getUser(userId: String) {
        viewModelScope.launch {
            try {
                val user = userRepository.getUser(userId)
                _userState.value = UserState.Success(user)
            } catch (e: Exception) {
                _userState.value = UserState.Error("Error al cargar usuario")
            }
        }
    }
}
```

##### 2. Mejor Testabilidad

Con este patrón, puedes simular fácilmente repositories para probar tu lógica de negocio, y simular data sources para probar tus repositories:

```kotlin
// Probando un ViewModel con un Repository simulado
class UserViewModelTest {
    @Test
    fun `getUser devuelve estado de éxito cuando el repository devuelve usuario`() = runTest {
        // Arrange
        val mockRepository = mockk<UserRepository>()
        val user = User("1", "Juan Pérez", "juan@ejemplo.com")
        coEvery { mockRepository.getUser("1") } returns user

        val viewModel = UserViewModel(mockRepository)

        // Act
        viewModel.getUser("1")

        // Assert
        assertEquals(UserState.Success(user), viewModel.userState.value)
    }
}

// Probando un Repository con DataSources simulados
class UserRepositoryTest {
    @Test
    fun `getUser devuelve datos de la fuente local cuando están disponibles`() = runTest {
        // Arrange
        val mockLocalDataSource = mockk<UserLocalDataSource>()
        val mockRemoteDataSource = mockk<UserRemoteDataSource>()
        val mockMapper = mockk<UserMapper>()

        val userEntity = UserEntity("1", "Juan", "Pérez", "juan@ejemplo.com")
        val user = User("1", "Juan Pérez", "juan@ejemplo.com")

        coEvery { mockLocalDataSource.getUser("1") } returns userEntity
        coEvery { mockMapper.mapEntityToDomain(userEntity) } returns user

        val repository = UserRepositoryImpl(mockRemoteDataSource, mockLocalDataSource, mockMapper)

        // Act
        val result = repository.getUser("1")

        // Assert
        assertEquals(user, result)
        coVerify(exactly = 1) { mockLocalDataSource.getUser("1") }
        coVerify(exactly = 0) { mockRemoteDataSource.getUser(any()) }
    }
}
```

##### 3. Flexibilidad y Adaptabilidad

Este patrón facilita el cambio de fuentes de datos sin afectar al resto de tu aplicación:

```kotlin
// Cambiar de REST API a GraphQL solo requiere cambiar la implementación del DataSource
class UserRemoteDataSourceRest(private val restApiService: RestApiService) : UserRemoteDataSource {
    override suspend fun getUser(userId: String): UserDto {
        return restApiService.getUser(userId)
    }
    // Otras implementaciones
}

class UserRemoteDataSourceGraphQL(private val graphQLClient: GraphQLClient) : UserRemoteDataSource {
    override suspend fun getUser(userId: String): UserDto {
        val response = graphQLClient.execute(UserQueries.GET_USER, mapOf("id" to userId))
        return response.data.user
    }
    // Otras implementaciones
}

// El Repository no necesita cambiar
class UserRepositoryImpl(
    private val remoteDataSource: UserRemoteDataSource, // Interfaz, no implementación concreta
    private val localDataSource: UserLocalDataSource,
    private val userMapper: UserMapper
) : UserRepository {
    // La implementación sigue siendo la misma
}
```

---

#### Mejores Prácticas para DataSources y Repositories

1. **Responsabilidad Única**: Mantén los DataSources enfocados en un solo origen de datos
2. **Diseño Basado en Interfaces**: Define los repositories como interfaces para mejor testabilidad
3. **Manejo de Errores**: Implementa un manejo robusto de errores en los repositories
4. **Estrategia de Caché**: Desarrolla una estrategia clara de caché (basada en tiempo, eventos, etc.)
5. **Soporte Offline**: Usa repositories para proporcionar funcionalidad offline
6. **Inyección de Dependencias**: Usa DI para proporcionar DataSources a los Repositories
7. **Nomenclatura Consistente**: Usa convenciones de nomenclatura consistentes en toda tu capa de datos
8. **Paginación**: Implementa paginación para conjuntos grandes de datos
9. **Patrones Reactivos**: Considera usar Flow o LiveData para observar cambios en los datos
10. **Soporte de Transacciones**: Implementa transacciones para operaciones que modifican múltiples entidades

---

### Conclusión

Los patrones DataSource y Repository proporcionan un enfoque potente para construir una capa de datos robusta en tus aplicaciones. Al separar las preocupaciones del acceso a datos y la lógica de negocio, estos patrones te ayudan a crear código que es más mantenible, testeable y adaptable al cambio.

Implementar estos patrones requiere alguna inversión inicial en arquitectura, pero los beneficios se hacen evidentes rápidamente a medida que tu aplicación crece en complejidad. Con una capa de datos bien diseñada, puedes añadir características más fácilmente, cambiar fuentes de datos, implementar estrategias de caché y proporcionar soporte offline.

Ya sea que estés construyendo una pequeña aplicación o un gran sistema empresarial, los principios de los patrones DataSource y Repository pueden ayudarte a crear una base sólida para tus necesidades de gestión de datos, llevando a un proyecto más exitoso y sostenible.
