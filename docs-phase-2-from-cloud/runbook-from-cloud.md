# Runbook: From-Cloud Terraform Assessment Pipeline
**Terraformer → Prepare Workspace → Terraform Plan → tfsec → OPA (tfplan)**

---

## Purpose
This runbook documents the complete **from-cloud** execution flow of the Terraform Assessment pipeline.

It demonstrates how existing Azure infrastructure is:
1. Exported using Terraformer (scoped to a Resource Group)
2. Normalized into a clean Terraform workspace
3. Planned safely (no backend, no apply)
4. Security-scanned with tfsec
5. Policy-validated using OPA on the Terraform plan

All execution artifacts are isolated under a single run folder:
```

runs/<run_id>/

````

This runbook is suitable for:
- Live demos
- Knowledge transfer
- Future troubleshooting
- Interview explanation

---

## Preconditions

### Project root
```bash
cd ~/devsecops360/terraform-assessment
````

### Required tools

* Python 3
* Terraform
* Terraformer
* tfsec
* OPA

### Verify tools (copy-paste)

```bash
which python3 && python3 --version
which terraform && terraform version
which terraformer && terraformer import azure --help
which tfsec && tfsec --version || true
which opa && opa version || true
```

---

## Authentication (choose one)

### Option A: Azure Service Principal

```bash
export ARM_CLIENT_ID="..."
export ARM_CLIENT_SECRET="..."
export ARM_TENANT_ID="..."
export ARM_SUBSCRIPTION_ID="..."
```

### Option B: Azure CLI

```bash
az login
az account show
export ARM_SUBSCRIPTION_ID="$(az account show --query id -o tsv)"
```

---

## Expected Run Folder Layout

After execution, each run produces:

```
runs/<run_id>/
├── metadata.json
├── logs/
│   ├── terraformer.log
│   ├── terraform_init.log
│   ├── terraform_plan.log
│   ├── tfsec.log
│   └── opa_tfplan.log
├── reports/
│   ├── stage1_terraformer.json
│   ├── stage1_5_prepare_plan_workspace.json
│   ├── tfsec.json
│   └── opa_tfplan.json
├── artifacts/
│   └── terraformer/
│       └── generated/...
└── workspaces/
    └── plan/
        ├── *.tf
        ├── .terraform/
        ├── plan.tfplan
        └── plan.json
```

---

## Stage 1: Trigger From-Cloud Execution (Terraformer)

Terraformer export **must be scoped to a Resource Group**.

### JSON input (IMPORTANT FIX #1)

```json
"cloud": {
  "resource_group": "ttms-prod-rg"
}
```

Python maps this to:

```
--resource-group=ttms-prod-rg
```

### Trigger command (copy-paste)

```bash
cat <<'JSON' | python3 -m app.api_entry
{
  "scenario": "from-cloud",
  "client": "ttms",
  "resources": "resource_group,virtual_network",
  "cloud": {
    "resource_group": "ttms-prod-rg"
  }
}
JSON
```

Expected output:

```json
{"run_id":"<RUN_ID>","status":"IN_PROGRESS"}
```

---

## Set RUN_ID

```bash
RUN_ID="<PASTE_RUN_ID_HERE>"
RUN_PATH="runs/$RUN_ID"
echo "$RUN_PATH"
ls -la "$RUN_PATH"
```

---

## Verify Stage 1: Terraformer Export

### Stage report

```bash
cat "$RUN_PATH/reports/stage1_terraformer.json"
```

Expected:

* `"status": "PASSED"`
* `"tf_file_count" > 0`

### Logs

```bash
tail -n 80 "$RUN_PATH/logs/terraformer.log"
```

### Artifacts

Terraformer creates a `generated/` folder under its output root.

```bash
find "$RUN_PATH/artifacts/terraformer" -maxdepth 4 -type d | head
find "$RUN_PATH/artifacts/terraformer" -name "*.tf" | head -n 30
```

---

## Stage 1.5: Prepare Terraform Plan Workspace

This stage copies Terraformer output into a clean plan workspace.

### Verify stage report

```bash
cat "$RUN_PATH/reports/stage1_5_prepare_plan_workspace.json"
```

### Verify workspace contents

```bash
find "$RUN_PATH/workspaces/plan" -name "*.tf" | head -n 30
```

Expected:

* `"status": "PASSED"`
* Workspace contains `.tf` files

---

## Terraform Init and Plan (Safe Mode)

All Terraform commands run inside:

```
runs/<run_id>/workspaces/plan
```

### Terraform init

```bash
cd "$RUN_PATH/workspaces/plan"

terraform init \
  -input=false \
  -backend=false \
  -no-color | tee "$RUN_PATH/logs/terraform_init.log"
```

### Terraform plan

```bash
terraform plan \
  -input=false \
  -no-color \
  -out plan.tfplan | tee "$RUN_PATH/logs/terraform_plan.log"
```

### Convert plan to JSON (for OPA)

```bash
terraform show -json plan.tfplan > plan.json
ls -la plan.tfplan plan.json
```

---

## tfsec Scan (CI-Friendly)

### IMPORTANT FIX #2

Use `--no-colour` to avoid ANSI characters in CI logs.

### Run tfsec

```bash
cd "$RUN_PATH/workspaces/plan"

tfsec . \
  --format json \
  --no-colour \
  > "$RUN_PATH/reports/tfsec.json" \
  2> "$RUN_PATH/logs/tfsec.log" || true
```

### Review results

```bash
jq '.results | length' "$RUN_PATH/reports/tfsec.json" 2>/dev/null || head -n 40 "$RUN_PATH/reports/tfsec.json"
tail -n 60 "$RUN_PATH/logs/tfsec.log"
```

---

## OPA Policy Check (Terraform Plan)

OPA validates the **actual execution plan**, not static code.

### Run OPA

```bash
cd "$RUN_PATH/workspaces/plan"

opa eval \
  --format pretty \
  --data "$PWD/../../../app/policies/tfplan" \
  --input plan.json \
  "data.tfplan.deny" | tee "$RUN_PATH/logs/opa_tfplan.log"
```

### Interpretation

* Empty output → policy passed
* Entries under `deny` → policy failed

---

## Final Verification

### Folder check

```bash
ls -la "$RUN_PATH/logs"
ls -la "$RUN_PATH/reports"
ls -la "$RUN_PATH/workspaces/plan"
```

### Quick stage summary

```bash
for f in "$RUN_PATH/reports/"*.json; do
  echo "----- $f"
  head -n 25 "$f"
done
```

---

## Fast Demo (5-Minute Flow)

```bash
cd ~/devsecops360/terraform-assessment

# Set ARM_* or use az login
export ARM_SUBSCRIPTION_ID="your-sub"

# Trigger
cat <<'JSON' | python3 -m app.api_entry
{
  "scenario": "from-cloud",
  "client": "ttms",
  "resources": "resource_group,virtual_network",
  "cloud": { "resource_group": "ttms-prod-rg" }
}
JSON

# Follow RUN_ID → inspect runs/<run_id>/
```

---

## Status

✅ Full from-cloud pipeline locked and verified:

**Terraformer → Prepare Workspace → Terraform Plan → tfsec → OPA**
