# RFC-012: Membresías (Asignación de Usuarios a Unidades Académicas)

## Metadata
- **ID:** RFC-012
- **Proceso:** Gestión Escolar
- **Subproceso:** Asignación de Roles
- **Prioridad:** Alta
- **Dependencias:** RFC-010 (CRUD Escuelas), RFC-011 (Jerarquía Académica)
- **Estado API:** ✅ Listo en main

## Descripción

Permite asignar usuarios (docentes, estudiantes, asistentes) a unidades académicas específicas con roles determinados. Una membresía representa la relación entre un usuario y una unidad académica (ej: "Juan Pérez es teacher en Grado 6 - Sección A"). Soporta membresías temporales con fechas de validez y expiración.

## Flujo de Usuario (UX)

### Ver Membresías de una Unidad (Admin/Coordinador)
1. Admin selecciona unidad académica (ej: Grado 6 - Sección A)
2. Click en "Ver Miembros"
3. Sistema carga membresías activas
4. Muestra tabla agrupada por rol:
   ```
   👨‍🏫 Docentes (2)
   - Juan Pérez (teacher) - Desde: 2024-01-15
   - María García (assistant) - Desde: 2024-02-01

   👥 Estudiantes (28)
   - Ana López (student) - Desde: 2024-01-10
   - Carlos Rodríguez (student) - Desde: 2024-01-10
   ...
   ```
5. Filtros disponibles: activas/inactivas, por rol, por fecha

### Asignar Usuario a Unidad (Admin)
1. Admin en vista de unidad académica
2. Click "Agregar Miembro"
3. Formulario muestra:
   - Seleccionar usuario (búsqueda por nombre/email)
   - Seleccionar rol (teacher/student/assistant/owner)
   - Fecha inicio (opcional, default: hoy)
   - Fecha fin (opcional)
4. Sistema valida que usuario no tenga membresía activa en esa unidad con ese rol
5. Crea membresía → Aparece en lista de miembros
6. Usuario ahora tiene acceso a contenido de esa unidad

### Ver Membresías de un Usuario
1. Admin busca usuario
2. Click en "Ver Asignaciones"
3. Sistema carga todas las membresías del usuario
4. Muestra lista agrupada por escuela/unidad:
   ```
   🏫 Colegio San José
     📖 Grado 6 - Sección A
       - Rol: teacher
       - Desde: 2024-01-15
       - Estado: Activa

     📖 Grado 5 - Sección B
       - Rol: assistant
       - Desde: 2024-01-10
       - Hasta: 2024-06-30
       - Estado: Activa (expira en 45 días)
   ```

### Editar Membresía
1. Admin en vista de membresías
2. Click en membresía específica
3. Puede modificar:
   - Rol
   - Fecha de inicio
   - Fecha de fin
4. Sistema valida cambios
5. Actualiza → Muestra confirmación

### Eliminar Membresía (Hard Delete)
1. Admin selecciona membresía
2. Click "Eliminar"
3. Confirmación modal:
   - "¿Eliminar asignación de [Usuario] como [Rol] en [Unidad]?"
   - "El usuario perderá acceso inmediatamente"
4. Confirma → Elimina permanentemente
5. Desaparece de lista

### Expirar Membresía
1. Admin selecciona membresía activa
2. Click "Expirar Ahora"
3. Confirmación modal
4. Sistema establece fecha fin = hoy
5. Membresía pasa a inactiva
6. Usuario pierde acceso

### Filtrar Membresías por Rol
1. Admin en vista de unidad
2. Selecciona filtro: "Solo Docentes"
3. Sistema filtra lista mostrando solo teachers/assistants/owners
4. Útil para ver rápidamente equipo docente

## Flujo de Datos (Técnico)

### Endpoints Involucrados

| Endpoint | Método | Descripción | Estado |
|----------|--------|-------------|--------|
| `/v1/memberships` | POST | Crear nueva membresía | ✅ |
| `/v1/memberships` | GET | Listar membresías por unidad | ⚠️ Sin paginación |
| `/v1/memberships/by-role` | GET | Filtrar por rol | ✅ |
| `/v1/memberships/:id` | GET | Obtener membresía específica | ✅ |
| `/v1/memberships/:id` | PUT | Actualizar membresía | ✅ |
| `/v1/memberships/:id` | DELETE | Eliminar (hard delete) | ✅ |
| `/v1/memberships/:id/expire` | POST | Expirar membresía | ✅ |
| `/v1/users/:userId/memberships` | GET | Membresías de usuario | ✅ |

