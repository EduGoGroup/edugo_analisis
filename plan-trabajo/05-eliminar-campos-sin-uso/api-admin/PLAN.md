# Plan: edugo-api-administracion - Eliminar Campos Sin Uso

**Proyecto:** edugo-api-administracion
**Rama base:** dev
**Rama feature:** `feature/PT-005-eliminar-campos-sin-uso`

**IMPORTANTE:** Ejecutar DESPUES de que infra tenga el release v0.14.1

---

## Fase 1: Preparacion

### Paso 1.1: Crear rama

```bash
cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion
git fetch origin
git checkout dev
git pull origin dev
git checkout -b feature/PT-005-eliminar-campos-sin-uso
```

### Paso 1.2: Actualizar dependencia de infra

```bash
go get github.com/EduGoGroup/edugo-infrastructure/postgres@v0.14.1
go mod tidy
```

---

## Fase 2: Buscar y Eliminar Referencias

### Paso 2.1: Buscar referencias a los campos

```bash
# Buscar todas las referencias
grep -r "EmailVerified\|email_verified" --include="*.go" .
grep -r "LastLoginAt\|last_login_at" --include="*.go" .
grep -r "Preferences\|preferences" --include="*.go" .
```

### Paso 2.2: Eliminar referencias en repositorios

Por cada archivo encontrado, eliminar las referencias a estos campos.

**Archivos tipicos a revisar:**
- `internal/infrastructure/persistence/postgres/repository/user_repository_impl.go`
- `internal/application/dto/user_dto.go`
- `internal/application/service/user_service.go`

### Paso 2.3: Eliminar de DTOs

Si existen DTOs que exponen estos campos, eliminarlos:

```go
// ELIMINAR de cualquier DTO:
EmailVerified  bool      `json:"email_verified"`
LastLoginAt    *string   `json:"last_login_at"`
Preferences    any       `json:"preferences"`
```

---

## Fase 3: Actualizar Tests

### Paso 3.1: Buscar tests afectados

```bash
grep -r "EmailVerified\|email_verified\|LastLoginAt\|last_login_at\|Preferences\|preferences" --include="*_test.go" .
```

### Paso 3.2: Eliminar referencias en tests

Por cada test encontrado, actualizar para no usar estos campos.

---

## Fase 4: Compilar y Test

### Paso 4.1: Compilar

```bash
go build ./...
```

### Paso 4.2: Ejecutar tests

```bash
go test ./... -v
go test ./... -cover
```

### Paso 4.3: Lint

```bash
golangci-lint run ./...
```

### Paso 4.4: Regenerar swagger

```bash
swag init -g cmd/main.go -o docs
```

---

## Fase 5: Commit y PR

### Paso 5.1: Commit

```bash
git add .
git commit -m "refactor: eliminar referencias a campos sin uso de users

- Eliminar referencias a email_verified
- Eliminar referencias a last_login_at
- Eliminar referencias a preferences
- Actualizar DTOs
- Actualizar tests
- Actualizar dependencia edugo-infrastructure@postgres/v0.14.1

Ticket: PT-005"
```

### Paso 5.2: Push y crear PR

```bash
git push -u origin feature/PT-005-eliminar-campos-sin-uso
```

**Titulo PR:** `refactor: Eliminar referencias a campos sin uso de users`

**Descripcion:**

```markdown
## Descripcion

Elimina referencias a campos de la tabla users que fueron eliminados en infra v0.14.1:

- `email_verified`
- `last_login_at`
- `preferences`

## Dependencias

- Requiere: `edugo-infrastructure/postgres@v0.14.1` (ya publicado)

## Cambios

- [x] Referencias en repositorios eliminadas
- [x] DTOs actualizados
- [x] Tests actualizados
- [x] Swagger regenerado

## Checklist

- [x] `go build ./...` exitoso
- [x] `go test ./...` exitoso
- [x] `golangci-lint run` limpio
- [x] Swagger actualizado
```

---

## Checklist Final

- [ ] Rama feature creada
- [ ] Dependencia de infra actualizada
- [ ] Referencias en repositorios eliminadas
- [ ] DTOs actualizados
- [ ] Tests actualizados
- [ ] Swagger regenerado
- [ ] Build exitoso
- [ ] Tests exitosos
- [ ] Lint limpio
- [ ] PR a dev creado
- [ ] PR a dev mergeado

---

**Siguiente tarea:** [../../06-endpoints-faltantes-api-admin/](../../06-endpoints-faltantes-api-admin/)
