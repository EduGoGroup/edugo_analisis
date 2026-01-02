# Cobertura de Tablas PostgreSQL en Endpoints de APIs

**Fecha:** 2025-12-24
**Autor:** Sistema de Análisis EduGo
**Objetivo:** Verificar cobertura de tablas PostgreSQL en endpoints de las APIs Admin y Mobile

---

## Resumen Ejecutivo

### Métricas Generales

- **Total de tablas en PostgreSQL:** 20
- **Tablas con endpoints (API Admin):** 8
- **Tablas con endpoints (API Mobile):** 8
- **Tablas huérfanas (sin endpoints):** 4
- **Gaps críticos detectados:** 1

### Estado de Cobertura

```
Cobertura API Admin:    40% (8/20 tablas)
Cobertura API Mobile:   40% (8/20 tablas)
Cobertura Combinada:    60% (12/20 tablas)
Tablas sin Uso:         20% (4/20 tablas)
```

---

## Detalle de Tablas y Endpoints

### 1. Tablas con Cobertura Completa (Ambas APIs)

| Tabla | API Admin | API Mobile | Operaciones CRUD |
|-------|-----------|------------|------------------|
| **users** | ✅ | ✅ | READ (ambas APIs) |
| **materials** | ✅ | ✅ | CRUD (Admin), CRUD (Mobile) |

### 2. Tablas con Cobertura Parcial (Una API)

#### 2.1 Solo en API Admin

| Tabla | Endpoints API Admin | Operaciones CRUD |
|-------|---------------------|------------------|
| **schools** | `/api/v1/schools/*` | CREATE, READ, UPDATE, DELETE |
| **academic_units** | `/api/v1/schools/:schoolId/units/*`, `/api/v1/units/*` | CREATE, READ, UPDATE, DELETE, RESTORE |
| **memberships** | Usado en repositorio | READ (via repository) |
| **subjects** | `/api/v1/subjects/*` (inferido) | READ |
| **units** | `/api/v1/units/*` | CRUD |
| **guardian_relations** | Usado en repositorio | READ (via repository) |

**Handlers detectados en API Admin:**
- `school_handler.go` → tabla `schools`
- `academic_unit_handler.go` → tabla `academic_units`
- `unit_handler.go` → tabla `units`
- `subject_handler.go` → tabla `subjects`
- `guardian_handler.go` → tabla `guardian_relations`
- `material_handler.go` → tabla `materials`
- `user_handler.go` → tabla `users`
- `unit_membership_handler.go` → tabla `memberships`
- `stats_handler.go` → múltiples tablas (estadísticas)

**Rutas registradas en router.go:**
```go
// Schools
POST   /api/v1/schools
GET    /api/v1/schools
GET    /api/v1/schools/:id
GET    /api/v1/schools/code/:code
PUT    /api/v1/schools/:id
DELETE /api/v1/schools/:id

// Academic Units (scoped to school)
POST   /api/v1/schools/:schoolId/units
GET    /api/v1/schools/:schoolId/units
GET    /api/v1/schools/:schoolId/units/tree (usa ltree)
GET    /api/v1/schools/:schoolId/units/by-type

// Academic Units (global)
GET    /api/v1/units/:id
PUT    /api/v1/units/:id
DELETE /api/v1/units/:id
POST   /api/v1/units/:id/restore
GET    /api/v1/units/:id/hierarchy-path (usa ltree)
```

#### 2.2 Solo en API Mobile

| Tabla | Endpoints API Mobile | Operaciones CRUD |
|-------|---------------------|------------------|
| **assessment** | `/v1/materials/:id/assessment` | READ, DELETE |
| **assessment_attempt** | `/v1/materials/:id/assessment/attempts`, `/v1/attempts/:id/results`, `/v1/users/me/attempts` | CREATE, READ |
| **assessment_attempt_answer** | Usado en repositorio | CREATE, READ |
| **refresh_tokens** | Usado en auth (migration Sprint 3) | CREATE, READ, DELETE |
| **login_attempts** | Usado en auth middleware | CREATE, READ |
| **material_versions** | `/v1/materials/:id/versions` | READ |

**Handlers detectados en API Mobile:**
- `material_handler.go` → tablas `materials`, `material_versions`
- `assessment_handler.go` → tablas `assessment`, `assessment_attempt`, `assessment_attempt_answer`
- `progress_handler.go` → tabla `progress` (⚠️ ver gap crítico)
- `stats_handler.go` → múltiples tablas

