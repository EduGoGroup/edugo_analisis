# PLAN DE ELIMINACIÓN DEFINITIVO - Tablas y Colecciones Sin Uso

**Fecha de Planificación:** 2026-01-01  
**Fecha de Ejecución:** 2025-12-23  
**Estado:** ✅ EJECUTADO Y COMPLETADO  
**Validación:** ✅ Completada (ver VALIDACION_TABLAS_SIN_USO.md)

---

## RESUMEN EJECUTIVO

**Total eliminado:** 11 estructuras (5 PostgreSQL + 6 MongoDB)  
**Proyectos afectados:** 1 (edugo-infrastructure)  
**Impacto en APIs/Worker:** ✅ NINGUNO (0 referencias validadas)  
**Riesgo:** BAJO  
**Tiempo real de ejecución:** ~2 horas  
**Commit:** e576963  
**PR:** #50 → #51 (mergeado a main)  
**Resultado:** ✅ EXITOSO - 28 archivos modificados, 1,629 líneas eliminadas

---

## PARTE 1: POSTGRESQL - 5 TABLAS A ELIMINAR

### Tablas a Eliminar

| # | Tabla | Descripción | Creada en Migración | Motivo Eliminación |
|---|-------|-------------|---------------------|-------------------|
| 1 | `user_active_context` | Contexto activo del usuario | 011_create_user_active_context.sql | Feature UI no implementada |
| 2 | `user_favorites` | Favoritos de usuarios | 012_create_user_favorites.sql | Feature UI no implementada |
| 3 | `user_activity_log` | Log de actividad de usuarios | 013_create_user_activity_log.sql | Feature UI no implementada |
| 4 | `feature_flags` | Sistema de feature flags | 014_create_feature_flags.sql | Deuda técnica Apple App |
| 5 | `feature_flag_overrides` | Overrides de feature flags | 015_create_feature_flag_overrides.sql | Dependiente de feature_flags |

### Objetos Relacionados a Eliminar

**ENUM a eliminar:**
- `activity_type` (usado solo por `user_activity_log`)

**Índices (eliminados automáticamente con DROP TABLE CASCADE):**
- `idx_user_active_context_user`
- `idx_user_active_context_school`
- `idx_feature_flags_key`
- `idx_feature_flags_enabled`
- Y otros...

**Triggers (eliminados automáticamente):**
- `set_updated_at_user_active_context`
- `set_updated_at_feature_flags`

### Proyecto Afectado: `edugo-infrastructure`

**Archivos a MODIFICAR:**

1. **Crear nueva migración de eliminación:**
   ```
   📁 edugo-infrastructure/postgres/migrations/structure/
   ├── ✅ 016_drop_unused_tables.sql (CREAR)
   ```

2. **Eliminar archivos de migración antiguos:**
   ```
   📁 edugo-infrastructure/postgres/migrations/
   ├── structure/
   │   ├── ❌ 011_create_user_active_context.sql (ELIMINAR)
   │   ├── ❌ 012_create_user_favorites.sql (ELIMINAR)
   │   ├── ❌ 013_create_user_activity_log.sql (ELIMINAR)
   │   ├── ❌ 014_create_feature_flags.sql (ELIMINAR)
   │   └── ❌ 015_create_feature_flag_overrides.sql (ELIMINAR)
   ├── constraints/
   │   ├── ❌ 011_create_user_active_context.sql (ELIMINAR)
   │   ├── ❌ 012_create_user_favorites.sql (ELIMINAR)
   │   ├── ❌ 013_create_user_activity_log.sql (ELIMINAR)
   │   ├── ❌ 014_create_feature_flags.sql (ELIMINAR)
   │   └── ❌ 015_create_feature_flag_overrides.sql (ELIMINAR)
   └── testing/
       ├── ❌ 006_demo_user_active_context.sql (ELIMINAR)
       ├── ❌ 007_demo_user_favorites.sql (ELIMINAR)
       ├── ❌ 008_demo_user_activity_log.sql (ELIMINAR)
       ├── ❌ 009_demo_feature_flags.sql (ELIMINAR)
       └── ❌ 010_demo_feature_flag_overrides.sql (ELIMINAR)
   ```

