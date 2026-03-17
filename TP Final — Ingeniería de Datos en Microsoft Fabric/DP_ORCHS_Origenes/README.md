# 🎛️ DP_ORCHS_Origenes

> **Pipeline orquestador maestro** del sistema de datos end-to-end sobre Microsoft Fabric.  
> Coordina la ejecución secuencial de los dos pipelines de ingesta Silver y el pipeline Gold,  
> garantizando que el modelo analítico **nunca se construya sobre datos incompletos**.

---

## 🗺️ Diagrama del Pipeline

```
                 ┌──────────────────────────────────────┐
                 │          DP_ORCHS_Origenes            │
                 │   Trigger: TR_Diario_0200_ARG         │
                 │   02:00 AM ARG  /  05:00 UTC          │
                 └──────────────┬───────────────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │  (dependsOn: []) │
              ▼                 ▼                  │
  ┌─────────────────────┐  ┌─────────────────────┐│
  │exec_DP_INGST_        │  │exec_DP_INGST_        ││  Ambas actividades
  │SilverADVWDB         │  │SilverSFTP            ││  sin dependencia
  │                     │  │                     ││  entre sí
  │ SQL Server ADVWDB   │  │ CSV SFTP simulado   ││
  │ → lh_silver         │  │ → lh_silver         ││
  │                     │  │                     ││
  │ timeout: 2h         │  │ timeout: 2h         ││
  │ retry: 0            │  │ retry: 0            ││
  └──────────┬──────────┘  └──────────┬──────────┘│
             │                        │
             │ Succeeded              │ Succeeded
             │                        │
             └─────────┬──────────────┘
                       │ (ambos deben completar)
                       ▼
           ┌───────────────────────┐
           │   exec_DP_ORCHS_Gold  │
           │                       │
           │  DimCalendario        │
           │  DimCliente           │
           │  DimProducto          │
           │  FactVentas           │
           │                       │
           │  timeout: 2h          │
           │  retry: 0             │
           └───────────────────────┘

  ┌───────────────────────┐     ┌───────────────────────┐
  │      fail_ADVWDB      │     │       fail_SFTP        │
  │  errorCode:           │     │  errorCode:            │
  │  ADVWDB_FAIL          │     │  SFTP_FAIL             │
  └───────────────────────┘     └───────────────────────┘
       ▲                               ▲
       │ Failed                        │ Failed
       │                               │
  exec_DP_INGST_SilverADVWDB    exec_DP_INGST_SilverSFTP
```

---

## ⚙️ Actividades

| # | Actividad | Tipo | `dependsOn` | Descripción |
|---|-----------|------|------------|-------------|
| 1 | `exec_DP_INGST_SilverADVWDB` | ExecutePipeline | `[]` — sin dependencias | Ejecuta el pipeline ADVWDB. `waitOnCompletion: true` |
| 2 | `exec_DP_INGST_SilverSFTP` | ExecutePipeline | `[]` — sin dependencias | Ejecuta el pipeline SFTP. `waitOnCompletion: true` |
| 3 | `exec_DP_ORCHS_Gold` | ExecutePipeline | `[ADVWDB Succeeded, SFTP Succeeded]` | Gold solo corre si **ambos** Silver completaron |
| 4 | `fail_ADVWDB` | Fail | `[ADVWDB Failed]` | Activa si ADVWDB falla. Código: `ADVWDB_FAIL` |
| 5 | `fail_SFTP` | Fail | `[SFTP Failed]` | Activa si SFTP falla. Código: `SFTP_FAIL` |

---

## 🔀 Grafo de Dependencias

```
                     START
                       │
           ┌───────────┴───────────┐
           │ dependsOn: []         │ dependsOn: []
           ▼                       ▼
  exec_DP_INGST_SilverADVWDB   exec_DP_INGST_SilverSFTP
           │                       │
     ┌─────┴──┐               ┌────┴──────┐
   Succ.   Failed           Succ.      Failed
     │       │               │            │
     │    fail_ADVWDB         │         fail_SFTP
     │    (ADVWDB_FAIL)       │         (SFTP_FAIL)
     │                        │
     └──────────┬─────────────┘
                │ ambos Succeeded
                ▼
        exec_DP_ORCHS_Gold
                │
              Succ.
                │
              END ✅
```

> **Garantía arquitectural:** `exec_DP_ORCHS_Gold` tiene `dependsOn` con condición `Succeeded`
> sobre **ambos** pipelines de ingesta. Si cualquiera de los dos falla, Gold nunca ejecuta
> y las actividades Fail correspondientes exponen el error con código descriptivo en el Monitor.

---

## 📌 Pipelines Referenciados

