# RFC-014: Gestión de Materias (Subjects)

## Metadata
- **ID:** RFC-014
- **Proceso:** Gestión Escolar
- **Subproceso:** Administración de Materias
- **Prioridad:** Media
- **Dependencias:** RFC-001 (Autenticación), RFC-010 (CRUD Escuelas)
- **Estado API:** ⚠️ Parcial (requiere validación de endpoints)

## Descripción

Permite a los administradores de escuela gestionar el catálogo de materias (subjects) disponibles para la institución. Las materias se utilizan para categorizar materiales educativos y pueden estar asociadas a grados específicos o ser transversales.

## Flujo de Usuario (UX)

### Pantalla: Lista de Materias

```
┌─────────────────────────────────────────┐
│  ← Configuración                        │
│                                          │
│  📚 Materias de la Escuela              │
│                                          │
│  🔍 Buscar materia...                   │
│                                          │
│  ┌────────────────────────────────────┐ │
│  │ 📐 Matemáticas                      │ │
│  │    Grados: 1-12 | Materiales: 45   │ │
│  │    Activa ✅                        │ │
│  ├────────────────────────────────────┤ │
│  │ 📜 Historia                         │ │
│  │    Grados: 6-12 | Materiales: 32   │ │
│  │    Activa ✅                        │ │
│  ├────────────────────────────────────┤ │
│  │ 🔬 Ciencias                         │ │
│  │    Grados: 1-12 | Materiales: 28   │ │
│  │    Activa ✅                        │ │
│  ├────────────────────────────────────┤ │
│  │ 🎨 Arte                             │ │
│  │    Grados: 1-9 | Materiales: 15    │ │
│  │    Activa ✅                        │ │
│  └────────────────────────────────────┘ │
│                                          │
│        [+ Nueva Materia]                │
│                                          │
└─────────────────────────────────────────┘
```

### Flujo: Crear Nueva Materia

1. Admin accede a "Gestión de Materias"
2. Click en "+ Nueva Materia"
3. Formulario muestra campos:
   - Nombre (requerido)
   - Código único (requerido)
   - Descripción (opcional)
   - Icono/Emoji (opcional)
   - Grados aplicables (multi-select)
   - Estado activo/inactivo
4. Sistema valida que código no exista
5. Materia creada → Aparece en lista
6. Puede asignarse a materiales

### Flujo: Editar Materia

1. Admin selecciona materia de la lista
2. Click en "Editar"
3. Modifica campos necesarios
4. Sistema valida cambios
5. Actualiza → Muestra confirmación

### Flujo: Desactivar Materia

1. Admin selecciona materia
2. Toggle de estado o click en "Desactivar"
3. Confirmación: "¿Desactivar materia? Los materiales existentes no se verán afectados."
4. Confirma → Materia marcada como inactiva
5. No aparece en selector de nuevos materiales

## Flujo de Datos (Técnico)

### Endpoints Propuestos

| Endpoint | Método | Descripción | Estado |
|----------|--------|-------------|--------|
| `/v1/schools/:school_id/subjects` | GET | Listar materias de escuela | ⚠️ Por validar |
| `/v1/schools/:school_id/subjects` | POST | Crear nueva materia | ⚠️ Por validar |
| `/v1/schools/:school_id/subjects/:id` | GET | Obtener materia | ⚠️ Por validar |
| `/v1/schools/:school_id/subjects/:id` | PUT | Actualizar materia | ⚠️ Por validar |
| `/v1/schools/:school_id/subjects/:id` | DELETE | Eliminar/desactivar | ⚠️ Por validar |

### Request/Response (TypeScript)

**Crear Materia - Request:**
```typescript
interface CreateSubjectRequest {
  name: string;                    // min=2, required
  code: string;                    // min=2, required, unique per school
  description?: string;
  icon?: string;                   // emoji o código de icono
  applicable_grades?: string[];    // ["1", "2", "3"] o ["all"]
  is_active?: boolean;             // default: true
  metadata?: Record<string, any>;  // datos adicionales
}

// Ejemplo
{
  "name": "Matemáticas",
  "code": "MAT",
  "description": "Área de ciencias exactas",
  "icon": "📐",
  "applicable_grades": ["1", "2", "3", "4", "5", "6", "7", "8", "9", "10", "11", "12"],
  "is_active": true
}
```

