# Terraform Assessment – UI Dashboard (Phase-1 MVP)

## Objective

Build a **simple, read-only dashboard UI** for the **Terraform Assessment** product that allows users to:

- Start an infrastructure assessment
- Track assessment runs
- Review security, drift, and policy results
- Download assessment artifacts

This is **NOT** a Terraform editor or deployment UI  
No `terraform apply`, no inline editing, no manual IaC uploads

---

## Scope (Phase-1 MVP)

- **Cloud**: Azure only
- **Assessment Scope**: Network only  
  (VNet, Subnet, NSG, NIC)
- **Assessment Modes**:
  - From existing cloud (default)
  - From Terraform repository (advanced / optional)

---

## Screen 1: Start Assessment

**Purpose:** Collect minimal inputs and trigger an assessment run.

### Fields

- **Client ID** (text, required)  
  Example: `ttms`

- **Cloud Provider** (dropdown)  
  - Azure (enabled)  
  - AWS / GCP (disabled – future)

- **Subscription ID** (text, required)

- **Scope** (read-only helper text)  
  `Network (VNet, Subnet, NSG, NIC)`

- **Assessment Mode** (radio)
  - From Existing Cloud (default)
  - From Terraform Repository

- **Repository Path** (text)  
  - Visible **only if** mode = From Terraform Repository

- **Run Name** (optional)  
  - Auto-generated if empty

### Actions
- **Start Assessment** (primary)
- Reset

### Validation
- `client_id` and `subscription_id` are mandatory
- `repo_path` is mandatory if mode = from-repo

---

## Screen 2: Assessment Runs List

**Purpose:** Show all assessment runs.

### Table Columns
- Run ID
- Client ID
- Cloud
- Scope
- Status (badge)
- Started At
- Completed At

### UI Status Values (only these)
- `IN_PROGRESS`
- `POLICY_PASSED`
- `POLICY_FAILED`
- `FAILED`

### Actions
- Click row → open Run Details
- Refresh

---

## UI Status Mapping (Important)

Backend internal statuses **must NOT be shown directly**.

### Backend → UI Mapping

- `POLICY_PASSED` → `POLICY_PASSED`
- `POLICY_FAILED` → `POLICY_FAILED`
- Any status ending with `_FAILED`  
  (e.g. `TF_PLAN_FAILED`, `OPA_FAILED`) → `FAILED`
- Any other state → `IN_PROGRESS`

---

## Screen 3: Run Details

**Purpose:** Display detailed assessment results.

### Header
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

Show in clear, non-technical language:

- **Overall Status**: Passed / Failed
- **Drift Detected**: Yes / No  
  (derived from `plan.has_changes`)
- **Security Issues**: total count
- **Policy Gate**: Passed / Failed  
  (derived from OPA decision)

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

**Note:**  
For large environments, the UI should support pagination or show only the top N findings (e.g., top 20) by default, with an option to view all.

---

### Tab 3: Planned Infrastructure Changes (Terraform Plan)

- Change summary:
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

Provide download links for:

- `tfsec.json`
- `tfplan.json`
- `opa_decision.json`
- `metadata.json`
- Logs:
  - `tfsec.log`
  - `tf_plan.log`
  - `opa.log`

🔒 **Security Note:**  
Initial implementation may use direct file paths for local development.  
In production, these will be replaced with **time-limited (signed) download URLs**.

---

## Backend API Contract (MVP)

### POST /runs — Start Assessment

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

### GET /runs — List Assessment Runs

Returns a list of run summaries.

#### Sample Response

```json
[
  {
    "run_id": "2026-01-23T122848Z-ttms",
    "client_id": "ttms",
    "status": "POLICY_PASSED",
    "started_at": "2026-01-23T12:28:48Z",
    "completed_at": "2026-01-23T12:29:10Z"
  },
  {
    "run_id": "2026-01-22T094112Z-abc",
    "client_id": "abc",
    "status": "IN_PROGRESS",
    "started_at": "2026-01-22T09:41:12Z",
    "completed_at": null
  }
]
```

---

### GET /runs/{run_id} — Get Run Details

Returns a single structured object containing:

* `summary`
* `security`
* `plan`
* `policy`
* `artifacts`

#### Sample Response

```json
{
  "run_id": "2026-01-23T122848Z-ttms",
  "client_id": "ttms",
  "cloud": "azure",
  "scope": "network",
  "status": "POLICY_PASSED",
  "started_at": "2026-01-23T12:28:48Z",
  "completed_at": "2026-01-23T12:29:10Z",

  "summary": {
    "overall_status": "PASSED",
    "drift_detected": false,
    "security_issues": 0,
    "policy_gate": "PASSED"
  },

  "security": {
    "severity_breakdown": {
      "critical": 0,
      "high": 0,
      "medium": 0,
      "low": 0
    },
    "top_findings": []
  },

  "plan": {
    "has_changes": false,
    "add": 0,
    "modify": 0,
    "delete": 0
  },

  "policy": {
    "passed": true,
    "deny_count": 0,
    "deny_messages": []
  },

  "artifacts": {
    "tfsec": "runs/<id>/reports/iac/tfsec.json",
    "tfplan": "runs/<id>/reports/tf/tfplan.json",
    "opa": "runs/<id>/reports/opa/opa_decision.json",
    "metadata": "runs/<id>/metadata/metadata.json"
  }
}
```

---

### GET /runs/{run_id}/artifacts/{artifact_name} — Download Artifact

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
* Simple language (assessment-style, not DevOps tool)
* No Terraform apply
* No inline editing
* No credential exposure
* No dashboards/graphs in MVP

---

## Definition of Done (UI MVP)

* User can start an assessment
* User can see runs list
* User can open run details
* User can review summary, security, plan, policy
* User can download artifacts

---

## Notes

This UI represents a **security and compliance assessment report**, not an infrastructure management console.

Future phases may include:

* Multi-cloud support
* Additional scopes
* Dashboards
* AI recommendations

