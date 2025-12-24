# Plan: edugo-infrastructure - Eliminar Tablas PostgreSQL

**Proyecto:** edugo-infrastructure
**Rama base:** dev
**Rama feature:** `feature/PT-001-eliminar-tablas-postgres-sin-uso`

---

## Fase 1: Preparacion

### Paso 1.1: Crear rama

```bash
cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure
git fetch origin
git checkout dev
git pull origin dev
git checkout -b feature/PT-001-eliminar-tablas-postgres-sin-uso
```

### Paso 1.2: Verificar estado actual

```bash
# Ver migraciones existentes
ls -la postgres/migrations/structure/

# Verificar que las tablas existen solo en migraciones
grep -r "user_active_context" --include="*.go" --include="*.sql"
grep -r "user_favorites" --include="*.go" --include="*.sql"
grep -r "user_activity_log" --include="*.go" --include="*.sql"
grep -r "feature_flags" --include="*.go" --include="*.sql"
```

---

## Fase 2: Crear Migraciones de Eliminacion

### Paso 2.1: Crear migracion para eliminar tablas

Crear archivo: `postgres/migrations/structure/016_drop_unused_tables.sql`

```sql
-- Migration: 016_drop_unused_tables.sql
-- Description: Eliminar tablas sin uso en el ecosistema
-- Author: [Tu nombre]
-- Date: 2025-12-XX
-- Ticket: PT-001

-- IMPORTANTE: Ejecutar backup antes de esta migracion en produccion
-- pg_dump -h localhost -U edugo -d edugo_db > backup_before_016.sql

-- Eliminar en orden inverso de dependencias
-- feature_flag_overrides depende de feature_flags

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

-- Comentario de finalizacion
COMMENT ON SCHEMA public IS 'Migracion 016: Tablas user_active_context, user_favorites, user_activity_log, feature_flags, feature_flag_overrides eliminadas por no uso';
```

### Paso 2.2: Crear migracion de rollback (opcional pero recomendado)

Crear archivo: `postgres/migrations/structure/016_drop_unused_tables.down.sql`

```sql
-- Rollback: Recrear tablas eliminadas
-- NOTA: Este rollback solo recrea estructuras, NO recupera datos

-- Ver migraciones originales 011-015 para estructura completa
-- Este archivo es solo para referencia, NO ejecutar automaticamente
```

---

## Fase 3: Actualizar Seeds de Testing

### Paso 3.1: Eliminar seeds de tablas eliminadas

```bash
# Eliminar archivos de seeds de testing
rm -f postgres/migrations/testing/006_demo_user_active_context.sql
rm -f postgres/migrations/testing/007_demo_user_favorites.sql
rm -f postgres/migrations/testing/008_demo_user_activity_log.sql
rm -f postgres/migrations/testing/009_demo_feature_flags.sql
rm -f postgres/migrations/testing/010_demo_feature_flag_overrides.sql
```

### Paso 3.2: Actualizar indice de seeds (si existe)

Si hay un archivo que lista seeds, actualizarlo para remover referencias.

---

## Fase 4: Compilar y Test

### Paso 4.1: Compilar

```bash
cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure

# Compilar todos los modulos
go build ./...
```

### Paso 4.2: Ejecutar tests

```bash
# Ejecutar tests
go test ./... -v

# Verificar cobertura
go test ./... -cover
```

### Paso 4.3: Lint

```bash
# Ejecutar linter
golangci-lint run ./...
```

---

## Fase 5: Documentar

### Paso 5.1: Actualizar CHANGELOG.md (si existe)

Agregar entrada:

```markdown
## [0.14.0] - 2025-12-XX

### Removed
- Tabla `user_active_context` - Fase 1 UI nunca implementada
- Tabla `user_favorites` - Funcionalidad no implementada
- Tabla `user_activity_log` - Analytics no implementado
- Tabla `feature_flags` - Deuda tecnica no resuelta
- Tabla `feature_flag_overrides` - Dependencia de feature_flags
- Seeds de testing para tablas eliminadas
```

### Paso 5.2: Commit

```bash
git add .
git commit -m "feat(postgres): eliminar tablas sin uso en ecosistema

- Eliminar user_active_context (Fase 1 UI no implementada)
- Eliminar user_favorites (funcionalidad no existe)
- Eliminar user_activity_log (analytics no implementado)
- Eliminar feature_flags y feature_flag_overrides (deuda tecnica)
- Eliminar seeds de testing correspondientes

BREAKING CHANGE: Tablas eliminadas permanentemente

Ticket: PT-001"
```

