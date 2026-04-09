# Dash0 MCP Crash Course — Companion Study Guide

**Course:** Dash0 MCP Crash Course (v1.0, PLATFORM_VERSION v3.1)
**Audience:** Sales Reps, Solutions Architects, Engineers / DevOps
**Format:** Self-paced crash course with flashcards, quizzes, drills, and rapid fire

---

## 1. Course Overview & Prerequisites

### What This Course Covers

The Dash0 MCP Crash Course teaches you how to use the Model Context Protocol (MCP) to turn any AI coding assistant into an observability expert with live access to Dash0 data. The course covers all 23 MCP tools, setup across multiple IDEs, real-world investigation workflows, and competitive positioning.

**Signal types accessible via MCP:**
- Logs (structured, with natural language filtering)
- Distributed traces (end-to-end, multi-service)
- Metrics and PromQL expressions
- Synthetic check results and availability history
- Core Web Vitals (LCP, FID, CLS, INP, TTFB)
- Kubernetes events (CrashLoopBackOff, OOMKilled, scheduling failures)
- Dashboard metadata and management

### Course Structure

| Phase | Lessons | Topics |
|---|---|---|
| Phase 1: Foundations | 1–3 | MCP protocol, Dash0 MCP server, setup, service discovery |
| Phase 2: Intermediate | 4–6 | Logs & traces, metrics & PromQL, synthetic & web vitals |
| Phase 3: Advanced | 7–8 | K8s & dashboards, competitive positioning |

### Recommended Study Schedule

| Session | Date | Duration | Topics |
|---|---|---|---|
| 1 | Apr 14 | 30 min | What is MCP? Lesson 1, Fundamentals flashcards |
| 2 | Apr 15 | 30 min | Setup & Service Discovery, Lessons 2–3 |
| 3 | Apr 16 | 45 min | Phase 1 flashcards + category quiz |
| 4 | Apr 21 | 45 min | Logs, Traces & Metrics, Lessons 4–5 |
| 5 | Apr 22 | 45 min | Synthetics, K8s & Scenarios, Lessons 6–7 |
| 6 | Apr 23 | 30 min | Competitive Positioning + Rapid Fire |
| 7 | Apr 24 | 45 min | Final Exam & review of weak categories |

### Prerequisites

- Basic understanding of observability concepts (logs, traces, metrics)
- Familiarity with at least one IDE (Claude Code, Cursor, VS Code, or Windsurf)
- Access to a Dash0 account with a dataset containing instrumented services
- A Dash0 auth token with "All permissions" on the target dataset(s)

### Role-Based Learning Paths

| Role | Focus Areas | Core Lessons |
|---|---|---|
| Sales Rep | Fundamentals, Pitch & Positioning | 1, 3, 8 |
| Solutions Architect | Fundamentals, Setup, Service Discovery, Logs, Metrics, Pitch | 1–6, 8 |
| Engineer / DevOps | All categories — full technical depth | 1–8 |

---

## 2. Phase 1: Foundations

### Lesson 1 — What is MCP?

**Model Context Protocol (MCP)** is an open standard by Anthropic that defines how Large Language Models connect to external tools and data sources. Think of it as USB-C for AI: a universal connector that lets LLMs call tools without custom integration code.

**Before MCP, connecting an LLM to observability required:**
1. Copy-pasting data from monitoring UIs into the chat
2. Writing custom function-calling integrations for each tool
3. Accepting that the LLM would answer from outdated training data

**MCP solves all three** with a standardized tool protocol.

**Key insight:** With MCP, the LLM autonomously decides *which* tool to call and *when* based on your natural language request. You don't call tools — the AI does.

#### The Dash0 MCP Server

The Dash0 MCP server is a **remote, hosted** implementation of the MCP standard. It exposes **23 tools** covering every observability signal in Dash0.

Critically: the server is remote — there's nothing to install locally. One config block in your IDE and the full Dash0 observability dataset is available through natural language.

#### Four Primary Use Cases (from Dash0 docs)

1. **Troubleshooting** — pull error rates, latency, and service health without leaving your IDE
2. **High-level statistics** — service availability summaries, request volumes, error distributions on-demand
3. **Implementation planning** — query dependencies, endpoints, and performance baselines before building new features
4. **Service listing** — retrieve structured service catalogs for architecture discussions or documentation

#### 23 Tools by Category

| Category | Tools |
|---|---|
| Service Discovery | `list_services`, topology tools, service health |
| Logs & Traces | `get_logs`, `get_traces`, correlation tools |
| Metrics & PromQL | `get_metrics`, `query_promql`, Query Builder |
| Synthetic & Web | Synthetic check results, web vitals queries |
| Kubernetes | `get_kubernetes_events` |
| Dashboards | `list_dashboards`, dashboard management |

---

### Lesson 2 — Setup & Configuration

#### Step 1: Get Your Credentials

From the Dash0 app, collect:

1. **MCP Endpoint URL** — Organization Settings → Endpoints → MCP (region-specific)
2. **Auth Token** — Organization Settings → Auth Tokens. Must have "All permissions" on the target dataset(s). Read-only tokens block dashboard creation and synthetic check management.

