# DevSecOps360 -- Terraform Assessment (Demo Runbook)

## Overview

This runbook demonstrates the Terraform Assessment workflow end-to-end
for both supported execution scenarios:

-   **Scenario A -- From Repository**
-   **Scenario B -- From Cloud (Terraformer)**

It explains:

-   How an assessment run is triggered
-   How each run is uniquely identified
-   End-to-end execution flow
-   Generated artifacts
-   How to verify each stage
-   How to interpret the final assessment result

------------------------------------------------------------------------

## Current Integration Status (Phase-1 MVP)

The CLI implementation is complete for both supported scenarios.

### Supported Scenarios

-   From Repository
-   From Cloud (Terraformer)

### Implemented Stages

-   Terraformer Export (From Cloud)
-   Prepare Plan Workspace
-   tfsec Security Scan
-   Terraform Validate
-   Terraform Plan (Read-only)
-   OPA Policy Evaluation

Each execution generates:

-   Metadata
-   Logs
-   Tool Reports
-   Terraform Artifacts

UI and Backend API integration is planned as the next phase.

------------------------------------------------------------------------

# 1. Prerequisites

## Required Tools

``` bash
python3 --version
terraform --version
terraformer version
az version
jq --version
```

If jq is unavailable, use:

``` bash
python3 -m json.tool <file.json>
```

------------------------------------------------------------------------

## Azure Authentication (From Cloud)

``` bash
az login

export ARM_SUBSCRIPTION_ID=$(az account show --query id -o tsv)
export ARM_TENANT_ID=$(az account show --query tenantId -o tsv)

echo $ARM_SUBSCRIPTION_ID
echo $ARM_TENANT_ID
```

------------------------------------------------------------------------

# 2. Repository

``` bash
cd ~/devsecops360/terraform-assessment
```

Expected:

    app/
    examples/
    runs/

------------------------------------------------------------------------

# 3. High-Level Execution Flow

## Scenario A -- From Repository

``` text
Terraform Code
      │
      ▼
tfsec
      │
      ▼
terraform validate
      │
      ▼
terraform plan
      │
      ▼
OPA Policy Evaluation
      │
      ▼
Assessment Summary
```

## Scenario B -- From Cloud

``` text
Azure Resource Group
      │
      ▼
Terraformer Export
      │
      ▼
Prepare Plan Workspace
      │
      ▼
tfsec
      │
      ▼
terraform validate
      │
      ▼
terraform plan
      │
      ▼
OPA Policy Evaluation
      │
      ▼
Assessment Summary
```

------------------------------------------------------------------------

# 4. Run Output Structure

    runs/<run_id>/
    ├── metadata/
    │   └── metadata.json
    ├── logs/
    │   ├── terraformer.log
    │   ├── tfsec.log
    │   ├── tf_validate.log
    │   ├── tf_plan.log
    │   └── opa.log
    ├── reports/
    │   ├── stage1_terraformer.json
    │   ├── stage1_5_prepare_plan_workspace.json
    │   ├── iac/
    │   │   └── tfsec.json
    │   ├── tf/
    │   │   ├── validate_summary.json
    │   │   ├── plan_summary.json
    │   │   ├── tfplan.bin
    │   │   └── tfplan.json
    │   └── opa/
    │       └── opa_decision.json
    ├── artifacts/
    │   └── terraformer/
    └── workspaces/
        └── plan/

------------------------------------------------------------------------

# 5. Scenario A -- From Repository

## Verify Sample Code

``` bash
ls -la examples/tf_local
```

## Trigger Assessment

``` bash
cat <<'JSON' | python3 -m app.api_entry
{
  "scenario":"from-repo",
  "client":"ttms",
  "repo_path":"examples/tf_local"
}
JSON
```

Expected:

``` json
{
  "run_id":"<run_id>",
  "status":"COMPLETED"
}
```

Store Run ID

``` bash
RUN_ID=<run_id>
```

Verify:

``` bash
python3 -m json.tool runs/$RUN_ID/metadata/metadata.json
python3 -m json.tool runs/$RUN_ID/reports/iac/tfsec.json
python3 -m json.tool runs/$RUN_ID/reports/tf/plan_summary.json
python3 -m json.tool runs/$RUN_ID/reports/opa/opa_decision.json
```

------------------------------------------------------------------------

# 6. Scenario B -- From Cloud

## Verify Resource Group

``` bash
az group list -o table

az resource list \
  --resource-group timoweb \
  --query "[].{Name:name,Type:type}" \
  -o table
```

## Trigger Assessment

Example:

``` bash
cat <<'JSON' | python3 -m app.api_entry
{
  "scenario":"from-cloud",
  "client":"ttms",
  "resources":"virtual_network",
  "cloud":{
      "resource_group":"timoweb"
  }
}
JSON
```

Expected

``` json
{
  "run_id":"<run_id>",
  "status":"COMPLETED"
}
```

Store Run ID

``` bash
RUN_ID=<run_id>
```

Verify:

``` bash
python3 -m json.tool runs/$RUN_ID/reports/stage1_terraformer.json
python3 -m json.tool runs/$RUN_ID/reports/stage1_5_prepare_plan_workspace.json
python3 -m json.tool runs/$RUN_ID/reports/iac/tfsec.json
python3 -m json.tool runs/$RUN_ID/reports/tf/validate_summary.json
python3 -m json.tool runs/$RUN_ID/reports/tf/plan_summary.json
python3 -m json.tool runs/$RUN_ID/reports/opa/opa_decision.json
python3 -m json.tool runs/$RUN_ID/metadata/metadata.json
```

------------------------------------------------------------------------

# 7. Result Interpretation

  Stage                Expected
  -------------------- ---------------
  Terraformer          PASSED
  Prepare Workspace    PASSED
  tfsec                RETURN_CODE=0
  Terraform Validate   passed=true
  Terraform Plan       passed=true
  OPA                  allow=true
  Metadata             POLICY_PASSED

------------------------------------------------------------------------

# 8. Troubleshooting

## ARM_SUBSCRIPTION_ID not set

``` bash
export ARM_SUBSCRIPTION_ID=$(az account show --query id -o tsv)
export ARM_TENANT_ID=$(az account show --query tenantId -o tsv)
```

## jq issue

Use:

``` bash
python3 -m json.tool <json-file>
```

## Terraformer exports zero resources

Verify Azure resources:

``` bash
az resource list --resource-group <rg> -o table
```

Choose the correct Terraformer resource type.

------------------------------------------------------------------------

# 9. Demo Summary

The Terraform Assessment pipeline now supports both execution scenarios.

## From Repository

-   tfsec
-   Terraform Validate
-   Terraform Plan
-   OPA

## From Cloud

-   Terraformer Export
-   Prepare Workspace
-   tfsec
-   Terraform Validate
-   Terraform Plan
-   OPA

Every assessment generates:

-   Metadata
-   Logs
-   Reports
-   Terraform Artifacts

No infrastructure changes are applied during assessment.

------------------------------------------------------------------------

# Appendix -- Quick Commands

## From Repository

``` bash
cat <<'JSON' | python3 -m app.api_entry
{
  "scenario":"from-repo",
  "client":"ttms",
  "repo_path":"examples/tf_local"
}
JSON
```

## From Cloud

``` bash
cat <<'JSON' | python3 -m app.api_entry
{
  "scenario":"from-cloud",
  "client":"ttms",
  "resources":"virtual_network",
  "cloud":{
      "resource_group":"timoweb"
  }
}
JSON
```
