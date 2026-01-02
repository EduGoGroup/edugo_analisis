# 06 - Estadísticas y Métricas

## Descripción

Este módulo documenta los dashboards y sistemas de estadísticas que permiten a docentes, administradores y estudiantes visualizar métricas de uso, rendimiento y progreso en la plataforma.

## RFCs en este Módulo

| RFC | Nombre | Estado | Audiencia |
|-----|--------|--------|-----------|
| [RFC-050](./RFC-050-stats-material.md) | Estadísticas por Material | ✅ | Docentes/Admin |
| [RFC-051](./RFC-051-stats-globales.md) | Estadísticas Globales | ✅ | Admin |
| [RFC-052](./RFC-052-dashboard-progreso.md) | Dashboard de Progreso del Estudiante | ⚠️ | Estudiantes |

## Tipos de Estadísticas

### Para Docentes/Admin (RFC-050, RFC-051)

- Total de visualizaciones por material
- Tasa de completitud
- Promedio de calificaciones
- Tasa de aprobación en quizzes
- Distribución de progreso

### Para Estudiantes (RFC-052)

- Materiales completados
- Quizzes aprobados
- Promedio personal
- Actividad semanal/mensual
- Logros/badges

## Endpoints

| Endpoint | Método | Descripción | Estado |
|----------|--------|-------------|--------|
| `/v1/materials/:id/stats` | GET | Estadísticas de un material | ✅ |
| `/v1/stats/school` | GET | Estadísticas globales de escuela | ✅ |
| `/v1/progress/dashboard` | GET | Dashboard de progreso personal | ⚠️ |

## Consideraciones de Cache

| Tipo | TTL | Razón |
|------|-----|-------|
| Stats Material | 5 min | Datos agregados, baja frecuencia de cambio |
| Stats Globales | 10 min | Consultas costosas |
| Dashboard Personal | 2 min | Datos dinámicos, usuario activo |

## Visualizaciones Recomendadas

1. **Cards de Métricas:** Números grandes con iconos
2. **Gráficos de Barras:** Actividad semanal
3. **Progress Bars:** Porcentajes de completitud
4. **Listas:** Materiales en progreso, evaluaciones recientes

---

**Última actualización:** 2024-12-24
