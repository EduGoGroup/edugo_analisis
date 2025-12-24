# Plan: edugo-api-mobile - Endpoint PUT Materials

**Proyecto:** edugo-api-mobile
**Rama base:** dev
**Rama feature:** `feature/PT-007-material-update-endpoint`

---

## Fase 1: Preparacion

### Paso 1.1: Crear rama

```bash
cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile
git fetch origin
git checkout dev
git pull origin dev
git checkout -b feature/PT-007-material-update-endpoint
```

### Paso 1.2: Revisar estructura existente

```bash
# Revisar handler actual
cat internal/infrastructure/http/handler/material_handler.go

# Revisar service
cat internal/application/service/material_service.go

# Revisar repository
cat internal/domain/repository/material_repository.go
```

---

## Fase 2: Crear DTO de Request

### Paso 2.1: Agregar DTO

**Archivo:** `internal/application/dto/material_dto.go`

```go
// UpdateMaterialRequest representa los campos actualizables de un material
type UpdateMaterialRequest struct {
    Title       *string   `json:"title,omitempty" validate:"omitempty,min=1,max=255"`
    Description *string   `json:"description,omitempty" validate:"omitempty,max=1000"`
    Tags        *[]string `json:"tags,omitempty"`
    IsPublic    *bool     `json:"is_public,omitempty"`
    UnitID      *string   `json:"unit_id,omitempty" validate:"omitempty,uuid"`
    SubjectID   *string   `json:"subject_id,omitempty" validate:"omitempty,uuid"`
}
```

---

## Fase 3: Implementar Repository

### Paso 3.1: Agregar metodo en interfaz

**Archivo:** `internal/domain/repository/material_repository.go`

```go
type MaterialRepository interface {
    // ... existentes
    Update(ctx context.Context, material *entities.Material) error
}
```

### Paso 3.2: Implementar metodo

**Archivo:** `internal/infrastructure/persistence/postgres/repository/material_repository_impl.go`

```go
func (r *MaterialRepositoryImpl) Update(ctx context.Context, material *entities.Material) error {
    // Actualizar solo campos no nulos
    return r.db.WithContext(ctx).
        Model(material).
        Select("title", "description", "tags", "is_public", "unit_id", "subject_id", "updated_at").
        Updates(material).Error
}
```

---

## Fase 4: Implementar Service

### Paso 4.1: Agregar metodo en service

**Archivo:** `internal/application/service/material_service.go`

```go
func (s *MaterialService) UpdateMaterial(
    ctx context.Context,
    materialID string,
    req dto.UpdateMaterialRequest,
    userID string,
) (*dto.MaterialResponse, error) {
    // 1. Parsear ID
    id, err := uuid.Parse(materialID)
    if err != nil {
        return nil, errors.NewBadRequest("invalid material ID")
    }

    // 2. Obtener material existente
    material, err := s.materialRepo.FindByID(ctx, id)
    if err != nil {
        return nil, err
    }

    // 3. Verificar permisos (solo creador o admin puede actualizar)
    if material.CreatedBy.String() != userID {
        // Verificar si es admin
        isAdmin, err := s.checkAdminPermission(ctx, userID, material.UnitID)
        if err != nil || !isAdmin {
            return nil, errors.NewForbidden("not authorized to update this material")
        }
    }

    // 4. Aplicar cambios
    if req.Title != nil {
        material.Title = *req.Title
    }
    if req.Description != nil {
        material.Description = req.Description
    }
    if req.Tags != nil {
        material.Tags = *req.Tags
    }
    if req.IsPublic != nil {
        material.IsPublic = *req.IsPublic
    }
    if req.UnitID != nil {
        unitID, err := uuid.Parse(*req.UnitID)
        if err != nil {
            return nil, errors.NewBadRequest("invalid unit ID")
        }
        material.UnitID = unitID
    }
    if req.SubjectID != nil {
        subjectID, err := uuid.Parse(*req.SubjectID)
        if err != nil {
            return nil, errors.NewBadRequest("invalid subject ID")
        }
        material.SubjectID = &subjectID
    }

    material.UpdatedAt = time.Now()

    // 5. Guardar cambios
    if err := s.materialRepo.Update(ctx, material); err != nil {
        return nil, err
    }

    // 6. Mapear a response
    return s.mapToResponse(material), nil
}
```

---

## Fase 5: Implementar Handler

### Paso 5.1: Agregar metodo en handler

**Archivo:** `internal/infrastructure/http/handler/material_handler.go`

```go
// @Summary Update material
// @Description Updates an existing material's metadata
// @Tags materials
// @Accept json
// @Produce json
// @Param id path string true "Material ID" format(uuid)
// @Param request body dto.UpdateMaterialRequest true "Update data"
// @Success 200 {object} dto.MaterialResponse
// @Failure 400 {object} httpdto.ErrorResponse
// @Failure 403 {object} httpdto.ErrorResponse "Not authorized"
// @Failure 404 {object} httpdto.ErrorResponse "Material not found"
// @Security BearerAuth
// @Router /v1/materials/{id} [put]
func (h *MaterialHandler) UpdateMaterial(c *gin.Context) {
    materialID := c.Param("id")
    userID := c.GetString("user_id")

    var req dto.UpdateMaterialRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, httpdto.ErrorResponse{
            Error: "invalid request body",
            Code:  "INVALID_REQUEST",
        })
        return
    }

    // Validar request
    if err := h.validator.Struct(req); err != nil {
        c.JSON(http.StatusBadRequest, httpdto.ErrorResponse{
            Error: err.Error(),
            Code:  "VALIDATION_ERROR",
        })
        return
    }

    material, err := h.materialService.UpdateMaterial(
        c.Request.Context(),
        materialID,
        req,
        userID,
    )
    if err != nil {
        _ = c.Error(err)
        return
    }

    h.logger.Info("material updated",
        "material_id", materialID,
        "user_id", userID,
    )

    c.JSON(http.StatusOK, material)
}
```

