---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Implementando Room Database en Proyectos Kotlin Multiplatform"
date: 2025-05-06T08:00:00+01:00
description: "Una guía completa para configurar y utilizar Room 2.7.1 en proyectos Kotlin Multiplatform, incluyendo configuración específica por plataforma y ejemplos prácticos."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/room-kmp.png
draft: false
tags: 
- kotlin
- multiplatform
- kmp
- room
- database
---

### Implementando Room Database en Proyectos Kotlin Multiplatform

La biblioteca de persistencia Room se ha convertido en el estándar para operaciones de base de datos en el desarrollo Android, ofreciendo una capa de abstracción sobre SQLite que permite un acceso robusto a la base de datos mientras aprovecha todo el poder de SQL. Con el lanzamiento de Room 2.7.1, ahora podemos integrar esta potente biblioteca en proyectos Kotlin Multiplatform (KMP), permitiéndonos compartir código de base de datos entre plataformas mientras aprovechamos las optimizaciones específicas de cada plataforma. Este artículo explora cómo configurar, implementar y optimizar Room en un entorno KMP.

---

#### Entendiendo Room en el Contexto de Kotlin Multiplatform

Room en KMP no es un puerto directo de la biblioteca Room específica de Android a otras plataformas. En su lugar, es una implementación estratégica donde:

1. Las anotaciones de Room y la funcionalidad principal se utilizan en el código común
2. La anotación `@ConstructedBy` y el patrón `RoomDatabaseConstructor` permiten la inicialización de la base de datos específica de cada plataforma
3. El compilador de Room genera automáticamente las implementaciones específicas de plataforma necesarias

Este enfoque nos permite definir nuestro esquema de base de datos, DAOs (Objetos de Acceso a Datos) y entidades en el código común, mientras que las operaciones subyacentes de la base de datos son manejadas por el compilador de Room para cada plataforma.

```kotlin
// En commonMain - Definición de entidad
@Entity(tableName = "users")
data class User(
    @PrimaryKey val id: String,
    val name: String,
    val email: String,
    val createdAt: Long
)

// En commonMain - Interfaz DAO
@Dao
interface UserDao {
    @Query("SELECT * FROM users")
    fun getAllUsers(): Flow<List<User>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertUser(user: User)

    @Delete
    suspend fun deleteUser(user: User)
}
```

---

#### Configurando Room 2.7.1 en un Proyecto KMP

Para integrar Room 2.7.1 en tu proyecto KMP, necesitarás configurar tus archivos de build apropiadamente. Aquí hay una guía paso a paso:

##### 1. Configurar el archivo build.gradle.kts en tu módulo compartido

```kotlin
plugins {
    kotlin("multiplatform")
    id("com.android.library")
    id("androidx.room")
    id("com.google.devtools.ksp") version "2.1.20-2.0.1" // KSP para procesamiento de anotaciones
}

kotlin {
    androidTarget()
    iosX64()
    iosArm64()
    iosSimulatorArm64()

    sourceSets {
        val commonMain by getting {
            dependencies {
                implementation("androidx.room:room-runtime:2.7.1")
            }
        }
    }
}

room {
   schemaDirectory("$projectDir/schemas")
}

// Configuración de KSP para Room
dependencies {
    add("kspAndroid", "androidx.room:room-compiler:2.7.1")
    add("kspIosX64", "androidx.room:room-compiler:2.7.1")
    add("kspIosArm64", "androidx.room:room-compiler:2.7.1")
    add("kspIosSimulatorArm64", "androidx.room:room-compiler:2.7.1")
}
```

##### 2. Configurar la clase de base de datos usando @ConstructedBy y RoomDatabaseConstructor

```kotlin
// En commonMain
@Database(entities = [User::class], version = 1)
@ConstructedBy(AppDatabaseConstructor::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}

// El compilador de Room genera las implementaciones `actual`.
@Suppress("NO_ACTUAL_FOR_EXPECT")
expect object AppDatabaseConstructor : RoomDatabaseConstructor<AppDatabase> {
    override fun initialize(): AppDatabase
}
```

---

#### Consideraciones Específicas por Plataforma

##### Implementación en Android

En Android, Room funciona de forma nativa ya que está diseñado para la plataforma. La implementación es sencilla:

```kotlin
// shared/src/androidMain/kotlin/Database.kt

fun getDatabaseBuilder(ctx: Context): RoomDatabase.Builder<AppDatabase> {
  val appContext = ctx.applicationContext
  val dbFile = appContext.getDatabasePath("my_room.db")
  return Room.databaseBuilder<AppDatabase>(
    context = appContext,
    name = dbFile.absolutePath
  )
}

// En androidMain
class DatabaseProvider(private val context: Context) {
    val database: AppDatabase by lazy {
        getDatabaseBuilder(context)
        .fallbackToDestructiveMigration() // Opcional: para desarrollo
        .build()
    }
}

// Uso en la aplicación Android
val databaseProvider = DatabaseProvider(applicationContext)
val userDao = databaseProvider.database.userDao()

// Recolectar usuarios como un Flow
lifecycleScope.launch {
    userDao.getAllUsers().collect { users ->
        // Actualizar UI con usuarios
    }
}
```

##### Implementación en iOS

Para iOS, Room ahora genera automáticamente las implementaciones necesarias. Necesitas proporcionar una función de constructor de base de datos que especifique la ruta de la base de datos:

```kotlin
// shared/src/iosMain/kotlin/Database.kt

fun getDatabaseBuilder(): RoomDatabase.Builder<AppDatabase> {
    val dbFilePath = documentDirectory() + "/my_room.db"
    return Room.databaseBuilder<AppDatabase>(
        name = dbFilePath,
    )
}

private fun documentDirectory(): String {
  val documentDirectory = NSFileManager.defaultManager.URLForDirectory(
    directory = NSDocumentDirectory,
    inDomain = NSUserDomainMask,
    appropriateForURL = null,
    create = false,
    error = null,
  )
  return requireNotNull(documentDirectory?.path)
}

// En iosMain
class DatabaseProvider {
    val database: AppDatabase by lazy {
        // Inicializar la base de datos usando la función de constructor
        getDatabaseBuilder().build()
    }
}

// No es necesario crear implementaciones personalizadas de AppDatabase o DAO para iOS
// El compilador de Room genera automáticamente todo el código necesario
// La interfaz DAO definida en commonMain se usa directamente
```

---

#### Ejemplo Práctico: Implementando un Patrón Repositorio

Para demostrar una implementación completa, vamos a crear un repositorio que utilice nuestra base de datos Room:

```kotlin
// En commonMain
class UserRepository(private val userDao: UserDao) {
    fun getAllUsers(): Flow<List<User>> = userDao.getAllUsers()

    suspend fun addUser(name: String, email: String) {
        val user = User(
            id = UUID.randomUUID().toString(),
            name = name,
            email = email,
            createdAt = Clock.System.now().toEpochMilliseconds()
        )
        userDao.insertUser(user)
    }

    suspend fun deleteUser(user: User) {
        userDao.deleteUser(user)
    }
}

// En commonMain - ViewModel o Presenter
class UserViewModel(private val userRepository: UserRepository) {
    val users: Flow<List<User>> = userRepository.getAllUsers()

    suspend fun addUser(name: String, email: String) {
        if (name.isNotBlank() && email.isNotBlank()) {
            userRepository.addUser(name, email)
        }
    }

    suspend fun deleteUser(user: User) {
        userRepository.deleteUser(user)
    }
}
```

---

#### Características Avanzadas de Room en KMP

Room ofrece varias características avanzadas que pueden aprovecharse en un entorno KMP:

##### 1. Migraciones

```kotlin
// En androidMain
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL("ALTER TABLE users ADD COLUMN age INTEGER DEFAULT 0 NOT NULL")
    }
}

// En el constructor de la base de datos
Room.databaseBuilder(context, AppDatabase::class.java, "app-database")
    .addMigrations(MIGRATION_1_2)
    .build()
```

##### 2. Conversores de Tipo

```kotlin
// En commonMain
class Converters {
    @TypeConverter
    fun fromTimestamp(value: Long?): Date? {
        return value?.let { Date(it) }
    }

    @TypeConverter
    fun dateToTimestamp(date: Date?): Long? {
        return date?.time
    }
}

// Añadir a la anotación de la base de datos
@Database(entities = [User::class], version = 1)
@TypeConverters(Converters::class)
@ConstructedBy(AppDatabaseConstructor::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}
```

##### 3. Relaciones Entre Entidades

```kotlin
// En commonMain
@Entity(tableName = "posts")
data class Post(
    @PrimaryKey val id: String,
    val userId: String, // Clave foránea a User
    val title: String,
    val content: String,
    val createdAt: Long
)

data class UserWithPosts(
    @Embedded val user: User,
    @Relation(
        parentColumn = "id",
        entityColumn = "userId"
    )
    val posts: List<Post>
)

@Dao
interface UserPostDao {
    @Transaction
    @Query("SELECT * FROM users")
    fun getUsersWithPosts(): Flow<List<UserWithPosts>>
}
```

---

#### Mejores Prácticas para Room en KMP

1. **Mantén las entidades simples y neutrales respecto a la plataforma**
   - Evita usar tipos específicos de plataforma en tus entidades
   - Usa tipos primitivos y strings donde sea posible
   - Usa conversores de tipo para tipos complejos

2. **Usa coroutines y Flow para operaciones asíncronas**
   - El soporte de Flow de Room funciona bien con coroutines de Kotlin
   - Esto proporciona una API consistente entre plataformas

3. **Implementa una capa de repositorio**
   - Abstrae las operaciones de base de datos detrás de un repositorio
   - Esto facilita cambiar implementaciones si es necesario

4. **Maneja la inicialización de base de datos específica de plataforma**
   - Usa inyección de dependencias para proporcionar la implementación correcta de la base de datos
   - Considera usar un patrón de fábrica para la creación de la base de datos

5. **Prueba tu código de base de datos exhaustivamente**
   - Escribe pruebas para tus DAOs en commonTest
   - Crea pruebas específicas de plataforma para implementaciones reales

---

### Conclusión

Integrar Room 2.7.1 en un proyecto Kotlin Multiplatform proporciona una forma poderosa de compartir código de base de datos entre plataformas mientras se aprovechan las fortalezas de las capacidades nativas de base de datos de cada plataforma. Mediante el uso de la anotación `@ConstructedBy` y el patrón `RoomDatabaseConstructor`, podemos definir nuestro esquema de base de datos, DAOs y entidades en código común, mientras que el compilador de Room genera automáticamente las implementaciones específicas de plataforma.

El enfoque descrito en este artículo proporciona una forma práctica de compartir lógica de base de datos entre plataformas con un mínimo de código específico de plataforma. El compilador de Room hace la mayor parte del trabajo pesado, generando automáticamente las implementaciones necesarias para cada plataforma. Esto reduce significativamente la cantidad de código repetitivo que necesitas escribir y mantener.

Siguiendo los pasos de configuración, consideraciones específicas de plataforma y mejores prácticas descritas en este artículo, puedes implementar con éxito Room en tus proyectos KMP y crear soluciones de base de datos robustas y eficientes que funcionen en múltiples plataformas con un mínimo de código específico de plataforma.
