Diferencias entre los modos de investigación (research_mode)

Los cuatro modos de investigación (pseudo_adaptive, llm_guided, bfs_deep, research_filter) determinan cómo se descubren y seleccionan enlaces secundarios a partir de las páginas iniciales (resultados de búsqueda). Su objetivo es profundizar en un sitio o tema siguiendo enlaces relevantes, pero cada uno tiene una estrategia distinta que afecta al rendimiento, exhaustividad y precisión.
1. pseudo_adaptive (predeterminado)

    Estrategia: Puntuación simple de palabras clave en las URLs.

    Cómo funciona:

        Las URLs iniciales se ordenan por número de coincidencias de palabras de la consulta en la propia URL.

        Se rastrean en lotes respetando la profundidad máxima.

        Los enlaces descubiertos se añaden a una cola con una puntuación basada también en coincidencias de keywords en la URL.

        Se siguen primero los enlaces con mayor puntuación.

    Fortalezas:

        Muy rápido, sin llamadas adicionales al LLM.

        Útil cuando la estructura de URLs es descriptiva (e.g., /wiki/Nombre_Artículo).

    Debilidades:

        No analiza el contenido real de la página, solo la URL.

        Puede seguir enlaces con URLs engañosas o poco relevantes.

    Ideal para: Sitios con URLs limpias y descriptivas (Wikipedia, blogs con slugs significativos). Búsquedas factuales donde la relevancia se refleja en el título/URL.

2. llm_guided

    Estrategia: Selección de enlaces guiada por un LLM (no implementada actualmente como LLM, sino con puntuación de keywords; el nombre es heredado de un diseño original). En la implementación actual, funciona de forma similar a pseudo_adaptive pero con una lógica ligeramente diferente: procesa una URL cada vez (modo secuencial) y puntúa enlaces descubiertos con keywords para decidir los siguientes.

    Cómo funciona:

        Rastrea una URL, extrae enlaces, los filtra por dominio (si include_external=False) y los puntúa con keywords.

        Añade los 3 mejores enlaces a la lista de procesamiento.

        No hay llamada real al LLM para elegir enlaces (a pesar del nombre), aunque la válvula RESEARCH_LLM_LINK_SELECTION existe y podría habilitarse en futuras versiones.

    Fortalezas:

        Más cauteloso que pseudo_adaptive porque solo sigue 3 enlaces por página.

        Permite un control más fino si se implementara la selección por LLM.

    Debilidades:

        Actualmente muy similar a pseudo_adaptive pero sin procesamiento en lote, por lo que es más lento.

        El nombre puede inducir a error.

    Ideal para: Cuando se quiere un rastreo conservador y se espera que los enlaces más relevantes aparezcan pronto. No recomendado en su estado actual a menos que se prefiera la exploración secuencial.

3. bfs_deep (Breadth-First Search profundo)

    Estrategia: Exploración por niveles (anchura) usando una cola FIFO, sin puntuación de relevancia.

    Cómo funciona:

        Todas las URLs iniciales se encolan.

        En cada iteración se procesa un lote del mismo nivel de profundidad.

        Los enlaces descubiertos se añaden al final de la cola, respetando el dominio base si include_external=False.

        No se prioriza por relevancia, solo por orden de descubrimiento.

    Fortalezas:

        Cobertura exhaustiva de un sitio, ideal para mapear toda una sección o documentación.

        No depende de la calidad de las URLs.

    Debilidades:

        Puede desperdiciar recursos en páginas poco relevantes.

        Sin límite de puntuación, puede seguir enlaces de navegación, pies de página, etc.

    Ideal para: Sitios muy estructurados y autocontenidos donde se desea una visión completa (documentación técnica, manuales, wikis internas). También para análisis de sitios completos.

