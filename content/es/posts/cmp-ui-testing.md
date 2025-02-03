---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Tests en Compose Multiplatform (CMP) desde Código Común"
date: 2025-02-03T08:00:00+01:00
description: "Aprende como escribir y correr tests de UI en Compose Multiplatform (CMP) desde el código común, consiguiendo consistencia entre plataformas para Android, iOS y Desktop."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/compose-test.png
draft: false
tags:
- kotlin
- compose
- cmp
- multiplatform
---

# **Tests en Compose Multiplatform (CMP) desde Código Común**

Compose Multiplatform (CMP) permite construir UI para múltiples plataformas utilizando Jetpack Compose. Afortunadamente, CMP también admite **escribir y ejecutar tests de UI en el código común**, lo que hace que los test sean más eficientes en todas las plataformas. En esta publicación, exploraremos cómo probar aplicaciones CMP utilizando `compose.uiTest` y ejecutarlas en Android, Desktop e iOS.

---  

## **1. Configuración de Test de UI Comunes**

CMP proporciona `compose.uiTest`, lo que permite escribir test de UI en el módulo compartido sin depender de plataformas específicas. Esto significa que puedes **escribir una vez y probar en todas partes**.

### **Actualización de la Configuración del Proyecto**

Para habilitar los tests, actualiza tu archivo `build.gradle.kts` en el módulo compartido:

```kotlin
kotlin {
    androidTarget {
        instrumentedTestVariant.sourceSetTree.set(KotlinSourceSetTree.test)
    }

    sourceSets {
        val commonTest by getting {
            dependencies {
                implementation(kotlin("test"))
                @OptIn(org.jetbrains.compose.ExperimentalComposeLibrary::class)
                implementation(compose.uiTest)
            }
        }
        val desktopTest by getting {
            dependencies {
                implementation(compose.desktop.currentOs)
            }
        }
    }
}
```

### **Declaración de Dependencias para Tests de UI en Android**

En el nivel raíz de tu archivo `build.gradle.kts`, agrega las dependencias necesarias para los tests en Android:

```kotlin
dependencies {
    androidTestImplementation("androidx.compose.ui:ui-test-junit4-android:1.5.4")
    debugImplementation("androidx.compose.ui:ui-test-manifest:1.5.4")
}
```

### **Configuración Específica para Tests en Android**

Para **tests instrumentadas en Android**, agrega lo siguiente a tu `build.gradle.kts`:

```kotlin
android {
    defaultConfig {
        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }
}
```

---  

## **2. Implementación de una UI de Contador Simple**

Vamos a crear un **CounterViewModel** que será probado:

```kotlin
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow

class CounterViewModel {
    private val _count = MutableStateFlow(0)
    val count: StateFlow<Int> = _count.asStateFlow()

    fun increment() {
        _count.value += 1
    }
}
```

Ahora, creemos la **UI con Composables** que interactúa con este ViewModel:

```kotlin
import androidx.compose.foundation.layout.*
import androidx.compose.material.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.testTag
import androidx.compose.ui.unit.dp

@Composable
fun CounterScreen(viewModel: CounterViewModel) {
    val count by viewModel.count.collectAsState()

    Column(
        modifier = Modifier.padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(text = count.toString(), fontSize = 24.sp, modifier = Modifier.testTag("counterText"))
        Spacer(modifier = Modifier.height(8.dp))
        Button(onClick = { viewModel.increment() }, modifier = Modifier.testTag("incrementButton")) {
            Text("Increment")
        }
    }
}
```

---  

## **3. Escribiendo un Test de UI Común en CMP**

Ahora, escribamos un **test de UI** en `commonTest` para validar que al hacer clic en el botón, el contador se incremente:

```kotlin
import androidx.compose.ui.test.*
import kotlin.test.Test
import kotlin.test.assertEquals

class CounterScreenTest {

    @OptIn(ExperimentalTestApi::class)
    @Test
    fun testButtonIncrementsCounter() = runComposeUiTest {
        val viewModel = CounterViewModel()

        setContent {
            CounterScreen(viewModel = viewModel)
        }

        onNodeWithTag("counterText").assertTextEquals("0")
        onNodeWithTag("incrementButton").performClick()
        onNodeWithTag("counterText").assertTextEquals("1")
    }
}
```

---  

## **4. Ejecutando los Tests en Múltiples Plataformas**

Ahora que hemos escrito nuestr test en código común, ejecutémosla en **Android, Desktop e iOS**.

### **Ejecutar Tests en Android**

Para **tests instrumentadas en Android**, ejecuta:

```bash
./gradlew connectedAndroidTest
```

### **Ejecutar Tests en Desktop**

```bash
./gradlew desktopTest
```

### **Ejecutar Tests en iOS**

Para **tests en el simulador de iOS**, ejecuta:

```bash
./gradlew :composeApp:iosSimulatorArm64Test
```

---  

## **5. ¿Por Qué Probar CMP desde Código Común?**

✅ **Escribe una vez, prueba en todas partes:** No es necesario duplicar tests en cada plataforma.  
✅ **Comportamiento consistente en todas las plataformas:** Garantiza que los elementos de la UI funcionen de la misma manera.  
✅ **Mantenimiento más fácil:** Una única suite de tests cubriendo todos los objetivos.

---  

## **Conclusión**

Con **los tests de UI en Compose Multiplatform**, podemos validar el comportamiento de la UI **desde código compartido** sin necesidad de implementaciones de test específicos por plataforma. La biblioteca `compose.uiTest` nos permite probar interacciones de UI como la verificación de texto y clics en botones, asegurando consistencia en Android, iOS y Desktop.
