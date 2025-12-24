# Analisis de Bases de Datos - Ecosistema EduGo

**Fecha de analisis:** 2025-12-23
**Proyectos analizados:** edugo-infrastructure, edugo-api-administracion, edugo-api-mobile, edugo-worker

---

## 1. PostgreSQL

### 1.1 Inventario de Tablas

| Tabla | Descripcion | PK | FKs | Indices | Estado de Uso |
|-------|-------------|----|----|---------|---------------|
| `users` | Usuarios del sistema (admin, teacher, student, guardian) | `id` (UUID) | `school_id -> schools.id` | email (unique), school_id | **ACTIVO** |
| `schools` | Instituciones educativas | `id` (UUID) | - | code (unique), name | **ACTIVO** |
| `academic_units` | Unidades academicas (grados, secciones) | `id` (UUID) | `school_id -> schools.id`, `parent_unit_id -> academic_units.id` | school_id+code (unique) | **ACTIVO** |
| `memberships` | Relacion usuario-unidad academica | `id` (UUID) | `user_id -> users.id`, `school_id -> schools.id`, `academic_unit_id -> academic_units.id` | user_id, school_id, academic_unit_id | **ACTIVO** |
| `materials` | Materiales educativos | `id` (UUID) | `school_id -> schools.id`, `uploaded_by_teacher_id -> users.id`, `academic_unit_id -> academic_units.id` | school_id, status | **ACTIVO** |
| `material_versions` | Versiones de materiales | `id` (UUID) | `material_id -> materials.id`, `changed_by -> users.id` | material_id | **ACTIVO** |
| `assessment` | Evaluaciones (metadata) | `id` (UUID) | `material_id -> materials.id` | material_id (unique) | **ACTIVO** |
| `assessment_attempt` | Intentos de evaluacion | `id` (UUID) | `assessment_id -> assessment.id`, `student_id -> users.id` | assessment_id, student_id, idempotency_key | **ACTIVO** |
| `assessment_attempt_answer` | Respuestas de intentos | `id` (UUID) | `attempt_id -> assessment_attempt.id` | attempt_id | **ACTIVO** |
| `subjects` | Materias/Asignaturas | `id` (UUID) | - | name | **ACTIVO** |
| `units` | Unidades organizativas | `id` (UUID) | `school_id -> schools.id`, `parent_unit_id -> units.id` | school_id | **ACTIVO** |
| `guardian_relations` | Relaciones tutor-estudiante | `id` (UUID) | `guardian_id -> users.id`, `student_id -> users.id`, `created_by -> users.id` | guardian_id, student_id | **ACTIVO** |
| `material_progress` | Progreso de lectura de materiales | `(material_id, user_id)` compuesta | `material_id -> materials.id`, `user_id -> users.id` | - | **ACTIVO** |
| `refresh_tokens` | Tokens de refresco JWT | `id` (UUID) | `user_id -> users.id` | token_hash, user_id | **ACTIVO** |
| `login_attempts` | Intentos de login (rate limiting) | `id` (SERIAL) | - | identifier, attempted_at | **ACTIVO** |
| `user_active_context` | Contexto activo del usuario | `id` | `user_id -> users.id`, `school_id -> schools.id` | user_id (unique) | **SIN USO** |
| `user_favorites` | Favoritos del usuario | `id` | `user_id -> users.id`, `entity_id` | user_id+entity | **SIN USO** |
| `user_activity_log` | Log de actividad | `id` | `user_id -> users.id` | user_id, action, created_at | **SIN USO** |
| `feature_flags` | Feature flags globales | `id` | - | key (unique) | **SIN USO** |
| `feature_flag_overrides` | Overrides de feature flags | `id` | `flag_id -> feature_flags.id`, `school_id -> schools.id` | flag_id+school_id | **SIN USO** |

### 1.2 Detalle de Campos por Tabla

