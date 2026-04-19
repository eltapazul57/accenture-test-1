# contexto.md — Ruta B: Ciencia de Datos

## ICD Financiero S.A. | Intern Challenge Day 2026 - II

> Última actualización: tras exploración inicial del dataset (Sección 0)

---

## Deadline y entrega

- **Fecha límite:** Lunes 20/04/2026 — 11:59 AM
- **Enviar a:** juan.arciniegas@accenture.com | valentina.a.sanchez@accenture.com
- **Nombre de archivo:** `APELLIDO_NOMBRE_RutaB`

---

## El dataset

**Archivo:** `Challenge_DataAI_Candidatos.csv`

- Separador: `;`
- Encoding: `utf-8-sig` (BOM) — **no latin-1**
- Registros: 1.044.110 (shape real post-carga: 1.044.093 no nulos)
- Rango temporal: 30/06/2014 — 31/03/2026 (cortes trimestrales)

### Parámetros de carga obligatorios

```python
df = pd.read_csv(DATA_PATH, sep=';', encoding='utf-8-sig', dtype={'CODIGO_ENTIDAD': str})
df.columns = df.columns.str.strip().str.rstrip(',')
df['QUEJAS_FAVOR_CONSUM_NOACEP_ENT'] = (
    df['QUEJAS_FAVOR_CONSUM_NOACEP_ENT']
    .str.rstrip(',').str.strip()
    .replace('', np.nan).astype(float)
)
```

### Columnas reales del CSV

| Campo                              | Tipo post-carga | Notas                                                                       |
| ---------------------------------- | --------------- | --------------------------------------------------------------------------- |
| `TIPO_ENTIDAD`                   | object          | Código numérico sin etiqueta de texto                                     |
| `CODIGO_ENTIDAD`                 | str             | Forzado a str — tiene ceros a la izquierda                                 |
| `NOMBRE_ENTIDAD`                 | str             | Capitalización inconsistente                                               |
| `FECHA_CORTE`                    | str             | Dos formatos mezclados:`dd/mm/yyyy` y `yyyy-mm-dd`                      |
| `UNIDAD_CAPTURA`                 | float64         | 1 = Entidad Vigilada, 2 = Defensor del Consumidor                           |
| `NOMBRE_UNIDAD_CAPTURA`          | str             | `'ENTIDAD VIGILADA'` o `'DEFENSORES DEL CONSUMIDOR FINANCIERO'`         |
| `CODIGO_PRODUCTO`                | float64         |                                                                             |
| `PRODUCTO`                       | str             | Espacios en blanco extra                                                    |
| `CODIGO_MOTIVO`                  | float64         |                                                                             |
| `MOTIVO`                         | str             | 155 categorías                                                             |
| `QUEJAS_PENDIENTES`              | float64         |                                                                             |
| `QUEJAS_RECIBIDAS`               | float64         | Mediana=3, máximo=25.576. Distribución extremadamente sesgada             |
| `QUEJAS_FINALIZADAS`             | float64         | Total cerradas = N1 + N2 (98.12% consistencia)                              |
| `QUEJAS_FINALIZADAS_N1`          | float64         | Resueltas en**primera línea** de atención                           |
| `QUEJAS_FINALIZADAS_N2`          | float64         | Resueltas en**segunda línea** (escaladas)                            |
| `QUEJAS_EN_TRAMITE`              | float64         | ~30 outliers > 5.000                                                        |
| `QUEJAS_FAVOR_CONSUM_ACEP_ENT`   | float64         | Quejas favor consumidor aceptadas por la entidad                            |
| `QUEJAS_FAVOR_CONSUM_NOACEP_ENT` | float64         | **Mayor indicador de riesgo regulatorio.** Requiere limpieza en carga |

### Columnas del enunciado que NO existen en el CSV

