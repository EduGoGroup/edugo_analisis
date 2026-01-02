# RFC-002: Renovación de Tokens JWT

## Metadata
- **ID:** RFC-002
- **Proceso:** Autenticación
- **Subproceso:** Refresh de Access Token
- **Prioridad:** Alta
- **Dependencias:** RFC-001 (Login)
- **Estado API:** ✅ Listo

## Descripción
Flujo de renovación automática del access token usando el refresh token cuando está próximo a expirar o ya expiró. Permite mantener la sesión del usuario activa sin interrumpir su experiencia.

## Flujo de Usuario (UX)
1. Usuario está navegando en la aplicación
2. Access token está próximo a expirar (ej: 2 minutos antes)
3. App renueva token automáticamente en background
4. Usuario continúa su actividad sin interrupciones
5. Si refresh falla (token expirado/inválido), redirige a login

## Flujo de Datos (Técnico)

### Diagrama de Secuencia
```
Usuario navegando → App detecta token próximo a expirar
                         ↓
            ┌────────────┴────────────┐
            ↓                         ↓
     Antes de expirar           Ya expiró
    (renovar en background)    (renovar en interceptor)
            ↓                         ↓
    POST /auth/refresh ← refresh_token
            ↓
     ┌──────┴──────┐
     ↓             ↓
  SUCCESS       FAIL (401)
     ↓             ↓
Nuevo access    Logout
  token           ↓
     ↓        Redirect login
Guardar en
localStorage
     ↓
Continuar
navegación
```

### Endpoints Involucrados

| Endpoint | Método | Descripción | Estado |
|----------|--------|-------------|--------|
| `/v1/auth/refresh` | POST | Renovar access token | ✅ Listo |

### Request/Response

**Request:**
```typescript
interface RefreshRequest {
  refresh_token: string; // JWT refresh token de login
}

// Ejemplo
{
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response (Éxito - 200 OK):**
```typescript
interface RefreshResponse {
  access_token: string;   // Nuevo access token (15 min)
  expires_in: number;     // 900 segundos
  token_type: 'Bearer';
}

// Ejemplo
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expires_in": 900,
  "token_type": "Bearer"
}
```

**Response (Error - 401 Unauthorized):**
```typescript
interface RefreshError {
  error: string;
  message: string;
  code: string;
}

// Refresh token expirado
{
  "error": "unauthorized",
  "message": "Refresh token expirado. Inicia sesión nuevamente.",
  "code": "REFRESH_TOKEN_EXPIRED"
}

// Refresh token inválido
{
  "error": "unauthorized",
  "message": "Refresh token inválido",
  "code": "INVALID_REFRESH_TOKEN"
}
```

## Estados y Transiciones

### Estados del Token
```
┌─────────────────────────────────────────────────────────────────┐
│                    ESTADOS DEL ACCESS TOKEN                      │
└─────────────────────────────────────────────────────────────────┘

VALID → (Time passes) → NEAR_EXPIRY → (Refresh in background) → VALID
                             ↓
                       ┌─────┴─────┐
                       ↓           ↓
                   EXPIRED    Refresh fails
                       ↓           ↓
              (Intercept 401)  INVALID
                       ↓           ↓
              Try refresh     Logout
                  ↓   ↓
          SUCCESS  FAIL
              ↓      ↓
          VALID  Logout
```

### Estrategias de Renovación

| Estrategia | Cuándo | Ventajas | Desventajas |
|------------|--------|----------|-------------|
| **Proactive** | 2 min antes de expirar | Sin interrupciones | Requiere timer |
| **Reactive** | En interceptor 401 | Más simple | Puede fallar request |
| **Híbrida** | Ambas combinadas | Mejor UX + fallback | Más complejo |

**Recomendado:** Estrategia Híbrida

## Manejo de Errores

| Código | Condición | Acción en UI |
|--------|-----------|--------------|
| 401 | Refresh token expirado | Logout automático + redirigir a login |
| 401 | Refresh token inválido | Logout automático + redirigir a login |
| 500 | Error servidor | Reintentar 1 vez, luego logout |
| Network | Sin conexión | Mantener sesión, reintentar cuando vuelva online |

## Consideraciones de UX

### Renovación Transparente
- Usuario NO debe ver indicadores de loading
- Renovación debe ser completamente en background
- No interrumpir navegación actual

### Indicadores (Solo si falla)
```typescript
// Si refresh falla, mostrar toast antes de logout
showToast({
  message: 'Tu sesión ha expirado. Inicia sesión nuevamente.',
  type: 'warning',
  duration: 3000,
  onClose: () => logout()
});
```

### Sin Mensajes de Éxito
- Renovación exitosa: NO mostrar mensaje
- Solo notificar al usuario en caso de error

## Almacenamiento Local

### Qué Actualizar
```typescript
// Actualizar solo access_token y expires_at
localStorage.setItem('access_token', newAccessToken);

