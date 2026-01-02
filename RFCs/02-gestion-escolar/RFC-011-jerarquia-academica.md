# RFC-011: Jerarquía Académica (Niveles, Grados, Secciones)

## Metadata
- **ID:** RFC-011
- **Proceso:** Gestión Escolar
- **Subproceso:** Unidades Académicas
- **Prioridad:** Alta
- **Dependencias:** RFC-010 (CRUD Escuelas)
- **Estado API:** ✅ Listo en main

## Descripción

Gestión de la estructura jerárquica académica de una escuela. Permite crear y organizar unidades académicas en forma de árbol (campus → nivel → grado → sección). Utiliza PostgreSQL ltree para manejar jerarquías de forma eficiente, permitiendo consultas de rutas, ancestros y descendientes.

## Flujo de Usuario (UX)

### Ver Árbol Jerárquico (Admin/Teacher)
1. Usuario selecciona una escuela
2. Click en "Estructura Académica"
3. Sistema carga árbol completo desde raíz
4. Muestra estructura expandible/colapsable:
   ```
   🏫 Colegio San José
     📍 Campus Principal
       📚 Primaria
         📖 Grado 1
           👥 Sección A
           👥 Sección B
         📖 Grado 2
           👥 Sección A
       📚 Bachillerato
         📖 Grado 6
           👥 Sección A
           👥 Sección B
           👥 Sección C
   ```
5. Cada nodo muestra: tipo, nombre, código, acciones

### Crear Unidad Académica
1. Admin selecciona nodo padre (o escuela para raíz)
2. Click "Agregar sub-unidad"
3. Formulario muestra:
   - Tipo (campus/nivel/grado/sección)
   - Nombre de visualización (requerido)
   - Código (opcional)
   - Descripción
4. Sistema valida que no cree ciclos
5. Crea unidad → Actualiza árbol automáticamente
6. Muestra nueva unidad en su posición jerárquica

### Editar Unidad
1. Admin click en nodo
2. Muestra detalle con opción "Editar"
3. Formulario pre-llenado
4. Permite cambiar: nombre, código, descripción, metadata
5. NO permite cambiar tipo ni padre (para evitar inconsistencias)
6. Actualiza → Refresca árbol

### Eliminar Unidad (Soft Delete)
1. Admin click en nodo
2. Click "Eliminar"
3. Confirmación modal:
   - "¿Eliminar [nombre]?"
   - Si tiene hijos: "Esto también marcará como inactivas las [N] sub-unidades"
4. Confirma → Soft delete (deleted_at)
5. Desaparece del árbol (con opción de ver eliminadas)

### Restaurar Unidad Eliminada
1. Admin activa "Ver eliminadas"
2. Unidades eliminadas aparecen en gris/tachado
3. Click "Restaurar"
4. Confirmación
5. Restaura → Vuelve a estado activo

### Ver Ruta Jerárquica
1. Usuario selecciona cualquier unidad
2. Sistema muestra breadcrumb:
   ```
   Colegio San José > Campus Principal > Bachillerato > Grado 6 > Sección A
   ```
3. Cada nivel es clickeable para navegar

### Filtrar por Tipo
1. Admin selecciona "Filtrar por tipo"
2. Opciones: campus, nivel, grado, sección
3. Lista solo unidades de ese tipo
4. Muestra jerarquía completa en vista de detalle

## Flujo de Datos (Técnico)

### Endpoints Involucrados

| Endpoint | Método | Descripción | Estado |
|----------|--------|-------------|--------|
| `/v1/schools/:id/units` | POST | Crear unidad académica | ✅ |
| `/v1/schools/:id/units` | GET | Listar unidades de escuela | ⚠️ Sin paginación |
| `/v1/schools/:id/units/tree` | GET | Árbol jerárquico completo | ✅ |
| `/v1/schools/:id/units/by-type` | GET | Filtrar por tipo | ✅ |
| `/v1/units/:id` | GET | Obtener unidad específica | ✅ |
| `/v1/units/:id` | PUT | Actualizar unidad | ✅ |
| `/v1/units/:id` | DELETE | Soft delete | ✅ |
| `/v1/units/:id/restore` | POST | Restaurar eliminada | ✅ |
| `/v1/units/:id/hierarchy-path` | GET | Ruta ltree completa | ✅ |

### Request/Response (TypeScript interfaces)

