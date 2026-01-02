# RFC-004: Validación de Sesión Activa

## Metadata
- **ID:** RFC-004
- **Proceso:** Autenticación
- **Subproceso:** Validación de Sesión
- **Prioridad:** Alta
- **Dependencias:** RFC-001 (Login), RFC-002 (Refresh)
- **Estado API:** ✅ Listo

## Descripción
Mecanismo para verificar continuamente que la sesión del usuario está activa y válida, protegiendo rutas privadas, validando permisos según rol y manejando casos de sesión expirada o invalidada.

## Flujo de Usuario (UX)
1. Usuario accede a cualquier página de la app
2. App verifica sesión activa antes de renderizar
3. Si sesión válida: muestra contenido
4. Si sesión expirada/inválida: redirige a login
5. Usuario navega entre páginas protegidas sin interrupciones
6. Si sesión expira durante navegación: mensaje + redirect

## Flujo de Datos (Técnico)

### Diagrama de Secuencia
```
Usuario → Accede a ruta protegida
              ↓
    Verificar token en localStorage
              ↓
      ┌───────┴────────┐
      ↓                ↓
   Token existe    Token no existe
      ↓                ↓
Validar expiración   Redirect /login
      ↓
  ┌───┴───┐
  ↓       ↓
Valid   Expired
  ↓       ↓
Render  Refresh
Content    ↓
      ┌────┴────┐
      ↓         ↓
  Success     Fail
      ↓         ↓
  Render    Redirect
  Content    /login
```

### Endpoints Involucrados

| Endpoint | Método | Descripción | Estado | Uso |
|----------|--------|-------------|--------|-----|
| `/v1/auth/verify` | POST | Validar token (backend-to-backend) | ✅ | Opcional |
| Todos los endpoints | ANY | Retornan 401 si token inválido | ✅ | Primario |

**NOTA:** La validación principal es **local** (verificar exp claim del JWT). El endpoint `/v1/auth/verify` es para validación entre servicios, no necesario en mobile.

### Request/Response

**Validación Local (Recomendado):**
```typescript
// Decodificar JWT localmente
import jwtDecode from 'jwt-decode';

interface JWTPayload {
  sub: string;
  email: string;
  role: string;
  school_id: string;
  exp: number; // Unix timestamp
  iat: number;
}

function isTokenValid(token: string): boolean {
  try {
    const payload = jwtDecode<JWTPayload>(token);

    // Verificar expiración
    const now = Math.floor(Date.now() / 1000);
    return payload.exp > now;

  } catch {
    return false;
  }
}
```

**Validación Remota (Opcional):**
```typescript
// Solo para casos especiales (ej: validar permisos específicos)
POST /v1/auth/verify
Authorization: Bearer <access_token>

// Response 200 OK
{
  "valid": true,
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "role": "teacher"
  }
}

// Response 401 Unauthorized
{
  "valid": false,
  "error": "Token expirado"
}
```

## Estados y Transiciones

### Estados de Sesión
```
┌─────────────────────────────────────────────────────────────────┐
│                    ESTADOS DE SESIÓN                             │
└─────────────────────────────────────────────────────────────────┘

NO_SESSION → (Login exitoso) → VALID
                                  ↓
                    ┌─────────────┴──────────────┐
                    ↓                            ↓
            (Time passes)                 (Invalidado)
                    ↓                            ↓
            NEAR_EXPIRY                      INVALID
                    ↓                            ↓
            (Refresh exitoso)            (Logout/Redirect)
                    ↓                            ↓
                 VALID                      NO_SESSION
```

### Matriz de Validación

| Validación | Cuándo | Acción si Inválido | Prioridad |
|------------|--------|-------------------|-----------|
| **At Startup** | Al cargar app | Redirect a login | CRÍTICA |
| **At Navigation** | Cambio de ruta | Redirect a login | CRÍTICA |
| **At API Call** | Antes de request | Intentar refresh | ALTA |
| **Periodic** | Cada 30 segundos | Intentar refresh | MEDIA |
| **On Focus** | App vuelve a foreground | Verificar expiración | MEDIA |

## Manejo de Errores

| Escenario | Detección | Acción |
|-----------|-----------|--------|
| Token no existe | localStorage vacío | Redirect a login |
| Token expirado | `exp` < now | Intentar refresh, si falla → logout |
| Token malformado | Decode falla | Limpiar storage + redirect |
| 401 en API call | Interceptor | Intentar refresh, si falla → logout |
| Usuario desactivado | API retorna 403 | Logout forzado + modal |
| Cambio de rol | Verificación periódica | Refrescar permisos |

## Consideraciones de UX

### Guards de Ruta
```typescript
// Proteger rutas según rol
interface RouteGuard {
  path: string;
  requiredRoles: string[];
  fallback: string;
}

const routes: RouteGuard[] = [
  {
    path: '/admin/*',
    requiredRoles: ['admin', 'super_admin'],
    fallback: '/dashboard'
  },
  {
    path: '/teacher/*',
    requiredRoles: ['teacher', 'admin', 'super_admin'],
    fallback: '/dashboard'
  },
  {
    path: '/student/*',
    requiredRoles: ['student'],
    fallback: '/login'
  }
];
```

