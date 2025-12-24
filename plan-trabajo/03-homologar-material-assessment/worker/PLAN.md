# Plan: edugo-worker - Verificar material_assessment

**Proyecto:** edugo-worker
**Estado:** Solo verificacion (sin cambios requeridos)

---

## Verificacion

El worker ya usa la coleccion correcta `material_assessment_worker`. Este documento solo documenta la verificacion.

### Paso 1: Confirmar uso de coleccion correcta

```bash
cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker

# Buscar referencias a material_assessment
grep -r "material_assessment" --include="*.go" .
```

**Resultado esperado:**
- Solo referencias a `material_assessment_worker`
- Repository en: `internal/infrastructure/persistence/mongodb/repository/material_assessment_repository.go`

### Paso 2: Verificar entity

```bash
# La entidad deberia usar el schema completo
cat internal/domain/entity/material_assessment.go
```

**Campos esperados:**
- material_id
- questions
- total_questions
- total_points
- version
- ai_model
- processing_time_ms
- metadata
- created_at
- updated_at

### Paso 3: Actualizar dependencia de infra (despues del release)

```bash
cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker

# Actualizar a nueva version de infra
go get github.com/EduGoGroup/edugo-infrastructure/mongodb@v0.12.1
go mod tidy
```

### Paso 4: Verificar compilacion

```bash
go build ./...
go test ./... -v
```

---

## Sin Cambios de Codigo

Este proyecto NO requiere cambios de codigo. Solo:
1. Actualizar dependencia de infra
2. Verificar compilacion

---

**Siguiente paso:** [../api-mobile/PLAN.md](../api-mobile/PLAN.md) - Modificar api-mobile
