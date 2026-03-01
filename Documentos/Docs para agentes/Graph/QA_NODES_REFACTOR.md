# Refactor de Nodos por Atributo de Calidad (QA)

Este documento describe, en detalle, los cambios realizados para soportar nodos de **estilos** y **tácticas** separados por atributo de calidad (QA), manteniendo la estructura general del grafo y mejorando mantenibilidad.

## Objetivo de la implementación

1. Separar nodos de estilos y tácticas por QA (latencia y escalabilidad).
2. Mantener compatibilidad con el enrutamiento existente del grafo.
3. Evitar hardcode excesivo y preparar el proyecto para escalar a nuevos QAs.
4. Hacer que el QA sea parte explícita del estado de conversación, similar al `intent`.

## Resumen de cambios

### 1) Estructura de archivos de nodos

Se reorganizaron nodos en carpetas por dominio:

- `src/graph/nodes/styles/`
  - `style.py`: nodo general de estilos (`style_node`) y factoría (`make_style_qa_node`).
  - `latency_style.py`: nodo específico `style_latency_node`.
  - `scalability_style.py`: nodo específico `style_scalability_node`.
  - `common.py`: implementación compartida (`style_node_impl`) y resolución QA.
  - `__init__.py`: exports del paquete.

- `src/graph/nodes/tactics/`
  - `tactics.py`: nodo general de tácticas (`tactics_node`) y factoría (`make_tactics_qa_node`).
  - `latency_tactics.py`: nodo específico `tactics_latency_node`.
  - `scalability_tactics.py`: nodo específico `tactics_scalability_node`.
  - `common.py`: implementación compartida (`tactics_node_impl`) y resolución QA.
  - `__init__.py`: exports del paquete.

Además, se eliminaron módulos antiguos de raíz:

- `src/graph/nodes/style.py`
- `src/graph/nodes/tactics.py`

### 2) Clasificación de QA en el `classifier`

Se extendió la salida estructurada del clasificador para incluir QA explícito.

- Archivo: `src/graph/state.py`
  - `ClassifyOut` ahora incluye: `quality_attribute: str`.

- Archivo: `src/graph/nodes/classifier.py`
  - El prompt del clasificador pide ahora `quality_attribute` en el mismo bloque donde pide `intent` y `language`.
  - El `quality_attribute` resultante se normaliza y se guarda en estado.
  - `resolved_index` (usado para RAG) prioriza QA clasificado y usa fallback si aplica.

### 3) Registro central de QA (mantenibilidad)

Se creó un registro único para normalización, catálogo y naming de nodos QA.

- Archivo: `src/graph/qa_registry.py`
  - `supported_qas()`: lee QAs desde `config/indices.json`.
  - `normalize_qa(value)`: convierte texto libre/sinónimos a ID canónico.
  - `style_node_name_for_qa(qa)`, `tactics_node_name_for_qa(qa)`: generan nombre de nodo.
  - `qa_to_focus_label(qa, default)`: evita ternarios hardcodeados en prompts.

Con esto se redujo duplicación de heurísticas QA en nodos.

### 4) Enrutamiento dinámico por QA en `workflow`

- Archivo: `src/graph/workflow.py`
  - Imports actualizados para usar paquetes `styles` y `tactics`.
  - `router(...)` usa `quality_attribute/resolved_index` normalizados para decidir:
    - `style` -> nodo especializado por QA (o fallback a `style`).
    - `tactics` -> nodo especializado por QA (o fallback a `tactics`).
  - Registro dinámico de nodos QA a partir de `supported_qas()`.
  - Edges de retorno de nodos especializados hacia `supervisor` también dinámicos.

### 5) Ajustes en ASR para coherencia de QA

- Archivo: `src/graph/nodes/asr.py`
  - Alineación de QA de pipeline usando normalización central.
  - Eliminación de ternario rígido para foco (`latency`/`scalability`) y uso de `qa_to_focus_label(...)`.
  - `quality_attribute` queda coherente para nodos siguientes (`style`/`tactics`).

## Comportamiento funcional resultante

1. El clasificador define `intent` y `quality_attribute` en una sola clasificación.
2. El supervisor mantiene `nextNode` lógico (`style`/`tactics`).
3. El router traduce eso a nodo QA-específico:
   - `style_latency` / `style_scalability`
   - `tactics_latency` / `tactics_scalability`
4. Si QA no es claro, se usa nodo general (`style` o `tactics`).

## Cómo agregar un nuevo QA (guía rápida)

1. Agregar QA en `config/indices.json` con:
   - `id`
   - `keywords_en`
   - `keywords_es`
   - (opcional) `display_name`
2. Reindexar vectorstore para que existan chunks etiquetados con ese QA.
3. Reiniciar la aplicación.

No debería requerir cambios manuales en `workflow.py`, ni crear rutas hardcode nuevas para ese QA.

## Validaciones realizadas

- Compilación de sintaxis del paquete grafo:
  - `python -m compileall src/graph`

Resultado: compilación correcta de los módulos modificados.

## Cosas aprendidas en este proceso

- **Aprendizaje clave:** cuando una dimensión de negocio (como QA) impacta múltiples nodos, prompts y rutas, conviene modelarla como una entidad de primer nivel (en estado + registro central) en lugar de resolverla con condicionales dispersos por archivo. Esto reduce acoplamiento, evita regresiones al escalar y simplifica futuras implementaciones.
