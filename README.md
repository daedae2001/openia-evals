# evaluaciones

Evals es un marco para evaluar modelos OpenAI y un registro de puntos de referencia de código abierto.

Puede usar Evals para crear y ejecutar evaluaciones que:
- usar conjuntos de datos para generar avisos,
- medir la calidad de las terminaciones proporcionadas por un modelo OpenAI, y
- comparar el rendimiento en diferentes conjuntos de datos y modelos.

Con Evals, nuestro objetivo es hacer que sea lo más simple posible construir una evaluación mientras se escribe la menor cantidad de código posible. Para comenzar, le recomendamos que siga estos pasos **en orden**:
1. Lea este documento y siga las [instrucciones de configuración a continuación] (README.md#Setup).
2. Aprenda a ejecutar evaluaciones existentes: [run-evals.md](docs/run-evals.md).
3. Familiarícese con las plantillas de evaluación existentes: [eval-templates.md](docs/eval-templates.md).
4. Siga el proceso para crear una evaluación: [build-eval.md](docs/build-eval.md)
5. Vea un ejemplo de implementación de lógica de evaluación personalizada: [custom-eval.md](docs/custom-eval.md).

Si cree que tiene una evaluación interesante, abra un PR con su contribución. El personal de OpenAI revisa activamente estas evaluaciones al considerar mejoras para los próximos modelos.

____________________
🚨 Por tiempo limitado, otorgaremos acceso a GPT-4 a quienes contribuyan con evaluaciones de alta calidad. Siga las instrucciones mencionadas anteriormente y tenga en cuenta que se ignorarán los envíos de spam o de baja calidad❗️

Se otorgará acceso a la dirección de correo electrónico asociada con una evaluación aceptada. Debido al gran volumen, no podemos otorgar acceso a ningún correo electrónico que no sea el utilizado para la solicitud de extracción.
____________________

## Configuración

Para ejecutar evaluaciones, deberá configurar y especificar su clave API de OpenAI. Puede generar uno en <https://platform.openai.com/account/api-keys>. Después de obtener una clave API, especifíquela usando la variable de entorno `OPENAI_API_KEY`. **Tenga en cuenta los [costos](https://openai.com/pricing) asociados con el uso de la API al ejecutar evaluaciones.**

**Versión mínima requerida: Python 3.9**

### Descargando evaluaciones

Nuestro registro Evals se almacena mediante [Git-LFS](https://git-lfs.com/). Una vez que haya descargado e instalado LFS, puede obtener las evaluaciones con:
```sh
git lfs buscar --todos
tirar de git lfs
```

Es posible que solo desee obtener datos para una evaluación seleccionada. Puede lograr esto a través de:
```sh
git lfs fetch --include=evals/registry/data/${tu evaluación}
tirar de git lfs
```

### Haciendo evaluaciones

Si va a crear evaluaciones, le sugerimos que clone este repositorio directamente desde GitHub e instale los requisitos con el siguiente comando:

```sh
pip instalar -e.
```

Usando `-e`, los cambios que realice en su evaluación se reflejarán inmediatamente sin tener que volver a instalar.

### Ejecutando evaluaciones

Si no desea contribuir con nuevas evaluaciones, sino simplemente ejecutarlas localmente, puede instalar el paquete de evaluaciones a través de pip:

```sh
evaluaciones de instalación de pip
```

Le brindamos la opción de registrar los resultados de su evaluación en una base de datos de Snowflake, si tiene una o desea configurar una. Para esta opción, deberá especificar además las variables de entorno `SNOWFLAKE_ACCOUNT`, `SNOWFLAKE_DATABASE`, `SNOWFLAKE_USERNAME` y `SNOWFLAKE_PASSWORD`.

## PREGUNTAS MÁS FRECUENTES

¿Tiene algún ejemplo de cómo construir una evaluación de principio a fin?

- ¡Sí! Estos están en la carpeta `examples`. Le recomendamos que también lea [build-eval.md](docs/build-eval.md) para obtener una comprensión más profunda de lo que sucede en estos ejemplos.

¿Tiene algún ejemplo de evaluaciones implementadas de múltiples maneras diferentes?

- ¡Sí! En particular, consulte `evals/registry/evals/coqa.yaml`. Hemos implementado pequeños subconjuntos del conjunto de datos [CoQA](https://stanfordnlp.github.io/coqa/) para varias plantillas de evaluación para ayudar a ilustrar las diferencias.

Cuando ejecuto una evaluación, a veces se cuelga al final (después del informe final). ¿Qué está sucediendo?

- Este es un problema conocido, pero debería poder interrumpirlo de manera segura y la evaluación debería finalizar inmediatamente después.

Hay mucho código y solo quiero hacer una evaluación rápida. ¿Ayuda? O,

Soy un ingeniero rápido de clase mundial. Elijo no codificar. ¿Cómo puedo aportar mi sabiduría?

- Si sigue una [plantilla de evaluación] existente (docs/eval-templates.md) para crear una evaluación básica o graduada por modelo, ¡no necesita escribir ningún código de evaluación! Simplemente proporcione sus datos en formato JSON y especifique sus parámetros de evaluación en YAML. [build-eval.md](docs/build-eval.md) lo guía a través de estos pasos, y puede complementar estas instrucciones con los cuadernos de Jupyter Notebook en la carpeta `examples` para ayudarlo a comenzar rápidamente. Sin embargo, tenga en cuenta que una buena evaluación inevitablemente requerirá una reflexión cuidadosa y una experimentación rigurosa.

## Descargo de responsabilidad

Al contribuir con Evals, usted acepta que la lógica y los datos de su evaluación estén bajo la misma licencia del MIT que este repositorio. Debe tener los derechos adecuados para cargar cualquier dato utilizado en una evaluación. OpenAI se reserva el derecho de utilizar estos datos en futuras mejoras de servicio de nuestro producto. Las contribuciones a OpenAI Evals estarán sujetas a nuestras Políticas de uso habituales: https://platform.openai.com/docs/usage-pol
