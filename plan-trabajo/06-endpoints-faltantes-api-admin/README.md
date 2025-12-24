# 06 - Endpoints Faltantes API Admin

**Estado:** Pendiente
**Prioridad:** Media
**Tiempo estimado:** 12-15 horas

---

## Resumen

Completar CRUD de Subjects y Guardian Relations en API Administracion.

## Endpoints Faltantes

### Subjects (3 endpoints)

| Endpoint | Metodo | Descripcion | Prioridad |
|----------|--------|-------------|-----------|
| `/v1/subjects/:id` | GET | Obtener materia por ID | Media |
| `/v1/subjects` | GET | Listar materias | Media |
| `/v1/subjects/:id` | DELETE | Eliminar materia | Baja |

### Guardian Relations (2 endpoints)

| Endpoint | Metodo | Descripcion | Prioridad |
|----------|--------|-------------|-----------|
| `/v1/guardian-relations/:id` | PUT | Actualizar relacion | Media |
| `/v1/guardian-relations/:id` | DELETE | Eliminar relacion | Media |

## Estado Actual

**Subjects:**
- POST `/v1/subjects` - Crear materia
- PATCH `/v1/subjects/:id` - Actualizar materia

**Guardian Relations:**
- POST `/v1/guardian-relations` - Crear relacion
- GET `/v1/guardian-relations/:id` - Obtener relacion
- GET `/v1/guardians/:guardian_id/relations` - Relaciones de guardian
- GET `/v1/students/:student_id/guardians` - Guardians de estudiante

## Proyectos Afectados

| Proyecto | Accion | Carpeta |
|----------|--------|---------|
| edugo-api-administracion | Agregar endpoints | [api-admin/](./api-admin/) |

## Orden de Ejecucion

```
1. edugo-api-administracion
   ├── Implementar GET /v1/subjects/:id
   ├── Implementar GET /v1/subjects
   ├── Implementar DELETE /v1/subjects/:id
   ├── Implementar PUT /v1/guardian-relations/:id
   ├── Implementar DELETE /v1/guardian-relations/:id
   ├── Actualizar swagger
   ├── Agregar tests
   ├── PR a dev
   └── Merge a dev
```

---

## Siguiente Paso

Ir a [api-admin/PLAN.md](./api-admin/PLAN.md) para ver el plan detallado.
