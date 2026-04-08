# Prometheus Certified Associate (PCA) — Study Guide

**Exam:** Prometheus Certified Associate (PCA) by Linux Foundation / CNCF
**Cost:** $250 (one retake included)
**Format:** 90 minutes, online proctored, multiple-choice
**Validity:** 2 years
**Depth:** Deep technical — PromQL fluency, configuration, alerting, instrumentation

---

## Exam Domains

| Domain | Weight | Focus |
|---|---|---|
| **Observability Concepts** | 18% | Metrics vs logs vs traces, push vs pull, service discovery, SLO/SLA/SLI |
| **Prometheus Fundamentals** | 20% | Architecture, configuration, data model, labels, scraping, limitations |
| **PromQL** | 28% | Selectors, rate/increase, aggregation, histogram_quantile, binary ops |
| **Instrumentation & Exporters** | 16% | Client libraries, metric types, naming, node_exporter, blackbox_exporter |
| **Alerting & Dashboarding** | 18% | Alerting rules, Alertmanager, grouping/inhibition/silencing, Grafana basics |

**PromQL is the largest domain at 28%.** Hands-on practice is essential.

---

## Phase 1: Observability Concepts (18%)

### The Three Pillars

| Signal | What | Example | Prometheus? |
|---|---|---|---|
| **Metrics** | Numeric measurements over time | CPU usage, request count, latency p99 | Yes — this is what Prometheus does |
| **Logs** | Discrete text events | "User 123 logged in at 10:03" | No — use Loki, ELK, etc. |
| **Traces** | Request flow across services | Request → API → DB → Cache → Response | No — use Jaeger, Tempo, etc. |

Prometheus is a **metrics** system. It does not handle logs or traces.

### Push vs Pull

**Pull model (Prometheus):** The server scrapes targets at regular intervals by fetching `/metrics` endpoints. Targets expose metrics; Prometheus decides when to collect.

**Push model (e.g., StatsD, OTLP):** Applications push metrics to a central collector. The application decides when to send.

**Prometheus uses pull.** The exception: the **Pushgateway** allows short-lived jobs (batch jobs, cron jobs) to push metrics because they may exit before Prometheus scrapes them.

### Service Discovery

Prometheus can automatically discover targets from:
- **Kubernetes** (`kubernetes_sd_configs`) — pods, services, endpoints, nodes
- **Consul** (`consul_sd_configs`)
- **AWS EC2** (`ec2_sd_configs`)
- **DNS** (`dns_sd_configs`)
- **File-based** (`file_sd_configs`) — JSON/YAML files that can be updated externally
- **Static** (`static_configs`) — hardcoded targets

### SLO / SLA / SLI

| Term | Meaning | Example |
|---|---|---|
| **SLI** (Service Level Indicator) | The metric being measured | p99 latency, error rate, availability |
| **SLO** (Service Level Objective) | The target for the SLI | "p99 latency < 200ms", "99.9% availability" |
| **SLA** (Service Level Agreement) | Business contract with consequences | "If availability drops below 99.9%, customer gets credits" |

**SLI → SLO → SLA** (measurement → target → contract)

---

## Phase 2: Prometheus Fundamentals (20%)

### Architecture

```
┌─────────────┐     scrapes      ┌──────────┐
│  Prometheus  │ ──────────────── │  Targets  │ (apps, exporters)
│   Server     │                  └──────────┘
│              │     pushes       ┌──────────┐
│  - TSDB      │ ◄─────────────  │Pushgateway│ (short-lived jobs)
│  - Rules     │                  └──────────┘
│  - HTTP API  │
└──────┬───────┘
       │ fires alerts
       ▼
┌──────────────┐    notifies    ┌──────────┐
│ Alertmanager │ ─────────────► │  PagerDuty│
│              │                │  Slack    │
│              │                │  Email    │
└──────────────┘                └──────────┘
       
┌──────────────┐   queries      ┌──────────┐
│   Grafana    │ ─────────────► │Prometheus │
│              │   (PromQL)     │  HTTP API │
└──────────────┘                └──────────┘
```

### Key Components
- **Prometheus server** — scrapes, stores (TSDB), evaluates rules, serves HTTP API
- **Alertmanager** — receives alerts, deduplicates, groups, routes, silences, sends notifications
- **Pushgateway** — intermediary for short-lived jobs
- **Client libraries** — instrument your code (Go, Java, Python, Ruby, .NET, Rust)
- **Exporters** — expose third-party metrics in Prometheus format (node_exporter, blackbox_exporter, etc.)

