# INFORME ANÁLISIS ULTRATHINK: edugo-api-mobile

**Fecha:** 2025-12-24
**Versión del proyecto:** v0.15.0
**Rama analizada:** dev
**Objetivo:** Evaluar si la API está lista para que el frontend comience a consumirla

---

## 📊 RESUMEN EJECUTIVO

### Estado General: ⚠️ PARCIALMENTE LISTO

| Aspecto | Estado | Nivel de Riesgo |
|---------|--------|-----------------|
| **Ramas Git** | ⚠️ DESINCRONIZADAS | MEDIO |
| **Swagger** | ✅ ACTUALIZADO | BAJO |
| **Arquitectura** | ✅ SÓLIDA | BAJO |
| **Endpoints** | ✅ FUNCIONALES | BAJO |
| **Contratos** | ✅ DOCUMENTADOS | BAJO |
| **Responsabilidad BD** | ✅ CORRECTA | BAJO |
| **Eventos RabbitMQ** | ✅ DEFINIDOS | BAJO |
| **Dependencias Worker** | ⚠️ CRÍTICAS | ALTO |

### Recomendación Principal
**El frontend PUEDE comenzar a consumir la API**, pero debe estar consciente de:
1. Las diferencias entre dev y main (usar dev para desarrollo)
2. La dependencia crítica del worker para assessments y summaries
3. Que algunos endpoints retornarán 404 hasta que el worker procese los PDFs

---

## 1️⃣ ESTADO DE RAMAS GIT

### ❌ RAMAS DESINCRONIZADAS

**Último commit dev:**
```
cc7f686 feat(materials): Agregar endpoint PUT para actualizar materiales (#97)
Fecha: 2025-12-23
```

**Último commit main:**
```
869b628 Release: Sistema de repositorios mock para desarrollo sin Docker (#79)
Fecha: 2025-11-25
```

### Diferencias Críticas

La rama `dev` está **adelantada** con respecto a `main` con las siguientes mejoras:

