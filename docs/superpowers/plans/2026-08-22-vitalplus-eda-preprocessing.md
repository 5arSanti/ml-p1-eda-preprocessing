# VitalPlus EDA & Preprocessing Notebook — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `main.ipynb` — a reproducible R (IRkernel) Jupyter notebook for Google Colab that covers CRISP-DM sections 3.1–3.5 for VitalPlus patient data, with deep coverage of 3.2–3.4 and exports `data/vitalplus_pacientes_procesado.csv`.

**Architecture:** Single linear notebook (`main.ipynb`). Markdown cells explain each step in Spanish; code cells have one responsibility each. `datos_crudos` is read once in setup and used in 3.2; `datos_limpios` is built in 3.3 and used in 3.4. Two small helper functions (`reporte_nulos`, `winsorizar_iqr`) avoid duplication.

**Tech Stack:** R 4.x, IRkernel, Google Colab, tidyverse, lubridate, skimr, corrplot, stringi

## Global Constraints

- Entregable de código: solo `main.ipynb` (informe PDF/Word fuera de alcance).
- Entorno: R nativo (IRkernel) en Google Colab.
- Nombre del archivo: `main.ipynb`.
- CSV procesado: código en notebook + archivo `data/vitalplus_pacientes_procesado.csv` en el repo.
- Outliers duración/costo: winsorización IQR (1.5×) en 3.3.
- Carga de datos en Colab: clonar repo GitHub → ruta relativa `data/vitalplus_pacientes.csv`.
- Secciones 3.1 y 3.5: superficiales; 3.2, 3.3, 3.4: completas.
- Cada paso 3.2–3.4: celda markdown explicativa en español + celda código separada.
- Títulos de sección: `## 3.X — ...` al iniciar cada fase CRISP-DM.

**Spec reference:** `docs/superpowers/specs/2026-08-22-vitalplus-eda-preprocessing-design.md`

---

## File Map

| File | Responsibility |
|------|----------------|
| `main.ipynb` | Notebook completo: setup, 3.1–3.5, gráficos, exportación |
| `data/vitalplus_pacientes.csv` | Entrada (existente) |
| `data/vitalplus_pacientes_procesado.csv` | Salida generada por Task 4 |

---

### Task 1: Scaffold notebook — título, 3.1, setup Colab, helpers

**Files:**
- Create: `main.ipynb`

**Interfaces:**
- Consumes: ninguno
- Produces: notebook con kernelspec R, objetos `datos_crudos`, funciones `reporte_nulos()`, `winsorizar_iqr()`, `limpiar_costo()`

- [ ] **Step 1: Crear `main.ipynb` con metadata IRkernel**

Crear notebook Jupyter con:

```json
{
  "kernelspec": {
    "display_name": "R",
    "language": "R",
    "name": "ir"
  },
  "language_info": {
    "name": "R",
    "codemirror_mode": "r",
    "file_extension": ".r",
    "mimetype": "text/x-r-source",
    "pygments_lexer": "r"
  }
}
```

- [ ] **Step 2: Celda markdown — título e índice**

```markdown
# Trabajo Práctico ML 1 — Preprocesamiento y EDA
## Clínica VitalPlus

**Asignatura:** Machine Learning  
**Herramienta:** R en Jupyter (Google Colab, kernel R / IRkernel)

### Índice
1. [3.1 Comprensión del negocio](#31)
2. [Setup — paquetes, repositorio y datos](#setup)
3. [3.2 Comprensión de los datos](#32)
4. [3.3 Preparación de los datos](#33)
5. [3.4 Análisis Exploratorio de Datos (EDA)](#34)
6. [3.5 Cierre](#35)

> **Colab:** Runtime → Change runtime type → **R**. Reemplazar `REPO_URL` en el setup con la URL de tu repositorio GitHub.
```

- [ ] **Step 3: Celda markdown — `## 3.1 — Comprensión del negocio`**

```markdown
## 3.1 — Comprensión del negocio

Clínica VitalPlus es un centro de diagnóstico médico con sedes en varias ciudades de Colombia. La dirección quiere entender qué factores se asocian con la **duración**, el **costo** y la **satisfacción** de los exámenes, como paso previo a construir un modelo de regresión que estime el costo de un examen a partir de su duración.

**Objetivo de negocio:** Identificar patrones en los datos operativos que permitan anticipar el costo de los exámenes según su duración, mejorar la planificación de recursos y la experiencia del paciente.

**Objetivo de este análisis exploratorio:** Evaluar la calidad del dataset, limpiarlo de forma reproducible y explorar la relación duración–costo (y otras variables relevantes) para preparar el terreno del modelo de regresión.

**Criterios de éxito medibles:**
1. Dataset procesado sin valores nulos en `duracion_minutos` y `costo_examen` (variables del futuro modelo).
2. Relación duración–costo cuantificada (correlación + gráfico de dispersión) y documentada con interpretación de negocio.
```

