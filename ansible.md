
```

ANSIBLE DIR STRUCTURE
-----------------------

An Ansible **role** is just a self-contained bundle of files organized in a standard way so that anyone (or any playbook) can pick it up and use it without hunting through dozens of unrelated files. Here’s the typical layout, with plain-English explanations and little examples of what goes where:

roles/
└── <rolename>/
    ├── tasks/
    │   └── main.yml
    ├── handlers/
    │   └── main.yml
    ├── templates/
    │   └── myapp.conf.j2
    ├── files/
    │   └── myapp-1.2.3.tar.gz
    ├── defaults/
    │   └── main.yml
    ├── vars/
    │   └── main.yml
    ├── meta/
    │   └── main.yml
    └── README.md

1. `tasks/main.yml`
--------------------
What it is: The heart of your role – this is the list of steps Ansible runs on each host.
Why it matters: You keep all your “install, configure, start” tasks here (or in files you include from here).
Example snippet:


- name: Install MyApp package
  yum:
    name: myapp-{{ myapp_version }}
    state: present

- name: Deploy configuration file
  template:
    src: myapp.conf.j2
    dest: /etc/myapp/myapp.conf
  notify: Restart MyApp

- name: Ensure service is running
  service:
    name: myapp
    state: started
    enabled: true


### 2. `handlers/main.yml`
---------------------------

What it is: Special tasks that only run **once**, after all normal tasks are done, *but only if* you’ve “notified” them.
Why it matters: Keeps restart/reload logic in one spot so you don’t accidentally restart a service multiple times.
Example snippet:

# roles/myrole/handlers/main.yml
- name: Restart MyApp
  service:
    name: myapp
    state: restarted



### 3. `templates/`
---------------------
**What it is:** Jinja2 template files (ending in `.j2`) that get rendered with your variables.
**Why it matters:** Lets you bake in variables, loops, or conditionals into config files.
**Example file:**

# roles/myrole/templates/myapp.conf.j2
[server]
port = {{ myapp_port }}
log_level = {{ myapp_log_level }}

Let’s say you have a service that can optionally run in TLS mode, depending on a variable you set in your playbook or inventory. In your templates/ folder you might have:
# roles/myrole/templates/myapp.conf.j2

[server]
port = {{ myapp_port }}

{% if enable_tls %}
# TLS is turned on
tls_enabled = true
tls_cert_file = {{ tls_cert_path }}
tls_key_file  = {{ tls_key_path }}
{% else %}
# TLS is turned off
tls_enabled = false
{% endif %}

# Only include debug settings if log_level is DEBUG
{% if myapp_log_level == "DEBUG" %}
debug_mode = true
debug_output = /var/log/myapp/debug.log
{% endif %}


### 4. `files/`
-------------------
**What it is:** “Raw” files to copy exactly as-is (no templating).
**Why it matters:** Use it for binaries, archives, SSL certs—anything you don’t need to substitute variables into.
**Example usage in tasks:**

- name: Upload MyApp tarball
  copy:
    src: myapp-1.2.3.tar.gz
    dest: /tmp/



### 5. `defaults/main.yml`
--------------------------
**What it is:** Low-priority, “sensible fallback” variables.
**Why it matters:** Everyone can override these via playbook vars, inventory, or extra-vars—so there’s no surprise if someone needs to tweak a value.
**Example content:**


# roles/myrole/defaults/main.yml
myapp_version: "1.2.3"
myapp_port: 8080
myapp_log_level: "INFO"


---

### 6. `vars/main.yml`
---------------------
**What it is:** Higher-priority, almost “hard-coded” variables.
**Why it matters:** Use this for values you rarely expect users to change (like internal file paths or package names). They override defaults and most other vars, but *can* be overridden with `-e`.
**Example content:**

# roles/myrole/vars/main.yml
install_dir: /opt/myapp
service_user: myapp


### 7. `meta/main.yml`
---------------------
**What it is:** Metadata about the role itself (author, supported platforms) and any role dependencies.
**Why it matters:** Helps when you publish to Ansible Galaxy or if this role needs another role to run first.
**Example content:**

# roles/myrole/meta/main.yml
galaxy_info:
  author: you
  description: Installs and configures MyApp
  license: MIT
  min_ansible_version: 2.9
  platforms:
    - name: Ubuntu
      versions: [focal, jammy]
dependencies:
  - role: geerlingguy.java

```