**Subject Response:**
```typescript
interface SubjectResponse {
  id: string;                      // UUID
  school_id: string;               // UUID
  name: string;
  code: string;
  description: string;
  icon: string;
  applicable_grades: string[];
  is_active: boolean;
  materials_count: number;         // Número de materiales usando esta materia
  metadata: Record<string, any>;
  created_at: string;              // ISO8601
  updated_at: string;              // ISO8601
}

// Lista response
type SubjectListResponse = SubjectResponse[];
```

**Ejemplo Response:**
```json
{
  "id": "subj-123",
  "school_id": "school-456",
  "name": "Matemáticas",
  "code": "MAT",
  "description": "Área de ciencias exactas incluyendo álgebra, geometría y cálculo",
  "icon": "📐",
  "applicable_grades": ["1", "2", "3", "4", "5", "6", "7", "8", "9", "10", "11", "12"],
  "is_active": true,
  "materials_count": 45,
  "metadata": {},
  "created_at": "2024-01-15T10:00:00Z",
  "updated_at": "2024-12-20T15:30:00Z"
}
```

**Update Request:**
```typescript
interface UpdateSubjectRequest {
  name?: string;
  code?: string;
  description?: string;
  icon?: string;
  applicable_grades?: string[];
  is_active?: boolean;
  metadata?: Record<string, any>;
}
```

### Validaciones

```typescript
const subjectValidations = {
  name: {
    required: true,
    minLength: 2,
    maxLength: 100
  },
  code: {
    required: true,
    minLength: 2,
    maxLength: 20,
    pattern: /^[A-Z0-9_-]+$/,  // Solo mayúsculas, números, guiones
    unique: true               // Único por escuela
  },
  applicable_grades: {
    validValues: ['1', '2', '3', '4', '5', '6', '7', '8', '9', '10', '11', '12', 'all']
  }
};
```

## Estados y Transiciones

```
┌─────────┐
│ CREANDO │
└────┬────┘
     │
     ▼
┌─────────┐     TOGGLE          ┌──────────┐
│ ACTIVA  │◄───────────────────►│ INACTIVA │
└────┬────┘                     └──────────┘
     │
     │ DELETE (solo si materials_count = 0)
     ▼
┌──────────┐
│ELIMINADA │
└──────────┘

Estados:
- ACTIVA: is_active = true, aparece en selectores
- INACTIVA: is_active = false, no aparece en selectores
- ELIMINADA: Solo posible si no hay materiales asociados
```

## Manejo de Errores

| Código | Significado | Mensaje UI |
|--------|-------------|------------|
| 400 | Validación fallida | Mostrar errores por campo |
| 401 | No autenticado | Redirigir a login |
| 403 | Sin permisos | "No tienes permisos para gestionar materias" |
| 404 | Materia no encontrada | "Materia no encontrada" |
| 409 | Código duplicado | "Ya existe una materia con este código" |
| 422 | No se puede eliminar | "No se puede eliminar: hay materiales asociados" |
| 500 | Error servidor | "Error del sistema, intenta más tarde" |

### Manejo de Conflicto al Eliminar

```typescript
if (error.response?.status === 422 && error.code === 'HAS_MATERIALS') {
  showModal({
    title: 'No se puede eliminar',
    message: `Esta materia tiene ${subject.materials_count} materiales asociados. Desactívala en lugar de eliminarla.`,
    actions: [
      { text: 'Cancelar', style: 'cancel' },
      { text: 'Desactivar', onPress: () => deactivateSubject(subject.id) }
    ]
  });
}
```

## Consideraciones de UX

### Loading State

```typescript
function SubjectsListSkeleton() {
  return (
    <div className="space-y-3 p-4">
      {[1, 2, 3, 4, 5].map(i => (
        <div key={i} className="animate-pulse flex items-center gap-4 p-4 bg-gray-100 rounded-lg">
          <div className="w-10 h-10 bg-gray-200 rounded-full" />
          <div className="flex-1">
            <div className="h-5 bg-gray-200 rounded w-32 mb-2" />
            <div className="h-4 bg-gray-200 rounded w-48" />
          </div>
        </div>
      ))}
    </div>
  );
}
```

