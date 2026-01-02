# RFC-041: Manejo de Estado "Procesando" con Polling

## Metadata
- **ID:** RFC-041
- **Proceso:** Resúmenes IA / Evaluaciones
- **Subproceso:** Estrategia de Polling para Contenido en Procesamiento
- **Prioridad:** Alta
- **Dependencias:** RFC-030 (Obtener Quiz), RFC-040 (Obtener Resumen), Worker
- **Estado API:** ✅ Listo (infraestructura soporta polling)

## Descripción

Define la estrategia unificada para manejar el estado "procesando" de materiales cuyo contenido IA (resumen/quiz) aún no está generado. Implementa polling inteligente, UX progresiva y degradación graceful.

## Flujo de Usuario (UX)

### Progresión Visual del Estado

```
Estado 1: Material Subido (uploaded)
┌──────────────────────────────────────┐
│  📄 Material cargado                  │
│  El PDF se subió correctamente       │
│  Procesamiento iniciará pronto...    │
│                                      │
│  [Ver PDF]  [Volver]                 │
└──────────────────────────────────────┘

Estado 2: Iniciando Procesamiento (processing - 0%)
┌──────────────────────────────────────┐
│  ⚙️ Iniciando procesamiento...        │
│  ░░░░░░░░░░░░░░░░░░░░ 0%            │
│  Preparando análisis del documento   │
│                                      │
│  Tiempo estimado: ~2 minutos         │
└──────────────────────────────────────┘

Estado 3: Extrayendo Texto (processing - 25%)
┌──────────────────────────────────────┐
│  📖 Extrayendo texto del PDF...       │
│  █████░░░░░░░░░░░░░░░ 25%           │
│  Analizando 10 páginas...            │
│                                      │
│  Tiempo restante: ~1.5 minutos       │
└──────────────────────────────────────┘

Estado 4: Generando Resumen (processing - 50%)
┌──────────────────────────────────────┐
│  🤖 Generando resumen con IA...       │
│  ██████████░░░░░░░░░░ 50%           │
│  Identificando conceptos clave...    │
│                                      │
│  Tiempo restante: ~1 minuto          │
└──────────────────────────────────────┘

Estado 5: Generando Quiz (processing - 75%)
┌──────────────────────────────────────┐
│  📝 Generando preguntas de evaluación...│
│  ███████████████░░░░░ 75%           │
│  Creando quiz interactivo...         │
│                                      │
│  Tiempo restante: ~30 segundos       │
└──────────────────────────────────────┘

Estado 6: Finalizando (processing - 95%)
┌──────────────────────────────────────┐
│  ✨ Finalizando...                    │
│  ███████████████████░ 95%           │
│  Guardando contenido generado...     │
│                                      │
│  Tiempo restante: ~15 segundos       │
└──────────────────────────────────────┘

Estado 7: Listo (ready)
┌──────────────────────────────────────┐
│  ✅ Material procesado exitosamente   │
│  ████████████████████ 100%          │
│  Contenido disponible para estudio   │
│                                      │
│  [Ver Resumen]  [Tomar Quiz]         │
└──────────────────────────────────────┘
```

## Flujo de Datos (Técnico)

### Estrategia de Polling Inteligente

```typescript
interface PollingConfig {
  // Intervalos de polling
  initialInterval: number;      // 3000ms (3s) - inicio rápido
  normalInterval: number;       // 5000ms (5s) - normal
  slowInterval: number;         // 10000ms (10s) - después de 2min

  // Límites
  maxAttempts: number;          // 120 intentos
  maxDuration: number;          // 600000ms (10min)

  // Backoff
  useExponentialBackoff: boolean; // false (lineal)
  backoffMultiplier: number;     // 1.5 (si exponencial)

  // Callbacks
  onProgress?: (attempt: number, max: number, percentage: number) => void;
  onStateChange?: (oldState: string, newState: string) => void;
  onReady?: () => void;
  onTimeout?: () => void;
  onError?: (error: Error) => void;
}

const DEFAULT_POLLING_CONFIG: PollingConfig = {
  initialInterval: 3000,
  normalInterval: 5000,
  slowInterval: 10000,
  maxAttempts: 120,
  maxDuration: 600000,
  useExponentialBackoff: false,
  backoffMultiplier: 1.0
};
```

