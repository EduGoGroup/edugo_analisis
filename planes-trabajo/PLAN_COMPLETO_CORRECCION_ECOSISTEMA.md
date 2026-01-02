# PLAN COMPLETO: CORRECCIÓN INTEGRAL ECOSISTEMA EDUGO

**Fecha de Creación:** 2025-12-24
**Versión:** 1.0
**Autor:** Sistema de Análisis Automatizado
**Basado en:** Análisis de 4 informes consolidados

---

## ORDEN DE EJECUCIÓN

**Secuencia de Proyectos (Respetando Dependencias):**

1. **edugo-shared** (Fundación - DTOs compartidos)
2. **edugo-api-administracion** (Autenticación centralizada)
3. **edugo-api-mobile** (Consumidor de servicios)
4. **edugo-worker** (Procesamiento asíncrono)
5. **edugo-infrastructure** (Validación y documentación)

---

## PROYECTO: edugo-shared

### FASE 1: Creación de Módulo de Eventos Compartidos

#### Pre-requisitos
- Go 1.25 instalado
- Repositorio edugo-shared clonado
- Ninguna API debe estar consumiendo aún (se migrará después)

#### Pasos

##### Paso 1.1: Crear estructura de directorios
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-shared/events/`
**Acción:** Crear carpeta

```bash
mkdir -p /Users/jhoanmedina/source/EduGo/repos-separados/edugo-shared/events
```

**Tests requeridos:**
- ✅ Verificar que carpeta existe: `test -d /path/to/events && echo OK`

##### Paso 1.2: Crear envelope estándar de eventos
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-shared/events/envelope.go`
**Líneas:** 1-25 (nuevo archivo)
**Cambio:** Crear estructura base de envelope

```go
package events

import (
	"time"

	"github.com/google/uuid"
)

// Event es el envelope estándar para todos los eventos del ecosistema
type Event struct {
	EventID      string      `json:"event_id"`
	EventType    string      `json:"event_type"`
	EventVersion string      `json:"event_version"`
	Timestamp    time.Time   `json:"timestamp"`
	Payload      interface{} `json:"payload"`
}

// NewEvent crea un nuevo envelope con valores por defecto
func NewEvent(eventType string, payload interface{}) *Event {
	return &Event{
		EventID:      uuid.New().String(),
		EventType:    eventType,
		EventVersion: "1.0.0",
		Timestamp:    time.Now().UTC(),
		Payload:      payload,
	}
}
```

**Tests requeridos:**
- Unit: Validar creación de envelope con UUID único
- Unit: Validar que timestamp es UTC
- Unit: Validar que version default es 1.0.0

##### Paso 1.3: Crear DTOs de eventos de materiales
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-shared/events/material.go`
**Líneas:** 1-80 (nuevo archivo)
**Cambio:** Definir eventos de material

```go
package events

import (
	"time"
)

// MaterialUploadedPayload evento cuando se sube un material
type MaterialUploadedPayload struct {
	MaterialID        string                 `json:"material_id"`
	SchoolID          string                 `json:"school_id"`
	TeacherID         string                 `json:"teacher_id"` // Consistente con API Mobile
	FileURL           string                 `json:"file_url"`
	S3Key             string                 `json:"s3_key"` // Agregado para Worker
	FileSizeBytes     int64                  `json:"file_size_bytes"`
	FileType          string                 `json:"file_type"`
	PreferredLanguage string                 `json:"preferred_language,omitempty"` // Opcional
	Metadata          map[string]interface{} `json:"metadata,omitempty"`
}

// MaterialCompletedPayload evento cuando usuario completa material (100%)
type MaterialCompletedPayload struct {
	MaterialID  string    `json:"material_id"`
	SchoolID    string    `json:"school_id"`
	UserID      string    `json:"user_id"`
	CompletedAt time.Time `json:"completed_at"`
}

// MaterialDeletedPayload evento cuando se elimina un material
type MaterialDeletedPayload struct {
	MaterialID string    `json:"material_id"`
	SchoolID   string    `json:"school_id,omitempty"`
	DeletedBy  string    `json:"deleted_by"`
	DeletedAt  time.Time `json:"deleted_at"`
}

// MaterialReprocessPayload evento para reprocesar material
type MaterialReprocessPayload struct {
	MaterialID        string `json:"material_id"`
	TeacherID         string `json:"teacher_id"`
	S3Key             string `json:"s3_key"`
	PreferredLanguage string `json:"preferred_language,omitempty"`
	Reason            string `json:"reason,omitempty"` // Por qué se reprocesa
}
```

**Tests requeridos:**
- Unit: Validar serialización JSON de cada payload
- Unit: Validar deserialización JSON sin errores
- Unit: Validar campos opcionales (omitempty)

##### Paso 1.4: Crear DTOs de eventos de evaluaciones
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-shared/events/assessment.go`
**Líneas:** 1-50 (nuevo archivo)
**Cambio:** Definir eventos de assessment

```go
package events

import (
	"time"
)

// AssessmentGeneratedPayload evento cuando Worker genera un quiz
type AssessmentGeneratedPayload struct {
	MaterialID       string `json:"material_id"`
	MongoDocumentID  string `json:"mongo_document_id"`
	QuestionsCount   int    `json:"questions_count"`
	ProcessingTimeMs int    `json:"processing_time_ms,omitempty"`
}

// AssessmentAttemptRecordedPayload evento cuando estudiante completa quiz
type AssessmentAttemptRecordedPayload struct {
	MaterialID string                 `json:"material_id"`
	UserID     string                 `json:"user_id"`
	AttemptID  string                 `json:"attempt_id"`
	Score      float64                `json:"score"`
	MaxScore   float64                `json:"max_score"`
	Passed     bool                   `json:"passed"`
	Answers    map[string]interface{} `json:"answers,omitempty"`
	Timestamp  time.Time              `json:"timestamp"`
}
```

**Tests requeridos:**
- Unit: Validar tipos de datos (float64 para scores)
- Unit: Validar serialización de map[string]interface{}

##### Paso 1.5: Crear DTOs de eventos de estudiantes
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-shared/events/student.go`
**Líneas:** 1-20 (nuevo archivo)
**Cambio:** Definir evento de inscripción

```go
package events

import (
	"time"
)

// StudentEnrolledPayload evento cuando estudiante se inscribe a una unidad
type StudentEnrolledPayload struct {
	StudentID string    `json:"student_id"`
	UnitID    string    `json:"unit_id"`
	SchoolID  string    `json:"school_id"`
	Timestamp time.Time `json:"timestamp"`
}
```

**Tests requeridos:**
- Unit: Validar formato timestamp ISO8601

##### Paso 1.6: Estandarizar constantes de tipos de eventos
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-shared/common/types/enum/event.go`
**Líneas:** 1-30 (actualizar existente)
**Cambio:** Asegurar formato punto (material.uploaded) en todas las constantes

```go
package enum

type EventType string

const (
	// Material events - usar formato PUNTO (estándar CloudEvents)
	EventMaterialUploaded          EventType = "material.uploaded"
	EventMaterialReprocess         EventType = "material.reprocess"
	EventMaterialDeleted           EventType = "material.deleted"
	EventMaterialCompleted         EventType = "material.completed"

	// Assessment events
	EventAssessmentGenerated       EventType = "assessment.generated"
	EventAssessmentAttemptRecorded EventType = "assessment.attempt_recorded"

	// Student events
	EventStudentEnrolled           EventType = "student.enrolled"
)
```

**Tests requeridos:**
- Unit: Validar que todas las constantes usan formato punto
- Unit: Validar conversión string(EventType) sin errores

##### Paso 1.7: Crear archivo README de eventos
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-shared/events/README.md`
**Líneas:** 1-100 (nuevo archivo)
**Cambio:** Documentar catálogo de eventos

```markdown
# Catálogo de Eventos - EduGo Platform

## Formato Estándar (CloudEvents)

Todos los eventos usan el envelope definido en `envelope.go`:

