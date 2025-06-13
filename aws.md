
# 🟢 Basic AWS Interview Questions (Beginner)

------------------------------------------------------------------------------------------------------------------------------------------------------------------

## ✅ General

✅ 1. What is AWS? What are its main services?
📌 Answer: AWS (Amazon Web Services) is a secure, cloud services platform by Amazon that provides on-demand computing resources (servers, storage, databases, etc.) over the internet, with pay-as-you-go pricing.
🔧 Main Categories of AWS Services:
| Category       | Example Services             | Purpose                                     |
| -------------- | ---------------------------- | ------------------------------------------- |
| **Compute**    | EC2, Lambda, ECS             | Run apps and services                       |
| **Storage**    | S3, EBS, Glacier             | Store and retrieve data                     |
| **Database**   | RDS, DynamoDB                | Managed relational/non-relational databases |
| **Networking** | VPC, Route 53, ELB           | Securely connect and route traffic          |
| **Security**   | IAM, KMS, Shield             | Access control and data protection          |
| **DevOps**     | CodePipeline, CloudFormation | Automation & CI/CD                          |
| **AI/ML**      | SageMaker, Rekognition       | Build and deploy ML models                  |

💬 Follow-up Qs:
Q: What are some popular use cases of AWS?
A: Hosting websites, backup & restore, data lakes, serverless apps, media processing, AI/ML, disaster recovery.
Q: How does AWS billing work?
A: It's pay-as-you-go. You’re charged based on usage (per hour/second/GB/request) depending on the service

------------------------------------------------------------------------------------------------------------------------------------------------------------------

✅ 2. What is the difference between EC2, S3, and RDS?
📌 Answer:
| Service | Full Form                   | Purpose         | Key Feature                                              |
| ------- | --------------------------- | --------------- | -------------------------------------------------------- |
| **EC2** | Elastic Compute Cloud       | Virtual servers | Choose OS, instance type, auto-scaling                   |
| **S3**  | Simple Storage Service      | Object storage  | Store files (e.g., images, logs) in buckets              |
| **RDS** | Relational Database Service | Managed SQL DB  | Auto-patching, backups, supports MySQL, PostgreSQL, etc. |
💬 Follow-up Qs:
Q: When would you use EC2 over Lambda?
A: Use EC2 for long-running apps or custom environments. Use Lambda for short-lived, event-based functions.
Q: Is S3 good for storing databases?
A: No, S3 is object storage, not designed for relational queries—use RDS or DynamoDB for databases.

------------------------------------------------------------------------------------------------------------------------------------------------------------------

✅ 3. Explain the regions and availability zones in AWS.
📌 Answer:
Region: A physical location (e.g., us-east-1, ap-south-1) that contains multiple data centers.
Availability Zone (AZ): One or more isolated data centers in a region. Designed for fault tolerance.
Example: us-east-1 has 6 AZs (us-east-1a, 1b, 1c, ...)
🧠 Workflow Example:
Highly available web app:
> Deploy EC2 in multiple AZs in a single region behind an ELB.
> Use RDS Multi-AZ for failover.
> Store static files in S3 (regionally redundant).

💬 Follow-up Qs:
Q: Can resources in one region access another?
A: Yes, but cross-region traffic has latency & cost. Use services like CloudFront or global tables (DynamoDB) for replication.
Q: How do you make your app resilient to AZ failures?
A: Use Auto Scaling Groups across AZs, enable Multi-AZ for databases, and Route53 for failover routing.

------------------------------------------------------------------------------------------------------------------------------------------------------------------

✅ 4. What is the shared responsibility model in AWS?
📌 Answer: It defines who is responsible for what in cloud security:
| Responsibility             | AWS    | Customer |
| -------------------------- | ------ | -------- |
| Physical infra             | ✅      | ❌        |
| Network, storage, compute  | ✅      | ❌        |
| Patching OS, apps (on EC2) | ❌      | ✅        |
| Data encryption            | Shared | ✅        |
| IAM policies               | ❌      | ✅        |
Think: AWS secures the cloud, you secure what's in the cloud.

💬 Follow-up Qs:
Q: Who is responsible for data encryption?
A: AWS provides tools (KMS, SSE), but you must enable and manage encryption for your data.
Q: Is AWS responsible for EC2 security patches?
A: No, if you're using EC2. You manage the OS and app updates.

------------------------------------------------------------------------------------------------------------------------------------------------------------------

✅ 5. What are the types of cloud computing models: IaaS, PaaS, SaaS?
📌 Answer:
| Model    | Full Form                   | Example                | What You Manage       |
| -------- | --------------------------- | ---------------------- | --------------------- |
| **IaaS** | Infrastructure as a Service | EC2, EBS, VPC          | OS, runtime, apps     |
| **PaaS** | Platform as a Service       | Elastic Beanstalk, RDS | Only the app/data     |
| **SaaS** | Software as a Service       | Gmail, Dropbox         | Nothing – just use it |

🧠 Workflow Example:
Hosting a Web App:
> IaaS: You use EC2 → manage OS, install web server
> PaaS: Use Elastic Beanstalk → deploy code, AWS handles provisioning
> SaaS: Use tools like Salesforce → just login and use

💬 Follow-up Qs:
Q: Is Lambda IaaS or PaaS?
A: PaaS – You only upload code; AWS manages the infra, runtime, and scaling.
Q: What’s the main advantage of SaaS?
A: No maintenance, quick deployment, subscription-based access to ready-to-use tools.

------------------------------------------------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------------------------------------------------

## ✅ Load Balancer

1- ✅ What is a Load Balancer? Why do we use it?
📌 Answer: A Load Balancer is a service or device that distributes incoming traffic across multiple targets (e.g., EC2 instances, containers, IPs) to: * Improve availability and reliability   * Prevent server overload   * Enable fault tolerance and scaling

