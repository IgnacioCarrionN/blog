---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Aprovechando expect/actual en Kotlin Multiplatform para Implementaciones Nativas"
date: 2025-04-22T08:00:00+01:00
description: "Cómo utilizar el mecanismo expect/actual de Kotlin Multiplatform para crear implementaciones específicas de plataforma con interfaces compartidas."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/expect-actual-post.png
draft: false
tags: 
- kotlin
- multiplatform
- kmp
---

### Aprovechando expect/actual en Kotlin Multiplatform para Implementaciones Nativas

Kotlin Multiplatform (KMP) ha surgido como una poderosa solución para compartir código entre diferentes plataformas, permitiendo al mismo tiempo implementaciones específicas de plataforma cuando sea necesario. En el centro de esta capacidad está el mecanismo expect/actual, que permite a los desarrolladores definir una API común en código compartido y proporcionar implementaciones específicas de plataforma. Este artículo explora cómo utilizar eficazmente expect/actual para crear aplicaciones multiplataforma robustas con implementaciones nativas.

---

#### Entendiendo expect/actual en Kotlin Multiplatform

El mecanismo expect/actual es el enfoque de Kotlin para manejar código específico de plataforma en un proyecto multiplataforma. Consta de dos componentes clave:

1. **Declaraciones expect**: Definen qué funcionalidad se requiere en el código común
2. **Implementaciones actual**: Proporcionan implementaciones específicas de plataforma de esa funcionalidad

```kotlin
// En código común
expect class PlatformDateFormatter() {
    fun format(date: Long): String
}

// En código específico de Android
actual class PlatformDateFormatter {
    actual fun format(date: Long): String {
        val dateFormat = SimpleDateFormat("yyyy-MM-dd HH:mm:ss", Locale.getDefault())
        return dateFormat.format(Date(date))
    }
}

// En código específico de iOS
actual class PlatformDateFormatter {
    actual fun format(date: Long): String {
        val dateFormatter = NSDateFormatter()
        dateFormatter.dateFormat = "yyyy-MM-dd HH:mm:ss"
        return dateFormatter.stringFromDate(NSDate(timeIntervalSince1970 = date / 1000.0))
    }
}
```

La declaración expect sirve como un contrato que debe ser cumplido por cada implementación específica de plataforma. Esto asegura que el código común pueda confiar en que cierta funcionalidad estará disponible, independientemente de la plataforma.

---

#### Ventajas de Usar expect/actual en KMP

El mecanismo expect/actual ofrece varios beneficios significativos para el desarrollo multiplataforma:

1. **Compartir Código con Optimizaciones Específicas de Plataforma**
   - Compartir lógica de negocio, modelos y algoritmos entre plataformas
   - Implementar optimizaciones específicas de plataforma donde sea necesario
   - Aprovechar las APIs nativas de la plataforma para un mejor rendimiento y experiencia de usuario

2. **Seguridad de Tipos Entre Plataformas**
   - El compilador asegura que todas las declaraciones expect tengan implementaciones actual correspondientes
   - La comprobación de tipos funciona a través de los límites de la plataforma
   - La refactorización es más segura ya que los cambios en las declaraciones expect deben reflejarse en todas las implementaciones actual

3. **Mejor Experiencia de Desarrollo**
   - Clara separación entre interfaces compartidas e implementaciones específicas de plataforma
   - Soporte del IDE para navegar entre declaraciones expect y actual
   - Mantenimiento más fácil ya que la API común se define en un solo lugar

4. **Camino de Adopción Gradual**
   - Comenzar con código específico de plataforma y gradualmente pasar a implementaciones compartidas
   - Elegir selectivamente qué componentes compartir y cuáles mantener específicos de plataforma
   - Integrar con bases de código existentes sin reescrituras completas

---

#### Ejemplos Prácticos de expect/actual en Acción

Exploremos algunos ejemplos prácticos de cómo se puede utilizar expect/actual en aplicaciones del mundo real.

##### Ejemplo 1: Almacenamiento Específico de Plataforma

```kotlin
// En commonMain
expect class LocalStorage {
    fun saveString(key: String, value: String)
    fun getString(key: String): String?
    fun clear()
}

// En androidMain
actual class LocalStorage {
    private val sharedPreferences = context.getSharedPreferences("app_prefs", Context.MODE_PRIVATE)

    actual fun saveString(key: String, value: String) {
        sharedPreferences.edit().putString(key, value).apply()
    }

    actual fun getString(key: String): String? {
        return sharedPreferences.getString(key, null)
    }

    actual fun clear() {
        sharedPreferences.edit().clear().apply()
    }
}

// En iosMain
actual class LocalStorage {
    private val userDefaults = NSUserDefaults.standardUserDefaults

    actual fun saveString(key: String, value: String) {
        userDefaults.setObject(value, key)
    }

    actual fun getString(key: String): String? {
        return userDefaults.stringForKey(key)
    }

    actual fun clear() {
        userDefaults.dictionaryRepresentation().keys.forEach {
            userDefaults.removeObjectForKey(it)
        }
    }
}
```

Este ejemplo demuestra cómo crear una interfaz de almacenamiento común mientras se aprovechan los mecanismos de almacenamiento específicos de plataforma (SharedPreferences en Android y NSUserDefaults en iOS).

---

##### Ejemplo 2: Monitoreo de Conectividad de Red

