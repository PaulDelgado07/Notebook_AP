# ¿Está actualizada mi dependencia? Explorando los desafíos de las actualizaciones automatizadas de dependencias

**Autores:** Miguel Botto-Tobar¹², Santiago A. Vidal³, Claudia Marcos³

¹ Facultad de Informática, Universidad Abierta Interamericana (UAI), Buenos Aires, Argentina.  
² Universidad de Guayaquil, Guayaquil, Ecuador.  
³ Universidad Nacional del Centro de la Provincia de Buenos Aires (UNICEN), Tandil, Argentina.

**Correspondencia:** miguelangel.bottotobar@alumnos.uai.edu.ar  
**DOI:** https://doi.org/10.21203/rs.3.rs-9390837/v1  
**Fecha de publicación:** 11 de mayo de 2026  
**Licencia:** Creative Commons Attribution 4.0 International

> **Nota:** Este es un preprint que no ha sido sometido a revisión por pares. No debe considerarse concluyente, utilizarse para informar la práctica clínica ni ser referenciado por los medios de comunicación como información validada.

---

**Palabras clave:** gestión de dependencias, JUnit, coevolución de código de prueba, generación de código, Maven, mantenimiento de software.

---

## Resumen

Las actualizaciones de dependencias a menudo requieren ediciones coordinadas en el código de la aplicación o de las pruebas; sin embargo, las herramientas de automatización convencionales actualizan únicamente los manifiestos de construcción y dejan la adaptación del código en manos de los desarrolladores. Este artículo evalúa si estos co-cambios pueden aprenderse para las actualizaciones de JUnit en proyectos Java basados en Maven. A partir de 2,223 commits de actualización extraídos de GitHub, derivamos un conjunto de datos a nivel de fragmentos (snippets) de 8,735 pares de entrada-salida, cada uno de los cuales combina un bloque de dependencia XML, un fragmento de prueba en Java y una instrucción en lenguaje natural.

Un análisis empírico de cuatro partes muestra que la mayoría de las actualizaciones de JUnit 3.x/4.x son conservadoras: el 78.8% de los pares a nivel de método cambian menos del 8% de los caracteres (mediana de la distancia de Levenshtein normalizada: 0.02), y la magnitud varía poco entre los tipos de versiones (distancias de edición promedio: 0.466-0.486; Kruskal-Wallis $p=0.026$). La descomposición de fragmentos, los marcadores de modalidad y el filtrado de casi duplicados arrojan una tasa de copia de 0.71 dentro de un límite de 512 tokens. Un modelo CodeT5 ajustado (fine-tuned) alcanza un EM-Java del 38.5% y un CodeBLEU de 0.854 en fragmentos reservados, superando a una línea base que solo actualiza el manifiesto (0% y 0.502), obteniendo las mayores ganancias en ediciones sustanciales (+0.528 CodeBLEU). Bajo nuestro entorno de construcción, los parches generados por el modelo compilan y superan las pruebas en los 37 casos evaluables (5 de 21 repositorios); los incrementos que solo modifican el manifiesto fallan en el 50% de los casos de control. El conjunto de datos y el marco de evaluación se han publicado abiertamente.

---

## 1. Introducción

Los sistemas de software modernos dependen en gran medida de bibliotecas y marcos de trabajo de terceros que normalmente se denominan dependencias. En ecosistemas como Maven, los paquetes están conectados por densas redes de dependencias en las que el uso de versiones obsoletas (frecuentemente descrito como retraso técnico o *technical lag*) se asocia con una mayor exposición a vulnerabilidades conocidas, incompatibilidades y costos adicionales de mantenimiento [1-3]. Por esta razón, actualizar las dependencias no es una mejora ocasional, sino una obligación recurrente de mantenimiento para reducir los riesgos de seguridad y confiabilidad, y para mantener la capacidad de evolución de los sistemas.

Sin embargo, realizar una actualización rara vez consiste en un simple cambio de versión: los mantenedores a menudo necesitan conocimientos específicos del ecosistema para interpretar las notas de lanzamiento y los cambios drásticos (*breaking changes*), resolver conflictos de dependencias y adaptar el código del cliente y las pruebas en consecuencia; las actualizaciones pueden ser disruptivas, consumir mucho tiempo y ser riesgosas. Trabajos previos en diversos gestores de paquetes demuestran que muchos proyectos se quedan rezagados respecto a las últimas versiones disponibles y que este retraso puede persistir durante largos períodos, incluso cuando las actualizaciones están disponibles [2, 3]. En consecuencia, existe una creciente necesidad de herramientas y técnicas que reduzcan el esfuerzo manual necesario para mantener las dependencias al día.

Para acortar esta brecha, se han desarrollado numerosas herramientas automatizadas de gestión de dependencias. En el ecosistema Java/Maven, el Versions Maven Plugin puede listar y aplicar las actualizaciones disponibles en las dependencias analizando los archivos `pom.xml`. A nivel de plataforma, servicios alojados como GitHub Dependabot y Renovate abren automáticamente *pull requests* que incrementan las entradas del manifiesto a versiones más recientes, con un enfoque particular en las actualizaciones relacionadas con la seguridad [4-6]. La evidencia indica que estas herramientas facilitan la detección y aplicación de actualizaciones, reduciendo así la tendencia de los proyectos a quedarse rezagados [3, 5, 6]. No obstante, todas estas herramientas operan exclusivamente en el manifiesto de dependencias; es decir, proponen el cambio de versión pero dejan cualquier adaptación requerida en el código de la aplicación y de las pruebas enteramente en manos del desarrollador.

A pesar del uso generalizado de verificadores automáticos de dependencias (Dependabot, Renovate, Maven Enforcer), las actualizaciones de los marcos de prueba como JUnit representan un caso particularmente desafiante: a menudo siguen rompiendo los proyectos porque requieren cambios de código a nivel de API que las herramientas basadas solo en manifiestos no pueden anticipar. Cuando una actualización no compila o altera el comportamiento observable, los desarrolladores deben ajustar manualmente tanto el código de la aplicación como el de las pruebas para adaptarlos a la nueva versión de la biblioteca. Los trabajos previos sobre migración de bibliotecas y evolución de APIs han producido técnicas para la extracción de migraciones y la generación de parches, aunque estos se dirigen en gran medida al código de producción [7-12].

Las migraciones en el código de prueba, en particular las actualizaciones de marcos de trabajo, siguen estando poco exploradas a pesar de su alta frecuencia y relevancia práctica. El código de prueba a menudo constituye una parte sustancial de la base de código de un proyecto y sirve como la principal red de seguridad durante la evolución del software [13]; sin embargo, rara vez es el objetivo de las técnicas de migración. Cuando las actualizaciones de dependencias rompen las pruebas, los desarrolladores deben elegir entre reparaciones manuales costosas o posponer las actualizaciones, lo que acumula retraso técnico [2, 3], y las herramientas actuales casi no brindan soporte directo para actualizar las pruebas, más allá de reportar los fallos. Dado que cada una de estas actualizaciones deja un rastro de coevolución en el control de versiones, el historial acumulado entre proyectos constituye, en principio, una fuente reutilizable de conocimiento sobre la evolución, una que podría aprenderse para asistir futuros cambios de forma automática.

