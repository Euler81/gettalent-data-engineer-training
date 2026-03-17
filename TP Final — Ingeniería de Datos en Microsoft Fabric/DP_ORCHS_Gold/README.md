# 🥇 DP_ORCHS_Gold

> Pipeline orquestador que construye el **modelo dimensional Gold** de Microsoft Fabric.  
> Ejecuta los 4 notebooks de transformación en secuencia estricta, enriquece con **APIs externas**  
> y entrega tablas listas para consumo por herramientas de Business Intelligence.

---

## 🗺️ Diagrama del Pipeline

```
lh_config.dbo.origin_gold
           │
           ▼
  ┌─────────────────────┐
  │   lk_origin_gold    │  Lookup — lee todos los notebooks Gold activos
  │                     │  ordenados por campo 'orden' (firstRowOnly: false)
  └──────────┬──────────┘
             │ Succeeded
             ▼
  ┌───────────────────────────────────────────────────────────────────┐
  │  fe_notebooks_gold  (ForEach — isSequential: true)                │
  │                                                                   │
  │  Iteración 1 — DimCalendario  ┐                                  │
  │  ┌────────────────────────┐   │                                  │
  │  │  nb_gold               │   │  notebookId: @item().notebook_id │
  │  │  NB_TRNSF_GoldVentas   │   │  defaultLakehouse: lh_gold       │
  │  │  _DimCalendario        │   │  timeout: 20min  retry: 2        │
  │  └──────────┬─────────────┘   │                                  │
  │             │ Succeeded       │                                  │
  │             ▼                 │                                  │
  │  ┌────────────────────────┐   │                                  │
  │  │  wt_between_notebooks  │   │  Wait — 180 segundos             │
  │  └──────────┬─────────────┘   │                                  │
  │             │                 │                                  │
  │  Iteración 2 — DimCliente   ──┘                                  │
  │  Iteración 3 — DimProducto                                       │
  │  Iteración 4 — FactVentas                                        │
  │             (mismo patrón: nb_gold → Wait 180s)                  │
  └───────────────────────────────────────────────────────────────────┘
             │ Failed
             ▼
  ┌─────────────────────┐
  │     fail_gold       │  Fail — código GOLD_FAIL
  └─────────────────────┘
```

---

## ⚙️ Actividades

| # | Actividad | Tipo | Descripción |
|---|-----------|------|-------------|
| 1 | `lk_origin_gold` | Lookup | Lee `lh_config.dbo.origin_gold` ordenada por `orden`. `firstRowOnly: false` |
| 2 | `fe_notebooks_gold` | ForEach | Itera cada notebook en secuencia (`isSequential: true`) |
| 3 | `nb_gold` | Notebook | Ejecuta `@item().notebook_id` con `lh_gold` como default lakehouse |
| 4 | `wt_between_notebooks` | Wait | **180 segundos** entre cada notebook — libera sesión Livy |
| 5 | `fail_gold` | Fail | Se activa si el ForEach falla. Código: `GOLD_FAIL` |

---

## 📋 Configuración — `lh_config.dbo.origin_gold`

El pipeline resuelve **dinámicamente** qué notebook ejecutar en cada iteración.  
`notebookId: @item().notebook_id` — cero hardcodeos de IDs en el pipeline.

```sql
SELECT *
FROM lh_config.dbo.origin_gold
WHERE activo = 1
ORDER BY orden
```

| `orden` | `nombre_notebook` | `notebook_id` | `modelo` | `nombre_tabla` |
|---------|-------------------|---------------|----------|---------------|
| 1 | `NB_TRNSF_GoldVentas_DimCalendario` | `e9e0a8f4-...` | `ventas` | `d_calendario` |
| 2 | `NB_TRNSF_GoldVentas_DimCliente` | `5938d1ac-...` | `ventas` | `d_cliente` |
| 3 | `NB_TRNSF_GoldVentas_DimProducto` | `dde44e7e-...` | `ventas` | `d_producto` |
| 4 | `NB_TRNSF_GoldVentas_FactVentas` | `ed7a53a9-...` | `ventas` | `f_ventas` |

> **¿Por qué este orden?** `FactVentas` hace JOINs con las 3 dimensiones. Deben existir antes.

---

## 🔗 Lakehouses configurados en el notebook `nb_gold`

