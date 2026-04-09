# Dash0 API Crash Course — Companion Study Guide

**Course:** Dash0 API Crash Course (v1.0, Platform v3.1)
**Audience:** Sales Reps, Solutions Architects, Engineers/DevOps
**Goal:** Fluency with the Dash0 API surface — from first auth token to production observability-as-code

---

## 1. Course Overview & Prerequisites

### What This Course Covers

The Dash0 API has three distinct surfaces, each serving a different role:

| Surface | Purpose | Direction |
|---|---|---|
| **OTLP Ingestion** | Send telemetry (traces, metrics, logs) | Write (inbound) |
| **Prometheus Query API** | Query stored metrics via PromQL | Read (outbound) |
| **Management API** | CRUD dashboards, check-rules, config | Control plane |

Every action available in the Dash0 UI is also accessible via API. This makes Dash0 automation-friendly and GitOps-ready from day one — and it is a key differentiator against UI-first observability platforms.

### Prerequisites

**Required:**
- Basic HTTP/REST concepts (GET, POST, PUT, DELETE, response codes)
- Familiarity with `curl` for command-line HTTP calls
- Understanding of what traces, metrics, and logs are

**Helpful but not required:**
- PromQL basics (will be covered in Phase 2)
- OpenTelemetry concepts (OTLP, OTel Collector, resource attributes)
- Kubernetes and GitOps workflows

**Tools to install:**
```bash
brew install curl jq          # curl for HTTP calls, jq for JSON pretty-printing
brew install opentelemetry-collector  # optional, for OTel Collector exercises
```

### Learning Phases

| Phase | Focus | Lessons | Time |
|---|---|---|---|
| Phase 1 (Week 1) | Foundations: API overview, auth, OTLP ingestion | 1–3 | ~2h |
| Phase 2 (Week 2) | Prometheus queries, Management API CRUD | 4–6 | ~2.5h |
| Phase 3 (Week 3) | Observability-as-code, integrations, competitive positioning | 7–8 | ~2h |

---

## 2. Phase 1: Foundations

### API Architecture

All Dash0 API traffic is region-specific. Two hostname patterns per region:

```
Ingestion:  ingress.<region>.dash0.com   (write path — OTLP in)
API/Query:  api.<region>.dash0.com       (read/control path)
```

Example for EU West 1:
- `ingress.eu-west-1.dash0.com` — receives telemetry
- `api.eu-west-1.dash0.com` — Management API and Prometheus query API

**Common regions:** `eu-west-1`, `us-east-1`, `us-west-2` (verify current list in Dash0 docs)

### Auth Tokens

All Dash0 API requests use Bearer token authentication. Tokens:
- Are created in the Dash0 UI under **Settings > Auth Tokens**
- Have the format `ds0_<random-string>`
- Are scoped per organization
- Support different permission levels

**Auth header format:**
```
Authorization: Bearer ds0_YOUR_TOKEN_HERE
```

This same header format is used for all three API surfaces (OTLP ingestion, Prometheus query, and Management API).

### Endpoints Reference (Phase 1 Focus)

| Endpoint | Protocol | Port | Purpose |
|---|---|---|---|
| `ingress.<region>.dash0.com` | OTLP/gRPC | 4317 | Telemetry ingestion (gRPC) |
| `ingress.<region>.dash0.com` | OTLP/HTTP | 4318 | Telemetry ingestion (HTTP) |
| `ingress.<region>.dash0.com/v1/traces` | HTTPS | 443 | Send traces |
| `ingress.<region>.dash0.com/v1/metrics` | HTTPS | 443 | Send metrics |
| `ingress.<region>.dash0.com/v1/logs` | HTTPS | 443 | Send logs |

All endpoints require TLS. HTTP uses port 443, but the OTLP standard ports (4317/4318) are the primary ports for SDK configuration.

### OTLP Transport: gRPC vs. HTTP

| | gRPC (port 4317) | HTTP (port 4318) |
|---|---|---|
| **Format** | Binary Protobuf | JSON or Protobuf |
| **Best for** | High-volume production | Debugging, proxied environments |
| **curl-friendly** | No | Yes |
| **Firewall-friendly** | Sometimes not | Yes |
| **Default for OTel SDKs** | Yes (most SDKs) | Config option |

### OTel SDK Environment Variables

