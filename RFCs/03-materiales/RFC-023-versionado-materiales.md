# RFC-023: Versionado de Materiales

## Metadata
- **ID:** RFC-023
- **Proceso:** Materiales
- **Subproceso:** Actualización y Versionado
- **Prioridad:** Media
- **Dependencias:** RFC-020 (Listado), RFC-021 (Subida PDF)
- **Estado API:** ⚠️ Parcial (PUT /materials/:id solo en dev)

## Descripción
Sistema de control de versiones para materiales educativos que permite a docentes actualizar metadata y archivo PDF manteniendo un historial completo de cambios. Cada actualización del PDF genera una nueva versión, preservando versiones anteriores para auditoría y posible reversión.

## Flujo de Usuario (UX)

### Como Docente
1. **Acceso**: Navega a material propio (uploaded_by_teacher_id coincide)
2. **Opciones de Edición**:
   - **Editar metadata** (título, descripción, materia, grado) - Sin crear versión nueva
   - **Actualizar PDF** - Genera versión nueva automáticamente
3. **Editar Metadata**:
   - Click en "Editar"
   - Formulario precargado con valores actuales
   - Cambiar campos deseados
   - Guardar → Material actualizado, `updated_at` cambia
4. **Actualizar PDF**:
   - Click en "Actualizar archivo"
   - Confirmación: "¿Crear nueva versión? La versión actual se archivará"
   - Acepta → Flujo similar a RFC-021 (subida PDF)
   - Nueva versión creada → Versión anterior archivada
   - Worker reprocesa PDF → Nuevo resumen y quiz
5. **Ver Historial**:
   - Click en "Ver versiones"
   - Lista cronológica de versiones:
     - Versión 3 (actual) - 24 Dic 2024, 10:30 AM
     - Versión 2 - 20 Dic 2024, 3:45 PM
     - Versión 1 - 15 Dic 2024, 9:00 AM
   - Cada versión muestra: fecha, quién la creó, tamaño archivo
6. **Restaurar Versión** (Futuro):
   - Seleccionar versión anterior
   - Click "Restaurar" → Crea nueva versión copiando la antigua

### Como Estudiante
- Ve la versión más reciente automáticamente
- NO ve historial de versiones (transparente para ellos)
- Su progreso de lectura se mantiene entre versiones (basado en material_id)

### Estados Visuales
- 📝 **Editando metadata**: Modal/form con campos editables
- 📤 **Subiendo nueva versión**: Barra de progreso (igual que RFC-021)
- 🕒 **Historial**: Timeline visual de versiones
- ⚠️ **Confirmación destructiva**: Modal antes de actualizar PDF

## Flujo de Datos (Técnico)

### Diagrama de Secuencia - Actualizar Metadata

```
Cliente           API Mobile         PostgreSQL
  |                    |                   |
  |--- PUT /materials/:id ---------------->|
  |  + Body: {title, description}          |
  |  + Header: Bearer JWT                  |
  |                    |                   |
  |                    |---SELECT--------->|
  |                    |  materials        |
  |                    |  WHERE id=?       |
  |                    |<--Material--------|
  |                    |                   |
  |                    |---Validate ownership (teacher_id == JWT.user_id)
  |                    |                   |
  |                    |---UPDATE--------->|
  |                    |  SET title=?,     |
  |                    |  description=?,   |
  |                    |  updated_at=NOW() |
  |                    |  WHERE id=?       |
  |                    |<--OK--------------|
  |                    |                   |
  |<--200 OK-----------|                   |
  |  Body: MaterialResponse                |
```

### Diagrama de Secuencia - Actualizar PDF (Crear Versión)

