# RFC-013: Relaciones Acudientes (Guardian-Student)

## Metadata
- **ID:** RFC-013
- **Proceso:** Gestión Escolar
- **Subproceso:** Relaciones Familiares
- **Prioridad:** Media
- **Dependencias:** RFC-010 (CRUD Escuelas)
- **Estado API:** ⚠️ Solo en rama dev (NO en main)

## Descripción

Gestión de relaciones entre acudientes (guardians) y estudiantes. Permite registrar vínculos familiares (padre, madre, abuelo, tío, etc.) y mantener información de contacto de emergencia. Los acudientes pueden tener acceso limitado a información académica de sus pupilos.

**⚠️ IMPORTANTE:** Estos endpoints están SOLO en la rama `dev` de edugo-api-administracion. Requiere merge dev→main antes de usar en producción.

## Flujo de Usuario (UX)

### Ver Acudientes de un Estudiante (Admin/Coordinador)
1. Admin busca estudiante
2. Click en "Ver Acudientes"
3. Sistema carga relaciones activas
4. Muestra tarjetas por acudiente:
   ```
   👨‍👩‍👧 Acudientes de Ana López

   📧 María García
   - Relación: Madre
   - Email: maria.garcia@example.com
   - Teléfono: +57 300 123 4567
   - Estado: Activo
   - Desde: 2024-01-10

   📧 Carlos López
   - Relación: Padre
   - Email: carlos.lopez@example.com
   - Teléfono: +57 310 987 6543
   - Estado: Activo
   - Desde: 2024-01-10
   ```
5. Opciones: Agregar acudiente, Editar, Eliminar

### Registrar Acudiente para Estudiante
1. Admin en vista de estudiante
2. Click "Agregar Acudiente"
3. Formulario muestra:
   - Seleccionar usuario acudiente (búsqueda)
   - Tipo de relación (padre/madre/abuelo/etc.)
4. Sistema valida que usuario tenga rol guardian
5. Crea relación → Aparece en lista
6. Acudiente puede ahora ver información del estudiante

### Ver Estudiantes de un Acudiente
1. Admin selecciona usuario guardian
2. Click en "Ver Pupilos"
3. Sistema carga estudiantes vinculados
4. Muestra lista:
   ```
   👥 Pupilos de María García

   📖 Ana López (Hija)
   - Grado: 6° - Sección A
   - Estado: Activo

   📖 Juan López (Hijo)
   - Grado: 3° - Sección B
   - Estado: Activo
   ```

### Editar Relación
1. Admin en vista de relación
2. Click "Editar"
3. Puede modificar:
   - Tipo de relación
4. NO puede cambiar guardian ni estudiante (crear nueva relación)
5. Actualiza → Muestra confirmación

### Eliminar Relación (Soft Delete)
1. Admin selecciona relación
2. Click "Eliminar"
3. Confirmación modal:
   - "¿Eliminar vínculo de [Guardian] como [Relación] de [Estudiante]?"
   - "El acudiente perderá acceso a información del estudiante"
4. Confirma → Soft delete (deleted_at)
5. Desaparece de lista activas

## Flujo de Datos (Técnico)

### Endpoints Involucrados

| Endpoint | Método | Descripción | Estado |
|----------|--------|-------------|--------|
| `/v1/guardian-relations` | POST | Crear relación guardian-student | ⚠️ Solo dev |
| `/v1/guardian-relations/:id` | GET | Obtener relación específica | ⚠️ Solo dev |
| `/v1/guardian-relations/:id` | PUT | Actualizar relación | ⚠️ Solo dev |
| `/v1/guardian-relations/:id` | DELETE | Soft delete de relación | ⚠️ Solo dev |
| `/v1/guardians/:guardian_id/relations` | GET | Relaciones de un guardian | ⚠️ Solo dev |
| `/v1/students/:student_id/guardians` | GET | Guardians de un estudiante | ⚠️ Solo dev |

### Request/Response (TypeScript interfaces)

