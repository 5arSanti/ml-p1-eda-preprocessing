## UNIVERSIDAD LIBRE

Programa de Ingeniería — Asignatura: Machine Learning

## TRABAJO PRÁCTICO — Preprocesamiento y EDA como base para un modelo de regresión

## Instrucciones generales

- Herramienta: libre elección (R en jupyter), siempre que el análisis sea reproducible.

## 1. Contexto de negocio

Clínica VitalPlus es un centro de diagnóstico médico (radiografías, ecografías, tomografías, resonancias y laboratorio) con sedes en varias ciudades de Colombia. La dirección quiere entender qué factores están asociados a la duración y al costo de los exámenes, y a la satisfacción de los pacientes, como paso previo a construir un modelo de regresión que permita estimar el costo de un examen a partir de su duración — algo que trabajarán en una siguiente entrega.

En este trabajo actúan como el equipo analista que prepara el terreno para ese modelo: aplicando CRISP-DM, deben entender el problema, evaluar la calidad de los datos disponibles, prepararlos, y realizar un análisis exploratorio enfocado en la futura relación duración-costo y en los demás factores relevantes. No se pide ajustar el modelo de regresión todavía — el alcance llega hasta el Análisis Exploratorio de Datos (EDA).

## 2. Datos entregados

Se entrega el archivo vitalplus_pacientes.csv con 576 registros. El archivo contiene, de forma intencional, problemas de calidad de datos típicos de un entorno real (valores nulos, duplicados, outliers, inconsistencias de formato y de categorías). Diccionario de datos:

| Variable | Descripción | Tipo esperado |
| --- | --- | --- |
| id_paciente | Identificador único del paciente | Numérico |
| edad | Edad del paciente en años | Numérico |
| sede | Ciudad/sede donde se realizó el examen | Categórico (con |
|   |   | inconsistencias) |


| Variable | Descripción | Tipo esperado |
| --- | --- | --- |
| tipo_examen | Tipo de examen diagnóstico realizado | Categórico (con |
|   |   | inconsistencias) |
| fecha_cita | Fecha en que se realizó el examen | Texto (formatos mixtos) |
| dias_espera_cita | Días entre la solicitud y la fecha del examen | Numérico |
| duracion_minutos | Duración del examen en minutos | Numérico |
| costo_examen | Costo del examen (COP) | Numérico (algunos como |
|   |   | texto) |
| canal_remision | Cómo llegó el paciente (EPS, particular, etc.) | Categórico (con |
|   |   | inconsistencias) |
| calificacion_satisfaccion | Calificación de satisfacción (esperado 1 a 5) | Numérico/ordinal |
| reingreso_30dias | 1 si el paciente volvió a solicitar otro examen en 30 | Binario |
|   | días, 0 si no |   |

Nota: las variables duracion_minutos y costo_examen son las que más adelante se usarán para el modelo de regresión — presten especial atención a su calidad y a la relación entre ambas.

## 3. Alcance del trabajo (organizado por fases de CRISP-DM)

## 3.1 Comprensión del negocio

- Redacten el objetivo de negocio de Clínica VitalPlus y el objetivo de este análisis exploratorio, conectándolo con el futuro modelo de regresión duración costo.

- Definan al menos dos criterios de éxito medibles para este análisis.

## 3.2 Comprensión de los datos

- Describan la estructura del dataset: número de registros y variables, tipos de datos reales encontrados.

- Elaboren un reporte de calidad de datos por columna: cantidad y porcentaje de valores nulos, filas duplicadas, valores fuera de rango u outliers, e inconsistencias de categorías detectadas.

## 3.3 Preparación de los datos (preprocesamiento)

- Traten cada uno de los problemas identificados en 3.2: valores nulos, duplicados, outliers, formatos de fecha inconsistentes, valores numéricos guardados como texto y categorías inconsistentes (sede, tipo de examen, canal de remisión).

- Presten especial cuidado a duracion_minutos y costo_examen: son las variables del futuro modelo, así que su tratamiento debe pensarse considerando que se usarán en una regresión (por ejemplo, qué hacer con outliers que podrían distorsionar la relación entre ambas).

- Para cada decisión de tratamiento, expliquen por qué eligieron esa técnica y no otra.

- Incluyan como anexo el dataset ya procesado (CSV) o el código que lo genera.

## 3.4 Análisis Exploratorio de Datos (EDA)


- Univariado: analicen la distribución de al menos 4 variables relevantes (mínimo 2 numéricas y 2 categóricas, incluyendo obligatoriamente duracion_minutos y costo_examen) con el gráfico apropiado y una interpretación escrita.

- Bivariado: exploren la relación entre duración y costo (gráfico de dispersión obligatorio, como anticipo de la futura regresión) y al menos otras 2 relaciones relevantes para el negocio (por ejemplo, satisfacción vs. días de espera, costo vs. tipo de examen).

- Calculen la matriz de correlación de las variables numéricas e interprétenla, prestando atención particular a qué tan fuerte luce la relación entre duración y costo.

- Redacten al menos 3 hallazgos (insights) en lenguaje claro y no técnico, dirigidos a la dirección de Clínica VitalPlus.

## 3.5 Cierre

- Retomen el objetivo de negocio del punto 3.1: ¿los datos, ya preparados, lucen listos para construir el modelo de regresión duración costo? ¿Qué advertirían al equipo que lo construya (por ejemplo, sobre outliers, asimetría, o relaciones que no parecen lineales)?

## 4. Entregables

- Un informe (PDF o Word) que desarrolle, en orden, las secciones 3.1 a 3.5, con texto explicativo (no solo capturas de pantalla o código sin interpretar).

- El código utilizado (notebook .ipynb, script .py, .R o .Rmd), reproducible y comentado.

- Nombren los archivos como: Apellido_Nombre_TrabajoPracticoML (informe) y Apellido_Nombre_TrabajoPracticoML_codigo (código), comprimidos en un único .zip si son varios archivos.

## 5. Rúbrica de evaluación (100 puntos)

| Criterio | Descripción | Puntos |
| --- | --- | --- |
| Comprensión del negocio | Objetivo de negocio, objetivo del análisis y criterios de éxito, claros y | 10 |
|   | conectados con el futuro modelo. |   |
| Comprensión y calidad | Descripción correcta de la estructura del dataset y reporte de calidad | 15 |
| de los datos | completo y bien cuantificado. |   |
|   | Tratamiento adecuado de nulos, duplicados, outliers e |   |
| Preprocesamiento | inconsistencias, con justificación argumentada, y atención particular | 25 |
|   | a duración y costo. |   |
| EDA univariado | Distribuciones analizadas con el gráfico apropiado y una | 15 |
|   | interpretación de negocio correcta. |   |
| EDA bivariado y | Relación duración-costo y otras relaciones exploradas | 20 |
| correlación | correctamente, matriz de correlación interpretada. |   |


| Criterio | Descripción | Puntos |
| --- | --- | --- |
| Insights y conclusiones | Hallazgos accionables y cierre que evalúa si los datos están listos | 10 |
|   | para el futuro modelo. |   |
| Claridad y | Informe bien estructurado; código ordenado, comentado y ejecutable | 5 |
| reproducibilidad | de principio a fin. |   |

— Fin de la guía del trabajo práctico —