**Total archivos a eliminar:** 15 archivos

### Script de Migración PostgreSQL

**Archivo:** `edugo-infrastructure/postgres/migrations/structure/016_drop_unused_tables.sql`

```sql
-- Migración 016: Eliminar tablas sin uso
-- Fecha: 2026-01-01
-- Descripción: Elimina tablas creadas para features UI no implementadas
-- Validación: VALIDACION_TABLAS_SIN_USO.md

-- Eliminar tablas en orden de dependencias (hijas primero)
DROP TABLE IF EXISTS feature_flag_overrides CASCADE;
DROP TABLE IF EXISTS feature_flags CASCADE;
DROP TABLE IF EXISTS user_activity_log CASCADE;
DROP TABLE IF EXISTS user_favorites CASCADE;
DROP TABLE IF EXISTS user_active_context CASCADE;

-- Eliminar ENUM asociado
DROP TYPE IF EXISTS activity_type;

-- Comentario de auditoría
COMMENT ON SCHEMA public IS 'Schema limpio después de eliminación de tablas sin uso (feature_flags, user_active_context, user_favorites, user_activity_log)';
```

**Archivo de rollback:** `edugo-infrastructure/postgres/migrations/structure/016_drop_unused_tables.down.sql`

```sql
-- Rollback de migración 016
-- ADVERTENCIA: Este rollback recrea las tablas VACÍAS
-- No recupera datos eliminados

-- Recrear desde archivos originales (si es necesario)
-- Ver archivos: 011_*.sql, 012_*.sql, 013_*.sql, 014_*.sql, 015_*.sql
```

### Proyectos NO Afectados

| Proyecto | Referencias Encontradas | Impacto |
|----------|------------------------|---------|
| `edugo-api-administracion` | ❌ 0 archivos | ✅ NINGUNO |
| `edugo-api-mobile` | ❌ 0 archivos | ✅ NINGUNO |
| `edugo-worker` | ❌ 0 archivos | ✅ NINGUNO |
| `edugo-shared` | ❌ 0 archivos | ✅ NINGUNO |
| `edugo-dev-environment` | ⚠️ Seeds de testing | ⚠️ Actualizar seeds |

---

## PARTE 2: MONGODB - 6 COLECCIONES ELIMINADAS ✅

### Colecciones Eliminadas

| # | Colección | Descripción | Migración Original | Motivo Eliminación | Estado |
|---|-----------|-------------|-------------------|-------------------|--------|
| 1 | `material_assessment` | Evaluaciones (duplicada) | 001_material_assessment.go | Duplicado de material_assessment_worker | ✅ ELIMINADA |
| 2 | `material_content` | Contenido extraído | 002_material_content.go | Worker nunca implementó uso | ✅ ELIMINADA |
| 3 | `assessment_attempt_result` | Resultados de attempts | 003_assessment_attempt_result.go | Datos en PostgreSQL | ✅ ELIMINADA |
| 4 | `audit_logs` | Trail de auditoría | 004_audit_logs.go | Sin implementación | ✅ ELIMINADA |
| 5 | `notifications` | Notificaciones | 005_notifications.go | Sin implementación | ✅ ELIMINADA |
| 6 | `analytics_events` | Eventos de analytics | 006_analytics_events.go | Sin implementación | ✅ ELIMINADA |

### Proyecto Afectado: `edugo-infrastructure`

**Archivos a MODIFICAR:**

1. **Eliminar archivos de migración:**
   ```
   📁 edugo-infrastructure/mongodb/migrations/
   ├── structure/
   │   ├── ❌ 002_material_content.go (ELIMINAR)
   │   ├── ❌ 003_assessment_attempt_result.go (ELIMINAR)
   │   ├── ❌ 004_audit_logs.go (ELIMINAR)
   │   ├── ❌ 005_notifications.go (ELIMINAR)
   │   └── ❌ 006_analytics_events.go (ELIMINAR)
   └── constraints/
       ├── ❌ 002_material_content_indexes.go (ELIMINAR)
       ├── ❌ 003_assessment_attempt_result_indexes.go (ELIMINAR)
       ├── ❌ 004_audit_logs_indexes.go (ELIMINAR)
       ├── ❌ 005_notifications_indexes.go (ELIMINAR)
       └── ❌ 006_analytics_events_indexes.go (ELIMINAR)
   ```