- [ ] **Step 4: Celda markdown — Setup intro**

```markdown
## Setup — paquetes, repositorio y datos

Antes del análisis instalamos las librerías necesarias, clonamos el repositorio con el dataset y definimos funciones auxiliares que reutilizaremos en las secciones 3.2 y 3.3.
```

- [ ] **Step 5: Celda código R — instalar paquetes**

```r
paquetes <- c("tidyverse", "lubridate", "skimr", "corrplot", "stringi")
for (pkg in paquetes) {
  if (!requireNamespace(pkg, quietly = TRUE)) {
    install.packages(pkg, repos = "https://cloud.r-project.org")
  }
}

library(tidyverse)
library(lubridate)
library(skimr)
library(corrplot)
library(stringi)
```

- [ ] **Step 6: Celda código R — clonar repo y leer datos**

```r
# Reemplazar con la URL real del repositorio antes de ejecutar en Colab
REPO_URL <- "https://github.com/<tu-usuario>/ml-p1-eda-preprocessing.git"
REPO_DIR <- "ml-p1-eda-preprocessing"

if (!dir.exists(REPO_DIR)) {
  system(paste("git clone", REPO_URL))
}
setwd(REPO_DIR)

datos_crudos <- readr::read_csv(
  "data/vitalplus_pacientes.csv",
  show_col_types = FALSE
)
```

- [ ] **Step 7: Celda código R — funciones auxiliares**

```r
reporte_nulos <- function(df) {
  df %>%
    summarise(across(everything(), ~ sum(is.na(.x)))) %>%
    pivot_longer(everything(), names_to = "variable", values_to = "nulos") %>%
    mutate(porcentaje = round(100 * nulos / nrow(df), 2)) %>%
    arrange(desc(nulos))
}

limpiar_costo <- function(x) {
  x <- as.character(x)
  x <- str_replace_all(x, "[$\\s]", "")
  # Formato colombiano: punto como separador de miles (ej. 85.000)
  x <- str_replace_all(x, "\\.", "")
  as.numeric(x)
}

winsorizar_iqr <- function(x, mult = 1.5) {
  q <- quantile(x, probs = c(0.25, 0.75), na.rm = TRUE)
  iqr_val <- q[2] - q[1]
  lim_inf <- q[1] - mult * iqr_val
  lim_sup <- q[2] + mult * iqr_val
  recortados <- sum(x < lim_inf | x > lim_sup, na.rm = TRUE)
  list(
    valor = pmin(pmax(x, lim_inf), lim_sup),
    lim_inf = lim_inf,
    lim_sup = lim_sup,
    recortados = recortados
  )
}

normalizar_texto <- function(x) {
  x <- str_trim(as.character(x))
  x <- stri_trans_general(x, "Latin-ASCII")
  str_to_lower(x)
}
```

- [ ] **Step 8: Verificar scaffold**

Run (si R está disponible localmente):

```bash
jupyter nbconvert --to notebook --execute main.ipynb --output /tmp/main_test.ipynb 2>&1 | tail -5
```

En Colab: ejecutar celdas hasta setup; verificar `nrow(datos_crudos) == 576`.

Expected: sin errores; `datos_crudos` con 576 filas y 11 columnas.

- [ ] **Step 9: Commit**

```bash
git add main.ipynb
git commit -m "feat: scaffold main.ipynb with setup and helpers"
```

---

### Task 2: Sección 3.2 — Comprensión de los datos

**Files:**
- Modify: `main.ipynb` (añadir celdas después del setup)

**Interfaces:**
- Consumes: `datos_crudos`, `reporte_nulos()`
- Produces: reportes impresos en notebook (sin modificar `datos_crudos`)

- [ ] **Step 1: Celda markdown título e intro 3.2**

```markdown
## 3.2 — Comprensión de los datos

En esta sección describimos la estructura del dataset y elaboramos un reporte de calidad por columna: valores nulos, filas duplicadas, valores fuera de rango u outliers, e inconsistencias en categorías y formatos. Todo el análisis se realiza sobre `datos_crudos` sin modificar los datos.
```

- [ ] **Step 2: Celda código — estructura del dataset**

```r
cat("Registros:", nrow(datos_crudos), "\n")
cat("Variables:", ncol(datos_crudos), "\n")
cat("Nombres:", paste(names(datos_crudos), collapse = ", "), "\n\n")
glimpse(datos_crudos)
skim(datos_crudos)
```

- [ ] **Step 3: Celda markdown — interpretación estructura**

```markdown
El dataset contiene **576 registros** y **11 variables**. Al leer el CSV, `costo_examen` aparece como texto (character) porque algunos valores incluyen símbolos de moneda y separadores de miles. `fecha_cita` también es texto con formatos de fecha mezclados. Las variables numéricas (`edad`, `duracion_minutos`, `calificacion_satisfaccion`) tienen valores nulos parciales que debemos cuantificar.
```

