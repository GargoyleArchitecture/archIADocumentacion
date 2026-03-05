# Backend ArchIA — Overview para Frontend

## Estructura general del backend

- Entrypoint API principal: `back/src/main.py` (`app = FastAPI(title="ArquIA API", lifespan=lifespan)`).
- App HTTP adicional para exploracion RAG: `back/chroma_web.py` (`app = FastAPI(title="ChromaDB Explorer")`).
- Orquestacion conversacional: `src.graph` (grafo LangGraph invocado desde `POST /message`).
- Persistencia de estado conversacional: `src/memory.py` (`load_arch_flow`, `save_arch_flow`) + SQLite para feedback en `back/feedback_db/feedback.db`.
- Servicios de diagrama: `src/services/diagram_ir.py`, `src/services/diagram_render.py`, `src/services/diagram_export.py`.
- RAG/embeddings: `src/rag_agent.py` + ChromaDB (`back/chroma_db`).

```text
Frontend -> back/src/main.py (FastAPI)
        -> /message -> src.graph (LangGraph) -> diagram/rag/asr nodes
        -> /diagram/export -> diagram_ir + diagram_render
        -> /feedback -> SQLite feedback_db
RAG Explorer (separado): back/chroma_web.py -> Chroma collection
```

- Modulos/capas principales:
- `back/src/main.py` — endpoints HTTP, middleware, lifecycle y armado de payloads para frontend.
- `back/src/graph/` — workflow, nodos y estado del agente.
- `back/src/services/` — rendering/export de diagramas, ingest de documentos, fabrica LLM.
- `back/src/memory.py` — almacenamiento por `user_id/session_id` para contexto de conversacion.
- `back/chroma_web.py` — API y UI para inspeccion del vector store.

## Configuración y ejecución local

- Requisitos segun repo: Python `3.11.x`, Poetry, dependencias (`README.md`, `pyproject.toml`).
- Instalacion:

```bash
poetry install
```

- API principal (puerto 8000):

```bash
cd back
poetry run uvicorn src.main:app --port 8000
```

- ChromaDB Explorer (puerto 8001):

```bash
cd back
poetry run python chroma_web.py
```

- Atajo Windows para Explorer:

```bat
back\start_chroma_web.bat
```

- Indexacion RAG (carga docs en vectorstore):

```bash
cd back
poetry run python build_vectorstore.py
```

- Tests:

```bash
poetry run pytest
```

- Variables de entorno relevantes (definidas/usadas en codigo):
- `OPENAI_API_KEY` (`README.md`, `back/src/services/llm_factory.py`).
- `GRAPHVIZ_ENGINE` (`back/src/main.py`, `back/src/graph/nodes/diagram.py`).
- `CHROMA_DIR` (`back/chroma_web.py`).
- `OPENAI_EMBED_MODEL` (`back/src/rag_agent.py`, `back/build_vectorstore.py`).
- `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_VERSION`, `AZURE_OPENAI_EMBEDDINGS_DEPLOYMENT` (`back/src/rag_agent.py`).

## Autenticación, CORS y headers (si aplica)

- CORS en app principal (`back/src/main.py`):
- `allow_origins`: `http://localhost:5173`, `http://127.0.0.1:5173`.
- `allow_credentials=True`, `allow_methods=["*"]`, `allow_headers=["*"]`.
- Header funcional en `POST /message`:
- `X-User-Id` opcional; si no llega, se usa `session_id` como identidad (`user_id = request.headers.get("X-User-Id") or session_id`).
- Content types usados por endpoints:
- `multipart/form-data` para `POST /message`, `POST /feedback`, `POST /test`.
- `image/svg+xml`, `text/vnd.graphviz`, `application/xml`, `application/json` en respuestas de export.
- Middleware UTF-8 (`UTF8Middleware`) asegura `application/json; charset=utf-8` cuando aplica.

## Catálogo de endpoints

### App principal: ArquIA API (`back/src/main.py`)

