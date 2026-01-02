# RFC-002: Polling para Estados de Procesamiento (Worker)

## Metadata
- **ID:** RFC-002
- **Proceso:** Arquitectura General
- **Subproceso:** Polling de Worker
- **Prioridad:** Alta
- **Dependencias:** RFC-000, RFC-001
- **Estado API:** ⚠️ Parcial (Worker dependency)

## Descripción
Define la estrategia de polling para monitorear el estado de procesamiento de materiales por parte del Worker. El Worker procesa PDFs con IA para generar resúmenes y evaluaciones, proceso que puede tomar entre 30 segundos y 10 minutos dependiendo del tamaño del archivo.

## Flujo de Usuario (UX)
1. Docente sube un PDF exitosamente
2. App muestra "Procesando material con IA..."
3. App realiza polling periódico del estado del material
4. App actualiza indicador de progreso (estimado)
5. Cuando procesamiento completa, app muestra "Material listo"
6. Usuario puede acceder a resumen y evaluaciones

## Flujo de Datos (Técnico)

### Diagrama de Secuencia
```
Docente → Upload PDF → S3
            ↓
Frontend → POST /materials/:id/upload-complete → API Mobile
            ↓
API Mobile → Emite evento "material.uploaded" → RabbitMQ
            ↓
Worker ← Consume evento
Worker → Descarga PDF de S3
Worker → Procesa con IA (30s - 10min)
Worker → Guarda en MongoDB (summary, assessment)
Worker → Actualiza PostgreSQL (status = 'ready')
            ↑
Frontend ← Polling cada 5s ← GET /materials/:id
            ↓
Detecta status = 'ready'
            ↓
Muestra "Material listo" → Permite acceder a quiz/resumen
```

### Endpoints Involucrados

| Endpoint | Método | Descripción | Estado |
|----------|--------|-------------|--------|
| `/v1/materials/:id` | GET | Obtener estado actual del material | ✅ |
| `/v1/materials/:id/assessment` | GET | Verificar si assessment está listo | ⚠️ Worker |
| `/v1/materials/:id/summary` | GET | Verificar si resumen está listo | ⚠️ Worker |

### Request/Response

**Request (GET /materials/:id):**
```typescript
// Sin body, solo headers de autenticación
GET /v1/materials/550e8400-e29b-41d4-a716-446655440000
Authorization: Bearer <jwt_token>
```

**Response:**
```typescript
interface MaterialResponse {
  id: string;
  title: string;
  description: string;
  file_url: string | null;
  status: 'uploaded' | 'processing' | 'ready' | 'failed';
  processing_started_at: string | null;
  processing_completed_at: string | null;
  created_at: string;
  updated_at: string;
}

// Ejemplo - Procesando
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "title": "Matemáticas Grado 5",
  "status": "processing",
  "file_url": "materials/550e8400.../file.pdf",
  "processing_started_at": "2024-12-24T10:00:00Z",
  "processing_completed_at": null,
  "created_at": "2024-12-24T09:55:00Z",
  "updated_at": "2024-12-24T10:00:00Z"
}

// Ejemplo - Listo
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "title": "Matemáticas Grado 5",
  "status": "ready",
  "file_url": "materials/550e8400.../file.pdf",
  "processing_started_at": "2024-12-24T10:00:00Z",
  "processing_completed_at": "2024-12-24T10:02:30Z", // 2.5 minutos
  "created_at": "2024-12-24T09:55:00Z",
  "updated_at": "2024-12-24T10:02:30Z"
}

// Ejemplo - Error
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "title": "Matemáticas Grado 5",
  "status": "failed",
  "file_url": "materials/550e8400.../file.pdf",
  "processing_started_at": "2024-12-24T10:00:00Z",
  "processing_completed_at": "2024-12-24T10:05:00Z",
  "created_at": "2024-12-24T09:55:00Z",
  "updated_at": "2024-12-24T10:05:00Z"
}
```

## Estados y Transiciones

### Diagrama de Estados del Material
```
┌─────────────────────────────────────────────────────────────────┐
│                    ESTADOS DEL MATERIAL                          │
└─────────────────────────────────────────────────────────────────┘

UPLOADED → (Worker inicia) → PROCESSING
                                  ↓
                    ┌─────────────┴────────────┐
                    ↓                          ↓
                 SUCCESS                    FAILED
                    ↓
                 READY
```

**Transiciones válidas:**
1. `uploaded` → `processing` (cuando worker consume evento)
2. `processing` → `ready` (procesamiento exitoso)
3. `processing` → `failed` (error en procesamiento)

