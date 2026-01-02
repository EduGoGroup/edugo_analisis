# RFC-040: Obtener Resumen Generado por IA

## Metadata
- **ID:** RFC-040
- **Proceso:** Resúmenes IA
- **Subproceso:** Obtener Resumen de Material
- **Prioridad:** Alta
- **Dependencias:** RFC-010 (Listar Materiales), RFC-012 (Ver Detalle Material), Worker (crítico)
- **Estado API:** ⚠️ IMPLEMENTADO CON BUG (requiere fix de nombre de colección)

> **ACTUALIZACIÓN 2025-01-02:** Este RFC está implementado pero con un bug:
> - **Worker:** ✅ Genera resumen y guarda en MongoDB `material_summaries`
> - **API Mobile:** ✅ Endpoint `GET /v1/materials/:id/summary` existe (`summary_handler.go`)
> - **BUG:** 🐛 API Mobile busca en colección `material_summary` (singular) pero Worker guarda en `material_summaries` (plural)
> - **FIX REQUERIDO:** Cambiar línea 20 de `summary_repository_impl.go` de `material_summary` a `material_summaries`

## Descripción

Permite a los estudiantes obtener el resumen generado automáticamente por IA del contenido de un material educativo. El resumen se genera durante el procesamiento del PDF por el Worker y se almacena en MongoDB.

## Flujo de Usuario (UX)

### Pantalla: Resumen del Material

```
┌─────────────────────────────────────────┐
│  ← Matemáticas Grado 5                  │
│                                          │
│  [📄 Ver PDF]  [📝 Quiz]  [📖 Resumen]  │
│                                          │
│  ┌────────────────────────────────┐     │
│  │  📖 Resumen Generado por IA     │     │
│  │                                 │     │
│  │  🎯 Ideas Principales           │     │
│  │  • Las funciones matemáticas... │     │
│  │  • El álgebra básica permite... │     │
│  │  • Los teoremas fundamentales... │     │
│  │                                 │     │
│  │  📚 Conceptos Clave              │     │
│  │  ┌──────────────────────┐       │     │
│  │  │ Función              │       │     │
│  │  │ Una relación que...  │       │     │
│  │  └──────────────────────┘       │     │
│  │  ┌──────────────────────┐       │     │
│  │  │ Ecuación             │       │     │
│  │  │ Igualdad con...      │       │     │
│  │  └──────────────────────┘       │     │
│  │                                 │     │
│  │  📋 Secciones                    │     │
│  │  1. Introducción                 │     │
│  │     Este material explica...     │     │
│  │                                 │     │
│  │  2. Desarrollo                   │     │
│  │     Los conceptos principales... │     │
│  │                                 │     │
│  │  3. Conclusión                   │     │
│  │     El estudio del álgebra...    │     │
│  │                                 │     │
│  │  📊 Estadísticas                 │     │
│  │  Palabras: ~1,500               │     │
│  │  Tiempo lectura: ~6 min         │     │
│  └────────────────────────────────┘     │
│                                          │
│  🤖 Generado con IA                      │
│  Última actualización: 24 Dic 2024       │
└─────────────────────────────────────────┘
```

### Estados del Resumen

**Si disponible (status='ready'):**
- Mostrar resumen completo
- Ideas principales
- Conceptos clave
- Secciones organizadas

**Si procesando (status='processing'):**
```
┌──────────────────────────────────────┐
│  ⏳ Generando resumen con IA...      │
│  ────────────────── 60%              │
│  El resumen estará disponible en     │
│  aproximadamente 1 minuto            │
│                                      │
│  [Actualizar]  [Notificarme]         │
└──────────────────────────────────────┘
```

**Si no disponible (404):**
```
┌──────────────────────────────────────┐
│  🤖 Resumen aún no generado           │
│  El sistema está analizando el       │
│  contenido del material.             │
│  Intenta nuevamente en unos minutos. │
│                                      │
│  [Actualizar]  [Ver Material]        │
└──────────────────────────────────────┘
```

## Flujo de Datos (Técnico)

### Diagrama de Secuencia

```
Usuario → Frontend → API Mobile → MongoDB
                                     ↓
                              material_summary
                              (colección)
```