#### Nuevos Features en dev (No en main):
1. **PUT /v1/materials/:id** - Endpoint para actualizar materiales (PR #97)
2. **Homologación colecciones MongoDB** - assessment y summary (PR #96)
3. **Release v0.15.0 y v0.14.0** - Mejoras varias
4. **Observabilidad mejorada** - Request ID, logging estructurado, métricas Prometheus

#### Archivos Afectados (294 archivos cambiados):
- **+12,086 líneas agregadas**
- **-65,535 líneas eliminadas**
- Reorganización masiva de documentación
- Actualización de dependencias (go.mod, go.sum)
- Mejoras en middleware (logging, métricas, request_id)
- Refactorización de servicios y handlers

### ⚠️ RECOMENDACIÓN
- **Frontend debe apuntar a rama `dev`** para desarrollo
- **Solicitar sync dev → main antes de producción**
- Verificar con DevOps cuál es la rama de deploy actual

---

## 2️⃣ ESTADO SWAGGER

### ✅ SWAGGER ACTUALIZADO Y COMPLETO

**Ubicación:** `/docs/swagger.yaml`, `/docs/swagger.json`
**UI Disponible:** `http://localhost:8080/swagger/index.html`
**Última actualización:** 2025-12-23

### Endpoints Documentados (18 totales)

#### 🏥 Health Check (1)
- `GET /health` - Sin autenticación, con opción `?detail=1`

#### 📚 Materials (8)
| Método | Endpoint | Requiere Teacher | Descripción |
|--------|----------|------------------|-------------|
| `GET` | `/v1/materials` | ❌ | Listar todos los materiales |
| `POST` | `/v1/materials` | ✅ | Crear nuevo material |
| `GET` | `/v1/materials/:id` | ❌ | Obtener material específico |
| `PUT` | `/v1/materials/:id` | ✅ | **NUEVO** Actualizar material |
| `GET` | `/v1/materials/:id/versions` | ❌ | Historial de versiones |
| `POST` | `/v1/materials/:id/upload-url` | ✅ | Generar URL presignada S3 para subir |
| `GET` | `/v1/materials/:id/download-url` | ❌ | Generar URL presignada S3 para descargar |
| `POST` | `/v1/materials/:id/upload-complete` | ✅ | Notificar que subida S3 completó |

#### 📝 Assessments (5)
| Método | Endpoint | Descripción |
|--------|----------|-------------|
| `GET` | `/v1/materials/:id/assessment` | Obtener quiz (SIN respuestas correctas) |
| `POST` | `/v1/materials/:id/assessment/attempts` | Crear intento y obtener calificación |
| `GET` | `/v1/attempts/:id/results` | Resultados detallados de un intento |
| `GET` | `/v1/users/me/attempts` | Historial de intentos del usuario (paginado) |
| `GET` | `/v1/materials/:id/summary` | Obtener resumen IA del material |

#### 📈 Progress (1)
| Método | Endpoint | Descripción |
|--------|----------|-------------|
| `PUT` | `/v1/progress` | **UPSERT** progreso (idempotente) |

#### 📊 Stats (2)
| Método | Endpoint | Requiere Admin | Descripción |
|--------|----------|----------------|-------------|
| `GET` | `/v1/materials/:id/stats` | ❌ | Estadísticas de un material |
| `GET` | `/v1/stats/global` | ✅ | Estadísticas globales del sistema |

### DTOs Documentados (17)

#### Request DTOs
- `CreateMaterialRequest` - title, description, subject, grade
- `UpdateMaterialRequest` - **NUEVO** - todos campos opcionales
- `GenerateUploadURLRequest` - file_name, content_type
- `UploadCompleteRequest` - file_url, file_type, file_size_bytes
- `CreateAttemptRequest` - answers[], time_spent_seconds
- `UserAnswerDTO` - question_id, selected_answer_id, time_spent_seconds
- `UpsertProgressRequest` - user_id, material_id, progress_percentage, last_page

#### Response DTOs
- `MaterialResponse` - 15 campos (id, title, status, file_url, school_id, etc.)
- `MaterialWithVersionsResponse` - material + versions[]
- `MaterialVersionResponse` - version_number, created_at, changed_by
- `AssessmentResponse` - questions[], title, max_attempts
- `QuestionDTO` / `OptionDTO` - Estructura de preguntas
- `AttemptResultResponse` - score, passed, feedback[], can_retake
- `AttemptHistoryResponse` - attempts[], total_count, pagination
- `AnswerFeedbackDTO` - is_correct, correct_answer, message
- `GenerateUploadURLResponse` - upload_url, file_url, expires_in
- `GenerateDownloadURLResponse` - download_url, expires_in
- `ProgressResponse` - user_id, material_id, progress_percentage
- `HealthResponse` / `DetailedHealthResponse` - status, components

### ✅ VERIFICACIÓN
- Swagger sincronizado con handlers reales
- DTOs match con código
- Validaciones documentadas (min, max, required)
- Ejemplos presentes
- Seguridad (BearerAuth) documentada

---

## 3️⃣ ESTRUCTURA DEL PROYECTO

### Arquitectura: Clean Architecture + DDD

```
edugo-api-mobile/
├── cmd/                           # Entrypoint
│   └── main.go                    # Inicialización Bootstrap
├── internal/
│   ├── application/               # Capa de Aplicación
│   │   ├── dto/                   # Data Transfer Objects
│   │   │   └── material_dto.go
│   │   ├── service/               # Lógica de negocio
│   │   │   ├── assessment_attempt_service.go
│   │   │   ├── material_service.go
│   │   │   ├── progress_service.go
│   │   │   ├── stats_service.go
│   │   │   └── summary_service.go
│   │   └── usecase/               # Casos de uso complejos (vacío por ahora)
│   ├── domain/                    # Capa de Dominio (核心业务)
│   │   ├── repositories/          # Interfaces de repositorio
│   │   ├── repository/            # Tipos y contratos
│   │   ├── services/              # Servicios de dominio
│   │   ├── valueobject/           # Value Objects (Score, TimeSpent, etc.)
│   │   └── errors/                # Errores de dominio
│   ├── infrastructure/            # Capa de Infraestructura
│   │   ├── http/
│   │   │   ├── handler/           # HTTP Handlers (Controllers)
│   │   │   │   ├── material_handler.go
│   │   │   │   ├── assessment_handler.go
│   │   │   │   ├── progress_handler.go
│   │   │   │   ├── stats_handler.go
│   │   │   │   └── health_handler.go
│   │   │   ├── middleware/        # Middleware personalizados
│   │   │   │   ├── auth.go        # RequireTeacher, RequireAdmin
│   │   │   │   ├── remote_auth.go # Validación JWT contra api-admin
│   │   │   │   ├── logging.go     # Logging estructurado
│   │   │   │   ├── metrics.go     # Prometheus metrics
│   │   │   │   └── request_id.go  # Request ID propagation
│   │   │   └── router/
│   │   │       └── router.go      # Definición de rutas
│   │   ├── persistence/
│   │   │   ├── postgres/
│   │   │   │   └── repository/    # Repositorios PostgreSQL
│   │   │   │       ├── material_repository_impl.go
│   │   │   │       ├── assessment_repository.go
│   │   │   │       ├── attempt_repository.go
│   │   │   │       ├── answer_repository.go
│   │   │   │       ├── progress_repository_impl.go
│   │   │   │       └── user_repository_impl.go
│   │   │   ├── mongodb/
│   │   │   │   └── repository/    # Repositorios MongoDB
│   │   │   │       ├── assessment_document_repository.go
│   │   │   │       └── summary_repository_impl.go
│   │   │   └── mock/              # Mocks para desarrollo sin Docker
│   │   │       ├── postgres/
│   │   │       ├── mongodb/
│   │   │       └── dataset/       # Dataset de prueba
│   │   ├── messaging/
│   │   │   └── rabbitmq/
│   │   │       ├── publisher.go   # RabbitMQ Publisher
│   │   │       ├── events.go      # Definición de eventos
│   │   │       └── resilient_publisher.go
│   │   └── storage/
│   │       └── s3/
│   │           └── client.go      # Cliente AWS S3
│   ├── bootstrap/                 # Inicialización y DI
│   │   ├── bootstrap.go
│   │   ├── bridge.go              # Adaptadores edugo-shared
│   │   └── config.go
│   ├── client/                    # Clientes HTTP externos
│   │   └── auth_client.go         # Validación tokens api-admin
│   ├── config/                    # Configuración
│   │   ├── config.go
│   │   └── loader.go
│   └── container/                 # Dependency Injection Container
│       ├── factory.go
│       ├── services.go
│       ├── repositories.go
│       ├── handlers.go
│       └── infrastructure.go
├── config/                        # Archivos de configuración
│   └── config.yaml
├── docs/                          # Documentación generada (Swagger)
│   ├── swagger.yaml
│   ├── swagger.json
│   └── docs.go
├── documents/                     # Documentación técnica
│   ├── README.md
│   ├── ARCHITECTURE.md
│   ├── DATABASE.md
│   ├── API-REFERENCE.md
│   ├── SETUP.md
│   └── improvements/              # Deuda técnica y mejoras
└── test/                          # Tests de integración
    └── integration/
```

### Capas de Arquitectura

#### 1. **Domain** (Núcleo)
- **NO** depende de nada externo
- Define contratos (interfaces de repositorio)
- Value Objects inmutables
- Errores de negocio

#### 2. **Application** (Lógica de Negocio)
- Orquesta el dominio
- Implementa casos de uso
- DTOs para input/output
- Depende SOLO de Domain

#### 3. **Infrastructure** (Implementaciones)
- Implementa interfaces del domain
- Acceso a BD (PostgreSQL, MongoDB)
- HTTP handlers (Gin)
- Cliente S3, RabbitMQ
- Depende de Application y Domain

#### 4. **Bootstrap** (Inicialización)
- Inyección de dependencias
- Configuración
- Adaptadores para edugo-shared

### Patrón: Repository Pattern + Service Pattern

```
HTTP Request → Handler → Service → Repository → Database
                  ↓          ↓           ↓
                 DTO    Domain Logic   Entities
```

---

## 4️⃣ INVENTARIO COMPLETO DE ENDPOINTS

### Tabla Maestra de Endpoints

| # | Método | Ruta | Handler | Auth | Role | PostgreSQL | MongoDB | S3 | RabbitMQ | Worker Dependency |
|---|--------|------|---------|------|------|------------|---------|----|---------|--------------------|
| 1 | GET | `/health` | HealthHandler.Check | ❌ | - | ✅ | ✅ | ✅ | ❌ | ❌ |
| 2 | GET | `/v1/materials` | MaterialHandler.ListMaterials | ✅ | Any | `materials` | ❌ | ❌ | ❌ | ❌ |
| 3 | POST | `/v1/materials` | MaterialHandler.CreateMaterial | ✅ | Teacher+ | `materials` | ❌ | ❌ | ✅ `material.uploaded` | ❌ |
| 4 | GET | `/v1/materials/:id` | MaterialHandler.GetMaterial | ✅ | Any | `materials` | ❌ | ❌ | ❌ | ❌ |
| 5 | PUT | `/v1/materials/:id` | MaterialHandler.UpdateMaterial | ✅ | Teacher+ | `materials` | ❌ | ❌ | ❌ | ❌ |
| 6 | GET | `/v1/materials/:id/versions` | MaterialHandler.GetMaterialWithVersions | ✅ | Any | `materials`, `material_versions` | ❌ | ❌ | ❌ | ❌ |
| 7 | POST | `/v1/materials/:id/upload-url` | MaterialHandler.GenerateUploadURL | ✅ | Teacher+ | `materials` | ❌ | ✅ | ❌ | ❌ |
| 8 | GET | `/v1/materials/:id/download-url` | MaterialHandler.GenerateDownloadURL | ✅ | Any | `materials` | ❌ | ✅ | ❌ | ❌ |
| 9 | POST | `/v1/materials/:id/upload-complete` | MaterialHandler.NotifyUploadComplete | ✅ | Teacher+ | `materials` | ❌ | ❌ | ✅ `material.uploaded` | ❌ |
| 10 | GET | `/v1/materials/:id/summary` | SummaryHandler.GetSummary | ✅ | Any | ❌ | `material_summary` | ❌ | ❌ | ✅ **CRÍTICO** |
| 11 | GET | `/v1/materials/:id/assessment` | AssessmentHandler.GetMaterialAssessment | ✅ | Any | `assessment` | `material_assessment_worker` | ❌ | ❌ | ✅ **CRÍTICO** |
| 12 | POST | `/v1/materials/:id/assessment/attempts` | AssessmentHandler.CreateMaterialAttempt | ✅ | Any | `assessment`, `assessment_attempt`, `assessment_attempt_answer` | `material_assessment_worker` | ❌ | ❌ | ✅ **CRÍTICO** |
| 13 | GET | `/v1/attempts/:id/results` | AssessmentHandler.GetAttemptResults | ✅ | Own | `assessment_attempt`, `assessment_attempt_answer` | ❌ | ❌ | ❌ | ❌ |
| 14 | GET | `/v1/users/me/attempts` | AssessmentHandler.GetUserAttemptHistory | ✅ | Own | `assessment_attempt` | ❌ | ❌ | ❌ | ❌ |
| 15 | PUT | `/v1/progress` | ProgressHandler.UpsertProgress | ✅ | Own/Admin | `material_progress` | ❌ | ❌ | ✅ `material.completed` (si 100%) | ❌ |
| 16 | GET | `/v1/materials/:id/stats` | StatsHandler.GetMaterialStats | ✅ | Any | `material_progress`, `assessment_attempt` | ❌ | ❌ | ❌ | ❌ |
| 17 | GET | `/v1/stats/global` | StatsHandler.GetGlobalStats | ✅ | Admin+ | `materials`, `assessment_attempt` | ❌ | ❌ | ❌ | ❌ |
| 18 | GET | `/metrics` | Prometheus | ❌ | - | ❌ | ❌ | ❌ | ❌ | ❌ |

### Leyenda
- **Auth:** Requiere JWT Bearer Token
- **Role:** Restricción de rol (Any=cualquiera autenticado, Own=solo el propio usuario, Teacher+=teacher/admin/super_admin, Admin+=admin/super_admin)
- **Worker Dependency:** ⚠️ Endpoints que NO funcionarán si el worker no ha procesado el material

---

## 5️⃣ ANÁLISIS DETALLADO POR ENDPOINT

### 🏥 Health Check
```
GET /health
```
- **DTOs:** HealthResponse, DetailedHealthResponse
- **Validaciones:** Ninguna
- **Auth:** NO
- **PostgreSQL:** Ping connection
- **MongoDB:** Ping connection
- **RabbitMQ:** Check connection (opcional)
- **S3:** Check bucket access (opcional)
- **Response:**
  - 200: Sistema saludable
  - 503: Algún componente no disponible

### 📚 Materials - Listar

```
GET /v1/materials
```
- **Handler:** MaterialHandler.ListMaterials
- **Service:** MaterialService.ListMaterials
- **Repository:** MaterialRepository.List
- **DTOs:**
  - Response: `[]MaterialResponse`
- **Validaciones:** Ninguna (sin filtros por ahora)
- **Auth:** JWT requerido
- **PostgreSQL:**
  - SELECT de tabla `materials`
  - Campos: id, school_id, uploaded_by_teacher_id, academic_unit_id, title, description, subject, grade, file_url, file_type, file_size_bytes, status, is_public, processing_started_at, processing_completed_at, created_at, updated_at, deleted_at
- **MongoDB:** NO
- **RabbitMQ:** NO
- **Worker Dependency:** NO

### 📚 Materials - Crear

```
POST /v1/materials
```
- **Handler:** MaterialHandler.CreateMaterial
- **Service:** MaterialService.CreateMaterial
- **Repository:** MaterialRepository.Create
- **DTOs:**
  - Request: `CreateMaterialRequest` (title, description, subject, grade)
  - Response: `MaterialResponse`
- **Validaciones:**
  - title: required, min=3, max=200
  - description: max=1000
- **Auth:** JWT + RequireTeacher middleware
- **Claims extraídos del JWT:**
  - user_id (author)
  - school_id (contexto)
- **PostgreSQL:**
  - INSERT INTO `materials` (id, school_id, uploaded_by_teacher_id, title, description, subject, grade, status='uploaded', created_at, updated_at)
- **MongoDB:** NO
- **RabbitMQ:** ✅ Emite evento `material.uploaded` después de upload completo (NO aquí)
- **Worker Dependency:** NO
- **Notas:** Material se crea con status='uploaded', file_url=NULL

### 📚 Materials - Actualizar (NUEVO)

```
PUT /v1/materials/:id
```
- **Handler:** MaterialHandler.UpdateMaterial
- **Service:** MaterialService.UpdateMaterial
- **Repository:** MaterialRepository.Update
- **DTOs:**
  - Request: `UpdateMaterialRequest` (todos campos opcionales: title, description, subject, grade, academic_unit_id, is_public)
  - Response: `MaterialResponse`
- **Validaciones:**
  - title: min=3, max=200 (si presente)
  - description: max=1000 (si presente)
- **Auth:** JWT + RequireTeacher middleware
- **Autorización:** Solo el teacher que subió el material puede actualizarlo
- **PostgreSQL:**
  - SELECT FROM `materials` WHERE id = $1 (verificar ownership)
  - UPDATE `materials` SET ... WHERE id = $1
- **MongoDB:** NO
- **RabbitMQ:** NO
- **Worker Dependency:** NO
- **Notas:** Endpoint agregado en PR #97 (rama dev)

### 📚 Materials - Generar URL Upload S3

```
POST /v1/materials/:id/upload-url
```
- **Handler:** MaterialHandler.GenerateUploadURL
- **Service:** S3Storage.GeneratePresignedUploadURL
- **DTOs:**
  - Request: `GenerateUploadURLRequest` (file_name, content_type)
  - Response: `GenerateUploadURLResponse` (upload_url, file_url, expires_in)
- **Validaciones:**
  - file_name: no debe contener "..", "/", "\" (path traversal)
  - content_type: required
- **Auth:** JWT + RequireTeacher middleware
- **PostgreSQL:**
  - SELECT FROM `materials` WHERE id = $1 (verificar que existe)
- **MongoDB:** NO
- **S3:** ✅ GeneratePresignedURL para PUT
  - Bucket: configurado en env
  - Key: `materials/{material_id}/{file_name}`
  - Expiration: 15 minutos
- **RabbitMQ:** NO
- **Worker Dependency:** NO

### 📚 Materials - Notificar Upload Completo

```
POST /v1/materials/:id/upload-complete
```
- **Handler:** MaterialHandler.NotifyUploadComplete
- **Service:** MaterialService.NotifyUploadComplete
- **Repository:** MaterialRepository.UpdateStatus
- **DTOs:**
  - Request: `UploadCompleteRequest` (file_url, file_type, file_size_bytes)
  - Response: 204 No Content
- **Validaciones:** Ninguna
- **Auth:** JWT + RequireTeacher middleware
- **PostgreSQL:**
  - UPDATE `materials` SET file_url=$1, file_type=$2, file_size_bytes=$3, status='processing', processing_started_at=NOW(), updated_at=NOW() WHERE id=$4
- **MongoDB:** NO
- **RabbitMQ:** ✅ **EMITE** `material.uploaded`
  - Exchange: edugo.events
  - Routing Key: material.uploaded
  - Payload:
    ```json
    {
      "material_id": "uuid",
      "school_id": "uuid",
      "teacher_id": "uuid",
      "file_url": "s3://...",
      "file_size_bytes": 123456,
      "file_type": "application/pdf",
      "metadata": {}
    }
    ```
- **Worker Dependency:** NO
- **Notas:**
  - Este endpoint dispara el procesamiento del worker
  - Worker escucha `material.uploaded` y procesa el PDF
  - Worker genera resumen y assessment, luego emite `assessment.generated`

### 📚 Materials - Obtener Resumen IA

```
GET /v1/materials/:id/summary
```
- **Handler:** SummaryHandler.GetSummary
- **Service:** SummaryService.GetSummary
- **Repository:** SummaryRepository.FindByMaterialID (MongoDB)
- **DTOs:** Response: estructura dinámica (map[string]interface{})
- **Validaciones:**
  - material_id: UUID válido
- **Auth:** JWT requerido
- **PostgreSQL:** NO
- **MongoDB:** ✅ **LECTURA** de colección `material_summary`
  - Filter: `{ "material_id": "uuid" }`
  - Campos esperados: material_id, summary_text, generated_at, metadata
- **RabbitMQ:** NO
- **Worker Dependency:** ✅ **CRÍTICO**
  - El resumen NO existe hasta que el worker procese el PDF
  - Si no existe: 404 Not Found
  - Frontend debe manejar este caso

### 📝 Assessments - Obtener Quiz

```
GET /v1/materials/:id/assessment
```
- **Handler:** AssessmentHandler.GetMaterialAssessment
- **Service:** AssessmentAttemptService.GetAssessmentByMaterialID
- **Repositories:**
  - AssessmentRepository.FindByMaterialID (PostgreSQL - metadata)
  - AssessmentDocumentRepository.FindByMaterialID (MongoDB - preguntas)
- **DTOs:** Response: `AssessmentResponse`
- **Validaciones:**
  - material_id: UUID válido
- **Auth:** JWT requerido
- **PostgreSQL:** ✅ **LECTURA** de tabla `assessment`
  - SELECT id, material_id, mongo_document_id, questions_count, total_questions, max_attempts, pass_threshold, time_limit_minutes, estimated_time_minutes FROM assessment WHERE material_id = $1
- **MongoDB:** ✅ **LECTURA** de colección `material_assessment_worker`
  - Filter: `{ "material_id": "uuid" }`
  - Proyección: questions (SIN respuestas correctas)
  - Estructura:
    ```json
    {
      "material_id": "uuid",
      "questions": [
        {
          "id": "q1",
          "text": "Pregunta...",
          "type": "multiple_choice",
          "options": [
            { "id": "opt1", "text": "Opción A" },
            { "id": "opt2", "text": "Opción B" }
          ]
        }
      ]
    }
    ```
- **RabbitMQ:** NO
- **Worker Dependency:** ✅ **CRÍTICO**
  - El assessment NO existe hasta que el worker procese el PDF
  - Si no existe: 404 Not Found
  - Worker genera assessment después de procesar material.uploaded

### 📝 Assessments - Crear Intento

```
POST /v1/materials/:id/assessment/attempts
```
- **Handler:** AssessmentHandler.CreateMaterialAttempt
- **Service:** AssessmentAttemptService.CreateAttempt
- **Repositories:**
  - AssessmentRepository.FindByMaterialID (PostgreSQL)
  - AssessmentDocumentRepository.FindByMaterialID (MongoDB - para respuestas correctas)
  - AttemptRepository.Save (PostgreSQL)
  - AnswerRepository.SaveBulk (PostgreSQL)
- **DTOs:**
  - Request: `CreateAttemptRequest` (answers[], time_spent_seconds)
  - Response: `AttemptResultResponse` (score, passed, feedback[])
- **Validaciones:**
  - answers: required, minItems=1
  - time_spent_seconds: required, min=1, max=7200
  - UserAnswerDTO: question_id, selected_answer_id, time_spent_seconds >= 0
- **Auth:** JWT requerido
- **Claims extraídos:**
  - user_id (student)
- **PostgreSQL:** ✅ **ESCRITURA**
  - INSERT INTO `assessment_attempt` (id, assessment_id, student_id, score, max_score, passed, time_spent_seconds, started_at, completed_at)
  - INSERT INTO `assessment_attempt_answer` (id, attempt_id, question_index, student_answer, correct_answer, is_correct, time_spent_seconds) - bulk
- **MongoDB:** ✅ **LECTURA** de `material_assessment_worker` (para validar respuestas)
- **RabbitMQ:** NO
- **Worker Dependency:** ✅ **CRÍTICO**
  - Requiere que assessment exista en MongoDB
  - El scoring se hace en servidor validando contra respuestas correctas en MongoDB

### 📝 Assessments - Resultados de Intento

```
GET /v1/attempts/:id/results
```
- **Handler:** AssessmentHandler.GetAttemptResults
- **Service:** AssessmentAttemptService.GetAttemptResult
- **Repository:** AttemptRepository.FindByIDWithAnswers (PostgreSQL)
- **DTOs:** Response: `AttemptResultResponse`
- **Validaciones:**
  - attempt_id: UUID válido
- **Auth:** JWT requerido
- **Autorización:** Solo el estudiante dueño del intento puede verlo (o admin)
- **PostgreSQL:** ✅ **LECTURA**
  - SELECT FROM `assessment_attempt` WHERE id = $1 AND student_id = $2
  - SELECT FROM `assessment_attempt_answer` WHERE attempt_id = $1
- **MongoDB:** NO
- **RabbitMQ:** NO
- **Worker Dependency:** NO

### 📝 Assessments - Historial Usuario

```
GET /v1/users/me/attempts
```
- **Handler:** AssessmentHandler.GetUserAttemptHistory
- **Service:** AssessmentAttemptService.GetAttemptHistory
- **Repository:** AttemptRepository.FindByStudentID (PostgreSQL)
- **DTOs:** Response: `AttemptHistoryResponse` (attempts[], total_count, limit, page)
- **Validaciones:**
  - limit: min=1, max=100, default=10
  - offset: min=0, default=0
- **Auth:** JWT requerido
- **Claims extraídos:**
  - user_id (student)
- **PostgreSQL:** ✅ **LECTURA**
  - SELECT COUNT(*) FROM `assessment_attempt` WHERE student_id = $1
  - SELECT id, assessment_id, student_id, score, max_score, passed, completed_at, material_id, material_title FROM `assessment_attempt` WHERE student_id = $1 ORDER BY completed_at DESC LIMIT $2 OFFSET $3
- **MongoDB:** NO
- **RabbitMQ:** NO
- **Worker Dependency:** NO

### 📈 Progress - Upsert

```
PUT /v1/progress
```
- **Handler:** ProgressHandler.UpsertProgress
- **Service:** ProgressService.UpdateProgress
- **Repository:** ProgressRepository.Upsert (PostgreSQL)
- **DTOs:**
  - Request: `UpsertProgressRequest` (user_id, material_id, progress_percentage, last_page)
  - Response: `ProgressResponse`
- **Validaciones:**
  - user_id: required, UUID
  - material_id: required, UUID
  - progress_percentage: required, min=0, max=100
  - last_page: opcional, int
- **Auth:** JWT requerido
- **Autorización:**
  - Usuario solo puede actualizar su propio progreso (user_id == JWT.user_id)
  - Excepción: Admin puede actualizar progreso de cualquiera
- **Claims extraídos:**
  - user_id (authenticated)
  - school_id (contexto)
- **PostgreSQL:** ✅ **UPSERT**
  - INSERT INTO `material_progress` (user_id, material_id, school_id, progress_percentage, last_page, updated_at)
  - ON CONFLICT (user_id, material_id, school_id) DO UPDATE SET progress_percentage=$1, last_page=$2, updated_at=NOW()
- **MongoDB:** NO
- **RabbitMQ:** ✅ **EMITE** `material.completed` (solo si progress_percentage = 100)
  - Exchange: edugo.events
  - Routing Key: material.completed
  - Payload:
    ```json
    {
      "material_id": "uuid",
      "school_id": "uuid",
      "user_id": "uuid",
      "completed_at": "2024-12-23T10:00:00Z"
    }
    ```
- **Worker Dependency:** NO
- **Notas:** Operación idempotente (UPSERT)

### 📊 Stats - Material

```
GET /v1/materials/:id/stats
```
- **Handler:** StatsHandler.GetMaterialStats
- **Service:** StatsService.GetMaterialStats
- **Repositories:**
  - MaterialRepository.FindByID (PostgreSQL)
  - ProgressRepository (consultas agregadas)
  - AttemptRepository (consultas agregadas)
- **DTOs:** Response: estructura dinámica (map[string]interface{})
- **Validaciones:**
  - material_id: UUID válido
- **Auth:** JWT requerido
- **PostgreSQL:** ✅ **LECTURA AGREGADA**
  - SELECT COUNT(*) FROM `material_progress` WHERE material_id = $1
  - SELECT AVG(progress_percentage) FROM `material_progress` WHERE material_id = $1
  - SELECT COUNT(*) FROM `material_progress` WHERE material_id = $1 AND progress_percentage = 100
  - SELECT AVG(score), COUNT(*) FROM `assessment_attempt` WHERE material_id = $1
- **MongoDB:** NO
- **RabbitMQ:** NO
- **Worker Dependency:** NO
- **Respuesta esperada:**
  ```json
  {
    "material_id": "uuid",
    "total_views": 150,
    "completion_rate": 65.5,
    "average_score": 78.2,
    "total_attempts": 45
  }
  ```

### 📊 Stats - Global

```
GET /v1/stats/global
```
- **Handler:** StatsHandler.GetGlobalStats
- **Service:** StatsService.GetGlobalStats
- **Repositories:**
  - MaterialRepository (COUNT)
  - AttemptRepository (agregados)
- **DTOs:** Response: estructura dinámica (map[string]interface{})
- **Validaciones:** Ninguna
- **Auth:** JWT + RequireAdmin middleware
- **PostgreSQL:** ✅ **LECTURA AGREGADA**
  - SELECT COUNT(*) FROM `materials` WHERE deleted_at IS NULL
  - SELECT COUNT(*) FROM `assessment_attempt` WHERE completed_at IS NOT NULL
  - SELECT AVG(score) FROM `assessment_attempt` WHERE completed_at IS NOT NULL
- **MongoDB:** NO
- **RabbitMQ:** NO
- **Worker Dependency:** NO
- **Respuesta esperada:**
  ```json
  {
    "total_materials": 250,
    "total_attempts": 1500,
    "global_average_score": 75.8
  }
  ```

---

## 6️⃣ ANÁLISIS DE CONTRATOS

### Comunicación con API Admin

#### Validación de Tokens JWT

**Método:** Validación LOCAL (preferida) con fallback a REMOTA

**Configuración:**
```yaml
# config.yaml
auth:
  jwt_secret: ${JWT_SECRET}  # MISMO secret que api-admin
  jwt_issuer: "edugo-central"

  # Validación remota (fallback)
  remote_enabled: true
  base_url: ${API_ADMIN_URL}  # http://api-admin:8082
```

**Flujo:**
1. **Validación LOCAL (rápida):**
   - api-mobile usa el MISMO JWT secret que api-admin
   - Valida firma y claims localmente
   - Sin llamada HTTP

2. **Validación REMOTA (fallback opcional):**
   - Si JWT secret no disponible
   - O si falla validación local
   - Llama a `GET {API_ADMIN_URL}/v1/auth/validate-token`
   - Circuit breaker para evitar cascading failures

**Claims esperados en JWT:**
```json
{
  "sub": "user_id (UUID)",
  "email": "user@school.edu",
  "role": "student|teacher|admin|super_admin",
  "school_id": "uuid",
  "iss": "edugo-central",
  "exp": 1234567890,
  "iat": 1234567890
}
```

**Middleware:** `RemoteAuthMiddleware`
- Ubicación: `internal/infrastructure/http/middleware/remote_auth.go`
- Extrae: user_id, email, role, school_id
- Almacena en contexto Gin para handlers

#### Dependencias de API Admin

**QUÉ CONSUME de api-admin:**
- ✅ Tokens JWT válidos
- ✅ Claims: user_id, email, role, school_id
- ❌ **NO** consulta datos de usuarios directamente
- ❌ **NO** consulta datos de escuelas directamente

**QUÉ PRODUCE para api-admin:**
- ❌ Nada (no hay comunicación inversa)

### Comunicación con Worker

#### QUÉ CONSUME del Worker (MongoDB)

**1. Resúmenes IA**
- **Colección:** `material_summary`
- **Formato esperado:**
  ```json
  {
    "_id": "ObjectID",
    "material_id": "uuid",
    "summary_text": "Resumen generado por IA...",
    "key_points": ["punto1", "punto2"],
    "generated_at": "ISO8601",
    "model_version": "gpt-4",
    "metadata": {}
  }
  ```
- **Endpoint dependiente:** `GET /v1/materials/:id/summary`
- **Comportamiento si no existe:** 404 Not Found

**2. Assessments (Quizzes)**
- **Colección:** `material_assessment_worker`
- **Formato esperado:**
  ```json
  {
    "_id": "ObjectID",
    "material_id": "uuid",
    "questions": [
      {
        "id": "q1",
        "text": "¿Cuál es...?",
        "type": "multiple_choice",
        "options": [
          { "id": "opt1", "text": "Opción A" },
          { "id": "opt2", "text": "Opción B" },
          { "id": "opt3", "text": "Opción C" },
          { "id": "opt4", "text": "Opción D" }
        ],
        "correct_answer": "opt2",
        "explanation": "Porque...",
        "difficulty": "medium"
      }
    ],
    "generated_at": "ISO8601",
    "total_questions": 10,
    "estimated_time_minutes": 15,
    "pass_threshold": 70,
    "max_attempts": 3
  }
  ```
- **Endpoints dependientes:**
  - `GET /v1/materials/:id/assessment` (sin correct_answer)
  - `POST /v1/materials/:id/assessment/attempts` (usa correct_answer para scoring)
- **Comportamiento si no existe:** 404 Not Found

#### QUÉ PRODUCE para el Worker (RabbitMQ)

**Evento:** `material.uploaded`
- **Exchange:** `edugo.events` (type: topic)
- **Routing Key:** `material.uploaded`
- **Emitido por:** MaterialHandler.NotifyUploadComplete
- **Estructura:**
  ```json
  {
    "event_id": "uuid",
    "event_type": "material.uploaded",
    "event_version": "1.0",
    "timestamp": "2024-12-23T10:00:00Z",
    "payload": {
      "material_id": "uuid",
      "school_id": "uuid",
      "teacher_id": "uuid",
      "file_url": "s3://bucket/materials/uuid/file.pdf",
      "file_size_bytes": 123456,
      "file_type": "application/pdf",
      "metadata": {}
    }
  }
  ```

**Evento:** `material.completed`
- **Exchange:** `edugo.events` (type: topic)
- **Routing Key:** `material.completed`
- **Emitido por:** ProgressService.UpdateProgress (cuando progress=100%)
- **Estructura:**
  ```json
  {
    "event_id": "uuid",
    "event_type": "material.completed",
    "event_version": "1.0",
    "timestamp": "2024-12-23T10:05:00Z",
    "payload": {
      "material_id": "uuid",
      "school_id": "uuid",
      "user_id": "uuid",
      "completed_at": "2024-12-23T10:05:00Z"
    }
  }
  ```

**Evento ESPERADO del Worker:** `assessment.generated`
- **Exchange:** `edugo.events`
- **Routing Key:** `assessment.generated`
- **Consumido por:** (Actualmente NO hay consumer en api-mobile)
- **Estructura esperada:**
  ```json
  {
    "event_id": "uuid",
    "event_type": "assessment.generated",
    "event_version": "1.0",
    "timestamp": "2024-12-23T10:02:00Z",
    "payload": {
      "material_id": "uuid",
      "mongo_document_id": "ObjectID",
      "questions_count": 10,
      "processing_time_ms": 15000
    }
  }
  ```

**⚠️ NOTA:** api-mobile NO consume eventos del worker actualmente. Solo escribe en RabbitMQ y lee de MongoDB.

### Consistencia de DTOs

#### ✅ DTOs CONSISTENTES entre proyectos

**MaterialResponse** (compartido con api-admin):
- Mismo esquema de 15 campos
- UUIDs en formato string
- Timestamps ISO8601
- Status enum: uploaded, processing, ready, failed

**AssessmentResponse** (estructura propia):
- No se comparte directamente con api-admin
- Frontend es el único consumidor

**ProgressResponse** (estructura propia):
- No se comparte directamente con api-admin
- Frontend es el único consumidor

#### ⚠️ POSIBLES INCONSISTENCIAS

**UserInfo en JWT vs UserResponse de api-admin:**
- api-mobile NO consulta endpoint `/users/:id` de api-admin
- Solo usa claims del JWT
- Si JWT está desactualizado, puede haber inconsistencia de role o email

**Recomendación:**
- Implementar mecanismo de invalidación de JWT cuando usuario cambia de rol
- O endpoint en api-admin para validar claims actuales

---

## 7️⃣ RESPONSABILIDAD DE BASE DE DATOS

### ✅ PROYECTO CORRECTO - SIN MIGRACIONES PROPIAS

#### Archivos de Migración Encontrados
```
❌ NO HAY MIGRACIONES EN ESTE PROYECTO
```
Solo encontrados:
- `.idea/copilot.data.migration.*` (archivos del IDE, no del proyecto)

#### Uso de Infraestructura Compartida

**go.mod dependencies:**
```go
github.com/EduGoGroup/edugo-infrastructure/postgres v0.13.0
github.com/EduGoGroup/edugo-infrastructure/schemas v0.1.1
```

**Entidades utilizadas:**
- Material (PostgreSQL)
- MaterialVersion (PostgreSQL)
- Assessment (PostgreSQL)
- AssessmentAttempt (PostgreSQL)
- AssessmentAttemptAnswer (PostgreSQL)
- MaterialProgress (PostgreSQL)
- User (PostgreSQL)

**Repositorios:**
- Implementan interfaces del domain
- Usan entidades de `edugo-infrastructure/postgres`
- **NO definen estructura de tablas**
- **NO ejecutan CREATE TABLE**
- Solo queries CRUD

#### Verificación de Entidades vs Infraestructura

**Ejemplo:** MaterialRepository

```go
// internal/infrastructure/persistence/postgres/repository/material_repository_impl.go
import (
    "github.com/EduGoGroup/edugo-infrastructure/postgres"
)

func (r *MaterialRepositoryImpl) Create(ctx context.Context, material *postgres.Material) error {
    return r.db.WithContext(ctx).Create(material).Error
}
```

**✅ CORRECTO:** Usa entidad de infraestructura, no define su propia estructura.

#### Colecciones MongoDB

**Colecciones accedidas:**
1. `material_assessment_worker` - Creada por worker
2. `material_summary` - Creada por worker

**✅ CORRECTO:** api-mobile solo LECTURA, worker define estructura.

### 🎯 CONCLUSIÓN: RESPONSABILIDAD CORRECTA

- ✅ NO hay migraciones propias
- ✅ Usa entidades de `edugo-infrastructure`
- ✅ NO define estructura de tablas
- ✅ Respeta separación de responsabilidades
- ✅ MongoDB: solo lectura de colecciones del worker

**NO HAY BANDERAS CRÍTICAS**

---

## 8️⃣ ANÁLISIS DE EVENTOS EMITIDOS

### Eventos Definidos (RabbitMQ)

**Archivo:** `internal/infrastructure/messaging/rabbitmq/events.go`

#### 1. material.uploaded

**Cuándo se emite:** Después de que docente sube PDF a S3 y notifica completitud

**Handler que emite:** MaterialHandler.NotifyUploadComplete

**Service que publica:** MaterialService.NotifyUploadComplete

**Estructura:**
```json
{
  "event_id": "uuid (generado)",
  "event_type": "material.uploaded",
  "event_version": "1.0",
  "timestamp": "2024-12-23T10:00:00Z",
  "payload": {
    "material_id": "uuid",
    "school_id": "uuid",
    "teacher_id": "uuid",
    "file_url": "materials/uuid/file.pdf",
    "file_size_bytes": 123456,
    "file_type": "application/pdf",
    "metadata": {}
  }
}
```

**Consumidores esperados:**
- ✅ edugo-worker (procesa PDF, genera resumen y assessment)

**Documentación:**
- ✅ Definido en código
- ✅ Estructura documentada en comments
- ⚠️ No hay documentación externa (README, wiki)

#### 2. material.completed

**Cuándo se emite:** Cuando estudiante completa 100% de progreso en un material

**Service que publica:** ProgressService.UpdateProgress (condición: progress_percentage == 100)

**Estructura:**
```json
{
  "event_id": "uuid (generado)",
  "event_type": "material.completed",
  "event_version": "1.0",
  "timestamp": "2024-12-23T10:05:00Z",
  "payload": {
    "material_id": "uuid",
    "school_id": "uuid",
    "user_id": "uuid",
    "completed_at": "2024-12-23T10:05:00Z"
  }
}
```

**Consumidores esperados:**
- ⚠️ No hay consumidor actual (posible: analytics, gamification futura)

**Documentación:**
- ✅ Definido en código
- ✅ Estructura documentada en comments
- ⚠️ No hay documentación externa

#### 3. assessment.generated (ESPERADO, no emitido)

**Evento esperado del worker:**

**Estructura:**
```json
{
  "event_id": "uuid",
  "event_type": "assessment.generated",
  "event_version": "1.0",
  "timestamp": "2024-12-23T10:02:00Z",
  "payload": {
    "material_id": "uuid",
    "mongo_document_id": "ObjectID",
    "questions_count": 10,
    "processing_time_ms": 15000
  }
}
```

**⚠️ NOTA:** api-mobile NO consume este evento actualmente. Solo detecta assessment mediante polling (GET /assessment).

### Publisher: RabbitMQPublisher

**Características:**
- ✅ Publisher confirms habilitado
- ✅ Persistent messages (DeliveryMode: amqp.Persistent)
- ✅ Timeout de confirmación: 5 segundos
- ✅ Propaga request_id en headers AMQP para tracing distribuido
- ✅ Resilient publisher con retry logic (resilient_publisher.go)

**Configuración:**
```yaml
rabbitmq:
  url: ${RABBITMQ_URL}
  exchange: "edugo.events"
  exchange_type: "topic"
```

**Headers propagados:**
```
X-Request-ID: {request_id from context}
```

### Documentación de Eventos

**Estado actual:**
- ✅ Código bien estructurado
- ✅ Factory functions para crear eventos
- ✅ JSON serialization
- ⚠️ **FALTA:** Documentación centralizada de eventos del ecosistema
- ⚠️ **FALTA:** Schema registry para validación
- ⚠️ **FALTA:** Versionado de eventos

**Recomendación:**
- Crear documento `EVENTS.md` con todos los eventos del ecosistema
- Especificar consumers esperados
- Documentar compatibilidad entre versiones

---

## 9️⃣ ENDPOINT DE SALUD

### GET /health

**Ubicación:** `internal/infrastructure/http/handler/health_handler.go`

**Handler:** HealthHandler.Check

**Características:**

#### Modo Simple (default)
```
GET /health
```

**Response 200:**
```json
{
  "status": "healthy",
  "service": "edugo-api-mobile",
  "version": "v0.15.0",
  "postgres": "connected",
  "mongodb": "connected",
  "timestamp": "2024-12-23T10:00:00Z"
}
```

**Response 503:**
```json
{
  "status": "unhealthy",
  "service": "edugo-api-mobile",
  "version": "v0.15.0",
  "postgres": "disconnected",
  "mongodb": "connected",
  "timestamp": "2024-12-23T10:00:00Z"
}
```

#### Modo Detallado
```
GET /health?detail=1
```

**Response 200:**
```json
{
  "status": "healthy",
  "service": "edugo-api-mobile",
  "version": "v0.15.0",
  "timestamp": "2024-12-23T10:00:00Z",
  "total_time": "15ms",
  "components": {
    "postgres": {
      "status": "healthy",
      "latency": "5ms",
      "optional": false,
      "error": null
    },
    "mongodb": {
      "status": "healthy",
      "latency": "8ms",
      "optional": false,
      "error": null
    },
    "rabbitmq": {
      "status": "healthy",
      "latency": "2ms",
      "optional": true,
      "error": null
    },
    "s3": {
      "status": "healthy",
      "latency": "10ms",
      "optional": true,
      "error": null
    }
  }
}
```

**Componentes verificados:**
- ✅ PostgreSQL (REQUIRED) - Ping connection
- ✅ MongoDB (REQUIRED) - Ping connection
- ✅ RabbitMQ (OPTIONAL) - Connection status
- ✅ S3 (OPTIONAL) - Bucket access

**Lógica de estado:**
- `healthy`: Todos los componentes REQUIRED están OK
- `unhealthy`: Al menos un componente REQUIRED falló
- Componentes OPTIONAL no afectan el status general

**Timeout por componente:** 3 segundos

**Health check paralelo:** Todos los checks se ejecutan concurrentemente

### Integración con Kubernetes/Docker

**Liveness Probe:**
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
```

**Readiness Probe:**
```yaml
readinessProbe:
  httpGet:
    path: /health?detail=1
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

---

## 🔟 DEPENDENCIAS DEL WORKER

### Endpoints BLOQUEADOS sin Worker

| Endpoint | Comportamiento sin Worker | Impacto Frontend |
|----------|---------------------------|------------------|
| `GET /v1/materials/:id/summary` | **404 Not Found** | ⚠️ ALTO - Feature completa no disponible |
| `GET /v1/materials/:id/assessment` | **404 Not Found** | ⚠️ ALTO - Feature completa no disponible |
| `POST /v1/materials/:id/assessment/attempts` | **404 Not Found** (no assessment) | ⚠️ ALTO - Feature completa no disponible |

### Flujo Completo con Worker

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      FLUJO MATERIAL + WORKER                                 │
└─────────────────────────────────────────────────────────────────────────────┘

1. DOCENTE crea material
   POST /v1/materials
   → Material creado con status='uploaded', file_url=NULL

2. DOCENTE genera URL de subida
   POST /v1/materials/:id/upload-url
   → Recibe presigned URL S3

3. DOCENTE sube PDF a S3
   PUT https://s3.amazonaws.com/...
   → PDF almacenado en S3

4. DOCENTE notifica completitud
   POST /v1/materials/:id/upload-complete
   → Material actualizado: status='processing', file_url='s3://...'
   → api-mobile EMITE evento RabbitMQ: material.uploaded

5. WORKER consume evento
   → Descarga PDF de S3
   → Procesa con IA (OpenAI/Claude)
   → Genera:
     a) Resumen → MongoDB collection 'material_summary'
     b) Assessment → MongoDB collection 'material_assessment_worker'
   → Actualiza PostgreSQL table 'assessment' con metadata
   → Actualiza PostgreSQL 'materials' status='ready'
   → EMITE evento: assessment.generated

