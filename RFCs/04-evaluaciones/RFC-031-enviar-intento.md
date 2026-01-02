# RFC-031: Enviar Respuestas de Quiz

## Metadata
- **ID:** RFC-031
- **Proceso:** Evaluaciones
- **Subproceso:** Crear Intento de Evaluación
- **Prioridad:** Alta
- **Dependencias:** RFC-030 (Obtener Quiz), Worker (crítico)
- **Estado API:** ❌ Bloqueado (depende Worker para quiz existente)

## Descripción

Permite a los estudiantes enviar sus respuestas a un quiz y recibir calificación automática con feedback detallado. El scoring se realiza en el servidor validando contra las respuestas correctas almacenadas en MongoDB.

## Flujo de Usuario (UX)

### Pantalla: Realizando Quiz

```
┌─────────────────────────────────────────┐
│  ← Evaluación: Matemáticas Grado 5      │
│                                          │
│  Pregunta 3 de 10      Tiempo: 08:45    │
│  ━━━━━━━━━━░░░░░░░░░░ 30%              │
│                                          │
│  ¿Cuál es el resultado de 5 × 8?        │
│                                          │
│  ○ 35                                    │
│  ● 40  ✓                                │
│  ○ 45                                    │
│  ○ 50                                    │
│                                          │
│  [← Anterior]  [Siguiente →]             │
│                                          │
│  ┌────────────────────────────────┐     │
│  │ Preguntas:                      │     │
│  │ ● ● ● ○ ○ ○ ○ ○ ○ ○            │     │
│  │ ✓ ✓ ✓                          │     │
│  └────────────────────────────────┘     │
│                                          │
│           [Finalizar Quiz]               │
└─────────────────────────────────────────┘
```

### Confirmación de Envío

```
┌─────────────────────────────────────────┐
│  ¿Finalizar Evaluación?                  │
│                                          │
│  Has respondido 8 de 10 preguntas       │
│  Tiempo restante: 05:23                  │
│                                          │
│  ⚠️ No podrás cambiar tus respuestas     │
│     después de enviar.                   │
│                                          │
│  [Revisar]  [Enviar Evaluación]          │
└─────────────────────────────────────────┘
```

### Pantalla de Resultados

```
┌─────────────────────────────────────────┐
│  Resultados de Evaluación                │
│                                          │
│  ┌────────────────────────────────┐     │
│  │  Tu Calificación                │     │
│  │                                 │     │
│  │       85/100                    │     │
│  │     ★★★★☆                       │     │
│  │                                 │     │
│  │  ✅ Aprobado                    │     │
│  └────────────────────────────────┘     │
│                                          │
│  Respuestas correctas: 8/10              │
│  Tiempo empleado: 12:34                  │
│  Intentos usados: 2/3                    │
│                                          │
│  📊 Desglose por Pregunta:               │
│                                          │
│  1. ✅ Correcta - Definición de...       │
│  2. ✅ Correcta - Teorema de...          │
│  3. ❌ Incorrecta                        │
│     Tu respuesta: B                      │
│     Correcta: C                          │
│     💡 La respuesta correcta es C        │
│        porque...                         │
│  ...                                     │
│                                          │
│  [Ver Material]  [Reintentar]            │
└─────────────────────────────────────────┘
```

### Estados de Intento

**Durante el Quiz:**
- Mostrar cronómetro en tiempo real
- Guardar progreso cada 30s (opcional)
- Validar que todas las preguntas tengan respuesta antes de enviar

**Enviando:**
```
┌──────────────────────────────────────┐
│  📤 Enviando respuestas...            │
│  ━━━━━━━━━━━━━━━━━━━━━━            │
│  Calculando calificación...          │
└──────────────────────────────────────┘
```

**Calificado:**
- Mostrar resultado con animación
- Feedback inmediato
- Opciones: ver detalle, reintentar, volver

## Flujo de Datos (Técnico)

### Diagrama de Secuencia

