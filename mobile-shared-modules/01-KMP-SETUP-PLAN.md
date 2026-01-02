# Plan de Setup: edugo-kmp-shared

## Resumen Ejecutivo

Este documento define el plan completo para crear el proyecto **edugo-kmp-shared**, 
un proyecto Kotlin Multiplatform con múltiples módulos independientes equivalente a `edugo-shared` de Go.

**Plataformas objetivo**: 
- Android API 35+
- JVM Desktop
- Kotlin/JS (Web) - con preparación para WASM futuro

**Kotlin Version**: 2.1.0+
**KMP**: Estable desde noviembre 2023

---

## 1. Versiones de Dependencias (libs.versions.toml)

```toml
# gradle/libs.versions.toml

[versions]
kotlin = "2.1.0"
kotlinx-coroutines = "1.9.0"
kotlinx-serialization = "1.7.3"
kotlinx-datetime = "0.6.1"
ktor = "3.0.2"
kermit = "2.0.4"
multiplatform-settings = "1.2.0"
uuid = "0.8.4"
android-gradle = "8.7.0"
android-compileSdk = "35"
android-minSdk = "26"
android-targetSdk = "35"

[libraries]
# Coroutines
kotlinx-coroutines-core = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-core", version.ref = "kotlinx-coroutines" }
kotlinx-coroutines-android = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-android", version.ref = "kotlinx-coroutines" }
kotlinx-coroutines-test = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-test", version.ref = "kotlinx-coroutines" }

# Serialization
kotlinx-serialization-json = { module = "org.jetbrains.kotlinx:kotlinx-serialization-json", version.ref = "kotlinx-serialization" }

# DateTime
kotlinx-datetime = { module = "org.jetbrains.kotlinx:kotlinx-datetime", version.ref = "kotlinx-datetime" }

# Ktor Client
ktor-client-core = { module = "io.ktor:ktor-client-core", version.ref = "ktor" }
ktor-client-content-negotiation = { module = "io.ktor:ktor-client-content-negotiation", version.ref = "ktor" }
ktor-serialization-kotlinx-json = { module = "io.ktor:ktor-serialization-kotlinx-json", version.ref = "ktor" }
ktor-client-logging = { module = "io.ktor:ktor-client-logging", version.ref = "ktor" }
ktor-client-okhttp = { module = "io.ktor:ktor-client-okhttp", version.ref = "ktor" }
ktor-client-darwin = { module = "io.ktor:ktor-client-darwin", version.ref = "ktor" }
ktor-client-java = { module = "io.ktor:ktor-client-java", version.ref = "ktor" }
ktor-client-js = { module = "io.ktor:ktor-client-js", version.ref = "ktor" }

# Logging
kermit = { module = "co.touchlab:kermit", version.ref = "kermit" }

# Settings/Storage
multiplatform-settings = { module = "com.russhwolf:multiplatform-settings", version.ref = "multiplatform-settings" }
multiplatform-settings-coroutines = { module = "com.russhwolf:multiplatform-settings-coroutines", version.ref = "multiplatform-settings" }

# UUID
uuid = { module = "com.benasher44:uuid", version.ref = "uuid" }

# Testing
kotlin-test = { module = "org.jetbrains.kotlin:kotlin-test", version.ref = "kotlin" }

[plugins]
kotlin-multiplatform = { id = "org.jetbrains.kotlin.multiplatform", version.ref = "kotlin" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
android-library = { id = "com.android.library", version.ref = "android-gradle" }
```

---

## 2. Estructura del Proyecto