6. ESTUDIANTE puede consumir
   GET /v1/materials/:id/summary → ✅ 200 OK
   GET /v1/materials/:id/assessment → ✅ 200 OK
   POST /v1/materials/:id/assessment/attempts → ✅ 201 Created
```

### Tiempo de Procesamiento Esperado

**Variables:**
- Tamaño del PDF
- Número de páginas
- Complejidad del contenido
- Carga del worker

**Estimación:**
- PDF de 10 páginas: ~30-60 segundos
- PDF de 50 páginas: ~2-5 minutos
- PDF de 100 páginas: ~5-10 minutos

### Estrategias Frontend para Manejar Procesamiento

#### 1. Polling Status Material
```typescript
async function waitForMaterialReady(materialId: string): Promise<void> {
  const maxAttempts = 60; // 5 minutos con polling cada 5s

  for (let i = 0; i < maxAttempts; i++) {
    const material = await api.getMaterial(materialId);

    if (material.status === 'ready') {
      return; // Material procesado
    }

    if (material.status === 'failed') {
      throw new Error('Material processing failed');
    }

    await sleep(5000); // Esperar 5 segundos
  }

  throw new Error('Timeout waiting for material');
}
```

#### 2. Verificar Existencia de Assessment
```typescript
async function getAssessmentIfAvailable(materialId: string) {
  try {
    return await api.getAssessment(materialId);
  } catch (error) {
    if (error.status === 404) {
      return null; // Assessment aún no generado
    }
    throw error;
  }
}
```

#### 3. UI Progressive Enhancement
```typescript
// Mostrar material inmediatamente
const material = await api.getMaterial(id);
displayMaterial(material);