4. research_filter

    Estrategia: Rastreo selectivo con retroalimentación de relevancia del contenido.

    Cómo funciona:

        Rastrea una URL inicial (semilla).

        Evalúa la relevancia del contenido extraído contando coincidencias de keywords.

        De los enlaces descubiertos, selecciona hasta 15, los puntúa por keywords en la URL y elige los mejores (máximo 3) para seguir.

        Solo sigue enlaces si su puntuación > 0 (contiene al menos una keyword).

        Se detiene cuando se alcanza el límite de páginas.

    Fortalezas:

        Feedback del contenido real para decidir si una fuente es prometedora.

        Evita seguir enlaces desde páginas irrelevantes.

        Más robusto que la simple puntuación de URL.

    Debilidades:

        Requiere al menos una keyword en la URL del enlace; puede perder enlaces con URLs opacas.

        No propaga puntuación entre niveles (solo mira el contenido de la página actual).

    Ideal para: Investigaciones donde las páginas semilla son de alta calidad y se espera que los enlaces relevantes contengan términos de la consulta. Bueno para filtrar ruido.

¿Cómo tomar una decisión informada sobre el modo óptimo?

La elección del modo de investigación depende de varios factores:

    Naturaleza de la consulta:

        Factual, definiciones, noticias: pseudo_adaptive o incluso desactivar investigación. No se necesita profundidad.

        Comparaciones, análisis, revisiones: research_filter o bfs_deep si el tema está muy acotado.

        Exploración exhaustiva de un dominio: bfs_deep (por ejemplo, "todas las versiones de la API de Kubernetes").

    Estructura esperada de los sitios objetivo:

        Si los resultados de búsqueda apuntan a sitios con URLs descriptivas (Wikipedia, blogs con slugs), pseudo_adaptive funciona bien.

        Si los sitios usan URLs opacas (parámetros, IDs numéricos), pseudo_adaptive fallará; es mejor research_filter o incluso no investigar.

        Si se trata de un único dominio que se quiere mapear (e.g., la documentación de un proyecto), bfs_deep con include_external=False es ideal.

    Presupuesto de tokens y tiempo:

        pseudo_adaptive y bfs_deep pueden disparar el número de páginas rastreadas, consumiendo muchos tokens.

        llm_guided y research_filter son más conservadores.

    ¿Necesitas seguir enlaces externos?

        La válvula RESEARCH_INCLUDE_EXTERNAL permite saltar a otros dominios. En modos como bfs_deep, si se activa, puede provocar una explosión de URLs.

Recomendación práctica (sin tener que elegir manualmente)

Puedes automatizar la selección del modo basándote en heurísticas simples. Por ejemplo, antes de ejecutar la investigación, analiza los dominios de las URLs recolectadas:

    Si más del 50% de las URLs pertenecen a un mismo dominio y ese dominio es conocido por tener estructura wiki/documental (wikipedia.org, readthedocs.io, github.com), sugiere bfs_deep con include_external=False.

    Si las URLs son de múltiples dominios y las URLs contienen slugs con guiones o barras descriptivas, pseudo_adaptive es seguro y rápido.

    Si se detectan muchos parámetros de consulta en las URLs o dominios de redes sociales, mejor evitar la investigación o usar research_filter para filtrar agresivamente.

Actualmente, el usuario decide manualmente, pero un futuro "modo automático" podría implementar una función _suggest_research_mode(urls, query) que devuelva el modo más prometedor.
Resumen visual
Modo	Velocidad	Exhaustividad	Precisión	Dependencia de calidad URLs
pseudo_adaptive	Alta	Media	Baja	Alta
llm_guided	Baja	Baja-Media	Media	Media
bfs_deep	Media	Alta	Baja	Baja (requiere filtro extra)
research_filter	Media	Media	Alta	Media

Consejo final: Activa el modo investigación solo cuando la consulta requiera múltiples fuentes o sea compleja. Para búsquedas simples, déjalo desactivado; el rastreo normal con filtro semántico y LLM ya ofrece muy buenos resultados sin la sobrecarga de seguir enlaces.