**Regions:** US West, EU, AP each have a different MCP endpoint URL.

#### Claude Code Configuration

Create `.mcp.json` in the project root (or `~/.claude/mcp.json` globally):

```json
{
  "mcpServers": {
    "dash0": {
      "type": "streamableHttp",
      "url": "https://your-region.mcp.dash0.com",
      "headers": {
        "Authorization": "Bearer YOUR_AUTH_TOKEN"
      }
    }
  }
}
```

Restart Claude Code. The Dash0 server appears in the MCP tools panel.

#### Fallback Config (legacy stdio clients)

Some MCP clients implemented the older stdio transport before Streamable HTTP was standardized. Use `mcp-remote` as a local bridge:

```json
{
  "mcpServers": {
    "dash0": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://your-region.mcp.dash0.com",
               "--header", "Authorization: Bearer ${DASH0_AUTH_TOKEN}"],
      "env": {
        "DASH0_AUTH_TOKEN": "YOUR_AUTH_TOKEN"
      }
    }
  }
}
```

The `mcp-remote` package runs locally as a bridge, connecting the IDE's stdio MCP client to the Dash0 remote HTTP endpoint.

#### Verifying Connection

Ask: `"List all services in Dash0."` — if you see your service catalog, MCP is working.

---

### Lesson 3 — Service Discovery

Service discovery is your starting point for any investigation. The Dash0 MCP tools give you a complete picture of your instrumented service catalog, their relationships, and current health — all through natural language.

#### The `list_services` Tool

Use it to:
- `"Show me all services in Dash0"` — full service catalog
- `"Which services appeared in the last 24 hours?"` — newly deployed services
- `"How many services do we have in production?"` — infrastructure sizing

#### Service Topology & Dependencies

Critical for impact analysis:
- `"What services does checkout-service depend on?"`
- `"Which services have upstream dependencies on payment-service?"`
- `"If database-primary goes down, which services are affected?"`

#### RED Metrics per Service

| Metric | Meaning | Question to Ask |
|---|---|---|
| **R**ate | Requests per second | "What's the request rate for payment-service?" |
| **E**rrors | Error rate % | "What's the error rate for payment-service?" |
| **D**uration | Latency (p50/p95/p99) | "What's the p99 latency for payment-service?" |

**Combined RED query:** `"Show me all services with error rate above 0.5% and p99 latency above 2s in the last hour"`

---

## 3. Phase 2: Intermediate

### Lesson 4 — Logs & Traces

#### `get_logs`: Natural Language Log Queries

The `get_logs` tool accepts natural language and returns structured log records. You don't need to know Dash0's log query syntax, attribute names, or filter operators.

| Traditional | MCP |
|---|---|
| Learn query syntax, attribute paths, filter operators | `"Show me error logs for payment-service in the last hour"` |

**The LLM adds value beyond retrieval:** it summarizes patterns, identifies anomalies, compares with previous time windows, and suggests investigation steps — all in one response.

#### `get_traces`: Distributed Trace Retrieval

Key natural language patterns:
- `"Show me the slowest traces for checkout-service in the last 30 minutes"`
- `"Find all failed traces for payment-service today"`
- `"Show me the full trace for trace ID abc123 including all service hops"`
- `"Which operation in order-service takes the most time on average?"`

#### Log-Trace Correlation

One of MCP's most powerful capabilities. Instead of switching between log and trace views and manually matching trace IDs:

`"Show me logs correlated with trace ID abc123"` — the LLM calls `get_traces` for the trace ID, then `get_logs` filtered by the same trace ID, presenting both in a unified view.

---

### Lesson 5 — Metrics & PromQL

#### Two Ways to Query Metrics

| Tool | Best For |
|---|---|
| `get_metrics` | Natural language metric retrieval — no PromQL needed |
| `query_promql` | Direct PromQL execution — LLM generates the expression from natural language |

**PromQL without the learning curve:** the LLM generates `histogram_quantile` expressions, `rate()` calculations, and multi-label selectors from plain English.

#### Common Metrics Queries

| Natural Language | Generated PromQL |
|---|---|
| "p99 latency for api-gateway" | `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))` |
| "Error rate for payment-service" | `rate(http_requests_total{status=~"5..",service="payment-service"}[5m])` |
| "Request volume last 24h" | `sum(increase(http_requests_total[24h])) by (service)` |
| "Memory usage by pod" | `container_memory_usage_bytes` grouped by pod |

#### The Query Builder Capability

An interface for constructing and executing PromQL queries directly through MCP for real-time analysis. Supports iterative refinement: `"That query is too broad — can you filter it to only the production namespace?"` — the LLM refines the PromQL and re-runs it.

---

### Lesson 6 — Synthetic & Web Vitals

#### Synthetic Monitoring via MCP

Synthetic checks are automated tests that periodically probe your endpoints — HTTP checks, API tests, multi-step browser flows.

Key natural language queries:
- `"Show me the current status of all synthetic checks"`
- `"Which synthetic checks have been failing intermittently in the last 4 hours?"`
- `"What is the average response time for the /checkout endpoint synthetic check?"`
- `"Is the payment gateway URL check passing right now?"`

