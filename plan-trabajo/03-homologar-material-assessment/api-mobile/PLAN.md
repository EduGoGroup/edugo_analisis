# Plan: edugo-api-mobile - Homologar material_assessment

**Proyecto:** edugo-api-mobile
**Rama base:** dev
**Rama feature:** `feature/PT-003-homologar-material-assessment`

**IMPORTANTE:** Ejecutar DESPUES de que infra tenga el release v0.12.1

---

## Fase 1: Preparacion

### Paso 1.1: Crear rama

```bash
cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile
git fetch origin
git checkout dev
git pull origin dev
git checkout -b feature/PT-003-homologar-material-assessment
```

### Paso 1.2: Actualizar dependencia de infra

```bash
# Actualizar a la version con coleccion homologada
go get github.com/EduGoGroup/edugo-infrastructure/mongodb@v0.12.1
go mod tidy
```

---

## Fase 2: Modificar Repository

### Paso 2.1: Identificar archivo a modificar

```bash
# Archivo principal a modificar
cat internal/infrastructure/persistence/mongodb/repository/assessment_document_repository.go
```

### Paso 2.2: Cambiar nombre de coleccion

Modificar linea 83 (aproximadamente):

**ANTES:**
```go
collection: db.Collection("material_assessment"),
```

**DESPUES:**
```go
collection: db.Collection("material_assessment_worker"),
```

### Paso 2.3: Actualizar struct AssessmentDocument

Agregar campos adicionales del schema worker:

**ANTES:**
```go
type AssessmentDocument struct {
    ID         primitive.ObjectID `bson:"_id,omitempty"`
    MaterialID string             `bson:"material_id"`
    Questions  []QuestionDoc      `bson:"questions"`
    Metadata   MetadataDoc        `bson:"metadata"`
    CreatedAt  time.Time          `bson:"created_at"`
    UpdatedAt  time.Time          `bson:"updated_at"`
}
```

**DESPUES:**
```go
type AssessmentDocument struct {
    ID               primitive.ObjectID `bson:"_id,omitempty"`
    MaterialID       string             `bson:"material_id"`
    Questions        []QuestionDoc      `bson:"questions"`
    TotalQuestions   int                `bson:"total_questions"`
    TotalPoints      int                `bson:"total_points"`
    Version          int                `bson:"version"`
    AIModel          string             `bson:"ai_model"`
    ProcessingTimeMs int                `bson:"processing_time_ms"`
    Metadata         MetadataDoc        `bson:"metadata"`
    CreatedAt        time.Time          `bson:"created_at"`
    UpdatedAt        time.Time          `bson:"updated_at"`
}
```

---

## Fase 3: Adaptar Service (si es necesario)

### Paso 3.1: Verificar assessment_attempt_service.go

```bash
cat internal/application/service/assessment_attempt_service.go
```

Si el service usa campos nuevos del schema, adaptar la logica.

**Generalmente no se requieren cambios** si solo se leen questions y metadata.

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

---

## Fase 5: Test Manual

### Paso 5.1: Levantar ambiente local

```bash
# Asegurar que MongoDB tiene datos migrados
docker-compose up -d

# Ejecutar API
go run cmd/main.go
```

### Paso 5.2: Probar endpoints de assessment

```bash
# Obtener un assessment
curl -X GET http://localhost:8081/v1/assessments/{material_id}

# Verificar que la respuesta incluye datos correctos
```

---

## Fase 6: Documentar y Commit

### Paso 6.1: Commit

```bash
git add .
git commit -m "feat(mongodb): usar coleccion material_assessment_worker

- Cambiar nombre de coleccion material_assessment -> material_assessment_worker
- Actualizar struct con campos adicionales del schema worker
- Actualizar dependencia edugo-infrastructure@mongodb/v0.12.1

Ticket: PT-003"
```

---

## Fase 7: Pull Request a Dev

### Paso 7.1: Push rama

```bash
git push -u origin feature/PT-003-homologar-material-assessment
```

### Paso 7.2: Crear PR

**Titulo:** `feat(mongodb): Usar coleccion material_assessment_worker`

**Descripcion:**

```markdown
## Descripcion

Actualiza api-mobile para usar la coleccion unificada `material_assessment_worker` en lugar de `material_assessment`.

## Dependencias

- Requiere: `edugo-infrastructure/mongodb@v0.12.1` (ya publicado)
- Requiere: Migracion de datos ejecutada en MongoDB

## Cambios

- [x] Nombre de coleccion actualizado en repository
- [x] Struct AssessmentDocument actualizado con nuevos campos
- [x] Dependencia de infra actualizada

## Checklist

- [x] `go build ./...` exitoso
- [x] `go test ./...` exitoso
- [x] `golangci-lint run` limpio
- [x] Test manual con datos migrados

## Notas de Deploy

1. Verificar que migracion de datos fue ejecutada en staging/prod
2. Deploy api-mobile despues de confirmar datos migrados
```

### Paso 7.3: Review y merge

---

## Checklist Final

- [ ] Rama feature creada
- [ ] Dependencia de infra actualizada
- [ ] Nombre de coleccion cambiado
- [ ] Struct actualizado
- [ ] Build exitoso
- [ ] Tests exitosos
- [ ] Lint limpio
- [ ] Test manual completado
- [ ] PR a dev creado
- [ ] PR a dev mergeado

---

**Siguiente tarea:** [../../04-homologar-material-summary/](../../04-homologar-material-summary/)
