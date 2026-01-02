# RFC-030: Obtener Quiz Generado por IA

## Metadata
- **ID:** RFC-030
- **Proceso:** Evaluaciones
- **Subproceso:** Obtener Quiz
- **Prioridad:** Alta
- **Dependencias:** RFC-010 (Listar Materiales), RFC-012 (Ver Detalle Material), Worker (crítico)
- **Estado API:** ❌ Bloqueado (depende Worker)

## Descripción

Permite a los estudiantes obtener el quiz (cuestionario de evaluación) generado automáticamente por IA a partir del contenido de un material educativo. El quiz se genera durante el procesamiento del PDF por el Worker y se almacena en MongoDB.

## Flujo de Usuario (UX)

### Pantalla: Detalle de Material

```
┌─────────────────────────────────────────┐
│  ← Matemáticas Grado 5                  │
│                                          │
│  [📄 Ver PDF]  [📝 Tomar Quiz]  [📖 Resumen] │
│                                          │
│  Estado: ✅ Listo para estudiar          │
│                                          │
│  ┌──────────────────────────────────┐   │
│  │ 📝 Evaluación Disponible          │   │
│  │                                   │   │
│  │ Preguntas: 10                     │   │
│  │ Tiempo estimado: 15 minutos       │   │
│  │ Intentos disponibles: 3           │   │
│  │ Nota mínima: 70%                  │   │
│  │                                   │   │
│  │      [Comenzar Evaluación]        │   │
│  └──────────────────────────────────┘   │
│                                          │
│  Historial de intentos:                  │
│  1. 85% - Aprobado - hace 2 días         │
│  2. 65% - Reprobado - hace 1 semana      │
│                                          │
└─────────────────────────────────────────┘
```

### Estados del Material

**Si status = 'ready':**
- Mostrar botón "Comenzar Evaluación"
- Mostrar metadata del quiz (preguntas, tiempo, intentos)

**Si status = 'processing':**
```
┌──────────────────────────────────────┐
│  ⏳ Generando evaluación...           │
│  ────────────────── 60%              │
│  El quiz estará disponible en        │
│  aproximadamente 2 minutos           │
│                                      │
│  [Reintentar]  [Notificarme]         │
└──────────────────────────────────────┘
```

**Si status = 'failed':**
```
┌──────────────────────────────────────┐
│  ❌ Error al generar evaluación       │
│  No fue posible procesar el material │
│                                      │
│  [Reportar Problema]  [Volver]       │
└──────────────────────────────────────┘
```

**Si Worker no disponible (404):**
```
┌──────────────────────────────────────┐
│  🤖 Evaluación aún no generada        │
│  El sistema está procesando el       │
│  material. Intenta nuevamente en     │
│  unos minutos.                       │
│                                      │
│  [Actualizar]  [Notificarme]         │
└──────────────────────────────────────┘
```

## Flujo de Datos (Técnico)

### Diagrama de Secuencia

```
Usuario → Frontend → API Mobile → PostgreSQL → MongoDB
                                      ↓            ↓
                                  assessment    material_
                                  (metadata)    assessment_worker
                                                (preguntas)
```

### Flujo Detallado

1. **Usuario hace clic en "Ver Evaluación"**
2. **Frontend verifica estado del material:**
   ```typescript
   const material = await getMaterial(materialId);

   if (material.status !== 'ready') {
     showProcessingState(material.status);
     return;
   }
   ```

3. **Frontend solicita quiz:**
   ```typescript
   try {
     const quiz = await getAssessment(materialId);
     showQuiz(quiz);
   } catch (error) {
     if (error.status === 404) {
       showNotReadyMessage();
       startPolling(); // Verificar cada 5s
     }
   }
   ```

4. **API Mobile consulta dos fuentes:**
   - **PostgreSQL (assessment):** metadata, configuración
   - **MongoDB (material_assessment_worker):** preguntas SIN respuestas correctas

5. **API Mobile combina y retorna quiz**

