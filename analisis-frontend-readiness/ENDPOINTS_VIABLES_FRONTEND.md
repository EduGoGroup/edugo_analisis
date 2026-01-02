# INFORME: ENDPOINTS VIABLES PARA DESARROLLO FRONTEND

**Fecha de Generación:** 2025-12-24
**Proyecto:** EduGo Platform
**Versión:** Consolidado API Admin + API Mobile
**Analista:** Claude Sonnet 4.5

---

## 📊 RESUMEN EJECUTIVO

### Totales por Estado

| Estado | API Admin | API Mobile | Total |
|--------|-----------|------------|-------|
| ✅ LISTO | 28 | 9 | 37 |
| ⚠️ PARCIAL | 10 | 6 | 16 |
| ❌ BLOQUEADO | 0 | 3 | 3 |
| 🔧 STUB | 0 | 0 | 0 |
| **TOTAL** | **38** | **18** | **56** |

### Calificación de Readiness

| API | Readiness | Comentario |
|-----|-----------|-----------|
| **API Admin** | 8.5/10 | ✅ Listo para producción (rama dev). Requiere sync dev→main y configurar CORS |
| **API Mobile** | 7.0/10 | ⚠️ Parcialmente listo. Dependencia crítica del Worker para assessments/summaries |
| **ECOSISTEMA** | 7.8/10 | ⚠️ Frontend puede comenzar CON precauciones |

---

## 🔴 BLOQUEOS CRÍTICOS A RESOLVER

### 1. Rama dev NO sincronizada con main (API Admin)
**Impacto:** ALTO
**Descripción:**
- Rama `dev` tiene 7,900 líneas más que `main`
- Endpoints de Subjects y Guardians NO están en `main`
- Swagger desactualizado en `main`

**Endpoints afectados:**
- `POST /v1/subjects`
- `GET /v1/subjects`
- `GET /v1/subjects/:id`
- `PATCH /v1/subjects/:id`
- `DELETE /v1/subjects/:id`
- `POST /v1/guardian-relations`
- `GET /v1/guardian-relations/:id`
- `PUT /v1/guardian-relations/:id`
- `DELETE /v1/guardian-relations/:id`
- `GET /v1/guardians/:guardian_id/relations`
- `GET /v1/students/:student_id/guardians`

**Acción requerida:**
```bash
# Merge dev → main
cd edugo-api-administracion
git checkout main
git merge dev
git push origin main
```

**Responsable:** Backend Team + DevOps
**Prioridad:** 🔴 CRÍTICA (antes de frontend)

---

### 2. CORS no configurado en main.go (API Admin)
**Impacto:** ALTO
**Descripción:**
- El middleware CORS existe en `router.go` pero NO se usa en `main.go`
- Frontend no podrá hacer requests cross-origin

**Acción requerida:**
```go
// En cmd/main.go, agregar:
r.Use(corsMiddleware())

func corsMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Writer.Header().Set("Access-Control-Allow-Origin", "*")
        c.Writer.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, PATCH, DELETE, OPTIONS")
        c.Writer.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")

        if c.Request.Method == "OPTIONS" {
            c.AbortWithStatus(204)
            return
        }

        c.Next()
    }
}
```

**Responsable:** Backend Team
**Prioridad:** 🔴 CRÍTICA (antes de frontend)

---

### 3. Worker NO disponible bloquea features (API Mobile)
**Impacto:** ALTO
**Descripción:**
- Endpoints de assessments y summaries dependen 100% del worker
- Worker procesa PDFs con IA (OpenAI/Claude) para generar contenido en MongoDB

**Endpoints bloqueados sin Worker:**
- `GET /v1/materials/:id/assessment` → 404 Not Found
- `POST /v1/materials/:id/assessment/attempts` → 404 Not Found
- `GET /v1/materials/:id/summary` → 404 Not Found

**Tiempo estimado de procesamiento:**
- PDF 10 páginas: ~30-60 segundos
- PDF 50 páginas: ~2-5 minutos
- PDF 100 páginas: ~5-10 minutos

**Acción requerida:**
1. Verificar que el worker esté desplegado y funcional
2. Monitorear eventos RabbitMQ: `material.uploaded` → `assessment.generated`
3. Validar colecciones MongoDB: `material_assessment_worker`, `material_summary`

**Responsable:** DevOps + Worker Team
**Prioridad:** 🔴 CRÍTICA (bloqueante para feature completa)

---

## 📋 API ADMINISTRACIÓN (Puerto 8081)

### Base URL
```
http://localhost:8081/v1
```

### Swagger UI
```
http://localhost:8081/swagger/index.html
```

---

### 🔐 AUTENTICACIÓN (Públicos)

| Método | Endpoint | Estado | Notas |
|--------|----------|--------|-------|
| POST | `/v1/auth/login` | ✅ LISTO | Retorna JWT + refresh token |
| POST | `/v1/auth/refresh` | ✅ LISTO | Renueva access token |
| POST | `/v1/auth/logout` | ✅ LISTO | Invalida sesión |
| POST | `/v1/auth/verify` | ✅ LISTO | Para servicios internos |

