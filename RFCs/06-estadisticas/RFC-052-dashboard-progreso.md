# RFC-052: Dashboard de Progreso del Estudiante

## Metadata
- **ID:** RFC-052
- **Proceso:** Estadísticas
- **Subproceso:** Dashboard de Progreso Personal
- **Prioridad:** Media
- **Dependencias:** RFC-001 (Autenticación), RFC-024 (Sistema de Progreso), RFC-033 (Historial Intentos)
- **Estado API:** ⚠️ Parcial (requiere endpoint agregado)

## Descripción

Dashboard personalizado que muestra al estudiante un resumen de su progreso académico, incluyendo materiales completados, evaluaciones realizadas, tiempo de estudio y métricas de rendimiento. Proporciona una vista consolidada del avance del estudiante en la plataforma.

## Flujo de Usuario (UX)

### Pantalla: Dashboard de Progreso

```
┌─────────────────────────────────────────┐
│  👤 Hola, María!                        │
│  Tu progreso esta semana                │
│                                          │
│  ┌────────────────────────────────────┐ │
│  │  🏆 Resumen General                 │ │
│  │                                      │ │
│  │  ┌──────┐ ┌──────┐ ┌──────┐        │ │
│  │  │  12  │ │  8   │ │ 85%  │        │ │
│  │  │Mater.│ │Quizzes│ │Prom. │        │ │
│  │  │Leídos│ │Aprob.│ │Calif.│        │ │
│  │  └──────┘ └──────┘ └──────┘        │ │
│  └────────────────────────────────────┘ │
│                                          │
│  📈 Actividad Semanal                    │
│  ┌────────────────────────────────────┐ │
│  │ L   M   M   J   V   S   D          │ │
│  │ ▓   ▓▓  ▓   ▓▓▓ ▓▓  ░   ░          │ │
│  └────────────────────────────────────┘ │
│                                          │
│  📚 Materiales en Progreso               │
│  ┌────────────────────────────────────┐ │
│  │ Álgebra Módulo 2          75% ▓▓▓░ │ │
│  │ Historia Universal        45% ▓▓░░ │ │
│  │ Física Básica            20% ▓░░░ │ │
│  └────────────────────────────────────┘ │
│                                          │
│  📝 Evaluaciones Recientes               │
│  ┌────────────────────────────────────┐ │
│  │ Álgebra Quiz 1      ✅ 92% Aprobado │ │
│  │ Historia Quiz 2     ✅ 78% Aprobado │ │
│  │ Física Quiz 1       ❌ 55% Reprobar │ │
│  └────────────────────────────────────┘ │
│                                          │
│  🎯 Logros                               │
│  🏅 Primera evaluación    🌟 Racha 5 días │
│                                          │
└─────────────────────────────────────────┘
```

### Componentes del Dashboard

1. **Header Personalizado**
   - Saludo con nombre del estudiante
   - Periodo de tiempo seleccionable (semana, mes, total)

2. **Tarjetas de Resumen**
   - Materiales completados/en progreso
   - Evaluaciones aprobadas/totales
   - Promedio de calificaciones

3. **Gráfico de Actividad**
   - Visualización semanal/mensual
   - Días con actividad destacados

4. **Lista de Materiales en Progreso**
   - Ordenados por progreso (descendente)
   - Acceso rápido para continuar

5. **Evaluaciones Recientes**
   - Últimas 5 evaluaciones
   - Estado (aprobado/reprobado)

6. **Logros/Badges** (opcional)
   - Gamificación básica

## Flujo de Datos (Técnico)

### Endpoints Involucrados

| Endpoint | Método | Descripción | Estado |
|----------|--------|-------------|--------|
| `/v1/progress/dashboard` | GET | Obtener dashboard consolidado | ⚠️ Por implementar |
| `/v1/progress` | GET | Lista de progreso por material | ✅ Listo |
| `/v1/attempts` | GET | Historial de intentos | ✅ Listo |

### Endpoint Propuesto: Dashboard Consolidado

```http
GET /v1/progress/dashboard
Authorization: Bearer {jwt_token}
```

**Query Parameters:**
```typescript
interface DashboardQuery {
  period?: 'week' | 'month' | 'all';  // default: 'week'
}
```

### Request/Response (TypeScript)

