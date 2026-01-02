# RFC-021: Subida de Material PDF

## Metadata
- **ID:** RFC-021
- **Proceso:** Materiales
- **Subproceso:** Upload con Procesamiento Asíncrono
- **Prioridad:** Alta
- **Dependencias:** RFC-001 (Autenticación), RFC-020 (Listado Materiales)
- **Estado API:** ✅ Listo (incluye Worker)

## Descripción
Proceso completo de subida de un material educativo en formato PDF, que incluye:
1. Creación de metadata en PostgreSQL
2. Generación de URL presignada de S3
3. Upload directo del PDF a S3 (sin pasar por backend)
4. Notificación de completitud que dispara el Worker
5. Procesamiento asíncrono con IA (extracción PDF + resumen + quiz)
6. Actualización de estado a "ready"

Este es un flujo **asíncrono** crítico que involucra múltiples sistemas: API Mobile, S3, RabbitMQ y Worker.

## Flujo de Usuario (UX)

### Como Docente
1. **Inicio**: Click en "Subir Material"
2. **Formulario**: Completa metadata
   - Título (obligatorio)
   - Descripción (opcional)
   - Materia/Subject (opcional)
   - Grado (opcional)
3. **Selección**: Elige archivo PDF desde su dispositivo
4. **Validación Cliente**:
   - Formato PDF
   - Tamaño máximo 100MB
   - Preview de metadata
5. **Upload**:
   - Barra de progreso (0-100%)
   - Estimación de tiempo restante
   - Opción de cancelar
6. **Confirmación**:
   - Material subido exitosamente
   - Estado: "Procesando con IA"
   - Estimación: "Listo en ~2 minutos"
7. **Espera Activa**: (Opcional)
   - Polling cada 5s para actualizar estado
   - Notificación push cuando esté listo
   - Puede navegar a otra página (proceso en background)

### Estados Visuales Durante el Flujo
```
📝 Completando metadata
  ↓
📤 Subiendo archivo (0% → 100%)
  ↓
⏳ Procesando con IA (estimado 2-5 min)
  ↓
✅ Material listo para usar
```

### Manejo de Errores Usuario
- **PDF muy grande**: "El archivo supera los 100MB permitidos"
- **PDF corrupto**: "No se pudo leer el archivo PDF. Verifica que no esté dañado"
- **Sin conexión**: "Parece que perdiste conexión. El upload se pausará automáticamente"
- **Timeout S3**: "El upload está tardando mucho. ¿Deseas reintentar?"
- **Error Worker**: "Hubo un problema procesando el PDF. Puedes reprocesarlo más tarde"

## Flujo de Datos (Técnico)

### Diagrama de Secuencia (Flujo Completo con Worker)