**DTOs:**
```typescript
// LoginRequest
{
  email: string;
  password: string;
}

// LoginResponse
{
  access_token: string;
  refresh_token: string;
  expires_in: number; // segundos
  user: {
    id: string;
    email: string;
    name: string;
    role: string;
  }
}
```

**Ejemplo de uso:**
```typescript
const response = await axios.post('/v1/auth/login', {
  email: 'admin@school.edu',
  password: 'password123'
});

const token = response.data.access_token;

// Usar en requests subsecuentes
axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
```

---

### 🏫 SCHOOLS (Escuelas)

| Método | Endpoint | Estado | Notas |
|--------|----------|--------|-------|
| POST | `/v1/schools` | ✅ LISTO | Requiere JWT |
| GET | `/v1/schools` | ⚠️ PARCIAL | Sin paginación, retorna todas |
| GET | `/v1/schools/code/:code` | ✅ LISTO | Búsqueda por código |
| GET | `/v1/schools/:id` | ✅ LISTO | Por UUID |
| PUT | `/v1/schools/:id` | ✅ LISTO | Actualización completa |
| DELETE | `/v1/schools/:id` | ✅ LISTO | Soft delete |

**DTOs:**
```typescript
// CreateSchoolRequest
{
  name: string; // min=3
  code: string; // min=3
  address?: string;
  city?: string;
  country?: string; // default: "CO"
  contact_email?: string;
  contact_phone?: string;
  subscription_tier?: "free" | "basic" | "premium"; // default: "free"
  max_teachers?: number; // default: 50
  max_students?: number; // default: 500
  metadata?: Record<string, any>;
}

// SchoolResponse
{
  id: string; // UUID
  name: string;
  code: string;
  address: string;
  city: string;
  country: string;
  contact_email: string;
  contact_phone: string;
  subscription_tier: string;
  max_teachers: number;
  max_students: number;
  is_active: boolean;
  metadata: Record<string, any>;
  created_at: string; // ISO8601
  updated_at: string; // ISO8601
}
```

**Limitación:**
- ⚠️ `GET /v1/schools` retorna TODAS las escuelas sin paginación
- Recomendación: Implementar filtros y paginación antes de producción

---

### 🎓 ACADEMIC UNITS (Jerarquía Académica)

| Método | Endpoint | Estado | Notas |
|--------|----------|--------|-------|
| POST | `/v1/schools/:id/units` | ✅ LISTO | Crear unidad académica |
| GET | `/v1/schools/:id/units` | ⚠️ PARCIAL | Acepta `?includeDeleted=bool` |
| GET | `/v1/schools/:id/units/tree` | ✅ LISTO | 🌳 **Jerarquía completa** |
| GET | `/v1/schools/:id/units/by-type` | ✅ LISTO | Filtro por tipo |
| GET | `/v1/units/:id` | ✅ LISTO | Unidad específica |
| PUT | `/v1/units/:id` | ✅ LISTO | Actualizar unidad |
| DELETE | `/v1/units/:id` | ✅ LISTO | Soft delete |
| POST | `/v1/units/:id/restore` | ✅ LISTO | Restaurar eliminada |
| GET | `/v1/units/:id/hierarchy-path` | ✅ LISTO | Ruta ltree completa |

**DTOs:**
```typescript
// CreateAcademicUnitRequest
{
  parent_unit_id?: string; // UUID (null para raíz)
  type: string; // Validado por valueobject.ParseUnitType
  display_name: string; // min=3, max=255
  code?: string; // min=2, max=50
  description?: string;
  metadata?: Record<string, any>;
}

// AcademicUnitResponse
{
  id: string;
  parent_unit_id: string | null;
  school_id: string;
  type: string;
  display_name: string;
  code: string;
  description: string;
  metadata: Record<string, any>;
  created_at: string;
  updated_at: string;
  deleted_at: string | null;
}

// UnitTreeNode (jerarquía)
{
  id: string;
  type: string;
  display_name: string;
  code: string;
  depth: number;
  children: UnitTreeNode[]; // Recursivo
}
```

**Feature Destacada:**
- 🌳 `GET /v1/schools/:id/units/tree` retorna árbol completo usando PostgreSQL ltree
- Útil para renderizar jerarquías en UI (campus → nivel → grado → sección)

**Ejemplo de uso:**
```typescript
// Obtener árbol completo
const tree = await api.get(`/v1/schools/${schoolId}/units/tree`);

// Renderizar recursivamente
function renderTree(nodes: UnitTreeNode[]) {
  return nodes.map(node => (
    <TreeNode key={node.id} label={node.display_name} depth={node.depth}>
      {node.children && renderTree(node.children)}
    </TreeNode>
  ));
}
```

---

### 👥 MEMBERSHIPS (Membresías a Unidades)

| Método | Endpoint | Estado | Notas |
|--------|----------|--------|-------|
| POST | `/v1/memberships` | ✅ LISTO | Asignar usuario a unidad |
| GET | `/v1/memberships` | ⚠️ PARCIAL | Query params: `?unit_id=uuid&activeOnly=bool` |
| GET | `/v1/memberships/by-role` | ✅ LISTO | Filtrar por rol |
| GET | `/v1/memberships/:id` | ✅ LISTO | Membresía específica |
| PUT | `/v1/memberships/:id` | ✅ LISTO | Actualizar membresía |
| DELETE | `/v1/memberships/:id` | ✅ LISTO | Hard delete |
| POST | `/v1/memberships/:id/expire` | ✅ LISTO | Expirar membresía |
| GET | `/v1/users/:userId/memberships` | ✅ LISTO | Membresías de usuario |

