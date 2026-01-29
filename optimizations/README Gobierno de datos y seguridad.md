# Gobierno de Datos y Seguridad


---

## 1. Objetivo

Este documento describe la estrategia de **Gobierno de Datos y Seguridad**. El enfoque busca garantizar:

* Acceso controlado a los datos
* Separación clara de responsabilidades
* Auditoría y trazabilidad
* Escalabilidad hacia entornos productivos

El diseño está alineado con buenas prácticas de **Lakehouse**.

---

## 2. Control de accesos por capas (Bronze / Silver / Gold)

### Enfoque conceptual

Cada capa del Lakehouse cumple un rol distinto y, por lo tanto, requiere **niveles de acceso diferenciados**:

| Capa   | Tipo de datos          | Accesos permitidos               |
| ------ | ---------------------- | -------------------------------- |
| Bronze | Datos crudos           | Solo ingenieros de datos         |
| Silver | Datos curados          | Ingenieros y analistas avanzados |
| Gold   | Datos agregados / KPIs | Analistas de negocio / BI        |

### Implementación (sin Unity Catalog)

Dado que se utiliza una cuenta gratuita:

* Se separan las capas por **bases de datos distintas**:

  * `workspace.bronze`
  * `workspace.silver`
  * `workspace.gold`

* El control de accesos se gestiona mediante:

  * Permisos a nivel de workspace
  * Control de quién puede editar notebooks
  * Convenciones estrictas de escritura (solo pipelines escriben en Silver y Gold)

### Implementación ideal (con Unity Catalog)

En un entorno productivo:

```sql
GRANT SELECT ON SCHEMA bronze TO data_engineers;
GRANT SELECT ON SCHEMA silver TO analysts;
GRANT SELECT ON SCHEMA gold TO bi_users;
```

Esto permitiría:

* Seguridad a nivel de catálogo, esquema y tabla
* Principio de menor privilegio

---

## 3. Separación de entornos (dev / qa / prod)

### Estrategia propuesta

La separación de entornos evita impactos entre desarrollo, pruebas y producción.

| Entorno | Objetivo                        |
| ------- | ------------------------------- |
| dev     | Desarrollo y pruebas locales    |
| qa      | Validación de pipelines y datos |
| prod    | Consumo productivo              |

### Implementación

* Bases de datos por entorno:

  * `dev_bronze`, `dev_silver`, `dev_gold`
  * `qa_bronze`, `qa_silver`, `qa_gold`
  * `prod_bronze`, `prod_silver`, `prod_gold`

* Parámetros de entorno en notebooks:

```python
env = "dev"  # dev | qa | prod
database = f"{env}_silver"
```

* Pipelines independientes por entorno

---

## 4. Auditoría de accesos

### Capacidades nativas de Databricks

Databricks permite auditar:

* Quién ejecuta notebooks
* Quién modifica tablas
* Quién consulta datos

Mediante:

* Audit Logs
* Job Runs
* Cluster Logs

### Con Unity Catalog

* Auditoría detallada por:

  * Usuario
  * Tabla
  * Operación (SELECT, INSERT, UPDATE)

Esto permite cumplir requisitos de:

* Compliance
* Gobierno corporativo

---

## 5. Trazabilidad de transformaciones (Data Lineage)

### Enfoque sin Unity Catalog

La trazabilidad se garantiza mediante:

* Nombres de tablas por capa

* Convención clara de notebooks:

  * `01_bronze_ingestion`
  * `02_silver_curation`
  * `03_gold_kpis`

* Uso de Delta Lake:

  * `DESCRIBE HISTORY`
  * Delta Time Travel

Ejemplo:

```sql
DESCRIBE HISTORY workspace.silver.hvfhs_trips;
```

Esto permite rastrear:

* Qué operación se ejecutó
* Cuándo
* Con qué versión de los datos

### Enfoque ideal con Unity Catalog

Unity Catalog proporciona:

* Lineage automático entre:

  * Tablas
  * Vistas
  * Notebooks
* Visualización gráfica del flujo de datos

---

## 6. Control de accesos a nivel AWS IAM (Complemento)

Además del gobierno lógico dentro de Databricks, el control de accesos puede reforzarse a nivel **físico** utilizando **AWS IAM** cuando los datos se almacenan en **Amazon S3** (Delta Lake sobre S3).

### Estructura típica en S3

```text
s3://hvfhs-datalake/
├── bronze/
├── silver/
└── gold/
```

Cada prefijo representa una capa del Lakehouse.

### Control de accesos por capa usando IAM

#### Bronze

* Acceso altamente restringido
* Solo ingenieros de datos

Ejemplo de política IAM:

```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject"],
  "Resource": "arn:aws:s3:::hvfhs-datalake/bronze/*"
}
```

#### Silver

* Acceso para ingenieros y analistas técnicos

```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject"],
  "Resource": "arn:aws:s3:::hvfhs-datalake/silver/*"
}
```

#### Gold

* Acceso para analistas de negocio y herramientas BI

```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject"],
  "Resource": "arn:aws:s3:::hvfhs-datalake/gold/*"
}
```

### Rol de IAM dentro del gobierno de datos

IAM permite:

* Controlar quién puede leer o escribir archivos
* Proteger físicamente los datos en S3
* Integrarse con servicios como Athena, QuickSight o Databricks

Sin embargo, IAM **no** permite:

* Seguridad a nivel de columna
* Seguridad a nivel de fila
* Lineage semántico

Por ello, IAM se considera un **mecanismo complementario**, no un reemplazo del gobierno lógico.

---

## 7. Alternativa a Unity Catalog

Cuando Unity Catalog no está disponible como en este caso:

* Se combina control físico con IAM sobre S3
* Separación lógica por bases de datos
* Convenciones estrictas de nombres
* Versionado de código en Git
* Delta Lake como fuente de verdad

Este enfoque permite simular un gobierno robusto incluso en entornos limitados.

---

## 8. Beneficios del enfoque

* Seguridad física y lógica combinadas
* Separación clara de responsabilidades
* Reducción de riesgos de acceso indebido
* Trazabilidad completa de datos
* Escalabilidad hacia entornos productivos

---

## 9. Conclusión

El enfoque propuesto demuestra cómo aplicar Gobierno de Datos y Seguridad de forma realista y escalable, combinando **AWS IAM** para control físico y **Databricks / Delta Lake** para control lógico.