---

### ✅ **Basic Ansible Interview Questions**

#### 1. **What is Ansible and why is it used?**

> Ansible is an open-source automation tool used for configuration management, application deployment, and task automation. It uses YAML to define tasks and works over SSH, making it agentless and simple to set up.

---

#### 2. **How does Ansible connect to remote machines?**

> It connects over **SSH** by default. For Windows systems, it can use **WinRM**. No agents are required on the managed nodes.

---

#### 3. **What is an Ansible inventory file? What formats does it support?**

> The inventory file lists target hosts. It can be written in:

* INI (default)
* YAML
* Dynamic inventory via scripts or cloud plugins (e.g., AWS EC2)

**Example (INI):**

```ini
[webservers]
web1.example.com
web2.example.com
```

---

#### 4. **What is the difference between a play and a playbook?**

* **Play**: A single set of instructions mapped to a group of hosts.
* **Playbook**: A collection of plays written in YAML.

---

#### 5. **What are Ansible modules?**

> Ansible modules are reusable, standalone scripts that Ansible runs to perform specific tasks like installing packages, managing files, users, services, and more. They are the building blocks of Ansible playbooks.

**Example:**

```yaml
- name: Install nginx
  ansible.builtin.yum:
    name: nginx
    state: present
```
##### Types of modules are there in Ansible?
Ansible modules can be categorized into:
- Core modules: 
- Extras/community modules
- Custom modules: Developed by users for specific tasks

1.  **Core modules**

Core modules are built-in modules that come pre-installed with Ansible. They are fully supported, stable, and cover most common automation tasks like installing packages, copying files, managing users, services, and system configurations.
They don’t require any extra installation — just use them directly in your playbooks.

2. **Extras / Community Modules**

- Community modules are maintained by the Ansible community and provided through collections. 
- These are not part of the Ansible core, so you need to install them separately using ansible-galaxy collection install.
- Let's say I'm automating MySQL user and database creation on remote servers.
Instead of writing shell scripts or using core modules, I can install the community.mysql collection and directly use modules like mysql_user and mysql_db.
```
ansible-galaxy collection install community.mysql

- name: Create a MySQL user
  community.mysql.mysql_user:
    name: appuser
    password: securepass
    priv: '*.*:ALL'
    state: present
```

3. **Custom Modules**

Custom modules are created by users when there’s no existing module to perform a specific task. It’s basically writing your own Python script that accepts arguments from a playbook and returns output in Ansible's expected format.


```
Real-World Use Cases for Custom Modules
---------------------------------------
> **Custom modules are used when no existing Ansible module or collection can handle a unique business requirement**, usually involving:

* Internal APIs
* Legacy systems
* Complex multi-step logic
* Proprietary tools
* Tight integration with organization-specific workflows

 🔹 **1. Internal REST API Integration (CMDB, Ticketing, Approvals)**
-----------------------------------------------------------------------
🧑‍💼 **Scenario:**
Your company has a custom **CMDB** (Configuration Management Database) with its own API. For every server created, you must register it in the CMDB with details like hostname, IP, and environment.

🧠 **Why not use `uri` module repeatedly?**
Too complex, needs headers, retries, error handling. You’d rather wrap it into a clean reusable custom module.

🛠️ **Custom module: `cmdb_register`**

- name: Register host in CMDB
  company.cmdb.register:
    hostname: "{{ inventory_hostname }}"
    ip: "{{ ansible_host }}"
    environment: "production"


> This hides all the API logic and makes playbooks clean and reusable.

-------------------------------------------------------------------------------------------------------


### 🔹 **2. Legacy Infrastructure – Custom Monitoring Agent Installer**
-------------------------------------------------------------------------
🧑‍💼 **Scenario:**
You work at a company that uses a **legacy monitoring tool** that needs:

* RPM install
* Registering to a license server
* Modifying two config files

No existing module handles this exact flow.

🛠️ **Custom module: `legacy_monitor_setup`**

- name: Setup legacy monitoring agent
  legacy.monitor.setup:
    server: "lic01.infra.local"
    key: "{{ license_key }}"


> This avoids complex shell scripting and makes agent installs consistent and idempotent.

-------------------------------------------------------------------------------------------------------

### 🔹 **3. Custom Validations Before Deployment**
--------------------------------------------------

🧑‍💼 **Scenario:**
Before deploying an app, you want to:

* Validate DB connectivity
* Check disk space on `/var`
* Confirm a secret file exists

Rather than writing 10 shell commands + conditionals, you bundle all into one custom module that returns `"ready": true`.

🛠️ **Custom module: `pre_deploy_check`**

- name: Run pre-deployment checks
  infra.validator.pre_deploy:
    db_host: "{{ db_host }}"
    min_disk_gb: 5
    required_files:
      - /etc/secrets/token.txt

-------------------------------------------------------------------------------------------------------

### 🔹 **4. Secure Secrets Fetch from Vaults**
-----------------------------------------------

🧑‍💼 **Scenario:**
You use a **custom secrets manager** (not AWS/GCP Vault) inside your company that can only be accessed via internal SDK.

You write a custom module to:

* Authenticate using a token
* Retrieve secret values
* Export them as facts for later use

🛠️ **Custom module: `get_internal_secret`**

- name: Fetch DB credentials from corp vault
  company.secret.get:
    secret_id: db_prod_credentials
  register: db_creds


# USE CASE OF CUSTOM MODULE 
---------------------------

> I use custom modules when I need clean, reusable logic that Ansible doesn’t support out-of-the-box — especially for:
* Internal APIs (CMDB, secrets manager)
* Complex validation before deployment
* Legacy tool integrations
* Reducing repetition and simplifying playbooks
> They make my automation more maintainable and readable across teams.

```

