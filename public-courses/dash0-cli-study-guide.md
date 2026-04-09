# Dash0 CLI — Companion Study Guide

**Course:** Dash0 CLI Crash Course (`dash0-cli-crash-course.html`)
**Audience:** Sales Reps, Solutions Architects, Engineers / DevOps
**Format:** Self-paced, 3-phase curriculum over ~3 weeks
**Related courses:** Dash0 API Crash Course · Dash0 MCP Crash Course

---

## Course Overview & Prerequisites

### What This Course Covers

The Dash0 CLI is the terminal-native interface to the Dash0 observability platform. This course teaches:

- **Installation and configuration** — Homebrew, Docker, source builds, profiles
- **Asset management CRUD** — dashboards, check rules, views, synthetic checks
- **GitOps apply workflow** — idempotent `dash0 apply`, dry-run, CI/CD integration
- **Telemetry querying** — logs, spans, traces, PromQL metrics
- **Sending telemetry** — OTLP log/span injection, deployment annotations
- **Agent mode** — AI coding agent auto-detection, machine-readable output
- **Competitive positioning** — Dash0 vs Datadog CLI, Grafana Grizzly, Dynatrace Monaco

### Prerequisites

| Prerequisite | Why It Matters |
|---|---|
| Basic terminal / shell familiarity | All CLI commands run in a terminal |
| Understanding of YAML | Asset definitions are YAML files |
| Familiarity with OpenTelemetry concepts (logs, spans, metrics) | Telemetry querying and sending use OTel attribute paths |
| A Dash0 account with an API token | Required for the hands-on exercises |
| Docker (optional, for CI/CD labs) | Used in the GitHub Actions exercise |

### Learner Roles

The course offers three role-specific tracks:

| Role | Focus | Core Lessons |
|---|---|---|
| **Sales Rep** | Pitch points, competitive positioning, objection handling | Lessons 1, 3, 8 |
| **Solutions Architect** | Architecture, demo workflows, customer Q&A | Lessons 1–6, 8 |
| **Engineer / DevOps** | Full hands-on: commands, YAML, CI/CD, troubleshooting | All lessons (1–8) |

---

## Phase 1: Foundations

*Estimated time: ~75 minutes across 3 lessons*

### Lesson 1 — What is the Dash0 CLI?

#### The Three Dash0 Interfaces

| Interface | User | Use Case |
|---|---|---|
| **UI** | Human (browser) | Exploration, visualization, building dashboards interactively |
| **API** | Machine (HTTP REST) | Programmatic integrations, custom tooling, webhook handlers |
| **CLI** | Human (terminal) / AI agent | Daily operations, GitOps, querying, scripting |

The CLI wraps the same HTTP API the UI uses, exposing it as ergonomic shell commands. It also bridges the CLI and AI agent worlds via **agent mode** — more on this in Phase 3.

#### Four CLI Capability Areas

```
1. Asset Management    — CRUD for dashboards, check rules, views, synthetic checks
2. GitOps Apply        — dash0 apply -f (idempotent, like kubectl apply)
3. Telemetry Querying  — logs, spans, traces, PromQL metrics from the terminal
4. Sending Telemetry   — OTLP log/span injection for pipeline testing and CI/CD annotations
```

#### Key Facts to Memorize

- Open source: `github.com/dash0hq/dash0-cli`
- Config stored in `~/.dash0/` (override with `DASH0_CONFIG_DIR`)
- Output formats: `table` (default), `wide`, `json`, `yaml`, `csv` — via `-o` flag
- Experimental commands require `-X` flag placed immediately after `dash0`

---

### Lesson 2 — Installation & Profiles

#### Installation Methods

**Homebrew (recommended for local use)**
```bash
brew tap dash0hq/dash0-cli https://github.com/dash0hq/dash0-cli
brew install dash0
```

**Docker (CI/CD and containerized environments)**
```bash
docker run ghcr.io/dash0hq/cli:latest [command]
# Supports linux/amd64 and linux/arm64
```

**Build from source (Go 1.22+ required)**
```bash
git clone https://github.com/dash0hq/dash0-cli.git
cd dash0-cli && make install
```

#### Profile Management

Each profile stores three values: **API URL**, **OTLP ingress URL**, and **auth token**.

```bash
# Create a profile
dash0 config profiles create dev \
  --api-url https://api.us-west-2.aws.dash0.com \
  --otlp-url https://ingress.us-west-2.aws.dash0.com \
  --auth-token auth_xxx

# List profiles
dash0 config profiles list

# Switch active profile
dash0 config profiles select staging

# Inspect active config (token is partially masked)
dash0 config show

# Delete a profile
dash0 config profiles delete old-profile
```

#### Environment Variables for CI/CD

```bash
DASH0_CONFIG_DIR=/tmp/dash0-ci   # Override config directory (essential in CI)
DASH0_AGENT_MODE=true            # Enable machine-readable JSON output
DASH0_AUTH_TOKEN=auth_xxx        # Override auth token without editing profile
```

> **Key insight:** In CI/CD pipelines, always set `DASH0_CONFIG_DIR` to a writable temp directory (e.g., `/tmp/dash0-ci`). The home directory may be read-only or shared between pipeline runs.

---

### Lesson 3 — Asset Management CRUD

#### Four Asset Types, One Pattern

Every asset type (`dashboards`, `check-rules`, `views`, `synthetic-checks`) exposes the same five subcommands:

| Subcommand | Description |
|---|---|
| `list` | List all assets of this type |
| `get <id>` | Fetch a single asset by ID |
| `create -f <file>` | Create a new asset from a YAML file |
| `update [id] -f <file>` | Update an existing asset (ID from CLI or YAML) |
| `delete <id>` | Delete an asset (use `--force` in scripts) |