### Endpoints Involucrados

**Principal:**
```
GET /v1/materials/:id/assessment
```

**Prerequisito:**
```
GET /v1/materials/:id
```

### Request/Response (TypeScript)

**Request:**
```typescript
interface GetAssessmentRequest {
  // Sin body, solo materialId en URL
}
```

**Response Exitoso (200):**
```typescript
interface AssessmentResponse {
  id: string;                    // UUID del assessment
  material_id: string;           // UUID del material
  title: string;                 // "Evaluación: Matemáticas Grado 5"
  questions: QuestionDTO[];      // Lista de preguntas
  questions_count: number;       // 10
  total_questions: number;       // 10 (puede ser subset en el futuro)
  max_attempts: number;          // 3
  pass_threshold: number;        // 70 (porcentaje)
  time_limit_minutes: number;    // 30
  estimated_time_minutes: number;// 15
}

interface QuestionDTO {
  id: string;                    // "q_abc123"
  text: string;                  // "¿Cuál es la definición de...?"
  type: "multiple_choice";       // Tipo de pregunta
  options: OptionDTO[];          // Opciones de respuesta
  // ❌ NO incluye correct_answer (solo servidor)
}

interface OptionDTO {
  id: string;                    // "opt_a"
  text: string;                  // "Opción A"
}
```

**Response Error (404 - No generado aún):**
```json
{
  "error": "not_found",
  "message": "Assessment not generated yet",
  "code": "ASSESSMENT_NOT_READY"
}
```

**Response Error (404 - Material no existe):**
```json
{
  "error": "not_found",
  "message": "Material not found",
  "code": "MATERIAL_NOT_FOUND"
}
```

**Response Error (500 - Worker falló):**
```json
{
  "error": "internal_server_error",
  "message": "Failed to retrieve assessment",
  "code": "ASSESSMENT_RETRIEVAL_FAILED"
}
```

## Estados y Transiciones

### Estado del Material (Prerequisito)

```
Material States:
┌──────────┐
│ uploaded │ → PDF subido, worker no ha procesado
└──────────┘
     ↓
┌────────────┐
│ processing │ → Worker extrayendo texto, generando quiz
└────────────┘
     ↓
┌───────┐
│ ready │ → Quiz generado y disponible ✅
└───────┘
     ↓ (si falla)
┌────────┐
│ failed │ → Error en generación ❌
└────────┘
```

### Estado de Disponibilidad del Quiz

```
Quiz Availability States:

┌────────────────┐
│ NOT_GENERATED  │ → Material en processing/uploaded
│ (404)          │ → Mostrar "Generando..."
└────────────────┘
        ↓
┌────────────────┐
│ AVAILABLE      │ → Material en ready
│ (200)          │ → Mostrar quiz completo
└────────────────┘
        ↓ (si worker falla)
┌────────────────┐
│ GENERATION_    │ → Material en failed
│ FAILED (404)   │ → Mostrar error
└────────────────┘
```

## Manejo de Errores

### Errores de Red

**Timeout (>30s):**
```typescript
try {
  const quiz = await getAssessment(materialId, { timeout: 30000 });
} catch (error) {
  if (error.code === 'TIMEOUT') {
    showError('La conexión está lenta. Intenta nuevamente.');
    offerRetry();
  }
}
```

### Errores de Negocio

**404 - Assessment no generado:**
```typescript
if (error.status === 404 && error.code === 'ASSESSMENT_NOT_READY') {
  // Normal: material aún procesándose
  showMessage('El quiz está siendo generado...');
  startPolling(materialId, {
    interval: 5000,      // Cada 5 segundos
    maxAttempts: 60,     // 5 minutos total
    onReady: () => loadAssessment(materialId),
    onTimeout: () => showTimeoutMessage()
  });
}
```

**404 - Material no existe:**
```typescript
if (error.status === 404 && error.code === 'MATERIAL_NOT_FOUND') {
  showError('Material no encontrado');
  redirectTo('/materials');
}
```