| Actividad | Pipeline | ID |
|-----------|----------|----|
| `exec_DP_INGST_SilverADVWDB` | `DP_INGST_SilverADVWDB` | `8ea63884-e41b-4f37-8266-b9513bdfcfd2` |
| `exec_DP_INGST_SilverSFTP` | `DP_INGST_SilverSFTP` | `6c64e546-8773-49d5-b91e-f217c511fddc` |
| `exec_DP_ORCHS_Gold` | `DP_ORCHS_Gold` | `e47ef6e7-5733-4d16-86fa-892bb81a34da` |

---

## 🔁 Trigger Automático

| Parámetro | Valor |
|-----------|-------|
| **Nombre** | `TR_Diario_0200_ARG` |
| **Hora de ejecución** | 05:00 UTC = **02:00 AM Argentina** |
| **Frecuencia** | Diaria |
| **Pipeline activado** | `DP_ORCHS_Origenes` |

```
Zona horaria Argentina (ART = UTC-3)
  05:00 UTC  →  02:00 AM ART
```

> El horario de las 02:00 AM garantiza que los archivos CSV del día anterior ya fueron
> depositados en `lh_origin_get_talent` y que la actividad de usuarios sobre SQL Server
> AdventureWorks está en su mínimo.

---

## 🏗️ Comportamiento en el Entorno Trial

### Diseño arquitectural vs ejecución real

El JSON define `exec_DP_INGST_SilverADVWDB` y `exec_DP_INGST_SilverSFTP` con `dependsOn: []`,
lo que **permite** ejecución en paralelo. Sin embargo, el entorno Trial ejecuta ambas
actividades de forma **prácticamente secuencial** por limitación de capacidad Spark.

```
Arquitectura diseñada (lógica):          Comportamiento observado (Trial):

ADVWDB ──Succ.──┐                        ADVWDB ──6m 15s──▶ SFTP ──14m 27s──▶ Gold
                ├──▶ Gold                   (secuencial por capacidad limitada)
SFTP   ──Succ.──┘
```

**Esta es una decisión correcta y funcional.** El motor de Fabric respeta el grafo de
dependencias y ejecuta ambos en cuanto tiene capacidad disponible. El resultado final
es idéntico al diseño paralelo: Gold solo ejecuta cuando ambos completaron con éxito.

---

## 🛡️ Políticas de Ejecución

| Actividad | Timeout | Retry | Retry Delay |
|-----------|---------|-------|-------------|
| `exec_DP_INGST_SilverADVWDB` | **2 horas** | 0 | 30 seg |
| `exec_DP_INGST_SilverSFTP` | **2 horas** | 0 | 30 seg |
| `exec_DP_ORCHS_Gold` | **2 horas** | 0 | 30 seg |

> **¿Por qué timeout de 2 horas sin retry?** En el entorno Trial, el retry de un
> pipeline completo podría generar duplicados en Silver si el pipeline hijo ya escribió
> datos parciales antes de fallar. Es más seguro revisar el error y re-ejecutar
> manualmente con contexto completo.

---

## 🛡️ Manejo de Errores

| Escenario | Actividad que dispara | Código de error | Comportamiento |
|-----------|----------------------|----------------|----------------|
| `DP_INGST_SilverADVWDB` falla | `fail_ADVWDB` | `ADVWDB_FAIL` | Gold no ejecuta. Error visible en Monitor. |
| `DP_INGST_SilverSFTP` falla | `fail_SFTP` | `SFTP_FAIL` | Gold no ejecuta. Error visible en Monitor. |
| `DP_ORCHS_Gold` falla | _(no hay Fail para Gold en este pipeline)_ | — | El pipeline termina en estado Failed. |
| Ambos Silver exitosos | `exec_DP_ORCHS_Gold` | — | Gold ejecuta normalmente. |

### Mensajes de error configurados

```
fail_ADVWDB:
  message:   "Error en DP_ORCHS_Origenes — falló exec_DP_INGST_SilverADVWDB. Gold no ejecutado."
  errorCode: "ADVWDB_FAIL"

fail_SFTP:
  message:   "Error en DP_ORCHS_Origenes — falló exec_DP_INGST_SilverSFTP. Gold no ejecutado."
  errorCode: "SFTP_FAIL"
```

---

## 🔄 Flujo de Datos de Extremo a Extremo

