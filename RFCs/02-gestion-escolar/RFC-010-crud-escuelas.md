# RFC-010: CRUD de Escuelas

## Metadata
- **ID:** RFC-010
- **Proceso:** Gestión Escolar
- **Subproceso:** Administración de Escuelas
- **Prioridad:** Alta
- **Dependencias:** RFC-001 (Autenticación)
- **Estado API:** ✅ Listo en main

## Descripción

Permite a los administradores crear, listar, consultar, actualizar y eliminar escuelas en el sistema. Cada escuela representa una institución educativa con su configuración específica de límites de usuarios, tier de subscripción y datos de contacto.

## Flujo de Usuario (UX)

### Crear Escuela (Admin)
1. Admin accede a "Gestión de Escuelas"
2. Click en "Nueva Escuela"
3. Formulario muestra campos:
   - Nombre (requerido)
   - Código único (requerido)
   - Dirección
   - Ciudad
   - País (default: Colombia)
   - Email de contacto
   - Teléfono de contacto
   - Tier de subscripción (free/basic/premium)
   - Límite de profesores (default: 50)
   - Límite de estudiantes (default: 500)
4. Sistema valida que código no exista
5. Escuela creada → Muestra confirmación con ID generado
6. Redirige a vista de detalle

### Listar Escuelas
1. Admin accede a "Gestión de Escuelas"
2. Sistema carga lista completa de escuelas activas
3. Muestra tarjetas/tabla con: nombre, código, ciudad, tier, activa/inactiva
4. Filtros disponibles: búsqueda por nombre, código, ciudad, tier
5. Click en escuela → Vista de detalle

### Ver Detalle
1. Admin selecciona una escuela
2. Muestra toda la información de la escuela
3. Opciones disponibles: Editar, Eliminar, Ver Unidades Académicas

### Editar Escuela
1. Admin en vista de detalle
2. Click "Editar"
3. Formulario pre-llenado con datos actuales
4. Modifica campos necesarios
5. Sistema valida cambios
6. Actualiza → Muestra confirmación

### Eliminar Escuela (Soft Delete)
1. Admin en vista de detalle
2. Click "Eliminar"
3. Confirmación modal: "¿Seguro? Esto marcará la escuela como inactiva"
4. Confirma → Escuela marcada deleted_at
5. Desaparece de listado principal

### Buscar por Código
1. Admin ingresa código en buscador
2. Sistema busca escuela específica
3. Si existe → Muestra detalle
4. Si no existe → "No se encontró escuela con ese código"

## Flujo de Datos (Técnico)

### Endpoints Involucrados

| Endpoint | Método | Descripción | Estado |
|----------|--------|-------------|--------|
| `/v1/schools` | POST | Crear nueva escuela | ✅ |
| `/v1/schools` | GET | Listar todas las escuelas | ⚠️ Sin paginación |
| `/v1/schools/:id` | GET | Obtener escuela por UUID | ✅ |
| `/v1/schools/code/:code` | GET | Buscar por código único | ✅ |
| `/v1/schools/:id` | PUT | Actualizar escuela completa | ✅ |
| `/v1/schools/:id` | DELETE | Soft delete de escuela | ✅ |

### Request/Response (TypeScript interfaces)

