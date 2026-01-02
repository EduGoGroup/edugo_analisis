# RFC-050: Estadísticas por Material Individual

## Metadata
- **ID:** RFC-050
- **Proceso:** Estadísticas
- **Subproceso:** Estadísticas por Material
- **Prioridad:** Media
- **Dependencias:** RFC-010 (Listado de Materiales), RFC-030 (Sistema de Progreso)
- **Estado API:** ✅ Listo
- **Endpoints:** API Mobile

## Descripción

Visualización de métricas y estadísticas específicas de un material educativo, mostrando información sobre su uso, rendimiento de estudiantes, tasa de completitud y resultados de evaluaciones. Este dashboard permite a docentes y administradores monitorear la efectividad del material.

## Flujo de Usuario (UX)

### Docente/Admin

1. Usuario navega al detalle de un material
2. Selecciona pestaña "Estadísticas" o botón "Ver métricas"
3. App muestra dashboard con:
   - Total de visualizaciones
   - Tasa de completitud (%)
   - Promedio de calificaciones en quizzes
   - Total de intentos de evaluación
4. Puede ver gráficos de tendencia temporal (opcional)
5. Puede exportar datos (funcionalidad futura)

### Estudiante

1. Usuario accede a material que ha completado
2. Ve sus propias estadísticas personales:
   - Progreso actual
   - Intentos de quiz realizados
   - Mejor calificación obtenida
   - Tiempo invertido

## Flujo de Datos (Técnico)

### Endpoints Involucrados

| Endpoint | Método | Descripción | Estado |
|----------|--------|-------------|--------|
| `/v1/materials/:id/stats` | GET | Obtiene estadísticas agregadas del material | ✅ Listo |
| `/v1/materials/:id` | GET | Metadata del material (para contexto) | ✅ Listo |

### Request/Response (TypeScript)

#### Request
```typescript
// GET /v1/materials/:id/stats
// Headers
{
  Authorization: "Bearer {jwt_token}"
}

// Query params (opcional - futuro)
interface StatsFilters {
  date_from?: string;  // ISO8601
  date_to?: string;    // ISO8601
  academic_unit_id?: string; // Filtrar por unidad
}
```

#### Response
```typescript
interface MaterialStatsResponse {
  material_id: string;
  material_title: string; // Para contexto

  // Métricas de visualización
  total_views: number; // Número de usuarios únicos que vieron
  completion_rate: number; // Porcentaje (0-100)
  users_completed: number; // Usuarios que llegaron a 100%

  // Métricas de evaluación
  average_score: number; // Promedio de calificaciones (0-100)
  total_attempts: number; // Total de intentos de quiz
  unique_students_attempted: number; // Estudiantes únicos que intentaron
  pass_rate: number; // Porcentaje de intentos aprobados

  // Metadatos
  generated_at: string; // ISO8601
}
```

#### Ejemplo de respuesta
```json
{
  "material_id": "123e4567-e89b-12d3-a456-426614174000",
  "material_title": "Álgebra Básica - Grado 5",
  "total_views": 150,
  "completion_rate": 65.5,
  "users_completed": 98,
  "average_score": 78.2,
  "total_attempts": 245,
  "unique_students_attempted": 120,
  "pass_rate": 71.4,
  "generated_at": "2024-12-23T15:30:00Z"
}
```

### Lógica de Negocio

**Cálculo de métricas (backend):**

```sql
-- Total views: Usuarios únicos con progreso > 0
SELECT COUNT(DISTINCT user_id)
FROM material_progress
WHERE material_id = $1 AND progress_percentage > 0;

-- Completion rate: Promedio de progreso
SELECT AVG(progress_percentage)
FROM material_progress
WHERE material_id = $1;

-- Users completed: Usuarios con 100%
SELECT COUNT(DISTINCT user_id)
FROM material_progress
WHERE material_id = $1 AND progress_percentage = 100;

-- Average score: Promedio de calificaciones
SELECT AVG(score)
FROM assessment_attempt
WHERE material_id = $1 AND completed_at IS NOT NULL;

-- Total attempts
SELECT COUNT(*)
FROM assessment_attempt
WHERE material_id = $1 AND completed_at IS NOT NULL;

-- Pass rate: Intentos con passed=true
SELECT
  ROUND((COUNT(CASE WHEN passed = true THEN 1 END) * 100.0 / COUNT(*)), 2)
FROM assessment_attempt
WHERE material_id = $1 AND completed_at IS NOT NULL;
```

