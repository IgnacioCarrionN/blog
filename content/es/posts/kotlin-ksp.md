---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Explorando Kotlin KSP"
date: 2024-12-27T08:00:00+01:00
description: "Kotlin Avanzado - Kotlin Symbol Processing (KSP)"
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

# Explorando Kotlin Symbol Processing (KSP) con ejemplos prácticos

Kotlin Symbol Processing (KSP) es una herramienta muy potente usada para simplificar el procesamiento de anotaciones en Kotlin. Comparado con `kapt` (Kotlin Annotation Processing Tool), KSP es más rápido, ofrece mejor integración con Kotlin y reduce los tiempos de compilación de forma significativa. En este post, exploraremos los fundamentos de KSP, discutiremos cómo funciona y mostraremos como su uso en librerías populares como Koin y Room.

---

## Qué es KSP?

KSP es una API ligera y eficiente para procesar código Kotlin. Permite crear procesadores de anotaciones que funcionan directamente con la sintaxis de Kotlin en lugar de depender de herramientas basadas en Java. Esto lo convierte en una opción ideal para proyectos orientados a Kotlin.

### Beneficios de KSP:

1. **Velocidad**: Procesa código Kotlin más rápido que `kapt`.
2. **Diseño centrado en Kotlin**: Funciona directamente con los constructos del lenguaje Kotlin, evitando abstracciones basadas en Java.
3. **Ligero**: Reduce el código repetitivo y se integra perfectamente con Gradle.
4. **Compatibilidad**: Muchas bibliotecas populares ahora son compatibles con KSP de manera nativa.

---

## Setting Up KSP in Your Project

Agrega el plugin de KSP a tu proyecto

### Configuración de Gradle

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

Reemplaza `<ksp-processor-library>` con la dependencia del procesador específico de la biblioteca, como se muestra en los ejemplos a continuación.

---

## Ejemplo 1: KSP con las anotaciones de Koin

Koin desde la versión 3.4.0, permite definir dependencias a través de anotaciones, que luego son procesadas usando KSP para generar los módulos de Koin.

### Configuración de Koin con KSP

Añade las siguientes dependencias:

```kotlin
dependencies {
    implementation("io.insert-koin:koin-core:<version>")
    implementation("io.insert-koin:koin-annotations:<version>")
    ksp("io.insert-koin:koin-ksp-compiler:<version>")
}
```

### Anota las clases

Usa las anotaciones de Koin para definir tu grafo de dependencias:

```kotlin
@Module
@ComponentScan
class AppModule

@Single
class UserRepository

@Factory
class UserService(private val userRepository: UserRepository)
```

### Módulo generado

El procesador de KSP genera automáticamente un módulo de Koin. Puedes incluirlo en la configuración de tu aplicación:

```kotlin
fun main() {
    startKoin {
        modules(AppModuleModule().module)
    }
}
```

Esto elimina la necesidad de escribir manualmente el módulo de Koin, ahorrando tiempo y reduciendo el código repetitivo.

---

## Example 2: KSP con base de datos Room

Room es un ORM ampliamente utilizado para Android. Con KSP, Room procesa anotaciones más rápidamente, reduciendo significativamente los tiempos de compilación

### Configuración de Room con KSP

Agrega las siguientes dependencias:

```kotlin
dependencies {
    implementation("androidx.room:room-runtime:<version>")
    ksp("androidx.room:room-compiler:<version>")
}
```

### Anota las entidades

```kotlin
@Entity
data class User(
    @PrimaryKey val id: Int,
    val name: String
)
```

### Generar DAO y Base de Datos

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

Usando KSP, Room genera el código necesario de forma automática, reduciendo el código repetitivo.

---

## Como crear un procesador KSP personalizado

Construyamos un procesador KSP personalizado que genere una clase `Builder` para clases de datos anotadas con `@GenerateBuilder`.

### Crear el módulo

Primero, debes crear un módulo con la API para KSP.

```kotlin
dependencies {
    implementation("com.google.devtools.ksp:symbol-processing-api:<version>")
}
```

### Definir la anotación

```kotlin
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.SOURCE)
annotation class GenerateBuilder
```

### Lógica del procesador KSP

El procesador puede generar dinámicamente una clase `Builder` basada en las propiedades de la data class con la anotación. Necesitas crear una clase que extienda `SymbolProcessor` donde todo el trabajo se realizará en la función `process`, y una clase extendiendo `SymbolProcessorProvider`, que proveerá de la implementación del `SymbolProcessor`.

Aquí la implementación de `SymbolProcessor`:

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
            else -> throw IllegalArgumentException("Non-nullable type $propType requires a default value")
        }
    }
}
```

Y aquí la clase que extiende de `SymbolProcessorProvider`:

```kotlin
class KspBuilderProvider : SymbolProcessorProvider {
    override fun create(environment: SymbolProcessorEnvironment): SymbolProcessor {
        return KspBuilderProcessor(environment.codeGenerator)
    }
}
```

Con estas dos clases ya solo falta crear un fichero con ruta `src/main/resources/META-INF/services` y nombre `com.google.devtools.ksp.processing.SymbolProcessorProvider`. Su contenido será el nombre completo de la clase que extiende de `SymbolProcessorProvider` que acabas de crear. En este caso quedaría así:

```
com.example.kspbuilder.KspBuilderProvider
```

---

## Usando el procesador KSP personalizado

### Agregar el procesador personalizado

Añade el plugin KSP al fichero `build.gradle.kts` en el módulo donde quieres utilizar la anotación:

```kotlin
plugins {
    id("com.google.devtools.ksp") version "<version>"
}

dependencies {
    implementation(project(":KspBuilder"))
    ksp(project(":KspBuilder"))
}
```

### Anotar la clase

Crea una data class con la anotación:

```kotlin
@GenerateBuilder
class Person(val id: Int, val name: String, val age: Int, val address: Address?)

@GenerateBuilder
class Address(val id: Int, val name: String, val country: String)
```

### Código generado

Después de compilar el proyecto, el código generado con KSP se localiza en el directorio `build/generated/ksp`.

Para la data class `Person`, la clase builder generada se ve así:

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

Para la data class `Address`, la clase builder generada sería así:

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

### Ejemplo de uso

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

## Conclusión

KSp es una herramienta muy importante para los desarrolladores Kotlin. Su diseño ligero y centrado en Kotlin hace que sea un reemplazo perfecto de `kapt`, su habilidad para generar código dinámicamente abre un gran abanico de posibilidades. Tanto si usas KSP con librerías como Koin y Room o creas tu propio procesador para tu caso de uso, KSP brinda las herramientas necesarias para elevar tu desarrollo al siguiente nivel.

Intenta integrar KSP en tu próximo proyecto y observa los beneficios de primera mano!

Aquí dejo el repositorio con el código utilizado para crear el procesador KSP personalizado [Github Repo](https://github.com/IgnacioCarrionN/KspTest)
