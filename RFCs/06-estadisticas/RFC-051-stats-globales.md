# RFC-051: Estadísticas Globales del Sistema

## Metadata
- **ID:** RFC-051
- **Proceso:** Estadísticas
- **Subproceso:** Estadísticas Globales
- **Prioridad:** Media
- **Dependencias:** RFC-050 (Stats por Material)
- **Estado API:** ✅ Listo
- **Endpoints:** API Mobile
- **Restricción:** Solo usuarios con rol Admin o Super Admin

## Descripción

Dashboard de métricas globales del sistema educativo, mostrando información agregada de todos los materiales, evaluaciones y actividad de estudiantes. Permite a administradores monitorear la salud general de la plataforma y tomar decisiones basadas en datos.

## Flujo de Usuario (UX)

### Administrador

1. Usuario con rol admin/super_admin accede al panel de administración
2. Navega a sección "Estadísticas Globales" o "Dashboard"
3. App muestra métricas del sistema:
   - Total de materiales en la plataforma
   - Total de intentos de evaluaciones realizados
   - Promedio global de calificaciones
   - Tendencias de uso (opcional)
4. Puede filtrar por periodo de tiempo (futuro)
5. Puede exportar reportes (futuro)
6. Puede hacer drill-down a stats específicas de materiales

### Estudiante/Docente

- NO tiene acceso a este endpoint
- Si intenta acceder: `403 Forbidden`
- UI debe ocultar esta opción para roles no autorizados

## Flujo de Datos (Técnico)

### Endpoints Involucrados

| Endpoint | Método | Descripción | Estado | Restricción |
|----------|--------|-------------|--------|-------------|
| `/v1/stats/global` | GET | Estadísticas globales del sistema | ✅ Listo | Admin+ |

### Request/Response (TypeScript)

#### Request
```typescript
// GET /v1/stats/global
// Headers
{
  Authorization: "Bearer {jwt_token}" // Debe ser admin o super_admin
}

// Query params (futuro - filtros temporales)
interface GlobalStatsFilters {
  date_from?: string;  // ISO8601
  date_to?: string;    // ISO8601
  school_id?: string;  // Filtrar por escuela específica
}
```

#### Response
```typescript
interface GlobalStatsResponse {
  // Métricas de contenido
  total_materials: number; // Total de materiales NO eliminados
  total_materials_ready: number; // Materiales procesados y listos
  total_materials_processing: number; // En procesamiento
  total_materials_failed: number; // Fallidos

  // Métricas de evaluaciones
  total_attempts: number; // Total de intentos de quiz
  total_unique_students: number; // Estudiantes únicos que intentaron
  global_average_score: number; // Promedio global de calificaciones (0-100)
  global_pass_rate: number; // Porcentaje de intentos aprobados

  // Métricas de actividad (opcional - futuro)
  total_users?: number; // Total de usuarios activos
  total_schools?: number; // Total de escuelas
  materials_created_last_30_days?: number;

  // Metadatos
  generated_at: string; // ISO8601
  period?: {
    from: string; // ISO8601
    to: string;   // ISO8601
  };
}
```

#### Ejemplo de respuesta
```json
{
  "total_materials": 250,
  "total_materials_ready": 230,
  "total_materials_processing": 15,
  "total_materials_failed": 5,
  "total_attempts": 1500,
  "total_unique_students": 350,
  "global_average_score": 75.8,
  "global_pass_rate": 68.3,
  "generated_at": "2024-12-23T16:00:00Z"
}
```

### Lógica de Negocio

**Cálculo de métricas (backend):**

```sql
-- Total materials (no eliminados)
SELECT COUNT(*)
FROM materials
WHERE deleted_at IS NULL;

-- Materials por status
SELECT
  COUNT(*) FILTER (WHERE status = 'ready') as ready,
  COUNT(*) FILTER (WHERE status = 'processing') as processing,
  COUNT(*) FILTER (WHERE status = 'failed') as failed
FROM materials
WHERE deleted_at IS NULL;

-- Total attempts
SELECT COUNT(*)
FROM assessment_attempt
WHERE completed_at IS NOT NULL;

-- Unique students
SELECT COUNT(DISTINCT student_id)
FROM assessment_attempt
WHERE completed_at IS NOT NULL;

-- Global average score
SELECT AVG(score)
FROM assessment_attempt
WHERE completed_at IS NOT NULL;

-- Global pass rate
SELECT
  ROUND((COUNT(CASE WHEN passed = true THEN 1 END) * 100.0 / COUNT(*)), 2)
FROM assessment_attempt
WHERE completed_at IS NOT NULL;
```

