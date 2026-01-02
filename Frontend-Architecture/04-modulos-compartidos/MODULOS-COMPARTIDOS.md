# Módulos Compartidos - Arquitectura Modular

## Filosofía de Diseño

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        PRINCIPIO DE MODULARIDAD                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   "Escribir una vez, usar en todas partes, mantener en un solo lugar"       │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         CAPA DE ABSTRACCIÓN                          │   │
│   ├─────────────────────────────────────────────────────────────────────┤   │
│   │                                                                      │   │
│   │   Shared Library          Feature Modules           Core Modules    │   │
│   │   (Lógica de negocio)     (UI por funcionalidad)    (UI base)       │   │
│   │                                                                      │   │
│   │   ┌───────────────┐       ┌───────────────┐       ┌───────────────┐ │   │
│   │   │ Repositories  │       │ feature-auth  │       │   core-ui     │ │   │
│   │   │ Use Cases     │       │feature-mater. │       │  core-utils   │ │   │
│   │   │ Models        │       │feature-quizzes│       │core-navigation│ │   │
│   │   │ Network       │       │feature-progress│      │               │ │   │
│   │   │ Storage       │       │feature-admin  │       │               │ │   │
│   │   └───────────────┘       └───────────────┘       └───────────────┘ │   │
│   │          │                       │                       │          │   │
│   │          └───────────────────────┼───────────────────────┘          │   │
│   │                                  ▼                                   │   │
│   │                        ┌───────────────┐                            │   │
│   │                        │  Aplicaciones │                            │   │
│   │                        │  (Composición)│                            │   │
│   │                        └───────────────┘                            │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Proyecto Apple (edugo-apple)

### EduGoShared (Framework)

Framework local compartido entre todos los targets Apple.

```
EduGoShared/
├── Sources/
│   ├── Networking/
│   │   ├── APIClient.swift          # Cliente HTTP base
│   │   ├── Endpoints.swift          # Definición de endpoints
│   │   ├── AuthInterceptor.swift    # Inyección de tokens
│   │   └── ErrorHandler.swift       # Mapeo de errores HTTP
│   │
│   ├── Models/
│   │   ├── DTOs/                    # Data Transfer Objects
│   │   │   ├── AuthDTOs.swift
│   │   │   ├── MaterialDTOs.swift
│   │   │   └── QuizDTOs.swift
│   │   └── Entities/                # Modelos de dominio
│   │       ├── User.swift
│   │       ├── Material.swift
│   │       └── QuizAttempt.swift
│   │
│   ├── Repositories/
│   │   ├── AuthRepository.swift     # Abstracción de auth
│   │   ├── MaterialsRepository.swift
│   │   └── QuizRepository.swift
│   │
│   ├── Storage/
│   │   ├── KeychainManager.swift    # Tokens seguros
│   │   ├── CoreDataStack.swift      # Persistencia local
│   │   └── CacheManager.swift       # Cache en memoria/disco
│   │
│   └── Utilities/
│       ├── Extensions/
│       │   ├── Date+Formatting.swift
│       │   └── String+Validation.swift
│       └── Constants.swift          # URLs, timeouts, etc.
│
└── Tests/
    └── EduGoSharedTests/
```

**Uso por Target**:

| Componente | EduGoStudent | EduGoTeacher | EduGoGuardian |
|------------|--------------|--------------|---------------|
| APIClient | ✅ | ✅ | ✅ |
| AuthDTOs | ✅ | ✅ | ✅ |
| MaterialDTOs | ✅ | ✅ | ❌ |
| QuizDTOs | ✅ | ❌ | ❌ |
| KeychainManager | ✅ | ✅ | ✅ |
| CacheManager | ✅ | ✅ Parcial | ❌ |

---

## Proyecto KMP (edugo-kmp)

### :shared (KMP Library)

Biblioteca multiplataforma con lógica de negocio.

```
shared/
├── src/
│   ├── commonMain/kotlin/com/edugo/shared/
│   │   ├── di/
│   │   │   └── SharedModule.kt        # Koin modules
│   │   ├── network/
│   │   │   ├── api/
│   │   │   │   ├── AuthApi.kt
│   │   │   │   ├── MaterialsApi.kt
│   │   │   │   └── QuizApi.kt
│   │   │   ├── HttpClientFactory.kt
│   │   │   └── interceptors/
│   │   │       └── AuthInterceptor.kt
│   │   ├── models/
│   │   │   ├── dto/
│   │   │   └── domain/
│   │   ├── repositories/
│   │   │   ├── AuthRepository.kt
│   │   │   ├── MaterialsRepository.kt
│   │   │   └── QuizRepository.kt
│   │   ├── usecases/
│   │   │   ├── auth/
│   │   │   │   ├── LoginUseCase.kt
│   │   │   │   └── RefreshTokenUseCase.kt
│   │   │   └── materials/
│   │   │       └── GetMaterialsUseCase.kt
│   │   └── storage/
│   │       ├── TokenStorage.kt        # expect
│   │       └── PreferencesStorage.kt  # expect
│   │
│   ├── androidMain/                   # actual implementations
│   ├── desktopMain/
│   └── wasmJsMain/
```

