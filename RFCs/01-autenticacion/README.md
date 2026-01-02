# 01 - Autenticación

## Descripción

Este módulo documenta los flujos de autenticación y manejo de sesiones para la aplicación mobile de EduGo. Incluye login, refresh de tokens, logout y validación de sesiones.

## RFCs en este Módulo

| RFC | Nombre | Estado | Endpoint |
|-----|--------|--------|----------|
| [RFC-001](./RFC-001-login-usuario.md) | Login con Email/Password | ✅ | POST /v1/auth/login |
| [RFC-002](./RFC-002-refresh-token.md) | Renovación de Tokens JWT | ✅ | POST /v1/auth/refresh |
| [RFC-003](./RFC-003-logout.md) | Cierre de Sesión | ✅ | POST /v1/auth/logout |
| [RFC-004](./RFC-004-validacion-sesion.md) | Validación de Sesión Activa | ✅ | GET /v1/auth/me |

## Flujo General de Autenticación

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Login     │────▶│   Tokens    │────▶│   App Use   │
│  (RFC-001)  │     │   Stored    │     │   Normal    │
└─────────────┘     └─────────────┘     └──────┬──────┘
                                               │
                    ┌──────────────────────────┤
                    │                          │
              Token Expired?              ▼ Logout
                    │                    (RFC-003)
                    ▼                          │
              ┌─────────────┐                  │
              │   Refresh   │                  │
              │  (RFC-002)  │                  │
              └──────┬──────┘                  │
                     │                         │
              Success? ─No──▶ Redirect Login ◀─┘
                     │
                    Yes
                     │
                     ▼
              Continue App Use
```

## Tokens JWT

### Estructura del Access Token

```typescript
interface JWTClaims {
  sub: string;       // user_id
  email: string;
  role: 'student' | 'teacher' | 'admin' | 'super_admin';
  school_id: string;
  iss: 'edugo-central';
  exp: number;       // Expiración (15 min)
  iat: number;       // Issued at
}
```

### Tiempos de Expiración

| Token | Duración | Almacenamiento |
|-------|----------|----------------|
| Access Token | 15 minutos | SecureStore |
| Refresh Token | 7 días | SecureStore |

## Almacenamiento Seguro

```typescript
import * as SecureStore from 'expo-secure-store';

const TOKEN_KEYS = {
  ACCESS: 'edugo_access_token',
  REFRESH: 'edugo_refresh_token',
};

export const tokenStorage = {
  async setTokens(access: string, refresh: string) {
    await SecureStore.setItemAsync(TOKEN_KEYS.ACCESS, access);
    await SecureStore.setItemAsync(TOKEN_KEYS.REFRESH, refresh);
  },

  async getAccessToken(): Promise<string | null> {
    return SecureStore.getItemAsync(TOKEN_KEYS.ACCESS);
  },

  async getRefreshToken(): Promise<string | null> {
    return SecureStore.getItemAsync(TOKEN_KEYS.REFRESH);
  },

  async clearTokens() {
    await SecureStore.deleteItemAsync(TOKEN_KEYS.ACCESS);
    await SecureStore.deleteItemAsync(TOKEN_KEYS.REFRESH);
  },
};
```

## Manejo de Errores de Auth

| Código | Situación | Acción |
|--------|-----------|--------|
| 401 | Token expirado | Intentar refresh |
| 401 | Refresh fallido | Redirect a login |
| 403 | Sin permisos | Mostrar error |
| 423 | Cuenta bloqueada | Mostrar mensaje especial |

## Hook de Autenticación

```typescript
export function useAuth() {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    checkSession();
  }, []);

  const checkSession = async () => {
    try {
      const token = await tokenStorage.getAccessToken();
      if (token) {
        const userData = await authService.validateSession();
        setUser(userData);
      }
    } catch (error) {
      await tokenStorage.clearTokens();
    } finally {
      setIsLoading(false);
    }
  };

  const login = async (email: string, password: string) => {
    const response = await authService.login(email, password);
    await tokenStorage.setTokens(response.access_token, response.refresh_token);
    setUser(response.user);
  };

  const logout = async () => {
    await authService.logout();
    await tokenStorage.clearTokens();
    setUser(null);
  };

  return { user, isLoading, login, logout };
}
```

---

**Última actualización:** 2024-12-24
