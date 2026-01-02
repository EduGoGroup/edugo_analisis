# RFC-033: Historial de Intentos por Material

## Metadata
- **ID:** RFC-033
- **Proceso:** Evaluaciones
- **Subproceso:** Historial de Intentos del Usuario
- **Prioridad:** Media
- **Dependencias:** RFC-031 (Enviar Intento), RFC-032 (Ver Resultados)
- **Estado API:** ✅ Listo

## Descripción

Permite a los estudiantes visualizar el historial completo de sus intentos de evaluación, con paginación, filtros y acceso directo a resultados detallados.

## Flujo de Usuario (UX)

### Pantalla: Mi Historial de Evaluaciones

```
┌─────────────────────────────────────────┐
│  Mi Historial de Evaluaciones            │
│                                          │
│  Filtros: [Todos ▼] [Ordenar: Reciente ▼]│
│                                          │
│  ┌────────────────────────────────┐     │
│  │ Matemáticas Grado 5             │     │
│  │ 24 Dic 2024, 10:30 AM          │     │
│  │                                 │     │
│  │ ✅ 85/100 - Aprobado            │     │
│  │ Tiempo: 12:34  Intento 2/3     │     │
│  │                                 │     │
│  │ [Ver Detalles]                  │     │
│  └────────────────────────────────┘     │
│                                          │
│  ┌────────────────────────────────┐     │
│  │ Álgebra Básica                  │     │
│  │ 20 Dic 2024, 3:15 PM           │     │
│  │                                 │     │
│  │ ❌ 65/100 - Reprobado           │     │
│  │ Tiempo: 18:20  Intento 1/3     │     │
│  │                                 │     │
│  │ [Ver Detalles] [Reintentar]    │     │
│  └────────────────────────────────┘     │
│                                          │
│  Mostrando 2 de 15 intentos              │
│  [← Anterior]  [Siguiente →]             │
└─────────────────────────────────────────┘
```

### Filtros Disponibles

- **Por Estado:** Todos, Aprobado, Reprobado
- **Por Fecha:** Última semana, Último mes, Todo
- **Ordenar:** Más reciente, Más antiguo, Mejor score, Peor score

## Flujo de Datos (Técnico)

### Endpoints Involucrados

```
GET /v1/users/me/attempts?limit=10&offset=0&material_id=xxx
```

### Request/Response (TypeScript)

**Query Parameters:**
```typescript
interface AttemptHistoryQuery {
  limit?: number;          // 1-100, default 10
  offset?: number;         // default 0
  material_id?: string;    // Filtrar por material específico
  passed?: boolean;        // Filtrar por aprobado/reprobado
  order_by?: 'completed_at' | 'score'; // default: completed_at
  order_dir?: 'asc' | 'desc'; // default: desc
}
```

**Response (200 OK):**
```typescript
interface AttemptHistoryResponse {
  attempts: AttemptSummaryDTO[];
  total_count: number;
  limit: number;
  offset: number;
  has_more: boolean;
}

interface AttemptSummaryDTO {
  attempt_id: string;
  material_id: string;
  material_title: string;
  score: number;
  max_score: number;
  passed: boolean;
  time_spent_seconds: number;
  completed_at: string;
  attempt_number: number; // Intento N de X
  max_attempts: number;
}
```

## Código de Ejemplo (Mobile - TypeScript)

### Service: HistoryService

```typescript
// services/HistoryService.ts

export class HistoryService {
  private api: AxiosInstance;

  constructor(baseURL: string) {
    this.api = axios.create({ baseURL, timeout: 10000 });
  }

  async getAttemptHistory(
    query: AttemptHistoryQuery = {}
  ): Promise<AttemptHistoryResponse> {

    const params = new URLSearchParams({
      limit: String(query.limit || 10),
      offset: String(query.offset || 0),
      order_dir: query.order_dir || 'desc'
    });

    if (query.material_id) params.append('material_id', query.material_id);
    if (query.passed !== undefined) params.append('passed', String(query.passed));
    if (query.order_by) params.append('order_by', query.order_by);

    const response = await this.api.get<AttemptHistoryResponse>(
      `/v1/users/me/attempts?${params.toString()}`
    );

    return response.data;
  }
}
```