// Intentar cargar assessment (puede fallar)
try {
  const assessment = await api.getAssessment(id);
  displayAssessment(assessment);
} catch {
  showMessage("El quiz estará disponible en unos minutos...");
}

// Intentar cargar resumen (puede fallar)
try {
  const summary = await api.getSummary(id);
  displaySummary(summary);
} catch {
  showMessage("El resumen estará disponible en unos minutos...");
}
```

### WebSockets / Server-Sent Events (FUTURO)

**⚠️ NO IMPLEMENTADO actualmente**

**Recomendación futura:**
```
WS /v1/materials/:id/processing-status
→ Emite eventos de progreso en tiempo real
```

**Eventos esperados:**
```json
{"type": "processing_started", "progress": 0}
{"type": "pdf_parsed", "progress": 25}
{"type": "summary_generated", "progress": 50}
{"type": "assessment_generated", "progress": 75}
{"type": "completed", "progress": 100, "status": "ready"}
```

---

## 1️⃣1️⃣ BANDERAS CRÍTICAS

### ✅ NO HAY VIOLACIONES DE RESPONSABILIDAD BD

#### Verificaciones Realizadas

**1. Búsqueda de Migraciones:**
```bash
find . -name "*migration*" -o -name "*migrate*"
```
**Resultado:** Solo archivos del IDE, ninguna migración SQL.

**2. Verificación CREATE TABLE:**
```bash
grep -r "CREATE TABLE" internal/
```
**Resultado:** Sin coincidencias.

**3. Verificación ALTER TABLE:**
```bash
grep -r "ALTER TABLE" internal/
```
**Resultado:** Sin coincidencias.

**4. Verificación DROP TABLE:**
```bash
grep -r "DROP TABLE" internal/
```
**Resultado:** Sin coincidencias.

**5. Uso de Infraestructura Compartida:**
```go
// go.mod
github.com/EduGoGroup/edugo-infrastructure/postgres v0.13.0
github.com/EduGoGroup/edugo-infrastructure/schemas v0.1.1
```
✅ **CORRECTO:** Usa entidades de infraestructura.

**6. Colecciones MongoDB:**
- `material_assessment_worker` - ✅ Creada por worker, api-mobile solo lectura
- `material_summary` - ✅ Creada por worker, api-mobile solo lectura

### 🎯 CONCLUSIÓN: ARQUITECTURA CORRECTA

- ✅ Separación clara de responsabilidades
- ✅ API Mobile NO define estructura de BD
- ✅ Proyecto Infrastructure centraliza esquemas
- ✅ Worker define colecciones MongoDB que solo lee api-mobile
- ✅ Clean Architecture implementada correctamente

**NO HAY BANDERAS ROJAS**

---

## 1️⃣2️⃣ RECOMENDACIONES PARA FRONTEND

### 🟢 LO QUE ESTÁ LISTO

#### 1. Autenticación
- ✅ JWT Bearer Token
- ✅ Claims documentados
- ✅ Middleware de autenticación funcional
- ✅ Validación contra api-admin

**Implementación frontend:**
```typescript
const api = axios.create({
  baseURL: 'http://localhost:8080',
  headers: {
    'Authorization': `Bearer ${token}` // Token de api-admin
  }
});
```

#### 2. CRUD Materiales
- ✅ Listar materiales
- ✅ Crear material
- ✅ Actualizar material (NUEVO en dev)
- ✅ Obtener material específico
- ✅ Historial de versiones

**Flujo completo documentado:**
1. Login en api-admin → obtener JWT
2. Crear material → recibir material_id
3. Generar URL upload → recibir presigned URL
4. Upload PDF a S3
5. Notificar completitud → dispara worker
6. Polling status hasta 'ready'

#### 3. Progreso de Lectura
- ✅ UPSERT idempotente
- ✅ Validación de ownership
- ✅ Admin puede actualizar progreso de cualquier usuario

#### 4. Estadísticas
- ✅ Stats por material (público)
- ✅ Stats globales (solo admin)

### 🟡 LO QUE REQUIERE PRECAUCIÓN

#### 1. Assessments y Summaries
**⚠️ DEPENDEN DEL WORKER**

**Estrategia recomendada:**
```typescript
interface Material {
  id: string;
  status: 'uploaded' | 'processing' | 'ready' | 'failed';
  // ... otros campos
}

