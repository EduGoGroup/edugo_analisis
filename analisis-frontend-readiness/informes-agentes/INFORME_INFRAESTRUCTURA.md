# INFORME EXHAUSTIVO: ANÁLISIS DE EDUGO-INFRASTRUCTURE

**Fecha de Análisis:** 2025-12-24
**Proyecto Analizado:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure`
**Rama Principal:** `main`
**Último Commit:** `00f951f - Merge pull request #49 from EduGoGroup/feature/add-school-id-to-users`

---

## RESUMEN EJECUTIVO

### ESTADO DEL PROYECTO
- ✅ **ESTADO:** Repositorio centralizado de migraciones funcional y activo
- ✅ **ORGANIZACIÓN:** Estructura modular clara (PostgreSQL + MongoDB)
- ✅ **ÚLTIMA MIGRACIÓN:** 017_add_school_id_to_users (23 Dec 2024)
- ✅ **ARQUITECTURA:** Sistema de 4 capas documentado (structure, constraints, testing, seeds)

### HALLAZGOS CRÍTICOS

#### 🟢 NO SE ENCONTRARON BANDERAS CRÍTICAS
- ✅ NO hay migraciones SQL en `edugo-api-administracion` (solo referencias git internas)
- ✅ NO hay migraciones SQL en `edugo-api-mobile` (solo referencias git internas)
- ✅ NO hay migraciones SQL en `edugo-worker` (solo referencias git internas)
- ✅ Toda la infraestructura está correctamente centralizada en `edugo-infrastructure`

---

## RAMAS GIT DISPONIBLES

### Ramas Locales Principales
```
* main                                     (rama principal)
  dev                                      (desarrollo)
  clean                                    (limpieza)
  feature/add-school-id-to-users          (completada y mergeada)
  feature/fase1-ui-database-infrastructure
  feature/mongodb-4-layer-architecture
  feature/postgres-4-layer-architecture
  feature/simplify-modules
  fix/add-missing-update-updated-at-function
  fix/implement-real-demo-user-passwords
```

### Commits Recientes en Main
```
00f951f - Merge pull request #49 from EduGoGroup/feature/add-school-id-to-users
4da46a5 - feat: agregar school_id a tabla users
1fe3b39 - Merge pull request #48 from EduGoGroup/clean
```

### Comparación Main vs Dev
- **Estado:** Main y dev están sincronizados (sin diferencias)

---

## INVENTARIO COMPLETO: POSTGRESQL

### FUNCIONES BASE
**Archivo:** `postgres/migrations/structure/000_create_functions.sql`

| Función | Propósito | Uso |
|---------|-----------|-----|
| `update_updated_at_column()` | Actualiza campo `updated_at` automáticamente | Trigger BEFORE UPDATE en todas las tablas con `updated_at` |

---

### TABLA 1: users
**Archivo:** `postgres/migrations/structure/001_create_users.sql`
**Migración:** `001_create_users.up.sql`
**Owner:** infrastructure
**Usada por:** api-admin, api-mobile, worker

#### Campos
| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Identificador único |
| email | VARCHAR(255) | NOT NULL, UNIQUE | Email del usuario |
| password_hash | VARCHAR(255) | NOT NULL | Hash de contraseña |
| first_name | VARCHAR(100) | NOT NULL | Nombre |
| last_name | VARCHAR(100) | NOT NULL | Apellido |
| role | VARCHAR(50) | NOT NULL, CHECK IN ('admin', 'teacher', 'student', 'guardian') | Rol del usuario |
| is_active | BOOLEAN | NOT NULL, DEFAULT true | Estado activo |
| email_verified | BOOLEAN | NOT NULL, DEFAULT false | Email verificado |
| school_id | UUID | FOREIGN KEY schools(id), ON DELETE SET NULL | Escuela principal (agregado en migración 017) |
| created_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha creación |
| updated_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha actualización |
| deleted_at | TIMESTAMP WITH TIME ZONE | NULL | Soft delete |

#### Índices
- `idx_users_email` ON (email)
- `idx_users_role` ON (role)
- `idx_users_active` ON (is_active)
- `idx_users_created_at` ON (created_at DESC)
- `idx_users_school_id` ON (school_id)

#### Constraints (archivo: `postgres/migrations/constraints/001_create_users.sql`)
- `users_email_unique` UNIQUE (email)
- `users_role_check` CHECK (role IN ('admin', 'teacher', 'student', 'guardian'))

---

### TABLA 2: schools
**Archivo:** `postgres/migrations/structure/002_create_schools.sql`
**Migración:** `002_create_schools.up.sql`
**Owner:** infrastructure
**Usada por:** api-admin, api-mobile, worker

#### Campos
| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Identificador único |
| name | VARCHAR(255) | NOT NULL | Nombre de la escuela |
| code | VARCHAR(50) | NOT NULL | Código único |
| address | TEXT | NULL | Dirección |
| city | VARCHAR(100) | NULL | Ciudad |
| country | VARCHAR(100) | NOT NULL, DEFAULT 'Chile' | País |
| phone | VARCHAR(50) | NULL | Teléfono |
| email | VARCHAR(255) | NULL | Email institucional |
| metadata | JSONB | DEFAULT '{}'::jsonb | Metadata extensible (logo, config) |
| is_active | BOOLEAN | NOT NULL, DEFAULT true | Estado activo |
| subscription_tier | VARCHAR(50) | NOT NULL, DEFAULT 'free' | free, basic, premium, enterprise |
| max_teachers | INTEGER | NOT NULL, DEFAULT 10 | Límite de profesores |
| max_students | INTEGER | NOT NULL, DEFAULT 100 | Límite de estudiantes |
| created_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha creación |
| updated_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha actualización |
| deleted_at | TIMESTAMP WITH TIME ZONE | NULL | Soft delete |

#### Índices
- índices estándar (no detallados en archivo)

#### Constraints (archivo: `postgres/migrations/constraints/002_create_schools.sql`)
- constraints estándar

---

### TABLA 3: academic_units
**Archivo:** `postgres/migrations/structure/003_create_academic_units.sql`
**Migración:** `003_create_academic_units.up.sql`
**Owner:** infrastructure
**Usada por:** api-admin (jerarquía), api-mobile (plano), worker

#### Campos
| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Identificador único |
| parent_unit_id | UUID | FOREIGN KEY academic_units(id), ON DELETE SET NULL | Unidad padre (jerarquía opcional) |
| school_id | UUID | NOT NULL, FOREIGN KEY schools(id), ON DELETE CASCADE | Escuela propietaria |
| name | VARCHAR(255) | NOT NULL | Nombre de la unidad |
| code | VARCHAR(50) | NOT NULL | Código único |
| type | VARCHAR(50) | NOT NULL, CHECK IN (...) | school, grade, class, section, club, department |
| description | TEXT | NULL | Descripción |
| level | VARCHAR(50) | NULL | Nivel académico |
| academic_year | INTEGER | DEFAULT 0 | Año académico (0 = sin año específico) |
| metadata | JSONB | DEFAULT '{}'::jsonb | Metadata extensible |
| is_active | BOOLEAN | NOT NULL, DEFAULT true | Estado activo |
| created_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha creación |
| updated_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha actualización |
| deleted_at | TIMESTAMP WITH TIME ZONE | NULL | Soft delete |