```
edugo-kmp-shared/
├── settings.gradle.kts
├── build.gradle.kts
├── gradle.properties
├── gradle/
│   ├── libs.versions.toml
│   └── wrapper/
├── build-logic/
│   ├── settings.gradle.kts
│   ├── build.gradle.kts
│   └── src/main/kotlin/
│       └── kmp.library.gradle.kts    # Convention plugin
├── .github/
│   └── workflows/
│       ├── ci.yml
│       ├── release.yml
│       └── publish.yml
├── README.md
├── CHANGELOG.md
│
├── modules/
│   ├── common/                       # EduGoCommon
│   │   ├── build.gradle.kts
│   │   └── src/
│   │       ├── commonMain/kotlin/
│   │       ├── commonTest/kotlin/
│   │       ├── androidMain/kotlin/
│   │       ├── jvmMain/kotlin/
│   │       └── jsMain/kotlin/
│   │
│   ├── logger/                       # EduGoLogger
│   │   ├── build.gradle.kts
│   │   └── src/
│   │       ├── commonMain/kotlin/
│   │       ├── androidMain/kotlin/
│   │       ├── jvmMain/kotlin/
│   │       └── jsMain/kotlin/
│   │
│   ├── network/                      # EduGoNetwork
│   │   ├── build.gradle.kts
│   │   └── src/
│   │       ├── commonMain/kotlin/
│   │       ├── androidMain/kotlin/
│   │       ├── jvmMain/kotlin/
│   │       └── jsMain/kotlin/
│   │
│   ├── storage/                      # EduGoStorage
│   │   ├── build.gradle.kts
│   │   └── src/
│   │       ├── commonMain/kotlin/
│   │       ├── androidMain/kotlin/
│   │       ├── jvmMain/kotlin/
│   │       └── jsMain/kotlin/
│   │
│   ├── auth/                         # EduGoAuth
│   │   ├── build.gradle.kts
│   │   └── src/
│   │       ├── commonMain/kotlin/
│   │       ├── commonTest/kotlin/
│   │       └── ...
│   │
│   ├── analytics/                    # EduGoAnalytics
│   │   ├── build.gradle.kts
│   │   └── src/
│   │       └── ...
│   │
│   ├── roles/                        # EduGoRoles
│   │   ├── build.gradle.kts
│   │   └── src/
│   │       └── commonMain/kotlin/    # Solo commonMain
│   │
│   ├── models/                       # EduGoModels
│   │   ├── build.gradle.kts
│   │   └── src/
│   │       └── commonMain/kotlin/
│   │
│   └── api/                          # EduGoAPI
│       ├── build.gradle.kts
│       └── src/
│           └── ...
```

---

## 3. settings.gradle.kts

```kotlin
pluginManagement {
    includeBuild("build-logic")
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}

dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
    }
}

rootProject.name = "edugo-kmp-shared"

include(":modules:common")
include(":modules:logger")
include(":modules:network")
include(":modules:storage")
include(":modules:auth")
include(":modules:analytics")
include(":modules:roles")
include(":modules:models")
include(":modules:api")
```

---

## 4. Convention Plugin (build-logic)

### build-logic/src/main/kotlin/kmp.library.gradle.kts

```kotlin
import org.jetbrains.kotlin.gradle.dsl.JvmTarget

plugins {
    id("org.jetbrains.kotlin.multiplatform")
    id("com.android.library")
}

kotlin {
    // Targets
    androidTarget {
        compilations.all {
            compileTaskProvider.configure {
                compilerOptions {
                    jvmTarget.set(JvmTarget.JVM_17)
                }
            }
        }
    }
    
    jvm("desktop") {
        compilations.all {
            compileTaskProvider.configure {
                compilerOptions {
                    jvmTarget.set(JvmTarget.JVM_17)
                }
            }
        }
    }
    
    js(IR) {
        browser {
            testTask {
                useKarma {
                    useChromeHeadless()
                }
            }
        }
        binaries.library()
    }
    
    // Prepare for future WASM support
    // wasmJs { browser() }
    
    // Source sets
    sourceSets {
        val commonMain by getting
        val commonTest by getting {
            dependencies {
                implementation(kotlin("test"))
            }
        }
        
        val androidMain by getting
        val desktopMain by getting
        val jsMain by getting
    }
}

android {
    namespace = "com.edugo.shared.${project.name}"
    compileSdk = 35
    
    defaultConfig {
        minSdk = 26
    }
    
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
}
```

---

## 5. Ejemplos de Código por Módulo

### 5.1 modules/common

