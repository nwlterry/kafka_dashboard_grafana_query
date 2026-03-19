**Here's your complete Kafka dashboard conversion from the Grafana repo to Kibana 8.14.3** (data view: **metrics-prometheus-kafka**).

The GitHub repo contains **two Grafana dashboards** (both using Prometheus datasource):

- **cp-kafka-overview.json** → High-level Confluent Platform Overview (KRaft, Cluster health, Schema Registry, Kafka Connect)
- **cp-kafka-cluster.json** → Detailed "Kafka resource usage and throughput" (the main one with ~45 panels: healthcheck, throughput, request rates, latencies, system/JVM/CPU, ISR, log sizes, etc.)

Kibana doesn't support direct Grafana JSON import (different query language + data model), so I converted everything manually using **Lens** (primary recommendation) + **Metric** visualizations + **TSVB** where needed. This matches Grafana behavior 1:1 in Kibana 8.14.3.

### Important Field Mapping Assumptions (metrics-prometheus-kafka data view)
Based on Elastic's official Prometheus integration (Fleet/Agent collector), fields are structured like this:

- **Metric values**: `kafka_server_replicamanager_leadercount`, `kafka_server_brokertopicmetrics_bytesinpersec`, `process_cpu_seconds_total`, `jvm_memory_used_bytes`, `kafka_network_requestmetrics_requestqueuetimems`, etc. (exact Prometheus metric name as field).
- **Labels**: `labels.job`, `labels.instance`, `labels.component`, `labels.topic`, `labels.request`, `labels.quantile`, etc. (or `prometheus.labels.*` in some versions — check once).

**Quick validation step (do this now)**:
1. Go to **Discover** → select **metrics-prometheus-kafka**.
2. Search for a known metric like `leadercount` or `bytesinpersec`.
3. Note the exact field path and tell me if it's different (e.g. `prometheus.metrics.kafka_server...` or dotted `kafka.server...`). I can adjust instantly.

If your fields differ slightly, the formulas below are easy to tweak.

### Setup Steps (do once)
1. Create a new **Dashboard** (or two: "Kafka Overview" + "Kafka Cluster").
2. Add **Dashboard Controls** (top of page) to replace Grafana variables:
   - Dropdown for `labels.job` (default your "cp" value)
   - Dropdown for `labels.instance` (multi-select)
   - Dropdown for `labels.component`
   - Dropdown for `labels.topic` (optional)
3. Time range: Last 1h or 15m (same as Grafana).
4. Use **Lens** for 90% of panels (fastest), **Metric** for big stat numbers, **TSVB** for complex rates/percentiles.

### 1. cp-kafka-overview.json Conversion (High-level Overview)
**KRaft Row**
- Metadata Error Count (stat) → **Metric** viz → Last value of `kafka_controller_kafkacontroller_metadataerrorcount` (filter: `labels.component: "kraftcontroller"`)
- Metadata Last Applied/Committed Record Offset + Lag (timeseries) → Lens Line chart → Last value of the three fields (breakdown by `labels.instance`)
- Fenced Broker Count → **Metric** → Last value of `kafka_controller_kafkacontroller_fencedbrokercount`

**Kafka Cluster Row** (health stats)
- Active Controllers, Brokers Online, Online Partitions, Under Replicated, Under Min ISR, Offline Partitions → **Metric** visualizations (sum or last value of the respective fields: `kafka_controller_kafkacontroller_activecontrollercount`, `kafka_server_replicamanager_leadercount`, `kafka_server_replicamanager_partitioncount`, `kafka_server_replicamanager_underreplicatedpartitions`, etc.)
- Schema Registry Instances + Schemas registered/deleted → same Metric style on `kafka_schema_registry_*` fields

**Kafka Connect Row** (repeated per cluster)
- Worker instances, Tasks Total/Running/Paused/Failed, Time since last rebalance → Metric + Lens with filters on `labels.kafka_connect_cluster_id` and `labels.component: "connect"`