Para abordar esta brecha, este artículo examina las actualizaciones del marco de pruebas JUnit en proyectos Java basados en Maven y cómo estas actualizaciones afectan tanto a la configuración como al código de prueba. JUnit es un estándar de facto para las pruebas unitarias en el ecosistema Java; de hecho, aparece como dependencia en la mayoría de los proyectos basados en Maven, lo que lo convierte en un sujeto ideal para estudiar la coevolución del código de prueba a gran escala. Nuestro objetivo principal es comprender cómo evolucionan realmente las dependencias de JUnit en la práctica y proporcionar un conjunto de datos reutilizable que capture estos co-cambios. Como objetivo secundario, evaluamos si un modelo especializado y consciente de JUnit puede aprovechar este conjunto de datos para generar parches de actualización de dependencias realistas. Más específicamente, estudiamos (i) hasta qué punto los parches generados se asemejan a las ediciones escritas por los desarrolladores, (ii) cómo varía el esfuerzo de actualización entre los diferentes tipos de lanzamientos de JUnit, y (iii) cómo influyen las elecciones en la construcción del conjunto de datos en la capacidad de aprendizaje de estas actualizaciones. Organizamos nuestro estudio en torno a las siguientes preguntas de investigación:

- **RQ1.** ¿Hasta qué punto puede el modelo reproducir parches de actualización de dependencias escritos por desarrolladores para JUnit?
- **RQ2.** ¿Hasta qué punto difiere la magnitud de la actualización entre los lanzamientos de parches (*patch*), menores (*minor*) y otros lanzamientos de JUnit?
- **RQ3.** ¿Cómo influyen el preprocesamiento y el diseño del conjunto de datos en la capacidad de aprendizaje de las actualizaciones de dependencias de JUnit?
- **RQ4.** ¿Cómo se relaciona el enfoque propuesto con las herramientas y flujos de trabajo de actualización de dependencias existentes?

Para responder a estas preguntas, combinamos métricas de similitud léxica (ROUGE, BLEU, distancia de Levenshtein normalizada) con estimaciones de CodeBLEU conscientes de la estructura [14] para la RQ1; una caracterización empírica del comportamiento de actualización a través de los tipos de lanzamiento para la RQ2; un análisis a nivel de corpus de los efectos del preprocesamiento para la RQ3; y una comparación frente a una línea base de actualizadores de solo manifiesto junto con una validación funcional mediante un oleoducto (*pipeline*) de Tasa de Aprobación de Construcción y Pruebas (BTR) sobre 172 casos de 21 repositorios para la RQ4.

En general, este artículo realiza cuatro contribuciones:

1. Una caracterización empírica de 2,223 commits de actualización de JUnit que muestra que la mayoría de las actualizaciones son conservadoras;
2. Un conjunto de datos a nivel de fragmentos (*snippets*) de 8,735 pares de entrada-salida para la generación de actualizaciones orientadas a JUnit;
3. Una evaluación comparativa de CodeT5-base y CodeT5+ para la generación de parches de actualización de dependencias, demostrando que el tamaño de la ventana de contexto es un factor crítico para la corrección funcional; y
4. El posicionamiento de la generación basada en modelos dentro de los flujos de trabajo de gestión de dependencias existentes, incluyendo la validación funcional condicional mediante BTR en 5 repositorios evaluables (de 21 muestreados).

Al centrarse en un marco de pruebas concreto y ampliamente utilizado, este trabajo aporta evidencia empírica y artefactos a la emergente intersección entre la gestión automatizada de dependencias y los modelos de generación de código. Ilustra cómo se puede aprovechar la coevolución histórica de la configuración y las pruebas para entrenar modelos especializados que ayuden a los desarrolladores con actualizaciones de baja disrupción y, además, destaca tanto las oportunidades como los límites de tales enfoques en el contexto de JUnit 4.x y Maven.

---

## 2. Trabajo Relacionado

Las bibliotecas de terceros son hoy en día la columna vertebral del desarrollo de software moderno; sin embargo, mantenerlas actualizadas sigue siendo difícil en la práctica. Varios estudios empíricos han cartografiado este panorama a través de los ecosistemas y documentan tanto un retraso técnico persistente como una adopción irregular de la disciplina de versiones. Kula et al. analizaron 4,600 proyectos de GitHub y 2,700 bibliotecas, y reportaron que el 81.5% de los sistemas conservan dependencias obsoletas incluso cuando se divulgan vulnerabilidades, en muchos casos debido a brechas de conciencia y al esfuerzo percibido [15]. Además, Dietrich et al. midieron las prácticas de versionado en 17 gestores de paquetes y más de 70 millones de dependencias; sus resultados revelan una adopción limitada y específica de cada ecosistema para el versionado semántico, así como una tensión sustancial entre las restricciones de versión estrictas y flexibles [16]. Ambos estudios apuntan a la misma conclusión: las organizaciones dependen en gran medida de paquetes externos, pero la actualización sistemática está lejos de ser una rutina. Estudios de ecosistemas más recientes en dominios especializados (por ejemplo, entornos de aprendizaje automático) refuerzan esta observación y destacan desafíos adicionales como las dependencias transitivas frecuentes y la fragilidad de la compatibilidad [17, 18]. En resumen, la evidencia identifica un doble desafío: (i) cómo determinar cuándo es necesaria una actualización y (ii) cómo ejecutarla sin romper las construcciones (*builds*) o las pruebas.

Con el fin de reducir el esfuerzo manual, las plataformas han adoptado bots de actualización (por ejemplo, Dependabot, Renovate) que crean *pull requests* (PRs) para incrementar versiones. He et al. realizaron un estudio exploratorio sobre el impacto de Dependabot y la recepción por parte de los desarrolladores: tras su adopción, los proyectos reducen el retraso técnico y suelen ser receptivos a las PRs; sin embargo, los desarrolladores también moderan las notificaciones y reportan la falta de señales de compatibilidad, y una fracción no trivial termina abandonando el bot [6]. Hejderup y Gousios, por otro lado, investigaron el grado en que las pruebas realmente ejercitan las llamadas a la API de las dependencias en 521 proyectos Java bien probados; encontraron una cobertura de solo el 58% de las llamadas directas y el 21% de las transitivas. Sobre la base de estos resultados, argumentan que la automatización que depende únicamente de las pruebas puede pasar por alto conflictos sutiles [19]. La implicación es pragmática: una ejecución exitosa de la integración continua (CI) tras la PR de un bot no es una prueba confiable de compatibilidad funcional, especialmente para cambios transitivos o desviaciones de comportamiento que las pruebas no capturan.

Las encuestas destacan cómo las bibliotecas de terceros introducen vulnerabilidades y, en algunos casos, cargas útiles maliciosas [20]. Por ejemplo, la colección "Backstabber's Knife Collection" de Ohm et al. cataloga 174 paquetes maliciosos del mundo real en npm, PyPI y RubyGems, y describe técnicas de ataque comunes; muestran cómo las prácticas del ecosistema permiten vectores de inyección en todo el árbol de dependencias [21]. Ladisa et al., a su vez, sistematizan las estrategias de explotación en tiempo de instalación y de ejecución disponibles para los atacantes a través de las características del gestor de paquetes y ofrecen recomendaciones defensivas para los consumidores de dependencias [22]. En paralelo, trabajos recientes sobre la Lista de Materiales de Software (SBOM) subrayan tanto la promesa como la variabilidad de las herramientas utilizadas para enumerar y escanear dependencias, una variabilidad que puede cambiar materialmente las evaluaciones de vulnerabilidad [23, 24].