```typescript
// ============= CREATE SCHOOL =============
interface CreateSchoolRequest {
  name: string;                              // min=3, required
  code: string;                              // min=3, required, unique
  address?: string;
  city?: string;
  country?: string;                          // default: "CO"
  contact_email?: string;                    // email format
  contact_phone?: string;
  subscription_tier?: "free" | "basic" | "premium"; // default: "free"
  max_teachers?: number;                     // default: 50
  max_students?: number;                     // default: 500
  metadata?: Record<string, any>;            // JSON libre
}

// Ejemplo
const createRequest: CreateSchoolRequest = {
  name: "Colegio San José",
  code: "CSJ001",
  address: "Calle 123 #45-67",
  city: "Bogotá",
  country: "CO",
  contact_email: "admin@colegiosanjose.edu.co",
  contact_phone: "+57 1 234 5678",
  subscription_tier: "basic",
  max_teachers: 100,
  max_students: 1000,
  metadata: {
    foundation_year: 1950,
    principal: "María García"
  }
};

// ============= SCHOOL RESPONSE =============
interface SchoolResponse {
  id: string;                                // UUID
  name: string;
  code: string;
  address: string;
  city: string;
  country: string;
  contact_email: string;
  contact_phone: string;
  subscription_tier: "free" | "basic" | "premium";
  max_teachers: number;
  max_students: number;
  is_active: boolean;
  metadata: Record<string, any>;
  created_at: string;                        // ISO8601
  updated_at: string;                        // ISO8601
}

// ============= UPDATE SCHOOL =============
interface UpdateSchoolRequest {
  name?: string;                             // min=3 si provisto
  code?: string;                             // min=3, unique si provisto
  address?: string;
  city?: string;
  country?: string;
  contact_email?: string;
  contact_phone?: string;
  subscription_tier?: "free" | "basic" | "premium";
  max_teachers?: number;
  max_students?: number;
  metadata?: Record<string, any>;
}

// ============= LIST RESPONSE =============
type SchoolListResponse = SchoolResponse[];
```

### Validaciones del Backend

```typescript
// Validaciones automáticas
const validations = {
  name: {
    required: true,
    minLength: 3,
    maxLength: 255
  },
  code: {
    required: true,
    minLength: 3,
    maxLength: 50,
    unique: true,
    pattern: /^[A-Z0-9_-]+$/  // Solo mayúsculas, números, guiones
  },
  contact_email: {
    format: "email",
    optional: true
  },
  subscription_tier: {
    enum: ["free", "basic", "premium"],
    default: "free"
  },
  max_teachers: {
    min: 1,
    max: 10000,
    default: 50
  },
  max_students: {
    min: 1,
    max: 100000,
    default: 500
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
┌─────────┐     UPDATE      ┌────────────┐
│ ACTIVA  ├────────────────►│  ACTIVA    │
│(normal) │                 │ (modificada)│
└────┬────┘                 └──────┬─────┘
     │                             │
     │ DELETE (soft)               │ DELETE (soft)
     ▼                             ▼
┌──────────┐                 ┌──────────┐
│ELIMINADA │                 │ELIMINADA │
│(deleted) │                 │(deleted) │
└──────────┘                 └──────────┘

Estados:
- ACTIVA: deleted_at = NULL, is_active = true
- ELIMINADA: deleted_at != NULL (no aparece en listados normales)

Nota: No hay restauración de escuelas eliminadas en la API actual
```

## Manejo de Errores

| Código HTTP | Código Error | Significado | Acción en UI |
|-------------|--------------|-------------|--------------|
| 400 | INVALID_REQUEST | Datos inválidos (validación) | Mostrar errores por campo |
| 401 | UNAUTHORIZED | Token inválido/expirado | Redirigir a login |
| 403 | FORBIDDEN | Usuario no tiene permisos | Mostrar "Sin permisos" |
| 404 | NOT_FOUND | Escuela no existe | Mostrar "No encontrada" |
| 409 | CONFLICT | Código duplicado | Mostrar "Código ya existe" |
| 500 | INTERNAL_ERROR | Error del servidor | Mostrar "Error del sistema, intente más tarde" |

### Estructura de Error

```typescript
interface APIError {
  error: string;           // "conflict", "not_found", etc.
  message: string;         // Mensaje amigable
  code: string;            // "SCHOOL_CODE_DUPLICATE"
  details?: {
    field?: string;        // Campo que causó el error
    constraint?: string;   // Restricción violada
  };
}

// Ejemplo
const errorResponse: APIError = {
  error: "conflict",
  message: "Ya existe una escuela con ese código",
  code: "SCHOOL_CODE_DUPLICATE",
  details: {
    field: "code",
    constraint: "unique_school_code"
  }
};
```

## Consideraciones de UX

### Validaciones del Formulario (Frontend)