- [ ] **Step 4: Celda código — reporte de nulos**

```r
tabla_nulos <- reporte_nulos(datos_crudos)
print(tabla_nulos)
```

- [ ] **Step 5: Celda markdown — interpretación nulos**

```markdown
Las columnas con más valores faltantes son `costo_examen` (~6%) y `duracion_minutos` (~4%). También hay nulos en `edad`, `canal_remision` y `calificacion_satisfaccion` (~3% cada una). Las variables `id_paciente`, `fecha_cita`, `dias_espera_cita` y `reingreso_30dias` no tienen nulos. Estos vacíos deben tratarse en la fase 3.3, con especial atención a duración y costo por ser las variables del futuro modelo.
```

- [ ] **Step 6: Celda código — duplicados**

```r
duplicados_exactos <- sum(duplicated(datos_crudos))
duplicados_id <- sum(duplicated(datos_crudos$id_paciente))

cat("Filas duplicadas (exactas):", duplicados_exactos, "\n")
cat("id_paciente repetidos:", duplicados_id, "\n")

datos_crudos %>%
  count(id_paciente) %>%
  filter(n > 1) %>%
  head(10)
```

- [ ] **Step 7: Celda markdown — interpretación duplicados**

```markdown
Se detectan **16 filas duplicadas** y **16 identificadores de paciente repetidos**. Esto sugiere registros duplicados por error de captura, no pacientes distintos con el mismo ID. En el preprocesamiento eliminaremos duplicados conservando la primera aparición de cada `id_paciente`.
```

- [ ] **Step 8: Celda código — outliers y fuera de rango**

```r
vars_numericas <- c(
  "edad", "dias_espera_cita", "duracion_minutos",
  "calificacion_satisfaccion"
)

costo_num_crudo <- limpiar_costo(datos_crudos$costo_examen)

boxplot(
  datos_crudos$duracion_minutos,
  main = "duracion_minutos (crudo)",
  ylab = "minutos"
)
boxplot(
  costo_num_crudo,
  main = "costo_examen (crudo, limpiado)",
  ylab = "COP"
)

fuera_rango_cal <- sum(
  datos_crudos$calificacion_satisfaccion < 1 |
    datos_crudos$calificacion_satisfaccion > 5,
  na.rm = TRUE
)
fuera_rango_edad <- sum(
  datos_crudos$edad < 0 | datos_crudos$edad > 120,
  na.rm = TRUE
)

cat("Calificación fuera de 1-5:", fuera_rango_cal, "\n")
cat("Edad fuera de 0-120:", fuera_rango_edad, "\n")
cat("Max duracion_minutos:", max(datos_crudos$duracion_minutos, na.rm = TRUE), "\n")
cat("Max costo_examen:", max(costo_num_crudo, na.rm = TRUE), "\n")
```

- [ ] **Step 9: Celda markdown — interpretación outliers**

```markdown
Los boxplots revelan valores extremos en **duración** (hasta 400 minutos) y **costo** (valores muy superiores al rango típico). Además hay **5 calificaciones fuera del rango 1–5** y **4 edades fuera de 0–120 años**, que son errores de captura. En 3.3 corregiremos rangos inválidos y aplicaremos winsorización IQR a duración y costo para reducir el efecto de outliers en la futura regresión, sin eliminar filas.
```

- [ ] **Step 10: Celda código — inconsistencias categóricas**

```r
cat("--- sede (valores únicos) ---\n")
datos_crudos %>% count(sede, sort = TRUE) %>% print(n = 20)

cat("\n--- tipo_examen (valores únicos) ---\n")
datos_crudos %>% count(tipo_examen, sort = TRUE) %>% print(n = 20)

cat("\n--- canal_remision (valores únicos) ---\n")
datos_crudos %>% count(canal_remision, sort = TRUE) %>% print(n = 20)
```

- [ ] **Step 11: Celda markdown — interpretación categorías**

```markdown
Las tres variables categóricas muestran **inconsistencias de formato**: mayúsculas/minúsculas (`BOGOTA` vs `Bogotá`), acentos (`Bogota` vs `Bogotá`), abreviaturas (`Lab` vs `Laboratorio`, `Eco` vs `Ecografía`, `Rx` vs `Radiografía`) y variantes de canal (`EPS` vs `E.P.S.`, `Prepagada` vs `Medicina prepagada`). En 3.3 aplicaremos un diccionario de mapeo explícito hacia categorías canónicas.
```

- [ ] **Step 12: Celda código — formatos mixtos fecha y costo**

```r
cat("Muestra de formatos en fecha_cita:\n")
datos_crudos %>%
  distinct(fecha_cita) %>%
  slice_head(n = 15)

costo_texto <- as.character(datos_crudos$costo_examen)
formato_moneda <- sum(grepl("[$]", costo_texto), na.rm = TRUE)
cat("\nRegistros de costo con símbolo $:", formato_moneda, "\n")

cat("\nMuestra de costos con formato no estándar:\n")
datos_crudos %>%
  filter(grepl("[$]", as.character(costo_examen))) %>%
  select(id_paciente, costo_examen) %>%
  head(10)
```

