---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Reducción del Tamaño de Aplicaciones: Proguard, R8, App Bundles y Reducción de Recursos"
date: 2025-06-27T08:00:00+01:00
description: "Una guía completa para reducir el tamaño de aplicaciones Android utilizando Proguard, R8, App Bundles y técnicas de reducción de recursos para un mejor rendimiento y experiencia de usuario."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/r8.png
draft: false
tags: 
- android
- optimización
- proguard
- r8
- app-bundles
---

### Reducción del Tamaño de Aplicaciones: Proguard, R8, App Bundles y Reducción de Recursos

A medida que las aplicaciones móviles crecen en complejidad y funcionalidades, gestionar su tamaño se vuelve cada vez más importante. Las aplicaciones grandes provocan mayores tasas de abandono durante la instalación, consumen más almacenamiento del dispositivo y a menudo resultan en un rendimiento más lento. Esta guía completa explora varias técnicas para reducir el tamaño de tu aplicación Android, incluyendo la reducción de código con Proguard y R8, distribución moderna con App Bundles, y estrategias de optimización de recursos.

---

#### Entendiendo los Componentes del Tamaño de una Aplicación

Antes de adentrarnos en las técnicas de optimización, es importante entender qué contribuye al tamaño de tu aplicación:

1. **Código**: Bytecode de Java/Kotlin, incluyendo el código de tu aplicación y bibliotecas
2. **Recursos**: Imágenes, layouts, animaciones y otros assets
3. **Bibliotecas nativas**: Código C/C++ compilado para diferentes arquitecturas
4. **Assets**: Archivos sin procesar incluidos en tu aplicación

Cada componente requiere diferentes estrategias de optimización, y un enfoque integral aborda todas estas áreas.

```
// Un desglose típico del tamaño de una aplicación podría ser:
// - Código (DEX): 30-40%
// - Recursos: 30-40%
// - Bibliotecas nativas: 15-25%
// - Assets: 5-15%
```

---

#### Reducción de Código con Proguard y R8

La reducción de código es una de las formas más efectivas de reducir el tamaño de la aplicación al eliminar código no utilizado y optimizar lo que queda.

##### Proguard: El Enfoque Tradicional

Proguard ha sido la herramienta estándar para la reducción de código en Android durante muchos años. Realiza varias optimizaciones:

1. **Reducción**: Elimina clases, campos, métodos y atributos no utilizados
2. **Optimización**: Optimiza el bytecode y elimina rutas de código no utilizadas
3. **Ofuscación**: Renombra clases, campos y métodos con nombres más cortos
4. **Preverificación**: Añade información de preverificación para simplificar el proceso de verificación en tiempo de ejecución

Para habilitar Proguard en el archivo build.gradle de tu aplicación, necesitas establecer minifyEnabled a true y especificar los archivos de reglas de Proguard:

```
android {
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro"
        }
    }
}
```

Un archivo de configuración básico de Proguard podría incluir:

```
# Mantener la clase de aplicación
-keep public class com.example.MyApplication

# Mantener todas las clases en el paquete androidx
-keep class androidx.** { *; }

# Mantener todas las clases que tienen la anotación @Keep
-keep class * {
    @androidx.annotation.Keep *;
}
-keepclasseswithmembers class * {
    @androidx.annotation.Keep <methods>;
}
-keepclasseswithmembers class * {
    @androidx.annotation.Keep <fields>;
}
```

###### Reglas Comunes de Proguard para Bibliotecas Populares

Cuando utilizas bibliotecas de terceros, a menudo necesitas reglas específicas de Proguard para asegurar que funcionen correctamente después de la reducción. Aquí hay ejemplos para algunas bibliotecas populares:

**Retrofit y OkHttp:**
```
# Retrofit
-keepattributes Signature
-keepattributes *Annotation*
-keep class retrofit2.** { *; }
-keepclasseswithmembers class * {
    @retrofit2.http.* <methods>;
}

# OkHttp
-keepattributes Signature
-keepattributes *Annotation*
-keep class okhttp3.** { *; }
-keep interface okhttp3.** { *; }
-dontwarn okhttp3.**
-dontwarn okio.**
```

**Gson:**
```
# Gson
-keepattributes Signature
-keepattributes *Annotation*
-keep class com.google.gson.** { *; }
-keep class * implements com.google.gson.TypeAdapter
-keep class * implements com.google.gson.TypeAdapterFactory
-keep class * implements com.google.gson.JsonSerializer
-keep class * implements com.google.gson.JsonDeserializer
-keepclassmembers,allowobfuscation class * {
  @com.google.gson.annotations.SerializedName <fields>;
}
```

