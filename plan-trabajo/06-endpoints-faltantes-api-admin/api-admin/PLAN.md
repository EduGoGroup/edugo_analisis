# Plan: edugo-api-administracion - Endpoints Faltantes

**Proyecto:** edugo-api-administracion
**Rama base:** dev
**Rama feature:** `feature/PT-006-endpoints-faltantes-subjects-guardians`

---

## Fase 1: Preparacion

### Paso 1.1: Crear rama

```bash
cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion
git fetch origin
git checkout dev
git pull origin dev
git checkout -b feature/PT-006-endpoints-faltantes-subjects-guardians
```

---

## Fase 2: Implementar Subjects CRUD

### Paso 2.1: GET /v1/subjects/:id

**Archivo:** `internal/infrastructure/http/handler/subject_handler.go`

Agregar metodo:

```go
// @Summary Get subject by ID
// @Description Retrieves a subject by its unique identifier
// @Tags subjects
// @Accept json
// @Produce json
// @Param id path string true "Subject ID" format(uuid)
// @Success 200 {object} dto.SubjectResponse
// @Failure 404 {object} httpdto.ErrorResponse
// @Router /v1/subjects/{id} [get]
func (h *SubjectHandler) GetSubject(c *gin.Context) {
    id := c.Param("id")

    subject, err := h.subjectService.GetSubject(c.Request.Context(), id)
    if err != nil {
        _ = c.Error(err)
        return
    }

    c.JSON(http.StatusOK, subject)
}
```

**Router:** `internal/infrastructure/http/router/router.go`

```go
subjects := v1.Group("/subjects")
{
    subjects.POST("", subjectHandler.CreateSubject)
    subjects.GET("/:id", subjectHandler.GetSubject)  // NUEVO
    subjects.PATCH("/:id", subjectHandler.UpdateSubject)
}
```

### Paso 2.2: GET /v1/subjects (Listar)

**Repositorio:** `internal/domain/repository/subject_repository.go`

Agregar metodo a la interfaz:

```go
type SubjectRepository interface {
    // ... existentes
    FindAll(ctx context.Context) ([]*entities.Subject, error)
    FindBySchoolID(ctx context.Context, schoolID string) ([]*entities.Subject, error)
}
```

**Implementacion:** `internal/infrastructure/persistence/postgres/repository/subject_repository_impl.go`

```go
func (r *SubjectRepositoryImpl) FindAll(ctx context.Context) ([]*entities.Subject, error) {
    var subjects []*entities.Subject
    if err := r.db.WithContext(ctx).Find(&subjects).Error; err != nil {
        return nil, err
    }
    return subjects, nil
}

func (r *SubjectRepositoryImpl) FindBySchoolID(ctx context.Context, schoolID string) ([]*entities.Subject, error) {
    var subjects []*entities.Subject
    if err := r.db.WithContext(ctx).Where("school_id = ?", schoolID).Find(&subjects).Error; err != nil {
        return nil, err
    }
    return subjects, nil
}
```

**Service:** `internal/application/service/subject_service.go`

```go
func (s *SubjectService) ListSubjects(ctx context.Context, schoolID string) ([]dto.SubjectResponse, error) {
    var subjects []*entities.Subject
    var err error

    if schoolID != "" {
        subjects, err = s.subjectRepo.FindBySchoolID(ctx, schoolID)
    } else {
        subjects, err = s.subjectRepo.FindAll(ctx)
    }

    if err != nil {
        return nil, err
    }

    responses := make([]dto.SubjectResponse, len(subjects))
    for i, s := range subjects {
        responses[i] = dto.SubjectResponse{
            ID:        s.ID.String(),
            Name:      s.Name,
            Code:      s.Code,
            SchoolID:  s.SchoolID.String(),
            CreatedAt: s.CreatedAt,
            UpdatedAt: s.UpdatedAt,
        }
    }

    return responses, nil
}
```

**Handler:** `internal/infrastructure/http/handler/subject_handler.go`

```go
// @Summary List subjects
// @Description Lists all subjects, optionally filtered by school
// @Tags subjects
// @Accept json
// @Produce json
// @Param school_id query string false "Filter by school ID" format(uuid)
// @Success 200 {array} dto.SubjectResponse
// @Router /v1/subjects [get]
func (h *SubjectHandler) ListSubjects(c *gin.Context) {
    schoolID := c.Query("school_id")

    subjects, err := h.subjectService.ListSubjects(c.Request.Context(), schoolID)
    if err != nil {
        _ = c.Error(err)
        return
    }

    c.JSON(http.StatusOK, subjects)
}
```

**Router:** Agregar ruta

```go
subjects.GET("", subjectHandler.ListSubjects)  // NUEVO
```

### Paso 2.3: DELETE /v1/subjects/:id

**Service:** `internal/application/service/subject_service.go`