💡 Why Use a Load Balancer?
| Benefit                               | Explanation                                                 |
| ------------------------------------- | ----------------------------------------------------------- |
| 🔁 **High Availability**              | If one instance fails, traffic is routed to healthy ones    |
| ⚖️ **Efficient Resource Utilization** | Spreads load evenly across all instances                    |
| 📈 **Scalability**                    | Works seamlessly with Auto Scaling                          |
| 🔐 **SSL Termination**                | Offloads SSL decryption from backend servers                |
| 🌍 **Geo or Path-based Routing**      | Supports intelligent traffic distribution (e.g., API vs UI) |

🧠 Real-World Workflow (Web App Deployment):
```
User Request
     ↓
   Route 53 (DNS)
     ↓
+-----------------------+
|  Load Balancer (ALB)  | ← Health Checks
+-----------------------+
   ↓        ↓       ↓
 EC2-1    EC2-2   EC2-3  (In different AZs)
```
* All user requests first hit the Load Balancer.
* The LB checks health of each instance.
* Routes requests only to healthy and least-loaded instances.

🔍 Types of AWS Load Balancers:
| Type                  | Use Case                                | Protocol Support |
| --------------------- | --------------------------------------- | ---------------- |
| **ALB (Application)** | HTTP/HTTPS apps (Layer 7)               | HTTP, HTTPS      |
| **NLB (Network)**     | High-performance, low-latency (Layer 4) | TCP, UDP, TLS    |
| **CLB (Classic)**     | Legacy workloads                        | HTTP, TCP        |

💬 Follow-up Questions:
Q: What’s the difference between ALB and NLB?
A: 
| Feature      | ALB                     | NLB                          |
| ------------ | ----------------------- | ---------------------------- |
| Layer        | Layer 7 (Application)   | Layer 4 (Network)            |
| Protocols    | HTTP/HTTPS              | TCP/UDP/TLS                  |
| Routing Type | Path-based, Host-based  | IP and Port-based            |
| Use Case     | Web apps, Microservices | High-throughput, gaming, IoT |

Q: How does a load balancer perform health checks?
A: It sends periodic HTTP (or TCP) requests to a predefined endpoint (e.g., /healthcheck) on each target. If it fails multiple times, that instance is marked unhealthy and traffic is stopped.

Q: Can one Load Balancer span multiple Availability Zones?
A: Yes. AWS Load Balancers are multi-AZ by design, enhancing fault tolerance and high availability.

------------------------------------------------------------------------------------------------------------------------------------------------------------------

2- ✅ Types of LB in AWS 
AWS provides three main types of Load Balancers under the ELB service:
| Load Balancer Type                  | Layer                 | Protocol Support       | Best For                                |
| ----------------------------------- | --------------------- | ---------------------- | --------------------------------------- |
| **Application Load Balancer (ALB)** | Layer 7 (Application) | HTTP, HTTPS, WebSocket | Web apps, microservices                 |
| **Network Load Balancer (NLB)**     | Layer 4 (Transport)   | TCP, UDP, TLS          | Real-time apps, high performance        |
| **Gateway Load Balancer (GWLB)**    | Layer 3/4 (Network)   | GENEVE protocol        | Deploying 3rd-party security appliances |

| 🔍 Feature              | 🟩 ALB (Application Load Balancer)               | 🟦 NLB (Network Load Balancer)                       |
| ----------------------- | ------------------------------------------------ | ---------------------------------------------------- |
| **OSI Layer**           | Layer 7 (Application Layer)                      | Layer 4 (Transport Layer)                            |
| **Routing Type**        | Content-based (host, path, header, query string) | Connection-based (IP\:Port)                          |
| **Protocols Supported** | HTTP, HTTPS, WebSocket                           | TCP, UDP, TLS                                        |
| **Performance**         | Good (for web traffic)                           | Extreme (millions of req/sec, ultra-low latency)     |
| **Latency**             | Slightly higher due to Layer 7 inspection        | Very low latency (\~ms), suitable for real-time apps |
| **Target Types**        | EC2, IPs, Lambda, containers                     | EC2, IPs, private links                              |
| **Use Cases**           | Web apps, microservices, REST APIs               | Gaming, financial systems, IoT, VoIP                 |
| **Static IP Support**   | ❌ No (DNS name only)                             | ✅ Yes (Elastic IP support)                           |
| **TLS Termination**     | ✅ Yes (HTTPS)                                    | ✅ Yes (TLS)                                          |
| **Sticky Sessions**     | ✅ Supported via cookies                          | ✅ Supported via source IP                            |
| **WebSocket Support**   | ✅ Yes                                            | ❌ No native support                                  |
| **WAF Support**         | ✅ Integrated with AWS WAF                        | ❌ Not supported                                      |
| **Health Checks**       | HTTP/HTTPS (content-level)                       | TCP/HTTP/HTTPS (connection-level)                    |
| **Access Logging**      | ✅ Enabled via S3 logs                            | ✅ Enabled via VPC Flow Logs (not detailed app logs)  |
| **Pricing**             | Based on LCU + usage                             | Based on LCU + data processed                        |
| **DNS Name Format**     | `my-alb-123456.elb.amazonaws.com`                | `my-nlb-123456.elb.amazonaws.com` or EIP             |

* protocol
  ✅ ALB (Application Load Balancer)
Protocols: HTTP, HTTPS, WebSocket
🧠 ALB inspects the content of each request (like headers, path, etc.) to make routing decisions.
These are Layer 7 (Application Layer) protocols.
| Protocol      | Description                                                                       |
| ------------- | --------------------------------------------------------------------------------- |
| **HTTP**      | Standard protocol for web traffic (port 80)                                       |
| **HTTPS**     | Secure version of HTTP using SSL/TLS (port 443)                                   |
| **WebSocket** | Protocol for persistent, real-time bi-directional communication (e.g., chat apps) |
 
