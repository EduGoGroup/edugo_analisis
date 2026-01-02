# Análisis de Salud de Contratos/DTOs - Ecosistema EduGo

**Fecha de Análisis:** 2025-12-24
**Proyectos Analizados:**
- edugo-api-mobile (Publicador de eventos)
- edugo-worker (Consumidor de eventos)
- edugo-shared (DTOs y tipos compartidos)

---

## Resumen Ejecutivo

### Estado General: ⚠️ **PROBLEMAS CRÍTICOS DETECTADOS**

**Problemas Principales:**
1. **Desacoplamiento Total**: API Mobile y Worker tienen DTOs completamente diferentes para los mismos eventos
2. **Sin DTOs Compartidos**: No existe un contrato compartido en `edugo-shared` para los eventos
3. **Diferencias Estructurales**: Los campos entre publicador y consumidor NO coinciden
4. **Eventos Huérfanos**: Evento `material.completed` publicado pero sin consumidor
5. **Formato Inconsistente**: API Mobile usa formato punto (`material.uploaded`), Worker usa underscore (`material_uploaded`)

---

## Tabla de Eventos y Estado de Contratos

| Evento | Publicador | Consumidor | DTO Compartido | Estado | Riesgo |
|--------|-----------|-----------|---------------|---------|--------|
| `material.uploaded` | ✅ API Mobile | ✅ Worker (`material_uploaded`) | ❌ NO | ⚠️ **INCOMPATIBLE** | 🔴 ALTO |
| `material.completed` | ✅ API Mobile | ❌ NO | ❌ NO | ❌ **HUÉRFANO** | 🔴 ALTO |
| `assessment.generated` | ✅ API Mobile | ❌ NO | ❌ NO | ❌ **HUÉRFANO** | 🔴 ALTO |
| `material_deleted` | ❌ NO | ✅ Worker | ❌ NO | ⚠️ **SIN PUBLICADOR** | 🟡 MEDIO |
| `assessment_attempt` | ❌ NO | ✅ Worker | ❌ NO | ⚠️ **SIN PUBLICADOR** | 🟡 MEDIO |
| `student_enrolled` | ❌ NO | ✅ Worker | ❌ NO | ⚠️ **SIN PUBLICADOR** | 🟡 MEDIO |
| `material_reprocess` | ❌ NO | ✅ Worker | ❌ NO | ⚠️ **SIN PUBLICADOR** | 🟡 MEDIO |

---

## Análisis Detallado por Evento

### 1. material.uploaded / material_uploaded ⚠️

**Estado:** INCOMPATIBLE - Estructuras diferentes

#### API Mobile (Publicador)
**Ubicación:** `/edugo-api-mobile/internal/infrastructure/messaging/rabbitmq/events.go`

```go
type Event struct {
    EventID      string      `json:"event_id"`
    EventType    string      `json:"event_type"`
    EventVersion string      `json:"event_version"`
    Timestamp    time.Time   `json:"timestamp"`
    Payload      interface{} `json:"payload"`
}

type MaterialUploadedPayload struct {
    MaterialID    string                 `json:"material_id"`
    SchoolID      string                 `json:"school_id"`
    TeacherID     string                 `json:"teacher_id"`
    FileURL       string                 `json:"file_url"`
    FileSizeBytes int64                  `json:"file_size_bytes"`
    FileType      string                 `json:"file_type"`
    Metadata      map[string]interface{} `json:"metadata,omitempty"`
}
```

**Publicación:**
- Exchange: `edugo.materials`
- Routing Key: `material.uploaded`
- Archivo: `/edugo-api-mobile/internal/application/service/material_service.go:196`

#### Worker (Consumidor)
**Ubicación:** `/edugo-worker/internal/application/dto/event_dto.go`

```go
type MaterialUploadedEvent struct {
    EventType         string    `json:"event_type"`
    MaterialID        string    `json:"material_id"`
    AuthorID          string    `json:"author_id"`     // ⚠️ DIFERENTE: API usa TeacherID
    S3Key             string    `json:"s3_key"`        // ⚠️ DIFERENTE: API usa FileURL
    PreferredLanguage string    `json:"preferred_language"` // ⚠️ NO EXISTE en API
    Timestamp         time.Time `json:"timestamp"`
}
```

**Consumidor:**
- Event Type: `material_uploaded`
- Processor: `MaterialUploadedProcessor`
- Archivo: `/edugo-worker/internal/application/processor/material_uploaded_processor.go:328`

#### ❌ DISCREPANCIAS CRÍTICAS:

1. **Estructura Envelope:**
   - API Mobile: Usa envelope estándar con `event_id`, `event_version`, `payload`
   - Worker: Espera campos planos sin envelope

