# Plan Consolidado: edugo-infrastructure

**Proyecto:** edugo-infrastructure
**Rama base:** dev
**Rama feature:** `feature/PT-000-release-consolidado-limpieza`

**Releases:**
- `postgres/v0.15.0`
- `mongodb/v0.13.0`

---

## Fase 1: Preparacion

### Paso 1.1: Crear rama

```bash
cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure
git fetch origin
git checkout dev
git pull origin dev
git checkout -b feature/PT-000-release-consolidado-limpieza
```

### Paso 1.2: Verificar estado actual

```bash
# Ver migraciones PostgreSQL existentes
ls -la postgres/migrations/structure/

# Ver migraciones MongoDB existentes
ls -la mongodb/migrations/structure/
ls -la mongodb/migrations/constraints/

# Ultimo numero de migracion PostgreSQL
ls postgres/migrations/structure/ | tail -1

# Ultimo numero de migracion MongoDB
ls mongodb/migrations/structure/ | tail -1
```

---

## Fase 2: PostgreSQL - Eliminar Tablas Sin Uso (Tarea 01)

### Paso 2.1: Crear migracion de eliminacion de tablas

Crear archivo: `postgres/migrations/structure/016_drop_unused_tables.sql`

```sql
-- Migration: 016_drop_unused_tables.sql
-- Description: Eliminar tablas sin uso en el ecosistema
-- Date: 2025-12-XX
-- Tickets: PT-001

-- IMPORTANTE: Ejecutar backup antes de esta migracion en produccion
-- pg_dump -h localhost -U edugo -d edugo_db > backup_before_016.sql

-- Eliminar en orden inverso de dependencias

-- 1. Eliminar feature_flag_overrides (depende de feature_flags)
DROP TABLE IF EXISTS feature_flag_overrides CASCADE;

-- 2. Eliminar feature_flags
DROP TABLE IF EXISTS feature_flags CASCADE;

-- 3. Eliminar user_activity_log y su tipo ENUM
DROP TABLE IF EXISTS user_activity_log CASCADE;
DROP TYPE IF EXISTS activity_type CASCADE;

-- 4. Eliminar user_favorites
DROP TABLE IF EXISTS user_favorites CASCADE;

-- 5. Eliminar user_active_context
DROP TABLE IF EXISTS user_active_context CASCADE;

COMMENT ON SCHEMA public IS 'Migracion 016: Tablas eliminadas por no uso';
```

### Paso 2.2: Eliminar seeds de testing

```bash
rm -f postgres/migrations/testing/006_demo_user_active_context.sql
rm -f postgres/migrations/testing/007_demo_user_favorites.sql
rm -f postgres/migrations/testing/008_demo_user_activity_log.sql
rm -f postgres/migrations/testing/009_demo_feature_flags.sql
rm -f postgres/migrations/testing/010_demo_feature_flag_overrides.sql
```

### Paso 2.3: Commit parcial

```bash
git add postgres/
git commit -m "feat(postgres): eliminar tablas sin uso - PT-001

- Crear migracion 016_drop_unused_tables.sql
- Eliminar: user_active_context, user_favorites, user_activity_log
- Eliminar: feature_flags, feature_flag_overrides
- Eliminar seeds de testing correspondientes

BREAKING CHANGE: Tablas eliminadas permanentemente"
```

---

## Fase 3: PostgreSQL - Eliminar Campos Sin Uso (Tarea 05)

### Paso 3.1: Crear migracion de eliminacion de campos

Crear archivo: `postgres/migrations/structure/017_drop_unused_user_fields.sql`

