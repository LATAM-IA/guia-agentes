# Capítulo 10. Agentes de código y modelos de lenguaje para código

Un **agente de código (coding agent)** permite realizar tareas que requieren escribir, ejecutar o modificar programas. Puede preparar un análisis para una persona que no programa o trabajar sobre un repositorio con un equipo de ingeniería. Su utilidad depende del resultado que entrega y de los cambios que produce, no de cuánto código genera.

Un modelo puede proponer una función en una respuesta. El agente añade herramientas, contexto y control de ejecución para leer archivos, solicitar operaciones, recibir sus resultados y continuar. **Generar código no equivale a ejecutarlo; ejecutarlo sin errores no demuestra que resuelva la necesidad.**

En los capítulos anteriores definimos observaciones, herramientas, planificación y evaluación. Aquí los conectaremos con dos tareas relacionadas: consultar registros de un día para producir un análisis y añadir un filtro por fecha a una aplicación existente. Los ejemplos serán **ilustrativos y propios**, sin modificar una aplicación real ni ejecutar un agente durante la lectura.

El código ofrece comprobaciones útiles: podemos ejecutar un programa y contrastar su comportamiento con casos definidos. Aun así, una prueba solo cubre lo que comprueba. Seguridad, integración y utilidad pueden requerir evidencias adicionales.

# Quién utiliza y quién construye estos sistemas

El objetivo de la persona determina qué interfaz necesita y qué decisiones debe poder revisar. No trataremos a quien pide un gráfico como si necesariamente pudiera auditar el programa generado para producirlo.

| Participante | Qué necesita recibir o controlar |
|---|---|
| Persona que pide un resultado | Datos utilizados, artefacto final, limitaciones y efectos relevantes |
| Persona que construye software mediante lenguaje natural | Comportamiento visible, decisiones de alcance y oportunidades de revisión |
| Profesional que trabaja sobre un repositorio | Cambios concretos, pruebas, contexto técnico y control de integración |
| Quien construye el agente | Herramientas, permisos, estado, interfaces y evaluación del sistema |
| Quien entrena el modelo | Datos, objetivos de entrenamiento y validación de capacidades |

Estos papeles pueden coincidir en una persona. Distinguirlos ayuda a decidir qué debemos explicar, qué puede automatizarse y quién es responsable de autorizar una acción con consecuencias.

Una persona puede describir lo que necesita sin conocer la estructura del código. El agente tendrá que investigar o pedir aclaraciones antes de fijar una implementación. No convertiremos una solicitud incompleta en permiso para decidir requisitos de negocio.

Entrenar un modelo y construir un agente también son tareas distintas. Podemos desarrollar la aplicación usando un modelo ya disponible; disponer de un modelo especializado no configura por sí solo permisos, herramientas o criterios de finalización.

# Construir un agente de código

Empezaremos por delimitar una tarea y el entorno donde puede realizarse. El agente debe saber qué datos puede leer, qué puede modificar y qué evidencia necesita para entregar el resultado.

## Obtener un resultado sin exigir que la persona programe

Para continuar el ejemplo del filtro, supongamos un archivo autorizado con estos datos sintéticos. El contrato define fechas en UTC y un intervalo que incluye el inicio del día y excluye el inicio del siguiente.

| Registro | Fecha y hora UTC | Unidades |
|---|---|---|
| A | 2026-11-02 23:59:59 | 2 |
| B | 2026-11-03 00:00:00 | 3 |
| C | 2026-11-03 12:00:00 | 5 |
| D | 2026-11-03 23:59:59 | 4 |
| E | 2026-11-04 00:00:00 | 7 |

La persona pide consultar el 3 de noviembre de 2026. El resultado esperado contiene B, C y D: **tres registros y doce unidades**. Un gráfico de unidades por día tendría que distinguir el conteo de filas de la suma de unidades; son medidas diferentes.

El agente puede necesitar obtener los datos por tres rutas:

- **Archivo aportado:** leer el archivo autorizado y comprobar formato, columnas y valores ausentes.
- **Archivo que debe localizar:** buscar dentro del directorio permitido, sin explorar toda la computadora por defecto.
- **Base de datos:** consultar las tablas autorizadas con un contrato de lectura y reglas de interpretación del esquema.

Después podrá solicitar la transformación y, si se pidió, la generación de un artefacto como un gráfico. El ejecutor realiza el programa y devuelve resultados, errores y referencias a los archivos producidos. El agente debe comprobar que el artefacto existe y corresponde a los datos antes de anunciar que está listo.

Si faltan datos o la zona horaria no está definida, la siguiente acción puede ser pedir una aclaración. Un error de ejecución puede aportar información para corregir el programa, pero repetir intentos sin entender el error no garantiza avanzar.

No es obligatorio mostrar todo el código a la persona que pidió el análisis. Sí debemos poder explicar de dónde salen las cifras, qué limitaciones tienen y qué operaciones con efectos se realizaron. La responsabilidad de contener esos efectos no se traslada al usuario por ocultarle el código.

## Herramientas de código

Las herramientas conectan las solicitudes del modelo con operaciones concretas. Definiremos entradas, salidas, permisos y errores para cada una, manteniendo la distinción entre **solicitud**, **ejecución** y **resultado observado**.

Una operación puede devolver datos estructurados, texto, un error o una referencia a un archivo. No necesitamos reducir todas las salidas a una cadena indistinta ni usar la misma señal para «solicitado», «terminado» y «fallido».

### Lectura y edición de archivos

Las operaciones de archivos permiten localizar código, leer fragmentos y preparar cambios. Empezaremos por acceso de lectura a un directorio conocido. Si la tarea requiere escritura, delimitaremos los archivos o rutas permitidos y cómo se revisará el cambio.

Para leer conviene identificar archivo, versión y rango de contenido. Para editar necesitamos saber sobre qué estado se aplica el cambio. Si otra persona modificó el archivo después de nuestra lectura, una sustitución completa basada en contenido antiguo puede borrar su trabajo.

En nuestro diseño, el ejecutor podrá comprobar una versión o un contenido esperado antes de aplicar un parche. Si la comprobación no coincide, devolverá un conflicto para releer y decidir. El modelo no debe interpretar ese conflicto como autorización para forzar la escritura.

La restricción de rutas debe hacerse cumplir en el software. Un nombre que parece estar dentro del proyecto puede resolver mediante enlaces a otro lugar. Definiremos la frontera real de acceso, sin asumir que un prefijo de texto representa todo el aislamiento necesario.

