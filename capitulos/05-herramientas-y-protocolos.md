
# Capítulo 5. Uso de herramientas, aprendizaje y protocolos

Las **herramientas (tools)** permiten que nuestro agente consulte información, realice cálculos y ejecute operaciones sobre otros sistemas. Puede utilizarlas para recuperar un recuerdo, leer un archivo o crear un evento en un calendario. Así conectamos lo aprendido sobre memoria con una nueva capacidad: obtener información y producir cambios mediante operaciones ejecutables.

Una herramienta puede ser una función de nuestra aplicación o una integración con un servicio externo mediante una **interfaz de programación de aplicaciones (API)**. También puede consultar otro modelo especializado. Para utilizarla, el modelo necesita conocer qué hace, qué argumentos acepta y en qué situaciones resulta apropiada.

**Solicitar una herramienta no equivale a ejecutarla.** El modelo genera una **llamada a herramienta (tool call)** que identifica la operación y sus argumentos. Un componente de ejecución procesa esa solicitud y devuelve un resultado. Según la integración, ese componente pertenece a nuestra aplicación o a la infraestructura del proveedor. La documentación de Claude distingue ambas posibilidades como herramientas del cliente y del servidor: “Client tools” y “Server tools”. [Documentación oficial sobre uso de herramientas](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview).

Consideremos un **ejemplo ilustrativo**, sin ejecutar ninguna operación real. Una persona pregunta: «¿Cuánto es 24 por 17?». Si el agente dispone de una herramienta de multiplicación, la interacción puede seguir este recorrido:

1. **Solicitud del modelo:** propone utilizar `multiply` con los argumentos `24` y `17`.
2. **Ejecución:** la aplicación valida los argumentos y ejecuta la función.
3. **Resultado observado:** la herramienta devuelve `408`.
4. **Respuesta:** el modelo utiliza ese resultado para contestar.

El resultado es comprobable: 24 × 17 = 408. Sin embargo, escribir «voy a multiplicar» no demuestra que se haya ejecutado una función. Para saber qué ocurrió necesitamos registrar la solicitud, su ejecución y el resultado devuelto. Si la operación falla, ese error también forma parte de lo observado.

Las herramientas pueden **consultar información o modificar el entorno**. Leer un calendario y crear una reunión son operaciones diferentes, aunque pertenezcan a la misma integración. Esta distinción nos ayuda a decidir qué permisos necesita cada herramienta y qué acciones requieren confirmación. El acceso a una herramienta tampoco garantiza que la información recuperada sea correcta o que una operación produzca el efecto esperado.

