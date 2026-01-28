
# **tfsec – Security Scanning in Terraform Assessment**

## 1. Introduction

**tfsec** is an **open-source static analysis security scanner** for **Terraform code**.
It is designed to identify **security misconfigurations** in Infrastructure as Code (IaC) **before** infrastructure is deployed or modified.

In the **Terraform Assessment** project, tfsec plays a critical role in detecting **security risks early**, without making any infrastructure changes.

---

## 2. What is tfsec?

**tfsec** scans Terraform files (`.tf`) and evaluates them against a large set of **predefined security rules** based on industry best practices and compliance standards.

It answers questions like:

* Is this infrastructure **secure by default**?
* Are there **open ports**, **public access**, or **missing encryption**?
* Does this Terraform configuration violate **security best practices**?

🔹 **Important:**
tfsec **does not deploy**, **does not connect to cloud**, and **does not modify infrastructure**.

---

## 3. Why tfsec is used in our project

Our project is **Terraform Assessment**, not Terraform deployment.

### Project goal:

> Assess infrastructure for **security, compliance, and policy risks** without applying changes.

tfsec fits perfectly because:

* It works **before `terraform apply`**
* It provides **machine-readable JSON output**
* It integrates cleanly into CI/CD pipelines
* It focuses on **security misconfigurations**, not policy logic

---

## 4. Where tfsec fits in our pipeline

Our assessment pipeline stages:

```
Stage 0 → INIT (create run + metadata)
Stage 2 → tfsec (IaC security scan)
Stage 5 → terraform validate
Stage 6 → terraform plan
Stage 7 → OPA policy evaluation
```

### Why tfsec runs early

* Security issues should be detected **as early as possible**
* Cheaper to fix IaC issues before planning or deployment
* Prevents unsafe infrastructure from moving forward

---

## 5. What tfsec checks (with examples)

### Common Azure / Network issues tfsec detects

#### 🔴 High-risk findings

* NSG allows `0.0.0.0/0` on SSH (22) or RDP (3389)
* Public IP attached to sensitive resources
* Storage account without encryption
* Disabled logging or monitoring
* Weak TLS / SSL configuration

#### 🟢 Secure configurations

* Private subnets only
* Restricted NSG rules
* Encryption enabled
* Least-privilege network access

---

## 6. tfsec output in our project

tfsec produces a **JSON report** stored at:

```
runs/<run_id>/reports/iac/tfsec.json
```

### Typical tfsec JSON fields

* Rule ID
* Severity (`CRITICAL`, `HIGH`, `MEDIUM`, `LOW`)
* Resource / file location
* Description of the issue
* Recommendation

### How UI uses tfsec output

* Total security issue count
* Severity breakdown
* Top findings list
* Human-readable explanations

---

## 7. Why tfsec is NOT enough alone

tfsec focuses on **technical security misconfigurations**, but it does **not**:

* Enforce organization-specific rules
* Understand business policies
* Evaluate runtime cloud configuration

That’s where **OPA** and **Scout Suite** come in.

---

## 8. tfsec vs OPA (Key Differences)

| Aspect      | tfsec                               | OPA                            |
| ----------- | ----------------------------------- | ------------------------------ |
| Purpose     | Security misconfiguration detection | Policy enforcement             |
| Works on    | Terraform code                      | Terraform plan / decisions     |
| Rule type   | Predefined security rules           | Custom organizational policies |
| Output      | Security findings                   | Allow / deny decisions         |
| Example     | “Port 22 open to internet”          | “Deletes are not allowed”      |
| Flexibility | Fixed rules                         | Fully customizable             |

### In simple terms

* **tfsec** asks: *“Is this configuration secure?”*
* **OPA** asks: *“Is this configuration allowed by policy?”*

---

## 9. Why OPA comes after tfsec

OPA evaluates **decisions**, not static code.

It needs:

* Terraform plan output (`tfplan.json`)
* Context of resource changes

So the order makes sense:

1. tfsec → find security issues
2. terraform plan → understand changes
3. OPA → enforce governance rules

---

## 10. Why Scout Suite comes later (and why it’s different)

### What is Scout Suite?

**Scout Suite** is a **cloud security posture assessment tool** that scans **live cloud environments** using cloud APIs.

It checks:

* IAM permissions
* Network exposure
* Logging & monitoring
* Compliance posture

### Key differences from tfsec

| Aspect | tfsec             | Scout Suite                 |
| ------ | ----------------- | --------------------------- |
| Input  | Terraform code    | Live cloud environment      |
| Timing | Before deployment | After infrastructure exists |
| Access | No cloud access   | Requires cloud credentials  |
| Scope  | IaC security      | Cloud posture & compliance  |

### Why Scout Suite comes later

* tfsec works without cloud access
* Scout Suite needs real cloud resources
* Scout Suite validates **what actually exists**, not what is planned

---

## 11. How all three tools work together

```
Terraform Code
   ↓
tfsec (security misconfigurations)
   ↓
terraform plan (change detection)
   ↓
OPA (policy enforcement)
   ↓
Scout Suite (cloud posture validation)
```

Each tool covers a **different layer of risk**.

---

## 12. Why this design is strong ?

*“Why did you use tfsec?”*:

> We used tfsec to perform static security analysis on Terraform code before any infrastructure changes. It detects insecure configurations early and integrates well into a DevSecOps assessment pipeline.

*“Why not only OPA?”*:

> OPA enforces policy decisions, but tfsec focuses on technical security misconfigurations. Both are needed to cover security and governance.

*“Why Scout Suite later?”*:

> Scout Suite evaluates real cloud posture, while tfsec and OPA work on IaC and plans. Using all three provides full lifecycle security.

---

## 13. One-line memory summary

* **tfsec** → IaC security scanner (before deployment)
* **OPA** → Policy enforcement (plan-time)
* **Scout Suite** → Cloud posture assessment (runtime)

---

## 14. Conclusion

tfsec is a **foundational security layer** in our Terraform Assessment project.
It ensures infrastructure code follows security best practices **before** it reaches planning, policy enforcement, or real cloud environments.

Together with **OPA** and **Scout Suite**, it enables a **complete DevSecOps assessment workflow**.
