# RFC-003: Cierre de Sesión

## Metadata
- **ID:** RFC-003
- **Proceso:** Autenticación
- **Subproceso:** Logout de Usuario
- **Prioridad:** Alta
- **Dependencias:** RFC-001 (Login)
- **Estado API:** ✅ Listo

## Descripción
Flujo completo de cierre de sesión del usuario, incluyendo invalidación de tokens en el servidor, limpieza de datos locales, detención de timers y redireccionamiento a la pantalla de login.

## Flujo de Usuario (UX)
1. Usuario hace clic en "Cerrar Sesión"
2. App muestra confirmación (opcional)
3. Usuario confirma cierre de sesión
4. App muestra indicador de carga breve
5. App limpia datos locales
6. App redirige a pantalla de login
7. Usuario ve pantalla de login limpia

## Flujo de Datos (Técnico)

### Diagrama de Secuencia
```
Usuario → Click "Cerrar Sesión"
              ↓
       (Confirmación opcional)
              ↓
    POST /auth/logout → API Admin
              ↓
    Invalidar refresh token en servidor
              ↓
     Frontend recibe 200 OK
              ↓
    Limpiar localStorage
              ↓
    Detener timers de refresh
              ↓
    Limpiar cache (IndexedDB)
              ↓
    Redirigir a /login
```

### Endpoints Involucrados

| Endpoint | Método | Descripción | Estado |
|----------|--------|-------------|--------|
| `/v1/auth/logout` | POST | Invalidar sesión en servidor | ✅ Listo |

### Request/Response

**Request:**
```typescript
// Sin body, solo headers de autenticación
POST /v1/auth/logout
Authorization: Bearer <access_token>
```

**Response (Éxito - 200 OK / 204 No Content):**
```typescript
// Response vacío o mensaje simple
{
  "message": "Sesión cerrada exitosamente"
}

// O simplemente 204 No Content
```

**Response (Error - 401 Unauthorized):**
```typescript
// Token ya expirado/inválido
{
  "error": "unauthorized",
  "message": "Token inválido",
  "code": "INVALID_TOKEN"
}

// NOTA: Incluso si falla, frontend debe limpiar local
```

## Estados y Transiciones

### Estados del Logout
```
┌─────────────────────────────────────────────────────────────────┐
│                    ESTADOS DEL LOGOUT                            │
└─────────────────────────────────────────────────────────────────┘

AUTHENTICATED → (Click logout) → LOGGING_OUT
                                       ↓
                          ┌────────────┴────────────┐
                          ↓                         ↓
                   API Success                  API Fail
                          ↓                         ↓
                  Limpiar local             Limpiar local igual
                          ↓                         ↓
                          └────────────┬────────────┘
                                       ↓
                              UNAUTHENTICATED
                                       ↓
                              Redirect /login
```

### Tipos de Logout

| Tipo | Trigger | Limpieza Servidor | Limpieza Local | UX |
|------|---------|-------------------|----------------|-----|
| **Manual** | Usuario hace clic | ✅ Sí | ✅ Sí | Confirmación opcional |
| **Auto (401)** | Token expirado | ❌ No | ✅ Sí | Toast de sesión expirada |
| **Forzado** | Admin desactiva cuenta | ❌ No | ✅ Sí | Modal explicativo |

## Manejo de Errores

| Código | Condición | Acción |
|--------|-----------|--------|
| 200/204 | Logout exitoso | Limpiar local + redirigir |
| 401 | Token ya inválido | Limpiar local + redirigir (ignorar error) |
| 500 | Error servidor | Limpiar local + redirigir (ignorar error) |
| Network | Sin conexión | Limpiar local + redirigir (logout local) |

**Principio:** Siempre limpiar datos locales, incluso si API falla.

## Consideraciones de UX

### Confirmación de Logout
```typescript
// Opcional: pedir confirmación solo si hay trabajo sin guardar
function shouldConfirmLogout(): boolean {
  return hasUnsavedChanges(); // Ej: progreso sin sincronizar
}

if (shouldConfirmLogout()) {
  showConfirmDialog({
    title: '¿Cerrar sesión?',
    message: 'Tienes cambios sin guardar. ¿Deseas cerrar sesión de todas formas?',
    confirmText: 'Sí, cerrar sesión',
    cancelText: 'Cancelar',
    onConfirm: () => logout()
  });
} else {
  logout(); // Sin confirmación
}
```

### Loading State
```typescript
// Breve indicador durante logout
const [loggingOut, setLoggingOut] = useState(false);

async function logout() {
  setLoggingOut(true);

  try {
    // Intentar invalidar en servidor (no crítico)
    await api.logout();
  } catch {
    // Ignorar errores
  } finally {
    // Siempre limpiar local
    cleanupSession();
    setLoggingOut(false);
    redirectToLogin();
  }
}
```

### Mensajes
- **Logout manual:** Sin mensaje (redirigir directamente)
- **Logout auto (401):** Toast "Tu sesión ha expirado"
- **Logout forzado:** Modal "Tu cuenta ha sido desactivada. Contacta soporte."

