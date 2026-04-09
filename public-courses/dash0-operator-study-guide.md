# Dash0 Operator — Companion Study Guide

**Course:** Dash0 Operator Crash Course (v1.0)
**Audience:** Sales Reps, Solutions Architects, Engineers/DevOps
**Depth:** From zero-code story to production-grade OTTL filters

---

## 1. Course Overview & Prerequisites

### What This Course Covers

The Dash0 Operator is a Kubernetes operator that delivers zero-code observability. One Helm install and one YAML per namespace gives you traces, metrics, logs, and profiles — all powered by OpenTelemetry — without modifying application code.

This guide covers:
- Phase 1: Foundations — CRD hierarchy, Helm install, auto-instrumentation
- Phase 2: Intermediate — Export patterns, telemetry pipeline, Prometheus/Perses
- Phase 3: Advanced — Self-monitoring, profiling, auto-namespace monitoring, troubleshooting

### Prerequisites

| Prerequisite | Notes |
|---|---|
| Kubernetes cluster access | A `kind`, `minikube`, or real cluster at K8s 1.25.16+ |
| `kubectl` configured | `kubectl cluster-info` should succeed |
| Helm 3.x installed | `helm version` should show v3.x |
| Dash0 account + auth token | Settings → API Tokens in the Dash0 UI |
| Basic K8s concepts | Pods, Deployments, Namespaces, CRDs, DaemonSets |

### Role-Based Learning Paths

| Role | Focus Areas | Lessons |
|---|---|---|
| Sales Rep | Zero-code story, competitive positioning, objection handling | 1, 3, 8 |
| Solutions Architect | CRD hierarchy, export patterns, namespace strategies, demo-ready | 1–6, 8 |
| Engineer / DevOps | All of the above + OTTL, troubleshooting, Helm values deep dive | 1–8 |

---

## 2. Phase 1: Foundations

### The CRD Hierarchy

The Dash0 Operator exposes three layers of configuration:

```
Dash0OperatorConfiguration (cluster-scoped, one per cluster)
  └─ Dash0Monitoring (namespace-scoped, one per namespace)
       └─ Workloads (Deployments, DaemonSets, StatefulSets, Jobs, CronJobs, Pods)
```

**Dash0OperatorConfiguration** — the global "master config":
- Sets the OTLP/gRPC export target(s), auth, and Dash0 API endpoint
- Cluster-wide feature flags (profiling, Prometheus CRD support, self-monitoring)
- One per cluster; cluster-scoped resource

**Dash0Monitoring** — per-namespace telemetry config:
- Enables traces, metrics, logs, and events for that namespace
- Controls instrumentation mode, log collection, Prometheus scraping
- Can override the global export config for that namespace
- Supports OTTL filters and transforms

**Workloads** — the leaf nodes:
- Controlled by the Dash0Monitoring resource above them
- Per-workload opt-out via `dash0.com/enable: "false"` label

### Beyond Telemetry: GitOps CRDs

The operator also manages four additional CRD types that sync to Dash0 via the REST API:

| CRD | Purpose | Requires |
|---|---|---|
| `PersesDashboard` | Dashboard definitions synced to Dash0 | `apiEndpoint` configured |
| `PrometheusRule` | Alert rules synced as Dash0 check rules | `apiEndpoint` configured |
| `Dash0SyntheticCheck` | Synthetic monitors (uptime, API probes) | `apiEndpoint` configured |
| `Dash0View` | Saved views on traces, logs, resources | `apiEndpoint` configured |

Setting `spec.telemetryCollection.enabled: false` disables all collectors — the operator runs in IaC-only mode, managing CRD sync without any collector footprint.

### Helm Install

**Add the repo:**
```bash
helm repo add dash0-operator https://dash0hq.github.io/dash0-operator
helm repo update dash0-operator
```

**Minimal install (two mandatory values):**
```bash
helm install \
  dash0-operator \
  dash0-operator/dash0-operator \
  --namespace dash0-system \
  --create-namespace \
  --set operator.dash0Export.enabled=true \
  --set operator.dash0Export.endpoint=ingress.eu-west-1.aws.dash0.com:4317 \
  --set operator.dash0Export.token=YOUR_AUTH_TOKEN \
  --set operator.dash0Export.apiEndpoint=https://api.eu-west-1.aws.dash0.com
```

**Endpoint format:** Always `ingress.<region>.<cloud>.dash0.com:4317`. No `https://` prefix required (tolerated but not needed for gRPC).

**Enable a namespace:**
```yaml
apiVersion: operator.dash0.com/v1alpha1
kind: Dash0Monitoring
metadata:
  name: dash0-monitoring
  namespace: my-app
spec:
  instrumentWorkloads:
    mode: created-and-updated
  logCollection:
    enabled: true
```

### Auto-Instrumentation

#### How It Works

When a pod is created or updated in a monitored namespace, the **mutating admission webhook** intercepts the pod spec before Kubernetes stores it. The webhook:

1. Injects a `dash0-instrumentation` init container
2. Mounts a shared volume at `/__otel_auto_instrumentation/`
3. Adds environment variables (`JAVA_TOOL_OPTIONS`, `NODE_OPTIONS`, `CORECLR_ENABLE_PROFILING`, etc.)
4. Sets `OTEL_EXPORTER_OTLP_ENDPOINT` pointing to the on-node DaemonSet collector

