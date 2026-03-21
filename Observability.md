<!-- Generated: 2026-03-12 | Prometheus 3.2 | Elasticsearch 9.3 | Kibana 9.3 -->

# Observability Stack Interview Preparation — Tech Lead / Architect Level
> Prometheus 3.2 · Elasticsearch 9.3 · Kibana 9.3 | Generated: 2026-03-12 | Career OS

---

## Stack Overview

These three tools form the dominant open-source observability stack at architect level:
- **Prometheus** — metrics collection, storage, and alerting
- **Elasticsearch** — distributed search and log/event storage
- **Kibana** — visualisation, dashboards, and log exploration (Elasticsearch frontend)

They are frequently deployed together (Prometheus for metrics, ELK for logs) alongside Grafana for Prometheus visualisation. Interviewers probe both individual depth and how the tools interoperate.

---

# SECTION 1 — Prometheus 3.2

---
**L1–L5 Topics identified:**
- What is Prometheus / Pull model vs Push model
- Data Model — metrics types (Counter / Gauge / Histogram / Summary)
- PromQL — query language fundamentals
- Alerting — Alertmanager, routing, inhibition, silencing
- Service Discovery — Kubernetes SD, static configs
- Storage — TSDB internals, remote write/read
- Federation & High Availability
- Exporters & Instrumentation

---

### Foundations
#### What is Prometheus / Pull vs Push Model

* **Question:** What is Prometheus, how does its pull model work, and what are the architectural trade-offs vs a push model?
* **Answer:**
  - Prometheus is an open-source systems monitoring and alerting toolkit. It collects metrics by **scraping** HTTP endpoints — targets expose metrics at `/metrics`, and Prometheus periodically pulls from them. This is the pull model.
  - **Pull model advantages:** (1) Prometheus controls the scrape interval — it knows immediately if a target is down (scrape fails). (2) No instrumentation agent required on targets — any process exposing an HTTP endpoint can be scraped. (3) Easier to reason about backpressure — if a target is overwhelmed, Prometheus keeps scraping at its own rate. (4) Simple debugging — you can `curl` the `/metrics` endpoint manually.
  - **Pull model trade-offs:** (1) Requires Prometheus to reach the target — problematic in push-firewall topologies (targets behind NAT). (2) Short-lived jobs (batch jobs) may finish before the next scrape — use **Pushgateway** as an intermediary for ephemeral jobs. (3) Scrape interval is the minimum resolution — metrics that spike and recover between scrapes are missed.
  - **Push model** (used by InfluxDB, StatsD, OpenTelemetry Collector): Metrics are sent to the aggregator by the target. Better for short-lived processes; harder to detect target failure.
  - **Prometheus 3.x (2024+):** Native OpenTelemetry ingestion — OTLP push directly to Prometheus without a collector. UTF-8 metric names. Significant PromQL performance improvements.
* **Example:**
```yaml
# prometheus.yml — scrape config
global:
  scrape_interval: 15s     # pull every 15s

scrape_configs:
  - job_name: 'api-service'
    static_configs:
      - targets: ['api:8080']  # Prometheus pulls /metrics from this

  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:    # dynamic discovery
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"
```
```go
// Target instrumentation — expose /metrics
import "github.com/prometheus/client_golang/prometheus/promhttp"
http.Handle("/metrics", promhttp.Handler())
```
* **Technical Terms to Include:** pull model, scrape, `/metrics` endpoint, Pushgateway, scrape interval, service discovery, OTLP, OpenTelemetry, target, job, instance
* **Gotcha:** The Pushgateway is **not** a general-purpose metrics aggregation layer. Metrics pushed to the Pushgateway persist until explicitly deleted — a batch job that pushes on success and then fails on the next run will leave stale "success" metrics. Always delete Pushgateway metrics after job completion, or use `--push.disable-consistency-check` carefully.
* **Follow-Up:** "How does Prometheus handle target failure detection?" → If a scrape fails (connection refused, timeout, non-200 response), Prometheus records the metric `up=0` for that target. Alerting on `up == 0` for more than a threshold duration (e.g., 5 minutes) is the standard pattern for detecting dead services. This is why the pull model is considered superior for availability monitoring — failure is observable as a scrape failure. → They're testing alerting-on-down awareness.
* **Conclusion:** Prometheus's pull model makes target health an implicit first-class metric — every scrape failure is observable — while its simplicity (any HTTP endpoint is a valid target) makes it the de facto standard for cloud-native metrics collection.