// Verificar status antes de intentar cargar assessment
if (material.status === 'ready') {
  // Intento seguro
  const assessment = await api.getAssessment(material.id);
} else if (material.status === 'processing') {
  showMessage("El material está siendo procesado...");
  // Iniciar polling
} else if (material.status === 'failed') {
  showError("Error al procesar el material");
}
```

#### 2. Manejo de Errores 404
```typescript
async function getAssessmentSafely(materialId: string) {
  try {
    return await api.getAssessment(materialId);
  } catch (error) {
    if (error.response?.status === 404) {
      // Normal: assessment aún no generado
      return { available: false };
    }
    // Error real
    throw error;
  }
}
```

#### 3. Diferencias dev vs main
**⚠️ USAR RAMA DEV PARA DESARROLLO**

**Endpoints SOLO en dev:**
- `PUT /v1/materials/:id` - Actualizar material

**Verificar con backend qué rama está en cada ambiente:**
- Dev environment → rama `dev`
- Staging/Prod → confirmar si está en `main` o `dev`

### 🔴 LO QUE FALTA IMPLEMENTAR

#### 1. WebSockets para Progreso en Tiempo Real
**Estado:** ❌ No implementado

**Workaround actual:**
```typescript
// Polling manual cada 5 segundos
const interval = setInterval(async () => {
  const material = await api.getMaterial(id);
  if (material.status === 'ready') {
    clearInterval(interval);
    loadAssessment(id);
  }
}, 5000);
```

#### 2. Paginación en ListMaterials
**Estado:** ⚠️ Parcialmente implementado

**Actual:**
```
GET /v1/materials
→ Retorna TODOS los materiales (sin filtros)
```

**Recomendado para futuro:**
```
GET /v1/materials?page=1&limit=20&school_id=xxx&subject=Math
```

#### 3. Cache de Validaciones JWT
**Estado:** ✅ Implementado en backend, transparente para frontend

**Nota:** Backend cachea validaciones por 60 segundos. Frontend no necesita manejar esto.

### 📋 Checklist de Integración

#### Antes de Empezar
- [ ] Verificar qué rama está desplegada en el ambiente de desarrollo
- [ ] Obtener credenciales JWT de api-admin
- [ ] Revisar Swagger UI: `http://localhost:8080/swagger/index.html`
- [ ] Verificar health check: `http://localhost:8080/health?detail=1`

