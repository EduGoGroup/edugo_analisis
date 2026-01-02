# RFC-025: Ver Detalle de Material

## Metadata
- **ID:** RFC-025
- **Proceso:** Materiales
- **Subproceso:** Consulta de Detalle
- **Prioridad:** Alta
- **Dependencias:** RFC-001 (Autenticación), RFC-020 (Listado Materiales)
- **Estado API:** ✅ Listo

## Descripción

Permite a estudiantes y docentes visualizar el detalle completo de un material educativo específico, incluyendo metadata, estado de procesamiento, acciones disponibles (ver PDF, tomar quiz, ver resumen) y progreso personal del usuario.

## Flujo de Usuario (UX)

### Pantalla: Detalle de Material

```
┌─────────────────────────────────────────┐
│  ← Materiales                           │
│                                          │
│  ┌──────────────────────────────────┐   │
│  │  📄 Álgebra Lineal - Módulo 1     │   │
│  │                                    │   │
│  │  Estado: ✅ Listo                  │   │
│  │  Materia: Matemáticas              │   │
│  │  Grado: 10                         │   │
│  │  Subido: hace 3 días               │   │
│  │  Tamaño: 2.5 MB                    │   │
│  └──────────────────────────────────┘   │
│                                          │
│  ┌──────────────────────────────────┐   │
│  │  📊 Tu Progreso                    │   │
│  │  ▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░ 45%        │   │
│  │  Última lectura: página 23         │   │
│  └──────────────────────────────────┘   │
│                                          │
│  ┌────────┐ ┌────────┐ ┌────────┐       │
│  │ 📄 PDF │ │ 📝 Quiz│ │📖Resumen│      │
│  └────────┘ └────────┘ └────────┘       │
│                                          │
│  📝 Descripción                          │
│  Este material cubre los conceptos       │
│  fundamentales de vectores y matrices... │
│                                          │
│  👤 Subido por                           │
│  Prof. Juan Pérez                        │
│                                          │
└─────────────────────────────────────────┘
```

### Estados del Material

**Si status = 'ready':**
- Mostrar todas las acciones disponibles
- Badge verde "✅ Listo"

**Si status = 'processing':**
```
┌──────────────────────────────────────┐
│  Estado: ⏳ Procesando                │
│  ────────────────── 60%              │
│  Generando resumen y evaluación...   │
│                                      │
│  ┌────────┐ ┌────────┐ ┌────────┐   │
│  │ 📄 PDF │ │ 📝 Quiz│ │📖Resumen│  │
│  │  ✅    │ │   ⏳   │ │   ⏳   │  │
│  └────────┘ └────────┘ └────────┘   │
└──────────────────────────────────────┘
```

**Si status = 'uploaded':**
- Solo PDF disponible
- Quiz y Resumen deshabilitados
- Mensaje: "El contenido está siendo procesado"

**Si status = 'failed':**
- PDF disponible
- Quiz y Resumen no disponibles
- Mensaje: "Hubo un error al procesar. Contacta al docente."

## Flujo de Datos (Técnico)

### Diagrama de Secuencia

```
Usuario → Frontend → API Mobile → PostgreSQL
                                      ↓
                                 Material +
                                 Progress +
                                 Teacher Info
```

### Endpoints Involucrados

| Endpoint | Método | Descripción | Estado |
|----------|--------|-------------|--------|
| `/v1/materials/:id` | GET | Obtener detalle del material | ✅ Listo |
| `/v1/progress?material_id=:id` | GET | Obtener progreso del usuario | ✅ Listo |

### Request/Response (TypeScript)

**Request:**
```typescript
// GET /v1/materials/:id
// Headers
{
  "Authorization": "Bearer eyJhbGciOiJIUzI1NiIs...",
  "Content-Type": "application/json"
}
```