### Flujo Detallado

1. **Usuario hace clic en "Ver Resumen"**

2. **Frontend verifica estado del material:**
   ```typescript
   const material = await getMaterial(materialId);

   if (material.status !== 'ready') {
     showProcessingState(material.status);
     return;
   }
   ```

3. **Frontend solicita resumen:**
   ```typescript
   try {
     const summary = await getSummary(materialId);
     showSummary(summary);
   } catch (error) {
     if (error.status === 404) {
       showNotReadyMessage();
       startPolling(); // Cada 10s
     }
   }
   ```

4. **API Mobile consulta MongoDB:**
   - Colección: `material_summary`
   - Query: `{ "material_id": "uuid" }`

5. **API Mobile retorna resumen**

### Endpoints Involucrados

**Principal:**
```
GET /v1/materials/:id/summary
```

**Prerequisito:**
```
GET /v1/materials/:id
```

### Request/Response (TypeScript)

**Request:**
```typescript
// Sin body, solo materialId en URL
```

**Response Exitoso (200):**
```typescript
interface SummaryResponse {
  material_id: string;
  main_ideas: string[];              // Ideas principales (3-5)
  key_concepts: KeyConceptDTO[];     // Conceptos clave (5-10)
  sections: SectionDTO[];            // Secciones organizadas
  glossary?: GlossaryEntryDTO[];     // Glosario opcional
  word_count: number;                // Total palabras
  estimated_reading_time: number;    // Minutos
  generated_at: string;              // ISO8601
  model_version?: string;            // "gpt-4", "claude-3", "fallback"
}

interface KeyConceptDTO {
  term: string;                      // "Función"
  definition: string;                // "Una relación que..."
}

interface SectionDTO {
  title: string;                     // "Introducción"
  content: string;                   // Texto de la sección
  order: number;                     // Orden de presentación
}

interface GlossaryEntryDTO {
  term: string;
  definition: string;
}
```

**Ejemplo Response:**
```json
{
  "material_id": "material-uuid-123",
  "main_ideas": [
    "Las funciones matemáticas establecen relaciones entre conjuntos de números",
    "El álgebra básica permite resolver ecuaciones de primer grado",
    "Los teoremas fundamentales del álgebra son aplicables en múltiples contextos"
  ],
  "key_concepts": [
    {
      "term": "Función",
      "definition": "Una relación que asigna a cada elemento de un conjunto exactamente un elemento de otro conjunto"
    },
    {
      "term": "Ecuación",
      "definition": "Igualdad matemática que contiene una o más incógnitas"
    }
  ],
  "sections": [
    {
      "title": "Introducción",
      "content": "Este material explica los conceptos fundamentales del álgebra básica...",
      "order": 1
    },
    {
      "title": "Desarrollo",
      "content": "Los conceptos principales a estudiar son las funciones, ecuaciones...",
      "order": 2
    }
  ],
  "word_count": 1500,
  "estimated_reading_time": 6,
  "generated_at": "2024-12-24T10:02:00Z",
  "model_version": "gpt-4-turbo-preview"
}
```

**Response Error (404 - No generado):**
```json
{
  "error": "not_found",
  "message": "Summary not generated yet",
  "code": "SUMMARY_NOT_READY"
}
```

**Response Error (404 - Material no existe):**
```json
{
  "error": "not_found",
  "message": "Material not found",
  "code": "MATERIAL_NOT_FOUND"
}
```

## Estados y Transiciones

### Estado del Material (Prerequisito)

```
Material States (para resumen):

┌──────────┐
│ uploaded │ → PDF subido, worker no ha procesado
└──────────┘
     ↓
┌────────────┐
│ processing │ → Worker extrayendo y generando resumen
└────────────┘
     ↓
┌───────┐
│ ready │ → Resumen generado y disponible ✅
└───────┘
     ↓ (si falla)
┌────────┐
│ failed │ → Error en generación ❌
└────────┘
```

### Disponibilidad del Resumen

