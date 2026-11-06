# Capítulo 9. Comprensión multimodal

La **comprensión multimodal (multimodal understanding)** permite que nuestro agente utilice información que no llega solamente como texto: una página escaneada, una captura de pantalla, una grabación o un vídeo. Sirve cuando convertir todo a texto por adelantado perdería detalles necesarios para resolver la tarea.

Hasta aquí distinguimos memoria, herramientas, planificación y evaluación. La multimodalidad amplía lo que podemos observar; no sustituye esas responsabilidades. Interpretar una captura no demuestra que se haya pulsado un botón, y transcribir una orden no significa que esté autorizada.

Continuaremos con un **ejemplo ilustrativo propio**: una persona quiere consultar registros del 3 de noviembre de 2026. Algunos datos están en una tabla de un documento escaneado y la fecha aparece como `03/11/2026`, bajo un contrato que define el formato día/mes/año. El agente necesita identificar la fila, conservar su procedencia y distinguir lo leído de cualquier cálculo posterior.

No se ha generado ni analizado una imagen real para este ejemplo. Los resultados que describiremos son expectativas didácticas, no mediciones de un modelo.

En este capítulo estudiaremos cómo se representan las modalidades, cómo se conectan con un modelo de lenguaje y qué contrato necesitamos para incorporarlas al agente. Nos centraremos en **comprender entradas**. Generar imágenes, voz o vídeo es otra capacidad, que no viene garantizada por aceptar esas entradas.

# Qué es un modelo de lenguaje multimodal

Un **modelo de lenguaje multimodal (multimodal large language model, MLLM)** combina lenguaje con una o más modalidades adicionales. Un **modelo de visión y lenguaje (vision-language model, VLM)** trabaja específicamente con información visual y textual. Los nombres de las familias no bastan para determinar qué acepta cada versión.

Para elegir un modelo comprobaremos por separado entradas admitidas, salidas disponibles, combinaciones permitidas y contrato del servicio. Un modelo que responde preguntas sobre imágenes puede no aceptar audio; otro que interpreta audio puede producir solo texto. La interfaz de un proveedor puede exponer solo parte de las capacidades descritas en un informe técnico.

Una arquitectura modular frecuente contiene un **codificador (encoder)** que extrae características de la entrada, un **conector (connector)** que permite utilizarlas y un modelo de lenguaje que produce la respuesta. Para generar otras modalidades puede haber módulos de salida adicionales. Este esquema ayuda a estudiar varios modelos, pero no es una definición obligatoria de toda arquitectura multimodal.

