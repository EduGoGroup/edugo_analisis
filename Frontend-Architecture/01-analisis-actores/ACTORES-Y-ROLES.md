# Análisis de Actores y Roles - EduGo Platform

## Actores Identificados en las RFCs

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           JERARQUÍA DE ACTORES                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                        ┌─────────────────┐                                   │
│                        │  SUPER ADMIN    │                                   │
│                        │  (Sistema)      │                                   │
│                        └────────┬────────┘                                   │
│                                 │                                            │
│              ┌──────────────────┼──────────────────┐                         │
│              ▼                  ▼                  ▼                         │
│     ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                   │
│     │   OWNER     │    │ COORDINATOR │    │   ADMIN     │                   │
│     │ (Director)  │    │             │    │ (Escuela)   │                   │
│     └──────┬──────┘    └──────┬──────┘    └──────┬──────┘                   │
│            │                  │                  │                           │
│            └──────────────────┼──────────────────┘                           │
│                               ▼                                              │
│                      ┌─────────────┐                                         │
│                      │   TEACHER   │                                         │
│                      │  (Docente)  │                                         │
│                      └──────┬──────┘                                         │
│                             │                                                │
│              ┌──────────────┼──────────────┐                                 │
│              ▼                             ▼                                 │
│     ┌─────────────┐               ┌─────────────┐                           │
│     │   STUDENT   │◄──────────────│  GUARDIAN   │                           │
│     │ (Estudiante)│   relación    │   (Padre)   │                           │
│     └─────────────┘               └─────────────┘                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Detalle por Actor

### 1. ESTUDIANTE (Student)
**Descripción**: Usuario principal de la plataforma. Consume contenido educativo.

| Característica | Detalle |
|----------------|---------|
| **Frecuencia de uso** | ALTA (diario) |
| **Dispositivos** | Móvil (principal), Tablet, Desktop, Web |
| **Necesita offline** | SÍ (crítico para lectura en transporte) |
| **API principal** | Mobile API (8080) |

**Funcionalidades (RFCs)**:
| RFC | Funcionalidad | Prioridad |
|-----|---------------|-----------|
| RFC-001 | Login con email/password | CRÍTICA |
| RFC-002 | Renovación tokens JWT | CRÍTICA |
| RFC-003 | Logout | ALTA |
| RFC-004 | Validación sesión | CRÍTICA |
| RFC-020 | Ver listado de materiales | ALTA |
| RFC-022 | Descargar/Leer PDFs | CRÍTICA |
| RFC-024 | Guardar progreso de lectura | ALTA |
| RFC-025 | Ver detalle de material | ALTA |
| RFC-030 | Obtener quiz generado | ALTA |
| RFC-031 | Responder quiz | ALTA |
| RFC-032 | Ver resultados de quiz | ALTA |
| RFC-033 | Ver historial de intentos | MEDIA |
| RFC-040 | Ver resumen IA | MEDIA |
| RFC-041 | Ver estado de procesamiento | MEDIA |
| RFC-052 | Dashboard de progreso personal | ALTA |

**Flujo principal**:
```
Login → Ver Materiales → Seleccionar Material → Leer PDF → Hacer Quiz → Ver Progreso
```

---

### 2. DOCENTE (Teacher)
**Descripción**: Crea y gestiona contenido educativo. Supervisa el progreso de estudiantes.

| Característica | Detalle |
|----------------|---------|
| **Frecuencia de uso** | MEDIA-ALTA (varios días por semana) |
| **Dispositivos** | Desktop (principal), Tablet, Móvil |
| **Necesita offline** | PARCIAL (para revisar, no para subir) |
| **API principal** | Mobile API (8080) + Admin API para reportes |

**Funcionalidades (RFCs)**:
| RFC | Funcionalidad | Prioridad |
|-----|---------------|-----------|
| RFC-001-004 | Autenticación completa | CRÍTICA |
| RFC-020 | Ver listado de materiales | ALTA |
| RFC-021 | Subir material PDF | CRÍTICA |
| RFC-022 | Descargar/Ver PDFs | ALTA |
| RFC-025 | Ver detalle de material | ALTA |
| RFC-002 (00) | Ver estado de procesamiento Worker | ALTA |
| RFC-041 | Manejo de estado procesando | ALTA |
| RFC-050 | Ver estadísticas por material | ALTA |
| RFC-051 | Ver estadísticas globales (limitado) | MEDIA |
| RFC-052 | Ver progreso de estudiantes | ALTA |

**Flujo principal**:
```
Login → Subir Material → Esperar Procesamiento → Ver Estadísticas → Revisar Progreso Estudiantes
```

---

### 3. GUARDIAN (Padre/Tutor)
**Descripción**: Supervisa el progreso académico de sus pupilos (hijos/tutorados).

| Característica | Detalle |
|----------------|---------|
| **Frecuencia de uso** | BAJA (semanal o mensual) |
| **Dispositivos** | Web (principal), Móvil (opcional) |
| **Necesita offline** | NO |
| **API principal** | Mobile API (8080) - solo lectura |