```
Cliente         API Mobile      PostgreSQL      S3          RabbitMQ        Worker          MongoDB
  |                  |               |            |             |              |               |
  |--- POST /materials ---------->|               |            |             |              |               |
  |  (metadata)      |               |            |             |              |               |
  |                  |--INSERT------>|            |             |              |               |
  |                  |  material     |            |             |              |               |
  |                  |  status='uploaded'         |             |              |               |
  |                  |<--material_id-|            |             |              |               |
  |<--201 Created----|               |            |             |              |               |
  |  {id, status}    |               |            |             |              |               |
  |                  |               |            |             |              |               |
  |--- POST /materials/:id/upload-url ---------->|            |             |              |               |
  |                  |               |            |             |              |               |
  |                  |---GeneratePresignedURL---->|            |             |              |               |
  |                  |               |            |             |              |               |
  |                  |<--presigned_url, expires---|            |             |              |               |
  |<--200 OK---------|               |            |             |              |               |
  |  {upload_url}    |               |            |             |              |               |
  |                  |               |            |             |              |               |
  |===== PUT presigned_url (archivo PDF) =======>|             |              |               |
  |                  |               |            |--store----> |              |               |
  |<======= 200 OK (S3 responde directamente) ===|             |              |               |
  |                  |               |            |             |              |               |
  |--- POST /materials/:id/upload-complete ----->|            |             |              |               |
  |  {file_url, size}|               |            |             |              |               |
  |                  |--UPDATE------>|            |             |              |               |
  |                  |  status='processing'       |             |              |               |
  |                  |  file_url, size            |             |              |               |
  |                  |<--OK----------|            |             |              |               |
  |                  |                            |             |              |               |
  |                  |---PUBLISH EVENT----------->|------------>|              |               |
  |                  |  material.uploaded         |             |              |               |
  |<--204 No Content-|                            |             |              |               |
  |                  |                            |             |              |               |
  |                  |                            |             |   <--consume event           |
  |                  |                            |             |              |               |
  |                  |                            |             |              |---Download PDF from S3-->
  |                  |                            |             |              |<--PDF bytes---|
  |                  |                            |             |              |               |
  |                  |                            |             |              |--Extract Text (pdfcpu)
  |                  |                            |             |              |               |
  |                  |                            |             |              |--Generate Summary (OpenAI/Fallback)
  |                  |                            |             |              |               |
  |                  |                            |             |              |--Generate Quiz (OpenAI/Fallback)
  |                  |                            |             |              |               |
  |                  |                            |             |              |--INSERT------>|
  |                  |                            |             |              |  summary      |
  |                  |                            |             |              |<--OK----------|
  |                  |                            |             |              |               |
  |                  |                            |             |              |--INSERT------>|
  |                  |                            |             |              |  assessment   |
  |                  |                            |             |              |<--OK----------|
  |                  |                            |             |              |               |
  |                  |<--UPDATE (status='ready')--|<-----------|              |               |
  |                  |                            |             |              |               |
  |--- GET /materials/:id (polling) ------------->|            |             |              |               |
  |<--200 OK---------|               |            |             |              |               |
  |  {status:'ready'}|               |            |             |              |               |
```

### Endpoints Involucrados

#### 1. Crear Material (Metadata)
```http
POST /v1/materials
Authorization: Bearer {jwt_token}
Content-Type: application/json

{
  "title": "Álgebra Lineal - Módulo 1",
  "description": "Introducción a vectores y matrices",
  "subject": "Mathematics",
  "grade": "10"
}
```

**Response 201 Created:**
```typescript
{
  id: "550e8400-e29b-41d4-a716-446655440000",
  school_id: "660e8400-e29b-41d4-a716-446655440111",
  uploaded_by_teacher_id: "770e8400-e29b-41d4-a716-446655440222",
  academic_unit_id: null,
  title: "Álgebra Lineal - Módulo 1",
  description: "Introducción a vectores y matrices",
  subject: "Mathematics",
  grade: "10",
  file_url: null,
  file_type: "",
  file_size_bytes: 0,
  status: "uploaded",
  is_public: false,
  processing_started_at: null,
  processing_completed_at: null,
  created_at: "2024-12-24T10:00:00Z",
  updated_at: "2024-12-24T10:00:00Z",
  deleted_at: null
}
```

#### 2. Generar URL Presignada S3
```http
POST /v1/materials/:id/upload-url
Authorization: Bearer {jwt_token}
Content-Type: application/json

{
  "file_name": "algebra-lineal.pdf",
  "content_type": "application/pdf"
}
```

**Response 200 OK:**
```typescript
{
  upload_url: "https://s3.amazonaws.com/edugo-materials/...",
  file_url: "materials/uuid/algebra-lineal.pdf",
  expires_in: 900  // 15 minutos
}
```

#### 3. Upload a S3 (Directo, sin pasar por backend)
```http
PUT {upload_url}
Content-Type: application/pdf
Content-Length: {file_size}

[Binary PDF data]
```

**Response 200 OK (de S3):**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<CompleteMultipartUploadResult>
  <ETag>"abc123def456"</ETag>
</CompleteMultipartUploadResult>
```

#### 4. Notificar Completitud (Dispara Worker)
```http
POST /v1/materials/:id/upload-complete
Authorization: Bearer {jwt_token}
Content-Type: application/json

{
  "file_url": "materials/uuid/algebra-lineal.pdf",
  "file_type": "application/pdf",
  "file_size_bytes": 2048576
}
```

**Response 204 No Content**

**Side Effect:**
- Material actualizado a `status='processing'`
- Evento `material.uploaded` publicado en RabbitMQ
- Worker comienza procesamiento

### Request/Response (TypeScript)

```typescript
// 1. CreateMaterialRequest
interface CreateMaterialRequest {
  title: string;          // min=3, max=200
  description?: string;   // max=1000
  subject?: string;
  grade?: string;
}

