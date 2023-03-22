# Cómo agregar una evaluación personalizada

Este tutorial lo guiará a través de un ejemplo simple de cómo escribir y agregar una evaluación personalizada. La evaluación de ejemplo pondrá a prueba la capacidad del modelo para hacer aritmética básica. Asumiremos que siguió las instrucciones de configuración en [README] (../README.md) y revisó los otros documentos sobre cómo ejecutar y compilar evaluaciones.

Al escribir sus propias evaluaciones, los principales archivos de interés son:
- `evals/api.py`, que proporciona interfaces y utilidades comunes utilizadas por los creadores de evaluaciones para tomar muestras de los modelos y procesar los resultados,
- `evals/record.py`, que define las clases de registradores que registran los resultados de evaluación de diferentes maneras, como en un archivo JSON local o en una base de datos remota de Snowflake, y
- `evals/metrics.py`, que define varias métricas comunes de interés.

Estos archivos proporcionan un conjunto de herramientas para escribir nuevas evaluaciones. Una vez que haya seguido este tutorial, podrá ver un ejemplo más realista de estas herramientas en acción con la [traducción automática](../evals/elsuite/translate.py) [eval example](../examples/lafand- mt.ipynb), que también implementa una lógica de evaluación personalizada en lugar de usar una plantilla existente.

## Crea tus conjuntos de datos

El primer paso es crear los conjuntos de datos para su evaluación. Aquí, crearemos trenes de juguete y juegos de prueba de solo dos ejemplos cada uno. Los ejemplos de prueba son en lo que evaluaremos el modelo, e incluiremos los ejemplos de trenes como ejemplos de pocos disparos en el indicador del modelo.