**Funcionalidades (RFCs)**:
| RFC | Funcionalidad | Prioridad |
|-----|---------------|-----------|
| RFC-001-004 | Autenticación | CRÍTICA |
| RFC-013 | Ver relaciones con pupilos | ALTA |
| RFC-052 | Ver dashboard de progreso de pupilos | CRÍTICA |

**Flujo principal**:
```
Login → Ver Lista de Pupilos → Seleccionar Pupilo → Ver Progreso Académico
```

**Nota**: RFC-013 está solo en rama `dev`, requiere merge a `main`.

---

### 4. ADMINISTRADOR (Admin/Owner/Coordinator)
**Descripción**: Gestiona la configuración de la escuela, usuarios y estructura académica.

| Característica | Detalle |
|----------------|---------|
| **Frecuencia de uso** | BAJA (mensual o por ciclo escolar) |
| **Dispositivos** | Desktop Web (exclusivo) |
| **Necesita offline** | NO |
| **API principal** | Admin API (8081) |

**Funcionalidades (RFCs)**:
| RFC | Funcionalidad | Prioridad |
|-----|---------------|-----------|
| RFC-001-004 | Autenticación | CRÍTICA |
| RFC-010 | CRUD de Escuelas | CRÍTICA |
| RFC-011 | Gestionar Jerarquía Académica | CRÍTICA |
| RFC-012 | Asignar Membresías (roles a usuarios) | CRÍTICA |
| RFC-013 | Gestionar Relaciones Acudientes | ALTA |
| RFC-014 | Gestionar Catálogo de Materias | ALTA |
| RFC-051 | Ver Estadísticas Globales | MEDIA |

**Flujo principal**:
```
Login → Configurar Escuela → Crear Jerarquía → Asignar Usuarios → Crear Materias
```

---

## Matriz de Permisos por Actor

| Funcionalidad | Student | Teacher | Guardian | Admin |
|---------------|---------|---------|----------|-------|
| **Autenticación** | ✅ | ✅ | ✅ | ✅ |
| **Ver Materiales** | ✅ Solo asignados | ✅ Todos de su sección | ❌ | ❌ |
| **Subir Materiales** | ❌ | ✅ | ❌ | ❌ |
| **Leer PDFs** | ✅ | ✅ | ❌ | ❌ |
| **Hacer Quiz** | ✅ | ❌ | ❌ | ❌ |
| **Ver Resultados Quiz** | ✅ Propios | ✅ De sus estudiantes | ✅ De pupilos | ❌ |
| **Ver Progreso** | ✅ Propio | ✅ De sus estudiantes | ✅ De pupilos | ✅ Global |
| **Gestionar Escuela** | ❌ | ❌ | ❌ | ✅ |
| **Gestionar Usuarios** | ❌ | ❌ | ❌ | ✅ |
| **Ver Estadísticas** | ❌ | ✅ Parcial | ❌ | ✅ Total |

---

## Aplicaciones por Actor

### Decisión: Separar por Rol Principal

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        MAPEO ACTOR → APLICACIÓN                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ACTOR          →    APLICACIÓN           →    PLATAFORMAS                 │
│   ─────────────────────────────────────────────────────────────────────     │
│   Student        →    EduGo Student        →    iOS, Android, Web, Desktop  │
│   Teacher        →    EduGo Teacher        →    iOS, Android, Web, Desktop  │
│   Guardian       →    EduGo Guardian       →    Web (móvil opcional)        │
│   Admin/Owner    →    EduGo Admin          →    Web (solo)                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Justificación de Apps Separadas

### Student App
- **UX optimizada** para consumo de contenido
- **Offline-first** crítico (estudiantes en transporte, zonas sin internet)
- **Gamificación** futura (badges, streaks)
- **Interfaz simplificada** sin distracciones

### Teacher App
- **Interfaz de creación** (upload, revisión)
- **Dashboards de seguimiento** de estudiantes
- **Notificaciones** de actividad estudiantil
- **Herramientas de productividad** (bulk actions)

### Guardian App (Web-first)
- **Frecuencia baja** no justifica app nativa
- **Solo lectura** de progreso
- **Responsive web** suficiente
- **Futuro**: notificaciones push via PWA

### Admin Dashboard (Solo Web)
- **Operaciones administrativas** complejas
- **Tablas, formularios, bulk operations** mejor en desktop
- **Frecuencia muy baja** (setup inicial, cambios anuales)
- **No requiere offline** ni acceso móvil

---

## Consideraciones de Roles Híbridos

### Escenario: Docente que también es Padre
```
Usuario: María García
Roles: Teacher (en Colegio ABC), Guardian (de su hijo Juan)

Solución A: Dos apps instaladas (recomendado)
- EduGo Teacher: para gestionar sus materiales
- EduGo Guardian: para ver progreso de Juan

Solución B: Cambio de contexto en una app (futuro)
- Selector de rol en perfil
- Switch entre vistas
```

### Escenario: Owner que también da clases
```
Usuario: Director López
Roles: Owner (del colegio), Teacher (da clases de matemáticas)

Solución: Admin Web + Teacher App
- Admin Dashboard: configuración escolar
- EduGo Teacher: para su rol docente
```

---

## Siguiente Documento

Ver [02-estructura-proyectos/](../02-estructura-proyectos/) para la arquitectura técnica de cada proyecto.
