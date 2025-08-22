---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Inyección de Dependencias + Inversión de Dependencias: Código más Robusto y Testeable"
date: 2025-08-22T08:00:00+01:00
description: "Cómo aplicar el Principio de Inversión de Dependencias de SOLID junto con un framework de Inyección de Dependencias conduce a código desacoplado, robusto y altamente testeable."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/dip.png
draft: false
tags:
- architecture
- solid
- dependency-injection
- testing
---

### Inyección de Dependencias + Inversión de Dependencias: Código más Robusto y Testeable

Las aplicaciones modernas evolucionan rápido: se añaden funcionalidades, se multiplican las plataformas y los equipos crecen. En este contexto, el acoplamiento fuerte se convierte en un freno al cambio y en una fuente de tests frágiles. Dos ideas clave nos ayudan a combatir esta complejidad:

- Principio de Inversión de Dependencias (DIP) de SOLID: los módulos de alto nivel no deben depender de los de bajo nivel. Ambos deben depender de abstracciones.
- Inyección de Dependencias (DI): técnica y conjunto de herramientas para proveer esas dependencias desde fuera en lugar de construirlas internamente.

En este artículo veremos por qué importa DIP, cómo DI lo hace cumplir y cómo usar un framework de DI vuelve tu código más robusto y fácil de testear, con ejemplos en Kotlin aplicables a otros lenguajes.

---

#### Inversión de Dependencias en pocas palabras (la “D” de SOLID)

DIP promueve diseñar alrededor de abstracciones (interfaces) en lugar de implementaciones concretas. El objetivo es desacoplar la lógica de negocio (alto nivel) de los detalles de infraestructura (bajo nivel):

- Los módulos de alto nivel dependen de interfaces que definen o controlan
- Los módulos de bajo nivel implementan esas interfaces
- La composición ocurre en los bordes (inicio de la app, contenedor DI, fábricas)

Esto reduce el efecto dominó de los cambios y facilita la sustitución y las pruebas.

```kotlin
// Política de alto nivel
interface PaymentProcessor {
    suspend fun charge(amount: Money): Result<Unit>
}

// Implementación de bajo nivel
class StripePaymentProcessor(private val api: StripeApi) : PaymentProcessor {
    override suspend fun charge(amount: Money): Result<Unit> = api.charge(amount)
}
```

---

#### Qué aporta la Inyección de Dependencias

DI es la práctica de proveer implementaciones concretas a los componentes desde fuera. En lugar de construir dependencias con `new` (o dentro del constructor) en una clase, las aceptamos por parámetros de constructor (o setters), y una raíz de composición (a menudo un framework de DI) cablea el grafo. Beneficios:

- Desacoplo: las clases dependen de interfaces, no de cómo crearlas
- Testeabilidad: las dependencias se reemplazan fácilmente con fakes/mocks
- Responsabilidad Única: las clases se enfocan en comportamiento, no en construcción
- Reemplazabilidad: intercambiar implementaciones sin tocar los consumidores

```kotlin
class CheckoutService(private val paymentProcessor: PaymentProcessor) {
    suspend fun checkout(cart: Cart): Result<Unit> {
        // solo lógica de negocio
        return paymentProcessor.charge(cart.total())
    }
}
```

`CheckoutService` no conoce Stripe, PayPal ni clientes HTTP: solo la abstracción.

---

#### Antes vs Después: Un ejemplo concreto

Sin DIP/DI (fuertemente acoplado):

```kotlin
class CheckoutService {
    private val api = StripeApi(httpClient = HttpClient())
    private val processor = StripePaymentProcessor(api)

    suspend fun checkout(cart: Cart): Result<Unit> {
        // Difícil de testear: se cuelan red y detalles de Stripe
        return processor.charge(cart.total())
    }
}
```

Problemas:
- Difícil de testear unitariamente (requiere red o herramientas de mockeo pesadas)
- Cambiar proveedor (Stripe -> PayPal) obliga a tocar esta clase
- Viola SRP al mezclar construcción y reglas de negocio

Con DIP + DI:

```kotlin
class CheckoutService(private val paymentProcessor: PaymentProcessor) {
    suspend fun checkout(cart: Cart): Result<Unit> = paymentProcessor.charge(cart.total())
}
```

Composición (con cualquier estilo de DI):

```kotlin
val api = StripeApi(HttpClient())
val processor: PaymentProcessor = StripePaymentProcessor(api)
val service = CheckoutService(processor)
```

El test se vuelve trivial:

```kotlin
class FakePaymentProcessor : PaymentProcessor {
    var lastAmount: Money? = null
    override suspend fun charge(amount: Money): Result<Unit> {
        lastAmount = amount
        return Result.success(Unit)
    }
}

@Test
fun `checkout cobra el total del carrito`() = runTest {
    val fake = FakePaymentProcessor()
    val service = CheckoutService(fake)

    val result = service.checkout(cart = Cart(listOf(Line("abc", 100))))

    assertTrue(result.isSuccess)
    assertEquals(Money(100), fake.lastAmount)
}
```

---

#### Usar un framework de DI: por qué ayuda

A medida que la aplicación crece, el cableado manual se vuelve propenso a errores. Los frameworks de DI actúan como motor de composición:

- Centralizan grafos de objetos y ciclos de vida (singleton, scoped, transient)
- Fuerzan dependencias explícitas (inyección por constructor)
- Aportan validación en compilación o en tiempo de ejecución

Opciones comunes según ecosistema:

- Kotlin/Android: Hilt/Dagger (tiempo de compilación, anotaciones), Koin (DSL), Kodein
- JVM/backend: Spring Framework/Spring Boot (anotaciones), Guice
- Multiplataforma: Koin funciona en KMP; muchos usan composición manual o Service Locator en el código compartido

Ejemplo con Koin (DSL simple):

```kotlin
val appModule = module {
    single { HttpClient() }
    single { StripeApi(get()) }
    single<PaymentProcessor> { StripePaymentProcessor(get()) }
    factory { CheckoutService(get()) }
}

startKoin { modules(appModule) }

val service: CheckoutService = get()
```

Ejemplo con Hilt (Android):

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object PaymentsModule {
    @Provides fun provideHttpClient(): HttpClient = HttpClient()
    @Provides fun provideStripeApi(client: HttpClient) = StripeApi(client)
    @Provides fun provideProcessor(api: StripeApi): PaymentProcessor = StripePaymentProcessor(api)
}

@HiltViewModel
class CheckoutViewModel @Inject constructor(
    private val service: CheckoutService
) : ViewModel() {
    // ...
}
```

---

#### Robustez y Testeabilidad en la práctica

- Menor radio de impacto: cambios de infraestructura no se propagan a la lógica de negocio
- Tests deterministas: inyecta fakes; sin singletons ocultos ni estado global
- Fronteras claras: las interfaces modelan las costuras del sistema
- Mantenimiento más fácil: los constructores hacen visibles y revisables las dependencias

Checklist:

- ¿Mis módulos de alto nivel dependen solo de abstracciones que controlan?
- ¿Puedo intercambiar implementaciones sin editar los consumidores?
- ¿Los constructores son la única vía de entrada de dependencias?
- ¿Puedo testear sin red, disco o hilos?

---

#### Errores comunes y cómo evitarlos

- Antipatrón Service Locator: evita obtener dependencias de un registro global dentro de las clases; prefiere inyección por constructor
- Módulos “Dios”: divide módulos de DI por feature/límite para evitar grafos gigantes
- Sobre-abstracción: no crees interfaces sin implementaciones alternativas o valor de test
- Estado oculto: evita singletons estáticos; prefiere ciclos de vida gestionados por DI

---

#### Conclusión

Aplicar el Principio de Inversión de Dependencias y adoptar la Inyección de Dependencias produce un código más mantenible, extensible y testeable. Empieza pequeño: define interfaces en tus fronteras, inyecta dependencias por constructor y, conforme crezca tu grafo, introduce un framework de DI. La recompensa es notable: arquitectura más limpia, tests más rápidos y cambios más seguros.