### Diagrama de Flujo de Polling

```
┌─────────────────────┐
│ Detectar material   │
│ en "processing"     │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Iniciar polling     │
│ interval: 3s        │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ GET /materials/:id  │
│ Verificar status    │
└──────────┬──────────┘
           │
     ┌─────┴─────┐
     │           │
status='ready'  status='processing'
     │           │
     ▼           ▼
┌─────────┐  ┌──────────────┐
│ Mostrar │  │ Calcular     │
│contenido│  │ progreso     │
└─────────┘  │ estimado     │
             └──────┬───────┘
                    │
                    ▼
             ┌──────────────┐
             │ Actualizar   │
             │ UI con %     │
             └──────┬───────┘
                    │
                    ▼
             ┌──────────────┐
             │ Esperar      │
             │ interval     │
             └──────┬───────┘
                    │
                    ▼
             ┌──────────────┐
             │ Intentos++   │
             └──────┬───────┘
                    │
          ┌─────────┴─────────┐
          │                   │
  intentos < max     intentos >= max
          │                   │
          ▼                   ▼
    [Continuar]         ┌──────────┐
          │             │ Timeout  │
          └─────────────┤ Error    │
                        └──────────┘
```

## Estados y Transiciones

### Estados del Material

```typescript
type MaterialStatus = 'uploaded' | 'processing' | 'ready' | 'failed';

interface MaterialState {
  status: MaterialStatus;
  processing_started_at?: string;
  processing_completed_at?: string;
  estimated_completion_time?: number; // segundos
}
```

### Transiciones de Estado

```
uploaded → processing (Worker inicia)
  ↓
processing → ready (Worker completa exitosamente)
  ↓ (si falla)
processing → failed (Error en Worker)
```

## Código de Ejemplo (Mobile - TypeScript)

### Servicio de Polling Unificado

```typescript
// services/PollingService.ts

export class PollingService<T> {
  private config: PollingConfig;
  private attempts: number = 0;
  private startTime: number = 0;
  private currentInterval: number;
  private abortController?: AbortController;

  constructor(config: Partial<PollingConfig> = {}) {
    this.config = { ...DEFAULT_POLLING_CONFIG, ...config };
    this.currentInterval = this.config.initialInterval;
  }

  /**
   * Iniciar polling hasta que el recurso esté listo
   */
  async poll(
    fetchFn: () => Promise<T>,
    isReadyFn: (data: T) => boolean
  ): Promise<T> {

    this.attempts = 0;
    this.startTime = Date.now();
    this.abortController = new AbortController();

    return this.executePoll(fetchFn, isReadyFn);
  }

  private async executePoll(
    fetchFn: () => Promise<T>,
    isReadyFn: (data: T) => boolean
  ): Promise<T> {

    while (this.shouldContinue()) {
      try {
        // Verificar abort
        if (this.abortController?.signal.aborted) {
          throw new PollingAbortedError('Polling was cancelled');
        }

        // Fetch data
        const data = await fetchFn();

        // Calcular progreso
        const percentage = this.calculateProgress();
        this.config.onProgress?.(this.attempts, this.config.maxAttempts, percentage);

        // Verificar si está listo
        if (isReadyFn(data)) {
          this.config.onReady?.();
          return data;
        }

        // Incrementar intentos
        this.attempts++;

        // Ajustar intervalo según tiempo transcurrido
        this.adjustInterval();

        // Esperar siguiente intento
        await this.sleep(this.currentInterval);

      } catch (error: any) {
        if (error instanceof PollingAbortedError) {
          throw error;
        }

        // Propagar otros errores
        this.config.onError?.(error);
        throw error;
      }
    }

    // Timeout alcanzado
    const timeoutError = new PollingTimeoutError(
      `Polling timed out after ${this.attempts} attempts`
    );
    this.config.onTimeout?.();
    throw timeoutError;
  }

  private shouldContinue(): boolean {
    // Verificar límite de intentos
    if (this.attempts >= this.config.maxAttempts) {
      return false;
    }

    // Verificar límite de duración
    const elapsed = Date.now() - this.startTime;
    if (elapsed >= this.config.maxDuration) {
      return false;
    }

    return true;
  }

  private adjustInterval() {
    const elapsed = Date.now() - this.startTime;

    // Primeros 30s: intervalo rápido
    if (elapsed < 30000) {
      this.currentInterval = this.config.initialInterval;
    }
    // 30s - 2min: intervalo normal
    else if (elapsed < 120000) {
      this.currentInterval = this.config.normalInterval;
    }
    // Después de 2min: intervalo lento
    else {
      this.currentInterval = this.config.slowInterval;
    }

    // Aplicar backoff si configurado
    if (this.config.useExponentialBackoff) {
      this.currentInterval *= this.config.backoffMultiplier;
    }
  }

  private calculateProgress(): number {
    const elapsed = Date.now() - this.startTime;
    const estimatedDuration = 120000; // 2 minutos estimado

    // Progreso estimado basado en tiempo
    const timeProgress = Math.min((elapsed / estimatedDuration) * 100, 95);

    // Combinar con intentos
    const attemptProgress = (this.attempts / this.config.maxAttempts) * 100;

    // Usar el mayor (más optimista)
    return Math.min(Math.max(timeProgress, attemptProgress), 95);
  }

  abort() {
    this.abortController?.abort();
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// Custom Errors
export class PollingTimeoutError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'PollingTimeoutError';
  }
}

export class PollingAbortedError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'PollingAbortedError';
  }
}
```