```kotlin
// src/commonMain/kotlin/com/edugo/shared/common/errors/AppError.kt
package com.edugo.shared.common.errors

import kotlinx.serialization.Serializable

@Serializable
data class AppError(
    val code: ErrorCode,
    val message: String,
    val details: Map<String, String>? = null
) : Exception(message)

@Serializable
enum class ErrorCode {
    // Auth
    UNAUTHORIZED,
    TOKEN_EXPIRED,
    INVALID_CREDENTIALS,
    
    // Network
    NETWORK_ERROR,
    TIMEOUT,
    SERVER_ERROR,
    
    // Validation
    VALIDATION_FAILED,
    NOT_FOUND,
    
    // Storage
    STORAGE_FAILED,
    
    // General
    UNKNOWN
}

// Extension functions
fun ErrorCode.toAppError(message: String): AppError = AppError(this, message)
```

### 5.2 modules/logger

```kotlin
// src/commonMain/kotlin/com/edugo/shared/logger/Logger.kt
package com.edugo.shared.logger

interface Logger {
    fun debug(message: String, tag: String? = null)
    fun info(message: String, tag: String? = null)
    fun warn(message: String, tag: String? = null)
    fun error(message: String, throwable: Throwable? = null, tag: String? = null)
}

enum class LogLevel {
    DEBUG, INFO, WARN, ERROR
}

// src/commonMain/kotlin/com/edugo/shared/logger/KermitLogger.kt
package com.edugo.shared.logger

import co.touchlab.kermit.Logger as Kermit

class KermitLogger(
    private val tag: String = "EduGo"
) : Logger {
    
    private val kermit = Kermit.withTag(tag)
    
    override fun debug(message: String, tag: String?) {
        if (tag != null) {
            Kermit.withTag(tag).d { message }
        } else {
            kermit.d { message }
        }
    }
    
    override fun info(message: String, tag: String?) {
        if (tag != null) {
            Kermit.withTag(tag).i { message }
        } else {
            kermit.i { message }
        }
    }
    
    override fun warn(message: String, tag: String?) {
        if (tag != null) {
            Kermit.withTag(tag).w { message }
        } else {
            kermit.w { message }
        }
    }
    
    override fun error(message: String, throwable: Throwable?, tag: String?) {
        if (tag != null) {
            Kermit.withTag(tag).e(throwable) { message }
        } else {
            kermit.e(throwable) { message }
        }
    }
}
```

### 5.3 modules/network