---

#### 6. **How do you define variables in Ansible?**

> Variables can be defined in:

* Inventory (`inventory.ini`)
* Playbook
* Group/host vars
* `vars` or `vars_files` sections
* Facts (gathered from remote systems)

---

#### 7. **What is `ansible.cfg` and what can you configure there?**

> ansible.cfg is the main configuration file for Ansible. It controls how Ansible behaves — like which inventory file to use, connection settings, logging, privilege escalation, etc:

By default, Ansible looks for this file in the following order:
 - ANSIBLE_CONFIG (environment variable path)
 - ./ansible.cfg (in the current directory)
 - ~/.ansible.cfg (user home directory)
 - /etc/ansible/ansible.cfg (global config)

✅ Does Ansible expect a specific filename?
Yes — Ansible always expects the configuration file to be named ansible.cfg, regardless of the location.


---

#### 8. **What is idempotency in Ansible?**

> Idempotency means that running the same playbook multiple times will not cause unintended changes, as long as the desired state is already achieved.
In other words, Ansible only makes changes when needed, not every time the playbook runs.

```
- name: Ensure nginx is installed
  apt:
    name: nginx
    state: present
```
---

#### 9. **What is the use of handlers in Ansible?**

> Handlers are tasks triggered only when notified. Typically used for restart actions.

```yaml
tasks:
  - name: Update config file
    copy:
      src: my.cnf
      dest: /etc/my.cnf
    notify: Restart MySQL

handlers:
  - name: Restart MySQL
    service:
      name: mysql
      state: restarted
```

```
HANDLERS INTERVIEW QUESTION 
----------------------------

🔸 Q1. Can a handler run multiple times in a playbook?

> No. A handler is triggered only **once per play**, even if it's notified by multiple tasks. It runs at the **end of the play** (after all tasks are done), not immediately after the notify.

-----------------------------------------------------------------------------------

🔸 Q2. What happens if a handler is notified, but the handler task fails?

> If the handler fails, the playbook will also fail unless `ignore_errors: yes` is used in the handler. Since handlers run after all tasks, a failure in a handler can be hard to trace unless logs or tags are used properly.

💡 **Tip:** You can use `force_handlers: yes` to **force handler execution** even if previous tasks failed.

-----------------------------------------------------------------------------------

🔸 Q3. Can handlers trigger other handlers?

> Not directly. Handlers **cannot notify other handlers** like regular tasks can.
> But a handler can include a task file that contains logic, or call a role with its own handlers.

💡 **What this tests:** Knowing the **limits of notify chains** and how to work around them.

-----------------------------------------------------------------------------------

Q4. What if a task fails *before* it can notify a handler? Will the handler run?
> No. If a task fails and doesn’t reach the notify statement, the handler will not be queued. You can use `ignore_errors: yes` to bypass failure, but then you're on your own to trigger the handler logic.

-----------------------------------------------------------------------------------

🔸 Q5. Can handlers use `when` conditions or loops?

> * Handlers **can use `when` conditions.
> * But **handlers cannot use `loop`**, `with_items`, or other looping constructs.

handlers:
  - name: Restart app only on staging
    service:
      name: myapp
      state: restarted
    when: inventory_hostname in groups['staging']

💡 **Tests knowledge** of handler constraints.

-----------------------------------------------------------------------------------

🔸 Q6. Can you run a handler *immediately* when a task changes, not at the end of the play?

> Normally, handlers run at the end.
> But if I need immediate execution, I can use **`meta: flush_handlers`** right after a task block to force them to run early.

tasks:
  - name: Modify config
    copy:
      src: new.conf
      dest: /etc/myapp.conf
    notify: Restart app

  - meta: flush_handlers

-----------------------------------------------------------------------------------

🔸 **Q7. What’s the difference between handlers and normal tasks with `when` and `changed_when`?**

> Tasks can also be conditional and trigger actions, but **handlers are meant to separate restart/reload actions** from logic, ensuring they run only once even if multiple changes happen.
> They are part of good playbook **structure and idempotency**.

-----------------------------------------------------------------------------------

```


