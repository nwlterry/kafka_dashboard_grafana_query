The Formula last_value(prometheus.jvm_memory_bytes_used.value, kql='prometheus.labels.area: "heap"' ) / last_value ( prometheus.jvm_memory_bytes_max.value, kql='prometheus.labels.area: "heap"' )) cannot be parsed

last_value(`prometheus.jvm_memory_bytes_used.value`, kql='prometheus.labels.area: "heap"') 
/ 
last_value(`prometheus.jvm_memory_bytes_max.value`, kql='prometheus.labels.area: "heap"') ?? 0

last_value(`prometheus.jvm_memory_bytes_used.value`, kql='prometheus.labels.area: "heap"') 
/ last_value(`prometheus.jvm_memory_bytes_max.value`, kql='prometheus.labels.area: "heap"') ?? 0

last_value(`prometheus.jvm_memory_bytes_used.value`, 
  kql='prometheus.labels.area: "heap" AND prometheus.labels.job: "$job" AND prometheus.labels.instance: "$instance"') 
/ 
last_value(`prometheus.jvm_memory_bytes_max.value`, 
  kql='prometheus.labels.area: "heap" AND prometheus.labels.job: "$job" AND prometheus.labels.instance: "$instance"') ?? 0
