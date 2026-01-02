# RFC-004: Estrategia de Manejo de Errores HTTP

## Metadata
- **ID:** RFC-004
- **Proceso:** Arquitectura General
- **Subproceso:** Manejo de Errores
- **Prioridad:** Alta
- **Dependencias:** RFC-000
- **Estado API:** ✅ Listo

## Descripción
Define la estrategia unificada de manejo de errores HTTP en toda la aplicación mobile, estableciendo cómo capturar, transformar, mostrar y recuperarse de errores provenientes de las APIs (Admin y Mobile).

## Flujo de Usuario (UX)
1. Usuario realiza una acción (ej: crear material)
2. App envía request a API
3. Si hay error, app captura y clasifica el error
4. App muestra mensaje apropiado según el tipo de error
5. App ofrece acciones de recuperación (reintentar, cancelar, editar)
6. Usuario puede recuperarse o es redirigido según la gravedad

## Flujo de Datos (Técnico)

### Diagrama de Secuencia
```
Usuario → Frontend → API
                       ↓
                   (Error HTTP)
                       ↓
          Error Interceptor captura
                       ↓
            Clasificar tipo de error
                       ↓
        ┌──────────────┴──────────────┐
        ↓                              ↓
    Error Fatal                   Error Recoverable
  (401, 403, 500)                 (400, 404, 409, 422)
        ↓                              ↓
   Logout / Redirect             Mostrar mensaje + acción
```

### Endpoints Involucrados
Todos los endpoints de ambas APIs pueden retornar errores.

### Request/Response

**Estructura de Error de API Admin:**
```typescript
interface AdminAPIError {
  error: string;      // "unauthorized", "not_found", etc.
  message: string;    // Mensaje amigable
  code: string;       // "INVALID_CREDENTIALS", "MATERIAL_NOT_READY"
}
```

**Estructura de Error de API Mobile:**
```typescript
// Similar estructura
interface MobileAPIError {
  error: string;
  message: string;
  code: string;
  details?: Record<string, string[]>; // Errores de validación por campo
}
```

**Errores de Validación (422):**
```typescript
interface ValidationError extends MobileAPIError {
  details: {
    [field: string]: string[]; // Múltiples errores por campo
  };
}

// Ejemplo:
{
  "error": "validation_error",
  "message": "Errores de validación",
  "code": "VALIDATION_ERROR",
  "details": {
    "title": ["El título es requerido", "Debe tener al menos 3 caracteres"],
    "email": ["Email inválido"]
  }
}
```

## Estados y Transiciones

### Clasificación de Errores
```typescript
enum ErrorSeverity {
  FATAL = 'fatal',           // Requiere logout/restart
  CRITICAL = 'critical',     // Funcionalidad bloqueada
  RECOVERABLE = 'recoverable', // Usuario puede reintentar
  WARNING = 'warning'        // No bloquea flujo
}

enum ErrorCategory {
  AUTHENTICATION = 'auth',     // 401, 403
  VALIDATION = 'validation',   // 400, 422
  NOT_FOUND = 'not_found',    // 404
  CONFLICT = 'conflict',      // 409
  SERVER = 'server',          // 500, 503
  NETWORK = 'network',        // Timeout, No internet
  WORKER = 'worker'           // Dependencia del worker
}
```

### Matriz de Clasificación
| Código HTTP | Categoría | Severidad | Acción |
|-------------|-----------|-----------|--------|
| 400 | VALIDATION | RECOVERABLE | Mostrar errores + permitir editar |
| 401 | AUTHENTICATION | FATAL | Logout + redirigir a login |
| 403 | AUTHENTICATION | CRITICAL | Mostrar "Sin permisos" + volver |
| 404 | NOT_FOUND | RECOVERABLE | Distinguir: no existe vs procesando |
| 409 | CONFLICT | RECOVERABLE | Mostrar "Ya existe" + sugerir acción |
| 422 | VALIDATION | RECOVERABLE | Mostrar errores por campo |
| 500 | SERVER | CRITICAL | Mensaje genérico + reintentar |
| 503 | SERVER | CRITICAL | Mensaje mantenimiento + esperar |
| Timeout | NETWORK | RECOVERABLE | Verificar conexión + reintentar |

## Manejo de Errores

### Tabla de Errores Comunes

#### Autenticación (401)
| Código | Significado | Acción en UI |
|--------|-------------|--------------|
| 401 | Token expirado/inválido | Intentar refresh automático, si falla → logout |
| 401 | Credenciales incorrectas | Mostrar "Email o contraseña incorrectos" |
| 401 | Sin token | Redirigir a login |

