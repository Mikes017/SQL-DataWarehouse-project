# Data Warehouse Medallion: tareas del proyecto y convenciones de nombres

Proyecto: data warehouse de ventas que integra dos sistemas fuente (**CRM** y **ERP**, en archivos CSV). Usa la arquitectura **Medallion** (Bronze → Silver → Gold) en **SQL Server 2022 Express** dentro de Docker.

- Diagrama de arquitectura: [`medallion_architecture.drawio`](medallion_architecture.drawio) / [`.png`](medallion_architecture.png)
- Conexión y comandos de Docker: `../../Docker/SQLServer-Docker.md`

**Contenido**

1. [Tareas detalladas](#parte-1-tareas-detalladas)
   - [Fase 0. Entorno](#fase-0-entorno)
   - [Fase 1. Análisis de requerimientos](#fase-1-análisis-de-requerimientos)
   - [Fase 2. Diseño de la arquitectura](#fase-2-diseño-de-la-arquitectura)
   - [Fase 3. Inicialización del proyecto](#fase-3-inicialización-del-proyecto)
   - [Fase 4. Capa Bronze](#fase-4-capa-bronze)
   - [Fase 5. Capa Silver](#fase-5-capa-silver)
   - [Fase 6. Capa Gold](#fase-6-capa-gold)
   - [Fase 7. Documentación y entrega](#fase-7-documentación-y-entrega)
   - [Fase 8 (opcional). Análisis y reportes](#fase-8-opcional-análisis-y-reportes)
2. [Convenciones de nombres](#parte-2-convenciones-de-nombres)

---

## Parte 1. Tareas detalladas

Leyenda: `[x]` hecho · `[ ]` pendiente

### Fase 0. Entorno

- [x] Instalar SQL Server 2022 Express (contenedor Docker `sqlserver`, puerto 1433)
- [x] Instalar el cliente: extensión **SQL Server (mssql)** de VS Code (SSMS no existe para Linux)
- [x] Copiar los CSV al contenedor, en `/var/opt/mssql/datasets/`

  ```bash
  docker exec sqlserver mkdir -p /var/opt/mssql/datasets
  docker cp datasets/source_crm sqlserver:/var/opt/mssql/datasets/
  docker cp datasets/source_erp sqlserver:/var/opt/mssql/datasets/
  ```
  Si cambias algún CSV, vuelve a copiarlo con los mismos comandos.

- [ ] Conectarte desde VS Code y comprobar que respondes `SELECT @@VERSION;`

### Fase 1. Análisis de requerimientos

- [ ] Leer y resumir los requerimientos del proyecto:
  - **Fuentes:** CRM y ERP en CSV.
  - **Calidad:** limpiar y resolver problemas de calidad antes del análisis.
  - **Integración:** un solo modelo de datos amigable para consultas analíticas.
  - **Alcance:** solo el dataset más reciente; no se requiere historial (no hay SCD tipo 2).
  - **Documentación:** del modelo de datos, para negocio y para analistas.
- [ ] Definir las preguntas de negocio que el modelo debe responder: comportamiento de clientes, desempeño de productos y tendencias de ventas.

### Fase 2. Diseño de la arquitectura

- [x] Elegir el enfoque: **Medallion** (frente a Inmon, Kimball o Data Vault)
- [x] Definir qué hace cada capa:

  | | Bronze | Silver | Gold |
  |---|---|---|---|
  | Definición | Datos crudos, tal cual vienen de la fuente | Datos limpios y estandarizados | Datos listos para negocio |
  | Objetivo | Trazabilidad y depuración | Preparar los datos para análisis | Consumo para reportes y analítica |
  | Tipo de objeto | Tablas | Tablas | Vistas |
  | Método de carga | Full load (truncate & insert) | Full load (truncate & insert) | Ninguno |
  | Transformaciones | Ninguna | Limpieza, estandarización, normalización, columnas derivadas, enriquecimiento | Integración, agregación, reglas de negocio |
  | Modelado | Ninguno | Ninguno | Esquema estrella |
  | Audiencia | Data engineers | Data engineers y analistas | Analistas y usuarios de negocio |

- [x] Dibujar el diagrama de arquitectura (`docs/medallion_architecture.drawio`)
- [ ] Explicar en el reporte el principio de **separación de responsabilidades** (SoC): cada capa tiene una sola responsabilidad y no se mezclan.

### Fase 3. Inicialización del proyecto

- [ ] Leer y adoptar las [convenciones de nombres](#parte-2-convenciones-de-nombres)
- [ ] Crear el repositorio Git con la estructura `datasets/`, `docs/`, `scripts/{bronze,silver,gold}/` y `tests/`
- [ ] Crear la base de datos y los esquemas con `scripts/init_database.sql`
  - ⚠️ Este script **borra y vuelve a crear** la BD `DataWarehouse` si ya existe.
  - Crea la BD `DataWarehouse` y los esquemas `bronze`, `silver` y `gold`.
- [ ] Verificarlo con `SELECT name FROM sys.schemas WHERE name IN ('bronze','silver','gold');`

### Fase 4. Capa Bronze

**Objetivo:** cargar los 6 CSV tal cual, sin transformar nada.

- [ ] **Analizar los sistemas fuente.** Documenta, para cada fuente:
  - quién es el dueño de los datos y qué proceso de negocio cubre
  - formato y volumen
  - si la carga es completa o incremental
  - cómo se conecta (aquí son archivos en carpetas)
- [ ] **Crear las tablas** con `scripts/bronze/ddl_bronze.sql`. Las columnas llevan los mismos nombres que en el CSV.
- [ ] **Adaptar `scripts/bronze/proc_load_bronze.sql` a Docker/Linux.** Hay que hacer dos cambios:
  1. **Rutas.** El script usa rutas de Windows (`C:\sql\dwh_project\datasets\...`). Cámbialas por las del contenedor:

     | Tabla | Ruta nueva |
     |---|---|
     | `bronze.crm_cust_info` | `/var/opt/mssql/datasets/source_crm/cust_info.csv` |
     | `bronze.crm_prd_info` | `/var/opt/mssql/datasets/source_crm/prd_info.csv` |
     | `bronze.crm_sales_details` | `/var/opt/mssql/datasets/source_crm/sales_details.csv` |
     | `bronze.erp_loc_a101` | `/var/opt/mssql/datasets/source_erp/LOC_A101.csv` |
     | `bronze.erp_cust_az12` | `/var/opt/mssql/datasets/source_erp/CUST_AZ12.csv` |
     | `bronze.erp_px_cat_g1v2` | `/var/opt/mssql/datasets/source_erp/PX_CAT_G1V2.csv` |

  2. **Fin de línea de `LOC_A101.csv`.** Este archivo usa finales de línea de Windows (CRLF); los demás usan LF. Sin ajuste, cada valor de `cntry` termina con un `\r` invisible. Entonces `TRIM(cntry) = 'DE'` nunca coincide en Silver, y los países no se normalizan. Para corregirlo, agrega `ROWTERMINATOR` **solo** en ese `BULK INSERT`:
     ```sql
     BULK INSERT bronze.erp_loc_a101
     FROM '/var/opt/mssql/datasets/source_erp/LOC_A101.csv'
     WITH (
         FIRSTROW = 2,
         FIELDTERMINATOR = ',',
         ROWTERMINATOR = '0x0d0a',
         TABLOCK
     );
     ```
     Lo probé en el contenedor: sin `ROWTERMINATOR`, las 18,484 filas quedan con `\r`; con `'0x0d0a'`, ninguna.

- [ ] **Crear y ejecutar** el procedimiento: `EXEC bronze.load_bronze;`
  - Ya incluye `TRY...CATCH`, mensajes `PRINT` y la duración de cada tabla.
- [ ] **Validar los conteos.** Cada tabla debe tener tantas filas como el CSV (sin contar el encabezado):

  | Tabla | Filas esperadas |
  |---|---|
  | `bronze.crm_cust_info` | 18,494 |
  | `bronze.crm_prd_info` | 397 |
  | `bronze.crm_sales_details` | 60,398 |
  | `bronze.erp_cust_az12` | 18,484 |
  | `bronze.erp_loc_a101` | 18,484 |
  | `bronze.erp_px_cat_g1v2` | 37 |

- [ ] **Revisar que las columnas no estén corridas.** Compara un `SELECT TOP 10 *` de cada tabla con el CSV.
- [ ] **Documentar el flujo de datos** Fuente → Bronze (diagrama *data flow*).
- [ ] Hacer commit: `feat(bronze): add ddl and load procedure`

### Fase 5. Capa Silver

**Objetivo:** limpiar y estandarizar cada tabla. Se mantiene la misma estructura que en la fuente.

- [ ] **Explorar los datos y entender cómo se relacionan las tablas** (diagrama de integración):
  - `crm_cust_info.cst_key` ↔ `erp_cust_az12.cid` (en el ERP lleva el prefijo `NAS`) ↔ `erp_loc_a101.cid` (en el ERP lleva un guion)
  - `crm_prd_info.prd_key`: sus primeros 5 caracteres son `cat_id`, que se une con `erp_px_cat_g1v2.id`
  - `crm_sales_details.sls_prd_key` → `crm_prd_info.prd_key`; `sls_cust_id` → `crm_cust_info.cst_id`
- [ ] **Crear las tablas** con `scripts/silver/ddl_silver.sql`. Incluye la columna técnica `dwh_create_date DATETIME2 DEFAULT GETDATE()`.
- [ ] **Para cada tabla:** revisa los problemas en Bronze, transforma, inserta en Silver y vuelve a revisar en Silver.

  | Tabla | Problemas detectados y transformación |
  |---|---|
  | `crm_cust_info` | IDs nulos o duplicados: se queda el registro más reciente (`ROW_NUMBER() ... ORDER BY cst_create_date DESC`). Espacios en nombres: `TRIM`. Códigos: `S/M` → `Single/Married`, `F/M` → `Female/Male`, el resto → `n/a`. |
  | `crm_prd_info` | `prd_key` se parte en `cat_id` (con `-` → `_`) y `prd_key`. Costo nulo → `0`. `M/R/S/T` → `Mountain/Road/Other Sales/Touring`. `prd_end_dt` se recalcula como `LEAD(prd_start_dt) - 1` para que las fechas no se traslapen. |
  | `crm_sales_details` | Fechas en formato entero `YYYYMMDD`: si valen 0 o no tienen 8 dígitos → `NULL`; si no, se convierten a `DATE`. Regla de negocio `sales = quantity × price`: se recalcula `sls_sales` si es nulo, negativo o no cuadra, y se deriva `sls_price` si es inválido. |
  | `erp_cust_az12` | Se quita el prefijo `NAS` de `cid`. Fechas de nacimiento futuras → `NULL`. Género normalizado a `Female/Male/n/a`. |
  | `erp_loc_a101` | Se quita el guion de `cid`. `DE` → `Germany`, `US/USA` → `United States`, vacío → `n/a`. (Depende del arreglo de CRLF de la Fase 4.) |
  | `erp_px_cat_g1v2` | Los datos ya vienen limpios; se cargan sin cambios. |

- [ ] **Crear y ejecutar** el procedimiento: `EXEC silver.load_silver;`
- [ ] **Correr las pruebas de calidad** `tests/quality_checks_silver.sql`. Cada consulta debe devolver **0 filas**. Revisan:
  - llaves primarias nulas o duplicadas
  - espacios sobrantes
  - costos nulos o negativos
  - fechas inválidas o en orden incorrecto
  - consistencia de `sales = quantity × price`
- [ ] Actualizar el diagrama de flujo de datos con Bronze → Silver
- [ ] Hacer commit: `feat(silver): add data cleansing and load procedure`

### Fase 6. Capa Gold

**Objetivo:** un modelo estrella orientado a negocio, construido como vistas sobre Silver.

- [ ] **Identificar los objetos de negocio:** Clientes, Productos y Ventas.
- [ ] **Construir las vistas** con `scripts/gold/ddl_gold.sql`:

  | Vista | Tipo | Se construye con | Notas |
  |---|---|---|---|
  | `gold.dim_customers` | Dimensión | `crm_cust_info` + `erp_cust_az12` + `erp_loc_a101` | Surrogate key `customer_key`. El género del CRM tiene prioridad; si viene como `n/a`, se usa el del ERP. |
  | `gold.dim_products` | Dimensión | `crm_prd_info` + `erp_px_cat_g1v2` | Surrogate key `product_key`. Solo productos vigentes (`prd_end_dt IS NULL`). |
  | `gold.fact_sales` | Hechos | `crm_sales_details` + las dos dimensiones | Se une a las dimensiones por sus surrogate keys. Medidas: `sales_amount`, `quantity`, `price`. |

- [ ] Renombrar las columnas a nombres de negocio en inglés y `snake_case` (por ejemplo, `cst_firstname` → `first_name`).
- [ ] Revisar que los joins **no dupliquen filas**: compara el conteo antes y después de cada `LEFT JOIN`.
- [ ] **Correr las pruebas de calidad** `tests/quality_checks_gold.sql`:
  - las surrogate keys deben ser únicas
  - ningún hecho debe quedar sin su dimensión (0 filas)
- [ ] **Diagramar el modelo estrella** en draw.io (`fact_sales` en el centro, con las 2 dimensiones y la cardinalidad 1:N)
- [ ] **Escribir o revisar el catálogo de datos** (`docs/data_catalog.md`): columna, tipo y descripción de cada vista.
- [ ] Completar el diagrama de flujo de datos hasta Gold
- [ ] Hacer commit: `feat(gold): add star schema views`

### Fase 7. Documentación y entrega

- [ ] README propio con objetivo, arquitectura (incluye el PNG), cómo correr el proyecto (Docker, luego `init_database`, `load_bronze`, `load_silver`, `ddl_gold`) y estructura del repo
- [ ] Diagramas finales exportados a PNG: arquitectura, flujo de datos, integración y modelo estrella
- [ ] Prueba de punta a punta: con la BD vacía, correr todo en orden, ejecutar las pruebas y que pasen
- [ ] Commit final y push

### Fase 8 (opcional). Análisis y reportes

- [ ] **EDA:** exploración de dimensiones (países, categorías), rangos de fechas, medidas totales, magnitudes y rankings (top/bottom N)
- [ ] **Analítica avanzada:** cambios en el tiempo, acumulados, desempeño año contra año, segmentación de clientes y productos, parte-del-todo
- [ ] **Vistas de reporte:** `gold.report_customers` y `gold.report_products`

---

## Parte 2. Convenciones de nombres

### Principios generales

- **`snake_case`:** minúsculas con guion bajo (`_`) entre palabras.
- **Idioma:** inglés para todos los nombres de objetos (tablas, columnas, procedimientos). Los comentarios y la documentación pueden ir en español.
- **Palabras reservadas:** no usar palabras reservadas de SQL como nombres (`order`, `date`, `user`, `key`…).
- **Consistencia:** el mismo concepto lleva el mismo nombre en todas las capas.

### Base de datos y esquemas

| Objeto | Nombre |
|---|---|
| Base de datos | `DataWarehouse` |
| Esquemas (uno por capa) | `bronze`, `silver`, `gold` |

### Tablas

#### Bronze y Silver: `<sourcesystem>_<entity>`

- `<sourcesystem>`: sistema de origen en minúsculas (`crm`, `erp`).
- `<entity>`: nombre **exacto** de la tabla o archivo en la fuente, sin renombrar.
- Ejemplos: `crm_cust_info`, `erp_loc_a101`.
- Las columnas también conservan su nombre original (`cst_id`, `prd_key`, `sls_ord_num`, `cid`, `cntry`…).

#### Gold: `<category>_<entity>`

- `<category>`: el papel del objeto (`dim`, `fact`, `report`).
- `<entity>`: nombre de negocio, descriptivo y **en plural**.

| Prefijo | Significado | Ejemplos |
|---|---|---|
| `dim_` | Dimensión | `dim_customers`, `dim_products` |
| `fact_` | Tabla de hechos | `fact_sales` |
| `report_` | Vista de reporte o agregada | `report_customers`, `report_sales_monthly` |

### Columnas

| Tipo | Patrón | Ejemplo | Regla |
|---|---|---|---|
| Surrogate key | `<entity>_key` | `customer_key`, `product_key` | Llave primaria de cada dimensión, generada en el DWH (`ROW_NUMBER()`). |
| Llave de negocio | `<entity>_id` / `<entity>_number` | `customer_id`, `product_number` | Viene del sistema fuente. |
| Columna técnica | `dwh_<column_name>` | `dwh_create_date` | Metadatos que genera el DWH, no la fuente. |
| Fechas | `<event>_date` | `order_date`, `shipping_date` | En Gold, siempre con el sufijo `_date`. |
| Medidas | nombre de negocio | `sales_amount`, `quantity`, `price` | Solo en tablas `fact_`. |

En Gold, las columnas se renombran de su nombre técnico a uno de negocio:

| Silver | Gold |
|---|---|
| `cst_firstname` | `first_name` |
| `cst_gndr` | `gender` |
| `cntry` | `country` |
| `prd_nm` | `product_name` |
| `sls_order_dt` | `order_date` |
| `sls_sales` | `sales_amount` |

### Valores estandarizados (Silver y Gold)

- Códigos expandidos a texto legible: `M` → `Male` / `Married` (según la columna), `DE` → `Germany`.
- Valores desconocidos o vacíos: `'n/a'`, nunca cadena vacía.
- Fechas inválidas: `NULL`.

### Procedimientos almacenados: `load_<layer>`

| Procedimiento | Qué hace |
|---|---|
| `bronze.load_bronze` | Carga los CSV en Bronze (truncate + `BULK INSERT`) |
| `silver.load_silver` | Carga Silver desde Bronze (truncate + `INSERT ... SELECT`) |

Gold no tiene procedimiento: son vistas, y siempre reflejan Silver.

### Archivos y scripts

| Tipo | Patrón | Ejemplo |
|---|---|---|
| DDL por capa | `scripts/<layer>/ddl_<layer>.sql` | `scripts/silver/ddl_silver.sql` |
| Carga por capa | `scripts/<layer>/proc_load_<layer>.sql` | `scripts/bronze/proc_load_bronze.sql` |
| Pruebas de calidad | `tests/quality_checks_<layer>.sql` | `tests/quality_checks_gold.sql` |
| Diagramas | `docs/<topic>.drawio` (+ `.png` exportado) | `docs/medallion_architecture.drawio` |
| Documentación | `docs/<topic>.md` | `docs/data_catalog.md` |

### Commits de Git

Formato: `<type>(<layer>): <short description>`

- Tipos: `feat` (algo nuevo), `fix` (corrección), `docs` (documentación), `test` (pruebas), `refactor` (reorganizar sin cambiar el comportamiento).
- Ejemplos: `feat(bronze): add load_bronze procedure`, `fix(bronze): set CRLF rowterminator for loc_a101`, `docs: add star schema diagram`.
