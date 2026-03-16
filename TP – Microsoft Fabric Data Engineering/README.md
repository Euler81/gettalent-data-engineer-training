<h1 align="center">
<span style="color:#FFD700;">π</span> Data Strategy & Consulting
</h1>

<h1 align="center">
<span style="color:#FFD700;">π</span> Data Strategy & Consulting
</h1>

================================================================================
                    TP4 - PIPELINE DE INGESTA ADVENTUREWORKS
                    Microsoft Fabric - Arquitectura Medallón
================================================================================

📋 TABLA DE CONTENIDOS
================================================================================
1. Introducción
2. Objetivo del Proyecto
3. Tecnologías Utilizadas
4. Requisitos Previos
5. Arquitectura de la Solución
6. Alcance del Proyecto
7. Instrucciones de Implementación
8. Estructura de Datos
9. Parámetros del Pipeline
10. Tablas Procesadas
11. Características Técnicas
12. Troubleshooting
13. Fecha de Entrega

================================================================================
1. INTRODUCCIÓN
================================================================================

El presente documento describe la implementación completa de la solución de 
ingesta de datos desarrollada en el Trabajo Práctico 4 (TP4), utilizando 
Microsoft Fabric y siguiendo la arquitectura Medallón (Landing → Bronze).

La solución permite ingerir 22 tablas de la base de datos AdventureWorks2019 
mediante un pipeline dinámico y parametrizado que soporta cargas FULL e 
INCREMENTALES.

================================================================================
2. OBJETIVO DEL PROYECTO
================================================================================

✅ Extraer 22 tablas de AdventureWorks2019 mediante Data Gateway
✅ Implementar un pipeline dinámico y parametrizado
✅ Soportar cargas FULL e INCREMENTALES
✅ Almacenar datos crudos en Landing (formato Parquet con compresión Snappy)
✅ Transformar y enriquecer datos en Bronze (formato Delta con metadatos)
✅ Externalizar configuración mediante lakehouse de configuración
✅ Usar un solo pipeline para todas las tablas
✅ Implementar lakehouse de configuración y tablas de parametría

================================================================================
3. TECNOLOGÍAS UTILIZADAS
================================================================================

┌─────────────────────┬────────────────────────────────────────────────────────┐
│ Tecnología          │ Propósito                                              │
├─────────────────────┼────────────────────────────────────────────────────────┤
│ Microsoft Fabric    │ Plataforma de análisis de datos unificada              │
│ Data Factory        │ Orquestación de pipelines ETL                          │
│ Data Gateway        │ Conexión segura on-premises a cloud                    │
│ Lakehouse           │ Almacenamiento en capas (Landing/Bronze)               │
│ Parquet             │ Formato de almacenamiento en Landing (Snappy)          │
│ Delta Lake          │ Formato de tablas en Bronze con ACID transactions      │
│ SQL Server          │ Base de datos origen (AdventureWorks2019)              │
└─────────────────────┴────────────────────────────────────────────────────────┘

================================================================================
4. REQUISITOS PREVIOS
================================================================================

4.1 ACCESOS Y CONFIGURACIÓN INICIAL
───────────────────────────────────
✅ Acceder al workspace asignado en Fabric
   🔗 https://app.fabric.microsoft.com/
   📧 Usuario y contraseña enviados por correo
   🔐 Cambiar contraseña y configurar MS Authenticator

✅ Asociar área de trabajo a repositorio Git personal
   (Puede realizarse al finalizar el proyecto)

✅ Tener instalada la base de datos AdventureWorks2019

4.2 PREPARACIÓN DE LA BASE DE DATOS
───────────────────────────────────
Modificar fechas en tablas transaccionales para tener años 2024 y 2026:

UPDATE Sales.SalesOrderHeader 
SET OrderDate = DATEADD(YEAR, 4, OrderDate) 
WHERE YEAR(OrderDate) = 2020;

UPDATE Sales.SalesOrderDetail 
SET ModifiedDate = DATEADD(YEAR, 4, ModifiedDate) 
WHERE YEAR(ModifiedDate) = 2020;