```sql
-- Migration: 017_drop_unused_user_fields.sql
-- Description: Eliminar campos sin uso en tabla users
-- Date: 2025-12-XX
-- Tickets: PT-005

-- IMPORTANTE: Backup antes de ejecutar
-- pg_dump -h localhost -U edugo -d edugo_db -t users > backup_users_before_017.sql

-- 1. Eliminar campo email_verified (sin logica de verificacion)
ALTER TABLE users DROP COLUMN IF EXISTS email_verified;

-- 2. Eliminar campo last_login_at (sin registro de login)
ALTER TABLE users DROP COLUMN IF EXISTS last_login_at;

-- 3. Eliminar campo preferences (sin UI de preferencias)
ALTER TABLE users DROP COLUMN IF EXISTS preferences;

COMMENT ON TABLE users IS 'Migracion 017: Campos eliminados por no uso';
```

### Paso 3.2: Actualizar entity User

Modificar archivo: `postgres/entities/user.go`

**ELIMINAR los siguientes campos del struct User:**

```go
// ELIMINAR estas lineas:
EmailVerified  bool           `gorm:"column:email_verified;not null;default:false"`
LastLoginAt    *time.Time     `gorm:"column:last_login_at"`
Preferences    datatypes.JSON `gorm:"column:preferences;default:'{}'"`
```

### Paso 3.3: Actualizar seeds de testing

Revisar y modificar seeds que incluyan estos campos:

```bash
# Buscar referencias
grep -r "email_verified\|last_login_at\|preferences" postgres/migrations/testing/
```

Eliminar columnas de los INSERT statements encontrados.

### Paso 3.4: Commit parcial

```bash
git add postgres/
git commit -m "feat(postgres): eliminar campos sin uso de users - PT-005

- Crear migracion 017_drop_unused_user_fields.sql
- Eliminar: email_verified, last_login_at, preferences
- Actualizar entity User
- Actualizar seeds de testing

BREAKING CHANGE: Campos eliminados permanentemente"
```

---

## Fase 4: MongoDB - Eliminar Colecciones Sin Uso (Tarea 02)

### Paso 4.1: Eliminar migraciones de estructura

```bash
rm -f mongodb/migrations/structure/002_material_content.go
rm -f mongodb/migrations/structure/003_assessment_attempt_result.go
rm -f mongodb/migrations/structure/004_audit_logs.go
rm -f mongodb/migrations/structure/005_notifications.go
rm -f mongodb/migrations/structure/006_analytics_events.go
```

### Paso 4.2: Eliminar migraciones de constraints/indexes

```bash
rm -f mongodb/migrations/constraints/002_material_content_indexes.go
rm -f mongodb/migrations/constraints/003_assessment_attempt_result_indexes.go
rm -f mongodb/migrations/constraints/004_audit_logs_indexes.go
rm -f mongodb/migrations/constraints/005_notifications_indexes.go
rm -f mongodb/migrations/constraints/006_analytics_events_indexes.go
```

### Paso 4.3: Eliminar seeds

```bash
rm -f mongodb/seeds/material_content.js
rm -f mongodb/seeds/assessment_attempt_result.js
rm -f mongodb/seeds/audit_logs.js
rm -f mongodb/seeds/notifications.js
rm -f mongodb/seeds/analytics_events.js

rm -f seeds/mongodb/material_content.js
rm -f seeds/mongodb/assessment_attempt_result.js
rm -f seeds/mongodb/audit_logs.js
rm -f seeds/mongodb/notifications.js
rm -f seeds/mongodb/analytics_events.js
```

### Paso 4.4: Crear script de limpieza

Crear archivo: `mongodb/scripts/cleanup_unused_collections.js`

```javascript
// Script: cleanup_unused_collections.js
// Description: Eliminar colecciones sin uso del ecosistema
// Tickets: PT-002

print("=== Limpieza de colecciones sin uso ===");
print("Fecha: " + new Date().toISOString());

var collectionsToDelete = [
    "material_content",
    "assessment_attempt_result",
    "audit_logs",
    "notifications",
    "analytics_events"
];

collectionsToDelete.forEach(function(collName) {
    var exists = db.getCollectionNames().indexOf(collName) !== -1;

    if (exists) {
        var count = db[collName].count();
        print("Coleccion: " + collName + " - Documentos: " + count);

        if (count > 0) {
            print("  ADVERTENCIA: Coleccion tiene datos. Backup recomendado.");
        }

        db[collName].drop();
        print("  ELIMINADA");
    } else {
        print("Coleccion: " + collName + " - NO EXISTE");
    }
});

print("=== Limpieza completada ===");
```