```
Cliente        API Mobile    PostgreSQL    material_versions    S3    RabbitMQ    Worker
  |                |              |               |             |        |          |
  |--- POST /materials/:id/upload-url ---------->|             |        |          |
  |                |              |               |             |        |          |
  |                |---INSERT---->|-------------->|             |        |          |
  |                |  INTO material_versions       |             |        |          |
  |                |  (material_id, version_number, |            |        |          |
  |                |   file_url_old, changed_by)   |             |        |          |
  |                |<--version_id-|---------------|             |        |          |
  |                |              |               |             |        |          |
  |                |---GeneratePresignedURL---------------------->|        |          |
  |                |              |               |             |        |          |
  |<--200 OK-------|              |               |             |        |          |
  |  {upload_url}  |              |               |             |        |          |
  |                |              |               |             |        |          |
  |===== PUT presigned_url (archivo PDF) ======================>|        |          |
  |<====== 200 OK ==============================================|        |          |
  |                |              |               |             |        |          |
  |--- POST /materials/:id/upload-complete ----->|               |        |          |
  |                |              |               |             |        |          |
  |                |---UPDATE---->|               |             |        |          |
  |                |  materials   |               |             |        |          |
  |                |  SET file_url=NEW            |             |        |          |
  |                |  status='processing'         |             |        |          |
  |                |  processing_started_at=NOW() |             |        |          |
  |                |<--OK---------|               |             |        |          |
  |                |              |               |             |        |          |
  |                |---PUBLISH EVENT------------------------------>------->|
  |                |  material.uploaded           |             |        |          |
  |                |              |               |             |        |   (consume)
  |                |              |               |             |        |          |
  |                |              |               |             |        |   (reprocesa)
  |                |              |               |             |        |          |
  |                |              |               |             |        |   MongoDB:
  |                |              |               |             |        |   - Nuevo summary
  |                |              |               |             |        |   - Nuevo assessment
  |                |              |               |             |        |          |
  |                |<--UPDATE (status='ready')--<---------------|<-------|<---------|
```

### Endpoints Involucrados

#### 1. Actualizar Metadata (Solo dev)
```http
PUT /v1/materials/:id
Authorization: Bearer {jwt_token}
Content-Type: application/json

{
  "title": "Álgebra Lineal - Módulo 1 (Actualizado)",
  "description": "Introducción a vectores, matrices y determinantes",
  "subject": "Mathematics",
  "grade": "10",
  "academic_unit_id": "uuid",
  "is_public": false
}
```

**Response 200 OK:** (MaterialResponse completo)

**Response 403 Forbidden:**
```typescript
{
  error: "forbidden",
  message: "Solo el docente que subió el material puede editarlo"
}
```

#### 2. Ver Historial de Versiones
```http
GET /v1/materials/:id/versions
Authorization: Bearer {jwt_token}
```

**Response 200 OK:**
```typescript
{
  material: MaterialResponse,  // Versión actual
  versions: [
    {
      id: "version-uuid-3",
      material_id: "material-uuid",
      version_number: 3,
      file_url: "materials/uuid/file-v3.pdf",
      file_size_bytes: 2048576,
      changed_by: "teacher-uuid",
      changed_by_name: "Prof. García",
      created_at: "2024-12-24T10:30:00Z"
    },
    {
      id: "version-uuid-2",
      material_id: "material-uuid",
      version_number: 2,
      file_url: "materials/uuid/file-v2.pdf",
      file_size_bytes: 1950000,
      changed_by: "teacher-uuid",
      changed_by_name: "Prof. García",
      created_at: "2024-12-20T15:45:00Z"
    },
    {
      id: "version-uuid-1",
      material_id: "material-uuid",
      version_number: 1,
      file_url: "materials/uuid/file-v1.pdf",
      file_size_bytes: 1800000,
      changed_by: "teacher-uuid",
      changed_by_name: "Prof. García",
      created_at: "2024-12-15T09:00:00Z"
    }
  ]
}
```

### Request/Response (TypeScript)

**UpdateMaterialRequest:**
```typescript
interface UpdateMaterialRequest {
  title?: string;          // min=3, max=200
  description?: string;    // max=1000
  subject?: string;
  grade?: string;
  academic_unit_id?: string;
  is_public?: boolean;
}
```

