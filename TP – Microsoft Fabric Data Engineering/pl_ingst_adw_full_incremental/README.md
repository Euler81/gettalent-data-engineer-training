# 🏭 pl_ingst_adw_full_incremental

> Pipeline de ingesta desde **SQL Server AdventureWorks** hacia **Microsoft Fabric Lakehouse**.  
> Soporta carga **FULL** (overwrite) e **INCREMENTAL** (append por rango de fechas) de forma dinámica,  
> sin hardcodeos: todas las tablas y estrategias se leen desde una tabla de configuración centralizada.

---

## 🗺️ Diagrama del Pipeline

```
  Parámetros opcionales
  fechaDesde / fechaHasta
           │
           ▼
  ┌──────────────────────────────┐
  │      LKUP_Get_Tables         │  Lookup — lee config.table_load_config
  │                              │  desde lh_config (firstRowOnly: false)
  │  timeout: 12h  retry: 3      │
  └──────────────┬───────────────┘
                 │ Succeeded
                 ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │  FEACH_Process_Tables  (ForEach — batchCount: 5, paralelo)       │
  │                                                                  │
  │  Por cada tabla en config.table_load_config:                     │
  │                                                                  │
  │  ┌──────────────────────────────────────────────────────────┐    │
  │  │  CJ_INGST_Dynamic_Table  (Copy)                          │    │
  │  │                                                          │    │
  │  │  SQL Server AdventureWorks2019                           │    │
  │  │    → query dinámica (FULL o INCREMENTAL)                 │    │
  │  │    → lh_landing/Files/proyecto/fuente/{schema}/{tabla}/  │    │
  │  │      {tabla}_{fecha}.parquet  (Snappy)                   │    │
  │  └───────────────────────────┬──────────────────────────────┘    │
  │                              │ Succeeded                         │
  │                              ▼                                   │
  │  ┌──────────────────────────────────────────────────────────┐    │
  │  │  CJ_CONVERTIR_Bronze  (Copy)                             │    │
  │  │                                                          │    │
  │  │  lh_landing/Files/proyecto/fuente/{schema}/{tabla}/*.parquet  │
  │  │    → lh_bronze/Tables/bronze.br_{schema}_{tabla}         │    │
  │  │    → FULL: Overwrite  /  INCREMENTAL: Append             │    │
  │  │    → applyVOrder: true  /  mergeSchema: true             │    │
  │  │    → metadata: ingestion_date_ts, pl_run_id,             │    │
  │  │                process_date, load_type, exercise_number   │    │
  │  └──────────────────────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────────────────────┘
                 │ Succeeded
                 ▼
  ┌──────────────────────────────┐
  │  CJ_EXTRAER_Metadata_Sales   │  Copy — extrae catálogo de tablas
  │                              │  del schema Sales de AdventureWorks
  │  → lh_landing/Files/         │  → CSV datado por ejecución
  │    proyecto/metadata/        │
  │    sales_tables/AAAAMMDD/    │
  │    sales_tables_AAAAMMDD.csv │
  └──────────────────────────────┘
```

---

## ⚙️ Actividades

| # | Actividad | Tipo | `dependsOn` | Descripción |
|---|-----------|------|------------|-------------|
| 1 | `LKUP_Get_Tables` | Lookup | `[]` | Lee todas las tablas activas desde `config.table_load_config` en `lh_config` |
| 2 | `FEACH_Process_Tables` | ForEach | `[LKUP Succeeded]` | Procesa tablas en paralelo (`batchCount: 5`) |
| 3 | `CJ_INGST_Dynamic_Table` | Copy (inner) | `[]` | SQL Server → Parquet en `lh_landing` con query dinámica |
| 4 | `CJ_CONVERTIR_Bronze` | Copy (inner) | `[CJ_INGST Succeeded]` | Parquet en `lh_landing` → tabla Delta en `lh_bronze` |
| 5 | `CJ_EXTRAER_Metadata_Sales` | Copy | `[ForEach Succeeded]` | Extrae catálogo del schema `Sales` → CSV en `lh_landing` |

---

## 🔧 Parámetros del Pipeline

| Parámetro | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `fechaDesde` | `string` | _(vacío = ayer)_ | Fecha inicio del rango incremental. Formato: `YYYY-MM-DD` |
| `fechaHasta` | `string` | _(vacío = hoy)_ | Fecha fin del rango incremental. Formato: `YYYY-MM-DD` |
| `pl_run_id` | `string` | `@pipeline().RunId` | ID de ejecución — se propaga a la metadata de Bronze |

