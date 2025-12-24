# Convenciones del Ecosistema EduGo

## Indice

1. [Arquitectura del Ecosistema](#arquitectura-del-ecosistema)
2. [Estructura de Proyectos](#estructura-de-proyectos)
3. [Convenciones de Nomenclatura](#convenciones-de-nomenclatura)
4. [Patron de Dependencias](#patron-de-dependencias)
5. [Convenciones de Codigo](#convenciones-de-codigo)
6. [Base de Datos](#base-de-datos)
7. [Mensajeria](#mensajeria)
8. [Configuracion](#configuracion)

---

## Arquitectura del Ecosistema

El ecosistema EduGo esta compuesto por 6 repositorios principales organizados en dos capas:

### Capa Base (Bibliotecas Compartidas)

| Proyecto | Descripcion | Modulo Go |
|----------|-------------|-----------|
| **edugo-shared** | Codigo comun reutilizable: middleware, conexiones a BD, logger, bootstrap, auth | `github.com/EduGoGroup/edugo-shared/*` |
| **edugo-infrastructure** | Migraciones, entidades y esquemas de BD (PostgreSQL y MongoDB) | `github.com/EduGoGroup/edugo-infrastructure/*` |
| **edugo-dev-environment** | Ambiente de desarrollo con Docker Compose para frontend developers | N/A (solo Docker) |

### Capa de Aplicacion (Servicios)

| Proyecto | Descripcion | Puerto | Uso Principal |
|----------|-------------|--------|---------------|
| **edugo-api-administracion** | API para administracion del sistema | TBD | Gestion de colegios, usuarios, materias |
| **edugo-api-mobile** | API para aplicacion movil | TBD | Endpoints de alta frecuencia |
| **edugo-worker** | Procesador de tareas en background | N/A | Procesamiento asincrono via RabbitMQ |

---

## Estructura de Proyectos

### Estructura Base de APIs (api-admin, api-mobile)

```
proyecto/
├── cmd/
│   └── main.go                    # Punto de entrada
├── config/
│   └── *.yaml                     # Archivos de configuracion
├── docs/
│   ├── swagger.json               # Documentacion OpenAPI
│   └── swagger.yaml
├── internal/
│   ├── application/
│   │   ├── dto/                   # Data Transfer Objects
│   │   └── service/               # Servicios de aplicacion
│   ├── auth/
│   │   ├── dto/
│   │   ├── handler/
│   │   ├── middleware/
│   │   ├── repository/
│   │   └── service/
│   ├── bootstrap/                 # Inicializacion de dependencias
│   │   ├── adapter/
│   │   ├── bootstrap.go
│   │   ├── bridge.go
│   │   └── custom_factories.go
│   ├── config/                    # Carga de configuracion
│   │   ├── config.go
│   │   ├── loader.go
│   │   └── validator.go
│   ├── container/                 # Inyeccion de dependencias
│   │   └── container.go
│   ├── domain/                    # Entidades de dominio
│   └── infrastructure/            # Implementaciones de infra
│       └── http/
│           ├── handler/
│           └── router/
├── test/
│   ├── integration/
│   └── unit/
├── docker-compose.yml
├── Dockerfile
├── go.mod
├── go.sum
└── Makefile
```

### Estructura del Worker

```
edugo-worker/
├── cmd/
│   └── main.go
├── config/
├── internal/
│   ├── application/
│   │   ├── dto/
│   │   │   └── event_dto.go
│   │   └── processor/             # Procesadores de eventos
│   │       ├── processor.go
│   │       ├── registry.go
│   │       ├── material_uploaded_processor.go
│   │       ├── material_deleted_processor.go
│   │       ├── material_reprocess_processor.go
│   │       ├── assessment_attempt_processor.go
│   │       └── student_enrolled_processor.go
│   ├── bootstrap/
│   ├── client/                    # Clientes HTTP externos
│   │   └── auth_client.go
│   ├── config/
│   ├── container/
│   ├── domain/
│   │   ├── constants/
│   │   ├── service/
│   │   └── valueobject/
│   └── infrastructure/
│       ├── messaging/
│       │   ├── consumer/
│       │   └── publisher/
│       ├── nlp/
│       ├── pdf/
│       └── persistence/
│           └── mongodb/
│               └── repository/
└── test/
```

### Estructura de edugo-shared

```
edugo-shared/
├── auth/                          # Autenticacion y autorizacion
├── bootstrap/                     # Inicializacion de servicios
│   ├── bootstrap.go
│   ├── factory_*.go               # Factories para cada servicio
│   ├── init_*.go                  # Inicializadores
│   ├── resources.go
│   └── health_check.go
├── common/                        # Utilidades comunes
├── config/                        # Configuracion base
├── database/
│   ├── mongodb/                   # Conexion MongoDB
│   │   ├── config.go
│   │   └── connection.go
│   └── postgres/                  # Conexion PostgreSQL
│       ├── config.go
│       ├── connection.go
│       └── transaction.go
├── lifecycle/                     # Ciclo de vida de aplicacion
├── logger/                        # Abstraccion de logging
│   ├── logger.go                  # Interface
│   ├── zap_logger.go
│   └── logrus_logger.go
├── messaging/                     # RabbitMQ
├── middleware/
│   └── gin/
│       ├── context.go
│       └── jwt_auth.go
├── storage/                       # AWS S3
└── testing/                       # Utilidades de testing
```

### Estructura de edugo-infrastructure

```
edugo-infrastructure/
├── mongodb/
│   ├── cmd/
│   │   └── migrate/
│   ├── entities/                  # Entidades MongoDB
│   │   ├── material_assessment.go
│   │   ├── material_summary.go
│   │   └── material_event.go
│   └── migrations/
│       ├── structure/             # Migraciones de estructura
│       │   ├── 001_material_assessment.go
│       │   ├── 002_material_content.go
│       │   └── ...
│       └── constraints/           # Indices y constraints
│           ├── 001_material_assessment_indexes.go
│           └── ...
├── postgres/
│   ├── cmd/
│   │   ├── migrate/
│   │   └── runner/
│   ├── entities/                  # Entidades PostgreSQL (GORM)
│   │   ├── user.go
│   │   ├── school.go
│   │   ├── subject.go
│   │   ├── unit.go
│   │   ├── material.go
│   │   ├── material_version.go
│   │   ├── assessment.go
│   │   ├── assessment_attempt.go
│   │   ├── assessment_attempt_answer.go
│   │   ├── academic_unit.go
│   │   └── guardian_relation.go
│   └── migrations/
│       └── sql/                   # Migraciones SQL
├── schemas/                       # Validadores de esquema
│   └── validator.go
└── tools/
    └── mock-generator/            # Generador de datos mock
```

---

## Convenciones de Nomenclatura

### Paquetes Go

- **Nombres en minusculas**: `bootstrap`, `handler`, `service`
- **Singular para paquetes**: `handler` no `handlers`
- **Descriptivo y conciso**: `dto`, `valueobject`

### Archivos

| Tipo | Patron | Ejemplo |
|------|--------|---------|
| Handler | `{entidad}_handler.go` | `user_handler.go` |
| Service | `{entidad}_service.go` | `material_service.go` |
| Repository | `{entidad}_repository.go` | `token_repository.go` |
| DTO | `{entidad}_dto.go` | `assessment_dto.go` |
| Test unitario | `{archivo}_test.go` | `service_test.go` |
| Test integracion | `{flujo}_integration_test.go` | `verify_integration_test.go` |
| Migracion SQL | `{numero}_{descripcion}.sql` | `001_create_users.sql` |
| Migracion Go | `{numero}_{descripcion}.go` | `001_material_assessment.go` |

### Variables y Funciones

- **camelCase** para variables locales: `userService`, `materialID`
- **PascalCase** para exportados: `NewUserService`, `MaterialHandler`
- **Acronimos en mayusculas**: `userID`, `httpClient`, `sqlDB`

### Constantes

```go
// Constantes de eventos
const (
    EventMaterialUploaded    = "material.uploaded"
    EventMaterialDeleted     = "material.deleted"
    EventAssessmentAttempted = "assessment.attempted"
)
```

---

## Patron de Dependencias

### Jerarquia de Importacion

```
edugo-shared (base)
       ↓
edugo-infrastructure (BD)
       ↓
edugo-api-* / edugo-worker (aplicaciones)
```

### Modulos Disponibles en edugo-shared

| Modulo | Import Path | Descripcion |
|--------|-------------|-------------|
| Auth | `github.com/EduGoGroup/edugo-shared/auth` | JWT y autenticacion |
| Bootstrap | `github.com/EduGoGroup/edugo-shared/bootstrap` | Inicializacion de servicios |
| Common | `github.com/EduGoGroup/edugo-shared/common` | Utilidades generales |
| Lifecycle | `github.com/EduGoGroup/edugo-shared/lifecycle` | Gestion de ciclo de vida |
| Logger | `github.com/EduGoGroup/edugo-shared/logger` | Abstraccion de logging |
| Middleware | `github.com/EduGoGroup/edugo-shared/middleware/gin` | Middleware para Gin |
| Testing | `github.com/EduGoGroup/edugo-shared/testing` | Helpers de testing |

### Modulos Disponibles en edugo-infrastructure

| Modulo | Import Path | Descripcion |
|--------|-------------|-------------|
| Postgres Entities | `github.com/EduGoGroup/edugo-infrastructure/postgres` | Entidades GORM |
| MongoDB Entities | `github.com/EduGoGroup/edugo-infrastructure/mongodb` | Entidades MongoDB |
| Schemas | `github.com/EduGoGroup/edugo-infrastructure/schemas` | Validadores |
| Mock Generator | `github.com/EduGoGroup/edugo-infrastructure/tools/mock-generator` | Datos de prueba |

---

## Convenciones de Codigo

### Patron Handler-Service-Repository

```go
// Handler (presentation layer)
type UserHandler struct {
    service UserService
    logger  logger.Logger
}

func (h *UserHandler) GetUser(c *gin.Context) {
    // Validar request
    // Llamar service
    // Responder
}

// Service (business logic)
type UserService struct {
    repo   UserRepository
    logger logger.Logger
}

func (s *UserService) GetUserByID(ctx context.Context, id uuid.UUID) (*User, error) {
    // Logica de negocio
    return s.repo.FindByID(ctx, id)
}

// Repository (data access)
type UserRepository interface {
    FindByID(ctx context.Context, id uuid.UUID) (*User, error)
    Save(ctx context.Context, user *User) error
}
```

### DTOs

```go
// Request DTO
type CreateUserRequest struct {
    Email     string `json:"email" binding:"required,email"`
    FirstName string `json:"first_name" binding:"required,min=2"`
    LastName  string `json:"last_name" binding:"required,min=2"`
    Role      string `json:"role" binding:"required,oneof=student teacher admin"`
}

// Response DTO
type UserResponse struct {
    ID        uuid.UUID `json:"id"`
    Email     string    `json:"email"`
    FirstName string    `json:"first_name"`
    LastName  string    `json:"last_name"`
    Role      string    `json:"role"`
    CreatedAt time.Time `json:"created_at"`
}
```

### Manejo de Errores

```go
// Errores de dominio
var (
    ErrUserNotFound     = errors.New("user not found")
    ErrInvalidCredentials = errors.New("invalid credentials")
)

// En handlers
func (h *UserHandler) GetUser(c *gin.Context) {
    user, err := h.service.GetUser(ctx, id)
    if err != nil {
        switch {
        case errors.Is(err, ErrUserNotFound):
            c.JSON(http.StatusNotFound, gin.H{"error": err.Error()})
        default:
            h.logger.Error("error getting user", logger.Error(err))
            c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        }
        return
    }
    c.JSON(http.StatusOK, user)
}
```

---

## Base de Datos

### PostgreSQL

- **ORM**: GORM v1.25+
- **Driver**: pgx v5
- **Migraciones**: SQL puro con numeracion secuencial

#### Entidades Principales

| Tabla | Descripcion |
|-------|-------------|
| `users` | Usuarios del sistema |
| `schools` | Colegios/instituciones |
| `subjects` | Materias |
| `units` | Unidades tematicas |
| `materials` | Materiales educativos |
| `material_versions` | Versiones de materiales |
| `assessments` | Evaluaciones |
| `assessment_attempts` | Intentos de evaluacion |
| `assessment_attempt_answers` | Respuestas de intentos |
| `academic_units` | Periodos academicos |
| `guardian_relations` | Relaciones acudiente-estudiante |

### MongoDB

- **Driver**: mongo-driver v1.17+
- **Migraciones**: Go con estructura/constraints separados

#### Colecciones Principales

| Coleccion | Descripcion |
|-----------|-------------|
| `material_assessments` | Evaluaciones de materiales |
| `material_content` | Contenido de materiales |
| `material_summaries` | Resumenes de materiales |
| `material_events` | Eventos de materiales |
| `assessment_attempt_results` | Resultados de intentos |
| `notifications` | Notificaciones |
| `analytics_events` | Eventos de analytics |
| `audit_logs` | Logs de auditoria |

---

## Mensajeria

### RabbitMQ

- **Biblioteca**: `github.com/rabbitmq/amqp091-go`
- **Patron**: Event-driven con processors

#### Eventos del Sistema

| Evento | Productor | Consumidor | Descripcion |
|--------|-----------|------------|-------------|
| `material.uploaded` | api-mobile | worker | Material subido |
| `material.deleted` | api-mobile | worker | Material eliminado |
| `material.reprocess` | api-admin | worker | Reprocesar material |
| `assessment.attempted` | api-mobile | worker | Intento de evaluacion |
| `student.enrolled` | api-admin | worker | Estudiante matriculado |

#### Estructura de Processor

```go
type Processor interface {
    EventType() string
    Process(ctx context.Context, event Event) error
}

type MaterialUploadedProcessor struct {
    // dependencias
}

func (p *MaterialUploadedProcessor) EventType() string {
    return "material.uploaded"
}

func (p *MaterialUploadedProcessor) Process(ctx context.Context, event Event) error {
    // Logica de procesamiento
}
```

---

## Configuracion

### Formato

- **Formato**: YAML
- **Biblioteca**: Viper
- **Ambiente**: Variables de entorno con prefijo `EDUGO_`

### Estructura Tipica

```yaml
server:
  host: "0.0.0.0"
  port: 8080
  mode: "debug"  # debug, release, test

database:
  postgres:
    host: "localhost"
    port: 5432
    user: "edugo"
    password: "secret"
    database: "edugo_db"
    sslmode: "disable"

  mongodb:
    uri: "mongodb://localhost:27017"
    database: "edugo_mongo"

rabbitmq:
  host: "localhost"
  port: 5672
  user: "guest"
  password: "guest"
  vhost: "/"

jwt:
  secret: "your-secret-key"
  expiration: "24h"

logging:
  level: "info"
  format: "json"
```

---

## Testing

### Tipos de Tests

| Tipo | Ubicacion | Convencion |
|------|-----------|------------|
| Unit | `*_test.go` junto al archivo | Mismo paquete |
| Integration | `test/integration/` | Paquete separado |

### Herramientas

- **Framework**: `testify` (assert, require, mock)
- **Containers**: `testcontainers-go` para PostgreSQL, MongoDB, RabbitMQ
- **Mocks**: `mockery` o `go-mock`

### Ejemplo de Test de Integracion

```go
func TestMaterialFlow(t *testing.T) {
    suite := NewIntegrationSuite(t)
    defer suite.Cleanup()

    // Arrange
    material := suite.CreateTestMaterial()

    // Act
    resp := suite.POST("/api/materials", material)

    // Assert
    assert.Equal(t, http.StatusCreated, resp.Code)
}
```

---

## Version de Go

- **Version minima**: Go 1.25
- **Modules**: Activados (`go.mod` en cada proyecto)

---

## Versionado

- **Formato**: Semantic Versioning (SemVer)
- **Tags**: `v0.X.Y` para shared e infrastructure
- **Branches**: `main` (produccion), `develop` (desarrollo)

---

*Documento generado automaticamente - Ultima actualizacion: Diciembre 2024*
