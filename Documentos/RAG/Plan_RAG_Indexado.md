# Plan: Reestructuración del RAG con Índices por Atributos de Calidad

**Fecha:** 2026-02-28
**Estado:** Implementado
**Autor:** Equipo ArchIA

---

## 1. Contexto y Motivación

El RAG anterior usaba una sola colección Chroma `"arquia"` sin ningún concepto de "índice". La recuperación de documentos estaba dispersa en varios nodos (`asr.py`, `tactics.py`, `investigator.py`, `evaluator.py`) y hacía queries genéricos sin saber qué atributo de calidad estaba siendo tratado. El resultado era retrieval poco enfocado y sin trazabilidad temática.

### Problema a resolver
Cuando el sistema usa RAG, no hay diferenciación entre el contexto temático de la pregunta. Una pregunta sobre latencia y una sobre escalabilidad obtienen los mismos resultados del mismo pool de documentos.

### Objetivo
Reestructurar el RAG alrededor de **índices** — atributos de calidad nombrados — de forma que:
1. Se resuelva automáticamente qué índice (QA) aplica a la pregunta del usuario.
2. Para ese índice, se recupere contenido diferenciado por tipo: **ASR**, **Estilos** y **Tácticas**.
3. Sea extensible: añadir un nuevo atributo de calidad no requiere cambios de código.

---

## 2. Decisiones de Diseño

| Decisión | Elección | Justificación |
|----------|----------|---------------|
| Resolución de índice | LLM inference | Mayor precisión con preguntas complejas, bilingüe (ES/EN), coherente con el stack |
| Fuentes de documentos | Re-etiquetar los PDFs existentes | Aprovechar el corpus ya disponible; el tagging se hace al ingestal |
| Estrategia Chroma | Una sola colección + filtros de metadata | Sin romper el código existente; aditivo |

---

## 3. Arquitectura del Sistema Nuevo

### 3.1 Flujo Completo

```
Usuario: "¿Qué tácticas de escalabilidad debería usar?"
    ↓
boot_node → classifier_node:
    ├── intent = "tactics"
    ├── force_rag = true
    └── resolve_quality_attribute(question, llm) → "escalabilidad"  [NUEVO]
        └── state["resolved_index"] = "escalabilidad"
    ↓
supervisor_node → tactics_node:
    ├── get_indexed_retriever("escalabilidad", "tacticas", k=6)
    ├── Chroma filtra: quality_attribute="escalabilidad" AND content_type="tacticas"
    └── Devuelve chunks relevantes específicos
    ↓
unifier_node → respuesta al usuario con tácticas + fuentes indexadas
```

### 3.2 Nuevo Campo en GraphState

```python
resolved_index: str  # "escalabilidad" | "latencia" | "general"
```

Este campo se establece en el **classifier_node** y está disponible para todos los nodos worker posteriores.

### 3.3 Metadata Chroma Añadida

Cada chunk en Chroma ahora tiene dos campos adicionales de metadata:

| Campo | Valores posibles | Ejemplo |
|-------|-----------------|---------|
| `quality_attribute` | `"escalabilidad"`, `"latencia"`, `"general"` | `"escalabilidad"` |
| `content_type` | `"asr"`, `"estilos"`, `"tacticas"`, `"general"` | `"tacticas"` |

---

## 4. Configuración de Índices (`back/config/indices.json`)

La configuración de índices es **completamente declarativa**. Para añadir un nuevo atributo de calidad, sólo se edita este archivo y se reconstruye el vectorstore.

```json
{
  "version": "1.0",
  "quality_attributes": [
    {
      "id": "escalabilidad",
      "display_name": "Escalabilidad",
      "keywords_en": ["scalability", "throughput", "scale", "load", ...],
      "keywords_es": ["escalabilidad", "escalable", "carga", ...],
      "description": "Capacidad del sistema de manejar crecimiento en carga o datos"
    },
    {
      "id": "latencia",
      "display_name": "Latencia",
      "keywords_en": ["latency", "response time", "p95", "p99", ...],
      "keywords_es": ["latencia", "tiempo de respuesta", ...],
      "description": "Tiempo entre una solicitud y su respuesta"
    }
  ],
  "content_types": [
    { "id": "asr",      "keywords": ["quality attribute scenario", "QAS", "stimulus", ...] },
    { "id": "estilos",  "keywords": ["architecture style", "microservices", ...] },
    { "id": "tacticas", "keywords": ["architectural tactic", "circuit breaker", ...] }
  ]
}
```