**Rutas registradas en router.go:**
```go
// Materials (Protected)
GET    /v1/materials
GET    /v1/materials/:id
GET    /v1/materials/:id/versions
GET    /v1/materials/:id/download-url
GET    /v1/materials/:id/summary
GET    /v1/materials/:id/assessment
GET    /v1/materials/:id/stats
POST   /v1/materials (teacher+)
POST   /v1/materials/:id/upload-complete (teacher+)
POST   /v1/materials/:id/upload-url (teacher+)
PUT    /v1/materials/:id (teacher+)

// Assessments
POST   /v1/materials/:id/assessment/attempts
GET    /v1/attempts/:id/results
GET    /v1/users/me/attempts

// Progress
PUT    /v1/progress (idempotent UPSERT)

// Stats (admin only)
GET    /v1/stats/global
```

### 3. Tablas Huérfanas (Sin Endpoints Directos)

Estas tablas existen en la base de datos pero no tienen endpoints públicos en ninguna API:

| Tabla | Propósito | Razón de Ausencia | Prioridad |
|-------|-----------|-------------------|-----------|
| **user_active_context** | Almacena contexto activo del usuario (school_id actual) | Manejo interno de sesión | BAJA |
| **user_favorites** | Favoritos de usuarios | Feature no implementada aún | MEDIA |
| **user_activity_log** | Log de actividad de usuarios | Auditoría/Analytics | BAJA |
| **feature_flags** | Feature flags del sistema | Configuración interna | BAJA |
| **feature_flag_overrides** | Overrides de feature flags por usuario/org | Configuración interna | BAJA |

**Análisis:**
- `user_active_context`: Usado internamente por middleware de auth
- `user_favorites`: Feature pendiente de implementar en frontend
- `user_activity_log`: Para analytics y auditoría (no requiere endpoints CRUD)
- `feature_flags` y `feature_flag_overrides`: Configuración interna, no expuesta por seguridad

**Recomendación:**
- ✅ **user_active_context**: OK sin endpoints (uso interno)
- ⚠️ **user_favorites**: Implementar endpoints si se requiere feature de favoritos
- ✅ **user_activity_log**: OK sin endpoints (solo inserción interna)
- ✅ **feature_flags**: OK sin endpoints (configuración)

---

## Gaps Críticos Detectados

### Gap #1: Mismatch de Nombre de Tabla `progress` vs `material_progress`

**Severidad:** 🔴 CRÍTICO

**Descripción:**
- La migración `016_create_progress.up.sql` crea tabla llamada `progress`
- El código de API Mobile usa el nombre `material_progress` en las queries

**Evidencia:**

**En Migración:**
```sql
-- /edugo-infrastructure/postgres/migrations/016_create_progress.up.sql
CREATE TABLE IF NOT EXISTS progress (
    material_id UUID NOT NULL REFERENCES materials(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    ...
)
```

**En Código API Mobile:**
```go
// /edugo-api-mobile/.../progress_repository_impl.go
INSERT INTO material_progress (material_id, user_id, percentage, ...)
FROM material_progress
UPDATE material_progress
```

**Impacto:**
- ❌ El endpoint `PUT /v1/progress` fallará en runtime
- ❌ Las queries a `material_progress` resultarán en "relation does not exist"
- ❌ Feature de progreso de materiales completamente rota

**Acción Requerida:**
1. **OPCIÓN A (Recomendada):** Actualizar código para usar tabla `progress`
   - Cambiar todas las referencias de `material_progress` a `progress` en `progress_repository_impl.go`
   - Ejecutar tests de integración

2. **OPCIÓN B:** Crear migración para renombrar tabla
   ```sql
   ALTER TABLE progress RENAME TO material_progress;
   ```

**Prioridad:** ALTA - Bloquea feature de progreso

---

## Tablas Definidas en Migraciones

### Listado Completo de 20 Tablas

Basado en análisis de `/edugo-infrastructure/postgres/migrations/`:

1. ✅ **users** (001_create_users.up.sql)
2. ✅ **schools** (002_create_schools.up.sql)
3. ✅ **academic_units** (003_create_academic_units.up.sql)
4. ✅ **memberships** (004_create_memberships.up.sql)
5. ✅ **materials** (005_create_materials.up.sql)
6. ✅ **assessment** (006_create_assessments.up.sql)
7. ✅ **assessment_attempt** (007_create_assessment_attempts.up.sql)
8. ✅ **assessment_attempt_answer** (008_create_assessment_answers.up.sql)
9. ❌ **refresh_tokens** (constraints/009_create_refresh_tokens.sql)
10. ❌ **login_attempts** (constraints/010_create_login_attempts.sql)
11. ❌ **user_active_context** (constraints/011_create_user_active_context.sql)
12. ❌ **user_favorites** (constraints/012_create_user_favorites.sql)
13. ❌ **user_activity_log** (constraints/013_create_user_activity_log.sql)
14. ❌ **feature_flags** (constraints/014_create_feature_flags.sql)
15. ❌ **feature_flag_overrides** (constraints/015_create_feature_flag_overrides.sql)
16. ✅ **material_versions** (012_create_material_versions.up.sql)
17. ✅ **subjects** (013_create_subjects.up.sql)
18. ✅ **units** (014_create_units.up.sql)
19. ✅ **guardian_relations** (015_create_guardian_relations.up.sql)
20. ⚠️ **progress** (016_create_progress.up.sql) - **GAP CRÍTICO**

