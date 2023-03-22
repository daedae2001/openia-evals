# Cómo ejecutar evaluaciones

Proporcionamos dos interfaces de línea de comandos (CLI): `oaieval` para ejecutar una única evaluación y `oaievalset` para ejecutar un conjunto de evaluaciones.

## Ejecutando una evaluación

Cuando utilice el comando `oaieval`, deberá proporcionar tanto el modelo que desea evaluar como la evaluación para ejecutar. P.ej.,
```sh
oaieval gpt-3.5-turbo test-match
```

En este ejemplo, `gpt-3.5-turbo` es el modelo a evaluar y `test-match` es la evaluación a ejecutar. Los nombres de modelo válidos son aquellos a los que tiene acceso a través de la API. Los nombres de evaluación válidos se especifican en los archivos YAML en `evals/registry/evals`, y sus implementaciones correspondientes se pueden encontrar en `evals/elsuite`.

Estos CLI pueden aceptar varios indicadores para modificar su comportamiento predeterminado. Por ejemplo:
- Si desea iniciar sesión en una base de datos de Snowflake (que ya configuró como se describe en [README] (../README.md)), agregue `--no-local-run`.
- De forma predeterminada, iniciar sesión localmente o en Snowflake escribirá en `tmp/evallogs`, y puede cambiar esto configurando un `--record_path` diferente.

Puede ejecutar `oaieval --help` para ver una lista completa de opciones de CLI.

## Ejecutar un conjunto de evaluación

```sh
prueba oaievalset gpt-3.5-turbo
```

De manera similar, `oaievalset` también espera un nombre de modelo y un nombre de conjunto de evaluación, para los cuales las opciones válidas se especifican en los archivos YAML en `evals/registry/eval_sets`.

De forma predeterminada, ejecutamos con 10 subprocesos, y cada subproceso se agota y se reinicia después de 40 segundos. Puede configurar esto, por ejemplo,

```sh
EVALS_THREADS=42 EVALS_THREAD_TIMEOUT=600 prueba oaievalset gpt-3.5-turbo
```
Ejecutar con más subprocesos hará que la evaluación sea más rápida, aunque tenga en cuenta los costos y sus [límites de tasa] (https://platform.openai.com/docs/guides/rate-limits/overview). Puede ser necesario ejecutar con un tiempo de espera de subproceso más alto si espera que cada muestra tome mucho tiempo, por ejemplo, los datos contienen indicaciones largas que provocan respuestas largas del modelo.

Si tiene que detener su carrera o se bloquea, ¡lo tenemos cubierto! `oaievalset` registra las evaluaciones que terminaron en `/tmp/oaievalset/{model}.{eval_set}.progress.txt`. Simplemente puede volver a ejecutar el comando para continuar donde lo dejó. Si desea ejecutar el conjunto de evaluación desde el principio, elimine este archivo de progreso.

Desafortunadamente, no puede reanudar una sola evaluación desde el medio. Tendrá que reiniciar desde el principio, así que intente que sus evaluaciones individuales se ejecuten rápidamente.