**500 - Error de servidor:**
```typescript
if (error.status === 500) {
  showError('Error al cargar la evaluación. Intenta más tarde.');
  logError(error); // Para debugging
}
```

### Degradación Graceful

**Worker no disponible pero Worker con Fallback activo:**
- Quiz generado con calidad reducida
- Preguntas genéricas basadas en extracción simple
- Mostrar disclaimer: "Esta evaluación fue generada automáticamente y puede no reflejar completamente el contenido."

## Consideraciones de UX

### Polling para Material en Procesamiento

```typescript
interface PollingOptions {
  interval: number;        // 5000ms (5 segundos)
  maxAttempts: number;     // 60 (5 minutos total)
  backoffMultiplier?: number; // Opcional: 1.5 (aumentar intervalo)
}

async function pollUntilReady(
  materialId: string,
  options: PollingOptions
): Promise<AssessmentResponse> {

  let attempt = 0;
  let currentInterval = options.interval;

  while (attempt < options.maxAttempts) {
    try {
      const assessment = await getAssessment(materialId);
      return assessment; // Éxito

    } catch (error) {
      if (error.status === 404 && error.code === 'ASSESSMENT_NOT_READY') {
        // Continuar polling
        await sleep(currentInterval);

        // Backoff exponencial (opcional)
        if (options.backoffMultiplier) {
          currentInterval *= options.backoffMultiplier;
        }

        attempt++;
        updateProgressIndicator(attempt, options.maxAttempts);

      } else {
        throw error; // Error real
      }
    }
  }

  throw new Error('Timeout: Assessment not ready after polling');
}
```

### Skeleton Loaders

```typescript
function showAssessmentSkeleton() {
  return (
    <div className="assessment-skeleton">
      <div className="skeleton-title" />
      <div className="skeleton-metadata">
        <div className="skeleton-pill" />
        <div className="skeleton-pill" />
        <div className="skeleton-pill" />
      </div>
      <div className="skeleton-questions">
        {[1, 2, 3].map(i => (
          <div key={i} className="skeleton-question">
            <div className="skeleton-text" />
            <div className="skeleton-options">
              <div className="skeleton-option" />
              <div className="skeleton-option" />
              <div className="skeleton-option" />
              <div className="skeleton-option" />
            </div>
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Animaciones de Carga

```typescript
// Mostrar progreso estimado
function showProcessingAnimation(estimatedSeconds: number) {
  const startTime = Date.now();

  const interval = setInterval(() => {
    const elapsed = (Date.now() - startTime) / 1000;
    const progress = Math.min((elapsed / estimatedSeconds) * 100, 95);

    updateProgressBar(progress);
    updateMessage(getProgressMessage(progress));

    if (progress >= 95) {
      clearInterval(interval);
      showMessage('Finalizando...');
    }
  }, 500);

  return () => clearInterval(interval);
}

function getProgressMessage(progress: number): string {
  if (progress < 25) return 'Analizando PDF...';
  if (progress < 50) return 'Extrayendo contenido clave...';
  if (progress < 75) return 'Generando preguntas...';
  return 'Finalizando evaluación...';
}
```

### Notificaciones

```typescript
// Opcional: Notificar cuando esté listo
async function enableNotifications(materialId: string) {
  if ('Notification' in window && Notification.permission === 'granted') {
    const checkInterval = setInterval(async () => {
      try {
        const quiz = await getAssessment(materialId);
        clearInterval(checkInterval);

        new Notification('Quiz Disponible', {
          body: 'La evaluación de tu material está lista',
          icon: '/icons/quiz-ready.png',
          tag: `quiz-${materialId}`
        });

      } catch (error) {
        // Continuar esperando
      }
    }, 10000); // Cada 10 segundos
  }
}
```

## Almacenamiento Local

### Caché de Assessment

```typescript
interface AssessmentCache {
  materialId: string;
  assessment: AssessmentResponse;
  cachedAt: number; // timestamp
  expiresAt: number; // timestamp
}