- [ ] **Step 13: Celda markdown — resumen reporte de calidad**

```markdown
### Resumen del reporte de calidad

| Problema | Columnas afectadas | Magnitud |
|----------|---------------------|----------|
| Valores nulos | costo, duración, edad, sede, tipo_examen, canal, calificación | 1–6% por columna |
| Duplicados | id_paciente | 16 registros |
| Outliers | duracion_minutos, costo_examen | Valores extremos visibles en boxplots |
| Fuera de rango | calificacion_satisfaccion, edad | 5 y 4 registros respectivamente |
| Categorías inconsistentes | sede, tipo_examen, canal_remision | Múltiples variantes por categoría |
| Formatos mixtos | fecha_cita, costo_examen | Varios formatos de fecha; costos con `$` y puntos de miles |

Este diagnóstico guía el pipeline de limpieza en la sección 3.3.
```

- [ ] **Step 14: Verificar 3.2**

Ejecutar celdas 3.2 en Colab. Expected: tablas impresas, boxplots renderizados, sin modificación de `datos_crudos`.

- [ ] **Step 15: Commit**

```bash
git add main.ipynb
git commit -m "feat: add section 3.2 data quality report"
```

---

### Task 3: Sección 3.3 — Preparación de los datos (pipeline completo)

**Files:**
- Modify: `main.ipynb`

**Interfaces:**
- Consumes: `datos_crudos`, `limpiar_costo()`, `winsorizar_iqr()`, `normalizar_texto()`, `reporte_nulos()`
- Produces: `datos_limpios` (data.frame, 560 filas aprox., 0 nulos)

- [ ] **Step 1: Celda markdown título e intro 3.3**

```markdown
## 3.3 — Preparación de los datos

A partir de los problemas detectados en 3.2, aplicamos un pipeline de limpieza secuencial. Cada decisión incluye la técnica elegida y la alternativa descartada. El resultado final se guarda en `datos_limpios` y se exporta como CSV.
```

- [ ] **Step 2: Celda markdown + código — eliminar duplicados**

Markdown:

```markdown
### 3.3.1 — Duplicados

**Problema:** 16 registros con `id_paciente` repetido.  
**Técnica:** Eliminar duplicados por `id_paciente`, conservando la primera aparición (`distinct(.keep_all = TRUE)`).  
**Alternativa descartada:** Mantener duplicados o promediar valores — distorsionaría conteos y relaciones en el EDA.
```

Código:

```r
datos_limpios <- datos_crudos %>%
  distinct(id_paciente, .keep_all = TRUE)

cat("Filas antes:", nrow(datos_crudos), "→ después:", nrow(datos_limpios), "\n")
```

- [ ] **Step 3: Celda markdown + código — estandarizar categorías**

Markdown:

```markdown
### 3.3.2 — Categorías inconsistentes

**Problema:** Variantes de sede, tipo de examen y canal de remisión (mayúsculas, acentos, abreviaturas).  
**Técnica:** Normalizar texto (trim, minúsculas, sin acentos) y mapear con diccionario explícito hacia categorías canónicas. Valores nulos o no reconocidos → `"Desconocido"`.  
**Alternativa descartada:** Imputar con la moda — sesgaría hacia Bogotá/EPS que ya son las categorías más frecuentes.
```

Código:

```r
map_sede <- c(
  "bogota" = "Bogotá", "cali" = "Cali",
  "medellin" = "Medellín", "bucaramanga" = "Bucaramanga"
)
map_tipo <- c(
  "laboratorio" = "Laboratorio", "lab" = "Laboratorio",
  "ecografia" = "Ecografía", "eco" = "Ecografía",
  "radiografia" = "Radiografía", "rx" = "Radiografía",
  "resonancia" = "Resonancia", "rmn" = "Resonancia",
  "tomografia" = "Tomografía", "tac" = "Tomografía"
)
map_canal <- c(
  "eps" = "EPS", "e.p.s." = "EPS",
  "particular" = "Particular",
  "remitido" = "Remitido",
  "prepagada" = "Medicina prepagada",
  "medicina prepagada" = "Medicina prepagada"
)

datos_limpios <- datos_limpios %>%
  mutate(
    sede_norm = normalizar_texto(sede),
    tipo_norm = normalizar_texto(tipo_examen),
    canal_norm = normalizar_texto(canal_remision),
    sede = map_sede[sede_norm],
    tipo_examen = map_tipo[tipo_norm],
    canal_remision = map_canal[canal_norm]
  ) %>%
  mutate(
    sede = if_else(is.na(sede), "Desconocido", sede),
    tipo_examen = if_else(is.na(tipo_examen), "Desconocido", tipo_examen),
    canal_remision = if_else(is.na(canal_remision), "Desconocido", canal_remision)
  ) %>%
  select(-sede_norm, -tipo_norm, -canal_norm)

datos_limpios %>% count(sede, sort = TRUE)
datos_limpios %>% count(tipo_examen, sort = TRUE)
datos_limpios %>% count(canal_remision, sort = TRUE)
```

