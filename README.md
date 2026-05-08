# 📊 Tutorial de Grafana para Big Data

> **Nivel:** Iniciación · **Herramientas:** Grafana, Kafka, Spark, Prometheus, Docker  
> **Objetivo:** Instalar Grafana, conectar fuentes de datos y construir un dashboard de monitorización de pipelines.

---

## Índice

1. [Bloque 1 — Instalación y configuración inicial](#bloque-1--instalación-y-configuración-inicial)
2. [Bloque 2 — Creación del primer dashboard](#bloque-2--creación-del-primer-dashboard)
3. [Bloque 3 — Tipos de visualización](#bloque-3--tipos-de-visualización)
4. [Bloque 4 — Dashboard Kafka + Spark](#bloque-4--dashboard-kafka--spark)
5. [Referencia de queries](#referencia-de-queries)

---

## Bloque 1 — Instalación y configuración inicial

### Opción A · Docker (recomendado para el aula)

Docker garantiza que todos los alumnos tienen exactamente la misma versión y configuración.

```bash
docker run -d \
  --name=grafana \
  -p 3000:3000 \
  grafana/grafana-enterprise
```

| Parámetro | Valor |
|-----------|-------|
| Puerto | `3000` |
| URL de acceso | `http://localhost:3000` |
| Usuario por defecto | `admin` |
| Contraseña por defecto | `admin` (se pide cambiarla al entrar) |

> 💡 **Tip para el aula:** Si los alumnos trabajan en red local, levanta Grafana con Docker en el servidor del aula y comparte la IP. Todos acceden sin instalar nada individualmente.

---

### Opción B · Grafana Cloud (sin instalación)

Crea una cuenta gratuita en [grafana.com](https://grafana.com/). Ideal para Big Data porque ya incluye nativamente:

- ✅ Almacenamiento de métricas (Prometheus)
- ✅ Almacenamiento de logs (Loki)
- ✅ Sin configuración de red local

> ⚠️ **Límite del tier gratuito:** 10 000 series activas de métricas y 50 GB de logs/mes.

---

### Docker Compose completo (con Prometheus)

Para un entorno más completo con persistencia de datos:

```yaml
version: '3.8'

services:
  grafana:
    image: grafana/grafana-enterprise
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=bigdata2024
    depends_on:
      - prometheus

  prometheus:
    image: prom/prometheus
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus

volumes:
  grafana-data:
  prometheus-data:
```

---

## Bloque 2 — Creación del primer dashboard

### Paso 1 · Añadir un Data Source

1. Ve al menú lateral: **Connections → Data Sources**
2. Haz clic en **Add data source**
3. Selecciona la base de datos (para Big Data: PostgreSQL o Prometheus)
4. Introduce las credenciales. En entornos locales, desactiva SSL.
5. Haz clic en **Save & Test** → debe aparecer el check verde ✅

**Data sources recomendados para Big Data:**

| Data Source | Puerto | Caso de uso |
|-------------|--------|-------------|
| PostgreSQL | 5432 | Datos transaccionales, ETL results |
| Prometheus | 9090 | Métricas de infraestructura, Kafka, Spark |
| MySQL | 3306 | Datos de aplicación |
| InfluxDB | 8086 | Series temporales de alta frecuencia |
| Elasticsearch | 9200 | Logs y búsqueda full-text |

---

### Paso 2 · Crear el Dashboard y el primer Panel

1. Menú lateral → icono **+** (Create) → **Dashboard**
2. Clic en **+ Add visualization**
3. Selecciona el data source configurado en el paso anterior

---

### Paso 3 · Query Editor

Aquí es donde los alumnos de Big Data pasan la mayor parte del tiempo.

**Query SQL básica para serie temporal:**

```sql
SELECT
  time_column AS "time",
  value_column AS "value"
FROM your_table
WHERE $__timeFilter(time_column)
ORDER BY time_column ASC
```

**Query SQL con agrupación temporal:**

```sql
SELECT
  $__timeGroupAlias(created_at, '5m'),
  COUNT(*) AS "eventos"
FROM events
WHERE $__timeFilter(created_at)
GROUP BY 1
ORDER BY 1
```

> 💡 **Macros de Grafana más importantes:**
>
> | Macro | Descripción |
> |-------|-------------|
> | `$__timeFilter(col)` | Filtra por el rango de tiempo seleccionado en el panel |
> | `$__timeGroup(col, '5m')` | Agrupa por intervalos de tiempo |
> | `$__timeGroupAlias(col, '5m')` | Igual que el anterior pero añade alias `time` automáticamente |
> | `$__interval` | Devuelve el intervalo óptimo según el zoom actual |

---

### Tip avanzado · Importar dashboards de la comunidad

En lugar de crear desde cero, usa el **Grafana Dashboard Store**:

1. Ve a [dashboards.grafana.com](https://grafana.com/grafana/dashboards/)
2. Busca "PostgreSQL", "Kafka" o "Spark"
3. Copia el **ID numérico** del dashboard
4. En Grafana: **Dashboards → New → Import** y pega el ID

**IDs populares para Big Data:**

| ID | Dashboard |
|----|-----------|
| `9628` | PostgreSQL Database |
| `893` | Docker and system monitoring |
| `3662` | Prometheus 2.0 stats |
| `7589` | Apache Kafka |
| `11883` | Spark Metrics |

---

## Bloque 3 — Tipos de visualización

### Time series — métricas en el tiempo

El más usado en Big Data. Ideal para CPU, memoria, registros/segundo, errores.

```
Cuándo usarlo: cualquier métrica que evoluciona en el tiempo.
Requiere: columna "time" + columna de valor numérico.
```

### Bar gauge — comparativa entre categorías

Útil para comparar el rendimiento de distintos nodos, tablas o servicios.

```
Cuándo usarlo: comparar N elementos en un instante dado.
Soporta: thresholds de color automáticos (verde/amarillo/rojo).
```

### Stat — KPI único destacado

Muestra un único valor grande con sparkline opcional.

```
Cuándo usarlo: totales, latencia media, tasa de error.
Ideal: primera fila del dashboard como resumen ejecutivo.
```

### Table — datos tabulares

Para ver datos en crudo con múltiples campos.

```
Cuándo usarlo: logs de errores, listado de transacciones, estado de particiones.
Permite: ordenar, filtrar, color por columna con overrides.
```

### Gauge — porcentaje de uso o capacidad

Visualiza un valor dentro de un rango, estilo velocímetro.

```
Cuándo usarlo: uso de disco, memoria, CPU.
Los thresholds colorean el arco: verde (ok) → amarillo → rojo.
```

### Opciones clave que aplican a todos los paneles

| Opción | Descripción |
|--------|-------------|
| **Transformations** | Filtra, agrupa o pivota datos sin tocar la query SQL |
| **Overrides** | Aplica colores o umbrales a campos concretos |
| **Thresholds** | Define rangos rojo/amarillo/verde automáticamente |
| **Alertas** | Notifica por Slack, email o webhook cuando un valor supera un umbral |

---

## Bloque 4 — Dashboard Kafka + Spark

### Arquitectura del pipeline

```
Productores → Kafka Brokers → Consumer Groups → Spark Streaming → Almacenamiento
                    ↓                                   ↓
              kafka-exporter                     spark-metrics
                    ↓                                   ↓
                            Prometheus
                                ↓
                             Grafana
```

### Exporters necesarios

**kafka-exporter** — expone métricas de brokers y consumer groups:

```bash
docker run -d \
  --name kafka-exporter \
  -p 9308:9308 \
  danielqsj/kafka-exporter \
  --kafka.server=kafka:9092
```

**spark-metrics** — añade en `spark-defaults.conf`:

```properties
spark.metrics.conf.driver.sink.prometheusServlet.class=org.apache.spark.metrics.sink.PrometheusServlet
spark.metrics.conf.executor.sink.prometheusServlet.class=org.apache.spark.metrics.sink.PrometheusServlet
spark.metrics.conf.driver.sink.prometheusServlet.path=/metrics/prometheus
spark.metrics.conf.executor.sink.prometheusServlet.path=/metrics/prometheus
```

---

### Queries PromQL para los paneles principales

**Throughput de mensajes por topic:**

```promql
sum(rate(kafka_topic_partition_current_offset[1m])) by (topic)
```

**Consumer lag total por grupo:**

```promql
sum(kafka_consumergroup_lag) by (consumergroup, topic)
```

**Records procesados por Spark Streaming:**

```promql
rate(spark_streaming_inputRate_total[1m])
```

**Latencia de batch (p95):**

```promql
histogram_quantile(0.95, rate(spark_streaming_processingDelay_bucket[5m]))
```

**CPU por executor de Spark:**

```promql
spark_executor_cpuTime / spark_executor_runTime * 100
```

**Memoria heap usada:**

```promql
spark_executor_memoryUsed / spark_executor_maxMemory * 100
```

---

### Configuración de alertas

**Alerta de consumer lag crítico:**

```yaml
# En Grafana Alerting → Alert rules → New alert rule
Condition: avg() OF query(A, 5m, now) IS ABOVE 20000
Evaluation interval: 1m
Pending period: 5m

# Notification channel: Slack
Message: |
  🚨 Consumer lag crítico en {{ $labels.topic }}
  Lag actual: {{ $value | humanize }}
  Grupo: {{ $labels.consumergroup }}
```

---

### Estructura recomendada del dashboard

```
┌─────────────────────────────────────────────────────┐
│  ROW 1: Resumen ejecutivo (4 stat panels)           │
│  Mensajes/seg | Consumer lag | Records Spark | Errores │
├──────────────────────────┬──────────────────────────┤
│  ROW 2: Kafka            │                          │
│  Throughput por topic    │  Consumer lag por topic  │
│  (time series)           │  (bar gauge)             │
├──────────────────────────┴──────────────────────────┤
│  ROW 3: Brokers                                     │
│  Estado de brokers Kafka (table)                    │
├──────────────────────────┬──────────────────────────┤
│  ROW 4: Spark            │                          │
│  Records procesados/seg  │  Latencia de batch       │
│  (time series)           │  p50/p95/p99 (time series)│
├─────────────┬────────────┴─────────────┬────────────┤
│  ROW 5:     │                          │            │
│  CPU por    │  Memoria heap            │  Jobs      │
│  executor   │  usada vs libre          │  activos   │
└─────────────┴──────────────────────────┴────────────┘
```

---

## Referencia de queries

### PostgreSQL

```sql
-- Serie temporal de eventos por minuto
SELECT
  $__timeGroupAlias(created_at, '1m'),
  COUNT(*) AS "eventos/min"
FROM events
WHERE $__timeFilter(created_at)
GROUP BY 1
ORDER BY 1;

-- Top 10 tablas por tamaño
SELECT
  relname AS "tabla",
  pg_size_pretty(pg_total_relation_size(relid)) AS "tamaño"
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 10;

-- Tasa de error por servicio
SELECT
  $__timeGroupAlias(ts, '5m'),
  service,
  COUNT(*) FILTER (WHERE status = 'error') * 100.0 / COUNT(*) AS "tasa_error_%"
FROM logs
WHERE $__timeFilter(ts)
GROUP BY 1, 2
ORDER BY 1;
```

### Prometheus / PromQL

```promql
# Uso de CPU del host
100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memoria disponible en %
(node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Tasa de errores HTTP 5xx
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
  /
sum(rate(http_requests_total[5m])) by (service) * 100
```

---

## Recursos adicionales

- 📖 [Documentación oficial de Grafana](https://grafana.com/docs/)
- 🔌 [Grafana Dashboard Store](https://grafana.com/grafana/dashboards/)
- 📦 [kafka-exporter en Docker Hub](https://hub.docker.com/r/danielqsj/kafka-exporter)
- ⚡ [Spark Metrics System](https://spark.apache.org/docs/latest/monitoring.html)
- 🔔 [Grafana Alerting](https://grafana.com/docs/grafana/latest/alerting/)

---

*Tutorial creado para el curso de Big Data · Grafana v10+*
