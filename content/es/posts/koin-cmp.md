---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Usando Koin en Compose Multiplatform"
date: 2025-01-02T08:00:00+01:00
description: "Usando Koin en Compose Multiplatform desde el código común con posibilidad de configurar cada una de las plataformas."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/koin-cmp.png
draft: false
tags: 
- kotlin
- multiplatform
- cmp
- compose
- koin
---

# Usando Koin en Compose Multiplatform

La inyección de dependencias es algo imprescindible para crear aplicaciones escalables, y *Koin* hace que sea muy sencillo, incluso en proyectos con Compose Multiplatform. Con la nueva función composable `KoinApplication`, puedes inicializar Koin directamente desde el código común, reduciendo la cantidad de código necesario mientras se mantiene la flexibilidad de configurar cada plataforma por separado. Vamos a ver un ejemplo.

---

## Project Setup

Empieza creando un proyecto de Compose Multiplatform usando el [KMP Wizard](https://kmp.jetbrains.com/), seleccionando Android, iOS, Desktop y Web como plataformas. Para este ejemplo no vamos a incluir Server como plataforma.

---

## Añadiendo las dependencias

Usa el version catalog de Gradle para incluir las dependencias necesarias de Koin en `libs.versions.toml`:

```toml
[versions]
koin-bom = "4.1.0-Beta1"

[libraries]
koin-bom = { module = "io.insert-koin:koin-bom", version.ref = "koin-bom" }
koin-core = { module = "io.insert-koin:koin-core" }
koin-android = { module = "io.insert-koin:koin-android" }
koin-compose = { module = "io.insert-koin:koin-compose" }
koin-compose-viewModel = { module = "io.insert-koin:koin-compose-viewmodel" }
```

---

## Definiendo los módulos de Koin

Vamos a crear dos módulos de Koin: `appModule` y `platformModule`. El `platformModule` define las dependencias específicas de cada plataforma.

### Módulos compartidos

```kotlin
val appModule = module {
    viewModelOf(::MainViewModel)
    factoryOf(::GetJokeUseCase)
    singleOf(::DefaultJokeRepository) bind JokeRepository::class
    singleOf(::JokeJsonDataSource) bind JokeDataSource::class
    single { Json { ignoreUnknownKeys = true } }
}

val Module.localPreferencesDefinition
    get() = singleOf(::InMemoryLocalPreferences) bind LocalPreferences::class

expect val platformModule: Module
```

### Módulos específicos de cada plataforma

Para Android vamos a usar una implementación de la interfaz de LocalPreferences que depende del contexto de Android por lo que necesitamos un módulo distinto al resto de plataformas:

```kotlin
actual val platformModule: Module
    get() = module {
        singleOf(::AndroidPreferences) bind LocalPreferences::class
    }
```

Para iOS, Desktop y Web, reutilizaremos la `localPreferencesDefinition` que se puede ver más arriba:

```kotlin
actual val platformModule: Module
    get() = module {
        localPreferencesDefinition
    }
```

---

## Configurando la App

En el archivo `App.kt`, podemos usar la función composable `KoinApplication`. Añadiendo el parámetro `KoinAppDeclaration` como opcional y con valor por defecto a null.

```kotlin
@Composable
@Preview
fun App(koinAppDeclaration: KoinAppDeclaration? = null) {
    KoinApplication(
        application = {
            koinAppDeclaration?.invoke(this)
            modules(appModule, platformModule)
        }
    ) {
        MaterialTheme {
            MainScreen()
        }
    }
}
```

En Android, usamos la lambda para proveer el contexto y activar el logging:

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            App {
                androidLogger(Level.DEBUG)
                androidContext(this@MainActivity)
            }
        }
    }
}
```

Esta flexibilidad nos asegura configuraciones específicas por plataforma, como inyectar el contexto de Android, sin afectar al resto de plataformas.

---

## Corriendo la App

Compila la aplicación en cada plataforma. Podrás probar que todo está funcionando y cada plataforma recibe la configuración que necesita para funcionar.

## Conclusión

La nueva función composable `KoinApplication` simplifica la inyección de dependencias en Compose Multiplatform permitiendo inizializar Koin de forma compartida manteniendo la posibilidad de configurar cada plataforma por separado si fuera necesario. Esta forma de proceder reduce el código necesario y promueve la reusabilidad del código entre plataformas.

Puedes descargar el código completo para este ejemplo en [GitHub](https://github.com/IgnacioCarrionN/KMPKoin).

También si necesitas más información acerca de las diferentes opciones para declarar dependencias en *Koin* puedes visitar un post que publiqué en LinkedIn: [Koin DSL](https://www.linkedin.com/posts/nacho-carrion_koin-declaring-dependencies-activity-7279802828687101953-W3rs?utm_source=share&utm_medium=member_desktop)
