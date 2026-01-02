# PROMPT DE CONTINUACIÓN: ANÁLISIS ECOSISTEMA EDUGO

**Fecha de Generación:** 2025-12-24
**Propósito:** Continuar análisis de frontend-readiness con 3 tareas delegadas a agentes
**Modo Requerido:** ULTRATHINK

---

## INSTRUCCIONES PARA CLAUDE

```
IMPORTANTE: Este prompt contiene el contexto completo de un análisis previo del ecosistema EduGo.

MODO DE OPERACIÓN:
1. USA ULTRATHINK para análisis profundo
2. DELEGA A AGENTES para preservar tu ventana de contexto
3. TÚ ERES ORQUESTADOR - no ejecutes directamente, delega
4. Cada agente debe recibir contexto completo y específico de su tarea
5. Los agentes deben ejecutar SIN ESPERAR APROBACIÓN

ESTRUCTURA DE TRABAJO:
- TAREA 1: Plan rápido "por donde pasa la novia" (desbloquear frontend)
- TAREA 2: Plan completo (terminar todos los puntos)
- TAREA 3: RFCs por proceso/subproceso para mobile

Cada plan debe estar organizado por: PROYECTO → FASE → PASOS DETALLADOS
```

---

## CONTEXTO DEL ECOSISTEMA

### Repositorios Analizados

| Proyecto | Ruta | Puerto | Tecnología |
|----------|------|--------|------------|
| edugo-api-administracion | `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion` | 8081 | Go 1.25, Gin, PostgreSQL |
| edugo-api-mobile | `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile` | 8080 | Go 1.25, Gin, PostgreSQL, MongoDB |
| edugo-worker | `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker` | N/A | Go, RabbitMQ, MongoDB, S3 |
| edugo-infrastructure | `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure` | N/A | PostgreSQL migrations, MongoDB schemas |
| edugo-shared | `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-shared` | N/A | Go, DTOs compartidos |
| edugo_analisis | `/Users/jhoanmedina/source/EduGo/repos-separados/edugo_analisis` | N/A | Documentación de análisis |

### Informes Generados (LEER ESTOS ARCHIVOS)

```
/Users/jhoanmedina/source/EduGo/repos-separados/edugo_analisis/analisis-frontend-readiness/
├── informes-agentes/
│   ├── INFORME_API_ADMIN.md        ← Análisis completo API Admin (38 endpoints)
│   ├── INFORME_API_MOBILE.md       ← Análisis completo API Mobile (18 endpoints)
│   ├── INFORME_WORKER.md           ← Análisis completo Worker (5 processors)
│   └── INFORME_INFRAESTRUCTURA.md  ← Análisis BD (20 tablas PG, 9 colecciones Mongo)
├── CONSOLIDADO_ECOSISTEMA.md       ← Resumen ejecutivo consolidado
├── SALUD_CONTRATOS.md              ← Análisis de eventos API↔Worker (CRÍTICO)
├── COBERTURA_TABLAS_ENDPOINTS.md   ← Mapeo tablas vs endpoints
└── ENDPOINTS_VIABLES_FRONTEND.md   ← Clasificación endpoints para frontend
```

---

## HALLAZGOS CRÍTICOS DEL ANÁLISIS PREVIO

### 1. BLOQUEANTES PARA FRONTEND

#### 1.1 API Admin - CORS No Configurado
**Severidad:** 🔴 CRÍTICO
**Archivo:** `edugo-api-administracion/cmd/main.go`
**Problema:** El middleware CORS no está aplicado en main.go
**Solución:** Agregar `r.Use(corsMiddleware())`
**Tiempo estimado:** 30 minutos

#### 1.2 API Admin - Rama dev adelantada 7,900 líneas
**Severidad:** 🔴 CRÍTICO
**Endpoints solo en dev (NO en main):**
- POST/GET/PATCH/DELETE `/v1/subjects`
- POST/GET/PUT/DELETE `/v1/guardian-relations`
- GET `/v1/guardians/:guardian_id/relations`
- GET `/v1/students/:student_id/guardians`
**Solución:** Merge dev → main con PR
**Tiempo estimado:** 2-4 horas (incluye review)

#### 1.3 API Mobile - Rama dev adelantada
**Severidad:** 🟡 MEDIO
**Diferencias:** PUT `/v1/materials/:id` solo en dev
**Solución:** Merge dev → main con PR
**Tiempo estimado:** 1-2 horas