```typescript
// ============= CREATE RELATION =============
interface CreateGuardianRelationRequest {
  guardian_id: string;                // UUID, requerido
  student_id: string;                 // UUID, requerido
  relationship_type: RelationshipType; // Requerido
}

type RelationshipType =
  | "father"        // Padre
  | "mother"        // Madre
  | "grandfather"   // Abuelo
  | "grandmother"   // Abuela
  | "uncle"         // Tío
  | "aunt"          // Tía
  | "other";        // Otro (tutor legal, etc.)

// Ejemplo
const createRequest: CreateGuardianRelationRequest = {
  guardian_id: "uuid-maria-garcia",
  student_id: "uuid-ana-lopez",
  relationship_type: "mother"
};

// ============= GUARDIAN RELATION RESPONSE =============
interface GuardianRelationResponse {
  id: string;                         // UUID
  guardian_id: string;                // UUID
  student_id: string;                 // UUID
  relationship_type: RelationshipType;
  is_active: boolean;
  created_at: string;                 // ISO8601
  updated_at: string;                 // ISO8601
  created_by: string;                 // UUID del admin que creó la relación

  // Campos extendidos (joins opcionales)
  guardian?: {
    id: string;
    name: string;
    email: string;
    phone?: string;
  };
  student?: {
    id: string;
    name: string;
    email: string;
    current_grade?: string;
  };
}

// ============= UPDATE RELATION =============
interface UpdateGuardianRelationRequest {
  relationship_type?: RelationshipType;  // Único campo modificable
}

// Nota: NO se puede cambiar guardian_id ni student_id
// Para cambiar personas, eliminar relación y crear nueva
```

### Validaciones del Backend

```typescript
const validations = {
  guardian_id: {
    required: true,
    uuid: true,
    exists: true,                     // Debe existir en tabla users
    hasGuardianRole: true,            // Usuario debe tener rol guardian
    sameSchool: true                  // Mismo school_id que estudiante
  },
  student_id: {
    required: true,
    uuid: true,
    exists: true,                     // Debe existir en tabla users
    hasStudentRole: true,             // Usuario debe tener rol student
    sameSchool: true
  },
  relationship_type: {
    required: true,
    enum: ["father", "mother", "grandfather", "grandmother", "uncle", "aunt", "other"]
  }
};

// Validaciones de negocio
const businessRules = {
  noDuplicateRelation: true,          // No permitir misma relación 2 veces
  maxGuardiansPerStudent: null,       // Sin límite (puede tener múltiples)
  maxStudentsPerGuardian: null        // Sin límite (puede cuidar múltiples)
};
```

### Lógica de Tabla PostgreSQL

```sql
CREATE TABLE guardian_relations (
  id UUID PRIMARY KEY,
  guardian_id UUID NOT NULL REFERENCES users(id),
  student_id UUID NOT NULL REFERENCES users(id),
  relationship_type VARCHAR(20) NOT NULL CHECK (relationship_type IN (
    'father', 'mother', 'grandfather', 'grandmother', 'uncle', 'aunt', 'other'
  )),
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
  deleted_at TIMESTAMP,
  created_by UUID NOT NULL REFERENCES users(id),  -- Admin que creó

  -- Restricciones
  CONSTRAINT no_self_guardian CHECK (guardian_id != student_id),
  CONSTRAINT unique_active_relation UNIQUE (guardian_id, student_id, relationship_type)
  WHERE deleted_at IS NULL
);

-- Índices
CREATE INDEX idx_guardian_relations_guardian ON guardian_relations(guardian_id)
WHERE deleted_at IS NULL;

CREATE INDEX idx_guardian_relations_student ON guardian_relations(student_id)
WHERE deleted_at IS NULL;
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
│(normal) │  (cambio tipo)  │ (modificada)│
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

Nota: No hay restauración de relaciones en la API actual
```

## Manejo de Errores

