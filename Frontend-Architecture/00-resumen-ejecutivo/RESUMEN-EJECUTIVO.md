# Resumen Ejecutivo - Arquitectura Frontend EduGo

## Fecha de Análisis
2026-01-02 (Actualizado)

## Estado del Backend

> **ACTUALIZACIÓN 2025-01-02:** Se verificó el estado real de implementación del backend:
> 
> | Módulo | Estado | Notas |
> |--------|--------|-------|
> | Autenticación | ✅ Completo | Login, refresh, logout funcionando |
> | Gestión Escolar | ✅ Completo | RFC-013, RFC-014 ya en main |
> | Materiales | ✅ Completo | Upload, download, progreso |
> | **Quizzes IA** | ✅ **COMPLETO** | Worker genera quiz + API Mobile expone endpoints |
> | Resúmenes IA | ⚠️ Parcial | Worker genera resumen, falta endpoint API Mobile |
> | Estadísticas | ✅ Completo | Dashboard y progreso |

## Contexto

Se analizaron **28 RFCs** distribuidos en **6 módulos** para determinar la estructura óptima de proyectos frontend, considerando:

- **Plataforma Apple**: Un proyecto único para iOS/iPadOS/macOS (SwiftUI)
- **Kotlin Multiplatform (KMP)**: Un proyecto base para Android, Desktop y Web (WASM)
- **Material Design 3**: Para todas las plataformas no-Apple

---

## Decisión Arquitectónica Principal

### RECOMENDACIÓN: **Separación por Roles + Módulos Compartidos**

Tras analizar las RFCs, los actores, frecuencias de uso y requisitos offline, se recomienda:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ESTRUCTURA DE APLICACIONES                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │   ESTUDIANTE │  │   DOCENTE    │  │   GUARDIAN   │               │
│  │   (Student)  │  │  (Teacher)   │  │   (Parent)   │               │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤               │
│  │ iOS/iPadOS   │  │ iOS/iPadOS   │  │ Web Only     │               │
│  │ macOS        │  │ macOS        │  │ (opcional    │               │
│  │ Android      │  │ Android      │  │  móvil)      │               │
│  │ Desktop      │  │ Desktop      │  │              │               │
│  │ Web          │  │ Web          │  │              │               │
│  └──────────────┘  └──────────────┘  └──────────────┘               │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                    PANEL ADMINISTRACIÓN                         │ │
│  │                      (Admin Dashboard)                          │ │
│  ├────────────────────────────────────────────────────────────────┤ │
│  │  Solo Web - No requiere offline - Baja frecuencia de uso       │ │
│  │  (Puede ser parte del proyecto KMP-Web o proyecto separado)    │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Justificación de Separación por Roles

### Por qué NO una app única para todo:

| Factor | App Única | Apps Separadas | Ganador |
|--------|-----------|----------------|---------|
| **Tamaño de app** | ~50MB+ todo incluido | ~15-25MB por rol | Separadas |
| **Complejidad UX** | Menús complejos, permisos dinámicos | UX optimizada por rol | Separadas |
| **Seguridad** | Código admin expuesto a estudiantes | Mínima superficie de ataque | Separadas |
| **Actualizaciones** | Todos afectados por cambios admin | Actualizaciones independientes | Separadas |
| **Stores** | Una sola app genérica | Mejor posicionamiento SEO/ASO | Separadas |
| **Onboarding** | Confuso, múltiples flujos | Claro, enfocado | Separadas |
| **Mantenimiento** | Menor duplicación aparente | Módulos compartidos mitigan | Empate |

### Caso especial: Usuarios con múltiples roles

Un docente que también es padre/guardian puede:
1. Instalar ambas apps (recomendado)
2. Cambiar de contexto dentro de la misma sesión (si se implementa)

La arquitectura modular permite que si en el futuro se decide unificar, sea posible.

---

## Estructura de Proyectos Recomendada

### Proyecto 1: edugo-apple (Swift/SwiftUI)
```
edugo-apple/
├── EduGoShared/           # Framework compartido
│   ├── Networking/        # API clients, interceptors
│   ├── Models/            # DTOs, entities
│   ├── Storage/           # CoreData, Keychain
│   └── Utilities/         # Extensions, helpers
├── EduGoStudent/          # Target: Estudiantes
├── EduGoTeacher/          # Target: Docentes
└── EduGoGuardian/         # Target: Padres (opcional MVP+1)
```

### Proyecto 2: edugo-kmp (Kotlin Multiplatform)
```
edugo-kmp/
├── shared/                # Lógica multiplataforma
│   ├── commonMain/        # Código compartido
│   ├── androidMain/       # Específico Android
│   ├── desktopMain/       # Específico Desktop
│   └── wasmMain/          # Específico Web WASM
├── features/              # Módulos de funcionalidad
│   ├── feature-auth/
│   ├── feature-materials/
│   ├── feature-quizzes/
│   ├── feature-progress/
│   └── feature-admin/     # Solo para admin-web
├── app-student-android/
├── app-student-desktop/
├── app-student-web/
├── app-teacher-android/
├── app-teacher-desktop/
├── app-teacher-web/
└── app-admin-web/         # Solo Web
```

---

## Resumen de Apps por Plataforma

| Aplicación | Apple | Android | Desktop | Web |
|------------|-------|---------|---------|-----|
| **EduGo Student** | iOS/iPadOS/macOS | KMP | KMP | KMP WASM |
| **EduGo Teacher** | iOS/iPadOS/macOS | KMP | KMP | KMP WASM |
| **EduGo Guardian** | No (Web first) | No (Web first) | No | KMP WASM |
| **EduGo Admin** | No | No | No | Solo Web |

---

## Prioridad de Desarrollo Sugerida

### Fase 1 - MVP (Semanas 1-8)
1. edugo-kmp/app-student-android + app-student-web
2. edugo-apple/EduGoStudent (iOS)
3. RFCs: 001-004, 020, 022, 024, 025, 030-032

### Fase 2 - Teacher App (Semanas 9-12)
1. edugo-kmp/app-teacher-android + app-teacher-web
2. edugo-apple/EduGoTeacher (iOS)
3. RFCs: 021, 050-052

### Fase 3 - Desktop + Admin (Semanas 13-16)
1. edugo-kmp/app-student-desktop + app-teacher-desktop
2. edugo-kmp/app-admin-web
3. RFCs: 010-014

### Fase 4 - Guardian + Pulido (Semanas 17-20)
1. edugo-kmp/app-guardian-web (o módulo en teacher)
2. edugo-apple/EduGoGuardian (opcional)
3. RFCs: 013, mejoras UX

---

## Documentos Relacionados

1. [01-analisis-actores/](../01-analisis-actores/) - Análisis detallado de actores
2. [02-estructura-proyectos/](../02-estructura-proyectos/) - Arquitectura técnica
3. [03-rfcs-por-aplicacion/](../03-rfcs-por-aplicacion/) - Mapeo RFC a App
4. [04-modulos-compartidos/](../04-modulos-compartidos/) - Módulos reutilizables

---

## Consideraciones Especiales

### Web vs Nativo
- **Estudiantes/Docentes**: Necesitan offline, cache local (IndexedDB/CoreData)
- **Admin/Guardian**: No requieren offline crítico, Web-first es suficiente

### Material Design 3
- Todos los proyectos KMP usarán MD3
- Apple usará SwiftUI nativo (HIG compliance)

### Modularidad
- Ambos proyectos (Apple y KMP) serán modulares
- Máximo reuso de lógica de negocio
- UI específica por plataforma