**DTOs:**
```typescript
// CreateMembershipRequest
{
  unit_id: string; // UUID
  user_id: string; // UUID
  role: string; // owner, teacher, assistant, student, guardian
  valid_from?: string; // ISO8601
  valid_until?: string; // ISO8601
}

// MembershipResponse
{
  id: string;
  unit_id: string;
  user_id: string;
  role: string;
  enrolled_at: string;
  withdrawn_at: string | null;
  is_active: boolean;
  created_at: string;
  updated_at: string;
}
```

**Roles disponibles:**
- `owner` - Dueño de la unidad
- `teacher` - Docente
- `assistant` - Asistente
- `student` - Estudiante
- `guardian` - Acudiente/Tutor

**Nota:**
- ⚠️ No hay middleware de autorización por rol. Cualquier usuario autenticado puede modificar membresías.
- Recomendación: Implementar RBAC en backend

---

### 📚 SUBJECTS (Materias) - ⚠️ SOLO EN DEV

| Método | Endpoint | Estado | Notas |
|--------|----------|--------|-------|
| POST | `/v1/subjects` | ⚠️ PARCIAL | Solo en rama dev |
| GET | `/v1/subjects` | ⚠️ PARCIAL | Query: `?school_id=uuid` |
| GET | `/v1/subjects/:id` | ⚠️ PARCIAL | Solo en dev |
| PATCH | `/v1/subjects/:id` | ⚠️ PARCIAL | Solo en dev |
| DELETE | `/v1/subjects/:id` | ⚠️ PARCIAL | Soft delete, solo dev |

**⚠️ ADVERTENCIA:**
- Estos endpoints NO están en rama `main`
- Requiere merge dev → main antes de usar en producción
- Verificar con DevOps qué rama está desplegada

**DTOs:**
```typescript
// CreateSubjectRequest
{
  name: string; // min=2
  description?: string;
  metadata?: string;
}

// SubjectResponse
{
  id: string;
  name: string;
  description: string;
  metadata: string;
  is_active: boolean;
  created_at: string;
  updated_at: string;
}
```

---

### 👨‍👩‍👧 GUARDIAN RELATIONS (Acudientes) - ⚠️ SOLO EN DEV

| Método | Endpoint | Estado | Notas |
|--------|----------|--------|-------|
| POST | `/v1/guardian-relations` | ⚠️ PARCIAL | Solo en dev |
| GET | `/v1/guardian-relations/:id` | ⚠️ PARCIAL | Solo en dev |
| PUT | `/v1/guardian-relations/:id` | ⚠️ PARCIAL | Solo en dev |
| DELETE | `/v1/guardian-relations/:id` | ⚠️ PARCIAL | Soft delete, solo dev |
| GET | `/v1/guardians/:guardian_id/relations` | ⚠️ PARCIAL | Relaciones de acudiente |
| GET | `/v1/students/:student_id/guardians` | ⚠️ PARCIAL | Acudientes de estudiante |

**⚠️ ADVERTENCIA:**
- Estos endpoints NO están en rama `main`
- Requiere merge dev → main antes de usar en producción

**DTOs:**
```typescript
// CreateGuardianRelationRequest
{
  guardian_id: string; // UUID
  student_id: string; // UUID
  relationship_type: "father" | "mother" | "grandfather" | "grandmother" | "uncle" | "aunt" | "other";
}

// GuardianRelationResponse
{
  id: string;
  guardian_id: string;
  student_id: string;
  relationship_type: string;
  is_active: boolean;
  created_at: string;
  updated_at: string;
  created_by: string; // UUID del admin
}
```

---

### 🏥 Health Check (API Admin)

| Método | Endpoint | Estado | Notas |
|--------|----------|--------|-------|
| GET | `/health` | ⚠️ PARCIAL | No valida PostgreSQL |

**Response:**
```json
{
  "status": "healthy",
  "service": "edugo-api-admin"
}
```

**Limitación:**
- ⚠️ No valida conexión a PostgreSQL
- Solo responde 200 si el servidor está up
- Recomendación: Mejorar health check para validar dependencias

---

## 📱 API MOBILE (Puerto 8080)

### Base URL
```
http://localhost:8080/v1
```

### Swagger UI
```
http://localhost:8080/swagger/index.html
```

---

### 🏥 Health Check (API Mobile)

| Método | Endpoint | Estado | Notas |
|--------|----------|--------|-------|
| GET | `/health` | ✅ LISTO | Modo simple |
| GET | `/health?detail=1` | ✅ LISTO | Modo detallado con latencias |

**Response (simple):**
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

**Response (detallado):**
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

**Lógica de estado:**
- `healthy`: Todos los componentes REQUIRED (PostgreSQL, MongoDB) están OK
- `unhealthy`: Al menos un componente REQUIRED falló
- Componentes OPTIONAL (RabbitMQ, S3) no afectan el status general

---

### 📚 MATERIALS (Materiales de Estudio)

