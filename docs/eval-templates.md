# Plantillas existentes para evaluaciones

Al usar Evals, hemos descubierto varias "plantillas" que se adaptan a muchos puntos de referencia diferentes. Hemos implementado estas plantillas en `evals/elsuite` para simplificar el desarrollo de nuevas evaluaciones. ¡Creemos que, con estas plantillas, muchas evaluaciones no requerirán ningún código para implementarlas! En su lugar, elegirá una de las plantillas existentes y simplemente especificará el conjunto de datos y los parámetros.

## Plantillas de evaluación básicas

En los casos en los que la respuesta del modelo deseado tiene muy poca variación, como responder preguntas de opción múltiple o preguntas simples con una respuesta directa, hemos encontrado que las siguientes plantillas son útiles.

Para completar un modelo `a` y una lista de referencia de respuestas correctas `B`, se implementan las siguientes evaluaciones:
- [`basic/match.py:Match`](../evals/elsuite/basic/match.py): `any([b.comienza con(a) para b en B])`
- [`basic/includes.py:Incluye`](../evals/elsuite/basic/includes.py): `any([(a en b) para b en B])`
- [`basic/fuzzy_match.py:FuzzyMatch`](../evals/elsuite/basic/fuzzy_match.py): `any([(a en b o b en a) para b en B])`

La plantilla de evaluación que utilice dependerá de su caso de uso. Siempre se recomienda que inspeccione las terminaciones de su modelo, ya que esto lo ayudará a determinar cómo y si ajustar su solicitud (o sus respuestas de referencia) y elegir su plantilla de evaluación. Los puntos de referencia académicos a menudo se ajustan al molde de estas evaluaciones básicas, y hemos implementado varios ejemplos completos de evaluaciones académicas como cuadernos Jupyter en la carpeta "ejemplos".

A veces, la [lógica de evaluación personalizada](custom-eval.md) se adaptará mejor a sus necesidades. Un ejemplo de esto es [traducción automática](../evals/elsuite/translate.py) [eval example](../examples/lafand-mt.ipynb), en el que hay una métrica única y claramente definida que deseamos utilizar en nuestra evaluación. Debe utilizar su mejor criterio al decidir entre la lógica de evaluación personalizada, el uso de una plantilla de evaluación básica o el uso de evaluaciones graduadas por modelos, como se describe a continuación.

## La plantilla de evaluación calificada por el modelo

En los casos en que la respuesta del modelo deseado puede contener una variación significativa, como responder una pregunta abierta, hemos descubierto que usar el modelo para calificarse a sí mismo es una estrategia viable para la evaluación automatizada. En general, el modelo de evaluación y el modelo que se evalúa no tienen que ser iguales, aunque supondremos que están aquí para facilitar la explicación.
[`modelgraded/classify.py:ModelBasedClassify`](../evals/elsuite/modelgraded/classify.py) implementa la lógica principal detrás de nuestra plantilla de evaluación calificada por modelo. En resumen, obtenemos la finalización del modelo en el mensaje original, lo envolvemos en un mensaje de evaluación y obtenemos la finalización del modelo en el mensaje de evaluación, que analizamos en nuestras métricas de interés. Fundamentalmente, el indicador de evaluación debe preparar el modelo para responder de tal manera que sea fácilmente analizable, por ejemplo, en formato de opción múltiple o con un simple sí/no. Describimos algunos ejemplos de evaluaciones calificadas por modelos a continuación, pero primero especificamos los parámetros para esta plantilla de evaluación.
### Parámetros para evaluaciones graduadas por modelo

Consulte la clase [`classify.py:ModelBasedClassify`](../evals/elsuite/modelgraded/classify.py) para ver cómo se usan estos parámetros en el código.