// 2. GenerateUploadURLRequest
interface GenerateUploadURLRequest {
  file_name: string;      // Sin path traversal (.., /, \)
  content_type: string;   // "application/pdf"
}

interface GenerateUploadURLResponse {
  upload_url: string;     // Válida por 15 minutos
  file_url: string;       // Path relativo en S3
  expires_in: number;     // Segundos
}

// 3. UploadCompleteRequest
interface UploadCompleteRequest {
  file_url: string;
  file_type: string;
  file_size_bytes: number;
}

// Evento RabbitMQ (emitido por API Mobile)
interface MaterialUploadedEvent {
  event_type: "material_uploaded";
  material_id: string;
  author_id: string;
  s3_key: string;
  preferred_language: "es" | "en";
  timestamp: string;  // ISO8601
}
```

## Estados y Transiciones

### Estados del Material Durante Upload

```
📝 null (inicial)
  ↓
uploaded (metadata creada, sin archivo)
  ↓
uploaded (con file_url, esperando notificación)
  ↓
processing (Worker procesando)
  ↓ (éxito)
ready (resumen + quiz listos)
  ↓ (error)
failed (error en procesamiento)
```

### Transiciones Detalladas

**null → uploaded (POST /materials)**
- Trigger: Docente completa formulario
- Acción: INSERT en PostgreSQL
- Campos: metadata básica, `status='uploaded'`, `file_url=null`

**uploaded → uploaded con file_url (S3 upload + POST upload-complete)**
- Trigger: Upload exitoso a S3 + notificación
- Acción: UPDATE `file_url`, `file_type`, `file_size_bytes`

**uploaded → processing (Worker consume evento)**
- Trigger: Worker recibe `material.uploaded` de RabbitMQ
- Acción: UPDATE `status='processing'`, `processing_started_at=NOW()`

**processing → ready (Worker completa)**
- Trigger: Worker finaliza generación de resumen + quiz
- Acción: UPDATE `status='ready'`, `processing_completed_at=NOW()`
- Side Effect: Inserts en MongoDB (`material_summaries`, `material_assessment_worker`)

**processing → failed (Worker error)**
- Trigger: PDF corrupto, timeout OpenAI, error extracción
- Acción: UPDATE `status='failed'`
- Log: Error detallado en logs del Worker

### Tiempos Estimados
- **Upload S3**: 5-30 segundos (depende de tamaño y conexión)
- **Worker Processing**:
  - PDF 10 páginas: ~30-60 segundos
  - PDF 50 páginas: ~2-5 minutos
  - PDF 100 páginas: ~5-10 minutos

## Manejo de Errores

### Error 1: Validación Cliente (Pre-Upload)
```typescript
function validatePDF(file: File): ValidationResult {
  const errors: string[] = [];

  // Validar tipo
  if (file.type !== 'application/pdf') {
    errors.push('Solo se permiten archivos PDF');
  }

  // Validar tamaño (100MB)
  const MAX_SIZE = 100 * 1024 * 1024;
  if (file.size > MAX_SIZE) {
    errors.push(`El archivo supera los 100MB permitidos (${formatBytes(file.size)})`);
  }

  // Validar tamaño mínimo (1KB)
  if (file.size < 1024) {
    errors.push('El archivo está vacío o es muy pequeño');
  }

  return {
    valid: errors.length === 0,
    errors
  };
}
```

### Error 2: Timeout en S3 Upload
```typescript
async function uploadToS3WithRetry(
  presignedUrl: string,
  file: File,
  maxRetries = 3
): Promise<void> {
  let attempt = 0;

  while (attempt < maxRetries) {
    try {
      await axios.put(presignedUrl, file, {
        headers: { 'Content-Type': 'application/pdf' },
        timeout: 120_000, // 2 minutos
        onUploadProgress: (progressEvent) => {
          const percentCompleted = Math.round(
            (progressEvent.loaded * 100) / progressEvent.total
          );
          updateProgress(percentCompleted);
        }
      });

      return; // Éxito

    } catch (error) {
      attempt++;

      if (axios.isCancel(error)) {
        throw new Error('Upload cancelado por el usuario');
      }

      if (error.code === 'ECONNABORTED') {
        if (attempt === maxRetries) {
          throw new Error('Timeout: El upload tardó demasiado. Verifica tu conexión.');
        }
        // Esperar antes de reintentar (exponential backoff)
        await sleep(Math.pow(2, attempt) * 1000);
        continue;
      }

      throw error;
    }
  }
}
```

### Error 3: Worker Falla al Procesar
```typescript
// Polling para detectar estado 'failed'
async function waitForProcessing(materialId: string): Promise<Material> {
  const MAX_ATTEMPTS = 60; // 5 minutos con polling cada 5s
  const INTERVAL = 5000;

  for (let i = 0; i < MAX_ATTEMPTS; i++) {
    const material = await api.get(`/v1/materials/${materialId}`);

    if (material.status === 'ready') {
      return material; // Éxito
    }

    if (material.status === 'failed') {
      throw new ProcessingError(
        'Error al procesar el PDF. Posibles causas:\n' +
        '- PDF escaneado sin texto\n' +
        '- PDF corrupto\n' +
        '- Error de servicio OpenAI'
      );
    }

    if (material.status === 'processing') {
      // Actualizar UI con tiempo transcurrido
      const elapsed = i * (INTERVAL / 1000);
      updateStatus(`Procesando... (${elapsed}s)`);
      await sleep(INTERVAL);
      continue;
    }
  }

  throw new Error('Timeout: El procesamiento tardó más de 5 minutos');
}