### Request/Response (TypeScript interfaces)

```typescript
// ============= CREATE MEMBERSHIP =============
interface CreateMembershipRequest {
  unit_id: string;                    // UUID, requerido
  user_id: string;                    // UUID, requerido
  role: MembershipRole;               // Requerido
  valid_from?: string;                // ISO8601, opcional (default: now)
  valid_until?: string;               // ISO8601, opcional (null = sin expiración)
}

type MembershipRole =
  | "owner"        // Dueño/Director de la unidad
  | "teacher"      // Docente titular
  | "assistant"    // Asistente/Co-docente
  | "student"      // Estudiante
  | "guardian";    // Acudiente (uso limitado)

// Ejemplo
const createRequest: CreateMembershipRequest = {
  unit_id: "uuid-seccion-6a",
  user_id: "uuid-juan-perez",
  role: "teacher",
  valid_from: "2024-01-15T00:00:00Z",
  valid_until: "2024-12-15T23:59:59Z"  // Membresía por año escolar
};

// ============= MEMBERSHIP RESPONSE =============
interface MembershipResponse {
  id: string;                         // UUID
  unit_id: string;                    // UUID
  user_id: string;                    // UUID
  role: MembershipRole;
  enrolled_at: string;                // ISO8601 - Cuándo se asignó
  withdrawn_at: string | null;        // ISO8601 - Cuándo se retiró (si aplica)
  is_active: boolean;                 // Calculado: now >= valid_from && now <= valid_until
  created_at: string;                 // ISO8601
  updated_at: string;                 // ISO8601

  // Campos extendidos (joins opcionales)
  user?: {
    id: string;
    name: string;
    email: string;
  };
  unit?: {
    id: string;
    display_name: string;
    type: string;
  };
}

// ============= UPDATE MEMBERSHIP =============
interface UpdateMembershipRequest {
  role?: MembershipRole;              // Cambiar rol
  valid_from?: string;                // Cambiar fecha inicio
  valid_until?: string;               // Cambiar fecha fin (null = sin expiración)
}

// ============= LIST PARAMS =============
interface ListMembershipsParams {
  unit_id?: string;                   // Filtrar por unidad
  activeOnly?: boolean;               // Default: false
}

interface ListByRoleParams {
  unit_id: string;                    // Requerido
  role: MembershipRole;               // Requerido
  activeOnly?: boolean;               // Default: false
}

interface ListByUserParams {
  activeOnly?: boolean;               // Default: false
}
```

### Validaciones del Backend

```typescript
const validations = {
  unit_id: {
    required: true,
    uuid: true,
    exists: true                      // Debe existir en BD
  },
  user_id: {
    required: true,
    uuid: true,
    exists: true,                     // Debe existir en BD
    sameSchool: true                  // Usuario debe tener school_id coincidente
  },
  role: {
    required: true,
    enum: ["owner", "teacher", "assistant", "student", "guardian"]
  },
  valid_from: {
    iso8601: true,
    optional: true,
    default: "now"
  },
  valid_until: {
    iso8601: true,
    optional: true,
    afterValidFrom: true              // Debe ser posterior a valid_from
  }
};

// Validaciones de negocio
const businessRules = {
  noDuplicateActiveRole: true,        // No permitir 2 membresías activas con mismo rol en misma unidad
  ownerLimit: 1,                      // Solo 1 owner activo por unidad
  teacherLimit: null,                 // Sin límite (puede haber múltiples co-docentes)
  studentLimit: null                  // Sin límite
};
```

### Lógica de Estado Activo

```sql
-- Una membresía está activa si:
SELECT *,
  CASE
    WHEN withdrawn_at IS NOT NULL THEN false
    WHEN valid_from IS NOT NULL AND NOW() < valid_from THEN false
    WHEN valid_until IS NOT NULL AND NOW() > valid_until THEN false
    ELSE true
  END AS is_active
FROM memberships;

-- Estados posibles:
-- PENDIENTE: valid_from > now (aún no inicia)
-- ACTIVA: valid_from <= now <= valid_until
-- EXPIRADA: valid_until < now
-- RETIRADA: withdrawn_at != NULL
```

