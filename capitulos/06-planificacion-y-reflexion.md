# Capítulo 6. Planificación y reflexión

La **planificación (planning)** permite organizar el trabajo que nuestro agente necesita realizar para alcanzar un objetivo. La **reflexión (reflection)** permite revisar sus resultados y decidir si conviene corregir una acción, modificar el plan o solicitar ayuda. Ambas capacidades resultan útiles cuando una tarea requiere varios pasos y la información disponible cambia durante la ejecución.

Planificar no consiste únicamente en escribir una lista. También implica reconocer qué pasos dependen de otros, qué información falta y cómo comprobaremos que el objetivo se ha cumplido. Un plan puede elaborarse antes de actuar o construirse progresivamente a partir de los resultados obtenidos.

Consideremos un **ejemplo ilustrativo**, sin ejecutar ninguna operación: una persona pide añadir un filtro por fecha a una aplicación. El agente podría comenzar por revisar los requisitos y el código existente, definir el comportamiento esperado, escribir una prueba e implementar el cambio. Si la prueba falla, necesitará examinar el resultado antes de decidir qué corregir.

En ese recorrido debemos distinguir tres eventos:

1. **Solicitud del modelo:** pide ejecutar la herramienta de pruebas.
2. **Ejecución de la herramienta:** el programa encargado de las herramientas ejecuta las pruebas.
3. **Resultado observado:** devuelve los casos aprobados, los fallos o un error de ejecución.