#### **users**
```sql
id               UUID PRIMARY KEY DEFAULT gen_random_uuid()
email            VARCHAR(255) NOT NULL UNIQUE
password_hash    VARCHAR(255) NOT NULL
first_name       VARCHAR(100) NOT NULL
last_name        VARCHAR(100) NOT NULL
role             VARCHAR(50) NOT NULL  -- admin, teacher, student, guardian
school_id        UUID REFERENCES schools(id)  -- nullable para super_admin
is_active        BOOLEAN DEFAULT true
email_verified   BOOLEAN DEFAULT false
created_at       TIMESTAMP DEFAULT NOW()
updated_at       TIMESTAMP DEFAULT NOW()
deleted_at       TIMESTAMP  -- soft delete
```

#### **schools**
```sql
id                UUID PRIMARY KEY DEFAULT gen_random_uuid()
name              VARCHAR(255) NOT NULL
code              VARCHAR(50) NOT NULL UNIQUE
address           TEXT
city              VARCHAR(100)
country           VARCHAR(100)
phone             VARCHAR(50)
email             VARCHAR(255)
metadata          JSONB DEFAULT '{}'
is_active         BOOLEAN DEFAULT true
subscription_tier VARCHAR(50) DEFAULT 'free'
max_teachers      INTEGER DEFAULT 10
max_students      INTEGER DEFAULT 100
created_at        TIMESTAMP DEFAULT NOW()
updated_at        TIMESTAMP DEFAULT NOW()
deleted_at        TIMESTAMP
```

#### **academic_units**
```sql
id              UUID PRIMARY KEY DEFAULT gen_random_uuid()
parent_unit_id  UUID REFERENCES academic_units(id)
school_id       UUID NOT NULL REFERENCES schools(id)
type            VARCHAR(50) NOT NULL  -- grade, section, class, course
name            VARCHAR(255) NOT NULL
code            VARCHAR(50) NOT NULL
description     TEXT
level           INTEGER DEFAULT 0
academic_year   VARCHAR(20)
metadata        JSONB DEFAULT '{}'
is_active       BOOLEAN DEFAULT true
created_at      TIMESTAMP DEFAULT NOW()
updated_at      TIMESTAMP DEFAULT NOW()
deleted_at      TIMESTAMP
UNIQUE(school_id, code)
```

#### **memberships**
```sql
id               UUID PRIMARY KEY DEFAULT gen_random_uuid()
user_id          UUID NOT NULL REFERENCES users(id)
school_id        UUID NOT NULL REFERENCES schools(id)
academic_unit_id UUID REFERENCES academic_units(id)
role             VARCHAR(50) NOT NULL  -- teacher, student, coordinator
metadata         JSONB DEFAULT '{}'
is_active        BOOLEAN DEFAULT true
enrolled_at      TIMESTAMP DEFAULT NOW()
withdrawn_at     TIMESTAMP
created_at       TIMESTAMP DEFAULT NOW()
updated_at       TIMESTAMP DEFAULT NOW()
```

#### **materials**
```sql
id                       UUID PRIMARY KEY DEFAULT gen_random_uuid()
school_id                UUID NOT NULL REFERENCES schools(id)
uploaded_by_teacher_id   UUID NOT NULL REFERENCES users(id)
academic_unit_id         UUID REFERENCES academic_units(id)
title                    VARCHAR(500) NOT NULL
description              TEXT
subject                  VARCHAR(100)
grade                    VARCHAR(50)
file_url                 TEXT NOT NULL
file_type                VARCHAR(50) NOT NULL
file_size_bytes          BIGINT NOT NULL
status                   VARCHAR(50) DEFAULT 'uploaded'  -- uploaded, processing, ready, failed
processing_started_at    TIMESTAMP
processing_completed_at  TIMESTAMP
is_public                BOOLEAN DEFAULT false
is_deleted               BOOLEAN DEFAULT false
created_at               TIMESTAMP DEFAULT NOW()
updated_at               TIMESTAMP DEFAULT NOW()
deleted_at               TIMESTAMP
```