## Estados y Transiciones

```
┌─────────┐
│ CREANDO │
└────┬────┘
     │
     ▼
┌──────────┐
│ PENDIENTE│  (si valid_from > now)
│(not started)│
└────┬─────┘
     │ Pasa el tiempo (valid_from)
     ▼
┌─────────┐     UPDATE      ┌────────────┐
│ ACTIVA  ├────────────────►│  ACTIVA    │
│(current)│                 │ (modificada)│
└────┬────┘                 └──────┬─────┘
     │                             │
     │ Pasa el tiempo              │ EXPIRE
     │ (valid_until)               │
     ▼                             ▼
┌──────────┐                 ┌──────────┐
│ EXPIRADA │                 │ RETIRADA │
│(expired) │                 │(withdrawn)│
└──────────┘                 └──────────┘
     │                             │
     │ DELETE                      │ DELETE
     ▼                             ▼
┌──────────┐                 ┌──────────┐
│ ELIMINADA│                 │ ELIMINADA│
│(hard del)│                 │(hard del)│
└──────────┘                 └──────────┘

Estados:
- PENDIENTE: valid_from > now, is_active = false
- ACTIVA: valid_from <= now <= valid_until, is_active = true
- EXPIRADA: valid_until < now, is_active = false
- RETIRADA: withdrawn_at != NULL, is_active = false

Nota: DELETE es hard delete (elimina permanentemente)
```

## Manejo de Errores

| Código HTTP | Código Error | Significado | Acción en UI |
|-------------|--------------|-------------|--------------|
| 400 | INVALID_REQUEST | Datos inválidos | Mostrar errores por campo |
| 400 | INVALID_DATE_RANGE | valid_until < valid_from | "Fecha fin debe ser posterior a inicio" |
| 401 | UNAUTHORIZED | Token inválido | Redirigir a login |
| 403 | FORBIDDEN | Sin permisos | "No tiene permisos" |
| 404 | NOT_FOUND | Membresía no existe | "Membresía no encontrada" |
| 404 | UNIT_NOT_FOUND | Unidad no existe | "Unidad académica no encontrada" |
| 404 | USER_NOT_FOUND | Usuario no existe | "Usuario no encontrado" |
| 409 | CONFLICT | Membresía duplicada | "Usuario ya tiene este rol en esta unidad" |
| 409 | SCHOOL_MISMATCH | Usuario de otra escuela | "Usuario pertenece a otra escuela" |
| 500 | INTERNAL_ERROR | Error del servidor | "Error del sistema" |

### Estructura de Error

```typescript
interface MembershipError extends APIError {
  details?: {
    field?: string;
    user_id?: string;
    unit_id?: string;
    existing_membership_id?: string;  // Si hay conflicto
  };
}

// Ejemplo de error de duplicado
const duplicateError: MembershipError = {
  error: "conflict",
  message: "El usuario ya tiene una membresía activa como teacher en esta unidad",
  code: "DUPLICATE_ACTIVE_MEMBERSHIP",
  details: {
    user_id: "uuid-juan",
    unit_id: "uuid-6a",
    existing_membership_id: "uuid-membership-existente"
  }
};
```

## Consideraciones de UX

### Vista de Tabla de Membresías

```typescript
interface MembershipTableProps {
  memberships: MembershipResponse[];
  onEdit: (membership: MembershipResponse) => void;
  onDelete: (membership: MembershipResponse) => void;
  onExpire: (membership: MembershipResponse) => void;
}

const MembershipTable = ({ memberships, ...handlers }: MembershipTableProps) => {
  const grouped = groupByRole(memberships);

  return (
    <div className="memberships-table">
      {Object.entries(grouped).map(([role, members]) => (
        <div key={role} className="role-group">
          <h3>{getRoleIcon(role)} {getRoleLabel(role)} ({members.length})</h3>
          <table>
            <thead>
              <tr>
                <th>Usuario</th>
                <th>Email</th>
                <th>Desde</th>
                <th>Hasta</th>
                <th>Estado</th>
                <th>Acciones</th>
              </tr>
            </thead>
            <tbody>
              {members.map(member => (
                <tr key={member.id} className={member.is_active ? '' : 'inactive'}>
                  <td>{member.user?.name}</td>
                  <td>{member.user?.email}</td>
                  <td>{formatDate(member.enrolled_at)}</td>
                  <td>{member.withdrawn_at ? formatDate(member.withdrawn_at) : 'Permanente'}</td>
                  <td>
                    <Badge status={getMembershipStatus(member)} />
                  </td>
                  <td>
                    <button onClick={() => handlers.onEdit(member)}>✏️</button>
                    {member.is_active && (
                      <button onClick={() => handlers.onExpire(member)}>⏸️</button>
                    )}
                    <button onClick={() => handlers.onDelete(member)}>🗑️</button>
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      ))}
    </div>
  );
};

function getMembershipStatus(membership: MembershipResponse): 'active' | 'pending' | 'expired' | 'withdrawn' {
  if (membership.withdrawn_at) return 'withdrawn';
  if (!membership.is_active) {
    const validFrom = new Date(membership.enrolled_at);
    const now = new Date();
    if (validFrom > now) return 'pending';
    return 'expired';
  }
  return 'active';
}
```