// Guardar en localStorage
function cacheAssessment(
  materialId: string,
  assessment: AssessmentResponse,
  ttlSeconds: number = 3600 // 1 hora
) {
  const cache: AssessmentCache = {
    materialId,
    assessment,
    cachedAt: Date.now(),
    expiresAt: Date.now() + (ttlSeconds * 1000)
  };

  localStorage.setItem(
    `assessment_${materialId}`,
    JSON.stringify(cache)
  );
}

// Recuperar de cache
function getCachedAssessment(
  materialId: string
): AssessmentResponse | null {

  const cached = localStorage.getItem(`assessment_${materialId}`);
  if (!cached) return null;

  const cache: AssessmentCache = JSON.parse(cached);

  // Verificar expiración
  if (Date.now() > cache.expiresAt) {
    localStorage.removeItem(`assessment_${materialId}`);
    return null;
  }

  return cache.assessment;
}
```

### Uso del Caché

```typescript
async function getAssessmentWithCache(
  materialId: string
): Promise<AssessmentResponse> {

  // 1. Intentar desde cache
  const cached = getCachedAssessment(materialId);
  if (cached) {
    console.log('Assessment loaded from cache');
    return cached;
  }

  // 2. Obtener de API
  const assessment = await getAssessment(materialId);

  // 3. Guardar en cache
  cacheAssessment(materialId, assessment);

  return assessment;
}
```

## Código de Ejemplo (Mobile - TypeScript)

### Service: AssessmentService

```typescript
// services/AssessmentService.ts

import axios, { AxiosInstance } from 'axios';

interface AssessmentServiceConfig {
  baseURL: string;
  timeout?: number;
}

export class AssessmentService {
  private api: AxiosInstance;

  constructor(config: AssessmentServiceConfig) {
    this.api = axios.create({
      baseURL: config.baseURL,
      timeout: config.timeout || 30000,
      headers: {
        'Content-Type': 'application/json'
      }
    });

    // Interceptor para agregar token
    this.api.interceptors.request.use((config) => {
      const token = localStorage.getItem('access_token');
      if (token) {
        config.headers.Authorization = `Bearer ${token}`;
      }
      return config;
    });
  }

  /**
   * Obtener assessment de un material
   * @throws AssessmentNotReadyError si el quiz no está generado (404)
   * @throws MaterialNotFoundError si el material no existe (404)
   * @throws NetworkError si hay problemas de red
   */
  async getAssessment(materialId: string): Promise<AssessmentResponse> {
    try {
      const response = await this.api.get<AssessmentResponse>(
        `/v1/materials/${materialId}/assessment`
      );
      return response.data;

    } catch (error: any) {
      if (error.response?.status === 404) {
        const code = error.response.data?.code;

        if (code === 'ASSESSMENT_NOT_READY') {
          throw new AssessmentNotReadyError(materialId);
        }

        if (code === 'MATERIAL_NOT_FOUND') {
          throw new MaterialNotFoundError(materialId);
        }
      }

      throw new NetworkError('Failed to fetch assessment', error);
    }
  }

  /**
   * Esperar hasta que el assessment esté disponible
   */
  async waitForAssessment(
    materialId: string,
    options: PollingOptions = { interval: 5000, maxAttempts: 60 }
  ): Promise<AssessmentResponse> {

    let attempt = 0;

    while (attempt < options.maxAttempts) {
      try {
        return await this.getAssessment(materialId);

      } catch (error) {
        if (error instanceof AssessmentNotReadyError) {
          await this.sleep(options.interval);
          attempt++;
          continue;
        }
        throw error;
      }
    }

    throw new PollingTimeoutError(
      `Assessment not ready after ${options.maxAttempts} attempts`
    );
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// Custom Errors
export class AssessmentNotReadyError extends Error {
  constructor(public materialId: string) {
    super(`Assessment not ready for material ${materialId}`);
    this.name = 'AssessmentNotReadyError';
  }
}

export class MaterialNotFoundError extends Error {
  constructor(public materialId: string) {
    super(`Material ${materialId} not found`);
    this.name = 'MaterialNotFoundError';
  }
}

export class PollingTimeoutError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'PollingTimeoutError';
  }
}

