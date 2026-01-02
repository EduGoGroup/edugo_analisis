# EduGo Student - Mapeo de RFCs

## Resumen de la Aplicación

**Propósito**: Aplicación principal para estudiantes. Permite consumir materiales educativos, realizar evaluaciones y monitorear el progreso académico personal.

**Plataformas**:
- iOS/iPadOS/macOS (edugo-apple/EduGoStudent)
- Android (edugo-kmp/app-student-android)
- Desktop Windows/Linux/macOS (edugo-kmp/app-student-desktop)
- Web WASM (edugo-kmp/app-student-web)

**API Principal**: Mobile API (puerto 8080)

---

## RFCs Incluidas

### Módulo: Autenticación (Prioridad: CRÍTICA)

| RFC | Nombre | Pantallas | Componentes |
|-----|--------|-----------|-------------|
| RFC-001 | Login Email/Password | `LoginScreen` | EmailField, PasswordField, LoginButton |
| RFC-002 | Renovación Tokens | Background Service | TokenRefreshService |
| RFC-003 | Cierre de Sesión | `SettingsScreen`, Navigation | LogoutButton, ConfirmDialog |
| RFC-004 | Validación Sesión | App Startup, Router | AuthGuard, SessionValidator |

**Flujo de Usuario**:
```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  Splash  │───>│  Login   │───>│  Home    │───>│ Logout   │
│  Screen  │    │  Screen  │    │  Screen  │    │ (Settings)│
└──────────┘    └──────────┘    └──────────┘    └──────────┘
     │               │                              │
     ▼               ▼                              ▼
  Verificar      POST /auth/   Token refresh    POST /auth/
  tokens         login         automático       logout
```

---

### Módulo: Materiales (Prioridad: ALTA)

| RFC | Nombre | Pantallas | Componentes |
|-----|--------|-----------|-------------|
| RFC-020 | Listado Materiales | `MaterialsListScreen` | MaterialCard, FilterChips, SearchBar |
| RFC-022 | Descarga/Visualización | `PdfReaderScreen` | PdfViewer, ProgressBar, DownloadButton |
| RFC-024 | Progreso de Lectura | `PdfReaderScreen` | Auto-save, ProgressIndicator |
| RFC-025 | Detalle Material | `MaterialDetailScreen` | MaterialHeader, ActionButtons, SummaryCard |

**Flujo de Usuario**:
```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Materials   │───>│   Material   │───>│  PDF Reader  │
│    List      │    │   Detail     │    │              │
└──────────────┘    └──────────────┘    └──────────────┘
       │                   │                    │
       ▼                   ▼                    ▼
  GET /materials    GET /materials/:id   Presigned URL S3
  (filtros)         (+ resumen + quiz)   + auto-save progress
```

**Wireframes Conceptuales**:

```
┌─────────────────────────────────────┐
│  Materials List                    ≡ │
├─────────────────────────────────────┤
│ [Todas ▼] [Matemáticas] [Ciencias]  │
│                                     │
│ ┌─────────────────────────────────┐ │
│ │ 📄 Álgebra Básica               │ │
│ │ Matemáticas • 15 págs • 60%     │ │
│ │ ━━━━━━━━━━━━░░░░░░░░░░░░░       │ │
│ └─────────────────────────────────┘ │
│                                     │
│ ┌─────────────────────────────────┐ │
│ │ 📄 Fotosíntesis                 │ │
│ │ Ciencias • 12 págs • ✅ Quiz    │ │
│ │ ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │ │
│ └─────────────────────────────────┘ │
│                                     │
│ ┌─────────────────────────────────┐ │
│ │ 📄 Historia del Siglo XX       │ │
│ │ Historia • 25 págs • Nuevo      │ │
│ │ ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │ │
│ └─────────────────────────────────┘ │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ ←  Álgebra Básica              ⋮   │
├─────────────────────────────────────┤
│                                     │
│  ┌─────────────────────────────┐   │
│  │        [PDF Preview]        │   │
│  │                             │   │
│  └─────────────────────────────┘   │
│                                     │
│  Matemáticas • Prof. García        │
│  15 páginas • Subido: 15 Dic 2025  │
│                                     │
│  Tu progreso: 60% (pág 9/15)       │
│  ━━━━━━━━━━━━░░░░░░░░░░░░░░        │
│                                     │
│  📝 Resumen IA                     │
│  ┌─────────────────────────────┐   │
│  │ Puntos clave:               │   │
│  │ • Variables y constantes    │   │
│  │ • Ecuaciones lineales       │   │
│  │ • Sistemas de ecuaciones    │   │
│  └─────────────────────────────┘   │
│                                     │
│  ┌─────────────┐ ┌─────────────┐   │
│  │  📖 LEER   │ │ 📝 QUIZ    │   │
│  └─────────────┘ └─────────────┘   │
│                                     │
└─────────────────────────────────────┘
```