## Almacenamiento Local

### Qué Limpiar

```typescript
function cleanupSession(): void {
  // 1. Limpiar localStorage
  localStorage.removeItem('access_token');
  localStorage.removeItem('refresh_token');
  localStorage.removeItem('user');
  localStorage.removeItem('expires_at');

  // 2. Limpiar sessionStorage
  sessionStorage.clear();

  // 3. Limpiar memory cache
  cacheManager.clearMemory();

  // 4. Limpiar IndexedDB (opcional, puede preservar para próximo login)
  // cacheManager.clear(); // Descomentar si se requiere limpieza total

  // 5. Detener timers
  proactiveRefresher.stop();
  pollingManager.stopAll();
}
```

### Qué Preservar (Opcional)
- **Preferencias de UI:** Tema, idioma (NO sensible)
- **Logs de debug:** Para troubleshooting
- **Última escuela:** Para pre-llenar en próximo login

### Estrategia de Limpieza

| Dato | Acción | Razón |
|------|--------|-------|
| Tokens | ✅ Eliminar | Seguridad crítica |
| User info | ✅ Eliminar | Privacidad |
| Cache de materiales | ⚠️ Opcional | Puede preservar para offline |
| Preferencias UI | ❌ Preservar | Mejora UX, no sensible |
| Logs de error | ❌ Preservar | Para debugging |

## Código de Ejemplo (Mobile)

### Servicio de Logout
```typescript
import axios from 'axios';
import { cacheManager } from './CacheManager';
import { proactiveRefresher } from './TokenRefresher';
import { pollingManager } from './PollingManager';

class LogoutService {
  private readonly API_URL = 'http://localhost:8081/v1';

  async logout(type: 'manual' | 'auto' | 'forced' = 'manual'): Promise<void> {
    // 1. Notificar al servidor (best effort)
    await this.notifyServer().catch(() => {
      // Ignorar errores, continuar con limpieza local
      console.warn('Failed to notify server of logout');
    });

    // 2. Limpiar sesión local
    this.cleanupSession();

    // 3. Mostrar mensaje según tipo
    this.showLogoutMessage(type);

    // 4. Redirigir a login
    this.redirectToLogin();
  }

  private async notifyServer(): Promise<void> {
    const token = localStorage.getItem('access_token');

    if (!token) return;

    await axios.post(
      `${this.API_URL}/auth/logout`,
      {},
      {
        headers: {
          Authorization: `Bearer ${token}`
        },
        timeout: 3000 // Timeout corto, no es crítico
      }
    );
  }

  private cleanupSession(): void {
    // Detener servicios activos
    proactiveRefresher.stop();
    pollingManager.stopAll();

    // Limpiar storage
    const keysToRemove = [
      'access_token',
      'refresh_token',
      'user',
      'expires_at',
      'sync_queue',
      'polling_state'
    ];

    keysToRemove.forEach(key => localStorage.removeItem(key));

    // Limpiar sessionStorage
    sessionStorage.clear();

    // Limpiar memory cache
    cacheManager.clearMemory();

    // Opcional: limpiar IndexedDB
    // await cacheManager.clear();
  }

  private showLogoutMessage(type: 'manual' | 'auto' | 'forced'): void {
    switch (type) {
      case 'manual':
        // Sin mensaje
        break;

      case 'auto':
        showToast({
          message: 'Tu sesión ha expirado. Inicia sesión nuevamente.',
          type: 'warning',
          duration: 4000
        });
        break;

      case 'forced':
        showModal({
          title: 'Sesión Cerrada',
          message: 'Tu cuenta ha sido desactivada. Contacta al administrador.',
          type: 'error',
          confirmText: 'Entendido'
        });
        break;
    }
  }

  private redirectToLogin(): void {
    // Limpiar history para evitar volver atrás
    window.location.replace('/login');
  }
}

export const logoutService = new LogoutService();
```

### Hook de React
```typescript
import { useState } from 'react';
import { logoutService } from './LogoutService';

export function useLogout() {
  const [loggingOut, setLoggingOut] = useState(false);

  const logout = async (type: 'manual' | 'auto' | 'forced' = 'manual') => {
    setLoggingOut(true);

    try {
      await logoutService.logout(type);
    } catch (error) {
      console.error('Logout error:', error);
      // Continuar de todas formas
      logoutService.logout(type);
    } finally {
      setLoggingOut(false);
    }
  };

  return { logout, loggingOut };
}

// Uso en componente
function UserMenu() {
  const { logout, loggingOut } = useLogout();

  return (
    <button
      onClick={() => logout('manual')}
      disabled={loggingOut}
    >
      {loggingOut ? 'Cerrando sesión...' : 'Cerrar Sesión'}
    </button>
  );
}
```

