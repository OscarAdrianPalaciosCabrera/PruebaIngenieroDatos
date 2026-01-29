# Separación de bases de datos por capa (Bronze / Silver / Gold)

## Contexto

En esta implementación del Data Lake se utilizan **bases de datos distintas para cada capa** (Bronze, Silver y Gold). Esta decisión responde tanto a **buenas prácticas de arquitectura de datos** como a **limitaciones prácticas de una cuenta gratuita de Databricks**.

---

## Arquitectura lógica por capas

Cada capa del Data Lake se materializa en una base de datos independiente:

* `workspace.bronze`
* `workspace.silver`
* `workspace.gold`

Esto permite mantener una separación clara entre:

* Datos crudos
* Datos depurados
* Datos analíticos

---

## Justificación técnica de la separación

### 1 Aislamiento lógico y claridad arquitectónica

El uso de bases de datos separadas por capa facilita:

* Identificación rápida del nivel de procesamiento del dato
* Reducción de errores al consumir información incorrecta
* Lectura y mantenimiento más simples del Data Lake

Desde el punto de vista de ingeniería:

> Una tabla debe expresar claramente su nivel de madurez solo con su ubicación.

---

### 2 Simulación de gobierno de datos en cuenta gratuita

En entornos productivos con **Unity Catalog**, la separación por capas suele realizarse mediante:

* Catálogos
* Esquemas
* Políticas de acceso

Sin embargo, en una **cuenta gratuita de Databricks**:

* No siempre se dispone de Unity Catalog
* No es posible configurar controles avanzados de seguridad

Por ello, el uso de **bases de datos separadas** permite **simular**:

* Aislamiento por dominio
* Control de acceso por capa
* Organización por entorno lógico

---

### 3 Preparación para control de accesos

La separación por base de datos permite definir, incluso de forma conceptual:

* Accesos de solo lectura a Gold
* Accesos restringidos a Bronze
* Acceso de ingeniería a Silver

Ejemplo conceptual:

* Analistas → `workspace.gold`
* Científicos de datos → `workspace.silver`, `workspace.gold`
* Ingenieros de datos → todas las capas

Esto facilita la transición a un entorno productivo real.

---

### 4 Escalabilidad y migración futura

Aunque en esta prueba se utiliza una sola cuenta y un solo workspace, esta estructura permite:

* Migrar fácilmente a Unity Catalog
* Separar catálogos por entorno (`dev`, `qa`, `prod`)
* Aplicar políticas de seguridad sin refactorizar tablas

La arquitectura queda alineada con escenarios empresariales reales.

---

## Relación con la cuenta gratuita

Debido a las limitaciones de la cuenta gratuita:

* Se utiliza un único workspace
* Se simula la separación de dominios mediante bases de datos
* No se crean catálogos adicionales

Esta decisión prioriza:

* Claridad
* Orden
* Buenas prácticas

Sin introducir complejidad innecesaria.

---

## Arquitectura

![Arquitectura](docs/Diagrama.png)

---
## Conclusión

El uso de **bases de datos separadas para cada capa** permite:

* Mantener una arquitectura clara y mantenible
* Simular gobierno de datos en un entorno limitado
* Preparar el diseño para escenarios productivos
* Alinear la solución con buenas prácticas Lakehouse

Esta estrategia es especialmente adecuada para pruebas técnicas y entornos de desarrollo con recursos acotados.
