# RFC-001: Login con Email/Password

## Metadata
- **ID:** RFC-001
- **Proceso:** Autenticación
- **Subproceso:** Login de Usuario
- **Prioridad:** Alta
- **Dependencias:** RFC-000 (Flujo de Datos)
- **Estado API:** ✅ Listo

## Descripción
Flujo completo de login de usuario mediante email y contraseña en API Admin, incluyendo validación, generación de tokens JWT, almacenamiento local y manejo de errores específicos de autenticación.

## Flujo de Usuario (UX)
1. Usuario abre la aplicación
2. Si no tiene sesión activa, ve pantalla de login
3. Usuario ingresa email y contraseña
4. Usuario presiona "Iniciar Sesión"
5. App muestra indicador de carga
6. Si credenciales correctas: redirige a dashboard
7. Si credenciales incorrectas: muestra mensaje de error
8. Si cuenta inactiva: muestra mensaje específico

## Flujo de Datos (Técnico)

### Diagrama de Secuencia
```
Usuario → Frontend → API Admin
                         ↓
             Validar credenciales en PostgreSQL
                         ↓
                    ┌────┴────┐
                    ↓         ↓
                 SUCCESS    FAIL
                    ↓         ↓
            Generar JWT   Error 401
                    ↓
        Access + Refresh Tokens
                    ↓
            Frontend recibe
                    ↓
        Guardar en localStorage
                    ↓
        Extraer claims del JWT
                    ↓
      Redirigir a dashboard
```

### Endpoints Involucrados

| Endpoint | Método | Descripción | Estado |
|----------|--------|-------------|--------|
| `/v1/auth/login` | POST | Login de usuario | ✅ Listo |

### Request/Response

**Request:**
```typescript
interface LoginRequest {
  email: string;    // required, formato email válido
  password: string; // required, min 6 caracteres
}

// Ejemplo
{
  "email": "profesor@escuela.edu",
  "password": "MiPassword123"
}
```

**Response (Éxito - 200 OK):**
```typescript
interface LoginResponse {
  access_token: string;   // JWT válido por 15 minutos
  refresh_token: string;  // JWT válido por 7 días
  expires_in: number;     // Segundos hasta expiración (900)
  user: {
    id: string;           // UUID del usuario
    email: string;
    name: string;
    role: 'student' | 'teacher' | 'admin' | 'super_admin';
    school_id: string;    // UUID de la escuela
    is_active: boolean;
  };
}

// Ejemplo
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expires_in": 900,
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "profesor@escuela.edu",
    "name": "Juan Pérez",
    "role": "teacher",
    "school_id": "660e8400-e29b-41d4-a716-446655440001",
    "is_active": true
  }
}
```

**Response (Error - 401 Unauthorized):**
```typescript
interface LoginError {
  error: string;    // Tipo de error
  message: string;  // Mensaje amigable
  code: string;     // Código específico
}

// Ejemplo - Credenciales inválidas
{
  "error": "unauthorized",
  "message": "Credenciales inválidas",
  "code": "INVALID_CREDENTIALS"
}

// Ejemplo - Usuario inactivo
{
  "error": "forbidden",
  "message": "Tu cuenta ha sido desactivada. Contacta al administrador.",
  "code": "USER_INACTIVE"
}
```

**Response (Error - 400 Bad Request):**
```typescript
// Validación de campos
{
  "error": "bad_request",
  "message": "Errores de validación",
  "code": "VALIDATION_ERROR",
  "details": {
    "email": ["Email inválido"],
    "password": ["Contraseña debe tener al menos 6 caracteres"]
  }
}
```

## Estados y Transiciones

### Estados de Autenticación
```
┌─────────────────────────────────────────────────────────────────┐
│                    ESTADOS DE LOGIN                              │
└─────────────────────────────────────────────────────────────────┘

UNAUTHENTICATED → (Submit form) → AUTHENTICATING
                                        ↓
                          ┌─────────────┴──────────────┐
                          ↓                            ↓
                     SUCCESS (200)                ERROR (401/400/500)
                          ↓                            ↓
                  AUTHENTICATED                  UNAUTHENTICATED
                          ↓                     (mostrar error)
                  (redirect dashboard)
```

### Claims en el JWT
```typescript
interface JWTClaims {
  sub: string;       // user_id (UUID)
  email: string;
  role: 'student' | 'teacher' | 'admin' | 'super_admin';
  school_id: string; // UUID
  iss: 'edugo-central';    // Issuer
  exp: number;       // Expiration (Unix timestamp)
  iat: number;       // Issued at (Unix timestamp)
}

// Ejemplo decodificado
{
  "sub": "550e8400-e29b-41d4-a716-446655440000",
  "email": "profesor@escuela.edu",
  "role": "teacher",
  "school_id": "660e8400-e29b-41d4-a716-446655440001",
  "iss": "edugo-central",
  "exp": 1703420100, // 15 minutos después
  "iat": 1703419200
}
```

## Manejo de Errores