**Key value:** availability status embedded in your IDE workflow — no dashboard context switch. The LLM explains *why* a check might be failing, not just that it is.

#### Core Web Vitals

| Metric | Measures | Good Threshold |
|---|---|---|
| **LCP** | Largest Contentful Paint — loading performance | < 2.5s |
| **CLS** | Cumulative Layout Shift — visual stability | < 0.1 |
| **INP** | Interaction to Next Paint — responsiveness | < 200ms |
| **TTFB** | Time to First Byte — server response | < 800ms |
| **FID** | First Input Delay — input latency | < 100ms |

**Deployment correlation:** `"Did the LCP score for the homepage worsen after the 2pm deployment today?"` — the LLM queries web vitals time series and correlates with the deployment timestamp automatically.

---

## 4. Phase 3: Advanced

### Lesson 7 — Kubernetes & Dashboards

#### `get_kubernetes_events`

K8s events are the first place to look when application behavior is abnormal.

Key natural language queries:
- `"Which pods are in CrashLoopBackOff right now?"`
- `"Show me OOMKilled events from the last hour"`
- `"Were there any scheduling failures in the production namespace today?"`
- `"Show me all Kubernetes events for the payment-service pods in the last 30 minutes"`

**K8s + telemetry correlation:** `"Did the OOMKill at 3:42pm cause the error spike in checkout-service?"` — the LLM calls both `get_kubernetes_events` and `get_metrics` and connects the dots.

#### Post-Deploy Health Validation

High-value automated workflow for CI/CD:

```
"The payment-service was just deployed at 14:32. Check for the following in the 
15 minutes following deployment:
(1) error rate increase above 1%
(2) p99 latency increase above 20%
(3) any Kubernetes restarts
Report HEALTHY or DEGRADED."
```

The LLM autonomously calls `query_promql` and `get_kubernetes_events` and returns a structured health assessment — enabling automated deployment gates.

#### `list_dashboards`

Returns all dashboards in your Dash0 organization:
- `"Which dashboards do we have for the payment service?"`
- `"List all dashboards created in the last month"`
- `"Is there a SLO dashboard for checkout-service?"`

---

### Lesson 8 — Competitive Positioning

#### The AI-Native Observability Story

Most observability tools are adding AI features. But these are AI features bolted onto existing tools. MCP is different: it's an open protocol that brings observability data into the AI tools engineers already use.

**Instead of observability tools getting smarter, your AI coding assistant gains observability superpowers.**

#### Core Value Statement

> "Dash0 MCP turns your AI coding assistant into an observability expert — with live access to logs, traces, metrics, and Kubernetes events, all in plain English."

#### MECE Argument: MCP Over Traditional Observability UI

1. **No context switching** — stay in your IDE
2. **No syntax learning** — natural language replaces query languages
3. **Intelligent analysis** — the LLM interprets data, doesn't just return it
4. **Composable** — combine multiple signals in one response

---

## 5. All 23 MCP Tools Reference Table

| # | Tool Name | Category | Description | Key Parameters |
|---|---|---|---|---|
| 1 | `list_services` | Service Discovery | Lists all services instrumented in Dash0 with metadata | dataset, last-seen filter |
| 2 | `get_service_health` | Service Discovery | Returns RED metrics (Rate, Errors, Duration) for a specific service | service name, time window |
| 3 | `get_service_topology` | Service Discovery | Returns upstream/downstream dependency graph for a service | service name |
| 4 | `get_service_dependencies` | Service Discovery | Lists services that call or are called by a given service | service name, direction |
| 5 | `get_logs` | Logs & Traces | Retrieves log records with natural language filtering | service, time range, severity, text filter |
| 6 | `get_traces` | Logs & Traces | Retrieves distributed traces with span details | service, time range, duration threshold, trace ID |
| 7 | `get_trace_by_id` | Logs & Traces | Retrieves a specific trace by trace ID including all service hops | trace ID |
| 8 | `correlate_logs_traces` | Logs & Traces | Returns logs and trace data correlated by trace ID | trace ID |
| 9 | `get_metrics` | Metrics & PromQL | Retrieves time series metric data in natural language | service, metric name, time window |
| 10 | `query_promql` | Metrics & PromQL | Executes a PromQL expression directly against Dash0 | PromQL expression, time range, step |
| 11 | `list_metric_names` | Metrics & PromQL | Lists available metric names for a service or dataset | service name, prefix filter |
| 12 | `get_metric_labels` | Metrics & PromQL | Returns available label names and values for a metric | metric name |
| 13 | `get_synthetic_checks` | Synthetic & Web | Lists all synthetic checks with status and metadata | dataset, status filter |
| 14 | `get_synthetic_check_history` | Synthetic & Web | Returns historical pass/fail results for a synthetic check | check name, time range |
| 15 | `get_web_vitals` | Synthetic & Web | Retrieves Core Web Vitals (LCP, CLS, INP, TTFB, FID) | page URL, time range |
| 16 | `get_availability` | Synthetic & Web | Returns uptime/availability percentage for an endpoint | endpoint name, time range |
| 17 | `get_kubernetes_events` | Kubernetes | Returns K8s events including OOMKilled, CrashLoopBackOff, scheduling failures | namespace, event type, time range |
| 18 | `get_kubernetes_resources` | Kubernetes | Returns K8s resource state (pods, deployments, nodes) correlated with telemetry | resource type, namespace |
| 19 | `list_dashboards` | Dashboards | Lists all dashboards in the Dash0 organization with names, IDs, metadata | dataset |
| 20 | `get_dashboard` | Dashboards | Retrieves a specific dashboard's definition | dashboard ID |
| 21 | `create_dashboard` | Dashboards | Creates a new dashboard (requires full-permission token) | dashboard name, panel definitions |
| 22 | `update_dashboard` | Dashboards | Updates an existing dashboard (requires full-permission token) | dashboard ID, updated definition |
| 23 | `triage` | Service Discovery | Navigates from a symptom to root cause by correlating service health, resource state, and failed health checks | incident description, affected services |