#### Autorización (403)
| Código | Significado | Acción en UI |
|--------|-------------|--------------|
| 403 | Usuario inactivo | "Tu cuenta ha sido desactivada. Contacta soporte." |
| 403 | Sin permisos para recurso | "No tienes permisos para esta acción" + volver |
| 403 | Role insuficiente | "Esta función solo está disponible para [rol]" |

#### Validación (400, 422)
| Código | Significado | Acción en UI |
|--------|-------------|--------------|
| 400 | Request mal formado | Mostrar errores específicos por campo |
| 422 | Validación fallida | Resaltar campos con error + mensajes debajo |

#### No Encontrado (404)
| Código | Significado | Acción en UI |
|--------|-------------|--------------|
| 404 | Recurso no existe | "No encontramos este recurso" |
| 404 | Assessment no generado | "El quiz está siendo generado..." + polling |
| 404 | Summary no generado | "El resumen estará disponible pronto..." + polling |

#### Conflicto (409)
| Código | Significado | Acción en UI |
|--------|-------------|--------------|
| 409 | Código duplicado | "Ya existe una escuela con este código" |
| 409 | Email duplicado | "Este email ya está registrado" |

#### Servidor (500, 503)
| Código | Significado | Acción en UI |
|--------|-------------|--------------|
| 500 | Error interno | "Algo salió mal. Intenta nuevamente." + botón retry |
| 503 | Servicio no disponible | "Sistema en mantenimiento. Intenta más tarde." |

#### Network Errors
| Tipo | Significado | Acción en UI |
|------|-------------|--------------|
| Timeout | Request tardó mucho | "La conexión es lenta. ¿Reintentar?" |
| Network Error | Sin internet | "Verifica tu conexión a internet" + modo offline |

## Consideraciones de UX

### Mensajes de Error
```typescript
const ErrorMessages = {
  // Genéricos
  generic: 'Algo salió mal. Por favor intenta nuevamente.',
  network: 'Verifica tu conexión a internet',
  timeout: 'La operación está tardando más de lo esperado',

  // Autenticación
  invalidCredentials: 'Email o contraseña incorrectos',
  sessionExpired: 'Tu sesión ha expirado. Inicia sesión nuevamente.',
  unauthorized: 'No tienes permisos para realizar esta acción',

  // Validación
  requiredField: (field: string) => `${field} es requerido`,
  minLength: (field: string, min: number) =>
    `${field} debe tener al menos ${min} caracteres`,
  invalidEmail: 'Email inválido',

  // Recursos
  notFound: 'No encontramos este recurso',
  alreadyExists: 'Este recurso ya existe',

  // Worker
  assessmentProcessing: 'El quiz está siendo generado. Intenta en unos minutos.',
  summaryProcessing: 'El resumen estará disponible pronto.',
  materialFailed: 'Error al procesar el material. Contacta soporte.'
};
```

### Estados de Carga
```typescript
interface LoadingState {
  isLoading: boolean;
  message: string;
  canCancel: boolean;
}

// Ejemplos
const LoadingStates = {
  login: { isLoading: true, message: 'Iniciando sesión...', canCancel: false },
  uploadPDF: { isLoading: true, message: 'Subiendo archivo...', canCancel: true },
  processing: { isLoading: true, message: 'Procesando con IA...', canCancel: false }
};
```

### Confirmaciones
- Errores fatales (401): Modal con botón "Aceptar" → logout
- Errores críticos (500): Toast con botón "Reintentar"
- Errores recuperables (422): Inline en formulario
- Warnings: Toast dismissible

## Almacenamiento Local

### Qué NO Cachear
- Errores (nunca persistir)
- Requests fallidos (registrar en log, no guardar)

### Logging de Errores
```typescript
interface ErrorLog {
  timestamp: number;
  severity: ErrorSeverity;
  category: ErrorCategory;
  statusCode?: number;
  message: string;
  endpoint: string;
  userId?: string;
  requestId: string;
  stack?: string;
}

// Persistir últimos 50 errores para debugging
const errorLogger = {
  log(error: ErrorLog) {
    const logs = this.getLogs();
    logs.unshift(error);
    if (logs.length > 50) logs.pop();
    localStorage.setItem('error_logs', JSON.stringify(logs));
  },

  getLogs(): ErrorLog[] {
    return JSON.parse(localStorage.getItem('error_logs') || '[]');
  },

  clear() {
    localStorage.removeItem('error_logs');
  }
};
```

### TTL del Cache
N/A - Los errores no se cachean

