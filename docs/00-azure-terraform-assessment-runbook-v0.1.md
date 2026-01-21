# DevSecOps360 — Azure Terraform Assessment Runbook (MVP v0.1)

**Version:** 0.1
**Scope:** Azure-only, Scenario A + B, MVP focuses on: **Terraformer + tfsec + Terraform plan + OPA + Report**
**Primary goal (MVP):** Start from Azure infra (or Terraform repo) → produce **assessment_summary.json** and **terraform plan json + opa decision json**
**Non-goals (MVP):** No apply, no automatic remediation in cloud, ScoutSuite optional (post-MVP)

---

## 0) What this pipeline does (simple)

DevSecOps360 runs a workflow that can start from:

* **Scenario A (From Cloud):** existing Azure subscription (no Terraform)
* **Scenario B (From Repo):** existing Terraform code

Then it:

1. Produces Terraform artifacts (Scenario A only)
2. Scans Terraform code (tfsec)
3. Normalizes/cleans Terraform (your product brain; can be minimal in MVP)
4. Runs terraform fmt/validate + terraform plan
5. Runs OPA policy gate on plan JSON
6. Builds a final JSON report for API/UI

---

## 1) Inputs needed for every run

### Common inputs

* `client_id` (example: `ttms`)
* `scenario`: `from-cloud` or `from-repo`
* `run_mode`: `assessment`
* `scope`: what we assess (MVP: **network only**)
* `run_id`: generated

### Scenario A specific inputs (Azure from cloud)

* Azure auth context:

  * Azure DevOps service connection / service principal / workload identity
* `subscription_id`
* optional: resource group list, services filter

### Scenario B specific inputs (Terraform repo)

* `repo_url` or local path
* branch/ref

---

## 2) Stable run directory layout (must not change)

All outputs for a run live under:

`/runs/{run_id}/`

### Standard folders

* `metadata/` — run metadata, input params
* `artifacts/raw/` — immutable raw dumps (Terraformer)
* `artifacts/staging/` — normalized + intermediate outputs
* `artifacts/production/` — final clean outputs (modules + client tfvars)
* `reports/` — tfsec, opa, final summary, graph
* `logs/` — logs per stage

### Example

```
/runs/2026-01-21T170500Z-ttms/
  metadata/metadata.json
  artifacts/raw/terraformer/...
  artifacts/staging/normalized/...
  artifacts/staging/tfplan/tfplan.json
  artifacts/production/modules/network/...
  artifacts/production/clients/ttms/terraform.tfvars
  reports/iac/tfsec.json
  reports/policy/opa_decision.json
  reports/final/assessment_summary.json
  logs/*.log
```

---

## 3) Workflow stages (end-to-end)

## Stage 0 — INIT RUN

### Purpose

Create a run workspace and persist inputs.

### Steps

1. Generate `run_id`
2. Create directory structure under `/runs/{run_id}/`
3. Write `metadata.json` with request + derived info

### Outputs

* `metadata/metadata.json`
* `logs/init.log`

### Fail if

* invalid request
* cannot create workspace

---

## Stage 1 — DISCOVER INFRA (Terraformer) [Scenario A only]

### Purpose

Export Azure infra into raw Terraform files + tfstate.

### MVP boundaries (Azure)

* Terraformer Azure provider
* **Network scope only** (start with your Phase-1 resources)

### Steps

1. Authenticate to Azure (via service connection or SP)
2. Run Terraformer for Azure subscription + scope filters
3. Collect outputs into:

   * `artifacts/raw/terraformer/...`
4. Do **not edit** raw outputs

### Outputs

* `artifacts/raw/terraformer/**.tf`
* `artifacts/raw/terraformer/terraform.tfstate`
* `logs/terraformer.log`

### Fail if

* auth fails
* no output produced

---

## Stage 2 — IAC SCAN (tfsec) [Scenario A + B]

### Purpose

Static Terraform security scan.

### MVP rule

* Scan **raw** (Scenario A) or **repo** (Scenario B)
* Later: scan staging/production too

### Steps

1. Identify scan target directory
2. Run tfsec
3. Save results JSON

### Outputs

* `reports/iac/tfsec.json`
* `logs/tfsec.log`

### Fail if

* tool error / invalid output
* (optional policy) high severity threshold exceeded