The init container copies agent binaries to `/__otel_auto_instrumentation/agents/` before the app container starts.

#### Four Supported Runtimes

| Runtime | Requirement | OTel Agent |
|---|---|---|
| Java | Java 8+ | Upstream OTel Java agent (javaagent.jar) |
| Node.js | Node 16+ | Dash0's Node.js OTel distribution (enhanced upstream SDK) |
| .NET | .NET 6+ | CLR profiler env vars |
| Python | Python 3.9+ (beta, opt-in) | Requires `operator.instrumentation.enablePythonAutoInstrumentation=true` |

#### Opt-In vs Opt-Out

**Opt-out model (default):** `mode: all` or `mode: created-and-updated` — every eligible workload is instrumented unless labeled `dash0.com/enable: "false"`.

**Opt-in model:** Use `labelSelector` in the v1beta1 API — only workloads with matching labels are instrumented:

```yaml
spec:
  instrumentWorkloads:
    mode: created-and-updated
    labelSelector:
      matchLabels:
        dash0.com/enable: "true"
```

#### Three Instrumentation Modes

| Mode | Behavior | When to Use |
|---|---|---|
| `all` | Instruments ALL workloads including existing ones (restarts pods) | Maintenance windows, greenfield |
| `created-and-updated` | Only new or changed workloads — no restarts for existing pods | Safe production rollout |
| `none` | No instrumentation at all | Logs/events only, or regulated services |

---

## 3. Phase 2: Intermediate

### Export Configuration

#### Three Export Types

```yaml
spec:
  exports:
    # Type 1: Dash0 (recommended — enables CRD sync)
    - dash0:
        endpoint: ingress.eu-west-1.aws.dash0.com:4317
        authorization:
          token: your-token           # inline (less secure)
          # OR:
          secretRef:
            name: dash0-secret        # Kubernetes Secret name
            key: token                # Secret key
        dataset: production           # optional, defaults to "default"

    # Type 2: Generic gRPC (any OTLP/gRPC backend)
    - grpc:
        endpoint: tempo.monitoring.svc:4317
        headers:
          - name: Authorization
            value: Bearer grafana-token

    # Type 3: HTTP/OTLP (Loki, Jaeger, custom backends)
    - http:
        endpoint: http://loki.monitoring.svc:3100/otlp
        encoding: proto               # or "json"
```

Multiple exports fan out telemetry to all configured backends simultaneously.

#### Auth: `token` vs `secretRef`

| Method | Storage | Security | When to Use |
|---|---|---|---|
| `token` | Rendered into a Kubernetes ConfigMap | Readable by anyone with cluster API access | Dev/test only |
| `secretRef` | Stored in a Kubernetes Secret | Base64-encoded (not encrypted by default) | Production — pair with KMS for at-rest encryption |

#### Per-Namespace Export Override

A `Dash0Monitoring` resource can **replace** (not merge) the global export for its namespace:

```yaml
apiVersion: operator.dash0.com/v1alpha1
kind: Dash0Monitoring
metadata:
  name: dash0-monitoring
  namespace: team-alpha
spec:
  exports:
    - dash0:
        endpoint: ingress.us-east-1.aws.dash0.com:4317
        authorization:
          secretRef:
            name: team-alpha-dash0-secret
            key: token
```

This enables multi-tenant clusters where each team routes telemetry to their own Dash0 organization.

### Telemetry Pipeline Architecture

#### DaemonSet vs Deployment Collectors

| Collector Type | Deployment | Responsibilities |
|---|---|---|
| DaemonSet collector | One pod per node | Pod logs (filelog receiver from `/var/log/pods/`), node-level metrics, receives traces from workloads on the same node via OTLP |
| Deployment collector | Single (or replicated) deployment | Cluster metrics, Kubernetes API events, cross-node trace aggregation |

The DaemonSet collector stores filelog checkpoint offsets on the node — if the pod restarts, it resumes from where it left off (no duplicate log delivery).

#### OTTL Filters and Transforms

Filters and transforms are configured at the `Dash0Monitoring` level. **Filter executes BEFORE transform.**

```yaml
spec:
  # Drop health check spans + internal metrics
  filter:
    traces:
      span:
        - 'attributes["http.route"] == "/health"'
        - 'attributes["http.route"] == "/ready"'
    metrics:
      datapoint:
        - 'metric.name == "http.server.request.duration" and attributes["http.route"] =~ "/internal/.*"'

  # Enrich spans with a custom attribute
  transform:
    trace_statements:
      - context: span
        statements:
          - 'set(attributes["environment"], "production")'
```

OTTL conditions within a `span:` or `datapoint:` list are **OR'd** — any single match drops the item. Use `&&` within a single expression for AND logic.

**Error mode conflict resolution:** When multiple namespaces specify different `error_mode` values (`propagate`, `ignore`, `silent`), the most severe wins: `propagate > ignore > silent`.

### Prometheus & Perses Integration

#### Three Ways to Collect Prometheus Metrics

**1. Annotation scraping** — no CRDs needed:
```yaml
# On any pod:
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
  prometheus.io/path: "/metrics"
  prometheus.io/scheme: "http"
```
Enable per-namespace: `spec.prometheusScraping.enabled: true` in `Dash0Monitoring`.