**MaterialVersionResponse:**
```typescript
interface MaterialVersionResponse {
  id: string;              // UUID de la versión
  material_id: string;     // UUID del material padre
  version_number: number;  // 1, 2, 3, ...
  file_url: string;        // S3 path de esta versión
  file_size_bytes: number;
  changed_by: string;      // UUID del teacher
  changed_by_name: string; // Nombre del teacher (join)
  created_at: string;      // ISO8601
}

interface MaterialWithVersionsResponse {
  material: MaterialResponse;           // Versión actual
  versions: MaterialVersionResponse[];  // Historial completo
}
```

## Estados y Transiciones

### Estados Durante Actualización

**Actualizar Metadata (sin PDF):**
```
Material v1 (ready)
  ↓
UPDATE metadata
  ↓
Material v1 (ready, updated_at cambiado)
```
- Sin cambio de versión
- Sin reprocesamiento Worker
- Solo actualiza campos editables

**Actualizar PDF (nueva versión):**
```
Material v1 (ready)
  ↓
INSERT material_versions (v1 archivada)
  ↓
Material v2 (processing)
  ↓ (Worker)
Material v2 (ready)
```
- Nueva versión creada automáticamente
- Versión anterior archivada en `material_versions`
- Worker reprocesa → nuevo resumen + quiz

### Numeración de Versiones
- Autoincremental por material
- Primera versión: 1
- Segunda versión: 2
- ...
- Versión N: N

**Cálculo:**
```sql
-- Backend calcula próximo version_number
SELECT COALESCE(MAX(version_number), 0) + 1
FROM material_versions
WHERE material_id = $1;
```

## Manejo de Errores

### Error 1: Usuario Sin Permisos
```typescript
try {
  await materialsApi.update(materialId, {
    title: 'Nuevo título'
  });
} catch (error) {
  if (error.response?.status === 403) {
    showError(
      'Solo el docente que creó este material puede editarlo. ' +
      'Contacta a ' + material.uploaded_by_teacher_name
    );
  }
}
```

### Error 2: Metadata Inválida
```typescript
try {
  await materialsApi.update(materialId, {
    title: 'AB' // Muy corto (min=3)
  });
} catch (error) {
  if (error.response?.status === 422) {
    const validationErrors = error.response.data.errors;

    validationErrors.forEach((err: ValidationError) => {
      showFieldError(err.field, err.message);
    });
  }
}

// Ejemplo de errores de validación
{
  errors: [
    { field: 'title', message: 'El título debe tener al menos 3 caracteres' },
    { field: 'description', message: 'La descripción supera los 1000 caracteres' }
  ]
}
```

### Error 3: Versión No Encontrada
```typescript
try {
  const { versions } = await materialsApi.getVersions(materialId);

  if (versions.length === 0) {
    showInfo('Este material aún no tiene versiones anteriores');
  }
} catch (error) {
  if (error.response?.status === 404) {
    showError('Material no encontrado');
  }
}
```

### Error 4: Conflicto de Actualización Simultánea
```typescript
// Optimistic locking usando updated_at
try {
  await materialsApi.update(materialId, {
    title: 'Nuevo título'
  }, {
    expectedUpdatedAt: material.updated_at
  });
} catch (error) {
  if (error.response?.status === 409) {
    showWarning(
      'Otra persona ha actualizado este material. ' +
      'Recarga la página para ver los cambios más recientes.'
    );

    // Recargar material
    await queryClient.invalidateQueries(['material', materialId]);
  }
}
```

## Consideraciones de UX

### Confirmación Antes de Actualizar PDF
```typescript
function UpdatePDFButton({ material }: { material: MaterialResponse }) {
  const handleUpdatePDF = async () => {
    // Mostrar confirmación
    const confirmed = await confirm({
      title: '¿Actualizar archivo PDF?',
      message:
        'Se creará una nueva versión del material. ' +
        'La versión actual se archivará y los estudiantes verán la nueva versión. ' +
        'El material será reprocesado con IA (~2-5 minutos).',
      confirmText: 'Sí, actualizar',
      cancelText: 'Cancelar',
      variant: 'warning'
    });

    if (!confirmed) return;

    // Proceder con actualización
    navigate(`/materials/${material.id}/update-pdf`);
  };

  return (
    <Button onClick={handleUpdatePDF} variant="secondary">
      <UploadIcon /> Actualizar PDF
    </Button>
  );
}
```