```
Summary Availability:

┌────────────────┐
│ NOT_GENERATED  │ → Material en processing/uploaded
│ (404)          │ → Mostrar "Generando..."
└────────────────┘
        ↓
┌────────────────┐
│ AVAILABLE      │ → Material en ready
│ (200)          │ → Mostrar resumen completo
└────────────────┘
        ↓ (si worker falla)
┌────────────────┐
│ GENERATION_    │ → Material en failed
│ FAILED (404)   │ → Mostrar error
└────────────────┘
```

## Manejo de Errores

### Errores de Red

**Timeout:**
```typescript
try {
  const summary = await getSummary(materialId, { timeout: 30000 });
} catch (error) {
  if (error.code === 'TIMEOUT') {
    showError('La conexión está lenta. Intenta nuevamente.');
    offerRetry();
  }
}
```

### Errores de Negocio

**404 - Resumen no generado:**
```typescript
if (error.status === 404 && error.code === 'SUMMARY_NOT_READY') {
  showMessage('El resumen está siendo generado...');
  startPolling(materialId, {
    interval: 10000,     // Cada 10 segundos
    maxAttempts: 60,     // 10 minutos total
    onReady: () => loadSummary(materialId),
    onTimeout: () => showTimeoutMessage()
  });
}
```

**404 - Material no existe:**
```typescript
if (error.status === 404 && error.code === 'MATERIAL_NOT_FOUND') {
  showError('Material no encontrado');
  redirectTo('/materials');
}
```

### Degradación Graceful

**Worker con Fallback activo:**
- Resumen generado con calidad reducida
- Basado en extracción simple (primeras oraciones)
- Mostrar disclaimer:
  ```
  ⚠️ Este resumen fue generado automáticamente
     y puede no capturar todos los detalles.
  ```

## Consideraciones de UX

### Polling para Resumen en Procesamiento

```typescript
async function pollUntilSummaryReady(
  materialId: string,
  options: PollingOptions = { interval: 10000, maxAttempts: 60 }
): Promise<SummaryResponse> {

  let attempt = 0;

  while (attempt < options.maxAttempts) {
    try {
      const summary = await getSummary(materialId);
      return summary; // Éxito

    } catch (error) {
      if (error.status === 404 && error.code === 'SUMMARY_NOT_READY') {
        // Continuar polling
        await sleep(options.interval);
        attempt++;
        updateProgress(attempt, options.maxAttempts);

      } else {
        throw error; // Error real
      }
    }
  }

  throw new Error('Timeout: Summary not ready after polling');
}
```

### Skeleton Loader para Resumen

```typescript
function SummarySkeleton() {
  return (
    <div className="summary-skeleton">
      <div className="skeleton-section">
        <div className="skeleton-title" />
        <div className="skeleton-list">
          <div className="skeleton-item" />
          <div className="skeleton-item" />
          <div className="skeleton-item" />
        </div>
      </div>

      <div className="skeleton-section">
        <div className="skeleton-title" />
        <div className="skeleton-cards">
          <div className="skeleton-card" />
          <div className="skeleton-card" />
          <div className="skeleton-card" />
        </div>
      </div>

      <div className="skeleton-section">
        <div className="skeleton-title" />
        <div className="skeleton-text" />
        <div className="skeleton-text" />
        <div className="skeleton-text" />
      </div>
    </div>
  );
}
```

### Visualización de Resumen