**2. Prometheus CRDs** (ServiceMonitor, PodMonitor, ScrapeConfig):
```bash
helm upgrade dash0-operator dash0-operator/dash0-operator \
  --set operator.prometheusCrdSupportEnabled=true \
  --reuse-values
```
Requires the Prometheus CRDs already installed in the cluster. The **OpenTelemetry Target Allocator** is deployed to watch these CRDs and assign scrape targets to the DaemonSet collector on the same node as the monitored workload.

**3. PrometheusRule sync** — requires `apiEndpoint`:
```bash
# Set synchronizePrometheusRules=true (default)
# Operator watches PrometheusRule CRs → calls Dash0 API → creates check rules
```

#### Perses Dashboard Sync

```yaml
apiVersion: perses.dev/v1alpha1
kind: PersesDashboard
metadata:
  name: my-service-dashboard
  namespace: my-app
  # To exclude from sync:
  # labels:
  #   dash0.com/enable: "false"
spec:
  # ... Perses dashboard JSON spec (same format as Dash0 UI export)
```

When `synchronizePersesDashboards: true` (default) and `apiEndpoint` is configured, the operator syncs PersesDashboard CRs to Dash0 dashboards automatically.

---

## 4. Phase 3: Advanced

### Self-Monitoring

The operator sends its own telemetry (metrics, logs, traces about operator health) to the configured Dash0 backend. Enabled by default (`spec.selfMonitoring.enabled: true`).

**Key behavior:** Self-monitoring bypasses `telemetryCollection.enabled` — it works even in IaC-only mode (no collectors deployed). The operator sends directly to the configured export endpoint.

Monitor for: controller reconciliation latency, webhook errors, CRD sync failures.

### Profiling (Opt-In)

Profiling is disabled by default:
```bash
helm upgrade dash0-operator dash0-operator/dash0-operator \
  --set profilingEnabled=true \
  --reuse-values
```

Or via `Dash0OperatorConfiguration`:
```yaml
spec:
  profiling:
    enabled: true
```

### Auto-Namespace Monitoring

For clusters with many namespaces, enable auto-monitoring instead of creating `Dash0Monitoring` resources manually:

```yaml
# In helm values or --set flags:
operator:
  autoMonitorNamespaces:
    enabled: true
  monitoringTemplate:
    spec:
      instrumentWorkloads:
        mode: created-and-updated
      logCollection:
        enabled: true
```

**Always excluded from auto-monitoring:**
- `kube-system`
- `kube-node-lease`
- `kube-public`
- `dash0-system` (operator namespace)

**Opt a namespace out:**
```bash
kubectl label namespace my-namespace dash0.com/enable=false
```

The default `labelSelector` is `dash0.com/enable!=false` — labeling a namespace with `dash0.com/enable=false` causes the operator to skip it. If you delete the auto-created `Dash0Monitoring` without the label, the operator recreates it.

### Resource Attributes (Kubernetes Context Enrichment)

The operator's managed collectors automatically add the following resource attributes to all telemetry:

| Attribute | Source |
|---|---|
| `k8s.namespace.name` | Pod namespace |
| `k8s.pod.name` | Pod name |
| `k8s.pod.uid` | Pod UID |
| `k8s.node.name` | Node where pod runs |
| `k8s.deployment.name` | Owning Deployment (if any) |
| `k8s.replicaset.name` | Owning ReplicaSet |
| `service.name` | Derived from workload name |
| `k8s.pod.label.<name>` | All pod labels (when `collectPodLabelsAndAnnotations.enabled: true`) |
| `k8s.pod.annotation.<name>` | All pod annotations (when enabled) |

### Rate Limiting for Large Rollouts

When instrumenting an existing cluster with many workloads using `mode: all`, stagger pod restarts:

```yaml
operator:
  instrumentation:
    delayAfterEachWorkloadMillis: 500    # 0.5s between each workload
    delayAfterEachNamespaceMillis: 2000  # 2s between namespaces
    debug: true                          # verbose init container logs
```

---

## 5. CRD Reference Table

| CRD | Scope | Key Fields | Purpose |
|---|---|---|---|
| `Dash0OperatorConfiguration` | Cluster | `spec.exports[]`, `spec.selfMonitoring`, `spec.telemetryCollection.enabled`, `spec.prometheusCrdSupport.enabled` | Global operator config; export targets, feature flags |
| `Dash0Monitoring` | Namespace | `spec.instrumentWorkloads.mode`, `spec.logCollection.enabled`, `spec.prometheusScraping.enabled`, `spec.filter`, `spec.transform`, `spec.exports[]` | Per-namespace telemetry control |
| `PersesDashboard` | Namespace | `spec` (Perses dashboard JSON) | Dashboard-as-code; synced to Dash0 via API |
| `PrometheusRule` | Namespace | `spec.groups[].rules[]` (standard Prometheus format) | Alert rule-as-code; synced to Dash0 check rules |
| `Dash0SyntheticCheck` | Namespace | `spec` (Dash0 synthetic check YAML) | Synthetic monitor definitions |
| `Dash0View` | Namespace | `spec` (Dash0 view YAML) | Saved views on traces/logs/resources |

### Dash0OperatorConfiguration Export Type Fields