```
Usuario → Frontend → API Mobile → PostgreSQL + MongoDB
                           ↓
                    1. Validar assessment existe
                    2. Validar intentos disponibles
                    3. Obtener respuestas correctas (MongoDB)
                    4. Calcular score
                    5. Guardar intento (PostgreSQL)
                    6. Guardar respuestas (PostgreSQL)
                    7. Retornar resultado con feedback
```

### Flujo Detallado

1. **Usuario finaliza quiz**
   - Valida que todas las preguntas estén respondidas
   - Calcula tiempo total empleado
   - Muestra confirmación

2. **Frontend envía respuestas:**
   ```typescript
   const attempt = {
     answers: [
       { question_id: 'q1', selected_answer_id: 'opt_a', time_spent_seconds: 45 },
       { question_id: 'q2', selected_answer_id: 'opt_c', time_spent_seconds: 60 }
     ],
     time_spent_seconds: 754 // Total: 12min 34s
   };
   ```

3. **API Mobile procesa:**
   - Valida que el assessment exista
   - Verifica intentos disponibles (max_attempts)
   - Obtiene respuestas correctas de MongoDB
   - Calcula score: `(correctas / total) * 100`
   - Genera feedback por pregunta
   - Guarda intento en PostgreSQL
   - Guarda respuestas individuales en PostgreSQL

4. **API Mobile retorna resultado:**
   - Score calculado
   - Aprobado/Reprobado (según pass_threshold)
   - Feedback por pregunta
   - Información de reintentos

### Endpoints Involucrados

**Principal:**
```
POST /v1/materials/:id/assessment/attempts
```

**Consulta previa:**
```
GET /v1/materials/:id/assessment
```

### Request/Response (TypeScript)

**Request:**
```typescript
interface CreateAttemptRequest {
  answers: UserAnswerDTO[];
  time_spent_seconds: number; // min=1, max=7200 (2 horas)
}

interface UserAnswerDTO {
  question_id: string;        // ID de la pregunta
  selected_answer_id: string; // ID de la opción seleccionada
  time_spent_seconds: number; // Tiempo en esta pregunta
}
```

**Ejemplo Request:**
```json
{
  "answers": [
    {
      "question_id": "q_abc123",
      "selected_answer_id": "opt_b",
      "time_spent_seconds": 45
    },
    {
      "question_id": "q_def456",
      "selected_answer_id": "opt_c",
      "time_spent_seconds": 60
    },
    {
      "question_id": "q_ghi789",
      "selected_answer_id": "opt_a",
      "time_spent_seconds": 30
    }
  ],
  "time_spent_seconds": 754
}
```

**Response Exitoso (201 Created):**
```typescript
interface AttemptResultResponse {
  attempt_id: string;               // UUID del intento
  score: number;                    // Puntaje obtenido (0-100)
  max_score: number;                // Puntaje máximo (100)
  passed: boolean;                  // Aprobó según threshold
  feedback: AnswerFeedbackDTO[];    // Feedback por pregunta
  can_retake: boolean;              // Puede reintentar
  attempts_used: number;            // Intentos usados
  max_attempts: number;             // Intentos máximos
  time_spent_seconds: number;       // Tiempo total empleado
  completed_at: string;             // ISO8601
}

interface AnswerFeedbackDTO {
  question_id: string;
  question_text: string;            // Texto de la pregunta
  is_correct: boolean;              // ¿Respuesta correcta?
  selected_answer: string;          // ID de opción seleccionada
  selected_answer_text: string;     // Texto de opción seleccionada
  correct_answer: string;           // ID de opción correcta
  correct_answer_text: string;      // Texto de opción correcta
  explanation: string;              // Explicación de la respuesta
  points_earned: number;            // Puntos obtenidos
  max_points: number;               // Puntos máximos
}
```

