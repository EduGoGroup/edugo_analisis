# 00 - Arquitectura

## Descripción

Este módulo documenta los patrones arquitectónicos y estrategias transversales que aplican a toda la aplicación mobile de EduGo. Son fundamentos que deben considerarse en la implementación de cualquier funcionalidad.

## RFCs en este Módulo

| RFC | Nombre | Estado | Descripción |
|-----|--------|--------|-------------|
| [RFC-000](./RFC-000-flujo-datos-mobile.md) | Flujo General de Datos | ✅ | Arquitectura de comunicación API-Mobile |
| [RFC-002](./RFC-002-polling-estados.md) | Polling para Estados | ✅ | Estrategia de polling para procesamiento asíncrono |
| [RFC-003](./RFC-003-almacenamiento-local.md) | Almacenamiento Local | ✅ | Cache, IndexedDB, estrategias offline |
| [RFC-004](./RFC-004-manejo-errores.md) | Manejo de Errores HTTP | ✅ | Estrategia unificada de errores |

## Conceptos Clave

### Flujo de Datos (RFC-000)

```
Mobile App → API Gateway → Backend Services → Database
     ↑                                            ↓
     └──────────── Response ←────────────────────┘
```

### Polling Pattern (RFC-002)

Para materiales en procesamiento (status=processing):

```typescript
const POLLING_CONFIG = {
  interval: 5000,      // 5 segundos
  maxAttempts: 60,     // 5 minutos máximo
  backoff: 1.5         // Factor de incremento
};
```

### Cache Strategy (RFC-003)

| Tipo de Dato | Storage | TTL | Estrategia |
|--------------|---------|-----|------------|
| JWT | SecureStore | 15 min | Refresh automático |
| User Profile | AsyncStorage | 1 hora | Stale-while-revalidate |
| Materials List | IndexedDB | 30 min | Cache-first |
| PDF Files | FileSystem | Permanente | Download on demand |

### Error Handling (RFC-004)

| Código | Acción | UI Feedback |
|--------|--------|-------------|
| 401 | Refresh token o logout | Redirect a login |
| 403 | Mostrar error | "No tienes permiso" |
| 404 | Remover de cache | "No encontrado" |
| 429 | Retry con backoff | "Intenta más tarde" |
| 500+ | Retry 3 veces | "Error del servidor" |

## Implementación Base

### HTTP Client Configurado

```typescript
import axios from 'axios';

const apiClient = axios.create({
  baseURL: process.env.API_URL,
  timeout: 30000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Interceptor de autenticación
apiClient.interceptors.request.use((config) => {
  const token = getToken();
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Interceptor de errores
apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      const refreshed = await refreshToken();
      if (refreshed) {
        return apiClient.request(error.config);
      }
      logout();
    }
    return Promise.reject(error);
  }
);
```

## Dependencias Recomendadas

```json
{
  "axios": "^1.6.0",
  "react-query": "^5.0.0",
  "zustand": "^4.4.0",
  "date-fns": "^2.30.0",
  "expo-secure-store": "^12.0.0"
}
```

---

**Última actualización:** 2024-12-24
