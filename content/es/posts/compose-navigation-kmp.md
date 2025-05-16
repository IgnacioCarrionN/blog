---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Implementando Navegación en Proyectos Compose Multiplatform"
date: 2025-05-16T08:00:00+01:00
description: "Una guía completa para implementar org.jetbrains.androidx.navigation en proyectos Compose Multiplatform, con enfoque en el soporte estable para iOS en Compose Multiplatform 1.8.0."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/compose-navigation-kmp.png
draft: false
tags: 
- kotlin
- multiplatform
- kmp
- compose
- navigation
---

### Implementando Navegación en Proyectos Compose Multiplatform

Con la última versión de Compose Multiplatform (1.8.0), el soporte para iOS ha sido declarado estable, marcando un hito significativo para el desarrollo de interfaces de usuario multiplataforma con Kotlin. Uno de los componentes clave para construir aplicaciones robustas es la navegación, y la biblioteca org.jetbrains.androidx.navigation proporciona una solución potente que puede integrarse en proyectos Compose Multiplatform. Este artículo explora cómo implementar la navegación en un entorno Compose Multiplatform, permitiéndote compartir la lógica de navegación entre las plataformas Android e iOS.

---

#### Entendiendo la Navegación en Compose Multiplatform

La navegación en Compose Multiplatform sigue los mismos principios que la navegación en Jetpack Compose para Android, pero con adaptaciones para funcionar en múltiples plataformas. La arquitectura de navegación consiste en:

1. **NavController**: La API central para la navegación que mantiene un registro de las entradas en la pila de retroceso y maneja las operaciones de navegación
2. **NavHost**: Un componible que muestra el destino actual de un NavController
3. **NavGraph**: Una colección de destinos que define posibles rutas de navegación

En un contexto de Compose Multiplatform, la navegación:

1. Permite compartir la lógica de navegación entre plataformas
2. Admite requisitos de navegación específicos de cada plataforma cuando es necesario

Este enfoque nos permite definir nuestra estructura de navegación en código común, mientras acomodamos comportamientos específicos de plataforma cuando sea necesario.

```kotlin
// En commonMain - Configuración básica de navegación
@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(navController = navController, startDestination = "home") {
        composable("home") {
            HomeScreen(
                onNavigateToDetails = { itemId ->
                    navController.navigate("details/$itemId")
                }
            )
        }
        composable(
            route = "details/{itemId}",
            arguments = listOf(navArgument("itemId") { type = NavType.StringType })
        ) { backStackEntry ->
            val itemId = backStackEntry.arguments?.getString("itemId") ?: ""
            DetailsScreen(
                itemId = itemId,
                onNavigateBack = {
                    navController.popBackStack()
                }
            )
        }
    }
}
```

---

#### Configurando la Navegación en un Proyecto Compose Multiplatform

Para integrar la navegación en tu proyecto Compose Multiplatform, sigue estos pasos:

##### 1. Configurar el archivo build.gradle.kts en tu módulo compartido

```kotlin
plugins {
    kotlin("multiplatform")
    id("com.android.library")
    id("org.jetbrains.compose")
}

kotlin {
    androidTarget()
    iosX64()
    iosArm64()
    iosSimulatorArm64()

    sourceSets {
        val commonMain by getting {
            dependencies {
                // Compose Multiplatform
                implementation(compose.runtime)
                implementation(compose.foundation)
                implementation(compose.material)

                // Navegación para Compose
                implementation("org.jetbrains.androidx.navigation:navigation-compose:2.8.0")
            }
        }
    }
}
```

##### 2. Crear un NavHost compartido en código común

```kotlin
// En commonMain/kotlin/navigation/AppNavigation.kt

@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(
        navController = navController,
        startDestination = "home"
    ) {
        // Define tu grafo de navegación aquí
        composable("home") {
            HomeScreen(
                onNavigateToProfile = {
                    navController.navigate("profile")
                },
                onNavigateToSettings = {
                    navController.navigate("settings")
                }
            )
        }

        composable("profile") {
            ProfileScreen(
                onNavigateBack = {
                    navController.popBackStack()
                }
            )
        }

        composable("settings") {
            SettingsScreen(
                onNavigateBack = {
                    navController.popBackStack()
                }
            )
        }
    }
}
```

---

#### Consideraciones Específicas por Plataforma

##### Implementación en Android

En Android, la navegación se integra perfectamente con el ciclo de vida de la plataforma:

```kotlin
// En androidMain/kotlin/MainActivity.kt

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        setContent {
            AppTheme {
                AppNavigation()
            }
        }
    }
}
```

##### Implementación en iOS

Con Compose Multiplatform 1.8.0, el soporte para iOS ahora es estable, haciendo que la implementación de navegación sea más confiable:

```kotlin
// En iosMain/kotlin/MainViewController.kt

fun MainViewController() = ComposeUIViewController {
    AppTheme {
        AppNavigation()
    }
}
```

---

#### Técnicas Avanzadas de Navegación

##### 1. Grafos de Navegación Anidados

Para aplicaciones más complejas, puedes organizar tu navegación con grafos anidados:

```kotlin
@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(navController = navController, startDestination = "home") {
        composable("home") { HomeScreen(navController) }

        // Grafo de navegación anidado para el flujo de autenticación
        navigation(startDestination = "login", route = "auth") {
            composable("login") { LoginScreen(navController) }
            composable("register") { RegisterScreen(navController) }
            composable("forgot_password") { ForgotPasswordScreen(navController) }
        }

        // Grafo de navegación anidado para configuraciones
        navigation(startDestination = "settings_main", route = "settings") {
            composable("settings_main") { SettingsMainScreen(navController) }
            composable("appearance") { AppearanceSettingsScreen(navController) }
            composable("notifications") { NotificationSettingsScreen(navController) }
        }
    }
}
```

