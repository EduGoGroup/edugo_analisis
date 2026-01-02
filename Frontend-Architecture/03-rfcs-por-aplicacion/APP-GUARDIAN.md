# EduGo Guardian - Mapeo de RFCs

## Resumen de la Aplicación

**Propósito**: Aplicación para padres/tutores. Permite visualizar el progreso académico de sus pupilos (hijos, nietos, etc.) de manera simplificada.

**Plataformas**:
- **Web WASM principalmente** (edugo-kmp/app-guardian-web)
- Móvil opcional en fases futuras

**API Principal**: Mobile API (puerto 8080) - solo lectura

**Justificación Web-First**:
- Frecuencia de uso baja (semanal o mensual)
- Solo visualización (no creación de contenido)
- No requiere offline
- PWA puede cubrir necesidades básicas móviles

---

## RFCs Incluidas

### Módulo: Autenticación (Prioridad: CRÍTICA)

| RFC | Nombre | Pantallas | Componentes |
|-----|--------|-----------|-------------|
| RFC-001 | Login Email/Password | `LoginPage` | EmailField, PasswordField |
| RFC-002 | Renovación Tokens | Background | TokenRefreshService |
| RFC-003 | Cierre de Sesión | `ProfilePage` | LogoutButton |
| RFC-004 | Validación Sesión | App Startup | AuthGuard |

---

### Módulo: Relaciones con Pupilos (Prioridad: CRÍTICA)

| RFC | Nombre | Pantallas | Componentes |
|-----|--------|-----------|-------------|
| RFC-013 | Ver Relaciones | `PupilsListPage` | PupilCard, RelationshipBadge |

**NOTA: RFC-013 ya fue mergeado a main (PT-006). Endpoints disponibles.

---

### Módulo: Progreso de Pupilos (Prioridad: CRÍTICA)

| RFC | Nombre | Pantallas | Componentes |
|-----|--------|-----------|-------------|
| RFC-052 | Dashboard Progreso | `PupilProgressPage` | ProgressChart, SubjectCards, ActivityFeed |

---

## Wireframes

### Lista de Pupilos
```
┌─────────────────────────────────────┐
│  Mis Hijos/Pupilos                  │
├─────────────────────────────────────┤
│                                     │
│  Bienvenido/a, Carmen               │
│                                     │
│  ┌─────────────────────────────────┐│
│  │ 👦 Juan Pérez García            ││
│  │ Relación: Madre                 ││
│  │ 5° Primaria - Sección A         ││
│  │ ━━━━━━━━━━━━░░░ 75% progreso    ││
│  │ [Ver detalles →]                ││
│  └─────────────────────────────────┘│
│                                     │
│  ┌─────────────────────────────────┐│
│  │ 👧 María Pérez García           ││
│  │ Relación: Madre                 ││
│  │ 3° Primaria - Sección B         ││
│  │ ━━━━━━━━━━━━━━━━░░ 85% progreso ││
│  │ [Ver detalles →]                ││
│  └─────────────────────────────────┘│
│                                     │
└─────────────────────────────────────┘
```

### Detalle de Progreso
```
┌─────────────────────────────────────┐
│ ← Juan Pérez García                 │
├─────────────────────────────────────┤
│                                     │
│  5° Primaria - Sección A            │
│  Colegio ABC                        │
│                                     │
│  Resumen General                    │
│  ┌─────────────────────────────────┐│
│  │  ┌────┐ ┌────┐ ┌────┐ ┌────┐   ││
│  │  │ 12 │ │ 8  │ │75% │ │ 5h │   ││
│  │  │Mat.│ │Quiz│ │Prom│ │Lect│   ││
│  │  │Leid│ │Hech│ │Quiz│ │Sem │   ││
│  │  └────┘ └────┘ └────┘ └────┘   ││
│  └─────────────────────────────────┘│
│                                     │
│  Progreso por Materia               │
│  ┌─────────────────────────────────┐│
│  │ Matemáticas         ━━━━░░ 80% ││
│  │ Último quiz: 85% ✓              ││
│  │                                 ││
│  │ Ciencias            ━━━━━░ 90% ││
│  │ Último quiz: 92% ✓              ││
│  │                                 ││
│  │ Español             ━━━░░░ 65% ││
│  │ Quiz pendiente ⚠️                ││
│  │                                 ││
│  │ Historia            ━━░░░░ 45% ││
│  │ Materiales sin leer             ││
│  └─────────────────────────────────┘│
│                                     │
│  Actividad Reciente                 │
│  ┌─────────────────────────────────┐│
│  │ 📖 Leyó "Álgebra Cap. 3"       ││
│  │    Ayer, 18:30                  ││
│  │                                 ││
│  │ ✅ Completó quiz Ciencias       ││
│  │    Hace 2 días - 92%            ││
│  │                                 ││
│  │ 📖 Leyó "Fotosíntesis"         ││
│  │    Hace 3 días                  ││
│  └─────────────────────────────────┘│
│                                     │
└─────────────────────────────────────┘
```

---

## Navegación de la App

```
┌─────────────────────────────────────────────────────────────┐
│                    EduGo Guardian Navigation                 │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  [Splash] ──▶ [Login] ──▶ [PupilsList] ──▶ [PupilProgress]  │
│                               │                              │
│                               └──▶ [Profile/Settings]        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Comparación con Otras Apps

| Característica | Student | Teacher | Guardian | Admin |
|----------------|---------|---------|----------|-------|
| Ver progreso propio | ✅ | ❌ | ❌ | ❌ |
| Ver progreso de otros | ❌ | ✅ estudiantes | ✅ pupilos | ❌ |
| Crear contenido | ❌ | ✅ | ❌ | ❌ |
| Configurar escuela | ❌ | ❌ | ❌ | ✅ |
| Offline necesario | ✅ | Parcial | ❌ | ❌ |
| Plataformas | Todas | Todas | Web | Web |

---

## Prioridad de Implementación

Esta es la **última aplicación** en prioridad ya que:
1. RFC-013 ya disponible en main
2. Menor impacto en el flujo educativo principal
3. Funcionalidad limitada (solo lectura)

### Fase 4 (después de Admin) - Semanas 17-20
1. Autenticación (RFC-001-004) - reuso de componentes
2. Lista de pupilos (RFC-013)
3. Dashboard de progreso (RFC-052) - reuso de Student

---

## Consideraciones Especiales

### PWA Features
- Add to Home Screen
- Push notifications (opcional) para:
  - Quiz completado
  - Material nuevo disponible
  - Alertas de bajo rendimiento

### Responsive Design
- Optimizado para móvil aunque sea Web
- Layout adaptativo tablet/desktop

### Reuso de Componentes
- `ProgressChart` → del módulo :feature-progress
- `SubjectCards` → del módulo :feature-progress
- Tema y componentes base → :core-ui
