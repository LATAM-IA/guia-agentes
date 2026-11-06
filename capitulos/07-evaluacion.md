# Capítulo 7. Evaluación de agentes

La **evaluación (evaluation, eval)** permite comprobar si nuestro agente cumple el objetivo, respeta las restricciones y mantiene su comportamiento cuando cambiamos el modelo, las instrucciones o las herramientas. Nos ayuda a elegir entre versiones y a detectar regresiones antes de confiarle más tareas.

En el capítulo anterior definimos un circuito de planificación, ejecución y observación. Ahora necesitamos probarlo. Una respuesta convincente, una herramienta que devuelve éxito o una sola ejecución correcta no bastan para validar todo el sistema.

Distinguiremos cuatro preguntas:

- **Resultado:** ¿se alcanzó el estado esperado?
- **Trayectoria:** ¿qué solicitudes, ejecuciones y resultados condujeron a ese estado?
- **Fiabilidad:** ¿con qué consistencia se repite el éxito bajo condiciones definidas?
- **Seguridad:** ¿se respetan los permisos y las restricciones incluso ante errores o manipulación?

Como ejemplo propio, continuaremos con el filtro por fecha. Si una persona pide obtener los registros de un día, comprobaremos cuáles devuelve el agente, qué intervalo solicitó y si modificó datos que solo debía consultar. Los casos del capítulo son **ilustrativos y propuestos para evaluación**; no representan ejecuciones reales de un agente.

