# Proyecto Apple - edugo-apple

## Visión General

Un único proyecto Xcode con múltiples targets que comparte código entre iOS, iPadOS y macOS.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           PROYECTO edugo-apple                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                        EduGoShared (Framework)                       │   │
│   │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │   │
│   │  │Networking│ │  Models  │ │ Storage  │ │  Utils   │ │   DI     │  │   │
│   │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘  │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                    ▲                                         │
│              ┌─────────────────────┼─────────────────────┐                  │
│              │                     │                     │                  │
│   ┌──────────┴──────────┐ ┌───────┴────────┐ ┌─────────┴─────────┐        │
│   │    EduGoStudent     │ │  EduGoTeacher  │ │  EduGoGuardian    │        │
│   │  (iOS/iPadOS/macOS) │ │(iOS/iPadOS/mac)│ │ (opcional futuro) │        │
│   ├─────────────────────┤ ├────────────────┤ ├───────────────────┤        │
│   │ - Lectura PDFs      │ │ - Upload PDFs  │ │ - Ver progreso    │        │
│   │ - Quizzes           │ │ - Estadísticas │ │   de pupilos      │        │
│   │ - Progreso personal │ │ - Monitoreo    │ │                   │        │
│   └─────────────────────┘ └────────────────┘ └───────────────────┘        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Estructura de Directorios

```
edugo-apple/
├── EduGo.xcworkspace              # Workspace principal
├── EduGo.xcodeproj                # Proyecto Xcode
│
├── EduGoShared/                   # Framework compartido (SPM local)
│   ├── Package.swift
│   ├── Sources/
│   │   ├── Networking/
│   │   │   ├── APIClient.swift
│   │   │   ├── Endpoints/
│   │   │   │   ├── AuthEndpoints.swift
│   │   │   │   ├── MaterialsEndpoints.swift
│   │   │   │   └── QuizEndpoints.swift
│   │   │   ├── Interceptors/
│   │   │   │   ├── AuthInterceptor.swift
│   │   │   │   └── RetryInterceptor.swift
│   │   │   └── NetworkMonitor.swift
│   │   │
│   │   ├── Models/
│   │   │   ├── DTOs/
│   │   │   │   ├── AuthDTOs.swift
│   │   │   │   ├── MaterialDTOs.swift
│   │   │   │   └── QuizDTOs.swift
│   │   │   └── Entities/
│   │   │       ├── User.swift
│   │   │       ├── Material.swift
│   │   │       └── QuizAttempt.swift
│   │   │
│   │   ├── Storage/
│   │   │   ├── KeychainManager.swift
│   │   │   ├── CoreDataStack.swift
│   │   │   ├── CacheManager.swift
│   │   │   └── OfflineSync/
│   │   │       └── SyncManager.swift
│   │   │
│   │   ├── Repositories/
│   │   │   ├── AuthRepository.swift
│   │   │   ├── MaterialsRepository.swift
│   │   │   └── QuizRepository.swift
│   │   │
│   │   └── Utilities/
│   │       ├── Extensions/
│   │       ├── Helpers/
│   │       └── Constants.swift
│   │
│   └── Tests/
│       └── EduGoSharedTests/
│
├── EduGoStudent/                  # App Estudiante
│   ├── App/
│   │   ├── EduGoStudentApp.swift
│   │   └── AppDelegate.swift
│   ├── Features/
│   │   ├── Auth/
│   │   │   ├── Views/
│   │   │   │   └── LoginView.swift
│   │   │   └── ViewModels/
│   │   │       └── LoginViewModel.swift
│   │   ├── Materials/
│   │   │   ├── Views/
│   │   │   │   ├── MaterialsListView.swift
│   │   │   │   ├── MaterialDetailView.swift
│   │   │   │   └── PDFReaderView.swift
│   │   │   └── ViewModels/
│   │   │       └── MaterialsViewModel.swift
│   │   ├── Quizzes/
│   │   │   ├── Views/
│   │   │   │   ├── QuizView.swift
│   │   │   │   └── ResultsView.swift
│   │   │   └── ViewModels/
│   │   │       └── QuizViewModel.swift
│   │   └── Progress/
│   │       ├── Views/
│   │       │   └── ProgressDashboardView.swift
│   │       └── ViewModels/
│   │           └── ProgressViewModel.swift
│   ├── Resources/
│   │   ├── Assets.xcassets
│   │   └── Localizable.strings
│   ├── Supporting/
│   │   ├── Info.plist
│   │   └── Entitlements/
│   └── Tests/
│
├── EduGoTeacher/                  # App Docente
│   ├── App/
│   │   └── EduGoTeacherApp.swift
│   ├── Features/
│   │   ├── Auth/                  # Reusa ViewModels de Shared
│   │   ├── Materials/
│   │   │   ├── Views/
│   │   │   │   ├── MaterialsListView.swift
│   │   │   │   ├── UploadMaterialView.swift
│   │   │   │   └── ProcessingStatusView.swift
│   │   │   └── ViewModels/
│   │   │       └── UploadViewModel.swift
│   │   ├── Statistics/
│   │   │   ├── Views/
│   │   │   │   ├── MaterialStatsView.swift
│   │   │   │   └── StudentProgressView.swift
│   │   │   └── ViewModels/
│   │   │       └── StatsViewModel.swift
│   │   └── Students/
│   │       ├── Views/
│   │       │   └── StudentsListView.swift
│   │       └── ViewModels/
│   │           └── StudentsViewModel.swift
│   ├── Resources/
│   └── Supporting/
│
└── EduGoGuardian/                 # App Padres (futuro)
    ├── App/
    ├── Features/
    │   ├── Auth/
    │   ├── Pupils/
    │   └── Progress/
    └── Resources/
```

