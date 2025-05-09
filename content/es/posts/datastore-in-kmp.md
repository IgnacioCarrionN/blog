---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Implementando DataStore en Proyectos Kotlin Multiplatform"
date: 2025-05-09T08:00:00+01:00
description: "Una guía completa para configurar y utilizar DataStore en proyectos Kotlin Multiplatform, incluyendo configuración específica por plataforma y ejemplos prácticos."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/datastore-kmp.png
draft: false
tags: 
- kotlin
- multiplatform
- kmp
- datastore
- preferences
---

### Implementando DataStore en Proyectos Kotlin Multiplatform

DataStore es una solución moderna de almacenamiento de datos desarrollada por Google como reemplazo de SharedPreferences. Proporciona una API consistente y segura en cuanto a tipos para almacenar pares clave-valor y objetos tipados con soporte para coroutines y Flow de Kotlin. Con los recientes avances en Kotlin Multiplatform (KMP), ahora podemos integrar DataStore en nuestros proyectos KMP, permitiéndonos compartir código de preferencias y almacenamiento de datos entre plataformas. Este artículo explora cómo configurar, implementar y optimizar DataStore en un entorno KMP.

---

#### Entendiendo DataStore en el Contexto de Kotlin Multiplatform

DataStore en KMP está diseñado para proporcionar una API consistente entre plataformas mientras aprovecha los mecanismos de almacenamiento específicos de cada plataforma. Hay dos tipos de DataStore:

1. **Preferences DataStore**: Para almacenar pares clave-valor
2. **Proto DataStore**: Para almacenar objetos tipados utilizando Protocol Buffers

En un contexto KMP, DataStore sigue el patrón expect/actual donde:

1. El código común define las interfaces esperadas y los modelos de datos
2. Las implementaciones específicas de plataforma proporcionan los mecanismos de almacenamiento reales
3. La API se mantiene consistente entre plataformas, utilizando coroutines y Flow

Este enfoque nos permite definir nuestros patrones de acceso a datos en código común, mientras que las operaciones de almacenamiento subyacentes son manejadas por implementaciones específicas de plataforma.

```kotlin
// En commonMain - Interfaz DataStore
interface UserPreferences {
    val userData: Flow<UserData>
    suspend fun updateUsername(name: String)
    suspend fun updateEmail(email: String)
    suspend fun clearData()
}

// En commonMain - Modelo de datos
data class UserData(
    val username: String = "",
    val email: String = "",
    val isLoggedIn: Boolean = false
)
```

---

#### Configurando DataStore en un Proyecto KMP

Para integrar DataStore en tu proyecto KMP, necesitarás configurar tus archivos de build apropiadamente. Aquí hay una guía paso a paso:

##### 1. Configurar el archivo build.gradle.kts en tu módulo compartido

```kotlin
plugins {
    kotlin("multiplatform")
    id("com.android.library")
    id("com.google.devtools.ksp") version "2.1.20-2.0.1" // Para Proto DataStore
}

kotlin {
    androidTarget()
    iosX64()
    iosArm64()
    iosSimulatorArm64()

    sourceSets {
        val commonMain by getting {
            dependencies {
                // Para Preferences DataStore
                implementation("androidx.datastore:datastore-preferences-core:1.1.0")

                // Para coroutines
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
            }
        }
    }
}
```

##### 2. Crear la instancia de DataStore desde el código común

```kotlin
/**
 * Gets the singleton DataStore instance, creating it if necessary.
 */
fun createDataStore(producePath: () -> String): DataStore<Preferences> =
   PreferenceDataStoreFactory.createWithPath(
      produceFile = { producePath().toPath() }
   )

internal const val dataStoreFileName = "dice.preferences_pb"
```

---

#### Consideraciones Específicas por Plataforma

##### Implementación en Android

