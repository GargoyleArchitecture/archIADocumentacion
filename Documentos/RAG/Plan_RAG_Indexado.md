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

*Documento generado automáticamente durante la sesión de implementación del 2026-02-28.*