**Response 200 OK:**
```typescript
interface DashboardResponse {
  user: {
    id: string;
    name: string;
    role: string;
  };

  period: {
    start: string;      // ISO8601
    end: string;        // ISO8601
    type: 'week' | 'month' | 'all';
  };

  summary: {
    materials_started: number;
    materials_completed: number;
    completion_rate: number;        // 0-100
    quizzes_attempted: number;
    quizzes_passed: number;
    pass_rate: number;              // 0-100
    average_score: number;          // 0-100
    total_study_time_minutes: number;
  };

  activity: ActivityDay[];          // 7 o 30 días

  materials_in_progress: MaterialProgressDTO[];

  recent_attempts: RecentAttemptDTO[];

  achievements: AchievementDTO[];
}

interface ActivityDay {
  date: string;                     // "2024-12-23"
  study_minutes: number;
  materials_accessed: number;
  quizzes_taken: number;
}

interface MaterialProgressDTO {
  material_id: string;
  title: string;
  subject: string;
  progress_percentage: number;
  last_accessed_at: string;
}

interface RecentAttemptDTO {
  attempt_id: string;
  material_id: string;
  material_title: string;
  score: number;
  passed: boolean;
  completed_at: string;
}

interface AchievementDTO {
  id: string;
  name: string;
  description: string;
  icon: string;
  unlocked_at: string;
}
```

**Ejemplo Response:**
```json
{
  "user": {
    "id": "user-123",
    "name": "María García",
    "role": "student"
  },
  "period": {
    "start": "2024-12-18",
    "end": "2024-12-24",
    "type": "week"
  },
  "summary": {
    "materials_started": 15,
    "materials_completed": 12,
    "completion_rate": 80.0,
    "quizzes_attempted": 10,
    "quizzes_passed": 8,
    "pass_rate": 80.0,
    "average_score": 85.5,
    "total_study_time_minutes": 420
  },
  "activity": [
    { "date": "2024-12-18", "study_minutes": 45, "materials_accessed": 2, "quizzes_taken": 1 },
    { "date": "2024-12-19", "study_minutes": 90, "materials_accessed": 3, "quizzes_taken": 2 },
    { "date": "2024-12-20", "study_minutes": 30, "materials_accessed": 1, "quizzes_taken": 0 },
    { "date": "2024-12-21", "study_minutes": 120, "materials_accessed": 4, "quizzes_taken": 3 },
    { "date": "2024-12-22", "study_minutes": 75, "materials_accessed": 2, "quizzes_taken": 1 },
    { "date": "2024-12-23", "study_minutes": 0, "materials_accessed": 0, "quizzes_taken": 0 },
    { "date": "2024-12-24", "study_minutes": 60, "materials_accessed": 1, "quizzes_taken": 1 }
  ],
  "materials_in_progress": [
    {
      "material_id": "mat-1",
      "title": "Álgebra Módulo 2",
      "subject": "Matemáticas",
      "progress_percentage": 75,
      "last_accessed_at": "2024-12-24T10:30:00Z"
    },
    {
      "material_id": "mat-2",
      "title": "Historia Universal",
      "subject": "Historia",
      "progress_percentage": 45,
      "last_accessed_at": "2024-12-23T15:00:00Z"
    }
  ],
  "recent_attempts": [
    {
      "attempt_id": "att-1",
      "material_id": "mat-1",
      "material_title": "Álgebra Quiz 1",
      "score": 92,
      "passed": true,
      "completed_at": "2024-12-24T11:00:00Z"
    }
  ],
  "achievements": [
    {
      "id": "first-quiz",
      "name": "Primera Evaluación",
      "description": "Completaste tu primera evaluación",
      "icon": "🏅",
      "unlocked_at": "2024-12-18T10:00:00Z"
    }
  ]
}
```

### Alternativa: Construir Dashboard en Frontend

Si el endpoint consolidado no está disponible, el frontend puede construir el dashboard combinando:

```typescript
async function buildDashboard(): Promise<DashboardData> {
  const [progressList, attemptsList] = await Promise.all([
    api.get('/v1/progress'),
    api.get('/v1/attempts')
  ]);

  // Procesar y agregar datos en frontend
  const summary = calculateSummary(progressList, attemptsList);
  const activity = calculateActivity(progressList, attemptsList);

  return {
    summary,
    activity,
    materials_in_progress: filterInProgress(progressList),
    recent_attempts: attemptsList.slice(0, 5)
  };
}
```

## Estados y Transiciones

### Estados del Dashboard

