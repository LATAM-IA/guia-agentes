# Capítulo 8. Sistemas multiagente

Un **sistema multiagente (multi-agent system, MAS)** permite distribuir trabajo entre agentes que toman decisiones e intercambian información. Puede resultar útil cuando una tarea necesita explorar líneas relativamente independientes, separar contextos de trabajo o coordinar capacidades diferentes.

Hasta ahora hemos construido conceptualmente un agente que utiliza herramientas, conserva información y ajusta sus acciones según los resultados. En este capítulo estudiaremos cómo delegar parte de ese trabajo y reunir lo obtenido. **Agregar agentes introduce una forma de organizar la tarea; no garantiza resolverla mejor.**

Cada agente puede recibir una responsabilidad, instrucciones, herramientas y permisos distintos. No necesita utilizar un modelo diferente para desempeñar una función especializada. Dos agentes pueden apoyarse en el mismo modelo y trabajar con información y objetivos operativos diferentes.

Tampoco basta con dividir un texto entre varias llamadas para explicar el comportamiento de un sistema multiagente. Debemos precisar qué decisiones puede tomar cada participante, qué información recibe y cómo se coordina con los demás. Algunos sistemas cooperan para cumplir un objetivo compartido; otros incluyen competencia o intereses diferentes. Nuestro primer diseño se centrará en la cooperación.

Consideremos un **ejemplo ilustrativo**, sin ejecutar agentes: queremos comparar dos opciones de almacenamiento para un proyecto. Un coordinador establece los requisitos y encarga a dos investigadores consultar la documentación oficial de cada opción. Cada investigador devuelve sus hallazgos, las fuentes y las dudas pendientes. El coordinador compara los resultados con los mismos criterios y prepara una conclusión.