## Visualización de Datos

### Componentes UI Recomendados

#### 1. Cards de Métricas Principales
```typescript
interface MetricCard {
  title: string;
  value: number | string;
  icon: string;
  color: 'blue' | 'green' | 'orange' | 'red';
  trend?: {
    direction: 'up' | 'down';
    percentage: number;
  };
}

const metricCards: MetricCard[] = [
  {
    title: 'Total de Visualizaciones',
    value: stats.total_views,
    icon: '👁️',
    color: 'blue'
  },
  {
    title: 'Tasa de Completitud',
    value: `${stats.completion_rate.toFixed(1)}%`,
    icon: '✅',
    color: 'green'
  },
  {
    title: 'Promedio de Calificaciones',
    value: `${stats.average_score.toFixed(1)}%`,
    icon: '📊',
    color: 'orange'
  },
  {
    title: 'Tasa de Aprobación',
    value: `${stats.pass_rate.toFixed(1)}%`,
    icon: '🎯',
    color: stats.pass_rate >= 70 ? 'green' : 'red'
  }
];
```

#### 2. Gráfico de Progreso
```typescript
// Gráfico de distribución de progreso (opcional - requiere endpoint adicional)
interface ProgressDistribution {
  range: string; // "0-25%", "26-50%", "51-75%", "76-100%"
  count: number;
}

// Renderizar como bar chart o pie chart
```

#### 3. Tabla de Desglose
```typescript
interface StatsBreakdown {
  label: string;
  value: number;
  percentage?: number;
}

const breakdown: StatsBreakdown[] = [
  {
    label: 'Estudiantes que iniciaron',
    value: stats.total_views,
    percentage: 100
  },
  {
    label: 'Estudiantes que completaron',
    value: stats.users_completed,
    percentage: (stats.users_completed / stats.total_views) * 100
  },
  {
    label: 'Estudiantes que intentaron quiz',
    value: stats.unique_students_attempted,
    percentage: (stats.unique_students_attempted / stats.total_views) * 100
  }
];
```

### Formatos de Números

```typescript
// Utilidades de formateo
const formatNumber = (num: number): string => {
  return new Intl.NumberFormat('es-CO').format(num);
};

const formatPercentage = (num: number, decimals: number = 1): string => {
  return `${num.toFixed(decimals)}%`;
};

const formatScore = (score: number): string => {
  const color = score >= 70 ? 'green' : score >= 50 ? 'orange' : 'red';
  return `<span class="text-${color}">${score.toFixed(1)}</span>`;
};
```

### Indicadores Visuales

```typescript
// Semáforo para métricas
const getScoreColor = (score: number): string => {
  if (score >= 80) return 'bg-green-500'; // Excelente
  if (score >= 60) return 'bg-yellow-500'; // Bueno
  if (score >= 40) return 'bg-orange-500'; // Mejorable
  return 'bg-red-500'; // Crítico
};

const getCompletionBadge = (rate: number): string => {
  if (rate >= 75) return '🏆 Excelente';
  if (rate >= 50) return '👍 Bueno';
  if (rate >= 25) return '⚠️ Mejorable';
  return '❌ Bajo';
};
```

## Consideraciones de UX

### Skeleton Loaders

```typescript
const StatsSkeleton = () => (
  <div className="space-y-4">
    {/* Cards de métricas */}
    <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
      {[1, 2, 3, 4].map(i => (
        <div key={i} className="animate-pulse">
          <div className="h-24 bg-gray-200 rounded-lg"></div>
        </div>
      ))}
    </div>

    {/* Gráfico */}
    <div className="animate-pulse">
      <div className="h-64 bg-gray-200 rounded-lg"></div>
    </div>
  </div>
);
```

### Estados Vacíos

```typescript
// Sin datos aún
if (stats.total_views === 0) {
  return (
    <EmptyState
      icon="📊"
      title="Sin estadísticas aún"
      description="Este material aún no ha sido visualizado por estudiantes."
      action={
        <Button onClick={() => shareMaterial(material.id)}>
          Compartir Material
        </Button>
      }
    />
  );
}

// Sin evaluaciones
if (stats.total_attempts === 0) {
  return (
    <Alert variant="info">
      <AlertIcon icon="ℹ️" />
      <AlertDescription>
        El material ha sido visto por {stats.total_views} estudiantes,
        pero aún no han realizado evaluaciones.
      </AlertDescription>
    </Alert>
  );
}
```