### Hook: useAttemptHistory

```typescript
// hooks/useAttemptHistory.ts

export function useAttemptHistory(options: AttemptHistoryQuery = {}) {
  const [data, setData] = useState<AttemptHistoryResponse | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  const service = useMemo(
    () => new HistoryService(process.env.REACT_APP_API_MOBILE_URL!),
    []
  );

  const fetchHistory = useCallback(async () => {
    setLoading(true);
    setError(null);

    try {
      const result = await service.getAttemptHistory(options);
      setData(result);
    } catch (err: any) {
      setError(err);
    } finally {
      setLoading(false);
    }
  }, [service, options]);

  useEffect(() => {
    fetchHistory();
  }, [fetchHistory]);

  return {
    attempts: data?.attempts || [],
    totalCount: data?.total_count || 0,
    hasMore: data?.has_more || false,
    loading,
    error,
    refetch: fetchHistory
  };
}
```

### Componente: AttemptHistoryList

```typescript
// components/AttemptHistoryList.tsx

export const AttemptHistoryList: React.FC = () => {
  const [page, setPage] = useState(0);
  const [filters, setFilters] = useState<AttemptHistoryQuery>({});

  const itemsPerPage = 10;
  const offset = page * itemsPerPage;

  const { attempts, totalCount, hasMore, loading, error } = useAttemptHistory({
    ...filters,
    limit: itemsPerPage,
    offset
  });

  if (loading && page === 0) return <HistorySkeleton />;
  if (error) return <ErrorMessage error={error} />;

  return (
    <div className="attempt-history">
      <h2>Mi Historial de Evaluaciones</h2>

      <Filters filters={filters} onChange={setFilters} />

      <div className="attempts-list">
        {attempts.map(attempt => (
          <AttemptCard key={attempt.attempt_id} attempt={attempt} />
        ))}
      </div>

      {attempts.length === 0 && !loading && (
        <EmptyState message="No tienes intentos de evaluación aún" />
      )}

      {totalCount > 0 && (
        <Pagination
          currentPage={page}
          totalItems={totalCount}
          itemsPerPage={itemsPerPage}
          hasMore={hasMore}
          onPageChange={setPage}
        />
      )}
    </div>
  );
};
```

### Componente: AttemptCard

```typescript
// components/AttemptCard.tsx

export const AttemptCard: React.FC<{ attempt: AttemptSummaryDTO }> = ({ attempt }) => {
  const navigate = useNavigate();

  const percentage = (attempt.score / attempt.max_score) * 100;
  const formattedDate = formatDistanceToNow(new Date(attempt.completed_at), {
    addSuffix: true,
    locale: es
  });

  return (
    <div className={`attempt-card ${attempt.passed ? 'passed' : 'failed'}`}>
      <div className="card-header">
        <h3>{attempt.material_title}</h3>
        <span className="date">{formattedDate}</span>
      </div>

      <div className="card-body">
        <div className="score-section">
          <span className={`status-badge ${attempt.passed ? 'success' : 'danger'}`}>
            {attempt.passed ? '✅ Aprobado' : '❌ Reprobado'}
          </span>
          <span className="score">
            {attempt.score}/{attempt.max_score} ({Math.round(percentage)}%)
          </span>
        </div>

        <div className="metadata">
          <span>⏱️ {formatDuration(attempt.time_spent_seconds)}</span>
          <span>📊 Intento {attempt.attempt_number}/{attempt.max_attempts}</span>
        </div>
      </div>

      <div className="card-actions">
        <button onClick={() => navigate(`/attempts/${attempt.attempt_id}`)}>
          Ver Detalles
        </button>

        {!attempt.passed && attempt.attempt_number < attempt.max_attempts && (
          <button onClick={() => navigate(`/materials/${attempt.material_id}/assessment`)}>
            Reintentar
          </button>
        )}
      </div>
    </div>
  );
};
```

## Consideraciones de UX

### 1. Paginación Eficiente