Configure any OpenTelemetry SDK to send to Dash0 with three environment variables — no code changes needed:

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT=https://ingress.eu-west-1.dash0.com
export OTEL_EXPORTER_OTLP_HEADERS=Authorization=Bearer ds0_YOUR_TOKEN
export OTEL_SERVICE_NAME=my-service
# Optional but recommended:
export OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production,service.version=1.0.0
```

---

## 3. Phase 2: Intermediate

### OTLP Ingestion

**Sending logs with curl (the canonical test):**
```bash
curl -X POST https://ingress.eu-west-1.dash0.com/v1/logs \
  -H 'Authorization: Bearer ds0_YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "resourceLogs": [{
      "resource": {
        "attributes": [{
          "key": "service.name",
          "value": {"stringValue": "curl-test"}
        }]
      },
      "scopeLogs": [{
        "logRecords": [{
          "body": {"stringValue": "Hello from Dash0 API!"},
          "severityText": "INFO"
        }]
      }]
    }]
  }'
```

Expected response: `HTTP 200 OK` with body `{}`

**Critical resource attributes to always include:**

| Attribute | Required | Purpose |
|---|---|---|
| `service.name` | **Yes** | Routing, filtering, UI display |
| `service.version` | Recommended | Version comparison |
| `service.namespace` | Recommended | Multi-service grouping |
| `deployment.environment` | Recommended | Env filtering (prod/staging) |
| `host.name` | Recommended | Host-level correlation |

### PromQL Queries via the Prometheus API

Dash0 exposes a Prometheus-compatible query API at `api.<region>.dash0.com/prometheus`.

**Two query endpoints:**

| Endpoint | Use Case | Returns |
|---|---|---|
| `/prometheus/api/v1/query` | Point-in-time value | `resultType: vector` |
| `/prometheus/api/v1/query_range` | Time series over window | `resultType: matrix` |

**Response envelope (all Prometheus API responses):**
```json
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {"__name__": "up", "job": "myapp"},
        "value": [1700000000, "1"]
      }
    ]
  }
}
```

For range queries, `resultType` is `"matrix"` and each result has `"values"` (array of `[timestamp, value]` pairs) instead of `"value"`.

### Management API

The Management API lives at `api.<region>.dash0.com/api/v1/` and manages dashboards and check-rules.

**Dashboard CRUD:**
```
GET    /api/v1/dashboards          # list all dashboards
POST   /api/v1/dashboards          # create new dashboard
GET    /api/v1/dashboards/<id>     # get one dashboard
PUT    /api/v1/dashboards/<id>     # update (full replace)
DELETE /api/v1/dashboards/<id>     # delete (irreversible — export first!)
```

**Check-rule CRUD:**
```
GET    /api/v1/check-rules         # list all alert rules
POST   /api/v1/check-rules         # create new rule
GET    /api/v1/check-rules/<id>    # get one rule
PUT    /api/v1/check-rules/<id>    # update
DELETE /api/v1/check-rules/<id>    # delete
```

**Response codes:**
- `200 OK` — success (GET, PUT)
- `201 Created` — resource created (POST)
- `204 No Content` — deleted (DELETE)
- `400 Bad Request` — malformed payload
- `401 Unauthorized` — invalid or missing token
- `404 Not Found` — resource does not exist
- `429 Too Many Requests` — rate limited

---

## 4. Phase 3: Advanced

### Observability-as-Code

The core pattern: store dashboards and alert rules as files in Git, deploy via the Management API in CI/CD pipelines.

**Why this matters:**
- Code review for observability changes (PRs for dashboards)
- Rollback via `git revert`
- Environment promotion (dev → staging → prod)
- Disaster recovery
- Team collaboration on observability config

**Two deployment tools:**

1. **Bash/curl scripts** — simplest, no extra tooling
2. **Terraform** — `registry.terraform.io/providers/dash0hq/dash0` — for teams already using IaC

The Dash0 CLI (`dash0 apply`, `dash0 get`, `dash0 delete`) is a thin convenience wrapper over the same Management API REST calls.

### Integrations Overview

| Integration | How It Works | Use Case |
|---|---|---|
| **Grafana** | Add Prometheus data source pointing to Dash0's Prometheus endpoint | Keep existing Grafana dashboards, switch data source |
| **OTel Collector** | Use standard `otlp` exporter with Dash0 endpoint + auth header | Send from existing Collector pipelines |
| **Prometheus** | Configure `remote_write` to Dash0's remote_write endpoint | Long-term storage for existing Prometheus metrics |
| **Dash0 Operator** | Kubernetes operator auto-instruments pods via mutating webhook | Zero-code auto-instrumentation for k8s |

### Competitive Positioning (API-Specific)

**vs. Datadog:** Datadog requires proprietary agents and uses a non-standard query language (NRQL). Dashboard format is proprietary and not portable. Switching requires re-instrumentation. Dash0 uses OTLP + PromQL + Perses — all open standards. Pointing OTel exporters at a new endpoint is all that's needed to switch.

**vs. Grafana Cloud:** Open source friendly but the API surface is fragmented across Loki, Mimir, Tempo, and Grafana — each with different APIs and auth models. Dash0 has a single unified API surface with one auth model.

**vs. Dynatrace:** Powerful auto-discovery comes with deep proprietary lock-in (OneAgent, DQL, proprietary dashboard format). Dash0's Operator provides similar Kubernetes auto-instrumentation without proprietary agents.

---

## 5. Endpoint Reference Table

| Method | Path | Description |
|---|---|---|
| `POST` | `ingress.<region>.dash0.com/v1/traces` | Send OTLP traces (HTTP) |
| `POST` | `ingress.<region>.dash0.com/v1/metrics` | Send OTLP metrics (HTTP) |
| `POST` | `ingress.<region>.dash0.com/v1/logs` | Send OTLP logs (HTTP) |
| `POST` | `ingress.<region>.dash0.com/prometheus/remote/write` | Prometheus remote_write |
| `GET/POST` | `api.<region>.dash0.com/prometheus/api/v1/query` | Instant PromQL query |
| `GET/POST` | `api.<region>.dash0.com/prometheus/api/v1/query_range` | Range PromQL query |
| `GET` | `api.<region>.dash0.com/prometheus/api/v1/labels` | List all label names |
| `GET` | `api.<region>.dash0.com/prometheus/api/v1/label/<name>/values` | Label values |
| `GET` | `api.<region>.dash0.com/prometheus/api/v1/series` | List matching time series |
| `GET` | `api.<region>.dash0.com/api/v1/dashboards` | List all dashboards |
| `POST` | `api.<region>.dash0.com/api/v1/dashboards` | Create dashboard |
| `GET` | `api.<region>.dash0.com/api/v1/dashboards/<id>` | Get one dashboard |
| `PUT` | `api.<region>.dash0.com/api/v1/dashboards/<id>` | Update dashboard |
| `DELETE` | `api.<region>.dash0.com/api/v1/dashboards/<id>` | Delete dashboard |
| `GET` | `api.<region>.dash0.com/api/v1/check-rules` | List all check-rules |
| `POST` | `api.<region>.dash0.com/api/v1/check-rules` | Create check-rule |
| `GET` | `api.<region>.dash0.com/api/v1/check-rules/<id>` | Get one check-rule |
| `PUT` | `api.<region>.dash0.com/api/v1/check-rules/<id>` | Update check-rule |
| `DELETE` | `api.<region>.dash0.com/api/v1/check-rules/<id>` | Delete check-rule |

**OTLP gRPC:** `ingress.<region>.dash0.com:4317` (all three signal types, Protobuf)

---

## 6. Auth Token Setup Walkthrough

1. **Open the Dash0 UI** and navigate to **Settings > Auth Tokens**
2. Click **Create Token**
3. Give it a descriptive name (e.g., `dev-local`, `ci-pipeline-staging`)
4. Select permissions scope — for full API access, choose org-level read/write
5. Copy the token immediately — it is only shown once. Format: `ds0_<random>`
6. Store it securely (1Password, HashiCorp Vault, GitHub Actions secrets, etc.)

**Test your token immediately:**
```bash
export DASH0_TOKEN="ds0_your_token_here"
export DASH0_REGION="eu-west-1"