## Visualización de Datos

### Componentes UI Recomendados

#### 1. Panel de Métricas Principales (Grid)

```typescript
interface DashboardMetric {
  title: string;
  value: number | string;
  subtitle?: string;
  icon: string;
  color: 'blue' | 'green' | 'orange' | 'red' | 'purple';
  change?: {
    direction: 'up' | 'down';
    percentage: number;
    label: string;
  };
}

const dashboardMetrics: DashboardMetric[] = [
  {
    title: 'Total de Materiales',
    value: stats.total_materials,
    subtitle: `${stats.total_materials_ready} listos`,
    icon: '📚',
    color: 'blue'
  },
  {
    title: 'Evaluaciones Realizadas',
    value: formatNumber(stats.total_attempts),
    subtitle: `${stats.total_unique_students} estudiantes`,
    icon: '📝',
    color: 'purple'
  },
  {
    title: 'Promedio Global',
    value: `${stats.global_average_score.toFixed(1)}%`,
    icon: '📊',
    color: stats.global_average_score >= 70 ? 'green' : 'orange'
  },
  {
    title: 'Tasa de Aprobación',
    value: `${stats.global_pass_rate.toFixed(1)}%`,
    icon: '🎯',
    color: stats.global_pass_rate >= 70 ? 'green' : 'red'
  }
];
```

#### 2. Gráfico de Estado de Materiales (Pie Chart)

```typescript
interface MaterialStatusData {
  label: string;
  value: number;
  color: string;
  percentage: number;
}

const materialStatusData: MaterialStatusData[] = [
  {
    label: 'Listos',
    value: stats.total_materials_ready,
    color: '#10b981', // green
    percentage: (stats.total_materials_ready / stats.total_materials) * 100
  },
  {
    label: 'Procesando',
    value: stats.total_materials_processing,
    color: '#f59e0b', // orange
    percentage: (stats.total_materials_processing / stats.total_materials) * 100
  },
  {
    label: 'Fallidos',
    value: stats.total_materials_failed,
    color: '#ef4444', // red
    percentage: (stats.total_materials_failed / stats.total_materials) * 100
  }
];
```

#### 3. Indicadores de Salud del Sistema

```typescript
interface HealthIndicator {
  metric: string;
  status: 'excellent' | 'good' | 'warning' | 'critical';
  value: number;
  threshold: {
    excellent: number;
    good: number;
    warning: number;
  };
}

const getHealthStatus = (value: number, thresholds: any): HealthIndicator['status'] => {
  if (value >= thresholds.excellent) return 'excellent';
  if (value >= thresholds.good) return 'good';
  if (value >= thresholds.warning) return 'warning';
  return 'critical';
};

const healthIndicators: HealthIndicator[] = [
  {
    metric: 'Tasa de Aprobación',
    value: stats.global_pass_rate,
    status: getHealthStatus(stats.global_pass_rate, {
      excellent: 80,
      good: 70,
      warning: 50
    }),
    threshold: { excellent: 80, good: 70, warning: 50 }
  },
  {
    metric: 'Promedio de Calificaciones',
    value: stats.global_average_score,
    status: getHealthStatus(stats.global_average_score, {
      excellent: 85,
      good: 75,
      warning: 60
    }),
    threshold: { excellent: 85, good: 75, warning: 60 }
  },
  {
    metric: 'Tasa de Éxito de Procesamiento',
    value: (stats.total_materials_ready / stats.total_materials) * 100,
    status: getHealthStatus(
      (stats.total_materials_ready / stats.total_materials) * 100,
      { excellent: 95, good: 90, warning: 80 }
    ),
    threshold: { excellent: 95, good: 90, warning: 80 }
  }
];

// Renderizar indicador
const HealthBadge = ({ status }: { status: HealthIndicator['status'] }) => {
  const config = {
    excellent: { icon: '🟢', label: 'Excelente', color: 'green' },
    good: { icon: '🟡', label: 'Bueno', color: 'yellow' },
    warning: { icon: '🟠', label: 'Atención', color: 'orange' },
    critical: { icon: '🔴', label: 'Crítico', color: 'red' }
  };

  const { icon, label, color } = config[status];

  return (
    <span className={`inline-flex items-center px-3 py-1 rounded-full bg-${color}-100 text-${color}-800`}>
      {icon} {label}
    </span>
  );
};
```

