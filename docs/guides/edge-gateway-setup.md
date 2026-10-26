---
page_title: "Deploying Customer-VPC Edge Gateways"
subcategory: "Guides"
description: |-
  Register and deploy Cloptima edge gateways in your own VPC or Kubernetes cluster using Terraform.
---

# Deploying Customer-VPC Edge Gateways

Cloptima edge gateways run in your own VPC or Kubernetes cluster, so inference traffic flows from your applications through a gateway you operate while governance stays centrally managed in Cloptima.

## Architecture

In an edge deployment:
- **Inference traffic**: Applications send requests to the edge gateway running in your environment, which forwards them to your LLM providers.
- **Provider credentials**: The edge gateway uses provider credentials that you supply locally in your environment.
- **Governance**: The edge gateway authenticates to Cloptima with its edge PAT and keeps the gateway policies, rate limits, and budgets that apply to it in sync.
- **Usage telemetry**: The edge gateway reports usage to Cloptima for cost reporting and budgets.

```
┌────────────────────────────────────────────────────────┐
│ Customer VPC / Private Kubernetes                      │
│                                                        │
│  App Workload ──► Cloptima Edge Gateway ──► LLM        │
│                     │                      Provider    │
│                     │                                  │
│                     │ (Governance sync & usage)        │
└─────────────────────┼──────────────────────────────────┘
                      │
                      ▼
               Cloptima Cloud
         (Policies, Budgets, Analytics)
```

---

## Step 1: Register the Edge Instance

Use `cloptima_llm_edge_instance` to register an edge gateway in Terraform:

```terraform
resource "cloptima_llm_edge_instance" "vpc_prod" {
  name                   = "us-east-1-k8s-edge-cluster"
  environment            = "prod"
  region                 = "us-east-1"
  deployment_type        = "kubernetes"
  lease_duration_seconds = 86400 # 24-hour lease

  metadata = jsonencode({
    cluster_id = "prod-eks-01"
    account_id = "123456789012"
    managed_by = "terraform"
  })
}
```

The edge PAT is available in `cloptima_llm_edge_instance.vpc_prod.edge_pat` only from the apply that creates the instance; it is not recovered on import.

### Accepted Parameters
- `environment`: `dev`, `staging`, `prod`, `sandbox`
- `deployment_type`: `kubernetes`, `docker_compose`, `ecs`, `vm`, `local`, `other`
- `lease_duration_seconds`: `3600` (1 hour) to `2592000` (30 days)

---

## Step 2: Deploy the Edge Gateway with Helm

Store the edge PAT in a Kubernetes Secret and install the Cloptima edge gateway Helm chart. This example uses the Terraform `kubernetes` and `helm` providers:

```terraform
resource "kubernetes_namespace" "ai_gateway" {
  metadata {
    name = "ai-gateway"
  }
}

resource "kubernetes_secret" "cloptima_edge" {
  metadata {
    name      = "cloptima-edge-credentials"
    namespace = kubernetes_namespace.ai_gateway.metadata[0].name
  }

  data = {
    edge-pat = cloptima_llm_edge_instance.vpc_prod.edge_pat
  }
}

resource "helm_release" "cloptima_edge" {
  name       = "cloptima-edge-gateway"
  namespace  = kubernetes_namespace.ai_gateway.metadata[0].name
  repository = "oci://ghcr.io/cloptima/charts"
  chart      = "cloptima-edge-gateway"

  values = [yamlencode({
    edge = {
      patSecret = {
        name = kubernetes_secret.cloptima_edge.metadata[0].name
        key  = "edge-pat"
      }
    }
    redis = {
      embedded = {
        enabled = true
      }
    }
  })]
}
```

The chart requires a state store for the gateway. `redis.embedded.enabled = true` deploys one alongside the gateway; to use an existing instance instead, set `redis.url` or `redis.urlSecret`.

After the gateway starts and registers, the edge instance `status` changes from `pending` to `active`.

---

## Step 3: Provision a Telemetry Key (Optional)

Telemetry keys authenticate a separate collector or agent that sends LLM usage telemetry to Cloptima:

```terraform
resource "cloptima_telemetry_key" "edge_collector" {
  name            = "us-east-1-edge-telemetry"
  expires_in_days = 90
}

output "telemetry_access_token" {
  value     = cloptima_telemetry_key.edge_collector.access_token
  sensitive = true
}
```

---

## Budgets Across Edge Gateways

Edge gateways do not have budgets of their own. A policy's `daily_budget_usd` and `monthly_budget_usd`, and the organization-wide ceiling on the baseline policy, are shared by the cloud gateway and every edge gateway in your organization:

- Under a blocking budget (`budget_mode = "hard"`), each edge gateway holds a share of the budget's remaining room for the current UTC day, also counted toward the organization ceiling. It admits requests against that share locally and asks Cloptima for more in the background.
- Cloptima sizes each share from the room left in the budget and never grants more than that room, so all gateways together stay within one budget.
- Each edge gateway reports its spend about every 30 seconds. When the UTC day ends, the day's share closes at what the edge actually spent.
- If an edge gateway asks Cloptima for more budget and the request fails, the policy's `fail_mode` decides: `fail_closed` refuses requests that need more budget, and `fail_open` keeps serving and counts the spend.
- If an edge gateway has lost contact with Cloptima, the policy's `edge_offline_keep_serving` decides instead. When it is `true` (the default), the edge gateway keeps serving for up to 24 hours and applies the spend to your budgets when it reconnects. When it is `false`, the edge gateway serves only within the budget share it already holds.

There is nothing to configure for this: share sizes and refresh timing are managed by Cloptima. To give one group of edge gateways a budget of its own, bind a separate policy to the virtual keys or environment that group serves:

```terraform
resource "cloptima_llm_gateway_policy" "org_baseline" {
  name               = "org-baseline"
  is_baseline        = true
  budget_mode        = "hard"
  monthly_budget_usd = 40000.00
}

resource "cloptima_llm_gateway_policy" "eu_edge" {
  name             = "eu-edge-production"
  mode             = "enforce"
  budget_mode      = "hard"
  daily_budget_usd = 500.00
}

resource "cloptima_llm_gateway_policy_binding" "eu_edge" {
  policy_id   = cloptima_llm_gateway_policy.eu_edge.id
  environment = "production-eu"
}

# Confirm what governs that traffic before it arrives
data "cloptima_llm_gateway_effective_policy" "eu_edge" {
  environment = "production-eu"
  actor_type  = "service"

  depends_on = [cloptima_llm_gateway_policy_binding.eu_edge]
}
```

---

## Step 4: Lifecycle

- **Lease renewal**: A running edge gateway keeps its lease current with Cloptima automatically.
- **Changes**: Changing any `cloptima_llm_edge_instance` attribute replaces the instance and issues a new edge PAT; the Kubernetes Secret is updated in the same apply, then restart the edge gateway pods so they use the new edge PAT.
- **Revocation**: Destroying the resource revokes the edge instance and its edge PAT. An instance revoked outside Terraform is recreated on the next apply.