### Estrategia Offline
```typescript
// Detectar offline
window.addEventListener('offline', () => {
  showOfflineBanner();
  queuePendingRequests();
});

window.addEventListener('online', () => {
  hideOfflineBanner();
  syncPendingRequests();
});
```

## Código de Ejemplo (Mobile)

### Error Interceptor
```typescript
import axios, { AxiosError } from 'axios';

interface APIErrorResponse {
  error: string;
  message: string;
  code: string;
  details?: Record<string, string[]>;
}

class ErrorHandler {
  private static instance: ErrorHandler;

  private constructor() {}

  static getInstance(): ErrorHandler {
    if (!ErrorHandler.instance) {
      ErrorHandler.instance = new ErrorHandler();
    }
    return ErrorHandler.instance;
  }

  handle(error: AxiosError<APIErrorResponse>): never {
    // Log error
    this.logError(error);

    // Clasificar y transformar
    const transformedError = this.transformError(error);

    // Acciones automáticas
    this.performAutomaticActions(transformedError);

    // Propagar error transformado
    throw transformedError;
  }

  private transformError(error: AxiosError<APIErrorResponse>): AppError {
    if (!error.response) {
      // Network error
      return new AppError({
        category: ErrorCategory.NETWORK,
        severity: ErrorSeverity.RECOVERABLE,
        message: error.code === 'ECONNABORTED'
          ? ErrorMessages.timeout
          : ErrorMessages.network,
        canRetry: true
      });
    }

    const { status, data } = error.response;

    switch (status) {
      case 400:
      case 422:
        return new AppError({
          category: ErrorCategory.VALIDATION,
          severity: ErrorSeverity.RECOVERABLE,
          message: data?.message || 'Errores de validación',
          details: data?.details,
          canRetry: false
        });

      case 401:
        return new AppError({
          category: ErrorCategory.AUTHENTICATION,
          severity: ErrorSeverity.FATAL,
          message: data?.message || ErrorMessages.sessionExpired,
          canRetry: false
        });

      case 403:
        return new AppError({
          category: ErrorCategory.AUTHENTICATION,
          severity: ErrorSeverity.CRITICAL,
          message: data?.message || ErrorMessages.unauthorized,
          canRetry: false
        });

      case 404:
        // Distinguir entre recurso no existe vs procesando
        const isWorkerDependent = this.isWorkerDependentEndpoint(
          error.config?.url || ''
        );

        return new AppError({
          category: isWorkerDependent ? ErrorCategory.WORKER : ErrorCategory.NOT_FOUND,
          severity: ErrorSeverity.RECOVERABLE,
          message: isWorkerDependent
            ? this.getWorkerMessage(error.config?.url || '')
            : ErrorMessages.notFound,
          canRetry: isWorkerDependent
        });

      case 409:
        return new AppError({
          category: ErrorCategory.CONFLICT,
          severity: ErrorSeverity.RECOVERABLE,
          message: data?.message || ErrorMessages.alreadyExists,
          canRetry: false
        });

      case 500:
      case 503:
        return new AppError({
          category: ErrorCategory.SERVER,
          severity: ErrorSeverity.CRITICAL,
          message: status === 503
            ? 'Sistema en mantenimiento'
            : ErrorMessages.generic,
          canRetry: true
        });

      default:
        return new AppError({
          category: ErrorCategory.SERVER,
          severity: ErrorSeverity.CRITICAL,
          message: ErrorMessages.generic,
          canRetry: true
        });
    }
  }

  private isWorkerDependentEndpoint(url: string): boolean {
    return url.includes('/assessment') || url.includes('/summary');
  }

  private getWorkerMessage(url: string): string {
    if (url.includes('/assessment')) {
      return ErrorMessages.assessmentProcessing;
    }
    if (url.includes('/summary')) {
      return ErrorMessages.summaryProcessing;
    }
    return ErrorMessages.generic;
  }

  private performAutomaticActions(error: AppError) {
    if (error.severity === ErrorSeverity.FATAL) {
      // Logout automático
      localStorage.clear();
      window.location.href = '/login';
    }
  }

  private logError(error: AxiosError<APIErrorResponse>) {
    errorLogger.log({
      timestamp: Date.now(),
      severity: this.getSeverity(error.response?.status || 0),
      category: this.getCategory(error.response?.status || 0),
      statusCode: error.response?.status,
      message: error.response?.data?.message || error.message,
      endpoint: error.config?.url || 'unknown',
      userId: this.getUserId(),
      requestId: error.config?.headers?.['X-Request-ID'] as string,
      stack: error.stack
    });
  }

  private getSeverity(status: number): ErrorSeverity {
    if (status === 401) return ErrorSeverity.FATAL;
    if ([403, 500, 503].includes(status)) return ErrorSeverity.CRITICAL;
    return ErrorSeverity.RECOVERABLE;
  }

  private getCategory(status: number): ErrorCategory {
    if ([401, 403].includes(status)) return ErrorCategory.AUTHENTICATION;
    if ([400, 422].includes(status)) return ErrorCategory.VALIDATION;
    if (status === 404) return ErrorCategory.NOT_FOUND;
    if (status === 409) return ErrorCategory.CONFLICT;
    if ([500, 503].includes(status)) return ErrorCategory.SERVER;
    return ErrorCategory.SERVER;
  }

  private getUserId(): string | undefined {
    try {
      const user = JSON.parse(localStorage.getItem('user') || '{}');
      return user.id;
    } catch {
      return undefined;
    }
  }
}

// Clase de Error personalizada
class AppError extends Error {
  category: ErrorCategory;
  severity: ErrorSeverity;
  canRetry: boolean;
  details?: Record<string, string[]>;

  constructor(options: {
    category: ErrorCategory;
    severity: ErrorSeverity;
    message: string;
    canRetry: boolean;
    details?: Record<string, string[]>;
  }) {
    super(options.message);
    this.category = options.category;
    this.severity = options.severity;
    this.canRetry = options.canRetry;
    this.details = options.details;
    this.name = 'AppError';
  }
}

// Configurar interceptor
axios.interceptors.response.use(
  response => response,
  error => ErrorHandler.getInstance().handle(error)
);
```