**Glide:**
```
# Glide
-keep public class * implements com.bumptech.glide.module.GlideModule
-keep class * extends com.bumptech.glide.module.AppGlideModule {
 <init>(...);
}
-keep public enum com.bumptech.glide.load.ImageHeaderParser$** {
  **[] $VALUES;
  public *;
}
```

###### Solución de Problemas Comunes con Proguard

Cuando utilizas Proguard, podrías encontrarte con algunos problemas comunes:

1. **ClassNotFoundException o NoSuchMethodError**: Esto suele ocurrir cuando una clase o método utilizado a través de reflexión es eliminado o renombrado. Solución: Añade reglas keep para las clases afectadas.

2. **Falta de serialización/deserialización**: Las bibliotecas de análisis JSON como Gson dependen de los nombres de clases y campos. Solución: Añade reglas para preservar los nombres de las clases del modelo.

3. **Fallos en bibliotecas de terceros**: Algunas bibliotecas no son totalmente compatibles con la ofuscación. Solución: Consulta la documentación de la biblioteca para conocer las reglas de Proguard recomendadas.

4. **Depuración de fallos ofuscados**: Utiliza el archivo mapping.txt generado durante la compilación para desofuscar los rastros de pila:

```
// Añade esto a tu build.gradle para mantener el archivo de mapeo
android {
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro"
            testProguardFiles getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro"
        }
    }
}
```

##### R8: El Sucesor Moderno

R8 es el compilador predeterminado actual de Android que combina las capacidades de Proguard con optimizaciones adicionales:

1. **Inlining más agresivo**: Integra más métodos para reducir el recuento de métodos
2. **Fusión de clases**: Combina clases con funcionalidad similar
3. **Optimización de enumeraciones**: Convierte enumeraciones a enteros cuando es posible
4. **Optimizaciones específicas de Kotlin**: Mejor manejo de características específicas de Kotlin
5. **Tree shaking**: Eliminación más agresiva de código no utilizado
6. **Desazucarización**: Permite usar características más nuevas del lenguaje Java en versiones antiguas de Android

R8 está habilitado por defecto en Android Gradle Plugin 3.4.0 y versiones superiores. Puedes habilitar la optimización de R8 en modo completo configurando la desazucarización de la biblioteca principal en tu archivo build.gradle:

```
android {
    compileOptions {
        // Habilitar la desazucarización de la biblioteca principal
        coreLibraryDesugaringEnabled true
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }

    buildTypes {
        release {
            // Habilitar R8 en modo completo
            minifyEnabled true
            proguardFiles getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro"
        }
    }
}

dependencies {
    // Añadir la dependencia de desazucarización
    coreLibraryDesugaring "com.android.tools:desugar_jdk_libs:1.1.5"
}
```

###### Comparando Proguard y R8

Aunque R8 está diseñado para ser un reemplazo directo de Proguard, hay algunas diferencias clave:

| Característica | Proguard | R8 |
|----------------|----------|-----|
| Reducción de código | Sí | Sí (más agresivo) |
| Ofuscación | Sí | Sí |
| Optimización | Sí | Sí (más agresivo) |
| Soporte para Kotlin | Limitado | Mejorado |
| Velocidad de compilación | Más lento | Más rápido |
| Características de Java 8+ | No | Sí (con desazucarización) |
| Configuración | Compleja | Más simple (usa reglas de Proguard) |

###### Reglas Específicas de R8

Aunque R8 utiliza reglas de Proguard, hay algunas reglas específicas de R8 que puedes usar:

```
# Mantener la clase pero permitir que sus métodos sean eliminados si no se usan
-keepclassmembers class com.example.MyClass { *; }

# Mantener la clase y todos sus miembros, pero permitir la optimización
-keepclasseswithmembers class com.example.MyClass { *; }

# Específico de R8: Permitir inlining agresivo de un método
-alwaysinline class com.example.MyClass { 
    void myMethod(); 
}

# Específico de R8: Prevenir inlining de un método
-neverinline class com.example.MyClass { 
    void myMethod(); 
}
```

###### Depuración de Problemas con R8

Al solucionar problemas con R8, estos enfoques pueden ser útiles:

1. **Inspeccionar la salida de R8**: Usa la bandera `--debug` para ver qué está haciendo R8:

```
android.buildTypes.release.proguardFiles += file("debug-rules.pro")
```

Con debug-rules.pro conteniendo:
```
-printusage unused.txt
-printseeds seeds.txt
-printmapping mapping.txt
-verbose
```

2. **Deshabilitar optimizaciones específicas**: Si sospechas que una optimización particular está causando problemas:

```
-optimizations !class/merging/vertical,!field/*,!method/inlining/*
```