---

#### Metric Types — Counter, Gauge, Histogram, Summary

* **Question:** What are the four Prometheus metric types, when do you use each, and what are the gotchas with Histograms?
* **Answer:**
  - **Counter:** Monotonically increasing value — resets to 0 on restart. Represents cumulative events: requests served, errors, bytes sent. Never decreases. Use `rate()` or `increase()` in PromQL to get the per-second rate. Example: `http_requests_total`.
  - **Gauge:** A value that goes up and down. Represents current state: memory usage, active connections, queue depth, temperature. Example: `process_resident_memory_bytes`, `queue_depth`.
  - **Histogram:** Samples observations and counts them in configurable buckets. Calculates quantiles server-side (at query time) from bucket data. Shipped buckets are cumulative. Example: `http_request_duration_seconds_bucket`. Use `histogram_quantile(0.99, rate(...)[5m])` for p99 latency.
  - **Summary:** Pre-computes quantiles on the client side (at instrumentation time). Buckets are fixed at instrumentation — cannot aggregate across instances at query time. Use only for single-instance quantile requirements where aggregation is not needed.
  - **Histogram vs Summary:** Histograms are aggregatable across instances — you can compute p99 across a fleet. Summaries are not — each instance's quantiles are independent and cannot be meaningfully averaged. Prefer Histogram for server-side latency in distributed systems.
  - **Prometheus 3.x Native Histograms:** New exponential bucket schema — higher precision, lower cardinality, automatically computed at query time. Replaces manual bucket configuration. Opt in via `ENABLE_FEATURE=native-histograms`.
* **Example:**
```go
// Counter
requestsTotal := prometheus.NewCounterVec(
    prometheus.CounterOpts{Name: "http_requests_total", Help: "Total HTTP requests"},
    []string{"method", "status"},
)
requestsTotal.WithLabelValues("GET", "200").Inc()

// Gauge
activeConnections := prometheus.NewGauge(prometheus.GaugeOpts{
    Name: "active_connections", Help: "Current active connections",
})
activeConnections.Set(42)

// Histogram — buckets for latency
requestDuration := prometheus.NewHistogramVec(
    prometheus.HistogramOpts{
        Name:    "http_request_duration_seconds",
        Buckets: prometheus.DefBuckets, // .005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10
    },
    []string{"handler"},
)
// PromQL: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```
* **Technical Terms to Include:** Counter, Gauge, Histogram, Summary, `rate()`, `increase()`, `histogram_quantile()`, bucket, cumulative histogram, client-side vs server-side quantile, cardinality, native histograms
* **Gotcha:** **Cardinality explosion** — a label with high cardinality (user IDs, URLs with IDs, request GUIDs) multiplied across all metric series can exhaust Prometheus memory. A metric with 10 labels each having 100 values = 10^10 potential series. Rule: labels should have bounded, low cardinality values. Use recording rules to aggregate before storing.
* **Follow-Up:** "What is the difference between `rate()` and `irate()` in PromQL?" → `rate()` computes the per-second average rate over the entire range window — smooths out spikes. `irate()` uses only the last two data points — extremely responsive to spikes but noisy. Use `rate()` for dashboards and alerting (stable signal). Use `irate()` only when you need to detect momentary spikes. → They're testing PromQL precision.
* **Conclusion:** Choosing the right metric type is an API design decision — Counters for cumulative events (always use `rate()`), Gauges for current state, Histograms for latency/size distributions with cross-instance aggregation, and Summary only for single-instance pre-computed quantiles where aggregation is never needed.

---

#### PromQL — Query Language

