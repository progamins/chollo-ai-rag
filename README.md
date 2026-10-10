# 🤖 Chollo & Glam — Asistente de compras con IA (RAG)

[![Python](https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.115+-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-pgvector-4169E1?logo=postgresql&logoColor=white)](https://supabase.com)
[![OpenAI](https://img.shields.io/badge/OpenAI-embeddings+LLM-412991?logo=openai&logoColor=white)](https://openai.com)
[![Vercel](https://img.shields.io/badge/Vercel-serverless-000?logo=vercel&logoColor=white)](https://vercel.com)
[![CI](https://img.shields.io/badge/CI-GitHub_Actions-2088FF?logo=githubactions&logoColor=white)](.github/workflows/ci.yml)
[![License](https://img.shields.io/badge/license-MIT-22C55E)](LICENSE)

Microservicio de **búsqueda semántica y RAG** (Retrieval-Augmented Generation) sobre el catálogo real de la tienda [Chollo & Glam](https://github.com/progamins/website). El asistente entiende lenguaje natural ("un regalo elegante para aniversario"), recupera los productos más relevantes por **similitud de embeddings** (coseno, pgvector) y genera una recomendación **con citas a productos reales** — nunca inventa información del catálogo.

> Evolución independiente del módulo `ai-search/` del repo de la tienda: mismo código, repo propio para despliegue y portfolio.

## Arquitectura

```
                       ┌─────────────────────────────────────────────┐
 Navegador ──POST──►  │ Tienda Astro (repo website)                 │
                       │  /asistente  (UI)                           │
                       │  /api/asistente (proxy server-side)        │
                       └──────────────────┬──────────────────────────┘
                                          │ fetch + X-API-Key (solo server)
                                          ▼
                       ┌─────────────────────────────────────────────┐
                       │ ESTE SERVICIO — FastAPI (Vercel Python)     │
                       │  POST /api/search   → solo embeddings      │
                       │  POST /api/chat     → RAG + LLM + citas    │
                       │  POST /api/ingest   → embeddea catálogo    │
                       └───────┬──────────────────────┬───────────────┘
                               │                      │
                               ▼                      ▼
               ┌───────────────────────┐   ┌──────────────────────────┐
               │ Supabase PostgreSQL   │   │ OpenAI (nube)            │
               │ + pgvector            │   │  · text-embedding-3-small│
               │ · tabla documents     │   │  · gpt-4o-mini           │
               │ · índice IVF (coseno) │   │ O Ollama (local, gratis) │
               │ · RLS activada        │   │  · nomic-embed-text      │
               │ · RPC match_documents │   │  · llama3.1              │
               └───────────────────────┘   └──────────────────────────┘
```

## Stack

| Capa | Tecnología |
|---|---|
| API | Python 3.12 · **FastAPI** · Pydantic (validación estricta) |
| Búsqueda vectorial | **Supabase PostgreSQL + pgvector** (índice IVF, similitud coseno, función RPC, RLS) |
| IA | **OpenAI** (embeddings + LLM) con proveedor alternativo **Ollama** local (API compatible) |
| Ingesta | `scripts/ingest.mjs` — Node, por lotes, idempotente (upsert por `external_id`) |
| Deploy | **Vercel** (Python serverless) · CI con **GitHub Actions** |

## API

| Método | Ruta | Auth | Descripción |
|---|---|---|---|
| GET | `/api/health` | — | Estado de proveedores (sin secretos) |
| POST | `/api/search` | rate limit | Búsqueda semántica: top-k con score |
| POST | `/api/chat` | rate limit | RAG: respuesta LLM con citas `[n]` + sources |
| POST | `/api/ingest` | **X-API-Key** | Embeddea y guarda lotes (≤100 docs) |
| DELETE | `/api/documents?source=` | **X-API-Key** | Borra documentos de un origen |

```bash
# Búsqueda semántica
curl -X POST $BASE/api/search -H 'Content-Type: application/json' \
  -d '{"query":"anillo de plata elegante","limit":5}'

# RAG con citas
curl -X POST $BASE/api/chat -H 'Content-Type: application/json' \
  -d '{"query":"regalo para aniversario hasta S/ 200"}'
```

## Puesta en marcha

### 1 · Supabase (pgvector)

1. Crea un proyecto en [supabase.com](https://supabase.com).
2. **SQL Editor → New query** → ejecuta `supabase/schema.sql` completo.
3. Anota **Project URL** y **service_role key** (Settings → API). La tabla queda con **RLS activada y sin políticas públicas**: solo accede la service_role key.

### 2 · Vercel

1. **Add New → Project** → importa este repo.
2. Framework Preset: *Other* — Vercel detecta `api/index.py` (Python) automáticamente.
3. **Environment Variables** (Production + Preview):

| Variable | Valor |
|---|---|
| `SUPABASE_URL` | `https://TU-PROYECTO.supabase.co` |
| `SUPABASE_SERVICE_KEY` | service_role key (secreta) |
| `API_KEY` | `python -c "import secrets; print(secrets.token_urlsafe(32))"` |
| `AI_PROVIDER` | `openai` (o `ollama`) |
| `OPENAI_API_KEY` | clave de OpenAI (si provider=openai) |
| `EMBEDDING_MODEL` | `text-embedding-3-small` |
| `EMBEDDING_DIM` | `1536` (OpenAI) · `768` (Ollama nomic-embed-text) |
| `CHAT_MODEL` | `gpt-4o-mini` |
| `ALLOWED_ORIGINS` | dominio de la tienda |

4. Deploy y verifica: `GET https://tu-proyecto.vercel.app/api/health`

### 3 · Ingesta del catálogo

```bash
npm install better-sqlite3
AI_SEARCH_URL="https://tu-proyecto.vercel.app" \
AI_SEARCH_KEY="la-misma-API_KEY" \
DB_PATH="ruta/a/chollo.db" \
node scripts/ingest.mjs
```

Con `REINGEST_ALL=1` reemplaza todo el origen `catalogo`.

### 4 · Conectar la tienda

En el servidor del sitio Astro ([repo website](https://github.com/progamins/website)):

```
AI_SEARCH_URL=https://tu-proyecto.vercel.app
AI_SEARCH_KEY=la-misma-API_KEY
```

## Desarrollo local

```bash
python -m venv venv && venv\Scripts\activate      # Windows (Linux/macOS: source venv/bin/activate)
pip install -r requirements.txt
copy .env.example .env                            # rellena tus claves
uvicorn api.index:app --reload --port 8000
```

## Seguridad

- **Row-Level Security** activada en la tabla (solo service_role accede)
- **API key** para endpoints administrativos (comparación en tiempo constante `hmac.compare_digest`)
- **Rate limiting** por IP con ventana deslizante (search/chat 20/min, ingest 10, delete 5)
- **Validación estricta** con Pydantic: longitudes, rangos, tipos en TODA entrada
- **CORS con lista explícita** de orígenes — nunca `*`
- **Security headers** en todas las respuestas (CSP `default-src 'none'`, nosniff, X-Frame-Options DENY, no-store)
- **Errores genéricos** al cliente — sin stack traces ni detalles internos
- **Secretos solo en variables de entorno** (`.env.example` documentado, `.gitignore`)
- Prompt con **guardrails**: el modelo solo responde con el contexto del catálogo y cita fuentes `[n]`
- La tienda consume el servicio vía **proxy server-side**: URL y claves nunca llegan al navegador

## Qué demuestra este proyecto

- **IA aplicada de extremo a extremo**: embeddings, búsqueda semántica, RAG con citas, prompt engineering con guardrails
- **Python + FastAPI**: API REST con validación Pydantic, middleware, manejo de errores
- **PostgreSQL avanzado**: pgvector, índices IVF, funciones RPC, Row-Level Security
- **Doble proveedor de IA** con la misma base de código: OpenAI (nube) ↔ Ollama (local)
- **Cloud + CI/CD**: Vercel serverless, GitHub Actions, configuración por entorno, ingesta idempotente
- **Integración con producto real**: no es un demo aislado — consume el catálogo en producción de la tienda

## Licencia

[MIT](LICENSE) · © 2026 Edwin Rosas Albines — Progamins
