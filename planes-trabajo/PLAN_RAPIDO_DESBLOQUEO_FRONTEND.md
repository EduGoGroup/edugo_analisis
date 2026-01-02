# PLAN RÁPIDO: DESBLOQUEO FRONTEND

**Fecha de Creación:** 2025-12-24
**Objetivo:** Habilitar mínimo viable para que frontend pueda comenzar desarrollo
**Alcance:** Solo fixes bloqueantes - NO refactorizaciones
**Tiempo Estimado Total:** 1-2 días

---

## RESUMEN EJECUTIVO

Este plan implementa **únicamente los cambios críticos** necesarios para desbloquear el desarrollo del frontend, dejando mejoras de calidad y optimizaciones para un plan completo posterior.

### Proyectos Afectados
- ✅ edugo-api-administracion (CORS + PR dev→main)
- ✅ edugo-api-mobile (PR dev→main)
- ✅ edugo-worker (Fix DTO MaterialUploadedEvent)

### NO Incluye (Para Plan Completo)
- ❌ Crear DTOs compartidos en edugo-shared
- ❌ Implementar consumers faltantes (material.completed, assessment.generated)
- ❌ Implementar publishers faltantes (material_deleted, assessment_attempt, student_enrolled)
- ❌ Completar processors stub (AssessmentAttemptProcessor, StudentEnrolledProcessor)
- ❌ Activar OpenAI en Worker (quedará con fallback)
- ❌ Paginación en endpoints
- ❌ RBAC en API Admin
- ❌ Mejoras en health checks

---

## PROYECTO 1: edugo-api-administracion

### FASE 1: Configurar CORS

**Tiempo Estimado:** 30 minutos
**Responsable:** Backend Lead API Admin
**Rama de Trabajo:** `fix/cors-config-20251224`

#### Pre-requisitos
- [ ] Validar rama dev actualizada:
  ```bash
  cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion
  git fetch origin
  git status
  ```
- [ ] Crear rama de trabajo:
  ```bash
  git checkout dev
  git pull origin dev
  git checkout -b fix/cors-config-20251224
  ```

#### Pasos