##### 2. Pasando Argumentos Entre Destinos

```kotlin
// Definir rutas con argumentos
private object Routes {
    const val ITEM_DETAILS = "item_details/{itemId}"

    fun itemDetails(itemId: String) = "item_details/$itemId"
}

// En tu NavHost
NavHost(navController = navController, startDestination = "items_list") {
    composable("items_list") {
        ItemsListScreen(
            onItemClick = { itemId ->
                navController.navigate(Routes.itemDetails(itemId))
            }
        )
    }

    composable(
        route = Routes.ITEM_DETAILS,
        arguments = listOf(navArgument("itemId") { type = NavType.StringType })
    ) { backStackEntry ->
        val itemId = backStackEntry.arguments?.getString("itemId") ?: ""
        ItemDetailsScreen(itemId = itemId)
    }
}
```

##### 3. Manejando Enlaces Profundos (Deep Links)

```kotlin
NavHost(
    navController = navController,
    startDestination = "home"
) {
    composable(
        route = "home",
        deepLinks = listOf(
            navDeepLink { uriPattern = "https://example.com/home" }
        )
    ) {
        HomeScreen()
    }

    composable(
        route = "product/{productId}",
        arguments = listOf(navArgument("productId") { type = NavType.StringType }),
        deepLinks = listOf(
            navDeepLink { 
                uriPattern = "https://example.com/product/{productId}" 
            }
        )
    ) { backStackEntry ->
        val productId = backStackEntry.arguments?.getString("productId") ?: ""
        ProductScreen(productId = productId)
    }
}
```

##### 4. Seguridad de Tipos con Kotlin DSL (Navigation 2.8.0+)

A partir de Navigation 2.8.0, puedes usar un DSL de Kotlin con seguridad de tipos para definir tu grafo de navegación utilizando anotaciones @Serializable y clases de datos, lo que proporciona seguridad en tiempo de compilación para rutas y argumentos sin necesidad de definir rutas como cadenas de texto:

```kotlin
// Define tus destinos con argumentos tipados
@Serializable
object Home

// Define una ruta de perfil que toma un ID
@Serializable
data class Profile(val id: String)

// Crea un grafo de navegación con seguridad de tipos
@Composable
fun TypeSafeNavigation() {
    val navController = rememberNavController()

    NavHost(navController = navController, startDestination = Home) {
        composable<Home> {
            HomeScreen(onNavigateToProfile = { id ->
                navController.navigate(Profile(id))
            })
        }

        composable<Profile> { backStackEntry ->
            val profile: Profile = backStackEntry.toRoute()
            ProfileScreen(profile.id)
        }
    }
}

// Navega con seguridad de tipos
navController.navigate(Profile(id = "123"))
```

Beneficios de usar el DSL con seguridad de tipos y @Serializable:

1. **Sin rutas de texto**: No es necesario definir cadenas de ruta o plantillas de cadenas
2. **Completamente tipado**: Los argumentos son parte de la estructura de datos de la ruta
3. **Seguridad en tiempo de compilación**: Los errores tipográficos y de tipo se detectan durante la compilación
4. **Acceso directo a objetos**: Accede a los parámetros de ruta directamente como propiedades del objeto de ruta
5. **Navegación simplificada**: Navega pasando objetos directamente a la función navigate

Para habilitar el DSL con seguridad de tipos, asegúrate de usar Navigation 2.8.0 o superior:

```kotlin
implementation("org.jetbrains.androidx.navigation:navigation-compose:2.8.0")
```

---

#### Mejores Prácticas para Navegación en Compose Multiplatform

1. **Crear una capa de abstracción de navegación**
   - Definir una interfaz común para acciones de navegación
   - Implementar detalles específicos de plataforma detrás de esta interfaz
   - Esto facilita el manejo de diferencias entre plataformas

2. **Usar navegación con seguridad de tipos con @Serializable**
   - Aprovechar el DSL con seguridad de tipos y anotaciones @Serializable en lugar de rutas basadas en cadenas de texto
   - Definir destinos como clases de datos y objetos para seguridad en tiempo de compilación
   - Acceder a los parámetros de ruta directamente como propiedades de objetos de ruta
   - Esto elimina las rutas basadas en texto, previene errores tipográficos y facilita la refactorización

3. **Gestionar el estado de navegación adecuadamente**
   - Considerar usar una solución de gestión de estado como ViewModel
   - Mantener la lógica de navegación separada de la lógica de UI
   - Esto hace que tu código sea más testeable y mantenible

4. **Probar tus flujos de navegación**
   - Escribir pruebas para tu lógica de navegación
   - Verificar que los argumentos se pasen correctamente
   - Asegurar que los enlaces profundos funcionen según lo esperado

---

### Conclusión

Con Compose Multiplatform 1.8.0 declarando el soporte para iOS como estable, implementar navegación en aplicaciones multiplataforma se ha vuelto más confiable y sencillo. El enfoque descrito en este artículo proporciona una forma práctica de compartir lógica de navegación entre plataformas mientras se acomodan requisitos específicos de cada plataforma.

Siguiendo los pasos de configuración, consideraciones específicas de plataforma y mejores prácticas descritas en este artículo, puedes implementar con éxito la navegación en tus proyectos Compose Multiplatform y crear experiencias de usuario consistentes e intuitivas en las plataformas Android e iOS.

Para información más detallada, consulta la [documentación oficial de navegación en Compose Multiplatform](https://www.jetbrains.com/help/kotlin-multiplatform-dev/compose-navigation.html).