### Timeline de Versiones
```typescript
import { Timeline, TimelineItem } from '@/components/ui';
import { formatDistanceToNow } from 'date-fns';
import { es } from 'date-fns/locale';

function VersionHistory({ versions }: { versions: MaterialVersionResponse[] }) {
  return (
    <Timeline>
      {versions.map((version, index) => {
        const isCurrent = index === 0;
        const timeAgo = formatDistanceToNow(
          new Date(version.created_at),
          { addSuffix: true, locale: es }
        );

        return (
          <TimelineItem
            key={version.id}
            icon={isCurrent ? <CheckIcon /> : <ClockIcon />}
            variant={isCurrent ? 'success' : 'default'}
          >
            <div className="flex justify-between items-start">
              <div>
                <h4 className="font-semibold">
                  Versión {version.version_number}
                  {isCurrent && (
                    <Badge variant="success" className="ml-2">Actual</Badge>
                  )}
                </h4>

                <p className="text-sm text-gray-600">
                  Por {version.changed_by_name} • {timeAgo}
                </p>

                <p className="text-xs text-gray-500">
                  Tamaño: {formatBytes(version.file_size_bytes)}
                </p>
              </div>

              <div className="flex gap-2">
                <Button
                  size="sm"
                  variant="ghost"
                  onClick={() => downloadVersion(version.id)}
                >
                  <DownloadIcon />
                </Button>

                {!isCurrent && (
                  <Button
                    size="sm"
                    variant="ghost"
                    onClick={() => restoreVersion(version.id)}
                  >
                    <RestoreIcon />
                  </Button>
                )}
              </div>
            </div>
          </TimelineItem>
        );
      })}
    </Timeline>
  );
}
```

### Diff Visual de Metadata (Futuro)
```typescript
// Mostrar cambios entre versiones
function MetadataChanges({ oldVersion, newVersion }) {
  const changes: Change[] = [];

  if (oldVersion.title !== newVersion.title) {
    changes.push({
      field: 'Título',
      old: oldVersion.title,
      new: newVersion.title
    });
  }

  if (oldVersion.description !== newVersion.description) {
    changes.push({
      field: 'Descripción',
      old: oldVersion.description,
      new: newVersion.description
    });
  }

  return (
    <div className="border rounded p-4">
      <h3 className="font-semibold mb-2">Cambios realizados:</h3>

      {changes.map(change => (
        <div key={change.field} className="mb-2">
          <span className="font-medium">{change.field}:</span>
          <div className="flex gap-2 items-center mt-1">
            <span className="line-through text-gray-400">{change.old}</span>
            <ArrowRightIcon className="text-gray-400" />
            <span className="text-green-600">{change.new}</span>
          </div>
        </div>
      ))}
    </div>
  );
}
```

### Badge de "Actualizado Recientemente"
```typescript
function MaterialCard({ material }: { material: MaterialResponse }) {
  // Mostrar badge si fue actualizado en las últimas 48 horas
  const isRecentlyUpdated = Date.now() - new Date(material.updated_at).getTime() < 48 * 60 * 60 * 1000;

  return (
    <Card>
      <CardHeader>
        <div className="flex justify-between items-start">
          <h3>{material.title}</h3>

          {isRecentlyUpdated && (
            <Badge variant="info">
              <SparklesIcon /> Actualizado
            </Badge>
          )}
        </div>
      </CardHeader>

      {/* ... resto del card */}
    </Card>
  );
}
```

## Almacenamiento Local