```typescript
function SummaryView({ summary }: { summary: SummaryResponse }) {
  return (
    <div className="summary-view">
      {/* Ideas Principales */}
      <section className="main-ideas">
        <h2>🎯 Ideas Principales</h2>
        <ul>
          {summary.main_ideas.map((idea, i) => (
            <li key={i}>{idea}</li>
          ))}
        </ul>
      </section>

      {/* Conceptos Clave */}
      <section className="key-concepts">
        <h2>📚 Conceptos Clave</h2>
        <div className="concepts-grid">
          {summary.key_concepts.map((concept, i) => (
            <div key={i} className="concept-card">
              <h3>{concept.term}</h3>
              <p>{concept.definition}</p>
            </div>
          ))}
        </div>
      </section>

      {/* Secciones */}
      <section className="sections">
        <h2>📋 Contenido Organizado</h2>
        {summary.sections.map((section, i) => (
          <div key={i} className="section">
            <h3>{section.order}. {section.title}</h3>
            <p>{section.content}</p>
          </div>
        ))}
      </section>

      {/* Glosario */}
      {summary.glossary && summary.glossary.length > 0 && (
        <section className="glossary">
          <h2>📖 Glosario</h2>
          <dl>
            {summary.glossary.map((entry, i) => (
              <React.Fragment key={i}>
                <dt>{entry.term}</dt>
                <dd>{entry.definition}</dd>
              </React.Fragment>
            ))}
          </dl>
        </section>
      )}

      {/* Footer */}
      <footer className="summary-footer">
        <p>
          🤖 Generado con IA
          {summary.model_version && ` (${summary.model_version})`}
        </p>
        <p>
          📊 {summary.word_count} palabras ·
          ⏱️ ~{summary.estimated_reading_time} min de lectura
        </p>
        <p className="generated-date">
          Generado: {formatDate(summary.generated_at)}
        </p>
      </footer>
    </div>
  );
}
```

## Almacenamiento Local

### Caché de Resumen

```typescript
interface SummaryCache {
  materialId: string;
  summary: SummaryResponse;
  cachedAt: number;
  expiresAt: number;
}

// Guardar en localStorage (cache largo: 1 día)
function cacheSummary(
  materialId: string,
  summary: SummaryResponse,
  ttlSeconds: number = 86400 // 24 horas
) {
  const cache: SummaryCache = {
    materialId,
    summary,
    cachedAt: Date.now(),
    expiresAt: Date.now() + (ttlSeconds * 1000)
  };

  localStorage.setItem(
    `summary_${materialId}`,
    JSON.stringify(cache)
  );
}

// Recuperar de cache
function getCachedSummary(materialId: string): SummaryResponse | null {
  const cached = localStorage.getItem(`summary_${materialId}`);
  if (!cached) return null;

  const cache: SummaryCache = JSON.parse(cached);

  // Verificar expiración
  if (Date.now() > cache.expiresAt) {
    localStorage.removeItem(`summary_${materialId}`);
    return null;
  }

  return cache.summary;
}
```

## Código de Ejemplo (Mobile - TypeScript)

### Service: SummaryService

```typescript
// services/SummaryService.ts

export class SummaryService {
  private api: AxiosInstance;

  constructor(baseURL: string) {
    this.api = axios.create({
      baseURL,
      timeout: 30000,
      headers: { 'Content-Type': 'application/json' }
    });

    this.api.interceptors.request.use((config) => {
      const token = localStorage.getItem('access_token');
      if (token) {
        config.headers.Authorization = `Bearer ${token}`;
      }
      return config;
    });
  }

  async getSummary(materialId: string): Promise<SummaryResponse> {
    // Verificar cache primero
    const cached = getCachedSummary(materialId);
    if (cached) return cached;

    // Obtener de API
    try {
      const response = await this.api.get<SummaryResponse>(
        `/v1/materials/${materialId}/summary`
      );

      // Guardar en cache
      cacheSummary(materialId, response.data);

      return response.data;

    } catch (error: any) {
      if (error.response?.status === 404) {
        const code = error.response.data?.code;

        if (code === 'SUMMARY_NOT_READY') {
          throw new SummaryNotReadyError(materialId);
        }

        if (code === 'MATERIAL_NOT_FOUND') {
          throw new MaterialNotFoundError(materialId);
        }
      }

      throw new NetworkError('Failed to fetch summary', error);
    }
  }

  async waitForSummary(
    materialId: string,
    options: PollingOptions = { interval: 10000, maxAttempts: 60 }
  ): Promise<SummaryResponse> {

    let attempt = 0;

    while (attempt < options.maxAttempts) {
      try {
        return await this.getSummary(materialId);

      } catch (error) {
        if (error instanceof SummaryNotReadyError) {
          await this.sleep(options.interval);
          attempt++;
          continue;
        }
        throw error;
      }
    }

    throw new PollingTimeoutError('Summary not ready after polling');
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// Custom Errors
export class SummaryNotReadyError extends Error {
  constructor(public materialId: string) {
    super(`Summary not ready for material ${materialId}`);
    this.name = 'SummaryNotReadyError';
  }
}

export class MaterialNotFoundError extends Error {
  constructor(public materialId: string) {
    super(`Material ${materialId} not found`);
    this.name = 'MaterialNotFoundError';
  }
}

export class PollingTimeoutError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'PollingTimeoutError';
  }
}

export class NetworkError extends Error {
  constructor(message: string, public originalError: any) {
    super(message);
    this.name = 'NetworkError';
  }
}
```