#### **assessment**
```sql
id                  UUID PRIMARY KEY DEFAULT gen_random_uuid()
material_id         UUID NOT NULL UNIQUE REFERENCES materials(id)
mongo_document_id   VARCHAR(50) NOT NULL  -- ObjectId de MongoDB
questions_count     INTEGER NOT NULL DEFAULT 0
total_questions     INTEGER
title               VARCHAR(500)
pass_threshold      INTEGER  -- 0-100
max_attempts        INTEGER  -- NULL = ilimitado
time_limit_minutes  INTEGER  -- NULL = sin limite
status              VARCHAR(50) DEFAULT 'draft'
created_at          TIMESTAMP DEFAULT NOW()
updated_at          TIMESTAMP DEFAULT NOW()
deleted_at          TIMESTAMP
```

#### **assessment_attempt**
```sql
id                  UUID PRIMARY KEY DEFAULT gen_random_uuid()
assessment_id       UUID NOT NULL REFERENCES assessment(id)
student_id          UUID NOT NULL REFERENCES users(id)
score               DECIMAL(5,2)
max_score           DECIMAL(5,2)
time_spent_seconds  INTEGER
started_at          TIMESTAMP NOT NULL
completed_at        TIMESTAMP
created_at          TIMESTAMP DEFAULT NOW()
idempotency_key     VARCHAR(100) UNIQUE
```

#### **assessment_attempt_answer**
```sql
id                  UUID PRIMARY KEY DEFAULT gen_random_uuid()
attempt_id          UUID NOT NULL REFERENCES assessment_attempt(id) ON DELETE CASCADE
question_index      INTEGER NOT NULL
student_answer      TEXT
is_correct          BOOLEAN
points_earned       DECIMAL(5,2)
max_points          DECIMAL(5,2)
time_spent_seconds  INTEGER
answered_at         TIMESTAMP NOT NULL
created_at          TIMESTAMP DEFAULT NOW()
updated_at          TIMESTAMP DEFAULT NOW()
```

#### **material_progress**
```sql
material_id      UUID NOT NULL REFERENCES materials(id)
user_id          UUID NOT NULL REFERENCES users(id)
percentage       INTEGER NOT NULL DEFAULT 0  -- 0-100
last_page        INTEGER NOT NULL DEFAULT 0
status           VARCHAR(50) DEFAULT 'not_started'  -- not_started, in_progress, completed
last_accessed_at TIMESTAMP DEFAULT NOW()
created_at       TIMESTAMP DEFAULT NOW()
updated_at       TIMESTAMP DEFAULT NOW()
PRIMARY KEY (material_id, user_id)
```

### 1.3 Diagrama de Relaciones (Textual)

```
                              +-----------+
                              |   users   |
                              +-----------+
                                    |
            +-------+-------+-------+-------+-------+
            |       |       |       |       |       |
            v       v       v       v       v       v
        schools  memberships  materials  assessment  guardian
            |       |             |       _attempt   _relations
            |       |             |           |
            v       v             v           v
      academic   (user+     material    assessment
        _units   school+     _versions  _attempt
            |     unit)           |      _answer
            |                     |
            +---------+-----------+
                      |
                      v
               material_progress
```

**Relaciones principales:**
- `users` -> `schools` (N:1) - Usuario pertenece a escuela
- `schools` -> `academic_units` (1:N) - Escuela tiene unidades
- `academic_units` -> `academic_units` (1:N) - Jerarquia padre-hijo
- `users` + `schools` + `academic_units` -> `memberships` - Matriculacion
- `users` -> `materials` (1:N) - Profesor sube materiales
- `materials` -> `assessment` (1:1) - Material tiene evaluacion
- `assessment` -> `assessment_attempt` (1:N) - Evaluacion tiene intentos
- `assessment_attempt` -> `assessment_attempt_answer` (1:N) - Intento tiene respuestas
- `materials` + `users` -> `material_progress` - Progreso de usuario en material

### 1.4 Uso por Proyecto

