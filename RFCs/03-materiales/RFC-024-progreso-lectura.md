# RFC-024: Registro de Progreso de Lectura

## Metadata
- **ID:** RFC-024
- **Proceso:** Materiales
- **Subproceso:** Tracking de Lectura
- **Prioridad:** Alta
- **Dependencias:** RFC-022 (Descarga Material)
- **Estado API:** ✅ Listo

## Descripción
Sistema de registro de progreso de lectura que permite a estudiantes guardar automáticamente su avance al leer materiales PDF. El progreso se sincroniza en tiempo real con el backend y se muestra visualmente mediante barras de progreso. Al completar 100% de un material, se emite un evento de completitud.

## Flujo de Usuario (UX)

### Como Estudiante
1. **Inicio de Lectura**: Abre material PDF en viewer integrado
2. **Restauración**: Si ya había leído antes, se restaura última página
3. **Lectura**:
   - Navega por páginas del PDF
   - Progreso se calcula automáticamente (página actual / total páginas)
   - Barra de progreso se actualiza en tiempo real
4. **Auto-Guardado**:
   - Cada 30 segundos mientras lee
   - Al cambiar de página importante (múltiplo de 5)
   - Al cerrar viewer o salir de la página
5. **Completitud**:
   - Al llegar a última página → 100%
   - Animación de celebración
   - Badge "Completado" en lista de materiales
6. **Sincronización**:
   - Offline: Progreso guardado en IndexedDB
   - Online: Sincroniza automáticamente al reconectar

### Estados Visuales
- 📖 **Leyendo**: Barra de progreso animada, página actual resaltada
- 💾 **Guardando**: Icono de sync girando
- ✅ **Completado**: Badge verde, confetti animation
- 📡 **Offline**: Banner amarillo "Progreso se guardará al reconectar"

### Indicadores de Progreso
```
[████████████████░░░░] 75%
Página 45 de 60 • Última sesión: hace 2 horas
```

## Flujo de Datos (Técnico)

### Diagrama de Secuencia

```
Cliente          API Mobile      PostgreSQL      RabbitMQ
  |                   |               |              |
  |--- GET /materials/:id ----------->|              |
  |                   |               |              |
  |                   |---SELECT----->|              |
  |                   |  material     |              |
  |                   |<--Material----|              |
  |                   |               |              |
  |<--200 OK----------|               |              |
  |  {status: ready}  |               |              |
  |                   |               |              |
  |--- GET progreso anterior -------->|              |
  |                   |               |              |
  |                   |---SELECT----->|              |
  |                   |  FROM material_progress      |
  |                   |  WHERE user_id=? AND material_id=?
  |                   |<--Progress----|              |
  |                   |  (45%, pg 27) |              |
  |                   |               |              |
  |<--200 OK----------|               |              |
  |  {progress: 45%,  |               |              |
  |   last_page: 27}  |               |              |
  |                   |               |              |
  |--- Restaurar página 27 (cliente) |              |
  |                   |               |              |
  |-- (lectura por 30s) --|           |              |
  |                   |               |              |
  |--- PUT /v1/progress ------------->|              |
  |  {progress: 50%,  |               |              |
  |   last_page: 30}  |               |              |
  |                   |               |              |
  |                   |---UPSERT----->|              |
  |                   |  INSERT INTO material_progress
  |                   |  ON CONFLICT (user_id, material_id, school_id)
  |                   |  DO UPDATE SET progress_percentage=?, last_page=?, updated_at=NOW()
  |                   |<--OK----------|              |
  |                   |               |              |
  |<--200 OK----------|               |              |
  |                   |               |              |
  |-- (lectura hasta el final) --|   |              |
  |                   |               |              |
  |--- PUT /v1/progress ------------->|              |
  |  {progress: 100%, |               |              |
  |   last_page: 60}  |               |              |
  |                   |               |              |
  |                   |---UPSERT----->|              |
  |                   |<--OK----------|              |
  |                   |               |              |
  |                   |---PUBLISH EVENT ------------->
  |                   |  material.completed          |
  |                   |               |              |
  |<--200 OK----------|               |              |
  |  {completed: true}|               |              |
  |                   |               |              |
  |--- Mostrar celebración (cliente) |              |
```

### Endpoints Involucrados

