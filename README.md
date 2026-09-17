# Terraform Provider for Cloptima AI Gateway

[![Terraform Registry](https://img.shields.io/badge/Terraform%20Registry-cloptima%2Fcloptima-blue.svg)](https://registry.terraform.io/providers/cloptima/cloptima/latest)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

The **Cloptima Terraform Provider** enables platform engineering, security, and FinOps teams to manage AI gateway infrastructure as code. Define LLM routing, exact and semantic caching, real-time safety guardrails, MCP tool servers, VPC edge instances, PromptOps release gates, and unit economics cost attribution declaratively using Terraform.

---

## Quickstart

### 1. Requirements

- [Terraform](https://www.terraform.io/downloads.html) >= 1.0
- A Cloptima account with a Personal Access Token (PAT) carrying the `ai:admin` scope

### 2. Configure the Provider

Add the provider declaration to your Terraform configuration:

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

Export your credentials:

```bash
export CLOPTIMA_PAT="clop_pat_..."
```

### 3. Basic Example: Governance Policy, Credential & Virtual Key

```terraform
# 1. Connect a BYOK LLM Provider Credential
resource "cloptima_llm_provider_credential" "openai_prod" {
  display_name  = "OpenAI Production"
  provider_name = "openai"
  api_key       = var.openai_api_key
}

# 2. Establish AI Gateway Governance Policy
resource "cloptima_llm_gateway_policy" "prod_policy" {
  name = "production-gateway-policy"

  # Enforce limits and guardrails (the default, observe, only records decisions)
  mode = "enforce"

  # Exact response cache
  exact_cache_enabled     = true
  exact_cache_mode        = "enforce"
  exact_cache_ttl_seconds = 86400

  # Spend bounds (USD)
  daily_budget_usd   = 500.00
  monthly_budget_usd = 10000.00

  # Rate limits
  request_rate_limit_per_minute = 120
  token_rate_limit_per_minute   = 100000

  # Model access
  allowed_models = [
    "anthropic/claude-sonnet-4-6",
    "openai/gpt-4o",
    "openai/gpt-4o-mini"
  ]

  # Active guardrails
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

# 3. Bind Policy to Target Environment and App
resource "cloptima_llm_gateway_policy_binding" "copilot_prod_binding" {
  policy_id   = cloptima_llm_gateway_policy.prod_policy.id
  team_id     = "engineering"
  app_id      = "customer-copilot"
  environment = "production"
  priority    = 10
}

# 4. Issue Developer Virtual Key
resource "cloptima_llm_virtual_key" "copilot_key" {
  name                  = "copilot-prod-key"
  team_id               = "engineering"
  app_id                = "customer-copilot"
  environment           = "production"
  default_credential_id = cloptima_llm_provider_credential.openai_prod.id
  expires_in_days       = 180
}

output "copilot_virtual_key_token" {
  value       = cloptima_llm_virtual_key.copilot_key.access_token
  description = "Bearer token for application requests through the AI Gateway"
  sensitive   = true
}
```

---

## Supported Resources & Data Sources

### Resources

| Resource | Description |
|---|---|
| [`cloptima_llm_gateway_policy`](https://registry.terraform.io/providers/cloptima/cloptima/latest/docs/resources/llm_gateway_policy) | Governance policies (rate limits, caching, budgets, safety guardrails, model routing). |
| [`cloptima_llm_gateway_policy_binding`](https://registry.terraform.io/providers/cloptima/cloptima/latest/docs/resources/llm_gateway_policy_binding) | Binds policies to team, application, environment, actor, or principal scopes. |
| [`cloptima_llm_provider_credential`](https://registry.terraform.io/providers/cloptima/cloptima/latest/docs/resources/llm_provider_credential) | Manages BYOK provider credentials (OpenAI, Anthropic, Azure, Google, etc.). |
| [`cloptima_llm_virtual_key`](https://registry.terraform.io/providers/cloptima/cloptima/latest/docs/resources/llm_virtual_key) | Issues application-scoped virtual keys for inference authentication and attribution. |
| [`cloptima_llm_gateway_tool_server`](https://registry.terraform.io/providers/cloptima/cloptima/latest/docs/resources/llm_gateway_tool_server) | Integrates and governs Model Context Protocol (MCP) tool servers. |
| [`cloptima_llm_edge_instance`](https://registry.terraform.io/providers/cloptima/cloptima/latest/docs/resources/llm_edge_instance) | Registers edge gateway instances that run in your own VPC or Kubernetes cluster. |
| [`cloptima_telemetry_key`](https://registry.terraform.io/providers/cloptima/cloptima/latest/docs/resources/telemetry_key) | Issues telemetry keys for sending LLM usage telemetry to Cloptima. |
| [`cloptima_llm_prompt_template`](https://registry.terraform.io/providers/cloptima/cloptima/latest/docs/resources/llm_prompt_template) | Manages version-controlled prompt templates for workloads and agents. |
| [`cloptima_llm_prompt_version`](https://registry.terraform.io/providers/cloptima/cloptima/latest/docs/resources/llm_prompt_version) | Creates immutable prompt versions and optionally activates them. |
| [`cloptima_llm_dataset`](https://registry.terraform.io/providers/cloptima/cloptima/latest/docs/resources/llm_dataset) | Manages prompt evaluation datasets, including golden datasets. |
| [`cloptima_llm_unit_economics_denominator`](https://registry.terraform.io/providers/cloptima/cloptima/latest/docs/resources/llm_unit_economics_denominator) | Configures business metric denominators for FinOps cost-per-unit tracking. |

### Data Sources

| Data Source | Description |
|---|---|
| [`cloptima_llm_gateway_policy`](https://registry.terraform.io/providers/cloptima/cloptima/latest/docs/data-sources/llm_gateway_policy) | Look up an existing gateway policy by ID or name. |
| [`cloptima_llm_provider_credential`](https://registry.terraform.io/providers/cloptima/cloptima/latest/docs/data-sources/llm_provider_credential) | Look up an existing provider credential by ID, display name, or provider name. |
| [`cloptima_llm_gateway_tool_server`](https://registry.terraform.io/providers/cloptima/cloptima/latest/docs/data-sources/llm_gateway_tool_server) | Look up an existing MCP tool server by ID or name. |
| [`cloptima_llm_virtual_key`](https://registry.terraform.io/providers/cloptima/cloptima/latest/docs/data-sources/llm_virtual_key) | Look up an existing virtual key by ID or name. |

---

## Guides & Documentation

- [Getting Started with AI Gateway Governance](https://registry.terraform.io/providers/cloptima/cloptima/latest/docs/guides/getting-started)
- [Deploying Customer-VPC Edge Gateways](https://registry.terraform.io/providers/cloptima/cloptima/latest/docs/guides/edge-gateway-setup)
- [PromptOps: Prompt Versioning & Evaluation Datasets](https://registry.terraform.io/providers/cloptima/cloptima/latest/docs/guides/promptops-workflows)
- [FinOps & LLM Unit Economics](https://registry.terraform.io/providers/cloptima/cloptima/latest/docs/guides/finops-unit-economics)

---

## License

This provider is licensed under the Apache License, Version 2.0.