const expiresAt = Date.now() + (expiresIn * 1000);
localStorage.setItem('expires_at', expiresAt.toString());

// Refresh token NO cambia
// User info NO cambia
```

### TTL del Cache
- **Access Token:** 15 minutos (renovado periódicamente)
- **Refresh Token:** 7 días (NO se renueva automáticamente)

## Código de Ejemplo (Mobile)

### Servicio de Refresh
```typescript
import axios from 'axios';

class TokenRefreshService {
  private readonly API_URL = 'http://localhost:8081/v1';
  private refreshInProgress: Promise<string> | null = null;

  async refreshAccessToken(): Promise<string> {
    // Evitar múltiples refresh simultáneos
    if (this.refreshInProgress) {
      return this.refreshInProgress;
    }

    const refreshToken = localStorage.getItem('refresh_token');

    if (!refreshToken) {
      throw new Error('No refresh token available');
    }

    this.refreshInProgress = this.performRefresh(refreshToken);

    try {
      const newAccessToken = await this.refreshInProgress;
      return newAccessToken;
    } finally {
      this.refreshInProgress = null;
    }
  }

  private async performRefresh(refreshToken: string): Promise<string> {
    try {
      const response = await axios.post<RefreshResponse>(
        `${this.API_URL}/auth/refresh`,
        { refresh_token: refreshToken },
        {
          timeout: 5000,
          headers: {
            'Content-Type': 'application/json'
          }
        }
      );

      const { access_token, expires_in } = response.data;

      // Actualizar localStorage
      localStorage.setItem('access_token', access_token);

      const expiresAt = Date.now() + (expires_in * 1000);
      localStorage.setItem('expires_at', expiresAt.toString());

      return access_token;

    } catch (error: any) {
      // Si falla refresh, limpiar sesión
      this.handleRefreshFailure();
      throw error;
    }
  }

  private handleRefreshFailure(): void {
    // Limpiar localStorage
    localStorage.removeItem('access_token');
    localStorage.removeItem('refresh_token');
    localStorage.removeItem('user');
    localStorage.removeItem('expires_at');

    // Redirigir a login
    window.location.href = '/login';
  }

  shouldRefresh(): boolean {
    const expiresAt = localStorage.getItem('expires_at');
    if (!expiresAt) return false;

    const expiryTime = parseInt(expiresAt);
    const now = Date.now();

    // Renovar si falta menos de 2 minutos
    const twoMinutes = 2 * 60 * 1000;
    return (expiryTime - now) < twoMinutes;
  }

  getTimeUntilExpiry(): number {
    const expiresAt = localStorage.getItem('expires_at');
    if (!expiresAt) return 0;

    const expiryTime = parseInt(expiresAt);
    return Math.max(0, expiryTime - Date.now());
  }
}

export const tokenRefreshService = new TokenRefreshService();
```

### Interceptor de Axios
```typescript
import axios, { AxiosError, InternalAxiosRequestConfig } from 'axios';
import { tokenRefreshService } from './TokenRefreshService';

// Interceptor de request: agregar token
axios.interceptors.request.use(
  (config: InternalAxiosRequestConfig) => {
    const token = localStorage.getItem('access_token');

    if (token && config.headers) {
      config.headers.Authorization = `Bearer ${token}`;
    }

    return config;
  },
  (error) => Promise.reject(error)
);

// Interceptor de response: manejar 401 y refresh
axios.interceptors.response.use(
  (response) => response,
  async (error: AxiosError) => {
    const originalRequest = error.config as InternalAxiosRequestConfig & {
      _retry?: boolean;
    };

    // Solo intentar refresh una vez por request
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      try {
        // Intentar refresh
        const newToken = await tokenRefreshService.refreshAccessToken();

        // Actualizar header del request original
        if (originalRequest.headers) {
          originalRequest.headers.Authorization = `Bearer ${newToken}`;
        }

        // Reintentar request original con nuevo token
        return axios(originalRequest);

      } catch (refreshError) {
        // Refresh falló, propagar error
        return Promise.reject(refreshError);
      }
    }

    return Promise.reject(error);
  }
);
```

### Timer Proactivo
```typescript
class ProactiveTokenRefresher {
  private refreshTimer: NodeJS.Timeout | null = null;

  start(): void {
    this.scheduleNextRefresh();
  }

  stop(): void {
    if (this.refreshTimer) {
      clearTimeout(this.refreshTimer);
      this.refreshTimer = null;
    }
  }

