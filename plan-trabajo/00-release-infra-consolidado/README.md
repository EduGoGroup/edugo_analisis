# 00 - Release Consolidado de Infrastructure

**Estado:** Pendiente
**Prioridad:** CRITICA (Libera dependencias para todos los proyectos)
**Tiempo estimado:** 1-2 dias

---

## Objetivo

Consolidar TODAS las tareas de `edugo-infrastructure` en un UNICO release para liberar la dependencia de manera transversal a todos los proyectos.

## Ventajas de Consolidar

1. **Un solo release** - Los proyectos dependientes actualizan una vez
2. **Menos conflictos** - Todas las migraciones en secuencia
3. **Testing integrado** - Probar todo junto antes de release
4. **Deployment simplificado** - Un solo tag, un solo release

## Tareas Consolidadas

| Origen | Descripcion | Tipo |
|--------|-------------|------|
| Tarea 01 | Eliminar 5 tablas PostgreSQL sin uso | PostgreSQL |
| Tarea 02 | Eliminar 5 colecciones MongoDB sin uso | MongoDB |
| Tarea 03 | Eliminar migracion duplicada material_assessment | MongoDB |
| Tarea 04 | Crear script migracion material_summaries | MongoDB |
| Tarea 05 | Eliminar campos sin uso de tabla users | PostgreSQL |

## Releases Resultantes

| Modulo | Version | Cambios |
|--------|---------|---------|
| `postgres` | v0.15.0 | Eliminar tablas + campos |
| `mongodb` | v0.13.0 | Eliminar colecciones + homologar |

## Impacto en Otros Proyectos

Despues de este release, los proyectos pueden actualizar:

```bash
# En cada proyecto dependiente
go get github.com/EduGoGroup/edugo-infrastructure/postgres@v0.15.0
go get github.com/EduGoGroup/edugo-infrastructure/mongodb@v0.13.0
go mod tidy
```

## Orden de Ejecucion Post-Release

```
1. Release edugo-infrastructure (este plan)
   └── postgres/v0.15.0 + mongodb/v0.13.0

2. Ejecutar scripts de migracion en ambientes
   ├── Scripts PostgreSQL (eliminar tablas y campos)
   └── Scripts MongoDB (eliminar colecciones, migrar datos)

3. Actualizar proyectos dependientes (pueden hacerse en paralelo):
   ├── edugo-api-administracion (Tarea 05, 06)
   ├── edugo-api-mobile (Tarea 03, 04, 07)
   └── edugo-worker (Tarea 03 - verificacion)
```

---

## Siguiente Paso

Ir a [PLAN.md](./PLAN.md) para ver el plan detallado consolidado.
