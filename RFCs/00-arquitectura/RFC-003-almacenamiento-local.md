# RFC-003: Estrategia de Cache y Almacenamiento Local

## Metadata
- **ID:** RFC-003
- **Proceso:** Arquitectura General
- **Subproceso:** Almacenamiento Local y Cache
- **Prioridad:** Media
- **Dependencias:** RFC-000
- **Estado API:** ✅ Listo

## Descripción
Define la estrategia de almacenamiento local (localStorage, sessionStorage, IndexedDB) y cache de datos para optimizar la experiencia de usuario, reducir llamadas a API y permitir funcionalidad offline limitada.

## Flujo de Usuario (UX)
1. Usuario accede a la app
2. App verifica cache local primero
3. Si cache válido, muestra datos inmediatamente (Stale While Revalidate)
4. App refresca datos en background
5. Usuario continúa trabajando sin interrupciones
6. Cambios se sincronizan automáticamente

## Flujo de Datos (Técnico)

### Diagrama de Secuencia
```
Usuario → App
          ↓
    Verificar Cache Local
          ↓
   ┌──────┴──────┐
   ↓             ↓
Cache Hit    Cache Miss
   ↓             ↓
Retornar     Fetch API
Cached Data      ↓
   ↓         Guardar en
Refresh in    Cache
Background       ↓
   ↓         Retornar Data
   └─────────┬──┘
             ↓
        Usuario ve datos
```

### Componentes de Storage

| Storage | Uso | Tamaño Límite | Persistencia | Sincronía |
|---------|-----|---------------|--------------|-----------|
| **localStorage** | Auth tokens, preferencias | ~5-10 MB | Hasta logout/clear | Síncrono |
| **sessionStorage** | Estado de sesión temporal | ~5-10 MB | Hasta cerrar tab | Síncrono |
| **IndexedDB** | Datos grandes (materiales, cache) | ~50 MB - 1 GB+ | Hasta clear | Asíncrono |
| **Memory Cache** | Cache en RAM para sesión actual | Ilimitado | Hasta refresh | Síncrono |

### Endpoints Cacheables

| Endpoint | Método | TTL | Storage | Invalidar en |
|----------|--------|-----|---------|--------------|
| `/v1/materials` | GET | 5 min | IndexedDB | POST, PUT, DELETE de material |
| `/v1/materials/:id` | GET | 5 min | IndexedDB | PUT, DELETE del material |
| `/v1/schools/:id/units/tree` | GET | 15 min | IndexedDB | POST, PUT, DELETE de unit |
| `/v1/schools` | GET | 10 min | IndexedDB | POST, PUT, DELETE de school |
| `/v1/materials/:id/assessment` | GET | Sesión | Memory | Nunca (inmutable) |
| `/v1/materials/:id/summary` | GET | Sesión | Memory | Nunca (inmutable) |
| `/v1/users/me/attempts` | GET | No cachear | N/A | Siempre fresh |
| `/v1/progress` | PUT | No cachear | N/A | Sincronizar inmediato |

## Estados y Transiciones

### Estados del Cache
```typescript
type CacheStatus =
  | 'fresh'     // Dentro del TTL, usar directamente
  | 'stale'     // Expirado pero usable, refrescar en background
  | 'invalid'   // Inválido, fetch obligatorio
  | 'empty';    // No existe cache

// Máquina de estados
const CacheStateMachine = {
  fresh: {
    onRead: 'return cache',
    onExpire: 'transition to stale'
  },
  stale: {
    onRead: 'return cache + refresh background',
    onRefreshComplete: 'transition to fresh'
  },
  invalid: {
    onRead: 'fetch from API',
    onFetchComplete: 'transition to fresh'
  },
  empty: {
    onRead: 'fetch from API',
    onFetchComplete: 'transition to fresh'
  }
};
```

### Request/Response

**Estructura de Entrada en Cache:**
```typescript
interface CacheEntry<T> {
  key: string;
  data: T;
  timestamp: number;
  ttl: number; // milliseconds
  version: string; // Para invalidar cache antiguo
}

// Ejemplo
{
  "key": "materials_list",
  "data": [...], // Array de materiales
  "timestamp": 1703419200000,
  "ttl": 300000, // 5 minutos
  "version": "v1"
}
```

## Manejo de Errores

| Situación | Acción |
|-----------|--------|
| IndexedDB no disponible | Fallback a memoria (solo sesión actual) |
| Quota exceeded | Limpiar cache antiguo, mostrar warning |
| Corrupción de datos | Limpiar cache completo, refetch |
| Error de serialización | Log error, no cachear ese item |

