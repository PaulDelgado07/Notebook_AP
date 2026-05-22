# 8. Propagación hacia adelante (Forward Propagation)

## 8.1 Introducción
Una vez que conocemos qué es una neurona artificial, qué son los pesos, el bias, las funciones de activación y cómo se organiza la arquitectura de una red, el siguiente paso natural es entender cómo circula la información dentro del modelo.

Esa circulación de datos desde la entrada hasta la salida se llama propagación hacia adelante, o en inglés, *forward propagation*.

Este proceso es fundamental porque describe cómo una red neuronal produce una predicción. Antes de poder aprender, la red primero necesita calcular una salida. Y ese cálculo ocurre justamente en la propagación hacia adelante.

## 8.2 ¿Qué es la propagación hacia adelante?
La propagación hacia adelante es el proceso mediante el cual los datos de entrada atraviesan las distintas capas de la red neuronal hasta llegar a la salida.

En este recorrido, cada neurona:
* Recibe valores de entrada.
* Calcula una combinación ponderada.
* Aplica una función de activación.
* Entrega una salida a la capa siguiente.

Este flujo continúa capa tras capa hasta generar la predicción final del modelo.

## 8.3 Por qué es importante
La propagación hacia adelante es importante porque representa el comportamiento operativo de la red. Es el momento en que el modelo "piensa" usando sus parámetros actuales.

Sin este paso:
* No habría predicciones.
* No podría calcularse el error.
* No existiría base para el aprendizaje posterior.

Por eso el entrenamiento de una red siempre comienza, en cada ejemplo, con una propagación hacia adelante.

## 8.4 La dirección del flujo
El nombre "hacia adelante" viene de la dirección en que se mueve la información:
1. Primero desde la capa de entrada,
2. luego por las capas ocultas,
3. y finalmente hacia la capa de salida.

Es decir, la información avanza desde los datos iniciales hasta la respuesta final. No hay corrección de errores todavía; en esta etapa solo se calcula una salida.

## 8.5 Qué ocurre en una neurona durante el forward
En cada neurona ocurre el mismo patrón básico:
1. Se reciben una o varias entradas.
2. Cada entrada se multiplica por su peso correspondiente.
3. Se suman todos los productos.
4. Se agrega el bias.
5. Se aplica una función de activación.
6. Se produce una salida.

Ese valor de salida se convierte luego en entrada para otras neuronas de la capa siguiente.

## 8.6 Fórmula básica del cálculo
En forma matemática, el cálculo de una neurona puede expresarse así:

$$z = (x_1 \cdot w_1) + (x_2 \cdot w_2) + \dots + b$$
$$a = 	ext{activacion}(z)$$

Donde:
* $x$ son las entradas,
* $w$ son los pesos,
* $b$ es el bias,
* $z$ es la suma ponderada,
* $a$ es la salida activada.

Este patrón se repite en cada neurona y en cada capa de la red.

## 8.7 Desde una neurona hasta una capa completa
Cuando trabajamos con una capa completa, no se calcula una sola neurona, sino muchas al mismo tiempo. Cada neurona de la capa recibe las salidas de la capa anterior y produce su propia activación.

Por eso, la salida de una capa no es un único número, sino un conjunto de valores que representan lo que esa capa ha aprendido a detectar o transformar. Esos valores se convierten en la entrada de la capa siguiente.

## 8.8 El papel de la capa de entrada
La capa de entrada es el punto de partida del *forward propagation*. Allí se colocan los datos originales del problema.

Si el problema es tabular, las entradas pueden ser variables como edad, ingresos o antigüedad. Si el problema es una imagen, las entradas pueden ser los valores numéricos de los píxeles. Si es texto, pueden ser representaciones vectoriales de palabras o tokens.

La capa de entrada no transforma mucho por sí misma. Su función es alimentar el resto de la red.

## 8.9 El papel de las capas ocultas
En las capas ocultas ocurre el verdadero procesamiento interno. Cada una toma la salida de la capa anterior y la transforma en una nueva representación.

Podemos pensar que cada capa oculta realiza una especie de refinamiento de la información:
* Extrae patrones relevantes.
* Combina señales.
* Genera una representación más útil para la tarea.

Cuantas más capas haya, más etapas de transformación ocurren durante la propagación hacia adelante.

## 8.10 El papel de la capa de salida
La capa de salida es el destino final del *forward propagation*. Allí se produce la predicción de la red.

