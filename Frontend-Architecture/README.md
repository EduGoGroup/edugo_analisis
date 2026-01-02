# EduGo Frontend Architecture

## Análisis Completo de Arquitectura Frontend

**Fecha de Análisis**: 2026-01-01  
**RFCs Analizadas**: 28  
**Módulos Backend Cubiertos**: 6

---

## Decisión Principal

### **APPS SEPARADAS POR ROL + ARQUITECTURA MODULAR**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          ECOSISTEMA EDUGO FRONTEND                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   PROYECTO APPLE (edugo-apple)            PROYECTO KMP (edugo-kmp)          │
│   Swift/SwiftUI                           Kotlin/Compose + Material 3       │
│   ┌───────────────────────────────┐      ┌───────────────────────────────┐  │
│   │                               │      │                               │  │
│   │   EduGoStudent               │      │   app-student-android         │  │
│   │   - iOS/iPadOS/macOS         │      │   app-student-desktop         │  │
│   │                               │      │   app-student-web (WASM)      │  │
│   │   EduGoTeacher               │      │                               │  │
│   │   - iOS/iPadOS/macOS         │      │   app-teacher-android         │  │
│   │                               │      │   app-teacher-desktop         │  │
│   │   EduGoGuardian (futuro)     │      │   app-teacher-web (WASM)      │  │
│   │   - iOS/iPadOS/macOS         │      │                               │  │
│   │                               │      │   app-guardian-web (WASM)     │  │
│   │   Framework Compartido:      │      │                               │  │
│   │   - EduGoShared              │      │   app-admin-web (WASM only)   │  │
│   │                               │      │                               │  │
│   └───────────────────────────────┘      │   Módulos Compartidos:        │  │
│                                          │   - :shared                   │  │
│                                          │   - :core-ui                  │  │
│                                          │   - :features/*               │  │
│                                          │                               │  │
│                                          └───────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Resumen de Aplicaciones

| Aplicación | Actor | Apple | Android | Desktop | Web |
|------------|-------|-------|---------|---------|-----|
| **EduGo Student** | Estudiante | ✅ SwiftUI | ✅ Compose | ✅ Compose | ✅ WASM |
| **EduGo Teacher** | Docente | ✅ SwiftUI | ✅ Compose | ✅ Compose | ✅ WASM |
| **EduGo Guardian** | Padre/Tutor | ❌ | ❌ | ❌ | ✅ WASM |
| **EduGo Admin** | Administrador | ❌ | ❌ | ❌ | ✅ WASM |

---

## Documentación Detallada

### [00-resumen-ejecutivo/](./00-resumen-ejecutivo/)
- Decisión arquitectónica principal
- Justificación de separación por roles
- Roadmap de desarrollo

### [01-analisis-actores/](./01-analisis-actores/)
- Identificación de actores (Student, Teacher, Guardian, Admin)
- Funcionalidades por rol
- Matriz de permisos

### [02-estructura-proyectos/](./02-estructura-proyectos/)
- **PROYECTO-APPLE.md**: Estructura de edugo-apple (SwiftUI)
- **PROYECTO-KMP.md**: Estructura de edugo-kmp (Kotlin Multiplatform)

### [03-rfcs-por-aplicacion/](./03-rfcs-por-aplicacion/)
- **APP-STUDENT.md**: RFCs 001-004, 020-025, 030-033, 040-041, 052
- **APP-TEACHER.md**: RFCs 001-004, 020-022, 025, 041, 050-052
- **APP-ADMIN.md**: RFCs 001-004, 010-014, 051
- **APP-GUARDIAN.md**: RFCs 001-004, 013, 052

### [04-modulos-compartidos/](./04-modulos-compartidos/)
- Arquitectura modular
- Feature modules
- Matriz de dependencias

---

## Fases de Desarrollo Sugeridas

### Fase 1: MVP Student (Semanas 1-8)
**Objetivo**: Estudiantes pueden leer materiales y hacer quizzes

- `edugo-kmp/app-student-android`
- `edugo-kmp/app-student-web`
- `edugo-apple/EduGoStudent`
- RFCs: 001-004, 020, 022, 024, 025, 030-032

### Fase 2: Teacher App (Semanas 9-12)
**Objetivo**: Docentes pueden subir materiales y ver estadísticas

- `edugo-kmp/app-teacher-android`
- `edugo-kmp/app-teacher-web`
- `edugo-apple/EduGoTeacher`
- RFCs: 021, 050-052

### Fase 3: Desktop + Admin (Semanas 13-16)
**Objetivo**: Apps desktop y panel administrativo

- `edugo-kmp/app-student-desktop`
- `edugo-kmp/app-teacher-desktop`
- `edugo-kmp/app-admin-web`
- RFCs: 010-014, 051

### Fase 4: Guardian + Pulido (Semanas 17-20)
**Objetivo**: App para padres y mejoras generales

- `edugo-kmp/app-guardian-web`
- Notificaciones push
- Mejoras de UX
- RFCs: 013, mejoras

---

## Stack Tecnológico

### Apple (edugo-apple)
| Categoría | Tecnología |
|-----------|------------|
| UI | SwiftUI (iOS 16+) |
| Arquitectura | MVVM + Clean Architecture |
| Networking | URLSession + async/await |
| Storage | CoreData + Keychain |
| PDF | PDFKit nativo |

### KMP (edugo-kmp)
| Categoría | Tecnología |
|-----------|------------|
| UI | Compose Multiplatform |
| Design System | Material Design 3 |
| Networking | Ktor Client |
| DI | Koin |
| Storage | SQLDelight + DataStore |
| Web | Kotlin/WASM |

---

## Consideraciones Clave

### Offline Support
- **Student/Teacher**: CRÍTICO para móvil (cache de materiales, sync diferido)
- **Guardian/Admin**: NO requerido (Web-only)

### Material Design 3
- Todos los proyectos KMP usan MD3
- Apple usa HIG nativo (SwiftUI)

### Modularidad
- Máximo reuso de código entre apps
- Features desacoplados por módulo
- Build rápido gracias a compilación incremental

### Seguridad
- Tokens en almacenamiento seguro (Keychain/EncryptedPrefs)
- Refresh proactivo de tokens
- Limpieza completa en logout

---

## Dependencias del Backend

### APIs
- **Admin API** (8081): Autenticación, gestión escolar
- **Mobile API** (8080): Materiales, quizzes, progreso

### Servicios Externos
- **S3**: Almacenamiento de PDFs (presigned URLs)
- **Worker**: Procesamiento IA (requiere RabbitMQ)

### RFCs Pendientes en Backend
- ✅ RFC-013 (Acudientes): Ya disponible en main
- ✅ RFC-014 (Materias): Ya disponible en main
- ⚠️ RFC-030/031 (Quiz IA): Depende de Worker

---

## Próximos Pasos

1. **Validar** esta arquitectura con el equipo
2. **Crear repositorios** `edugo-apple` y `edugo-kmp`
3. **Setup inicial** de proyectos con estructura base
4. **Implementar** módulo de autenticación primero (RFC-001-004)
5. **Iterar** según fases definidas
