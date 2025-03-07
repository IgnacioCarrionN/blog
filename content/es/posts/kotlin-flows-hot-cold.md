---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Entendiendo los Hot y Cold Flows en Kotlin"
date: 2024-03-06T08:00:00+01:00
description: "Una guía completa para entender las diferencias entre hot y cold flows en Kotlin, con ejemplos prácticos"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/flows.png
draft: false
tags:
- kotlin
- coroutines
- flows
---

# Entendiendo los Hot y Cold Flows en Kotlin

Kotlin Flow es una potente característica para manejar flows reactivos de datos. Uno de los conceptos fundamentales para entender cuando trabajamos con flows es la distinción entre hot y cold flows. Este artículo explicará las diferencias y proporcionará ejemplos prácticos de ambos tipos.

## Cold Flows: El Comportamiento por Defecto

Los cold flows son el tipo predeterminado en Kotlin Flow. Comienzan a producir valores solo cuando un collector empieza a recolectar de ellos. Cada collector obtiene su propio flow independiente de valores desde el principio.

### Características de los Cold Flows:

- Comienzan a producir valores solo cuando son recolectados
- Cada collector recibe todos los valores desde el principio
- Los valores se producen independientemente para cada collector
- Los recursos no se comparten entre collectors

Aquí hay un ejemplo de un flujo cold:

```kotlin
fun numbers(): Flow<Int> = flow {
    println("Flow started")
    for (i in 1..3) {
        delay(100)
        emit(i)
    }
}

suspend fun main() {
    val numbersFlow = numbers()

    // First collector
    println("First collector starting")
    numbersFlow.collect { value: Int ->
        println("Collector 1: $value")
    }

    // Second collector
    println("Second collector starting")
    numbersFlow.collect { value: Int ->
        println("Collector 2: $value")
    }
}
```

Output:
```
First collector starting
Flow started
Collector 1: 1
Collector 1: 2
Collector 1: 3
Second collector starting
Flow started
Collector 2: 1
Collector 2: 2
Collector 2: 3
```

## Hot Flows: Estado Compartido y Eventos

Los hot flows, por otro lado, pueden comenzar a producir valores independientemente de los collectors y pueden compartir el mismo flow de valores entre múltiples collectors. Son útiles para representar eventos en tiempo real o estado compartido.

### Tipos de Hot Flows:

1. **StateFlow**: Para representar estado
2. **SharedFlow**: Para representar eventos

### Ejemplo de StateFlow:

```kotlin
class TemperatureSensor {
    private val _temperature = MutableStateFlow(20.0) // Initial temperature
    val temperature: StateFlow<Double> = _temperature.asStateFlow()

    fun updateTemperature(newTemp: Double) {
        _temperature.update { newTemp }
    }
}

suspend fun main() {
    val sensor = TemperatureSensor()

    // First collector
    launch {
        sensor.temperature.collect { temp: Double ->
            println("Display 1: $temp°C")
        }
    }

    delay(100)
    println("Updating temperature to 22.5°C")
    sensor.updateTemperature(22.5)

    delay(100)
    // Late collector - will only see the current value (22.5) and future updates
    launch {
        println("Display 2 starting collection...")
        sensor.temperature.collect { temp: Double ->
            println("Display 2: $temp°C")
        }
    }

    delay(100)
    println("Updating temperature to 23.0°C")
    sensor.updateTemperature(23.0)
}
```

Output:
```
Display 1: 20.0°C
Updating temperature to 22.5°C
Display 1: 22.5°C
Display 2 starting collection...
Display 2: 22.5°C
Updating temperature to 23.0°C
Display 1: 23.0°C
Display 2: 23.0°C
```

### Ejemplo de SharedFlow:

```kotlin
class EventBus {
    private val _events = MutableSharedFlow<String>()
    val events = _events.asSharedFlow()

    suspend fun emit(event: String) {
        _events.emit(event)
    }
}

suspend fun main() {
    val eventBus = EventBus()

    // First subscriber
    launch {
        eventBus.events.collect { event: String ->
            println("Subscriber 1: $event")
        }
    }

    delay(100)
    println("Emitting first event")
    eventBus.emit("User logged in")

    delay(100)
    // Late subscriber - will miss the "User logged in" event
    launch {
        println("Subscriber 2 starting collection...")
        eventBus.events.collect { event: String ->
            println("Subscriber 2: $event")
        }
    }

    delay(100)
    println("Emitting second event")
    eventBus.emit("Data updated")
}
```

Output:
```
Subscriber 1: User logged in
Subscriber 2 starting collection...
Emitting second event
Subscriber 1: Data updated
Subscriber 2: Data updated
```

## Diferencias Clave Entre Hot y Cold Flows

1. **Tiempo de Ejecución**
   - Cold Flow: Se ejecuta por cada collector
   - Hot Flow: Puede ejecutarse independientemente de los collectors

2. **Compartición de Valores**
   - Cold Flow: Cada collector obtiene su propio flow
   - Hot Flow: Múltiples collectors comparten el mismo flow

3. **Uso de Recursos**
   - Cold Flow: Recursos asignados por collector
   - Hot Flow: Recursos compartidos entre collectors

4. **Casos de Uso**
   - Cold Flow: Transformaciones de datos, consultas a base de datos
   - Hot Flow: Estados de UI, eventos en tiempo real, broadcasts

## Cuándo Usar Cada Tipo

### Usa Cold Flows Cuando:
- Cada collector necesita su propio flow independiente
- Estás realizando operaciones que consumen muchos recursos
- Necesitas reiniciar el flow desde el principio para cada collector

### Usa Hot Flows Cuando:
- Múltiples partes de tu app necesitan el mismo flow de datos
- Estás manejando estado de UI
- Necesitas transmitir eventos a múltiples suscriptores
- Quieres compartir recursos entre collectors

## Mejores Prácticas

1. **Cold Flows**
   - Úsalos para operaciones que deben ejecutarse independientemente
   - Considera usar buffer() para optimización de rendimiento
   - Limpia los recursos en onCompletion

2. **Hot Flows**
   - Usa StateFlow para gestión de estado
   - Usa SharedFlow para eventos
   - Considera cuidadosamente los tamaños de replay y buffer
   - Maneja la contrapresión adecuadamente

Al entender estas diferencias, puedes elegir el tipo correcto de flow para tu caso de uso específico y crear flows reactivos más eficientes y mantenibles en tus aplicaciones Kotlin.
