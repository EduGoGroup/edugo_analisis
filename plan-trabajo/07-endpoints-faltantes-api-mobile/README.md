# 07 - Endpoints Faltantes API Mobile

**Estado:** Pendiente
**Prioridad:** Alta
**Tiempo estimado:** 6-8 horas

---

## Resumen

Implementar el endpoint PUT para actualizar materiales en API Mobile.

## Endpoint Faltante

| Endpoint | Metodo | Descripcion | Prioridad |
|----------|--------|-------------|-----------|
| `/v1/materials/:id` | PUT | Actualizar material | **Alta** |

## Estado Actual

**Materials en API Mobile:**
- POST `/v1/materials` - Crear material
- GET `/v1/materials` - Listar materiales
- GET `/v1/materials/:id` - Obtener material
- GET `/v1/materials/:id/versions` - Historial versiones
- POST `/v1/materials/:id/upload-url` - URL de upload S3
- POST `/v1/materials/:id/upload-complete` - Completar upload
- GET `/v1/materials/:id/download-url` - URL de descarga S3
- GET `/v1/materials/:id/summary` - Resumen (MongoDB)
- GET `/v1/materials/:id/stats` - Estadisticas
- **FALTA:** PUT `/v1/materials/:id` - Actualizar

## Por Que es Alta Prioridad

El endpoint PUT es esencial para:
- Corregir errores en metadata de materiales
- Cambiar titulo, descripcion, tags
- Actualizar visibilidad (publico/privado)
- Mover material a otra unidad academica

## Proyectos Afectados

| Proyecto | Accion | Carpeta |
|----------|--------|---------|
| edugo-api-mobile | Implementar endpoint PUT | [api-mobile/](./api-mobile/) |

## Orden de Ejecucion

```
1. edugo-api-mobile
   ├── Crear DTO UpdateMaterialRequest
   ├── Agregar metodo Update en repository
   ├── Agregar metodo en service
   ├── Implementar handler
   ├── Agregar ruta
   ├── Actualizar swagger
   ├── Agregar tests
   ├── PR a dev
   └── Merge a dev
```

---

## Siguiente Paso

Ir a [api-mobile/PLAN.md](./api-mobile/PLAN.md) para ver el plan detallado.
