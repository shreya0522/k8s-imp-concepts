
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
I chose aws s3 sync because it’s simple, zero-cost, and quick to run using familiar CLI tools. While DataSync does offer extra features (ACL preservation, built-in validation, scheduling), setting it up—creating agents, permissions, IAM roles—introduced delays. For our use case, the benefits didn’t outweigh the setup overhead.

======================================================================================================================================================================

# 3. Addressed diverse tasks, including resolving upstream issues, clearing server storage, fixing Jenkins pipeline errors, scaling servers, and mitigating 2-3 critical downtimes to ensure uninterrupted operations

## nginx & upstream issue 

In our production stack, traffic first hits an AWS ALB front door, then an NGINX-based API-gateway tier that forwards to a second layer of ALBs (booking, payments, etc.). Because Amazon load-balancers publish ephemeral IPs that can change at any time , NGINX occasionally cached an outdated address for an upstream and started returning “110: no host found” errors. 

A) Root Cause — Stale DNS in NGINX
| Symptom                                              | Technical Explanation                                                                                                                                                                                                                                                |
| ---------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Intermittent 502/504 with `upstream timed out (110)` | Open-source NGINX **resolves the upstream hostname only once at worker start-up** and stores the resulting IP in memory([serverfault.com][1], [stackoverflow.com][2]).                                                                                               |
| Why it broke                                         | Each internal ALB is addressed by a DNS name whose A-records can rotate at any moment for scaling or AZ failover([stackoverflow.com][3], [repost.aws][4]). When an ALB’s IP changed, NGINX kept sending traffic to the now-stale IP, triggering connection timeouts. |

[1]: https://serverfault.com/questions/240476/how-to-force-nginx-to-resolve-dns-of-a-dynamic-hostname-everytime-when-doing-p?utm_source=chatgpt.com "How to force nginx to resolve DNS (of a dynamic hostname ..."
[2]: https://stackoverflow.com/questions/26956979/error-with-ip-and-nginx-as-reverse-proxy?utm_source=chatgpt.com "Error with IP and Nginx as reverse proxy - Stack Overflow"
[3]: https://stackoverflow.com/questions/3821333/does-amazon-ec2-elastic-load-balancers-ip-ever-change?utm_source=chatgpt.com "Does Amazon EC2 Elastic Load Balancer's IP ever Change?"
[4]: https://repost.aws/questions/QUyjryn7t7SOOhYQDtpfR2pg/application-load-balancer-ip-change-event?utm_source=chatgpt.com "Application Load Balancer IP Change Event | AWS re:Post"


B) Immediate Mitigation
- Rebuilt NGINX with the dynamic upstream resolve patch (a third-party module that re-queries DNS on every health-check) to stop hard-caching addresses.
- Added a resolver directive pointing to AmazonProvidedDNS so that DNS lookups respect TTLs
Impact: Errors stopped, but the custom build created a maintenance burden and exposed an unrelated bug in that patch version.

C) Permanent Fix — Upgrade to NGINX 1.27.3
> From OSS 1.27.0 onward, NGINX native upstreams support the server <hostname> resolve; flag inside upstream blocks, dynamically re-resolving DNS without third-party modules. So I upgraded to NGINX 1.27.3 (the first stable build after the patch), removed the custom module, and set
```
upstream booking_backend {
    zone booking 64k;
    server booking-alb.internal resolve;
}
resolver 169.254.169.253 valid=10s;  # AmazonProvidedDNS
```
Each worker now refreshes the ALB’s IP list every 10 s (TTL-bound)

D) How to Phrase It in the Interview
Our API gateway (open-source NGINX) cached the IPs of internal ALBs. When AWS rotated those IPs, NGINX threw 110: no host found. I first hot-fixed the issue by recompiling NGINX with a dynamic-DNS module, then adopted the native server … resolve; feature in NGINX 1.27.3. That upgrade removed the custom patch, respected DNS TTLs, and permanently stopped upstream-resolution outages.

---------------------------------------------------------------------------------------------------------------------------------------------------

## ✅  What is AWS Resolver?
AWS Route 53 Resolver is the DNS service inside VPCs. It:
- Resolves public DNS (like google.com)
- Resolves private DNS (like internal-api.company.local)
- Can be used by EC2, ECS, Lambda, etc
- 🔹 Special IP Address : Inside every VPC, AWS provides a built-in DNS resolver at: ```169.254.169.253```  This is called the AmazonProvidedDNS. You can use this IP in /etc/resolv.conf or in NGINX resolver directives to resolve DNS inside the VPC.

What are resolver and server … resolve; in NGINX?
🔹 resolver in NGINX  Defines which DNS server NGINX should use for runtime lookups.Example: ```resolver 169.254.169.253 valid=10s;```   169.254.169.253 is AWS's internal DNS. valid=10s tells NGINX to cache DNS results for 10 seconds.
🔹 ```server ... resolve; in NGINX``` This is used inside an upstream block to tell NGINX: “Don’t cache this server’s IP forever — re-resolve it periodically.”
Example: This is available in NGINX 1.27.0+.
```
upstream backend_service {
    server my-service.internal resolve;
}
```

Interview Phrase
"In AWS, I used resolver 169.254.169.253 in NGINX to resolve internal DNS names like ALBs. I combined that with server ... resolve; in upstream blocks (NGINX 1.27+) to make sure IPs were refreshed automatically. This prevented stale-DNS issues when ALB endpoints changed.

---------------------------------------------------------------------------------------------------------------------------------------------------