```kotlin
// src/commonMain/kotlin/com/edugo/shared/network/HttpClient.kt
package com.edugo.shared.network

import io.ktor.client.*
import io.ktor.client.call.*
import io.ktor.client.plugins.*
import io.ktor.client.plugins.contentnegotiation.*
import io.ktor.client.plugins.logging.*
import io.ktor.client.request.*
import io.ktor.client.statement.*
import io.ktor.http.*
import io.ktor.serialization.kotlinx.json.*
import kotlinx.serialization.json.Json
import com.edugo.shared.common.errors.AppError
import com.edugo.shared.common.errors.ErrorCode

expect fun createPlatformHttpClient(): HttpClient

class EduGoHttpClient(
    private val baseUrl: String,
    private val logger: com.edugo.shared.logger.Logger
) {
    private val json = Json {
        ignoreUnknownKeys = true
        isLenient = true
        prettyPrint = false
    }
    
    private val client = createPlatformHttpClient().config {
        install(ContentNegotiation) {
            json(this@EduGoHttpClient.json)
        }
        
        install(Logging) {
            level = LogLevel.INFO
        }
        
        install(HttpTimeout) {
            requestTimeoutMillis = 30_000
            connectTimeoutMillis = 10_000
        }
        
        defaultRequest {
            url(baseUrl)
            contentType(ContentType.Application.Json)
        }
    }
    
    suspend inline fun <reified T> get(
        path: String,
        headers: Map<String, String> = emptyMap()
    ): T {
        return execute {
            client.get(path) {
                headers.forEach { (key, value) -> header(key, value) }
            }
        }
    }
    
    suspend inline fun <reified T, reified B> post(
        path: String,
        body: B,
        headers: Map<String, String> = emptyMap()
    ): T {
        return execute {
            client.post(path) {
                headers.forEach { (key, value) -> header(key, value) }
                setBody(body)
            }
        }
    }
    
    suspend inline fun <reified T> execute(
        block: () -> HttpResponse
    ): T {
        try {
            val response = block()
            
            if (!response.status.isSuccess()) {
                throw AppError(
                    code = when (response.status) {
                        HttpStatusCode.Unauthorized -> ErrorCode.UNAUTHORIZED
                        HttpStatusCode.NotFound -> ErrorCode.NOT_FOUND
                        else -> ErrorCode.SERVER_ERROR
                    },
                    message = "HTTP ${response.status.value}: ${response.status.description}"
                )
            }
            
            return response.body()
        } catch (e: Exception) {
            when (e) {
                is AppError -> throw e
                is HttpRequestTimeoutException -> throw AppError(
                    code = ErrorCode.TIMEOUT,
                    message = "Request timed out"
                )
                else -> throw AppError(
                    code = ErrorCode.NETWORK_ERROR,
                    message = e.message ?: "Network error"
                )
            }
        }
    }
    
    fun close() {
        client.close()
    }
}

// src/androidMain/kotlin/com/edugo/shared/network/HttpClient.android.kt
package com.edugo.shared.network

import io.ktor.client.*
import io.ktor.client.engine.okhttp.*

actual fun createPlatformHttpClient(): HttpClient = HttpClient(OkHttp)

// src/jvmMain/kotlin/com/edugo/shared/network/HttpClient.jvm.kt
package com.edugo.shared.network

import io.ktor.client.*
import io.ktor.client.engine.java.*

actual fun createPlatformHttpClient(): HttpClient = HttpClient(Java)

// src/jsMain/kotlin/com/edugo/shared/network/HttpClient.js.kt
package com.edugo.shared.network

import io.ktor.client.*
import io.ktor.client.engine.js.*

actual fun createPlatformHttpClient(): HttpClient = HttpClient(Js)
```

### 5.4 modules/storage

```kotlin
// src/commonMain/kotlin/com/edugo/shared/storage/SecureStorage.kt
package com.edugo.shared.storage

interface SecureStorage {
    suspend fun save(key: String, value: String)
    suspend fun load(key: String): String?
    suspend fun delete(key: String)
    suspend fun clear()
}

// src/commonMain/kotlin/com/edugo/shared/storage/SettingsStorage.kt
package com.edugo.shared.storage

import com.russhwolf.settings.Settings
import com.russhwolf.settings.coroutines.SuspendSettings
import com.russhwolf.settings.coroutines.toSuspendSettings

expect fun createSettings(): Settings

class SettingsStorage : SecureStorage {
    private val settings: SuspendSettings = createSettings().toSuspendSettings()
    
    override suspend fun save(key: String, value: String) {
        settings.putString(key, value)
    }
    
    override suspend fun load(key: String): String? {
        return settings.getStringOrNull(key)
    }
    
    override suspend fun delete(key: String) {
        settings.remove(key)
    }
    
    override suspend fun clear() {
        settings.clear()
    }
}

// src/androidMain/kotlin/com/edugo/shared/storage/Settings.android.kt
package com.edugo.shared.storage

import android.content.Context
import com.russhwolf.settings.Settings
import com.russhwolf.settings.SharedPreferencesSettings

// Context debe ser proporcionado por la aplicación
object AndroidContext {
    lateinit var context: Context
}

actual fun createSettings(): Settings {
    return SharedPreferencesSettings(
        AndroidContext.context.getSharedPreferences(
            "edugo_secure_prefs",
            Context.MODE_PRIVATE
        )
    )
}

// src/jvmMain/kotlin/com/edugo/shared/storage/Settings.jvm.kt
package com.edugo.shared.storage

import com.russhwolf.settings.PreferencesSettings
import com.russhwolf.settings.Settings
import java.util.prefs.Preferences

actual fun createSettings(): Settings {
    return PreferencesSettings(
        Preferences.userRoot().node("edugo")
    )
}

// src/jsMain/kotlin/com/edugo/shared/storage/Settings.js.kt
package com.edugo.shared.storage

import com.russhwolf.settings.Settings
import com.russhwolf.settings.StorageSettings

actual fun createSettings(): Settings {
    return StorageSettings()
}
```