### Formatos y Utilidades

```typescript
// Formatear números grandes
const formatLargeNumber = (num: number): string => {
  if (num >= 1000000) {
    return `${(num / 1000000).toFixed(1)}M`;
  }
  if (num >= 1000) {
    return `${(num / 1000).toFixed(1)}K`;
  }
  return num.toString();
};

// Ejemplos:
// 1500 → "1.5K"
// 1234567 → "1.2M"

// Comparación con periodo anterior (futuro)
interface Comparison {
  current: number;
  previous: number;
  change: number;
  changePercentage: number;
  trend: 'up' | 'down' | 'stable';
}

const calculateComparison = (current: number, previous: number): Comparison => {
  const change = current - previous;
  const changePercentage = previous > 0 ? (change / previous) * 100 : 0;

  return {
    current,
    previous,
    change,
    changePercentage,
    trend: change > 0 ? 'up' : change < 0 ? 'down' : 'stable'
  };
};

// Renderizar cambio
const ChangeIndicator = ({ comparison }: { comparison: Comparison }) => (
  <div className={`flex items-center text-sm ${
    comparison.trend === 'up' ? 'text-green-600' :
    comparison.trend === 'down' ? 'text-red-600' :
    'text-gray-600'
  }`}>
    {comparison.trend === 'up' && '↑'}
    {comparison.trend === 'down' && '↓'}
    {comparison.trend === 'stable' && '→'}
    <span className="ml-1">
      {Math.abs(comparison.changePercentage).toFixed(1)}%
    </span>
  </div>
);
```

## Consideraciones de UX

### Autorización en UI

```typescript
// Verificar rol antes de mostrar opción
const canViewGlobalStats = (user: User): boolean => {
  return ['admin', 'super_admin'].includes(user.role);
};

// En navegación
{canViewGlobalStats(currentUser) && (
  <MenuItem
    icon="📊"
    label="Estadísticas Globales"
    onPress={() => navigation.navigate('GlobalStats')}
  />
)}

// En pantalla
if (!canViewGlobalStats(currentUser)) {
  return (
    <UnauthorizedScreen
      message="No tienes permisos para ver estadísticas globales"
    />
  );
}
```

### Skeleton Loaders

```typescript
const GlobalStatsSkeleton = () => (
  <div className="space-y-6">
    {/* Métricas principales */}
    <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
      {[1, 2, 3, 4].map(i => (
        <div key={i} className="animate-pulse">
          <div className="h-32 bg-gray-200 rounded-lg"></div>
        </div>
      ))}
    </div>

    {/* Gráficos */}
    <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
      <div className="animate-pulse">
        <div className="h-64 bg-gray-200 rounded-lg"></div>
      </div>
      <div className="animate-pulse">
        <div className="h-64 bg-gray-200 rounded-lg"></div>
      </div>
    </div>

    {/* Tabla */}
    <div className="animate-pulse">
      <div className="h-48 bg-gray-200 rounded-lg"></div>
    </div>
  </div>
);
```

### Estados Especiales

```typescript
// Sistema recién instalado (sin datos)
if (stats.total_materials === 0) {
  return (
    <EmptyState
      icon="🚀"
      title="Bienvenido al Dashboard"
      description="Aún no hay datos para mostrar. Los docentes deben comenzar a subir materiales."
      action={
        <Button onClick={() => navigation.navigate('CreateMaterial')}>
          Subir Primer Material
        </Button>
      }
    />
  );
}

// Alerta de salud crítica
if (stats.global_pass_rate < 50) {
  return (
    <Alert variant="warning" className="mb-4">
      <AlertIcon icon="⚠️" />
      <AlertTitle>Atención: Tasa de Aprobación Baja</AlertTitle>
      <AlertDescription>
        La tasa de aprobación global es de {stats.global_pass_rate.toFixed(1)}%,
        por debajo del umbral recomendado de 70%.
        Considera revisar la dificultad de los materiales.
      </AlertDescription>
    </Alert>
  );
}

// Muchos materiales fallidos
const failureRate = (stats.total_materials_failed / stats.total_materials) * 100;
if (failureRate > 10) {
  return (
    <Alert variant="error" className="mb-4">
      <AlertIcon icon="🔴" />
      <AlertTitle>Problema en el Procesamiento</AlertTitle>
      <AlertDescription>
        {stats.total_materials_failed} materiales ({failureRate.toFixed(1)}%)
        han fallado en el procesamiento. Contacta al equipo técnico.
      </AlertDescription>
      <AlertAction>
        <Button onClick={() => navigation.navigate('FailedMaterials')}>
          Ver Materiales Fallidos
        </Button>
      </AlertAction>
    </Alert>
  );
}
```