### Paso 5.2: Agregar ruta

**Archivo:** `internal/infrastructure/http/router/router.go`

```go
materials := v1.Group("/materials")
{
    materials.POST("", materialHandler.CreateMaterial)
    materials.GET("", materialHandler.ListMaterials)
    materials.GET("/:id", materialHandler.GetMaterial)
    materials.PUT("/:id", materialHandler.UpdateMaterial)  // NUEVO
    materials.GET("/:id/versions", materialHandler.GetMaterialVersions)
    // ... resto de rutas
}
```

---

## Fase 6: Compilar y Test

### Paso 6.1: Compilar

```bash
go build ./...
```

### Paso 6.2: Regenerar swagger

```bash
swag init -g cmd/main.go -o docs
```

### Paso 6.3: Ejecutar tests

```bash
go test ./... -v
go test ./... -cover
```

### Paso 6.4: Lint

```bash
golangci-lint run ./...
```

---

## Fase 7: Agregar Tests

### Paso 7.1: Tests de handler

**Archivo:** `internal/infrastructure/http/handler/material_handler_test.go`

Agregar tests:

```go
func TestUpdateMaterial(t *testing.T) {
    tests := []struct {
        name           string
        materialID     string
        userID         string
        request        dto.UpdateMaterialRequest
        expectedStatus int
    }{
        {
            name:           "update title success",
            materialID:     "valid-uuid",
            userID:         "creator-user-id",
            request:        dto.UpdateMaterialRequest{Title: ptr("New Title")},
            expectedStatus: http.StatusOK,
        },
        {
            name:           "update by non-creator forbidden",
            materialID:     "valid-uuid",
            userID:         "other-user-id",
            request:        dto.UpdateMaterialRequest{Title: ptr("New Title")},
            expectedStatus: http.StatusForbidden,
        },
        {
            name:           "material not found",
            materialID:     "non-existent-uuid",
            userID:         "any-user",
            request:        dto.UpdateMaterialRequest{},
            expectedStatus: http.StatusNotFound,
        },
        {
            name:           "invalid material ID",
            materialID:     "invalid-id",
            userID:         "any-user",
            request:        dto.UpdateMaterialRequest{},
            expectedStatus: http.StatusBadRequest,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Implementar test
        })
    }
}

func ptr(s string) *string {
    return &s
}
```

---

## Fase 8: Commit y PR

### Paso 8.1: Commit

```bash
git add .
git commit -m "feat(materials): agregar endpoint PUT para actualizar materiales

- Crear DTO UpdateMaterialRequest
- Agregar metodo Update en repository
- Implementar UpdateMaterial en service
- Implementar handler con validaciones
- Verificar permisos (creador o admin)
- Agregar ruta PUT /v1/materials/:id
- Actualizar swagger
- Agregar tests

Ticket: PT-007"
```

### Paso 8.2: Push y crear PR

```bash
git push -u origin feature/PT-007-material-update-endpoint
```

**Titulo PR:** `feat(materials): Agregar endpoint PUT para actualizar materiales`

**Descripcion:**

```markdown
## Descripcion

Implementa el endpoint faltante para actualizar materiales:

**PUT /v1/materials/:id**

## Campos Actualizables

- `title` - Titulo del material
- `description` - Descripcion
- `tags` - Etiquetas
- `is_public` - Visibilidad
- `unit_id` - Mover a otra unidad
- `subject_id` - Cambiar materia

## Permisos

- Solo el creador del material puede actualizarlo
- Administradores tambien pueden actualizar

## Cambios

- [x] DTO UpdateMaterialRequest creado
- [x] Repository.Update implementado
- [x] Service.UpdateMaterial implementado
- [x] Handler con validaciones
- [x] Ruta agregada
- [x] Swagger actualizado
- [x] Tests agregados

## Checklist

- [x] `go build ./...` exitoso
- [x] `swag init` ejecutado
- [x] `go test ./...` exitoso
- [x] `golangci-lint run` limpio
```

---

## Checklist Final

- [ ] Rama feature creada
- [ ] DTO UpdateMaterialRequest creado
- [ ] Repository.Update implementado
- [ ] Service.UpdateMaterial implementado
- [ ] Handler implementado con validaciones
- [ ] Verificacion de permisos
- [ ] Ruta agregada
- [ ] Swagger regenerado
- [ ] Tests agregados
- [ ] Build exitoso
- [ ] Tests exitosos
- [ ] Lint limpio
- [ ] PR a dev creado
- [ ] PR a dev mergeado

---

**Siguiente tarea:** [../../08-worker-fase1/](../../08-worker-fase1/)
