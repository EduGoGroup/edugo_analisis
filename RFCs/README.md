# RFCs para Desarrollo Mobile - EduGo Platform

**Fecha de Generación:** 2024-12-24
**Versión:** 2.0
**Estado:** Completo (6 Módulos Documentados)

---

## Resumen Ejecutivo

Este directorio contiene los RFCs (Request for Comments) que documentan los flujos técnicos y de usuario para el desarrollo de la aplicación mobile de EduGo. Cada RFC incluye:

- Especificaciones de endpoints
- Interfaces TypeScript
- Código de ejemplo funcional (React/TypeScript)
- Estrategias de cache y manejo de errores
- Consideraciones de UX

### Estado de las APIs

| API | Puerto | Estado | Endpoints Documentados |
|-----|--------|--------|------------------------|
| **API Admin** | 8081 | ✅ Funcional | 15+ endpoints |
| **API Mobile** | 8080 | ✅ Funcional | 18+ endpoints |
| **Worker** | 8083 | ⚠️ Requerido | Procesamiento IA |

---

## Índice de RFCs por Módulo

### 00-arquitectura/ (Fundamentos)

Patrones y estrategias transversales para toda la aplicación.

| RFC | Nombre | Estado |
|-----|--------|--------|
| [RFC-000](./00-arquitectura/RFC-000-flujo-datos-mobile.md) | Flujo General de Datos | ✅ Listo |
| [RFC-002](./00-arquitectura/RFC-002-polling-estados.md) | Polling para Estados de Procesamiento | ✅ Listo |
| [RFC-003](./00-arquitectura/RFC-003-almacenamiento-local.md) | Almacenamiento Local y Cache | ✅ Listo |
| [RFC-004](./00-arquitectura/RFC-004-manejo-errores.md) | Estrategia de Manejo de Errores HTTP | ✅ Listo |

---

### 01-autenticacion/ (Sesiones)

Flujos de autenticación y manejo de sesiones.

| RFC | Nombre | Estado |
|-----|--------|--------|
| [RFC-001](./01-autenticacion/RFC-001-login-usuario.md) | Login con Email/Password | ✅ Listo |
| [RFC-002](./01-autenticacion/RFC-002-refresh-token.md) | Renovación de Tokens JWT | ✅ Listo |
| [RFC-003](./01-autenticacion/RFC-003-logout.md) | Cierre de Sesión | ✅ Listo |
| [RFC-004](./01-autenticacion/RFC-004-validacion-sesion.md) | Validación de Sesión Activa | ✅ Listo |

---

### 02-gestion-escolar/ (Administración)

CRUD de entidades administrativas de la escuela.

| RFC | Nombre | Estado |
|-----|--------|--------|
| [RFC-010](./02-gestion-escolar/RFC-010-crud-escuelas.md) | CRUD de Escuelas | ✅ Listo |
| [RFC-011](./02-gestion-escolar/RFC-011-jerarquia-academica.md) | Jerarquía Académica (Árbol de Unidades) | ✅ Listo |
| [RFC-012](./02-gestion-escolar/RFC-012-membresias.md) | Gestión de Membresías | ✅ Listo |
| [RFC-013](./02-gestion-escolar/RFC-013-relaciones-acudientes.md) | Relaciones de Acudientes | ✅ Listo |
| [RFC-014](./02-gestion-escolar/RFC-014-gestion-materias.md) | Gestión de Materias (Subjects) | ⚠️ Parcial |

---

### 03-materiales/ (Contenido Educativo)

Gestión de materiales educativos y PDFs.

| RFC | Nombre | Estado |
|-----|--------|--------|
| [RFC-020](./03-materiales/RFC-020-listado-materiales.md) | Listado de Materiales | ✅ Listo |
| [RFC-021](./03-materiales/RFC-021-subida-pdf.md) | Subida de PDF (Teacher) | ✅ Listo |
| [RFC-022](./03-materiales/RFC-022-descarga-material.md) | Descarga de Material | ✅ Listo |
| [RFC-023](./03-materiales/RFC-023-versionado-materiales.md) | Versionado de Materiales | ✅ Listo |
| [RFC-024](./03-materiales/RFC-024-progreso-lectura.md) | Progreso de Lectura | ✅ Listo |
| [RFC-025](./03-materiales/RFC-025-ver-detalle-material.md) | Ver Detalle de Material | ✅ Listo |

---

### 04-evaluaciones/ (Quizzes)

Sistema de evaluaciones generadas por IA.

| RFC | Nombre | Estado |
|-----|--------|--------|
| [RFC-030](./04-evaluaciones/RFC-030-obtener-quiz.md) | Obtener Quiz Generado por IA | ✅ IMPLEMENTADO |
| [RFC-031](./04-evaluaciones/RFC-031-enviar-intento.md) | Enviar Respuestas de Quiz | ✅ IMPLEMENTADO |
| [RFC-032](./04-evaluaciones/RFC-032-ver-resultados.md) | Ver Resultados de Intento | ✅ Listo |
| [RFC-033](./04-evaluaciones/RFC-033-historial-intentos.md) | Historial de Intentos | ✅ Listo |

> **ACTUALIZACIÓN 2025-01-02:** RFC-030 y RFC-031 están completamente implementados:
> - Worker genera quiz automáticamente al procesar PDF
> - API Mobile expone endpoints para obtener quiz y enviar respuestas
> - Scoring 100% servidor-side (seguro)

---

### 05-resumenes-ia/ (Contenido Generado)

Resúmenes y contenido generado por IA.

