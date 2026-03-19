Here is a conversion guide for the **cp-kafka-overview.json** Grafana dashboard (high-level Confluent Platform / Kafka overview, focused on KRaft mode + cluster health + Connect/SR) to **Kibana 8.14.3** using the data view **metrics-prometheus-kafka**.

From the dashboard JSON, the Prometheus queries use labels like `job="$job"`, `component="kraftcontroller"` / `"kafkabroker"` / `"connect"` etc. In Kibana (Elastic Prometheus integration style), the fields are typically:

- Metric values: `prometheus.kafka_controller_kafkacontroller_metadataerrorcount.value` (or without `.value` in some setups)
- Labels: `prometheus.labels.job`, `prometheus.labels.component`, `prometheus.labels.instance`

**Critical first step** — verify in **Discover** (data view = metrics-prometheus-kafka):

Search for `metadataerrorcount` → confirm exact field name (likely `prometheus.kafka_controller_kafkacontroller_metadataerrorcount.value` or `kafka_controller_kafkacontroller_metadataerrorcount`).

Do the same for `activecontrollercount`, `bytesinpersec`, `jvm_memory_used_bytes`. Reply with 2–3 examples if they differ from the `prometheus.<metric>.value` pattern.

Assuming standard Elastic Prometheus naming (`prometheus.<full_metric_name>.value` + `prometheus.labels.*`), here is the mapping + setup.

### General Setup in Kibana

1. **Create new Dashboard** → Name: "Kafka Overview (converted from Grafana)"
2. **Add Controls** (top bar) to mimic Grafana variables:
   - Control type: Options list (dropdown)
   - Field: `prometheus.labels.job` (or `labels.job` if not prefixed)
   - Name: Job
   - Multi-select: off (or on if you want)
   - Similarly add:
     - `prometheus.labels.component` (for broker/controller/connect filtering)
     - `prometheus.labels.instance` (broker host/instance selector)
3. Use **Lens** for most panels (preferred in 8.x), **Metric** for single big numbers, **TSVB** only if Lens percentile/aggregation is not enough.
4. Time range: Last 15 minutes or 1 hour (match Grafana default)
5. All filters use KQL: e.g. `prometheus.labels.component : "kraftcontroller" AND prometheus.labels.job : "$yourjob"`

### Panel-by-Panel Conversion (KRaft + Cluster Health + Connect)

**Row: KRaft**

1. **Metadata Error Count** (Stat / big number)
   - Type: **Metric** visualization (or Lens → Metric)
   - Aggregation: Last value
   - Field: `prometheus.kafka_controller_kafkacontroller_metadataerrorcount.value`
   - Filter (KQL): `prometheus.labels.component : "kraftcontroller" AND prometheus.labels.job : *` (or your job variable)
   - Color thresholds: green 0, orange >0, red >5 (add via Lens thresholds)
   - Title: Metadata Error Count

2. **Metadata Last Applied Record Offset** (Time series / line chart per instance)
   - Type: **Lens** → Lines
   - Y-axis: Last value of `prometheus.kafka_controller_kafkacontroller_lastappliedrecordoffset.value`
   - Breakdown by: Terms → `prometheus.labels.instance` (or `host.hostname` if used)
   - Filter: `prometheus.labels.component : "kraftcontroller"`
   - Legend: bottom, show instance names
   - Title: Metadata Last Applied Record Offset

3. **Metadata Last Committed Record Offset** (similar time series)
   - Same as above, but field: `prometheus.kafka_controller_kafkacontroller_lastcommittedrecordoffset.value`

4. **Metadata Last Applied - Committed Record Offset Lag** (time series)
   - Type: Lens → Lines
   - Use Formula: `last_value(prometheus.kafka_controller_kafkacontroller_lastappliedrecordoffset.value) - last_value(prometheus.kafka_controller_kafkacontroller_lastcommittedrecordoffset.value)`
   - Breakdown by: `prometheus.labels.instance`
   - Filter: same as above
   - Good if lag stays near 0

5. **Fenced Broker Count** (Stat)
   - Type: Metric
   - Last value of `prometheus.kafka_controller_kafkacontroller_fencedbrokercount.value`
   - Filter: `prometheus.labels.component : "kraftcontroller"`

**Row: Kafka Cluster**

6–13. Health stats (all big single-value panels → use **Metric** viz)

- **Active Controllers**
  - Field: `prometheus.kafka_controller_kafkacontroller_activecontrollercount.value`
  - Expected: usually 1 (or 3/5 in ensemble)
  - Filter: `prometheus.labels.component : "kraftcontroller"`

- **Brokers Online** (unique count)
  - Lens → Metric
  - Unique count of `prometheus.labels.instance` (or `host.hostname`)
  - Filter: `prometheus.labels.component : "kafkabroker"`

- **Online Partitions**
  - Sum or Last value of `prometheus.kafka_server_replicamanager_partitioncount.value`

- **Under Replicated Partitions**
  - Last value `prometheus.kafka_server_replicamanager_underreplicatedpartitions.value`
  - Threshold: >0 → yellow/red

- **Under Min ISR Partitions**
  - `prometheus.kafka_server_replicamanager_undermin ISRpartitions.value`

- **Offline Partitions**
  - `prometheus.kafka_server_replicamanager_offlinepartitionscount.value`

- **Preferred Replica Imbalance Count**
  - `prometheus.kafka_controller_kafkacontroller_preferredreplicaimbalancecount.value`

- **Unclean Leader Elections Rate**
  - Formula: `counter_rate(prometheus.kafka_controller_controllerstats_uncleanleaderelectionspersec.value)`
  - Or last value if gauge

**Schema Registry section** (if present in your JSON – usually 3–4 panels)

- Schema Registry Instances Online → Unique count on instance where component/schema-registry related label exists
- Schemas registered/deleted → counter_rate on schema registry create/delete metrics
- Compatibility level changes, etc.

**Kafka Connect section** (repeated per cluster or single)

- Worker instances running/failed
  - Lens Metric: Last value on `kafka_connect_connect_worker_total_task_count` etc.
  - Filter: `prometheus.labels.component : "connect"` or specific connect cluster label

### Quick Implementation Steps

1. Create dashboard
2. Add controls for job/component/instance
3. Add first panel (Metadata Error Count) using Lens/Metric as above
4. Duplicate panels and change field/filter/title for others (fastest way)
5. For time series: switch to Lens → Lines, add breakdown by instance
6. Add thresholds/colors where errors/high values should stand out (red/yellow)

Once you confirm the **exact field names** (e.g. paste 2 examples from Discover like `prometheus.kafka_...value`), I can refine the formulas/filters precisely — or help with the more detailed `cp-kafka-cluster.json` dashboard next.

Which panels look most important to get perfect first (KRaft health? Cluster stats? Connect?)? Or paste the field examples now.