#### Core Command Examples

```bash
# List all dashboards
dash0 dashboards list

# Export a dashboard as YAML for version control
dash0 dashboards get <id> -o yaml > dashboard.yaml

# Create a check rule from a YAML file
dash0 check-rules create -f rule.yaml

# Update a view (ID read from YAML if omitted on CLI)
dash0 views update -f view.yaml

# Delete a synthetic check without confirmation prompt
dash0 synthetic-checks delete <id> --force
```

#### The Edit-Round-Trip Pattern

```bash
# Step 1: Export current state
dash0 dashboards get <id> -o yaml > my-dashboard.yaml

# Step 2: Edit locally
vim my-dashboard.yaml

# Step 3: Apply changes (idempotent)
dash0 apply -f my-dashboard.yaml
```

#### Prometheus CRD Import

The CLI accepts **PrometheusRule CRD files** directly and converts alerting rules to Dash0 check rules automatically — no manual translation required:

```bash
dash0 check-rules create -f prometheus-rules.yaml   # PrometheusRule CRD
```

This is a major migration accelerator for teams moving from Prometheus/Alertmanager to Dash0.

---

## Phase 2: Intermediate

*Estimated time: ~75 minutes across 3 lessons*

### Lesson 4 — GitOps with `dash0 apply`

#### Why `dash0 apply`?

`dash0 apply` is the **idempotent, GitOps-native** command for deploying Dash0 assets. Think of it as `kubectl apply` for observability config. Run it ten times with the same input — the result is always the same as running it once.

| Command | Behavior | When to Use |
|---|---|---|
| `dash0 dashboards create -f` | Always creates; **fails** if resource exists | Creating new resources explicitly |
| `dash0 dashboards update -f` | Always updates; **fails** if resource does not exist | Updating known existing resources |
| `dash0 apply -f` | Creates **or** updates; never fails on pre-existence | GitOps CI/CD pipelines |

#### Core Apply Patterns

```bash
# Apply a single file
dash0 apply -f dashboard.yaml

# Apply all YAML files in a directory (recursive)
dash0 apply -f observability/

# Dry-run — validate without applying (use on PRs)
dash0 apply -f observability/ --dry-run

# Pipe from stdin (dynamic generation)
cat generated-dashboard.yaml | dash0 apply -f -

# Multi-document YAML (multiple resources separated by ---)
dash0 apply -f all-resources.yaml
```

#### Supported `kind` Values in YAML

`dash0 apply` infers the resource type from the `kind` field:

- `Dashboard`
- `PersesDashboard`
- `CheckRule`
- `SyntheticCheck`
- `View`

#### Recommended GitOps Workflow

```
PR opened  →  dash0 apply --dry-run  (validate, no changes)
PR merged  →  dash0 apply            (deploy to target environment)
Rollback   →  git revert + re-merge  (re-run pipeline with previous YAML)
```

Store YAML under `observability/` in the same repo as application code. Changes go through code review — treating observability config with the same rigor as infrastructure.

---

### Lesson 5 — Querying Telemetry

#### Four Query Commands

| Signal | Command | GA? |
|---|---|---|
| Logs | `dash0 -X logs query` | Experimental (`-X` required) |
| Spans | `dash0 -X spans query` | Experimental (`-X` required) |
| Traces | `dash0 -X traces get <trace-id>` | Experimental (`-X` required) |
| Metrics | `dash0 metrics instant --query 'promql'` | Generally available (no `-X`) |

#### Log Query Flags

```bash
# Time range (relative timestamps)
dash0 -X logs query --from now-1h --to now
dash0 -X logs query --from now-30m          # --to defaults to now

# Filter by OTel attributes (multiple filters are ANDed)
dash0 -X logs query \
  --from now-1h \
  --filter "service.name is checkout-service" \
  --filter "severity.text is ERROR"

# Limit results
dash0 -X logs query --limit 100

# CSV export with selected columns
dash0 -X logs query -o csv \
  --column time --column service.name --column body

# JSON for scripting
dash0 -X logs query -o json
```

#### Span and Trace Queries

```bash
# Query spans
dash0 -X spans query \
  --from now-2h \
  --filter "service.name is api-gateway"

# Get all spans for a trace ID
dash0 -X traces get <trace-id>

# Widen the search window for older traces
dash0 -X traces get <trace-id> --from now-6h

# Follow span links across service boundaries
dash0 -X traces get <trace-id> --follow-span-links
```

> **`--follow-span-links` explained:** In async messaging, batch jobs, and fan-out patterns, related work may have different trace IDs connected only by span links. This flag traverses those links to retrieve the full picture of distributed work.

#### PromQL Metrics Queries

```bash
# Instant query (current value)
dash0 metrics instant --query 'sum(rate(http_requests_total[5m]))'

# Range query (time series)
dash0 metrics range \
  --query 'avg(cpu_usage_seconds_total)' \
  --from now-1h --to now --step 5m
```

---

### Lesson 6 — Sending Telemetry

#### Why Send Telemetry from the CLI?

Three practical scenarios:

1. **Pipeline validation** — confirm data arrives with correct attributes before instrumenting production
2. **Deployment annotations** — create searchable deployment markers in Dash0 logs
3. **Local dev testing** — validate OTel routing without needing a running instrumented service

#### Sending Logs