| Método | Endpoint | Estado | Notas |
|--------|----------|--------|-------|
| GET | `/v1/materials` | ⚠️ PARCIAL | Sin paginación, retorna todos |
| POST | `/v1/materials` | ✅ LISTO | Requiere role=teacher+ |
| GET | `/v1/materials/:id` | ✅ LISTO | Material específico |
| PUT | `/v1/materials/:id` | ⚠️ PARCIAL | **NUEVO** Solo en rama dev |
| GET | `/v1/materials/:id/versions` | ✅ LISTO | Historial de versiones |
| POST | `/v1/materials/:id/upload-url` | ✅ LISTO | URL presignada S3 (15 min) |
| GET | `/v1/materials/:id/download-url` | ✅ LISTO | URL presignada S3 (15 min) |
| POST | `/v1/materials/:id/upload-complete` | ✅ LISTO | Dispara worker |

**DTOs:**
```typescript
// CreateMaterialRequest
{
  title: string; // min=3, max=200
  description?: string; // max=1000
  subject?: string;
  grade?: string;
}

// UpdateMaterialRequest (NUEVO - solo dev)
{
  title?: string; // min=3, max=200
  description?: string; // max=1000
  subject?: string;
  grade?: string;
  academic_unit_id?: string;
  is_public?: boolean;
}

// MaterialResponse
{
  id: string;
  school_id: string;
  uploaded_by_teacher_id: string;
  academic_unit_id: string | null;
  title: string;
  description: string;
  subject: string;
  grade: string;
  file_url: string | null;
  file_type: string;
  file_size_bytes: number;
  status: "uploaded" | "processing" | "ready" | "failed";
  is_public: boolean;
  processing_started_at: string | null;
  processing_completed_at: string | null;
  created_at: string;
  updated_at: string;
  deleted_at: string | null;
}

// GenerateUploadURLRequest
{
  file_name: string; // Sin path traversal
  content_type: string; // e.g., "application/pdf"
}

// GenerateUploadURLResponse
{
  upload_url: string; // Presigned URL válida por 15 minutos
  file_url: string; // URL final del archivo
  expires_in: number; // Segundos
}

// UploadCompleteRequest
{
  file_url: string;
  file_type: string;
  file_size_bytes: number;
}
```

**Flujo completo de Upload:**
```typescript
// 1. Crear material
const material = await api.post('/v1/materials', {
  title: 'Matemáticas Grado 5',
  description: 'Álgebra básica',
  subject: 'Mathematics',
  grade: '5'
});

// 2. Generar URL de subida
const uploadData = await api.post(`/v1/materials/${material.id}/upload-url`, {
  file_name: 'algebra-basica.pdf',
  content_type: 'application/pdf'
});

// 3. Subir PDF a S3 (sin autenticación, directo a AWS)
await axios.put(uploadData.upload_url, pdfFile, {
  headers: { 'Content-Type': 'application/pdf' }
});

// 4. Notificar completitud (dispara worker)
await api.post(`/v1/materials/${material.id}/upload-complete`, {
  file_url: uploadData.file_url,
  file_type: 'application/pdf',
  file_size_bytes: pdfFile.size
});

// 5. Polling para esperar procesamiento
async function waitForReady(materialId: string) {
  for (let i = 0; i < 60; i++) { // 5 minutos
    const mat = await api.get(`/v1/materials/${materialId}`);
    if (mat.status === 'ready') return mat;
    if (mat.status === 'failed') throw new Error('Processing failed');
    await sleep(5000); // Esperar 5 segundos
  }
  throw new Error('Timeout');
}

const readyMaterial = await waitForReady(material.id);
```

**Eventos emitidos:**
- `material.uploaded` → RabbitMQ (procesado por worker)
- `material.completed` → RabbitMQ (cuando usuario completa 100%)

---

### 📝 ASSESSMENTS (Evaluaciones/Quizzes) - ⚠️ DEPENDE DEL WORKER

| Método | Endpoint | Estado | Notas |
|--------|----------|--------|-------|
| GET | `/v1/materials/:id/assessment` | ❌ BLOQUEADO | Requiere worker activo |
| POST | `/v1/materials/:id/assessment/attempts` | ❌ BLOQUEADO | Requiere worker activo |
| GET | `/v1/attempts/:id/results` | ✅ LISTO | Después de crear intento |
| GET | `/v1/users/me/attempts` | ✅ LISTO | Historial paginado |

**⚠️ DEPENDENCIA CRÍTICA:**
- El worker procesa el PDF con IA (OpenAI/Claude)
- Genera preguntas de opción múltiple en MongoDB
- Sin worker: `404 Not Found`