```typescript
// ============= CREATE UNIT =============
interface CreateAcademicUnitRequest {
  parent_unit_id?: string;         // UUID, null para unidades raíz
  type: UnitType;                  // Requerido
  display_name: string;            // min=3, max=255, requerido
  code?: string;                   // min=2, max=50, opcional
  description?: string;
  metadata?: Record<string, any>;
}

type UnitType =
  | "campus"        // Sede física
  | "level"         // Nivel educativo (primaria, bachillerato)
  | "grade"         // Grado (1°, 2°, 3°, etc.)
  | "section";      // Sección/Grupo (A, B, C)

// Ejemplo
const createRequest: CreateAcademicUnitRequest = {
  parent_unit_id: "uuid-del-grado-6",
  type: "section",
  display_name: "Sección A",
  code: "6A",
  description: "Grupo A de sexto grado - 30 estudiantes",
  metadata: {
    capacity: 30,
    classroom: "Aula 201",
    director: "Prof. Juan Pérez"
  }
};

// ============= UNIT RESPONSE =============
interface AcademicUnitResponse {
  id: string;                      // UUID
  parent_unit_id: string | null;   // UUID o null si es raíz
  school_id: string;               // UUID
  type: UnitType;
  display_name: string;
  code: string;
  description: string;
  metadata: Record<string, any>;
  created_at: string;              // ISO8601
  updated_at: string;              // ISO8601
  deleted_at: string | null;       // ISO8601 o null si activa
}

// ============= TREE NODE (jerarquía) =============
interface UnitTreeNode {
  id: string;
  type: UnitType;
  display_name: string;
  code: string;
  depth: number;                   // Profundidad en árbol (0 = raíz)
  children: UnitTreeNode[];        // Recursivo
}

// Ejemplo de árbol
const tree: UnitTreeNode[] = [
  {
    id: "uuid-1",
    type: "campus",
    display_name: "Campus Principal",
    code: "CP",
    depth: 0,
    children: [
      {
        id: "uuid-2",
        type: "level",
        display_name: "Primaria",
        code: "PRI",
        depth: 1,
        children: [
          {
            id: "uuid-3",
            type: "grade",
            display_name: "Grado 1",
            code: "G1",
            depth: 2,
            children: [
              {
                id: "uuid-4",
                type: "section",
                display_name: "Sección A",
                code: "1A",
                depth: 3,
                children: []
              }
            ]
          }
        ]
      }
    ]
  }
];

// ============= HIERARCHY PATH =============
interface HierarchyPathResponse {
  path: AcademicUnitResponse[];    // Desde raíz hasta unidad
  ltree_path: string;              // Ej: "CP.PRI.G1.1A"
}

// ============= UPDATE UNIT =============
interface UpdateAcademicUnitRequest {
  display_name?: string;           // min=3, max=255
  code?: string;                   // min=2, max=50
  description?: string;
  metadata?: Record<string, any>;
  // NO permite cambiar: parent_unit_id, type, school_id
}

// ============= LIST PARAMS =============
interface ListUnitsParams {
  includeDeleted?: boolean;        // Default: false
}

interface FilterByTypeParams {
  type: UnitType;                  // Requerido
}
```

### Validaciones del Backend

```typescript
const validations = {
  parent_unit_id: {
    uuid: true,
    exists: true,              // Debe existir en BD
    sameSchool: true,          // Debe pertenecer a misma escuela
    noCycles: true             // No crear ciclos en jerarquía
  },
  type: {
    required: true,
    enum: ["campus", "level", "grade", "section"]
  },
  display_name: {
    required: true,
    minLength: 3,
    maxLength: 255
  },
  code: {
    minLength: 2,
    maxLength: 50,
    optional: true,
    unique: false              // Puede repetirse entre escuelas
  }
};
```

### Lógica de ltree (PostgreSQL)