### 5.5 modules/auth

```kotlin
// src/commonMain/kotlin/com/edugo/shared/auth/TokenManager.kt
package com.edugo.shared.auth

import com.edugo.shared.storage.SecureStorage
import com.edugo.shared.common.errors.AppError
import com.edugo.shared.common.errors.ErrorCode
import kotlinx.coroutines.sync.Mutex
import kotlinx.coroutines.sync.withLock
import kotlinx.serialization.Serializable
import kotlinx.serialization.json.Json

@Serializable
data class TokenInfo(
    val accessToken: String,
    val refreshToken: String,
    val expiresAt: Long // Unix timestamp
) {
    val isExpired: Boolean
        get() = kotlinx.datetime.Clock.System.now().epochSeconds > expiresAt
}

class TokenManager(
    private val storage: SecureStorage,
    private val refresher: suspend (String) -> TokenInfo
) {
    private val mutex = Mutex()
    private var cachedToken: TokenInfo? = null
    
    private val json = Json { ignoreUnknownKeys = true }
    
    suspend fun getAccessToken(): String = mutex.withLock {
        val token = cachedToken ?: loadFromStorage()
        
        if (token == null) {
            throw AppError(ErrorCode.UNAUTHORIZED, "No token available")
        }
        
        if (token.isExpired) {
            val newToken = refresher(token.refreshToken)
            saveToken(newToken)
            return newToken.accessToken
        }
        
        return token.accessToken
    }
    
    suspend fun saveToken(token: TokenInfo) = mutex.withLock {
        cachedToken = token
        storage.save("auth_token", json.encodeToString(TokenInfo.serializer(), token))
    }
    
    suspend fun clearToken() = mutex.withLock {
        cachedToken = null
        storage.delete("auth_token")
    }
    
    private suspend fun loadFromStorage(): TokenInfo? {
        val stored = storage.load("auth_token") ?: return null
        return try {
            json.decodeFromString(TokenInfo.serializer(), stored).also {
                cachedToken = it
            }
        } catch (e: Exception) {
            null
        }
    }
}

// src/commonMain/kotlin/com/edugo/shared/auth/JWTDecoder.kt
package com.edugo.shared.auth

import kotlinx.serialization.Serializable
import kotlinx.serialization.json.Json
import kotlin.io.encoding.Base64
import kotlin.io.encoding.ExperimentalEncodingApi

@Serializable
data class JWTPayload(
    val sub: String,
    val exp: Long,
    val iat: Long,
    val roles: List<String>? = null
) {
    val isExpired: Boolean
        get() = kotlinx.datetime.Clock.System.now().epochSeconds > exp
}

object JWTDecoder {
    private val json = Json { ignoreUnknownKeys = true }
    
    @OptIn(ExperimentalEncodingApi::class)
    fun decode(token: String): JWTPayload {
        val parts = token.split(".")
        require(parts.size == 3) { "Invalid JWT format" }
        
        val payloadBase64 = parts[1]
            .replace("-", "+")
            .replace("_", "/")
        
        // Pad if necessary
        val padded = payloadBase64.padEnd(
            (payloadBase64.length + 3) / 4 * 4, '='
        )
        
        val decoded = Base64.decode(padded)
        val payloadJson = decoded.decodeToString()
        
        return json.decodeFromString(JWTPayload.serializer(), payloadJson)
    }
}
```

### 5.6 modules/roles

