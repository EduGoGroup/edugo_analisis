# Proyecto Kotlin Multiplatform - edugo-kmp

## Visión General

Un proyecto Kotlin Multiplatform (KMP) modular que compila para Android, Desktop (JVM) y Web (WASM), usando Compose Multiplatform con Material Design 3.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           PROYECTO edugo-kmp                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                          :shared (KMP Library)                       │   │
│   │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │   │
│   │  │commonMain│ │androidMai│ │desktopMai│ │ wasmMain │ │  Tests   │  │   │
│   │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘  │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                    ▲                                         │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                        :features (Módulos UI)                        │   │
│   │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │   │
│   │  │  :auth   │ │:materials│ │ :quizzes │ │:progress │ │  :admin  │  │   │
│   │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘  │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                    ▲                                         │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         Aplicaciones                                 │   │
│   │                                                                      │   │
│   │  STUDENT                   TEACHER                   ADMIN          │   │
│   │  ┌────────┐               ┌────────┐               ┌────────┐       │   │
│   │  │Android │               │Android │               │  Web   │       │   │
│   │  │Desktop │               │Desktop │               │ (WASM) │       │   │
│   │  │  Web   │               │  Web   │               └────────┘       │   │
│   │  └────────┘               └────────┘                                │   │
│   │                                                                      │   │
│   │  GUARDIAN (futuro)                                                  │   │
│   │  ┌────────┐                                                         │   │
│   │  │  Web   │                                                         │   │
│   │  └────────┘                                                         │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Estructura de Directorios