```sql
-- Estructura de tabla
CREATE TABLE academic_units (
  id UUID PRIMARY KEY,
  parent_unit_id UUID REFERENCES academic_units(id),
  school_id UUID NOT NULL,
  type VARCHAR(20) NOT NULL,
  display_name VARCHAR(255) NOT NULL,
  code VARCHAR(50),
  path LTREE NOT NULL,        -- Ruta jerárquica: 'campus1.level2.grade3'
  depth INTEGER NOT NULL,     -- Profundidad: 0, 1, 2, 3...
  created_at TIMESTAMP,
  deleted_at TIMESTAMP
);

-- Índice para búsquedas jerárquicas
CREATE INDEX academic_units_path_idx ON academic_units USING GIST (path);

-- Consultas ltree
-- Encontrar todos los descendientes de una unidad
SELECT * FROM academic_units
WHERE path <@ 'campus1.level2'::ltree AND deleted_at IS NULL;

-- Encontrar ruta completa de una unidad
SELECT * FROM academic_units
WHERE id = 'uuid-sección-a';
-- Resultado: path = 'campus1.level2.grade3.section4'
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
┌──────────┐     RESTORE     ┌──────────┐
│ELIMINADA │◄────────────────│ELIMINADA │
│(deleted) │                 │(deleted) │
└──────────┘                 └──────────┘
     │
     │ RESTORE
     ▼
┌─────────┐
│ ACTIVA  │
│(restored)│
└─────────┘

Estados:
- ACTIVA: deleted_at = NULL (aparece en listados normales)
- ELIMINADA: deleted_at != NULL (solo aparece con includeDeleted=true)

Restricciones:
- Al eliminar unidad con hijos, se eliminan todos los descendientes
- Al restaurar, se debe restaurar toda la rama desde el padre
```

## Manejo de Errores

| Código HTTP | Código Error | Significado | Acción en UI |
|-------------|--------------|-------------|--------------|
| 400 | INVALID_REQUEST | Datos inválidos | Mostrar errores por campo |
| 400 | CYCLE_DETECTED | Intentar crear ciclo en jerarquía | "No se puede crear jerarquía circular" |
| 401 | UNAUTHORIZED | Token inválido | Redirigir a login |
| 403 | FORBIDDEN | Sin permisos | "No tiene permisos para modificar estructura" |
| 404 | NOT_FOUND | Unidad no existe | "Unidad no encontrada" |
| 404 | PARENT_NOT_FOUND | Padre no existe | "Unidad padre no encontrada" |
| 409 | DIFFERENT_SCHOOL | Padre de otra escuela | "El padre debe pertenecer a la misma escuela" |
| 500 | INTERNAL_ERROR | Error del servidor | "Error del sistema" |

### Estructura de Error

```typescript
interface HierarchyError extends APIError {
  details?: {
    field?: string;
    parent_id?: string;
    detected_cycle?: string[];  // IDs que forman el ciclo
  };
}

// Ejemplo de error de ciclo
const cycleError: HierarchyError = {
  error: "bad_request",
  message: "No se puede crear una jerarquía circular",
  code: "CYCLE_DETECTED",
  details: {
    detected_cycle: ["uuid-1", "uuid-2", "uuid-3", "uuid-1"]
  }
};
```

## Consideraciones de UX

### Visualización del Árbol

```typescript
// Componente de árbol colapsable
interface TreeNodeProps {
  node: UnitTreeNode;
  level: number;
  onSelect: (node: UnitTreeNode) => void;
  onEdit: (node: UnitTreeNode) => void;
  onDelete: (node: UnitTreeNode) => void;
  onAddChild: (parentNode: UnitTreeNode) => void;
}

const TreeNode = ({ node, level, ...handlers }: TreeNodeProps) => {
  const [isExpanded, setIsExpanded] = useState(true);
  const hasChildren = node.children.length > 0;

  const icon = getIconForType(node.type);
  const indent = level * 20; // px

  return (
    <div className="tree-node" style={{ paddingLeft: `${indent}px` }}>
      <div className="node-header">
        {hasChildren && (
          <button onClick={() => setIsExpanded(!isExpanded)}>
            {isExpanded ? '▼' : '▶'}
          </button>
        )}
        <span className="node-icon">{icon}</span>
        <span className="node-name">{node.display_name}</span>
        <span className="node-code">({node.code})</span>
        <div className="node-actions">
          <button onClick={() => handlers.onAddChild(node)}>+</button>
          <button onClick={() => handlers.onEdit(node)}>✏️</button>
          <button onClick={() => handlers.onDelete(node)}>🗑️</button>
        </div>
      </div>

      {isExpanded && hasChildren && (
        <div className="node-children">
          {node.children.map(child => (
            <TreeNode
              key={child.id}
              node={child}
              level={level + 1}
              {...handlers}
            />
          ))}
        </div>
      )}
    </div>
  );
};

function getIconForType(type: UnitType): string {
  const icons: Record<UnitType, string> = {
    campus: '🏫',
    level: '📚',
    grade: '📖',
    section: '👥'
  };
  return icons[type];
}
```