### Data Model

Every time series is uniquely identified by:
- **Metric name** — what is being measured (e.g., `http_requests_total`)
- **Labels** — key-value pairs for dimensions (e.g., `{method="GET", status="200"}`)

Notation: `metric_name{label1="value1", label2="value2"}`

**Samples** are `(timestamp, float64 value)` pairs.

### Configuration (prometheus.yml)

```yaml
global:
  scrape_interval: 15s      # How often to scrape (default 1m)
  evaluation_interval: 15s  # How often to evaluate rules (default 1m)

rule_files:
  - "alerts.yml"
  - "recording_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
  
  - job_name: "node"
    static_configs:
      - targets: ["node-exporter:9100"]
```

### Limitations

Know these for the exam:
- **Not for logs or traces** — metrics only
- **Not for 100% accuracy** — designed for reliability, may lose data under load
- **Single-server by default** — no built-in clustering (use Thanos/Cortex/Mimir for HA)
- **Local storage is not durable** — not designed for long-term storage without remote write
- **Pull model** — can't monitor targets behind firewalls without Pushgateway or reverse proxies

---

## Phase 3: PromQL (28%) — The Largest Domain

### Selectors

```promql
# Instant vector — current value
http_requests_total

# With label matcher
http_requests_total{method="GET"}

# Regex matcher
http_requests_total{method=~"GET|POST"}

# Negative matcher
http_requests_total{status!="200"}

# Range vector — values over time window
http_requests_total[5m]
```

**Label matchers:** `=` (equal), `!=` (not equal), `=~` (regex match), `!~` (regex not match)

### Essential Functions

**rate()** — Per-second average rate of increase (for counters). The most important function.
```promql
rate(http_requests_total[5m])
```

**irate()** — Instant rate using last two data points. Volatile — use for graphs, NOT alerting.

**increase()** — Total increase over time range (= rate × seconds). Human-readable.
```promql
increase(http_requests_total[1h])  # Total requests in last hour
```

**histogram_quantile()** — Calculate percentiles from histograms.
```promql
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

### Aggregation Operators

```promql
# Sum all request rates by method
sum(rate(http_requests_total[5m])) by (method)

# Average memory across all instances
avg(node_memory_MemAvailable_bytes) by (instance)

# Top 5 highest CPU consumers
topk(5, rate(node_cpu_seconds_total{mode!="idle"}[5m]))

# Count number of targets up
count(up == 1)
```

Aggregation operators: `sum`, `avg`, `min`, `max`, `count`, `count_values`, `topk`, `bottomk`, `quantile`, `stddev`, `stdvar`

Keywords: `by` (keep these labels), `without` (drop these labels)

### Binary Operators

```promql
# Error rate percentage
sum(rate(http_requests_total{status=~"5.."}[5m])) /
sum(rate(http_requests_total[5m])) * 100

# Memory usage percentage
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) /
node_memory_MemTotal_bytes * 100
```

### absent() — Critical for Alerting

```promql
# Alert when a metric disappears (target down)
absent(up{job="myapp"})
```

Returns 1 if the metric has no data — essential for "something is missing" alerts.

---

## Phase 4: Instrumentation & Exporters (16%)

### Metric Types

| Type | Direction | Use For | Key Functions |
|---|---|---|---|
| **Counter** | Only goes up | Requests, errors, bytes sent | `rate()`, `increase()` |
| **Gauge** | Up and down | Temperature, memory, queue size | Direct value, `delta()` |
| **Histogram** | Buckets + sum + count | Latency, response sizes | `histogram_quantile()` |
| **Summary** | Quantiles at collection time | Similar to histogram | Direct quantile values |

**Counter vs Gauge:** If it can decrease, it's a gauge. If it only resets to zero (on restart), it's a counter.

**Histogram vs Summary:** Histograms are server-side aggregatable (can sum across instances). Summaries compute quantiles client-side and cannot be aggregated.

### Naming Conventions

- Prefix with application/domain: `http_`, `process_`, `node_`
- Use base units: `_seconds` (not ms), `_bytes` (not MB)
- Counters: must end with `_total` (e.g., `http_requests_total`)
- Use `_info` for metadata pseudo-metrics
- Use `_ratio` for percentages (0-1 scale, not 0-100)
- Avoid high-cardinality labels (no user IDs, email addresses)

### Key Exporters

| Exporter | Port | Purpose |
|---|---|---|
| **node_exporter** | 9100 | Linux/macOS system metrics (CPU, memory, disk, network) |
| **blackbox_exporter** | 9115 | Probe endpoints (HTTP, TCP, ICMP, DNS) |
| **mysqld_exporter** | 9104 | MySQL server metrics |
| **Prometheus itself** | 9090 | Self-monitoring at `/metrics` |

### Exposition Format

```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 1234 1625000000000
```

Lines: `# HELP` (description), `# TYPE` (metric type), then `metric{labels} value [timestamp]`

