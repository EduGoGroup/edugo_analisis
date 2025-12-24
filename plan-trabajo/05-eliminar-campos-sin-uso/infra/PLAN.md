# Plan: edugo-infrastructure - Eliminar Campos Sin Uso

**Proyecto:** edugo-infrastructure
**Rama base:** dev
**Rama feature:** `feature/PT-005-eliminar-campos-sin-uso`

---

## Fase 1: Preparacion

### Paso 1.1: Crear rama

```bash
cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure
git fetch origin
git checkout dev
git pull origin dev
git checkout -b feature/PT-005-eliminar-campos-sin-uso
```

### Paso 1.2: Verificar campos en entity

```bash
cat postgres/entities/user.go
```

Verificar que existen los campos:
- EmailVerified
- LastLoginAt
- Preferences

---

## Fase 2: Crear Migracion de Eliminacion

### Paso 2.1: Crear migracion

Crear archivo: `postgres/migrations/structure/017_drop_unused_user_fields.sql`

```sql
-- Migration: 017_drop_unused_user_fields.sql
-- Description: Eliminar campos sin uso en tabla users
-- Author: [Tu nombre]
-- Date: 2025-12-XX
-- Ticket: PT-005

-- IMPORTANTE: Backup antes de ejecutar en produccion
-- pg_dump -h localhost -U edugo -d edugo_db -t users > backup_users_before_017.sql

-- 1. Eliminar campo email_verified
-- Sin logica de verificacion de email implementada
ALTER TABLE users DROP COLUMN IF EXISTS email_verified;

-- 2. Eliminar campo last_login_at
-- Sin registro de ultimo login implementado
ALTER TABLE users DROP COLUMN IF EXISTS last_login_at;

-- 3. Eliminar campo preferences
-- Sin UI de preferencias de usuario
ALTER TABLE users DROP COLUMN IF EXISTS preferences;

-- Comentario de finalizacion
COMMENT ON TABLE users IS 'Migracion 017: Campos email_verified, last_login_at, preferences eliminados por no uso';
```

### Paso 2.2: Crear migracion de rollback (opcional)

Crear archivo: `postgres/migrations/structure/017_drop_unused_user_fields.down.sql`

```sql
-- Rollback: Recrear campos eliminados
-- NOTA: Los datos NO son recuperables sin backup

ALTER TABLE users ADD COLUMN IF NOT EXISTS email_verified BOOLEAN NOT NULL DEFAULT false;
ALTER TABLE users ADD COLUMN IF NOT EXISTS last_login_at TIMESTAMP WITH TIME ZONE;
ALTER TABLE users ADD COLUMN IF NOT EXISTS preferences JSONB DEFAULT '{}';
```

---

## Fase 3: Actualizar Entity

### Paso 3.1: Modificar user.go

Archivo: `postgres/entities/user.go`

**ELIMINAR los siguientes campos del struct:**

```go
// ELIMINAR:
EmailVerified  bool           `gorm:"column:email_verified;not null;default:false"`
LastLoginAt    *time.Time     `gorm:"column:last_login_at"`
Preferences    datatypes.JSON `gorm:"column:preferences;default:'{}'"`
```

---

## Fase 4: Actualizar Seeds de Testing

### Paso 4.1: Modificar seeds que inserten usuarios

Revisar y modificar archivos de seeds que incluyan estos campos:

```bash
# Buscar referencias en seeds
grep -r "email_verified\|last_login_at\|preferences" postgres/migrations/testing/
```

Eliminar las columnas de los INSERT statements.

---

## Fase 5: Compilar y Test

### Paso 5.1: Compilar

```bash
go build ./...
```

### Paso 5.2: Ejecutar tests

```bash
go test ./... -v
go test ./... -cover
```

### Paso 5.3: Lint

```bash
golangci-lint run ./...
```

---

## Fase 6: Documentar y Commit

### Paso 6.1: Actualizar CHANGELOG

```markdown
## [0.14.1] - 2025-12-XX

### Removed
- Campo `email_verified` de tabla users - Sin verificacion implementada
- Campo `last_login_at` de tabla users - Sin registro de login
- Campo `preferences` de tabla users - Sin UI de preferencias
```

### Paso 6.2: Commit

```bash
git add .
git commit -m "feat(postgres): eliminar campos sin uso en tabla users

- Eliminar email_verified (sin verificacion de email)
- Eliminar last_login_at (sin registro de login)
- Eliminar preferences (sin UI de preferencias)
- Actualizar entity User
- Actualizar seeds de testing

Ticket: PT-005"
```

---

## Fase 7: Pull Request a Dev

### Paso 7.1: Push

```bash
git push -u origin feature/PT-005-eliminar-campos-sin-uso
```

### Paso 7.2: Crear PR

**Titulo:** `feat(postgres): Eliminar campos sin uso en tabla users`

**Descripcion:**

```markdown
## Descripcion

Elimina campos de la tabla users que fueron creados pero nunca implementados:

- `email_verified` - Sin logica de verificacion de email
- `last_login_at` - Sin registro de ultimo login
- `preferences` - Sin UI de preferencias de usuario

## Verificacion

Se verifico que estos campos NO tienen logica:
- [x] Sin endpoints de verificacion de email
- [x] Sin middleware de registro de login
- [x] Sin UI de preferencias

## Checklist

- [x] Migracion creada
- [x] Entity actualizada
- [x] Seeds actualizados
- [x] `go build ./...` exitoso
- [x] `go test ./...` exitoso
- [x] `golangci-lint run` limpio

## BREAKING CHANGE

Campos eliminados permanentemente. Backup requerido antes de migracion.
```

### Paso 7.3: Review y merge

---

## Fase 8: Release

### Paso 8.1: PR de dev a main

### Paso 8.2: Merge y release

```bash
git checkout main
git pull origin main
git tag postgres/v0.14.1
git push origin postgres/v0.14.1
```

**Titulo release:** `edugo-infrastructure/postgres v0.14.1`

---

## Checklist Final

- [ ] Rama feature creada
- [ ] Migracion de eliminacion creada
- [ ] Entity User actualizada
- [ ] Seeds actualizados
- [ ] Build exitoso
- [ ] Tests exitosos
- [ ] Lint limpio
- [ ] PR a dev mergeado
- [ ] PR a main mergeado
- [ ] Release publicado

---

**Siguiente paso:** [../api-admin/PLAN.md](../api-admin/PLAN.md)