#### Índices
- índices estándar de FK y búsquedas

#### Constraints (archivo: `postgres/migrations/constraints/003_create_academic_units.sql`)
- `academic_units_parent_fkey` FOREIGN KEY (parent_unit_id) → academic_units(id) ON DELETE SET NULL
- `academic_units_school_fkey` FOREIGN KEY (school_id) → schools(id) ON DELETE CASCADE
- `academic_units_unique_code` UNIQUE(school_id, code, academic_year)
- `academic_units_no_self_reference` CHECK (id != parent_unit_id)
- `academic_units_type_check` CHECK (type IN ('school', 'grade', 'class', 'section', 'club', 'department'))

#### Funciones Especiales
- `prevent_academic_unit_cycles()` - Previene ciclos en jerarquía con detección de hasta 50 niveles
- Trigger: `trigger_prevent_academic_unit_cycles` BEFORE INSERT OR UPDATE

#### Vistas
- `v_academic_unit_tree` - Vista CTE recursiva para navegación jerárquica completa

---

### TABLA 4: memberships
**Archivo:** `postgres/migrations/structure/004_create_memberships.sql`
**Migración:** `004_create_memberships.up.sql`
**Owner:** infrastructure
**Usada por:** api-admin, api-mobile, worker

#### Campos
| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Identificador único |
| user_id | UUID | NOT NULL, FOREIGN KEY users(id) | Usuario |
| school_id | UUID | NOT NULL, FOREIGN KEY schools(id) | Escuela |
| academic_unit_id | UUID | NULL, FOREIGN KEY academic_units(id) | Unidad académica |
| role | VARCHAR(50) | NOT NULL | teacher, student, guardian, coordinator, admin, assistant |
| metadata | JSONB | DEFAULT '{}'::jsonb | Permisos específicos, configuración |
| is_active | BOOLEAN | NOT NULL, DEFAULT true | Estado activo |
| enrolled_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha de inicio |
| withdrawn_at | TIMESTAMP WITH TIME ZONE | NULL | Fecha de fin (NULL = activo) |
| created_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha creación |
| updated_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha actualización |

#### Constraints (archivo: `postgres/migrations/constraints/004_create_memberships.sql`)
- Foreign keys a users, schools, academic_units

---

### TABLA 5: materials
**Archivo:** `postgres/migrations/structure/005_create_materials.sql`
**Migración:** `005_create_materials.up.sql`
**Owner:** infrastructure
**Usada por:** api-mobile, worker

#### Campos
| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Identificador único |
| school_id | UUID | NOT NULL, FOREIGN KEY schools(id), ON DELETE CASCADE | Escuela propietaria |
| uploaded_by_teacher_id | UUID | NOT NULL, FOREIGN KEY users(id), ON DELETE RESTRICT | Profesor que subió |
| academic_unit_id | UUID | NULL, FOREIGN KEY academic_units(id), ON DELETE SET NULL | Unidad académica |
| title | VARCHAR(255) | NOT NULL | Título del material |
| description | TEXT | NULL | Descripción |
| subject | VARCHAR(100) | NULL | Materia |
| grade | VARCHAR(50) | NULL | Grado |
| file_url | TEXT | NOT NULL | URL del archivo |
| file_type | VARCHAR(100) | NOT NULL | Tipo de archivo |
| file_size_bytes | BIGINT | NOT NULL | Tamaño en bytes |
| status | VARCHAR(50) | NOT NULL, DEFAULT 'uploaded', CHECK IN (...) | uploaded, processing, ready, failed |
| processing_started_at | TIMESTAMP WITH TIME ZONE | NULL | Inicio procesamiento |
| processing_completed_at | TIMESTAMP WITH TIME ZONE | NULL | Fin procesamiento |
| is_public | BOOLEAN | NOT NULL, DEFAULT false | Visibilidad pública |
| created_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha creación |
| updated_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha actualización |
| deleted_at | TIMESTAMP WITH TIME ZONE | NULL | Soft delete |

#### Constraints (archivo: `postgres/migrations/constraints/005_create_materials.sql`)
- `materials_school_fkey` → schools(id) ON DELETE CASCADE
- `materials_teacher_fkey` → users(id) ON DELETE RESTRICT
- `materials_unit_fkey` → academic_units(id) ON DELETE SET NULL
- `materials_status_check` CHECK (status IN ('uploaded', 'processing', 'ready', 'failed'))

---

### TABLA 6: assessment
**Archivo:** `postgres/migrations/structure/006_create_assessment.sql`
**Migración:** `006_create_assessments.up.sql`
**Owner:** infrastructure
**Usada por:** api-mobile, worker

#### Campos
| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Identificador único |
| material_id | UUID | NOT NULL, FOREIGN KEY materials(id) | Material asociado |
| mongo_document_id | VARCHAR(24) | NOT NULL | ObjectId del documento en MongoDB material_assessment |
| questions_count | INTEGER | NOT NULL, DEFAULT 0 | Cantidad de preguntas |
| status | VARCHAR(50) | NOT NULL, DEFAULT 'generated' | Estado del assessment |
| title | VARCHAR(255) | NULL | Título (opcional si viene de MongoDB) |
| pass_threshold | INTEGER | DEFAULT 70 | Porcentaje mínimo para aprobar (0-100) |
| max_attempts | INTEGER | NULL | Máximo de intentos (NULL = ilimitado) |
| time_limit_minutes | INTEGER | NULL | Límite de tiempo (NULL = sin límite) |
| total_questions | INTEGER | NULL | Total de preguntas (sync con questions_count) |
| created_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha creación |
| updated_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha actualización |
| deleted_at | TIMESTAMP WITH TIME ZONE | NULL | Soft delete |

#### Constraints (archivo: `postgres/migrations/constraints/006_create_assessment.sql`)
- Foreign keys y validaciones estándar

---

### TABLA 7: assessment_attempt
**Archivo:** `postgres/migrations/structure/007_create_assessment_attempt.sql`
**Migración:** `007_create_assessment_attempts.up.sql`
**Owner:** infrastructure
**Usada por:** api-mobile

#### Campos
| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Identificador único |
| assessment_id | UUID | NOT NULL, FOREIGN KEY assessment(id) | Assessment asociado |
| student_id | UUID | NOT NULL, FOREIGN KEY users(id) | Estudiante |
| started_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Inicio del intento |
| completed_at | TIMESTAMP WITH TIME ZONE | NULL | Fin del intento |
| score | DECIMAL(5,2) | NULL | Puntuación obtenida |
| max_score | DECIMAL(5,2) | NULL | Puntuación máxima |
| percentage | DECIMAL(5,2) | NULL | Porcentaje (0-100) |
| status | VARCHAR(50) | NOT NULL, DEFAULT 'in_progress' | Estado del intento |
| time_spent_seconds | INTEGER | NULL | Tiempo total en segundos (max 2 horas) |
| idempotency_key | VARCHAR(64) | NULL | Clave para prevenir duplicados |
| created_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha creación |
| updated_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha actualización |

