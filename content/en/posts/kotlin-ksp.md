---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "A deep dive into Kotlin KSP"
date: 2024-12-27T08:00:00+01:00
description: "Advanced Kotlin - Kotlin Symbol Processing (KSP)"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/koin-annotations.png
draft: false
tags: 
- kotlin
- android
- advanced
---

# A Deep Dive into Kotlin Symbol Processing (KSP) with Practical Examples

Kotlin Symbol Processing (KSP) is a powerful tool introduced to streamline annotation processing in Kotlin. Compared to `kapt` (Kotlin Annotation Processing Tool), KSP is faster, offers better integration with Kotlin, and reduces build times significantly. In this post, we’ll explore the fundamentals of KSP, discuss how it works, and demonstrate its use with popular libraries like Koin and Room.

---

## What is KSP?

KSP is a lightweight and efficient API for processing Kotlin source code. It allows you to build annotation processors that work directly with Kotlin's syntax tree rather than relying on Java-based tools. This makes it a perfect fit for Kotlin-first projects.

### Benefits of KSP:

1. **Speed**: Processes Kotlin code faster than `kapt`.
2. **Kotlin-First**: Works directly with Kotlin language constructs, avoiding Java-based abstractions.
3. **Lightweight**: Reduces boilerplate and integrates seamlessly with Gradle.
4. **Compatibility**: Many popular libraries now support KSP natively.

---

## Setting Up KSP in Your Project

Add the KSP plugin to your project:

### Gradle Configuration

```kotlin
plugins {
    kotlin("jvm") version "<latest-kotlin-version>"
    id("com.google.devtools.ksp") version "<latest-ksp-version>"
}

repositories {
    mavenCentral()
}

dependencies {
    implementation(kotlin("stdlib"))
    ksp("<ksp-processor-library>")
}
```

Replace `<ksp-processor-library>` with the library-specific processor dependency, as shown in the examples below.

---

## Example 1: KSP with Koin Dependency Injection

Koin, starting from version 3.4.0, provides annotations to define dependencies, which are processed using KSP to generate Koin modules.

### Setup Koin with KSP

Add the following dependencies:

```kotlin
dependencies {
    implementation("io.insert-koin:koin-core:<version>")
    implementation("io.insert-koin:koin-annotations:<version>")
    ksp("io.insert-koin:koin-ksp-compiler:<version>")
}
```

### Annotate Classes

Use Koin annotations to define your dependency graph:

```kotlin
@Module
@ComponentScan
class AppModule

@Single
class UserRepository

@Factory
class UserService(private val userRepository: UserRepository)
```

### Generated Module

The KSP processor automatically generates a Koin module for you. You can include it in your application setup:

```kotlin
fun main() {
    startKoin {
        modules(AppModuleModule().module)
    }
}
```

This eliminates the need to manually write the Koin module, saving time and reducing boilerplate.

---

## Example 2: KSP with Room Database

Room is a widely-used ORM for Android. With KSP, Room processes annotations faster, reducing build times significantly.

### Setup Room with KSP

Add the following dependencies:

```kotlin
dependencies {
    implementation("androidx.room:room-runtime:<version>")
    ksp("androidx.room:room-compiler:<version>")
}
```

### Annotate Entities

```kotlin
@Entity
data class User(
    @PrimaryKey val id: Int,
    val name: String
)
```

### Generate DAO and Database

```kotlin
@Dao
interface UserDao {
    @Query("SELECT * FROM User")
    fun getAllUsers(): List<User>
}

@Database(entities = [User::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}
```

Using KSP, Room generates the necessary code behind the scenes, reducing boilerplate.

---

## How to Create a Custom KSP Processor

Let’s build a custom KSP processor that generates a `Builder` class for data classes annotated with `@GenerateBuilder`. This is a practical and commonly useful feature for many projects.

### Create the Module

First, you should create a module with the API for KSP.

```kotlin
dependencies {
    implementation("com.google.devtools.ksp:symbol-processing-api:<version>")
}
```

### Define the Annotation

```kotlin
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.SOURCE)
annotation class GenerateBuilder
```

### KSP Processor Logic

The processor can dynamically generate a `Builder` class based on the properties of the annotated data class. You need to create a class extending `SymbolProcessor` where all the work will be done in the `process` function, and a class extending `SymbolProcessorProvider`, which will provide the implementation of the `SymbolProcessor`.

Here is the `SymbolProcessor` implementation:

```kotlin
class KspBuilderProcessor(
    private val codeGenerator: CodeGenerator
) : SymbolProcessor {
    override fun process(resolver: Resolver): List<KSAnnotated> {
        val symbols = resolver.getSymbolsWithAnnotation(GenerateBuilder::class.qualifiedName.toString())
            .filterIsInstance<KSClassDeclaration>()

        symbols.forEach { symbol ->
            val className = symbol.simpleName.asString()
            val packageName = symbol.packageName.asString()
            val generatedClassName = "${className}Builder"

            val file = codeGenerator.createNewFile(
                dependencies = Dependencies(false, symbol.containingFile!!),
                packageName = packageName,
                fileName = generatedClassName
            )

            val properties = symbol.getAllProperties()

            val builderProperties = mutableListOf<String>()
            val setters = mutableListOf<String>()
            val buildMethodParams = mutableListOf<String>()

            properties.forEach { property ->
                val propName = property.simpleName.asString()
                val propType = property.type.resolve().declaration.simpleName.asString()
                    .let { if (property.type.resolve().isMarkedNullable) "$it?" else it }
                val defaultValue = getDefaultValueFromProperty(property)
                builderProperties.add("   private var $propName: $propType = $defaultValue")
                setters.add("   fun set${propName.replaceFirstChar { it.uppercase() }}($propName: $propType) = apply { this.$propName = $propName }")
                buildMethodParams.add("           $propName = this.$propName")
            }

            val builderClass = buildString {
                appendLine("package $packageName")
                appendLine()
                appendLine("class $generatedClassName {")
                builderProperties.forEach { property ->
                    appendLine(property)
                }
                appendLine()
                setters.forEach { setter ->
                    appendLine(setter)
                }
                appendLine()
                appendLine("    fun build(): $className {")
                appendLine("        return $className(")
                buildMethodParams.forEach { methodParam ->
                    appendLine(methodParam)
                }
                appendLine("        )")
                appendLine("    }")
                appendLine("}")
                appendLine()
                appendLine("fun ${generatedClassName.replaceFirstChar { it.lowercase() }}(block: $generatedClassName.() -> Unit): $className {")
                appendLine("    return $generatedClassName().apply(block).build()")
                appendLine("}")
            }
            file.write(builderClass.toByteArray())
            file.close()
        }
        return symbols.filterNot { it.validate() }.toList()
    }

    private fun getDefaultValueFromProperty(property: KSPropertyDeclaration): String {
        val propType = property.type.resolve().declaration.qualifiedName?.asString() ?: "Any"
        val isNullable = property.type.resolve().isMarkedNullable

        return if (isNullable) "null" else when (propType) {
            "kotlin.String" -> "\"\""
            "kotlin.Int", "kotlin.Long", "kotlin.Short", "kotlin.Byte" -> "0"
            "kotlin.Double", "kotlin.Float" -> "0.0"
            "kotlin.Boolean" -> "false"
            else -> property.findDefaultValue() ?: "throw IllegalArgumentException(\"Non-nullable type \$propType requires a default value\")"
        }
    }

    private fun KSPropertyDeclaration.findDefaultValue(): String? {
        // Implement logic to parse and retrieve default values from annotations or source code
        return null
    }
}
```

And here is the `SymbolProcessorProvider`:

```kotlin
class KspBuilderProvider : SymbolProcessorProvider {
    override fun create(environment: SymbolProcessorEnvironment): SymbolProcessor {
        return KspBuilderProcessor(environment.codeGenerator)
    }
}
```

With these two classes created, you need to create a file with the path `src/main/resources/META-INF/services` and name `com.google.devtools.ksp.processing.SymbolProcessorProvider`. Its content will be the full name of the `SymbolProcessorProvider` class you just created. In this case, it is:

```
com.example.kspbuilder.KspBuilderProvider
```

---

## Use the Custom KSP Processor

### Add Custom Processor

Add the KSP plugin to the `build.gradle.kts` file on the module where you want to use the annotation:

```kotlin
plugins {
    id("com.google.devtools.ksp") version "<version>"
}

dependencies {
    implementation(project(":KspBuilder"))
    ksp(project(":KspBuilder"))
}
```

### Annotate a Class

Create a data class with the custom annotation:

```kotlin
@GenerateBuilder
class Person(val id: Int, val name: String, val age: Int, val address: Address?)

@GenerateBuilder
class Address(val id: Int, val name: String, val country: String)
```

### Generated Output

After building the project, the generated code with KSP will be located under the `build/generated/ksp` folder.

For the `Person` data class, the generated builder class would look like this:

```kotlin
class PersonBuilder {
   private var id: Int = 0
   private var name: String = ""
   private var age: Int = 0
   private var address: Address? = null

   fun setId(id: Int) = apply { this.id = id }
   fun setName(name: String) = apply { this.name = name }
   fun setAge(age: Int) = apply { this.age = age }
   fun setAddress(address: Address?) = apply { this.address = address }

    fun build(): Person {
        return Person(
           id = this.id,
           name = this.name,
           age = this.age,
           address = this.address
        )
    }
}

fun personBuilder(block: PersonBuilder.() -> Unit): Person {
    return PersonBuilder().apply(block).build()
}
```

For the `Address` data class, the generated builder class would look like this:

```kotlin
class AddressBuilder {
   private var id: Int = 0
   private var name: String = ""
   private var country: String = ""

   fun setId(id: Int) = apply { this.id = id }
   fun setName(name: String) = apply { this.name = name }
   fun setCountry(country: String) = apply { this.country = country }

    fun build(): Address {
        return Address(
           id = this.id,
           name = this.name,
           country = this.country
        )
    }
}

fun addressBuilder(block: AddressBuilder.() -> Unit): Address {
    return AddressBuilder().apply(block).build()
}
```

### Usage Example

```kotlin
val person = personBuilder {
      setId(10)
      setName("Test")
      setAge(100)
      setAddress(
          addressBuilder {
              setId(10)
              setName("AddressTest")
              setCountry("Spain")
          }
      )
}
```

---

## Conclusion

Kotlin Symbol Processing is a game-changer for Kotlin developers. Its lightweight and Kotlin-first design makes it a perfect replacement for `kapt`, and its ability to generate code dynamically opens up new possibilities. Whether you’re using KSP with established libraries like Koin and Room or building custom processors for your use case, KSP provides the tools you need to take your development to the next level.

Try integrating KSP into your project and see the performance benefits firsthand! 

Here is the repository with the code for the custom KSP processor [Github Repo](https://github.com/IgnacioCarrionN/KspTest)