## Incident Management (Downtime)

 Elasticsearch Cluster Instability
----------------------------------
One of the major downtimes I resolved involved a persistent restart issue in our Elasticsearch cluster (v6.8). The cluster was unstable and kept flapping — nodes were disconnecting and reconnecting repeatedly, and indexing stopped. Upon analyzing the logs, I found multiple underlying root causes.”

🔍 Root Cause 1: Split-Brain Scenario
- Elasticsearch allows any master-eligible node to be elected as master.
- Our discovery.zen.minimum_master_nodes was not configured, which led to split-brain — multiple nodes thought they were master.
- As per official guidance, it should be set to: (number of master-eligible nodes / 2) + 1
- In our case, we had 3 master-eligible nodes, so the value should’ve been: ```discovery.zen.minimum_master_nodes: 2```
- Once configured and rolled out consistently, master election stabilized.

🔍 Root Cause 2: Discovery Configuration Inconsistency
- The parameter discovery.zen.ping.unicast.hosts was not consistently set across nodes.
- Some nodes had missing or incorrect IPs of peer nodes.
- This prevented proper cluster formation and caused intermittent network partitioning.
- I corrected this by ensuring all master/data nodes had the same list of IPs:
```
discovery.zen.ping.unicast.hosts: ["172.30.6.121:9300","172.30.6.119:9300","172.30.6.22:9300","172.30.6.245:9300","172.30.6.169:9300"]
```

🔥 Secondary Failure: OOM Crashes
- After the cluster stabilized, we faced another issue: OutOfMemoryErrors (OOM).
- The heap was crashing under load, and I noticed:
```
-Xms8g
-Xmx8g
```
Which was too low for our workload. According to Elastic best practices:
- Heap size should be 50% of total RAM, but not more than 32GB. so I adjusted:

✅ Interview Summary Line
“I resolved a major production downtime involving Elasticsearch 6.8. The root cause was a split-brain issue due to missing minimum_master_nodes and inconsistent unicast.hosts. Once fixed, the cluster stabilized — but later hit OOMs, which I solved by increasing the Java heap size to 32GB. I also added alerting and documented the fix for future recovery.”

---------------------------------------------------------------------------------------------------------------------------------------------------

Amazon ElastiCache for Redis
----------------------------
🛑 Situation
| Item                | Detail                                                                                                      |
| ------------------- | ----------------------------------------------------------------------------------------------------------- |
| **Service**         | ElastiCache ( Redis cluster, single shard, in-memory store for sessions & API responses )                   |
| **Symptom**         | *Primary node went “In‐Memory OOM”* → replica promotion loops → application latency spikes → partial outage |
| **Immediate Cause** | Redis reported `ERR maxmemory limit reached` every few seconds; INFO command showed *memory almost 100 %*.  |

🔎 Root-Cause Analysis
1. No TTLs on many keys: The application team had recently introduced several new caches, but forgot to set EXPIRE. Keys piled up indefinitely; dataset size grew ~4 × in two days.
2. Default eviction policy = noeviction (ElastiCache default). When memory was full, Redis refused writes instead of freeing space, triggering errors and failovers.
3. Metrics Evidence:
   - DatabaseMemoryUsagePercentage raced from 65 % → 98 %.
   - CurrItems kept increasing; EvictedKeys remained 0.

🛠️ Actions Taken
| Step                                | Command / Setting                                                 | Purpose                                                                       |                                                        |
| ----------------------------------- | ----------------------------------------------------------------- | ----------------------------------------------------------------------------- | ------------------------------------------------------ |
| **1. Enabled automatic eviction**   | `maxmemory-policy allkeys-lru`                                    | Allow Redis to drop the **Least-Recently-Used** key when memory hits the cap. |                                                        |
| **2. Added 20 % headroom**          | `reserved-memory 20` (ElastiCache parameter)                      | Avoid edge-case crashes during rehashing and failover.                        |                                                        |
| **3. Kicked off immediate cleanup** | \`redis-cli --scan                                                | xargs redis-cli DEL\` (targeted large unused keys)                            | Lower memory pressure while policy change took effect. |
| **4. Introduced key TTL standards** | Updated application code → `SET key value EX 1800`                | Ensure new writes disappear automatically after 30 min.                       |                                                        |
| **5. Monitoring & Alerts**          | CloudWatch Alarm: `DatabaseMemoryUsagePercentage > 80% for 5 min` | Proactive notification before next incident.                                  |                                                        |

✅ Result
- Outage contained in ~15 minutes; writes resumed once LRU evicted cold data.
- Long-term dataset stabilized at ~55 % utilisation.
- No repeat OOMs in the following 6 months.

📖 Lessons & Preventive Controls
- Always pair SET with TTL – enforced via a helper wrapper in the codebase.
- Select an eviction policy that matches workload (allkeys-lru > volatile-lru when app may overlook TTLs).
- Capacity alarms with actionable runbooks (scale-up vs flush strategy).
- Periodic keyspace review – INFO keyspace & MEMORY USAGE sampling to detect fat keys.

30-Second Interview Summary
“Our Redis-backed ElastiCache started throwing maxmemory errors because new keys were written without TTLs. With the default noeviction policy the cluster ran out of RAM, causing write failures and replica churn. I mitigated it by switching the parameter group to allkeys-lru, adding reserved-memory headroom, and bulk-deleting the heaviest cold keys. After the fire-fight I worked with developers to enforce TTLs in code and set CloudWatch alarms to prevent recurrence.”






     