- [ ] **Step 4: Celda markdown + código — parsear fechas**

Markdown:

```markdown
### 3.3.3 — Formatos de fecha inconsistentes

**Problema:** `fecha_cita` mezcla formatos (`YYYY-MM-DD`, `DD-Mon-YYYY`, `DD/MM/YYYY`, etc.).  
**Técnica:** `parse_date_time()` con múltiples órdenes (`ymd`, `dmy`, `mdy`, `bmdy`, `bdmy`).  
**Alternativa descartada:** `as.Date()` con un solo formato — falla con formatos heterogéneos.
```

Código:

```r
datos_limpios <- datos_limpios %>%
  mutate(
    fecha_cita = parse_date_time(
      fecha_cita,
      orders = c("ymd", "dmy", "mdy", "bmdy", "bdmy"),
      quiet = TRUE
    )
  )

fechas_na <- sum(is.na(datos_limpios$fecha_cita))
cat("Fechas no parseadas:", fechas_na, "\n")
```

- [ ] **Step 5: Celda markdown + código — limpiar costo_examen**

Markdown:

```markdown
### 3.3.4 — costo_examen almacenado como texto

**Problema:** Valores con `$`, espacios y punto como separador de miles (ej. `$ 85.000`).  
**Técnica:** Función `limpiar_costo()` — elimina `$` y espacios, quita puntos de miles, convierte a numérico.  
**Alternativa descartada:** Eliminar filas no parseables — perderíamos ~54 registros con información útil.
```

Código:

```r
datos_limpios <- datos_limpios %>%
  mutate(costo_examen = limpiar_costo(costo_examen))

cat("Nulos en costo tras limpieza:", sum(is.na(datos_limpios$costo_examen)), "\n")
summary(datos_limpios$costo_examen)
```

- [ ] **Step 6: Celda markdown + código — corregir rangos**

Markdown:

```markdown
### 3.3.5 — Valores fuera de rango

**Problema:** Calificaciones fuera de 1–5 y edades fuera de 0–120.  
**Técnica:** Reemplazar valores inválidos por `NA` para imputar después.  
**Alternativa descartada:** Conservar valores erróneos — distorsionarían estadísticas y correlaciones.
```

Código:

```r
datos_limpios <- datos_limpios %>%
  mutate(
    calificacion_satisfaccion = if_else(
      calificacion_satisfaccion < 1 | calificacion_satisfaccion > 5,
      NA_real_, calificacion_satisfaccion
    ),
    edad = if_else(edad < 0 | edad > 120, NA_real_, edad)
  )
```

- [ ] **Step 7: Celda markdown + código — imputar nulos**

Markdown:

```markdown
### 3.3.6 — Valores nulos

**Problema:** Nulos restantes en edad, calificación, sede, tipo_examen, canal, duración y costo.  
**Técnicas:**
- `edad` y `calificacion_satisfaccion`: mediana global (robusta ante outliers).
- `sede`, `tipo_examen`, `canal_remision`: categoría `"Desconocido"`.
- `duracion_minutos` y `costo_examen`: mediana **por `tipo_examen`** (un laboratorio no debe imputarse con la duración de una resonancia).

**Alternativa descartada para duración/costo:** mediana global — ignoraría que distintos tipos de examen tienen duraciones y costos muy diferentes.
```

Código:

```r
mediana_edad <- median(datos_limpios$edad, na.rm = TRUE)
mediana_cal <- median(datos_limpios$calificacion_satisfaccion, na.rm = TRUE)

medias_por_tipo <- datos_limpios %>%
  group_by(tipo_examen) %>%
  summarise(
    med_dur = median(duracion_minutos, na.rm = TRUE),
    med_costo = median(costo_examen, na.rm = TRUE),
    .groups = "drop"
  )

datos_limpios <- datos_limpios %>%
  left_join(medias_por_tipo, by = "tipo_examen") %>%
  mutate(
    edad = if_else(is.na(edad), mediana_edad, edad),
    calificacion_satisfaccion = if_else(
      is.na(calificacion_satisfaccion), mediana_cal, calificacion_satisfaccion
    ),
    sede = if_else(is.na(sede), "Desconocido", sede),
    tipo_examen = if_else(is.na(tipo_examen), "Desconocido", tipo_examen),
    canal_remision = if_else(is.na(canal_remision), "Desconocido", canal_remision),
    duracion_minutos = if_else(is.na(duracion_minutos), med_dur, duracion_minutos),
    costo_examen = if_else(is.na(costo_examen), med_costo, costo_examen)
  ) %>%
  select(-med_dur, -med_costo)

print(reporte_nulos(datos_limpios))
```

