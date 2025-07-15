### 🧩 **Terraform Situational Interview Questions**

---

#### ✅ Q1: *"You applied a Terraform plan and accidentally deleted a critical S3 bucket. What could have caused this? How would you prevent this in the future?"*

**Assesses:** Understanding of destructive changes, lifecycle rules, safety

**Answer tips:**

**Cause:**

Terraform likely planned the deletion because of one of these reasons:
- The bucket resource was removed or renamed in the .tf files.
- An immutable property (e.g., bucket name) was changed, triggering a recreate.
- There was manual drift — someone changed or deleted the bucket in AWS outside Terraform.

**Prevention:**
- Add a lifecycle { prevent_destroy = true } block to protect critical resources.
- Use remote backend with locking to avoid multiple users accidentally applying conflicting plans.
- Enforce manual review of terraform plan before apply (no auto-approve).
- Add an IAM policy that denies s3:DeleteBucket on production buckets.

> note: not in everycase terraform is gonna recreate resource after manual changes like if i manually change tag from console its not gonna destroy & create (ie recreate ) those resource but only An immutable property (e.g., bucket name) changes , it is gonna recreate those resource 
---

#### ✅ Q2: *"You’re working with a remote team. Everyone runs `terraform apply` from their machine, and you often see state corruption. How do you fix this?"*

**Assesses:** Remote state management and collaboration

**Answer tips:**

* Never use local state in teams
* Use **remote state**: S3 + DynamoDB (for locking)
* Set up backend with locking to avoid concurrent applies
* Add a CI/CD pipeline so Terraform is run centrally, not manually

---

#### ✅ Q3: *"Your Terraform code shows a plan to destroy and recreate a resource that hasn’t changed. Why might that happen, and how do you avoid downtime?"*

**Assesses:** Infra drift, dependencies, immutable resources

**Answer tips:**

* Some resource types (e.g., AWS instance types, IPs) are **immutable**
* Might be external drift (manual AWS change)
* Avoid downtime:

  * Use `create_before_destroy`
  * Use `lifecycle` block
  * Investigate with `terraform state show <resource>`

```
Meaning of Some resource types (e.g., AWS instance types, IPs) are immutable"
-----------------------------------------------------------------------------

✅ Explanation:
In Terraform and cloud infrastructure: 🔒 Immutable = "Cannot be changed after creation"
Some cloud resource attributes cannot be modified in-place. If you change such attributes in Terraform, Terraform will:
- Destroy the existing resource
- Create a new one with the new configuration

🧱 Examples of Immutable Fields:
---------------------------------
| Resource Type           | Immutable Attribute Examples  | What Happens If Changed?               |
| ----------------------- | ----------------------------- | -------------------------------------- |
| `aws_instance` (EC2)    | `instance_type`, `subnet_id`  | Terraform destroys old and creates new |
| `aws_s3_bucket`         | `bucket` (i.e. bucket name)   | Recreates the bucket                   |
| `aws_lb` (ALB/NLB)      | `internal` flag               | Replaces the load balancer             |
| `aws_db_instance` (RDS) | `engine`, `availability_zone` | Replaces the DB instance               |

💥 Why it matters:
----------------------
If you change an immutable field in your Terraform .tf files and run terraform apply, it will:
1- Destroy the old resource
2- Create a new one
3- Potentially cause downtime or data loss if not handled properly

🛡️ How to prevent issues:
-----------------------------
1- Use lifecycle { create_before_destroy = true } to avoid downtime
2- Avoid changing immutable fields without backup plans
3- Review terraform plan carefully before apply
```

---

#### ✅ Q4: *"A developer asks you to provision the same infra for dev, staging, and prod. How do you organize your Terraform code?"*

**Assesses:** Code reuse, structure, module design

**Answer tips:**

* Use **modules** for shared infra (VPC, EC2, RDS)
* Separate **workspaces** or folders for `dev`, `stage`, `prod`
* Pass in variables (e.g. tags, CIDRs, instance size)
* Use remote state per environment

---

#### ✅ Q5: *"Your Terraform apply failed halfway through. Some resources were created, some not. How do you recover?"*

**Assesses:** Partial failure handling and recovery

**Answer tips:**

* Use `terraform refresh` to sync state with reality {not used anymore}
* Manually fix or remove faulty resources (if needed)
* Run `terraform apply` again — it's idempotent
* Use `target` to apply specific resources (e.g., `--target=aws_instance.foo`) if needed

---

#### ✅ Q6: *"You want to give different teams read-only access to state files but allow only the DevOps team to apply changes. How do you implement this?"*

**Assesses:** Security and access control

**Answer tips:**

* Use remote backend (e.g. S3) with **IAM permissions**
* DevOps: full access to `terraform plan/apply`
* Developers: read-only access to `statefile`
* Store state in S3, lock via DynamoDB

---

#### ✅ Q7: *"You made manual changes in AWS console, and now Terraform wants to recreate the resource. What do you do?"*

**Assesses:** Drift handling

**Answer tips:**

* Use `terraform plan` to see diff
* If changes are intentional, update TF code to match (apna code me chnages kr do)
* Avoid manual changes outside Terraform unless emergency

---

#### ✅ Q8: *"You created a Terraform module, but another team is facing versioning issues. How would you manage this module long-term?"*

**Assesses:** Module lifecycle and best practices

**Answer tips:**

* Version your modules in Git (with tags)
* Store them in a central Terraform registry or private module registry
* Add `version = "~> 1.0"` in module blocks
* Use changelogs and proper input validation

---

#### ✅ Q9: *"You were provisioning EC2 using Terraform and suddenly hit an API rate limit. What can you do?"*

**Assesses:** Retry handling and provider behavior

**Answer tips:**

* Terraform handles retries internally but can fail
* Use `-parallelism=N` flag to reduce concurrent requests
* Implement retry logic in provider config
* Consider spacing out applies in CI/CD

---

#### ✅ Q10: *"Your `terraform apply` fails due to a missing variable or wrong value. What can you do to improve user experience for others?"*

**Assesses:** Usability, documentation, onboarding

**Answer tips:**

**Give each input a name, explanation, and fallback**
Instead of “just pass a value,” you tell Terraform what each setting is for, what type it should be (like text or a list), and even provide a default if the user doesn’t set one. That way, new users immediately see what they need to fill in.

**Ship a simple “how-to” document**
In your module’s folder, include a README.md that explains:
- What the module does
- What inputs it needs (with descriptions)
- What it outputs
- A small example of how to call it

**Provide ready-to-use defaults**
Drop a file named defaults.auto.tfvars alongside your code, with example values. Terraform automatically reads it, so beginners can run terraform apply without extra flags.



---
