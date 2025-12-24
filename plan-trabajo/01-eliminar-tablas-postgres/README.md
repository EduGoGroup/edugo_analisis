# 01 - Eliminar Tablas PostgreSQL Sin Uso

**Estado:** Pendiente
**Prioridad:** Alta
**Tiempo estimado:** 4-6 horas

---

## Resumen

Eliminar 5 tablas de PostgreSQL que fueron creadas pero nunca utilizadas en el codigo.

## Tablas a Eliminar

| Tabla | Migracion | Razon |
|-------|-----------|-------|
| `user_active_context` | 011 | Fase 1 UI nunca implementada |
| `user_favorites` | 012 | Funcionalidad de favoritos no existe |
| `user_activity_log` | 013 | Analytics UI no implementado |
| `feature_flags` | 014 | Deuda tecnica Apple App |
| `feature_flag_overrides` | 015 | Phase 2 nunca implementado |

## Proyectos Afectados

| Proyecto | Accion | Carpeta |
|----------|--------|---------|
| edugo-infrastructure | Crear migraciones de eliminacion | [infra/](./infra/) |

**NOTA:** Estas tablas NO tienen referencias en api-admin, api-mobile ni worker. Solo se requieren cambios en infra.

## Orden de Ejecucion

```
1. edugo-infrastructure
   ├── Crear migraciones de eliminacion
   ├── Actualizar seeds de testing
   ├── PR a dev
   ├── Merge a dev
   ├── PR de dev a main
   ├── Merge a main
   └── GitHub Release: postgres/v0.14.0
```

## Verificacion Pre-Eliminacion

Antes de eliminar, ejecutar estos comandos para confirmar que no hay referencias:

```bash
# En cada proyecto, buscar referencias a las tablas
cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion
grep -r "user_active_context\|user_favorites\|user_activity_log\|feature_flags\|feature_flag_overrides" --include="*.go"

cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile
grep -r "user_active_context\|user_favorites\|user_activity_log\|feature_flags\|feature_flag_overrides" --include="*.go"

cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker
grep -r "user_active_context\|user_favorites\|user_activity_log\|feature_flags\|feature_flag_overrides" --include="*.go"
```

Si no hay resultados, es seguro eliminar.

---

## Siguiente Paso

Ir a [infra/PLAN.md](./infra/PLAN.md) para ver el plan detallado de cambios en edugo-infrastructure.