| Código HTTP | Código Error | Significado | Acción en UI |
|-------------|--------------|-------------|--------------|
| 400 | INVALID_REQUEST | Datos inválidos | Mostrar errores por campo |
| 400 | SELF_GUARDIAN | guardian_id = student_id | "Una persona no puede ser su propio acudiente" |
| 401 | UNAUTHORIZED | Token inválido | Redirigir a login |
| 403 | FORBIDDEN | Sin permisos | "No tiene permisos" |
| 404 | NOT_FOUND | Relación no existe | "Relación no encontrada" |
| 404 | GUARDIAN_NOT_FOUND | Usuario guardian no existe | "Acudiente no encontrado" |
| 404 | STUDENT_NOT_FOUND | Usuario student no existe | "Estudiante no encontrado" |
| 409 | DUPLICATE_RELATION | Relación ya existe | "Esta relación ya está registrada" |
| 409 | INVALID_ROLE | Usuario sin rol apropiado | "Usuario debe tener rol guardian/student" |
| 409 | SCHOOL_MISMATCH | Diferentes escuelas | "Guardian y estudiante deben pertenecer a la misma escuela" |
| 500 | INTERNAL_ERROR | Error del servidor | "Error del sistema" |

### Estructura de Error

```typescript
interface GuardianRelationError extends APIError {
  details?: {
    field?: string;
    guardian_id?: string;
    student_id?: string;
    existing_relation_id?: string;  // Si hay duplicado
  };
}

// Ejemplo de error de duplicado
const duplicateError: GuardianRelationError = {
  error: "conflict",
  message: "Ya existe una relación activa entre este acudiente y estudiante con este tipo",
  code: "DUPLICATE_GUARDIAN_RELATION",
  details: {
    guardian_id: "uuid-maria",
    student_id: "uuid-ana",
    existing_relation_id: "uuid-relation-existente"
  }
};
```

## Consideraciones de UX

### Vista de Tarjetas de Acudientes

```typescript
interface GuardianCardProps {
  relation: GuardianRelationResponse;
  onEdit: (relation: GuardianRelationResponse) => void;
  onDelete: (relation: GuardianRelationResponse) => void;
}

const GuardianCard = ({ relation, onEdit, onDelete }: GuardianCardProps) => {
  const icon = getRelationshipIcon(relation.relationship_type);
  const label = getRelationshipLabel(relation.relationship_type);

  return (
    <div className="guardian-card">
      <div className="card-header">
        <h3>{icon} {relation.guardian?.name}</h3>
        <span className="relationship-badge">{label}</span>
      </div>

      <div className="card-body">
        <div className="contact-info">
          <p>📧 {relation.guardian?.email}</p>
          {relation.guardian?.phone && (
            <p>📞 {relation.guardian.phone}</p>
          )}
        </div>

        <div className="meta-info">
          <p><strong>Estado:</strong> {relation.is_active ? 'Activo' : 'Inactivo'}</p>
          <p><strong>Desde:</strong> {formatDate(relation.created_at)}</p>
        </div>
      </div>

      <div className="card-actions">
        <button onClick={() => onEdit(relation)}>✏️ Editar</button>
        <button onClick={() => onDelete(relation)} className="danger">
          🗑️ Eliminar
        </button>
      </div>
    </div>
  );
};

function getRelationshipIcon(type: RelationshipType): string {
  const icons: Record<RelationshipType, string> = {
    father: '👨',
    mother: '👩',
    grandfather: '👴',
    grandmother: '👵',
    uncle: '👨‍🦱',
    aunt: '👩‍🦱',
    other: '👤'
  };
  return icons[type];
}

function getRelationshipLabel(type: RelationshipType): string {
  const labels: Record<RelationshipType, string> = {
    father: 'Padre',
    mother: 'Madre',
    grandfather: 'Abuelo',
    grandmother: 'Abuela',
    uncle: 'Tío',
    aunt: 'Tía',
    other: 'Otro'
  };
  return labels[type];
}
```

### Vista Bidireccional

```typescript
// Vista: Estudiantes de un Guardian
const GuardianStudentsView = ({ guardianId }: { guardianId: string }) => {
  const { relations, loading } = useGuardianRelations(guardianId);

  if (loading) return <Skeleton count={3} />;

  return (
    <div className="students-list">
      <h2>👥 Estudiantes a Cargo</h2>
      {relations.map(relation => (
        <div key={relation.id} className="student-item">
          <div className="student-info">
            <h3>{relation.student?.name}</h3>
            <p className="relationship">
              {getRelationshipLabel(relation.relationship_type)}
            </p>
          </div>
          <div className="student-meta">
            <p>{relation.student?.current_grade}</p>
            <p>Estado: {relation.is_active ? 'Activo' : 'Inactivo'}</p>
          </div>
        </div>
      ))}
    </div>
  );
};
```