**Ejemplo Response:**
```json
{
  "attempt_id": "attempt-uuid-123",
  "score": 85,
  "max_score": 100,
  "passed": true,
  "feedback": [
    {
      "question_id": "q_abc123",
      "question_text": "¿Cuál es el resultado de 5 × 8?",
      "is_correct": true,
      "selected_answer": "opt_b",
      "selected_answer_text": "40",
      "correct_answer": "opt_b",
      "correct_answer_text": "40",
      "explanation": "5 multiplicado por 8 es igual a 40",
      "points_earned": 10,
      "max_points": 10
    },
    {
      "question_id": "q_def456",
      "question_text": "¿Qué es una función?",
      "is_correct": false,
      "selected_answer": "opt_a",
      "selected_answer_text": "Un tipo de variable",
      "correct_answer": "opt_c",
      "correct_answer_text": "Una relación matemática",
      "explanation": "Una función es una relación matemática que asigna a cada elemento de un conjunto exactamente un elemento de otro conjunto",
      "points_earned": 0,
      "max_points": 10
    }
  ],
  "can_retake": true,
  "attempts_used": 2,
  "max_attempts": 3,
  "time_spent_seconds": 754,
  "completed_at": "2024-12-24T10:30:00Z"
}
```

**Response Error (400 - Validación):**
```json
{
  "error": "validation_error",
  "message": "Invalid request data",
  "code": "VALIDATION_ERROR",
  "details": {
    "answers": "must not be empty",
    "time_spent_seconds": "must be between 1 and 7200"
  }
}
```

**Response Error (403 - Sin intentos):**
```json
{
  "error": "forbidden",
  "message": "No attempts remaining",
  "code": "MAX_ATTEMPTS_REACHED",
  "attempts_used": 3,
  "max_attempts": 3
}
```

**Response Error (404 - Assessment no existe):**
```json
{
  "error": "not_found",
  "message": "Assessment not found",
  "code": "ASSESSMENT_NOT_FOUND"
}
```

**Response Error (409 - Intento en progreso):**
```json
{
  "error": "conflict",
  "message": "There is an ongoing attempt",
  "code": "ATTEMPT_IN_PROGRESS",
  "attempt_id": "attempt-uuid-123"
}
```

## Estados y Transiciones

### Estado del Intento

```
Attempt States:

┌──────────┐
│ NOT_     │
│ STARTED  │ → Usuario viendo quiz, no ha enviado
└──────────┘
     ↓ (enviar respuestas)
┌──────────┐
│ SUBMIT-  │
│ TING     │ → POST a /attempts en progreso
└──────────┘
     ↓
┌──────────┐
│ GRADING  │ → Servidor calculando score
└──────────┘
     ↓
┌──────────┐
│ COMPLE-  │
│ TED      │ → Resultado disponible ✅
└──────────┘
```

### Validación de Intentos

```
Attempt Validation:

┌─────────────────────┐
│ Check attempts_used │ → attempts_used < max_attempts?
└─────────────────────┘
          ↓ SÍ                    ↓ NO
┌─────────────────┐     ┌──────────────────┐
│ Allow attempt   │     │ Return 403       │
│ Process answers │     │ MAX_ATTEMPTS_    │
│ Return result   │     │ REACHED          │
└─────────────────┘     └──────────────────┘
```

## Manejo de Errores

### Errores de Validación

**Respuestas vacías:**
```typescript
if (answers.length === 0) {
  throw new ValidationError('Debes responder al menos una pregunta');
}
```

**Tiempo inválido:**
```typescript
if (time_spent_seconds < 1 || time_spent_seconds > 7200) {
  throw new ValidationError('Tiempo inválido (1s - 2h)');
}
```

**Preguntas faltantes:**
```typescript
const allQuestions = assessment.questions.map(q => q.id);
const answeredQuestions = answers.map(a => a.question_id);
const missing = allQuestions.filter(q => !answeredQuestions.includes(q));

if (missing.length > 0) {
  showWarning(`Faltan ${missing.length} preguntas por responder`);
  // Permitir enviar parcialmente o requerir completitud
}
```

