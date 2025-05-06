---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Implementing Room Database in Kotlin Multiplatform Projects"
date: 2025-05-06T08:00:00+01:00
description: "A comprehensive guide to configuring and using Room 2.7.1 in Kotlin Multiplatform projects, including platform-specific setup and practical examples."
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

### Implementing Room Database in Kotlin Multiplatform Projects

Room persistence library has become the standard for database operations in Android development, offering an abstraction layer over SQLite that enables robust database access while harnessing the full power of SQL. With the release of Room 2.7.1, we can now integrate this powerful library into Kotlin Multiplatform (KMP) projects, allowing us to share database code across platforms while leveraging platform-specific optimizations. This blog post explores how to configure, implement, and optimize Room in a KMP environment.

---

#### Understanding Room in Kotlin Multiplatform Context

Room in KMP is not a direct port of the Android-specific Room library to other platforms. Instead, it's a strategic implementation where:

1. The Room annotations and core functionality are used in the common code
2. The `@ConstructedBy` annotation and `RoomDatabaseConstructor` pattern enable platform-specific database initialization
3. The Room compiler automatically generates the necessary platform-specific implementations

This approach allows us to define our database schema, DAOs (Data Access Objects), and entities in the common code, while the underlying database operations are handled by the Room compiler for each platform.

```kotlin
// In commonMain - Entity definition
@Entity(tableName = "users")
data class User(
    @PrimaryKey val id: String,
    val name: String,
    val email: String,
    val createdAt: Long
)

// In commonMain - DAO interface
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

#### Setting Up Room 2.7.1 in a KMP Project

To integrate Room 2.7.1 into your KMP project, you'll need to configure your build files appropriately. Here's a step-by-step guide:

##### 1. Configure the build.gradle.kts file in your shared module

```kotlin
plugins { 
    kotlin("multiplatform")
    id("com.android.library")
    id("com.google.devtools.ksp") version "2.1.20-2.0.1" // KSP for annotation processing
    id("androidx.room")
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

// KSP configuration for Room
dependencies {
    add("kspAndroid", "androidx.room:room-compiler:2.7.1")
    add("kspIosX64", "androidx.room:room-compiler:2.7.1")
    add("kspIosArm64", "androidx.room:room-compiler:2.7.1")
    add("kspIosSimulatorArm64", "androidx.room:room-compiler:2.7.1")
}
```

##### 2. Set up the database class using @ConstructedBy and RoomDatabaseConstructor

```kotlin
// In commonMain
@Database(entities = [User::class], version = 1)
@ConstructedBy(AppDatabaseConstructor::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}

// The Room compiler generates the `actual` implementations.
@Suppress("NO_ACTUAL_FOR_EXPECT")
expect object AppDatabaseConstructor : RoomDatabaseConstructor<AppDatabase> {
    override fun initialize(): AppDatabase
}
```

---

#### Platform-Specific Considerations

##### Android Implementation

On Android, Room works natively as it's designed for the platform. The implementation is straightforward:

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

// In androidMain
class DatabaseProvider(private val context: Context) {
    val database: AppDatabase by lazy {
        getDatabaseBuilder(context)
        .fallbackToDestructiveMigration() // Optional: for development
        .build()
    }
}

// Usage in Android app
val databaseProvider = DatabaseProvider(applicationContext)
val userDao = databaseProvider.database.userDao()

// Collect users as a Flow
lifecycleScope.launch {
    userDao.getAllUsers().collect { users ->
        // Update UI with users
    }
}
```

##### iOS Implementation

For iOS, Room now automatically generates the necessary implementations. You need to provide a database builder function that specifies the database path:

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

// In iosMain
class DatabaseProvider {
    val database: AppDatabase by lazy {
        // Initialize the database using the builder function
        getDatabaseBuilder().build()
    }
}

// No need to create custom AppDatabase or DAO implementations for iOS
// The Room compiler generates all the necessary code automatically
// The DAO interface defined in commonMain is used directly
```

---

#### Practical Example: Implementing a Repository Pattern

To demonstrate a complete implementation, let's create a repository that uses our Room database:

```kotlin
// In commonMain
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

// In commonMain - ViewModel or Presenter
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

#### Advanced Room Features in KMP

Room offers several advanced features that can be leveraged in a KMP environment:

##### 1. Migrations

```kotlin
// In androidMain
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL("ALTER TABLE users ADD COLUMN age INTEGER DEFAULT 0 NOT NULL")
    }
}

// In database builder
Room.databaseBuilder(context, AppDatabase::class.java, "app-database")
    .addMigrations(MIGRATION_1_2)
    .build()
```

##### 2. Type Converters

```kotlin
// In commonMain
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

// Add to database annotation
@Database(entities = [User::class], version = 1)
@TypeConverters(Converters::class)
@ConstructedBy(AppDatabaseConstructor::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}
```

##### 3. Relations Between Entities

```kotlin
// In commonMain
@Entity(tableName = "posts")
data class Post(
    @PrimaryKey val id: String,
    val userId: String, // Foreign key to User
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

#### Best Practices for Room in KMP

1. **Keep entities simple and platform-neutral**
   - Avoid using platform-specific types in your entities
   - Use primitive types and strings where possible
   - Use type converters for complex types

2. **Use coroutines and Flow for asynchronous operations**
   - Room's Flow support works well with Kotlin coroutines
   - This provides a consistent API across platforms

3. **Implement a repository layer**
   - Abstract database operations behind a repository
   - This makes it easier to switch implementations if needed

4. **Handle platform-specific database initialization**
   - Use dependency injection to provide the correct database implementation
   - Consider using a factory pattern for database creation

5. **Test your database code thoroughly**
   - Write tests for your DAOs in commonTest
   - Create platform-specific tests for actual implementations

---

### Conclusion

Integrating Room 2.7.1 into a Kotlin Multiplatform project provides a powerful way to share database code across platforms while leveraging the strengths of each platform's native database capabilities. By using the `@ConstructedBy` annotation and `RoomDatabaseConstructor` pattern, we can define our database schema, DAOs, and entities in common code, while the Room compiler automatically generates the platform-specific implementations.

The approach outlined in this post provides a practical way to share database logic across platforms with minimal platform-specific code. The Room compiler does most of the heavy lifting, generating the necessary implementations for each platform automatically. This significantly reduces the amount of boilerplate code you need to write and maintain.

By following the configuration steps, platform-specific considerations, and best practices outlined in this post, you can successfully implement Room in your KMP projects and create robust, efficient database solutions that work across multiple platforms with minimal platform-specific code.