```typescript
const formValidations = {
  name: {
    required: "El nombre es obligatorio",
    minLength: { value: 3, message: "Mínimo 3 caracteres" },
    maxLength: { value: 255, message: "Máximo 255 caracteres" }
  },
  code: {
    required: "El código es obligatorio",
    minLength: { value: 3, message: "Mínimo 3 caracteres" },
    pattern: {
      value: /^[A-Z0-9_-]+$/,
      message: "Solo mayúsculas, números, guiones y guión bajo"
    }
  },
  contact_email: {
    pattern: {
      value: /^[^\s@]+@[^\s@]+\.[^\s@]+$/,
      message: "Email inválido"
    }
  },
  max_teachers: {
    min: { value: 1, message: "Mínimo 1 profesor" },
    max: { value: 10000, message: "Máximo 10,000 profesores" }
  },
  max_students: {
    min: { value: 1, message: "Mínimo 1 estudiante" },
    max: { value: 100000, message: "Máximo 100,000 estudiantes" }
  }
};
```

### Skeleton Loaders

```typescript
// Durante carga de lista
<SchoolCardSkeleton count={6} />

// Durante carga de detalle
<SchoolDetailSkeleton />
```

### Estados de Carga

```typescript
interface SchoolUIState {
  schools: SchoolResponse[];
  selectedSchool: SchoolResponse | null;
  loading: boolean;              // Para lista
  loadingDetail: boolean;        // Para detalle
  creating: boolean;             // Durante POST
  updating: boolean;             // Durante PUT
  deleting: boolean;             // Durante DELETE
  error: APIError | null;
}
```

### Confirmaciones

```typescript
// Antes de eliminar
const confirmDelete = {
  title: "¿Eliminar escuela?",
  message: `¿Está seguro de eliminar "${school.name}"? Esta acción marcará la escuela como inactiva.`,
  confirmText: "Sí, eliminar",
  cancelText: "Cancelar",
  type: "danger"
};
```

## Almacenamiento Local

### Cache de Escuelas (opcional)

```typescript
// Guardar lista en localStorage para acceso rápido
localStorage.setItem('schools_cache', JSON.stringify({
  data: schools,
  timestamp: Date.now(),
  ttl: 5 * 60 * 1000  // 5 minutos
}));

// Leer cache
function getCachedSchools(): SchoolResponse[] | null {
  const cache = localStorage.getItem('schools_cache');
  if (!cache) return null;

  const parsed = JSON.parse(cache);
  const isExpired = Date.now() - parsed.timestamp > parsed.ttl;

  return isExpired ? null : parsed.data;
}
```

## Código de Ejemplo (Mobile - TypeScript)

### Service

```typescript
// services/SchoolService.ts
import axios from 'axios';

const API_BASE = process.env.REACT_APP_API_ADMIN_URL || 'http://localhost:8081/v1';

export class SchoolService {

  /**
   * Crear nueva escuela
   */
  async createSchool(data: CreateSchoolRequest): Promise<SchoolResponse> {
    const response = await axios.post<SchoolResponse>(
      `${API_BASE}/schools`,
      data,
      {
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }
    );
    return response.data;
  }

  /**
   * Listar todas las escuelas
   * ⚠️ NOTA: Sin paginación, retorna todas
   */
  async listSchools(): Promise<SchoolResponse[]> {
    const response = await axios.get<SchoolResponse[]>(
      `${API_BASE}/schools`,
      {
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }
    );
    return response.data;
  }

  /**
   * Obtener escuela por UUID
   */
  async getSchool(id: string): Promise<SchoolResponse> {
    const response = await axios.get<SchoolResponse>(
      `${API_BASE}/schools/${id}`,
      {
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }
    );
    return response.data;
  }

  /**
   * Buscar escuela por código único
   */
  async getSchoolByCode(code: string): Promise<SchoolResponse> {
    const response = await axios.get<SchoolResponse>(
      `${API_BASE}/schools/code/${code}`,
      {
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }
    );
    return response.data;
  }

  /**
   * Actualizar escuela
   */
  async updateSchool(id: string, data: UpdateSchoolRequest): Promise<SchoolResponse> {
    const response = await axios.put<SchoolResponse>(
      `${API_BASE}/schools/${id}`,
      data,
      {
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }
    );
    return response.data;
  }

  /**
   * Eliminar escuela (soft delete)
   */
  async deleteSchool(id: string): Promise<void> {
    await axios.delete(
      `${API_BASE}/schools/${id}`,
      {
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }
    );
  }
}

function getAuthToken(): string {
  const token = localStorage.getItem('access_token');
  if (!token) throw new Error('No auth token');
  return token;
}
```