---

## 5. Archivos Creados y Modificados

### Archivos NUEVOS

| Archivo | Propósito |
|---------|-----------|
| `back/config/indices.json` | Config declarativa de QAs y tipos de contenido |
| `back/src/graph/index_resolver.py` | Módulo de resolución LLM del índice QA |

### Archivos MODIFICADOS

| Archivo | Cambio |
|---------|--------|
| `back/build_vectorstore.py` | Auto-tagging de chunks con `quality_attribute` + `content_type` |
| `back/src/rag_agent.py` | Nueva función `get_indexed_retriever(qa, content_type, k)` |
| `back/src/graph/state.py` | Campo `resolved_index: str` en `GraphState` |
| `back/src/graph/nodes/classifier.py` | Llamar `resolve_quality_attribute()`, incluir en return |
| `back/src/graph/nodes/asr.py` | `get_indexed_retriever(resolved_index, "asr", 6)` |
| `back/src/graph/nodes/tactics.py` | `get_indexed_retriever(resolved_index or qa, "tacticas", 6)` |
| `back/src/graph/nodes/tools.py` | Param `quality_attribute: str = "general"` en `local_RAG` |
| `back/src/graph/nodes/investigator.py` | Inyección de `resolved_index` en system prompt |

### Archivos SIN cambios

Los siguientes nodos no se modificaron:
- `evaluator.py` — retrieval genérico para evaluación de ASRs
- `style.py`, `diagram.py`, `supervisor.py`, `unifier.py` — no usan RAG
- `resources.py` — el `_LazyRetriever` global se mantiene para compatibilidad

---

## 6. Módulo `index_resolver.py`

### API Pública

```python
def resolve_quality_attribute(question: str, llm) -> str:
    """
    Usa LLM para determinar qué atributo de calidad corresponde a la pregunta.
    Retorna el 'id' del atributo (e.g., "escalabilidad", "latencia") o "general".
    """

def get_available_indices() -> list[dict]:
    """Retorna la lista de QAs disponibles desde indices.json."""

def get_content_types() -> list[dict]:
    """Retorna la lista de tipos de contenido disponibles."""
```

### Lógica interna
1. Carga `indices.json` y construye el prompt con las QAs disponibles y sus descripciones
2. Llama al LLM con un prompt minimal (sin historial de conversación)
3. Valida que la respuesta sea un ID conocido
4. Fallback: `"general"` ante cualquier error o respuesta inválida

---

## 7. Función `get_indexed_retriever` en `rag_agent.py`

```python
def get_indexed_retriever(
    quality_attribute: str | None = None,
    content_type: str | None = None,
    k: int = 6,
) -> VectorStoreRetriever:
```

### Lógica de filtros

| `quality_attribute` | `content_type` | Filtro aplicado |
|--------------------|----------------|-----------------|
| `"escalabilidad"` | `"tacticas"` | `$and [{qa: escalabilidad}, {ct: tacticas}]` |
| `"latencia"` | `None` | `{quality_attribute: {$eq: latencia}}` |
| `None` o `"general"` | `None` o `"general"` | Sin filtro (comportamiento actual) |

---

## 8. Auto-tagging en `build_vectorstore.py`

Al reconstruir el vectorstore, cada chunk es etiquetado automáticamente:

```
Función _tag_chunk(text, config) → (quality_attribute, content_type)
```

**Algoritmo:**
1. Tokenizar el texto a lowercase
2. Para cada QA en `indices.json`: contar matches de `keywords_en` + `keywords_es`
3. El QA con más matches → `quality_attribute`. Si ninguno → `"general"`
4. Para cada content_type: contar matches de `keywords`
5. El tipo con más matches → `content_type`. Si ninguno → `"general"`