> **Note:** Tool names above represent the functional categories based on course content. Exact API tool names may vary — verify current tool names in the Dash0 MCP server documentation at [dash0.com/docs/dash0/ai/mcp](https://www.dash0.com/docs/dash0/ai/mcp).

---

## 6. Setup Guides for Each Supported IDE

### Claude Code

```json
{
  "mcpServers": {
    "dash0": {
      "type": "streamableHttp",
      "url": "https://your-region.mcp.dash0.com",
      "headers": {
        "Authorization": "Bearer YOUR_AUTH_TOKEN"
      }
    }
  }
}
```

**File location:** `.mcp.json` in project root OR `~/.claude/mcp.json` globally
**Verify:** MCP tools panel in Claude Code sidebar should list the 23 Dash0 tools

---

### Cursor

In Cursor Settings → MCP Servers, add:

```json
{
  "mcpServers": {
    "dash0": {
      "type": "streamableHttp",
      "url": "https://your-region.mcp.dash0.com",
      "headers": {
        "Authorization": "Bearer YOUR_AUTH_TOKEN"
      }
    }
  }
}
```

**Verify:** Cursor Composer panel should show available MCP tools

---

### Windsurf

In Windsurf settings (MCP section), add:

```json
{
  "mcpServers": {
    "dash0": {
      "type": "streamableHttp",
      "url": "https://your-region.mcp.dash0.com",
      "headers": {
        "Authorization": "Bearer YOUR_AUTH_TOKEN"
      }
    }
  }
}
```

---

### VS Code (GitHub Copilot)

In `.vscode/settings.json` (workspace) or user-level settings:

```json
{
  "github.copilot.mcp.servers": {
    "dash0": {
      "type": "streamableHttp",
      "url": "https://your-region.mcp.dash0.com",
      "headers": {
        "Authorization": "Bearer YOUR_AUTH_TOKEN"
      }
    }
  }
}
```

---

### Cline

In Cline's MCP settings panel (Claude extension for VS Code), add the Dash0 server with the streamableHttp transport type, endpoint URL, and Authorization Bearer header.

---

### Zed

In Zed's `settings.json` under the `context_servers` key, add the Dash0 MCP configuration with the appropriate transport type and auth header.

---

### Legacy / Custom Clients (mcp-remote bridge)

For any client that doesn't support Streamable HTTP transport natively:

```json
{
  "mcpServers": {
    "dash0": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://your-region.mcp.dash0.com",
               "--header", "Authorization: Bearer ${DASH0_AUTH_TOKEN}"],
      "env": {
        "DASH0_AUTH_TOKEN": "YOUR_AUTH_TOKEN"
      }
    }
  }
}
```

`mcp-remote` translates stdio transport to the Dash0 remote Streamable HTTP endpoint. Supports LangChain agents, custom Python frameworks, and any tool using the older MCP stdio spec.

---

### Enterprise / Team Rollout Pattern

1. **Platform team** creates auth tokens — read-only for devs, full-permission for SREs
2. **Distribute config** — share the `.mcp.json` template with env vars for tokens
3. **Validate** — each engineer runs `"List all services in Dash0"` to confirm connection
4. **Enablement** — share this study guide's Natural Language Query Patterns section

> Full IDE instructions with copy-paste configs: [dash0.com/hub/integrations?tag=AI](https://www.dash0.com/hub/integrations?tag=AI)

---

## 7. Natural Language Query Patterns

### Service Discovery

| Natural Language Prompt | Tools Invoked | Use Case |
|---|---|---|
| "Show me all services in Dash0" | `list_services` | Service catalog overview |
| "Which services appeared in the last 24 hours?" | `list_services` + timestamp filter | New deployment detection |
| "What services does checkout-service depend on?" | `get_service_topology` | Dependency mapping |
| "Which services have upstream dependencies on payment-service?" | `get_service_dependencies` | Impact analysis |
| "Show me all services with error rate above 1% in the last hour" | `get_service_health` (all) | Triage prioritization |

### Logs

| Natural Language Prompt | Tools Invoked | Use Case |
|---|---|---|
| "Show me error logs for payment-service in the last hour" | `get_logs` | Error investigation |
| "Show me the most recent 50 error logs for checkout-service" | `get_logs` | Bounded log retrieval |
| "Find all logs containing 'connection pool exhausted' today" | `get_logs` | Text-based log search |
| "Show me WARN and ERROR logs across all services in the last 15 minutes" | `get_logs` | Incident triage |

### Traces

| Natural Language Prompt | Tools Invoked | Use Case |
|---|---|---|
| "Find traces for checkout-service that took longer than 2 seconds" | `get_traces` | Latency investigation |
| "Show me failed traces for payment-service today" | `get_traces` | Error trace investigation |
| "Show me the full trace for trace ID abc123 including all service hops" | `get_trace_by_id` | End-to-end trace view |
| "Show me logs correlated with trace ID abc123" | `get_traces` + `get_logs` | Log-trace correlation |

### Metrics & PromQL

| Natural Language Prompt | Tools Invoked | Use Case |
|---|---|---|
| "What is the p99 latency for api-gateway over the last hour?" | `query_promql` (histogram_quantile) | Latency percentile |
| "Compare the error rate for payment-service vs. checkout-service" | `query_promql` (x2) | Multi-service comparison |
| "Give me request volume, error count, and p95 latency for order-service in the last 24 hours" | `get_metrics` + `query_promql` | Health summary |
| "Show me memory usage by pod in the production namespace" | `query_promql` | Resource monitoring |

### Kubernetes

| Natural Language Prompt | Tools Invoked | Use Case |
|---|---|---|
| "Which pods are in CrashLoopBackOff right now?" | `get_kubernetes_events` | Incident triage |
| "Show me OOMKilled events from the last hour and correlate with memory metrics" | `get_kubernetes_events` + `query_promql` | Root cause analysis |
| "Were there any scheduling failures in the production namespace today?" | `get_kubernetes_events` | Capacity investigation |
| "Did the 3pm deploy to payment-service cause any K8s restarts?" | `get_kubernetes_events` + `query_promql` | Post-deploy validation |

### Synthetic & Web Vitals

| Natural Language Prompt | Tools Invoked | Use Case |
|---|---|---|
| "Show me the current status of all synthetic checks" | `get_synthetic_checks` | Availability overview |
| "Which synthetic checks have failed in the last 4 hours?" | `get_synthetic_check_history` | Availability investigation |
| "Did the LCP score for the homepage worsen after the 2pm deployment?" | `get_web_vitals` | Web performance / deploy correlation |
| "Is checkout.example.com currently passing its synthetic check?" | `get_synthetic_checks` | Single-endpoint check |

---

## 8. MCP vs. CLI vs. API Decision Matrix

| Criterion | MCP | CLI | API |
|---|---|---|---|
| **Primary user** | Engineers in AI-assisted IDEs | DevOps / CI/CD automation | Custom integration builders |
| **Interface** | Natural language in your IDE | Terminal commands | HTTP requests + JSON |
| **Context switching** | None — stays in IDE | Low — terminal tab | Low — code/scripts |
| **Learning curve** | Minimal — describe what you want | Medium — memorize command flags | High — learn endpoints, params, auth |
| **Query power** | Full observability dataset, LLM-generated PromQL | Structured queries, scripted | Maximum — full API surface |
| **Best for** | Real-time investigation during development | Automation, GitOps, CI/CD pipelines | Custom dashboards, data pipelines, integrations |
| **PromQL support** | Yes — LLM generates expressions | Yes — explicit flags | Yes — raw endpoint |
| **Dashboard creation** | Yes — natural language description | Yes — YAML/JSON input | Yes — full CRUD |
| **Incident investigation** | Excellent — multi-signal in one thread | Good — scriptable | Not ideal for ad-hoc use |
| **Team onboarding** | 1 config file, no training needed | Requires CLI docs/training | Requires dev resources |

### When to Use Which

**Use MCP when:**
- Investigating issues during development
- Debugging in real time without leaving your IDE
- Engineers unfamiliar with PromQL need metric insights
- You want the LLM to correlate multiple signals and explain findings

**Use CLI when:**
- Automated health checks in CI/CD pipelines
- GitOps workflows managing Dash0 assets
- Scripted queries run on a schedule
- Terminal-native workflows (no AI agent needed)

**Use API when:**
- Building custom integrations (internal tools, data pipelines)
- Bulk operations (migrating dashboards, mass alert updates)
- Embedding Dash0 data into third-party systems
- Maximum flexibility and control over payloads

---

## 9. Incident Investigation Workflows Using MCP

### Standard Incident Investigation (5-Step Pattern)

This pattern applies to any service degradation alert. All steps run in your IDE via natural language.

**Step 1 — Establish the timeline**
```
"Show me the error rate and latency for [service] over the last 2 hours.
When did the degradation start?"
```
*Tools invoked:* `query_promql` — returns time series, LLM identifies the inflection point

**Step 2 — Get recent error logs**
```
"Show me error logs for [service] from the last 30 minutes."
```
*Tools invoked:* `get_logs` — returns top errors with timestamps, messages, stack traces

**Step 3 — Identify bottleneck operations**
```
"Show me the slowest traces for [service] in the last 30 minutes.
What operation is taking the longest?"
```
*Tools invoked:* `get_traces` — returns duration-sorted traces, LLM names the bottleneck

**Step 4 — Check service dependencies**
```
"Are any of the services that [service] calls also showing elevated errors?"
```
*Tools invoked:* `get_service_topology` + `get_service_health` — checks upstream services

**Step 5 — Check Kubernetes events**
```
"Are there any Kubernetes events for pods running [service] in the last hour?"
```
*Tools invoked:* `get_kubernetes_events` — surfaces restarts, OOMKills, scheduling failures

**Total time:** ~2–4 minutes. Zero UI navigation. Zero PromQL written. All in the IDE.

---

### Post-Deployment Validation Pattern

Run automatically after every deployment (as an AI agent in CI/CD or manually in IDE):

```
"[Service] was just deployed at [timestamp] with version [version].
Check for the following in the 15 minutes following deployment:
(1) error rate increase above 1%
(2) p99 latency increase above 20%
(3) any Kubernetes restarts or OOMKilled events
Report status as HEALTHY or DEGRADED with evidence."
```

---

### Cross-Window Incident Analysis

For post-incident reviews:

```
"Compare the following for [service]:
(1) error rate and p99 latency during the 2 hours before the incident (10pm–midnight)
(2) during the incident (midnight–2am)
(3) the 2 hours after resolution (2am–4am)
Summarize what changed and when it normalized."
```

*Tools invoked:* `query_promql` with time range offsets — LLM synthesizes comparative summary

---

### Cascading Failure Investigation

When an alert points to a symptom, not a root cause:

```
"checkout-service is showing elevated errors. Check:
- What services does checkout-service depend on?
- Are any of those dependencies also degraded?
- Which K8s events correlate with the start of the degradation?"
```

---

## 10. Scenario Drills with Model Answers

---

**Drill 1 — Sales objection: Datadog AI assistant**

*Scenario:* A prospect says: "We already have Datadog with an AI assistant built in. Why would we need Dash0 MCP?"

<details>
<summary>Model Answer</summary>

Highlight the open protocol advantage. Datadog's AI assistant is proprietary and locked to their UI — it doesn't live in your IDE. Engineers have to switch to Datadog, ask a question, then return to their code.

Dash0 MCP is built on the Model Context Protocol open standard. It works natively in Claude Code, Cursor, Windsurf, Cline, and any MCP-compatible tool your engineers already use.

**The workflow difference is significant:** with Datadog's AI, engineers context-switch to the Datadog UI. With Dash0 MCP, they never leave their IDE. They ask in Claude Code: *"What's causing the error spike in checkout-service?"* and get a live answer — no copy-paste, no context switch.

Dash0 MCP also exposes all 23 tools including PromQL execution, Kubernetes event correlation, and synthetic check queries. That's a full observability interface, not just a dashboard search feature.

**Closing question:** "How often does your team stay in Datadog's interface versus coding? If observability lived in their IDE, would they use it more?"

</details>

---

**Drill 2 — Security objection: data access scope**

*Scenario:* A customer's security team asks what data the MCP server can access before approving it.

<details>
<summary>Model Answer</summary>

The Dash0 MCP server accesses only your Dash0 observability dataset: logs, distributed traces, metrics (including PromQL queries), synthetic check results, web vitals, Kubernetes events, and Dash0 asset metadata (dashboards, synthetic check definitions).

It does **not** have access to: your application infrastructure, cloud provider APIs, databases, code repositories, or any system outside of Dash0.

Access is scoped to specific datasets via the auth token you configure. You create a token with "All permissions" on the datasets you choose — nothing more. Want read-only access to logs and traces but no dashboard modification? Use a token with restricted dataset permissions.

The server is remote-hosted on Dash0's infrastructure. The auth token is passed as an HTTP header on each request — it's never stored on the developer's machine beyond the IDE config file.

**Recommended approach for the security team:** Start with a read-heavy dataset in a non-production environment and validate the access scope before granting production dataset permissions.

</details>

---

**Drill 3 — "Why not just use the UI and CLI?"**

*Scenario:* A prospect asks: "Why should we use MCP instead of just giving engineers access to the Dash0 UI and the Dash0 CLI?"

<details>
<summary>Model Answer</summary>

MCP, the CLI, and the UI serve different contexts — the answer is "use all three," but MCP fills a gap the others don't.

The **CLI** is great for scripted workflows, CI/CD pipelines, and GitOps operations. The **UI** is best for exploration, building dashboards, and visual investigation. But both require deliberate context switching — you stop coding and go to another tool.

**MCP is the zero-context-switch option.** When an engineer is in Claude Code debugging a function and notices something odd in the logs, they can ask *"What's the error rate for this service?"* without opening a browser or running a terminal command. The answer comes back in the same conversation thread.

The bigger picture: as AI coding agents become standard in engineering workflows, tools that integrate with those agents gain compounding value. MCP is how observability enters the AI coding loop — your AI assistant can reference real production data when helping debug or design code.

**Closing question:** "When was the last time an engineer stopped coding to check observability data because it was too much friction to switch tools?"

</details>

---

**Drill 4 — Live incident demo: payment-service degradation**

*Scenario:* Walk through debugging a service degradation in payment-service using only natural language in Claude Code.

<details>
<summary>Model Answer (Full Demo Script)</summary>

**Step 1:** *"Show me the error rate and latency for payment-service over the last 2 hours. Has anything changed in the last 30 minutes?"*
→ LLM calls `query_promql`, returns time series: "Latency started rising at 11:42pm, currently 7.3s p99, baseline is 340ms"

**Step 2:** *"Show me error logs for payment-service from the last 30 minutes."*
→ LLM calls `get_logs`, returns: "Connection pool exhausted — upstream: database-primary"

**Step 3:** *"Show me the slowest traces for payment-service in the last 30 minutes — what operation is taking the longest?"*
→ LLM calls `get_traces` with duration sort, identifies the bottleneck operation

**Step 4:** *"Are any of the services that payment-service calls also showing elevated errors?"*
→ LLM calls `get_service_topology` + `get_service_health`, checks upstream services

**Step 5:** *"Are there any Kubernetes events for pods running payment-service in the last hour?"*
→ LLM calls `get_kubernetes_events`, returns: "OOMKilled on database-primary-0 at 11:41pm, restart took 90s"

**Root cause identified:** OOMKill on database-primary caused connection pool exhaustion cascading to payment-service.

**Total time:** ~2 minutes. Zero UI navigation. Zero PromQL written. All in the IDE.

</details>

---

**Drill 5 — CI/CD post-deployment health check**

*Scenario:* A customer asks: "Can we use MCP to automate post-deployment health checks in our CI/CD pipeline?"

<details>
<summary>Model Answer</summary>

Yes — this is one of the highest-value production use cases. The pattern:

After deployment completes in your CI/CD pipeline (GitHub Actions, GitLab CI, etc.), trigger a step that invokes an AI agent (e.g., Claude via the Anthropic API) with Dash0 MCP configured. Agent prompt:

*"The payment-service was just deployed at {TIMESTAMP} with version {VERSION}. Check for the following in the 15 minutes following deployment: (1) error rate increase above 1%, (2) p99 latency increase above 20%, (3) any Kubernetes restarts or OOMKilled events. Report status as HEALTHY or DEGRADED with evidence."*

The agent autonomously calls `get_metrics`, `query_promql`, and `get_kubernetes_events` through MCP and returns a structured health assessment. Gate the deployment promotion or trigger a Slack notification based on the result.

This is fundamentally different from custom health check scripts — you describe the check in natural language, the LLM figures out which MCP tools to call, and the result is actionable.

**Prerequisites:** MCP server configured, Dash0 auth token with appropriate dataset permissions, Claude API key or similar LLM with MCP support.

</details>

---

**Drill 6 — Enterprise governance at scale**

*Scenario:* A prospect at a large enterprise says: "We have 500 engineers. How do we govern who can access observability data through MCP?"

<details>
<summary>Model Answer</summary>

Governance at scale is straightforward — MCP uses token-based access control that maps directly to standard least-privilege principles:

**Token strategy:**
- Read-only tokens for developers (logs, traces, metrics queries only)
- Full-permission tokens for SREs and senior engineers (includes dashboard management and synthetic check operations)
- Separate tokens for production and staging datasets — most developers only need staging access

**Token distribution:** Engineers add the token to their IDE config once. Since it's in a local config file (not shared), the blast radius of a compromised token is limited to one person's dataset access.

**Auditability:** All MCP tool calls go through the Dash0 API with the auth token — Dash0's audit log captures who accessed what data. Same audit trail as direct API access.

**Practical answer:** MCP has the same governance model as direct API access — because that's what it is under the hood. The MCP layer doesn't bypass any access controls; it calls the same Dash0 APIs with the same token permissions.

**Recommended rollout for strict environments:** Start with read-only staging access for all engineers. Escalate to production access through a standard request process.

</details>

---

**Drill 7 — Custom agent framework integration**

*Scenario:* A solutions architect customer asks: "Can we embed Dash0 MCP in our custom internal AI tooling — not Claude Code, but our own internal agent framework?"

<details>
<summary>Model Answer</summary>

Yes — because Dash0 MCP is built on the open MCP standard with Streamable HTTP transport, any MCP-compatible client can connect to it.

**Option 1 — Native MCP support:** Add the Dash0 endpoint as a remote MCP server in your framework's MCP client configuration. Use `streamableHttp` transport type, the regional endpoint URL, and an Authorization Bearer header. Your agents automatically discover and call all 23 Dash0 tools.

**Option 2 — mcp-remote bridge:** For older frameworks using stdio or HTTP+SSE transport, `npx mcp-remote` translates to the Dash0 Streamable HTTP endpoint.

**Option 3 — Direct HTTP:** The Streamable HTTP transport is standard HTTP POST. Call the Dash0 MCP endpoint directly with the MCP protocol JSON payload and your auth token. Useful for lightweight integrations that don't need a full MCP client library.

**Example use case:** A custom internal "ops assistant" built on LangChain or a proprietary LLM framework can add Dash0 as a tool provider. The agent then gains natural language access to logs, traces, metrics, and K8s events — integrated into whatever internal workflow the team has built.

**Key point:** MCP's open protocol design specifically enables this ecosystem use case. Dash0 invested in remote hosting precisely so any agent — not just Claude Code — can access observability data without additional infrastructure.

</details>

---

## 11. Competitive Cheat Sheet

### Dash0 MCP vs. Datadog AI Analyst

| Dimension | Datadog AI Analyst | Dash0 MCP |
|---|---|---|
| Protocol | Proprietary | Open standard (MCP) |
| Where it runs | Datadog UI only | Any MCP-compatible IDE |
| Context switching | Required — must open Datadog UI | None — stays in IDE |
| IDE integration | None | Claude Code, Cursor, Windsurf, VS Code + Copilot, Cline, Zed |
| Data access | Live (Datadog data only) | Live (full Dash0 dataset) |
| PromQL support | Yes (within Datadog) | Yes, LLM-generated from natural language |
| K8s events | Yes | Yes |
| Open source | No | GitHub: github.com/dash0hq/mcp-dash0 |

**Differentiating question:** "Does Datadog's AI assistant live inside your IDE, or does it require opening the Datadog UI?"

---

### Dash0 MCP vs. Grafana + ChatGPT

| Dimension | Grafana + ChatGPT | Dash0 MCP |
|---|---|---|
| Integration | Disconnected — manual copy-paste | Integrated — LLM queries live data directly |
| Data freshness | Whatever you copy-paste | Real-time |
| Context | Two separate tools, two separate windows | Single conversation in your IDE |
| Intelligence | ChatGPT interprets whatever you paste | LLM queries, retrieves, and interprets in one step |
| Setup | Two separate tools, no official integration | One config block |

**One-liner:** "Grafana + ChatGPT is like asking a colleague for help but only being allowed to show them a printout. Dash0 MCP gives them live screen access."

---

### Competitive Positioning Summary

**Open protocol advantage:**
Dash0 MCP is built on the MCP open standard. It works in any MCP-compatible tool your engineers already use. Proprietary AI observability assistants require you to go to their tool. Dash0 MCP comes to your tool.

**Zero install friction:**
The server is remote-hosted. No npm packages, no local binary, no version management. One config block scales to 500 engineers.

**Full signal coverage:**
Logs, traces, metrics (PromQL), synthetics, web vitals, K8s events, and dashboards — all accessible through natural language in a single conversation.

**Handling security objections:**
- Data scope is bounded to Dash0 observability data only
- Access is governed by token permissions (same as direct API access)
- No write access to application infrastructure
- Audit log captures all MCP tool calls

---

## 12. Quick Reference Card

### The 5 Most-Used Tools

| Tool | When to Use |
|---|---|
| `list_services` | Start of any investigation — see what's in the environment |
| `get_logs` | Any error or anomaly — pull structured log records |
| `get_traces` | Latency or timeout issues — find the bottleneck operation |
| `query_promql` | Metric trends, percentiles, comparisons |
| `get_kubernetes_events` | Pod crashes, restarts, OOMKills |

### Setup Checklist

- [ ] Get MCP endpoint URL from Org Settings → Endpoints → MCP
- [ ] Create auth token with "All permissions" on target dataset(s)
- [ ] Add `.mcp.json` to project root (or globally in `~/.claude/mcp.json`)
- [ ] Restart IDE
- [ ] Test: ask `"List all services in Dash0"` — verify service catalog appears
- [ ] Try: `"Show me error logs for [service] in the last hour"` — confirm log access

### Key Facts to Memorize

- MCP = Model Context Protocol (open standard by Anthropic)
- Dash0 MCP server exposes **23 tools**
- Transport: **Streamable HTTP** (remote hosted, no local install)
- Endpoint URL: Org Settings → Endpoints → MCP
- Auth token: needs **"All permissions"** on target datasets
- Config key (Claude Code): `mcpServers`
- Fallback for legacy clients: `npx mcp-remote`
- RED = Rate, Errors, Duration

### Natural Language Formula

```
[Action] + [Subject] + [Time Range] + [Optional Filter]

Examples:
"Show me error logs for payment-service in the last hour"
"Compare the p99 latency for checkout-service vs. payment-service over the last 2 hours"
"Which pods are in CrashLoopBackOff right now?"
"List all services with error rate above 1% in the last 30 minutes"
```

### Core Value Statement (memorize this)

> "Dash0 MCP turns your AI coding assistant into an observability expert — with live access to logs, traces, metrics, and Kubernetes events, all in plain English."

---

## References

- [Dash0 MCP Documentation](https://www.dash0.com/docs/dash0/ai/mcp)
- [Dash0 MCP GitHub Repository](https://github.com/dash0hq/mcp-dash0)
- [Dash0 MCP Changelog](https://www.dash0.com/changelog/dash0-mcp-server)
- [Dash0 Integrations Hub (per-IDE configs)](https://www.dash0.com/hub/integrations?tag=AI)
- [Model Context Protocol Specification](https://modelcontextprotocol.io/introduction)
- [Dash0 MCP Use Cases](https://www.dash0.com/docs/dash0/ai/mcp/use-mcp)