================================================================================
5. ARQUITECTURA DE LA SOLUCIÓN
================================================================================

5.1 DIAGRAMA DE ARQUITECTURA
────────────────────────────

 ────────────────────────────────────────────────────────────────────────────
│                    MICROSOFT FABRIC (CLOUD)                                │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │          PIPELINE: pl_ingst_adw_full_incremental                    │   │
│  │                                                                     │   │
│  │  ┌──────────┐    ┌─────────────┐    ┌──────────────────┐            │   │
│  │  │  Lookup  │───▶│   ForEach   │───▶│   Copy Metadata  │           │   │
│  │  │ Get Tables│   │(Batch=5)    │    │   Sales          │            │   │
│  │  └──────────┘    └──────┬──────┘    └──────────────────┘            │   │
│  │                         │                                           │   │
│  │              ┌──────────┴──────────┐                                │   │
│  │              ▼                     ▼                                │   │
│  │       ┌──────────┐          ┌──────────┐                            │   │
│  │       │   Copy   │─────────▶│   Copy   │                            │  │
│  │       │  INGEST  │          │  BRONZE  │                            │   │
│  │       │ Dynamic  │          │ CONVERT  │                            │   │
│  │       └──────────┘          └──────────┘                            │   │
│  │        (Landing)              (Bronze)                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                      │
│  │ lh_config    │  │ lh_landing   │  │  lh_bronze   │                      │
│  │ (Config)     │  │   (Raw)      │  │ (Processed)  │                      │
│  └──────────────┘  └──────────────┘  └──────────────┘                      │
 ────────────────────────────────────────────────────────────────────────────
                                   ▲
                                   │
                            ┌──────┴──────┐
                            │Data Gateway │
                            └──────┬──────┘
                                   │
                    ┌──────────────────────────────┐
                    │   ON-PREMISES                │
                    │  AdventureWorks2019          │
                    │  (SQL Server)                │
                    └──────────────────────────────┘

5.2 COMPONENTES PRINCIPALES
───────────────────────────

┌─────────────────────┬────────────────────────────────────────────────────────┐
│ Componente          │ Descripción                                            │
├─────────────────────┼────────────────────────────────────────────────────────┤
│ Pipeline Principal  │ pl_ingst_adw_full_incremental                          │
│                     │ Orquesta la ingesta de 22 tablas con ejecución         │
│                     │ paralela (batch size = 5)                              │
├─────────────────────┼────────────────────────────────────────────────────────┤
│ Lakehouse Config    │ lh_config                                              │
│                     │ Almacena config.table_load_config con metadatos        │
│                     │ (schema, table, load_type, folder_date_pattern,        │
│                     │ is_active, exercise_number)                            │
├─────────────────────┼────────────────────────────────────────────────────────┤
│ Data Gateway        │ Conexión segura entre Fabric (cloud) y                 │
│                     │ AdventureWorks2019 (on-premises)                       │
├─────────────────────┼────────────────────────────────────────────────────────┤
│ Lakehouse Landing   │ lh_landing                                             │
│                     │ Datos crudos en Parquet con compresión Snappy          │
│                     │ Path: proyecto/fuente/<esq>/<tab>/AAAAMMDD/            │
│                     │       <tab>_AAAAMMDD.parquet                           │
├─────────────────────┼────────────────────────────────────────────────────────┤
│ Lakehouse Bronze    │ lh_bronze                                              │
│                     │ Tablas Delta con nomenclatura br_<esq>_<tab>           │
│                     │ 5 columnas de metadatos adicionales                    │
└─────────────────────┴────────────────────────────────────────────────────────┘

================================================================================
6. ALCANCE DEL PROYECTO
================================================================================

6.1 TABLAS POR ESQUEMA
──────────────────────

ESQUEMA PRODUCTION (3 tablas - Carga FULL)
──────────────────────────────────────────
✅ Product
✅ ProductSubcategory
✅ ProductCategory

ESQUEMA PERSON (1 tabla - Carga FULL)
─────────────────────────────────────
✅ ContactType

ESQUEMA SALES (18 tablas)
─────────────────────────