# Test: send an empty log payload — should return 200 {}
curl -s -w "\nHTTP %{http_code}\n" \
  -X POST "https://ingress.${DASH0_REGION}.dash0.com/v1/logs" \
  -H "Authorization: Bearer ${DASH0_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"resourceLogs":[]}'
```

Expected output:
```
{}
HTTP 200
```

If you see `HTTP 401`, double-check:
- Token is copied correctly (no trailing spaces/newlines)
- Header format is exactly `Authorization: Bearer ds0_...`
- Token has not expired or been revoked
- You are hitting the correct regional endpoint

---

## 7. curl Examples for Common Operations

### Send a Trace (Minimal Payload)
```bash
curl -X POST https://ingress.eu-west-1.dash0.com/v1/traces \
  -H "Authorization: Bearer ${DASH0_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "resourceSpans": [{
      "resource": {
        "attributes": [
          {"key": "service.name", "value": {"stringValue": "curl-trace-test"}}
        ]
      },
      "scopeSpans": [{
        "spans": [{
          "traceId": "0102030405060708090a0b0c0d0e0f10",
          "spanId": "0102030405060708",
          "name": "test-span",
          "kind": 1,
          "startTimeUnixNano": "1700000000000000000",
          "endTimeUnixNano":   "1700000001000000000",
          "status": {"code": 1}
        }]
      }]
    }]
  }'