### Badges de Estado

```typescript
const StatusBadge = ({ status }: { status: MembershipStatus }) => {
  const config = {
    active: { color: 'green', label: 'Activa' },
    pending: { color: 'blue', label: 'Pendiente' },
    expired: { color: 'orange', label: 'Expirada' },
    withdrawn: { color: 'red', label: 'Retirada' }
  };

  const { color, label } = config[status];

  return <span className={`badge badge-${color}`}>{label}</span>;
};
```

### Selector de Usuario (Autocomplete)

```typescript
interface UserSelectorProps {
  schoolId: string;
  onSelect: (user: UserSummary) => void;
  excludeUserIds?: string[];  // Usuarios ya asignados
}

const UserSelector = ({ schoolId, onSelect, excludeUserIds = [] }: UserSelectorProps) => {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<UserSummary[]>([]);

  const searchUsers = async (q: string) => {
    const users = await userService.searchUsers(schoolId, q);
    const filtered = users.filter(u => !excludeUserIds.includes(u.id));
    setResults(filtered);
  };

  return (
    <div className="user-selector">
      <input
        type="text"
        value={query}
        onChange={(e) => {
          setQuery(e.target.value);
          if (e.target.value.length >= 3) {
            searchUsers(e.target.value);
          }
        }}
        placeholder="Buscar usuario por nombre o email..."
      />
      {results.length > 0 && (
        <ul className="autocomplete-results">
          {results.map(user => (
            <li key={user.id} onClick={() => onSelect(user)}>
              <div className="user-info">
                <strong>{user.name}</strong>
                <span>{user.email}</span>
              </div>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
};
```

## Almacenamiento Local

### Cache de Membresías por Unidad

```typescript
interface MembershipCache {
  unitId: string;
  memberships: MembershipResponse[];
  timestamp: number;
  ttl: number;  // 2 minutos
}

function cacheMemberships(unitId: string, memberships: MembershipResponse[]) {
  const cache: MembershipCache = {
    unitId,
    memberships,
    timestamp: Date.now(),
    ttl: 2 * 60 * 1000
  };
  localStorage.setItem(`memberships_${unitId}`, JSON.stringify(cache));
}

function getCachedMemberships(unitId: string): MembershipResponse[] | null {
  const cached = localStorage.getItem(`memberships_${unitId}`);
  if (!cached) return null;

  const parsed: MembershipCache = JSON.parse(cached);
  const isExpired = Date.now() - parsed.timestamp > parsed.ttl;

  return isExpired ? null : parsed.memberships;
}
```

## Código de Ejemplo (Mobile - TypeScript)

### Service