---

#### 10. **What is the command to run a playbook?**

```bash
ansible-playbook site.yml
```

You can add:

* `-i inventory.ini` to specify inventory
* `--check` for dry run
* `--limit` to target specific hosts

---

#### 11. **Ansible Strategy**

```
🔹 1. Linear (Default)
---------------------------
How it works:
-------------
Ansible runs **task 1** on **all hosts** before moving to **task 2**, and so on .

When to use it:
------------------
* Ideal for playbooks where tasks must be synchronized across hosts—e.g., updating a cluster in lock-step.
* Ensures consistency and predictability.


🔹 2. Free
------------
How it works:
------------
Hosts progress at their own pace—task 2 on one host may start while others are still on task 1. 

Benefits:
-------------
* Faster execution in heterogeneous environments.
* No host blocks others if it's slow.

Use case:
-----------
Great for independent, non-dependent tasks like log collection or status gathering.
You can enable it in a play:

- hosts: all
  strategy: free
  tasks:
    - …

Or globally in `ansible.cfg`:

[defaults]
strategy = free


 🔹 3. Other Built-in Strategies
------------------------------------

| Strategy         | Description                                                                            |
| ---------------- | -------------------------------------------------------------------------------------- |
| **debug**        | Interactive, step-by-step execution—useful for debugging. ([docs.ansible.com][5])      |
| **host\_pinned** | Similar to `free`, but pins execution to a fixed number of hosts defined by `serial`.  |



🔹 4. Control Flow Options (Not Strategies, but related)
--------------------------------------------------------------
These keywords help shape execution flow:

1. 🧩 serial: Run the play in batches.Example: serial: 3 means Ansible completes all tasks on the first 3 hosts before moving to the next batch2. 

2. throttle: Limits the number of concurrent hosts per task/block. Useful for tasks that are CPU-heavy or API‑rate limited. Doesn't exceed forks or serial limits
🛠️ Example: Limiting Parallel Host Execution for a Heavy Task

- hosts: all
  gather_facts: false
  tasks:
    - name: Run heavy script with throttle
      script: /path/to/heavy_task.sh
      throttle: 2

➡️ What happens:
- Ansible will execute heavy_task.sh on only 2 hosts at a time, regardless of how many forks or hosts you have.
- Once those finish, it processes the next 2 hosts, and so on

3. forks: Sets the total number of parallel processes (hosts) across the playbook.Default is 5, configurable in ansible.cfg or via -f

4. order: The order keyword controls the sequence in which hosts are picked from the inventory.
Common values:
- inventory – default, follows the order in the inventory file
- sorted – alphabetically sorted
- reverse – reverse of inventory
- shuffle – random order

5. run_once: true  It’s useful when the task needs to be done only once, not on every host. Runs a task only on the first host 
🧩 Use Case:  You want to generate a config file once and then copy it to all servers.
This runs the task only once, on localhost, and skips all other hosts.

- name: Generate config on control node
  template:
    src: myapp.j2
    dest: /tmp/generated.conf
  delegate_to: localhost
  run_once: true

------------------------------------------------------------------------------------------

```

---

#### 12. **Dynamic Inventory**

A dynamic inventory is an external script or plugin that Ansible uses to fetch real-time host data from sources like:
- AWS EC2, Azure, GCP, DigitalOcean
- Kubernetes, VMware, OpenStack
- Custom REST APIs or CMDBs
Instead of a static inventory.ini, it dynamically pulls the host list at runtime.