### Actualización y Refresh

```typescript
// Auto-refresh cada 5 minutos
const [autoRefresh, setAutoRefresh] = useState(true);

useEffect(() => {
  if (!autoRefresh) return;

  const interval = setInterval(() => {
    fetchGlobalStats();
  }, 5 * 60 * 1000); // 5 minutos

  return () => clearInterval(interval);
}, [autoRefresh]);

// Toggle auto-refresh
<Switch
  label="Actualización automática"
  checked={autoRefresh}
  onChange={setAutoRefresh}
/>

// Manual refresh con timestamp
<RefreshButton
  onPress={handleRefresh}
  lastUpdated={stats.generated_at}
/>
```

## Almacenamiento Local

### Cache de Estadísticas Globales

```typescript
const GLOBAL_STATS_CACHE_KEY = 'global_stats_cache';

interface CachedGlobalStats {
  stats: GlobalStatsResponse;
  cachedAt: number; // Unix timestamp
}

// Guardar en cache
const cacheGlobalStats = (stats: GlobalStatsResponse) => {
  const cached: CachedGlobalStats = {
    stats,
    cachedAt: Date.now()
  };

  localStorage.setItem(GLOBAL_STATS_CACHE_KEY, JSON.stringify(cached));
};

// Leer de cache
const getCachedGlobalStats = (): GlobalStatsResponse | null => {
  const raw = localStorage.getItem(GLOBAL_STATS_CACHE_KEY);
  if (!raw) return null;

  const cached: CachedGlobalStats = JSON.parse(raw);

  // Validar TTL (10 minutos para stats globales)
  const TTL_MS = 10 * 60 * 1000;
  if (Date.now() - cached.cachedAt > TTL_MS) {
    return null; // Cache expirado
  }

  return cached.stats;
};
```

### TTL Recomendado

```typescript
const GLOBAL_STATS_TTL = {
  local_storage: 10 * 60 * 1000,    // 10 minutos (datos más estables)
  memory_cache: 5 * 60 * 1000,      // 5 minutos
  background_refresh: 15 * 60 * 1000 // 15 minutos
};

// Estrategia: Cache con revalidación periódica
const useGlobalStatsWithCache = () => {
  const [stats, setStats] = useState<GlobalStatsResponse | null>(null);
  const [loading, setLoading] = useState(true);
  const [lastFetch, setLastFetch] = useState<Date | null>(null);

  const fetchStats = async (forceRefresh = false) => {
    try {
      // 1. Si no es force, intentar cache primero
      if (!forceRefresh) {
        const cached = getCachedGlobalStats();
        if (cached) {
          setStats(cached);
          setLoading(false);
          setLastFetch(new Date(cached.generated_at));
          return;
        }
      }

      // 2. Fetch fresh data
      setLoading(true);
      const response = await api.get('/v1/stats/global');
      setStats(response.data);
      cacheGlobalStats(response.data);
      setLastFetch(new Date());

    } catch (error) {
      // Si falla, intentar servir cache expirado como fallback
      const cached = localStorage.getItem(GLOBAL_STATS_CACHE_KEY);
      if (cached) {
        const parsedCache = JSON.parse(cached);
        setStats(parsedCache.stats);
      } else {
        throw error;
      }
    } finally {
      setLoading(false);
    }
  };

  return { stats, loading, lastFetch, refresh: fetchStats };
};
```

### Invalidación de Cache