---

### Módulo: Quizzes (Prioridad: ALTA) ✅ BACKEND COMPLETO

| RFC | Nombre | Pantallas | Componentes | Estado API |
|-----|--------|-----------|-------------|------------|
| RFC-030 | Obtener Quiz IA | `QuizScreen` | QuestionCard, AnswerOptions | ✅ IMPLEMENTADO |
| RFC-031 | Enviar Respuestas | `QuizScreen` | SubmitButton, ProgressIndicator | ✅ IMPLEMENTADO |
| RFC-032 | Ver Resultados | `ResultsScreen` | ScoreCard, AnswerReview | ✅ IMPLEMENTADO |
| RFC-033 | Historial Intentos | `HistoryScreen` | AttemptsList, AttemptCard | ✅ IMPLEMENTADO |

> **ACTUALIZACIÓN 2025-01-02:** Todo el backend de Quizzes está implementado:
> - Worker genera quiz automáticamente al procesar PDF (10 preguntas)
> - `GET /v1/materials/:id/assessment` - Obtener quiz (sin respuestas correctas)
> - `POST /v1/materials/:id/assessment/attempts` - Enviar respuestas + scoring servidor
> - `GET /v1/attempts/:id/results` - Ver resultados de intento
> - `GET /v1/users/me/attempts` - Historial de intentos

**Flujo de Usuario**:
```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  Quiz    │───>│  Answer  │───>│ Results  │───>│ History  │
│  Start   │    │ Questions│    │  Screen  │    │  Screen  │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
     │               │               │               │
     ▼               ▼               ▼               ▼
  GET /assess   POST /attempts  GET /results   GET /history
  (sanitizado)  (score server)  (con feedback) (paginado)
```

---

### Módulo: Resúmenes IA (Prioridad: MEDIA) ⚠️ PARCIAL

| RFC | Nombre | Pantallas | Componentes | Estado API |
|-----|--------|-----------|-------------|------------|
| RFC-040 | Ver Resumen | `MaterialDetailScreen` | SummaryCard, ConceptsList | ⚠️ PENDIENTE endpoint |
| RFC-041 | Estado Procesando | `MaterialDetailScreen`, `ProcessingOverlay` | LoadingSpinner, ProgressMessage | ✅ Lógica lista |

> **NOTA:** El Worker YA genera resúmenes y los guarda en MongoDB (`material_summaries`).
> Falta crear endpoint en API Mobile `GET /v1/materials/:id/summary` para exponerlos.

---

### Módulo: Progreso Personal (Prioridad: ALTA)

| RFC | Nombre | Pantallas | Componentes |
|-----|--------|-----------|-------------|
| RFC-052 | Dashboard Progreso | `ProgressDashboardScreen` | ProgressChart, SubjectCards, StatsGrid |

**Wireframe Dashboard**:
```
┌─────────────────────────────────────┐
│  Mi Progreso                       ≡ │
├─────────────────────────────────────┤
│                                     │
│  Bienvenido, Juan                   │
│                                     │
│  ┌─────────────────────────────────┐│
│  │  Esta semana                    ││
│  │  ┌────┐ ┌────┐ ┌────┐ ┌────┐   ││
│  │  │ 5  │ │ 3  │ │85% │ │ 2h │   ││
│  │  │Mat.│ │Quiz│ │Prom│ │Lect│   ││
│  │  └────┘ └────┘ └────┘ └────┘   ││
│  └─────────────────────────────────┘│
│                                     │
│  Por Materia                        │
│  ┌─────────────────────────────────┐│
│  │ Matemáticas         ━━━━░░ 80% ││
│  │ Ciencias            ━━━━━░ 90% ││
│  │ Historia            ━━░░░░ 45% ││
│  │ Lenguaje            ━━━░░░ 70% ││
│  └─────────────────────────────────┘│
│                                     │
│  Materiales Pendientes              │
│  ┌─────────────────────────────────┐│
│  │ 📄 Historia del Siglo XX    →  ││
│  │ 📄 Gramática Avanzada       →  ││
│  └─────────────────────────────────┘│
│                                     │
└─────────────────────────────────────┘
```

