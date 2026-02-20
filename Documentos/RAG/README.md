# README -- Sección RAG

## 1. Propósito

Esta carpeta contiene todos los recursos relacionados con la
implementación, evaluación y documentación técnica del sistema **RAG
(Retrieval-Augmented Generation)** del proyecto.

El objetivo es aislar claramente los componentes de recuperación de
información y su integración con modelos generativos, evitando mezclar
esta capa con investigación general o documentación académica.

------------------------------------------------------------------------

## 2. Alcance de la Carpeta

Aquí deben almacenarse exclusivamente materiales vinculados a:

-   Diseño del pipeline RAG.
-   Estrategias de chunking y segmentación de documentos.
-   Métodos de embedding y selección de modelos.
-   Configuración de bases vectoriales.
-   Evaluación de recuperación (precision@k, recall@k, MRR, etc.).
-   Pruebas de integración entre retriever y LLM.
-   Experimentos de optimización y benchmarking.

------------------------------------------------------------------------

## 3. Estructura Recomendada

Ejemplo de organización interna:

/rag\
│\
├── /pipeline → Definición y orquestación del flujo RAG\
├── /embeddings → Configuración y pruebas de modelos de embeddings\
├── /vectorstore → Implementación y pruebas de bases vectoriales\
├── /evaluacion → Métricas y análisis de rendimiento\
└── /experimentos → Pruebas controladas y comparativas

La estructura puede ajustarse según evolución del proyecto, pero debe
mantener separación conceptual clara.

------------------------------------------------------------------------

## 4. Qué NO debe incluir esta carpeta

No deben colocarse aquí:

-   Marco teórico general o bibliografía (va en investigación).
-   Documentación académica universitaria.
-   Notas personales sin relación directa con RAG.
-   Código general del sistema que no interactúe con el pipeline de
    recuperación.

------------------------------------------------------------------------

## 5. Principios Técnicos

La implementación RAG debe documentar:

-   Fuente de datos.
-   Estrategia de indexación.
-   Modelo de embeddings utilizado.
-   Método de recuperación.
-   Configuración del LLM.
-   Métricas de evaluación aplicadas.

Sin estos elementos, el componente no es reproducible ni auditable.

------------------------------------------------------------------------

## 6. Control de Versiones

README generado el: 20-02-2026
