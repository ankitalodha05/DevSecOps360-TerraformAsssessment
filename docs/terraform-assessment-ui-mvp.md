# Terraform Assessment – UI Dashboard (Phase-1 MVP)

## Objective

Build a **simple, read-only dashboard UI** for the **Terraform Assessment** product that allows users to:

- Start an infrastructure assessment
- View assessment run status
- Review security, plan, and policy results
- Download assessment artifacts

⚠️ **This is NOT a Terraform editor or deployment UI**  
⚠️ **No infrastructure apply actions**

---

## Scope (Phase-1 MVP)

- Cloud: **Azure only**
- Assessment Scope: **Network only**
  - VNet
  - Subnet
  - NSG
  - NIC
- Mode:
  - From existing cloud (default)
  - From Terraform repository (optional toggle)

---

## Screens to Build

### Screen 1: Start Assessment

**Purpose:** Collect minimal inputs and trigger assessment.

#### Fields
- **Client ID** (text, required)  
  Example: `ttms`

- **Cloud Provider** (dropdown)  
  - Azure (enabled)
  - AWS / GCP (disabled, future)

- **Subscription ID** (text, required)

- **Assessment Scope** (dropdown)
  - Network (only option for MVP)

- **Assessment Mode** (radio)
  - From Existing Cloud (default)
  - From Terraform Repository

- **Repository Path** (text)  
  - Visible **only if** mode = From Terraform Repository

- **Run Name** (optional)
  - Auto-generated if empty

#### Actions
- **Start Assessment** (primary button)
- Reset

#### Validation
- `client_id` and `subscription_id` are mandatory
- `repo_path` mandatory if mode = from-repo

---

### Screen 2: Assessment Runs List

**Purpose:** Show all assessment runs.

#### Table Columns
- Run ID
- Client ID
- Cloud
- Scope
- Status (badge)
  - `IN_PROGRESS`
  - `POLICY_PASSED`
  - `POLICY_FAILED`
  - `FAILED`
- Started At
- Completed At

#### Actions
- Click row → open Run Details
- Refresh list

---

### Screen 3: Run Details

**Purpose:** Display detailed assessment results.

#### Header
- Run ID
- Client ID
- Cloud
- Scope
- Status badge
- Start time
- End time
- Duration

---

## Run Details Tabs

### Tab 1: Summary (Default)

- Overall Status: Passed / Failed
- Drift Detected: Yes / No
- Security Issues Count
- Policy Result: Passed / Failed

---

### Tab 2: Security Findings (tfsec)

- Severity breakdown:
  - Critical
  - High
  - Medium
  - Low
- Findings table:
  - Rule ID
  - Severity
  - Resource / File
  - Message

---

### Tab 3: Planned Infrastructure Changes (Terraform Plan)

- Changes summary:
  - Add
  - Modify
  - Delete (highlight red if > 0)
- Optional expandable list:
  - Resource addresses

---

### Tab 4: Policy Evaluation (OPA)

- Policy Result: Passed / Failed
- Deny Count
- Deny Messages (list)

Plain-English explanation example:
> “This assessment failed because deleting existing infrastructure is not allowed by policy.”

---

### Tab 5: Downloads (Advanced)

Download links for:
- `tfsec.json`
- `tfplan.json`
- `opa_decision.json`
- `metadata.json`
- Logs:
  - `tfsec.log`
  - `tf_plan.log`
  - `opa.log`

---

## Backend API Contract (MVP)

### Start Assessment
**POST** `/runs`

```json
{
  "client_id": "ttms",
  "cloud": "azure",
  "subscription_id": "<subscription-id>",
  "scope": "network",
  "mode": "from-cloud",
  "repo_path": null,
  "run_name": "optional"
}
````

Response:

```json
{
  "run_id": "2026-01-23T...",
  "status": "IN_PROGRESS"
}
```

---

### List Runs

**GET** `/runs`

Returns list of runs with summary metadata.

---

### Get Run Details

**GET** `/runs/{run_id}`

Returns:

* Run metadata
* Stage summaries (tfsec, plan, opa)
* Artifact download paths / URLs

---

### Download Artifacts

**GET** `/runs/{run_id}/artifacts/{artifact_name}`

Examples:

* `tfsec.json`
* `tfplan.json`
* `opa_decision.json`
* `metadata.json`
* `logs/tf_plan.log`

---

## UX Rules (Must Follow)

* Read-only UI
* Clear status indicators
* Explain results in simple language
* No Terraform apply
* No inline code editing
* No credential exposure
* No dashboards/graphs for MVP

---

## Definition of Done (UI MVP)

* User can start an assessment
* User can see run status in list
* User can open run details
* User can review summary, findings, plan, policy
* User can download assessment artifacts

---

## Notes

This UI is designed as a **security and compliance assessment report**, not a DevOps control panel.
Focus on clarity, explainability, and audit-readiness.

Future phases may add:

* Multi-cloud
* More scopes
* Dashboards
* AI recommendations