* **Question:** What are the key PromQL concepts — instant vectors, range vectors, selectors, and aggregation operators — and how do you write a production SLO query?
* **Answer:**
  - **Instant vector:** A set of time series at a single point in time. `http_requests_total` returns the current value of all series with that metric name.
  - **Range vector:** A set of time series over a time range. `http_requests_total[5m]` returns the last 5 minutes of samples for each series. Required by `rate()`, `irate()`, `increase()`.
  - **Selectors:** Filter series by label matchers. `{job="api", status=~"5.."}` matches series where job is "api" and status matches the regex `5..` (5xx errors). `!=`, `!~` for negative/negative-regex matching.
  - **Aggregation operators:** `sum()`, `avg()`, `max()`, `min()`, `count()`, `topk()`, `bottomk()`. Use `by` or `without` to control which labels survive aggregation. `sum by (status) (rate(...))` — sum across all instances, keep the `status` label.
  - **Functions:** `rate()` (counter rate), `increase()` (counter increase over range), `delta()` (gauge change over range), `absent()` (fires when series missing — alerting on target disappearance), `predict_linear()` (disk full prediction).
* **Example:**
```promql
# Error rate — 5xx errors as % of total requests
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))

# p99 latency across all API instances
histogram_quantile(0.99,
  sum by (le) (rate(http_request_duration_seconds_bucket{job="api"}[5m]))
)

# SLO: Availability over 30 days (ratio of successful requests)
1 - (
  sum(rate(http_requests_total{status=~"5.."}[30d]))
  /
  sum(rate(http_requests_total[30d]))
)

# Alert: disk will fill in 4 hours at current rate
predict_linear(node_filesystem_avail_bytes[1h], 4 * 3600) < 0

# Alert: service is down (no time series — target disappeared)
absent(up{job="api"})
```
* **Technical Terms to Include:** instant vector, range vector, label matcher, regex selector, aggregation operator, `rate()`, `histogram_quantile()`, `absent()`, `predict_linear()`, recording rule, subquery, `offset`
* **Gotcha:** `rate()` requires a range at least 2x the scrape interval to return meaningful results. With a 15s scrape interval, `rate(metric[15s])` may return no data (only one sample in the window). Use at least `[30s]` — typically `[1m]` or `[5m]` for dashboards. Using too short a range causes gaps in graphs and false absence alerts.
* **Follow-Up:** "What is a recording rule and why does it matter at scale?" → A recording rule pre-computes an expensive PromQL query and stores the result as a new metric — written back into Prometheus's TSDB. Dashboard queries then read the pre-computed metric instead of recomputing on every load. Critical at scale: a dashboard with 20 panels each computing `sum(rate(...))` over millions of series is extremely expensive. Recording rules run once per evaluation interval (e.g., every minute) and amortise the cost. → They're testing production Prometheus performance awareness.
* **Conclusion:** PromQL is Prometheus's primary interface for both dashboards and alerting — mastering range vectors, aggregation operators, and `histogram_quantile` is the difference between useful SLO dashboards and ones that lie silently due to incorrect window sizes or missing `by` clauses.

---

#### Alerting — Alertmanager, Routing, Silencing

* **Question:** How does Prometheus alerting work end-to-end — from rule evaluation to notification — and how do you design a routing tree?
* **Answer:**
  - **Alert lifecycle:** Prometheus evaluates alerting rules at the `evaluation_interval` (default 1m). When an expression evaluates to true, the alert enters **pending** state. After the `for` duration (e.g., `for: 5m`) — the alert fires (becomes **active**). Prometheus sends firing alerts to Alertmanager.
  - **Alertmanager responsibilities:** (1) **Grouping** — combine related alerts into a single notification (e.g., all alerts from one cluster). (2) **Routing** — send to the right receiver (PagerDuty for critical, Slack for warning). (3) **Inhibition** — suppress lower-priority alerts when a higher-priority alert fires (e.g., suppress individual service alerts when the whole datacenter is down). (4) **Silencing** — mute alerts during maintenance windows.
  - **Routing tree:** A tree of `route` blocks. Each alert matches the most specific route based on label matchers. The root route is the default. Child routes refine matching. Each route specifies a receiver (Slack, PagerDuty, email, webhook).
  - **`for` duration:** The grace period before an alert fires. Prevents alert storms on transient spikes. Too short: noisy. Too long: delayed incident response. Typical: 1–5 minutes for infrastructure, 30s for latency SLOs.
