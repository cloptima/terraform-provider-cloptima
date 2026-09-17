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

## Step 4: Lifecycle

- **Lease renewal**: A running edge gateway keeps its lease current with Cloptima automatically.
- **Changes**: Changing any `cloptima_llm_edge_instance` attribute replaces the instance and issues a new edge PAT; the Kubernetes Secret is updated in the same apply, then restart the edge gateway pods so they use the new edge PAT.
- **Revocation**: Destroying the resource revokes the edge instance and its edge PAT. An instance revoked outside Terraform is recreated on the next apply.