```json
{
  "defaultLakehouse": { "id": "902f27e9-...", "name": "lh_gold" },
  "additionalLakehouses": [
    { "id": "4d897e0a-...", "name": "lh_silver" },
    { "id": "bcc2be74-...", "name": "lh_config" }
  ]
}
```

Todos los notebooks Gold tienen acceso a `lh_gold`, `lh_silver` y `lh_config` en cada ejecución.

---

## ⏱️ Por qué el Wait de 180 segundos

> El entorno **Trial de Microsoft Fabric** tiene capacidad limitada de sesiones Spark/Livy.  
> Sin la espera, el segundo notebook intenta iniciar antes de que la sesión del primero se libere,  
> generando error **HTTP 430 — Too Many Requests**.

```
DimCalendario (8 celdas, 2 APIs externas) ──▶  Wait 180s  ──▶  DimCliente
DimCliente    (5 celdas)                  ──▶  Wait 180s  ──▶  DimProducto
DimProducto   (5 celdas)                  ──▶  Wait 180s  ──▶  FactVentas
FactVentas    (5 celdas, 2 queries SQL)   ──▶  (fin)
```

Los 180s de Gold son más amplios que los 90s/180s de SFTP porque `DimCalendario`
realiza múltiples llamadas HTTP a APIs externas que pueden tardar 10-20 segundos adicionales.

---

## 📓 Notebooks

---

### 📅 `NB_TRNSF_GoldVentas_DimCalendario`

**Propósito:** Generar la dimensión de tiempo completa enriquecida con feriados nacionales argentinos y datos climáticos históricos.

| Celda | Función |
|-------|---------|
| 0 — Imports | `requests`, `json`, `datetime` |
| 1 — Config | Lee `lh_config.dbo.origin_gold`, construye `tabla_cal` dinámicamente |
| 2 — Tabla base | Script de profesores: `sequence()` de fechas 2020 → fin de año actual |
| 3 — API Feriados 🇦🇷 | Llama a `argentinadatos.com/v1/feriados/{año}` — construye DataFrame de feriados |
| 4 — MERGE Feriados | MERGE sobre `d_calendario` usando `t.Fecha = f.Fecha_fer` |
| 5 — API Clima 🌡️ | Llama a `Open-Meteo archive` — temperatura histórica Buenos Aires 2011→hoy |
| 6 — MERGE Clima | MERGE sobre `d_calendario` con temperatura media, máxima y mínima |
| 7 — Verificar | COUNT, filas con feriado, cobertura de temperatura |

**Estructura base generada (script de profesores):**

```sql
CREATE TABLE lh_gold.ventas.d_calendario
USING DELTA AS
WITH dates AS (
    SELECT explode(
        sequence(to_date('2020-01-01'),
                 last_day(make_date(year(current_date()), 12, 1)),
                 interval 1 day)
    ) AS Fecha
)
SELECT
    Fecha,
    day(Fecha)                  AS Nro_Dia,
    -- Lunes / Martes / ... / Domingo
    CASE dayofweek(Fecha) ...   AS Dia,
    -- Enero / Febrero / ... / Diciembre
    CASE month(Fecha) ...       AS Mes,
    year(Fecha)                 AS Year,
    concat('S', weekofyear(Fecha)) AS Semana,
    (year*10000 + month*100 + day)  AS DateKeyOrder,
    date_trunc('week', Fecha)   AS StartOfWeek,
    concat(year, lpad(month,2,'0')) AS CodTiempo,
    concat('Ene-2024', ...)     AS Mes_Year,
    quarter(Fecha)              AS QuarterNumber,
    concat('Q', quarter(Fecha)) AS Trimestre,
    FALSE                       AS es_feriado,
    NULL                        AS nombre_feriado
FROM dates
```

