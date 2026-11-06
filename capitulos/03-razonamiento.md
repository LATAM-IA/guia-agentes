# Capítulo 3. Razonamiento con modelos de lenguaje extensos

Los **modelos de razonamiento** generan pasos intermedios antes de dar una respuesta final. Ese procesamiento les permite descomponer un problema, explorar posibles soluciones y revisar resultados antes de responder. Los pasos pueden permanecer ocultos o mostrarse de forma resumida.

Una analogía útil para entender esta diferencia es la distinción entre **Sistema 1 y Sistema 2** que presenta Daniel Kahneman (2011). El Sistema 1 opera de manera rápida y automática, apoyándose en asociaciones aprendidas. El Sistema 2 requiere atención y esfuerzo para abordar una tarea de forma deliberada. [Referencia editorial](https://academic.macmillan.com/academictrade/9780374533557/thinkingfastandslow/).

Podemos comparar una respuesta directa del modelo con el Sistema 1: produce una salida sin dedicar una etapa extensa a desarrollar soluciones intermedias. Los modelos de razonamiento se parecen, en ese sentido limitado, al Sistema 2: emplean más procesamiento para trabajar sobre el problema antes de entregar una respuesta. Esta comparación no implica que los modelos piensen como las personas ni que dedicar más procesamiento garantice una solución correcta.

En un agente, el razonamiento ayuda a decidir cómo avanzar: qué información falta, qué herramienta conviene utilizar y si el resultado obtenido permite continuar o requiere revisar el plan. Por ejemplo, un agente de programación puede proponer una secuencia de cambios, ejecutar una prueba y usar el error que recibe para ajustar su siguiente acción. El código que rodea al modelo ejecuta esas acciones y devuelve sus resultados al contexto.

En este capítulo vamos a explorar cómo mejorar esas capacidades. Estudiaremos el papel del entrenamiento y del procesamiento durante la ejecución, también llamado **inferencia**. Después veremos técnicas y ejemplos de código para generar posibles soluciones, comprobarlas y usar esa evaluación para orientar la respuesta.

**Referencia bibliográfica**

Kahneman, D. (2011). *Thinking, fast and slow*. Farrar, Straus and Giroux.

# El cambio de paradigma del cálculo en tiempo de entrenamiento al cálculo en tiempo de prueba

Antes de analizar con más detalle los modelos LLM de razonamiento, es importante comprender por qué actualmente se les presta tanta atención. Esto se relaciona con un cambio fundamental que ha transformado nuestro enfoque, pasando de optimizar el cálculo en tiempo de entrenamiento al cálculo en tiempo de prueba.

## Cálculo en tiempo de entrenamiento frente a cálculo en tiempo de prueba

Un modelo puede mejorar por el entrenamiento que recibió y por el procesamiento que dedica a resolver una tarea cuando lo usamos.

Durante el **entrenamiento**, el cómputo se utiliza para aprender y ajustar los pesos del modelo. Ese aprendizaje se aprovecha después en muchas consultas.

Durante la **inferencia**, también llamada aquí tiempo de prueba, el cómputo se utiliza para resolver la consulta actual. Puede dedicarse a explorar posibles soluciones y comprobar resultados. En el uso habitual, este proceso no modifica los pesos del modelo.

Al construir un agente, necesitamos decidir cuánto esfuerzo requiere cada tarea. Una consulta sencilla puede resolverse con una respuesta directa; un problema complejo puede justificar más pasos, tiempo y costo. **Generar más tokens por sí solo no garantiza una respuesta mejor: el procesamiento adicional tiene que ayudar a resolver o verificar el problema.**

## Leyes de escala

Las **leyes de escala** describen cómo cambia el rendimiento de un modelo al aumentar los recursos de entrenamiento: capacidad de cómputo, cantidad de datos y número de parámetros. Las mejoras suelen tener **rendimientos decrecientes**: invertir el doble no significa obtener un modelo el doble de capaz.

**Para aprovechar bien el presupuesto de entrenamiento, hay que equilibrar la capacidad de cómputo, la cantidad de datos y el tamaño del modelo.** Los trabajos de Kaplan y Chinchilla estudiaron cómo distribuir esos recursos. Aumentar los parámetros sin suficientes datos puede dejar un modelo insuficientemente entrenado. Chinchilla mostró que, con el mismo presupuesto, un modelo más pequeño entrenado con más datos podía superar a uno más grande.

Otra posibilidad es dedicar más cómputo a la **inferencia**. Esto puede significar explorar varias soluciones, compararlas o comprobar sus resultados con herramientas. **Escalar la inferencia no significa solamente producir una respuesta más larga.**

En algunas tareas, un modelo más pequeño que explora y verifica varias soluciones puede superar a uno más grande que responde directamente. Esa compensación tiene límites: su utilidad depende de la tarea, de las capacidades del modelo y de cómo se emplea el procesamiento adicional.

Al construir un agente, podemos **ajustar el esfuerzo según la dificultad de la tarea**. Una consulta sencilla puede resolverse directamente; un problema complejo puede justificar varios intentos y un verificador. Necesitamos evaluar si la mejora compensa el tiempo y el costo adicionales.

Los rendimientos decrecientes tampoco demuestran que hayamos llegado a un límite definitivo. Indican que cada mejora exige más recursos bajo las condiciones estudiadas; no que mejorar sea imposible.

### Cómo ajustar el esfuerzo según la tarea

| Tipo de tarea | Esfuerzo inicial | Cómo aplicarlo | Cuándo aumentarlo |
| --- | --- | --- | --- |
| Extraer datos, clasificar o convertir formatos | **Bajo** | Una llamada con instrucciones y estructura de salida claras. Validar campos y formato. | Si omite información o incumple reglas aun teniendo los datos necesarios. |
| Resumir, traducir o editar un texto | **Bajo** | Dar el texto completo y definir qué debe conservar. Comparar la salida con el original. | Si necesita relacionar documentos, resolver contradicciones o preservar matices difíciles. |
| Comparar opciones o preparar un plan | **Medio** | Explicitar objetivos, restricciones y criterios de comparación. | Si hay dependencias difíciles, alternativas incompatibles o consecuencias que no evaluó. |
| Implementar un cambio acotado de código | **Medio** | Dar contexto del código y criterios de aceptación. Comprobar con tests. | Si falla un test de comportamiento o aparecen interacciones que requieren más análisis. |
| Diagnosticar un error difícil o revisar un cambio complejo | **Alto** | Trabajar con una reproducción, evidencia y verificaciones concretas. | Probar un nivel superior si el diagnóstico sigue fallando y existe una forma de comprobar la solución. |
| Resolver problemas matemáticos o lógicos de varios pasos | **Medio o alto** | Comprobar el resultado con cálculo, ejecución o un verificador independiente. | Si la comprobación detecta errores; puede convenir explorar otra solución. |
| Investigar información reciente | **Medio, con búsqueda** | Consultar fuentes primarias y contrastar fechas y afirmaciones. | Si hace falta sintetizar evidencia contradictoria. Si faltan datos, buscar más: aumentar el esfuerzo no los aporta. |

Orientación inicial para evaluar en cada agente; los niveles dependen del modelo. **Cada vez que cambies de modelo o de versión, vuelve a comprobar los niveles de esfuerzo con tareas representativas de tu agente.** Compara la calidad de los resultados, el tiempo y el costo: un mismo nivel de esfuerzo puede comportarse de forma distinta entre modelos y versiones. Síntesis basada en la [guía de OpenAI del 2 de octubre de 2026](https://openai.com/index/practical-guide-building-gpt-6/) y la [documentación de esfuerzo de Claude](https://platform.claude.com/docs/en/build-with-claude/effort), consultadas el 2 de noviembre de 2026.

# Cómo dedicar más cómputo a una tarea

Podemos explorar varias soluciones y comprobar cuál funciona mejor, o mejorar la forma en que el modelo genera sus respuestas mediante instrucciones o entrenamiento.

En los siguientes apartados veremos cómo aplicar estas estrategias. Algunas se pueden incorporar al agente durante la ejecución; otras requieren modificar el entrenamiento del modelo.

## Modelos de recompensa

Un **modelo de recompensa** se usa para evaluar las respuestas de otro modelo. Les asigna puntuaciones que pueden orientar el entrenamiento, ayudar a elegir entre varias respuestas o indicar qué solución conviene seguir explorando.

Por ejemplo, si un agente genera tres explicaciones para una misma pregunta, un modelo de recompensa puede puntuarlas según su claridad y precisión. El sistema puede usar esas puntuaciones para seleccionar una.

Para aprender a evaluar, el modelo se entrena con ejemplos calificados. Esos ejemplos pueden indicar qué respuesta prefirieron las personas, qué resultado era correcto o en qué paso apareció un error. La puntuación representa lo que el evaluador aprendió a valorar.

No todos los agentes necesitan un modelo de recompensa. Si podemos comprobar el resultado con una calculadora, una regla o tests, esa comprobación puede ser suficiente. Usar un modelo resulta útil cuando evaluar requiere un juicio más difícil de expresar con reglas.

Podemos evaluar una solución de dos maneras: por su resultado final o por los pasos que llevaron a él.

### Evaluar el resultado

Un **modelo de recompensa por resultado** (*outcome reward model*, ORM) evalúa la respuesta final, sin puntuar por separado los pasos que llevaron a ella.

Por ejemplo, le pedimos al modelo que calcule el precio final de un producto de $20.000 con un 15 % de descuento. Podemos evaluar si responde **$17.000**, sin revisar cómo hizo el cálculo.

En este caso, una regla o una calculadora bastan para verificar el resultado: no necesitamos otro modelo. Para evaluar aspectos como la claridad de una explicación o la utilidad de una respuesta, podemos usar un modelo de recompensa que estime su calidad.

### Evaluar el proceso

Un **modelo de recompensa por proceso** (*process reward model*, PRM) evalúa los pasos intermedios de una solución.

Por ejemplo, le pedimos al modelo que explique cómo repartir por igual una cuenta de $36.000 entre cuatro personas, incluyendo una propina del 10 %. El evaluador podría revisar esta secuencia:

1. La propina es $3.600.
2. El total con propina es $39.600.
3. Cada persona debe pagar $9.900.

Si el segundo paso sumara mal la propina, el evaluador podría señalar ese error antes de que se trasladara al reparto. Se evalúan los pasos que el sistema recibe, no pensamientos internos a los que no tiene acceso.

### Cómo usamos estas evaluaciones

Durante el **entrenamiento**, las puntuaciones sirven como señal para ajustar el modelo y favorecer respuestas mejor evaluadas.

Durante la **ejecución**, pueden ayudarnos a elegir entre varias respuestas completas o a decidir qué solución parcial conviene seguir explorando. El modelo de recompensa puntúa; el sistema que lo utiliza decide cómo actuar con esa puntuación.

Evaluar el resultado y el proceso aporta información distinta: una respuesta correcta puede contener pasos equivocados, y una solución que empieza bien puede terminar mal. Por eso se pueden combinar ambas evaluaciones.

**El evaluador también puede equivocarse. Una puntuación alta indica que la solución cumple sus criterios; no garantiza que sea correcta.**

## Prompt engineering

**Prompt engineering** es diseñar las instrucciones y el contexto que recibe un modelo para resolver una tarea. Incluye explicar el objetivo, aportar información relevante y definir las condiciones que debe cumplir la respuesta.

En un agente, esas instrucciones también indican cuándo consultar una herramienta, qué información necesita antes de continuar y cómo comprobar el resultado. **Permiten orientar el comportamiento del modelo sin modificar sus pesos.** Las instrucciones no garantizan que el modelo las cumpla: el código del agente debe validar los resultados y controlar las acciones que permite ejecutar.

### Guiar mediante instrucciones y ejemplos

Podemos explicar una tarea directamente o mostrar cómo esperamos que se resuelva. Según la cantidad de ejemplos incluidos en el contexto, hablamos de:

| Técnica | Qué recibe el modelo | Ejemplo |
| --- | --- | --- |
| **Zero-shot** | Una instrucción sin ejemplos resueltos. | «Clasificá el mensaje como consulta o reclamo: “Me cobraron dos veces”. Respondé solo con la categoría». |
| **One-shot** | Una instrucción y un ejemplo resuelto. | «Clasificá como consulta o reclamo. Ejemplo: “¿Cuándo llega mi pedido?” → consulta. Ahora clasificá: “Me cobraron dos veces”». |
| **Few-shot** | Una instrucción y varios ejemplos resueltos. | «Clasificá como consulta o reclamo. Ejemplos: “¿Cuándo llega mi pedido?” → consulta; “Recibí un producto roto” → reclamo. Ahora clasificá: “Me cobraron dos veces”». |

Los ejemplos comunican el criterio y el formato esperado. Conviene elegirlos para representar los casos que el agente necesita distinguir. **Agregar más ejemplos no garantiza mejores resultados:** pueden ser redundantes, contradictorios o poco adecuados para la tarea.

### Chain-of-thought: desarrollar una solución por pasos

**Chain-of-thought** (CoT) es una técnica de prompting que utiliza ejemplos con pasos intermedios o instrucciones para desarrollar una solución antes de presentar el resultado. Permite comparar qué ocurre cuando pedimos una respuesta directa y cuando proponemos un procedimiento explícito.

Por ejemplo, podemos enviar esta instrucción sin ejemplos:

```text
Un pedido tiene 3 artículos de $8.000 cada uno.
Se aplica un descuento del 10 % al subtotal y después se suman $2.000 de envío.
¿Cuánto se paga en total? Respondé solo con el importe.
```

La respuesta esperada es **$23.600**. Para comparar, podemos mantener el mismo problema y cambiar únicamente la instrucción final:

```text
Calculá el subtotal, aplicá el descuento y sumá el envío al final.
Mostrá esos cálculos y el importe total.
```

Una solución correcta sería:

```text
Subtotal: 3 × $8.000 = $24.000.
Descuento: 10 % de $24.000 = $2.400.
Total: $24.000 − $2.400 + $2.000 = $23.600.
```

También podemos incluir este problema resuelto como ejemplo y pedir que aplique el procedimiento a otro pedido. Así combinamos una demostración con pasos intermedios y one-shot prompting.

El procedimiento escrito permite comprobar si se respetó el orden de las operaciones. **No demuestra que la respuesta directa fuera incorrecta ni que pedir pasos mejore siempre el resultado.** Para decidir qué instrucción usar, necesitamos comparar ambas variantes con tareas representativas y verificar sus respuestas.

Los pasos que el modelo escribe son una explicación generada, no un registro fiel de su procesamiento interno. Agregar «pensemos paso a paso» tampoco garantiza una mejora en cualquier modelo o tarea.

### Conectar las soluciones con un verificador

Si generamos varias soluciones, necesitamos evaluar cuál conservar. En el ejemplo del pedido, una calculadora o una función puede comprobar el total. También podemos revisar que el descuento se aplique antes de sumar el envío.

En tareas donde la evaluación requiere un juicio, podemos usar un modelo de recompensa para puntuar las respuestas o los pasos escritos. Generar alternativas y evaluarlas nos lleva a la siguiente estrategia: la búsqueda guiada por verificadores.

# Búsqueda guiada por verificadores (search against verifiers)

La **búsqueda guiada por verificadores** consiste en generar posibles soluciones, evaluarlas y usar esa evaluación para elegir una o seguir buscando.

Por ejemplo, un agente intenta corregir un error de código:

1. Genera una posible corrección.
2. Ejecuta los tests para comprobarla.
3. Si falla, usa el resultado para probar otra solución.
4. Si pasa, conserva esa candidata para las verificaciones restantes.

También puede generar varias candidatas y compararlas. El verificador puede ser un test, una calculadora, una regla o un modelo que puntúa la calidad del resultado o de los pasos escritos.

La diferencia con pedir una sola respuesta es que el agente dedica esfuerzo a explorar y comprobar alternativas. Podemos ajustar ese esfuerzo modificando la cantidad de candidatas o de intentos, sin volver a entrenar el modelo que las genera.

**La utilidad de la búsqueda depende del verificador: pasar unos tests no demuestra que el código esté libre de errores.**

## Autoresearch

Podemos aplicar esta misma lógica a un ciclo de experimentación automática: proponer un cambio, ejecutar un experimento, medir el resultado y conservar o descartar el cambio antes de probar el siguiente. A esta forma de organizar la experimentación la llamamos aquí **autoresearch**.

| Búsqueda guiada por verificadores | Ciclo de autoresearch |
| --- | --- |
| Genera una solución candidata. | Propone un cambio al código o a la configuración. |
| Evalúa la candidata con un verificador. | Ejecuta un experimento y mide una métrica. |
| Usa el resultado para elegir o seguir buscando. | Conserva o descarta el cambio y prepara el siguiente experimento. |

La búsqueda guiada por verificadores es la idea general; autoresearch organiza ese ciclo alrededor de experimentos con un objetivo medible.

**Mejorar la métrica no garantiza mejorar lo que realmente nos importa.** Si el experimento busca reducir el tiempo de ejecución, también necesita comprobar que los resultados sigan siendo correctos. Conviene definir antes de empezar qué se puede modificar, cómo se evalúa una mejora y cuándo se detiene la búsqueda.

### Documentación de Andrej Karpathy

El proyecto **autoresearch de Andrej Karpathy** aplica este ciclo a experimentos de entrenamiento de un modelo de lenguaje en una GPU. El agente modifica `train.py`, ejecuta un entrenamiento con un presupuesto fijo de cinco minutos y compara la métrica de validación `val_bpb`, donde un valor menor es mejor.

- [Repositorio y README oficial](https://github.com/karpathy/autoresearch): explica el objetivo, la estructura del proyecto, los requisitos y cómo ejecutar un experimento.
- [program.md: instrucciones del agente](https://github.com/karpathy/autoresearch/blob/master/program.md): define el alcance de los cambios, la línea base, el registro de resultados y el criterio para conservar o descartar experimentos.

## Autoconsistencia (self-consistency)

La **autoconsistencia** consiste en generar varias soluciones para una misma tarea y seleccionar la respuesta final que aparece con mayor frecuencia. Las soluciones pueden seguir caminos diferentes y llegar al mismo resultado.

Por ejemplo, pedimos calcular el total de un pedido y obtenemos cinco respuestas:

| Intento | Resultado |
| --- | ---: |
| 1 | $23.600 |
| 2 | $25.600 |
| 3 | $23.600 |
| 4 | $21.600 |
| 5 | $23.600 |

El sistema selecciona **$23.600**, porque es el resultado más frecuente. Antes de contar, debe reconocer respuestas equivalentes: «23.600 pesos» y «$23.600» representan el mismo importe.

A diferencia de un verificador que comprueba la solución, esta técnica **selecciona por acuerdo entre respuestas**. No necesita necesariamente un modelo de recompensa ni conocer de antemano el resultado correcto.

Puede ayudar cuando el modelo llega a la respuesta correcta por distintos caminos, pero algunas ejecuciones cometen errores. Sin embargo, **que varias respuestas coincidan no demuestra que sean correctas**: el modelo puede repetir el mismo error en todos los intentos.

Cada solución adicional consume tokens y puede aumentar el tiempo de ejecución. Por eso conviene definir un presupuesto y comprobar si la votación mejora los resultados frente a una sola respuesta. También hay variantes que ajustan la cantidad de intentos durante la ejecución para aprovechar mejor ese presupuesto.

Para nuestro agente, esta estrategia resulta más sencilla de aplicar cuando las respuestas finales se pueden comparar claramente, como números o categorías. Si contamos con una comprobación directa, podemos usarla para verificar la respuesta seleccionada.

## Selección de la mejor respuesta entre N candidatas (best-of-N sampling)

**Best-of-N sampling** consiste en generar una cantidad determinada de soluciones candidatas, evaluarlas y seleccionar la que obtiene la puntuación más alta. La letra **N** representa cuántas candidatas generamos.

A diferencia de la autoconsistencia, no elegimos la respuesta que más se repite. **Elegimos la mejor evaluada según un criterio definido.**

Por ejemplo, le pedimos al modelo una función que calcule el total de un pedido aplicando descuentos y gastos de envío. Generamos tres versiones y ejecutamos los mismos tests sobre cada una:

| Candidata | Tests superados |
| --- | ---: |
| A | 6 de 10 |
| B | 10 de 10 |
| C | 8 de 10 |

Seleccionamos la candidata **B** porque supera más comprobaciones. Esto no demuestra que funcione en todos los casos: su evaluación depende de lo que cubren los tests.

Podemos evaluar el **resultado final**, como en este ejemplo, o los **pasos intermedios** de una solución. También podemos combinar ambos criterios. El evaluador puede ser una regla, un conjunto de tests o un modelo de recompensa.

La técnica permite dedicar más cómputo a una tarea sin volver a entrenar el modelo que genera las respuestas. Sin embargo, generar y evaluar más candidatas aumenta el costo y puede aumentar el tiempo de ejecución.

**Más candidatas no garantizan una respuesta mejor.** Si el evaluador se equivoca, podemos terminar seleccionando una respuesta que obtiene una puntuación alta sin resolver correctamente la tarea.

Para nuestro agente, necesitamos definir cuántas candidatas generar, cómo puntuarlas y qué hacer si ninguna cumple los requisitos. El presupuesto y el criterio de selección deben comprobarse con tareas representativas.

# Mejorar cómo el modelo genera soluciones (modifying the proposal distribution)

Hasta ahora vimos cómo generar varias soluciones y seleccionar una mediante votación o evaluación. Otra estrategia es **entrenar al modelo para que produzca mejores soluciones desde el principio**.

Cuando genera texto, el modelo calcula probabilidades para los posibles tokens siguientes. El entrenamiento ajusta sus pesos y modifica esas probabilidades. Así puede favorecer comportamientos útiles para resolver tareas, como descomponer un problema o comprobar un resultado antes de responder.

Podemos orientar ese aprendizaje de dos maneras:

- **Ajuste fino supervisado (SFT):** entrenamos al modelo con ejemplos de las respuestas y los procedimientos que queremos que aprenda.
- **Aprendizaje por refuerzo (RL):** evaluamos las respuestas que genera y usamos recompensas para orientar el ajuste de sus pesos.

Por ejemplo, para resolver problemas de descuentos, podemos mostrar soluciones donde primero se calcula el subtotal, después se aplica el descuento y finalmente se suma el envío. También podemos recompensar las respuestas cuyo resultado sea correcto. Son señales de entrenamiento diferentes que pueden combinarse.

**Esta estrategia modifica el modelo durante el entrenamiento.** La búsqueda entre candidatas, en cambio, puede aplicarse durante la ejecución sin cambiar sus pesos. También podemos usar ambas: un modelo entrenado para resolver mejor una tarea y un verificador que compruebe sus resultados.

Al construir nuestro agente, esta distinción nos ayuda a decidir dónde intervenir: en las instrucciones y verificaciones de la ejecución, o en el entrenamiento del modelo. Los siguientes apartados profundizan en SFT y RL.

## Ajuste fino supervisado (supervised fine-tuning, SFT)

El **ajuste fino supervisado** permite adaptar un modelo mediante ejemplos de las respuestas que esperamos. Para enseñar un procedimiento, los datos pueden incluir una pregunta, una solución desarrollada y la respuesta final.

Por ejemplo, podemos preparar problemas de compras que muestren cómo calcular el subtotal, aplicar un descuento y sumar el envío. Durante el entrenamiento, el modelo ajusta sus pesos para reproducir las secuencias de los ejemplos.

Los ejemplos pueden provenir de personas o de otros modelos, pero necesitan revisión. Una explicación convincente con un cálculo incorrecto también puede convertirse en un patrón aprendido. **La calidad de los ejemplos y su relación con la tarea importan más que acumular explicaciones largas.**

A diferencia de few-shot prompting, estos ejemplos se usan para modificar los pesos durante el entrenamiento; no se agregan solamente al contexto de una consulta.

### Flan-PaLM: aprender de instrucciones y ejemplos

**Flan-PaLM** es un caso histórico de ajuste con instrucciones. El trabajo combinó numerosas tareas, incluyendo ejemplos con soluciones intermedias, y evaluó cómo influían el tamaño del modelo, la variedad de tareas y los datos de chain-of-thought.

La idea que nos interesa es que **un modelo preentrenado puede mejorar su manera de responder mediante ejemplos de tareas y procedimientos**. No necesitamos interpretar este resultado como prueba de que escribir pasos garantice razonar correctamente.

Referencia: [Chung et al., Scaling Instruction-Finetuned Language Models, JMLR, 2024](https://www.jmlr.org/papers/v25/23-0870.html).

### s1: escalado en tiempo de prueba (simple test-time scaling)

El proyecto **s1** ajustó Qwen2.5-32B-Instruct con un conjunto seleccionado de 1.000 preguntas y soluciones desarrolladas. La selección consideró dificultad, diversidad y calidad. Partía de un modelo ya entrenado: no creó sus capacidades desde cero con solo esos ejemplos.

También exploró **budget forcing**, una técnica para controlar la duración de la generación. Podía finalizarla antes o extenderla agregando «Wait» cuando el modelo intentaba terminar, para que continuara trabajando sobre la solución.

El caso muestra dos intervenciones diferentes: preparar el comportamiento mediante SFT y controlar el cómputo durante la ejecución. Sus resultados corresponden a las tareas y al modelo evaluados. **Agregar «Wait» a cualquier modelo no garantiza una mejora ni sustituye un mecanismo de control compatible.**

Para nuestro agente, la enseñanza es evaluar conjuntamente la calidad de los datos y el presupuesto de ejecución, en lugar de asumir que necesitamos siempre más ejemplos o más tokens.

Referencias: [Muennighoff et al., s1, 2025](https://arxiv.org/abs/2501.19393) y [repositorio oficial](https://github.com/simplescaling/s1).

## Aprendizaje por refuerzo (reinforcement learning, RL)

En el aprendizaje por refuerzo, el modelo genera respuestas y recibe señales de recompensa que orientan el ajuste de sus pesos. Cuando podemos comprobar el resultado automáticamente, hablamos de **recompensas verificables**: por ejemplo, verificar una solución matemática o ejecutar tests sobre un programa.

El entrenamiento puede favorecer procedimientos que obtienen mejores resultados, aunque no se haya proporcionado una solución desarrollada para cada problema. Eso no garantiza que el modelo descubra siempre un procedimiento válido: aprende a optimizar la señal que recibe.

**Diseñar la recompensa también es definir qué errores podemos pasar por alto.** Si solo comprobamos el formato, podemos favorecer respuestas bien presentadas que no resuelven la tarea.

### DeepSeek-R1-Zero: aprender con recompensas

**DeepSeek-R1-Zero** exploró RL sobre DeepSeek-V3-Base sin una etapa previa de SFT específica para ese experimento. Utilizó GRPO y recompensas basadas en precisión y formato.

Los autores observaron conductas de revisión y exploración de soluciones, junto con problemas de legibilidad y mezcla de idiomas. Es un resultado experimental sobre un modelo previamente preentrenado, no un sistema que aprendió desde cero sin datos.

### DeepSeek-R1: combinar supervisión y refuerzo

**DeepSeek-R1** combinó ejemplos iniciales de calidad, RL orientado al razonamiento, generación y filtrado de datos, otra etapa de SFT y una fase final de RL que también atendía utilidad y seguridad.

Generar y filtrar datos permite aprovechar respuestas del propio sistema para preparar nuevos ejemplos de entrenamiento. El filtro importa tanto como la generación: una respuesta incorrecta que supera la revisión puede incorporarse a los datos.

**SFT y RL se pueden complementar.** Este caso no demuestra que SFT solo memorice ni que RL garantice generalizar. Para nuestro agente, usar un modelo así tampoco significa que sus pesos cambien con cada interacción.

Referencia para ambos casos: [DeepSeek-AI, DeepSeek-R1, 2025](https://arxiv.org/abs/2501.12948).

## Razonamiento nativo (native reasoning)

Usamos aquí **razonamiento nativo** para referirnos a modelos entrenados para dedicar procesamiento intermedio a una tarea. No es un protocolo único: la manera de solicitarlo depende del modelo y del sistema que lo ejecuta.

Un servidor puede exponer un nivel de esfuerzo o un modo de generación. Al ejecutar un modelo local, su **plantilla de chat** convierte los mensajes y roles al formato que espera. Los marcadores especiales pertenecen a ese formato: no se traducen al español ni se copian entre familias de modelos.

Por ejemplo, Qwen3-8B documenta `enable_thinking` como opción de su plantilla de chat. Esto es una característica de ese modelo y su integración, no una instrucción universal. Referencia: [ficha oficial de Qwen3-8B](https://huggingface.co/Qwen/Qwen3-8B).

Para conectar un modelo al agente:

1. Comprobamos qué controles admite el modelo y la versión del servidor.
2. Configuramos el modo o el esfuerzo mediante esa interfaz.
3. Distinguimos la respuesta final, las solicitudes de herramientas y los demás campos que exponga el proveedor.
4. Probamos la configuración con tareas representativas antes de usarla como valor habitual.

**Ocultar los pasos en la interfaz no significa eliminar su costo, y mostrar una explicación no demuestra cómo se produjo internamente la respuesta.** Necesitamos medir el uso y validar el resultado con la información que realmente ofrece la integración.

# Otras formas de desarrollar el razonamiento

Las siguientes líneas de investigación amplían el tipo de información que puede procesar un modelo y estudian cómo aprovechar mejor el cómputo. Algunos ejemplos son trabajos históricos; otros exploran mecanismos que no están disponibles en cualquier API.

Para nuestro agente, conviene distinguir una capacidad que podemos configurar hoy de una técnica que requiere entrenamiento o cambios en el modelo.

## Razonamiento multimodal (multimodal reasoning)

Un agente puede necesitar interpretar una captura de pantalla, una factura escaneada o un gráfico antes de decidir qué hacer. El **razonamiento multimodal** combina información de distintas modalidades para resolver la tarea.

Por ejemplo, para revisar una factura, primero necesita leer los importes y reconocer a qué conceptos corresponden. Después puede calcular si el total coincide. **Un error de percepción puede producir una conclusión incorrecta aunque el cálculo posterior sea válido.**

Las instrucciones, la selección entre candidatas y el entrenamiento también pueden aplicarse a estas tareas. El trabajo **Multimodal-CoT** estudió separar la generación de una explicación basada en texto e imagen de la obtención de la respuesta final. Es un enfoque concreto, no un requisito para todos los modelos multimodales. [Zhang et al., Multimodal Chain-of-Thought Reasoning in Language Models](https://arxiv.org/abs/2302.00923).

Otros trabajos estudian distintas formas de entrenamiento:

- **LLaVA-CoT** organiza las soluciones en etapas de resumen, descripción visual, razonamiento y conclusión. [Artículo original](https://arxiv.org/abs/2411.10440).
- **Reason-RFT** combina SFT y refuerzo para tareas de razonamiento visual. [Artículo original](https://arxiv.org/abs/2503.20752).

En nuestro agente, conviene separar la extracción de información de su comprobación. Podemos contrastar los importes leídos con una calculadora y pedir una imagen más clara cuando no se distingan los datos. Una explicación extensa no recupera información que el sistema no pudo observar.

## Razonamiento eficiente (efficient reasoning)

El objetivo del **razonamiento eficiente** es resolver la tarea con un uso adecuado de tiempo y recursos. Una respuesta corta no es necesariamente barata si antes consumió mucho procesamiento; una respuesta larga puede incluir información que el usuario necesita.

### Chain-of-Draft: pasos escritos más concisos

**Chain-of-Draft (CoD)** propone generar anotaciones intermedias breves en lugar de explicaciones extensas. Sus autores evaluaron si podían reducir tokens conservando el rendimiento en las tareas estudiadas.

Por ejemplo, la solución de un pedido podría resumirse como «Subtotal: 24.000; descuento: 2.400; envío: 2.000; total: 23.600». Esto ilustra un formato compacto, no una prueba de que esa instrucción ahorre cómputo en cualquier modelo.

**Reducir la explicación visible no garantiza reducir el razonamiento interno.** Hay que comprobar qué tokens contabiliza la integración y si el resultado mantiene su calidad. [Xu et al., Chain of Draft, 2025](https://arxiv.org/abs/2502.18600).

### Controlar el presupuesto durante el entrenamiento

También se puede entrenar al modelo para utilizar presupuestos distintos. **O1-Pruner** estudia reducir la longitud bajo restricciones de precisión; **L1** combina objetivos de exactitud y longitud solicitada. **Kimi k1.5** explora estrategias de entrenamiento y transferencia entre soluciones largas y cortas. Son métodos específicos, no una penalización por longitud que debamos aplicar a toda respuesta.

Referencias: [O1-Pruner](https://arxiv.org/abs/2501.12570), [L1](https://arxiv.org/abs/2503.04697) y [Kimi k1.5](https://arxiv.org/abs/2501.12599).

### Ajustar el esfuerzo de nuestro agente

Podemos empezar por la tabla de esfuerzo de este capítulo y evaluar distintas configuraciones con las mismas tareas. Registramos si se resolvieron correctamente, el tiempo, el costo y los intentos necesarios.

Si faltan datos, buscamos información; si falla una herramienta, revisamos la ejecución; si el problema requiere más análisis, probamos otro nivel de esfuerzo o una estrategia de verificación. **Aumentar el esfuerzo no corrige por sí solo cualquier tipo de fallo.**

Los límites de tokens o intentos ayudan a controlar el presupuesto, pero también pueden interrumpir una solución antes de completarse. El agente debe distinguir un resultado válido de una ejecución que terminó por alcanzar su límite.

## Razonamiento en el espacio latente (latent-space reasoning)

El **razonamiento en el espacio latente** estudia cómo realizar pasos intermedios mediante representaciones numéricas internas, sin convertir cada uno en palabras.

En **Coconut (Chain of Continuous Thought)**, un estado oculto del modelo se reutiliza como entrada del siguiente paso, en lugar de decodificarlo primero como un token de texto. La propuesta explora cómo resolver tareas utilizando esas representaciones continuas. [Hao et al., Training Large Language Models to Reason in a Continuous Latent Space](https://arxiv.org/abs/2412.06769).

**CODI** estudia transferir aprendizaje desde una tarea con soluciones explícitas hacia otra con representaciones continuas mediante autodestilación. Sus resultados pertenecen a las escalas y tareas evaluadas; no prueban una ventaja universal frente a los pasos escritos. [Shen et al., CODI](https://arxiv.org/abs/2502.21074).

Estas técnicas requieren mecanismos específicos de entrenamiento o ejecución. **No ver los pasos de un modelo comercial no demuestra que use razonamiento latente:** podría generar tokens intermedios que la interfaz no muestra.

Para construir nuestro agente, no necesitamos implementar estos métodos. La consecuencia práctica es conservar entradas, acciones, observaciones y resultados verificables, sin depender de acceder al proceso interno del modelo.

# Qué nos llevamos para construir el agente

En este capítulo distinguimos dos lugares donde intervenir: **durante el entrenamiento del modelo y durante su ejecución**. SFT y RL modifican los pesos; las instrucciones, el presupuesto de inferencia y la selección entre candidatas permiten actuar sobre una tarea sin volver a entrenar el modelo.

La elección depende de qué necesita resolver nuestro agente:

| Necesidad | Qué podemos evaluar |
| --- | --- |
| Comunicar mejor la tarea | Instrucciones claras y ejemplos pertinentes. |
| Comparar soluciones | Autoconsistencia para votar o best-of-N para seleccionar por evaluación. |
| Comprobar un resultado | Reglas, cálculo, tests o un evaluador apropiado. |
| Resolver una tarea difícil | Más esfuerzo, exploración de alternativas o un modelo con capacidades adecuadas. |
| Controlar el costo | Presupuestos de ejecución y medición del costo por tarea resuelta. |
| Adaptar de forma persistente el modelo | SFT o RL, si contamos con datos, evaluación y recursos de entrenamiento. |

**El razonamiento ayuda a decidir, pero no sustituye la evidencia ni la ejecución.** El arnés sigue siendo responsable de ejecutar herramientas, registrar lo que ocurrió y comprobar las condiciones necesarias para continuar.

El siguiente componente es la [memoria](04-memoria.md): cómo conservar y recuperar información relevante para que el agente pueda continuar su trabajo entre pasos e interacciones. Después incorporaremos [herramientas](05-herramientas-y-protocolos.md) y [planificación y reflexión](06-planificacion-y-reflexion.md).