Los trabajos empíricos observan que las actualizaciones a menudo requieren ediciones sincronizadas (por ejemplo, migrar APIs o adaptar pruebas) en lugar de cambios aislados de versión [15, 16]. Más allá de los bots, la comunidad de análisis de programas proporciona componentes fundamentales para las transformaciones automatizadas: la diferenciación de árboles de sintaxis abstracta (AST) (por ejemplo, GumTree) admite el análisis de cambios de grano fino [25]; Spoon, además, ofrece un marco robusto de transformación de Java adecuado para refactorizaciones y migraciones programáticas [26]. RefactoringMiner, adicionalmente, aporta una detección de refactorizaciones de alta precisión y diffs AST a nivel de commit, lo que permite extraer patrones de cambio consistentes que vinculan las ediciones de configuración con las ediciones de código.

Los grandes modelos preentrenados para código han avanzado rápidamente y ahora admiten tareas de transformación más allá de la síntesis de código a partir de lenguaje natural. CodeBERT (solo codificador) aprende representaciones bimodales de lenguaje natural y de programación y, como resultado, mejora las tareas de búsqueda y comprensión [27]. PLBART adopta una arquitectura de secuencia a secuencia (*sequence-to-sequence*) entrenada mediante eliminación de ruido para servir tanto a la comprensión como a la generación en Java/Python [28]. CodeT5 pasa a un codificador-decodificador unificado con un preentrenamiento consciente de identificadores (es decir, distingue identificadores y aprovecha los comentarios mediante generación dual), mostrando resultados sólidos en múltiples tareas de generación [29].

Los modelos de lenguaje grandes posteriores de tipo solo decodificador, entrenados en corpus enormemente mayores, incluidos Code Llama [30], StarCoder2 [31], DeepSeek-Coder-V2 [32], y modelos más recientes ajustados por instrucciones como Qwen2.5-Coder y las series o1/o3, han mejorado su rendimiento en tareas de refinamiento, reparación y migración de código. Estudios recientes han aprovechado estas capacidades (a menudo mediante *prompting* de cero o pocos ejemplos, agentes o ajuste fino ligero) con el fin de automatizar las reparaciones de código ante actualizaciones de dependencias que rompen el sistema o migraciones de bibliotecas [33-36]. Nuestro trabajo se diferencia en su enfoque en la generación emparejada proactiva tanto de la actualización de la declaración de dependencia de Maven como de las adaptaciones correspondientes en el código de prueba para el caso específico de las migraciones de JUnit 3→4, y utiliza el aprendizaje supervisado de secuencia a secuencia a partir de co-cambios históricos extraídos, en lugar de una reparación basada en LLMs posterior al fallo.

---

## 3. Metodología

En este estudio, investigamos la automatización de las actualizaciones de dependencias en proyectos Java utilizando Maven y JUnit hasta febrero de 2025. Construimos un conjunto de datos a partir de repositorios de código abierto de GitHub y ajustamos un modelo CodeT5 para generar actualizaciones que modifican tanto las declaraciones de dependencia en `pom.xml` como el código de prueba Java asociado. El paquete de replicación de nuestro estudio, que incluye los scripts de generación del conjunto de datos, los conjuntos de datos crudos y procesados, y el código de entrenamiento del modelo, está disponible públicamente en [REVISIÓN DOBLE CIEGA].

### 3.1 Recopilación de datos

#### 3.1.1 Selección de repositorios

Para crear nuestro conjunto de datos, desarrollamos un script personalizado en Python que emplea la API de GitHub para recopilar datos de repositorios que cumplieran con los siguientes criterios de selección: (i) al menos 20 commits, para garantizar una actividad continua; (ii) un mínimo de 2 colaboradores activos, para reflejar prácticas de software colaborativas; (iii) al menos 20 estrellas y 10 bifurcaciones (*forks*), como indicadores de validación de la comunidad; y (iv) el uso explícito de JUnit declarado en el `pom.xml`, verificado mediante el análisis de los bloques de dependencias de Maven. Nos centramos en JUnit dado que se utiliza ampliamente en proyectos Java y porque sus actualizaciones pueden requerir cambios coordinados en el código de prueba más allá de las ediciones del manifiesto.

La aplicación de estos criterios dio como resultado un conjunto de 43 repositorios Java basados en Maven alojados en GitHub. Cada repositorio se clonó localmente para poder inspeccionar su historial completo de control de versiones.

#### 3.1.2 Identificación de commits

Un archivo `pom.xml` (*Project Object Model*) de Maven es un descriptor XML que declara dependencias, complementos (*plugins*) y metadatos del proyecto. Dado que cada proyecto Maven debe proporcionar un `pom.xml`, es un artefacto confiable para la identificación de las versiones de las dependencias. Extrajimos los commits que modificaban las versiones de las dependencias de JUnit y registramos los metadatos del repositorio, las versiones de JUnit antes y después de la actualización, el tipo de actualización (parche, menor, mayor) clasificado según el versionado semántico [37], los fragmentos XML del POM antes y después del cambio, y los archivos de prueba de Java asociados, centrándonos en las declaraciones de importación, las anotaciones (por ejemplo, `@Test`, `@Before`) y los cuerpos de los métodos.

No todos los commits que editan el `pom.xml` son adecuados para nuestro análisis. Incluimos aquellos commits que cambian la etiqueta `<version>` de una dependencia JUnit existente, afectando potencialmente al código de prueba mediante la modificación de importaciones, anotaciones o cuerpos de métodos en archivos bajo `src/test/java`, y que representan actualizaciones de JUnit de un solo paso que pueden clasificarse como parches, menores o mayores. Excluimos los commits de fusión (*merge commits*), los commits donde el `pom.xml` solo cambia en formato o comentarios, y los commits que añaden o eliminan JUnit por completo en lugar de actualizar su versión. Los commits que actualizan la versión de JUnit pero no modifican ningún archivo de prueba se mantienen en el corpus a nivel de commit, pero se excluyen del conjunto de datos de entrenamiento a nivel de fragmento.

### 3.2 Construcción de los datos

Tras aplicar los criterios de filtrado, obtenemos un conjunto de datos a nivel de commit de 2,223 commits de actualización de JUnit 3.x/4.x, que utilizamos para caracterizar cómo evolucionan las declaraciones de dependencias de JUnit en la práctica. Del subconjunto de commits que también modifican las pruebas, derivamos un conjunto de datos a nivel de fragmentos de 8,735 pares estructurados de entrada-salida que vinculan las actualizaciones de dependencias de JUnit en el `pom.xml` con las ediciones correspondientes en el código de prueba.

Cada par vincula un contexto previo a la actualización con la actualización correspondiente escrita por el desarrollador y contiene:

- **Entrada:** un bloque de dependencia XML, un fragmento de código de prueba en Java y una instrucción de la tarea, concatenados con marcadores de modalidad explícitos (ver Sección 3.3).
- **Salida:** el bloque de dependencia XML actualizado y el código Java modificado correspondiente, utilizando el mismo formato de marcadores.

Serializamos los ejemplos en formato JSONL para una ingesta reproducible y publicamos tanto las formas crudas como las procesadas en el paquete de replicación. Conservamos explícitamente las actualizaciones de JUnit que en Maven suelen tener el alcance (*scope*) de test. Aunque los trabajos anteriores a menudo excluyen las dependencias con alcance de prueba, las incluimos porque las actualizaciones de JUnit afectan directamente a las suites de prueba de Java y son críticas para la confiabilidad del software. Las dependencias se categorizaron en actualizaciones de parches, menores y mayores siguiendo el versionado semántico [37]. No observamos JUnit 5 en los commits de actualización extraídos.

### 3.3 Preprocesamiento