```

### Send a Metric (Gauge)
```bash
curl -X POST https://ingress.eu-west-1.dash0.com/v1/metrics \
  -H "Authorization: Bearer ${DASH0_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "resourceMetrics": [{
      "resource": {
        "attributes": [{"key": "service.name", "value": {"stringValue": "curl-metric-test"}}]
      },
      "scopeMetrics": [{
        "metrics": [{
          "name": "test.gauge",
          "unit": "1",
          "gauge": {
            "dataPoints": [{
              "asDouble": 42.0,
              "timeUnixNano": "1700000000000000000",
              "attributes": [{"key": "env", "value": {"stringValue": "dev"}}]
            }]
          }
        }]
      }]
    }]
  }'
```

### Instant PromQL Query
```bash
curl -G "https://api.eu-west-1.dash0.com/prometheus/api/v1/query" \
  -H "Authorization: Bearer ${DASH0_TOKEN}" \
  --data-urlencode "query=up" \
  --data-urlencode "time=$(date +%s)" | jq '.'
```

### Range Query (last 1 hour, 60-second steps)
```bash
NOW=$(date +%s)
START=$((NOW - 3600))

curl -G "https://api.eu-west-1.dash0.com/prometheus/api/v1/query_range" \
  -H "Authorization: Bearer ${DASH0_TOKEN}" \
  --data-urlencode "query=rate(http_requests_total[5m])" \
  --data-urlencode "start=${START}" \
  --data-urlencode "end=${NOW}" \
  --data-urlencode "step=60" | jq '.data.result[0]'
```

### List All Dashboards
```bash
curl -s \
  -H "Authorization: Bearer ${DASH0_TOKEN}" \
  "https://api.eu-west-1.dash0.com/api/v1/dashboards" | jq '[.[] | {id: .metadata.name, title: .spec.display.name}]'
```

### Create a Dashboard from a File
```bash
curl -X POST "https://api.eu-west-1.dash0.com/api/v1/dashboards" \
  -H "Authorization: Bearer ${DASH0_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @dashboard.json
```

### Update an Existing Dashboard
```bash
DASHBOARD_ID="my-service-overview"
curl -X PUT "https://api.eu-west-1.dash0.com/api/v1/dashboards/${DASHBOARD_ID}" \
  -H "Authorization: Bearer ${DASH0_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @dashboard.json
```

### Create a Check Rule
```bash
curl -X POST "https://api.eu-west-1.dash0.com/api/v1/check-rules" \
  -H "Authorization: Bearer ${DASH0_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "metadata": {"name": "high-error-rate"},
    "spec": {
      "expr": "rate(http_errors_total[5m]) / rate(http_requests_total[5m]) > 0.05",
      "for": "2m",
      "labels": {"severity": "warning", "team": "backend"},
      "annotations": {
        "summary": "High HTTP error rate",
        "description": "Error rate is {{ $value | humanizePercentage }} over last 5m"
      }
    }
  }'
```

### Export All Dashboards to JSON Files
```bash
#!/bin/bash
TOKEN="${DASH0_TOKEN}"
ENDPOINT="https://api.eu-west-1.dash0.com"

curl -s -H "Authorization: Bearer $TOKEN" \
  "$ENDPOINT/api/v1/dashboards" | \
  jq -c '.[]' | \
  while read -r dash; do
    NAME=$(echo "$dash" | jq -r '.metadata.name')
    echo "$dash" | jq '.' > "dashboard-${NAME}.json"
    echo "Exported: $NAME"
  done
```

---

## 8. PromQL Query Patterns via API

All queries are URL-encoded and passed in the `query` parameter.

### Basic Counter Rate
```promql
rate(http_requests_total[5m])
```
Per-second average rate over a 5-minute window. Always use `rate()` (not raw counter values) for counters.

### Filter by Label
```promql
http_requests_total{service_name="checkout", status_code="200"}
```
Use `=` for exact match, `!=` for not-equal, `=~` for regex, `!~` for negative regex.

### HTTP Error Rate (as percentage)
```promql
rate(http_requests_total{status=~"5.."}[5m])
  /
