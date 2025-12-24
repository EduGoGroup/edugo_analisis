# INFORME DETALLADO DE ACCIONES SOBRE DATOS
## Ecosistema EduGo - Limpieza de Bases de Datos

**Fecha de Análisis:** 23 de Diciembre, 2025
**Analista:** Sistema de Análisis EduGo
**Alcance:** PostgreSQL, MongoDB, API-Admin, API-Mobile, Worker

---

## RESUMEN EJECUTIVO

### Hallazgos Principales

- **PostgreSQL:** 5 tablas sin uso detectado (0 referencias en código)
- **MongoDB:** 5 colecciones sin uso detectado (0 referencias en código)
- **Colecciones Duplicadas:** 2 casos de duplicación con schemas diferentes
- **Campos sin uso:** 3 campos que requieren decisión (eliminar o implementar)

### Impacto General

- **Riesgo Total:** BAJO - Las tablas/colecciones identificadas NO tienen dependencias en código productivo
- **Acciones Requeridas:** 15 acciones distribuidas en 4 categorías
- **Tiempo Estimado:** 12-16 horas de trabajo técnico
- **Dependencias:** Requiere coordinación entre equipos de API-Admin, API-Mobile y Worker

---

## 1. TABLAS/COLECCIONES A ELIMINAR

### 1.1 PostgreSQL - `user_active_context`

**Base de datos:** PostgreSQL
**Migración:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/migrations/structure/011_create_user_active_context.sql`

#### CÓDIGO A ELIMINAR

```
Archivos a eliminar:
1. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/migrations/structure/011_create_user_active_context.sql
2. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/migrations/testing/006_demo_user_active_context.sql

Archivos a modificar:
- Ninguno (no hay referencias en código)
```

#### IMPACTO EN APIs

- **API-Mobile:** ✅ Sin impacto (0 referencias)
- **API-Admin:** ✅ Sin impacto (0 referencias)
- **Worker:** ✅ Sin impacto (no aplica)

#### IMPACTO EN WORKER

No aplica - Esta tabla está diseñada para uso de APIs frontend.

#### RIESGO

**BAJO** - Justificación:
- Tabla creada para FASE 1 UI (selector de escuela) nunca implementada
- Sin entidades, repositorios o handlers que la referencien
- Sin foreign keys desde otras tablas
- Comentario en migración indica "Parte de FASE 1 UI Roadmap" (no implementado)

#### PASOS DE ELIMINACIÓN

1. Crear migración de rollback en PostgreSQL
2. Ejecutar en ambientes de desarrollo primero
3. Validar que no hay datos críticos (tabla vacía o solo datos de testing)
4. Eliminar archivos de migraciones
5. Ejecutar migración de eliminación en producción

---

### 1.2 PostgreSQL - `user_favorites`

**Base de datos:** PostgreSQL
**Migración:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/migrations/structure/012_create_user_favorites.sql`

#### CÓDIGO A ELIMINAR

```
Archivos a eliminar:
1. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/migrations/structure/012_create_user_favorites.sql
2. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/migrations/testing/007_demo_user_favorites.sql

Archivos a modificar:
- Ninguno (no hay referencias en código)
```

#### IMPACTO EN APIs

- **API-Mobile:** ✅ Sin impacto (0 referencias)
- **API-Admin:** ✅ Sin impacto (0 referencias)
- **Worker:** ✅ Sin impacto (no aplica)

#### IMPACTO EN WORKER

No aplica - Funcionalidad de UI.

#### RIESGO

**BAJO** - Justificación:
- Tabla diseñada para FASE 1 UI (funcionalidad de favoritos) no implementada
- Sin entidades, repositorios o endpoints
- Sin referencias en handlers
- Comentario indica "Parte de FASE 1 UI Roadmap - Funcionalidad de favoritos en apps"

---

### 1.3 PostgreSQL - `user_activity_log`

**Base de datos:** PostgreSQL
**Migración:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/migrations/structure/013_create_user_activity_log.sql`

#### CÓDIGO A ELIMINAR

```
Archivos a eliminar:
1. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/migrations/structure/013_create_user_activity_log.sql
2. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/migrations/testing/008_demo_user_activity_log.sql

Tipo ENUM a eliminar:
- activity_type (definido en misma migración)

Archivos a modificar:
- Ninguno (no hay referencias en código)
```

#### IMPACTO EN APIs

- **API-Mobile:** ✅ Sin impacto (0 referencias)
- **API-Admin:** ✅ Sin impacto (0 referencias)
- **Worker:** ✅ Sin impacto (no aplica)

#### IMPACTO EN WORKER

No aplica - Log de actividades de usuario para analytics UI.

#### RIESGO

**BAJO** - Justificación:
- Tabla para historial y analytics de usuario (FASE 1 UI) no implementada
- Sin entidades, repositorios o servicios
- Enum `activity_type` solo usado por esta tabla
- Comentario indica "Parte de FASE 1 UI Roadmap - Actividad reciente en apps"

#### NOTA IMPORTANTE

Si en el futuro se requiere tracking de actividades, **considerar usar MongoDB** (`analytics_events` collection) en lugar de PostgreSQL, ya que es más apropiado para datos de eventos no relacionales.

---

### 1.4 PostgreSQL - `feature_flags`

**Base de datos:** PostgreSQL
**Migración:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/migrations/structure/014_create_feature_flags.sql`

#### CÓDIGO A ELIMINAR

```
Archivos a eliminar:
1. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/migrations/structure/014_create_feature_flags.sql
2. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/migrations/testing/009_demo_feature_flags.sql

Archivos a modificar:
- Ninguno (no hay referencias en código)
```

#### IMPACTO EN APIs

- **API-Mobile:** ✅ Sin impacto (0 referencias)
- **API-Admin:** ✅ Sin impacto (0 referencias)
- **Worker:** ✅ Sin impacto (no aplica)

#### IMPACTO EN WORKER

No aplica - Sistema de feature flags para control remoto de apps.

#### RIESGO

**BAJO** - Justificación:
- Sistema de feature flags marcado como "Deuda Técnica"
- Requerido por Apple App (según comentario en migración)
- Sin implementación en APIs actuales
- Referencia a spec en ruta externa: `/Users/jhoanmedina/source/EduGo/EduUI/apple-app/docs/backend-specs/feature-flags/BACKEND-SPEC-FEATURE-FLAGS.md`
- **IMPORTANTE:** Tabla relacionada `feature_flag_overrides` también debe eliminarse

#### CONSIDERACIÓN FUTURA

Si se implementa Apple App en el futuro, evaluar soluciones SaaS de feature flags (LaunchDarkly, Firebase Remote Config) en lugar de implementación custom.

---

### 1.5 PostgreSQL - `feature_flag_overrides`

**Base de datos:** PostgreSQL
**Migración:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/migrations/structure/015_create_feature_flag_overrides.sql`

#### CÓDIGO A ELIMINAR

```
Archivos a eliminar:
1. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/migrations/structure/015_create_feature_flag_overrides.sql
2. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/migrations/testing/010_demo_feature_flag_overrides.sql