2. **Eliminar seeds:**
   ```
   📁 edugo-infrastructure/
   ├── mongodb/seeds/
   │   ├── ❌ material_content.js (ELIMINAR)
   │   ├── ❌ assessment_attempt_result.js (ELIMINAR)
   │   ├── ❌ audit_logs.js (ELIMINAR)
   │   ├── ❌ notifications.js (ELIMINAR)
   │   └── ❌ analytics_events.js (ELIMINAR)
   └── seeds/mongodb/
       ├── ❌ material_content.js (ELIMINAR - si existe)
       ├── ❌ assessment_attempt_result.js (ELIMINAR - si existe)
       ├── ❌ audit_logs.js (ELIMINAR - si existe)
       ├── ❌ notifications.js (ELIMINAR - si existe)
       └── ❌ analytics_events.js (ELIMINAR - si existe)
   ```

3. **Actualizar runner de migraciones:**
   ```
   📁 edugo-infrastructure/mongodb/migrations/cmd/
   └── 📝 runner.go (MODIFICAR - eliminar llamadas a Create*)
   ```

**Total archivos a eliminar:** ~15 archivos

### Script de Eliminación MongoDB

**Archivo:** `edugo-infrastructure/mongodb/scripts/drop_unused_collections.js`

```javascript
// Script de eliminación de colecciones sin uso
// Fecha: 2026-01-01
// Validación: VALIDACION_TABLAS_SIN_USO.md

// Conectar a la base de datos correcta
use edugo_db; // O el nombre correcto de tu BD

// Verificar colecciones antes de eliminar
print("=== Colecciones a eliminar ===");
print("analytics_events: " + db.analytics_events.count() + " documentos");
print("notifications: " + db.notifications.count() + " documentos");
print("audit_logs: " + db.audit_logs.count() + " documentos");
print("assessment_attempt_result: " + db.assessment_attempt_result.count() + " documentos");
print("material_content: " + db.material_content.count() + " documentos");

// Solicitar confirmación
print("\n¿Continuar con eliminación? (ejecutar manualmente cada drop)");

// Eliminar colecciones
db.analytics_events.drop();
print("✓ analytics_events eliminada");

db.notifications.drop();
print("✓ notifications eliminada");

db.audit_logs.drop();
print("✓ audit_logs eliminada");

db.assessment_attempt_result.drop();
print("✓ assessment_attempt_result eliminada");

db.material_content.drop();
print("✓ material_content eliminada");

print("\n=== Eliminación completada ===");

// Verificar colecciones restantes
print("\nColecciones restantes:");
db.getCollectionNames().forEach(function(name) {
    print("- " + name);
});
```

### Actualizar Runner de Migraciones

**Archivo:** `edugo-infrastructure/mongodb/migrations/cmd/runner.go`

**ANTES:**
```go
// Ejecutar migraciones de estructura
structure.CreateMaterialAssessment(ctx, db),           // 001
structure.CreateMaterialContent(ctx, db),              // 002 ← ELIMINAR
structure.CreateAssessmentAttemptResult(ctx, db),      // 003 ← ELIMINAR
structure.CreateAuditLogs(ctx, db),                    // 004 ← ELIMINAR
structure.CreateNotifications(ctx, db),                // 005 ← ELIMINAR
structure.CreateAnalyticsEvents(ctx, db),              // 006 ← ELIMINAR
structure.CreateMaterialSummary(ctx, db),              // 007
structure.CreateMaterialAssessmentWorker(ctx, db),     // 008
```

