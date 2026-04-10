# actions-terraform-core

Core reusable GitHub Actions workflows and composite actions for managing Terraform infrastructure.
This repository is intended for homelab use and provides a consistent, opinionated pipeline for planning, applying, and destroying Terraform-managed infrastructure backed by Terraform Cloud and Google Cloud Platform (GCP).

---

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Reusable Workflows](#reusable-workflows)
  - [Plan](#plan-workflow)
  - [Apply](#apply-workflow)
  - [Destroy](#destroy-workflow)
- [Composite Actions](#composite-actions)
  - [bootstrap-repository](#bootstrap-repository)
  - [terraform-cloud-login](#terraform-cloud-login)
  - [terraform-plan](#terraform-plan)
- [Usage Examples](#usage-examples)

---

## Overview

This repository contains three reusable workflows and three composite actions that together implement a full Terraform CI/CD lifecycle. The workflows are designed to be called from other repositories using the `workflow_call` event.

The toolchain relies on [mise](https://mise.jdx.dev/) for task execution (`mise plan`, `mise apply`, `mise init`, etc.) and expects a `mise.toml` or equivalent task configuration to be present in the calling repository.

---

## Prerequisites

Before using these workflows, ensure the following are in place:

- A Terraform Cloud organization and workspace configured for the target environment.
- A GCP service account with the appropriate IAM permissions.
- Workload Identity Federation (WIF) configured between GitHub Actions and the GCP service account.
- The Terraform Cloud API token stored as a secret in GCP Secret Manager under the name `terraform-cloud-api-token` (configurable).
- `mise` tasks defined in the calling repository: `init`, `validate`, `plan`, `apply`, and `destroy`.

---

## Reusable Workflows

### Plan Workflow

**File:** `.github/workflows/plan.yaml`

Runs a Terraform plan against the target environment. This workflow is intended to run on pull requests or on demand to preview infrastructure changes before they are applied.

**Steps executed:**
1. Check out the repository.
2. Bootstrap the repository toolchain via `mise`.
3. Authenticate to GCP using Workload Identity Federation.
4. Retrieve the Terraform Cloud API token from GCP Secret Manager.
5. Run Terraform init, validate, and plan. Results are posted to the workflow step summary and uploaded as artifacts.

**Inputs:**

| Input | Required | Default | Description |
|---|---|---|---|
| `environment` | Yes | - | The target environment name (e.g., `production`, `homelab`). |
| `gcp_sa_email` | Yes | - | The GCP service account email to impersonate. |
| `gcp_wif_resource_id` | Yes | - | The full Workload Identity Provider resource ID. |
| `concurrency_group` | No | `create-plan-${{ github.ref }}` | Concurrency group name to prevent duplicate runs. |
| `run_name` | No | `Plan Terraform changes - ${{ github.ref }}` | Display name of the workflow run. |
| `runs_on` | No | `ubuntu-22.04` | Runner label to use. |
| `gcp_token_lifetime` | No | `300s` | Lifetime of the GCP access token. |

---

### Apply Workflow

**File:** `.github/workflows/apply.yaml`

Applies a Terraform plan to the target environment. This workflow is intended to run after a plan has been reviewed and approved, typically on merge to the main branch.

**Steps executed:**
1. Check out the repository.
2. Bootstrap the repository toolchain via `mise`.
3. Authenticate to GCP using Workload Identity Federation.
4. Retrieve the Terraform Cloud API token from GCP Secret Manager.
5. Run Terraform plan (for a final pre-apply review).
6. Run `mise apply` to apply the changes.

**Inputs:**

| Input | Required | Default | Description |
|---|---|---|---|
| `environment` | Yes | - | The target environment name (e.g., `production`, `homelab`). |
| `gcp_sa_email` | Yes | - | The GCP service account email to impersonate. |
| `gcp_wif_resource_id` | Yes | - | The full Workload Identity Provider resource ID. |
| `concurrency_group` | No | `apply-plan-${{ github.ref }}` | Concurrency group name. Cancellation is disabled to prevent partial applies. |
| `run_name` | No | `Apply Terraform changes - ${{ github.ref }}` | Display name of the workflow run. |
| `runs_on` | No | `ubuntu-22.04` | Runner label to use. |
| `gcp_token_lifetime` | No | `300s` | Lifetime of the GCP access token. |

---

### Destroy Workflow

**File:** `.github/workflows/destroy.yaml`

Destroys all Terraform-managed infrastructure in the target environment. This workflow requires explicit opt-in via the `approve_destroy` flag and is intended to be triggered manually.

**Steps executed:**
1. Check out the repository.
2. Bootstrap the repository toolchain via `mise`.
3. Authenticate to GCP using Workload Identity Federation.
4. Retrieve the Terraform Cloud API token from GCP Secret Manager.
5. Run `mise destroy` — only if `approve_destroy` is set to `true`.

**Inputs:**

| Input | Required | Default | Description |
|---|---|---|---|
| `environment` | Yes | - | The target environment name. |
| `gcp_sa_email` | Yes | - | The GCP service account email to impersonate. |
| `gcp_wif_resource_id` | Yes | - | The full Workload Identity Provider resource ID. |
| `approve_destroy` | No | `false` | Safety flag. Must be set to `true` for the destroy step to execute. |
| `concurrency_group` | No | `destroy-infrastructure-${{ github.ref }}` | Concurrency group name. |
| `run_name` | No | `Destroy Terraform infrastructure` | Display name of the workflow run. |
| `runs_on` | No | `ubuntu-22.04` | Runner label to use. |
| `gcp_token_lifetime` | No | `300s` | Lifetime of the GCP access token. |

---

## Composite Actions

### bootstrap-repository

**File:** `.github/actions/bootstrap-repository/action.yaml`

Bootstraps the repository environment using [mise](https://mise.jdx.dev/). This action installs the tools and runtimes defined in the calling repository's mise configuration, preparing the runner for subsequent Terraform steps. It runs with the `experimental` flag enabled to support the latest mise features.

This action takes no inputs.

---

### terraform-cloud-login

**File:** `.github/actions/terraform-cloud-login/action.yaml`

Retrieves the Terraform Cloud API token from GCP Secret Manager and exposes it as a masked output for use by subsequent steps. The token is also written to `GITHUB_ENV` as `TF_API_TOKEN`.

This action requires that GCP authentication has already been performed (i.e., the `google-github-actions/auth` step has run and exported credentials to the environment).

**Inputs:**

| Input | Required | Default | Description |
|---|---|---|---|
| `terraform_access_token_secret_name` | No | `terraform-cloud-api-token` | Name of the GCP Secret Manager secret that holds the Terraform Cloud API token. |

**Outputs:**

| Output | Description |
|---|---|
| `terraform_cloud_api_token` | The Terraform Cloud API token, masked in logs. Pass this to `TF_TOKEN_app_terraform_io` for Terraform CLI authentication. |

---

### terraform-plan

**File:** `.github/actions/terraform-plan/action.yaml`

Runs a full Terraform plan sequence: init, validate, and plan. Log output from each step is captured, stripped of ANSI colour codes, and stored as both step outputs and workflow artifacts. A formatted summary is posted to the GitHub Actions step summary page.

**Inputs:**

| Input | Required | Default | Description |
|---|---|---|---|
| `environment` | Yes | - | Environment name, used to label artifacts and the step summary. |
| `plan_command` | No | `mise plan` | The command used to execute the Terraform plan step. |

**Outputs:**

| Output | Description |
|---|---|
| `plan_outcome` | Outcome of the plan step (`success`, `failure`, `skipped`). |
| `init_outcome` | Outcome of the init step. |
| `check_outcome` | Outcome of the validate step. |
| `plan_output` | Cleaned stdout from the plan command. |
| `init_output` | Cleaned stdout from the init command. |
| `check_output` | Cleaned stdout from the validate command. |

Logs from all three steps are uploaded as a workflow artifact named `Terraform-logs-<environment>-<sha>` and retained for 7 days.

---

## Usage Examples

### Calling the Plan workflow

```yaml
jobs:
  plan:
    uses: your-org/actions-terraform-core/.github/workflows/plan.yaml@main
    with:
      environment: homelab
      gcp_sa_email: terraform@my-project.iam.gserviceaccount.com
      gcp_wif_resource_id: projects/123456789/locations/global/workloadIdentityPools/my-pool/providers/my-provider
```

### Calling the Apply workflow

```yaml
jobs:
  apply:
    uses: your-org/actions-terraform-core/.github/workflows/apply.yaml@main
    with:
      environment: homelab
      gcp_sa_email: terraform@my-project.iam.gserviceaccount.com
      gcp_wif_resource_id: projects/123456789/locations/global/workloadIdentityPools/my-pool/providers/my-provider
```

### Calling the Destroy workflow

```yaml
jobs:
  destroy:
    uses: your-org/actions-terraform-core/.github/workflows/destroy.yaml@main
    with:
      environment: homelab
      gcp_sa_email: terraform@my-project.iam.gserviceaccount.com
      gcp_wif_resource_id: projects/123456789/locations/global/workloadIdentityPools/my-pool/providers/my-provider
      approve_destroy: true
```

> The destroy workflow will not execute the destroy step unless `approve_destroy` is explicitly set to `true`.