#### 1. Actualizar Progreso (UPSERT)
```http
PUT /v1/progress
Authorization: Bearer {jwt_token}
Content-Type: application/json

{
  "user_id": "770e8400-e29b-41d4-a716-446655440222",
  "material_id": "550e8400-e29b-41d4-a716-446655440000",
  "progress_percentage": 75,
  "last_page": 45
}
```

**Response 200 OK:**
```typescript
{
  user_id: "770e8400-e29b-41d4-a716-446655440222",
  material_id: "550e8400-e29b-41d4-a716-446655440000",
  progress_percentage: 75,
  last_page: 45,
  updated_at: "2024-12-24T10:30:00Z"
}
```

**Response 403 Forbidden:**
```typescript
{
  error: "forbidden",
  message: "Solo puedes actualizar tu propio progreso"
}
```

#### 2. Obtener Progreso Actual (Opcional - puede usar GET /materials/:id)
```http
GET /v1/progress?user_id={uuid}&material_id={uuid}
Authorization: Bearer {jwt_token}
```

**Response 200 OK:**
```typescript
{
  user_id: "uuid",
  material_id: "uuid",
  progress_percentage: 45,
  last_page: 27,
  updated_at: "2024-12-24T09:00:00Z"
}
```

**Response 404 Not Found:**
```typescript
{
  error: "not_found",
  message: "No hay progreso registrado para este material"
}
```

### Request/Response (TypeScript)

**UpsertProgressRequest:**
```typescript
interface UpsertProgressRequest {
  user_id: string;           // UUID (debe coincidir con JWT)
  material_id: string;       // UUID
  progress_percentage: number; // 0-100
  last_page?: number;        // Opcional
}
```

**ProgressResponse:**
```typescript
interface ProgressResponse {
  user_id: string;
  material_id: string;
  progress_percentage: number;
  last_page: number;
  updated_at: string; // ISO8601
}
```

**Evento RabbitMQ (material.completed):**
```typescript
interface MaterialCompletedEvent {
  event_type: "material.completed";
  material_id: string;
  school_id: string;
  user_id: string;
  completed_at: string; // ISO8601
}
```

## Estados y Transiciones

### Estados del Progreso
```
Sin progreso (null)
  ↓
0% (primera vez que abre)
  ↓
1-99% (leyendo)
  ↓
100% (completado)
```

### Transiciones
- **null → 0%**: Primer acceso al material
- **0% → N%**: Usuario navega por páginas
- **N% → 100%**: Usuario llega a última página
- **100% → 100%**: Idempotente (puede releer sin afectar)

### Cálculo de Progreso
```typescript
function calculateProgress(currentPage: number, totalPages: number): number {
  if (totalPages === 0) return 0;
  return Math.round((currentPage / totalPages) * 100);
}

// Ejemplos
calculateProgress(1, 60);   // 2%
calculateProgress(30, 60);  // 50%
calculateProgress(60, 60);  // 100%
```

## Manejo de Errores

### Error 1: User ID No Coincide con JWT
```typescript
try {
  await api.updateProgress({
    user_id: 'otro-usuario-uuid', // ❌ No coincide con JWT
    material_id: materialId,
    progress_percentage: 50
  });
} catch (error) {
  if (error.response?.status === 403) {
    // Backend rechaza
    console.error('Intento de actualizar progreso de otro usuario');

    // Usar user_id correcto del JWT
    const currentUserId = getCurrentUserIdFromToken();

    await api.updateProgress({
      user_id: currentUserId, // ✅ Correcto
      material_id: materialId,
      progress_percentage: 50
    });
  }
}
```

### Error 2: Progreso Fuera de Rango
```typescript
// Validación cliente antes de enviar
function validateProgress(percentage: number): boolean {
  return percentage >= 0 && percentage <= 100;
}

// Uso
const progress = calculateProgress(currentPage, totalPages);

if (!validateProgress(progress)) {
  console.error('Progreso inválido:', progress);
  return; // No enviar al backend
}

await api.updateProgress({
  user_id: currentUser.id,
  material_id: materialId,
  progress_percentage: Math.max(0, Math.min(100, progress)) // Clamp
});
```

