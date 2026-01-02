# RFC-020: Listado de Materiales de una Sección

## Metadata
- **ID:** RFC-020
- **Proceso:** Materiales
- **Subproceso:** Listado y Consulta
- **Prioridad:** Alta
- **Dependencias:** RFC-001 (Autenticación), RFC-010 (Unidades Académicas)
- **Estado API:** ✅ Listo

## Descripción
Permite a estudiantes y docentes visualizar la lista de materiales educativos disponibles para una sección/unidad académica específica. Los materiales pueden estar en diferentes estados (uploaded, processing, ready, failed) dependiendo del procesamiento del Worker.

## Flujo de Usuario (UX)

### Como Estudiante
1. Navega a su sección/curso asignado
2. Ve lista de materiales con metadata básica:
   - Título del material
   - Descripción breve
   - Materia/Subject
   - Grado
   - Estado (badge visual)
   - Progreso personal (si ya comenzó a leerlo)
3. Puede filtrar por materia o buscar por título
4. Click en material lo lleva a vista detallada

### Como Docente
1. Navega a sección que enseña
2. Ve todos los materiales (propios y de otros docentes)
3. Puede ver estadísticas resumidas de cada material
4. Opción de editar/eliminar solo sus propios materiales

### Estados Visuales
- 🟢 **Ready**: Material listo con resumen y quiz
- 🟡 **Processing**: Generando contenido IA (mostrar spinner)
- ⚪ **Uploaded**: Archivo subido, esperando procesamiento
- 🔴 **Failed**: Error al procesar (mostrar opción de reprocesar)

## Flujo de Datos (Técnico)

### Diagrama de Secuencia
```
Cliente                  API Mobile              PostgreSQL
  |                          |                        |
  |---GET /v1/materials----->|                        |
  |  + Header: Bearer JWT    |                        |
  |                          |---SELECT materials---->|
  |                          |   WHERE school_id=?    |
  |                          |<---MaterialResponse[]--|
  |<---200 OK----------------|                        |
  | Body: MaterialResponse[] |                        |
```

### Endpoints Involucrados

**Endpoint Principal:**
```http
GET /v1/materials
Authorization: Bearer {jwt_token}
```

**Query Parameters (futuro):**
```typescript
?school_id=uuid           // Filtrar por escuela
&academic_unit_id=uuid    // Filtrar por unidad
&subject=Mathematics      // Filtrar por materia
&status=ready            // Filtrar por estado
&limit=20                // Paginación (no implementado)
&offset=0                // Paginación (no implementado)
```

### Request/Response (TypeScript)

**Request:**
```typescript
// Headers
{
  "Authorization": "Bearer eyJhbGciOiJIUzI1NiIs...",
  "Content-Type": "application/json"
}

// Sin body (GET request)
```

**Response 200 OK:**
```typescript
interface MaterialResponse {
  id: string;                           // UUID
  school_id: string;                    // UUID
  uploaded_by_teacher_id: string;       // UUID
  academic_unit_id: string | null;      // UUID o null
  title: string;                        // "Álgebra Lineal - Módulo 1"
  description: string;                  // "Introducción a vectores..."
  subject: string;                      // "Mathematics"
  grade: string;                        // "10"
  file_url: string | null;              // S3 URL o null
  file_type: string;                    // "application/pdf"
  file_size_bytes: number;              // 2048576
  status: "uploaded" | "processing" | "ready" | "failed";
  is_public: boolean;                   // true/false
  processing_started_at: string | null; // ISO8601
  processing_completed_at: string | null; // ISO8601
  created_at: string;                   // ISO8601
  updated_at: string;                   // ISO8601
  deleted_at: string | null;            // ISO8601 (soft delete)
}

// Array de materiales
type MaterialListResponse = MaterialResponse[];
```

**Response 401 Unauthorized:**
```typescript
{
  "error": "unauthorized",
  "message": "Token inválido o expirado"
}
```

**Response 500 Internal Server Error:**
```typescript
{
  "error": "internal_server_error",
  "message": "Error al consultar materiales"
}
```

## Estados y Transiciones

### Estados del Material
```
uploaded → processing → ready
                     ↓
                   failed
```

**uploaded**: Archivo PDF subido a S3, esperando que Worker lo procese

**processing**: Worker está extrayendo texto, generando resumen y quiz

**ready**: Material completamente procesado y disponible

**failed**: Error en procesamiento (PDF corrupto, timeout, error OpenAI)

### Transiciones
- **uploaded → processing**: Cuando Worker consume evento `material.uploaded`
- **processing → ready**: Cuando Worker completa generación exitosamente
- **processing → failed**: Si Worker encuentra error irrecuperable
- **failed → processing**: Si docente solicita reprocesar (endpoint futuro)

## Manejo de Errores

### Error: Token Expirado (401)
```typescript
try {
  const materials = await api.get('/v1/materials');
} catch (error) {
  if (error.response?.status === 401) {
    // Intentar refresh token
    await refreshAccessToken();
    // Reintentar
    return await api.get('/v1/materials');
  }
  throw error;
}
```