### Hook de React para Manejo de Errores
```typescript
import { useState } from 'react';
import { AppError, ErrorCategory, ErrorSeverity } from './ErrorHandler';

interface UseErrorHandlerReturn {
  error: AppError | null;
  setError: (error: AppError | null) => void;
  clearError: () => void;
  retry?: () => void;
}

export function useErrorHandler(
  retryFn?: () => void
): UseErrorHandlerReturn {
  const [error, setError] = useState<AppError | null>(null);

  const clearError = () => setError(null);

  const retry = retryFn ? () => {
    clearError();
    retryFn();
  } : undefined;

  return { error, setError, clearError, retry };
}

// Componente de UI para mostrar errores
function ErrorDisplay({
  error,
  onRetry,
  onDismiss
}: {
  error: AppError;
  onRetry?: () => void;
  onDismiss: () => void;
}) {
  const isInline = error.category === ErrorCategory.VALIDATION;

  if (isInline) {
    return (
      <div className="validation-errors">
        {Object.entries(error.details || {}).map(([field, errors]) => (
          <div key={field} className="field-error">
            <span className="field-name">{field}:</span>
            <ul>
              {errors.map((msg, i) => (
                <li key={i}>{msg}</li>
              ))}
            </ul>
          </div>
        ))}
      </div>
    );
  }

  return (
    <div className={`error-toast severity-${error.severity}`}>
      <div className="error-icon">⚠️</div>
      <div className="error-content">
        <h4>{error.message}</h4>
        {error.canRetry && onRetry && (
          <button onClick={onRetry}>Reintentar</button>
        )}
        <button onClick={onDismiss}>Cerrar</button>
      </div>
    </div>
  );
}
```

## Notas de Implementación

### Estrategia de Retry
```typescript
async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  maxRetries: number = 3,
  baseDelay: number = 1000
): Promise<T> {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (i === maxRetries - 1) throw error;

      // Backoff exponencial
      const delay = baseDelay * Math.pow(2, i);
      await sleep(delay);
    }
  }
  throw new Error('Max retries reached');
}

// Uso
const materials = await retryWithBackoff(
  () => api.getMaterials(),
  3, // max 3 intentos
  1000 // 1s, 2s, 4s
);
```

### Gotchas Conocidos
1. **401 en múltiples requests simultáneos:** Puede intentar refresh múltiples veces. Usar lock/semáforo.
2. **404 en worker endpoints:** Requiere lógica especial para distinguir de errores reales.
3. **Network errors:** No tienen response, validar `error.response` antes de acceder.
4. **CORS errors:** Aparecen como network errors, difícil distinguir.

### Optimizaciones Sugeridas
1. **Debounce de errores:** No mostrar el mismo error múltiples veces en corto tiempo
2. **Circuit breaker:** Dejar de intentar temporalmente si servicio está caído
3. **Queue de errores:** Mostrar uno a la vez, no todos simultáneamente
4. **Error boundaries (React):** Capturar errores de renderizado
5. **Sentry/Error tracking:** Enviar errores críticos a servicio de monitoreo