### Paso 4.5: Commit parcial

```bash
git add mongodb/
git commit -m "feat(mongodb): eliminar colecciones sin uso - PT-002

- Eliminar: material_content, assessment_attempt_result
- Eliminar: audit_logs, notifications, analytics_events
- Eliminar migraciones de estructura y constraints
- Eliminar seeds correspondientes
- Agregar script de limpieza para BD existentes

BREAKING CHANGE: Colecciones eliminadas permanentemente"
```

---

## Fase 5: MongoDB - Homologar material_assessment (Tarea 03)

### Paso 5.1: Eliminar migracion duplicada

```bash
rm -f mongodb/migrations/structure/001_material_assessment.go
rm -f mongodb/migrations/constraints/001_material_assessment_indexes.go
rm -f mongodb/seeds/material_assessment.js
rm -f seeds/mongodb/material_assessment.js
```

### Paso 5.2: Crear script de migracion de datos

Crear archivo: `mongodb/scripts/migrate_material_assessment.js`

```javascript
// Script: migrate_material_assessment.js
// Description: Migrar datos de material_assessment a material_assessment_worker
// Tickets: PT-003

print("=== Migracion material_assessment -> material_assessment_worker ===");
print("Fecha: " + new Date().toISOString());

var oldCount = db.material_assessment.count();
var workerCount = db.material_assessment_worker.count();

print("Documentos en material_assessment: " + oldCount);
print("Documentos en material_assessment_worker: " + workerCount);

if (oldCount === 0) {
    print("No hay documentos para migrar. Saliendo.");
    quit();
}

function calculateTotalPoints(questions) {
    if (!questions) return 0;
    return questions.reduce(function(sum, q) {
        return sum + (q.points || 1);
    }, 0);
}

var migrated = 0, skipped = 0, errors = 0;

db.material_assessment.find().forEach(function(oldDoc) {
    try {
        var exists = db.material_assessment_worker.findOne({material_id: oldDoc.material_id});
        if (exists) {
            print("  SKIP: " + oldDoc.material_id);
            skipped++;
            return;
        }

        var newDoc = {
            material_id: oldDoc.material_id,
            questions: oldDoc.questions || [],
            total_questions: oldDoc.questions ? oldDoc.questions.length : 0,
            total_points: calculateTotalPoints(oldDoc.questions),
            version: oldDoc.version || 1,
            ai_model: (oldDoc.metadata && oldDoc.metadata.generated_by) ? oldDoc.metadata.generated_by : "migrated",
            processing_time_ms: 0,
            metadata: oldDoc.metadata || {},
            created_at: oldDoc.created_at || new Date(),
            updated_at: new Date()
        };

        db.material_assessment_worker.insertOne(newDoc);
        print("  OK: " + oldDoc.material_id);
        migrated++;
    } catch (e) {
        print("  ERROR: " + oldDoc.material_id + " - " + e.message);
        errors++;
    }
});

print("");
print("=== Resumen ===");
print("Migrados: " + migrated);
print("Omitidos: " + skipped);
print("Errores: " + errors);

if (errors === 0 && migrated + skipped === oldCount) {
    print("");
    print("EXITO: Ejecutar db.material_assessment.drop() despues de verificar");
}
```

### Paso 5.3: Commit parcial