### Error: Sin Materiales (200 con array vacío)
```typescript
const materials = await api.get('/v1/materials');

if (materials.length === 0) {
  // Mostrar estado vacío con CTA
  showEmptyState({
    icon: '📚',
    message: 'Aún no hay materiales en esta sección',
    action: userRole === 'teacher' ? 'Subir Material' : null
  });
}
```

### Error: PostgreSQL Caído (500)
```typescript
try {
  const materials = await api.get('/v1/materials');
} catch (error) {
  if (error.response?.status === 500) {
    showError('Servicio temporalmente no disponible. Intenta más tarde.');
    // Log para debugging
    console.error('Error fetching materials:', error);
  }
}
```

## Consideraciones de UX

### Loading States
```typescript
// Mostrar skeleton loaders mientras carga
<MaterialList loading={true}>
  {Array(5).fill(0).map((_, i) => (
    <SkeletonMaterialCard key={i} />
  ))}
</MaterialList>
```

### Agrupación Visual por Estado
```typescript
// Agrupar materiales por estado para mejor escaneo visual
const groupedMaterials = {
  ready: materials.filter(m => m.status === 'ready'),
  processing: materials.filter(m => m.status === 'processing'),
  uploaded: materials.filter(m => m.status === 'uploaded'),
  failed: materials.filter(m => m.status === 'failed')
};

// Mostrar primero los ready, luego processing, etc.
```

### Indicadores de Progreso
```typescript
// Si material está en processing, mostrar indicador animado
{material.status === 'processing' && (
  <Badge variant="warning" animated>
    <Spinner size="sm" /> Procesando...
  </Badge>
)}

// Si está ready, mostrar badge con check
{material.status === 'ready' && (
  <Badge variant="success">
    <CheckIcon /> Listo
  </Badge>
)}
```

### Timestamps Relativos
```typescript
import { formatDistanceToNow } from 'date-fns';
import { es } from 'date-fns/locale';

// Mostrar "hace 2 horas" en lugar de timestamp exacto
const relativeTime = formatDistanceToNow(
  new Date(material.created_at),
  { addSuffix: true, locale: es }
);
// "hace 2 horas", "hace 3 días"
```

## Almacenamiento Local

### Cache de Lista (Redux/Zustand)
```typescript
interface MaterialsState {
  materials: MaterialResponse[];
  loading: boolean;
  lastFetched: number; // Unix timestamp
  error: string | null;
}

// Cache por 5 minutos
const CACHE_DURATION = 5 * 60 * 1000;

async function fetchMaterials(forceRefresh = false) {
  const now = Date.now();
  const cacheValid = (now - lastFetched) < CACHE_DURATION;

  if (!forceRefresh && cacheValid && materials.length > 0) {
    // Usar cache
    return materials;
  }

  // Fetch fresh
  const fresh = await api.get('/v1/materials');
  setMaterials(fresh);
  setLastFetched(now);
  return fresh;
}
```

### Offline Support (IndexedDB)
```typescript
// Guardar para acceso offline
await db.materials.bulkPut(materials);

// Leer offline
const cachedMaterials = await db.materials.toArray();

// Mostrar con indicador offline
if (!navigator.onLine && cachedMaterials.length > 0) {
  showOfflineBanner();
  return cachedMaterials;
}
```

## Código de Ejemplo (Mobile - TypeScript)

### Hook Personalizado
```typescript
import { useQuery } from '@tanstack/react-query';
import { materialsApi } from '@/services/api';

interface UseMaterialsOptions {
  academicUnitId?: string;
  subject?: string;
  status?: MaterialStatus;
}

export function useMaterials(options: UseMaterialsOptions = {}) {
  return useQuery({
    queryKey: ['materials', options],
    queryFn: () => materialsApi.list(options),
    staleTime: 5 * 60 * 1000, // 5 min
    refetchOnWindowFocus: true,
    refetchInterval: (data) => {
      // Refetch cada 10s si hay materiales en processing
      const hasProcessing = data?.some(m => m.status === 'processing');
      return hasProcessing ? 10_000 : false;
    }
  });
}

// Uso en componente
function MaterialsList() {
  const { data: materials, isLoading, error, refetch } = useMaterials({
    academicUnitId: currentUnit.id
  });

  if (isLoading) return <SkeletonList />;
  if (error) return <ErrorState onRetry={refetch} />;
  if (!materials?.length) return <EmptyState />;

  return (
    <div className="grid gap-4">
      {materials.map(material => (
        <MaterialCard key={material.id} material={material} />
      ))}
    </div>
  );
}
```

### Service Layer
```typescript
// services/api/materials.ts
import axios from 'axios';

const api = axios.create({
  baseURL: import.meta.env.VITE_API_MOBILE_URL,
  timeout: 10_000
});

// Interceptor para agregar JWT
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('access_token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export const materialsApi = {
  async list(filters?: {
    academicUnitId?: string;
    subject?: string;
    status?: string;
  }): Promise<MaterialResponse[]> {
    const params = new URLSearchParams();

    if (filters?.academicUnitId) {
      params.append('academic_unit_id', filters.academicUnitId);
    }
    if (filters?.subject) {
      params.append('subject', filters.subject);
    }
    if (filters?.status) {
      params.append('status', filters.status);
    }

    const response = await api.get('/v1/materials', { params });
    return response.data;
  }
};
```

