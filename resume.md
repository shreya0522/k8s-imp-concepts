
# 1- Developed an Ansible role to configure monitoring across 73 servers, significantly enhancing infrastructure observability and reducing manual effort.

1. What was the purpose of the Ansible role you developed?

```
I developed an idempotent Ansible role to deploy a complete monitoring stack — Node Exporter, Prometheus, and Grafana — across 73 Ubuntu servers.
The primary goal was to automate observability, starting with disk space metrics due to prior volume-related outages.
I deployed Node Exporter as a static binary to handle mixed OS versions (Ubuntu 14.04 to 22.04) and used conditional service setup based on ansible_service_mgr (systemd vs init.d).
Prometheus was configured with EC2 service discovery and relabeling to automatically discover instances and tag them with metadata like server_name and instance_id.
Grafana was provisioned using templates — including pre-built dashboards and Prometheus as a data source. We used Grafana variables ($server_name, $instance_id) to make dashboards fully dynamic and user-friendly.
For alerting, I set up contact points using Gmail SMTP and Google Chat for critical alerts, and linked them with Grafana alert rules defined via PromQL.
Finally, I designed cloud-native security groups to expose Node Exporter only to Prometheus, and locked down Grafana access to trusted IPs or VPN.
The setup significantly improved observability, enabled proactive alerting, and eliminated manual drift in monitoring configuration.

Optional Add-on if asked “Any challenge you solved?”
----------------------------------------------------
One challenge was inconsistent init systems across Ubuntu versions — some used systemd, others init.d. I solved this by shipping Node Exporter as a static binary and using Ansible’s fact-based conditionals to apply the correct service file dynamically.
```

> I wrote the role to automate observability across all production hosts. Its immediate goal was to deploy Node Exporter to every running instance .  
> The most critical metric we needed right away was disk-space utilisation (several incidents had been caused by silent volume fill-ups),but the exporter also gave us CPU, memory, and networking insight. 
> By codifying this in an Ansible role we eliminated repetitive, manual installs and ensured new servers were monitored automatically via our dynamic inventory.”

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
I keep defaults for same global values, vars for anything driven by facts or inventory, and a single restart handler that individual tasks notify only when their template or package changes.

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
The Challenge: Mixed Ubuntu Versions (14.04 → 22.04)
------------------------------------------------------
System differences: 
  - Older Ubuntu (14.04) uses the old init.d scripts
  - Newer Ubuntu (16.04+) uses systemd service units

Dependency drift: 
  - Package repositories on 14.04 only had very old Prometheus/Grafana
  - On 22.04 the distro’s version might be newer than what I wanted

Because of those gaps, I couldn’t rely on “just apt install node-exporter” everywhere, nor could I use the exact same service-file mechanism on every host.

My Solution:
-------------
Static Binaries + Conditional Service Setup
-----------------------------------------------

1- Ship a single, “just works” binary
  - I downloaded the Node Exporter tarball (e.g. node_exporter-1.8.1.linux-amd64.tar.gz) once
  - In my role’s files/ folder I dropped that archive
  - On every host—new or old—I simply un-tarred it under /opt/node_exporter/
  - No package manager, no missing dependencies

2- Detect the init system at runtime
  - Ansible automatically knows whether the host uses systemd or not via the built-in fact ansible_service_mgr

- name: Unpack Node Exporter
  unarchive:
    src: node_exporter-{{ node_exporter_version }}.linux-amd64.tar.gz
    dest: /opt/node_exporter
    remote_src: yes

- name: Install systemd unit for Node Exporter
  template:
    src: node_exporter.service.j2
    dest: /etc/systemd/system/node_exporter.service
  when: ansible_service_mgr == "systemd"
  notify: restart node_exporter

- name: Install init.d script for Node Exporter
  template:
    src: node_exporter.init.j2
    dest: /etc/init.d/node_exporter
    mode: '0755'
  when: ansible_service_mgr != "systemd"
  notify: restart node_exporter

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
Relabeling is Prometheus’s built-in way to take the "metadata it discovers about a scrape target" (things like IP addresses, AWS tags, file names, etc.) and turn or filter those into the exact labels you want on your metrics.