3. **Usar la herramienta de reescritura de R8**: Para desofuscar rastros de pila de fallos en producción:

```
java -jar r8.jar retrace --mapping mapping.txt obfuscated_stacktrace.txt
```

---

#### Reducción y Optimización de Recursos

Los recursos a menudo representan una parte significativa del tamaño de la aplicación. Aquí hay estrategias para optimizarlos:

##### Habilitar la Reducción de Recursos

La reducción de recursos elimina los recursos no utilizados de tu aplicación empaquetada. Puedes habilitarla en tu archivo build.gradle estableciendo shrinkResources a true junto con minifyEnabled.

##### Optimización de Imágenes

1. **Usar formato WebP**: Convierte imágenes PNG a WebP para una mejor compresión

   WebP proporciona una compresión superior en comparación con PNG y JPEG mientras mantiene buena calidad. Puedes convertir tus imágenes usando Android Studio:

   - Haz clic derecho en un recurso drawable
   - Selecciona "Convert to WebP"
   - Configura las opciones de codificación (con pérdida o sin pérdida)
   - Revisa el ahorro de tamaño y la calidad

   Para conversión por lotes, puedes usar la herramienta de línea de comandos:

   ```
   cwebp -q 80 input.png -o output.webp
   ```

2. **Drawables vectoriales**: Usa drawables vectoriales en lugar de múltiples archivos PNG para diferentes densidades:

```xml
<!-- vector_drawable.xml -->
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="24dp"
    android:height="24dp"
    android:viewportWidth="24"
    android:viewportHeight="24">
    <path
        android:fillColor="#FF000000"
        android:pathData="M12,2C6.48,2 2,6.48 2,12s4.48,10 10,10 10,-4.48 10,-10S17.52,2 12,2z"/>
</vector>
```

   Los drawables vectoriales son independientes de la resolución y típicamente mucho más pequeños que sus contrapartes bitmap. Para asegurar la compatibilidad con versiones antiguas de Android, añade esto a tu build.gradle:

   ```
   android {
       defaultConfig {
           vectorDrawables.useSupportLibrary = true
       }
   }
   ```

3. **Recursos específicos de densidad**: Proporciona solo los recursos de densidad necesarios configurando resConfigs en tu archivo build.gradle

   ```
   android {
       defaultConfig {
           // Incluye solo los buckets de densidad que necesitas
           resConfigs "nodpi", "hdpi", "xhdpi", "xxhdpi"
       }
   }
   ```

4. **Comprimir archivos PNG**: Usa herramientas como pngquant o TinyPNG para optimizar archivos PNG sin pérdida significativa de calidad:

   ```
   pngquant --quality=65-80 image.png
   ```

5. **Carga diferida para imágenes grandes**: En lugar de incluir imágenes grandes en tu aplicación, considera cargarlas bajo demanda desde un servidor.

6. **Tintado de drawables**: Usa una sola imagen en escala de grises y aplica tintes programáticamente en lugar de incluir múltiples versiones coloreadas.

   ```xml
   <!-- En archivo de layout -->
   <ImageView
       android:id="@+id/icon"
       android:layout_width="24dp"
       android:layout_height="24dp"
       android:src="@drawable/icon_grayscale"
       app:tint="@color/colorPrimary" />
   ```

##### Recursos de Idioma

Limita los idiomas incluidos para reducir el tamaño del APK configurando resConfigs en tu archivo build.gradle para incluir solo los idiomas que tu aplicación soporta:

```
android {
    defaultConfig {
        // Incluye solo los idiomas que soportas
        resConfigs "en", "es", "fr", "de"
    }
}
```

Esta configuración elimina los archivos de recursos para idiomas no soportados, lo que puede reducir significativamente el tamaño de la aplicación, especialmente para aplicaciones con muchos recursos de cadenas.

##### Optimización de Layouts

1. **Aplanar jerarquías de vistas**: Simplifica tus layouts para reducir el número de ViewGroups anidados:

   - Usa ConstraintLayout en lugar de LinearLayouts anidados
   - Evita vistas contenedoras innecesarias
   - Usa el Layout Inspector para identificar jerarquías de vistas excesivamente complejas

2. **Reutiliza layouts con <include>**: Extrae componentes UI comunes en archivos de layout separados e inclúyelos:

   ```xml
   <include layout="@layout/common_header" />
   ```

3. **Usa estilos y temas**: Define estilos para atributos de vista comunes en lugar de repetirlos:

   ```xml
   <style name="AppButton">
       <item name="android:background">@drawable/button_bg</item>
       <item name="android:textColor">@color/button_text</item>
       <item name="android:padding">8dp</item>
   </style>
   ```