### Hook de React

```typescript
// hooks/useSchools.ts
import { useState, useEffect } from 'react';
import { SchoolService } from '../services/SchoolService';

export function useSchools() {
  const [schools, setSchools] = useState<SchoolResponse[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<APIError | null>(null);

  const service = new SchoolService();

  const loadSchools = async () => {
    setLoading(true);
    setError(null);
    try {
      const data = await service.listSchools();
      setSchools(data);
    } catch (err: any) {
      setError(err.response?.data || { error: 'unknown', message: 'Error al cargar escuelas' });
    } finally {
      setLoading(false);
    }
  };

  const createSchool = async (data: CreateSchoolRequest) => {
    setLoading(true);
    setError(null);
    try {
      const newSchool = await service.createSchool(data);
      setSchools(prev => [...prev, newSchool]);
      return newSchool;
    } catch (err: any) {
      setError(err.response?.data);
      throw err;
    } finally {
      setLoading(false);
    }
  };

  const updateSchool = async (id: string, data: UpdateSchoolRequest) => {
    setLoading(true);
    setError(null);
    try {
      const updated = await service.updateSchool(id, data);
      setSchools(prev => prev.map(s => s.id === id ? updated : s));
      return updated;
    } catch (err: any) {
      setError(err.response?.data);
      throw err;
    } finally {
      setLoading(false);
    }
  };

  const deleteSchool = async (id: string) => {
    setLoading(true);
    setError(null);
    try {
      await service.deleteSchool(id);
      setSchools(prev => prev.filter(s => s.id !== id));
    } catch (err: any) {
      setError(err.response?.data);
      throw err;
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    loadSchools();
  }, []);

  return {
    schools,
    loading,
    error,
    createSchool,
    updateSchool,
    deleteSchool,
    reload: loadSchools
  };
}
```

### Componente de Formulario

```typescript
// components/SchoolForm.tsx
import React from 'react';
import { useForm } from 'react-hook-form';

interface SchoolFormProps {
  school?: SchoolResponse;  // Para edición
  onSubmit: (data: CreateSchoolRequest) => Promise<void>;
  onCancel: () => void;
}

export function SchoolForm({ school, onSubmit, onCancel }: SchoolFormProps) {
  const { register, handleSubmit, formState: { errors, isSubmitting } } = useForm({
    defaultValues: school || {
      country: 'CO',
      subscription_tier: 'free',
      max_teachers: 50,
      max_students: 500
    }
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div className="form-group">
        <label>Nombre *</label>
        <input
          {...register('name', {
            required: 'El nombre es obligatorio',
            minLength: { value: 3, message: 'Mínimo 3 caracteres' }
          })}
          className={errors.name ? 'error' : ''}
        />
        {errors.name && <span className="error-msg">{errors.name.message}</span>}
      </div>

      <div className="form-group">
        <label>Código *</label>
        <input
          {...register('code', {
            required: 'El código es obligatorio',
            minLength: { value: 3, message: 'Mínimo 3 caracteres' },
            pattern: {
              value: /^[A-Z0-9_-]+$/,
              message: 'Solo mayúsculas, números, guiones y guión bajo'
            }
          })}
          placeholder="Ej: CSJ001"
          className={errors.code ? 'error' : ''}
        />
        {errors.code && <span className="error-msg">{errors.code.message}</span>}
      </div>

      <div className="form-row">
        <div className="form-group">
          <label>Ciudad</label>
          <input {...register('city')} />
        </div>

        <div className="form-group">
          <label>País</label>
          <select {...register('country')}>
            <option value="CO">Colombia</option>
            <option value="MX">México</option>
            <option value="AR">Argentina</option>
            {/* ... más países */}
          </select>
        </div>
      </div>

      <div className="form-group">
        <label>Email de contacto</label>
        <input
          type="email"
          {...register('contact_email', {
            pattern: {
              value: /^[^\s@]+@[^\s@]+\.[^\s@]+$/,
              message: 'Email inválido'
            }
          })}
          className={errors.contact_email ? 'error' : ''}
        />
        {errors.contact_email && <span className="error-msg">{errors.contact_email.message}</span>}
      </div>

      <div className="form-group">
        <label>Teléfono de contacto</label>
        <input {...register('contact_phone')} placeholder="+57 1 234 5678" />
      </div>

      <div className="form-row">
        <div className="form-group">
          <label>Tier de Subscripción</label>
          <select {...register('subscription_tier')}>
            <option value="free">Gratuito</option>
            <option value="basic">Básico</option>
            <option value="premium">Premium</option>
          </select>
        </div>

        <div className="form-group">
          <label>Límite de Profesores</label>
          <input
            type="number"
            {...register('max_teachers', {
              min: { value: 1, message: 'Mínimo 1' },
              valueAsNumber: true
            })}
            className={errors.max_teachers ? 'error' : ''}
          />
        </div>

        <div className="form-group">
          <label>Límite de Estudiantes</label>
          <input
            type="number"
            {...register('max_students', {
              min: { value: 1, message: 'Mínimo 1' },
              valueAsNumber: true
            })}
            className={errors.max_students ? 'error' : ''}
          />
        </div>
      </div>

      <div className="form-actions">
        <button type="button" onClick={onCancel} disabled={isSubmitting}>
          Cancelar
        </button>
        <button type="submit" disabled={isSubmitting}>
          {isSubmitting ? 'Guardando...' : school ? 'Actualizar' : 'Crear'}
        </button>
      </div>
    </form>
  );
}
```

