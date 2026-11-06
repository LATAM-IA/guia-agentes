La memoria permite que un agente continúe una tarea sin tener que empezar de cero en cada interacción. Sirve para conservar lo que pidió el usuario, registrar acciones realizadas y recuperar información que hará falta después.

Imagina que le pides a un agente de programación que corrija un error. Primero revisa un archivo y después ejecuta una prueba. Para decidir cómo seguir, necesita saber qué encontró y cuál fue el resultado. Si esa información no está disponible en la siguiente llamada al modelo, podría repetir el trabajo o tomar una decisión que contradiga lo que acaba de comprobar.

**El modelo utiliza la información que recibe en cada llamada. El sistema que lo rodea se encarga de conservarla y volver a proporcionársela cuando corresponde.** Esa información puede venir del historial reciente, de un registro de ejecución o de un almacenamiento que persiste entre sesiones.

Construir memoria requiere decidir qué vale la pena guardar y cuándo recuperarlo. Una instrucción corregida puede reemplazar a la anterior; un resultado antiguo puede haber dejado de ser válido. Conservar todo tampoco asegura que el modelo encuentre lo relevante.

En este capítulo empezaremos por mantener el historial de una conversación. Después veremos cómo gestionar su crecimiento y recuperar información almacenada fuera de ella.

**Guardar experiencias no modifica por sí mismo los pesos del modelo.** Puede ayudar al agente a resolver tareas futuras si recupera esas experiencias y las usa de manera adecuada.

# Tipos de memoria (types of memory)

Distinguir tipos de memoria ayuda a decidir qué información necesita el agente y cómo utilizarla. Podemos organizarla según la función que cumple: mantener el trabajo actual, recuperar experiencias, consultar conocimientos o seguir un procedimiento.

Imagina un agente que debe corregir un error en una aplicación:

| Tipo | Qué conserva | Ejemplo |
|---|---|---|
| **Memoria de trabajo (working memory)** | Información necesaria para continuar la tarea actual. | El error informado, los archivos revisados y el resultado de la última prueba. |
| **Memoria episódica (episodic memory)** | Registros de experiencias anteriores, incluidas las acciones realizadas y sus resultados. | Una corrección anterior de un error similar y las pruebas que permitieron comprobarla. |
| **Memoria semántica (semantic memory)** | Hechos y conocimientos que pueden utilizarse en distintas tareas. | La documentación del proyecto y las reglas de su API. |
| **Memoria procedimental (procedural memory)** | Instrucciones sobre cómo realizar una tarea. | El procedimiento del proyecto para reproducir un error, corregirlo y verificar la solución. |