### :core-ui

Componentes UI compartidos con Material Design 3.

```
core/core-ui/
├── src/commonMain/kotlin/com/edugo/core/ui/
│   ├── theme/
│   │   ├── EduGoTheme.kt
│   │   ├── Colors.kt
│   │   ├── Typography.kt
│   │   └── Shapes.kt
│   │
│   ├── components/
│   │   ├── buttons/
│   │   │   ├── EduGoButton.kt
│   │   │   ├── EduGoIconButton.kt
│   │   │   └── EduGoFAB.kt
│   │   ├── inputs/
│   │   │   ├── EduGoTextField.kt
│   │   │   ├── EduGoPasswordField.kt
│   │   │   └── EduGoSearchBar.kt
│   │   ├── cards/
│   │   │   ├── EduGoCard.kt
│   │   │   └── EduGoListItem.kt
│   │   ├── feedback/
│   │   │   ├── LoadingIndicator.kt
│   │   │   ├── ErrorMessage.kt
│   │   │   └── EmptyState.kt
│   │   └── layout/
│   │       ├── EduGoScaffold.kt
│   │       └── EduGoTopBar.kt
│   │
│   └── resources/
│       └── strings.xml               # i18n
```

### :core-utils

Utilidades compartidas.

```
core/core-utils/
├── src/commonMain/kotlin/com/edugo/core/utils/
│   ├── Extensions.kt
│   ├── DateUtils.kt
│   ├── ValidationUtils.kt
│   └── FormatUtils.kt
```

---

## Feature Modules

Módulos de funcionalidad que pueden ser incluidos selectivamente por cada app.

### :feature-auth

```
features/feature-auth/
├── src/commonMain/kotlin/com/edugo/feature/auth/
│   ├── LoginScreen.kt
│   ├── LoginViewModel.kt
│   ├── components/
│   │   ├── EmailField.kt
│   │   ├── PasswordField.kt
│   │   └── LoginButton.kt
│   └── di/
│       └── AuthFeatureModule.kt
```

**Usado por**: Student ✅ | Teacher ✅ | Guardian ✅ | Admin ✅

### :feature-materials

```
features/feature-materials/
├── src/commonMain/kotlin/com/edugo/feature/materials/
│   ├── list/
│   │   ├── MaterialsListScreen.kt
│   │   └── MaterialsListViewModel.kt
│   ├── detail/
│   │   ├── MaterialDetailScreen.kt
│   │   └── MaterialDetailViewModel.kt
│   ├── upload/                        # Solo Teacher
│   │   ├── UploadScreen.kt
│   │   └── UploadViewModel.kt
│   └── reader/
│       ├── PdfReaderScreen.kt
│       └── PdfReaderViewModel.kt
```

**Usado por**: Student ✅ | Teacher ✅ | Guardian ❌ | Admin ❌

### :feature-quizzes

```
features/feature-quizzes/
├── src/commonMain/kotlin/com/edugo/feature/quizzes/
│   ├── quiz/
│   │   ├── QuizScreen.kt
│   │   └── QuizViewModel.kt
│   ├── results/
│   │   ├── ResultsScreen.kt
│   │   └── ResultsViewModel.kt
│   └── history/
│       ├── HistoryScreen.kt
│       └── HistoryViewModel.kt
```

**Usado por**: Student ✅ | Teacher ❌ | Guardian ❌ | Admin ❌

### :feature-progress

```
features/feature-progress/
├── src/commonMain/kotlin/com/edugo/feature/progress/
│   ├── dashboard/
│   │   ├── ProgressDashboardScreen.kt
│   │   └── ProgressDashboardViewModel.kt
│   └── components/
│       ├── ProgressCard.kt
│       ├── SubjectProgressChart.kt
│       └── ActivityFeed.kt
```

**Usado por**: Student ✅ | Teacher ✅ (parcial) | Guardian ✅ | Admin ❌

### :feature-statistics

