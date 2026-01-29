# Optimización del Lakehouse: OPTIMIZE, ZORDER y Data Skipping

## Objetivo

Este documento describe la estrategia de optimización aplicada sobre la capa **Gold** del Data Lake, explicando cómo el uso de **OPTIMIZE** y **ZORDER** en Delta Lake permite:

* Reducir la latencia de consultas analíticas
* Minimizar el I/O mediante **data skipping**
* Compactar archivos pequeños (small files problem)
* Justificar decisiones de particionado y su impacto

La optimización se aplicó sobre la tabla:

```
workspace.gold.hvfhs_kpis_diarios
```

---

## Contexto de la tabla Gold

La tabla Gold contiene **KPIs diarios agregados** derivados del dataset HVFHS de NYC TLC. Sus principales características son:

* Bajo volumen de datos (1 registro por día)
* Uso principal en consultas analíticas y dashboards
* Filtros frecuentes por rangos de fecha (`trip_date`)

Ejemplo típico de consulta:

```sql
SELECT *
FROM workspace.gold.hvfhs_kpis_diarios
WHERE trip_date BETWEEN '2025-01-01' AND '2025-01-15';
```

---

## Estrategia de optimización aplicada

### 1 OPTIMIZE – Compactación de archivos

Se ejecutó el comando:

```sql
OPTIMIZE workspace.gold.hvfhs_kpis_diarios
ZORDER BY (trip_date);
```

**OPTIMIZE** realiza una compactación física de los archivos Parquet que componen la tabla Delta, resolviendo el *small files problem*.

Beneficios:

* Reduce el número de archivos (`numFiles`)
* Disminuye el overhead de metadata
* Mejora la eficiencia de lectura

Tras la ejecución, la tabla quedó compuesta por un único archivo compacto, lo cual es esperado dado el bajo volumen agregado de la capa Gold.

---

### 2 ZORDER – Organización física de los datos

El uso de **ZORDER BY (trip_date)** reorganiza físicamente los datos dentro de los archivos Parquet, agrupando valores similares de la columna `trip_date`.

 Importante:
* Su efecto es físico y se refleja en el plan de ejecución

---

## Data Skipping: cómo se reduce la latencia

### ¿Qué es Data Skipping?

Delta Lake mantiene estadísticas por archivo (min, max, count, nulls) para cada columna. Durante una consulta con filtros, el motor puede **descartar archivos completos** que no contienen los valores buscados.

Esto evita leer archivos innecesarios y reduce significativamente el I/O.

---

### Relación entre ZORDER y Data Skipping

Gracias a ZORDER:

* Los valores de `trip_date` quedan físicamente agrupados
* Las estadísticas min/max por archivo son más precisas
* Se incrementa la efectividad del data skipping

---

### Evidencia mediante EXPLAIN

Al ejecutar:

```sql
EXPLAIN
SELECT *
FROM workspace.gold.hvfhs_kpis_diarios
WHERE trip_date = '2025-01-10';
```

Se observa en el plan de ejecución:

* **DataFilters** aplicados a nivel de lectura NO despues de la lectura
* **DictionaryFilters** sobre `trip_date`
* Uso de **PhotonScan parquet**
* Lectura de un único archivo optimizado

Esto confirma que el motor puede aplicar **data skipping efectivo**, reduciendo la latencia y el consumo de recursos.

---

### ¿Por qué NO se particiona la tabla Gold?

La tabla presenta:

* Muy baja cardinalidad
* Un registro por día
* Bajo volumen total

Particionar por fecha generaría:

* Carpetas con muy pocos datos
* Overhead innecesario de metadata
* Mayor complejidad en esta prueba

 Por esta razón, se opta por **no usar particiones** (`partitionColumns = []`).

---

##  Impacto en consultas analíticas

La estrategia aplicada permite:

*  Menor latencia en consultas por fecha
*  Menor cantidad de archivos leídos
*  Menor uso de memoria y CPU
*  Mejor experiencia en dashboards y BI

Especialmente en consultas de tipo:

* KPIs diarios
* Comparaciones por rangos temporales
* Agregaciones históricas

---

## Conclusión

La combinación de **OPTIMIZE + ZORDER** en la capa Gold permite:

* Resolver el small files problem
* Habilitar data skipping efectivo
* Reducir significativamente el I/O
* Mejorar la latencia de consultas analíticas

Todo esto sin incurrir en sobre-particionamiento, alineándose con buenas prácticas de diseño de arquitecturas Lakehouse en Delta Lake.

---