---

## Phase 5: Alerting & Dashboarding (18%)

### Alerting Rules

```yaml
# alerts.yml
groups:
  - name: example
    rules:
      - alert: HighErrorRate
        expr: sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Error rate above 5%"
          description: "Error rate is {{ $value | humanizePercentage }}"
```

- **alert** — name of the alert
- **expr** — PromQL expression (must resolve to boolean)
- **for** — how long the condition must be true before firing (avoids flapping)
- **labels** — additional labels attached to the alert (used for routing)
- **annotations** — human-readable descriptions (templates supported)

### Recording Rules

Pre-compute expensive queries and store as new metrics:

```yaml
groups:
  - name: recording
    rules:
      - record: job:http_requests:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job)
```

Naming convention: `level:metric:operations` (e.g., `job:http_requests:rate5m`)

### Alertmanager Concepts

**Grouping:** Combine similar alerts into one notification. E.g., all instances of "HighMemory" grouped into a single Slack message.

**Inhibition:** Suppress alerts when a related higher-severity alert is firing. E.g., suppress "InstanceDown" alerts when "DatacenterDown" is active.

**Silencing:** Temporarily mute alerts (maintenance windows). Set via the Alertmanager UI.

**Routing:** Alert → route tree → receiver (Slack, PagerDuty, email, webhook). Routes match on labels.

### Dashboarding (Grafana)

- Grafana is the standard visualization tool for Prometheus
- Add Prometheus as a data source (HTTP API at port 9090)
- Dashboards use PromQL queries
- Variables allow dynamic filtering (e.g., dropdown for `instance` label)
- Alerts can be defined in Grafana too (separate from Prometheus alerting rules)

---

## Quick Reference Card

| Item | Value |
|---|---|
| Default scrape interval | 1m (commonly set to 15s) |
| Default eval interval | 1m (commonly set to 15s) |
| Default metrics path | `/metrics` |
| Prometheus port | 9090 |
| Alertmanager port | 9093 |
| node_exporter port | 9100 |
| blackbox_exporter port | 9115 |
| Counter suffix | `_total` |
| Base time unit | seconds |
| Base data unit | bytes |
| rate() input | range vector (e.g., `[5m]`) |
| rate() output | instant vector (per-second rate) |
| Label matchers | `=`, `!=`, `=~`, `!~` |
| Range durations | ms, s, m, h, d, w, y |
| Recording rule naming | `level:metric:operations` |
| Pull model exception | Pushgateway (for short-lived jobs) |

---

## Readiness Checklist

- [ ] Can explain Prometheus architecture (server, Alertmanager, exporters, Pushgateway)
- [ ] Know the data model: metric name + labels + samples
- [ ] Can write prometheus.yml from scratch (global, scrape_configs, alerting)
- [ ] Know all 4 metric types and when to use each
- [ ] Can write rate(), increase(), histogram_quantile() queries fluently
- [ ] Know aggregation operators (sum, avg, topk) with by/without
- [ ] Can write alerting rules with expr, for, labels, annotations
- [ ] Understand Alertmanager: grouping, inhibition, silencing, routing
- [ ] Know recording rule syntax and naming convention
- [ ] Can explain push vs pull, and when to use Pushgateway
- [ ] Know key exporters (node_exporter, blackbox_exporter) and their ports
- [ ] Understand naming conventions (_total, _seconds, _bytes, _info)
- [ ] Can explain SLI/SLO/SLA with examples
- [ ] Know Prometheus limitations (no logs/traces, single server, local storage)