Archivos a modificar:
- Ninguno (no hay referencias en código)
```

#### IMPACTO EN APIs

- **API-Mobile:** ✅ Sin impacto (0 referencias)
- **API-Admin:** ✅ Sin impacto (0 referencias)
- **Worker:** ✅ Sin impacto (no aplica)

#### IMPACTO EN WORKER

No aplica.

#### RIESGO

**BAJO** - Justificación:
- Tabla dependiente de `feature_flags` (debe eliminarse junto con la tabla padre)
- Sin referencias en código
- Marcada como "Phase 2 de Feature Flags" (nunca implementado Phase 1)

---

### 1.6 MongoDB - `material_content`

**Base de datos:** MongoDB
**Migración:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/migrations/structure/002_material_content.go`

#### CÓDIGO A ELIMINAR

```
Archivos a eliminar:
1. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/migrations/structure/002_material_content.go
2. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/migrations/constraints/002_material_content_indexes.go
3. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/seeds/material_content.js
4. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/seeds/mongodb/material_content.js

Archivos a modificar en WORKER:
- Ninguno (no hay referencias en código del worker)
```

#### IMPACTO EN APIs

- **API-Mobile:** ✅ Sin impacto (0 referencias)
- **API-Admin:** ✅ Sin impacto (0 referencias)
- **Worker:** ✅ Sin impacto (0 referencias)

#### IMPACTO EN WORKER

✅ **Sin impacto** - Aunque la migración indica "Used by: worker", no hay repositorios, servicios o procesadores que la utilicen.

#### RIESGO

**BAJO** - Justificación:
- Colección diseñada para almacenar contenido extraído de PDFs (texto raw, contenido estructurado)
- Sin implementación en worker (procesamiento de PDFs no desarrollado)
- Sin entidades en `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/entities/`
- Sin repositorios en worker que la referencien

#### NOTA TÉCNICA

Esta colección fue diseñada para almacenar contenido procesado de materiales (PDF extraction, transcripts, etc.) pero la funcionalidad nunca fue implementada. El worker actual genera directamente summaries y assessments sin pasar por esta etapa intermedia.

---

### 1.7 MongoDB - `assessment_attempt_result`

**Base de datos:** MongoDB
**Migración:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/migrations/structure/003_assessment_attempt_result.go`

#### CÓDIGO A ELIMINAR

```
Archivos a eliminar:
1. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/migrations/structure/003_assessment_attempt_result.go
2. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/migrations/constraints/003_assessment_attempt_result_indexes.go
3. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/seeds/assessment_attempt_result.js
4. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/seeds/mongodb/assessment_attempt_result.js

Archivos a modificar:
- Ninguno (no hay referencias en código)
```

#### IMPACTO EN APIs

- **API-Mobile:** ✅ Sin impacto (0 referencias)
- **API-Admin:** ✅ Sin impacto (0 referencias)
- **Worker:** ✅ Sin impacto (no aplica)

#### IMPACTO EN WORKER

No aplica - Colección para resultados de attemps de estudiantes.

#### RIESGO

**BAJO** - Justificación:
- Diseñada para almacenar respuestas detalladas de assessments
- La información está actualmente en PostgreSQL (`assessment_attempts` y `assessment_answers`)
- Sin entidades ni repositorios que la usen
- Migración indica "Used by: api-mobile" pero no hay implementación

#### DECISIÓN ARQUITECTÓNICA

Los resultados de attemps se almacenan en PostgreSQL (datos relacionales) en lugar de MongoDB. Esta colección representa una decisión arquitectónica que fue revertida durante el desarrollo.

---

### 1.8 MongoDB - `audit_logs`

**Base de datos:** MongoDB
**Migración:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/migrations/structure/004_audit_logs.go`

#### CÓDIGO A ELIMINAR

```
Archivos a eliminar:
1. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/migrations/structure/004_audit_logs.go
2. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/migrations/constraints/004_audit_logs_indexes.go
3. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/seeds/audit_logs.js
4. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/seeds/mongodb/audit_logs.js

Archivos a modificar:
- Ninguno (no hay referencias en código)
```

#### IMPACTO EN APIs

- **API-Mobile:** ✅ Sin impacto (0 referencias)
- **API-Admin:** ✅ Sin impacto (0 referencias)
- **Worker:** ✅ Sin impacto (0 referencias)

#### IMPACTO EN WORKER

✅ **Sin impacto** - Aunque migración indica "Used by: api-mobile, worker", no hay implementación.

#### RIESGO

**MEDIO** - Justificación:
- Sistema de auditoría es importante para compliance y seguridad
- **PERO** no está implementado actualmente
- Sin logger de auditoría en APIs
- Sin middleware que registre eventos

#### RECOMENDACIÓN

**OPCIÓN A - ELIMINAR (recomendada para MVP):**
- Eliminar colección
- Implementar auditoría en fase posterior con herramienta especializada

**OPCIÓN B - IMPLEMENTAR:**
- Crear middleware de auditoría en API-Admin
- Implementar logger de eventos en Worker
- Crear endpoints de consulta de logs
- Tiempo estimado: 16-20 horas

**Decisión recomendada:** ELIMINAR y agregar a roadmap para Fase 2 con solución SaaS (Elastic, CloudWatch Logs, etc.)

---

### 1.9 MongoDB - `notifications`

**Base de datos:** MongoDB
**Migración:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/migrations/structure/005_notifications.go`

#### CÓDIGO A ELIMINAR

```
Archivos a eliminar:
1. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/migrations/structure/005_notifications.go
2. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/migrations/constraints/005_notifications_indexes.go
3. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/seeds/notifications.js
4. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/seeds/mongodb/notifications.js

Archivos a modificar:
- /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile/.github/testing-config.yml
  (eliminar referencias a "notifications" en configuración de testing)
```

#### IMPACTO EN APIs

- **API-Mobile:** ⚠️ Impacto mínimo (solo en archivo de testing config)
- **API-Admin:** ✅ Sin impacto (0 referencias)
- **Worker:** ✅ Sin impacto (no aplica)

#### IMPACTO EN WORKER

No aplica - Sistema de notificaciones para usuarios.

#### RIESGO

**BAJO** - Justificación:
- Sistema de notificaciones no implementado
- Solo referencia en archivo de configuración de testing
- Sin repositorios, servicios o handlers
- Migración indica "Used by: api-mobile" pero sin implementación

#### NOTA

Única referencia encontrada en `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile/.github/testing-config.yml` sugiere que estaba planificado pero nunca implementado.

---

### 1.10 MongoDB - `analytics_events`

**Base de datos:** MongoDB
**Migración:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/migrations/structure/006_analytics_events.go`

#### CÓDIGO A ELIMINAR

```
Archivos a eliminar:
1. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/migrations/structure/006_analytics_events.go
2. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/migrations/constraints/006_analytics_events_indexes.go
3. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/seeds/analytics_events.js
4. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/seeds/mongodb/analytics_events.js

Archivos a modificar:
- Ninguno (no hay referencias en código)
```

#### IMPACTO EN APIs

- **API-Mobile:** ✅ Sin impacto (0 referencias)
- **API-Admin:** ✅ Sin impacto (0 referencias)
- **Worker:** ✅ Sin impacto (0 referencias)

#### IMPACTO EN WORKER

✅ **Sin impacto** - Aunque migración indica "Used by: api-mobile, worker", no hay implementación.

#### RIESGO

**MEDIO** - Justificación:
- Analytics es importante para entender comportamiento de usuarios
- **PERO** no está implementado actualmente
- Sin tracking de eventos en APIs
- Sin procesamiento de analytics en worker

