<h1 align="center">
<span style="color:#FFD700;">π</span> Pi Data Strategy & Consulting
</h1>


# 🏭 TP Final — Ingeniería de Datos en Microsoft Fabric

> **Autor:** Euler, Diego  
> **Curso:** Ingeniería de Datos en Azure / Microsoft Fabric  


---

## 📋 Descripción General

Pipeline de datos **end-to-end** sobre **Microsoft Fabric** que implementa una arquitectura Medallion completa, integrando múltiples fuentes de datos, aplicando buenas prácticas de ingeniería de datos y documentando cada decisión de diseño.

El sistema ingesta datos desde **dos fuentes heterogéneas** (SQL Server AdventureWorks y archivos CSV vía SFTP simulado), los transforma a través de tres capas de calidad progresiva y los consolida en un **modelo dimensional Gold** listo para consumo por herramientas de Business Intelligence.

---

## 🏗️ Arquitectura — Medallion

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Microsoft Fabric                                │
│                                                                         │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│  │ LANDING  │───▶│  BRONZE  │───▶│  SILVER  │───▶│   GOLD   │          │
│  │  (raw)   │    │ (string) │    │  (typed) │    │ (modelo) │          │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘          │
│       ▲                                               ▲                 │
│       │                                               │                 │
│  ┌────┴──────┐                               ┌────────┴──────┐         │
│  │CSV (SFTP) │                               │   API Feriados│         │
│  │SQL Server │                               │   API Clima   │         │
│  └───────────┘                               └───────────────┘         │
│                           ┌──────────┐                                  │
│                           │  CONFIG  │◀── parametrización global        │
│                           └──────────┘                                  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 🗄️ Lakehouses

| Lakehouse | Función |
|-----------|---------|
| `lh_landing` | Zona de aterrizaje. CSVs de SFTP y staging SQL Server. |
| `lh_bronze` | Datos normalizados a string. Formato Delta Lake. |
| `lh_silver` | Datos tipados, limpios y deduplicados. Carga incremental con watermark. |
| `lh_gold` | Modelo dimensional analítico. Listo para BI. |
| `lh_config` | Tablas de configuración. Controla el comportamiento de todos los notebooks. |

---

## 🔌 Fuentes de Datos

### 🗃️ AdventureWorks (SQL Server)

| Tabla | Estrategia | Descripción |
|-------|-----------|-------------|
| `Sales.SalesOrderDetail` | Incremental + UPSERT | Detalles de líneas de venta. MERGE por SalesOrderDetailID. |
| `Production.Product` | Carga Total | Catálogo de productos. Overwrite completo en cada ejecución. |

### 📁 SFTP Simulado (archivos CSV)

| Tabla | Estrategia | Descripción |
|-------|-----------|-------------|
| `Sales.SalesOrderHeader` | Incremental (sin upsert) | Encabezados de órdenes. Watermark por fecha del archivo. |
| `Person.Person` | Carga Total | Datos de personas/clientes. |

> **Convención de archivos:** `NombreTabla_AAAAMMDD.csv`  
> Ejemplo: `SalesOrderHeader_20240315.csv`

### 🌐 APIs Externas