### Búsqueda y Filtros

```typescript
function SubjectFilters({
  onSearch,
  onFilterActive
}: SubjectFiltersProps) {
  return (
    <div className="flex gap-4 p-4">
      <input
        type="text"
        placeholder="🔍 Buscar materia..."
        onChange={(e) => onSearch(e.target.value)}
        className="flex-1 px-4 py-2 border rounded-lg"
      />
      <select
        onChange={(e) => onFilterActive(e.target.value)}
        className="px-4 py-2 border rounded-lg"
      >
        <option value="all">Todas</option>
        <option value="active">Activas</option>
        <option value="inactive">Inactivas</option>
      </select>
    </div>
  );
}
```

### Formulario de Materia

```typescript
function SubjectForm({
  subject,
  onSubmit,
  onCancel
}: SubjectFormProps) {
  const { register, handleSubmit, formState: { errors, isSubmitting } } = useForm({
    defaultValues: subject || {
      is_active: true,
      applicable_grades: []
    }
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4 p-4">
      {/* Nombre */}
      <div>
        <label className="block text-sm font-medium mb-1">Nombre *</label>
        <input
          {...register('name', {
            required: 'El nombre es obligatorio',
            minLength: { value: 2, message: 'Mínimo 2 caracteres' }
          })}
          className={`w-full px-4 py-2 border rounded-lg ${errors.name ? 'border-red-500' : ''}`}
          placeholder="Ej: Matemáticas"
        />
        {errors.name && <span className="text-red-500 text-sm">{errors.name.message}</span>}
      </div>

      {/* Código */}
      <div>
        <label className="block text-sm font-medium mb-1">Código *</label>
        <input
          {...register('code', {
            required: 'El código es obligatorio',
            pattern: {
              value: /^[A-Z0-9_-]+$/,
              message: 'Solo mayúsculas, números y guiones'
            }
          })}
          className={`w-full px-4 py-2 border rounded-lg ${errors.code ? 'border-red-500' : ''}`}
          placeholder="Ej: MAT"
        />
        {errors.code && <span className="text-red-500 text-sm">{errors.code.message}</span>}
      </div>

      {/* Descripción */}
      <div>
        <label className="block text-sm font-medium mb-1">Descripción</label>
        <textarea
          {...register('description')}
          className="w-full px-4 py-2 border rounded-lg"
          rows={3}
          placeholder="Descripción opcional de la materia..."
        />
      </div>

      {/* Icono */}
      <div>
        <label className="block text-sm font-medium mb-1">Icono</label>
        <div className="flex gap-2">
          {['📐', '📜', '🔬', '🎨', '🌍', '💻', '📖', '🎵'].map(emoji => (
            <button
              key={emoji}
              type="button"
              onClick={() => setValue('icon', emoji)}
              className={`text-2xl p-2 rounded ${watch('icon') === emoji ? 'bg-blue-100 ring-2 ring-blue-500' : ''}`}
            >
              {emoji}
            </button>
          ))}
        </div>
      </div>

      {/* Grados */}
      <div>
        <label className="block text-sm font-medium mb-1">Grados Aplicables</label>
        <div className="flex flex-wrap gap-2">
          {['1', '2', '3', '4', '5', '6', '7', '8', '9', '10', '11', '12'].map(grade => (
            <label key={grade} className="inline-flex items-center">
              <input
                type="checkbox"
                value={grade}
                {...register('applicable_grades')}
                className="mr-1"
              />
              <span>{grade}°</span>
            </label>
          ))}
        </div>
      </div>

      {/* Estado */}
      <div className="flex items-center gap-2">
        <input type="checkbox" {...register('is_active')} />
        <label>Materia activa</label>
      </div>

      {/* Acciones */}
      <div className="flex gap-4 pt-4">
        <button
          type="button"
          onClick={onCancel}
          className="flex-1 py-2 border rounded-lg"
          disabled={isSubmitting}
        >
          Cancelar
        </button>
        <button
          type="submit"
          className="flex-1 py-2 bg-blue-500 text-white rounded-lg"
          disabled={isSubmitting}
        >
          {isSubmitting ? 'Guardando...' : subject ? 'Actualizar' : 'Crear'}
        </button>
      </div>
    </form>
  );
}
```

