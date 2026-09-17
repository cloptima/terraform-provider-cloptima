---
page_title: "PromptOps: Prompt Versioning & Evaluation Datasets"
subcategory: "Guides"
description: |-
  Manage prompts as code with Terraform: prompt templates, immutable prompt versions, activation, and evaluation datasets.
---

# PromptOps: Prompt Versioning & Evaluation Datasets

Treating system prompts as application code makes production AI systems easier to review and change safely. The Cloptima Terraform Provider lets engineering teams define, review, version, and activate prompts through standard GitOps workflows.

## Workflow Overview

```
 Git Pull Request (Prompt Update)
        │
        ▼
 Code Review & terraform plan
        │
        ▼
 Terraform Apply
   ├── Create new version (cloptima_llm_prompt_version)
   └── Activate new version (active = true)
        │
        ▼
 Production environments: after the prompt deployment
 release gate is approved, the next apply activates it
```

---

## Step 1: Define the Prompt Template

A `cloptima_llm_prompt_template` holds the metadata for a specific workload or agent:

```terraform
resource "cloptima_llm_prompt_template" "billing_assistant" {
  name        = "billing-faq-assistant"
  description = "RAG-augmented prompt template for customer billing inquiries"
  owner       = "fintech-eng"
  app_id      = "billing-copilot"
  environment = "production"

  tags = [
    "billing",
    "faq",
    "tier-1"
  ]
}
```

---

## Step 2: Declare Immutable Prompt Versions

Each iteration of the prompt content is declared as a `cloptima_llm_prompt_version`. Changing `content` or `change_summary` creates a new version rather than editing an existing one.

```terraform
# Version 1 (Initial Release)
resource "cloptima_llm_prompt_version" "v1" {
  template_id    = cloptima_llm_prompt_template.billing_assistant.id
  change_summary = "Initial prompt with concise tone"
  active         = false # Superseded

  content = "You are a billing specialist for Acme Corp. Answer questions regarding customer invoices using the provided context."
}

# Version 2 (Current Release)
resource "cloptima_llm_prompt_version" "v2" {
  template_id    = cloptima_llm_prompt_template.billing_assistant.id
  change_summary = "Refined prompt with dispute policy clarifications"
  active         = true

  content = "You are an expert billing representative for Acme Corp. If the customer disputes charges, adhere strictly to the 30-day refund window."
}
```

### Activation Behavior
- Activating a new version supersedes the previously active version. Earlier version resources do not show a plan diff when they are superseded; use each version's `status` attribute to see its current state.
- Templates in the `prod` or `production` environment require an approved prompt deployment release gate before a version can be activated. Until the gate is approved, the version is kept as a `draft`, and the next `terraform apply` retries activation.
- Versions are permanent history: destroying a version resource removes it from Terraform state but does not deactivate the version.

---

## Step 3: Capture Evaluation Datasets

Use `cloptima_llm_dataset` to capture a snapshot of recent requests for an application as an evaluation dataset. A dataset captures up to 1000 recent LLM requests when it is created, and its records do not change afterwards:

```terraform
resource "cloptima_llm_dataset" "billing_golden_bench" {
  name        = "billing-faq-golden-benchmark"
  description = "Snapshot of billing copilot requests used to evaluate prompt releases"
  app_id      = "billing-copilot"
  is_golden   = true
}
```

Set `is_golden = true` to mark the dataset as a golden dataset for evaluations. Changing `name`, `description`, or `app_id` captures a new dataset.

---

## Step 4: GitOps Promotion Pattern

1. **Pull Request**: A developer adds `cloptima_llm_prompt_version.v3` with updated `content` and opens a pull request.
2. **Review**: Reviewers check the prompt change and the `terraform plan` output.
3. **Merge & Apply**: `terraform apply` creates version 3 and requests its activation. In production environments, the version stays a `draft` until the prompt deployment release gate is approved; the next `terraform apply` after approval activates it.
