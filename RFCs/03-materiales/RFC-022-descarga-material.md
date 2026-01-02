# RFC-022: Descarga y Visualización de Material

## Metadata
- **ID:** RFC-022
- **Proceso:** Materiales
- **Subproceso:** Descarga y Lectura
- **Prioridad:** Alta
- **Dependencias:** RFC-020 (Listado Materiales), RFC-024 (Progreso Lectura)
- **Estado API:** ✅ Listo

## Descripción
Permite a usuarios autenticados (estudiantes, docentes) descargar y visualizar materiales educativos en formato PDF almacenados en S3. El acceso se realiza mediante URLs presignadas con expiración de 15 minutos para garantizar seguridad. El sistema registra automáticamente el progreso de lectura.

## Flujo de Usuario (UX)

### Como Estudiante
1. **Navegación**: Accede a la lista de materiales de su sección
2. **Selección**: Click en material con estado "ready"
3. **Vista Previa**: Se muestra metadata completa:
   - Título y descripción
   - Resumen IA generado
   - Número de páginas estimado
   - Tamaño del archivo
   - Progreso anterior (si lo tiene)
4. **Opciones de Visualización**:
   - **Leer en línea** (viewer integrado) - Preferido
   - **Descargar PDF** (para lectura offline)
5. **Lectura en Línea**:
   - Viewer PDF responsive
   - Controles: página anterior/siguiente, zoom, pantalla completa
   - Auto-guardado de progreso cada 30 segundos
   - Marcador de última página leída
6. **Descarga**:
   - Click en "Descargar"
   - Se genera URL temporal (15 min)
   - Navegador descarga automáticamente
   - Archivo guardado localmente

### Como Docente
- Mismas opciones que estudiante
- Adicional: Ver quiénes han descargado/leído el material
- Estadísticas de engagement

### Estados Visuales
- ✅ **Material listo**: Botones de lectura/descarga activos
- ⏳ **Procesando**: Botones deshabilitados, mensaje "Generando contenido..."
- 🔴 **Error**: Mensaje de error con opción de reportar

## Flujo de Datos (Técnico)

### Diagrama de Secuencia

```
Cliente            API Mobile         PostgreSQL        S3
  |                     |                  |             |
  |--- GET /materials/:id/download-url -->|             |
  |     + Header: Bearer JWT               |             |
  |                     |                  |             |
  |                     |---SELECT-------->|             |
  |                     |  materials       |             |
  |                     |  WHERE id=?      |             |
  |                     |<--Material-------|             |
  |                     |                  |             |
  |                     |---Validate file_url != null -->|
  |                     |                  |             |
  |                     |---Generate Presigned URL------>|
  |                     |   (GET, expires 15min)         |
  |                     |<--presigned_url----------------|
  |                     |                  |             |
  |<--200 OK------------|                  |             |
  |  {download_url,     |                  |             |
  |   expires_in: 900}  |                  |             |
  |                     |                  |             |
  |===== GET presigned_url (directo a S3) =============>|
  |                     |                  |             |
  |<====== PDF bytes (streaming) =====================--|
  |                     |                  |             |
  |--- PUT /v1/progress (cada 30s) ------>|             |
  |  {progress: 45%,    |                  |             |
  |   last_page: 23}    |                  |             |
  |                     |---UPSERT-------->|             |
  |<--200 OK------------|                  |             |
```

### Endpoints Involucrados

#### 1. Generar URL de Descarga
```http
GET /v1/materials/:id/download-url
Authorization: Bearer {jwt_token}
```

**Response 200 OK:**
```typescript
{
  download_url: "https://s3.amazonaws.com/edugo-materials/...",
  expires_in: 900  // 15 minutos
}
```

**Response 404 Not Found:**
```typescript
{
  error: "not_found",
  message: "Material no encontrado o sin archivo"
}
```

**Response 403 Forbidden:**
```typescript
{
  error: "forbidden",
  message: "No tienes acceso a este material"
}
```