rate(http_requests_total[5m])
  * 100
```

### P99 Latency from Histogram
```promql
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

### Aggregate Across Services (sum)
```promql
sum by (service_name) (rate(http_requests_total[5m]))
```

### Top 5 Services by Request Rate
```promql
topk(5, sum by (service_name) (rate(http_requests_total[5m])))
```

### Detect Missing Metrics (up check)
```promql
up == 0
```

### SLO Error Budget Burn Rate
```promql
(
  rate(http_errors_total[1h]) / rate(http_requests_total[1h])
) / 0.001  # assuming 99.9% SLO target (0.1% error budget)
```

### curl for Filtered Range Query
```bash
curl -G "https://api.eu-west-1.dash0.com/prometheus/api/v1/query_range" \
  -H "Authorization: Bearer ${DASH0_TOKEN}" \
  --data-urlencode 'query=rate(http_requests_total{service_name="checkout",status=~"5.."}[5m])' \
  --data-urlencode "start=$(($(date +%s) - 3600))" \
  --data-urlencode "end=$(date +%s)" \
  --data-urlencode "step=60"
```

**Note:** Always use `--data-urlencode` for PromQL queries in curl. Curly braces and quotes in label matchers require URL encoding.

### Label Discovery Queries
```bash
# List all label names
curl -s -H "Authorization: Bearer ${DASH0_TOKEN}" \
  "https://api.eu-west-1.dash0.com/prometheus/api/v1/labels" | jq '.data'

# List values for a specific label
curl -s -H "Authorization: Bearer ${DASH0_TOKEN}" \
  "https://api.eu-west-1.dash0.com/prometheus/api/v1/label/service_name/values" | jq '.data'

# Find all series matching a selector
curl -G -s -H "Authorization: Bearer ${DASH0_TOKEN}" \
  "https://api.eu-west-1.dash0.com/prometheus/api/v1/series" \
  --data-urlencode 'match[]={job="myapp"}' | jq '.data | length'
```

---

## 9. Dashboard-as-Code YAML Examples