#### 1.4 Worker - Dependencia para 3 Endpoints
**Severidad:** 🔴 CRÍTICO (pero degradable)
**Endpoints afectados:**
- GET `/v1/materials/:id/summary` → 404 sin Worker
- GET `/v1/materials/:id/assessment` → 404 sin Worker
- POST `/v1/materials/:id/assessment/attempts` → 404 sin Worker
**Nota:** Frontend puede avanzar con polling y mensajes de "procesando"
**Tiempo estimado:** N/A (Worker ya funciona, es dependencia de runtime)

### 2. CONTRATOS ROTOS API Mobile ↔ Worker

#### 2.1 Evento `material.uploaded` - INCOMPATIBLE
**Severidad:** 🔴 CRÍTICO
**Problema:** DTOs diferentes entre publicador y consumidor

**API Mobile publica (events.go):**
```go
type MaterialUploadedPayload struct {
    MaterialID    string `json:"material_id"`
    SchoolID      string `json:"school_id"`      // ⚠️ Worker NO tiene
    TeacherID     string `json:"teacher_id"`     // ⚠️ Worker usa AuthorID
    FileURL       string `json:"file_url"`       // ⚠️ Worker usa S3Key
    FileSizeBytes int64  `json:"file_size_bytes"` // ⚠️ Worker NO tiene
    FileType      string `json:"file_type"`      // ⚠️ Worker NO tiene
}
```

**Worker espera (event_dto.go):**
```go
type MaterialUploadedEvent struct {
    MaterialID        string `json:"material_id"`
    AuthorID          string `json:"author_id"`          // ⚠️ DIFERENTE
    S3Key             string `json:"s3_key"`             // ⚠️ DIFERENTE
    PreferredLanguage string `json:"preferred_language"` // ⚠️ API NO envía
}
```

**Solución Rápida:** Actualizar Worker para aceptar formato de API Mobile
**Solución Completa:** Crear DTOs compartidos en edugo-shared

#### 2.2 Eventos Huérfanos (publicados sin consumidor)
- `material.completed` - API Mobile publica, nadie consume
- `assessment.generated` - API Mobile publica, nadie consume

#### 2.3 Processors sin Publicador
- `material_deleted` - Worker espera, nadie publica
- `assessment_attempt` - Worker espera (stub), nadie publica
- `student_enrolled` - Worker espera (stub), nadie publica
- `material_reprocess` - Worker espera, nadie publica

#### 2.4 Formato de Event Type Inconsistente
- API Mobile usa: `material.uploaded` (punto)
- Worker usa: `material_uploaded` (underscore)
- edugo-shared define: `material.uploaded` (punto) pero Worker no lo usa

### 3. GAP TABLA PROGRESS

**Severidad:** 🔴 CRÍTICO
**Problema:** Nombre de tabla incorrecto en código

**Migración (016_create_progress.up.sql):**
```sql
CREATE TABLE IF NOT EXISTS progress (...)
```

**Código API Mobile (progress_repository_impl.go):**
```go
INSERT INTO material_progress (...)  // ⚠️ TABLA NO EXISTE
FROM material_progress               // ⚠️ TABLA NO EXISTE
```

**Solución:** Cambiar `material_progress` → `progress` en el código

### 4. PROCESSORS STUB EN WORKER

**AssessmentAttemptProcessor:** Solo hace log, no procesa
**StudentEnrolledProcessor:** Solo hace log, no procesa

### 5. OPENAI NO ACTIVO EN WORKER

**Estado:** Estructura existe pero `callOpenAIAPI()` retorna error
**Fallback:** SmartClient con análisis básico de texto (~60% calidad)
**Impacto:** Resúmenes y quizzes con menor calidad

---

## RESUMEN DE ENDPOINTS POR ESTADO

### API ADMINISTRACIÓN (38 endpoints)

| Módulo | Listos | Parciales | Bloqueados |
|--------|--------|-----------|------------|
| Auth | 4 | 0 | 0 |
| Schools | 5 | 1 | 0 |
| Academic Units | 9 | 0 | 0 |
| Memberships | 8 | 0 | 0 |
| Subjects | 0 | 5 | 0 |
| Guardian Relations | 0 | 5 | 0 |
| Health | 0 | 1 | 0 |
| **TOTAL** | **26** | **12** | **0** |

**Parciales:** Sin paginación, solo en dev, CORS faltante

### API MOBILE (18 endpoints)