| Export Type | Required Fields | Optional Fields |
|---|---|---|
| `dash0` | `endpoint`, `authorization.token` OR `authorization.secretRef` | `apiEndpoint`, `dataset` |
| `grpc` | `endpoint` | `headers[]` (name/value pairs) |
| `http` | `endpoint` | `headers[]`, `encoding` (`proto` or `json`) |

### Dash0Monitoring instrumentWorkloads v1alpha1 vs v1beta1

| API Version | Field Type | Supported Fields |
|---|---|---|
| `v1alpha1` | String (`all`/`created-and-updated`/`none`) | Simple mode string only |
| `v1beta1` | Struct | `mode`, `labelSelector` (matchLabels/matchExpressions), `traceContext.propagators` |

---

## 6. Helm Values Reference

### Mandatory Values

| Value | Description | Example |
|---|---|---|
| `operator.dash0Export.endpoint` | OTLP/gRPC ingress URL | `ingress.eu-west-1.aws.dash0.com:4317` |
| `operator.dash0Export.token` | Auth token (or use `secretRef`) | `auth_xxxx` |

### Important Optional Values

| Value | Default | Description |
|---|---|---|
| `operator.dash0Export.enabled` | `false` | Auto-creates `Dash0OperatorConfiguration` on startup |
| `operator.dash0Export.apiEndpoint` | `""` | Dash0 REST API endpoint (`https://api.*.dash0.com`); required for CRD sync |
| `operator.dash0Export.secretRef.name` | `""` | K8s Secret name for auth token |
| `operator.dash0Export.secretRef.key` | `""` | Key within the Secret |
| `operator.dash0Export.dataset` | `"default"` | Target dataset in the Dash0 org |
| `operator.selfMonitoring.enabled` | `true` | Send operator health telemetry to Dash0 |
| `operator.telemetryCollection.enabled` | `true` | Deploy OTel collectors; set `false` for IaC-only mode |
| `operator.prometheusCrdSupportEnabled` | `false` | Enable ServiceMonitor/PodMonitor/ScrapeConfig support |
| `operator.autoMonitorNamespaces.enabled` | `false` | Auto-create `Dash0Monitoring` for every namespace |
| `operator.monitoringTemplate.spec` | `{}` | Template spec for auto-created `Dash0Monitoring` resources |
| `operator.instrumentation.enablePythonAutoInstrumentation` | `false` | Enable Python 3.9+ (beta) auto-instrumentation |
| `operator.instrumentation.delayAfterEachWorkloadMillis` | `0` | Rate-limit delay between workloads during bulk instrumentation |
| `operator.instrumentation.delayAfterEachNamespaceMillis` | `0` | Rate-limit delay between namespaces |
| `operator.instrumentation.debug` | `false` | Verbose init container logging |
| `gke.autopilot.enabled` | `false` | Enable GKE Autopilot compatibility |
| `profilingEnabled` | `false` | Enable OTLP profiling collection |
| `operator.targetAllocator.mTls.enabled` | `false` | Enable mTLS between Target Allocator and collectors |

---

## 7. Environment Variables Injected on Workloads

When the mutating webhook instruments a pod, these environment variables are injected:

| Variable | Runtime | Value |
|---|---|---|
| `JAVA_TOOL_OPTIONS` | Java | `-javaagent:/__otel_auto_instrumentation/agents/javaagent.jar` |
| `NODE_OPTIONS` | Node.js | `--require /__otel_auto_instrumentation/agents/node/dash0-node.js` |
| `CORECLR_ENABLE_PROFILING` | .NET | `1` |
| `CORECLR_PROFILER` | .NET | `{...}` (CLR profiler GUID) |
| `CORECLR_PROFILER_PATH` | .NET | `/__otel_auto_instrumentation/agents/...` |
| `PYTHONPATH` | Python | `/__otel_auto_instrumentation/agents/python/...` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | All | `http://localhost:4317` (DaemonSet collector on same node) |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | Python only | `http/protobuf` |
| `OTEL_PROPAGATORS` | All (if configured) | Value from `spec.instrumentWorkloads.traceContext.propagators` |
| `OTEL_RESOURCE_ATTRIBUTES` | All | Pre-populated with k8s resource attributes |

The init container mounts the shared volume at `/__otel_auto_instrumentation/agents/` before the application container starts. The app container reads agent binaries from this path via the injected env vars.

---

## 8. Scenario Drills

### Scenario 1: Zero-Code Story (Sales)

**Context:** A prospect says, "Our developers are too busy to add instrumentation code to every service."

<details>
<summary>Model Answer</summary>

The Dash0 Operator is built exactly for this. You install it once via Helm into your cluster — a 2-minute operation. Then you apply one YAML file per namespace — 10 lines of config. Every eligible Java, Node.js, .NET, and Python workload in that namespace automatically gets OpenTelemetry traces flowing to Dash0, with no application code changes, no SDK imports, no Dockerfile modifications.

The mechanism is a Kubernetes mutating admission webhook — a standard K8s pattern. When a pod starts, the webhook injects an init container that copies OTel agent binaries onto a shared volume. The application's JVM or Node.js runtime loads the agent automatically via injected environment variables. The application code is completely untouched.

The ask: let us set it up in your dev namespace this afternoon. 30 minutes, and you'll see traces for every service in that namespace without touching a single line of code.