  private scheduleNextRefresh(): void {
    const timeUntilRefresh = this.calculateRefreshTime();

    this.refreshTimer = setTimeout(async () => {
      if (tokenRefreshService.shouldRefresh()) {
        try {
          await tokenRefreshService.refreshAccessToken();
          console.log('Token refreshed proactively');
        } catch (error) {
          console.error('Proactive refresh failed:', error);
        }
      }

      // Programar siguiente verificación
      this.scheduleNextRefresh();
    }, timeUntilRefresh);
  }

  private calculateRefreshTime(): number {
    const timeUntilExpiry = tokenRefreshService.getTimeUntilExpiry();

    // Refresh 2 minutos antes de expirar
    const twoMinutes = 2 * 60 * 1000;
    const refreshAt = timeUntilExpiry - twoMinutes;

    // Mínimo 1 minuto entre verificaciones
    const oneMinute = 60 * 1000;

    return Math.max(refreshAt, oneMinute);
  }
}

export const proactiveRefresher = new ProactiveTokenRefresher();

// Iniciar al hacer login
export function startProactiveRefresh() {
  proactiveRefresher.start();
}

// Detener al hacer logout
export function stopProactiveRefresh() {
  proactiveRefresher.stop();
}
```

### Hook de React
```typescript
import { useEffect } from 'react';
import { proactiveRefresher } from './ProactiveRefresher';

export function useTokenRefresh() {
  useEffect(() => {
    // Verificar si hay sesión activa
    const hasToken = !!localStorage.getItem('access_token');

    if (hasToken) {
      proactiveRefresher.start();
    }

    // Cleanup
    return () => {
      proactiveRefresher.stop();
    };
  }, []);
}

// Uso en App
function App() {
  useTokenRefresh(); // Activar refresh proactivo

  return <Router>...</Router>;
}
```

### Componente de Prueba
```tsx
import { useState, useEffect } from 'react';
import { tokenRefreshService } from './TokenRefreshService';

export function TokenStatus() {
  const [timeLeft, setTimeLeft] = useState(0);

  useEffect(() => {
    const interval = setInterval(() => {
      const time = tokenRefreshService.getTimeUntilExpiry();
      setTimeLeft(time);
    }, 1000);

    return () => clearInterval(interval);
  }, []);

  const minutes = Math.floor(timeLeft / 60000);
  const seconds = Math.floor((timeLeft % 60000) / 1000);

  return (
    <div className="token-status">
      <span>
        Token expira en: {minutes}m {seconds}s
      </span>
    </div>
  );
}
```

## Notas de Implementación

### Estrategia Recomendada: Híbrida
```typescript
// 1. Timer proactivo (background)
proactiveRefresher.start(); // Renueva 2 min antes

// 2. Interceptor reactivo (fallback)
axios.interceptors.response.use(...); // Maneja 401
```

### Race Conditions
```typescript
// Evitar múltiples refresh simultáneos
private refreshInProgress: Promise<string> | null = null;

async refreshAccessToken(): Promise<string> {
  if (this.refreshInProgress) {
    return this.refreshInProgress; // Esperar refresh en progreso
  }

  this.refreshInProgress = this.performRefresh();

  try {
    return await this.refreshInProgress;
  } finally {
    this.refreshInProgress = null;
  }
}
```

### Gotchas Conocidos
1. **Múltiples tabs:** Cada tab intentará refresh independientemente
2. **Refresh loop:** Si refresh devuelve 401, no reintentar infinitamente
3. **Background tab:** Timer puede pausarse, usar Page Visibility API
4. **Clock skew:** Usar servidor time, no cliente
5. **Network delay:** Considerar latencia en cálculo de timing

### Optimizaciones Sugeridas
1. **Broadcast Channel API:** Sincronizar refresh entre tabs
2. **Service Worker:** Centralizar refresh para toda la app
3. **Server timing:** Backend envía tiempo restante en headers
4. **Sliding expiry:** Backend extiende TTL en cada request exitoso
5. **Silent refresh:** Iframe oculto para refresh sin interrumpir

### Ejemplo Broadcast Channel (Múltiples Tabs)
```typescript
const channel = new BroadcastChannel('auth_channel');

// Al refrescar token
channel.postMessage({
  type: 'token_refreshed',
  access_token: newToken,
  expires_at: newExpiresAt
});

// Escuchar en otras tabs
channel.onmessage = (event) => {
  if (event.data.type === 'token_refreshed') {
    localStorage.setItem('access_token', event.data.access_token);
    localStorage.setItem('expires_at', event.data.expires_at);
  }
};
```
