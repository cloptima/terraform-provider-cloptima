# Terraform Provider for Cloptima

The Cloptima Terraform provider enables infrastructure-as-code management of Cloptima AI Gateway governance resources, including routing policies, policy bindings, provider credentials (BYOK), virtual keys, and MCP tool servers.

## Documentation

Full documentation for provider resources and data sources is available on the [Terraform Registry](https://registry.terraform.io/providers/cloptima/cloptima/latest/docs).

## Requirements

- [Terraform](https://www.terraform.io/downloads.html) >= 1.0
- Cloptima Account and Personal Access Token (PAT) with `ai:admin` scope

## Using the Provider

Add the provider to your Terraform configuration:

```hcl
terraform {
  required_providers {
    cloptima = {
      source  = "cloptima/cloptima"
      version = "~> 0.1"
    }
  }
}

provider "cloptima" {
  # Configuration can also be set via environment variables:
  # CLOPTIMA_ENDPOINT and CLOPTIMA_PAT
  endpoint  = "https://api.cloptima.ai/graphql"
  pat_token = var.cloptima_pat_token
}
```

### Authentication

The provider can be configured with:

- `endpoint`: The Cloptima API Gateway GraphQL endpoint (or via `CLOPTIMA_ENDPOINT` environment variable).
- `pat_token`: Personal Access Token with the `ai:admin` scope (or via `CLOPTIMA_PAT` environment variable).

## Example Usage

### Managing a Gateway Policy

```hcl
resource "cloptima_llm_gateway_policy" "default" {
  name = "production-default-policy"
  mode = "enforce"

  allowed_models = [
    "anthropic/claude-3-5-sonnet",
    "openai/gpt-4o"
  ]

  # Spend controls
  daily_budget_usd   = 100.00
  monthly_budget_usd = 2500.00

  # Rate limits
  request_rate_limit_per_minute = 120
  token_rate_limit_per_minute   = 500000

  # Semantic cache configuration
  semantic_cache_enabled              = true
  semantic_cache_mode                 = "enforce"
  semantic_cache_similarity_threshold = 0.85
  semantic_cache_max_age_seconds      = 86400

  # Guardrails
  guardrail_output_action = "redact"
}
```

### Binding a Policy to a Team/Environment

```hcl
resource "cloptima_llm_gateway_policy_binding" "backend_prod" {
  policy_id   = cloptima_llm_gateway_policy.default.id
  team_id     = "backend"
  environment = "production"
}
```

### Provider Credentials (BYOK)

```hcl
resource "cloptima_llm_provider_credential" "openai" {
  provider_name = "openai"
  display_name  = "Organization OpenAI production key"
  api_key       = var.openai_api_key
}
```

### Virtual Keys

```hcl
resource "cloptima_llm_virtual_key" "service_account" {
  name        = "backend-service"
  environment = "production"
  team_id     = "backend"
}
```

## Resources

- `cloptima_llm_gateway_policy`: Defines routing, caching, spending limits, and safety guardrails.
- `cloptima_llm_gateway_policy_binding`: Binds policies to teams, apps, environments, or actors.
- `cloptima_llm_provider_credential`: Manages BYOK (Bring Your Own Key) upstream provider credentials.
- `cloptima_llm_virtual_key`: Issues virtual keys (inference-scoped API keys) for routing client traffic through the gateway.
- `cloptima_llm_gateway_tool_server`: Registers and manages external Model Context Protocol (MCP) tool servers.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
