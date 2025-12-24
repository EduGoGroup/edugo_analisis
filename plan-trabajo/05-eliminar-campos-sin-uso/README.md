# 05 - Eliminar Campos Sin Uso

**Estado:** Pendiente
**Prioridad:** Media
**Tiempo estimado:** 4-6 horas

---

## Resumen

Eliminar campos de base de datos que fueron creados pero nunca implementados en el codigo.

## Campos a Eliminar

| Campo | Tabla | BD | Razon |
|-------|-------|-----|-------|
| `email_verified` | users | PostgreSQL | Sin logica de verificacion |
| `last_login_at` | users | PostgreSQL | Sin registro de login |
| `preferences` | users | PostgreSQL | Sin UI de preferencias |

## Decision Arquitectonica

Estos campos representan funcionalidades planificadas pero no implementadas. Opciones:

1. **ELIMINAR (recomendado para MVP):** Limpiar BD y reducir complejidad
2. **IMPLEMENTAR:** Agregar funcionalidad completa (no recomendado ahora)

## Proyectos Afectados

| Proyecto | Accion | Carpeta |
|----------|--------|---------|
| edugo-infrastructure | Crear migracion de eliminacion | [infra/](./infra/) |
| edugo-api-administracion | Eliminar referencias en entities | [api-admin/](./api-admin/) |

## Orden de Ejecucion

```
1. edugo-infrastructure (PRIMERO)
   ├── Crear migracion para eliminar campos
   ├── Actualizar entidad User
   ├── PR a dev -> Merge
   ├── PR de dev a main -> Merge
   └── GitHub Release: postgres/v0.14.1

2. edugo-api-administracion (DESPUES)
   ├── Actualizar go.mod con nueva version de infra
   ├── Eliminar referencias a campos en repositorios
   ├── Actualizar tests
   ├── PR a dev
   └── Merge a dev
```

## Riesgo

**BAJO** - Campos sin logica dependiente

---

## Siguiente Paso

Ir a [infra/PLAN.md](./infra/PLAN.md) para ver el plan detallado.