#### Constraints (archivo: `postgres/migrations/constraints/007_create_assessment_attempt.sql`)
- Foreign keys y validaciones estándar

---

### TABLA 8: assessment_attempt_answer
**Archivo:** `postgres/migrations/structure/008_create_assessment_attempt_answer.sql`
**Migración:** `008_create_assessment_answers.up.sql`
**Owner:** infrastructure
**Usada por:** api-mobile

#### Campos
| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Identificador único |
| attempt_id | UUID | NOT NULL, FOREIGN KEY assessment_attempt(id) | Intento asociado |
| question_index | INTEGER | NOT NULL | Índice de la pregunta (0-based) |
| student_answer | TEXT | NULL | Respuesta del estudiante (JSON, string, etc) |
| is_correct | BOOLEAN | NULL | Si es correcta |
| points_earned | DECIMAL(5,2) | NULL | Puntos obtenidos |
| max_points | DECIMAL(5,2) | NULL | Puntos máximos |
| time_spent_seconds | INTEGER | NULL | Tiempo en segundos para esta pregunta |
| answered_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha de respuesta |
| created_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha creación |
| updated_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha actualización |

#### Constraints (archivo: `postgres/migrations/constraints/008_create_assessment_attempt_answer.sql`)
- Foreign keys y validaciones estándar

---

### TABLA 9: refresh_tokens
**Archivo:** `postgres/migrations/structure/009_create_refresh_tokens.sql`
**Owner:** infrastructure
**Usada por:** api-admin, api-mobile

#### Campos
| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| id | UUID | PRIMARY KEY | Identificador único |
| token_hash | VARCHAR(255) | NOT NULL, UNIQUE | Hash del refresh token |
| user_id | UUID | NOT NULL, FOREIGN KEY users(id), ON DELETE CASCADE | Usuario propietario |
| client_info | JSONB | NULL | Info del cliente (navegador, IP, etc.) |
| expires_at | TIMESTAMP WITH TIME ZONE | NOT NULL | Fecha de expiración |
| created_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha creación |
| revoked_at | TIMESTAMP WITH TIME ZONE | NULL | Timestamp de revocación |
| replaced_by | UUID | NULL, FOREIGN KEY refresh_tokens(id), ON DELETE SET NULL | Token de reemplazo (rotation) |

#### Índices
- `idx_refresh_tokens_user_id` ON (user_id)
- `idx_refresh_tokens_token_hash` ON (token_hash)
- `idx_refresh_tokens_expires_at` ON (expires_at)
- `idx_refresh_tokens_revoked_at` ON (revoked_at) WHERE revoked_at IS NOT NULL

---

### TABLA 10: login_attempts
**Archivo:** `postgres/migrations/structure/010_create_login_attempts.sql`
**Owner:** infrastructure
**Usada por:** api-admin, api-mobile

#### Campos
| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| id | SERIAL | PRIMARY KEY | Identificador único |
| identifier | VARCHAR(255) | NOT NULL | Email o IP según attempt_type |
| attempt_type | VARCHAR(50) | NOT NULL, CHECK IN ('email', 'ip') | Tipo de intento |
| successful | BOOLEAN | NOT NULL, DEFAULT false | Si fue exitoso |
| user_agent | TEXT | NULL | User agent del cliente |
| ip_address | VARCHAR(45) | NULL | IP del cliente |
| attempted_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha del intento |

#### Índices
- `idx_login_attempts_identifier` ON (identifier)
- `idx_login_attempts_attempted_at` ON (attempted_at)
- `idx_login_attempts_identifier_attempted_at` ON (identifier, attempted_at)
- `idx_login_attempts_successful` ON (successful)
- `idx_login_attempts_rate_limit` ON (identifier, successful, attempted_at) WHERE successful = false

---

### TABLA 11: user_active_context
**Archivo:** `postgres/migrations/structure/011_create_user_active_context.sql`
**Owner:** infrastructure
**Usada por:** api-admin, api-mobile (FASE 1 UI Roadmap)

#### Campos
| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Identificador único |
| user_id | UUID | NOT NULL, UNIQUE, FOREIGN KEY users(id), ON DELETE CASCADE | Usuario (solo un contexto por usuario) |
| school_id | UUID | NOT NULL, FOREIGN KEY schools(id), ON DELETE CASCADE | Escuela actualmente seleccionada |
| unit_id | UUID | NULL, FOREIGN KEY academic_units(id), ON DELETE SET NULL | Unidad académica activa (opcional) |
| created_at | TIMESTAMP WITH TIME ZONE | DEFAULT NOW() | Fecha creación |
| updated_at | TIMESTAMP WITH TIME ZONE | DEFAULT NOW() | Fecha actualización |

#### Índices
- `idx_user_active_context_user` ON (user_id)
- `idx_user_active_context_school` ON (school_id)

#### Trigger
- `set_updated_at_user_active_context` BEFORE UPDATE → `update_updated_at_column()`

---

### TABLA 12: user_favorites
**Archivo:** `postgres/migrations/structure/012_create_user_favorites.sql`
**Owner:** infrastructure
**Usada por:** api-admin, api-mobile (FASE 1 UI Roadmap)

#### Campos
| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Identificador único |
| user_id | UUID | NOT NULL, FOREIGN KEY users(id), ON DELETE CASCADE | Usuario |
| material_id | UUID | NOT NULL, FOREIGN KEY materials(id), ON DELETE CASCADE | Material favorito |
| created_at | TIMESTAMP WITH TIME ZONE | DEFAULT NOW() | Fecha agregado a favoritos |

#### Constraints
- `uq_user_favorites_user_material` UNIQUE(user_id, material_id)

#### Índices
- `idx_user_favorites_user` ON (user_id)
- `idx_user_favorites_material` ON (material_id)
- `idx_user_favorites_created` ON (created_at DESC)

---

### TABLA 13: user_activity_log
**Archivo:** `postgres/migrations/structure/013_create_user_activity_log.sql`
**Owner:** infrastructure
**Usada por:** api-admin, api-mobile (FASE 1 UI Roadmap)

#### Tipo ENUM
```sql
CREATE TYPE activity_type AS ENUM (
    'material_started', 'material_progress', 'material_completed',
    'summary_viewed',
    'quiz_started', 'quiz_completed', 'quiz_passed', 'quiz_failed'
);
```