* **Example:**
```yaml
# Prometheus alerting rule
groups:
  - name: api-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          / sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "Error rate {{ $value | humanizePercentage }}"
          runbook: "https://runbooks.internal/high-error-rate"

# Alertmanager routing
route:
  receiver: default-slack
  group_by: ['alertname', 'cluster']
  group_wait: 30s       # wait before sending first notification in group
  group_interval: 5m    # wait before re-notifying same group
  repeat_interval: 4h   # repeat notification if still firing

  routes:
    - match:
        severity: critical
      receiver: pagerduty
      continue: false   # stop matching — critical goes only to PagerDuty

inhibit_rules:
  - source_match:
      alertname: ClusterDown
    target_match_re:
      alertname: ".+"   # suppress all other alerts when cluster is down
    equal: ['cluster']  # only if same cluster label
```
* **Technical Terms to Include:** alerting rule, `for` duration, pending state, firing state, Alertmanager, receiver, routing tree, grouping, inhibition, silencing, `group_by`, `group_wait`, `group_interval`, runbook URL, label-based routing
* **Gotcha:** `continue: false` in a route stops routing after the first match. Without it (default: `false`), a critical alert goes to PagerDuty and stops — it doesn't also go to Slack. If you want both, set `continue: true` on the PagerDuty route. Forgetting this means critical alerts are silently not reaching secondary channels.
* **Follow-Up:** "How do you prevent alert storms during a deployment?" → (1) Pre-create a silence in Alertmanager for the deployment window — suppresses all alerts matching a label (e.g., `env=production`) for the duration. (2) Use the `inhibit_rules` to suppress downstream alerts when an upstream service is intentionally down. (3) Use `for: 10m` to give the deployment time to stabilise before alerts fire. → They're testing operational alerting maturity.
* **Conclusion:** Alertmanager's routing tree, inhibition rules, and silence management are the operational layer that determines whether on-call engineers receive actionable signals or an inbox of noise — designing these correctly is as important as writing the PromQL expressions.

---

# SECTION 2 — Elasticsearch 9.3

---
**L1–L5 Topics identified:**
- What is Elasticsearch / Inverted Index / Distributed search
- Data Model — Index / Shard / Replica / Document
- Query DSL — match / term / bool / aggregations
- Mapping — dynamic vs explicit, field types
- Cluster Architecture — master / data / ingest / coordinating nodes
- Performance — shard sizing, segment merging, index lifecycle
- ECS (Elastic Common Schema) & Data Streams

---

### Core Architecture
#### What is Elasticsearch / Inverted Index

* **Question:** What is Elasticsearch, how does an inverted index work, and why is it the foundation of full-text search performance?
* **Answer:**
  - Elasticsearch is a distributed, RESTful search and analytics engine built on Apache Lucene. It stores documents as JSON, indexes them for full-text search, and enables near-real-time search across billions of documents.
  - **Inverted index:** The inverse of a document-to-words mapping. Instead of "document → words it contains," an inverted index maps "word → documents that contain it." When you index a document, Elasticsearch tokenises the text (via an analyser), normalises the tokens (lowercase, stemming, stop-words), and records each token with its posting list (which document IDs contain it, and at what position).
  - **Why it's fast:** Searching for a term is a hash lookup into the inverted index — O(1) lookup for the posting list, then a union/intersection of posting lists for multi-term queries. Compare to a full-text scan of all documents — O(n). Elasticsearch makes full-text queries logarithmic in practice.
  - **Analyser pipeline:** `character filter → tokeniser → token filter`. Example: "Quick Brown Fox" → `["quick", "brown", "fox"]` (lowercase, whitespace tokenise). The same analyser runs at index time and query time — mismatching them is a common bug.
  - **Elasticsearch 9.x:** ES|QL (Elasticsearch Query Language) — a new SQL-like pipe-based query language for Elasticsearch. Semantic search via `semantic_text` field type with integrated vector embeddings. Significant improvements to sparse vector search.