**Columnas completas de `d_calendario`:**

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `Fecha` | `date` | Clave primaria |
| `Nro_Dia` | `int` | Número de día del mes |
| `Dia` | `string` | Lunes / Martes ... Domingo |
| `Nro_Mes` | `string` | 01 a 12 (con cero) |
| `Mes` | `string` | Enero / Febrero ... Diciembre |
| `MesCorto` | `string` | Ene / Feb ... Dic |
| `Year` | `int` | Año |
| `Semana` | `string` | S1 ... S53 |
| `DateKeyOrder` | `int` | AAAAMMDD para ordenamiento |
| `StartOfWeek` | `date` | Lunes de la semana |
| `EndOfWeek` | `date` | Domingo de la semana |
| `Semana_Year` | `string` | S12 (semana del año) |
| `QuarterNumber` | `int` | 1 a 4 |
| `Trimestre` | `string` | Q1 / Q2 / Q3 / Q4 |
| `CodeMes` | `string` | 03/24 |
| `CodTiempo` | `string` | 202403 |
| `Mes_Year` | `string` | Mar-2024 |
| `es_feriado` | `boolean` | Desde API ArgentinaDatos 🇦🇷 |
| `nombre_feriado` | `string` | Nombre del feriado (si aplica) |
| `tempMax` | `float` | Temperatura máxima — Open-Meteo 🌡️ |
| `tempMin` | `float` | Temperatura mínima — Open-Meteo 🌡️ |
| `tempMedia` | `float` | Temperatura media — Open-Meteo 🌡️ |

**APIs externas:**

```python
# 🇦🇷 API Feriados Argentina — obligatorio TP
url = f'https://api.argentinadatos.com/v1/feriados/{anio}'
resp = requests.get(url, timeout=10)
# devuelve: [{"fecha": "2024-01-01", "nombre": "Año Nuevo", "tipo": "inamovible"}, ...]

# 🌡️ API Open-Meteo — BONUS
BASE_URL = 'https://archive-api.open-meteo.com/v1/archive'  # histórico
FORECAST_URL = 'https://api.open-meteo.com/v1/forecast'      # días recientes
# parámetros: latitude=-34.6, longitude=-58.4, daily=temperature_2m_max/min/mean
# cobertura: 2011 → hoy (AÑO_INICIO = 2011 cubre todo el rango de f_ventas)
```

---

### 👤 `NB_TRNSF_GoldVentas_DimCliente`

**Propósito:** Transformar `lh_silver.SFTP.Person` en la dimensión `d_cliente` con atributos desnormalizados en español.

| Celda | Función |
|-------|---------|
| 0 — Config | `lakehouse.get('lh_gold')` + `lakehouse.get('lh_silver')`, construye `abfs_gold` dinámicamente desde `origin_gold` |
| 1 — Leer Silver | `SELECT * FROM lh_silver.SFTP.Person LIMIT 1000` |
| 2 — Transformar | Renombrar columnas + construir `nombre_completo` |
| 3 — Escribir Gold | `notebookutils.fs.rm(abfs_gold)` + `.write.overwrite.save(abfs_gold)` |
| 4 — Verificar | `spark.read.format("delta").load(abfs_gold)` |

**Transformaciones aplicadas:**

```python
df_dim = (df_silver
    .withColumnRenamed("BusinessEntityID", "id_cliente")
    .withColumn("nombre_completo",
        F.concat_ws(" ",
            F.col("FirstName"),
            F.when(F.col("MiddleName").isNotNull(), F.col("MiddleName")).otherwise(F.lit("")),
            F.col("LastName")
        )
    )
    .withColumnRenamed("PersonType",     "tipo_persona")
    .withColumnRenamed("EmailPromotion", "acepta_email_promo")
    .withColumnRenamed("ModifiedDate",   "fecha_modificacion")
    .select("id_cliente", "nombre_completo", "tipo_persona",
            "acepta_email_promo", "fecha_modificacion")
)
```

**Esquema final `d_cliente`:**

| Campo | Tipo | Origen |
|-------|------|--------|
| `id_cliente` | `integer` | `BusinessEntityID` |
| `nombre_completo` | `string` | `concat_ws(FirstName, MiddleName, LastName)` |
| `tipo_persona` | `string` | `PersonType` — SC / IN / SP / EM / GC |
| `acepta_email_promo` | `integer` | `EmailPromotion` — 0 / 1 / 2 |
| `fecha_modificacion` | `timestamp` | `ModifiedDate` |

---

### 📦 `NB_TRNSF_GoldVentas_DimProducto`

**Propósito:** Transformar `lh_silver.ADVWDB.Product` en la dimensión `d_producto` con nomenclatura en español.

| Celda | Función |
|-------|---------|
| 0 — Config | `lakehouse.get('lh_gold')` + `lakehouse.get('lh_silver')`, paths dinámicos desde `origin_gold` |
| 1 — Leer Silver | `SELECT * FROM lh_silver.ADVWDB.Product LIMIT 1000` |
| 2 — Transformar | 14 renombrados a español + selección de columnas relevantes |
| 3 — Escribir Gold | `notebookutils.fs.rm(abfs_gold)` + `.write.overwrite.save(abfs_gold)` |
| 4 — Verificar | Ordenado por `id_producto` con `precio_lista` y `linea_producto` |

