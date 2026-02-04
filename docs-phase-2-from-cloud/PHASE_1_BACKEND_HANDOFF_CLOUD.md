# DevSecOps360 – Phase-1 Backend Handoff (Cloud Scenario)

## Objective

This document explains the **Phase-1 backend expectations for the cloud-based assessment** (`mode=from-cloud`) in DevSecOps360.

This is an **extension of the existing Phase-1 contract**, not a new flow.

Scope remains **read-only and locked**.

---

## Key Difference from Repo Scenario

| Repo Scenario          | Cloud Scenario           |
|-----------------------|--------------------------|
| Terraform from repo   | Terraform generated from cloud |
| mode=from-repo        | mode=from-cloud          |
| repo_path required    | subscription_id required |
| Static TF source      | Terraformer-based discovery |

**Everything after run creation is identical**.

---

## High-Level Flow (Cloud)

UI → Backend API → Python Assessment → runs/<run_id>/

- No terraform apply
- No resource changes
- No Azure mutation
- Backend still reads filesystem only

---

## POST /runs (Cloud)

### Input Payload

```json
{
  "client_id": "ttms",
  "cloud": "azure",
  "subscription_id": "<azure-subscription-id>",
  "scope": "network",
  "mode": "from-cloud",
  "repo_path": null,
  "run_name": "optional"
}
````

### Backend Behavior

* Backend calls Python entry:

  ```
  python3 -m app.api_entry
  ```
* Python:

  * Authenticates using Azure CLI / SP
  * Generates Terraform (Terraformer)
  * Runs validate → plan → tfsec → OPA
  * Creates run folder under `runs/`

### Response

```json
{
  "run_id": "2026-02-04T...Z",
  "status": "IN_PROGRESS"
}
```

---

## Assessment Output Structure (Same as Repo)

```text
runs/<run_id>/
├── metadata/metadata.json
├── reports/
│   ├── iac/tfsec.json
│   ├── tf/tfplan.json
│   └── opa/opa_decision.json
└── logs/
```

**UI and backend parsing logic remains unchanged**

---

## Backend API Responsibilities (Unchanged)

* `GET /runs`
* `GET /runs/:run_id`
* `GET /runs/:run_id/artifacts/:name`

All APIs behave **exactly the same** as repo scenario.

---

## Out of Scope (Phase-1 Cloud)

* Terraform apply
* Remediation
* State storage
* DB integration
* Multi-subscription orchestration

---

## Key Clarification (Important)

Backend is **not cloud-aware**.

It:

* Does not talk to Azure
* Does not run Terraform
* Does not interpret infra

Backend only:

> reads assessment results from filesystem and returns JSON.

---

## Status

Cloud scenario follows the **same Phase-1 contract**
Safe, read-only, production-defensible
Ready for UI / backend integration