```bash
git add mongodb/
git commit -m "feat(mongodb): homologar material_assessment - PT-003

- Eliminar migracion duplicada 001_material_assessment.go
- Eliminar indexes y seeds correspondientes
- material_assessment_worker es ahora la coleccion canonica
- Agregar script de migracion de datos

BREAKING CHANGE: Usar material_assessment_worker en lugar de material_assessment"
```

---

## Fase 6: MongoDB - Script Migracion material_summaries (Tarea 04)

### Paso 6.1: Crear script de migracion

Crear archivo: `mongodb/scripts/migrate_material_summaries.js`

```javascript
// Script: migrate_material_summaries.js
// Description: Migrar datos de material_summaries (plural) a material_summary (singular)
// Tickets: PT-004

print("=== Migracion material_summaries -> material_summary ===");
print("Fecha: " + new Date().toISOString());

var oldExists = db.getCollectionNames().indexOf("material_summaries") !== -1;
if (!oldExists) {
    print("No existe coleccion material_summaries. Saliendo.");
    quit();
}

var oldCount = db.material_summaries.count();
print("Documentos en material_summaries: " + oldCount);

if (oldCount === 0) {
    print("No hay documentos para migrar.");
    quit();
}

function generateSummary(mainIdeas) {
    if (!mainIdeas || mainIdeas.length === 0) return "";
    return mainIdeas.join(". ") + ".";
}

function calculateWordCount(mainIdeas) {
    if (!mainIdeas) return 0;
    return mainIdeas.join(" ").split(/\s+/).length;
}

function detectLanguage(mainIdeas) {
    if (!mainIdeas || mainIdeas.length === 0) return "es";
    var text = mainIdeas.join(" ").toLowerCase();
    if (text.match(/\b(el|la|los|las|de|del|y|que|en)\b/)) return "es";
    if (text.match(/\b(the|and|of|to|in|is|that)\b/)) return "en";
    return "es";
}

var migrated = 0, skipped = 0, errors = 0;

db.material_summaries.find().forEach(function(oldDoc) {
    try {
        var exists = db.material_summary.findOne({material_id: oldDoc.material_id});
        if (exists) {
            print("  SKIP: " + oldDoc.material_id);
            skipped++;
            return;
        }

        var newDoc = {
            material_id: oldDoc.material_id,
            summary: generateSummary(oldDoc.main_ideas),
            key_points: oldDoc.main_ideas || [],
            language: detectLanguage(oldDoc.main_ideas),
            word_count: calculateWordCount(oldDoc.main_ideas),
            version: 1,
            ai_model: "migrated",
            processing_time_ms: 0,
            metadata: {
                migrated_from: "material_summaries",
                original_sections: oldDoc.sections || [],
                original_glossary: oldDoc.glossary || [],
                original_key_concepts: oldDoc.key_concepts || []
            },
            created_at: oldDoc.created_at || new Date(),
            updated_at: new Date()
        };

        db.material_summary.insertOne(newDoc);
        print("  OK: " + oldDoc.material_id);
        migrated++;
    } catch (e) {
        print("  ERROR: " + oldDoc.material_id + " - " + e.message);
        errors++;
    }
});

print("");
print("=== Resumen ===");
print("Migrados: " + migrated);
print("Omitidos: " + skipped);
print("Errores: " + errors);

if (errors === 0) {
    print("");
    print("EXITO: Ejecutar db.material_summaries.drop() despues de verificar");
}
```

### Paso 6.2: Commit parcial

```bash
git add mongodb/
git commit -m "feat(mongodb): agregar script migracion material_summaries - PT-004

- Script para migrar material_summaries (plural) a material_summary (singular)
- Transformacion de schema a formato canonico
- material_summary es la coleccion canonica"
```

---

## Fase 7: Compilar y Test

### Paso 7.1: Compilar todos los modulos

```bash
cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure
go build ./...
```

### Paso 7.2: Ejecutar tests

```bash
go test ./... -v
go test ./... -cover
```

### Paso 7.3: Lint

```bash
golangci-lint run ./...
```

