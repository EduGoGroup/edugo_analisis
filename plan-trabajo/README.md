# Plan de Trabajo - Ecosistema EduGo

**Fecha de creacion:** 2025-12-23
**Estado:** En planificacion

---

## Estrategia de Releases

### Release Consolidado de Infrastructure (PRIMERO)

Para liberar dependencias de manera transversal, todas las tareas de `edugo-infrastructure` se consolidan en un unico release:

| Modulo | Version | Contenido |
|--------|---------|-----------|
| `postgres` | v0.15.0 | Eliminar tablas + campos sin uso |
| `mongodb` | v0.13.0 | Eliminar colecciones + homologar |

**Ver:** [00-release-infra-consolidado/](./00-release-infra-consolidado/)

---

## Indice de Tareas

### Tarea 00 - Release Consolidado Infra (CRITICO)

| # | Tarea | Prioridad | Tiempo Est. |
|---|-------|-----------|-------------|
| 00 | [Release Infra Consolidado](./00-release-infra-consolidado/) | **CRITICA** | 1-2 dias |

Esta tarea consolida:
- Tarea 01: Eliminar tablas PostgreSQL
- Tarea 02: Eliminar colecciones MongoDB
- Tarea 03 (infra): Eliminar migracion duplicada
- Tarea 04 (infra): Script migracion summaries
- Tarea 05 (infra): Eliminar campos de users

---

### Tareas Post-Release Infra

| # | Tarea | Proyectos Afectados | Prioridad | Tiempo Est. |
|---|-------|---------------------|-----------|-------------|
| 03 | [Homologar material_assessment](./03-homologar-material-assessment/) | api-mobile, worker | Alta | 6-8h |
| 04 | [Homologar material_summary](./04-homologar-material-summary/) | api-mobile | Alta | 4-6h |
| 05 | [Eliminar campos sin uso](./05-eliminar-campos-sin-uso/) | api-admin | Media | 2-3h |
| 06 | [Endpoints faltantes API Admin](./06-endpoints-faltantes-api-admin/) | api-admin | Media | 12-15h |
| 07 | [Endpoints faltantes API Mobile](./07-endpoints-faltantes-api-mobile/) | api-mobile | Alta | 6-8h |

---

### Tareas Worker (Largo Plazo)

| # | Tarea | Proyectos Afectados | Prioridad | Tiempo Est. |
|---|-------|---------------------|-----------|-------------|
| 08 | [Worker Fase 1 - Core](./08-worker-fase1/) | worker | Alta | 2-3 sem |
| 09 | [Worker Fase 2 - Integraciones](./09-worker-fase2/) | worker | Alta | 3-4 sem |

---

## Flujo de Trabajo Actualizado

### Fase 1: Release Infrastructure (1-2 dias)

```
00-release-infra-consolidado
├── PostgreSQL
│   ├── Migracion 016: Eliminar tablas sin uso
│   └── Migracion 017: Eliminar campos de users
├── MongoDB
│   ├── Eliminar colecciones sin uso
│   ├── Eliminar migracion duplicada material_assessment
│   └── Scripts de migracion de datos
├── PR a dev -> Merge
├── PR de dev a main -> Merge
└── Releases: postgres/v0.15.0 + mongodb/v0.13.0
```

### Fase 2: Ejecutar Migraciones en Ambientes (1 dia)

```
En cada ambiente (dev, staging, prod):
├── Backup PostgreSQL
├── Backup MongoDB
├── Ejecutar migraciones PostgreSQL
├── Ejecutar scripts MongoDB
├── Verificar integridad
└── Eliminar colecciones antiguas
```

### Fase 3: Actualizar Proyectos Dependientes (3-5 dias)

Pueden ejecutarse en paralelo:

```
edugo-api-administracion
├── Actualizar go.mod: postgres@v0.15.0
├── Tarea 05: Eliminar referencias a campos
├── Tarea 06: Endpoints faltantes Subjects/Guardians
└── PR a dev

edugo-api-mobile
├── Actualizar go.mod: mongodb@v0.13.0
├── Tarea 03: Usar material_assessment_worker
├── Tarea 04: Usar material_summary
├── Tarea 07: Endpoint PUT materials
└── PR a dev

edugo-worker
├── Actualizar go.mod: mongodb@v0.13.0
├── Verificar uso de material_assessment_worker (sin cambios)
└── PR a dev (si hay cambios)
```

### Fase 4: Worker Core (2-3 semanas)

```
08-worker-fase1
├── Integracion S3 real
├── Extraccion PDF real
├── Integracion OpenAI real
└── MaterialUploadedProcessor funcional
```