| API | Uso | Obligatoriedad |
|-----|-----|---------------|
| [ArgentinaDatos — Feriados](https://argentinadatos.com/api/v1/feriados/) | Enriquece `d_calendario` con feriados nacionales | ✅ Obligatorio |
| [Open-Meteo](https://api.open-meteo.com) | Temperatura histórica Buenos Aires en `d_calendario` | 🌟 Bonus |

---

## ⚙️ Pipelines

```
DP_ORCHS_Origenes (Orquestador Maestro — trigger 02:00 AM ARG)
│
├── exec_DP_INGST_SilverADVWDB
│   └── Lookup → Filter → ForEach
│       └── IfCondition
│           ├── [SalesOrderDetail] Copy SQL → NB_TRNSF_SilverADVWDB_SalesOrderDetail
│           └── [Product]          Copy SQL → NB_TRNSF_SilverADVWDB_Product
│
├── exec_DP_INGST_SilverSFTP
│   └── Lookup → Filter → ForEach
│       └── Copy CSV → Wait(90s) → NB_Bronze → Wait(180s) → NB_Silver
│           ├── SalesOrderHeader
│           └── Person
│
└── exec_DP_ORCHS_Gold
    └── DimCalendario → Wait → DimCliente → Wait → DimProducto → Wait → FactVentas
```

| Pipeline | Descripción | Duración típica |
|----------|-------------|----------------|
| `DP_ORCHS_Origenes` | Orquestador maestro con trigger diario | ~35 min |
| `DP_INGST_SilverADVWDB` | Ingesta AdventureWorks → Silver | ~6–7 min |
| `DP_INGST_SilverSFTP` | Ingesta CSV (SFTP) → Silver | ~14–15 min |
| `DP_ORCHS_Gold` | Construcción del modelo dimensional | ~14 min |

---

## 📓 Notebooks

### 🔧 Configuración (ejecución manual única)

| Notebook | Función |
|----------|---------|
| `NB_CNFGS_SetupSchemas` | Crea schemas en los 4 lakehouses |
| `NB_CNFGS_SetupLhConfig` | Carga tablas de configuración en `lh_config` |

### 🥉 Bronze — SFTP

| Notebook | Fuente | Estrategia |
|----------|--------|-----------|
| `NB_INGST_BronzeSFTP_Person` | CSV Person | Total — todo string |
| `NB_INGST_BronzeSFTP_SalesOrderHeader` | CSV SalesOrderHeader | Incremental por `fecha_archivo` |

### 🥈 Silver — SFTP

| Notebook | Fuente | Estrategia |
|----------|--------|-----------|
| `NB_TRNSF_SilverSFTP_Person` | Bronze Person | Total — tipado y limpieza |
| `NB_TRNSF_SilverSFTP_SalesOrderHeader` | Bronze SalesOrderHeader | Incremental con watermark |

### 🥈 Silver — AdventureWorks

| Notebook | Fuente | Estrategia |
|----------|--------|-----------|
| `NB_TRNSF_SilverADVWDB_Product` | Staging lh_silver | Total — overwrite |
| `NB_TRNSF_SilverADVWDB_SalesOrderDetail` | Staging lh_silver | **MERGE** Delta Lake |

### 🥇 Gold

| Notebook | Tabla destino | Descripción |
|----------|--------------|-------------|
| `NB_TRNSF_GoldVentas_DimCalendario` | `d_calendario` | Fechas + API Feriados AR + API Clima |
| `NB_TRNSF_GoldVentas_DimCliente` | `d_cliente` | Personas/clientes desde Silver Person |
| `NB_TRNSF_GoldVentas_DimProducto` | `d_producto` | Productos desde Silver Product |
| `NB_TRNSF_GoldVentas_FactVentas` | `f_ventas` | JOIN SOD × SOH + analítica de negocio |

---

## 🌟 Modelo de Datos Gold — Esquema en Estrella

```
                        ┌─────────────────┐
                        │  d_calendario   │
                        │─────────────────│
                        │ Fecha (PK)      │
                        │ CodTiempo       │
                        │ Mes_Year        │
                        │ NomDia / NomMes │
                        │ esFeriado 🇦🇷  │
                        │ tempMedia 🌡️   │
                        └────────┬────────┘
                                 │
┌─────────────────┐    ┌────────┴──────────┐    ┌─────────────────┐
│   d_cliente     │    │     f_ventas      │    │   d_producto    │
│─────────────────│    │───────────────────│    │─────────────────│
│ id_cliente (PK) │◀───│ id_cliente (FK)   │    │ id_producto(PK) │
│ nombre_completo │    │ id_producto (FK)  │───▶│ nombre_producto │
│ tipo_persona    │    │ fecha_orden (FK)  │    │ linea_producto  │
│ acepta_email    │    │ id_orden          │    │ precio_lista    │
└─────────────────┘    │ cantidad          │    │ costo_estandar  │
                        │ precio_unitario   │    └─────────────────┘
                        │ descuento         │
                        │ total_linea       │
                        │ total_orden       │
                        │ es_online         │
                        └───────────────────┘
```

### 📊 Consultas Analíticas Implementadas

- **Top 10 clientes** por ventas semana actual vs. semana anterior
- **Ventas por producto y mes** con JOIN a dimensiones calendario y producto

---

## 🔄 Estrategias de Carga

| Tabla | Capa | Estrategia | Mecanismo |
|-------|------|-----------|-----------|
| `Person` | Bronze→Silver | Total | Overwrite completo |
| `SalesOrderHeader` | Bronze→Silver | Incremental | Watermark por `fecha_archivo` |
| `Product` | Silver | Total | DROP TABLE + saveAsTable |
| `SalesOrderDetail` | Silver | Incremental | **MERGE** Delta Lake sobre PK |
| Tablas Gold | Gold | Total | DROP TABLE + saveAsTable |

---

## 🔁 Lógica de Reprocesamiento ⭐ Bonus

Los notebooks `NB_TRNSF_SilverSFTP_SalesOrderHeader` y `NB_TRNSF_SilverADVWDB_SalesOrderDetail` soportan reprocesamiento por rango de fechas:

```python
# Parámetros de entrada al notebook
fecha_inicio_reproceso = "2024-01-01"
fecha_fin_reproceso    = "2024-01-31"
```

El notebook:
1. 🗑️ Elimina registros Silver del rango indicado
2. 🔄 Resetea el watermark al inicio del rango
3. ✅ Reprocesa desde Bronze sin duplicados

---

## ⚙️ Configuración — `lh_config`

Toda la parametrización del sistema vive en `lh_config`. **Cero hardcodeos** en notebooks ni pipelines.

### `origin_bronze_silver`

| Campo | Descripción |
|-------|-------------|
| `nombre_tabla` | Nombre de la tabla a procesar |
| `origen` | `SFTP` o `ADVWDB` |
| `pks` | Primary keys separadas por coma |
| `tipo_carga` | `total` o `incremental` |
| `parametros_incrementales` | Campo de fecha para watermark |
| `ultimo_valor_incremental` | Último valor procesado (se actualiza automáticamente) |
| `activo` | `1` = activo, `0` = deshabilitado |

### `origin_gold`

| Campo | Descripción |
|-------|-------------|
| `nombre_notebook` | Nombre del notebook Gold |
| `modelo` | Schema destino (ej: `ventas`) |
| `nombre_tabla` | Tabla Gold destino (ej: `d_cliente`) |
| `activo` | `1` = activo |

---

## 🚀 Ejecución

### Primer Ejecución

```bash
# 1. Crear schemas
Ejecutar: NB_CNFGS_SetupSchemas

# 2. Cargar configuración
Ejecutar: NB_CNFGS_SetupLhConfig

# 3. Pipeline Silver ADVWDB
Run: DP_INGST_SilverADVWDB   → esperar ~7 min → verificar 9/9 Succeeded

# 4. Pipeline Silver SFTP
Run: DP_INGST_SilverSFTP     → esperar ~15 min → verificar 13/13 Succeeded

# 5. Pipeline Gold
Run: DP_ORCHS_Gold           → esperar ~15 min → verificar 10/10 Succeeded
```

### Ejecución Diaria Automática

```
Trigger: TR_Diario_0200_ARG
Hora:    05:00 UTC = 02:00 AM Argentina
Pipeline: DP_ORCHS_Origenes
Duración: ~35 minutos end-to-end
```

### Validación Post-Ejecución

```sql
-- Silver
SELECT COUNT(*) FROM lh_silver.ADVWDB.Product           -- ~500 filas
SELECT COUNT(*) FROM lh_silver.ADVWDB.SalesOrderDetail  -- ~1000 filas
SELECT COUNT(*) FROM lh_silver.SFTP.SalesOrderHeader
SELECT COUNT(*) FROM lh_silver.SFTP.Person              -- ~1000 filas

-- Gold
SELECT COUNT(*) FROM lh_gold.ventas.d_calendario        -- 2000+ filas
SELECT COUNT(*) FROM lh_gold.ventas.d_cliente           -- ~1000 filas
SELECT COUNT(*) FROM lh_gold.ventas.d_producto          -- ~500 filas
SELECT COUNT(*) FROM lh_gold.ventas.f_ventas            -- ~1000 filas
```

---

## ✅ Checklist del TP

- [x] Definir 2 tablas ADVW (SalesOrderDetail + Product)
- [x] Definir tablas Gold (d_calendario, d_cliente, d_producto, f_ventas)
- [x] Crear archivos CSV "sucios" con formato `NombreTabla_AAAAMMDD.csv`
- [x] Pipeline Landing → Bronze (todo string → Delta)
- [x] Pipeline Bronze → Silver (limpieza y tipado)
- [x] Pipeline Silver → Gold (modelado dimensional)
- [x] Integrar API de Feriados Argentina en `d_calendario`
- [x] Implementar carga incremental con watermark
- [x] Implementar parametrización desde tabla de config
- [x] ADVWDB con TOP 1000 en desarrollo
- [x] Validar funcionamiento end-to-end ✅ `35/35 actividades Succeeded`
- [x] Documento técnico v4
- [x] 🌟 **BONUS** — API de Clima Open-Meteo en `d_calendario`
- [x] 🌟 **BONUS** — Lógica de reprocesamiento por rango de fechas

---

## 🛠️ Stack Tecnológico

| Tecnología | Uso |
|-----------|-----|
| **Microsoft Fabric** | Plataforma principal (Lakehouse, Data Factory, Spark) |
| **Apache Spark / PySpark** | Motor de procesamiento distribuido |
| **Delta Lake** | Formato de almacenamiento (ACID, MERGE, schema evolution) |
| **Microsoft Fabric Data Factory** | Orquestación de pipelines |
| **SQL Server AdventureWorks2019** | Fuente de datos relacional |
| **Python 3** | Notebooks PySpark |
| **API ArgentinaDatos** | Feriados nacionales Argentina |
| **API Open-Meteo** | Datos históricos de temperatura |

---

## 📁 Estructura de IDs (Workspace gt-deuler-dev)

```
Workspace ID : 9a547f55-a8f7-4c89-b935-fb3da8226ed0
├── lh_landing  : f27e2853-1dca-44d9-94f8-08477a5836ff
├── lh_bronze   : e9dcb661-05e8-46bd-be1f-a9cb5ee27446
├── lh_silver   : 4d897e0a-b68c-4840-9910-752862308e5b
├── lh_gold     : 902f27e9-dd04-40cf-831a-2610a16fb9a7
└── lh_config   : bcc2be74-375f-42f1-b1c1-b60b5eef55ef

Conexión AdventureWorks : cafffe28-7d56-4ca4-ac0e-3ac73f036368
```

---

## 📊 Resultados Verificados en Producción

| Pipeline | Run | Actividades | Duración | Estado |
|----------|-----|------------|---------|--------|
| `DP_ORCHS_Origenes` | 10/03/2026 | 3/3 | 34m 44s | ✅ Succeeded |
| `DP_INGST_SilverADVWDB` | 09/03/2026 | 9/9 | 10m 29s | ✅ Succeeded |
| `DP_INGST_SilverSFTP` | 09/03/2026 | 13/13 | 14m 30s | ✅ Succeeded |
| `DP_ORCHS_Gold` | 09/03/2026 | 10/10 | 20m 46s | ✅ Succeeded |

### Tablas Gold con datos verificados

| Tabla | Filas | Estado |
|-------|-------|--------|
| `lh_gold.ventas.d_calendario` | 2000+ | ✅ |
| `lh_gold.ventas.d_cliente` | 1000 | ✅ |
| `lh_gold.ventas.d_producto` | 1000 | ✅ |
| `lh_gold.ventas.f_ventas` | 1000 | ✅ |

---

## 📖 Decisiones Técnicas Clave

### `notebookutils.lakehouse.get()` vs `runtime.context`

```python
# ❌ Falla cuando el notebook es invocado por un pipeline (sesión Livy)
_ctx  = notebookutils.runtime.context    # → None en pipeline
WS_ID = _ctx['workspaceId']

# ✅ Funciona tanto en ejecución manual como invocado por pipeline
_lh   = notebookutils.lakehouse.get('lh_silver')
WS_ID = _lh.workspaceId
ART   = _lh.id
```

### Catálogo 3-part vs paths ABFS

```python
# ✅ Para la mayoría de operaciones — más simple y legible
spark.sql("SELECT * FROM lh_silver.ADVWDB.Product")
df.write.saveAsTable("lh_gold.ventas.d_producto")

# ✅ Requerido para DeltaTable.forPath() y operaciones MERGE
abfs = f"abfss://{WS_ID}@onelake.dfs.fabric.microsoft.com/{ART_SILVER}/Tables/ADVWDB/SalesOrderDetail"
DeltaTable.forPath(spark, abfs)
```

### Actividades Wait en entorno Trial

```
Wait1 (90s)  → entre Copy CSV y NB_Bronze  
Wait2 (180s) → entre NB_Bronze y NB_Silver
```
> El entorno Trial de Fabric tiene capacidad limitada de sesiones Spark concurrentes. Sin los waits, los notebooks consecutivos compiten por la misma sesión Livy y fallan con error 430.


---

*Trabajo Práctico Final — Ingeniería de Datos en Microsoft Fabric — Marzo 2026*