#### ✅ Real-World Use Case

- You're working with AWS and want Ansible to auto-fetch all EC2 instances with the tag env=prod.
```ansible-inventory -i aws_ec2.yml --list```
Where aws_ec2.yml is a plugin config using amazon.aws.aws_ec2.

#### 🔧 How to Use It (Basic)

1. Install the aws_ec2 ansible plugin and its dependencies (boto3 and botocore)
```pip3 install boto3 botocore```

2. Set up ansible.cfg file 
```enable_plugins = aws_ec2```

3. Create inventory ```aws_ec2.yml``` file :
```
plugin: amazon.aws.aws_ec2
regions:
  - ap-south-1
filters:
  tag:Environment: production
keyed_groups:
  - key: tags.Name
```

4. To use this plugin, we need credentials to access other instances. We can do this in two ways.
- Attach Role (aws_profile) [Recommended]
- AWS Credentials (aws_access_key, aws_secret_key)


5. Run playbook with:
```
ansible-playbook -i aws_ec2.yml site.yml
```
> https://www.cloudthat.com/resources/blog/step-by-step-guide-to-integrate-ansible-dynamic-inventory-plugin-for-aws-ec2-instances


```
🧨 Twisted/Advanced Interview Questions on dynamic inventory
-------------------------------------------------------------

🔸 Q1. How does Ansible know which plugin to use in a dynamic inventory file?
We will configure it in config file of ansible (ansible.cfg)
enable_plugins = aws_ec2

🔸 Q2. What’s the difference between script-based and plugin-based dynamic inventories?
- Script-based (legacy): Python/JSON script that outputs JSON.
- Plugin-based (modern): YAML configuration file + plugin logic from collections. Plugin-based is preferred and more maintainable.

🔸 Q3. Can you group hosts using tags or metadata?
Yes, dynamic inventory plugins like aws_ec2, azure_rm, or netbox support options such as keyed_groups and groups (or conditional_groups) to automatically build host groups based on tags, metadata, or custom logic.


```

--- 

#### 13. What are Host Vars and Group Vars?
| Type           | Meaning                                  | Scope     |
| -------------- | ---------------------------------------- | --------- |
| **Host Vars**  | Variables specific to a single host      | Per host  |
| **Group Vars** | Variables shared by all hosts in a group | Per group |

✅ Example: Inventory with Host and Group Vars

#### inventory.ini
```
[web]
web1 ansible_host=192.168.1.10
web2 ansible_host=192.168.1.11

[db]
db1 ansible_host=192.168.1.20
```

#### group_vars/web.yml
```
http_port: 80
env: staging
```

#### host_vars/web1.yml
```
env: production
```

####  Now:
- Both web1 and web2 will get http_port: 80
- But web1 will override env to production

📁 Directory Layout (Best Practice)
```
inventory/
├── hosts.ini
├── group_vars/
│   ├── web.yml
│   └── db.yml
├── host_vars/
│   ├── web1.yml
│   └── db1.yml
```

🧠 When to Use What?
| Use Case                          | Use Type                   |
| --------------------------------- | -------------------------- |
| Shared port or config for a group | Group var                  |
| Unique IP, secret, or path        | Host var                   |
| Override group setting for one    | Host var (higher priority) |

🔺 Precedence (Who wins?)
- If both are defined:
Host vars > Group vars
Example: If both group_vars/web.yml and host_vars/web1.yml define env, the value from host_vars/web1.yml will be used.

🗣️ Interview-Ready Summary
- “Host vars are variables tied to a single host. Group vars are shared by all hosts in a group.
Host vars take precedence over group vars if both are defined.
I typically use group vars for common settings (like ports, roles), and host vars for overrides (like IPs or secrets).”