### Fase 5: Worker Integraciones (3-4 semanas)

```
09-worker-fase2
├── Email (SendGrid/SES)
├── Push Notifications (Firebase)
├── AssessmentAttemptProcessor
└── StudentEnrolledProcessor
```

---

## Orden de Ejecucion Recomendado

### Sprint 1: Infrastructure (Dia 1-2)

```
Dia 1:
├── 00-release-infra-consolidado
│   ├── Crear rama feature
│   ├── Implementar cambios PostgreSQL
│   ├── Implementar cambios MongoDB
│   └── PR a dev

Dia 2:
├── Review y merge a dev
├── PR de dev a main
├── Merge a main
├── Crear releases
└── Ejecutar migraciones en ambientes
```

### Sprint 2: APIs (Dia 3-7)

```
Dia 3-4:
├── edugo-api-administracion
│   ├── Tarea 05: Eliminar referencias
│   └── Tarea 06: Endpoints Subjects/Guardians

Dia 5-7:
├── edugo-api-mobile
│   ├── Tarea 03: Homologar assessment
│   ├── Tarea 04: Homologar summary
│   └── Tarea 07: PUT materials
```

### Sprint 3-6: Worker (Semana 2-6)

```
Semana 2-3:
└── 08-worker-fase1 (S3, PDF, OpenAI)

Semana 4-6:
└── 09-worker-fase2 (Email, Push, Processors)
```

---

## Nomenclatura de Ramas

| Tipo | Patron | Ejemplo |
|------|--------|---------|
| Release Consolidado | `feature/PT-000-release-consolidado-xxx` | `feature/PT-000-release-consolidado-limpieza` |
| Feature | `feature/PT-[num]-descripcion` | `feature/PT-003-homologar-material-assessment` |
| Fix | `fix/PT-[num]-descripcion` | `fix/PT-007-material-update` |

---

## Checklist General por Tarea

Cada tarea debe cumplir:

- [ ] Rama creada desde `dev`
- [ ] Cambios realizados segun plan
- [ ] `go build ./...` exitoso
- [ ] `go test ./...` exitoso (cobertura minima 60%)
- [ ] `golangci-lint run` sin errores criticos
- [ ] Documentacion actualizada si aplica
- [ ] PR creado con descripcion detallada
- [ ] Code review aprobado
- [ ] Merge a `dev`
- [ ] (Si es shared/infra) PR de `dev` a `main`
- [ ] (Si es shared/infra) GitHub Release creado

---

## Diagrama de Dependencias

```
                    ┌─────────────────────────────────┐
                    │  00-release-infra-consolidado   │
                    │  postgres/v0.15.0               │
                    │  mongodb/v0.13.0                │
                    └───────────────┬─────────────────┘
                                    │
            ┌───────────────────────┼───────────────────────┐
            │                       │                       │
            ▼                       ▼                       ▼
    ┌───────────────┐       ┌───────────────┐       ┌───────────────┐
    │ api-admin     │       │ api-mobile    │       │ worker        │
    │ Tarea 05, 06  │       │ Tarea 03,04,07│       │ (verificar)   │
    └───────────────┘       └───────────────┘       └───────┬───────┘
                                                            │
                                                            ▼
                                                    ┌───────────────┐
                                                    │ Tarea 08, 09  │
                                                    │ Worker Core   │
                                                    └───────────────┘
```

---

## Archivos de Referencia

### Informes de Analisis

- [INFORME_ACCIONES_DATOS.md](../INFORME_ACCIONES_DATOS.md) - Analisis de BD
- [INFORME_ENDPOINTS_WORKER.md](../INFORME_ENDPOINTS_WORKER.md) - Analisis de APIs
- [INFORME_CONSOLIDADO_ECOSISTEMA.md](../documentacion/INFORME_CONSOLIDADO_ECOSISTEMA.md) - Vision general

### Documentacion

- [CONVENCIONES.md](../documentacion/CONVENCIONES.md) - Convenciones del ecosistema
- [BASE_DE_DATOS.md](../documentacion/BASE_DE_DATOS.md) - Esquemas de BD
- [ENDPOINTS_Y_WORKERS.md](../documentacion/ENDPOINTS_Y_WORKERS.md) - APIs y Worker

---

## Contacto

- **Arquitecto:** TBD
- **Tech Lead:** TBD
- **Fecha objetivo MVP:** TBD

---

*Este documento es el indice principal. La carpeta 00 contiene el plan consolidado de infrastructure que debe ejecutarse primero.*