---

## Fase 8: Documentar

### Paso 8.1: Actualizar CHANGELOG.md

```markdown
## [Unreleased]

### postgres/v0.15.0 - 2025-12-XX

#### Removed
- Tabla `user_active_context` - Fase 1 UI nunca implementada
- Tabla `user_favorites` - Funcionalidad no implementada
- Tabla `user_activity_log` - Analytics no implementado
- Tabla `feature_flags` - Deuda tecnica no resuelta
- Tabla `feature_flag_overrides` - Dependencia de feature_flags
- Campo `email_verified` de users - Sin verificacion implementada
- Campo `last_login_at` de users - Sin registro de login
- Campo `preferences` de users - Sin UI de preferencias
- Seeds de testing para tablas/campos eliminados

#### Breaking Changes
- Tablas y campos eliminados permanentemente
- Backup requerido antes de ejecutar migraciones

---

### mongodb/v0.13.0 - 2025-12-XX

#### Removed
- Coleccion `material_content` - Procesamiento PDF no implementado
- Coleccion `assessment_attempt_result` - Datos en PostgreSQL
- Coleccion `audit_logs` - Usar SaaS para auditoria
- Coleccion `notifications` - Push notifications no implementado
- Coleccion `analytics_events` - Analytics usa servicio externo
- Migracion duplicada `001_material_assessment.go`

#### Changed
- `material_assessment_worker` es ahora la unica coleccion de assessments
- `material_summary` es la coleccion canonica (singular)

#### Added
- Script `cleanup_unused_collections.js`
- Script `migrate_material_assessment.js`
- Script `migrate_material_summaries.js`

#### Breaking Changes
- Colecciones eliminadas permanentemente
- APIs deben usar `material_assessment_worker` en lugar de `material_assessment`
- APIs deben usar `material_summary` en lugar de `material_summaries`
```

### Paso 8.2: Commit documentacion

```bash
git add CHANGELOG.md
git commit -m "docs: actualizar CHANGELOG con cambios consolidados

- Documentar eliminacion de tablas PostgreSQL
- Documentar eliminacion de campos de users
- Documentar eliminacion de colecciones MongoDB
- Documentar homologacion de colecciones
- Documentar scripts de migracion"
```

---

## Fase 9: Pull Request a Dev

### Paso 9.1: Push rama

```bash
git push -u origin feature/PT-000-release-consolidado-limpieza
```

### Paso 9.2: Crear PR

**Titulo:** `feat: Release consolidado de limpieza - PostgreSQL y MongoDB`

**Descripcion:**

```markdown
## Descripcion

Release consolidado que incluye todas las tareas de limpieza de infraestructura:

### PostgreSQL (postgres/v0.15.0)

**Tablas eliminadas:**
- `user_active_context` - Fase 1 UI nunca implementada
- `user_favorites` - Funcionalidad no implementada
- `user_activity_log` - Analytics no implementado
- `feature_flags` - Deuda tecnica
- `feature_flag_overrides` - Dependencia de feature_flags

**Campos eliminados de users:**
- `email_verified` - Sin verificacion
- `last_login_at` - Sin registro
- `preferences` - Sin UI

### MongoDB (mongodb/v0.13.0)

**Colecciones eliminadas:**
- `material_content`
- `assessment_attempt_result`
- `audit_logs`
- `notifications`
- `analytics_events`

**Homologacion:**
- `material_assessment` -> usar `material_assessment_worker`
- `material_summaries` -> usar `material_summary`

## Tickets

- PT-001: Eliminar tablas PostgreSQL
- PT-002: Eliminar colecciones MongoDB
- PT-003: Homologar material_assessment
- PT-004: Homologar material_summary
- PT-005: Eliminar campos sin uso

## Checklist

- [x] Migracion 016 (drop tables)
- [x] Migracion 017 (drop fields)
- [x] Entity User actualizada
- [x] Colecciones MongoDB eliminadas
- [x] Scripts de migracion creados
- [x] `go build ./...` exitoso
- [x] `go test ./...` exitoso
- [x] `golangci-lint run` limpio
- [x] CHANGELOG actualizado

## BREAKING CHANGES

1. Tablas PostgreSQL eliminadas
2. Campos de users eliminados
3. Colecciones MongoDB eliminadas
4. Cambio de nombre de colecciones

## Migracion en Ambientes

```bash
# 1. Backup PostgreSQL
pg_dump -h localhost -U edugo -d edugo_db > backup_before_v0.15.0.sql