class ProcessingError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'ProcessingError';
  }
}
```

### Error 4: URL Presignada Expirada
```typescript
// Regenerar URL si han pasado >14 minutos desde que se generó
async function ensureValidUploadURL(
  materialId: string,
  generatedAt: number
): Promise<string> {
  const now = Date.now();
  const elapsed = (now - generatedAt) / 1000; // segundos

  if (elapsed > 14 * 60) { // 14 minutos
    // Regenerar
    const response = await api.post(
      `/v1/materials/${materialId}/upload-url`,
      {
        file_name: originalFileName,
        content_type: 'application/pdf'
      }
    );
    return response.upload_url;
  }

  return cachedUploadURL;
}
```

## Consideraciones de UX

### Feedback Visual Continuo
```typescript
// Estados del componente UploadMaterial
type UploadState =
  | { step: 'metadata'; progress: 0 }
  | { step: 'uploading'; progress: number } // 0-100
  | { step: 'processing'; estimatedTime: number }
  | { step: 'completed'; materialId: string }
  | { step: 'error'; error: string };

// Componente de progreso
function UploadProgress({ state }: { state: UploadState }) {
  if (state.step === 'metadata') {
    return <StepIndicator current={1} total={3} />;
  }

  if (state.step === 'uploading') {
    return (
      <ProgressBar
        value={state.progress}
        label={`Subiendo archivo ${state.progress}%`}
        animated
      />
    );
  }

  if (state.step === 'processing') {
    return (
      <div>
        <Spinner />
        <p>Procesando con IA...</p>
        <p className="text-sm text-gray-500">
          Estimado: {state.estimatedTime} segundos
        </p>
      </div>
    );
  }

  if (state.step === 'completed') {
    return (
      <SuccessMessage>
        <CheckIcon /> Material listo
        <Link to={`/materials/${state.materialId}`}>Ver material</Link>
      </SuccessMessage>
    );
  }

  if (state.step === 'error') {
    return (
      <ErrorMessage>
        <AlertIcon /> {state.error}
        <Button onClick={retry}>Reintentar</Button>
      </ErrorMessage>
    );
  }
}
```

### Cancelación de Upload
```typescript
// Usar AbortController para cancelar upload a S3
const abortController = new AbortController();

async function uploadWithCancellation(url: string, file: File) {
  try {
    await axios.put(url, file, {
      signal: abortController.signal,
      onUploadProgress: updateProgress
    });
  } catch (error) {
    if (axios.isCancel(error)) {
      // Mostrar confirmación
      const confirmed = await confirm(
        '¿Seguro que deseas cancelar? Perderás el progreso.'
      );
      if (confirmed) {
        // Limpiar material creado en backend
        await api.delete(`/v1/materials/${materialId}`);
      }
    }
    throw error;
  }
}

