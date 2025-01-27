---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Mejorando el Rendimiento de las Aplicaciones Android con Baseline Profiles"
date: 2025-01-27T08:00:00+01:00
description: "Mejorando el Rendimiento de las Aplicaciones Android con Baseline Profiles con un ejemplo real"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/baseline-profile.png
draft: false
tags:
- kotlin
- architecture
---


# **Mejorando el Rendimiento de las Aplicaciones Android con Baseline Profiles**

## **Introducción**

En el mundo móvil actual, los usuarios esperan que las aplicaciones se inicien al instante y funcionen sin problemas. La optimización del rendimiento es crucial, especialmente en lo que respecta al tiempo de inicio y la ejecución en tiempo de ejecución.

Los **Baseline Profiles** de Android ofrecen una forma eficaz de acelerar el inicio de la aplicación y mejorar el rendimiento en tiempo de ejecución al precompilar rutas de código críticas. Google Play incluso recomienda el uso de Baseline Profiles para mejorar la experiencia del usuario, especialmente en aplicaciones con una renderización de UI compleja o dependencias pesadas.

En esta guía, exploraremos qué son los Baseline Profiles, cómo funcionan y cómo integrarlos en tu aplicación Android utilizando un **módulo dedicado de Baseline Profile** para lograr tiempos de inicio más rápidos y un mejor rendimiento en tiempo de ejecución.

---

## **¿Qué son los Baseline Profiles?**

Las aplicaciones de Android utilizan el **Android Runtime (ART)** para ejecutar código, empleando **Just-In-Time (JIT)** y **Ahead-Of-Time (AOT)** compilation.

Sin embargo, la compilación JIT puede introducir retrasos durante el inicio de la aplicación, ya que el código no está completamente optimizado inicialmente. **Baseline Profiles** resuelven este problema al permitir que los desarrolladores especifiquen rutas de código críticas que deben precompilarse antes de ejecutar la aplicación.

### **¿Cómo funcionan los Baseline Profiles?**

- **Precompilación**: Contienen definiciones de métodos y clases que se precompilan y optimizan antes de la ejecución.
- **Instalación**: Estos perfiles se instalan en el dispositivo del usuario en el primer inicio de la aplicación.
- **Optimización**: ART los utiliza para mejorar la velocidad de ejecución, especialmente durante los inicios en frío.

---

## **¿Cuándo deberías usar Baseline Profiles?**

Los Baseline Profiles son especialmente beneficiosos en los siguientes escenarios:

✅ **Optimización del Inicio de la Aplicación** – Reducción del tiempo de inicio en frío mediante la precompilación de secuencias de lanzamiento.  
✅ **Mejora del Rendimiento del Desplazamiento** – Garantiza una renderización de UI más fluida.  
✅ **Optimización de Funcionalidades Frecuentemente Usadas** – Precompilación de la lógica con la que los usuarios interactúan con frecuencia.

---

## **Configuración de Baseline Profiles en un Módulo Separado**

### **Paso 1: Crear el Módulo de Baseline Profile**

1. **Abrir Android Studio**:
    - Navegar a `File` → `New` → `New Module`.
    - Seleccionar la plantilla `Baseline Profile Generator`.

2. **Configurar el Módulo**:
    - **Aplicación Objetivo**: Elegir el módulo de la aplicación para el cual se generará el Baseline Profile.
    - **Nombre del Módulo**: Asignar un nombre, por ejemplo, `baselineprofile`.
    - **Nombre del Paquete**: Definir el nombre del paquete para el módulo.
    - **Lenguaje**: Seleccionar Kotlin o Java.
    - **Lenguaje de Configuración de Build**: Elegir entre Kotlin Script (KTS) o Groovy.

3. **Finalizar**: Hacer clic en `Finish` para crear el módulo.

Este proceso configura un nuevo módulo que contiene las configuraciones y el código necesarios para generar y evaluar Baseline Profiles.

---

### **Paso 2: Definir el Generador de Baseline Profile**

#### **Generador de Baseline Profile Basado en Jetpack Compose**

```kotlin
@RunWith(AndroidJUnit4::class)
@LargeTest
class BaselineProfileGenerator {

    @get:Rule
    val baselineProfileRule = BaselineProfileRule()

    @get:Rule
    val composeTestRule = createAndroidComposeRule<MainActivity>()

    @Test
    fun baselineProfile() = baselineProfileRule.collect(
        packageName = "com.example.app",
        includeInStartupProfile = true,
        profileBlock = {

            // 1. **Inicio en Frío de la Aplicación**
            startActivityAndWait()

            // 2. **Navegar a la Pantalla Principal**
            composeTestRule.onNodeWithText("Get Started")
                .performClick()
            composeTestRule.waitForIdle()

            // 3. **Desplazarse a través de una Lista LazyColumn**
            composeTestRule.onNodeWithTag("itemList")
                .performScrollToIndex(10)
            composeTestRule.waitForIdle()

            // 4. **Abrir una Pantalla de Detalle**
            composeTestRule.onNodeWithTag("item_0")
                .performClick()
            composeTestRule.waitForIdle()

            // 5. **Realizar una Búsqueda**
            composeTestRule.onNodeWithTag("searchInput")
                .performTextInput("Kotlin")
            composeTestRule.waitForIdle()

            // 6. **Navegación hacia atrás**
            composeTestRule.onNodeWithContentDescription("Back")
                .performClick()
            composeTestRule.waitForIdle()
        }
    )
}
```

---

### **Paso 3: Generar e Integrar el Baseline Profile**

1. **Generar el Baseline Profile**:
    - Ejecutar la configuración `Generate Baseline Profile` creada por la plantilla.
    - Alternativamente, ejecutar la siguiente tarea de Gradle:

      ```shell
      ./gradlew :app:generateBaselineProfile
      ```  

   Esto genera el Baseline Profile y lo copia en el directorio apropiado dentro del módulo de la aplicación.

2. **Integrar el Baseline Profile**:
    - El perfil generado se incluye automáticamente en los assets del módulo de la aplicación.
    - Asegurar que el archivo `build.gradle` del módulo de la aplicación incluya las dependencias necesarias:

      ```kotlin
      dependencies {
          implementation("androidx.profileinstaller:profileinstaller:1.3.0")
          baselineProfile(project(":baselineprofile"))
      }
      ```

   La biblioteca `profileinstaller` instala el Baseline Profile en los dispositivos de los usuarios, y la dependencia `baselineProfile` vincula el perfil generado a la aplicación.

---

## **Conclusión**

Los Baseline Profiles son una herramienta poderosa para mejorar el **tiempo de inicio y el rendimiento en tiempo de ejecución** sin aumentar el tamaño del APK. Al precompilar rutas de código críticas, tu aplicación se iniciará más rápido, brindando una **experiencia más fluida y receptiva** a los usuarios.

Al integrar Baseline Profiles mediante un **módulo dedicado**, garantizas un enfoque modular, mantenible y escalable para la optimización del rendimiento.

Si aún no lo has hecho, **comienza a usar Baseline Profiles hoy mismo** y mide su impacto en el rendimiento de tu aplicación.