</details>

---

### Scenario 2: Security Team Won't Allow Automatic Pod Modification

**Context:** A prospect's security team rejects anything that automatically modifies pods.

<details>
<summary>Model Answer</summary>

This is a valid concern — and the operator has specific knobs for it.

**Knob one: opt-in mode.** Set `instrumentWorkloads.mode: none` in the `Dash0Monitoring` resource. Nothing is instrumented by default. Then use `labelSelector` in the v1beta1 API — only pods explicitly labeled `dash0.com/enable: "true"` get instrumented. Security gets a per-workload allowlist with no surprises.

**Knob two: RBAC review.** The webhook operates with a tightly scoped service account. Review it: `kubectl get clusterrolebinding -l app.kubernetes.io/name=dash0-operator`. All permissions are standard and documented in the Helm chart.

**Knob three: what the init container actually does.** It copies agent binaries (a JAR, a Node module) to `/__otel_auto_instrumentation/agents/` and sets environment variables (`JAVA_TOOL_OPTIONS`, `NODE_OPTIONS`). No kernel modules, no eBPF, no network proxying. The application process is unmodified.

Offer: share the webhook configuration YAML in advance and schedule a security review session. Most security teams approve after seeing the bounded blast radius.

</details>

---

### Scenario 3: Rolling Out to 200 Microservices

**Context:** A customer wants observability across 200 services without a big-bang cutover.

<details>
<summary>Model Answer</summary>

**Phase 1:** Install the operator without enabling any namespace. `helm install` sets up the operator in `dash0-system`. No webhooks fire, no pods restart. Verify: `kubectl get pods -n dash0-system`.

**Phase 2:** Enable monitoring on dev/staging first. Apply `Dash0Monitoring` with `mode: created-and-updated`. Existing pods are NOT restarted — instrumentation only applies to new or updated deployments.

**Phase 3:** Once confident in dev, apply to one production namespace with `mode: created-and-updated`. Instrumentation rolls in naturally with the next deploy cycle.

**Phase 4:** When ready for a full sweep, switch `mode: all` in a maintenance window. Use the rate-limiting flags:
```yaml
operator:
  instrumentation:
    delayAfterEachWorkloadMillis: 500
    delayAfterEachNamespaceMillis: 2000
```

**Opt-out safety valve:** Label any service `dash0.com/enable: "false"` to exclude it entirely — for high-traffic, sensitive, or externally managed workloads.

Result: 200 services instrumented over 2-4 weeks with no big-bang cutover and full rollback at every phase.

</details>

---

### Scenario 4: Multi-Tenant Cluster (Different Teams, Different Orgs)

**Context:** 20 teams share one cluster. Each wants separate Dash0 billing and ability to opt out.

<details>
<summary>Model Answer</summary>

The Dash0 Operator's multi-tenancy model handles this natively:

**Per-namespace export override:** Each team's `Dash0Monitoring` resource includes an `exports[]` array pointing to their own Dash0 organization endpoint and auth token. Team A's telemetry goes to org A; Team B's to org B. No cross-pollination.

**Opt-out entirely:** A team can set `instrumentWorkloads.mode: none` and `logCollection.enabled: false`. The namespace has a `Dash0Monitoring` but collects nothing.

**RBAC delegation:** Give each team RBAC to create/update `Dash0Monitoring` in their own namespace only. They control their instrumentation settings without touching the cluster-wide operator config.

**Billing separation:** Each team's Dash0 organization handles its own ingestion billing. The platform team pays nothing per-namespace.

**Bootstrap automation:** Enable `autoMonitorNamespaces.enabled: true` with a default template (`mode: created-and-updated`). Teams that need custom settings apply their own `Dash0Monitoring` to replace the auto-created one.

</details>

---

### Scenario 5: OTTL Filter for High-Cardinality Metrics

**Context:** An engineer needs to drop per-pod HTTP metrics with URL paths as dimensions to control cardinality.

<details>
<summary>Model Answer</summary>

Use `spec.filter` in the `Dash0Monitoring` resource:

```yaml
spec:
  filter:
    metrics:
      # Drop datapoints where URL contains a UUID
      datapoint:
        - 'IsMatch(attributes["http.url"], ".*/[0-9a-f]{8}-.*")'

      # Drop entire metric for specific service
      metric:
        - 'name == "http.server.request.duration" and resource.attributes["service.name"] == "payment-service"'
```

Key rules:
- Conditions in the list are **OR'd** — any match drops the item
- Use `&&` within a single expression for AND conditions
- `metric`-level filters drop the entire metric; `datapoint`-level filters drop individual data points

After applying, verify no OTTL errors:
```bash
kubectl logs -n dash0-system daemonset/dash0-operator-opentelemetry-collector | grep -i error
```

Check Dash0 metric cardinality to confirm reduction.

</details>

---

### Scenario 6: Dual Export (Dash0 + Grafana Stack Simultaneously)

**Context:** An engineer needs to send telemetry to both Dash0 AND an existing Grafana/Loki/Tempo stack.

<details>
<summary>Model Answer</summary>

Multi-export is a first-class feature. Configure the `exports[]` array in `Dash0OperatorConfiguration`:

```yaml
apiVersion: operator.dash0.com/v1alpha1
kind: Dash0OperatorConfiguration
spec:
  exports:
    - dash0:
        endpoint: ingress.eu-west-1.aws.dash0.com:4317
        authorization:
          token: your-dash0-token
    - grpc:
        endpoint: tempo.monitoring.svc.cluster.local:4317
    - http:
        endpoint: http://loki.monitoring.svc.cluster.local:3100/otlp
        encoding: proto
```

For per-namespace targeting (only certain namespaces export to Grafana), add `exports[]` to the `Dash0Monitoring` resource for those namespaces.

Caveat: multiple exports increase network egress from collector pods. Monitor collector resource usage:
```bash
kubectl top pods -n dash0-system
```

Note: CRD sync (dashboards, check rules) only works for `dash0` export type — `grpc`/`http` exports receive telemetry data only.

</details>

---

### Scenario 7: PrometheusRule Migration to Dash0

**Context:** A customer wants to migrate their existing Prometheus alerting rules to Dash0 using the operator.

<details>
<summary>Model Answer</summary>

**Step 1:** Enable Prometheus CRD support and set the API endpoint:
```bash
helm upgrade dash0-operator dash0-operator/dash0-operator \
  --set operator.prometheusCrdSupportEnabled=true \
  --set operator.dash0Export.apiEndpoint=https://api.eu-west-1.aws.dash0.com \
  --reuse-values
```

**Step 2:** Verify metrics are flowing (annotation scraping or ServiceMonitors already in use).

**Step 3:** Apply existing `PrometheusRule` resources unchanged:
```bash
kubectl apply -f existing-rules/
```
The operator watches for `PrometheusRule` resources and calls the Dash0 API to create corresponding check rules. Same YAML format they already have.

**Step 4:** Verify sync in Dash0 → Alerting. Check operator logs:
```bash
kubectl logs -n dash0-system deployment/dash0-operator-controller-manager | grep PrometheusRule
```

**Edge case:** To exclude a specific `PrometheusRule` from sync, label it:
```yaml
labels:
  dash0.com/enable: "false"
```

</details>

---

### Scenario 8: Java App Not Being Instrumented

**Context:** A Java app was deployed in a monitored namespace but no traces appear in Dash0.

<details>
<summary>Model Answer</summary>

Work through this checklist in order:

**Step 1: Dash0Monitoring exists?**
```bash
kubectl get dash0monitoring -n <namespace>
```
If absent, create one with `mode: all`.

**Step 2: Workload excluded by label?**
```bash
kubectl get deployment <name> -n <namespace> -o jsonpath='{.metadata.labels}'
```
If `dash0.com/enable: "false"` is present, remove it.

**Step 3: Init container being injected?**
```bash
kubectl describe pod <pod-name> | grep -A5 'Init Containers'
```
Look for `dash0-instrumentation`. If absent, the webhook isn't firing.

**Step 4: Webhook active?**
```bash
kubectl get mutatingwebhookconfigurations | grep dash0
```
Verify the namespace selector matches.

**Step 5: JAVA_TOOL_OPTIONS set?**
```bash
kubectl exec <pod> -- env | grep JAVA_TOOL_OPTIONS
```
Should be `-javaagent:/__otel_auto_instrumentation/agents/javaagent.jar`.

**Step 6: JVM loading the agent?**
```bash
kubectl logs <pod> | grep 'OpenTelemetry'
```
Look for `[otel.javaagent]` startup lines.

**Step 7: Network reachability?**
```bash
kubectl exec <pod> -- curl -v http://dash0-operator-opentelemetry-collector.dash0-system:4318/v1/traces
```
HTTP 200/405 = collector reachable. If blocked, check NetworkPolicy.

</details>

---

## 9. Key Exercises with kubectl/Helm Commands

### Exercise 1: Install and Verify

```bash
# Add Helm repo
helm repo add dash0-operator https://dash0hq.github.io/dash0-operator
helm repo update dash0-operator

# Install operator
helm install dash0-operator dash0-operator/dash0-operator \
  --namespace dash0-system \
  --create-namespace \
  --set operator.dash0Export.enabled=true \
  --set operator.dash0Export.endpoint=ingress.eu-west-1.aws.dash0.com:4317 \
  --set operator.dash0Export.token=YOUR_TOKEN \
  --set operator.dash0Export.apiEndpoint=https://api.eu-west-1.aws.dash0.com

# Verify operator is running
kubectl get pods -n dash0-system
kubectl get dash0operatorconfiguration
kubectl get crds | grep dash0
```

### Exercise 2: Enable a Namespace and Verify Instrumentation

```bash
# Create the monitoring resource
cat <<EOF | kubectl apply -f -
apiVersion: operator.dash0.com/v1alpha1
kind: Dash0Monitoring
metadata:
  name: dash0-monitoring
  namespace: my-app
spec:
  instrumentWorkloads:
    mode: created-and-updated
  logCollection:
    enabled: true
EOF

# Verify a pod gets instrumented (deploy/restart any pod, then):
kubectl describe pod -n my-app <pod-name> | grep -A5 'Init Containers'
kubectl exec -n my-app <pod-name> -- env | grep -E 'JAVA_TOOL_OPTIONS|NODE_OPTIONS|OTEL'
```

### Exercise 3: Configure Dual Export