#### **edugo-api-administracion**
| Repositorio | Tabla(s) | Operaciones |
|-------------|----------|-------------|
| `UserRepository` | users | Create, FindByID, FindByEmail, Update, SoftDelete, List |
| `SchoolRepository` | schools | Create, FindByID, FindByCode, Update, Delete, List |
| `AcademicUnitRepository` | academic_units | Create, FindByID, FindBySchoolID, FindByParentID, Update, SoftDelete |
| `UnitMembershipRepository` | memberships | Create, FindByID, FindByUser, FindByUnit, Update, Delete |
| `SubjectRepository` | subjects | Create, FindByID, Update, Delete, List |
| `UnitRepository` | units | Create, FindByID, Update, Delete, List |
| `GuardianRepository` | guardian_relations | CreateRelation, FindByGuardian, FindByStudent, Update, Delete |
| `MaterialRepository` | materials | Delete, Exists |
| `StatsRepository` | users, schools, subjects, guardian_relations | GetGlobalStats (solo lectura) |

#### **edugo-api-mobile**
| Repositorio | Tabla(s) | Operaciones |
|-------------|----------|-------------|
| `UserRepository` | users | FindByID, FindByEmail, Update |
| `MaterialRepository` | materials, material_versions | Create, FindByID, Update, List, FindByAuthor, UpdateStatus |
| `AssessmentRepository` | assessment | FindByID, FindByMaterialID, Save, Delete |
| `AttemptRepository` | assessment_attempt, assessment_attempt_answer | FindByID, FindByStudentAndAssessment, Save, Count |
| `AnswerRepository` | assessment_attempt_answer | FindByAttemptID, Save, FindByQuestionID |
| `ProgressRepository` | material_progress | Save, FindByMaterialAndUser, Update, Upsert, CountActiveUsers |
| `RefreshTokenRepository` | refresh_tokens | Store, FindByTokenHash, Revoke, RevokeAllByUserID, DeleteExpired |
| `LoginAttemptRepository` | login_attempts | RecordAttempt, CountFailedAttempts, IsRateLimited |
| `SummaryRepository` (MongoDB) | material_summaries | Save, FindByMaterialID, Delete, Exists |
| `AssessmentDocumentRepository` (MongoDB) | material_assessment | FindByMaterialID, FindByID, Save, Delete |

#### **edugo-worker**
| Repositorio | Tabla(s)/Colecciones | Operaciones |
|-------------|----------------------|-------------|
| `MaterialAssessmentRepository` (MongoDB) | material_assessment_worker | Create, FindByMaterialID, FindByID, Update, Delete |
| `MaterialEventRepository` (MongoDB) | material_event | Create, FindByID, Update, FindByMaterialID, FindPending |
| `MaterialSummaryRepository` (MongoDB) | material_summary | Create, FindByMaterialID, FindByID, Update, Delete |

---

## 2. MongoDB

### 2.1 Inventario de Colecciones

| Coleccion | Descripcion | Indices | Estado de Uso |
|-----------|-------------|---------|---------------|
| `material_assessment` | Preguntas de evaluaciones (api-mobile) | material_id (unique) | **ACTIVO** |
| `material_assessment_worker` | Preguntas generadas por worker | material_id (unique), created_at | **ACTIVO** |
| `material_content` | Contenido procesado de materiales | material_id (unique), version | **SIN USO DETECTADO** |
| `assessment_attempt_result` | Resultados detallados de intentos | attempt_id, user_id+assessment_id | **SIN USO DETECTADO** |
| `audit_logs` | Logs de auditoria | user_id, action, created_at (TTL 90 dias) | **SIN USO DETECTADO** |
| `notifications` | Notificaciones push | user_id, status, created_at (TTL 30 dias) | **SIN USO DETECTADO** |
| `analytics_events` | Eventos de analitica | event_type, user_id, created_at (TTL 365 dias) | **SIN USO DETECTADO** |
| `material_summary` | Resumenes de materiales (worker) | material_id (unique) | **ACTIVO** |
| `material_summaries` | Resumenes de materiales (api-mobile) | material_id | **ACTIVO** |
| `material_event` | Eventos de procesamiento | material_id, event_type, status, created_at (TTL) | **ACTIVO** |

### 2.2 Estructura de Documentos