| Columna documentada en enunciado   | Estado                                                            |
| ---------------------------------- | ----------------------------------------------------------------- |
| `QUEJAS_FINALIZAD_FAVOR_CONSUM`  | No existe — dataset tiene `N1` y `N2` por nivel de atención |
| `QUEJAS_FINALIZAD_FAVOR_ENTIDAD` | No existe                                                         |
| `TIEMPO_PROMEDIO_EN_SISTEMA_(H)` | No existe                                                         |

---

## Problemas de calidad — verdad empírica

| # | Problema                                    | Campo                              | Volumen         | Impacto en modelado                 |
| - | ------------------------------------------- | ---------------------------------- | --------------- | ----------------------------------- |
| 1 | Fechas en formato mixto                     | `FECHA_CORTE`                    | ~1.500          | Sin esto no hay dimensión temporal |
| 2 | Nombres entidad en minúsculas              | `NOMBRE_ENTIDAD`                 | ~900            | Duplica entidades en agrupaciones   |
| 3 | Espacios extra en texto                     | `PRODUCTO`                       | ~600            | Fragmenta categorías en encoding   |
| 4 | Inconsistencia lógica N1+N2 ≠ FINALIZADAS | Múltiples                         | ~19.700 (1.88%) | Corrompe variable objetivo          |
| 5 | Duplicados exactos                          | Todas                              | ~150            | Data leakage en split train/test    |
| 6 | Outliers extremos                           | `QUEJAS_EN_TRAMITE`              | ~30 (> 5.000)   | Distorsiona features de backlog     |
| 7 | Tipo mixto                                  | `CODIGO_ENTIDAD`                 | ~300            | Menor — manejado en carga          |
| 8 | Trailing commas en valores                  | `QUEJAS_FAVOR_CONSUM_NOACEP_ENT` | Todo el CSV     | Columna inutilizable sin fix        |
| 9 | 16 filas completamente nulas                | Todas                              | 16              | Eliminación directa en S1          |

---

## Pregunta central — Ruta B

> ¿Qué combinaciones de entidad, producto y motivo concentran el mayor riesgo regulatorio —medido por tasa de resolución a favor del consumidor, acumulación de backlog y tendencia 2014-2026— y qué puede predecir o anticipar ese riesgo?

---

## Variable objetivo — decisión pendiente (Sección 3)

El enunciado asume `TASA_FAVOR = QUEJAS_FINALIZAD_FAVOR_CONSUM / QUEJAS_FINALIZADAS` — columna que no existe en el CSV.

**Lo disponible:**

- `QUEJAS_FAVOR_CONSUM_ACEP_ENT`: favor consumidor aceptado por entidad
- `QUEJAS_FAVOR_CONSUM_NOACEP_ENT`: favor consumidor rechazado por entidad
- `ACEP + NOACEP == FINALIZADAS` solo en 75.60% → no reconstruyen el total

**Decisión a tomar en S3:** construir variable objetivo desde `ACEP + NOACEP` como proxy, documentando la limitación explícitamente.

```python
# Proxy — justificar en S3
TASA_FAVOR = (ACEP + NOACEP) / QUEJAS_FINALIZADAS
ALTO_RIESGO = (TASA_FAVOR > 0.60)  # umbral a validar con distribución real
```

---

## Métricas clave

```python
QUEJAS_NETAS = QUEJAS_RECIBIDAS - QUEJAS_FINALIZADAS        # backlog
TASA_NO_ACEPTACION = NOACEP / (ACEP + NOACEP)               # riesgo regulatorio
OUTLIER_EN_TRAMITE = 5_000
ANIO_INICIO_RIESGO = 2021
```

---

## Entregables requeridos

| Entregable                                  | Descripción                                                                                |
| ------------------------------------------- | ------------------------------------------------------------------------------------------- |
| Notebook `.ipynb`                         | EDA completo + pipeline de modelado con justificaciones. Debe contar una historia.          |
| Reporte de métricas                        | Tabla de resultados con interpretación de negocio para cada métrica.                      |
| Visualizaciones clave                       | Mínimo 5 gráficas del EDA con conclusiones escritas.                                      |
| **Tarea Mandatoria (30% de la nota)** | Documento separado: Top 5 hallazgos + 2-5 hipótesis + resumen ejecutivo 1 página para VP. |