**DTOs:**
```typescript
// AssessmentResponse (GET /assessment)
{
  id: string;
  material_id: string;
  title: string;
  questions: QuestionDTO[];
  questions_count: number;
  total_questions: number;
  max_attempts: number;
  pass_threshold: number; // Porcentaje (ej: 70)
  time_limit_minutes: number;
  estimated_time_minutes: number;
}

// QuestionDTO (SIN respuesta correcta)
{
  id: string;
  text: string;
  type: "multiple_choice";
  options: OptionDTO[];
}

// OptionDTO
{
  id: string;
  text: string;
}

// CreateAttemptRequest
{
  answers: UserAnswerDTO[];
  time_spent_seconds: number; // min=1, max=7200
}

// UserAnswerDTO
{
  question_id: string;
  selected_answer_id: string;
  time_spent_seconds: number; // >= 0
}

// AttemptResultResponse
{
  attempt_id: string;
  score: number;
  max_score: number;
  passed: boolean;
  feedback: AnswerFeedbackDTO[];
  can_retake: boolean;
  attempts_used: number;
  max_attempts: number;
}

// AnswerFeedbackDTO
{
  question_id: string;
  is_correct: boolean;
  correct_answer: string; // ID de la opción correcta
  selected_answer: string;
  explanation: string;
}

// AttemptHistoryResponse (GET /users/me/attempts)
{
  attempts: AttemptSummaryDTO[];
  total_count: number;
  limit: number;
  offset: number;
}
```

**Flujo de Assessment:**
```typescript
// 1. Verificar que material esté listo
const material = await api.get(`/v1/materials/${materialId}`);
if (material.status !== 'ready') {
  showMessage('El material está siendo procesado...');
  return;
}

// 2. Intentar obtener assessment
try {
  const assessment = await api.get(`/v1/materials/${materialId}/assessment`);

  // 3. Renderizar quiz (SIN mostrar respuestas correctas)
  renderQuiz(assessment);

  // 4. Usuario responde
  const answers = collectAnswers();

  // 5. Enviar intento
  const result = await api.post(`/v1/materials/${materialId}/assessment/attempts`, {
    answers: answers,
    time_spent_seconds: totalTime
  });

  // 6. Mostrar resultados con feedback
  showResults(result);

} catch (error) {
  if (error.response?.status === 404) {
    showMessage('El quiz está siendo generado. Intenta en unos minutos.');
  } else {
    throw error;
  }
}
```

**Tiempo estimado de generación:**
- PDF 10 páginas: ~30-60 segundos
- PDF 50 páginas: ~2-5 minutos
- PDF 100 páginas: ~5-10 minutos

---

### 📖 SUMMARIES (Resúmenes IA) - ⚠️ DEPENDE DEL WORKER

| Método | Endpoint | Estado | Notas |
|--------|----------|--------|-------|
| GET | `/v1/materials/:id/summary` | ❌ BLOQUEADO | Requiere worker activo |

**⚠️ DEPENDENCIA CRÍTICA:**
- El worker genera resumen con IA
- Sin worker: `404 Not Found`

**Response (estructura dinámica):**
```json
{
  "material_id": "uuid",
  "summary_text": "Resumen generado por IA...",
  "key_points": [
    "Punto importante 1",
    "Punto importante 2"
  ],
  "generated_at": "2024-12-23T10:02:00Z",
  "model_version": "gpt-4",
  "metadata": {}
}
```

**Manejo en Frontend:**
```typescript
async function getSummaryIfAvailable(materialId: string) {
  try {
    return await api.get(`/v1/materials/${materialId}/summary`);
  } catch (error) {
    if (error.response?.status === 404) {
      return null; // Resumen aún no generado
    }
    throw error;
  }
}

// Uso
const summary = await getSummaryIfAvailable(materialId);
if (summary) {
  displaySummary(summary);
} else {
  showMessage('El resumen estará disponible en unos minutos...');
}
```

---

### 📈 PROGRESS (Progreso de Lectura)

| Método | Endpoint | Estado | Notas |
|--------|----------|--------|-------|
| PUT | `/v1/progress` | ✅ LISTO | **UPSERT** idempotente |

**DTOs:**
```typescript
// UpsertProgressRequest
{
  user_id: string; // UUID (debe coincidir con JWT o ser admin)
  material_id: string; // UUID
  progress_percentage: number; // min=0, max=100
  last_page?: number;
}

// ProgressResponse
{
  user_id: string;
  material_id: string;
  progress_percentage: number;
  last_page: number;
  updated_at: string;
}
```

**Características:**
- ✅ Operación idempotente (UPSERT)
- ✅ Usuario solo puede actualizar su propio progreso
- ✅ Admin puede actualizar progreso de cualquiera
- ✅ Emite evento `material.completed` cuando llega a 100%

**Ejemplo de uso:**
```typescript
// Actualizar progreso cada X segundos mientras lee
async function updateProgress(materialId: string, percentage: number, page: number) {
  await api.put('/v1/progress', {
    user_id: currentUser.id, // Del JWT
    material_id: materialId,
    progress_percentage: percentage,
    last_page: page
  });
}

// Llamar al pasar página
onPageChange((page, totalPages) => {
  const percentage = Math.round((page / totalPages) * 100);
  updateProgress(materialId, percentage, page);
});
```

---

### 📊 STATS (Estadísticas)

| Método | Endpoint | Estado | Notas |
|--------|----------|--------|-------|
| GET | `/v1/materials/:id/stats` | ✅ LISTO | Estadísticas de material |
| GET | `/v1/stats/global` | ✅ LISTO | Solo admin |

**DTOs:**
```typescript
// Material Stats Response
{
  material_id: string;
  total_views: number;
  completion_rate: number; // Porcentaje
  average_score: number;
  total_attempts: number;
}

// Global Stats Response (solo admin)
{
  total_materials: number;
  total_attempts: number;
  global_average_score: number;
}
```

