# 08 - Worker Fase 1 - Core

**Estado:** Pendiente
**Prioridad:** Alta
**Tiempo estimado:** 2-3 semanas

---

## Resumen

Implementar funcionalidad core del Worker que actualmente esta simulada/hardcoded.

## Estado Actual

El Worker tiene processors definidos pero con logica simulada:

| Processor | Estado | Problema |
|-----------|--------|----------|
| MaterialUploadedProcessor | Simulado | No descarga PDF de S3, no llama OpenAI |
| MaterialDeletedProcessor | Funcional | OK |
| MaterialReprocessProcessor | Simulado | Mismo problema que Upload |
| AssessmentAttemptProcessor | Solo logging | Sin logica real |
| StudentEnrolledProcessor | Solo logging | Sin logica real |

## Objetivos Fase 1

1. **Integracion S3 real** - Descargar PDFs para procesamiento
2. **Extraccion de texto PDF** - Usar biblioteca real (pdfcpu, unidoc)
3. **Integracion OpenAI real** - Generar resumenes y quizzes
4. **Processors basicos funcionando** - MaterialUploaded y MaterialDeleted

## Fuera de Scope Fase 1

- AssessmentAttemptProcessor (Fase 2)
- StudentEnrolledProcessor (Fase 2)
- Notificaciones push (Fase 2)
- Analytics avanzados (Fase 2)

## Proyectos Afectados

| Proyecto | Accion | Carpeta |
|----------|--------|---------|
| edugo-worker | Implementar integraciones | [worker/](./worker/) |

## Orden de Ejecucion

```
1. edugo-worker
   ├── Fase 1.1: Integracion S3
   ├── Fase 1.2: Extraccion PDF
   ├── Fase 1.3: Integracion OpenAI
   ├── Fase 1.4: MaterialUploadedProcessor
   ├── Fase 1.5: Tests de integracion
   ├── PR a dev
   └── Merge a dev
```

---

## Siguiente Paso

Ir a [worker/PLAN.md](./worker/PLAN.md) para ver el plan detallado.