#### RECOMENDACIÓN

**OPCIÓN A - ELIMINAR (recomendada):**
- Eliminar colección
- Usar solución SaaS de analytics (Google Analytics, Mixpanel, Amplitude)

**OPCIÓN B - IMPLEMENTAR:**
- Crear middleware de tracking en APIs
- Implementar procesador de eventos en Worker
- Crear dashboards de analytics
- Tiempo estimado: 40+ horas

**Decisión recomendada:** ELIMINAR y usar herramienta especializada de analytics.

---

## 2. TABLAS/COLECCIONES A HOMOLOGAR

### 2.1 MongoDB - `material_assessment` vs `material_assessment_worker`

**Problema:** Duplicación de colecciones con SCHEMAS DIFERENTES para el mismo propósito.

#### ANÁLISIS DE COLECCIONES

**Collection 1: `material_assessment`** (API-Mobile)
- **Ubicación:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/migrations/structure/001_material_assessment.go`
- **Usado por:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile/internal/infrastructure/persistence/mongodb/repository/assessment_document_repository.go`
- **Schema:** Simplificado, sin validaciones estrictas
- **Campos requeridos:** `material_id`, `questions`, `metadata`, `created_at`, `updated_at`
- **Validaciones:** Mínimas

**Collection 2: `material_assessment_worker`** (Worker)
- **Ubicación:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/migrations/structure/008_material_assessment_worker.go`
- **Usado por:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/internal/infrastructure/persistence/mongodb/repository/material_assessment_repository.go`
- **Schema:** Completo, con validaciones estrictas
- **Campos requeridos:** `material_id`, `questions`, `total_questions`, `total_points`, `version`, `ai_model`, `processing_time_ms`, `created_at`, `updated_at`
- **Validaciones:**
  - UUID v4 pattern para `material_id`
  - `questions` min 3, max 20
  - `difficulty` enum: easy, medium, hard
  - `question_type` enum: multiple_choice, true_false, open
  - `ai_model` enum: gpt-4, gpt-3.5-turbo, gpt-4-turbo, gpt-4o

#### VERSIÓN CANÓNICA

**Recomendación:** `material_assessment_worker` debe ser la versión canónica.

**Justificación:**
1. Schema más completo y robusto
2. Incluye metadata de procesamiento (ai_model, processing_time_ms)
3. Validaciones más estrictas
4. Refleja mejor la estructura de la entidad en `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/entities/material_assessment.go`
5. Worker es la fuente de verdad (genera los assessments)

#### CÓDIGO A MODIFICAR EN API-MOBILE

```
Archivos a modificar:
1. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile/internal/infrastructure/persistence/mongodb/repository/assessment_document_repository.go
   - Línea 83: Cambiar "material_assessment" → "material_assessment_worker"
   - Actualizar struct AssessmentDocument para incluir campos adicionales:
     * total_questions (int)
     * total_points (int)
     * version (int)
     * ai_model (string)
     * processing_time_ms (int)

2. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile/internal/application/service/assessment_attempt_service.go
   - Adaptar lógica para trabajar con campos adicionales del schema worker

Handlers afectados:
- /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile/internal/infrastructure/http/handler/assessment_handler.go
  (Sin cambios necesarios en handlers - cambios solo en capa de persistencia)
```

#### CÓDIGO A MODIFICAR EN API-ADMIN

```
No aplica - API-Admin no usa colección de assessments
```

#### CÓDIGO A MODIFICAR EN WORKER

```
Sin cambios - Worker ya usa material_assessment_worker
```

#### MIGRACIÓN REQUERIDA

**Tipo:** Migración de datos + Eliminación de colección antigua

**Pasos:**

1. **Validar datos existentes en `material_assessment`:**
```javascript
// Script de validación
db.material_assessment.find().forEach(function(doc) {
    // Verificar que todos los documentos tienen material_id
    if (!doc.material_id) {
        print("ERROR: Documento sin material_id: " + doc._id);
    }
});
```

2. **Migrar datos a `material_assessment_worker`:**
```javascript
// Script de migración
db.material_assessment.find().forEach(function(oldDoc) {
    var newDoc = {
        material_id: oldDoc.material_id,
        questions: oldDoc.questions,
        total_questions: oldDoc.questions ? oldDoc.questions.length : 0,
        total_points: calculateTotalPoints(oldDoc.questions),
        version: oldDoc.version || 1,
        ai_model: oldDoc.metadata?.generated_by || "unknown",
        processing_time_ms: 0, // No disponible en schema antiguo
        metadata: oldDoc.metadata || {},
        created_at: oldDoc.created_at,
        updated_at: oldDoc.updated_at
    };

    db.material_assessment_worker.replaceOne(
        { material_id: newDoc.material_id },
        newDoc,
        { upsert: true }
    );
});

function calculateTotalPoints(questions) {
    if (!questions) return 0;
    return questions.reduce((sum, q) => sum + (q.points || 1), 0);
}
```

3. **Verificar migración:**
```javascript
// Verificar conteo
var oldCount = db.material_assessment.count();
var newCount = db.material_assessment_worker.count();
print("Old: " + oldCount + ", New: " + newCount);

// Verificar sample
db.material_assessment_worker.findOne();
```

4. **Eliminar colección antigua:**
```javascript
// Solo después de validar en producción
db.material_assessment.drop();
```

5. **Eliminar archivos de migración antigua:**
```bash
rm /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/migrations/structure/001_material_assessment.go
rm /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/migrations/constraints/001_material_assessment_indexes.go
```

#### RIESGO

**MEDIO** - Justificación:
- Requiere migración de datos en producción
- Cambios en API-Mobile que requieren testing exhaustivo
- Posible downtime durante migración
- Necesidad de rollback plan

**Mitigación:**
1. Ejecutar migración en ambiente de staging primero
2. Crear backup completo antes de migración en producción
3. Migración con validación en cada paso
4. Mantener colección antigua hasta validar 100% funcionalidad
5. Rollback plan: revertir cambios en API-Mobile y mantener ambas colecciones temporalmente

#### TIEMPO ESTIMADO

- Modificación de código API-Mobile: 4-6 horas
- Creación y testing de scripts de migración: 3-4 horas
- Testing en staging: 2-3 horas
- Ejecución en producción: 1-2 horas
- **Total:** 10-15 horas

---

### 2.2 MongoDB - `material_summary` vs `material_summaries`

**Problema:** Duplicación de colecciones con NOMBRES DIFERENTES para el mismo propósito.

#### ANÁLISIS DE COLECCIONES

**Collection 1: `material_summary`** (Worker)
- **Ubicación:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/migrations/structure/007_material_summary.go`
- **Usado por:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/internal/infrastructure/persistence/mongodb/repository/material_summary_repository.go`
- **Schema:** Completo con validaciones estrictas
- **Entity:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/entities/material_summary.go`
- **Campos:** `material_id`, `summary`, `key_points`, `language`, `word_count`, `version`, `ai_model`, `processing_time_ms`, `metadata`, `created_at`, `updated_at`

**Collection 2: `material_summaries`** (API-Mobile - PLURAL)
- **Ubicación:** No tiene migración formal (creada dinámicamente)
- **Usado por:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile/internal/infrastructure/persistence/mongodb/repository/summary_repository_impl.go` (línea 20)
- **Schema:** Simplificado, sin validaciones
- **Campos:** `material_id`, `main_ideas`, `key_concepts`, `sections`, `glossary`, `created_at`