#### Campos
| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Identificador único |
| user_id | UUID | NOT NULL, FOREIGN KEY users(id), ON DELETE CASCADE | Usuario |
| activity_type | activity_type | NOT NULL | Tipo de actividad |
| material_id | UUID | NULL, FOREIGN KEY materials(id), ON DELETE SET NULL | Material asociado |
| school_id | UUID | NULL, FOREIGN KEY schools(id), ON DELETE SET NULL | Escuela asociada |
| metadata | JSONB | DEFAULT '{}' | Datos adicionales (score, pages, time_spent) |
| created_at | TIMESTAMP WITH TIME ZONE | DEFAULT NOW() | Timestamp de la actividad |

#### Índices
- `idx_user_activity_user_created` ON (user_id, created_at DESC)
- `idx_user_activity_school` ON (school_id, created_at DESC)
- `idx_user_activity_type` ON (activity_type)
- `idx_user_activity_rate_limit` ON (user_id, activity_type, created_at)

---

### TABLA 14: feature_flags
**Archivo:** `postgres/migrations/structure/014_create_feature_flags.sql`
**Owner:** infrastructure
**Usada por:** api-admin, api-mobile (Sistema de Feature Flags - Apple App)

#### Campos
| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Identificador único |
| key | VARCHAR(100) | NOT NULL, UNIQUE | Identificador del flag (ej: biometric_login) |
| name | VARCHAR(255) | NOT NULL | Nombre descriptivo |
| description | TEXT | NULL | Descripción del feature |
| enabled | BOOLEAN | NOT NULL, DEFAULT false | Estado global |
| enabled_globally | BOOLEAN | NOT NULL, DEFAULT false | Si true, ignora segmentación |
| category | VARCHAR(50) | NULL | Categoría del feature |
| priority | INTEGER | NOT NULL, DEFAULT 0 | Prioridad |
| minimum_app_version | VARCHAR(20) | NULL | Versión mínima de app |
| minimum_build_number | INTEGER | NULL | Build number mínimo |
| maximum_app_version | VARCHAR(20) | NULL | Versión máxima de app |
| maximum_build_number | INTEGER | NULL | Build number máximo |
| enabled_for_roles | JSONB | DEFAULT '[]' | Array de roles habilitados |
| enabled_for_user_ids | JSONB | DEFAULT '[]' | Array de UUIDs habilitados (whitelist) |
| disabled_for_user_ids | JSONB | DEFAULT '[]' | Array de UUIDs deshabilitados (blacklist) |
| rollout_percentage | INTEGER | DEFAULT 100, CHECK (0-100) | Porcentaje para A/B testing |
| is_experimental | BOOLEAN | NOT NULL, DEFAULT false | Flag experimental |
| requires_restart | BOOLEAN | NOT NULL, DEFAULT false | Requiere reiniciar app |
| is_debug_only | BOOLEAN | NOT NULL, DEFAULT false | Solo en builds debug |
| affects_security | BOOLEAN | NOT NULL, DEFAULT false | Requiere aprobación especial |
| created_at | TIMESTAMP WITH TIME ZONE | DEFAULT NOW() | Fecha creación |
| updated_at | TIMESTAMP WITH TIME ZONE | DEFAULT NOW() | Fecha actualización |
| created_by | UUID | NULL, FOREIGN KEY users(id), ON DELETE SET NULL | Creador |
| updated_by | UUID | NULL, FOREIGN KEY users(id), ON DELETE SET NULL | Último modificador |

#### Constraints
- `chk_valid_rollout` CHECK (rollout_percentage >= 0 AND rollout_percentage <= 100)
- `chk_valid_build_numbers` CHECK (minimum_build_number IS NULL OR maximum_build_number IS NULL OR minimum_build_number <= maximum_build_number)

#### Índices
- `idx_feature_flags_key` ON (key)
- `idx_feature_flags_enabled` ON (enabled)
- `idx_feature_flags_category` ON (category)
- `idx_feature_flags_updated_at` ON (updated_at)

#### Trigger
- `set_updated_at_feature_flags` BEFORE UPDATE → `update_updated_at_column()`

---

### TABLA 15: feature_flag_overrides
**Archivo:** `postgres/migrations/structure/015_create_feature_flag_overrides.sql`
**Owner:** infrastructure
**Usada por:** api-admin, api-mobile (Phase 2 Feature Flags)

#### Campos
| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Identificador único |
| feature_flag_id | UUID | NOT NULL, FOREIGN KEY feature_flags(id), ON DELETE CASCADE | Feature flag |
| user_id | UUID | NOT NULL, FOREIGN KEY users(id), ON DELETE CASCADE | Usuario |
| enabled | BOOLEAN | NOT NULL | Estado override |
| reason | TEXT | NULL | Razón del override (debugging, testing) |
| expires_at | TIMESTAMP WITH TIME ZONE | NULL | Expiración (NULL = permanente) |
| created_at | TIMESTAMP WITH TIME ZONE | DEFAULT NOW() | Fecha creación |
| created_by | UUID | NULL, FOREIGN KEY users(id), ON DELETE SET NULL | Admin que creó |

#### Constraints
- `uq_ff_overrides_flag_user` UNIQUE(feature_flag_id, user_id)

#### Índices
- `idx_ff_overrides_user` ON (user_id)
- `idx_ff_overrides_flag` ON (feature_flag_id)
- `idx_ff_overrides_expires` ON (expires_at) WHERE expires_at IS NOT NULL

---

### TABLA 16: material_versions
**Archivo:** `postgres/migrations/012_create_material_versions.up.sql`
**Owner:** infrastructure
**Usada por:** api-mobile, worker

#### Campos
| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Identificador único |
| material_id | UUID | NOT NULL, FOREIGN KEY materials(id), ON DELETE CASCADE | Material |
| version_number | INTEGER | NOT NULL, CHECK (version_number > 0) | Número de versión (1, 2, 3...) |
| title | VARCHAR(255) | NOT NULL | Título de esta versión |
| content_url | TEXT | NOT NULL | URL del contenido |
| changed_by | UUID | NOT NULL, FOREIGN KEY users(id) | Usuario que creó la versión |
| created_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha creación |

#### Constraints
- UNIQUE(material_id, version_number)

#### Índices
- `idx_material_versions_material_id` ON (material_id)
- `idx_material_versions_version_number` ON (material_id, version_number DESC)
- `idx_material_versions_changed_by` ON (changed_by)
- `idx_material_versions_created_at` ON (created_at DESC)

---

### TABLA 17: subjects
**Archivo:** `postgres/migrations/013_create_subjects.up.sql`
**Owner:** infrastructure
**Usada por:** api-admin, api-mobile

#### Campos
| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Identificador único |
| name | VARCHAR(255) | NOT NULL | Nombre de la materia |
| description | TEXT | NULL | Descripción detallada |
| metadata | JSONB | NULL | Metadata adicional |
| is_active | BOOLEAN | NOT NULL, DEFAULT true | Estado activo |
| created_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha creación |
| updated_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha actualización |