> Si `fechaDesde` y `fechaHasta` están vacíos, el pipeline usa **ayer** como inicio y **hoy** como fin de forma automática.

---

## 🔧 Parámetros ARM Template (despliegue)

| Parámetro | Descripción |
|-----------|-------------|
| `lh_config` | `artifactId` del lakehouse de configuración |
| `conn_adventureworks` | ID de la conexión SQL Server AdventureWorks2019 |
| `lh_landing` | `artifactId` del lakehouse landing |
| `lh_bronze` | `artifactId` del lakehouse bronze |

---

## 📋 Tabla de Configuración — `lh_config.config.table_load_config`

El pipeline lee **dinámicamente** qué tablas procesar y con qué estrategia.  
Agregar una nueva tabla al pipeline requiere únicamente insertar un registro en esta tabla.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `schema_name` | `string` | Schema en SQL Server (ej: `Sales`, `Production`) |
| `table_name` | `string` | Nombre de la tabla (ej: `SalesOrderDetail`) |
| `load_type` | `string` | `FULL` = overwrite total / `INCREMENTAL` = append por fecha |
| `source_query_template` | `string` | Query personalizada con `@{fecha_desde}` y `@{fecha_hasta}`. Si está vacía usa `SELECT * FROM schema.tabla` |
| `exercise_number` | `string` | Identificador del ejercicio — se propaga a metadata de Bronze |

---

## 🔀 Lógica de la Query Dinámica — `CJ_INGST_Dynamic_Table`

La query SQL que se ejecuta sobre AdventureWorks se construye en tiempo de ejecución:

```
SI source_query_template está vacío:
    → SELECT * FROM {schema_name}.{table_name}

SI source_query_template tiene valor:
    → Reemplaza @{fecha_desde} con:
        pipeline().parameters.fechaDesde  (si no está vacío)
        formatDateTime(addDays(utcnow(), -1), 'yyyy-MM-dd')  (si está vacío = ayer)
    → Reemplaza @{fecha_hasta} con:
        pipeline().parameters.fechaHasta  (si no está vacío)
        formatDateTime(utcnow(), 'yyyy-MM-dd')  (si está vacío = hoy)
```

**Expresión Data Factory completa:**

```
@if(
  or(empty(item().source_query_template),
     equals(trim(item().source_query_template), '')),

  concat('SELECT * FROM ', item().schema_name, '.', item().table_name),

  replace(
    replace(
      item().source_query_template,
      '@{fecha_desde}',
      if(empty(pipeline().parameters.fechaDesde),
         formatDateTime(addDays(utcnow(), -1), 'yyyy-MM-dd'),
         pipeline().parameters.fechaDesde)
    ),
    '@{fecha_hasta}',
    if(empty(pipeline().parameters.fechaHasta),
       formatDateTime(utcnow(), 'yyyy-MM-dd'),
       pipeline().parameters.fechaHasta)
  )
)
```

**Ejemplos de queries resultantes:**

```sql
-- FULL sin template → trae todo
SELECT * FROM Production.Product

-- INCREMENTAL con template personalizado
SELECT * FROM Sales.SalesOrderHeader
WHERE OrderDate >= '2024-03-01' AND OrderDate <= '2024-03-31'

-- INCREMENTAL sin parámetros → usa ayer/hoy automáticamente
SELECT * FROM Sales.SalesOrderDetail
WHERE ModifiedDate >= '2024-03-19' AND ModifiedDate <= '2024-03-20'
```

---

## 📁 Estructura de Archivos en `lh_landing`

### Archivos de datos (Parquet)

```
lh_landing/Files/proyecto/fuente/
├── sales/
│   ├── SalesOrderDetail/
│   │   ├── 20240320/                     ← INCREMENTAL: carpeta por fecha
│   │   │   └── SalesOrderDetail_20240320.parquet
│   │   └── 20240321/
│   │       └── SalesOrderDetail_20240321.parquet
│   └── SalesOrderHeader/
│       └── 20240320/
│           └── SalesOrderHeader_20240320.parquet
└── production/
    └── Product/
        └── 20240320/                     ← FULL: carpeta por fecha de ejecución
            └── Product_20240320.parquet
```

