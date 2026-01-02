# RFC-032: Ver Resultados de Intento

## Metadata
- **ID:** RFC-032
- **Proceso:** Evaluaciones
- **Subproceso:** Ver Resultados de Intento
- **Prioridad:** Alta
- **Dependencias:** RFC-031 (Enviar Intento)
- **Estado API:** ✅ Listo

## Descripción

Permite a los estudiantes consultar los resultados detallados de un intento de evaluación previo, incluyendo calificación, feedback por pregunta y opciones de reintento.

## Flujo de Usuario (UX)

### Pantalla: Resultados Detallados

```
┌─────────────────────────────────────────┐
│  ← Resultado de Evaluación               │
│                                          │
│  Matemáticas Grado 5                     │
│  Completado: 24 Dic 2024, 10:30 AM      │
│                                          │
│  ┌────────────────────────────────┐     │
│  │      Tu Calificación            │     │
│  │                                 │     │
│  │         85/100                  │     │
│  │       ★★★★☆                     │     │
│  │                                 │     │
│  │      ✅ Aprobado                │     │
│  │  (Nota mínima: 70%)             │     │
│  └────────────────────────────────┘     │
│                                          │
│  📊 Estadísticas                         │
│  ├─ Respuestas correctas: 8/10           │
│  ├─ Tiempo empleado: 12:34               │
│  ├─ Intentos usados: 2/3                 │
│  └─ Puedes reintentar: ✅ Sí             │
│                                          │
│  📝 Desglose por Pregunta                │
│                                          │
│  Pregunta 1  ✅                          │
│  ¿Cuál es el resultado de 5 × 8?        │
│  Tu respuesta: 40 ✓                      │
│  Puntos: 10/10                           │
│                                          │
│  Pregunta 2  ❌                          │
│  ¿Qué es una función?                    │
│  Tu respuesta: Un tipo de variable       │
│  Correcta: Una relación matemática       │
│  💡 Una función es una relación...       │
│  Puntos: 0/10                            │
│                                          │
│  [Reintentar]  [Ver Material]            │
└─────────────────────────────────────────┘
```

## Flujo de Datos (Técnico)

### Endpoints Involucrados

```
GET /v1/attempts/:id/results
```

### Request/Response (TypeScript)

**Request:**
```typescript
// Sin body, attemptId en URL params
```

**Response (200 OK):**
```typescript
interface AttemptResultResponse {
  attempt_id: string;
  student_id: string;
  material_id: string;
  material_title: string;
  score: number;
  max_score: number;
  passed: boolean;
  feedback: AnswerFeedbackDTO[];
  can_retake: boolean;
  attempts_used: number;
  max_attempts: number;
  time_spent_seconds: number;
  started_at: string;
  completed_at: string;
}

interface AnswerFeedbackDTO {
  question_id: string;
  question_text: string;
  is_correct: boolean;
  selected_answer: string;
  selected_answer_text: string;
  correct_answer: string;
  correct_answer_text: string;
  explanation: string;
  points_earned: number;
  max_points: number;
}
```

## Código de Ejemplo (Mobile - TypeScript)

### Service: ResultsService

```typescript
// services/ResultsService.ts

export class ResultsService {
  private api: AxiosInstance;

  constructor(baseURL: string) {
    this.api = axios.create({ baseURL, timeout: 10000 });
  }

  async getAttemptResults(attemptId: string): Promise<AttemptResultResponse> {
    // Verificar cache primero
    const cached = getCachedAttemptResult(attemptId);
    if (cached) return cached;

    // Obtener de API
    const response = await this.api.get<AttemptResultResponse>(
      `/v1/attempts/${attemptId}/results`
    );

    // Guardar en cache (los resultados son inmutables)
    cacheAttemptResult(attemptId, response.data);

    return response.data;
  }
}
```

### Hook: useAttemptResults

```typescript
// hooks/useAttemptResults.ts

export function useAttemptResults(attemptId: string) {
  const [results, setResults] = useState<AttemptResultResponse | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    const service = new ResultsService(process.env.REACT_APP_API_MOBILE_URL!);

    service.getAttemptResults(attemptId)
      .then(setResults)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [attemptId]);

  return { results, loading, error };
}
```

### Componente: ResultsView

```typescript
// components/ResultsView.tsx

export const ResultsView: React.FC<{ attemptId: string }> = ({ attemptId }) => {
  const { results, loading, error } = useAttemptResults(attemptId);

  if (loading) return <ResultsSkeleton />;
  if (error) return <ErrorMessage error={error} />;
  if (!results) return null;

  return (
    <div className="results-view">
      <ScoreCard
        score={results.score}
        maxScore={results.max_score}
        passed={results.passed}
      />

      <Statistics
        correctAnswers={results.feedback.filter(f => f.is_correct).length}
        totalQuestions={results.feedback.length}
        timeSpent={results.time_spent_seconds}
        attemptsUsed={results.attempts_used}
        maxAttempts={results.max_attempts}
        canRetake={results.can_retake}
      />

      <FeedbackList feedback={results.feedback} />

      <Actions
        materialId={results.material_id}
        canRetake={results.can_retake}
      />
    </div>
  );
};
```