✅ NLB (Network Load Balancer)
Protocols: TCP, UDP, TLS
🧠 NLB doesn’t inspect content — it simply forwards traffic based on IP and Port, making it extremely fast and lightweight.
These are Layer 4 (Transport Layer) protocols.
| Protocol | Description                                      | Use Case                                      |
| -------- | ------------------------------------------------ | --------------------------------------------- |
| **TCP**  | Reliable, connection-based protocol              | Web servers, databases, email                 |
| **UDP**  | Fast, connectionless protocol                    | VoIP, video streaming, gaming                 |
| **TLS**  | Secure version of TCP (Transport Layer Security) | Secure non-HTTP traffic like FTPS, SMTP, MQTT |

💡 Why this matters:
| Use Case                                      | Use     | Why                                           |
| --------------------------------------------- | ------- | --------------------------------------------- |
| Host/path-based routing                       | **ALB** | Only ALB can "read" URLs and headers          |
| Real-time voice chat (UDP)                    | **NLB** | Only NLB supports UDP                         |
| Secure custom protocols (e.g., MQTT over TLS) | **NLB** | Only NLB supports TLS (non-HTTP)              |
| WebSocket chat app                            | **ALB** | ALB supports persistent WebSocket connections |

🧪 Example Scenario:
| Scenario                                       | Choose |
| ---------------------------------------------- | ------ |
| API Gateway → `/api/v1` and `/admin`           | ALB    |
| Trading platform → Low latency TCP connections | NLB    |
| IoT device sending UDP packets                 | NLB    |
| Chat web app using WebSockets                  | ALB    |


🔧 Architecture Workflows (Visual)

1. Application Load Balancer (ALB) Workflow
```
Client
  ↓
Route 53 (DNS)
  ↓
+-----------------------------+
| ALB (Layer 7)               |
|  - Path-based routing       |
|  - Host-based routing       |
+-----------------------------+
     ↓           ↓
 [Target Group 1]  [Target Group 2]
     ↓                  ↓
  EC2: /api/*       EC2: /app/*
```
* ALB can route based on URL (e.g., /api goes to backend; /app goes to frontend).
* Perfect for microservices, containers, web apps.

2. Network Load Balancer (NLB) Workflow
```
Client
  ↓
Route 53 (DNS)
  ↓
+-----------------------------+
| NLB (Layer 4)               |
|  - Fast, static IP routing  |
|  - TCP, UDP traffic         |
+-----------------------------+
     ↓           ↓
   EC2-1        EC2-2 (Same or different AZ)
```
* Supports millions of requests/sec, very low latency.
* Used in VoIP, gaming, financial systems

3. Gateway Load Balancer (GWLB) Workflow
```
Client
  ↓
  VPC Endpoint or NLB
  ↓
+-----------------------------+
| GWLB (GENEVE-based routing) |
| - Transparent forwarding    |
| - Integrates with firewall  |
+-----------------------------+
        ↓
  3rd-Party Virtual Appliance
        ↓
   Back to workload VPC
```
* Use case: You want to deploy a firewall, deep packet inspection, or IDS/IPS in front of your VPC traffic.
* GWLB makes it transparent to the app, like a bump-in-the-wire security system.

💬 Follow-up Interview Questions:
1. Q: When should I use ALB over NLB?
A: Use ALB when you need advanced routing logic (e.g., based on path or host).Use NLB when you need ultra-low latency, static IP, or non-HTTP protocols (e.g., TCP, UDP).

2. Q: Can I attach ECS (containers) to a Load Balancer?
A: Yes. Both ALB and NLB support integration with ECS using target groups.

3. Q: What is a Target Group in Load Balancing?
A: A Target Group is a set of endpoints (EC2, Lambda, IP) that a Load Balancer routes traffic to.Each target group supports health checks, routing, and monitoring independently.

4. Q: How does health check work in ALB vs NLB?
A:
| LB Type | Health Check Type                 |
| ------- | --------------------------------- |
| ALB     | HTTP/HTTPS (content-based)        |
| NLB     | TCP/HTTP/HTTPS (connection-based) |

📦 Real-world Use Cases
| Scenario                                         | Preferred LB |
| ------------------------------------------------ | ------------ |
| Host/path-based routing (e.g., `/api`, `/login`) | ALB          |
| Real-time app (gaming, trading, chat)            | NLB          |
| App requiring WebSocket or WAF                   | ALB          |
| Internal app needing fixed IP for whitelisting   | NLB          |
| IoT sensor ingestion via TCP/UDP                 | NLB          |
| REST API gateway front-end                       | ALB          |

1. Q: Can I use both ALB and NLB together?
A: Yes. Common setup is: NLB as an entry point with static IP , Forward traffic to ALB for Layer 7 routing

2. Q: How does ALB route traffic between containers in ECS?
A: ALB uses dynamic port mapping with ECS. Each container gets a unique port, and ALB maps traffic via target groups.

3. Q: What happens when a target in NLB is unhealthy?
A: NLB uses TCP health checks. If the target doesn't respond within UnhealthyThresholdCount, it's deregistered until it recovers.

4. Q: Why can’t ALB give static IPs?
A: ALB uses DNS-based failover across AZs for high availability, so IPs are dynamic. Use NLB or Route 53 + ALIAS if static IP is needed.

5. Q: Can both ALB and NLB be internal (private)?
A: ✅ Yes. You can configure both as internet-facing or internal, depending on the use case.

------------------------------------------------------------------------------------------------------------------------------------------------------------------