```typescript
// services/MembershipService.ts
import axios from 'axios';

const API_BASE = process.env.REACT_APP_API_ADMIN_URL || 'http://localhost:8081/v1';

export class MembershipService {

  /**
   * Crear nueva membresía
   */
  async createMembership(data: CreateMembershipRequest): Promise<MembershipResponse> {
    const response = await axios.post<MembershipResponse>(
      `${API_BASE}/memberships`,
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
   * Listar membresías por unidad
   * ⚠️ NOTA: Sin paginación
   */
  async listMembershipsByUnit(
    unitId: string,
    activeOnly: boolean = false
  ): Promise<MembershipResponse[]> {
    const response = await axios.get<MembershipResponse[]>(
      `${API_BASE}/memberships`,
      {
        params: { unit_id: unitId, activeOnly },
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }
    );
    return response.data;
  }

  /**
   * Listar membresías por rol
   */
  async listMembershipsByRole(
    unitId: string,
    role: MembershipRole,
    activeOnly: boolean = false
  ): Promise<MembershipResponse[]> {
    const response = await axios.get<MembershipResponse[]>(
      `${API_BASE}/memberships/by-role`,
      {
        params: { unit_id: unitId, role, activeOnly },
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }
    );
    return response.data;
  }

  /**
   * Obtener membresía específica
   */
  async getMembership(id: string): Promise<MembershipResponse> {
    const response = await axios.get<MembershipResponse>(
      `${API_BASE}/memberships/${id}`,
      {
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }
    );
    return response.data;
  }

  /**
   * Listar membresías de un usuario
   */
  async listMembershipsByUser(
    userId: string,
    activeOnly: boolean = false
  ): Promise<MembershipResponse[]> {
    const response = await axios.get<MembershipResponse[]>(
      `${API_BASE}/users/${userId}/memberships`,
      {
        params: { activeOnly },
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }
    );
    return response.data;
  }

  /**
   * Actualizar membresía
   */
  async updateMembership(
    id: string,
    data: UpdateMembershipRequest
  ): Promise<MembershipResponse> {
    const response = await axios.put<MembershipResponse>(
      `${API_BASE}/memberships/${id}`,
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
   * Eliminar membresía (hard delete)
   */
  async deleteMembership(id: string): Promise<void> {
    await axios.delete(
      `${API_BASE}/memberships/${id}`,
      {
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }
    );
  }

  /**
   * Expirar membresía inmediatamente
   */
  async expireMembership(id: string): Promise<void> {
    await axios.post(
      `${API_BASE}/memberships/${id}/expire`,
      {},
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
// hooks/useMemberships.ts
import { useState, useEffect } from 'react';
import { MembershipService } from '../services/MembershipService';

export function useMemberships(unitId: string, activeOnly: boolean = true) {
  const [memberships, setMemberships] = useState<MembershipResponse[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<APIError | null>(null);

  const service = new MembershipService();

  const loadMemberships = async () => {
    setLoading(true);
    setError(null);
    try {
      const data = await service.listMembershipsByUnit(unitId, activeOnly);
      setMemberships(data);
    } catch (err: any) {
      setError(err.response?.data);
    } finally {
      setLoading(false);
    }
  };

  const createMembership = async (data: CreateMembershipRequest) => {
    setLoading(true);
    setError(null);
    try {
      const newMembership = await service.createMembership(data);
      setMemberships(prev => [...prev, newMembership]);
      return newMembership;
    } catch (err: any) {
      setError(err.response?.data);
      throw err;
    } finally {
      setLoading(false);
    }
  };

  const updateMembership = async (id: string, data: UpdateMembershipRequest) => {
    setLoading(true);
    setError(null);
    try {
      const updated = await service.updateMembership(id, data);
      setMemberships(prev => prev.map(m => m.id === id ? updated : m));
      return updated;
    } catch (err: any) {
      setError(err.response?.data);
      throw err;
    } finally {
      setLoading(false);
    }
  };

  const deleteMembership = async (id: string) => {
    setLoading(true);
    setError(null);
    try {
      await service.deleteMembership(id);
      setMemberships(prev => prev.filter(m => m.id !== id));
    } catch (err: any) {
      setError(err.response?.data);
      throw err;
    } finally {
      setLoading(false);
    }
  };

  const expireMembership = async (id: string) => {
    setLoading(true);
    setError(null);
    try {
      await service.expireMembership(id);
      await loadMemberships(); // Recargar para actualizar estado
    } catch (err: any) {
      setError(err.response?.data);
      throw err;
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    if (unitId) {
      loadMemberships();
    }
  }, [unitId, activeOnly]);

  return {
    memberships,
    loading,
    error,
    createMembership,
    updateMembership,
    deleteMembership,
    expireMembership,
    reload: loadMemberships
  };
}
```

### Componente de Formulario