```
Advance ques on HOST & GROUP Vars 
----------------------------------

🔹 Q1. If the same variable is defined in both group_vars/web.yml and host_vars/web1.yml, which one will Ansible use?
Answer: Ansible uses the value from host_vars/web1.yml.Host vars always take precedence over group vars.

🔹 Q2. What happens if a host belongs to multiple groups and those groups define the same variable?
Answer: Ansible applies group vars in this order of precedence (lowest to highest):
> all
> Parent groups
> Child groups
> Group vars defined in group_vars/
> Inline vars, play vars, host vars (take over from here)
> If two groups define the same variable, the more specific group (child) wins.

🔹 Q3. Can group or host vars be defined inline in the inventory file?
Answer: Yes. You can define vars directly inside inventory.ini like:
web1 ansible_host=192.168.1.10 env=staging
But this has lower precedence than host_vars/.

🔹 Q4. You define a variable in group_vars/all.yml and also pass it using --extra-vars. Which one is used?
Answer: --extra-vars wins. Extra vars have the highest precedence of all.

🔹 Q5. If a host is in multiple groups, how does Ansible determine which group_vars to apply first?
Answer:  Ansible prioritizes variables from **more specific groups (child groups)** over **general or parent groups** when a host belongs to multiple groups.

 🔁 Group Hierarchy: Example Setup / How Ansible Decides "Specificity"
 ----------------------------------

[all]
web1

[web]
web1

[production]
web1

[envs:children]
web
production

This means:
-----------
- web1 is in two child groups: web and production
- Both web and production are children of the parent group envs

Now
------
✅ group_vars/all.yml >> env: "all"
✅ group_vars/envs.yml >> env: "envs"
✅ group_vars/web.yml >> env: "web"
✅ group_vars/production.yml >> env: "production"



🧠 What Ansible Does Internally
-------------------------------------
- When Ansible builds its host/group tree, it walks down from all → envs → its children (web, production) → host (web1)
- Now, when multiple groups define the same variable (env), Ansible resolves conflicts by using the deepest match in the group tree — the most specific level.

        all
         │
       envs
      /     \
   web    production
     \      /
     web1 (host)


➡ Why production wins:
----------------------------
- Even though both web and production are children of envs
- production happens to be loaded after web
- Ansible applies group vars in depth-first, lexical order
So the last matching group's var in that load order wins

⚠️ This is not about children structure alone, it’s also about load order.

✅ Final Rule of Thumb:
-----------------------

When multiple groups match and define the same variable, the last loaded group's variable wins — unless overridden by host_vars or extra-vars.

🗣️ Interview Version
---------------------
Ansible loads group vars from all matching groups, but when there's a conflict, the most specific group wins.
In this case, web1 is in all, envs, web, and production, and production is the most specific child group. So its env value overrides the others.


---

🔹 Q6. Where should you define secrets: in host_vars or group_vars?
Answer: - Use host_vars if secrets are host-specific, like private keys or tokens.
        - Use group_vars if the secret is shared, like a DB password for all web servers.
And ideally, encrypt them using Ansible Vault.

🔹 Q7. What happens if group_vars is a file instead of a directory?
Answer: It works, but the structure must be:

group_vars/
  web.yml

File name must match group name exactly.
If the group is webservers, the file should be webservers.yml.

🔹 Q8. Can you use Jinja2 inside host or group vars?
Answer: Yes — but only for non-recursive references.
Example:
base_dir: "/opt/{{ app_name }}"
Just avoid self-referencing or circular dependency in variable files.
```

---

#### 14. 🧠 What is Variable Precedence?

When the same variable is defined in multiple places, Ansible uses a priority system to decide which value wins.
The higher the precedence, the more power it has to override others.

| Level                                    | Example                                      | Precedence |
| ---------------------------------------- | -------------------------------------------- | ---------- |
| 1. Role defaults                         | `defaults/main.yml` in a role                | 🟢 Lowest  |
| 2. Inventory file/group\_vars/host\_vars | `inventory.ini`, `group_vars/`, `host_vars/` |            |
| 3. Playbook vars                         | `vars:` in a playbook                        |            |
| 4. Task-level vars                       | `vars:` inside a task block                  |            |
| 5. `set_fact`                            | Set dynamically during play                  |            |
| 6. Registered vars                       | From previous tasks                          |            |
| 7. Environment vars                      | `env:` or actual OS env vars                 |            |
| 8. Command-line extra vars               | `--extra-vars "env=prod"`                    | 🔴 Highest |