4. **Elimina recursos no utilizados**: Usa la herramienta Lint para identificar recursos no utilizados:

   - En Android Studio, ve a Analyze > Inspect Code
   - Busca "Unused resources" en los resultados
   - Elimina o marca los recursos como tools:keep si es necesario

---

#### Técnicas Avanzadas

##### App Bundles y Entrega Dinámica

Los Android App Bundles (AAB) representan una mejora significativa sobre los APKs tradicionales para la distribución de aplicaciones. Optimizan la entrega proporcionando solo el código y los recursos necesarios para cada dispositivo. Aquí te explicamos por qué deberías considerar usar App Bundles en lugar de APKs:

1. **Tamaños de descarga más pequeños**: Los App Bundles generan APKs optimizados para cada configuración de dispositivo, reduciendo los tamaños de descarga hasta en un 15% en comparación con un APK universal. Los usuarios solo descargan el código y los recursos que necesitan para su dispositivo específico.

2. **Optimización automática de recursos de idioma**: Los App Bundles entregan solo los recursos de idioma que coinciden con la configuración del dispositivo del usuario, eliminando traducciones innecesarias.

3. **Optimización de densidad de pantalla**: Solo se entregan los recursos de densidad de pantalla apropiados a cada dispositivo, evitando la inclusión de recursos gráficos no utilizados.

4. **Entrega de código específico para cada arquitectura**: Los usuarios reciben solo las bibliotecas nativas que coinciden con la arquitectura de CPU de su dispositivo (ARM, ARM64, x86, etc.) en lugar de todas las arquitecturas.

5. **Entrega dinámica de características**: Los App Bundles permiten la entrega bajo demanda de características que no son necesarias para el lanzamiento inicial de la aplicación, reduciendo aún más el tamaño de descarga inicial.

6. **Publicación simplificada de aplicaciones**: Los desarrolladores mantienen un solo artefacto para todas las configuraciones de dispositivos, y Google Play se encarga de la generación de APKs optimizados.

7. **Play Feature Delivery**: Permite opciones avanzadas de entrega como entrega condicional, entrega instantánea y entrega bajo demanda de características de la aplicación.

8. **Actualizaciones más sencillas**: Paquetes de actualización más pequeños, ya que los usuarios solo necesitan descargar los cambios relevantes para la configuración de su dispositivo.

Puedes configurar divisiones de idioma, densidad y ABI en tu archivo build.gradle para aprovechar al máximo los App Bundles:

##### Características bajo Demanda con Módulos de Características Dinámicas

Divide tu aplicación en módulos base y de características que pueden descargarse bajo demanda. Esto requiere configurar módulos de características dinámicas en tu proyecto.

##### Optimización de Bibliotecas Nativas

Reduce el tamaño de las bibliotecas nativas incluyendo solo ABIs específicos y habilitando la compresión de bibliotecas nativas en tu archivo build.gradle.

---

#### Medición y Monitoreo del Tamaño de la Aplicación

Para optimizar eficazmente el tamaño de la aplicación, necesitas medirlo y monitorearlo durante todo el desarrollo:

1. **APK Analyzer**: Utiliza el APK Analyzer integrado en Android Studio para inspeccionar los contenidos de tu APK
2. **Tarea personalizada de Gradle**: Crea una tarea para informar sobre el tamaño de la aplicación después de cada compilación
3. **Integración CI**: Añade comprobaciones de tamaño de aplicación a tu pipeline de CI

Puedes crear una tarea personalizada de Gradle para informar sobre el tamaño de la aplicación después de cada compilación, lo que ayuda a rastrear los cambios de tamaño a lo largo del tiempo.

---

### Conclusión

Reducir el tamaño de la aplicación es un desafío multifacético que requiere atención al código, los recursos y los mecanismos de entrega. Al implementar las técnicas discutidas en esta publicación—reducción de código con Proguard y R8, optimización de recursos, App Bundles y otros mecanismos de entrega avanzados—puedes reducir significativamente el tamaño de tu aplicación mientras mantienes o incluso mejoras su rendimiento.

Recuerda que la optimización del tamaño de la aplicación es un proceso continuo. Analiza regularmente la composición de tu aplicación, monitorea los cambios de tamaño con cada versión y refina continuamente tus estrategias de optimización. Migrar de APKs tradicionales a App Bundles puede proporcionar beneficios inmediatos con un esfuerzo mínimo, convirtiéndolo en uno de los pasos más efectivos que puedes tomar. El resultado será una aplicación más eficiente que proporciona una mejor experiencia de usuario a través de descargas más rápidas, requisitos de almacenamiento reducidos y un mejor rendimiento en tiempo de ejecución.