## Consideraciones de UX

### Loading States
- **Cache Hit:** Mostrar datos inmediatamente, actualizar en background sin loader
- **Cache Miss:** Mostrar skeleton loader mientras carga
- **Cache Stale:** Mostrar datos + badge "Actualizando..."

### Optimistic Updates
```typescript
// Para acciones que modifican datos
async function optimisticUpdate<T>(
  action: () => Promise<T>,
  rollbackFn: () => void
) {
  try {
    // 1. Actualizar UI inmediatamente (optimista)
    updateUIOptimistically();

    // 2. Ejecutar acción real
    const result = await action();

    // 3. Confirmar en cache
    updateCache(result);

    return result;
  } catch (error) {
    // 4. Revertir cambios en UI
    rollbackFn();
    throw error;
  }
}
```

## Almacenamiento Local

### Qué Cachear

#### 1. Datos de Autenticación (localStorage)
```typescript
interface AuthCache {
  access_token: string;
  refresh_token: string;
  expires_at: number;
  user: {
    id: string;
    email: string;
    name: string;
    role: string;
    school_id: string;
  };
}
```

#### 2. Listas de Recursos (IndexedDB)
```typescript
interface ResourceCache {
  // Materiales
  materials: {
    list: Material[];
    lastFetch: number;
    ttl: 300000; // 5 min
  };

  // Escuelas
  schools: {
    list: School[];
    lastFetch: number;
    ttl: 600000; // 10 min
  };

  // Árbol de unidades
  units: {
    [schoolId: string]: {
      tree: UnitTreeNode[];
      lastFetch: number;
      ttl: 900000; // 15 min
    };
  };
}
```

#### 3. Progreso de Usuario (IndexedDB + Sync Queue)
```typescript
interface ProgressCache {
  // Progreso local (puede estar offline)
  local: {
    [materialId: string]: {
      percentage: number;
      lastPage: number;
      lastUpdate: number;
      synced: boolean; // false si está en cola de sync
    };
  };

  // Cola de sincronización pendiente
  syncQueue: {
    [materialId: string]: ProgressUpdate;
  };
}
```

#### 4. Preferencias de Usuario (localStorage)
```typescript
interface UserPreferences {
  theme: 'light' | 'dark' | 'auto';
  language: 'es' | 'en';
  notifications: boolean;
  autoSaveProgress: boolean;
  defaultView: 'grid' | 'list';
}
```

### Qué NO Cachear
- ❌ Evaluaciones (assessment) - Siempre fresh para evitar trampa
- ❌ Intentos de assessment - Datos sensibles
- ❌ Tokens de refresh (por seguridad, regenerar)
- ❌ Datos de otros usuarios (privacidad)
- ❌ Archivos PDF grandes (usar presigned URLs)

### TTL del Cache

| Tipo de Dato | TTL | Razón |
|--------------|-----|-------|
| Auth Tokens | Hasta `exp` claim | Seguridad |
| Lista de Materiales | 5 minutos | Cambios frecuentes |
| Detalle de Material | 5 minutos | Cambios frecuentes |
| Árbol de Unidades | 15 minutos | Cambios infrecuentes |
| Escuelas | 10 minutos | Cambios infrecuentes |
| Assessment | Sesión actual | Inmutable después de generado |
| Summary | Sesión actual | Inmutable después de generado |
| Progreso de Usuario | No cachear, sync inmediato | Datos críticos |
| Preferencias | Indefinido | Solo cambio manual |

### Estrategia Offline

```typescript
// Detección de estado offline/online
class OfflineManager {
  private isOnline = navigator.onLine;
  private syncQueue: SyncQueue = new SyncQueue();

  constructor() {
    window.addEventListener('online', () => this.handleOnline());
    window.addEventListener('offline', () => this.handleOffline());
  }

  private handleOffline() {
    this.isOnline = false;
    showBanner('Sin conexión. Los cambios se sincronizarán automáticamente.');
  }

  private async handleOnline() {
    this.isOnline = false;
    showBanner('Conexión restaurada. Sincronizando...');
    await this.syncQueue.processAll();
    showBanner('Sincronización completa.', 'success');
  }

  async queueUpdate(update: PendingUpdate) {
    this.syncQueue.add(update);
    if (this.isOnline) {
      await this.syncQueue.processAll();
    }
  }
}
```

## Código de Ejemplo (Mobile)

