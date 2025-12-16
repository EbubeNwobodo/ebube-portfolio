---
title: "Processing 100TB of Daily Logs: A Fault-Tolerant ELK Pipeline"
date: 2025-04-10
summary: "Designing a high-volume logging architecture using Elasticsearch, Fluentd, and Kafka to reduce incident detection time by 40%."
tags: ["ELK Stack", "Kubernetes", "Python", "Observability", "Big Data"]
weight: 2
cover:
  image: "images/log-file_11068792.png" # Optional placeholder
  alt: "Log Aggregation Architecture"
  caption: "Flow of logs from Microservices to Kibana Dashboards"
---

## The Problem
As the microservices architecture scaled, the volume of generated logs exploded to over **100GB per day**. The existing logging solution was struggling to keep up, resulting in:
* **Blind Spots:** Logs were frequently dropped during peak traffic.
* **Slow RCA:** Incident detection time was lagging, increasing the Mean Time to Acknowledgment (MTTA).
* **High Costs:** Storing unoptimized, raw logs was becoming prohibitively expensive.

## The Solution
I engineered a fault-tolerant logging pipeline designed to decouple log ingestion from storage. By introducing a buffering layer and optimizing the indexing strategy, we achieved high availability and faster queries.

## Business Impact
1. **Reduced Incident Detection Time by 40%:** The centralized dashboard allowed the NOC team to correlate errors across services instantly.
2. **Zero Data Loss:** The buffering strategy ensured 100% log delivery even during load spikes.
3. **Cost Efficiency:** By filtering "noise" logs at the Fluentd level before they reached storage, we reduced storage costs significantly.

### Key Architecture Decisions
1.  **Fluentd as the Shipper:** Deployed as a DaemonSet on Kubernetes nodes to collect logs directly from container `stdout`.
2.  **Kafka Buffer:** Implemented Apache Kafka as an intermediate buffer to handle backpressure during traffic spikes.
3.  **Elasticsearch Optimization:** Implemented "Hot-Warm-Cold" architecture to keep recent logs on fast NVMe storage for quick querying, while archiving older logs to cheaper storage.

## Technical Implementation
The critical component was configuring **Fluentd** to handle network jitters without losing data. I utilized local file buffering to ensure durability even if the downstream aggregator was temporarily unreachable.

### Code Snippet: Fluentd Buffer Configuration
*This configuration ensures that if Elasticsearch applies backpressure, Fluentd buffers logs locally instead of dropping them.*

```ruby
<match app.**>
  @type elasticsearch
  host elasticsearch.logging.svc.cluster.local
  port 9200
  logstash_format true
  
  # Buffer Configuration for Reliability
  <buffer>
    @type file
    path /var/log/fluentd-buffers/kubernetes.system.buffer
    flush_mode interval
    retry_type exponential_backoff
    flush_interval 5s
    retry_forever false
    retry_max_interval 30
    chunk_limit_size 2M
    queue_limit_length 8
    overflow_action block
  </buffer>
</match>