### Intérprete de código y entorno aislado

Un **intérprete de código (code interpreter)** ejecuta programas. Un **entorno aislado (sandbox)** aplica una frontera sobre recursos y efectos. Tener un intérprete o iniciar un proceso local no garantiza que exista esa frontera.

Un contenedor y una máquina virtual son mecanismos diferentes; su seguridad depende de configuración, montajes, privilegios y acceso al sistema. La documentación de **gVisor** describe una capa de aislamiento que intercepta llamadas al sistema mediante un núcleo de aplicación por sandbox. Es un ejemplo de frontera adicional, no una promesa de seguridad para cualquier configuración. [Documentación oficial](https://gvisor.dev/docs/).

Para nuestro agente definiremos directorios montados, permisos, conectividad y límites de tiempo, procesos, memoria y almacenamiento. Un entorno por sesión ayuda a separar estados, pero no garantiza por sí solo que registros, copias o archivos exportados desaparezcan al terminar. La conservación y eliminación necesitan una política explícita.

La API de Gemini documenta una herramienta de ejecución de Python. Eso describe una capacidad del servicio, no un mecanismo idéntico al de cualquier otro proveedor ni una garantía sobre ejecutar código en nuestra propia computadora. [Documentación oficial de ejecución](https://aistudio.google.com/docs/code-execution).

Podemos elegir un servicio o un ejecutor propio según la tarea y sus restricciones. La primera integración no necesita acceso general a internet ni a las credenciales de la persona para calcular las doce unidades del ejemplo.

### Línea de comandos y preparación del entorno

Una **herramienta de línea de comandos (command-line tool)** permite ejecutar programas, preparar dependencias o lanzar pruebas. Puede operar dentro de un entorno restringido; no es necesariamente el nivel siguiente de una escala que siempre conduce a mayor libertad.

El ejecutor debe conocer el directorio de trabajo y los límites de cada proceso. Registraremos comando solicitado, ejecución iniciada, salida, estado y código de finalización. Un identificador de proceso activo no equivale a un resultado terminado.

Preparar el entorno también puede ejecutar código: instalar una dependencia, lanzar un script del repositorio o iniciar una herramienta puede activar comportamientos que no aparecen en el comando superficial. Los contenidos del proyecto no se vuelven confiables por estar en una carpeta local.

Para nuestra tarea utilizaremos los comandos documentados y el entorno necesario para reproducir el problema. Si falta una herramienta, distinguiremos entre informar la limitación e instalarla; la autorización para editar una función no siempre incluye modificar la máquina o acceder a nuevos servicios.

Una confirmación humana puede ser útil si presenta el alcance y los efectos de forma comprensible. No sustituye los límites del ejecutor ni funciona como garantía cuando la persona no tiene información suficiente para evaluar la operación.

### Uso de computadoras e inspección visual

Las herramientas de **uso de computadoras (computer use)** permiten observar interfaces y solicitar acciones sobre ellas. Pueden ayudar a verificar que un cambio de código produce el comportamiento esperado en la aplicación.

Para el filtro por fecha, ejecutar la prueba de la consulta y utilizar el control en el navegador comprueban cosas distintas. La consulta puede estar bien mientras la interfaz envía una fecha equivocada o no permite completar la tarea.

Una captura describe un instante y una vista. Si el estado cambió, las coordenadas propuestas pueden haber quedado obsoletas. El agente debe asociar la solicitud con la observación pertinente y comprobar el resultado de la acción, en lugar de asumir éxito porque hizo clic.

La percepción visual amplía la evidencia disponible, pero no revela el pensamiento interno del modelo ni autoriza acciones sugeridas por texto incrustado en una pantalla. Mantendremos los límites de origen y permisos del capítulo de comprensión multimodal.

### Búsqueda de código

La búsqueda textual permite localizar nombres, mensajes de error y referencias. Una búsqueda estructural sobre un **árbol de sintaxis abstracta (abstract syntax tree, AST)** reconoce patrones de código según su estructura.

**ast-grep** documenta patrones con elementos de código y metavariables. Puede evitar depender de diferencias irrelevantes de formato, pero no ignora todos los nombres automáticamente ni equivale a un análisis completo de tipos y efectos. La regla debe expresar qué puede variar y qué debe coincidir. [Documentación oficial](https://ast-grep.github.io/guide/pattern-syntax.html).

La **búsqueda semántica (semantic search)** utiliza representaciones para recuperar candidatos relacionados con una consulta, como «dónde se filtran los registros». No demuestra que un candidato sea la única implementación ni recupera necesariamente todos los usos relevantes.

**CoIR**, publicado el 3 de julio de 2024, reúne tareas para evaluar recuperación de información de código. Ayuda a estudiar esa capacidad, pero una puntuación del modelo de recuperación no valida un parche producido después. [Publicación original](https://arxiv.org/abs/2407.02883).

En nuestro agente combinaremos búsqueda y lectura según la pregunta. Antes de cambiar la función del filtro, comprobaremos su implementación y sus consumidores. Si el índice pertenece a otra revisión del repositorio, los resultados pueden estar desactualizados.

### Consultas SQL y significado del esquema

Una herramienta **SQL** permite consultar una base de datos según sus permisos. El modelo puede generar una consulta, pero la aplicación debe validar el alcance y el servicio ejecutarla con credenciales adecuadas.

La **vinculación de esquemas (schema linking)** relaciona expresiones de la solicitud con tablas, columnas y valores. Una **capa semántica (semantic layer)** puede aportar definiciones de negocio, como qué estados cuentan para un informe o qué zona horaria corresponde a una fecha.

Si alguien pide «registros del día 3», el agente necesita saber qué columna representa la fecha y cómo está almacenada. Que una consulta sea SQL válido no demuestra que interprete correctamente esa intención.

**Spider 2.0**, publicado el 12 de noviembre de 2024, estudia flujos de texto a SQL con bases y operaciones de contexto empresarial. Es evidencia de la diferencia entre traducir una pregunta simple y resolver tareas con metadatos y varias operaciones. [Publicación original](https://arxiv.org/abs/2411.07763).

Para empezar usaremos permisos de lectura y límites de consulta. PostgreSQL documenta el modo de transacción **READ ONLY** y las operaciones que restringe. Ese modo no sustituye controlar roles, recursos y superficies adicionales del servicio. Tampoco aceptaremos «empieza por SELECT» como prueba completa de ausencia de efectos. [Documentación oficial](https://www.postgresql.org/docs/current/sql-set-transaction.html).

Los significados de negocio se mantendrán como información revisable y con procedencia. Una corrección propuesta a partir de un error de consulta puede ser una hipótesis útil; no debe convertirse automáticamente en una regla institucional permanente.

## Permisos y contenido no confiable

La tarea define qué efectos puede producir el agente. El **principio de mínimo privilegio (principle of least privilege)** consiste en proporcionar solo el acceso necesario para ese alcance, no en confiar en que una instrucción compensará permisos ilimitados.

**RedCode**, publicado el 12 de noviembre de 2024, propone evaluar generación y ejecución de código riesgoso. **SandboxEval**, publicado el 27 de marzo de 2025, presenta una batería experimental para estudiar propiedades de entornos que ejecutan código no confiable. Son evaluaciones de superficies y configuraciones concretas, no certificados de seguridad universal. [RedCode](https://arxiv.org/abs/2411.07781), [SandboxEval](https://arxiv.org/abs/2504.00018).

Un archivo del repositorio, una incidencia o la salida de una herramienta pueden contener **inyección de instrucciones (prompt injection)**. El agente debe utilizarlos como información de la tarea bajo las reglas autorizadas, sin permitir que se otorguen nuevos permisos a sí mismos.

El estudio **Your Agent, Their Asset**, publicado el 6 de abril de 2026, analiza envenenamiento del estado persistente en OpenClaw. Conservamos su lección sobre proteger capacidades, instrucciones e información guardada. Sus resultados corresponden a la plataforma y configuración estudiadas; no prueban que toda arquitectura de agente sea irreparable. [Investigación original](https://arxiv.org/abs/2604.04759).

En nuestro diseño protegeremos el estado de autorización y los verificadores frente al código que se está evaluando. El proceso de trabajo tampoco recibirá secretos o acceso de publicación si no los necesita. Aceptar una operación concreta no cambia automáticamente el alcance de las siguientes.

# Gestionar el contexto del repositorio

El modelo necesita suficiente contexto para localizar el problema y proponer un cambio coherente. El desafío no es introducir todos los archivos, sino seleccionar información vigente sin perder requisitos, dependencias o evidencias importantes.

## Instrucciones del proyecto y fuentes de contexto

Leeremos las instrucciones pertinentes, documentación y estado del checkout antes de modificar código. Debemos distinguir convenciones del proyecto, petición del usuario, contenido de archivos y resultados de herramientas; no todos tienen el mismo papel ni autoridad.

El formato **AGENTS.md** ofrece un lugar predecible para instrucciones sobre preparación, pruebas y convenciones. Es documentación para orientar agentes, no un mecanismo de aislamiento o de autorización del sistema operativo. Su alcance depende también de la herramienta que lo interpreta. [Especificación y ejemplos oficiales](https://agents.md/).

Para el filtro buscaremos el contrato de fechas, la implementación actual, pruebas y consumidores. Una descripción de alto nivel puede orientar esa búsqueda, pero no sustituye leer el fragmento que vamos a editar.

Conservaremos rutas y revisión de las fuentes. Si recibimos un resumen sin evidencia suficiente, podremos comprobar el archivo actual en vez de tratar el resumen como una transcripción exacta del código.

## Mapas del repositorio

Un **mapa del repositorio (repository map)** resume archivos y relaciones útiles para orientarse. Puede contener estructura, símbolos, firmas y referencias, sin incluir cada línea de implementación.

La documentación de **Aider** describe un mapa compacto que selecciona símbolos relevantes mediante un grafo de dependencias y un presupuesto de tokens. Es un diseño documentado de esa herramienta, no una prueba de que todo resumen deba usar el mismo algoritmo. [Fuente de los autores](https://aider.chat/docs/repomap.html).

Un mapa sirve para decidir dónde leer después. Puede omitir funciones pequeñas pero decisivas o quedar desactualizado tras cambios. No lo consideraremos una demostración de que conocemos todos los efectos de un parche.

Para nuestro agente mantendremos el mapa asociado a una revisión o lo regeneraremos cuando sea necesario. Si la tarea es pequeña, una búsqueda dirigida y lectura de unos archivos pueden resultar suficientes; no añadiremos un índice complejo por defecto.

## Caché KV, caché de prefijos y caché del prompt

La **caché de claves y valores (key-value cache, KV cache)** conserva representaciones de atención para evitar recalcular parte del procesamiento previo. La **caché de prefijos (prefix caching)** puede reutilizar trabajo de un prefijo compartido entre solicitudes. Una función de **caché del prompt (prompt caching)** expuesta por un proveedor añade su contrato de uso, vigencia y facturación. El prompt puede contener instrucciones, mensajes y otros elementos admitidos por la API; su contenido no se limita a instrucciones.

Estos conceptos se relacionan, pero no son sinónimos de memoria del agente ni de guardar una respuesta completa para reutilizarla. La documentación de **vLLM** describe reutilización de caché KV para prefijos compartidos y distingue el ahorro en procesamiento de entrada de la generación de salida: «does not reduce the time of generating new tokens». Es decir, no reduce el tiempo de generar los tokens nuevos. [Documentación oficial](https://docs.vllm.ai/en/latest/features/automatic_prefix_caching/).

Para favorecer reutilización podemos mantener estable un prefijo con instrucciones pertinentes y añadir después los resultados nuevos. En Claude, la documentación requiere coincidencia exacta de los segmentos elegibles y aclara que el prompt caching no elimina la generación de tokens de salida. Las condiciones concretas deben comprobarse para el servicio utilizado. [Documentación oficial](https://platform.claude.com/docs/en/build-with-claude/prompt-caching).

Cambiar un segmento puede invalidar la reutilización desde ese punto sin borrar necesariamente un prefijo anterior todavía idéntico. La vigencia, ubicación de puntos de caché y límites de la implementación importan. No garantizaremos que toda la trayectoria quede disponible indefinidamente ni que leer de caché sea gratuito.

El artículo de Manus publicado el 18 de julio de 2025 presenta estabilidad del prefijo y serialización consistente como prácticas aprendidas en su sistema. Conservamos esas ideas como experiencia de ingeniería, no como obligación de maximizar la caché a costa de conservar contenido innecesario. [Fuente primaria](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus).

La caché no amplía por sí sola la ventana de contexto ni hace correcto el contenido repetido. Para nuestro agente mediremos su efecto cuando evaluemos una configuración concreta, sin tratar una tasa alta de aciertos como sustituto del éxito de la tarea.

## Compactación y recuperación del estado

La **compactación del contexto (context compaction)** reduce o reorganiza información para continuar cuando la conversación crece. Puede resumir decisiones y resultados, pero también perder detalles. El entorno y los archivos no se restauran por resumir la conversación.

Para el trabajo de código conservaremos objetivo, restricciones, versión del repositorio, archivos tocados, resultados comprobados, bloqueos y siguiente paso. Diferenciaremos «prueba solicitada» de «prueba terminada» y los errores observados de las hipótesis sobre su causa.

Podemos mantener secciones con distinta frecuencia de cambio. La compactación de una parte puede conservar un prefijo estable o requerir reconstruirlo, según el sistema. No prometemos una tasa de reutilización sin comprobar la serialización y el servicio.

Las experiencias de Anthropic con agentes de larga duración, publicadas el 26 de noviembre de 2025, utilizan progreso incremental y artefactos de continuidad entre sesiones. Son soluciones de un entorno de desarrollo específico. [Fuente primaria](https://www.anthropic.com/research/effective-harnesses-for-long-running-agents).

Al retomar una tarea comprobaremos el estado actual. Si otra persona cambió la consulta del filtro, el resumen anterior sigue siendo un registro histórico; no autoriza sobrescribir el archivo con la versión recordada.

# Trabajar sobre una aplicación existente

Un **agente de ingeniería de software (software engineering agent)** debe coordinar lectura, reproducción, modificación y verificación en un proyecto real. Generar una función aislada no cubre dependencias, preparación del entorno o compatibilidad con consumidores existentes.

## Herramientas e interfaces para el agente

La **infraestructura del agente (agent harness)** organiza el bucle de trabajo, las herramientas, el contexto y los mecanismos de continuidad. El ejecutor es la parte que realiza las operaciones autorizadas. Cambiar el modelo o cambiar esa infraestructura son intervenciones diferentes.

**SWE-agent**, publicado el 6 de mayo de 2024, estudia cómo una **interfaz agente-computadora (agent-computer interface, ACI)** diseñada para la tarea cambia la interacción con el repositorio. No es solo una ayuda para modelos antiguos: la interfaz y su información disponible forman parte del sistema evaluado. [Publicación original](https://arxiv.org/abs/2405.15793).

Las herramientas pueden reducir errores con rangos de lectura claros, mensajes de conflicto y resultados estructurados. Un modelo más capaz puede aprovecharlas de otra manera, pero no demuestra que ya no importen el entorno o el ejecutor.

Para nuestra primera versión preferiremos una interfaz pequeña que podamos observar: lectura y búsqueda, propuesta de parche y ejecución de verificaciones autorizadas. Ampliaremos sus capacidades cuando una tarea concreta lo requiera y podamos evaluar sus efectos.

## Del problema reportado a un cambio verificable

**SWE-bench**, publicado inicialmente el 10 de octubre de 2023, propone resolver incidencias sobre estados definidos de repositorios y evaluar los parches con pruebas. Hay variantes con tareas y criterios diferentes; no describiremos toda la familia como una colección única de cientos de ejemplos. [Publicación original](https://arxiv.org/abs/2310.06770).

En nuestra aplicación utilizaremos el problema reportado y el contrato como referencia, sin confundirlos con una instrucción de copiar el parche de otra persona. La reproducción debe mostrar qué comportamiento falta o falla.

Para el filtro, el recorrido propuesto es:

1. Acordar el intervalo, zona horaria y resultados esperados.
2. Localizar la consulta, sus consumidores y pruebas existentes.
3. Añadir o ajustar una prueba que muestre el comportamiento pendiente.
4. Implementar el cambio mínimo dentro del alcance autorizado.
5. Ejecutar la prueba pertinente y las regresiones necesarias; comprobar la integración.
6. Revisar el diff y entregar evidencia, límites y estado del trabajo.

Este orden refleja nuestra decisión de trabajar con pruebas antes del cambio. No es una ejecución ya realizada. Si el entorno impide reproducir el caso, registraremos esa limitación en lugar de declarar una prueba fallida o aprobada que nunca terminó.

La ausencia de errores en un comando no demuestra el cumplimiento del requisito. Necesitamos comprobar que se ejecutaron las pruebas esperadas sobre la versión modificada y que sus aserciones representan el comportamiento relevante.

## Planificación y revisión del alcance

El plan puede comenzar con investigación y cambiar cuando aparezca evidencia. No necesitamos detallar toda la implementación antes de leer el repositorio ni usar siempre el modelo más costoso para planificar.

Separaremos decisiones de implementación de decisiones de producto. Elegir cómo representar un intervalo bajo un contrato acordado puede ser una decisión técnica; cambiar qué significa «día» para la persona puede requerir una definición de alcance.

En una tarea larga mantendremos objetivos verificables y un estado explícito por paso. Si una prueba falla después del parche, ese resultado orienta la investigación; no confirma automáticamente una causa concreta ni justifica un refactor ajeno a la tarea.

La experiencia de diseño de harnesses publicada por Anthropic el 24 de marzo de 2026 explora roles de planificación, generación y evaluación. Es una opción de arquitectura con resultados en sus condiciones, no una razón para repartir cualquier cambio entre varios agentes. [Fuente primaria](https://www.anthropic.com/research/harness-design-long-running-apps).

## Aislamiento del trabajo y entrega

Una copia de trabajo separada puede evitar que los cambios de una tarea se mezclen con otros. **Git worktree** permite varias copias de trabajo vinculadas a un repositorio. No equivale a una frontera de seguridad: puede compartir configuración, historial y otros recursos con el resto del sistema. [Documentación oficial](https://git-scm.com/docs/git-worktree).

Para nuestro agente registraremos rama, revisión de partida y cambios existentes antes de editar. Conservaremos archivos ajenos y evitaremos operaciones que borren trabajo para obtener un estado aparentemente limpio.

Si dos actores necesitan modificar el mismo archivo, separaremos sus cambios o estableceremos una forma de integración. Crear dos procesos sin esa coordinación no resuelve el conflicto. Un parche válido sobre una revisión puede dejar de serlo sobre otra.

La entrega incluirá qué cambió, por qué, cómo se verificó y qué queda pendiente. Confirmar cambios en Git, publicarlos, abrir una solicitud de revisión y fusionarlos son acciones distintas, con la autorización definida por el proyecto y la persona responsable.

# Flujos de trabajo específicos y selección de candidatos

Un **flujo de trabajo (workflow)** fija parte de la secuencia en código. Puede ser apropiado para una tarea acotada que no necesita decidir libremente cada paso. Utilizar un LLM dentro de ese flujo no obliga a convertirlo en un agente con acceso general al entorno.

## Agentless: localizar, reparar y validar

**Agentless**, publicado el 1 de julio de 2024, organiza reparación de software en fases que sus autores llaman «localization, repair, and patch validation»: localización, reparación y validación de parches. El sistema conserva control explícito de las fases en vez de dejar al modelo decidir libremente todas las acciones. Es un resultado de investigación sobre tareas y versiones concretas. [Publicación original](https://arxiv.org/abs/2407.01489).

La localización reduce el espacio de código que se entrega para reparar. Después se generan candidatos y se utilizan pruebas de regresión y de reproducción para seleccionarlos. La receta tiene decisiones y mecanismos de respaldo específicos; no se reduce a «elegir siempre el parche que pase más pruebas inventadas».

Para nuestro producto, las pruebas generadas pueden aportar señales, pero no reemplazan las restricciones obligatorias ni una evaluación independiente. Una candidatura que parece mejor entre opciones defectuosas puede seguir sin cumplir el objetivo.

Conservamos la distinción entre usar una prueba para ordenar candidatos y usarla como condición de aceptación. La primera ayuda a buscar; la segunda define qué podemos entregar bajo el contrato acordado.

## CodeMonkeys y cómputo durante la resolución

**CodeMonkeys**, publicado el 24 de enero de 2025, estudia generación de varios candidatos mediante trayectorias que editan y ejecutan pruebas. Combina más iteraciones dentro de una trayectoria con varias trayectorias por problema, y amortiza preparación de contexto entre esos intentos. [Publicación original](https://arxiv.org/abs/2501.14723).

Eso explica por qué en su configuración puede ser razonable invertir trabajo previo en localizar archivos. No demuestra que cualquier tarea deba enviar todo el repositorio a un LLM ni que generar más candidatos sea siempre rentable.

En nuestro agente limitaremos intentos y costo, mantendremos separadas las versiones candidatas y evaluaremos también la estrategia de selección. El éxito de algún candidato no equivale al éxito del sistema que debe elegir uno sin conocer la solución correcta.

# Contrato de ejecución del agente

La primera integración necesita un **ejecutor (executor)** que haga cumplir permisos y traduzca solicitudes en operaciones observables. No basta con asociar nombres a funciones y confiar en que el modelo evitará efectos ajenos al objetivo.

## Estados y resultados de herramientas

El registro debe permitir reconstruir qué ocurrió. Para cada solicitud conservaremos un identificador, operación y argumentos pertinentes, junto con el estado de ejecución y el resultado asociado.

| Evento | Qué registra | Qué no demuestra por sí solo |
|---|---|---|
| Solicitud | Operación propuesta por el modelo | Que haya empezado a ejecutarse |
| Validación | Argumentos, alcance y permisos comprobados | Que la operación vaya a terminar correctamente |
| Inicio | Proceso o trabajo creado | Que haya terminado |
| Resultado | Datos, salida, errores y estado de finalización | Que se haya cumplido todo el objetivo |
| Verificación | Comprobación del efecto o artefacto esperado | Que no existan fallos fuera de lo comprobado |

Para comandos guardaremos salida estándar, salida de error y código de finalización cuando estén disponibles. Si truncamos la salida para contexto, conservaremos ese hecho y una referencia al registro necesario. «No aparece el error en el fragmento» no significa «no hubo error».

Un error de validación se resolverá revisando la solicitud. Un servicio temporalmente indisponible puede justificar un reintento acotado. Una operación cuyo efecto se desconoce requiere comprobar el estado antes de repetirse si puede duplicar cambios.

Cuando se agote el presupuesto o se cancele el trabajo, registraremos lo que sigue activo. Interrumpir la respuesta del modelo no garantiza que todos los procesos externos se hayan detenido.

## Interfaz y seguimiento del progreso

La interfaz puede mostrar objetivo, paso actual, solicitudes, resultados y cambios preparados. Una explicación breve de por qué se propone una acción puede ayudar a revisar el trabajo, pero no se presentará como acceso al pensamiento interno del modelo.

Distinguir estados es más importante que asignar colores. Una solicitud de ejecución puede aparecer como pendiente, activa, terminada o fallida; el resultado debe provenir del ejecutor. No mostraremos una respuesta de ejemplo como si fuera una trayectoria capturada de un agente real.

Para una entrega con archivos comprobaremos que la referencia es accesible y corresponde a la versión generada. Un mensaje de «archivo guardado» sin comprobar el archivo puede ocultar un fallo o una ruta equivocada.

Estas son decisiones de contrato para una futura implementación. Aquí no creamos una CLI ni declaramos construido un agente operativo a partir de fragmentos de código incompletos.

## Primera unidad de construcción

Empezaremos con una tarea de lectura y cálculo sobre el archivo sintético: identificar B, C y D y sumar doce unidades. Podremos comprobar la función ejecutora sin modelo y después evaluar si el agente solicita correctamente la operación y usa su resultado.

En una segunda unidad incorporaremos la propuesta de parche para el filtro de una aplicación de prueba. La escritura permanecerá limitada al checkout autorizado y acompañada de verificación. Los permisos de publicación o acceso a servicios no se heredan de ese paso.

Cada unidad debe dejar un resultado completo que podamos revisar. No necesitamos añadir un sistema de múltiples agentes, una interfaz avanzada o entrenamiento propio para validar el circuito básico.

# Modelos de lenguaje para código

Un **modelo de lenguaje para código (code language model)** aprende patrones de programas y su relación con instrucciones, documentación y otros datos. Para impulsar un agente también necesita generar solicitudes de herramientas y utilizar resultados en una tarea de varios pasos, según el entrenamiento y la integración.

La calidad del modelo importa, pero no determina por sí sola el resultado. El mismo modelo puede comportarse de manera distinta con otro contexto, herramientas, presupuesto o entorno. Un modelo que resuelve funciones aisladas no queda validado automáticamente para modificar repositorios.

## Etapas de entrenamiento

La **etapa de preentrenamiento (pretraining)** aprende regularidades a partir de los datos y objetivos definidos por quienes desarrollan el modelo. Puede incluir código, texto, documentación, matemáticas y otras fuentes. Una mayor proporción de código no garantiza mejor desempeño en toda tarea.

El **ajuste fino supervisado (supervised fine-tuning, SFT)** entrena con ejemplos de entradas y salidas o trayectorias. Puede enseñar formatos de llamadas y conductas como leer antes de editar o comprobar un cambio. No es «aprendizaje por refuerzo de secuencias» y no sucede automáticamente cuando mostramos instrucciones al modelo.

El **aprendizaje por refuerzo (reinforcement learning, RL)** ajusta una política utilizando señales de recompensa bajo un procedimiento de entrenamiento. Con **recompensas verificables (reinforcement learning with verifiable rewards, RLVR)**, parte de la señal puede obtenerse ejecutando pruebas u otros verificadores.

La **destilación (distillation)** utiliza señales o ejemplos de otro modelo o etapa para entrenar el sistema objetivo. Qué capacidades conserva depende de datos, objetivo y evaluación. Estas etapas pueden combinarse de maneras diferentes; no hay una receta única que todos los modelos de código sigan.

## Trayectorias y recompensas verificables

Una trayectoria de entrenamiento puede incluir la tarea, contexto, solicitudes de herramientas, resultados y respuesta final. Si se recopila de una ejecución real, distinguiremos lo ejecutado de las explicaciones generadas. No convertiremos una simulación textual en evidencia de que sus herramientas funcionaron.

Para aprender de pruebas necesitamos que el verificador sea fiable y no pueda alterarse para obtener una aprobación artificial. Un programa que pasa casos incompletos puede seguir siendo incorrecto. Optimizar una recompensa no equivale a satisfacer todos los requisitos del producto.

**DeepSeek-R1**, publicado inicialmente el 22 de enero de 2025, estudia entrenamiento por refuerzo para razonamiento, incluidas tareas verificables. Es un antecedente para comprender el papel de esas señales, no una demostración de que toda capacidad para actuar sobre repositorios surja del mismo entrenamiento. [Publicación original](https://arxiv.org/abs/2501.12948).

En nuestro ejemplo, una prueba que solo compruebe que el resultado no está vacío permitiría múltiples errores. Un verificador más pertinente comprueba B, C y D, exclusión de A y E, ausencia de duplicados y suma de doce unidades. Aun así, no demostraría otros contratos que no incluimos.

## Recetas publicadas: Qwen3-Coder y GLM-5

La presentación de **Qwen3-Coder**, publicada el 22 de julio de 2025, describe preentrenamiento intensivo en código y entrenamiento orientado a interacción con entornos. Es una receta de sus autores para versiones concretas, no un requisito de escala para construir nuestra aplicación. [Publicación del equipo](https://qwenlm.github.io/blog/qwen3-coder/).

El informe **GLM-5**, publicado el 17 de febrero de 2026, distingue preentrenamiento, entrenamiento intermedio y etapas de ajuste, RL y destilación. Describe infraestructura asíncrona para separar generación de experiencias y entrenamiento. No significa que una conversación con ese modelo actualice sus pesos durante el uso ordinario. [Informe técnico](https://arxiv.org/abs/2602.15763).

Estos trabajos muestran la importancia de tareas de código, herramientas y experiencias de varios pasos dentro de algunas recetas actuales. Sus cifras de entrenamiento y resultados no establecen un tamaño mínimo ni permiten atribuir todo error de nuestro agente a falta de parámetros.

Para seleccionar un modelo evaluaremos la combinación real de formato de herramientas, tareas, recursos y permisos. La licencia, requisitos de ejecución y condiciones del servicio se comprobarán para la versión elegida, en vez de recomendar un modelo por una cifra o disponibilidad que puede cambiar.

# Datos sintéticos y mejora del sistema

Los **datos sintéticos (synthetic data)** se generan mediante procedimientos o modelos para ampliar ejemplos de entrenamiento. Pueden incluir problemas, consultas, código, pruebas y trayectorias. Que los datos sean generados no demuestra que sean correctos, diversos o representativos.

## Generar, filtrar y evaluar

El uso de modelos para producir instrucciones es anterior a 2024. **Self-Instruct**, publicado el 20 de diciembre de 2022, genera instrucciones, entradas y salidas y filtra ejemplos inválidos o similares antes del ajuste. Conservamos ese antecedente sin presentarlo como una práctica nacida dos años después. [Publicación original](https://arxiv.org/abs/2212.10560).

Para código, un proceso puede generar una especificación, proponer soluciones y ejecutar verificadores antes de incorporar ejemplos. Debemos evitar que solución y test compartan el mismo malentendido y se validen entre sí. La revisión independiente, la diversidad de casos y la separación de conjuntos siguen siendo necesarias.

**OmniSQL**, publicado el 4 de marzo de 2025, presenta generación a escala de datos de texto a SQL y modelos entrenados con ellos. Es un ejemplo de síntesis estructurada con recursos publicados, no evidencia de que cualquier consulta sintética refleje los significados de nuestra organización. [Publicación original](https://arxiv.org/abs/2503.02240).

Versionaremos fuentes, transformaciones y criterios de filtrado si construimos un conjunto propio. Revisaremos permisos de uso y evitaremos incorporar secretos, respuestas reservadas de evaluación o datos cuya procedencia no podamos justificar.

## Aprender de errores sin confundir memoria y entrenamiento

Un fallo reproducible del filtro puede convertirse en una prueba de regresión. Una explicación útil puede guardarse con su evidencia como información del proyecto. Ninguna de esas operaciones cambia por sí sola los parámetros del modelo.

Si más adelante decidimos entrenar, separaremos ese proceso de la operación cotidiana: selección de datos, entrenamiento, evaluación y adopción de una versión. No promoveremos automáticamente una salida generada a regla permanente ni asumiremos una mejora ilimitada por repetir el ciclo.

También puede haber mejoras en la aplicación sin entrenamiento: una herramienta con mejor contrato, una búsqueda adecuada, un verificador corregido o un permiso más acotado. Diagnosticar el primer punto donde falla el sistema nos ayuda a elegir qué cambiar.

# Verificar utilidad y condiciones de entrega

La evaluación debe incluir lo que necesita la persona, además de tareas públicas. En nuestro caso comprobaremos datos, interfaz, permisos y conservación de cambios ajenos. Una métrica única de generación de código no cubre esas dimensiones.

**SWE Atlas**, publicado el 8 de mayo de 2026, amplía la evaluación a preguntas sobre código, escritura de pruebas y refactorización. Es una propuesta de benchmark que permite separar habilidades, sin afirmar que sus tareas representen todo el trabajo de un equipo. [Publicación original](https://arxiv.org/abs/2605.08366).

Los recursos del entorno forman parte del experimento. La experiencia de Anthropic publicada el 5 de febrero de 2026 muestra que límites e infraestructura pueden cambiar resultados de evaluaciones de código. No trasladaremos sus magnitudes a nuestra configuración sin medirla. [Fuente primaria](https://www.anthropic.com/research/infrastructure-noise).

## Casos para nuestra primera versión

La siguiente tabla es una **propuesta propia de evaluación**, no resultados de un agente ejecutado.

| Caso | Evidencia esperada |
|---|---|
| Lectura y cálculo de datos conocidos | Selección B/C/D, tres registros y doce unidades |
| Límite de fecha | Incluye inicio del día y excluye inicio del siguiente |
| Archivo modificado desde la lectura | Detecta conflicto y no sobrescribe cambios ajenos |
| Solicitud fuera del directorio permitido | El ejecutor impide el acceso y registra la denegación |
| Prueba que falla antes del parche | El fallo corresponde al comportamiento que se quiere corregir |
| Cambio aplicado | Pasan los casos pertinentes y se comprueba la integración |
| Ejecución pendiente o cancelada | Informa qué terminó y qué sigue activo o sin comprobar |
| Contenido con instrucciones ajenas | No amplía permisos ni altera criterios de aprobación |
| Reanudación tras compactar | Recupera el objetivo y verifica el estado actual antes de continuar |

Las pruebas generadas por el agente se revisarán como código. Mantendremos verificaciones que no pueda cambiar para mejorar artificialmente la puntuación. Si una herramienta falta o el entorno falla, registraremos esa limitación por separado del resultado funcional.

Antes de adoptar un cambio compararemos calidad, recursos y tareas con una línea base. Al aumentar intentos también aumenta el trabajo de ejecución y selección; no presentaremos el mejor candidato conocido por el evaluador como desempeño del sistema desplegado.

## Productividad y trabajo humano

El tiempo hasta obtener un borrador no es todo el tiempo hasta integrar un cambio. Para nuestro agente tendremos en cuenta revisión, correcciones, pruebas y tareas que la persona debe repetir, manteniendo un criterio de calidad comparable.

METR publicó el 10 de julio de 2025 un estudio controlado sobre desarrolladores experimentados y herramientas de principios de 2025. Sus resultados no coincidieron con las expectativas de aceleración de los participantes. Es evidencia para esa población y configuración, no una estimación universal ni actual de toda herramienta de código. [Estudio y presentación de los autores](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/).

En su actualización del 24 de febrero de 2026, METR describe sesgos de selección y dificultades para medir el tiempo cuando se usan agentes en paralelo. No trataremos esos datos posteriores como una cifra definitiva de mejora. La lección útil es declarar condiciones y medir el trabajo completo, sin confundir percepción con impacto comprobado. [Actualización de los autores](https://metr.org/blog/2026-02-24-uplift-update/).

# Lo que queda definido

El agente de código que construiremos tendrá un objetivo acotado, herramientas con permisos, estado de ejecución observable y verificadores pertinentes. Comenzará por lectura y cálculo antes de incorporar cambios sobre una aplicación de prueba.

Las decisiones de alcance y publicación permanecerán en sus responsables. El modelo podrá proponer operaciones y explicaciones; el ejecutor aplicará los límites y registrará lo ocurrido. La finalización dependerá de la evidencia disponible, no de una frase de éxito.

Este capítulo define el contrato y los pasos de construcción; no entrega una implementación ejecutada. El trabajo posterior deberá probar cada unidad con herramientas reales y después evaluar el circuito completo.

# Resumen

Los agentes de código conectan modelos con archivos, intérpretes, comandos, bases de datos e interfaces. La frontera de ejecución, el contexto y la evaluación son partes del sistema, junto con el modelo que propone el trabajo.

La búsqueda, los mapas, la caché y la compactación ayudan a administrar información, con propósitos y límites diferentes. Los flujos fijos y los agentes con decisiones dinámicas ofrecen alternativas que debemos comparar según la tarea.

El entrenamiento con código, trayectorias y recompensas puede desarrollar capacidades del modelo. Los datos sintéticos amplían ejemplos, pero requieren filtrado, procedencia y validación. Guardar una lección o ejecutar un test no equivale a entrenar durante el uso.

Para construir nuestro agente avanzaremos desde un resultado pequeño y verificable hacia cambios de software revisables. Conservaremos la separación entre solicitud, ejecución, resultado y aceptación que atraviesa toda la guía.

# Referencias y alcance de las fuentes

Fecha de referencia: **3 de noviembre de 2026**. La ventana de novedades revisada abarca del **3 de noviembre de 2023 al 3 de noviembre de 2026**. Se mantienen fuentes anteriores que explican mecanismos todavía útiles. Las primeras publicaciones, revisiones y consultas se identifican por separado; el rastreo reciente no prueba publicación reciente.

Las propuestas de investigación describen resultados en sus tareas y condiciones. Los informes de modelos y experiencias de proveedores no establecen rendimiento universal. La documentación de herramientas respalda contratos consultados, no una garantía de estabilidad futura. Las tablas, decisiones de implementación y ejemplos de nuestro agente son aportaciones propias, sin reproducción de los experimentos citados.

## Herramientas, entorno y seguridad

- gVisor. [What is gVisor?](https://gvisor.dev/docs/). Documentación oficial consultada el 3 de noviembre de 2026; sin fecha editorial de primera publicación verificada. Fronteras de ejecución y núcleo de aplicación.
- Google. [Code execution](https://aistudio.google.com/docs/code-execution). Documentación oficial consultada el 3 de noviembre de 2026; sin fecha editorial de primera publicación verificada. Ejecución de Python como herramienta del servicio.
- ast-grep. [Pattern Syntax](https://ast-grep.github.io/guide/pattern-syntax.html). Documentación oficial consultada el 3 de noviembre de 2026; sin fecha editorial verificada. Patrones estructurales y metavariables.
- Li, Xiangyang et al. [CoIR: A Comprehensive Benchmark for Code Information Retrieval Models](https://arxiv.org/abs/2407.02883). Primera publicación: 3 de julio de 2024; revisión consultada: 6 de junio de 2025.
- Lei, Fangyu et al. [Spider 2.0: Evaluating Language Models on Real-World Enterprise Text-to-SQL Workflows](https://arxiv.org/abs/2411.07763). Primera publicación: 12 de noviembre de 2024; revisión consultada: 17 de marzo de 2025.
- PostgreSQL. [SET TRANSACTION](https://www.postgresql.org/docs/current/sql-set-transaction.html). Documentación oficial consultada el 3 de noviembre de 2026. Modos de transacción y límites documentados de READ ONLY; debe fijarse la versión del motor en una implementación.
- Guo, Chengquan et al. [RedCode: Risky Code Execution and Generation Benchmark for Code Agents](https://arxiv.org/abs/2411.07781). Primera publicación: 12 de noviembre de 2024.
- Rabin, Rafiqul et al. [SandboxEval: Towards Securing Test Environment for Untrusted Code](https://arxiv.org/abs/2504.00018). Primera publicación: 27 de marzo de 2025, aunque el identificador arXiv comience por 2504. Versión preliminar de investigación.
- Wang, Zijun et al. [Your Agent, Their Asset: A Real-World Safety Analysis of OpenClaw](https://arxiv.org/abs/2604.04759). Primera publicación: 6 de abril de 2026. Estado persistente y ataques en la plataforma estudiada.

## Contexto, interfaces y flujos de trabajo

- AGENTS.md. [Formato abierto y ejemplos](https://agents.md/). Consultado el 3 de noviembre de 2026; sin fecha editorial de primera publicación verificada. Instrucciones para agentes en repositorios.
- Aider. [Repository map](https://aider.chat/docs/repomap.html). Documentación de los autores consultada el 3 de noviembre de 2026. Símbolos, relaciones y selección según presupuesto.
- vLLM. [Automatic Prefix Caching](https://docs.vllm.ai/en/latest/features/automatic_prefix_caching/). Documentación oficial consultada el 3 de noviembre de 2026; sin fecha editorial verificada. Reutilización de prefijos.
- Claude Platform. [Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching). Documentación oficial consultada el 3 de noviembre de 2026; sin fecha editorial de primera publicación verificada. Condiciones de coincidencia y procesamiento.
- Ji, Yichao «Peak». [Context Engineering for AI Agents: Lessons from Building Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus). Publicado el 18 de julio de 2025 según el artículo. Experiencia de ingeniería de Manus.
- Anthropic. [Effective harnesses for long-running agents](https://www.anthropic.com/research/effective-harnesses-for-long-running-agents). Publicado el 26 de noviembre de 2025. Progreso incremental y continuidad entre sesiones.
- Yang, John et al. [SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering](https://arxiv.org/abs/2405.15793). Primera publicación: 6 de mayo de 2024; revisión consultada: 11 de noviembre de 2024.
- Jimenez, Carlos E. et al. [SWE-bench: Can Language Models Resolve Real-World GitHub Issues?](https://arxiv.org/abs/2310.06770). Primera publicación: 10 de octubre de 2023; revisión de noviembre de 2024. Fundamento anterior a la ventana por su origen.
- Anthropic. [Harness design for long-running application development](https://www.anthropic.com/research/harness-design-long-running-apps). Publicado el 24 de marzo de 2026. Experiencia con roles de planificación, generación y evaluación.
- Git. [git-worktree](https://git-scm.com/docs/git-worktree). Documentación oficial consultada el 3 de noviembre de 2026; sin fecha de primera publicación del mecanismo verificada aquí. Copias de trabajo vinculadas.
- Xia, Chunqiu Steven et al. [Agentless: Demystifying LLM-based Software Engineering Agents](https://arxiv.org/abs/2407.01489). Primera publicación: 1 de julio de 2024; versión consultada revisada el 29 de octubre de 2024. Localización, reparación y validación de parches.
- Ehrlich, Ryan et al. [CodeMonkeys: Scaling Test-Time Compute for Software Engineering](https://arxiv.org/abs/2501.14723). Primera publicación: 24 de enero de 2025; versión consultada revisada el 3 de febrero de 2025. Generación y selección de candidatos.

## Entrenamiento, datos e impacto

- DeepSeek-AI et al. [DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning](https://arxiv.org/abs/2501.12948). Primera publicación: 22 de enero de 2025; versión consultada revisada el 4 de enero de 2026. RL para razonamiento, sin equiparar esa evaluación con el trabajo de un agente completo.
- Qwen Team. [Qwen3-Coder: Agentic Coding in the World](https://qwenlm.github.io/blog/qwen3-coder/). Publicado el 22 de julio de 2025. Descripción de entrenamiento y capacidades por sus autores.
- GLM-5-Team et al. [GLM-5: from Vibe Coding to Agentic Engineering](https://arxiv.org/abs/2602.15763). Primera publicación: 17 de febrero de 2026; versión consultada revisada el 24 de febrero de 2026. Etapas de entrenamiento e infraestructura de RL.
- Wang, Yizhong et al. [Self-Instruct: Aligning Language Models with Self-Generated Instructions](https://arxiv.org/abs/2212.10560). Primera publicación: 20 de diciembre de 2022. Antecedente de datos sintéticos anterior a la ventana.
- Li, Haoyang et al. [OmniSQL: Synthesizing High-quality Text-to-SQL Data at Scale](https://arxiv.org/abs/2503.02240). Primera publicación: 4 de marzo de 2025; versión consultada revisada el 13 de julio de 2025.
- Raghavendra, Mohit et al. [SWE Atlas: Benchmarking Coding Agents Beyond Issue Resolution](https://arxiv.org/abs/2605.08366). Primera publicación: 8 de mayo de 2026. Evaluación de preguntas sobre código, tests y refactorización.
- Anthropic. [Quantifying infrastructure noise in agentic coding evals](https://www.anthropic.com/research/infrastructure-noise). Publicado el 5 de febrero de 2026. Experimentos sobre condiciones de ejecución.
- METR. [Measuring the Impact of Early-2025 AI on Experienced Open-Source Developer Productivity](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/). Presentación publicada el 10 de julio de 2025. Se utiliza como estudio situado, sin extrapolar su resultado al estado actual de todos los agentes.
- METR. [We are Changing our Developer Productivity Experiment Design](https://metr.org/blog/2026-02-24-uplift-update/). Publicado el 24 de febrero de 2026. Actualización sobre datos posteriores y límites de medición.
