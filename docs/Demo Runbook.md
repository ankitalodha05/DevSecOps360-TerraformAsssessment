# DevSecOps360 – Terraform Assessment (Demo Runbook)

This document is a step-by-step runbook to demonstrate the Terraform Assessment workflow end-to-end in a demo.

It covers:
- How an assessment run is triggered
- How runs are uniquely identified and stored
- Where each tool’s output is generated
- How to verify security, plan, and policy results
- How to interpret the final assessment status

---

> **Deep Dive (Optional Reading)**  
> Detailed explanation of tfsec, its role in the assessment pipeline, and how it differs from OPA and Scout Suite:  
> [tfsec – Complete Security Guide](tfsec-overview-and-role-in-terraform-assessment.md)

---

## 1) Prerequisites (Before Demo)

### 1.1 Required tools

Verify that the following tools are installed:

```bash
python3 --version
terraform --version
jq --version
````

If any command is not found, install the tool before starting the demo.

---

### 1.2 Repository location

Navigate to the Terraform Assessment repository root:

```bash
cd ~/devsecops360/terraform-assessment
pwd
ls -la
```

Expected:

* `app/` → pipeline code
* `examples/` → sample Terraform inputs
* `runs/` → assessment outputs (created after trigger)

---

## 2) High-Level Execution Flow

When a run is triggered:

1. A unique `run_id` is generated
2. A dedicated folder is created under `runs/<run_id>/`
3. Tools execute sequentially:

   * tfsec (security scan)
   * terraform plan (no apply)
   * OPA (policy evaluation)
4. Results are written as JSON artifacts
5. Final status is updated in metadata

---

## 3) Run Output Structure

Each execution creates a dedicated folder:

```
runs/<run_id>/
  metadata/
    metadata.json
  reports/
    iac/
      tfsec.json
    tf/
      tfplan.json
    opa/
      opa_decision.json
  logs/
    *.log