```go
func (s *SubjectService) DeleteSubject(ctx context.Context, id string) error {
    subjectID, err := uuid.Parse(id)
    if err != nil {
        return errors.NewBadRequest("invalid subject ID")
    }

    return s.subjectRepo.Delete(ctx, subjectID)
}
```

**Handler:**

```go
// @Summary Delete subject
// @Description Soft deletes a subject
// @Tags subjects
// @Param id path string true "Subject ID" format(uuid)
// @Success 204 "No Content"
// @Failure 404 {object} httpdto.ErrorResponse
// @Router /v1/subjects/{id} [delete]
func (h *SubjectHandler) DeleteSubject(c *gin.Context) {
    id := c.Param("id")

    err := h.subjectService.DeleteSubject(c.Request.Context(), id)
    if err != nil {
        _ = c.Error(err)
        return
    }

    c.Status(http.StatusNoContent)
}
```

**Router:**

```go
subjects.DELETE("/:id", subjectHandler.DeleteSubject)  // NUEVO
```

---

## Fase 3: Implementar Guardian Relations

### Paso 3.1: PUT /v1/guardian-relations/:id

**DTO:** `internal/application/dto/guardian_dto.go`

Agregar request:

```go
type UpdateGuardianRelationRequest struct {
    RelationshipType *string `json:"relationship_type,omitempty"`
    Notes            *string `json:"notes,omitempty"`
    IsActive         *bool   `json:"is_active,omitempty"`
}
```

**Repository:** `internal/domain/repository/guardian_repository.go`

Agregar metodo:

```go
type GuardianRepository interface {
    // ... existentes
    Update(ctx context.Context, relation *entities.GuardianRelation) error
}
```

**Implementacion:** `internal/infrastructure/persistence/postgres/repository/guardian_repository_impl.go`

```go
func (r *GuardianRepositoryImpl) Update(ctx context.Context, relation *entities.GuardianRelation) error {
    return r.db.WithContext(ctx).Save(relation).Error
}
```

**Service:** `internal/application/service/guardian_service.go`

```go
func (s *GuardianService) UpdateGuardianRelation(
    ctx context.Context,
    id string,
    req dto.UpdateGuardianRelationRequest,
) (*dto.GuardianRelationResponse, error) {
    relationID, err := uuid.Parse(id)
    if err != nil {
        return nil, errors.NewBadRequest("invalid relation ID")
    }

    relation, err := s.guardianRepo.FindByID(ctx, relationID)
    if err != nil {
        return nil, err
    }

    // Actualizar campos si fueron enviados
    if req.RelationshipType != nil {
        relation.RelationshipType = *req.RelationshipType
    }
    if req.Notes != nil {
        relation.Notes = req.Notes
    }
    if req.IsActive != nil {
        relation.IsActive = *req.IsActive
    }

    relation.UpdatedAt = time.Now()

    if err := s.guardianRepo.Update(ctx, relation); err != nil {
        return nil, err
    }

    return s.mapToResponse(relation), nil
}
```

**Handler:** `internal/infrastructure/http/handler/guardian_handler.go`

```go
// @Summary Update guardian relation
// @Description Updates an existing guardian-student relationship
// @Tags guardian-relations
// @Accept json
// @Produce json
// @Param id path string true "Relation ID" format(uuid)
// @Param request body dto.UpdateGuardianRelationRequest true "Update data"
// @Success 200 {object} dto.GuardianRelationResponse
// @Failure 400 {object} httpdto.ErrorResponse
// @Failure 404 {object} httpdto.ErrorResponse
// @Router /v1/guardian-relations/{id} [put]
func (h *GuardianHandler) UpdateGuardianRelation(c *gin.Context) {
    id := c.Param("id")
    var req dto.UpdateGuardianRelationRequest

    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, httpdto.ErrorResponse{
            Error: "invalid request body",
            Code:  "INVALID_REQUEST",
        })
        return
    }

    relation, err := h.guardianService.UpdateGuardianRelation(c.Request.Context(), id, req)
    if err != nil {
        _ = c.Error(err)
        return
    }

    h.logger.Info("guardian relation updated", "relation_id", id)
    c.JSON(http.StatusOK, relation)
}
```

**Router:**

```go
guardianRelations.PUT("/:id", guardianHandler.UpdateGuardianRelation)  // NUEVO
```

### Paso 3.2: DELETE /v1/guardian-relations/:id

**Repository:** Agregar metodo Delete (soft delete)

```go
func (r *GuardianRepositoryImpl) Delete(ctx context.Context, id uuid.UUID) error {
    return r.db.WithContext(ctx).
        Model(&entities.GuardianRelation{}).
        Where("id = ?", id).
        Update("deleted_at", time.Now()).Error
}
```

**Service:**