🔄 Carga INCREMENTAL (2 tablas):
   • SalesOrderHeader  - Filtrado por OrderDate
   • SalesOrderDetail  - Filtrado por ModifiedDate

✅ Carga FULL (16 tablas):
   • Customer
   • SalesPerson
   • SalesTerritory
   • ShoppingCartItem
   • SpecialOffer
   • SpecialOfferProduct
   • Store
   • CountryRegionCurrency
   • CreditCard
   • Currency
   • CurrencyRate
   • PersonCreditCard
   • SalesOrderHeaderSalesReason
   • SalesPersonQuotaHistory
   • SalesReason
   • SalesTaxRate

6.2 METADATOS DEL SISTEMA
─────────────────────────
📋 sys.tables + sys.schemas (esquema Sales)
   Query: SELECT s.name, t.name FROM sys.tables t 
          INNER JOIN sys.schemas s ON t.schema_id = s.schema_id 
          WHERE s.name = 'Sales'

================================================================================
7. INSTRUCCIONES DE IMPLEMENTACIÓN
================================================================================

PASO 1: CREAR LAKEHOUSES
────────────────────────
CREATE LAKEHOUSE lh_config;
CREATE LAKEHOUSE lh_landing;
CREATE LAKEHOUSE lh_bronze;

-- Crear tabla de configuración en lh_config
CREATE TABLE config.table_load_config (
    schema_name STRING,
    table_name STRING,
    load_type STRING,              -- 'FULL' o 'INCREMENTAL'
    folder_date_pattern STRING,    -- 'AAAAMMDD'
    source_query_template STRING,  -- Query personalizado (opcional)
    is_active INT,                 -- 1 = activo, 0 = inactivo
    exercise_number STRING         -- 'TP4'
);

PASO 2: INSTALAR DATA GATEWAY
─────────────────────────────
🔗 Descargar: https://go.microsoft.com/fwlink/?linkid=825440

1. Ejecutar instalador como administrador
2. Seleccionar modo: "Standard" (recomendado)
3. Iniciar sesión con cuenta de Azure
4. Registrar gateway en el tenant
5. Configurar cluster (nuevo o existente)
6. Verificar estado: Servicio debe estar "En ejecución"

PASO 3: CONFIGURAR CONEXIÓN EN FABRIC
─────────────────────────────────────
1. Ir a Settings → Manage connections and gateways
2. Seleccionar Data Gateway instalado
3. Crear nueva conexión:
   - Tipo: SQL Server
   - Servidor: <nombre_servidor_onprem>
   - Base de datos: AdventureWorks2019
   - Autenticación: Windows/SQL Server
   - Nombre: conn_adventureworks
4. Testear conexión ✅
5. Guardar configuración

PASO 4: POBLAR TABLA DE CONFIGURACIÓN
─────────────────────────────────────
-- Tablas Production (FULL)
INSERT INTO config.table_load_config VALUES
('Production', 'Product', 'FULL', 'AAAAMMDD', NULL, 1, 'TP4'),
('Production', 'ProductSubcategory', 'FULL', 'AAAAMMDD', NULL, 1, 'TP4'),
('Production', 'ProductCategory', 'FULL', 'AAAAMMDD', NULL, 1, 'TP4'),

-- Tabla Person (FULL)
('Person', 'ContactType', 'FULL', 'AAAAMMDD', NULL, 1, 'TP4'),

-- Tablas Sales INCREMENTALES
('Sales', 'SalesOrderHeader', 'INCREMENTAL', 'AAAAMMDD', 
 'SELECT * FROM Sales.SalesOrderHeader WHERE OrderDate 
  BETWEEN ''@{fecha_desde}'' AND ''@{fecha_hasta}''', 1, 'TP4'),
('Sales', 'SalesOrderDetail', 'INCREMENTAL', 'AAAAMMDD',
 'SELECT * FROM Sales.SalesOrderDetail WHERE ModifiedDate 
  BETWEEN ''@{fecha_desde}'' AND ''@{fecha_hasta}''', 1, 'TP4'),