### Loading States
```typescript
// Mostrar loader mientras valida sesión
const SessionStates = {
  loading: {
    show: true,
    message: 'Verificando sesión...'
  },
  valid: {
    show: false
  },
  invalid: {
    show: false,
    redirect: '/login'
  }
};
```

### Mensajes de Error
- **Sesión expirada:** "Tu sesión ha expirado. Inicia sesión nuevamente."
- **Sin permisos:** "No tienes permisos para acceder a esta página."
- **Usuario desactivado:** "Tu cuenta ha sido desactivada. Contacta soporte."

## Almacenamiento Local

### Qué Verificar
```typescript
interface SessionData {
  access_token: string | null;
  expires_at: string | null;
  user: string | null; // JSON
  refresh_token: string | null;
}

function getSessionData(): SessionData {
  return {
    access_token: localStorage.getItem('access_token'),
    expires_at: localStorage.getItem('expires_at'),
    user: localStorage.getItem('user'),
    refresh_token: localStorage.getItem('refresh_token')
  };
}
```

### TTL del Cache
N/A - Validación es en tiempo real

### Estrategia Offline
```typescript
// Permitir acceso limitado offline si sesión era válida
if (!navigator.onLine && wasValidBeforeOffline()) {
  allowOfflineAccess();
  showBanner('Sin conexión. Funcionalidad limitada.');
} else {
  requireOnlineValidation();
}
```

## Código de Ejemplo (Mobile)

### Servicio de Validación
```typescript
import jwtDecode from 'jwt-decode';

interface SessionValidator {
  isValid(): boolean;
  requiresRefresh(): boolean;
  getUser(): User | null;
  hasRole(role: string | string[]): boolean;
}

class SessionValidatorImpl implements SessionValidator {
  isValid(): boolean {
    const token = localStorage.getItem('access_token');
    if (!token) return false;

    const expiresAt = localStorage.getItem('expires_at');
    if (!expiresAt) return false;

    const expiry = parseInt(expiresAt);
    return Date.now() < expiry;
  }

  requiresRefresh(): boolean {
    if (!this.isValid()) return false;

    const expiresAt = localStorage.getItem('expires_at');
    if (!expiresAt) return true;

    const expiry = parseInt(expiresAt);
    const now = Date.now();

    // Refrescar si faltan menos de 5 minutos
    const fiveMinutes = 5 * 60 * 1000;
    return (expiry - now) < fiveMinutes;
  }

  getUser(): User | null {
    const userJson = localStorage.getItem('user');
    if (!userJson) return null;

    try {
      return JSON.parse(userJson);
    } catch {
      return null;
    }
  }

  hasRole(roles: string | string[]): boolean {
    const user = this.getUser();
    if (!user) return false;

    const requiredRoles = Array.isArray(roles) ? roles : [roles];
    return requiredRoles.includes(user.role);
  }

  validateToken(token: string): boolean {
    try {
      const payload = jwtDecode<JWTPayload>(token);
      const now = Math.floor(Date.now() / 1000);
      return payload.exp > now;
    } catch {
      return false;
    }
  }
}

export const sessionValidator = new SessionValidatorImpl();
```

### Route Guard (React Router)
```tsx
import { Navigate, Outlet } from 'react-router-dom';
import { sessionValidator } from './SessionValidator';

interface ProtectedRouteProps {
  requiredRoles?: string[];
  fallback?: string;
}

export function ProtectedRoute({
  requiredRoles,
  fallback = '/login'
}: ProtectedRouteProps) {
  // Verificar sesión válida
  if (!sessionValidator.isValid()) {
    return <Navigate to="/login" replace />;
  }

  // Verificar rol si se especificó
  if (requiredRoles && !sessionValidator.hasRole(requiredRoles)) {
    return <Navigate to={fallback} replace />;
  }

  // Renderizar componentes hijos
  return <Outlet />;
}

// Uso en Router
function AppRouter() {
  return (
    <Routes>
      <Route path="/login" element={<LoginPage />} />

      {/* Rutas públicas */}
      <Route path="/" element={<HomePage />} />

      {/* Rutas protegidas */}
      <Route element={<ProtectedRoute />}>
        <Route path="/dashboard" element={<Dashboard />} />
      </Route>

      {/* Rutas de admin */}
      <Route element={<ProtectedRoute requiredRoles={['admin', 'super_admin']} />}>
        <Route path="/admin/*" element={<AdminPanel />} />
      </Route>

      {/* Rutas de teacher */}
      <Route element={<ProtectedRoute requiredRoles={['teacher', 'admin']} />}>
        <Route path="/teacher/*" element={<TeacherPanel />} />
      </Route>

      {/* 404 */}
      <Route path="*" element={<NotFound />} />
    </Routes>
  );
}
```

