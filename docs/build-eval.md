# Construyendo una evaluación

Este documento recorre el proceso de extremo a extremo para crear una evaluación, que es un conjunto de datos y una elección de clase de evaluación. La carpeta `examples` contiene cuadernos Jupyter que siguen los pasos a continuación para crear varias evaluaciones académicas, lo que ayuda a ilustrar el proceso general.

Los pasos de este proceso son crear su conjunto de datos, registrar una nueva evaluación con su conjunto de datos y ejecutar su evaluación. Fundamentalmente, asumimos que está utilizando una [plantilla de evaluación existente](eval-templates.md) lista para usar (si ese no es el caso, consulte [este ejemplo de creación de una evaluación personalizada](custom-eval.md)) . Si está interesado en contribuir con su evaluación públicamente, también incluimos algunos criterios en la parte inferior de lo que creemos que hace una evaluación interesante.

Estamos buscando evaluadores en las siguientes categorías:

- Rechazos excesivos
- Seguridad
- Orientación de mensajes del sistema
- Alucinaciones en estado salvaje
- Razonamiento matemático/lógico/físico
- Caso de uso del mundo real (describa en su PR cómo se usaría esta capacidad en un producto)
- Otra capacidad fundamental

Si tiene una evaluación que cae fuera de esta categoría pero que aún es un ejemplo diverso, ¡contribuya con ella!

## Formatear tus datos

Una vez que tenga en mente una evaluación que desea implementar, deberá convertir sus muestras al formato de líneas JSON correcto (JSONL). Un archivo JSONL es solo un archivo JSON con un objeto JSON único por línea.

Incluimos algunos ejemplos de archivos de evaluación JSONL en [registry/data/README.md](../evals/registry/data/README.md)