## Almacenamiento Local

### Cache de Materias

```typescript
interface SubjectsCache {
  schoolId: string;
  subjects: SubjectResponse[];
  cachedAt: number;
}

// Cache de 10 minutos (datos cambian poco)
const SUBJECTS_TTL = 10 * 60 * 1000;

function cacheSubjects(schoolId: string, subjects: SubjectResponse[]) {
  const cache: SubjectsCache = {
    schoolId,
    subjects,
    cachedAt: Date.now()
  };

  localStorage.setItem(`subjects_${schoolId}`, JSON.stringify(cache));
}

function getCachedSubjects(schoolId: string): SubjectResponse[] | null {
  const cached = localStorage.getItem(`subjects_${schoolId}`);
  if (!cached) return null;

  const cache: SubjectsCache = JSON.parse(cached);

  if (Date.now() - cache.cachedAt > SUBJECTS_TTL) {
    return null; // Cache expirado
  }

  return cache.subjects;
}

// Invalidar al crear/editar/eliminar
function invalidateSubjectsCache(schoolId: string) {
  localStorage.removeItem(`subjects_${schoolId}`);
}
```

## Código de Ejemplo (Mobile - TypeScript)

### Service

```typescript
// services/SubjectService.ts
export class SubjectService {
  private api: AxiosInstance;

  constructor(baseURL: string) {
    this.api = axios.create({ baseURL, timeout: 10000 });
    this.setupInterceptors();
  }

  async listSubjects(schoolId: string): Promise<SubjectResponse[]> {
    // Verificar cache
    const cached = getCachedSubjects(schoolId);
    if (cached) return cached;

    const response = await this.api.get(`/v1/schools/${schoolId}/subjects`);
    cacheSubjects(schoolId, response.data);
    return response.data;
  }

  async createSubject(schoolId: string, data: CreateSubjectRequest): Promise<SubjectResponse> {
    const response = await this.api.post(`/v1/schools/${schoolId}/subjects`, data);
    invalidateSubjectsCache(schoolId);
    return response.data;
  }

  async updateSubject(schoolId: string, subjectId: string, data: UpdateSubjectRequest): Promise<SubjectResponse> {
    const response = await this.api.put(`/v1/schools/${schoolId}/subjects/${subjectId}`, data);
    invalidateSubjectsCache(schoolId);
    return response.data;
  }

  async deleteSubject(schoolId: string, subjectId: string): Promise<void> {
    await this.api.delete(`/v1/schools/${schoolId}/subjects/${subjectId}`);
    invalidateSubjectsCache(schoolId);
  }

  async toggleActive(schoolId: string, subjectId: string, isActive: boolean): Promise<SubjectResponse> {
    return this.updateSubject(schoolId, subjectId, { is_active: isActive });
  }
}
```

### Hook

```typescript
// hooks/useSubjects.ts
export function useSubjects(schoolId: string) {
  const [subjects, setSubjects] = useState<SubjectResponse[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  const service = new SubjectService(process.env.REACT_APP_API_ADMIN_URL!);

  const fetchSubjects = useCallback(async () => {
    try {
      setLoading(true);
      const data = await service.listSubjects(schoolId);
      setSubjects(data);
      setError(null);
    } catch (err) {
      setError(err as Error);
    } finally {
      setLoading(false);
    }
  }, [schoolId]);

  const createSubject = async (data: CreateSubjectRequest) => {
    const newSubject = await service.createSubject(schoolId, data);
    setSubjects(prev => [...prev, newSubject]);
    return newSubject;
  };

  const updateSubject = async (subjectId: string, data: UpdateSubjectRequest) => {
    const updated = await service.updateSubject(schoolId, subjectId, data);
    setSubjects(prev => prev.map(s => s.id === subjectId ? updated : s));
    return updated;
  };

  const deleteSubject = async (subjectId: string) => {
    await service.deleteSubject(schoolId, subjectId);
    setSubjects(prev => prev.filter(s => s.id !== subjectId));
  };

  const toggleActive = async (subjectId: string, isActive: boolean) => {
    return updateSubject(subjectId, { is_active: isActive });
  };

  useEffect(() => {
    fetchSubjects();
  }, [fetchSubjects]);

  return {
    subjects,
    loading,
    error,
    createSubject,
    updateSubject,
    deleteSubject,
    toggleActive,
    refresh: fetchSubjects,
    activeSubjects: subjects.filter(s => s.is_active)
  };
}
```