```
┌──────────┐
│ LOADING  │ → Cargando datos iniciales
└────┬─────┘
     │
     ├─── (éxito) ───►┌───────┐
     │                │ READY │ → Dashboard completo
     │                └───────┘
     │
     ├─── (vacío) ───►┌───────┐
     │                │ EMPTY │ → Sin datos para mostrar
     │                └───────┘
     │
     └─── (error) ───►┌───────┐
                      │ ERROR │ → Error al cargar
                      └───────┘
```

## Manejo de Errores

| Código | Significado | Acción UI |
|--------|-------------|-----------|
| 200 | Datos cargados | Mostrar dashboard |
| 200 (vacío) | Usuario nuevo | Mostrar onboarding |
| 401 | No autenticado | Redirigir a login |
| 500 | Error servidor | Mostrar error + retry |

### Estado Vacío (Usuario Nuevo)

```typescript
if (summary.materials_started === 0) {
  return (
    <EmptyDashboard>
      <h2>¡Bienvenido a EduGo!</h2>
      <p>Aún no has comenzado ningún material.</p>
      <Button onClick={() => navigate('/materials')}>
        Explorar Materiales
      </Button>
    </EmptyDashboard>
  );
}
```

## Consideraciones de UX

### Loading State

```typescript
function DashboardSkeleton() {
  return (
    <div className="space-y-6 p-4">
      {/* Header */}
      <div className="animate-pulse">
        <div className="h-8 bg-gray-200 rounded w-48 mb-2" />
        <div className="h-4 bg-gray-200 rounded w-32" />
      </div>

      {/* Summary Cards */}
      <div className="grid grid-cols-3 gap-4">
        {[1, 2, 3].map(i => (
          <div key={i} className="h-24 bg-gray-200 rounded-lg animate-pulse" />
        ))}
      </div>

      {/* Activity Chart */}
      <div className="h-32 bg-gray-200 rounded-lg animate-pulse" />

      {/* Materials List */}
      <div className="space-y-2">
        {[1, 2, 3].map(i => (
          <div key={i} className="h-16 bg-gray-200 rounded-lg animate-pulse" />
        ))}
      </div>
    </div>
  );
}
```

### Visualización de Actividad

```typescript
// Componente de gráfico de actividad semanal
function ActivityChart({ activity }: { activity: ActivityDay[] }) {
  const maxMinutes = Math.max(...activity.map(d => d.study_minutes), 1);

  return (
    <div className="flex items-end justify-between h-24 gap-2">
      {activity.map((day, i) => {
        const height = (day.study_minutes / maxMinutes) * 100;
        const dayName = ['L', 'M', 'M', 'J', 'V', 'S', 'D'][i];

        return (
          <div key={day.date} className="flex flex-col items-center flex-1">
            <div
              className={`w-full rounded-t ${
                day.study_minutes > 0 ? 'bg-blue-500' : 'bg-gray-200'
              }`}
              style={{ height: `${Math.max(height, 5)}%` }}
            />
            <span className="text-xs text-gray-500 mt-1">{dayName}</span>
          </div>
        );
      })}
    </div>
  );
}
```

### Pull-to-Refresh

```typescript
const [refreshing, setRefreshing] = useState(false);

const handleRefresh = async () => {
  setRefreshing(true);
  await fetchDashboard();
  setRefreshing(false);
};

<ScrollView
  refreshControl={
    <RefreshControl
      refreshing={refreshing}
      onRefresh={handleRefresh}
      colors={['#3b82f6']}
    />
  }
>
  {/* Dashboard content */}
</ScrollView>
```

### Periodo Seleccionable

```typescript
function PeriodSelector({
  value,
  onChange
}: {
  value: 'week' | 'month' | 'all';
  onChange: (period: 'week' | 'month' | 'all') => void;
}) {
  return (
    <div className="flex gap-2">
      {(['week', 'month', 'all'] as const).map(period => (
        <button
          key={period}
          onClick={() => onChange(period)}
          className={`px-3 py-1 rounded-full text-sm ${
            value === period
              ? 'bg-blue-500 text-white'
              : 'bg-gray-100 text-gray-600'
          }`}
        >
          {period === 'week' ? 'Semana' : period === 'month' ? 'Mes' : 'Todo'}
        </button>
      ))}
    </div>
  );
}
```

## Almacenamiento Local

### Cache del Dashboard

