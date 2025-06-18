
# 1- Developed an Ansible role to configure monitoring across 73 servers, significantly enhancing infrastructure observability and reducing manual effort.

1. What was the purpose of the Ansible role you developed?
I wrote the role to automate observability across all production hosts. Its immediate goal was to deploy Node Exporter to every running instance .  The most critical metric we needed right away was disk-space utilisation (several incidents had been caused by silent volume fill-ups),but the exporter also gave us CPU, memory, and networking insight.
By codifying this in an Ansible role we eliminated repetitive, manual installs and ensured new servers were monitored automatically via our dynamic inventory.”

2. What monitoring tool(s) did your role configure?
Node Exporter on all Ubuntu application and database nodes (73 servers). Prometheus (scrape-target configuration + systemd service) and Grafana (provisioning of data-source and default dashboards)

3. 4. How did you ensure the role was idempotent?
A: by using idempotent modules like lineinfile , copy , file , apt , get_url module can be made idempotent if **checksum:** , unarchive module  is used with **creates:** or **checksum:** for idempotence
*  Archive extraction used creates: so the task runs only when the binary isn’t there.
       - Let’s say your Ansible role is extracting a Node Exporter .tar.gz file like this:
  ```
- name: Extract Node Exporter binary
  unarchive:
    src: /tmp/node_exporter.tar.gz
    dest: /opt/node_exporter/
    remote_src: yes
  args:
    creates: /opt/node_exporter/node_exporter
```
🔍 So What Does creates: Do?
- Ansible checks if the file /opt/node_exporter/node_exporter already exists.
- If it does exist, Ansible skips this task.
- If it does not exist, Ansible runs the task to extract the archive.

Why Is This Useful?
It makes the role idempotent: Prevents unnecessary re-extraction every time you run the playbook.


4. How did you structure your Ansible role?
```
roles/
└─ monitoring/
   ├─ tasks/
   │  ├─ install_node_exporter.yml
   │  ├─ configure_prometheus.yml
   │  └─ configure_grafana.yml
   ├─ handlers/
   │  └─ restart_services.yml
   ├─ templates/
   │  ├─ prometheus.yml.j2
   │  └─ grafana_datasource.yml.j2
   ├─ defaults/
   │  └─ main.yml          # ports, versions, scrape interval
   ├─ vars/
   │  └─ main.yml          # derived facts & host-specific vars
   ├─ files/
   │  └─ node_exporter-1.8.1.linux-amd64.tar.gz
   └─ meta/
      └─ main.yml          # galaxy info & dependency stub
```
I keep defaults for sane global values, vars for anything driven by facts or inventory, and a single restart handler that individual tasks notify only when their template or package changes.

5. What modules did you use in the role?
| Module                      | Why it was used                                                  |
| --------------------------- | ---------------------------------------------------------------- |
| `apt`                       | Install Node Exporter, Prometheus, Grafana (via official repos). |
| `unarchive` / `get_url`     | Download & unpack Node Exporter tarball when repo not desired.   |
| `template`                  | Render `prometheus.yml`, Grafana data-source, and dashboards.    |
| `lineinfile`                | Append scrape configs for edge cases without re-templating file. |
| `service` / `systemd`       | Enable & start services; control restarts through handlers.      |
| `copy`                      | Ship pre-built Grafana dashboards JSON into `/var/lib/grafana/`. |
| `command` (with `creates:`) | Reload Grafana provisioning API post-copy.                       |

6. Did you handle OS differences (Ubuntu vs RHEL)? How?
In this environment, all 73 nodes were running Ubuntu. However, the setup can be made flexible by using Ansible's gather_facts feature along "with" conditional logic based on the ansible_os_family variable.

7. Any challenge you faced ? 
One of the main challenges I faced while setting up monitoring was the diversity in Ubuntu versions across our infrastructure — we had servers ranging from Ubuntu 14.04 up to 22.04. This created compatibility issues, especially with system dependencies and service management (like systemd vs init.d). I had to carefully choose versions of Node Exporter, Prometheus, and Grafana that would work reliably across all these environments. In some cases, I used static binaries for Node Exporter to avoid package conflicts, and made sure the Ansible role handled service registration appropriately depending on the OS version.”
OR 
One challenge I faced in setting up monitoring was handling multiple Ubuntu versions (14.04 to 22.04). To ensure compatibility, I used static Node Exporter binaries and tailored the Ansible role for service management based on the OS version.