**Transiciones inválidas:**
- `ready` → cualquier estado (inmutable)
- `failed` → cualquier estado (requiere resubida)

### Tiempos Estimados

| Tamaño PDF | Páginas | Tiempo Estimado | Timeout Recomendado |
|------------|---------|-----------------|---------------------|
| < 5 MB | 1-10 | 30-60 segundos | 2 minutos |
| 5-20 MB | 10-50 | 1-3 minutos | 5 minutos |
| > 20 MB | 50-100 | 3-10 minutos | 15 minutos |

## Manejo de Errores

| Situación | Código HTTP | Acción en UI |
|-----------|-------------|--------------|
| Material no encontrado | 404 | "Material no existe" |
| Material en status failed | 200 (status='failed') | "Error al procesar. Intenta subir nuevamente." |
| Timeout de polling | N/A (client-side) | "Procesamiento está tardando más de lo esperado. Puedes volver más tarde." |
| Worker no disponible | Status queda en 'processing' | Detectar timeout y mostrar mensaje |

## Consideraciones de UX

### Indicadores de Progreso

```typescript
// Progreso estimado basado en tiempo transcurrido
function estimateProgress(
  startedAt: string,
  estimatedDuration: number = 120000 // 2 minutos default
): number {
  const elapsed = Date.now() - new Date(startedAt).getTime();
  const progress = Math.min((elapsed / estimatedDuration) * 100, 95);
  return Math.round(progress);
}

// Estados visuales
interface ProcessingState {
  status: 'uploaded' | 'processing' | 'ready' | 'failed';
  message: string;
  icon: string;
  showProgress: boolean;
  progress?: number;
}

const ProcessingStates: Record<string, ProcessingState> = {
  uploaded: {
    status: 'uploaded',
    message: 'Material subido, iniciando procesamiento...',
    icon: '📤',
    showProgress: true,
    progress: 0
  },
  processing: {
    status: 'processing',
    message: 'Procesando con inteligencia artificial...',
    icon: '🤖',
    showProgress: true,
    progress: undefined // Calculado dinámicamente
  },
  ready: {
    status: 'ready',
    message: '¡Material procesado exitosamente!',
    icon: '✅',
    showProgress: false
  },
  failed: {
    status: 'failed',
    message: 'Error al procesar el material',
    icon: '❌',
    showProgress: false
  }
};
```

### Mensajes según Duración

```typescript
function getProgressMessage(elapsedMs: number): string {
  if (elapsedMs < 30000) {
    return 'Analizando contenido...';
  } else if (elapsedMs < 60000) {
    return 'Generando resumen con IA...';
  } else if (elapsedMs < 120000) {
    return 'Creando preguntas de evaluación...';
  } else if (elapsedMs < 300000) {
    return 'Casi listo, finalizando procesamiento...';
  } else {
    return 'Este archivo es grande, puede tardar unos minutos más...';
  }
}
```

### Skeleton Loaders
- Mostrar skeleton mientras polling está activo
- Mantener estructura de la vista de material
- Pulse animation para indicar actividad

## Almacenamiento Local

### Qué Cachear
```typescript
interface PollingState {
  materialId: string;
  startedAt: number;
  lastPollAt: number;
  attemptCount: number;
  estimatedDuration: number;
}

// Guardar estado de polling para recuperar en caso de navegación
localStorage.setItem(
  `polling_${materialId}`,
  JSON.stringify(pollingState)
);
```

### TTL del Cache
- **Estado de polling:** Hasta que se complete o falle
- **Limpiar al detectar:** `status = 'ready'` o `status = 'failed'`

### Estrategia Offline
```typescript
// Pausar polling si se pierde conexión
window.addEventListener('offline', () => {
  pauseAllPolling();
  showMessage('Sin conexión. El procesamiento continuará en segundo plano.');
});

window.addEventListener('online', () => {
  resumeAllPolling();
  showMessage('Conexión restaurada. Verificando estado...');
});
```

## Código de Ejemplo (Mobile)