---

### 📊 Metrics (Prometheus)

| Método | Endpoint | Estado | Notas |
|--------|----------|--------|-------|
| GET | `/metrics` | ✅ LISTO | Formato Prometheus |

**Métricas expuestas:**
- Request count
- Request duration
- Request errors
- Database connection pool
- RabbitMQ message count

**No requiere autenticación** (configurar firewall en producción)

---

## 🔒 SEGURIDAD Y AUTENTICACIÓN

### JWT Bearer Token (Centralizado)

**Flujo:**
1. Login en API Admin → Recibir JWT
2. Usar mismo JWT en API Mobile
3. Validación LOCAL en API Mobile (sin llamadas HTTP)

**Claims en JWT:**
```typescript
{
  sub: string; // user_id (UUID)
  email: string;
  role: "student" | "teacher" | "admin" | "super_admin";
  school_id: string; // UUID
  iss: "edugo-central";
  exp: number; // Unix timestamp
  iat: number; // Unix timestamp
}
```

**Middleware de Autorización:**

| Middleware | Restricción |
|------------|-------------|
| `JWTAuthMiddleware` | Requiere JWT válido |
| `RequireTeacher` | Role debe ser teacher, admin o super_admin |
| `RequireAdmin` | Role debe ser admin o super_admin |

**⚠️ NOTA IMPORTANTE:**
- API Admin NO tiene middleware de autorización por rol
- Cualquier usuario autenticado puede acceder a todos los endpoints
- Recomendación: Implementar RBAC en API Admin

---

## 📋 CHECKLIST DE INTEGRACIÓN FRONTEND

### Antes de Empezar

- [ ] **Verificar qué rama está desplegada:**
  - [ ] API Admin: ¿dev o main?
  - [ ] API Mobile: ¿dev o main?
- [ ] **Validar servicios externos:**
  - [ ] Worker está desplegado y funcional
  - [ ] RabbitMQ está configurado
  - [ ] S3 bucket está accesible
  - [ ] PostgreSQL y MongoDB están up
- [ ] **Configurar CORS:**
  - [ ] Verificar que API Admin tenga CORS habilitado
  - [ ] Verificar que API Mobile tenga CORS habilitado
- [ ] **Obtener credenciales de prueba:**
  - [ ] Usuario teacher para crear materiales
  - [ ] Usuario student para consumir materiales
  - [ ] Usuario admin para stats globales

---

### Implementación por Feature

#### ✅ Feature: Autenticación
- [ ] Login (POST /v1/auth/login)
- [ ] Almacenar JWT en localStorage/sessionStorage
- [ ] Agregar interceptor para incluir Authorization header
- [ ] Refresh token automático antes de expiración
- [ ] Logout (POST /v1/auth/logout)
- [ ] Redirección a login en 401

#### ✅ Feature: Gestión de Escuelas (Admin)
- [ ] Listar escuelas (GET /v1/schools)
- [ ] Crear escuela (POST /v1/schools)
- [ ] Editar escuela (PUT /v1/schools/:id)
- [ ] Ver escuela (GET /v1/schools/:id)
- [ ] Buscar por código (GET /v1/schools/code/:code)
- [ ] Eliminar escuela (DELETE /v1/schools/:id)

#### ✅ Feature: Jerarquía Académica (Admin)
- [ ] Crear unidad académica (POST /v1/schools/:id/units)
- [ ] Listar unidades (GET /v1/schools/:id/units)
- [ ] **Renderizar árbol jerárquico** (GET /v1/schools/:id/units/tree)
- [ ] Filtrar por tipo (GET /v1/schools/:id/units/by-type)
- [ ] Editar unidad (PUT /v1/units/:id)
- [ ] Eliminar unidad (DELETE /v1/units/:id)
- [ ] Restaurar unidad (POST /v1/units/:id/restore)

#### ✅ Feature: Membresías (Admin)
- [ ] Asignar usuario a unidad (POST /v1/memberships)
- [ ] Listar membresías de unidad (GET /v1/memberships)
- [ ] Filtrar por rol (GET /v1/memberships/by-role)
- [ ] Ver membresías de usuario (GET /v1/users/:userId/memberships)
- [ ] Editar membresía (PUT /v1/memberships/:id)
- [ ] Eliminar membresía (DELETE /v1/memberships/:id)
- [ ] Expirar membresía (POST /v1/memberships/:id/expire)

#### ⚠️ Feature: Materias (Admin - Solo dev)
- [ ] Verificar si endpoints están disponibles (depende de rama)
- [ ] Listar materias (GET /v1/subjects)
- [ ] Crear materia (POST /v1/subjects)
- [ ] Editar materia (PATCH /v1/subjects/:id)
- [ ] Eliminar materia (DELETE /v1/subjects/:id)

#### ⚠️ Feature: Acudientes (Admin - Solo dev)
- [ ] Verificar si endpoints están disponibles (depende de rama)
- [ ] Crear relación acudiente-estudiante (POST /v1/guardian-relations)
- [ ] Ver relaciones de acudiente (GET /v1/guardians/:guardian_id/relations)
- [ ] Ver acudientes de estudiante (GET /v1/students/:student_id/guardians)
- [ ] Editar relación (PUT /v1/guardian-relations/:id)
- [ ] Eliminar relación (DELETE /v1/guardian-relations/:id)