| Módulo | Listos | Parciales | Bloqueados |
|--------|--------|-----------|------------|
| Health | 2 | 0 | 0 |
| Materials | 5 | 2 | 0 |
| Assessments | 2 | 0 | 2 |
| Summaries | 0 | 0 | 1 |
| Progress | 1 | 0 | 0 |
| Stats | 2 | 0 | 0 |
| **TOTAL** | **12** | **2** | **3** |

**Bloqueados:** Dependen del Worker procesando

---

## TAREA 1: PLAN RÁPIDO "POR DONDE PASA LA NOVIA"

### Objetivo
Desbloquear al equipo de frontend lo más rápido posible. Solo lo estrictamente necesario para que puedan trabajar.

### Criterios de "Rápido"
- Solo fixes que bloquean completamente al frontend
- No refactorizaciones
- No mejoras de calidad
- No features nuevas
- Mínimo viable para que frontend avance

### Estructura del Plan

El agente debe generar un documento con esta estructura:

```markdown
# PLAN RÁPIDO: DESBLOQUEO FRONTEND

## PROYECTO: [nombre]
### FASE 1: [nombre descriptivo]
#### Pre-requisitos
- [ ] Validar rama dev actualizada: `git fetch && git status`
- [ ] Crear rama de trabajo: `git checkout -b fix/[descripcion]-[fecha]`

#### Pasos
1. [Paso detallado con archivo y líneas]
2. [Paso detallado con archivo y líneas]
...

#### Post-requisitos
- [ ] Documentar cambios en [archivo]
- [ ] Actualizar Swagger: `swag init -g cmd/main.go`
- [ ] Compilar: `go build ./...`
- [ ] Lint: `golangci-lint run`
- [ ] Tests unitarios: `go test ./... -short`
- [ ] Tests integración: `go test ./... -tags=integration`
- [ ] Commit atómico: `git commit -m "[tipo]: [descripción]"`
- [ ] Push: `git push -u origin [rama]`
- [ ] Crear PR a dev

### FASE 2: ...
```

### Alcance Tarea 1

**INCLUIR (solo lo bloqueante):**

1. **edugo-api-administracion**
   - Agregar CORS en main.go (30 min)
   - Merge dev → main (crear PR, no refactorizar)

2. **edugo-api-mobile**
   - Merge dev → main (crear PR, no refactorizar)

3. **edugo-worker**
   - Adaptar DTO `MaterialUploadedEvent` para aceptar formato de API Mobile
   - Solo mapear campos: TeacherID→AuthorID, FileURL→S3Key

4. **edugo-api-mobile (fix progress)**
   - Cambiar `material_progress` → `progress` en queries

**NO INCLUIR (dejar para Tarea 2):**
- Crear DTOs compartidos en edugo-shared
- Implementar consumers faltantes
- Implementar publishers faltantes
- Completar processors stub
- Activar OpenAI
- Paginación en endpoints
- RBAC en API Admin
- Mejoras en health checks

### Archivo de Salida
```
/Users/jhoanmedina/source/EduGo/repos-separados/edugo_analisis/planes-trabajo/PLAN_RAPIDO_DESBLOQUEO_FRONTEND.md
```

---

## TAREA 2: PLAN COMPLETO

### Objetivo
Terminar TODOS los puntos identificados en el análisis. Plan exhaustivo organizado por proyecto.

### Estructura del Plan

```markdown
# PLAN COMPLETO: CORRECCIÓN INTEGRAL ECOSISTEMA

## PROYECTO: [nombre]
### FASE 1: [nombre descriptivo]
#### Pre-requisitos
- [ ] Validar rama dev actualizada: `git fetch && git status`
- [ ] Crear rama de trabajo: `git checkout -b feat/[descripcion]-[fecha]`

#### Pasos
1. [Paso detallado]
   - Archivo: [ruta completa]
   - Líneas: [inicio-fin]
   - Cambio: [descripción técnica]
   - Test requerido: [tipo: unit/integration/e2e]

2. [Paso detallado]
...

#### Tests Requeridos
| Tipo | Archivo | Descripción | Ejecutar en |
|------|---------|-------------|-------------|
| Unit | xxx_test.go | [desc] | Local |
| Integration | xxx_integration_test.go | [desc] | CI/Pipeline |

#### Configuración TestContainer (si aplica)
```go
// Ejemplo de configuración compartida
```

#### Post-requisitos
- [ ] Documentar cambios en README o docs/
- [ ] Actualizar Swagger: `swag init -g cmd/main.go`
- [ ] Compilar: `go build ./...`
- [ ] Lint: `golangci-lint run`
- [ ] Tests unitarios: `go test ./... -short`
- [ ] Tests integración: `go test ./... -tags=integration`
- [ ] Validar TestContainer compartido
- [ ] Commit atómico: `git commit -m "[tipo]: [descripción]"`
- [ ] Push: `git push -u origin [rama]`
- [ ] Crear PR a dev con descripción detallada

### FASE 2: ...
```