Escribir «las pruebas pasan» no demuestra que se hayan ejecutado. Tampoco basta con que pasen para afirmar que la funcionalidad satisface todos los requisitos. Necesitamos comprobar el comportamiento que pidió la persona. La guía de evaluación de Anthropic distingue el registro de la interacción del resultado alcanzado en el entorno: “The outcome is the final state in the environment”. [Fuente primaria, 9 de enero de 2026](https://www.anthropic.com/research/demystifying-evals-for-ai-agents).

**El plan debe poder cambiar cuando aparece evidencia nueva.** Si descubrimos que la aplicación guarda fechas en distintas zonas horarias, quizá debamos aclarar qué significa filtrar por un día antes de continuar. Actualizar el plan permite responder a ese hallazgo sin perder el objetivo original.

No todos los agentes necesitan un plan completo por adelantado. Algunas tareas pueden resolverse eligiendo una acción, observando su resultado y decidiendo el siguiente paso. *ReAct* estudió esta combinación de razonamiento expresado en texto, acciones y observaciones. Conservamos esa conexión entre actuar y recibir información del entorno, sin tratar el texto generado como acceso al pensamiento interno del modelo. [Publicación original, 2022](https://arxiv.org/abs/2210.03629).

La reflexión añade una revisión de lo ocurrido: qué resultado esperábamos, qué evidencia recibimos y qué conviene intentar después. **Una crítica generada por el modelo no demuestra que haya detectado correctamente el problema.** Siempre que sea posible, la contrastaremos con pruebas, consultas al entorno o revisión humana.

Para construir nuestro agente, mantendremos un estado explícito del trabajo: pasos pendientes, acciones realizadas, resultados comprobados y bloqueos. En tareas que atraviesen varias sesiones, conservaremos esa información para poder retomarlas. Los registros persistentes de avance son una práctica descrita en experiencias recientes de ingeniería con agentes de larga duración; su formato dependerá de nuestra aplicación. [Fuente primaria, 26 de noviembre de 2025](https://www.anthropic.com/research/effective-harnesses-for-long-running-agents).

También definiremos cuándo detenerse: al verificar el objetivo, alcanzar un límite de ejecución o encontrar un bloqueo que requiera intervención. Repetir acciones no garantiza progresar. La guía *Building effective agents* recomienda incorporar condiciones de parada para mantener el control del proceso. [Fuente primaria, 19 de diciembre de 2024](https://www.anthropic.com/research/building-effective-agents).

En este capítulo construiremos ese circuito de planificación, ejecución, observación y revisión. Distinguiremos los cambios de plan durante una tarea de los métodos de entrenamiento que modifican los parámetros del modelo: **revisar un resultado durante la ejecución no equivale, por sí mismo, a entrenar al agente**.

# Planificación

La **planificación (planning)** ayuda a nuestro agente a decidir qué acciones necesita realizar, en qué orden y bajo qué condiciones para alcanzar un objetivo. Resulta útil cuando una acción depende del resultado de otra o cuando debemos coordinar varios pasos.

Conviene distinguir el **objetivo**, que describe el resultado buscado, del **plan**, que propone cómo alcanzarlo. Por ejemplo, «añadir un filtro por fecha» es un objetivo. Localizar dónde se consultan los datos, comprobar cómo se representan las fechas y definir el comportamiento del filtro son pasos de un posible plan. Este es un **ejemplo ilustrativo**: todavía no se ha consultado ni modificado ninguna aplicación.

El plan puede seguir una secuencia definida por el desarrollador o incorporar decisiones del modelo durante la ejecución. También podemos combinar ambas opciones: permitir que el modelo proponga el siguiente paso y mantener en el código las reglas de autorización y los límites. *Building effective agents* distingue estos patrones como **flujos de trabajo (workflows)** y **agentes (agents)**. [Fuente primaria, 19 de diciembre de 2024](https://www.anthropic.com/research/building-effective-agents).

**Un plan generado es una propuesta que debemos contrastar con el entorno.** Puede omitir una dependencia, suponer que existe una herramienta o incluir una operación que no está permitida. Describir una acción tampoco significa haberla solicitado ni ejecutado.

Para nuestro agente, comenzaremos por dividir el objetivo en tareas manejables. Después veremos cómo ordenar las acciones y revisar lo que queda por hacer cuando recibimos nuevos resultados.

## Descomposición de tareas

La **descomposición de tareas (task decomposition)** permite convertir un objetivo amplio en unidades cuyo avance podamos comprobar. No todos los objetivos necesitan subdividirse: una consulta sencilla puede resolverse con una sola operación. Descomponer resulta útil cuando hay incertidumbre, dependencias o resultados intermedios que conviene verificar.

En nuestro ejemplo del filtro por fecha, una primera descomposición podría ser:

| Tarea | Información necesaria | Evidencia para completarla |
|---|---|---|
| Precisar el comportamiento | Requisitos y zona horaria del producto | Regla acordada para los límites de fecha |
| Revisar la consulta existente | Código y contrato de los datos | Ubicación de la consulta y representación de las fechas |
| Escribir una prueba del filtro | Regla acordada y entorno de pruebas | Caso reproducible que detecte la ausencia del comportamiento |
| Implementar el filtro | Consulta identificada y prueba | Cambio que satisface el caso definido |
| Verificar la integración | Aplicación con el cambio | Resultado observado al usar el filtro y pruebas de regresión |

La tabla es una **propuesta didáctica propia**, no un registro de trabajo ejecutado. Algunas tareas pueden dividirse más; otras pueden desaparecer si descubrimos que el sistema ya ofrece esa capacidad.

Una subtarea no equivale necesariamente a una llamada a herramienta. «Revisar la consulta» puede requerir buscar archivos, leer varios fragmentos y comparar contratos. A la inversa, una herramienta puede resolver varias operaciones internas mediante una sola solicitud.

Para nuestro agente, describiremos cada tarea mediante un resultado esperado y sus dependencias. Evitaremos pasos vagos como «mejorar todo el sistema», porque no permiten decidir si terminamos. También evitaremos descomponer tanto que la administración de la lista cueste más que resolver el objetivo.

### Cadena de pensamiento y alternativas

La **cadena de pensamiento (chain-of-thought, CoT)** puede favorecer la resolución de ciertos problemas mediante pasos intermedios expresados en texto. El trabajo original de Wei y colaboradores, publicado el 28 de enero de 2022, estudió ejemplos de resolución incluidos en las instrucciones. Sus resultados corresponden a los modelos y tareas evaluados. [Publicación original](https://arxiv.org/abs/2201.11903).

**Una secuencia de pasos explicativos no equivale a un plan operativo.** Para calcular el costo de tres entradas de 12 unidades y una comisión de 4, una explicación puede mostrar 3 × 12 = 36 y 36 + 4 = 40. Un plan del agente debe especificar, cuando corresponda, qué operación solicitar, con qué datos y cómo comprobar su resultado. El cálculo es un ejemplo propio, comprobable, sin ejecución de herramientas.

Tampoco interpretaremos una explicación generada como una transcripción del pensamiento interno. En experimentos publicados por Anthropic el 3 de abril de 2025, las explicaciones de Claude 3.7 Sonnet y DeepSeek R1 no siempre reconocían pistas que habían influido en sus respuestas. Esto limita la confianza que podemos depositar en esas explicaciones como mecanismo de auditoría. [Investigación primaria](https://www.anthropic.com/research/reasoning-models-dont-say-think).

Dos técnicas históricas amplían la generación de candidatos:

- **Autoconsistencia (self-consistency).** Genera varias resoluciones y agrega sus respuestas finales para elegir la más consistente. El trabajo de 2022 no selecciona una trayectoria óptima: utiliza el acuerdo entre respuestas como señal. Puede aumentar el costo y mantener un error compartido. [Publicación original](https://arxiv.org/abs/2203.11171).
- **Árbol de pensamientos (Tree of Thoughts, ToT).** Propone y evalúa pasos intermedios dentro de una búsqueda, con posibilidad de explorar alternativas o retroceder. El trabajo de 2023 lo evaluó en tareas delimitadas; no demuestra que buscar más ramas sea siempre mejor para un agente. La evaluación de los candidatos también puede equivocarse. [Publicación original](https://arxiv.org/abs/2305.10601).

Conservamos estas técnicas por sus distinciones conceptuales. No las incorporaremos automáticamente a cada tarea ni asumiremos que una instrucción de «pensar paso a paso» mejora cualquier modelo. En nuestro primer agente pediremos un plan breve y comprobable; compararemos estrategias cuando dispongamos de tareas representativas y un presupuesto de evaluación.

### Planificación explícita

Un **plan explícito (explicit plan)** hace visible la propuesta de trabajo para que la aplicación o una persona pueda revisarla. Conviene cuando necesitamos coordinar dependencias, acordar alcance o retomar una tarea. Tener un plan visible no obliga a resolver todo en una sola llamada al modelo.

Dos antecedentes ayudan a entender esta separación:

- **De menor a mayor (least-to-most prompting, LtM).** Primero descompone el problema; después resuelve los subproblemas en secuencia utilizando las respuestas anteriores. La publicación de 2022 estudió manipulación simbólica, generalización composicional y razonamiento matemático. Su aporte es separar la descomposición de la resolución; no demuestra por sí mismo ejecución fiable sobre sistemas externos. [Fuente primaria](https://arxiv.org/abs/2205.10625).
- **Planificar y resolver (Plan-and-Solve prompting, PS).** La propuesta de 2023 pide elaborar un plan y resolver según ese plan, sin aportar demostraciones de la tarea. PS+ añade instrucciones más detalladas. Se evaluó con GPT-3 en conjuntos de razonamiento, por lo que sus resultados no equivalen a una validación de agentes actuales con herramientas. [Fuente primaria](https://arxiv.org/abs/2305.04091).

Podemos conservar esa separación sin copiar sus instrucciones literalmente. Para el filtro, el modelo podría proponer primero averiguar la zona horaria y localizar la consulta. Solo después de esas observaciones tendría sentido detallar la implementación.

Un plan inicial demasiado preciso puede apoyarse en supuestos falsos. En nuestro agente detallaremos el trabajo inmediato y mantendremos provisional lo que dependa de información todavía no obtenida. Cambiar el plan deberá tener una causa identificable: un requisito aclarado, un resultado inesperado o una dependencia descubierta.

### Dependencias y validación del plan

Validar un plan permite detectar pasos imposibles o incompatibles antes de ejecutarlos. Distinguiremos las **precondiciones (preconditions)**, que deben cumplirse para una acción, de sus **efectos esperados (expected effects)**, que después habrá que comprobar.

En el ejemplo del filtro, no podemos verificar su integración sin disponer de una versión ejecutable del cambio. Podemos leer documentación y código de manera independiente si ambas consultas no necesitan los resultados de la otra. Modificar la misma consulta desde dos acciones concurrentes, en cambio, puede introducir conflictos.

La propuesta **LLM-Modulo**, publicada inicialmente el 2 de febrero de 2024, combina candidatos generados por un LLM con verificadores externos basados en modelos del dominio. Es una propuesta de arquitectura de investigación, no un requisito para todo agente ni una prueba de imposibilidad para cualquier modelo futuro. Su lección útil es separar proponer de verificar. [Artículo de posición y publicación en ICML 2024](https://arxiv.org/abs/2402.01817).

Para nuestro agente empezaremos con comprobaciones sencillas: herramientas existentes, argumentos válidos, dependencias satisfechas y permisos disponibles. Si más adelante resolvemos problemas con restricciones formales, como horarios o asignaciones, podremos delegar esas restricciones a un solucionador específico. Un verificador solo garantiza las propiedades que realmente comprueba bajo sus supuestos.

## Secuenciación de acciones

La **secuenciación de acciones (action sequencing)** determina qué operación realizar a continuación. Una lista de objetivos no define por sí sola esa secuencia: necesitamos considerar lo ya observado, los errores y las restricciones de ejecución.

Después de cada resultado, nuestro agente podrá continuar con una tarea preparada, revisar el plan, pedir información o finalizar. Esta decisión no requiere que el modelo vuelva a generar toda la lista en cada turno. La aplicación puede mantener el estado y proporcionarle el contexto pertinente.

### Razonar y actuar con ReAct

**ReAct (Reasoning and Acting)** permite intercalar texto de razonamiento, acciones y observaciones para adaptar la resolución a la información que devuelve el entorno. El trabajo de Yao y colaboradores se publicó inicialmente el 6 de octubre de 2022 y se presentó en ICLR 2023. Es un fundamento histórico útil, no el origen de toda planificación autónoma. [Publicación original](https://arxiv.org/abs/2210.03629).

En su formulación, los pensamientos son texto generado; las acciones se dirigen al entorno y las observaciones provienen de la interacción. **El modelo no debe inventar el resultado de una herramienta y tratarlo como una observación real.** La aplicación incorpora el resultado recibido antes de solicitar la siguiente decisión.

Consideremos una **trayectoria ilustrativa (trajectory)** para resolver (8 + 6) × 3 − 2 con herramientas de cálculo:

| Paso | Solicitud del modelo | Ejecución y resultado ilustrativos |
|---|---|---|
| 1 | Sumar 8 y 6 | La herramienta devuelve 14 |
| 2 | Multiplicar 14 por 3 | La herramienta devuelve 42 |
| 3 | Restar 2 a 42 | La herramienta devuelve 40 |
| 4 | Responder con el resultado | La respuesta comunica 40 |

No se ejecutaron estas llamadas. La expresión permite comprobar el resultado de manera independiente. Si la multiplicación devolviera un error, no podríamos registrar 42 como resultado observado ni continuar con ese dato como si se hubiera recibido.

Para nuestro agente conservaremos el intercambio entre solicitudes y resultados, sin exigir que publique una explicación extensa en cada turno. Si necesitamos justificar una decisión, pediremos una explicación breve apoyada en evidencia; no la consideraremos acceso al pensamiento interno.

### Llamadas estructuradas y control del bucle

Las **llamadas estructuradas a herramientas (structured tool calls)** permiten recibir solicitudes en un formato definido por la integración. Así evitamos depender de extraer acciones de un texto libre. Esto no significa que la herramienta ya se haya ejecutado ni demuestra que el modelo haya sido entrenado con un algoritmo concreto como ReAct.

En las herramientas del cliente de Claude, la documentación distingue `tool_use`, con identificador, nombre y entrada, de `tool_result`, asociado a la solicitud mediante `tool_use_id`. El resultado puede comunicar un error con `is_error`. Son detalles de esa API; otras integraciones tienen sus propios contratos. [Documentación oficial, consultada el 3 de noviembre de 2026](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls).

Nuestro **bucle del agente (agent loop)** seguirá este contrato, como decisión de diseño:

1. Recibir el objetivo y recuperar el estado pertinente.
2. Pedir al modelo la siguiente respuesta o solicitud de acción.
3. Comprobar el nombre de la herramienta, sus argumentos, los permisos y las dependencias.
4. Ejecutar solo las operaciones autorizadas y registrar su estado.
5. Incorporar el resultado o error con el identificador de la solicitud.
6. Actualizar el avance y comprobar si corresponde continuar o detenerse.

Una respuesta sin llamadas a herramientas puede ser una respuesta final, una pregunta al usuario, una negativa o una salida incompleta. **La ausencia de llamadas no demuestra que el objetivo esté cumplido.** El sistema debe interpretar la respuesta y contrastarla con los criterios de finalización.

### Errores, reintentos y condiciones de parada

Controlar los errores permite que una dificultad local no se convierta en un ciclo de acciones repetidas. En nuestro diseño diferenciaremos un argumento inválido, una autorización denegada, un fallo transitorio y una operación cuyo estado todavía se desconoce.

Si una herramienta de escritura agota su tiempo de espera, quizá haya realizado el cambio antes de perderse la respuesta. Repetirla sin comprobar el estado puede duplicar efectos. Antes de reintentar, consultaremos el resultado cuando sea posible o utilizaremos una operación con protección frente a duplicados, según el contrato del servicio. No aplicaremos la misma política de reintentos a todas las herramientas.

Definiremos límites de llamadas, tiempo y costo, y una salida cuando falte información o autorización. Un límite alcanzado debe dejar un estado claro: qué se completó, qué quedó pendiente y por qué se detuvo. Una tarea bloqueada no se registrará como exitosa.

Estas son reglas que elegimos para nuestro agente. No las delegaremos exclusivamente a una instrucción que el modelo pueda incumplir. En el capítulo de evaluación construiremos casos para comprobar tanto el comportamiento normal como estos fallos.

### Estado persistente y recuperación

Guardar el **estado de ejecución (execution state)** permite retomar una tarea después de una interrupción o un cambio de contexto. No basta con conservar el plan inicial: necesitamos saber qué acciones se realizaron y qué evidencia respalda el avance.

Mantendremos un registro con el objetivo vigente, las tareas y sus dependencias, el último estado conocido de las operaciones y los resultados necesarios para continuar. Una tarea podrá estar pendiente, en curso, completada o bloqueada; el registro de sus llamadas distinguirá lo solicitado de lo ejecutado y lo observado.

La experiencia de Anthropic publicada el 26 de noviembre de 2025 describe progreso incremental y artefactos persistentes para enlazar sesiones. Su demostración está orientada a desarrollo web y no garantiza continuidad para cualquier tarea. [Fuente primaria](https://www.anthropic.com/research/effective-harnesses-for-long-running-agents).

Al retomar trabajo comprobaremos el entorno: otro proceso o una persona puede haber cambiado archivos o datos. Un resumen sirve para recuperar contexto; no sustituye la comprobación del estado actual. También limitaremos lo que guardamos a la información necesaria, sin copiar credenciales ni resultados sensibles completos a un registro general.

## Aprender a secuenciar acciones

Entrenar un modelo para generar mejores trayectorias puede reducir algunos errores de selección y ejecución propuesta. Es una intervención diferente de cambiar las instrucciones o actualizar el plan: **el entrenamiento modifica parámetros; el uso ordinario del contexto no los modifica por sí mismo**.

### Ajuste fino supervisado con FireAct

El **ajuste fino supervisado (supervised fine-tuning, SFT)** permite entrenar el modelo con demostraciones de comportamiento. En este contexto, los ejemplos pueden contener preguntas, solicitudes de herramientas y resultados de una trayectoria.

**FireAct**, publicado el 9 de octubre de 2023, estudió el ajuste con trayectorias de distintas tareas y métodos de indicación. Utilizó demostraciones generadas por un modelo más capaz para entrenar agentes y evaluar preguntas con búsqueda. Su utilidad histórica es mostrar que el comportamiento del agente puede aprenderse de trayectorias, sin depender únicamente de ejemplos incluidos en cada consulta. [Publicación original](https://arxiv.org/abs/2310.05915).

Imitar trayectorias exitosas no garantiza manejar situaciones que no aparecen en los datos. Una respuesta final correcta tampoco demuestra que todas las acciones de la demostración sean apropiadas. Si entrenáramos nuestro modelo, revisaríamos la calidad, permisos y cobertura de esas trayectorias. Para la primera versión utilizaremos un modelo compatible con herramientas y evaluaremos sus fallos antes de decidir si hace falta entrenamiento adicional.

### Optimización de trayectorias y preferencias

La **optimización de trayectorias basada en exploración (Exploration-Based Trajectory Optimization, ETO)** permite aprovechar errores obtenidos durante la interacción. La publicación del 4 de marzo de 2024 alterna exploración y entrenamiento: recoge trayectorias fallidas, construye pares de preferencias y actualiza el modelo para favorecer trayectorias exitosas. Es evidencia experimental en las tareas evaluadas. [Publicación original](https://arxiv.org/abs/2403.02502).

ETO utiliza **optimización directa de preferencias (Direct Preference Optimization, DPO)**. Conviene precisar el mecanismo: DPO entrena con pares preferido/no preferido mediante un objetivo de clasificación; no necesita el bucle de optimización de políticas con un modelo de recompensa separado característico de otros métodos de RLHF. Está relacionado con ese marco, pero describirlo simplemente como cualquier algoritmo de aprendizaje por refuerzo oculta la diferencia. [Trabajo original de DPO, 29 de mayo de 2023](https://arxiv.org/abs/2305.18290).

Para nuestro agente, guardar el error y su contexto será útil incluso sin entrenar. Podremos convertir un fallo reproducible en un caso de evaluación. Convertir después esos registros en datos de entrenamiento requerirá un proceso separado de selección y validación.

# Lo que deja definido este capítulo

Hasta aquí tenemos un contrato de construcción, no una implementación ejecutada en este texto. El agente necesita coordinar estas responsabilidades:

| Responsabilidad | Resultado que debe producir |
|---|---|
| Planificación | Tareas con dependencias y criterios de finalización |
| Ejecución | Solicitudes validadas y operaciones autorizadas |
| Registro | Asociación entre solicitud, ejecución, resultado y estado |
| Control | Límites, interrupciones y recuperación ante errores |
| Memoria | Estado pertinente para continuar entre turnos o sesiones |
| Evaluación | Evidencia de avance y de cumplimiento del objetivo |

No necesitamos una clase distinta para cada fila. Podemos empezar con una estructura sencilla y separar módulos cuando sus responsabilidades lo justifiquen. Tampoco supondremos que añadir un bucle convierte al sistema en fiable: los permisos, la observación y la comprobación del objetivo forman parte del comportamiento que debemos construir.

# Reflexión y mejora del agente

Revisar una solución puede servir para corregirla antes de entregarla o para evitar un error en el siguiente intento. Pero «mejora» puede significar cambios diferentes: corregir una salida, cambiar un plan, conservar una lección en memoria o actualizar los parámetros del modelo.

| Mecanismo | Qué cambia | Qué no demuestra por sí solo |
|---|---|---|
| Revisión de una respuesta | El contenido de esa respuesta | Mejora en tareas futuras |
| Replanificación | Las acciones pendientes | Que el objetivo se cumplirá |
| Memoria de experiencias | La información disponible en otros intentos | Que la lección sea correcta o generalizable |
| Entrenamiento | Los parámetros del modelo | Mejora en todos los dominios |

Esta distinción es una síntesis didáctica propia. La utilizaremos para elegir intervenciones y evaluar sus efectos por separado.

## Reflexión basada en evidencia

La **reflexión (reflection)** permite contrastar lo esperado con lo observado y proponer una corrección. La observación informa de lo ocurrido; la reflexión interpreta esa información. Confundirlas permite que una hipótesis del modelo termine registrada como un hecho.

En el filtro por fecha, una observación ilustrativa podría ser: «el caso que espera incluir el registro del día 10 devuelve cero registros». Una hipótesis sería: «puede haber un desfase de zona horaria». Para comprobarla, necesitamos revisar los datos y el cálculo de límites. La hipótesis no se convierte en causa por estar escrita con seguridad.

### Autorrefinamiento con Self-Refine

**Self-Refine** permite revisar una salida mediante un ciclo de generación, retroalimentación y refinamiento con el mismo modelo, sin entrenamiento adicional. El trabajo se publicó el 30 de marzo de 2023 y evaluó tareas diversas. Su aporte es separar la crítica de la nueva versión y utilizar criterios para dirigirla. [Publicación original](https://arxiv.org/abs/2303.17651).

Como ejemplo propio, podríamos pedir que una descripción del filtro indique qué ocurre cuando falta una fecha. La revisión detectaría esa omisión y una nueva versión aclararía el comportamiento. Este ejemplo ilustra el uso del método; no representa un experimento realizado ni un resultado garantizado.

**Más revisiones no implican más aciertos.** Un estudio sobre autocorrección intrínseca encontró que los modelos evaluados podían mantener errores o empeorar sin retroalimentación externa. Su primera publicación fue el 3 de octubre de 2023; la versión de marzo de 2024 es una revisión, no su fecha de origen. No generalizaremos esos resultados a todos los modelos actuales. [Estudio original](https://arxiv.org/abs/2310.01798).

En nuestro agente usaremos criterios concretos, un límite de revisiones y una comparación con la versión anterior. Si no aparece evidencia útil ni mejora comprobable, no mantendremos el ciclo solo porque el modelo pueda generar otra crítica.

### Memoria de experiencias con Reflexion

**Reflexion** permite usar retroalimentación de un intento para orientar el siguiente mediante memoria episódica. Su publicación inicial es del 20 de marzo de 2023. Los autores lo describen como aprendizaje mediante lenguaje, “not by updating weights”. [Publicación original](https://arxiv.org/abs/2303.11366).

El marco distingue un **actor (actor)** que realiza la tarea, un **evaluador (evaluator)** que produce retroalimentación y un componente de **autorreflexión (self-reflection)** que genera una lección para intentos posteriores. Son roles funcionales: no requieren tres modelos distintos. El evaluador puede utilizar reglas, pruebas o un LLM, según la tarea.

Para el filtro, una lección ilustrativa sería: «al comparar días de calendario, comprobar la zona horaria antes de convertir límites a UTC». Solo la conservaríamos si los resultados la respaldan. Una explicación equivocada almacenada puede perjudicar intentos futuros.

Conservaremos las lecciones con su contexto y evidencia, y las revisaremos cuando cambien las condiciones. La memoria modifica la información disponible, no los parámetros. Que el evaluador apruebe tampoco garantiza corrección; su criterio y sus fallos necesitan evaluación.

### Crítica asistida por herramientas con CRITIC

**CRITIC** permite revisar una salida utilizando información de herramientas externas. El trabajo se publicó inicialmente el 19 de mayo de 2023, con una revisión en febrero de 2024. Parte de una salida inicial, consulta herramientas para evaluar aspectos de ella y genera una corrección basada en esa retroalimentación. [Publicación original](https://arxiv.org/abs/2305.11738).

En nuestro ejemplo, un test puede comprobar qué registros devuelve el filtro. Una búsqueda de documentación puede aclarar la semántica de una función de fechas. Son evidencias distintas: la documentación explica un contrato; el test observa un comportamiento en un entorno concreto.

La selección de la herramienta y la interpretación de su salida también pueden fallar. Por eso asociaremos cada conclusión con la comprobación que la respalda. Una cita o un test aislado no validan automáticamente toda la solución.

### Separar producción y evaluación

Separar la creación de una solución de su evaluación puede ayudar cuando la autocrítica es demasiado indulgente. No exige necesariamente otro agente: una prueba determinista o una revisión humana pueden cumplir esa función. Si utilizamos un evaluador basado en modelos, definiremos una rúbrica y lo contrastaremos con ejemplos revisados.

Una experiencia publicada por Anthropic el 24 de marzo de 2026 describe una arquitectura con planificación, generación y evaluación para desarrollo de aplicaciones. Es una exploración de ingeniería con modelos y tareas concretos, no una arquitectura universalmente superior. Su conclusión útil para nosotros es que separar los roles no basta: el evaluador necesita criterios fiables. [Fuente primaria](https://www.anthropic.com/research/harness-design-long-running-apps).

Para el filtro, comprobaremos por separado que la consulta devuelve los registros esperados y que una persona puede utilizar el control en la aplicación. No reemplazaremos ambas comprobaciones por la opinión de otro modelo.

## Mejora mediante entrenamiento

El entrenamiento puede enseñar al modelo a revisar respuestas o aprovechar retroalimentación de forma más eficaz. Requiere acceso a parámetros entrenables, recursos y evaluación; no sucede automáticamente porque el agente conserve una conversación.

### Introspección recursiva con RISE

**RISE (Recursive IntroSpEction)** busca enseñar corrección iterativa mediante ajuste fino. El trabajo, publicado el 25 de julio de 2024, formula la resolución como una interacción de varios turnos: el modelo aprende a modificar una respuesta después de intentos fallidos, con retroalimentación del entorno opcional. No consiste únicamente en añadir «revisa tu respuesta» a una instrucción. [Publicación original](https://arxiv.org/abs/2407.18219).

Su utilidad conceptual es distinguir la capacidad aprendida de corregir de la petición de hacerlo. Los resultados corresponden a sus modelos y tareas; no prueban mejora ilimitada ni que cualquier agente de herramientas vaya a beneficiarse igual. Lo conservamos como antecedente experimental, fuera de la implementación inicial.

### Aprendizaje por refuerzo en tiempo de prueba

El **aprendizaje por refuerzo en tiempo de prueba (Test-Time Reinforcement Learning, TTRL)** propone actualizar parámetros usando datos sin etiquetas explícitas. Publicado el 22 de abril de 2025, genera varias respuestas, extrae la respuesta más frecuente como pseudoetiqueta, asigna recompensas por coincidencia y entrena mediante **optimización de políticas relativas a grupos (Group Relative Policy Optimization, GRPO)**. [Publicación original](https://arxiv.org/abs/2504.16084).

La diferencia con la autoconsistencia es sustancial: elegir una respuesta no modifica parámetros; TTRL los actualiza. Tampoco convierte el acuerdo en verdad. Si varias respuestas repiten el mismo error, la pseudoetiqueta puede ser incorrecta. Los resultados experimentales no garantizan señales fiables en cualquier dominio.

Como analogía propia, varias resoluciones que coinciden en 40 pueden proporcionar una señal de acuerdo; verificar la operación sigue siendo otra comprobación. TTRL es una propuesta de entrenamiento experimental, no una función que activaremos al dar memoria a nuestro agente.

### R-Zero

**R-Zero**, publicado el 7 de agosto de 2025 y revisado el 13 de febrero de 2026, estudia generación de datos y entrenamiento mediante dos modelos inicializados desde una base común: un **retador (Challenger)** propone problemas y un **solucionador (Solver)** intenta resolverlos. Se entrenan por fases para producir un currículo de dificultad ajustada. [Publicación y versión revisada](https://arxiv.org/abs/2508.05004).

La recompensa del retador favorece problemas con acuerdo intermedio entre respuestas del solucionador y penaliza repeticiones. Ese acuerdo no es la probabilidad de estar en lo cierto. Después se seleccionan problemas y se entrena el solucionador con recompensas basadas en pseudoetiquetas obtenidas por votación.

**Coincidir con una pseudoetiqueta no equivale a verificar contra una respuesta externa correcta.** «Zero data» se refiere a no partir de tareas y etiquetas para ese proceso; los modelos base ya fueron entrenados. Es investigación experimental, no evidencia de mejora sin límites ni garantía de generar tareas válidas para nuestro producto.

### Decidir qué incorporar al agente

Para nuestra primera versión utilizaremos planificación explícita cuando ayude, un bucle observable y revisión apoyada en pruebas. Conservaremos experiencias útiles en memoria con su evidencia. No actualizaremos pesos durante las conversaciones.

Si la evaluación detecta un fallo recurrente, primero comprobaremos qué lo causa: falta de información, una interfaz confusa, un error de ejecución, un criterio de evaluación incorrecto o una limitación del modelo. Cambiar instrucciones, herramientas o contexto puede resolver problemas que un entrenamiento costoso no corregiría por sí solo.

Si más adelante entrenamos, separaremos datos de entrenamiento y de evaluación, compararemos versiones con tareas representativas y comprobaremos regresiones antes de adoptar el modelo. Estas son decisiones de nuestro diseño; los artículos anteriores explican mecanismos posibles, no obligaciones para construir un agente.

# Resumen

La planificación organiza objetivos y dependencias; la secuenciación elige acciones según el estado actual. ReAct ofrece un antecedente para combinar solicitudes de acción con observaciones, mientras que las API estructuradas facilitan representar ese intercambio sin extraerlo de texto libre.

La reflexión genera hipótesis de corrección. Las pruebas, herramientas y personas aportan evidencias con límites propios. Self-Refine revisa salidas; Reflexion conserva lecciones entre intentos; CRITIC incorpora herramientas a la revisión. Ninguno garantiza mejorar por el solo hecho de iterar.

FireAct, ETO y RISE muestran formas de entrenar con trayectorias o revisiones. TTRL y R-Zero estudian señales y datos generados durante procesos de entrenamiento. Distinguimos esos cambios de parámetros de actualizar un plan o guardar información en memoria.

Para nuestro agente queda definido un circuito con tareas verificables, solicitudes autorizadas, resultados asociados a cada ejecución, estado persistente y condiciones de parada. Los ejemplos del capítulo son ilustrativos; todavía debemos implementar y comprobar ese contrato. En el capítulo de evaluación mediremos el resultado alcanzado y el proceso que lo produjo.

# Referencias y alcance de las fuentes

Revisión editorial con fecha de referencia **3 de noviembre de 2026**. La ventana de novedades consultada abarca del **3 de noviembre de 2023 al 3 de noviembre de 2026**. Las fechas siguientes corresponden a primeras publicaciones verificadas o revisiones identificadas, no a fechas de rastreo. Las fuentes anteriores se conservan cuando explican un mecanismo que seguimos utilizando; no se presentan como novedades.

Los trabajos experimentales describen resultados bajo sus condiciones de evaluación. Las experiencias de ingeniería de un proveedor orientan decisiones, pero no establecen superioridad universal ni adopción generalizada. La documentación de API describe contratos disponibles al consultarla; no le atribuimos una fecha de publicación reciente sin evidencia.

## Fundamentos y propuestas de investigación

- Wei, Jason et al. [Chain-of-Thought Prompting Elicits Reasoning in Large Language Models](https://arxiv.org/abs/2201.11903). Primera publicación: 28 de enero de 2022. Antecedente de pasos intermedios expresados en texto.
- Wang, Xuezhi et al. [Self-Consistency Improves Chain of Thought Reasoning in Language Models](https://arxiv.org/abs/2203.11171). Primera publicación: 21 de marzo de 2022. Agregación de respuestas de varias resoluciones.
- Zhou, Denny et al. [Least-to-Most Prompting Enables Complex Reasoning in Large Language Models](https://arxiv.org/abs/2205.10625). Primera publicación: 21 de mayo de 2022. Descomposición y resolución secuencial.
- Yao, Shunyu et al. [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629). Primera publicación: 6 de octubre de 2022; ICLR 2023. Interacción entre acciones y observaciones.
- Shinn, Noah et al. [Reflexion: Language Agents with Verbal Reinforcement Learning](https://arxiv.org/abs/2303.11366). Primera publicación: 20 de marzo de 2023. Retroalimentación lingüística y memoria sin actualizar pesos.
- Madaan, Aman et al. [Self-Refine: Iterative Refinement with Self-Feedback](https://arxiv.org/abs/2303.17651). Primera publicación: 30 de marzo de 2023. Revisión de salidas con el mismo modelo.
- Wang, Lei et al. [Plan-and-Solve Prompting: Improving Zero-Shot Chain-of-Thought Reasoning by Large Language Models](https://arxiv.org/abs/2305.04091). Primera publicación: 6 de mayo de 2023. Planificación y resolución mediante instrucciones.
- Yao, Shunyu et al. [Tree of Thoughts: Deliberate Problem Solving with Large Language Models](https://arxiv.org/abs/2305.10601). Primera publicación: 17 de mayo de 2023; revisión: 3 de diciembre de 2023. Búsqueda entre pasos candidatos, anterior a la ventana de novedades por su origen.
- Gou, Zhibin et al. [CRITIC: Large Language Models Can Self-Correct with Tool-Interactive Critiquing](https://arxiv.org/abs/2305.11738). Primera publicación: 19 de mayo de 2023; revisión: 21 de febrero de 2024. Revisión asistida por herramientas.
- Rafailov, Rafael et al. [Direct Preference Optimization: Your Language Model is Secretly a Reward Model](https://arxiv.org/abs/2305.18290). Primera publicación: 29 de mayo de 2023. Objetivo de entrenamiento por preferencias utilizado en ETO.
- Huang, Jie et al. [Large Language Models Cannot Self-Correct Reasoning Yet](https://arxiv.org/abs/2310.01798). Primera publicación: 3 de octubre de 2023; revisión: 14 de marzo de 2024. Límites de autocorrección intrínseca en sus experimentos.
- Chen, Baian et al. [FireAct: Toward Language Agent Fine-tuning](https://arxiv.org/abs/2310.05915). Primera publicación: 9 de octubre de 2023. Ajuste de agentes con trayectorias.
- Kambhampati, Subbarao et al. [LLMs Can't Plan, But Can Help Planning in LLM-Modulo Frameworks](https://arxiv.org/abs/2402.01817). Primera publicación: 2 de febrero de 2024; ICML 2024. Artículo de posición sobre generación y verificación externa de planes.
- Song, Yifan et al. [Trial and Error: Exploration-Based Trajectory Optimization for LLM Agents](https://arxiv.org/abs/2403.02502). Primera publicación: 4 de marzo de 2024. Exploración y entrenamiento por preferencias de trayectorias.
- Qu, Yuxiao et al. [Recursive Introspection: Teaching Language Model Agents How to Self-Improve](https://arxiv.org/abs/2407.18219). Primera publicación: 25 de julio de 2024. Ajuste fino para corrección iterativa mediante RISE.
- Zuo, Yuxin et al. [TTRL: Test-Time Reinforcement Learning](https://arxiv.org/abs/2504.16084). Primera publicación: 22 de abril de 2025; versión consultada revisada el 30 de junio de 2025. Entrenamiento experimental con recompensas estimadas.
- Huang, Chengsong et al. [R-Zero: Self-Evolving Reasoning LLM from Zero Data](https://arxiv.org/abs/2508.05004). Primera publicación: 7 de agosto de 2025; versión consultada revisada el 13 de febrero de 2026. Generación de currículo y entrenamiento experimental.

## Experiencias de ingeniería y documentación

- Anthropic. [Building effective agents](https://www.anthropic.com/research/building-effective-agents). Publicado el 19 de diciembre de 2024. Patrones de orquestación y límites de ejecución; su advertencia sobre cambios posteriores en herramientas impide tratar el catálogo de productos de 2024 como actual.
- Anthropic. [Reasoning models don't always say what they think](https://www.anthropic.com/research/reasoning-models-dont-say-think). Publicado el 3 de abril de 2025. Investigación sobre fidelidad de explicaciones en los modelos evaluados.
- Anthropic. [Effective harnesses for long-running agents](https://www.anthropic.com/research/effective-harnesses-for-long-running-agents). Publicado el 26 de noviembre de 2025. Experiencia de ingeniería con progreso incremental entre sesiones.
- Anthropic. [Demystifying evals for AI agents](https://www.anthropic.com/research/demystifying-evals-for-ai-agents). Publicado el 9 de enero de 2026. Distinción entre registro de interacción y resultado en el entorno.
- Anthropic. [Harness design for long-running application development](https://www.anthropic.com/research/harness-design-long-running-apps). Publicado el 24 de marzo de 2026. Experiencia con roles de planificación, generación y evaluación.
- Claude Platform. [Handle tool calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls). Documentación consultada el 3 de noviembre de 2026; sin fecha editorial verificada. Contrato de solicitudes, resultados e identificación de errores.