```json
{
  "event_id": "uuid-v4",
  "event_type": "material.uploaded",
  "event_version": "1.0.0",
  "timestamp": "2024-12-23T10:00:00Z",
  "payload": { ... }
}
```

## Eventos de Materiales

### material.uploaded
**Publicador:** API Mobile
**Consumidor:** Worker
**Payload:** `MaterialUploadedPayload`

**Flujo:**
1. Docente sube PDF a S3
2. API Mobile publica evento
3. Worker consume y procesa PDF con IA

### material.completed
**Publicador:** API Mobile
**Consumidor:** Worker (futuro: Analytics)
**Payload:** `MaterialCompletedPayload`

**Flujo:**
1. Estudiante alcanza 100% progreso
2. API Mobile publica evento
3. Worker puede enviar notificaciones/analytics

### material.deleted
**Publicador:** API Mobile
**Consumidor:** Worker
**Payload:** `MaterialDeletedPayload`

**Flujo:**
1. Admin elimina material
2. API Mobile publica evento
3. Worker limpia MongoDB

## Versionado

- **Cambios compatibles:** Incrementar PATCH (1.0.0 → 1.0.1)
- **Nuevos campos opcionales:** Incrementar MINOR (1.0.0 → 1.1.0)
- **Breaking changes:** Incrementar MAJOR (1.0.0 → 2.0.0)
```

**Tests requeridos:**
- Manual: Revisar documentación con equipo

#### Tests Requeridos (Fase 1)

| Test | Tipo | Ubicación | Comando |
|------|------|-----------|---------|
| Compilación de módulo events | Unit | `/edugo-shared/events` | `go build ./events` |
| JSON marshalling de todos los payloads | Unit | `/edugo-shared/events/*_test.go` | `go test ./events -v` |
| Validación de constantes EventType | Unit | `/edugo-shared/common/types/enum/event_test.go` | `go test ./common/types/enum -v` |
| Envelope UUID único | Unit | `/edugo-shared/events/envelope_test.go` | `go test ./events -run TestEnvelope` |

#### Configuración TestContainer

No aplica para este proyecto (solo DTOs, sin lógica de negocio).

#### Post-requisitos
- ✅ Módulo `events` compila sin errores
- ✅ Tests unitarios al 100% de cobertura
- ✅ Documentación README.md completa
- ✅ Tag git: `v1.0.0` del módulo events
- ✅ Publicar go module (go mod tidy en raíz)

---

## PROYECTO: edugo-api-administracion

### FASE 2: Implementación RBAC y Configuración CORS

#### Pre-requisitos
- edugo-shared fase 1 completada
- PostgreSQL con tabla `users` (campo `role`)
- JWT middleware existente funcional

#### Pasos

##### Paso 2.1: Configurar CORS en main.go
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion/cmd/main.go`
**Líneas:** ~70-90 (justo después de inicializar router)
**Cambio:** Agregar middleware CORS

```go
// Después de: r := gin.Default()

// Configurar CORS
r.Use(func(c *gin.Context) {
	origin := c.GetHeader("Origin")
	allowedOrigins := strings.Split(os.Getenv("CORS_ALLOWED_ORIGINS"), ",")

	// Validar origin
	for _, allowed := range allowedOrigins {
		if origin == strings.TrimSpace(allowed) {
			c.Writer.Header().Set("Access-Control-Allow-Origin", origin)
			break
		}
	}

	c.Writer.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, PATCH, DELETE, OPTIONS")
	c.Writer.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization, X-Request-ID")
	c.Writer.Header().Set("Access-Control-Expose-Headers", "X-Request-ID")
	c.Writer.Header().Set("Access-Control-Max-Age", "86400")

	if c.Request.Method == "OPTIONS" {
		c.AbortWithStatus(http.StatusNoContent)
		return
	}

	c.Next()
})
```

**Tests requeridos:**
- Integration: Enviar OPTIONS request y validar headers CORS
- Integration: Enviar GET con Origin permitido
- Integration: Enviar GET con Origin NO permitido (debe fallar)

##### Paso 2.2: Crear middleware RBAC
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion/internal/infrastructure/http/middleware/rbac_middleware.go`
**Líneas:** 1-80 (nuevo archivo)
**Cambio:** Implementar autorización por roles

```go
package middleware

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

type RBACMiddleware struct {
	allowedRoles []string
}

func NewRBACMiddleware(roles ...string) *RBACMiddleware {
	return &RBACMiddleware{
		allowedRoles: roles,
	}
}

func (m *RBACMiddleware) Handle() gin.HandlerFunc {
	return func(c *gin.Context) {
		// Obtener role del JWT (inyectado por JWTAuthMiddleware)
		roleInterface, exists := c.Get("user_role")
		if !exists {
			c.JSON(http.StatusUnauthorized, gin.H{
				"error": "unauthorized",
				"message": "No role found in token",
			})
			c.Abort()
			return
		}

		userRole, ok := roleInterface.(string)
		if !ok {
			c.JSON(http.StatusInternalServerError, gin.H{
				"error": "internal_error",
				"message": "Invalid role format",
			})
			c.Abort()
			return
		}

		// Validar si el rol está permitido
		for _, allowed := range m.allowedRoles {
			if userRole == allowed {
				c.Next()
				return
			}
		}

		// Rol no autorizado
		c.JSON(http.StatusForbidden, gin.H{
			"error": "forbidden",
			"message": "Insufficient permissions",
			"required_roles": m.allowedRoles,
		})
		c.Abort()
	}
}

// Helpers predefinidos
func RequireAdmin() gin.HandlerFunc {
	return NewRBACMiddleware("admin", "super_admin").Handle()
}

func RequireTeacherOrAdmin() gin.HandlerFunc {
	return NewRBACMiddleware("teacher", "admin", "super_admin").Handle()
}

func RequireOwnerOrAdmin() gin.HandlerFunc {
	return NewRBACMiddleware("owner", "admin", "super_admin").Handle()
}
```

**Tests requeridos:**
- Unit: Validar rechazo de usuario sin role en contexto
- Unit: Validar aceptación de role permitido
- Unit: Validar rechazo de role no permitido
- Integration: Request con JWT de estudiante a endpoint de admin (403)

##### Paso 2.3: Aplicar RBAC a rutas críticas
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion/internal/infrastructure/http/router/router.go`
**Líneas:** ~50-150 (rutas protegidas)
**Cambio:** Agregar middleware RBAC

```go
// Antes:
v1.POST("/schools", jwtMiddleware.Handle(), h.schoolHandler.Create)

// Después:
v1.POST("/schools",
	jwtMiddleware.Handle(),
	middleware.RequireAdmin(),
	h.schoolHandler.Create,
)

// Aplicar a todas las rutas de:
// - Schools: RequireAdmin()
// - Academic Units: RequireOwnerOrAdmin()
// - Memberships: RequireAdmin()
// - Subjects: RequireTeacherOrAdmin()
// - Guardian Relations: RequireAdmin()
```

**Tests requeridos:**
- Integration: Usuario teacher intenta crear school (403)
- Integration: Usuario admin crea school (201)
- Integration: Usuario owner crea unidad en su escuela (201)

##### Paso 2.4: Mejorar health check con validación PostgreSQL
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion/internal/infrastructure/http/handler/health_handler.go`
**Líneas:** 1-60 (actualizar existente)
**Cambio:** Agregar ping a PostgreSQL

```go
package handler

import (
	"context"
	"database/sql"
	"net/http"
	"time"

	"github.com/gin-gonic/gin"
)

type HealthHandler struct {
	db *sql.DB
}

func NewHealthHandler(db *sql.DB) *HealthHandler {
	return &HealthHandler{db: db}
}

func (h *HealthHandler) Check(c *gin.Context) {
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()

	// Ping a PostgreSQL
	if err := h.db.PingContext(ctx); err != nil {
		c.JSON(http.StatusServiceUnavailable, gin.H{
			"status": "unhealthy",
			"service": "edugo-api-admin",
			"error": "postgres connection failed",
		})
		return
	}

	c.JSON(http.StatusOK, gin.H{
		"status": "healthy",
		"service": "edugo-api-admin",
		"postgres": "connected",
		"timestamp": time.Now().UTC().Format(time.RFC3339),
	})
}
```

**Tests requeridos:**
- Integration: Health check con PostgreSQL up (200)
- Integration: Health check con PostgreSQL down (503)

##### Paso 2.5: Implementar paginación en GET /schools
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion/internal/infrastructure/http/handler/school_handler.go`
**Líneas:** ~50-80 (método List)
**Cambio:** Agregar parámetros limit y offset

```go
func (h *SchoolHandler) List(c *gin.Context) {
	// Parsear parámetros de paginación
	limit := 50 // default
	offset := 0

	if limitStr := c.Query("limit"); limitStr != "" {
		if parsed, err := strconv.Atoi(limitStr); err == nil && parsed > 0 && parsed <= 100 {
			limit = parsed
		}
	}

	if offsetStr := c.Query("offset"); offsetStr != "" {
		if parsed, err := strconv.Atoi(offsetStr); err == nil && parsed >= 0 {
			offset = parsed
		}
	}

	// Llamar a servicio con paginación
	schools, total, err := h.schoolService.List(c.Request.Context(), limit, offset)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}

	c.JSON(http.StatusOK, gin.H{
		"schools": schools,
		"total": total,
		"limit": limit,
		"offset": offset,
	})
}
```

**Tests requeridos:**
- Integration: GET /schools sin params (default limit=50)
- Integration: GET /schools?limit=10&offset=20
- Integration: GET /schools?limit=1000 (debe limitar a 100)

#### Tests Requeridos (Fase 2)

| Test | Tipo | Ubicación | Comando |
|------|------|-----------|---------|
| CORS preflight | Integration | `/tests/integration/cors_test.go` | `go test ./tests/integration -run TestCORS` |
| RBAC admin-only endpoint | Integration | `/tests/integration/rbac_test.go` | `go test ./tests/integration -run TestRBAC` |
| Health check PostgreSQL | Integration | `/tests/integration/health_test.go` | `go test ./tests/integration -run TestHealth` |
| Paginación schools | Integration | `/tests/integration/school_test.go` | `go test ./tests/integration -run TestSchoolPagination` |

#### Configuración TestContainer

```go
// tests/integration/setup_test.go
package integration_test

import (
	"context"
	"testing"

	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/wait"
)

func SetupPostgresContainer(t *testing.T) testcontainers.Container {
	ctx := context.Background()

	req := testcontainers.ContainerRequest{
		Image:        "postgres:16-alpine",
		ExposedPorts: []string{"5432/tcp"},
		Env: map[string]string{
			"POSTGRES_USER":     "test",
			"POSTGRES_PASSWORD": "test",
			"POSTGRES_DB":       "edugo_test",
		},
		WaitingFor: wait.ForListeningPort("5432/tcp"),
	}

	container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: req,
		Started:          true,
	})
	if err != nil {
		t.Fatalf("Failed to start container: %s", err)
	}

	return container
}
```

#### Post-requisitos
- ✅ CORS configurado y testeado con frontend
- ✅ RBAC aplicado a todas las rutas críticas
- ✅ Health check valida PostgreSQL
- ✅ Paginación implementada en listas
- ✅ Tests de integración al 80%

---

### FASE 3: Migrar a DTOs Compartidos y Publisher de student.enrolled

#### Pre-requisitos
- edugo-shared v1.0.0 disponible
- Fase 2 completada

#### Pasos

##### Paso 3.1: Agregar dependencia de edugo-shared
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion/go.mod`
**Líneas:** ~10 (dependencies)
**Cambio:** Agregar módulo

```go
require (
	github.com/edugo/shared v1.0.0
	// ... otras dependencias
)
```

**Tests requeridos:**
- Unit: `go mod tidy && go build ./...`

##### Paso 3.2: Crear event publisher para student.enrolled
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion/internal/infrastructure/messaging/rabbitmq/event_publisher.go`
**Líneas:** 1-100 (nuevo archivo)
**Cambio:** Implementar publisher RabbitMQ

```go
package rabbitmq

import (
	"context"
	"encoding/json"
	"time"

	"github.com/edugo/shared/events"
	sharedEnum "github.com/edugo/shared/common/types/enum"
	amqp "github.com/rabbitmq/amqp091-go"
)

type EventPublisher struct {
	channel *amqp.Channel
}

func NewEventPublisher(channel *amqp.Channel) *EventPublisher {
	return &EventPublisher{channel: channel}
}

func (p *EventPublisher) PublishStudentEnrolled(
	ctx context.Context,
	studentID, unitID, schoolID string,
) error {
	payload := events.StudentEnrolledPayload{
		StudentID: studentID,
		UnitID:    unitID,
		SchoolID:  schoolID,
		Timestamp: time.Now().UTC(),
	}

	envelope := events.NewEvent(string(sharedEnum.EventStudentEnrolled), payload)

	body, err := json.Marshal(envelope)
	if err != nil {
		return err
	}

	return p.channel.PublishWithContext(
		ctx,
		"edugo.events", // exchange
		string(sharedEnum.EventStudentEnrolled), // routing key
		false, // mandatory
		false, // immediate
		amqp.Publishing{
			ContentType:  "application/json",
			Body:         body,
			DeliveryMode: amqp.Persistent,
			Timestamp:    time.Now(),
		},
	)
}
```

**Tests requeridos:**
- Integration: Publicar evento y consumir desde cola temporal
- Integration: Validar formato JSON del mensaje

##### Paso 3.3: Integrar publisher en MembershipService
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion/internal/application/service/membership_service.go`
**Líneas:** ~50 (método Create, después de guardar en DB)
**Cambio:** Publicar evento cuando role=student

```go
// Después de: membership, err := s.repo.Create(ctx, newMembership)

// Si es estudiante, publicar evento
if membership.Role == "student" {
	if err := s.eventPublisher.PublishStudentEnrolled(
		ctx,
		membership.UserID,
		membership.UnitID,
		membership.SchoolID, // Obtener del contexto o unit
	); err != nil {
		// Log error pero no fallar (evento es best-effort)
		s.logger.Error("Failed to publish student enrolled event", "error", err)
	}
}
```

**Tests requeridos:**
- Integration: Crear membership con role=student y verificar evento en RabbitMQ
- Integration: Crear membership con role=teacher y verificar que NO se publica evento

#### Tests Requeridos (Fase 3)

| Test | Tipo | Ubicación | Comando |
|------|------|-----------|---------|
| Importar módulo shared | Unit | Raíz | `go build ./...` |
| Publicar student.enrolled | Integration | `/tests/integration/events_test.go` | `go test ./tests/integration -run TestStudentEnrolled` |
| Validar formato envelope | Integration | `/tests/integration/events_test.go` | `go test ./tests/integration -run TestEventEnvelope` |

#### Configuración TestContainer

```go
// Agregar RabbitMQ a setup_test.go
func SetupRabbitMQContainer(t *testing.T) testcontainers.Container {
	ctx := context.Background()

	req := testcontainers.ContainerRequest{
		Image:        "rabbitmq:3-management-alpine",
		ExposedPorts: []string{"5672/tcp", "15672/tcp"},
		WaitingFor:   wait.ForListeningPort("5672/tcp"),
	}

	container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: req,
		Started:          true,
	})
	if err != nil {
		t.Fatalf("Failed to start RabbitMQ: %s", err)
	}

	return container
}
```

#### Post-requisitos
- ✅ Eventos student.enrolled se publican correctamente
- ✅ Worker puede consumir eventos (validar en siguiente proyecto)
- ✅ Tests de integración con RabbitMQ al 70%

---

## PROYECTO: edugo-api-mobile

### FASE 4: Migrar a DTOs Compartidos y Publishers de Eventos Faltantes

#### Pre-requisitos
- edugo-shared v1.0.0 disponible
- RabbitMQ configurado

#### Pasos

##### Paso 4.1: Agregar dependencia de edugo-shared
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile/go.mod`
**Líneas:** ~10
**Cambio:** Agregar módulo

```go
require (
	github.com/edugo/shared v1.0.0
	// ... otras
)
```

##### Paso 4.2: Reemplazar DTOs locales por compartidos en event_publisher.go
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile/internal/infrastructure/messaging/rabbitmq/event_publisher.go`
**Líneas:** 1-200 (actualizar existente)
**Cambio:** Importar y usar DTOs de edugo-shared

```go
package rabbitmq

import (
	"context"
	"encoding/json"
	"time"

	"github.com/edugo/shared/events"
	sharedEnum "github.com/edugo/shared/common/types/enum"
	amqp "github.com/rabbitmq/amqp091-go"
)

type EventPublisher struct {
	channel *amqp.Channel
}

func NewEventPublisher(channel *amqp.Channel) *EventPublisher {
	return &EventPublisher{channel: channel}
}

// PublishMaterialUploaded usa DTOs compartidos
func (p *EventPublisher) PublishMaterialUploaded(
	ctx context.Context,
	materialID, schoolID, teacherID, fileURL, s3Key, fileType string,
	fileSizeBytes int64,
	metadata map[string]interface{},
) error {
	payload := events.MaterialUploadedPayload{
		MaterialID:    materialID,
		SchoolID:      schoolID,
		TeacherID:     teacherID,
		FileURL:       fileURL,
		S3Key:         s3Key, // NUEVO: Agregar S3Key para Worker
		FileSizeBytes: fileSizeBytes,
		FileType:      fileType,
		Metadata:      metadata,
	}

	envelope := events.NewEvent(string(sharedEnum.EventMaterialUploaded), payload)

	body, err := json.Marshal(envelope)
	if err != nil {
		return err
	}

	return p.channel.PublishWithContext(
		ctx,
		"edugo.materials",
		string(sharedEnum.EventMaterialUploaded),
		false,
		false,
		amqp.Publishing{
			ContentType:  "application/json",
			Body:         body,
			DeliveryMode: amqp.Persistent,
			Timestamp:    time.Now(),
		},
	)
}

// PublishMaterialDeleted (NUEVO)
func (p *EventPublisher) PublishMaterialDeleted(
	ctx context.Context,
	materialID, schoolID, deletedBy string,
) error {
	payload := events.MaterialDeletedPayload{
		MaterialID: materialID,
		SchoolID:   schoolID,
		DeletedBy:  deletedBy,
		DeletedAt:  time.Now().UTC(),
	}

	envelope := events.NewEvent(string(sharedEnum.EventMaterialDeleted), payload)

	body, err := json.Marshal(envelope)
	if err != nil {
		return err
	}

	return p.channel.PublishWithContext(
		ctx,
		"edugo.materials",
		string(sharedEnum.EventMaterialDeleted),
		false,
		false,
		amqp.Publishing{
			ContentType:  "application/json",
			Body:         body,
			DeliveryMode: amqp.Persistent,
		},
	)
}

// PublishAssessmentAttemptRecorded (NUEVO)
func (p *EventPublisher) PublishAssessmentAttemptRecorded(
	ctx context.Context,
	materialID, userID, attemptID string,
	score, maxScore float64,
	passed bool,
	answers map[string]interface{},
) error {
	payload := events.AssessmentAttemptRecordedPayload{
		MaterialID: materialID,
		UserID:     userID,
		AttemptID:  attemptID,
		Score:      score,
		MaxScore:   maxScore,
		Passed:     passed,
		Answers:    answers,
		Timestamp:  time.Now().UTC(),
	}

	envelope := events.NewEvent(string(sharedEnum.EventAssessmentAttemptRecorded), payload)

	body, err := json.Marshal(envelope)
	if err != nil {
		return err
	}

	return p.channel.PublishWithContext(
		ctx,
		"edugo.assessments",
		string(sharedEnum.EventAssessmentAttemptRecorded),
		false,
		false,
		amqp.Publishing{
			ContentType:  "application/json",
			Body:         body,
			DeliveryMode: amqp.Persistent,
		},
	)
}
```

**Tests requeridos:**
- Integration: Publicar material.uploaded y validar estructura con Worker
- Integration: Publicar material.deleted y validar consumo
- Integration: Publicar assessment.attempt_recorded

##### Paso 4.3: Integrar publisher material.deleted en MaterialService
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile/internal/application/service/material_service.go`
**Líneas:** ~200 (método Delete)
**Cambio:** Publicar evento después de soft delete

```go
// En método Delete, después de marcar como deleted en DB
func (s *MaterialService) Delete(ctx context.Context, materialID, userID string) error {
	// ... lógica existente de soft delete ...

	// Publicar evento para que Worker limpie MongoDB
	if err := s.eventPublisher.PublishMaterialDeleted(
		ctx,
		materialID,
		material.SchoolID,
		userID, // quien eliminó
	); err != nil {
		s.logger.Error("Failed to publish material.deleted", "error", err)
		// No fallar, evento es best-effort
	}

	return nil
}
```

**Tests requeridos:**
- Integration: Eliminar material y verificar evento en RabbitMQ
- Integration: Verificar que Worker limpia colecciones MongoDB

##### Paso 4.4: Integrar publisher assessment.attempt_recorded en AssessmentService
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile/internal/application/service/assessment_service.go`
**Líneas:** ~150 (método CreateAttempt)
**Cambio:** Publicar evento después de guardar intento

```go
// En método CreateAttempt, después de guardar en PostgreSQL
func (s *AssessmentService) CreateAttempt(ctx context.Context, req CreateAttemptRequest) (*AttemptResult, error) {
	// ... lógica existente ...

	// Publicar evento para analytics/notificaciones
	if err := s.eventPublisher.PublishAssessmentAttemptRecorded(
		ctx,
		req.MaterialID,
		req.UserID,
		attempt.ID,
		attempt.Score,
		assessment.MaxScore,
		attempt.Passed,
		answersMap, // convertir answers a map
	); err != nil {
		s.logger.Error("Failed to publish assessment.attempt_recorded", "error", err)
	}

	return result, nil
}
```

**Tests requeridos:**
- Integration: Crear intento y verificar evento
- Integration: Validar payload con todos los campos

##### Paso 4.5: Implementar paginación en GET /materials
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile/internal/infrastructure/http/handler/material_handler.go`
**Líneas:** ~40 (método List)
**Cambio:** Similar a API Admin

```go
func (h *MaterialHandler) List(c *gin.Context) {
	limit := 50
	offset := 0

	if limitStr := c.Query("limit"); limitStr != "" {
		if parsed, err := strconv.Atoi(limitStr); err == nil && parsed > 0 && parsed <= 100 {
			limit = parsed
		}
	}

	if offsetStr := c.Query("offset"); offsetStr != "" {
		if parsed, err := strconv.Atoi(offsetStr); err == nil && parsed >= 0 {
			offset = parsed
		}
	}

	materials, total, err := h.materialService.List(c.Request.Context(), limit, offset)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}

	c.JSON(http.StatusOK, gin.H{
		"materials": materials,
		"total": total,
		"limit": limit,
		"offset": offset,
	})
}
```

**Tests requeridos:**
- Integration: GET /materials con paginación

##### Paso 4.6: Corregir GAP CRÍTICO - Tabla progress vs material_progress
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile/internal/infrastructure/persistence/postgres/repository/progress_repository_impl.go`
**Líneas:** Todas las queries (buscar "material_progress")
**Cambio:** Reemplazar por "progress"

```go
// ANTES:
const insertQuery = `
	INSERT INTO material_progress (user_id, material_id, progress_percentage, last_page, updated_at)
	...
`

// DESPUÉS:
const insertQuery = `
	INSERT INTO progress (user_id, material_id, progress_percentage, last_page, updated_at)
	...
`

// Aplicar a TODAS las queries:
// - INSERT INTO progress
// - UPDATE progress
// - SELECT FROM progress
```

**Tests requeridos:**
- Integration: UPSERT progreso y validar que se guarda en tabla "progress"
- Integration: Verificar que query anterior falla (para confirmar fix)

#### Tests Requeridos (Fase 4)

| Test | Tipo | Ubicación | Comando |
|------|------|-----------|---------|
| Migración DTOs compartidos | Unit | Raíz | `go build ./...` |
| Publisher material.deleted | Integration | `/tests/integration/events_test.go` | `go test ./tests/integration -run TestMaterialDeleted` |
| Publisher assessment.attempt_recorded | Integration | `/tests/integration/events_test.go` | `go test ./tests/integration -run TestAssessmentAttempt` |
| Fix tabla progress | Integration | `/tests/integration/progress_test.go` | `go test ./tests/integration -run TestProgressUpsert` |
| Paginación materials | Integration | `/tests/integration/material_test.go` | `go test ./tests/integration -run TestMaterialPagination` |

#### Configuración TestContainer

```go
// Similar a API Admin, agregar RabbitMQ y PostgreSQL
func SetupTestEnvironment(t *testing.T) (*sql.DB, *amqp.Connection) {
	postgresContainer := SetupPostgresContainer(t)
	rabbitContainer := SetupRabbitMQContainer(t)

	// Conectar a PostgreSQL
	dbURL := getContainerConnectionString(postgresContainer)
	db, err := sql.Open("postgres", dbURL)
	if err != nil {
		t.Fatal(err)
	}

	// Conectar a RabbitMQ
	rabbitURL := getContainerConnectionString(rabbitContainer)
	conn, err := amqp.Dial(rabbitURL)
	if err != nil {
		t.Fatal(err)
	}

	return db, conn
}
```

#### Post-requisitos
- ✅ DTOs compartidos integrados
- ✅ Eventos material.deleted y assessment.attempt_recorded publicándose
- ✅ GAP tabla progress corregido
- ✅ Paginación implementada
- ✅ Tests de integración al 80%

---

## PROYECTO: edugo-worker

### FASE 5: Migrar a DTOs Compartidos y Completar Processors Stub

#### Pre-requisitos
- edugo-shared v1.0.0 disponible
- API Mobile publicando eventos con DTOs compartidos

#### Pasos

##### Paso 5.1: Agregar dependencia de edugo-shared
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/go.mod`
**Líneas:** ~10
**Cambio:** Agregar módulo

```go
require (
	github.com/edugo/shared v1.0.0
)
```

##### Paso 5.2: Actualizar MaterialUploadedProcessor para usar DTOs compartidos
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/internal/application/processor/material_uploaded_processor.go`
**Líneas:** 1-400 (actualizar)
**Cambio:** Importar shared events y deserializar envelope

```go
package processor

import (
	"context"
	"encoding/json"

	"github.com/edugo/shared/events"
	sharedEnum "github.com/edugo/shared/common/types/enum"
)

type MaterialUploadedProcessor struct {
	s3Client      S3Client
	pdfExtractor  PDFExtractor
	nlpService    NLPService
	mongoRepo     MongoRepository
	postgresRepo  PostgresRepository
}

func (p *MaterialUploadedProcessor) Process(ctx context.Context, message []byte) error {
	// Deserializar ENVELOPE primero
	var envelope events.Event
	if err := json.Unmarshal(message, &envelope); err != nil {
		return fmt.Errorf("failed to unmarshal envelope: %w", err)
	}

	// Validar event type
	if envelope.EventType != string(sharedEnum.EventMaterialUploaded) {
		return fmt.Errorf("unexpected event type: %s", envelope.EventType)
	}

	// Deserializar payload
	payloadBytes, err := json.Marshal(envelope.Payload)
	if err != nil {
		return fmt.Errorf("failed to marshal payload: %w", err)
	}

	var payload events.MaterialUploadedPayload
	if err := json.Unmarshal(payloadBytes, &payload); err != nil {
		return fmt.Errorf("failed to unmarshal payload: %w", err)
	}

	// Usar S3Key del payload compartido
	pdfBytes, err := p.s3Client.Download(ctx, payload.S3Key)
	if err != nil {
		return fmt.Errorf("failed to download from S3: %w", err)
	}

	// ... resto de lógica existente (extracción PDF, NLP, guardar MongoDB)

	return nil
}
```

**Tests requeridos:**
- Integration: Enviar evento con envelope y validar deserialización
- Integration: Procesar PDF completo y validar MongoDB

##### Paso 5.3: Actualizar MaterialDeletedProcessor
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/internal/application/processor/material_deleted_processor.go`
**Líneas:** 1-100 (actualizar)
**Cambio:** Usar DTOs compartidos

```go
package processor

import (
	"context"
	"encoding/json"

	"github.com/edugo/shared/events"
	sharedEnum "github.com/edugo/shared/common/types/enum"
)

type MaterialDeletedProcessor struct {
	mongoRepo MongoRepository
}

func (p *MaterialDeletedProcessor) Process(ctx context.Context, message []byte) error {
	var envelope events.Event
	if err := json.Unmarshal(message, &envelope); err != nil {
		return err
	}

	if envelope.EventType != string(sharedEnum.EventMaterialDeleted) {
		return fmt.Errorf("unexpected event type: %s", envelope.EventType)
	}

	payloadBytes, _ := json.Marshal(envelope.Payload)
	var payload events.MaterialDeletedPayload
	if err := json.Unmarshal(payloadBytes, &payload); err != nil {
		return err
	}

	// Eliminar de MongoDB
	if err := p.mongoRepo.DeleteMaterialAssessment(ctx, payload.MaterialID); err != nil {
		return err
	}

	if err := p.mongoRepo.DeleteMaterialSummary(ctx, payload.MaterialID); err != nil {
		return err
	}

	return nil
}
```

**Tests requeridos:**
- Integration: Eliminar material y verificar limpieza MongoDB

##### Paso 5.4: Completar AssessmentAttemptProcessor (actualmente stub)
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/internal/application/processor/assessment_attempt_processor.go`
**Líneas:** 1-150 (reemplazar stub)
**Cambio:** Implementar lógica de notificaciones

```go
package processor

import (
	"context"
	"encoding/json"
	"fmt"

	"github.com/edugo/shared/events"
	sharedEnum "github.com/edugo/shared/common/types/enum"
)

type AssessmentAttemptProcessor struct {
	notificationService NotificationService
	analyticsRepo       AnalyticsRepository
}

func NewAssessmentAttemptProcessor(
	notificationService NotificationService,
	analyticsRepo AnalyticsRepository,
) *AssessmentAttemptProcessor {
	return &AssessmentAttemptProcessor{
		notificationService: notificationService,
		analyticsRepo:       analyticsRepo,
	}
}

func (p *AssessmentAttemptProcessor) Process(ctx context.Context, message []byte) error {
	var envelope events.Event
	if err := json.Unmarshal(message, &envelope); err != nil {
		return err
	}

	if envelope.EventType != string(sharedEnum.EventAssessmentAttemptRecorded) {
		return fmt.Errorf("unexpected event type: %s", envelope.EventType)
	}

	payloadBytes, _ := json.Marshal(envelope.Payload)
	var payload events.AssessmentAttemptRecordedPayload
	if err := json.Unmarshal(payloadBytes, &payload); err != nil {
		return err
	}

	// Guardar en analytics
	if err := p.analyticsRepo.RecordAttempt(ctx, payload); err != nil {
		// Log pero no fallar
		fmt.Printf("Failed to record analytics: %v\n", err)
	}

	// Enviar notificación si aprobó
	if payload.Passed {
		message := fmt.Sprintf(
			"¡Felicitaciones! Has aprobado con %.1f/%.1f puntos",
			payload.Score,
			payload.MaxScore,
		)
		if err := p.notificationService.SendToUser(ctx, payload.UserID, message); err != nil {
			// Log pero no fallar
			fmt.Printf("Failed to send notification: %v\n", err)
		}
	}

	return nil
}
```

**Tests requeridos:**
- Integration: Procesar intento aprobado y verificar notificación
- Integration: Procesar intento reprobado sin notificación

##### Paso 5.5: Completar StudentEnrolledProcessor (actualmente stub)
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/internal/application/processor/student_enrolled_processor.go`
**Líneas:** 1-100 (reemplazar stub)
**Cambio:** Implementar lógica de bienvenida

```go
package processor

import (
	"context"
	"encoding/json"
	"fmt"

	"github.com/edugo/shared/events"
	sharedEnum "github.com/edugo/shared/common/types/enum"
)

type StudentEnrolledProcessor struct {
	notificationService NotificationService
	emailService        EmailService
}

func NewStudentEnrolledProcessor(
	notificationService NotificationService,
	emailService EmailService,
) *StudentEnrolledProcessor {
	return &StudentEnrolledProcessor{
		notificationService: notificationService,
		emailService:        emailService,
	}
}

func (p *StudentEnrolledProcessor) Process(ctx context.Context, message []byte) error {
	var envelope events.Event
	if err := json.Unmarshal(message, &envelope); err != nil {
		return err
	}

	if envelope.EventType != string(sharedEnum.EventStudentEnrolled) {
		return fmt.Errorf("unexpected event type: %s", envelope.EventType)
	}

	payloadBytes, _ := json.Marshal(envelope.Payload)
	var payload events.StudentEnrolledPayload
	if err := json.Unmarshal(payloadBytes, &payload); err != nil {
		return err
	}

	// Enviar notificación de bienvenida
	welcomeMessage := fmt.Sprintf(
		"¡Bienvenido a la unidad académica! Comienza a explorar tus materiales.",
	)
	if err := p.notificationService.SendToUser(ctx, payload.StudentID, welcomeMessage); err != nil {
		fmt.Printf("Failed to send welcome notification: %v\n", err)
	}

	// Opcional: Enviar email de bienvenida
	// ...

	return nil
}
```

**Tests requeridos:**
- Integration: Procesar inscripción y verificar notificación

##### Paso 5.6: Implementar consumer para material.completed
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/internal/application/processor/material_completed_processor.go`
**Líneas:** 1-120 (nuevo archivo)
**Cambio:** Crear processor para analytics

```go
package processor

import (
	"context"
	"encoding/json"
	"fmt"

	"github.com/edugo/shared/events"
	sharedEnum "github.com/edugo/shared/common/types/enum"
)

type MaterialCompletedProcessor struct {
	analyticsRepo       AnalyticsRepository
	notificationService NotificationService
}

func NewMaterialCompletedProcessor(
	analyticsRepo AnalyticsRepository,
	notificationService NotificationService,
) *MaterialCompletedProcessor {
	return &MaterialCompletedProcessor{
		analyticsRepo:       analyticsRepo,
		notificationService: notificationService,
	}
}

func (p *MaterialCompletedProcessor) Process(ctx context.Context, message []byte) error {
	var envelope events.Event
	if err := json.Unmarshal(message, &envelope); err != nil {
		return err
	}

	if envelope.EventType != string(sharedEnum.EventMaterialCompleted) {
		return fmt.Errorf("unexpected event type: %s", envelope.EventType)
	}

	payloadBytes, _ := json.Marshal(envelope.Payload)
	var payload events.MaterialCompletedPayload
	if err := json.Unmarshal(payloadBytes, &payload); err != nil {
		return err
	}

	// Registrar en analytics
	if err := p.analyticsRepo.RecordCompletion(ctx, payload); err != nil {
		fmt.Printf("Failed to record completion: %v\n", err)
	}

	// Notificar al profesor (opcional)
	message := fmt.Sprintf(
		"Estudiante completó material: %s",
		payload.MaterialID,
	)
	// Buscar profesor del material y notificar
	// ...

	return nil
}
```

**Tests requeridos:**
- Integration: Procesar completitud y verificar analytics

##### Paso 5.7: Registrar nuevos processors en container
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/internal/infrastructure/container/container.go`
**Líneas:** ~70-80
**Cambio:** Agregar nuevo processor

```go
// Después de registrar processors existentes

// Material Completed
materialCompletedProc := processor.NewMaterialCompletedProcessor(
	c.analyticsRepo,
	c.notificationService,
)
c.ProcessorRegistry.Register(
	string(sharedEnum.EventMaterialCompleted),
	materialCompletedProc,
)
```

**Tests requeridos:**
- Integration: Verificar que registry contiene todos los processors

##### Paso 5.8: Activar integración OpenAI (opcional pero recomendado)
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/internal/infrastructure/nlp/openai_client.go`
**Líneas:** ~50 (inicialización)
**Cambio:** Usar API key real si está disponible

```go
func NewOpenAIClient() NLPService {
	apiKey := os.Getenv("OPENAI_API_KEY")

	if apiKey == "" {
		// Log warning y usar fallback
		log.Warn("OPENAI_API_KEY not set, using fallback NLP")
		return NewFallbackNLPService()
	}

	client := openai.NewClient(apiKey)

	return &OpenAINLPService{
		client: client,
		model:  "gpt-4-turbo-preview", // o "gpt-3.5-turbo" para reducir costos
	}
}
```

**Tests requeridos:**
- Manual: Procesar PDF con OpenAI y validar calidad del resumen
- Manual: Comparar fallback vs OpenAI

#### Tests Requeridos (Fase 5)

| Test | Tipo | Ubicación | Comando |
|------|------|-----------|---------|
| Deserialización envelope | Unit | `/tests/unit/processor_test.go` | `go test ./tests/unit -run TestEnvelopeDeserialization` |
| MaterialUploadedProcessor con DTOs compartidos | Integration | `/tests/integration/processor_test.go` | `go test ./tests/integration -run TestMaterialUploaded` |
| MaterialDeletedProcessor limpia MongoDB | Integration | `/tests/integration/processor_test.go` | `go test ./tests/integration -run TestMaterialDeleted` |
| AssessmentAttemptProcessor notificaciones | Integration | `/tests/integration/processor_test.go` | `go test ./tests/integration -run TestAssessmentAttempt` |
| StudentEnrolledProcessor bienvenida | Integration | `/tests/integration/processor_test.go` | `go test ./tests/integration -run TestStudentEnrolled` |
| MaterialCompletedProcessor analytics | Integration | `/tests/integration/processor_test.go` | `go test ./tests/integration -run TestMaterialCompleted` |

#### Configuración TestContainer

```go
// Setup completo con PostgreSQL, MongoDB y RabbitMQ
func SetupWorkerTestEnvironment(t *testing.T) (
	*sql.DB,
	*mongo.Database,
	*amqp.Connection,
) {
	ctx := context.Background()

	// PostgreSQL
	pgContainer := SetupPostgresContainer(t)
	dbURL := getConnectionString(pgContainer)
	db, _ := sql.Open("postgres", dbURL)

	// MongoDB
	mongoContainer := testcontainers.ContainerRequest{
		Image:        "mongo:7-alpine",
		ExposedPorts: []string{"27017/tcp"},
		WaitingFor:   wait.ForListeningPort("27017/tcp"),
	}
	mongoC, _ := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: mongoContainer,
		Started:          true,
	})
	mongoURL := getConnectionString(mongoC)
	mongoClient, _ := mongo.Connect(ctx, options.Client().ApplyURI(mongoURL))

	// RabbitMQ
	rabbitContainer := SetupRabbitMQContainer(t)
	rabbitURL := getConnectionString(rabbitContainer)
	rabbitConn, _ := amqp.Dial(rabbitURL)

	return db, mongoClient.Database("edugo_test"), rabbitConn
}
```

#### Post-requisitos
- ✅ DTOs compartidos integrados en todos los processors
- ✅ Processors stub completados
- ✅ Consumer para material.completed registrado
- ✅ OpenAI activado (o fallback funcional)
- ✅ Tests de integración al 80%

---

## PROYECTO: edugo-infrastructure

### FASE 6: Validación y Documentación de Esquemas

#### Pre-requisitos
- Todos los proyectos anteriores completados
- PostgreSQL y MongoDB desplegados

#### Pasos

##### Paso 6.1: Validar todas las migraciones up/down
**Archivo:** N/A
**Acción:** Ejecutar script de validación

```bash
#!/bin/bash
# validate_migrations.sh

cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure

# Ejecutar migraciones up
migrate -path postgres/migrations -database "postgres://test:test@localhost:5432/edugo_test?sslmode=disable" up

# Verificar que todas las tablas existen
psql -U test -d edugo_test -c "\dt" | grep -E "(users|schools|materials|progress)" || exit 1

# Ejecutar migraciones down
migrate -path postgres/migrations -database "postgres://test:test@localhost:5432/edugo_test?sslmode=disable" down

# Verificar que no quedan tablas
psql -U test -d edugo_test -c "\dt" | grep -q "users" && exit 1 || echo "OK"

echo "✅ Migraciones validadas correctamente"
```

**Tests requeridos:**
- Integration: Migrar up y validar 20 tablas
- Integration: Migrar down y validar 0 tablas

##### Paso 6.2: Documentar esquema MongoDB
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/SCHEMA.md`
**Líneas:** 1-200 (nuevo archivo)
**Cambio:** Documentar colecciones

```markdown
# Esquema MongoDB - EduGo Platform

## Colecciones

### material_assessment_worker
**Propósito:** Almacenar quizzes generados por IA

**Campos:**
- `_id`: ObjectId
- `material_id`: UUID (string) - FK a PostgreSQL materials.id
- `title`: string
- `questions`: array
  - `id`: UUID (string)
  - `text`: string
  - `type`: "multiple_choice"
  - `options`: array
    - `id`: UUID (string)
    - `text`: string
    - `is_correct`: boolean
  - `explanation`: string (opcional)
- `questions_count`: int
- `pass_threshold`: float (0-100)
- `time_limit_minutes`: int
- `created_at`: ISODate
- `updated_at`: ISODate

**Índices:**
```javascript
db.material_assessment_worker.createIndex({ material_id: 1 }, { unique: true })
db.material_assessment_worker.createIndex({ created_at: -1 })
```

### material_summary
**Propósito:** Almacenar resúmenes generados por IA

**Campos:**
- `_id`: ObjectId
- `material_id`: UUID (string) - FK a PostgreSQL materials.id
- `summary_text`: string
- `key_points`: array of strings
- `model_version`: string (ej: "gpt-4-turbo")
- `generated_at`: ISODate
- `metadata`: object (flexible)

**Índices:**
```javascript
db.material_summary.createIndex({ material_id: 1 }, { unique: true })
```

### audit_logs
**Propósito:** Auditoría de acciones del sistema

**Campos:**
- `_id`: ObjectId
- `user_id`: UUID (string)
- `action`: string (ej: "material.uploaded")
- `resource_type`: string (ej: "material")
- `resource_id`: UUID (string)
- `timestamp`: ISODate
- `ip_address`: string
- `user_agent`: string
- `metadata`: object

**Índices:**
```javascript
db.audit_logs.createIndex({ user_id: 1, timestamp: -1 })
db.audit_logs.createIndex({ resource_id: 1 })
db.audit_logs.createIndex({ action: 1, timestamp: -1 })
```

## Scripts de Inicialización

Ver `/mongodb/init/` para scripts de creación de índices.
```

**Tests requeridos:**
- Manual: Revisión de documentación con equipo

##### Paso 6.3: Crear scripts de inicialización MongoDB
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/mongodb/init/01_create_indexes.js`
**Líneas:** 1-50 (nuevo archivo)
**Cambio:** Script mongosh

```javascript
// 01_create_indexes.js
// Ejecutar con: mongosh < 01_create_indexes.js

db = db.getSiblingDB('edugo');

// Material Assessment Worker
db.material_assessment_worker.createIndex(
  { material_id: 1 },
  { unique: true }
);
db.material_assessment_worker.createIndex(
  { created_at: -1 }
);

// Material Summary
db.material_summary.createIndex(
  { material_id: 1 },
  { unique: true }
);

// Audit Logs
db.audit_logs.createIndex(
  { user_id: 1, timestamp: -1 }
);
db.audit_logs.createIndex(
  { resource_id: 1 }
);
db.audit_logs.createIndex(
  { action: 1, timestamp: -1 }
);

// Analytics Events
db.analytics_events.createIndex(
  { event_type: 1, timestamp: -1 }
);
db.analytics_events.createIndex(
  { user_id: 1, timestamp: -1 }
);

// Notifications
db.notifications.createIndex(
  { user_id: 1, read_at: 1 }
);
db.notifications.createIndex(
  { created_at: -1 }
);

print('✅ Índices creados correctamente');
```

**Tests requeridos:**
- Integration: Ejecutar script y validar índices con `db.collection.getIndexes()`

##### Paso 6.4: Validar performance de ltree en academic_units
**Archivo:** N/A
**Acción:** Script de benchmark

```sql
-- benchmark_ltree.sql

-- Insertar 1000 nodos jerárquicos
DO $$
DECLARE
  i INT;
  parent_path ltree;
BEGIN
  FOR i IN 1..1000 LOOP
    IF i % 10 = 0 THEN
      parent_path := ('root.' || (i/10)::text)::ltree;
    ELSE
      parent_path := ('root.' || (i/10)::text || '.' || i::text)::ltree;
    END IF;

    INSERT INTO academic_units (school_id, type, display_name, path)
    VALUES (
      'school-uuid-here',
      'grade',
      'Test Unit ' || i,
      parent_path
    );
  END LOOP;
END $$;

-- Benchmark: Buscar todos los descendientes de un nodo
EXPLAIN ANALYZE
SELECT * FROM academic_units
WHERE path <@ 'root.50';

-- Debe usar índice GIST y tomar < 10ms
```

**Tests requeridos:**
- Integration: Ejecutar benchmark y validar que usa índice

##### Paso 6.5: Agregar índice faltante en tabla progress
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/migrations/018_add_progress_indexes.up.sql`
**Líneas:** 1-10 (nuevo archivo)
**Cambio:** Crear migración

```sql
-- 018_add_progress_indexes.up.sql

-- Índice compuesto para búsquedas por usuario y material
CREATE INDEX IF NOT EXISTS idx_progress_user_material
ON progress (user_id, material_id);

-- Índice para búsquedas de progreso completo
CREATE INDEX IF NOT EXISTS idx_progress_percentage
ON progress (progress_percentage)
WHERE progress_percentage = 100;
```

**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/postgres/migrations/018_add_progress_indexes.down.sql`
**Líneas:** 1-5 (nuevo archivo)
**Cambio:** Rollback

```sql
-- 018_add_progress_indexes.down.sql

DROP INDEX IF EXISTS idx_progress_user_material;
DROP INDEX IF EXISTS idx_progress_percentage;
```

**Tests requeridos:**
- Integration: Migrar up y validar índices con `\d progress`

##### Paso 6.6: Actualizar ARCHITECTURE.md con diagrama actualizado
**Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure/ARCHITECTURE.md`
**Líneas:** ~100-200 (sección de relaciones)
**Cambio:** Documentar eventos RabbitMQ

```markdown
## Eventos RabbitMQ

### Exchanges

| Exchange | Tipo | Propósito |
|----------|------|-----------|
| `edugo.materials` | topic | Eventos de materiales |
| `edugo.assessments` | topic | Eventos de evaluaciones |
| `edugo.events` | topic | Eventos generales del sistema |

### Routing Keys (formato punto)

- `material.uploaded` - Material subido a S3
- `material.completed` - Usuario completó material (100%)
- `material.deleted` - Material eliminado
- `material.reprocess` - Solicitud de reprocesamiento
- `assessment.generated` - Quiz generado por Worker
- `assessment.attempt_recorded` - Intento de quiz registrado
- `student.enrolled` - Estudiante inscrito a unidad

### Flujo de Eventos

```
[API Mobile] --material.uploaded--> [Worker] --assessment.generated--> [Analytics]
             --material.completed-->          --material.deleted-->
```

### Versionado

Todos los eventos usan envelope estándar con campo `event_version: "1.0.0"`.

Ver `edugo-shared/events/README.md` para catálogo completo.
```

**Tests requeridos:**
- Manual: Revisión de documentación

#### Tests Requeridos (Fase 6)

| Test | Tipo | Ubicación | Comando |
|------|------|-----------|---------|
| Migraciones up/down | Integration | Script bash | `./validate_migrations.sh` |
| Índices MongoDB | Integration | Script mongosh | `mongosh < 01_create_indexes.js` |
| Performance ltree | Integration | Script SQL | `psql < benchmark_ltree.sql` |
| Nueva migración 018 | Integration | migrate CLI | `migrate up` |

#### Configuración TestContainer

```go
// Setup para validar migraciones
func TestMigrationsUpDown(t *testing.T) {
	ctx := context.Background()

	pgContainer := testcontainers.ContainerRequest{
		Image:        "postgres:16-alpine",
		ExposedPorts: []string{"5432/tcp"},
		Env: map[string]string{
			"POSTGRES_USER": "test",
			"POSTGRES_PASSWORD": "test",
			"POSTGRES_DB": "edugo_test",
		},
		WaitingFor: wait.ForListeningPort("5432/tcp"),
	}

	container, _ := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: pgContainer,
		Started:          true,
	})
	defer container.Terminate(ctx)

	// Ejecutar migraciones
	dbURL := getConnectionString(container)

	// Up
	cmd := exec.Command("migrate", "-path", "postgres/migrations", "-database", dbURL, "up")
	output, err := cmd.CombinedOutput()
	if err != nil {
		t.Fatalf("Migration up failed: %s", output)
	}

	// Validar tablas
	db, _ := sql.Open("postgres", dbURL)
	var count int
	db.QueryRow("SELECT count(*) FROM information_schema.tables WHERE table_schema='public'").Scan(&count)

	if count != 20 {
		t.Errorf("Expected 20 tables, got %d", count)
	}

	// Down
	cmd = exec.Command("migrate", "-path", "postgres/migrations", "-database", dbURL, "down")
	cmd.Run()

	// Validar que no quedan tablas
	db.QueryRow("SELECT count(*) FROM information_schema.tables WHERE table_schema='public'").Scan(&count)
	if count != 0 {
		t.Errorf("Expected 0 tables after down, got %d", count)
	}
}
```

#### Post-requisitos
- ✅ Migraciones up/down validadas
- ✅ Esquema MongoDB documentado
- ✅ Índices MongoDB creados
- ✅ Índices PostgreSQL optimizados
- ✅ ARCHITECTURE.md actualizado con eventos
- ✅ Tests de infraestructura al 90%

---

## RESUMEN DE TIEMPOS ESTIMADOS

| Proyecto | Fase | Tiempo Estimado | Criticidad |
|----------|------|-----------------|------------|
| edugo-shared | Fase 1: DTOs compartidos | 2 días | 🔴 CRÍTICA |
| edugo-api-administracion | Fase 2: RBAC y CORS | 3 días | 🔴 CRÍTICA |
| edugo-api-administracion | Fase 3: Eventos y DTOs | 2 días | 🟡 ALTA |
| edugo-api-mobile | Fase 4: DTOs y publishers | 3 días | 🔴 CRÍTICA |
| edugo-worker | Fase 5: DTOs y processors | 4 días | 🟡 ALTA |
| edugo-infrastructure | Fase 6: Validación | 2 días | 🟢 MEDIA |
| **TOTAL** | | **16 días** | |

---

## VALIDACIÓN FINAL (Post-Todas las Fases)

### Checklist de Integración Completa

- [ ] **Flujo end-to-end material.uploaded:**
  1. Subir PDF desde frontend
  2. API Mobile publica evento con DTOs compartidos
  3. Worker consume, procesa y genera assessment/summary
  4. Frontend puede obtener quiz y resumen

- [ ] **Flujo end-to-end student.enrolled:**
  1. Admin crea membership con role=student
  2. API Admin publica evento
  3. Worker envía notificación de bienvenida

- [ ] **Flujo end-to-end material.deleted:**
  1. Admin elimina material en API Mobile
  2. API Mobile publica evento
  3. Worker limpia MongoDB

- [ ] **Flujo end-to-end assessment.attempt_recorded:**
  1. Estudiante completa quiz
  2. API Mobile publica evento
  3. Worker registra analytics y envía notificación

- [ ] **Validación CORS:**
  1. Frontend hace request desde localhost:3000
  2. API Admin y API Mobile responden con headers CORS correctos

- [ ] **Validación RBAC:**
  1. Usuario student intenta crear school → 403 Forbidden
  2. Usuario admin crea school → 201 Created

- [ ] **Validación paginación:**
  1. GET /schools?limit=10&offset=0 → 10 resultados
  2. GET /materials?limit=50 → máximo 50 resultados

- [ ] **Validación health checks:**
  1. GET /health en ambas APIs → 200 con componentes healthy
  2. Detener PostgreSQL → GET /health → 503 unhealthy

### Tests de Performance

```bash
# Benchmark material upload completo
time curl -X POST http://localhost:8080/v1/materials \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"title":"Test","subject":"Math","grade":"5"}'

# Debe completar en < 500ms

# Benchmark árbol jerárquico ltree con 1000 nodos
psql -c "EXPLAIN ANALYZE SELECT * FROM academic_units WHERE path <@ 'root.campus1'"

# Debe usar índice GIST y completar en < 10ms

# Benchmark procesamiento PDF por Worker
# PDF de 50 páginas debe completar en < 5 minutos
```

---

## ANEXO: Variables de Entorno Requeridas

### API Admin
```bash
CORS_ALLOWED_ORIGINS=http://localhost:3000,https://app.edugo.com
DATABASE_URL=postgres://user:pass@localhost:5432/edugo
JWT_SECRET=shared-secret-with-api-mobile
RABBITMQ_URL=amqp://guest:guest@localhost:5672/
```

### API Mobile
```bash
DATABASE_URL=postgres://user:pass@localhost:5432/edugo
MONGODB_URL=mongodb://localhost:27017/edugo
JWT_SECRET=shared-secret-with-api-admin
RABBITMQ_URL=amqp://guest:guest@localhost:5672/
S3_BUCKET=edugo-materials
S3_REGION=us-east-1
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
```

### Worker
```bash
DATABASE_URL=postgres://user:pass@localhost:5432/edugo
MONGODB_URL=mongodb://localhost:27017/edugo
RABBITMQ_URL=amqp://guest:guest@localhost:5672/
S3_BUCKET=edugo-materials
S3_REGION=us-east-1
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
OPENAI_API_KEY=sk-... # Opcional, usa fallback si no existe
```

---

## CONTACTO Y RESPONSABLES

| Área | Responsable | Siguiente Acción |
|------|-------------|------------------|
| edugo-shared | Backend Lead | Crear módulo events |
| edugo-api-administracion | Backend Lead Admin | Implementar RBAC y CORS |
| edugo-api-mobile | Backend Lead Mobile | Migrar a DTOs compartidos |
| edugo-worker | Backend Lead Worker | Completar processors stub |
| edugo-infrastructure | DevOps Lead | Validar migraciones |
| Testing | QA Lead | Ejecutar checklist de integración |
| Frontend | Frontend Lead | Validar endpoints con nuevo CORS |

---

**FIN DEL PLAN COMPLETO**

*Generado:* 2025-12-24
*Herramienta:* Sistema de Análisis Automatizado
*Basado en:* 4 informes consolidados del ecosistema EduGo
*Versión:* 1.0