#### **material_assessment** (api-mobile)
```json
{
  "_id": ObjectId,
  "material_id": "uuid-string",
  "title": "string",
  "questions": [
    {
      "id": "string",
      "text": "string",
      "type": "multiple_choice|true_false|short_answer",
      "options": [
        { "id": "string", "text": "string" }
      ],
      "correct_answer": "string",
      "feedback": {
        "correct": "string",
        "incorrect": "string"
      },
      "difficulty": "easy|medium|hard",
      "tags": ["string"]
    }
  ],
  "metadata": {
    "generated_by": "string",
    "generation_date": ISODate,
    "prompt_version": "string",
    "total_questions": int,
    "estimated_time_minutes": int
  },
  "version": int,
  "created_at": ISODate,
  "updated_at": ISODate
}
```

#### **material_assessment_worker** (worker)
```json
{
  "_id": ObjectId,
  "material_id": "uuid-string",
  "questions": [
    {
      "question_id": "string",
      "question_text": "string",
      "question_type": "multiple_choice|true_false|open",
      "options": [
        { "option_id": "string", "option_text": "string" }
      ],
      "correct_answer": "string",
      "explanation": "string",
      "points": int,
      "difficulty": "easy|medium|hard",
      "tags": ["string"]
    }
  ],
  "total_questions": int,
  "total_points": int,
  "version": int,
  "ai_model": "string",
  "processing_time_ms": int,
  "token_usage": {
    "prompt_tokens": int,
    "completion_tokens": int,
    "total_tokens": int
  },
  "metadata": {
    "average_difficulty": "string",
    "estimated_time_min": int,
    "source_length": int,
    "has_images": bool
  },
  "created_at": ISODate,
  "updated_at": ISODate
}
```

#### **material_summary** (worker)
```json
{
  "_id": ObjectId,
  "material_id": "uuid-string",
  "summary": "string",
  "key_points": ["string"],
  "language": "es|en|pt",
  "word_count": int,
  "version": int,
  "ai_model": "string",
  "processing_time_ms": int,
  "token_usage": {...},
  "metadata": {
    "source_length": int,
    "has_images": bool
  },
  "created_at": ISODate,
  "updated_at": ISODate
}
```

#### **material_event** (worker)
```json
{
  "_id": ObjectId,
  "event_type": "material_uploaded|material_reprocess|assessment_attempt",
  "material_id": "uuid-string",
  "user_id": "uuid-string",
  "payload": { ... },  // flexible
  "status": "pending|processing|completed|failed",
  "error_msg": "string|null",
  "stack_trace": "string|null",
  "retry_count": int,
  "processed_at": ISODate,
  "created_at": ISODate,
  "updated_at": ISODate
}
```

### 2.3 Uso por Proyecto

| Proyecto | Colecciones Usadas | Proposito |
|----------|-------------------|-----------|
| edugo-api-mobile | material_assessment, material_summaries | Consulta de preguntas y resumenes |
| edugo-worker | material_assessment_worker, material_summary, material_event | Generacion IA y auditoria |

---

## 3. Flujos de Datos

### 3.1 Flujo: Gestion de Usuarios

```
[api-admin]
     |
     v
+----------+    +------------+    +-----------+
|  Create  | -> | PostgreSQL | <- |  mobile   |
|   User   |    |   users    |    | FindByID/ |
+----------+    +------------+    |  Email    |
                     |            +-----------+
                     v
              +------------+
              | memberships|
              | (matricula)|
              +------------+
                     |
                     v
              +--------------+
              |academic_units|
              +--------------+
```

**Tablas involucradas:**
- `users` - Creacion/consulta
- `schools` - Asociacion
- `memberships` - Matriculacion en unidades
- `academic_units` - Asignacion de grado/seccion

### 3.2 Flujo: Materiales Educativos

```
[api-mobile]              [worker]                    [api-mobile]
     |                        |                            |
     v                        |                            |
+----------+                  |                            |
|  Upload  |                  |                            |
| Material |                  |                            |
+----------+                  |                            |
     |                        |                            |
     v                        v                            |
+-----------+   evento   +---------+                       |
| materials | ---------> | process |                       |
| (status=  |   NATS/    |         |                       |
| uploaded) |   Redis    +---------+                       |
+-----------+                  |                           |
                              v                            |
                    +-------------------+                  |
                    | MongoDB:          |                  |
                    | material_summary  |                  |
                    | material_assess   |                  |
                    | _worker           |                  |
                    +-------------------+                  |
                              |                            |
                              v                            |
                    +-----------+                          |
                    | materials | <------------------------+
                    | (status=  |     consulta
                    |  ready)   |
                    +-----------+
                              |
                              v
                    +-------------------+
                    | MongoDB:          |
                    | material_summaries|  <-- consulta
                    | material_assessment|
                    +-------------------+
```

