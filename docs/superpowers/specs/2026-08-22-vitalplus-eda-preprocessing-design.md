# Diseño: Trabajo Práctico ML 1 — Preprocesamiento y EDA VitalPlus

**Fecha:** 2026-08-22  
**Estado:** Aprobado por el usuario  
**Alcance:** Notebook R reproducible en Google Colab (`main.ipynb`)

---

## 1. Contexto y objetivo

Clínica VitalPlus necesita preparar datos de exámenes diagnósticos para un futuro modelo de regresión que estime **costo** a partir de **duración**. Este trabajo cubre las fases CRISP-DM 3.1–3.5 de la guía `Guia_Trabajo_Practico_ML_1.md`, con énfasis en:

| Sección | Profundidad |
|---------|-------------|
| 3.1 Comprensión del negocio | Superficial |
| 3.2 Comprensión de los datos | Completa |
| 3.3 Preparación de los datos | Completa |
| 3.4 EDA | Completa |
| 3.5 Cierre | Superficial |

**Fuera de alcance:** informe PDF/Word (se entrega por separado), ajuste de modelo de regresión.

**Dataset:** `data/vitalplus_pacientes.csv` — 576 registros, 11 variables, problemas intencionales de calidad.

---

## 2. Decisiones del usuario

| Tema | Decisión |
|------|----------|
| Entregable de código | Solo `main.ipynb` |
| Entorno | R nativo (IRkernel) en Google Colab |
| Nombre del archivo | `main.ipynb` |
| CSV procesado | Código en notebook + archivo `data/vitalplus_pacientes_procesado.csv` en el repo |
| Outliers duración/costo | A criterio del implementador (estrategia definida en §5.6) |
| Carga de datos en Colab | Clonar repo GitHub → ruta relativa `data/vitalplus_pacientes.csv` |

---

## 3. Enfoque arquitectónico

**Enfoque elegido:** notebook lineal monolítico (un solo `main.ipynb`, orden 3.1 → 3.5).

**Alternativas descartadas:**

- Notebook + scripts R externos — más archivos, peor alineado con entrega única.
- Notebook con muchas funciones reutilizables — útil en producción, pero menos legible para evaluación académica.

**Excepción:** funciones R mínimas solo donde eviten duplicación (tabla de nulos, winsorización IQR).

**Convención de celdas:**

| Tipo | Rol |
|------|-----|
| Markdown `## 3.X — Título` | Inicia cada fase CRISP-DM |
| Markdown explicativo | Qué se hace y por qué (español, lenguaje natural) |
| Código R | Una responsabilidad por celda |
| Markdown interpretación | Resultados tras tablas/gráficos |

**Objetos R principales:**

- `datos_crudos` — lectura inicial sin modificar (usado en 3.2).
- `datos_limpios` — resultado del pipeline 3.3 (usado en 3.4 y exportación).

---

## 4. Setup Colab (antes de 3.2)

### 4.1 Paquetes R

Instalar si faltan:

- `tidyverse` (dplyr, ggplot2, readr, stringr)
- `lubridate`
- `skimr`
- `corrplot`

Opcional: `janitor` (solo utilidades de nombres de columnas, no para valores de negocio).

### 4.2 Clonado del repositorio

Celda con URL configurable al inicio:

```r
# Reemplazar con la URL real del repositorio antes de ejecutar en Colab
REPO_URL <- "https://github.com/<tu-usuario>/ml-p1-eda-preprocessing.git"
if (!dir.exists("ml-p1-eda-preprocessing")) {
  system(paste("git clone", REPO_URL))
}
setwd("ml-p1-eda-preprocessing")
```

### 4.3 Lectura inicial

```r
datos_crudos <- readr::read_csv("data/vitalplus_pacientes.csv", show_col_types = FALSE)
```

---

## 5. Sección 3.1 — Comprensión del negocio (superficial)

**Solo celdas markdown** (sin código):