* **Example:**
```json
// Index a document
PUT /logs/_doc/1
{
  "message": "Application started successfully",
  "level": "INFO",
  "@timestamp": "2026-03-12T10:00:00Z",
  "service": "api-service"
}

// Inverted index entry (conceptual) after analysis:
// "application" → [doc:1]
// "started"     → [doc:1]
// "successfully"→ [doc:1]

// Query — full text match
GET /logs/_search
{
  "query": {
    "match": {
      "message": "application started"  // tokenised, matched against inverted index
    }
  }
}

// ES|QL — Elasticsearch 9.x pipe-based query
POST /_query
{
  "query": "FROM logs | WHERE level == \"ERROR\" | STATS count = COUNT(*) BY service | SORT count DESC | LIMIT 10"
}
```
* **Technical Terms to Include:** inverted index, analyser, tokeniser, token filter, posting list, near-real-time (NRT), Lucene segment, refresh interval, `_source`, `_id`, ES|QL, semantic search
* **Gotcha:** Elasticsearch search is **near-real-time**, not real-time. Newly indexed documents are only searchable after a **refresh** (default: every 1 second). A document indexed at T=0 may not appear in search results until T=1s. For bulk indexing, increasing `refresh_interval` to `30s` or `-1` (manual refresh) dramatically improves throughput — but documents won't be searchable until refreshed.
* **Follow-Up:** "What is the difference between `match` and `term` queries?" → `match` applies the analyser to the search text before matching — use for full-text search on `text` fields. `term` does exact, case-sensitive matching without analysis — use for `keyword`, `boolean`, `numeric` fields. Using `match` on a `keyword` field or `term` on a `text` field produces incorrect or no results. → They're testing query type precision.
* **Conclusion:** Elasticsearch's inverted index makes full-text search a lookup rather than a scan — but exploiting it correctly requires understanding the analyser pipeline, the distinction between `text` and `keyword` field types, and the near-real-time refresh cycle that determines when indexed data becomes searchable.

---

#### Data Model — Index, Shard, Replica, Document

* **Question:** How does Elasticsearch's distributed data model work — shards, replicas, and the implications for cluster sizing?
* **Answer:**
  - **Index:** A logical namespace for a collection of documents — analogous to a database table. Backed by one or more shards.
  - **Shard:** The unit of distribution. An index is split into N primary shards (set at creation, immutable). Each shard is a full Lucene index. Shards are distributed across data nodes — enabling horizontal scaling.
  - **Replica:** A copy of a primary shard. Replicas provide: (1) High availability — if the primary shard's node fails, a replica is promoted. (2) Read throughput — search requests can be served by either the primary or replica. Replicas can be changed at runtime (unlike primary shard count).
  - **Document routing:** A document is routed to `shard = hash(document_id) % num_primary_shards`. Consistent — the same document always goes to the same shard. Custom routing is possible but must be consistent to avoid "lost" documents.
  - **Shard sizing:** Too few shards: can't scale out. Too many shards: Lucene overhead per shard is significant — each shard is a JVM heap consumer. Rule of thumb: 10–50GB per shard. Use Data Streams (automatic rollover) to manage shard count over time for time-series data (logs, metrics).
  - **Elasticsearch 9.x Stateless Architecture:** Compute and storage separation — data nodes can be stateless, pulling shard data from object storage (S3/GCS). Enables faster node recovery and elastic scaling.
* **Example:**
```json
// Create index with explicit shard count
PUT /application-logs
{
  "settings": {
    "number_of_shards": 3,     // primary shards — immutable after creation
    "number_of_replicas": 1    // 1 replica per primary — can change later
  }
}

// Data Streams — for time-series log data (auto-rollover)
PUT /_index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "data_stream": {},
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1
    },
    "lifecycle": {
      "name": "logs-policy"   // ILM policy — auto rollover, delete after 30d
    }
  }
}
```
* **Technical Terms to Include:** primary shard, replica shard, shard allocation, routing, Data Stream, ILM (Index Lifecycle Management), rollover, segment, Lucene index, oversharding, node roles
* **Gotcha:** Primary shard count is **immutable** after index creation. The only way to change it is to reindex — a slow, resource-intensive operation. Always plan shard count upfront based on expected data volume. Data Streams with ILM rollover policies sidestep this by creating new backing indices automatically as data grows.
* **Follow-Up:** "What is oversharding and how does it affect cluster performance?" → Each shard is a Lucene index with its own thread pool, file handles, and heap overhead. A cluster with thousands of small shards uses more heap per shard than the data warrants, degrades cluster state management (every shard state is stored in cluster state), and slows down queries (coordinating a query across 1000 shards takes more time than 10). Aim for 10-50GB per shard. Use the ILM rollover `max_shards_per_node` constraint. → They're testing cluster operations depth.
* **Conclusion:** Elasticsearch's shard model is the primary lever for horizontal scalability and high availability — but immutable primary shard count and the overhead of oversharding make pre-planning and Data Stream–based rollover the mandatory operational patterns for production log and metrics clusters.