// Botón cancelar
<Button onClick={() => abortController.abort()}>
  Cancelar upload
</Button>
```

### Estimación de Tiempo de Procesamiento
```typescript
// Estimar basado en tamaño de archivo y páginas promedio
function estimateProcessingTime(fileSizeBytes: number): number {
  // Asumir 50KB por página promedio
  const estimatedPages = Math.ceil(fileSizeBytes / (50 * 1024));

  // 2-5 segundos por página (depende de OpenAI)
  const minSeconds = estimatedPages * 2;
  const maxSeconds = estimatedPages * 5;

  const avgSeconds = Math.floor((minSeconds + maxSeconds) / 2);

  return Math.max(30, avgSeconds); // Mínimo 30 segundos
}

// Uso
const estimate = estimateProcessingTime(pdfFile.size);
showMessage(`Tiempo estimado: ${formatDuration(estimate)}`);
```

## Almacenamiento Local

### Guardar Progreso de Upload
```typescript
// Persistir en IndexedDB para recuperar si se cierra la app
interface UploadProgress {
  materialId: string;
  fileName: string;
  fileSize: number;
  uploadProgress: number; // 0-100
  status: 'uploading' | 'processing' | 'completed' | 'failed';
  createdAt: number;
  uploadUrl?: string;
  uploadUrlGeneratedAt?: number;
}

// Guardar progreso
await db.uploadProgress.put({
  materialId,
  fileName: file.name,
  fileSize: file.size,
  uploadProgress: currentProgress,
  status: 'uploading',
  createdAt: Date.now(),
  uploadUrl: presignedUrl,
  uploadUrlGeneratedAt: Date.now()
});

// Recuperar al reabrir app
const pendingUploads = await db.uploadProgress
  .where('status')
  .equals('uploading')
  .toArray();

if (pendingUploads.length > 0) {
  showRecoveryDialog({
    message: `Tienes ${pendingUploads.length} upload(s) pendiente(s)`,
    actions: ['Continuar', 'Cancelar']
  });
}
```

### Cache de Materiales Subidos
```typescript
// Actualizar cache local después de upload exitoso
const materialsCache = await db.materials.toArray();
materialsCache.push(newMaterial);
await db.materials.bulkPut(materialsCache);

// Invalidar query cache de React Query
queryClient.invalidateQueries({ queryKey: ['materials'] });
```

## Código de Ejemplo (Mobile - TypeScript)

### Hook Completo de Upload
```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { useState } from 'react';
import { materialsApi } from '@/services/api';

interface UploadMaterialParams {
  metadata: CreateMaterialRequest;
  file: File;
}

interface UploadState {
  step: 'metadata' | 'uploading' | 'processing' | 'completed' | 'error';
  progress: number;
  error?: string;
  materialId?: string;
}

export function useUploadMaterial() {
  const queryClient = useQueryClient();
  const [uploadState, setUploadState] = useState<UploadState>({
    step: 'metadata',
    progress: 0
  });

  const uploadMutation = useMutation({
    mutationFn: async ({ metadata, file }: UploadMaterialParams) => {
      try {
        // Paso 1: Crear material
        setUploadState({ step: 'metadata', progress: 0 });
        const material = await materialsApi.create(metadata);

        // Paso 2: Generar URL presignada
        const { upload_url, file_url } = await materialsApi.generateUploadURL(
          material.id,
          { file_name: file.name, content_type: 'application/pdf' }
        );

        // Paso 3: Upload a S3
        setUploadState({ step: 'uploading', progress: 0 });

        await uploadToS3(upload_url, file, (progress) => {
          setUploadState({ step: 'uploading', progress });
        });

        // Paso 4: Notificar completitud
        await materialsApi.notifyUploadComplete(material.id, {
          file_url,
          file_type: file.type,
          file_size_bytes: file.size
        });

        // Paso 5: Esperar procesamiento
        setUploadState({ step: 'processing', progress: 0 });

        const readyMaterial = await waitForProcessing(material.id, (elapsed) => {
          setUploadState({
            step: 'processing',
            progress: Math.min(95, Math.floor((elapsed / 300) * 100)) // Max 5 min
          });
        });

        setUploadState({
          step: 'completed',
          progress: 100,
          materialId: readyMaterial.id
        });

        return readyMaterial;

      } catch (error) {
        setUploadState({
          step: 'error',
          progress: 0,
          error: error instanceof Error ? error.message : 'Error desconocido'
        });
        throw error;
      }
    },
    onSuccess: () => {
      // Invalidar cache de materiales
      queryClient.invalidateQueries({ queryKey: ['materials'] });
    }
  });

  return {
    upload: uploadMutation.mutate,
    uploadAsync: uploadMutation.mutateAsync,
    isUploading: uploadMutation.isPending,
    uploadState,
    reset: () => setUploadState({ step: 'metadata', progress: 0 })
  };
}