```typescript
// components/MembershipForm.tsx
import React from 'react';
import { useForm } from 'react-hook-form';

interface MembershipFormProps {
  unitId: string;
  membership?: MembershipResponse;  // Para edición
  onSubmit: (data: CreateMembershipRequest) => Promise<void>;
  onCancel: () => void;
}

export function MembershipForm({ unitId, membership, onSubmit, onCancel }: MembershipFormProps) {
  const { register, handleSubmit, formState: { errors, isSubmitting } } = useForm({
    defaultValues: membership || {
      unit_id: unitId,
      role: 'student'
    }
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div className="form-group">
        <label>Rol *</label>
        <select
          {...register('role', { required: 'El rol es obligatorio' })}
          className={errors.role ? 'error' : ''}
        >
          <option value="student">👥 Estudiante</option>
          <option value="teacher">👨‍🏫 Docente</option>
          <option value="assistant">🧑‍🏫 Asistente</option>
          <option value="owner">👑 Director</option>
          <option value="guardian">👨‍👩‍👧 Acudiente</option>
        </select>
        {errors.role && <span className="error-msg">{errors.role.message}</span>}
      </div>

      <div className="form-row">
        <div className="form-group">
          <label>Fecha de Inicio</label>
          <input
            type="date"
            {...register('valid_from')}
          />
          <small>Dejar vacío para inicio inmediato</small>
        </div>

        <div className="form-group">
          <label>Fecha de Fin</label>
          <input
            type="date"
            {...register('valid_until')}
          />
          <small>Dejar vacío para membresía permanente</small>
        </div>
      </div>

      <div className="form-actions">
        <button type="button" onClick={onCancel} disabled={isSubmitting}>
          Cancelar
        </button>
        <button type="submit" disabled={isSubmitting}>
          {isSubmitting ? 'Guardando...' : membership ? 'Actualizar' : 'Crear'}
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
   - Cualquier usuario autenticado puede modificar membresías
   - **RECOMENDACIÓN:** Solo admin/coordinador deberían crear/modificar
   - Validar en frontend mientras se implementa RBAC en backend

2. **Validación de Escuela:**
   - Backend valida que usuario pertenezca a la misma escuela que la unidad
   - Previene asignaciones cross-school

### Performance

1. **Consultas sin Paginación:**
   - Para 50-100 membresías por unidad: aceptable
   - Para >200: considerar paginación en backend
   - Alternativa temporal: paginación en frontend

2. **Joins Opcionales:**
   - Response puede incluir datos de usuario y unidad
   - Reduce requests adicionales
   - Útil para tablas

### Casos de Uso Comunes

```typescript
// Asignar docente a sección
await service.createMembership({
  unit_id: 'uuid-6a',
  user_id: 'uuid-juan-perez',
  role: 'teacher',
  valid_from: '2024-01-15T00:00:00Z',
  valid_until: '2024-12-15T23:59:59Z'
});

// Asignar estudiante permanente
await service.createMembership({
  unit_id: 'uuid-6a',
  user_id: 'uuid-ana-lopez',
  role: 'student'
  // Sin fechas = permanente
});

// Cambiar rol de asistente a titular
await service.updateMembership('uuid-membership', {
  role: 'teacher'
});

// Expirar membresía al finalizar año
await service.expireMembership('uuid-membership');
```

### Reportes Útiles

```typescript
// Obtener todos los docentes activos de una escuela
async function getActiveTeachers(schoolId: string): Promise<MembershipResponse[]> {
  const units = await unitService.listUnits(schoolId);
  const memberships: MembershipResponse[] = [];

  for (const unit of units) {
    const teachers = await service.listMembershipsByRole(unit.id, 'teacher', true);
    memberships.push(...teachers);
  }

  // Eliminar duplicados (docente en múltiples unidades)
  const uniqueUsers = new Map<string, MembershipResponse>();
  memberships.forEach(m => {
    if (!uniqueUsers.has(m.user_id)) {
      uniqueUsers.set(m.user_id, m);
    }
  });

  return Array.from(uniqueUsers.values());
}

// Contar estudiantes por unidad
async function getStudentCount(unitId: string): Promise<number> {
  const students = await service.listMembershipsByRole(unitId, 'student', true);
  return students.length;
}
```

### Próximos Pasos

1. ✅ Implementar asignación masiva (bulk creation)
2. ✅ Importar membresías desde Excel
3. ✅ Exportar listados de estudiantes/docentes
4. ✅ Dashboard de asignaciones por escuela
5. ✅ Notificaciones al asignar/expirar membresías
6. ✅ Histórico de cambios de membresías
7. ✅ Validación de capacidad máxima por unidad