### Selector de Tipo de Relación

```typescript
const RelationshipTypeSelector = ({ value, onChange }: {
  value: RelationshipType;
  onChange: (type: RelationshipType) => void;
}) => {
  const options: { value: RelationshipType; label: string; icon: string }[] = [
    { value: 'mother', label: 'Madre', icon: '👩' },
    { value: 'father', label: 'Padre', icon: '👨' },
    { value: 'grandmother', label: 'Abuela', icon: '👵' },
    { value: 'grandfather', label: 'Abuelo', icon: '👴' },
    { value: 'aunt', label: 'Tía', icon: '👩‍🦱' },
    { value: 'uncle', label: 'Tío', icon: '👨‍🦱' },
    { value: 'other', label: 'Otro (Tutor legal, etc.)', icon: '👤' }
  ];

  return (
    <div className="relationship-selector">
      {options.map(option => (
        <button
          key={option.value}
          onClick={() => onChange(option.value)}
          className={value === option.value ? 'selected' : ''}
        >
          <span className="icon">{option.icon}</span>
          <span className="label">{option.label}</span>
        </button>
      ))}
    </div>
  );
};
```

## Almacenamiento Local

### Cache de Relaciones

```typescript
interface RelationsCache {
  entityId: string;  // guardian_id o student_id
  type: 'guardian' | 'student';
  relations: GuardianRelationResponse[];
  timestamp: number;
  ttl: number;  // 5 minutos
}

function cacheRelations(
  entityId: string,
  type: 'guardian' | 'student',
  relations: GuardianRelationResponse[]
) {
  const cache: RelationsCache = {
    entityId,
    type,
    relations,
    timestamp: Date.now(),
    ttl: 5 * 60 * 1000
  };
  localStorage.setItem(`relations_${type}_${entityId}`, JSON.stringify(cache));
}

function getCachedRelations(
  entityId: string,
  type: 'guardian' | 'student'
): GuardianRelationResponse[] | null {
  const cached = localStorage.getItem(`relations_${type}_${entityId}`);
  if (!cached) return null;

  const parsed: RelationsCache = JSON.parse(cached);
  const isExpired = Date.now() - parsed.timestamp > parsed.ttl;

  return isExpired ? null : parsed.relations;
}
```

## Código de Ejemplo (Mobile - TypeScript)

### Service