- [ ] **Step 8: Celda markdown + código — winsorización IQR**

Markdown:

```markdown
### 3.3.7 — Outliers en duracion_minutos y costo_examen

**Problema:** Valores extremos en duración (hasta 400 min) y costo que pueden distorsionar una regresión lineal.  
**Técnica:** Winsorización IQR (1.5×): recortar valores por debajo de Q1−1.5·IQR y por encima de Q3+1.5·IQR al límite correspondiente.  
**Alternativas descartadas:**
- *Eliminar filas outlier:* reduce el tamaño muestral (~560 filas).
- *Conservar sin tratar:* apalanca coeficientes de la regresión futura.
- *Log-transform aquí:* cambia la interpretabilidad; se evaluará en 3.5 para el modelado.
```

Código:

```r
res_dur <- winsorizar_iqr(datos_limpios$duracion_minutos)
res_costo <- winsorizar_iqr(datos_limpios$costo_examen)

cat("Duración — recortados:", res_dur$recortados,
    "| límite inf:", res_dur$lim_inf, "| límite sup:", res_dur$lim_sup, "\n")
cat("Costo — recortados:", res_costo$recortados,
    "| límite inf:", res_costo$lim_inf, "| límite sup:", res_costo$lim_sup, "\n")

datos_limpios <- datos_limpios %>%
  mutate(
    duracion_minutos = res_dur$valor,
    costo_examen = res_costo$valor
  )
```

- [ ] **Step 9: Celda código — exportar CSV**

```r
readr::write_csv(datos_limpios, "data/vitalplus_pacientes_procesado.csv")
cat("Exportado: data/vitalplus_pacientes_procesado.csv\n")
cat("Filas finales:", nrow(datos_limpios), "\n")
```

- [ ] **Step 10: Celda markdown — resumen antes/después**

```markdown
### Resumen del preprocesamiento

| Aspecto | Antes (crudo) | Después (limpio) |
|---------|---------------|------------------|
| Filas | 576 | 560 (16 duplicados eliminados) |
| Nulos en duración/costo | Sí | No |
| Categorías | Inconsistentes | 5 sedes, 6 tipos, 5 canales canónicos |
| Outliers duración/costo | Extremos visibles | Winsorizados (IQR 1.5×) |

El dataset procesado está listo para el EDA en 3.4 y como base para el futuro modelo de regresión.
```

- [ ] **Step 11: Verificar pipeline 3.3**

Run (Python sanity check on exported CSV after executing notebook):

```bash
python3 -c "
import pandas as pd
df = pd.read_csv('data/vitalplus_pacientes_procesado.csv')
assert df.shape[0] == 560, f'Expected 560 rows, got {df.shape[0]}'
assert df.isnull().sum().sum() == 0, 'Nulls remain in processed CSV'
print('OK:', df.shape, 'nulls=', df.isnull().sum().sum())
"
```

Expected: 560 filas, 0 nulos totales.

- [ ] **Step 12: Commit**

```bash
git add main.ipynb data/vitalplus_pacientes_procesado.csv
git commit -m "feat: add section 3.3 preprocessing pipeline and export CSV"
```

---

### Task 4: Sección 3.4 — EDA

**Files:**
- Modify: `main.ipynb`

**Interfaces:**
- Consumes: `datos_limpios`
- Produces: gráficos ggplot2, matriz de correlación, insights en markdown

- [ ] **Step 1: Celda markdown título e intro 3.4**

```markdown
## 3.4 — Análisis Exploratorio de Datos (EDA)

Exploramos la distribución de variables individuales (univariado), relaciones entre variables (bivariado) y la matriz de correlación. El foco principal es la relación **duración–costo**, anticipo del futuro modelo de regresión.
```

- [ ] **Step 2: Univariado — duracion_minutos**

Markdown intro:

```markdown
### 3.4.1 — Distribución de duracion_minutos

La duración del examen es la variable independiente del futuro modelo. Analizamos su distribución para entender si los exámenes son mayormente cortos o si hay una cola larga hacia procedimientos más extensos.
```

Código:

```r
ggplot(datos_limpios, aes(x = duracion_minutos)) +
  geom_histogram(aes(y = after_stat(density)), bins = 30, fill = "steelblue", alpha = 0.7) +
  geom_density(color = "darkblue", linewidth = 1) +
  labs(
    title = "Distribución de duracion_minutos",
    x = "Duración (minutos)", y = "Densidad"
  ) +
  theme_minimal()
```

Markdown interpretación:

```markdown
La distribución es **asimétrica hacia la derecha**: la mayoría de exámenes dura menos de 30 minutos (laboratorios y radiografías), pero existe una cola de procedimientos más largos (resonancias, tomografías). Esto implica que la relación con el costo podría no ser uniforme en todos los tipos de examen.
```

- [ ] **Step 3: Univariado — costo_examen**