```typescript
interface DashboardCache {
  userId: string;
  period: string;
  data: DashboardResponse;
  cachedAt: number;
}

// Cache corto (2 minutos) porque datos cambian frecuentemente
const DASHBOARD_TTL = 2 * 60 * 1000;

function cacheDashboard(userId: string, period: string, data: DashboardResponse) {
  const cache: DashboardCache = {
    userId,
    period,
    data,
    cachedAt: Date.now()
  };

  localStorage.setItem(`dashboard_${userId}_${period}`, JSON.stringify(cache));
}

function getCachedDashboard(userId: string, period: string): DashboardResponse | null {
  const cached = localStorage.getItem(`dashboard_${userId}_${period}`);
  if (!cached) return null;

  const cache: DashboardCache = JSON.parse(cached);

  if (Date.now() - cache.cachedAt > DASHBOARD_TTL) {
    localStorage.removeItem(`dashboard_${userId}_${period}`);
    return null;
  }

  return cache.data;
}
```

## Código de Ejemplo (Mobile - TypeScript)

### Hook

```typescript
// hooks/useDashboard.ts
import { useState, useEffect, useCallback } from 'react';

interface UseDashboardOptions {
  period?: 'week' | 'month' | 'all';
  autoRefresh?: boolean;
  refreshInterval?: number;
}

export function useDashboard(options: UseDashboardOptions = {}) {
  const {
    period = 'week',
    autoRefresh = false,
    refreshInterval = 60000
  } = options;

  const [dashboard, setDashboard] = useState<DashboardResponse | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  const fetchDashboard = useCallback(async () => {
    try {
      setLoading(true);
      setError(null);

      const userId = getCurrentUserId();

      // Verificar cache
      const cached = getCachedDashboard(userId, period);
      if (cached) {
        setDashboard(cached);
        setLoading(false);
      }

      // Fetch fresh data
      const response = await api.get(`/v1/progress/dashboard?period=${period}`);
      setDashboard(response.data);
      cacheDashboard(userId, period, response.data);

    } catch (err) {
      // Si hay cache, usar aunque esté expirado
      const userId = getCurrentUserId();
      const stale = localStorage.getItem(`dashboard_${userId}_${period}`);
      if (stale) {
        setDashboard(JSON.parse(stale).data);
      }
      setError(err as Error);
    } finally {
      setLoading(false);
    }
  }, [period]);

  useEffect(() => {
    fetchDashboard();
  }, [fetchDashboard]);

  // Auto-refresh
  useEffect(() => {
    if (!autoRefresh) return;

    const interval = setInterval(fetchDashboard, refreshInterval);
    return () => clearInterval(interval);
  }, [autoRefresh, refreshInterval, fetchDashboard]);

  return {
    dashboard,
    loading,
    error,
    refetch: fetchDashboard,
    isEmpty: dashboard?.summary.materials_started === 0
  };
}
```

### Componente Principal

```typescript
// components/Dashboard.tsx
import React, { useState } from 'react';
import { useDashboard } from '../hooks/useDashboard';
import { SummaryCards } from './SummaryCards';
import { ActivityChart } from './ActivityChart';
import { MaterialsInProgress } from './MaterialsInProgress';
import { RecentAttempts } from './RecentAttempts';

export function Dashboard() {
  const [period, setPeriod] = useState<'week' | 'month' | 'all'>('week');
  const { dashboard, loading, error, refetch, isEmpty } = useDashboard({ period });

  if (loading && !dashboard) {
    return <DashboardSkeleton />;
  }

  if (error && !dashboard) {
    return (
      <ErrorState
        message="Error al cargar el dashboard"
        onRetry={refetch}
      />
    );
  }

  if (isEmpty) {
    return <EmptyDashboard />;
  }

  if (!dashboard) return null;

  return (
    <ScrollView className="bg-gray-50">
      {/* Header */}
      <View className="bg-white p-4 pb-6">
        <Text className="text-2xl font-bold">
          👤 Hola, {dashboard.user.name}!
        </Text>
        <Text className="text-gray-600">Tu progreso esta semana</Text>

        <View className="mt-4">
          <PeriodSelector value={period} onChange={setPeriod} />
        </View>
      </View>

      {/* Summary Cards */}
      <View className="p-4">
        <SummaryCards summary={dashboard.summary} />
      </View>

      {/* Activity Chart */}
      <View className="bg-white mx-4 p-4 rounded-lg mb-4">
        <Text className="font-semibold mb-4">📈 Actividad Semanal</Text>
        <ActivityChart activity={dashboard.activity} />
      </View>

      {/* Materials in Progress */}
      <View className="bg-white mx-4 p-4 rounded-lg mb-4">
        <Text className="font-semibold mb-4">📚 Materiales en Progreso</Text>
        <MaterialsInProgress materials={dashboard.materials_in_progress} />
      </View>

      {/* Recent Attempts */}
      <View className="bg-white mx-4 p-4 rounded-lg mb-4">
        <Text className="font-semibold mb-4">📝 Evaluaciones Recientes</Text>
        <RecentAttempts attempts={dashboard.recent_attempts} />
      </View>

      {/* Achievements */}
      {dashboard.achievements.length > 0 && (
        <View className="bg-white mx-4 p-4 rounded-lg mb-4">
          <Text className="font-semibold mb-4">🎯 Logros</Text>
          <View className="flex-row flex-wrap gap-2">
            {dashboard.achievements.map(achievement => (
              <View key={achievement.id} className="bg-yellow-50 px-3 py-2 rounded-full">
                <Text>{achievement.icon} {achievement.name}</Text>
              </View>
            ))}
          </View>
        </View>
      )}
    </ScrollView>
  );
}
```