2. **Campos Faltantes en Worker:**
   - `school_id` - NO está en DTO del Worker
   - `file_url` - Worker espera `s3_key` en su lugar
   - `file_size_bytes` - NO está en DTO del Worker
   - `file_type` - NO está en DTO del Worker
   - `metadata` - NO está en DTO del Worker

3. **Campos Faltantes en API:**
   - `preferred_language` - API Mobile NO lo envía

4. **Campos con Nombres Diferentes:**
   - API: `teacher_id` → Worker: `author_id`
   - API: `file_url` → Worker: `s3_key`

5. **Formato Event Type:**
   - API: `material.uploaded` (punto)
   - Worker: `material_uploaded` (underscore)

**Impacto:** 🔴 **CRÍTICO** - El evento NO puede ser procesado correctamente por el Worker

---

### 2. material.completed ❌

**Estado:** HUÉRFANO - Publicado pero sin consumidor

#### API Mobile (Publicador)
**Ubicación:** `/edugo-api-mobile/internal/infrastructure/messaging/rabbitmq/events.go`

```go
type MaterialCompletedPayload struct {
    MaterialID  string    `json:"material_id"`
    SchoolID    string    `json:"school_id"`
    UserID      string    `json:"user_id"`
    CompletedAt time.Time `json:"completed_at"`
}
```

**Publicación:**
- Exchange: `edugo.events`
- Routing Key: `material.completed`
- Archivo: `/edugo-api-mobile/internal/application/service/progress_service.go:130`

#### Worker (Consumidor)
❌ **NO EXISTE CONSUMIDOR**

**Impacto:** 🔴 **ALTO** - Eventos se publican pero nunca se procesan. Funcionalidad perdida.

---

### 3. assessment.generated ❌

**Estado:** HUÉRFANO - Publicado pero sin consumidor

#### API Mobile (Publicador)
**Ubicación:** `/edugo-api-mobile/internal/infrastructure/messaging/rabbitmq/events.go`

```go
type AssessmentGeneratedPayload struct {
    MaterialID       string `json:"material_id"`
    MongoDocumentID  string `json:"mongo_document_id"`
    QuestionsCount   int    `json:"questions_count"`
    ProcessingTimeMs int    `json:"processing_time_ms,omitempty"`
}
```

**Código Preparado:**
- Método: `PublishAssessmentGenerated` existe en EventPublisher
- Archivo: `/edugo-api-mobile/internal/infrastructure/messaging/rabbitmq/event_publisher.go:49`

#### Worker (Consumidor)
❌ **NO EXISTE CONSUMIDOR**

**Impacto:** 🟡 **MEDIO** - El método existe pero no se usa actualmente. Funcionalidad preparada pero no conectada.

---

### 4. material_deleted ⚠️

**Estado:** SIN PUBLICADOR - Solo consumidor

#### API Mobile (Publicador)
❌ **NO EXISTE PUBLICADOR**

#### Worker (Consumidor)
**Ubicación:** `/edugo-worker/internal/application/dto/event_dto.go`

```go
type MaterialDeletedEvent struct {
    EventType  string    `json:"event_type"`
    MaterialID string    `json:"material_id"`
    Timestamp  time.Time `json:"timestamp"`
}
```

**Consumidor:**
- Event Type: `material_deleted`
- Processor: `MaterialDeletedProcessor`
- Archivo: `/edugo-worker/internal/application/processor/material_deleted_processor.go:28`
- Registrado: ✅ Sí (container.go:76)

**Impacto:** 🟡 **MEDIO** - Worker está preparado para eliminar data en MongoDB, pero nadie envía el evento.

---

### 5. assessment_attempt ⚠️

**Estado:** SIN PUBLICADOR - Solo consumidor

#### API Mobile (Publicador)
❌ **NO EXISTE PUBLICADOR**

#### Worker (Consumidor)
**Ubicación:** `/edugo-worker/internal/application/dto/event_dto.go`

```go
type AssessmentAttemptEvent struct {
    EventType  string                 `json:"event_type"`
    MaterialID string                 `json:"material_id"`
    UserID     string                 `json:"user_id"`
    Answers    map[string]interface{} `json:"answers"`
    Score      float64                `json:"score"`
    Timestamp  time.Time              `json:"timestamp"`
}
```

**Consumidor:**
- Event Type: `assessment_attempt`
- Processor: `AssessmentAttemptProcessor` (stub - solo logging)
- Archivo: `/edugo-worker/internal/application/processor/assessment_attempt_processor.go:20`
- Registrado: ✅ Sí (container.go:77)

**Impacto:** 🟡 **MEDIO** - Processor existe pero no hace nada útil (solo log). Nadie publica el evento.

---

### 6. student_enrolled ⚠️

**Estado:** SIN PUBLICADOR - Solo consumidor

