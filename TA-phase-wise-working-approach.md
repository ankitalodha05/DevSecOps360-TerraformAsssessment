# TerraformAsssessment– Phase-wise Working Approach (Simple)

## Phase 1 – MVP (Current Phase)(Completed)

**Goal:** Make the assessment pipeline stable and reliable.

- Start from Azure cloud or Terraform repository
- Generate Terraform (only when starting from cloud)
- Perform minimal normalization to make Terraform plan-ready
- Run:
  - `terraform validate`
  - `terraform plan` (safe mode, no apply)
  - `tfsec` (Infrastructure-as-Code security scan)
  - `OPA` (policy evaluation on `plan.json`)
- Produce structured outputs for API and UI consumption

**Outcome:**

- Safe, read-only infrastructure assessment
- Security and policy results generated
- No changes made to cloud resources

---

## Phase 2 – AI Cleanup (Planned)

**Goal:** Convert assessment output into reusable, production-ready Terraform.

- Expand cleanup beyond minimal fixes
- Normalize resource naming and tags
- Refactor resources using `for_each` and maps
- Generate:
  - Reusable Terraform modules
  - Client-specific `terraform.tfvars`
- Keep assessment and cleanup outputs separate

**Summary:**

AI cleanup is planned as Phase-2. MVP only includes minimal normalization needed for plan-ready assessment. Once UI integration is stable, we’ll expand cleanup to generate reusable modules, client tfvars, and `for_each` refactors.

---

## Phase 3 – Cloud Posture & Enrichment (Future)

**Goal:** Add runtime visibility and deeper insights.

- Integrate Scout Suite for live cloud posture scanning
- Add optional infrastructure graphs and visualizations
- Combine IaC assessment and runtime posture in a unified report
- Expand policy coverage and scoring

**Outcome:**

- Complete infrastructure risk visibility across code and runtime

---

## Overall Direction

- Phase 1 proves the pipeline works end-to-end
- Phase 2 makes Terraform reusable and clean
- Phase 3 adds runtime posture and advanced insights