### Errores de Negocio

**Sin intentos disponibles:**
```typescript
try {
  await createAttempt(materialId, answers);
} catch (error) {
  if (error.code === 'MAX_ATTEMPTS_REACHED') {
    showModal({
      title: 'Sin intentos disponibles',
      message: `Has usado los ${error.max_attempts} intentos permitidos`,
      actions: [
        { label: 'Ver Resultados', onClick: () => redirectTo(`/attempts/${lastAttemptId}`) },
        { label: 'Ver Material', onClick: () => redirectTo(`/materials/${materialId}`) }
      ]
    });
  }
}
```

**Assessment no encontrado:**
```typescript
if (error.code === 'ASSESSMENT_NOT_FOUND') {
  showError('La evaluación no está disponible');
  redirectTo(`/materials/${materialId}`);
}
```

### Errores de Red

**Timeout durante envío:**
```typescript
try {
  const result = await createAttempt(materialId, answers, {
    timeout: 60000 // 60 segundos
  });
} catch (error) {
  if (error.code === 'TIMEOUT') {
    // Guardar respuestas localmente
    saveAttemptToLocalStorage(materialId, answers);

    showError('La conexión se perdió. Tus respuestas se guardaron localmente.');
    offerRetry();
  }
}
```

**Pérdida de conexión:**
```typescript
window.addEventListener('offline', () => {
  showWarning('Sin conexión. Tus respuestas se guardarán localmente.');
  enableOfflineMode();
});

window.addEventListener('online', () => {
  showSuccess('Conexión restaurada. ¿Enviar respuestas guardadas?');
  offerSync();
});
```

## Consideraciones de UX

### Guardado Automático de Progreso

```typescript
interface QuizProgress {
  materialId: string;
  assessmentId: string;
  answers: Map<string, string>; // question_id → answer_id
  startedAt: number;
  lastSavedAt: number;
}

// Guardar progreso cada 30 segundos
const autoSave = setInterval(() => {
  const progress: QuizProgress = {
    materialId,
    assessmentId,
    answers: currentAnswers,
    startedAt: startTime,
    lastSavedAt: Date.now()
  };

  localStorage.setItem(
    `quiz_progress_${materialId}`,
    JSON.stringify(progress)
  );

  showToast('Progreso guardado', { duration: 1000 });
}, 30000);

// Recuperar progreso al cargar
function loadProgress(materialId: string): QuizProgress | null {
  const saved = localStorage.getItem(`quiz_progress_${materialId}`);
  if (!saved) return null;

  const progress: QuizProgress = JSON.parse(saved);

  // Verificar que no esté muy antiguo (>1 hora)
  const age = Date.now() - progress.lastSavedAt;
  if (age > 3600000) {
    localStorage.removeItem(`quiz_progress_${materialId}`);
    return null;
  }

  return progress;
}
```

### Confirmación Antes de Enviar

```typescript
function confirmSubmit(
  answers: UserAnswerDTO[],
  totalQuestions: number
): Promise<boolean> {

  const answeredCount = answers.length;
  const unanswered = totalQuestions - answeredCount;

  return new Promise((resolve) => {
    showModal({
      title: '¿Finalizar evaluación?',
      message: unanswered > 0
        ? `Has respondido ${answeredCount} de ${totalQuestions} preguntas. ${unanswered} quedan sin responder.`
        : `Has respondido todas las preguntas.`,
      actions: [
        {
          label: 'Revisar',
          variant: 'secondary',
          onClick: () => resolve(false)
        },
        {
          label: 'Enviar Evaluación',
          variant: 'primary',
          onClick: () => resolve(true)
        }
      ]
    });
  });
}
```

### Animación de Resultados

