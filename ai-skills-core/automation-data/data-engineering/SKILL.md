---
id: "data-engineering-principles"
version: "1.0.0"
domain: "Data"
technologies:
  - "SQL (ANSI / BigQuery / PostgreSQL)"
  - "Dimensional Modeling (Kimball)"
  - "Data Partitioning & Clustering"
  - "OLTP vs OLAP"
triggers:
  - "Diseñar modelos de datos relacionales o analíticos (Data Warehouses, Data Marts)"
  - "Optimizar costos y tiempos de ejecución de queries SQL en BigQuery o PostgreSQL"
  - "Diseñar tablas de hechos (Fact Tables) y dimensiones (Dimension Tables)"
  - "Estructurar particiones o clusters sobre tablas analíticas de alto volumen"
related_skills:
  - "etl-pipeline-patterns"
  - "gcp-core-services"
source_generation: "core"
---

# Ingeniería de Datos — Modelado Dimensional, Particionamiento y Optimización SQL

> **TL;DR para IA:** Utiliza esta skill para diseñar almacenes de datos analíticos (esquemas en estrella), optimizar el escaneo de bytes en BigQuery mediante particionamiento y clustering, y evitar antipatrones de costo en motores cloud.

---

## 1. Dualidad Arquitectónica: OLTP vs OLAP

| Característica | OLTP (Transaccional - ej. Postgres, MySQL) | OLAP (Analítico - ej. BigQuery, Snowflake, DuckDB) |
|---|---|---|
| **Propósito** | Operación diaria, CRUD de baja latencia | Reportes, agregaciones y análisis masivo |
| **Diseño** | 3ra Forma Normal (3NF) para evitar redundancias | Esquema en Estrella (Star Schema) desnormalizado |
| **Almacenamiento**| Orientado a filas (Row-oriented) | Orientado a columnas (Columnar) |
| **Lecturas** | Pocas columnas de muchas filas puntuales | Pocas columnas sobre millones de filas |

---

## 2. Antipatrones Críticos en Almacenes Analíticos

### ❌ Antipatrón 1: `SELECT *` en motores columnares (ej. BigQuery)
```sql
-- ❌ MAL: Escanea el 100% de las columnas de la tabla. Costo y latencia multiplicados x10
SELECT * FROM `empresa.analitica.eventos` WHERE fecha = '2024-01-01';
```
**Regla**: En almacenes columnares, proyectar **únicamente** las columnas requeridas (`SELECT id, total FROM ...`).

### ❌ Antipatrón 2: Tablas de hechos sin particionamiento por fecha
Consultar 1 día de métricas termina forzando al motor a escanear años enteros de datos históricos.

### ❌ Antipatrón 3: Joins sobre campos de texto libre no indexados / sin clustering
Cruzar dos tablas de millones de registros sobre strings arbitrarios sin claves normalizadas causa *data spilling* a disco y timeouts.

---

## 3. Reglas de Oro

1. **Particionar por Fecha / Timestamp**: En tablas con más de 1 GB o crecimiento diario constante, particionar por la columna temporal principal.
2. **Clustering por Columnas de Filtrado Frecuente**: Aplicar cluster en campos con alta cardinalidad comúnmente usados en cláusulas `WHERE` y `GROUP BY` (ej. `cliente_id`, `pais_codigo`, `estado`).
3. **Esquema en Estrella (Kimball)**:
   - **Fact Table**: Métricas numéricas acumuladas y claves foráneas (`monto_venta`, `cantidad_unidades`).
   - **Dimension Tables**: Contexto textual descriptivo (`dim_cliente`, `dim_producto`, `dim_tiempo`).
4. **Tipos de Datos Mínimos y Precisos**: Usar `INT64`/`INTEGER` o `DATE` en lugar de `STRING` para fechas y números identificadores.

---

## 4. Patrón DDL: Tabla Particionada y Agrupada en BigQuery

```sql
-- Creación de una Fact Table optimizada para costos y velocidad
CREATE OR REPLACE TABLE `proyecto_gcp.analitica.fact_ventas`
(
    id_venta STRING NOT NULL,
    id_cliente INT64 NOT NULL,
    id_producto INT64 NOT NULL,
    fecha_venta DATE NOT NULL,
    monto_total NUMERIC(12, 2) NOT NULL,
    moneda STRING(3) NOT NULL,
    canal_venta STRING(20) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
)
-- 1. Particionamiento: Aísla físicamente las lecturas por día
PARTITION BY fecha_venta
-- 2. Clustering: Ordena internamente por las claves de búsqueda habituales
CLUSTER BY id_cliente, canal_venta
OPTIONS (
    description = "Fact table de transacciones de ventas comerciales",
    require_partition_filter = TRUE -- Obliga al analista/IA a filtrar por fecha, evitando cobros excesivos
);
```

---

## 5. Patrón SQL: Consulta Analítica de Alto Rendimiento

```sql
-- ✅ BIEN: Uso estricto de partición + clustering + agregación temprana
SELECT
    dim_cli.categoria,
    fact.canal_venta,
    COUNT(DISTINCT fact.id_venta) AS total_pedidos,
    SUM(fact.monto_total) AS facturacion_total
FROM
    `proyecto_gcp.analitica.fact_ventas` AS fact
INNER JOIN
    `proyecto_gcp.analitica.dim_cliente` AS dim_cli
    ON fact.id_cliente = dim_cli.id_cliente
WHERE
    -- Filtro mandatorio de partición: escanea únicamente la ventana temporal necesaria
    fact.fecha_venta BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY) AND CURRENT_DATE()
    AND fact.canal_venta IN ('ECOMMERCE', 'APP')
GROUP BY
    1, 2
ORDER BY
    facturacion_total DESC;
```

---

## 6. Checklist de Verificación

- [ ] Todas las tablas analíticas > 1 GB cuentan con particionamiento temporal
- [ ] La opción `require_partition_filter = TRUE` está activa en tablas productivas de alto volumen
- [ ] No existen sentencias `SELECT *` en vistas productivas, dashboards o pipelines
- [ ] Las dimensiones de tipo Slowly Changing Dimensions (SCD Tipo 2) contienen campos `fecha_inicio`, `fecha_fin` e `is_current`
- [ ] Las métricas numéricas financieras utilizan tipos decimales de precisión exacta (`NUMERIC` / `BIGNUMERIC`), nunca `FLOAT64`