Markdown intro + código + interpretación (patrón igual):

Código:

```r
ggplot(datos_limpios, aes(x = costo_examen)) +
  geom_histogram(bins = 30, fill = "darkorange", alpha = 0.7) +
  scale_x_continuous(labels = scales::comma) +
  labs(
    title = "Distribución de costo_examen",
    x = "Costo (COP)", y = "Frecuencia"
  ) +
  theme_minimal()
```

Interpretación markdown:

```markdown
El costo también presenta **asimetría positiva**: la mayoría de exámenes cuesta entre 80.000 y 200.000 COP, con un grupo minoritario de procedimientos de alto costo. Esta asimetría sugiere que el equipo de modelado podría evaluar una transformación logarítmica del costo en la siguiente entrega.
```

- [ ] **Step 4: Univariado — tipo_examen (categórica)**

Código:

```r
datos_limpios %>%
  count(tipo_examen) %>%
  ggplot(aes(x = reorder(tipo_examen, n), y = n)) +
  geom_col(fill = "seagreen") +
  coord_flip() +
  labs(title = "Frecuencia por tipo de examen", x = "Tipo", y = "Cantidad") +
  theme_minimal()
```

Interpretación: laboratorios y ecografías concentran la mayor demanda.

- [ ] **Step 5: Univariado — canal_remision (categórica)**

Código:

```r
datos_limpios %>%
  count(canal_remision) %>%
  ggplot(aes(x = reorder(canal_remision, n), y = n)) +
  geom_col(fill = "purple4") +
  coord_flip() +
  labs(title = "Frecuencia por canal de remisión", x = "Canal", y = "Cantidad") +
  theme_minimal()
```

Interpretación: EPS y particular son los canales dominantes.

- [ ] **Step 6: Bivariado — duración vs costo (obligatorio)**

Markdown intro:

```markdown
### 3.4.2 — Relación duración vs costo (anticipo de regresión)

Este gráfico de dispersión es el análisis central: muestra si a mayor duración del examen corresponde un mayor costo, que es la hipótesis del futuro modelo de regresión lineal.
```

Código:

```r
cor_dur_costo <- cor(datos_limpios$duracion_minutos, datos_limpios$costo_examen)
cat("Correlación duración–costo:", round(cor_dur_costo, 3), "\n")

ggplot(datos_limpios, aes(x = duracion_minutos, y = costo_examen)) +
  geom_point(alpha = 0.5, color = "steelblue") +
  geom_smooth(method = "lm", se = TRUE, color = "red") +
  scale_y_continuous(labels = scales::comma) +
  labs(
    title = "Duración vs Costo del examen",
    subtitle = paste0("Correlación = ", round(cor_dur_costo, 3)),
    x = "Duración (minutos)", y = "Costo (COP)"
  ) +
  theme_minimal()
```

Interpretación: relación positiva visible pero con dispersión; no todos los puntos siguen la línea de tendencia.

- [ ] **Step 7: Bivariado — satisfacción vs días de espera**

Código:

```r
ggplot(datos_limpios, aes(x = dias_espera_cita, y = calificacion_satisfaccion)) +
  geom_point(alpha = 0.5, color = "coral") +
  geom_smooth(method = "lm", se = TRUE, color = "darkred") +
  labs(
    title = "Satisfacción vs Días de espera",
    x = "Días de espera", y = "Calificación (1-5)"
  ) +
  theme_minimal()
```

Interpretación: documentar si mayor espera se asocia con menor satisfacción según la tendencia observada.

- [ ] **Step 8: Bivariado — costo vs tipo de examen**

Código:

```r
ggplot(datos_limpios, aes(x = tipo_examen, y = costo_examen)) +
  geom_boxplot(fill = "lightblue") +
  scale_y_continuous(labels = scales::comma) +
  labs(title = "Costo por tipo de examen", x = "Tipo", y = "Costo (COP)") +
  theme_minimal() +
  theme(axis.text.x = element_text(angle = 45, hjust = 1))
```

Interpretación: resonancias y tomografías tienen costos medianos superiores a laboratorios y radiografías.

- [ ] **Step 9: Matriz de correlación**

Markdown intro + código:

```r
vars_corr <- datos_limpios %>%
  select(
    edad, dias_espera_cita, duracion_minutos,
    costo_examen, calificacion_satisfaccion, reingreso_30dias
  )

mat_corr <- cor(vars_corr, use = "complete.obs")
print(round(mat_corr, 3))

corrplot(
  mat_corr,
  method = "color",
  type = "upper",
  addCoef.col = "black",
  number.cex = 0.7,
  tl.cex = 0.8,
  title = "Matriz de correlación",
  mar = c(0, 0, 2, 0)
)
```

Markdown interpretación correlación:

```markdown
La correlación entre **duracion_minutos** y **costo_examen** es positiva y es la más relevante para el futuro modelo. Las demás correlaciones son débiles o moderadas. Esto confirma que duración es un predictor razonable del costo, aunque no explica todo la variabilidad (hay otros factores como tipo de examen y canal).
```