**Tablas/Colecciones involucradas:**
- PostgreSQL: `materials`, `material_versions`
- MongoDB: `material_summary`, `material_assessment_worker`, `material_event`
- MongoDB (mobile): `material_summaries`, `material_assessment`

### 3.3 Flujo: Evaluaciones

```
[api-mobile]
     |
     v
+------------+     +-------------------+
| assessment | --> | MongoDB:          |
| (postgres) |     | material_assessment|
+------------+     | (preguntas)       |
     |             +-------------------+
     |                      |
     v                      v
+------------------+   +-----------+
| assessment_attempt|   | Correccion|
| (postgres)        |   | respuestas|
+------------------+   +-----------+
     |                      |
     v                      v
+------------------------+
| assessment_attempt_    |
| answer (postgres)      |
+------------------------+
     |
     v
+------------------+
| material_progress|
| (actualizar %)   |
+------------------+
```

**Datos divididos:**
- PostgreSQL: Metadata de assessment, intentos, respuestas, scores
- MongoDB: Contenido de preguntas, opciones, feedback

### 3.4 Flujo: Eventos del Worker

```
[Trigger]
    |
    v
+--------------+
| material_event|  <-- pending
| (MongoDB)     |
+--------------+
    |
    v
+---------+
| Worker  |
| Process |
+---------+
    |
    +---> [Exito] --> material_event.status = completed
    |                 --> material_summary (crear)
    |                 --> material_assessment_worker (crear)
    |                 --> materials.status = ready (Postgres)
    |
    +---> [Error] --> material_event.status = failed
                      material_event.retry_count++
                      --> Si retry < 3: reencolar
                      --> Si retry >= 3: dead letter
```

---

## 4. ANALISIS CRITICO

### 4.1 Tablas/Colecciones Sin Uso

#### PostgreSQL - Tablas SIN USO en repositorios:

| Tabla | Definida en | Problema |
|-------|-------------|----------|
| `user_active_context` | 011_create_user_active_context.sql | No existe entidad ni repositorio que la use |
| `user_favorites` | 012_create_user_favorites.sql | No existe entidad ni repositorio que la use |
| `user_activity_log` | 013_create_user_activity_log.sql | No existe entidad ni repositorio que la use |
| `feature_flags` | 014_create_feature_flags.sql | No existe entidad ni repositorio que la use |
| `feature_flag_overrides` | 015_create_feature_flag_overrides.sql | No existe entidad ni repositorio que la use |

#### MongoDB - Colecciones SIN USO detectado:

| Coleccion | Definida en | Problema |
|-----------|-------------|----------|
| `material_content` | 002_material_content.go | No hay repositorio que acceda a esta coleccion |
| `assessment_attempt_result` | 003_assessment_attempt_result.go | No hay repositorio que acceda a esta coleccion |
| `audit_logs` | 004_audit_logs.go | No hay repositorio que acceda a esta coleccion |
| `notifications` | 005_notifications.go | No hay repositorio que acceda a esta coleccion |
| `analytics_events` | 006_analytics_events.go | No hay repositorio que acceda a esta coleccion |

### 4.2 Campos Sin Uso Detectados

| Tabla | Campo | Observacion |
|-------|-------|-------------|
| `users` | `email_verified` | Se define pero no hay logica de verificacion de email implementada |
| `materials` | `is_deleted` vs `deleted_at` | Hay dos campos para soft delete, inconsistente |
| `assessment` | `deleted_at` | Definido pero Delete() hace hard delete, no soft delete |
| `schools` | `subscription_tier`, `max_teachers`, `max_students` | Definidos pero sin logica de enforcement |