```go
func (s *GuardianService) DeleteGuardianRelation(ctx context.Context, id string) error {
    relationID, err := uuid.Parse(id)
    if err != nil {
        return errors.NewBadRequest("invalid relation ID")
    }

    return s.guardianRepo.Delete(ctx, relationID)
}
```

**Handler:**

```go
// @Summary Delete guardian relation
// @Description Soft deletes a guardian-student relationship
// @Tags guardian-relations
// @Param id path string true "Relation ID" format(uuid)
// @Success 204 "No Content"
// @Failure 404 {object} httpdto.ErrorResponse
// @Router /v1/guardian-relations/{id} [delete]
func (h *GuardianHandler) DeleteGuardianRelation(c *gin.Context) {
    id := c.Param("id")

    err := h.guardianService.DeleteGuardianRelation(c.Request.Context(), id)
    if err != nil {
        _ = c.Error(err)
        return
    }

    h.logger.Info("guardian relation deleted", "relation_id", id)
    c.Status(http.StatusNoContent)
}
```

**Router:**

```go
guardianRelations.DELETE("/:id", guardianHandler.DeleteGuardianRelation)  // NUEVO
```

---

## Fase 4: Compilar y Test

### Paso 4.1: Compilar

```bash
go build ./...
```

### Paso 4.2: Regenerar swagger

```bash
swag init -g cmd/main.go -o docs
```

### Paso 4.3: Ejecutar tests

```bash
go test ./... -v
go test ./... -cover
```

### Paso 4.4: Lint

```bash
golangci-lint run ./...
```

---

## Fase 5: Agregar Tests

### Paso 5.1: Tests de Subjects

Crear archivo: `internal/infrastructure/http/handler/subject_handler_test.go`

```go
func TestGetSubject(t *testing.T) {
    // Test obtener subject existente
    // Test subject no encontrado
    // Test ID invalido
}

func TestListSubjects(t *testing.T) {
    // Test listar todos
    // Test filtrar por school_id
    // Test lista vacia
}

func TestDeleteSubject(t *testing.T) {
    // Test delete exitoso
    // Test subject no encontrado
}
```

### Paso 5.2: Tests de Guardian Relations

Crear archivo: `internal/infrastructure/http/handler/guardian_handler_test.go`

```go
func TestUpdateGuardianRelation(t *testing.T) {
    // Test update exitoso
    // Test relacion no encontrada
    // Test request invalido
}

func TestDeleteGuardianRelation(t *testing.T) {
    // Test delete exitoso
    // Test relacion no encontrada
}
```

---

## Fase 6: Commit y PR

### Paso 6.1: Commit

```bash
git add .
git commit -m "feat(api): completar CRUD de Subjects y Guardian Relations

Subjects:
- Agregar GET /v1/subjects/:id
- Agregar GET /v1/subjects (con filtro por school_id)
- Agregar DELETE /v1/subjects/:id

Guardian Relations:
- Agregar PUT /v1/guardian-relations/:id
- Agregar DELETE /v1/guardian-relations/:id

- Actualizar swagger
- Agregar tests

Ticket: PT-006"
```

### Paso 6.2: Push y crear PR

```bash
git push -u origin feature/PT-006-endpoints-faltantes-subjects-guardians
```

**Titulo PR:** `feat(api): Completar CRUD de Subjects y Guardian Relations`

**Descripcion:**

```markdown
## Descripcion

Completa los endpoints faltantes de CRUD:

### Subjects
- [x] GET `/v1/subjects/:id` - Obtener materia
- [x] GET `/v1/subjects` - Listar materias (filtro por school_id)
- [x] DELETE `/v1/subjects/:id` - Eliminar materia

### Guardian Relations
- [x] PUT `/v1/guardian-relations/:id` - Actualizar relacion
- [x] DELETE `/v1/guardian-relations/:id` - Eliminar relacion

## Cambios

- Nuevos metodos en repositories
- Nuevos metodos en services
- Nuevos handlers
- Rutas agregadas
- Swagger actualizado
- Tests agregados

## Checklist

- [x] `go build ./...` exitoso
- [x] `swag init` ejecutado
- [x] `go test ./...` exitoso
- [x] `golangci-lint run` limpio
```

---

## Checklist Final

- [ ] Rama feature creada
- [ ] GET /v1/subjects/:id implementado
- [ ] GET /v1/subjects implementado
- [ ] DELETE /v1/subjects/:id implementado
- [ ] PUT /v1/guardian-relations/:id implementado
- [ ] DELETE /v1/guardian-relations/:id implementado
- [ ] Swagger regenerado
- [ ] Tests agregados
- [ ] Build exitoso
- [ ] Tests exitosos
- [ ] Lint limpio
- [ ] PR a dev creado
- [ ] PR a dev mergeado

---

**Siguiente tarea:** [../../07-endpoints-faltantes-api-mobile/](../../07-endpoints-faltantes-api-mobile/)
