
Terraform:
use case
state locking
basic terraform functions
variable types
modules
why modules
any code you have written
types of provisioners

# --------------------------------------------------------------------------

# what is terraform 
Terraform is an Infrastructure as Code (IaC) tool created by HashiCorp.
It allows you to define your entire infrastructure — like servers, databases, networks, and cloud services — as code in .tf files, which can be version-controlled and reused across environments.

Terraform supports multiple cloud providers (like AWS, Azure, GCP), and uses a safe execution plan (terraform plan and apply) to provision and update resources efficiently.

# --------------------------------------------------------------------------


# Terraform Usecase 

Terraform is used for defining infrastructure as code. Typical interview use cases include:

1. Single-cloud deployments (e.g. full AWS stacks).
- Scenario: You need to provision all infrastructure (VPC, subnets, EC2 instances, RDS, etc.) in AWS.
- How Terraform helps: Write HCL code once, then manage all resources through terraform apply and terraform destroy 
- Why it matters: Consistent, version-controlled infrastructure, repeatable across environments.


 2. Multi-Cloud Infrastructure
- Scenario: You want resources spread across AWS and Azure for resilience or vendor flexibility.
- How Terraform helps: Use multiple providers with aliases (e.g. provider "aws" and provider "azurerm").
- Define both AWS and Azure resources in one repo, maintaining shared modules 
- Why it matters: Avoids vendor lock-in and supports redundancy across clouds.

3. 📦 3. N-Tier Application Architecture
- Scenario: Deploy a multi-tier app with database, backend, frontend, monitoring, and DNS.
- How Terraform helps: Define each layer (e.g. MySQL, app servers, ALB, Datadog agent) as modular components; manage dependencies automatically 
- Why it matters: Logic flows (DB → app → monitoring), modularity, easier maintenance and updates.

4. 🧪 4. Consistent Environments & CI/CD
- Scenario: Need repeatable dev, staging, QA, and prod environments for testing.
- How Terraform helps: Use modules and workspaces to replicate infrastructure across environments 
linkedin.com
- Why it matters: Keeps environments aligned, reduces “it works on dev” issues, speeds up testing and rollback.

🧩 5. Kubernetes/EKS Provisioning
- Scenario: Set up a Kubernetes cluster and deploy applications with Helm charts.
- How Terraform helps: Use AWS EKS provider to provision cluster, then Kubernetes/Helm providers to deploy workloads 
- Why it matters: Infrastructure and app-infra defined in one pipeline, fully automated.

# --------------------------------------------------------------------------

# state locking 
State locking ensures that only one user or process can modify the Terraform state at a time.
This prevents race conditions or corrupt states when multiple people run terraform apply simultaneously.

#### ✅ How is state locking achieved then?
You use Amazon DynamoDB (along with S3) to enable state locking when using remote backend.

#### Real Setup (Best Practice)
terraform {
  backend "s3" {
    bucket         = "my-tf-state-bucket"
    key            = "dev/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-lock-table"  # 💡 enables locking
  }
}

* bucket: stores the actual state file
* dynamodb_table: used for locking

### ✅ S3 Native State Locking in Terraform (without dynamodb)
🛠️ Before (Terraform <1.10):
Using the S3 backend required a separate DynamoDB table for locking.

Terraform would create a lock entry in DynamoDB to prevent concurrent apply runs.

✅ Now (Terraform ≥1.10):
* Terraform supports native S3 state locking using the new use_lockfile setting 
* This leverages S3 conditional writes:
   - Terraform creates a .tflock file alongside your state file.
   - If another process has already created that lock file, your terraform apply fails with a lock error.
* You no longer need a DynamoDB table unless you explicitly want extra redundancy

🌐 How to enable it:
```
terraform {
  backend "s3" {
    bucket       = "my-state-bucket"
    key          = "prod/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true
    use_lockfile = true  # Enables S3 native locking
  }
}
```

Terraform originally used DynamoDB for state locking with its S3 backend. As of Terraform 1.10+, you can enable S3-native locking using the use_lockfile parameter — Terraform creates a .tflock file in the bucket using S3 conditional writes, removing the need for extra DynamoDB tables.