#### API Mobile (Publicador)
❌ **NO EXISTE PUBLICADOR**

#### Worker (Consumidor)
**Ubicación:** `/edugo-worker/internal/application/dto/event_dto.go`

```go
type StudentEnrolledEvent struct {
    EventType string    `json:"event_type"`
    StudentID string    `json:"student_id"`
    UnitID    string    `json:"unit_id"`
    Timestamp time.Time `json:"timestamp"`
}
```

**Consumidor:**
- Event Type: `student_enrolled`
- Processor: `StudentEnrolledProcessor` (stub - solo logging)
- Archivo: `/edugo-worker/internal/application/processor/student_enrolled_processor.go:20`
- Registrado: ✅ Sí (container.go:78)

**Impacto:** 🟡 **MEDIO** - Processor existe pero no hace nada útil. Nadie publica el evento.

---

### 7. material_reprocess ⚠️

**Estado:** SIN PUBLICADOR - Solo consumidor

#### API Mobile (Publicador)
❌ **NO EXISTE PUBLICADOR**

#### Worker (Consumidor)
**Ubicación:** Reutiliza `MaterialUploadedEvent` DTO

```go
// Usa el mismo DTO que material_uploaded
type MaterialUploadedEvent struct {
    EventType         string    `json:"event_type"`
    MaterialID        string    `json:"material_id"`
    AuthorID          string    `json:"author_id"`
    S3Key             string    `json:"s3_key"`
    PreferredLanguage string    `json:"preferred_language"`
    Timestamp         time.Time `json:"timestamp"`
}
```

**Consumidor:**
- Event Type: `material_reprocess`
- Processor: `MaterialReprocessProcessor` (wrapper sobre MaterialUploadedProcessor)
- Archivo: `/edugo-worker/internal/application/processor/material_reprocess_processor.go:32`
- Registrado: ✅ Sí (container.go:75)

**Impacto:** 🟡 **MEDIO** - Funcionalidad útil pero nunca se activa porque nadie publica el evento.

---

## Análisis de edugo-shared

### DTOs Compartidos: ❌ NO EXISTEN

**Búsqueda realizada en:**
- `/edugo-shared/common/types/`
- Todos los subdirectorios de edugo-shared

**Resultado:**
- ❌ NO existe ningún DTO compartido para eventos
- ✅ Solo existe `enum/event.go` con constantes de tipos de evento

### Enums en edugo-shared

**Ubicación:** `/edugo-shared/common/types/enum/event.go`

```go
type EventType string

const (
    EventMaterialUploaded          EventType = "material.uploaded"
    EventMaterialReprocess         EventType = "material.reprocess"
    EventMaterialDeleted           EventType = "material.deleted"
    EventMaterialPublished         EventType = "material.published"
    EventMaterialArchived          EventType = "material.archived"
    EventAssessmentAttemptRecorded EventType = "assessment.attempt_recorded"
    EventAssessmentCompleted       EventType = "assessment.completed"
    EventStudentEnrolled           EventType = "student.enrolled"
    EventStudentProgress           EventType = "student.progress"
    EventUserCreated               EventType = "user.created"
    EventUserUpdated               EventType = "user.updated"
    EventUserDeactivated           EventType = "user.deactivated"
)
```

**Observaciones:**
1. ✅ Define nombres de eventos en formato punto (correcto)
2. ⚠️ NO se usa en Worker (Worker usa strings hardcoded con underscore)
3. ⚠️ NO se usa en API Mobile de forma consistente
4. ❌ NO incluye los DTOs/estructuras de los eventos

---

## Riesgos de Integración

### 🔴 Riesgos CRÍTICOS

1. **Evento `material.uploaded` Roto**
   - Worker NO puede deserializar correctamente los eventos de API Mobile
   - Campos faltantes causan que el procesamiento falle o use valores por defecto
   - `school_id` se pierde completamente
   - `file_url` vs `s3_key` puede causar errores de acceso a archivos

2. **Eventos Huérfanos Desperdicados**
   - `material.completed`: Información valiosa de progreso del usuario se publica pero nunca se procesa
   - `assessment.generated`: Confirmación de generación de quiz se pierde

3. **Falta de Contrato Compartido**
   - Cada proyecto define su propia versión de los eventos
   - Sin source of truth único
   - Alto riesgo de divergencia futura
   - Dificulta mantenimiento y evolución

### 🟡 Riesgos MEDIOS

4. **Consumers sin Publicadores**
   - Código muerto en Worker esperando eventos que nunca llegan
   - Confusión sobre qué funcionalidad está activa
   - Mantenimiento de código innecesario

5. **Inconsistencia de Formato**
   - API usa formato punto: `material.uploaded`
   - Worker usa formato underscore: `material_uploaded`
   - Enum en shared usa formato punto pero Worker no lo usa
   - Confusión en debugging y logs