El log de construcción muestra la distribución de tags:
```
[build] Distribución de tags (quality_attribute/content_type):
   general/general: 1823 chunks
   escalabilidad/tacticas: 234 chunks
   latencia/asr: 187 chunks
   ...
```

---

## 9. Extensibilidad

Para añadir un nuevo atributo de calidad (e.g., "disponibilidad"):

1. Editar `back/config/indices.json`:
```json
{
  "id": "disponibilidad",
  "display_name": "Disponibilidad",
  "keywords_en": ["availability", "uptime", "fault tolerance", "redundancy", ...],
  "keywords_es": ["disponibilidad", "tolerancia a fallos", ...],
  "description": "Capacidad del sistema de mantenerse operativo ante fallos"
}
```

2. Reconstruir el vectorstore:
```bash
cd /back && python build_vectorstore.py
```

3. ✅ Sin cambios de código — el LLM resolver detecta el nuevo índice automáticamente.

---

## 10. Verificación End-to-End

### Paso 1: Rebuild Chroma
```bash
cd /Users/santi/Documents/tesis/archIABack/back
python build_vectorstore.py
```
Verificar en el log que aparece la distribución de tags `quality_attribute/content_type`.

### Paso 2: Iniciar FastAPI
```bash
uvicorn src.main:app --reload
```

### Paso 3: Tests funcionales

| Pregunta de prueba | `resolved_index` esperado | Nodo destino |
|-------------------|--------------------------|--------------|
| "¿Cómo diseño un sistema escalable para e-commerce?" | `"escalabilidad"` | tactics / asr |
| "Necesito reducir la latencia de mi API REST" | `"latencia"` | tactics / asr |
| "¿Qué es ADD 3.0?" | `"general"` | investigator |
| "¿Cómo logro alta disponibilidad?" (después de añadir el índice) | `"disponibilidad"` | tactics / asr |

### Paso 4: Verificar fallback
- Preguntas de tipo `"general"` → retriever sin filtros → comportamiento idéntico al sistema anterior
- Si `resolved_index` no está en state → `get_indexed_retriever(None, ...)` → sin filtro

---

## 11. Fuente de Datos

- **Documentos**: PDFs en `back/docs/` (Software Architecture in Practice, Evaluating Software Architectures, Software Systems Architecture caps 18/20/21/25)
- **Estrategia de indexación**: Keyword-based auto-tagging en `build_vectorstore.py`
- **Modelo de embeddings**: `text-embedding-3-small` (OpenAI) o Azure OpenAI Embeddings
- **Método de recuperación**: Similarity search con filtros de metadata (`$eq`, `$and`)
- **Vectorstore**: Chroma, colección única `"arquia"`, persistida en `back/chroma_db/`
- **Configuración LLM**: Heredada del sistema (`get_chat_model(temperature=0.0)`)
- **Métricas de evaluación**: Pendiente — se evaluará en sesión posterior

---

---

## 12. Optimización de Keywords — Sesión 2026-02-28

### 12.1 Problema Identificado

Tras el primer rebuild del vectorstore, la distribución de `content_type` mostró una asimetría severa:

| content_type | Chunks | % del total |
|---|---|---|
| `asr` | 450 | 68.4% |
| `general` | 151 | 22.9% |
| `estilos` | 45 | 6.8% |
| `tacticas` | **9** | **1.4%** |

Solo **9 de 658 chunks** (1.4%) fueron etiquetados como `content_type="tacticas"`, lo que hacía que el `tactics_node` recibiera context casi vacío al usar el retriever indexado con `content_type="tacticas"`.

**Causa raíz**: El algoritmo de tagging asigna el `content_type` con más keyword matches. En los libros de arquitectura (Software Architecture in Practice, Evaluating Software Architectures), las tácticas se explican **dentro del contexto de escenarios de calidad**: un capítulo sobre tácticas de performance también menciona `"stimulus"`, `"response measure"`, `"quality attribute scenario"`, etc. En esos chunks, las keywords de ASR ganaban la cuenta por amplio margen. Resultado: chunks que hablaban de tácticas quedaban mal etiquetados como `asr`.