-- Tablas Sales FULL (16 tablas)
('Sales', 'Customer', 'FULL', 'AAAAMMDD', NULL, 1, 'TP4'),
('Sales', 'SalesPerson', 'FULL', 'AAAAMMDD', NULL, 1, 'TP4'),
('Sales', 'SalesTerritory', 'FULL', 'AAAAMMDD', NULL, 1, 'TP4'),
('Sales', 'ShoppingCartItem', 'FULL', 'AAAAMMDD', NULL, 1, 'TP4'),
('Sales', 'SpecialOffer', 'FULL', 'AAAAMMDD', NULL, 1, 'TP4'),
('Sales', 'SpecialOfferProduct', 'FULL', 'AAAAMMDD', NULL, 1, 'TP4'),
('Sales', 'Store', 'FULL', 'AAAAMMDD', NULL, 1, 'TP4'),
('Sales', 'CountryRegionCurrency', 'FULL', 'AAAAMMDD', NULL, 1, 'TP4'),
('Sales', 'CreditCard', 'FULL', 'AAAAMMDD', NULL, 1, 'TP4'),
('Sales', 'Currency', 'FULL', 'AAAAMMDD', NULL, 1, 'TP4'),
('Sales', 'CurrencyRate', 'FULL', 'AAAAMMDD', NULL, 1, 'TP4'),
('Sales', 'PersonCreditCard', 'FULL', 'AAAAMMDD', NULL, 1, 'TP4'),
('Sales', 'SalesOrderHeaderSalesReason', 'FULL', 'AAAAMMDD', NULL, 1, 'TP4'),
('Sales', 'SalesPersonQuotaHistory', 'FULL', 'AAAAMMDD', NULL, 1, 'TP4'),
('Sales', 'SalesReason', 'FULL', 'AAAAMMDD', NULL, 1, 'TP4'),
('Sales', 'SalesTaxRate', 'FULL', 'AAAAMMDD', NULL, 1, 'TP4');

PASO 5: IMPORTAR PIPELINE
─────────────────────────
1. En Fabric Workspace: New → Data Factory
2. Import pipeline → Seleccionar pl_ingst_adw_full_incremental.json
3. Configurar linked services:
   - lh_config → apuntar a lakehouse config
   - conn_adventureworks → conexión Data Gateway
   - lh_landing → lakehouse landing
   - lh_bronze → lakehouse bronze
4. Publicar pipeline ✅

PASO 6: EJECUTAR PIPELINE
─────────────────────────

🔵 EJECUCIÓN AUTOMÁTICA (DEFAULT)
   ▶️ Click en "Run" sin parámetros
   
   Valores automáticos:
   - fechaDesde = ayer (utcnow() - 1 día)
   - fechaHasta = hoy (utcnow())
   - pl_run_id = GUID automático
   
   Ideal para:
   - Ejecuciones programadas diarias
   - Cargas incrementales automáticas

🟢 EJECUCIÓN MANUAL CON PARÁMETROS
   ▶️ Click en "Run" con parámetros personalizados
   
   Ejemplo:
   fechaDesde: "2024-01-01"
   fechaHasta: "2024-01-31"
   pl_run_id: "@pipeline().RunId"
   
   Ideal para:
   - Reprocesamiento histórico
   - Cargas retroactivas
   - Testing y desarrollo

PASO 7: VALIDAR RESULTADOS
──────────────────────────
-- Verificar en lh_landing:
📁 Files/proyecto/fuente/Production/Product/20240209/
   └── Product_20240209.parquet

-- Verificar en lh_bronze:
📁 Tables/bronze/
   ├── br_production_product (Delta)
   ├── br_sales_salesorderheader (Delta)
   └── ... (22 tablas)

-- Consultar metadatos:
SELECT COUNT(*), load_type, process_date 
FROM bronze.br_sales_salesorderheader 
GROUP BY load_type, process_date;

================================================================================
8. ESTRUCTURA DE DATOS
================================================================================

8.1 CAPA LANDING (RAW DATA)
───────────────────────────

ESTRUCTURA DE CARPETAS:
lh_landing/Files/proyecto/fuente/<esquema>/<tabla>/AAAAMMDD/<tabla>_AAAAMMDD.parquet