```
🔹 Q1. If a variable is defined in both host_vars and in a playbook vars:, which one takes precedence?
✅ Answer: The variable defined in the playbook vars: section wins — because play-level vars are higher in precedence than host_vars.

🔹 Q2. Which one wins: a variable defined via set_fact vs group_vars?
✅ Answer:  set_fact wins.
set_fact is dynamic and defined during play execution, so it overrides any static group/host vars.

🔹 Q3. You define env: production in:
> defaults/main.yml (in a role)
> host_vars/hostname.yml
> vars: in play
> and also pass --extra-vars "env=qa"
👉 Which one wins?
✅ Answer: --extra-vars "env=qa" wins.
Extra-vars always have the highest precedence.

🔹 Q4. Can set_fact override --extra-vars?
❌ Answer: No.
Extra-vars are always the highest — even set_fact cannot override them.

🔹 Q5. If a variable is defined in both vars_files and host_vars, which one wins?
✅ Answer: vars_files wins.
Because vars defined explicitly in the playbook (including files) are higher than host_vars.

🔹 Q6. What happens if you use include_vars to load a file that defines a variable already set in group_vars?
✅ Answer: The variable from include_vars wins — because it’s loaded during task execution, just like set_fact.

🔹 Q7. You define the same var in:
- group_vars
- Role defaults/main.yml
- Role vars/main.yml
👉 Which one wins?
✅ Answer: The variable in role vars (vars/main.yml) wins.
Role vars > group_vars > role defaults

```
---

# SET FACTS

```
🎯 Use Case: Setting a Deployment Path Based on OS
---------------------------------------------------------
Imagine you're deploying an app on different Linux distributions. The app path is different depending on the OS.

You want: 
> /opt/myapp on Ubuntu
> /var/lib/myapp on RedHat

🔧 Without set_fact — this gets messy:
You’d need when conditions everywhere for each path.

✅ With set_fact — it's clean and dynamic:
---------------------------------------------

- name: Set deploy path based on OS
  set_fact:
    deploy_path: "{{ '/opt/myapp' if ansible_facts['os_family'] == 'Debian' else '/var/lib/myapp' }}"


Now you can use {{ deploy_path }} in all tasks that need the app path.

- name: Copy app files
  copy:
    src: ./app/
    dest: "{{ deploy_path }}/"

🧠 Why This Helps
-------------------
> Makes your playbooks dynamic and environment-aware
> Helps reduce conditionals in every task
> Keeps logic in one place, easy to change

🗣️ Interview Way to Say It:
-----------------------------
I use set_fact when I need to define a variable dynamically based on OS, API results, or previous task output.
For example, in one playbook I set a deploy_path using set_fact based on the server’s OS, and reused it cleanly across 5 tasks.
It made my playbook cleaner and removed repeated when conditions
```
---

# Async Poll Interval

Ansible lets you run long-running tasks asynchronously using the async and poll options. These help you not block the playbook while waiting.

✅ Basic Concept:
- async: Maximum runtime (in seconds) for the task
- poll: How Ansible checks if the task is done

📘 Example
```
- name: Run a long job in background
  shell: /usr/bin/long_process.sh
  async: 300     # let it run for up to 5 mins
  poll: 0        # don't wait, run it in background

This starts the job and moves on immediately (like fire-and-forget).
```

🔁 To check status later:
```
- name: Wait for long job to complete
  async_status:
    jid: "{{ job_result.ansible_job_id }}"
  register: job_status
  until: job_status.finished
  retries: 10
  delay: 15
jid: Job ID returned from the async task

poll here is replaced by until, retries, and delay
```

🎯 Common Scenarios to Use It:
- Waiting for large file copy
- Long database migration
- Triggering scripts on multiple hosts in parallel
- System reboot tasks


# Raw Module 
- The raw module lets you run raw shell commands on a remote host without needing Python installed.
It sends the command as-is over SSH — no formatting, no modules, no facts, nothing else.

🧠 Why is this useful?
- Because Ansible normally requires Python on the target machine. But what if Python is missing or broken?

✅ Use raw to:
- Install Python
- Run quick commands on very minimal systems (e.g. Docker, new EC2s, routers)

📘 Example: Install Python on a fresh server
```
- name: Install Python on minimal server
  raw: apt-get update && apt-get install -y python3

Once Python is installed, you can switch to normal Ansible modules like apt, copy, template, etc.
```
---

# ⚡ How to Make Ansible Playbooks Faster

#### 🔹 1. **Increase Forks**
Ansible runs tasks in parallel across hosts using **forks**.

```ini
# in ansible.cfg
[defaults]
forks = 20
```
⏱️ Default is just 5 — increasing it allows Ansible to target more hosts at once.

---

