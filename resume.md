
# 1- Developed an Ansible role to configure monitoring across 73 servers, significantly enhancing infrastructure observability and reducing manual effort.

1. What was the purpose of the Ansible role you developed?
> I wrote the role to automate observability across all production hosts. Its immediate goal was to deploy Node Exporter to every running instance .  The most critical metric we needed right away was disk-space utilisation (several incidents had been caused by silent volume fill-ups),but the exporter also gave us CPU, memory, and networking insight.
By codifying this in an Ansible role we eliminated repetitive, manual installs and ensured new servers were monitored automatically via our dynamic inventory.”

2. What monitoring tool(s) did your role configure?
> Node Exporter on all Ubuntu application and database nodes (73 servers). Prometheus (scrape-target configuration + systemd service) and Grafana (provisioning of data-source and default dashboards)

3. How did you ensure the role was idempotent?
> A: by using idempotent modules like lineinfile , copy , file , apt , get_url module can be made idempotent 

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
```
| Module                      | Why it was used                                                  |
| --------------------------- | ---------------------------------------------------------------- |
| `apt`                       | Install Node Exporter, Prometheus, Grafana (via official repos). |
| `unarchive` / `get_url`     | Download & unpack Node Exporter tarball when repo not desired.   |
| `template`                  | Render `prometheus.yml`, Grafana data-source, and dashboards.    |
| `lineinfile`                | Append scrape configs for edge cases without re-templating file. |
| `service` / `systemd`       | Enable & start services; control restarts through handlers.      |
| `copy`                      | Ship pre-built Grafana dashboards JSON into `/var/lib/grafana/`. |
| `command` (with `creates:`) | Reload Grafana provisioning API post-copy.                       |
```

6. Did you handle OS differences (Ubuntu vs RHEL)? How?
```
In this environment, all 73 nodes were running Ubuntu. However, the setup can be made flexible by using Ansible's gather_facts feature along "with" conditional logic based on the ansible_os_family variable.
```

7. Any challenge you faced ? 
```
One of the main challenges I faced while setting up monitoring was the diversity in Ubuntu versions across our infrastructure — we had servers ranging from Ubuntu 14.04 up to 22.04. This created compatibility issues, especially with system dependencies and service management (like systemd vs init.d). I had to carefully choose versions of Node Exporter, Prometheus, and Grafana that would work reliably across all these environments. In some cases, I used static binaries for Node Exporter to avoid package conflicts, and made sure the Ansible role handled service registration appropriately depending on the OS version.”
OR 
One challenge I faced in setting up monitoring was handling multiple Ubuntu versions (14.04 to 22.04). To ensure compatibility, I used static Node Exporter binaries and tailored the Ansible role for service management based on the OS version.
```

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