#### ✅ Feature: Materiales (Mobile)
- [ ] Listar materiales (GET /v1/materials)
- [ ] Ver material (GET /v1/materials/:id)
- [ ] **Crear material (docente):**
  - [ ] POST /v1/materials
  - [ ] POST /v1/materials/:id/upload-url
  - [ ] Upload PDF a S3 (PUT presigned URL)
  - [ ] POST /v1/materials/:id/upload-complete
  - [ ] Polling status hasta 'ready'
- [ ] Actualizar material (PUT /v1/materials/:id) - Solo dev
- [ ] Ver historial de versiones (GET /v1/materials/:id/versions)
- [ ] Descargar PDF (GET /v1/materials/:id/download-url)

#### ⚠️ Feature: Assessments (Mobile - Requiere Worker)
- [ ] **Verificar status='ready' antes de intentar**
- [ ] Obtener assessment (GET /v1/materials/:id/assessment)
- [ ] Manejo de 404 (mostrar "Generando quiz...")
- [ ] Renderizar quiz sin respuestas correctas
- [ ] Crear intento (POST /v1/materials/:id/assessment/attempts)
- [ ] Mostrar resultados con feedback
- [ ] Ver historial de intentos (GET /v1/users/me/attempts)
- [ ] Paginación de historial (query params: limit, offset)

#### ⚠️ Feature: Resúmenes IA (Mobile - Requiere Worker)
- [ ] **Verificar status='ready' antes de intentar**
- [ ] Obtener resumen (GET /v1/materials/:id/summary)
- [ ] Manejo de 404 (mostrar "Generando resumen...")
- [ ] Renderizar resumen con key points

#### ✅ Feature: Progreso de Lectura (Mobile)
- [ ] Actualizar progreso (PUT /v1/progress)
- [ ] Mostrar barra de progreso
- [ ] Detectar completitud (100%)
- [ ] Persistir última página leída

#### ✅ Feature: Estadísticas (Mobile)
- [ ] Stats de material (GET /v1/materials/:id/stats)
- [ ] Stats globales (GET /v1/stats/global) - Solo admin
- [ ] Visualización con gráficas

#### ✅ Feature: Health Checks
- [ ] API Admin health (GET /health)
- [ ] API Mobile health (GET /health?detail=1)
- [ ] Mostrar estado de componentes en panel admin

---

### Manejo de Errores

- [ ] **401 Unauthorized:**
  - [ ] Limpiar JWT
  - [ ] Redirigir a login
- [ ] **403 Forbidden:**
  - [ ] Mostrar mensaje "No tienes permisos"
  - [ ] Deshabilitar acciones según rol
- [ ] **404 Not Found:**
  - [ ] Distinguir: recurso no existe vs worker procesando
  - [ ] Mostrar mensaje apropiado
- [ ] **409 Conflict:**
  - [ ] Mostrar mensaje "Ya existe"
- [ ] **422 Validation Error:**
  - [ ] Mostrar errores por campo
- [ ] **500 Internal Server Error:**
  - [ ] Mostrar mensaje genérico
  - [ ] Log para debugging

---

## 🚨 ESTRATEGIAS PARA WORKER DEPENDENCY

### Polling de Status Material

```typescript
async function waitForMaterialReady(
  materialId: string,
  options = { maxAttempts: 60, intervalMs: 5000 }
): Promise<MaterialResponse> {

  for (let i = 0; i < options.maxAttempts; i++) {
    const material = await api.get(`/v1/materials/${materialId}`);

    switch (material.status) {
      case 'ready':
        return material; // Éxito

      case 'failed':
        throw new Error('Material processing failed');

      case 'processing':
        // Continuar esperando
        await sleep(options.intervalMs);
        break;

      case 'uploaded':
        // Aún no ha iniciado procesamiento
        await sleep(options.intervalMs);
        break;
    }
  }

  throw new Error('Timeout waiting for material processing');
}

// Uso
try {
  showLoader('Procesando material...');
  const material = await waitForMaterialReady(materialId);
  showSuccess('Material listo!');
  loadAssessment(materialId);
} catch (error) {
  showError('Error al procesar material');
}
```

---

### Progressive Enhancement UI

```typescript
// Mostrar contenido disponible inmediatamente
async function loadMaterialPage(materialId: string) {

  // 1. Cargar metadata (siempre disponible)
  const material = await api.get(`/v1/materials/${materialId}`);
  displayMaterialInfo(material);

  // 2. Mostrar PDF si existe (siempre disponible después de upload)
  if (material.file_url) {
    displayPDFViewer(material.file_url);
  }

  // 3. Intentar cargar resumen (puede no estar listo)
  try {
    const summary = await api.get(`/v1/materials/${materialId}/summary`);
    displaySummary(summary);
  } catch (error) {
    if (error.response?.status === 404) {
      showPlaceholder('Generando resumen con IA...', {
        icon: '🤖',
        action: 'retry',
        onRetry: () => loadMaterialPage(materialId)
      });
    }
  }

  // 4. Intentar cargar assessment (puede no estar listo)
  try {
    const assessment = await api.get(`/v1/materials/${materialId}/assessment`);
    displayAssessment(assessment);
  } catch (error) {
    if (error.response?.status === 404) {
      showPlaceholder('Generando quiz...', {
        icon: '📝',
        action: 'retry',
        onRetry: () => loadMaterialPage(materialId)
      });
    }
  }

  // 5. Cargar progreso del usuario (siempre disponible)
  const progress = await getUserProgress(materialId);
  displayProgressBar(progress.progress_percentage);
}
```

