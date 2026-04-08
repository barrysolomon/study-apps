# Dash0 Terraform Provider — Deep Technical Crash Course

**For:** Barry Solomon, Senior SA @ Dash0
**Deadline:** End of week (April 10, 2026)
**Depth:** Deep technical — write HCL, troubleshoot, demo-ready

---

## Phase 1: Foundations (Day 1 — Wednesday)

### What It Is

The Dash0 Terraform provider (`dash0hq/dash0`) lets you manage observability resources as code. Currently supports 4 resource types:

| Resource | HCL Type | YAML Format | UI Export Path |
|---|---|---|---|
| Dashboards | `dash0_dashboard` | Perses Dashboard CRD | Dashboards → Settings → Source → Download YAML |
| Synthetic Checks | `dash0_synthetic_check` | Dash0 Synthetic YAML | Synthetics → Select check → Download YAML |
| Check Rules (Alerts) | `dash0_check_rule` | PrometheusRule CRD | Check Rules → Select rule → Download YAML |
| Views | `dash0_view` | Dash0 View YAML | Views → Select view → Download YAML |

### Provider Setup

```hcl
terraform {
  required_providers {
    dash0 = {
      source  = "dash0hq/dash0"
      version = "~> 1.6.0"
    }
  }
}

provider "dash0" {
  # Option A: env vars (recommended for CI/CD)
  # export DASH0_URL="https://api.us-west-2.aws.dash0.com"
  # export DASH0_AUTH_TOKEN="auth_xxxx"
  
  # Option B: inline (for local dev)
  url        = "https://api.us-west-2.aws.dash0.com"
  auth_token = "auth_xxxx"
}
```

**Auth token:** Settings → API Tokens in the Dash0 UI. Needs org-level scope.

**API endpoints by region:**
- US West: `https://api.us-west-2.aws.dash0.com`
- EU: `https://api.eu-west-1.aws.dash0.com`
- (Verify current endpoints in Dash0 docs — these may have expanded)

### Key Concept: YAML-First Design

Every resource takes a `dataset` string and a `*_yaml` string. The YAML is the source of truth. The Terraform provider wraps the Dash0 API to create/update/delete these YAML-defined resources.

This means:
- You export YAML from the UI to bootstrap
- You version-control the YAML files
- Terraform handles lifecycle (create, detect drift, update, destroy)
- The YAML format matches what the Dash0 Operator uses for K8s CRDs

### Exercise 1: Bootstrap

1. Install Terraform (if not already): `brew install terraform`
2. Create a new directory: `mkdir dash0-tf && cd dash0-tf`
3. Create `main.tf` with the provider block above
4. Run `terraform init` — verify the provider downloads
5. Get an API token from Dash0 settings
6. Export your env vars
7. Create a simple dashboard in the Dash0 UI, export its YAML
8. Add a `dash0_dashboard` resource pointing to the YAML file
9. Run `terraform plan` — see what Terraform wants to create
10. Run `terraform apply` — watch it deploy

---

## Phase 2: Resources Deep Dive (Day 2 — Thursday)

### dash0_dashboard

```hcl
resource "dash0_dashboard" "service_overview" {
  dataset        = "default"
  dashboard_yaml = file("${path.module}/dashboards/service-overview.yaml")
}
```

- `dataset` — scopes the dashboard to a dataset (e.g., "default", "production")
- `dashboard_yaml` — Perses Dashboard CRD format (same as K8s operator)
- On update: Terraform detects YAML changes and pushes the update
- On destroy: dashboard is removed from Dash0