| Método | Path                | Descripción corta                                    | Request (params/body)                                                                                                      | Response                                                                                                                      | Handler (ruta::función)            |
| ------- | ------------------- | ----------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- | ----------------------------------- |
| GET     | `/`               | Estado base y rutas utiles                            | Sin params                                                                                                                 | JSON:`{"status":"ok","docs":"/docs","health":"/health"}`                                                                    | `back/src/main.py::root`          |
| GET     | `/health`         | Health check                                          | Sin params                                                                                                                 | JSON:`{"status":"ok"}`                                                                                                      | `back/src/main.py::health`        |
| GET     | `/diagrams`       | Exporta diagrama del workflow interno del grafo       | Query:`format` (`dot` o `svg`)                                                                                       | Archivo `workflow.dot` (`text/vnd.graphviz`) o `workflow.svg` (`image/svg+xml`)                                       | `back/src/main.py::diagrams`      |
| GET     | `/diagram/export` | Exporta ultimo diagrama de arquitectura de una sesion | Query:`session_id` (req), `format` (`svg                                                                               | dot                                                                                                                           | dot_drawio                          |
| POST    | `/message`        | Endpoint conversacional principal (texto + adjuntos)  | Form:`message` (req), `session_id` (req), `image1` (opcional), `image2` (opcional). Header opcional: `X-User-Id` | JSON:`endMessage`, `diagram`, `messages`, `session_id`, `message_id`, `thread_id`, `suggestions`, `rag_trace` | `back/src/main.py::message`       |
| POST    | `/feedback`       | Guarda voto por mensaje                               | Form:`session_id`, `message_id`, `thumbs_up`, `thumbs_down`                                                        | JSON:`{"status":"Feedback recorded successfully"}`                                                                          | `back/src/main.py::feedback`      |
| POST    | `/test`           | Endpoint mock de prueba                               | Form:`message` (req), `file` (opcional)                                                                                | JSON mock con `diagram`, `endMessage`, `messages`                                                                       | `back/src/main.py::test_endpoint` |

- Codigos de respuesta principales definidos en handlers:
- `/message`: `400` (faltan campos), `500` (error en `graph.invoke`).
- `/diagrams`: `500` (`RuntimeError` de export).
- `/diagram/export`: `404` (sin diagrama en sesion), `400` (input invalido), `500` (fallo de export/render).

### App auxiliar: ChromaDB Explorer (`back/chroma_web.py`)

| Método | Path           | Descripción corta             | Request (params/body)                                                     | Response                                                                          | Handler (ruta::función)              |
| ------- | -------------- | ------------------------------ | ------------------------------------------------------------------------- | --------------------------------------------------------------------------------- | ------------------------------------- |
| GET     | `/`          | UI HTML para explorar ChromaDB | Sin params                                                                | HTML (`text/html`)                                                              | `back/chroma_web.py::home`          |
| GET     | `/api/stats` | Estadisticas de coleccion      | Sin params                                                                | JSON:`total_documents`, `collection_name`, `sources`, `persist_directory` | `back/chroma_web.py::get_stats`     |
| GET     | `/search`    | Busqueda semantica             | Query:`query` (req), `k` (default 5, `1..20`)                       | JSON con `query`, `k`, `total_results`, `results[{content, metadata}]`    | `back/chroma_web.py::search`        |
| GET     | `/documents` | Paginacion de documentos       | Query:`limit` (default 10, `1..100`), `offset` (default 0, `>=0`) | JSON con `limit`, `offset`, `documents[{content, metadata}]`                | `back/chroma_web.py::get_documents` |

## Flujos típicos de integración (frontend)

- Flujo conversacional base:
- `POST /message` (form-data con `message` + `session_id`) -> renderizar `endMessage` + `messages` + `suggestions`.
- Flujo conversacional con archivos:
- `POST /message` con `image1/image2` (imagen o PDF) -> backend agrega contexto de documento e invoca grafo con ese contexto.
- Flujo de diagramas de arquitectura:
- `POST /message` (turno que genera diagrama) -> `GET /diagram/export?session_id=...&format=svg&detail_level=overview`.
- Flujo de expansion/foco:
- `POST /message` (diagrama base) -> `GET /diagram/export?...&detail_level=detailed&focus=<overview_node_id>`.
- Flujo de feedback de respuesta:
- `POST /message` -> frontend usa `message_id` de respuesta -> `POST /feedback` con `thumbs_up/thumbs_down`.
- Flujo de observabilidad RAG (tooling):
- levantar `chroma_web.py` -> `GET /api/stats`, `GET /search`, `GET /documents` para validar indexacion/busqueda.

## Ejemplos rápidos (curl / HTTP)

```bash
# 1) Health
curl -X GET "http://localhost:8000/health"

# 2) Mensaje base (multipart/form-data)
curl -X POST "http://localhost:8000/message" \
  -H "X-User-Id: user_demo" \
  -F "message=Genera un diagrama de componentes para este ASR" \
  -F "session_id=session_demo_01"

# 3) Mensaje con PDF adjunto
curl -X POST "http://localhost:8000/message" \
  -F "message=Analiza este documento" \
  -F "session_id=session_demo_01" \
  -F "image1=@./docs/arquitectura.pdf;type=application/pdf"

# 4) Export diagrama overview en SVG
curl -X GET "http://localhost:8000/diagram/export?session_id=session_demo_01&format=svg&detail_level=overview" -o architecture_overview.svg

# 5) Export diagrama detailed en DOT
curl -X GET "http://localhost:8000/diagram/export?session_id=session_demo_01&format=dot&detail_level=detailed" -o architecture_detailed.dot

# 6) Export draw.io nativo
curl -X GET "http://localhost:8000/diagram/export?session_id=session_demo_01&format=drawio&detail_level=overview" -o architecture.drawio

# 7) Registrar feedback de un mensaje
curl -X POST "http://localhost:8000/feedback" \
  -F "session_id=session_demo_01" \
  -F "message_id=1" \
  -F "thumbs_up=1" \
  -F "thumbs_down=0"

# 8) Export workflow interno del grafo
curl -X GET "http://localhost:8000/diagrams?format=dot" -o workflow.dot

# 9) Chroma stats (app separada)
curl -X GET "http://localhost:8001/api/stats"

# 10) Chroma semantic search
curl -G "http://localhost:8001/search" --data-urlencode "query=latency tactics" --data "k=5"
```

## Archivos clave del backend (mapa rápido)

- `back/src/main.py` — API principal: lifecycle, middleware, endpoints y payloads al frontend.
- `back/src/graph/workflow.py` — compilacion del grafo LangGraph y enrutamiento entre nodos.
- `back/src/graph/resources.py` — recursos compartidos del grafo (LLM, retriever, checkpointer/tracing).
- `back/src/graph/nodes/supervisor.py` — orquestacion de intenciones y seleccion de nodos.
- `back/src/graph/nodes/diagram.py` — generacion de diagramas (DOT/IR + detalle overview/detailed).
- `back/src/services/diagram_ir.py` — IR de diagrama y transformaciones (`build_overview`, `build_expanded_view`).
- `back/src/services/diagram_render.py` — render a `svg`, `dot`, `dot_drawio`, `drawio`.
- `back/src/services/diagram_export.py` — export del workflow del grafo para documentacion.
- `back/src/services/doc_ingest.py` — extraccion de texto de PDFs adjuntos.
- `back/src/rag_agent.py` — inicializacion/carga de vectorstore y embeddings.
- `back/src/memory.py` — memoria persistente por usuario/sesion (`arch_flow`).
- `back/chroma_web.py` — app FastAPI de exploracion de ChromaDB.
- `back/build_vectorstore.py` — pipeline de indexacion de documentos a ChromaDB.
- `pyproject.toml` — dependencias, version de Python y config de tests (`back/tests`).