### 4.3 Flujos Incompletos

1. **Verificacion de Email**
   - Campo `email_verified` existe en `users`
   - No hay endpoint ni logica para verificar email
   - Los usuarios pueden operar sin verificar

2. **Limites de Suscripcion**
   - `max_teachers` y `max_students` en `schools`
   - No hay validacion al crear usuarios o memberships
   - Se podria exceder el limite sin restriccion

3. **Soft Delete Inconsistente**
   - `materials`: usa `is_deleted` (bool) + `deleted_at` (timestamp)
   - `users`, `schools`, `academic_units`: usan solo `deleted_at`
   - `assessment`: tiene `deleted_at` pero repositorio hace hard delete

4. **Sincronizacion Assessment Postgres-MongoDB**
   - `assessment.questions_count` y `assessment.total_questions` deberian sincronizarse con MongoDB
   - No hay mecanismo automatico de sincronizacion
   - Posible inconsistencia de datos

5. **Material Versions sin Workflow**
   - `material_versions` existe y se consulta
   - No hay logica clara de cuando se crea una version
   - El campo `changed_by` podria quedar sin poblar

### 4.4 Problemas de Integridad

1. **FKs Potencialmente Huerfanas:**
   - `memberships.academic_unit_id` -> Si se elimina unit, quedan memberships huerfanas
   - `materials.academic_unit_id` -> Si se elimina unit, materiales quedan huerfanos
   - `assessment.material_id` -> Si se elimina material (hard delete), assessment queda huerfano

2. **Cascadas Faltantes:**
   - `assessment_attempt` no tiene ON DELETE CASCADE a `assessment`
   - Al eliminar material, no se elimina assessment automaticamente

3. **Datos Huerfanos en MongoDB:**
   - Al eliminar material en Postgres, documentos en MongoDB quedan huerfanos
   - No hay proceso de limpieza cross-database

### 4.5 Optimizaciones Necesarias

#### Indices Faltantes:

| Tabla | Indice Sugerido | Query que beneficia |
|-------|-----------------|---------------------|
| `materials` | `(uploaded_by_teacher_id, created_at DESC)` | FindByAuthor con ordenamiento |
| `materials` | `(school_id, status)` | List por escuela filtrando status |
| `assessment_attempt` | `(student_id, completed_at DESC)` | Historial de estudiante |
| `memberships` | `(school_id, role, is_active)` | Buscar miembros por rol |
| `login_attempts` | `(identifier, attempted_at, successful)` | Rate limiting queries |

#### Queries Potencialmente Lentas:

1. `GetHierarchyPath` en `AcademicUnitRepository` - Usa recursion sin CTE
2. `FindByStudent` en `AttemptRepository` - N+1 query para cargar answers
3. `CountFailedAttempts` - Conversion de interval con string concatenation

### 4.6 Sincronizacion Postgres-MongoDB

| Problema | Descripcion | Impacto |
|----------|-------------|---------|
| Colecciones Duplicadas | `material_assessment` (mobile) vs `material_assessment_worker` (worker) | Datos potencialmente desincronizados |
| Colecciones Duplicadas | `material_summary` (worker) vs `material_summaries` (mobile) | Datos potencialmente desincronizados |
| Schema Diferente | Las dos colecciones de assessment tienen schemas distintos | Dificil mantener consistencia |
| Sin Transacciones Cross-DB | Crear assessment en Postgres + MongoDB no es atomico | Posible inconsistencia |
| ObjectID vs UUID | MongoDB usa ObjectID, Postgres usa UUID, referencia por string | Posible error de mapeo |

---

## 5. Recomendaciones

### 5.1 Cambios Urgentes (CRITICO)

1. **Unificar colecciones MongoDB duplicadas**
   - Eliminar `material_assessment` y usar solo `material_assessment_worker`
   - Eliminar `material_summaries` y usar solo `material_summary`
   - O establecer un patron claro de sincronizacion

2. **Implementar limpieza cross-database**
   - Al eliminar material en Postgres, eliminar documentos en MongoDB
   - Crear job periodico para limpiar huerfanos