```typescript
// Invalidar cuando se completa procesamiento de material
const onMaterialProcessed = () => {
  localStorage.removeItem(GLOBAL_STATS_CACHE_KEY);
  fetchGlobalStats();
};

// Invalidar cuando se completa un quiz
const onAssessmentAttemptCompleted = () => {
  localStorage.removeItem(GLOBAL_STATS_CACHE_KEY);
  // No refrescar inmediatamente (batch updates)
};

// Invalidar al cambiar de día (para stats diarias futuras)
useEffect(() => {
  const midnight = new Date();
  midnight.setHours(24, 0, 0, 0);

  const msUntilMidnight = midnight.getTime() - Date.now();

  const timeout = setTimeout(() => {
    localStorage.removeItem(GLOBAL_STATS_CACHE_KEY);
    fetchGlobalStats();
  }, msUntilMidnight);

  return () => clearTimeout(timeout);
}, []);
```

## Código de Ejemplo (Mobile - TypeScript)

### Hook Personalizado

```typescript
import { useState, useEffect } from 'react';
import { api } from '@/services/api';
import { useAuth } from '@/contexts/AuthContext';

export const useGlobalStats = (autoRefresh: boolean = false) => {
  const { user } = useAuth();

  const [stats, setStats] = useState<GlobalStatsResponse | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  // Verificar autorización
  const isAuthorized = ['admin', 'super_admin'].includes(user?.role || '');

  const fetchStats = async () => {
    if (!isAuthorized) {
      setError(new Error('No autorizado'));
      setLoading(false);
      return;
    }

    try {
      setLoading(true);

      // Intentar cache primero
      const cached = getCachedGlobalStats();
      if (cached) {
        setStats(cached);
        setLoading(false);
      }

      // Fetch fresh data
      const response = await api.get('/v1/stats/global');
      setStats(response.data);
      cacheGlobalStats(response.data);
      setError(null);

    } catch (err) {
      setError(err as Error);
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    fetchStats();

    // Auto-refresh cada 5 minutos
    if (autoRefresh) {
      const interval = setInterval(fetchStats, 5 * 60 * 1000);
      return () => clearInterval(interval);
    }
  }, [isAuthorized, autoRefresh]);

  return {
    stats,
    loading,
    error,
    isAuthorized,
    refresh: fetchStats
  };
};
```

### Componente Principal