### Hook Unificado: usePolling

```typescript
// hooks/usePolling.ts

interface UsePollingOptions<T> {
  fetchFn: () => Promise<T>;
  isReadyFn: (data: T) => boolean;
  enabled?: boolean;
  config?: Partial<PollingConfig>;
}

interface UsePollingReturn<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
  progress: number;
  isReady: boolean;
  cancel: () => void;
  retry: () => void;
}

export function usePolling<T>(options: UsePollingOptions<T>): UsePollingReturn<T> {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  const [progress, setProgress] = useState(0);
  const [isReady, setIsReady] = useState(false);

  const pollingServiceRef = useRef<PollingService<T>>();

  const startPolling = useCallback(async () => {
    setLoading(true);
    setError(null);
    setProgress(0);
    setIsReady(false);

    const service = new PollingService<T>({
      ...options.config,
      onProgress: (attempt, max, percentage) => {
        setProgress(percentage);
      },
      onReady: () => {
        setIsReady(true);
      },
      onError: (err) => {
        setError(err);
      }
    });

    pollingServiceRef.current = service;

    try {
      const result = await service.poll(options.fetchFn, options.isReadyFn);
      setData(result);
      setProgress(100);
    } catch (err: any) {
      setError(err);
    } finally {
      setLoading(false);
    }
  }, [options]);

  useEffect(() => {
    if (options.enabled) {
      startPolling();
    }

    return () => {
      pollingServiceRef.current?.abort();
    };
  }, [options.enabled, startPolling]);

  return {
    data,
    loading,
    error,
    progress,
    isReady,
    cancel: () => pollingServiceRef.current?.abort(),
    retry: startPolling
  };
}
```

### Hook: useMaterialProcessing