### Componente de Tarjetas de Resumen

```typescript
// components/SummaryCards.tsx
interface SummaryCardsProps {
  summary: DashboardResponse['summary'];
}

export function SummaryCards({ summary }: SummaryCardsProps) {
  const cards = [
    {
      value: summary.materials_completed,
      label: 'Materiales',
      sublabel: 'Completados',
      icon: '📚',
      color: 'blue'
    },
    {
      value: summary.quizzes_passed,
      label: 'Quizzes',
      sublabel: 'Aprobados',
      icon: '✅',
      color: 'green'
    },
    {
      value: `${summary.average_score.toFixed(0)}%`,
      label: 'Promedio',
      sublabel: 'Calificación',
      icon: '📊',
      color: summary.average_score >= 70 ? 'green' : 'orange'
    }
  ];

  return (
    <View className="flex-row gap-4">
      {cards.map((card, i) => (
        <View
          key={i}
          className={`flex-1 bg-${card.color}-50 p-4 rounded-lg items-center`}
        >
          <Text className="text-3xl">{card.icon}</Text>
          <Text className="text-2xl font-bold mt-2">{card.value}</Text>
          <Text className="text-sm text-gray-600">{card.label}</Text>
          <Text className="text-xs text-gray-500">{card.sublabel}</Text>
        </View>
      ))}
    </View>
  );
}
```

## Notas de Implementación

### Backend Requerido

1. **Endpoint `/v1/progress/dashboard`:**
   - Requiere implementación en API Mobile
   - Debe agregar datos de múltiples tablas
   - Considerar cache server-side

2. **Queries Optimizadas:**
   ```sql
   -- Materiales completados esta semana
   SELECT COUNT(*) FROM material_progress
   WHERE user_id = $1
     AND progress_percentage = 100
     AND updated_at >= $2;

   -- Promedio de calificaciones
   SELECT AVG(score) FROM assessment_attempts
   WHERE user_id = $1
     AND completed_at >= $2;
   ```

### Performance

1. **Cache de 2 minutos** para dashboard
2. **Lazy loading** de secciones secundarias
3. **Optimistic updates** al completar acciones

### Accesibilidad

```typescript
<View
  accessible={true}
  accessibilityRole="summary"
  accessibilityLabel={`Has completado ${summary.materials_completed} materiales y aprobado ${summary.quizzes_passed} quizzes con un promedio de ${summary.average_score} por ciento`}
>
  <SummaryCards summary={summary} />
</View>
```

### Testing

```typescript
describe('Dashboard', () => {
  it('should display summary correctly', async () => {
    const mockDashboard = createMockDashboard();
    mockApi.get.mockResolvedValue({ data: mockDashboard });

    const { getByText } = render(<Dashboard />);

    await waitFor(() => {
      expect(getByText('12')).toBeTruthy(); // materials_completed
      expect(getByText('85%')).toBeTruthy(); // average_score
    });
  });

  it('should show empty state for new user', async () => {
    const mockDashboard = createMockDashboard({
      summary: { materials_started: 0, ...emptyStats }
    });
    mockApi.get.mockResolvedValue({ data: mockDashboard });

    const { getByText } = render(<Dashboard />);

    await waitFor(() => {
      expect(getByText('¡Bienvenido a EduGo!')).toBeTruthy();
    });
  });
});
```

---

**Última actualización:** 2025-12-24
**Versión:** 1.0
**Estado:** ⚠️ Requiere endpoint backend