| Código | Condición | Mensaje UI | Acción |
|--------|-----------|------------|--------|
| 400 | Email inválido | "Ingresa un email válido" | Resaltar campo email |
| 400 | Password vacío | "La contraseña es requerida" | Resaltar campo password |
| 401 | Credenciales incorrectas | "Email o contraseña incorrectos" | Permitir reintentar |
| 403 | Usuario inactivo | "Tu cuenta ha sido desactivada. Contacta al administrador." | Bloquear login, mostrar soporte |
| 500 | Error servidor | "Error al iniciar sesión. Intenta nuevamente." | Permitir reintentar |
| Network | Sin conexión | "Verifica tu conexión a internet" | Permitir reintentar |
| Timeout | Request > 10s | "La conexión está tardando. Intenta nuevamente." | Permitir reintentar |

## Consideraciones de UX

### Loading States
```typescript
const LoadingStates = {
  idle: {
    buttonText: 'Iniciar Sesión',
    disabled: false,
    showSpinner: false
  },
  loading: {
    buttonText: 'Iniciando sesión...',
    disabled: true,
    showSpinner: true
  }
};
```

### Validación de Formulario
```typescript
interface FormValidation {
  email: {
    required: 'Email es requerido';
    pattern: {
      value: /^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$/i;
      message: 'Email inválido';
    };
  };
  password: {
    required: 'Contraseña es requerida';
    minLength: {
      value: 6;
      message: 'Contraseña debe tener al menos 6 caracteres';
    };
  };
}
```

### Mensajes de Error
- **Específicos:** "Email o contraseña incorrectos" (no revelar cuál campo falló)
- **Accionables:** Incluir botón "¿Olvidaste tu contraseña?" (futuro)
- **Amigables:** Evitar términos técnicos como "401 Unauthorized"

### Confirmaciones
- **Success:** No mostrar toast, redirigir directamente
- **Error:** Toast o mensaje inline debajo del formulario

## Almacenamiento Local

### Qué Cachear
```typescript
// En localStorage (persistente)
localStorage.setItem('access_token', loginResponse.access_token);
localStorage.setItem('refresh_token', loginResponse.refresh_token);
localStorage.setItem('user', JSON.stringify(loginResponse.user));

// Calcular y guardar expiración
const expiresAt = Date.now() + (loginResponse.expires_in * 1000);
localStorage.setItem('expires_at', expiresAt.toString());
```

### TTL del Cache
- **Access Token:** 15 minutos (según `expires_in`)
- **Refresh Token:** 7 días (según JWT `exp` claim)
- **User Info:** Hasta logout o token expirado

### Estrategia Offline
```typescript
// Login requiere conexión, no funciona offline
if (!navigator.onLine) {
  showError('Necesitas conexión a internet para iniciar sesión');
  disableLoginButton();
}

// Escuchar cambios de conectividad
window.addEventListener('online', () => {
  enableLoginButton();
  hideOfflineMessage();
});

window.addEventListener('offline', () => {
  disableLoginButton();
  showOfflineMessage();
});
```

## Código de Ejemplo (Mobile)

### Servicio de Autenticación
```typescript
import axios from 'axios';
import jwtDecode from 'jwt-decode';

interface AuthService {
  login(email: string, password: string): Promise<LoginResponse>;
  logout(): void;
  isAuthenticated(): boolean;
  getUser(): User | null;
  getAccessToken(): string | null;
}

class AuthServiceImpl implements AuthService {
  private readonly API_URL = 'http://localhost:8081/v1';

  async login(email: string, password: string): Promise<LoginResponse> {
    try {
      const response = await axios.post<LoginResponse>(
        `${this.API_URL}/auth/login`,
        { email, password },
        {
          timeout: 10000, // 10 segundos
          headers: {
            'Content-Type': 'application/json'
          }
        }
      );

      const data = response.data;

      // Guardar en localStorage
      this.saveAuthData(data);

      return data;
    } catch (error: any) {
      if (error.response) {
        // Error de API
        throw new AuthError(
          error.response.data.code,
          error.response.data.message
        );
      } else if (error.request) {
        // Sin respuesta del servidor
        throw new AuthError(
          'NETWORK_ERROR',
          'Verifica tu conexión a internet'
        );
      } else {
        // Error de configuración
        throw new AuthError(
          'UNKNOWN_ERROR',
          'Error al iniciar sesión'
        );
      }
    }
  }

  logout(): void {
    // Limpiar localStorage
    localStorage.removeItem('access_token');
    localStorage.removeItem('refresh_token');
    localStorage.removeItem('user');
    localStorage.removeItem('expires_at');

    // Opcional: notificar al servidor
    axios.post(`${this.API_URL}/auth/logout`).catch(() => {
      // Ignorar errores, ya limpiamos local
    });

    // Redirigir a login
    window.location.href = '/login';
  }

  isAuthenticated(): boolean {
    const token = this.getAccessToken();
    if (!token) return false;

    const expiresAt = localStorage.getItem('expires_at');
    if (!expiresAt) return false;

    return Date.now() < parseInt(expiresAt);
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

  getAccessToken(): string | null {
    return localStorage.getItem('access_token');
  }

  private saveAuthData(data: LoginResponse): void {
    localStorage.setItem('access_token', data.access_token);
    localStorage.setItem('refresh_token', data.refresh_token);
    localStorage.setItem('user', JSON.stringify(data.user));

    const expiresAt = Date.now() + (data.expires_in * 1000);
    localStorage.setItem('expires_at', expiresAt.toString());
  }
}

class AuthError extends Error {
  constructor(
    public code: string,
    message: string
  ) {
    super(message);
    this.name = 'AuthError';
  }
}

export const authService = new AuthServiceImpl();
```