// Uso en componente
function UploadMaterialForm() {
  const { upload, isUploading, uploadState } = useUploadMaterial();

  const handleSubmit = (e: FormEvent) => {
    e.preventDefault();

    const formData = new FormData(e.target as HTMLFormElement);
    const file = formData.get('pdf_file') as File;

    // Validar
    const validation = validatePDF(file);
    if (!validation.valid) {
      showErrors(validation.errors);
      return;
    }

    // Iniciar upload
    upload({
      metadata: {
        title: formData.get('title') as string,
        description: formData.get('description') as string,
        subject: formData.get('subject') as string,
        grade: formData.get('grade') as string
      },
      file
    });
  };

  return (
    <form onSubmit={handleSubmit}>
      <Input name="title" label="Título" required />
      <Textarea name="description" label="Descripción" />
      <Select name="subject" label="Materia" />
      <Select name="grade" label="Grado" />
      <FileInput
        name="pdf_file"
        accept="application/pdf"
        maxSize={100 * 1024 * 1024}
        required
      />

      <UploadProgress state={uploadState} />

      <Button type="submit" disabled={isUploading}>
        {isUploading ? 'Subiendo...' : 'Subir Material'}
      </Button>
    </form>
  );
}
```

### Funciones Auxiliares
```typescript
// Upload a S3 con tracking de progreso
async function uploadToS3(
  presignedUrl: string,
  file: File,
  onProgress: (percentage: number) => void
): Promise<void> {
  await axios.put(presignedUrl, file, {
    headers: { 'Content-Type': 'application/pdf' },
    timeout: 120_000,
    onUploadProgress: (progressEvent) => {
      if (progressEvent.total) {
        const percentCompleted = Math.round(
          (progressEvent.loaded * 100) / progressEvent.total
        );
        onProgress(percentCompleted);
      }
    }
  });
}

// Esperar procesamiento con polling
async function waitForProcessing(
  materialId: string,
  onProgress?: (elapsedSeconds: number) => void
): Promise<MaterialResponse> {
  const MAX_ATTEMPTS = 60;
  const INTERVAL = 5000;

  for (let i = 0; i < MAX_ATTEMPTS; i++) {
    const material = await materialsApi.get(materialId);

    if (material.status === 'ready') {
      return material;
    }

    if (material.status === 'failed') {
      throw new Error('Error al procesar el PDF');
    }

    const elapsed = (i + 1) * (INTERVAL / 1000);
    onProgress?.(elapsed);

    await sleep(INTERVAL);
  }

  throw new Error('Timeout procesando material');
}

function sleep(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms));
}
```

## Notas de Implementación

### Evento RabbitMQ (Backend)
```go
// internal/infrastructure/messaging/rabbitmq/events.go
type MaterialUploadedEvent struct {
    EventID     string    `json:"event_id"`
    EventType   string    `json:"event_type"`
    EventVersion string   `json:"event_version"`
    Timestamp   time.Time `json:"timestamp"`
    Payload     MaterialUploadedPayload `json:"payload"`
}

type MaterialUploadedPayload struct {
    MaterialID   string            `json:"material_id"`
    SchoolID     string            `json:"school_id"`
    TeacherID    string            `json:"teacher_id"`
    FileURL      string            `json:"file_url"`
    FileSizeBytes int64            `json:"file_size_bytes"`
    FileType     string            `json:"file_type"`
    Metadata     map[string]string `json:"metadata"`
}