#### VERSIÓN CANÓNICA

**Recomendación:** `material_summary` (singular) debe ser la versión canónica.

**Justificación:**
1. Tiene migración formal con schema validation
2. Schema más completo con metadata de IA
3. Usado por Worker (fuente de verdad que genera summaries)
4. Tiene entidad definida en infrastructure
5. **Convención:** Colecciones MongoDB en singular (consistente con `material_assessment_worker`)

#### CÓDIGO A MODIFICAR EN API-MOBILE

```
Archivos a modificar:
1. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile/internal/infrastructure/persistence/mongodb/repository/summary_repository_impl.go

Cambios específicos:

Línea 20:
ANTES:
    collection: db.Collection("material_summaries"),
DESPUÉS:
    collection: db.Collection("material_summary"),

Líneas 24-37 (método Save):
ANTES:
    doc := bson.M{
        "material_id":  summary.MaterialID.String(),
        "main_ideas":   summary.MainIdeas,
        "key_concepts": summary.KeyConcepts,
        "sections":     summary.Sections,
        "glossary":     summary.Glossary,
        "created_at":   time.Now(),
    }

DESPUÉS:
    doc := bson.M{
        "material_id":  summary.MaterialID.String(),
        "summary":      generateSummaryText(summary.MainIdeas), // Nuevo campo
        "key_points":   summary.MainIdeas, // Renombrar
        "language":     "es", // Default español
        "word_count":   calculateWordCount(summary.MainIdeas),
        "version":      1,
        "ai_model":     "unknown", // Metadata no disponible en API
        "processing_time_ms": 0,
        "created_at":   time.Now(),
        "updated_at":   time.Now(),
    }

Líneas 40-70 (método FindByMaterialID):
- Adaptar para leer nuevos campos del schema worker
- Mapear "key_points" → "MainIdeas" en domain
```

2. `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile/internal/application/service/summary_service.go`
   - Adaptar para trabajar con nueva estructura de datos

Handlers afectados:
- `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile/internal/infrastructure/http/handler/summary_handler.go`
  (Sin cambios necesarios - cambios solo en capa de persistencia)
```

#### CÓDIGO A MODIFICAR EN API-ADMIN

```
No aplica - API-Admin no usa summaries
```

#### CÓDIGO A MODIFICAR EN WORKER

```
Sin cambios - Worker ya usa material_summary (singular, canónico)
```

#### MIGRACIÓN REQUERIDA

**Tipo:** Migración de datos + Rename collection

**Pasos:**

1. **Validar datos existentes en `material_summaries`:**
```javascript
// Contar documentos
var count = db.material_summaries.count();
print("Total documentos en material_summaries: " + count);

// Sample de datos
db.material_summaries.findOne();
```

2. **Migrar datos a `material_summary`:**
```javascript
// Script de migración con transformación de schema
db.material_summaries.find().forEach(function(oldDoc) {
    var newDoc = {
        material_id: oldDoc.material_id,
        summary: generateSummaryFromMainIdeas(oldDoc.main_ideas),
        key_points: oldDoc.main_ideas || [],
        language: detectLanguage(oldDoc.main_ideas) || "es",
        word_count: calculateWordCount(oldDoc.main_ideas),
        version: 1,
        ai_model: "migrated", // Marca para identificar datos migrados
        processing_time_ms: 0,
        metadata: {
            migrated_from: "material_summaries",
            original_created_at: oldDoc.created_at
        },
        created_at: oldDoc.created_at || new Date(),
        updated_at: new Date()
    };

    db.material_summary.replaceOne(
        { material_id: newDoc.material_id },
        newDoc,
        { upsert: true }
    );
});

// Helper functions
function generateSummaryFromMainIdeas(mainIdeas) {
    if (!mainIdeas || mainIdeas.length === 0) return "";
    return mainIdeas.join(". ") + ".";
}

function calculateWordCount(mainIdeas) {
    if (!mainIdeas) return 0;
    var text = mainIdeas.join(" ");
    return text.split(/\s+/).length;
}

function detectLanguage(mainIdeas) {
    // Lógica simple de detección
    if (!mainIdeas || mainIdeas.length === 0) return "es";
    var text = mainIdeas.join(" ").toLowerCase();
    // Palabras comunes en español
    if (text.match(/\b(el|la|los|las|de|del|y|que|en)\b/)) return "es";
    // Palabras comunes en inglés
    if (text.match(/\b(the|and|of|to|in|is|that)\b/)) return "en";
    return "es"; // Default
}
```

3. **Verificar migración:**
```javascript
var oldCount = db.material_summaries.count();
var newCount = db.material_summary.count();
print("Old: " + oldCount + ", New: " + newCount);

// Verificar que todos los material_ids fueron migrados
db.material_summaries.find({}, {material_id: 1}).forEach(function(doc) {
    var exists = db.material_summary.findOne({material_id: doc.material_id});
    if (!exists) {
        print("ERROR: No migrado: " + doc.material_id);
    }
});
```

4. **Eliminar colección antigua:**
```javascript
// Solo después de validar en producción
db.material_summaries.drop();
```

#### RIESGO

**BAJO-MEDIO** - Justificación:
- Schemas similares, migración más simple que assessments
- Solo requiere cambios en API-Mobile (Worker ya usa schema correcto)
- Transformación de datos relativamente directa

**Mitigación:**
1. Backup completo antes de migración
2. Testing exhaustivo en staging
3. Mantener colección antigua hasta validar 100%
4. Monitorear logs de API-Mobile después de deploy

#### TIEMPO ESTIMADO

- Modificación de código API-Mobile: 3-4 horas
- Creación y testing de scripts de migración: 2-3 horas
- Testing en staging: 2 horas
- Ejecución en producción: 1 hora
- **Total:** 8-10 horas

---

## 3. CAMPOS A LIMPIAR

### 3.1 `users.email_verified`

**Tabla:** `users` (PostgreSQL)
**Campo:** `email_verified BOOLEAN NOT NULL DEFAULT false`
**Migración:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/migrations/001_create_users.up.sql` (línea 13)

#### ANÁLISIS

**Referencias encontradas:**
- Entity: `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/entities/user.go`
- Repository: `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion/internal/infrastructure/persistence/postgres/repository/user_repository_impl.go`
- Seeds: Múltiples archivos de testing

**Uso actual:**
- Campo presente en struct pero sin lógica de verificación
- Sin endpoints de verificación de email
- Sin envío de emails de verificación
- Siempre es `false` en datos reales

#### OPCIÓN A: ELIMINAR

**Archivos a modificar:**

```
1. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/migrations/001_create_users.up.sql
   - Crear migración nueva: 018_remove_email_verified.up.sql
   ALTER TABLE users DROP COLUMN email_verified;

2. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/entities/user.go
   - Eliminar campo EmailVerified de struct

3. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion/internal/infrastructure/persistence/postgres/repository/user_repository_impl.go
   - Eliminar referencias a email_verified en queries

4. Seeds/testing:
   - /Users/jhoanmedina/source/EduGo/repos-separados/edugo-dev-environment/seeds/postgresql/02_users.sql
   - /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/tools/mock-generator/pkg/generator/dataset_generator.go
```