#### Índices
- `idx_subjects_name` ON (name)
- `idx_subjects_is_active` ON (is_active)
- `idx_subjects_created_at` ON (created_at DESC)
- `idx_subjects_metadata` ON (metadata) USING GIN

---

### TABLA 18: units
**Archivo:** `postgres/migrations/014_create_units.up.sql`
**Owner:** infrastructure
**Usada por:** api-admin

#### Campos
| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Identificador único |
| school_id | UUID | NOT NULL, FOREIGN KEY schools(id), ON DELETE CASCADE | Escuela |
| parent_unit_id | UUID | NULL, FOREIGN KEY units(id), ON DELETE SET NULL | Unidad padre |
| name | VARCHAR(255) | NOT NULL, CHECK (length(name) >= 2) | Nombre (mín 2 caracteres) |
| description | TEXT | NULL | Descripción |
| is_active | BOOLEAN | NOT NULL, DEFAULT true | Estado activo |
| created_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha creación |
| updated_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha actualización |

#### Constraints
- CHECK (id != parent_unit_id) - No puede ser su propio padre

#### Índices
- `idx_units_school_id` ON (school_id)
- `idx_units_parent_unit_id` ON (parent_unit_id)
- `idx_units_name` ON (name)
- `idx_units_is_active` ON (is_active)
- `idx_units_created_at` ON (created_at DESC)
- `idx_units_hierarchy` ON (school_id, parent_unit_id, is_active)

---

### TABLA 19: guardian_relations
**Archivo:** `postgres/migrations/015_create_guardian_relations.up.sql`
**Owner:** infrastructure
**Usada por:** api-admin, api-mobile

#### Campos
| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Identificador único |
| guardian_id | UUID | NOT NULL, FOREIGN KEY users(id), ON DELETE CASCADE | Apoderado |
| student_id | UUID | NOT NULL, FOREIGN KEY users(id), ON DELETE CASCADE | Estudiante |
| relationship_type | VARCHAR(50) | NOT NULL, CHECK IN (...) | father, mother, grandfather, grandmother, uncle, aunt, sibling, legal_guardian, other |
| is_active | BOOLEAN | NOT NULL, DEFAULT true | Estado activo |
| created_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha creación |
| updated_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha actualización |
| created_by | VARCHAR(255) | NOT NULL | Usuario que creó la relación |

#### Constraints
- UNIQUE(guardian_id, student_id)
- CHECK (guardian_id != student_id) - El apoderado no puede ser el mismo estudiante

#### Índices
- `idx_guardian_relations_guardian_id` ON (guardian_id)
- `idx_guardian_relations_student_id` ON (student_id)
- `idx_guardian_relations_relationship_type` ON (relationship_type)
- `idx_guardian_relations_is_active` ON (is_active)
- `idx_guardian_relations_created_at` ON (created_at DESC)
- `idx_guardian_relations_active_guardian` ON (guardian_id, is_active)
- `idx_guardian_relations_active_student` ON (student_id, is_active)

---

### TABLA 20: progress
**Archivo:** `postgres/migrations/016_create_progress.up.sql`
**Owner:** infrastructure
**Usada por:** api-mobile

#### Campos
| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| material_id | UUID | PRIMARY KEY (compuesta), FOREIGN KEY materials(id), ON DELETE CASCADE | Material |
| user_id | UUID | PRIMARY KEY (compuesta), FOREIGN KEY users(id), ON DELETE CASCADE | Usuario |
| percentage | INTEGER | NOT NULL, DEFAULT 0, CHECK (0-100) | Porcentaje de progreso |
| last_page | INTEGER | NOT NULL, DEFAULT 0, CHECK (>= 0) | Última página leída |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'not_started', CHECK IN (...) | not_started, in_progress, completed |
| last_accessed_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Última vez accedido |
| created_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha creación |
| updated_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | Fecha actualización |

#### Primary Key
- PRIMARY KEY (material_id, user_id) - Un progreso por material y usuario

#### Índices
- `idx_progress_user_id` ON (user_id)
- `idx_progress_material_id` ON (material_id)
- `idx_progress_status` ON (status)
- `idx_progress_last_accessed_at` ON (last_accessed_at DESC)
- `idx_progress_percentage` ON (percentage DESC)
- `idx_progress_user_status` ON (user_id, status)
- `idx_progress_material_status` ON (material_id, status)

---

## RESUMEN TABLAS POSTGRESQL

**Total de Tablas:** 20

| # | Tabla | Propósito | Proyecto Principal |
|---|-------|-----------|-------------------|
| 1 | users | Usuarios del sistema | api-admin, api-mobile, worker |
| 2 | schools | Instituciones educativas | api-admin, api-mobile, worker |
| 3 | academic_units | Unidades académicas jerárquicas | api-admin, api-mobile, worker |
| 4 | memberships | Relaciones usuario-escuela-unidad | api-admin, api-mobile, worker |
| 5 | materials | Materiales educativos | api-mobile, worker |
| 6 | assessment | Assessments/Quizzes (referencia a MongoDB) | api-mobile, worker |
| 7 | assessment_attempt | Intentos de estudiantes | api-mobile |
| 8 | assessment_attempt_answer | Respuestas individuales | api-mobile |
| 9 | refresh_tokens | Tokens JWT para sesiones | api-admin, api-mobile |
| 10 | login_attempts | Rate limiting y auditoría | api-admin, api-mobile |
| 11 | user_active_context | Contexto/escuela activa (FASE 1 UI) | api-admin, api-mobile |
| 12 | user_favorites | Materiales favoritos (FASE 1 UI) | api-admin, api-mobile |
| 13 | user_activity_log | Log de actividades (FASE 1 UI) | api-admin, api-mobile |
| 14 | feature_flags | Sistema de feature flags (Apple App) | api-admin, api-mobile |
| 15 | feature_flag_overrides | Sobrescrituras por usuario (Phase 2) | api-admin, api-mobile |
| 16 | material_versions | Historial de versiones de materiales | api-mobile, worker |
| 17 | subjects | Materias/Asignaturas | api-admin, api-mobile |
| 18 | units | Unidades organizacionales | api-admin |
| 19 | guardian_relations | Relaciones apoderado-estudiante | api-admin, api-mobile |
| 20 | progress | Progreso de lectura por usuario | api-mobile |

---

## INVENTARIO COMPLETO: MONGODB

### COLECCIÓN 1: material_assessment
**Archivo:** `mongodb/migrations/structure/001_material_assessment.go`
**Owner:** infrastructure
**Usada por:** api-mobile, worker

#### Schema Validation
```javascript
{
  "bsonType": "object",
  "required": ["material_id", "questions", "metadata", "created_at", "updated_at"],
  "properties": {
    "material_id": { "bsonType": "string", "description": "UUID of material in PostgreSQL" },
    "questions": {
      "bsonType": "array",
      "items": {
        "bsonType": "object",
        "required": ["question_index", "question_text", "question_type", "options"]
      }
    },
    "metadata": { "bsonType": "object" },
    "created_at": { "bsonType": "date" },
    "updated_at": { "bsonType": "date" }
  }
}
```

