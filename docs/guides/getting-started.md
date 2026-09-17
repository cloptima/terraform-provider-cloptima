---
page_title: "Getting Started with AI Gateway Governance"
subcategory: "Guides"
description: |-
  Step-by-step guide to provisioning your first AI gateway governance policies, BYOK credentials, and virtual keys with Terraform.
---

# Getting Started with Cloptima AI Gateway Governance

This guide walks through configuring the Cloptima Terraform Provider to manage AI gateway policies, multi-provider BYOK credentials, and virtual keys as code.

## Architecture Overview

Cloptima acts as a gateway between your application workloads and AI providers (such as OpenAI, Anthropic, Google Vertex AI, or Azure OpenAI).

Every inference request is authenticated with a **Virtual Key** and governed by the **Gateway Policy** bound to that key's scope. The `team_id`, `app_id`, and `environment` set on the virtual key are authoritative and cannot be overridden by request headers:

```
App Workload (Virtual Key)
        │
        ▼
 Cloptima AI Gateway
   ├── Rate Limits (requests / tokens per minute)
   ├── Spend Budgets (daily / monthly)
   ├── Response Caching
   ├── Guardrails (PII, secrets, toxicity, prompt injection)
   └── Model Access & Routing
        │
        ▼
 Upstream Provider (BYOK Credential)
 (OpenAI, Anthropic, Bedrock, etc.)
```

---

## Step 1: Provider Authentication

To run Terraform against Cloptima, generate a **Personal Access Token (PAT)** in the Cloptima Console with the `ai:admin` scope.

Export the PAT in your shell:

```bash
export CLOPTIMA_PAT="clop_pat_..."
```

Declare the provider configuration:

```terraform
terraform {
  required_providers {
    cloptima = {
      source  = "cloptima/cloptima"
      version = "~> 0.3"
    }
  }
}

provider "cloptima" {}
```

---

## Step 2: Configure BYOK Provider Credentials

Register your organization's provider API keys. Cloptima stores them securely and never returns them after creation:

```terraform
resource "cloptima_llm_provider_credential" "anthropic_prod" {
  display_name  = "Anthropic Production BYOK"
  provider_name = "anthropic"
  api_key       = var.anthropic_api_key
}

resource "cloptima_llm_provider_credential" "openai_prod" {
  display_name  = "OpenAI Production BYOK"
  provider_name = "openai"
  api_key       = var.openai_api_key
}
```

---

## Step 3: Establish a Governance Policy

Define spend controls, caching behaviors, and safety guardrails in a `cloptima_llm_gateway_policy`. Policies default to `observe` mode, which records decisions without blocking; set `mode = "enforce"` to apply limits and guardrails:

```terraform
resource "cloptima_llm_gateway_policy" "standard_prod" {
  name = "standard-production-policy"
  mode = "enforce"

  # Exact response cache to reduce redundant calls and latency
  exact_cache_enabled     = true
  exact_cache_mode        = "enforce"
  exact_cache_ttl_seconds = 86400

  # Rate limiting and spend ceilings
  request_rate_limit_per_minute = 120
  token_rate_limit_per_minute   = 100000
  daily_budget_usd              = 250.00
  monthly_budget_usd            = 5000.00

  # Model access rules
  allowed_models = [
    "anthropic/claude-sonnet-4-6",
    "openai/gpt-4o",
    "openai/gpt-4o-mini"
  ]

  # Real-time safety guardrails
  guardrail_detectors_enabled = [
    "prompt_injection",
    "jailbreak",
    "toxicity",
    "pii",
    "secret"
  ]
  guardrail_output_action              = "block"
  guardrail_cost_mode                  = "enforce"
  guardrail_max_cost_per_request_cents = 10
}
```

---

## Step 4: Bind the Policy to Applications

Use `cloptima_llm_gateway_policy_binding` to associate the policy with specific teams, applications, or environments:

```terraform
resource "cloptima_llm_gateway_policy_binding" "copilot_prod" {
  policy_id   = cloptima_llm_gateway_policy.standard_prod.id
  team_id     = "platform-engineering"
  app_id      = "customer-copilot"
  environment = "production"
  priority    = 10
}
```

---

## Step 5: Provision Virtual Keys

Applications authenticate to the gateway using virtual keys. Virtual keys keep provider secrets out of client code and carry the attribution scope used for policy matching and cost tracking:

```terraform
resource "cloptima_llm_virtual_key" "copilot" {
  name                  = "customer-copilot-prod-key"
  team_id               = "platform-engineering"
  app_id                = "customer-copilot"
  environment           = "production"
  default_credential_id = cloptima_llm_provider_credential.openai_prod.id
  expires_in_days       = 180
}

# The virtual key value is available only from the apply that creates it
output "copilot_gateway_token" {
  value     = cloptima_llm_virtual_key.copilot.access_token
  sensitive = true
}
```

---

## Step 6: Make Application Inference Calls

Applications can now send OpenAI-compatible requests to the Cloptima AI Gateway. OpenAI SDK clients use `https://api.cloptima.ai/v1/ai` as their base URL:

```bash
curl https://api.cloptima.ai/v1/ai/chat/completions \
  -H "Authorization: Bearer <copilot_gateway_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "openai/gpt-4o",
    "messages": [
      {"role": "user", "content": "Explain Terraform in three sentences."}
    ]
  }'
```
