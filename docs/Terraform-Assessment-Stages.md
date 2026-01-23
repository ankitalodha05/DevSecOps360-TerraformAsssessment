# Terraform Assessment – Demo Guide

## Demo Commands for Latest Run

```bash
RUN_ID=$(ls -1 runs | tail -n 1)

cat "runs/$RUN_ID/metadata/metadata.json" | jq .status
ls -R "runs/$RUN_ID/reports" | sed -n '1,120p'
cat "runs/$RUN_ID/reports/opa/opa_decision.json" | jq .
````

---

## Terraform Assessment – What Each Stage Does

### Stage 0 – INIT

This stage creates a unique run ID and the folder structure for the assessment.
It also stores basic metadata such as run time and current status for audit and tracking.

**Files to show:**

* `runs/<run_id>/metadata/metadata.json`

**Latest run ID command:**

```bash
ls -1 runs | tail -n 1
```

---

### Stage 2 – tfsec (Security Scan)

This stage scans the Terraform code for security misconfigurations such as open access, insecure settings, or bad practices.
No infrastructure is deployed in this stage.

**Files to show:**

* `runs/<run_id>/reports/iac/tfsec.json`
* `runs/<run_id>/logs/tfsec.log`

---

### Stage 5 – Terraform Validate

This stage runs `terraform fmt`, `terraform init`, and `terraform validate` to ensure the Terraform code is correctly formatted and syntactically valid.

**Files to show:**

* `runs/<run_id>/reports/tf/validate_summary.json`
* `runs/<run_id>/logs/tf_validate.log`

---

### Stage 6 – Terraform Plan

This stage runs `terraform plan` to understand what changes Terraform would make if applied.
The plan is converted into JSON format so it can be analyzed by policy tools.
No `terraform apply` happens here.

**Files to show:**

* `runs/<run_id>/reports/tf/tfplan.json`
* `runs/<run_id>/reports/tf/tfplan.bin`
* `runs/<run_id>/logs/tf_plan.log`

---

### Stage 7 – OPA Policy Check

This stage evaluates the Terraform plan JSON against OPA policies.
Currently, the policy blocks any plan that tries to delete existing resources.
Based on this evaluation, the assessment is marked as PASSED or FAILED.

**Files to show:**

* `runs/<run_id>/reports/opa/opa_decision.json`
* `runs/<run_id>/logs/opa.log`
* `runs/<run_id>/metadata/metadata.json`

---

## One-line Summary

Terraform code is scanned for security issues, validated for correctness, planned to understand impact, and then evaluated with OPA policies to block destructive changes before apply.