#### 2. Actualizar Progreso de Lectura (Ver RFC-024)
```http
PUT /v1/progress
Authorization: Bearer {jwt_token}
Content-Type: application/json

{
  user_id: "770e8400-e29b-41d4-a716-446655440222",
  material_id: "550e8400-e29b-41d4-a716-446655440000",
  progress_percentage: 45,
  last_page: 23
}
```

### Request/Response (TypeScript)

**GenerateDownloadURLResponse:**
```typescript
interface GenerateDownloadURLResponse {
  download_url: string;  // Presigned URL válida por 15 min
  expires_in: number;    // 900 segundos
}
```

**Opciones de Configuración S3:**
```typescript
// Backend genera URL con parámetros específicos
const params = {
  Bucket: 'edugo-materials',
  Key: material.file_url,
  Expires: 900, // 15 minutos
  ResponseContentType: 'application/pdf',
  ResponseContentDisposition: `inline; filename="${sanitizedFileName}"`,
  // 'inline' para visualizar en navegador
  // 'attachment' para forzar descarga
};
```

## Estados y Transiciones

### Estados del Material para Descarga
```
Material con file_url !== null
  ↓
Solicitar download_url
  ↓
URL presignada generada (válida 15 min)
  ↓
Usuario descarga/visualiza
  ↓
Progreso auto-guardado cada 30s
```

### Expiración de URL
- **Generada**: Timestamp actual
- **Expira**: Timestamp + 15 minutos
- **Comportamiento**: Si usuario intenta acceder después de 15 min, recibe 403 Forbidden de S3
- **Solución Frontend**: Regenerar URL automáticamente si detecta expiración

## Manejo de Errores

### Error 1: Material Sin Archivo
```typescript
try {
  const { download_url } = await api.getMaterialDownloadURL(materialId);
} catch (error) {
  if (error.response?.status === 404) {
    showError(
      'Este material aún no tiene un archivo PDF subido. ' +
      'Contacta al docente.'
    );
  }
}
```

### Error 2: URL Expirada
```typescript
// Detectar error 403 de S3 e intentar regenerar
async function downloadWithRetry(materialId: string) {
  let attempts = 0;
  const MAX_ATTEMPTS = 2;

  while (attempts < MAX_ATTEMPTS) {
    try {
      // Obtener URL presignada
      const { download_url } = await api.getMaterialDownloadURL(materialId);

      // Intentar descargar
      const response = await fetch(download_url);

      if (response.status === 403) {
        // URL expirada, regenerar
        attempts++;
        continue;
      }

      if (!response.ok) {
        throw new Error(`S3 error: ${response.status}`);
      }

      return await response.blob();

    } catch (error) {
      attempts++;
      if (attempts === MAX_ATTEMPTS) {
        throw new Error('No se pudo descargar el material después de 2 intentos');
      }
    }
  }
}
```

### Error 3: Descarga Interrumpida
```typescript
// Usar service worker para reintentar descargas
async function downloadWithResume(materialId: string, fileName: string) {
  const { download_url } = await api.getMaterialDownloadURL(materialId);

  // Iniciar descarga con soporte para Range requests (si S3 lo permite)
  const response = await fetch(download_url, {
    method: 'GET',
    headers: {
      'Range': 'bytes=0-'  // Solicitar desde byte 0
    }
  });

  if (!response.ok) {
    throw new Error('Error descargando PDF');
  }

  // Crear blob y descargar
  const blob = await response.blob();
  const url = URL.createObjectURL(blob);

  const a = document.createElement('a');
  a.href = url;
  a.download = fileName;
  a.click();

  URL.revokeObjectURL(url);
}
```

### Error 4: Sin Conexión Durante Lectura
```typescript
// Detectar offline y mostrar mensaje
window.addEventListener('offline', () => {
  if (isReadingMaterial) {
    showWarning(
      'Conexión perdida. El progreso se guardará cuando vuelvas a conectarte.'
    );
    // Guardar progreso en IndexedDB
    savePendingProgress({
      materialId,
      progress: currentProgress,
      lastPage: currentPage,
      timestamp: Date.now()
    });
  }
});

// Sincronizar cuando vuelva online
window.addEventListener('online', async () => {
  const pendingProgress = await getPendingProgress();

  for (const progress of pendingProgress) {
    try {
      await api.updateProgress(progress);
      await deletePendingProgress(progress.id);
    } catch (error) {
      console.error('Error syncing progress:', error);
    }
  }
});
```

