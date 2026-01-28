# Terraform Assessment – Backend Inputs (Phase-1)

## Subject
Terraform Assessment backend inputs – Phase-1

---

## Overview

This document describes how the **NodeJS backend** should behave for the **Phase-1 Terraform Assessment UI**.

The backend acts as a bridge between:
- **React UI** and
- **Python assessment engine** (which generates run folders and reports)

---

## Base Folder Configuration

- **Environment Variable:** `RUNS_BASE_PATH`
- **Server Path (example):**
```

~/devsecops360/terraform-assessment/runs

```
- **Default (if not set):**
```

./runs

```

All assessment runs are stored under this base folder.

---

## Per-Run Folder Structure

Each assessment run produces the following structure:

```

runs/<run_id>/
├── metadata/
│   └── metadata.json
├── reports/
│   ├── iac/
│   │   └── tfsec.json
│   ├── tf/
│   │   └── tfplan.json
│   └── opa/
│       └── opa_decision.json
└── logs/
└── *.log

````

---

## API Endpoints (NodeJS)

### 1. POST `/runs`

**Purpose:** Start a new assessment run.

**Input Payload:**
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

**MVP Behavior:**

* Call a Python wrapper/CLI that starts the assessment.
* The Python process should print JSON to stdout, for example:

```json
{
  "run_id": "2026-01-23T122848Z-ttms",
  "status": "IN_PROGRESS"
}
```

**Response:**

```json
{
  "run_id": "2026-01-23T122848Z-ttms",
  "status": "IN_PROGRESS"
}
```

---

### 2. GET `/runs`

**Purpose:** List all assessment runs.

**Behavior:**

* Read all folders under `RUNS_BASE_PATH`.
* For each run, read `metadata/metadata.json`.

**Response Shape:**

```json
{
  "run_id": "...",
  "client_id": "...",
  "cloud": "azure",
  "scope": "network",
  "status": "POLICY_PASSED",
  "started_at": "2026-01-23T12:28:48Z",
  "completed_at": "2026-01-23T12:29:10Z"
}
```

---

### 3. GET `/runs/:run_id`

**Purpose:** Fetch detailed results for a single run.

**Files Read:**

* `metadata/metadata.json`
* `reports/iac/tfsec.json`
* `reports/tf/tfplan.json`
* `reports/opa/opa_decision.json`

**Response:**

* Must match exactly the JSON structure defined in the **Terraform Assessment UI MVP document**:

  * `summary`
  * `security`
  * `plan`
  * `policy`
  * `artifacts`

**Error Handling:**

* If any report file is missing:

  * Return default values (0 counts, empty arrays)
  * Do **not** fail the API

---

### 4. GET `/runs/:run_id/artifacts/:name`

**Purpose:** Download assessment artifacts.

**Behavior:**

* Serve files from the selected run folder.
* Only allow a fixed whitelist of filenames (security).

**Allowed Artifact Names:**

```
tfsec.json
tfplan.json
opa_decision.json
metadata.json
tfsec.log
tf_plan.log
opa.log
```

---

## UI Status Mapping Rules

Backend internal statuses must be mapped to UI-friendly values:

| Backend Status        | UI Status     |
| --------------------- | ------------- |
| POLICY_PASSED         | POLICY_PASSED |
| POLICY_FAILED         | POLICY_FAILED |
| *_FAILED (any suffix) | FAILED        |
| Anything else         | IN_PROGRESS   |

---

## Local Development Notes

* Use `sample-runs.tgz` as reference test data.
* Unzip it locally.
* Set `RUNS_BASE_PATH` to the extracted `runs/` folder during development.

---

## Notes

* Backend is **read-only** except for triggering a run.
* No Terraform apply.
* No inline editing.
* This API supports **Phase-1 MVP only**.
