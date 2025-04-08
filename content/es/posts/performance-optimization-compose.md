---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Optimización de Rendimiento en Jetpack Compose"
date: 2024-04-08T08:00:00+01:00
description: "Aprende técnicas esenciales y mejores prácticas para optimizar el rendimiento en aplicaciones con Jetpack Compose, incluyendo optimización de composición, control de recomposición y gestión de memoria"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/remember-optimization.png
draft: false
tags:
- android
- compose
- performance
- optimization
---

# Optimización de Rendimiento en Jetpack Compose

La optimización del rendimiento es crucial para ofrecer una experiencia de usuario fluida en aplicaciones con Jetpack Compose. Este artículo explora técnicas clave y mejores prácticas para asegurar que tus funciones composables sean eficientes y tengan un buen rendimiento.

## Entendiendo la Composición y Recomposición

Uno de los aspectos fundamentales del rendimiento en Compose es entender cómo funcionan la composición y recomposición:

### Recomposición Inteligente

Compose utiliza recomposición inteligente para actualizar solo las partes de la UI que necesitan cambiar. Entender qué dispara la recomposición y cómo minimizar su alcance es crucial para la optimización del rendimiento.

```kotlin
@Composable
fun ExpensiveCalculation(numbers: List<Int>) {
    // Mal: Operación costosa realizada en cada recomposición
    val average = numbers.takeIf { it.isNotEmpty() }
        ?.average()
        ?: 0.0

    // Bien: Operación costosa cacheada y recalculada solo cuando cambia el input
    val cachedAverage = remember(numbers) {
        numbers.takeIf { it.isNotEmpty() }
            ?.average()
            ?: 0.0
    }

    Column {
        // Esto se recalculará en cada recomposición
        Text("Promedio Actual: ${"%.2f".format(average)}")

        // Esto usará el valor cacheado
        Text("Promedio Cacheado: ${"%.2f".format(cachedAverage)}")
    }
}
```

### Tipos Estables e Inmutabilidad

Los tipos estables son cruciales para el sistema de recomposición inteligente de Compose. Un tipo se considera estable cuando Compose puede garantizar que su método equals() es consistente con sus propiedades y que las propiedades mismas no cambiarán sin disparar una recomposición.

```kotlin
// Mal: Tipo inestable - propiedades mutables pueden cambiar sin notificar a Compose
data class UserState(
    var name: String,  // Propiedad mutable puede cambiar silenciosamente
    var age: Int      // Los cambios no dispararán recomposición
)

// Bien: Tipo estable - propiedades inmutables y estabilidad explícita
@Stable  // Indica a Compose que este tipo tiene una igualdad predecible
data class UserState(
    val name: String,  // Propiedad inmutable
    val age: Int      // Los cambios requieren crear una nueva instancia
)
```

El uso de tipos estables proporciona varios beneficios:
1. Recomposición más eficiente - Compose puede omitir la recomposición de partes de la UI cuando sabe que los datos no han cambiado
2. Comportamiento predecible - Los cambios en los datos siempre disparan actualizaciones apropiadas de la UI
3. Seguridad entre hilos - Los datos inmutables son seguros para compartir entre corrutinas

## Optimizaciones Clave de Rendimiento

### 1. Gestión de Estado con remember y derivedStateOf

Las funciones `remember` y `derivedStateOf` sirven diferentes propósitos en la gestión de estado:

```kotlin
@Composable
fun UserProfile(user: User, items: List<Item>) {
    // Mal: Recalculando en cada recomposición
    val filteredItems = items.filter { it.userId == user.id }

    // Bien: Cacheando cálculo con remember
    val cachedItems = remember(items, user.id) {
        items.filter { it.userId == user.id }
    }

    // Mejor: Usando derivedStateOf para computaciones reactivas
    val reactiveItems by remember(items) {
        derivedStateOf { 
            items.filter { it.userId == user.id }
        }
    }

    // reactiveItems se actualizará automáticamente cuando items cambie
    // y solo disparará recomposición cuando el resultado filtrado cambie
    LazyColumn {
        itemsIndexed(
            items = reactiveItems,
            key = { _: Int, item: Item -> item.id }
        ) { _: Int, item: Item ->
            ItemRow(item)
        }
    }
}
```