# 2. Backup MongoDB
mongodump --db edugo_db --out /backup/before_v0.13.0

# 3. Ejecutar migraciones PostgreSQL
psql -h localhost -U edugo -d edugo_db -f postgres/migrations/structure/016_drop_unused_tables.sql
psql -h localhost -U edugo -d edugo_db -f postgres/migrations/structure/017_drop_unused_user_fields.sql

# 4. Ejecutar scripts MongoDB
mongo edugo_db mongodb/scripts/cleanup_unused_collections.js
mongo edugo_db mongodb/scripts/migrate_material_assessment.js
mongo edugo_db mongodb/scripts/migrate_material_summaries.js

# 5. Eliminar colecciones antiguas (despues de verificar)
mongo edugo_db --eval "db.material_assessment.drop()"
mongo edugo_db --eval "db.material_summaries.drop()"
```
```

### Paso 9.3: Review y merge a dev

---

## Fase 10: Release

### Paso 10.1: PR de dev a main

Crear PR de `dev` -> `main`

**Titulo:** `release: edugo-infrastructure postgres/v0.15.0 + mongodb/v0.13.0`

### Paso 10.2: Merge a main

### Paso 10.3: Crear tags y releases

```bash
git checkout main
git pull origin main

# Tag PostgreSQL
git tag postgres/v0.15.0
git push origin postgres/v0.15.0

# Tag MongoDB
git tag mongodb/v0.13.0
git push origin mongodb/v0.13.0
```

### Paso 10.4: Crear GitHub Releases

**Release 1: postgres/v0.15.0**

```markdown
## edugo-infrastructure/postgres v0.15.0

### Removed
- Table `user_active_context`
- Table `user_favorites`
- Table `user_activity_log`
- Table `feature_flags`
- Table `feature_flag_overrides`
- Field `email_verified` from users
- Field `last_login_at` from users
- Field `preferences` from users

### Breaking Changes
- All removed tables/fields are permanent
- Backup required before migration

### Migration
Run migrations 016 and 017 after backup.
```

**Release 2: mongodb/v0.13.0**

```markdown
## edugo-infrastructure/mongodb v0.13.0

### Removed
- Collection `material_content`
- Collection `assessment_attempt_result`
- Collection `audit_logs`
- Collection `notifications`
- Collection `analytics_events`
- Duplicate migration for `material_assessment`

### Changed
- Use `material_assessment_worker` (canonical)
- Use `material_summary` (singular, canonical)

### Added
- `mongodb/scripts/cleanup_unused_collections.js`
- `mongodb/scripts/migrate_material_assessment.js`
- `mongodb/scripts/migrate_material_summaries.js`

### Breaking Changes
- Run migration scripts before updating dependent projects
```

---

## Fase 11: Verificacion Post-Release

### Paso 11.1: Verificar releases disponibles

```bash
# Verificar PostgreSQL
go list -m -versions github.com/EduGoGroup/edugo-infrastructure/postgres

# Verificar MongoDB
go list -m -versions github.com/EduGoGroup/edugo-infrastructure/mongodb
```

### Paso 11.2: Test de actualizacion

```bash
# En un proyecto de prueba
go get github.com/EduGoGroup/edugo-infrastructure/postgres@v0.15.0
go get github.com/EduGoGroup/edugo-infrastructure/mongodb@v0.13.0
go build ./...
```