```typescript
function animateScore(finalScore: number, duration: number = 2000) {
  const startTime = Date.now();
  const startScore = 0;

  const animate = () => {
    const elapsed = Date.now() - startTime;
    const progress = Math.min(elapsed / duration, 1);

    // Easing function (ease-out cubic)
    const eased = 1 - Math.pow(1 - progress, 3);
    const currentScore = Math.round(startScore + (finalScore - startScore) * eased);

    updateScoreDisplay(currentScore);

    if (progress < 1) {
      requestAnimationFrame(animate);
    } else {
      // Mostrar estrellas o confetti si aprobó
      if (finalScore >= passingScore) {
        showConfetti();
      }
    }
  };

  requestAnimationFrame(animate);
}
```

### Retroalimentación Visual

```typescript
function renderFeedbackItem(feedback: AnswerFeedbackDTO) {
  return (
    <div className={`feedback-item ${feedback.is_correct ? 'correct' : 'incorrect'}`}>
      <div className="feedback-header">
        <span className="feedback-icon">
          {feedback.is_correct ? '✅' : '❌'}
        </span>
        <span className="question-text">{feedback.question_text}</span>
        <span className="points">
          {feedback.points_earned}/{feedback.max_points} pts
        </span>
      </div>

      {!feedback.is_correct && (
        <div className="feedback-details">
          <div className="your-answer">
            <strong>Tu respuesta:</strong> {feedback.selected_answer_text}
          </div>
          <div className="correct-answer">
            <strong>Correcta:</strong> {feedback.correct_answer_text}
          </div>
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

## Almacenamiento Local

### Guardar Intento Fallido

```typescript
interface FailedAttempt {
  materialId: string;
  assessmentId: string;
  answers: UserAnswerDTO[];
  time_spent_seconds: number;
  failedAt: number;
  error: string;
}

function saveFailedAttempt(
  materialId: string,
  assessmentId: string,
  answers: UserAnswerDTO[],
  time_spent_seconds: number,
  error: Error
) {
  const failedAttempt: FailedAttempt = {
    materialId,
    assessmentId,
    answers,
    time_spent_seconds,
    failedAt: Date.now(),
    error: error.message
  };

  localStorage.setItem(
    `failed_attempt_${materialId}`,
    JSON.stringify(failedAttempt)
  );
}

function retryFailedAttempt(materialId: string): Promise<AttemptResultResponse> {
  const saved = localStorage.getItem(`failed_attempt_${materialId}`);
  if (!saved) throw new Error('No failed attempt found');

  const attempt: FailedAttempt = JSON.parse(saved);

  return createAttempt(
    attempt.materialId,
    {
      answers: attempt.answers,
      time_spent_seconds: attempt.time_spent_seconds
    }
  ).then(result => {
    // Éxito, limpiar intento fallido
    localStorage.removeItem(`failed_attempt_${materialId}`);
    return result;
  });
}
```

### Caché de Resultados

```typescript
interface CachedAttemptResult {
  attemptId: string;
  result: AttemptResultResponse;
  cachedAt: number;
}

function cacheAttemptResult(
  attemptId: string,
  result: AttemptResultResponse
) {
  const cached: CachedAttemptResult = {
    attemptId,
    result,
    cachedAt: Date.now()
  };

  localStorage.setItem(
    `attempt_result_${attemptId}`,
    JSON.stringify(cached)
  );
}

function getCachedAttemptResult(
  attemptId: string
): AttemptResultResponse | null {
  const saved = localStorage.getItem(`attempt_result_${attemptId}`);
  if (!saved) return null;

  const cached: CachedAttemptResult = JSON.parse(saved);

  // No expira (los resultados son inmutables)
  return cached.result;
}
```

## Código de Ejemplo (Mobile - TypeScript)

### Service: AttemptService

```typescript
// services/AttemptService.ts

import axios, { AxiosInstance } from 'axios';

export class AttemptService {
  private api: AxiosInstance;

  constructor(baseURL: string) {
    this.api = axios.create({
      baseURL,
      timeout: 60000, // 60s para dar tiempo al scoring
      headers: {
        'Content-Type': 'application/json'
      }
    });

    this.api.interceptors.request.use((config) => {
      const token = localStorage.getItem('access_token');
      if (token) {
        config.headers.Authorization = `Bearer ${token}`;
      }
      return config;
    });
  }