### Alcance Tarea 2

**POR PROYECTO:**

#### 1. edugo-shared (Primero - Base para otros)
- Crear módulo `events/` con DTOs compartidos
- Definir Event envelope estándar
- Definir payloads para cada evento
- Estandarizar formato event_type (usar punto: `material.uploaded`)
- Tests unitarios para serialización/deserialización
- Documentar en README

#### 2. edugo-api-administracion
- Implementar RBAC (middleware por rol)
- Agregar paginación a endpoints de lista
- Mejorar health check (validar PostgreSQL)
- Migrar a DTOs compartidos de edugo-shared
- Implementar publisher para `student.enrolled` (cuando se crea membership)
- Tests de integración con TestContainer
- Actualizar Swagger completo
- Documentar nuevos endpoints

#### 3. edugo-api-mobile
- Migrar eventos a DTOs compartidos de edugo-shared
- Implementar publisher para `material.deleted`
- Implementar publisher para `assessment.attempt_recorded`
- Agregar paginación a lista de materiales
- Tests de integración con TestContainer
- Actualizar Swagger completo
- Documentar cambios

#### 4. edugo-worker
- Migrar a DTOs compartidos de edugo-shared
- Implementar consumer para `material.completed`
- Implementar consumer para `assessment.generated`
- Completar `AssessmentAttemptProcessor` (real, no stub)
- Completar `StudentEnrolledProcessor` (real, no stub)
- Implementar publicador de `material.deleted` al eliminar de MongoDB
- Activar OpenAI (configurar API key, manejar errores)
- Tests de integración
- Documentar flujos

#### 5. edugo-infrastructure
- Verificar que todas las tablas tengan migraciones up/down correctas
- Documentar esquema MongoDB con JSON Schema
- Agregar índices faltantes si los hay

### Archivo de Salida
```
/Users/jhoanmedina/source/EduGo/repos-separados/edugo_analisis/planes-trabajo/PLAN_COMPLETO_CORRECCION_ECOSISTEMA.md
```

---

## TAREA 3: RFCs PARA MOBILE

### Objetivo
Crear documentos RFC (Request for Comments) organizados por proceso/subproceso que sirvan como guía para el desarrollo del frontend mobile.

### Estructura de Organización

```
RFCs/
├── 01-autenticacion/
│   ├── RFC-001-login-usuario.md
│   ├── RFC-002-refresh-token.md
│   ├── RFC-003-logout.md
│   └── RFC-004-validacion-sesion.md
├── 02-gestion-escolar/
│   ├── RFC-010-crud-escuelas.md
│   ├── RFC-011-jerarquia-academica.md
│   ├── RFC-012-membresías.md
│   └── RFC-013-relaciones-acudientes.md
├── 03-materiales/
│   ├── RFC-020-listado-materiales.md
│   ├── RFC-021-subida-pdf.md
│   ├── RFC-022-descarga-material.md
│   ├── RFC-023-versionado-materiales.md
│   └── RFC-024-progreso-lectura.md
├── 04-evaluaciones/
│   ├── RFC-030-obtener-quiz.md
│   ├── RFC-031-enviar-intento.md
│   ├── RFC-032-ver-resultados.md
│   └── RFC-033-historial-intentos.md
├── 05-resumenes-ia/
│   ├── RFC-040-obtener-resumen.md
│   └── RFC-041-manejo-estado-procesando.md
├── 06-estadisticas/
│   ├── RFC-050-stats-material.md
│   └── RFC-051-stats-globales.md
└── 00-arquitectura/
    ├── RFC-000-flujo-datos-mobile.md
    ├── RFC-001-manejo-errores.md
    ├── RFC-002-polling-estados.md
    └── RFC-003-almacenamiento-local.md
```

### Estructura de Cada RFC

