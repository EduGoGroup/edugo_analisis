# Documentacion de Endpoints y Workers - Ecosistema EduGo

**Fecha de generacion:** 2025-12-23
**Version del documento:** 1.1 (Actualizado con swagger regenerado)

---

## Indice

1. [API Administracion](#1-api-administracion)
2. [API Mobile](#2-api-mobile)
3. [Workers](#3-workers)
4. [Analisis Critico](#4-analisis-critico)
5. [Resumen Ejecutivo](#5-resumen-ejecutivo)

---

## 1. API Administracion

**Ubicacion:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion`
**Puerto por defecto:** 8081
**Base Path:** `/v1`

### 1.1 Endpoints por Modulo

#### Modulo: Autenticacion (Auth)

| Metodo | Ruta | Handler | Auth | Descripcion |
|--------|------|---------|------|-------------|
| POST | `/v1/auth/login` | AuthHandler.Login | No | Autentica usuario y retorna tokens JWT |
| POST | `/v1/auth/refresh` | AuthHandler.Refresh | No | Genera nuevo access token usando refresh token |
| POST | `/v1/auth/logout` | AuthHandler.Logout | Bearer | Invalida el access token actual |
| POST | `/v1/auth/switch-context` | AuthHandler.SwitchContext | Bearer | Cambia el contexto de escuela del usuario |
| POST | `/v1/auth/verify` | VerifyHandler.VerifyToken | No | Verifica validez de token JWT |
| POST | `/v1/auth/verify-bulk` | VerifyHandler.VerifyTokenBulk | X-Service-API-Key | Verifica multiples tokens (servicios internos) |

#### Modulo: Escuelas (Schools)

| Metodo | Ruta | Handler | Auth | Descripcion |
|--------|------|---------|------|-------------|
| POST | `/v1/schools` | SchoolHandler.CreateSchool | JWT | Crear nueva escuela |
| GET | `/v1/schools` | SchoolHandler.ListSchools | JWT | Listar todas las escuelas |
| GET | `/v1/schools/{id}` | SchoolHandler.GetSchool | JWT | Obtener escuela por ID |
| GET | `/v1/schools/code/{code}` | SchoolHandler.GetSchoolByCode | JWT | Obtener escuela por codigo unico |
| PUT | `/v1/schools/{id}` | SchoolHandler.UpdateSchool | JWT | Actualizar escuela |
| DELETE | `/v1/schools/{id}` | SchoolHandler.DeleteSchool | JWT | Eliminar escuela (soft delete) |

#### Modulo: Unidades Academicas (Academic Units)

| Metodo | Ruta | Handler | Auth | Descripcion |
|--------|------|---------|------|-------------|
| POST | `/v1/schools/{schoolId}/units` | AcademicUnitHandler.CreateUnit | JWT | Crear unidad academica en escuela |
| GET | `/v1/schools/{schoolId}/units` | AcademicUnitHandler.ListUnitsBySchool | JWT | Listar unidades de una escuela |
| GET | `/v1/schools/{schoolId}/units/tree` | AcademicUnitHandler.GetUnitTree | JWT | Obtener arbol jerarquico de unidades |
| GET | `/v1/schools/{schoolId}/units/by-type` | AcademicUnitHandler.ListUnitsByType | JWT | Listar unidades por tipo |
| GET | `/v1/units/{id}` | AcademicUnitHandler.GetUnit | JWT | Obtener unidad por ID |
| PUT | `/v1/units/{id}` | AcademicUnitHandler.UpdateUnit | JWT | Actualizar unidad academica |
| PATCH | `/v1/units/{id}` | UnitHandler.UpdateUnit | JWT | Actualizar unidad (parcial) |
| DELETE | `/v1/units/{id}` | AcademicUnitHandler.DeleteUnit | JWT | Eliminar unidad (soft delete) |
| POST | `/v1/units/{id}/restore` | AcademicUnitHandler.RestoreUnit | JWT | Restaurar unidad eliminada |
| GET | `/v1/units/{id}/hierarchy-path` | AcademicUnitHandler.GetHierarchyPath | JWT | Obtener ruta jerarquica completa |
| POST | `/v1/units/{id}/members` | UnitHandler.AssignMember | JWT | Asignar miembro a unidad |

#### Modulo: Unidades (Units - Legacy)

| Metodo | Ruta | Handler | Auth | Descripcion |
|--------|------|---------|------|-------------|
| POST | `/v1/units` | UnitHandler.CreateUnit | JWT | Crear unidad organizacional |

#### Modulo: Membresías (Memberships)

| Metodo | Ruta | Handler | Auth | Descripcion |
|--------|------|---------|------|-------------|
| POST | `/v1/memberships` | UnitMembershipHandler.CreateMembership | JWT | Crear membresia (asignar usuario a unidad) |
| GET | `/v1/memberships/{id}` | UnitMembershipHandler.GetMembership | JWT | Obtener membresia por ID |
| PUT | `/v1/memberships/{id}` | UnitMembershipHandler.UpdateMembership | JWT | Actualizar membresia |
| DELETE | `/v1/memberships/{id}` | UnitMembershipHandler.DeleteMembership | JWT | Eliminar membresia permanentemente |
| POST | `/v1/memberships/{id}/expire` | UnitMembershipHandler.ExpireMembership | JWT | Expirar membresia (set valid_until = now) |
| GET | `/v1/units/{unitId}/memberships` | UnitMembershipHandler.ListByUnit | JWT | Listar membresías de una unidad |
| GET | `/v1/units/{unitId}/memberships/by-role` | UnitMembershipHandler.ListByRole | JWT | Listar membresías por rol |
| GET | `/v1/users/{userId}/memberships` | UnitMembershipHandler.ListByUser | JWT | Listar membresías de un usuario |

#### Modulo: Usuarios (Users)

| Metodo | Ruta | Handler | Auth | Descripcion |
|--------|------|---------|------|-------------|
| POST | `/v1/users` | UserHandler.CreateUser | JWT | Crear nuevo usuario |
| GET | `/v1/users/{id}` | UserHandler.GetUser | JWT | Obtener usuario por ID |
| PATCH | `/v1/users/{id}` | UserHandler.UpdateUser | JWT | Actualizar informacion de usuario |
| DELETE | `/v1/users/{id}` | UserHandler.DeleteUser | JWT | Soft delete de usuario (desactivar) |

#### Modulo: Materias (Subjects)

| Metodo | Ruta | Handler | Auth | Descripcion |
|--------|------|---------|------|-------------|
| POST | `/v1/subjects` | SubjectHandler.CreateSubject | JWT | Crear nueva materia |
| PATCH | `/v1/subjects/{id}` | SubjectHandler.UpdateSubject | JWT | Actualizar materia |

#### Modulo: Relaciones Acudiente-Estudiante (Guardian Relations)

| Metodo | Ruta | Handler | Auth | Descripcion |
|--------|------|---------|------|-------------|
| POST | `/v1/guardian-relations` | GuardianHandler.CreateRelation | JWT | Crear relacion acudiente-estudiante |
| GET | `/v1/guardian-relations/{id}` | GuardianHandler.GetRelation | JWT | Obtener relacion por ID |
| GET | `/v1/guardians/{guardian_id}/relations` | GuardianHandler.GetByGuardian | JWT | Listar estudiantes de un acudiente |
| GET | `/v1/students/{student_id}/guardians` | GuardianHandler.GetByStudent | JWT | Listar acudientes de un estudiante |

#### Modulo: Materiales (Materials)

| Metodo | Ruta | Handler | Auth | Descripcion |
|--------|------|---------|------|-------------|
| DELETE | `/v1/materials/{id}` | MaterialHandler.DeleteMaterial | JWT | Soft delete de material |

#### Modulo: Estadisticas (Stats)

| Metodo | Ruta | Handler | Auth | Descripcion |
|--------|------|---------|------|-------------|
| GET | `/v1/stats/global` | StatsHandler.GetGlobalStats | JWT | Estadisticas globales del sistema |

#### Endpoints de Infraestructura

| Metodo | Ruta | Handler | Auth | Descripcion |
|--------|------|---------|------|-------------|
| GET | `/health` | inline | No | Health check basico |
| GET | `/swagger/*any` | ginSwagger | No | Documentacion Swagger UI |

### 1.2 Detalles de Endpoints

#### POST /v1/auth/login

**Request Body:**
```json
{
  "email": "usuario@example.com",
  "password": "password123"
}
```

**Response 200:**
```json
{
  "access_token": "eyJhbGc...",
  "refresh_token": "eyJhbGc...",
  "expires_in": 3600,
  "token_type": "Bearer",
  "user": {
    "id": "uuid",
    "email": "usuario@example.com",
    "first_name": "Juan",
    "last_name": "Perez",
    "full_name": "Juan Perez",
    "role": "teacher",
    "school_id": "uuid"
  }
}
```

**Codigos de Error:**
- 400: Request invalido
- 401: Credenciales invalidas
- 403: Usuario inactivo
- 500: Error interno

---

#### POST /v1/auth/verify

**Request Body:**
```json
{
  "token": "eyJhbGc..."
}
```

**Response 200:**
```json
{
  "valid": true,
  "user_id": "uuid",
  "email": "usuario@example.com",
  "role": "teacher",
  "school_id": "uuid",
  "expires_at": "2024-01-15T11:30:00Z"
}
```

**Headers opcionales:**
- `X-Service-API-Key`: API Key para rate limiting diferenciado

---

#### POST /v1/schools

**Request Body:**
```json
{
  "name": "Colegio San Jose",
  "code": "CSJ001",
  "address": "Calle 123",
  "city": "Bogota",
  "country": "CO",
  "contact_phone": "+57 1234567",
  "contact_email": "contacto@colegiosanjose.edu",
  "subscription_tier": "basic",
  "max_students": 500,
  "max_teachers": 50,
  "metadata": {}
}
```

**Response 201:**
```json
{
  "id": "uuid",
  "name": "Colegio San Jose",
  "code": "CSJ001",
  "address": "Calle 123",
  "city": "Bogota",
  "country": "CO",
  "contact_email": "contacto@colegiosanjose.edu",
  "contact_phone": "+57 1234567",
  "subscription_tier": "basic",
  "max_students": 500,
  "max_teachers": 50,
  "is_active": true,
  "metadata": {},
  "created_at": "2024-01-15T10:00:00Z",
  "updated_at": "2024-01-15T10:00:00Z"
}
```

---

#### POST /v1/schools/{schoolId}/units

**Path Params:**
- `schoolId`: UUID de la escuela

**Request Body:**
```json
{
  "display_name": "Sexto Grado A",
  "code": "6A",
  "type": "section",
  "description": "Seccion A del sexto grado",
  "parent_unit_id": "uuid-del-grado",
  "metadata": {
    "capacity": 30
  }
}
```

**Tipos de unidad validos:** `grade`, `section`, `club`, `department`

---

#### POST /v1/memberships

**Request Body:**
```json
{
  "user_id": "uuid-usuario",
  "unit_id": "uuid-unidad",
  "role": "teacher",
  "valid_from": "2024-01-15T00:00:00Z",
  "valid_until": "2024-12-31T23:59:59Z"
}
```

**Roles validos:** `owner`, `teacher`, `assistant`, `student`, `guardian`

---

#### POST /v1/users

**Request Body:**
```json
{
  "email": "usuario@example.com",
  "password": "securepassword123",
  "first_name": "Juan",
  "last_name": "Perez",
  "role": "teacher"
}
```

**Response 201:**
```json
{
  "id": "uuid",
  "email": "usuario@example.com",
  "first_name": "Juan",
  "last_name": "Perez",
  "full_name": "Juan Perez",
  "role": "teacher",
  "is_active": true,
  "created_at": "2024-01-15T10:00:00Z",
  "updated_at": "2024-01-15T10:00:00Z"
}
```

---

#### GET /v1/stats/global

**Response 200:**
```json
{
  "total_users": 1500,
  "total_active_users": 1200,
  "total_schools": 25,
  "total_subjects": 50,
  "total_guardian_relations": 800
}
```

---

## 2. API Mobile

**Ubicacion:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile`
**Puerto por defecto:** 8080
**Base Path:** `/`

### 2.1 Endpoints por Modulo

#### Modulo: Materiales (Materials)

| Metodo | Ruta | Handler | Auth | Rol Minimo | Descripcion |
|--------|------|---------|------|------------|-------------|
| GET | `/v1/materials` | MaterialHandler.ListMaterials | JWT | any | Listar todos los materiales |
| POST | `/v1/materials` | MaterialHandler.CreateMaterial | JWT | teacher | Crear nuevo material |
| GET | `/v1/materials/{id}` | MaterialHandler.GetMaterial | JWT | any | Obtener material por ID |
| GET | `/v1/materials/{id}/versions` | MaterialHandler.GetMaterialWithVersions | JWT | any | Obtener material con historial de versiones |
| GET | `/v1/materials/{id}/download-url` | MaterialHandler.GenerateDownloadURL | JWT | any | Generar URL presignada de descarga |
| POST | `/v1/materials/{id}/upload-url` | MaterialHandler.GenerateUploadURL | JWT | teacher | Generar URL presignada de subida |
| POST | `/v1/materials/{id}/upload-complete` | MaterialHandler.NotifyUploadComplete | JWT | teacher | Notificar que subida a S3 completo |
| GET | `/v1/materials/{id}/summary` | SummaryHandler.GetSummary | JWT | any | Obtener resumen generado por IA |
| GET | `/v1/materials/{id}/stats` | StatsHandler.GetMaterialStats | JWT | any | Obtener estadisticas del material |

#### Modulo: Evaluaciones (Assessments)

| Metodo | Ruta | Handler | Auth | Descripcion |
|--------|------|---------|------|-------------|
| GET | `/v1/materials/{id}/assessment` | AssessmentHandler.GetMaterialAssessment | JWT | Obtener cuestionario (sin respuestas correctas) |
| POST | `/v1/materials/{id}/assessment/attempts` | AssessmentHandler.CreateMaterialAttempt | JWT | Crear intento de evaluacion |
| GET | `/v1/attempts/{id}/results` | AssessmentHandler.GetAttemptResults | JWT | Obtener resultados de un intento |
| GET | `/v1/users/me/attempts` | AssessmentHandler.GetUserAttemptHistory | JWT | Historial de intentos del usuario |

#### Modulo: Progreso (Progress)

| Metodo | Ruta | Handler | Auth | Descripcion |
|--------|------|---------|------|-------------|
| PUT | `/v1/progress` | ProgressHandler.UpsertProgress | JWT | Actualizar progreso de lectura (idempotente) |

#### Modulo: Estadisticas (Stats)

| Metodo | Ruta | Handler | Auth | Rol Minimo | Descripcion |
|--------|------|---------|------|------------|-------------|
| GET | `/v1/stats/global` | StatsHandler.GetGlobalStats | JWT | admin | Estadisticas globales del sistema |

#### Endpoints de Infraestructura

| Metodo | Ruta | Handler | Auth | Descripcion |
|--------|------|---------|------|-------------|
| GET | `/health` | HealthHandler.Check | No | Health check (detail=1 para detallado) |
| GET | `/metrics` | promhttp | No | Metricas Prometheus |
| GET | `/swagger/*` | SetupSwaggerUI | No | Documentacion Swagger |

### 2.2 Detalles de Endpoints

#### POST /v1/materials

**Request Body:**
```json
{
  "title": "Introduccion al Calculo",
  "description": "Guia completa de calculo diferencial e integral",
  "subject": "Mathematics",
  "grade": "12th Grade"
}
```

**Validaciones:**
- `title`: requerido, 3-200 caracteres
- `description`: opcional, max 1000 caracteres

**Response 201:**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "title": "Introduccion al Calculo",
  "description": "Guia completa de calculo diferencial e integral",
  "subject": "Mathematics",
  "grade": "12th Grade",
  "status": "uploaded",
  "school_id": "660e8400-e29b-41d4-a716-446655440001",
  "academic_unit_id": "880e8400-e29b-41d4-a716-446655440003",
  "uploaded_by_teacher_id": "770e8400-e29b-41d4-a716-446655440002",
  "file_url": null,
  "file_type": null,
  "file_size_bytes": null,
  "is_public": false,
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T10:30:00Z"
}
```

---

#### GET /v1/materials/{id}/assessment

**Response 200:**
```json
{
  "assessment_id": "uuid",
  "material_id": "uuid",
  "title": "Evaluacion: Introduccion al Calculo",
  "total_questions": 10,
  "time_limit_minutes": 30,
  "pass_threshold": 70,
  "max_attempts": 3,
  "estimated_time_minutes": 25,
  "questions": [
    {
      "id": "q1",
      "text": "Cual es la derivada de x^2?",
      "type": "multiple_choice",
      "options": [
        {"id": "a", "text": "x"},
        {"id": "b", "text": "2x"},
        {"id": "c", "text": "x^2"},
        {"id": "d", "text": "2"}
      ]
    }
  ]
}
```

**NOTA:** Las respuestas correctas NO se incluyen en este endpoint por seguridad.

---

#### POST /v1/materials/{id}/assessment/attempts

**Request Body:**
```json
{
  "answers": [
    {
      "question_id": "q1",
      "selected_answer_id": "b",
      "time_spent_seconds": 45
    }
  ],
  "time_spent_seconds": 1200
}
```

**Validaciones:**
- `answers`: requerido, minimo 1 respuesta
- `time_spent_seconds`: requerido, 1-7200 segundos

**Response 201:**
```json
{
  "attempt_id": "uuid",
  "assessment_id": "uuid",
  "score": 80,
  "max_score": 100,
  "passed": true,
  "pass_threshold": 70,
  "correct_answers": 8,
  "total_questions": 10,
  "time_spent_seconds": 1200,
  "started_at": "2024-01-15T10:00:00Z",
  "completed_at": "2024-01-15T10:20:00Z",
  "can_retake": true,
  "previous_best_score": 75,
  "feedback": [
    {
      "question_id": "q1",
      "question_text": "Cual es la derivada de x^2?",
      "is_correct": true,
      "selected_option": "b",
      "correct_answer": "b",
      "message": "Correcto!"
    }
  ]
}
```

---

#### PUT /v1/progress

**Request Body:**
```json
{
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "material_id": "660e8400-e29b-41d4-a716-446655440001",
  "progress_percentage": 75,
  "last_page": 45
}
```

**Validaciones:**
- `user_id`: requerido, UUID valido
- `material_id`: requerido, UUID valido
- `progress_percentage`: requerido, 0-100

**Response 200:**
```json
{
  "user_id": "550e8400-...",
  "material_id": "660e8400-...",
  "progress_percentage": 75,
  "last_page": 45,
  "message": "progress updated successfully"
}
```

**Autorizacion:**
- Usuario solo puede actualizar su propio progreso
- Admins pueden actualizar progreso de cualquier usuario

---

#### GET /health?detail=1

**Response 200 (detallado):**
```json
{
  "status": "healthy",
  "service": "edugo-api-mobile",
  "version": "1.0",
  "timestamp": "2024-01-15T10:30:00Z",
  "total_time": "150ms",
  "components": {
    "postgres": {
      "status": "healthy",
      "latency": "5ms"
    },
    "mongodb": {
      "status": "healthy",
      "latency": "8ms"
    },
    "rabbitmq": {
      "status": "not_configured",
      "latency": "0ms",
      "optional": true
    },
    "s3": {
      "status": "healthy",
      "latency": "120ms",
      "optional": true
    }
  }
}
```

---

## 3. Workers

**Ubicacion:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker`

### 3.1 Processors Registrados

| Evento | Processor | Cola | Descripcion |
|--------|-----------|------|-------------|
| `material_uploaded` | MaterialUploadedProcessor | material_events | Procesa PDF: extraccion, resumen IA, quiz IA |
| `material_deleted` | MaterialDeletedProcessor | material_events | Limpia datos en MongoDB (summary, assessment) |
| `material_reprocess` | MaterialReprocessProcessor | material_events | Reprocesa material (reutiliza MaterialUploadedProcessor) |
| `assessment_attempt` | AssessmentAttemptProcessor | material_events | Procesa intento de evaluacion (notificaciones, analytics) |
| `student_enrolled` | StudentEnrolledProcessor | material_events | Procesa inscripcion de estudiante (onboarding) |

### 3.2 Detalles de Processors

#### MaterialUploadedProcessor

**Evento de entrada:**
```json
{
  "event_type": "material_uploaded",
  "material_id": "uuid",
  "author_id": "uuid",
  "s3_key": "materials/uuid/archivo.pdf",
  "preferred_language": "es",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

**Flujo de procesamiento:**
1. Actualiza estado a `processing` en PostgreSQL
2. Extrae texto del PDF (actualmente simulado)
3. Genera resumen con OpenAI (actualmente simulado)
4. Guarda resumen en MongoDB (`material_summaries`)
5. Genera quiz con IA (actualmente simulado)
6. Guarda quiz en MongoDB (`material_assessments`)
7. Actualiza estado a `completed` en PostgreSQL

**Datos producidos:**
- Coleccion MongoDB: `material_summaries`
- Coleccion MongoDB: `material_assessments`
- Campo PostgreSQL: `materials.processing_status`

**Repositorios utilizados:**
- PostgreSQL: materials table
- MongoDB: material_summaries, material_assessments

---

#### MaterialDeletedProcessor

**Evento de entrada:**
```json
{
  "event_type": "material_deleted",
  "material_id": "uuid",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

**Flujo de procesamiento:**
1. Elimina documento de `material_summaries` en MongoDB
2. Elimina documento de `material_assessments` en MongoDB

**NOTA:** No modifica datos en PostgreSQL (ya fue soft-deleted por la API)

---

#### MaterialReprocessProcessor

**Evento de entrada:** Mismo formato que `material_uploaded`

**Flujo de procesamiento:**
1. Loguea que es un reproceso
2. Delega completamente a `MaterialUploadedProcessor.processEvent()`

---

#### AssessmentAttemptProcessor

**Evento de entrada:**
```json
{
  "event_type": "assessment_attempt",
  "material_id": "uuid",
  "user_id": "uuid",
  "answers": {"q1": "a", "q2": "b"},
  "score": 85.5,
  "timestamp": "2024-01-15T10:30:00Z"
}
```

**Flujo de procesamiento:**
1. Loguea el intento recibido
2. (TODO) Enviar notificacion al docente si score bajo
3. (TODO) Actualizar estadisticas
4. (TODO) Registrar en tabla de analytics

**Estado actual:** Solo logging, logica de negocio pendiente de implementar

---

#### StudentEnrolledProcessor

**Evento de entrada:**
```json
{
  "event_type": "student_enrolled",
  "student_id": "uuid",
  "unit_id": "uuid",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

**Flujo de procesamiento:**
1. Loguea la inscripcion
2. (TODO) Enviar email de bienvenida
3. (TODO) Crear registro de onboarding
4. (TODO) Notificar al teacher

**Estado actual:** Solo logging, logica de negocio pendiente de implementar

---

## 4. Analisis Critico

### 4.1 Problemas Detectados

#### CRITICO

| ID | Problema | Ubicacion | Descripcion | Recomendacion |
|----|----------|-----------|-------------|---------------|
| C01 | ~~Swagger desactualizado en API Admin~~ | ~~`/docs/swagger.yaml`~~ | ~~El swagger documenta endpoints que NO existen~~ | **RESUELTO** - Swagger regenerado el 2025-12-23 |
| C02 | Processors incompletos en Worker | `assessment_attempt_processor.go`, `student_enrolled_processor.go` | Los processors solo hacen logging, sin logica de negocio real | Implementar logica de notificaciones, analytics y onboarding |
| C03 | Procesamiento IA simulado | `material_uploaded_processor.go` | Extraccion PDF, resumen IA y generacion de quiz estan simulados con datos hardcodeados | Integrar bibliotecas de PDF y OpenAI API |

#### ALTO

| ID | Problema | Ubicacion | Descripcion | Recomendacion |
|----|----------|-----------|-------------|---------------|
| A01 | Falta paginacion en listados | Ambas APIs | `ListSchools`, `ListMaterials`, `ListMembershipsByUnit` no tienen paginacion | Agregar parametros `limit`, `offset`, `page` |
| A02 | Falta endpoint UPDATE material en Mobile | API Mobile | Solo puede crear materiales, no actualizar metadatos | Agregar PUT `/v1/materials/{id}` |

#### MEDIO

| ID | Problema | Ubicacion | Descripcion | Recomendacion |
|----|----------|-----------|-------------|---------------|
| M01 | Metricas solo en Mobile | API Mobile router | Solo Mobile tiene endpoint `/metrics` para Prometheus | Agregar metricas a API Admin |
| M02 | Health check basico en Admin | API Admin main.go | Solo retorna `{"status": "ok"}`, no verifica DB | Implementar health check detallado como en Mobile |
| M03 | Logging inconsistente | Ambos proyectos | Algunos logs en ingles, otros en espanol | Unificar idioma de logs (preferiblemente ingles) |
| M04 | Request ID solo en Mobile | API Mobile middleware | Solo Mobile genera X-Request-ID para tracing | Agregar a API Admin |

#### BAJO

| ID | Problema | Ubicacion | Descripcion | Recomendacion |
|----|----------|-----------|-------------|---------------|
| B01 | Constantes magicas | Varios handlers | Tiempos como `15*time.Minute`, `1*time.Hour` hardcodeados | Extraer a configuracion |
| B02 | Falta versionado de Worker | Worker main.go | No tiene variables Version/BuildTime como API Admin | Agregar ldflags en Makefile |

### 4.2 Endpoints Duplicados

| Funcionalidad | API Admin | API Mobile | Observacion |
|---------------|-----------|------------|-------------|
| Stats globales | `/v1/stats/global` | `/v1/stats/global` | Ambos tienen el endpoint, diferentes implementaciones |

**Conclusion:** No hay duplicacion problematica. Las APIs tienen responsabilidades claramente separadas.

### 4.3 Cobertura de CRUD

| Entidad | CREATE | READ | UPDATE | DELETE | API |
|---------|--------|------|--------|--------|-----|
| School | ✓ | ✓ | ✓ | ✓ | Admin |
| Academic Unit | ✓ | ✓ | ✓ | ✓ + Restore | Admin |
| Membership | ✓ | ✓ | ✓ | ✓ + Expire | Admin |
| User | ✓ | ✓ | ✓ | ✓ | Admin |
| Subject | ✓ | - | ✓ | - | Admin |
| Guardian Relation | ✓ | ✓ | - | - | Admin |
| Material | ✓ | ✓ | - | ✓ | Mobile/Admin |
| Progress | UPSERT | - | - | - | Mobile |
| Assessment Attempt | ✓ | ✓ | - | - | Mobile |

### 4.4 Mejoras Recomendadas

#### Prioridad Alta

1. **Implementar logica en processors**
   - `AssessmentAttemptProcessor`: notificaciones, analytics
   - `StudentEnrolledProcessor`: email bienvenida, onboarding

2. **Agregar paginacion a listados**
   - Patron estandar: `?page=1&limit=20` o `?offset=0&limit=20`
   - Respuesta con metadata: `{data: [], total: N, page: X, limit: Y}`

3. **Completar CRUD de entidades**
   - Subject: agregar READ y DELETE
   - Guardian Relation: agregar UPDATE y DELETE

#### Prioridad Media

4. **Unificar health checks**
   - API Admin debe verificar PostgreSQL
   - Agregar endpoint `/health?detail=1` a API Admin

5. **Agregar metricas Prometheus a API Admin**
   - Reutilizar middleware de Mobile
   - Endpoint `/metrics`

6. **Implementar request ID en API Admin**
   - Middleware que genere/propague X-Request-ID
   - Incluir en logs para tracing

#### Prioridad Baja

7. **Extraer configuracion de tiempos**
   - Crear constantes o variables de config para TTLs
   - URLs presignadas, timeouts, etc.

---

## 5. Resumen Ejecutivo

### Estadisticas

| Metrica | API Admin | API Mobile | Worker |
|---------|-----------|------------|--------|
| Total endpoints | 46 | 17 | N/A |
| Endpoints publicos | 8 | 3 | N/A |
| Endpoints protegidos | 38 | 14 | N/A |
| Processors | N/A | N/A | 5 |
| Modulos | 8 | 4 | N/A |

### Desglose API Admin (46 endpoints)

| Modulo | Cantidad |
|--------|----------|
| Auth | 6 |
| Schools | 6 |
| Academic Units | 11 |
| Units (legacy) | 1 |
| Memberships | 8 |
| Users | 4 |
| Subjects | 2 |
| Guardian Relations | 4 |
| Materials | 1 |
| Stats | 1 |
| Infraestructura | 2 |

### Desglose API Mobile (17 endpoints)

| Modulo | Cantidad |
|--------|----------|
| Materials | 9 |
| Assessments | 4 |
| Progress | 1 |
| Stats | 1 |
| Infraestructura | 2 |

### Issues por Severidad

| Severidad | Cantidad | Accion Requerida |
|-----------|----------|------------------|
| CRITICO | 2 | Inmediata |
| ALTO | 2 | Proximo sprint |
| MEDIO | 4 | Planificar |
| BAJO | 2 | Backlog |

### Arquitectura General

```
                    +-------------------+
                    |   Frontend Web    |
                    |   Frontend Mobile |
                    +---------+---------+
                              |
              +---------------+---------------+
              |                               |
    +---------v---------+           +---------v---------+
    |   API Admin       |           |   API Mobile      |
    |   (Port 8081)     |           |   (Port 8080)     |
    |                   |           |                   |
    | - Auth (6)        |   auth    | - Materials (9)   |
    | - Schools (6)     |<--------->| - Assessments (4) |
    | - Units (12)      |  verify   | - Progress (1)    |
    | - Memberships (8) |           | - Stats (1)       |
    | - Users (4)       |           +--------+----------+
    | - Subjects (2)    |                    |
    | - Guardians (4)   |                    v
    | - Materials (1)   |          +-------------------+
    | - Stats (1)       |          |     MongoDB       |
    +---------+---------+          | - summaries       |
              |                    | - assessments     |
              v                    | - attempts        |
    +-------------------+          +-------------------+
    |    PostgreSQL     |                    ^
    | (edugo_db)        |                    |
    +-------------------+          +-------------------+
                                   |    RabbitMQ       |
                                   | - material_events |
                                   +--------+----------+
                                            |
                                            v
                                   +-------------------+
                                   |     Worker        |
                                   |                   |
                                   | - MaterialUploaded|
                                   | - MaterialDeleted |
                                   | - MaterialReprocess|
                                   | - AssessmentAttempt|
                                   | - StudentEnrolled |
                                   +-------------------+
                                            |
                                            v
                                   +-------------------+
                                   |    OpenAI API     |
                                   | (pendiente)       |
                                   +-------------------+
```

### Conclusiones

1. **Las APIs tienen una excelente separacion de responsabilidades**: Admin maneja estructura organizacional (46 endpoints), Mobile maneja operaciones diarias de usuarios (17 endpoints).

2. **El swagger esta ahora sincronizado** con el codigo real (regenerado el 2025-12-23).

3. **El worker tiene la estructura correcta** pero los processors estan incompletos, funcionando solo como stubs.

4. **API Mobile esta mas madura** con mejor manejo de errores, metricas y health checks detallados.

5. **Falta implementar** la integracion real con OpenAI para resumen y generacion de quizzes.

6. **API Admin tiene cobertura CRUD completa** para las entidades principales (School, Units, Memberships, Users).

---

*Documento generado automaticamente. Ultima actualizacion: 2025-12-23 - Swagger regenerado.*
