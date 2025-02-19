---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Gestión Confiable del Tiempo con la API TrustedTime en Android"
date: 2025-02-19T08:00:00+01:00
description: "La API TrustedTime aprovecha la infraestructura segura de Google para ofrecer una marca de tiempo confiable"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/trusted-time-api.png
draft: false
tags:
- kotlin
- android
- google
---

**Gestión Confiable del Tiempo con la API TrustedTime en Android**

La gestión precisa del tiempo es crucial para muchas funcionalidades de las aplicaciones, como la programación de tareas, el registro de transacciones y la seguridad. Sin embargo, depender del reloj del sistema de un dispositivo puede ser problemático, ya que los usuarios pueden modificar la configuración de la hora. Para abordar este problema, Google ha introducido la API TrustedTime, que proporciona una fuente de tiempo confiable y resistente a manipulaciones para las aplicaciones de Android.

## Comprendiendo la API TrustedTime

La API TrustedTime aprovecha la infraestructura segura de Google para ofrecer una marca de tiempo confiable, independiente de la configuración horaria local del dispositivo. Se sincroniza periódicamente con los servidores de tiempo precisos de Google, reduciendo la necesidad de solicitudes frecuentes a la red. La API también tiene en cuenta la deriva del reloj, alertando a los desarrolladores cuando la precisión del tiempo puede degradarse entre sincronizaciones.

## Importancia de la Gestión Precisa del Tiempo

Depender únicamente del reloj de un dispositivo puede causar problemas como:

- **Inconsistencia de Datos:** Las aplicaciones que dependen del orden de los eventos pueden enfrentar corrupción de datos si los usuarios manipulan la hora del dispositivo.
- **Riesgos de Seguridad:** Las medidas de seguridad basadas en el tiempo, como OTP y controles de acceso, requieren un reloj preciso.
- **Programaciones No Confiables:** Los recordatorios y eventos programados pueden fallar si la hora del dispositivo es incorrecta.
- **Deriva del Reloj:** Los relojes internos pueden desviarse debido a factores como cambios de temperatura y niveles de batería.
- **Problemas de Sincronización entre Dispositivos:** Configuraciones horarias inconsistentes en diferentes dispositivos pueden interrumpir la sincronización de datos.
- **Alto Consumo de Batería y Datos:** Consultar constantemente servidores de tiempo en la red consume recursos, lo que TrustedTime ayuda a optimizar.

## Aplicaciones Prácticas de TrustedTime

La API TrustedTime mejora la seguridad y confiabilidad en varios escenarios:

- **Aplicaciones Financieras:** Garantiza marcas de tiempo precisas para transacciones.
- **Juegos:** Previene exploits basados en el tiempo.
- **E-Commerce:** Realiza un seguimiento preciso del procesamiento y entrega de pedidos.
- **Ofertas por Tiempo Limitado:** Garantiza que las promociones expiren correctamente.
- **Dispositivos IoT:** Sincroniza relojes en múltiples dispositivos.
- **Aplicaciones de Productividad:** Mantiene marcas de tiempo precisas en la edición de documentos en la nube.

## Integrando TrustedTime en tu Aplicación Android

### 1. Agregar la Dependencia

Incluye la API TrustedTime en tu archivo `build.gradle`:

```groovy
implementation 'com.google.android.gms:play-services-time:16.0.1'
```

### 2. Inicializar TrustedTimeClient

```kotlin
class MyApp : Application() {
    var trustedTimeClient: TrustedTimeClient? = null
        private set

    override fun onCreate() {
        super.onCreate()
        TrustedTime.createClient(this).addOnCompleteListener { task ->
            if (task.isSuccessful) {
                trustedTimeClient = task.result
            } else {
                // Manejar error
            }
        }
    }
}
```

### 3. Usando TrustedTime en una Actividad o Fragment

Para recuperar la hora confiable dentro de una actividad o fragmento, podemos acceder al `TrustedTimeClient` desde la clase de aplicación:

```kotlin
class MainActivity : AppCompatActivity() {
    private val trustedTimeClient: TrustedTimeClient?
        get() = (application as MyApp).trustedTimeClient

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate()
        setContentView(R.layout.activity_main)
        getTrustedTime()
    }

    private fun getTrustedTime() {
        val currentTimeMillis = trustedTimeClient?.computeCurrentUnixEpochMillis()
            ?: System.currentTimeMillis()
        Log.d("TrustedTime", "Hora confiable actual: $currentTimeMillis")
    }
}
```

Al recuperar `trustedTimeClient` desde `MyApp`, evitamos inicializaciones redundantes y aseguramos una única fuente confiable de tiempo en toda la aplicación.

### 4. Recuperar TrustedTime con un Mecanismo de Respaldo

```kotlin
val currentTimeMillis = trustedTimeClient?.computeCurrentUnixEpochMillis()
    ?: System.currentTimeMillis()
```

Esto garantiza un respaldo al reloj del sistema si TrustedTime no está disponible.

## Consideraciones y Limitaciones

- **Requiere Conexión a Internet:** TrustedTime necesita acceso a Internet después del arranque del dispositivo para funcionar correctamente.
- **Deriva del Reloj:** Aunque la API proporciona una estimación del error, no puede prevenir completamente la deriva del reloj.
- **Protección contra Manipulación:** Reduce, pero no elimina por completo, los riesgos de manipulación del tiempo.
- **Disponibilidad y Limitaciones de la API TrustedTime:** La API TrustedTime está disponible en todos los dispositivos que ejecutan Google Play Services en Android 5 (Lollipop) y versiones superiores.



Al integrar la API TrustedTime, los desarrolladores pueden mejorar la precisión y seguridad de las funcionalidades dependientes del tiempo en sus aplicaciones, garantizando una experiencia consistente y confiable para los usuarios.