export class NetworkError extends Error {
  constructor(message: string, public originalError: any) {
    super(message);
    this.name = 'NetworkError';
  }
}
```

### Hook: useAssessment

```typescript
// hooks/useAssessment.ts

import { useState, useEffect, useCallback } from 'react';
import { AssessmentService, AssessmentNotReadyError } from '../services/AssessmentService';

interface UseAssessmentOptions {
  materialId: string;
  autoFetch?: boolean;
  enablePolling?: boolean;
  pollingInterval?: number;
}

interface UseAssessmentReturn {
  assessment: AssessmentResponse | null;
  loading: boolean;
  error: Error | null;
  isReady: boolean;
  progress: number; // Progreso estimado de generación (0-100)
  refetch: () => Promise<void>;
}

export function useAssessment(
  options: UseAssessmentOptions
): UseAssessmentReturn {

  const [assessment, setAssessment] = useState<AssessmentResponse | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  const [isReady, setIsReady] = useState(false);
  const [progress, setProgress] = useState(0);

  const service = new AssessmentService({
    baseURL: process.env.REACT_APP_API_MOBILE_URL || 'http://localhost:8080'
  });

  const fetchAssessment = useCallback(async () => {
    setLoading(true);
    setError(null);

    try {
      // Intentar obtener desde cache primero
      const cached = getCachedAssessment(options.materialId);
      if (cached) {
        setAssessment(cached);
        setIsReady(true);
        setLoading(false);
        return;
      }

      // Obtener de API
      if (options.enablePolling) {
        // Con polling automático
        const result = await service.waitForAssessment(
          options.materialId,
          {
            interval: options.pollingInterval || 5000,
            maxAttempts: 60,
            onProgress: (attempt, max) => {
              setProgress((attempt / max) * 100);
            }
          }
        );

        setAssessment(result);
        setIsReady(true);
        cacheAssessment(options.materialId, result);

      } else {
        // Sin polling
        const result = await service.getAssessment(options.materialId);
        setAssessment(result);
        setIsReady(true);
        cacheAssessment(options.materialId, result);
      }

    } catch (err: any) {
      if (err instanceof AssessmentNotReadyError) {
        setIsReady(false);
        setError(err);
      } else {
        setError(err);
      }
    } finally {
      setLoading(false);
    }
  }, [options.materialId, options.enablePolling, options.pollingInterval]);

  useEffect(() => {
    if (options.autoFetch) {
      fetchAssessment();
    }
  }, [options.autoFetch, fetchAssessment]);

  return {
    assessment,
    loading,
    error,
    isReady,
    progress,
    refetch: fetchAssessment
  };
}
```

### Componente: AssessmentLoader

```typescript
// components/AssessmentLoader.tsx

import React from 'react';
import { useAssessment } from '../hooks/useAssessment';
import { AssessmentView } from './AssessmentView';
import { AssessmentSkeleton } from './AssessmentSkeleton';
import { ProcessingMessage } from './ProcessingMessage';
import { ErrorMessage } from './ErrorMessage';

interface AssessmentLoaderProps {
  materialId: string;
  onStartAttempt?: (assessmentId: string) => void;
}

