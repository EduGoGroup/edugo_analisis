# RFCs de Gestión Escolar

Este directorio contiene los RFCs (Request for Comments) para el módulo de Gestión Escolar de EduGo Mobile.

## RFCs Generados

### RFC-010: CRUD de Escuelas
- **Archivo:** `RFC-010-crud-escuelas.md`
- **Líneas:** 788
- **Estado API:** ✅ Listo en main
- **Prioridad:** Alta
- **Descripción:** CRUD completo de escuelas con gestión de datos institucionales, límites de usuarios y configuración de subscripción.

**Endpoints principales:**
- `POST /v1/schools` - Crear escuela
- `GET /v1/schools` - Listar escuelas
- `GET /v1/schools/:id` - Obtener escuela
- `GET /v1/schools/code/:code` - Buscar por código
- `PUT /v1/schools/:id` - Actualizar escuela
- `DELETE /v1/schools/:id` - Soft delete

---

### RFC-011: Jerarquía Académica
- **Archivo:** `RFC-011-jerarquia-academica.md`
- **Líneas:** 960
- **Estado API:** ✅ Listo en main
- **Prioridad:** Alta
- **Descripción:** Gestión de estructura jerárquica (campus → nivel → grado → sección) usando PostgreSQL ltree.

**Endpoints principales:**
- `POST /v1/schools/:id/units` - Crear unidad
- `GET /v1/schools/:id/units` - Listar unidades
- `GET /v1/schools/:id/units/tree` - Árbol jerárquico completo
- `GET /v1/schools/:id/units/by-type` - Filtrar por tipo
- `GET /v1/units/:id` - Obtener unidad
- `PUT /v1/units/:id` - Actualizar unidad
- `DELETE /v1/units/:id` - Soft delete
- `POST /v1/units/:id/restore` - Restaurar eliminada
- `GET /v1/units/:id/hierarchy-path` - Ruta ltree

**Características destacadas:**
- Visualización en árbol interactivo
- Soporte para ltree (PostgreSQL)
- Validación automática de ciclos
- Restauración de unidades eliminadas

---

### RFC-012: Membresías
- **Archivo:** `RFC-012-membresias.md`
- **Líneas:** 930
- **Estado API:** ✅ Listo en main
- **Prioridad:** Alta
- **Descripción:** Asignación de usuarios (teachers, students, assistants) a unidades académicas con roles y fechas de validez.

**Endpoints principales:**
- `POST /v1/memberships` - Crear membresía
- `GET /v1/memberships` - Listar por unidad
- `GET /v1/memberships/by-role` - Filtrar por rol
- `GET /v1/memberships/:id` - Obtener membresía
- `PUT /v1/memberships/:id` - Actualizar
- `DELETE /v1/memberships/:id` - Hard delete
- `POST /v1/memberships/:id/expire` - Expirar
- `GET /v1/users/:userId/memberships` - Membresías de usuario

**Roles disponibles:**
- `owner` - Dueño/Director
- `teacher` - Docente titular
- `assistant` - Asistente
- `student` - Estudiante
- `guardian` - Acudiente

---

### RFC-013: Relaciones Acudientes
- **Archivo:** `RFC-013-relaciones-acudientes.md`
- **Líneas:** 914
- **Estado API:** ⚠️ Solo en rama dev
- **Prioridad:** Media
- **Descripción:** Gestión de relaciones guardian-student con tipos de parentesco.

**⚠️ ADVERTENCIA:** Endpoints solo disponibles en rama `dev`. Requiere merge dev→main.

**Endpoints principales:**
- `POST /v1/guardian-relations` - Crear relación
- `GET /v1/guardian-relations/:id` - Obtener relación
- `PUT /v1/guardian-relations/:id` - Actualizar
- `DELETE /v1/guardian-relations/:id` - Soft delete
- `GET /v1/guardians/:guardian_id/relations` - Estudiantes de guardian
- `GET /v1/students/:student_id/guardians` - Guardians de estudiante

**Tipos de relación:**
- `father` - Padre
- `mother` - Madre
- `grandfather` - Abuelo
- `grandmother` - Abuela
- `uncle` - Tío
- `aunt` - Tía
- `other` - Otro (tutor legal)

---

## Resumen de Estadísticas

| RFC | Líneas | Endpoints | Estado |
|-----|--------|-----------|--------|
| RFC-010 | 788 | 6 | ✅ Listo |
| RFC-011 | 960 | 9 | ✅ Listo |
| RFC-012 | 930 | 8 | ✅ Listo |
| RFC-013 | 914 | 6 | ⚠️ Solo dev |
| **TOTAL** | **3,592** | **29** | **24 listos, 6 en dev** |

## Dependencias entre RFCs

```
RFC-010 (Escuelas)
    ↓
RFC-011 (Jerarquía) ← Depende de RFC-010
    ↓
RFC-012 (Membresías) ← Depende de RFC-010 y RFC-011
    ↓
RFC-013 (Guardians) ← Depende de RFC-010
```

## Flujo de Implementación Recomendado