```

Purpose of each file:

* `metadata.json` → overall run status and execution details
* `tfsec.json` → security scan findings
* `tfplan.json` → terraform plan in JSON format
* `opa_decision.json` → policy evaluation result
* `logs/` → execution logs for troubleshooting

---

## 4) Demo Scenario A: from-repo (Full Pipeline)

This is the recommended demo scenario.

It executes:

* tfsec
* terraform plan
* OPA policy checks

and produces all assessment artifacts.

---

### 4.1 Verify input Terraform code

The demo uses sample Terraform code located at:

```bash
ls -la examples/tf_local
```

Expected:

* Terraform configuration files such as `main.tf`, `providers.tf`, etc.

Screenshot to capture:

* Screenshot A1: `ls -la examples/tf_local`

---

## 5) Trigger the Assessment Run

### 5.1 Start a new run

```bash
cat <<'JSON' | python3 -m app.api_entry
{
  "scenario": "from-repo",
  "client": "ttms",
  "repo_path": "examples/tf_local"
}
JSON
```

**What this command does:**

* Sends a JSON request to the pipeline entrypoint via standard input
* Starts a new Terraform Assessment using the specified Terraform code

**Expected output:**

```json
{"run_id":"<run_id>","status":"IN_PROGRESS"}
```

**What this output means:**

* `run_id` uniquely identifies this execution
* `IN_PROGRESS` indicates the pipeline has started and continues running asynchronously

Screenshot to capture:

* Screenshot A2: trigger command and returned JSON

---

## 6) Locate the Run Folder

### 6.1 Store the run ID in a variable

```bash
RUN_ID=<run_id_returned_by_trigger>
echo $RUN_ID
```

**Purpose:**

* Stores the run identifier in a variable for reuse
* Prevents repeated copy-paste errors
* Keeps subsequent commands clean and consistent

---

### 6.2 Confirm run folder creation

```bash
ls -la "runs/$RUN_ID"
```

Expected:

* `metadata/`
* `reports/`
* `logs/`

Screenshot to capture:

* Screenshot A3: run folder visible under `runs/`

---

## 7) Check Run Status (Metadata)

### 7.1 View complete metadata

```bash
cat "runs/$RUN_ID/metadata/metadata.json" | jq .
```

### 7.2 View only the status field

```bash
cat "runs/$RUN_ID/metadata/metadata.json" | jq .status
```

Possible final statuses:

* `POLICY_PASSED` → all checks passed
* `POLICY_FAILED` → blocked by policy
* `FAILED` → execution error

Screenshot to capture:

* Screenshot A4: metadata.json with status

---

## 8) Verify Artifact Generation

### 8.1 Confirm output files exist

```bash
ls "runs/$RUN_ID/reports/iac/tfsec.json"
ls "runs/$RUN_ID/reports/tf/tfplan.json"
ls "runs/$RUN_ID/reports/opa/opa_decision.json"
```

If paths are printed without errors, all stages completed successfully.

Screenshot to capture:

* Screenshot A5: artifact file checks

---

## 9) Review Tool Outputs

### 9.1 tfsec – Security Scan Results

```bash
cat "runs/$RUN_ID/reports/iac/tfsec.json" | jq .
```

Optional:

```bash
cat "runs/$RUN_ID/reports/iac/tfsec.json" | jq '.results | length'
```

Interpretation:

* Empty results array means no security issues were detected

Screenshot:

* Screenshot A6: tfsec output

---

### 9.2 Terraform Plan (Read-only)

```bash
cat "runs/$RUN_ID/reports/tf/tfplan.json" | jq .
```

Key points to highlight:

* Resources detected
* No apply performed
* Plan generated successfully

Optional:

```bash
cat "runs/$RUN_ID/reports/tf/tfplan.json" | jq '.resource_changes | length'
```

Screenshot:

* Screenshot A7: terraform plan summary

---

### 9.3 OPA Policy Decision

```bash
cat "runs/$RUN_ID/reports/opa/opa_decision.json" | jq .
```

Highlight during demo:

* `allow: true` / `passed: true`
* Empty `deny` list

Screenshot:

* Screenshot A8: OPA decision output

---

## 10) Logs (Optional Validation)

### 10.1 List logs

```bash
ls -la "runs/$RUN_ID/logs"
```

### 10.2 Open a log file

```bash
cat "runs/$RUN_ID/logs/tfsec.log" | sed -n '1,120p'
```

Screenshot:

* Screenshot A9: logs directory and sample log output

---

## 11) Demo Scenario B: from-cloud (Init Only)

This scenario currently:

* Creates run folder and metadata
* Does not yet generate tfsec, plan, or OPA outputs

### 11.1 Trigger from-cloud

```bash
cat <<'JSON' | python3 -m app.api_entry
{
  "scenario": "from-cloud",
  "client": "ttms",
  "subscription_id": "SUBSCRIPTION_ID_HERE",
  "tenant_id": "TENANT_ID_HERE"
}
JSON
```

### 11.2 Verify run

```bash
RUN_ID=$(ls -1 runs | tail -n 1)
ls -la "runs/$RUN_ID"
cat "runs/$RUN_ID/metadata/metadata.json" | jq .
```

Screenshot:

* Screenshot B1: from-cloud run metadata

---

## 12) Demo Summary

This demo shows that:

* The pipeline can be triggered programmatically
* Each execution is isolated using a unique run ID
* Security, plan, and policy checks run without applying infrastructure
* Results are stored as structured, machine-readable JSON
* Final outcome is clearly recorded in metadata

---

## Appendix: Quick Demo Commands

### Trigger run

```bash
cat <<'JSON' | python3 -m app.api_entry
{
  "scenario": "from-repo",
  "client": "ttms",
  "repo_path": "examples/tf_local"
}
JSON
```

### Verify outputs

```bash
cat "runs/$RUN_ID/metadata/metadata.json" | jq .status
ls "runs/$RUN_ID/reports/iac/tfsec.json"
ls "runs/$RUN_ID/reports/tf/tfplan.json"
ls "runs/$RUN_ID/reports/opa/opa_decision.json"
```