Aplicamos pasos de preprocesamiento diseñados para preservar la estructura sintáctica al tiempo que se reduce el ruido:

- **Normalización de espacios en blanco:** colapsamos las líneas en blanco consecutivas y los espacios finales tanto en los fragmentos XML como Java para que la variación de formato no infle las distancias de edición.
- **Eliminación de comentarios:** eliminamos los comentarios de línea y de bloque del código Java para centrar el modelo en el contenido estructural y semántico en lugar de las anotaciones en lenguaje natural.
- **Marcadores de modalidad:** anteponemos los tokens `[XML]`, `[JAVA]` y `[TASK]` a las secciones correspondientes de cada ejemplo. Estos marcadores proporcionan límites claros entre las tres modalidades y se registran como tokens especiales en el tokenizador.
- **Deduplicación:** eliminamos los pares de entrada-salida que fuesen casi duplicados utilizando la distancia de Levenshtein normalizada [38] para reducir la sobrerrepresentación de ediciones altamente repetitivas.
- **División:** dividimos los 8,735 ejemplos aleatoriamente en un 80% para entrenamiento (6,988 ejemplos) y un 20% para un conjunto de prueba reservado (1,747 ejemplos) utilizando una semilla aleatoria fija (seed=42) para garantizar la reproducibilidad. Esta división se realiza a nivel de ejemplo, no a nivel de repositorio; discutimos esto como una amenaza a la validez en la Sección 6.

Las entradas a nivel de fragmento tienen una longitud promedio de 535 caracteres (σ=265), y ninguna secuencia de entrada o salida supera el límite de 1,024 tokens del codificador tras la tokenización de subpalabras. Sin embargo, para CodeT5-base, el 55% de las entradas superó su límite de 512 tokens, lo que provocó truncamientos; esto motivó la adopción de CodeT5+, como se describe en la Sección 3.4.

### 3.4 Modelo

Inicialmente ajustamos CodeT5-base [29], un transformador codificador-decodificador con 220 millones de parámetros y un contexto máximo de 512 tokens. No obstante, este modelo produjo salidas truncadas en una fracción significativa de las entradas (el 55% superó el límite de 512 tokens), lo que condujo a predicciones incoherentes y a un BTR=0% al ser evaluado a través del flujo de parcheo quirúrgico a nivel de commit (Sección 4.4.2). Por lo tanto, ampliamos la evaluación a CodeT5+ [39], que admite una ventana de contexto de 1,024 tokens. Como resultado, esto eliminó por completo el truncamiento y permitió la generación de parches funcionales. A menos que se indique lo contrario, los resultados reportados en la RQ4 corresponden al modelo CodeT5+.

#### 3.4.1 Configuración del entrenamiento

Utilizamos la biblioteca Hugging Face Transformers (v4.47.1) [40] para ajustar ambos modelos con los siguientes hiperparámetros compartidos: tasa de aprendizaje $3\times10^{-5}$, 10 épocas, tamaño de lote de 2 con acumulación de gradiente en 4 pasos (tamaño de lote efectivo de 8), optimizador AdamW con un programa lineal de tasa de aprendizaje y entrenamiento de precisión mixta (fp16). La longitud máxima de la secuencia difiere entre los modelos: 512 tokens para CodeT5-base y 1,024 tokens para CodeT5+. Los tres marcadores de modalidad (`[XML]`, `[JAVA]`, `[TASK]`) se registraron como tokens especiales en el tokenizador de ambos modelos, y la capa de incrustación (*embedding*) se redimensionó en consecuencia. La semilla de entrenamiento se fijó en 42 en ambos casos.

#### 3.4.2 Formato de entrada y salida

Cada secuencia de entrada está estructurada como una concatenación de tres secciones separadas por los tokens especiales:

```
[XML] <dependency>...<version>4.12</version>...</dependency>
[JAVA] public void testSomething() {...}
[TASK] Actualizar la versión del artifact: junit
```

La salida (etiqueta) sigue el mismo formato y contiene el bloque de dependencia XML actualizado junto con el código de prueba Java modificado. Durante el forzamiento del docente (*teacher-forcing*), el decodificador aprende a generar ambas secciones condicionadas al contexto de entrada. Durante la inferencia, el modelo genera la secuencia de salida completa de forma autorregresiva utilizando una búsqueda de haz (*beam search*) con un ancho de haz de 4 y una penalización de longitud de 1.0.

#### 3.4.3 Decodificación y postprocesamiento

En el momento de la inferencia, el modelo genera la salida con `skip_special_tokens=True`, lo que elimina los marcadores de modalidad de la secuencia decodificada. Extraemos las secciones Java y XML tanto de la predicción como de la referencia utilizando heurísticas basadas en límites: para las referencias con marcadores explícitos de `[JAVA]`, extraemos el texto entre `[JAVA]` y `[TASK]`; para las salidas del modelo sin marcadores, identificamos la última etiqueta de cierre XML y tratamos el texto restante como la sección Java. Esta extracción permite obtener métricas justas a nivel de componente (EM-Java, EM-XML).

### 3.5 Evaluación

Evaluamos el modelo ajustado en la división de prueba reservada ($N=200$ muestreados aleatoriamente de los 1,747 ejemplos de prueba, debido a restricciones computacionales). Para cada ejemplo de prueba, el modelo recibe la secuencia de entrada y genera un parche candidato, que luego comparamos con el parche escrito por el desarrollador utilizando las siguientes métricas:

#### 3.5.1 EM-Java (Exact Match - Java)

La fracción de predicciones cuya sección Java extraída coincide exactamente con la referencia tras la normalización de espacios en blanco:

$$EM\text{-}Java = \frac{1}{N}\sum_{i=1}^{N}\mathcal{F}[\hat{y}_{i}^{java}=y_{i}^{java}]$$

donde $\hat{y}_{i}^{java}$ y $y_{i}^{java}$ denotan los fragmentos de Java extraídos de la predicción y de la referencia, respectivamente. Esta métrica aísla la calidad de la generación de código del incremento de versión en el XML.

#### 3.5.2 CodeBLEU

Adoptamos CodeBLEU [14], una métrica compuesta que aumenta el BLEU estándar [41] con componentes conscientes del código:

$$CodeBLEU = \alpha \cdot BLEU + \beta \cdot BLEU_{w} + \gamma \cdot Match_{syntax} + \delta \cdot Match_{dataflow}$$

Fijamos $\alpha = \beta = \gamma = \delta = 0.25$ siguiendo la formulación original. CodeBLEU se calcula únicamente en las secciones de Java extraídas. Nuestra implementación utiliza una aproximación basada en expresiones regulares para $Match_{syntax}$ y $Match_{dataflow}$ en lugar de un analizador completo de tree-sitter; reportamos esto como una limitación en la Sección 6.

#### 3.5.3 Métricas adicionales

También reportamos BLEU [41], ROUGE-L [42], la distancia de Levenshtein normalizada [38] y las subpuntuaciones individuales de $Match_{syntax}$ y $Match_{dataflow}$. Asimismo, reportamos el EM-XML (coincidencia exacta en la sección XML extraída) y analizamos la distribución de errores entre las predicciones no exactas utilizando una taxonomía basada en la similitud a nivel de token: casi fallos (*near-misses*, >80% de similitud), actualizaciones parciales, salidas truncadas y transformaciones incorrectas.

#### 3.5.4 Línea base (Baseline)

