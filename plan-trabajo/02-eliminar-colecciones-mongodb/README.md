# 02 - Eliminar Colecciones MongoDB Sin Uso

**Estado:** Pendiente
**Prioridad:** Alta
**Tiempo estimado:** 3-4 horas

---

## Resumen

Eliminar 5 colecciones de MongoDB que fueron creadas pero nunca utilizadas en el codigo.

## Colecciones a Eliminar

| Coleccion | Migracion | Razon |
|-----------|-----------|-------|
| `material_content` | 002 | Procesamiento PDF no implementado |
| `assessment_attempt_result` | 003 | Datos estan en PostgreSQL |
| `audit_logs` | 004 | Sistema de auditoria no implementado |
| `notifications` | 005 | Push notifications no implementado |
| `analytics_events` | 006 | Analytics usa SaaS externo |

## Proyectos Afectados

| Proyecto | Accion | Carpeta |
|----------|--------|---------|
| edugo-infrastructure | Eliminar migraciones y seeds | [infra/](./infra/) |

**NOTA:** Estas colecciones NO tienen referencias en api-admin, api-mobile ni worker. Solo se requieren cambios en infra.

## Orden de Ejecucion

```
1. edugo-infrastructure
   ├── Eliminar migraciones de estructura
   ├── Eliminar migraciones de constraints/indexes
   ├── Eliminar seeds
   ├── PR a dev
   ├── Merge a dev
   ├── PR de dev a main
   ├── Merge a main
   └── GitHub Release: mongodb/v0.12.0
```

## Verificacion Pre-Eliminacion

Antes de eliminar, ejecutar estos comandos para confirmar que no hay referencias:

```bash
# En cada proyecto, buscar referencias a las colecciones
cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-administracion
grep -r "material_content\|assessment_attempt_result\|audit_logs\|notifications\|analytics_events" --include="*.go"

cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-api-mobile
grep -r "material_content\|assessment_attempt_result\|audit_logs\|notifications\|analytics_events" --include="*.go"

cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-worker
grep -r "material_content\|assessment_attempt_result\|audit_logs\|notifications\|analytics_events" --include="*.go"
```

Si no hay resultados en codigo (solo en configs), es seguro eliminar.

---

## Siguiente Paso

Ir a [infra/PLAN.md](./infra/PLAN.md) para ver el plan detallado de cambios en edugo-infrastructure.
