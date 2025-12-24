# 03 - Homologar material_assessment

**Estado:** Pendiente
**Prioridad:** Alta
**Tiempo estimado:** 12-16 horas

---

## Resumen

Unificar las colecciones `material_assessment` y `material_assessment_worker` en una sola coleccion canonica.

## Problema

Existen 2 colecciones para el mismo proposito con schemas diferentes:

| Coleccion | Usado por | Schema |
|-----------|-----------|--------|
| `material_assessment` | api-mobile | Simplificado, sin validaciones |
| `material_assessment_worker` | worker | Completo, validaciones estrictas |

## Solucion

**Coleccion canonica:** `material_assessment_worker`

**Justificacion:**
1. Schema mas completo y robusto
2. Incluye metadata de procesamiento (ai_model, processing_time_ms)
3. Validaciones estrictas
4. Worker es la fuente de verdad (genera los assessments)

## Proyectos Afectados

| Proyecto | Accion | Carpeta |
|----------|--------|---------|
| edugo-infrastructure | Eliminar migracion duplicada | [infra/](./infra/) |
| edugo-worker | Sin cambios (ya usa schema correcto) | [worker/](./worker/) |
| edugo-api-mobile | Cambiar nombre coleccion | [api-mobile/](./api-mobile/) |

## Orden de Ejecucion

```
1. edugo-infrastructure (PRIMERO)
   ├── Eliminar migracion 001_material_assessment.go
   ├── Eliminar indexes correspondientes
   ├── Crear script de migracion de datos
   ├── PR a dev -> Merge
   ├── PR de dev a main -> Merge
   └── GitHub Release: mongodb/v0.12.1

2. edugo-worker (verificacion solamente)
   └── Verificar que usa material_assessment_worker (sin cambios)

3. edugo-api-mobile (ULTIMO)
   ├── Actualizar go.mod con nueva version de infra
   ├── Cambiar nombre de coleccion en repository
   ├── Adaptar struct para nuevos campos
   ├── PR a dev
   └── Merge a dev
```

## Riesgo

**MEDIO** - Requiere migracion de datos en produccion

**Mitigacion:**
1. Ejecutar migracion en staging primero
2. Backup completo antes de migracion
3. Validacion en cada paso
4. Mantener coleccion antigua hasta validar 100%

---

## Siguiente Paso

Ir a [infra/PLAN.md](./infra/PLAN.md) para ver el plan detallado.