### Cache Manager (IndexedDB)
```typescript
import { openDB, DBSchema, IDBPDatabase } from 'idb';

interface EduGoDB extends DBSchema {
  materials: {
    key: string;
    value: CacheEntry<Material[]>;
  };
  material_details: {
    key: string;
    value: CacheEntry<Material>;
  };
  schools: {
    key: string;
    value: CacheEntry<School[]>;
  };
  units: {
    key: string;
    value: CacheEntry<UnitTreeNode[]>;
  };
  progress: {
    key: string; // materialId
    value: {
      percentage: number;
      lastPage: number;
      lastUpdate: number;
      synced: boolean;
    };
  };
}

class CacheManager {
  private db: IDBPDatabase<EduGoDB> | null = null;
  private memoryCache = new Map<string, any>();

  async init() {
    this.db = await openDB<EduGoDB>('edugo-cache', 1, {
      upgrade(db) {
        // Crear stores
        db.createObjectStore('materials');
        db.createObjectStore('material_details');
        db.createObjectStore('schools');
        db.createObjectStore('units');
        db.createObjectStore('progress');
      },
    });
  }

  async get<T>(
    store: keyof EduGoDB,
    key: string
  ): Promise<T | null> {
    if (!this.db) await this.init();

    try {
      const entry = await this.db!.get(store, key) as CacheEntry<T>;

      if (!entry) return null;

      // Verificar si está dentro del TTL
      const age = Date.now() - entry.timestamp;

      if (age > entry.ttl) {
        // Cache expirado, eliminar
        await this.delete(store, key);
        return null;
      }

      return entry.data;
    } catch (error) {
      console.error('Cache get error:', error);
      return null;
    }
  }

  async set<T>(
    store: keyof EduGoDB,
    key: string,
    data: T,
    ttl: number = 300000 // 5 min default
  ): Promise<void> {
    if (!this.db) await this.init();

    const entry: CacheEntry<T> = {
      key,
      data,
      timestamp: Date.now(),
      ttl,
      version: 'v1'
    };

    try {
      await this.db!.put(store, entry as any, key);
    } catch (error) {
      if ((error as any).name === 'QuotaExceededError') {
        await this.clearOldest(store);
        await this.db!.put(store, entry as any, key);
      } else {
        console.error('Cache set error:', error);
      }
    }
  }

  async delete(store: keyof EduGoDB, key: string): Promise<void> {
    if (!this.db) await this.init();
    await this.db!.delete(store, key);
  }

  async clear(store?: keyof EduGoDB): Promise<void> {
    if (!this.db) await this.init();

    if (store) {
      await this.db!.clear(store);
    } else {
      // Limpiar todos los stores
      const stores: (keyof EduGoDB)[] = [
        'materials',
        'material_details',
        'schools',
        'units',
        'progress'
      ];
      await Promise.all(stores.map(s => this.db!.clear(s)));
    }
  }

  private async clearOldest(store: keyof EduGoDB): Promise<void> {
    // Implementar lógica LRU (Least Recently Used)
    // Por simplicidad, limpiar todo el store
    await this.clear(store);
  }

  // Memory cache para datos de sesión
  getMemory<T>(key: string): T | null {
    return this.memoryCache.get(key) || null;
  }

  setMemory<T>(key: string, data: T): void {
    this.memoryCache.set(key, data);
  }

  clearMemory(): void {
    this.memoryCache.clear();
  }
}

export const cacheManager = new CacheManager();
```

### Stale While Revalidate Pattern
```typescript
async function fetchWithCache<T>(
  cacheKey: string,
  fetchFn: () => Promise<T>,
  options: {
    ttl?: number;
    store?: keyof EduGoDB;
    useMemory?: boolean;
  } = {}
): Promise<T> {
  const {
    ttl = 300000, // 5 min
    store = 'materials',
    useMemory = false
  } = options;

  // 1. Verificar memory cache primero (más rápido)
  if (useMemory) {
    const memoryData = cacheManager.getMemory<T>(cacheKey);
    if (memoryData) return memoryData;
  }

  // 2. Verificar IndexedDB cache
  const cachedData = await cacheManager.get<T>(store, cacheKey);

  if (cachedData) {
    // Retornar cache y refrescar en background
    fetchFn()
      .then(freshData => {
        cacheManager.set(store, cacheKey, freshData, ttl);
        if (useMemory) cacheManager.setMemory(cacheKey, freshData);
      })
      .catch(error => {
        console.warn('Background refresh failed:', error);
      });

    return cachedData;
  }

  // 3. Cache miss, fetch y cachear
  const freshData = await fetchFn();
  await cacheManager.set(store, cacheKey, freshData, ttl);
  if (useMemory) cacheManager.setMemory(cacheKey, freshData);

  return freshData;
}

// Uso
const materials = await fetchWithCache(
  'materials_list',
  () => api.getMaterials(),
  { ttl: 300000, store: 'materials' }
);
```

