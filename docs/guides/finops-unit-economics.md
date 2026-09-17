---
page_title: "FinOps & LLM Unit Economics"
subcategory: "Guides"
description: |-
  Connect AI inference spend directly to business metrics by defining unit economics denominators and cost attribution models in Terraform.
---

# FinOps & LLM Unit Economics

Tracking raw inference spend in total dollars or token counts makes it difficult to assess business ROI. A spike in LLM spend might represent wasteful queries—or it might reflect a healthy 10x surge in active customers.

Cloptima enables **Unit Economics Attribution**, allowing FinOps and finance teams to normalize AI gateway spend against core operational business units.

---

## What is a Unit Economics Denominator?

A denominator represents a business driver that scales with your application's usage:
- **Cost per Monthly Active User (MAU)**: `Total AI Gateway Spend / Active Users`
- **Cost per Completed Order**: `Copilot Inference Spend / Total Completed Checkouts`
- **Cost per Support Ticket Resolved**: `Agent Spend / Resolved Support Cases`

By standardizing these metrics, engineering teams can set clear unit-cost efficiency targets (e.g. "keep copilot cost under \$0.05 per checkout").

---

## Step 1: Provision Business Denominators

Use `cloptima_llm_unit_economics_denominator` to configure the business units Cloptima uses when reporting LLM cost per unit:

```terraform
# Denominator 1: Cost per Monthly Active User
resource "cloptima_llm_unit_economics_denominator" "mau" {
  unit_type           = "monthly_active_user"
  display_name        = "Monthly Active User"
  description         = "Inference spend normalized against verified monthly active accounts"
  default_granularity = "monthly"
  default_group_by    = "app_id"
  is_default          = true
}

# Denominator 2: Cost per Completed Checkout
resource "cloptima_llm_unit_economics_denominator" "checkout" {
  unit_type           = "completed_checkout"
  display_name        = "Completed Checkout Order"
  description         = "AI copilot cost per completed e-commerce transaction"
  default_granularity = "daily"
  default_group_by    = "team_id"
  is_default          = false
}

# Denominator 3: Cost per Support Resolution
resource "cloptima_llm_unit_economics_denominator" "support_ticket" {
  unit_type           = "resolved_ticket"
  display_name        = "Resolved Support Ticket"
  description         = "Support agent LLM cost per ticket successfully resolved"
  default_granularity = "daily"
  default_group_by    = "app_id"
  is_default          = false
}
```

### Parameter Guidelines
- `unit_type`: A unique, lowercase alphanumeric identifier with underscores (e.g. `monthly_active_user`, `completed_checkout`).
- `default_granularity`: `daily`, `weekly`, or `monthly`.
- `default_group_by`: Default dimension to group reports by (e.g. `app_id`, `team_id`, `model`).
- `is_default`: Whether this is the organization's default unit.

---

## Step 2: Combine with Gateway Policies for FinOps Governance

Connect your unit economics targets with protective policy ceilings in `cloptima_llm_gateway_policy`:

```terraform
resource "cloptima_llm_gateway_policy" "ecommerce_copilot" {
  name = "ecommerce-copilot-policy"
  mode = "enforce"

  # Exact response caching cuts repeated query costs
  exact_cache_enabled     = true
  exact_cache_mode        = "enforce"
  exact_cache_ttl_seconds = 86400

  # Spend bounds protect against runaway loops (USD)
  daily_budget_usd   = 500.00
  monthly_budget_usd = 10000.00

  allowed_models = [
    "openai/gpt-4o-mini",
    "anthropic/claude-sonnet-4-6"
  ]
}
```

---

## Step 3: Importing Existing Denominator Settings

If you have already configured denominators in the Cloptima Console, you can import them into Terraform using their `unit_type`:

```bash
terraform import cloptima_llm_unit_economics_denominator.mau monthly_active_user
```