**Perses format note:** Dash0 dashboards use the [Perses](https://perses.dev/) CRD format. This is also what the Dash0 Kubernetes Operator expects. Same YAML, two deployment paths (Terraform or K8s).

### dash0_synthetic_check

```hcl
resource "dash0_synthetic_check" "api_health" {
  dataset              = "default"
  synthetic_check_yaml = file("${path.module}/synthetics/api-health.yaml")
}
```

- Defines HTTP/browser checks that run from Dash0's global PoPs
- YAML defines the check endpoint, frequency, assertions, locations
- Export from UI: Synthetics → select check → Download YAML

### dash0_check_rule

```hcl
resource "dash0_check_rule" "error_rate" {
  dataset         = "production"
  check_rule_yaml = file("${path.module}/alerts/error-rate.yaml")
}
```

- Uses **PrometheusRule CRD format** (apiVersion: monitoring.coreos.com/v1)
- Currently supports one group with one rule per resource
- Supports Dash0-specific annotations: `dash0-threshold-critical`, `dash0-threshold-degraded`, `dash0-enabled`
- PromQL expressions work exactly like in the Dash0 UI

### dash0_view

```hcl
resource "dash0_view" "my_view" {
  dataset   = "default"
  view_yaml = file("${path.module}/views/custom-view.yaml")
}
```

- Custom data views for organizing observability data
- Export from UI: Views → select view → Download YAML

### Exercise 2: Full Resource Set

1. Export one of each resource type from the Dash0 UI
2. Create a Terraform config with all 4 resource types
3. Organize: `dashboards/`, `synthetics/`, `alerts/`, `views/` subdirectories
4. Run `terraform plan` and `terraform apply`
5. Modify a dashboard YAML, re-run plan — observe the diff
6. Destroy one resource with `terraform destroy -target=dash0_dashboard.service_overview`

---

## Phase 3: Advanced Patterns (Day 3 — Friday)

### Multi-Environment with Workspaces or Vars

```hcl
variable "environment" {
  description = "Target environment"
  type        = string
  default     = "staging"
}

variable "dash0_dataset" {
  description = "Dash0 dataset for this environment"
  type        = string
}

resource "dash0_dashboard" "service_overview" {
  dataset        = var.dash0_dataset
  dashboard_yaml = file("${path.module}/dashboards/service-overview.yaml")
}
```

Then use tfvars per environment:
```
# staging.tfvars
dash0_dataset = "staging"

# production.tfvars
dash0_dataset = "production"
```

Apply: `terraform apply -var-file=production.tfvars`

### CI/CD Integration

```yaml
# GitHub Actions example
- name: Terraform Apply
  env:
    DASH0_URL: ${{ secrets.DASH0_API_URL }}
    DASH0_AUTH_TOKEN: ${{ secrets.DASH0_AUTH_TOKEN }}
  run: |
    terraform init
    terraform apply -auto-approve
```

### State Management

- Remote state backend (S3, GCS, Terraform Cloud) is essential for team use
- State tracks the Dash0 resource IDs — losing state means orphaned resources
- Import existing resources: `terraform import dash0_dashboard.name <id>`

### Terraform + K8s Operator: When to Use Which

| Factor | Terraform Provider | K8s Operator |
|---|---|---|
| Already using Terraform | Preferred | — |
| K8s-native GitOps (ArgoCD, Flux) | — | Preferred |
| Non-K8s environments | Only option | N/A |
| Dashboard co-located with app code | — | Preferred |
| Centralized observability team | Preferred | — |
| Multi-cloud / multi-cluster | Preferred | Per-cluster |

### Exercise 3: Multi-Env Deploy

1. Set up a Terraform workspace or var-file approach for staging/prod
2. Deploy the same dashboard to two different datasets
3. Modify the staging dashboard, verify prod is untouched
4. Set up a `terraform plan` in a CI script (even local shell script)
5. Practice `terraform import` on an existing Dash0 dashboard

---

## Quick Reference Card

| Item | Value |
|---|---|
| Provider source | `dash0hq/dash0` |
| Current version | `~> 1.6.0` (verify in registry) |
| Auth env vars | `DASH0_URL`, `DASH0_AUTH_TOKEN` |
| Resources | `dash0_dashboard`, `dash0_synthetic_check`, `dash0_check_rule`, `dash0_view` |
| YAML export | UI → resource → Settings/Source → Download YAML |
| Dashboard format | Perses CRD |
| Alert format | PrometheusRule CRD |
| GitHub repo | github.com/dash0hq/terraform-provider-dash0 |
| Registry | registry.terraform.io/providers/dash0hq/dash0 |
| API docs | api-docs.dash0.com |
| Also on OpenTofu | Yes |

---

## Game Day Readiness Checklist

- [ ] Can explain what the Dash0 Terraform provider does in 30 seconds
- [ ] Can set up provider from scratch (init, auth, first resource)
- [ ] Can export YAML from all 4 UI locations
- [ ] Can write HCL for all 4 resource types
- [ ] Understand Perses format (dashboards) vs PrometheusRule format (alerts)
- [ ] Can articulate Terraform vs K8s Operator decision criteria
- [ ] Can set up multi-environment deployment with vars
- [ ] Can troubleshoot common issues (auth, state, drift)
- [ ] Know where to find docs (registry, GitHub, Dash0 hub page)
