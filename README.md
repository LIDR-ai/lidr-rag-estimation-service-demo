# Sistema RAG de estimación de proyectos

Plataforma web para estimar presupuestos de proyectos tecnológicos usando **RAG** (Retrieval-Augmented Generation). Permite ingestar presupuestos históricos, indexarlos con embeddings vectoriales, y generar estimaciones de horas y costes para nuevos proyectos basándose en datos reales.

El sistema consta de dos repositorios independientes:

- **[rag-estimation-platform](https://github.com/LIDR-ai/lidr-rag-estimation-platform-demo)** — Aplicación web Rails para la gestión de estimaciones, exportación PDF/JSON y UI.
- **rag-estimation-service** (este repo) — Servicio RAG con FastAPI, pgvector y OpenAI para indexación y estimación con IA.

## 🎥 Vídeo del tutorial

[![Ver el tutorial](https://img.youtube.com/vi/E-Hm9IRViYo/maxresdefault.jpg)](https://youtu.be/E-Hm9IRViYo)

▶️ [Ver el tutorial en vídeo](https://youtu.be/E-Hm9IRViYo)

## Descripción

Esta demo avanzada añade varias funcionalidades con IA a un software real de gestión de proyectos de software: síntesis de información a partir de reuniones con clientes, y estimación de tareas, esfuerzo y coste a partir de esa reunión y el histórico de proyectos de la empresa.

Qué aprenderás:

- ✅ Cómo diseñar un sistema RAG real que transforme conversaciones con clientes en estimaciones estructuradas de proyectos software.
- ✅ Qué arquitectura, datos y guardrails hacen falta para pasar de una demo con LLMs a un producto fiable en producción.
- ✅ Cómo integrar IA generativa en proyectos de software reales: bases de datos vectoriales, integración de modelos vía API, indexación de información, chunking y vectorización, búsqueda semántica y más.

Verás cómo conviven en el código los servicios de IA, sin necesidad de flujos complejos ni de piezas externas al proyecto.


## Prerequisites

- **Python 3.11+**
- **Docker and Docker Compose** (for PostgreSQL with pgvector)
- **OpenAI API key** (for embeddings and LLM generation)

## Installation and Local Development

### Option A: All-in-Docker (simplest)

```bash
cp .env.example .env
# Edit .env and set your OPENAI_API_KEY
docker compose up --build
```

### Option B: Local Development (recommended for development)

1. **Clone the repository:**
   ```bash
   git clone <repo-url>
   cd rag-estimation-service
   ```

2. **Create and activate a virtual environment:**
   ```bash
   python3.11 -m venv .venv
   source .venv/bin/activate
   ```

3. **Install dependencies:**
   ```bash
   pip install -e ".[dev]"
   ```

4. **Start PostgreSQL with pgvector:**
   ```bash
   docker compose up -d postgres
   ```

5. **Configure environment variables:**
   ```bash
   cp .env.example .env
   # Edit .env and set your OPENAI_API_KEY
   ```

6. **Run database migrations:**
   ```bash
   alembic upgrade head
   ```

7. **(Optional) Load sample data:**
   ```bash
   python scripts/seed_data.py
   ```

8. **Start the development server:**
   ```bash
   uvicorn app.main:app --reload
   ```

The service will be available at:
- API: http://localhost:8000/api/v1/health
- Swagger UI: http://localhost:8000/docs
- ReDoc: http://localhost:8000/redoc

### Environment Variables

| Variable | Default | Description |
|---|---|---|
| `DATABASE_URL` | `postgresql+asyncpg://dev:dev@localhost:5432/estimations` | PostgreSQL connection string |
| `DATABASE_SCHEMA` | `rag` | Database schema name |
| `OPENAI_API_KEY` | *(required)* | OpenAI API key |
| `EMBEDDING_MODEL` | `text-embedding-3-small` | Embedding model for vectorization |
| `EMBEDDING_DIMENSIONS` | `1536` | Embedding vector dimensions |
| `LLM_MODEL` | `o4-mini` | LLM model for estimation generation |
| `SERVICE_PORT` | `8000` | HTTP server port |
| `ENVIRONMENT` | `development` | `development` / `staging` / `production` |
| `LOG_LEVEL` | `DEBUG` | `DEBUG` / `INFO` / `WARNING` / `ERROR` |
| `API_KEY` | `dev-api-key` | API key for authentication |

## Tests

```bash
pytest
```

## Project Structure

```
rag-estimation-service/
├── app/
│   ├── main.py             # FastAPI app, lifespan, middleware
│   ├── config.py           # Settings (Pydantic BaseSettings)
│   ├── dependencies.py     # Dependency injection
│   ├── api/v1/             # API endpoints (versioned)
│   ├── api/schemas/        # Pydantic request/response schemas
│   ├── core/               # RAG pipeline (chunking, embeddings, retrieval)
│   ├── models/             # SQLAlchemy models
│   ├── services/           # Business logic
│   └── utils/              # Logging, helpers
├── alembic/                # Database migrations
├── estimation_samples/     # Sample historical quote JSON files
├── scripts/
│   └── seed_data.py        # Load sample budgets into the database
├── transcriptions/         # Sample meeting transcriptions
├── tests/                  # Test suite
├── Dockerfile              # Multi-stage build
└── docker-compose.yml      # Dev environment (postgres + service)
```

## API Endpoints

All endpoints are under `/api/v1`. Authentication is done via `X-API-Key` header.

### `GET /health`

Health check for the service and its dependencies (database, pgvector).

**Response:**
```json
{
  "status": "healthy",
  "version": "0.1.0",
  "environment": "development",
  "dependencies": {
    "database": { "status": "healthy", "latency_ms": 1.23 },
    "pgvector": { "status": "healthy", "version": "0.7.0" }
  }
}
```

Status is `"healthy"` when all dependencies are up, `"degraded"` otherwise.

---

### `POST /ingest`

Ingest a historical software quote. The service chunks the quote and generates embeddings for semantic search.

**Request body:** `IngestRequest` — contains a `quote` object with project details, scope blocks, items, team, and conditions. See Swagger UI for the full schema.

**Key fields:** `quote.project.title`, `quote.scope_blocks[]`, `quote.items[]`, `quote.currency`

**Response (200):**
```json
{
  "status": "completed",
  "document_id": "uuid",
  "message": "Quote processed: 8 chunks created"
}
```

**Errors:** `409` duplicate quote, `413` payload > 5MB, `503` embedding service unavailable.

---

### `GET /ingest/{document_id}/status`

Check the ingestion status of a document.

**Response (200):**
```json
{
  "document_id": "uuid",
  "status": "completed",
  "project_title": "E-commerce Platform",
  "chunks_created": 8,
  "processing_time_ms": 1234,
  "breakdown": {
    "project_overview": 1,
    "scope_block": 3,
    "line_item": 3,
    "phase": 1,
    "team_conditions": 0
  }
}
```

**Errors:** `404` document not found.

---

### `DELETE /ingest/{document_id}`

Delete a document and all its associated chunks.

**Response (200):**
```json
{
  "document_id": "uuid",
  "status": "deleted",
  "chunks_deleted": 8
}
```

**Errors:** `404` document not found.

---

### `POST /search`

Semantic search over vectorized chunks from ingested quotes.

**Request body:**
```json
{
  "query": "e-commerce platform with payment integration",
  "top_k": 10,
  "min_similarity": 0.6,
  "filters": {
    "chunk_types": ["scope_block", "line_item"],
    "technologies": ["React", "Node.js"]
  }
}
```

Valid `chunk_types`: `project_overview`, `scope_block`, `line_item`, `phase`, `team_conditions`.

**Response (200):**
```json
{
  "results": [
    {
      "chunk_id": "uuid",
      "chunk_type": "scope_block",
      "similarity_score": 0.89,
      "final_score": 0.91,
      "content_text": "...",
      "project_title": "E-commerce Marketplace",
      "source_document_id": "uuid",
      "technologies": ["Node.js", "Vue.js"]
    }
  ],
  "total_results": 5,
  "query_processing_time_ms": 120,
  "detected_technologies": ["React", "Node.js"],
  "suggested_chunk_types": ["scope_block"]
}
```

**Errors:** `400` empty query, `503` embedding service unavailable.

---

### `POST /estimate`

Generate an effort estimation using RAG. Retrieves similar historical chunks and uses an LLM to produce a structured estimate with three-point estimation (optimistic/expected/pessimistic).

**Request body:**
```json
{
  "query": "Mobile fitness app with social features and gamification",
  "context": {
    "project_type": "mobile app",
    "technologies_preferred": ["React Native", "Firebase"],
    "team_size": 3,
    "complexity": "medium"
  },
  "options": {
    "top_k": 10,
    "min_similarity": 0.6,
    "include_references": true,
    "estimation_format": "detailed",
    "currency": "EUR",
    "skip_validation": false
  }
}
```

**Response (200):** Contains `estimation` (summary, effort hours, confidence score, task breakdown, technologies, notes), `references` (similar historical chunks used), and `metadata` (models used, processing time).

**Errors:** `400` invalid query, `404` no relevant chunks found, `503` LLM/embedding unavailable, `504` LLM timeout.

---

### `POST /estimate/validate`

Validate a task breakdown against historical references. Compares each task's hours with similar past projects and suggests adjustments.

**Request body:**
```json
{
  "breakdown": [
    {
      "name": "Backend Development",
      "tasks": [
        { "name": "API REST implementation", "hours": 40 },
        { "name": "Database design", "hours": 16 }
      ]
    }
  ],
  "estimated_effort": {
    "optimistic": { "hours": 120 },
    "expected": { "hours": 160 },
    "pessimistic": { "hours": 200 }
  },
  "currency": "EUR"
}
```

**Response (200):** Contains `validated_breakdown` (tasks with adjusted hours and reasons), `estimated_effort`, `adjustment_notes`, and validation statistics.

**Errors:** `503` LLM/embedding unavailable, `504` LLM timeout.

---

### `POST /estimate/batch`

Run multiple estimations in a single request (1-20 queries).

**Request body:**
```json
{
  "queries": [
    { "id": "auth-module", "query": "Authentication system with OAuth2 and SSO" },
    { "id": "payments", "query": "Payment gateway integration with Stripe" }
  ],
  "shared_context": {
    "project_type": "web app",
    "complexity": "high"
  },
  "options": {
    "estimation_format": "detailed"
  }
}
```

**Response (200):** Contains `estimations[]` (individual results or errors per query), `aggregated` (total effort and overall confidence), and `metadata`.

**Errors:** `400` invalid request, `503` services unavailable.

---

### `GET /stats`

System statistics: document counts by status, chunk counts by type, and search metrics (last 30 days).

**Response (200):**
```json
{
  "documents_by_status": { "completed": 10, "failed": 1 },
  "chunks_by_type": { "scope_block": 30, "line_item": 45, "project_overview": 10 },
  "search_metrics_last_30d": {
    "total_searches": 150,
    "avg_response_time_ms": 85.3,
    "avg_top_score": 0.7823
  },
  "embedding_model": "text-embedding-3-small",
  "embedding_dimensions": 1536
}
```

---

### `POST /generate-quote`

Generate a complete software quote from a meeting transcription. Uses a 3-step pipeline: (1) analyze transcription with reasoning model, (2) RAG search for similar historical projects, (3) generate detailed quote.

**Request body:**
```json
{
  "transcription": "Full text of the client meeting transcription (min 100 chars)...",
  "context": {
    "client_name": "John Doe",
    "client_company": "Acme Corp",
    "currency": "EUR",
    "technologies_preferred": ["React", "Node.js"],
    "team_size_hint": 4,
    "budget_hint": 50000
  }
}
```

**Response (200):** Contains `quote` (full structured quote ready for delivery), `analysis` (structured analysis of the transcription), and `metadata` (tokens used, processing times, RAG stats).

**Errors:** `400` invalid transcription, `503` reasoning model/embedding unavailable, `504` reasoning model timeout.

---

## Interactive Documentation

For complete request/response schemas and to try out the API interactively:

- **Swagger UI:** http://localhost:8000/docs
- **ReDoc:** http://localhost:8000/redoc

---

## Copyright

Copyright (c) 2026 LIDR.co. Licensed under the MIT License.

**Español:**
El contenido de este repositorio es parte del programa AI Engineering de LIDR.co. Si quieres aprender AI Engineering con los pros, y obtener más plantillas y recursos como estos, puedes encontrar toda la información en la página oficial: https://lidr.co/ai-engineering

**English:**
The content of this repository is part of the AI Engineering program by LIDR.co. If you want to learn AI Engineering with the pros and get more templates and resources like these, you can find all the information on the official website: https://lidr.co/ai-engineering