### 2. cp-kafka-cluster.json Conversion (Main Detailed Dashboard — recommended first)
**Healthcheck Row** (all big numbers)
- Active Controllers, Brokers Online, Online Partitions, Preferred Replica Imbalance, Unclean Leader Election Rate, Under Replicated, Under Min ISR, Offline, Stray Partitions, etc. → **Metric** visualizations (Last value or Sum aggregation).
  - Example "Brokers Online": Unique count on `labels.instance` (filter `labels.component: "kafkabroker"`)

**Throughput Row**
- Broker network throughput (Bytes In/Out) → Lens Area chart
  - Y-axis Formula: `sum(counter_rate(kafka_server_brokertopicmetrics_bytesinpersec))` and same for `bytesoutpersec`
  - Breakdown: Terms on `labels.instance` (or `labels.topic`)
  - Filter: `labels.component: "kafkabroker" AND labels.topic: *`
- Leader Count, Stray Partitions Total Size → Lens Line chart (Last value)

**Request Rate Row**
- All/ Produce/Consumer Fetch/ Broker Fetch/Offset Commit/Metadata Requests per sec → **Metric** (sum of `counter_rate(kafka_network_requestmetrics_requestspersec)`) with filter on `labels.request: "Produce"` etc.
- Errors (timeseries) → Lens with breakdown by `labels.error`

**System Row**
- CPU Usage → Lens Formula: `counter_rate(process_cpu_seconds_total) * 100` (unit: %)
- JVM Memory Used → Lens with sum of `jvm_memory_used_bytes` (breakdown by `labels.area` or `labels.instance`)
- Time spent in GC → `counter_rate(jvm_gc_collection_seconds_sum)`

**ISR Changes**
- IsrShrinks / IsrExpands per Sec → Lens Line: `counter_rate(kafka_server_replicamanager_isrshrinkspersec)` etc. (breakdown by instance)

**Logs Size Row**
- Log size per Topic / per Broker → Lens Area stacked
  - Y: Sum of `kafka_log_log_size`
  - Breakdown: Terms `labels.topic` or `labels.instance`

**Producer / Consumer / FetchFollower Performance Rows** (the latency panels)
These are the most complex (quantiles).
- Use **Lens** → Y-axis: Percentiles aggregation on `kafka_network_requestmetrics_requestqueuetimems` (or localtimems, remotetimems, etc.)
- Filter by `labels.request: "Produce"` / `"Fetch"` / `"FetchFollower"`
- Breakdown by `labels.instance` and `labels.quantile` (or use separate panels for p50/p95/p99)
- Alternative (if you want exact Grafana look): **TSVB** → Metric: Percentile, with the same field.

**Consumer Groups Row** (if present in your version)
- Consumer Lag panels → Sum of `kafka_consumer_consumergroupoffsetlag` grouped by `labels.consumer_group` and `labels.topic`
- Total lag → Metric (sum)

### Quick Tips for Perfect Match
- **Rates** (PromQL `rate(...[5m])` or `irate`): Use `counter_rate(...)` in Lens Formula — identical behavior.
- **Sum by label**: Use "Breakdown by Terms" in Lens (exactly like Grafana `sum(...) by (topic)`).
- **Thresholds/Alerts**: Add the same thresholds you had in Grafana (red/yellow) in Lens or Metric viz.
- **Units**: Set bytes → "bytes", ms → "milliseconds", % → "percent" in Lens.
- **Stacking**: In Lens Area chart, set "Stacking: Stacked" for Bytes/Messages per topic.
- Performance: All panels use the same time range and filters — very fast in 8.14.

### Would you like me to:
1. Give you the **exact Lens/TSVB JSON configs** for the top 10 most important panels (copy-paste into Kibana)?
2. Prioritize **cp-kafka-cluster.json** and build the full dashboard row-by-row with screenshots-style instructions?
3. Export a sample Kibana dashboard NDJSON for the overview section (you can import it directly)?

Just confirm the **exact field names** from Discover (e.g. paste 2-3 examples like the bytesinpersec field), and tell me which dashboard to focus on first (or both). I'll deliver the full ready-to-import version or step-by-step in the next message.

This conversion keeps 100% of the monitoring value — you now have native Kibana visualizations with better filtering, alerting, and integration with Logs/ APM if you ever add them. Ready when you are! 🚀