```typescript
// services/GuardianRelationService.ts
import axios from 'axios';

const API_BASE = process.env.REACT_APP_API_ADMIN_URL || 'http://localhost:8081/v1';

export class GuardianRelationService {

  /**
   * Crear relación guardian-student
   */
  async createRelation(data: CreateGuardianRelationRequest): Promise<GuardianRelationResponse> {
    const response = await axios.post<GuardianRelationResponse>(
      `${API_BASE}/guardian-relations`,
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
   * Obtener relación específica
   */
  async getRelation(id: string): Promise<GuardianRelationResponse> {
    const response = await axios.get<GuardianRelationResponse>(
      `${API_BASE}/guardian-relations/${id}`,
      {
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }
    );
    return response.data;
  }

  /**
   * Actualizar relación (solo tipo)
   */
  async updateRelation(
    id: string,
    data: UpdateGuardianRelationRequest
  ): Promise<GuardianRelationResponse> {
    const response = await axios.put<GuardianRelationResponse>(
      `${API_BASE}/guardian-relations/${id}`,
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
   * Eliminar relación (soft delete)
   */
  async deleteRelation(id: string): Promise<void> {
    await axios.delete(
      `${API_BASE}/guardian-relations/${id}`,
      {
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }
    );
  }

  /**
   * Obtener relaciones de un guardian (sus pupilos)
   */
  async getGuardianRelations(guardianId: string): Promise<GuardianRelationResponse[]> {
    const response = await axios.get<GuardianRelationResponse[]>(
      `${API_BASE}/guardians/${guardianId}/relations`,
      {
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }
    );
    return response.data;
  }

  /**
   * Obtener guardians de un estudiante (sus acudientes)
   */
  async getStudentGuardians(studentId: string): Promise<GuardianRelationResponse[]> {
    const response = await axios.get<GuardianRelationResponse[]>(
      `${API_BASE}/students/${studentId}/guardians`,
      {
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }
    );
    return response.data;
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
// hooks/useGuardianRelations.ts
import { useState, useEffect } from 'react';
import { GuardianRelationService } from '../services/GuardianRelationService';

export function useStudentGuardians(studentId: string) {
  const [relations, setRelations] = useState<GuardianRelationResponse[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<APIError | null>(null);

  const service = new GuardianRelationService();

  const loadRelations = async () => {
    setLoading(true);
    setError(null);
    try {
      const data = await service.getStudentGuardians(studentId);
      setRelations(data);
    } catch (err: any) {
      setError(err.response?.data);
    } finally {
      setLoading(false);
    }
  };

  const createRelation = async (data: CreateGuardianRelationRequest) => {
    setLoading(true);
    setError(null);
    try {
      const newRelation = await service.createRelation(data);
      setRelations(prev => [...prev, newRelation]);
      return newRelation;
    } catch (err: any) {
      setError(err.response?.data);
      throw err;
    } finally {
      setLoading(false);
    }
  };

  const updateRelation = async (id: string, data: UpdateGuardianRelationRequest) => {
    setLoading(true);
    setError(null);
    try {
      const updated = await service.updateRelation(id, data);
      setRelations(prev => prev.map(r => r.id === id ? updated : r));
      return updated;
    } catch (err: any) {
      setError(err.response?.data);
      throw err;
    } finally {
      setLoading(false);
    }
  };

  const deleteRelation = async (id: string) => {
    setLoading(true);
    setError(null);
    try {
      await service.deleteRelation(id);
      setRelations(prev => prev.filter(r => r.id !== id));
    } catch (err: any) {
      setError(err.response?.data);
      throw err;
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    if (studentId) {
      loadRelations();
    }
  }, [studentId]);

  return {
    relations,
    loading,
    error,
    createRelation,
    updateRelation,
    deleteRelation,
    reload: loadRelations
  };
}
```

### Componente de Formulario

```typescript
// components/GuardianRelationForm.tsx
import React from 'react';
import { useForm } from 'react-hook-form';

interface GuardianRelationFormProps {
  studentId: string;
  relation?: GuardianRelationResponse;  // Para edición
  onSubmit: (data: CreateGuardianRelationRequest) => Promise<void>;
  onCancel: () => void;
}

export function GuardianRelationForm({
  studentId,
  relation,
  onSubmit,
  onCancel
}: GuardianRelationFormProps) {
  const { register, handleSubmit, watch, formState: { errors, isSubmitting } } = useForm({
    defaultValues: relation || {
      student_id: studentId,
      relationship_type: 'mother'
    }
  });

  const selectedType = watch('relationship_type');

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {!relation && (
        <div className="form-group">
          <label>Seleccionar Acudiente *</label>
          {/* Componente de búsqueda de usuarios con rol guardian */}
          <UserSelector
            role="guardian"
            onSelect={(user) => setValue('guardian_id', user.id)}
          />
          {errors.guardian_id && <span className="error-msg">Debe seleccionar un acudiente</span>}
        </div>
      )}

      <div className="form-group">
        <label>Tipo de Relación *</label>
        <RelationshipTypeSelector
          value={selectedType}
          onChange={(type) => setValue('relationship_type', type)}
        />
      </div>

      {selectedType === 'other' && (
        <div className="form-group">
          <label>Especificar Relación</label>
          <input
            type="text"
            placeholder="Ej: Tutor legal, Prima, etc."
          />
          <small>Para tipos de relación no listados</small>
        </div>
      )}

      <div className="form-actions">
        <button type="button" onClick={onCancel} disabled={isSubmitting}>
          Cancelar
        </button>
        <button type="submit" disabled={isSubmitting}>
          {isSubmitting ? 'Guardando...' : relation ? 'Actualizar' : 'Crear'}
        </button>
      </div>
    </form>
  );
}
```

## Notas de Implementación

### ⚠️ Estado de Disponibilidad

**CRÍTICO:** Estos endpoints están SOLO en rama `dev`:

```bash
# Verificar rama actual
cd /path/to/edugo-api-administracion
git branch

# Si está en main, estos endpoints NO están disponibles
# Requiere merge dev → main antes de usar en producción
```