### Actualización en Tiempo Real

```typescript
// Pull-to-refresh
const [refreshing, setRefreshing] = useState(false);

const handleRefresh = async () => {
  setRefreshing(true);
  try {
    const freshStats = await api.get(`/v1/materials/${materialId}/stats`);
    setStats(freshStats);
  } finally {
    setRefreshing(false);
  }
};

// Mostrar timestamp de última actualización
<Text className="text-gray-500 text-sm">
  Última actualización: {formatRelativeTime(stats.generated_at)}
</Text>
```

### Tooltips Informativos

```typescript
const MetricTooltip = ({ metric, description }: { metric: string; description: string }) => (
  <Tooltip content={description}>
    <QuestionCircleIcon className="w-4 h-4 text-gray-400 ml-1" />
  </Tooltip>
);

// Uso
<MetricCard
  title={
    <>
      Tasa de Completitud
      <MetricTooltip
        metric="completion_rate"
        description="Porcentaje promedio de progreso de todos los estudiantes que iniciaron este material."
      />
    </>
  }
  value={`${stats.completion_rate}%`}
/>
```

## Almacenamiento Local

### Cache de Estadísticas

```typescript
interface CachedStats {
  materialId: string;
  stats: MaterialStatsResponse;
  cachedAt: number; // Unix timestamp
}

// Storage key
const STATS_CACHE_KEY = 'material_stats_cache';

// Guardar en cache
const cacheStats = (materialId: string, stats: MaterialStatsResponse) => {
  const cached: CachedStats = {
    materialId,
    stats,
    cachedAt: Date.now()
  };

  localStorage.setItem(
    `${STATS_CACHE_KEY}_${materialId}`,
    JSON.stringify(cached)
  );
};

// Leer de cache
const getCachedStats = (materialId: string): MaterialStatsResponse | null => {
  const raw = localStorage.getItem(`${STATS_CACHE_KEY}_${materialId}`);
  if (!raw) return null;

  const cached: CachedStats = JSON.parse(raw);

  // Validar TTL
  const TTL_MS = 5 * 60 * 1000; // 5 minutos
  if (Date.now() - cached.cachedAt > TTL_MS) {
    return null; // Cache expirado
  }

  return cached.stats;
};
```

### TTL Recomendado

```typescript
const STATS_TTL = {
  local_storage: 5 * 60 * 1000,      // 5 minutos
  memory_cache: 2 * 60 * 1000,       // 2 minutos
  background_refresh: 10 * 60 * 1000 // 10 minutos
};

// Estrategia: Stale-While-Revalidate
const useStatsWithCache = (materialId: string) => {
  const [stats, setStats] = useState<MaterialStatsResponse | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // 1. Cargar de cache inmediatamente
    const cached = getCachedStats(materialId);
    if (cached) {
      setStats(cached);
      setLoading(false);
    }

    // 2. Refrescar en background
    api.get(`/v1/materials/${materialId}/stats`)
      .then(freshStats => {
        setStats(freshStats);
        cacheStats(materialId, freshStats);
      })
      .finally(() => setLoading(false));

  }, [materialId]);

  return { stats, loading };
};
```

### Invalidación de Cache

```typescript
// Invalidar cuando se crea un nuevo intento de quiz
const onQuizAttemptCompleted = (materialId: string) => {
  // Limpiar cache local
  localStorage.removeItem(`${STATS_CACHE_KEY}_${materialId}`);

  // Refrescar stats
  fetchStats(materialId);
};

// Invalidar cuando se actualiza progreso
const onProgressUpdated = (materialId: string) => {
  // Si llegó a 100%, invalidar cache
  localStorage.removeItem(`${STATS_CACHE_KEY}_${materialId}`);
};
```

## Código de Ejemplo (Mobile - TypeScript)

### Hook Personalizado