### Polling Manager
```typescript
interface PollingConfig {
  interval: number;
  maxAttempts: number;
  backoffMultiplier: number;
  timeoutMs: number;
}

class MaterialPollingManager {
  private activePolls = new Map<string, NodeJS.Timeout>();
  private config: PollingConfig;

  constructor(config: Partial<PollingConfig> = {}) {
    this.config = {
      interval: 5000, // 5 segundos
      maxAttempts: 120, // 10 minutos máximo
      backoffMultiplier: 1.0, // Sin backoff (intervalo constante)
      timeoutMs: 600000, // 10 minutos timeout total
      ...config
    };
  }

  async startPolling(
    materialId: string,
    onUpdate: (material: MaterialResponse) => void,
    onComplete: (material: MaterialResponse) => void,
    onError: (error: Error) => void
  ): Promise<void> {
    // Evitar polling duplicado
    if (this.activePolls.has(materialId)) {
      console.warn(`Polling already active for material ${materialId}`);
      return;
    }

    const startTime = Date.now();
    let attemptCount = 0;

    const poll = async () => {
      try {
        attemptCount++;

        // Verificar timeout total
        if (Date.now() - startTime > this.config.timeoutMs) {
          this.stopPolling(materialId);
          onError(new Error('Timeout: El procesamiento está tardando más de lo esperado'));
          return;
        }

        // Verificar max attempts
        if (attemptCount > this.config.maxAttempts) {
          this.stopPolling(materialId);
          onError(new Error('Max attempts reached'));
          return;
        }

        // Fetch estado actual
        const material = await api.getMaterial(materialId);

        // Callback de actualización
        onUpdate(material);

        // Verificar si completó
        if (material.status === 'ready') {
          this.stopPolling(materialId);
          onComplete(material);
          return;
        }

        // Verificar si falló
        if (material.status === 'failed') {
          this.stopPolling(materialId);
          onError(new Error('Material processing failed'));
          return;
        }

        // Continuar polling
        const timeout = setTimeout(poll, this.config.interval);
        this.activePolls.set(materialId, timeout);

      } catch (error) {
        this.stopPolling(materialId);
        onError(error as Error);
      }
    };

    // Iniciar polling
    poll();
  }

  stopPolling(materialId: string): void {
    const timeout = this.activePolls.get(materialId);
    if (timeout) {
      clearTimeout(timeout);
      this.activePolls.delete(materialId);
    }
  }

  stopAll(): void {
    this.activePolls.forEach((timeout, materialId) => {
      this.stopPolling(materialId);
    });
  }

  isPolling(materialId: string): boolean {
    return this.activePolls.has(materialId);
  }
}

// Singleton
export const pollingManager = new MaterialPollingManager();
```

### Hook de React para Polling
```typescript
import { useState, useEffect, useCallback } from 'react';

interface UsePollingOptions {
  enabled?: boolean;
  interval?: number;
  maxAttempts?: number;
  onComplete?: (material: MaterialResponse) => void;
  onError?: (error: Error) => void;
}

export function useMaterialPolling(
  materialId: string,
  options: UsePollingOptions = {}
) {
  const [material, setMaterial] = useState<MaterialResponse | null>(null);
  const [isPolling, setIsPolling] = useState(false);
  const [progress, setProgress] = useState(0);
  const [elapsedTime, setElapsedTime] = useState(0);
  const [error, setError] = useState<Error | null>(null);

  const enabled = options.enabled !== false;

  const startPolling = useCallback(() => {
    if (!enabled || !materialId) return;

    setIsPolling(true);
    setError(null);

    const startTime = Date.now();

    pollingManager.startPolling(
      materialId,
      // onUpdate
      (updatedMaterial) => {
        setMaterial(updatedMaterial);

        // Calcular progreso estimado
        const elapsed = Date.now() - startTime;
        setElapsedTime(elapsed);

        if (updatedMaterial.status === 'processing') {
          const estimatedProgress = estimateProgress(
            updatedMaterial.processing_started_at || new Date().toISOString(),
            120000 // 2 minutos estimado
          );
          setProgress(estimatedProgress);
        }
      },
      // onComplete
      (completedMaterial) => {
        setMaterial(completedMaterial);
        setProgress(100);
        setIsPolling(false);
        options.onComplete?.(completedMaterial);
      },
      // onError
      (err) => {
        setError(err);
        setIsPolling(false);
        options.onError?.(err);
      }
    );
  }, [materialId, enabled, options]);

  const stopPolling = useCallback(() => {
    pollingManager.stopPolling(materialId);
    setIsPolling(false);
  }, [materialId]);

  useEffect(() => {
    // Auto-start si material está en processing
    if (enabled && material?.status === 'processing' && !isPolling) {
      startPolling();
    }

    // Cleanup
    return () => {
      stopPolling();
    };
  }, [enabled, material?.status, isPolling, startPolling, stopPolling]);

  return {
    material,
    isPolling,
    progress,
    elapsedTime,
    error,
    startPolling,
    stopPolling
  };
}

// Uso en componente
function MaterialProcessingView({ materialId }: { materialId: string }) {
  const {
    material,
    isPolling,
    progress,
    elapsedTime,
    error,
    startPolling
  } = useMaterialPolling(materialId, {
    onComplete: (material) => {
      showNotification('¡Material listo!', 'success');
      navigateToMaterial(material.id);
    },
    onError: (error) => {
      showNotification(error.message, 'error');
    }
  });

  if (error) {
    return (
      <ErrorView
        error={error}
        onRetry={startPolling}
      />
    );
  }

  if (!material) {
    return <LoadingSkeleton />;
  }

  if (material.status === 'ready') {
    return <MaterialReadyView material={material} />;
  }

  if (material.status === 'failed') {
    return <MaterialFailedView material={material} />;
  }

  return (
    <ProcessingView
      material={material}
      progress={progress}
      elapsedTime={elapsedTime}
      message={getProgressMessage(elapsedTime)}
    />
  );
}
```

