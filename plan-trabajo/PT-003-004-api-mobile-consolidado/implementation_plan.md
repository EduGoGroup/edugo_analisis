# Plan: Consolidar PT-003 y PT-004 en API-Mobile

**Proyecto:** edugo-api-mobile  
**Rama base:** dev  
**Rama feature:** `feature/PT-003-004-homologar-colecciones-mongodb`  
**Fecha:** 2025-12-23

---

## Descripción

Este plan consolida las tareas PT-003 (homologar material_assessment) y PT-004 (homologar material_summary) en una sola rama, modificando los repositorios MongoDB de api-mobile para usar los nombres de colección correctos.

> [!IMPORTANT]
> **Estado de Releases en Infra:** Actualmente las últimas versiones son `postgres/v0.13.0` y `mongodb/v0.11.0`. El README del plan de trabajo menciona que se requieren `mongodb/v0.13.0`, pero ese release aún no existe. **Procederemos sin actualizar la dependencia de infra** ya que los cambios son solo de nombre de colección.

---

## Revisión del Usuario Requerida

> [!WARNING]  
> Los releases de infra mencionados en los planes originales (`mongodb/v0.12.1` y `mongodb/v0.13.0`) **NO existen todavía**. La última versión de mongodb es `v0.11.0`.

**Pregunta:** ¿Desea que cree los releases faltantes de infra antes de proceder, o continuamos solo con los cambios de nombre de colección en api-mobile?

---

## Cambios Propuestos

### Repositorio MongoDB Assessment

#### [MODIFY] [assessment_document_repository.go](file:///Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile/internal/infrastructure/persistence/mongodb/repository/assessment_document_repository.go)

**Cambio 1:** Línea 83 - Cambiar nombre de colección

```diff
-		collection: db.Collection("material_assessment"),
+		collection: db.Collection("material_assessment_worker"),
```

**Cambio 2:** Agregar campos adicionales al struct `AssessmentDocument` (líneas 31-40):

```diff
 type AssessmentDocument struct {
 	ID         primitive.ObjectID `bson:"_id,omitempty"`
 	MaterialID string             `bson:"material_id"`
 	Title      string             `bson:"title"`
 	Questions  []Question         `bson:"questions"`
+	TotalQuestions   int          `bson:"total_questions"`
+	TotalPoints      int          `bson:"total_points"`
+	AIModel          string       `bson:"ai_model"`
+	ProcessingTimeMs int          `bson:"processing_time_ms"`
 	Metadata   Metadata           `bson:"metadata"`
 	Version    int                `bson:"version"`
 	CreatedAt  time.Time          `bson:"created_at"`
 	UpdatedAt  time.Time          `bson:"updated_at"`
 }
```

---

#### [MODIFY] [assessment_document_repository_integration_test.go](file:///Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile/internal/infrastructure/persistence/mongodb/repository/assessment_document_repository_integration_test.go)

**Cambio:** Línea 54 - Actualizar nombre de colección en cleanup del test

```diff
-	_ = s.MongoDB.Collection("material_assessment").Drop(ctx)
+	_ = s.MongoDB.Collection("material_assessment_worker").Drop(ctx)
```

---

### Repositorio MongoDB Summary

#### [MODIFY] [summary_repository_impl.go](file:///Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile/internal/infrastructure/persistence/mongodb/repository/summary_repository_impl.go)

**Cambio 1:** Línea 20 - Cambiar nombre de colección

```diff
-		collection: db.Collection("material_summaries"),
+		collection: db.Collection("material_summary"),
```

**Cambio 2:** Actualizar método `Save` para usar el nuevo schema (líneas 24-37):

```diff
 func (r *mongoSummaryRepository) Save(ctx context.Context, summary *repository.MaterialSummary) error {
 	doc := bson.M{
-		"material_id":  summary.MaterialID.String(),
-		"main_ideas":   summary.MainIdeas,
-		"key_concepts": summary.KeyConcepts,
-		"sections":     summary.Sections,
-		"glossary":     summary.Glossary,
-		"created_at":   time.Now(),
+		"material_id":        summary.MaterialID.String(),
+		"summary":            generateSummaryText(summary.MainIdeas),
+		"key_points":         summary.MainIdeas,
+		"language":           "es",
+		"word_count":         calculateWordCount(summary.MainIdeas),
+		"version":            1,
+		"ai_model":           "api-mobile",
+		"processing_time_ms": 0,
+		"metadata": bson.M{
+			"key_concepts": summary.KeyConcepts,
+			"sections":     summary.Sections,
+			"glossary":     summary.Glossary,
+		},
+		"created_at": time.Now(),
+		"updated_at": time.Now(),
 	}
```

**Cambio 3:** Agregar funciones helper al final del archivo:

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

**Cambio 4:** Agregar import de `strings` al inicio del archivo.

---

## Flujo de Trabajo

### 1. Preparación Git

```bash
cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile
git fetch origin
git checkout dev
git pull origin dev
git checkout -b feature/PT-003-004-homologar-colecciones-mongodb
```

### 2. Implementar Cambios

- Modificar `assessment_document_repository.go`
- Modificar `assessment_document_repository_integration_test.go`
- Modificar `summary_repository_impl.go`

### 3. Verificación

```bash
# Compilar
go build ./...

# Tests unitarios
go test ./... -v -count=1

# Lint
golangci-lint run ./...
```

### 4. Commit y PR

```bash
git add .
git commit -m "feat(mongodb): homologar colecciones assessment y summary

- Cambiar material_assessment -> material_assessment_worker
- Cambiar material_summaries -> material_summary
- Agregar campos adicionales al struct AssessmentDocument
- Adaptar método Save de summary al nuevo schema
- Actualizar test de integración

Tickets: PT-003, PT-004"

git push -u origin feature/PT-003-004-homologar-colecciones-mongodb
```

---

## Plan de Verificación

### Tests Automatizados

**Comando para ejecutar tests:**
```bash
cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile
go build ./...
go test ./... -v -count=1
golangci-lint run ./...
```

**Tests de integración (requieren Docker):**
```bash
go test ./internal/infrastructure/persistence/mongodb/repository/... -tags=integration -v
```

### Verificación Manual

> [!NOTE]
> La verificación manual completa requiere tener MongoDB con los datos migrados. Por ahora, verificaremos que el código compila y pasa los tests unitarios.

1. Verificar que `go build ./...` compila sin errores
2. Verificar que `go test ./...` pasa todos los tests
3. Verificar que `golangci-lint run ./...` no reporta errores críticos

---

## Checklist

- [ ] Rama feature creada desde dev
- [ ] Colección `material_assessment` → `material_assessment_worker`
- [ ] Struct `AssessmentDocument` actualizado con nuevos campos
- [ ] Test de integración actualizado
- [ ] Colección `material_summaries` → `material_summary`
- [ ] Método `Save` de summary adaptado al nuevo schema
- [ ] Funciones helper agregadas
- [ ] `go build ./...` exitoso
- [ ] `go test ./...` exitoso
- [ ] `golangci-lint run` limpio
- [ ] PR a dev creado

---

## Estructura de Archivos

```
edugo-api-mobile/internal/infrastructure/persistence/mongodb/repository/
├── assessment_document_repository.go           # Modificar
├── assessment_document_repository_integration_test.go  # Modificar
└── summary_repository_impl.go                  # Modificar
```