| RFC | Nombre | Estado |
|-----|--------|--------|
| [RFC-040](./05-resumenes-ia/RFC-040-obtener-resumen.md) | Obtener Resumen Generado por IA | ❌ Depende Worker |
| [RFC-041](./05-resumenes-ia/RFC-041-manejo-estado-procesando.md) | Manejo de Estado Procesando | ✅ Listo |

---

### 06-estadisticas/ (Métricas y Progreso)

Dashboards y estadísticas de uso.

| RFC | Nombre | Estado |
|-----|--------|--------|
| [RFC-050](./06-estadisticas/RFC-050-stats-material.md) | Estadísticas por Material | ✅ Listo |
| [RFC-051](./06-estadisticas/RFC-051-stats-globales.md) | Estadísticas Globales | ✅ Listo |
| [RFC-052](./06-estadisticas/RFC-052-dashboard-progreso.md) | Dashboard de Progreso del Estudiante | ⚠️ Parcial |

---

## Estadísticas

| Métrica | Valor |
|---------|-------|
| **Total RFCs** | 28 |
| **Listos para implementar** | 24 |
| **Dependen del Worker** | 1 (RFC-040 Resumen) |
| **Parciales (requieren backend)** | 3 |

> **ACTUALIZACIÓN 2025-01-02:** Quiz IA (RFC-030, RFC-031) ahora están implementados

---

## Estructura de Cada RFC

Todos los RFCs siguen la siguiente estructura estandarizada:

```markdown
## Metadata
- ID, Proceso, Subproceso
- Prioridad, Dependencias
- Estado API

## Descripción
Qué hace desde perspectiva del usuario

## Flujo de Usuario (UX)
Pasos del usuario en la interfaz
Mockups ASCII

## Flujo de Datos (Técnico)
- Diagrama de Secuencia
- Endpoints Involucrados
- Request/Response (TypeScript)

## Estados y Transiciones
Diagrama de máquina de estados

## Manejo de Errores
Tabla de códigos HTTP y acciones

## Consideraciones de UX
- Loading states
- Skeleton loaders
- Mensajes de error
- Confirmaciones

## Almacenamiento Local
- Qué cachear
- TTL del cache
- Estrategia offline

## Código de Ejemplo (Mobile)
Implementación funcional en TypeScript/React
- Service Layer
- Custom Hooks
- Componentes React

## Notas de Implementación
- Gotchas conocidos
- Optimizaciones sugeridas
- Testing
```

---

## Información Técnica Base

### Endpoints Base

```typescript
const API_URLS = {
  admin: 'http://localhost:8081/v1',   // Autenticación, Escuelas
  mobile: 'http://localhost:8080/v1'   // Materiales, Evaluaciones, Progreso
};
```

### JWT Claims Estándar

```typescript
interface JWTClaims {
  sub: string;       // user_id
  email: string;
  role: 'student' | 'teacher' | 'admin' | 'super_admin';
  school_id: string;
  iss: 'edugo-central';
  exp: number;       // Expiración (15 min)
  iat: number;
}
```

### Tecnologías Recomendadas

**Frontend Mobile:**
- React / React Native
- TypeScript
- Axios para HTTP
- React Query (opcional) para cache
- Zustand para estado global
- date-fns para fechas

---

## Dependencias Críticas

### Worker (edugo-worker)

Los siguientes RFCs requieren que el Worker esté activo:

| RFC | Funcionalidad | Estado | Alternativa |
|-----|---------------|--------|-------------|
| RFC-030 | Obtener Quiz | ✅ IMPLEMENTADO | Worker + API Mobile listos |
| RFC-031 | Enviar Intento | ✅ IMPLEMENTADO | Scoring servidor-side |
| RFC-040 | Obtener Resumen | ⚠️ PENDIENTE | Mostrar "Generando..." |

> **NOTA:** El Worker genera tanto Quiz como Resumen automáticamente al procesar PDF.
> RFC-040 requiere endpoint en API Mobile para exponer el resumen (ya existe en MongoDB `material_summaries`).

**Verificación del Worker:**
```bash
curl http://localhost:8083/health
```

---

## Recursos Adicionales

### Análisis Base (Referencia)
- [Informe API Admin](../analisis-frontend-readiness/informes-agentes/INFORME_API_ADMIN.md)
- [Informe API Mobile](../analisis-frontend-readiness/informes-agentes/INFORME_API_MOBILE.md)
- [Consolidado Ecosistema](../analisis-frontend-readiness/CONSOLIDADO_ECOSISTEMA.md)

### Swagger UIs
- API Admin: http://localhost:8081/swagger/index.html
- API Mobile: http://localhost:8080/swagger/index.html

### Health Checks
- API Admin: http://localhost:8081/health
- API Mobile: http://localhost:8080/health?detail=1

---

## Leyenda de Estados

| Estado | Significado |
|--------|-------------|
| ✅ Listo | Endpoint funcional, RFC completo |
| ⚠️ Parcial | Requiere implementación backend adicional |
| ❌ Depende Worker | Requiere Worker activo para funcionar |
| 🔧 Stub | Solo mockup, no implementado |

---

## Historial de Versiones

### v2.0 - 2024-12-24
- Reorganización de numeración de RFCs
- Agregado RFC-014 (Gestión de Materias)
- Agregado RFC-025 (Ver Detalle Material)
- Agregado RFC-052 (Dashboard de Progreso)
- Corregida duplicación de RFC-001
- Actualizado índice completo

### v1.0 - 2024-12-24
- Generación inicial de 29 RFCs
- 6 módulos documentados
- Estructura estandarizada

---

**Generado por:** Claude Opus 4.5
**Proyecto:** EduGo Platform
**Repositorio Base:** edugo_analisis