```typescript
import React from 'react';
import { View, ScrollView, RefreshControl, Text } from 'react-native';
import { useGlobalStats } from '@/hooks/useGlobalStats';
import { DashboardCard } from '@/components/stats/DashboardCard';
import { HealthIndicatorsList } from '@/components/stats/HealthIndicatorsList';
import { MaterialStatusChart } from '@/components/stats/MaterialStatusChart';
import { GlobalStatsSkeleton } from '@/components/stats/GlobalStatsSkeleton';
import { UnauthorizedScreen } from '@/components/UnauthorizedScreen';

export const GlobalStatsScreen: React.FC = () => {
  const { stats, loading, error, isAuthorized, refresh } = useGlobalStats(true);
  const [refreshing, setRefreshing] = useState(false);

  const handleRefresh = async () => {
    setRefreshing(true);
    await refresh();
    setRefreshing(false);
  };

  // Verificar autorización
  if (!isAuthorized) {
    return (
      <UnauthorizedScreen
        title="Acceso Restringido"
        message="No tienes permisos para ver estadísticas globales"
      />
    );
  }

  // Loading inicial
  if (loading && !stats) {
    return <GlobalStatsSkeleton />;
  }

  // Error
  if (error) {
    return (
      <ErrorState
        title="Error al cargar estadísticas"
        message={error.message}
        onRetry={refresh}
      />
    );
  }

  // Sin datos
  if (!stats || stats.total_materials === 0) {
    return (
      <EmptyState
        icon="🚀"
        title="Bienvenido al Dashboard"
        description="Aún no hay datos para mostrar"
      />
    );
  }

  return (
    <ScrollView
      refreshControl={
        <RefreshControl refreshing={refreshing} onRefresh={handleRefresh} />
      }
      className="bg-gray-50"
    >
      {/* Header */}
      <View className="p-4 bg-white border-b border-gray-200">
        <Text className="text-2xl font-bold">Dashboard Global</Text>
        <Text className="text-sm text-gray-500">
          Última actualización: {formatRelativeTime(stats.generated_at)}
        </Text>
      </View>

      {/* Alertas de salud */}
      {stats.global_pass_rate < 50 && (
        <Alert variant="warning" className="m-4">
          <AlertTitle>⚠️ Tasa de Aprobación Baja</AlertTitle>
          <AlertDescription>
            La tasa global es {stats.global_pass_rate.toFixed(1)}%
          </AlertDescription>
        </Alert>
      )}

      {/* Métricas principales */}
      <View className="grid grid-cols-2 md:grid-cols-4 gap-4 p-4">
        <DashboardCard
          title="Total Materiales"
          value={stats.total_materials}
          subtitle={`${stats.total_materials_ready} listos`}
          icon="📚"
          color="blue"
        />

        <DashboardCard
          title="Evaluaciones"
          value={formatLargeNumber(stats.total_attempts)}
          subtitle={`${stats.total_unique_students} estudiantes`}
          icon="📝"
          color="purple"
        />

        <DashboardCard
          title="Promedio Global"
          value={`${stats.global_average_score.toFixed(1)}%`}
          icon="📊"
          color={stats.global_average_score >= 70 ? 'green' : 'orange'}
        />

        <DashboardCard
          title="Tasa Aprobación"
          value={`${stats.global_pass_rate.toFixed(1)}%`}
          icon="🎯"
          color={stats.global_pass_rate >= 70 ? 'green' : 'red'}
        />
      </View>

      {/* Indicadores de salud */}
      <View className="p-4">
        <Text className="text-lg font-semibold mb-3">Indicadores de Salud</Text>
        <HealthIndicatorsList stats={stats} />
      </View>

      {/* Estado de materiales */}
      <View className="p-4">
        <Text className="text-lg font-semibold mb-3">Estado de Materiales</Text>
        <MaterialStatusChart
          ready={stats.total_materials_ready}
          processing={stats.total_materials_processing}
          failed={stats.total_materials_failed}
        />
      </View>

      {/* Desglose detallado */}
      <View className="p-4 bg-white mt-4">
        <Text className="text-lg font-semibold mb-3">Desglose Detallado</Text>

        <DetailRow
          label="Total de materiales"
          value={stats.total_materials}
        />
        <DetailRow
          label="Materiales listos"
          value={stats.total_materials_ready}
          percentage={(stats.total_materials_ready / stats.total_materials) * 100}
          color="green"
        />
        <DetailRow
          label="Materiales procesando"
          value={stats.total_materials_processing}
          percentage={(stats.total_materials_processing / stats.total_materials) * 100}
          color="orange"
        />
        <DetailRow
          label="Materiales fallidos"
          value={stats.total_materials_failed}
          percentage={(stats.total_materials_failed / stats.total_materials) * 100}
          color="red"
        />

        <View className="border-t border-gray-200 my-3" />

        <DetailRow
          label="Total de evaluaciones"
          value={stats.total_attempts}
        />
        <DetailRow
          label="Estudiantes únicos"
          value={stats.total_unique_students}
        />
        <DetailRow
          label="Promedio por estudiante"
          value={Math.round(stats.total_attempts / stats.total_unique_students)}
        />
      </View>
    </ScrollView>
  );
};

// Componente auxiliar
const DetailRow = ({
  label,
  value,
  percentage,
  color
}: {
  label: string;
  value: number;
  percentage?: number;
  color?: string;
}) => (
  <View className="flex-row justify-between py-3 border-b border-gray-100">
    <Text className="text-gray-700">{label}</Text>
    <View className="flex-row items-center">
      <Text className="font-semibold mr-2">{formatNumber(value)}</Text>
      {percentage !== undefined && (
        <Text className={`text-sm text-${color || 'gray'}-600`}>
          ({percentage.toFixed(1)}%)
        </Text>
      )}
    </View>
  </View>
);
```

### Componente de Gráfico de Estado