**Leyenda:**
- ✅ Tiene endpoints en al menos una API
- ❌ Huérfana (sin endpoints)
- ⚠️ Problema detectado

---

## Análisis de Repositorios

### API Admin - Repositorios Implementados

Ubicación: `/edugo-api-administracion/internal/infrastructure/persistence/postgres/repository/`

```
academic_unit_repository_impl.go  → academic_units
guardian_repository_impl.go        → guardian_relations
material_repository_impl.go        → materials
school_repository_impl.go          → schools
stats_repository_impl.go           → múltiples (stats agregadas)
subject_repository_impl.go         → subjects
unit_membership_repository_impl.go → memberships
unit_repository_impl.go            → units
user_repository_impl.go            → users
```

**Queries detectadas:**
- `FROM academic_units` (múltiples variantes con WHERE, JOIN)
- `FROM schools`
- `FROM subjects`
- `FROM units`
- `FROM users`
- `FROM memberships`
- `FROM guardian_relations`
- `FROM materials`

### API Mobile - Repositorios Implementados

Ubicación: `/edugo-api-mobile/internal/infrastructure/persistence/postgres/repository/`

```
assessment_repository.go           → assessment
attempt_repository.go              → assessment_attempt
answer_repository.go               → assessment_attempt_answer
material_repository_impl.go        → materials, material_versions
progress_repository_impl.go        → material_progress (⚠️ GAP)
refresh_token_repository_impl.go   → refresh_tokens
login_attempt_repository_impl.go   → login_attempts
user_repository_impl.go            → users
```

**Queries detectadas:**
- `FROM assessment`
- `FROM assessment_attempt`
- `FROM assessment_attempt_answer`
- `FROM materials`
- `FROM material_versions`
- `FROM material_progress` (⚠️ debería ser `progress`)
- `FROM refresh_tokens`
- `FROM login_attempts`
- `FROM users`

---

## Recomendaciones por Prioridad

### 🔴 Prioridad CRÍTICA (Bloquea funcionalidad)

1. **Corregir mismatch tabla `progress` / `material_progress`**
   - Acción: Actualizar queries en `progress_repository_impl.go`
   - Archivo: `/edugo-api-mobile/.../progress_repository_impl.go`
   - Cambio: `material_progress` → `progress` en todas las queries
   - Impacto: Desbloquea feature de progreso de materiales
   - Esfuerzo: 30 minutos
   - Testing: Ejecutar tests de integración de progress

### 🟡 Prioridad ALTA (Mejora cobertura)

2. **Implementar endpoints para `user_favorites`**
   - Si se requiere feature de favoritos en frontend
   - Endpoints sugeridos:
     ```
     POST   /v1/users/me/favorites
     GET    /v1/users/me/favorites
     DELETE /v1/users/me/favorites/:materialId
     ```
   - API sugerida: Mobile (feature de usuario final)
   - Esfuerzo: 1-2 días

3. **Documentar uso de `user_active_context`**
   - Validar que se use correctamente en middleware de auth
   - Crear tests de integración
   - Esfuerzo: 4 horas

### 🟢 Prioridad MEDIA (Mejoras)

4. **Verificar coherencia de nomenclatura**
   - Revisar si hay más casos de mismatch de nombres
   - Estandarizar convención: singular vs plural
   - Esfuerzo: 2 horas

5. **Agregar endpoints de auditoría para `user_activity_log`**
   - Solo si se requiere dashboard de auditoría
   - API sugerida: Admin
   - Endpoints: `GET /api/v1/audit/users/:id/activity`
   - Esfuerzo: 1 día

### 🔵 Prioridad BAJA (Opcional)

6. **Feature Flags Admin UI**
   - Endpoints CRUD para `feature_flags` y `feature_flag_overrides`
   - Solo para admin/super_admin
   - Esfuerzo: 2-3 días

---

## Apéndice: Estructura de Directorios Analizados

