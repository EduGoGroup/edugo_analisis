# 04 - Evaluaciones (Quizzes)

## Descripción

Este módulo documenta el sistema de evaluaciones automáticas generadas por IA a partir del contenido de los materiales educativos. Los quizzes son generados por el Worker y permiten a los estudiantes evaluar su comprensión del material.

## Dependencia Crítica: Worker

**IMPORTANTE:** Los RFCs 030 y 031 requieren que el Worker esté activo y procesando materiales. Sin Worker, los endpoints retornarán 404 indicando que el quiz no está disponible.

```bash
# Verificar Worker
curl http://localhost:8083/health
```

## RFCs en este Módulo

| RFC | Nombre | Estado | Dependencia |
|-----|--------|--------|-------------|
| [RFC-030](./RFC-030-obtener-quiz.md) | Obtener Quiz Generado por IA | ❌ | Worker |
| [RFC-031](./RFC-031-enviar-intento.md) | Enviar Respuestas de Quiz | ❌ | Worker + Quiz |
| [RFC-032](./RFC-032-ver-resultados.md) | Ver Resultados de Intento | ✅ | Ninguna |
| [RFC-033](./RFC-033-historial-intentos.md) | Historial de Intentos | ✅ | Ninguna |

## Flujo General

```
Material (status=ready)
         ↓
GET /materials/:id/assessment
         ↓
    Quiz disponible
         ↓
POST /materials/:id/assessment/attempts
         ↓
    Scoring en servidor
         ↓
GET /attempts/:id/results
         ↓
    Feedback detallado
```

## Endpoints Principales

| Endpoint | Método | Descripción |
|----------|--------|-------------|
| `/v1/materials/:id/assessment` | GET | Obtener quiz (preguntas sin respuestas) |
| `/v1/materials/:id/assessment/attempts` | POST | Enviar intento con respuestas |
| `/v1/attempts/:id/results` | GET | Ver resultados de un intento |
| `/v1/attempts` | GET | Listar historial de intentos |

## Consideraciones de Implementación

1. **Polling:** Si quiz no disponible (404), implementar polling cada 5-10 segundos
2. **Scoring:** SIEMPRE en servidor (nunca en frontend por seguridad)
3. **Intentos:** Validar intentos disponibles antes de permitir envío
4. **Cache:** Cachear quiz por 1 hora (no cambia), resultados permanentemente

## Tiempos de Generación

| Tamaño PDF | Tiempo Estimado |
|------------|-----------------|
| 10 páginas | 30-60 segundos |
| 50 páginas | 2-5 minutos |
| 100 páginas | 5-10 minutos |

---

**Última actualización:** 2024-12-24