**Expresión de fileName:**
```
@concat(
  item().table_name, '_',
  if(equals(item().load_type, 'INCREMENTAL'),
     replace(fechaHasta, '-', ''),          ← fecha del rango: 20240320
     formatDateTime(utcnow(), 'yyyyMMdd')   ← fecha de ejecución: 20240320
  ),
  '.parquet'
)
```

**Expresión de folderPath:**
```
@concat(
  'proyecto/fuente/', toLower(item().schema_name), '/', item().table_name, '/',
  if(equals(item().load_type, 'INCREMENTAL'),
     replace(fechaHasta, '-', ''),          ← 20240320
     formatDateTime(utcnow(), 'yyyyMMdd')   ← 20240320
  )
)
```

> **Formato:** Parquet con compresión **Snappy** y VertiParquet habilitado.

### Archivos de metadata (CSV)

```
lh_landing/Files/proyecto/metadata/sales_tables/
└── 20240320/
    └── sales_tables_20240320.csv
```

---

## 🥉 Tabla Bronze — `CJ_CONVERTIR_Bronze`

### Nomenclatura de tablas

```
bronze.br_{schema}_{tabla}

Ejemplos:
  bronze.br_sales_salesorderdetail
  bronze.br_sales_salesorderheader
  bronze.br_production_product
```

**Expresión:**
```
@concat('br_', toLower(item().schema_name), '_', toLower(item().table_name))
```

### Estrategia por tipo de carga

| `load_type` | `tableActionOption` | Comportamiento |
|-------------|--------------------|-|
| `FULL` | `Overwrite` | Reemplaza toda la tabla Bronze en cada ejecución |
| `INCREMENTAL` | `Append` | Agrega los registros del rango al acumulado histórico |

### Propiedades Delta

| Propiedad | Valor | Descripción |
|-----------|-------|-------------|
| `applyVOrder` | `true` | Optimiza la compresión y velocidad de lectura en OneLake |
| `mergeSchema` | `true` | Acepta nuevas columnas sin fallar si el schema evoluciona |
| `partitionOption` | `None` | Sin particionamiento en Bronze |

### Columnas de metadata agregadas automáticamente

| Columna | Valor | Descripción |
|---------|-------|-------------|
| `ingestion_date_ts` | `@utcnow()` | Timestamp exacto de la ingesta |
| `pl_run_id` | `@pipeline().RunId` | ID único del run del pipeline |
| `process_date` | `@formatDateTime(utcnow(), 'yyyy-MM-dd')` | Fecha de procesamiento |
| `load_type` | `@item().load_type` | `FULL` o `INCREMENTAL` |
| `exercise_number` | `@item().exercise_number` | Identificador del ejercicio |

---

## 🔀 ForEach — Ejecución Paralela

```
FEACH_Process_Tables
  isSequential: false  ← NO secuencial
  batchCount: 5        ← hasta 5 tablas en paralelo simultáneamente
```

> Con `batchCount: 5`, si `table_load_config` tiene 10 tablas activas,  
> el pipeline procesa las primeras 5 en paralelo, luego las siguientes 5.  
> Esto reduce significativamente el tiempo total de ejecución vs. procesamiento secuencial.

---

## 📊 `CJ_EXTRAER_Metadata_Sales` — Catálogo de Tablas

Al finalizar el ForEach, el pipeline extrae automáticamente el catálogo completo del schema `Sales`:

```sql
SELECT
    s.name  AS schema_name,
    t.name  AS table_name,
    CASE WHEN t.type = 'U' THEN 'BASE TABLE'
         WHEN t.type = 'V' THEN 'VIEW'
    END     AS table_type,
    t.create_date,
    t.modify_date
FROM sys.tables t
INNER JOIN sys.schemas s ON t.schema_id = s.schema_id
WHERE s.name = 'Sales'
ORDER BY t.name
```

**Output:** CSV delimitado por coma, primera fila como header, nombrado `sales_tables_AAAAMMDD.csv`.  
**Uso:** Inventario y auditoría del origen — permite detectar nuevas tablas o cambios en el schema.

---

## 🔄 Flujo de Datos Completo