### Hook: useSummary

```typescript
// hooks/useSummary.ts

export function useSummary(
  materialId: string,
  options: { autoFetch?: boolean; enablePolling?: boolean } = {}
) {
  const [summary, setSummary] = useState<SummaryResponse | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  const [isReady, setIsReady] = useState(false);

  const service = new SummaryService(
    process.env.REACT_APP_API_MOBILE_URL || 'http://localhost:8080'
  );

  const fetchSummary = useCallback(async () => {
    setLoading(true);
    setError(null);

    try {
      const result = options.enablePolling
        ? await service.waitForSummary(materialId)
        : await service.getSummary(materialId);

      setSummary(result);
      setIsReady(true);

    } catch (err: any) {
      if (err instanceof SummaryNotReadyError) {
        setIsReady(false);
      }
      setError(err);
    } finally {
      setLoading(false);
    }
  }, [materialId, options.enablePolling]);

  useEffect(() => {
    if (options.autoFetch) {
      fetchSummary();
    }
  }, [options.autoFetch, fetchSummary]);

  return {
    summary,
    loading,
    error,
    isReady,
    refetch: fetchSummary
  };
}
```

### Componente: SummaryLoader

```typescript
// components/SummaryLoader.tsx

export const SummaryLoader: React.FC<{ materialId: string }> = ({ materialId }) => {
  const {
    summary,
    loading,
    error,
    isReady,
    refetch
  } = useSummary(materialId, {
    autoFetch: true,
    enablePolling: true
  });

  if (loading && !summary) {
    return <SummarySkeleton />;
  }

  if (error && !isReady) {
    return (
      <ProcessingMessage
        message="El resumen está siendo generado..."
        onRetry={refetch}
      />
    );
  }

  if (error && !summary) {
    return (
      <ErrorMessage
        message="Error al cargar el resumen"
        onRetry={refetch}
      />
    );
  }

  if (summary) {
    return <SummaryView summary={summary} />;
  }

  return null;
};
```

## Notas de Implementación

### 1. Dependencia Crítica del Worker

**Bloqueante:**
- El Worker procesa el PDF con IA (OpenAI/Claude/Fallback)
- Sin Worker: 404 permanente

**Verificación:**
```bash
# DevOps debe validar:
curl http://worker:8083/health

# Verificar cola RabbitMQ
rabbitmqctl list_queues | grep material.uploaded
```

### 2. Tiempos de Generación

**Estimaciones:**
- PDF 10 páginas: ~30-60 segundos
- PDF 50 páginas: ~1-3 minutos
- PDF 100 páginas: ~3-8 minutos

### 3. Calidad del Resumen

**Con OpenAI/Claude:**
- Ideas principales bien identificadas
- Conceptos clave precisos
- Secciones bien organizadas

**Con Fallback:**
- Ideas principales = primeras oraciones
- Conceptos = palabras más frecuentes
- Secciones = división por tercios
- Mostrar disclaimer

### 4. Accesibilidad

```typescript
<section role="region" aria-label="Resumen del material">
  <h2 id="summary-heading">Resumen Generado por IA</h2>
  <div aria-labelledby="summary-heading">
    {/* Contenido */}
  </div>
</section>
```

### 5. Testing

```typescript
describe('SummaryService', () => {
  it('should fetch summary successfully', async () => {
    const service = new SummaryService('http://test');
    const summary = await service.getSummary('material-123');
    expect(summary.main_ideas.length).toBeGreaterThan(0);
  });

  it('should throw SummaryNotReadyError on 404', async () => {
    await expect(
      service.getSummary('material-123')
    ).rejects.toThrow(SummaryNotReadyError);
  });
});
```