El uso de herramientas puede organizarse de dos maneras. En un **flujo de trabajo (workflow)**, el código establece una secuencia: buscar documentos, recuperar sus contenidos y generar un resumen. En un sistema con **selección dinámica de herramientas**, el modelo decide el siguiente paso según la solicitud y los resultados disponibles. Puede buscar otra fuente, pedir un dato que falta o responder sin realizar más llamadas. Anthropic distingue estos patrones en *Building effective agents*, publicado en diciembre de 2024. [Fuente primaria](https://www.anthropic.com/research/building-effective-agents).

Ambos patrones pueden combinarse. Podemos permitir que el modelo elija una consulta y mantener en el código las reglas de autorización, los límites de ejecución y el manejo de errores. La autonomía indica qué decisiones delegamos al modelo; por sí sola no demuestra que el sistema resuelva mejor la tarea.

Para construir nuestro agente, empezaremos con un conjunto pequeño de herramientas y un circuito observable: describir las operaciones disponibles, recibir una solicitud, validarla, ejecutarla y devolver su resultado al modelo. Primero utilizaremos operaciones de lectura y cálculo. Después incorporaremos acciones que modifican otros sistemas y definiremos sus permisos.

A medida que crezca el conjunto de herramientas, también tendremos que administrar el contexto. No siempre será conveniente cargar todas sus definiciones ni todos los resultados intermedios. La **búsqueda de herramientas (tool search)** y las **llamadas programáticas a herramientas (programmatic tool calling)** ofrecen formas de descubrir operaciones cuando se necesitan y componerlas mediante código. Son capacidades disponibles en algunas plataformas; evaluaremos su utilidad cuando nuestro agente necesite esa escala. [Anuncio de disponibilidad general, febrero de 2026](https://claude.com/resources/articles/improved-web-search-with-dynamic-filtering).

En este capítulo veremos cómo crear y describir herramientas, cómo seleccionarlas y cómo procesar sus resultados. También distinguiremos el aprendizaje de uso de herramientas durante el entrenamiento de su utilización mediante instrucciones, y estudiaremos los protocolos que permiten conectar el agente con servicios externos.

# Uso de herramientas

Para que nuestro agente utilice herramientas necesitamos conectar las decisiones del modelo con operaciones ejecutables y devolverle información sobre lo ocurrido. Este circuito permite que continúe la tarea a partir de resultados observados.

Conviene separar la **preparación de las herramientas** de su **ciclo de ejecución**. Crear una función y describírsela al modelo no son pasos que debamos repetir cada vez que la utiliza.

La preparación comprende dos tareas:

- **Creación de herramientas (tool creation).** Implementamos una operación o conectamos una ya existente. Definimos qué hace, qué devuelve y cómo comunica sus errores.
- **Definición de herramientas (tool definition).** Describimos las operaciones disponibles para el modelo: sus nombres, propósitos y argumentos. Esta descripción constituye su interfaz de uso; no sustituye al código que las ejecuta.

Con las herramientas disponibles, comienza el ciclo:

1. **Selección de herramientas (tool selection).** El modelo decide si necesita una herramienta y cuál utilizar, dentro de las opciones y restricciones que recibe. También puede responder sin herramientas o pedir información que falta.
2. **Llamada a herramienta (tool call).** El modelo genera una solicitud con el nombre de la operación y sus argumentos. La aplicación comprueba que la herramienta exista, valida los datos y verifica los permisos. Si corresponde, obtiene la confirmación del usuario antes de ejecutarla.
3. **Procesamiento del resultado (tool result processing).** El componente de ejecución devuelve un resultado o informa de un error. La aplicación lo asocia con la solicitud correspondiente y lo incorpora al contexto que recibirá el modelo.

**La solicitud, la ejecución y el resultado son eventos distintos.** La documentación de Gemini lo indica expresamente: “The model *doesn't* execute the function itself”. En las funciones que implementamos, nuestra aplicación se encarga de ejecutarlas y devolver sus resultados. [Documentación oficial de llamadas a funciones](https://ai.google.dev/gemini-api/docs/function-calling).

Veamos un **ejemplo ilustrativo**, sin consultar archivos reales. Una persona pide: «Busca la política de devoluciones y dime cuántos días tengo para devolver un producto». El modelo solicita una búsqueda. La aplicación valida la consulta, ejecuta la herramienta y devuelve un fragmento que indica un plazo de treinta días.

Ese fragmento puede ser suficiente para responder, pero también puede indicar que el plazo depende del tipo de producto. En ese caso, el siguiente paso podría ser consultar otra sección o pedir una aclaración. **El resultado de una herramienta alimenta la siguiente decisión; no siempre cierra la tarea.**

Esta separación entre el modelo y el programa externo ya aparece en *What Are Tools Anyway?*, de marzo de 2024. La conservamos como fundamento del circuito que construiremos. [Publicación original](https://arxiv.org/html/2403.15452v1).

El circuito también debe representar sus límites. Una herramienta puede fallar, devolver información incompleta o indicar que una operación sigue pendiente. Si un servicio acepta un trabajo para procesarlo después, esa aceptación no demuestra que haya terminado.

Cuando una operación modifica el entorno, distinguiremos el mensaje devuelto del efecto que necesitamos comprobar. Una respuesta del agente que diga «archivo guardado» no demuestra por sí sola que el archivo exista con el contenido esperado. La guía de evaluación de agentes de Anthropic, publicada en enero de 2026, distingue precisamente el registro de la interacción del resultado final en el entorno. [Fuente primaria](https://www.anthropic.com/research/demystifying-evals-for-ai-agents).

En nuestro agente conservaremos un registro que permita identificar qué se solicitó, qué llegó a ejecutarse y qué resultado se recibió. Los siguientes apartados desarrollan cada pieza, comenzando por la creación de herramientas.

## Creación de herramientas

Crear una herramienta permite que nuestro agente realice una operación mediante código, en lugar de depender de una respuesta generada para resolverla. Podemos implementar una función local o conectar una API existente. En ambos casos necesitamos establecer qué entradas acepta, qué resultado devuelve y qué ocurre cuando no puede completar la operación.

**Empezaremos por una función que podamos probar sin el modelo.** Después la conectaremos al agente. Así podremos distinguir un fallo en la implementación de una solicitud incorrecta del modelo.

Como ejemplo didáctico, utilizaremos `multiply`, una herramienta que multiplica dos números enteros. Para esta primera versión elegimos un contrato acotado:

- Recibe dos argumentos obligatorios, `a` y `b`, ambos enteros entre −1 000 000 y 1 000 000.
- Devuelve su producto como número entero.
- Rechaza argumentos ausentes, valores de otro tipo o números fuera del intervalo permitido.
- No consulta servicios externos ni modifica datos.

El intervalo es una decisión de nuestro ejemplo, no una restricción general de las herramientas. Nos permite empezar con cálculos pequeños y resultados exactos.

Los siguientes casos describen el comportamiento esperado; **no representan llamadas realizadas por un modelo**:

| Entrada | Resultado esperado |
|---|---|
| `a = 24`, `b = 17` | `408` |
| `a = -3`, `b = 7` | `-21` |
| `a = 0`, `b = 17` | `0` |
| `a = "24"`, `b = 17` | Error de validación: `a` es una cadena |
| Falta `b` | Error de validación: falta un argumento obligatorio |

No necesitamos aceptar cadenas porque la solicitud proceda de un modelo. Los formatos estructurados pueden representar números, cadenas, listas y objetos. JSON Schema distingue expresamente los tipos numéricos “integer” y “number”. Elegiremos el tipo que corresponda a cada operación. [Referencia oficial de tipos numéricos](https://json-schema.org/understanding-json-schema/reference/numeric).

Esta primera herramienta no acepta decimales. Si después necesitamos cálculos con importes monetarios u otras reglas de precisión, definiremos su representación y redondeo antes de ampliar el contrato.

Para que la aplicación encuentre la función utilizaremos un **registro de herramientas (tool registry)**: una asociación entre nombres conocidos e implementaciones. Al recibir una solicitud para `multiply`, el ejecutor consulta ese registro y obtiene la función correspondiente. Un nombre desconocido debe producir un error; no debe interpretarse como código para ejecutar.

Junto a la implementación mantendremos su descripción y las reglas que necesita el ejecutor. **La descripción informa al modelo; los permisos los aplica el software.** Indicar que una herramienta requiere confirmación no basta si el componente de ejecución no hace cumplir esa condición.

La documentación debe explicar el propósito, los argumentos, el resultado y los límites de la herramienta. Algunos SDK pueden obtener parte de esa información de tipos o cadenas de documentación, pero debemos comprobar qué descripción termina recibiendo el modelo. No podemos asumir que conoce el código interno.

Tampoco necesitamos convertir cada función de nuestra aplicación en una herramienta. Una operación con un propósito claro puede agrupar varias llamadas internas si eso facilita la tarea. La guía de Anthropic de septiembre de 2025 recomienda seleccionar herramientas según tareas reales y evaluar su funcionamiento con el agente. [Fuente primaria](https://www.anthropic.com/research/writing-tools-for-agents).

Antes de conectar `multiply`, comprobaremos sus resultados y sus errores de validación. Más adelante evaluaremos si el modelo la selecciona y solicita correctamente. **Que la función funcione no demuestra que el agente sepa utilizarla.**

En el siguiente apartado describiremos esta herramienta para el modelo mediante su nombre, sus instrucciones de uso y el esquema de sus argumentos.

## Definición de la herramienta

Definir una herramienta permite que el modelo identifique para qué sirve y cómo solicitarla. Ya tenemos una operación ejecutable; ahora debemos comunicar su interfaz sin exigir que el modelo conozca su implementación.

La **definición de una herramienta (tool definition)** describe su nombre, propósito y argumentos. También debe aclarar cuándo utilizarla, qué devuelve y cuáles son sus límites. En las API con llamadas estructuradas, parte de esa información se expresa mediante un **esquema de entrada (input schema)**. La documentación de Claude lo describe como un objeto JSON Schema que define los parámetros esperados. [Documentación oficial](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools).

Para entender cómo utiliza esa información el modelo, conviene distinguir dos procesos que se complementan:

- **Aprendizaje durante el entrenamiento (training).** El entrenamiento puede modificar los parámetros del modelo para mejorar la selección de herramientas, la generación de argumentos o el uso de resultados. Puede incluir herramientas concretas y situaciones diseñadas para favorecer la generalización.
- **Uso de definiciones durante la inferencia (inference).** La aplicación proporciona información sobre las herramientas disponibles para la tarea. El modelo utiliza esa información dentro de su contexto. Incorporar una descripción o un ejemplo al contexto no actualiza por sí mismo sus parámetros.

Un modelo entrenado para generar llamadas a herramientas no conoce automáticamente nuestras funciones privadas. Necesita recibir su interfaz o disponer de un mecanismo para descubrirla. A la inversa, proporcionar una descripción no garantiza que cualquier modelo pueda utilizarla correctamente: debemos comprobar la compatibilidad del modelo, el formato y el componente que interpreta sus solicitudes.

El informe de DeepSeek-V3.2, publicado en diciembre de 2025, describe entrenamiento con tareas de agente sintetizadas para mejorar estas capacidades. Es un ejemplo de cómo se desarrolla la capacidad del modelo; no significa que debamos repetir ese entrenamiento al incorporar una herramienta a nuestra aplicación. [Informe técnico original](https://arxiv.org/html/2512.02556v1).

Para nuestro agente, empezaremos proporcionando la definición de `multiply` a un modelo compatible con el mecanismo elegido. Una **descripción ilustrativa**, coherente con el contrato anterior, sería:

> Multiplica dos enteros. Recibe los argumentos obligatorios `a` y `b`, cada uno entre −1 000 000 y 1 000 000. Devuelve su producto como entero. No acepta decimales ni cadenas y no modifica datos externos.

Esta descripción comunica el contrato, pero no ejecuta la multiplicación ni hace cumplir sus restricciones. El ejecutor sigue siendo responsable de validar la solicitud y aplicar las reglas correspondientes.

Podemos acompañar la definición con ejemplos de argumentos válidos. La documentación actual de Claude incluye el campo opcional `input_examples` para mostrar patrones de entrada junto al esquema. Estos ejemplos complementan la descripción; no sustituyen la validación ni garantizan que el modelo elija la operación adecuada. [Documentación oficial sobre ejemplos](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools#providing-tool-use-examples).

En los siguientes subapartados veremos dos formas de comunicar esta interfaz: mediante instrucciones en lenguaje natural y mediante definiciones estructuradas para llamadas a funciones.

### Utilizando el lenguaje natural

Las instrucciones en **lenguaje natural (natural language)** permiten describir herramientas sin depender de una interfaz específica de llamadas a funciones. Este enfoque sirve para comprender qué información necesita el modelo y cómo la aplicación reconoce una solicitud.

Para solicitar una operación, debemos acordar un formato reconocible. En un **ejemplo ilustrativo**, podríamos indicar que una solicitud contiene el nombre `multiply` y los argumentos enteros `a = 24` y `b = 17`, mientras que una respuesta final contiene texto para el usuario. Son dos tipos de salida que nuestro programa debe distinguir.

Elegir un formato tiene consecuencias. El **analizador sintáctico (parser)** identifica la solicitud y extrae sus campos. El **validador (validator)** comprueba después que esos datos cumplan el contrato. Una salida incompleta o ambigua debe producir un error controlado; no debe convertirse en código para ejecutar.

Proporcionar instrucciones y demostraciones puede orientar al modelo, como se estudia en el **aprendizaje en contexto (in-context learning, ICL)**. No garantiza que respete el formato en todos los intentos. [Dong y colaboradores, primera versión de diciembre de 2022, revisión de octubre de 2024](https://arxiv.org/abs/2301.00234).

Para nuestro agente, este enfoque explica el mecanismo. Si el modelo y el servicio ofrecen llamadas estructuradas compatibles, evaluaremos esa interfaz antes de construir un protocolo de texto propio. En ambos casos debemos enviar la definición cuando corresponda; su presencia en una conversación anterior no garantiza que siga en el contexto disponible.

### Utilizando llamadas a funciones estructuradas

Las **llamadas a funciones (function calling)** ofrecen una interfaz para declarar herramientas y recibir solicitudes separadas de la respuesta dirigida al usuario. La aplicación sigue proporcionando las implementaciones y gestionando su ejecución.

Una definición suele combinar un nombre, una descripción y un esquema de argumentos. La descripción explica el propósito y las condiciones de uso; el esquema expresa restricciones estructurales. Para nuestra herramienta, la definición debe conservar este contrato:

| Elemento | Definición de nuestro ejemplo |
|---|---|
| Nombre | `multiply` |
| Propósito | Multiplicar dos enteros |
| Argumentos | `a` y `b`, ambos obligatorios |
| Tipo y límites | Enteros entre −1 000 000 y 1 000 000 |
| Resultado | Producto entero |
| Exclusiones | No acepta cadenas ni decimales |

El nombre del campo que contiene el esquema, el subconjunto de JSON Schema admitido y la representación de la llamada dependen del servicio. La documentación de Claude utiliza `input_schema`; otros adaptadores pueden utilizar nombres distintos. No debemos confundir esa interfaz con la plantilla interna de un modelo. [Documentación oficial de definiciones](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools).

Algunos servicios permiten **uso estricto de herramientas (strict tool use)** para restringir los argumentos al esquema admitido. Anthropic anunció sus salidas estructuradas en noviembre de 2025 y documentó su disponibilidad general en febrero de 2026. Es una capacidad de esa plataforma, no una garantía disponible en cualquier modelo o configuración. [Anuncio y actualizaciones](https://claude.com/resources/articles/structured-outputs-on-the-claude-developer-platform).

**Cumplir el esquema no demuestra que la solicitud sea pertinente, esté autorizada o vaya a completarse.** El ejecutor conserva esas responsabilidades. Tampoco desaparece la necesidad de explicar la herramienta: un entero válido puede ser el argumento equivocado para la tarea.

Los ejemplos de entradas pueden aclarar convenciones difíciles de expresar solo con tipos. La incorporación de **ejemplos de uso de herramientas (tool use examples)** y de búsqueda de herramientas en Claude, anunciada en noviembre de 2025, muestra una implementación de estas prácticas. Sus resultados publicados pertenecen a las pruebas del proveedor; debemos evaluar nuestro propio agente. [Fuente primaria](https://www.anthropic.com/research/advanced-tool-use).

## Selección de herramientas

La **selección de herramientas (tool selection)** permite decidir si una operación disponible ayuda a resolver la solicitud. Seleccionar también puede significar no utilizar ninguna herramienta o pedir un dato que falta.

Si una persona pide «multiplica 24 por 17», `multiply` corresponde a la operación. Si pide «multiplica 24 por la cantidad de asistentes», falta un dato: el modelo debe obtenerlo de una fuente autorizada o solicitarlo. Inventar esa cantidad produciría una llamada estructuralmente válida con un argumento sin fundamento. Estos son **casos ilustrativos**, no resultados medidos de un modelo.

Para favorecer una selección correcta, utilizaremos nombres distinguibles, descripciones que aclaren cuándo corresponde cada herramienta y un conjunto inicial pequeño. No fijamos un número universal: la dificultad depende del solapamiento entre operaciones, la tarea, el modelo y la información disponible. La guía de Anthropic sobre diseño de herramientas recomienda empezar con operaciones orientadas a tareas y medir su utilización. [Guía de septiembre de 2025](https://www.anthropic.com/research/writing-tools-for-agents).

Si la tarea requiere varias operaciones, las dependencias importan. Una consulta cuyo argumento proviene de otra debe esperar su resultado. Solo consideraremos ejecución paralela cuando las solicitudes sean independientes y sus efectos lo permitan. La planificación de secuencias se desarrollará en el capítulo siguiente.

### Descubrimiento de herramientas

El **descubrimiento de herramientas (tool discovery)** permite localizar operaciones relevantes sin presentar todo el catálogo en cada decisión. Podemos buscar sus nombres y descripciones mediante reglas, búsqueda textual o recuperación semántica. Una base vectorial es una opción, no un requisito.

ToolLLM, publicado inicialmente en julio de 2023, investiga recuperación y utilización de catálogos amplios de API. Nos interesa como antecedente de la selección de herramientas candidatas; sus evaluaciones no establecen el tamaño óptimo de nuestro catálogo. [Publicación original](https://arxiv.org/abs/2307.16789).

En nuestro agente empezaremos con las definiciones necesarias para las primeras tareas. Si el catálogo crece, evaluaremos cargar otras bajo demanda. **Recuperar una definición no equivale a autorizar su ejecución.** Una búsqueda puede omitir una herramienta útil o devolver varias que se parecen; necesitamos comprobar tanto la recuperación como la selección posterior.

## Llamada de herramientas

Una **llamada a herramienta (tool call)** transforma la decisión del modelo en una solicitud que el ejecutor puede procesar. Debemos conservar su nombre, sus argumentos y el identificador que proporcione la interfaz para asociarla con el resultado.

En nuestro **ejemplo ilustrativo**, el modelo solicita `multiply` con `a = 24` y `b = 17`. Antes de invocar la función, el ejecutor comprueba que el nombre esté registrado, que ambos argumentos existan y que cumplan el tipo y el intervalo acordados. También aplica las reglas de autorización correspondientes a la operación.

Conviene distinguir estas comprobaciones:

| Comprobación | Pregunta que resuelve |
|---|---|
| Análisis de la solicitud | ¿Podemos interpretar su estructura? |
| Validación del contrato | ¿Están los argumentos requeridos y tienen valores admitidos? |
| Reglas del dominio | ¿La operación tiene sentido en el estado actual? |
| Autorización | ¿La operación está permitida para este usuario y esta tarea? |

La separación es una **síntesis de diseño para nuestro agente**. No implica que cada comprobación necesite una clase o una capa independiente.

Un analizador JSON y un validador de esquemas resuelven problemas diferentes. La biblioteca Python `jsonschema`, por ejemplo, valida una instancia frente a un esquema; no extrae por sí sola cualquier llamada mezclada con prosa. [Documentación del proyecto](https://python-jsonschema.readthedocs.io/en/stable/validate/).

Si recibimos `a = "24"`, nuestra herramienta rechaza la cadena según su contrato. Si el nombre no existe, el ejecutor informa del error. No buscamos una función arbitraria en el entorno ni interpretamos texto generado como código ejecutable.

## Procesamiento de la salida de la herramienta

Devolver el **resultado de la herramienta (tool result)** permite que el modelo continúe con información sobre la operación ejecutada. La aplicación debe mantener la correspondencia entre solicitud y resultado, especialmente cuando existen varias llamadas.

Tras ejecutar `multiply` con los argumentos anteriores, la función devuelve `408`. Incorporamos ese resultado mediante la estructura que exige el adaptador y solicitamos al modelo el siguiente paso. No simulamos que el modelo multiplicó: registramos que pidió una operación y que el ejecutor obtuvo un resultado.

La representación del resultado depende de la interfaz. Ollama muestra mensajes con rol `tool`; Claude utiliza bloques `tool_result` dentro de un mensaje con rol `user` para herramientas del cliente. Ese nombre de rol forma parte del contrato de la API y no convierte el resultado en una instrucción nueva de la persona. [Ollama](https://docs.ollama.com/capabilities/tool-calling), [Claude](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview).

**Una salida de herramienta es evidencia de lo que esa operación devolvió, no una garantía de verdad universal.** Una búsqueda puede devolver un documento desactualizado; un servicio puede informar que aceptó un trabajo todavía pendiente. Para modificaciones relevantes necesitamos distinguir el mensaje de respuesta del efecto que debemos comprobar.

La respuesta final puede explicar o resumir la observación, pero no debe inventar una ejecución ausente. Si no obtuvimos un resultado, el agente debe reflejar esa limitación.

# Integración de las herramientas con el agente

Conectar herramientas exige coordinar el modelo, el registro, el ejecutor y el historial. La siguiente secuencia es el **diseño didáctico que construiremos**, no una afirmación de que ya existe una implementación ejecutable:

1. Preparar la solicitud del usuario, las instrucciones y las herramientas disponibles.
2. Obtener una salida del modelo.
3. Distinguir una respuesta final de una solicitud de herramienta.
4. Validar y autorizar cada solicitud antes de ejecutarla.
5. Registrar la operación y devolver su resultado o error.
6. Continuar si la tarea lo requiere y quedan recursos disponibles.

El modelo genera solicitudes; el ejecutor realiza operaciones; el historial conserva lo comunicado. El **registro de ejecución (trace)** debe permitir reconstruir qué ocurrió sin confundir una explicación generada con un evento ejecutado.

ReAct, presentado en octubre de 2022 y publicado en ICLR 2023, estudia la intercalación de texto de razonamiento, acciones y observaciones para resolver tareas. Conservamos la idea de observar resultados antes de ajustar el siguiente paso. Su formato de demostración no es obligatorio para todas las API. [Investigación original](https://arxiv.org/abs/2210.03629).

Una explicación del modelo tampoco proporciona acceso fiel a todo su proceso interno. Turpin y colaboradores muestran casos donde las explicaciones omiten factores que influyeron en las respuestas. Registraremos las salidas disponibles como salidas generadas, no como pruebas de pensamientos internos. [Estudio de 2023](https://arxiv.org/abs/2305.04388).

## Llamadas nativas y adaptadores

Las **llamadas nativas a herramientas (native tool calling)** permiten utilizar una interfaz compatible con el modelo sin diseñar desde cero un protocolo de texto. La implementación sigue dependiendo de un conjunto concreto: modelo, plantilla, motor de inferencia y API.

Un **adaptador (adapter)** traduce entre las estructuras de nuestra aplicación y las que espera ese conjunto. La definición que enviamos, los tokens de la plantilla y el objeto devuelto por el servicio son representaciones distintas. Hugging Face documenta cómo las plantillas de chat incorporan herramientas; no todas utilizan los mismos delimitadores. [Documentación de Transformers](https://huggingface.co/docs/transformers/en/chat_extras).

Gemma 4 ofrece un ejemplo documentado de llamadas a funciones y del procesamiento de sus respuestas. Su guía advierte que se validen nombres y argumentos y que se utilice un mapa de funciones permitidas. Esto describe una integración concreta, no un formato universal. [Documentación de Google](https://ai.google.dev/gemma/docs/capabilities/text/function-calling-gemma4).

Para nuestro agente comprobaremos el recorrido completo: la solicitud se interpreta, el ejecutor obtiene `408`, el resultado regresa por el adaptador y la respuesta final lo utiliza. Que aparezca un campo de llamadas en la respuesta de la API solo demuestra que recibimos una solicitud.

## Composición mediante código

Las **llamadas programáticas a herramientas (programmatic tool calling)** permiten expresar con código una secuencia de operaciones, bucles o transformaciones, en lugar de pedir al modelo una nueva decisión después de cada operación. El código se ejecuta en un entorno que debe ofrecer y controlar la aplicación.

Por ejemplo, para consultar varias fuentes independientes y conservar solo sus identificadores podríamos procesar los resultados antes de enviarlos al contexto del modelo. Es un **ejemplo de diseño**, no un programa ejecutado. La ejecución de código con MCP descrita por Anthropic en noviembre de 2025 muestra esta forma de composición. [Fuente primaria](https://www.anthropic.com/research/code-execution-with-mcp).

No la necesitamos para nuestra primera multiplicación. La evaluaremos cuando una tarea justifique agrupar operaciones. Los permisos, los límites de recursos y el registro de cada efecto siguen siendo necesarios; permitir composición no significa habilitar código arbitrario sin controles.

## Herramientas y memoria

La memoria permite conservar resultados útiles entre pasos. Al resumir una conversación debemos distinguir las instrucciones de funcionamiento, las definiciones de herramientas y los hechos observados. Reemplazar todo el contexto por un resumen puede eliminar información necesaria para continuar.

Nuestra decisión de diseño será conservar la relación entre una llamada pendiente y su resultado, y separar las observaciones confirmadas de lo que el modelo propuso hacer. Un resumen puede decir que se obtuvo `408`; no debe decir que una operación se completó si solo consta su solicitud.

La información sobre las herramientas también necesita mantenerse coherente con su implementación. Si cambia el contrato de `multiply`, debemos actualizar su definición y los ejemplos antes de reutilizarlos. La memoria no debe convertirse en la fuente de una interfaz antigua que contradiga la herramienta actual.

# Ejecución fiable y límites

Un agente necesita poder detenerse y explicar qué impidió continuar. Como **decisiones de implementación para nuestro agente**, estableceremos límites de pasos, tiempos de espera y recursos, además de un estado explícito para operaciones incompletas.

## Errores, operaciones pendientes y reintentos

Distinguiremos un error de argumentos, una operación denegada y un fallo del servicio. El primero puede requerir corregir la solicitud; el segundo exige respetar la denegación; el tercero puede admitir un reintento según sus condiciones. No todos los errores deben enviarse al modelo con detalles internos o credenciales.

Un **tiempo de espera agotado (timeout)** tampoco demuestra que el servicio no haya realizado la operación. Puede haber procesado una solicitud sin que recibamos la respuesta. Repetir una búsqueda y repetir la creación de un evento no tienen las mismas consecuencias.

Cuando una operación modifica datos, comprobaremos su estado antes de repetirla si el resultado quedó incierto. Si el servicio ofrece una **clave de idempotencia (idempotency key)**, podremos utilizarla según su contrato para reconocer reintentos de la misma operación. Una clave inventada que el servicio ignora no aporta esa propiedad.

Para trabajos diferidos conservaremos el identificador y el estado que devuelva el servicio. Una aceptación significa que el trabajo fue recibido, no que finalizó. Los mecanismos concretos de seguimiento pertenecen a cada integración.

El circuito se detendrá cuando tenga una respuesta final, necesite un dato o una confirmación, reciba una denegación que impida continuar o alcance un límite. No utilizaremos únicamente «no apareció una herramienta» como sinónimo de tarea completada: la salida podría ser incompleta o inválida.

## Resultados externos e inyección de instrucciones

La **inyección de instrucciones (prompt injection)** aparece cuando contenido externo intenta modificar el comportamiento del agente. Una página recuperada puede incluir órdenes para enviar información a otro sitio aunque la persona solo haya pedido un resumen.

AgentDojo, presentado en junio de 2024, evalúa este problema mediante agentes que utilizan herramientas sobre datos no confiables. Sus experimentos permiten estudiar ataques y defensas, no certificar una protección universal. [Publicación original](https://arxiv.org/abs/2406.13352).

En nuestro diseño, un documento recuperado es información para la tarea, no una autorización para ampliar permisos. Los controles del ejecutor deben limitar las operaciones y sus destinos. Las confirmaciones que correspondan se basarán en la acción concreta y sus argumentos, no en una descripción vaga generada por el modelo.

Una habilidad o integración de terceros también puede incorporar instrucciones o scripts. Cargar ese contenido no lo vuelve confiable ni le concede permisos adicionales. Esta frontera seguirá siendo necesaria aunque la llamada cumpla perfectamente un esquema.

# Comprobar el uso de herramientas

Para saber si el agente funciona debemos evaluar tanto sus decisiones como el resultado de la tarea. El formato correcto de una llamada es una condición parcial.

BFCL evalúa capacidades de llamadas a funciones, incluidas secuencias y abstención. τ-bench estudia interacción con usuarios y políticas del dominio, utilizando el estado final y la fiabilidad entre intentos. ToolSandbox incorpora herramientas con estado y dependencias entre operaciones. Son referencias para diseñar evaluaciones, no sustitutos de nuestros casos reales. [BFCL, ICML 2025](https://proceedings.mlr.press/v267/patil25a.html), [τ-bench, junio de 2024](https://arxiv.org/abs/2406.12045), [ToolSandbox, agosto de 2024](https://arxiv.org/abs/2408.04682).

Nuestra primera batería será una **propuesta propia de casos de prueba**, todavía no resultados de una ejecución:

| Caso | Comportamiento que comprobaremos |
|---|---|
| Multiplicar 24 por 17 | Solicitud válida, ejecución con resultado `408` y respuesta coherente |
| Falta un operando | Aclaración o consulta autorizada; ningún valor inventado |
| Llega una cadena en lugar de un entero | Rechazo del ejecutor según el contrato |
| Se solicita una herramienta desconocida | Error controlado; ninguna ejecución arbitraria |
| La herramienta disponible no sirve para la tarea | No utilizarla por obligación |
| El servicio falla o deja un trabajo pendiente | Conservar el estado y evitar afirmar éxito |
| Un resultado incluye una orden ajena a la tarea | No ampliar permisos a partir de ese contenido |
| Se alcanza el límite de pasos | Detenerse e informar del trabajo pendiente |

Distinguiremos **evaluación del proceso** y **evaluación del resultado**. La primera puede comprobar validación, permisos y trazabilidad; la segunda comprueba si se cumplió el objetivo. No exigiremos una única secuencia cuando existan varias rutas válidas, aunque las restricciones de autorización deben cumplirse en todas.

La guía de evaluación de Anthropic de enero de 2026 separa el registro de la interacción del estado final del entorno. Esa distinción será especialmente útil cuando incorporemos herramientas que modifican datos. [Fuente primaria](https://www.anthropic.com/research/demystifying-evals-for-ai-agents).

El capítulo de Evaluación desarrollará la medición entre intentos, los conjuntos de tareas y las comparaciones al cambiar de modelo. Aquí dejamos definidos los comportamientos que nuestra implementación deberá permitir observar.

# Aprendizaje de herramientas

Comprender el **aprendizaje de herramientas (tool learning)** ayuda a elegir qué esperamos del modelo y qué debemos implementar nosotros. Proporcionar ejemplos durante una solicitud, ajustar parámetros con datos y optimizar mediante recompensas son mecanismos distintos que pueden combinarse.

No seguiremos una escalera obligatoria de ICL a SFT y después a RL. Para empezar utilizaremos un modelo compatible y evaluaremos su comportamiento. Entrenarlo requeriría un objetivo y evidencia que justifiquen ese trabajo adicional.

## Aprendizaje en contexto

El **aprendizaje en contexto (in-context learning, ICL)** permite orientar la respuesta mediante instrucciones y ejemplos presentes en la entrada, sin actualizar los parámetros del modelo durante esa inferencia. La selección y presentación de los ejemplos puede afectar al resultado. [Survey de Dong y colaboradores](https://arxiv.org/abs/2301.00234).

Un ejemplo de `multiply` puede mostrar los argumentos enteros `24` y `17` y el resultado esperado `408`. Si lo incluimos como demostración, debe quedar identificado como tal. No lo registraremos como una ejecución real del agente.

Los ejemplos ayudan a comunicar el contrato, pero pueden ser incompletos o inducir un patrón inadecuado. Debemos probar solicitudes distintas de las demostraciones. Agregar más ejemplos tampoco garantiza una mejora: consume contexto y puede introducir contradicciones.

## Ajuste fino supervisado y Toolformer

El **ajuste fino supervisado (supervised fine-tuning, SFT)** modifica parámetros utilizando ejemplos de comportamiento esperado. Para herramientas, esos datos pueden contener solicitudes, argumentos, resultados y respuestas. Su calidad debe contemplar errores y distintas formas válidas de resolver la tarea.

Toolformer, presentado en febrero de 2023, es un antecedente de entrenamiento **autosupervisado (self-supervised learning)** para herramientas. A partir de pocas demostraciones por API, genera llamadas candidatas sobre texto, las ejecuta y filtra aquellas cuyo resultado ayuda a predecir los tokens posteriores. Después ajusta el modelo con los datos enriquecidos. Su criterio de filtrado se basa en pérdida predictiva, no en una verificación general de verdad. [Método original](https://arxiv.org/html/2302.04761v1).

El trabajo utiliza GPT-J y demuestra mejoras en las tareas estudiadas. También reconoce limitaciones para encadenar herramientas en su configuración. No permite concluir que todas las aplicaciones actuales deban insertar llamadas en texto ni que un formato particular sea universal.

Para nuestro agente, la lección es separar demostraciones, datos de entrenamiento y registro de ejecución. No necesitamos reproducir Toolformer para registrar nuestra función.

## Aprendizaje por refuerzo

El **aprendizaje por refuerzo (reinforcement learning, RL)** ajusta el comportamiento mediante recompensas. En uso de herramientas, estas pueden valorar argumentos, decisiones intermedias o el resultado final, según el entorno de entrenamiento.

El **razonamiento integrado con herramientas (tool-integrated reasoning, TIR)** intercala generación del modelo, solicitudes y resultados externos para continuar la resolución. No es exclusivo de RL: las instrucciones, los ejemplos y distintos entrenamientos pueden favorecer ese patrón. El término describe cómo se organiza la interacción; no demuestra que sus explicaciones sean un registro fiel del proceso interno. Search-R1 es uno de los estudios de entrenamiento que veremos a continuación.

Diseñar la recompensa es parte del problema. Coincidir con una llamada de referencia no siempre demuestra que se cumplió la tarea; una ruta distinta podría ser válida. Del mismo modo, obtener el resultado esperado no demuestra que se respetaron permisos o límites.

SFT y RL pueden complementarse. Ninguno garantiza generalización, y la comparación depende de los datos, modelos, tareas y criterios evaluados. No trasladaremos una mejora experimental a una promesa sobre cualquier herramienta de nuestro agente.

### ToolRL: recompensas para solicitudes de herramientas

ToolRL, publicado en abril de 2025, estudia recompensas de formato y corrección de llamadas, descomponiendo esta última en aspectos como nombre de herramienta y parámetros. Utiliza GRPO y experimenta con modelos Qwen2.5 y otros modelos pequeños. [Investigación original](https://arxiv.org/html/2504.13958v1).

Sus resultados permiten estudiar cómo influye el diseño de recompensas dentro de esas condiciones. El artículo también evalúa incentivar salidas más largas: aumentar su longitud no mejora consistentemente la tarea y puede perjudicar algunos modelos. Esto no prueba que razonar más sea siempre inútil; cuestiona utilizar la longitud como sustituto del éxito.

Para nosotros, aporta una distinción útil entre formato correcto y solicitud correcta. No adoptaremos su configuración de entrenamiento como requisito de implementación.

### Search-R1: aprender a buscar durante la generación

Search-R1, presentado en marzo de 2025 y revisado en agosto del mismo año, entrena modelos para intercalar generación y consultas a un buscador. El sistema ejecuta la búsqueda e incorpora el contenido recuperado para continuar. [Investigación original](https://arxiv.org/html/2503.09516v5).

El trabajo estudia PPO y GRPO; en la versión citada utiliza PPO por defecto. Excluye los tokens recuperados del objetivo de optimización mediante **enmascaramiento de la pérdida (loss masking)**: son observaciones externas, no acciones generadas por el modelo.

Sus evaluaciones se centran en tareas de preguntas y respuestas con una configuración concreta de recuperación. No demuestran que la misma recompensa permita aprender cualquier herramienta con efectos externos. Para nuestro agente conservamos la distinción entre solicitar una búsqueda, ejecutarla y utilizar lo recuperado.

# Protocolo de contexto del modelo

El **Protocolo de contexto del modelo (Model Context Protocol, MCP)** permite reutilizar integraciones entre aplicaciones de IA y fuentes de datos o herramientas. Anthropic lo presentó el 25 de noviembre de 2024. Su utilidad es acordar cómo describir y solicitar capacidades, reduciendo trabajo de integración repetido. [Anuncio original](https://www.anthropic.com/news/model-context-protocol).

MCP no convierte automáticamente cualquier modelo en un buen usuario de herramientas. La aplicación sigue decidiendo qué presenta al modelo, cómo interpreta su solicitud y qué operaciones permite ejecutar. Tampoco elimina las credenciales, los cambios de API o las pruebas de compatibilidad.

Para los detalles del protocolo, este capítulo toma como referencia la **revisión 2026-07-28**, a la que dirige la especificación oficial consultada. Frente a 2025-11-25, esa revisión elimina el intercambio inicial y las sesiones del protocolo, e incorpora los metadatos de versión y capacidades en cada solicitud. Las tareas experimentales pasan a una extensión. Los clientes antiguos pueden requerir otro flujo; no mezclaremos ejemplos de ambas revisiones. [Especificación](https://modelcontextprotocol.io/specification/2026-07-28), [cambios entre versiones](https://modelcontextprotocol.io/specification/2026-07-28/changelog).

## Componentes principales

El protocolo distingue tres participantes:

| Participante | Responsabilidad en la integración |
|---|---|
| **Aplicación anfitriona (host)** | Coordina el modelo, el contexto, los permisos y la interacción con la persona |
| **Cliente (client)** | Implementa la comunicación MCP dentro del host |
| **Servidor (server)** | Expone capacidades y conecta con la implementación o el servicio correspondiente |

El servidor puede ofrecer **herramientas (tools)**, **recursos (resources)** y **plantillas de instrucciones (prompts)**. Son conceptos diferentes: una plantilla aporta instrucciones reutilizables; un recurso proporciona contenido; una herramienta permite solicitar una operación. La aplicación decide cómo integrarlos en su experiencia. [Especificación del protocolo](https://modelcontextprotocol.io/specification/2026-07-28).

MCP utiliza mensajes JSON-RPC 2.0 para esa comunicación. Esto no obliga al modelo a producir directamente mensajes del protocolo: el host y su cliente traducen entre la interfaz del modelo y la solicitud MCP. [Especificación JSON-RPC](https://www.jsonrpc.org/specification).

## El flujo MCP

Consideremos un **ejemplo ilustrativo**: una persona pide resumir los cinco commits más recientes de un repositorio al que tiene acceso.

1. El host obtiene, mediante su cliente, las herramientas expuestas por un servidor MCP autorizado.
2. Selecciona las definiciones que presentará al modelo para esa tarea.
3. El modelo solicita la herramienta correspondiente con el repositorio y el límite de cinco resultados.
4. El host valida y autoriza la solicitud; el cliente la comunica al servidor.
5. El servidor ejecuta la operación mediante su integración y devuelve datos o un error.
6. El host conserva la asociación entre solicitud y resultado y entrega lo pertinente al modelo para redactar el resumen.

En MCP, `tools/list` permite enumerar herramientas y `tools/call` solicitarlas. El servidor expone las que implementa; no tiene por qué devolver todos los endpoints del servicio subyacente. La especificación también define esquemas y resultados estructurados. [Herramientas en MCP](https://modelcontextprotocol.io/specification/2026-07-28/server/tools).

Un servidor mantenido puede centralizar cambios de su integración, pero una actualización puede afectar nombres, argumentos o permisos. Necesitamos verificar la combinación de versiones que utilicemos.

Las anotaciones de una herramienta describen su comportamiento; no prueban que sea segura. La especificación exige considerar esas anotaciones no confiables salvo que procedan de servidores confiables. Los controles de acceso siguen siendo responsabilidad de la integración. [Requisitos de herramientas](https://modelcontextprotocol.io/specification/2026-07-28/server/tools).

Para nuestro agente incorporaremos MCP cuando una integración reutilizable lo justifique. No lo necesitamos para ejecutar `multiply` localmente. **A2A (Agent2Agent)** aborda comunicación entre aplicaciones de agentes; lo reservamos para el capítulo de sistemas multiagente, sin tratarlo como sustituto de MCP. [Proyecto oficial A2A](https://github.com/a2aproject/A2A).

# Habilidades

Las **habilidades (skills)** permiten reutilizar instrucciones, referencias y recursos para una tarea. Una herramienta define una operación que podemos solicitar; una habilidad explica un procedimiento que puede combinar varias operaciones.

Agent Skills proporciona un formato abierto para organizar ese material. Una habilidad contiene al menos un archivo `SKILL.md` con metadatos e instrucciones; puede incluir scripts y otros recursos. Su compatibilidad depende de la aplicación que los descubre y utiliza. [Descripción del formato](https://agentskills.io/home).

En un **ejemplo propio**, una habilidad `meeting-notes` podría explicar cómo transformar una transcripción en decisiones y acciones pendientes. Sus instrucciones deben distinguir quién asumió un compromiso, qué fecha se mencionó y qué información no consta. No autorizarían enviar esas notas a otras personas si la tarea solo pide redactarlas.

## El archivo SKILL.md

`SKILL.md` combina una cabecera **YAML (frontmatter)** con un cuerpo Markdown. Los campos obligatorios son `name` y `description`. El nombre debe seguir las restricciones del formato y coincidir con el directorio; por eso utilizamos `meeting-notes`, con guion. La descripción ayuda a determinar cuándo conviene cargar la habilidad. [Especificación de Agent Skills](https://agentskills.io/specification).

La **divulgación progresiva (progressive disclosure)** organiza la carga de información: primero los metadatos, después las instrucciones al activar la habilidad y finalmente los recursos necesarios. El host debe implementar ese comportamiento; crear una carpeta no activa por sí solo la carga progresiva.

Las cantidades de tokens que ofrece la especificación son orientaciones de diseño. Los metadatos reales varían y un cuerpo extenso sigue consumiendo contexto. La carga bajo demanda reduce información inicial, pero no garantiza que todas las tareas requieran menos tokens.

## Los recursos agrupados

En nuestra habilidad ilustrativa podríamos separar una plantilla de notas, referencias sobre distintos tipos de reunión y un script opcional para transformar datos. **No todos los recursos necesitan leerse o ejecutarse en cada tarea.** Las instrucciones deben indicar cuándo corresponde consultar cada uno.

Una plantilla puede especificar encabezados y campos obligatorios. Un script, en cambio, realiza una operación y necesita ejecutarse mediante una herramienta, con los permisos y límites que correspondan. El contenido de la habilidad no sustituye ese control.

Si la transcripción no incluye una fecha de entrega, las notas deben decir que no consta o pedir aclaración, según la tarea. El formato esperado no justifica inventar datos para rellenarlo.

Para nuestro agente, una habilidad será pertinente cuando exista un procedimiento recurrente que convenga mantener aparte de las instrucciones generales. Antes de añadirla comprobaremos que la aplicación pueda descubrirla, cargar sus recursos y completar tareas representativas sin asumir permisos nuevos.

# Lo que deja definido este capítulo

Tenemos el diseño de un circuito donde el modelo solicita operaciones, el ejecutor aplica sus contratos y el historial conserva resultados observados. La herramienta `multiply` mantiene el mismo contrato de enteros en todos los ejemplos. Las consultas, la memoria, las integraciones y las habilidades se conectan mediante responsabilidades distinguibles.

**El texto no demuestra que el agente esté implementado ni que haya superado las pruebas propuestas.** Para esa implementación necesitaremos verificar el registro, el adaptador, los límites y los casos de éxito y error. Sus resultados se registrarán por separado de los ejemplos ilustrativos.

Las investigaciones explican mecanismos y límites: ReAct estudia intercalar acciones y observaciones; Toolformer genera datos para aprender uso de API; ToolRL analiza recompensas; Search-R1 entrena interacción con búsqueda. No convierten una técnica en requisito universal.

MCP aporta un protocolo de integración y Agent Skills un formato para procedimientos reutilizables. Ninguno elimina la necesidad de validar operaciones, comprobar resultados y evaluar la tarea completa.

En el capítulo de planificación utilizaremos este circuito para decidir secuencias y adaptar el siguiente paso a las observaciones. La evaluación del agente deberá comprobar tanto el objetivo cumplido como las restricciones respetadas durante su ejecución.

# Referencias y alcance de las fuentes

Las publicaciones siguientes respaldan los mecanismos que se explican en el capítulo. Sus resultados pertenecen a sus condiciones experimentales. Las recomendaciones para nuestro agente se identifican como decisiones didácticas o propuestas de diseño. La documentación viva se consultó el 3 de noviembre de 2026; esa fecha no es su fecha de publicación.

| Fuente primaria | Fecha o versión y uso en el capítulo |
|---|---|
| [Wang y colaboradores: What Are Tools Anyway?](https://arxiv.org/abs/2403.15452) | 18/03/2024. Herramientas como interfaces a programas externos |
| [Yao y colaboradores: ReAct](https://arxiv.org/abs/2210.03629) | Primera versión 06/10/2022; revisión 10/03/2023. Generación, acciones y observaciones |
| [Turpin y colaboradores: Unfaithful Explanations](https://arxiv.org/abs/2305.04388) | Primera versión 07/05/2023. Límites de las explicaciones generadas |
| [Dong y colaboradores: A Survey on In-context Learning](https://arxiv.org/abs/2301.00234) | Primera versión 31/12/2022; revisión 05/10/2024. Orientación mediante contexto |
| [Schick y colaboradores: Toolformer](https://arxiv.org/abs/2302.04761) | 09/02/2023. Datos autosupervisados para uso de herramientas |
| [Qin y colaboradores: ToolLLM](https://arxiv.org/abs/2307.16789) | Primera versión 31/07/2023. Recuperación en catálogos amplios |
| [Qian y colaboradores: ToolRL](https://arxiv.org/abs/2504.13958) | 16/04/2025. Recompensas para uso de herramientas |
| [Jin y colaboradores: Search-R1](https://arxiv.org/abs/2503.09516) | Primera versión 12/03/2025; revisión citada 05/08/2025. Búsqueda y entrenamiento con RL |
| [DeepSeek-AI: DeepSeek-V3.2](https://arxiv.org/abs/2512.02556) | 02/12/2025. Ejemplo de entrenamiento con tareas de agente sintetizadas |
| [Patil y colaboradores: BFCL](https://proceedings.mlr.press/v267/patil25a.html) | ICML 2025. Evaluación de llamadas a funciones |
| [Yao y colaboradores: τ-bench](https://arxiv.org/abs/2406.12045) | 17/06/2024. Políticas, estado final y fiabilidad |
| [Lu y colaboradores: ToolSandbox](https://arxiv.org/abs/2408.04682) | Primera versión 08/08/2024; revisión 16/04/2025. Herramientas con estado |
| [Debenedetti y colaboradores: AgentDojo](https://arxiv.org/abs/2406.13352) | Primera versión 19/06/2024. Evaluación de inyección de instrucciones |
| [Anthropic: Building effective agents](https://www.anthropic.com/research/building-effective-agents) | 19/12/2024. Distinción entre flujos y selección dinámica; no se adopta todo su tooling como actual |
| [Anthropic: Writing effective tools](https://www.anthropic.com/research/writing-tools-for-agents) | 11/09/2025. Diseño y evaluación orientados a tareas |
| [Anthropic: Structured outputs](https://claude.com/resources/articles/structured-outputs-on-the-claude-developer-platform) | 14/11/2025; actualización de disponibilidad general 04/02/2026. Conformidad con esquemas |
| [Anthropic: Code execution with MCP](https://www.anthropic.com/research/code-execution-with-mcp) | 04/11/2025. Composición y procesamiento de resultados mediante código |
| [Anthropic: Advanced tool use](https://www.anthropic.com/research/advanced-tool-use) | 24/11/2025. Búsqueda de herramientas, llamadas programáticas y ejemplos |
| [Anthropic: Improved web search](https://claude.com/resources/articles/improved-web-search-with-dynamic-filtering) | 17/02/2026. Anuncio de disponibilidad general de capacidades de herramientas |
| [Anthropic: Demystifying evals for AI agents](https://www.anthropic.com/research/demystifying-evals-for-ai-agents) | 09/01/2026. Registro de interacción y resultado final |
| [Anthropic: Introducing MCP](https://www.anthropic.com/news/model-context-protocol) | 25/11/2024. Origen del protocolo |

Las referencias normativas de integración son [MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28), su [historial de cambios](https://modelcontextprotocol.io/specification/2026-07-28/changelog), la sección de [herramientas](https://modelcontextprotocol.io/specification/2026-07-28/server/tools), [JSON-RPC 2.0](https://www.jsonrpc.org/specification) y la [especificación de Agent Skills](https://agentskills.io/specification).

Para tipos, validación y adaptación se utilizan la referencia de [JSON Schema](https://json-schema.org/understanding-json-schema/reference/numeric), [jsonschema para Python](https://python-jsonschema.readthedocs.io/en/stable/validate/), [Claude](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview), [definiciones de Claude](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools), [Gemini](https://ai.google.dev/gemini-api/docs/function-calling), [Ollama](https://docs.ollama.com/capabilities/tool-calling), [Transformers](https://huggingface.co/docs/transformers/en/chat_extras) y [Gemma 4](https://ai.google.dev/gemma/docs/capabilities/text/function-calling-gemma4). Sus formatos no se presentan como intercambiables sin un adaptador.