```kotlin
// En commonMain
expect class NetworkMonitor() {
    fun startMonitoring(onConnectivityChange: (Boolean) -> Unit)
    fun stopMonitoring()
}

// En androidMain
actual class NetworkMonitor {
    private var connectivityManager: ConnectivityManager? = null
    private var networkCallback: ConnectivityManager.NetworkCallback? = null

    actual fun startMonitoring(onConnectivityChange: (Boolean) -> Unit) {
        connectivityManager = context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
        networkCallback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) {
                onConnectivityChange(true)
            }

            override fun onLost(network: Network) {
                onConnectivityChange(false)
            }
        }

        val networkRequest = NetworkRequest.Builder().build()
        connectivityManager?.registerNetworkCallback(networkRequest, networkCallback!!)
    }

    actual fun stopMonitoring() {
        networkCallback?.let { callback ->
            connectivityManager?.unregisterNetworkCallback(callback)
        }
    }
}

// En iosMain
actual class NetworkMonitor {
    private var reachability: SCNetworkReachability? = null

    actual fun startMonitoring(onConnectivityChange: (Boolean) -> Unit) {
        reachability = SCNetworkReachabilityCreateWithName(null, "www.apple.com")

        SCNetworkReachabilitySetCallback(reachability) { _, flags, _ ->
            val isReachable = flags.contains(SCNetworkReachabilityFlags.Reachable)
            onConnectivityChange(isReachable)
        }

        SCNetworkReachabilityScheduleWithRunLoop(reachability, CFRunLoopGetMain(), kCFRunLoopCommonModes)
    }

    actual fun stopMonitoring() {
        reachability?.let { reach ->
            SCNetworkReachabilityUnscheduleFromRunLoop(reach, CFRunLoopGetMain(), kCFRunLoopCommonModes)
        }
    }
}
```

Este ejemplo muestra cómo monitorear la conectividad de red utilizando APIs específicas de plataforma mientras se mantiene una interfaz consistente en el código compartido.

---

#### Mejores Prácticas para Usar expect/actual

Para aprovechar al máximo el mecanismo expect/actual, considera estas mejores prácticas:

1. **Mantén las declaraciones expect mínimas**
   - Define solo lo que es necesario para que el código común funcione
   - Evita exponer detalles específicos de plataforma en la declaración expect
   - Usa interfaces cuando sea posible para definir comportamiento en lugar de implementación

2. **Usa expect/actual estratégicamente**
   - No todo necesita ser una declaración expect/actual
   - Considera alternativas como implementaciones de interfaces para casos más simples
   - Reserva expect/actual para casos donde necesites integración profunda con la plataforma

3. **Organiza tu código efectivamente**
   - Sigue la estructura estándar de conjuntos de fuentes de KMP (commonMain, androidMain, iosMain, etc.)
   - Agrupa declaraciones expect/actual relacionadas
   - Considera usar archivos separados para implementaciones expect/actual complejas

4. **Maneja características específicas de plataforma con elegancia**
   - Usa expect/actual para proporcionar alternativas para características no disponibles en todas las plataformas
   - Considera funcionalidad opcional que se degrade con elegancia
   - Documenta claramente las limitaciones específicas de plataforma

5. **Prueba tanto el código común como el específico de plataforma**
   - Escribe pruebas para la interfaz común en commonTest
   - Crea pruebas específicas de plataforma para implementaciones actual
   - Usa mocks o dobles de prueba cuando sea apropiado

---

#### Patrones Avanzados con expect/actual

A medida que te sientas más cómodo con expect/actual, puedes aprovechar patrones más avanzados:

##### Delegación a Bibliotecas Específicas de Plataforma

```kotlin
// En commonMain
expect class JsonParser() {
    fun parse(jsonString: String): Map<String, Any?>
    fun stringify(map: Map<String, Any?>): String
}

// En androidMain
actual class JsonParser {
    private val gson = Gson()

    actual fun parse(jsonString: String): Map<String, Any?> {
        val type = object : TypeToken<Map<String, Any?>>() {}.type
        return gson.fromJson(jsonString, type)
    }

    actual fun stringify(map: Map<String, Any?>): String {
        return gson.toJson(map)
    }
}

// En iosMain
actual class JsonParser {
    actual fun parse(jsonString: String): Map<String, Any?> {
        val nsString = NSString.create(string = jsonString)
        val data = nsString.dataUsingEncoding(NSUTF8StringEncoding)
        val nsObject = NSJSONSerialization.JSONObjectWithData(data!!, 0, null)
        return nsObject as Map<String, Any?>
    }

    actual fun stringify(map: Map<String, Any?>): String {
        val nsData = NSJSONSerialization.dataWithJSONObject(map, 0, null)
        return NSString.create(data = nsData!!, encoding = NSUTF8StringEncoding) as String
    }
}
```

Este patrón te permite aprovechar bibliotecas específicas de plataforma (Gson para Android y NSJSONSerialization para iOS) mientras mantienes una API consistente.

---

### Conclusión

El mecanismo expect/actual es una piedra angular del desarrollo de Kotlin Multiplatform, permitiendo a los desarrolladores escribir código compartido mientras aprovechan las capacidades específicas de plataforma. Al definir una interfaz común con declaraciones expect y proporcionar implementaciones específicas de plataforma con declaraciones actual, puedes crear aplicaciones que comparten lógica de negocio mientras aprovechan las características nativas de la plataforma.

A medida que construyes aplicaciones multiplataforma, recuerda que expect/actual es solo una herramienta en tu kit de herramientas de KMP. Úsalo juiciosamente, junto con otros enfoques como interfaces y clases abstractas, para crear el equilibrio adecuado entre compartir código y optimización específica de plataforma.

Con el enfoque correcto para expect/actual, puedes reducir significativamente la duplicación de código, mejorar la mantenibilidad y entregar aplicaciones de alta calidad en múltiples plataformas sin sacrificar la experiencia nativa que los usuarios esperan.