EJEMPLOS:
✅ FULL:
   lh_landing/Files/proyecto/fuente/Production/Product/
   20240209/Product_20240209.parquet

✅ INCREMENTAL:
   lh_landing/Files/proyecto/fuente/Sales/SalesOrderHeader/
   20240209/SalesOrderHeader_20240209.parquet

CARACTERÍSTICAS:
┌─────────────────────┬────────────────────────────────────────────────────────┐
│ Propiedad           │ Valor                                                  │
├─────────────────────┼────────────────────────────────────────────────────────┤
│ Formato             │ Parquet                                                │
│ Compresión          │ Snappy                                                 │
│ Particionamiento    │ Por fecha (AAAAMMDD)                                   │
│ Contenido           │ Datos crudos sin transformación                        │
│ Nomenclatura        │ <tabla>_AAAAMMDD.parquet                               │
└─────────────────────┴────────────────────────────────────────────────────────┘

8.2 CAPA BRONZE (PROCESSED DATA)
────────────────────────────────

NOMENCLATURA DE TABLAS:
br_<esquema>_<tabla>

EJEMPLOS:
br_production_product
br_production_productsubcategory
br_production_productcategory
br_person_contacttype
br_sales_salesorderheader
br_sales_salesorderdetail
... (16 tablas Sales adicionales)

ESQUEMA DE METADATOS (5 COLUMNAS ADICIONALES):
┌─────────────────────┬───────────┬────────────────────────────────────────────┐
│ Columna             │ Tipo      │ Descripción                                │
├─────────────────────┼───────────┼────────────────────────────────────────────┤
│ ingestion_date_ts   │ TIMESTAMP │ Timestamp UTC de la ingesta                │
│                     │           │ Ej: 2024-02-09 15:30:45.123                │
├─────────────────────┼───────────┼────────────────────────────────────────────┤
│ pl_run_id           │ STRING    │ ID único de ejecución del pipeline         │
│                     │           │ Ej: a834ac00-5cfb-4598-bb1d-4fb3b898fb3a   │
├─────────────────────┼───────────┼────────────────────────────────────────────┤
│ process_date        │ DATE      │ Fecha de procesamiento                     │
│                     │           │ Ej: 2024-02-09                             │
├─────────────────────┼───────────┼────────────────────────────────────────────┤
│ load_type           │ STRING    │ Tipo de carga aplicada                     │
│                     │           │ FULL / INCREMENTAL                         │
├─────────────────────┼───────────┼────────────────────────────────────────────┤
│ exercise_number     │ STRING    │ Identificador del ejercicio                │
│                     │           │ TP4                                        │
└─────────────────────┴───────────┴────────────────────────────────────────────┘

EJEMPLO DE CONSULTA:
────────────────────
SELECT 
    productid,
    name,
    productnumber,
    ingestion_date_ts,
    pl_run_id,
    load_type
FROM bronze.br_production_product
WHERE process_date = '2024-02-09'

================================================================================
9. PARÁMETROS DEL PIPELINE
================================================================================

9.1 CONFIGURACIÓN DE PARÁMETROS
───────────────────────────────
┌─────────────────────┬──────────┬─────────────────────────────────────────────┐
│ Parámetro           │ Tipo     │ Descripción                                 │
├─────────────────────┼──────────┼─────────────────────────────────────────────┤
│ fechaDesde          │ string   │ Fecha inicial para cargas incrementales     │
│                     │          │ Default: hoy - 1 día (automático)           │
│                     │          │ Formato: yyyy-MM-dd                         │
│                     │          │ Ejemplo: 2024-02-08                         │
├─────────────────────┼──────────┼─────────────────────────────────────────────┤
│ fechaHasta          │ string   │ Fecha final para cargas incrementales       │
│                     │          │ Default: hoy (automático)                   │
│                     │          │ Formato: yyyy-MM-dd                         │
│                     │          │ Ejemplo: 2024-02-09                         │
│                     │          │ Nota: Fecha NO inclusiva                    │
├─────────────────────┼──────────┼─────────────────────────────────────────────┤
│ pl_run_id           │ string   │ ID de ejecución del pipeline                │
│                     │          │ Default: @pipeline().RunId                  │
│                     │          │ Generación: Automática                      │
└─────────────────────┴──────────┴─────────────────────────────────────────────┘