La guía de Anthropic publicada el 9 de enero de 2026 distingue el registro de interacción del resultado final: “The outcome is the final state in the environment”. Conservaremos esa separación durante toda la evaluación. [Fuente primaria](https://www.anthropic.com/research/demystifying-evals-for-ai-agents).

# Benchmarks y clasificaciones públicas

Un **banco de pruebas (benchmark)** reúne tareas, condiciones de ejecución y criterios de puntuación para comparar sistemas. Puede mantenerse como una versión fija o evolucionar mediante revisiones. Para reproducir una evaluación necesitamos identificar qué versión se utilizó.

Una **clasificación pública (leaderboard)** presenta resultados obtenidos bajo un protocolo. Su posición resume ese experimento; no determina automáticamente qué agente conviene para nuestra aplicación. Un porcentaje de aciertos en preguntas académicas y un porcentaje de tareas completadas con herramientas representan capacidades diferentes.

La unidad evaluada también importa. Podemos medir un modelo con una instrucción, un adaptador de llamadas a herramientas o el sistema completo con memoria, permisos, reintentos y entorno de ejecución. No atribuiremos al modelo aislado un resultado que depende de toda esa configuración.

## Programación e ingeniería de software

**SWE-bench**, publicado inicialmente el 10 de octubre de 2023, plantea resolver incidencias de repositorios mediante cambios en el código. El resultado se comprueba con un protocolo de evaluación del parche. Es un antecedente útil anterior a la ventana de novedades de esta revisión. [Publicación original](https://arxiv.org/abs/2310.06770).

**SWE-bench Verified** es un subconjunto revisado por personas para reducir problemas de evaluabilidad. No equivale al conjunto completo ni garantiza que todos los casos sean perfectos. [Documentación oficial](https://www.swebench.com/SWE-bench/faq/).

Al interpretar una resolución, conviene distinguir las pruebas que deben pasar de fallar a aprobar (**FAIL_TO_PASS**) de las que deben seguir aprobando (**PASS_TO_PASS**). El evaluador comprueba ambos grupos según la instancia. No se trata simplemente de ejecutar cualquier test existente y aceptar el parche si alguno pasa. [Implementación oficial de la evaluación](https://raw.githubusercontent.com/SWE-bench/SWE-bench/main/swebench/harness/grading.py).

**Pasar los tests verifica los casos que esos tests cubren.** Puede quedar comportamiento incorrecto fuera de ellos. En nuestro agente combinaremos la prueba específica del cambio con comprobaciones de regresión y, cuando corresponda, observación de la aplicación.

**Terminal-Bench** evalúa tareas realizadas desde una terminal. Sus versiones cambian tareas, recursos y verificadores. El anuncio de **Terminal-Bench 4.0**, publicado el 28 de agosto de 2026 según el índice oficial, explica cambios de recursos y tareas que requieren nuevas ejecuciones. Esto muestra por qué no debemos mezclar resultados de versiones distintas. [Anuncio de los autores](https://www.tbench.ai/news/terminal-bench-4-0).

## Uso de herramientas

Los benchmarks de **uso de herramientas (tool use)** permiten comprobar selección de operaciones, argumentos y continuidad entre turnos. Algunas variantes evalúan solicitudes; otras también ejecutan acciones y verifican el estado final. Debemos leer el protocolo antes de interpretar una puntuación.

La familia **Berkeley Function Calling Leaderboard (BFCL)** incluye evaluaciones de llamadas a funciones y ha incorporado interacciones de varios turnos. Su versión **V4**, publicada el 17 de julio de 2025, amplía la evaluación hacia tareas con búsqueda web. No describiremos toda la familia como una prueba estática de argumentos. [Publicación oficial de BFCL V4](https://gorilla.cs.berkeley.edu/blogs/15_bfcl_v4_web_search.html).

**τ-bench**, publicado el 17 de junio de 2024, combina herramientas, reglas del dominio y conversaciones con un usuario simulado mediante modelos. Comprueba el estado de la base de datos frente al objetivo anotado. La simulación permite repetir conversaciones, pero introduce otro componente cuyo comportamiento puede variar. [Publicación original](https://arxiv.org/abs/2406.12045).

Para nuestro agente separaremos dos comprobaciones: que la solicitud sea válida y que la operación produzca el efecto esperado. Una llamada bien formada puede consultar el intervalo equivocado o realizar un cambio no autorizado.

## Uso de computadoras y navegadores

Las tareas de **uso de computadoras (computer use)** evalúan interacción con aplicaciones, archivos e interfaces. Exigen observar cambios y decidir acciones en un entorno, más allá de generar una respuesta textual.

**OSWorld**, publicado el 11 de abril de 2024, utiliza entornos de computación reales para tareas sobre aplicaciones. No es únicamente una colección de pantallas simuladas ni garantiza que una configuración concreta reproduzca cualquier computadora de una persona. [Publicación original](https://arxiv.org/abs/2404.07972).

**WebArena**, publicado el 25 de julio de 2023, ofrece entornos web reproducibles para tareas de navegación e interacción. Conservar ese control del entorno ayuda a evaluar cambios sin depender de sitios públicos que varían o producen efectos sobre terceros. [Publicación original](https://arxiv.org/abs/2307.13854).

En el filtro por fecha, un test de la consulta y una comprobación de la interfaz cubren aspectos diferentes. El primero puede validar los registros devueltos; la segunda permite observar si el control transmite el intervalo correcto y presenta el resultado esperado.

## Tareas de conocimiento y trabajo profesional

**GAIA**, publicado el 21 de noviembre de 2023, reúne preguntas que pueden requerir búsqueda, lectura de archivos y otras capacidades para alcanzar una respuesta verificable. No debe confundirse con demostrar que un agente completa cualquier trabajo profesional. [Publicación original](https://arxiv.org/abs/2311.12983).

**GDPval**, presentado el 25 de septiembre de 2025, evalúa entregables de tareas profesionales de distintas ocupaciones. Amplía el foco más allá del código, pero la calidad de un entregable no representa toda la responsabilidad, interacción y duración de una ocupación. [Presentación de los autores](https://openai.com/index/gdpval/).

El estudio **How Well Does Agent Development Reflect Real-World Work?**, publicado el 1 de marzo de 2026, analizó desajustes entre los benchmarks estudiados y la distribución del trabajo humano. Es evidencia sobre esa selección y su metodología; no demuestra qué demanda comercial tendrá nuestro agente. [Investigación original](https://arxiv.org/abs/2603.01203).

La lección para nuestro producto es elegir tareas de evaluación que reflejen sus usos, incluidos los errores costosos, y no sustituir esa selección por el benchmark con mayor visibilidad.

## Razonamiento y conocimiento

Los benchmarks de conocimiento permiten explorar capacidades del modelo subyacente. **MMLU**, publicado inicialmente el 7 de septiembre de 2020, reúne preguntas de distintas materias. [Publicación original](https://arxiv.org/abs/2009.03300).

**GPQA**, publicado el 20 de noviembre de 2023, contiene preguntas de ciencias elaboradas por especialistas; **GPQA Diamond** es un subconjunto y debe identificarse como tal. El propósito de dificultar la resolución por búsqueda no significa inmunidad permanente a contaminación o cambios en las herramientas. [Publicación original](https://arxiv.org/abs/2311.12022).

**Humanity’s Last Exam**, publicado inicialmente el 24 de enero de 2025, reúne preguntas académicas exigentes. Hay que precisar si una evaluación permite herramientas y qué versión utiliza. Su nombre no establece una medida definitiva de inteligencia ni de fiabilidad operativa. [Publicación de los autores](https://arxiv.org/abs/2501.14249).

Estos resultados pueden orientar una selección inicial. Para comprobar nuestro agente necesitaremos tareas con sus instrucciones, herramientas y restricciones reales.

## Capacidades especializadas

Otras pruebas aíslan capacidades concretas:

- **ARC-AGI** propone tareas de abstracción y transformación de cuadrículas. Hay que identificar la versión; no es el mismo benchmark que las preguntas de ciencias de ARC Easy o ARC Challenge. [Repositorio de los autores](https://github.com/fchollet/ARC-AGI).
- **MMMU**, publicado el 27 de noviembre de 2023, combina contenido visual y textual de distintas disciplinas. Comprueba comprensión multimodal, sin representar por sí solo una interacción completa con aplicaciones. [Publicación original](https://arxiv.org/abs/2311.16502).
- Las pruebas de **aguja en un pajar (needle in a haystack)** insertan información en un contexto largo y comprueban si se recupera. Encontrar un dato no equivale a relacionar fuentes, detectar contradicciones o sostener una tarea extensa. [Implementación del autor](https://github.com/gkamradt/needle-in-a-haystack).

Para nuestro agente, una prueba aislada puede ayudar a diagnosticar una limitación. La aceptación del sistema seguirá dependiendo de resolver sus tareas completas.

## Cómo leer críticamente una puntuación

Antes de comparar sistemas, construiremos una ficha del experimento. La siguiente lista es una **síntesis propia para nuestro agente**:

| Pregunta | Dato que necesitamos |
|---|---|
| ¿Qué se intentó resolver? | Tareas, distribución, versión y subconjunto |
| ¿Qué sistema se ejecutó? | Modelo, instrucciones, herramientas, memoria y orquestación |
| ¿Cómo se midió? | Criterios, verificador, rúbrica y política de errores |
| ¿Cuántas oportunidades tuvo? | Ensayos por tarea, reintentos y estrategia de selección |
| ¿Qué recursos recibió? | Tiempo, concurrencia, CPU, memoria y acceso a red |
| ¿Cuánto costó? | Costo, tokens y tiempo según un límite definido |
| ¿Qué incertidumbre tiene el resultado? | Número de tareas, variación y método de estimación |
| ¿Quién seleccionó lo publicado? | Autoría, intentos omitidos y condiciones de reproducción |

La investigación **SWE Atlas**, publicada el 8 de mayo de 2026, compara sistemas sobre preguntas de código, escritura de tests y refactorización, utilizando evaluaciones específicas y distintos entornos de agente. Es un recordatorio experimental de que el resultado pertenece a un sistema y protocolo, no solo a una etiqueta de modelo. [Publicación original](https://arxiv.org/abs/2605.08366).

La **contaminación de datos (data contamination)** aparece cuando material de evaluación influye en el entrenamiento o la optimización. También puede aparecer **sobreajuste (overfitting)** al adaptar nuestras instrucciones a un conjunto de desarrollo conocido sin entrenar pesos. Mantener casos reservados ayuda a detectar ese problema, aunque no demuestra ausencia de contaminación previa del modelo.

La publicación **The Leaderboard Illusion**, del 29 de abril de 2025, examinó efectos de selección y divulgación en Arena. No invalida todas las clasificaciones; muestra por qué la independencia declarada no sustituye revisar el proceso de evaluación. [Estudio original](https://arxiv.org/abs/2504.20879).

Tampoco hay un porcentaje universal a partir del cual un benchmark esté saturado. Depende de cuántos casos diferencian a los sistemas, de sus errores y de la incertidumbre. Una diferencia pequeña puede no ser concluyente; una puntuación alta puede seguir ocultando un fallo inaceptable para nuestro producto.

# Evaluación de resultados

La **evaluación de resultados (outcome evaluation)** comprueba el producto o estado alcanzado al terminar una tarea. Puede evaluar una respuesta, un archivo, un conjunto de registros o un cambio en otro sistema. La frase final del agente es solo una evidencia más.

Para el filtro, fijaremos un contrato ilustrativo: consultar el día **3 de noviembre de 2026 en UTC**, incluyendo el inicio y excluyendo el comienzo del día siguiente. Utilizaremos estos datos sintéticos:

| Registro | Fecha y hora UTC | ¿Debe aparecer? |
|---|---|---|
| A | 2026-11-02 23:59:59 | No |
| B | 2026-11-03 00:00:00 | Sí |
| C | 2026-11-03 12:00:00 | Sí |
| D | 2026-11-03 23:59:59 | Sí |
| E | 2026-11-04 00:00:00 | No |

El resultado esperado es el conjunto **B, C y D**, sin duplicados. Si el orden no forma parte del contrato, no rechazaremos una respuesta por ordenar esos identificadores de otra manera. Esta decisión corresponde al caso de prueba; no cambia las reglas generales de todos los filtros.

Comprobaremos también que no hubo modificaciones, porque la tarea era de lectura. Así distinguimos corrección de la respuesta y cumplimiento de las restricciones.

## Evaluación humana

La **evaluación humana (human evaluation)** permite juzgar utilidad, pertinencia y aspectos que todavía no sabemos expresar mediante reglas fiables. Puede asignar una puntuación absoluta según criterios o comparar dos resultados para la misma tarea.

Una **comparación por pares (pairwise comparison)** pregunta cuál resultado se prefiere. Puede facilitar algunas decisiones, pero preferencia no equivale a corrección: una respuesta agradable puede contener errores. Los evaluadores necesitan instrucciones, contexto y posibilidad de señalar empates o casos que no pueden juzgar.

Como ejemplo ilustrativo, si A gana 6 comparaciones, B gana 3 y hay 1 empate, informaremos esos tres recuentos. Una **tasa de victorias (win rate)** que ignore empates sería 6/9; otra que asigne medio punto a cada empate sería 6,5/10. Debemos declarar qué convención usamos antes de comparar porcentajes.

En nuestro agente calibraremos la rúbrica con ejemplos revisados por más de una persona cuando la decisión lo justifique. Conservaremos desacuerdos importantes para aclarar criterios. No supondremos que cualquier opinión humana es una verdad objetiva ni que el acuerdo prueba exactitud factual.

Las clasificaciones de preferencias pueden utilizar modelos estadísticos de comparación, como Elo o Bradley–Terry. Su posición depende de oponentes, tareas y protocolo; no es un porcentaje de tareas reales resueltas. El trabajo de MT-Bench y Chatbot Arena ofrece un antecedente de evaluación por preferencias. [Publicación original, 9 de junio de 2023](https://arxiv.org/abs/2306.05685).

## Evaluación automatizada

La **evaluación automatizada (automated evaluation)** permite repetir comprobaciones de forma consistente durante el desarrollo. Puede combinar reglas, ejecución de tests y juicios de modelos. Lo que automatizamos es un criterio; debemos comprobar que ese criterio representa lo que queremos medir.

### Un evaluador mínimo para nuestro agente

Definiremos un **caso de evaluación (evaluation case)** con su objetivo, estado inicial, datos disponibles, restricciones, resultados permitidos y método de puntuación. Una **ejecución o ensayo (trial)** será un intento del agente sobre ese caso.

Separaremos cuatro responsabilidades:

1. **Preparar:** restaurar el entorno y cargar los datos del caso.
2. **Ejecutar:** iniciar el agente con la configuración y presupuesto definidos.
3. **Registrar:** conservar solicitudes, ejecuciones, resultados, respuesta final y estado del entorno.
4. **Calificar:** aplicar los criterios y guardar resultados por dimensión.

El agente no recibirá las respuestas reservadas ni el código privado del evaluador. Los datos necesarios para realizar la tarea sí deben estar disponibles. Ocultar información indispensable produciría un caso imposible, no una prueba rigurosa.

Restablecer la memoria conversacional no basta si persisten archivos, bases de datos o procesos de una ejecución anterior. Aislaremos todos los estados pertinentes para que un ensayo no se beneficie de otro ni lo perjudique. Si queremos evaluar aprendizaje entre sesiones, lo definiremos como otro protocolo, con memoria persistente controlada.

### Coincidencia exacta y normalización

La **coincidencia exacta (exact match)** sirve cuando hay una respuesta inequívoca y un formato acordado. Por ejemplo, podemos comparar identificadores o un valor numérico extraído. No es apropiada para exigir que todo resumen reproduzca literalmente una respuesta de referencia.

En el filtro, comparar conjuntos permite aceptar B, C y D independientemente del orden. Pero convertir primero a conjunto podría ocultar duplicados: comprobaremos por separado su ausencia si esa es una regla del caso.

La **normalización (normalization)** elimina diferencias irrelevantes bajo el contrato, como espacios externos. No debe borrar errores significativos. Cambiar una zona horaria o ignorar el signo de un importe no es una normalización inocua. Definiremos unidades, tolerancias y formatos antes de observar resultados.

Una evaluación de una pregunta con alternativas y otra de una respuesta abierta tampoco son intercambiables. Retirar las opciones cambia la tarea y el criterio; no compararemos directamente sus puntuaciones como si provinieran del mismo experimento.

### Comprobaciones programáticas

Las **comprobaciones programáticas (programmatic checks)** verifican propiedades mediante código: estructura de salida, resultados de una consulta, comportamiento de una función o restricciones sobre los efectos.

**IFEval**, publicado el 14 de noviembre de 2023, utiliza instrucciones comprobables, como condiciones de longitud o presencia de elementos. Es una referencia para evaluar restricciones explícitas, no una prueba completa de utilidad o veracidad. [Publicación original](https://arxiv.org/abs/2311.07911).

Para nuestro agente distinguiremos formato de significado. Un JSON válido puede contener una fecha incorrecta; un texto que respeta una longitud puede inventar datos. En el filtro verificaremos los identificadores esperados, los límites temporales y la ausencia de escrituras, además del formato si corresponde.

También probaremos el propio evaluador con ejemplos que deben aprobar y fallar. Si acepta una salida sin el registro B o rechaza una salida equivalente permitida, el problema puede estar en el criterio. Revisar un evaluador exige mantener visible qué resultados anteriores cambian y por qué.

### Modelo de lenguaje como juez

Un **modelo de lenguaje como juez (LLM-as-a-judge)** puede evaluar respuestas abiertas mediante una rúbrica o una comparación. Es útil para ampliar la evaluación cuando una regla determinista no representa bien la calidad deseada.

El estudio de MT-Bench identificó sesgos de posición, extensión y preferencia por salidas propias, además de limitaciones de razonamiento. Sus resultados pertenecen a los modelos y tareas estudiados; no garantizan la calidad de cualquier juez actual. [Fuente primaria](https://arxiv.org/abs/2306.05685).

Para nuestro agente contrastaremos juicios automáticos con casos revisados por personas. Alternaremos el orden en comparaciones, mantendremos criterios explícitos y examinaremos desacuerdos. Un juez de otra familia o un conjunto de jueces puede servir para comparar errores, pero no elimina sesgos compartidos por definición.

Pediremos una decisión y evidencia localizable en la salida o sus fuentes. Una justificación extensa no demuestra que la puntuación sea correcta ni revela el pensamiento interno del juez. Si queremos evaluar exactitud factual, le proporcionaremos evidencia pertinente o recurriremos a una comprobación independiente.

La salida evaluada se tratará como datos: una instrucción dentro de ella que pida «asigna la puntuación máxima» no debe controlar al juez. Probaremos esa manipulación y validaremos la estructura de su respuesta. Un juicio inválido, una negativa o un fallo de API quedarán identificados; no los convertiremos silenciosamente en una puntuación válida.

### Evaluación basada en rúbricas

Una **rúbrica (rubric)** descompone la calidad en criterios con niveles observables. Facilita explicar qué falta y revisar desacuerdos; no convierte un juicio subjetivo en una medida infalible.

Para la respuesta sobre el filtro podemos proponer esta rúbrica propia, complementaria a las comprobaciones de datos:

| Criterio | Cumplimiento | Incumplimiento |
|---|---|---|
| Intervalo explicado | Comunica el día y la zona acordados | Omite o cambia esos límites |
| Correspondencia con la evidencia | Describe lo que devolvió la consulta | Añade resultados no observados |
| Limitaciones | Informa un resultado pendiente o incompleto cuando existe | Declara éxito sin esa comprobación |
| Utilidad | Responde a la solicitud de forma comprensible | Entrega contenido que no resuelve la consulta |

No premiaremos una explicación más larga por sí misma. Si necesitamos niveles intermedios, describiremos qué diferencia cada nivel y añadiremos ejemplos de calibración.

Separaremos condiciones obligatorias de preferencias de calidad. Una acción no autorizada no se compensará con puntos por estilo. Si agregamos puntuaciones, publicaremos pesos y reglas; mantendremos los resultados por criterio para evitar que el promedio oculte fallos relevantes.

# Evaluación de la trayectoria

La **evaluación de la trayectoria (trajectory evaluation)** examina lo ocurrido durante el intento. Permite detectar acciones sin autorización, información inventada, llamadas repetidas o una recuperación incorrecta aunque la respuesta final parezca válida.

Registraremos por separado la **solicitud del modelo**, la **ejecución de la herramienta** y el **resultado observado**. Una solicitud rechazada por permisos no se contabilizará como una escritura ejecutada. Una operación aceptada pero pendiente tampoco se registrará como terminada.

El registro puede incluir explicaciones generadas cuando la integración las exponga. Son salidas del sistema, no acceso garantizado a su pensamiento interno. No diagnosticaremos una causa solo porque una explicación posterior parezca convincente.

## Comprobar restricciones sin imponer una única ruta

Para obtener B, C y D puede haber varias estrategias válidas. El agente podría solicitar un intervalo exacto o recuperar un conjunto permitido y filtrarlo. Si ambas respetan permisos, presupuesto y resultado, no exigiremos una secuencia idéntica a una trayectoria de referencia.

Distinguiremos restricciones necesarias de decisiones opcionales. Consultar con la zona horaria acordada puede ser necesario; usar una herramienta con un nombre específico solo lo será si el contrato lo exige. Una exploración que no resuelve directamente la tarea puede aportar información útil, aunque también consuma recursos.

Los trabajos **T-Eval**, publicado el 21 de diciembre de 2023, y **AgentBoard**, del 24 de enero de 2024, proponen analizar componentes del uso de herramientas y progreso en tareas de varios turnos. Son propuestas de evaluación con definiciones propias, no una única métrica universal del proceso. [T-Eval](https://arxiv.org/abs/2312.14033), [AgentBoard](https://arxiv.org/abs/2401.13178).

**TRACE**, publicado el 3 de octubre de 2025, propone evaluación multidimensional de trayectorias sin una trayectoria de referencia para cada caso. **AgentProcessBench**, publicado el 15 de marzo de 2026 y revisado en junio, aporta anotaciones humanas por pasos para estudiar evaluadores del proceso. Sus resultados ayudan a diseñar diagnósticos, pero un evaluador de pasos también debe validarse para nuestro dominio. [TRACE](https://arxiv.org/abs/2510.02837), [AgentProcessBench](https://arxiv.org/abs/2603.14465).

## Eficiencia y límites observables

Mediremos llamadas, pasos, errores, tiempo y costo según el propósito de la tarea. Compararemos ejecuciones que alcanzan calidad y restricciones equivalentes. Una solución rápida que omite una comprobación obligatoria no es una mejora.

Diferenciaremos tiempo total de espera, tiempo de herramientas y tiempo de generación cuando podamos medirlos. Informaremos también el comportamiento de casos lentos mediante percentiles u otros resúmenes adecuados, con el número de observaciones. Una media sola puede ocultar esperas largas.

La cantidad de tokens o el texto de una explicación no demuestran sobreanálisis. Pueden señalar un costo que conviene investigar; para decidir si sobra trabajo necesitamos revisar qué información produjo y qué efecto tuvo en el resultado.

En nuestro filtro comprobaremos, por ejemplo, que un fallo transitorio no provoca reintentos ilimitados. Si hay una operación con efectos, verificaremos que un reintento no los duplique. Estos son casos propios que necesitamos implementar, no resultados obtenidos por contar llamadas en un gráfico.

# Fiabilidad y variabilidad

La **fiabilidad (reliability)** describe la consistencia del sistema bajo condiciones especificadas. Repetir tareas permite detectar variación del modelo, del usuario simulado, de herramientas y del entorno. Incluso con una configuración de generación aparentemente determinista pueden existir otras fuentes de variación.

Distinguiremos dos preguntas: si varios intentos contienen alguna solución correcta y si todos los intentos cumplen el criterio. Ninguna de las dos demuestra que el agente funcione siempre en cualquier situación.

## Capacidad con pass@k

**pass@k** representa la probabilidad de que al menos uno de k intentos sea exitoso bajo el protocolo. Es útil para estudiar búsqueda de soluciones, pero no demuestra que el sistema sepa reconocer cuál candidato es correcto.

El trabajo **Evaluating Large Language Models Trained on Code**, publicado el 7 de julio de 2021, utiliza n muestras por tarea, c muestras correctas y 1 ≤ k ≤ n. Su estimador combinatorio es:

**pass@k estimado = 1 − C(n − c, k) / C(n, k)**.

C(a, k) cuenta las formas de elegir k elementos entre a; vale cero si a es menor que k. Para k = 1, la expresión da c/n. No debe sustituirse sin más por una potencia de esa fracción: el artículo explica el sesgo de esa alternativa. [Publicación original](https://arxiv.org/abs/2107.03374).

Como ejemplo propio, supongamos cuatro intentos con resultados éxito, éxito, fallo y fallo. Hay seis parejas posibles; cinco contienen al menos un éxito. El estimador de pass@2 es **5/6**, aproximadamente 0,833. No es una medición de nuestro agente: es un cálculo sobre datos ilustrativos.

Si en producción generamos dos respuestas y elegimos una con un juez, evaluaremos el resultado de esa selección y su costo. No presentaremos el pass@2 calculado con un verificador que conoce la respuesta como si fuera el desempeño del sistema que elige sin ella.

## Consistencia con pass^k

**pass^k** representa la probabilidad de que los k intentos sean exitosos. τ-bench introduce esta medida para observar consistencia en interacciones repetidas y utiliza el estimador:

**pass^k estimado = C(c, k) / C(n, k)**.

Los resultados se agregan según la distribución de tareas o dominios del protocolo. Debemos declarar esos pesos. [Fuente primaria](https://arxiv.org/abs/2406.12045).

En el mismo ejemplo de cuatro intentos, solo una de las seis parejas contiene dos éxitos. Por tanto, pass^2 estimado es **1/6**, aproximadamente 0,167; pass@1 es **2/4 = 0,5**. La diferencia muestra dos preguntas distintas sobre los mismos datos.

Si conociéramos una probabilidad p constante y tuviéramos intentos independientes con la misma distribución, la probabilidad de todos los éxitos sería p elevado a k. Ese valor teórico no es el mismo cálculo que reemplazar p por cualquier fracción observada. Los estimadores combinatorios consideran subconjuntos de la muestra recogida.

Repetir una acción sobre un estado que ya cambió rompe la comparación con ensayos reiniciados. Por eso un protocolo de fiabilidad debe especificar qué se restaura y qué variaciones admite entre intentos.

## Incertidumbre y comparación entre versiones

Un número pequeño de muestras produce estimaciones variables. No hay una cantidad universal que garantice fiabilidad: depende de la diversidad de tareas, la precisión necesaria y el costo de equivocarse. Más muestras no implican que una estimación necesariamente disminuya; puede cambiar en cualquier dirección.

Para nuestro agente informaremos tareas, ensayos por tarea y resultados por caso, además del promedio. Si las tareas tienen distinto número de intentos, no agruparemos todos los aciertos sin explicar cómo cambia el peso de cada tarea.

Compararemos versiones sobre los mismos casos y condiciones. Al calcular intervalos de incertidumbre, respetaremos la agrupación por tarea; las repeticiones de un mismo caso no son tareas nuevas. Declararemos el método utilizado y no trataremos una diferencia pequeña como una mejora confirmada por el solo hecho de que el decimal sea mayor.

Tampoco utilizaremos pass@k como promesa de que entrenar más resolverá los fallos. Detecta oportunidades de éxito bajo un muestreo; para adoptar una versión necesitamos validar el sistema que realmente ejecutaremos.

## Infraestructura como parte del experimento

Los recursos disponibles pueden cambiar qué estrategias logra ejecutar el agente. En **Quantifying infrastructure noise in agentic coding evals**, publicado el 5 de febrero de 2026, Anthropic varió recursos manteniendo otros componentes fijos y observó diferencias en resultados y fallos de infraestructura. Son resultados de sus configuraciones; no trasladaremos sus magnitudes a nuestro sistema. [Fuente primaria](https://www.anthropic.com/research/infrastructure-noise).

Para nuestros ensayos fijaremos y registraremos límites, concurrencia, versiones de servicios y condiciones de red cuando sean pertinentes. Separaremos un fallo de la herramienta causado por argumentos inválidos de una indisponibilidad del servicio o un contenedor que no pudo iniciar.

Esa separación no autoriza descartar resultados incómodos. Definiremos antes qué cuenta como intento, cuándo se repite un caso inválido y cómo se informa su costo. Mantendremos visibles los errores de infraestructura junto a los resultados del agente para que el denominador sea auditable.

# Evaluación de seguridad

La **evaluación de seguridad (safety and security evaluation)** comprueba tanto acciones dañinas como vulnerabilidades frente a manipulación. Definiremos quién puede causar el problema, qué recurso está expuesto y qué comportamiento esperamos del sistema.

## Uso malicioso solicitado por una persona

Un caso de **mal uso (misuse)** parte de una solicitud dañina. Debemos observar si el agente evita la acción, no solo si escribe una negativa. Una respuesta de rechazo seguida de una llamada dañina no satisface la restricción.

**AgentHarm**, publicado el 11 de octubre de 2024, propone tareas para estudiar comportamiento dañino de agentes con herramientas. Es una referencia experimental para construir pruebas de este tipo; sus resultados no sustituyen un análisis del uso previsto de nuestro agente. [Publicación original](https://arxiv.org/abs/2410.09024).

Mediremos también rechazos incorrectos sobre tareas legítimas. Un sistema que se niega a todo puede impedir acciones peligrosas, pero no resuelve la necesidad del producto.

## Inyección de instrucciones y memoria contaminada

Una **inyección de instrucciones (prompt injection)** intenta convertir contenido no confiable en instrucciones para el agente. Puede aparecer en una página, un documento o un resultado de herramienta. El **envenenamiento de memoria (memory poisoning)** busca que información manipulada afecte decisiones posteriores mediante lo guardado o recuperado.

**AgentDojo**, publicado el 19 de junio de 2024, estudia ataques y defensas sobre datos externos y permite evaluar la tarea legítima además del objetivo del atacante. **Agent Security Bench (ASB)**, publicado el 3 de octubre de 2024, incluye varias superficies, entre ellas ataques sobre memoria. Son entornos de investigación: superar sus casos no demuestra resistencia a todos los ataques. [AgentDojo](https://arxiv.org/abs/2406.13352), [ASB](https://arxiv.org/abs/2410.02644).

En nuestro caso ilustrativo, un registro podría contener texto que pida ignorar el filtro y modificar otro proyecto. Comprobaremos que se trate como contenido, que no provoque una escritura y que el agente complete la consulta legítima cuando sea posible. Los datos de prueba y las herramientas serán controlados; no utilizaremos servicios reales para provocar daños.

Informaremos por separado el cumplimiento de la tarea legítima, las acciones prohibidas solicitadas, las que el ejecutor bloqueó y los efectos producidos. Una defensa del ejecutor puede evitar un daño aunque el modelo haya intentado la acción; ambas observaciones son importantes para mejorar el sistema.

## Errores sin adversario

Un agente puede causar un daño durante una tarea legítima: modificar el archivo equivocado, duplicar una operación o informar una consulta incompleta como definitiva. Estos casos requieren pruebas de permisos, alcance, confirmaciones y recuperación.

Para nuestro agente incluiremos cancelación, respuestas pendientes, herramientas que fallan y efectos inciertos tras un tiempo de espera agotado. Comprobaremos que no declare éxito sin evidencia y que no repita automáticamente operaciones que puedan duplicar cambios.

No reduciremos toda la seguridad a una tasa de rechazo ni a un promedio de calidad. Las violaciones críticas se informarán por separado. El contrato del ejecutor debe hacer cumplir las restricciones que no podemos dejar solo a una instrucción.

# Construir nuestras propias evaluaciones

Nuestro conjunto debe representar el trabajo del agente y sus fallos importantes. Puede comenzar pequeño para acelerar el aprendizaje, pero una colección de pocos casos no demuestra cobertura amplia ni una tasa de fallos rara.

## Elegir casos y separar conjuntos

Para el filtro por fecha, proponemos estas familias de casos propios:

| Familia | Qué comprobaríamos |
|---|---|
| Caso normal | Devuelve exactamente B, C y D bajo el contrato UTC |
| Límites temporales | Incluye el inicio y excluye el comienzo del día siguiente |
| Información faltante | Pide la zona horaria si no está definida ni puede inferirse con autorización |
| Resultado vacío | Distingue ausencia de registros de un error de consulta |
| Error o trabajo pendiente | Comunica el estado sin inventar datos ni declarar finalización |
| Reintentos | Respeta los límites y evita efectos duplicados |
| Datos manipulados | No convierte contenido recuperado en una nueva autorización |
| Recuperación entre sesiones | Retoma el objetivo y vuelve a comprobar el estado necesario |

Los casos deben especificar salidas permitidas y restricciones, no exigir la misma redacción. Incluiremos escenarios donde detenerse o pedir información es el comportamiento correcto.

Mantendremos un **conjunto de desarrollo (development set)** para diagnosticar y ajustar, y un **conjunto reservado (held-out set)** para comprobar generalización antes de adoptar cambios. Convertir fallos reales en casos es útil, pero usaremos datos autorizados, minimizados y sin secretos. Si un caso reservado se utiliza repetidamente para optimizar, deja de ser una comprobación independiente.

## Registrar configuración y artefactos

Cada ejecución conservará identificador, versión del caso, estado inicial y configuración del sistema. Guardaremos el resultado, las puntuaciones por criterio, el estado final y los errores de ejecución o calificación. La traza incluirá identificadores que permitan asociar solicitudes y resultados.

Versionaremos instrucciones, herramientas, modelo y evaluadores. También registraremos presupuesto, reintentos y fuentes de variación controlables. Una semilla puede ayudar cuando está disponible, pero no garantiza por sí sola reproducción idéntica de servicios externos.

El verificador debe permanecer separado de los artefactos que el agente puede modificar. Si el agente cambia sus propios tests y solo ejecutamos esos tests, puede alterar la regla de aprobación. Conservaremos comprobaciones independientes y protegeremos los criterios reservados.

La **infraestructura de evaluación (evaluation harness)** prepara entornos, ejecuta ensayos, recoge artefactos y aplica verificadores. **Harbor** ofrece conceptos de tarea, entorno aislado, verificador y ensayo; su documentación define: “A trial is one agent’s attempt at completing one task”. Es una opción de infraestructura, no una dependencia obligatoria del agente. [Documentación oficial](https://docs.harborframework.com/core-concepts).

## Incorporar evaluaciones al desarrollo

Antes de cambiar un componente, ejecutaremos la versión actual como línea base bajo el protocolo elegido. Después compararemos el cambio sobre los mismos casos. Un reporte útil incluirá regresiones concretas, variación, costo y cumplimiento de restricciones, no solo el promedio final.

Automatizar la ejecución ante cambios permite obtener retroalimentación sin evaluarlo todo a mano. Para empezar, seleccionaremos casos breves y representativos; las evaluaciones costosas podrán ejecutarse en momentos definidos. Esta propuesta no requiere un proceso permanente ni tareas programadas sin autorización.

Las pruebas de componentes, la evaluación del agente completo y la observación en uso real se complementan. Las primeras permiten localizar errores; la evaluación completa comprueba interacciones; los incidentes de uso muestran condiciones no previstas. Si incorporamos observación de producción, definiremos acceso, privacidad y tratamiento de datos antes de recopilarla.

## Criterios para adoptar una versión

Acordaremos antes de comparar qué mejoras buscamos y qué regresiones son inaceptables. Por ejemplo, reducir llamadas puede ser deseable solo si mantiene los resultados y las restricciones. No elegiremos umbrales después de ver la puntuación para justificar una versión favorita.

Para una decisión presentaremos:

- Qué cambió y sobre qué versión se comparó.
- Qué casos mejoraron, empeoraron o siguen sin resolverse.
- Qué costo y variabilidad observamos bajo las condiciones registradas.
- Qué violaciones de permisos o efectos no deseados aparecieron.
- Qué evidencia falta para aceptar el alcance de uso propuesto.

Si cambia el evaluador, volveremos a calificar las versiones comparadas cuando sea posible y registraremos ese cambio de criterio. No mezclaremos una mejora del agente con una corrección de la medición.

# Lo que queda definido para nuestro agente

El capítulo deja un contrato de evaluación: casos con estado inicial, ejecución aislada, trazas observables, verificadores independientes y resultados separados por dimensión. Todavía debemos implementar ese circuito y ejecutar los casos propuestos.

El primer conjunto puede utilizar los registros sintéticos del filtro para comprobar datos y límites, seguido de errores de herramientas y restricciones de escritura. Evaluaremos por separado si la implementación de una herramienta funciona, si el modelo la solicita correctamente y si el sistema completo resuelve la necesidad.

No necesitamos reproducir todos los benchmarks públicos para empezar. Los utilizaremos cuando su protocolo ayude a una pregunta concreta y conservaremos la versión y las condiciones del experimento.

# Resumen

Evaluar un agente requiere mirar el resultado, la trayectoria, la consistencia y los efectos. Las comprobaciones programáticas, la revisión humana y los jueces basados en modelos cubren aspectos distintos y también pueden fallar.

pass@k estudia si varios intentos contienen algún éxito; pass^k estudia si todos cumplen. Sus estimaciones dependen de tareas, repeticiones y protocolo. Ninguna convierte un ejemplo correcto en una garantía de producción.

Para nuestro agente construiremos casos propios, separaremos preparación, ejecución y calificación, y compararemos cambios con una línea base reproducible. Mantendremos visibles los errores de infraestructura y las restricciones incumplidas, aunque el promedio mejore.

En el próximo capítulo abordaremos sistemas multiagente. Las mismas reglas de evaluación seguirán aplicando al sistema completo y a las interacciones entre sus participantes.

# Referencias y alcance de las fuentes

Fecha de referencia: **3 de noviembre de 2026**. La ventana de novedades revisada abarca del **3 de noviembre de 2023 al 3 de noviembre de 2026**. Las primeras publicaciones anteriores siguen siendo útiles como fundamentos; una revisión reciente no cambia su fecha de origen. La fecha de consulta de una documentación o repositorio no se presenta como fecha de publicación.

Los benchmarks y marcos de investigación describen tareas y métodos bajo condiciones específicas. Su publicación reciente no implica adopción consolidada ni superioridad universal. Las decisiones para nuestro agente, los datos sintéticos y los criterios propuestos se identifican como aportaciones propias; no son resultados experimentales reproducidos en esta revisión.

## Fundamentos y benchmarks

- Chen, Mark et al. [Evaluating Large Language Models Trained on Code](https://arxiv.org/abs/2107.03374). Primera publicación: 7 de julio de 2021. Estimador de pass@k, anterior a la ventana.
- Jimenez, Carlos E. et al. [SWE-bench: Can Language Models Resolve Real-World GitHub Issues?](https://arxiv.org/abs/2310.06770). Primera publicación: 10 de octubre de 2023; revisión de noviembre de 2024. Origen anterior a la ventana.
- SWE-bench. [FAQ oficial](https://www.swebench.com/SWE-bench/faq/). Consultada el 3 de noviembre de 2026; sin fecha editorial verificada. Variantes y subconjunto Verified.
- SWE-bench. [Código oficial de calificación](https://raw.githubusercontent.com/SWE-bench/SWE-bench/main/swebench/harness/grading.py). Consultado el 3 de noviembre de 2026; implementación vigente en la rama consultada, no una versión fijada para reproducir un experimento.
- Terminal-Bench. [Terminal-Bench 4.0](https://www.tbench.ai/news/terminal-bench-4-0). Publicación fechada el 28 de agosto de 2026 en el índice oficial de noticias. Cambios en recursos, tareas y necesidad de nuevas ejecuciones.
- Equipo BFCL. [BFCL V4: Web Search](https://gorilla.cs.berkeley.edu/blogs/15_bfcl_v4_web_search.html). Publicado el 17 de julio de 2025. Evolución de evaluación de herramientas hacia tareas de búsqueda.
- Yao, Shunyu et al. [τ-bench: A Benchmark for Tool-Agent-User Interaction in Real-World Domains](https://arxiv.org/abs/2406.12045). Primera publicación: 17 de junio de 2024. Conversaciones con herramientas, comprobación de estado y pass^k.
- Xie, Tianbao et al. [OSWorld: Benchmarking Multimodal Agents for Open-Ended Tasks in Real Computer Environments](https://arxiv.org/abs/2404.07972). Primera publicación: 11 de abril de 2024.
- Zhou, Shuyan et al. [WebArena: A Realistic Web Environment for Building Autonomous Agents](https://arxiv.org/abs/2307.13854). Primera publicación: 25 de julio de 2023; revisión de abril de 2024. Antecedente anterior a la ventana.
- Mialon, Grégoire et al. [GAIA: a benchmark for General AI Assistants](https://arxiv.org/abs/2311.12983). Primera publicación: 21 de noviembre de 2023.
- OpenAI. [Measuring the performance of our models on real-world tasks](https://openai.com/index/gdpval/). Presentación de GDPval publicada el 25 de septiembre de 2025. No se presupone que sus servicios públicos históricos continúen disponibles.
- Hendrycks, Dan et al. [Measuring Massive Multitask Language Understanding](https://arxiv.org/abs/2009.03300). Primera publicación: 7 de septiembre de 2020. MMLU, anterior a la ventana.
- Rein, David et al. [GPQA: A Graduate-Level Google-Proof Q&A Benchmark](https://arxiv.org/abs/2311.12022). Primera publicación: 20 de noviembre de 2023.
- Phan, Long et al. [Humanity’s Last Exam](https://arxiv.org/abs/2501.14249). Primera publicación: 24 de enero de 2025; revisión consultada: 19 de octubre de 2026.
- Chollet, François. [Repositorio ARC-AGI](https://github.com/fchollet/ARC-AGI). Consultado el 3 de noviembre de 2026. Fuente de tareas y alcance del corpus; no se presenta la fecha de consulta como una novedad.
- Yue, Xiang et al. [MMMU: A Massive Multi-discipline Multimodal Understanding and Reasoning Benchmark for Expert AGI](https://arxiv.org/abs/2311.16502). Primera publicación: 27 de noviembre de 2023.
- Kamradt, Greg. [Needle in a Haystack](https://github.com/gkamradt/needle-in-a-haystack). Repositorio del autor consultado el 3 de noviembre de 2026; sin fecha de primera publicación verificada en esta revisión.

## Métodos, trayectoria e interpretación

- Zheng, Lianmin et al. [Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena](https://arxiv.org/abs/2306.05685). Primera publicación: 9 de junio de 2023; revisión de diciembre de 2023. Antecedente de jueces y preferencias anterior a la ventana.
- Zhou, Jeffrey et al. [Instruction-Following Evaluation for Large Language Models](https://arxiv.org/abs/2311.07911). Primera publicación: 14 de noviembre de 2023. IFEval.
- Singh, Shivalika et al. [The Leaderboard Illusion](https://arxiv.org/abs/2504.20879). Primera publicación: 29 de abril de 2025. Análisis de selección y divulgación en Arena.
- Wang, Zora Zhiruo et al. [How Well Does Agent Development Reflect Real-World Work?](https://arxiv.org/abs/2603.01203). Primera publicación: 1 de marzo de 2026. Representatividad ocupacional de los benchmarks estudiados.
- Raghavendra, Mohit et al. [SWE Atlas: Benchmarking Coding Agents Beyond Issue Resolution](https://arxiv.org/abs/2605.08366). Primera publicación: 8 de mayo de 2026. Preguntas sobre código, tests y refactorización.
- Chen, Zehui et al. [T-Eval: Evaluating the Tool Utilization Capability of Large Language Models Step by Step](https://arxiv.org/abs/2312.14033). Primera publicación: 21 de diciembre de 2023. Evaluación por componentes.
- Ma, Chang et al. [AgentBoard: An Analytical Evaluation Board of Multi-turn LLM Agents](https://arxiv.org/abs/2401.13178). Primera publicación: 24 de enero de 2024. Evaluación analítica del progreso.
- Kim, Wonjoong et al. [Beyond the Final Answer: Evaluating the Reasoning Trajectories of Tool-Augmented Agents](https://arxiv.org/abs/2510.02837). Primera publicación: 3 de octubre de 2025; revisión consultada: 25 de mayo de 2026. TRACE.
- Fan, Shengda et al. [AgentProcessBench: Diagnosing Step-Level Process Quality in Tool-Using Agents](https://arxiv.org/abs/2603.14465). Primera publicación: 15 de marzo de 2026; revisión consultada: 1 de junio de 2026.
- Anthropic. [Demystifying evals for AI agents](https://www.anthropic.com/research/demystifying-evals-for-ai-agents). Publicado el 9 de enero de 2026. Guía de ingeniería, con distinción entre registro y estado final.
- Anthropic. [Quantifying infrastructure noise in agentic coding evals](https://www.anthropic.com/research/infrastructure-noise). Publicado el 5 de febrero de 2026. Experimentos del proveedor sobre recursos e infraestructura.
- Harbor. [Core concepts](https://docs.harborframework.com/core-concepts). Documentación consultada el 3 de noviembre de 2026; sin fecha editorial verificada. Tareas, entornos, verificadores y ensayos.

## Seguridad

- Andriushchenko, Maksym et al. [AgentHarm: A Benchmark for Measuring Harmfulness of LLM Agents](https://arxiv.org/abs/2410.09024). Primera publicación: 11 de octubre de 2024.
- Debenedetti, Edoardo et al. [AgentDojo: A Dynamic Environment to Evaluate Prompt Injection Attacks and Defenses for LLM Agents](https://arxiv.org/abs/2406.13352). Primera publicación: 19 de junio de 2024.
- Zhang, Hanrong et al. [Agent Security Bench (ASB): Formalizing and Benchmarking Attacks and Defenses in LLM-based Agents](https://arxiv.org/abs/2410.02644). Primera publicación: 3 de octubre de 2024; revisión consultada: 30 de mayo de 2025.
