<h1 align="center">
<span style="color:#FFD700;">π</span> PI Data Strategy & Consulting
</h1>

# Data Engineering Pipeline: Medallion Architecture 🚀

Este proyecto implementa una solución integral de ingeniería de datos en **Microsoft Fabric** utilizando **PySpark**. La solución sigue la arquitectura de Medallón para procesar datos de e-commerce, desde la ingesta de archivos crudos hasta la consolidación de una capa analítica enriquecida mediante APIs externas.

## 📋 Descripción del Proyecto

El objetivo es transformar datos dispersos en una fuente de verdad única (**One Big Table**). El pipeline gestiona la calidad del dato de forma progresiva, aplicando reglas de negocio y enriquecimiento geográfico y financiero en tiempo real.

### Características Principales:
* **Procesamiento Distribuido:** Uso de PySpark para manejar grandes volúmenes de datos.
* **Arquitectura Medallón:**
    * **Bronze (Files):** Ingestión de 7 datasets en formato Parquet con metadatos de auditoría.
    * **Silver (Tables):** Limpieza profunda y enriquecimiento mediante consumo de **APIs REST**.
    * **Gold (Analytical):** Creación de una OBT (One Big Table) con más de 120,000 registros.
* **Enriquecimiento Dinámico:** Integración con `ExchangeRate API` y `REST Countries API`.

---

## 🏗️ Arquitectura de la Solución

El flujo de datos está diseñado para garantizar la trazabilidad y calidad en cada etapa dentro de **Microsoft Fabric**:

### Flujo de Trabajo:
1.  **Ingestión (Bronze):** Captura de archivos CSV desde el Lakehouse de origen y preservación del estado crudo en Parquet.
2.  **Transformación (Silver):** Aplicación de tipado estricto, limpieza de nulos y normalización de divisas.
3.  **Consolidación (Gold):** Join masivo de dimensiones y hechos para generar una tabla lista para BI.

---

## ⚙️ Configuración y Tecnologías

El stack tecnológico utilizado asegura escalabilidad y rendimiento:

| Componente | Tecnología | Propósito |
| :--- | :--- | :--- |
| **Engine** | PySpark (Spark SQL) | Transformación y procesamiento de datos. |
| **Plataforma** | Microsoft Fabric | Entorno de orquestación y almacenamiento OneLake. |
| **Formato** | Delta Lake / Parquet | Almacenamiento optimizado con soporte ACID. |
| **APIs** | REST (Python Requests) | Enriquecimiento de tipos de cambio y geografía. |

### Métricas Finales:
* **Registros Procesados:** 120,348 registros finales.
* **Atributos:** 57 columnas enriquecidas en la capa Gold.

---

## 📁 Estructura del Repositorio

* `01_Bronze_Ingestion.ipynb`: Notebook de carga inicial y estandarización.
* `02_Silver_Transformations.ipynb`: Lógica de limpieza e integración con APIs.
* `03_Gold_OBT.ipynb`: Construcción de la tabla maestra analítica.
* `Documentacion.pdf`: Memoria técnica detallada con diagramas de arquitectura.

---

## 🚀 Cómo Ejecutar
1. Importar los Notebooks en un Workspace de **Microsoft Fabric**.
2. Configurar los Lakehouses de destino (`lh_bronze`, `lh_silver`, `lh_gold`).
3. Ejecutar secuencialmente los notebooks para completar el ciclo de vida del dato.

---

<p align="center">
<b>Desarrollado por:</b> Diego Euler | <b>Programa:</b> Get Talent v4 - Data Engineering
</p>