> Mapping raw metadata into nice labels
Your AWS instance has a tag called Name=web-server-prod. By relabeling:
source_labels: [__meta_ec2_tag_Name]
target_label: server_name
you copy that tag into a metric label called server_name="web-server-prod". That’s exactly turning a raw tag into a user-friendly name in your graphs and alerts.

But you can also…
- Rename any label (e.g. change job to service_name)
- Drop targets you don’t want to scrape (by matching or not matching a regex)
- Compose new labels by combining parts of other labels
- Overwrite the built-in __address__ so you scrape a different port

the common example is “take the EC2 Name tag and turn it into a nice server_name label” — but relabeling is really Prometheus’s flexible label editor/filter that runs before (and in metric relabeling, after) scraping.



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
1. Define your variables
----------------------------
In Grafana’s dashboard settings, you create variables like:
- $job: Pulls from the Prometheus job label (for example, “node_exporter”)
- $node: Pulls from the Prometheus instance or nodename label (for example, “ip-172-31-0-12”)
- $ec2name: Pulls from the custom server_name label you added via relabeling (for example, “web-prod-1”)

Each variable is set up with a Query type, using a Prometheus helper function:
> label_values(<metric>{job="$job"}, server_name)

This asks Prometheus “give me all the unique server_name values for targets in this job” and turns them into the dropdown options.

2. Hook your panels up to those variables
----------------------------------------------
When you write your Prometheus query in a Grafana panel, you include those variables in the {} selector. For example, to chart user-mode CPU usage on whichever server the user picks:
rate(node_cpu_seconds_total{
  job="$job",
  nodename="$node",
  server_name="$ec2name",
  mode="user"
}[5m])

Grafana replaces $job, $node, and $ec2name with the user’s choices.
The result is that the graph only shows metrics from that one EC2 instance.

3. Why this matters
-------------------
- No hard-coding: You don’t have to list every server by hand.
- Interactive: Users can switch between servers or jobs on the fly.
- Clean dashboards: Everything stays in one place, and all filtering is driven by labels that Prometheus already knows about.

Because you relabeled your EC2 targets with server_name in Prometheus, Grafana can present a friendly “EC2 Name” dropdown instead of forcing people to pick IP addresses, making your dashboard easy to navigate and use.


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
```


13. suppose if promeheus is down for few hours so it wont be able to scrape metrics so isnt there any remedy for that 
```

1. Buffer Locally with the Write-Ahead Log
--------------------------------------------
Prometheus first writes every scraped sample into an on-disk Write-Ahead Log (WAL) before committing it to TSDB blocks  
By default it retains at least three WAL segments—and on busy servers enough to cover about two hours of data—so if Prometheus crashes or is shut down briefly, it can replay that log on restart and recover the missed samples 
You can tune WAL behavior (e.g. --storage.tsdb.min-block-duration, --storage.tsdb.wal-compression) to hold more history if you expect longer outages.

Prometheus uses the Write-Ahead Log (WAL) to make sure no scraped metrics get lost if the server crashes or is restarted:
A- Every sample is first appended to a disk-based log:
As soon as Prometheus scrapes a metric, it writes it into the WAL—an append-only file—so the data is safely on disk before anything else happens 

B- These log files live in a wal/ folder in fixed-size segments:
By default, each segment is 128 MB and Prometheus keeps at least three of them around, giving you a few hours’ worth of buffered data 

C. On restart, Prometheus “replays” the WAL to recover missed samples:
When the server comes back up, it reads those log segments and reinserts any metrics that hadn’t yet been compacted into its TSDB blocks—so your short outage leaves no gaps 

D. This is fast and durable
Writing to an append-only log is much quicker and simpler than updating a full database, yet still guarantees your data is on stable storage before Prometheus acknowledges it 

E. You can tune how much WAL you keep
Flags like --storage.tsdb.min-block-duration and --storage.tsdb.wal-compression let you control how often Prometheus rolls segments into its main database and how many WAL files to retain.


NOTE:
-----
WAL only helps recover data that was scraped just before a crash, not compensate for time while Prometheus was off

Can Prometheus fetch missing metrics after it restarts?
--------------------------------------------------------
No. Prometheus is a "pull-based" system. If it missed a scrape interval, it cannot go back in time to fetch old metrics from exporters. Exporters don’t store history — they just expose current values.