### Cache de Versiones
```typescript
// Cachear historial de versiones
interface CachedVersionHistory {
  materialId: string;
  versions: MaterialVersionResponse[];
  cachedAt: number;
}

async function cacheVersionHistory(
  materialId: string,
  versions: MaterialVersionResponse[]
) {
  await db.versionHistories.put({
    materialId,
    versions,
    cachedAt: Date.now()
  });
}

async function getCachedVersionHistory(
  materialId: string
): Promise<MaterialVersionResponse[] | null> {
  const cached = await db.versionHistories.get(materialId);

  if (!cached) return null;

  // Cache válido por 1 hora
  if (Date.now() - cached.cachedAt > 60 * 60 * 1000) {
    await db.versionHistories.delete(materialId);
    return null;
  }

  return cached.versions;
}
```

## Código de Ejemplo (Mobile - TypeScript)

### Hook de Actualización
```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { materialsApi } from '@/services/api';

export function useUpdateMaterial(materialId: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (updates: UpdateMaterialRequest) =>
      materialsApi.update(materialId, updates),

    onSuccess: (updatedMaterial) => {
      // Actualizar cache del material
      queryClient.setQueryData(
        ['material', materialId],
        updatedMaterial
      );

      // Invalidar lista de materiales
      queryClient.invalidateQueries({ queryKey: ['materials'] });

      showSuccess('Material actualizado exitosamente');
    },

    onError: (error: any) => {
      if (error.response?.status === 403) {
        showError('No tienes permisos para editar este material');
      } else if (error.response?.status === 422) {
        showError('Verifica los campos del formulario');
      } else {
        showError('Error al actualizar material');
      }
    }
  });
}

// Uso
function EditMaterialForm({ material }: { material: MaterialResponse }) {
  const { mutate: updateMaterial, isPending } = useUpdateMaterial(material.id);

  const handleSubmit = (e: FormEvent) => {
    e.preventDefault();

    const formData = new FormData(e.target as HTMLFormElement);

    updateMaterial({
      title: formData.get('title') as string,
      description: formData.get('description') as string,
      subject: formData.get('subject') as string,
      grade: formData.get('grade') as string
    });
  };

  return (
    <form onSubmit={handleSubmit}>
      <Input
        name="title"
        label="Título"
        defaultValue={material.title}
        required
        minLength={3}
        maxLength={200}
      />

      <Textarea
        name="description"
        label="Descripción"
        defaultValue={material.description}
        maxLength={1000}
      />

      <Select
        name="subject"
        label="Materia"
        defaultValue={material.subject}
      />

      <Select
        name="grade"
        label="Grado"
        defaultValue={material.grade}
      />

      <Button type="submit" disabled={isPending}>
        {isPending ? 'Guardando...' : 'Guardar cambios'}
      </Button>
    </form>
  );
}
```

### Hook de Historial
```typescript
export function useMaterialVersions(materialId: string) {
  return useQuery({
    queryKey: ['material-versions', materialId],
    queryFn: () => materialsApi.getVersions(materialId),
    staleTime: 5 * 60 * 1000 // 5 min
  });
}

// Uso
function MaterialVersionsPage({ materialId }: { materialId: string }) {
  const { data, isLoading, error } = useMaterialVersions(materialId);

  if (isLoading) return <Spinner />;
  if (error) return <ErrorState />;
  if (!data) return null;

  return (
    <div>
      <h2>Historial de Versiones</h2>

      <MaterialMetadata material={data.material} />

      <VersionHistory versions={data.versions} />
    </div>
  );
}
```

### Service API
```typescript
// services/api/materials.ts
export const materialsApi = {
  async update(
    materialId: string,
    updates: UpdateMaterialRequest
  ): Promise<MaterialResponse> {
    const response = await api.put(`/v1/materials/${materialId}`, updates);
    return response.data;
  },

  async getVersions(materialId: string): Promise<MaterialWithVersionsResponse> {
    const response = await api.get(`/v1/materials/${materialId}/versions`);
    return response.data;
  }
};
```

## Notas de Implementación

### Backend: Tabla material_versions
```sql
CREATE TABLE material_versions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  material_id UUID NOT NULL REFERENCES materials(id) ON DELETE CASCADE,
  version_number INTEGER NOT NULL,
  file_url TEXT NOT NULL,
  file_size_bytes BIGINT,
  changed_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,

  UNIQUE(material_id, version_number)
);

CREATE INDEX idx_material_versions_material_id ON material_versions(material_id);
CREATE INDEX idx_material_versions_created_at ON material_versions(created_at DESC);
```

