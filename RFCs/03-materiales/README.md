# RFCs - Módulo de Materiales

Este directorio contiene los RFCs (Request for Comments) para el desarrollo del módulo de Materiales en la aplicación mobile de EduGo.

## Índice de RFCs

### RFC-020: Listado de Materiales
**Archivo:** [RFC-020-listado-materiales.md](./RFC-020-listado-materiales.md)
- **Estado:** ✅ Listo
- **Prioridad:** Alta
- **Descripción:** Permite a estudiantes y docentes visualizar la lista de materiales educativos disponibles para una sección/unidad académica específica.
- **Endpoints:** `GET /v1/materials`
- **Características:**
  - Listado completo con metadata
  - Estados visuales (ready, processing, uploaded, failed)
  - Auto-refetch para materiales en procesamiento
  - Sin paginación (limitación actual)

---

### RFC-021: Subida de Material PDF
**Archivo:** [RFC-021-subida-pdf.md](./RFC-021-subida-pdf.md)
- **Estado:** ✅ Listo (incluye Worker)
- **Prioridad:** Alta
- **Descripción:** Proceso completo de subida de un material educativo en formato PDF con procesamiento asíncrono mediante Worker.
- **Endpoints:**
  - `POST /v1/materials` - Crear metadata
  - `POST /v1/materials/:id/upload-url` - Generar URL presignada S3
  - `POST /v1/materials/:id/upload-complete` - Notificar completitud
- **Características:**
  - Upload directo a S3 (sin pasar por backend)
  - URLs presignadas con expiración de 15 minutos
  - Procesamiento asíncrono con Worker (PDF extraction + IA)
  - Generación automática de resumen y quiz
  - Flujo completo: Upload → RabbitMQ → Worker → MongoDB

---

### RFC-022: Descarga y Visualización de Material
**Archivo:** [RFC-022-descarga-material.md](./RFC-022-descarga-material.md)
- **Estado:** ✅ Listo
- **Prioridad:** Alta
- **Descripción:** Permite a usuarios autenticados descargar y visualizar materiales PDF almacenados en S3.
- **Endpoints:** `GET /v1/materials/:id/download-url`
- **Características:**
  - URLs presignadas con expiración de 15 minutos
  - Viewer PDF integrado (react-pdf)
  - Auto-guardado de progreso cada 30 segundos
  - Descarga para lectura offline
  - Cache en IndexedDB

---

### RFC-023: Versionado de Materiales
**Archivo:** [RFC-023-versionado-materiales.md](./RFC-023-versionado-materiales.md)
- **Estado:** ⚠️ Parcial (PUT /materials/:id solo en dev)
- **Prioridad:** Media
- **Descripción:** Sistema de control de versiones para materiales educativos que permite actualizar metadata y archivo PDF manteniendo historial completo.
- **Endpoints:**
  - `PUT /v1/materials/:id` - Actualizar metadata (⚠️ solo dev)
  - `GET /v1/materials/:id/versions` - Ver historial
- **Características:**
  - Actualización de metadata sin crear versión nueva
  - Actualización de PDF genera versión automáticamente
  - Historial completo con timeline visual
  - Restauración de versiones (futuro)

---

### RFC-024: Progreso de Lectura
**Archivo:** [RFC-024-progreso-lectura.md](./RFC-024-progreso-lectura.md)
- **Estado:** ✅ Listo
- **Prioridad:** Alta
- **Descripción:** Sistema de registro de progreso de lectura con auto-guardado cada 30 segundos y sincronización offline.
- **Endpoints:** `PUT /v1/progress`
- **Características:**
  - Operación UPSERT idempotente
  - Auto-guardado transparente cada 30 segundos
  - Sincronización offline con IndexedDB
  - Evento `material.completed` al llegar a 100%
  - Restauración de última página leída
  - Barra de progreso visual

---

## Flujo Completo del Sistema

### 1. Docente Sube Material
```
RFC-021: Subida PDF
  ↓
Material creado (metadata)
  ↓
Upload PDF a S3
  ↓
Worker procesa (2-5 min)
  ↓
Material ready con resumen + quiz
```

### 2. Estudiante Consume Material
```
RFC-020: Listado
  ↓
Selecciona material
  ↓
RFC-022: Descarga/Visualiza
  ↓
RFC-024: Progreso auto-guardado
  ↓
100% → Evento completitud
```

### 3. Docente Actualiza Material
```
RFC-023: Versionado
  ↓
Opción 1: Actualizar metadata (sin versión)
Opción 2: Actualizar PDF (nueva versión)
  ↓
Worker reprocesa (si PDF cambió)
```