```
features/feature-statistics/
├── src/commonMain/kotlin/com/edugo/feature/statistics/
│   ├── MaterialStatsScreen.kt
│   ├── StudentProgressScreen.kt
│   ├── GlobalStatsScreen.kt
│   └── StatsViewModel.kt
```

**Usado por**: Student ❌ | Teacher ✅ | Guardian ❌ | Admin ✅ (Global)

### :feature-admin

```
features/feature-admin/
├── src/commonMain/kotlin/com/edugo/feature/admin/
│   ├── schools/
│   │   ├── SchoolsListScreen.kt
│   │   ├── SchoolDetailScreen.kt
│   │   └── SchoolsViewModel.kt
│   ├── hierarchy/
│   │   ├── HierarchyTreeScreen.kt
│   │   └── HierarchyViewModel.kt
│   ├── memberships/
│   │   ├── MembershipsScreen.kt
│   │   └── MembershipsViewModel.kt
│   └── subjects/
│       ├── SubjectsScreen.kt
│       └── SubjectsViewModel.kt
```

**Usado por**: Student ❌ | Teacher ❌ | Guardian ❌ | Admin ✅

---

## Matriz de Dependencias por App

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     DEPENDENCIAS DE MÓDULOS POR APP                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   MÓDULO              │ Student │ Teacher │ Guardian │ Admin │              │
│   ────────────────────┼─────────┼─────────┼──────────┼───────┤              │
│   :shared             │   ✅    │   ✅    │    ✅    │  ✅   │              │
│   :core-ui            │   ✅    │   ✅    │    ✅    │  ✅   │              │
│   :core-utils         │   ✅    │   ✅    │    ✅    │  ✅   │              │
│   :feature-auth       │   ✅    │   ✅    │    ✅    │  ✅   │              │
│   :feature-materials  │   ✅    │   ✅    │    ❌    │  ❌   │              │
│   :feature-quizzes    │   ✅    │   ❌    │    ❌    │  ❌   │              │
│   :feature-progress   │   ✅    │   ✅*   │    ✅    │  ❌   │              │
│   :feature-statistics │   ❌    │   ✅    │    ❌    │  ✅*  │              │
│   :feature-admin      │   ❌    │   ❌    │    ❌    │  ✅   │              │
│                                                                              │
│   * = uso parcial del módulo                                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Configuración Gradle (KMP)

### settings.gradle.kts
```kotlin
rootProject.name = "edugo-kmp"

include(":shared")
include(":core:core-ui")
include(":core:core-utils")

include(":features:feature-auth")
include(":features:feature-materials")
include(":features:feature-quizzes")
include(":features:feature-progress")
include(":features:feature-statistics")
include(":features:feature-admin")

include(":apps:app-student-android")
include(":apps:app-student-desktop")
include(":apps:app-student-web")
include(":apps:app-teacher-android")
include(":apps:app-teacher-desktop")
include(":apps:app-teacher-web")
include(":apps:app-admin-web")
include(":apps:app-guardian-web")
```

### App Student Android (build.gradle.kts)
```kotlin
plugins {
    id("com.android.application")
    kotlin("android")
    id("org.jetbrains.compose")
}

dependencies {
    implementation(project(":shared"))
    implementation(project(":core:core-ui"))
    implementation(project(":core:core-utils"))
    
    implementation(project(":features:feature-auth"))
    implementation(project(":features:feature-materials"))
    implementation(project(":features:feature-quizzes"))
    implementation(project(":features:feature-progress"))
    
    // NO incluye :feature-statistics ni :feature-admin
}
```

### App Admin Web (build.gradle.kts)
```kotlin
plugins {
    kotlin("multiplatform")
    id("org.jetbrains.compose")
}

kotlin {
    wasmJs {
        browser()
    }
    
    sourceSets {
        val wasmJsMain by getting {
            dependencies {
                implementation(project(":shared"))
                implementation(project(":core:core-ui"))
                implementation(project(":core:core-utils"))
                
                implementation(project(":features:feature-auth"))
                implementation(project(":features:feature-admin"))
                implementation(project(":features:feature-statistics"))
                
                // NO incluye materials, quizzes, progress
            }
        }
    }
}
```

---

## Beneficios de la Modularidad

1. **Menor tamaño de APK/binario**: Cada app solo incluye lo que necesita
2. **Build times más rápidos**: Compilación incremental por módulo
3. **Testing aislado**: Tests unitarios por módulo
4. **Equipos paralelos**: Diferentes devs pueden trabajar en diferentes features
5. **Reuso máximo**: Un cambio en :core-ui se refleja en todas las apps
6. **Feature flags**: Fácil habilitar/deshabilitar features por módulo