  /**
   * Crear intento de evaluación
   * @throws MaxAttemptsReachedError si no quedan intentos
   * @throws ValidationError si los datos son inválidos
   */
  async createAttempt(
    materialId: string,
    request: CreateAttemptRequest
  ): Promise<AttemptResultResponse> {

    try {
      const response = await this.api.post<AttemptResultResponse>(
        `/v1/materials/${materialId}/assessment/attempts`,
        request
      );

      // Guardar en cache
      cacheAttemptResult(response.data.attempt_id, response.data);

      return response.data;

    } catch (error: any) {
      // Manejar errores específicos
      if (error.response?.status === 403) {
        const code = error.response.data?.code;
        if (code === 'MAX_ATTEMPTS_REACHED') {
          throw new MaxAttemptsReachedError(
            error.response.data.attempts_used,
            error.response.data.max_attempts
          );
        }
      }

      if (error.response?.status === 400) {
        throw new ValidationError(
          error.response.data.message,
          error.response.data.details
        );
      }

      if (error.response?.status === 404) {
        throw new AssessmentNotFoundError(materialId);
      }

      // Error de red o timeout
      if (error.code === 'ECONNABORTED' || error.code === 'TIMEOUT') {
        // Guardar intento fallido localmente
        saveFailedAttempt(materialId, '', request.answers, request.time_spent_seconds, error);
        throw new NetworkError('Request timeout', error);
      }

      throw error;
    }
  }

  /**
   * Reintentar intento fallido
   */
  async retryFailedAttempt(materialId: string): Promise<AttemptResultResponse> {
    const failed = localStorage.getItem(`failed_attempt_${materialId}`);
    if (!failed) {
      throw new Error('No failed attempt to retry');
    }

    const attempt: FailedAttempt = JSON.parse(failed);

    const result = await this.createAttempt(materialId, {
      answers: attempt.answers,
      time_spent_seconds: attempt.time_spent_seconds
    });

    // Limpiar intento fallido
    localStorage.removeItem(`failed_attempt_${materialId}`);

    return result;
  }
}

// Custom Errors
export class MaxAttemptsReachedError extends Error {
  constructor(
    public attemptsUsed: number,
    public maxAttempts: number
  ) {
    super(`Maximum attempts reached: ${attemptsUsed}/${maxAttempts}`);
    this.name = 'MaxAttemptsReachedError';
  }
}

export class ValidationError extends Error {
  constructor(
    message: string,
    public details?: Record<string, string>
  ) {
    super(message);
    this.name = 'ValidationError';
  }
}

export class AssessmentNotFoundError extends Error {
  constructor(public materialId: string) {
    super(`Assessment not found for material ${materialId}`);
    this.name = 'AssessmentNotFoundError';
  }
}

export class NetworkError extends Error {
  constructor(message: string, public originalError: any) {
    super(message);
    this.name = 'NetworkError';
  }
}
```

### Hook: useQuizAttempt

```typescript
// hooks/useQuizAttempt.ts

import { useState, useCallback } from 'react';
import { AttemptService } from '../services/AttemptService';

interface UseQuizAttemptReturn {
  submitAttempt: (answers: UserAnswerDTO[], timeSpent: number) => Promise<void>;
  result: AttemptResultResponse | null;
  loading: boolean;
  error: Error | null;
}

