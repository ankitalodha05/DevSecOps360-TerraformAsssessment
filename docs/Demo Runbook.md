# DevSecOps360 – Terraform Assessment (Demo Runbook)

This document is a step-by-step runbook to demo the Terraform Assessment workflow end-to-end.
It shows:
- How to trigger a run
- Where the run outputs are stored
- How to verify each stage output (tfsec, plan, OPA)
- How to interpret final status from metadata

---

## 1) Prerequisites (Before Demo)

### 1.1 Tools installed
Run these commands and confirm versions show up:

```bash
python3 --version
terraform --version
jq --version
````

If any command is not found, install it before the demo.

### 1.2 Repository location

Go to the repository root:

```bash
cd ~/devsecops360/terraform-assessment
pwd
ls -la
```

Expected: you should see folders like `app/`, `examples/`, and `runs/` (if runs already exist).

---

## 2) Quick Architecture (What happens when we run)

### 2.1 Run folder structure (output)

Every run creates a unique folder:

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

* `metadata.json` = single source of truth for overall status
* `tfsec.json` = security scan output
* `tfplan.json` = terraform plan output (no apply)
* `opa_decision.json` = policy decision (pass/fail)

---

## 3) Demo Scenario A (Recommended): from-repo (Full pipeline)

This scenario runs the full pipeline:

* tfsec
* terraform plan
* OPA policy check
  And generates all artifacts under `runs/<run_id>`.

### 3.1 Confirm input repo_path exists

This demo uses the sample Terraform code in:

```bash
ls -la examples/tf_local
```

Expected: Terraform files are present (e.g., `main.tf`, `providers.tf`, etc).

Add screenshot:

* Screenshot A1: `ls -la examples/tf_local`

---

## 4) Trigger the Run (Python trigger via stdin)

### 4.1 Start a run

Copy-paste this exactly:

```bash
cat <<'JSON' | python3 -m app.api_entry
{
  "scenario": "from-repo",
  "client": "ttms",
  "repo_path": "examples/tf_local"
}
JSON
```

Expected output:

* A JSON response on stdout like:

  * `{"run_id":"<id>","status":"IN_PROGRESS"}`

Note:

* `IN_PROGRESS` is expected because the trigger returns quickly and the pipeline writes results into `runs/<run_id>`.

Add screenshot:

* Screenshot A2: trigger command + stdout response

---

## 5) Locate the Run Folder

### 5.1 Set RUN_ID (use the one printed by the trigger)

Example:

```bash
RUN_ID=2026-01-28T083604Z-unknown-a43942
echo $RUN_ID
```

### 5.2 Confirm run folder exists

```bash
ls -la "runs/$RUN_ID"
```

Add screenshot:

* Screenshot A3: run folder present under `runs/`

---

## 6) Check Run Status (Metadata)

### 6.1 View full metadata

```bash
cat "runs/$RUN_ID/metadata/metadata.json" | jq .
```

### 6.2 Check only status

```bash
cat "runs/$RUN_ID/metadata/metadata.json" | jq .status
```

Expected final statuses (examples):

* `POLICY_PASSED` (success)
* `POLICY_FAILED` (OPA blocked)
* `FAILED` (tool execution failed)

Add screenshot:

* Screenshot A4: metadata.json and status value

---

## 7) Verify All Stage Outputs Exist

### 7.1 Check artifact files

```bash
ls "runs/$RUN_ID/reports/iac/tfsec.json"
ls "runs/$RUN_ID/reports/tf/tfplan.json"
ls "runs/$RUN_ID/reports/opa/opa_decision.json"
```

Expected: all paths print successfully (no “No such file”).

Add screenshot:

* Screenshot A5: file existence checks

---

## 8) Demo Stage Outputs (Show What Each Tool Produced)

## 8.1 Stage: tfsec (Security Scan)

### Open the tfsec report

```bash
cat "runs/$RUN_ID/reports/iac/tfsec.json" | jq .
```

### Optional: Count findings (if schema has `results`)

```bash
cat "runs/$RUN_ID/reports/iac/tfsec.json" | jq '.results | length'
```

Add screenshot:

* Screenshot A6: tfsec report output / findings count

---

## 8.2 Stage: Terraform Plan (No Apply)

### Open plan JSON

```bash
cat "runs/$RUN_ID/reports/tf/tfplan.json" | jq .
```

### Optional: Count resource changes (if present)

```bash
cat "runs/$RUN_ID/reports/tf/tfplan.json" | jq '.resource_changes | length'
```

Add screenshot:

* Screenshot A7: tfplan.json summary

---

## 8.3 Stage: OPA Policy Decision

### Open OPA decision JSON

```bash
cat "runs/$RUN_ID/reports/opa/opa_decision.json" | jq .
```

What to highlight in demo:

* Whether policy allowed or denied
* Any deny messages (if present)

Add screenshot:

* Screenshot A8: opa_decision.json output

---

## 9) Show Logs (Optional, for troubleshooting proof)

### 9.1 List logs

```bash
ls -la "runs/$RUN_ID/logs"
```

### 9.2 Open a specific log (example)

```bash
ls -1 "runs/$RUN_ID/logs" | head -n 20
```

Then open relevant file (example):

```bash
cat "runs/$RUN_ID/logs/tfsec.log" | sed -n '1,120p'
```

Add screenshot:

* Screenshot A9: logs folder + first lines of a log

---

## 10) Demo Scenario B: from-cloud (Current behavior: init only)

This scenario currently runs only init stage (as per workflow).
It creates run folder and metadata, but does not generate tfsec/plan/opa outputs yet.

### 10.1 Trigger from-cloud

Replace placeholders with actual values if required by your workflow:

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

### 10.2 Verify output

```bash
RUN_ID=$(ls -1 runs | tail -n 1)
echo "$RUN_ID"
ls -la "runs/$RUN_ID"
cat "runs/$RUN_ID/metadata/metadata.json" | jq .
```

Expected:

* Run folder exists
* metadata exists
* reports for tfsec/plan/opa may not exist yet (init-only)

Add screenshot:

* Screenshot B1: from-cloud trigger + metadata verification

---

## 11) Common Issues and Quick Fixes

### 11.1 Status remains IN_PROGRESS for long time

Check logs:

```bash
ls -la "runs/$RUN_ID/logs"
```

And open the latest log:

```bash
LATEST_LOG=$(ls -1t "runs/$RUN_ID/logs" | head -n 1)
echo "$LATEST_LOG"
cat "runs/$RUN_ID/logs/$LATEST_LOG" | tail -n 120
```

### 11.2 Artifact file missing

Verify the folder tree:

```bash
ls -R "runs/$RUN_ID" | sed -n '1,200p'
```

### 11.3 jq not installed

Install on Ubuntu:

```bash
sudo apt-get update
sudo apt-get install -y jq
```

---

## 12) Demo Wrap-up (What was proven)

* Trigger accepts JSON request via stdin and returns a run_id
* Run outputs are stored in a traceable folder structure under `runs/<run_id>`
* Pipeline generates machine-readable JSON:

  * tfsec report
  * terraform plan output
  * OPA policy decision
* Final status is recorded in metadata (example: POLICY_PASSED)

---

## Appendix: Quick Demo Command Pack (Copy/Paste)

### A) Trigger run (from-repo)

```bash
cat <<'JSON' | python3 -m app.api_entry
{
  "scenario": "from-repo",
  "client": "ttms",
  "repo_path": "examples/tf_local"
}
JSON
```

### B) Set RUN_ID (replace with your output)

```bash
RUN_ID=REPLACE_WITH_RUN_ID
```

### C) Verify status + outputs

```bash
cat "runs/$RUN_ID/metadata/metadata.json" | jq .status
ls "runs/$RUN_ID/reports/iac/tfsec.json"
ls "runs/$RUN_ID/reports/tf/tfplan.json"
ls "runs/$RUN_ID/reports/opa/opa_decision.json"
```

### D) Open reports

```bash
cat "runs/$RUN_ID/reports/iac/tfsec.json" | jq .
cat "runs/$RUN_ID/reports/tf/tfplan.json" | jq .
cat "runs/$RUN_ID/reports/opa/opa_decision.json" | jq .
```