## Consideraciones de UX

### 1. Visualización de Score

```typescript
function ScoreCard({ score, maxScore, passed }: ScoreCardProps) {
  const percentage = (score / maxScore) * 100;
  const stars = Math.round(percentage / 20); // 0-5 estrellas

  return (
    <div className={`score-card ${passed ? 'passed' : 'failed'}`}>
      <div className="score-value">
        <span className="score">{score}</span>
        <span className="max-score">/{maxScore}</span>
      </div>

      <div className="stars">
        {Array.from({ length: 5 }).map((_, i) => (
          <Star key={i} filled={i < stars} />
        ))}
      </div>

      <div className="status">
        {passed ? '✅ Aprobado' : '❌ Reprobado'}
      </div>
    </div>
  );
}
```

### 2. Feedback por Pregunta

```typescript
function FeedbackItem({ feedback }: { feedback: AnswerFeedbackDTO }) {
  const [expanded, setExpanded] = useState(!feedback.is_correct);

  return (
    <div className={`feedback-item ${feedback.is_correct ? 'correct' : 'incorrect'}`}>
      <div className="feedback-header" onClick={() => setExpanded(!expanded)}>
        <span className="icon">{feedback.is_correct ? '✅' : '❌'}</span>
        <span className="question">{feedback.question_text}</span>
        <span className="points">{feedback.points_earned}/{feedback.max_points}</span>
        <button className="expand-btn">{expanded ? '▼' : '▶'}</button>
      </div>

      {expanded && (
        <div className="feedback-details">
          {!feedback.is_correct && (
            <>
              <div className="your-answer">
                <strong>Tu respuesta:</strong> {feedback.selected_answer_text}
              </div>
              <div className="correct-answer">
                <strong>Correcta:</strong> {feedback.correct_answer_text}
              </div>
            </>
          )}

          {feedback.explanation && (
            <div className="explanation">
              <strong>💡 Explicación:</strong>
              <p>{feedback.explanation}</p>
            </div>
          )}
        </div>
      )}
    </div>
  );
}
```

### 3. Indicadores de Reintento

```typescript
function RetakeIndicator({ attemptsUsed, maxAttempts, canRetake }: RetakeProps) {
  const remaining = maxAttempts - attemptsUsed;

  return (
    <div className="retake-indicator">
      <div className="attempts-bar">
        {Array.from({ length: maxAttempts }).map((_, i) => (
          <div
            key={i}
            className={`attempt-dot ${i < attemptsUsed ? 'used' : 'available'}`}
          />
        ))}
      </div>

      <p>
        Intentos usados: {attemptsUsed}/{maxAttempts}
        {canRetake && ` (${remaining} disponible${remaining !== 1 ? 's' : ''})`}
      </p>

      {canRetake ? (
        <button className="retake-btn">Reintentar Evaluación</button>
      ) : (
        <p className="no-retake">No quedan intentos disponibles</p>
      )}
    </div>
  );
}
```

## Almacenamiento Local

### Caché Permanente de Resultados

```typescript
// Los resultados son inmutables, cache sin expiración
function cacheAttemptResult(attemptId: string, result: AttemptResultResponse) {
  const cache = {
    attemptId,
    result,
    cachedAt: Date.now()
  };

  localStorage.setItem(`attempt_result_${attemptId}`, JSON.stringify(cache));
}

function getCachedAttemptResult(attemptId: string): AttemptResultResponse | null {
  const cached = localStorage.getItem(`attempt_result_${attemptId}`);
  if (!cached) return null;

  const { result } = JSON.parse(cached);
  return result;
}
```

## Notas de Implementación

### 1. Autorización

**Backend debe validar:**
- Solo el estudiante dueño del intento puede ver resultados
- O un admin/teacher puede ver resultados de sus estudiantes

```typescript
// BACKEND
if (attempt.student_id !== currentUser.id && !currentUser.isAdmin) {
  throw new ForbiddenError('Cannot view other user attempts');
}
```

### 2. Performance

**Optimizaciones:**
- Cache permanente en cliente (resultados inmutables)
- Lazy render de feedback (solo expandir si necesario)
- Paginación de feedback si >50 preguntas

### 3. Accesibilidad

```typescript
<div role="region" aria-label="Resultados de evaluación">
  <h2 id="score-heading">Tu Calificación</h2>
  <div aria-labelledby="score-heading">
    <span aria-label={`Calificación: ${score} de ${maxScore} puntos`}>
      {score}/{maxScore}
    </span>
  </div>
</div>
```