---

## Tecnologías y Dependencias

### Stack Tecnológico
| Categoría | Tecnología | Versión Mínima |
|-----------|------------|----------------|
| **UI** | SwiftUI | iOS 16+ |
| **Arquitectura** | MVVM + Clean Architecture | - |
| **Networking** | URLSession + async/await | - |
| **Storage Local** | CoreData + Keychain | - |
| **Dependency Injection** | Swift-Dependencies o manual | - |
| **PDF Viewer** | PDFKit nativo | - |
| **Tests** | XCTest + Quick/Nimble | - |

### Dependencias SPM Sugeridas
```swift
// Package.swift del proyecto principal
dependencies: [
    // Networking
    .package(url: "https://github.com/Alamofire/Alamofire.git", from: "5.8.0"),
    
    // Keychain
    .package(url: "https://github.com/evgenyneu/keychain-swift.git", from: "20.0.0"),
    
    // Image Loading (opcional)
    .package(url: "https://github.com/kean/Nuke.git", from: "12.0.0"),
    
    // Logging
    .package(url: "https://github.com/apple/swift-log.git", from: "1.5.0"),
]
```

---

## Targets y Schemes

### Targets
| Target | Tipo | Plataformas | Bundle ID |
|--------|------|-------------|-----------|
| EduGoShared | Framework | iOS, macOS | - |
| EduGoStudent | App | iOS, iPadOS, macOS | com.edugo.student |
| EduGoTeacher | App | iOS, iPadOS, macOS | com.edugo.teacher |
| EduGoGuardian | App | iOS, iPadOS, macOS | com.edugo.guardian |

### Schemes
```
- EduGoStudent-Debug
- EduGoStudent-Release
- EduGoTeacher-Debug
- EduGoTeacher-Release
- EduGoGuardian-Debug
- EduGoGuardian-Release
```

---

## Configuraciones por Plataforma

### iOS/iPadOS
- Soporta iPhone y iPad
- Orientación: Portrait (iPhone), All (iPad)
- Multitasking iPad: Slide Over, Split View

### macOS (Catalyst o Native)
**Opción A: Mac Catalyst** (recomendado inicialmente)
- Menor esfuerzo de desarrollo
- Comparte código UI con iOS
- Limitaciones en UX nativa

**Opción B: SwiftUI Nativo macOS** (futuro)
- Mejor experiencia macOS
- Menús nativos, window management
- Requiere más trabajo de adaptación

---

## RFCs Implementadas por App

### EduGoStudent
| RFC | Funcionalidad |
|-----|---------------|
| RFC-001 | Login |
| RFC-002 | Refresh Token |
| RFC-003 | Logout |
| RFC-004 | Validación Sesión |
| RFC-003 (00) | Cache Local |
| RFC-004 (00) | Manejo Errores |
| RFC-020 | Listado Materiales |
| RFC-022 | Descarga/Lectura PDF |
| RFC-024 | Progreso Lectura |
| RFC-025 | Detalle Material |
| RFC-030 | Quiz IA |
| RFC-031 | Respuestas Quiz |
| RFC-032 | Resultados Quiz |
| RFC-033 | Historial Intentos |
| RFC-040 | Resumen IA |
| RFC-041 | Estado Procesando |
| RFC-052 | Dashboard Progreso |

### EduGoTeacher
| RFC | Funcionalidad |
|-----|---------------|
| RFC-001-004 | Autenticación |
| RFC-003/004 (00) | Cache y Errores |
| RFC-020 | Listado Materiales |
| RFC-021 | Subida PDF |
| RFC-022 | Descarga/Lectura |
| RFC-025 | Detalle Material |
| RFC-002 (00) | Polling Procesamiento |
| RFC-041 | Estado Procesando |
| RFC-050 | Stats Material |
| RFC-052 | Progreso Estudiantes |

### EduGoGuardian (futuro)
| RFC | Funcionalidad |
|-----|---------------|
| RFC-001-004 | Autenticación |
| RFC-013 | Relaciones Pupilos |
| RFC-052 | Ver Progreso Pupilos |

---

## Consideraciones de Desarrollo

### Offline Support
- CoreData para persistencia local
- Sync queue para operaciones pendientes
- Network reachability monitoring

### Seguridad
- Tokens en Keychain (no UserDefaults)
- Certificate pinning (opcional)
- Biometric auth para re-login

### Performance
- Lazy loading de PDFs
- Image caching con Nuke
- Background fetch para sync