Esa salida puede tomar distintas formas:
* Un número real en problemas de regresión.
* Una probabilidad en clasificación binaria.
* Un conjunto de probabilidades en clasificación multiclase.

La forma exacta depende del problema y de la función de activación elegida para la salida.

## 8.11 Un ejemplo sencillo con una sola neurona
Supongamos una neurona que recibe dos entradas:
* $x_1 = 2$
* $x_2 = 3$

Y tiene los siguientes parámetros:
* $w_1 = 0.5$
* $w_2 = 1.0$
* $b = -1$

Entonces el cálculo sería:
$$z = (2 \cdot 0.5) + (3 \cdot 1.0) - 1$$
$$z = 1 + 3 - 1 = 3$$

Luego se aplica la función de activación. Si fuera ReLU, la salida sería 3. Si fuera una función escalón, la salida dependería del umbral. Si fuera Sigmoid, la salida sería un valor entre 0 y 1.

## 8.12 Un ejemplo con una capa oculta
Imaginemos ahora una red pequeña con:
* 2 entradas
* 1 capa oculta con 2 neuronas
* 1 capa de salida con 1 neurona

El *forward propagation* ocurriría así:
1. Las dos entradas llegan a las neuronas de la capa oculta.
2. Cada neurona oculta calcula su suma ponderada y su activación.
3. Las salidas de esas dos neuronas pasan a la capa de salida.
4. La neurona de salida hace su propio cálculo y produce la predicción final.

Esto muestra que el *forward propagation* es, esencialmente, una secuencia de transformaciones matemáticas encadenadas.

## 8.13 Qué significa que cada capa transforme la información
Decir que una capa transforma la información significa que toma un conjunto de valores y lo convierte en otro conjunto de valores más útil para la tarea.

Por ejemplo, en una red para imágenes:
* La entrada puede ser solo una matriz de píxeles.
* La primera capa puede detectar bordes.
* La siguiente puede detectar formas simples.
* Otra puede detectar partes de objetos.
* La salida final puede clasificar la imagen.

Todo eso ocurre durante la propagación hacia adelante.

## 8.14 Forward propagation y predicción
Cuando usamos una red neuronal ya entrenada para hacer predicciones, lo que realmente hacemos es ejecutar una propagación hacia adelante.

Es decir:
* Ingresamos nuevos datos.
* La red los procesa con sus pesos ya aprendidos.
* Produce una salida.

En ese contexto ya no se están ajustando parámetros. Solo se está utilizando el conocimiento que la red ya adquirió durante el entrenamiento.

## 8.15 Forward propagation durante el entrenamiento
Durante el entrenamiento también ocurre la propagación hacia adelante. La diferencia es que, después de obtener la salida, esa predicción se compara con la respuesta correcta.

Entonces el proceso completo durante el entrenamiento es:
1. *Forward propagation* para obtener una predicción.
2. Cálculo del error o pérdida.
3. *Backward propagation* para ajustar los parámetros.

Por eso el forward es el primer paso del ciclo de aprendizaje.

## 8.16 Relación con la función de pérdida
La salida del *forward propagation* no es el final del proceso cuando estamos entrenando. Esa salida se usa para calcular qué tan bien o qué tan mal se comportó el modelo. Esa medición se hace mediante la función de pérdida.

Si la predicción está lejos del valor correcto, la pérdida será grande. Si está cerca, será pequeña.

Más adelante veremos esto con detalle, pero es importante entender desde ahora que el *forward propagation* produce la predicción sobre la cual luego se mide el error.

## 8.17 Forward propagation y activaciones
Las funciones de activación juegan un papel central durante la propagación hacia adelante. No son un detalle extra: forman parte del propio cálculo que se realiza en cada capa.

Sin activaciones, cada capa haría una transformación lineal, y la red perdería gran parte de su capacidad para aprender patrones complejos.

Por eso, cuando pensamos en el *forward propagation*, debemos imaginar no solo sumas ponderadas, sino también activaciones que modelan relaciones no lineales.

## 8.18 Forward propagation y matrices
En la práctica, sobre todo cuando se trabaja con muchas neuronas y muchos ejemplos al mismo tiempo, no se hacen los cálculos uno por uno manualmente. Se usan operaciones matriciales.

Esto permite aprovechar mejor el hardware y calcular muchas neuronas de forma eficiente.