```kotlin
// src/commonMain/kotlin/com/edugo/shared/roles/SystemRole.kt
package com.edugo.shared.roles

import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable

@Serializable
enum class SystemRole {
    @SerialName("student")
    STUDENT,
    
    @SerialName("teacher")
    TEACHER,
    
    @SerialName("guardian")
    GUARDIAN,
    
    @SerialName("school_admin")
    SCHOOL_ADMIN,
    
    @SerialName("system_admin")
    SYSTEM_ADMIN;
    
    val displayName: String
        get() = when (this) {
            STUDENT -> "Estudiante"
            TEACHER -> "Profesor"
            GUARDIAN -> "Acudiente"
            SCHOOL_ADMIN -> "Administrador de Escuela"
            SYSTEM_ADMIN -> "Administrador del Sistema"
        }
    
    val permissions: Set<Permission>
        get() = when (this) {
            STUDENT -> setOf(
                Permission.VIEW_MATERIALS,
                Permission.TAKE_QUIZZES,
                Permission.VIEW_PROGRESS
            )
            TEACHER -> setOf(
                Permission.VIEW_MATERIALS,
                Permission.UPLOAD_MATERIALS,
                Permission.CREATE_QUIZZES,
                Permission.VIEW_STUDENT_PROGRESS
            )
            GUARDIAN -> setOf(
                Permission.VIEW_PROGRESS,
                Permission.VIEW_STUDENT_INFO
            )
            SCHOOL_ADMIN -> setOf(
                Permission.MANAGE_SCHOOL,
                Permission.MANAGE_USERS,
                Permission.VIEW_REPORTS
            )
            SYSTEM_ADMIN -> Permission.entries.toSet()
        }
    
    fun hasPermission(permission: Permission): Boolean = permission in permissions
}

// src/commonMain/kotlin/com/edugo/shared/roles/Permission.kt
package com.edugo.shared.roles

enum class Permission {
    VIEW_MATERIALS,
    UPLOAD_MATERIALS,
    TAKE_QUIZZES,
    CREATE_QUIZZES,
    VIEW_PROGRESS,
    VIEW_STUDENT_PROGRESS,
    VIEW_STUDENT_INFO,
    MANAGE_SCHOOL,
    MANAGE_USERS,
    VIEW_REPORTS
}
```

### 5.7 modules/models

```kotlin
// src/commonMain/kotlin/com/edugo/shared/models/User.kt
package com.edugo.shared.models

import com.edugo.shared.roles.SystemRole
import kotlinx.datetime.Instant
import kotlinx.serialization.Serializable

@Serializable
data class User(
    val id: String,
    val email: String,
    val firstName: String,
    val lastName: String,
    val roles: List<SystemRole>,
    val createdAt: Instant,
    val updatedAt: Instant
) {
    val fullName: String get() = "$firstName $lastName"
    
    fun hasRole(role: SystemRole): Boolean = role in roles
    
    val primaryRole: SystemRole get() = roles.firstOrNull() ?: SystemRole.STUDENT
}

// src/commonMain/kotlin/com/edugo/shared/models/Material.kt
package com.edugo.shared.models

import kotlinx.datetime.Instant
import kotlinx.serialization.Serializable

@Serializable
data class Material(
    val id: String,
    val title: String,
    val description: String?,
    val fileUrl: String,
    val mimeType: String,
    val sizeBytes: Long,
    val version: Int,
    val createdAt: Instant,
    val updatedAt: Instant
)

@Serializable
data class MaterialProgress(
    val materialId: String,
    val userId: String,
    val progress: Float, // 0.0 to 1.0
    val lastViewedAt: Instant,
    val completedAt: Instant?
)
```

---

## 6. Patrón expect/actual

Para código específico de plataforma, usar el patrón `expect/actual`:

```kotlin
// commonMain
expect fun createPlatformHttpClient(): HttpClient
expect fun createSettings(): Settings

// androidMain
actual fun createPlatformHttpClient(): HttpClient = HttpClient(OkHttp)
actual fun createSettings(): Settings = SharedPreferencesSettings(...)

// jvmMain  
actual fun createPlatformHttpClient(): HttpClient = HttpClient(Java)
actual fun createSettings(): Settings = PreferencesSettings(...)

// jsMain
actual fun createPlatformHttpClient(): HttpClient = HttpClient(Js)
actual fun createSettings(): Settings = StorageSettings()
```

---

## 7. CI/CD con GitHub Actions

### .github/workflows/ci.yml