#### Implementación por Feature

**Materials:**
- [ ] Implementar lista de materiales
- [ ] Implementar creación de material
- [ ] Implementar flujo de upload S3 (generar URL → upload → notificar)
- [ ] Implementar polling de status (uploaded → processing → ready)
- [ ] Implementar actualización de material (solo en dev)
- [ ] Implementar descarga de PDF (generar URL presignada)

**Assessments:**
- [ ] Verificar status='ready' antes de intentar cargar
- [ ] Implementar manejo de 404 (assessment no disponible)
- [ ] Implementar creación de intento
- [ ] Implementar visualización de resultados con feedback
- [ ] Implementar historial de intentos (paginado)

**Progress:**
- [ ] Implementar actualización de progreso (UPSERT)
- [ ] Implementar detección de 100% para evento 'completed'

**Stats:**
- [ ] Implementar stats por material (accesible a todos)
- [ ] Implementar stats globales (solo admin)

**Error Handling:**
- [ ] 401 Unauthorized → redirigir a login
- [ ] 403 Forbidden → mostrar mensaje de permisos
- [ ] 404 Not Found → distinguir entre recurso inexistente vs processing
- [ ] 500 Internal Server Error → mostrar error genérico

---

## 📊 RESUMEN DE TABLAS Y COLECCIONES