```bash
# Basic log send
dash0 logs send "Application started"

# With resource attributes (describe the entity producing the data)
dash0 logs send "Payment processed" \
  --resource-attribute service.name=payment-service \
  --resource-attribute service.version=2.1.0 \
  --resource-attribute deployment.environment=production

# With log attributes (describe the individual event)
dash0 logs send "Order created" \
  --resource-attribute service.name=order-service \
  --log-attribute order.id=12345 \
  --log-attribute customer.tier=premium

# With OTel severity (set both text and number for full compatibility)
dash0 logs send "DB connection failed" \
  --resource-attribute service.name=api-gateway \
  --severity-text ERROR \
  --severity-number 17
```

#### OTel Severity Number Reference

| Level | Text | Number |
|---|---|---|
| TRACE | TRACE | 1 |
| DEBUG | DEBUG | 5 |
| INFO | INFO | 9 |
| WARN | WARN | 13 |
| ERROR | ERROR | 17 |
| FATAL | FATAL | 21 |

Always set **both** `--severity-text` and `--severity-number` for maximum compatibility with log viewers that filter on either field.

#### Sending Spans (Experimental)

```bash
# Basic span
dash0 -X spans send \
  --name "GET /api/users" \
  --kind SERVER \
  --status-code OK \
  --duration 150ms \
  --resource-attribute service.name=api-gateway

# With span attributes
dash0 -X spans send \
  --name "process-payment" \
  --kind INTERNAL \
  --status-code OK \
  --duration 80ms \
  --resource-attribute service.name=payment-service \
  --span-attribute payment.method=credit_card \
  --span-attribute payment.amount=99.99
```

#### Supported Span Kinds

`CLIENT` · `SERVER` · `PRODUCER` · `CONSUMER` · `INTERNAL`

These match the standard OpenTelemetry `SpanKind` enum.

#### `--resource-attribute` vs `--log-attribute`

| Flag | Attaches to | Example Values |
|---|---|---|
| `--resource-attribute` | Resource (the entity) | `service.name`, `host.name`, `cloud.region` |
| `--log-attribute` | Individual log record | `order.id`, `http.method`, `customer.tier` |

Using the correct level matters for cardinality and querying efficiency.

---

## Phase 3: Advanced

*Estimated time: ~65 minutes across 2 lessons*

### Lesson 7 — Agent Mode & CI/CD

#### What Agent Mode Changes

When active, agent mode makes the CLI machine-readable by changing five behaviors:

| Behavior | Normal Mode | Agent Mode |
|---|---|---|
| Default output | Human-readable table | JSON |
| `--help` | Human-readable text | Structured JSON with flags and subcommands |
| Errors | Human-readable message to stderr | `{"error":"...","hint":"..."}` JSON on stderr |
| Confirmation prompts | Interactive prompts | Auto-confirmed (no prompt) |
| ANSI colors | Colored output | Plain text (no color codes) |

#### Enabling Agent Mode

```bash
# Explicit flag on any command
dash0 --agent-mode dashboards list

# Environment variable (recommended for CI/CD pipelines)
DASH0_AGENT_MODE=true dash0 apply -f observability/

# Disable explicitly (if auto-detected incorrectly)
DASH0_AGENT_MODE=false dash0 dashboards list
```

#### Auto-Detection in AI Coding Environments

The CLI **automatically activates agent mode** when it detects any of these environments — no configuration required:

- Claude Code
- Cursor
- Windsurf
- Cline
- Aider
- GitHub Copilot
- OpenAI Codex
- Any MCP server session (detected via `MCP_SESSION_ID`)

#### Structured Error Format in Agent Mode

```json
{
  "error": "profile not found",
  "hint": "run dash0 config profiles create"
}
```

The `hint` field gives actionable recovery steps, allowing AI agents to self-recover without human intervention.

#### CI/CD Pipeline Pattern (GitHub Actions)

```yaml
name: Deploy Observability Config
on:
  push:
    branches: [main]
    paths: ['observability/**']

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Create Dash0 profile
        env:
          DASH0_AUTH_TOKEN: ${{ secrets.DASH0_AUTH_TOKEN }}
          DASH0_API_URL: ${{ secrets.DASH0_API_URL }}
          DASH0_OTLP_URL: ${{ secrets.DASH0_OTLP_URL }}
        run: |
          export DASH0_CONFIG_DIR=/tmp/dash0-ci
          docker run -e DASH0_CONFIG_DIR -v /tmp/dash0-ci:/tmp/dash0-ci \
            ghcr.io/dash0hq/cli:latest config profiles create ci \
            --api-url $DASH0_API_URL \
            --otlp-url $DASH0_OTLP_URL \
            --auth-token $DASH0_AUTH_TOKEN

      - name: Deploy observability assets
        run: |
          export DASH0_AGENT_MODE=true
          export DASH0_CONFIG_DIR=/tmp/dash0-ci
          docker run -e DASH0_AGENT_MODE -e DASH0_CONFIG_DIR \
            -v /tmp/dash0-ci:/tmp/dash0-ci \
            -v $(pwd)/observability:/observability \
            ghcr.io/dash0hq/cli:latest apply -f /observability/

      - name: Annotate deployment in Dash0
        run: |
          export DASH0_CONFIG_DIR=/tmp/dash0-ci
          docker run -e DASH0_CONFIG_DIR -v /tmp/dash0-ci:/tmp/dash0-ci \
            ghcr.io/dash0hq/cli:latest logs send "Deployment complete" \
            --resource-attribute service.name=ci-pipeline \
            --log-attribute deployment.sha=${{ github.sha }} \
            --log-attribute deployment.env=production \
            --severity-text INFO --severity-number 9
```

---

### Lesson 8 — Competitive Positioning

#### Dash0 CLI vs Competitors

