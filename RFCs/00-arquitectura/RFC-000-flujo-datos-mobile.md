# RFC-000: Flujo General de Datos en la App Mobile

## Metadata
- **ID:** RFC-000
- **Proceso:** Arquitectura General
- **Subproceso:** Flujo de Datos
- **Prioridad:** Alta
- **Dependencias:** Ninguna (base para todos los demás RFCs)
- **Estado API:** ✅ Listo

## Descripción
Define el flujo general de datos entre el Frontend Mobile, API Admin, API Mobile y el Worker, estableciendo los patrones de comunicación, autenticación y sincronización de estado que utilizará toda la aplicación.

## Flujo de Usuario (UX)
1. Usuario abre la aplicación
2. App verifica si existe sesión activa (JWT local)
3. Si no hay sesión, redirige a login
4. Usuario interactúa con diferentes módulos (materiales, evaluaciones, progreso)
5. App sincroniza datos con backend
6. App muestra actualizaciones en tiempo real o mediante polling

## Flujo de Datos (Técnico)

### Diagrama de Secuencia General
```
┌─────────────────────────────────────────────────────────────────┐
│                      FLUJO GENERAL DE DATOS                      │
└─────────────────────────────────────────────────────────────────┘

Usuario → Mobile App → API Admin (Auth) → JWT Token
                         ↓
Usuario → Mobile App → API Mobile (Bearer JWT) → PostgreSQL/MongoDB
                         ↓                              ↓
                   RabbitMQ Events ←── Worker procesa PDFs
                         ↓                              ↓
                   Mobile App ← Polling/Notificaciones
```

### Componentes del Ecosistema

| Componente | Responsabilidad | Puerto | Base de Datos |
|------------|----------------|--------|---------------|
| **API Admin** | Autenticación, CRUD administrativo | 8081 | PostgreSQL |
| **API Mobile** | Materiales, Evaluaciones, Progreso | 8080 | PostgreSQL + MongoDB |
| **Worker** | Procesamiento IA (PDFs, Resúmenes, Quizzes) | N/A | MongoDB |
| **Frontend Mobile** | Interfaz de usuario | N/A | Cache local |

### Endpoints Principales

**API Admin (8081):**
- `POST /v1/auth/login` - Login
- `POST /v1/auth/refresh` - Renovar token
- `POST /v1/auth/logout` - Cerrar sesión
- `GET /v1/schools` - Listar escuelas
- `GET /v1/schools/:id/units/tree` - Jerarquía académica

**API Mobile (8080):**
- `GET /v1/materials` - Listar materiales
- `POST /v1/materials` - Crear material (teacher)
- `GET /v1/materials/:id/assessment` - Obtener quiz
- `POST /v1/materials/:id/assessment/attempts` - Realizar intento
- `PUT /v1/progress` - Actualizar progreso

### Request/Response

**Autenticación (Base para todos los requests):**
```typescript
// Headers comunes
interface CommonHeaders {
  Authorization: `Bearer ${string}`; // JWT de API Admin
  'Content-Type': 'application/json';
  'X-Request-ID': string; // Para tracing
}
```

**Claims del JWT:**
```typescript
interface JWTClaims {
  sub: string; // user_id (UUID)
  email: string;
  role: 'student' | 'teacher' | 'admin' | 'super_admin';
  school_id: string; // UUID
  iss: 'edugo-central';
  exp: number; // Unix timestamp
  iat: number; // Unix timestamp
}
```

## Estados y Transiciones

### Estados de la Aplicación
```
┌─────────────────────────────────────────────────────────────────┐
│                    ESTADOS DE LA APP                             │
└─────────────────────────────────────────────────────────────────┘

UNAUTHENTICATED → (Login exitoso) → AUTHENTICATED
                ← (Logout/Token expirado) ←

AUTHENTICATED → LOADING → READY
              ↓
              ERROR → RETRY → LOADING
```

### Estados de Recursos
```typescript
type MaterialStatus =
  | 'uploaded'     // PDF subido, esperando procesamiento
  | 'processing'   // Worker procesando
  | 'ready'        // Procesamiento completo
  | 'failed';      // Error en procesamiento

type NetworkState =
  | 'idle'         // Sin requests activos
  | 'loading'      // Cargando datos
  | 'success'      // Request exitoso
  | 'error';       // Error en request
```

## Manejo de Errores