## Notas de Implementación

### Seguridad

1. **Autorización:**
   - ⚠️ Actualmente NO hay middleware de roles en API Admin
   - Cualquier usuario autenticado puede modificar escuelas
   - **RECOMENDACIÓN:** Implementar RBAC en backend antes de producción
   - Mientras tanto, validar en frontend que usuario tenga rol admin

2. **Validación de Código:**
   - Backend valida unicidad de código
   - Frontend debe manejar error 409 CONFLICT apropiadamente
   - Sugerencia: validación asíncrona mientras usuario escribe

### Performance

1. **Lista sin Paginación:**
   - ⚠️ `GET /v1/schools` retorna TODAS las escuelas
   - Para 10-50 escuelas: aceptable
   - Para >100 escuelas: implementar paginación en backend
   - Alternativa temporal: paginación en frontend

2. **Cache:**
   - Considerar cache de lista en localStorage (5 min TTL)
   - Invalidar cache al crear/actualizar/eliminar
   - Útil para mejorar navegación entre vistas

### Búsqueda

```typescript
// Búsqueda local (frontend)
function filterSchools(schools: SchoolResponse[], query: string) {
  const lowerQuery = query.toLowerCase();
  return schools.filter(school =>
    school.name.toLowerCase().includes(lowerQuery) ||
    school.code.toLowerCase().includes(lowerQuery) ||
    school.city?.toLowerCase().includes(lowerQuery)
  );
}
```

### Metadata Extensible

```typescript
// Ejemplo de metadata personalizada
const customMetadata = {
  foundation_year: 1950,
  principal: "María García",
  accreditation: "A+",
  campus_size_m2: 5000,
  special_programs: ["STEM", "Arts", "Sports"],
  social_media: {
    facebook: "colegiosanjose",
    instagram: "@colegiosanjose"
  }
};
```

### Testing

```typescript
// Test de creación
describe('SchoolService.createSchool', () => {
  it('debe crear escuela con datos válidos', async () => {
    const data = {
      name: "Colegio Test",
      code: "TEST001"
    };

    const school = await service.createSchool(data);

    expect(school.id).toBeDefined();
    expect(school.name).toBe("Colegio Test");
    expect(school.code).toBe("TEST001");
    expect(school.is_active).toBe(true);
  });

  it('debe rechazar código duplicado', async () => {
    await expect(
      service.createSchool({ name: "Test", code: "EXIST001" })
    ).rejects.toMatchObject({
      response: { status: 409 }
    });
  });
});
```

### Próximos Pasos

1. ✅ Implementar CRUD básico
2. ⚠️ Agregar paginación en backend
3. ⚠️ Implementar RBAC en API Admin
4. ✅ Agregar filtros de búsqueda en frontend
5. ✅ Implementar cache local
6. ✅ Agregar exportación a CSV/Excel
7. ✅ Dashboard con estadísticas de escuelas