### Trigger para Crear Versión Automáticamente
```sql
-- Trigger que archiva versión anterior al actualizar file_url
CREATE OR REPLACE FUNCTION archive_material_version()
RETURNS TRIGGER AS $$
BEGIN
  -- Solo si file_url cambió
  IF OLD.file_url IS DISTINCT FROM NEW.file_url AND NEW.file_url IS NOT NULL THEN
    -- Calcular próximo version_number
    DECLARE
      next_version INTEGER;
    BEGIN
      SELECT COALESCE(MAX(version_number), 0) + 1
      INTO next_version
      FROM material_versions
      WHERE material_id = NEW.id;

      -- Insertar versión anterior
      INSERT INTO material_versions (
        material_id,
        version_number,
        file_url,
        file_size_bytes,
        changed_by
      ) VALUES (
        NEW.id,
        next_version,
        OLD.file_url,
        OLD.file_size_bytes,
        NEW.uploaded_by_teacher_id
      );
    END;
  END IF;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER material_version_trigger
BEFORE UPDATE ON materials
FOR EACH ROW
EXECUTE FUNCTION archive_material_version();
```

### Limitaciones Actuales
- ⚠️ **PUT /materials/:id SOLO en rama dev** (no en main)
- ⚠️ Sin límite de versiones (crece indefinidamente)
- ⚠️ Sin restauración de versiones anteriores (solo vista)
- ⚠️ Sin diff de contenido PDF (solo metadata)

### Mejoras Futuras
1. **Límite de Versiones**: Eliminar versiones >10 automáticamente
2. **Restaurar Versión**: Endpoint POST /materials/:id/restore/:version_number
3. **Diff PDF**: Comparar contenido entre versiones
4. **Rollback**: Revertir a versión anterior con un click
5. **Branches**: Sistema de branches como Git (experimental)

### Seguridad
- ✅ Solo owner puede actualizar metadata
- ✅ Solo owner puede subir nueva versión
- ⚠️ Cualquier usuario autenticado puede ver historial
- Recomendación: Restringir acceso a historial solo a teachers

### Testing
```typescript
describe('Material Versioning', () => {
  it('actualiza metadata sin crear versión', async () => {
    const material = await createTestMaterial();

    const updated = await materialsApi.update(material.id, {
      title: 'Título actualizado'
    });

    expect(updated.title).toBe('Título actualizado');
    expect(updated.updated_at).not.toBe(material.updated_at);

    // No debe haber creado versión
    const { versions } = await materialsApi.getVersions(material.id);
    expect(versions.length).toBe(0);
  });

  it('crea versión al actualizar PDF', async () => {
    const material = await createTestMaterial({ withPDF: true });

    // Actualizar PDF (simulado)
    await materialsApi.updatePDF(material.id, newPDFFile);

    // Debe haber creado versión
    const { versions } = await materialsApi.getVersions(material.id);
    expect(versions.length).toBe(1);
    expect(versions[0].version_number).toBe(1);
    expect(versions[0].file_url).toBe(material.file_url);
  });

  it('retorna versiones en orden descendente', async () => {
    const material = await createTestMaterial();

    // Crear 3 versiones
    await materialsApi.updatePDF(material.id, pdfFile1);
    await materialsApi.updatePDF(material.id, pdfFile2);
    await materialsApi.updatePDF(material.id, pdfFile3);

    const { versions } = await materialsApi.getVersions(material.id);

    expect(versions).toHaveLength(3);
    expect(versions[0].version_number).toBe(3); // Más reciente primero
    expect(versions[1].version_number).toBe(2);
    expect(versions[2].version_number).toBe(1); // Más antigua última
  });
});
```

---

**Última actualización:** 2025-12-24
**Versión:** 1.0
**Estado:** ⚠️ Parcial (requiere merge dev→main)