## Consideraciones de UX

### Viewer PDF Integrado
```typescript
// Usar react-pdf o pdf.js
import { Document, Page, pdfjs } from 'react-pdf';

pdfjs.GlobalWorkerOptions.workerSrc = `//cdnjs.cloudflare.com/ajax/libs/pdf.js/${pdfjs.version}/pdf.worker.js`;

function PDFViewer({ materialId }: { materialId: string }) {
  const [numPages, setNumPages] = useState<number>(0);
  const [currentPage, setCurrentPage] = useState<number>(1);
  const [pdfUrl, setPdfUrl] = useState<string>('');

  useEffect(() => {
    // Cargar URL presignada
    api.getMaterialDownloadURL(materialId).then(({ download_url }) => {
      setPdfUrl(download_url);
    });

    // Cargar progreso anterior
    api.getMaterialProgress(materialId).then(({ last_page }) => {
      if (last_page) {
        setCurrentPage(last_page);
      }
    });
  }, [materialId]);

  // Auto-guardar progreso cada 30s
  useEffect(() => {
    if (currentPage === 0 || numPages === 0) return;

    const interval = setInterval(() => {
      const progress = Math.round((currentPage / numPages) * 100);

      api.updateProgress({
        user_id: currentUser.id,
        material_id: materialId,
        progress_percentage: progress,
        last_page: currentPage
      });
    }, 30_000); // 30 segundos

    return () => clearInterval(interval);
  }, [currentPage, numPages, materialId]);

  return (
    <div className="pdf-viewer">
      <Document
        file={pdfUrl}
        onLoadSuccess={({ numPages }) => setNumPages(numPages)}
        loading={<Spinner />}
      >
        <Page
          pageNumber={currentPage}
          renderTextLayer={true}
          renderAnnotationLayer={false}
        />
      </Document>

      <div className="controls">
        <button
          onClick={() => setCurrentPage(p => Math.max(1, p - 1))}
          disabled={currentPage === 1}
        >
          Anterior
        </button>

        <span>
          Página {currentPage} de {numPages}
        </span>

        <button
          onClick={() => setCurrentPage(p => Math.min(numPages, p + 1))}
          disabled={currentPage === numPages}
        >
          Siguiente
        </button>
      </div>

      <ProgressBar
        value={(currentPage / numPages) * 100}
        label={`Progreso: ${Math.round((currentPage / numPages) * 100)}%`}
      />
    </div>
  );
}
```

### Opciones de Descarga
```typescript
// Permitir elegir entre visualizar o descargar
function MaterialActions({ material }: { material: MaterialResponse }) {
  const handleView = () => {
    // Abrir viewer integrado
    navigate(`/materials/${material.id}/read`);
  };

  const handleDownload = async () => {
    try {
      showLoader('Generando enlace de descarga...');

      const { download_url } = await api.getMaterialDownloadURL(material.id);

      // Forzar descarga con 'attachment'
      const response = await fetch(download_url);
      const blob = await response.blob();

      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = `${material.title}.pdf`;
      a.click();

      URL.revokeObjectURL(url);

      hideLoader();
      showSuccess('Descarga iniciada');

    } catch (error) {
      hideLoader();
      showError('Error al descargar el material');
    }
  };

  return (
    <div className="flex gap-2">
      <Button onClick={handleView} variant="primary">
        <EyeIcon /> Leer en línea
      </Button>

      <Button onClick={handleDownload} variant="secondary">
        <DownloadIcon /> Descargar PDF
      </Button>
    </div>
  );
}
```

### Indicador de Tiempo de Lectura
```typescript
// Estimar tiempo de lectura basado en número de páginas
function estimateReadingTime(numPages: number): string {
  // Asumir 2 minutos por página promedio
  const totalMinutes = numPages * 2;

  if (totalMinutes < 60) {
    return `~${totalMinutes} minutos`;
  }

  const hours = Math.floor(totalMinutes / 60);
  const minutes = totalMinutes % 60;

  return `~${hours}h ${minutes}m`;
}

