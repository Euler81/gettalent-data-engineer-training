<h1 align="center">
<span style="color:#FFD700;">π</span> Data Strategy & Consulting
</h1>

# 📊 TP – Microsoft Fabric Data Engineering

Repositorio correspondiente al **Trabajo Práctico 4 (TP4)** del programa **GetTalent – Junior Data Engineer**.

Este proyecto implementa un **pipeline de ingestión de datos en Microsoft Fabric** utilizando arquitectura **Medallion (Landing → Bronze)**.

---

# 🎯 Objetivos del TP

* ☁️ Trabajar con **Microsoft Fabric**
* 🔗 Configurar conexión con **AdventureWorks**
* ⚙️ Crear **Data Pipelines**
* 🪙 Implementar **arquitectura Medallion**
* 🔄 Procesar **cargas Full e Incrementales**
* 🗄️ Almacenar datos en **Lakehouse**

---

# 🌐 Acceso al entorno

Acceder al entorno en:

```id="c8mrg1"
https://app.fabric.microsoft.com
```

### 🔐 Configuración inicial

1️⃣ Cambiar contraseña
2️⃣ Configurar **Microsoft Authenticator**
3️⃣ Acceder al **Workspace asignado**

---

# 🧱 Arquitectura de datos

Se implementa la arquitectura **Medallion**:

```
Source (AdventureWorks)
        │
        ▼
📥 Landing Lakehouse
        │
        ▼
🥉 Bronze Lakehouse
```

### 🗄️ Lakehouses utilizados

* 📥 **Landing Lakehouse**
* 🥉 **Bronze Lakehouse**
* ⚙️ **Lakehouse de configuración**

---

# 🔌 Configuración de Data Gateway

Para conectar con la base local se utiliza:

🖥️ **On-Premises Data Gateway**

Pasos:

1️⃣ Instalar Gateway
2️⃣ Crear conexión
3️⃣ Conectar con **AdventureWorks**

---

# 📂 Extracción de datos

Las tablas a extraer son:

### 🏭 Production

* Product
* ProductSubCategory
* ProductCategory

### 👤 Person

* Contact

### 💰 Sales

* Todas las tablas del esquema **Sales**
* Metadatos desde **tablas del sistema**

---

# ⚙️ Pipeline de datos

Se debe implementar:

✅ **Un único pipeline**

Incluye:

* 📑 tablas de **parametría**
* ⚙️ configuración en **Lakehouse**

---

# 🔄 Estrategia de carga

## 📥 Carga Incremental

Tablas:

* SalesOrderHeader
* SalesOrderDetails

Parámetros del pipeline:

```id="c5hoq8"
fechaDesde
fechaHasta
```

Ejecución:

* Primera ejecución manual
* Luego automática:

```
fechaDesde = fecha actual - 1
fechaHasta = fecha actual
```

⚠️ `fechaHasta` **no debe incluirse en el rango**.

---

## 📦 Carga Full

El resto de las tablas se cargan con:

🔁 **Full Load**

---

# 🗂️ Almacenamiento en Landing

Formato de archivo:

📄 **Parquet**

Ruta de almacenamiento:

```id="ntet8r"
/Files/proyecto/fuente/<schema>/<tabla>/AAAAMMDD/
```

Nombre del archivo:

```id="43q19b"
<tabla>_AAAAMMDD.parquet
```

Ejemplo:

```id="zqpwaw"
/Files/proyecto/fuente/sales/SalesOrderHeader/20240201/SalesOrderHeader_20240201.parquet
```

---

# 🥉 Carga a Bronze

Los datos deben transformarse en:

⚡ **Delta Tables**

Se agregan columnas de metadatos:

| Columna           | Descripción          |
| ----------------- | -------------------- |
| ingestion_date_ts | timestamp de ingesta |
| pl_run_id         | id del pipeline      |

---

# 🛠️ Tecnologías utilizadas

* ☁️ Microsoft Fabric
* 🔄 Data Pipelines
* 🗄️ Lakehouse
* 📄 Parquet
* ⚡ Delta Tables
* 🔌 Data Gateway
* 🗃️ AdventureWorks

---

# 📅 Entrega

🕒 **Fecha límite:**

```
03/02 - 23:59
```

---

# 👨‍💻 Autor

📚 **GetTalent Program**
👨‍💻 Junior Data Engineer

---

# ⭐ Proyecto educativo

Este repositorio forma parte de la formación en **Ingeniería de Datos con Microsoft Fabric**.