export function useQuizAttempt(materialId: string): UseQuizAttemptReturn {
  const [result, setResult] = useState<AttemptResultResponse | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  const service = new AttemptService(
    process.env.REACT_APP_API_MOBILE_URL || 'http://localhost:8080'
  );

  const submitAttempt = useCallback(async (
    answers: UserAnswerDTO[],
    timeSpent: number
  ) => {
    setLoading(true);
    setError(null);

    try {
      const response = await service.createAttempt(materialId, {
        answers,
        time_spent_seconds: timeSpent
      });

      setResult(response);

      // Mostrar notificación de éxito
      if (response.passed) {
        showSuccessNotification('¡Felicidades! Has aprobado la evaluación');
      } else {
        showInfoNotification('Evaluación completada. Revisa tus resultados');
      }

    } catch (err: any) {
      setError(err);

      if (err instanceof MaxAttemptsReachedError) {
        showErrorNotification(`Sin intentos disponibles (${err.attemptsUsed}/${err.maxAttempts})`);
      } else if (err instanceof NetworkError) {
        showWarningNotification('Error de conexión. Tus respuestas se guardaron localmente');
      } else {
        showErrorNotification('Error al enviar la evaluación');
      }
    } finally {
      setLoading(false);
    }
  }, [materialId]);

  return {
    submitAttempt,
    result,
    loading,
    error
  };
}
```

### Componente: QuizSubmitter

```typescript
// components/QuizSubmitter.tsx

import React, { useState } from 'react';
import { useQuizAttempt } from '../hooks/useQuizAttempt';
import { ConfirmationModal } from './ConfirmationModal';
import { LoadingOverlay } from './LoadingOverlay';
import { ResultsView } from './ResultsView';

interface QuizSubmitterProps {
  materialId: string;
  assessmentId: string;
  answers: Map<string, string>; // question_id → answer_id
  totalQuestions: number;
  timeSpent: number;
  onBack?: () => void;
}