### PostgreSQL (Lectura/Escritura)

| Tabla | Operaciones | Endpoints |
|-------|-------------|-----------|
| `materials` | SELECT, INSERT, UPDATE | Todos los endpoints /materials |
| `material_versions` | SELECT | GET /materials/:id/versions |
| `assessment` | SELECT, INSERT, UPDATE | GET /assessment, POST /attempts |
| `assessment_attempt` | SELECT, INSERT | POST /attempts, GET /attempts/:id/results, GET /users/me/attempts |
| `assessment_attempt_answer` | SELECT, INSERT | POST /attempts, GET /attempts/:id/results |
| `material_progress` | SELECT, INSERT, UPDATE (UPSERT) | PUT /progress, GET /stats |
| `users` | SELECT | (Indirecto, via JWT) |
| `login_attempts` | INSERT | (Posiblemente legacy, no usado) |
| `refresh_tokens` | (Probablemente legacy, no usado) |

### MongoDB (Solo Lectura)

| Colección | Operaciones | Endpoints | Creada Por |
|-----------|-------------|-----------|------------|
| `material_assessment_worker` | FindOne | GET /assessment, POST /attempts | Worker |
| `material_summary` | FindOne | GET /summary | Worker |

### Eventos RabbitMQ (Escritura)

| Evento | Routing Key | Emitido Por |
|--------|-------------|-------------|
| `material.uploaded` | material.uploaded | POST /upload-complete |
| `material.completed` | material.completed | PUT /progress (si 100%) |