Cada objeto JSON representará un punto de datos en su evaluación. Las claves que necesita en el objeto JSON dependen de la plantilla de evaluación. Todas las plantillas esperan una tecla `"input"` que es el aviso, idealmente especificado en [formato de chat](https://platform.openai.com/docs/guides/chat/introduction) (aunque también se admiten cadenas). Recomendamos el formato de chat incluso si está evaluando modelos que no son de chat. Si está evaluando modelos de chat y no chat, manejamos la conversión entre avisos con formato de chat y avisos de cadena sin procesar (consulte la lógica de conversión [aquí] (../evals/prompt/base.py)).

Para las evaluaciones básicas `Match`, `Includes` y `FuzzyMatch`, la otra clave requerida es `"ideal"`, que es una cadena (o una lista de cadenas) que especifica la(s) respuesta(s) de referencia correcta. Para las evaluaciones graduadas por modelos, las claves requeridas varían según la evaluación, pero están determinadas por las `{claves}` en el `mensaje de evaluación` que no están cubiertas por los `argumentos` (opcionales).

Hemos implementado pequeños subconjuntos del conjunto de datos [CoQA](https://stanfordnlp.github.io/coqa/) para varias plantillas de evaluación para ilustrar cómo deben formatearse los datos. Consulte [`coqa/match.jsonl`](../evals/registry/data/coqa/match.jsonl) para ver un ejemplo de datos que son adecuados para la plantilla de evaluación básica `Match` y [`coqa/samples.jsonl `](../evals/registry/data/coqa/samples.jsonl) para los datos que son adecuados para las evaluaciones con calificación de modelo `fact` y `closedqa`. Tenga en cuenta que aunque estas dos evaluaciones graduadas por modelos esperan claves diferentes, podemos incluir el superconjunto de claves en nuestros datos para admitir ambas evaluaciones.

Si el archivo del conjunto de datos está en su máquina local, coloque el archivo `jsonl` en `evals/registry/evals/data/<eval_name>/samples.jsonl`. Si está en Cloud Object Storage, admitimos URL de estilo de ruta para las principales nubes (solo para su uso personal, no aceptaremos PR con URL de nube).

## Registrando la evaluación

Registre la evaluación agregando un archivo a `evals/registry/evals/<eval_name>.yaml` usando el formato de registro de elsuite. Por ejemplo, para una evaluación `Coincidencia`, sería:
```
<eval_name>:
  id: <eval_name>.dev.v0
  metrics: [accuracy]

<eval_name>.dev.v0:
  class: evals.elsuite.basic.match:Match
  args:
    samples_jsonl: <eval_name>/samples.jsonl
```

Al ejecutar la evaluación, los datos se buscarán en `evals/registry/data`, p. si `test_match/samples.jsonl` es la ruta del archivo proporcionada, se espera que los datos estén en `evals/registry/data/test_match/samples.jsonl`.

La convención de nomenclatura para evals tiene el formato `<eval_name>.<split>.<version>`.
- `<eval_name>` es el nombre de evaluación, usado para agrupar evaluaciones cuyas puntuaciones son comparables.
- `<split>` es la división de datos, utilizada para agrupar aún más las evaluaciones que están bajo el mismo `<base_eval>`. Por ejemplo, "val", "test" o "dev" para probar.
- `<version>` es la versión de la evaluación, que puede ser cualquier texto descriptivo que le gustaría usar (aunque es mejor si no contiene ".").

En general, ejecutar el mismo nombre de evaluación en el mismo modelo siempre debería dar resultados similares para que otros puedan reproducirlo. Por lo tanto, cuando cambie su evaluación, debe modificar la versión.

## Ejecutando la evaluación

Ahora puede ejecutar su evaluación en sus datos desde la CLI con su elección de modelo:
```
oaieval gpt-3.5-turbo <eval_name>
```
¡Felicitaciones, ha construido su evaluación! Siga iterando hasta que esté seguro de los resultados.

## Para evaluaciones graduadas por modelos: un flujo de trabajo paso a paso

Esperamos que las evaluaciones evaluadas por modelos existentes, como `fact`, `closedqa` y `battle`, se adapten a muchos casos de uso. Sin embargo, otros casos de uso pueden beneficiarse de una mayor personalización, por ejemplo, un indicador de evaluación diferente. Para estos, habrá un poco más de trabajo involucrado, ¡pero en general aún no se requiere codificación!

1. Si no puede usar una evaluación evaluada por modelo existente, cree un nuevo YAML en `evals/registry/modelgraded` para especificar los [parámetros](eval-templates.md#parameters-for-model-graded-evals) de su evaluación. Consulta [`humor.yaml`](../evals/registry/modelgraded/humor.yaml) para ver un ejemplo.
     - Tenga en cuenta que, incluso si está creando un nuevo YAML, puede que le resulte más fácil copiar un YAML existente como punto de partida. Por ejemplo, las evaluaciones calificadas por modelos que comparan la finalización de un modelo con una rúbrica pueden copiar `closedqa.yaml` y simplemente editar los `args`.
2. A continuación, creará su conjunto de datos y registrará su evaluación, como se describe anteriormente. Consulte [`joke_fruits_labeled.jsonl`](../evals/registry/data/test_metaeval/joke_fruits_labeled.jsonl) y [`joke-fruits`](../evals/registry/evals/test-modelgraded.yaml), para ejemplo.
     - Tenga en cuenta que se recomienda especificar `eval_type` en este paso, cuando registre su evaluación, en lugar del paso 1.
3. Ejecute su evaluación, por ejemplo, `oaleval gpt-3.5-turbo joke-fruits`.
4. (Recomendado) ¡Agregue una metaevaluación para la evaluación calificada por modelo! Cada evaluación graduada por modelo viene con algunas perillas para ajustar, principalmente `prompt` pero también `eval_type`. Para asegurarnos de que la evaluación sea de alta calidad, recomendamos que cada contribución de evaluación calificada por modelo venga con "etiquetas de elección", que son básicamente etiquetas proporcionadas por humanos para la elección de evaluación que debería haber hecho el modelo. Como ejemplo (simulando que estos chistes son realmente graciosos), vea las claves `"choice"` en [`joke_fruits_labeled.jsonl`](../evals/registry/data/test_metaeval/joke_fruits_labeled.jsonl), que no se usan por la evaluación `joke-fruits` pero son utilizados por la meta-evaluación [`joke-fruits-meta`](../evals/registry/evals/test-modelgraded.yaml) justo debajo de ella. Después de ejecutar la metaevaluación, por ejemplo, `oaieval gpt-3.5-turbo broma-frutas-meta`, el informe generará precisiones `metascore/`, que deben estar cerca de "1.0" para una buena evaluación calificada por el modelo.

## Criterios para contribuir con una evaluación

Importante: si está contribuyendo con código, asegúrese de ejecutar `pip install pre-commit; precommit install` antes de confirmar y presionar para garantizar que se ejecuten `black`, `isort` y `autoflake`.

Estamos interesados en seleccionar un conjunto diverso e interesante de evaluaciones para mejorar nuestros modelos en el futuro. Aquí hay algunos criterios para lo que consideramos una buena evaluación.
- [ ] La evaluación debe ser temáticamente coherente. Nos gustaría ver una serie de avisos que giran en torno al mismo caso de uso, dominio del tema, modo de falla, etc.
- [ ] La evaluación debe ser desafiante. Si GPT-4 o GPT-3.5-Turbo funcionan bien en todas las indicaciones, esto no es tan interesante. Por supuesto, la evaluación también debería ser posible dadas las limitaciones y restricciones de los modelos. A menudo, una buena regla general es si un ser humano (potencialmente un experto en la materia) podría hacerlo bien con las indicaciones.
- [ ] La evaluación debe ser direccionalmente clara. Los datos deben incluir una buena señal sobre cuál es el comportamiento correcto. Esto significa, por ejemplo, respuestas de referencia de alta calidad o una rúbrica exhaustiva para evaluar las respuestas.
- [ ] La evaluación debe elaborarse cuidadosamente. Antes de enviar, debe pensar si ha diseñado sus indicaciones para un buen rendimiento, si está utilizando la mejor plantilla de evaluación, si ha verificado sus resultados para garantizar la precisión, etc.

Una vez que esté listo para contribuir con su evaluación públicamente, envíe un PR y el equipo de OpenAI estará encantado de revisarlo. Asegúrese de completar todas las partes de la plantilla que se completa previamente en el mensaje de relaciones públicas. Tenga en cuenta que enviar un PR no garantiza que OpenAI eventualmente lo fusionará. Haremos nuestras propias verificaciones y usaremos nuestro mejor juicio al considerar qué evaluaciones seguir.