---

## Criterio diferenciador

Un modelo del 70% que identifica entidades con mayor riesgo y propone acción supervisora concreta supera a un 90% sin contexto de negocio. No se evalúa complejidad del modelo ni cantidad de técnicas.

**Señales positivas:** comentar el *por qué* no el *qué* · reconocer resultados inesperados · conectar hallazgo técnico con implicación de negocio · narrativa coherente de principio a fin.


---



**1. Contexto de negocio — qué es una PQRS y qué implica regulatoriamente**
La Superintendencia Financiera puede sancionar entidades con alta tasa de quejas a favor del consumidor. Ese es el riesgo real que se está modelando. Sin eso, el contexto no explica por qué `QUEJAS_FAVOR_CONSUM_NOACEP_ENT` es el indicador más crítico.

**2. Hallazgos empíricos de S1 que cambian lo documentado**

* Duplicados: el contexto dice "~150" — la realidad es 609.514. Eso hay que corregirlo.
* Las 16 filas "completamente nulas" no eran nulas — eran mal parseadas. Corregir descripción.
* El salto anómalo de volumen en 2023 (de ~16k a 225k registros) no está documentado. Es un hallazgo crítico que condiciona todo el análisis temporal.
* 17.290 registros con doble reporte desde la misma fuente — no está documentado.

**3. Decisiones tomadas con su justificación**

* `UNIDAD_CAPTURA` es una dimensión analítica, no un problema de calidad.
* El dataset post-S1 tiene 434.579 filas × 20 columnas como punto de partida para EDA.

# contexto.md — Ruta B: Ciencia de Datos (sincronizado con `routeB.ipynb`)

## ICD Financiero S.A. | Intern Challenge Day 2026 - II

> Actualización de consistencia: 19/04/2026 (S0–S2 ejecutados en notebook)

---

## Deadline y entrega

- **Fecha límite:** Lunes 20/04/2026 — 11:59 AM
- **Enviar a:** juan.arciniegas@accenture.com | valentina.a.sanchez@accenture.com
- **Nombre de archivo:** `APELLIDO_NOMBRE_RutaB`

---

## El dataset

**Archivo:** `Challenge_DataAI_Candidatos.csv`

- Separador: `;`
- Encoding esperado: `utf-8-sig` (BOM)
- Registros fuente: `1.044.110`
- Rango temporal: 30/06/2014 — 31/03/2026 (cortes trimestrales)

### Parámetros de carga obligatorios

```python
df = pd.read_csv(DATA_PATH, sep=';', encoding='utf-8-sig', dtype={'CODIGO_ENTIDAD': str})
df.columns = df.columns.str.strip().str.rstrip(',')
df['QUEJAS_FAVOR_CONSUM_NOACEP_ENT'] = (
    df['QUEJAS_FAVOR_CONSUM_NOACEP_ENT']
    .str.rstrip(',').str.strip()
    .replace('', np.nan).astype(float)
)
```

---

## Columnas reales del CSV (confirmadas)

- Se mantiene la estructura original reportada en `contexto_Actualizado.md`.
- Las columnas del enunciado que no existen en CSV siguen ausentes:
  - `QUEJAS_FINALIZAD_FAVOR_CONSUM`
  - `QUEJAS_FINALIZAD_FAVOR_ENTIDAD`
  - `TIEMPO_PROMEDIO_EN_SISTEMA_(H)`

---

## Problemas de calidad — estado consistente con notebook