```markdown
# RFC-XXX: [Nombre del Proceso]

## Metadata
- **ID:** RFC-XXX
- **Proceso:** [Nombre del proceso padre]
- **Subproceso:** [Nombre específico]
- **Prioridad:** Alta/Media/Baja
- **Dependencias:** RFC-YYY, RFC-ZZZ
- **Estado API:** ✅ Listo / ⚠️ Parcial / ❌ Bloqueado

## Descripción
[Qué hace este proceso desde la perspectiva del usuario]

## Flujo de Usuario (UX)
1. Usuario hace X
2. App muestra Y
3. Usuario interactúa con Z
...

## Flujo de Datos (Técnico)

### Diagrama de Secuencia
```
Usuario → Mobile App → API → [Worker] → Response → Mobile App → Usuario
```

### Endpoints Involucrados
| Endpoint | Método | Descripción | Estado |
|----------|--------|-------------|--------|
| /v1/xxx | POST | [desc] | ✅ |

### Request/Response

**Request:**
```typescript
interface XxxRequest {
  field1: string;
  field2: number;
}
```

**Response:**
```typescript
interface XxxResponse {
  id: string;
  ...
}
```

## Estados y Transiciones
[Diagrama de estados si aplica]

## Manejo de Errores
| Código | Significado | Acción en UI |
|--------|-------------|--------------|
| 400 | Validación | Mostrar errores por campo |
| 401 | No autenticado | Redirigir a login |
| 404 | No encontrado | Mensaje específico |
| 500 | Error servidor | Mensaje genérico + retry |

## Consideraciones de UX
- [Skeleton loaders]
- [Estados de carga]
- [Mensajes de error]
- [Confirmaciones]

## Almacenamiento Local
- [Qué cachear]
- [TTL del cache]
- [Estrategia offline]

## Código de Ejemplo (Mobile)
```typescript
// Ejemplo de implementación
async function xxx() {
  // ...
}
```

## Notas de Implementación
- [Consideraciones técnicas]
- [Gotchas conocidos]
- [Optimizaciones sugeridas]
```

### Archivo de Salida
```
/Users/jhoanmedina/source/EduGo/repos-separados/edugo_analisis/RFCs/
```

---

## CONFIGURACIÓN DE AGENTES

### Agente 1: Plan Rápido
```
Prompt para agente:

TAREA: Generar plan de trabajo RÁPIDO para desbloquear frontend.

CONTEXTO: Lee los archivos en /Users/jhoanmedina/source/EduGo/repos-separados/edugo_analisis/analisis-frontend-readiness/

ALCANCE ESTRICTO:
- Solo fixes bloqueantes
- No refactorizaciones
- No mejoras de calidad
- Mínimo viable

INCLUIR:
1. edugo-api-administracion: CORS en main.go + PR dev→main
2. edugo-api-mobile: PR dev→main
3. edugo-worker: Adaptar DTO MaterialUploadedEvent
4. edugo-api-mobile: Fix tabla progress

ESTRUCTURA: Por PROYECTO → FASE → PASOS
Cada fase debe incluir: pre-requisitos, pasos, post-requisitos (docs, swagger, compile, lint, tests, commit, push, PR)

ARCHIVO SALIDA: /Users/jhoanmedina/source/EduGo/repos-separados/edugo_analisis/planes-trabajo/PLAN_RAPIDO_DESBLOQUEO_FRONTEND.md

NO ESPERES APROBACIÓN. EJECUTA AHORA.
```

### Agente 2: Plan Completo
```
Prompt para agente:

TAREA: Generar plan de trabajo COMPLETO para corregir todo el ecosistema.

CONTEXTO: Lee los archivos en /Users/jhoanmedina/source/EduGo/repos-separados/edugo_analisis/analisis-frontend-readiness/

ALCANCE: TODO lo identificado en los análisis.

PROYECTOS EN ORDEN:
1. edugo-shared (DTOs compartidos)
2. edugo-api-administracion (RBAC, paginación, health, publishers)
3. edugo-api-mobile (migrar eventos, publishers, paginación)
4. edugo-worker (consumers, processors completos, OpenAI)
5. edugo-infrastructure (verificación)

ESTRUCTURA: Por PROYECTO → FASE → PASOS DETALLADOS
Cada paso debe incluir: archivo, líneas, cambio, test requerido
Cada fase debe incluir: pre-requisitos, pasos, tests, configuración testcontainer, post-requisitos

TESTS:
- Unitarios: locales
- Integración con TestContainer: CI/Pipeline
- Compartir contenedores para rapidez

ARCHIVO SALIDA: /Users/jhoanmedina/source/EduGo/repos-separados/edugo_analisis/planes-trabajo/PLAN_COMPLETO_CORRECCION_ECOSISTEMA.md

NO ESPERES APROBACIÓN. EJECUTA AHORA.
```