**DESPUÉS:**
```go
// Ejecutar migraciones de estructura
structure.CreateMaterialAssessment(ctx, db),           // 001
// 002-006: Eliminadas (colecciones sin uso)
structure.CreateMaterialSummary(ctx, db),              // 007
structure.CreateMaterialAssessmentWorker(ctx, db),     // 008
```

### Proyectos NO Afectados

| Proyecto | Referencias Encontradas | Impacto |
|----------|------------------------|---------|
| `edugo-api-administracion` | ❌ 0 archivos | ✅ NINGUNO |
| `edugo-api-mobile` | ❌ 0 archivos | ✅ NINGUNO |
| `edugo-worker` | ❌ 0 archivos | ✅ NINGUNO |
| `edugo-shared` | ❌ 0 archivos | ✅ NINGUNO |

---

## PARTE 3: PROYECTOS AFECTADOS - RESUMEN

### Único Proyecto Afectado: `edugo-infrastructure`

**Rama de trabajo:** `feature/PT-000-eliminar-tablas-sin-uso`

**Cambios requeridos:**

```
edugo-infrastructure/
├── postgres/
│   └── migrations/
│       ├── structure/
│       │   ├── ✅ 016_drop_unused_tables.sql (CREAR)
│       │   ├── ❌ 011_create_user_active_context.sql (ELIMINAR)
│       │   ├── ❌ 012_create_user_favorites.sql (ELIMINAR)
│       │   ├── ❌ 013_create_user_activity_log.sql (ELIMINAR)
│       │   ├── ❌ 014_create_feature_flags.sql (ELIMINAR)
│       │   └── ❌ 015_create_feature_flag_overrides.sql (ELIMINAR)
│       ├── constraints/ (eliminar 5 archivos)
│       └── testing/ (eliminar 5 archivos)
└── mongodb/
    ├── migrations/
    │   ├── structure/ (eliminar 5 archivos .go)
    │   ├── constraints/ (eliminar 5 archivos .go)
    │   └── cmd/
    │       └── 📝 runner.go (MODIFICAR)
    ├── seeds/ (eliminar 5 archivos .js)
    └── scripts/
        └── ✅ drop_unused_collections.js (CREAR)
```

**Total de archivos:**
- **A crear:** 2 archivos
- **A modificar:** 1 archivo
- **A eliminar:** ~30 archivos

### Proyectos NO Afectados (Validado con Grep)

```
✅ edugo-api-administracion
   - 0 referencias a tablas PostgreSQL
   - 0 referencias a colecciones MongoDB
   - NO requiere cambios
   - NO requiere actualización de go.mod

✅ edugo-api-mobile
   - 0 referencias a tablas PostgreSQL
   - 0 referencias a colecciones MongoDB
   - NO requiere cambios
   - NO requiere actualización de go.mod

✅ edugo-worker
   - 0 referencias a tablas PostgreSQL
   - 0 referencias a colecciones MongoDB
   - NO requiere cambios
   - NO requiere actualización de go.mod

✅ edugo-shared
   - NO contiene lógica de BD
   - NO requiere cambios

⚠️ edugo-dev-environment
   - Contiene seeds de testing
   - Requiere eliminar seeds de tablas borradas
   - Impacto: BAJO (solo datos de testing)
```

---

## PARTE 4: EJECUCIÓN REALIZADA - RESUMEN ✅

### Estado de Ejecución

**Fecha:** 2025-12-23  
**Rama:** `feature/PT-000-release-consolidado-limpieza`  
**Commit:** e576963  
**PR:** #50 → dev, luego #51 → main  
**Estado:** ✅ MERGEADO A MAIN

### Cambios Aplicados (Commit e576963)

```
feat: release consolidado limpieza infraestructura

PostgreSQL:
- Eliminar tablas sin uso: user_active_context, user_favorites,
  user_activity_log, feature_flags, feature_flag_overrides
- Eliminar campo email_verified de users
- Eliminar seeds de testing correspondientes

MongoDB:
- Eliminar colecciones sin uso: material_content, assessment_attempt_result,
  audit_logs, notifications, analytics_events
- Eliminar material_assessment duplicado (usar material_assessment_worker)
- Actualizar embed.go, seeds.go, mock_data.go, runner.go

BREAKING CHANGE: Tablas y colecciones eliminadas permanentemente.
Requiere drop + recreate de BD en desarrollo.
```