---

#### Query DSL & Aggregations

* **Question:** Explain the Elasticsearch Query DSL structure — `bool` query composition, `filter` vs `query` context, and aggregation architecture.
* **Answer:**
  - **Query context vs Filter context:** In query context (`query`), Elasticsearch computes a relevance score (`_score`) — documents are ranked by how well they match. In filter context (`filter`), there is no score — documents either match or don't. Filters are cached — dramatically faster for structured data (status codes, dates, enums). Use filter for everything that doesn't need scoring; use query for full-text relevance.
  - **`bool` query:** Combines multiple clauses: `must` (AND, contributes to score), `should` (OR, contributes to score), `filter` (AND, no score, cached), `must_not` (NOT, no score). The `bool` query is the primary composition mechanism.
  - **Aggregations:** Two types: (1) **Bucket aggregations** — group documents into buckets (`terms`, `date_histogram`, `range`). (2) **Metric aggregations** — compute a value from a bucket (`avg`, `sum`, `max`, `percentiles`, `cardinality`). Aggregations nest — bucket aggregations can contain sub-aggregations.
  - **ES|QL (Elasticsearch 9.x):** `FROM logs | STATS avg(response_time) BY service` — pipe-based, SQL-like. More readable than Query DSL for analytics queries.
* **Example:**
```json
// bool query — filter context for structured data, query context for text
GET /logs/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term":  { "level": "ERROR" } },          // exact match, cached
        { "range": { "@timestamp": { "gte": "now-1h" } } }
      ],
      "must": [
        { "match": { "message": "connection refused" } }  // scored full-text
      ]
    }
  },
  "aggs": {
    "errors_per_service": {
      "terms": { "field": "service.keyword", "size": 10 },  // bucket by service
      "aggs": {
        "avg_response_time": {
          "avg": { "field": "response_time_ms" }   // metric per bucket
        },
        "p99_latency": {
          "percentiles": { "field": "response_time_ms", "percents": [99] }
        }
      }
    }
  }
}
```
* **Technical Terms to Include:** Query DSL, bool query, filter context, query context, relevance score, `_score`, bucket aggregation, metric aggregation, `terms` agg, `date_histogram`, `percentiles`, filter cache, ES|QL
* **Gotcha:** Running aggregations on `text` fields fails — Elasticsearch cannot aggregate on analysed text (the field is stored as inverted index tokens, not the original value). Always aggregate on `keyword` sub-fields: `service.keyword` not `service`. Set `"fielddata": true` on `text` fields only as a last resort — it loads the uninverted index into heap and can cause OOM.
* **Follow-Up:** "What is the `cardinality` aggregation and what are its accuracy trade-offs?" → `cardinality` computes approximate distinct count using HyperLogLog++ — memory-efficient but not exact. The `precision_threshold` parameter controls accuracy vs memory (max 40,000). For exact counts: `terms` aggregation with `size` matching distinct count — but this loads all values into memory. For counts with billions of documents, approximate is the only viable option. → They're testing aggregation accuracy vs performance awareness.
* **Conclusion:** Elasticsearch's Query DSL composability — `bool` query with mixed filter/query contexts — is its primary performance lever: filters are cached and score-free, enabling sub-millisecond structured lookups while preserving full-text relevance scoring only where it's needed.

---

# SECTION 3 — Kibana 9.3

---
**L1–L4 Topics identified:**
- Kibana's Role — Elasticsearch frontend vs standalone features
- Discover — log exploration
- Dashboards & Visualisations
- Alerts & Actions
- Kibana Lens & ES|QL integration

---

### Core Features
#### Kibana Architecture & Role