---

## Fase 6: Pull Request a Dev

### Paso 6.1: Push rama

```bash
git push -u origin feature/PT-001-eliminar-tablas-postgres-sin-uso
```

### Paso 6.2: Crear PR

**Titulo:** `feat(postgres): Eliminar tablas PostgreSQL sin uso`

**Descripcion:**

```markdown
## Descripcion

Elimina 5 tablas de PostgreSQL que fueron creadas pero nunca utilizadas:

- `user_active_context` - Fase 1 UI nunca implementada
- `user_favorites` - Funcionalidad de favoritos no existe
- `user_activity_log` - Analytics UI no implementado
- `feature_flags` - Deuda tecnica Apple App
- `feature_flag_overrides` - Depende de feature_flags

## Verificacion

Se verifico que NO hay referencias a estas tablas en:
- [x] edugo-api-administracion
- [x] edugo-api-mobile
- [x] edugo-worker

## Checklist

- [x] Migracion de eliminacion creada
- [x] Seeds de testing eliminados
- [x] `go build ./...` exitoso
- [x] `go test ./...` exitoso
- [x] `golangci-lint run` sin errores
- [x] CHANGELOG actualizado

## BREAKING CHANGE

Las tablas eliminadas NO se pueden recuperar sin backup.
Asegurar backup de produccion antes de ejecutar migracion.

## Rollback

```sql
-- Ver migraciones 011-015 para recrear estructuras
-- Los datos NO son recuperables sin backup
```
```

### Paso 6.3: Esperar review y merge a dev

- Solicitar review
- Aprobar PR
- Merge a dev

---

## Fase 7: Release

### Paso 7.1: PR de dev a main

```bash
# En GitHub, crear PR de dev -> main
```

**Titulo:** `release: edugo-infrastructure/postgres v0.14.0`

### Paso 7.2: Merge a main

Despues de aprobar, merge a main.

### Paso 7.3: Crear GitHub Release

```bash
# Crear tag
git checkout main
git pull origin main
git tag postgres/v0.14.0
git push origin postgres/v0.14.0
```

En GitHub:
1. Ir a Releases
2. Create new release
3. Tag: `postgres/v0.14.0`
4. Title: `edugo-infrastructure/postgres v0.14.0`
5. Description:

```markdown
## Changes

### Removed
- Table `user_active_context`
- Table `user_favorites`
- Table `user_activity_log`
- Table `feature_flags`
- Table `feature_flag_overrides`
- Testing seeds for removed tables

### Breaking Changes
- Tables permanently removed. Backup required before migration.

## Migration

Run migration `016_drop_unused_tables.sql` after backing up database.
```

---

## Fase 8: Verificacion Post-Release

### Paso 8.1: Verificar que el release esta disponible

```bash
# En cualquier proyecto que use infra
go list -m -versions github.com/EduGoGroup/edugo-infrastructure/postgres
```

Debe mostrar `v0.14.0` en la lista.

### Paso 8.2: Actualizar proyectos dependientes (si es necesario)

Para esta tarea, NO es necesario actualizar otros proyectos porque las tablas no tienen referencias.

---

## Checklist Final

- [ ] Rama feature creada desde dev
- [ ] Migracion 016 creada
- [ ] Seeds eliminados
- [ ] `go build ./...` exitoso
- [ ] `go test ./...` exitoso
- [ ] `golangci-lint run` limpio
- [ ] CHANGELOG actualizado
- [ ] PR a dev creado
- [ ] PR a dev aprobado y mergeado
- [ ] PR de dev a main creado
- [ ] PR de dev a main aprobado y mergeado
- [ ] Tag `postgres/v0.14.0` creado
- [ ] GitHub Release publicado
- [ ] Release verificado con `go list`

---

## Notas

- **Backup:** SIEMPRE hacer backup antes de ejecutar en produccion
- **Rollback:** Solo posible con backup, las migraciones de eliminacion son destructivas
- **Dependencias:** Ninguna tabla tenia dependencias en codigo

---

**Siguiente tarea:** [02-eliminar-colecciones-mongodb](../../02-eliminar-colecciones-mongodb/)