```bash
# Edit the auto-created operator configuration
kubectl edit dash0operatorconfiguration dash0-operator-configuration-auto-resource

# Add a second export entry under spec.exports:
#   - grpc:
#       endpoint: otel-collector.monitoring.svc:4317

# Verify both collectors receive data
kubectl logs -n dash0-system daemonset/dash0-operator-opentelemetry-collector | tail -50
```

### Exercise 4: Add an OTTL Filter

```bash
kubectl edit dash0monitoring dash0-monitoring -n my-app

# Add under spec:
# filter:
#   traces:
#     span:
#       - 'attributes["http.route"] == "/health"'
#       - 'attributes["http.route"] == "/ready"'

# Verify no OTTL errors
kubectl logs -n dash0-system daemonset/dash0-operator-opentelemetry-collector | grep -i error
```

### Exercise 5: Enable Auto-Namespace Monitoring

```bash
helm upgrade dash0-operator dash0-operator/dash0-operator \
  --set operator.autoMonitorNamespaces.enabled=true \
  --set operator.monitoringTemplate.spec.instrumentWorkloads.mode=created-and-updated \
  --reuse-values

# Verify auto-created resources
kubectl get dash0monitoring -A

# Opt a namespace out
kubectl label namespace kube-monitoring dash0.com/enable=false
```

### Exercise 6: Enable Profiling

```bash
helm upgrade dash0-operator dash0-operator/dash0-operator \
  --set profilingEnabled=true \
  --reuse-values

kubectl get pods -n dash0-system -w
kubectl logs -n dash0-system deployment/dash0-operator-controller-manager | grep -i profil
```

### Exercise 7: Inspect Collector Config

```bash
# See the generated collector configuration
kubectl get configmap -n dash0-system | grep collector
kubectl describe configmap -n dash0-system <collector-configmap-name>

# Monitor collector resource usage
kubectl top pods -n dash0-system

# Live tail collector logs
kubectl logs -n dash0-system daemonset/dash0-operator-opentelemetry-collector -f
```

---

## 10. Competitive Cheat Sheet

### vs. Datadog Operator

| Dimension | Datadog Operator | Dash0 Operator |
|---|---|---|
| Instrumentation agent | Proprietary Datadog tracer | 100% upstream OpenTelemetry agents |
| Lock-in risk | High — leaving means re-instrumenting | None — OTel investment is portable |
| Cost model | Per APM host / per ingested span | Volume-based, no per-host fee |
| Source code | Closed source | Open source on GitHub |
| Signals | APM, metrics, logs (separate agents) | Traces, metrics, logs, profiles from one install |
| Prometheus support | Limited | Annotation scraping + ServiceMonitor/PodMonitor CRDs + PrometheusRule sync |

**Key objection handler:** "If you leave Datadog, you start from zero — their instrumentation is proprietary. The Dash0 Operator injects the same OTel Java agent Datadog's own tracing SDK is built on. If you ever leave Dash0, your instrumentation stays intact."

### vs. Grafana Alloy

| Dimension | Grafana Alloy | Dash0 Operator |
|---|---|---|
| Auto-instrumentation | None — collects existing telemetry only | Mutating webhook injects OTel agents into workloads |
| Zero-code traces | Not supported | Core capability |
| CRD management | None | Dashboards, alert rules, synthetic checks, views |
| Positioning | Great collector, not an operator | Full observability operator |

**Key message:** Alloy and the Dash0 Operator are complementary. Alloy for existing log pipelines; Dash0 Operator for zero-code auto-instrumentation. Many customers run both.

### vs. Dynatrace OneAgent

| Dimension | Dynatrace OneAgent | Dash0 Operator |
|---|---|---|
| Instrumentation approach | Proprietary DaemonSet, injects into every process | Kubernetes-native webhook + OTel init container |
| Protocol | Proprietary | OTLP (open standard) |
| Lock-in | Extremely high | None — OTel protocol, portable |
| Multi-backend | Not supported | Any OTLP-compatible backend via `exports[]` |
| Source visibility | Closed | Open source |

**Key message:** "Dynatrace owns the entire telemetry stack — agent, protocol, backend. The Dash0 Operator uses OpenTelemetry end-to-end. Adding a second backend (Grafana, Honeycomb) is trivial — one extra export entry in the config."

### vs. Manual OTel Setup

| Dimension | Manual OTel | Dash0 Operator |
|---|---|---|
| Workload instrumentation | Per-service SDK setup required | Zero-code, webhook injects automatically |
| Collector management | Hand-crafted YAML, manual updates | CRD-driven, auto-reconciled |
| K8s context enrichment | Manual k8sattributes processor config | Built-in, automatic |
| Operational overhead | High — every config change is a deploy cycle | Low — declarative CRDs |
| GitOps integration | Manual | PersesDashboard, PrometheusRule CRDs |

**Key message:** "Manual OTel works but has high operational overhead. The Dash0 Operator gives you Kubernetes-native lifecycle management — teams get observability without being OTel experts."

---

## 11. Troubleshooting Guide

### No Traces Appearing