```
                    SQL Server              lh_origin_get_talent
                 AdventureWorks             (CSVs SFTP simulado)
                        │                          │
                        ▼                          ▼
             ┌──────────────────┐      ┌──────────────────────┐
             │ DP_INGST_Silver  │      │ DP_INGST_Silver      │
             │ ADVWDB           │      │ SFTP                 │
             │                  │      │                      │
             │ SalesOrderDetail │      │ SalesOrderHeader     │
             │ (MERGE inc.)     │      │ (append inc.)        │
             │ Product          │      │ Person               │
             │ (total)          │      │ (total)              │
             └────────┬─────────┘      └──────────┬───────────┘
                      │                            │
                      └─────────┬──────────────────┘
                                │  ambos Succeeded
                                ▼
                     ┌──────────────────┐
                     │  DP_ORCHS_Gold   │
                     │                  │
                     │  d_calendario 🗓️ │  ← lh_silver + APIs externas
                     │  d_cliente    👤 │  ← lh_silver.SFTP.Person
                     │  d_producto   📦 │  ← lh_silver.ADVWDB.Product
                     │  f_ventas     📊 │  ← SOD × SOH (JOIN)
                     └────────┬─────────┘
                              │
                              ▼
                    lh_gold.ventas.*
               (listo para BI / Power BI)
```

---

## 📊 Run Verificado en Producción

**Fecha:** 10/03/2026 — 00:26 AM (Argentina)

| Actividad | Inicio | Duración | Estado |
|-----------|--------|---------|--------|
| `exec_DP_INGST_SilverADVWDB` | 00:26:19 | **6m 15s** | ✅ Succeeded |
| `exec_DP_INGST_SilverSFTP` | 00:32:35 | **14m 27s** | ✅ Succeeded |
| `exec_DP_ORCHS_Gold` | 00:47:02 | **14m 1s** | ✅ Succeeded |

```
00:26 ──────────────────────────────────────────────────── 01:01
  │
  ├── ADVWDB [■■■■■■] 6m 15s → termina 00:32:34
  │
  └──────────── SFTP  [■■■■■■■■■■■■■■] 14m 27s → termina 00:47:02
                │
                └──────────────────── Gold [■■■■■■■■■■■■■] 14m 1s → termina 01:01:03
                                                                             │
                                              Total end-to-end: 34m 44s ────┘
```

**Resultado:** 3/3 actividades Succeeded — 4 tablas Gold con datos ✅

---

## ▶️ Ejecución Manual

### Ejecutar el pipeline completo

```
Workspace gt-deuler-dev
→ DP_ORCHS_Origenes
→ Run Now
→ Esperar ~35 minutos
→ Verificar Monitor: 3/3 Succeeded
```

### Ejecutar solo un origen (para reproceso)

```bash
# Solo ADVWDB (sin pasar por el orquestador)
Abrir: DP_INGST_SilverADVWDB → Run Now → ~10 min

# Solo SFTP (sin pasar por el orquestador)
Abrir: DP_INGST_SilverSFTP → Run Now → ~15 min

# Solo Gold (cuando Silver ya está actualizado)
Abrir: DP_ORCHS_Gold → Run Now → ~20 min
```

### Configurar el trigger automático

```
DP_ORCHS_Origenes → Add trigger → New/Edit
  Name:       TR_Diario_0200_ARG
  Type:       Schedule
  Recurrence: Every 1 Day
  Start:      05:00:00 UTC  (= 02:00 AM Argentina)
→ Save → toggle: Started
```

---

## ✅ Validación Post-Ejecución

```sql
-- Verificar Silver actualizado
SELECT COUNT(*), MAX(ModifiedDate)
FROM lh_silver.ADVWDB.SalesOrderDetail      -- incremental MERGE

SELECT COUNT(*)
FROM lh_silver.ADVWDB.Product               -- total overwrite

SELECT COUNT(*), MAX(fecha_archivo)
FROM lh_silver.SFTP.SalesOrderHeader        -- incremental append

SELECT COUNT(*)
FROM lh_silver.SFTP.Person                  -- total overwrite

-- Verificar Gold construido
SELECT COUNT(*) FROM lh_gold.ventas.d_calendario   -- 2000+ filas
SELECT COUNT(*) FROM lh_gold.ventas.d_cliente      -- ~1000 filas
SELECT COUNT(*) FROM lh_gold.ventas.d_producto     -- ~500 filas
SELECT COUNT(*) FROM lh_gold.ventas.f_ventas       -- ~1000 filas
```

---

## 🔗 Recursos Relacionados

| Recurso | Descripción |
|---------|-------------|
| [`DP_INGST_SilverADVWDB`](../DP_INGST_SilverADVWDB/) | Pipeline de ingesta AdventureWorks → Silver |
| [`DP_INGST_SilverSFTP`](../DP_INGST_SilverSFTP/) | Pipeline de ingesta CSV SFTP → Silver |
| [`DP_ORCHS_Gold`](../DP_ORCHS_Gold/) | Pipeline orquestador del modelo Gold |
| [`DP_ORCHS_Origenes.json`](./DP_ORCHS_Origenes.json) | JSON del pipeline |

---

*TP Final — Ingeniería de Datos en Microsoft Fabric — Euler, Diego — Marzo 2026*