```
edugo-kmp/
├── build.gradle.kts                    # Build raíz
├── settings.gradle.kts                 # Configuración de módulos
├── gradle.properties                   # Propiedades de Gradle
│
├── shared/                             # Lógica de negocio compartida
│   ├── build.gradle.kts
│   └── src/
│       ├── commonMain/kotlin/com/edugo/shared/
│       │   ├── di/                     # Koin modules
│       │   │   └── SharedModule.kt
│       │   ├── network/
│       │   │   ├── ApiClient.kt
│       │   │   ├── HttpClientFactory.kt
│       │   │   ├── api/
│       │   │   │   ├── AuthApi.kt
│       │   │   │   ├── MaterialsApi.kt
│       │   │   │   └── QuizApi.kt
│       │   │   └── interceptors/
│       │   │       └── AuthInterceptor.kt
│       │   ├── models/
│       │   │   ├── dto/
│       │   │   │   ├── AuthDto.kt
│       │   │   │   ├── MaterialDto.kt
│       │   │   │   └── QuizDto.kt
│       │   │   └── domain/
│       │   │       ├── User.kt
│       │   │       ├── Material.kt
│       │   │       └── QuizAttempt.kt
│       │   ├── repositories/
│       │   │   ├── AuthRepository.kt
│       │   │   ├── MaterialsRepository.kt
│       │   │   └── QuizRepository.kt
│       │   ├── usecases/
│       │   │   ├── auth/
│       │   │   │   ├── LoginUseCase.kt
│       │   │   │   └── RefreshTokenUseCase.kt
│       │   │   ├── materials/
│       │   │   │   ├── GetMaterialsUseCase.kt
│       │   │   │   └── UploadMaterialUseCase.kt
│       │   │   └── quiz/
│       │   │       └── SubmitQuizUseCase.kt
│       │   └── storage/
│       │       ├── TokenStorage.kt
│       │       ├── PreferencesStorage.kt
│       │       └── CacheManager.kt
│       │
│       ├── androidMain/kotlin/com/edugo/shared/
│       │   ├── di/AndroidModule.kt
│       │   └── storage/
│       │       ├── AndroidTokenStorage.kt
│       │       └── AndroidPreferences.kt
│       │
│       ├── desktopMain/kotlin/com/edugo/shared/
│       │   ├── di/DesktopModule.kt
│       │   └── storage/
│       │       ├── DesktopTokenStorage.kt
│       │       └── DesktopPreferences.kt
│       │
│       ├── wasmJsMain/kotlin/com/edugo/shared/
│       │   ├── di/WasmModule.kt
│       │   └── storage/
│       │       ├── BrowserTokenStorage.kt
│       │       └── LocalStoragePreferences.kt
│       │
│       └── commonTest/
│           └── kotlin/
│
├── features/                           # Módulos de UI compartida
│   ├── feature-auth/
│   │   ├── build.gradle.kts
│   │   └── src/commonMain/kotlin/com/edugo/feature/auth/
│   │       ├── LoginScreen.kt
│   │       ├── LoginViewModel.kt
│   │       └── components/
│   │           ├── EmailField.kt
│   │           └── PasswordField.kt
│   │
│   ├── feature-materials/
│   │   ├── build.gradle.kts
│   │   └── src/commonMain/kotlin/com/edugo/feature/materials/
│   │       ├── list/
│   │       │   ├── MaterialsListScreen.kt
│   │       │   └── MaterialsListViewModel.kt
│   │       ├── detail/
│   │       │   ├── MaterialDetailScreen.kt
│   │       │   └── MaterialDetailViewModel.kt
│   │       ├── upload/                 # Solo Teacher
│   │       │   ├── UploadScreen.kt
│   │       │   └── UploadViewModel.kt
│   │       └── reader/
│   │           ├── PdfReaderScreen.kt
│   │           └── PdfReaderViewModel.kt
│   │
│   ├── feature-quizzes/
│   │   ├── build.gradle.kts
│   │   └── src/commonMain/kotlin/com/edugo/feature/quizzes/
│   │       ├── quiz/
│   │       │   ├── QuizScreen.kt
│   │       │   └── QuizViewModel.kt
│   │       ├── results/
│   │       │   ├── ResultsScreen.kt
│   │       │   └── ResultsViewModel.kt
│   │       └── history/
│   │           ├── HistoryScreen.kt
│   │           └── HistoryViewModel.kt
│   │
│   ├── feature-progress/
│   │   ├── build.gradle.kts
│   │   └── src/commonMain/kotlin/com/edugo/feature/progress/
│   │       ├── dashboard/
│   │       │   ├── ProgressDashboardScreen.kt
│   │       │   └── ProgressDashboardViewModel.kt
│   │       └── components/
│   │           ├── ProgressCard.kt
│   │           └── SubjectProgressChart.kt
│   │
│   ├── feature-statistics/             # Solo Teacher
│   │   ├── build.gradle.kts
│   │   └── src/commonMain/kotlin/com/edugo/feature/statistics/
│   │       ├── MaterialStatsScreen.kt
│   │       ├── StudentProgressScreen.kt
│   │       └── StatsViewModel.kt
│   │
│   └── feature-admin/                  # Solo Admin Web
│       ├── build.gradle.kts
│       └── src/commonMain/kotlin/com/edugo/feature/admin/
│           ├── schools/
│           │   ├── SchoolsListScreen.kt
│           │   ├── SchoolDetailScreen.kt
│           │   └── SchoolsViewModel.kt
│           ├── hierarchy/
│           │   ├── HierarchyTreeScreen.kt
│           │   └── HierarchyViewModel.kt
│           ├── memberships/
│           │   ├── MembershipsScreen.kt
│           │   └── MembershipsViewModel.kt
│           └── subjects/
│               ├── SubjectsScreen.kt
│               └── SubjectsViewModel.kt
│
├── core/                               # Módulos core
│   ├── core-ui/                        # Componentes UI compartidos
│   │   ├── build.gradle.kts
│   │   └── src/commonMain/kotlin/com/edugo/core/ui/
│   │       ├── theme/
│   │       │   ├── EduGoTheme.kt
│   │       │   ├── Colors.kt
│   │       │   ├── Typography.kt
│   │       │   └── Shapes.kt
│   │       ├── components/
│   │       │   ├── EduGoButton.kt
│   │       │   ├── EduGoTextField.kt
│   │       │   ├── EduGoCard.kt
│   │       │   ├── LoadingIndicator.kt
│   │       │   └── ErrorMessage.kt
│   │       └── navigation/
│   │           └── NavigationHost.kt
│   │
│   └── core-utils/                     # Utilidades compartidas
│       ├── build.gradle.kts
│       └── src/commonMain/kotlin/com/edugo/core/utils/
│           ├── Extensions.kt
│           ├── DateUtils.kt
│           └── ValidationUtils.kt
│
├── apps/                               # Aplicaciones finales
│   ├── app-student-android/
│   │   ├── build.gradle.kts
│   │   └── src/main/
│   │       ├── AndroidManifest.xml
│   │       ├── kotlin/com/edugo/student/
│   │       │   ├── EduGoStudentApp.kt
│   │       │   ├── MainActivity.kt
│   │       │   └── navigation/
│   │       │       └── StudentNavGraph.kt
│   │       └── res/
│   │
│   ├── app-student-desktop/
│   │   ├── build.gradle.kts
│   │   └── src/jvmMain/kotlin/com/edugo/student/
│   │       ├── Main.kt
│   │       └── navigation/
│   │           └── DesktopNavigation.kt
│   │
│   ├── app-student-web/
│   │   ├── build.gradle.kts
│   │   ├── src/wasmJsMain/kotlin/com/edugo/student/
│   │   │   └── Main.kt
│   │   └── src/wasmJsMain/resources/
│   │       └── index.html
│   │
│   ├── app-teacher-android/
│   │   ├── build.gradle.kts
│   │   └── src/main/kotlin/com/edugo/teacher/
│   │       ├── EduGoTeacherApp.kt
│   │       └── MainActivity.kt
│   │
│   ├── app-teacher-desktop/
│   │   ├── build.gradle.kts
│   │   └── src/jvmMain/kotlin/com/edugo/teacher/
│   │       └── Main.kt
│   │
│   ├── app-teacher-web/
│   │   ├── build.gradle.kts
│   │   └── src/wasmJsMain/kotlin/com/edugo/teacher/
│   │       └── Main.kt
│   │
│   └── app-admin-web/                  # Solo Web
│       ├── build.gradle.kts
│       └── src/wasmJsMain/kotlin/com/edugo/admin/
│           ├── Main.kt
│           └── navigation/
│               └── AdminNavGraph.kt
│
└── buildSrc/                           # Build logic compartida
    └── src/main/kotlin/
        ├── Dependencies.kt
        └── ProjectConfig.kt
```

