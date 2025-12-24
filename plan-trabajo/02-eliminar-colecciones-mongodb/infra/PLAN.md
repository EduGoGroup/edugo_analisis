# Plan: edugo-infrastructure - Eliminar Colecciones MongoDB

**Proyecto:** edugo-infrastructure
**Rama base:** dev
**Rama feature:** `feature/PT-002-eliminar-colecciones-mongodb-sin-uso`

---

## Fase 1: Preparacion

### Paso 1.1: Crear rama

```bash
cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure
git fetch origin
git checkout dev
git pull origin dev
git checkout -b feature/PT-002-eliminar-colecciones-mongodb-sin-uso
```

### Paso 1.2: Verificar estado actual

```bash
# Ver migraciones existentes
ls -la mongodb/migrations/structure/
ls -la mongodb/migrations/constraints/

# Verificar que las colecciones no tienen referencias
grep -r "material_content" --include="*.go" .
grep -r "assessment_attempt_result" --include="*.go" .
grep -r "audit_logs" --include="*.go" .
grep -r "notifications" --include="*.go" .
grep -r "analytics_events" --include="*.go" .
```

---

## Fase 2: Eliminar Archivos de Migraciones

### Paso 2.1: Eliminar migraciones de estructura

```bash
# Eliminar migraciones de creacion de colecciones
rm -f mongodb/migrations/structure/002_material_content.go
rm -f mongodb/migrations/structure/003_assessment_attempt_result.go
rm -f mongodb/migrations/structure/004_audit_logs.go
rm -f mongodb/migrations/structure/005_notifications.go
rm -f mongodb/migrations/structure/006_analytics_events.go
```

### Paso 2.2: Eliminar migraciones de constraints/indexes

```bash
# Eliminar archivos de indexes
rm -f mongodb/migrations/constraints/002_material_content_indexes.go
rm -f mongodb/migrations/constraints/003_assessment_attempt_result_indexes.go
rm -f mongodb/migrations/constraints/004_audit_logs_indexes.go
rm -f mongodb/migrations/constraints/005_notifications_indexes.go
rm -f mongodb/migrations/constraints/006_analytics_events_indexes.go
```

---

## Fase 3: Eliminar Seeds

### Paso 3.1: Eliminar seeds de MongoDB

```bash
# Eliminar seeds en mongodb/seeds/
rm -f mongodb/seeds/material_content.js
rm -f mongodb/seeds/assessment_attempt_result.js
rm -f mongodb/seeds/audit_logs.js
rm -f mongodb/seeds/notifications.js
rm -f mongodb/seeds/analytics_events.js

# Eliminar seeds en seeds/mongodb/
rm -f seeds/mongodb/material_content.js
rm -f seeds/mongodb/assessment_attempt_result.js
rm -f seeds/mongodb/audit_logs.js
rm -f seeds/mongodb/notifications.js
rm -f seeds/mongodb/analytics_events.js
```

### Paso 3.2: Actualizar script de ejecucion de seeds (si existe)

Verificar si hay un script que ejecute todos los seeds y remover las referencias.

---

## Fase 4: Crear Script de Limpieza para BD Existentes

### Paso 4.1: Crear script de limpieza

Crear archivo: `mongodb/scripts/cleanup_unused_collections.js`

```javascript
// Script: cleanup_unused_collections.js
// Description: Eliminar colecciones sin uso del ecosistema
// Author: [Tu nombre]
// Date: 2025-12-XX
// Ticket: PT-002

// IMPORTANTE: Ejecutar backup antes de esta operacion en produccion
// mongodump --db edugo_db --out /backup/edugo_db_before_cleanup

print("=== Limpieza de colecciones sin uso ===");
print("Fecha: " + new Date().toISOString());

// Lista de colecciones a eliminar
var collectionsToDelete = [
    "material_content",
    "assessment_attempt_result",
    "audit_logs",
    "notifications",
    "analytics_events"
];

collectionsToDelete.forEach(function(collName) {
    // Verificar si existe
    var exists = db.getCollectionNames().indexOf(collName) !== -1;

    if (exists) {
        // Contar documentos antes de eliminar
        var count = db[collName].count();
        print("Coleccion: " + collName + " - Documentos: " + count);

        if (count > 0) {
            print("  ADVERTENCIA: Coleccion tiene datos. Backup recomendado.");
        }

        // Eliminar coleccion
        db[collName].drop();
        print("  ELIMINADA");
    } else {
        print("Coleccion: " + collName + " - NO EXISTE (ya fue eliminada)");
    }
});

print("=== Limpieza completada ===");
```

---

## Fase 5: Compilar y Test

### Paso 5.1: Compilar

```bash
cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure

# Compilar todos los modulos
go build ./...
```

### Paso 5.2: Ejecutar tests

```bash
# Ejecutar tests
go test ./... -v

# Verificar cobertura
go test ./... -cover
```

### Paso 5.3: Lint

```bash
# Ejecutar linter
golangci-lint run ./...
```

---

## Fase 6: Documentar

### Paso 6.1: Actualizar CHANGELOG.md (si existe)

Agregar entrada:

```markdown
## [0.12.0] - 2025-12-XX

### Removed
- Coleccion `material_content` - Procesamiento PDF no implementado
- Coleccion `assessment_attempt_result` - Datos en PostgreSQL
- Coleccion `audit_logs` - Auditoria no implementada (usar SaaS)
- Coleccion `notifications` - Push notifications no implementado
- Coleccion `analytics_events` - Analytics usa servicio externo
- Seeds de testing para colecciones eliminadas
- Indexes para colecciones eliminadas
```