### Sync Queue para Offline
```typescript
interface PendingUpdate {
  id: string;
  type: 'progress' | 'create' | 'update' | 'delete';
  endpoint: string;
  method: 'POST' | 'PUT' | 'DELETE';
  data: any;
  timestamp: number;
  retries: number;
}

class SyncQueue {
  private queue: PendingUpdate[] = [];
  private processing = false;

  constructor() {
    this.loadFromStorage();
  }

  add(update: PendingUpdate) {
    this.queue.push(update);
    this.saveToStorage();
  }

  async processAll(): Promise<void> {
    if (this.processing) return;

    this.processing = true;

    try {
      while (this.queue.length > 0) {
        const update = this.queue[0];

        try {
          await this.processUpdate(update);
          this.queue.shift(); // Remover de la cola
        } catch (error) {
          update.retries++;

          if (update.retries >= 3) {
            // Demasiados intentos, remover
            this.queue.shift();
            console.error('Update failed after 3 retries:', update);
          }

          break; // Detener procesamiento si falla
        }
      }
    } finally {
      this.processing = false;
      this.saveToStorage();
    }
  }

  private async processUpdate(update: PendingUpdate): Promise<void> {
    const { endpoint, method, data } = update;

    switch (method) {
      case 'POST':
        await axios.post(endpoint, data);
        break;
      case 'PUT':
        await axios.put(endpoint, data);
        break;
      case 'DELETE':
        await axios.delete(endpoint);
        break;
    }
  }

  private loadFromStorage() {
    const stored = localStorage.getItem('sync_queue');
    if (stored) {
      this.queue = JSON.parse(stored);
    }
  }

  private saveToStorage() {
    localStorage.setItem('sync_queue', JSON.stringify(this.queue));
  }
}
```

### Hook de React
```typescript
import { useState, useEffect } from 'react';

export function useCachedAPI<T>(
  cacheKey: string,
  fetchFn: () => Promise<T>,
  options: {
    ttl?: number;
    enabled?: boolean;
  } = {}
) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  const enabled = options.enabled !== false;

  useEffect(() => {
    if (!enabled) return;

    let cancelled = false;

    async function load() {
      try {
        setLoading(true);
        const result = await fetchWithCache(cacheKey, fetchFn, {
          ttl: options.ttl
        });

        if (!cancelled) {
          setData(result);
        }
      } catch (err) {
        if (!cancelled) {
          setError(err as Error);
        }
      } finally {
        if (!cancelled) {
          setLoading(false);
        }
      }
    }

    load();

    return () => {
      cancelled = true;
    };
  }, [cacheKey, enabled]);

  const invalidate = async () => {
    await cacheManager.delete('materials', cacheKey);
    setData(null);
    setLoading(true);

    const result = await fetchFn();
    setData(result);
    setLoading(false);
  };

  return { data, loading, error, invalidate };
}

// Uso
function MaterialsList() {
  const { data: materials, loading, invalidate } = useCachedAPI(
    'materials_list',
    () => api.getMaterials(),
    { ttl: 300000 }
  );

  if (loading) return <Skeleton />;

  return (
    <div>
      <button onClick={invalidate}>Refrescar</button>
      <MaterialsGrid materials={materials} />
    </div>
  );
}
```

## Notas de Implementación

### Versionado de Cache
```typescript
// Invalidar cache antiguo cuando cambia estructura
const CACHE_VERSION = 'v2';

async function migrateCache() {
  const storedVersion = localStorage.getItem('cache_version');

  if (storedVersion !== CACHE_VERSION) {
    // Limpiar cache antiguo
    await cacheManager.clear();
    localStorage.setItem('cache_version', CACHE_VERSION);
  }
}
```

### Gotchas Conocidos
1. **QuotaExceededError:** IndexedDB tiene límite, implementar LRU
2. **Serialization:** No todos los tipos son serializables (funciones, Promises)
3. **Same-Origin:** IndexedDB solo accesible desde mismo dominio
4. **Private Browsing:** IndexedDB puede estar deshabilitado
5. **Concurrent Access:** Manejar múltiples tabs accediendo al mismo cache

### Optimizaciones Sugeridas
1. **Compression:** Comprimir datos grandes antes de cachear
2. **Lazy Loading:** Solo cargar cache cuando se necesite
3. **Background Sync API:** Para sincronización offline más robusta
4. **Service Workers:** Para cache de assets estáticos
5. **Cache Warming:** Precargar datos comunes en idle time