9.2 LÓGICA DE APLICACIÓN
────────────────────────
SI load_type = 'FULL':
    ✅ SELECT * FROM Production.Product
    
SI load_type = 'INCREMENTAL':
    ✅ SELECT * FROM Sales.SalesOrderHeader
       WHERE OrderDate BETWEEN '2024-02-08' AND '2024-02-09'
       
       └── fechaDesde: COALESCE(@pipeline().fechaDesde, ayer)
       └── fechaHasta: COALESCE(@pipeline().fechaHasta, hoy)

================================================================================
10. TABLAS PROCESADAS
================================================================================

10.1 RESUMEN POR ESQUEMA
────────────────────────
┌──────────────┬──────────┬────────────┬───────────────────┐
│ Esquema      │ Cantidad │ Carga FULL │ Carga INCREMENTAL │
│──────────────┤──────────┤────────────┤───────────────────┤
│ Production   │ 3        │ ✅ 3       │ ❌ 0             │
│ Person       │ 1        │ ✅ 1       │ ❌ 0             │
│ Sales        │ 18       │ ✅ 16      │ ✅ 2             │
├──────────────┼──────────┼────────────┼───────────────────┤
│ TOTAL        │ 22       │ 20         │ 2                 │
└──────────────┴──────────┴────────────┴───────────────────┘

10.2 DETALLE DE TABLAS
──────────────────────

📦 PRODUCTION (3 tablas - FULL)
   ✅ Production.Product
   ✅ Production.ProductSubcategory
   ✅ Production.ProductCategory

👤 PERSON (1 tabla - FULL)
   ✅ Person.ContactType

💰 SALES (18 tablas)

   🔄 INCREMENTALES (2):
      • Sales.SalesOrderHeader
        └── WHERE OrderDate BETWEEN @fechaDesde AND @fechaHasta
        └── Modo: Append
      
      • Sales.SalesOrderDetail
        └── WHERE ModifiedDate BETWEEN @fechaDesde AND @fechaHasta
        └── Modo: Append

   ✅ FULL (16):
      • Sales.Customer
      • Sales.SalesPerson
      • Sales.SalesTerritory
      • Sales.ShoppingCartItem
      • Sales.SpecialOffer
      • Sales.SpecialOfferProduct
      • Sales.Store
      • Sales.CountryRegionCurrency
      • Sales.CreditCard
      • Sales.Currency
      • Sales.CurrencyRate
      • Sales.PersonCreditCard
      • Sales.SalesOrderHeaderSalesReason
      • Sales.SalesPersonQuotaHistory
      • Sales.SalesReason
      • Sales.SalesTaxRate

================================================================================
11. CARACTERÍSTICAS TÉCNICAS
================================================================================

11.1 DISEÑO DINÁMICO Y ESCALABLE
────────────────────────────────
✅ Pipeline único que procesa 22 tablas sin hardcoding
✅ Configuración externalizada en lh_config
✅ Activación/desactivación de tablas vía is_active
✅ Fácil adición de nuevas tablas (INSERT en config)
✅ Paralelismo configurable (batchCount = 5)

BENEFICIOS:
   🔧 Mantenimiento simplificado
   📈 Escalabilidad horizontal
   🎯 Flexibilidad operativa

11.2 GESTIÓN INTELIGENTE DE CARGAS
──────────────────────────────────

🔵 CARGAS FULL:
   • Modo: Overwrite (sobrescritura completa)
   • Aplicación: Tablas maestras y de dimensión
   • Cantidad: 20 de 22 tablas (91%)
   • Ejemplo: Production.Product, Person.ContactType

🟢 CARGAS INCREMENTALES:
   • Modo: Append (acumulación histórica)
   • Filtrado: WHERE OrderDate BETWEEN fechaDesde AND fechaHasta
   • Aplicación: Tablas transaccionales
   • Cantidad: 2 de 22 tablas (9%)
   • Ejemplo: Sales.SalesOrderHeader, Sales.SalesOrderDetail
   • Parámetros automáticos para ejecución sin intervención manual