### Paso 6.2: Commit

```bash
git add .
git commit -m "feat(mongodb): eliminar colecciones sin uso en ecosistema

- Eliminar material_content (procesamiento PDF no implementado)
- Eliminar assessment_attempt_result (datos estan en PostgreSQL)
- Eliminar audit_logs (usar SaaS para auditoria)
- Eliminar notifications (push notifications no implementado)
- Eliminar analytics_events (analytics usa servicio externo)
- Eliminar seeds y indexes correspondientes
- Agregar script de limpieza para BD existentes

BREAKING CHANGE: Colecciones eliminadas permanentemente

Ticket: PT-002"
```

---

## Fase 7: Pull Request a Dev

### Paso 7.1: Push rama

```bash
git push -u origin feature/PT-002-eliminar-colecciones-mongodb-sin-uso
```

### Paso 7.2: Crear PR

**Titulo:** `feat(mongodb): Eliminar colecciones MongoDB sin uso`

**Descripcion:**

```markdown
## Descripcion

Elimina 5 colecciones de MongoDB que fueron creadas pero nunca utilizadas:

- `material_content` - Procesamiento de contenido PDF no implementado
- `assessment_attempt_result` - Datos almacenados en PostgreSQL
- `audit_logs` - Sistema de auditoria no implementado (evaluar SaaS)
- `notifications` - Push notifications no implementado
- `analytics_events` - Analytics usando servicio externo

## Verificacion

Se verifico que NO hay referencias a estas colecciones en:
- [x] edugo-api-administracion (0 referencias)
- [x] edugo-api-mobile (solo testing-config.yml)
- [x] edugo-worker (0 referencias)

## Checklist

- [x] Migraciones de estructura eliminadas
- [x] Migraciones de constraints/indexes eliminadas
- [x] Seeds eliminados
- [x] Script de limpieza creado
- [x] `go build ./...` exitoso
- [x] `go test ./...` exitoso
- [x] `golangci-lint run` sin errores
- [x] CHANGELOG actualizado

## BREAKING CHANGE

Las colecciones eliminadas NO se pueden recuperar sin backup.
Asegurar backup de MongoDB antes de ejecutar script de limpieza.

## Script de Limpieza

Para BD existentes, ejecutar:
```bash
mongo edugo_db mongodb/scripts/cleanup_unused_collections.js
```
```

### Paso 7.3: Esperar review y merge a dev

- Solicitar review
- Aprobar PR
- Merge a dev

---

## Fase 8: Release

### Paso 8.1: PR de dev a main

```bash
# En GitHub, crear PR de dev -> main
```

**Titulo:** `release: edugo-infrastructure/mongodb v0.12.0`

### Paso 8.2: Merge a main

Despues de aprobar, merge a main.

### Paso 8.3: Crear GitHub Release

```bash
# Crear tag
git checkout main
git pull origin main
git tag mongodb/v0.12.0
git push origin mongodb/v0.12.0
```

En GitHub:
1. Ir a Releases
2. Create new release
3. Tag: `mongodb/v0.12.0`
4. Title: `edugo-infrastructure/mongodb v0.12.0`
5. Description:

```markdown
## Changes

### Removed
- Collection `material_content`
- Collection `assessment_attempt_result`
- Collection `audit_logs`
- Collection `notifications`
- Collection `analytics_events`
- All related seeds and indexes

### Added
- Cleanup script for existing databases

### Breaking Changes
- Collections permanently removed. Backup required before cleanup.

## Cleanup

For existing databases, run the cleanup script:
```bash
mongodump --db edugo_db --out /backup/edugo_db_before_cleanup
mongo edugo_db mongodb/scripts/cleanup_unused_collections.js
```
```

---

## Fase 9: Verificacion Post-Release

### Paso 9.1: Verificar que el release esta disponible

```bash
# En cualquier proyecto que use infra
go list -m -versions github.com/EduGoGroup/edugo-infrastructure/mongodb
```

Debe mostrar `v0.12.0` en la lista.

### Paso 9.2: Actualizar proyectos dependientes (si es necesario)

Para esta tarea, NO es necesario actualizar otros proyectos porque las colecciones no tienen referencias.

---

## Checklist Final

- [ ] Rama feature creada desde dev
- [ ] Migraciones de estructura eliminadas
- [ ] Migraciones de constraints eliminadas
- [ ] Seeds eliminados
- [ ] Script de limpieza creado
- [ ] `go build ./...` exitoso
- [ ] `go test ./...` exitoso
- [ ] `golangci-lint run` limpio
- [ ] CHANGELOG actualizado
- [ ] PR a dev creado
- [ ] PR a dev aprobado y mergeado
- [ ] PR de dev a main creado
- [ ] PR de dev a main aprobado y mergeado
- [ ] Tag `mongodb/v0.12.0` creado
- [ ] GitHub Release publicado
- [ ] Release verificado con `go list`

---

## Notas

- **Backup:** SIEMPRE hacer backup de MongoDB antes de ejecutar script de limpieza
- **Rollback:** Solo posible con backup, las eliminaciones son destructivas
- **Dependencias:** Ninguna coleccion tenia dependencias en codigo
- **testing-config.yml en api-mobile:** Referencias en archivo de config no afectan funcionalidad

---

**Siguiente tarea:** [03-homologar-material-assessment](../../03-homologar-material-assessment/)
