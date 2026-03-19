The file at the provided link is a partial or mixed NDJSON export from Kibana, containing:

- One initial object that appears to be a saved **data view / index pattern** configuration (with `fieldAttrs` and a very long `fieldFormatMap` listing formats for many unrelated metrics like Aerospike, Ceph, Couchbase, etc. – this looks like it was copied from a broad or sample Prometheus data view, not specific to your Kafka one).
- Then multiple **visualization** objects (type: "visualization"), mostly **TSVB** (timeseries) panels and some **Lens** panels.
- No actual **dashboard** object (type: "dashboard" with `panelsJSON` array referencing panel IDs).

This explains why earlier import attempts failed – the file is **not a complete dashboard export**. Kibana dashboard NDJSON typically starts with a dashboard object, followed by referenced visualizations, controls, etc.

### What this file actually contains (key panels extracted)
The file has several working **visualization** objects you can import individually (via **Stack Management → Saved Objects → Import**). Here are the most relevant ones for Kafka monitoring (based on field names like `prometheus.kafka_server_*`, `prometheus.labels.*` – your data view seems to use **prometheus.** prefixed metric names):

- **Zookeeper Connections Per Second** (TSVB timeseries)
  - Uses `positive_rate(prometheus.kafka_server_sessionexpirelistener_zookeepersyncconnectspersec.value)`
  - Split by `host.hostname` with hardcoded broker hosts (LOPVBROKAPPS01 etc.)
  - Filter includes `prometheus.labels.topic : *`

- **Zookeeper Disconnects Per Second** (similar TSVB)

- **Zookeeper Expired Connections Per Second** (partial in the extract)

- **Consumer Group Counts** (various states like Preparing Rebalance, Dead, Completing Rebalance, Empty) – Lens metric with sum aggregations on fields like:
  - `prometheus.kafka_coordinator_group_groupmetadatamanager_numgroupspreparingrebalance.value`
  - `prometheus.kafka_coordinator_group_groupmetadatamanager_numgroupsdead.value`
  - etc.

- **Connection Count per Listener** (Lens XY / line chart)
  - Sum of `prometheus.kafka_server_socketservermetrics_connection_count.value`
  - Breakdown (split) by `prometheus.labels.listener` (descending order)
  - X-axis: time histogram on `@timestamp`

- **Connection Count per Broker** (Lens XY, similar but likely split by broker/instance)

These are already converted/adapted visualizations (some titles say "(converted)"), probably from the original Grafana dashboard.

### Recommended next steps to get a usable Kafka dashboard

1. **Import the existing visualizations first**  
   - Download the full raw file from the **raw** URL:  
     https://raw.githubusercontent.com/nwlterry/kafka_dashboard_grafana_query/main/kafka-dashboard-grafana/kibana_kafka_dashboard_sample.ndjson  
   - Open it in a text editor and **split it into separate files** if needed (one JSON object per file), or try importing the whole thing and see which objects succeed (Kibana will skip invalid ones).  
   - After import, go to **Analytics → Visualize** and look for the imported ones (titles like "Zookeeper Connections Per Second", "Connection Count per Listener (converted)", etc.).

2. **Create the actual dashboard manually**  
   - Go to **Analytics → Dashboard → Create new**  
   - Name it e.g. "Kafka Monitoring (from Grafana)"  
   - Add the imported visualizations above as panels  
   - Add new Lens visualizations for missing parts (Healthcheck, Throughput, Latencies, etc.) using the confirmed field naming pattern:  
     - Metric values: `prometheus.kafka_server_....value` (e.g. `prometheus.kafka_server_brokertopicmetrics_bytesinpersec.value`)  
     - Labels: `prometheus.labels.instance`, `prometheus.labels.topic`, `prometheus.labels.listener`, `prometheus.labels.request`, etc.  
     - Hostname: `host.hostname` (seems used for broker filtering)

   Quick examples for key panels (Lens):

   - **Bytes In per Sec (total)**  
     - Lens → Area or Line chart  
     - Y-axis: Formula → `counter_rate(prometheus.kafka_server_brokertopicmetrics_bytesinpersec.value)`  
     - Breakdown by: `prometheus.labels.topic` (or none for total)  
     - Filter: `prometheus.labels.topic : *`

   - **Leader Count**  
     - Lens → Metric (big number)  
     - Aggregation: Last value or Sum of `prometheus.kafka_server_replicamanager_leadercount.value`

   - **Active Controllers**  
     - Lens Metric → Last value of `prometheus.kafka_controller_kafkacontroller_activecontrollercount.value`

   - **Request Latency p99 (Produce)**  
     - Lens → Line chart  
     - Y-axis: Percentile 99 of `prometheus.kafka_network_requestmetrics_requestqueuetimems.value` (or relevant quantile field)  
     - Filter: `prometheus.labels.request : "Produce"`

   - **Under Replicated Partitions**  
     - Lens Metric → Sum or Last value of `prometheus.kafka_server_replicamanager_underreplicatedpartitions.value`

3. **Add dashboard controls** (replaces Grafana variables)  
   - Add control for `prometheus.labels.job` (if you use job filtering)  
   - Add control for `host.hostname` or `prometheus.labels.instance` (multi-select brokers)

4. **Once you have a partial dashboard working**  
   - Export it (Stack Management → Saved Objects → select your dashboard → Export → Include related objects)  
   - This will give a clean, complete NDJSON you can version-control or share.

Your data view naming pattern (`prometheus.<metric>.value` and `prometheus.labels.<label>`) is now clear from the sample – this matches common Elastic Prometheus scraping setups. Use that for any new Lens panels.

If you run into specific import errors or need exact Lens config for a particular panel (e.g. ISR changes, JVM memory, request rates with quantiles), paste the panel title or describe which one from the original Grafana repo, and I can give precise instructions.

Let me know what you see after trying to import the raw file or which panel to detail next!
