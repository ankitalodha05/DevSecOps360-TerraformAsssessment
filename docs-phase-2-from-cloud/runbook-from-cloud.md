# Runbook: From-Cloud Terraform Assessment Pipeline

**Terraformer → Normalize Workspace → Terraform Plan → tfsec → OPA (tfplan)**

---

## Purpose

This runbook documents the complete **from-cloud** execution flow of the Terraform Assessment pipeline.

It demonstrates how existing Azure infrastructure is:

1. Exported using Terraformer (scoped to a Resource Group)
2. Normalized into a clean, plan-ready Terraform workspace
3. Planned safely (no backend, no apply)
4. Security-scanned using tfsec
5. Policy-validated using OPA against the Terraform execution plan

All execution artifacts are isolated under a single run folder:

```
runs/<run_id>/
```

This runbook is suitable for:

* Live demos
* Knowledge transfer
* Future troubleshooting
* CI/CD execution reference
* Interview and architectural explanation

---

## Design Guarantees

This pipeline is intentionally designed to be:

* Read-only against cloud resources (no apply)
* Backend-free (no remote state, no locking)
* Fully reproducible per run via isolated run folders
* Safe for demos, audits, and CI execution
* Deterministic via pinned providers and normalized input

At no stage does this pipeline modify live infrastructure.

---

## Preconditions

### Project root

```bash
cd ~/devsecops360/terraform-assessment
```

---

### Required tools

* Python 3
* Terraform
* Terraformer
* tfsec
* OPA

---

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

---

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

Terraformer export is **intentionally scoped to a Resource Group** to ensure controlled, predictable exports.

---

### JSON input (IMPORTANT FIX #1)

```json
"cloud": {
  "resource_group": "ttms-prod-rg"
}
```

Python maps this internally to:

```
--resource-group=ttms-prod-rg
```

---

### Trigger command (copy-paste)

```bash
cat <<'JSON' | python3 -m app.api_entry
{
  "scenario": "from-cloud",
  "client": "ttms",
  "resources":
"resource_group,virtual_network,subnet,route_table,network_security_group,network_interface,public_ip,load_balancer,application_gateway,nat_gateway,virtual_machine,managed_disk,storage_account,container_registry,container_registry_webhook,app_service,app_service_plan,key_vault,private_endpoint,private_dns_zone,ssh_public_key",
  "cloud": {
    "resource_group": "ttms-aks"
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

---

### Logs

```bash
tail -n 80 "$RUN_PATH/logs/terraformer.log"
```

---

### Artifacts

Terraformer creates a `generated/` folder under its output root.

```bash
find "$RUN_PATH/artifacts/terraformer" -maxdepth 4 -type d | head
find "$RUN_PATH/artifacts/terraformer" -name "*.tf" | head -n 30
```
### Quick proof key resources are exported (if present in RG)

```bash
grep -R 'resource "azurerm_windows_virtual_machine"\|resource "azurerm_linux_virtual_machine"\|resource "azurerm_virtual_machine"' -n "$RUN_PATH/artifacts/terraformer" | head
grep -R 'resource "azurerm_network_interface"' -n "$RUN_PATH/artifacts/terraformer" | head
grep -R 'resource "azurerm_public_ip"' -n "$RUN_PATH/artifacts/terraformer" | head
grep -R 'resource "azurerm_private_endpoint"' -n "$RUN_PATH/artifacts/terraformer" | head
```
---

## Stage 1.5: Prepare Terraform Plan Workspace

Terraformer output is **not directly plan-safe for Azure**.

This stage normalizes the generated code by:

* Injecting required `azurerm` provider `features {}` block
* Patching known AzureRM issues (e.g. invalid flow timeout)
* Pinning provider source and version
* Running `terraform fmt` for clean formatting
* Producing a deterministic, plan-ready workspace

This prevents false failures during `terraform init` and `terraform plan`.

---

### Verify stage report

```bash
cat "$RUN_PATH/reports/stage1_5_prepare_plan_workspace.json"
```

Expected:

* `"status": "PASSED"`
* No reported patch errors

---

### Verify workspace contents

```bash
find "$RUN_PATH/workspaces/plan" -name "*.tf" | head -n 30
```

---

## Terraform Init and Plan (Safe Mode)

All Terraform commands run inside:

```
runs/<run_id>/workspaces/plan
```

---

### Terraform init

```bash
cd "$RUN_PATH/workspaces/plan"

