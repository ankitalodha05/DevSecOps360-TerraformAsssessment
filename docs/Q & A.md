# Terraform Assessment – FAQ

## Q: What is this Terraform Assessment repo?
**A:**  
This repo is used to assess Terraform code without deploying it.  
It checks security, correctness, change impact, and policy compliance at plan time.

---

## Q: Are you deploying anything here?
**A:**  
No. We only run `terraform plan`, never `terraform apply`.

---

## Q: Why do you generate tfplan.json?
**A:**  
Terraform plan output is converted to JSON so it can be analyzed programmatically and evaluated by OPA policies.

---

## Q: What policy have you applied in OPA?
**A:**  
We have applied a safety policy that blocks any Terraform plan which tries to delete existing resources.

---

## Q: How does OPA decide pass or fail?
**A:**  
OPA reads the Terraform plan JSON.  
- If it finds a delete action, the policy denies it and the assessment fails.  
- If there are no deletes, the assessment passes.

---

## Q: Where can I see the final decision?
**A:**  
The final decision is stored in:
- `metadata.json`
- `opa_decision.json`  

These files are used for audit and traceability.

---

## Q: Why is this useful in real projects?
**A:**  
It prevents accidental or unauthorized destructive changes by enforcing guardrails before infrastructure is applied.

---

## Q: Why tfsec?
**A:**  
tfsec is used to detect security misconfigurations in Terraform code before planning or deployment.

### What tfsec checks:
- Publicly exposed resources  
- Open security groups  
- Insecure storage settings  
- Missing encryption  
- Weak IAM or access configurations  

tfsec performs all checks **without deploying anything**.

---

## Q: Why is ScoutSuite used in a later stage?
**A:**  
ScoutSuite scans the **actual cloud environment**, not Terraform code.  
Because of this, it is more useful after we understand what Terraform is planning to change.

---

## Q: Why not run ScoutSuite first?
**A:**  
Running ScoutSuite first produces a long list of existing issues without context.

By running it later, we can compare:
- What already exists in the cloud  
- What Terraform is planning to change  

This makes the findings more meaningful and actionable.

---

## Q: How do the stages connect? (Simple logic)
**A:**  
- tfsec → checks Terraform code security  
- terraform plan → shows intended changes  
- OPA → enforces governance on planned changes  
- ScoutSuite → validates real cloud posture after or alongside changes  

---

## Important Note
ScoutSuite is intentionally run later because it scans the **live cloud environment**, while tfsec and OPA work on Terraform code and Terraform plan.  
Running ScoutSuite later provides better context and avoids unnecessary noise.

---