**Response 200 OK:**
```typescript
interface MaterialDetailResponse {
  id: string;                           // UUID
  school_id: string;                    // UUID
  uploaded_by_teacher_id: string;       // UUID
  academic_unit_id: string | null;      // UUID o null
  title: string;                        // "Álgebra Lineal - Módulo 1"
  description: string;                  // "Introducción a vectores..."
  subject: string;                      // "Mathematics"
  grade: string;                        // "10"
  file_url: string | null;              // S3 URL presigned
  file_type: string;                    // "application/pdf"
  file_size_bytes: number;              // 2621440
  status: MaterialStatus;               // "uploaded" | "processing" | "ready" | "failed"
  is_public: boolean;                   // true/false
  processing_started_at: string | null; // ISO8601
  processing_completed_at: string | null; // ISO8601
  created_at: string;                   // ISO8601
  updated_at: string;                   // ISO8601

  // Información del docente (join)
  teacher?: {
    id: string;
    name: string;
    email: string;
  };

  // Progreso del usuario actual (si existe)
  user_progress?: {
    progress_percentage: number;        // 0-100
    last_page: number;
    last_accessed_at: string;
  };

  // Disponibilidad de features
  features: {
    pdf_available: boolean;
    quiz_available: boolean;
    summary_available: boolean;
  };
}

type MaterialStatus = "uploaded" | "processing" | "ready" | "failed";
```

**Ejemplo Response:**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "school_id": "660e8400-e29b-41d4-a716-446655440001",
  "uploaded_by_teacher_id": "770e8400-e29b-41d4-a716-446655440002",
  "academic_unit_id": "880e8400-e29b-41d4-a716-446655440003",
  "title": "Álgebra Lineal - Módulo 1",
  "description": "Este material cubre los conceptos fundamentales de vectores y matrices, incluyendo operaciones básicas y aplicaciones.",
  "subject": "Mathematics",
  "grade": "10",
  "file_url": "https://edugo-materials.s3.amazonaws.com/550e8400.pdf?X-Amz-...",
  "file_type": "application/pdf",
  "file_size_bytes": 2621440,
  "status": "ready",
  "is_public": false,
  "processing_started_at": "2024-12-20T10:00:00Z",
  "processing_completed_at": "2024-12-20T10:02:30Z",
  "created_at": "2024-12-20T09:55:00Z",
  "updated_at": "2024-12-20T10:02:30Z",
  "teacher": {
    "id": "770e8400-e29b-41d4-a716-446655440002",
    "name": "Juan Pérez",
    "email": "juan.perez@escuela.edu"
  },
  "user_progress": {
    "progress_percentage": 45,
    "last_page": 23,
    "last_accessed_at": "2024-12-23T15:30:00Z"
  },
  "features": {
    "pdf_available": true,
    "quiz_available": true,
    "summary_available": true
  }
}
```

**Response 404 Not Found:**
```json
{
  "error": "not_found",
  "message": "Material no encontrado",
  "code": "MATERIAL_NOT_FOUND"
}
```

**Response 403 Forbidden:**
```json
{
  "error": "forbidden",
  "message": "No tienes acceso a este material",
  "code": "ACCESS_DENIED"
}
```

## Estados y Transiciones

### Estado del Material

```
┌──────────┐
│ uploaded │ → PDF subido, Worker no ha procesado
└────┬─────┘
     │ Worker consume evento
     ▼
┌────────────┐
│ processing │ → Worker extrayendo texto, generando quiz/resumen
└────┬───────┘
     │
     ├─── (éxito) ───►┌───────┐
     │                │ ready │ → Material completamente procesado ✅
     │                └───────┘
     │
     └─── (error) ───►┌────────┐
                      │ failed │ → Error en procesamiento ❌
                      └────────┘
