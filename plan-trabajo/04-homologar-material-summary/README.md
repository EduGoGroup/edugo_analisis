# 04 - Homologar material_summary

**Estado:** Pendiente
**Prioridad:** Alta
**Tiempo estimado:** 8-10 horas

---

## Resumen

Unificar las colecciones `material_summary` (singular) y `material_summaries` (plural) en una sola coleccion canonica.

## Problema

Existen 2 colecciones para el mismo proposito con nombres y schemas diferentes:

| Coleccion | Usado por | Schema |
|-----------|-----------|--------|
| `material_summary` | worker | Completo, con validaciones |
| `material_summaries` | api-mobile | Simplificado, sin migracion formal |

## Solucion

**Coleccion canonica:** `material_summary` (singular)

**Justificacion:**
1. Tiene migracion formal con schema validation
2. Schema mas completo con metadata de IA
3. Worker es la fuente de verdad que genera summaries
4. Convencion: colecciones MongoDB en singular

## Proyectos Afectados

| Proyecto | Accion | Carpeta |
|----------|--------|---------|
| edugo-infrastructure | Verificar migracion (sin cambios) | [infra/](./infra/) |
| edugo-api-mobile | Cambiar nombre de coleccion | [api-mobile/](./api-mobile/) |

## Orden de Ejecucion

```
1. edugo-infrastructure (solo verificacion)
   └── Confirmar que migracion existe en 007_material_summary.go

2. Ejecutar migracion de datos en MongoDB
   └── Script para migrar material_summaries -> material_summary

3. edugo-api-mobile
   ├── Cambiar nombre de coleccion en repository
   ├── Adaptar struct para nuevos campos
   ├── PR a dev
   └── Merge a dev
```

## Riesgo

**BAJO-MEDIO** - Schemas similares, migracion directa

---

## Siguiente Paso

Ir a [infra/PLAN.md](./infra/PLAN.md) para ver el plan detallado.