```
SQL Server AdventureWorks2019
├── Sales.SalesOrderDetail    → INCREMENTAL (WHERE ModifiedDate entre fechas)
├── Sales.SalesOrderHeader    → INCREMENTAL (WHERE OrderDate entre fechas)
├── Production.Product        → FULL        (SELECT * FROM)
└── ...otras tablas de config

         │
         │  [CJ_INGST_Dynamic_Table]
         │  Query dinámica desde config
         │  Output: Parquet + Snappy
         ▼
lh_landing/Files/proyecto/fuente/{schema}/{tabla}/{fecha}/
         {tabla}_{fecha}.parquet

         │
         │  [CJ_CONVERTIR_Bronze]
         │  Lee todos los Parquet de la carpeta (wildcard *.parquet)
         │  FULL → Overwrite / INCREMENTAL → Append
         │  Agrega columnas de metadata
         ▼
lh_bronze/Tables/bronze.br_{schema}_{tabla}
(Delta Lake — applyVOrder — mergeSchema)

         │  [Post ForEach]
         │  [CJ_EXTRAER_Metadata_Sales]
         ▼
lh_landing/Files/proyecto/metadata/sales_tables/{fecha}/
         sales_tables_{fecha}.csv
```

---

## 🛡️ Políticas de Ejecución

| Actividad | Timeout | Retry | Retry Delay |
|-----------|---------|-------|-------------|
| `LKUP_Get_Tables` | 12h | 3 | 60 seg |
| `CJ_INGST_Dynamic_Table` | _(default)_ | 3 | 60 seg |
| `CJ_CONVERTIR_Bronze` | 12h | 3 | 60 seg |
| `CJ_EXTRAER_Metadata_Sales` | _(default)_ | 3 | 60 seg |

---

## ▶️ Ejecución

### Carga incremental (rango de fechas específico)

```
Pipeline: pl_ingst_adw_full_incremental
Parámetros:
  fechaDesde: 2024-03-01
  fechaHasta: 2024-03-31
```

### Carga incremental (ayer → hoy, sin parámetros)

```
Pipeline: pl_ingst_adw_full_incremental
Parámetros: (dejar vacíos)
→ El pipeline usa addDays(utcnow(), -1) como inicio y utcnow() como fin
```

### Carga FULL (todas las tablas marcadas como FULL en config)

```
Pipeline: pl_ingst_adw_full_incremental
Parámetros: (dejar vacíos)
→ Las tablas con load_type = 'FULL' siempre hacen SELECT * y Overwrite en Bronze
→ Las tablas con load_type = 'INCREMENTAL' usan las fechas calculadas automáticamente
```

### Agregar una nueva tabla

```sql
-- Insertar registro en lh_config.config.table_load_config
INSERT INTO config.table_load_config VALUES (
  'Sales',                                        -- schema_name
  'SalesOrderDetail',                             -- table_name
  'INCREMENTAL',                                  -- load_type
  'SELECT * FROM Sales.SalesOrderDetail WHERE ModifiedDate >= ''@{fecha_desde}'' AND ModifiedDate <= ''@{fecha_hasta}''',
  '001'                                           -- exercise_number
)
-- No requiere modificar el pipeline.
```

---

## ✅ Validación Post-Ejecución

```sql
-- Verificar tabla Bronze con metadata
SELECT
    load_type,
    process_date,
    pl_run_id,
    COUNT(*) AS filas
FROM bronze.br_sales_salesorderdetail
GROUP BY load_type, process_date, pl_run_id
ORDER BY process_date DESC

-- Verificar Parquet en landing (via Spark)
SELECT input_file_name(), COUNT(*)
FROM lh_landing.`proyecto/fuente/sales/SalesOrderDetail/*/*.parquet`
GROUP BY 1

-- Verificar catálogo de metadata
SELECT * FROM lh_landing.`proyecto/metadata/sales_tables/20240320/sales_tables_20240320.csv`
```

---

## 🔗 Recursos Relacionados

| Recurso | Descripción |
|---------|-------------|
| `lh_config.config.table_load_config` | Tabla de configuración principal |
| `lh_landing` | Almacén de archivos Parquet y CSV intermedios |
| `lh_bronze` | Destino final — tablas Delta con metadata |
| [`pl_ingst_adw_full_incremental.json`](./pl_ingst_adw_full_incremental.json) | JSON ARM del pipeline |

---

*Microsoft Fabric — Data Engineering — Euler, Diego — Febrero 2026*