### Error 3: Sincronización Offline
```typescript
// Guardar en IndexedDB si offline
async function updateProgressWithFallback(
  progress: UpsertProgressRequest
): Promise<void> {
  if (!navigator.onLine) {
    // Guardar offline
    await db.pendingProgress.put({
      id: `${progress.material_id}_${Date.now()}`,
      ...progress,
      timestamp: Date.now()
    });

    showInfo('Sin conexión. Progreso guardado localmente.');
    return;
  }

  try {
    // Intentar sincronizar
    await api.updateProgress(progress);

  } catch (error) {
    // Si falla, guardar offline
    await db.pendingProgress.put({
      id: `${progress.material_id}_${Date.now()}`,
      ...progress,
      timestamp: Date.now()
    });

    console.error('Error syncing progress, saved offline:', error);
  }
}

// Sincronizar cuando vuelva online
window.addEventListener('online', async () => {
  const pending = await db.pendingProgress.toArray();

  for (const progress of pending) {
    try {
      await api.updateProgress(progress);
      await db.pendingProgress.delete(progress.id);
      console.log('Progress synced:', progress.id);
    } catch (error) {
      console.error('Error syncing:', error);
    }
  }
});
```

### Error 4: Rate Limiting (Demasiadas Actualizaciones)
```typescript
// Debounce para evitar sobrecarga del servidor
import { debounce } from 'lodash-es';

const debouncedUpdateProgress = debounce(
  async (progress: UpsertProgressRequest) => {
    try {
      await api.updateProgress(progress);
    } catch (error) {
      if (error.response?.status === 429) {
        showWarning('Demasiadas actualizaciones. Espera un momento.');
      }
    }
  },
  5000, // 5 segundos
  { leading: true, trailing: true }
);

// Uso
function onPageChange(currentPage: number, totalPages: number) {
  const progress = calculateProgress(currentPage, totalPages);

  debouncedUpdateProgress({
    user_id: currentUser.id,
    material_id: materialId,
    progress_percentage: progress,
    last_page: currentPage
  });
}
```

## Consideraciones de UX

### Auto-Guardado Transparente
```typescript
// Hook para auto-guardar progreso
import { useEffect, useRef } from 'react';

function useAutoSaveProgress(
  materialId: string,
  currentPage: number,
  totalPages: number
) {
  const intervalRef = useRef<NodeJS.Timeout>();
  const lastSavedRef = useRef<number>(0);

  useEffect(() => {
    // Guardar inmediatamente si es página importante (múltiplo de 5)
    if (currentPage % 5 === 0 && currentPage !== lastSavedRef.current) {
      saveProgress(currentPage);
    }

    // Auto-guardar cada 30 segundos
    intervalRef.current = setInterval(() => {
      if (currentPage !== lastSavedRef.current) {
        saveProgress(currentPage);
      }
    }, 30_000);

    return () => {
      if (intervalRef.current) {
        clearInterval(intervalRef.current);
      }

      // Guardar al desmontar (salir de lectura)
      if (currentPage !== lastSavedRef.current) {
        saveProgress(currentPage);
      }
    };
  }, [currentPage, totalPages, materialId]);

  async function saveProgress(page: number) {
    const progress = calculateProgress(page, totalPages);

    await api.updateProgress({
      user_id: currentUser.id,
      material_id: materialId,
      progress_percentage: progress,
      last_page: page
    });

    lastSavedRef.current = page;
  }
}

// Uso en componente
function PDFReader({ materialId }: { materialId: string }) {
  const [currentPage, setCurrentPage] = useState(1);
  const [numPages, setNumPages] = useState(0);

  // Auto-guardar progreso
  useAutoSaveProgress(materialId, currentPage, numPages);

  return (
    <PDFViewer
      onPageChange={setCurrentPage}
      onLoadSuccess={({ numPages }) => setNumPages(numPages)}
    />
  );
}
```

### Indicador de Guardado
```typescript
function SavingIndicator({ isSaving }: { isSaving: boolean }) {
  if (!isSaving) return null;

  return (
    <div className="flex items-center gap-2 text-sm text-gray-500">
      <Spinner size="sm" className="animate-spin" />
      <span>Guardando progreso...</span>
    </div>
  );
}

// Uso con estado de loading
function PDFReader({ materialId }: { materialId: string }) {
  const [isSaving, setIsSaving] = useState(false);

  const saveProgress = async (page: number) => {
    setIsSaving(true);

    try {
      await api.updateProgress({...});
    } finally {
      setIsSaving(false);
    }
  };

  return (
    <div>
      <SavingIndicator isSaving={isSaving} />
      <PDFViewer onPageChange={saveProgress} />
    </div>
  );
}
```