---

## Dependencias entre RFCs

```
RFC-001 (Autenticación)
  ↓
RFC-020 (Listado) ← Base para navegación
  ├─→ RFC-021 (Subida) ← Docentes crean materiales
  │     ↓
  │   Worker procesa
  │     ↓
  ├─→ RFC-022 (Descarga) ← Estudiantes leen
  │     ↓
  │   RFC-024 (Progreso) ← Tracking automático
  │
  └─→ RFC-023 (Versionado) ← Docentes actualizan
```

---

## Estado de Implementación por RFC

| RFC | Nombre | Estado API | Estado Worker | Frontend Ready |
|-----|--------|-----------|---------------|----------------|
| RFC-020 | Listado | ✅ Listo | N/A | ✅ Sí |
| RFC-021 | Subida PDF | ✅ Listo | ✅ Listo | ✅ Sí |
| RFC-022 | Descarga | ✅ Listo | N/A | ✅ Sí |
| RFC-023 | Versionado | ⚠️ Parcial (dev) | ✅ Listo | ⚠️ Después de merge |
| RFC-024 | Progreso | ✅ Listo | N/A | ✅ Sí |

---

## Bloqueantes Críticos

### 1. Merge dev → main (API Mobile)
**Impacto:** RFC-023 (Versionado)
- Endpoint `PUT /v1/materials/:id` solo está en rama dev
- Requiere sincronización dev → main antes de producción

### 2. Worker Funcionando
**Impacto:** RFC-021 (Subida PDF)
- Sin Worker: materiales quedan en status `processing` indefinidamente
- Resúmenes y quizzes NO se generan
- Frontend debe manejar degradación graceful

---

## Orden de Implementación Recomendado

### Sprint 1 - Base (Semana 1-2)
1. ✅ RFC-020: Listado de Materiales
   - Implementar UI de lista
   - Filtros básicos
   - Estados visuales

### Sprint 2 - Upload (Semana 2-3)
2. ✅ RFC-021: Subida de Material PDF
   - Formulario de metadata
   - Integración con S3
   - Polling de estado Worker
   - Manejo de errores

### Sprint 3 - Lectura (Semana 3-4)
3. ✅ RFC-022: Descarga y Visualización
   - Viewer PDF integrado
   - Controles de navegación
   - Cache offline
4. ✅ RFC-024: Progreso de Lectura
   - Auto-guardado cada 30s
   - Sincronización offline
   - Barra de progreso

### Sprint 4 - Avanzado (Semana 4-5)
5. ⚠️ RFC-023: Versionado (después de merge)
   - Edición de metadata
   - Actualización de PDF
   - Historial de versiones

---

## Tecnologías Utilizadas

### Frontend (Mobile)
- **React + TypeScript**
- **React Query** - Estado servidor y cache
- **Axios** - HTTP client
- **react-pdf** - Viewer PDF
- **IndexedDB** - Almacenamiento offline
- **date-fns** - Formateo de fechas

### Backend (API Mobile)
- **Go 1.25**
- **Gin** - Framework HTTP
- **PostgreSQL** - BD relacional
- **MongoDB** - BD documentos (resúmenes, quizzes)
- **AWS S3** - Almacenamiento archivos
- **RabbitMQ** - Cola de mensajes

### Worker
- **Go 1.25**
- **pdfcpu** - Extracción PDF
- **OpenAI API / Fallback** - Generación IA
- **MongoDB** - Escritura de resultados

---

## Recursos Adicionales

### Documentación de Referencia
- [INFORME_API_MOBILE.md](../../analisis-frontend-readiness/informes-agentes/INFORME_API_MOBILE.md)
- [INFORME_WORKER.md](../../analisis-frontend-readiness/informes-agentes/INFORME_WORKER.md)
- [ENDPOINTS_VIABLES_FRONTEND.md](../../analisis-frontend-readiness/ENDPOINTS_VIABLES_FRONTEND.md)
- [CONSOLIDADO_ECOSISTEMA.md](../../analisis-frontend-readiness/CONSOLIDADO_ECOSISTEMA.md)

### Swagger UIs
- **API Mobile:** http://localhost:8080/swagger/index.html

### Health Checks
- **API Mobile:** http://localhost:8080/health?detail=1

---

## Contacto

Para dudas o aclaraciones sobre estos RFCs, contactar al equipo de desarrollo mobile.

---

**Última actualización:** 2025-12-24
**Versión:** 1.0
**Generado por:** Claude Sonnet 4.5