### 🟢 Puntos Positivos

1. ✅ Worker tiene processor registry bien estructurado
2. ✅ API Mobile usa envelope pattern para eventos
3. ✅ edugo-shared tiene enums de tipos de evento (aunque no se usan consistentemente)
4. ✅ Separation of concerns: cada proyecto puede evolucionar independientemente (pero necesita mejor coordinación)

---

## Recomendaciones de Solución

### Prioridad 1: URGENTE 🔴

1. **Crear DTOs Compartidos en edugo-shared**
   ```
   edugo-shared/
   └── events/
       ├── material.go      (MaterialUploadedEvent, MaterialCompletedEvent, etc.)
       ├── assessment.go    (AssessmentAttemptEvent, AssessmentGeneratedEvent)
       ├── student.go       (StudentEnrolledEvent)
       └── envelope.go      (Event envelope estándar)
   ```

2. **Migrar API Mobile y Worker a usar DTOs compartidos**
   - Mantener compatibilidad temporal con versiones antiguas
   - Usar `event_version` field para manejo de breaking changes

3. **Estandarizar Formato de Event Type**
   - Decidir: punto (`material.uploaded`) o underscore (`material_uploaded`)
   - Recomendación: Usar formato punto (más estándar en event-driven)
   - Usar constantes de `edugo-shared/common/types/enum/event.go`

### Prioridad 2: IMPORTANTE 🟡

4. **Implementar Publicadores Faltantes**
   - `material_deleted` en API Mobile cuando se elimine un material
   - `assessment_attempt` en API Mobile cuando usuario complete quiz
   - `student_enrolled` en API Administración cuando estudiante se inscriba

5. **Implementar Consumers Faltantes**
   - `material.completed` en Worker para analytics/notificaciones
   - `assessment.generated` en Worker para actualizar estado/notificar

6. **Validación de Esquemas**
   - Implementar JSON Schema validation para eventos
   - Tests de contrato entre publicador y consumidor
   - CI/CD checks para detectar breaking changes

### Prioridad 3: MEJORA CONTINUA 🟢

7. **Documentación de Eventos**
   - Crear catálogo de eventos en edugo-shared
   - Documentar payload de cada evento
   - Documentar flujos de publicador → consumidor

8. **Monitoring y Alertas**
   - Dead letter queues para eventos no procesables
   - Métricas de eventos publicados vs consumidos
   - Alertas de eventos huérfanos

9. **Versionado de Eventos**
   - Estrategia de versionado semántico
   - Política de deprecación de eventos
   - Backward compatibility guidelines

---

## Impacto en Frontend

### Riesgos Directos para Frontend:

1. **Material Upload Flow Roto** 🔴
   - Frontend espera que material uploaded dispare procesamiento
   - Si Worker no puede procesar evento, material queda en estado inconsistente
   - Usuario no recibe feedback correcto

2. **Progress Tracking Incompleto** 🔴
   - `material.completed` no se procesa
   - Analytics de completitud pueden estar incorrectos
   - Notificaciones a profesores pueden no funcionar

3. **Assessment Generation Sin Feedback** 🟡
   - `assessment.generated` se publica pero no se procesa
   - Frontend no puede mostrar estado correcto de quiz
   - Polling puede fallar esperando estado que nunca cambia

### Recomendaciones para Frontend:

1. ✅ Implementar polling con timeout razonable (no esperar indefinidamente)
2. ✅ Mostrar estados intermedios claros al usuario
3. ✅ Implementar retry logic en caso de timeouts
4. ⚠️ NO asumir que eventos se procesan instantáneamente
5. ⚠️ Manejar estados de error/inconsistencia con gracia

---

## Conclusión

El estado actual de los contratos entre proyectos presenta **RIESGOS CRÍTICOS** que deben abordarse urgentemente:

- **0 de 7 eventos** tienen contratos compartidos correctos
- **1 de 7 eventos** tiene publicador y consumidor conectados (y con estructura incompatible)
- **3 eventos** se publican sin consumidor
- **4 eventos** tienen consumidor sin publicador

**Próximos Pasos Inmediatos:**

1. ✅ Crear issue de URGENTE para alinear DTOs de `material.uploaded`
2. ✅ Crear módulo `events` en edugo-shared con DTOs compartidos
3. ✅ Migrar API Mobile y Worker a usar DTOs compartidos
4. ✅ Implementar tests de contrato entre proyectos
5. ✅ Documentar eventos y flujos en README de edugo-shared

**Tiempo Estimado de Resolución:** 2-3 sprints para resolver todos los issues críticos.

---

**Documento generado automáticamente por análisis de código.**
**Última actualización:** 2025-12-24