8. Architecture of node exporter , prometheus , grafana ?
🏗️ Architecture: Node Exporter + Prometheus + Grafana
  ```
                ┌────────────────────────────┐
                │        Grafana UI          │
                │  (Dashboards & Alerts)     │
                └────────▲────────▲──────────┘
                         │        │
                         │        │
                ┌────────┴────────┴──────────┐
                │     Prometheus Server      │
                │ - Scrapes metrics          │
                │ - Stores time-series data  │
                │ - Alert rules              │
                └────────▲────────▲──────────┘
                         │        │
                ┌────────┘        └────────────┐
                │                             │
      ┌─────────┴─────────┐        ┌──────────┴─────────┐
      │   Node Exporter   │        │    Node Exporter   │
      │   (Server A)      │        │    (Server B)      │
      └───────────────────┘        └────────────────────┘
```
🔹 A. Node Exporter (Installed on Each Monitored Server)
Purpose: Exposes hardware and OS-level metrics as HTTP endpoints (in Prometheus format). Examples of metrics:
> CPU usage   > Memory   > Disk I/O   > Network  > Filesystem usage
> Listens on: :9100 by default (e.g., http://server:9100/metrics)
> Lightweight & stateless: No config, no DB, just exposes /metrics.

🔹 B. Prometheus Server (Usually on Bastion or Monitoring Node)
* Purpose:
    > Pulls metrics from each Node Exporter every N seconds.
    > Stores all data as time-series in its own internal database (TSDB).
    > Evaluates alert rules (e.g., disk > 90% → alert).
* Config file: prometheus.yml defines scrape targets and intervals.
* Pull-based model: Prometheus reaches out to Node Exporters (not the other way).
* Also exposes metrics for itself (monitor Prometheus via Prometheus!).

🔹 C. Grafana (Visualization Layer)
* Purpose:
    > Connects to Prometheus as a data source.
    > Lets users create interactive dashboards and custom visualizations.
    > Sends alerts via email, Slack, PagerDuty, etc.
* Runs as a web UI: default on http://localhost:3000
* Provisioning: Can load dashboards and data sources via JSON or YAML templates.

🔁 How They Work Together
- Node Exporter runs on each server → exposes metrics at /metrics.
- Prometheus scrapes these metrics periodically and stores them.
- Grafana reads from Prometheus to render beautiful graphs and dashboards.

9. How you would configure security groups (SGs) in a cloud environment
Let’s assume:
suppose prometheus/grafana on bastion host & node exporter on all servers

🔹 A. Node Exporter SG (attached to app/database servers)
| Direction | Type       | Protocol | Port   | Source                         | Reason                                         |
| --------- | ---------- | -------- | ------ | ------------------------------ | ---------------------------------------------- |
| Inbound   | Custom TCP | TCP      | `9100` | **Monitoring SG / Bastion IP** | Allow Prometheus to scrape metrics             |
| Outbound  | ALL        | ALL      | ALL    | 0.0.0.0/0                      | Allow internet/Prometheus connection (default) |

🔹 B. Prometheus + Grafana SG (attached to bastion/monitoring node)
| Direction | Type       | Protocol | Port   | Source                   | Reason                             |
| --------- | ---------- | -------- | ------ | ------------------------ | ---------------------------------- |
| Inbound   | HTTP       | TCP      | `3000` | **Your IP / Admin CIDR** | Access Grafana UI from browser     |
| Inbound   | Custom TCP | TCP      | `9090` | **Your IP / Admin CIDR** | Access Prometheus UI               |
| Outbound  | Custom TCP | TCP      | `9100` | **Node Exporter SG**     | Scrape metrics from Node Exporters |

🔐 Best Practices
- Avoid 0.0.0.0/0 on port 9100 — limit it to only Prometheus host.
- Use Security Group references instead of raw IPs (e.g., "allow Bastion SG to access App SG on 9100").
- Restrict access to Grafana UI (3000) and Prometheus UI (9090) to VPN or specific IPs.
- Enable basic auth / reverse proxy / NGINX + TLS if exposing UIs.

🧱 Diagram Summary
```
[Grafana:3000] <--- Your IP / VPN
[Prometheus:9090] <--- Your IP / VPN
   |
   └─> [Node Exporter:9100] on each App/DB Server
```

10. explain full workflow of set up that was done ?
🧠 Full Flow Explained Simply (Your Setup)

* Prometheus Setup:
  > Prometheus scrapes metrics from EC2 instances using ec2_sd_config (based on tags and instance state).
  > Relabeling is done to get instance name, ID, and private IP to show up in Grafana as variables like “Server”.

* Grafana Dashboards
  > Grafana uses Prometheus as a data source.
  > Dashboards are built for CPU, disk, memory etc. with dropdowns for EC2 instance names (server_name).
  > Variable selection allows users to pick a server and see metrics just for that one.

* Grafana Alerting
  > Alert rules (as seen in the "Alert Rules" tab) are defined directly on dashboard panels or in the alerting UI.
  > These rules use PromQL queries, just like Prometheus would.
  > Example: alert when CPU usage > 80% for 5 mins.

 * Contact Points
 > You’ve set up contact points in Grafana (email, Google Chat).
> These are used directly by Grafana, not via Alertmanager.
> Each alert rule is tied to a notification policy which links to the contact point.
> Example: critical alerts go to Slack or email.

11. Relabelling concept and how it is used in our set up ?
🧠 What is Relabeling in Prometheus?
Relabeling is a way to transform metadata before it’s stored or scraped. Think of it as a filter or editor for labels — you can:
  - Add, remove, rename, or modify labels
  - Control which targets get scraped or how they're named
  - Organize or enrich metrics using instance info (like name, IP, ID)
  - Relabeling happens at different stages:
          * Target relabeling: Before scraping, to modify the target's address or labels
          * Metric relabeling: After scraping, to modify labels on metrics themselves (less common)
)