2. Alerting 
-------------
uptime kuma >> if any issue then alert 
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
```
I handled a critical production incident where our Elasticsearch 6.8 cluster was repeatedly restarting due to a split-brain condition — minimum_master_nodes was misconfigured, leading to unstable master elections.
I resolved it by setting the correct quorum and standardizing unicast.hosts across all nodes to fix discovery inconsistencies.
Post-recovery, I addressed frequent OOM crashes by optimizing the JVM heap size to 32GB, aligned with Elastic's best practices.
I also implemented alerting for cluster state changes and documented the root cause to improve incident response for the future

------------------------------------------------------------------
INITIAL DISCOVERY : frequent restarts.

WHY : SPLIT BRAIN ISSUE 
        bcoz of misconfig 
        - min master node incorrect
        - unicast hots: ip of all hosts was not correctly set up 

split brain issue 
-----------------
Split-brain in Elasticsearch occurs when multiple nodes think they're the master, usually due to a misconfigured minimum_master_nodes.

A> minimum_master_nodes: According to Elastic's best practices, use this formula:
minimum_master_nodes = (number_of_master_eligible_nodes / 2) + 1
For a 3-node cluster: discovery.zen.minimum_master_nodes: 2 . This means: At least 2 out of 3 nodes must agree to elect a master. Prevents any one isolated node from becoming master on its own

B> Also ensure this: 
discovery.zen.ping.unicast.hosts:
  - "172.30.6.121"
  - "172.30.6.122"
  - "172.30.6.123"
✅ Why? It ensures that:
- Every node knows how to reach the others
- They can properly participate in election
- Prevents partial discovery and ghost masters

| Misconfiguration                  | Effect                                             |
| --------------------------------- | -------------------------------------------------- |
| `minimum_master_nodes` too low    | Split-brain → multiple masters, data corruption    |
| Missing `unicast.hosts` entries   | Nodes fail to discover each other, form partitions |
| Inconsistent configs across nodes | Flapping cluster, delayed elections, errors        |

------------------------------------------------------------------

```

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
IPSec is a protocol that encrypts and protects data while it travels over the internet between two networks or devices.

A secure tunnel that hides and protects your data from anyone who might try to read, change, or spy on it while it’s moving from Point A to Point B — like between AWS and Azure.

🛡️ What does IPSec do?
| Function           | In Plain English                          |
| ------------------ | ----------------------------------------- |
| **Encryption**     | Scrambles your data so no one can read it |
| **Authentication** | Confirms both sides are trusted           |
| **Integrity**      | Ensures data wasn't tampered with         |

🧱 IPSec is often used in:
- Site-to-site VPNs (like AWS ↔ Azure)
- Remote access VPNs
- Secure communication over public internet

Example:
Your EC2 in AWS sends data to Azure. Without IPSec, anyone on the internet could see or alter it.
With IPSec, the data is encrypted, authenticated, and safely delivered — like putting it in a locked armored truck instead of an open envelope.


---

### ✅ **4. Can you walk me through the steps you followed to set up the VPN?**
```
To set up the site-to-site VPN between AWS and Azure, I used StrongSwan as the IPsec VPN solution.

🔧 Steps on AWS Side:
---------------------
1- Provisioned a VPN Gateway EC2:
- Launched a Linux EC2 instance in a public subnet to act as the StrongSwan VPN gateway.

what do you mean by vpn g/w?
A VPC Gateway is not exactly a router, but it acts like one by enabling communication between your VPC and the outside world (internet or VPN).basically it encrypts and routes traffic between networks

2-Installed and Configured StrongSwan: 
- Installed StrongSwan (apt install strongswan) and configured /etc/ipsec.conf and /etc/ipsec.secrets with connection parameters.
- Used pre-shared key (PSK) authentication.

3- Enabled IP Forwarding:
- Updated /etc/sysctl.conf to enable IP forwarding (net.ipv4.ip_forward = 1) and applied it with sysctl -p.

4- Disabled Source/Destination Check:
- Disabled source/dest checks on the StrongSwan EC2 

5- Updated Route Tables:
Added a route in the private subnet’s route table to forward Azure-bound traffic (e.g., 10.1.0.0/16) via the StrongSwan instance’s ENI.

.

🔧 Mirrored the Same on Azure:
--------------------------------
- Provisioned a Linux VM in Azure to act as the VPN endpoint.
- Installed StrongSwan, configured matching IPsec settings (same PSK, subnets, etc.).
- Enabled IP forwarding and disabled reverse path filtering.
- Updated Azure route tables (UDRs) to send AWS-bound traffic via the VPN VM.

🔄 Traffic Flow Explanation:
----------------------------
🔹 AWS → Azure:
* What happens on the AWS side:
- An EC2 sends a packet destined for the Azure subnet (e.g., 10.1.0.0/16).
- The AWS VPC route table has a route saying:
- "Send packets to Azure CIDR via the StrongSwan EC2 instance."
- The EC2’s packet arrives at the StrongSwan instance in plain (unencrypted) form.
- StrongSwan encrypts the packet using IPsec.
- It then sends the encrypted packet through the IPsec tunnel to the Azure side.

On the Azure side:
- The StrongSwan VM on Azure receives the encrypted packet.
- It decrypts the packet using the shared IPsec configuration.
- Then it forwards the plain packet to the destination Azure VM (based on its local routing table).

```

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