**Renombrados completos (14 columnas → español):**

| Origen (Silver) | Destino (Gold) |
|-----------------|----------------|
| `ProductID` | `id_producto` |
| `Name` | `nombre_producto` |
| `ProductNumber` | `codigo_producto` |
| `Color` | `color` |
| `StandardCost` | `costo_estandar` |
| `ListPrice` | `precio_lista` |
| `Size` | `talla` |
| `ProductLine` | `linea_producto` |
| `Class` | `clase` |
| `Style` | `estilo` |
| `SellStartDate` | `fecha_inicio_venta` |
| `SellEndDate` | `fecha_fin_venta` |
| `DiscontinuedDate` | `fecha_discontinuacion` |
| `ModifiedDate` | `fecha_modificacion` |

> `linea_producto`: `R` = Road · `M` = Mountain · `T` = Touring · `S` = Other Sales

---

### 📊 `NB_TRNSF_GoldVentas_FactVentas`

**Propósito:** Construir la tabla de hechos central mediante JOIN de las dos fuentes Silver y ejecutar queries analíticas de negocio.

| Celda | Función |
|-------|---------|
| 0 — Config | `lakehouse.get()` + lee `origin_bronze_silver` y `origin_gold` para construir todos los nombres de tabla dinámicamente |
| 1 — JOIN Silver | `SalesOrderDetail INNER JOIN SalesOrderHeader ON SalesOrderID` → 13 campos seleccionados |
| 2 — Escribir Gold | `DROP TABLE IF EXISTS` + `saveAsTable` + `USE lh_gold.ventas` para SQL 1-part |
| 3 — Analítica 1 | **Top 10 clientes** semana actual vs semana anterior |
| 4 — Analítica 2 | **Ventas por producto y mes** con JOIN a `d_calendario` y `d_producto` |

**JOIN que construye `f_ventas`:**

```python
df_fact = (
    df_sod.alias('sod')                         # lh_silver.ADVWDB.SalesOrderDetail
    .join(df_soh.alias('soh'), 'SalesOrderID', 'inner')  # lh_silver.SFTP.SalesOrderHeader
    .select(
        F.col('sod.SalesOrderDetailID').alias('id_detalle'),    # PK
        F.col('sod.SalesOrderID').alias('id_orden'),
        F.col('sod.ProductID').alias('id_producto'),            # FK → d_producto
        F.col('soh.CustomerID').alias('id_cliente'),            # FK → d_cliente
        F.col('soh.OrderDate').alias('fecha_orden'),            # FK → d_calendario
        F.col('soh.TerritoryID').alias('id_territorio'),
        F.col('sod.OrderQty').alias('cantidad'),
        F.col('sod.UnitPrice').alias('precio_unitario'),
        F.col('sod.UnitPriceDiscount').alias('descuento'),
        F.col('sod.LineTotal').alias('total_linea'),
        F.col('soh.TotalDue').alias('total_orden'),
        F.col('soh.OnlineOrderFlag').alias('es_online'),
        F.col('sod.ModifiedDate').alias('fecha_modificacion')
    )
)
```

**Query 1 — Top 10 clientes semana actual vs anterior:**

```sql
CREATE OR REPLACE TEMP VIEW vw_top_clientes AS
WITH semanas AS (
    SELECT
        date_trunc('week', current_date())         AS inicio_semana_actual,
        date_add(date_trunc('week', current_date()), -7) AS inicio_semana_anterior
),
ventas_por_semana AS (
    SELECT
        f.id_cliente,
        CASE
            WHEN f.fecha_orden >= s.inicio_semana_actual                    THEN 'semana_actual'
            WHEN f.fecha_orden >= s.inicio_semana_anterior
             AND f.fecha_orden <  s.inicio_semana_actual                    THEN 'semana_anterior'
        END AS semana,
        SUM(f.total_linea)         AS total_ventas,
        COUNT(DISTINCT f.id_orden) AS cant_ordenes
    FROM f_ventas f
    CROSS JOIN semanas s
    WHERE f.fecha_orden >= s.inicio_semana_anterior
    GROUP BY f.id_cliente, semana
)
SELECT
    id_cliente,
    MAX(CASE WHEN semana = 'semana_actual'   THEN total_ventas  ELSE 0 END) AS ventas_semana_actual,
    MAX(CASE WHEN semana = 'semana_anterior' THEN total_ventas  ELSE 0 END) AS ventas_semana_anterior,
    MAX(CASE WHEN semana = 'semana_actual'   THEN cant_ordenes  ELSE 0 END) AS ordenes_semana_actual,
    MAX(CASE WHEN semana = 'semana_anterior' THEN cant_ordenes  ELSE 0 END) AS ordenes_semana_anterior
FROM ventas_por_semana
GROUP BY id_cliente
ORDER BY ventas_semana_actual DESC
LIMIT 10
```