**Riesgo:** BAJO - Sin lógica dependiente

#### OPCIÓN B: IMPLEMENTAR

**Endpoints a agregar:**

```
API-Admin:
POST /v1/auth/verify-email
  - Enviar email con token de verificación

GET /v1/auth/verify/:token
  - Validar token y marcar email_verified = true

Archivos a crear:
1. internal/application/service/email_verification_service.go
2. internal/infrastructure/email/smtp_client.go
3. internal/infrastructure/http/handler/email_verification_handler.go
4. Templates de email en templates/verification_email.html

Lógica a implementar:
- Generación de tokens de verificación (JWT o random)
- Almacenamiento temporal de tokens (Redis o tabla verification_tokens)
- Envío de emails (SMTP config)
- Middleware para proteger endpoints solo a usuarios verificados
```

**Tiempo estimado:** 12-16 horas

#### RECOMENDACIÓN

**ELIMINAR** - Justificación:
- Sistema MVP no requiere verificación de email
- Complejidad innecesaria en fase actual
- Puede agregarse en Fase 2 si se requiere
- Eliminar reduce deuda técnica

---

### 3.2 `materials.is_deleted` vs `deleted_at`

**Tabla:** `materials` (PostgreSQL)
**Problema:** REDUNDANCIA - Soft delete implementado con `deleted_at` pero sin campo `is_deleted`

#### ANÁLISIS

**Estado actual:**
- Tabla tiene campo `deleted_at TIMESTAMP` (línea 23)
- **NO** tiene campo `is_deleted BOOLEAN`
- Soft delete implementado correctamente con `deleted_at`

**NOTA:** El análisis inicial indicó que había duplicación, pero después de revisar la migración:
```sql
-- /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/migrations/005_create_materials.up.sql
CREATE TABLE IF NOT EXISTS materials (
    ...
    deleted_at TIMESTAMP WITH TIME ZONE
);
```

**NO hay campo `is_deleted`** - Solo existe `deleted_at`.

#### CONCLUSIÓN

✅ **NO HAY ACCIÓN REQUERIDA**

La tabla `materials` implementa soft delete correctamente usando solo `deleted_at`. No hay duplicación ni redundancia.

**Queries correctas:**
```sql
-- Material activo
WHERE deleted_at IS NULL

-- Material eliminado
WHERE deleted_at IS NOT NULL

-- Soft delete
UPDATE materials SET deleted_at = NOW() WHERE id = $1
```

---

### 3.3 `schools.subscription_tier`, `max_teachers`, `max_students`

**Tabla:** `schools` (PostgreSQL)
**Campos:**
- `subscription_tier VARCHAR(50) NOT NULL DEFAULT 'free'`
- `max_teachers INTEGER NOT NULL DEFAULT 10`
- `max_students INTEGER NOT NULL DEFAULT 100`

**Migración:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/migrations/002_create_schools.up.sql` (líneas 17-19)

#### ANÁLISIS

**Referencias encontradas:**
- Entity: `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/entities/school.go`
- Service: `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion/internal/application/service/school_service.go`
- DTO: `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion/internal/application/dto/school_dto.go`
- Config: `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion/config/config.yaml`

**Uso actual:**
- Campos presentes en structs y DTOs
- **SIN enforcement:** No hay validación que impida crear más teachers/students que el límite
- **SIN lógica de negocio:** No hay restricciones basadas en subscription_tier
- Todos los valores siempre en default (free, 10, 100)

#### OPCIÓN A: ELIMINAR

**Archivos a modificar:**

```
1. Migración nueva: 018_remove_subscription_fields.up.sql
   ALTER TABLE schools
   DROP COLUMN subscription_tier,
   DROP COLUMN max_teachers,
   DROP COLUMN max_students;

2. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/entities/school.go
   - Eliminar campos SubscriptionTier, MaxTeachers, MaxStudents

3. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion/internal/application/dto/school_dto.go
   - Eliminar campos de DTOs

4. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion/internal/infrastructure/persistence/postgres/repository/school_repository_impl.go
   - Actualizar queries (eliminar referencias a campos)

5. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion/config/config.yaml
   - Eliminar configuraciones de subscription

6. Seeds:
   - /Users/jhoanmedina/source/EduGo/repos-separados/edugo-dev-environment/seeds/postgresql/01_schools.sql
```

**Riesgo:** BAJO - Sin lógica de enforcement actual

#### OPCIÓN B: IMPLEMENTAR

**Lógica a implementar:**

```
1. Middleware de validación de límites:
   - Verificar max_teachers antes de crear membership de teacher
   - Verificar max_students antes de crear membership de student
   - Retornar error 403 si se excede límite

Archivos a crear/modificar:
- internal/domain/service/subscription_validator.go
- internal/application/service/membership_service.go (agregar validación)

2. Endpoints de gestión de subscripción:
POST /v1/schools/:id/subscription
  - Actualizar tier de subscripción
  - Recalcular límites basados en tier

GET /v1/schools/:id/usage
  - Retornar uso actual vs límites

3. Lógica de tiers:
const subscriptionLimits = {
    free: { teachers: 10, students: 100 },
    basic: { teachers: 50, students: 500 },
    premium: { teachers: 200, students: 2000 },
    enterprise: { teachers: unlimited, students: unlimited }
}

4. Testing:
- Tests de validación de límites
- Tests de upgrade/downgrade de subscription
```

**Tiempo estimado:** 16-20 horas

#### RECOMENDACIÓN

**ELIMINAR** - Justificación:
- Sistema de subscripciones no es requerimiento MVP
- Complejidad significativa (validaciones, billing, etc.)
- Puede implementarse en Fase posterior con solución SaaS (Stripe, Chargebee)
- Todos los clientes actuales en tier "free" sin diferenciación

**Alternativa futura:** Implementar con servicio externo de billing que maneje tiers, límites y pagos.

---

## 4. TABLAS/COLECCIONES A AGREGAR

### 4.1 PostgreSQL - `user_sessions`

**Propósito:** Gestión de sesiones de usuario con tokens JWT

#### JUSTIFICACIÓN

Actualmente el sistema de autenticación genera JWT tokens pero no hay:
- Tracking de sesiones activas
- Capacidad de invalidar tokens
- Logout en todos los dispositivos
- Protección contra tokens robados

#### TABLA PROPUESTA

```sql
-- Migración: 018_create_user_sessions.up.sql
CREATE TABLE IF NOT EXISTS user_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash VARCHAR(255) NOT NULL UNIQUE,
    device_info JSONB DEFAULT '{}',
    ip_address VARCHAR(45),
    user_agent TEXT,
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    last_activity_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    revoked_at TIMESTAMP WITH TIME ZONE,

    CONSTRAINT chk_expires_future CHECK (expires_at > created_at)
);

CREATE INDEX idx_user_sessions_user ON user_sessions(user_id);
CREATE INDEX idx_user_sessions_token ON user_sessions(token_hash);
CREATE INDEX idx_user_sessions_expires ON user_sessions(expires_at);
CREATE INDEX idx_user_sessions_active ON user_sessions(user_id, revoked_at)
    WHERE revoked_at IS NULL;