Comparamos el modelo con una línea base de solo manifiesto que simula el comportamiento de herramientas como Dependabot [43] y Renovate [44]: aplica el cambio de versión en el `pom.xml` pero deja el código de prueba Java intacto. En el conjunto de datos a nivel de fragmentos, donde cada ejemplo contiene por construcción un cambio de código Java, la línea base de solo manifiesto logra un 0% de EM-Java por definición.

#### 3.5.5 Alcance y limitaciones de las métricas

Compilar y ejecutar cada proyecto para cada parche generado es computacionalmente costoso a esta escala. Por esta razón, complementamos las métricas de similitud con una evaluación funcional de la Tasa de Aprobación de Construcción y Pruebas (BTR) sobre 172 casos estratificados de 21 repositorios de código abierto, detallada en la Sección 4.4.2. Es decir, las métricas de similitud sirven como aproximaciones de la corrección del parche en la división reservada completa (bajo una partición a nivel de ejemplo), mientras que BTR proporciona evidencia funcional condicional en el subconjunto evaluable.

---

## 4. Resultados

Esta sección aborda nuestras cuatro preguntas de investigación presentando los análisis cuantitativos y cualitativos derivados del conjunto de datos, las salidas del modelo y la caracterización empírica del comportamiento real de las actualizaciones de JUnit.

### 4.1 RQ1: ¿Hasta qué punto puede el modelo reproducir los parches de actualización de dependencias escritos por los desarrolladores?

Para responder a la RQ1, evaluamos con qué precisión el modelo ajustado reproduce los parches de actualización de JUnit escritos por desarrolladores utilizando tres perspectivas complementarias: (i) métricas de similitud a nivel de token (ROUGE, BLEU) calculadas en la división de prueba reservada, (ii) un análisis a nivel de corpus sobre qué tan conservadoras son las actualizaciones reales de JUnit 4.x, medido mediante la distancia de edición normalizada en 2,474 pares a nivel de método extraídos del corpus completo de commits, y (iii) la inspección cualitativa de salidas representativas del modelo para ilustrar los patrones típicos de concordancia y los modos de fallo.

#### 4.1.1 Fidelidad léxica y solapamiento

La Tabla 1 presenta las estadísticas descriptivas para ROUGE y BLEU en la división de prueba reservada. Las puntuaciones se expresan como porcentajes.

**Tabla 1:** Estadísticas descriptivas para ROUGE y BLEU en la división de prueba reservada (puntuaciones en %).

| Métrica | Media | Desv. Est. | Mín | 25% | 50% | 75% | Máx |
|---------|-------|------------|-----|-----|-----|-----|-----|
| ROUGE-1 | 88.40 | 4.36 | 80.00 | 85.50 | 88.89 | 91.70 | 94.74 |
| ROUGE-2 | 82.69 | 3.76 | 75.93 | 80.25 | 82.97 | 85.34 | 88.89 |
| ROUGE-L | 88.40 | 4.36 | 80.00 | 85.50 | 88.89 | 91.70 | 94.74 |
| BLEU-4  | 80.15 | 3.05 | 73.73 | 78.25 | 80.72 | 82.41 | 83.67 |

Estos resultados indican que los parches generados por el modelo están cerca de las referencias escritas por los desarrolladores a nivel de tokens y de n-gramas cortos. Un ROUGE-L de alrededor del 88% sugiere un solapamiento sustancial de subsecuencias, mientras que un BLEU-4 de alrededor del 80% indica que muchos fragmentos de cuatro gramos se reproducen exactamente.

En el caso mostrado en la Figura 2, la salida del modelo reproduce exactamente el parche del desarrollador: se añade una única cláusula `throws Exception` mientras que el resto del método permanece intacto. Este tipo de edición localizada y sintácticamente patronada es la transformación más común en el corpus y representa el punto más fuerte del modelo.

```java
// Código original (Pre-update)
public void forcedShutdownVerifyingLogs() {
}

// Parche del desarrollador / Generación del modelo
public void forcedShutdownVerifyingLogs() throws Exception {
    // cláusula throws añadida
}
```
*Fig. 2: Actualización de baja distancia: el modelo introduce una cláusula `throws Exception`, coincidiendo exactamente con el parche del desarrollador.*

Por el contrario, la Figura 3 muestra un caso en el que el desarrollador renombra simultáneamente un método y reemplaza su cuerpo. El solapamiento léxico cae drásticamente a pesar de que ambas versiones son pruebas plausibles de forma aislada, lo que ilustra una limitación de las métricas basadas en texto para parches semánticamente equivalentes pero sintácticamente divergentes.

```java
// - Código eliminado (Antes)
- public void shouldDecodeTwoCommands() throws IOException {
-     ...
-     assertThat(bye.getCommandType(), is(BYE_ACK));
- }

// + Código añadido (Después)
+ public void shouldThrowUnsupportedException5() {
+     TestLessInputStreamBuilder builder = new TestLessInputStreamBuilder();
+     builder.getCachableCommands().provideNewTest();
+ }
+ //-- reescritura completa
```
*Fig. 3: Actualización de alta distancia: reescritura completa del método donde el solapamiento léxico es bajo a pesar de que ambas versiones son pruebas plausibles.*

#### 4.1.2 Distancias de edición y conservadurismo en las actualizaciones

Para contextualizar las puntuaciones de similitud, analizamos la magnitud de los cambios en los commits reales de actualización de JUnit 4.x. A partir de los 2,223 commits de nuestro corpus, extraemos 2,474 pares a nivel de método donde están disponibles tanto la implementación anterior como la posterior a la actualización. Para estos pares, la distancia de Levenshtein normalizada entre el código anterior y posterior a la actualización tiene una media de 0.075, una mediana de 0.020 y una desviación estándar de 0.142, con un 78.8% de los pares por debajo de 0.08.

La gran mayoría de las actualizaciones a nivel de método de JUnit 4.x son conservadoras: menos del 8% de los caracteres cambian en una actualización típica, y las modificaciones generalmente se limitan a refinamientos de aserciones, ajustes de importación o adiciones de cláusulas throws. Esta regularidad estructural crea un panorama de aprendizaje favorable para un modelo de secuencia a secuencia; es decir, el modelo puede aprender una estrategia de "copiar y editar" que reproduce la mayor parte de la entrada y aplica modificaciones locales dirigidas.

Este análisis de conservadurismo cubre únicamente los cuerpos de los métodos. Los contextos de anotación (por ejemplo, bloques `@Test` con código circundante) muestran una divergencia sustancialmente mayor (media de Levenshtein normalizada de 0.77 a través de 4,245 pares), ya que la heurística de extracción captura código circundante que a menudo cambia estructuralmente entre versiones.

#### 4.1.3 Comportamiento estructural y CodeBLEU

A nivel de commit, solo entre el 4% y el 6% de los commits implican cambios sustanciales en los cuerpos de los métodos de prueba; incluso en esos casos, los cambios suelen ser modificaciones menores en lugar de adiciones o eliminaciones de métodos enteros. La inspección manual de los pares de referencia del modelo confirma que este preserva en gran medida las firmas de los métodos y las listas de parámetros, enfocando sus ediciones dentro de los cuerpos de los métodos y los bloques de dependencias XML. CodeT5-base alcanza un CodeBLEU de 0.854 y un EM-Java del 38.5%, mientras que CodeT5+ alcanza un CodeBLEU = 0.868 y un EM-Java = 43.6% en la división reservada a nivel de fragmento.

#### 4.1.4 Análisis de errores

Para caracterizar el panorama de dificultad que enfrenta el modelo, clasificamos cada muestra de prueba en función de la divergencia entre la línea base de solo manifiesto y el parche de referencia escrito por el desarrollador. Definimos cuatro categorías de error sobre la base de la similitud a nivel de token:

- **Casi fallo (*Near-miss*):** similitud de tokens superior al 80%, lo que indica que solo se requiere una edición menor y localizada.
- **Actualización parcial (*Partial update*):** similitud entre el 50% y el 80%, donde el desarrollador aplicó cambios adicionales.
- **Transformación incorrecta (*Wrong transform*):** similitud inferior al 50%, correspondiente a reescrituras sustanciales de métodos.
- **Truncado (*Truncated*):** la referencia es más de dos veces más larga que la entrada, lo que indica código nuevo significativo.

**Tabla 2:** Taxonomía de divergencia entre la línea base de solo manifiesto y el parche del desarrollador en la división de prueba reservada a nivel de fragmento ($N=500$).

| Categoría | Cantidad | % | Implicación |
|-----------|----------|---|-------------|
| Casi fallo (>80% sim.) | 182 | 36.4 | Una edición menor es suficiente |
| Actualización parcial (50-80%) | 210 | 42.0 | Se necesitan cambios de código moderados |
| Transformación errónea (<50%) | 77 | 15.4 | Se requiere una reescritura mayor |
| Truncado (<50% longitud) | 31 | 6.2 | Se debe generar código nuevo |
| **Total (todos los no exactos)** | **500** | **100.0** | |

Estos resultados revelan un panorama de dificultad gradual. Más de un tercio de los casos de prueba (36.4%) son casi fallos donde el modelo solo necesita aprender pequeñas ediciones localizadas. Otro 42.0% requiere cambios moderados. El 21.6% restante cae en categorías más difíciles: transformaciones incorrectas (15.4%) y casos truncados (6.2%).

### 4.2 RQ2: ¿Hasta qué punto difiere la magnitud de la actualización entre los lanzamientos de parches, menores y mayores de JUnit?

Para responder a la RQ2, operamos sobre el conjunto de datos filtrado en formato JSONL a nivel de fragmento utilizado para el entrenamiento y la evaluación ($N=8,735$ pares). Para cada par, extraemos la versión de JUnit de la etiqueta `<version>` en el XML de entrada y salida y clasificamos la actualización como parche, menor o mayor de acuerdo con el versionado semántico [37].

#### 4.2.1 Distribución del tipo de cambio y de actualización

**Tabla 3:** Distribución de los tipos de cambios a través de los 8,735 pares a nivel de fragmento.

| Tipo de Cambio | Cantidad | % |
|----------------|----------|---|
| Ediciones en aserciones / sitios de llamada API | 7,220 | 82.7% |
| Ediciones a nivel de anotación | 9 | 0.1% |
| Otros / Estructurales | 1,506 | 17.2% |
| **Total de registros** | **8,735** | **100.0%** |

**Tabla 4:** Distribución y distancias de edición normalizadas por tipo de actualización.

| Tipo | # Pares | % Pares | Media | Mediana | Q1 | Q3 |
|------|---------|---------|-------|---------|----|----|
| Parche (*Patch*) | 4,183 | 47.9% | 0.474 | 0.431 | 0.262 | 0.697 |
| Menor (*Minor*) | 2,542 | 29.1% | 0.466 | 0.424 | 0.242 | 0.692 |
| Mayor (*Major*) | 2,010 | 23.0% | 0.486 | 0.456 | 0.278 | 0.707 |

#### 4.2.2 Distancias de edición normalizadas entre tipos de actualización

Las distancias medias caen en una franja estrecha (0.466-0.486), con medianas entre 0.424 y 0.456 e rangos intercuartílicos ampliamente similares. Una prueba de Kruskal-Wallis arroja $H=7.28$, $p=0.026$, lo que indica una diferencia estadísticamente significativa pero pequeña entre los tres grupos, impulsada principalmente por las actualizaciones mayores.

#### 4.2.3 Magnitud consciente de la estructura (CodeBLEU)

**Tabla 5:** CodeBLEU entre fragmentos Java anteriores y posteriores a la actualización por tipo de actualización ($N=8,735$ pares).

| Tipo | # Pares | Media | Mediana | IQR |
|------|---------|-------|---------|-----|
| Parche (*Patch*) | 4,183 | 0.489 | 0.513 | [0.317, 0.643] |
| Menor (*Minor*) | 2,542 | 0.482 | 0.513 | [0.286, 0.656] |
| Mayor (*Major*) | 2,010 | 0.470 | 0.513 | [0.292, 0.560] |

CodeBLEU confirma el patrón observado con la distancia de edición: los tres tipos de actualización ocupan una estrecha franja de medias (0.470-0.489) y comparten la misma mediana (0.513). Una prueba de Kruskal-Wallis arroja $H=6.98$, $p=0.030$, estadísticamente significativa pero de pequeña magnitud.

**Conclusión RQ2:** La etiqueta de versionado semántico (parche/menor/mayor) es un predictor débil de cuánto código de prueba cambia realmente durante una actualización de dependencias de JUnit.

### 4.3 RQ3: ¿Cómo influyen el preprocesamiento y el diseño del conjunto de datos en la capacidad de aprendizaje de las actualizaciones de dependencias de JUnit?

#### 4.3.1 Diseño del flujo de trabajo y propiedades del corpus

La primera decisión de diseño consiste en descomponer las actualizaciones a nivel de commit en pares a nivel de fragmento, lo que genera aproximadamente 3.93 pares de entrada-salida por commit. La descomposición a nivel de fragmento garantiza que ningún ejemplo supere los 512 tokens (0% de desbordamiento), mientras que el corpus a nivel de commit tiene un 7.0% de ejemplos que superan este umbral.

**Tabla 6:** Estadísticas de longitud de secuencia para los 8,735 pares a nivel de fragmento.

| Métrica | Media | Mediana | P75 | Máx | >512 |
|---------|-------|---------|-----|-----|------|
| Entrada (caracteres) | 535 | 507 | 678 | 2,087 | — |
| Salida (caracteres) | 474 | 480 | 619 | 2,674 | — |
| Entrada (tokens WS) | 38.2 | 36 | 44 | 224 | 0% |
| Salida (tokens WS) | 29.8 | 28 | 34 | 277 | 0% |

#### 4.3.2 Panorama de la capacidad de aprendizaje

La tasa de copia promedio (fracción de tokens de salida que aparecen en la entrada) es de 0.714 (mediana 0.786), lo que indica que una estrategia de copiar y editar puede producir la mayor parte de la salida.

**Tabla 7:** Franjas de dificultad por similitud Java entre entrada y salida.

| Franja de similitud | N | % | Promedio tokens entrada | Promedio tokens salida |
|--------------------|---|---|------------------------|------------------------|
| Alta (>0.80) | 1,528 | 17.5 | 34.0 | 26.5 |
| Moderada (0.50-0.80) | 3,456 | 39.6 | 35.6 | 27.7 |
| Baja (<0.50) | 3,751 | 42.9 | 42.3 | 33.0 |

### 4.4 RQ4: ¿Cómo se relaciona el enfoque propuesto con las herramientas y flujos de trabajo de actualización de dependencias existentes?

#### 4.4.1 Comparación cuantitativa: Solo Manifiesto vs. Parches Generados por el Modelo

**Tabla 8:** Comparación de la línea base en la división reservada a nivel de fragmento ($N=200$).