export const QuizSubmitter: React.FC<QuizSubmitterProps> = ({
  materialId,
  assessmentId,
  answers,
  totalQuestions,
  timeSpent,
  onBack
}) => {

  const [showConfirmation, setShowConfirmation] = useState(false);
  const { submitAttempt, result, loading, error } = useQuizAttempt(materialId);

  const handleSubmitClick = () => {
    setShowConfirmation(true);
  };

  const handleConfirm = async () => {
    setShowConfirmation(false);

    // Convertir Map a array de UserAnswerDTO
    const answersArray: UserAnswerDTO[] = Array.from(answers.entries()).map(
      ([question_id, selected_answer_id]) => ({
        question_id,
        selected_answer_id,
        time_spent_seconds: 0 // TODO: trackear tiempo por pregunta
      })
    );

    await submitAttempt(answersArray, timeSpent);
  };

  // Si ya hay resultado, mostrar vista de resultados
  if (result) {
    return <ResultsView result={result} onBack={onBack} />;
  }

  const answeredCount = answers.size;
  const unansweredCount = totalQuestions - answeredCount;

  return (
    <>
      <button
        onClick={handleSubmitClick}
        disabled={answeredCount === 0}
        className="submit-button"
      >
        Finalizar Evaluación
      </button>

      {showConfirmation && (
        <ConfirmationModal
          title="¿Finalizar evaluación?"
          message={
            unansweredCount > 0
              ? `Has respondido ${answeredCount} de ${totalQuestions} preguntas. ${unansweredCount} quedan sin responder.`
              : `Has respondido todas las preguntas.`
          }
          warning="No podrás cambiar tus respuestas después de enviar."
          onConfirm={handleConfirm}
          onCancel={() => setShowConfirmation(false)}
        />
      )}

      {loading && (
        <LoadingOverlay message="Enviando respuestas y calculando calificación..." />
      )}

      {error && !loading && (
        <ErrorBanner error={error} onRetry={handleSubmitClick} />
      )}
    </>
  );
};
```

## Notas de Implementación

### 1. Scoring en Servidor (Seguridad)

**Crítico:**
- Las respuestas correctas NUNCA deben enviarse al cliente
- El scoring SIEMPRE se realiza en el servidor
- MongoDB almacena `correct_answer` que solo lee el backend

**Flujo de Validación:**
```typescript
// BACKEND (API Mobile)
async function scoreAttempt(
  answers: UserAnswerDTO[],
  assessmentId: string
): Promise<AttemptResultResponse> {

  // 1. Obtener preguntas con respuestas correctas de MongoDB
  const assessment = await assessmentRepository.findByIdWithAnswers(assessmentId);

  // 2. Calcular score
  let correct = 0;
  const feedback: AnswerFeedbackDTO[] = [];

  for (const answer of answers) {
    const question = assessment.questions.find(q => q.id === answer.question_id);
    if (!question) continue;

    const isCorrect = answer.selected_answer_id === question.correct_answer;
    if (isCorrect) correct++;

    feedback.push({
      question_id: question.id,
      question_text: question.text,
      is_correct: isCorrect,
      selected_answer: answer.selected_answer_id,
      selected_answer_text: question.options.find(o => o.id === answer.selected_answer_id)?.text || '',
      correct_answer: question.correct_answer,
      correct_answer_text: question.options.find(o => o.id === question.correct_answer)?.text || '',
      explanation: question.explanation || '',
      points_earned: isCorrect ? question.points : 0,
      max_points: question.points
    });
  }

  const score = (correct / assessment.questions.length) * 100;
  const passed = score >= assessment.pass_threshold;

  return {
    score,
    passed,
    feedback,
    // ...
  };
}
```

### 2. Validación de Intentos

**Lógica:**
```typescript
// BACKEND
async function validateAttempts(
  userId: string,
  assessmentId: string,
  maxAttempts: number
): Promise<void> {

  const attempts = await attemptRepository.countByUserAndAssessment(
    userId,
    assessmentId
  );

  if (attempts >= maxAttempts) {
    throw new MaxAttemptsReachedError(attempts, maxAttempts);
  }
}
```

### 3. Tiempo de Respuesta

**SLA:**
- Scoring de 10 preguntas: <2s
- Scoring de 50 preguntas: <5s
- Timeout frontend: 60s

**Optimización:**
```typescript
// BACKEND - Usar consultas eficientes
const assessment = await assessmentRepository.findByIdWithAnswers(
  assessmentId,
  {
    select: ['id', 'questions.id', 'questions.correct_answer', 'questions.points']
  }
);
// Solo traer campos necesarios para scoring
```

### 4. Testing

**Unit Tests:**
```typescript
describe('AttemptService', () => {
  it('should create attempt successfully', async () => {
    const service = new AttemptService('http://test');
    const result = await service.createAttempt('material-123', {
      answers: mockAnswers,
      time_spent_seconds: 600
    });
    expect(result.score).toBeGreaterThanOrEqual(0);
    expect(result.score).toBeLessThanOrEqual(100);
  });

  it('should throw MaxAttemptsReachedError when no attempts left', async () => {
    await expect(
      service.createAttempt('material-123', mockRequest)
    ).rejects.toThrow(MaxAttemptsReachedError);
  });
});
```

**E2E Tests:**
```typescript
describe('Quiz Flow', () => {
  it('should complete full quiz flow', async () => {
    // 1. Obtener quiz
    const quiz = await getAssessment('material-123');

    // 2. Simular respuestas
    const answers = quiz.questions.map(q => ({
      question_id: q.id,
      selected_answer_id: q.options[0].id, // Primera opción
      time_spent_seconds: 30
    }));

    // 3. Enviar intento
    const result = await createAttempt('material-123', {
      answers,
      time_spent_seconds: 600
    });

    // 4. Verificar resultado
    expect(result.attempt_id).toBeDefined();
    expect(result.feedback.length).toBe(quiz.questions.length);
  });
});
```

### 5. Monitoreo

**Métricas:**
- Tiempo promedio de scoring
- Distribución de scores (histograma)
- Tasa de aprobación
- Intentos promedio hasta aprobar

**Logging:**
```typescript
logger.info('Attempt created', {
  userId,
  materialId,
  assessmentId,
  score: result.score,
  passed: result.passed,
  attemptsUsed: result.attempts_used,
  timeSpent: request.time_spent_seconds
});
```