```typescript
// hooks/useMaterialProcessing.ts

export function useMaterialProcessing(materialId: string) {
  const [material, setMaterial] = useState<MaterialResponse | null>(null);

  const {
    data,
    loading,
    error,
    progress,
    isReady,
    cancel,
    retry
  } = usePolling<MaterialResponse>({
    fetchFn: async () => {
      const service = new MaterialService(process.env.REACT_APP_API_MOBILE_URL!);
      return await service.getMaterial(materialId);
    },
    isReadyFn: (mat) => mat.status === 'ready',
    enabled: true,
    config: {
      initialInterval: 3000,
      normalInterval: 5000,
      slowInterval: 10000,
      maxAttempts: 120,
      maxDuration: 600000
    }
  });

  useEffect(() => {
    if (data) {
      setMaterial(data);
    }
  }, [data]);

  return {
    material,
    loading,
    error,
    progress,
    isProcessing: material?.status === 'processing',
    isReady: material?.status === 'ready',
    hasFailed: material?.status === 'failed',
    cancel,
    retry
  };
}
```

### Componente: ProcessingIndicator

```typescript
// components/ProcessingIndicator.tsx

export const ProcessingIndicator: React.FC<{
  progress: number;
  status: MaterialStatus;
}> = ({ progress, status }) => {

  const getMessage = () => {
    if (progress < 25) return 'Iniciando procesamiento...';
    if (progress < 50) return 'Extrayendo texto del PDF...';
    if (progress < 75) return 'Generando resumen con IA...';
    if (progress < 95) return 'Generando preguntas de evaluación...';
    return 'Finalizando...';
  };

  const getEstimatedTime = () => {
    const remaining = ((100 - progress) / 100) * 120; // 2 min total estimado
    if (remaining < 30) return '~15 segundos';
    if (remaining < 60) return '~30 segundos';
    if (remaining < 90) return '~1 minuto';
    return `~${Math.ceil(remaining / 60)} minutos`;
  };

  return (
    <div className="processing-indicator">
      <div className="processing-icon">
        {status === 'processing' && <SpinnerIcon />}
        {status === 'ready' && <CheckIcon />}
        {status === 'failed' && <ErrorIcon />}
      </div>

      <div className="processing-content">
        <h3>{getMessage()}</h3>

        <div className="progress-bar">
          <div
            className="progress-fill"
            style={{ width: `${progress}%` }}
          />
        </div>

        <div className="progress-info">
          <span>{Math.round(progress)}%</span>
          {status === 'processing' && (
            <span className="time-remaining">
              Tiempo restante: {getEstimatedTime()}
            </span>
          )}
        </div>

        {status === 'failed' && (
          <p className="error-message">
            Error al procesar el material. Intenta nuevamente.
          </p>
        )}
      </div>
    </div>
  );
};
```

### Componente: MaterialContentLoader

```typescript
// components/MaterialContentLoader.tsx

export const MaterialContentLoader: React.FC<{ materialId: string }> = ({ materialId }) => {
  const {
    material,
    loading,
    error,
    progress,
    isProcessing,
    isReady,
    hasFailed,
    cancel,
    retry
  } = useMaterialProcessing(materialId);

  // Mostrar skeleton inicial
  if (loading && !material) {
    return <MaterialSkeleton />;
  }

  // Error al cargar material
  if (error && !material) {
    return <ErrorMessage error={error} onRetry={retry} />;
  }

  // Material cargado pero procesando
  if (material && isProcessing) {
    return (
      <div className="material-processing">
        <ProcessingIndicator
          progress={progress}
          status={material.status}
        />

        <div className="processing-actions">
          <button onClick={cancel}>Cancelar</button>
          <p>Puedes cerrar esta página y volver más tarde</p>
        </div>
      </div>
    );
  }

  // Material procesado con error
  if (material && hasFailed) {
    return (
      <div className="material-failed">
        <ErrorIcon />
        <h2>Error al procesar material</h2>
        <p>No fue posible generar el contenido IA</p>
        <button onClick={retry}>Reintentar Procesamiento</button>
      </div>
    );
  }

  // Material listo
  if (material && isReady) {
    return (
      <MaterialContent material={material} />
    );
  }

  return null;
};
```

## Consideraciones de UX

### 1. Notificaciones Push