| Código | Significado | Acción en UI |
|--------|-------------|--------------|
| 200 | Éxito | Mostrar datos |
| 201 | Creado | Redirigir o actualizar lista |
| 204 | Sin contenido | Confirmar acción exitosa |
| 400 | Request inválido | Mostrar errores por campo |
| 401 | No autenticado | Redirigir a login, limpiar JWT |
| 403 | Sin permisos | Mensaje "No tienes permisos para esta acción" |
| 404 | No encontrado | Distinguir: recurso inexistente vs procesando |
| 409 | Conflicto | Mensaje "Ya existe este recurso" |
| 422 | Error de validación | Mostrar errores específicos por campo |
| 500 | Error servidor | Mensaje genérico + opción retry |
| 503 | Servicio no disponible | Mensaje de mantenimiento |

## Consideraciones de UX

### Loading States
```typescript
// Estados de carga según el tipo de operación
const LoadingStates = {
  initial: 'Cargando...', // Primera carga
  refresh: 'Actualizando...', // Refresh
  loadMore: 'Cargando más...', // Paginación
  processing: 'Procesando...', // Worker
  uploading: 'Subiendo archivo...' // Upload S3
};
```

### Skeleton Loaders
- Usar skeletons para listas y detalles
- Mantener estructura visual consistente
- Evitar "flash" de contenido vacío

### Mensajes de Error
- Específicos y accionables
- Incluir botón de "Reintentar" cuando aplique
- Mostrar tiempo estimado en procesamiento

### Confirmaciones
- Confirmar acciones destructivas (eliminar, expirar)
- Toast/Snackbar para acciones exitosas
- Modal para confirmaciones críticas

## Almacenamiento Local

### Qué Cachear
```typescript
interface CacheStrategy {
  // Datos de sesión
  auth: {
    access_token: string;
    refresh_token: string;
    user: UserInfo;
    expires_at: number;
  };

  // Cache de datos
  materials: {
    list: Material[];
    lastFetch: number;
    ttl: 5 * 60 * 1000; // 5 minutos
  };

  // Progreso de usuario
  progress: {
    [materialId: string]: {
      percentage: number;
      lastPage: number;
      lastSync: number;
    };
  };

  // Preferencias de UI
  preferences: {
    theme: 'light' | 'dark';
    language: string;
  };
}
```

### TTL del Cache
- **JWT:** Validar contra `exp` claim
- **Materiales:** 5 minutos
- **Progreso:** Sincronizar inmediatamente, cachear para offline
- **Evaluaciones:** No cachear (siempre fresh)
- **Preferencias:** Indefinido (hasta logout)

### Estrategia Offline
```typescript
// Strategy: Stale While Revalidate
async function fetchWithCache<T>(
  key: string,
  fetchFn: () => Promise<T>,
  ttl: number = 5 * 60 * 1000
): Promise<T> {
  const cached = getFromCache(key);

  if (cached && (Date.now() - cached.timestamp) < ttl) {
    // Retornar cache y refrescar en background
    fetchFn().then(data => setCache(key, data));
    return cached.data;
  }

  // Cache expirado o no existe
  const data = await fetchFn();
  setCache(key, data);
  return data;
}
```

## Código de Ejemplo (Mobile)