### Migraciones PostgreSQL
```
/edugo-infrastructure/postgres/migrations/
├── 001_create_users.up.sql
├── 002_create_schools.up.sql
├── 003_create_academic_units.up.sql
├── 004_create_memberships.up.sql
├── 005_create_materials.up.sql
├── 006_create_assessments.up.sql
├── 007_create_assessment_attempts.up.sql
├── 008_create_assessment_answers.up.sql
├── 009_extend_assessment_schema.up.sql
├── 010_extend_assessment_attempt.up.sql
├── 011_extend_assessment_answer.up.sql
├── 012_create_material_versions.up.sql
├── 013_create_subjects.up.sql
├── 014_create_units.up.sql
├── 015_create_guardian_relations.up.sql
├── 016_create_progress.up.sql (⚠️)
├── 017_add_school_id_to_users.up.sql
└── constraints/
    ├── 001_create_users.sql
    ├── 002_create_schools.sql
    ├── ...
    ├── 009_create_refresh_tokens.sql
    ├── 010_create_login_attempts.sql
    ├── 011_create_user_active_context.sql
    ├── 012_create_user_favorites.sql
    ├── 013_create_user_activity_log.sql
    ├── 014_create_feature_flags.sql
    └── 015_create_feature_flag_overrides.sql
```

### API Admin (Go)
```
/edugo-api-administracion/
└── internal/
    ├── infrastructure/
    │   ├── http/
    │   │   ├── handler/
    │   │   │   ├── academic_unit_handler.go
    │   │   │   ├── guardian_handler.go
    │   │   │   ├── material_handler.go
    │   │   │   ├── school_handler.go
    │   │   │   ├── subject_handler.go
    │   │   │   ├── unit_handler.go
    │   │   │   ├── unit_membership_handler.go
    │   │   │   ├── user_handler.go
    │   │   │   └── stats_handler.go
    │   │   └── router/
    │   │       └── router.go
    │   └── persistence/
    │       └── postgres/
    │           └── repository/
    │               ├── academic_unit_repository_impl.go
    │               ├── guardian_repository_impl.go
    │               ├── material_repository_impl.go
    │               ├── school_repository_impl.go
    │               ├── stats_repository_impl.go
    │               ├── subject_repository_impl.go
    │               ├── unit_membership_repository_impl.go
    │               ├── unit_repository_impl.go
    │               └── user_repository_impl.go
    └── auth/
        └── repository/
            ├── token_repository.go (interfaces comentadas)
            └── user_repository.go
```

### API Mobile (Go)
```
/edugo-api-mobile/
└── internal/
    ├── infrastructure/
    │   ├── http/
    │   │   ├── handler/
    │   │   │   ├── assessment_handler.go
    │   │   │   ├── material_handler.go
    │   │   │   ├── progress_handler.go
    │   │   │   ├── stats_handler.go
    │   │   │   └── summary_handler.go
    │   │   └── router/
    │   │       └── router.go
    │   └── persistence/
    │       └── postgres/
    │           └── repository/
    │               ├── assessment_repository.go
    │               ├── attempt_repository.go
    │               ├── answer_repository.go
    │               ├── material_repository_impl.go
    │               ├── progress_repository_impl.go (⚠️)
    │               ├── refresh_token_repository_impl.go
    │               ├── login_attempt_repository_impl.go
    │               └── user_repository_impl.go
    └── domain/
        └── repository/ (interfaces)
```

---

## Conclusiones

### Hallazgos Principales

1. ✅ **Cobertura Razonable**: 60% de las tablas tienen endpoints en al menos una API
2. ⚠️ **Gap Crítico**: Mismatch de nombre `progress` vs `material_progress` bloquea funcionalidad
3. ✅ **Tablas Huérfanas Justificadas**: Las 4 tablas sin endpoints tienen razones válidas
4. ✅ **Separación de Concerns**: API Admin maneja gestión, API Mobile maneja consumo estudiantil
5. 📊 **Oportunidades**: Feature de favoritos puede agregar valor al frontend

### Estado de Preparación para Frontend

**Para Dashboard Admin:**
- ✅ Schools: Completo (CRUD)
- ✅ Academic Units: Completo (CRUD + ltree)
- ✅ Materials: Completo (CRUD)
- ✅ Users: Completo (READ, gestión via memberships)
- ✅ Subjects: Completo (READ)
- ✅ Guardian Relations: Completo (READ)
- ⚠️ Stats: Disponible pero revisar performance

**Para App Mobile:**
- ✅ Materials: Completo (CRUD + versiones)
- ✅ Assessments: Completo (intentos + resultados)
- 🔴 Progress: **BLOQUEADO** (requiere fix de nombre de tabla)
- ✅ Stats: Completo (admin only)
- ⚠️ Favorites: No implementado

### Próximos Pasos

1. **INMEDIATO**: Corregir gap de `material_progress` → `progress`
2. **CORTO PLAZO**: Implementar endpoints de favoritos si se requiere en frontend
3. **MEDIANO PLAZO**: Agregar endpoints de auditoría si se requiere dashboard
4. **LARGO PLAZO**: Considerar admin UI para feature flags

---

**Generado:** 2025-12-24
**Herramienta:** Claude Code Agent
**Repositorios Analizados:**
- edugo-infrastructure (migraciones)
- edugo-api-administracion (handlers + repositories)
- edugo-api-mobile (handlers + repositories)