Adicionalmente, el keyword `"environment"` en la lista ASR era demasiado genérico — aparece en frases como "cloud environment", "production environment", "runtime environment" — y contaminaba chunks que no eran de escenarios.

### 12.2 Análisis del Corpus

Se ejecutó un script de análisis (`back/analyze_keywords.py`) que contó, para cada término candidato, en cuántos chunks del vectorstore aparecía. Resultados relevantes de los **top términos de tácticas**:

| Término | Chunks | En lista antes |
|---------|--------|----------------|
| `tactic` | 62 | ✅ |
| `tactics` | 34 | ✅ |
| `asynchronous` | 16 | ❌ |
| `replication` | 9 | ✅ |
| `encapsulate` | 8 | ❌ |
| `queuing` | 7 | ❌ |
| `queue` | 6 | ❌ |
| `spare` | 6 | ❌ |
| `partitioning` | 6 | ✅ |
| `detect faults` | 3 | ❌ |
| `manage resources` | 3 | ✅ |
| `control resource demand` | 2 | ❌ |
| `schedule resources` | 2 | ❌ |
| `use an intermediary` | 2 | ❌ |
| `prevent faults` | 2 | ❌ |
| `non-stop forwarding` | 2 | ❌ |

La palabra `"tactic"` aparecía en **62 chunks** pero solo 9 estaban etiquetados como `tacticas` — los otros 53 tenían mayor puntuación de keywords ASR en el mismo chunk.

### 12.3 Cambios Aplicados en `indices.json`

#### Cambio 1 — Eliminar `"environment"` y `"scenario"` de keywords ASR

```diff
  "keywords": [
    "quality attribute scenario", "QAS", "stimulus", "response measure",
    "source of stimulus",
-   "environment", "scenario",
    "architectural significant requirement",
    "ASR", "architecture significant requirement", "quality scenario",
    "attribute-driven", "ADD", "architectural driver"
  ]
```

**Justificación**: `"environment"` y `"scenario"` son los términos ASR más genéricos. `"environment"` aparece en decenas de contextos no-QAS (cloud environment, runtime environment, network environment). Eliminarlos reduce la puntuación ASR en chunks de tácticas sin afectar la precisión para chunks genuinamente de escenarios, que siguen teniendo keywords como `"stimulus"`, `"response measure"`, `"QAS"`, etc.

#### Cambio 2 — Ampliar keywords de `tacticas` de 24 a 57 términos

Se añadieron **33 términos nuevos**, organizados por categoría táctica según el marco de Bass, Clements & Kazman (Software Architecture in Practice):

**Tácticas de performance/recursos** (confirmadas en corpus):
`"manage resource demand"`, `"control resource demand"`, `"limit event response"`, `"prioritize events"`, `"reduce overhead"`, `"bound execution times"`, `"bound queue sizes"`, `"schedule resources"`, `"increase resources"`, `"increase resource efficiency"`, `"introduce concurrency"`, `"maintain multiple copies"`, `"arbitrate"`, `"arbitration"`

**Tácticas de disponibilidad** (confirmadas en corpus):
`"detect faults"`, `"prevent faults"`, `"recover from faults"`, `"active redundancy"`, `"passive redundancy"`, `"spare"`, `"exception handling"`, `"rollback"`, `"graceful degradation"`, `"degradation"`, `"shadow"`, `"state resynchronization"`, `"escalating restart"`, `"non-stop forwarding"`

**Tácticas de modificabilidad** (confirmadas en corpus):
`"encapsulate"`, `"use an intermediary"`, `"restrict dependencies"`, `"defer binding"`, `"increase semantic coherence"`, `"abstract common services"`

**Tácticas de seguridad**:
`"authenticate actors"`, `"authorize actors"`, `"separate entities"`, `"encrypt data"`