```typescript
import { useState, useEffect } from 'react';
import { api } from '@/services/api';

interface UseMaterialStatsOptions {
  autoRefresh?: boolean;
  refreshInterval?: number;
}

export const useMaterialStats = (
  materialId: string,
  options: UseMaterialStatsOptions = {}
) => {
  const { autoRefresh = false, refreshInterval = 60000 } = options;

  const [stats, setStats] = useState<MaterialStatsResponse | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  const fetchStats = async () => {
    try {
      setLoading(true);

      // Intentar cache primero
      const cached = getCachedStats(materialId);
      if (cached) {
        setStats(cached);
        setLoading(false);
      }

      // Fetch fresh data
      const response = await api.get(`/v1/materials/${materialId}/stats`);
      setStats(response.data);
      cacheStats(materialId, response.data);
      setError(null);

    } catch (err) {
      setError(err as Error);
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    fetchStats();

    // Auto-refresh
    if (autoRefresh) {
      const interval = setInterval(fetchStats, refreshInterval);
      return () => clearInterval(interval);
    }
  }, [materialId, autoRefresh, refreshInterval]);

  return {
    stats,
    loading,
    error,
    refresh: fetchStats
  };
};
```

### Componente de Estadísticas

```typescript
import React from 'react';
import { View, Text, ScrollView, RefreshControl } from 'react-native';
import { useMaterialStats } from '@/hooks/useMaterialStats';
import { MetricCard } from '@/components/stats/MetricCard';
import { StatsSkeleton } from '@/components/stats/StatsSkeleton';
import { EmptyState } from '@/components/EmptyState';

interface MaterialStatsScreenProps {
  materialId: string;
}

export const MaterialStatsScreen: React.FC<MaterialStatsScreenProps> = ({
  materialId
}) => {
  const { stats, loading, error, refresh } = useMaterialStats(materialId, {
    autoRefresh: false
  });

  const [refreshing, setRefreshing] = useState(false);

  const handleRefresh = async () => {
    setRefreshing(true);
    await refresh();
    setRefreshing(false);
  };

  if (loading && !stats) {
    return <StatsSkeleton />;
  }

  if (error) {
    return (
      <EmptyState
        icon="⚠️"
        title="Error al cargar estadísticas"
        description={error.message}
        action={
          <Button onPress={refresh}>Reintentar</Button>
        }
      />
    );
  }

  if (!stats || stats.total_views === 0) {
    return (
      <EmptyState
        icon="📊"
        title="Sin estadísticas aún"
        description="Este material aún no ha sido visualizado por estudiantes."
      />
    );
  }

  return (
    <ScrollView
      refreshControl={
        <RefreshControl refreshing={refreshing} onRefresh={handleRefresh} />
      }
    >
      {/* Header */}
      <View className="p-4 bg-white border-b border-gray-200">
        <Text className="text-xl font-bold">{stats.material_title}</Text>
        <Text className="text-sm text-gray-500">
          Actualizado: {formatRelativeTime(stats.generated_at)}
        </Text>
      </View>

      {/* Métricas principales */}
      <View className="grid grid-cols-2 gap-4 p-4">
        <MetricCard
          title="Visualizaciones"
          value={stats.total_views}
          icon="👁️"
          color="blue"
        />

        <MetricCard
          title="Completitud"
          value={`${stats.completion_rate.toFixed(1)}%`}
          icon="✅"
          color="green"
          subtitle={`${stats.users_completed} completados`}
        />

        <MetricCard
          title="Calificación Promedio"
          value={`${stats.average_score.toFixed(1)}%`}
          icon="📊"
          color="orange"
          subtitle={`${stats.total_attempts} intentos`}
        />

        <MetricCard
          title="Tasa de Aprobación"
          value={`${stats.pass_rate.toFixed(1)}%`}
          icon="🎯"
          color={stats.pass_rate >= 70 ? 'green' : 'red'}
        />
      </View>

      {/* Desglose detallado */}
      <View className="p-4 bg-gray-50">
        <Text className="text-lg font-semibold mb-3">Desglose</Text>

        <StatsRow
          label="Estudiantes que iniciaron"
          value={stats.total_views}
          percentage={100}
        />

        <StatsRow
          label="Estudiantes que completaron"
          value={stats.users_completed}
          percentage={(stats.users_completed / stats.total_views) * 100}
        />

        <StatsRow
          label="Estudiantes que intentaron quiz"
          value={stats.unique_students_attempted}
          percentage={(stats.unique_students_attempted / stats.total_views) * 100}
        />
      </View>
    </ScrollView>
  );
};

// Componente auxiliar
const StatsRow = ({ label, value, percentage }: {
  label: string;
  value: number;
  percentage: number;
}) => (
  <View className="flex-row justify-between py-2 border-b border-gray-200">
    <Text className="text-gray-700">{label}</Text>
    <View className="flex-row items-center">
      <Text className="font-semibold mr-2">{value}</Text>
      <Text className="text-sm text-gray-500">({percentage.toFixed(1)}%)</Text>
    </View>
  </View>
);
```