**Query 2 — Ventas por producto y mes:**

```sql
SELECT
    c.CodTiempo                       AS periodo,
    c.Mes_Year                        AS mes_anio,
    p.nombre_producto,
    p.linea_producto,
    COUNT(DISTINCT f.id_orden)        AS cant_ordenes,
    SUM(f.cantidad)                   AS unidades_vendidas,
    ROUND(SUM(f.total_linea), 2)      AS total_ventas,
    ROUND(AVG(f.precio_unitario), 2)  AS precio_promedio
FROM f_ventas f
JOIN d_calendario c ON f.fecha_orden  = c.Fecha
JOIN d_producto   p ON f.id_producto  = p.id_producto
GROUP BY c.CodTiempo, c.Mes_Year, p.nombre_producto, p.linea_producto
ORDER BY c.CodTiempo DESC, total_ventas DESC
LIMIT 1000
```

> **¿Por qué `USE lh_gold.ventas`?** Spark SQL no acepta nombres de 3 partes (`lh_gold.ventas.f_ventas`)
> en cláusulas `FROM` dentro de JOINs complejos. El `USE` establece el catálogo activo y permite
> referenciar las tablas con 1 parte (`f_ventas`, `d_calendario`, `d_producto`).

---

## 🌟 Modelo Dimensional Gold — Esquema en Estrella

```
                    ┌──────────────────────────┐
                    │      d_calendario        │
                    │──────────────────────────│
                    │ Fecha (PK)               │
                    │ CodTiempo / Mes_Year      │
                    │ Dia / Mes / Year          │
                    │ Semana / Trimestre        │
                    │ es_feriado 🇦🇷           │
                    │ nombre_feriado            │
                    │ tempMax/Min/Media 🌡️     │
                    └──────────┬───────────────┘
                               │ fecha_orden = Fecha
                               │
┌─────────────────┐   ┌────────┴──────────────┐   ┌──────────────────────┐
│   d_cliente     │   │      f_ventas          │   │     d_producto       │
│─────────────────│   │───────────────────────│   │──────────────────────│
│ id_cliente (PK) │◀──│ id_cliente (FK)        │──▶│ id_producto (PK)     │
│ nombre_completo │   │ id_producto (FK)       │   │ nombre_producto      │
│ tipo_persona    │   │ fecha_orden (FK)       │   │ codigo_producto      │
│ acepta_email    │   │ id_detalle (PK)        │   │ linea_producto       │
│ fecha_modif.    │   │ id_orden               │   │ color / talla        │
└─────────────────┘   │ id_territorio          │   │ precio_lista         │
                       │ cantidad               │   │ costo_estandar      │
                       │ precio_unitario        │   │ fecha_inicio_venta  │
                       │ descuento              │   └──────────────────────┘
                       │ total_linea            │
                       │ total_orden            │
                       │ es_online              │
                       └───────────────────────┘
```

---

## 🔄 Flujo de Datos Detallado

```
lh_silver.SFTP.Person              lh_silver.ADVWDB.Product
       │                                    │
       │  [DimCliente]                      │  [DimProducto]
       ▼                                    ▼
lh_gold.ventas.d_cliente       lh_gold.ventas.d_producto

lh_silver.ADVWDB.SalesOrderDetail  +  lh_silver.SFTP.SalesOrderHeader
                    │  INNER JOIN ON SalesOrderID
                    │  [FactVentas]
                    ▼
           lh_gold.ventas.f_ventas
                    │
                    │  JOINs analíticos
                    ├──▶ d_calendario  (fecha_orden = Fecha)
                    └──▶ d_producto    (id_producto = id_producto)

                                              ↑
lh_silver  →  [DimCalendario]  →  lh_gold.ventas.d_calendario
         + API Feriados AR 🇦🇷
         + API Open-Meteo   🌡️
```