### Cliente API con Autenticación
```typescript
import axios, { AxiosInstance } from 'axios';

class EduGoAPIClient {
  private adminAPI: AxiosInstance;
  private mobileAPI: AxiosInstance;

  constructor() {
    this.adminAPI = axios.create({
      baseURL: 'http://localhost:8081/v1',
      timeout: 10000
    });

    this.mobileAPI = axios.create({
      baseURL: 'http://localhost:8080/v1',
      timeout: 30000 // Mayor timeout para procesamiento
    });

    // Interceptor para agregar JWT
    [this.adminAPI, this.mobileAPI].forEach(api => {
      api.interceptors.request.use(config => {
        const token = this.getToken();
        if (token) {
          config.headers.Authorization = `Bearer ${token}`;
        }
        config.headers['X-Request-ID'] = this.generateRequestId();
        return config;
      });

      // Interceptor para manejar refresh
      api.interceptors.response.use(
        response => response,
        async error => {
          if (error.response?.status === 401) {
            const refreshed = await this.refreshToken();
            if (refreshed) {
              error.config.headers.Authorization = `Bearer ${this.getToken()}`;
              return axios.request(error.config);
            }
            this.logout();
          }
          return Promise.reject(error);
        }
      );
    });
  }

  private getToken(): string | null {
    return localStorage.getItem('access_token');
  }

  private generateRequestId(): string {
    return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }

  async login(email: string, password: string) {
    const response = await this.adminAPI.post('/auth/login', {
      email,
      password
    });

    localStorage.setItem('access_token', response.data.access_token);
    localStorage.setItem('refresh_token', response.data.refresh_token);
    localStorage.setItem('user', JSON.stringify(response.data.user));

    return response.data;
  }

  async refreshToken(): Promise<boolean> {
    try {
      const refreshToken = localStorage.getItem('refresh_token');
      if (!refreshToken) return false;

      const response = await this.adminAPI.post('/auth/refresh', {
        refresh_token: refreshToken
      });

      localStorage.setItem('access_token', response.data.access_token);
      return true;
    } catch {
      return false;
    }
  }

  logout() {
    localStorage.removeItem('access_token');
    localStorage.removeItem('refresh_token');
    localStorage.removeItem('user');
    window.location.href = '/login';
  }

  // Métodos de API Mobile
  async getMaterials() {
    const response = await this.mobileAPI.get('/materials');
    return response.data;
  }

  async getMaterial(id: string) {
    const response = await this.mobileAPI.get(`/materials/${id}`);
    return response.data;
  }

  async updateProgress(materialId: string, percentage: number, lastPage: number) {
    const user = JSON.parse(localStorage.getItem('user') || '{}');
    const response = await this.mobileAPI.put('/progress', {
      user_id: user.id,
      material_id: materialId,
      progress_percentage: percentage,
      last_page: lastPage
    });
    return response.data;
  }
}

export const api = new EduGoAPIClient();
```

### Hook de React para Gestión de Estado
```typescript
import { useState, useEffect } from 'react';

interface UseAPIOptions<T> {
  onSuccess?: (data: T) => void;
  onError?: (error: Error) => void;
  initialData?: T;
}

export function useAPI<T>(
  fetchFn: () => Promise<T>,
  options: UseAPIOptions<T> = {}
) {
  const [data, setData] = useState<T | undefined>(options.initialData);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  const execute = async () => {
    try {
      setLoading(true);
      setError(null);
      const result = await fetchFn();
      setData(result);
      options.onSuccess?.(result);
      return result;
    } catch (err) {
      const error = err as Error;
      setError(error);
      options.onError?.(error);
      throw error;
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    execute();
  }, []);

  return { data, loading, error, refetch: execute };
}

// Uso
function MaterialsList() {
  const { data: materials, loading, error, refetch } = useAPI(
    () => api.getMaterials(),
    {
      onSuccess: (materials) => console.log(`Cargados ${materials.length} materiales`),
      onError: (error) => console.error('Error al cargar materiales', error)
    }
  );

  if (loading) return <Skeleton />;
  if (error) return <ErrorMessage error={error} onRetry={refetch} />;

  return <MaterialsGrid materials={materials} />;
}
```

## Notas de Implementación

### Clean Architecture en Frontend
```
┌─────────────────────────────────────────────────────────────────┐
│                    CAPAS DEL FRONTEND                            │
└─────────────────────────────────────────────────────────────────┘

Presentación (UI)
    ↓ usa
Aplicación (Hooks, Context)
    ↓ usa
Dominio (Tipos, Validaciones)
    ↓ usa
Infraestructura (API Clients, Storage)
```

### Patrones Recomendados
1. **Repository Pattern:** Abstraer llamadas a API
2. **Observer Pattern:** Para actualizaciones en tiempo real
3. **State Machine:** Para flujos complejos (upload de archivos)
4. **Optimistic Updates:** Para mejor UX (progreso, likes)

### Gotchas Conocidos
1. **Token Refresh:** Puede causar race conditions si múltiples requests fallan simultáneamente
2. **Polling:** No usar intervalos fijos, usar backoff exponencial
3. **Cache:** Invalidar cache al hacer logout
4. **Offline:** Manejar cola de sincronización pendiente

### Optimizaciones Sugeridas
1. **Request Deduplication:** No hacer requests duplicados
2. **Debounce:** Para búsquedas y autoguardado
3. **Lazy Loading:** Componentes y rutas
4. **Code Splitting:** Por módulo (admin, student, teacher)
5. **Image Optimization:** Comprimir antes de subir
6. **Bundle Size:** Analizar y reducir dependencias