### Breadcrumb de Navegación

```typescript
// Componente de ruta jerárquica
interface BreadcrumbProps {
  path: AcademicUnitResponse[];
  onNavigate: (unit: AcademicUnitResponse) => void;
}

const HierarchyBreadcrumb = ({ path, onNavigate }: BreadcrumbProps) => (
  <nav className="breadcrumb">
    {path.map((unit, index) => (
      <React.Fragment key={unit.id}>
        <button
          onClick={() => onNavigate(unit)}
          className="breadcrumb-item"
        >
          {unit.display_name}
        </button>
        {index < path.length - 1 && <span className="separator">›</span>}
      </React.Fragment>
    ))}
  </nav>
);
```

### Drag & Drop (Futuro)

```typescript
// Propuesta para reorganizar jerarquía
interface DragDropUnitProps {
  node: UnitTreeNode;
  onMove: (unitId: string, newParentId: string) => Promise<void>;
}

// Validaciones antes de permitir drop:
// 1. No mover a descendiente (crearía ciclo)
// 2. Validar que tipo sea compatible con nuevo padre
// 3. Misma escuela
```

## Almacenamiento Local

### Cache de Árbol

```typescript
// Cachear árbol completo (regenera rápido)
interface CachedTree {
  schoolId: string;
  tree: UnitTreeNode[];
  timestamp: number;
  ttl: number;  // 5 minutos
}

function cacheTree(schoolId: string, tree: UnitTreeNode[]) {
  const cache: CachedTree = {
    schoolId,
    tree,
    timestamp: Date.now(),
    ttl: 5 * 60 * 1000
  };
  localStorage.setItem(`tree_${schoolId}`, JSON.stringify(cache));
}

function getCachedTree(schoolId: string): UnitTreeNode[] | null {
  const cached = localStorage.getItem(`tree_${schoolId}`);
  if (!cached) return null;

  const parsed: CachedTree = JSON.parse(cached);
  const isExpired = Date.now() - parsed.timestamp > parsed.ttl;

  return isExpired ? null : parsed.tree;
}
```

## Código de Ejemplo (Mobile - TypeScript)

### Service