### Archivos Modificados (28 archivos totales)

**PostgreSQL:**
- ✅ `postgres/entities/user.go` - Removido campo email_verified
- ❌ `postgres/migrations/structure/001_create_users.sql` - Actualizado
- ❌ 5 archivos structure/*.sql eliminados (011-015)
- ❌ 5 archivos testing/*.sql eliminados (006-010)

**MongoDB:**
- ✅ `mongodb/migrations/cmd/runner.go` - Actualizado (78 líneas modificadas)
- ❌ 6 archivos structure/*.go eliminados (001-006)
- ❌ 6 archivos constraints/*_indexes.go eliminados
- ✅ `mongodb/migrations/embed.go` - Actualizado (24 líneas eliminadas)
- ✅ `mongodb/migrations/mock_data.go` - Actualizado
- ✅ `mongodb/migrations/seeds.go` - Actualizado

**Estadísticas:**
- 28 archivos modificados
- 40 inserciones (+)
- 1,629 deleciones (-)

### Validaciones Completadas ✅

1. ✅ Build exitoso
2. ✅ Sin conflictos de merge
3. ✅ PR aprobado y mergeado
4. ✅ Cambios reflejados en main y dev
5. ✅ Sin regresiones reportadas

---

## PARTE 5: CHECKLIST DE VALIDACIÓN - ✅ COMPLETADO

### Pre-Ejecución ✅

- ✅ Validación de referencias (0 encontradas en APIs/Worker)
- ✅ PR revisado y aprobado
- ✅ Tests pasando en CI/CD
- ✅ Linter sin errores
- ✅ Documentación de validación creada

### Post-Ejecución ✅

- ✅ Tablas PostgreSQL eliminadas
- ✅ Colecciones MongoDB eliminadas
- ✅ Archivos de migración eliminados
- ✅ Runner.go actualizado
- ✅ Build exitoso
- ✅ PR mergeado a main (PR #51)
- ✅ Sin errores en merge
- ✅ Cambios sincronizados en dev y main

### Verificación en Repositorio ✅

**Ejecutado el 2026-01-01:**
```bash
cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure
git checkout main
git pull origin main

# Resultado: Fast-forward a commit 91e90a7
# 28 archivos modificados, 1,629 líneas eliminadas
# Sin errores de merge
```

**Estado actual:**
- ✅ Ramas dev y main sincronizadas
- ✅ Commit e576963 presente en ambas ramas
- ✅ Todos los archivos eliminados confirmados
- ✅ Sin archivos pendientes de eliminación

---

## ANEXO: COMANDOS DE VERIFICACIÓN

### Verificar Eliminación PostgreSQL

```sql
-- Conectar a BD
psql -h localhost -U postgres -d edugo_db

-- Verificar tablas eliminadas (debe retornar 0)
SELECT count(*) FROM information_schema.tables 
WHERE table_name IN (
    'user_active_context',
    'user_favorites',
    'user_activity_log',
    'feature_flags',
    'feature_flag_overrides'
);

-- Verificar ENUM eliminado (debe retornar 0)
SELECT count(*) FROM pg_type 
WHERE typname = 'activity_type';
```

### Verificar Eliminación MongoDB

```javascript
// Conectar a BD
mongo mongodb://localhost:27017/edugo_db

// Listar todas las colecciones
db.getCollectionNames()

// Verificar colecciones eliminadas (no deben aparecer)
db.getCollectionNames().filter(name => 
    name.match(/material_content|assessment_attempt_result|audit_logs|notifications|analytics_events/)
)
```

---

**FIN DEL PLAN DE ELIMINACIÓN**

*Documento generado: 2026-01-01*  
*Estado: LISTO PARA EJECUCIÓN*  
*Próximo paso: Ejecutar FASE 1 (Preparación)*