### Componente de UI
```tsx
function ProcessingView({
  material,
  progress,
  elapsedTime,
  message
}: {
  material: MaterialResponse;
  progress: number;
  elapsedTime: number;
  message: string;
}) {
  return (
    <div className="processing-container">
      <div className="processing-header">
        <h2>{material.title}</h2>
        <span className="processing-badge">
          🤖 Procesando con IA
        </span>
      </div>

      <div className="progress-section">
        <ProgressBar value={progress} max={100} />
        <p className="progress-text">{progress}%</p>
      </div>

      <div className="status-message">
        <p>{message}</p>
        <small>
          Tiempo transcurrido: {formatElapsedTime(elapsedTime)}
        </small>
      </div>

      <div className="info-box">
        <p>
          Estamos analizando tu material con inteligencia artificial
          para generar:
        </p>
        <ul>
          <li>📝 Resumen automático del contenido</li>
          <li>❓ Preguntas de evaluación</li>
          <li>💡 Puntos clave</li>
        </ul>
      </div>

      <button
        className="secondary-button"
        onClick={() => window.history.back()}
      >
        Volver y revisar más tarde
      </button>
    </div>
  );
}

function formatElapsedTime(ms: number): string {
  const seconds = Math.floor(ms / 1000);
  const minutes = Math.floor(seconds / 60);
  const remainingSeconds = seconds % 60;

  if (minutes === 0) {
    return `${seconds} segundos`;
  }
  return `${minutes}m ${remainingSeconds}s`;
}
```

## Notas de Implementación

### Optimización de Intervalo
```typescript
// Backoff exponencial (opcional)
class AdaptivePollingManager extends MaterialPollingManager {
  protected calculateInterval(attemptCount: number): number {
    // Primeros 10 intentos: 5s
    if (attemptCount <= 10) return 5000;

    // Siguientes 20 intentos: 10s
    if (attemptCount <= 30) return 10000;

    // Resto: 20s
    return 20000;
  }
}
```

### Gotchas Conocidos
1. **Memory leaks:** Limpiar timeouts en unmount
2. **Múltiples polling:** Evitar duplicados para mismo material
3. **Background tab:** Pausar polling si tab no está activo (Page Visibility API)
4. **Offline:** Pausar polling si se pierde conexión

### Optimizaciones Sugeridas
1. **Page Visibility API:** Pausar polling en background
2. **Request deduplication:** No hacer múltiples requests simultáneos
3. **Exponential backoff:** Aumentar intervalo gradualmente
4. **Server-Sent Events (futuro):** Reemplazar polling con push notifications
5. **WebSocket (futuro):** Notificaciones en tiempo real del worker

### Estrategia de Migración a WebSocket (Futuro)
```typescript
// Propuesta para futuro
class MaterialWebSocketManager {
  private ws: WebSocket | null = null;

  connect(materialId: string) {
    this.ws = new WebSocket(
      `ws://localhost:8080/v1/materials/${materialId}/processing`
    );

    this.ws.onmessage = (event) => {
      const update = JSON.parse(event.data);
      this.handleUpdate(update);
    };
  }

  private handleUpdate(update: ProcessingUpdate) {
    switch (update.type) {
      case 'processing_started':
        showProgress(0, 'Procesando PDF...');
        break;
      case 'pdf_parsed':
        showProgress(25, 'PDF analizado');
        break;
      case 'summary_generated':
        showProgress(50, 'Resumen generado');
        break;
      case 'assessment_generated':
        showProgress(75, 'Quiz generado');
        break;
      case 'completed':
        showProgress(100, 'Completado');
        this.ws?.close();
        break;
    }
  }
}
```