| | Datadog CLI | Grafana (grizzly) | Dynatrace Monaco | **Dash0 CLI** |
|---|---|---|---|---|
| **Asset management** | Process wrapping only | Grafana dashboards only | YAML config (own format) | Dashboards + check rules + views + synthetics |
| **Telemetry querying** | No | No | No | Logs, spans, traces, PromQL |
| **Telemetry injection** | trace injection only | No | No | Logs + spans via OTLP |
| **AI agent mode** | No | No | No | Yes — auto-detects 8+ environments |
| **Prometheus CRD import** | No | No | No | Yes — automatic conversion |
| **GitOps / idempotent apply** | No | Partial | Yes (complex) | Yes — `kubectl apply` semantics |
| **First-party support** | Yes | Community | Yes | Yes |

#### Four Dash0 CLI Differentiators

1. **Unified tool** — Asset management + telemetry querying + telemetry injection in one binary. No competitor covers all three capabilities.
2. **GitOps-native** — `dash0 apply` was designed for CI/CD from day one: dry-run, multi-doc YAML, idempotent operations.
3. **AI agent mode** — Auto-detects 8+ AI coding environments. Structured JSON help and error hints. First-mover in AI-native observability tooling.
4. **Prometheus compatibility** — Accepts PrometheusRule CRDs directly, automatic conversion eliminates rewrites when migrating from Prometheus alerting.

#### Sales Objection Handling

| Objection | Response |
|---|---|
| "We're UI-only" | Start with backup/audit value: `dash0 dashboards get -o yaml` as a safety net requires zero CLI adoption day-to-day. For growing teams, GitOps prevents dashboard drift. |
| "We use Terraform" | Complementary, not competing. Terraform provisions Dash0 environments (datasets, integrations). CLI manages assets within those environments and provides live telemetry access — capabilities Terraform can't offer. |
| "We already have kubectl" | Same `apply` semantics, different layer. `kubectl` deploys workloads; `dash0 apply` deploys the observability config alongside those workloads. Engineers already know the mental model. |
| "Grafana CLI manages our dashboards" | Grafana's CLI (grizzly) is limited to dashboards and is community-maintained. Dash0 CLI covers dashboards + check rules + synthetics + views, adds live querying and telemetry injection, and has first-party support with native AI agent mode. |

---

## Command Reference Table

### Config & Profile Commands

| Command | Synopsis |
|---|---|
| `dash0 config profiles create <name>` | Create a new named profile with `--api-url`, `--otlp-url`, `--auth-token` |
| `dash0 config profiles list` | List all profiles and show the active one |
| `dash0 config profiles select <name>` | Switch the active profile |
| `dash0 config profiles delete <name>` | Delete a profile |
| `dash0 config show` | Display the active profile's config (token partially masked) |

### Asset Management Commands

| Command | Synopsis |
|---|---|
| `dash0 dashboards list` | List all dashboards in the active profile |
| `dash0 dashboards get <id> [-o yaml\|json]` | Fetch a single dashboard; use `-o yaml` to export |
| `dash0 dashboards create -f <file>` | Create a dashboard from a YAML file |
| `dash0 dashboards update [id] -f <file>` | Update a dashboard (ID from CLI or YAML) |
| `dash0 dashboards delete <id> [--force]` | Delete a dashboard |
| `dash0 check-rules list` | List all check (alerting) rules |
| `dash0 check-rules get <id>` | Fetch a single check rule |
| `dash0 check-rules create -f <file>` | Create a check rule (accepts PrometheusRule CRD) |
| `dash0 check-rules update [id] -f <file>` | Update a check rule |
| `dash0 check-rules delete <id> [--force]` | Delete a check rule |
| `dash0 views list` | List all views |
| `dash0 views get <id>` | Fetch a single view |
| `dash0 views create -f <file>` | Create a view |
| `dash0 views update [id] -f <file>` | Update a view |
| `dash0 views delete <id> [--force]` | Delete a view |
| `dash0 synthetic-checks list` | List all synthetic checks |
| `dash0 synthetic-checks get <id>` | Fetch a single synthetic check |
| `dash0 synthetic-checks create -f <file>` | Create a synthetic check |
| `dash0 synthetic-checks update [id] -f <file>` | Update a synthetic check |
| `dash0 synthetic-checks delete <id> [--force]` | Delete a synthetic check |

### GitOps Apply Commands

| Command | Synopsis |
|---|---|
| `dash0 apply -f <file\|dir>` | Idempotent create-or-update from a YAML file or directory |
| `dash0 apply -f <dir> --dry-run` | Validate YAML schema and preview changes without applying |
| `dash0 apply -f -` | Read from stdin (pipe from template generator) |

### Querying Commands

| Command | Synopsis |
|---|---|
| `dash0 -X logs query` | Query logs (experimental) |
| `dash0 -X logs query --from <ts> --to <ts>` | Filter by time range (relative: `now-1h`, absolute: ISO 8601) |
| `dash0 -X logs query --filter "<attr> is <val>"` | Filter by OTel attribute (multiple `--filter` flags are ANDed) |
| `dash0 -X logs query --limit <n>` | Cap number of results returned |
| `dash0 -X logs query -o csv --column <attr>` | Output as CSV with selected attribute columns |
| `dash0 -X spans query` | Query spans — same flags as logs query (experimental) |
| `dash0 -X traces get <trace-id>` | Retrieve all spans for a trace ID (experimental) |
| `dash0 -X traces get <id> --follow-span-links` | Also retrieve linked traces across service boundaries |
| `dash0 metrics instant --query '<promql>'` | Execute a PromQL instant query (generally available) |
| `dash0 metrics range --query '<promql>'` | Execute a PromQL range query with `--from`, `--to`, `--step` |