export const AssessmentLoader: React.FC<AssessmentLoaderProps> = ({
  materialId,
  onStartAttempt
}) => {

  const {
    assessment,
    loading,
    error,
    isReady,
    progress,
    refetch
  } = useAssessment({
    materialId,
    autoFetch: true,
    enablePolling: true,
    pollingInterval: 5000
  });

  // Estado: Cargando inicial
  if (loading && !assessment) {
    return <AssessmentSkeleton />;
  }

  // Estado: Error (no generado aún)
  if (error && !isReady) {
    return (
      <ProcessingMessage
        message="El quiz está siendo generado..."
        progress={progress}
        onRetry={refetch}
      />
    );
  }

  // Estado: Error real
  if (error && !assessment) {
    return (
      <ErrorMessage
        message="Error al cargar la evaluación"
        onRetry={refetch}
      />
    );
  }

  // Estado: Quiz disponible
  if (assessment) {
    return (
      <AssessmentView
        assessment={assessment}
        onStart={() => onStartAttempt?.(assessment.id)}
      />
    );
  }

  return null;
};
```

## Notas de Implementación

### 1. Dependencia Crítica del Worker

**Bloqueante:**
- El Worker DEBE estar desplegado y funcional
- El Worker procesa el PDF con IA (OpenAI/Claude/Fallback)
- Sin Worker: 404 permanente

**Verificación Previa:**
```bash
# DevOps debe validar:
curl http://worker:8083/health
# Esperado: {"status": "healthy"}

# Verificar cola RabbitMQ
rabbitmqctl list_queues | grep material.uploaded
# Esperado: material.uploaded <número>
```

### 2. Tiempos de Generación

**Estimaciones según tamaño del PDF:**
- PDF 10 páginas: ~30-60 segundos
- PDF 50 páginas: ~2-5 minutos
- PDF 100 páginas: ~5-10 minutos

**Configurar timeouts apropiados:**
```typescript
const ASSESSMENT_TIMEOUTS = {
  initial: 30000,     // 30s primer intento
  polling: 5000,      // 5s entre reintentos
  maxPolling: 300000  // 5min total de polling
};
```

### 3. Caché y Performance

**Cache Strategy:**
- Guardar assessment en localStorage por 1 hora
- Invalidar cache si material se actualiza
- Limpiar cache al cerrar sesión

**Bundle Size:**
- Lazy load componente de Quiz
- Code splitting por ruta
- Preload assessment al hover "Ver Evaluación"

### 4. Accesibilidad

**Lectores de Pantalla:**
```typescript
<div role="status" aria-live="polite" aria-atomic="true">
  {loading && "Cargando evaluación..."}
  {isReady && `Evaluación lista con ${assessment.questions_count} preguntas`}
</div>
```

**Teclado:**
- Tab para navegar opciones
- Enter para seleccionar
- Espaciador para marcar/desmarcar

### 5. Testing

**Unit Tests:**
```typescript
describe('AssessmentService', () => {
  it('should fetch assessment successfully', async () => {
    const service = new AssessmentService({ baseURL: 'http://test' });
    const assessment = await service.getAssessment('material-123');
    expect(assessment.questions_count).toBeGreaterThan(0);
  });

  it('should throw AssessmentNotReadyError on 404', async () => {
    // Mock 404 response
    await expect(
      service.getAssessment('material-123')
    ).rejects.toThrow(AssessmentNotReadyError);
  });
});
```

**Integration Tests:**
```typescript
describe('Assessment Flow', () => {
  it('should poll until assessment is ready', async () => {
    // Simular procesamiento gradual
    const assessment = await service.waitForAssessment('material-123', {
      interval: 100,
      maxAttempts: 10
    });
    expect(assessment).toBeDefined();
  });
});
```

### 6. Monitoreo

**Métricas a Trackear:**
- Tiempo promedio de generación
- Tasa de timeout en polling
- Tasa de éxito/fallo del Worker
- Cantidad de intentos por estudiante

**Logging:**
```typescript
logger.info('Assessment requested', {
  materialId,
  userId: currentUser.id,
  cached: !!cachedAssessment
});

logger.warn('Assessment not ready, polling started', {
  materialId,
  attempt: 1,
  maxAttempts: 60
});

logger.error('Assessment generation failed', {
  materialId,
  error: error.message
});
```