Usaremos el nuevo formato de chat descrito [aquí](https://platform.openai.com/docs/guides/chat/introduction). De manera predeterminada, recomendamos que todas las evaluaciones se escriban con formato de chat si desea evaluar nuestros nuevos modelos. Bajo el capó, [convertimos] (../evals/prompt/base.py) datos formateados de chat en cadenas sin procesar para modelos más antiguos que no son de chat.

Para crear los conjuntos de datos de juguetes, en su terminal, escriba:
```bash
echo -e '[{"función": "sistema", "contenido": "2+2=", "nombre": "usuario_ejemplo"}, {"función": "sistema", "contenido": "4" , "nombre": "example_assistant"}]\n[{"función": "sistema", "contenido": "4*4=", "nombre": "ejemplo_usuario"}, {"función": "sistema" , "contenido": "16", "nombre": "example_assistant"}]' > /tmp/train.jsonl
echo -e '[{"función": "sistema", "contenido": "48+2=", "nombre": "usuario_ejemplo"}, {"función": "sistema", "contenido": "50" , "nombre": "example_assistant"}]\n[{"función": "sistema", "contenido": "5*20=", "nombre": "ejemplo_usuario"}, {"función": "sistema" , "contenido": "100", "nombre": "example_assistant"}]' > /tmp/test.jsonl
```

## Crear una evaluación

El siguiente paso es escribir una clase de Python que represente la evaluación real. Esta clase usa sus conjuntos de datos para crear indicaciones, que se pasan al modelo para generar finalizaciones. Las clases de evaluación generalmente heredarán de la clase base `evals.Eval` (definida en `evals/eval.py`) y anularán dos métodos: `eval_sample` y `run`.

Vamos a crear un archivo llamado `arithmetic.py` en la carpeta `evals/elsuite`. Comenzaremos definiendo la clase eval. Su método `__init__` tomará los argumentos que necesitamos (referencias al tren y conjuntos de prueba) junto con otros `kwargs` que serán manejados por la clase base. También definiremos el método `run` que toma un `registrador` y devuelve las métricas finales de interés.
```python
import random
import textwrap

import evals
import evals.metrics

class Arithmetic(evals.Eval):
    def __init__(self, train_jsonl, test_jsonl, train_samples_per_prompt=2, **kwargs):
        super().__init__(**kwargs)
        self.train_jsonl = train_jsonl
        self.test_jsonl = test_jsonl
        self.train_samples_per_prompt = train_samples_per_prompt

    def run(self, recorder):
        """
        Called by the `oaieval` CLI to run the eval. The `eval_all_samples` method calls `eval_sample`.
        """
        self.train_samples = evals.get_jsonl(self.train_jsonl)
        test_samples = evals.get_jsonl(self.test_jsonl)
        self.eval_all_samples(recorder, test_samples)

        # Record overall metrics
        return {
            "accuracy": evals.metrics.get_accuracy(recorder.get_events("match")),
        }
```

En general, la mayoría de los métodos `run` seguirán el mismo patrón que se muestra aquí: cargar los datos, llamar a `eval_all_samples` y agregar los resultados (en este caso, usando la función `get_accuracy` en `evals/metrics.py`). `eval_all_samples` toma tanto `recorder` como `test_samples` y, bajo el capó, llamará al método `eval_sample` en cada muestra en `test_samples`. Así que escribamos ese método `eval_sample` ahora:

```
    python
    def eval_sample(self, test_sample, rng: random.Random):
        """
        Called by the `eval_all_samples` method to evaluate a single sample.

        ARGS
        ====
        `test_sample`: a line from the JSONL test file
        `rng`: should be used for any randomness that is needed during evaluation

        This method does the following:
        1. Generate a prompt that contains the task statement, a few examples, and the test question.
        2. Check if the model generates the correct answer.
        """
        stuffing = rng.sample(self.train_samples, self.train_samples_per_prompt)

        prompt = [
            {"role": "system", "content": "Solve the following math problems"},
        ]

        for i, sample in enumerate(stuffing + [test_sample]):
            if i < len(stuffing):
                prompt += [
                    {"role": "system", "content": sample["problem"], "name": "example_user"},
                    {"role": "system", "content": sample["answer"], "name": "example_assistant"},
                ]
            else:
                prompt += [{"role": "user", "content": sample["problem"]}]

        evals.check_sampled_text(self.model_spec, prompt, expected=sample["answer"])
```
Notarás que `eval_sample` no toma `recorder` como argumento. Esto se debe a que `eval_all_samples` lo establece como el grabador predeterminado antes de llamar a `eval_sample`, y las utilidades de grabación definidas en `evals/record.py` usan el grabador predeterminado. En este ejemplo, el método `eval_sample` transfiere gran parte del trabajo pesado a la función de utilidad `evals.check_sampled_text`, que se define en `evals/api.py`. Esta función de utilidad consulta el modelo, definido por `self.model_spec`, con el `prompt` dado y verifica si el resultado coincide con la respuesta `esperada` (o una de ellas, si se le da una lista). Luego registra estas coincidencias (o no coincidencias) usando la grabadora predeterminada.

Los métodos `eval_sample` pueden variar mucho según su caso de uso. Si está creando evaluaciones personalizadas, es una buena idea familiarizarse con las funciones disponibles en `evals/record.py`, `evals/metrics.py` y especialmente `evals/api.py`.

## Registre su evaluación

El siguiente paso es registrar su eval en el registro para que pueda ejecutarse utilizando la CLI `oaieval`.

Vamos a crear un archivo llamado `arithmetic.yaml` en la carpeta `evals/registry/evals` y agregar una entrada para nuestra evaluación de la siguiente manera:

```yaml
# Definir una evaluación base
aritmética:
   # id especifica la evaluación para la que esta evaluación es un alias
   # en este caso, aritmetic es un alias de arithmetic.dev.match-v1
   # Cuando ejecuta `oaieval davinci arithmetic`, en realidad está ejecutando `oaieval davinci arithmetic.dev.match-v1`
   id: aritmética.dev.match-v1
   # Las métricas que registra esta evaluación
   # La primera métrica se considerará la métrica principal
   métricas: [precisión]
   descripción: Evalúa la habilidad aritmética
# Definir la evaluación
aritmética.dev.match-v1:
   # Especifique el nombre de la clase como una ruta punteada al módulo y la clase
   clase: evals.elsuite.arithmetic:Aritmética
   # Especificar los argumentos como un diccionario de URI JSONL
   # Estos argumentos pueden ser cualquier cosa que quieras pasar al constructor de la clase
   argumentos:
     tren_jsonl: /tmp/tren.jsonl
     prueba_jsonl: /tmp/prueba.jsonl
```

El campo `args` debe coincidir con los argumentos que espera su método `__init__` de la clase eval.

## Ejecute su evaluación

El paso final es ejecutar su evaluación y ver los resultados.

```sh
pip install .  # you can omit this if you used `pip install -e .` to install
oaieval gpt-3.5-turbo arithmetic
```
Si ejecuta el modelo `gpt-3.5-turbo`, debería ver un resultado similar a este (hemos limpiado ligeramente el resultado aquí para facilitar la lectura):
```
% oaieval gpt-3.5-turbo arithmetic
... [registry.py:147] Loading registry from .../evals/registry/evals
... [registry.py:147] Loading registry from .../.evals/evals
... [oaieval.py:139] Run started: <run_id>
... [eval.py:32] Evaluating 2 samples
... [eval.py:138] Running in threaded mode with 1 threads!
100%|██████████████████████████████████████████| 2/2 [00:00<00:00,  3.35it/s]
... [record.py:320] Final report: {'accuracy': 1.0}. Logged to /tmp/evallogs/<run_id>_gpt-3.5-turbo_arithmetic.jsonl
... [oaieval.py:170] Final report:
... [oaieval.py:172] accuracy: 1.0
... [record.py:309] Logged 6 rows of events to /tmp/evallogs/<run_id>_gpt-3.5-turbo_arithmetic.jsonl: insert_time=2.038ms
```
