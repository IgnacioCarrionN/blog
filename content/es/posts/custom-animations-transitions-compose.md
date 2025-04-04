---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Animaciones y Transiciones Personalizadas en Jetpack Compose"
date: 2025-04-04T08:00:00+01:00
description: "Análisis profundo sobre la creación de animaciones y transiciones personalizadas en Jetpack Compose, cubriendo APIs de animación, transiciones personalizadas y optimización de rendimiento"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/pulsating-dot.png
draft: false
tags:
- android
- compose
- animation
- transitions
---

# Animaciones y Transiciones Personalizadas en Jetpack Compose

Crear animaciones fluidas y significativas es crucial para ofrecer una experiencia de usuario pulida. Este artículo explora cómo crear animaciones y transiciones personalizadas en Jetpack Compose, desde animaciones básicas hasta implementaciones personalizadas complejas.

## Creando Animaciones Personalizadas

Las animaciones personalizadas permiten efectos visuales más complejos y únicos:

### Especificaciones de Animación Personalizadas

```kotlin
@Composable
fun CustomAnimatedButton(
    onClick: () -> Unit,
    content: @Composable () -> Unit
) {
    var isPressed by remember { mutableStateOf(false) }
    val scope = rememberCoroutineScope()

    val scale by animateFloatAsState(
        targetValue = if (isPressed) 0.95f else 1f,
        animationSpec = spring(
            dampingRatio = 0.4f,
            stiffness = Spring.StiffnessLow
        )
    )

    Box(
        modifier = Modifier
            .scale(scale)
            .clickable(
                interactionSource = remember { MutableInteractionSource() },
                indication = null
            ) {
                isPressed = true
                onClick()
                // Restablecer después de la animación usando un scope consciente del ciclo de vida
                scope.launch {
                    delay(100)
                    isPressed = false
                }
            }
    ) {
        content()
    }
}
```

![Custom Animated Button](images/kotlin/custom-animated-button.gif)

### Animaciones Infinitas

Para animaciones continuas como indicadores de carga:

```kotlin
@Composable
fun PulsatingDot(
    color: Color,
    size: Dp = 20.dp
) {
    val infiniteTransition = rememberInfiniteTransition(label = "pulsating")
    val scale by infiniteTransition.animateFloat(
        initialValue = 0.6f,
        targetValue = 1f,
        animationSpec = infiniteRepeatable(
            animation = tween(1000),
            repeatMode = RepeatMode.Reverse
        ),
        label = "scale"
    )

    Box(
        modifier = Modifier
            .size(size)
            .scale(scale)
            .background(color, CircleShape)
    )
}
```

![Pulsating Dot](images/kotlin/pulsating-dot.gif)

## Implementando Transiciones Personalizadas

Las transiciones ayudan a crear cambios suaves entre diferentes estados de la UI:

### Transiciones de Contenido

```kotlin
@Composable
fun AnimatedContent(
    content: @Composable () -> Unit,
    modifier: Modifier = Modifier
) {
    AnimatedVisibility(
        visible = true,
        enter = fadeIn() + expandVertically(),
        exit = fadeOut() + shrinkVertically()
    ) {
        Box(modifier = modifier) {
            content()
        }
    }
}
```

![Animated Content](images/kotlin/animated-content.gif)

### Especificaciones de Transición Personalizadas

```kotlin
@Composable
fun CustomTransitionCard(
    expanded: Boolean,
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit
) {
    val transition = updateTransition(
        targetState = expanded,
        label = "card_transition"
    )

    val cardElevation by transition.animateDp(
        label = "elevation",
        targetValueByState = { isExpanded: Boolean ->
            if (isExpanded) 8.dp else 2.dp
        }
    )

    val cardRoundedCorners by transition.animateDp(
        label = "corner_radius",
        targetValueByState = { isExpanded: Boolean ->
            if (isExpanded) 0.dp else 16.dp
        }
    )

    Card(
        modifier = modifier,
        elevation = CardDefaults.cardElevation(defaultElevation = cardElevation),
        shape = RoundedCornerShape(cardRoundedCorners)
    ) {
        content()
    }
}
```

![Custom Transition Card](images/kotlin/custom-transition-card.gif)

## Mejores Prácticas y Optimización de Rendimiento

Al implementar animaciones personalizadas, ten en cuenta estas mejores prácticas:

### 1. Gestión del Estado de Animación

Mantén el estado de la animación cerca de donde se usa:

```kotlin
@Composable
fun OptimizedAnimation() {
    // ❌ No almacenes valores de animación a nivel de composable
    // var scale by remember { mutableStateOf(1f) }

    // ✅ Usa AnimationState o APIs animate*
    val scale by animateFloatAsState(
        targetValue = 1f,
        label = "scale"
    )
}
```

### 2. Consideraciones de Rendimiento

- Usa `remember` para cálculos costosos
- Evita animar parámetros de layout cuando sea posible
- Considera usar `LaunchedEffect` para animaciones complejas

## Conclusión

Las animaciones y transiciones personalizadas en Jetpack Compose proporcionan herramientas poderosas para crear experiencias de usuario atractivas. Al comprender los conceptos fundamentales y seguir las mejores prácticas, puedes crear animaciones fluidas y eficientes que mejoren la interfaz de usuario de tu aplicación.

Recuerda:
- Comienza con animaciones simples y aumenta gradualmente la complejidad
- Prueba las animaciones en diferentes dispositivos y tamaños de pantalla
- Considera las implicaciones de accesibilidad
- Monitoriza el impacto en el rendimiento
- Usa principios de animación para crear transiciones significativas

Para temas más avanzados, consulta nuestros otros artículos sobre optimización de rendimiento y gestión de estado en Compose.

Puedes encontrar todos los ejemplos de este post en el siguiente repositorio de GitHub: [ComposeAnimations](https://github.com/IgnacioCarrionN/ComposeAnimations)
