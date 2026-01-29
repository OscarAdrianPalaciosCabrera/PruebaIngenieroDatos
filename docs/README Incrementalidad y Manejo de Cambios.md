# Incrementalidad Avanzada y Manejo de Cambios (CDC)

## Dataset HVFHS – Enfoque Lakehouse (Bronze / Silver / Gold)

---

## 1. Contexto del dataset

El dataset **HVFHS (High Volume For-Hire Services)** contiene información de viajes de alto volumen asociados a servicios como Uber y Lyft. Por su naturaleza operativa, presenta los siguientes retos:

* Alto volumen de registros diarios
* Llegadas potencialmente tardías
* Posibles reprocesos por correcciones de datos
* Consumo principalmente analítico mediante métricas agregadas diarias (capa Gold)

### Columnas clave

* `trip_date`
* `pickup_datetime`
* `dropoff_datetime`
* `trip_time`
* `trip_miles`
* `base_passenger_fare`
* Flags operativos (`shared_request_flag`, `wav_request_flag`, etc.)

---

## 2. Estrategia de incrementalidad por capa

### Bronze – Ingesta cruda (append-only)

#### Rol

* Almacenar los datos **tal como llegan desde la fuente**
* Mantener trazabilidad completa para auditoría y reprocesos

#### Estrategia incremental

* Carga **append-only**
* No se eliminan ni actualizan registros
* Se agrega metadata de ingesta:

  * `ingestion_timestamp`
  * `source_file`

#### Justificación

* Permite auditoría completa
* Facilita reprocesos controlados
* Evita pérdida de información original

---

### Silver – Datos depourados y consistentes

#### Rol

* Limpieza de datos
* Tipificación correcta de columnas
* Validaciones de negocio
* Normalización de timestamps (zona horaria, nulos)
* Control de calidad y duplicados

---

## 3. Manejo de reprocesos

### Escenario

Se detectan errores en:

* Cálculo de duración del viaje
* Normalización de timestamps
* Reglas de validación

### Estrategia aplicada

* Reprocesos **por rango de fechas (`trip_date`)**

Ejemplo:

```sql
DELETE FROM workspace.silver.hvfhs_trips
WHERE trip_date BETWEEN '2025-01-01' AND '2025-01-07';
```

Posteriormente, se recalcula únicamente ese rango desde Bronze.

#### Justificación

* El volumen de datos es elevado
* Reprocesar todo el histórico sería costoso
* `trip_date` es una clave natural del negocio

---

## 4. Manejo de llegadas tardías

### Escenario

Un viaje correspondiente al `2025-01-10` llega al sistema el `2025-01-12`.

### Estrategia propuesta

* Toleranica en rango de reproceso (por ejemplo, últimos 7 días)
* Uso de `MERGE` en la capa Silver utilizando la clave natural del viaje

```sql
MERGE INTO workspace.silver.hvfhs_trips t
USING workspace.bronze.hvfhs_trips s
ON t.trip_id = s.trip_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;
```

#### Justificación

* Permite corregir viajes incompletos o tardíos
* Evita reprocesar meses completos de información

---

## 5. Manejo de duplicados

### Escenario

El mismo `trip_id` puede llegar más de una vez desde la fuente.

### Estrategia aplicada

* Deduplicación en Silver
* Se conserva el registro más reciente según `ingestion_timestamp`

```sql
ROW_NUMBER() OVER (
  PARTITION BY trip_id
  ORDER BY ingestion_timestamp DESC
)
```

#### Justificación

* Evita inflar métricas en la capa Gold
* Garantiza unicidad lógica por viaje

---

## 6. Versionado de datos

### Append (Bronze)

* Cada carga agrega nuevos registros
* Nunca se sobrescriben datos

### Merge (Silver)

* Permite correcciones y upserts
* Ideal para datasets operativos como HVFHS

### Soft Deletes (opcional)

* Para eliminar logicamente los viajes invalidados se marcan con una nueva columna:

```sql
is_deleted = true
```

* Silver y Gold filtran:

```sql
WHERE is_deleted = false
```

---

## 7. Control de snapshots (Delta Time Travel)

Delta Lake permite consultar versiones históricas:

```sql
SELECT *
FROM workspace.silver.hvfhs_trips
VERSION AS OF 10;
```

Uso principal:

* Auditoría
* Debug
* Comparación antes/después de reprocesos

---

## Impacto en la capa Gold (KPIs)

### Características

* Datos agregados por `trip_date`
* Métricas calculadas:

  * Viajes promedio por hora
  * Ingresos diarios
  * Duración promedio del viaje
  * Distancia promedio del viaje

### Estrategia incremental

* Recalcular únicamente fechas afectadas
* Consumir exclusivamente datos ya validados desde Silver

Ejemplo:

```sql
WHERE trip_date BETWEEN current_date - 7 AND current_date
```

---

## ¿Que obtenemos con este enfoquwe?

* Escalabilidad
* Bajo costo computacional
* Alta trazabilidad
* Alineado al patrón Lakehouse
* Diseño realista para entornos productivos