1. Objetivo de negocio de VitalPlus (1 párrafo): entender factores asociados a duración, costo y satisfacción; preparar estimación costo ← duración.
2. Objetivo del análisis exploratorio + **2 criterios de éxito medibles**, por ejemplo:
   - Dataset procesado con ≤1% de nulos en `duracion_minutos` y `costo_examen`.
   - Relación duración–costo cuantificada (correlación reportada) y documentada con gráfico de dispersión.

---

## 6. Sección 3.2 — Comprensión de los datos

### 6.1 Estructura del dataset

**Código:** `nrow()`, `ncol()`, `names()`, `glimpse()`, `skim()`.

**Interpretación esperada:**

- 576 filas, 11 columnas.
- `costo_examen` leído como texto (character).
- `fecha_cita` como texto con formatos mixtos.
- Resto mayormente numérico con nulos parciales.

### 6.2 Reporte de calidad por columna

#### Nulos (conteo y %)

| Columna | ~% nulos (crudo) |
|---------|------------------|
| costo_examen | 5.9% |
| duracion_minutos | 3.8% |
| edad, canal_remision, calificacion_satisfaccion | 2.8% |
| tipo_examen | 2.1% |
| sede | 1.4% |
| id_paciente, fecha_cita, dias_espera_cita, reingreso_30dias | 0% |

#### Duplicados

- 16 filas duplicadas exactas.
- 16 `id_paciente` repetidos.

#### Outliers / fuera de rango

| Variable | Hallazgo |
|----------|----------|
| duracion_minutos | máx. 400 min (IQR detecta extremos) |
| costo_examen | máx. ~3.2M COP tras limpieza numérica |
| calificacion_satisfaccion | 5 valores fuera de 1–5 (incl. -1 y 7) |
| edad | 4 valores fuera de 0–120 |

#### Inconsistencias categóricas

**sede:** `Bogota`, `Bogotá`, `BOGOTA`, `bogota`, `Medellín`, `medellin`, etc.  
**tipo_examen:** `Lab`, `Laboratorio`, `Eco`, `Ecografía`, `Rx`, `Radiografía`, `RMN`, `resonancia`, etc.  
**canal_remision:** `EPS`, `E.P.S.`, `eps`, `Particular`, `particular`, `Remitido`, `Prepagada`, `Medicina prepagada`, etc.

#### Formatos mixtos

- **fecha_cita:** `2023-10-23`, `28-Apr-2024`, `12-31-2025`, `07/10/2025`, `10/03/2024`, etc.
- **costo_examen:** valores como `$ 85.000` además de numéricos puros (~54 registros con formato no estándar).

### 6.3 Estructura de celdas 3.2

| # | Tipo | Contenido |
|---|------|-----------|
| 1 | Markdown | `## 3.2 — Comprensión de los datos` |
| 2 | Markdown | Introducción al reporte de calidad |
| 3 | Código | Estructura (glimpse, skim) |
| 4 | Markdown | Interpretación estructura |
| 5 | Código | Tabla nulos conteo/% |
| 6 | Markdown | Interpretación nulos |
| 7 | Código | Duplicados |
| 8 | Markdown | Interpretación duplicados |
| 9 | Código | Boxplots + conteo IQR variables numéricas |
| 10 | Markdown | Interpretación outliers |
| 11 | Código | Frecuencias categorías sin normalizar |
| 12 | Markdown | Interpretación inconsistencias |
| 13 | Código | Muestra formatos fecha y costo |
| 14 | Markdown | Resumen consolidado del reporte |

---

## 7. Sección 3.3 — Preparación de los datos

Pipeline secuencial sobre copia de `datos_crudos` → `datos_limpios`.

### 7.1 Duplicados

**Técnica:** `distinct(id_paciente, .keep_all = TRUE)` conservando primera aparición.  
**Justificación:** duplicado por identificador = error de registro; no promediar ni imputar.

### 7.2 Estandarización categórica

**sede →** Bogotá, Cali, Medellín, Bucaramanga, Desconocido  
**tipo_examen →** Laboratorio, Ecografía, Radiografía, Resonancia, Tomografía, Desconocido  
**canal_remision →** EPS, Particular, Remitido, Medicina prepagada, Desconocido  