### Sending Telemetry Commands

| Command | Synopsis |
|---|---|
| `dash0 logs send "<message>"` | Send a log entry via OTLP to the active profile's ingress URL |
| `dash0 logs send ... --resource-attribute k=v` | Attach a resource-level attribute (entity: service.name, host.name) |
| `dash0 logs send ... --log-attribute k=v` | Attach a log-record-level attribute (event: order.id, http.status) |
| `dash0 logs send ... --severity-text <level>` | Set log severity text (INFO, WARN, ERROR, etc.) |
| `dash0 logs send ... --severity-number <n>` | Set log severity number (9=INFO, 13=WARN, 17=ERROR) |
| `dash0 -X spans send --name <name>` | Send a span via OTLP (experimental) |
| `dash0 -X spans send ... --kind <kind>` | Set span kind: CLIENT, SERVER, PRODUCER, CONSUMER, INTERNAL |
| `dash0 -X spans send ... --status-code <code>` | Set span status: OK, ERROR, UNSET |
| `dash0 -X spans send ... --duration <dur>` | Set span duration (e.g., `150ms`, `2s`) |
| `dash0 -X spans send ... --span-attribute k=v` | Attach a span-level attribute |

---

## Scenario Drills with Model Answers

### Scenario 1 — Sales: The Datadog UI Prospect

> **Setup:** A prospect says: "We already manage everything through the Datadog UI. Why would we need a CLI?"

<details>
<summary>Model answer (click to reveal)</summary>

Start with the **"snowflake dashboard" problem**: when everything lives in a UI, dashboards drift between environments, changes aren't reviewed, and rollbacks require manual recreation. The Dash0 CLI solves this by treating observability config as code — export dashboards as YAML, commit to Git, and deploy with `dash0 apply` in CI/CD.

**Distinguish from Datadog CLI:** Datadog does have a CLI (dogwrap, ddtrace), but it focuses on process wrapping and trace injection — not managing the observability layer itself. There's no `datadog apply -f dashboards/` equivalent. Dash0 CLI covers the full asset lifecycle.

**Close with the 2am scenario:** Where does their team go when an alert fires? If they're clicking through the Datadog UI to find the right dashboard, that's friction. Dash0 CLI lets engineers query logs, spans, and metrics directly from the terminal — and it integrates natively with AI coding agents for the next generation of on-call workflows.

</details>

---

### Scenario 2 — Sales: The Terraform / Infrastructure-as-Code Team

> **Setup:** A prospect asks: "How is Dash0 CLI different from Terraform or Pulumi for managing dashboards?"

<details>
<summary>Model answer (click to reveal)</summary>

**Position as complementary, not competing.** Teams use Terraform to provision Dash0 environments (datasets, integrations), and Dash0 CLI to manage observability assets within those environments.

**Key distinctions:**
- Terraform uses HCL and a provider abstraction layer; Dash0 CLI works directly with the native YAML format — export, edit, re-apply, no translation layer, no state file overhead.
- `terraform plan` is for infrastructure state. `dash0 apply --dry-run` is for observability asset validation — lighter weight for the day-2 asset management workflow.
- Dash0 CLI is also a **live operations tool**: query logs, run PromQL, send test spans. Terraform can't tell you why your service is erroring at 3am. Dash0 CLI can.

**Recommended pitch:** "Terraform for environment setup, Dash0 CLI for everything you do after the environment exists."

</details>

---

### Scenario 3 — Sales: The UI-Only Team

> **Setup:** A customer says: "Our team doesn't use command-line tools — UI only. This isn't relevant to us."

<details>
<summary>Model answer (click to reveal)</summary>

**Don't push the full GitOps story yet.** Start with the simpler value:

**Backup and safety net:** `dash0 dashboards get <id> -o yaml` before any change gives an instant snapshot. If someone makes a breaking change or accidentally deletes a dashboard, recovery is trivial. This safety net requires zero CLI adoption day-to-day.

**Scale argument:** As teams grow, manual UI management breaks down — multiple environments, multiple contributors, no audit trail. Ask: "What happens when you have dashboards in dev, staging, and prod that have drifted from each other?"

**AI workflow bridge:** The CLI has first-class support for AI coding agents (Claude Code, Cursor, Windsurf). As those workflows mature, the CLI becomes the natural observability integration point — even for teams that never open a terminal themselves.

</details>

---

### Scenario 4 — SA: Live Dashboard-as-Code Demo

> **Setup:** A customer wants to see a live dashboard-as-code workflow demo.

<details>
<summary>Model answer (click to reveal)</summary>

**Tell the "before" story first:** A dashboard exists in the UI, created manually with no version history. What happens when someone changes a threshold? No audit trail, no rollback.

**Three-step demo loop:**

1. **Export:** `dash0 dashboards list` → find the ID → `dash0 dashboards get <id> -o yaml > my-service-dashboard.yaml`. Open the file — show it's readable YAML. Make a small visible change (add a panel title, adjust a threshold).

2. **Apply:** `dash0 apply -f my-service-dashboard.yaml`. Show the dashboard updated in the UI in real time. Run the same command again — show idempotency (no error, no duplicate).

3. **CI/CD angle:** Show a GitHub Actions snippet:
   - PR → `dash0 apply --dry-run` (validates, no changes)
   - Merge → `dash0 apply` (deploys the change)
   
   Ask: "When was the last time a dashboard change caused an alert to miss a real incident? With this workflow, every change is auditable and reversible."

