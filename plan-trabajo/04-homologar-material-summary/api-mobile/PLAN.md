# Plan: edugo-api-mobile - Homologar material_summary

**Proyecto:** edugo-api-mobile
**Rama base:** dev
**Rama feature:** `feature/PT-004-homologar-material-summary`

**IMPORTANTE:** Ejecutar DESPUES de migrar datos en MongoDB

---

## Fase 1: Preparacion

### Paso 1.1: Crear rama

```bash
cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile
git fetch origin
git checkout dev
git pull origin dev
git checkout -b feature/PT-004-homologar-material-summary
```

### Paso 1.2: Verificar archivo a modificar

```bash
cat internal/infrastructure/persistence/mongodb/repository/summary_repository_impl.go
```

---

## Fase 2: Modificar Repository

### Paso 2.1: Cambiar nombre de coleccion

Archivo: `internal/infrastructure/persistence/mongodb/repository/summary_repository_impl.go`

**ANTES (linea 20 aprox):**
```go
collection: db.Collection("material_summaries"),
```

**DESPUES:**
```go
collection: db.Collection("material_summary"),
```

### Paso 2.2: Actualizar metodo Save

**ANTES (lineas 24-37 aprox):**
```go
doc := bson.M{
    "material_id":  summary.MaterialID.String(),
    "main_ideas":   summary.MainIdeas,
    "key_concepts": summary.KeyConcepts,
    "sections":     summary.Sections,
    "glossary":     summary.Glossary,
    "created_at":   time.Now(),
}
```

**DESPUES:**
```go
doc := bson.M{
    "material_id":        summary.MaterialID.String(),
    "summary":            generateSummaryText(summary.MainIdeas),
    "key_points":         summary.MainIdeas,
    "language":           "es", // Default espanol
    "word_count":         calculateWordCount(summary.MainIdeas),
    "version":            1,
    "ai_model":           "api-mobile",
    "processing_time_ms": 0,
    "metadata": bson.M{
        "key_concepts": summary.KeyConcepts,
        "sections":     summary.Sections,
        "glossary":     summary.Glossary,
    },
    "created_at": time.Now(),
    "updated_at": time.Now(),
}
```

### Paso 2.3: Agregar funciones helper

```go
func generateSummaryText(mainIdeas []string) string {
    if len(mainIdeas) == 0 {
        return ""
    }
    return strings.Join(mainIdeas, ". ") + "."
}

func calculateWordCount(mainIdeas []string) int {
    if len(mainIdeas) == 0 {
        return 0
    }
    text := strings.Join(mainIdeas, " ")
    return len(strings.Fields(text))
}
```

### Paso 2.4: Actualizar metodo FindByMaterialID

Adaptar para leer el nuevo schema:

```go
func (r *SummaryRepositoryImpl) FindByMaterialID(ctx context.Context, materialID uuid.UUID) (*domain.Summary, error) {
    var doc struct {
        MaterialID      string    `bson:"material_id"`
        Summary         string    `bson:"summary"`
        KeyPoints       []string  `bson:"key_points"`
        Language        string    `bson:"language"`
        WordCount       int       `bson:"word_count"`
        Version         int       `bson:"version"`
        AIModel         string    `bson:"ai_model"`
        ProcessingTimeMs int      `bson:"processing_time_ms"`
        Metadata        bson.M    `bson:"metadata"`
        CreatedAt       time.Time `bson:"created_at"`
        UpdatedAt       time.Time `bson:"updated_at"`
    }

    filter := bson.M{"material_id": materialID.String()}
    err := r.collection.FindOne(ctx, filter).Decode(&doc)
    if err != nil {
        if err == mongo.ErrNoDocuments {
            return nil, nil
        }
        return nil, err
    }

    // Mapear a domain.Summary
    summary := &domain.Summary{
        MaterialID:  materialID,
        MainIdeas:   doc.KeyPoints, // Mapear key_points a MainIdeas
        KeyConcepts: extractMetadataField(doc.Metadata, "key_concepts"),
        Sections:    extractMetadataField(doc.Metadata, "sections"),
        Glossary:    extractMetadataField(doc.Metadata, "glossary"),
        CreatedAt:   doc.CreatedAt,
    }

    return summary, nil
}

func extractMetadataField(metadata bson.M, field string) []string {
    if metadata == nil {
        return nil
    }
    if val, ok := metadata[field]; ok {
        if arr, ok := val.(primitive.A); ok {
            result := make([]string, len(arr))
            for i, v := range arr {
                result[i], _ = v.(string)
            }
            return result
        }
    }
    return nil
}
```

---

## Fase 3: Compilar y Test

### Paso 3.1: Compilar

```bash
go build ./...
```

### Paso 3.2: Ejecutar tests

```bash
go test ./... -v
go test ./... -cover
```

### Paso 3.3: Lint

```bash
golangci-lint run ./...
```

---

## Fase 4: Test Manual

### Paso 4.1: Levantar ambiente local

```bash
# Asegurar que MongoDB tiene datos migrados
docker-compose up -d

# Ejecutar API
go run cmd/main.go
```

### Paso 4.2: Probar endpoints de summary

```bash
# Obtener un summary
curl -X GET http://localhost:8081/v1/summaries/{material_id}

# Verificar respuesta correcta
```

---

## Fase 5: Commit y PR

### Paso 5.1: Commit

```bash
git add .
git commit -m "feat(mongodb): usar coleccion material_summary (singular)

- Cambiar nombre de coleccion material_summaries -> material_summary
- Adaptar struct para schema canonico de worker
- Agregar funciones helper para transformacion

Ticket: PT-004"
```

### Paso 5.2: Push y crear PR

```bash
git push -u origin feature/PT-004-homologar-material-summary
```

**Titulo PR:** `feat(mongodb): Usar coleccion material_summary`

**Descripcion:**

```markdown
## Descripcion

Actualiza api-mobile para usar la coleccion `material_summary` (singular)
en lugar de `material_summaries` (plural).

## Dependencias

- Requiere: Migracion de datos ejecutada en MongoDB

## Cambios

- [x] Nombre de coleccion actualizado
- [x] Metodo Save adaptado al nuevo schema
- [x] Metodo FindByMaterialID adaptado

## Checklist

- [x] `go build ./...` exitoso
- [x] `go test ./...` exitoso
- [x] `golangci-lint run` limpio
- [x] Test manual completado

## Notas de Deploy

1. Ejecutar script de migracion de datos primero
2. Deploy api-mobile despues de confirmar datos migrados
3. Eliminar coleccion antigua despues de validar
```

---

## Checklist Final

- [ ] Rama feature creada
- [ ] Nombre de coleccion cambiado
- [ ] Metodo Save adaptado
- [ ] Metodo FindByMaterialID adaptado
- [ ] Funciones helper agregadas
- [ ] Build exitoso
- [ ] Tests exitosos
- [ ] Lint limpio
- [ ] Test manual completado
- [ ] PR a dev creado
- [ ] PR a dev mergeado

---

**Siguiente tarea:** [../../05-eliminar-campos-sin-uso/](../../05-eliminar-campos-sin-uso/)