---

## 🛡️ Política de Ejecución

| Actividad | Timeout | Retry | Retry Delay |
|-----------|---------|-------|-------------|
| `nb_gold` (todos los notebooks) | **20 minutos** | **2** | **60 segundos** |
| `wt_between_notebooks` | N/A | N/A | 180 seg fijos |

> El timeout de 20 minutos es suficiente para `DimCalendario` (el notebook más largo) que incluye
> generación de ~2000+ fechas + 2 llamadas a APIs externas + 2 operaciones MERGE Delta.

---

## 🛡️ Manejo de Errores

| Escenario | Comportamiento |
|-----------|---------------|
| Silver vacío (0 filas) | El notebook lanza `Exception` explícita con mensaje descriptivo |
| API Feriados no disponible | `try/except` → imprime advertencia y continúa sin MERGE de feriados |
| API Open-Meteo no disponible | `try/except` → imprime advertencia y continúa sin MERGE de clima |
| Fallo en cualquier notebook | Retry automático x2 con 60s de delay |
| Fallo tras 2 retries | `fail_gold` se activa → expone error `GOLD_FAIL` en Monitor de Fabric |
| `FactVentas` sin dimensiones | No ocurre: el ForEach secuencial garantiza que las dimensiones ya existen |

---

## 📊 Runs Verificados en Producción

| Fecha | Actividades | Duración | Estado |
|-------|------------|---------|--------|
| 09/03/2026 17:11 | 10 / 10 | 20m 46s | ✅ Succeeded |

### Desglose de actividades (10 total)

```
lk_origin_gold                      →  1 actividad
fe_notebooks_gold (ForEach × 4)
  └── Por notebook (× 4):
      nb_gold                       →  4 actividades  (DimCalendario + DimCliente + DimProducto + FactVentas)
      wt_between_notebooks          →  4 actividades  (Wait 180s entre cada uno)
fail_gold (no disparado)            →  0 actividades
      (Wait final de FactVentas no existe — el pipeline termina luego del último notebook)
──────────────────────────────────────────────────────────────────────────
Nota: el ForEach ejecuta nb_gold → Wait → nb_gold → Wait ... pero el último
      Wait después de FactVentas igualmente se ejecuta. Total: 4 nb + 4 waits = 8
      + lk_origin_gold + fe_notebooks_gold = 10 actividades ✅
```

### Tablas Gold generadas

| Tabla | Filas | Notas |
|-------|-------|-------|
| `lh_gold.ventas.d_calendario` | 2000+ | Incluye feriados 🇦🇷 y temperatura 🌡️ |
| `lh_gold.ventas.d_cliente` | ~1000 | Desde Silver Person |
| `lh_gold.ventas.d_producto` | ~1000 | Desde Silver Product |
| `lh_gold.ventas.f_ventas` | ~1000 | JOIN SOD × SOH |

---

## 🔗 Recursos Relacionados

| Recurso | Descripción |
|---------|-------------|
| [`DP_ORCHS_Origenes`](../DP_ORCHS_Origenes/) | Orquestador maestro que invoca este pipeline |
| [`NB_TRNSF_GoldVentas_DimCalendario.ipynb`](./notebooks/NB_TRNSF_GoldVentas_DimCalendario.ipynb) | Notebook DimCalendario |
| [`NB_TRNSF_GoldVentas_DimCliente.ipynb`](./notebooks/NB_TRNSF_GoldVentas_DimCliente.ipynb) | Notebook DimCliente |
| [`NB_TRNSF_GoldVentas_DimProducto.ipynb`](./notebooks/NB_TRNSF_GoldVentas_DimProducto.ipynb) | Notebook DimProducto |
| [`NB_TRNSF_GoldVentas_FactVentas.ipynb`](./notebooks/NB_TRNSF_GoldVentas_FactVentas.ipynb) | Notebook FactVentas |
| [`DP_ORCHS_Gold.json`](./DP_ORCHS_Gold.json) | JSON del pipeline |
| [API ArgentinaDatos](https://argentinadatos.com/api/v1/feriados/) | Feriados nacionales |
| [API Open-Meteo](https://api.open-meteo.com) | Temperatura histórica |

---

*TP Final — Ingeniería de Datos en Microsoft Fabric — Euler, Diego — Marzo 2026*