1. **Abrir archivo `cmd/main.go`**
   - **Ruta completa:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion/cmd/main.go`
   - **Línea de inserción:** Después de línea 72 (`r := gin.Default()`)

2. **Agregar middleware CORS ANTES de definir rutas**
   - **Código exacto a insertar:**
   ```go
   // Configurar CORS
   r.Use(corsMiddleware())
   ```

3. **Agregar función corsMiddleware() al final del archivo**
   - **Posición:** Antes del cierre del archivo (después de todas las funciones)
   - **Código exacto:**
   ```go
   // corsMiddleware configura CORS para permitir requests del frontend
   func corsMiddleware() gin.HandlerFunc {
       return func(c *gin.Context) {
           // Leer origins permitidos desde env (fallback: localhost:3000)
           allowedOrigins := os.Getenv("CORS_ALLOWED_ORIGINS")
           if allowedOrigins == "" {
               allowedOrigins = "http://localhost:3000,http://localhost:3001"
           }

           c.Writer.Header().Set("Access-Control-Allow-Origin", allowedOrigins)
           c.Writer.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, PATCH, DELETE, OPTIONS")
           c.Writer.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
           c.Writer.Header().Set("Access-Control-Max-Age", "86400")

           // Manejar preflight requests
           if c.Request.Method == "OPTIONS" {
               c.AbortWithStatus(204)
               return
           }

           c.Next()
       }
   }
   ```

4. **Agregar import de `os` si no existe**
   - **Verificar línea 8:** Si no está `"os"`, agregar en bloque de imports

#### Post-requisitos
- [ ] Compilar para verificar sintaxis:
  ```bash
  cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion
  go build ./...
  ```
- [ ] Ejecutar lint:
  ```bash
  golangci-lint run --timeout 5m
  ```
- [ ] Ejecutar tests unitarios:
  ```bash
  go test ./... -short -v
  ```
- [ ] Probar CORS localmente:
  ```bash
  # Terminal 1: Levantar API
  go run cmd/main.go

  # Terminal 2: Probar con curl
  curl -X OPTIONS http://localhost:8081/v1/schools \
    -H "Origin: http://localhost:3000" \
    -H "Access-Control-Request-Method: POST" \
    -H "Access-Control-Request-Headers: Content-Type, Authorization" \
    -v

  # Verificar headers de respuesta:
  # - Access-Control-Allow-Origin: http://localhost:3000
  # - Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS
  # - Status: 204
  ```
- [ ] Commit atómico:
  ```bash
  git add cmd/main.go
  git commit -m "fix(cors): Agregar middleware CORS en main.go

  - Configura CORS para permitir requests desde frontend
  - Lee origins permitidos desde CORS_ALLOWED_ORIGINS env var
  - Fallback: localhost:3000,localhost:3001
  - Maneja preflight OPTIONS requests

  🤖 Generated with [Claude Code](https://claude.com/claude-code)

  Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>"
  ```
- [ ] Push a remoto:
  ```bash
  git push -u origin fix/cors-config-20251224
  ```
- [ ] Crear PR a dev:
  ```bash
  gh pr create --base dev --title "fix(cors): Agregar middleware CORS en main.go" \
    --body "## Resumen
  Agrega configuración CORS en API Admin para permitir requests del frontend.

  ## Cambios
  - ✅ Middleware CORS en main.go
  - ✅ Configuración via env var CORS_ALLOWED_ORIGINS
  - ✅ Manejo de preflight OPTIONS

  ## Testing
  - ✅ Compilación exitosa
  - ✅ Lint pasando
  - ✅ Tests unitarios OK
  - ✅ Prueba manual con curl

  🤖 Generated with [Claude Code](https://claude.com/claude-code)"
  ```

---

### FASE 2: Merge dev → main

**Tiempo Estimado:** 1 día (incluye code review y testing)
**Responsable:** DevOps Lead + Backend Lead
**Rama Destino:** `main`

#### Pre-requisitos
- [ ] Verificar FASE 1 completada y mergeada a dev
- [ ] Validar diferencias dev vs main:
  ```bash
  cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion
  git fetch origin
  git log origin/main..origin/dev --oneline | wc -l
  # Debe mostrar cantidad de commits adelantados
  ```
- [ ] Crear rama de trabajo desde dev:
  ```bash
  git checkout dev
  git pull origin dev
  git checkout -b merge/dev-to-main-20251224
  ```

#### Pasos

1. **Code Review de cambios dev → main**
   - **Acción:** Revisar commits que NO están en main
   ```bash
   git log origin/main..origin/dev --oneline --graph
   ```
   - **Verificar:**
     - ✅ Endpoints de Subjects (5 endpoints)
     - ✅ Endpoints de Guardian Relations (6 endpoints)
     - ✅ Swagger actualizado
     - ✅ Tests correspondientes
     - ❌ NO debe haber trabajo en progreso (WIP)
     - ❌ NO debe haber código comentado masivamente

2. **Ejecutar test suite completo**
   ```bash
   cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion

   # Tests unitarios
   go test ./... -short -v -race -coverprofile=coverage.out

   # Tests de integración (requiere Docker)
   go test ./... -tags=integration -v
   ```

3. **Verificar Swagger actualizado**
   ```bash
   # Regenerar Swagger
   swag init -g cmd/main.go --output docs

   # Verificar que no hay cambios (debe estar actualizado)
   git diff docs/
   ```

4. **Merge dev → main**
   ```bash
   git checkout main
   git pull origin main
   git merge --no-ff dev -m "chore(release): Merge dev to main - Subjects y Guardian Relations

  Endpoints agregados:
  - POST /v1/subjects
  - GET /v1/subjects
  - GET /v1/subjects/:id
  - PATCH /v1/subjects/:id
  - DELETE /v1/subjects/:id
  - POST /v1/guardian-relations
  - GET /v1/guardian-relations/:id
  - PUT /v1/guardian-relations/:id
  - DELETE /v1/guardian-relations/:id
  - GET /v1/guardians/:guardian_id/relations
  - GET /v1/students/:student_id/guardians

  🤖 Generated with [Claude Code](https://claude.com/claude-code)

  Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>"
   ```

5. **Resolver conflictos si existen**
   - Si hay conflictos, resolver manualmente
   - Priorizar código de dev (es más reciente)
   - Ejecutar tests después de resolver conflictos

#### Post-requisitos
- [ ] Compilar versión final:
  ```bash
  go build ./...
  ```
- [ ] Ejecutar lint:
  ```bash
  golangci-lint run --timeout 5m
  ```
- [ ] Tests completos:
  ```bash
  go test ./... -v
  ```
- [ ] Push a main:
  ```bash
  git push origin main
  ```
- [ ] Crear tag de versión:
  ```bash
  git tag -a v1.1.0 -m "Release v1.1.0 - Subjects y Guardian Relations"
  git push origin v1.1.0
  ```
- [ ] Smoke testing en ambiente staging:
  ```bash
  # Después del deploy a staging
  curl -X POST http://staging-api-admin:8081/v1/subjects \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"name":"Matemáticas","description":"Test"}'

  # Verificar respuesta 201 Created
  ```
- [ ] Actualizar documentación en README si aplica
- [ ] Notificar a equipo de frontend que endpoints están disponibles en main

---

## PROYECTO 2: edugo-api-mobile

### FASE 1: Merge dev → main

**Tiempo Estimado:** 4 horas
**Responsable:** Backend Lead API Mobile
**Rama de Trabajo:** `merge/dev-to-main-20251224`

#### Pre-requisitos
- [ ] Validar rama dev actualizada:
  ```bash
  cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile
  git fetch origin
  git status
  ```
- [ ] Verificar diferencias:
  ```bash
  git log origin/main..origin/dev --oneline | wc -l
  # Ver cantidad de commits
  git diff origin/main origin/dev --stat
  # Ver archivos modificados
  ```
- [ ] Crear rama de trabajo:
  ```bash
  git checkout dev
  git pull origin dev
  git checkout -b merge/dev-to-main-20251224
  ```

#### Pasos

1. **Revisar cambios principales dev → main**
   ```bash
   git log origin/main..origin/dev --oneline --graph --all
   ```
   - **Verificar:**
     - ✅ Endpoint PUT /v1/materials/:id (nuevo)
     - ✅ Cambios en queries de progress
     - ✅ Swagger actualizado
     - ❌ NO debe haber dependencias rotas

2. **Ejecutar tests**
   ```bash
   cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile

   # Tests unitarios
   go test ./... -short -v -race

   # Tests de integración
   go test ./... -tags=integration -v
   ```

3. **Merge dev → main**
   ```bash
   git checkout main
   git pull origin main
   git merge --no-ff dev -m "chore(release): Merge dev to main - Material updates

  Cambios principales:
  - Endpoint PUT /v1/materials/:id para actualizar materiales
  - Fix queries progress: material_progress → progress
  - Swagger actualizado

  🤖 Generated with [Claude Code](https://claude.com/claude-code)

  Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>"
   ```

#### Post-requisitos
- [ ] Compilar:
  ```bash
  go build ./...
  ```
- [ ] Lint:
  ```bash
  golangci-lint run --timeout 5m
  ```
- [ ] Tests finales:
  ```bash
  go test ./... -v
  ```
- [ ] Push a main:
  ```bash
  git push origin main
  ```
- [ ] Tag de versión:
  ```bash
  git tag -a v0.16.0 -m "Release v0.16.0 - Material updates"
  git push origin v0.16.0
  ```
- [ ] Smoke testing:
  ```bash
  # Después del deploy
  curl -X GET http://staging-api-mobile:8080/health?detail=1
  # Verificar status: healthy
  ```

---

## PROYECTO 3: edugo-worker

### FASE 1: Adaptar DTO MaterialUploadedEvent

**Tiempo Estimado:** 2 horas
**Responsable:** Backend Lead Worker
**Rama de Trabajo:** `fix/material-uploaded-dto-20251224`

**IMPORTANTE:** Este fix es **MÍNIMO VIABLE**. Solo mapea los campos necesarios para que el worker pueda procesar el evento de API Mobile. NO incluye crear DTOs compartidos en edugo-shared.

#### Pre-requisitos
- [ ] Validar rama dev actualizada:
  ```bash
  cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker
  git fetch origin
  git status
  ```
- [ ] Crear rama de trabajo:
  ```bash
  git checkout dev
  git pull origin dev
  git checkout -b fix/material-uploaded-dto-20251224
  ```

#### Pasos

1. **Modificar archivo `internal/application/dto/event_dto.go`**
   - **Ruta completa:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/internal/application/dto/event_dto.go`
   - **Líneas a modificar:** 5-13

2. **Reemplazar struct MaterialUploadedEvent**
   - **Código ACTUAL (líneas 5-13):**
   ```go
   type MaterialUploadedEvent struct {
       EventType         string    `json:"event_type"`
       MaterialID        string    `json:"material_id"`
       AuthorID          string    `json:"author_id"`
       S3Key             string    `json:"s3_key"`
       PreferredLanguage string    `json:"preferred_language"`
       Timestamp         time.Time `json:"timestamp"`
   }
   ```

   - **Código NUEVO:**
   ```go
   // MaterialUploadedEvent representa el evento recibido de API Mobile
   // NOTA: Este es un mapeo TEMPORAL hasta implementar DTOs compartidos en edugo-shared
   type MaterialUploadedEvent struct {
       // Campos del envelope (API Mobile usa Event wrapper)
       EventID      string    `json:"event_id"`
       EventType    string    `json:"event_type"`
       EventVersion string    `json:"event_version"`
       Timestamp    time.Time `json:"timestamp"`

       // Payload anidado
       Payload MaterialUploadedPayload `json:"payload"`
   }

   // MaterialUploadedPayload contiene los datos del material
   type MaterialUploadedPayload struct {
       MaterialID    string                 `json:"material_id"`
       SchoolID      string                 `json:"school_id"`
       TeacherID     string                 `json:"teacher_id"`    // API usa teacher_id
       FileURL       string                 `json:"file_url"`       // API usa file_url (no s3_key)
       FileSizeBytes int64                  `json:"file_size_bytes"`
       FileType      string                 `json:"file_type"`
       Metadata      map[string]interface{} `json:"metadata,omitempty"`
   }

   // GetMaterialID helper para acceder al material ID desde el payload
   func (e MaterialUploadedEvent) GetMaterialID() string {
       return e.Payload.MaterialID
   }

   // GetS3Key helper para compatibilidad (FileURL es la URL completa, extraer key si es necesario)
   func (e MaterialUploadedEvent) GetS3Key() string {
       // Por ahora retornar FileURL directamente
       // TODO: Si es necesario extraer solo la key del path S3, implementar lógica aquí
       return e.Payload.FileURL
   }

   // GetAuthorID helper para mapear TeacherID → AuthorID
   func (e MaterialUploadedEvent) GetAuthorID() string {
       return e.Payload.TeacherID
   }
   ```

3. **Modificar processor para usar nueva estructura**
   - **Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/internal/application/processor/material_uploaded_processor.go`
   - **Buscar línea que accede a `event.MaterialID`** (aproximadamente línea 50-60)
   - **Cambiar:** `event.MaterialID` → `event.GetMaterialID()`
   - **Cambiar:** `event.S3Key` → `event.GetS3Key()`
   - **Cambiar:** `event.AuthorID` → `event.GetAuthorID()`

4. **Buscar TODAS las referencias en el processor**
   ```bash
   cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker
   grep -n "event\.MaterialID\|event\.S3Key\|event\.AuthorID" internal/application/processor/material_uploaded_processor.go
   ```
   - Reemplazar cada ocurrencia con los helpers correspondientes

5. **Actualizar consumer para event type correcto**
   - **Archivo:** `/Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker/internal/infrastructure/messaging/consumer.go`
   - **Buscar:** Registro de consumer para `material_uploaded`
   - **Verificar que escuche:** `material.uploaded` (con punto, no underscore)
   - Si está hardcoded con underscore, cambiar a `material.uploaded`

#### Post-requisitos
- [ ] Compilar:
  ```bash
  go build ./...
  ```
- [ ] Lint:
  ```bash
  golangci-lint run --timeout 5m
  ```
- [ ] Tests unitarios:
  ```bash
  go test ./internal/application/dto -v
  go test ./internal/application/processor -v -short
  ```
- [ ] Test de integración simulado:
  ```bash
  # Crear archivo test_event.json con estructura de API Mobile
  cat > test_event.json <<EOF
  {
    "event_id": "123e4567-e89b-12d3-a456-426614174000",
    "event_type": "material.uploaded",
    "event_version": "1.0",
    "timestamp": "2025-12-24T10:00:00Z",
    "payload": {
      "material_id": "mat-123",
      "school_id": "school-456",
      "teacher_id": "teacher-789",
      "file_url": "s3://edugo-materials/test.pdf",
      "file_size_bytes": 1024,
      "file_type": "application/pdf",
      "metadata": {}
    }
  }
  EOF

  # Verificar deserialización
  go test ./internal/application/dto -v -run TestMaterialUploadedEvent
  ```
- [ ] Commit atómico:
  ```bash
  git add internal/application/dto/event_dto.go
  git add internal/application/processor/material_uploaded_processor.go
  git commit -m "fix(dto): Adaptar MaterialUploadedEvent para compatibilidad con API Mobile

  - Cambia estructura plana a envelope + payload
  - Mapea TeacherID → AuthorID con helper
  - Mapea FileURL → S3Key con helper
  - Agrega helpers GetMaterialID(), GetS3Key(), GetAuthorID()
  - Actualiza processor para usar helpers

  NOTA: Este es un fix TEMPORAL. Pendiente migrar a DTOs compartidos en edugo-shared.

  Relacionado: #ISSUE_CONTRATOS_DTO

  🤖 Generated with [Claude Code](https://claude.com/claude-code)

  Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>"
  ```
- [ ] Push:
  ```bash
  git push -u origin fix/material-uploaded-dto-20251224
  ```
- [ ] Crear PR:
  ```bash
  gh pr create --base dev --title "fix(dto): Adaptar MaterialUploadedEvent para API Mobile" \
    --body "## Problema
  Worker no puede deserializar eventos de API Mobile porque:
  - API usa envelope (Event wrapper) con payload anidado
  - Worker esperaba estructura plana
  - Nombres de campos diferentes (TeacherID vs AuthorID, FileURL vs S3Key)

  ## Solución TEMPORAL
  - ✅ Adapta DTO para recibir envelope + payload
  - ✅ Agrega helpers para mapear campos
  - ✅ Actualiza processor para usar helpers
  - ⚠️ NO crea DTOs compartidos (queda para plan completo)

  ## Testing
  - ✅ Compilación exitosa
  - ✅ Tests unitarios OK
  - ✅ Test de deserialización con JSON de ejemplo

  ## Próximos Pasos
  - [ ] Crear DTOs compartidos en edugo-shared
  - [ ] Migrar ambos proyectos a DTOs compartidos
  - [ ] Implementar validación de esquemas

  🤖 Generated with [Claude Code](https://claude.com/claude-code)"
  ```

---

## VALIDACIÓN FINAL DEL PLAN

### Checklist de Completitud

#### API Admin ✅
- [x] CORS configurado en main.go
- [x] PR dev→main creada y mergeada
- [x] Endpoints Subjects disponibles en main
- [x] Endpoints Guardian Relations disponibles en main
- [x] Swagger actualizado en main
- [x] Tag de versión creado

#### API Mobile ✅
- [x] PR dev→main creada y mergeada
- [x] Endpoint PUT /materials/:id disponible en main
- [x] Tag de versión creado

#### Worker ✅
- [x] DTO MaterialUploadedEvent adaptado
- [x] Processor actualizado para usar helpers
- [x] Tests pasando
- [x] PR creada a dev

### Tests de Integración End-to-End

**Después de completar todos los pasos:**

1. **Test: Frontend → API Admin (Login)**
   ```bash
   # Desde navegador o curl
   curl -X POST http://localhost:8081/v1/auth/login \
     -H "Content-Type: application/json" \
     -H "Origin: http://localhost:3000" \
     -d '{"email":"admin@test.com","password":"password123"}' \
     -v

   # Verificar:
   # - Status: 200 OK
   # - Header Access-Control-Allow-Origin presente
   # - Response: { access_token, refresh_token, ... }
   ```

2. **Test: Frontend → API Mobile (Materials)**
   ```bash
   curl -X GET http://localhost:8080/v1/materials \
     -H "Authorization: Bearer $TOKEN" \
     -H "Origin: http://localhost:3000" \
     -v

   # Verificar:
   # - Status: 200 OK
   # - Response: array de materiales
   ```

3. **Test: API Mobile → Worker (Material Upload)**
   ```bash
   # 1. Crear material
   MATERIAL_ID=$(curl -X POST http://localhost:8080/v1/materials \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"title":"Test Material","description":"Test"}' \
     | jq -r '.id')

   # 2. Simular upload complete (dispara evento a Worker)
   curl -X POST http://localhost:8080/v1/materials/$MATERIAL_ID/upload-complete \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{
       "file_url": "s3://bucket/test.pdf",
       "file_type": "application/pdf",
       "file_size_bytes": 1024
     }'

   # 3. Verificar logs del Worker
   # Debe mostrar: "Processing material uploaded event" sin errores de deserialización

   # 4. Polling status del material
   for i in {1..12}; do
     STATUS=$(curl -s http://localhost:8080/v1/materials/$MATERIAL_ID \
       -H "Authorization: Bearer $TOKEN" | jq -r '.status')
     echo "Attempt $i: Status = $STATUS"
     if [ "$STATUS" = "ready" ] || [ "$STATUS" = "failed" ]; then
       break
     fi
     sleep 5
   done

   # Verificar:
   # - Status final: "ready" o "processing" (si OpenAI no está activo)
   # - Sin errores en logs del Worker
   ```

### Criterios de Aceptación

✅ **El plan se considera EXITOSO si:**
1. Frontend puede hacer login en API Admin sin errores CORS
2. Frontend puede listar materiales de API Mobile
3. Worker puede recibir y deserializar eventos de API Mobile sin errores
4. Todas las PRs están mergeadas a sus respectivas ramas destino
5. Todos los tests automáticos pasan

⚠️ **Limitaciones conocidas (esperadas):**
- Worker usará fallback de OpenAI (calidad reducida en resúmenes/quizzes)
- No hay DTOs compartidos (se implementará en plan completo)
- No hay RBAC en API Admin (se implementará en plan completo)
- No hay paginación en listas (se implementará en plan completo)

---

## DOCUMENTACIÓN DE CAMBIOS

### Variables de Entorno Nuevas

**API Admin (`edugo-api-administracion`):**
```bash
# .env o secrets
CORS_ALLOWED_ORIGINS=http://localhost:3000,http://localhost:3001,https://app.edugo.com
```

### Archivos Modificados

**edugo-api-administracion:**
- ✅ `cmd/main.go` - Agregar middleware CORS

**edugo-api-mobile:**
- ℹ️ Sin cambios de código (solo merge)

**edugo-worker:**
- ✅ `internal/application/dto/event_dto.go` - Adaptar MaterialUploadedEvent
- ✅ `internal/application/processor/material_uploaded_processor.go` - Usar helpers

### Deuda Técnica Creada

**IMPORTANTE:** Este plan crea DEUDA TÉCNICA que debe resolverse en el plan completo:

1. **DTOs Duplicados**
   - API Mobile y Worker tienen definiciones separadas del mismo evento
   - **Solución futura:** Migrar a edugo-shared/events/

2. **Helpers Temporales**
   - GetMaterialID(), GetS3Key(), GetAuthorID() son wrappers temporales
   - **Solución futura:** Eliminar cuando se usen DTOs compartidos

3. **Sin Validación de Esquemas**
   - No hay validación JSON Schema de eventos
   - **Solución futura:** Implementar contract testing

4. **Eventos Huérfanos**
   - material.completed, assessment.generated se publican sin consumidor
   - **Solución futura:** Implementar consumers correspondientes

---

## PRÓXIMOS PASOS (Plan Completo)

Este plan rápido desbloquea el frontend, pero debe seguirse con:

1. **Sprint 1: DTOs Compartidos**
   - Crear `edugo-shared/events/` con todos los DTOs
   - Migrar API Mobile a usar DTOs compartidos
   - Migrar Worker a usar DTOs compartidos
   - Eliminar helpers temporales

2. **Sprint 2: Completar Eventos**
   - Implementar consumer para material.completed
   - Implementar consumer para assessment.generated
   - Implementar publisher para material_deleted
   - Implementar publisher para assessment_attempt
   - Implementar publisher para student_enrolled

3. **Sprint 3: Activar OpenAI**
   - Configurar API key de OpenAI en Worker
   - Activar generación real de resúmenes
   - Activar generación real de quizzes
   - Tests de calidad de contenido generado

4. **Sprint 4: Seguridad y Rendimiento**
   - Implementar RBAC en API Admin
   - Implementar paginación en listas
   - Implementar rate limiting
   - Mejorar health checks

---

## CONTACTOS Y RESPONSABLES

| Área | Responsable | Proyecto |
|------|-------------|----------|
| CORS + Merge API Admin | Backend Lead Admin | edugo-api-administracion |
| Merge API Mobile | Backend Lead Mobile | edugo-api-mobile |
| Fix DTO Worker | Backend Lead Worker | edugo-worker |
| Code Review | Tech Lead | Todos |
| Deploy Staging | DevOps Lead | Todos |
| Smoke Testing | QA Lead | Todos |

---

## ANEXOS

### A. Comandos Útiles

**Ver diferencias entre ramas:**
```bash
git diff origin/main origin/dev --stat
git log origin/main..origin/dev --oneline --graph
```

**Regenerar Swagger:**
```bash
swag init -g cmd/main.go --output docs
```

**Ejecutar tests con coverage:**
```bash
go test ./... -coverprofile=coverage.out
go tool cover -html=coverage.out
```

**Ver logs de RabbitMQ:**
```bash
docker logs edugo-rabbitmq -f --tail 100
```

**Ver logs de Worker:**
```bash
docker logs edugo-worker -f --tail 100 | grep "material.uploaded"
```

### B. Referencias

- **CONSOLIDADO_ECOSISTEMA.md** - Estado general del ecosistema
- **SALUD_CONTRATOS.md** - Análisis de DTOs y eventos
- **ENDPOINTS_VIABLES_FRONTEND.md** - Endpoints disponibles para frontend
- **Swagger API Admin:** http://localhost:8081/swagger/index.html
- **Swagger API Mobile:** http://localhost:8080/swagger/index.html

---

**FIN DEL PLAN RÁPIDO**

*Generado: 2025-12-24*
*Versión: 1.0*
*Tipo: Plan de Desbloqueo Mínimo Viable*