Cuando más adelante programemos en PyTorch, veremos que gran parte del trabajo se basa en tensores y multiplicaciones matriciales. Ese formalismo es simplemente una manera eficiente de implementar la propagación hacia adelante.

## 8.19 Un ejemplo intuitivo con clasificación binaria
Supongamos que tenemos una red que debe predecir si un correo es spam o no spam.

Durante la propagación hacia adelante:
* La red recibe variables como cantidad de enlaces, palabras sospechosas, longitud del mensaje, etc.
* Las capas ocultas combinan esas señales.
* La capa final produce una probabilidad.
* Si la probabilidad es alta, el correo se clasifica como spam.

Todo ese cálculo interno, desde la entrada hasta la probabilidad final, es *forward propagation*.

## 8.20 Un ejemplo intuitivo con imágenes
Imaginemos una red que debe clasificar si una imagen contiene un gato o un perro.

Durante la propagación hacia adelante:
* La imagen entra como una gran cantidad de valores numéricos.
* Las primeras capas detectan patrones básicos.
* Las capas intermedias combinan esos patrones en estructuras más complejas.
* Las capas finales producen la probabilidad de cada clase.

El usuario solo ve el resultado final, pero internamente han ocurrido muchas transformaciones sucesivas.

## 8.21 Qué NO hace el forward propagation
Es importante no confundir la propagación hacia adelante con el aprendizaje completo. Durante el *forward propagation*:
* **No** se actualizan pesos.
* **No** se corrigen errores.
* **No** se optimiza el modelo.

Todo eso ocurre después, cuando se calcula la pérdida y se aplica la retropropagación junto con el descenso del gradiente. El *forward* solo produce una salida usando los parámetros actuales.

## 8.22 Por qué el forward se repite tantas veces
Durante el entrenamiento, la red procesa muchos ejemplos y lo hace repetidas veces a lo largo de múltiples épocas. Eso significa que el *forward propagation* se ejecuta una enorme cantidad de veces.

Cada vez que la red ve un *batch* de datos:
1. Realiza *forward propagation*.
2. Calcula la pérdida.
3. Actualiza parámetros.

Por eso la eficiencia del *forward propagation* también es importante desde el punto de vista computacional.

## 8.23 Forward propagation y generalización
La forma en que la red realiza la propagación hacia adelante determina la calidad de sus predicciones. Si los pesos están bien aprendidos, el *forward propagation* producirá salidas útiles incluso en datos nuevos.

En ese sentido, la generalización de la red se manifiesta justamente en la calidad de sus predicciones durante el forward con ejemplos no vistos.

## 8.24 Resumen del proceso completo
Podemos resumir la propagación hacia adelante así:
1. Ingresan los datos en la capa de entrada.
2. Cada capa realiza sumas ponderadas y aplica activaciones.
3. Las salidas de una capa se convierten en entradas de la siguiente.
4. La capa final produce la predicción del modelo.

Ese flujo es el corazón operativo de una red neuronal.

## 8.25 Relación con PyTorch
En PyTorch, la propagación hacia adelante suele definirse dentro del método `forward` de un modelo. Allí se especifica exactamente cómo pasan los datos por las capas.

Cuando más adelante implementemos redes en código, verás que el método `forward` representa justamente el recorrido que aquí estamos estudiando conceptualmente. Entender este tema te dará una base muy sólida para interpretar el código de un modelo y no solo escribirlo de forma mecánica.

## 8.26 Qué debes recordar de este tema
* La propagación hacia adelante es el proceso por el cual la información va desde la entrada hasta la salida.
* En cada neurona se calcula una suma ponderada y luego se aplica una activación.
* Las capas ocultas transforman progresivamente la información.
* La capa de salida produce la predicción final.
* El *forward propagation* ocurre tanto durante la inferencia como durante el entrenamiento.
* Durante el *forward* no se actualizan parámetros.
* La salida del *forward* se usa luego para calcular la pérdida.
* En PyTorch, este proceso suele escribirse en el método `forward`.

## 8.27 Conclusión
La propagación hacia adelante describe cómo una red neuronal convierte datos de entrada en una predicción. Es el proceso que pone en funcionamiento toda la arquitectura del modelo.

Entender el *forward propagation* es esencial porque permite ver a la red como una secuencia organizada de cálculos y no como una caja negra. El siguiente paso natural será estudiar cómo se mide la calidad de esa predicción, es decir, la función de pérdida.