---

## Stage 3 — CLOUD SCAN (ScoutSuite) [Post-MVP optional]

### Purpose

Runtime cloud posture scan (separate from IaC scan).

### MVP

Skip by default.

### Outputs (when enabled)

* `reports/cloud/scoutsuite.json` (+ optional HTML)
* `logs/scoutsuite.log`

---

## Stage 4 — NORMALIZE + AI CLEANUP (Product brain)

### Purpose

Convert raw Terraform (or repo Terraform) into consistent, reusable, reviewable Terraform.

### MVP version (minimal but real)

* standard formatting, naming normalization
* mapping keys to stable identifiers
* generation of client tfvars maps (for your network module)

### Steps (MVP)

1. Copy from raw/repo into staging workspace
2. Apply normalization rules (naming/tags)
3. Generate `production/modules/...` and `production/clients/...`
4. Ensure no hard-coded secrets in output (goal)

### Outputs

* `artifacts/staging/normalized/...`
* `artifacts/production/modules/...`
* `artifacts/production/clients/{client_id}/terraform.tfvars`
* `logs/normalize.log`

### Fail if

* mappings missing for required resources
* output invalid

---

## Stage 5 — TF VALIDATE

### Purpose

Ensure cleaned Terraform is valid.

### Steps

1. `terraform fmt` (write changes)
2. `terraform validate`

### Outputs

* `logs/terraform_fmt.log`
* `logs/terraform_validate.log`

### Fail if

* validate fails

---

## Stage 6 — PLAN AND OPA (policy gate)

### Purpose

Generate Terraform plan and evaluate with OPA.

### Required behavior

* Even if OPA denies, artifacts MUST be written for debugging.

### Steps

1. Run `terraform plan` and save plan file
2. Convert plan to JSON
3. Run OPA evaluation using policy bundle
4. Produce decision JSON (allow/deny + reasons)

### Outputs

* `artifacts/staging/tfplan/tfplan.json`
* `reports/policy/opa_decision.json`
* `logs/terraform_plan.log`
* `logs/opa_eval.log`

### Fail if

* plan fails
* OPA tool error
* (policy) deny in strict mode

---

## Stage 7 — INFRA GRAPH (optional in MVP)

### Purpose

Generate resource graph from tfstate/plan.

### MVP

Optional. Can be added after pipeline works end-to-end.

### Outputs

* `reports/graph/graph.json` or svg
* `logs/graph.log`

---

## Stage 8 — REPORT (final)

### Purpose

Aggregate everything into one stable JSON response for UI/API.

### Steps

1. Read tfsec results
2. Read OPA decision
3. Include key artifact references
4. Create `assessment_summary.json`

### Outputs

* `reports/final/assessment_summary.json`
* `logs/report.log`

### Fail if

* required inputs missing
* schema invalid

---

## 4) API contract (minimal for MVP)

### POST start run

`POST /runs`

* returns `{ run_id }` immediately (async)

### GET status

`GET /runs/{run_id}/status`

* `QUEUED | RUNNING | FAILED | COMPLETED`

### GET results

`GET /runs/{run_id}/results`

* returns `assessment_summary.json` content

---

## 5) MVP policy set (OPA)

### MVP policy goal

Demonstrate policy gate works.

Example MVP policy ideas (Azure):

* Deny any NSG rule allowing inbound from `0.0.0.0/0` to port 22
* Deny public IP association on NICs in “private” subnet (if you track it)

*(Policies can evolve; MVP needs only one deny rule for demo.)*

---

## 6) What “done” looks like (acceptance criteria)

A run is successful when:

* `reports/final/assessment_summary.json` exists
* `artifacts/staging/tfplan/tfplan.json` exists
* `reports/policy/opa_decision.json` exists
* tfsec report exists
* status = `COMPLETED` (or `FAILED` with artifacts for debug)

---

## 7) Step-by-step implementation plan (what we do next)

We will implement in this order:

1. Repo scaffolding + docs committed
2. Workspace + run_id creation (Stage 0)
3. Scenario B first (repo → tfsec → plan → OPA → report)
   *(faster to validate pipeline wiring)*
4. Add Scenario A discovery (Terraformer)
5. Add Normalize minimal rules for network module
6. Add graph + ScoutSuite later

---