🔍 Your Setup: Where Relabeling Happens
Your config:
```

- job_name: 'ec2'
  ec2_sd_configs:
    - region: us-east-1
      filters:
        - name: "instance-state-name"
          values: ["running"]
  relabel_configs:
    - source_labels: [__meta_ec2_private_ip]
      target_label: __address__
      replacement: "$1:9100"
    - source_labels: [__meta_ec2_tag_Name]
      target_label: server_name
    - source_labels: [__meta_ec2_instance_id]
      target_label: instance_id
```

Let’s explain this step-by-step
🔗 EC2 SD (Service Discovery)
This line: ``` ec2_sd_configs: ``` means Prometheus will dynamically discover all EC2 instances in the given region (here, us-east-1) with the filter you gave:
```
- name: "instance-state-name"
  values: ["running"]
```
It only looks at instances in "running" state.

For each discovered instance, Prometheus auto-assigns meta labels like:
| Meta Label               | Example Value   |
| ------------------------ | --------------- |
| `__meta_ec2_private_ip`  | `172.31.25.100` |
| `__meta_ec2_tag_Name`    | `blog-prod-01`  |
| `__meta_ec2_instance_id` | `i-0abcd1234`   |

🔧 Then Comes Relabeling
Now let’s map how each of your relabel_configs works:

🔹 A. Replace __address__ with EC2 private IP + port
```
- source_labels: [__meta_ec2_private_ip]
  target_label: __address__
  replacement: "$1:9100"
```
* Prometheus expects a target like host:port
* By default, it won’t know where Node Exporter is running
* So this line says: “Take the private IP of the EC2 instance and assume metrics are on port 9100”
* 🔍 Result: If EC2 private IP = 172.31.25.100, the target becomes: 172.31.25.100:9100

9100

🔹 B. Add a label called server_name from the EC2 Name tag
```
- source_labels: [__meta_ec2_tag_Name]
  target_label: server_name
```
- This means: “Attach the instance Name tag (like blog-prod-01) to the label server_name”
- This is what shows up in your Grafana dropdown (Server variable)

🔹 C. Add a label for instance ID
```
- source_labels: [__meta_ec2_instance_id]
  target_label: instance_id
```
 - Adds a label like instance_id="i-0123456789abcd"
 - This is not directly used in your dashboards (maybe in alerts or traceability).

🖼️ In Grafana (as shown in your screenshot)
Thanks to relabeling: You have dropdown filters like Server and Job in Grafana. These are populated using labels like server_name, which came from your EC2 instance tags via relabeling

🎯 Summary (For Interviews)
"In our Prometheus setup, we used ec2_sd_configs to dynamically discover running EC2 instances. Then we applied relabeling to:
  - Set the correct scrape target using the instance’s private IP and port (Node Exporter on 9100)
  - Extract the EC2 Name tag into a custom label server_name
  - Add instance_id as a label for traceability.
These relabeled values were used in Grafana to build dashboards with server dropdowns and targeted alert rules."

---------------------------------------------------------------------------------------------------------------------------------------------

Q- relabelling in my set up 
Absolutely — let's go deeper step by step into **Prometheus relabeling**, how it connects to **EC2 service discovery**, and how Grafana variables like `$job`, `$node`, etc., work with those labels.

---

## 🔁 Step 1: Prometheus EC2 Service Discovery + Relabeling

Prometheus discovers EC2 instances using `ec2_sd_configs`. Here's a simplified version of what it might look like in your `prometheus.yml`:

```yaml
- job_name: 'node_exporter_ec2'
  ec2_sd_configs:
    - region: us-east-1
      port: 9100
      filters:
        - name: tag:Monitoring
          values: [enabled]
  relabel_configs:
    - source_labels: [__meta_ec2_private_ip]
      target_label: __address__
      replacement: $1:9100

    - source_labels: [__meta_ec2_tag_Name]
      target_label: server_name

    - source_labels: [__meta_ec2_instance_id]
      target_label: instance_id
```

### 📌 What This Means

| Relabel Config                         | Purpose                                                        |
| -------------------------------------- | -------------------------------------------------------------- |
| `__meta_ec2_private_ip ➝ __address__`  | Sets target to `<private_ip>:9100` for scraping                |
| `__meta_ec2_tag_Name ➝ server_name`    | Adds a new label called `server_name` using the EC2 tag `Name` |
| `__meta_ec2_instance_id ➝ instance_id` | Adds the EC2 instance ID as a label                            |

So, every time Prometheus scrapes metrics, each metric will look like:

```txt
node_cpu_seconds_total{job="node_exporter", instance="172.31.12.5:9100", server_name="prod-app-1", instance_id="i-1234abcd"}
```

These **labels become available in Grafana** as variables!

---

## 🧩 Step 2: Grafana Variables

You’ve defined these:

### ✅ `job` variable

```text
label_values(up, job)
```

* Lists all jobs Prometheus is scraping (`node_exporter`, `blackbox`, etc.)
* Used to filter all other variables

---

### ✅ `node` variable

```text
label_values(node_uname_info{job="$job"}, nodename)
```

Here’s what this does:

* Prometheus collects the `node_uname_info` metric from node\_exporter
* That metric has a label `nodename` (which comes from the OS)
* This variable lists **all nodenames** for the selected job

Example:

```txt
node_uname_info{nodename="ubuntu18-blog-02", job="node_exporter", instance="172.31.1.2:9100"}
```

So the dropdown shows:

```
ubuntu18-blog-01
ubuntu18-blog-02
ubuntu-staging
...
```

> You can then use `$node` in your dashboard panels to filter specific hosts.

---

### ✅ `diskdevices` variable

```text
[a-z]+|nvme[0-9]+n[0-9]+|mmcblk[0-9]+
```

* This is a **regex variable** — not tied to any Prometheus metric directly
* You’re probably using it inside a PromQL query like:

  ```promql
  node_disk_read_bytes_total{device=~"$diskdevices"}
  ```

---

## 🧠 Key Concept: Relabeling Makes Labels → Grafana Can Use Them!

Prometheus does not scrape EC2 "name tags" by default. It scrapes metrics from `/metrics`, which don’t include that info. But with **`relabel_configs`**, you can inject that EC2 metadata (e.g., Name, instance ID) as labels.

Grafana only sees **Prometheus labels**. So:

* You inject `server_name` → now you can make a Grafana variable like:

  ```text
  label_values(up, server_name)
  ```

* You inject `instance_id` → now you can filter dashboards using EC2 IDs

---

## 📊 How You Use Variables in Panels

Say you make a CPU Usage panel with this query:

```promql
rate(node_cpu_seconds_total{job="$job", nodename="$node", mode="user"}[5m])
```

Here:

* `$job` controls which Prometheus job to scrape (ex: node\_exporter)
* `$node` filters on hostname (from `nodename` label)

Your user picks from the dropdowns → dashboard auto-filters.

---

## 📎 Optional: Add EC2 `Name` tag as a Grafana variable

If you added `server_name` in relabel configs, you can add this in Grafana:

* Variable name: `ec2name`
* Definition:

  ```text
  label_values(up{job="$job"}, server_name)
  ```

Then use `$ec2name` in your dashboards or in panel titles.

---

## 🧩 Recap (End-to-End Flow)

1. **EC2 instances are discovered** dynamically using `ec2_sd_configs`
2. **Relabeling injects EC2 metadata** like tags into Prometheus labels
3. **Prometheus scrapes metrics**, now enriched with labels like `server_name`, `instance_id`, `nodename`
4. **Grafana uses those labels** to create templating variables
5. **Dashboards dynamically filter** on job, node, disk device, etc.

---

Let me know if you'd like to:

* See a **real example of `prometheus.yml`** with relabeling
* Add new Grafana variables for instance ID or `server_name`
* Or understand how `label_values()` works in more depth with [relabeling](f) and dynamic dashboards





======================================================================================================================================================================
# 2- Migrated S3 buckets from North Virginia to Mumbai region, ensuring data availability and compliance.