---

### Módulo: Infraestructura (Prioridad: CRÍTICA)

| RFC | Nombre | Implementación |
|-----|--------|----------------|
| RFC-000 | Flujo General Datos | `ApiClient`, `NetworkModule` |
| RFC-002 (00) | Polling Estados | `PollingService`, `MaterialStatusObserver` |
| RFC-003 (00) | Cache Local | `CacheManager`, `OfflineRepository` |
| RFC-004 (00) | Manejo Errores | `ErrorHandler`, `ErrorMapper`, `ErrorUI` |

---

## Navegación de la App

```
┌─────────────────────────────────────────────────────────────┐
│                    EduGo Student Navigation                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  [Splash] ──▶ [Login] ──▶ ┌──────────────────────────────┐  │
│                           │         Main Tabs            │  │
│                           │  ┌────┐ ┌────┐ ┌────┐ ┌────┐ │  │
│                           │  │Home│ │Mat.│ │Prog│ │Prof│ │  │
│                           │  └──┬─┘ └──┬─┘ └──┬─┘ └──┬─┘ │  │
│                           └────┼──────┼──────┼──────┼────┘  │
│                                │      │      │      │       │
│                                ▼      │      │      │       │
│                           [Home]      │      │      │       │
│                                       ▼      │      │       │
│                              ┌─[MaterialsList]│      │       │
│                              │        │       │      │       │
│                              │   ┌────▼────┐  │      │       │
│                              │   │Detail   │  │      │       │
│                              │   └────┬────┘  │      │       │
│                              │   ┌────┴────┐  │      │       │
│                              │   ▼         ▼  │      │       │
│                              │ [Reader] [Quiz]│      │       │
│                              │           │    │      │       │
│                              │      ┌────▼───┐│      │       │
│                              │      │Results ││      │       │
│                              │      └────────┘│      │       │
│                              └────────────────┘      │       │
│                                              ┌───────▼─────┐ │
│                                              │  Progress   │ │
│                                              │  Dashboard  │ │
│                                              └─────────────┘ │
│                                                      ┌───────▼─────┐
│                                                      │  Settings   │
│                                                      │  - Profile  │
│                                                      │  - Logout   │
│                                                      └─────────────┘
└─────────────────────────────────────────────────────────────┘
```

---

## Prioridad de Implementación

### Sprint 1-2: Core (Semanas 1-4)
1. ✅ Autenticación completa (RFC-001 a 004)
2. ✅ Infraestructura base (RFC-000, 003, 004)
3. ✅ Listado de materiales (RFC-020)

### Sprint 3-4: Lectura (Semanas 5-8)
4. ✅ Detalle de material (RFC-025)
5. ✅ Lector PDF (RFC-022)
6. ✅ Progreso de lectura (RFC-024)

### Sprint 5-6: Quizzes (Semanas 9-12)
7. ✅ Ver/hacer quiz (RFC-030, 031)
8. ✅ Resultados (RFC-032, 033)
9. ✅ Resúmenes IA (RFC-040, 041)

### Sprint 7-8: Pulido (Semanas 13-16)
10. ✅ Dashboard progreso (RFC-052)
11. ✅ Offline support mejorado
12. ✅ Polish y performance

---

## Consideraciones Técnicas

### Offline Support
- Materiales: Cache de metadata + descarga PDF para offline
- Quizzes: Cache de preguntas, sync de respuestas cuando hay conexión
- Progreso: Auto-save local, sync cuando hay conexión

### Notificaciones (futuro)
- Nuevo material disponible
- Quiz pendiente
- Recordatorio de lectura

### Performance
- Lazy loading de listas
- Prefetch de siguiente página del PDF
- Image caching para thumbnails