### Hook de React
```typescript
import { useState, useEffect } from 'react';
import { useNavigate } from 'react-router-dom';
import { sessionValidator } from './SessionValidator';

export function useSessionValidation() {
  const navigate = useNavigate();
  const [isValid, setIsValid] = useState(false);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Validar al montar
    const valid = sessionValidator.isValid();

    setIsValid(valid);
    setLoading(false);

    if (!valid) {
      navigate('/login', { replace: true });
    }

    // Validar periódicamente
    const interval = setInterval(() => {
      const stillValid = sessionValidator.isValid();

      if (!stillValid) {
        clearInterval(interval);
        navigate('/login', { replace: true });
      }
    }, 30000); // Cada 30 segundos

    return () => clearInterval(interval);
  }, [navigate]);

  return { isValid, loading };
}

// Uso en componente
function DashboardPage() {
  const { isValid, loading } = useSessionValidation();

  if (loading) {
    return <LoadingScreen />;
  }

  if (!isValid) {
    return null; // Ya está redirigiendo
  }

  return <DashboardContent />;
}
```

### Validación en App Lifecycle
```typescript
import { useEffect } from 'react';

export function useAppLifecycle() {
  useEffect(() => {
    // Validar cuando app vuelve a foreground
    const handleVisibilityChange = () => {
      if (document.visibilityState === 'visible') {
        if (!sessionValidator.isValid()) {
          logoutService.logout('auto');
        } else if (sessionValidator.requiresRefresh()) {
          tokenRefreshService.refreshAccessToken();
        }
      }
    };

    document.addEventListener('visibilitychange', handleVisibilityChange);

    return () => {
      document.removeEventListener('visibilitychange', handleVisibilityChange);
    };
  }, []);
}

// Usar en App root
function App() {
  useAppLifecycle();
  // ... resto de la app
}
```

### Higher Order Component
```tsx
// HOC para proteger componentes
export function withAuth<P extends object>(
  Component: React.ComponentType<P>,
  requiredRoles?: string[]
) {
  return function AuthenticatedComponent(props: P) {
    const { isValid, loading } = useSessionValidation();

    if (loading) {
      return <LoadingScreen />;
    }

    if (!isValid) {
      return <Navigate to="/login" replace />;
    }

    if (requiredRoles && !sessionValidator.hasRole(requiredRoles)) {
      return <NoPermissionsPage />;
    }

    return <Component {...props} />;
  };
}

// Uso
const AdminDashboard = withAuth(
  DashboardComponent,
  ['admin', 'super_admin']
);
```

## Notas de Implementación

### Validación en Múltiples Capas
```typescript
// 1. Router Level - Guards de ruta
<Route element={<ProtectedRoute />} />

// 2. Component Level - HOC o hook
const { isValid } = useSessionValidation();

// 3. API Level - Interceptor
axios.interceptors.request.use(config => {
  if (!sessionValidator.isValid()) {
    throw new Error('Session expired');
  }
  return config;
});

// 4. Action Level - Verificar antes de acción crítica
async function deleteResource() {
  if (!sessionValidator.isValid()) {
    throw new Error('Session expired');
  }
  await api.delete('/resource');
}
```

### Estrategia de Cache de Validación
```typescript
// Cachear resultado de validación brevemente (evitar re-cálculo)
class CachedSessionValidator {
  private lastCheck: number = 0;
  private lastResult: boolean = false;
  private readonly CACHE_TTL = 5000; // 5 segundos

  isValid(): boolean {
    const now = Date.now();

    if (now - this.lastCheck < this.CACHE_TTL) {
      return this.lastResult;
    }

    this.lastResult = sessionValidator.isValid();
    this.lastCheck = now;

    return this.lastResult;
  }

  invalidateCache(): void {
    this.lastCheck = 0;
  }
}
```

### Gotchas Conocidos
1. **Clock skew:** Usar servidor time, no cliente
2. **Infinite redirects:** Asegurar que /login no requiera auth
3. **Race conditions:** Múltiples validaciones simultáneas
4. **Stale validation:** Cachear resultado puede dar falsos positivos
5. **Deep links:** Validar antes de renderizar ruta directa

### Optimizaciones Sugeridas
1. **Lazy validation:** Solo validar al acceder a ruta protegida
2. **Smart refresh:** Refrescar solo si usuario está activo
3. **Background validation:** Validar en Service Worker
4. **Role cache:** Cachear permisos para evitar re-calcular
5. **Preemptive logout:** Detectar inactividad y logout automático

### Ejemplo de Timeout por Inactividad
```typescript
class InactivityDetector {
  private timeout: NodeJS.Timeout | null = null;
  private readonly INACTIVITY_LIMIT = 30 * 60 * 1000; // 30 minutos

  constructor() {
    this.resetTimer();
    this.setupEventListeners();
  }

  private setupEventListeners(): void {
    const events = ['mousedown', 'keydown', 'scroll', 'touchstart'];

    events.forEach(event => {
      document.addEventListener(event, () => this.resetTimer());
    });
  }

  private resetTimer(): void {
    if (this.timeout) {
      clearTimeout(this.timeout);
    }

    this.timeout = setTimeout(() => {
      logoutService.logout('auto');
    }, this.INACTIVITY_LIMIT);
  }
}

// Iniciar detector
export const inactivityDetector = new InactivityDetector();
```
