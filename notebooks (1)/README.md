# Arquitectura de Capas del Data Lake (Bronze / Silver / Gold)

Este documento describe el propósito, responsabilidades y transformaciones realizadas en cada una de las capas del Data Lake, siguiendo el patrón **Lakehouse** y buenas prácticas de ingeniería de datos.

---

## Capa Bronze – Ingesta cruda de datos

### Objetivo

La capa **Bronze** tiene como objetivo almacenar los datos **tal como llegan desde la fuente**, sin transformaciones de negocio, garantizando:

* Persistencia del dato original
* Trazabilidad
* Capacidad de reprocesamiento
* Auditoría de ingestas

---

### Fuente de datos

* Dataset: **High Volume For-Hire Vehicle Trip Records (HVFHS)**
* Proveedor: NYC TLC
* Periodo principal: Enero 2025

---

### Procesamiento aplicado

En esta capa:

* Se leen los datos desde la fuente original
* **No se modifican tipos de datos ni valores**
* Se agregan columnas de metadata técnica:

  * `ingestion_timestamp`: fecha y hora de ingesta
  * `source_file`: archivo de origen

Esto permite rastrear cuándo y desde dónde ingresó cada registro al Data Lake.

---

### Almacenamiento

* Formato: **Delta Lake (Parquet)**
* Modo de escritura: `append`
* Esquema: `workspace.bronze.hvfhs`

---
### A tener en cuenta
* La capa Bronze es **inmutable**
* No se eliminan ni corrigen registros
* Ante errores, se reprocesa desde esta capa

---

## Capa Silver – Depuración y estandarización

### Objetivo

La capa **Silver** transforma los datos crudos en un dataset:

* Limpio
* Estandarizado
* Con tipos de datos correctos
* Listo para análisis y agregaciones

---

### Transformaciones aplicadas

En esta capa se realizan las siguientes acciones:

#### Conversión explícita de tipos

* Normalización de columnas numéricas
* Conversión de flags (`Y/N`) a valores booleanos

#### Normalización de timestamps

* Conversión a formato estándar
* Uso consistente de zona horaria (UTC)
* Validación de valores nulos

#### Reglas de calidad de datos

Se filtran registros inválidos, por ejemplo:

* `pickup_datetime` nulo
* `dropoff_datetime` anterior al `pickup_datetime`
* Distancias o montos negativos

#### Enriquecimiento técnico

* Cálculo de duración del viaje en minutos

---

### Control de históricos

* Los datos se almacenan en formato Delta
* Se preserva el histórico de registros
* Se habilita el uso de `DESCRIBE HISTORY` para auditoría

---

### Almacenamiento

* Formato: **Delta Lake**
* Esquema: `workspace.silver.hvfhs`
* Datos listos para consumo analítico

---

### Consideraciones de diseño

* La capa Silver es la **fuente confiable de datos**
* Aquí se centraliza la lógica de calidad
* Puede ser base para múltiples capas Gold

---

## Capa Gold – Exposición analítica

### Objetivo

La capa **Gold** contiene datos:

* Agregados
* Modelados para negocio
* Optimizados para consultas analíticas

Es la capa consumida por:

* Dashboards
* Reportes
* Análisis de KPIs

---

### KPIs implementados

Se calculan los siguientes **KPIs diarios**:

* Viajes promedio por hora
* Ingresos totales (fare + fees + impuestos)
* Duración promedio del viaje (minutos)
* Distancia promedio del viaje (millas)

---

### Procesamiento aplicado

* Agregaciones por `trip_date`
* Cálculos estadísticos (AVG, SUM)
* Unión de métricas en una tabla consolidada

---

### Optimización aplicada

* Uso de **OPTIMIZE + ZORDER por `trip_date`**
* Compactación de archivos pequeños
* Mejora del **data skipping**
* Reducción de latencia en consultas por rango temporal

---

### Almacenamiento

* Formato: **Delta Lake**
* Esquema: `workspace.gold.hvfhs_kpis_diarios`

---

### Consideraciones de diseño

* No se particiona la tabla debido a su bajo volumen
* Se prioriza simplicidad y performance
* Diseñada para consumo directo por BI

---

## Conclusión

La separación en capas **Bronze, Silver y Gold** permite:

* Trazabilidad completa del dato
* Reprocesamiento controlado
* Escalabilidad
* Claridad en responsabilidades
* Optimización del performance analítico

Este enfoque sigue las mejores prácticas de arquitecturas **Lakehouse con Delta Lake** y es adecuado para entornos de alto volumen y analítica avanzada.