| #  | Problema                                          | Campo                              |     Volumen actualizado | Estado                                               |
| -- | ------------------------------------------------- | ---------------------------------- | ----------------------: | ---------------------------------------------------- |
| 1  | Fechas en formato mixto                           | `FECHA_CORTE`                    |                  ~1.500 | Corregido con `pd.to_datetime(..., dayfirst=True)` |
| 2  | Capitalización inconsistente                     | `NOMBRE_ENTIDAD`                 |                    ~900 | Normalizado a mayúsculas                            |
| 3  | Espacios extra                                    | `PRODUCTO`, `MOTIVO`           |                    ~600 | Corregido con `.str.strip()`                       |
| 4  | Inconsistencia lógica `N1 + N2 != FINALIZADAS` | múltiples                         |                  ~1.88% | Se documenta como limitación                        |
| 5  | Duplicados exactos                                | todas                              |       **609.514** | Eliminados en S1                                     |
| 6  | Outliers extremos                                 | `QUEJAS_EN_TRAMITE`              |            ~30 (>5.000) | Se conservan y documentan                            |
| 7  | Tipo mixto                                        | `CODIGO_ENTIDAD`                 |                    ~300 | Mitigado con `dtype=str`                           |
| 8  | Trailing commas en valores                        | `QUEJAS_FAVOR_CONSUM_NOACEP_ENT` |                  masivo | Corregido en carga                                   |
| 9  | Filas “nulas” reportadas inicialmente           | todas                              | **No eran nulas** | Eran filas mal parseadas                             |
| 10 | Filas mal parseadas por comillas                  | parser CSV                         |            **17** | Eliminadas en S1                                     |
| 11 | Doble reporte misma fuente                        | clave natural                      |        **17.290** | Se documenta como limitación                        |
| 12 | Salto de granularidad en 2023                     | temporal                           |                crítico | Confirmado (cambio de reporte)                       |

---

## Hallazgos críticos incorporados

1. **No eran 16 filas completamente nulas**: eran **17 filas mal parseadas** (17/18 columnas nulas por error de parser).
2. **Duplicados exactos reales**: **609.514** (no ~150).
3. **Cambio estructural en 2023**:
   - En serie depurada (post-S1): salto de ~14k a ~79k registros/año.
   - Se confirma cambio de granularidad (más combinaciones producto-motivo), no expansión proporcional del universo de entidades.
4. **Punto de partida analítico post-S1**: **434.579 filas × 20 columnas**.
5. **`UNIDAD_CAPTURA`**: dimensión analítica válida, no problema de calidad.

---

## Pregunta central — Ruta B

> ¿Qué combinaciones de entidad, producto y motivo concentran el mayor riesgo regulatorio —medido por tasa de resolución a favor del consumidor, acumulación de backlog y tendencia temporal— y qué puede anticipar ese riesgo?

---

## Variable objetivo (ajustada al dato real)

Como no existe `QUEJAS_FINALIZAD_FAVOR_CONSUM`, se usa proxy:

```python
TOTAL_FAVOR = QUEJAS_FAVOR_CONSUM_ACEP_ENT + QUEJAS_FAVOR_CONSUM_NOACEP_ENT
TASA_FAVOR = TOTAL_FAVOR / QUEJAS_FINALIZADAS
ALTO_RIESGO = (TASA_FAVOR > 0.70)
```

### Notas de cobertura (S2)

- `TASA_FAVOR` solo es calculable en **25.2%** de los registros post-S1.
- `ACEP + NOACEP == FINALIZADAS` se cumple aprox. **75.6%**.
- Sin casos `TASA_FAVOR > 1` en el subconjunto calculable.

---

## Métricas clave

```python
QUEJAS_NETAS = QUEJAS_RECIBIDAS - QUEJAS_FINALIZADAS
TASA_NO_ACEPTACION = NOACEP / (ACEP + NOACEP)
OUTLIER_EN_TRAMITE = 5_000
ANIO_INICIO_RIESGO = 2021
```

---

## Entregables requeridos

- Notebook `.ipynb` con historia analítica (EDA + modelado).
- Reporte de métricas con interpretación de negocio.
- Mínimo 5 visualizaciones con conclusión escrita.
- Documento mandatorio: top hallazgos + hipótesis + resumen ejecutivo (1 página).

---

## Criterio diferenciador (se mantiene)

Un modelo útil para supervisión (aunque no sea el más complejo) vale más que alta métrica sin contexto regulatorio ni acción.