COMMENT ON TABLE user_sessions IS 'Sesiones activas de usuarios para gestión de tokens JWT';
COMMENT ON COLUMN user_sessions.token_hash IS 'Hash SHA256 del token JWT para búsqueda rápida';
COMMENT ON COLUMN user_sessions.revoked_at IS 'Si NOT NULL, sesión fue invalidada (logout)';
```

#### ENTIDADES/REPOSITORIOS A CREAR

```
1. /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/entities/user_session.go

package entities

type UserSession struct {
    ID             string    `db:"id"`
    UserID         string    `db:"user_id"`
    TokenHash      string    `db:"token_hash"`
    DeviceInfo     JSONB     `db:"device_info"`
    IPAddress      *string   `db:"ip_address"`
    UserAgent      *string   `db:"user_agent"`
    ExpiresAt      time.Time `db:"expires_at"`
    LastActivityAt time.Time `db:"last_activity_at"`
    CreatedAt      time.Time `db:"created_at"`
    RevokedAt      *time.Time `db:"revoked_at"`
}

2. API-Admin/API-Mobile: internal/domain/repository/user_session_repository.go

type UserSessionRepository interface {
    Create(ctx context.Context, session *UserSession) error
    FindByTokenHash(ctx context.Context, tokenHash string) (*UserSession, error)
    FindActiveByUserID(ctx context.Context, userID string) ([]*UserSession, error)
    UpdateLastActivity(ctx context.Context, sessionID string) error
    RevokeByID(ctx context.Context, sessionID string) error
    RevokeAllByUserID(ctx context.Context, userID string) error
    DeleteExpired(ctx context.Context) (int64, error)
}

3. API-Admin/API-Mobile: internal/infrastructure/persistence/postgres/repository/user_session_repository_impl.go
```

#### ENDPOINTS QUE LO USARÍAN

```
API-Admin y API-Mobile:

POST /v1/auth/login
  - Crear sesión al hacer login exitoso
  - Almacenar token_hash, device_info, ip_address

POST /v1/auth/logout
  - Revocar sesión actual
  - SET revoked_at = NOW()

POST /v1/auth/logout-all
  - Revocar todas las sesiones del usuario
  - Útil si se sospecha robo de credenciales

GET /v1/auth/sessions
  - Listar sesiones activas del usuario
  - Mostrar dispositivo, IP, última actividad

DELETE /v1/auth/sessions/:id
  - Revocar sesión específica
  - "Cerrar sesión en otro dispositivo"

Middleware de autenticación:
- Validar que token_hash existe en user_sessions
- Verificar que revoked_at IS NULL
- Actualizar last_activity_at
```

#### TIEMPO ESTIMADO

- Migración y entidad: 1 hora
- Repositorio: 2-3 horas
- Modificar auth service: 3-4 horas
- Endpoints de gestión de sesiones: 4-5 horas
- Testing: 3-4 horas
- **Total:** 13-17 horas

---

### 4.2 PostgreSQL - `password_reset_tokens`

**Propósito:** Tokens seguros para reset de contraseña

#### JUSTIFICACIÓN

Sistema de "Olvidé mi contraseña" no implementado. Actualmente no hay forma de que un usuario recupere acceso si olvida su contraseña.

#### TABLA PROPUESTA

```sql
-- Migración: 019_create_password_reset_tokens.up.sql
CREATE TABLE IF NOT EXISTS password_reset_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash VARCHAR(255) NOT NULL UNIQUE,
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    used_at TIMESTAMP WITH TIME ZONE,

    CONSTRAINT chk_expires_future CHECK (expires_at > created_at)
);

CREATE INDEX idx_password_reset_user ON password_reset_tokens(user_id);
CREATE INDEX idx_password_reset_token ON password_reset_tokens(token_hash);
CREATE INDEX idx_password_reset_expires ON password_reset_tokens(expires_at);

COMMENT ON TABLE password_reset_tokens IS 'Tokens de un solo uso para reset de contraseña';
COMMENT ON COLUMN password_reset_tokens.token_hash IS 'Hash SHA256 del token enviado por email';
COMMENT ON COLUMN password_reset_tokens.used_at IS 'Si NOT NULL, token ya fue usado';
```

#### ENTIDADES/REPOSITORIOS A CREAR

```
1. Entity:
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/entities/password_reset_token.go

2. Repository:
internal/domain/repository/password_reset_repository.go

type PasswordResetRepository interface {
    Create(ctx context.Context, token *PasswordResetToken) error
    FindValidByTokenHash(ctx context.Context, tokenHash string) (*PasswordResetToken, error)
    MarkAsUsed(ctx context.Context, tokenID string) error
    DeleteExpired(ctx context.Context) (int64, error)
}

3. Service:
internal/application/service/password_reset_service.go
```

#### ENDPOINTS QUE LO USARÍAN

```
API-Admin y API-Mobile:

POST /v1/auth/forgot-password
  - Body: { "email": "user@example.com" }
  - Genera token, envía email
  - Token válido por 1 hora

POST /v1/auth/reset-password
  - Body: { "token": "...", "new_password": "..." }
  - Valida token, actualiza contraseña
  - Marca token como usado

Requiere:
- Email service para envío de tokens
- Template de email con link de reset
```

#### TIEMPO ESTIMADO

- Migración y entidad: 1 hora
- Repositorio y service: 3-4 horas
- Email service integration: 4-5 horas
- Endpoints: 3-4 horas
- Testing: 3-4 horas
- **Total:** 14-18 horas

---

### 4.3 MongoDB - `student_progress`

**Propósito:** Tracking detallado de progreso de estudiantes en materiales

#### JUSTIFICACIÓN

La tabla PostgreSQL `progress` solo almacena:
- `material_id`
- `student_id`
- `status` (not_started, in_progress, completed)
- `progress_percentage`

Falta tracking de:
- Páginas/secciones vistas
- Tiempo dedicado por sección
- Historial de progreso
- Patrones de estudio

#### COLECCIÓN PROPUESTA

```javascript
// Migración: 009_student_progress.go
{
    material_id: "uuid",
    student_id: "uuid",

    // Progreso general
    status: "in_progress", // not_started, in_progress, completed
    overall_percentage: 45.5,

    // Tracking detallado
    sections_viewed: [
        {
            section_index: 0,
            section_title: "Introducción",
            viewed_at: ISODate(),
            time_spent_seconds: 120,
            completed: true
        },
        {
            section_index: 1,
            section_title: "Capítulo 1",
            viewed_at: ISODate(),
            time_spent_seconds: 300,
            completed: false
        }
    ],

    // Actividad
    first_accessed_at: ISODate(),
    last_accessed_at: ISODate(),
    total_time_spent_seconds: 420,
    access_count: 5,

    // Metadata
    device_info: {
        last_device: "mobile",
        last_os: "iOS 17"
    },

    created_at: ISODate(),
    updated_at: ISODate()
}

// Índices
db.student_progress.createIndex({ "student_id": 1, "material_id": 1 }, { unique: true });
db.student_progress.createIndex({ "student_id": 1, "last_accessed_at": -1 });
db.student_progress.createIndex({ "material_id": 1 });
```

#### ENTIDADES/REPOSITORIOS A CREAR

```
1. Entity:
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/entities/student_progress.go

2. Repository (API-Mobile):
internal/infrastructure/persistence/mongodb/repository/student_progress_repository.go