### Fase 1: Fundamentos (Semana 1-2)
1. ✅ RFC-010: CRUD Escuelas
   - Crear/editar/eliminar escuelas
   - Buscar por código
   - Validaciones de formulario

### Fase 2: Estructura Académica (Semana 2-3)
2. ✅ RFC-011: Jerarquía Académica
   - Visualización de árbol
   - CRUD de unidades
   - Navegación por niveles
   - Filtros por tipo

### Fase 3: Asignaciones (Semana 3-4)
3. ✅ RFC-012: Membresías
   - Asignar usuarios a unidades
   - Gestión de roles
   - Filtros por rol
   - Vista por usuario

### Fase 4: Relaciones Familiares (Semana 4-5)
4. ⚠️ RFC-013: Guardians (esperar merge dev→main)
   - Vincular acudientes-estudiantes
   - Vista bidireccional
   - Contactos de emergencia

## Bloqueantes Críticos

### 🔴 CRÍTICO - API Admin
1. **Configurar CORS en main.go**
   - Impacto: Frontend no podrá consumir API
   - Tiempo: 1 hora
   - Responsable: Backend Team

2. **Merge rama dev → main**
   - Impacto: 6 endpoints de guardians NO disponibles
   - Tiempo: 1 día (con testing)
   - Responsable: DevOps + Backend Team

### ⚠️ MEDIO - Mejoras
3. **Implementar RBAC**
   - Impacto: Cualquier usuario autenticado puede modificar
   - Tiempo: 1 semana
   - Recomendación: Alta prioridad

4. **Agregar paginación**
   - Impacto: Listas grandes de escuelas/unidades/membresías
   - Tiempo: 3 días
   - Recomendación: Media prioridad

## Características Comunes en Todos los RFCs

### Estructura de Cada RFC
- ✅ Metadata (ID, prioridad, dependencias, estado)
- ✅ Descripción del proceso
- ✅ Flujo de usuario (UX)
- ✅ Flujo de datos (técnico)
- ✅ Endpoints involucrados
- ✅ Request/Response TypeScript interfaces
- ✅ Estados y transiciones
- ✅ Manejo de errores
- ✅ Consideraciones de UX
- ✅ Almacenamiento local
- ✅ Código de ejemplo completo
- ✅ Notas de implementación

### Código de Ejemplo Incluido
Cada RFC incluye:
- **Service completo:** Clase TypeScript con todos los métodos
- **Hook de React:** Custom hook con estado y operaciones CRUD
- **Componente de Formulario:** Formulario con validaciones React Hook Form
- **Componentes de UI:** Tablas, tarjetas, selectores
- **Validaciones:** Frontend y backend
- **Manejo de errores:** Códigos HTTP y mensajes amigables
- **Cache:** Estrategias de localStorage

### Tecnologías Utilizadas
- **Backend:** Go 1.25, Gin, PostgreSQL, edugo-shared
- **Frontend (ejemplos):** TypeScript, React, React Hook Form, Axios
- **Base de datos:** PostgreSQL (con ltree para jerarquías)
- **Autenticación:** JWT Bearer Token (centralizado en API Admin)

## Notas Importantes

### Seguridad
- ⚠️ **Sin RBAC:** Actualmente cualquier usuario autenticado puede modificar todo
- ✅ **JWT centralizado:** Mismo token para API Admin y API Mobile
- ✅ **Validación de escuela:** Backend valida que operaciones sean intra-escuela

### Performance
- ⚠️ **Sin paginación:** GET retorna todos los registros
- ✅ **ltree eficiente:** Consultas jerárquicas optimizadas en PostgreSQL
- ✅ **Cache recomendado:** TTL de 2-5 minutos en localStorage

### Validaciones
- ✅ **Backend completo:** Validaciones de negocio en API
- ✅ **Frontend proactivo:** Validaciones en formularios para UX
- ✅ **Mensajes amigables:** Errores estructurados con códigos

## Recursos Adicionales

### Documentación de Referencia
- **Análisis API Admin:** `/analisis-frontend-readiness/informes-agentes/INFORME_API_ADMIN.md`
- **Endpoints Viables:** `/analisis-frontend-readiness/ENDPOINTS_VIABLES_FRONTEND.md`
- **Consolidado Ecosistema:** `/analisis-frontend-readiness/CONSOLIDADO_ECOSISTEMA.md`

### Swagger UIs
- **API Admin:** http://localhost:8081/swagger/index.html
- **API Mobile:** http://localhost:8080/swagger/index.html

### Health Checks
- **API Admin:** http://localhost:8081/health
- **API Mobile:** http://localhost:8080/health?detail=1

## Próximos Pasos

1. ✅ Revisar RFCs con equipo de frontend
2. ⚠️ Resolver bloqueantes críticos (CORS, merge dev→main)
3. ✅ Comenzar implementación Fase 1 (RFC-010)
4. ✅ Crear componentes de UI base
5. ✅ Configurar routing y navegación
6. ✅ Implementar autenticación
7. ✅ Tests unitarios de componentes
8. ✅ Tests de integración con API

---

**Generado:** 2025-12-24
**Analista:** Claude Sonnet 4.5
**Total de líneas de código de ejemplo:** 3,592