### Agente 3: RFCs (puede ser múltiples agentes en paralelo)
```
Prompt para agente(s):

TAREA: Generar RFCs por proceso/subproceso para desarrollo mobile.

CONTEXTO: Lee los archivos en /Users/jhoanmedina/source/EduGo/repos-separados/edugo_analisis/analisis-frontend-readiness/

ORGANIZACIÓN POR PROCESOS:
1. Autenticación (4 RFCs)
2. Gestión Escolar (4 RFCs)
3. Materiales (5 RFCs)
4. Evaluaciones (4 RFCs)
5. Resúmenes IA (2 RFCs)
6. Estadísticas (2 RFCs)
7. Arquitectura (4 RFCs)

CADA RFC DEBE INCLUIR:
- Metadata (ID, proceso, prioridad, dependencias, estado API)
- Descripción del usuario
- Flujo UX
- Flujo de datos técnico
- Endpoints con request/response TypeScript
- Estados y transiciones
- Manejo de errores por código HTTP
- Consideraciones UX (loaders, estados, mensajes)
- Almacenamiento local (cache, TTL, offline)
- Código de ejemplo
- Notas de implementación

CARPETA SALIDA: /Users/jhoanmedina/source/EduGo/repos-separados/edugo_analisis/RFCs/

CREAR ESTRUCTURA DE CARPETAS:
- 00-arquitectura/
- 01-autenticacion/
- 02-gestion-escolar/
- 03-materiales/
- 04-evaluaciones/
- 05-resumenes-ia/
- 06-estadisticas/

NO ESPERES APROBACIÓN. EJECUTA AHORA.
```

---

## ORDEN DE EJECUCIÓN SUGERIDO

1. **Primero:** Lanzar Agente 1 (Plan Rápido) - Es corto y urgente
2. **Segundo:** Lanzar Agente 2 (Plan Completo) - Más extenso
3. **Tercero:** Lanzar Agentes 3.x (RFCs) - Pueden ser paralelos por proceso

Los agentes 2 y 3 pueden ejecutarse en paralelo ya que son independientes.

---

## VALIDACIÓN FINAL

Al terminar todos los agentes, el orquestador debe:

1. Verificar que se crearon los archivos:
   - `planes-trabajo/PLAN_RAPIDO_DESBLOQUEO_FRONTEND.md`
   - `planes-trabajo/PLAN_COMPLETO_CORRECCION_ECOSISTEMA.md`
   - `RFCs/` con todas las subcarpetas y archivos

2. Crear un archivo índice:
   - `planes-trabajo/INDICE_PLANES.md`
   - `RFCs/INDICE_RFCs.md`

3. Reportar resumen al usuario con:
   - Archivos generados
   - Tiempo estimado total por plan
   - Dependencias críticas
   - Recomendación de orden de ejecución

---

## NOTAS ADICIONALES PARA EL AGENTE ORQUESTADOR

### Sobre Tests
- **Tests Unitarios:** Deben poder correrse localmente sin dependencias externas
- **Tests de Integración:** Usar TestContainer para PostgreSQL, MongoDB, RabbitMQ
- **TestContainer Compartido:** Configurar para reutilizar contenedores entre tests
- **CI/Pipeline:** Los tests de integración pesados van al pipeline, no bloquean desarrollo local

### Sobre Commits
- Usar Conventional Commits: `feat:`, `fix:`, `docs:`, `test:`, `refactor:`
- Commits atómicos: un commit por cambio lógico
- No mezclar fixes con features

### Sobre PRs
- Título descriptivo
- Descripción con contexto
- Checklist de validaciones
- Link a documentación actualizada

### Sobre Documentación
- Actualizar README si cambia setup
- Actualizar Swagger si cambian endpoints
- Actualizar docs/ si cambian flujos
- Mantener CHANGELOG.md si existe

---

## INICIO DE EJECUCIÓN

```
Claude, al leer este prompt:

1. USA ULTRATHINK para planificar la ejecución
2. Crea la carpeta planes-trabajo/ si no existe
3. Crea la carpeta RFCs/ con subcarpetas si no existen
4. Lanza los agentes según el orden sugerido
5. Espera resultados
6. Genera índices
7. Reporta al usuario

IMPORTANTE: Delega TODO a agentes. Tú solo orquestas y reportas.
```

---

**FIN DEL PROMPT**