### Celebración al Completar
```typescript
import confetti from 'canvas-confetti';

function onMaterialCompleted() {
  // Animación confetti
  confetti({
    particleCount: 100,
    spread: 70,
    origin: { y: 0.6 }
  });

  // Mostrar modal de celebración
  showModal({
    title: '¡Felicitaciones! 🎉',
    message: 'Has completado este material',
    actions: [
      {
        label: 'Continuar al quiz',
        onClick: () => navigate(`/materials/${materialId}/assessment`)
      },
      {
        label: 'Volver a lista',
        onClick: () => navigate('/materials')
      }
    ]
  });

  // Actualizar badge en lista
  queryClient.invalidateQueries(['materials']);
}

// Detectar completitud
useEffect(() => {
  if (progress === 100 && !wasCompleted) {
    onMaterialCompleted();
    setWasCompleted(true);
  }
}, [progress]);
```

### Barra de Progreso Visual
```typescript
function ProgressBar({ value, label }: { value: number; label?: string }) {
  return (
    <div className="w-full">
      {label && (
        <div className="flex justify-between mb-1 text-sm">
          <span>{label}</span>
          <span className="font-semibold">{value}%</span>
        </div>
      )}

      <div className="w-full bg-gray-200 rounded-full h-2.5">
        <div
          className="bg-blue-600 h-2.5 rounded-full transition-all duration-300"
          style={{ width: `${value}%` }}
        />
      </div>
    </div>
  );
}

// Con colores según progreso
function getProgressColor(value: number): string {
  if (value < 25) return 'bg-red-500';
  if (value < 50) return 'bg-orange-500';
  if (value < 75) return 'bg-yellow-500';
  if (value < 100) return 'bg-blue-500';
  return 'bg-green-500';
}
```

### Última Sesión
```typescript
import { formatDistanceToNow } from 'date-fns';
import { es } from 'date-fns/locale';

function LastReadInfo({ progress }: { progress: ProgressResponse }) {
  const timeAgo = formatDistanceToNow(
    new Date(progress.updated_at),
    { addSuffix: true, locale: es }
  );

  return (
    <div className="text-sm text-gray-500">
      <ClockIcon className="inline mr-1" />
      Última sesión {timeAgo}
      {progress.last_page && (
        <span> • Página {progress.last_page}</span>
      )}
    </div>
  );
}
```

## Almacenamiento Local

### Progreso Pendiente de Sync
```typescript
// Estructura en IndexedDB
interface PendingProgress {
  id: string;                    // `${materialId}_${timestamp}`
  user_id: string;
  material_id: string;
  progress_percentage: number;
  last_page: number;
  timestamp: number;             // Cuándo se creó offline
  syncAttempts: number;          // Reintentos fallidos
}

// Guardar offline
async function savePendingProgress(progress: UpsertProgressRequest) {
  await db.pendingProgress.put({
    id: `${progress.material_id}_${Date.now()}`,
    ...progress,
    timestamp: Date.now(),
    syncAttempts: 0
  });
}

// Sincronizar con retry exponencial
async function syncPendingProgress() {
  const pending = await db.pendingProgress.toArray();

  for (const progress of pending) {
    try {
      await api.updateProgress(progress);

      // Éxito: eliminar de pending
      await db.pendingProgress.delete(progress.id);

    } catch (error) {
      // Incrementar intentos
      progress.syncAttempts++;

      if (progress.syncAttempts >= 5) {
        // Demasiados intentos: eliminar o notificar al usuario
        console.error('Max sync attempts reached:', progress.id);
        await db.pendingProgress.delete(progress.id);
      } else {
        // Actualizar con nuevo intento
        await db.pendingProgress.put(progress);
      }
    }
  }
}
```

### Cache de Progreso
```typescript
// Cachear último progreso para mostrar inmediatamente
interface CachedProgress {
  materialId: string;
  progress: ProgressResponse;
  cachedAt: number;
}

async function cacheProgress(
  materialId: string,
  progress: ProgressResponse
) {
  await db.cachedProgress.put({
    materialId,
    progress,
    cachedAt: Date.now()
  });
}

async function getCachedProgress(
  materialId: string
): Promise<ProgressResponse | null> {
  const cached = await db.cachedProgress.get(materialId);

  if (!cached) return null;

  // Cache válido por 1 hora
  if (Date.now() - cached.cachedAt > 60 * 60 * 1000) {
    await db.cachedProgress.delete(materialId);
    return null;
  }

  return cached.progress;
}
```