---

## Checklist Final

### Preparacion
- [ ] Rama feature creada desde dev

### PostgreSQL
- [ ] Migracion 016 (drop tables) creada
- [ ] Migracion 017 (drop fields) creada
- [ ] Entity User actualizada
- [ ] Seeds de testing actualizados
- [ ] Commit PostgreSQL realizado

### MongoDB
- [ ] Migraciones de colecciones sin uso eliminadas
- [ ] Indexes eliminados
- [ ] Seeds eliminados
- [ ] Script cleanup_unused_collections.js creado
- [ ] Migracion duplicada material_assessment eliminada
- [ ] Script migrate_material_assessment.js creado
- [ ] Script migrate_material_summaries.js creado
- [ ] Commits MongoDB realizados

### Validacion
- [ ] `go build ./...` exitoso
- [ ] `go test ./...` exitoso
- [ ] `golangci-lint run` limpio
- [ ] CHANGELOG actualizado

### Release
- [ ] PR a dev creado
- [ ] PR a dev aprobado y mergeado
- [ ] PR de dev a main creado
- [ ] PR de dev a main aprobado y mergeado
- [ ] Tag `postgres/v0.15.0` creado
- [ ] Tag `mongodb/v0.13.0` creado
- [ ] GitHub Release postgres publicado
- [ ] GitHub Release mongodb publicado
- [ ] Releases verificados con `go list`

---

## Comandos de Migracion para Ambientes

### Desarrollo Local

```bash
# PostgreSQL
psql -h localhost -U edugo -d edugo_db -f postgres/migrations/structure/016_drop_unused_tables.sql
psql -h localhost -U edugo -d edugo_db -f postgres/migrations/structure/017_drop_unused_user_fields.sql

# MongoDB
mongo edugo_db mongodb/scripts/cleanup_unused_collections.js
mongo edugo_db mongodb/scripts/migrate_material_assessment.js
mongo edugo_db mongodb/scripts/migrate_material_summaries.js
mongo edugo_db --eval "db.material_assessment.drop(); db.material_summaries.drop();"
```

### Staging

```bash
# BACKUP PRIMERO
pg_dump -h staging-host -U edugo -d edugo_db > backup_staging_$(date +%Y%m%d).sql
mongodump --host staging-mongo --db edugo_db --out /backup/staging_$(date +%Y%m%d)

# Luego ejecutar los mismos comandos de desarrollo
```

### Produccion

```bash
# BACKUP OBLIGATORIO
pg_dump -h prod-host -U edugo -d edugo_db > backup_prod_$(date +%Y%m%d).sql
mongodump --host prod-mongo --db edugo_db --out /backup/prod_$(date +%Y%m%d)

# Ejecutar en ventana de mantenimiento
# Notificar a equipos antes de ejecutar
```

---

## Siguiente Paso

Una vez completado este release, los proyectos dependientes pueden actualizar:

1. **edugo-api-administracion** - Actualizar infra + completar Tarea 05 y 06
2. **edugo-api-mobile** - Actualizar infra + completar Tarea 03, 04 y 07
3. **edugo-worker** - Actualizar infra (verificacion solamente)

Ver carpetas individuales para planes de cada proyecto:
- [../03-homologar-material-assessment/api-mobile/](../03-homologar-material-assessment/api-mobile/)
- [../04-homologar-material-summary/api-mobile/](../04-homologar-material-summary/api-mobile/)
- [../05-eliminar-campos-sin-uso/api-admin/](../05-eliminar-campos-sin-uso/api-admin/)
- [../06-endpoints-faltantes-api-admin/api-admin/](../06-endpoints-faltantes-api-admin/api-admin/)
- [../07-endpoints-faltantes-api-mobile/api-mobile/](../07-endpoints-faltantes-api-mobile/api-mobile/)