# --------------------------------------------------------------------------

# basic Terraform functions


---

## 🧠 **1. `concat()`** – Combine strings or lists
✅ Use case: Combine tags, subnet lists, or CIDRs dynamically.


```hcl
concat(["a", "b"], ["c", "d"])  # => ["a", "b", "c", "d"]
```
✅ Using concat() – Merge default + env tags
```
locals {
  default_tags = [
    { Key = "Owner", Value = "DevOps" },
    { Key = "Environment", Value = "dev" }
  ]

  extra_tags = [
    { Key = "Team", Value = "Backend" }
  ]

  all_tags = concat(local.default_tags, local.extra_tags)
}

```

result
```
[
  { Key = "Owner", Value = "DevOps" },
  { Key = "Environment", Value = "dev" },
  { Key = "Team", Value = "Backend" }
]

```
> In this case, we're combining the common tags with extra tags to apply all of them to a resource."



---

## 🔤 **2. `join()`** – Join list items into a string
✅ Use case: Format region names, DNS entries, file paths, etc.

```hcl
join("-", ["us", "east", "1"])  # => "us-east-1"
```

✅ Using join() – Create region string from parts:

```
locals {
  region_parts = ["us", "east", "1"]
  region_name  = join("-", local.region_parts)
}
```
💬 Result:

```
"us-east-1"
```
---

## 🔢 **3. `length()`** – Count elements in a list or string

```hcl
length(["a", "b", "c"])         # => 3
length("hello")                # => 5
```

---

## 🆚 **4. `contains()`** – Check if list has a value

```hcl
contains(["a", "b"], "a")      # => true
```

---

## 💬 **5. `lower()` / `upper()`** – Convert case

```hcl
lower("HELLO")                 # => "hello"
upper("dev")                   # => "DEV"
```

---

## 🔄 **6. `replace()`** – Replace part of a string

```hcl
replace("dev.env", ".env", ".prod")  # => "dev.prod"
```

---

## ⏱ **7. `timestamp()`** – Get current time in RFC3339 format

```hcl
timestamp()                   # => "2025-06-25T20:14:52Z"
```

---

## ✅ **8. `lookup()`** – Get a key from a map with default fallback
lookup() is a Terraform function used to safely get a value from a map using a key.
If the key doesn’t exist, it returns a default fallback value instead of throwing an error.
✅ Syntax:
```
lookup(map, key, default)
```
- map: the map (key-value pair) you're querying from
- key: the key to look for
- default: value to return if the key is not found

🧪 Example 1 – Key exists:
```
locals {
  instance_type_map = {
    dev  = "t2.micro"
    prod = "t3.medium"
  }

  selected_type = lookup(local.instance_type_map, "dev", "t2.small")
}
```

##### Result: "t2.micro"
✅ Because "dev" exists, it returns "t2.micro".



🧪 Example 2 – Key does NOT exist:
```
selected_type = lookup(local.instance_type_map, "qa", "t2.small")
```

##### Result: "t2.small" (default)
✅ "qa" doesn't exist in the map, so it falls back to "t2.small".


🚨 Without lookup()?
If you access like this:
```
local.instance_type_map["qa"]
```
Terraform will throw an error: key not found.
##### ✅ So lookup() is safer — prevents failures when the key is missing.







```hcl
lookup({ dev = "t2.micro", prod = "t3.medium" }, "stage", "t2.small")
# => "t2.small"
```

---

## 📦 **9. `merge()`** – Merge two or more maps

```hcl
merge({a = 1}, {b = 2})        # => {a = 1, b = 2}
```

---

## 🧪 **10. `element()`** – Get list item by index

```hcl
element(["a", "b", "c"], 1)    # => "b"
```

---

## ✅ Summary Table (Interview Quick Glance):