```kotlin
// shared/src/androidMain/kotlin/DataStore.kt

fun createDataStoreAndroid(context: Context): DataStore<Preferences> = createDataStore(
   producePath = { context.filesDir.resolve(dataStoreFileName).absolutePath }
)
```

##### Implementación en iOS

```kotlin
// shared/src/iosMain/kotlin/DataStore.kt

fun createDataStoreIOS(): DataStore<Preferences> = createDataStore(
   producePath = {
      val documentDirectory: NSURL? = NSFileManager.defaultManager.URLForDirectory(
         directory = NSDocumentDirectory,
         inDomain = NSUserDomainMask,
         appropriateForURL = null,
         create = false,
         error = null,
      )
      requireNotNull(documentDirectory).path + "/$dataStoreFileName"
   }
)
```

---

#### Ejemplo Práctico: Implementando un Repositorio de Preferencias de Usuario

Para demostrar una implementación completa, vamos a crear un repositorio que utilice nuestro DataStore:

```kotlin
// En commonMain
class UserPreferencesRepository(private val dataStore: PreferencesDataStore) {
    // Definir claves de preferencias
    private object PreferenceKeys {
        val USERNAME = stringPreferencesKey("username")
        val EMAIL = stringPreferencesKey("email")
        val IS_LOGGED_IN = booleanPreferencesKey("is_logged_in")
    }

    // Obtener datos de usuario como un Flow
    val userData: Flow<UserData> = dataStore.data.map { preferences ->
        UserData(
            username = preferences[PreferenceKeys.USERNAME] ?: "",
            email = preferences[PreferenceKeys.EMAIL] ?: "",
            isLoggedIn = preferences[PreferenceKeys.IS_LOGGED_IN] ?: false
        )
    }

    // Actualizar nombre de usuario
    suspend fun updateUsername(name: String) {
        dataStore.updateData { preferences ->
            preferences.toMutablePreferences().apply {
                this[PreferenceKeys.USERNAME] = name
            }
        }
    }

    // Actualizar email
    suspend fun updateEmail(email: String) {
        dataStore.updateData { preferences ->
            preferences.toMutablePreferences().apply {
                this[PreferenceKeys.EMAIL] = email
            }
        }
    }

    // Establecer estado de inicio de sesión
    suspend fun setLoggedIn(isLoggedIn: Boolean) {
        dataStore.updateData { preferences ->
            preferences.toMutablePreferences().apply {
                this[PreferenceKeys.IS_LOGGED_IN] = isLoggedIn
            }
        }
    }

    // Borrar todos los datos
    suspend fun clearData() {
        dataStore.updateData { preferences ->
            preferences.toMutablePreferences().apply {
                remove(PreferenceKeys.USERNAME)
                remove(PreferenceKeys.EMAIL)
                remove(PreferenceKeys.IS_LOGGED_IN)
            }
        }
    }
}

// En commonMain - ViewModel o Presenter
class UserViewModel(private val userPreferencesRepository: UserPreferencesRepository) {
    val userData: Flow<UserData> = userPreferencesRepository.userData

    suspend fun updateUserProfile(username: String, email: String) {
        if (username.isNotBlank()) {
            userPreferencesRepository.updateUsername(username)
        }

        if (email.isNotBlank()) {
            userPreferencesRepository.updateEmail(email)
        }
    }

    suspend fun login() {
        userPreferencesRepository.setLoggedIn(true)
    }

    suspend fun logout() {
        userPreferencesRepository.setLoggedIn(false)
    }

    suspend fun clearUserData() {
        userPreferencesRepository.clearData()
    }
}
```

---

#### Características Avanzadas de DataStore en KMP

DataStore ofrece varias características avanzadas que pueden aprovecharse en un entorno KMP:

##### 1. Proto DataStore para Objetos Tipados

Si necesitas almacenar objetos complejos, Proto DataStore proporciona una solución segura en cuanto a tipos:

```proto
// Define tu estructura de datos en un archivo .proto
syntax = "proto3";

option java_package = "com.example.app";
option java_multiple_files = true;

message UserPreferences {
  string username = 1;
  string email = 2;
  bool is_logged_in = 3;
}
```

```kotlin
// En commonMain - Crear un serializador
class UserPreferencesSerializer : Serializer<UserPreferences> {
    override val defaultValue: UserPreferences = UserPreferences.getDefaultInstance()

    override suspend fun readFrom(input: InputStream): UserPreferences {
        return UserPreferences.parseFrom(input)
    }

    override suspend fun writeTo(t: UserPreferences, output: OutputStream) {
        t.writeTo(output)
    }
}

// Implementación específica de plataforma para Proto DataStore
```

##### 2. Migración de Datos

```kotlin
// En androidMain - Migrando de SharedPreferences a DataStore
val dataStore = context.createDataStore(
    name = "user_preferences",
    produceMigrations = { context ->
        listOf(
            SharedPreferencesMigration(
                context = context,
                sharedPreferencesName = "legacy_preferences"
            )
        )
    }
)
```

##### 3. Manejo de Excepciones

```kotlin
// En commonMain - Manejo de excepciones durante operaciones de datos
val userData = dataStore.data
    .catch { exception ->
        // Manejar excepción (por ejemplo, corrupción de datos)
        if (exception is IOException) {
            emit(emptyPreferences())
        } else {
            throw exception
        }
    }
    .map { preferences ->
        // Mapear preferencias a tu modelo de datos
        UserData(
            username = preferences[USERNAME] ?: "",
            email = preferences[EMAIL] ?: ""
        )
    }
```

---

#### Mejores Prácticas para DataStore en KMP

1. **Usar el patrón expect/actual de manera efectiva**
   - Definir interfaces claras en el código común
   - Implementar detalles específicos de plataforma en clases actual
   - Mantener la API consistente entre plataformas

2. **Aprovechar coroutines y Flow para operaciones asíncronas**
   - Las operaciones de DataStore son asíncronas por diseño
   - Usar Flow para observar cambios en tus datos almacenados
   - Aplicar operadores de Flow como `map`, `filter` y `combine` para transformaciones de datos

3. **Crear una capa de repositorio**
   - Abstraer las operaciones de DataStore detrás de un repositorio
   - Esto facilita cambiar implementaciones si es necesario
   - Proporciona una API limpia para tu lógica de negocio

4. **Manejar errores con elegancia**
   - Usar el operador `catch` para manejar excepciones en tu Flow
   - Proporcionar valores predeterminados cuando los datos no se pueden leer
   - Considerar implementar mecanismos de reintento para operaciones críticas

5. **Optimizar para el rendimiento**
   - Minimizar el número de actualizaciones de DataStore
   - Agrupar cambios relacionados
   - Usar `distinctUntilChanged()` para evitar emisiones innecesarias

6. **Probar tu código de DataStore exhaustivamente**
   - Escribir pruebas para tus repositorios en commonTest
   - Usar dobles de prueba para simular diferentes escenarios

---

### Conclusión

Integrar DataStore en un proyecto Kotlin Multiplatform proporciona una forma moderna y segura en cuanto a tipos para almacenar y acceder a datos entre plataformas.

El enfoque descrito en este artículo proporciona una forma práctica de compartir lógica de preferencias y almacenamiento de datos entre plataformas con un mínimo de código específico de plataforma. El soporte de DataStore para coroutines y Flow lo convierte en una opción natural para proyectos KMP, permitiendo operaciones de datos reactivas y asíncronas con una API consistente.

Siguiendo los pasos de configuración, consideraciones específicas de plataforma y mejores prácticas descritas en este artículo, puedes implementar con éxito DataStore en tus proyectos KMP y crear soluciones de almacenamiento de datos robustas y eficientes que funcionen en múltiples plataformas con un mínimo de código específico de plataforma.