```

### Disponibilidad de Features por Estado

| Estado | PDF | Quiz | Resumen |
|--------|-----|------|---------|
| uploaded | ✅ | ❌ | ❌ |
| processing | ✅ | ⏳ | ⏳ |
| ready | ✅ | ✅ | ✅ |
| failed | ✅ | ❌ | ❌ |

## Manejo de Errores

| Código | Significado | Acción UI |
|--------|-------------|-----------|
| 200 | Material encontrado | Mostrar detalle |
| 401 | Token inválido/expirado | Redirigir a login |
| 403 | Sin acceso al material | Mostrar "Sin permisos" |
| 404 | Material no existe | Mostrar "No encontrado", volver a lista |
| 500 | Error de servidor | Mostrar error genérico + retry |

### Manejo de 404

```typescript
if (error.response?.status === 404) {
  showToast('Material no encontrado');
  navigation.navigate('MaterialsList');
}
```

### Manejo de 403

```typescript
if (error.response?.status === 403) {
  showModal({
    title: 'Sin Acceso',
    message: 'No tienes permisos para ver este material. Contacta a tu docente.',
    actions: [
      { text: 'Volver', onPress: () => navigation.goBack() }
    ]
  });
}
```

## Consideraciones de UX

### Loading State

```typescript
function MaterialDetailSkeleton() {
  return (
    <div className="space-y-4 p-4">
      {/* Header */}
      <div className="animate-pulse">
        <div className="h-8 bg-gray-200 rounded w-3/4 mb-2" />
        <div className="h-4 bg-gray-200 rounded w-1/2" />
      </div>

      {/* Status Badge */}
      <div className="h-6 bg-gray-200 rounded w-24" />

      {/* Progress */}
      <div className="bg-gray-100 p-4 rounded-lg">
        <div className="h-4 bg-gray-200 rounded w-32 mb-2" />
        <div className="h-2 bg-gray-200 rounded w-full" />
      </div>

      {/* Action Buttons */}
      <div className="flex gap-4">
        <div className="h-12 bg-gray-200 rounded flex-1" />
        <div className="h-12 bg-gray-200 rounded flex-1" />
        <div className="h-12 bg-gray-200 rounded flex-1" />
      </div>

      {/* Description */}
      <div className="space-y-2">
        <div className="h-4 bg-gray-200 rounded w-full" />
        <div className="h-4 bg-gray-200 rounded w-full" />
        <div className="h-4 bg-gray-200 rounded w-2/3" />
      </div>
    </div>
  );
}
```

### Formateo de Datos

```typescript
// Formatear tamaño de archivo
function formatFileSize(bytes: number): string {
  if (bytes < 1024) return `${bytes} B`;
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`;
  return `${(bytes / (1024 * 1024)).toFixed(1)} MB`;
}

// Formatear fecha relativa
import { formatDistanceToNow } from 'date-fns';
import { es } from 'date-fns/locale';

function formatRelativeDate(isoDate: string): string {
  return formatDistanceToNow(new Date(isoDate), {
    addSuffix: true,
    locale: es
  });
}
// "hace 3 días", "hace 2 horas"
```

### Status Badge Component

```typescript
interface StatusBadgeProps {
  status: MaterialStatus;
}

function StatusBadge({ status }: StatusBadgeProps) {
  const config = {
    uploaded: { color: 'gray', icon: '📤', text: 'Subido' },
    processing: { color: 'yellow', icon: '⏳', text: 'Procesando' },
    ready: { color: 'green', icon: '✅', text: 'Listo' },
    failed: { color: 'red', icon: '❌', text: 'Error' }
  };

  const { color, icon, text } = config[status];

  return (
    <span className={`badge badge-${color}`}>
      {icon} {text}
    </span>
  );
}
```

## Almacenamiento Local

### Cache de Material

```typescript
interface MaterialCache {
  materialId: string;
  material: MaterialDetailResponse;
  cachedAt: number;
  expiresAt: number;
}

// Guardar en cache (5 minutos)
function cacheMaterial(material: MaterialDetailResponse) {
  const cache: MaterialCache = {
    materialId: material.id,
    material,
    cachedAt: Date.now(),
    expiresAt: Date.now() + (5 * 60 * 1000)
  };

  localStorage.setItem(`material_${material.id}`, JSON.stringify(cache));
}

// Recuperar de cache
function getCachedMaterial(materialId: string): MaterialDetailResponse | null {
  const cached = localStorage.getItem(`material_${materialId}`);
  if (!cached) return null;

  const cache: MaterialCache = JSON.parse(cached);

  if (Date.now() > cache.expiresAt) {
    localStorage.removeItem(`material_${materialId}`);
    return null;
  }

  return cache.material;
}
```

### Invalidación de Cache

```typescript
// Invalidar cuando se actualiza progreso
function onProgressUpdated(materialId: string) {
  localStorage.removeItem(`material_${materialId}`);
}

// Invalidar cuando status cambia
function onStatusChanged(materialId: string) {
  localStorage.removeItem(`material_${materialId}`);
}
```

## Código de Ejemplo (Mobile - TypeScript)

### Service

```typescript
// services/MaterialService.ts
import axios, { AxiosInstance } from 'axios';

export class MaterialService {
  private api: AxiosInstance;

  constructor(baseURL: string) {
    this.api = axios.create({
      baseURL,
      timeout: 15000
    });

    this.api.interceptors.request.use((config) => {
      const token = localStorage.getItem('access_token');
      if (token) {
        config.headers.Authorization = `Bearer ${token}`;
      }
      return config;
    });
  }

  async getMaterial(materialId: string): Promise<MaterialDetailResponse> {
    // Verificar cache primero
    const cached = getCachedMaterial(materialId);
    if (cached) {
      // Refrescar en background
      this.fetchAndCache(materialId);
      return cached;
    }

    return this.fetchAndCache(materialId);
  }

  private async fetchAndCache(materialId: string): Promise<MaterialDetailResponse> {
    const response = await this.api.get<MaterialDetailResponse>(
      `/v1/materials/${materialId}`
    );

    cacheMaterial(response.data);
    return response.data;
  }
}
```

### Hook

```typescript
// hooks/useMaterial.ts
import { useState, useEffect, useCallback } from 'react';
import { MaterialService } from '../services/MaterialService';

interface UseMaterialOptions {
  autoFetch?: boolean;
  pollWhileProcessing?: boolean;
  pollingInterval?: number;
}

export function useMaterial(
  materialId: string,
  options: UseMaterialOptions = {}
) {
  const {
    autoFetch = true,
    pollWhileProcessing = true,
    pollingInterval = 5000
  } = options;

  const [material, setMaterial] = useState<MaterialDetailResponse | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  const service = new MaterialService(
    process.env.REACT_APP_API_MOBILE_URL || 'http://localhost:8080'
  );

  const fetchMaterial = useCallback(async () => {
    try {
      setLoading(true);
      setError(null);

      const data = await service.getMaterial(materialId);
      setMaterial(data);

      return data;
    } catch (err) {
      setError(err as Error);
      throw err;
    } finally {
      setLoading(false);
    }
  }, [materialId]);

  // Auto-fetch on mount
  useEffect(() => {
    if (autoFetch) {
      fetchMaterial();
    }
  }, [autoFetch, fetchMaterial]);

  // Poll while processing
  useEffect(() => {
    if (!pollWhileProcessing || !material) return;

    if (material.status === 'processing') {
      const interval = setInterval(() => {
        fetchMaterial();
      }, pollingInterval);

      return () => clearInterval(interval);
    }
  }, [material?.status, pollWhileProcessing, pollingInterval, fetchMaterial]);

  return {
    material,
    loading,
    error,
    refetch: fetchMaterial,
    isProcessing: material?.status === 'processing',
    isReady: material?.status === 'ready'
  };
}
```

### Componente

```typescript
// components/MaterialDetail.tsx
import React from 'react';
import { useMaterial } from '../hooks/useMaterial';
import { StatusBadge } from './StatusBadge';
import { ProgressBar } from './ProgressBar';
import { ActionButton } from './ActionButton';

interface MaterialDetailProps {
  materialId: string;
  onViewPDF: () => void;
  onTakeQuiz: () => void;
  onViewSummary: () => void;
}

export function MaterialDetail({
  materialId,
  onViewPDF,
  onTakeQuiz,
  onViewSummary
}: MaterialDetailProps) {
  const {
    material,
    loading,
    error,
    refetch,
    isProcessing
  } = useMaterial(materialId);

  if (loading && !material) {
    return <MaterialDetailSkeleton />;
  }

  if (error) {
    return (
      <ErrorState
        message="Error al cargar el material"
        onRetry={refetch}
      />
    );
  }

  if (!material) {
    return <NotFoundState />;
  }

  return (
    <div className="material-detail">
      {/* Header */}
      <header className="mb-6">
        <h1 className="text-2xl font-bold mb-2">{material.title}</h1>
        <div className="flex items-center gap-4 text-sm text-gray-600">
          <span>📚 {material.subject}</span>
          <span>🎓 Grado {material.grade}</span>
          <span>📄 {formatFileSize(material.file_size_bytes)}</span>
        </div>
        <div className="mt-2">
          <StatusBadge status={material.status} />
        </div>
      </header>

      {/* Progress (if exists) */}
      {material.user_progress && (
        <section className="bg-gray-50 p-4 rounded-lg mb-6">
          <h2 className="font-semibold mb-2">📊 Tu Progreso</h2>
          <ProgressBar
            value={material.user_progress.progress_percentage}
            max={100}
          />
          <p className="text-sm text-gray-600 mt-2">
            Última lectura: página {material.user_progress.last_page}
          </p>
        </section>
      )}

      {/* Processing Indicator */}
      {isProcessing && (
        <div className="bg-yellow-50 border border-yellow-200 p-4 rounded-lg mb-6">
          <div className="flex items-center gap-2">
            <span className="animate-spin">⏳</span>
            <span>Generando contenido con IA...</span>
          </div>
          <p className="text-sm text-gray-600 mt-1">
            El quiz y resumen estarán disponibles en unos minutos.
          </p>
        </div>
      )}

      {/* Action Buttons */}
      <section className="grid grid-cols-3 gap-4 mb-6">
        <ActionButton
          icon="📄"
          label="Ver PDF"
          onClick={onViewPDF}
          disabled={!material.features.pdf_available}
        />
        <ActionButton
          icon="📝"
          label="Quiz"
          onClick={onTakeQuiz}
          disabled={!material.features.quiz_available}
          loading={isProcessing}
        />
        <ActionButton
          icon="📖"
          label="Resumen"
          onClick={onViewSummary}
          disabled={!material.features.summary_available}
          loading={isProcessing}
        />
      </section>

      {/* Description */}
      <section className="mb-6">
        <h2 className="font-semibold mb-2">📝 Descripción</h2>
        <p className="text-gray-700">{material.description}</p>
      </section>

      {/* Teacher Info */}
      {material.teacher && (
        <section className="border-t pt-4">
          <h2 className="font-semibold mb-2">👤 Subido por</h2>
          <p className="text-gray-700">{material.teacher.name}</p>
          <p className="text-sm text-gray-500">
            {formatRelativeDate(material.created_at)}
          </p>
        </section>
      )}
    </div>
  );
}
```

## Notas de Implementación

### Seguridad

1. **Autorización por School:**
   - Backend debe verificar que usuario pertenece a la misma escuela
   - Materiales públicos pueden ser vistos por cualquiera de la escuela

2. **URLs Presigned S3:**
   - `file_url` debe ser URL presigned con expiración corta (15 min)
   - Regenerar URL si expiró

### Performance

1. **Cache Strategy:**
   - Cache de 5 minutos para detalles
   - Invalidar al actualizar progreso
   - Stale-while-revalidate para mejor UX

2. **Polling:**
   - Solo cuando `status === 'processing'`
   - Intervalo de 5 segundos
   - Detener cuando `status === 'ready'` o `failed`

### Testing

```typescript
describe('MaterialDetail', () => {
  it('should display material info correctly', async () => {
    const mockMaterial = createMockMaterial({ status: 'ready' });
    mockApi.get.mockResolvedValue({ data: mockMaterial });

    const { getByText } = render(
      <MaterialDetail materialId="123" {...mockHandlers} />
    );

    await waitFor(() => {
      expect(getByText(mockMaterial.title)).toBeTruthy();
      expect(getByText('✅ Listo')).toBeTruthy();
    });
  });

  it('should poll while processing', async () => {
    const mockMaterial = createMockMaterial({ status: 'processing' });
    mockApi.get.mockResolvedValue({ data: mockMaterial });

    render(<MaterialDetail materialId="123" {...mockHandlers} />);

    // Esperar 6 segundos (más que el intervalo de 5s)
    await waitFor(() => {
      expect(mockApi.get).toHaveBeenCalledTimes(2);
    }, { timeout: 7000 });
  });

  it('should disable actions when not available', async () => {
    const mockMaterial = createMockMaterial({
      status: 'uploaded',
      features: {
        pdf_available: true,
        quiz_available: false,
        summary_available: false
      }
    });
    mockApi.get.mockResolvedValue({ data: mockMaterial });

    const { getByText } = render(
      <MaterialDetail materialId="123" {...mockHandlers} />
    );

    await waitFor(() => {
      const quizButton = getByText('Quiz').closest('button');
      expect(quizButton).toBeDisabled();
    });
  });
});
```

---

**Última actualización:** 2025-12-24
**Versión:** 1.0
**Estado:** ✅ Listo para implementación