---

## Dependencias Principales

### Versiones (gradle.properties)
```properties
kotlin.version=2.0.21
compose.version=1.7.1
ktor.version=3.0.2
koin.version=4.0.0
coroutines.version=1.9.0
```

### Dependencias por Módulo

#### :shared
```kotlin
// build.gradle.kts
dependencies {
    // Ktor para networking
    commonMainImplementation("io.ktor:ktor-client-core:$ktorVersion")
    commonMainImplementation("io.ktor:ktor-client-content-negotiation:$ktorVersion")
    commonMainImplementation("io.ktor:ktor-serialization-kotlinx-json:$ktorVersion")
    
    // Koin para DI
    commonMainImplementation("io.insert-koin:koin-core:$koinVersion")
    
    // Coroutines
    commonMainImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:$coroutinesVersion")
    
    // DateTime
    commonMainImplementation("org.jetbrains.kotlinx:kotlinx-datetime:0.6.1")
    
    // Platform-specific Ktor engines
    androidMainImplementation("io.ktor:ktor-client-okhttp:$ktorVersion")
    desktopMainImplementation("io.ktor:ktor-client-java:$ktorVersion")
    wasmJsMainImplementation("io.ktor:ktor-client-js:$ktorVersion")
}
```

#### :core-ui
```kotlin
dependencies {
    commonMainImplementation(compose.runtime)
    commonMainImplementation(compose.foundation)
    commonMainImplementation(compose.material3)
    commonMainImplementation(compose.ui)
    commonMainImplementation(compose.components.resources)
}
```