🔹 A. Node Exporter (Installed on Each Monitored Server)
-----------------------------------------------------------
Purpose: Exposes hardware and OS-level metrics as HTTP endpoints (in Prometheus format). Examples of metrics:
> CPU usage   > Memory   > Disk I/O   > Network  > Filesystem usage
> Listens on: :9100 by default (e.g., http://server:9100/metrics)
> Lightweight & stateless: No config, no DB, just exposes /metrics.

🔹 B. Prometheus Server (Usually on Bastion or Monitoring Node)
---------------------------------------------------------------
* Purpose:
    > Pulls metrics from each Node Exporter every N seconds.
    > Stores all data as time-series in its own internal database (TSDB).
    > Evaluates alert rules (e.g., disk > 90% → alert).
* Config file: prometheus.yml defines scrape targets and intervals.
* Pull-based model: Prometheus reaches out to Node Exporters (not the other way).
* Also exposes metrics for itself (monitor Prometheus via Prometheus!).

🔹 C. Grafana (Visualization Layer)
------------------------------------
* Purpose:
    > Connects to Prometheus as a data source.
    > Lets users create interactive dashboards and custom visualizations.
    > Sends alerts via email, Slack, PagerDuty, etc.
* Runs as a web UI: default on http://localhost:3000
* Provisioning: Can load dashboards and data sources via JSON or YAML templates.

🔁 How They Work Together
---------------------------
- Node Exporter runs on each server → exposes metrics at /metrics.
- Prometheus scrapes these metrics periodically and stores them.
- Grafana reads from Prometheus to render beautiful graphs and dashboards.

```

9. How you would configure security groups (SGs) in a cloud environment

```
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
------------------
- Avoid 0.0.0.0/0 on port 9100 — limit it to only Prometheus host.
- Use Security Group references instead of raw IPs (e.g., "allow Bastion SG to access App SG on 9100").
- Restrict access to Grafana UI (3000) and Prometheus UI (9090) to VPN or specific IPs.
- Enable basic auth / reverse proxy / NGINX + TLS if exposing UIs.

🧱 Diagram Summary
--------------------

[Grafana:3000] <--- Your IP / VPN
[Prometheus:9090] <--- Your IP / VPN
   |
   └─> [Node Exporter:9100] on each App/DB Server


```

10. Explain full workflow of set up that was done ?

```
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

```

11. Relabelling concept and how it is used in our set up ?
```
🧠 What is Relabeling in Prometheus?
---------------------------------------
Relabeling is a way to transform metadata before it’s stored or scraped. Think of it as a filter or editor for labels — you can:
  - Add, remove, rename, or modify labels
  - Control which targets get scraped or how they're named
  - Organize or enrich metrics using instance info (like name, IP, ID)
  - Relabeling happens at different stages:
          * Target relabeling: Before scraping, to modify the target's address or labels
          * Metric relabeling: After scraping, to modify labels on metrics themselves (less common)


🔍 Your Setup: Where Relabeling Happens
----------------------------------------
Your config:
----------------

- job_name: 'ec2'    -----
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


Let’s explain this step-by-step
--------------------------------
1- job_name: 'ec2' 
 This is the name of the job shown in Prometheus UI (/targets). You're scraping metrics from EC2 instances.

2- ec2_sd_configs:
    - region: us-east-1
      filters:
        - name: "instance-state-name"
          values: ["running"]

 This tells Prometheus to:
    - Look for EC2 instances in us-east-1
    - Only consider running instances (skip stopped or terminated)

3-   relabel_configs:
    - source_labels: [__meta_ec2_private_ip]
      target_label: __address__
      replacement: "$1:9100"

- This sets the scrape target IP and port: 
   * Uses each instance's private IP
   * Appends :9100, assuming Node Exporter runs on port 9100

4- - source_labels: [__meta_ec2_tag_Name]
     target_label: server_name

This adds a label "server_name" to the target, based on the "EC2 tag Name"
✅ Example:
-  If the EC2 instance has a tag like: "Name": "web-server-prod"
- Then Prometheus will attach: server_name="web-server-prod"

5-  - source_labels: [__meta_ec2_instance_id]
      target_label: instance_id

Adds a label called instance_id to each target — useful to identify machines in Grafana or alerting.

✅ Real-Life Example Summary:
-----------------------------
we used this config so Prometheus could automatically find all EC2 instances in us-east-1, scrape metrics from them (via port 9100), and attach useful labels like:
 - Server name (from tags)
 - Instance ID
This helped us monitor 72+ EC2 instances without manually listing each one.


--------------------------------------------------------------------------------------

🖼️ In Grafana (as shown in your screenshot)
Thanks to relabeling: You have dropdown filters like Server and Job in Grafana. These are populated using labels like server_name, which came from your EC2 instance tags via relabeling

--------------------------------------------------------------------------------------

🎯 Summary (For Interviews)
----------------------------
"In our Prometheus setup, we used ec2_sd_configs to dynamically discover "running EC2 instances". Then we applied "relabeling" to:
  - Set the correct scrape target using the instance’s private IP and port (Node Exporter on 9100)
  - Extract the "EC2 Name tag" into a "custom label server_name"
  - Add "instance_id" as a label for traceability.
These relabeled values were used in Grafana to build dashboards with server dropdowns and targeted alert rules."
```
--------------------------------------------------------------------------------------------------------------

12. 📊 Variables in grafana dashbord 
--------------------------------------

You're using **Grafana + Prometheus** to monitor **EC2 servers** (like CPU, memory, etc.).
You want your dashboard to let users **choose a server from a dropdown** — and then see graphs **just for that one**.

```

🔧 How do we do this?
---------------------
* We use variables in Grafana — like `$job`, `$node`, `$ec2name`.
* These act like filters or dropdowns in your dashboard.

🧪 Example query:
------------------
rate(node_cpu_seconds_total{job="$job", nodename="$node", mode="user"}[5m])
    
* This query shows metrics for one EC2 server.
* `$job` → lets user choose what kind of job (like `node_exporter`)
* `$node` → lets user pick **which EC2 server** (by hostname or IP)

So user selects from dropdowns:
* Job → node\_exporter
* Node → ip-172-31-0-12

And Grafana shows metrics only for that EC2.

📎 What about EC2 Name tag?
------------------------------
EC2 machines usually have a **Name** tag in AWS (like `web-prod-1`, `db-test-2`).
Prometheus can fetch this using config:

- source_labels: [__meta_ec2_tag_Name]
  target_label: server_name

This adds a label called `server_name` to each EC2 in Prometheus.

✅ How to show EC2 Name in Grafana
---------------------------------------
* You can create a new dropdown variable in Grafana:

* Name: `ec2name`
* Query:   label_values(up{job="$job"}, server_name)
  
Now the user sees this dropdown:
* Choose EC2 Name: [web-prod-1] [api-node] [db-prod]
* And Grafana can use `$ec2name` to filter graphs.

🧠 In Simple Words:
-------------------
* You create dropdowns using **Grafana variables**
* You connect those to **labels in Prometheus**
* Prometheus knows about EC2s and their tags like `server_name`
* You use `$job`, `$node`, or `$ec2name` in queries to show only **what the user selects**
```

====================================================================================================

# 2- Migrated S3 buckets from North Virginia to Mumbai region, ensuring data availability and compliance.
```
I chose aws s3 sync because it’s simple, zero-cost, and quick to run using familiar CLI tools. While DataSync does offer extra features (ACL preservation, built-in validation, scheduling), setting it up—creating agents, permissions, IAM roles—introduced delays. For our use case, the benefits didn’t outweigh the setup overhead.
```
======================================================================================================

# 3. Addressed diverse tasks, including resolving upstream issues, clearing server storage, fixing Jenkins pipeline errors, scaling servers, and mitigating 2-3 critical downtimes to ensure uninterrupted operations

## Nginx & upstream issue 

In our production stack, traffic first hits an AWS ALB front door, then an NGINX-based API-gateway tier that forwards to a second layer of ALBs (booking, payments, etc.). Because Amazon load-balancers publish ephemeral IPs that can change at any time , NGINX occasionally cached an outdated address for an upstream and started returning “110: no host found” errors. 

A) Root Cause — Stale DNS in NGINX
------------------------------------
| Symptom                                              | Technical Explanation                                                                                                                                                                                                                                                |
| ---------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Intermittent 502/504 with `upstream timed out (110)` | Open-source NGINX **resolves the upstream hostname only once at worker start-up** and stores the resulting IP in memory([serverfault.com][1], [stackoverflow.com][2]).                                                                                               |
| Why it broke                                         | Each internal ALB is addressed by a DNS name whose A-records can rotate at any moment for scaling or AZ failover([stackoverflow.com][3], [repost.aws][4]). When an ALB’s IP changed, NGINX kept sending traffic to the now-stale IP, triggering connection timeouts. |

[1]: https://serverfault.com/questions/240476/how-to-force-nginx-to-resolve-dns-of-a-dynamic-hostname-everytime-when-doing-p?utm_source=chatgpt.com "How to force nginx to resolve DNS (of a dynamic hostname ..."
[2]: https://stackoverflow.com/questions/26956979/error-with-ip-and-nginx-as-reverse-proxy?utm_source=chatgpt.com "Error with IP and Nginx as reverse proxy - Stack Overflow"
[3]: https://stackoverflow.com/questions/3821333/does-amazon-ec2-elastic-load-balancers-ip-ever-change?utm_source=chatgpt.com "Does Amazon EC2 Elastic Load Balancer's IP ever Change?"
[4]: https://repost.aws/questions/QUyjryn7t7SOOhYQDtpfR2pg/application-load-balancer-ip-change-event?utm_source=chatgpt.com "Application Load Balancer IP Change Event | AWS re:Post"


B) Immediate Mitigation
------------------------------------
- Rebuilt NGINX with the dynamic upstream resolve patch (a third-party module that re-queries DNS on every health-check) to stop hard-caching addresses.
- Added a resolver directive pointing to AmazonProvidedDNS so that DNS lookups respect TTLs
Impact: Errors stopped, but the custom build created a maintenance burden and exposed an unrelated bug in that patch version.

C) Permanent Fix — Upgrade to NGINX 1.27.3
------------------------------------------
> From OSS 1.27.0 onward, NGINX native upstreams support the server <hostname> resolve; flag inside upstream blocks, dynamically re-resolving DNS without third-party modules. So I upgraded to NGINX 1.27.3 (the first stable build after the patch), removed the custom module, and set
```
upstream booking_backend {
    zone booking 64k;
    server booking-alb.internal resolve;
}

 resolver 169.254.169.253 valid=10s;  # AmazonProvidedDNS
```
> Each worker now refreshes the ALB’s IP list every 10 s (TTL-bound)

D) How to Phrase It in the Interview ? 
* Our API gateway (open-source NGINX) cached the IPs of internal ALBs. When AWS rotated those IPs, NGINX threw 110: no host found. I first hot-fixed the issue by recompiling NGINX with a dynamic-DNS module, then adopted the native server … resolve; feature in NGINX 1.27.3. That upgrade removed the custom patch, respected DNS TTLs, and permanently stopped upstream-resolution outages.

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

# Incident Management (Downtime)
----------------------------------

#### Elasticsearch Cluster Instability
--------------------------------------------

One of the major downtimes I resolved involved a persistent restart issue in our Elasticsearch cluster (v6.8). The cluster was unstable and kept flapping — nodes were disconnecting and reconnecting repeatedly, and indexing stopped. Upon analyzing the logs, I found multiple underlying root causes.”

#### 🔍 Root Cause 1: Split-Brain Scenario
--------------------------------------------
- Elasticsearch allows any master-eligible node to be elected as master.
- Our discovery.zen.minimum_master_nodes was not configured, which led to split-brain — multiple nodes thought they were master.
- As per official guidance, it should be set to: (number of master-eligible nodes / 2) + 1
- In our case, we had 3 master-eligible nodes, so the value should’ve been: ```discovery.zen.minimum_master_nodes: 2```
- Once configured and rolled out consistently, master election stabilized.

#### 🔍 Root Cause 2: Discovery Configuration Inconsistency
----------------------------------------------------------
- The parameter discovery.zen.ping.unicast.hosts was not consistently set across nodes.
- Some nodes had missing or incorrect IPs of peer nodes.
- This prevented proper cluster formation and caused intermittent network partitioning.
- I corrected this by ensuring all master/data nodes had the same list of IPs:
```
discovery.zen.ping.unicast.hosts: ["172.30.6.121:9300","172.30.6.119:9300","172.30.6.22:9300","172.30.6.245:9300","172.30.6.169:9300"]
```

#### 🔥 Secondary Failure: OOM Crashes
------------------------------------------
- After the cluster stabilized, we faced another issue: OutOfMemoryErrors (OOM).
- The heap was crashing under load, and I noticed:
```
-Xms8g
-Xmx8g
```
Which was too low for our workload. According to Elastic best practices:
- Heap size should be 50% of total RAM, but not more than 32GB. so I adjusted:

#### ✅ Interview Summary Line
> “I resolved a major production downtime involving Elasticsearch 6.8. The root cause was a split-brain issue due to missing minimum_master_nodes and inconsistent unicast.hosts. Once fixed, the cluster stabilized — but later hit OOMs, which I solved by increasing the Java heap size to 32GB. I also added alerting and documented the fix for future recovery.”

---------------------------------------------------------------------------------------------------------------------------------------------------

# Amazon ElastiCache for Redis
----------------------------

### 🛑 Situation
| Item                | Detail                                                                                                      |
| ------------------- | ----------------------------------------------------------------------------------------------------------- |
| **Service**         | ElastiCache ( Redis cluster, single shard, in-memory store for sessions & API responses )                   |
| **Symptom**         | *Primary node went “In‐Memory OOM”* → replica promotion loops → application latency spikes → partial outage |
| **Immediate Cause** | Redis reported `ERR maxmemory limit reached` every few seconds; INFO command showed *memory almost 100 %*.  |

### 🔎 Root-Cause Analysis
```
1. No TTLs on many keys: The application team had recently introduced several new caches, but forgot to set EXPIRE. Keys piled up indefinitely; dataset size grew ~4 × in two days.

2. Default eviction policy = noeviction (ElastiCache default). When memory was full, Redis refused writes instead of freeing space, triggering errors and failovers.

3. Metrics Evidence:
   - DatabaseMemoryUsagePercentage raced from 65 % → 98 %.
   - CurrItems kept increasing; EvictedKeys remained 0.
```

### 🛠️ Actions Taken
| Step                                | Command / Setting                                                 | Purpose                                                                       |                                                        |
| ----------------------------------- | ----------------------------------------------------------------- | ----------------------------------------------------------------------------- | ------------------------------------------------------ |
| **1. Enabled automatic eviction**   | `maxmemory-policy allkeys-lru`                                    | Allow Redis to drop the **Least-Recently-Used** key when memory hits the cap. |                                                        |
| **2. Added 20 % headroom**          | `reserved-memory 20` (ElastiCache parameter)                      | Avoid edge-case crashes during rehashing and failover.                        |                                                        |
| **3. Kicked off immediate cleanup** | \`redis-cli --scan                                                | xargs redis-cli DEL\` (targeted large unused keys)                            | Lower memory pressure while policy change took effect. |
| **4. Introduced key TTL standards** | Updated application code → `SET key value EX 1800`                | Ensure new writes disappear automatically after 30 min.                       |                                                        |
| **5. Monitoring & Alerts**          | CloudWatch Alarm: `DatabaseMemoryUsagePercentage > 80% for 5 min` | Proactive notification before next incident.                                  |                                                        |

### ✅ Result
- Outage contained in ~15 minutes; writes resumed once LRU evicted cold data.
- Long-term dataset stabilized at ~55 % utilisation.
- No repeat OOMs in the following 6 months.

### 📖 Lessons & Preventive Controls
- Always pair SET with TTL – enforced via a helper wrapper in the codebase.
- Select an eviction policy that matches workload (allkeys-lru > volatile-lru when app may overlook TTLs).
- Capacity alarms with actionable runbooks (scale-up vs flush strategy).
- Periodic keyspace review – INFO keyspace & MEMORY USAGE sampling to detect fat keys.

### 30-Second Interview Summary
“Our Redis-backed ElastiCache started throwing maxmemory errors because new keys were written without TTLs. With the default noeviction policy the cluster ran out of RAM, causing write failures and replica churn. I mitigated it by switching the parameter group to allkeys-lru, adding reserved-memory headroom, and bulk-deleting the heaviest cold keys. After the fire-fight I worked with developers to enforce TTLs in code and set CloudWatch alarms to prevent recurrence.”


====================================================================================================

# VPN 

### ✅ 1. What is a VPN and why is it used in DevOps or cloud setups?
- A VPN (Virtual Private Network) is a secure, encrypted connection over the internet between a user/device and a private network. It creates a “tunnel” that hides data from outside parties.

- 🔧 In DevOps and cloud environments, VPN is used to:
  * Secure remote access to private infrastructure (e.g., EC2 instances, databases) not exposed to the public internet.
  * Connect developers or ops teams to a private VPC/network.
  * Allow cross-site or cross-region communication securely (e.g., between on-prem and AWS).
  * Protect CI/CD pipelines when tools (like Jenkins) need to connect securely to remote environments.
  

### ✅ 2. What are the differences between site-to-site VPN and client-to-site VPN?
| Feature                 | Site-to-Site VPN                                    | Client-to-Site VPN                                 |
| ----------------------- | --------------------------------------------------- | -------------------------------------------------- |
| 🔗 **Connection Type**  | Connects **two networks** (e.g., AWS VPC ↔ On-prem) | Connects **individual users/devices** to a network |
| 👤 **User Access**      | Not user-specific; traffic routed automatically     | User-specific; requires authentication             |
| 🛠️ **Use Case**        | Permanent hybrid cloud setup, office ↔ cloud        | Dev/Engineer connecting securely to VPC            |
| 🔐 **Security**         | IPsec tunnels, usually set up on gateways           | Often uses OpenVPN, WireGuard, etc.                |
| 💻 **Client Software?** | No (router-to-router)                               | Yes (user needs VPN client installed)              |


---

### ✅ **1. What is StrongSwan, and why did you use it over cloud-managed VPNs?**

**StrongSwan** is an open-source IPsec-based VPN solution that supports IKEv1 and IKEv2 protocols. I chose it because:

* It’s cost-effective (no extra charges compared to cloud-managed VPNs).
* Offers **more customization and control**.
* Works across cloud platforms and even on-prem.
* Managed VPNs (e.g., AWS VPN Gateway) were either unavailable or more complex/costly to configure across providers.

---

### ✅ **2. What is a site-to-site VPN? How is it different from a client-to-site VPN?**

* A **site-to-site VPN** connects two networks (e.g., AWS VPC and Azure VNet), allowing internal communication across them over an encrypted tunnel.
* A **client-to-site VPN** connects a single user (like your laptop) to a remote network.
* I implemented **site-to-site** to allow instances in AWS and Azure to communicate **over private IPs** securely.

---

### ✅ **3. What is IPsec and how does it work in StrongSwan?**

**IPsec (Internet Protocol Security)** provides secure communication over IP networks. It does:

* **Encrypt** and **authenticate** packets.
* Establishes a **tunnel** using protocols like **IKE (Internet Key Exchange)**.

In StrongSwan:

* I used **IKEv2** to negotiate the security parameters.
* Traffic between the two sites is **encapsulated and encrypted**, using the configured PSK and algorithms like `aes256-sha2_256`.

---

### ✅ **4. Can you walk me through the steps you followed to set up the VPN?**

Yes, here’s the workflow:

#### On **AWS Side**:

* Created a VPC, subnet, internet gateway, and an EC2 instance.
* Installed StrongSwan and configured:

  * `/etc/ipsec.conf` for connection details.
  * `/etc/ipsec.secrets` with the PSK.
* Enabled IP forwarding and disabled source/destination checks on the instance.
* Added a route in the VPC route table to forward traffic for Azure subnet via the EC2 instance. 
     - in route table in **destination field** entered the cidr of azure vnet . This CIDR blocks specifies the range if IP addresses assigned to azure vnet 
     - in **target field** select **instance** and then choose the instance ID of the server where strongswan is installed. This indicates that specified destination cidr block should be reachable via the selected ec2 instance  
   > Whenever someone in my VPC wants to talk to something in Azure (like 10.1.x.x), send that traffic through my VPN EC2 instance — it’ll handle the connection to Azure

#### On **Azure Side**:

* Created a Resource Group, VNet, subnet, and VM.
* Installed StrongSwan with mirrored configuration.
* Enabled IP forwarding on VM and through Azure Network Interface settings.
* Created a route table to send AWS-bound traffic via the Azure StrongSwan VM.

Finally, restarted StrongSwan with `ipsec restart`, and verified tunnel status and connectivity with `ipsec status` and `ping` between private IPs.

---

### ✅ **5. What were the VPC CIDRs used in AWS and Azure? Why should they not overlap?**

* AWS VPC CIDR: `172.0.0.0/16`
* Azure VNet CIDR: `10.0.0.0/16`

**They must not overlap** because:

* Routing and NAT won’t work correctly if both networks have the same address range.
* Overlapping CIDRs create ambiguity in packet delivery.

---

### ✅ **6. Why did you need to disable source/destination checks in AWS?**

AWS EC2 instances by default drop traffic not meant for them. But my EC2 instance (running StrongSwan) was **routing** packets between AWS and Azure.

* Disabling source/destination checks lets it forward traffic.
* It was **critical for enabling transit of VPN-encrypted packets**.

OR 

you need to disable source/destination checks in AWS because your EC2 instance was acting like a router, not just a regular server.

🧠 Simplified Explanation:
- By default, every EC2 instance in AWS is designed to only accept and respond to traffic that is specifically meant for it (its own IP).
- But in your case:
      * You set up a VPN tunnel between AWS and Azure using StrongSwan.
      * The EC2 instance wasn’t just receiving traffic for itself — it was receiving traffic meant for Azure and forwarding it across the VPN.
      * This behavior is not allowed by default, so you had to disable the "source/destination check".

---

### ✅ **7. What is the role of IP forwarding in this setup?**

Enabling **`net.ipv4.ip_forward=1`** allows the Linux VM to **forward traffic between interfaces**, essentially turning it into a router.

* Without it, packets would be received but not forwarded.
* It is required both in AWS and Azure VMs for the tunnel to work.

OR 

It allows the Linux machine to forward packets from one network to another — just like a router.
🧠 Simplified Explanation:
* Your StrongSwan instance has:
   - One interface connected to the AWS side
   - One encrypted tunnel connected to the Azure side
   - But by default, Linux only handles traffic for itself.
   - So to allow Linux to pass packets from AWS to Azure and back, you need to enable: ```net.ipv4.ip_forward = 1```
   - This tells the Linux kernel: “I’m not just a machine. I’m acting like a router — please forward packets between interfaces.”

---

### ✅ **8. Where did you configure the PSK, and how is it secured?**

* Configured in `/etc/ipsec.secrets` on both VMs.
* Example:
  `<public IP AWS> <public IP Azure> : PSK "base64generatedkey"`

**Security considerations**:

* Stored with strict file permissions (readable by root only).
* Key was generated using `openssl rand -base64 64`.

---

### ✅ **9. What are the key configuration files for StrongSwan? What entries did you make there?**

**Files:**

1. `/etc/ipsec.conf`: Contains connection definitions.
2. `/etc/ipsec.secrets`: Holds authentication secrets (PSK).

**Sample ipsec.conf entry**:

```conf
conn siteA-to-siteB
 authby=secret
 left=<public IP A>
 leftsubnet=<private subnet A>
 right=<public IP B>
 rightsubnet=<private subnet B>
 auto=start
 ike=aes256-sha2_256-modp1024!
 esp=aes256-sha2_256!
```

---

### ✅ **10. What does the `conn` block represent in the ipsec.conf file?**

It defines a **connection profile**, specifying:

* **Peers** (left and right IPs),
* **Subnets** to be connected,
* **Protocols/algorithms** for encryption,
* **Tunnel lifetime**, retry policies, etc.

The `auto=start` ensures the connection is established on service start.

---

### ✅ **11. Which protocol version did you use — IKEv1 or IKEv2? Why?**

I used **IKEv2** because:

* It’s more secure and efficient than IKEv1.
* Supports **mobility**, **NAT traversal**, and **faster rekeying**.
* StrongSwan supports both, but IKEv2 is recommended unless the peer only supports IKEv1.

---

### ✅ **12. How did you configure routing in AWS and Azure to allow cross-cloud communication?**

On **AWS**:

* Added a route in the public route table:

  * **Destination**: Azure subnet CIDR (`10.0.0.0/16`)
  * **Target**: StrongSwan EC2 instance

On **Azure**:

* Created a **User Defined Route (UDR)**:

  * **Destination**: AWS subnet CIDR (`172.0.0.0/16`)
  * **Next Hop**: Private IP of Azure StrongSwan VM
* Associated the UDR with the subnet.

---

### ✅ **13. What issues did you face when the tunnel came up but traffic didn’t pass? How did you debug it?**

* Tunnel showed as **established**, but no traffic passed.
* Root cause: IP forwarding was disabled, and routing was missing.
* Debug steps:

  * Verified with `ipsec statusall`
  * Checked `tcpdump` for ESP traffic
  * Checked system logs at `/var/log/syslog` for StrongSwan messages
  * Ensured **firewall rules allowed UDP 500, 4500 and ESP protocol (50)**
  * Fixed routes and enabled IP forwarding

---

### ✅ **14. How do you test if VPN is working? Can you explain with a command?**

After establishing the tunnel, I launched test VMs in each network and ran:

```bash
ping <private IP of Azure VM>   # from AWS
ping <private IP of AWS VM>     # from Azure
```

Also verified:

```bash
ipsec statusall
```

This confirms tunnel status and active Security Associations (SAs).

---

### ✅ **15. How did you ensure that your VPN setup is secure?**

* Used **AES-256 encryption** with SHA2 for hashing.
* Random 512-bit PSK generated using OpenSSL.
* Firewalls were restricted to **only necessary ports** (UDP 500, 4500, and ESP).
* VPC/Subnet isolation was maintained.
* Configuration files were protected using appropriate Linux permissions.

---

### ✅ **16. How can you monitor if the VPN tunnel goes down or fails to connect?**

* Run: `ipsec status` or `ipsec listall`
* Use `systemctl status strongswan` for service health
* Enable **charondebug** in `ipsec.conf` for detailed logs
* Monitor `/var/log/syslog` for reconnection attempts or failures

---

### ✅ **17. Did you face any challenges during the setup? How did you solve them?**
Yes:
* **Tunnel established but no traffic passed**: Resolved by enabling IP forwarding and fixing routing tables.
* **StrongSwan not starting**: Due to misformatted `ipsec.conf`.
* **Firewall blocks**: Opened UDP 500/4500 and protocol 50 on security groups and NSGs.
* **CIDR overlap confusion**: Double-checked with `ipcalc` before setup.

---

### ✅ **18. If given a similar task again, what would you do differently?**

* Use a **Terraform module** to automate the network/VPN setup.
* Add **monitoring and alerting** using cron + `ipsec status` checks.
* Keep a checklist of network and routing configuration to avoid manual errors.

---

### ✅ **19. Have you used this VPN for anything practical (e.g., data sync, DB connection)?**

Yes:

* It was used to **enable communication between services in AWS and Azure** for internal APIs.
* We tested **private database replication** and **SSH access** across clouds via private IPs.

---

### ✅ **20. What happens when you are start vpn?**
1- You run: ```sudo ipsec up vpn-connection-name```

2- StrongSwan:
  - Initiates IKEv2 with the remote peer (e.g., Azure VPN gateway)
  - Authenticates both sides (e.g., with a pre-shared key)
  - Negotiates encryption settings (AES, SHA, DH group, etc.)
  - Establishes Security Associations (SAs)
  - Passes SAs to kernel via ip xfrm interface

3- Kernel:
  - Encrypts packets leaving the system
  - Decrypts packets coming in
  - Forwards packets to/from the tunnel like a router


# ======================================================================================
     
# Set up Elastic APM for monitoring Node.js and Flask applications, enabling real-time performance tracking and optimizing application efficiency

















# ======================================================================================

# Created Ansible roles for configuring a PostgreSQL cluster with automatic failover using Patroni and  HAProxy, ensuring high availability and reliability of database services.
# Implemented monitoring solutions for PostgreSQL databases to track performance metrics and detect issues proactively

```
                     ┌──────────────────────────┐
                     │      Client Apps         │
                     │ psql / Python / Grafana  │
                     └──────────┬───────────────┘
                                │  (TCP 5000 SQL)
                      Read/Write│
                                ▼
                        ┌─────────────┐
                        │   HAProxy   │
                        │ 5000 / 7000 │
                        └─────┬───────┘
          HTTP health (8008)  │
        ──────────────────────┼─────────────────────
        │                     │                     │
┌───────▼───────┐     ┌───────▼───────┐     ┌───────▼───────┐
│  Patroni/PG   │     │  Patroni/PG   │     │  Patroni/PG   │
│   node1       │     │   node2       │     │   node3       │
│ 5432 & 8008   │     │ 5432 & 8008   │     │ 5432 & 8008   │
└───┬─────┬─────┘     └───┬─────┬─────┘     └───┬─────┬─────┘
    │     │               │     │               │     │
    │ etcd watch (2379)   │     │ etcd watch    │     │
    └─────────┬───────────┘     └───────────────┘     │
              │                                       │
              ▼                                       ▼
        ┌──────────────┐                       ┌──────────────┐
        │     etcd     │  (cluster state)      │ Prometheus   │
        │   2379/2380  │                       │ + Grafana    │
        └──────────────┘                       └──────────────┘
```

| Legend          | Meaning                                                         |
| --------------- | --------------------------------------------------------------- |
| **Solid line**  | SQL traffic (clients ↔ leader)                                  |
| **Dashed line** | Control-plane traffic (health checks, leader-election, metrics) |

```
INTERVIEW ANSWER
------------------
I set up a highly available PostgreSQL cluster using Patroni, HAProxy, and etcd, automated through Ansible roles.
The architecture had three PostgreSQL nodes managed by Patroni, with etcd for leader election and HAProxy as a load balancer.

Applications connected to HAProxy on port 5000, which always forwarded SQL traffic to the current leader node.
Patroni exposed health endpoints on each node, and HAProxy used these to detect which node was the primary.
Failover was automatic — if the leader crashed, Patroni would detect it via etcd, trigger a new election, and HAProxy would route traffic to the new primary.

I also configured settings like use_pg_rewind so that old leaders could rejoin the cluster quickly after failover, without a full resync.
This setup ensured minimal downtime and automatic failover, without needing manual intervention or affecting the app layer.


```
1️⃣ Client side
-----------------
• Who?
> psql, ORM libraries, BI tools, Grafana dashboards—anything that speaks PostgreSQL.

• What do they do?
> They open a single connection to haproxy:5000. They never need to know which node is currently primary.

2️⃣ HAProxy (Layer-4 TCP load balancer)
---------------------------------------
| Port     | Purpose                                                                         |
| -------- | ------------------------------------------------------------------------------- |
| **5000** | Front-end for SQL; forwards to whichever backend node reports “I’m the leader.” |
| **7000** | Optional stats UI for health visualization.                                     |

##### Health detection workflow
- Every 3 s (inter 3s) HAProxy calls ```http://<node>:8008/health``` (Patroni’s REST endpoint).
- Only the primary returns HTTP 200. Replicas reply 503.
- HAProxy marks one and only one backend UP; all traffic rides that connection.

3️⃣ Patroni-managed Postgres nodes
-------------------------------------

| Component        | Port     | Function                         |
| ---------------- | -------- | -------------------------------- |
| **PostgreSQL**   | **5432** | Regular SQL service.             |
| **Patroni REST** | **8008** | Health, switchover, reconfigure. |

#### What Patroni does every 10 s (loop_wait)
- Reads the cluster key in etcd.
- If it is the leader, it updates the TTL (ttl: 30 s).
- If TTL expires, replicas start a new election.

Replication
- Leader ships WAL to replicas.
- use_pg_rewind: true lets an old primary catch up fast after failover.

#### 4️⃣ etcd - the DCS (Distributed Config Store)
| Port     | Role                                                     |
| -------- | -------------------------------------------------------- |
| **2379** | Client API; Patroni watches and writes here.             |
| **2380** | Peer traffic (only relevant if you add more etcd nodes). |

Key paths
```
/db/postgres/   ← your scope + namespace
  ├─ leader          (contains leader node name + TTL)
  ├─ members/<uuid>  (status JSON for each instance)
  └─ config          (global settings)

```
If the leader crashes, TTL expires, key disappears → replicas race to become leader (Raft lock).

#### 5️⃣ Prometheus + Grafana (Observability)
 - postgres_exporter on every DB node scrapes localhost:5432 and exposes metrics on 9187 (default from your systemd unit).
 - Prometheus scrapes those endpoints, stores time-series.
 - Grafana visualizes WAL lag, TPS, replication delay, etc.

#### 🔄 End-to-End Flow Example (Normal Operation)
1- Insert Query ```INSERT INTO orders … ; ```
Client → HAProxy 5000 → Leader’s port 5432 (say, node1).

2- Commit finishes; leader flushes WAL to disk and streams to node2, node3.

3- Health Beat
Every 3 s HAProxy polls /health on all nodes.
node1 returns 200 (“role=master”), node2/3 return 503.

4- Metrics
postgres_exporter on node1 updates pg_stat_activity; Prometheus scrapes; Grafana dashboard shows spike in TPS.

#### ⚠️ Flow During Failover
1- node1 crashes (power loss).
2- TTL in etcd (/db/postgres/leader) expires after 30 s.
3- node2 & node3 both see key gone → run leader-election.
The one with smallest timeline/WAL lag wins (say node2).
4- node2 writes new leader key; promotes itself.
5- HAProxy’s next health check hits /health on node2 → gets 200.
HAProxy flips backend UP to node2; client traffic now flows there.
6- node1 comes back → Patroni sees it’s no longer leader → runs pg_rewind → rejoins as replica.
Failover completes in ~5-10 s of client brown-out (depends on your timers).

####  Experience Talking Points
- Single-endpoint simplicity: “Apps only know haproxy:5000; we never re-deploy clients on failover.”
- Fast recovery: “Patroni + etcd TTL of 30 s, HAProxy health check every 3 s → worst-case switchover < ~10 s.”
- Safe catch-up: “Enabled use_pg_rewind and replication slots, so an old primary re-joins without full basebackup.”
- Observability first: “postgres_exporter feeds Prometheus; we alert on WAL lag > 60 s or HAProxy backend-down events.”
- Future hardening: “For prod we’d add a 3-node etcd quorum and run two HAProxy instances with keepalived VIP.”

-------------------------------------------------------------------------------------------------

## 🔄 What is WAL?

**WAL = Write-Ahead Log**

In PostgreSQL:

* Every **change** (INSERT, UPDATE, DELETE, CREATE TABLE, etc.) is first **written to WAL logs** before it's written to the actual data files.
* This ensures **data durability and crash recovery**.

> 🧠 Think of WAL like a **journal** of everything that changed in the database — used by both crash recovery and replication.

---

## 📤 How Do Replicas Sync via WAL?

In your setup (Postgres + Patroni), **replication is streaming-based**, and the replicas sync using **WAL streaming replication**.

### 🔁 WAL-Based Streaming Replication Workflow:

1. **Primary node** writes data changes to WAL (`pg_wal/` directory).
2. **Replicas** (standby nodes) **connect to the primary** using the `replicator` user (as defined in your `patroni.yml`).
3. Replicas **continuously stream WAL data** from the primary using a **WAL receiver process**.
4. The replicas **replay the WAL changes** on their own local copies of the database — making them exact copies.

---

## 🔐 What Type of Replication Is This?

| Type                     | Your Setup                                                                                                     |
| ------------------------ | -------------------------------------------------------------------------------------------------------------- |
| **Physical replication** | ✅ Yes                                                                                                          |
| Streaming mode           | ✅ Yes                                                                                                          |
| Synchronous              | ❌ Optional (you’re likely using **asynchronous** unless `synchronous_mode: true` is enabled in Patroni config) |

### 🔵 Physical Replication

* Replicates **raw WAL logs**, not logical rows.
* Faster, reliable, but the replica cannot be queried independently (read-only allowed though).

---

## 🧪 How It Syncs on New Replica Join (pg\_basebackup)

When a new node (say, `node2`) joins:

1. It does a **`pg_basebackup`** from the current leader → copies full data directory.
2. It connects to the leader as a **replica**, starts streaming WAL.
3. It catches up to the current LSN (Log Sequence Number) and begins **real-time replay**.

If a node goes down and rejoins:

* **Patroni uses `pg_rewind`** (if enabled) to sync WAL differences instead of redoing full copy.

---

## 🧠 Important Concepts

| Term                   | Meaning                                                         |
| ---------------------- | --------------------------------------------------------------- |
| `pg_wal/`              | Directory where WAL logs are stored                             |
| `wal_level = replica`  | Enables replication; already set by Patroni                     |
| `hot_standby = on`     | Allows read queries on standby                                  |
| `replication slot`     | Ensures WAL logs aren’t deleted until replica has consumed them |
| `use_pg_rewind = true` | Enables WAL delta catch-up instead of full reinitialization     |

---

## 💡 Example From Your Patroni Config

In `patroni-node1.yml.j2`:

```yaml
postgresql:
  authentication:
    replication:
      username: {{ replication_user }}
      password: {{ replica_password }}
```

In `pg_hba`:

```yaml
- host replication replicator {{ node2_ip }}/0 md5
```

This allows node2 to:

* Authenticate using the `replicator` user
* Connect to node1
* Begin streaming WAL → Replay changes → Become a replica

---

## ✅ Summary

> In your Patroni cluster, replicas stay in sync with the leader via **streaming physical WAL replication**. The leader writes all changes to WAL files, which are streamed in real time to the replicas. This ensures that all nodes have a consistent view of the database, enabling seamless failover and high availability.

---

Would you like to test WAL sync health using a query or a monitoring metric (like replication lag)?


