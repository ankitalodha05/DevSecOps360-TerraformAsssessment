# Terraform Assessment – Backend Trigger & API Contract (Phase-1 MVP)

## Purpose

This document defines how the **NodeJS backend** should trigger the **Python Terraform Assessment pipeline** and expose APIs for the **React UI dashboard**.

This is **Phase-1 MVP**:
- Read-only UI
- No terraform apply
- No inline editing
- Azure + Network scope only

---

## High-Level Architecture

```

React UI
↓ (REST API)
NodeJS Backend
↓ (exec command)
Python Assessment Engine
↓
runs/<run_id>/...

```

### Responsibilities
- **Python** → actual assessment engine (creates runs folder)
- **NodeJS** → API layer (trigger + read results)
- **React** → UI only (talks to Node APIs)

---

## Decision (Locked)

**Option A selected:**  
`POST /runs` will **trigger the real Python pipeline** and return a real `run_id`.

---

## Runtime Data Source

### Base Directory
```

RUNS_BASE_PATH

```

- Default: `./runs`
- On server: `~/devsecops360/terraform-assessment/runs`

---

## Per-Run Folder Structure

```

runs/<run_id>/
├── metadata/
│   └── metadata.json
├── reports/
│   ├── iac/tfsec.json
│   ├── tf/tfplan.json
│   └── opa/opa_decision.json
└── logs/
├── tfsec.log
├── tf_plan.log
└── opa.log

```

---

## Python Wrapper (Pipeline Entry)

### File
```

app/api_entry.py

````

### Purpose
- Accept JSON input from Node (stdin)
- Call existing assessment function
- Print JSON to stdout (Node will parse)

### Code

```python
import json
import sys
from app.main import run_terraform_assessment

def main():
    input_data = json.loads(sys.stdin.read())

    result = run_terraform_assessment(
        client_id=input_data["client_id"],
        cloud=input_data.get("cloud", "azure"),
        subscription_id=input_data["subscription_id"],
        scope=input_data.get("scope", "network"),
        mode=input_data.get("mode", "from-cloud"),
        repo_path=input_data.get("repo_path"),
        run_name=input_data.get("run_name")
    )

    print(json.dumps({
        "run_id": result["run_id"],
        "status": "IN_PROGRESS"
    }))

if __name__ == "__main__":
    main()
````

### Expected Output (stdout)

```json
{
  "run_id": "2026-01-28T...",
  "status": "IN_PROGRESS"
}
```

---

## NodeJS Backend – API Contract

### Environment Variable

```
RUNS_BASE_PATH=./runs
```

---

## API Endpoints

### 1) POST `/runs` — Start Assessment

#### Input

```json
{
  "client_id": "ttms",
  "cloud": "azure",
  "subscription_id": "xxxx",
  "scope": "network",
  "mode": "from-cloud",
  "repo_path": null,
  "run_name": "optional"
}
```

#### Behavior

* Call Python wrapper:

  ```
  python3 -m app.api_entry
  ```
* Pass request JSON via stdin
* Parse stdout JSON

#### Response

```json
{
  "run_id": "2026-01-28T...",
  "status": "IN_PROGRESS"
}
```

---

### 2) GET `/runs` — List Runs

#### Behavior

* Read folders under `RUNS_BASE_PATH`
* For each run read `metadata/metadata.json`

#### Response

```json
[
  {
    "run_id": "2026-01-28T...",
    "client_id": "ttms",
    "cloud": "azure",
    "scope": "network",
    "status": "POLICY_PASSED",
    "started_at": "2026-01-28T10:00:00Z",
    "completed_at": "2026-01-28T10:01:30Z"
  }
]
```

---

### UI Status Mapping (IMPORTANT)

Backend internal status → UI status:

* `POLICY_PASSED` → `POLICY_PASSED`
* `POLICY_FAILED` → `POLICY_FAILED`
* Any status ending with `_FAILED` → `FAILED`
* Anything else → `IN_PROGRESS`

---

### 3) GET `/runs/:run_id` — Run Details

#### Behavior

* Read:

  * `metadata.json`
  * `tfsec.json`
  * `tfplan.json`
  * `opa_decision.json`
* If any file is missing → return defaults (don’t crash)

#### Response Shape

Must match **terraform-assessment-ui-mvp.md** exactly:

* `summary`
* `security`
* `plan`
* `policy`
* `artifacts`

---

### 4) GET `/runs/:run_id/artifacts/:name` — Download Artifact

#### Allowed Files (Whitelist Only)

```
tfsec.json
tfplan.json
opa_decision.json
metadata.json
tfsec.log
tf_plan.log
opa.log
```

#### File Mapping

| API name          | File path                     |
| ----------------- | ----------------------------- |
| tfsec.json        | reports/iac/tfsec.json        |
| tfplan.json       | reports/tf/tfplan.json        |
| opa_decision.json | reports/opa/opa_decision.json |
| metadata.json     | metadata/metadata.json        |
| tfsec.log         | logs/tfsec.log                |
| tf_plan.log       | logs/tf_plan.log              |
| opa.log           | logs/opa.log                  |

---

## Error Handling Rules

* Missing report files → return zero counts / empty arrays
* Never expose arbitrary file paths
* Never allow `../` access

---

## Local Development Tip

Use `sample-runs.tgz`:

1. Extract locally
2. Set `RUNS_BASE_PATH` to extracted `runs/`
3. Develop APIs without running real pipeline

---

## Phase-1 Scope Reminder

* Read-only UI
* No terraform apply
* No dashboards/graphs
* Azure + Network only

---

## End of Document