| Check | Command | Expected Result |
|---|---|---|
| `Dash0Monitoring` exists | `kubectl get dash0monitoring -n <ns>` | Resource present |
| Workload not excluded | `kubectl get deploy <name> -o yaml \| grep dash0.com/enable` | Label absent or `"true"` |
| Init container injected | `kubectl describe pod <pod> \| grep -A5 'Init Containers'` | `dash0-instrumentation` present |
| Webhook active | `kubectl get mutatingwebhookconfigurations \| grep dash0` | Webhook present |
| OTel env vars set | `kubectl exec <pod> -- env \| grep JAVA_TOOL_OPTIONS` | Agent javaagent path present |
| JVM loading agent | `kubectl logs <pod> \| grep 'otel.javaagent'` | Agent startup lines |
| Collector reachable | `kubectl exec <pod> -- curl -v http://...:4318/v1/traces` | HTTP 200 or 405 |

### No Metrics Appearing

| Check | Command | Expected Result |
|---|---|---|
| Prometheus scraping enabled | `kubectl get dash0monitoring -n <ns> -o yaml \| grep prometheusScraping` | `enabled: true` |
| Annotations present | `kubectl describe pod <pod> \| grep prometheus.io` | `scrape: "true"` annotation |
| Target Allocator running | `kubectl get pods -n dash0-system \| grep target-allocator` | Pod running |
| Target Allocator logs | `kubectl logs -n dash0-system deploy/dash0-operator-target-allocator \| grep scrape` | No errors |

### No Logs Appearing

| Check | Command | Expected Result |
|---|---|---|
| Log collection enabled | `kubectl get dash0monitoring -n <ns> -o yaml \| grep logCollection` | `enabled: true` |
| DaemonSet collector health | `kubectl get pods -n dash0-system -l component=collector` | All pods Running |
| DaemonSet collector logs | `kubectl logs -n dash0-system daemonset/dash0-operator-opentelemetry-collector \| grep -i error` | No errors |
| Node permissions | `kubectl exec -n dash0-system <daemonset-pod> -- ls /var/log/pods` | Readable |

### CRD Sync Not Working (Dashboards/Rules)

| Check | Fix |
|---|---|
| `apiEndpoint` not configured | Add `--set operator.dash0Export.apiEndpoint=https://api.*.dash0.com` |
| Auth token doesn't have API access | Verify token has org-level scope in Dash0 Settings → API Tokens |
| PersesDashboard has `dash0.com/enable: "false"` label | Remove the label |
| Perses CRDs not installed | Install Perses operator CRDs first |

### Auto-Namespace Monitoring Not Creating Resources

| Check | Fix |
|---|---|
| Namespace labeled `dash0.com/enable=false` | Remove the label if you want it monitored |
| Namespace is `kube-system`, `kube-public`, `kube-node-lease`, or `dash0-system` | These are permanently excluded; add `Dash0Monitoring` manually |
| `autoMonitorNamespaces.enabled` is `false` | Enable via Helm upgrade |

### `dash0-operator-configuration-auto-resource` Being Overwritten

**Root cause:** This reserved name is owned by the Helm chart. Manual edits are overwritten on operator restart by design.

**Fix:** Create your `Dash0OperatorConfiguration` with any other name. The operator will not overwrite resources with non-reserved names.

---

## 12. Quick Reference Card

### The Five Most Important Commands

```bash
# 1. Check operator health
kubectl get pods -n dash0-system

# 2. List all monitoring configs
kubectl get dash0monitoring -A

# 3. Check if a workload is instrumented
kubectl describe pod <pod-name> | grep -A5 'Init Containers'

# 4. Tail collector logs
kubectl logs -n dash0-system daemonset/dash0-operator-opentelemetry-collector -f

# 5. Check webhook config
kubectl get mutatingwebhookconfigurations | grep dash0
```

### The Three CRD Labels That Control Everything

| Label | Target | Effect |
|---|---|---|
| `dash0.com/enable: "false"` | Workload (Deployment, etc.) | Disables auto-instrumentation for this workload |
| `dash0.com/enable: "false"` | Namespace | Excludes namespace from auto-namespace monitoring |
| `dash0.com/enable: "false"` | PersesDashboard / PrometheusRule | Excludes resource from CRD sync to Dash0 |

### Instrumentation Mode Decision Tree

```
Need zero disruption to running pods?
  YES → mode: created-and-updated
  NO  → mode: all (restarts pods)

Need to disable instrumentation entirely?
  → mode: none

Need to instrument only specific workloads?
  → mode: created-and-updated + labelSelector.matchLabels
```

### Export Type Decision Tree

```
Sending to Dash0?
  → dash0 export type (also enables CRD sync for dashboards/rules)

Sending to any OTLP/gRPC backend (Tempo, Jaeger, etc.)?
  → grpc export type

Sending to HTTP/OTLP backend (Loki, etc.)?
  → http export type (choose encoding: proto or json)

Multiple backends?
  → Define multiple entries in the exports[] array
```

### Key Numbers

| Value | Number |
|---|---|
| Minimum K8s version | 1.25.16 |
| OTLP/gRPC port | 4317 |
| OTLP/HTTP port | 4318 |
| Operator namespace | `dash0-system` |
| Auto-resource name | `dash0-operator-configuration-auto-resource` |
| Supported runtimes | 4 (Java, Node.js, .NET, Python beta) |
| Init container agent path | `/__otel_auto_instrumentation/agents/` |
| Default dataset | `default` |

---

*Study guide for the Dash0 Operator Crash Course (v1.0) — generated 2026-04-09*
