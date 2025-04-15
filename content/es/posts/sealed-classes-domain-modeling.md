---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Aprovechando las Sealed Classes e Interfaces para un Mejor Modelado de Dominio"
date: 2025-04-15T08:00:00+01:00
description: "Cómo utilizar las sealed classes e interfaces de Kotlin para crear modelos de dominio más robustos y con seguridad de tipos."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/sealed-classes.png
draft: false
tags: 
- kotlin
- architecture
- domain modeling
- type safety
---

### Aprovechando las Sealed Classes e Interfaces para un Mejor Modelado de Dominio

El modelado de dominio es un aspecto crucial del desarrollo de software, representando los conceptos y reglas de negocio fundamentales en tu aplicación. Kotlin proporciona potentes características de lenguaje que pueden ayudar a crear modelos de dominio más expresivos, con seguridad de tipos y mantenibles. Entre estas características, las sealed classes e interfaces destacan como herramientas particularmente valiosas. Este artículo explora cómo aprovechar estas características de Kotlin para construir mejores modelos de dominio.

---

#### Entendiendo las Sealed Classes e Interfaces

Las sealed classes e interfaces en Kotlin son construcciones especiales que restringen la jerarquía de un tipo. Cuando una clase o interfaz se marca como `sealed`, todas sus subclases deben definirse dentro del mismo archivo (o, desde Kotlin 1.5, dentro del mismo módulo como subclases directas).

```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val message: String, val cause: Exception? = null) : Result<Nothing>()
    object Loading : Result<Nothing>()
}

fun main() {
    val result: Result<String> = Result.Success("Data loaded successfully")

    val message = when (result) {
        is Result.Success -> "Success: ${result.data}"
        is Result.Error -> "Error: ${result.message}"
        is Result.Loading -> "Loading..."
    }

    println(message) // Output: Success: Data loaded successfully
}
```

Los beneficios clave de las sealed classes incluyen:

1. **Expresiones when exhaustivas**: El compilador asegura que todas las posibles subclases sean manejadas en una expresión `when`
2. **Jerarquía restringida**: Todas las subclases deben ser conocidas en tiempo de compilación
3. **Seguridad de tipos**: El compilador puede verificar que todos los casos sean manejados
4. **Expresividad**: Comunican claramente que un tipo tiene un conjunto limitado de subtipos

---

#### Modelado de Dominio con Sealed Classes

Exploremos cómo las sealed classes pueden mejorar el modelado de dominio a través de ejemplos prácticos.

##### Ejemplo 1: Modelando Métodos de Pago

Considera una aplicación de comercio electrónico que necesita manejar diferentes métodos de pago:

```kotlin
sealed class PaymentMethod {
    data class CreditCard(
        val cardNumber: String,
        val expiryDate: String,
        val cvv: String
    ) : PaymentMethod()

    data class PayPal(val email: String) : PaymentMethod()

    data class BankTransfer(
        val accountNumber: String,
        val bankCode: String
    ) : PaymentMethod()

    object Cash : PaymentMethod()
}

class PaymentProcessor {
    fun process(payment: Payment) {
        val message = when (payment.method) {
            is PaymentMethod.CreditCard -> {
                val card = payment.method
                "Processing credit card payment with card ending with ${card.cardNumber.takeLast(4)}"
            }
            is PaymentMethod.PayPal -> {
                "Processing PayPal payment for ${payment.method.email}"
            }
            is PaymentMethod.BankTransfer -> {
                "Processing bank transfer from account ${payment.method.accountNumber}"
            }
            PaymentMethod.Cash -> {
                "Processing cash payment"
            }
        }
        println(message)
    }
}

data class Payment(
    val amount: Double,
    val currency: String,
    val method: PaymentMethod
)
```

Este enfoque ofrece varias ventajas:

1. **Seguridad de tipos**: El compilador asegura que manejemos todos los métodos de pago
2. **Código autodocumentado**: La sealed class muestra claramente todos los posibles métodos de pago
3. **Extensibilidad**: Añadir un nuevo método de pago es tan simple como añadir una nueva subclase
4. **Pattern matching**: La expresión `when` proporciona una forma limpia de manejar diferentes métodos de pago

---

##### Ejemplo 2: Modelando Respuestas de API

Las sealed classes son particularmente útiles para modelar respuestas de API:

```kotlin
sealed class ApiResponse<out T> {
    data class Success<T>(val data: T) : ApiResponse<T>()
    data class Error(val code: Int, val message: String) : ApiResponse<Nothing>()
    object Loading : ApiResponse<Nothing>()
    object Empty : ApiResponse<Nothing>()
}

class UserRepository {
    fun getUser(id: String): ApiResponse<User> {
        return try {
            // Simulate API call
            if (id == "123") {
                ApiResponse.Success(User("123", "John Doe"))
            } else {
                ApiResponse.Error(404, "User not found")
            }
        } catch (e: Exception) {
            ApiResponse.Error(500, e.message ?: "Unknown error")
        }
    }
}

data class User(val id: String, val name: String)

fun main() {
    val repository = UserRepository()

    val response = repository.getUser("123")

    val result = when (response) {
        is ApiResponse.Success -> "User found: ${response.data.name}"
        is ApiResponse.Error -> "Error: ${response.message} (${response.code})"
        ApiResponse.Loading -> "Loading..."
        ApiResponse.Empty -> "No user data available"
    }

    println(result) // Output: User found: John Doe
}
```

Este patrón es ampliamente utilizado en el desarrollo de Android con arquitecturas como MVI (Model-View-Intent) y ayuda a crear una clara separación entre diferentes estados de datos.

---

#### Interfaces Selladas para Jerarquías Más Flexibles

Kotlin 1.5 introdujo interfaces selladas, que proporcionan más flexibilidad que las sealed classes porque una clase puede implementar múltiples interfaces:

```kotlin
sealed interface Error {
    val message: String
}

sealed interface NetworkError : Error

data class ServerError(override val message: String) : NetworkError
data class ConnectionError(override val message: String) : NetworkError

sealed interface DatabaseError : Error

data class QueryError(override val message: String) : DatabaseError
data class TransactionError(override val message: String) : DatabaseError

class ErrorHandler {
    fun handle(error: Error) {
        val action = when (error) {
            is ServerError -> "Retry server request"
            is ConnectionError -> "Check internet connection"
            is QueryError -> "Fix database query"
            is TransactionError -> "Rollback transaction"
        }

        println("Error: ${error.message}. Action: $action")
    }
}
```

Las interfaces selladas permiten jerarquías más complejas mientras mantienen los beneficios de exhaustividad y seguridad de tipos.

---

#### Patrones Avanzados de Modelado de Dominio

Exploremos algunos patrones avanzados usando sealed classes e interfaces.

##### Máquinas de Estado con Sealed Classes

Las sealed classes son excelentes para implementar máquinas de estado:

```kotlin
sealed class OrderState {
    object Created : OrderState()
    data class Processing(val startTime: Long) : OrderState()
    data class Shipped(val trackingNumber: String) : OrderState()
    data class Delivered(val deliveryTime: Long) : OrderState()
    data class Cancelled(val reason: String) : OrderState()
}

class OrderStateMachine {
    fun transition(currentState: OrderState, event: OrderEvent): OrderState {
        return when (currentState) {
            is OrderState.Created -> handleCreatedState(event)
            is OrderState.Processing -> handleProcessingState(event)
            is OrderState.Shipped -> handleShippedState(event)
            is OrderState.Delivered -> currentState // Terminal state
            is OrderState.Cancelled -> currentState // Terminal state
        }
    }

    private fun handleCreatedState(event: OrderEvent): OrderState {
        return when (event) {
            is OrderEvent.StartProcessing -> OrderState.Processing(System.currentTimeMillis())
            is OrderEvent.CancelOrder -> OrderState.Cancelled(event.reason)
            else -> throw IllegalStateException("Invalid event $event for Created state")
        }
    }

    private fun handleProcessingState(event: OrderEvent): OrderState {
        return when (event) {
            is OrderEvent.ShipOrder -> OrderState.Shipped(event.trackingNumber)
            is OrderEvent.CancelOrder -> OrderState.Cancelled(event.reason)
            else -> throw IllegalStateException("Invalid event $event for Processing state")
        }
    }

    private fun handleShippedState(event: OrderEvent): OrderState {
        return when (event) {
            is OrderEvent.DeliverOrder -> OrderState.Delivered(System.currentTimeMillis())
            else -> throw IllegalStateException("Invalid event $event for Shipped state")
        }
    }
}

sealed class OrderEvent {
    object StartProcessing : OrderEvent()
    data class ShipOrder(val trackingNumber: String) : OrderEvent()
    object DeliverOrder : OrderEvent()
    data class CancelOrder(val reason: String) : OrderEvent()
}
```

Este patrón asegura que:
1. Todos los posibles estados estén explícitamente definidos
2. Las transiciones de estado estén controladas y validadas
3. El compilador ayude a asegurar que todos los estados sean manejados
4. El código sea autodocumentado respecto a los posibles estados y transiciones

---

#### Mejores Prácticas para Usar Sealed Classes en el Modelado de Dominio

1. **Usa sealed classes para representar conjuntos finitos de posibilidades**
   - Respuestas de API (Éxito, Error, Cargando)
   - Máquinas de estado (Creado, Procesando, Completado)
   - Patrones de comando (Añadir, Eliminar, Actualizar)

2. **Prefiere interfaces selladas cuando las clases necesiten implementar múltiples interfaces**
   - Jerarquías de error
   - Capacidades de características
   - Preocupaciones transversales

3. **Combina con data classes para objetos de valor inmutables**
   - Hace tu modelo de dominio más predecible
   - Proporciona implementaciones de equals(), hashCode() y toString()
   - Permite declaraciones de desestructuración

4. **Aprovecha las expresiones when exhaustivas**
   - Deja que el compilador asegure que todos los casos sean manejados
   - Usa la declaración `when` sin una rama `else` para forzar el manejo de todos los casos

5. **Mantén la jerarquía poco profunda**
   - Las jerarquías profundas pueden volverse difíciles de entender
   - Considera la composición sobre la herencia para comportamientos complejos

6. **Usa sealed classes anidadas para conceptos relacionados**
   - Ayuda a organizar el código y mantener el contexto
   - Reduce la contaminación del espacio de nombres

---

### Conclusión

Las sealed classes e interfaces son herramientas poderosas para el modelado de dominio en Kotlin. Proporcionan seguridad de tipos, verificación de exhaustividad y expresión clara de conceptos de negocio. Al aprovechar estas características, puedes crear modelos de dominio más robustos, mantenibles y autodocumentados.

Recuerda que un buen modelado de dominio consiste en expresar claramente los conceptos y reglas de negocio en tu código. Las sealed classes e interfaces ayudan a lograr este objetivo proporcionando una forma de modelar conjuntos finitos de posibilidades de manera segura en cuanto a tipos. Ya sea que estés construyendo una plataforma de comercio electrónico, un sistema de gestión de contenido o una aplicación móvil, estas características de Kotlin pueden mejorar significativamente la calidad de tu modelo de dominio.

Como con cualquier herramienta, la clave está en saber cuándo y cómo aplicarla. Usa sealed classes e interfaces cuando necesites representar un conjunto cerrado de posibilidades, y combínalas con otras características de Kotlin como data classes y funciones de extensión para crear modelos de dominio expresivos y mantenibles.