---

## 🎯 CONCLUSIÓN FINAL

### Estado General: ⚠️ PARCIALMENTE LISTO CON CONDICIONES

#### ✅ ASPECTOS POSITIVOS

1. **Arquitectura Sólida**
   - Clean Architecture bien implementada
   - Separación de responsabilidades clara
   - Uso correcto de infraestructura compartida

2. **API Completa y Documentada**
   - 18 endpoints funcionales
   - Swagger actualizado
   - DTOs bien definidos

3. **Integración Correcta**
   - Validación JWT con api-admin
   - Eventos RabbitMQ definidos
   - Worker integration clara

4. **Health Checks Robustos**
   - Modo simple y detallado
   - Verificación de todos los componentes
   - Listo para Kubernetes

#### ⚠️ ASPECTOS A CONSIDERAR

1. **Ramas Desincronizadas**
   - dev adelantada vs main
   - Frontend debe apuntar a dev
   - Necesario sync antes de producción

2. **Dependencia Crítica del Worker**
   - Assessments y Summaries NO disponibles sin worker
   - Frontend debe manejar 404s gracefully
   - Implementar polling para status

3. **Falta de Tiempo Real**
   - Sin WebSockets para progreso
   - Polling manual necesario
   - Experiencia UX mejorable

### Recomendación Final: ✅ FRONTEND PUEDE COMENZAR

**CON LAS SIGUIENTES CONDICIONES:**

1. **Apuntar a rama `dev` para desarrollo**
2. **Implementar manejo robusto de errores:**
   - 404 en /assessment → "Procesando..."
   - 404 en /summary → "Generando resumen..."
   - Polling de material.status
3. **Documentar dependencias del worker en la UI:**
   - "El quiz estará disponible en unos minutos"
   - Indicador de progreso o spinner
4. **Coordinar con DevOps:**
   - Confirmar rama desplegada en cada ambiente
   - Verificar configuración de worker
5. **Testing end-to-end:**
   - Flujo completo: crear material → upload → esperar worker → consumir assessment

### Timeline Estimado de Integración

**Semana 1:**
- Integración básica de materials (CRUD)
- Health check
- Autenticación JWT

**Semana 2:**
- Flujo de upload S3
- Progress tracking
- Stats

**Semana 3:**
- Assessments (con manejo de worker dependency)
- Historial de intentos
- Testing end-to-end

**Semana 4:**
- Polish y UX improvements
- Error handling refinado
- Performance optimization

---

## 📚 RECURSOS ADICIONALES

### Documentación del Proyecto

| Documento | Ubicación | Descripción |
|-----------|-----------|-------------|
| README principal | `/documents/README.md` | Overview del proyecto |
| Arquitectura | `/documents/ARCHITECTURE.md` | Arquitectura detallada |
| Base de Datos | `/documents/DATABASE.md` | Esquemas y relaciones |
| API Reference | `/documents/API-REFERENCE.md` | Documentación de endpoints |
| Setup | `/documents/SETUP.md` | Guía de instalación |
| Flows | `/documents/FLOWS.md` | Diagramas de flujo |

### Swagger UI
```
http://localhost:8080/swagger/index.html
```

### Repositorios Relacionados
- `edugo-infrastructure` - Esquemas y migraciones
- `edugo-shared` - Utilidades compartidas
- `edugo-api-admin` - Autenticación centralizada
- `edugo-worker` - Procesamiento IA

### Contactos Sugeridos
- **Backend Lead:** Para dudas de implementación
- **DevOps:** Para configuración de ambientes
- **Product:** Para priorización de features

---

**Fin del Informe**

*Generado el 2025-12-24 por análisis ULTRATHINK*