| Enfoque | EM-Java (%) | CodeBLEU | BLEU | Sintaxis | Flujo de datos |
|---------|-------------|----------|------|----------|----------------|
| Solo manifiesto | 0.0 | 0.502 | 0.486 | 0.293 | 0.726 |
| CodeT5-base (512 tokens) | 38.5 | 0.854 | 0.822 | 0.796 | 0.960 |
| CodeT5+ (1024 tokens) | 43.6 | 0.850 | 0.868 | 0.880 | 0.869 |

**Tabla 9:** CodeBLEU por nivel de dificultad en la división a nivel de fragmento ($N=200$).

| Nivel | N | Solo manifiesto | CodeT5-base | Δ |
|-------|---|-----------------|-------------|---|
| Trivial ($sim. > 0.90$) | 13 | 0.923 | 0.838 | -0.085 |
| Moderado ($sim. \in [0.50, 0.90]$) | 97 | 0.629 | 0.876 | +0.248 |
| Significativo ($sim. < 0.50$) | 90 | 0.303 | 0.832 | +0.528 |

El desglose por niveles revela un claro patrón de complementariedad: el modelo aporta el mayor valor precisamente en los casos donde las herramientas de solo manifiesto se quedan cortas (+0.528 CodeBLEU en casos significativos).

#### 4.4.2 Validación funcional: Tasa de Aprobación de Construcción y Pruebas (BTR)

**Tabla 10:** Resultados de BTR en repositorios evaluables ($N=5$ repositorios, 47 casos). IC de Wilson al 95%.

| Enfoque | Casos | BTR (%) | IC 95% (Wilson) |
|---------|-------|---------|-----------------|
| Solo manifiesto | 10 | 50.0 | [23.7%, 76.3%] |
| CodeT5+ (modelo) | 37 | 100.0 | [90.6%, 100.0%] |

**Tabla 11:** Desglose de BTR por repositorio. PASS: exitoso. FC: fallo de compilación. FT: fallo de prueba. IE: error de infraestructura.

| Repositorio | N | PASS | FC | FT | IE | BTR |
|-------------|---|------|----|----|----|----|
| maven-common-artifact-filters | 39 | 34 | 0 | 5 | 0 | 87.2% |
| maven-gpg-plugin | 3 | 3 | 0 | 0 | 0 | 100.0% |
| executor | 2 | 2 | 0 | 0 | 0 | 100.0% |
| download-maven-plugin | 2 | 2 | 0 | 0 | 0 | 100.0% |
| maven-help-plugin | 1 | 1 | 0 | 0 | 0 | 100.0% |
| **Subtotal Evaluable** | **47** | **42** | **0** | **5** | **0** | **89.4%** |
| olingo-odata4 | 4 | 0 | 4 | 3 | 0 | — |
| maven-enforcer | 14 | 0 | 1 | 3 | 10 | — |
| jasmine-maven-plugin | 13 | 0 | 13 | 0 | 0 | — |
| maven-shade-plugin | 13 | 0 | 0 | 0 | 13 | — |
| sortpom | 13 | 0 | 8 | 5 | 0 | — |
| Otros (11 repositorios) | 29 | 0 | 23 | 5 | 1 | — |
| **Total** | **172** | **42** | **10** | **0** | **16** | **24.4%** |

Todos los 37 parches generados por el modelo compilan y superan las pruebas en los 5 repositorios evaluables (BTR Condicional = 100%), mientras que los incrementos de solo manifiesto fallan en el 50% de los casos de control. Los 16 repositorios restantes no pudieron evaluarse debido a restricciones de infraestructura.

---

## 5. Discusión

### 5.1 Evolución conservadora de JUnit e implicaciones para el modelado

Los análisis en la RQ1 y la RQ2 convergen en un hallazgo central: las actualizaciones de JUnit 3.x/4.x en los repositorios estudiados son predominantemente conservadoras, y este conservadurismo se mantiene en todas las categorías de versionado semántico (Tablas 4-5). Desde el punto de vista del modelado, esta regularidad estructural crea un panorama de aprendizaje favorable: un modelo de secuencia a secuencia puede adoptar una estrategia de copiar y editar en la que la mayoría de los tokens de salida se retienen de la entrada y solo se modifica un pequeño número de posiciones.

Sin embargo, la taxonomía de errores (Tabla 2) delinea los límites de esta ventaja. Los casos de casi fallos y actualizaciones parciales (78.4% combinados) están bien al alcance del modelo, mientras que los casos de transformaciones incorrectas y truncados (21.6%) requieren reescrituras en múltiples sitios o una lógica de prueba completamente nueva que un modelo a nivel de fragmento no puede reconstruir únicamente a partir del contexto local.

La evidencia a través de las preguntas RQ1-RQ3 sugiere que el enfoque es más sólido para co-cambios locales, de sintaxis regular y observados repetitivamente: principalmente ediciones de importación, ediciones a nivel de anotación y ediciones en aserciones o sitios de llamada de la API (Tabla 3). Es intrínsecamente más débil para refactorizaciones estructurales (por ejemplo, movimiento de métodos, renombramientos, extracción/inclusión de métodos) e incompatibilidades semánticas, que requieren un razonamiento entre múltiples archivos y conocimientos de diseño específicos del proyecto.

### 5.2 Posicionamiento respecto a las herramientas existentes

La RQ4 comparó los modelos ajustados frente a la estrategia de solo manifiesto y los situó dentro del panorama más amplio de herramientas. Los resultados cuantitativos (Tablas 8 y 9) muestran que el modelo añade el mayor valor precisamente allí donde las herramientas de solo manifiesto se quedan cortas; a saber, en las actualizaciones que requieren una adaptación sustancial del código de prueba (nivel de cambio significativo). Por el contrario, para los casos triviales, simplemente copiar la entrada ya produce un alto solapamiento, y las modificaciones menores del modelo pueden incluso reducir la similitud superficial.

Los actualizadores de solo manifiesto como Dependabot y Renovate están bien adaptados para proponer cambios de versión e integrarse en flujos de trabajo basados en PRs; sin embargo, se detienen en el punto donde falla la compilación o las pruebas [43, 44, 46]. El modelo ajustado ocupa un terreno intermedio: más especializado que el *prompting* de LLMs de propósito general, de alcance más amplio que los actualizadores de solo manifiesto y menos restringido semánticamente que los motores de refactorización de los IDEs; de ahí la necesidad de una validación mediante construcción y pruebas como mecanismo de control.

### 5.3 Implicaciones prácticas

El patrón de complementariedad observado en la RQ4 sugiere un escenario de integración natural: un bot propone un incremento de la versión de JUnit en el `pom.xml` (por ejemplo, a través de una *pull request*), y luego se invoca al modelo para generar ediciones candidatas para los archivos de prueba afectados en esa misma PR. Una estimación de la dificultad podría determinar qué archivos justifican parches generados por el modelo frente a aquellos donde el incremento de solo manifiesto es suficiente.

Para los equipos que mantienen grandes suites de prueba en muchos módulos, incluso la automatización parcial de estos ajustes puede acortar el ciclo de retroalimentación entre las actualizaciones de dependencias y las construcciones estables. El modelo produce el parche exacto del desarrollador en una fracción sustancial de los casos a nivel de fragmento (Tabla 8) y un casi fallo en otra proporción; en otras palabras, la edición sugerida suele ser correcta o requiere únicamente un refinamiento manual menor.

Una implicación adicional concierne a los contextos de desarrollo multilingües. Debido a que las descripciones de las tareas en nuestro conjunto de datos están expresadas en español, el trabajo ilustra cómo los modelos enfocados en código pueden alinearse con instrucciones en lenguaje natural distintas al inglés sin sacrificar la calidad de la generación de código.