- `mensaje`: El mensaje de evaluación que debe incluir la finalización del modelo en el mensaje original, potencialmente junto con alguna otra información, y dirigir el modelo para proporcionar una evaluación que sea fácilmente analizable. Las partes indicadas con llaves (es decir, `{clave}`) se completan con los datos `input_outputs` o `args` adicionales (ver más abajo).
- `input_outputs`: una asignación que especifica qué entradas usar para generar qué finalizaciones. Para muchas evaluaciones, solo habrá un solo par de entrada-finalización, aunque puede haber más, por ejemplo, cuando se comparan dos finalizaciones entre sí.
- `choice_strings`: Las opciones que esperamos que contenga la finalización del modelo dada la solicitud de evaluación. Por ejemplo, `"ABCDE"` o `["Sí", "No", "No estoy seguro"]`. Cualquier otra opción devuelta por el modelo se analiza en `"__invalid__"`.
- `choice_scores` (opcional): una asignación de cada opción a su puntuación, que se registra como una métrica. Por ejemplo, si una respuesta `"Sí"` (resp. `"No"`) indica que la finalización original del modelo fue buena (resp. mala), podemos asignar a esta opción una puntuación de 1 (resp. 0).
- `eval_type` (opcional): cómo esperamos que el modelo formatee su respuesta al aviso de evaluación. Actualmente las opciones soportadas son:
   - `"cot_classify"` ("cadena de pensamientos y luego clasificar", es decir, razonar y luego responder) espera que la parte analizable de la respuesta (es decir, la parte que contiene la elección) esté al final de la finalización. Recomendamos esto como valor predeterminado, ya que normalmente proporciona las evaluaciones graduadas del modelo más precisas.
   - `"classify_cot"` (respuesta y luego motivo) espera que la respuesta del modelo contenga primero la elección.
   - `"clasificar"` espera que la respuesta del modelo solo contenga la opción.

   Hay dos formas de especificar `eval_type`. La forma recomendada es en el archivo YAML `evals/registry/evals`. Si se hace de esta manera, se agregará automáticamente una instrucción a `prompt` para dirigir el modelo hacia el formato esperado (ver `ANSWER_PROMPTS` en [el código](../evals/elsuite/modelgraded/classify.py)). Alternativamente, puede especificar `eval_type` en `evals/registry/modelgraded` YAML, pero deberá incluir una instrucción apropiada directamente en el `prompt`.
- `args` (opcional): si se especifica, se realizarán varias llamadas de evaluación en las que se modificará la solicitud de evaluación para cada llamada con un conjunto diferente de argumentos.
- `completion_sample_templates` (opcional): si se especifica, determina cómo se formateará la salida del modelo (o las salidas, si `multicomp_n > 1`) dentro de la finalización.
### Ejemplos de evaluaciones calificadas por el modelo

Para crear instancias de evaluaciones calificadas por modelos, cree un archivo YAML en `evals/registry/modelgraded` que especifica valores para los argumentos descritos anteriormente. Hemos proporcionado algunos ejemplos, que ilustran el proceso para crear una evaluación graduada por modelo, pero que también creemos que son lo suficientemente generales como para ser útiles para muchas evaluaciones.

[`fact.yaml`](../evals/registry/modelgraded/fact.yaml): una evaluación de coherencia fáctica que, dada una finalización `a` y una respuesta de referencia `b`, devuelve:
- `"A"` si `a` $\subseteq$ `b`, es decir, la respuesta enviada es un subconjunto de la respuesta del experto y es totalmente coherente con ella.
- `"B"` si `a` $\supseteq$ `b`, es decir, la respuesta enviada es un superconjunto de la respuesta del experto y es totalmente consistente con ella.
- `"C"` si `a` $=$ `b`, es decir, la respuesta enviada contiene todos los mismos detalles que la respuesta del experto.
- `"D"` si `a` $\neq$ `b`, es decir, hay un desacuerdo entre la respuesta enviada y la respuesta del experto.
- `"E"` si `a` $\approx$ `b`, es decir, las respuestas difieren, pero estas diferencias no importan desde la perspectiva de la factualidad.

[`closedqa.yaml`](../evals/registry/modelgraded/closedqa.yaml): una evaluación de respuesta a preguntas que, dada una indicación que contiene una pregunta y la información necesaria para responderla, verifica si la respuesta del modelo es:
- relevante, es decir, extraído de la información proporcionada en el mensaje,
- conciso, es decir, no contiene detalles o información innecesaria, y
- correcto, es decir, utiliza la información extraída para llegar a la conclusión correcta.

Tenga en cuenta que esta evaluación se implementa de manera más general como una evaluación de "verificación de criterios" que especifica que la solicitud de evaluación verifica un criterio dado y alimenta los deseos anteriores uno por uno. Creemos que se pueden implementar muchas otras evaluaciones especificando una "rúbrica" que detalle los criterios de interés y siguiendo el mismo mensaje y las opciones de sí/no.

[`battle.yaml`](../evals/registry/modelgraded/battle.yaml): una evaluación directa que compara las finalizaciones de dos modelos para dos indicaciones potencialmente diferentes. `choice_scores` se utiliza aquí para registrar con qué frecuencia se considera que la primera finalización es mejor que la segunda.

Incluimos ejemplos adicionales que prueban capacidades de modelos más específicas (como el humor) y, por lo tanto, son menos generalizables a otras evaluaciones. Sin embargo, estos ejemplos siguen sirviendo para ilustrar diferentes formas de escribir indicaciones de evaluación y configurar evaluaciones calificadas por modelos. Consulte [esta sección](build-eval.md#for-model-graded-evals-a-step-by-step-workflow) para obtener pasos más detallados sobre la creación de evaluaciones con calificación de modelo.
