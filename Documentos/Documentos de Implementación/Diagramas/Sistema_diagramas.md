# ArchIA — Sistema de Diagramas de Arquitectura

---

## Tabla de Contenidos

1. [Resumen](#1-resumen)
2. [Contexto y Objetivo](#2-contexto-y-objetivo)
3. [Alcance / Supuestos](#3-alcance--supuestos)
4. [Diseño / Arquitectura](#4-diseño--arquitectura)
   - [4.1 Niveles de detalle](#41-niveles-de-detalle)
   - [4.2 Mecanismo de consistencia Level 1 → Level 2](#42-mecanismo-de-consistencia-level-1--level-2)
   - [4.3 Gestión de sesión y contexto](#43-gestión-de-sesión-y-contexto)
5. [Flujo / Funcionamiento](#5-flujo--funcionamiento)
   - [5.1 Flujo de generación (request → archivo)](#51-flujo-de-generación-request--archivo)
   - [5.2 Detección de nivel de detalle](#52-detección-de-nivel-de-detalle)
6. [Componentes Clave](#6-componentes-clave)
7. [Cambios Realizados / Decisiones](#7-cambios-realizados--decisiones)
8. [Consideraciones y Riesgos](#8-consideraciones-y-riesgos)
9. [Cómo Usar / Operar](#9-cómo-usar--operar)
   - [9.1 Endpoints disponibles](#91-endpoints-disponibles)
   - [9.2 Ejemplos de uso con curl](#92-ejemplos-de-uso-con-curl)
   - [9.3 Patrón recomendado de integración frontend → backend](#93-patrón-recomendado-de-integración-frontend--backend)
   - [9.4 Referencia rápida: dónde tocar si quiero cambiar X](#94-referencia-rápida-dónde-tocar-si-quiero-cambiar-x)
10. [Notas / Inconsistencias](#10-notas--inconsistencias)
11. [Referencias Internas](#11-referencias-internas)

---

## 1. Resumen

- **Problema resuelto:** Los diagramas de arquitectura generados por LLM eran demasiado densos (30+ nodos) y difíciles de leer. Se implementó un sistema de **niveles de detalle** con simplificación automática.
- **Cambio principal:** Representación intermedia (IR) separada del rendering, que permite colapsar/expandir nodos de forma determinística y exportar a múltiples formatos (SVG, DOT, draw.io).
- **Nivel 1 (overview):** Por defecto, diagramas simplificados de 5–15 nodos máximo. Los grupos/clusters se colapsan en un solo nodo.
- **Nivel 2 (detailed):** Bajo demanda del usuario ("detallado", "full", "más detalle"), se genera el diagrama completo con todos los componentes.
- **Expansión focalizada:** Un nodo overview específico (e.g., `grp_backend`) puede expandirse para ver sus componentes internos sin regenerar todo.
- **Consistencia garantizada:** Un mapping `overview_node_id -> [detailed_node_ids]` asegura que el diagrama detallado sea una expansión coherente del overview, no un diagrama diferente.
- **Múltiples formatos:** SVG (render), DOT (exportable), DOT plano (para draw.io), XML nativo de draw.io.
- **Contexto por sesión:** El estado del diagrama se mantiene vía `session_id` persistido en SQLite; el frontend no necesita reenviar el prompt completo para expandir.

---

## 2. Contexto y Objetivo

**Problema:**
Los diagramas generados directamente por el LLM superaban con frecuencia los 30 nodos, resultando inlegibles en presentaciones y revisiones de arquitectura. No existía una manera controlada de navegar entre una vista general y una vista detallada sin regenerar desde cero.

**Objetivos:**

- Ofrecer una vista de alto nivel por defecto (overview, 5–15 nodos) sin intervención del usuario.
- Permitir una vista completa bajo demanda (detailed) mediante keywords conversacionales o parámetro explícito en el endpoint de exportación.
- Soportar expansión focalizada de un componente específico del overview.
- Garantizar que Level 2 sea siempre una expansión coherente de Level 1, no un diagrama independiente.
- Exportar a múltiples formatos (SVG, DOT, draw.io) desde el mismo estado de sesión.
- Mantener contexto entre mensajes para que el frontend solo necesite conservar `session_id`.

---

## 3. Alcance / Supuestos

- El sistema actúa exclusivamente sobre el backend (`back/`).
- La persistencia de sesión se maneja vía SQLite (`state_db/memory.db`).
- Los identificadores de sesión son únicamente `session_id`. No existen `conversation_id`, `diagram_id` ni `request_id` como identificadores de API.
- Los formatos de exportación soportados son: `svg`, `dot`, `dot_drawio`, `drawio`.
- `detail_level` acepta únicamente los valores `overview` y `detailed`.
- El endpoint `GET /diagrams` es para workflow interno del agente, no para diagramas del usuario.

---

## 4. Diseño / Arquitectura

### 4.1 Niveles de detalle

| Nivel                          | Intención                                  | Qué incluye                                                                | Qué colapsa/agrupa                                                                             | Cómo se solicita                                                                          |
| ------------------------------ | ------------------------------------------- | --------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| **overview** (default)   | Vista de alto nivel para presentaciones     | 5–15 nodos máximo: subsistemas, capas, servicios mayores                  | Grupos/clusters → 1 nodo con `kind=CLUSTER`. Edges internos eliminados, paralelos agregados. | Default, o endpoint con `detail_level=overview`                                          |
| **detailed**             | Diagrama completo con todos los componentes | Todos los nodos y edges que el LLM generó                                  | Nada (sin colapso)                                                                              | Usuario dice "detallado", "full", "más detalle", o endpoint con `detail_level=detailed` |
| **focused** (expansión) | Expandir un grupo específico del overview  | Nodos detallados del grupo + nodos externos conectados (marcados `[ext]`) | Solo muestra el subgrafo del grupo solicitado                                                   | Endpoint con `detail_level=detailed&focus=<overview_node_id>`                            |

**Implementación:**

- Enum `DetailLevel` en `back/src/services/diagram_ir.py`: `OVERVIEW`, `DETAILED`
- Función de colapso: `build_overview()`
- Función de expansión: `build_expanded_view()`

### 4.2 Mecanismo de consistencia Level 1 → Level 2

El Level 2 (detailed) refleja el Level 1 (overview) porque ambos comparten el mismo modelo subyacente, no son generaciones independientes del LLM.

1. **IDs estables y determinísticos:**

   - Nodos overview: `grp_<cluster_id>` (basado en el `group_id` del cluster original).
   - Nodos sin grupo: mantienen su `id` original.
   - Implementado en: `diagram_ir.py::build_overview()` líneas 390–400.
2. **Mapping bidireccional:**

   - `build_overview()` retorna `(overview_model, mapping)` donde `mapping: Dict[str, List[str]]`.
   - Ejemplo: `"grp_backend" -> ["auth_svc", "order_svc", "payment_svc"]`.
   - El mapping se guarda en `state["diagram"]["overview_mapping"]`.
   - Implementado en: `diagram_ir.py::build_overview()` líneas 360–480.
3. **Reglas determinísticas de agrupación:**

   - Si el modelo tiene `groups` (clusters) → cada grupo produce 1 nodo overview.
   - Nodos sin grupo se mantienen individuales.
   - Si no hay grupos → agrupa por `NodeKind` (database, service, etc.).
   - Implementado en: `diagram_ir.py::build_overview()` líneas 390–430.
4. **Agregación reproducible de edges:**

   - Edges entre nodos del mismo grupo overview → eliminados (internos).
   - Edges entre grupos overview → agregados; múltiples edges concatenan labels (`"HTTP | gRPC"`).
   - Implementado en: `diagram_ir.py::build_overview()` líneas 470–480.
5. **Expansión focalizada coherente:**

   - `build_expanded_view(detailed_model, mapping, focus_node_id)` busca en el mapping original.
   - Extrae todos los nodos detallados del grupo solicitado.
   - Incluye edges hacia/desde nodos externos (mostrados como `[ext] <label>`).
   - Implementado en: `diagram_ir.py::build_expanded_view()` líneas 515–579.
6. **Ordenamiento determinístico:**

   - `DiagramModel.sort_deterministic()` ordena nodos/edges alfabéticamente.
   - Garantiza que dos runs con el mismo input produzcan el mismo output.
   - Implementado en: `diagram_ir.py::DiagramModel.sort_deterministic()` líneas 148–152.

### 4.3 Gestión de sesión y contexto

- **`session_id`:** identificador obligatorio en `POST /message` (`Form(...)`) y en `GET /diagram/export` (`Query(...)`).
- **`thread_id`:** se fija como `thread_id = session_id` y se pasa al grafo en `config = {"configurable": {"thread_id": thread_id}}`.
- **Estado persistente:** `load_arch_flow()` / `save_arch_flow()` guardan el flujo en SQLite (tabla `memory`, archivo `state_db/memory.db`).
- **Diagrama previo guardado:** al terminar `POST /message`, se persiste `arch_flow["last_diagram"]` con `dot`, `dot_raw`, `dot_drawio`, `detail_level`, `overview_mapping`.
- **Checkpointer del grafo:** `graph = builder.compile(checkpointer=sqlite_saver)` con `sqlite_saver = MemorySaver()` en `back/src/graph/workflow.py` y `back/src/graph/resources.py`.

---

## 5. Flujo / Funcionamiento

### 5.1 Flujo de generación (request → archivo)

**Paso 1 — Request del usuario**

- `POST /message` con intención de diagrama detectada por keywords: `diagrama`, `diagram`, `componentes`.
- Archivo: `back/src/main.py::message()`

**Paso 2 — Nodo orquestador**

- `diagram_orchestrator_node()` determina el nivel de detalle (ver [5.2](#52-detección-de-nivel-de-detalle)).
- Archivo: `back/src/graph/nodes/diagram.py::diagram_orchestrator_node()`

**Paso 3 — Generación LLM**

- Invoca LLM con prompt diferenciado según nivel:
  - Overview: `DOT_SYSTEM_OVERVIEW` (instrucción: max 15 nodos, agrupar subsistemas).
  - Detailed: `DOT_SYSTEM` (instrucción: diagrama completo).
- Archivo prompts: `back/src/graph/consts.py`
- Función: `back/src/graph/nodes/diagram.py::_llm_nl_to_dot()`

**Paso 4 — Parseo a IR**

- DOT string → `DiagramModel` (nodos, edges, grupos).
- Función: `back/src/services/diagram_ir.py::parse_dot_to_model()`

**Paso 5 — Simplificación automática**

- Si el nivel es overview y el LLM generó >20 nodos, se colapsa programáticamente:
  - Grupos → 1 nodo overview cada uno.
  - Edges internos eliminados, externos agregados.
  - Retorna `(overview_model, mapping)` donde `mapping: Dict[str, List[str]]`.
- Función: `back/src/services/diagram_ir.py::build_overview()`

**Paso 6 — Rendering**

- IR → DOT final + SVG base64.
  - DOT: `back/src/services/diagram_render.py::render_dot()`
  - SVG: `back/src/services/diagram_render.py::render_svg_b64()`

**Paso 7 — Almacenamiento**

- `state["diagram"]` contiene: `svg_b64`, `dot`, `dot_raw`, `dot_drawio`, `detail_level`, `node_count`, `edge_count`, `overview_mapping`.
- Se guarda en sesión persistente (`state_db/`).

**Paso 8 — Exportación**

- `GET /diagram/export` lee `last_diagram` de memoria, re-aplica nivel, renderiza y devuelve archivo.
- Archivo: `back/src/main.py::diagram_export()`

### 5.2 Detección de nivel de detalle

Hay dos caminos para indicar el nivel deseado:

**Vía mensaje conversacional (`POST /message`):**

- Default: `detail_level="overview"`.
- Cambia a `"detailed"` si el texto contiene alguno de estos keywords: `detailed`, `detallado`, `full`, `more detail`, `más detalle`, `expand`, `expandir`, `ampliar`.
- Archivo: `back/src/graph/nodes/diagram.py`, array `detail_keywords`, líneas 90–100.

**Vía parámetro explícito en export (`GET /diagram/export`):**

- Parámetro `detail_level` acepta: `overview` | `detailed`.
- Parámetro opcional `focus` para expansión focalizada: acepta un `overview_node_id` (e.g., `grp_backend`).

---

## 6. Componentes Clave

| Archivo                                 | Responsabilidad                                                                                                                                                                                                                                                                                                                                |
| --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `back/src/services/diagram_ir.py`     | Representación intermedia (IR):`DiagramModel`, parser DOT→IR (`parse_dot_to_model()`), colapso (`build_overview()`), expansión (`build_expanded_view()`), ordenamiento (`sort_deterministic()`).                                                                                                                                  |
| `back/src/services/diagram_render.py` | Rendering IR→formatos:`render_dot()` (DOT con clusters), `render_svg()` (SVG vía Graphviz), `render_dot_drawio()` (DOT plano sin clusters/compound edges), `render_drawio()` (XML nativo draw.io). Contiene constantes de estilo: `_KIND_STYLES`, `_EDGE_STYLES`, `_DOT_GRAPH_ATTRS`, `_DOT_NODE_DEFAULTS` (líneas 40–80). |
| `back/src/graph/nodes/diagram.py`     | Nodo LangGraph:`diagram_orchestrator_node()` decide nivel, llama LLM via `_llm_nl_to_dot()`, parsea, colapsa, renderiza, almacena en state.                                                                                                                                                                                                |
| `back/src/graph/consts.py`            | Prompts del sistema:`DOT_SYSTEM` (detailed), `DOT_SYSTEM_OVERVIEW` (max 15 nodos).                                                                                                                                                                                                                                                         |
| `back/src/main.py`                    | Endpoints REST:`POST /message` y `GET /diagram/export` (línea 287).                                                                                                                                                                                                                                                                       |
| `back/src/graph/state.py`             | Schema del estado: campo `diagram` almacena metadata + `overview_mapping`.                                                                                                                                                                                                                                                                 |
| `back/src/memory.py`                  | Persistencia de sesión:`save_arch_flow()`, `load_arch_flow()` sobre SQLite (`state_db/memory.db`, tabla `memory`).                                                                                                                                                                                                                    |
| `back/src/graph/workflow.py`          | Compilación del grafo con checkpointer:`graph = builder.compile(checkpointer=sqlite_saver)`.                                                                                                                                                                                                                                                |
| `back/src/graph/resources.py`         | Inicialización de `sqlite_saver = MemorySaver()`.                                                                                                                                                                                                                                                                                           |
| `back/tests/test_diagram_pipeline.py` | Tests unitarios: parsing, overview, expansion, rendering, dot_drawio, drawio XML.                                                                                                                                                                                                                                                              |

---

## 7. Cambios Realizados / Decisiones

- **Introducción de representación intermedia (IR):** se creó `DiagramModel` como capa separada entre el DOT generado por el LLM y el rendering final. Permite operar sobre el grafo (colapsar, expandir, filtrar) de forma independiente al formato de salida.
- **Simplificación automática programática:** si el LLM genera un overview con >20 nodos, el sistema colapsa grupos en código sin repromptear al LLM, garantizando determinismo y ahorro de tokens.
- **Mapping overview ↔ detailed persistido en sesión:** se guarda en `state["diagram"]["overview_mapping"]` para que la expansión de Level 2 sea siempre coherente con el Level 1 mostrado al usuario.
- **Múltiples formatos de exportación desde un único estado:** SVG, DOT estándar, DOT plano para draw.io y XML nativo draw.io se generan todos desde el mismo `last_diagram` de sesión, sin regenerar con el LLM.
- **`session_id` como identificador único:** se decidió no usar `diagram_id`, `conversation_id` ni `request_id`; toda la trazabilidad pasa por `session_id`.

---

## 8. Consideraciones y Riesgos

- **Detección de nivel por keywords:** la detección en `POST /message` es sensible a los términos exactos del array `detail_keywords`. Frases como "nivel 2" o "dame más" no están contempladas en la versión documentada.
- **Dependencia de `session_id` estable:** la consistencia Level 1 → Level 2 depende de que el frontend conserve el mismo `session_id` entre requests. Un cambio de sesión implica pérdida del `overview_mapping` y del diagrama previo.
- **Umbral de colapso automático (>20 nodos):** si el LLM genera entre 15 y 20 nodos en modo overview, no se activa el colapso programático, pero el resultado puede superar el objetivo de 5–15 nodos. (No especificado el comportamiento exacto en este rango en los documentos.)

---

## 9. Cómo Usar / Operar

### 9.1 Endpoints disponibles

#### `POST /message`

- **Body** `multipart/form-data`:
  - `message: str` — requerido
  - `session_id: str` — requerido
  - `image1: UploadFile` — opcional
  - `image2: UploadFile` — opcional
- **Response** incluye: `diagram`, `session_id`, `message_id`, `thread_id`, `endMessage`.

#### `GET /diagram/export`

- **Query params:**
  - `session_id` — requerido
  - `format`: `svg` | `dot` | `dot_drawio` | `drawio`
  - `detail_level`: `overview` | `detailed` (default: `overview`)
  - `focus` — opcional; ID del nodo overview a expandir (e.g., `grp_backend`)
- **Content-Type de respuesta:**
  - `svg` → `image/svg+xml`
  - `dot`, `dot_drawio` → `text/vnd.graphviz`
  - `drawio` → `application/xml`

### 9.2 Ejemplos de uso con curl

```bash
# 1. Generar diagrama vía conversación (overview por default)
curl -X POST "http://localhost:8000/message" \
  -F "session_id=sess-123" \
  -F "message=Genera un diagrama de componentes para el ASR de latencia"
# La respuesta incluye diagram.svg_b64

# 2. Exportar overview en SVG
curl "http://localhost:8000/diagram/export?session_id=sess-123&format=svg" \
  -o arch_overview.svg

# 3. Exportar detailed en DOT
curl "http://localhost:8000/diagram/export?session_id=sess-123&format=dot&detail_level=detailed" \
  -o arch_detailed.dot

# 4. Exportar DOT compatible con draw.io (sin clusters, sin compound edges, sin ports)
curl "http://localhost:8000/diagram/export?session_id=sess-123&format=dot_drawio" \
  -o arch_drawio.dot
# Importar en draw.io: Extras → Edit Diagram → pegar DOT

# 5. Exportar nativo draw.io (XML mxGraph con posiciones de Graphviz)
curl "http://localhost:8000/diagram/export?session_id=sess-123&format=drawio" \
  -o architecture.drawio

# 6. Expandir un nodo específico (focused)
curl "http://localhost:8000/diagram/export?session_id=sess-123&format=svg&detail_level=detailed&focus=grp_backend" \
  -o backend_expanded.svg
# Muestra solo los nodos internos de grp_backend + sus conexiones externas
```

### 9.3 Referencia rápida: dónde tocar si quiero cambiar X

| Qué cambiar                                                | Dónde tocar                                                                                                                          |
| ----------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Reglas de colapso/agrupación en overview                   | `diagram_ir.py::build_overview()` líneas 360–510. Heurística de clustering por `group_id`, `NodeKind`, umbral `max_nodes`. |
| Mapeo de expansión Level 1 → Level 2                      | `diagram_ir.py::build_expanded_view()` líneas 515–579. Criterios de inclusión de nodos externos.                                 |
| Estilos/layout del diagrama (colores, shapes, fuentes)      | `diagram_render.py` líneas 40–80: constantes `_KIND_STYLES`, `_EDGE_STYLES`, `_DOT_GRAPH_ATTRS`, `_DOT_NODE_DEFAULTS`.    |
| Prompts del LLM                                             | `consts.py`: `DOT_SYSTEM` (detailed), `DOT_SYSTEM_OVERVIEW` (overview).                                                         |
| Keywords de detección de nivel de detalle                  | `diagram.py::diagram_orchestrator_node()` líneas 90–100: array `detail_keywords`.                                               |
| Endpoint / formatos de exportación (añadir nuevo formato) | `main.py::diagram_export()` línea 287+: nueva rama en el `if format == ...`.                                                     |
| Parser DOT → IR                                            | `diagram_ir.py::parse_dot_to_model()` líneas 222–350. Regex para clusters, nodes, edges.                                          |
| Rendering IR → DOT                                         | `diagram_render.py::render_dot()` líneas 111–188.                                                                                 |
| Rendering nativo draw.io (layout, estilos XML)              | `diagram_render.py::render_drawio()` líneas 501–636.                                                                              |

---

## 10. Referencias Internas

| Archivo                                 | Propósito en este documento                          |
| --------------------------------------- | ----------------------------------------------------- |
| `back/src/services/diagram_ir.py`     | IR, colapso, expansión, determinismo                 |
| `back/src/services/diagram_render.py` | Rendering a todos los formatos de salida              |
| `back/src/graph/nodes/diagram.py`     | Nodo orquestador, detección de nivel, llamada al LLM |
| `back/src/graph/consts.py`            | Prompts `DOT_SYSTEM` y `DOT_SYSTEM_OVERVIEW`      |
| `back/src/main.py`                    | Endpoints `POST /message` y `GET /diagram/export` |
| `back/src/graph/state.py`             | Schema de estado con campo `diagram`                |
| `back/src/memory.py`                  | Persistencia de sesión en SQLite                     |
| `back/src/graph/workflow.py`          | Compilación del grafo con checkpointer               |
| `back/src/graph/resources.py`         | Inicialización de `MemorySaver`                    |
| `back/tests/test_diagram_pipeline.py` | Tests unitarios del pipeline completo                 |
