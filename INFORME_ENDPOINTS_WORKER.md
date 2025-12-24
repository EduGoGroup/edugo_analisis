# INFORME DETALLADO: Endpoints y Worker - Ecosistema EduGo

**Fecha:** 2025-12-23
**Autor:** Análisis Arquitectura EduGo
**Versión:** 1.0

---

## TABLA DE CONTENIDOS

1. [Resumen Ejecutivo](#resumen-ejecutivo)
2. [Estado Actual del Ecosistema](#estado-actual-del-ecosistema)
3. [Endpoints Faltantes en APIs](#endpoints-faltantes-en-apis)
4. [Conexión Worker-APIs](#conexión-worker-apis)
5. [Processors a Completar en Worker](#processors-a-completar-en-worker)
6. [Integraciones Externas Pendientes](#integraciones-externas-pendientes)
7. [Endpoints que Dependen del Worker](#endpoints-que-dependen-del-worker)
8. [Timeline Sugerido](#timeline-sugerido)

---

## RESUMEN EJECUTIVO

### Estado General

El ecosistema EduGo se encuentra en diferentes estados de madurez:

- **API Admin**: ✅ Funcional, necesita completar CRUD de Subjects y Guardian Relations
- **API Mobile**: ✅ Funcional, falta endpoint UPDATE de materiales
- **Worker**: 🚧 En desarrollo activo (Fase 1 de 4), con funcionalidad core simulada

### Hallazgos Críticos

1. **Worker NO está procesando eventos realmente** - Solo hace logging
2. **Colecciones MongoDB duplicadas** - `material_assessment` (api-mobile) vs `material_assessment_worker` (worker)
3. **Procesamiento IA completamente simulado** - PDF extraction, resumen y quiz son hardcoded
4. **CRUD incompletos** - Subjects y Guardian Relations faltan endpoints
5. **2 processors sin implementar** - AssessmentAttempt y StudentEnrolled solo tienen logging

### Que Puede Avanzar Frontend AHORA

#### Funcionalidades Disponibles
- ✅ **Autenticación completa** (login, logout, refresh, verify)
- ✅ **Gestión de escuelas** (CRUD completo + búsqueda por código)
- ✅ **Unidades académicas** (CRUD + árbol jerárquico + filtros)
- ✅ **Memberships** (CRUD + expiración + consultas)
- ✅ **Usuarios** (CRUD completo)
- ✅ **Materiales** (crear, listar, obtener, versiones, upload/download URLs)
- ✅ **Evaluaciones** (obtener quiz, crear intento, ver resultados, historial)
- ✅ **Progreso** (upsert de progreso de usuario)

#### Funcionalidades Bloqueadas/Limitadas
- ⚠️ **Resumen de materiales** - Endpoint existe pero datos son mock del worker
- ⚠️ **Quiz de materiales** - Endpoint existe pero preguntas son mock del worker
- ❌ **Subjects GET/DELETE** - Endpoints no existen
- ❌ **Guardian Relations UPDATE/DELETE** - Endpoints no existen
- ❌ **Material UPDATE** - Endpoint no existe en api-mobile

---

## ESTADO ACTUAL DEL ECOSISTEMA

### API Admin - 46 Endpoints

#### Auth (6 endpoints) ✅
- POST `/v1/auth/login` - Login con credenciales
- POST `/v1/auth/logout` - Logout y revocación de token
- POST `/v1/auth/refresh` - Refresh token
- POST `/v1/auth/switch-context` - Cambiar contexto (school/unit)
- POST `/v1/auth/verify` - Verificar token único
- POST `/v1/auth/verify-bulk` - Verificar múltiples tokens

#### Schools (6 endpoints) ✅
- POST `/v1/schools` - Crear escuela
- GET `/v1/schools` - Listar escuelas
- GET `/v1/schools/:id` - Obtener escuela por ID
- GET `/v1/schools/code/:code` - Obtener escuela por código
- PUT `/v1/schools/:id` - Actualizar escuela
- DELETE `/v1/schools/:id` - Eliminar escuela

#### Academic Units (11 endpoints) ✅
- POST `/v1/schools/:schoolId/units` - Crear unidad académica
- GET `/v1/schools/:schoolId/units` - Listar unidades de escuela
- GET `/v1/schools/:schoolId/units/tree` - Árbol jerárquico (ltree)
- GET `/v1/schools/:schoolId/units/by-type` - Filtrar por tipo
- GET `/v1/units/:id` - Obtener unidad
- PUT `/v1/units/:id` - Actualizar unidad
- DELETE `/v1/units/:id` - Soft delete unidad
- POST `/v1/units/:id/restore` - Restaurar unidad eliminada
- GET `/v1/units/:id/hierarchy-path` - Path jerárquico (ltree)

#### Memberships (8 endpoints) ✅
- POST `/v1/memberships` - Crear membership
- GET `/v1/memberships/:id` - Obtener membership
- PUT `/v1/memberships/:id` - Actualizar membership
- DELETE `/v1/memberships/:id` - Eliminar membership
- POST `/v1/memberships/:id/expire` - Expirar membership
- GET `/v1/units/:unitId/memberships` - Memberships de unidad
- GET `/v1/users/:userId/memberships` - Memberships de usuario
- GET `/v1/memberships/role/:role` - Memberships por rol

#### Users (4 endpoints) ✅
- POST `/v1/users` - Crear usuario
- GET `/v1/users/:id` - Obtener usuario
- PUT `/v1/users/:id` - Actualizar usuario
- DELETE `/v1/users/:id` - Eliminar usuario

#### Subjects (2 endpoints) ⚠️ INCOMPLETO
- POST `/v1/subjects` - Crear materia
- PATCH `/v1/subjects/:id` - Actualizar materia
- ❌ **FALTA**: GET `/v1/subjects/:id`
- ❌ **FALTA**: GET `/v1/subjects`
- ❌ **FALTA**: DELETE `/v1/subjects/:id`

#### Guardian Relations (4 endpoints) ⚠️ INCOMPLETO
- POST `/v1/guardian-relations` - Crear relación guardian-estudiante
- GET `/v1/guardian-relations/:id` - Obtener relación por ID
- GET `/v1/guardians/:guardian_id/relations` - Relaciones de un guardian
- GET `/v1/students/:student_id/guardians` - Guardians de un estudiante
- ❌ **FALTA**: PUT `/v1/guardian-relations/:id`
- ❌ **FALTA**: DELETE `/v1/guardian-relations/:id`

#### Materials (1 endpoint) ✅
- DELETE `/v1/materials/:id` - Eliminar material (emite evento)

#### Stats (1 endpoint) ✅
- GET `/v1/stats/global` - Estadísticas globales del sistema

---

### API Mobile - 17 Endpoints

#### Materials (9 endpoints) ⚠️
- POST `/v1/materials` - Crear material (requiere teacher+)
- GET `/v1/materials` - Listar materiales
- GET `/v1/materials/:id` - Obtener material
- GET `/v1/materials/:id/versions` - Historial de versiones
- POST `/v1/materials/:id/upload-url` - Generar URL de upload S3
- POST `/v1/materials/:id/upload-complete` - Notificar upload completo
- GET `/v1/materials/:id/download-url` - Generar URL de descarga S3
- GET `/v1/materials/:id/summary` - Obtener resumen (datos de MongoDB)
- GET `/v1/materials/:id/stats` - Estadísticas del material
- ❌ **FALTA**: PUT `/v1/materials/:id` - Actualizar material

#### Assessments (4 endpoints) ✅
- GET `/v1/materials/:id/assessment` - Obtener quiz del material
- POST `/v1/materials/:id/assessment/attempts` - Crear intento de quiz
- GET `/v1/attempts/:id/results` - Obtener resultados de intento
- GET `/v1/users/me/attempts` - Historial de intentos del usuario

#### Progress (1 endpoint) ✅
- PUT `/v1/progress` - Upsert de progreso (idempotente)

#### Stats (1 endpoint) ✅
- GET `/v1/stats/global` - Estadísticas globales (admin only)

---

### Worker - 5 Processors

#### Processors Implementados (pero con lógica simulada)

1. **MaterialUploadedProcessor** ⚠️ SIMULADO
   - ✅ Recibe evento de material subido
   - ✅ Actualiza estado a `processing` en PostgreSQL
   - ⚠️ Simula extracción de PDF (no descarga de S3)
   - ⚠️ Simula generación de resumen con OpenAI (datos hardcoded)
   - ⚠️ Simula generación de quiz con OpenAI (datos hardcoded)
   - ✅ Guarda en MongoDB: `material_summaries` y `material_assessment_worker`
   - ✅ Actualiza estado a `completed` en PostgreSQL

2. **MaterialDeletedProcessor** ✅ FUNCIONAL
   - ✅ Elimina documentos de MongoDB cuando se borra material
   - ✅ Limpia `material_summaries` y `material_assessment_worker`

3. **MaterialReprocessProcessor** ⚠️ SIMULADO
   - ✅ Vuelve a procesar un material
   - ⚠️ Mismo flujo simulado que MaterialUploadedProcessor

#### Processors Incompletos (solo logging)

4. **AssessmentAttemptProcessor** ❌ NO IMPLEMENTADO
   - ✅ Recibe evento de intento de quiz
   - ❌ Solo hace logging, NO implementa lógica
   - **Debería hacer**:
     - Enviar notificación al docente si score bajo
     - Actualizar estadísticas de material
     - Registrar en tabla de analytics
     - Generar insights de aprendizaje

5. **StudentEnrolledProcessor** ❌ NO IMPLEMENTADO
   - ✅ Recibe evento de estudiante inscrito
   - ❌ Solo hace logging, NO implementa lógica
   - **Debería hacer**:
     - Enviar email de bienvenida
     - Crear registro de onboarding
     - Notificar al teacher de la unidad
     - Inicializar progreso del estudiante

---

## ENDPOINTS FALTANTES EN APIs

### 1. API Admin - Subjects CRUD Completo

#### 1.1. GET /v1/subjects/:id

**Handler**: `SubjectHandler.GetSubject` (nuevo método)
**API**: Admin
**Prioridad**: 🟡 Media

**Archivos a crear/modificar**:
```
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion/
  internal/infrastructure/http/handler/subject_handler.go (modificar)
  internal/infrastructure/http/router/router.go (agregar ruta)
```

**Service requerido**: Ya existe `SubjectService.GetSubject`
**Repository**: Ya existe `SubjectRepository.FindByID`
**DTO**: Ya existe `dto.SubjectResponse`
**Tabla**: `subjects` (PostgreSQL)

**Implementación sugerida**:
```go
// En subject_handler.go
func (h *SubjectHandler) GetSubject(c *gin.Context) {
    id := c.Param("id")

    subject, err := h.subjectService.GetSubject(c.Request.Context(), id)
    if err != nil {
        _ = c.Error(err)
        return
    }

    c.JSON(http.StatusOK, subject)
}

// En router.go - agregar en v1:
subjects := v1.Group("/subjects")
{
    subjects.POST("", subjectHandler.CreateSubject)
    subjects.GET("/:id", subjectHandler.GetSubject)  // NUEVO
    subjects.PATCH("/:id", subjectHandler.UpdateSubject)
}
```

---

#### 1.2. GET /v1/subjects (Listar)

**Handler**: `SubjectHandler.ListSubjects` (nuevo método)
**API**: Admin
**Prioridad**: 🟡 Media

**Service requerido**: Crear `SubjectService.ListSubjects`
**Repository**: Crear método `SubjectRepository.FindAll` o `FindBySchool`
**DTO**: `[]dto.SubjectResponse`
**Tabla**: `subjects`

**Archivos a crear**:
```
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion/
  internal/application/service/subject_service.go (agregar método ListSubjects)
  internal/domain/repository/subject_repository.go (agregar FindAll)
  internal/infrastructure/persistence/postgres/repository/subject_repository_impl.go (implementar FindAll)
  internal/infrastructure/http/handler/subject_handler.go (agregar ListSubjects)
```

**Implementación sugerida**:
```go
// En subject_service.go
func (s *SubjectService) ListSubjects(ctx context.Context, schoolID string) ([]dto.SubjectResponse, error) {
    subjects, err := s.subjectRepo.FindBySchool(ctx, schoolID)
    // ... mapeo a DTO
}

// En subject_repository.go (interfaz)
FindBySchool(ctx context.Context, schoolID string) ([]*entities.Subject, error)

// En subject_handler.go
func (h *SubjectHandler) ListSubjects(c *gin.Context) {
    schoolID := c.Query("school_id") // Filtro opcional
    subjects, err := h.subjectService.ListSubjects(c.Request.Context(), schoolID)
    // ...
    c.JSON(http.StatusOK, subjects)
}
```

---

#### 1.3. DELETE /v1/subjects/:id

**Handler**: `SubjectHandler.DeleteSubject` (nuevo método)
**API**: Admin
**Prioridad**: 🟢 Baja (soft delete es suficiente por ahora)

**Service requerido**: Crear `SubjectService.DeleteSubject`
**Repository**: Ya existe `SubjectRepository.Delete` (verificar si es soft delete)
**DTO**: No requiere (204 No Content)
**Tabla**: `subjects`

**Archivos a modificar**:
```
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion/
  internal/application/service/subject_service.go
  internal/infrastructure/http/handler/subject_handler.go
  internal/infrastructure/http/router/router.go
```

---

### 2. API Admin - Guardian Relations UPDATE y DELETE

#### 2.1. PUT /v1/guardian-relations/:id

**Handler**: `GuardianHandler.UpdateGuardianRelation` (nuevo método)
**API**: Admin
**Prioridad**: 🟡 Media

**Archivos a crear/modificar**:
```
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion/
  internal/application/dto/guardian_dto.go (crear UpdateGuardianRelationRequest)
  internal/application/service/guardian_service.go (agregar UpdateGuardianRelation)
  internal/domain/repository/guardian_repository.go (agregar Update)
  internal/infrastructure/persistence/postgres/repository/guardian_repository_impl.go (implementar Update)
  internal/infrastructure/http/handler/guardian_handler.go (agregar UpdateGuardianRelation)
  internal/infrastructure/http/router/router.go (agregar ruta)
```

**Service requerido**: Crear `GuardianService.UpdateGuardianRelation`
**Repository**: Crear `GuardianRepository.Update`
**DTO Request**:
```go
type UpdateGuardianRelationRequest struct {
    RelationshipType *string `json:"relationship_type,omitempty"` // "parent", "legal_guardian", etc.
    Notes            *string `json:"notes,omitempty"`
    IsActive         *bool   `json:"is_active,omitempty"`
}
```
**DTO Response**: `dto.GuardianRelationResponse` (ya existe)
**Tabla**: `guardian_relations`

**Implementación sugerida**:
```go
// En guardian_handler.go
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

    adminID := c.GetString("admin_id")
    relation, err := h.guardianService.UpdateGuardianRelation(
        c.Request.Context(),
        id,
        req,
        adminID,
    )

    if err != nil {
        _ = c.Error(err)
        return
    }

    h.logger.Info("guardian relation updated", "relation_id", id)
    c.JSON(http.StatusOK, relation)
}
```

---

#### 2.2. DELETE /v1/guardian-relations/:id

**Handler**: `GuardianHandler.DeleteGuardianRelation` (nuevo método)
**API**: Admin
**Prioridad**: 🟡 Media

**Archivos a modificar**:
```
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion/
  internal/application/service/guardian_service.go (agregar DeleteGuardianRelation)
  internal/domain/repository/guardian_repository.go (agregar Delete)
  internal/infrastructure/persistence/postgres/repository/guardian_repository_impl.go (implementar Delete)
  internal/infrastructure/http/handler/guardian_handler.go (agregar DeleteGuardianRelation)
  internal/infrastructure/http/router/router.go (agregar ruta)
```

**Service**: `GuardianService.DeleteGuardianRelation`
**Repository**: `GuardianRepository.Delete` (soft delete recomendado)
**DTO**: No requiere (204 No Content)
**Tabla**: `guardian_relations`

---

### 3. API Mobile - Material UPDATE

#### 3.1. PUT /v1/materials/:id

**Handler**: `MaterialHandler.UpdateMaterial` (nuevo método)
**API**: Mobile
**Prioridad**: 🔴 Alta (frontend necesita editar materiales)

**Archivos a crear/modificar**:
```
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile/
  internal/application/dto/material_dto.go (crear UpdateMaterialRequest)
  internal/application/service/material_service.go (agregar UpdateMaterial)
  internal/domain/repository/material_repository.go (agregar Update)
  internal/infrastructure/persistence/postgres/repository/material_repository_impl.go (implementar Update)
  internal/infrastructure/http/handler/material_handler.go (agregar UpdateMaterial)
  internal/infrastructure/http/router/router.go (agregar ruta)
```

**Service requerido**: Crear `MaterialService.UpdateMaterial`
**Repository**: Crear `MaterialRepository.Update`
**DTO Request**:
```go
type UpdateMaterialRequest struct {
    Title       *string `json:"title,omitempty" validate:"omitempty,min=3,max=200"`
    Description *string `json:"description,omitempty" validate:"omitempty,max=1000"`
    SubjectID   *string `json:"subject_id,omitempty" validate:"omitempty,uuid"`
}
```
**DTO Response**: `dto.MaterialResponse` (ya existe)
**Tabla**: `materials` (PostgreSQL)

**Implementación sugerida**:
```go
// En material_handler.go
func (h *MaterialHandler) UpdateMaterial(c *gin.Context) {
    materialID := c.Param("id")
    var req dto.UpdateMaterialRequest

    if err := c.ShouldBindJSON(&req); err != nil {
        h.logger.Warn("invalid request body", "error", err)
        c.JSON(http.StatusBadRequest, ErrorResponse{
            Error: "invalid request body",
            Code:  "INVALID_REQUEST",
        })
        return
    }

    // Validar que el usuario sea el autor o tenga permisos
    userID := ginmiddleware.MustGetUserID(c)

    material, err := h.materialService.UpdateMaterial(
        c.Request.Context(),
        materialID,
        req,
        userID,
    )

    if err != nil {
        if appErr, ok := errors.GetAppError(err); ok {
            c.JSON(appErr.StatusCode, ErrorResponse{
                Error: appErr.Message,
                Code:  string(appErr.Code),
            })
            return
        }
        c.JSON(http.StatusInternalServerError, ErrorResponse{
            Error: "internal server error",
            Code:  "INTERNAL_ERROR",
        })
        return
    }

    h.logger.Info("material updated", "material_id", materialID)
    c.JSON(http.StatusOK, material)
}

// En router.go - agregar en materials group:
materials.PUT("/:id",
    middleware.RequireTeacher(), // Solo teachers pueden editar
    c.Handlers.MaterialHandler.UpdateMaterial,
)
```

**Consideraciones importantes**:
- Validar que el usuario sea el autor del material o tenga permisos
- Si se actualiza un material con archivo ya procesado, considerar si se debe reprocesar
- Posiblemente emitir evento `material_updated` para que worker reprocese si cambió el archivo
- Implementar validación de permisos a nivel de servicio

---

## CONEXIÓN WORKER-APIs

### Problema: Colecciones MongoDB Duplicadas

**Estado actual**:
- **Worker** guarda en: `material_assessment_worker` y `material_summaries`
- **API Mobile** lee de: `material_assessment` y `material_summaries`

**Ubicación del problema**:
```
Worker:
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/
  internal/infrastructure/persistence/mongodb/repository/material_assessment_repository.go
    Línea 31: db.Collection("material_assessment_worker")  // ❌ Nombre incorrecto

API Mobile:
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile/
  internal/infrastructure/persistence/mongodb/repository/assessment_document_repository.go
    Línea 83: db.Collection("material_assessment")  // ✅ Nombre correcto
```

### Solución: Unificar Nombres de Colecciones

#### Acción 1: Cambiar nombre de colección en Worker

**Archivo a modificar**:
```
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/
  internal/infrastructure/persistence/mongodb/repository/material_assessment_repository.go
```

**Cambio**:
```go
// ANTES (línea 31)
collection: db.Collection("material_assessment_worker"),

// DESPUÉS
collection: db.Collection("material_assessment"),
```

**Archivos adicionales a verificar**:
```
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/
  internal/infrastructure/persistence/mongodb/repository/material_summary_repository.go
    - Verificar que use "material_summaries" (no "material_summaries_worker")
```

#### Acción 2: Actualizar Tests del Worker

**Archivos a modificar**:
```
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/
  internal/infrastructure/persistence/mongodb/repository/material_assessment_repository_test.go
    - Actualizar setup/teardown para usar "material_assessment"

  internal/infrastructure/persistence/mongodb/repository/material_summary_repository_test.go
    - Verificar que use "material_summaries"
```

#### Acción 3: Migración de Datos (si hay datos en producción)

**Si ya existe la colección `material_assessment_worker` en MongoDB de producción**:

```javascript
// Script de migración MongoDB
use edugo_db;

// Copiar datos de material_assessment_worker a material_assessment
db.material_assessment_worker.find().forEach(function(doc) {
    db.material_assessment.insert(doc);
});

// Verificar conteo
print("Documentos en material_assessment_worker:", db.material_assessment_worker.count());
print("Documentos en material_assessment:", db.material_assessment.count());

// Una vez verificado, eliminar colección vieja
// db.material_assessment_worker.drop();
```

**Ruta donde guardar el script**:
```
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/scripts/migrations/
  001_migrate_assessment_collection.js
```

### Flujo de Datos Correcto (después de la unificación)

```
┌──────────────┐                  ┌──────────────┐
│  API Mobile  │                  │    Worker    │
│              │                  │              │
│ 1. POST      │                  │              │
│ /materials   │───────┐          │              │
└──────────────┘       │          └──────────────┘
                       ↓
                ┌──────────────┐
                │  PostgreSQL  │
                │   materials  │
                │  (metadata)  │
                └──────────────┘
                       │
                       │ 2. EventBus
                       ↓
                ┌──────────────┐
                │   RabbitMQ   │
                │   "events"   │
                └──────────────┘
                       │
                       │ 3. Consume
                       ↓
                ┌──────────────┐
                │    Worker    │
                │   Procesa    │
                └──────────────┘
                       │
                       │ 4. Guarda resultado
                       ↓
          ┌────────────────────────┐
          │       MongoDB          │
          │ material_assessment    │ ← MISMO NOMBRE
          │ material_summaries     │ ← MISMO NOMBRE
          └────────────────────────┘
                       ↑
                       │ 5. Consulta
                       │
                ┌──────────────┐
                │  API Mobile  │
                │ GET /summary │
                │ GET /quiz    │
                └──────────────┘
```

### Repositorios a Unificar

#### Material Assessment

**Worker debe apuntar a**:
```go
// /edugo-worker/internal/infrastructure/persistence/mongodb/repository/material_assessment_repository.go
collection: db.Collection("material_assessment")  // Sin sufijo _worker
```

**API Mobile ya usa**:
```go
// /edugo-api-mobile/internal/infrastructure/persistence/mongodb/repository/assessment_document_repository.go
collection: db.Collection("material_assessment")  // ✅ Correcto
```

#### Material Summaries

**Ambos deben usar**:
```go
collection: db.Collection("material_summaries")  // Sin sufijo _worker
```

**Archivos a verificar**:
```
Worker:
  /edugo-worker/internal/infrastructure/persistence/mongodb/repository/material_summary_repository.go

API Mobile:
  /edugo-api-mobile/internal/infrastructure/persistence/mongodb/repository/summary_repository_impl.go
```

---

## PROCESSORS A COMPLETAR EN WORKER

### 1. AssessmentAttemptProcessor

**Estado actual**: ❌ Solo hace logging
**Archivo**: `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/internal/application/processor/assessment_attempt_processor.go`

#### Lógica de Negocio Faltante

**1. Registrar Analytics del Intento**
- Crear tabla `assessment_analytics` en PostgreSQL
- Guardar: material_id, user_id, score, tiempo_respuesta, fecha
- Permitir análisis de rendimiento por material y usuario

**2. Calcular Estadísticas del Material**
- Actualizar tabla `material_stats` (o similar)
- Calcular: promedio de score, intentos totales, tasa de aprobación
- Usar para mostrar dificultad del material

**3. Detectar Score Bajo y Notificar**
- Si `score < 60%`: enviar notificación al docente
- Evento: `student_needs_help` → otro processor o notificación directa
- Incluir: estudiante, material, score, fecha

**4. Actualizar Progreso del Estudiante**
- Si no existe endpoint dedicado, actualizar tabla de progreso
- Registrar material completado, score obtenido
- Calcular nivel de maestría del tema

#### Dependencias de Código

**Tablas/Colecciones requeridas**:
```sql
-- PostgreSQL
CREATE TABLE assessment_analytics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    material_id UUID NOT NULL REFERENCES materials(id),
    user_id UUID NOT NULL REFERENCES users(id),
    attempt_id UUID NOT NULL,
    score DECIMAL(5,2) NOT NULL,
    total_questions INT NOT NULL,
    correct_answers INT NOT NULL,
    time_spent_seconds INT,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_analytics_material ON assessment_analytics(material_id);
CREATE INDEX idx_analytics_user ON assessment_analytics(user_id);
```

**Servicios a crear**:
```
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/
  internal/domain/service/analytics_service.go (nuevo)
    - RecordAttempt(ctx, attemptData) error
    - CalculateMaterialStats(ctx, materialID) error
    - DetectLowScore(score float64) bool

  internal/domain/service/notification_service.go (nuevo)
    - NotifyTeacherLowScore(ctx, studentID, materialID, score) error
```

**Repositorios a crear**:
```
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/
  internal/domain/repository/analytics_repository.go (interfaz)
  internal/infrastructure/persistence/postgres/repository/analytics_repository_impl.go
```

**Eventos a emitir** (opcional):
- `student_needs_help` si score < 60%
- `material_completed` si es el primer intento aprobado

#### Archivos a Modificar/Crear

```
Modificar:
  /edugo-worker/internal/application/processor/assessment_attempt_processor.go
    - Implementar lógica completa en processEvent()

Crear:
  /edugo-worker/internal/domain/repository/analytics_repository.go
  /edugo-worker/internal/infrastructure/persistence/postgres/repository/analytics_repository_impl.go
  /edugo-worker/internal/domain/service/analytics_service.go
  /edugo-worker/internal/domain/service/notification_service.go
```

#### Está Planificado en el Roadmap?

**No está explícitamente en el roadmap actual**, pero encaja en:
- **Fase 3**: Testing y Calidad (agregar tests para este processor)
- **Backlog** según `DEUDA_TECNICA.md` DT-005

**Recomendación**: Agregar como tarea en **Fase 1.5** o **Fase 2.5**
- Después de completar integraciones externas (Fase 2)
- Antes de enfocarse en observabilidad (Fase 4)

---

### 2. StudentEnrolledProcessor

**Estado actual**: ❌ Solo hace logging
**Archivo**: `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/internal/application/processor/student_enrolled_processor.go`

#### Lógica de Negocio Faltante

**1. Inicializar Progreso del Estudiante**
- Crear registro en tabla `student_progress` (si existe)
- Inicializar materiales de la unidad académica
- Estado inicial: no iniciado, 0% completado

**2. Enviar Email de Bienvenida**
- Integrar con servicio de email (SendGrid, AWS SES, etc.)
- Template personalizado con:
  - Nombre del estudiante
  - Nombre de la unidad académica
  - Docentes de la unidad
  - Primeros pasos
  - Link a plataforma

**3. Notificar al Docente de la Unidad**
- Obtener teachers de la unidad (tabla `unit_memberships` con role=teacher)
- Enviar notificación in-app o email
- "Nuevo estudiante inscrito: [nombre]"

**4. Crear Registro de Onboarding**
- Tabla `student_onboarding` con checklist:
  - Completar perfil
  - Ver primer material
  - Hacer primer quiz
  - Conocer docentes
- Usar para gamificación o guías

#### Dependencias de Código

**Tablas requeridas**:
```sql
-- PostgreSQL
CREATE TABLE student_onboarding (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id UUID NOT NULL REFERENCES users(id),
    unit_id UUID NOT NULL REFERENCES academic_units(id),
    profile_completed BOOLEAN DEFAULT FALSE,
    first_material_viewed BOOLEAN DEFAULT FALSE,
    first_quiz_completed BOOLEAN DEFAULT FALSE,
    onboarding_completed BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP
);

CREATE TABLE student_progress (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id UUID NOT NULL REFERENCES users(id),
    unit_id UUID NOT NULL REFERENCES academic_units(id),
    material_id UUID REFERENCES materials(id),
    status VARCHAR(50) NOT NULL, -- "not_started", "in_progress", "completed"
    progress_percentage DECIMAL(5,2) DEFAULT 0.0,
    last_accessed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(student_id, unit_id, material_id)
);
```

**Servicios a crear**:
```
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/
  internal/domain/service/onboarding_service.go (nuevo)
    - CreateOnboardingRecord(ctx, studentID, unitID) error
    - InitializeProgress(ctx, studentID, unitID) error

  internal/infrastructure/email/email_service.go (nuevo)
    - SendWelcomeEmail(ctx, studentEmail, data) error

  internal/domain/service/notification_service.go (ampliar)
    - NotifyTeacherNewStudent(ctx, teacherIDs, studentID, unitID) error
```

**Repositorios a crear**:
```
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/
  internal/domain/repository/onboarding_repository.go
  internal/infrastructure/persistence/postgres/repository/onboarding_repository_impl.go
  internal/domain/repository/progress_repository.go
  internal/infrastructure/persistence/postgres/repository/progress_repository_impl.go
```

#### Archivos a Modificar/Crear

```
Modificar:
  /edugo-worker/internal/application/processor/student_enrolled_processor.go

Crear:
  /edugo-worker/internal/domain/repository/onboarding_repository.go
  /edugo-worker/internal/domain/repository/progress_repository.go
  /edugo-worker/internal/infrastructure/persistence/postgres/repository/onboarding_repository_impl.go
  /edugo-worker/internal/infrastructure/persistence/postgres/repository/progress_repository_impl.go
  /edugo-worker/internal/domain/service/onboarding_service.go
  /edugo-worker/internal/infrastructure/email/email_service.go
```

#### Está Planificado en el Roadmap?

**No está en el roadmap actual**, similar a AssessmentAttemptProcessor.

**Recomendación**:
- Agregar en **Backlog** o nueva **Fase 1.5**
- Menor prioridad que AssessmentAttemptProcessor
- Puede ser **post-MVP** ya que el enrollment funciona sin esto

---

## INTEGRACIONES EXTERNAS PENDIENTES

Según el roadmap del worker (Fase 2), las siguientes integraciones están **planificadas pero NO implementadas**:

### 1. OpenAI Client

**Estado**: ❌ No implementado (solo simulado con datos hardcoded)
**Fase planificada**: Fase 2 (3-4 semanas)

#### Que Falta Implementar

**Archivos a crear**:
```
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/
  internal/infrastructure/nlp/interface.go (interfaz común)
  internal/infrastructure/nlp/openai/client.go
  internal/infrastructure/nlp/openai/client_test.go
  internal/infrastructure/nlp/openai/prompts.go
  internal/infrastructure/nlp/openai/parser.go
  internal/infrastructure/nlp/openai/config.go
  internal/infrastructure/nlp/mock/mock_client.go (para tests)
```

**Funcionalidades requeridas**:

1. **Cliente HTTP OpenAI**
   - Implementar cliente con `net/http` o librería oficial de OpenAI
   - Autenticación con API Key
   - Manejo de rate limits (429)
   - Retry con backoff exponencial
   - Timeout configurable

2. **Prompts para Resumen**
```go
// prompts.go
const SummaryPromptTemplate = `
Analiza el siguiente texto de material educativo y genera un resumen estructurado.

TEXTO:
{{.PDFText}}

FORMATO DE RESPUESTA (JSON):
{
  "main_ideas": ["idea 1", "idea 2", "idea 3"],
  "key_concepts": {
    "concepto1": "definición",
    "concepto2": "definición"
  },
  "sections": [
    {"title": "Sección 1", "summary": "resumen de sección"}
  ],
  "glossary": {
    "término1": "definición"
  }
}
`
```

3. **Prompts para Quiz**
```go
const QuizPromptTemplate = `
Basándote en el siguiente texto educativo, genera {{.QuestionCount}} preguntas de opción múltiple.

TEXTO:
{{.PDFText}}

FORMATO DE RESPUESTA (JSON):
{
  "questions": [
    {
      "id": "q1",
      "text": "¿Pregunta?",
      "type": "multiple_choice",
      "options": [
        {"id": "a", "text": "Opción A"},
        {"id": "b", "text": "Opción B"},
        {"id": "c", "text": "Opción C"},
        {"id": "d", "text": "Opción D"}
      ],
      "correct_answer": "a",
      "feedback": {
        "correct": "Explicación de por qué es correcta",
        "incorrect": "Pista para respuesta incorrecta"
      }
    }
  ]
}
`
```

4. **Parser de Respuestas JSON**
   - Parsear respuesta de OpenAI a structs Go
   - Validar estructura de JSON
   - Manejar respuestas malformadas

**Configuración requerida** (`config.yaml`):
```yaml
nlp:
  provider: openai
  openai:
    api_key: ${OPENAI_API_KEY}
    model: gpt-4-turbo
    max_tokens: 2000
    temperature: 0.7
    timeout: 60s
```

**Variables de entorno** (`.env`):
```bash
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4-turbo
OPENAI_MAX_TOKENS=2000
OPENAI_TEMPERATURE=0.7
```

---

### 2. Extracción PDF

**Estado**: ❌ No implementado (solo logging)
**Fase planificada**: Fase 2 (3-4 semanas)

#### Que Falta Implementar

**Archivos a crear**:
```
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/
  internal/infrastructure/pdf/extractor.go
  internal/infrastructure/pdf/extractor_test.go
  internal/infrastructure/pdf/cleaner.go
  internal/infrastructure/pdf/testdata/simple.pdf
  internal/infrastructure/pdf/testdata/complex.pdf
```

**Funcionalidades requeridas**:

1. **Extractor de Texto**
   - Usar librería: `github.com/pdfcpu/pdfcpu` o `github.com/unidoc/unipdf`
   - Extraer texto de todas las páginas
   - Manejar PDFs con múltiples columnas
   - Detectar PDFs escaneados (sin texto, requiere OCR - post-MVP)

2. **Limpieza de Texto**
   - Eliminar caracteres especiales innecesarios
   - Normalizar espacios en blanco
   - Eliminar headers/footers repetitivos
   - Preservar estructura de párrafos

3. **Validación de PDFs**
   - Verificar que sea un PDF válido
   - Verificar que contenga texto (no solo imágenes)
   - Límite de tamaño configurable (ej. 50MB)

**Implementación sugerida**:
```go
// extractor.go
package pdf

import (
    "context"
    "fmt"
    "io"
    "github.com/pdfcpu/pdfcpu/pkg/api"
)

type Extractor interface {
    ExtractText(ctx context.Context, pdfReader io.Reader) (string, error)
}

type PDFCPUExtractor struct {
    maxSizeMB int
}

func NewPDFCPUExtractor(maxSizeMB int) *PDFCPUExtractor {
    return &PDFCPUExtractor{maxSizeMB: maxSizeMB}
}

func (e *PDFCPUExtractor) ExtractText(ctx context.Context, pdfReader io.Reader) (string, error) {
    // 1. Validar tamaño
    // 2. Extraer texto con pdfcpu
    // 3. Limpiar texto
    // 4. Retornar
}
```

**Configuración** (`config.yaml`):
```yaml
pdf:
  max_size_mb: 50
  allowed_types: [".pdf"]
```

---

### 3. Cliente S3

**Estado**: ⚠️ Parcialmente implementado
**Fase planificada**: Fase 2 (3-4 semanas)

#### Estado Actual

En **api-mobile** ya existe cliente S3 para **upload**:
```
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile/
  internal/infrastructure/storage/s3/client.go
```

**Funcionalidades existentes en api-mobile**:
- ✅ Generar presigned upload URLs
- ✅ Generar presigned download URLs
- ✅ Configuración con AWS SDK

#### Que Falta en Worker

**Archivos a crear**:
```
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/
  internal/infrastructure/storage/interface.go
  internal/infrastructure/storage/s3/client.go
  internal/infrastructure/storage/s3/client_test.go
  internal/infrastructure/storage/s3/downloader.go
  internal/infrastructure/storage/mock/mock_storage.go
```

**Funcionalidades requeridas**:

1. **Descarga de Archivos de S3**
   - Descargar archivo a memoria o disco temporal
   - Retry con backoff exponencial
   - Validación de tipo de archivo
   - Progress tracking (opcional)

2. **Manejo de Errores**
   - Archivo no existe (404)
   - Permisos denegados (403)
   - Errores de red (timeout)
   - Archivo demasiado grande

**Implementación sugerida**:
```go
// downloader.go
package s3

import (
    "context"
    "io"
    "github.com/aws/aws-sdk-go-v2/service/s3"
)

type Downloader interface {
    Download(ctx context.Context, bucket, key string) (io.ReadCloser, error)
}

type S3Downloader struct {
    client  *s3.Client
    timeout time.Duration
}

func (d *S3Downloader) Download(ctx context.Context, bucket, key string) (io.ReadCloser, error) {
    // 1. Crear contexto con timeout
    // 2. GetObject de S3
    // 3. Retornar ReadCloser
}
```

**Configuración** (`.env`):
```bash
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
S3_BUCKET_NAME=edugo-materials
S3_DOWNLOAD_TIMEOUT=120s
```

---

## ENDPOINTS QUE DEPENDEN DEL WORKER

Los siguientes endpoints de **api-mobile** están **funcionales** pero retornan **datos simulados** hasta que el worker esté completo:

### 1. GET /v1/materials/:id/summary

**API**: Mobile
**Handler**: `SummaryHandler.GetSummary`
**Archivo**: `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile/internal/infrastructure/http/handler/summary_handler.go`

#### Funcionalidad del Worker Necesaria

**Processor**: `MaterialUploadedProcessor`
**Fase del Roadmap**: Fase 2 (Integraciones Externas)

**Que necesita**:
1. ✅ Worker descarga PDF de S3
2. ✅ Worker extrae texto del PDF
3. ✅ Worker envía texto a OpenAI con prompt de resumen
4. ✅ Worker parsea respuesta JSON de OpenAI
5. ✅ Worker guarda en MongoDB colección `material_summaries`

**Estado actual**:
- ⚠️ Endpoint funciona y retorna datos
- ❌ Datos son hardcoded por el worker (líneas 64-71 de material_uploaded_processor.go)
- ❌ No hay generación real de IA

**Cuando estará listo**: Fin de **Fase 2** (~4-6 semanas desde inicio de Fase 1)

---

### 2. GET /v1/materials/:id/assessment

**API**: Mobile
**Handler**: `AssessmentHandler.GetMaterialAssessment`
**Archivo**: `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile/internal/infrastructure/http/handler/assessment_handler.go`

#### Funcionalidad del Worker Necesaria

**Processor**: `MaterialUploadedProcessor`
**Fase del Roadmap**: Fase 2 (Integraciones Externas)

**Que necesita**:
1. ✅ Worker descarga PDF de S3
2. ✅ Worker extrae texto del PDF
3. ✅ Worker envía texto a OpenAI con prompt de quiz
4. ✅ Worker parsea respuesta JSON con preguntas
5. ✅ Worker guarda en MongoDB colección `material_assessment`

**Estado actual**:
- ⚠️ Endpoint funciona y retorna quiz
- ❌ Preguntas son hardcoded (líneas 82-96 de material_uploaded_processor.go)
- ❌ No hay generación real de IA

**Cuando estará listo**: Fin de **Fase 2** (~4-6 semanas desde inicio de Fase 1)

---

### 3. POST /v1/materials/:id/assessment/attempts

**API**: Mobile
**Handler**: `AssessmentHandler.CreateMaterialAttempt`

**Funcionalidad del Worker Necesaria**: Ninguna crítica para creación

**Processor relacionado**: `AssessmentAttemptProcessor` (para procesamiento post-intento)

**Estado actual**:
- ✅ Endpoint funciona correctamente
- ✅ Guarda intento en PostgreSQL
- ⚠️ Emite evento `assessment_attempt` que el worker NO procesa (solo logging)

**Bloqueadores**:
- Worker **NO** genera analytics del intento
- Worker **NO** notifica a docentes si score bajo
- Worker **NO** actualiza estadísticas del material

**Cuando estará listo**: Requiere implementar `AssessmentAttemptProcessor` (no está en roadmap actual, sugerido para Fase 1.5)

---

### 4. GET /v1/materials/:id/stats

**API**: Mobile
**Handler**: `StatsHandler.GetMaterialStats`

**Funcionalidad del Worker Necesaria**: Analytics de intentos

**Processor relacionado**: `AssessmentAttemptProcessor`

**Estado actual**:
- Endpoint puede estar funcional
- Retorna estadísticas básicas de PostgreSQL (intentos, promedio score)
- NO retorna estadísticas avanzadas que requieren procesamiento del worker

**Bloqueadores**:
- Falta tabla `assessment_analytics` poblada por worker
- Falta cálculo de estadísticas agregadas por material

**Cuando estará listo**: Junto con `AssessmentAttemptProcessor` (Fase 1.5 sugerida)

---

## TIMELINE SUGERIDO

### Resumen de Prioridades

#### 🔴 Prioridad ALTA (Para Frontend)

1. **PUT /v1/materials/:id** (API Mobile)
   - Frontend necesita editar materiales
   - Estimación: 4-6 horas
   - **Iniciar**: Inmediatamente

2. **Unificar colecciones MongoDB**
   - Worker debe guardar en `material_assessment` (no `_worker`)
   - Estimación: 2 horas
   - **Iniciar**: Inmediatamente (parte de Fase 1)

#### 🟡 Prioridad MEDIA (Para Completitud)

3. **GET /v1/subjects** y **GET /v1/subjects/:id** (API Admin)
   - Completar CRUD de Subjects
   - Estimación: 6-8 horas
   - **Iniciar**: Sprint siguiente

4. **PUT /v1/guardian-relations/:id** y **DELETE** (API Admin)
   - Completar CRUD de Guardian Relations
   - Estimación: 6-8 horas
   - **Iniciar**: Sprint siguiente

5. **Worker Fase 2: Integraciones OpenAI y PDF**
   - Funcionalidad crítica para resúmenes y quiz reales
   - Estimación: 3-4 semanas (según roadmap)
   - **Iniciar**: Después de Fase 1 completa

#### 🟢 Prioridad BAJA (Post-MVP)

6. **DELETE /v1/subjects/:id** (API Admin)
   - Soft delete de materias
   - Estimación: 2-3 horas
   - **Iniciar**: Backlog

7. **AssessmentAttemptProcessor** completo (Worker)
   - Analytics y notificaciones de intentos
   - Estimación: 12-16 horas
   - **Iniciar**: Fase 1.5 o Backlog

8. **StudentEnrolledProcessor** completo (Worker)
   - Onboarding y emails de bienvenida
   - Estimación: 12-16 horas
   - **Iniciar**: Post-MVP

---

### Timeline Detallado (10 semanas)

#### Semana 1-2: Endpoints Críticos Frontend

**API Mobile**:
- [ ] Implementar `PUT /v1/materials/:id` (UpdateMaterial)
  - DTO, Service, Repository, Handler, Tests
  - **Responsable**: Team API Mobile
  - **Entregable**: Endpoint funcional testeado

**API Admin**:
- [ ] Implementar `GET /v1/subjects` (ListSubjects)
- [ ] Implementar `GET /v1/subjects/:id` (GetSubject)
  - Service, Repository, Handler, Tests
  - **Responsable**: Team API Admin
  - **Entregable**: CRUD Subjects completo (excepto delete)

**Worker**:
- [ ] Unificar nombre de colección MongoDB
  - `material_assessment_worker` → `material_assessment`
  - **Responsable**: Team Worker
  - **Entregable**: Worker guarda en colecciones correctas

#### Semana 3-4: Worker Fase 1 (Funcionalidad Crítica)

**Según plan-mejoras/fase-1/README.md**:
- [ ] Implementar ProcessorRegistry (routing real)
- [ ] Refactorizar bootstrap (eliminar doble puntero)
- [ ] Unificar logger
- [ ] Tests y cobertura >60%
- **Entregable**: Worker procesa eventos realmente (aunque con datos simulados)

#### Semana 5-8: Worker Fase 2 (Integraciones Externas)

**Según plan-mejoras/fase-2/README.md**:
- [ ] Implementar cliente OpenAI
  - Prompts resumen y quiz
  - Parser JSON
  - Manejo errores y rate limits

- [ ] Implementar extracción PDF
  - Librería pdfcpu o unidoc
  - Limpieza de texto

- [ ] Implementar cliente S3 (download)
  - Retry con backoff
  - Validación archivos

- [ ] Integrar todo en MaterialUploadedProcessor
  - Flujo: S3 → PDF → OpenAI → MongoDB

- **Entregable**: Resúmenes y quiz generados por IA real

**Estado de endpoints**:
- ✅ `GET /v1/materials/:id/summary` - Datos reales
- ✅ `GET /v1/materials/:id/assessment` - Preguntas reales

#### Semana 9-10: Guardian Relations y Refinamientos

**API Admin**:
- [ ] Implementar `PUT /v1/guardian-relations/:id`
- [ ] Implementar `DELETE /v1/guardian-relations/:id`
- **Entregable**: CRUD Guardian Relations completo

**Opcional** (si hay tiempo):
- [ ] Implementar `DELETE /v1/subjects/:id`
- [ ] Iniciar AssessmentAttemptProcessor

---

### Que Puede Hacer Frontend en Cada Fase

#### Fase 1 (Semana 1-2) - Endpoints Básicos
**Frontend puede desarrollar**:
- ✅ Gestión completa de Subjects (listar, crear, editar)
- ✅ Edición de materiales (título, descripción)
- ⚠️ Resúmenes y quiz (con datos mock del worker)

**Bloqueado**:
- ❌ Resúmenes y quiz reales (necesita Fase 2 del worker)

#### Fase 2 (Semana 3-4) - Worker Funcional
**Frontend puede desarrollar**:
- ✅ Monitoreo de estado de procesamiento de materiales
- ⚠️ Aún usando datos simulados de resumen/quiz

**Bloqueado**:
- ❌ Resúmenes y quiz reales (necesita Fase 3)

#### Fase 3 (Semana 5-8) - Worker con IA
**Frontend puede desarrollar**:
- ✅ **TODO funcional con datos reales**
- ✅ Resúmenes generados por IA
- ✅ Quiz generados por IA
- ✅ Flujo completo de material: crear → subir → procesar → mostrar resumen/quiz

**Desbloqueado**:
- ✅ Experiencia completa de usuario para materiales

#### Fase 4 (Semana 9-10) - Completitud
**Frontend puede desarrollar**:
- ✅ Gestión completa de Guardian Relations
- ✅ Funcionalidades administrativas completas

---

### Dependencias Entre Tareas

```
┌────────────────────────────────────────────────┐
│  Semana 1-2: Endpoints Críticos                │
│  - PUT /materials/:id                          │
│  - GET /subjects                               │
│  - Unificar colecciones MongoDB                │
└────────────────┬───────────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────────┐
│  Semana 3-4: Worker Fase 1                     │
│  - ProcessorRegistry                           │
│  - Routing real de eventos                     │
│  - Prerequisito para Fase 2                    │
└────────────────┬───────────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────────┐
│  Semana 5-8: Worker Fase 2                     │
│  - Cliente OpenAI                              │
│  - Extracción PDF                              │
│  - Cliente S3                                  │
│  DESBLOQUEA: Resúmenes y quiz reales           │
└────────────────┬───────────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────────┐
│  Semana 9-10: Guardian Relations               │
│  - PUT/DELETE endpoints                        │
│  - Opcional: AssessmentAttemptProcessor        │
└────────────────────────────────────────────────┘
```

---

## CONCLUSIONES Y RECOMENDACIONES

### Conclusiones

1. **Las APIs están en buen estado** - La mayoría de endpoints existen y funcionan
2. **Worker es el cuello de botella** - Funcionalidad de IA está completamente simulada
3. **Frontend puede avanzar** - Muchas features no dependen del worker
4. **Algunos CRUD están incompletos** - Subjects y Guardian Relations necesitan endpoints

### Recomendaciones Inmediatas

#### Para API Admin
1. ✅ Completar CRUD de Subjects (GET, GET list)
2. ✅ Completar CRUD de Guardian Relations (PUT, DELETE)
3. ⚠️ Documentar endpoints con Swagger/OpenAPI

#### Para API Mobile
1. 🔴 **URGENTE**: Implementar `PUT /v1/materials/:id`
2. ✅ Verificar que colecciones MongoDB sean correctas

#### Para Worker
1. 🔴 **CRÍTICO**: Cambiar `material_assessment_worker` → `material_assessment`
2. 🔴 **CRÍTICO**: Continuar con Fase 1 del roadmap (routing real)
3. 🔴 **CRÍTICO**: Ejecutar Fase 2 (OpenAI, PDF, S3) cuanto antes
4. ⚠️ Planificar implementación de AssessmentAttemptProcessor (no está en roadmap)

#### Para Frontend
1. ✅ Puede iniciar desarrollo con endpoints actuales
2. ⚠️ Usar datos mock para resúmenes y quiz (avisar al usuario)
3. ✅ Preparar UI para cuando datos reales estén disponibles (Semana 8)

---

## APÉNDICES

### A. Resumen de Archivos Críticos

#### API Admin - Rutas
```
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion/
  internal/infrastructure/http/router/router.go (definición de rutas)
  internal/infrastructure/http/handler/subject_handler.go (handlers subjects)
  internal/infrastructure/http/handler/guardian_handler.go (handlers guardians)
  internal/application/service/subject_service.go (lógica subjects)
  internal/application/service/guardian_service.go (lógica guardians)
```

#### API Mobile - Rutas
```
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile/
  internal/infrastructure/http/router/router.go (definición de rutas)
  internal/infrastructure/http/handler/material_handler.go (handlers materials)
  internal/infrastructure/http/handler/assessment_handler.go (handlers quiz)
  internal/infrastructure/http/handler/summary_handler.go (handlers resumen)
```

#### Worker - Processors
```
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/
  internal/application/processor/material_uploaded_processor.go (procesamiento material)
  internal/application/processor/assessment_attempt_processor.go (procesamiento intento - INCOMPLETO)
  internal/application/processor/student_enrolled_processor.go (procesamiento inscripción - INCOMPLETO)
  internal/infrastructure/persistence/mongodb/repository/material_assessment_repository.go (CAMBIAR COLECCIÓN)
```

#### Worker - Plan de Mejoras
```
/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/
  plan-mejoras/README.md (plan general)
  plan-mejoras/fase-1/README.md (funcionalidad crítica)
  plan-mejoras/fase-2/README.md (integraciones externas)
  documents/mejoras/DEUDA_TECNICA.md (deuda identificada)
```

---

### B. Checklist de Implementación

#### Endpoints Faltantes - API Admin

**Subjects**:
- [ ] GET /v1/subjects - Listar materias
- [ ] GET /v1/subjects/:id - Obtener materia
- [ ] DELETE /v1/subjects/:id - Eliminar materia (opcional)

**Guardian Relations**:
- [ ] PUT /v1/guardian-relations/:id - Actualizar relación
- [ ] DELETE /v1/guardian-relations/:id - Eliminar relación

#### Endpoints Faltantes - API Mobile

**Materials**:
- [ ] PUT /v1/materials/:id - Actualizar material

#### Worker - Fixes Críticos

- [ ] Cambiar `material_assessment_worker` → `material_assessment`
- [ ] Verificar colección `material_summaries` (sin sufijo _worker)
- [ ] Actualizar tests con nombres correctos

#### Worker - Fase 1 (Según Roadmap)

- [ ] Implementar ProcessorRegistry
- [ ] Adaptar processors a interfaz común
- [ ] Conectar registry a processMessage()
- [ ] Refactorizar bootstrap
- [ ] Unificar logger
- [ ] Tests >60% cobertura

#### Worker - Fase 2 (Según Roadmap)

- [ ] Implementar cliente OpenAI
- [ ] Implementar extracción PDF
- [ ] Implementar descarga S3
- [ ] Integrar en MaterialUploadedProcessor
- [ ] Tests con mocks

---

**FIN DEL INFORME**

---

**Próximos Pasos Sugeridos**:

1. Revisar este informe con el equipo
2. Priorizar tareas según necesidades de frontend
3. Asignar responsables por sección (API Admin, API Mobile, Worker)
4. Crear issues en GitHub con referencias a este documento
5. Iniciar con tareas de Prioridad ALTA (Semana 1-2)

**Contacto para Dudas**:
- Equipo de Arquitectura EduGo
- Fecha de revisión: 2025-12-23