### Componente con Confirmación
```tsx
import { useState } from 'react';
import { useLogout } from './useLogout';

export function LogoutButton() {
  const { logout, loggingOut } = useLogout();
  const [showConfirm, setShowConfirm] = useState(false);

  const handleLogoutClick = () => {
    // Verificar si hay cambios sin guardar
    if (hasUnsavedChanges()) {
      setShowConfirm(true);
    } else {
      logout('manual');
    }
  };

  const confirmLogout = () => {
    setShowConfirm(false);
    logout('manual');
  };

  return (
    <>
      <button
        onClick={handleLogoutClick}
        disabled={loggingOut}
        className="logout-button"
      >
        {loggingOut ? (
          <>
            <Spinner /> Cerrando sesión...
          </>
        ) : (
          <>
            🚪 Cerrar Sesión
          </>
        )}
      </button>

      {showConfirm && (
        <ConfirmDialog
          title="¿Cerrar sesión?"
          message="Tienes cambios sin guardar. ¿Deseas cerrar sesión de todas formas?"
          confirmText="Sí, cerrar sesión"
          cancelText="Cancelar"
          onConfirm={confirmLogout}
          onCancel={() => setShowConfirm(false)}
        />
      )}
    </>
  );
}

function hasUnsavedChanges(): boolean {
  // Verificar cola de sincronización
  const syncQueue = localStorage.getItem('sync_queue');
  if (syncQueue) {
    const queue = JSON.parse(syncQueue);
    return queue.length > 0;
  }
  return false;
}
```

### Logout Automático en 401
```typescript
// En interceptor de Axios
axios.interceptors.response.use(
  response => response,
  async (error: AxiosError) => {
    if (error.response?.status === 401) {
      const originalRequest = error.config as any;

      // No reintentar si ya intentamos refresh
      if (originalRequest._retry) {
        // Refresh falló, logout automático
        await logoutService.logout('auto');
        return Promise.reject(error);
      }

      // Primera vez, intentar refresh
      originalRequest._retry = true;

      try {
        const newToken = await tokenRefreshService.refreshAccessToken();
        originalRequest.headers.Authorization = `Bearer ${newToken}`;
        return axios(originalRequest);
      } catch (refreshError) {
        // Refresh falló, logout automático
        await logoutService.logout('auto');
        return Promise.reject(refreshError);
      }
    }

    return Promise.reject(error);
  }
);
```

### Logout desde Múltiples Tabs
```typescript
// Sincronizar logout entre tabs usando BroadcastChannel
const authChannel = new BroadcastChannel('auth_channel');

// Al hacer logout
authChannel.postMessage({ type: 'logout' });

// Escuchar en otras tabs
authChannel.onmessage = (event) => {
  if (event.data.type === 'logout') {
    // Limpiar local sin notificar servidor (ya lo hizo otra tab)
    cleanupSession();
    redirectToLogin();
  }
};
```

## Notas de Implementación

### Sincronización Previa al Logout
```typescript
async function logoutWithSync(): Promise<void> {
  // 1. Intentar sincronizar cambios pendientes
  const syncQueue = getSyncQueue();

  if (syncQueue.length > 0) {
    try {
      await syncPendingChanges();
    } catch (error) {
      // Si falla, preguntar al usuario
      const proceed = await confirmDialog(
        'Hay cambios sin sincronizar. ¿Cerrar sesión de todas formas?'
      );

      if (!proceed) return; // Cancelar logout
    }
  }

  // 2. Proceder con logout normal
  await logoutService.logout('manual');
}
```

### Gotchas Conocidos
1. **Navegación atrás:** Usar `location.replace()` en vez de `location.href`
2. **Cache del navegador:** Limpiar headers de cache en logout
3. **Service Workers:** Desregistrar service workers en logout
4. **WebSocket:** Cerrar conexiones activas
5. **Timers:** Detener todos los intervalos y timeouts

### Optimizaciones Sugeridas
1. **Logout batch:** Si múltiples tabs, solo una notifica al servidor
2. **Logout silencioso:** Para logout automático, sin animaciones
3. **Keep preferences:** Preservar tema/idioma para próximo login
4. **Analytics:** Trackear razón del logout (manual vs auto)
5. **Cleanup async:** Limpiar IndexedDB en background

### Ejemplo de Cleanup Completo
```typescript
async function completeCleanup(): Promise<void> {
  // 1. Detener servicios
  proactiveRefresher.stop();
  pollingManager.stopAll();

  // 2. Cerrar WebSockets (si aplica)
  // wsManager.closeAll();

  // 3. Limpiar storage
  localStorage.clear();
  sessionStorage.clear();

  // 4. Limpiar IndexedDB
  await cacheManager.clear();

  // 5. Desregistrar Service Workers (si aplica)
  if ('serviceWorker' in navigator) {
    const registrations = await navigator.serviceWorker.getRegistrations();
    await Promise.all(
      registrations.map(registration => registration.unregister())
    );
  }

  // 6. Limpiar cookies (si se usan)
  document.cookie.split(";").forEach(c => {
    document.cookie = c
      .replace(/^ +/, "")
      .replace(/=.*/, `=;expires=${new Date().toUTCString()};path=/`);
  });
}
```