#### 🔹 2. **Use `async` + `poll` for long-running tasks**
Run time-consuming tasks in the background:
```yaml
- name: Run long task
  shell: ./heavy.sh
  async: 300
  poll: 0
```
Then use `async_status` to check progress.

---

#### 🔹 3. **Disable Fact Gathering (if not needed)**
```yaml
- hosts: all
  gather_facts: false
```
⏱️ Speeds up playbooks that don't need host facts.

---

#### 🔹 4. **Use `tags` and `--tags` to run only specific tasks**

```yaml
- name: Restart nginx
  service:
    name: nginx
    state: restarted
  tags: restart
```

Run it with:

```bash
ansible-playbook play.yml --tags restart
```

✅ Skips everything else.

---

#### 🔹 5. **Limit Hosts During Testing**
```bash
ansible-playbook play.yml --limit web1
```
✅ Test on a subset before running on full inventory.

---

#### 🔹 6. **Avoid `shell`/`command` unless needed**

Use**native modules** like `yum`, `apt`, `copy`, etc. They are **faster** and **idempotent**, unlike `shell`.

---

### 🔹 7. **Use `serial` to batch hosts**

Avoid bottlenecks on large inventories:
```yaml
- hosts: all
  serial: 10
```
⏱️ Runs in chunks instead of all at once (helps balance speed + safety).

---

### 🔹 8. **Fact Caching**
Use Redis or JSON fact cache so facts aren't re-collected every time:
```ini
[defaults]
gathering = smart
fact_caching = jsonfile
fact_caching_connection = ./facts
```
---

#### 🔹 9. **Use `check_mode` for dry runs**
Run playbooks in dry-run mode to test logic without making changes:
```bash
ansible-playbook play.yml --check
```

---

#### 🔹 10. **Avoid unnecessary `register`/`set_fact`**
They're stored in memory and slow down execution if overused.

---

## 🗣️ Interview Summary
> I speed up Ansible playbooks by increasing forks, disabling unnecessary fact gathering, using async for long tasks, and applying tags to run only what’s needed.
> I also use serial batching for large inventories and avoid shell commands in favor of built-in modules.


---

# Difference between ROLES & PLAYBOOK
A playbook is the main YAML file that defines what Ansible should do on which hosts.
A role is a reusable, structured way to organize tasks, vars, files, and templates.
I use roles to keep playbooks clean and maintainable — especially in large projects or multi-environment setups.

---


# Secrets in Ansible  ? 
WATCH VIDEO


# 5 Config in Ansible 
----------------------

1- host_key_checking
------------------------- 
**If host_key_checking = **True** (default):**
It will fail with an error if that host's key is not already in known_hosts
It won’t prompt for yes/no because Ansible runs non-interactively

**If host_key_checking = **False**:**
Ansible will automatically accept that host's key
It won’t ask anything and will connect right away
It skips the prompt just like you answered “yes” in advance

2- forks = 5
----------------
n Ansible, a fork refers to a separate process that runs in parallel to execute tasks on remote hosts.
When Ansible runs a playbook, it doesn't execute tasks on all hosts one-by-one. Instead, it creates multiple forks (parallel connections) to run tasks on multiple hosts at the same time.

🧠 Suppose: You have 10 host , And your ansible.cfg is set like this: ```forks = 5``` 
And your playbook contains:
```
tasks:
  - name: Task 1
    shell: echo "Hello"

  - name: Task 2
    shell: echo "World"

```
🔁 What happens?
🔹 Task 1 execution:
Ansible will run Task 1 on 5 hosts at once (forks = 5) .
When those 5 are done, it runs Task 1 on the next 5:
🔹 Then Task 2 begins (same pattern):
Task 2 → runs on host1 to host5 (parallel)
Then on host6 to host10
🔁 It’s parallel per task, and batch size = forks.

3- inventory	
--------------
Default path to inventory file (e.g. inventory.ini)

4- ask_pass	
--------------
If True, prompt for SSH password (for non-key-based auth)

5- become vs ask_become_pass 
------------------------------
| Option            | Meaning                                                          |
| ----------------- | ---------------------------------------------------------------- |
| `become`          | **Enables privilege escalation** (e.g., using `sudo`)            |
| `ask_become_pass` | **Prompts for the sudo password** if the target host requires it |

🧠 Think of it like this:
- become = True → “I want to use sudo (or su) to run tasks as another user, usually root”
- ask_become_pass = True → “Ask me for the sudo password if it's required”