COMPARATIVA:
┌─────────────────┬──────────────┬─────────────────┐
│ Característica  │ FULL         │ INCREMENTAL     │
├─────────────────┼──────────────┼─────────────────┤
│ Modo            │ Overwrite    │ Append          │
│ Volumen         │ Completo     │ Delta (nuevos)  │
│ Rendimiento     │ Mayor carga  │ Menor carga     │
│ Uso             │ Dimensiones  │ Hechos/Transac. │
└─────────────────┴──────────────┴─────────────────┘

11.3 TRAZABILIDAD Y AUDITORÍA
─────────────────────────────
✅ Cada registro incluye 5 columnas de metadatos
✅ Formato Delta Lake con ACID transactions
✅ Time travel (versionado de datos)
✅ Capacidad de regenerar cargas históricas
✅ Auditoría completa por ejecución de pipeline

EJEMPLO DE CONSULTA DE AUDITORÍA:
─────────────────────────────────
SELECT 
    COUNT(*) as registros,
    load_type,
    process_date,
    pl_run_id,
    ingestion_date_ts
FROM bronze.br_sales_salesorderheader
GROUP BY 
    load_type, 
    process_date, 
    pl_run_id,
    ingestion_date_ts
ORDER BY ingestion_date_ts DESC;

11.4 OPTIMIZACIÓN DE ALMACENAMIENTO
───────────────────────────────────
✅ Parquet en Landing con compresión Snappy (60-80% reducción)
✅ Delta Lake en Bronze para consultas eficientes
✅ Particionamiento por fecha en Landing
✅ Wildcards en lectura para procesamiento dinámico

COMPARATIVA DE FORMATOS:
┌─────────────────┬──────────────┬─────────────────┐
│ Característica  │ Parquet      │ Delta Lake      │
├─────────────────┼──────────────┼─────────────────┤
│ Uso             │ Landing      │ Bronze          │
│ Compresión      │ Snappy       │ Snappy + Delta  │
│ ACID            │ ❌ No        │ ✅ Sí          │
│ Time Travel     │ ❌ No        │ ✅ Sí          │
│ Schema Evolution│ Limitado     │ Completo        │
│ Rendimiento     │ Alto         │ Muy Alto        │
└─────────────────┴──────────────┴─────────────────┘

================================================================================
12. TROUBLESHOOTING
================================================================================

❌ ERROR: DATA GATEWAY OFFLINE
──────────────────────────────
🔧 Solución:
   1. Verificar servicio en servidor on-premises
   2. Revisar logs en %ProgramFiles%\On-premises data gateway
   3. Reiniciar servicio
   4. Validar conectividad de red

❌ ERROR: TABLA NO ENCONTRADA EN BRONZE
───────────────────────────────────────
🔧 Solución:
   1. Verificar is_active = 1 en config.table_load_config
   2. Revisar logs de actividad ForEach
   3. Validar permisos en AdventureWorks2019
   4. Verificar path en Landing

❌ ERROR: CARGA INCREMENTAL DUPLICA DATOS
─────────────────────────────────────────
🔧 Solución:
   1. Verificar parámetros fechaDesde/fechaHasta
   2. Confirmar modo Append (no Overwrite)
   3. Revisar filtro WHERE OrderDate
   4. Validar unicidad en destino

❌ ERROR: PIPELINE FALLA EN COPY DATA
─────────────────────────────────────
🔧 Solución:
   1. Verificar linked services configurados correctamente
   2. Validar conexión a SQL Server
   3. Revisar permisos de lectura/escritura en lakehouses
   4. Verificar sintaxis de queries dinámicos


================================================================================
👨‍💻 AUTOR
================================================================================

Diego Euler
Microsoft Fabric - TP
Trabajo Práctico: Pipeline de Ingesta AdventureWorks

================================================================================
📄 LICENCIA
================================================================================

Proyecto educativo - Microsoft Fabric TP
Arquitectura Medallón con cargas FULL e INCREMENTALES