---

### Notificaciones en Tiempo Real (Futuro)

**⚠️ NO IMPLEMENTADO - Workaround actual: polling**

**Propuesta futura:**
```typescript
// WebSocket connection
const ws = new WebSocket(`ws://localhost:8080/v1/materials/${materialId}/processing`);

ws.onmessage = (event) => {
  const update = JSON.parse(event.data);

  switch (update.type) {
    case 'processing_started':
      showProgress(0, 'Procesando PDF...');
      break;

    case 'pdf_parsed':
      showProgress(25, 'PDF analizado');
      break;

    case 'summary_generated':
      showProgress(50, 'Resumen generado');
      reloadSummary();
      break;

    case 'assessment_generated':
      showProgress(75, 'Quiz generado');
      reloadAssessment();
      break;

    case 'completed':
      showProgress(100, 'Completado!');
      ws.close();
      break;

    case 'failed':
      showError(update.error);
      ws.close();
      break;
  }
};
```

---

## 📝 NOTAS FINALES

### Decisiones de Arquitectura

1. **Validación JWT Local (API Mobile):**
   - API Mobile usa el MISMO secret que API Admin
   - Validación local sin llamadas HTTP
   - Fallback a validación remota si falla

2. **Infraestructura Compartida:**
   - Entidades PostgreSQL centralizadas en `edugo-infrastructure`
   - Migraciones centralizadas (ningún proyecto las define)
   - Evita duplicación y inconsistencias

3. **Clean Architecture:**
   - Domain → Application → Infrastructure
   - Dependency Injection mediante containers
   - Interfaces de repositorio en domain

4. **Eventos Asíncronos:**
   - RabbitMQ con exchange type: topic
   - Publisher confirms habilitados
   - Request ID propagado para tracing distribuido

---

### Limitaciones Actuales

1. **Sin paginación en listas:**
   - `GET /v1/schools` retorna todas
   - `GET /v1/materials` retorna todos
   - `GET /v1/subjects` retorna todas
   - Implementar antes de producción

2. **Sin RBAC en API Admin:**
   - Middleware JWT valida autenticación
   - NO valida autorización por rol
   - Cualquier usuario autenticado puede modificar escuelas

3. **Sin WebSockets:**
   - Polling manual para status
   - UX mejorable
   - Propuesta: implementar WebSockets para progreso en tiempo real

4. **Health Check Básico (API Admin):**
   - No valida PostgreSQL
   - Solo responde 200 si servidor up

---

### Recomendaciones de Prioridad

#### 🔴 CRÍTICO (Bloqueante)
1. Merge dev → main en API Admin
2. Configurar CORS en main.go
3. Validar Worker desplegado y funcional
4. Configurar variables de entorno en todos los ambientes

#### 🟡 ALTA (Pre-producción)
1. Implementar RBAC en API Admin
2. Implementar paginación en listas
3. Mejorar health checks
4. Documentar contratos de eventos RabbitMQ

#### 🟢 MEDIA (Mejoras UX)
1. Implementar WebSockets para progreso
2. Cache de validaciones JWT
3. Rate limiting
4. Métricas de negocio en Prometheus

#### ⚪ BAJA (Nice to have)
1. Versionado de eventos
2. Schema registry para eventos
3. Dead letter queues en RabbitMQ
4. Circuit breakers en HTTP clients

---

## 📞 CONTACTO Y SOPORTE

### Responsables por Área

| Área | Responsable | Acción |
|------|-------------|--------|
| **Merge dev→main** | Backend Team | Coordinar con DevOps |
| **CORS Config** | Backend Team | Implementar en main.go |
| **Worker Status** | DevOps + Worker Team | Validar deployment |
| **Variables Env** | DevOps | Configurar en todos los ambientes |
| **Frontend Integration** | Frontend Team | Implementar checklist |

---

## 📚 DOCUMENTACIÓN ADICIONAL

### API Admin
- Swagger UI: `http://localhost:8081/swagger/index.html`
- Documentación auth: `/docs/auth/GUIA-INTEGRACION.md`
- Estrategia CI/CD: `/.github/workflows/docs/CI_CD_STRATEGY.md`

### API Mobile
- Swagger UI: `http://localhost:8080/swagger/index.html`
- Arquitectura: `/documents/ARCHITECTURE.md`
- Base de datos: `/documents/DATABASE.md`
- API Reference: `/documents/API-REFERENCE.md`
- Setup: `/documents/SETUP.md`
- Flows: `/documents/FLOWS.md`

---

**FIN DEL INFORME**

*Generado: 2025-12-24*
*Analista: Claude Sonnet 4.5*
*Versión: 1.0*