## Código de Ejemplo (Mobile - TypeScript)

### Hook Completo de Progreso
```typescript
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query';
import { progressApi } from '@/services/api';
import { useAuth } from '@/hooks/useAuth';

export function useMaterialProgress(materialId: string) {
  const { currentUser } = useAuth();
  const queryClient = useQueryClient();

  // Obtener progreso actual
  const { data: progress } = useQuery({
    queryKey: ['progress', materialId, currentUser.id],
    queryFn: () => progressApi.get(currentUser.id, materialId),
    enabled: !!currentUser.id && !!materialId,
    staleTime: 5 * 60 * 1000 // 5 min
  });

  // Actualizar progreso
  const updateMutation = useMutation({
    mutationFn: (data: Omit<UpsertProgressRequest, 'user_id'>) =>
      progressApi.update({
        user_id: currentUser.id,
        material_id: materialId,
        ...data
      }),

    onMutate: async (newProgress) => {
      // Optimistic update
      await queryClient.cancelQueries({
        queryKey: ['progress', materialId, currentUser.id]
      });

      const previousProgress = queryClient.getQueryData([
        'progress',
        materialId,
        currentUser.id
      ]);

      queryClient.setQueryData(
        ['progress', materialId, currentUser.id],
        {
          ...previousProgress,
          progress_percentage: newProgress.progress_percentage,
          last_page: newProgress.last_page,
          updated_at: new Date().toISOString()
        }
      );

      return { previousProgress };
    },

    onError: (err, newProgress, context) => {
      // Revertir optimistic update
      queryClient.setQueryData(
        ['progress', materialId, currentUser.id],
        context?.previousProgress
      );
    },

    onSettled: () => {
      // Invalidar cache
      queryClient.invalidateQueries({
        queryKey: ['progress', materialId, currentUser.id]
      });
    }
  });

  return {
    progress,
    updateProgress: updateMutation.mutate,
    updateProgressAsync: updateMutation.mutateAsync,
    isUpdating: updateMutation.isPending
  };
}

// Uso en componente PDFReader
function PDFReader({ materialId }: { materialId: string }) {
  const [currentPage, setCurrentPage] = useState(1);
  const [numPages, setNumPages] = useState(0);

  const { progress, updateProgress, isUpdating } = useMaterialProgress(materialId);

  // Restaurar última página leída
  useEffect(() => {
    if (progress?.last_page) {
      setCurrentPage(progress.last_page);
    }
  }, [progress]);

  // Auto-guardar progreso
  useEffect(() => {
    if (currentPage === 0 || numPages === 0) return;

    const interval = setInterval(() => {
      const progressPercentage = calculateProgress(currentPage, numPages);

      updateProgress({
        progress_percentage: progressPercentage,
        last_page: currentPage
      });
    }, 30_000); // Cada 30 segundos

    return () => clearInterval(interval);
  }, [currentPage, numPages, updateProgress]);

  // Guardar al cambiar página importante
  const handlePageChange = (newPage: number) => {
    setCurrentPage(newPage);

    if (newPage % 5 === 0) {
      const progressPercentage = calculateProgress(newPage, numPages);

      updateProgress({
        progress_percentage: progressPercentage,
        last_page: newPage
      });
    }
  };

  return (
    <div>
      <ProgressBar
        value={progress?.progress_percentage || 0}
        label={`Progreso de lectura`}
      />

      <SavingIndicator isSaving={isUpdating} />

      <Document
        file={pdfUrl}
        onLoadSuccess={({ numPages }) => setNumPages(numPages)}
      >
        <Page pageNumber={currentPage} />
      </Document>

      <PageControls
        currentPage={currentPage}
        totalPages={numPages}
        onPageChange={handlePageChange}
      />

      {progress && <LastReadInfo progress={progress} />}
    </div>
  );
}
```

### Service API
```typescript
// services/api/progress.ts
export const progressApi = {
  async get(userId: string, materialId: string): Promise<ProgressResponse> {
    const response = await api.get('/v1/progress', {
      params: { user_id: userId, material_id: materialId }
    });
    return response.data;
  },

  async update(data: UpsertProgressRequest): Promise<ProgressResponse> {
    const response = await api.put('/v1/progress', data);
    return response.data;
  }
};
```

## Notas de Implementación