Dash0 dashboards use the [Perses](https://perses.dev/) open-source format. Below are working examples.

### Minimal Dashboard Skeleton
```json
{
  "kind": "Dashboard",
  "metadata": {
    "name": "my-service-overview",
    "project": "default"
  },
  "spec": {
    "display": {
      "name": "My Service Overview"
    },
    "duration": "1h",
    "variables": [],
    "panels": {}
  }
}
```

### Dashboard with a Time Series Panel
```json
{
  "kind": "Dashboard",
  "metadata": {
    "name": "http-metrics",
    "project": "default"
  },
  "spec": {
    "display": {"name": "HTTP Metrics"},
    "duration": "1h",
    "panels": {
      "request-rate": {
        "kind": "Panel",
        "spec": {
          "display": {"name": "Request Rate (req/s)"},
          "plugin": {
            "kind": "TimeSeriesChart",
            "spec": {}
          },
          "queries": [{
            "kind": "TimeSeriesQuery",
            "spec": {
              "plugin": {
                "kind": "PrometheusTimeSeriesQuery",
                "spec": {
                  "query": "sum by (service_name) (rate(http_requests_total[5m]))"
                }
              }
            }
          }]
        }
      }
    },
    "layouts": [{
      "kind": "Grid",
      "spec": {
        "items": [{
          "x": 0, "y": 0, "width": 12, "height": 6,
          "content": {"$ref": "#/spec/panels/request-rate"}
        }]
      }
    }]
  }
}
```

### Dashboard Deploy Script (CI/CD Pattern)
```bash
#!/bin/bash
# deploy-dashboards.sh — sync all local dashboards to Dash0
set -euo pipefail

TOKEN="${DASH0_TOKEN:?DASH0_TOKEN is required}"
ENDPOINT="${DASH0_ENDPOINT:-https://api.eu-west-1.dash0.com}"
DASHBOARDS_DIR="${1:-./dashboards}"

for file in "${DASHBOARDS_DIR}"/*.json; do
  NAME=$(jq -r '.metadata.name' "$file")
  
  # Check if dashboard already exists
  HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
    -H "Authorization: Bearer $TOKEN" \
    "${ENDPOINT}/api/v1/dashboards/${NAME}")
  
  if [ "$HTTP_CODE" = "200" ]; then
    curl -s -X PUT "${ENDPOINT}/api/v1/dashboards/${NAME}" \
      -H "Authorization: Bearer $TOKEN" \
      -H "Content-Type: application/json" \
      --data-binary @"$file" > /dev/null
    echo "Updated: $NAME"
  else
    curl -s -X POST "${ENDPOINT}/api/v1/dashboards" \
      -H "Authorization: Bearer $TOKEN" \
      -H "Content-Type: application/json" \
      --data-binary @"$file" > /dev/null
    echo "Created: $NAME"
  fi
done

echo "Done."
```

### Terraform Resource for a Dashboard
```hcl
resource "dash0_dashboard" "service_overview" {
  dataset        = "default"
  dashboard_yaml = file("${path.module}/dashboards/service-overview.yaml")
}

resource "dash0_check_rule" "high_error_rate" {
  dataset         = "default"
  check_rule_yaml = file("${path.module}/rules/high-error-rate.yaml")
}
```

---

## 10. Scenario Drills with Model Answers

### Scenario 1 — Sales
**A prospect says: "We already have Datadog. Why would we switch?"**

<details>
<summary>Model Answer</summary>

Acknowledge the investment first — don't dismiss it. Then reframe:

"We're not asking you to rip out Datadog today. The question is: what happens when you want to grow, and what does switching look like if you ever want to?

Datadog requires proprietary agents and charges per host. Dash0 uses standard OpenTelemetry — so your instrumentation works with both platforms simultaneously. Many teams start Dash0 alongside Datadog for specific workloads, validate the value, and expand from there.

The migration story is simple: your OTel-instrumented apps just change one environment variable — `OTEL_EXPORTER_OTLP_ENDPOINT`. No re-instrumentation. That's the open standards advantage."

</details>

---

### Scenario 2 — Sales
**A prospect asks: "What's the difference between your API and just running Prometheus?"**

<details>
<summary>Model Answer</summary>

"Prometheus is an excellent self-hosted TSDB but comes with operational burden: storage limits, no SaaS management, limited long-term retention, and no native traces or logs.

Dash0 provides fully-managed long-term storage with a Prometheus-compatible query API — your existing PromQL queries work without changes. But you also get traces, logs, dashboards, and alert management in one platform.

It's the familiarity of Prometheus without the operational overhead. And you can even keep running Prometheus locally and use `remote_write` to ship metrics to Dash0 — zero disruption."

</details>

---

### Scenario 3 — Solutions Architect
**A customer wants to connect their existing OTel Collector to Dash0 without any code changes.**

<details>
<summary>Model Answer</summary>

Add an `otlp` exporter to their existing `collector.yaml` — no application changes needed:

```yaml
exporters:
  otlp/dash0:
    endpoint: ingress.eu-west-1.dash0.com:4317
    headers:
      Authorization: "Bearer ds0_YOUR_TOKEN"
    tls:
      insecure: false

service:
  pipelines:
    traces:
      exporters: [existing_exporter, otlp/dash0]
    metrics:
      exporters: [existing_exporter, otlp/dash0]
    logs:
      exporters: [existing_exporter, otlp/dash0]
```

This fans out telemetry to both the existing backend and Dash0. The team can validate Dash0 data in parallel without disrupting current observability.

</details>

---

### Scenario 4 — Solutions Architect
**A customer's OTLP data isn't appearing in Dash0 after configuration. Walk through the troubleshooting steps.**

<details>
<summary>Model Answer</summary>

Work through the checklist systematically:

1. **Test auth in isolation** — send an empty payload via curl:
   ```bash
   curl -w "\nHTTP %{http_code}" -X POST https://ingress.eu-west-1.dash0.com/v1/logs \
     -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"resourceLogs":[]}'
   ```
   - `200` = auth is fine, problem is elsewhere
   - `401` = token issue (format, validity, or scope)

2. **Verify header format** — it must be exactly `Authorization: Bearer ds0_...`. Not `X-API-Key`, not `Auth-Token`.

3. **Check regional endpoint** — token and endpoint must be for the same region. EU token + US endpoint = 401.

4. **Confirm `service.name` attribute is set** — Dash0 uses this for routing and display. Missing it won't prevent ingestion but makes data hard to find.

5. **Check OTel Collector logs** — look for export errors or connection timeouts.

6. **Verify TLS** — Dash0 requires HTTPS. If `insecure: true` is set, change to `false`.

</details>

---

### Scenario 5 — Solutions Architect
**A customer wants automated dashboard deployment in CI/CD. Walk through the approach.**

<details>
<summary>Model Answer</summary>

The pattern:

1. **Store dashboard JSON files in Git** — one file per dashboard, named by `metadata.name`.
2. **In CI (e.g., GitHub Actions)**:
   - `GET /api/v1/dashboards/<name>` — check if dashboard exists (HTTP 200 = exists, 404 = new)
   - If exists: `PUT /api/v1/dashboards/<name>` with updated JSON
   - If new: `POST /api/v1/dashboards` with the JSON
3. **Optionally**: Run `perses lint dashboard.json` in CI to validate schema before deploying.
4. **Environment isolation**: Use separate Dash0 tokens per environment (dev/staging/prod), each stored as a CI/CD secret.

This is exactly how the Dash0 CLI works internally — it's just a wrapper over these same REST calls with added conveniences like YAML parsing and diffing.

</details>

---

### Scenario 6 — Technical / Engineer
**Write a bash script that exports all Dash0 dashboards to local JSON files.**

<details>
<summary>Model Answer</summary>

```bash
#!/bin/bash
TOKEN="${DASH0_TOKEN:?}"
ENDPOINT="https://api.eu-west-1.dash0.com"
OUT_DIR="${1:-./dashboards-export}"

mkdir -p "$OUT_DIR"

curl -s -H "Authorization: Bearer $TOKEN" \
  "$ENDPOINT/api/v1/dashboards" | \
  jq -c '.[]' | \
  while read -r dash; do
    NAME=$(echo "$dash" | jq -r '.metadata.name')
    echo "$dash" | jq '.' > "${OUT_DIR}/dashboard-${NAME}.json"
    echo "Exported: $NAME"
  done

echo "Done. Files in: $OUT_DIR"
```

Run it: `DASH0_TOKEN=ds0_... bash export-dashboards.sh`

</details>

---

### Scenario 7 — Technical / Engineer
**Build a PromQL-based SLO alert for a checkout service with a 99.9% availability SLO.**

<details>
<summary>Model Answer</summary>

```bash
curl -X POST "https://api.eu-west-1.dash0.com/api/v1/check-rules" \
  -H "Authorization: Bearer ${DASH0_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "metadata": {"name": "checkout-slo-breach"},
    "spec": {
      "expr": "rate(http_errors_total{service_name=\"checkout\"}[5m]) / rate(http_requests_total{service_name=\"checkout\"}[5m]) > 0.001",
      "for": "5m",
      "labels": {
        "severity": "critical",
        "slo": "checkout-99.9",
        "team": "payments"
      },
      "annotations": {
        "summary": "SLO breach: checkout error rate above 0.1%",
        "description": "Error rate is {{ $value | humanizePercentage }} over last 5m"
      }
    }
  }'
```

Key points:
- `0.001` threshold = 0.1% error budget for 99.9% SLO
- `for: "5m"` prevents transient spikes from firing false alerts
- Label `slo: checkout-99.9` enables filtering all SLO rules in dashboards
- `humanizePercentage` template function formats the value in alert messages

</details>

---

### Scenario 8 — Technical / Engineer
**Set up Grafana to query Dash0 metrics. What exact configuration is needed?**

<details>
<summary>Model Answer</summary>

In Grafana: **Configuration > Data Sources > Add data source > Prometheus**

Fill in:
- **URL:** `https://api.eu-west-1.dash0.com/prometheus`
- **Auth:** toggle on **Custom HTTP Headers**
- **Header:** `Authorization` | **Value:** `Bearer ds0_YOUR_TOKEN`
- Leave all other settings as default

Click **Save & Test** — should show "Data source is working."

All existing Grafana dashboards that use PromQL now work against Dash0 data without modification. No plugin installation needed — Dash0's standard Prometheus API is natively compatible with Grafana's built-in Prometheus data source.

</details>

---

## 11. Competitive Cheat Sheet

### API Approach Comparison

| Dimension | Dash0 | Datadog | Grafana Cloud | Dynatrace |
|---|---|---|---|---|
| **Ingestion protocol** | OTLP (open standard) | Proprietary agent + OTLP | OTLP, Prometheus | Proprietary OneAgent |
| **Query language** | PromQL (open standard) | DQL / NRQL (proprietary) | PromQL, LogQL (open) | DQL (proprietary) |
| **Dashboard format** | Perses (open, CNCF) | Proprietary JSON | Grafana JSON (semi-open) | Proprietary |
| **Dashboard API** | Full CRUD REST | Full CRUD REST | REST (per-component) | REST |
| **GitOps story** | Native, 1st-class | Possible but complex | Possible but complex | Limited |
| **Migration cost** | Near-zero (OTel standard) | High (re-instrument) | Medium | High (OneAgent) |
| **Terraform provider** | Yes (`dash0hq/dash0`) | Yes | Yes | Yes |
| **Prometheus compatibility** | Native (remote_write, PromQL) | Limited (bridge) | Native (Mimir) | Limited |

### Key Differentiators to Memorize

**Datadog vs. Dash0:**
- Datadog: proprietary agents + NRQL query language + proprietary dashboard format = high lock-in
- Dash0: OTLP + PromQL + Perses = open at every layer, zero lock-in
- Migration: change one env var (`OTEL_EXPORTER_OTLP_ENDPOINT`), not a full re-instrumentation

**Grafana Cloud vs. Dash0:**
- Grafana Cloud: powerful but fragmented (Loki API, Mimir API, Tempo API, Grafana API — different auth for each)
- Dash0: single unified API surface, one auth token, one endpoint pattern
- Simpler operational model with equal open-standards commitment

**Dynatrace vs. Dash0:**
- Dynatrace: deep auto-discovery but deep lock-in (OneAgent required, DQL non-portable)
- Dash0 Operator: comparable k8s auto-instrumentation via standard OTel, no proprietary agent
- "Auto-instrumentation without agent lock-in"

### The 30-Second API Pitch

> "Dash0's API is built entirely on open standards — OTLP for ingestion, PromQL for querying, Perses for dashboards. This means your existing OTel instrumentation, Grafana dashboards, and PromQL queries work with Dash0 today. And your team can manage all observability configuration as code in Git — version-controlled, reviewed, and deployed like application infrastructure."

---

## 12. Quick Reference Card

### Auth
```
Header: Authorization: Bearer ds0_<token>
Token location: Dash0 UI → Settings → Auth Tokens
Token prefix: ds0_
```

### Endpoints (EU West 1)
```
OTLP gRPC:     ingress.eu-west-1.dash0.com:4317
OTLP HTTP:     ingress.eu-west-1.dash0.com:4318
Traces:        POST ingress.eu-west-1.dash0.com/v1/traces
Metrics:       POST ingress.eu-west-1.dash0.com/v1/metrics
Logs:          POST ingress.eu-west-1.dash0.com/v1/logs
Prometheus:    api.eu-west-1.dash0.com/prometheus/api/v1/...
Management:    api.eu-west-1.dash0.com/api/v1/...
```

### OTel SDK Env Vars
```bash
OTEL_EXPORTER_OTLP_ENDPOINT=https://ingress.eu-west-1.dash0.com
OTEL_EXPORTER_OTLP_HEADERS=Authorization=Bearer ds0_YOUR_TOKEN
OTEL_SERVICE_NAME=my-service
```

### Management API Paths
```
Dashboards:   /api/v1/dashboards[/<id>]
Check Rules:  /api/v1/check-rules[/<id>]
Methods:      GET (list/get), POST (create), PUT (update), DELETE
```

### Prometheus Query Params
```
Instant:  /query?query=<PromQL>&time=<unix>
Range:    /query_range?query=<PromQL>&start=<ts>&end=<ts>&step=<dur>
Labels:   /labels
Values:   /label/<name>/values
Series:   /series?match[]=<selector>
```

### HTTP Response Codes
```
200 OK           — success (GET, PUT, OTLP ingestion)
201 Created      — resource created (POST management)
204 No Content   — deleted (DELETE)
400 Bad Request  — malformed payload
401 Unauthorized — invalid/missing token
404 Not Found    — resource not found
429 Too Many Req — rate limited
```

### Rapid Fire Facts
```
OTLP gRPC port:     4317
OTLP HTTP port:     4318
Successful OTLP response: 200 {}
Dashboard format:   Perses (CNCF open source)
Query language:     PromQL
Dash0 token prefix: ds0_
API categories:     OTLP Ingestion, Prometheus Query, Management
```

---

*Companion to: `public-courses/dash0-api-crash-course.html` (v1.0, Platform v3.1)*
*Last updated: 2026-04-09*