```typescript
// Opcional: Notificar cuando complete
async function setupProcessingNotification(materialId: string) {
  if ('Notification' in window && Notification.permission === 'granted') {
    const service = new PollingService<MaterialResponse>({
      onReady: () => {
        new Notification('Material Listo', {
          body: 'El procesamiento de tu material ha finalizado',
          icon: '/icons/material-ready.png',
          tag: `material-${materialId}`,
          requireInteraction: false
        });
      }
    });

    await service.poll(
      () => getMaterial(materialId),
      (mat) => mat.status === 'ready'
    );
  }
}
```

### 2. Background Polling

```typescript
// Continuar polling en background tab
document.addEventListener('visibilitychange', () => {
  if (document.hidden) {
    // Tab oculto, reducir frecuencia de polling
    pollingService.updateConfig({ normalInterval: 15000 });
  } else {
    // Tab visible, restaurar frecuencia normal
    pollingService.updateConfig({ normalInterval: 5000 });
  }
});
```

### 3. Persistencia de Estado

```typescript
// Guardar estado de procesamiento en localStorage
function saveProcessingState(materialId: string, state: ProcessingState) {
  localStorage.setItem(
    `processing_${materialId}`,
    JSON.stringify({
      ...state,
      savedAt: Date.now()
    })
  );
}

// Recuperar al recargar página
function loadProcessingState(materialId: string): ProcessingState | null {
  const saved = localStorage.getItem(`processing_${materialId}`);
  if (!saved) return null;

  const state = JSON.parse(saved);

  // Si hace más de 10 minutos, descartar
  if (Date.now() - state.savedAt > 600000) {
    localStorage.removeItem(`processing_${materialId}`);
    return null;
  }

  return state;
}
```

## Manejo de Errores

### Timeout de Polling

```typescript
if (error instanceof PollingTimeoutError) {
  showModal({
    title: 'Procesamiento tardando más de lo esperado',
    message: 'El material puede tardar más tiempo del estimado. ¿Deseas continuar esperando?',
    actions: [
      {
        label: 'Seguir Esperando',
        onClick: () => {
          // Reiniciar polling con límites extendidos
          retryWithExtendedTimeout();
        }
      },
      {
        label: 'Volver Después',
        onClick: () => {
          // Guardar estado y redirigir
          saveState();
          redirectTo('/materials');
        }
      }
    ]
  });
}
```

### Material Falló

```typescript
if (material.status === 'failed') {
  showErrorMessage({
    title: 'Error al procesar material',
    message: 'No fue posible generar el resumen y evaluación',
    actions: [
      {
        label: 'Ver PDF Original',
        onClick: () => redirectTo(`/materials/${materialId}/pdf`)
      },
      {
        label: 'Reportar Problema',
        onClick: () => openSupportChat()
      }
    ]
  });
}
```

## Notas de Implementación

### 1. Optimización de Requests

**Evitar sobrecarga:**
- Usar intervalos adaptativos (3s → 5s → 10s)
- Limitar intentos máximos (120)
- Cancelar polling al navegar fuera

### 2. Monitoreo

**Métricas a trackear:**
- Tiempo promedio de procesamiento
- Tasa de timeout
- Distribución de tiempos por tamaño de PDF

### 3. Testing

```typescript
describe('PollingService', () => {
  it('should poll until ready', async () => {
    const service = new PollingService();
    let attempts = 0;

    const result = await service.poll(
      async () => {
        attempts++;
        return { status: attempts >= 3 ? 'ready' : 'processing' };
      },
      (data) => data.status === 'ready'
    );

    expect(result.status).toBe('ready');
    expect(attempts).toBe(3);
  });

  it('should timeout after max attempts', async () => {
    const service = new PollingService({ maxAttempts: 5 });

    await expect(
      service.poll(
        async () => ({ status: 'processing' }),
        () => false
      )
    ).rejects.toThrow(PollingTimeoutError);
  });
});
```

### 4. Accesibilidad

```typescript
<div
  role="status"
  aria-live="polite"
  aria-atomic="true"
  aria-label={`Procesamiento al ${Math.round(progress)}%`}
>
  <div aria-hidden="true">{getMessage()}</div>
</div>
```