```
main thing 
-----------

1. Disabling Source/Destination Checks
-----------------------------------------
By default, an EC2 instance will drop any packet that isn’t either to or from itself—this is the “source/destination check.” Disabling that check tells AWS:
- “Allow this instance to receive traffic not addressed to its own IP, and to send traffic not originating from its own IP.”
This is essential for any instance that must forward or NAT traffic, such as your VPN server.

2. Enabling IP Forwarding on the Instance
-------------------------------------------
Even with source/dest checks off, the Linux kernel itself won’t forward packets between network interfaces unless you turn it on. By setting:
sysctl -w net.ipv4.ip_forward=1
you convert your EC2 VM into a router, allowing it to accept encrypted VPN packets on one interface and forward the decrypted inner traffic out another

3. Adding a VPC Route for the Azure CIDR
-----------------------------------------
Your AWS VPC needs to know where to send packets destined for the Azure network (e.g. 10.1.0.0/16). By adding a route in the VPC’s route table:
- Destination: the Azure VNet CIDR block
- Target: your VPN EC2 instance
you ensure that any AWS-bound host sending to an Azure address will hand off its packet to the VPN instance rather than dropping it locally


Putting It All Together
------------------------

Traffic from AWS → Azure
--------------------------
> An EC2 in AWS sends to an Azure IP.
> The VPC route directs it to the StrongSwan VM.
> Because source/dest checks are off, the VM accepts it.
> With IP forwarding on, the VM decrypts and routes it over the IPsec tunnel to Azure.

Traffic from Azure → AWS
-------------------------
> The Azure VM forwards AWS-bound packets to its StrongSwan VM.
> That VM (with its own source/dest checks off and IP forwarding on) decrypts and forwards them back into AWS.
> The AWS VPC route table delivers them to the final EC2 target.

Without any one of these three—source/dest check disabled, IP forwarding enabled, or the VPC route pointing at your VPN instance—traffic would either be dropped by AWS, dropped by the Linux kernel, or never reach your VPN VM in the first place.

```

---

### ✅ **5. What were the VPC CIDRs used in AWS and Azure? Why should they not overlap?**

* AWS VPC CIDR: `172.0.0.0/16`
* Azure VNet CIDR: `10.0.0.0/16`

**They must not overlap** because:

* Routing and NAT won’t work correctly if both networks have the same address range.
* Overlapping CIDRs create ambiguity in packet delivery.

---

### ✅ **6. Why did you need to disable source/destination checks in AWS?**

see ans (ques 4 )
---

### ✅ **7. What is the role of IP forwarding in this setup?**

see ans (ques 4 )

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
WORKFLOW OF CRUD OPERATION BEING done in db
----------------------------------------------
Here’s what happens, in everyday terms, when your application does a “create, read, update or delete” (CRUD) on a Patroni-managed Postgres cluster behind HAProxy:

1- Your app opens a single connection to HAProxy
– Imagine you dial “haproxy.company.com:5000” from psql, Python, Grafana or any client.
– You don’t need to know which database node is the leader.