**Método:** `str_trim()`, `str_to_lower()`, quitar acentos (`stringi::stri_trans_general` o equivalente), diccionario de mapeo explícito documentado en markdown.

**Alternativa descartada:** moda para nulos categóricos — sesga hacia categorías dominantes.

### 7.3 Fechas

**Técnica:** `lubridate::parse_date_time(fecha_cita, orders = c("ymd", "dmy", "mdy", "bmdy", "bdmy"))`  
**Alternativa descartada:** `as.Date()` único — falla con formatos heterogéneos.

### 7.4 costo_examen (texto → numérico)

**Técnica:**

1. Eliminar `$`, espacios.
2. Tratar punto como separador de miles (formato colombiano `$ 85.000` → 85000).
3. `as.numeric()`.

**Alternativa descartada:** eliminar filas no parseables — pierde ~54 registros.

### 7.5 Corrección de rangos

| Variable | Acción |
|----------|--------|
| calificacion_satisfaccion | Fuera de 1–5 → `NA` |
| edad | Fuera de 0–120 → `NA` |

### 7.6 Imputación de nulos

| Variable | Técnica | Justificación |
|----------|---------|---------------|
| edad, calificacion_satisfaccion | Mediana global | Robusta ante outliers |
| sede, tipo_examen, canal_remision | Categoría "Desconocido" | Sin sesgo hacia moda |
| duracion_minutos, costo_examen | Mediana por `tipo_examen` | Respeta diferencia clínica entre tipos de examen |

### 7.7 Outliers — duracion_minutos y costo_examen

**Técnica elegida: winsorización IQR (1.5×)**

- Calcular Q1, Q3, IQR por variable.
- Recortar valores por debajo de `Q1 - 1.5*IQR` y por encima de `Q3 + 1.5*IQR` a esos límites.
- Documentar cuántos valores se recortaron por variable.

**Justificación:**

| Alternativa | Por qué se descarta |
|-------------|---------------------|
| Eliminar filas outlier | Reduce muestra (~560 filas); pierde casos clínicos válidos extremos |
| Conservar sin tratar | Distorsiona regresión lineal futura (apalancamiento de coeficientes) |
| Log-transform en 3.3 | Cambia unidad interpretable; se recomienda como opción en 3.5, no como limpieza obligatoria |

### 7.8 Exportación

```r
readr::write_csv(datos_limpios, "data/vitalplus_pacientes_procesado.csv")
```

Incluir markdown resumen antes/después: filas finales, nulos restantes (= 0), niveles categóricos finales.

### 7.9 Estructura de celdas 3.3

| # | Tipo | Contenido |
|---|------|-----------|
| 1 | Markdown | `## 3.3 — Preparación de los datos` |
| 2 | Markdown | Introducción pipeline |
| 3–4 | MD + código | Duplicados |
| 5–6 | MD + código | Categorías |
| 7–8 | MD + código | Fechas |
| 9–10 | MD + código | costo_examen numérico |
| 11–12 | MD + código | Rangos edad/calificacion |
| 13–14 | MD + código | Imputación nulos |
| 15–16 | MD + código | Winsorización duración/costo |
| 17 | Código | Export CSV |
| 18 | Markdown | Resumen antes/después |

Cada celda markdown de tratamiento debe incluir: **qué problema**, **técnica elegida**, **alternativa descartada y por qué**.

---

## 8. Sección 3.4 — EDA

Trabaja exclusivamente sobre `datos_limpios`.

### 8.1 Univariado (≥4 variables: ≥2 numéricas + ≥2 categóricas)

| Variable | Gráfico | Paquete |
|----------|---------|---------|
| duracion_minutos *(obligatoria)* | Histograma + densidad | ggplot2 |
| costo_examen *(obligatoria)* | Histograma | ggplot2 |
| tipo_examen *(categórica)* | Barras horizontales ordenadas | ggplot2 |
| canal_remision *(categórica)* | Barras | ggplot2 |

**Patrón:** markdown intro → código → markdown interpretación (negocio, no solo estadística).

### 8.2 Bivariado (≥3 relaciones)