### Componente de Login (React)
```tsx
import { useState, FormEvent } from 'react';
import { useNavigate } from 'react-router-dom';
import { authService } from './AuthService';

export function LoginPage() {
  const navigate = useNavigate();
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    setError(null);

    // Validación básica
    if (!email) {
      setError('Email es requerido');
      return;
    }

    if (!password) {
      setError('Contraseña es requerida');
      return;
    }

    if (password.length < 6) {
      setError('Contraseña debe tener al menos 6 caracteres');
      return;
    }

    try {
      setLoading(true);

      const response = await authService.login(email, password);

      // Login exitoso
      console.log('Login exitoso:', response.user);

      // Redirigir según rol
      const dashboardPath = getDashboardPath(response.user.role);
      navigate(dashboardPath);

    } catch (err: any) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="login-container">
      <div className="login-card">
        <h1>Iniciar Sesión</h1>

        <form onSubmit={handleSubmit}>
          <div className="form-group">
            <label htmlFor="email">Email</label>
            <input
              id="email"
              type="email"
              value={email}
              onChange={(e) => setEmail(e.target.value)}
              disabled={loading}
              autoComplete="email"
              required
            />
          </div>

          <div className="form-group">
            <label htmlFor="password">Contraseña</label>
            <input
              id="password"
              type="password"
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              disabled={loading}
              autoComplete="current-password"
              required
            />
          </div>

          {error && (
            <div className="error-message">
              ⚠️ {error}
            </div>
          )}

          <button
            type="submit"
            disabled={loading}
            className="login-button"
          >
            {loading ? (
              <>
                <Spinner /> Iniciando sesión...
              </>
            ) : (
              'Iniciar Sesión'
            )}
          </button>
        </form>

        {/* Futuro: Link de "Olvidé mi contraseña" */}
      </div>
    </div>
  );
}

function getDashboardPath(role: string): string {
  switch (role) {
    case 'student':
      return '/student/dashboard';
    case 'teacher':
      return '/teacher/dashboard';
    case 'admin':
    case 'super_admin':
      return '/admin/dashboard';
    default:
      return '/dashboard';
  }
}
```

### Hook Personalizado
```typescript
import { useState, useEffect } from 'react';
import { authService } from './AuthService';

export function useAuth() {
  const [user, setUser] = useState<User | null>(null);
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Verificar sesión al cargar
    const currentUser = authService.getUser();
    const isAuth = authService.isAuthenticated();

    setUser(currentUser);
    setIsAuthenticated(isAuth);
    setLoading(false);

    // Si no está autenticado, redirigir a login
    if (!isAuth && window.location.pathname !== '/login') {
      window.location.href = '/login';
    }
  }, []);

  const login = async (email: string, password: string) => {
    const response = await authService.login(email, password);
    setUser(response.user);
    setIsAuthenticated(true);
    return response;
  };

  const logout = () => {
    authService.logout();
    setUser(null);
    setIsAuthenticated(false);
  };

  return {
    user,
    isAuthenticated,
    loading,
    login,
    logout
  };
}

// Uso en App
function App() {
  const { user, isAuthenticated, loading } = useAuth();

  if (loading) {
    return <LoadingScreen />;
  }

  if (!isAuthenticated) {
    return <LoginPage />;
  }

  return <Dashboard user={user} />;
}
```

## Notas de Implementación

### Seguridad
1. **Nunca guardar contraseña:** Solo guardar tokens
2. **HTTPS obligatorio:** En producción, solo permitir HTTPS
3. **Validar JWT:** Verificar firma y expiración antes de usar
4. **Rate limiting:** Backend debe limitar intentos de login (no en scope mobile)

### Gotchas Conocidos
1. **CORS:** Verificar que API Admin tenga CORS habilitado
2. **Timeout:** Request de login puede tardar, usar timeout de 10s
3. **Redirect loops:** Verificar que página de login no requiera auth
4. **Token en URL:** Nunca pasar token en query params (usar header)

### Optimizaciones Sugeridas
1. **Remember me:** Opcional, aumentar TTL del refresh token
2. **Biometrics:** Futuro, usar Touch ID / Face ID
3. **SSO:** Futuro, integrar con Google/Microsoft
4. **2FA:** Futuro, autenticación de dos factores
5. **Error tracking:** Enviar errores de login fallidos a analytics