Esta clasificación toma conceptos de la memoria humana para describir funciones del sistema, como propone [CoALA, de Sumers et al. (2024)](https://openreview.net/forum?id=1i6ZCvflQJ). **No significa que el agente recuerde como una persona ni que necesite una base de datos para cada tipo de memoria.** Un mismo documento puede contener conocimientos y procedimientos.

También conviene distinguir la memoria externa del conocimiento incorporado durante el entrenamiento. La **memoria paramétrica (parametric memory)** es la información codificada en los pesos del modelo. Puede contribuir a generar una respuesta, pero no funciona como un registro que podamos consultar y actualizar con precisión.

La **memoria externa (external memory)** se conserva fuera de esos pesos: en mensajes, archivos, bases de datos o registros de ejecución. El sistema selecciona la información relevante y la proporciona al modelo cuando la necesita. Guardar un dato allí no implica que el modelo lo reciba automáticamente.

Para comenzar a construir nuestro agente, nos centraremos en la memoria de trabajo: conservar los mensajes y resultados necesarios para continuar una conversación. Más adelante incorporaremos mecanismos para recuperar información entre tareas y sesiones.

# Memoria a corto plazo (short-term memory)

La memoria a corto plazo mantiene disponible la información que el agente necesita para continuar una conversación o una tarea en curso. Puede incluir mensajes recientes, decisiones tomadas y resultados de herramientas.

Por ejemplo:

> **Usuario:** Necesito comparar dos departamentos. Mi presupuesto máximo es de 800 dólares mensuales.
>
> **Asistente:** ¿En qué zona estás buscando?
>
> **Usuario:** Cerca de la universidad.

Para responder a ese último mensaje, el modelo necesita conocer la solicitud inicial y el presupuesto. Si recibe únicamente «Cerca de la universidad», le faltará información para continuar la comparación.

El sistema puede conservar los mensajes e incluirlos en la siguiente llamada al modelo. En una tarea con herramientas, también debe conservar los resultados relevantes: qué departamentos encontró, cuánto cuestan y cuáles ya descartó.

**La información guardada solo ayuda al modelo si está disponible en el contexto de la llamada actual.** Registrar un resultado en un archivo o una base de datos no significa que el modelo pueda utilizarlo automáticamente.

Esta memoria tiene un límite práctico: el contexto admite una cantidad determinada de tokens. A medida que avanza la tarea, tendremos que decidir qué conservar completo, qué resumir y qué recuperar cuando haga falta.

Empezaremos por el mecanismo más sencillo: mantener el historial de mensajes de la conversación.

## Memoria de conversación (conversation memory)

La memoria de conversación permite que el agente utilice lo que se ha dicho durante el diálogo. El sistema conserva los mensajes y los incluye en las siguientes llamadas al modelo.

Cada mensaje identifica quién lo envió. En una conversación básica usamos los roles `user`, para el usuario, y `assistant`, para el modelo. Las instrucciones generales pueden enviarse mediante un mensaje `system`, según la interfaz utilizada.

Por ejemplo, después de este intercambio:

> **Usuario:** Busco un departamento cerca de la universidad. Mi presupuesto máximo es de 800 dólares mensuales.
>
> **Asistente:** ¿Cuántas habitaciones necesitas?
>
> **Usuario:** Dos habitaciones.

El historial podría representarse así:

```json
[
  {
    "role": "user",
    "content": "Busco un departamento cerca de la universidad. Mi presupuesto máximo es de 800 dólares mensuales."
  },
  {
    "role": "assistant",
    "content": "¿Cuántas habitaciones necesitas?"
  },
  {
    "role": "user",
    "content": "Dos habitaciones."
  }
]
```

Al enviar estos mensajes al modelo, la nueva respuesta puede tener en cuenta la ubicación, el presupuesto y el número de habitaciones.

Para incorporar este mecanismo a nuestro agente:

1. Guardamos el mensaje del usuario.
2. Construimos la solicitud con las instrucciones y el historial disponible.
3. Llamamos al modelo.
4. Guardamos su respuesta para la siguiente interacción.

**Cada mensaje debe conservar su rol y su orden.** Una respuesta del asistente podría contener un error; guardarla en el historial no la convierte en información verificada. Cuando incorporemos herramientas, también distinguiremos las solicitudes del modelo de los resultados que devuelven esas herramientas.

Conservar el historial en una lista permite mantener la conversación mientras el programa está en ejecución. Para recuperarla después de cerrar o reiniciar la aplicación, necesitamos guardarla en un almacenamiento persistente y asociarla con la conversación correspondiente.

El historial crecerá con cada intercambio. **No siempre podremos enviarlo completo:** tendremos que reservar espacio para la respuesta y seleccionar qué información sigue siendo necesaria. En los siguientes apartados veremos cómo recortarlo y resumirlo.

## Recorte del historial (trimming)

El recorte del historial permite limitar la cantidad de información que enviamos al modelo. Una estrategia sencilla consiste en conservar las interacciones más recientes y retirar las antiguas cuando el contexto supera el presupuesto disponible.

Imagina que, al comenzar una conversación, el usuario establece un presupuesto máximo de 800 dólares para alquilar un departamento. Después compara varias opciones y pregunta:

> «¿Cuál de los dos últimos departamentos me recomiendas?»

Si conservamos únicamente los mensajes recientes, podríamos mantener las descripciones de ambos departamentos y perder el presupuesto inicial. El modelo tendría información para compararlos, pero le faltaría una restricción que sigue vigente.

**Lo más reciente no siempre es lo más relevante.** El recorte reduce el contexto, pero puede eliminar información necesaria para resolver la tarea.

Para incorporarlo a nuestro agente, podemos separar:

- **Las instrucciones y los datos vigentes:** el objetivo, las restricciones y las decisiones que todavía afectan a la tarea.
- **El historial reciente:** las últimas interacciones necesarias para continuar el diálogo.

También debemos definir un presupuesto de tokens. Conservar los últimos diez mensajes no asegura que quepan: una respuesta breve y el contenido de un archivo pueden ocupar cantidades muy diferentes de espacio. Debemos considerar el conjunto de la solicitud y reservar capacidad para la respuesta.

Cuando haya herramientas, el recorte debe mantener la relación entre una llamada y su resultado. Eliminar una parte de ese intercambio puede dejar un historial incompleto o incompatible con la interfaz del modelo.

**Recortar lo que recibe el modelo no exige borrar el registro completo.** Podemos conservar la conversación y los eventos de ejecución en un almacenamiento separado para consultarlos o investigar errores. Ese registro no tiene que enviarse entero en cada llamada.

El recorte es un primer mecanismo para controlar el crecimiento del contexto. Cuando necesitamos conservar información de interacciones antiguas en menos espacio, podemos recurrir al resumen.

## Resumen del historial (conversation summarization)

Resumir el historial permite conservar información de una conversación utilizando menos tokens. En lugar de enviar todos los mensajes anteriores, el sistema prepara una versión condensada con lo necesario para continuar la tarea.

Por ejemplo, después de comparar varios departamentos, podríamos conservar este resumen:

> El usuario busca un departamento cerca de la universidad, con dos habitaciones y un presupuesto máximo de 800 dólares mensuales. Descartó la opción A porque cuesta 950 dólares. Las opciones B y C siguen pendientes de comparación. Falta confirmar si sus precios incluyen gastos comunes.

El resumen conserva las restricciones, las decisiones y lo que todavía falta resolver. No necesita reproducir cada pregunta y respuesta.

Una estrategia consiste en **resumir las interacciones antiguas y mantener completos los mensajes recientes**. En cada llamada, el modelo recibe ese resumen, los intercambios recientes y la nueva consulta.

Para incorporarlo a nuestro agente:

1. Definimos qué debe conservar el resumen: objetivo, restricciones vigentes, decisiones, resultados relevantes y asuntos pendientes.
2. Lo generamos o actualizamos cuando el historial alcanza un presupuesto de tokens.
3. Conservamos los mensajes recientes completos para mantener el detalle del intercambio actual.
4. Guardamos el registro original por separado para consultar detalles y comprobar posibles errores.

El mismo modelo u otro pueden producir el resumen. Hacerlo requiere procesamiento adicional, por lo que no es necesario actualizarlo después de cada mensaje.

**Un resumen es una interpretación del historial, no una copia exacta.** Puede omitir una condición, alterar una cifra o presentar una suposición como un hecho. Si volvemos a resumir una versión que ya contiene un error, podemos arrastrarlo a las siguientes interacciones.

Por eso conviene conservar literalmente los datos que no admiten aproximaciones, como identificadores, importes o mensajes de error. También debemos distinguir lo confirmado de lo pendiente y registrar las correcciones del usuario.

El resumen debe proporcionarse al modelo como contexto de la conversación. **Su contenido no debe adquirir la autoridad de las instrucciones del sistema:** resumir una petición del usuario o un resultado externo no cambia su origen ni su nivel de confianza.

# Memoria a largo plazo (long-term memory)

La memoria a largo plazo permite conservar información para utilizarla en futuras tareas o sesiones. Evita que el agente dependa únicamente del historial de la conversación actual.

Por ejemplo, un agente de programación podría recuperar una decisión tomada semanas atrás: el proyecto utiliza un proveedor de autenticación determinado y cualquier cambio debe mantener la compatibilidad con él. Esa decisión sigue siendo relevante aunque la conversación donde se acordó ya no esté en el contexto.

La información puede proceder de experiencias anteriores del agente o de fuentes que existen independientemente de él, como la documentación del proyecto. **Recuperar un documento externo y recordar una experiencia propia son operaciones distintas**, aunque ambas pueden proporcionar contexto útil.

Para incorporar esta memoria necesitamos resolver dos procesos:

- **Almacenamiento:** qué información conservamos, con qué origen y cómo la actualizamos o eliminamos.
- **Recuperación:** cómo encontramos la información pertinente y la incorporamos a la llamada actual al modelo.

La memoria también puede ofrecerse como una herramienta. El agente puede solicitar que se guarde una decisión, consultar una nota de una sesión anterior o actualizar un dato corregido. La aplicación ejecuta esas operaciones y controla qué información puede leer o modificar. **El modelo solicita la operación; el sistema debe confirmar si se realizó correctamente.** Esto permite recuperar información cuando hace falta, sin incluir toda la memoria en cada llamada. La [documentación de Memory tool de Anthropic](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool) muestra una implementación de este mecanismo con archivos persistentes.

El almacenamiento puede ser un archivo, una base de datos u otro sistema adecuado para la aplicación. No requiere necesariamente una base de datos vectorial: podemos recuperar información mediante identificadores, filtros, búsqueda de texto o similitud semántica.

**Persistir un dato no garantiza que el agente lo encuentre ni que siga siendo válido.** Conviene conservar su fuente y fecha, distinguir hechos de suposiciones y permitir que una corrección reemplace información anterior. Si la aplicación atiende a varios usuarios, la recuperación también debe respetar sus permisos de acceso.

Empezaremos por una técnica para seleccionar información externa e incorporarla al contexto del modelo: la generación aumentada por recuperación.

## Generación aumentada por recuperación (retrieval-augmented generation, RAG)

RAG permite responder preguntas utilizando información externa al modelo, como documentos de una organización, archivos de un proyecto o registros almacenados. El sistema busca información pertinente y la incorpora al contexto antes de generar una respuesta. El trabajo de [Lewis et al. (2020)](https://arxiv.org/abs/2005.11401) es una referencia fundacional de este enfoque.

Por ejemplo, un usuario pregunta:

> «¿Cuántos días tengo para solicitar el reembolso de un curso?»

El sistema busca la política de reembolsos vigente y recupera este fragmento:

> «Las solicitudes de reembolso se aceptan hasta 14 días después de la compra, siempre que el participante no haya iniciado el curso».

El modelo recibe la pregunta y ese fragmento. Puede responder indicando el plazo y la condición, junto con una referencia al documento consultado. No necesita haber aprendido esa política durante su entrenamiento.

### Preparar la información

Si trabajamos con documentos propios, primero necesitamos hacerlos consultables. Esto puede incluir:

- Extraer su contenido y dividir los documentos extensos en fragmentos (*chunks*).
- Conservar la fuente, la fecha, la versión y los permisos de acceso.
- Crear un índice que permita encontrar los fragmentos relevantes.

Una opción es utilizar **representaciones vectoriales (embeddings)**: un modelo convierte el texto en vectores numéricos que permiten buscar similitudes de significado. También podemos utilizar búsqueda por palabras clave, filtros o una combinación de métodos.

**RAG no exige una base de datos vectorial.** Si la aplicación ya dispone de un buscador o una base de datos consultable, podemos aprovecharlos. La [documentación de recuperación de LangChain](https://docs.langchain.com/oss/python/deepagents/retrieval) describe estas alternativas.

### Recuperar y responder

En una implementación básica, el proceso sigue estos pasos:

1. Recibir la pregunta y construir una consulta de búsqueda.
2. Recuperar información de las fuentes permitidas.
3. Seleccionar los fragmentos pertinentes para la pregunta.
4. Incorporarlos al contexto, junto con sus referencias.
5. Generar una respuesta basada en la información disponible.

La selección importa tanto como la búsqueda. Un documento puede tratar sobre reembolsos y pertenecer a otro producto o a una versión antigua de la política.

**Similitud no significa evidencia suficiente.** Recuperar los tres resultados más cercanos no demuestra que alguno contenga la respuesta. Si falta información, el sistema debe poder reconocerlo, solicitar una aclaración o realizar otra búsqueda.

### Comprobar que funciona

Para evaluar nuestro sistema debemos revisar por separado:

| Qué evaluamos | Qué necesitamos comprobar |
|---|---|
| **Recuperación** | Si encontró los documentos y fragmentos necesarios, correspondientes a la versión y al ámbito correctos. |
| **Respuesta** | Si respondió correctamente, respetó las condiciones de las fuentes y evitó añadir afirmaciones sin respaldo. |

En el ejemplo, encontrar la política correcta no basta si la respuesta omite que el curso no debe haberse iniciado. Del mismo modo, una respuesta bien redactada no compensa haber recuperado una política equivocada.

**RAG aporta información para fundamentar la respuesta; no garantiza que sea correcta.** Mantendremos las referencias para poder comprobar qué fuente respalda cada afirmación.

En nuestro agente comenzaremos con un flujo sencillo: buscar antes de responder. Más adelante veremos cómo permitir que el propio agente decida cuándo necesita consultar información y si debe continuar buscando.

# Gestión de la memoria persistente (persistent memory management)

La memoria persistente permite reutilizar información entre sesiones. Para que resulte útil, necesitamos seleccionar qué guardar, recuperarlo cuando haga falta y mantenerlo actualizado. No existe una arquitectura única: la documentación de Anthropic y LangChain muestra distintas implementaciones de este patrón.

| Decisión | Cómo se aplica |
|---|---|
| **Qué guardar** | Preferencias explícitas, decisiones confirmadas y resultados reutilizables. |
| **Cómo separarlo** | Memoria de la conversación, del usuario y del proyecto, con los permisos correspondientes. |
| **Cuándo recuperarlo** | Cargar lo imprescindible al inicio y consultar el resto mediante herramientas o búsquedas. |
| **Cómo actualizarlo** | Corregir datos, reemplazar decisiones anteriores y consolidar información repetida. |
| **Qué puede modificarse** | Distinguir notas editables de políticas o instrucciones que el agente solo puede leer. |

LangGraph separa el estado de una conversación de la memoria entre sesiones. Deep Agents permite archivos persistentes, lectura bajo demanda y actualización durante la conversación o mediante consolidación posterior. Estas son opciones de implementación, no requisitos para todos los agentes. Véanse la [descripción de memoria de LangGraph](https://docs.langchain.com/oss/python/concepts/memory) y la [documentación de memoria de Deep Agents](https://docs.langchain.com/oss/python/deepagents/memory).

Anthropic ofrece operaciones para leer, crear, actualizar y eliminar archivos de memoria. La aplicación controla el almacenamiento y ejecuta las solicitudes del modelo. **Solicitar que se guarde un dato no demuestra que se haya guardado:** el sistema debe registrar el resultado de la operación. Véase la [documentación de Memory tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool).

Para nuestro agente empezaremos con notas pequeñas y comprobables por usuario o proyecto, recuperadas cuando sean pertinentes. Conservaremos su fuente y permitiremos corregirlas. Por ejemplo, una decisión sobre el proveedor de autenticación pertenece al proyecto; una preferencia personal sobre el idioma de las respuestas pertenece al usuario.

**La frecuencia de uso no determina por sí sola qué conservar.** Una restricción puede seguir vigente aunque no se haya consultado durante semanas. Definiremos las reglas de actualización y eliminación según la aplicación, en lugar de asumir que toda información poco utilizada debe olvidarse.

## RAG agéntico (agentic RAG)

RAG agéntico permite que el agente decida qué información buscar y si necesita realizar nuevas consultas antes de responder. Es útil cuando una pregunta requiere combinar fuentes o cuando una búsqueda revela datos que necesitan comprobarse.

En un flujo básico de RAG, el sistema recupera información antes de generar la respuesta. En un flujo agéntico, el modelo puede solicitar búsquedas durante la resolución de la tarea y utilizar sus resultados para decidir el siguiente paso.

Por ejemplo, un usuario pregunta:

> «Compré el curso hace diez días y todavía no lo he iniciado. ¿Puedo solicitar un reembolso?»

El agente podría seguir esta secuencia:

1. Consultar la política de reembolsos vigente.
2. Identificar que el plazo y el inicio del curso son condiciones necesarias.
3. Consultar el registro de la compra y el estado del curso, si dispone de acceso autorizado.
4. Comparar los datos con la política.
5. Responder con las condiciones verificadas y sus fuentes.

Si no puede acceder al registro de la compra, debe explicar qué falta comprobar. **No encontrar un dato no permite asumir que se cumple una condición.**

### Cómo incorporarlo al agente

Ofrecemos la búsqueda como una herramienta. El modelo solicita una consulta; la aplicación la ejecuta y devuelve los resultados. Después, el modelo puede responder, pedir una aclaración o solicitar otra búsqueda.

Las fuentes pueden ser documentos, bases de datos, buscadores o APIs. No necesitamos varias fuentes ni varios agentes para implementar este mecanismo. La documentación de LangChain describe este patrón como recuperación mediante herramientas controlada por el agente. [RAG agéntico en LangChain](https://docs.langchain.com/oss/python/deepagents/retrieval).

Para nuestro agente empezaremos con una herramienta de búsqueda y un límite de consultas. Registraremos qué solicitó, qué resultados recibió y qué fuentes utilizó al responder.

**Más búsquedas no garantizan una respuesta mejor.** Debemos comprobar si cada consulta aporta evidencia necesaria y detener el proceso cuando podamos responder con respaldo, necesitemos información del usuario o alcancemos el presupuesto de ejecución.

### Cuándo conviene utilizarlo

| Situación | Punto de partida |
|---|---|
| La pregunta suele resolverse consultando una fuente conocida. | Un flujo fijo de búsqueda y respuesta. |
| La siguiente búsqueda depende de lo encontrado en la anterior. | RAG agéntico. |
| Hay tareas de recuperación independientes que justifican especialización o ejecución paralela. | Evaluar varios agentes y comparar su beneficio con el coste y la complejidad. |

La elección debe comprobarse con preguntas representativas. Evaluaremos la calidad de la respuesta, la evidencia recuperada y el tiempo y coste necesarios para obtenerla.

### A-MEM: notas relacionadas como memoria

A-MEM es una propuesta de investigación publicada en NeurIPS 2025 que organiza la memoria mediante notas relacionadas y actualizables. Se inspira en Zettelkasten, un método que conecta notas centradas en una idea. Es un ejemplo de arquitectura, no un paso obligatorio para construir nuestro agente. Véase la [publicación de A-MEM](https://papers.nips.cc/paper_files/paper/2025/hash/19909c36f51abc4856b4560aff3d36d6-Abstract-Conference.html).

Al incorporar una experiencia, el sistema crea una nota con su contenido, fecha y descripciones generadas por el modelo, como etiquetas y palabras clave. Busca recuerdos similares y utiliza el modelo para decidir qué vínculos establecer. La nueva información también puede motivar actualizaciones en las descripciones de notas anteriores.

Por ejemplo, un agente de programación podría relacionar una decisión de almacenar sesiones en Redis, un incidente que impidió validarlas y la corrección comprobada mediante una prueba. Recuperar una de esas notas podría conducir a las demás.

**Una relación propuesta por el modelo no demuestra una relación causal.** Las etiquetas y conexiones son información derivada y deben distinguirse de los registros originales.

Para nuestro agente empezaremos con notas pequeñas, identificadores y fuentes. Evaluaremos las relaciones automáticas si la búsqueda sencilla resulta insuficiente y las pruebas muestran una mejora que justifique su coste y complejidad.

### Search-o1: buscar mientras se resuelve una tarea

Search-o1 combina modelos de razonamiento con búsqueda externa. Sirve para tareas en las que no podemos anticipar toda la información necesaria: una primera búsqueda puede revelar una nueva pregunta que debemos resolver antes de responder.

Su propuesta incorpora un módulo llamado **Reason-in-Documents**, que analiza los documentos recuperados según la consulta y el estado de la resolución. El objetivo es extraer información útil para continuar, en lugar de incorporar documentos completos con contenido irrelevante. [Search-o1, EMNLP 2025](https://aclanthology.org/2025.emnlp-main.276/).

Por ejemplo, un agente investiga por qué dejó de funcionar una integración después de actualizar una biblioteca:

1. Consulta las notas de la versión y encuentra que cambió el mecanismo de autenticación.
2. Busca la guía de migración correspondiente.
3. Extrae los cambios necesarios y los compara con el código del proyecto.
4. Si falta confirmar una opción de configuración, realiza una consulta específica antes de proponer la corrección.

Este ejemplo ilustra el patrón; no es un resultado experimental del artículo.

**Cada búsqueda debe responder a una necesidad concreta de información.** Recuperar más documentos sin seleccionar lo pertinente puede consumir contexto y dificultar la tarea.

Para nuestro agente podemos adoptar ese principio mediante un bucle: identificar qué falta, solicitar una búsqueda, examinar el resultado y decidir cómo continuar. No necesitamos reproducir los marcadores internos de Search-o1; utilizaremos el mecanismo de herramientas compatible con el modelo elegido.

Al extraer o condensar información, conservaremos sus fuentes y las condiciones que afectan a su interpretación. Una síntesis puede omitir detalles o introducir errores, por lo que los documentos originales deben seguir disponibles para comprobarla.

Detendremos la búsqueda cuando exista evidencia suficiente para responder, sea necesaria una aclaración o se alcance el presupuesto de ejecución. **La decisión de terminar debe apoyarse en criterios verificables, no únicamente en la confianza expresada por el modelo.**

# Ingeniería de contexto (context engineering)

`Prompt engineering` e ingeniería de contexto buscan mejorar las respuestas del modelo, pero trabajan con un alcance diferente.

**Prompt engineering se centra en cómo redactamos y organizamos las instrucciones. Context engineering se ocupa de toda la información que recibe el modelo en cada paso**, incluidas esas instrucciones.

| | Prompt engineering | Context engineering |
|---|---|---|
| Pregunta principal | ¿Cómo le explicamos la tarea al modelo? | ¿Qué necesita recibir para resolverla en este momento? |
| Qué trabajamos | Instrucciones, ejemplos y formato de respuesta. | Instrucciones, historial, memoria, documentos y resultados de herramientas. |
| Ejemplo | «Identifica la causa del error y comprueba la corrección con una prueba». | Proporcionar el error, el código relacionado, las reglas del proyecto y los resultados de las pruebas ejecutadas. |

Son prácticas complementarias. Una instrucción clara puede resultar insuficiente si falta el archivo necesario para resolver el problema. Del mismo modo, proporcionar los archivos correctos no sustituye una explicación precisa de lo que debemos conseguir. En un agente, la selección del contexto se repite durante la tarea, porque cada acción produce información que puede cambiar el siguiente paso. [Anthropic, *Effective context engineering for AI agents*](https://www.anthropic.com/research/effective-context-engineering-for-ai-agents).

Por ejemplo, para corregir un error de autenticación, el agente podría necesitar:

| Componente | Información pertinente |
|---|---|
| Instrucciones | Reglas del proyecto y alcance del cambio autorizado. |
| Solicitud actual | El error que describe el usuario y el resultado esperado. |
| Estado de la tarea | Qué se revisó, qué se modificó y qué falta comprobar. |
| Evidencia | Código relacionado, registros del error y resultados de pruebas. |
| Herramientas disponibles | Qué operaciones puede solicitar y cómo utilizarlas. |
| Memoria recuperada | Decisiones anteriores sobre la autenticación que siguen vigentes. |

El contexto cambia durante el trabajo. Antes de investigar, necesitamos la descripción del error. Después de ejecutar una prueba, necesitamos su resultado. No todos los archivos o mensajes tienen que permanecer completos durante toda la tarea.

**Tener espacio disponible no significa que debamos llenarlo.** Una ventana amplia permite incluir más información, pero no garantiza que el modelo utilice correctamente cada detalle.

También debemos distinguir entre encontrar información y utilizarla para resolver una tarea. Localizar una frase dentro de un documento extenso no demuestra que el modelo pueda relacionar varias condiciones, detectar contradicciones o aplicar correctamente lo encontrado. Por eso, una prueba de recuperación como «aguja en un pajar» no basta para evaluar el trabajo completo de un agente.

Podemos combinar los mecanismos que ya vimos:

- Mantener completos los mensajes y resultados recientes que afectan al siguiente paso.
- Resumir el trabajo anterior conservando decisiones, restricciones y pendientes.
- Dejar documentos extensos fuera del contexto y recuperarlos cuando hagan falta.

La documentación actual de Deep Agents utiliza carga bajo demanda, almacenamiento externo y resumen para gestionar tareas prolongadas. Son opciones que podemos adaptar a nuestro sistema. [Ingeniería de contexto en Deep Agents](https://docs.langchain.com/oss/python/deepagents/context-engineering).

**Debemos conservar el origen y la autoridad de la información.** Una instrucción del usuario, una respuesta anterior del modelo y un documento recuperado cumplen funciones distintas. Resumirlos o reunirlos no debe convertir una suposición en un hecho ni una instrucción encontrada en un documento en una regla del sistema.

Para nuestro agente, construiremos el contexto antes de cada llamada a partir del estado de la tarea. Si pierde una restricción, repite una acción o responde sin evidencia, revisaremos qué información recibió en ese paso.

La meta es proporcionar contexto suficiente y pertinente. Comprobaremos su efecto en la calidad de los resultados, el coste y el tiempo de respuesta con tareas representativas.

## Ingeniería de contexto para sistemas multiagente

En un sistema multiagente debemos decidir qué información recibe cada agente y qué resultados comparte con los demás. Separar contextos permite que cada uno trabaje con la información pertinente para su tarea.

Por ejemplo, al investigar un error de autenticación, un agente podría revisar la documentación de la biblioteca mientras otro examina los registros del fallo. Cada uno necesita un encargo concreto y acceso a sus fuentes. No necesita recibir automáticamente toda la conversación ni todos los documentos del proyecto.

Un agente coordinador puede reunir sus resultados para decidir cómo continuar:

| Información que entrega | Información que espera recibir |
|---|---|
| Objetivo y pregunta que debe resolver cada agente. | Hallazgos que responden a esa pregunta. |
| Restricciones y fuentes disponibles. | Evidencia y referencias para comprobarlos. |
| Alcance del trabajo autorizado. | Limitaciones, dudas y comprobaciones pendientes. |

**El contexto de un agente no se comparte automáticamente con los demás.** La aplicación debe definir qué transmite: el encargo, ciertos mensajes, referencias a archivos o resultados concretos.

Esta separación puede evitar que búsquedas extensas llenen el contexto del coordinador. Sin embargo, también puede perderse información al resumir o transferir resultados. Por eso, un informe debe distinguir lo observado de lo supuesto y permitir consultar la evidencia original.

El modelo elegido para cada agente dependerá de la dificultad de su tarea. Especializar un agente significa delimitar su responsabilidad y su contexto; no implica utilizar necesariamente un modelo más pequeño.

**Añadir agentes introduce coordinación, llamadas al modelo y nuevas posibilidades de error.** Para nuestro sistema empezaremos con un solo agente. Incorporaremos otros cuando exista una tarea separable y las pruebas muestren un beneficio que justifique el coste y la complejidad. La [documentación de sistemas multiagente de LangChain](https://docs.langchain.com/oss/python/langchain/multi-agent) describe la gestión del contexto como una de las razones para utilizar esta arquitectura y señala que un solo agente puede ser suficiente.

Cuando lleguemos a la implementación multiagente, comprobaremos tanto el resultado final como los intercambios: qué recibió cada agente, qué devolvió y si el coordinador conservó las condiciones necesarias para resolver la tarea.

## Optimización del contexto (context optimization)

Optimizar el contexto consiste en decidir qué información recibe el modelo y cómo se presenta para que pueda continuar la tarea. Incluye los mensajes, la memoria recuperada, las instrucciones y las descripciones de herramientas.

Para nuestro agente trabajaremos sobre cuatro decisiones. Esta es una forma de organizar el tema, no una clasificación universal:

| Decisión | Para qué sirve |
|---|---|
| **Registrar y almacenar** | Conservar mensajes, acciones, resultados y fuentes para poder recuperarlos y revisar lo ocurrido. |
| **Seleccionar** | Elegir qué información necesita el modelo en el siguiente paso. |
| **Comprimir** | Reducir el espacio ocupado sin perder condiciones, decisiones o evidencia necesarias. |
| **Ordenar** | Organizar la información para distinguir instrucciones, antecedentes, resultados y solicitud actual. |

Estas decisiones están relacionadas. Podemos conservar un resultado completo fuera del contexto, enviar una síntesis y recuperar el original cuando necesitemos comprobar un detalle. La documentación actual de Deep Agents combina recuperación bajo demanda, almacenamiento externo y compresión; también contempla aislar contextos para tareas delegadas, como vimos en el apartado multiagente. Véase la [documentación de ingeniería de contexto de Deep Agents](https://docs.langchain.com/oss/python/deepagents/context-engineering).

Al ordenar la información, buscamos que su función y procedencia sean claras. La mejor posición de cada bloque debe comprobarse con el modelo y la tarea: **no existe un orden que garantice mejores respuestas en todos los casos.**

**Optimizar no significa utilizar siempre menos tokens.** Si falta una condición necesaria, añadirla puede mejorar el resultado. Si sobra información repetida o irrelevante, retirarla puede facilitar la tarea. Esta gestión se repite durante la ejecución del agente, como explica [Anthropic en su guía de ingeniería de contexto](https://www.anthropic.com/research/effective-context-engineering-for-ai-agents).

Comprobaremos los cambios con tareas representativas, evaluando si el agente conserva las restricciones, utiliza la evidencia y completa el trabajo, junto con el coste y el tiempo de respuesta.

### Seguimiento y almacenamiento del contexto (context tracking and storage)

Registrar lo que ocurre durante una tarea permite investigar errores y recuperar información para continuar el trabajo. Necesitamos distinguir qué recibió el modelo, qué solicitó hacer y qué ocurrió al ejecutar esa solicitud.

Por ejemplo, si un agente afirma que una prueba pasó, el registro debe permitir comprobar si la ejecutó, qué resultado obtuvo y si ese resultado corresponde a la versión del código que está revisando.

Para nuestro agente conservaremos información como esta:

| Qué registrar | Para qué sirve |
|---|---|
| Solicitud y correcciones del usuario | Reconstruir el objetivo y las restricciones vigentes. |
| Contexto enviado al modelo | Comprobar qué información tenía disponible en ese paso. |
| Solicitudes de herramientas | Identificar qué operación pidió y con qué parámetros. |
| Resultados y errores de ejecución | Distinguir una acción solicitada de una acción realizada. |
| Documentos y versiones consultadas | Comprobar las fuentes utilizadas. |
| Modelo y configuración | Comparar ejecuciones e investigar cambios de comportamiento. |

Los eventos deben relacionarse mediante identificadores de conversación, ejecución y paso. Así podemos seguir una tarea aunque incluya varias llamadas al modelo o a herramientas.

**El registro muestra lo que el sistema pudo observar; no revela los pensamientos internos del modelo.** Una explicación generada por el modelo puede guardarse como parte de su salida, pero no demuestra cómo produjo la respuesta.

También debemos distinguir el registro de la memoria que utilizaremos después. Podemos conservar un resultado para depurar sin enviarlo en las siguientes llamadas. Incorporarlo a la memoria requiere seleccionarlo y decidir cuándo recuperarlo.

No necesitamos copiar todos los documentos ni guardar todos los contenidos en cada evento. Podemos conservar referencias y versiones; cuando sea necesario revisar exactamente qué recibió el modelo, necesitaremos una copia controlada o una referencia a contenido que no cambie.

**Registrar todo indiscriminadamente puede almacenar datos sensibles y aumentar los costes.** Definiremos qué contenido conservar, quién puede consultarlo y durante cuánto tiempo. Las convenciones de OpenTelemetry para IA generativa distinguen los metadatos operativos de la captura opcional de mensajes y otros contenidos. [OpenTelemetry: convenciones para operaciones de IA generativa](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-spans.md).

Este registro nos permitirá investigar preguntas concretas: ¿faltaba información en el contexto?, ¿falló una herramienta?, ¿se utilizó una fuente antigua?, ¿el modelo interpretó mal un resultado correcto?

### Selección de contexto (context selection)

Seleccionar contexto consiste en elegir qué información necesita el modelo para continuar la tarea. No basta con encontrar documentos relacionados con el tema: necesitamos los que permitan responder la pregunta concreta.

Por ejemplo, ante una consulta sobre reembolsos, la búsqueda podría devolver políticas de distintos productos y versiones. Antes de enviarlas al modelo, debemos identificar cuáles corresponden al producto, la fecha y los permisos del usuario.

Un **modelo de reordenamiento (reranker)** puede evaluar los candidatos recuperados respecto a la consulta y priorizar los más pertinentes. Es una técnica vigente: servicios como Cohere reciben una consulta y un conjunto de documentos para devolverlos ordenados por relevancia. Su API también permite identificar pasajes relevantes dentro de los documentos con modelos compatibles. Esta extracción es una opción adicional, no un requisito de todo reranker. Véase la [documentación de Rerank](https://docs.cohere.com/reference/rerank).

**Un reranker no puede seleccionar una respuesta que la búsqueda inicial no recuperó.** Por eso evaluaremos tanto la recuperación de candidatos como su selección posterior.

Para nuestro agente aplicaremos primero las restricciones obligatorias, como permisos y ámbito del proyecto. Después seleccionaremos evidencia según la pregunta y el presupuesto de contexto. Conservaremos las fuentes y suficiente información circundante para interpretar cada fragmento.

Si los resultados no contienen evidencia suficiente, debemos poder realizar otra búsqueda o reconocer que falta información. Elegir los primeros resultados no demuestra que sean adecuados.

### Compresión de contexto (context compression)

Comprimir el contexto permite reducir su tamaño conservando la información necesaria para continuar. Podemos resumir intercambios antiguos, extraer fragmentos pertinentes o retirar resultados que ya no necesitamos mantener completos.

La **compactación (compaction)** aplica este principio al historial: sustituye parte de los mensajes anteriores por una representación más breve. La documentación actual de Anthropic incluye compactación de conversaciones para tareas prolongadas. Las modalidades bajo demanda y por umbral de tokens están documentadas como beta; su disponibilidad depende del modelo y de la plataforma utilizados. Véase la [documentación de compaction](https://platform.claude.com/docs/en/build-with-claude/compaction).

Por ejemplo, después de investigar un error, podemos conservar la causa identificada, los archivos modificados, las pruebas ejecutadas y los pendientes. Los registros extensos pueden permanecer almacenados fuera del contexto, disponibles para consultas posteriores.

**La meta no es comprimir al máximo, sino conservar lo necesario para actuar correctamente.** Un resumen que omite una restricción o altera un resultado puede perjudicar el trabajo aunque utilice menos tokens.

También podemos reducir redundancia antes de incorporar documentos. La **relevancia marginal máxima (maximal marginal relevance, MMR)** selecciona candidatos equilibrando su similitud con la consulta y su diferencia respecto a los ya seleccionados. No reescribe los documentos. El método fue presentado por [Carbonell y Goldstein (1998)](https://doi.org/10.1145/290941.291025).

En su formulación habitual, λ pondera la relevancia y `1 − λ` penaliza la redundancia. Por tanto, **un λ mayor favorece la relevancia; uno menor favorece la diversidad**. Así lo documenta la [implementación de MMR de LangChain](https://github.com/langchain-ai/langchain/blob/master/libs/partners/chroma/langchain_chroma/vectorstores.py).

Para nuestro agente comprobaremos que cualquier reducción conserve las restricciones vigentes, las fuentes y los resultados necesarios. Retirar contenido del contexto no exige eliminar el registro original.

### Ordenación del contexto (context ordering)

Ordenar el contexto ayuda a distinguir instrucciones, documentos, resultados de herramientas y solicitud actual. También permite mantener juntos los elementos que necesitan interpretarse en relación, como una llamada a una herramienta y su resultado.

El estudio *Lost in the Middle* mostró que, en los modelos y tareas evaluados, utilizar información situada en el centro de contextos largos podía resultar más difícil que utilizarla al principio o al final. Es un resultado de 2023 relevante como antecedente, pero no establece el comportamiento de todos los modelos actuales. Véase [Liu et al., *Lost in the Middle*](https://arxiv.org/abs/2307.03172).

La documentación actual de Claude recomienda, para entradas extensas, colocar los documentos antes de la consulta. Es una orientación específica que debemos contrastar con nuestro modelo y nuestras tareas. Véanse las [buenas prácticas de prompting de Claude](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices).

Para nuestro agente utilizaremos secciones identificables y conservaremos el orden de los eventos cuando importe para comprender lo ocurrido.

**Cambiar una información de posición no cambia su autoridad.** Un documento recuperado sigue siendo una fuente externa, aunque aparezca al principio de la solicitud.

Compararemos distintas organizaciones del contexto con las mismas tareas. No asumiremos que una única plantilla funciona mejor en todos los casos.

## Contexto como especificación (context as specification)

Los objetivos, las restricciones y los criterios de aceptación incluidos en el contexto permiten evaluar si el agente hizo el trabajo solicitado. En ese sentido, una parte del contexto funciona como especificación de la tarea.

Por ejemplo, «corregir el inicio de sesión sin cambiar el proveedor de autenticación» establece tanto un objetivo como un límite. Una solución que cambia de proveedor puede superar sus pruebas y aun así incumplir el encargo.

Podemos conservar estas condiciones en documentos como `REQUIREMENTS.md`, un plan de trabajo o instrucciones del proyecto. `AGENTS.md` ofrece un lugar reconocible para comunicar instrucciones a agentes de programación. Véase la [documentación de AGENTS.md](https://agents.md/).

**El contexto completo no equivale a una especificación.** También contiene resultados provisionales, respuestas del modelo y documentos externos que pueden ser incorrectos o estar desactualizados. Debemos distinguirlos de los requisitos confirmados.

Para nuestro agente mantendremos identificables el objetivo, el alcance autorizado y los criterios de aceptación. Cuando el usuario los corrija, registraremos el cambio y evitaremos seguir utilizando una versión anterior.

Conservar el contexto ayuda a investigar decisiones, pero no garantiza reproducir exactamente una ejecución. También pueden cambiar el modelo, las herramientas y las fuentes consultadas.

# Cómo evaluar la memoria del agente (memory evaluation)

La memoria resulta útil cuando el agente recupera información pertinente, reconoce sus cambios y la aplica correctamente. Comprobar que un dato se guardó es solo el primer paso: necesitamos observar qué ocurre cuando vuelve a utilizarse.

Podemos evaluar el sistema mediante conversaciones distribuidas en varias sesiones. En cada caso definimos qué información debería conservar, cuál debería recuperar y qué respuesta esperamos.

| Qué evaluamos | Ejemplo | Comportamiento esperado |
|---|---|---|
| **Recuperación** | El usuario indicó su presupuesto en una sesión anterior. | Recuperar el importe cuando vuelva a comparar opciones. |
| **Actualización** | El presupuesto cambió de 800 a 1.000 dólares. | Utilizar el nuevo importe para decisiones actuales. |
| **Razonamiento temporal** | Preguntamos cuál era el presupuesto antes del cambio. | Distinguir el valor anterior del vigente. |
| **Ámbito** | El usuario pidió ejemplos en Python para un proyecto concreto. | No convertirlo automáticamente en una preferencia para todos sus proyectos. |
| **Continuidad** | La aplicación se reinicia con una tarea pendiente. | Recuperar el estado confirmado y comprobar qué sigue vigente antes de continuar. |
| **Separación entre usuarios** | Dos personas utilizan el mismo agente. | Recuperar únicamente la información que cada una tiene permitido consultar. |
| **Información insuficiente** | Preguntamos por una preferencia que nunca se indicó. | Reconocer que no dispone del dato, sin inventarlo. |
| **Olvido solicitado** | El usuario pide eliminar una dirección. | Dejar de recuperarla desde la memoria activa y sus representaciones derivadas. |

LongMemEval evalúa capacidades como extracción de información, razonamiento entre sesiones, razonamiento temporal, actualización de conocimientos y reconocimiento de información ausente. Estas categorías ofrecen una base para diseñar nuestras propias pruebas. [LongMemEval, ICLR 2025](https://arxiv.org/abs/2410.10813).

## Distinguir dónde ocurrió el error

Si el agente utiliza un presupuesto antiguo, necesitamos revisar el proceso completo:

- **Almacenamiento:** ¿se guardó la corrección?
- **Recuperación:** ¿se encontró el recuerdo vigente?
- **Construcción del contexto:** ¿se incluyó ese recuerdo y se identificó cuál reemplazaba al anterior?
- **Respuesta o acción:** ¿el modelo utilizó correctamente la información recibida?

**Una respuesta incorrecta no demuestra por sí sola que falló la recuperación.** El agente puede recibir el dato correcto e interpretarlo mal. Separar estas etapas permite corregir la causa del problema.

## Comprobar qué se convierte en memoria

También evaluaremos el origen de los recuerdos. Una preferencia declarada por el usuario, una observación de una herramienta y una interpretación del modelo deben conservarse como cosas distintas.

Por ejemplo, visitar la documentación de Python no demuestra que el usuario prefiera ese lenguaje. Del mismo modo, una instrucción encontrada en una página externa no debe convertirse en una regla permanente del agente.

**Guardar y recuperar una afirmación repetidamente no la convierte en verdadera.** Conservaremos su fuente, ámbito y fecha, y distinguiremos lo confirmado de lo inferido.

## Verificar el olvido

Eliminar un recuerdo puede requerir más que borrar una entrada. El dato podría aparecer también en un resumen, un índice de búsqueda o una copia derivada utilizada por el sistema.

Comprobaremos que deje de estar disponible para las consultas posteriores dentro del alcance de eliminación definido por la aplicación. También distinguiremos retirar un dato de la memoria activa de eliminarlo de registros o copias de respaldo: son operaciones diferentes y debemos comunicar su alcance con precisión.

La investigación reciente de MemCut estudia precisamente la dificultad de cumplir solicitudes de olvido en sistemas de memoria. Es una propuesta de evaluación, no una garantía de que un método concreto resuelva el problema. [MemCut, octubre de 2026](https://arxiv.org/abs/2610.21405).

## Comparar el beneficio de la memoria

Para nuestro agente prepararemos un conjunto pequeño de casos representativos y compararemos su funcionamiento con y sin memoria. Revisaremos si mejora la continuidad y la corrección, y cuánto añade en tiempo de respuesta, tokens y almacenamiento.

**La memoria debe demostrar su utilidad en las tareas del agente.** Repetiremos las pruebas cuando cambiemos el modelo, la recuperación o la forma de resumir y guardar información.

# Resumen

La memoria permite conservar información y recuperarla para continuar una tarea o comenzar otra. El historial mantiene los intercambios de una conversación; la memoria persistente conserva información entre sesiones. RAG permite buscar fuentes externas e incorporarlas al contexto.

**Guardar información no significa que el modelo la tenga disponible.** El sistema debe seleccionarla y proporcionársela cuando resulte pertinente.

La ingeniería de contexto reúne estas decisiones. Determina qué recibe el modelo en cada paso, cómo se organiza y qué puede permanecer almacenado fuera de la llamada actual.

Para construir nuestro agente empezaremos con historial de mensajes, registro de acciones y resultados, y mecanismos sencillos de recuperación. Incorporaremos técnicas más complejas cuando podamos comprobar su beneficio.

En el siguiente capítulo veremos cómo el modelo solicita herramientas, cómo la aplicación las ejecuta y cómo sus resultados vuelven al contexto para decidir el siguiente paso.

# Referencias

Fuentes citadas en este capítulo. Las publicaciones describen métodos o resultados en sus condiciones de evaluación; la documentación describe implementaciones que pueden cambiar. Documentación consultada el 3 de noviembre de 2026.

## Publicaciones

- [*LongMemEval: Benchmarking Chat Assistants on Long-Term Interactive Memory*](https://arxiv.org/abs/2410.10813). ICLR 2025; prepublicación de 2024.
- [*MemCut: Every Secret Leaves a Trail in the Long-Term Memory of LLM Agents*](https://arxiv.org/abs/2610.21405). Prepublicación, octubre de 2026.

- Sumers, T. R., Yao, S., Narasimhan, K. y Griffiths, T. L. (2024). [*Cognitive Architectures for Language Agents*](https://openreview.net/forum?id=1i6ZCvflQJ). Transactions on Machine Learning Research. Prepublicación de 2023.
- Lewis, P. et al. (2020). [*Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks*](https://arxiv.org/abs/2005.11401). NeurIPS 2020.
- Xu, W. et al. (2025). [*A-Mem: Agentic Memory for LLM Agents*](https://papers.nips.cc/paper_files/paper/2025/hash/19909c36f51abc4856b4560aff3d36d6-Abstract-Conference.html). NeurIPS 2025.
- Li, X. et al. (2025). [*Search-o1: Agentic Search-Enhanced Large Reasoning Models*](https://aclanthology.org/2025.emnlp-main.276/). EMNLP 2025.
- Carbonell, J. y Goldstein, J. (1998). [*The use of MMR, diversity-based reranking for reordering documents and producing summaries*](https://doi.org/10.1145/290941.291025). SIGIR 1998, pp. 335–336.
- Liu, N. F. et al. (2023). [*Lost in the Middle: How Language Models Use Long Contexts*](https://arxiv.org/abs/2307.03172). Prepublicación, arXiv:2307.03172.

## Documentación y guías técnicas

- Anthropic (2025). [*Effective context engineering for AI agents*](https://www.anthropic.com/research/effective-context-engineering-for-ai-agents). 29 de septiembre de 2025.
- Anthropic. [*Memory tool*](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool).
- Anthropic. [*Compaction overview*](https://platform.claude.com/docs/en/build-with-claude/compaction).
- Anthropic. [*Prompting best practices*](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices).
- LangChain. [*Memory overview*](https://docs.langchain.com/oss/python/concepts/memory).
- LangChain. [*Memory in Deep Agents*](https://docs.langchain.com/oss/python/deepagents/memory).
- LangChain. [*Retrieval*](https://docs.langchain.com/oss/python/deepagents/retrieval).
- LangChain. [*Context engineering in Deep Agents*](https://docs.langchain.com/oss/python/deepagents/context-engineering).
- LangChain. [*Multi-agent*](https://docs.langchain.com/oss/python/langchain/multi-agent).
- LangChain. [Implementación de MMR en la integración con Chroma](https://github.com/langchain-ai/langchain/blob/master/libs/partners/chroma/langchain_chroma/vectorstores.py).
- Cohere. [*Rerank API (v2)*](https://docs.cohere.com/reference/rerank).
- OpenTelemetry. [*Semantic conventions for generative AI operations*](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-spans.md).
- AGENTS.md. [Formato abierto para instrucciones de agentes de programación](https://agents.md/).