**Chameleon**, publicado inicialmente el 16 de mayo de 2024, estudia **fusión temprana (early fusion)** mediante secuencias que combinan tokens de imágenes y texto. Es una alternativa de investigación al esquema de añadir un conector a un modelo de lenguaje ya entrenado. No debemos confundir esa fusión temprana con las capas de atención cruzada que estudiaremos después. [Publicación original](https://arxiv.org/abs/2405.09818).

Para construir nuestra aplicación no necesitamos entrenar estos componentes desde cero. Empezaremos usando una combinación documentada de modelo, procesador y servicio de inferencia, y evaluaremos lo que efectivamente puede observar en nuestras tareas.

# Codificación de modalidades

La **codificación (encoding)** convierte contenido en representaciones que el sistema puede procesar. Conviene separar tres niveles:

- El archivo o señal original: píxeles, muestras de audio, texto o vídeo.
- La preparación de entrada: tokenización, redimensionado, recortes, espectrogramas o selección de fotogramas, según el modelo.
- Las representaciones aprendidas: vectores y secuencias que se utilizan durante la inferencia.

Una **representación vectorial (embedding)** no es una descripción textual del contenido ni una observación directamente legible por una persona. Sus dimensiones dependen del modelo. Tener dos vectores del mismo tamaño no significa que sus coordenadas representen lo mismo o que podamos comparar sus valores con sentido.

La aplicación suele enviar archivos o referencias siguiendo una API; el procesador y el modelo realizan parte de estas transformaciones. No confundiremos la codificación de transporte, como base64, con aprender una representación visual o acústica.

## Texto

La tokenización divide texto en unidades del vocabulario del modelo. Una capa de embeddings transforma sus identificadores en vectores, y las capas posteriores los contextualizan. Una palabra puede ocupar varios tokens; no hay correspondencia universal entre palabras y unidades del modelo.

Podemos considerar tokenización y embeddings como el comienzo de la representación textual. Eso no los convierte en equivalentes a un codificador de recuperación que produce un vector semántico de un documento completo.

Al incorporar otra modalidad, debemos resolver compatibilidad de representación, posiciones y entrenamiento. No basta con insertar cualquier matriz junto a los embeddings de texto. El modelo necesita haber aprendido cómo utilizar esa información bajo el formato previsto.

En nuestro ejemplo, escribir «imagen de la tabla» sin adjuntar su contenido o una referencia accesible solo proporciona una frase. No demuestra que el modelo haya recibido los píxeles de la página.

## Imágenes

Las imágenes aportan texto visible, distribución espacial, objetos, gráficos y otros detalles. Qué información logra utilizar el modelo depende también de resolución, preparación y entrenamiento. Una imagen grande en el archivo original puede llegar reducida al componente de percepción.

### Transformador visual: ViT

El **transformador visual (Vision Transformer, ViT)** permite procesar una imagen como una secuencia de parches. El trabajo original, publicado el 22 de octubre de 2020, divide la imagen, proyecta los parches a vectores y añade información posicional antes del procesamiento mediante Transformer. Un parche no es una palabra; la comparación sirve para entender la construcción de una secuencia. [Publicación original](https://arxiv.org/abs/2010.11929).

Como cálculo ilustrativo, una imagen de 224 × 224 píxeles dividida en parches de 16 × 16 produce 14 × 14 = **196 parches**. La secuencia real puede incorporar otros elementos y depende de la arquitectura. Este cálculo no establece cuántos tokens cobra una API ni cómo procesa todas sus imágenes.

Si el redimensionado borra los dígitos pequeños de nuestra tabla, el modelo no puede recuperar de forma fiable esos detalles solo porque el resto de la página resulte reconocible. Decidiremos resolución y recortes según la tarea, conservando una referencia a la página completa.

### Alineación entre imágenes y texto: CLIP

El **preentrenamiento contrastivo de lenguaje e imágenes (Contrastive Language–Image Pre-training, CLIP)** aprende representaciones comparables para imágenes y sus descripciones. Su publicación del 26 de febrero de 2021 utiliza dos codificadores y una función de entrenamiento que favorece las correspondencias de cada lote frente a otras combinaciones. El trabajo estudia codificadores visuales convolucionales y variantes ViT; CLIP no es sinónimo de un único ViT. [Publicación original](https://arxiv.org/abs/2103.00020).

La **similitud (similarity)** permite ordenar candidatos para búsqueda o clasificación según textos propuestos. Una similitud alta no equivale a una probabilidad calibrada de que una afirmación sea verdadera. Tampoco demuestra lectura exacta de cada celda o relaciones espaciales complejas.

Para nuestro agente, un codificador así puede ayudar a recuperar una página candidata. Después debemos inspeccionar la evidencia pertinente. Encontrar una página parecida a «tabla de operaciones» no prueba que contenga la fila del día solicitado.

### Resolución dinámica y detalle visual

Adaptar la resolución permite asignar representación a imágenes con distintos tamaños y proporciones. Puede mejorar la lectura de detalles, pero también incrementa la longitud de la entrada o el costo de procesamiento según el sistema.

El informe **Qwen2.5-VL**, publicado el 19 de febrero de 2025, describe resolución dinámica y agrupación de características visuales antes de proyectarlas al modelo de lenguaje. Es un diseño concreto que combina detalle y compresión, no una regla universal sobre cuántos tokens necesita cada imagen. [Informe técnico](https://arxiv.org/abs/2502.13923).

**SigLIP 2**, publicado el 20 de febrero de 2025, presenta codificadores multilingües y variantes que admiten resoluciones diferentes conservando la proporción original. Es evidencia de que la preparación visual y el entrenamiento siguen evolucionando; no hace obsoletos por fecha los fundamentos de CLIP. [Publicación original](https://arxiv.org/abs/2502.14786).

Para nuestra tabla empezaremos con una página legible y, si hace falta, un recorte de la región pertinente. Registraremos la transformación aplicada. Evaluaremos si el recorte conserva encabezados y unidades, porque aislar una cifra puede eliminar el contexto necesario para interpretarla.

## Audio

El audio puede contener palabras, pausas, música, alarmas o sonidos del entorno. Transcribir habla, reconocer un evento acústico y localizar quién habló son tareas diferentes. No asumiremos que un modelo entrenado para una de ellas resuelve las demás.

La señal tiene una **frecuencia de muestreo (sample rate)** y uno o más canales. Remuestrear cambia la representación temporal de las muestras bajo un procedimiento definido; no equivale a cambiar arbitrariamente la velocidad. El procesador debe recibir el formato que espera.

### Representaciones del habla con wav2vec 2.0

**wav2vec 2.0**, publicado el 20 de junio de 2020, aprende representaciones de habla mediante preentrenamiento autosupervisado. Un extractor convolucional convierte la onda en características latentes, parte de esa secuencia se enmascara y un Transformer produce representaciones contextualizadas. El objetivo contrastivo distingue la representación cuantizada correspondiente frente a alternativas. [Publicación original](https://arxiv.org/abs/2006.11477).

La **cuantización (quantization)** proporciona objetivos discretos para ese entrenamiento; no significa que toda la onda se convierta en palabras ni que las representaciones utilizadas por el modelo sean una transcripción.

El **ajuste fino (fine-tuning)** posterior con datos transcritos permite adaptar el modelo al reconocimiento del habla. Es importante separar el aprendizaje de características de la tarea final: disponer de embeddings acústicos no demuestra que nuestro agente reconozca correctamente una fecha dictada.

### HuBERT y predicción de unidades ocultas

**HuBERT**, publicado el 14 de junio de 2021, utiliza agrupamiento para producir etiquetas objetivo y aprende a predecirlas en regiones enmascaradas. Su resumen especifica “an offline clustering step”. Es un objetivo de predicción de unidades ocultas, no simplemente el mismo aprendizaje contrastivo de wav2vec 2.0 con otro cuantizador. [Publicación original](https://arxiv.org/abs/2106.07447).

Las etiquetas de los grupos no son transcripciones escritas por una persona. Este antecedente permite entender cómo se aprende de audio sin etiquetar cada fragmento manualmente. No es un componente que debamos entrenar para empezar a utilizar audio en nuestro agente.

### Whisper y reconocimiento del habla

**Whisper** permite realizar **reconocimiento automático del habla (automatic speech recognition, ASR)** y otras tareas como traducción del habla al inglés. El trabajo se publicó el 6 de diciembre de 2022, antes de su publicación en ICML 2023. Utiliza una arquitectura codificador-decodificador entrenada con supervisión débil a gran escala y entradas basadas en espectrogramas log-mel. [Publicación original](https://arxiv.org/abs/2212.04356).

Un **espectrograma (spectrogram)** representa contenido en frecuencias a lo largo del tiempo; no contiene únicamente el tono de una voz. El codificador produce características y el decodificador genera secuencias según la tarea. Algunas arquitecturas posteriores reutilizan el codificador; otras aplicaciones utilizan la transcripción completa como entrada textual de otro modelo.

La ficha de los autores advierte que pueden aparecer “texts that are not actually spoken in the audio input”. Una transcripción es una salida del modelo que debe evaluarse, especialmente con ruido, silencios, nombres o fechas ambiguas. [Ficha oficial y limitaciones](https://github.com/openai/whisper/blob/main/model-card.md).

En nuestro agente conservaremos el archivo o referencia y, cuando estén disponibles, los intervalos de la transcripción. Si el día dictado no se distingue, pedir una aclaración puede ser el comportamiento correcto. No trataremos una transcripción dudosa como una orden confirmada.

### CLAP y sonidos descritos con lenguaje

El **preentrenamiento contrastivo de lenguaje y audio (Contrastive Language–Audio Pre-training, CLAP)** permite comparar sonidos con descripciones. El trabajo de Wu y colaboradores se publicó inicialmente el 12 de noviembre de 2022 y estudia recuperación y clasificación, incluida fusión de características para longitudes variables. No se limita a reconocer palabras. [Publicación original](https://arxiv.org/abs/2211.06687).

La preparación puede combinar una vista global del espectrograma y fragmentos locales. Reducir esa representación global no debe explicarse como acelerar toda la grabación hasta que dure un número fijo de segundos.

Uno de los codificadores estudiados es **HTS-AT**, publicado el 2 de febrero de 2022. Utiliza una estructura jerárquica sobre representaciones del audio para clasificación y detección de sonidos. El uso de parches de espectrograma no significa que el sistema escuche una fotografía o que todo codificador acústico sea un codificador de imágenes sin cambios. [Publicación original](https://arxiv.org/abs/2202.00874).

Si nuestra tarea consiste en buscar fragmentos con una alarma, la comparación audio-texto puede ser pertinente. Si necesitamos transcribir una fecha exacta, evaluaremos ASR o un modelo de audio adecuado. Una representación útil para recuperación no garantiza transcripción literal.

## Vídeo

El vídeo incorpora secuencia temporal, y puede incluir audio y subtítulos. El orden, el instante y la duración de lo observado forman parte de la información. Una lista de fotogramas sin tiempos puede ser insuficiente para responder cuándo ocurrió un evento.

El **muestreo de fotogramas (frame sampling)** decide qué imágenes se procesan. Un muestreo espaciado puede omitir una acción breve. Promediar embeddings de fotogramas sin conservar posiciones hace que una misma colección en distinto orden tenga el mismo promedio; ese resumen no permite reconstruir por sí solo qué ocurrió primero.

### ViViT y representaciones espaciotemporales

**ViViT (Video Vision Transformer)**, publicado el 29 de marzo de 2021, estudia modelos Transformer para clasificación de vídeo. Construye tokens espaciotemporales y compara formas de factorizar el procesamiento espacial y temporal. [Publicación original](https://arxiv.org/abs/2103.15691).

Un **tubelet** agrupa una región espacial a lo largo de varios fotogramas consecutivos. A diferencia de un parche de una sola imagen, incorpora un pequeño bloque de tiempo. Las variantes de atención deciden cómo relacionar esas unidades; no hay una única arquitectura ViViT que debamos imponer a todos los vídeos.

El mecanismo explica cómo puede modelarse movimiento, pero el rendimiento en clasificación no demuestra localización exacta de cada evento. Para nuestro agente tendremos que evaluar la tarea temporal que realmente necesitamos.

### TimeSformer y atención dividida

**TimeSformer**, publicado el 9 de febrero de 2021, adapta la atención sobre parches de fotogramas. Su esquema de **atención espaciotemporal dividida (divided space-time attention)** procesa relaciones temporales y espaciales por separado dentro de los bloques. [Publicación original](https://arxiv.org/abs/2102.05095).

La distinción es útil: reconocer qué objetos aparecen y determinar cómo cambian a lo largo del tiempo son problemas relacionados, pero diferentes. Los resultados de sus experimentos no establecen que la atención dividida sea siempre la opción más precisa o económica para cualquier agente.

### Fotogramas ordenados y modelos de secuencia

Otra opción consiste en codificar fotogramas con un modelo visual y entregar una secuencia al modelo que integra la información. Debe conservarse el orden o el tiempo y existir entrenamiento adecuado para utilizarlos. **Procesar texto secuencial no demuestra automáticamente comprensión del tiempo en vídeo.**

El informe Qwen2.5-VL describe muestreo con tasas variables y posiciones alineadas con tiempo absoluto. Ese diseño muestra por qué el índice de un fotograma y su tiempo real no son intercambiables. [Informe técnico](https://arxiv.org/abs/2502.13923).

También existen alternativas a la atención Transformer. **VideoMamba**, publicado el 11 de marzo de 2024, adapta un **modelo de espacio de estados (state space model, SSM)** al vídeo. No es una prueba de que baste con añadir posiciones a embeddings CLIP y dejar que cualquier LLM resuelva el resto. [Publicación original](https://arxiv.org/abs/2403.06977).

Para nuestro agente registraremos los tiempos de los fotogramas seleccionados y el intervalo cubierto. Si una captura a los 4 segundos muestra un botón desactivado y otra a los 6 lo muestra activo, solo podemos situar el cambio entre ambas observaciones; no afirmar el segundo exacto sin más evidencia. Este es un ejemplo lógico, sin un vídeo ejecutado.

### Audio, subtítulos y sincronización

Para relacionar lo que se oye con lo que se ve necesitamos conservar sus referencias temporales. Un subtítulo puede resumir, traducir o estar desfasado; no prueba por sí solo el instante de una palabra en la señal de audio.

El informe **Qwen2.5-Omni**, publicado el 26 de marzo de 2025, describe procesamiento de audio y vídeo con posiciones temporales alineadas. **Qwen3-Omni**, publicado el 22 de septiembre de 2025, desarrolla una arquitectura que combina percepción y generación. Son informes técnicos de modelos concretos, no garantías de sincronización perfecta en nuestras grabaciones. [Qwen2.5-Omni](https://arxiv.org/abs/2503.20215), [Qwen3-Omni](https://arxiv.org/abs/2509.17765).

En una grabación de pantalla con narración, nuestro agente distinguirá el intervalo escuchado, el fotograma inspeccionado y la interpretación que relaciona ambos. Si faltan imágenes o audio en una parte, dejará ese límite explícito.

## Un espacio para varias modalidades: ImageBind

**ImageBind**, publicado el 9 de mayo de 2023, aprende un espacio compartido para imágenes, texto, audio, profundidad, datos térmicos y de una **unidad de medición inercial (inertial measurement unit, IMU)**. Utiliza pares vinculados mediante la modalidad visual sin exigir entrenamiento con todas las combinaciones posibles. [Publicación original](https://arxiv.org/abs/2305.05665).

La aportación es aprender relaciones que permiten, por ejemplo, recuperación entre modalidades. No significa que todas las observaciones compartan idéntica semántica ni que podamos convertir cualquier sensor en evidencia intercambiable. Tampoco incluye por definición tacto, estados de ánimo o cualquier información imaginable.

El espacio compartido resulta del entrenamiento, no solo de proyectar vectores a igual longitud. Para nuestro agente podría ser útil al buscar un clip a partir de una descripción; comprobar qué ocurrió seguiría requiriendo inspeccionar el contenido recuperado.

# Conectar modalidades con el modelo de lenguaje

Un **conector multimodal (multimodal connector)** permite que el modelo utilice representaciones de otra modalidad. Puede adaptar dimensiones, seleccionar o comprimir características e introducir información visual mediante atención. La compatibilidad numérica es necesaria en algunas interfaces, pero no sustituye aprender la relación entre contenido e instrucciones.

Estudiaremos proyección, consultas y atención cruzada como familias didácticas. Pueden combinarse: un sistema puede usar consultas para reducir la secuencia y después una proyección o atención cruzada. No son tres categorías mutuamente excluyentes ni cubren toda arquitectura posible.

## Conectores basados en proyección

Una **proyección (projection)** transforma vectores mediante una capa lineal o un **perceptrón multicapa (multilayer perceptron, MLP)**. Puede aumentar o reducir la dimensión de cada vector. Eso es diferente de reducir el número de posiciones en la secuencia.

**LLaVA**, presentado en Visual Instruction Tuning el 17 de abril de 2023, conecta un codificador visual con un modelo de lenguaje mediante proyección. En la alineación inicial entrena el conector con los componentes principales congelados; en el ajuste de instrucciones actualiza el conector y el modelo de lenguaje, manteniendo congelado el codificador visual. Son etapas de esa receta, no reglas para todos los modelos. [Publicación original](https://arxiv.org/abs/2304.08485).

El modelo de lenguaje del trabajo es Vicuna; no asumiremos que todas las versiones posteriores de LLaVA comparten su misma base. **LLaVA-1.5**, publicado inicialmente el 5 de octubre de 2023, estudia una receta con un proyector MLP. Su revisión de 2024 no cambia la fecha de origen del trabajo. [Publicación original de la mejora](https://arxiv.org/abs/2310.03744).

Una proyección sencilla puede ser suficiente dentro de un sistema bien entrenado. Tampoco implica que cada parche deba enviarse sin compresión: Qwen2.5-VL agrupa características adyacentes antes de proyectarlas. Separaremos el cambio de anchura del vector de la reducción de longitud de la secuencia. [Informe técnico](https://arxiv.org/abs/2502.13923).

### Proyección, comprensión y generación

Los conectores pueden formar parte de sistemas con salidas diferentes. El informe **Phi-4-Mini** de marzo de 2025 distingue Phi-4-Mini, un modelo de lenguaje, de **Phi-4-Multimodal**, que integra texto, visión y audio mediante componentes y adaptaciones específicos. El título del informe no hace multimodal a cada modelo que describe. [Informe técnico](https://arxiv.org/abs/2503.01743).

**Janus**, publicado el 17 de octubre de 2024, desacopla la codificación visual para comprensión y generación dentro de un marco unificado. Es una decisión de arquitectura de ese trabajo, no evidencia de que un modelo que interpreta una imagen pueda generarla mediante la misma representación. [Publicación original](https://arxiv.org/abs/2410.13848).

Para nuestra aplicación elegiremos primero la tarea de entrada. No necesitamos activar generación de imágenes para leer la tabla ni añadir un generador por el hecho de utilizar un modelo multimodal.

## Conectores basados en consultas

Los **vectores de consulta aprendibles (learnable queries)** permiten extraer una secuencia acotada de características a partir de una entrada visual más larga. Aquí «consulta» nombra parámetros del conector; no son preguntas escritas por una persona ni llamadas a herramientas.

**BLIP-2**, publicado el 30 de enero de 2023, utiliza un **Querying Transformer (Q-Former)** y mantiene congelados el codificador visual y el modelo de lenguaje. En su primera etapa combina aprendizaje contrastivo imagen-texto, generación de texto condicionada por imágenes y clasificación de correspondencia. En la segunda conecta las representaciones de consulta al modelo de lenguaje para aprendizaje generativo. [Publicación original](https://arxiv.org/abs/2301.12597).

Los autores utilizan 32 consultas en sus experimentos. Es un cuello de botella aprendido: limita la secuencia entregada después, pero no garantiza preservar cada detalle de la imagen ni que el costo total sea constante ante cualquier resolución. El procesamiento visual anterior y la atención sobre sus características también cuentan.

El primer **Qwen-VL**, publicado el 24 de agosto de 2023, utiliza un adaptador visual con consultas entrenables para obtener una secuencia de tamaño definido. Comparte esa idea general de compresión, sin que debamos identificarlo como una copia literal del Q-Former de BLIP-2. [Publicación original](https://arxiv.org/abs/2308.12966).

## Fusión mediante atención cruzada

La **atención cruzada (cross-attention)** permite que representaciones del lenguaje consulten características visuales mediante módulos del modelo. Es otra forma de condicionar la generación, que puede combinarse con compresión previa.

**Flamingo**, publicado el 29 de abril de 2022, utiliza un **Perceiver Resampler** para obtener una secuencia visual acotada y añade bloques de atención cruzada al modelo de lenguaje. Mantiene congelados el codificador visual y los bloques preentrenados del lenguaje, mientras entrena los módulos incorporados. Su configuración principal utiliza un codificador visual NFNet, no exige un ViT. [Publicación original](https://arxiv.org/abs/2204.14198).

Sus **compuertas (gates)** modulan la contribución de las capas nuevas. El trabajo utiliza tanh de un parámetro inicializado en cero; así esas contribuciones comienzan anuladas y se incorporan durante el entrenamiento. Este mecanismo ayuda a entender la estabilidad de esa receta, sin convertirla en garantía general.

Añadir módulos a la arquitectura no obliga a actualizar todos los parámetros del modelo base. Por eso no concluiremos que la fusión exige siempre un ajuste fino completo o que supera necesariamente a los demás conectores.

## Cómo comparar conectores

La siguiente tabla es una **síntesis didáctica propia**, basada en los mecanismos descritos. No es un benchmark ni una clasificación de rendimiento.

| Enfoque | Decisión que ayuda a entender | Límite que conviene comprobar |
|---|---|---|
| Proyección | Cómo adaptar las características a la entrada del modelo | Dimensión por vector y longitud de secuencia son costos distintos |
| Consultas aprendibles | Cómo limitar la secuencia de características entregada | La compresión puede perder detalles; el codificador aún procesa la entrada |
| Atención cruzada | Cómo introducir información visual en bloques del lenguaje | Cambia la arquitectura y depende de su frecuencia y configuración |

No elegiremos un sistema solo por el nombre del conector. Datos, codificador, resolución, objetivos de entrenamiento y recursos también afectan el resultado. Una comparación útil mantiene tareas y presupuesto definidos y observa la calidad alcanzada.

Al utilizar modelos ya entrenados, estas decisiones suelen venir incorporadas. Para nuestro primer agente la tarea será elegir una configuración compatible y medir sus límites, no reemplazar un conector sin volver a entrenar y validar.

# Incorporar entradas multimodales al agente

La capacidad del modelo necesita un contrato de aplicación. Debemos recibir contenido, prepararlo sin perder información indispensable, enviarlo en el formato admitido y mantener su relación con las respuestas y acciones.

## Del archivo al contenido del mensaje

Un mensaje puede contener varias partes: instrucciones textuales, imágenes, audio o referencias a archivos, según la API. Un archivo accesible para nuestra aplicación no es necesariamente accesible para el servicio que ejecuta el modelo.

La documentación de Gemini distingue datos de imagen incluidos en la solicitud y archivos referenciados mediante su servicio. Las opciones y los tipos admitidos deben verificarse en el contrato elegido. No construiremos manualmente supuestos tokens visuales universales ni daremos por hecho que cualquier URL privada es legible. [Documentación oficial de imágenes](https://aistudio.google.com/docs/image-understanding).

**Base64 es transporte, no comprensión.** Un texto que contiene bytes codificados solo se convierte en entrada visual si la integración los reconoce como tal y el procesador puede decodificarlos. Declarar un tipo de archivo tampoco garantiza que el contenido corresponda a él.

En nuestro agente separaremos referencia original, metadatos y parte del mensaje enviada al modelo. Validaremos el archivo y las condiciones de acceso antes de invocar la inferencia. No imprimiremos credenciales o URLs firmadas completas en registros generales.

## Procedencia, regiones y marcas de tiempo

La **procedencia (provenance)** permite localizar la evidencia utilizada. Para un documento conservaremos archivo, versión y página; para una región, el recorte y su relación con la página; para audio o vídeo, los intervalos y el muestreo aplicados.

El **anclaje espacial o temporal (grounding)** relaciona una respuesta con una región o intervalo concreto. Una caja delimitadora, una coordenada o una marca de tiempo generada es una predicción que debe comprobarse. No convierte por sí sola una interpretación en un hecho verificado.

En nuestro ejemplo, una respuesta podría indicar «página 3, fila R17». La salida estructurada esperada podría separar fecha leída, cantidad, valor unitario y total calculado. Si el texto visible fuese `03/11/2026`, `3` y `24`, el cálculo 3 × 24 = 72 sería una operación posterior. **Leer 72 en la página y calcular 72 son afirmaciones diferentes.**

Este ejemplo usa datos inventados para mostrar el contrato; no demuestra OCR ni lectura visual ejecutada. Si un campo no es legible, lo registraremos como ausente o incierto en vez de completar el valor por plausibilidad.

## Documentos y recuperación visual

El **reconocimiento óptico de caracteres (optical character recognition, OCR)** permite extraer texto de una imagen. Puede combinarse con análisis de estructura para conservar tablas y orden de lectura. La extracción textual es útil, pero puede perder relaciones entre columnas, gráficos o notas.

Un **PDF** es un contenedor, no una modalidad homogénea. Puede contener texto seleccionable, páginas escaneadas, imágenes o una mezcla. Elegiremos extracción, representación visual o una combinación según el contenido y la tarea. Convertir cada página a texto plano no siempre conserva lo necesario para responder.

**ColPali**, publicado el 27 de junio de 2024, estudia recuperación de documentos mediante representaciones multivectoriales de sus páginas como imágenes y comparación por interacción tardía. Esa **interacción tardía (late interaction)** corresponde al mecanismo de recuperación; no es la misma operación que la fusión temprana de Chameleon ni las capas de Flamingo. [Publicación original](https://arxiv.org/abs/2407.01449).

La **generación aumentada por recuperación multimodal (multimodal retrieval-augmented generation, multimodal RAG)** puede recuperar páginas o regiones visuales y aportarlas al modelo que responde. ColPali ayuda a estudiar la etapa de recuperación; una puntuación de recuperación alta no demuestra que la respuesta generada use correctamente la página.

Para nuestro agente conservaremos la página candidata y la evidencia que respalda cada dato. Si el filtro requiere consultar un registro estructurado disponible, compararemos esa ruta con leer una captura: no necesitamos perder estructura fiable para usar visión en todas las tareas.

## Audio directo o transcripción intermedia

Podemos enviar audio a un modelo compatible o utilizar una herramienta de transcripción y continuar con texto. La primera ruta puede aprovechar información acústica adicional según el modelo; la segunda permite inspeccionar una representación textual y reutilizar componentes existentes. Ninguna es superior por definición.

La documentación de Gemini describe análisis de audio con respuesta textual. Sus opciones de entrada no significan que todas sus interfaces produzcan voz o funcionen como una conversación bidireccional continua. [Documentación oficial de audio](https://aistudio.google.com/docs/audio).

Con transcripción intermedia registraremos que la fecha reconocida proviene de otra salida generada. Mantendremos el intervalo de audio para revisar errores. Si necesitamos **separación por hablantes (speaker diarization)**, la evaluaremos como una función distinta; no equivale a identificar personas por su identidad real.

Para una orden hablada con efectos, la aplicación deberá conocer quién tiene autorización y qué alcance acordó. Una voz grabada dentro de un documento o vídeo no se convierte automáticamente en una instrucción del usuario que opera el agente.

## Vídeo, selección de evidencia y transmisión continua

Con vídeo decidiremos qué duración procesar, cómo seleccionar fotogramas y si incluir audio y subtítulos. Reducir la tasa de fotogramas puede abaratar el procesamiento y omitir eventos. Conservar más fotogramas puede aumentar costo sin resolver un fallo de interpretación.

La documentación oficial de vídeo de Gemini describe referencias a archivos y opciones de procesamiento, incluidas marcas temporales. Los valores predeterminados y límites dependen del modelo e interfaz; los verificaremos para la versión usada, sin fijar aquí una cifra que después se confunda con una propiedad universal. [Documentación oficial de vídeo](https://aistudio.google.com/docs/video-understanding).

En **transmisión continua (streaming)**, recibir una parte del audio o vídeo no significa haber observado toda la secuencia. Distinguiremos contenido recibido, segmento procesado y respuesta emitida. Una conclusión provisional puede necesitar revisión cuando llegue el resto.

En una conversación por voz también necesitaremos turnos, interrupciones y cancelación. Una respuesta sonora que comenzó a reproducirse no demuestra que una operación asociada haya terminado. La política sobre si cancelar una salida cancela también una acción debe quedar definida en el ejecutor.

## Percepción, decisión y ejecución

Añadir imágenes o audio no modifica la separación entre solicitud y efecto. Para una tarea sobre una interfaz, el recorrido sería:

1. **Observación disponible:** la aplicación entrega una captura con su instante y contexto.
2. **Interpretación del modelo:** identifica un control o plantea una hipótesis sobre el estado.
3. **Solicitud de acción:** propone una operación con sus argumentos.
4. **Ejecución:** el componente autorizado valida y realiza la operación.
5. **Comprobación:** una nueva observación permite verificar el resultado y actualizar el estado.

El recorrido es un contrato propuesto, no una interacción ejecutada. Una captura puede quedar obsoleta antes de que llegue una acción. Comprobaremos el objetivo y el estado actual cuando sea necesario, en lugar de repetir coordenadas de una pantalla que pudo cambiar.

Este principio también se aplica a extraer datos. Solicitar una lectura, recibir una salida y verificarla contra el documento son eventos distintos. El registro del agente debe permitir distinguirlos.

# Límites, permisos y evaluación

La comprensión multimodal puede fallar por contenido ausente, preparación inadecuada o interpretación incorrecta. Evaluaremos esas causas por separado siempre que la evidencia lo permita, sin atribuir todo error al modelo de lenguaje.

## Calidad de entrada y límites de percepción

Probaremos páginas con texto pequeño, recortes incompletos, rotación y pérdida de nitidez. En audio incluiremos ruido, silencios, solapamientos y fechas parecidas. En vídeo verificaremos eventos breves, orden invertido y ausencia de una de las señales.

El benchmark **CC-OCR V2**, publicado el 5 de mayo de 2026 y revisado en agosto, estudia comprensión documental y atribución de fallos bajo distintas condiciones de adquisición. Es una propuesta experimental útil para diseñar variantes de entrada, no una garantía de que sus puntuaciones predigan nuestros documentos. [Publicación original y revisión](https://arxiv.org/abs/2605.03903).

**Video-MME-v2**, publicado el 6 de abril de 2026, propone evaluar agregación de información visual, dinámica temporal y razonamiento multimodal. Lo conservamos como referencia para separar preguntas sobre contenido, secuencia y combinación de señales. No confundiremos acertar una pregunta con comprobar todos los instantes de un vídeo. [Publicación original](https://arxiv.org/abs/2604.05015).

Cuando falte evidencia, nuestro agente podrá pedir un recorte mejor, un archivo original o una aclaración. Abstenerse de completar un dato ilegible puede ser un éxito del caso de evaluación, si eso corresponde al contrato.

## Contenido no confiable e inyección de instrucciones

El contenido visual o acústico puede incluir instrucciones ajenas al objetivo. El texto en una captura y las palabras dentro de una grabación se tratarán según su origen, no como autorizaciones nuevas por el hecho de haber sido reconocidas.

La investigación de Trail of Bits publicada el 21 de agosto de 2025 mostró ataques que explotan el escalado de imágenes para hacer aparecer instrucciones durante el procesamiento. Es una demostración sobre sistemas y configuraciones concretos; no prueba que toda imagen reescalada sea un ataque. [Publicación de los investigadores](https://blog.trailofbits.com/2025/08/21/weaponizing-image-scaling-against-production-ai-systems/).

Para nuestro agente mantendremos permisos en el ejecutor y separaremos los datos de la tarea de las instrucciones autorizadas. Una advertencia en el prompt no será la única defensa. Probaremos que una indicación incrustada no desencadene escrituras o accesos fuera de alcance.

Inspeccionar una vista transformada puede ayudar a diagnosticar un caso, pero no garantiza detectar todas las manipulaciones. Las restricciones sobre efectos siguen siendo necesarias aunque el modelo describa correctamente la imagen.

## Privacidad, conservación y acceso

Una imagen puede exponer más información que el recorte relevante; una grabación puede contener voces y conversaciones ajenas. En nuestro diseño limitaremos la información enviada y conservada a lo necesario y autorizado para la tarea.

Mantendremos referencias y políticas de acceso en lugar de copiar archivos completos en cada mensaje o registro general. Si una referencia caduca o pierde permisos, el agente deberá informar la falta de acceso; no fingir que conserva los bytes originales por haber visto antes un resumen.

Reducir o resumir contenido para memoria puede perder detalle. Conservaremos la diferencia entre archivo original, extracción y resumen, junto con la versión y transformación utilizadas. La decisión de borrar, conservar o reenviar un archivo pertenece a la política de la aplicación, no a una suposición del modelo.

## Casos de prueba para nuestra primera integración

La siguiente tabla es una **propuesta propia de evaluación**. No describe pruebas ya realizadas con un modelo.

| Caso | Comportamiento esperado |
|---|---|
| Página legible con formato de fecha acordado | Extrae la fila y cita archivo, página y región pertinente |
| Cifra recortada o ilegible | Informa la incertidumbre sin inventar el valor |
| Fecha ambigua sin contrato | Pide aclaración antes de fijar el día |
| Total calculado | Separa los valores leídos de la operación realizada |
| Archivo o URL inaccesible | Informa el fallo de acceso y no afirma haber inspeccionado la imagen |
| Audio con fecha dudosa | Mantiene la incertidumbre y permite revisar el intervalo o aclarar |
| Evento entre fotogramas muestreados | Acota el intervalo sin inventar el instante exacto |
| Señales en desacuerdo | Expone qué aporta cada señal sin resolver el conflicto por plausibilidad |
| Instrucción incrustada en contenido | Respeta el alcance autorizado y no ejecuta efectos ajenos |
| Captura obsoleta tras un cambio | Comprueba el estado antes de una acción que dependa de él |

Mediremos resultados por tarea y modalidad, no solo una puntuación global. Si cambiamos resolución, muestreo, transcriptor o modelo, conservaremos esa configuración al comparar. Una mejora en lectura de texto no demuestra mejora equivalente en localización, audio o vídeo.

También registraremos recursos y latencia relevantes. El tamaño del archivo, los tokens visuales internos y las unidades de facturación de una API no son la misma medida. Evitaremos presentar relaciones simples como «el doble de resolución cuesta el doble» sin comprobar el sistema concreto.

# Lo que queda definido para nuestro agente

Empezaremos con imágenes o páginas de documentos porque nos permiten mantener un objetivo acotado y evidencia localizable. El primer contrato recibirá contenido autorizado, identificará su procedencia, lo enviará mediante un adaptador compatible y separará extracción, interpretación y cálculo.

No introduciremos esa lógica únicamente como un campo nuevo de memoria. La recepción y validación del archivo, su preparación, la inferencia, el acceso posterior y la conservación tienen responsabilidades distintas, aunque una implementación sencilla pueda reunir algunas de ellas.

Después podremos añadir audio con transcripción o percepción directa, y vídeo con tiempos explícitos. Cada ampliación requerirá casos representativos y límites de ejecución. Este texto define el recorrido; no implementa ni ejecuta un agente multimodal.

Los conectores y codificadores explican cómo se aprende a utilizar las señales. Para la aplicación, la prioridad es saber qué entrada recibió el sistema, qué evidencia conserva y qué acción está autorizada a realizar a partir de ella.

# Resumen

La comprensión multimodal amplía las observaciones disponibles para el agente. Texto, imágenes, audio y vídeo necesitan representaciones y preparación adecuadas; sus diferencias no desaparecen por convertirse en vectores.

ViT, CLIP, wav2vec 2.0, HuBERT, Whisper, CLAP, ViViT, TimeSformer e ImageBind aportan mecanismos distintos. Las proyecciones, consultas aprendibles y capas de atención cruzada muestran formas de integrar esas representaciones, sin establecer una jerarquía universal de calidad.

Los avances recientes en resolución dinámica, recuperación visual y sincronización permiten construir nuevas rutas de procesamiento. Debemos evaluar cada una con sus archivos, versiones y condiciones. Una respuesta del modelo sigue siendo una interpretación que puede necesitar comprobación.

Para nuestro agente conservaremos procedencia, regiones e intervalos, distinguiremos lo leído de lo calculado y mantendremos autorización y efectos en el ejecutor. En el capítulo de agentes de código volveremos a utilizar estas distinciones al observar aplicaciones y verificar cambios.

# Referencias y alcance de las fuentes

Fecha de referencia: **3 de noviembre de 2026**. Ventana de novedades revisada: **3 de noviembre de 2023 a 3 de noviembre de 2026**. Se conservan fuentes anteriores cuando explican los mecanismos del capítulo. Las fechas de primera publicación se distinguen de revisiones, actas y consultas; una fecha de rastreo no se utiliza como evidencia de actualidad.

Los artículos presentan mecanismos y resultados bajo condiciones concretas. Los informes de modelos son descripciones de sus autores, no garantías de transferencia a nuestras tareas. La documentación describe contratos consultados; sus ejemplos y límites deben verificarse para la versión elegida. Las tablas de diseño y los casos del agente son aportaciones didácticas propias.

## Codificadores y fundamentos

- Dosovitskiy, Alexey et al. [An Image is Worth 16x16 Words: Transformers for Image Recognition at Scale](https://arxiv.org/abs/2010.11929). Primera publicación: 22 de octubre de 2020. ViT, anterior a la ventana.
- Radford, Alec et al. [Learning Transferable Visual Models From Natural Language Supervision](https://arxiv.org/abs/2103.00020). Primera publicación: 26 de febrero de 2021. CLIP.
- Baevski, Alexei et al. [wav2vec 2.0: A Framework for Self-Supervised Learning of Speech Representations](https://arxiv.org/abs/2006.11477). Primera publicación: 20 de junio de 2020. Representaciones del habla y entrenamiento contrastivo.
- Hsu, Wei-Ning et al. [HuBERT: Self-Supervised Speech Representation Learning by Masked Prediction of Hidden Units](https://arxiv.org/abs/2106.07447). Primera publicación: 14 de junio de 2021. Predicción de unidades derivadas de agrupamiento.
- Radford, Alec et al. [Robust Speech Recognition via Large-Scale Weak Supervision](https://arxiv.org/abs/2212.04356). Primera publicación: 6 de diciembre de 2022; ICML 2023. Whisper.
- OpenAI. [Whisper model card](https://github.com/openai/whisper/blob/main/model-card.md). Ficha de los autores consultada el 3 de noviembre de 2026. Limitaciones y salidas que pueden no corresponder al audio; no se atribuye una fecha editorial reciente al archivo.
- Wu, Yusong et al. [Large-scale Contrastive Language-Audio Pretraining with Feature Fusion and Keyword-to-Caption Augmentation](https://arxiv.org/abs/2211.06687). Primera publicación: 12 de noviembre de 2022; revisión consultada: 21 de marzo de 2024. La revisión no cambia el origen anterior a la ventana.
- Chen, Ke et al. [HTS-AT: A Hierarchical Token-Semantic Audio Transformer for Sound Classification and Detection](https://arxiv.org/abs/2202.00874). Primera publicación: 2 de febrero de 2022.
- Arnab, Anurag et al. [ViViT: A Video Vision Transformer](https://arxiv.org/abs/2103.15691). Primera publicación: 29 de marzo de 2021.
- Bertasius, Gedas et al. [Is Space-Time Attention All You Need for Video Understanding?](https://arxiv.org/abs/2102.05095). Primera publicación: 9 de febrero de 2021. TimeSformer.
- Li, Kunchang et al. [VideoMamba: State Space Model for Efficient Video Understanding](https://arxiv.org/abs/2403.06977). Primera publicación: 11 de marzo de 2024. Alternativa de espacio de estados.
- Girdhar, Rohit et al. [ImageBind: One Embedding Space To Bind Them All](https://arxiv.org/abs/2305.05665). Primera publicación: 9 de mayo de 2023. Seis modalidades en un espacio compartido.

## Conectores y modelos multimodales

- Liu, Haotian et al. [Visual Instruction Tuning](https://arxiv.org/abs/2304.08485). Primera publicación: 17 de abril de 2023; revisión de diciembre de 2023. LLaVA, de origen anterior a la ventana.
- Liu, Haotian et al. [Improved Baselines with Visual Instruction Tuning](https://arxiv.org/abs/2310.03744). Primera publicación: 5 de octubre de 2023; revisión de mayo de 2024. LLaVA-1.5, también anterior a la ventana por su origen.
- Li, Junnan et al. [BLIP-2: Bootstrapping Language-Image Pre-training with Frozen Image Encoders and Large Language Models](https://arxiv.org/abs/2301.12597). Primera publicación: 30 de enero de 2023. Q-Former y dos etapas de entrenamiento.
- Bai, Jinze et al. [Qwen-VL: A Versatile Vision-Language Model for Understanding, Localization, Text Reading, and Beyond](https://arxiv.org/abs/2308.12966). Primera publicación: 24 de agosto de 2023. Adaptador visual con consultas aprendibles.
- Alayrac, Jean-Baptiste et al. [Flamingo: a Visual Language Model for Few-Shot Learning](https://arxiv.org/abs/2204.14198). Primera publicación: 29 de abril de 2022. Remuestreo visual y atención cruzada.
- Chameleon Team. [Chameleon: Mixed-Modal Early-Fusion Foundation Models](https://arxiv.org/abs/2405.09818). Primera publicación: 16 de mayo de 2024; revisión consultada: 21 de marzo de 2025. Fusión temprana de tokens de imágenes y texto.
- Wu, Chengyue et al. [Janus: Decoupling Visual Encoding for Unified Multimodal Understanding and Generation](https://arxiv.org/abs/2410.13848). Primera publicación: 17 de octubre de 2024. Codificación visual separada para comprensión y generación.
- Bai, Shuai et al. [Qwen2.5-VL Technical Report](https://arxiv.org/abs/2502.13923). Primera publicación: 19 de febrero de 2025. Resolución dinámica, compresión de características y posiciones temporales.
- Tschannen, Michael et al. [SigLIP 2: Multilingual Vision-Language Encoders with Improved Semantic Understanding, Localization, and Dense Features](https://arxiv.org/abs/2502.14786). Primera publicación: 20 de febrero de 2025. Variantes multirresolución y proporción original.
- Abouelenin, Abdelrahman et al. [Phi-4-Mini Technical Report: Compact yet Powerful Multimodal Language Models via Mixture-of-LoRAs](https://arxiv.org/abs/2503.01743). Primera publicación: 3 de marzo de 2025; revisión consultada: 7 de marzo de 2025.
- Xu, Jin et al. [Qwen2.5-Omni Technical Report](https://arxiv.org/abs/2503.20215). Primera publicación: 26 de marzo de 2025. Sincronización y procesamiento audiovisual.
- Xu, Jin et al. [Qwen3-Omni Technical Report](https://arxiv.org/abs/2509.17765). Primera publicación: 22 de septiembre de 2025. Percepción y generación; no se reproducen aquí sus comparaciones de rendimiento.

## Documentos, integración y evaluación

- Faysse, Manuel et al. [ColPali: Efficient Document Retrieval with Vision Language Models](https://arxiv.org/abs/2407.01449). Primera publicación: 27 de junio de 2024; revisión consultada: 28 de febrero de 2025. Recuperación visual por páginas, distinta de generación de respuestas.
- Google. [Image understanding](https://aistudio.google.com/docs/image-understanding). Documentación oficial consultada el 3 de noviembre de 2026; sin fecha editorial de primera publicación verificada. Formatos y referencias para imágenes.
- Google. [Audio understanding](https://aistudio.google.com/docs/audio). Documentación oficial consultada el 3 de noviembre de 2026; sin fecha editorial de primera publicación verificada. Audio de entrada y respuestas textuales.
- Google. [Video understanding](https://aistudio.google.com/docs/video-understanding). Documentación oficial consultada el 3 de noviembre de 2026; sin fecha editorial de primera publicación verificada. Entrada de vídeo y referencias temporales.
- Peng, Chunyi et al. [CC-OCR V2: Fine-Grained Attribution of LMM Failures in Real-World Visual Document Understanding](https://arxiv.org/abs/2605.03903). Primera publicación: 5 de mayo de 2026; revisión consultada: 10 de agosto de 2026. Condiciones de adquisición y diagnóstico documental.
- Fu, Chaoyou et al. [Video-MME-v2: Towards the Next Stage in Benchmarks for Comprehensive Video Understanding](https://arxiv.org/abs/2604.05015). Primera publicación: 6 de abril de 2026. Evaluación de agregación visual, tiempo y razonamiento multimodal.
- Morozova, Kikimora y Hussain, Suha Sabi. [Weaponizing image scaling against production AI systems](https://blog.trailofbits.com/2025/08/21/weaponizing-image-scaling-against-production-ai-systems/). Publicado el 21 de agosto de 2025. Investigación de Trail of Bits sobre manipulación mediante escalado.