</details>

---

### Scenario 5 — SA: Prometheus Migration Demo

> **Setup:** A customer asks: "Can you show me how to migrate our existing Prometheus alert rules to Dash0?"

<details>
<summary>Model answer (click to reveal)</summary>

**Lead with zero-rewrite migration:** Take an existing PrometheusRule CRD YAML from their repo and run:

```bash
dash0 check-rules create -f prometheus-rules.yaml
```

The CLI auto-detects the PrometheusRule format and converts alerting rules to Dash0 check rules — no translation, no rewriting. Show the resulting check rules in the Dash0 UI.

**Key message:** "You don't lose your investment in alert rule authoring. Your existing rules carry over directly."

**Follow-up value:** From there, those rules can be managed as code — `dash0 check-rules list`, export to YAML, commit to Git, deploy changes through CI/CD. That's a better workflow than most teams have with Prometheus today (where rules live in ConfigMaps with no review process).

</details>

---

### Scenario 6 — Engineering: CI/CD Pipeline Setup

> **Setup:** An engineer needs to set up a pipeline that auto-deploys Dash0 observability config on merge to main.

<details>
<summary>Model answer (click to reveal)</summary>

**Four-stage pipeline:**

**Stage 1 — PR validation:**
```bash
DASH0_CONFIG_DIR=/tmp/dash0-ci dash0 apply -f observability/ --dry-run
```

**Stage 2 — Profile setup (each run, from secrets):**
```bash
export DASH0_CONFIG_DIR=/tmp/dash0-ci
dash0 config profiles create ci \
  --api-url $DASH0_API_URL \
  --otlp-url $DASH0_OTLP_URL \
  --auth-token $DASH0_AUTH_TOKEN
```

**Stage 3 — Deploy on merge:**
```bash
DASH0_AGENT_MODE=true dash0 apply -f observability/
```

**Stage 4 — Annotate deployment:**
```bash
dash0 logs send "Observability config deployed" \
  --resource-attribute service.name=ci-pipeline \
  --log-attribute deployment.sha=$GITHUB_SHA \
  --log-attribute deployment.env=production \
  --severity-text INFO --severity-number 9
```

**Critical details:** Never rely on a persistent profile file in CI — create it from secrets each run. Set `DASH0_CONFIG_DIR` to a temp path to avoid permission issues. Use `DASH0_AGENT_MODE=true` for clean JSON output in pipeline logs.

</details>

---

### Scenario 7 — Engineering: Debugging `dash0 apply` That Succeeds But Doesn't Seem to Change Anything

> **Setup:** An engineer runs `dash0 apply -f dashboard.yaml` — the command exits 0 but the dashboard in the UI looks unchanged.

<details>
<summary>Model answer (click to reveal)</summary>

**Work through in this order:**

1. **Verify you're targeting the right environment:**
   ```bash
   dash0 config show
   ```
   Confirm the API URL matches where you expect the change. A common mistake: running apply against the dev profile while looking at the production UI.

