
1. CICD 

## 🚀 What is CI/CD?

> **CI/CD** stands for:

* **Continuous Integration (CI)**
* **Continuous Delivery (CD)** or
* **Continuous Deployment (CD)**


> **CI/CD** stands for **Continuous Integration** and **Continuous Delivery/Deployment**.
> It's a **development practice** that helps teams deliver code **faster, safer, and more often**.

---

### 🛠️ CI – Continuous Integration

> Developers push code frequently.
> Every change is:

* **Automatically built**
* **Tested**
* **Checked for errors**

✅ This helps catch bugs early and keeps the codebase clean.

---

### 🚀 CD – Continuous Delivery

> After the code passes all tests, it’s **packaged and ready for deployment**.
> Deployment is **manual**, but can happen at any time with one click.

---

### ⚙️ CD – Continuous Deployment

> The final step is **automating the deployment** too.
> If all tests pass, the code goes **live automatically** — no human approval needed.

---

### 🎯 Final One-Liner (interview-friendly):

> “CI/CD is about automating the build, test, and release process —
> so that software can be delivered quickly, safely, and repeatedly.”

---

## EXAMPLE 

✅ Continuous Integration (CI)
| Step | Action                         | Why it’s CI                       |
| ---- | ------------------------------ | --------------------------------- |
| 1️⃣  | Code pushed to Git             | Developer code commit triggers CI |
| 2️⃣  | Jenkins webhook triggers build | CI starts on push                 |
| 3️⃣  | Checkout code                  | Pulls latest code for testing     |
| 4️⃣  | Build project                  | Compiles code, builds `.jar`      |
| 5️⃣  | Run unit tests                 | Validates correctness of code     |
| 6️⃣  | Archive JAR                    | Stores built artifacts            |

➡️ CI ends when the code is tested and packaged into a deployable artifact.

🚀 Continuous Delivery / Deployment (CD)

| Step | Action                   | Why it’s CD                           |
| ---- | ------------------------ | ------------------------------------- |
| 7️⃣  | Optional Approval step   | Manual gate before release (Delivery) |
| 8️⃣  | SCP JAR to remote server | Starts release process                |
| 9️⃣  | SSH + restart service    | Deploys application                   |
| 🔟   | Smoke test               | Verifies deployment succeeded         |
➡️ CD starts from release approval (or deployment trigger) and ends after deployment & smoke testing.

===============================================================================

# SCRIPTED & DECLARATIVE



### ✅ Declarative Pipeline (💡 Recommended)

* **YAML-like structured syntax**
* Easy to read, safer to use
* Validated by Jenkins before execution
* Most modern plugins and examples support it

```groovy
pipeline {
  agent any
  stages {
    stage('Build') {
      steps {
        sh 'mvn clean install'
      }
    }
  }
}
```

✅ Best for:

* Simple to medium-complex CI/CD pipelines
* Teams who want **consistency**, **linting**, and **pipeline-as-code**

---

### ⚙️ Scripted Pipeline (Full Groovy DSL)

* Fully based on **Groovy** scripting
* More **flexible**, more **complex**
* Not validated in advance — errors only at runtime
* More powerful for dynamic use cases (custom loops, conditionals, etc.)

```groovy
node {
  stage('Build') {
    sh 'mvn clean install'
  }
}
```

✅ Best for:

* Advanced users needing custom logic
* Legacy pipelines or dynamic job behavior

---

### 📌 Summary Table

| Feature        | Declarative                  | Scripted                |
| -------------- | ---------------------------- | ----------------------- |
| Syntax         | YAML-like structure          | Groovy DSL              |
| Readability    | Easy to read & maintain      | Harder to read          |
| Error catching | Syntax checked before run    | Only fails at runtime   |
| Flexibility    | Moderate                     | Very high               |
| Use case       | Most pipelines (recommended) | Complex or legacy cases |
| Example        | `pipeline { ... }`           | `node { ... }`          |

---

### 🎯 Final One-Liner (Interview):

> “Declarative pipelines are structured and safe for most use cases, while scripted pipelines are Groovy-based and give you more control but are harder to manage and debug.”

Let me know if you want real-world scenarios for when to prefer each.



jenkins agent configure 
jenkins pipeline  
  - ecr login 
  - kon sa tag ka image 
  - kubeconfige configure hona chaiye for jenkins to be reachable 
  - kubectl set image deployment imgname ns  

  ------------------------------------------

  kubeconfig 

  ------------------------------------------

  ansible fetch secrets 

  ------------------------------------

  ingres class ... diff nginx ingres class and alb ingres class

  ----------------------------
  ingre class - alb 
  jb mai alb ingres create krte h to load balancer wo kaise craete krta h alb amd rules or targets kaise create krta h 

annotation hote h like vpc , subnet , sg 
  -----------------------------------------

  reserved hosts , nat