3- ✅ Why is NLB (Network Load Balancer) faster than ALB?
The Network Load Balancer (NLB) is faster primarily because of its design at Layer 4 (Transport Layer). Let’s break it down:
🔍 1. OSI Layer Processing
| Load Balancer | OSI Layer | Processing Type                                                   |
| ------------- | --------- | ----------------------------------------------------------------- |
| **NLB**       | Layer 4   | **Connection-based routing** (TCP/UDP)                            |
| **ALB**       | Layer 7   | **Content-based routing** (inspects HTTP headers, paths, cookies) |
➡️ NLB does not inspect packet content — it just forwards traffic based on IP address and port. NLB passes packets through without parsing the application-level data. It uses kernel-mode networking, bypassing much of the userspace processing that ALB does.Because of this, NLB can handle millions of requests per second with ultra-low latency (~<1 ms).
NLB supports static IP addresses and Elastic IPs, enabling direct, low-latency routing from clients.This avoids DNS resolution time or regional rerouting delays. It's also optimized for long-lived TCP connections, making it ideal for chat, database, or game servers.
➡️ ALB parses the entire HTTP request to make decisions — this adds overhead.

“NLB does not inspect packet content — it just forwards traffic based on IP address and port.”
> NLB operates at Layer 4 of the OSI model (Transport Layer). This means it only looks at: * source IP , * Destination IP , * Source Port , * Destination Port , * Protocol (TCP/UDP/TLS)
>🧱 It does not care about what’s inside the packet payload (no inspection of HTTP headers, cookies, etc.) . Because of this, there is no parsing, no header inspection, and no conditional logic. The connection is treated like a pipe — NLB receives the packet and immediately forwards it.

“It uses kernel-mode networking, bypassing much of the userspace processing that ALB does.”
> Most modern operating systems split processing into: a) User-space (application logic, e.g., ALB logic) , b) Kernel-space (system-level processing, e.g., networking, memory)
> ALB inspects each request in user space: a) Parses HTTP requests , b) Evaluates routing rules , c) Applies WAF if enabled d) Matches headers, paths, etc.
> ⏱️ User-space = Slower because: * Context switches between kernel ↔ user space , * Parsing is compute-intensive
> NLB, on the other hand, forwards traffic entirely within the Linux kernel using optimized paths like XDP (eXpress Data Path) or DPDK (Data Plane Development Kit). ⚡ This results in: > Lower CPU usage , > Lower memory overhead , > Faster packet forwarding (sub-millisecond latency)

"Handles Millions of Connections Simultaneously"
> Because of: * Kernel-level forwarding , * No per-request parsing , * No content-level routing logic
📈 NLB can scale horizontally to handle:
* Millions of concurrent TCP/UDP connections , * Without dropping connections , * With consistent throughput and low latency

“NLB supports static IP addresses and Elastic IPs, enabling direct, low-latency routing from clients.”
* Each NLB is assigned a static IP address per Availability Zone. , * You can also attach your own Elastic IPs.

🔎 Why is this faster?
* Many systems (banks, firewalls, legacy apps) require IP whitelisting — only NLB supports that natively. , * Static IP = No DNS lookup → No delay → No variability
⚠️ ALB uses dynamic DNS entries, and IPs can change — clients must resolve DNS each time, adding latency.

“It’s also optimized for long-lived TCP connections, making it ideal for chat, database, or game servers.”
* NLB can maintain idle connections for hours without dropping. , * Supports source IP preservation — the backend can see the actual client IP. , * TCP Keep-Alive and Connection Draining are optimized in NLB for low churn.

Ideal for TCP/UDP Workloads:
| Use Case                            | Why NLB Wins                                      |
| ----------------------------------- | ------------------------------------------------- |
| **Real-time gaming**                | Fast UDP routing without overhead                 |
| **Video conferencing (e.g., Zoom)** | Handles large numbers of simultaneous TCP streams |
| **Database proxies**                | Keeps connections alive, preserves source IP      |
| **MQTT, SMTP, FTP**                 | TLS passthrough + no content parsing = fast       |

------------------------------------------------------------------------------------------------------------------------------------------------------------------

4- ALB algorithm 
🔹 1. Default Algorithm: Round Robin
This is the only algorithm officially configurable in ALB. But — it’s not the whole story. 📌 What does ALB’s "Round Robin" really do?
At the target group level, ALB uses a modified round-robin strategy that:
* Skips unhealthy targets using the health check mechanism
* Resets the round-robin loop when targets are dynamically registered/deregistered
* Still respects sticky sessions (more below)

👉 Important note:
Unlike traditional round-robin which is purely round-based, ALB may do:
* Connection draining , * Zone-awareness , *  Latency adjustments based on backend responsiveness
(This is observed but not exposed to user control)

🔹 2. Sticky Sessions (Session Affinity)
AWS ALB supports session stickiness via cookies: * AWSALB — generated by AWS , * OR custom application cookies
🔧 How it works:
> If a user’s session is associated with a target, all future requests within the configured duration go to that target.
> Stickiness overrides round robin for those clients.
> Duration: Configurable between 1 sec to 7 days.

🔹 3. Host/Path/Query Header-Based Routing
ALB supports content-based routing, which is more powerful than algorithmic balancing. This means:
  * Based on the content of the request (Host, Path, Headers, Query Strings, HTTP Method), ALB routes the request to a specific target group. * Inside that target group, Round Robin applies across targets.
📌 So the real "routing logic" happens before the algorithm kicks in. This means ALB is more like a Layer 7 intelligent router + round-robin balancer.

❌ What ALB Does NOT Support:
Unlike some enterprise load balancers (e.g., NGINX Plus, F5), ALB does NOT support: * Least Connections , * Weighted Round Robin , * Random , * Latency-based routing inside target group