- [ ] **Step 10: Insights para la dirección**

```markdown
### Hallazgos para la dirección de VitalPlus

1. **Duración y costo van de la mano, pero no perfectamente.** Los exámenes más largos tienden a ser más costosos, lo que respalda la idea de estimar costos según duración. Sin embargo, hay variabilidad: dos exámenes con similar duración pueden tener costos distintos según el tipo de procedimiento.

2. **El tiempo de espera impacta la satisfacción.** Pacientes que esperan más días para su cita tienden a calificar peor el servicio. Reducir tiempos de espera podría mejorar la experiencia sin necesidad de reducir costos.

3. **La demanda se concentra en exámenes rápidos y canales EPS.** Laboratorios y radiografías dominan el volumen, y el canal EPS es el principal origen de pacientes. La planificación de capacidad debería priorizar estos servicios en las sedes con mayor demanda.
```

- [ ] **Step 11: Verificar 3.4**

Ejecutar celdas 3.4 en Colab. Expected: 7+ gráficos renderizados, correlación impresa, sin errores.

- [ ] **Step 12: Commit**

```bash
git add main.ipynb
git commit -m "feat: add section 3.4 EDA with plots and insights"
```

---

### Task 5: Sección 3.5, verificación final y limpieza

**Files:**
- Modify: `main.ipynb`

**Interfaces:**
- Consumes: `datos_limpios`, resultados de 3.4
- Produces: notebook completo ejecutable de principio a fin

- [ ] **Step 1: Celda markdown — `## 3.5 — Cierre`**

```markdown
## 3.5 — Cierre

### ¿Los datos están listos para el modelo de regresión duración → costo?

**Sí, con reservas.** El pipeline de limpieza eliminó duplicados, unificó categorías, imputó nulos y winsorizó outliers en duración y costo. El EDA muestra una relación positiva entre ambas variables. Sin embargo, la dispersión alrededor de la tendencia y la asimetría en el costo indican que un modelo lineal simple podría no capturar toda la variabilidad.

### Advertencias al equipo de modelado

1. **Asimetría en costo:** Evaluar transformación logarítmica de `costo_examen` para estabilizar varianza.
2. **No-linealidad:** La relación duración–costo podría variar por tipo de examen; considerar variables categóricas o modelos con interacciones.
3. **Supuestos de regresión:** Verificar linealidad, homocedasticidad y normalidad de residuos antes de confiar en intervalos de predicción.
4. **Winsorización aplicada:** Los valores extremos ya fueron recortados; documentar esto al interpretar predicciones en el rango alto de costos.
```

- [ ] **Step 2: Verificación final del notebook completo**

En Colab (kernel R):

1. Runtime → Change runtime type → R
2. Reemplazar `REPO_URL` con URL real
3. Run all cells

Checklist manual:
- [ ] 576 filas leídas en setup
- [ ] Secciones `## 3.1` a `## 3.5` presentes
- [ ] `data/vitalplus_pacientes_procesado.csv` generado con 560 filas
- [ ] Scatter duración–costo con línea de tendencia visible
- [ ] Matriz de correlación renderizada
- [ ] Sin errores en ninguna celda

Verificación local del CSV (Python):

```bash
python3 -c "
import pandas as pd
df = pd.read_csv('data/vitalplus_pacientes_procesado.csv')
print('rows', len(df), 'cols', len(df.columns), 'nulls', df.isnull().sum().sum())
assert len(df) == 560
assert df.isnull().sum().sum() == 0
print('ALL CHECKS PASSED')
"
```

Expected: `ALL CHECKS PASSED`

- [ ] **Step 3: Commit final**

```bash
git add main.ipynb
git commit -m "feat: add section 3.5 closing and complete VitalPlus EDA notebook"
```

---

## Plan Self-Review

| Spec requirement | Task |
|------------------|------|
| main.ipynb R/IRkernel Colab | Task 1 Step 1 |
| 3.1 superficial | Task 1 Step 3 |
| 3.2 reporte calidad completo | Task 2 |
| 3.3 todos problemas + justificación | Task 3 |
| Export CSV procesado | Task 3 Step 9 |
| 3.4 univariado ≥4 vars | Task 4 Steps 2–5 |
| 3.4 bivariado ≥3 relaciones | Task 4 Steps 6–8 |
| 3.4 correlación | Task 4 Step 9 |
| 3.4 ≥3 insights | Task 4 Step 10 |
| 3.5 superficial | Task 5 Step 1 |
| Markdown español por paso | All tasks |
| Celdas separadas por responsabilidad | All tasks |

**Placeholder scan:** No TBD/TODO found. `REPO_URL` has explicit replace instruction.

**Type consistency:** `datos_crudos` → `datos_limpios` used consistently; helper function signatures match across tasks.
