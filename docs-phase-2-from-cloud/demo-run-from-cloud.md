# Demo Runbook: From-Cloud Terraform Assessment (Live Demo)

## Objective of This Demo

This demo shows how we can **safely assess existing Azure infrastructure** without making any changes.

We will:

* Export infrastructure from Azure
* Convert it into Terraform
* Normalize it so Terraform can run cleanly
* Generate a Terraform plan (read-only)
* Run security checks (tfsec)
* Run policy checks (OPA)

**No resources are created, modified, or deleted.**

---

## High-Level Flow (What I Will Show)

1. Trigger assessment from cloud (Terraformer)
2. Observe run creation (`run_id`)
3. Inspect generated run folder
4. Verify stage results
5. Review metadata, logs, and reports

---

## Preconditions (Already Done)

* Azure authentication is available
* Required tools are installed (Terraform, Terraformer, tfsec, OPA)
* Project directory exists

```bash
cd ~/devsecops360/terraform-assessment
```

---

## Step 1: Trigger From-Cloud Assessment

### What this does

* Reads Azure infrastructure from a **specific Resource Group**
* Exports it using Terraformer
* Starts the full assessment pipeline

We intentionally scope to **one RG** for clean, predictable results.

---

### Demo Command (Copy-Paste)

```bash
cat <<'JSON' | python3 -m app.api_entry
{
  "scenario": "from-cloud",
  "client": "ttms",
  "resources": "resource_group,virtual_network",
  "cloud": {
    "resource_group": "ttms-aks"
  }
}
JSON
```

---

### Expected Output

```json
{"run_id":"2026-02-03T103853Z-unknown-93349f","status":"COMPLETED"}
```

### Explanation (Say this)

> This command triggers a full, read-only Terraform assessment for the `ttms-aks` resource group.
> The pipeline has completed successfully.

---

## Step 2: Navigate to the Run Folder

Each execution creates a **unique run folder**.

```bash
cd runs/2026-02-03T103853Z-unknown-93349f
ls
```

Expected folders:

```
artifacts  logs  metadata  reports  workspaces
```

### Explanation

> Every run is isolated. This allows parallel executions, easy debugging, and clean demos.

---

## Step 3: Understand Folder Structure

### 1. `metadata/`

High-level information about the run.

```bash
jq . metadata/metadata.json
```

Contains:

* scenario
* timestamps
* final status
* client context

---

### 2. `reports/`

Structured outputs from each stage.

```bash
ls reports/
```

Example:

* stage1_terraformer.json
* stage1_5_prepare_plan_workspace.json
* tfsec.json
* opa_tfplan.json

---

### 3. `logs/`

Raw execution logs for debugging.

```bash
ls logs/
```

Used when something fails.

---

### 4. `workspaces/`

Actual Terraform working directories.

```bash
ls workspaces/plan
```

Contains:

* Terraform files
* Terraform plan
* Plan JSON for OPA

---

## Step 4: Verify Stage Results

### Quick status check

```bash
jq '.status' reports/*.json
```

Expected output:

```
"PASSED"
"PASSED"
```

### Explanation

> Each stage produces a structured report.
> PASSED means the stage completed without blocking issues.

---

## Step 5: What Each Stage Did (Simple Explanation)

### Stage 1 – Terraformer Export

* Reads Azure resources from `ttms-aks`
* Converts them into Terraform code
* Stores raw output under `artifacts/`

---

### Stage 1.5 – Workspace Normalization

Terraformer output is not always plan-ready.

This stage:

* Adds required `azurerm` provider features
* Fixes known AzureRM issues
* Pins provider versions
* Runs `terraform fmt`

Result: **clean, plan-ready Terraform code**

---

### Terraform Init & Plan (Safe Mode)

* Runs without backend
* No state file
* No apply

Purpose:

> Validate correctness only

---

### tfsec – Security Scan

* Static analysis of Terraform code
* Finds security misconfigurations
* Read-only, CI-friendly

---

### OPA – Policy Check

* Evaluates **Terraform plan**
* Checks what Terraform would actually do
* Enforces organizational rules

---

## Step 6: Final Confirmation

```bash
ls logs
ls reports
ls workspaces/plan
```

Everything exists → pipeline completed successfully.

---

## Key Demo Takeaways (Say This at the End)

> This assessment is safe, repeatable, and fully automated.
> It gives security, compliance, and engineering teams visibility into infrastructure **without risk**.

---

## Demo Status

✅ **From-Cloud Terraform Assessment completed successfully**

**Terraformer → Normalize → Plan → tfsec → OPA**

---