| Function      | Purpose                          |
| ------------- | -------------------------------- |
| `concat()`    | Merge lists                      |
| `join()`      | List → string                    |
| `length()`    | Count elements                   |
| `contains()`  | Check presence in list           |
| `lower()`     | String to lowercase              |
| `replace()`   | Replace part of a string         |
| `timestamp()` | Get current UTC time             |
| `lookup()`    | Safe map key lookup with default |
| `merge()`     | Combine maps                     |
| `element()`   | Get item by index in list        |

---

# --------------------------------------------------------------------------

#  MOules & why modules 

📦 What are Terraform Modules?
A module is a reusable block of Terraform code that that helps you organize and reuse infrastructure like VPCs, EC2, databases, etc.
Think of it like a function in programming — you define it once and use it many times.

#### 🧠 Simple Example:
Let’s say you want to create a VPC with subnets, route tables, and gateways.

#### ❌ Without a Module:
You’d end up writing the same EC2 resource block again and again for each environment:
```
resource "aws_instance" "dev_server" {
  ami           = "ami-0abc1234"
  instance_type = "t2.micro"
  tags = {
    Name = "DevServer"
  }
}

resource "aws_instance" "qa_server" {
  ami           = "ami-0abc1234"
  instance_type = "t2.small"
  tags = {
    Name = "QAServer"
  }
}
```
This creates repetition, increases risk of mistakes, and becomes hard to manage.


#### ✅ With a Module:
##### Step 1: Create a reusable EC2 module
Folder: modules/ec2_instance/

```
main.tf:

resource "aws_instance" "this" {
  ami           = var.ami
  instance_type = var.instance_type
  tags = {
    Name = var.name
  }
}

variables.tf:

variable "ami" {}
variable "instance_type" {}
variable "name" {}
```