### Seguridad

1. **Autorización:**
   - Solo admin/coordinador deberían crear relaciones
   - Guardian puede ver sus propios estudiantes (validar en JWT)
   - Guardian NO puede modificar relaciones

2. **Validación de Escuela:**
   - Backend valida que guardian y student pertenezcan a misma escuela
   - Previene relaciones cross-school

3. **Privacidad:**
   - Información de contacto del guardian es sensible
   - Validar permisos antes de mostrar teléfono/email

### Casos de Uso Comunes

```typescript
// Registrar madre de estudiante
await service.createRelation({
  guardian_id: 'uuid-maria-garcia',
  student_id: 'uuid-ana-lopez',
  relationship_type: 'mother'
});

// Registrar padre del mismo estudiante
await service.createRelation({
  guardian_id: 'uuid-carlos-lopez',
  student_id: 'uuid-ana-lopez',
  relationship_type: 'father'
});

// Cambiar relación de "other" a "uncle"
await service.updateRelation('uuid-relation', {
  relationship_type: 'uncle'
});

// Obtener todos los acudientes de un estudiante
const guardians = await service.getStudentGuardians('uuid-ana-lopez');
// Retorna: [madre, padre]

// Obtener todos los pupilos de un guardian
const students = await service.getGuardianRelations('uuid-maria-garcia');
// Retorna: [Ana López, Juan López]
```

### Reportes Útiles

```typescript
// Estudiantes sin acudientes registrados
async function getStudentsWithoutGuardians(schoolId: string): Promise<UserResponse[]> {
  const students = await userService.listStudents(schoolId);
  const withoutGuardians: UserResponse[] = [];

  for (const student of students) {
    const guardians = await service.getStudentGuardians(student.id);
    if (guardians.length === 0) {
      withoutGuardians.push(student);
    }
  }

  return withoutGuardians;
}

// Acudientes con múltiples pupilos
async function getGuardiansWithMultipleStudents(schoolId: string): Promise<{
  guardian: UserResponse;
  studentCount: number;
}[]> {
  const guardians = await userService.listGuardians(schoolId);
  const result = [];

  for (const guardian of guardians) {
    const students = await service.getGuardianRelations(guardian.id);
    if (students.length > 1) {
      result.push({
        guardian,
        studentCount: students.length
      });
    }
  }

  return result;
}
```

### Integración con Notificaciones

```typescript
// Enviar notificación a acudientes cuando hay evento importante
async function notifyStudentGuardians(
  studentId: string,
  message: string
) {
  const relations = await service.getStudentGuardians(studentId);

  for (const relation of relations) {
    if (relation.is_active && relation.guardian?.email) {
      await emailService.send({
        to: relation.guardian.email,
        subject: `Notificación sobre ${relation.student?.name}`,
        body: message
      });
    }
  }
}

// Uso
await notifyStudentGuardians(
  'uuid-ana-lopez',
  'La estudiante Ana López ha sido destacada en el área de matemáticas'
);
```

### Validación de Rol Guardian

```typescript
// Antes de crear relación, validar que usuario tenga rol guardian
async function validateGuardianRole(userId: string): Promise<boolean> {
  const user = await userService.getUser(userId);

  // Verificar membresías con rol guardian
  const memberships = await membershipService.listMembershipsByUser(userId, true);
  const hasGuardianRole = memberships.some(m => m.role === 'guardian');

  return hasGuardianRole;
}

// Uso en formulario
const handleSubmit = async (data: CreateGuardianRelationRequest) => {
  const isValid = await validateGuardianRole(data.guardian_id);

  if (!isValid) {
    showError('El usuario seleccionado no tiene rol de acudiente');
    return;
  }

  await createRelation(data);
};
```

### Próximos Pasos

1. ⚠️ **URGENTE:** Merge rama dev → main
2. ✅ Implementar portal de acudientes
3. ✅ Notificaciones automáticas a guardians
4. ✅ Exportar lista de contactos de emergencia
5. ✅ Dashboard de comunicación guardian-escuela
6. ✅ Permisos granulares por guardian (qué puede ver)
7. ✅ Histórico de accesos del guardian
8. ✅ App móvil para guardians