```typescript
// Infinite scroll alternativo
export function useInfiniteAttemptHistory() {
  const [allAttempts, setAllAttempts] = useState<AttemptSummaryDTO[]>([]);
  const [page, setPage] = useState(0);
  const [hasMore, setHasMore] = useState(true);

  const loadMore = useCallback(async () => {
    const service = new HistoryService(process.env.REACT_APP_API_MOBILE_URL!);
    const result = await service.getAttemptHistory({
      limit: 10,
      offset: page * 10
    });

    setAllAttempts(prev => [...prev, ...result.attempts]);
    setHasMore(result.has_more);
    setPage(prev => prev + 1);
  }, [page]);

  useEffect(() => {
    if (page === 0) loadMore();
  }, []);

  return { attempts: allAttempts, hasMore, loadMore };
}
```

### 2. Filtros Interactivos

```typescript
export const Filters: React.FC<FiltersProps> = ({ filters, onChange }) => {
  return (
    <div className="filters">
      <select
        value={filters.passed?.toString() || 'all'}
        onChange={e => {
          const value = e.target.value;
          onChange({
            ...filters,
            passed: value === 'all' ? undefined : value === 'true'
          });
        }}
      >
        <option value="all">Todos</option>
        <option value="true">Aprobados</option>
        <option value="false">Reprobados</option>
      </select>

      <select
        value={filters.order_by || 'completed_at'}
        onChange={e => onChange({ ...filters, order_by: e.target.value as any })}
      >
        <option value="completed_at">Más reciente</option>
        <option value="score">Mejor calificación</option>
      </select>
    </div>
  );
};
```

### 3. Estadísticas Generales

```typescript
export const HistoryStats: React.FC<{ attempts: AttemptSummaryDTO[] }> = ({ attempts }) => {
  const totalAttempts = attempts.length;
  const passed = attempts.filter(a => a.passed).length;
  const passRate = totalAttempts > 0 ? (passed / totalAttempts) * 100 : 0;
  const avgScore = attempts.reduce((sum, a) => sum + a.score, 0) / totalAttempts;

  return (
    <div className="history-stats">
      <div className="stat-card">
        <span className="stat-value">{totalAttempts}</span>
        <span className="stat-label">Intentos</span>
      </div>

      <div className="stat-card">
        <span className="stat-value">{passed}</span>
        <span className="stat-label">Aprobados</span>
      </div>

      <div className="stat-card">
        <span className="stat-value">{Math.round(passRate)}%</span>
        <span className="stat-label">Tasa de Aprobación</span>
      </div>

      <div className="stat-card">
        <span className="stat-value">{Math.round(avgScore)}</span>
        <span className="stat-label">Promedio</span>
      </div>
    </div>
  );
};
```

## Almacenamiento Local

### Cache de Lista

```typescript
// Cache temporal (5 minutos)
function cacheAttemptHistory(query: string, data: AttemptHistoryResponse) {
  const cache = {
    query,
    data,
    cachedAt: Date.now(),
    expiresAt: Date.now() + 300000 // 5 min
  };

  sessionStorage.setItem(`attempt_history_${query}`, JSON.stringify(cache));
}

function getCachedAttemptHistory(query: string): AttemptHistoryResponse | null {
  const cached = sessionStorage.getItem(`attempt_history_${query}`);
  if (!cached) return null;

  const cache = JSON.parse(cached);

  if (Date.now() > cache.expiresAt) {
    sessionStorage.removeItem(`attempt_history_${query}`);
    return null;
  }

  return cache.data;
}
```

## Notas de Implementación

### 1. Performance

**Backend debe:**
- Usar índices en `student_id`, `completed_at`, `score`
- Limitar máximo de resultados por página (100)
- Optimizar queries con proyecciones

### 2. SEO (si aplica)

```typescript
// Meta tags para página de historial
<Helmet>
  <title>Mi Historial de Evaluaciones | EduGo</title>
  <meta name="robots" content="noindex" /> {/* Privado */}
</Helmet>
```

### 3. Exportación de Datos

```typescript
// Opcional: Exportar historial completo a CSV
async function exportHistory() {
  const service = new HistoryService(process.env.REACT_APP_API_MOBILE_URL!);

  const result = await service.getAttemptHistory({
    limit: 1000 // Obtener todos
  });

  const csv = convertToCSV(result.attempts);
  downloadFile(csv, 'mi-historial-evaluaciones.csv');
}
```