##### Step 2: Use that module in your main.tf:
```
module "dev_instance" {
  source        = "./modules/ec2_instance"
  ami           = "ami-0abc1234"
  instance_type = "t2.micro"
  name          = "DevServer"
}

module "qa_instance" {
  source        = "./modules/ec2_instance"
  ami           = "ami-0abc1234"
  instance_type = "t2.small"
  name          = "QAServer"
}
```
✅ What You Achieve:
- One module, used many times → DRY (Don't Repeat Yourself)
- Easy to test, version, and maintain
- Scales to dozens of teams with different configs but same logic
✅ Now this same module can be reused for dev/staging/prod just by passing different inputs!


#### 🧠 Why Use Modules?
1. ✅ Reusability
Define once, use in multiple environments (dev, staging, prod).

Example: One VPC module, reused across regions.

2. 🔧 Maintainability
Easier to manage and update code.

Changes in one module apply everywhere it's used.

3. 🧪 Consistency
Enforces standard configurations across all teams.

Reduces human error (e.g. naming, tagging, CIDR blocks).

4. 📁 Organization
Splits infrastructure into logical units (e.g. vpc, compute, storage).

Cleaner folder structure and better code readability.

5. ☁️ Separation of Concerns
Networking, compute, DB can be developed and tested independently.

# --------------------------------------------------------------------------

# types ofprovisioner

🛠️ What Are Provisioners in Terraform?
Provisioners allow Terraform to run scripts or commands on a resource after it’s created — like installing software, copying files, or configuring the OS.

#### 🔄 Types of Provisioners

1. ✅ local-exec
Runs a command on your local machine (where you run Terraform)
🧪 Example
```
provisioner "local-exec" {
  command = "echo Deployment finished > result.txt"
}
```
📌 Use case: Trigger a local script or CLI after deployment.


2. ✅ remote-exec
Runs a command on the remote machine (like EC2) using SSH or WinRM.

🧪 Example:
```
provisioner "remote-exec" {
  inline = [
    "sudo apt update",
    "sudo apt install nginx -y"
  ]
}
```
📌 Use case: Install packages or start services inside the instance.

3. ✅ file
Uploads a file from your local machine to the remote instance.

🧪 Example:
```
provisioner "file" {
  source      = "app.conf"
  destination = "/etc/myapp/app.conf"
}
```
📌 Use case: Push config files to EC2 or VM.

#### ⚠️ Important Notes (Good to Mention in Interviews):
Provisioners are not recommended for general use — try to use cloud-init, userdata, or automation tools like Ansible instead.
They are best used as a last resort (e.g., debugging, bootstrap scripts).
If a provisioner fails, the resource might be marked as tainted.
In Terraform, tainted = broken ✅ but still exists in your infrastructure.
It tells Terraform: "This resource is no longer safe — destroy and recreate it in the next apply

# ---------------------------------------------------------------------------------

# terraform files & use cases 

| File                               | Description                              | Use Case                                                 |
| ---------------------------------- | ---------------------------------------- | -------------------------------------------------------- |
| `main.tf`                          | Main config file with resources/modules  | Define infrastructure (EC2, VPC, etc.)                   |
| `variables.tf`                     | Declares input variables                 | Make infra configurable (e.g. `instance_type`, `region`) |
| `outputs.tf`                       | Declares output values                   | Show values after apply (e.g. public IP, ARN)            |
| `terraform.tfvars`                 | Provides actual values for variables     | Set environment-specific values                          |
| `provider.tf`                      | Declares cloud provider (AWS, GCP, etc.) | Connect to the right cloud and region                    |
| `backend.tf`                       | Configures remote state (S3, etc.)       | Enable team collaboration and state locking              |
| `versions.tf`                      | Locks Terraform and provider versions    | Ensure consistency across teams/systems                  |
| `.terraform.lock.hcl`              | Records exact provider versions          | Used for dependency locking (like `package-lock.json`)   |
| `.terraform/`                      | Local cache of providers/modules         | Optimizes init and reuse of dependencies                 |
| `terraform.tfstate`                | Current infra state (local or remote)    | Tracks what Terraform manages                            |
| `terraform.tfstate.backup`         | Auto-created backup of last state        | Recovery if something goes wrong                         |
| `*.auto.tfvars`                    | Auto-loaded variable values              | Automatically included without specifying `-var-file`    |
| `terraform.rc` or `~/.terraformrc` | CLI config (like credentials helper)     | Manage CLI behavior or private registry auth             |

# --------------------------------------------------------------------------------

# variable types

Here’s a clear breakdown of the **types of variables in Terraform** — with examples and interview-friendly explanations:

---

## 📦 **Terraform Variable Types**

Terraform supports **three base types** and **complex types**.

---

### 🔢 1. **String**

A plain text value.

```hcl
variable "env" {
  type    = string
  default = "dev"
}
```

🧠 Use for names, regions, AMIs, etc.

---

### 🔢 2. **Number**

A numeric value (integer or float).

```hcl
variable "instance_count" {
  type    = number
  default = 2
}
```

🧠 Use for counts, sizes, limits.

---

### 🟩 3. **Bool**

True or false value.

```hcl
variable "enable_logging" {
  type    = bool
  default = true
}
```

🧠 Useful for feature toggles (on/off).

---

## 🔀 **Complex Types**

---

### 📜 4. **List**
list of string
An ordered list of values (same type).

```hcl
variable "azs" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b"]
}
```

🧠 Good for subnets, zones, tags.

---

### 📚 5. **Map**
map of string 
Key-value pairs (like dictionaries).

```hcl
variable "instance_types" {
  type = map(string)
  default = {
    dev  = "t2.micro"
    prod = "t3.medium"
  }
}
```

🧠 Useful for environment-specific configs.

---

### 🧱 6. **Object**
list of objects
Structured fields with different types.

```hcl
variable "app_config" {
  type = object({
    name     = string
    port     = number
    enabled  = bool
  })
  default = {
    name    = "myapp"
    port    = 8080
    enabled = true
  }
}
```

🧠 Use for structured settings (e.g., app config).

---

### 🧰 7. **Tuple**

Ordered list of different types.

```hcl
variable "mixed_values" {
  type = tuple([string, number, bool])
  default = ["dev", 3, true]
}
```

🧠 Rare, used when order and type are both strict.

---

## 🧠 Interview Summary:

> Terraform variables can be simple types like `string`, `number`, and `bool`, or complex types like `list`, `map`, `object`, and `tuple`.
> This allows writing **reusable and type-safe infrastructure code**.

---







