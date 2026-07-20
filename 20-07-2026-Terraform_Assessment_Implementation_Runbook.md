# DevSecOps360 -- Terraform Assessment Implementation Runbook

## Overview

This runbook describes how to execute and verify the Terraform
Assessment pipeline for both supported scenarios:

-   **Scenario A -- From Repository**
-   **Scenario B -- From Cloud (Terraformer)**

The assessment pipeline performs:

1.  tfsec security scan
2.  Terraform validate
3.  Terraform plan (read-only)
4.  OPA policy evaluation

For **from-cloud**, Terraformer first exports Azure resources and
prepares a Terraform workspace before the assessment stages run.

------------------------------------------------------------------------

# 1. Prerequisites

## Required tools

``` bash
python3 --version
terraform --version
terraformer version
az version
jq --version
```

## Azure Login (from-cloud only)

``` bash
az login
az account show

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

Repository layout:

    app/
    examples/
    runs/

------------------------------------------------------------------------

# 3. Scenario A -- From Repository

## Trigger Assessment

``` bash
cat <<'JSON' | python3 -m app.api_entry
{
  "scenario": "from-repo",
  "client": "ttms",
  "repo_path": "examples/tf_local"
}
JSON
```

Expected:

``` json
{
  "run_id":"...",
  "status":"COMPLETED"
}
```

------------------------------------------------------------------------

## Verify Output

``` bash
RUN_ID=<run_id>

python3 -m json.tool runs/$RUN_ID/metadata/metadata.json

ls runs/$RUN_ID/reports/iac/tfsec.json
ls runs/$RUN_ID/reports/tf/tfplan.json
ls runs/$RUN_ID/reports/opa/opa_decision.json
```

------------------------------------------------------------------------

## Review Results

``` bash
python3 -m json.tool runs/$RUN_ID/reports/iac/tfsec.json

python3 -m json.tool runs/$RUN_ID/reports/tf/plan_summary.json

python3 -m json.tool runs/$RUN_ID/reports/opa/opa_decision.json
```

Expected:

-   tfsec completed
-   Terraform plan generated
-   OPA passed
-   Metadata status = POLICY_PASSED

------------------------------------------------------------------------

# 4. Scenario B -- From Cloud

## Verify Azure Resources

``` bash
az group list -o table

az resource list \
  --resource-group ttms-aks \
  --query "[].{Name:name,Type:type}" \
  -o table
```

------------------------------------------------------------------------

## Manual Terraformer Test

``` bash
terraformer import azure \
  --resources virtual_network \
  --resource-group ttms-aks \
  --path-output /tmp/tf_out \
  --compact
```

Verify:

``` bash
find /tmp/tf_out
```

------------------------------------------------------------------------

## Trigger From Cloud Assessment

Example Virtual Network:

``` bash
cat <<'JSON' | python3 -m app.api_entry
{
  "scenario":"from-cloud",
  "client":"ttms",
  "resources":"virtual_network",
  "cloud":{
    "resource_group":"ttms-aks"
  }
}
JSON
```

Example Storage Account:

``` bash
cat <<'JSON' | python3 -m app.api_entry
{
  "scenario":"from-cloud",
  "client":"ttms",
  "resources":"storage_account",
  "cloud":{
    "resource_group":"terraformer-testing"
  }
}
JSON
```

------------------------------------------------------------------------

## Verify Stages

``` bash
RUN_ID=<run_id>

python3 -m json.tool runs/$RUN_ID/reports/stage1_terraformer.json

python3 -m json.tool runs/$RUN_ID/reports/stage1_5_prepare_plan_workspace.json

python3 -m json.tool runs/$RUN_ID/reports/tf/validate_summary.json

python3 -m json.tool runs/$RUN_ID/reports/tf/plan_summary.json

python3 -m json.tool runs/$RUN_ID/reports/opa/opa_decision.json
```

Expected pipeline:

    Terraformer
        ↓
    Prepare Workspace
        ↓
    tfsec
        ↓
    Terraform Validate
        ↓
    Terraform Plan
        ↓
    OPA

------------------------------------------------------------------------

# 5. Run Folder

    runs/<run_id>/
    ├── metadata/
    ├── logs/
    ├── reports/
    │   ├── iac/
    │   ├── tf/
    │   ├── opa/
    │   ├── stage1_terraformer.json
    │   └── stage1_5_prepare_plan_workspace.json
    └── workspaces/

------------------------------------------------------------------------

# 6. Troubleshooting

## ARM_SUBSCRIPTION_ID missing

``` bash
export ARM_SUBSCRIPTION_ID=$(az account show --query id -o tsv)
```

## jq fails

Use:

``` bash
python3 -m json.tool <file.json>
```

## Terraformer exports zero resources

Verify:

``` bash
az resource list --resource-group <rg> -o table
```

Choose the matching Terraformer resource type.

------------------------------------------------------------------------

# 7. Success Criteria

Both scenarios should produce:

-   Completed run
-   Metadata generated
-   tfsec report
-   Terraform plan
-   OPA decision
-   Logs
-   POLICY_PASSED (when no violations exist)

------------------------------------------------------------------------

# 8. Future Enhancements

-   Multi-resource assessment
-   Multi-cloud support (Azure, AWS, GCP)
-   REST API integration
-   UI integration
-   Parallel assessment execution