### 2. Uso de Composition Local

```kotlin
// Mal: Cada componente hijo accede a CompositionLocal
@Composable
fun DeepNestedContent() {
    val theme = LocalTheme.current  // Accedido directamente
    val strings = LocalStrings.current  // Múltiples accesos a CompositionLocal
    val dimensions = LocalDimensions.current

    Column {
        Text(
            text = strings.title,
            style = theme.textStyle,
            modifier = Modifier.padding(dimensions.padding)
        )
        // Más contenido anidado con accesos repetidos a CompositionLocal
    }
}

// Bien: Elevando valores de CompositionLocal para minimizar búsquedas
@Composable
fun ParentContent() {
    // Acceso único a valores de CompositionLocal
    val theme = LocalTheme.current
    val strings = LocalStrings.current
    val dimensions = LocalDimensions.current

    DeepNestedContent(
        theme = theme,
        strings = strings,
        dimensions = dimensions
    )
}

@Composable
fun DeepNestedContent(
    theme: Theme,
    strings: Strings,
    dimensions: Dimensions
) {
    // Usar parámetros pasados en lugar de buscar valores de CompositionLocal
    Column {
        Text(
            text = strings.title,
            style = theme.textStyle,
            modifier = Modifier.padding(dimensions.padding)
        )
        // Más contenido anidado usando parámetros pasados
    }
}
```

### 3. Optimizaciones de LazyList

El renderizado eficiente de listas es crucial para un desplazamiento suave. Aquí hay optimizaciones clave para componentes LazyList:

```kotlin
@Composable
fun <T : Any> OptimizedList(items: List<T>) {
    LazyColumn {
        itemsIndexed(
            items = items,
            // Las claves estables ayudan a Compose a rastrear items a través de actualizaciones
            key = { _: Int, item: T -> item.hashCode() }
        ) { _: Int, item: T ->
            // Contenido para cada item
        }
    }
}
```

Optimizaciones clave para LazyList:
1. Proporcionar claves estables para ayudar a Compose a rastrear items a través de actualizaciones
2. Usar tamaños fijos cuando sea posible para evitar remedición
3. Mantener los composables de items ligeros
4. Evitar asignaciones innecesarias en el contenido de items
5. Usar `remember` para cachear computaciones costosas por item

## Medición y Monitoreo del Rendimiento

### Layout Inspector y Trazas de Composición

El Layout Inspector en Android Studio es una herramienta poderosa para depurar el rendimiento de UI en Compose. Proporciona información sobre la jerarquía de vistas de tu app, conteos de recomposición y modificadores aplicados a cada composable.

Para usar Layout Inspector con Compose:

1. Ejecuta tu app en modo debug
2. En la ventana de Dispositivos en Ejecución encontrarás un botón para Alternar Layout Inspector
3. Inspecciona la jerarquía de Compose:
   - Ver árbol de componentes
   - Verificar conteos de recomposición
   - Analizar cadenas de modificadores
   - Inspeccionar parámetros de composables

[Toggle Layout inspector](https://carrion.dev/images/kotlin/layout-inspector.png)

Métricas clave para monitorear en Layout Inspector:
1. Conteos de recomposición - Números altos indican oportunidades potenciales de optimización
2. Conteos de omisión - Verifica que tus Composables estén omitiendo recomposición cuando deberían
3. Complejidad de cadena de modificadores - Cadenas largas pueden afectar el rendimiento de medición/layout

### Pruebas de Rendimiento

```kotlin
@Test
fun performanceTest() {
    benchmarkRule.measureRepeated(
        packageName = "com.example.app",
        metrics = listOf(FrameTimingMetric()),
        iterations = 5
    ) {
        composeTestRule.setContent {
            YourComposable()
        }
    }
}
```

## Resumen de Mejores Prácticas

1. Usar tipos estables y estructuras de datos inmutables
2. Elevar computaciones costosas con `remember`
3. Implementar claves apropiadas en listas lazy
4. Minimizar el alcance de la recomposición
5. Perfilar y medir el rendimiento regularmente

Seguir estas técnicas de optimización ayudará a asegurar que tu UI en Compose permanezca responsiva y eficiente, proporcionando una mejor experiencia de usuario para tus aplicaciones.