#### Índices (archivo: `mongodb/migrations/constraints/001_material_assessment_indexes.go`)
- Índices en material_id, created_at, etc.

---

### COLECCIÓN 2: material_content
**Archivo:** `mongodb/migrations/structure/002_material_content.go`
**Owner:** infrastructure
**Usada por:** worker

#### Schema Validation
```javascript
{
  "bsonType": "object",
  "required": ["material_id", "content_type", "created_at", "updated_at"],
  "properties": {
    "material_id": { "bsonType": "string", "description": "UUID of material in PostgreSQL" },
    "content_type": {
      "bsonType": "string",
      "enum": ["pdf_extracted", "video_transcript", "document_parsed", "slides_extracted"]
    },
    "raw_text": { "bsonType": "string" },
    "structured_content": {
      "bsonType": "object",
      "properties": {
        "title": { "bsonType": "string" },
        "sections": { "bsonType": "array" },
        "summary": { "bsonType": "string" },
        "key_concepts": { "bsonType": "array" }
      }
    },
    "processing_info": { "bsonType": "object" },
    "created_at": { "bsonType": "date" },
    "updated_at": { "bsonType": "date" }
  }
}
```

#### Índices (archivo: `mongodb/migrations/constraints/002_material_content_indexes.go`)
- Índices en material_id, content_type, etc.

---

### COLECCIÓN 3: assessment_attempt_result
**Archivo:** `mongodb/migrations/structure/003_assessment_attempt_result.go`
**Owner:** infrastructure
**Usada por:** api-mobile

#### Schema Validation
```javascript
{
  "bsonType": "object",
  "required": ["attempt_id", "student_id", "assessment_id", "answers", "score", "started_at", "submitted_at", "created_at"],
  "properties": {
    "attempt_id": { "bsonType": "string", "description": "UUID of attempt in PostgreSQL" },
    "student_id": { "bsonType": "string", "description": "UUID of student in PostgreSQL" },
    "assessment_id": { "bsonType": "string", "description": "UUID of assessment in PostgreSQL" },
    "answers": {
      "bsonType": "array",
      "items": {
        "bsonType": "object",
        "required": ["question_index", "selected_option_index", "is_correct", "time_spent_seconds"],
        "properties": {
          "question_index": { "bsonType": "int" },
          "selected_option_index": { "bsonType": "int" },
          "is_correct": { "bsonType": "bool" },
          "time_spent_seconds": { "bsonType": "int" }
        }
      }
    },
    "score": {
      "bsonType": "object",
      "required": ["correct_count", "total_questions", "percentage"],
      "properties": {
        "correct_count": { "bsonType": "int" },
        "total_questions": { "bsonType": "int" },
        "percentage": { "bsonType": "double" }
      }
    },
    "started_at": { "bsonType": "date" },
    "submitted_at": { "bsonType": "date" },
    "created_at": { "bsonType": "date" }
  }
}
```

---

### COLECCIÓN 4: audit_logs
**Archivo:** `mongodb/migrations/structure/004_audit_logs.go`
**Owner:** infrastructure
**Usada por:** api-mobile, worker

#### Schema Validation (resumen)
```javascript
{
  "bsonType": "object",
  "required": ["event_type", "actor_id", "timestamp", "resource_type"],
  "properties": {
    "event_type": {
      "bsonType": "string",
      "enum": [
        "user.created", "user.updated", "user.deleted", "user.login", "user.logout",
        "school.created", "school.updated", "school.deleted",
        "material.uploaded", "material.updated", "material.deleted", "material.processed",
        "assessment.generated", "assessment.published", "assessment.archived",
        "attempt.started", "attempt.submitted", "attempt.graded",
        "membership.created", "membership.updated", "membership.deleted",
        "permission.granted", "permission.revoked",
        "system.backup", "system.restore", "system.migration"
      ]
    },
    "actor_id": { "bsonType": "string" },
    "actor_type": { "bsonType": "string", "enum": ["user", "system", "api", "worker"] },
    "timestamp": { "bsonType": "date" },
    "resource_type": { "bsonType": "string", "enum": ["user", "school", "academic_unit", "membership", "material", "assessment", "attempt", "system"] },
    "resource_id": { "bsonType": "string" },
    "action": { "bsonType": "string", "enum": ["create", "read", "update", "delete", "login", "logout", "upload", "process", "submit", "grade"] },
    "changes": { "bsonType": "object" },
    "metadata": { "bsonType": "object" },
    "ip_address": { "bsonType": "string" },
    "user_agent": { "bsonType": "string" },
    "severity": { "bsonType": "string", "enum": ["info", "warning", "error", "critical"] }
  }
}
```

---

### COLECCIÓN 5: notifications
**Archivo:** `mongodb/migrations/structure/005_notifications.go`
**Owner:** infrastructure
**Usada por:** api-mobile

#### Schema Validation (resumen)
```javascript
{
  "bsonType": "object",
  "required": ["user_id", "notification_type", "title", "is_read", "created_at"],
  "properties": {
    "user_id": { "bsonType": "string", "description": "UUID of user in PostgreSQL" },
    "notification_type": {
      "bsonType": "string",
      "enum": [
        "assessment.ready", "assessment.graded",
        "material.uploaded", "material.processed", "material.shared",
        "membership.added", "membership.removed",
        "deadline.approaching",
        "achievement.unlocked",
        "system.announcement", "system.maintenance"
      ]
    },
    "title": { "bsonType": "string" },
    "message": { "bsonType": "string" },
    "is_read": { "bsonType": "bool" },
    "priority": { "bsonType": "string", "enum": ["low", "medium", "high", "urgent"] },
    "category": { "bsonType": "string", "enum": ["academic", "administrative", "social", "system"] },
    "action_url": { "bsonType": "string" },
    "metadata": { "bsonType": "object" },
    "read_at": { "bsonType": "date" },
    "created_at": { "bsonType": "date" },
    "expires_at": { "bsonType": "date" }
  }
}
```

---

### COLECCIÓN 6: analytics_events
**Archivo:** `mongodb/migrations/structure/006_analytics_events.go`
**Owner:** infrastructure
**Usada por:** api-mobile, worker

#### Schema Validation (resumen)
```javascript
{
  "bsonType": "object",
  "required": ["event_name", "timestamp"],
  "properties": {
    "event_name": {
      "bsonType": "string",
      "enum": [
        "page.view",
        "material.view", "material.download", "material.search",
        "assessment.start", "assessment.complete", "assessment.abandon",
        "question.answer", "question.skip",
        "video.play", "video.pause", "video.complete",
        "session.start", "session.end",
        "feature.click",
        "error.occurred",
        "search.performed",
        "filter.applied"
      ]
    },
    "user_id": { "bsonType": "string" },
    "session_id": { "bsonType": "string" },
    "timestamp": { "bsonType": "date" },
    "properties": { "bsonType": "object" },
    "device": {
      "bsonType": "object",
      "properties": {
        "type": { "bsonType": "string" },
        "os": { "bsonType": "string" },
        "browser": { "bsonType": "string" }
      }
    },
    "location": {
      "bsonType": "object",
      "properties": {
        "country": { "bsonType": "string" },
        "city": { "bsonType": "string" },
        "timezone": { "bsonType": "string" }
      }
    },
    "context": {
      "bsonType": "object",
      "properties": {
        "page": { "bsonType": "string" },
        "referrer": { "bsonType": "string" },
        "url": { "bsonType": "string" }
      }
    }
  }
}
```