// Uso
<div className="text-sm text-gray-500">
  <ClockIcon />
  Tiempo estimado: {estimateReadingTime(material.page_count)}
</div>
```

### Marcador de Última Posición
```typescript
// Mostrar indicador visual de última página leída
function PageIndicator({ currentPage, lastReadPage, numPages }) {
  return (
    <div className="flex items-center gap-2">
      <span>Página {currentPage} de {numPages}</span>

      {lastReadPage && lastReadPage !== currentPage && (
        <Badge variant="info">
          Última leída: página {lastReadPage}
        </Badge>
      )}

      <Button
        size="sm"
        onClick={() => setCurrentPage(lastReadPage)}
        disabled={currentPage === lastReadPage}
      >
        Ir a última página leída
      </Button>
    </div>
  );
}
```

## Almacenamiento Local

### Cache de PDFs Descargados
```typescript
// Usar IndexedDB para cachear PDFs (opcional, para offline)
interface CachedPDF {
  materialId: string;
  blob: Blob;
  cachedAt: number;
  expiresAt: number;
}

async function cachePDF(materialId: string, blob: Blob) {
  const expiresAt = Date.now() + (24 * 60 * 60 * 1000); // 24 horas

  await db.cachedPDFs.put({
    materialId,
    blob,
    cachedAt: Date.now(),
    expiresAt
  });
}

async function getCachedPDF(materialId: string): Promise<Blob | null> {
  const cached = await db.cachedPDFs.get(materialId);

  if (!cached) return null;

  // Verificar expiración
  if (Date.now() > cached.expiresAt) {
    await db.cachedPDFs.delete(materialId);
    return null;
  }

  return cached.blob;
}

// Uso en componente
async function loadPDF(materialId: string) {
  // Intentar cargar desde cache primero
  let blob = await getCachedPDF(materialId);

  if (blob) {
    console.log('PDF loaded from cache');
    return URL.createObjectURL(blob);
  }

  // Si no está en cache, descargar
  const { download_url } = await api.getMaterialDownloadURL(materialId);
  const response = await fetch(download_url);
  blob = await response.blob();

  // Cachear para futuro uso
  await cachePDF(materialId, blob);

  return URL.createObjectURL(blob);
}
```

### Progreso Pendiente de Sync
```typescript
// Guardar progreso offline para sincronizar después
interface PendingProgress {
  id: string;
  materialId: string;
  userId: string;
  progressPercentage: number;
  lastPage: number;
  timestamp: number;
}

async function savePendingProgress(progress: Omit<PendingProgress, 'id'>) {
  await db.pendingProgress.put({
    id: `${progress.materialId}_${Date.now()}`,
    ...progress
  });
}

async function syncPendingProgress() {
  const pending = await db.pendingProgress.toArray();

  for (const progress of pending) {
    try {
      await api.updateProgress({
        user_id: progress.userId,
        material_id: progress.materialId,
        progress_percentage: progress.progressPercentage,
        last_page: progress.lastPage
      });

      // Eliminar de pending después de sync exitoso
      await db.pendingProgress.delete(progress.id);

    } catch (error) {
      console.error('Error syncing progress:', error);
      // Dejarlo en pending para reintentar después
    }
  }
}

// Sincronizar al volver online
window.addEventListener('online', syncPendingProgress);
```

## Código de Ejemplo (Mobile - TypeScript)

### Hook para Descarga
```typescript
import { useQuery } from '@tanstack/react-query';
import { materialsApi } from '@/services/api';

export function useMaterialDownloadURL(materialId: string, enabled = true) {
  return useQuery({
    queryKey: ['material-download-url', materialId],
    queryFn: () => materialsApi.getDownloadURL(materialId),
    enabled,
    staleTime: 10 * 60 * 1000, // 10 minutos (URL expira en 15)
    cacheTime: 0, // No cachear (URL expira)
    retry: 2,
    retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 3000)
  });
}