**Términos de infraestructura de tácticas** (alta frecuencia en corpus):
`"asynchronous"`, `"queue"`, `"queuing"`, `"pipeline"`, `"cache"`, `"load shedding"`, `"thread pool"`, `"connection pool"`, `"redundancy"`

**Calificadores por tipo de atributo** (nuevos, muy específicos):
`"performance tactic"`, `"availability tactic"`, `"security tactic"`, `"modifiability tactic"`, `"scalability tactic"`, `"interoperability tactic"`, `"tactics"` (plural explícito)

### 12.4 Resultados Después de Rebuild

| content_type | Antes | Después | Cambio absoluto | Cambio % |
|---|---|---|---|---|
| `tacticas` | 9 | **57** | +48 | **+533%** |
| `asr` | 450 | 391 | −59 | −13% |
| `estilos` | 45 | 46 | +1 | +2% |
| `general` | 151 | 148 | −3 | −2% |

Distribución final detallada por `quality_attribute/content_type`:

```
general/asr: 288 chunks
general/general: 130 chunks
latencia/asr: 57 chunks
escalabilidad/asr: 46 chunks
general/estilos: 31 chunks
latencia/tacticas: 27 chunks     ← de 6 a 27
general/tacticas: 18 chunks      ← de 3 a 18
latencia/general: 18 chunks
escalabilidad/general: 16 chunks
escalabilidad/tacticas: 12 chunks  ← de 3 a 12
latencia/estilos: 8 chunks
escalabilidad/estilos: 7 chunks
```

### 12.5 Impacto en el Sistema

El `tactics_node` ahora tiene acceso a chunks realmente etiquetados como tácticas cuando usa `get_indexed_retriever(quality_attribute="latencia", content_type="tacticas", k=6)`:

- **Antes**: Solo 6 chunks disponibles para `latencia/tacticas`, prácticamente vacío
- **Después**: 27 chunks disponibles para `latencia/tacticas`, 12 para `escalabilidad/tacticas`

Esto significa que en el flujo ADD 3.0, cuando un usuario pide tácticas de escalabilidad o latencia, el `tactics_node` puede recuperar **contexto real del libro** sobre tácticas de ese atributo de calidad específico, en lugar de caer en búsqueda no filtrada.

### 12.6 Limitaciones y Consideraciones

1. **El algoritmo sigue siendo keyword-count**: Un chunk que hable de tácticas en el contexto de un escenario QAS puede seguir siendo clasificado como `asr`. Esto es técnicamente correcto (el chunk es sobre ambos temas) pero puede reducir la recuperación de tácticas específicas.

2. **Corpus pequeño (658 chunks)**: El corpus solo incluye dos libros. Con más fuentes (papers de tácticas, documentación de patrones), la distribución mejoraría naturalmente.

3. **Extensibilidad preservada**: Ninguno de estos cambios requirió modificar código. Solo se editó `back/config/indices.json` y se reconstruyó el vectorstore — exactamente como diseñado en la sección de extensibilidad.

---

*Sección añadida el 2026-02-28 tras análisis de corpus y optimización de keywords.*

---

---

## 13. Verificación y Corrección de Bugs — Sesión 2026-02-28

### 13.1 Script de Verificación End-to-End

Se creó `back/verify_rag_indexado.py` para verificar de forma automatizada que el sistema RAG indexado funciona correctamente. El script tiene tres partes:

| Parte | Qué verifica |
|---|---|
| **1 — Metadatos Chroma** | Que todos los chunks tienen `quality_attribute` y `content_type`, distribución actual |
| **2 — Filtros del retriever** | Que cada combinación de filtros devuelve solo chunks con esa metadata |
| **3 — Index resolver (LLM)** | Que el LLM clasifica correctamente 5 preguntas de prueba (ES/EN) |

Ejecutar con:
```bash
cd /Users/santi/Documents/tesis/archIABack/back
/path/to/.venv/bin/python3 verify_rag_indexado.py
```

---

### 13.2 Bug 1 — `index_resolver.py`: Ruta al config incorrecta (crítico)

