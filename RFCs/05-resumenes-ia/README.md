# 05 - Resúmenes IA

## Descripción

Este módulo documenta el sistema de resúmenes automáticos generados por IA a partir del contenido de los materiales educativos. Los resúmenes incluyen ideas principales, conceptos clave y contenido organizado por secciones.

## Dependencia Crítica: Worker

**IMPORTANTE:** El RFC-040 requiere que el Worker esté activo. Sin Worker, el endpoint retornará 404 indicando que el resumen no está disponible.

```bash
# Verificar Worker
curl http://localhost:8083/health
```

## RFCs en este Módulo

| RFC | Nombre | Estado | Dependencia |
|-----|--------|--------|-------------|
| [RFC-040](./RFC-040-obtener-resumen.md) | Obtener Resumen Generado por IA | ❌ | Worker |
| [RFC-041](./RFC-041-manejo-estado-procesando.md) | Manejo de Estado Procesando | ✅ | Ninguna |

## Flujo General

```
Material (status=ready)
         ↓
GET /materials/:id/summary
         ↓
    ┌────┴────┐
    ↓         ↓
200 OK    404 Not Ready
    ↓         ↓
Mostrar   Polling cada 10s
Resumen   o mostrar "Generando..."
```

## Estructura del Resumen

```typescript
interface SummaryResponse {
  material_id: string;
  main_ideas: string[];           // 3-5 ideas principales
  key_concepts: KeyConceptDTO[];  // 5-10 conceptos clave
  sections: SectionDTO[];         // Contenido organizado
  glossary?: GlossaryEntryDTO[];  // Glosario opcional
  word_count: number;
  estimated_reading_time: number; // Minutos
  generated_at: string;
  model_version?: string;         // "gpt-4", "claude-3", "fallback"
}
```

## Calidad del Resumen

| Modelo | Calidad | Notas |
|--------|---------|-------|
| GPT-4 | Excelente | Ideas bien identificadas |
| Claude-3 | Excelente | Conceptos precisos |
| Fallback | Básica | Extracción simple, mostrar disclaimer |

## Consideraciones de Implementación

1. **Polling:** Si resumen no disponible (404), polling cada 10 segundos
2. **Cache:** Cachear resumen por 24 horas (no cambia una vez generado)
3. **Fallback:** Mostrar disclaimer si modelo es "fallback"
4. **Offline:** Guardar en IndexedDB para acceso sin conexión

---

**Última actualización:** 2024-12-24