type StudentProgressRepository interface {
    Save(ctx context.Context, progress *StudentProgress) error
    FindByStudentAndMaterial(ctx context.Context, studentID, materialID string) (*StudentProgress, error)
    FindRecentByStudent(ctx context.Context, studentID string, limit int) ([]*StudentProgress, error)
    UpdateSectionProgress(ctx context.Context, studentID, materialID string, section SectionProgress) error
}
```

#### ENDPOINTS QUE LO USARÍAN

```
API-Mobile:

PUT /v1/materials/:id/progress
  - Actualizar progreso (tracking de lectura)
  - Body: { section_index, time_spent, completed }

GET /v1/students/me/progress
  - Obtener progreso de todos los materiales del estudiante
  - Ordenado por último acceso

GET /v1/materials/:id/progress
  - Obtener progreso detallado de un material específico
  - Incluye secciones vistas, tiempo por sección

GET /v1/students/me/activity
  - Dashboard de actividad del estudiante
  - Materiales en progreso, tiempo total estudiado, etc.
```

#### SINCRONIZACIÓN CON POSTGRESQL

La tabla `progress` en PostgreSQL debe actualizarse en paralelo:
```
Cuando se actualiza student_progress en MongoDB:
1. Calcular overall_percentage
2. Determinar status (completed si overall_percentage == 100)
3. Actualizar tabla progress en PostgreSQL

Esto permite:
- Queries rápidas en PostgreSQL para reportes generales
- Tracking detallado en MongoDB para analytics
```

#### TIEMPO ESTIMADO

- Migración y entity: 2 horas
- Repository: 3-4 horas
- Service con sync a PostgreSQL: 4-5 horas
- Endpoints: 4-5 horas
- Testing: 3-4 horas
- **Total:** 16-20 horas

---

### 4.4 MongoDB - `material_feedback`

**Propósito:** Feedback y ratings de materiales por estudiantes

#### JUSTIFICACIÓN

No existe forma de que estudiantes:
- Califiquen materiales (estrellas)
- Den feedback sobre calidad
- Reporten problemas (material incompleto, errores)

Esto es valioso para:
- Mejorar calidad de materiales
- Identificar materiales problemáticos
- Analytics de satisfacción

#### COLECCIÓN PROPUESTA

```javascript
// Migración: 010_material_feedback.go
{
    material_id: "uuid",
    student_id: "uuid",

    // Rating
    rating: 4, // 1-5 estrellas

    // Feedback textual
    comment: "Excelente material, muy claro",

    // Categorías de feedback
    categories: {
        clarity: 5,        // 1-5
        usefulness: 4,     // 1-5
        difficulty: 3      // 1-5
    },

    // Reportes
    issues: [
        {
            type: "content_error",  // content_error, incomplete, unclear, technical
            description: "Falta página 5",
            reported_at: ISODate()
        }
    ],

    // Metadata
    device_info: {
        platform: "mobile",
        os: "iOS 17"
    },

    created_at: ISODate(),
    updated_at: ISODate()
}

// Índices
db.material_feedback.createIndex({ "material_id": 1, "student_id": 1 }, { unique: true });
db.material_feedback.createIndex({ "material_id": 1, "rating": 1 });
db.material_feedback.createIndex({ "student_id": 1, "created_at": -1 });
```

#### ENTIDADES/REPOSITORIOS A CREAR

```
1. Entity:
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/entities/material_feedback.go

2. Repository (API-Mobile):
internal/infrastructure/persistence/mongodb/repository/material_feedback_repository.go

type MaterialFeedbackRepository interface {
    Save(ctx context.Context, feedback *MaterialFeedback) error
    FindByMaterialID(ctx context.Context, materialID string) ([]*MaterialFeedback, error)
    GetAverageRating(ctx context.Context, materialID string) (float64, error)
    FindIssuesByMaterial(ctx context.Context, materialID string) ([]*MaterialFeedback, error)
}
```

#### ENDPOINTS QUE LO USARÍAN

```
API-Mobile:

POST /v1/materials/:id/feedback
  - Crear/actualizar feedback de material
  - Body: { rating, comment, categories, issues }

GET /v1/materials/:id/feedback
  - Obtener feedback del estudiante actual

GET /v1/materials/:id/rating
  - Obtener rating promedio del material
  - Response: { average: 4.2, count: 15 }

API-Admin:

GET /v1/materials/:id/feedback/all
  - Ver todos los feedbacks de un material
  - Útil para teachers/admins

GET /v1/materials/issues
  - Listar materiales con issues reportados
  - Priorizar arreglos
```

#### TIEMPO ESTIMADO

- Migración y entity: 1-2 horas
- Repository: 2-3 horas
- Service: 2-3 horas
- Endpoints: 3-4 horas
- Testing: 2-3 horas
- **Total:** 10-15 horas

---

## 5. RESUMEN EJECUTIVO

### Total de Acciones Requeridas

| Categoría | Cantidad | Prioridad | Tiempo Estimado |
|-----------|----------|-----------|-----------------|
| **Tablas/Colecciones a Eliminar** | 10 | Alta | 6-8 horas |
| **Duplicadas a Homologar** | 2 | Alta | 18-25 horas |
| **Campos a Limpiar** | 2 | Media | 2-3 horas |
| **Tablas/Colecciones a Agregar** | 4 | Baja | 53-70 horas |
| **TOTAL** | **18 acciones** | - | **79-106 horas** |

### Orden de Prioridad Sugerido

#### FASE 1: LIMPIEZA (Alta Prioridad) - Sprint 1

**Objetivo:** Eliminar deuda técnica y duplicaciones

1. **Eliminar tablas PostgreSQL sin uso** (4 horas)
   - `user_active_context`
   - `user_favorites`
   - `user_activity_log`
   - `feature_flags`
   - `feature_flag_overrides`

2. **Eliminar colecciones MongoDB sin uso** (2-4 horas)
   - `material_content`
   - `assessment_attempt_result`
   - `audit_logs`
   - `notifications`
   - `analytics_events`

3. **Eliminar campos sin uso** (2-3 horas)
   - `users.email_verified`
   - `schools.subscription_tier`, `max_teachers`, `max_students`

**Subtotal Fase 1:** 8-11 horas

---

#### FASE 2: HOMOLOGACIÓN (Alta Prioridad) - Sprint 2

**Objetivo:** Unificar colecciones duplicadas

4. **Homologar `material_summary` / `material_summaries`** (8-10 horas)
   - Menor riesgo, schema similar
   - Solo afecta API-Mobile

5. **Homologar `material_assessment` / `material_assessment_worker`** (10-15 horas)
   - Mayor complejidad, requiere migración de datos
   - Afecta API-Mobile

**Subtotal Fase 2:** 18-25 horas

---

#### FASE 3: NUEVAS FUNCIONALIDADES (Baja Prioridad) - Sprints 3-5

**Objetivo:** Implementar funcionalidades faltantes críticas

6. **Agregar `user_sessions`** (13-17 horas)
   - Crítico para seguridad
   - Habilita logout, gestión de sesiones

7. **Agregar `password_reset_tokens`** (14-18 horas)
   - Crítico para UX
   - Funcionalidad esperada por usuarios

8. **Agregar `student_progress`** (16-20 horas)
   - Importante para analytics
   - Mejora experiencia de estudiante

9. **Agregar `material_feedback`** (10-15 horas)
   - Nice to have
   - Mejora calidad de materiales

**Subtotal Fase 3:** 53-70 horas

---

### Dependencias entre Acciones

```
Diagrama de Dependencias:

Fase 1 (Limpieza)
├── Eliminar tablas PostgreSQL (paralelo)
│   ├── user_active_context
│   ├── user_favorites
│   ├── user_activity_log
│   ├── feature_flags
│   └── feature_flag_overrides
│
├── Eliminar colecciones MongoDB (paralelo)
│   ├── material_content
│   ├── assessment_attempt_result
│   ├── audit_logs
│   ├── notifications
│   └── analytics_events
│
└── Eliminar campos (paralelo)
    ├── email_verified
    └── subscription fields

    ↓ (No bloqueante)

Fase 2 (Homologación)
├── material_summaries → material_summary (puede hacerse primero)
│   └── Requiere: Modificar API-Mobile
│
└── material_assessment → material_assessment_worker (después de summaries)
    └── Requiere: Modificar API-Mobile, migración de datos

    ↓ (No bloqueante)

Fase 3 (Nuevas Funcionalidades) - Pueden hacerse en paralelo
├── user_sessions (independiente)
├── password_reset_tokens (independiente)
├── student_progress (independiente, pero beneficia de homologación)
└── material_feedback (independiente)
```

### Riesgos Generales

| Riesgo | Probabilidad | Impacto | Mitigación |
|--------|--------------|---------|------------|
| Pérdida de datos durante migración | Baja | Alto | Backups completos, testing en staging |
| Downtime en producción | Media | Medio | Migraciones fuera de horario pico |
| Regresiones en API-Mobile | Media | Alto | Testing exhaustivo, rollback plan |
| Tiempo de ejecución mayor a estimado | Alta | Bajo | Priorizar Fase 1 y 2, Fase 3 es opcional |

### Recomendaciones Finales

1. **Ejecutar Fase 1 inmediatamente**
   - Limpieza es de bajo riesgo
   - Reduce deuda técnica significativamente
   - Clarifica arquitectura para nuevos desarrolladores

2. **Fase 2 requiere coordinación**
   - Coordinar con equipo de API-Mobile
   - Testing exhaustivo en staging
   - Considerar feature flag para rollout gradual

3. **Fase 3 es opcional para MVP**
   - Evaluar prioridad según feedback de usuarios
   - `user_sessions` es la más crítica para seguridad
   - `password_reset` es la más crítica para UX
   - `student_progress` y `material_feedback` pueden postergarse

4. **Monitoreo post-cambios**
   - Logs de errores en APIs
   - Métricas de performance
   - Feedback de usuarios

---

## ANEXOS

### A. Scripts de Validación

#### A.1 Validar Referencias a Tablas Sin Uso

```bash
#!/bin/bash
# Validar que tablas marcadas para eliminación no tienen referencias

REPOS=(
    "/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile"
    "/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion"
    "/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker"
)

TABLES=(
    "user_active_context"
    "user_favorites"
    "user_activity_log"
    "feature_flags"
    "feature_flag_overrides"
)

for table in "${TABLES[@]}"; do
    echo "=== Buscando referencias a: $table ==="
    for repo in "${REPOS[@]}"; do
        echo "  Repositorio: $repo"
        grep -r "$table" "$repo" --include="*.go" --include="*.sql" || echo "    ✅ Sin referencias"
    done
    echo ""
done
```

#### A.2 Validar Datos en Colecciones MongoDB

```javascript
// Ejecutar en MongoDB shell
// Validar que colecciones sin uso están vacías o con solo datos de testing

const collections = [
    'material_content',
    'assessment_attempt_result',
    'audit_logs',
    'notifications',
    'analytics_events'
];

collections.forEach(coll => {
    const count = db[coll].count();
    print(`${coll}: ${count} documentos`);

    if (count > 0) {
        print(`  Sample:`);
        printjson(db[coll].findOne());
    }
});
```

### B. Checklist de Testing

#### B.1 Testing Post-Eliminación

```
□ Verificar que migraciones de eliminación se ejecutan sin errores
□ Verificar que APIs arrancan correctamente
□ Ejecutar test suite completo de cada API
□ Verificar que seeds/testing no fallan
□ Verificar que no hay referencias en logs de error
□ Testing manual de flujos principales:
  □ Login/Logout
  □ Crear escuela
  □ Subir material
  □ Tomar assessment
  □ Ver progreso
```

#### B.2 Testing Post-Homologación

```
□ API-Mobile arranca correctamente
□ Tests de integración pasan
□ Verificar endpoints de assessment:
  □ GET /v1/materials/:id/assessment
  □ POST /v1/materials/:id/assessment/attempts
□ Verificar endpoints de summary:
  □ GET /v1/materials/:id/summary
□ Validar estructura de datos en MongoDB
□ Verificar que Worker sigue funcionando correctamente
□ Testing de migración de datos:
  □ Todos los documentos migrados
  □ Sin pérdida de información
  □ Estructura correcta en nueva colección
```

### C. Comandos Útiles

#### C.1 Backup de PostgreSQL

```bash
# Backup completo
pg_dump -h localhost -U edugo -d edugo_db > backup_$(date +%Y%m%d_%H%M%S).sql

# Backup de tabla específica
pg_dump -h localhost -U edugo -d edugo_db -t users > users_backup.sql
```

#### C.2 Backup de MongoDB

```bash
# Backup completo
mongodump --uri="mongodb://localhost:27017/edugo_db" --out=backup_$(date +%Y%m%d_%H%M%S)

# Backup de colección específica
mongodump --uri="mongodb://localhost:27017/edugo_db" --collection=material_assessment --out=material_assessment_backup
```

#### C.3 Rollback de Migraciones

```bash
# PostgreSQL - rollback última migración
migrate -path /path/to/migrations -database "postgres://..." down 1

# MongoDB - no hay rollback automático, usar backup
mongorestore --uri="mongodb://localhost:27017/edugo_db" --dir=backup_folder
```

---

## CONCLUSIÓN

Este informe detalla **18 acciones** sobre la arquitectura de datos del ecosistema EduGo:

- **10 eliminaciones** de tablas/colecciones sin uso (bajo riesgo, alta prioridad)
- **2 homologaciones** de duplicaciones (riesgo medio, alta prioridad)
- **2 limpiezas** de campos sin uso (bajo riesgo, media prioridad)
- **4 adiciones** de funcionalidades faltantes (riesgo variable, baja prioridad)

**Recomendación principal:** Ejecutar Fase 1 (Limpieza) y Fase 2 (Homologación) en los próximos 2 sprints. Fase 3 (Nuevas Funcionalidades) puede planificarse para sprints posteriores según prioridades de negocio.

**Tiempo total estimado:**
- **Crítico (Fase 1 + 2):** 26-36 horas
- **Opcional (Fase 3):** 53-70 horas
- **Total:** 79-106 horas

**Impacto esperado:**
- ✅ Reducción de deuda técnica
- ✅ Arquitectura más clara y mantenible
- ✅ Menor confusión para nuevos desarrolladores
- ✅ Base sólida para futuras funcionalidades

---

**Generado:** 23 de Diciembre, 2025
**Próxima revisión:** Después de ejecutar Fase 1 y 2