* **Question:** What is Kibana's role in the Elastic Stack, what are its primary use cases, and what distinguishes it from Grafana?
* **Answer:**
  - Kibana is the visualisation and management UI for the Elastic Stack. It connects to Elasticsearch as its data source and provides: (1) **Discover** — ad-hoc log exploration and search. (2) **Dashboards** — composable visualisation panels. (3) **Lens** — drag-and-drop chart builder with automatic field type detection. (4) **Alerts** — rule-based alerting on Elasticsearch data (index threshold, ES|QL alerts). (5) **APM** — application performance monitoring (traces, spans). (6) **Fleet** — elastic agent management. (7) **Canvas** — custom presentation-style dashboards.
  - **Kibana vs Grafana:** Grafana is a multi-datasource visualisation tool — connects to Prometheus, Elasticsearch, Loki, InfluxDB, databases. Grafana for metrics dashboards (Prometheus), Kibana for log exploration and Elastic Stack management. In a mixed stack (Prometheus for metrics, ELK for logs), both coexist — each does what it's best at.
  - **Index Pattern / Data View:** The Kibana abstraction over an Elasticsearch index or Data Stream — defines which index/pattern to query and the time field to use for time-based filtering. `logs-*` matches all indices starting with `logs-`.
  - **Kibana 9.3 — Scheduled Reports (GA):** Generate PDF/CSV reports on a schedule and email them. `Agent Builder` (AI-powered dashboard and query assistant) GA in Elasticsearch solution environments.
* **Example:**
```
// KQL (Kibana Query Language) — Discover search bar
level: ERROR AND service: "api-service" AND @timestamp > now-1h

// ES|QL in Kibana Discover (9.x)
FROM logs-*
| WHERE level == "ERROR"
| STATS error_count = COUNT(*), avg_response = AVG(response_time_ms) BY service
| SORT error_count DESC

// Kibana Dashboard — Lens visualisation config (conceptual)
// Metric: Count of documents where level = ERROR
// Breakdown: By service.keyword (top 10)
// Time: Last 24 hours
// Chart: Bar chart
```
* **Technical Terms to Include:** Index Pattern, Data View, Discover, Lens, KQL, ES|QL, Kibana Alerts, APM, Fleet, Canvas, Elastic Agent, Data View time field, cross-cluster search
* **Gotcha:** KQL (Kibana Query Language) and Elasticsearch Query DSL are not the same. KQL is a simplified query syntax for the search bar — it doesn't support all DSL features (aggregations, scoring, nested queries). For complex queries, switch to ES|QL or the Elasticsearch API directly. Teams often hit KQL limitations and assume Elasticsearch can't do it — it can, just not via the search bar.
* **Follow-Up:** "How do you build an SLO dashboard in Kibana for error rates?" → Create a Lens visualisation: metric = "Count of documents" filtered to `level: ERROR` / "Count of all documents" → error rate. Add a threshold line at 5%. Combine with a p99 latency panel (average aggregation on `response_time_ms` with percentile metric). Set up a Kibana Rule (index threshold) to alert when error rate exceeds 5% for 5 minutes, notifying via Slack or PagerDuty connector. → They're testing production observability dashboard design.
* **Conclusion:** Kibana is the operational interface for the Elastic Stack — not just a visualisation tool but a full management and alerting platform; its strength is native, deep integration with Elasticsearch's query capabilities (ES|QL, aggregations, APM traces) that Grafana achieves only through a connector.

---

## Observability Stack Cross-Reference

| Concern | Tool | Key Concept |
|---|---|---|
| Service metrics (latency, errors, throughput) | Prometheus | Counters + Histograms + `rate()` |
| Infrastructure metrics (CPU, memory, disk) | Prometheus + node_exporter | Gauges + `predict_linear()` |
| Log storage & search | Elasticsearch | Inverted index + `bool` query |
| Log exploration & dashboards | Kibana | Discover + Lens + KQL |
| Metrics visualisation | Grafana (Prometheus datasource) | PromQL panels |
| Alerting on metrics | Prometheus Alertmanager | Routing tree + inhibition |
| Alerting on logs | Kibana Alerts | Index threshold + ES|QL rules |
| SLO tracking | Prometheus recording rules | Pre-computed ratio metrics |
| Distributed tracing | Elastic APM / Jaeger | Trace ID propagation |

---

*— End of Observability_Interview_QA.md —*
*Coverage: Prometheus (L1–L4, 4 QA units) | Elasticsearch (L1–L4, 3 QA units) | Kibana (L1–L3, 1 QA unit)*
*Calibrated for: Tech Lead / Architect level*