2- HAProxy checks “Who’s Master?”
– Every few seconds, HAProxy quietly asks each Patroni node: “Are you the primary?” (it hits their http://host:8008/health endpoint).
– Only the leader answers “Yes (200 OK)”; replicas say “No (503).”
– HAProxy keeps track of exactly one “up” node—the one you should talk to.

3- HAProxy forwards your SQL to the leader
– You send, for example, INSERT INTO users (name) VALUES ('Alice');
– HAProxy hands that SQL over to the primary Postgres on port 5432.

4- The leader runs your query and logs it
- Write-Ahead Log (WAL): First it scribbles the change into its journal on disk—so nothing is lost even if it crashes immediately afterward.
- Apply change: It updates the real database tables.
- Acknowledge: As soon as the transaction commits, Postgres sends back a “Done” message.

5- HAProxy sends the response back to you
– HAProxy grabs that “Done” or “Here are your rows” reply and pushes it back down your open connection.
– To your application, it looks just like talking to a single database.

6- Replicas catch up behind the scenes
– Immediately after the leader commits, it ships the WAL entries over to each standby node.
– Those replicas replay the same journal entries so they stay nearly identical copies.
– You don’t have to wait for them—your app already got the OK.

7- If the leader dies mid-operation
– Patroni (via etcd) notices the leader heart-beat stopped → they hold a quick election.
– A replica becomes the new leader.
– HAProxy’s next health check sees the new leader and instantly flips its traffic over.
– Your app’s next query still goes to “the” primary—no client-side changes needed.

```




```
INTERVIEW ANSWER
------------------

I set up a highly available PostgreSQL cluster using Patroni for replication and leader election, HAProxy for routing, and etcd as the DCS.
Patroni continuously monitored node health via etcd and auto-promoted a replica to primary on failure.
HAProxy detected the new leader using Patroni’s health API and routed traffic with zero manual change.
WAL-based replication ensured real-time sync to standbys, and use_pg_rewind allowed fast rejoin after failover.
All this was automated via Ansible, ensuring seamless failover and minimal downtime.

Q. How client actually connected to cluster?
Ans. Applications connected to HAProxy on port 5000, which always forwarded SQL traffic to the current leader node.

Q. What is pg_rewind?
Ans. pg_rewind is a PostgreSQL tool that lets a former primary rejoin the cluster as a replica after failover without a full resync, by replaying only the missing WAL changes from the new primary.

Q. What is WAL ?
Ans. WAL (Write-Ahead Logging) in PostgreSQL is a mechanism where changes are first written to a log file before being applied to the database.

🔍 In simple words:
WAL ensures data durability and crash recovery by recording every change in a sequential log before it's committed to disk.

✅ Benefits:
- Enables replication to standby servers
- Supports crash recovery
- Prevents data loss during failures

Q. How HA-PROXY was configured?
A. I configured HAProxy in TCP mode to route PostgreSQL traffic to the current primary node in a Patroni-managed cluster.
It used HTTP health checks on Patroni’s /health endpoint (port 8008) to detect which node was the leader, and forwarded all SQL traffic (on port 5000) exclusively to that node.
This setup didn’t perform classic load balancing across nodes, but instead ensured reliable leader-based routing — automatically switching to the new primary on failover.
I also enabled an HAProxy stats dashboard on port 7000 for real-time visibility, and tuned health check thresholds (fall, rise, on-marked-down) for fast failover.

✅ Who returns the 200/503 status codes?
 It is Patroni that returns the 200 OK or 503 Service Unavailable HTTP responses.
👉 HAProxy just checks those responses to decide where to route traffic.
- Each PostgreSQL node runs Patroni, which exposes a health endpoint: http://<node_ip>:8008/health
This /health endpoint behaves as:
- 200 OK → This node is the current leader (i.e., primary)
- 503 Service Unavailable → This node is a replica, not eligible for writes

Your HAProxy config has this:
-----------------------------
option httpchk
http-check expect status 200
check port 8008

This tells HAProxy to: "Only forward SQL traffic to nodes whose /health responds with 200."

✅ TL;DR:
Role	Returns 200/503	Purpose
Patroni	✅ Yes	Says "I'm leader" or "I'm not"
HAProxy	❌ No	Just checks the response to decide

So, Patroni returns the health codes, and HAProxy acts based on them.

Let me know if you want a curl example to test the /health endpoint manually.



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