```yaml
name: CI

on:
  push:
    branches: [main, dev]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup JDK 17
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'
    
    - name: Setup Gradle
      uses: gradle/actions/setup-gradle@v3
    
    - name: Build
      run: ./gradlew build
    
    - name: Run Tests
      run: ./gradlew allTests
    
  android-instrumented:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup JDK 17
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'
    
    - name: Enable KVM
      run: |
        echo 'KERNEL=="kvm", GROUP="kvm", MODE="0666", OPTIONS+="static_node=kvm"' | sudo tee /etc/udev/rules.d/99-kvm4all.rules
        sudo udevadm control --reload-rules
        sudo udevadm trigger --name-match=kvm
    
    - name: Android Tests
      uses: reactivecircus/android-emulator-runner@v2
      with:
        api-level: 35
        arch: x86_64
        script: ./gradlew connectedAndroidTest

  js-tests:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup JDK 17
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'
    
    - name: JS Browser Tests
      run: ./gradlew jsBrowserTest
```

### .github/workflows/publish.yml

```yaml
name: Publish

on:
  push:
    tags:
      - 'v*'

jobs:
  publish:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup JDK 17
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'
    
    - name: Publish to Maven
      run: ./gradlew publish
      env:
        MAVEN_USERNAME: ${{ secrets.MAVEN_USERNAME }}
        MAVEN_PASSWORD: ${{ secrets.MAVEN_PASSWORD }}
```

---

## 8. Versionado

### Estrategia: Monorepo con Version Catalog

Todas las versiones se manejan en `gradle/libs.versions.toml`.

### Formato de Tags

- Global: `v1.0.0`
- Por módulo (opcional): `common/v1.0.0`, `auth/v1.0.0`

### Publicación

Publicar a:
- Maven Central (para librerías públicas)
- GitHub Packages (para uso interno)
- JitPack (alternativa simple)

---

## 9. Grafo de Dependencias

```
common (base - sin dependencias)
   ↑
   ├── logger (depende de: common)
   │      ↑
   │      └── network (depende de: common, logger)
   │             ↑
   ├── storage (depende de: common)
   │      ↑
   │      └── auth (depende de: common, storage, network)
   │             ↑
   ├── roles (depende de: common)
   │      ↑
   │      └── models (depende de: common, roles)
   │             ↑
   │             └── api (depende de: network, auth, models)
   │
   └── analytics (depende de: common, logger)
```

---

## 10. Decisiones de Web Target

### Kotlin/JS vs WASM

**Decisión actual: Kotlin/JS**

Razón: Kotlin/JS es estable y maduro. WASM está en Alpha/Beta.

**Preparación para WASM:**
- Mantener código en `commonMain` tanto como sea posible
- Evitar dependencias específicas de JS cuando existan alternativas multiplataforma
- Cuando WASM sea estable (estimado 2025-2026), agregar target:

```kotlin
// Futuro
wasmJs {
    browser()
    binaries.library()
}
```

---

## 11. Fuentes Consultadas

- [Kotlin Multiplatform Overview](https://kotlinlang.org/docs/multiplatform.html)
- [KMP Stable Announcement (Nov 2023)](https://blog.jetbrains.com/kotlin/2023/11/kotlin-multiplatform-stable/)
- [Ktor 3.0 Release](https://ktor.io/docs/releases.html)
- [Kermit Logging Library](https://github.com/touchlab/Kermit)
- [Multiplatform Settings](https://github.com/russhwolf/multiplatform-settings)
- [kotlinx.datetime](https://github.com/Kotlin/kotlinx-datetime)
- [KMP Gradle Plugin Best Practices](https://kotlinlang.org/docs/multiplatform-discover-project.html)

---

## 12. Archivos de Referencia en edugo-shared (Go)

Los siguientes archivos de Go sirvieron como referencia para mantener consistencia:

- `edugo-shared/common/errors/errors.go` - Patrón de AppError y ErrorCode
- `edugo-shared/common/types/enum/role.go` - Definición de SystemRole

---

**Estado**: Plan completo listo para implementación
**Última actualización**: 2026-01-02