### Selector de Materias (para otros formularios)

```typescript
// components/SubjectSelector.tsx
interface SubjectSelectorProps {
  schoolId: string;
  value?: string;
  onChange: (subjectId: string) => void;
  placeholder?: string;
}

export function SubjectSelector({
  schoolId,
  value,
  onChange,
  placeholder = "Seleccionar materia"
}: SubjectSelectorProps) {
  const { activeSubjects, loading } = useSubjects(schoolId);

  if (loading) {
    return (
      <select disabled className="w-full px-4 py-2 border rounded-lg bg-gray-100">
        <option>Cargando materias...</option>
      </select>
    );
  }

  return (
    <select
      value={value || ''}
      onChange={(e) => onChange(e.target.value)}
      className="w-full px-4 py-2 border rounded-lg"
    >
      <option value="">{placeholder}</option>
      {activeSubjects.map(subject => (
        <option key={subject.id} value={subject.id}>
          {subject.icon} {subject.name}
        </option>
      ))}
    </select>
  );
}
```

## Notas de Implementación

### Backend Requerido

1. **Modelo Subject:**
   ```sql
   CREATE TABLE subjects (
     id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
     school_id UUID NOT NULL REFERENCES schools(id),
     name VARCHAR(100) NOT NULL,
     code VARCHAR(20) NOT NULL,
     description TEXT,
     icon VARCHAR(10),
     applicable_grades TEXT[], -- Array de strings
     is_active BOOLEAN DEFAULT true,
     metadata JSONB DEFAULT '{}',
     created_at TIMESTAMP DEFAULT NOW(),
     updated_at TIMESTAMP DEFAULT NOW(),
     deleted_at TIMESTAMP,
     UNIQUE(school_id, code)
   );
   ```

2. **Índices:**
   ```sql
   CREATE INDEX idx_subjects_school ON subjects(school_id);
   CREATE INDEX idx_subjects_active ON subjects(school_id, is_active);
   ```

### Alternativa: Usar Campo Subject del Material

Si no hay endpoint de subjects, el frontend puede:
1. Extraer subjects únicos de los materiales existentes
2. Mantener lista hardcoded de subjects comunes
3. Permitir texto libre en campo subject

```typescript
// Extraer subjects de materiales existentes
function extractSubjectsFromMaterials(materials: MaterialResponse[]): string[] {
  const subjects = new Set(materials.map(m => m.subject));
  return Array.from(subjects).sort();
}

// Subjects por defecto
const DEFAULT_SUBJECTS = [
  { code: 'MAT', name: 'Matemáticas', icon: '📐' },
  { code: 'HIS', name: 'Historia', icon: '📜' },
  { code: 'SCI', name: 'Ciencias', icon: '🔬' },
  { code: 'LIT', name: 'Literatura', icon: '📖' },
  { code: 'ART', name: 'Arte', icon: '🎨' },
  { code: 'MUS', name: 'Música', icon: '🎵' },
  { code: 'GEO', name: 'Geografía', icon: '🌍' },
  { code: 'CS', name: 'Informática', icon: '💻' }
];
```

### Testing

```typescript
describe('SubjectService', () => {
  it('should create subject with valid data', async () => {
    const data = { name: 'Test Subject', code: 'TEST' };
    const result = await service.createSubject('school-123', data);
    expect(result.name).toBe('Test Subject');
    expect(result.is_active).toBe(true);
  });

  it('should reject duplicate code', async () => {
    await expect(
      service.createSubject('school-123', { name: 'Test', code: 'EXISTING' })
    ).rejects.toMatchObject({ response: { status: 409 } });
  });

  it('should prevent deletion with materials', async () => {
    await expect(
      service.deleteSubject('school-123', 'subject-with-materials')
    ).rejects.toMatchObject({ response: { status: 422 } });
  });
});
```

---

**Última actualización:** 2025-12-24
**Versión:** 1.0
**Estado:** ⚠️ Requiere validación de endpoints backend