2. **Verify the resource ID:** The `id` field in the YAML must match the existing resource. Run:
   ```bash
   dash0 dashboards list -o json
   ```
   Compare the IDs. If the YAML has a wrong or missing ID, `apply` creates a new resource (you may have a duplicate you haven't found yet).

3. **Check the API response:** Run with `-o json` to inspect the response. If the API returned 200 with the updated resource, the issue is UI caching — hard-refresh the browser.

4. **Isolate with explicit update:**
   ```bash
   dash0 dashboards update <id> -f dashboard.yaml
   ```
   If this also appears to do nothing, the problem is in the YAML itself.

5. **Validate YAML schema:**
   ```bash
   dash0 apply -f dashboard.yaml --dry-run
   ```
   Look for warnings about malformed panels or dropped fields.

6. **Diff against current state:**
   ```bash
   dash0 dashboards get <id> -o yaml > current.yaml
   diff current.yaml dashboard.yaml
   ```
   Apply incrementally based on what actually differs.

</details>

---

### Scenario 8 — Engineering: Testing the Full OTLP Ingestion Pipeline

> **Setup:** An engineer wants to validate their OpenTelemetry pipeline end-to-end without deploying a full instrumented service.

<details>
<summary>Model answer (click to reveal)</summary>

**Step 1 — Send a test log:**
```bash
dash0 logs send "Pipeline validation test" \
  --resource-attribute service.name=pipeline-test \
  --resource-attribute service.version=1.0.0 \
  --resource-attribute deployment.environment=staging \
  --log-attribute test.run.id=abc123 \
  --severity-text INFO \
  --severity-number 9
```

**Step 2 — Query for it immediately:**
```bash
dash0 -X logs query \
  --from now-2m \
  --filter "service.name is pipeline-test" \
  --filter "test.run.id is abc123"
```

If the log appears with all attributes correctly mapped, the ingestion pipeline is healthy.

**Step 3 — Send a test span:**
```bash
dash0 -X spans send \
  --name "GET /validate" \
  --kind SERVER \
  --status-code OK \
  --duration 100ms \
  --resource-attribute service.name=pipeline-test
```

Verify the span appears in Dash0 traces view.

**Troubleshooting:** If telemetry doesn't appear, run `dash0 config show` to confirm the **OTLP URL** is set (it is separate from the API URL — the ingress endpoint, not the management API).

</details>

---

## Key Exercises with Step-by-Step Guides

### Exercise 1 — Install the CLI and Create Your First Profile

**Goal:** Get the CLI installed and connected to a Dash0 environment.

**Steps:**
1. Install via Homebrew:
   ```bash
   brew tap dash0hq/dash0-cli https://github.com/dash0hq/dash0-cli
   brew install dash0
   ```
2. Obtain your credentials from the Dash0 UI: Settings → Auth Tokens. Note your API URL and OTLP ingress URL.
3. Create a profile:
   ```bash
   dash0 config profiles create dev \
     --api-url https://api.us-west-2.aws.dash0.com \
     --otlp-url https://ingress.us-west-2.aws.dash0.com \
     --auth-token auth_xxx
   ```
4. Verify the profile:
   ```bash
   dash0 config show
   ```
5. Test connectivity:
   ```bash
   dash0 dashboards list
   ```
   An empty table is fine — the important thing is a non-error response.

---

### Exercise 2 — Export a Dashboard as YAML and Re-Apply It

**Goal:** Practice the full export-edit-apply round-trip.

**Steps:**
1. List your dashboards and choose one to practice with:
   ```bash
   dash0 dashboards list
   ```
2. Export it as YAML:
   ```bash
   dash0 dashboards get <id> -o yaml > test-dashboard.yaml
   ```
3. Open the file and inspect the structure. Note the `kind`, `id`, and resource definition fields.
4. Make a harmless visible change (e.g., add a trailing space to a title string).
5. Apply the file back:
   ```bash
   dash0 apply -f test-dashboard.yaml
   ```
6. The command should report "updated" (not "created"). The dashboard in the UI should be unchanged (your edit was cosmetic).
7. Run `dash0 apply -f test-dashboard.yaml` a second time. Confirm it's idempotent — no error, same result.

---

### Exercise 3 — Set Up a GitOps Directory and Use Dry-Run

**Goal:** Validate the `--dry-run` workflow that would run on a pull request.

**Steps:**
1. Create an `observability/` directory:
   ```bash
   mkdir observability
   ```
2. Export a dashboard and a check rule into it:
   ```bash
   dash0 dashboards get <id> -o yaml > observability/dashboard.yaml
   dash0 check-rules get <id> -o yaml > observability/check-rule.yaml
   ```
3. Run a dry-run against the directory:
   ```bash
   dash0 apply -f observability/ --dry-run
   ```
   Confirm both files are validated with no errors.
4. Introduce a deliberate YAML error in one file (delete the `kind:` line).
5. Run dry-run again — confirm it catches the error without applying anything.
6. Fix the YAML and run the actual apply:
   ```bash
   dash0 apply -f observability/
   ```

---

### Exercise 4 — Query Logs with Filters and Export as CSV

**Goal:** Practice the experimental log query workflow with multiple filters.

**Steps:**
1. Run a basic log query for the last 30 minutes:
   ```bash
   dash0 -X logs query --from now-30m --limit 20
   ```
2. Add a service name filter (replace `<your-service>` with a real service):
   ```bash
   dash0 -X logs query \
     --from now-30m \
     --filter "service.name is <your-service>" \
     --limit 20
   ```
3. Add a severity filter for ERROR logs:
   ```bash
   dash0 -X logs query \
     --from now-1h \
     --filter "service.name is <your-service>" \
     --filter "severity.text is ERROR" \
     --limit 50
   ```
4. Export as JSON and pipe to jq:
   ```bash
   dash0 -X logs query \
     --from now-30m \
     --filter "service.name is <your-service>" \
     -o json | jq '.[0]'
   ```
5. Export as CSV for spreadsheet analysis:
   ```bash
   dash0 -X logs query \
     --from now-30m \
     -o csv \
     --column time --column service.name --column severity.text --column body \
     > logs-export.csv
   ```

---

### Exercise 5 — Send a CI/CD Deployment Annotation

**Goal:** Simulate what a CI/CD pipeline would do to create a searchable deployment marker.

**Steps:**
1. Send a deployment start event:
   ```bash
   dash0 logs send "Deployment started" \
     --resource-attribute service.name=my-app \
     --resource-attribute service.version=v2.0.0 \
     --log-attribute deployment.sha=abc123def456 \
     --log-attribute deployment.env=staging \
     --log-attribute deployment.triggered-by=ci-pipeline \
     --severity-text INFO \
     --severity-number 9
   ```
2. Wait a few seconds, then send a completion event:
   ```bash
   dash0 logs send "Deployment complete" \
     --resource-attribute service.name=my-app \
     --resource-attribute service.version=v2.0.0 \
     --log-attribute deployment.sha=abc123def456 \
     --log-attribute deployment.env=staging \
     --severity-text INFO \
     --severity-number 9
   ```
3. Query for both events:
   ```bash
   dash0 -X logs query \
     --from now-5m \
     --filter "service.name is my-app" \
     --filter "deployment.sha is abc123def456"
   ```
4. Verify both log events appear with all attributes as filterable fields in the Dash0 UI.

---

### Exercise 6 — Test Agent Mode Output

**Goal:** Understand how agent mode changes CLI output for programmatic use.

**Steps:**
1. Run a list command in normal mode:
   ```bash
   dash0 dashboards list
   ```
   Observe the human-readable table output.

2. Run the same command with agent mode enabled:
   ```bash
   DASH0_AGENT_MODE=true dash0 dashboards list
   ```
   Observe: the output is now JSON.

3. Pipe the JSON output through jq:
   ```bash
   DASH0_AGENT_MODE=true dash0 dashboards list | jq '.[0]'
   ```

4. Trigger an error in agent mode:
   ```bash
   DASH0_AGENT_MODE=true dash0 dashboards get nonexistent-id-xyz 2>&1
   ```
   Observe the structured `{"error":"...","hint":"..."}` JSON on stderr.

5. Compare to the same error without agent mode:
   ```bash
   dash0 dashboards get nonexistent-id-xyz
   ```
   Note the difference: human-readable message vs machine-parseable JSON.

---

## Competitive Cheat Sheet

### CLI vs Datadog / Grafana / Dynatrace

| Capability | Datadog | Grafana | Dynatrace | **Dash0** |
|---|---|---|---|---|
| **Dashboard export/import** | UI only (no CLI) | grizzly (community, Grafana-only) | Monaco (own format) | `dash0 dashboards get -o yaml` + `apply` |
| **Alert rule management** | UI + API | No CLI | Monaco | `dash0 check-rules` + PrometheusRule import |
| **Synthetic check management** | UI only | No | Monaco | `dash0 synthetic-checks` |
| **Idempotent apply** | No | Partial | Yes | `dash0 apply -f` (kubectl semantics) |
| **Live log query** | `datadog logs` (limited) | No CLI | No CLI | `dash0 -X logs query` with OTel filters |
| **Live span/trace query** | No CLI | No CLI | No CLI | `dash0 -X spans query` + `traces get` |
| **PromQL metrics** | No | No CLI | No | `dash0 metrics instant` |
| **Telemetry injection** | ddtrace (process wrap) | No | No | `dash0 logs send` + `dash0 -X spans send` |
| **AI agent auto-detection** | No | No | No | Yes — 8 environments, zero-config |
| **Prometheus CRD import** | No | No | No | Yes — automatic conversion |
| **First-party, maintained** | Yes | Community | Yes | Yes |
| **Open source** | No | Yes (partial) | No | Yes |

### Key Differentiator Summary

**Dash0 CLI is the only observability CLI that:**
1. Manages the full asset lifecycle (dashboards + alerts + synthetics + views) in one binary
2. Also queries live telemetry (logs, spans, traces, PromQL) from the same binary
3. Also sends telemetry (logs, spans via OTLP) from the same binary
4. Auto-detects AI coding agent environments and provides machine-readable output with zero configuration

---

## Quick Reference Card

### One-Liners to Know by Heart

```bash
# Install
brew tap dash0hq/dash0-cli https://github.com/dash0hq/dash0-cli && brew install dash0

# Create + switch profile
dash0 config profiles create prod --api-url <url> --otlp-url <url> --auth-token <token>
dash0 config profiles select prod
dash0 config show

# Asset CRUD pattern (same for all 4 types)
dash0 dashboards list
dash0 dashboards get <id> -o yaml > dash.yaml
dash0 dashboards create -f dash.yaml
dash0 dashboards update <id> -f dash.yaml
dash0 dashboards delete <id> --force

# GitOps apply
dash0 apply -f observability/          # deploy
dash0 apply -f observability/ --dry-run  # validate (use on PRs)

# Query logs
dash0 -X logs query --from now-1h --filter "service.name is my-svc" --limit 100

# Query spans / retrieve trace
dash0 -X spans query --from now-2h --filter "service.name is api-gateway"
dash0 -X traces get <trace-id> --follow-span-links

# PromQL metrics
dash0 metrics instant --query 'sum(rate(http_requests_total[5m]))'

# Send telemetry
dash0 logs send "Deploy complete" \
  --resource-attribute service.name=my-app \
  --severity-text INFO --severity-number 9

dash0 -X spans send --name "GET /api" --kind SERVER --status-code OK \
  --duration 100ms --resource-attribute service.name=my-app

# Agent mode
DASH0_AGENT_MODE=true dash0 dashboards list
```

### Flags to Remember

| Flag | Effect |
|---|---|
| `-o json` | JSON output |
| `-o yaml` | YAML output |
| `-o csv` | CSV output |
| `-X` | Enable experimental commands |
| `--dry-run` | Validate without applying (apply only) |
| `--force` | Skip confirmation prompt (delete/destructive ops) |
| `--from now-1h` | Start of time range for queries |
| `--to now` | End of time range (default: now) |
| `--limit <n>` | Cap results returned |
| `--filter "<attr> is <val>"` | OTel attribute filter (stackable, ANDed) |
| `--follow-span-links` | Traverse span links in trace retrieval |
| `--agent-mode` | Enable machine-readable mode on any command |

### OTel Severity Quick Reference

```
TRACE=1   DEBUG=5   INFO=9   WARN=13   ERROR=17   FATAL=21
```

Always pair `--severity-text INFO` with `--severity-number 9` for full compatibility.

### Environment Variables

| Variable | Purpose |
|---|---|
| `DASH0_CONFIG_DIR` | Override `~/.dash0/` (essential in CI/CD) |
| `DASH0_AGENT_MODE` | `true` to enable / `false` to disable agent mode |
| `DASH0_AUTH_TOKEN` | Override auth token without editing the profile |

### Asset Type Quick-Reference

| Type | List | Get | Create | Update | Delete |
|---|---|---|---|---|---|
| Dashboards | `dashboards list` | `dashboards get <id>` | `dashboards create -f` | `dashboards update [id] -f` | `dashboards delete <id>` |
| Check Rules | `check-rules list` | `check-rules get <id>` | `check-rules create -f` | `check-rules update [id] -f` | `check-rules delete <id>` |
| Views | `views list` | `views get <id>` | `views create -f` | `views update [id] -f` | `views delete <id>` |
| Synthetic Checks | `synthetic-checks list` | `synthetic-checks get <id>` | `synthetic-checks create -f` | `synthetic-checks update [id] -f` | `synthetic-checks delete <id>` |

---

*Study guide for `dash0-cli-crash-course.html` — Dash0 CLI v1.0 / Platform v3.1*