### 5.4 Trabajo futuro

La prioridad más inmediata es una prueba de estrés de BTR con múltiples fragmentos que agrupe todas las predicciones del modelo que pertenecen al mismo commit, las aplique atómicamente y ejecute la suite de pruebas completa. Esto permitiría detectar fallos de interacción que el parcheo de un solo fragmento no puede descubrir.

En segundo lugar, ampliar el conjunto de datos y el modelo para cubrir JUnit 5 y otros marcos de prueba (por ejemplo, TestNG) permitiría evaluar si el enfoque se escala más allá del régimen conservador de JUnit 4.x. Tercero, ampliar la cobertura de BTR más allá de los 5 repositorios evaluables actuales en una infraestructura dedicada proporcionaría intervalos de confianza más estrechos. Cuarto, una prueba de rendimiento comparativa del flujo de trabajo que ejecute un bot de solo manifiesto junto con el modelo bajo la misma configuración de CI haría medible la ganancia a nivel de flujo de trabajo. Quinto, reemplazar el sistema alternativo de CodeBLEU basado en expresiones regulares por un analizador completo de tree-sitter eliminaría la advertencia de "aproximado" de la evaluación actual. Finalmente, integrar este enfoque en flujos de producción reales de CI y realizar un estudio con profesionales aportaría información sobre su utilidad y aceptación.

---

## 6. Amenazas a la Validez

### 6.1 Validez interna

**Granularidad a nivel de commit:** Algunos cambios de código Java en commits que modificaron dependencias de JUnit podrían no estar causalmente relacionados con la actualización de la dependencia, lo que introduce ruido. Mitigamos esto enfocándonos en importaciones, anotaciones y métodos de prueba con mayor probabilidad de estar vinculados a JUnit.

**División de datos:** Los 8,735 ejemplos a nivel de fragmento se dividen aleatoriamente (80/20) a nivel de ejemplo utilizando una semilla fija, en lugar de hacerlo a nivel de repositorio. Aunque el paso de deduplicación (umbral de similitud <0.95) reduce el riesgo de que ejemplos casi idénticos abarquen ambas particiones, no se puede descartar la filtración de información a través de estilos de codificación compartidos o convenciones del proyecto.

**Longitud de secuencia:** Todos los ejemplos a nivel de fragmento caben dentro del límite del codificador de ambos modelos. A nivel de clase, las secuencias superan de forma rutinaria este límite, y el truncamiento resultante es una causa primaria del fallo del modelo (CodeBLEU = 0.213, 100% de errores por truncamiento). Por lo tanto, la evaluación a nivel de clase se presenta como un control negativo.

**Tamaño de la muestra de evaluación:** Debido a restricciones computacionales, la evaluación del modelo utiliza $N=200$ de los 1,747 ejemplos de prueba reservados.

### 6.2 Validez externa

**Alcance de las dependencias:** Nuestro conjunto de datos se centra exclusivamente en JUnit 3.x y 4.x; no hay versiones de JUnit 5 presentes. Aplicar el mismo enfoque a otros lenguajes, marcos de prueba o gestores de dependencias requeriría una nueva recopilación de datos, preprocesamiento y evaluación.

**Muestra de repositorios:** Los 43 repositorios se seleccionaron utilizando criterios basados en la popularidad (estrellas, bifurcaciones, colaboradores). Los proyectos con diferentes modelos de gobernanza, tamaños o prácticas de prueba pueden exhibir patrones de coevolución distintos.

### 6.3 Validez de constructo

**Métricas de evaluación:** BLEU y ROUGE capturan la similitud superficial pero no garantizan la corrección semántica ni la ejecutabilidad del código. CodeBLEU añade componentes de coincidencia sintáctica y de flujo de datos; sin embargo, nuestra implementación utiliza una alternativa basada en expresiones regulares en lugar de un analizador completo de tree-sitter.

**Filtrado de similitud:** La deduplicación utiliza la distancia de Levenshtein a nivel de caracteres, una medida léxica que no captura la redundancia estructural. La similitud basada en AST o la similitud de Jaccard podrían proporcionar garantías de diversidad más sólidas.

**Extracción de EM-Java:** La métrica EM-Java depende de una heurística basada en límites para extraer la sección Java de las salidas del modelo. Aunque se verificó mediante inspección manual, los casos límite pueden introducir ruido en un número pequeño de comparaciones.

---

## 7. Conclusiones

Este artículo investigó cómo afectan las actualizaciones de dependencias de JUnit al código de prueba en proyectos Java basados en Maven y si estos co-cambios pueden aprenderse a partir de datos históricos. A partir de un corpus de 2,223 commits de actualización reales, construimos un conjunto de datos filtrado a nivel de fragmento de 8,735 pares de entrada-salida —publicado abiertamente para apoyar la reproducibilidad y la investigación futura— y lo analizamos a través de cuatro preguntas de investigación que cubren la fidelidad del corpus (RQ1), la magnitud de la actualización a través de las categorías de versionado semántico (RQ2), la influencia del preprocesamiento y el diseño del conjunto de datos en la capacidad de aprendizaje (RQ3), y el posicionamiento relativo frente a las herramientas de actualización de dependencias existentes (RQ4).

Las actualizaciones de JUnit 3.x/4.x son predominantemente conservadoras. A nivel de método, el 78.8% de los pares anteriores/posteriores a la actualización difieren en menos del 8% de su contenido de caracteres, con una mediana de la distancia de Levenshtein normalizada de 0.02. Además, este conservadurismo es estable entre los lanzamientos de parches, menores y mayores: las distancias de edición promedio caen dentro de una franja estrecha (0.466-0.486) y las medias de CodeBLEU son igualmente cercanas (0.470-0.489). Esto confirma que la etiqueta de versionado semántico es un predictor débil de cuánto cambia realmente el código de prueba.

Ambos modelos ajustados superan sustancialmente a la línea base de solo manifiesto en la división reservada a nivel de fragmento. En particular, CodeT5-base alcanza un EM-Java = 38.5% y un CodeBLEU = 0.854, mientras que CodeT5+ alcanza un EM-Java = 43.6% y un CodeBLEU = 0.868, en comparación con el 0% y 0.502 de la línea base. La mejora es mayor en el nivel de cambios significativos (+0.528 CodeBLEU). Además, la validación funcional condicional a través de BTR muestra que los parches generados por el modelo compilan y superan las pruebas en los 37 casos evaluables (5 de los 21 repositorios muestreados), mientras que los cambios de solo manifiesto fallan en el 50% de los casos de control.

No obstante, varias limitaciones matizan estos hallazgos. El corpus cubre únicamente JUnit 3.x/4.x, la implementación de CodeBLEU utiliza un sistema alternativo basado en expresiones regulares, y la evaluación reservada utiliza $N=200$ muestras. Las direcciones más inmediatas para el trabajo futuro son extender el conjunto de datos a JUnit 5 y otros marcos de trabajo, ampliar la cobertura de BTR más allá de los 5 repositorios actuales, reemplazar el sistema alternativo de expresiones regulares por tree-sitter y realizar una prueba de rendimiento comparativa del flujo de trabajo frente a los bots de solo manifiesto.

Independientemente de estas extensiones, la caracterización empírica de los patrones de coevolución de JUnit y el conjunto de datos a nivel de fragmento publicado abiertamente constituyen recursos reutilizables para la comunidad científica, aplicables a futuros modelos, herramientas y estudios empíricos sobre la gestión de dependencias más allá de la arquitectura específica evaluada aquí.