---

### COLECCIÓN 7: material_summary
**Archivo:** `mongodb/migrations/structure/007_material_summary.go`
**Owner:** infrastructure
**Usada por:** worker, api-mobile

#### Schema Validation
```javascript
{
  "bsonType": "object",
  "required": ["material_id", "summary", "key_points", "language", "word_count", "version", "ai_model", "processing_time_ms", "created_at", "updated_at"],
  "properties": {
    "material_id": {
      "bsonType": "string",
      "pattern": "^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-4[0-9a-fA-F]{3}-[89abAB][0-9a-fA-F]{3}-[0-9a-fA-F]{12}$",
      "description": "UUID v4 of material in PostgreSQL"
    },
    "summary": {
      "bsonType": "string",
      "minLength": 10,
      "maxLength": 5000
    },
    "key_points": {
      "bsonType": "array",
      "minItems": 1,
      "maxItems": 10,
      "items": { "bsonType": "string" }
    },
    "language": {
      "bsonType": "string",
      "enum": ["es", "en", "pt"]
    },
    "word_count": {
      "bsonType": "int",
      "minimum": 1
    },
    "version": {
      "bsonType": "int",
      "minimum": 1
    },
    "ai_model": {
      "bsonType": "string",
      "enum": ["gpt-4", "gpt-3.5-turbo", "gpt-4-turbo", "gpt-4o"]
    },
    "processing_time_ms": {
      "bsonType": "int",
      "minimum": 0
    },
    "metadata": { "bsonType": "object" },
    "created_at": { "bsonType": "date" },
    "updated_at": { "bsonType": "date" }
  }
}
```

---

### COLECCIÓN 8: material_assessment_worker
**Archivo:** `mongodb/migrations/structure/008_material_assessment_worker.go`
**Owner:** infrastructure
**Usada por:** worker

#### Schema Validation
```javascript
{
  "bsonType": "object",
  "required": ["material_id", "questions", "total_questions", "total_points", "version", "ai_model", "processing_time_ms", "created_at", "updated_at"],
  "properties": {
    "material_id": {
      "bsonType": "string",
      "pattern": "^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-4[0-9a-fA-F]{3}-[89abAB][0-9a-fA-F]{3}-[0-9a-fA-F]{12}$"
    },
    "questions": {
      "bsonType": "array",
      "minItems": 3,
      "maxItems": 20,
      "items": {
        "bsonType": "object",
        "required": ["question_id", "question_text", "question_type", "correct_answer", "points", "difficulty"],
        "properties": {
          "question_id": { "bsonType": "string" },
          "question_text": { "bsonType": "string" },
          "question_type": {
            "bsonType": "string",
            "enum": ["multiple_choice", "true_false", "open"]
          },
          "options": { "bsonType": "array" },
          "correct_answer": { "bsonType": "string" },
          "points": { "bsonType": "int" },
          "difficulty": {
            "bsonType": "string",
            "enum": ["easy", "medium", "hard"]
          },
          "explanation": { "bsonType": "string" }
        }
      }
    },
    "total_questions": {
      "bsonType": "int",
      "minimum": 3,
      "maximum": 20
    },
    "total_points": { "bsonType": "int" },
    "version": {
      "bsonType": "int",
      "minimum": 1
    },
    "ai_model": {
      "bsonType": "string",
      "enum": ["gpt-4", "gpt-3.5-turbo", "gpt-4-turbo", "gpt-4o"]
    },
    "processing_time_ms": {
      "bsonType": "int",
      "minimum": 0
    },
    "metadata": { "bsonType": "object" },
    "created_at": { "bsonType": "date" },
    "updated_at": { "bsonType": "date" }
  }
}
```

---

### COLECCIÓN 9: material_event
**Archivo:** `mongodb/migrations/structure/009_material_event.go`
**Owner:** infrastructure
**Usada por:** worker, api-mobile

#### Schema Validation
```javascript
{
  "bsonType": "object",
  "required": ["event_type", "payload", "status", "retry_count", "created_at", "updated_at"],
  "properties": {
    "event_type": {
      "bsonType": "string",
      "enum": [
        "material_uploaded",
        "material_reprocess",
        "material_deleted",
        "assessment_attempt",
        "student_enrolled",
        "student_unenrolled"
      ]
    },
    "material_id": {
      "bsonType": "string",
      "pattern": "^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-4[0-9a-fA-F]{3}-[89abAB][0-9a-fA-F]{3}-[0-9a-fA-F]{12}$"
    },
    "user_id": {
      "bsonType": "string",
      "pattern": "^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-4[0-9a-fA-F]{3}-[89abAB][0-9a-fA-F]{3}-[0-9a-fA-F]{12}$"
    },
    "payload": { "bsonType": "object" },
    "status": {
      "bsonType": "string",
      "enum": ["pending", "processing", "completed", "failed"]
    },
    "error_msg": {
      "bsonType": "string",
      "maxLength": 5000
    },
    "stack_trace": {
      "bsonType": "string",
      "maxLength": 10000
    },
    "retry_count": {
      "bsonType": "int",
      "minimum": 0
    },
    "next_retry_at": { "bsonType": "date" },
    "processed_at": { "bsonType": "date" },
    "created_at": { "bsonType": "date" },
    "updated_at": { "bsonType": "date" }
  }
}
```

---

## RESUMEN COLECCIONES MONGODB

**Total de Colecciones:** 9

| # | Colección | Propósito | Proyecto Principal | TTL |
|---|-----------|-----------|-------------------|-----|
| 1 | material_assessment | Assessments/Quizzes generados por IA | api-mobile, worker | No |
| 2 | material_content | Contenido extraído/procesado de materiales | worker | No |
| 3 | assessment_attempt_result | Resultados detallados de intentos | api-mobile | No |
| 4 | audit_logs | Trail de auditoría completo del sistema | api-mobile, worker | Posible |
| 5 | notifications | Notificaciones de usuarios | api-mobile | Posible |
| 6 | analytics_events | Eventos de analytics y tracking | api-mobile, worker | Posible |
| 7 | material_summary | Resúmenes generados por IA | worker, api-mobile | No |
| 8 | material_assessment_worker | Assessments procesados por worker | worker | No |
| 9 | material_event | Cola de eventos para procesamiento | worker, api-mobile | Posible |

---