```typescript
// services/AcademicUnitService.ts
import axios from 'axios';

const API_BASE = process.env.REACT_APP_API_ADMIN_URL || 'http://localhost:8081/v1';

export class AcademicUnitService {

  /**
   * Obtener árbol jerárquico completo de una escuela
   * 🌳 Feature principal para visualización
   */
  async getUnitTree(schoolId: string): Promise<UnitTreeNode[]> {
    const response = await axios.get<UnitTreeNode[]>(
      `${API_BASE}/schools/${schoolId}/units/tree`,
      {
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }
    );
    return response.data;
  }

  /**
   * Listar todas las unidades de una escuela (flat)
   */
  async listUnits(
    schoolId: string,
    includeDeleted: boolean = false
  ): Promise<AcademicUnitResponse[]> {
    const response = await axios.get<AcademicUnitResponse[]>(
      `${API_BASE}/schools/${schoolId}/units`,
      {
        params: { includeDeleted },
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }
    );
    return response.data;
  }

  /**
   * Filtrar unidades por tipo
   */
  async listUnitsByType(
    schoolId: string,
    type: UnitType
  ): Promise<AcademicUnitResponse[]> {
    const response = await axios.get<AcademicUnitResponse[]>(
      `${API_BASE}/schools/${schoolId}/units/by-type`,
      {
        params: { type },
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }
    );
    return response.data;
  }

  /**
   * Obtener unidad específica
   */
  async getUnit(id: string): Promise<AcademicUnitResponse> {
    const response = await axios.get<AcademicUnitResponse>(
      `${API_BASE}/units/${id}`,
      {
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }
    );
    return response.data;
  }

  /**
   * Obtener ruta jerárquica completa de una unidad
   * Útil para breadcrumbs
   */
  async getHierarchyPath(id: string): Promise<HierarchyPathResponse> {
    const response = await axios.get<HierarchyPathResponse>(
      `${API_BASE}/units/${id}/hierarchy-path`,
      {
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }
    );
    return response.data;
  }

  /**
   * Crear nueva unidad académica
   */
  async createUnit(
    schoolId: string,
    data: CreateAcademicUnitRequest
  ): Promise<AcademicUnitResponse> {
    const response = await axios.post<AcademicUnitResponse>(
      `${API_BASE}/schools/${schoolId}/units`,
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
   * Actualizar unidad
   */
  async updateUnit(
    id: string,
    data: UpdateAcademicUnitRequest
  ): Promise<AcademicUnitResponse> {
    const response = await axios.put<AcademicUnitResponse>(
      `${API_BASE}/units/${id}`,
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
   * Eliminar unidad (soft delete)
   */
  async deleteUnit(id: string): Promise<void> {
    await axios.delete(
      `${API_BASE}/units/${id}`,
      {
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }
    );
  }

  /**
   * Restaurar unidad eliminada
   */
  async restoreUnit(id: string): Promise<void> {
    await axios.post(
      `${API_BASE}/units/${id}/restore`,
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
// hooks/useAcademicUnits.ts
import { useState, useEffect } from 'react';
import { AcademicUnitService } from '../services/AcademicUnitService';

export function useAcademicUnits(schoolId: string) {
  const [tree, setTree] = useState<UnitTreeNode[]>([]);
  const [units, setUnits] = useState<AcademicUnitResponse[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<APIError | null>(null);

  const service = new AcademicUnitService();

  const loadTree = async () => {
    setLoading(true);
    setError(null);
    try {
      const data = await service.getUnitTree(schoolId);
      setTree(data);
    } catch (err: any) {
      setError(err.response?.data);
    } finally {
      setLoading(false);
    }
  };

  const createUnit = async (data: CreateAcademicUnitRequest) => {
    setLoading(true);
    setError(null);
    try {
      const newUnit = await service.createUnit(schoolId, data);
      await loadTree(); // Recargar árbol completo
      return newUnit;
    } catch (err: any) {
      setError(err.response?.data);
      throw err;
    } finally {
      setLoading(false);
    }
  };

  const updateUnit = async (id: string, data: UpdateAcademicUnitRequest) => {
    setLoading(true);
    setError(null);
    try {
      const updated = await service.updateUnit(id, data);
      await loadTree();
      return updated;
    } catch (err: any) {
      setError(err.response?.data);
      throw err;
    } finally {
      setLoading(false);
    }
  };

  const deleteUnit = async (id: string) => {
    setLoading(true);
    setError(null);
    try {
      await service.deleteUnit(id);
      await loadTree();
    } catch (err: any) {
      setError(err.response?.data);
      throw err;
    } finally {
      setLoading(false);
    }
  };

  const restoreUnit = async (id: string) => {
    setLoading(true);
    setError(null);
    try {
      await service.restoreUnit(id);
      await loadTree();
    } catch (err: any) {
      setError(err.response?.data);
      throw err;
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    if (schoolId) {
      loadTree();
    }
  }, [schoolId]);

  return {
    tree,
    units,
    loading,
    error,
    createUnit,
    updateUnit,
    deleteUnit,
    restoreUnit,
    reload: loadTree
  };
}
```

### Componente de Formulario