### Componente MaterialCard
```typescript
import { Badge, Card, CardContent, CardFooter } from '@/components/ui';
import { formatDistanceToNow } from 'date-fns';
import { es } from 'date-fns/locale';

interface MaterialCardProps {
  material: MaterialResponse;
}

export function MaterialCard({ material }: MaterialCardProps) {
  const statusConfig = {
    ready: { color: 'success', icon: '✅', label: 'Listo' },
    processing: { color: 'warning', icon: '⏳', label: 'Procesando' },
    uploaded: { color: 'info', icon: '📤', label: 'Subido' },
    failed: { color: 'error', icon: '❌', label: 'Error' }
  };

  const config = statusConfig[material.status];
  const timeAgo = formatDistanceToNow(
    new Date(material.created_at),
    { addSuffix: true, locale: es }
  );

  return (
    <Card className="hover:shadow-lg transition-shadow">
      <CardContent>
        <div className="flex justify-between items-start mb-2">
          <h3 className="text-lg font-semibold">{material.title}</h3>
          <Badge variant={config.color}>
            {config.icon} {config.label}
          </Badge>
        </div>

        <p className="text-sm text-gray-600 mb-3 line-clamp-2">
          {material.description}
        </p>

        <div className="flex gap-2 text-xs text-gray-500">
          <span>📚 {material.subject}</span>
          <span>•</span>
          <span>🎓 Grado {material.grade}</span>
          {material.file_size_bytes && (
            <>
              <span>•</span>
              <span>📄 {formatBytes(material.file_size_bytes)}</span>
            </>
          )}
        </div>
      </CardContent>

      <CardFooter className="text-xs text-gray-400">
        Subido {timeAgo}
      </CardFooter>
    </Card>
  );
}

function formatBytes(bytes: number): string {
  if (bytes < 1024) return `${bytes} B`;
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`;
  return `${(bytes / (1024 * 1024)).toFixed(1)} MB`;
}
```

## Notas de Implementación

### Limitaciones Actuales
- ⚠️ **Sin paginación**: Backend retorna TODOS los materiales. Implementar antes de producción si hay >100 materiales.
- ⚠️ **Sin filtros server-side**: Filtrado debe hacerse en cliente (menos eficiente).
- ⚠️ **Sin ordenamiento**: Materiales retornan en orden de creación (PostgreSQL default).

### Mejoras Futuras
1. **Paginación**:
   ```typescript
   GET /v1/materials?limit=20&offset=0
   ```

2. **Búsqueda Full-Text**:
   ```typescript
   GET /v1/materials?search=algebra
   ```

3. **Ordenamiento**:
   ```typescript
   GET /v1/materials?sort_by=created_at&order=desc
   ```

4. **Filtros Múltiples**:
   ```typescript
   GET /v1/materials?subject=Math,Science&grade=10,11
   ```

### Performance
- Cache de 5 minutos para reducir carga en DB
- Auto-refetch cada 10s si hay materiales en `processing`
- Skeleton loaders para perceived performance
- Lazy load de imágenes/previews

### Seguridad
- ✅ JWT requerido (autenticación)
- ⚠️ Sin autorización por rol (cualquier usuario autenticado puede listar)
- Recomendación: Filtrar materiales por `school_id` del usuario en backend

### Testing
```typescript
describe('MaterialsList', () => {
  it('muestra skeleton mientras carga', () => {
    render(<MaterialsList />);
    expect(screen.getByTestId('skeleton-list')).toBeInTheDocument();
  });

  it('muestra materiales cuando carga exitosamente', async () => {
    const materials = [mockMaterial({ status: 'ready' })];
    mockApi.get.mockResolvedValue({ data: materials });

    render(<MaterialsList />);

    await waitFor(() => {
      expect(screen.getByText(materials[0].title)).toBeInTheDocument();
    });
  });

  it('muestra estado vacío cuando no hay materiales', async () => {
    mockApi.get.mockResolvedValue({ data: [] });

    render(<MaterialsList />);

    await waitFor(() => {
      expect(screen.getByText(/no hay materiales/i)).toBeInTheDocument();
    });
  });

  it('refetch automático con materiales en processing', async () => {
    const materials = [mockMaterial({ status: 'processing' })];
    mockApi.get.mockResolvedValue({ data: materials });

    render(<MaterialsList />);

    // Esperar 11 segundos (intervalo de refetch)
    await waitFor(() => {
      expect(mockApi.get).toHaveBeenCalledTimes(2);
    }, { timeout: 12000 });
  });
});
```

---

**Última actualización:** 2025-12-24
**Versión:** 1.0
**Estado:** ✅ Listo para implementación