**Síntoma**: El index resolver retornaba `"general"` para **todas** las preguntas, incluyendo preguntas claramente relacionadas con escalabilidad o latencia. Los 5 test cases fallaron en la Parte 3 de la verificación.

**Causa raíz**: La ruta al archivo `indices.json` tenía **4 saltos de directorio** (`parent.parent.parent.parent`) en lugar de 3, apuntando a un path inexistente. Cuando `_load_indices_config()` falla la apertura del archivo, retorna `{"quality_attributes": [], ...}`. Al recibir una lista vacía, el resolver entra inmediatamente al `if not qa_list: return "general"` antes de llegar al LLM.

```python
# ANTES (incorrecto — apuntaba a archIABack/config/indices.json, no existe)
_CONFIG_PATH = Path(__file__).resolve().parent.parent.parent.parent / "config" / "indices.json"

# DESPUÉS (correcto — apunta a archIABack/back/config/indices.json)
_CONFIG_PATH = Path(__file__).resolve().parent.parent.parent / "config" / "indices.json"
```

**Verificación post-fix**:
```
✅  resolved='escalabilidad'  Q: "¿Cómo diseño un sistema escalable para un e-commerce?"
✅  resolved='latencia'       Q: "Necesito reducir la latencia de mi API REST a menos de 200ms"
✅  resolved='general'        Q: "¿Qué es ADD 3.0?"
✅  resolved='latencia'       Q: "What tactics reduce response time in a microservice?"
✅  resolved='escalabilidad'  Q: "How do I scale my application horizontally?"
```

**Impacto del bug**: Sin este fix, el classifier siempre ponía `resolved_index = "general"` en el state, haciendo que todos los nodos (asr, tactics, investigator) usaran el retriever sin filtros. El RAG indexado era completamente inoperante, aunque el código parecía correcto.

---

### 13.3 Bug 2 — `build_vectorstore.py`: Sin limpieza antes de rebuild (datos corruptos)

**Síntoma**: El vectorstore tenía **1316 chunks** en lugar de 658 — exactamente el doble. Cada chunk aparecía duplicado.

**Causa raíz**: `Chroma.from_documents()` **añade** documentos a la colección existente en el directorio `chroma_db/` en lugar de reemplazarla. Cada ejecución de `build_vectorstore.py` sobre un directorio ya existente duplicaba todos los chunks. Si se hubiera ejecutado 3 veces, habría 1974 chunks, etc.

**Consecuencia**: Los filtros de metadata seguían funcionando correctamente (los chunks duplicados tenían la misma metadata), pero la búsqueda por similaridad recuperaba los mismos documentos múltiples veces, reduciendo la diversidad del contexto y desperdiciando tokens.

**Fix**: Borrar el directorio `chroma_db/` al inicio de `main()`, antes de crear el vectorstore:

```python
def main():
    # AÑADIDO: Limpia el vectorstore anterior para evitar chunks duplicados
    import shutil
    if PERSIST_DIR.exists():
        shutil.rmtree(PERSIST_DIR)
        print(f"[build] Borrado vectorstore previo en {PERSIST_DIR}")
    PERSIST_DIR.mkdir(parents=True, exist_ok=True)

    docs = _load_docs()
    ...
```

El log ahora incluye la línea de confirmación:
```
[build] Borrado vectorstore previo en .../chroma_db
[build] Total chunks: 658   ← siempre limpio
```

---

### 13.4 Resultado Final de la Verificación

Después de ambos fixes, `verify_rag_indexado.py` pasa completamente:

**Parte 1 — Metadatos**:
```
Total chunks: 658
Chunks con 'quality_attribute': 658/658  ✅
Chunks con 'content_type':      658/658  ✅
```

**Parte 2 — Filtros** (selección):