## SEEDS Y DATOS DE TESTING

### Seeds PostgreSQL (Producción)
**Directorio:** `postgres/migrations/seeds/`

| Archivo | Contenido |
|---------|-----------|
| `001_system_config.sql` | Configuración del sistema inicial |

### Testing/Demo PostgreSQL
**Directorio:** `postgres/migrations/testing/`

| Archivo | Contenido |
|---------|-----------|
| `001_demo_users.sql` | Usuarios de demostración |
| `002_demo_schools.sql` | Escuelas de demostración |
| `003_demo_academic_units.sql` | Unidades académicas demo |
| `004_demo_memberships.sql` | Membresías demo |
| `005_demo_materials.sql` | Materiales demo |
| `006_demo_user_active_context.sql` | Contextos activos demo |
| `007_demo_user_favorites.sql` | Favoritos demo |
| `008_demo_user_activity_log.sql` | Logs de actividad demo |
| `009_demo_feature_flags.sql` | Feature flags demo |
| `010_demo_feature_flag_overrides.sql` | Overrides de flags demo |

### Seeds PostgreSQL (Directorio `/seeds/postgres/`)
**Directorio:** `seeds/postgres/`

| Archivo | Contenido |
|---------|-----------|
| `materials.sql` | Materiales seed |
| `schools.sql` | Escuelas seed |
| `users.sql` | Usuarios seed |

### Seeds MongoDB
**Archivo:** `mongodb/migrations/seeds.go`
**Función:** `ApplySeeds()` - Datos iniciales de producción

**Archivo:** `mongodb/migrations/mock_data.go`
**Función:** `ApplyMockData()` - Datos de prueba para desarrollo

---

## ARQUITECTURA DE 4 CAPAS

### PostgreSQL
```
Capa 1: structure/       → Esquema base (DDL, CREATE TABLE)
Capa 2: constraints/     → Constraints, FKs, Triggers, Vistas
Capa 3: seeds/           → Datos iniciales (producción)
Capa 4: testing/         → Datos de prueba (desarrollo)
```

### MongoDB
```
Capa 1: structure/       → Colecciones con schema validation
Capa 2: constraints/     → Índices
Capa 3: seeds.go         → ApplySeeds() - Datos iniciales
Capa 4: mock_data.go     → ApplyMockData() - Datos de prueba
```

---

## VERIFICACIÓN: BANDERAS CRÍTICAS

### RESULTADO: ✅ NINGUNA BANDERA CRÍTICA ENCONTRADA

#### Búsqueda en edugo-api-administracion
```bash
find /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion -name "*migration*" -o -name "*.sql"
```
**Resultado:**
- Solo encontrados archivos `.idea/copilot.data.migration.*` (metadata de IDE)
- ✅ NO hay archivos SQL de migraciones propias

#### Búsqueda en edugo-api-mobile
```bash
find /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile -name "*migration*" -o -name "*.sql"
```
**Resultado:**
- Solo encontrados archivos `.idea/copilot.data.migration.*` (metadata de IDE)
- ✅ NO hay archivos SQL de migraciones propias

#### Búsqueda en edugo-worker
```bash
find /Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker -name "*migration*" -o -name "*.sql"
```
**Resultado:**
- Solo encontrados archivos `.idea/copilot.data.migration.*` (metadata de IDE)
- ✅ NO hay archivos SQL de migraciones propias

---

## CONCLUSIONES Y RECOMENDACIONES

### ESTADO GENERAL
1. ✅ **CENTRALIZACIÓN CORRECTA:** Todas las migraciones están centralizadas en `edugo-infrastructure`
2. ✅ **SEPARACIÓN CLARA:** Los proyectos consumidores NO tienen migraciones propias
3. ✅ **ARQUITECTURA DOCUMENTADA:** Sistema de 4 capas bien definido
4. ✅ **VERSIONADO ACTIVO:** Última migración 017 (agregar school_id a users)

### PUNTOS POSITIVOS
- Estructura modular clara (PostgreSQL + MongoDB)
- Migraciones versionadas con up/down
- Seeds y testing data separados
- Constraints y validaciones robustas (ciclos, jerarquías)
- Documentación técnica completa (ARCHITECTURE.md)

### ÁREAS DE ATENCIÓN
1. **Sincronización Main-Dev:** Main y dev están sincronizados (sin diferencias pendientes)
2. **Feature Flags:** Sistema completo implementado para control remoto de features
3. **FASE 1 UI:** Tablas user_active_context, user_favorites, user_activity_log listos

### RECOMENDACIONES
1. ✅ Mantener centralización en `edugo-infrastructure`
2. ✅ Continuar usando arquitectura de 4 capas
3. ✅ Documentar nuevas migraciones en este formato
4. ⚠️ Considerar agregar TTL a colecciones MongoDB de logs/analytics
5. ⚠️ Revisar política de retención para audit_logs y analytics_events

---

## ANEXO: ESTRUCTURA DE DIRECTORIOS

```
edugo-infrastructure/
├── postgres/
│   ├── cmd/
│   │   ├── migrate/          # CLI migraciones (up, down, status)
│   │   └── runner/           # Runner 4 capas
│   ├── migrations/
│   │   ├── structure/        # Capa 1: DDL (15 archivos)
│   │   ├── constraints/      # Capa 2: FKs, Triggers (15 archivos)
│   │   ├── seeds/            # Capa 3: Datos iniciales (1 archivo)
│   │   ├── testing/          # Capa 4: Datos demo (10 archivos)
│   │   ├── 001-017_*.up.sql  # Migraciones versionadas (17 up)
│   │   └── 001-017_*.down.sql # Migraciones down (17 down)
├── mongodb/
│   ├── cmd/
│   │   └── migrate/          # CLI migraciones
│   ├── migrations/
│   │   ├── structure/        # Capa 1: Colecciones (9 archivos .go)
│   │   ├── constraints/      # Capa 2: Índices (9 archivos .go)
│   │   ├── seeds.go          # Capa 3: ApplySeeds()
│   │   └── mock_data.go      # Capa 4: ApplyMockData()
├── seeds/
│   ├── postgres/             # Seeds adicionales (3 archivos)
│   └── mongodb/              # Seeds MongoDB
├── schemas/                  # Schemas compartidos
├── scripts/                  # Scripts de automatización
├── docker/                   # Docker compose
└── ARCHITECTURE.md           # Documentación técnica

**Total Archivos SQL PostgreSQL:** 72+
**Total Archivos Go MongoDB:** 18+
**Total Migraciones PostgreSQL:** 17 (versionadas)
**Total Colecciones MongoDB:** 9
```

---

## METADATA DEL INFORME

- **Generado por:** Claude Agent (Análisis Exhaustivo Infrastructure)
- **Fecha:** 2025-12-24 17:35:00 CLT
- **Duración Análisis:** ~15 minutos
- **Archivos Analizados:** 90+
- **Líneas de Código Revisadas:** 5000+

---

**FIN DEL INFORME**