### Backend: Tabla material_progress
```sql
CREATE TABLE material_progress (
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  material_id UUID NOT NULL REFERENCES materials(id) ON DELETE CASCADE,
  school_id UUID NOT NULL REFERENCES schools(id) ON DELETE CASCADE,
  progress_percentage INTEGER NOT NULL CHECK (progress_percentage >= 0 AND progress_percentage <= 100),
  last_page INTEGER,
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,

  PRIMARY KEY (user_id, material_id, school_id)
);

CREATE INDEX idx_material_progress_user_id ON material_progress(user_id);
CREATE INDEX idx_material_progress_material_id ON material_progress(material_id);
CREATE INDEX idx_material_progress_updated_at ON material_progress(updated_at DESC);
```

### Operación UPSERT
```sql
-- PostgreSQL UPSERT
INSERT INTO material_progress (
  user_id,
  material_id,
  school_id,
  progress_percentage,
  last_page,
  updated_at
) VALUES ($1, $2, $3, $4, $5, NOW())
ON CONFLICT (user_id, material_id, school_id)
DO UPDATE SET
  progress_percentage = EXCLUDED.progress_percentage,
  last_page = EXCLUDED.last_page,
  updated_at = NOW();
```

### Evento material.completed
```go
// Backend emite evento cuando progreso = 100%
if req.ProgressPercentage == 100 {
    event := MaterialCompletedEvent{
        EventType:    "material.completed",
        MaterialID:   req.MaterialID,
        SchoolID:     claims.SchoolID,
        UserID:       req.UserID,
        CompletedAt:  time.Now(),
    }

    s.publisher.Publish(ctx, "edugo.events", "material.completed", event)
}
```

### Limitaciones Actuales
- ⚠️ Sin tracking de tiempo de lectura
- ⚠️ Sin analytics de páginas más leídas
- ⚠️ Sin detección de velocidad de lectura

### Mejoras Futuras
1. **Tiempo de Lectura**: Registrar cuánto tiempo pasa en cada página
2. **Analytics**: Heatmap de páginas más leídas
3. **Velocidad**: Estimar velocidad de lectura (páginas/minuto)
4. **Highlights**: Guardar fragmentos destacados por el estudiante
5. **Bookmarks**: Marcadores personalizados en páginas específicas

### Seguridad
- ✅ Usuario solo puede actualizar su propio progreso
- ✅ Admin puede actualizar progreso de cualquier usuario
- ✅ Validación de rango 0-100%
- ✅ Idempotente (UPSERT)

### Testing
```typescript
describe('Progress Tracking', () => {
  it('calcula progreso correctamente', () => {
    expect(calculateProgress(30, 60)).toBe(50);
    expect(calculateProgress(1, 60)).toBe(2);
    expect(calculateProgress(60, 60)).toBe(100);
  });

  it('guarda progreso offline', async () => {
    // Simular offline
    Object.defineProperty(navigator, 'onLine', { value: false });

    await updateProgressWithFallback({
      user_id: 'uuid',
      material_id: 'uuid',
      progress_percentage: 50
    });

    const pending = await db.pendingProgress.toArray();
    expect(pending).toHaveLength(1);
    expect(pending[0].progress_percentage).toBe(50);
  });

  it('sincroniza progreso al volver online', async () => {
    // Guardar offline
    await savePendingProgress({
      user_id: 'uuid',
      material_id: 'uuid',
      progress_percentage: 75,
      last_page: 45
    });

    // Simular online
    Object.defineProperty(navigator, 'onLine', { value: true });

    await syncPendingProgress();

    const pending = await db.pendingProgress.toArray();
    expect(pending).toHaveLength(0);

    expect(mockApi.put).toHaveBeenCalledWith('/v1/progress', {
      user_id: 'uuid',
      material_id: 'uuid',
      progress_percentage: 75,
      last_page: 45
    });
  });

  it('emite evento al completar 100%', async () => {
    const { updateProgress } = renderHook(() =>
      useMaterialProgress('material-uuid')
    ).result.current;

    await updateProgress({
      progress_percentage: 100,
      last_page: 60
    });

    expect(mockRabbitMQ.publish).toHaveBeenCalledWith(
      expect.objectContaining({
        event_type: 'material.completed'
      })
    );
  });
});
```

---

**Última actualización:** 2025-12-24
**Versión:** 1.0
**Estado:** ✅ Listo para implementación