```typescript
import React from 'react';
import { View, Text } from 'react-native';
import { PieChart } from '@/components/charts/PieChart';

interface MaterialStatusChartProps {
  ready: number;
  processing: number;
  failed: number;
}

export const MaterialStatusChart: React.FC<MaterialStatusChartProps> = ({
  ready,
  processing,
  failed
}) => {
  const total = ready + processing + failed;

  const data = [
    {
      label: 'Listos',
      value: ready,
      color: '#10b981',
      percentage: (ready / total) * 100
    },
    {
      label: 'Procesando',
      value: processing,
      color: '#f59e0b',
      percentage: (processing / total) * 100
    },
    {
      label: 'Fallidos',
      value: failed,
      color: '#ef4444',
      percentage: (failed / total) * 100
    }
  ];

  return (
    <View className="bg-white p-4 rounded-lg">
      <PieChart data={data} size={200} />

      {/* Leyenda */}
      <View className="mt-4 space-y-2">
        {data.map((item, index) => (
          <View key={index} className="flex-row items-center justify-between">
            <View className="flex-row items-center">
              <View
                className="w-4 h-4 rounded mr-2"
                style={{ backgroundColor: item.color }}
              />
              <Text className="text-gray-700">{item.label}</Text>
            </View>
            <View className="flex-row items-center">
              <Text className="font-semibold mr-2">{item.value}</Text>
              <Text className="text-sm text-gray-500">
                ({item.percentage.toFixed(1)}%)
              </Text>
            </View>
          </View>
        ))}
      </View>
    </View>
  );
};
```

## Notas de Implementación

### Seguridad

```typescript
// Middleware de autorización (backend ya lo implementa)
// Frontend debe verificar antes de mostrar UI

const requireAdmin = (component: React.ComponentType) => {
  return (props: any) => {
    const { user } = useAuth();

    if (!['admin', 'super_admin'].includes(user?.role || '')) {
      return <UnauthorizedScreen />;
    }

    return React.createElement(component, props);
  };
};

// Uso
export const GlobalStatsScreen = requireAdmin(GlobalStatsScreenComponent);
```

### Manejo de Errores HTTP

```typescript
// 403 Forbidden (usuario no es admin)
if (error.response?.status === 403) {
  showError('No tienes permisos para ver estadísticas globales');
  navigation.goBack();
}

// 503 Service Unavailable
if (error.response?.status === 503) {
  showError('El servicio de estadísticas está temporalmente no disponible');
  // Ofrecer reintento
}

// 500 Internal Server Error
if (error.response?.status === 500) {
  showError('Error interno del servidor. Contacta soporte.');
  // Log para debugging
  console.error('Global stats error:', error);
}
```

### Performance

1. **Cache de 10 minutos** (datos agregados cambian lentamente)
2. **Lazy loading de gráficos** (solo renderizar cuando sean visibles)
3. **Memoización de cálculos**:

```typescript
const materialSuccessRate = useMemo(() => {
  if (!stats) return 0;
  return (stats.total_materials_ready / stats.total_materials) * 100;
}, [stats]);
```

4. **Virtualization** si hay tablas largas de desglose

### Testing

```typescript
describe('GlobalStatsScreen', () => {
  it('should deny access to non-admin users', () => {
    const { getByText } = render(
      <GlobalStatsScreen />,
      { user: { role: 'student' } }
    );

    expect(getByText('Acceso Restringido')).toBeTruthy();
  });

  it('should display stats for admin', async () => {
    mockApi.get.mockResolvedValue({
      data: {
        total_materials: 250,
        total_attempts: 1500,
        global_average_score: 75.8
      }
    });

    const { getByText } = render(
      <GlobalStatsScreen />,
      { user: { role: 'admin' } }
    );

    await waitFor(() => {
      expect(getByText('250')).toBeTruthy();
      expect(getByText('1500')).toBeTruthy();
      expect(getByText('75.8%')).toBeTruthy();
    });
  });
});
```

### Mejoras Futuras

1. **Filtros temporales**: Ver stats de últimos 7/30/90 días
2. **Comparación temporal**: Comparar con periodo anterior
3. **Drill-down**: Click en métrica para ver desglose
4. **Exportar reportes**: PDF/CSV con datos completos
5. **Gráficos de tendencia**: Evolución temporal de métricas
6. **Alertas automáticas**: Notificar cuando métricas caen bajo umbral
7. **Segmentación**: Stats por escuela, grado, materia, etc.
8. **Predictivo**: ML para predecir tendencias

---

**Resumen:** Este RFC documenta la implementación del dashboard de estadísticas globales, restringido a administradores. El endpoint está listo en backend, requiere implementación de UI con verificación de roles, cache de 10 minutos y visualizaciones apropiadas para monitoreo de salud del sistema.