```typescript
// components/AcademicUnitForm.tsx
import React from 'react';
import { useForm } from 'react-hook-form';

interface AcademicUnitFormProps {
  parentUnit?: AcademicUnitResponse;  // Para contexto
  unit?: AcademicUnitResponse;        // Para edición
  onSubmit: (data: CreateAcademicUnitRequest) => Promise<void>;
  onCancel: () => void;
}

export function AcademicUnitForm({ parentUnit, unit, onSubmit, onCancel }: AcademicUnitFormProps) {
  const { register, handleSubmit, watch, formState: { errors, isSubmitting } } = useForm({
    defaultValues: unit || {
      parent_unit_id: parentUnit?.id || null
    }
  });

  const selectedType = watch('type');

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {parentUnit && (
        <div className="info-box">
          <strong>Padre:</strong> {parentUnit.display_name} ({parentUnit.type})
        </div>
      )}

      <div className="form-group">
        <label>Tipo *</label>
        <select
          {...register('type', { required: 'El tipo es obligatorio' })}
          disabled={!!unit}  // No permitir cambiar tipo al editar
          className={errors.type ? 'error' : ''}
        >
          <option value="">Seleccione...</option>
          <option value="campus">🏫 Campus/Sede</option>
          <option value="level">📚 Nivel Educativo</option>
          <option value="grade">📖 Grado</option>
          <option value="section">👥 Sección</option>
        </select>
        {errors.type && <span className="error-msg">{errors.type.message}</span>}
      </div>

      <div className="form-group">
        <label>Nombre de Visualización *</label>
        <input
          {...register('display_name', {
            required: 'El nombre es obligatorio',
            minLength: { value: 3, message: 'Mínimo 3 caracteres' },
            maxLength: { value: 255, message: 'Máximo 255 caracteres' }
          })}
          placeholder={getPlaceholderForType(selectedType)}
          className={errors.display_name ? 'error' : ''}
        />
        {errors.display_name && <span className="error-msg">{errors.display_name.message}</span>}
      </div>

      <div className="form-group">
        <label>Código</label>
        <input
          {...register('code', {
            minLength: { value: 2, message: 'Mínimo 2 caracteres' },
            maxLength: { value: 50, message: 'Máximo 50 caracteres' }
          })}
          placeholder="Ej: CP, PRI, G1, 1A"
          className={errors.code ? 'error' : ''}
        />
        {errors.code && <span className="error-msg">{errors.code.message}</span>}
      </div>

      <div className="form-group">
        <label>Descripción</label>
        <textarea
          {...register('description')}
          rows={3}
          placeholder="Información adicional sobre esta unidad..."
        />
      </div>

      <div className="form-actions">
        <button type="button" onClick={onCancel} disabled={isSubmitting}>
          Cancelar
        </button>
        <button type="submit" disabled={isSubmitting}>
          {isSubmitting ? 'Guardando...' : unit ? 'Actualizar' : 'Crear'}
        </button>
      </div>
    </form>
  );
}

function getPlaceholderForType(type: UnitType | undefined): string {
  const placeholders: Record<UnitType, string> = {
    campus: 'Ej: Campus Principal',
    level: 'Ej: Primaria',
    grade: 'Ej: Grado 1',
    section: 'Ej: Sección A'
  };
  return type ? placeholders[type] : 'Nombre de la unidad';
}
```

## Notas de Implementación

### PostgreSQL ltree

El backend usa la extensión ltree de PostgreSQL para manejar jerarquías eficientemente:

```sql
-- Ventajas de ltree:
-- 1. Consultas de ancestros/descendientes en O(log n)
-- 2. Índices GIST para búsquedas rápidas
-- 3. Operadores especializados (@>, <@, ~, etc.)
-- 4. Validación automática de ciclos

-- Ejemplo de path: 'campus1.level2.grade3.section4'
```

### Validación de Ciclos

```typescript
// El backend valida automáticamente que no se creen ciclos
// Frontend debe manejar el error apropiadamente

async function handleCreateUnit(data: CreateAcademicUnitRequest) {
  try {
    await service.createUnit(schoolId, data);
  } catch (error: any) {
    if (error.response?.data?.code === 'CYCLE_DETECTED') {
      showError('No se puede crear una jerarquía circular');
    } else {
      showError(error.response?.data?.message || 'Error al crear unidad');
    }
  }
}
```

### Performance

1. **Árbol completo:**
   - Para 50-100 unidades: excelente performance
   - Para >500 unidades: considerar lazy loading por rama
   - ltree permite consultas eficientes incluso con miles de nodos

2. **Cache:**
   - Cachear árbol completo en localStorage (5 min TTL)
   - Invalidar cache al crear/actualizar/eliminar
   - Útil para navegación rápida

### Metadata Extensible

```typescript
// Ejemplo de metadata por tipo
const campusMetadata = {
  address: "Calle 123 #45-67",
  phone: "+57 1 234 5678",
  principal: "María García",
  facilities: ["Biblioteca", "Laboratorio", "Cancha"]
};

const sectionMetadata = {
  capacity: 30,
  classroom: "Aula 201",
  director: "Prof. Juan Pérez",
  schedule: {
    start: "07:00",
    end: "13:00"
  }
};
```

### Próximos Pasos

1. ✅ Implementar visualización de árbol
2. ✅ Agregar drag & drop para reorganizar
3. ✅ Implementar búsqueda en árbol
4. ✅ Exportar estructura a Excel/PDF
5. ✅ Importar estructura desde Excel (bulk creation)
6. ✅ Duplicar estructura entre escuelas (template)
7. ✅ Estadísticas por rama del árbol