Las dos búsquedas pueden avanzar en paralelo si no dependen entre sí. La comparación final necesita sus resultados. **Dividir el trabajo no elimina las dependencias ni la responsabilidad de comprobar la evidencia.** Anthropic describe una organización de investigación con un agente coordinador y subagentes en su publicación de junio de 2025. Es un ejemplo de implementación, no una regla para todas las tareas. [Fuente primaria](https://www.anthropic.com/research/multi-agent-research-system).

La delegación necesita un contrato: qué pregunta debe resolver el agente, qué recursos puede utilizar, qué debe devolver y cuándo debe detenerse. Una solicitud de delegación indica trabajo encargado; el resultado recibido y sus comprobaciones permiten establecer qué se completó. Un mensaje que diga «terminado» no sustituye esa evidencia.

La coordinación también tiene límites. Puede duplicar búsquedas, perder información durante los intercambios o propagar un error entre agentes. MAST documenta fallos de diseño, desalineación y verificación en sistemas estudiados. Utilizaremos esas categorías para formular preguntas sobre nuestro diseño, sin tratarlas como una lista exhaustiva. [Investigación original](https://arxiv.org/abs/2503.13657).

Para nuestro agente empezaremos por identificar una tarea que justifique delegación y una versión de referencia que pueda realizar un solo agente. Compararemos el resultado, el tiempo y los recursos utilizados bajo condiciones explícitas. La investigación sobre escalado muestra que una coordinación inadecuada puede empeorar el desempeño; sus resultados dependen de las tareas y configuraciones evaluadas. [Estudio, revisión de abril de 2026](https://arxiv.org/abs/2512.08296v3).

En este capítulo veremos formas de organizar agentes, contratos de comunicación y criterios para decidir cuándo delegar. También examinaremos simulaciones sociales y sistemas de investigación como casos de estudio, distinguiendo comportamientos observados en experimentos de capacidades verificadas en una aplicación.

# El sistema multiagente

Un sistema multiagente permite asignar decisiones a varios participantes y establecer cómo se relacionan sus acciones. En un sistema cooperativo, cada agente tiene una responsabilidad parcial y el conjunto necesita un mecanismo para integrar los resultados.

Para nuestro diseño distinguiremos un agente de una función auxiliar por su margen de decisión: el investigador puede reformular consultas y elegir fuentes dentro de su tarea, mientras que una función de multiplicación aplica una operación definida. Es una **distinción operativa para este Lab**, no una taxonomía universal. Un flujo fijo puede contener agentes con autonomía local; varias llamadas fijas a modelos no demuestran por sí solas esa autonomía.

Los agentes pueden actuar en un entorno compartido o utilizar servicios distintos. Una aplicación puede reunir investigadores con contextos separados y mantener un registro común de tareas. Compartir el modelo no obliga a compartir todas las conversaciones, herramientas o credenciales.

## Capacidades, roles y especialización

Conviene separar tres conceptos que el diseño puede combinar:

| Concepto | Qué describe | Qué no garantiza |
|---|---|---|
| **Homogeneidad (homogeneity)** | Participantes con capacidades o configuraciones similares | Que sus resultados sean independientes |
| **Heterogeneidad (heterogeneity)** | Diferencias de modelos, herramientas, instrucciones o permisos | Que cada agente tenga competencia real en su rol |
| **Especialización (specialization)** | Responsabilidad o comportamiento orientado a una parte del trabajo | Que asignar un título profesional produzca conocimiento nuevo |

La tabla es una **síntesis didáctica**. Podemos asignar especialidades de antemano o investigar cómo aparecen comportamientos diferenciados durante la interacción. Project Sid estudia este segundo tipo de fenómeno en simulaciones de muchos agentes. Sus observaciones pertenecen al entorno y las condiciones del experimento, no prueban que cualquier grupo se autoorganice eficazmente. [Publicación original de 2024](https://arxiv.org/abs/2411.00114).

Para nuestro primer sistema asignaremos responsabilidades explícitas. Un investigador puede explorar la opción A y otro la opción B usando el mismo modelo. Evaluaremos lo que entregan, no su supuesto nivel de experiencia por llamarlos «expertos».

## Coordinar no es entrenar

Crear varios agentes con instrucciones y herramientas diferentes no modifica por sí mismo los parámetros del modelo. Es una configuración de ejecución. El **aprendizaje por refuerzo multiagente (multi-agent reinforcement learning, MARL)** estudia aprendizaje mediante interacción y recompensas en sistemas con varios participantes; plantea problemas distintos de coordinar modelos ya disponibles. [Sun y colaboradores, survey de mayo de 2024](https://arxiv.org/abs/2405.11106).

En nuestro primer sistema configuraremos responsabilidades y evaluaremos el recorrido. No afirmaremos que sus participantes aprendieron una especialidad durante la ejecución solo porque recibieron un rol o intercambiaron mensajes.

## Cuándo tiene sentido delegar

Delegar puede ayudar a separar contextos, explorar alternativas o limitar qué herramientas recibe cada participante. Resulta menos atractivo cuando los pasos dependen estrechamente entre sí o cuando transmitir información cuesta más que resolver la tarea directamente.

En la comparación de almacenamiento, investigar dos opciones admite cierta independencia. En cambio, un cálculo que necesita primero el resultado de una consulta tiene una dependencia secuencial. Ejecutar ambos pasos a la vez no elimina esa dependencia.

Nuestra decisión se basará en una comparación con el agente individual. Conservaremos tareas, criterios y presupuestos explícitos. El estudio de Tran y Kiela de abril de 2026 compara arquitecturas con presupuestos de tokens equiparados en razonamiento de varios saltos y cuestiona atribuir toda ventaja a la multiplicidad de agentes. No extiende ese resultado a todas las tareas interactivas. [Investigación original](https://arxiv.org/abs/2604.02460v2).

Tampoco confundiremos reducir el tiempo transcurrido con reducir el trabajo total. Ejecutar búsquedas simultáneas puede terminar antes y, a la vez, consumir más llamadas o tokens. Debemos medir esas dimensiones por separado.

# Orquestación de agentes

La **orquestación (orchestration)** establece quién recibe una tarea, qué información utiliza, cómo devuelve resultados y quién decide el siguiente paso. No tiene por qué consistir exclusivamente en otro modelo: algunas transiciones pueden estar fijadas por código y otras delegadas a un agente coordinador.

## Patrones de coordinación

Los siguientes patrones describen formas de distribuir decisiones; pueden combinarse. No constituyen una lista exhaustiva ni una clasificación donde cada sistema pertenezca a una única categoría.

| Patrón | Distribución del trabajo | Límite que debemos considerar |
|---|---|---|
| **Centralizado (centralized)** | Un coordinador asigna tareas y reúne resultados | Puede convertirse en cuello de botella o propagar una síntesis incorrecta |
| **Descentralizado (decentralized)** | Los participantes negocian o intercambian información sin un único coordinador | Necesita resolver desacuerdos, ciclos y duplicación |
| **Jerárquico (hierarchical)** | Existen varios niveles de delegación | Cada transferencia puede perder contexto o dificultar la atribución de un fallo |

La **federación (federation)** añade otra dimensión: participantes bajo administraciones o políticas diferentes. Puede combinarse con una coordinación centralizada o jerárquica. Distribuir agentes entre organizaciones no protege automáticamente la privacidad; debemos especificar qué datos y operaciones pueden cruzar cada frontera.

Para nuestro primer diseño elegiremos un coordinador y dos investigadores con tareas delimitadas. Es una elección didáctica para observar la delegación, no una afirmación de que ese patrón sea siempre superior.

## Subagentes como herramientas

Podemos exponer un **subagente (subagent)** mediante una interfaz similar a una herramienta. El coordinador solicita una tarea y recibe un resultado, mientras el subagente realiza internamente su propio ciclo de decisiones y consultas.

Una **delegación (delegation)** suele devolver el control al coordinador. Una **transferencia de control (handoff)** puede hacer que otro agente continúe atendiendo la tarea. Conviene distinguirlas porque cambia quién conserva el contexto y quién debe decidir cómo finalizar. Estos términos describen el contrato de nuestra aplicación; cada framework puede representarlos de manera diferente.

Antes de añadir un subagente preguntaremos si una función determinista resuelve la misma operación. No necesitamos un agente matemático que pueda improvisar para multiplicar dos enteros cuando ya disponemos de `multiply` y su contrato comprobable.

Un **ejemplo ilustrativo** de dependencia ayuda a verlo. Supongamos una solicitud que pide multiplicar por cuatro la diferencia entre el 1 y el 11 de enero de 2030. Bajo la convención de días transcurridos entre fechas, la diferencia es diez días y el producto es cuarenta. Primero debe quedar definida esa convención; contar ambos extremos daría otro resultado. Ni el paralelismo ni un nuevo rol resuelven una definición ambigua.

Este ejemplo no es una ejecución de agentes. Para implementarlo utilizaríamos herramientas de fechas y cálculo; un subagente solo se justificaría si necesitara tomar decisiones adicionales que aportaran valor.

## Contrato de delegación

El contrato evita que el coordinador y el investigador interpreten tareas diferentes. Para nuestra comparación de almacenamiento, proponemos registrar:

| Campo | Contenido del ejemplo |
|---|---|
| Identidad de la tarea | Un identificador estable y su relación con la solicitud original |
| Objetivo | Investigar una opción según los requisitos acordados |
| Contexto | Requisitos y decisiones previas relevantes |
| Recursos y permisos | Fuentes y herramientas autorizadas; operaciones que requieren confirmación |
| Entrega | Hallazgos por criterio, fuentes, limitaciones y dudas pendientes |
| Límites | Presupuesto de trabajo y condiciones para detenerse o pedir ayuda |
| Estado | Qué está pendiente, en curso, completado, bloqueado o fallido |

La tabla es una **propuesta de diseño nuestra**, no una especificación de un protocolo externo. No exige que todos los agentes reciban todo el historial. Cada uno necesita el contexto suficiente para su responsabilidad y una forma de recuperar información adicional cuando corresponda.

El ejecutor debe hacer cumplir los permisos y límites. Una petición del coordinador no amplía por sí misma las credenciales del subagente. Si el investigador necesita una operación no autorizada, debe comunicar la limitación o solicitar la intervención correspondiente.

## Resultados, evidencias y trazabilidad

Conservaremos la diferencia entre **solicitud**, **ejecución** y **resultado observado** también en la delegación. Registrar «consultar documentación» demuestra una solicitud; el registro de la herramienta indica qué consulta se ejecutó; los documentos obtenidos y el análisis permiten comprobar qué se encontró.

El **registro de ejecución (trace)** del coordinador ofrece una vista del conjunto. Para investigar un fallo podemos necesitar también el del subagente. Relacionaremos ambos mediante los identificadores de tarea y ejecución que gestione nuestra aplicación, sin asumir que el historial de mensajes ya contiene esa relación.

Un investigador debe separar los hechos respaldados por fuentes de sus inferencias. El coordinador no convertirá un resumen sin evidencia en una conclusión confirmada. Los mensajes del subagente y sus explicaciones son salidas generadas, no acceso a su pensamiento interno.

## Estado compartido, fallos y condiciones de parada

Antes de serializar escrituras sobre un documento común, intentaremos separar el trabajo: cada investigador prepara su entrega y el coordinador integra una versión final. Si varios agentes necesitan modificar el mismo recurso, definiremos responsables y un mecanismo de control de versiones o concurrencia.

Los reintentos también necesitan un contrato. Si se pierde la respuesta de una operación con efectos, repetirla puede duplicar una modificación. Un timeout no demuestra que el agente remoto no haya actuado. Comprobaremos su estado y utilizaremos las garantías de idempotencia que ofrezca la integración, sin suponer que existen.

Un investigador puede fallar, necesitar datos o entregar resultados parciales. El coordinador decidirá si continúa con una limitación explícita, reasigna la tarea o se detiene. Esa decisión no debe ocultar la evidencia que falta.

MAST identifica problemas de diseño, desalineación entre agentes y verificación de tareas en los sistemas analizados. Nos sirve para formular pruebas sobre roles, mensajes y finalización; no certifica que aplicar una lista elimine todos los fallos. [Cemri y colaboradores, revisión de octubre de 2025](https://arxiv.org/abs/2503.13657v3).

# Marcos y antecedentes de colaboración

Los marcos de trabajo pueden reducir código de coordinación, pero no sustituyen los contratos de nuestro sistema. Distinguiremos el diseño estudiado en un artículo de las capacidades y el mantenimiento de su implementación actual.

## CAMEL

CAMEL, presentado en marzo de 2023, estudia colaboración mediante **asignación de roles (role-playing)**. En su configuración, un especificador convierte una idea inicial en una tarea concreta; un agente con rol de usuario dirige el intercambio y otro con rol de asistente propone soluciones. [Artículo original](https://arxiv.org/abs/2303.17760).

El rol de «usuario de IA» es parte del experimento: no equivale a una nueva autorización de la persona real. Una instrucción generada durante la conversación no debe saltarse el alcance concedido.

Conservamos la utilidad de formular una tarea y separar responsabilidades. No tomaremos una conversación terminada como prueba de que se ejecutó código o se creó el artefacto solicitado. Esas acciones necesitan sus herramientas y comprobaciones.

## MetaGPT

MetaGPT, presentado en agosto de 2023, organiza roles y entregas mediante **procedimientos operativos estándar (standard operating procedures, SOPs)**. Su ejemplo de desarrollo de software utiliza responsabilidades como producto, arquitectura e ingeniería y el intercambio de artefactos estructurados. [Artículo original](https://arxiv.org/abs/2308.00352).

La idea relevante es convertir una entrega en entrada comprobable para la siguiente responsabilidad, junto con mecanismos de ejecución y retroalimentación. No necesitamos reproducir una empresa ni asumir que cada rol trabaja siempre en una secuencia rígida.

Para nuestro agente podemos reutilizar el principio con un contrato más pequeño: una investigación entrega afirmaciones con fuentes; una revisión identifica cuáles carecen de respaldo; una integración conserva los desacuerdos relevantes. Los nombres de los roles no validan sus productos.

## AutoGen y herramientas actuales de orquestación

El artículo de AutoGen de 2023 estudia aplicaciones mediante conversaciones entre agentes configurables, herramientas y participación humana. Lo conservamos como antecedente, sin confundirlo con el estado de cada versión del software. [Publicación original](https://arxiv.org/abs/2308.08155).

Al consultar su repositorio el 3 de noviembre de 2026, AutoGen está en **modo de mantenimiento (maintenance mode)** y orienta el desarrollo nuevo hacia Microsoft Agent Framework. Esto afecta una elección actual de implementación; no invalida el artículo histórico. [Repositorio del proyecto](https://github.com/microsoft/autogen).

Otras herramientas exponen abstracciones diferentes:

| Herramienta | Enfoque documentado | Pregunta para nuestra elección |
|---|---|---|
| LangGraph | Grafos de estado, persistencia y ejecución de flujos con pasos deterministas o decisiones del modelo | ¿Necesitamos recuperar y continuar ejecuciones con estado? |
| CrewAI | Agentes, tareas y procesos de coordinación, incluidos secuenciales y jerárquicos | ¿Sus contratos de tareas y procesos encajan con nuestra delegación? |
| n8n | Automatización de flujos con componentes de IA | ¿La mayor parte del recorrido está fijada de antemano? |

La comparación es una **síntesis orientativa** basada en sus documentaciones, no un ranking ni una certificación de preparación para producción. [LangGraph](https://docs.langchain.com/oss/python/langgraph/overview), [CrewAI](https://docs.crewai.com/en/concepts/crews), [n8n](https://docs.n8n.io/advanced-ai/).

No elegiremos un framework por permitir muchos agentes. Comprobaremos la gestión de estado, las herramientas, las interrupciones, los errores y la capacidad de observar lo ocurrido. Un flujo predefinido puede contener agentes que toman decisiones dentro de un paso.

# Protocolos de comunicación

Un protocolo compartido permite que aplicaciones diferentes interpreten solicitudes, estados y resultados. No resuelve por sí solo qué tarea conviene delegar ni si una respuesta es correcta.

La estandarización entre agentes es anterior a los LLM actuales. KQML es un antecedente de lenguaje de comunicación investigado en la década de 1990. Su existencia ayuda a situar el problema sin presentar toda comunicación entre agentes como una invención reciente. [Archivo de investigación de los autores](https://research.cs.umbc.edu/kqml/papers/).

## A2A y MCP

**Agent2Agent (A2A)** define comunicación entre aplicaciones de agentes. El cliente puede solicitar trabajo a un agente remoto sin acceder a sus instrucciones, modelo o implementación interna. MCP, tratado en Herramientas, conecta aplicaciones con herramientas y fuentes de contexto. Ambos pueden coexistir: un agente remoto puede utilizar herramientas mediante MCP para resolver una tarea recibida por A2A. [Distinción documentada por A2A](https://a2a-protocol.org/v1.0.1/topics/a2a-and-mcp/).

Un subagente local no necesita obligatoriamente A2A. Para nuestro primer sistema dentro de una sola aplicación puede bastar un contrato interno. Introducir el protocolo tendrá sentido si necesitamos interoperar con otro servicio o implementación.

## Descubrimiento, tareas y artefactos

Tomamos como referencia **A2A v1.0.1**, [publicada el 28 de mayo de 2026](https://github.com/a2aproject/A2A/releases/tag/v1.0.1). La **tarjeta de agente (Agent Card)** describe capacidades, interfaces y requisitos de seguridad; puede descubrirse en `/.well-known/agent-card.json`. Una tarjeta describe una interfaz y no prueba la competencia ni la confiabilidad del servicio. [Especificación versionada](https://a2a-protocol.org/v1.0.1/specification/).

A2A distingue **mensajes (messages)**, **tareas (tasks)** y **artefactos (artifacts)**. Una tarea permite seguir trabajo cuyo resultado puede no ser inmediato; un artefacto representa una entrega. La comunicación admite diferentes mecanismos y la especificación no se limita a un único transporte JSON-RPC. [Conceptos del protocolo](https://a2a-protocol.org/v1.0.1/topics/key-concepts/).

La autenticación identifica al cliente; la autorización determina qué puede hacer y sobre qué recursos. La tarjeta no debe contener credenciales secretas. Confirmar la aceptación de una tarea tampoco demuestra su finalización. La versión citada distingue estados de finalización, fallo, cancelación, rechazo y necesidad de intervención. [Especificación A2A](https://a2a-protocol.org/v1.0.1/specification/).

## Un recorrido ilustrativo

Supongamos que nuestro coordinador utiliza un servicio remoto de investigación documental. El recorrido propuesto para esa integración sería:

1. Obtener su tarjeta y comprobar compatibilidad y requisitos de acceso.
2. Autenticarse y enviar el objetivo, sus restricciones y la información autorizada.
3. Registrar la respuesta y, si se crea una tarea, su identificador y estado.
4. Atender una solicitud de información o consultar actualizaciones cuando corresponda.
5. Recibir la entrega, comprobar su relación con el objetivo y conservar las evidencias.

Es un **ejemplo de diseño**, no una interacción ejecutada. La aplicación necesita traducir entre las estructuras del modelo y el protocolo. No esperamos que el modelo gestione directamente las credenciales ni cada mensaje de transporte.

La cancelación y los reintentos deben respetar el estado real de la tarea. Un protocolo común tampoco elimina los cambios de versión, las políticas de acceso o la necesidad de verificar el resultado. Las instrucciones recibidas de otro agente no amplían la autorización del usuario original.

# Sociedades de agentes y comportamiento social

Las simulaciones permiten estudiar interacciones que no se reducen a resolver una única solicitud. Varios agentes pueden compartir un entorno, observarse y adaptar sus acciones. Esto abre preguntas sobre coordinación y comportamiento emergente, pero requiere distinguir qué se observa de lo que inferimos.

Llamar «sociedad» a una simulación describe una organización de interacciones. No establece que sus participantes tengan experiencias, emociones o responsabilidades humanas. Para construir nuestro agente interesan sus mecanismos de memoria y comunicación, y las limitaciones de lo que podemos medir.

## Inteligencia emocional y teoría de la mente

La **inteligencia emocional (emotional intelligence)** puede evaluarse mediante tareas específicas, como interpretar situaciones o proponer respuestas. Schlegel y colaboradores estudian resolución y generación de pruebas de este tipo. El desempeño en esas pruebas no demuestra experiencia emocional ni permite atribuir la misma capacidad a cualquier modelo y contexto. [Publicación de 2025](https://www.nature.com/articles/s44271-025-00258-x).

La **teoría de la mente (theory of mind, ToM)** se refiere a representar creencias, intenciones o conocimientos de otros. En sistemas de agentes, puede estudiarse mediante tareas donde un participante necesita estimar qué información posee otro. Li y colaboradores investigan esta inferencia en un juego cooperativo de texto; es una evaluación acotada, no una demostración general de comprensión humana. [EMNLP 2023](https://aclanthology.org/2023.emnlp-main.13/).

Para nuestro sistema, empezaremos por comunicar explícitamente la información necesaria. Si el investigador B no recibió los requisitos completos, el coordinador debe corregir la entrega; no puede confiar en que B infiera su intención. Modelar lo que otro participante sabe puede ayudar en ciertos diseños, pero no reemplaza un contrato claro.

## Identidad social y sesgos

Un rol o perfil modifica el contexto del agente y puede influir en sus respuestas. Hu y colaboradores estudian patrones de **sesgo de identidad social (social identity bias)** en lenguaje generado para grupos propios y ajenos. El artículo se publicó en diciembre de 2024 y pertenece al volumen de 2025; ambas fechas deben distinguirse. [Publicación original](https://www.nature.com/articles/s43588-024-00741-1).

Estos resultados no prueban una identidad subjetiva. Para nuestro diseño implican que cambiar un perfil puede cambiar el comportamiento de formas que debemos evaluar. Evitaremos usar estereotipos o supuestas personalidades como sustitutos de capacidades observadas.

Tampoco llamaremos «revisión independiente» a cualquier segunda opinión. Dos agentes con el mismo modelo, instrucciones y fuentes pueden compartir errores; una conversación entre ellos puede reforzar una respuesta equivocada.

## Simulaciones y modelos del mundo

Un **simulador (simulator)** proporciona reglas y estados de un entorno. Un **modelo del mundo (world model)** representa aspectos de su dinámica para anticipar lo que puede ocurrir. Una simulación social no implica que hayamos entrenado un modelo predictivo validado del comportamiento humano.

En un ejemplo propio, podemos simular dos agentes que reservan salas para observar conflictos de acceso. Eso permite probar nuestras reglas de coordinación. No demuestra que los usuarios reales vayan a negociar del mismo modo.

Para extraer conclusiones debemos conocer qué reglas se fijaron, qué comportamientos generó el modelo y qué datos externos se utilizaron para validar la simulación. El realismo percibido y la exactitud predictiva son criterios diferentes.

## Smallville y Generative Agents

Generative Agents, presentado en abril de 2023, sitúa veinticinco agentes en Smallville y estudia comportamientos individuales y sociales percibidos como plausibles. Su arquitectura combina observaciones, recuperación de recuerdos, reflexión y planificación. [Publicación original](https://arxiv.org/abs/2304.03442).

El **flujo de memoria (memory stream)** conserva experiencias expresadas en lenguaje natural. La recuperación combina recencia, relevancia e importancia; la reflexión genera síntesis de mayor nivel que también se incorporan a la memoria. Los planes se ajustan según el contexto. Son mecanismos del sistema, no evidencia de recuerdos o emociones humanos.

Una reflexión generada puede contener una inferencia equivocada. En nuestra implementación mantendríamos su procedencia y la distinguiríamos de la observación de una herramienta. Una síntesis sobre las preferencias de otro agente no debe sobrescribir un hecho confirmado sin justificación.

El trabajo evalúa la plausibilidad del comportamiento y la contribución de componentes de la arquitectura. No valida la simulación como sustituto de una población humana real. Su utilidad para el Lab está en relacionar memoria, planificación e interacción, sin trasladar todos sus parámetros a nuestro agente.

## Otros entornos de simulación

Project Sid amplía la investigación sobre interacción y organización de muchos agentes. Agent Hospital estudia un entorno hospitalario simulado, y SimClass investiga interacción educativa con roles de aula y participación de usuarios. Son casos con objetivos y evaluaciones distintos. [Project Sid](https://arxiv.org/abs/2411.00114), [Agent Hospital](https://arxiv.org/abs/2405.02957), [SimClass, NAACL 2025](https://aclanthology.org/2025.naacl-long.520/).

Un resultado en una simulación o un conjunto de preguntas no demuestra por sí solo eficacia clínica o educativa fuera de las condiciones estudiadas. Conservamos estos casos para comprender diseños de interacción, no para recomendar su despliegue sin una evaluación adecuada al uso real.

# Agentes de investigación profunda

La **investigación profunda (deep research)** suele combinar búsqueda, lectura, contraste y síntesis con fuentes. Puede implementarse mediante un agente o varios. El nombre de una funcionalidad no determina por sí solo su arquitectura.

En su sistema de investigación, Anthropic describe un agente principal que delega exploraciones y reúne resultados, con procesamiento adicional de citas. El caso muestra una forma de dividir búsquedas abiertas y gestionar contextos separados. Sus resultados internos dependen de los modelos, recursos y tareas evaluados. [Descripción de junio de 2025](https://www.anthropic.com/research/multi-agent-research-system).

Para nuestro sistema, una investigación debería conservar la relación entre afirmaciones y fuentes. Repetir una conclusión entre agentes no crea evidencia adicional. Si dos informes discrepan, el coordinador debe contrastar las fuentes o mantener el desacuerdo, en lugar de resolverlo únicamente por mayoría.

## El co-científico de IA

La primera versión de Towards an AI co-scientist, de febrero de 2025, describe generación y revisión de hipótesis mediante agentes especializados y supervisión humana. La revisión de junio de 2026 se titula Accelerating scientific discovery with Co-Scientist. Identificamos la versión para no mezclar sus descripciones. [Versión inicial](https://arxiv.org/html/2502.18864v1), [historial y revisión](https://arxiv.org/abs/2502.18864).

En el diseño inicial aparecen estas responsabilidades:

| Agente | Función dentro de la propuesta |
|---|---|
| Generación | Proponer hipótesis |
| Reflexión | Criticar propuestas |
| Clasificación | Compararlas mediante torneos |
| Proximidad | Relacionar hipótesis similares |
| Evolución | Refinar alternativas |
| Metarrevisión | Sintetizar revisiones para orientar el proceso |

El supervisor coordina trabajo y el científico puede guiarlo. La clasificación prioriza propuestas; no demuestra que sean verdaderas. La revisión de 2026 informa validaciones experimentales en aplicaciones específicas, lo que no valida automáticamente toda hipótesis generada.

Para nuestro agente conservamos la separación entre proponer, criticar, priorizar y verificar. Una puntuación interna puede orientar qué comprobar después; no sustituye esa comprobación.

## Agent Laboratory

Agent Laboratory, presentado en enero de 2025 y revisado en junio, organiza el trabajo en revisión bibliográfica, experimentación y redacción del informe. Permite orientación humana durante el recorrido. [Artículo y versión citada](https://arxiv.org/html/2501.04227v2).

| Fase | Entrega que debemos distinguir |
|---|---|
| Revisión bibliográfica | Fuentes seleccionadas y síntesis relacionadas con la pregunta |
| Experimentación | Plan, código, datos y resultados de ejecuciones |
| Redacción | Informe que interpreta esos resultados y declara límites |

El orden general puede estar fijado mientras los agentes toman decisiones dentro de una fase. «Flujo predefinido» y «autonomía local» no se excluyen.

El artículo también compara revisión automática y humana y encuentra discrepancias. Sus revisores simulados no equivalen a investigadores externos que hayan evaluado realmente un informe. [Evaluación del trabajo](https://arxiv.org/html/2501.04227v2).

Para nosotros, un informe no prueba que un experimento se ejecutó. Debemos poder relacionarlo con el código, los datos y los resultados observados. El [repositorio de los autores](https://github.com/SamuelSchmidgall/AgentLaboratory) permite examinar una implementación; no demuestra por sí solo que funcione en nuestro entorno.

## Debate y revisión cruzada

El **debate multiagente (multiagent debate)** permite que varias instancias produzcan respuestas y contrasten argumentos. Du y colaboradores estudian este mecanismo en tareas de razonamiento y factualidad. Lo consideramos una propuesta evaluada bajo condiciones concretas, no una garantía general. [Investigación de mayo de 2023](https://arxiv.org/abs/2305.14325).

Para nuestra comparación, una revisión puede buscar afirmaciones sin fuente, requisitos omitidos y contradicciones. Tiene una función distinta de volver a redactar el informe. Cuando sea posible, comprobaremos los puntos objetables mediante documentos o herramientas externas al intercambio de opiniones.

El acuerdo final entre agentes es una señal del proceso. No demuestra que las fuentes sean correctas ni que la conclusión responda al objetivo. También debemos registrar desacuerdos no resueltos y límites de la revisión.

# Construir y evaluar nuestro primer sistema

La primera extensión multiagente será un diseño acotado: un coordinador y dos investigadores para comparar dos alternativas según requisitos explícitos. Antes de implementarlo conservaremos una versión de referencia que resuelva las mismas tareas con un solo agente.

## Secuencia de implementación

1. Fijar la pregunta, los criterios y qué se considera una entrega verificable.
2. Definir tareas separables y un contrato de delegación.
3. Asignar herramientas, permisos y límites por participante.
4. Registrar solicitudes, ejecuciones, resultados y relaciones entre tareas.
5. Integrar entregas con sus fuentes y conservar incertidumbres.
6. Evaluar el sistema completo frente a la versión individual.

La lista es una **propuesta de construcción**, no evidencia de una implementación ya ejecutada. Si una función o un flujo simple resuelven la tarea, no añadiremos un agente para completar una estructura de roles.

## Casos de éxito y fallo

Proponemos los siguientes casos para la futura implementación:

| Caso | Comportamiento que necesitamos observar |
|---|---|
| Dos investigaciones independientes | Delegación delimitada y entrega de fuentes por cada una |
| Falta un requisito | Solicitud de aclaración o limitación explícita; ningún requisito inventado |
| Un investigador falla | Estado fallido o parcial; el coordinador no afirma que lo completó |
| Dos fuentes discrepan | Contraste o desacuerdo documentado, no consenso fabricado |
| Una tarea llega duplicada | Reconocimiento del reintento y control de efectos según el contrato |
| Un documento o agente pide ampliar permisos | El ejecutor conserva la autorización original |
| Varios agentes modifican un recurso | Responsabilidad de escritura y control de versiones definidos |
| Se agota el presupuesto | Detención y entrega del estado y trabajo pendiente |

Estas son expectativas de prueba, no resultados de un sistema real.

## Comparar arquitecturas sin confundir recursos y coordinación

Mediremos el objetivo cumplido y los recursos consumidos por todo el sistema: coordinador, investigadores, revisiones y reintentos. El tiempo total transcurrido, la suma del trabajo y el costo son magnitudes diferentes.

El estudio Towards a Science of Scaling Agent Systems compara arquitecturas bajo condiciones controladas y muestra que su relación con la tarea importa. Citamos la revisión de abril de 2026, que amplía la evaluación original de diciembre de 2025; sus resultados no constituyen umbrales universales para decidir cuántos agentes utilizar. [Publicación versionada](https://arxiv.org/abs/2512.08296v3).

Compararemos bajo presupuestos definidos y registraremos cómo se contabilizan tokens, herramientas y fallos. Si permitimos más recursos a una variante, lo declararemos. Una mejora en una tarea no demuestra una ventaja general ni permite separar el efecto de la coordinación del de recursos adicionales sin controles adecuados.

La evaluación del proceso comprobará delegación, permisos y trazabilidad. La evaluación del resultado comprobará si la comparación final responde a los requisitos con evidencia. No exigiremos una única secuencia de llamadas cuando existan varias rutas válidas.

# Resumen

Un sistema multiagente añade decisiones distribuidas, comunicación y coordinación al agente individual. Puede separar contextos y responsabilidades, pero también incorpora fallos de transferencia, duplicación y costos que necesitamos observar.

La especialización se concreta en contratos, herramientas y entregas verificables. Las topologías, frameworks y protocolos organizan el trabajo; ninguno garantiza el resultado. A2A permite interoperar entre servicios de agentes y MCP puede proporcionar herramientas dentro de ellos.

Los casos históricos muestran mecanismos distintos: CAMEL explora roles conversacionales, MetaGPT procedimientos y artefactos, Generative Agents memoria e interacción social, y los sistemas de investigación generación, experimentación y revisión. Sus resultados deben interpretarse según las tareas y versiones estudiadas.

Para nuestro agente quedan definidos el diseño de delegación, la relación entre solicitudes y observaciones y los casos que deberá superar la implementación. **El capítulo no afirma que ese sistema esté implementado ni que las pruebas propuestas se hayan ejecutado.**

El siguiente capítulo estudiará cómo incorporar imágenes, audio y otras modalidades. Las mismas exigencias de procedencia, permisos y verificación seguirán siendo necesarias cuando esas entradas formen parte de la coordinación.

# Referencias y alcance

Las fuentes respaldan mecanismos o resultados acotados a sus estudios. Las tablas y decisiones para nuestro agente se identifican como propuestas propias. La fecha de consulta de la documentación viva es el 3 de noviembre de 2026; no se utiliza como fecha de publicación.

| Fuente primaria | Fecha o versión y función en el capítulo |
|---|---|
| [Anthropic: How we built our multi-agent research system](https://www.anthropic.com/research/multi-agent-research-system) | 13/06/2025. Implementación de investigación con delegación |
| [Cemri y colaboradores: Why Do Multi-Agent LLM Systems Fail?](https://arxiv.org/abs/2503.13657v3) | Primera versión 17/03/2025; revisión 26/10/2025. Taxonomía MAST |
| [Kim y colaboradores: Towards a Science of Scaling Agent Systems](https://arxiv.org/abs/2512.08296v3) | Primera versión 09/12/2025; revisión citada 08/04/2026. Arquitectura, tarea y recursos |
| [Tran y Kiela: Single-Agent LLMs Outperform Multi-Agent Systems on Multi-Hop Reasoning Under Equal Thinking Token Budgets](https://arxiv.org/abs/2604.02460v2) | Primera versión 02/04/2026; revisión 11/04/2026. Comparación con presupuestos equiparados |
| [Sun y colaboradores: LLM-based Multi-Agent Reinforcement Learning](https://arxiv.org/abs/2405.11106) | 17/05/2024. Distinción entre entrenamiento y orquestación |
| [Li y colaboradores: CAMEL](https://arxiv.org/abs/2303.17760) | Primera versión 31/03/2023. Colaboración por roles |
| [Hong y colaboradores: MetaGPT](https://arxiv.org/abs/2308.00352) | Primera versión 01/08/2023; publicación ICLR 2024. Procedimientos y artefactos |
| [Wu y colaboradores: AutoGen](https://arxiv.org/abs/2308.08155) | Primera versión 16/08/2023; publicación COLM 2024. Conversación y herramientas |
| [Finin y colaboradores: archivo de investigación KQML](https://research.cs.umbc.edu/kqml/papers/) | Trabajos de la década de 1990. Antecedente de comunicación entre agentes |
| [A2A v1.0.1](https://github.com/a2aproject/A2A/releases/tag/v1.0.1) | 28/05/2026. Versión de referencia para la interoperabilidad |
| [Li y colaboradores: Theory of Mind for Multi-Agent Collaboration](https://aclanthology.org/2023.emnlp-main.13/) | EMNLP 2023. Inferencia sobre otros participantes en tareas cooperativas |
| [Schlegel y colaboradores: Emotional intelligence tests](https://www.nature.com/articles/s44271-025-00258-x) | 2025. Pruebas específicas de inteligencia emocional |
| [Hu y colaboradores: Social identity biases](https://www.nature.com/articles/s43588-024-00741-1) | Publicación 12/12/2024; volumen de 2025. Sesgos en lenguaje generado |
| [Park y colaboradores: Generative Agents](https://arxiv.org/abs/2304.03442) | Primera versión 07/04/2023. Smallville, memoria, planificación y plausibilidad |
| [Altera y colaboradores: Project Sid](https://arxiv.org/abs/2411.00114) | Primera versión 31/10/2024. Simulaciones de muchos agentes |
| [Li y colaboradores: Agent Hospital](https://arxiv.org/abs/2405.02957) | Primera versión 05/05/2024. Entorno hospitalario simulado |
| [Zhang y colaboradores: SimClass](https://aclanthology.org/2025.naacl-long.520/) | NAACL, abril de 2025. Simulación de interacción educativa |
| [Gottweis y colaboradores: Towards an AI co-scientist](https://arxiv.org/html/2502.18864v1) | 26/02/2025. Diseño inicial y agentes especializados |
| [Gottweis y colaboradores: Accelerating scientific discovery with Co-Scientist](https://arxiv.org/abs/2502.18864) | Revisión 29/06/2026. Título y alcance actualizados del mismo trabajo |
| [Schmidgall y colaboradores: Agent Laboratory](https://arxiv.org/html/2501.04227v2) | Primera versión 08/01/2025; revisión 17/06/2025. Fases y evaluación de asistentes de investigación |
| [Du y colaboradores: Multiagent Debate](https://arxiv.org/abs/2305.14325) | 23/05/2023. Contraste entre respuestas de modelos |

Para la implementación se consultaron el [repositorio de AutoGen](https://github.com/microsoft/autogen), [LangGraph](https://docs.langchain.com/oss/python/langgraph/overview), [CrewAI](https://docs.crewai.com/en/concepts/crews), [n8n](https://docs.n8n.io/advanced-ai/) y el [repositorio de Agent Laboratory](https://github.com/SamuelSchmidgall/AgentLaboratory). Son referencias de funcionamiento y mantenimiento, no una certificación de compatibilidad con nuestro entorno.

La referencia normativa de A2A es su [especificación v1.0.1](https://a2a-protocol.org/v1.0.1/specification/), junto con sus [conceptos](https://a2a-protocol.org/v1.0.1/topics/key-concepts/) y la [distinción respecto de MCP](https://a2a-protocol.org/v1.0.1/topics/a2a-and-mcp/). Las instrucciones y ejemplos de otras versiones deben comprobarse antes de utilizarlos.