// Uso
function MaterialDownloadButton({ materialId }: { materialId: string }) {
  const { data, isLoading, error, refetch } = useMaterialDownloadURL(
    materialId,
    false // No cargar automáticamente
  );

  const handleDownload = async () => {
    try {
      // Generar URL si no existe
      if (!data) {
        await refetch();
      }

      const url = data?.download_url;
      if (!url) return;

      // Descargar
      const response = await fetch(url);
      const blob = await response.blob();

      // Triggear descarga
      const objectURL = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = objectURL;
      a.download = `material-${materialId}.pdf`;
      a.click();

      URL.revokeObjectURL(objectURL);

    } catch (error) {
      console.error('Download error:', error);
      showError('Error al descargar el material');
    }
  };

  return (
    <Button onClick={handleDownload} disabled={isLoading}>
      {isLoading ? <Spinner /> : <DownloadIcon />}
      Descargar
    </Button>
  );
}
```

### Service API
```typescript
// services/api/materials.ts
export const materialsApi = {
  async getDownloadURL(materialId: string): Promise<GenerateDownloadURLResponse> {
    const response = await api.get(`/v1/materials/${materialId}/download-url`);
    return response.data;
  },

  async downloadPDF(materialId: string): Promise<Blob> {
    const { download_url } = await this.getDownloadURL(materialId);

    const response = await fetch(download_url);

    if (!response.ok) {
      throw new Error(`Failed to download PDF: ${response.status}`);
    }

    return await response.blob();
  }
};
```

## Notas de Implementación

### Seguridad
- ✅ URLs presignadas con expiración de 15 minutos
- ✅ JWT requerido para generar URL
- ✅ S3 bucket con acceso privado (no público)
- ✅ Permisos de lectura solo con URL firmada
- ⚠️ Sin autorización granular (cualquier usuario autenticado puede acceder a cualquier material)

**Recomendación:** Implementar validación de acceso basada en `school_id` y `academic_unit_id`

### Performance
- Cache de URLs presignadas por 10 minutos (menor que expiración)
- Lazy load de PDFs (solo cuando usuario click)
- Streaming de PDFs grandes (no cargar completo en memoria)
- Service Worker para cache offline (opcional)

### Limitaciones
- ⚠️ Sin soporte para marcadores/bookmarks personalizados
- ⚠️ Sin resaltado de texto
- ⚠️ Sin anotaciones en PDF
- ⚠️ Sin búsqueda dentro del PDF (depende del viewer)

### Mejoras Futuras
1. **Anotaciones**: Permitir resaltar y comentar
2. **Bookmarks**: Guardar marcadores personalizados
3. **Compartir**: Compartir material con otros usuarios
4. **Impresión**: Controlar permisos de impresión
5. **DRM**: Protección contra copia (si requerido)

### Testing
```typescript
describe('Material Download', () => {
  it('genera URL presignada válida', async () => {
    const materialId = 'test-uuid';

    const { download_url, expires_in } = await materialsApi.getDownloadURL(materialId);

    expect(download_url).toContain('s3.amazonaws.com');
    expect(expires_in).toBe(900);
  });

  it('descarga PDF exitosamente', async () => {
    const materialId = 'test-uuid';

    const blob = await materialsApi.downloadPDF(materialId);

    expect(blob.type).toBe('application/pdf');
    expect(blob.size).toBeGreaterThan(0);
  });

  it('maneja error 404 si material no existe', async () => {
    const invalidId = 'invalid-uuid';

    await expect(
      materialsApi.getDownloadURL(invalidId)
    ).rejects.toThrow('404');
  });

  it('regenera URL si está expirada', async () => {
    const materialId = 'test-uuid';

    // Primera generación
    const first = await materialsApi.getDownloadURL(materialId);

    // Simular expiración (esperar 16 minutos)
    jest.advanceTimersByTime(16 * 60 * 1000);

    // Segunda generación (debe ser diferente)
    const second = await materialsApi.getDownloadURL(materialId);

    expect(first.download_url).not.toBe(second.download_url);
  });
});
```

---

**Última actualización:** 2025-12-24
**Versión:** 1.0
**Estado:** ✅ Listo para implementación