terraform init \
  -input=false \
  -backend=false \
  -no-color | tee "$RUN_PATH/logs/terraform_init.log"
```

---

### Terraform plan

```bash
terraform plan \
  -input=false \
  -no-color \
  -out plan.tfplan | tee "$RUN_PATH/logs/terraform_plan.log"
```

---

### Convert plan to JSON (for OPA)

```bash
terraform show -json plan.tfplan > plan.json
ls -la plan.tfplan plan.json
```

---

## tfsec Scan (CI-Friendly)

tfsec performs **static security analysis** on Terraform code.

---

### IMPORTANT FIX #2

Use `--no-colour` to avoid ANSI characters in CI logs.

---

### Run tfsec

```bash
cd "$RUN_PATH/workspaces/plan"

tfsec . \
  --format json \
  --no-colour \
  > "$RUN_PATH/reports/tfsec.json" \
  2> "$RUN_PATH/logs/tfsec.log" || true
```

---

### Review results

```bash
jq '.results | length' "$RUN_PATH/reports/tfsec.json" 2>/dev/null || head -n 40 "$RUN_PATH/reports/tfsec.json"
tail -n 60 "$RUN_PATH/logs/tfsec.log"
```

---

## OPA Policy Check (Terraform Plan)

OPA validates the **actual Terraform execution plan**, not static code.

This ensures policies evaluate **what Terraform would create or modify**, not just what is written.

---

### Run OPA

```bash
cd "$RUN_PATH/workspaces/plan"

opa eval \
  --format pretty \
  --data "$PWD/../../../app/policies/tfplan" \
  --input plan.json \
  "data.tfplan.deny" | tee "$RUN_PATH/logs/opa_tfplan.log"
```

---

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

---

### Quick stage summary

```bash
for f in "$RUN_PATH/reports/"*.json; do
  echo "----- $f"
  head -n 25 "$f"
done
```

---

## Common Failures & Fixes

| Issue                          | Root Cause                    | Fixed In            |
| ------------------------------ | ----------------------------- | ------------------- |
| ResourceGroupNotFound          | Terraformer not scoped        | Stage 1 input       |
| Missing azurerm features block | Terraformer output incomplete | Stage 1.5           |
| Invalid flow timeout           | AzureRM schema mismatch       | Stage 1.5           |
| terraform fmt rc=3             | Unformatted files             | Stage 1.5           |
| ANSI characters in CI logs     | tfsec color output            | tfsec `--no-colour` |

---

## Fast Demo (5-Minute Flow)

```bash
cd ~/devsecops360/terraform-assessment

# Authenticate
export ARM_SUBSCRIPTION_ID="your-subscription-id"

# Trigger
cat <<'JSON' | python3 -m app.api_entry
{
  "scenario": "from-cloud",
  "client": "ttms",
  "resources": "resource_group,virtual_network,subnet,route_table,network_security_group,network_interface,public_ip,load_balancer,application_gateway,nat_gateway,virtual_machine,managed_disk,storage_account,container_registry,container_registry_webhook,app_service,app_service_plan,key_vault,private_endpoint,private_dns_zone,ssh_public_key",
  "cloud": { "resource_group": "ttms-aks" }
}
JSON

# Follow RUN_ID → inspect runs/<run_id>/
```

---

## Status

✅ **From-cloud Terraform Assessment pipeline fully locked and verified**

**Terraformer → Normalize Workspace → Terraform Plan → tfsec → OPA**