// Publicar evento
func (s *MaterialService) NotifyUploadComplete(...) error {
    event := MaterialUploadedEvent{
        EventID:      uuid.New().String(),
        EventType:    "material.uploaded",
        EventVersion: "1.0",
        Timestamp:    time.Now(),
        Payload: MaterialUploadedPayload{
            MaterialID:    materialID,
            SchoolID:      material.SchoolID,
            TeacherID:     material.UploadedByTeacherID,
            FileURL:       req.FileURL,
            FileSizeBytes: req.FileSizeBytes,
            FileType:      req.FileType,
            Metadata:      make(map[string]string),
        },
    }

    return s.publisher.Publish(ctx, "edugo.events", "material.uploaded", event)
}
```

### Worker Processing (Referencia)
```go
// edugo-worker/internal/application/processor/material_uploaded_processor.go
func (p *MaterialUploadedProcessor) processEvent(ctx context.Context, event MaterialUploadedEvent) error {
    // 1. Actualizar estado a 'processing'
    p.updateStatus(ctx, event.MaterialID, "processing")

    // 2. Descargar PDF de S3
    pdfReader, err := p.storageClient.Download(ctx, event.Payload.FileURL)
    defer pdfReader.Close()

    // 3. Extraer texto
    extractedText, err := p.pdfExtractor.Extract(ctx, pdfReader)

    // 4. Generar resumen con OpenAI/Fallback
    summary, err := p.nlpClient.GenerateSummary(ctx, extractedText)

    // 5. Generar quiz con OpenAI/Fallback
    quiz, err := p.nlpClient.GenerateQuiz(ctx, extractedText, 10)

    // 6. Guardar en MongoDB
    p.saveSummary(ctx, event.MaterialID, summary)
    p.saveAssessment(ctx, event.MaterialID, quiz)

    // 7. Actualizar estado a 'ready'
    p.updateStatus(ctx, event.MaterialID, "ready")

    return nil
}
```

### Seguridad S3
```typescript
// Backend genera URL presignada con permisos limitados
const params = {
  Bucket: 'edugo-materials',
  Key: `materials/${materialID}/${sanitizedFileName}`,
  Expires: 900, // 15 minutos
  ContentType: 'application/pdf',
  ServerSideEncryption: 'AES256'
};

const uploadURL = s3.getSignedUrl('putObject', params);

// Cliente SOLO puede hacer PUT a esa URL específica
// No puede listar bucket, eliminar, ni acceder a otros archivos
```

### Monitoreo y Alertas
```typescript
// Métricas a trackear
metrics.materialUploadStarted.inc();
metrics.materialUploadCompleted.inc({ status: 'success' });
metrics.materialUploadDuration.observe(durationMs);

// Alertas si Worker tarda mucho
if (processingDuration > 10 * 60 * 1000) { // 10 min
  logger.warn('Worker processing exceeded 10 minutes', {
    materialId,
    duration: processingDuration
  });
}
```

### Testing
```typescript
describe('Upload Material Flow', () => {
  it('completa flujo exitosamente', async () => {
    const mockFile = new File(['dummy'], 'test.pdf', {
      type: 'application/pdf'
    });

    const { result } = renderHook(() => useUploadMaterial());

    await act(async () => {
      await result.current.uploadAsync({
        metadata: { title: 'Test Material' },
        file: mockFile
      });
    });

    expect(result.current.uploadState.step).toBe('completed');
    expect(result.current.uploadState.materialId).toBeDefined();
  });

  it('maneja error de validación', async () => {
    const invalidFile = new File(['dummy'], 'test.txt', {
      type: 'text/plain'
    });

    const validation = validatePDF(invalidFile);

    expect(validation.valid).toBe(false);
    expect(validation.errors).toContain('Solo se permiten archivos PDF');
  });

  it('maneja timeout de S3', async () => {
    mockAxios.put.mockImplementation(() =>
      new Promise((_, reject) =>
        setTimeout(() => reject({ code: 'ECONNABORTED' }), 200)
      )
    );

    const { result } = renderHook(() => useUploadMaterial());

    await expect(
      result.current.uploadAsync({
        metadata: { title: 'Test' },
        file: mockPDF
      })
    ).rejects.toThrow('Timeout');
  });
});
```

---

**Última actualización:** 2025-12-24
**Versión:** 1.0
**Estado:** ✅ Listo para implementación
**Criticidad:** ALTA (flujo core del sistema)