| Relación | Gráfico | Obligatorio |
|----------|---------|-------------|
| duracion_minutos vs costo_examen | Scatter + `geom_smooth(method = "lm")` | Sí |
| calificacion_satisfaccion vs dias_espera_cita | Scatter o boxplot por bins | No |
| costo_examen vs tipo_examen | Boxplot por categoría | No |

### 8.3 Correlación

- Matriz de correlación de variables numéricas: `edad`, `dias_espera_cita`, `duracion_minutos`, `costo_examen`, `calificacion_satisfaccion`, `reingreso_30dias`.
- Visualización con `corrplot::corrplot()` o heatmap ggplot.
- Interpretación con énfasis en correlación **duración–costo**.

### 8.4 Insights (≥3, lenguaje no técnico)

Redactar al menos 3 hallazgos dirigidos a la dirección de VitalPlus. Ejemplos orientativos (valores finales salen de la ejecución):

1. Relación positiva duración–costo, pero con dispersión (exámenes similares en duración pueden variar en precio).
2. Mayor tiempo de espera se asocia con menor satisfaccion (si los datos lo confirman).
3. Concentración de demanda por sede o canal (EPS vs particular) con implicaciones operativas.

### 8.5 Estructura de celdas 3.4

| # | Tipo | Contenido |
|---|------|-----------|
| 1 | Markdown | `## 3.4 — Análisis Exploratorio de Datos (EDA)` |
| 2 | Markdown | Introducción EDA |
| 3–10 | MD + código + MD | Univariado (4 variables) |
| 11–17 | MD + código + MD | Bivariado (3 relaciones) |
| 18–19 | MD + código | Matriz correlación |
| 20 | Markdown | Interpretación correlación |
| 21 | Markdown | ≥3 insights para dirección |

---

## 9. Sección 3.5 — Cierre (superficial)

**Solo markdown** (2 celdas):

1. **¿Datos listos para regresión duración → costo?** — Sí, con reservas: pipeline aplicado, relación visible, winsorización reduce extremos, dispersión permanece.
2. **Advertencias al equipo de modelado** — asimetría en costo (evaluar log-transform), posible no-linealidad, variables categóricas no incluidas aún, revisar supuestos de linealidad y homocedasticidad.

---

## 10. Archivos del repositorio

| Archivo | Acción |
|---------|--------|
| `main.ipynb` | Crear — notebook principal |
| `data/vitalplus_pacientes.csv` | Existente — entrada |
| `data/vitalplus_pacientes_procesado.csv` | Generar al ejecutar 3.3 — salida |
| `docs/superpowers/specs/2026-08-22-vitalplus-eda-preprocessing-design.md` | Este documento |

---

## 11. Criterios de aceptación

- [ ] `main.ipynb` ejecuta de principio a fin en Colab con kernel R (IRkernel) sin errores.
- [ ] Secciones 3.1–3.5 con títulos markdown `## 3.X`.
- [ ] 3.2 incluye reporte cuantificado de nulos, duplicados, outliers e inconsistencias.
- [ ] 3.3 trata todos los problemas de 3.2 con justificación por decisión.
- [ ] 3.3 exporta `data/vitalplus_pacientes_procesado.csv`.
- [ ] 3.4 incluye ≥4 univariados (incl. duración y costo), scatter duración–costo, ≥2 bivariados adicionales, matriz correlación, ≥3 insights.
- [ ] Cada paso de 3.2–3.4 tiene celda markdown explicativa en español.
- [ ] Responsabilidades separadas por celdas de código.

---

## 12. Riesgos y mitigaciones

| Riesgo | Mitigación |
|--------|------------|
| URL del repo no configurada en Colab | Variable `REPO_URL` al inicio con comentario claro |
| IRkernel no disponible en Colab | Celda markdown con instrucciones: Runtime → Change runtime type → R |
| Parseo de fechas falla en casos edge | Reportar cuántas fechas quedaron NA; documentar en markdown |
| Winsorización oculta señal real | Documentar conteo de valores recortados; mencionar en 3.5 |