3. **Arreglar inconsistencia soft-delete en materials**
   - Decidir entre `is_deleted` o `deleted_at`
   - Unificar con el resto de tablas

4. **Agregar CASCADE o logica de limpieza**
   - `assessment_attempt_answer` ya tiene ON DELETE CASCADE (OK)
   - Agregar trigger o logica para assessment -> material

### 5.2 Mejoras Recomendadas (ALTO/MEDIO)

1. **Implementar verificacion de email**
   - Agregar endpoint de verificacion
   - Enviar email al registrarse
   - Bloquear ciertas acciones sin verificar

2. **Implementar limites de suscripcion**
   - Validar `max_teachers`/`max_students` al crear usuarios
   - Retornar error cuando se excede

3. **Optimizar GetHierarchyPath**
   - Usar CTE recursivo en lugar de bucle Go
   ```sql
   WITH RECURSIVE hierarchy AS (
     SELECT * FROM academic_units WHERE id = $1
     UNION ALL
     SELECT au.* FROM academic_units au
     JOIN hierarchy h ON au.id = h.parent_unit_id
   )
   SELECT * FROM hierarchy;
   ```

4. **Eliminar tablas sin uso**
   - Evaluar si se usaran `user_active_context`, `user_favorites`, etc.
   - Si no, crear migracion down para eliminarlas
   - Eliminar colecciones MongoDB sin uso

5. **Agregar indices sugeridos**
   - Priorizar los listados en seccion 4.5

### 5.3 Optimizaciones Sugeridas (BAJO)

1. **Consolidar schemas MongoDB**
   - Documentar schema canonico
   - Usar validadores de MongoDB

2. **Agregar metricas de uso de BD**
   - Query stats
   - Connection pool monitoring

3. **Implementar material_versions workflow**
   - Documentar cuando se crea version
   - Automatizar creacion en updates

4. **Documentar TTL de colecciones**
   - `audit_logs`: 90 dias
   - `notifications`: 30 dias
   - `analytics_events`: 365 dias
   - Verificar que los indices TTL estan activos

---

## 6. Resumen Ejecutivo

### Estadisticas Generales

| Metrica | PostgreSQL | MongoDB |
|---------|------------|---------|
| Tablas/Colecciones Definidas | 20 | 10 |
| Tablas/Colecciones en Uso | 15 | 5 |
| Tablas/Colecciones Sin Uso | 5 (25%) | 5 (50%) |
| Indices Definidos | ~25 | ~15 |
| Indices Faltantes Sugeridos | 5 | 0 |

### Issues Encontrados por Categoria

| Categoria | Cantidad | Severidad |
|-----------|----------|-----------|
| Tablas/Colecciones sin uso | 10 | Media |
| Campos sin uso | 4 | Baja |
| Flujos incompletos | 5 | Alta |
| Problemas de integridad | 4 | Alta |
| Indices faltantes | 5 | Media |
| Duplicacion de datos | 2 | Alta |

### Riesgo General

```
+--------+--------+--------+
| BAJO   | MEDIO  | ALTO   |
+--------+--------+--------+
|        |   X    |        |  <-- Estado actual
+--------+--------+--------+
```

El ecosistema tiene una arquitectura solida pero presenta riesgos medios-altos relacionados con:
- Sincronizacion PostgreSQL-MongoDB
- Datos huerfanos potenciales
- Tablas/colecciones sin uso que agregan complejidad
- Flujos de negocio incompletos (verificacion email, limites)

### Proximos Pasos Recomendados

1. **Inmediato (1-2 semanas):**
   - Unificar colecciones MongoDB duplicadas
   - Arreglar soft-delete en materials
   - Documentar decision sobre tablas sin uso

2. **Corto plazo (1 mes):**
   - Agregar indices faltantes
   - Implementar limpieza cross-database
   - Optimizar queries lentas

3. **Mediano plazo (2-3 meses):**
   - Implementar verificacion de email
   - Implementar limites de suscripcion
   - Eliminar tablas sin uso

---

*Documento generado automaticamente por analisis de codigo fuente.*
*Para actualizaciones, re-ejecutar el analisis contra los repositorios actuales.*
