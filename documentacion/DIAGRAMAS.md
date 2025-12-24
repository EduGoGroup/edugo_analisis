# Diagramas del Ecosistema EduGo

Este documento contiene los diagramas completos del ecosistema EduGo, incluyendo arquitectura, modelo de datos, flujos de procesos y dependencias.

---

## Indice de Diagramas

1. [Arquitectura del Sistema](#1-arquitectura-del-sistema)
   - [1.1 Vista General](#11-vista-general)
   - [1.2 Detalle de Componentes](#12-detalle-de-componentes)
   - [1.3 Arquitectura de Microservicios](#13-arquitectura-de-microservicios)
2. [Modelo de Datos](#2-modelo-de-datos)
   - [2.1 Diagrama ER - PostgreSQL](#21-diagrama-er---postgresql)
   - [2.2 Estructura MongoDB](#22-estructura-mongodb)
   - [2.3 Relaciones entre PostgreSQL y MongoDB](#23-relaciones-entre-postgresql-y-mongodb)
3. [Flujos de Proceso](#3-flujos-de-proceso)
   - [3.1 Flujo de Materiales Educativos](#31-flujo-de-materiales-educativos)
   - [3.2 Flujo de Evaluaciones](#32-flujo-de-evaluaciones)
   - [3.3 Flujo de Usuarios y Matriculas](#33-flujo-de-usuarios-y-matriculas)
4. [Eventos y Workers](#4-eventos-y-workers)
   - [4.1 Estado de Procesamiento](#41-estado-de-procesamiento)
   - [4.2 Flujo por Tipo de Evento](#42-flujo-por-tipo-de-evento)
   - [4.3 Arquitectura de Mensajeria](#43-arquitectura-de-mensajeria)
5. [Dependencias](#5-dependencias)
   - [5.1 Dependencias entre Proyectos](#51-dependencias-entre-proyectos)
   - [5.2 Stack Tecnologico](#52-stack-tecnologico)
6. [Infraestructura](#6-infraestructura)
   - [6.1 Ambiente de Desarrollo](#61-ambiente-de-desarrollo)
   - [6.2 Perfiles de Docker Compose](#62-perfiles-de-docker-compose)

---

## 1. Arquitectura del Sistema

### 1.1 Vista General

El ecosistema EduGo esta compuesto por multiples microservicios que trabajan en conjunto para proporcionar una plataforma educativa completa. La arquitectura sigue el patron de microservicios con comunicacion asincrona mediante colas de mensajes.

```mermaid
graph TB
    subgraph "Clientes"
        WEB[Web Admin<br/>Panel Administrativo]
        MOB[Mobile App<br/>Estudiantes/Profesores]
    end

    subgraph "API Gateway Layer"
        API_ADMIN[API Administracion<br/>:8082]
        API_MOBILE[API Mobile<br/>:8081]
    end

    subgraph "Procesamiento Asincrono"
        WORKER[Worker<br/>Procesador de Eventos]
        RMQ[RabbitMQ<br/>:5672/:15672]
    end

    subgraph "Persistencia"
        PG[(PostgreSQL<br/>:5432<br/>Datos Transaccionales)]
        MDB[(MongoDB<br/>:27017<br/>Datos Documentales)]
        S3[AWS S3<br/>Archivos/Materiales]
    end

    subgraph "Servicios Externos"
        OPENAI[OpenAI API<br/>GPT-4]
    end

    WEB --> API_ADMIN
    MOB --> API_MOBILE

    API_ADMIN --> PG
    API_ADMIN --> MDB

    API_MOBILE --> PG
    API_MOBILE --> MDB
    API_MOBILE --> S3
    API_MOBILE --> RMQ

    RMQ --> WORKER
    WORKER --> PG
    WORKER --> MDB
    WORKER --> S3
    WORKER --> OPENAI
```

### 1.2 Detalle de Componentes

Cada componente del ecosistema tiene responsabilidades especificas:

| Componente | Puerto | Responsabilidad |
|------------|--------|-----------------|
| API Administracion | 8082 | Gestion de colegios, usuarios, unidades academicas, materias |
| API Mobile | 8081 | Materiales educativos, evaluaciones, progreso de estudiantes |
| Worker | N/A | Procesamiento asincrono de materiales, generacion IA |
| PostgreSQL | 5432 | Datos transaccionales, relaciones, ACID |
| MongoDB | 27017 | Documentos complejos, preguntas, resumenes |
| RabbitMQ | 5672/15672 | Cola de mensajes, eventos asincronos |
| Redis | 6379 | Cache, sesiones (opcional) |

### 1.3 Arquitectura de Microservicios

```mermaid
graph LR
    subgraph "edugo-api-administracion"
        A_SCHOOL[SchoolService]
        A_USER[UserService]
        A_UNIT[AcademicUnitService]
        A_MEMBER[MembershipService]
        A_GUARD[GuardianService]
        A_SUBJ[SubjectService]
    end

    subgraph "edugo-api-mobile"
        M_MAT[MaterialService]
        M_ASSESS[AssessmentAttemptService]
        M_PROG[ProgressService]
        M_STATS[StatsService]
        M_SUMMARY[SummaryService]
    end

    subgraph "edugo-worker"
        W_UPLOAD[MaterialUploadedProcessor]
        W_DELETE[MaterialDeletedProcessor]
        W_REPROC[MaterialReprocessProcessor]
        W_ATTEMPT[AssessmentAttemptProcessor]
        W_ENROLL[StudentEnrolledProcessor]
    end

    A_SCHOOL --> PG[(PostgreSQL)]
    A_USER --> PG
    A_UNIT --> PG
    A_MEMBER --> PG

    M_MAT --> PG
    M_MAT --> S3[AWS S3]
    M_MAT --> RMQ[RabbitMQ]
    M_ASSESS --> PG
    M_ASSESS --> MDB[(MongoDB)]

    RMQ --> W_UPLOAD
    RMQ --> W_DELETE
    RMQ --> W_ATTEMPT

    W_UPLOAD --> PG
    W_UPLOAD --> MDB
    W_UPLOAD --> OPENAI[OpenAI]
```

---

## 2. Modelo de Datos

### 2.1 Diagrama ER - PostgreSQL

El modelo relacional de PostgreSQL maneja los datos transaccionales del sistema educativo.

```mermaid
erDiagram
    SCHOOLS ||--o{ USERS : "pertenecen a"
    SCHOOLS ||--o{ ACADEMIC_UNITS : "contiene"
    SCHOOLS ||--o{ MEMBERSHIPS : "tiene"
    SCHOOLS ||--o{ MATERIALS : "almacena"
    SCHOOLS ||--o{ UNITS : "organiza"

    USERS ||--o{ MEMBERSHIPS : "tiene"
    USERS ||--o{ MATERIALS : "sube"
    USERS ||--o{ ASSESSMENT_ATTEMPTS : "realiza"
    USERS ||--o{ PROGRESS : "registra"
    USERS ||--o{ GUARDIAN_RELATIONS : "es guardian"
    USERS ||--o{ GUARDIAN_RELATIONS : "es estudiante"

    ACADEMIC_UNITS ||--o{ ACADEMIC_UNITS : "padre de"
    ACADEMIC_UNITS ||--o{ MEMBERSHIPS : "asocia"
    ACADEMIC_UNITS ||--o{ MATERIALS : "pertenece a"

    UNITS ||--o{ UNITS : "padre de"

    MATERIALS ||--o{ MATERIAL_VERSIONS : "tiene versiones"
    MATERIALS ||--|| ASSESSMENTS : "genera"
    MATERIALS ||--o{ PROGRESS : "tiene progreso"

    ASSESSMENTS ||--o{ ASSESSMENT_ATTEMPTS : "tiene intentos"

    ASSESSMENT_ATTEMPTS ||--o{ ASSESSMENT_ATTEMPT_ANSWERS : "contiene respuestas"

    SCHOOLS {
        uuid id PK
        string name
        string code UK
        string address
        string city
        string country
        string phone
        string email
        jsonb metadata
        boolean is_active
        string subscription_tier
        int max_teachers
        int max_students
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    USERS {
        uuid id PK
        string email UK
        string password_hash
        string first_name
        string last_name
        string role
        uuid school_id FK
        boolean is_active
        boolean email_verified
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    ACADEMIC_UNITS {
        uuid id PK
        uuid parent_unit_id FK
        uuid school_id FK
        string name
        string code
        string type
        string description
        string level
        int academic_year
        jsonb metadata
        boolean is_active
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    MEMBERSHIPS {
        uuid id PK
        uuid user_id FK
        uuid school_id FK
        uuid academic_unit_id FK
        string role
        jsonb metadata
        boolean is_active
        timestamp enrolled_at
        timestamp withdrawn_at
        timestamp created_at
        timestamp updated_at
    }

    MATERIALS {
        uuid id PK
        uuid school_id FK
        uuid uploaded_by_teacher_id FK
        uuid academic_unit_id FK
        string title
        string description
        string subject
        string grade
        string file_url
        string file_type
        bigint file_size_bytes
        string status
        timestamp processing_started_at
        timestamp processing_completed_at
        boolean is_public
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    MATERIAL_VERSIONS {
        uuid id PK
        uuid material_id FK
        int version_number
        string title
        string content_url
        uuid changed_by FK
        timestamp created_at
    }

    ASSESSMENTS {
        uuid id PK
        uuid material_id FK
        string mongo_document_id
        int questions_count
        int total_questions
        string title
        int pass_threshold
        int max_attempts
        int time_limit_minutes
        string status
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    ASSESSMENT_ATTEMPTS {
        uuid id PK
        uuid assessment_id FK
        uuid student_id FK
        timestamp started_at
        timestamp completed_at
        decimal score
        decimal max_score
        decimal percentage
        int time_spent_seconds
        string idempotency_key
        string status
        timestamp created_at
        timestamp updated_at
    }

    ASSESSMENT_ATTEMPT_ANSWERS {
        uuid id PK
        uuid attempt_id FK
        int question_index
        string student_answer
        boolean is_correct
        decimal points_earned
        decimal max_points
        int time_spent_seconds
        timestamp answered_at
        timestamp created_at
        timestamp updated_at
    }

    SUBJECTS {
        uuid id PK
        string name
        string description
        jsonb metadata
        boolean is_active
        timestamp created_at
        timestamp updated_at
    }

    UNITS {
        uuid id PK
        uuid school_id FK
        uuid parent_unit_id FK
        string name
        string description
        boolean is_active
        timestamp created_at
        timestamp updated_at
    }

    GUARDIAN_RELATIONS {
        uuid id PK
        uuid guardian_id FK
        uuid student_id FK
        string relationship_type
        boolean is_active
        timestamp created_at
        timestamp updated_at
        string created_by
    }

    PROGRESS {
        uuid material_id PK
        uuid user_id PK
        int percentage
        int last_page
        string status
        timestamp last_accessed_at
        timestamp created_at
        timestamp updated_at
    }
```

### 2.2 Estructura MongoDB

MongoDB almacena documentos complejos que no encajan bien en el modelo relacional.

```mermaid
classDiagram
    class MaterialAssessment {
        +ObjectId _id
        +String material_id
        +Question[] questions
        +Int total_questions
        +Int total_points
        +Int version
        +String ai_model
        +Int processing_time_ms
        +TokenUsage token_usage
        +AssessmentMetadata metadata
        +DateTime created_at
        +DateTime updated_at
    }

    class Question {
        +String question_id
        +String question_text
        +String question_type
        +Option[] options
        +String correct_answer
        +String explanation
        +Int points
        +String difficulty
        +String[] tags
    }

    class Option {
        +String option_id
        +String option_text
    }

    class TokenUsage {
        +Int prompt_tokens
        +Int completion_tokens
        +Int total_tokens
    }

    class AssessmentMetadata {
        +String average_difficulty
        +Int estimated_time_min
        +Int source_length
        +Boolean has_images
    }

    class MaterialSummary {
        +ObjectId _id
        +String material_id
        +String summary
        +String[] key_points
        +String language
        +Int word_count
        +Int version
        +String ai_model
        +Int processing_time_ms
        +TokenUsage token_usage
        +SummaryMetadata metadata
        +DateTime created_at
        +DateTime updated_at
    }

    class SummaryMetadata {
        +Int source_length
        +Boolean has_images
    }

    class MaterialEvent {
        +ObjectId _id
        +String event_type
        +String material_id
        +String user_id
        +Object payload
        +String status
        +String error_msg
        +String stack_trace
        +Int retry_count
        +DateTime processed_at
        +DateTime created_at
        +DateTime updated_at
    }

    MaterialAssessment "1" --> "*" Question : contains
    Question "1" --> "*" Option : has
    MaterialAssessment "1" --> "0..1" TokenUsage : uses
    MaterialAssessment "1" --> "0..1" AssessmentMetadata : has
    MaterialSummary "1" --> "0..1" TokenUsage : uses
    MaterialSummary "1" --> "0..1" SummaryMetadata : has
```

### 2.3 Relaciones entre PostgreSQL y MongoDB

La arquitectura hibrida utiliza referencias entre ambas bases de datos:

```mermaid
graph TB
    subgraph "PostgreSQL - Datos Transaccionales"
        PG_MATERIALS[materials<br/>- id: UUID<br/>- status: string<br/>- file_url: string]
        PG_ASSESSMENTS[assessments<br/>- id: UUID<br/>- material_id: UUID<br/>- mongo_document_id: string]
        PG_ATTEMPTS[assessment_attempts<br/>- id: UUID<br/>- assessment_id: UUID]
    end

    subgraph "MongoDB - Datos Documentales"
        MDB_SUMMARY[material_summary<br/>- _id: ObjectId<br/>- material_id: string<br/>- summary: string]
        MDB_ASSESS[material_assessment<br/>- _id: ObjectId<br/>- material_id: string<br/>- questions: array]
        MDB_EVENTS[material_event<br/>- _id: ObjectId<br/>- material_id: string<br/>- event_type: string]
    end

    PG_MATERIALS -->|material_id| MDB_SUMMARY
    PG_MATERIALS -->|material_id| MDB_ASSESS
    PG_MATERIALS -->|material_id| MDB_EVENTS
    PG_ASSESSMENTS -->|mongo_document_id| MDB_ASSESS

    style PG_MATERIALS fill:#336699
    style PG_ASSESSMENTS fill:#336699
    style PG_ATTEMPTS fill:#336699
    style MDB_SUMMARY fill:#4DB33D
    style MDB_ASSESS fill:#4DB33D
    style MDB_EVENTS fill:#4DB33D
```

---

## 3. Flujos de Proceso

### 3.1 Flujo de Materiales Educativos

Este flujo describe el proceso completo desde que un profesor sube un material hasta que el estudiante lo consume.

```mermaid
sequenceDiagram
    participant P as Profesor
    participant API as API Mobile
    participant S3 as AWS S3
    participant PG as PostgreSQL
    participant RMQ as RabbitMQ
    participant W as Worker
    participant AI as OpenAI GPT-4
    participant MDB as MongoDB
    participant E as Estudiante

    Note over P,E: Fase 1: Subida de Material
    P->>API: POST /materials (metadata)
    API->>PG: INSERT material (status=uploaded)
    PG-->>API: material_id
    API-->>P: {material_id, presigned_url}

    P->>S3: PUT archivo (presigned URL)
    S3-->>P: 200 OK

    P->>API: POST /materials/{id}/upload-complete
    API->>PG: UPDATE material (file_url, file_size)
    API->>RMQ: publish(material.uploaded)
    API-->>P: 200 OK

    Note over P,E: Fase 2: Procesamiento con IA
    RMQ->>W: consume(material.uploaded)
    W->>PG: UPDATE status=processing
    W->>S3: GET archivo PDF
    S3-->>W: contenido PDF

    W->>W: Extraer texto del PDF

    W->>AI: Generar resumen
    AI-->>W: resumen + key_points
    W->>MDB: INSERT material_summary

    W->>AI: Generar evaluacion
    AI-->>W: questions + answers
    W->>MDB: INSERT material_assessment

    W->>PG: INSERT assessment (mongo_document_id)
    W->>PG: UPDATE status=ready
    W->>MDB: INSERT material_event (completed)

    Note over P,E: Fase 3: Consumo por Estudiante
    E->>API: GET /materials/{id}
    API->>PG: SELECT material
    API-->>E: material con status=ready

    E->>API: GET /materials/{id}/summary
    API->>MDB: FIND material_summary
    API-->>E: resumen y puntos clave

    E->>API: GET /materials/{id}/assessment
    API->>PG: SELECT assessment
    API->>MDB: FIND material_assessment
    API->>API: sanitizeQuestions()
    API-->>E: preguntas SIN respuestas correctas
```

### 3.2 Flujo de Evaluaciones

Proceso de toma y calificacion de evaluaciones por estudiantes.

```mermaid
sequenceDiagram
    participant E as Estudiante
    participant API as API Mobile
    participant PG as PostgreSQL
    participant MDB as MongoDB
    participant RMQ as RabbitMQ
    participant W as Worker

    Note over E,W: Fase 1: Obtener Evaluacion
    E->>API: GET /assessments/material/{material_id}
    API->>PG: SELECT assessment WHERE material_id
    API->>MDB: FIND material_assessment (mongo_document_id)
    API->>API: sanitizeQuestions()<br/>Remueve correct_answer
    API-->>E: AssessmentResponse (preguntas sin respuestas)

    Note over E,W: Fase 2: Enviar Respuestas
    E->>API: POST /assessments/{id}/attempts
    API->>PG: COUNT attempts WHERE student_id
    API->>API: canAttempt(max_attempts)?

    alt Max intentos alcanzado
        API-->>E: 403 max_attempts_reached
    else Puede intentar
        API->>MDB: FIND questions con correct_answer
        API->>API: validateAndScoreAnswers()

        Note right of API: Score calculado en SERVIDOR<br/>NUNCA confiar en cliente

        API->>PG: INSERT assessment_attempt
        API->>PG: INSERT assessment_attempt_answers[]
        API-->>E: AttemptResultResponse
    end

    Note over E,W: Fase 3: Procesamiento Asincrono (opcional)
    API->>RMQ: publish(assessment.attempt)
    RMQ->>W: consume(assessment.attempt)
    W->>W: Actualizar estadisticas
    W->>W: Enviar notificacion docente si score bajo

    Note over E,W: Fase 4: Consultar Historial
    E->>API: GET /assessments/attempts/history
    API->>PG: SELECT attempts WHERE student_id
    API-->>E: AttemptHistoryResponse
```

### 3.3 Flujo de Usuarios y Matriculas

Gestion de colegios, usuarios y matriculas desde la API de Administracion.

```mermaid
sequenceDiagram
    participant ADMIN as Super Admin
    participant API as API Admin
    participant PG as PostgreSQL

    Note over ADMIN,PG: Fase 1: Registro de Colegio
    ADMIN->>API: POST /schools
    API->>PG: SELECT EXISTS code
    alt Codigo existe
        API-->>ADMIN: 409 already_exists
    else Codigo disponible
        API->>PG: INSERT school
        API-->>ADMIN: SchoolResponse
    end

    Note over ADMIN,PG: Fase 2: Crear Usuario Admin del Colegio
    ADMIN->>API: POST /users (role=admin, school_id)
    API->>PG: SELECT EXISTS email
    API->>API: hashPassword(bcrypt cost=12)
    API->>PG: INSERT user
    API-->>ADMIN: UserResponse

    Note over ADMIN,PG: Fase 3: Crear Estructura Academica
    ADMIN->>API: POST /academic-units (type=grade)
    API->>PG: INSERT academic_unit (parent=null)
    ADMIN->>API: POST /academic-units (type=class, parent_id)
    API->>PG: INSERT academic_unit (parent=grade_id)
    API-->>ADMIN: AcademicUnitResponse[]

    Note over ADMIN,PG: Fase 4: Crear Profesores y Estudiantes
    ADMIN->>API: POST /users (role=teacher)
    API->>PG: INSERT user
    ADMIN->>API: POST /users (role=student)
    API->>PG: INSERT user

    Note over ADMIN,PG: Fase 5: Matricular Estudiantes
    ADMIN->>API: POST /memberships
    API->>PG: INSERT membership (user_id, academic_unit_id)
    API-->>ADMIN: MembershipResponse

    Note over ADMIN,PG: Fase 6: Asignar Acudientes
    ADMIN->>API: POST /guardians
    API->>PG: SELECT user WHERE role=guardian
    API->>PG: INSERT guardian_relation
    API-->>ADMIN: GuardianRelationResponse
```

---

## 4. Eventos y Workers

### 4.1 Estado de Procesamiento

Los materiales pasan por diferentes estados durante su procesamiento:

```mermaid
stateDiagram-v2
    [*] --> uploaded : Material creado
    uploaded --> processing : Worker toma evento
    processing --> ready : Procesamiento exitoso
    processing --> failed : Error en procesamiento
    failed --> processing : Reintento (max 3)
    failed --> [*] : Abandono tras 3 intentos
    ready --> [*] : Material disponible

    note right of uploaded
        Material metadata guardado
        Archivo subido a S3
        Evento publicado a RabbitMQ
    end note

    note right of processing
        Extraccion de texto PDF
        Generacion de resumen (IA)
        Generacion de evaluacion (IA)
    end note

    note right of ready
        Resumen en MongoDB
        Assessment en MongoDB
        Assessment ref en PostgreSQL
    end note
```

### 4.2 Flujo por Tipo de Evento

El Worker maneja diferentes tipos de eventos:

```mermaid
graph TB
    subgraph "Event Types"
        E1[material_uploaded]
        E2[material_reprocess]
        E3[material_deleted]
        E4[assessment_attempt]
        E5[student_enrolled]
    end

    subgraph "Processor Registry"
        REG[Registry<br/>map event_type -> Processor]
    end

    subgraph "Processors"
        P1[MaterialUploadedProcessor]
        P2[MaterialReprocessProcessor]
        P3[MaterialDeletedProcessor]
        P4[AssessmentAttemptProcessor]
        P5[StudentEnrolledProcessor]
    end

    E1 --> REG
    E2 --> REG
    E3 --> REG
    E4 --> REG
    E5 --> REG

    REG --> P1
    REG --> P2
    REG --> P3
    REG --> P4
    REG --> P5

    subgraph "MaterialUploadedProcessor Actions"
        A1[1. UPDATE status=processing]
        A2[2. GET PDF from S3]
        A3[3. Extract text]
        A4[4. Generate summary with AI]
        A5[5. Save summary to MongoDB]
        A6[6. Generate quiz with AI]
        A7[7. Save assessment to MongoDB]
        A8[8. INSERT assessment to PostgreSQL]
        A9[9. UPDATE status=completed]
    end

    P1 --> A1 --> A2 --> A3 --> A4 --> A5 --> A6 --> A7 --> A8 --> A9
```

### 4.3 Arquitectura de Mensajeria

Configuracion de RabbitMQ para el sistema:

```mermaid
graph LR
    subgraph "Publishers"
        API_MOB[API Mobile]
    end

    subgraph "RabbitMQ"
        subgraph "Exchanges"
            EX_MAT[edugo.materials<br/>type: topic]
        end

        subgraph "Queues"
            Q_UPLOAD[material.uploaded]
            Q_REPROCESS[material.reprocess]
            Q_ATTEMPT[assessment.attempt]
        end
    end

    subgraph "Consumers"
        WORKER[Worker]
    end

    API_MOB -->|routing_key: material.uploaded| EX_MAT
    API_MOB -->|routing_key: material.reprocess| EX_MAT
    API_MOB -->|routing_key: assessment.attempt| EX_MAT

    EX_MAT -->|binding: material.uploaded| Q_UPLOAD
    EX_MAT -->|binding: material.reprocess| Q_REPROCESS
    EX_MAT -->|binding: assessment.attempt| Q_ATTEMPT

    Q_UPLOAD --> WORKER
    Q_REPROCESS --> WORKER
    Q_ATTEMPT --> WORKER
```

---

## 5. Dependencias

### 5.1 Dependencias entre Proyectos

Grafo de dependencias de modulos Go internos:

```mermaid
graph TB
    subgraph "Aplicaciones"
        API_ADMIN[edugo-api-administracion]
        API_MOBILE[edugo-api-mobile]
        WORKER[edugo-worker]
    end

    subgraph "edugo-infrastructure"
        INFRA_PG[postgres<br/>v0.13.0]
        INFRA_MDB[mongodb<br/>v0.11.0]
        INFRA_SCHEMAS[schemas<br/>v0.1.1]
        INFRA_MOCK[mock-generator<br/>v0.1.0]
    end

    subgraph "edugo-shared"
        SHARED_AUTH[auth<br/>v0.10.0]
        SHARED_BOOTSTRAP[bootstrap<br/>v0.9.0]
        SHARED_COMMON[common<br/>v0.9.0]
        SHARED_LIFECYCLE[lifecycle<br/>v0.9.0]
        SHARED_LOGGER[logger<br/>v0.9.0]
        SHARED_MIDDLEWARE[middleware/gin<br/>v0.7.0]
        SHARED_TESTING[testing<br/>v0.9.0]
        SHARED_DB_PG[database/postgres<br/>v0.9.0]
        SHARED_DB_MDB[database/mongodb<br/>v0.9.0]
        SHARED_MSG[messaging/rabbit<br/>v0.9.0]
    end

    API_ADMIN --> INFRA_PG
    API_ADMIN --> INFRA_MOCK
    API_ADMIN --> SHARED_AUTH
    API_ADMIN --> SHARED_BOOTSTRAP
    API_ADMIN --> SHARED_COMMON
    API_ADMIN --> SHARED_LIFECYCLE
    API_ADMIN --> SHARED_LOGGER
    API_ADMIN --> SHARED_MIDDLEWARE
    API_ADMIN --> SHARED_TESTING

    API_MOBILE --> INFRA_PG
    API_MOBILE --> INFRA_SCHEMAS
    API_MOBILE --> INFRA_MOCK
    API_MOBILE --> SHARED_AUTH
    API_MOBILE --> SHARED_BOOTSTRAP
    API_MOBILE --> SHARED_COMMON
    API_MOBILE --> SHARED_LIFECYCLE
    API_MOBILE --> SHARED_LOGGER
    API_MOBILE --> SHARED_MIDDLEWARE
    API_MOBILE --> SHARED_TESTING

    WORKER --> INFRA_MDB
    WORKER --> SHARED_BOOTSTRAP
    WORKER --> SHARED_COMMON
    WORKER --> SHARED_DB_PG
    WORKER --> SHARED_LIFECYCLE
    WORKER --> SHARED_LOGGER
    WORKER --> SHARED_TESTING
```

### 5.2 Stack Tecnologico

Tecnologias utilizadas en el ecosistema:

```mermaid
graph TB
    subgraph "Lenguaje y Runtime"
        GO[Go 1.25]
    end

    subgraph "Frameworks Web"
        GIN[Gin Web Framework]
        SWAGGER[Swaggo/Swagger]
    end

    subgraph "Bases de Datos"
        PG[PostgreSQL 16 Alpine]
        MDB[MongoDB 7.0]
        REDIS[Redis 7 Alpine]
    end

    subgraph "ORM y Drivers"
        GORM[GORM v1.25+]
        MONGO_DRV[mongo-driver v1.17]
        PGX[pgx v5.5]
    end

    subgraph "Mensajeria"
        RMQ[RabbitMQ 3.12]
        AMQP[amqp091-go]
    end

    subgraph "Cloud Services"
        S3[AWS S3]
        AWS_SDK[aws-sdk-go-v2]
    end

    subgraph "IA"
        OPENAI[OpenAI API]
        GPT4[GPT-4 / GPT-4-turbo]
    end

    subgraph "Autenticacion"
        JWT[JWT golang-jwt/v5]
        BCRYPT[bcrypt]
    end

    subgraph "Observabilidad"
        PROM[Prometheus]
        OTEL[OpenTelemetry]
        LOGRUS[Logrus / Zap]
    end

    subgraph "Testing"
        TESTIFY[Testify]
        TESTCONT[Testcontainers-go]
    end

    GO --> GIN
    GO --> GORM
    GO --> MONGO_DRV
    GORM --> PGX
    GIN --> SWAGGER
```

---

## 6. Infraestructura

### 6.1 Ambiente de Desarrollo

Docker Compose consolidado para desarrollo local:

```mermaid
graph TB
    subgraph "Docker Network: edugo-network"
        subgraph "Infraestructura Base"
            PG[edugo-postgres<br/>PostgreSQL 16<br/>:5432]
            MDB[edugo-mongodb<br/>MongoDB 7.0<br/>:27017]
            RMQ[edugo-rabbitmq<br/>RabbitMQ 3.12<br/>:5672/:15672]
        end

        subgraph "Aplicaciones"
            API_MOB[edugo-api-mobile<br/>:8081]
            API_ADM[edugo-api-administracion<br/>:8082]
            WRK[edugo-worker]
        end

        subgraph "Utilidades"
            MIG[edugo-migrator<br/>Migraciones Auto]
            REDIS[edugo-redis<br/>:6379<br/>Opcional]
        end
    end

    subgraph "Volumenes Persistentes"
        V_PG[postgres-data]
        V_MDB[mongodb-data]
        V_RMQ[rabbitmq-data]
        V_REDIS[redis-data]
    end

    API_MOB --> PG
    API_MOB --> MDB
    API_MOB --> RMQ

    API_ADM --> PG
    API_ADM --> MDB

    WRK --> PG
    WRK --> MDB
    WRK --> RMQ

    MIG --> PG
    MIG --> MDB

    PG --> V_PG
    MDB --> V_MDB
    RMQ --> V_RMQ
    REDIS --> V_REDIS
```

### 6.2 Perfiles de Docker Compose

El sistema utiliza perfiles para flexibilidad en el despliegue:

```mermaid
graph LR
    subgraph "Perfiles Disponibles"
        P_NONE[Sin profile<br/>Solo infraestructura]
        P_APPS[--profile apps<br/>+ API Mobile]
        P_ADMIN[--profile admin<br/>+ API Admin]
        P_WORKER[--profile worker<br/>+ Worker]
        P_FULL[--profile full<br/>Todo]
        P_REDIS[--profile with-redis<br/>+ Redis]
    end

    subgraph "Servicios"
        S_PG[PostgreSQL]
        S_MDB[MongoDB]
        S_RMQ[RabbitMQ]
        S_API_MOB[API Mobile]
        S_API_ADM[API Admin]
        S_WRK[Worker]
        S_REDIS[Redis]
    end

    P_NONE --> S_PG
    P_NONE --> S_MDB
    P_NONE --> S_RMQ

    P_APPS --> S_PG
    P_APPS --> S_MDB
    P_APPS --> S_RMQ
    P_APPS --> S_API_MOB

    P_ADMIN --> S_PG
    P_ADMIN --> S_MDB
    P_ADMIN --> S_RMQ
    P_ADMIN --> S_API_ADM

    P_WORKER --> S_PG
    P_WORKER --> S_MDB
    P_WORKER --> S_RMQ
    P_WORKER --> S_WRK

    P_FULL --> S_PG
    P_FULL --> S_MDB
    P_FULL --> S_RMQ
    P_FULL --> S_API_MOB
    P_FULL --> S_API_ADM
    P_FULL --> S_WRK
    P_FULL --> S_REDIS
```

---

## Variables de Entorno Principales

| Variable | Descripcion | Valor por Defecto |
|----------|-------------|-------------------|
| `POSTGRES_DB` | Nombre de la base de datos | `edugo` |
| `POSTGRES_USER` | Usuario PostgreSQL | `edugo` |
| `POSTGRES_PASSWORD` | Contrasena PostgreSQL | `edugo123` |
| `POSTGRES_PORT` | Puerto PostgreSQL | `5432` |
| `MONGO_USER` | Usuario MongoDB | `edugo` |
| `MONGO_PASSWORD` | Contrasena MongoDB | `edugo123` |
| `MONGO_PORT` | Puerto MongoDB | `27017` |
| `RABBITMQ_USER` | Usuario RabbitMQ | `edugo` |
| `RABBITMQ_PASSWORD` | Contrasena RabbitMQ | `edugo123` |
| `RABBITMQ_PORT` | Puerto AMQP | `5672` |
| `RABBITMQ_MGMT_PORT` | Puerto Management UI | `15672` |
| `API_MOBILE_PORT` | Puerto API Mobile | `8081` |
| `API_ADMIN_PORT` | Puerto API Admin | `8082` |
| `JWT_SECRET` | Secreto para JWT | `dev-secret-key-change-in-production` |
| `OPENAI_API_KEY` | API Key de OpenAI | (requerido) |
| `S3_BUCKET` | Bucket S3 para materiales | `edugo-materials-dev-local` |
| `S3_REGION` | Region AWS S3 | `us-east-1` |

---

## Comandos Utiles

```bash
# Iniciar solo infraestructura
docker-compose up -d

# Iniciar todo el sistema
docker-compose --profile full up -d

# Ver logs del worker
docker logs -f edugo-worker

# Acceder a la consola de RabbitMQ
open http://localhost:15672  # usuario: edugo, pass: edugo123

# Conectar a PostgreSQL
psql -h localhost -U edugo -d edugo

# Conectar a MongoDB
mongosh "mongodb://edugo:edugo123@localhost:27017/edugo?authSource=admin"
```

---

*Documento generado automaticamente el 2024*