### Formateo de Fechas

```typescript
import { formatDistanceToNow } from 'date-fns';
import { es } from 'date-fns/locale';

export const formatRelativeTime = (isoDate: string): string => {
  return formatDistanceToNow(new Date(isoDate), {
    addSuffix: true,
    locale: es
  });
};

// Ejemplos:
// "hace 5 minutos"
// "hace 2 horas"
// "hace 3 días"
```

## Notas de Implementación

### Optimizaciones de Performance

1. **Cache de 5 minutos** para estadísticas (datos agregados no cambian constantemente)
2. **Stale-while-revalidate**: Mostrar cache mientras se actualiza en background
3. **Debounce en pull-to-refresh** para evitar llamadas múltiples
4. **Lazy loading** de gráficos complejos (solo renderizar cuando sean visibles)

### Manejo de Errores

```typescript
// Error: Material no encontrado
if (error.response?.status === 404) {
  showError('Material no encontrado');
  navigation.goBack();
}

// Error: Sin permisos (estudiante intentando ver stats de otro)
if (error.response?.status === 403) {
  showError('No tienes permisos para ver estas estadísticas');
}

// Error: Servicio no disponible
if (error.response?.status === 503) {
  showError('El servicio de estadísticas está temporalmente no disponible');
}
```

### Accesibilidad

```typescript
// Etiquetas descriptivas para lectores de pantalla
<MetricCard
  title="Tasa de Completitud"
  value={`${stats.completion_rate}%`}
  accessibilityLabel={`Tasa de completitud: ${stats.completion_rate} por ciento`}
  accessibilityHint="Porcentaje promedio de progreso de los estudiantes"
/>
```

### Testing

```typescript
describe('MaterialStatsScreen', () => {
  it('should display stats correctly', async () => {
    const mockStats: MaterialStatsResponse = {
      material_id: '123',
      material_title: 'Test Material',
      total_views: 100,
      completion_rate: 75.5,
      users_completed: 75,
      average_score: 82.3,
      total_attempts: 150,
      unique_students_attempted: 90,
      pass_rate: 68.5,
      generated_at: new Date().toISOString()
    };

    mockApi.get.mockResolvedValue({ data: mockStats });

    const { getByText } = render(
      <MaterialStatsScreen materialId="123" />
    );

    await waitFor(() => {
      expect(getByText('100')).toBeTruthy(); // total_views
      expect(getByText('75.5%')).toBeTruthy(); // completion_rate
      expect(getByText('82.3%')).toBeTruthy(); // average_score
    });
  });

  it('should handle empty state', async () => {
    mockApi.get.mockResolvedValue({
      data: { ...mockStats, total_views: 0 }
    });

    const { getByText } = render(
      <MaterialStatsScreen materialId="123" />
    );

    await waitFor(() => {
      expect(getByText('Sin estadísticas aún')).toBeTruthy();
    });
  });
});
```

### Mejoras Futuras

1. **Gráficos de tendencia temporal**: Mostrar evolución de métricas en el tiempo
2. **Comparación entre materiales**: Ver qué materiales tienen mejor rendimiento
3. **Filtros por unidad académica**: Ver stats solo de ciertos grupos/cursos
4. **Exportar a CSV/PDF**: Permitir descarga de reportes
5. **Notificaciones**: Alertar cuando un material tiene baja tasa de aprobación
6. **Predicciones**: ML para predecir qué estudiantes necesitan ayuda

---

**Resumen:** Este RFC documenta la implementación de estadísticas por material individual, proporcionando métricas clave sobre uso y rendimiento. El endpoint está listo en backend, requiere implementación de UI con cache de 5 minutos y manejo apropiado de estados vacíos.