ALB Load Distribution Logic
| Stage   | Logic                                          |
| ------- | ---------------------------------------------- |
| Stage 1 | Listener Rule Matching: Path/Host/Header/Query |
| Stage 2 | Target Group is selected                       |
| Stage 3 | Within that group → **Round Robin**            |
| Stage 4 | Sticky sessions override RR if enabled         |

🧠 Example: ALB with Path-Based Routing + Round Robin
Request: https://myapp.com/api/data
 * ALB matches:
      > Path rule: /api/* → Target Group A
      > Header: maybe also checks for Accept: application/json
* Inside Target Group A:
     > 3 healthy targets: EC2-1, EC2-2, EC2-3
     > Load is spread equally unless sticky sessions are enabled

------------------------------------------------------------------------------------------------------------------------------------------------------------------

5- NLB algorithm 
🔹 1. Default Algorithm: Flow Hashing (5-Tuple Hash)
All traffic distribution in NLB is done using a flow hash algorithm (TCP or UDP level).
🚀 The 5-tuple includes:
* Source IP  | * Source Port | * Destination IP | * Destination Port | * Protocol (TCP/UDP)

🔧 How does NLB's Flow Hashing work?
- NLB calculates a consistent hash for each new connection.
- That hash is used to pick a backend target.
- All packets in that connection are sent to the same target (connection stickiness by default).
- Helps achieve: * Connection stickiness , * Performance scaling , *  Even distribution for high connection churn workloads

🔹 2. Source IP Affinity ("Sticky" NLB)
- Optional setting: Enable client IP preservation
- This means the NLB will use source IP for stickiness
- Common for apps that expect long sessions or IP tracking

❌ What NLB Does NOT Support:
- No path/host/header inspection (Layer 4 only)
- No user-configurable algorithm selection
- No Round Robin or Least Connections logic

✅ Full Comparison Table: ALB vs NLB Algorithms
| Feature                  | ALB                                   | NLB                                |
| ------------------------ | ------------------------------------- | ---------------------------------- |
| Default Algo             | Round Robin                           | 5-tuple Hash                       |
| Sticky Session           | ✅ via Cookie                          | ✅ via Source IP                    |
| Content Routing          | ✅ Yes (Host, Path, Header)            | ❌ No                               |
| Custom Algorithm Support | ❌ No                                  | ❌ No                               |
| Load Awareness           | Partial (via health checks, AZ-aware) | No load awareness, just hash-based |
| Protocol Awareness       | HTTP-aware                            | TCP/UDP/TLS aware                  |

🔚 Conclusion (To Remember):
- ALB is Layer 7 smart — but only supports round robin + session affinity, with rich routing rules.
- NLB is Layer 4 fast — uses flow hash for sticky, low-latency routing, with static IP and TCP/UDP handling.
- Neither supports Least Connections or custom weighted algorithms — AWS keeps this managed and auto-scaled.

------------------------------------------------------------------------------------------------------------------------------------------------------------------

6 - 🔥 Advanced / Tricky ALB Interview Questions & Answers

1. Q: Can ALB route requests to Lambda functions?
A: Yes. ALB supports Lambda as a target type — this enables HTTP-based invocation of serverless functions with all ALB routing features.

2. Q: How does ALB determine if a target is unhealthy?
A: Through configurable health checks (HTTP/HTTPS) to a specified path. If the target fails consecutively for UnhealthyThresholdCount, it’s marked unhealthy and removed from rotation.

3. Q: What happens if all targets in a target group are unhealthy?
A: ALB continues sending requests to targets, even if all are unhealthy. No 5xx is returned from ALB unless the target returns it. This is a trick Q — ALB doesn’t fail itself.

4. Q: Does ALB support weighted routing between target groups like Route 53?
A: ❌ No. ALB cannot distribute requests across target groups with custom weights. It sends 100% traffic to the matched target group from the listener rule.

5. Q: Can ALB forward traffic to targets in different regions?
A: ❌ No. ALB is region-scoped and can only route to targets in the same region. To load balance across regions, use Route 53 + health checks.

6. Q: How does ALB support WebSocket traffic?
A: ALB fully supports WebSocket (ws:// and wss://) over HTTP/HTTPS. It maintains the long-lived TCP connection without timeout issues (up to 350 seconds idle timeout).

7. Q: Can ALB route based on HTTP headers or query parameters?
A: ✅ Yes. ALB listener rules support advanced rule conditions like: * Host , * Path , *  Header , * Query string , * HTTP method  , * Source IP
Use multiple conditions combined with “AND/OR” logic.

8. Q: What’s the max number of rules per ALB listener?
A: Up to 100 rules per listener, including the default rule.

9. Q: Can one ALB listener forward to multiple target groups?
A: ✅ Yes, by defining multiple rules on the same listener — each with different matching conditions (e.g., path /api → TG1, /admin → TG2).

10. Q: Can ALB terminate TLS at the edge and re-encrypt to backend?
A: ✅ Yes. This is called TLS re-encryption: * ALB terminates HTTPS  * Then sends HTTPS to backend EC2 (you must upload backend cert to ACM)

✅ What does “ALB terminates TLS and re-encrypts to backend” mean?
> 🔐 This is called TLS re-encryption.

🔹 💡 First: What is TLS termination?
Normally when using HTTPS:
``` Client (Browser) → ALB (HTTPS) ```
The client's HTTPS request is encrypted . ALB decrypts (terminates) the connection , Then it forwards the request in plain HTTP to backend. This is called "TLS Termination at ALB"

But…

🔸 Problem: What if the backend EC2 also expects HTTPS?
You might want: * End-to-end encryption, for compliance/security (e.g., HIPAA, PCI-DSS) , * Backend to verify client via mutual TLS , * Prevent plain-text traffic inside VPC
Then you need:  TLS Re-encryption Setup
```
Client Browser
    |
  [HTTPS]
    ↓
Application Load Balancer
    |
  [HTTPS again] ← TLS re-encryption
    ↓
Target: EC2 instance with HTTPS listener
```
ALB:
- Terminates client’s TLS (decrypts)
- Then initiates a new HTTPS (TLS) connection to the backend
- So traffic stays encrypted all the way

🔧 What needs to be done?
✅ On ALB: 
* Create an HTTPS listener (port 443)
* Upload TLS certificate for your domain (e.g., app.example.com) to AWS ACM
* Set target group protocol to HTTPS

✅ On Backend EC2:
* Run an app server (e.g., NGINX, Apache, Express) that supports HTTPS
* Use a TLS certificate (can be self-signed or from ACM if using ACM Private CA)

11. Q: How is sticky session behavior handled during Auto Scaling scale-in?
A: Sticky sessions may break if the target that user was “stuck” to is terminated. New requests would be round-robin'd until stickiness resets.

🧠 First: What are Sticky Sessions?
Also called session affinity, sticky sessions ensure: * A user is consistently routed to the same backend EC2 target , * Even if there are multiple instances behind a load balancer
This is critical for stateful web applications, like: * Shopping carts , * User dashboards , * Authenticated sessions

🔧 How does ALB implement sticky sessions?
> ALB injects a cookie: AWSALB , >  This cookie ties the client session to a specific target  , > Cookie lifetime is configurable (1 sec to 7 days)
ALB Logic:
 - Client makes request → ALB sends it to Target A
 - ALB sets AWSALB cookie with hash of Target A
 - On next request, ALB reads the cookie → sends user to same Target A

⚙️ Now: What Happens During Auto Scaling Scale-In
🔥 Scale-in = EC2 instance termination: Let’s say you have: * 3 EC2 targets: EC2-1, EC2-2, EC2-3 , * User A is sticky to EC2-3 , * Auto Scaling decides to terminate EC2-3
❌ Now the problem:
 * EC2-3 is removed from the target group
 * But user A still has AWSALB cookie that points to EC2-3
⚠️ What Happens Next?
❗ ALB behavior:
- ALB detects the target (EC2-3) no longer exists
- It ignores the stale cookie
- The user’s next request is sent to a new target via Round Robin
- A new AWSALB cookie is set for the new backend

🧪 Result:
- Session is broken
- If your application relies on in-memory session storage (not shared), user may be logged out, lose cart, etc.

🔁 Timeline of Events
| Step | Action                                         |
| ---- | ---------------------------------------------- |
| 1    | User logs in, hits `EC2-3`, sticky session set |
| 2    | Auto Scaling removes `EC2-3` due to scale-in   |
| 3    | User sends new request with cookie for `EC2-3` |
| 4    | ALB can’t find target → ignores cookie         |
| 5    | ALB picks another instance via Round Robin     |
| 6    | New sticky session cookie is generated         |

💡 How to Handle Sticky Session Breakage Gracefully?
✅ Use Shared Session Storage: Store sessions in ElastiCache (Redis/Memcached), DynamoDB, or RDS , So even if backend changes, user session persists
✅ Set longer Auto Scaling cooldowns: Prevent rapid scale-in from killing active sessions
✅ Use Application-based sticky logic:  Rather than depending on ALB, the app itself manages user affinity via tokens
✅ Use Lambda or stateless services: Avoid stickiness entirely by designing stateless apps

💬 Interview Follow-Up Questions
a). Q: Can ALB avoid terminating sticky targets during scale-in?
A: No. Auto Scaling is independent of ALB's stickiness. You cannot "pin" users to keep an instance alive. Use smarter scale-in policies (e.g., terminate lowest load).
b). Q: How can you make sticky sessions reliable during scale-in?
A: - Use external session stores (Redis) , - Make app stateless , - Gracefully drain connections before instance termination
c). Q: Is sticky session enabled by default in ALB?
A: ❌ No. You must explicitly enable it per target group, and configure cookie duration and behavior.

13. Q: What if multiple listener rules match a request?
A: ALB uses priority-based evaluation — it chooses the rule with the lowest numeric priority. If none match, the default rule is used.

14. Q: Can you use ALB with ECS Fargate?
A: ✅ Yes. ALB integrates natively with ECS/Fargate, supports dynamic port mapping, and handles service discovery via target groups.

15. Q: How does ALB handle CORS?
A: ALB does not handle CORS itself — it passes headers from backend responses. You must implement CORS headers in your app layer (e.g., Express, Flask, etc.).

16. Q: Can ALB support gRPC or HTTP/2?
A: ✅ ALB supports HTTP/2 and gRPC over HTTP/2. However:
* HTTP/2 is only supported on the frontend connection (client to ALB).
* ALB to target is still HTTP/1.1.

🚀 Bonus Tip: Tricky Misconception
Q: Does ALB do load balancing across multiple AZs automatically?
A- ✅ Yes — but only if you configure it correctly.
ALB is a region-scoped load balancer, which can span multiple Availability Zones (AZs) within that region. However, cross-AZ load balancing is not automatic by default unless all required settings are enabled.

🔧 For ALB to truly load balance across AZs, you must:
✅ 1. Enable AZs in the ALB configuration
When you create the ALB, you must explicitly select two or more AZs: ```us-east-1a, us-east-1b, etc.``` , If you only select one, it won’t balance across multiple AZs.

✅ 2. Register healthy targets in each AZ
If all targets are in only one AZ, ALB has nothing to route to in the others.

✅ 3. Enable Cross-Zone Load Balancing
ALB enables this by default, but it's important to understand:
 - With cross-zone load balancing ON: → ALB in AZ A can send traffic to targets in AZ B
 - With it OFF: → Each AZ’s ALB node sends traffic only to targets in its own AZ
Use Case: If you care about cost optimization, you might turn it off (as cross-AZ data transfer incurs cost).

🧠 Architecture Example (Correct Setup):
```
User
  ↓
ALB (spanning us-east-1a + 1b)
  ↓             ↓
EC2-A (AZ-1a)   EC2-B (AZ-1b)
```
With: * Cross-zone load balancing enabled , *  Healthy EC2 instances in both AZs ⇒ ALB evenly distributes requests between A and B.

❗ Gotcha (Trick Interview Angle):
 “If I enable 2 AZs in ALB but only put instances in one AZ, will it balance across both?”
Answer: ❌ No. ALB only routes to healthy registered targets.
Even if 2 AZs are enabled, if only 1 AZ has healthy instances, all traffic goes to that AZ.

🧮 Bonus: Cross-Zone Costs
| Feature               | ALB             | NLB              |
| --------------------- | --------------- | ---------------- |
| Cross-zone LB         | ✅ On by default | ❌ Off by default |
| Cross-AZ data charges | ✅ Yes           | ✅ Yes            |

💬 Follow-up Questions You Might Get:
1. Q: What happens if all targets in one AZ fail?
A: ALB will route all traffic to healthy targets in the other AZ, as long as cross-zone load balancing is enabled.

2. Q: Is ALB highly available if one AZ goes down?
A: ✅ Yes — as long as: * You registered targets in multiple AZs , * Your DNS (e.g., Route 53) points to the ALB DNS name * Your health checks are accurate

🧠 If You're Interviewing at a Deeper Level...
You may also get scenario questions like:
 - “How would you migrate from Classic Load Balancer to ALB?”
 - “How do you secure backend EC2s behind ALB using SGs?”
 - “How do you log requests that go through ALB?”

------------------------------------------------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------------------------------------------------

## ✅ ASG 

## 1: Basic  

1- 1. Q: What is an Auto Scaling Group (ASG) in AWS?
A: An ASg ensures that you always have the right number of EC2 instances running to handle your application load. It automatically launches or terminates EC2 instances based on defined conditions (like CPU usage, request count, schedules, etc.).

2. Q: What are the core components of an ASG?
| Component                    | Purpose                                               |
| ---------------------------- | ----------------------------------------------------- |
| **Launch Template / Config** | Blueprint for EC2: AMI, instance type, security group |
| **Min/Max/Desired Capacity** | Controls how many instances ASG maintains             |
| **Scaling Policies**         | Define how to scale (up/down)                         |
| **Health Check**             | Marks unhealthy instances for replacement             |
| **Target Group**             | For ALB/NLB integration                               |
| **Lifecycle Hooks**          | Add custom logic during scale in/out                  |

3. Q: Difference between desired, min, and max capacity in ASG?
- Min: Minimum number of instances that will always be running
- Max: Maximum limit the ASG can scale to
- Desired: The current target number ASG tries to maintain
When scaling in/out, desired count changes, within the bounds of min & max.

4. Q: Can ASG work without a Load Balancer?
A: ✅ Yes, but you lose benefits like: * Health-based routing | * Even traffic distribution | * Sticky sessions
Instead, you must manage direct instance-level endpoints manually.

------------------------------------------------------------------------------------------------------------------------------------------------------------------

##  2: Tricky / Advanced ASG

5. Q: What happens if an EC2 in ASG fails a health check?
A: ASG terminates the instance and launches a new one using the launch template.
If connected to an ALB/NLB, ALB’s health check can also trigger ASG replacement.

6. Q: Can ASG perform zero-downtime deployments?
A: ✅ Yes, using Rolling Updates, Blue/Green Deployments, or Lifecycle Hooks with step functions/shell scripts to gradually replace instances.

7. Q: How does ASG interact with Spot Instances?
A: ASG supports Mixed Instance Policies:
- You can combine On-Demand + Spot instances
-  Define weight and priority per instance type
- Handle Spot interruption notices gracefully

8. Q: What happens if you decrease desired capacity below current running instances?
A: ASG will terminate excess instances, typically choosing:
- Instances in AZs with more capacity
- Or least recently used instances

9. Q: What happens during a scale-in event with sticky sessions?
A: If the target instance is handling sticky sessions, users may lose sessions unless:
- You use external session stores (Redis, DynamoDB)
- You delay instance termination using Lifecycle Hooks

10. Q: Can you manually add an EC2 instance to an ASG?
A: ✅ Yes, using AttachInstances. But: ASG takes over its lifecycle , If that instance fails or scales down, ASG may terminate it

------------------------------------------------------------------------------------------------------------------------------------------------------------------

## 3: Scaling Policies – Static vs Dynamic (with Visuals)

✅ A. Static Scaling
| Type           | Behavior                                                        |
| -------------- | --------------------------------------------------------------- |
| **Fixed Size** | ASG maintains a constant number of instances regardless of load |
| **Use Case**   | Small apps, batch jobs, dev environments                        |

📉 Visual:
```
Min = Max = Desired = 3
  |
Always 3 running instances
  ↓
Load spikes — no scaling occurs
```
✅ B. Dynamic Scaling
Adjusts capacity automatically based on metrics, like CPU, memory, requests, queue length.
🔸 Subtypes of Dynamic Scaling:

1. Target Tracking Scaling (Recommended): 
Automatically adjusts capacity to maintain a target metric value (e.g., keep CPU at 60%)
* Easy to configure (just define the target)
* AWS manages all thresholds and math
Example:
```
Target: CPU Utilization = 60%
If actual CPU > 60% → Scale out
If actual CPU < 60% → Scale in
```
📈 Visual:
```
    Load ↑       Load ↓
      |            |
    +1 EC2      -1 EC2
```

2. Step Scaling
Define step thresholds and corresponding actions (fine-tuned scaling)

Example:
```
If CPU > 70% → Add 1 instance
If CPU > 85% → Add 2 instances
If CPU < 40% → Remove 1 instance
```
- Offers granular control
- Preferred for predictable workloads

📉 Visual:
```
 CPU: 72% → +1 EC2
 CPU: 87% → +2 EC2
 CPU: 35% → -1 EC2
```

3. Simple Scaling (Deprecated)
Old method — trigger action after a cooldown (no re-evaluation)
 - Slower, not recommended anymore
 - AWS replaced this with Target Tracking and Step Scaling

✅ C. Scheduled Scaling
Predefined scaling based on time (like cron jobs)
Example:
 * Scale out to 10 instances at 9 AM (office hours)
 * Scale in to 2 instances at 10 PM
Useful for:
 * Predictable loads (e.g., day/night traffic)
 * Cost optimization

📅 Visual:
```
09:00 → Set desired = 10
22:00 → Set desired = 2
```

🧠 Bonus Scenario-Based Question
🔥 Q: Suppose your ASG has: Target tracking set to CPU = 60% ,  Scheduled scaling set to desired = 2 at 8 PM | At 7:59 PM, load spikes and CPU reaches 90%. What happens at 8 PM?
A: 
* Scheduled action overrides dynamic scale-out
*  Desired count is forcibly reset to 2
ASG scales in even if CPU is high (unless cooldown prevents it)
⚠️ This is a trick question to test order-of-precedence understanding.

🧠 Common Follow-up Questions
Q: Which scaling policy takes precedence?
Scheduled scaling overrides dynamic temporarily during scheduled period

Q: What’s a cooldown period?
It’s a buffer time (default: 300s) after a scaling event before another is triggered. Target Tracking has its own internal cooldown logic.

Q: Can you combine multiple scaling policies?
✅ Yes: Use scheduled + target tracking + step scaling together , AWS picks the most aggressive scaling action

------------------------------------------------------------------------------------------------------------------------------------------------------------------

✅ 1. Cooldown Period (Core Concept)

🔹 What is a Cooldown Period?
A cooldown period is a waiting time after a scaling activity (scale-out or scale-in), during which no new scaling activities are allowed. This prevents rapid, repeated scaling which can lead to:
* Over-provisioning , * Cost spikes , * Application instability

🔧 Default Cooldown (Simple/Step Scaling)
- Default: 300 seconds (5 minutes) | - During cooldown: ASG ignores further scaling triggers , Instance metrics are not evaluated

📌 How it Works (Example)
| Time        | CPU (%)                                                   | Action                  |
| ----------- | --------------------------------------------------------- | ----------------------- |
| 10:00       | 85%                                                       | ASG triggers **+1 EC2** |
| 10:01–10:05 | Even if CPU remains 90% → ❌ **No action due to cooldown** |                         |
| 10:06       | Cooldown ends → ASG re-evaluates CPU                      |                         |
| 10:06       | CPU still 90% → ✅ Another scale-out occurs                |                         |

🧠 Cooldown Types in AWS
✅ A. Default Cooldown
- Applies globally to ASG (for simple/step scaling) , - Used by default unless overridden per policy

✅ B. Scaling Policy-Specific Cooldown
- In step scaling, you can define a custom cooldown per policy, - More granular than global cooldown , - You can have aggressive cooldowns for scale-out (e.g. 60s) and conservative ones for scale-in (e.g. 300s)

✅ C. Target Tracking Cooldown (Built-in)
- Doesn’t use traditional cooldown , - AWS manages an internal cooldown automatically - More responsive and intelligent → evaluates metrics continuously

------------------------------------------------------------------------------------------------------------------------------------------------------------------

✅ 2. Warm-up Time (for Target Tracking)
🔹 Purpose: When an instance is launched, it takes time before it becomes fully operational and starts serving traffic. If scaling policies read metrics too soon, you could trigger false scale-outs.

🔧 Solution: "Warm-up Time" configuration
- You can configure the warm-up period (e.g., 180s) in Target Tracking policies
- During warm-up: The new instance is excluded from aggregated metrics , After the period ends → instance is included in scaling decision

📌 Visual Timeline
```
Time 0 → Launch EC2
       → Warm-up 180s
       → Becomes "active" for scaling policy
```

------------------------------------------------------------------------------------------------------------------------------------------------------------------

✅ 3. Lifecycle Hooks
Allows you to pause instance launch or terminate to run custom logic before proceeding.
🔧 Use Case:  * Inject configuration | * Run security scans | * Save session data before termination

🔁 Two types of hooks:
| Hook Type                    | Purpose                           |
| ---------------------------- | --------------------------------- |
| **Launch Lifecycle Hook**    | Pause before EC2 enters InService |
| **Terminate Lifecycle Hook** | Pause before EC2 is terminated    |

During the pause: - You can invoke a Lambda, SNS, or SQS | - Complete the hook via ``` aws autoscaling complete-lifecycle-action ```

------------------------------------------------------------------------------------------------------------------------------------------------------------------

✅ 4. Instance Refresh
Allows you to gradually replace running instances with updated AMIs, configs, or user data without downtime . Supported natively in ASG
You define: * Batch size (e.g., 20%) | * Min healthy % | * Pause time between batches
✅ Better than deleting/replacing instances manually

💬 Interview Follow-Up Questions
1. Q: What happens if cooldown is too short?
A: It may cause scaling loops — e.g., new instances launch before previous ones stabilize, leading to excessive scaling.

2. Q: Can cooldown and warm-up overlap?
A: Yes — cooldown controls ASG actions, while warm-up affects metric evaluation. They operate in different layers.

3. Q: Can I bypass cooldown manually?
A: ✅ Yes, via the AWS CLI: ```aws autoscaling set-desired-capacity --honor-cooldown false```