#### Apps Android
```kotlin
dependencies {
    implementation(project(":shared"))
    implementation(project(":core:core-ui"))
    implementation(project(":features:feature-auth"))
    // ... otros features según el rol
    
    // Android specific
    implementation("androidx.activity:activity-compose:1.9.3")
    implementation("androidx.navigation:navigation-compose:2.8.4")
    implementation("io.insert-koin:koin-android:$koinVersion")
}
```

---

## Configuración Material Design 3

### Tema Base (core-ui/theme/EduGoTheme.kt)
```kotlin
@Composable
fun EduGoTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val colorScheme = if (darkTheme) {
        darkColorScheme(
            primary = EduGoPrimary,
            onPrimary = Color.White,
            secondary = EduGoSecondary,
            // ... MD3 color tokens
        )
    } else {
        lightColorScheme(
            primary = EduGoPrimary,
            onPrimary = Color.White,
            secondary = EduGoSecondary,
            // ...
        )
    }
    
    MaterialTheme(
        colorScheme = colorScheme,
        typography = EduGoTypography,
        shapes = EduGoShapes,
        content = content
    )
}
```

---

## Builds y Outputs

### Comandos de Build
```bash
# Android Student APK
./gradlew :apps:app-student-android:assembleRelease

# Desktop Student (todas las plataformas)
./gradlew :apps:app-student-desktop:packageDistributionForCurrentOS

# Web Student (WASM)
./gradlew :apps:app-student-web:wasmJsBrowserDistribution

# Todos los Android
./gradlew assembleRelease

# Todos los Desktop
./gradlew packageDistributionForCurrentOS

# Todos los Web
./gradlew wasmJsBrowserDistribution
```

### Outputs
| App | Plataforma | Output |
|-----|------------|--------|
| Student Android | Android | `app-student-android/build/outputs/apk/release/` |
| Student Desktop | Windows | `app-student-desktop/build/compose/binaries/main/msi/` |
| Student Desktop | macOS | `app-student-desktop/build/compose/binaries/main/dmg/` |
| Student Desktop | Linux | `app-student-desktop/build/compose/binaries/main/deb/` |
| Student Web | WASM | `app-student-web/build/dist/wasmJs/productionExecutable/` |

---

## RFCs por Aplicación

### app-student-* (Android/Desktop/Web)
Incluye features: `:auth`, `:materials`, `:quizzes`, `:progress`
| RFC | Descripción |
|-----|-------------|
| RFC-001-004 | Autenticación completa |
| RFC-003/004 (00) | Cache y errores |
| RFC-020-025 | Materiales y lectura |
| RFC-030-033 | Quizzes |
| RFC-040-041 | Resúmenes IA |
| RFC-052 | Dashboard progreso |

### app-teacher-* (Android/Desktop/Web)
Incluye features: `:auth`, `:materials`, `:statistics`
| RFC | Descripción |
|-----|-------------|
| RFC-001-004 | Autenticación |
| RFC-020-022, 025 | Materiales (ver) |
| RFC-021 | Upload PDF |
| RFC-002 (00), 041 | Polling procesamiento |
| RFC-050-052 | Estadísticas |

### app-admin-web (Solo WASM)
Incluye features: `:auth`, `:admin`
| RFC | Descripción |
|-----|-------------|
| RFC-001-004 | Autenticación |
| RFC-010 | CRUD Escuelas |
| RFC-011 | Jerarquía |
| RFC-012 | Membresías |
| RFC-013 | Acudientes |
| RFC-014 | Materias |
| RFC-051 | Stats globales |

---

## Consideraciones Especiales

### PDF Rendering
- **Android**: `implementation("com.github.AliMehrpour:PdfViewerKotlin:1.0.0")` o similar
- **Desktop**: `implementation("org.apache.pdfbox:pdfbox:3.0.0")`
- **Web**: `js-pdf-viewer` integrado via JS interop

### Offline Support (Android/Desktop)
- SQLDelight para cache local
- WorkManager (Android) / Coroutines (Desktop) para sync

### Web WASM Consideraciones
- No hay localStorage persistente por defecto (usar IndexedDB via JS interop)
- Service Worker para PWA features
- Tamaño del bundle WASM (~10-20MB inicial)