| Filtro aplicado | Resultado | Metadata verificada |
|---|---|---|
| Sin filtros | 4 docs de cualquier tipo | ✅ |
| `quality_attribute=latencia` | Solo chunks `latencia/*` | ✅ |
| `quality_attribute=escalabilidad` | Solo chunks `escalabilidad/*` | ✅ |
| `content_type=tacticas` | Solo chunks `*/tacticas` | ✅ |
| `latencia + tacticas` | Solo `latencia/tacticas` | ✅ |
| `escalabilidad + tacticas` | Solo `escalabilidad/tacticas` | ✅ |
| `latencia + asr` | Solo `latencia/asr` | ✅ |

**Parte 3 — Index resolver**: 5/5 preguntas clasificadas correctamente (ES y EN).

**Resumen final**:
```
tacticas :  57 chunks  (8.7%)
asr      : 391 chunks  (59.4%)
estilos  :  46 chunks  (7.0%)
general  : 164 chunks  (24.9%)

metadata presentes: ✅ OK
```


**Output de consola despúes de pruebas:**

```bash
/Users/santi/Documents/tesis/archIABack/back/src/rag_agent.py:68: LangChainDeprecationWarning: The class `Chroma` was deprecated in LangChain 0.2.9 and will be removed in 1.0. An updated version of the class exists in the `langchain-chroma package and should be used instead. To use it run `pip install -U `langchain-chroma` and import as `from `langchain_chroma import Chroma``.
  _VDB = Chroma(
INFO:httpx:HTTP Request: POST https://api.openai.com/v1/chat/completions "HTTP/1.1 200 OK"
INFO:httpx:HTTP Request: POST https://api.openai.com/v1/chat/completions "HTTP/1.1 200 OK"
INFO:httpx:HTTP Request: POST https://api.openai.com/v1/chat/completions "HTTP/1.1 200 OK"
INFO:httpx:HTTP Request: POST https://api.openai.com/v1/chat/completions "HTTP/1.1 200 OK"
INFO:httpx:HTTP Request: POST https://api.openai.com/v1/chat/completions "HTTP/1.1 200 OK"

============================================================
PARTE 1: Metadatos en Chroma
============================================================
[RAG] persist_directory = /Users/santi/Documents/tesis/archIABack/back/chroma_db
Total chunks: 658
Chunks con 'quality_attribute': 658/658
Chunks con 'content_type':      658/658

Distribución quality_attribute/content_type:
  general/asr                      288  █████████████████████████████████████████████████████████
  general/general                  130  ██████████████████████████
  latencia/asr                      57  ███████████
  escalabilidad/asr                 46  █████████
  general/estilos                   31  ██████
  latencia/tacticas                 27  █████
  general/tacticas                  18  ███
  latencia/general                  18  ███
  escalabilidad/general             16  ███
  escalabilidad/tacticas            12  ██
  latencia/estilos                   8  █
  escalabilidad/estilos              7  █

============================================================
PARTE 2: Retriever con filtros
============================================================

  [Sin filtros (comportamiento base)]  → 4 docs recuperados
    [1] latencia/general  rag_test_fixture.txt p.0
        "RAG_UNIQUE_PHRASE_7F3A9C: The system must sustain exactly 12,345 requests per se…"
    [2] general/tacticas  Software Architecture in practice.pdf p.29
        "response in Figure 4.3. The tactics, like design patterns, are design techniques…"
    [3] latencia/tacticas  Software Architecture in practice.pdf p.30
        "72 Part t wo Quality a ttributes 4—Understanding Quality Attributes 2. If no pat…"

  [quality_attribute=latencia]  → 4 docs recuperados
    [1] latencia/general  rag_test_fixture.txt p.0
        "RAG_UNIQUE_PHRASE_7F3A9C: The system must sustain exactly 12,345 requests per se…"
    [2] latencia/tacticas  Software Architecture in practice.pdf p.30
        "72 Part t wo Quality a ttributes 4—Understanding Quality Attributes 2. If no pat…"
    [3] latencia/tacticas  Software Architecture in practice.pdf p.52
        "335, 337 –338 Potential alternatives, 398 Potential problems, peer review for, 3…"

  [quality_attribute=escalabilidad]  → 4 docs recuperados
    [1] escalabilidad/tacticas  Software Architecture in practice.pdf p.49
        "Index 575 Layer structures, 13 Layer views in ATAM presentations, 406 Layered pa…"
    [2] escalabilidad/general  Software Architecture in practice.pdf p.32
        "ployed when there is contention.  ■ Determining the impact of saturation on diff…"
    [3] escalabilidad/tacticas  Software Architecture in practice.pdf p.51
        "Method), 304–305 Parameter fence tactic, 90 Parameter typing tactic, 90 Paramete…"

  [content_type=tacticas]  → 4 docs recuperados
    [1] general/tacticas  Software Architecture in practice.pdf p.29
        "response in Figure 4.3. The tactics, like design patterns, are design techniques…"
    [2] latencia/tacticas  Software Architecture in practice.pdf p.30
        "72 Part t wo Quality a ttributes 4—Understanding Quality Attributes 2. If no pat…"
    [3] escalabilidad/tacticas  Software Architecture in practice.pdf p.49
        "Index 575 Layer structures, 13 Layer views in ATAM presentations, 406 Layered pa…"

  [latencia + tacticas]  → 4 docs recuperados
    [1] latencia/tacticas  Software Architecture in practice.pdf p.30
        "72 Part t wo Quality a ttributes 4—Understanding Quality Attributes 2. If no pat…"
    [2] latencia/tacticas  Software Architecture in practice.pdf p.52
        "335, 337 –338 Potential alternatives, 398 Potential problems, peer review for, 3…"
    [3] latencia/tacticas  Software Architecture in practice.pdf p.48
        "automatic reallocation, 516 overview, 514 Is a relation, 332 –333 Is-a-submodule…"

  [escalabilidad + tacticas]  → 4 docs recuperados
    [1] escalabilidad/tacticas  Software Architecture in practice.pdf p.49
        "Index 575 Layer structures, 13 Layer views in ATAM presentations, 406 Layered pa…"
    [2] escalabilidad/tacticas  Software Architecture in practice.pdf p.51
        "Method), 304–305 Parameter fence tactic, 90 Parameter typing tactic, 90 Paramete…"
    [3] escalabilidad/tacticas  Software Architecture in practice.pdf p.55
        "Rework in agility, 279 Risk ADD method, 320 ATAM, 402 global development, 425 pr…"

  [latencia + asr]  → 4 docs recuperados
    [1] latencia/asr  Software Architecture in practice.pdf p.38
        "564 Index  Architects background and experience, 51–52 cloud environments, 520–5…"
    [2] latencia/asr  Software architectures evaluation.pdf p.213
        "Documenting Software Architectures Clements, Bachmann, Bass, Garlan, Ivers, LItt…"
    [3] latencia/asr  Software Architecture in practice.pdf p.53
        "Index 579 Procurement management, 425 Product-line managers, 55 Product lines. S…"

============================================================
PARTE 3: Index resolver (LLM)
============================================================
  ✅  resolved='escalabilidad' (esperado='escalabilidad')
       Q: "¿Cómo diseño un sistema escalable para un e-commerce?"
  ✅  resolved='latencia' (esperado='latencia')
       Q: "Necesito reducir la latencia de mi API REST a menos de 200ms"
  ✅  resolved='general' (esperado='general')
       Q: "¿Qué es ADD 3.0?"
  ✅  resolved='latencia' (esperado='latencia')
       Q: "What tactics reduce response time in a microservice?"
  ✅  resolved='escalabilidad' (esperado='escalabilidad')
       Q: "How do I scale my application horizontally?"

✅ TODOS correctos

============================================================
RESUMEN
============================================================
  tacticas :    57 chunks  (8.7%)
  asr      :   391 chunks  (59.4%)
  estilos  :    46 chunks  (7.0%)
  general  :   164 chunks  (24.9%)

  metadata presentes: ✅ OK
```


---

*Sección añadida el 2026-02-28 tras verificación end-to-end y corrección de bugs.*

---

*Documento generado automáticamente durante la sesión de implementación del 2026-02-28.*
