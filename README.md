1. What is Kubernetes and how does it work?
Kubernetes is a tool that helps you run and manage many containers across multiple servers. It handles scaling, deployment, and restarting containers automatically.

2. What is the difference between a Pod, ReplicaSet, Deployment, and StatefulSet?
Pod: Smallest unit in Kubernetes. It runs one or more containers together.
ReplicaSet: Ensures a set number of Pods are always running.
Deployment: Manages ReplicaSets and updates Pods easily.
StatefulSet: Like Deployment but used for apps that need stable names or storage (like databases).

3. Why are Pods ephemeral?
Pods are temporary—they can be deleted or replaced at any time due to scaling or failures. That’s why they are not meant to store permanent data.

4. What is the role of the Kubelet?
Kubelet runs on each node. It makes sure the containers in the Pods are running and healthy.

5. Explain the difference between kubectl apply and kubectl create.
kubectl create: Used to create a resource for the first time.
kubectl apply: Used to create or update a resource. Good for making changes.

6. What is a Namespace and why is it used?
A Namespace is a way to divide cluster resources between users or teams. It helps organize and manage large clusters.

# ===============================================================================================================================================

🏗️ 2. Architecture & Components

🧱 1. Describe the Kubernetes architecture.
Kubernetes has a Control Plane and Worker Nodes. The Control Plane manages the cluster.
Worker Nodes run the actual applications in Pods.
Key Components:
* Control Plane: API Server, Scheduler, Controller Manager, etcd
* Node Components: Kubelet, Kube Proxy, Container Runtime
Follow-up Interview Point:
👉 “How do these components talk to each other?”
Answer: Mostly through the Kube API Server, which acts as the central communication point.

🧩 2. What are the components of the Control Plane?
API Server – Frontend for the cluster, all requests go through this.
Scheduler – Places new Pods on suitable Nodes.
Controller Manager – Handles background tasks (like scaling, restarts). amke sure cluster stays in desired state
etcd – Stores the cluster’s configuration and state (key-value store).

Follow-up Interview Point:
👉 “Is API Server stateless?”
Yes. API Server is stateless and can be horizontally scaled.

Behind-the-scenes:
Controllers watch the cluster state via API Server, then make changes by writing back to it. etcd stores all this information.

📦 3. How does the Scheduler decide where to place a pod?
Simple Answer:
The Scheduler looks at available nodes and decides based on:
- Resource availability (CPU, RAM)
- Node labels, taints/tolerations
- Affinity/anti-affinity rules

Follow-up Interview Point:
👉 “Can we influence the scheduler decision?”
Yes, using nodeSelector, affinity rules, and taints/tolerations.

Behind-the-scenes:
Scheduler gets a list of candidate nodes from the API Server, applies filters, then scores them, and finally selects the best match.

🧠 4. What is the role of etcd?
etcd is a distributed key-value store that stores all cluster data—like config, status, secrets, etc.

Follow-up Interview Point:
👉 “What happens if etcd fails?”
Cluster can’t function fully—no scheduling or state management.
Important: Take etcd backups regularly—losing etcd = losing cluster state.

Behind-the-scenes: All changes (like pod creation, scaling) are written to etcd via API Server, and controllers act based on this data.

⚙️ 5. Explain the Controller Manager and what controllers it runs.
Controller Manager runs various controllers that make sure the cluster stays in the desired state.
Common Controllers:
- ReplicationController / ReplicaSet
- Node Controller (monitors node health)
- Job Controller
- Endpoint Controller

Follow-up Interview Point:
👉 “What happens when a node goes down?”
Node Controller notices and marks it as NotReady, and other controllers may restart pods elsewhere.

Behind-the-scenes:
Each controller watches etcd (via API Server), compares current vs. desired state, and makes changes.

📦 6. What is the Container Runtime Interface (CRI)?
The CRI (Container Runtime Interface) is a standard API. It defines how Kubernetes talks to container runtimes like containerd, CRI-O, etc. This means Kubernetes doesn’t care who makes the runtime — as long as it follows the CRI standard, Kubernetes can talk to it.
Kubernetes talks to the container runtime (like containerd) through CRI. This standardizes how Kubernetes tells the runtime to:
- Start containers
- Stop containers
- Pull images
- Check container status

Follow-up Interview Point:
👉 “Is Docker a runtime now?”
- Docker support was deprecated in Kubernetes 1.20+. Use containerd now.
What happened to Docker in Kubernetes?
Kubernetes deprecated Docker as a runtime starting from v1.20. It doesn’t mean Docker images won’t run — you can still build images using Docker, but Kubernetes uses containerd under the hood to run them.

- Important Point: CRI ensures flexibility—you can plug in different runtimes without changing Kubernetes core. The Kubelet uses CRI to start/stop containers on a node.

Behind-the-scenes:
Kubelet calls the CRI endpoint (like containerd's API) to manage containers in Pods.

7. How does the Kubelet use CRI to start/stop containers?
       > What is the Kubelet?
          The **Kubelet** is the agent that runs on every **worker node** in Kubernetes. Its job is to:
                           * Talk to the Kubernetes API server
                           * Ensure the containers (pods) defined in the API actually run on the node
                           * Report the node's status back to the control plane

      >  What is the CRI?
         CRI = Container Runtime Interface.It’s a **standard API** that the Kubelet uses to **talk to the container runtime** on the node — such as `containerd` or `CRI-O`.

      >  So, how does the Kubelet use CRI to start/stop containers?
             Here’s what happens **step by step** when the Kubelet gets instructions from the control plane:
              Example: Kubelet starting a Pod
                       1. **Kubernetes API Server** sends a pod spec (YAML) to the Kubelet.
                       2. The **Kubelet parses the pod spec**.
                       3. The Kubelet makes **gRPC API calls** (defined by the CRI) to the container runtime (like containerd), asking it to:
                             * Pull the required container image (`PullImage`)
                             * Create a sandbox for the pod (`RunPodSandbox`)
                             * Start containers (`CreateContainer`, `StartContainer`)
                       4. The container runtime **executes those instructions** and runs the containers.
            Example: Stopping Containers
                      If the pod is deleted or the container crashes, the Kubelet uses CRI calls like:
                             * StopContainer
                             * RemoveContainer
                             * RemovePodSandbox

          Diagram (Simplified):

                                  Control Plane
                                       ↓
                                  API Server
                                       ↓
                             Kubelet (on Worker Node)
                                       ↓ 
                            uses CRI (standard gRPC API)
                                       ↓ 
                          Container Runtime (containerd / CRI-O)
                                       ↓
                      Runs the actual containers on the node

# =========================================================================================================================================================
 ## 🧠 Kubelet – Node Agent
------------------------------------------------------------------------------------------------------------------------------------------------------------
1-How does the Kubelet detect changes in PodSpecs from the API server?
2-What is a Pod Sandbox, and how does the Kubelet manage it?
3-How does the Kubelet decide when to restart a failed container?
4-What happens if the container runtime is down, but the Kubelet is running?
5-How does the Kubelet determine resource availability for scheduling pods?

🧠 1. How does the Kubelet detect changes in PodSpecs from the API server?
Answer: Kubelet uses the Kubernetes API Server watch mechanism to monitor Pod changes.

Workflow:
* Kubelet registers itself with the API Server.
* It watches for PodSpecs (bound to its node) via /api/v1/pods.
* When a new or updated PodSpec is detected, Kubelet:
       - Validates it
       - Pulls container images
       - Starts containers via the Container Runtime Interface (CRI)

Follow-up:
👉 “Does Kubelet poll the API server?”
No, it uses watch (via long-lived HTTP connection) for real-time updates.

-------------------------------------------------------------------------------------------------

🧱 2. What is a Pod Sandbox, and how does the Kubelet manage it?
Answer: A Pod Sandbox is a special container that sets up the network namespace and security context for all containers in a Pod.

Workflow:
* Kubelet calls the CRI to create a Pod Sandbox.
* It sets up:
      -Network (IP address, DNS, hostname)
      -cgroups and namespaces
      -Then Kubelet launches containers inside that sandbox.

Key Point:
If the Pod Sandbox fails, all containers in that Pod are recreated.

Follow-up:
👉 “How do you check if the Pod sandbox is healthy?”
Check kubectl describe pod → under Events → look for FailedSandbox.

-------------------------------------------------------------------------------------------------

3. How does the Kubelet decide when to restart a failed container?
Answer: It follows the Pod's restartPolicy (Always, OnFailure, or Never).

Workflow:
    - Kubelet watches container status via CRI (containerd).
    - On failure (non-zero exit code):
          *If restartPolicy: Always → restart it immediately.
          *If OnFailure → restart only on non-zero exit.
          *If Never → don’t restart.

Important:
Restart logic is Kubelet’s job, not controlled by the container runtime.

Follow-up:
👉 “Where is the backoff delay handled?”
Kubelet applies exponential backoff to avoid rapid restarts.

👉 how kubelet watches containers 
1-Kubelet communicates with the container runtime (e.g., containerd) through the CRI gRPC API.
2-It uses CRI methods like:
    * ListContainers()  *ContainerStatus()   *ListPodSandbox()
3- Kubelet does not directly "watch" container state like the API Server watch mechanism. Instead, it polls container runtime status regularly through CRI calls.
4- Based on container status (e.g., exit code, health), Kubelet takes actions like:
    *Restarting containers *Reporting to the API Server  *Updating Pod status

-------------------------------------------------------------------------------------------------

⚠️ 4. What happens if the container runtime is down, but the Kubelet is running?
Answer:Kubelet cannot manage containers—Pod creation, deletion, and health checks will fail.

Workflow:
    - Kubelet pings CRI socket (e.g., /run/containerd/containerd.sock)
    - If unreachable:
              * Kubelet logs errors
              * Containers can't be created/stopped
              * Node may go NotReady if health checks fail

    Follow-up:
👉 “Will running containers keep working?”
Yes, existing containers may keep running, but Kubelet can’t monitor or restart them

-------------------------------------------------------------------------------------------------

📊 5. How does the Kubelet determine resource availability for scheduling pods?
Answer: Kubelet doesn’t schedule pods—it reports node resource status to the API Server, and the Scheduler makes decisions.

Workflow:
* Kubelet uses cAdvisor to collect:
      - CPU, memory usage
      - Disk and network stats
* Sends these metrics in a NodeStatus update to API Server.
* Scheduler uses this info to decide where to place pods.

Important:
Kubelet also enforces resource limits defined in the Pod (requests and limits) via cgroups.

Follow-up:
👉 “Can Kubelet reject a pod?”
Yes, if the Pod requests more resources than available, Kubelet may reject it (esp. with CPU/memory pressure).

# =========================================================================================================================================================
##   🌐 kube-proxy – Networking
-----------------------------------------------------------------------------------------------------------------------------------------------------------

#### what kube proxy is ? 
#### It's a networking component that runs on each node in a Kubernetes cluster.Its job is to make the Service IPs (like ClusterIP) actually work by sending traffic to the #### right backend Pods.
-
-----------------------------------------------------------------------------------------------------------------------------------------------------------
🌐 1. How does kube-proxy manage Service IPs and route traffic to pods?
Answer: 
What kube-proxy is:
    * It's a **networking component** that runs on **each node** in a Kubernetes cluster.
    * Its job is to **make the Service IPs (like ClusterIP) actually work** by sending traffic to the right backend Pods.

What happens when a Service is created?**
     1. You define a **Service** (e.g., `my-service`) of type `ClusterIP`.
     2. Kubernetes gives it a **virtual IP** (like `10.0.0.5`) inside the cluster.
     3. The Service **selects Pods** using a label selector (e.g., all Pods with label `app=web`).
     4. Kubernetes creates an **Endpoints object**, which lists the IP addresses and ports of the selected Pods (example: `10.244.1.7:80`, `10.244.2.3:80`).

What kube-proxy does:
         Now kube-proxy comes into action:
  Step 1: Watches for changes: kube-proxy is watching the Kubernetes API Server** for any changes in:
              * Service objects
              * Endpoints objects

  Step 2: Creates routing rules
         * kube-proxy reads the Service IP and backend Pod IPs.
         * Based on your OS and kube-proxy settings, it sets up either:
         * iptables rules (default in most clusters), or
         * IPVS rules (for better performance).

  These rules say: "If anyone tries to access 10.0.0.5:80, forward the request to one of these real Pods: 10.244.1.7:80 or 10.244.2.3:80."

  What happens during a real request:
           Let’s say a pod in the cluster does: ``` curl http://10.0.0.5:80```
             

 Behind the scenes:
          1. The **Linux networking stack** hits an **iptables rule** created by kube-proxy.
          2. This rule **redirects the request to one of the Pod IPs** randomly (load-balancing).
          3. The request **never even reaches kube-proxy as a process** — Linux handles it directly using the rules kube-proxy set earlier.

 So kube-proxy **programs the rules**, but **doesn’t touch every packet**.

 How kube-proxy load-balances traffic:
          * In iptables mode: It uses round-robin to forward to different Pods.
           *In IPVS mode**: It supports advanced load balancing algorithms like:
                   * Round Robin   * Least Connections   * Source Hashing
             You can choose IPVS by passing: ``` --proxy-mode=ipvs ```


Bonus: What are ClusterIP and Endpoints?
        * ClusterIP: is just a virtual IP — not tied to any real interface.  It only works inside the cluster.
        * The **Endpoints** object holds the **real Pod IPs** to which traffic should go.

---

 Summary (visualized):
```
User Pod (curl 10.0.0.5:80)
   ↓
Linux Networking Stack
   ↓
iptables rule (set by kube-proxy)
   ↓
Real Pod IP (e.g., 10.244.1.7:80)
```

-----------------------------------------------------------------------------------------------------------------------------------------------------------

🔁 2. What is the difference between iptables and IPVS modes in kube-proxy?
Both are proxy modes that kube-proxy can use to manage how traffic is routed from a Service IP to the actual Pod IPs.

🔹  iptables mode – Traditional default
💡 How it works:
           kube-proxy creates iptables rules (Linux firewall rules). 
            Every time a request hits a Service IP (like 10.0.0.5), the kernel checks the iptables rules and redirects the packet to one of the backend Pods.
⚙️ Workflow:
       Example rule: If traffic is for 10.0.0.5:80, redirect to one of: 10.244.1.7:80  10.244.2.5:80
       Load balancing is done using random or statistic match in iptables.

📉 Drawbacks:
        * Performance degrades when there are many Services and Pods because:
        * iptables is rule-order dependent — it has to process one rule at a time in order.
        * Every change (new Pod/Service) causes full rebuild of the rule chain — time-consuming.

🔹 2. IPVS mode – Advanced and optimized
💡 How it works:
           Uses Linux IPVS (IP Virtual Server), a built-in kernel-level load balancer.
           kube-proxy programs IPVS rules using the ipvsadm tool or kernel APIs.
           The kernel then uses a hash table, not a rule chain, for routing — much faster.

⚙️ Workflow:   Maintains a service table in the kernel:
                ``` IPVS: 10.0.0.5:80 → [10.244.1.7:80, 10.244.2.5:80] ```              
                 Picks one Pod using one of many algorithms: Round Robin ,Least Connections, Source IP Hash

🚀 Advantages: Much faster routing, especially in large clusters (1000s of Services/Pods).
               Rule updates are incremental — only changes are applied, not full table rewrite.


🔁 Side-by-side comparison:
| Feature         | **iptables**                           | **IPVS**                                |
| --------------- | -------------------------------------- | --------------------------------------- |
| **Mode**        | Sequential rule matching (`netfilter`) | Kernel-based virtual server             |
| **Performance** | Slower with scale                      | Faster, especially in large clusters    |
| **Rule Mgmt**   | Full table rebuild on updates          | Only affected entries updated           |
| **Algorithms**  | Only basic round-robin/statistic       | Many (Round Robin, Least Conn, etc.)    |
| **Memory use**  | Lighter, good for small clusters       | Slightly higher but scalable            |
| **Stability**   | Stable, default in most clusters       | Stable but needs IPVS modules installed |

🧠 Key Point:
          *  Use iptables in small/simple clusters.
          * Prefer IPVS in production or large-scale clusters for: Faster failovers , Scalability , Advanced load balancing

🛠️ How to enable IPVS mode: 
         In your kube-proxy config or DaemonSet:
``` 
command:
  - kube-proxy
  - --proxy-mode=ipvs
```
Also ensure the node has IPVS kernel modules:
``` lsmod | grep ip_vs ```

------------------------------------------------------------------------------------------------------------------------------------------------------------

🔄 3. How does kube-proxy handle updates to Service or Endpoints objects?
Answer: kube-proxy watches the Kubernetes API Server for changes to Service and Endpoints objects, and updates its routing rules (iptables or IPVS) accordingly — all this happens automatically and with minimal traffic disruption.

🧠 What triggers an update?
When something changes like: A new Pod is created or deleted,  A Service selector is updated, A Pod becomes Ready/Unready (so its IP should be added/removed).
This causes the Endpoints object to change, and that triggers kube-proxy to act.

⚙️ Behind the scenes: Workflow
1- 🕵️‍♂️ kube-proxy is continuously watching the API Server (using a Watch API). It monitors Service and Endpoints objects.
2- 🔄 If an update is detected:  For example: a Pod is deleted → the Endpoints list is now shorter.
3- 🔍 kube-proxy compares the old state and the new state of these objects.
    - What was added?  -What was removed? -What changed?
4- 🛠️ Based on the difference, it updates the networking rules:
        - If using iptables: it rebuilds the relevant rules.
        - If using IPVS: it incrementally updates the internal routing tables.
5 🚀 These changes take effect immediately, allowing traffic to be routed to the new set of healthy Pod IPs.

📦 Summary:
| Step            | What Happens                               |
| --------------- | ------------------------------------------ |
| Watch           | kube-proxy watches API Server for changes  |
| Detect          | Notices change in Service or Endpoints     |
| Compare         | Old vs new configuration                   |
| Update Rules    | Updates iptables/IPVS with correct Pod IPs |
| Forward Traffic | Keeps traffic flowing to valid Pods only   |

Follow-up:
👉 “Is the traffic affected during update?”
Minimal or no downtime if kube-proxy is healthy.

------------------------------------------------------------------------------------------------------------------------------------------------------------

🔄 4. Can kube-proxy be replaced or customized? (e.g., Cilium)
Answer: Yes, kube-proxy can be replaced by eBPF-based networking solutions like Cilium.
Common Alternatives:
> Cilium: Uses eBPF, replaces kube-proxy for high-performance routing.
> Calico, Kube-router: May also replace kube-proxy functionality.

Why replace it?
 - Better performance
 - Better observability
 - Native support for Network Policies

Follow-up:
👉 “What is eBPF?”
Extended Berkeley Packet Filter – allows fast packet processing in the Linux kernel without modifying kernel code.

------------------------------------------------------------------------------------------------------------------------------------------------------------

🧹 5. What happens in kube-proxy if a pod behind a service is deleted?
Answer:  When a Pod is deleted, the corresponding Endpoints object for the Service is updated, and kube-proxy removes that Pod’s IP from its routing rules — so traffic is no longer routed to the deleted Pod.

⚙️ Workflow (Step-by-step):
  1- ❌ A Pod (say 10.244.2.5) is deleted (manually or due to a crash).
  2- 🧠 The Kubernetes Endpoint controller detects this change. It updates the Endpoints object of the Service to remove 10.244.2.5.
  3- 🕵️‍♂️ kube-proxy is watching for these updates via the API Server (watch)
  4- 🔄 When it notices the Endpoints list has changed: > It updates iptables or IPVS rules. > Removes the rule forwarding to the deleted Pod.
  5- 🚫 Now, traffic going to the Service IP will not be forwarded to the dead Pod.
  6- ✔️ Users/Clients won’t hit the deleted Pod anymore — it’s cleanly removed from routing.

🔍 Behind the scenes (technical flow):
     * The Pod’s IP is removed from: Endpoints object of the Service (handled by Kube controller manager).
     * This triggers an event, which kube-proxy is subscribed to via a watch.
     * Based on your proxy mode:
          - iptables: the rules redirecting traffic to the Pod IP are removed.
          - IPVS: the backend server entry is deleted from the kernel’s load balancer.

❓ Follow-up Q: "Is there a delay in removal? 
Answer: Usually the delay is a few seconds or less — kube-proxy is fast.
But delays can happen if:
      - The API Server is under load.
      - kube-proxy is slow to process events.
      - The Pod termination signal is delayed (e.g., in a crash or node failure).
To reduce risk of stale traffic:
      - Use readiness/liveness probes to mark Pods "unready" before deletion.
      - Consider using graceful termination periods in deployments.

------------------------------------------------------------------------------------------------------------------------------------------------------------

# =========================================================================================================================================================
##   📦 API Server – Cluster Gateway 
-----------------------------------------------------------------------------------------------------------------------------------------------------------
1- 📡 How does the API server ensure high availability and leader election? How does Kubernetes make sure the API server stays up and available even if one instance goes down?
The API Server is made highly available by running multiple replicas behind a load balancer. The leader election is relevant for certain control plane components (like controller-manager or scheduler), not the API server itself — since all API server replicas are active, not just one.

🧱 High Availability (HA) of API Server
⚙️ Workflow:
1- You deploy multiple API server pods or instances (e.g., 3).
2- These are placed behind a load balancer (like AWS ELB or HAProxy).
3- Clients (like kubectl, kubelet, controllers) send requests to the load balancer.
4- The load balancer distributes traffic among healthy API servers.
5- All replicas are stateless — they store state in etcd, not locally.

📌 Important: API servers do not need leader election because they serve requests in parallel.

🤝 Leader Election – Applies to Controller Manager and Scheduler
For components like kube-controller-manager and kube-scheduler, only one instance should be active at a time, even if multiple replicas are running.
💡 How it works:
- All controller-manager/scheduler replicas try to become leader.
- They attempt to acquire a lease lock (usually stored in a Kubernetes ConfigMap or Lease object).
- One becomes the leader; others stay in standby mode.
- If the leader goes down: Others detect the lost lease.A new leader is elected automatically.
This is handled using Kubernetes leader-election library (based on distributed coordination via etcd).

🔐 Behind the Scenes (Leader Election):
- Lease object stored in Kubernetes API (e.g., /kube-system/leader-election).
- Uses etcd as the backend data store.
- Election happens by writing to the lease object:
      * First one to claim the lease wins.
      * Periodic renewal ensures leader is still alive.
      * Timeout → others can take over.

     🧪 Example Use Case:
   Let’s say 2 replicas of kube-controller-manager are running: Replica 1 becomes leader (writes lease).It manages deployments, replicasets, etc.Replica 2 does nothing — 
   just waits.If replica 1 dies, replica 2 sees lease timeout → takes over.

❓ Follow-Up Questions You Might Be Asked:
Q: Why does the API server not require leader election?
A: Because all API server instances are stateless and can serve traffic concurrently.

Q: What happens if etcd goes down?
A: The API servers will not function fully — they can’t store or retrieve cluster state.

Q: How is the health of API servers checked?
A: The load balancer uses health checks (e.g., /healthz) to route traffic only to healthy servers.

-----------------------------------------------------------------------------------------------------------------------------------------------------------
2 - ❓ What are Admission Controllers?
They are checkpoints inside the API server that act like gatekeepers for any create, update, delete requests in Kubernetes.

📦 Imagine this workflow:
      > You run:  ```kubectl create pod mypod.yaml```
      > Here’s what happens step-by-step:
           * ✅ Authentication – Are you a valid user?
           * ✅ Authorization – Are you allowed to create a Pod?
           * 🛑 Admission Controllers – Should we allow or reject this Pod based on: Rules, Security, Custom policies, Auto-editing (e.g., adding labels)
💾 If allowed → object is saved into etcd (the cluster database).

🎯 Why Admission Controllers are important:
           > They can stop risky or invalid requests.
           > They can automatically add required config (like labels or sidecar containers).
           > They are the last safety check before a resource is accepted into the cluster.
📌 Example: You try to create a Pod running as root. If PodSecurity Admission Controller is enabled, it can deny that Pod creation based on policy.

🔍 Common Admission Controllers:
| Controller Name              | What it does                               |
| ---------------------------- | ------------------------------------------ |
| `NamespaceLifecycle`         | Blocks resources in terminating namespaces |
| `LimitRanger`                | Enforces resource limits                   |
| `PodSecurity`                | Blocks insecure Pods                       |
| `MutatingAdmissionWebhook`   | Changes the object (e.g., injects sidecar) |
| `ValidatingAdmissionWebhook` | Validates but doesn't modify               |

-----------------------------------------------------------------------------------------------------------------------------------------------------------

3. 🔐 What is Authentication vs Authorization in the API Server?
✅ Short Answer: Authentication: Confirms who you are. Authorization: Decides what you are allowed to do.

⚙️ Workflow (Request Processing):
        > A request hits the API Server (e.g., kubectl get pods)
        > The API server checks:  Authentication → Is this user/token valid? Authorization → Is the user allowed to perform this action on this resource? 
         If both pass → Admission Controllers → Save to etcd

🔍 Authentication Methods:
    - Certificates  - Bearer tokens  - OIDC (OAuth2)  - Static password/token files (for testing)

🔍 Authorization Modes:
    - RBAC (Role-Based Access Control) ✅ most common  - ABAC  - Node  - Webhook

❓ Follow-up Questions You Might Get:
Q: “What happens if authn passes but authz fails?”
A: The request is rejected with a 403 Forbidden.

Q: “Can multiple authorization modes be enabled?”
A: Yes — the API server processes them in order and uses the first non-abstaining decision.

-----------------------------------------------------------------------------------------------------------------------------------------------------------

4. 🚦 How does the API server handle excessive request loads (rate limiting)?
✅ Short Answer: The Kubernetes API Server protects itself from overload by using rate limiting, priority levels, and concurrency controls. This ensures fair usage 
                  and prevents any user or component from overwhelming the server.

⚙️ Mechanisms Used:
  1. Client-side Rate Limiting (via kubelet, kubectl, etc. Clients like kubectl or kubelet have built-in limits: Example: default 5 QPS (queries 
                  per second) and 10 burst for kubelet. You can configure it: ``` kubectl --request-timeout=30s --v=9 ```

  2. API Server Priority and Fairness (APF): 
                    * Since Kubernetes v1.20+, the API server uses Priority and Fairness (APF).
                    * It divides traffic into “priority levels” and “flows”, like: > System components (high priority)  > User requests (medium)  > Background jobs (low)
             🔁 Each flow gets a fair share of API server resources. If a request exceeds its limit: It is either queued or rejected with 429 Too Many Requests.

🔐 Example Flow:
           > You have a script sending 1000 pod creation requests.
           > API server groups those requests into a “flow”.
           > If that flow is over its fair share, Kubernetes:
           > Queues some requests temporarily.
            > Drops or delays others to avoid overloading etcd and the API server.


❓ Follow-up Questions You May Get:
Q: What happens when the API server gets a lot of requests at once?
A: It queues or throttles requests based on priority/fairness.

Q: What status code is returned when throttled?
A: HTTP 429 Too Many Requests.

Q: How can you control a component’s rate?
A: By adjusting client flags like --kube-api-qps and --kube-api-burst.

-----------------------------------------------------------------------------------------------------------------------------------------------------------

5. What’s the role of etcd in response to an API request?
etcd is the backend database of Kubernetes. It stores the entire cluster state — all objects like Pods, Services, ConfigMaps, Secrets, etc. When an API request reads or writes data, the API server interacts with etcd to fetch or store that state.

⚙️ Workflow (Behind the Scenes):
Let’s say you run ```kubectl create pod mypod.yaml```  Here’s what happens: 
        - 🔐 Authentication & Authorization    - ✅ Admission Controllers    - 💾 etcd Write: API server stores the Pod object into etcd. etcd makes it durable, consistent, and replicated

Now, if you run ```kubectl get pods``` , API server reads the pod list from etcd , Returns it to you via kubectl

🔁 etcd Ensures:
| Feature               | Role                                      |
| --------------------- | ----------------------------------------- |
| **Consistency**       | All API servers get the same latest state |
| **High Availability** | Replicated across nodes                   |
| **Durability**        | Data survives restarts and failures       |

❓ Follow-up Questions You Might Be Asked:
Q: What happens if etcd goes down?
A: The cluster becomes read-only or unstable — new objects can’t be stored, and existing state may be outdated.

Q: Is data cached somewhere?
A: Yes, the API server may cache some objects in memory for performance, but etcd is the source of truth.

Q: What kind of database is etcd?
A: It’s a distributed key-value store, built on the Raft consensus algorithm for strong consistency.

# =========================================================================================================================================================
🔐 etcd – Key-Value Store
-----------------------------------------------------------------------------------------------------------------------------------------------------------
1. 📦 What exactly does etcd store? Can you give examples of keys?
etcd stores all cluster state as key-value pairs — this includes everything the Kubernetes API knows.
🔍 Examples of what etcd stores:
| Kubernetes Resource | etcd Stores                  |
| ------------------- | ---------------------------- |
| Pods                | Spec, status, metadata       |
| Nodes               | Node info and heartbeats     |
| ConfigMaps          | Config data                  |
| Secrets             | Base64-encoded secret values |
| ServiceAccounts     | Tokens, metadata             |
| Deployments, RS     | Their spec and relationships |
| Events, RBAC rules  | Audit and permission data    |

🗝️ Examples of actual etcd keys:
These keys look like file paths:
- /registry/pods/default/my-app-pod
- /registry/nodes/node-1
- /registry/secrets/default/db-password
- /registry/deployments/default/my-deployment
👉 The value stored under each key is the full object in JSON or protobuf format.

❓ Follow-up You Might Get:
Q: Can I read etcd directly?
A: Yes, but you should never manually edit etcd — always use kubectl or the API server.

-----------------------------------------------------------------------------------------------------------------------------------------------------------

2- 💥 What are the risks of a single etcd node failure?
If your etcd has only one node and it fails, the entire cluster becomes unstable.

🛑 Key R| Risk                 | What Happens                                                                     |
| -------------------- | -------------------------------------------------------------------------------- |
| ❌ No writes          | API Server can't store changes (e.g., new Pods or updates)                       |
| ⚠️ Read instability  | May not serve latest state reliably                                              |
| 🔄 Controllers break | Auto-scaling, healing, or deployments may stop                                   |
| 🧨 No recovery       | If the disk is lost and there’s no backup, the **cluster state is lost forever** |

💡 Best Practice:
- Run etcd as a 3-node cluster (or 5 in production) for:
  * High availability  * Quorum-based consistency (Raft)
👉 etcd needs a majority quorum (e.g., 2 of 3) to work.

❓ Follow-up You Might Be Asked:
Q: What’s the minimum number of etcd nodes for high availability?
A: 3 (majority is 2).

Q: What if 2 out of 3 etcd nodes fail?
A: Cluster becomes unavailable for writes (no quorum).

-----------------------------------------------------------------------------------------------------------------------------------------------------------

 3- 💾 How do you take and restore a backup of etcd?
* Use the etcdctl snapshot command to take a backup and restore it later into a new etcd cluster or node. 📸 To Take a Backup (Snapshot):
```
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /tmp/etcd-backup.db
```
✅ This saves a point-in-time copy of the etcd database to /tmp/etcd-backup.db.

* 🛠️ To Restore a Backup
```
  ETCDCTL_API=3 etcdctl snapshot restore /tmp/etcd-backup.db \
  --data-dir /var/lib/etcd-restore
```
Then use the restored --data-dir in your etcd startup config.

🔐 Important Notes:
- Always take backups periodically (daily or more).
- Back up before upgrades or config changes.
- Store snapshots off-node (e.g., S3, NFS, etc.).

❓ Follow-up Interview Questions:
Q: Can you automate etcd backup?
A: Yes — use a cronjob or Kubernetes Job + PV to back up regularly.

Q: Will restoring etcd restore everything in the cluster?
A: Yes — etcd contains all cluster objects. But it won’t restore persistent volumes unless backed up separately.

-----------------------------------------------------------------------------------------------------------------------------------------------------------
4-  🐢 What is the impact of etcd latency on the whole cluster?
High etcd latency slows down API operations, controller decisions, and scheduling — impacting the entire cluster’s responsiveness and stability.

⚠️ What Happens When etcd Is Slow:
| Area Affected                 | What You See                             |
| ----------------------------- | ---------------------------------------- |
| `kubectl` commands            | Slow or time out (e.g., `kubectl get`)   |
| Pod scheduling                | Scheduler can't read/write pod info fast |
| Controllers (e.g. Deployment) | Slow to create/update/delete pods        |
| Autoscaling & healing         | Delays in decision making                |
| Cert expiration               | May fail if renewal data can’t be stored |

🔍 Root Causes of Latency:
🔧 How to Monitor and Fix:
Monitor metrics like:
- etcd_disk_wal_fsync_duration_seconds
- etcd_server_has_leader (should always be 1)
- Use fast SSDs or dedicated nodes for etcd
- Keep etcd data size under control (avoid many Events or large Secrets)

❓ Follow-up Interview Questions:
Q: What’s the recommended storage type for etcd?
A: Fast SSDs, ideally on dedicated nodes.

Q: How can you reduce etcd load?
A: Clean up old Events, don’t store large config/secrets, use kube-apiserver caching.

-----------------------------------------------------------------------------------------------------------------------------------------------------------

5- 🔐 How does Kubernetes ensure consistency and availability in etcd reads/writes?
Kubernetes relies on etcd, which uses the Raft consensus algorithm to ensure: - Strong consistency (only one agreed-upon version of data) , - High availability (even if some etcd nodes fail)

⚙️ Workflow: etcd Write Operation (e.g., create a pod)
* 👤 You run: ``` kubectl create pod mypod.yaml   ```
* 🧠 The API Server:  Authenticates, authorizes, applies admission controls & sends a write request to the etcd leader node
* 📡 etcd Leader: - Proposes the write to its follower nodes via Raft , -  Waits for quorum (majority) to acknowledge the change (e.g., 2 of 3 nodes)
* 💾 If quorum is reached: - Leader commits the change and persists it , - Followers apply the change , - API Server gets a success response
✅ This guarantees strong consistency — all etcd nodes eventually have the same data.

🔁 Workflow: etcd Read Operation (e.g., kubectl get pods)
* Default Reads: Serve from the leader’s memory/cache (fast, consistent)
* Strong Consistency Reads:  Can force quorum read using --consistency=strong to ensure you're reading the latest committed state

🧠 How Raft Ensures Consistency:
| Concept       | Role                                  |
| ------------- | ------------------------------------- |
| **Leader**    | Accepts and coordinates writes        |
| **Followers** | Replicate data from leader            |
| **Quorum**    | Majority needed to commit a write     |
| **Term**      | Time period where a leader is elected |
⚠️ If the leader fails, a new leader is elected automatically — so etcd stays available.

📈 High Availability (HA) Strategy:
- Always use an odd number of etcd nodes (e.g., 3 or 5)
- Quorum = (N/2) + 1
Without quorum, writes will stop, but reads may continue

❓ Possible Interview Follow-ups:
Q: What happens if etcd loses quorum?
A: No writes can be made; the cluster becomes read-only or degraded.
Q: What ensures consistency in etcd?
A: The Raft protocol, which replicates logs and requires majority consensus.
Q: Can you improve etcd read performance?
A: Yes — by using API server caching, and not forcing strong reads unless necessary.

# =========================================================================================================================================================
📅 Scheduler
-----------------------------------------------------------------------------------------------------------------------------------------------------------

1- 📦 How does the Kubernetes Scheduler choose a node for a pod?
The Kube-scheduler selects the best node for a Pod by running a two-step process:
 - Filtering → Eliminate nodes that can't run the Pod
 - Scoring → Rank the remaining nodes and pick the best one

⚙️ Workflow (Behind the scenes):
- 🆕 A new Pod is created without a node assigned.
- 📡 The API server adds it to the queue.
- 🧠 Scheduler watches for unscheduled Pods.
- 🧪 Runs filters (predicates):
           * Node has enough CPU/RAM?
           * Node has required labels/taints?
           * Is node ready?
- 🧮 Runs scores (priorities):
          * Spread pods evenly (PodTopologySpread)
          * Prefer nodes with fewer Pods (LeastRequestedPriority)
          * Consider node affinity rules
✅ Picks the highest-scoring node and assigns the Pod to it.

🔁 It then patches the Pod:
```
spec:
  nodeName: chosen-node
```

❓ Follow-up Questions:
Q: Who actually starts the Pod?
A: The kubelet on the assigned node.
Q: What if all nodes fail filters?
A: Pod stays unscheduled until conditions change.

-----------------------------------------------------------------------------------------------------------------------------------------------------------

2 - 🧮 What are Predicates and Priorities (now Filter and Score)?
These were renamed in Kubernetes v1.19+:
| Old Term (v1.18 and below) | New Term (v1.19+) |
| -------------------------- | ----------------- |
| **Predicate**              | **Filter plugin** |
| **Priority**               | **Score plugin**  |

🧪 Filter Plugins (Predicates):
These eliminate non-eligible nodes:
* NodeUnschedulable (cordoned node)
* PodFitsResources (CPU/RAM not enough)
* PodToleratesNodeTaints
* NodeAffinity / NodeSelector
🔍 Think: "Can this node run the pod?"

📊 Score Plugins (Priorities):
These rank eligible nodes:
* LeastRequestedPriority (prefer low usage)
* BalancedAllocation (CPU/mem balance)
* TopologySpreadConstraint (spread across zones/racks)
* NodeAffinityPriority (custom rules)
🔍 Think: "Which of these eligible nodes is best?"

❓ Follow-up Questions:
Q: Can you customize this behavior?
A: Yes, via scheduler profiles or custom plugins.
Q: What plugin would spread pods across zones?
A: PodTopologySpread scoring plugin.

-----------------------------------------------------------------------------------------------------------------------------------------------------------

3 - 🚫  How do Taints and Tolerations influence scheduling decisions?
Taints are set on nodes to repel Pods. Tolerations are added to Pods to allow scheduling on tainted nodes. A Pod must tolerate all taints on a node to be scheduled there.

⚙️ Workflow:
- A node is tainted, for example: ```kubectl taint nodes node1 key=value:NoSchedule``` The scheduler sees this taint.  If a Pod has a matching toleration, it's allowed to be scheduled.  If not, it's filtered out during scheduling.

🧠 Use Case:
- Dedicate nodes for GPU or system workloads.
- Prevent general Pods from landing on sensitive nodes.

❓ Follow-up Questions:
Q: What are the taint effects?
A: NoSchedule, PreferNoSchedule, NoExecute
Q: What happens with NoExecute?
A: Evicts running Pods without a matching toleration.

-----------------------------------------------------------------------------------------------------------------------------------------------------------
❌ 4. What happens when no node fits the pod’s constraints?
The Pod remains Pending until a suitable node becomes available.
 Common reasons: 
    - Insufficient CPU/RAM , - Taints without matching tolerations , - Node affinity rules not met ,  - Volume limits exceeded , - Topology spread constraints too strict

❓ Follow-up Questions:
Q: How do you debug a Pending Pod?
A: Use kubectl describe pod and check Events section.

-----------------------------------------------------------------------------------------------------------------------------------------------------------
5. 🔄  How do Affinity / Anti-affinity rules influence scheduling?
They guide the scheduler to prefer or require that Pods are co-located or spread across nodes based on labels.

💡 Types:
| Type                  | Purpose                         | Example                            |
| --------------------- | ------------------------------- | ---------------------------------- |
| **Node Affinity**     | Schedule Pods on specific nodes | Based on node labels               |
| **Pod Affinity**      | Place Pods together             | E.g. web + cache on same node      |
| **Pod Anti-affinity** | Spread Pods apart               | E.g. avoid single point of failure |

⚙️ Workflow Example (Pod Anti-affinity):
```
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: web
        topologyKey: "kubernetes.io/hostname"
```
➡️ This rule prevents multiple web Pods on the same node.

❓ Follow-up Questions:
Q: What’s the difference between preferred and required?
A: required is mandatory (hard rule); preferred is best effort (soft rule).

Q: What is topologyKey?
A: It's the label like kubernetes.io/hostname or topology.kubernetes.io/zone used to determine groupings for affinit

# =========================================================================================================================================================
🔄 Controller Manager
-----------------------------------------------------------------------------------------------------------------------------------------------------------

1- 🧠 What are the types of controllers running in the Controller Manager?
The Controller Manager runs multiple built-in controllers that monitor the cluster state and reconcile it to the desired state.

⚙️ Common Controllers:
| Controller                     | Purpose                                          |
| ------------------------------ | ------------------------------------------------ |
| **Deployment Controller**      | Manages ReplicaSets for Deployments              |
| **ReplicaSet Controller**      | Ensures correct number of pod replicas           |
| **StatefulSet Controller**     | Manages unique identity pods with stable storage |
| **DaemonSet Controller**       | Ensures one Pod per node                         |
| **Job/CronJob Controllers**    | Run batch or scheduled jobs                      |
| **Node Controller**            | Tracks node status and handles node failures     |
| **EndpointSlice Controller**   | Manages network endpoint objects                 |
| **Service Account Controller** | Creates default service accounts                 |
| **Namespace Controller**       | Cleans up resources in deleted namespaces        |
| **PV/PVC Controllers**         | Manages volume provisioning and binding          |

🔄 How They Work:
- Each controller uses the Informer pattern:
- Watches objects (e.g., Deployments, Nodes)
- Compares actual vs desired state
- Issues API calls to fix the drift (e.g., creating/deleting pods)

❓ Follow-up Questions:
Q: Are these separate processes?
A: No, they run as goroutines inside the same controller-manager process.
Q: Can you add custom controllers?
A: Yes, via Operators or custom controllers using client-go.

-----------------------------------------------------------------------------------------------------------------------------------------------------------

2- 🚀  How does the Deployment controller ensure a rollout happens correctly?
The Deployment Controller manages rollouts by creating and updating ReplicaSets, ensuring Pods are updated gradually and safely.

⚙️ Workflow of a Rollout:
- You apply a new Deployment  ``` kubectl apply -f deployment.yaml ```
- The Deployment controller:
         * Detects the change
         * Creates a new ReplicaSet for the new version
         * Scales up the new RS and scales down the old RS
- It uses the strategy defined in the Deployment:
        * RollingUpdate (default)
        * Recreate (delete all, then create new)
- It honors parameters like:
        * maxUnavailable: how many old Pods can be down during update
        * maxSurge: how many extra new Pods can be created temporarily

💬 Example:
```
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 2
```
🧠 Behind the Scenes:
- Uses ReplicaSet controller to handle pod scaling
- Ensures new Pods are Ready before proceeding to next batch
- If a failure is detected, rollout can be paused or rolled back

❓ Follow-up Questions:
Q: How does the controller detect a rollout failure?
A: Based on readiness probes and Pod status, it monitors availability.
Q: How do you pause or undo a rollout?
A:
```
kubectl rollout pause deployment <name>
kubectl rollout undo deployment <name>
```

-----------------------------------------------------------------------------------------------------------------------------------------------------------

3- 🧨  What happens if the ReplicaSet controller fails?
If the ReplicaSet controller fails:
- No new Pods will be created or deleted for that ReplicaSet.
- The system becomes stale — current state will not be reconciled with desired state.

🧠 Important Notes:
- Existing Pods will keep running, since kubelet manages them on the nodes.
- Other controllers (e.g., Deployment controller) may still function.
-Once the controller comes back up, it resumes reconciliation using event-based watches and local cache.

❓ Follow-up Questions:
Q: Is there a backup ReplicaSet controller?
A: No, but the controller-manager process is usually replicated (in HA setups) to prevent this issue.
Q: Can it recover state?
A: Yes. Controllers are stateless; desired/actual state is stored in etcd, so they recover automatically.
-----------------------------------------------------------------------------------------------------------------------------------------------------------

4-🔁  What is a reconciliation loop, and how frequently do controllers run it?
A reconciliation loop is a logic loop where a controller:
"Continuously compares the desired state (from etcd) with the actual state (from the cluster), and makes changes to match them."

⚙️ Workflow:
-Controller watches a Kubernetes object (e.g., Deployment).
- Gets an event (add/update/delete).
- Fetches the current state (e.g., 1 Pod running, 3 desired).
- Takes action (e.g., create 2 more Pods).
- Waits for the next change/event.

⏱️ How frequently?
- Based on events (watch API)
- Also runs periodic re-sync (every few minutes, e.g., 5 mins) to handle missed events.

❓ Follow-up Questions:
Q: Is reconciliation continuous?
A: Yes, it’s reactive (event-driven) + proactive (periodic).

-----------------------------------------------------------------------------------------------------------------------------------------------------------

5 - ⚔️How does the controller handle conflicts between actual and desired state?
The controller overwrites the actual state to match the desired state defined in etcd (via Deployment, etc.).

⚙️ Workflow Example:
- Desired state: replicas: 3
- Actual state: only 1 Pod running (maybe 2 were deleted manually)
- Controller detects mismatch
- It creates 2 new Pods to fix it

🛡️ Conflict Handling:
- Controllers don’t care why the actual state is different — they just enforce the desired state.
- Manual changes not reflected in desired spec will be reverted.
Example: You manually delete a Pod — controller will recreate it.

❓ Follow-up Questions:
Q: What if someone manually changes a pod created by a ReplicaSet?
A: The controller will revert or replace it.

# =========================================================================================================================================================
🌩️ Cloud Controller Manager (if on cloud like AWS/GCP)
-----------------------------------------------------------------------------------------------------------------------------------------------------------

☁️ 1. What are the components of the Cloud Controller Manager?
CCM splits out cloud-specific logic from the main Kubernetes Controller Manager. It contains controllers that interact with the cloud provider’s APIs.

🧱 Key Components of CCM:
| Component              | Function                                              |
| ---------------------- | ----------------------------------------------------- |
| **Node Controller**    | Updates node status (e.g., shutdown, delete in cloud) |
| **Route Controller**   | Sets up network routes between nodes (VPC, etc.)      |
| **Service Controller** | Creates and manages LoadBalancers for Services        |
| **Volume Controller**  | Manages cloud storage (EBS, PD, etc.) for PVCs        |


🧠 Behind the scenes:
- Watches API objects like Node, Service, PersistentVolume
- Calls cloud provider APIs (e.g., AWS SDK) to provision infra

-----------------------------------------------------------------------------------------------------------------------------------------------------------

🔄 2. How does CCM manage Load Balancers and Persistent Volumes?
✅ Workflow for LoadBalancer Service:
- You create a Service with type: LoadBalancer.
- The Service Controller in CCM sees it.
- It calls the cloud provider API (e.g., AWS ELB or GCP LB).
- A load balancer is created and its IP is added to the Service status.

✅ Workflow for PersistentVolume:
- You create a PVC.
- The Volume Controller:
    * Talks to the cloud provider (e.g., AWS EBS, GCP PD)
    * Provisions a disk
- A PersistentVolume is created and bound to the PVC.

❓ Follow-up:
Q: What is in-tree vs out-of-tree?
A: CCM is part of the move to "out-of-tree" cloud providers (decouples cloud logic from Kubernetes core).

-----------------------------------------------------------------------------------------------------------------------------------------------------------

🌐 3. How does the CCM interact with cloud APIs?
 It uses cloud provider SDKs (e.g., AWS SDK, GCP SDK) to make API calls for:
- LoadBalancer creation/deletion
- Disk creation/attachment
- Route table setup
- Node lifecycle management

📦 Example:
```
cloudProvider.CreateLoadBalancer(service)
cloudProvider.AttachDisk(nodeName, volumeID)
```
➡️ This logic is implemented in cloud-specific CCM plugins (e.g., AWS CCM, GCP CCM).

-----------------------------------------------------------------------------------------------------------------------------------------------------------

💥 4. What happens if CCM fails or loses cloud API access?
❌ Impacts:
 - New LoadBalancers or Volumes will not be provisioned.
 - Node status updates (e.g., instance shutdowns) may be missed.
 - Routes may not be created.
➡️ Existing Pods and Volumes still work (managed by kubelet/kube-proxy).

🔁 Recovery:
When CCM restarts or API access is restored, it resumes reconciliation from etcd state.

❓ Follow-up:
Q: Will pods crash?
A: No, unless they depend on a resource that CCM failed to provision (like a PVC).

-----------------------------------------------------------------------------------------------------------------------------------------------------------

🔄 5. How is CCM different from the regular Controller Manager?
| Aspect                  | Controller Manager                   | Cloud Controller Manager                        |
| ----------------------- | ------------------------------------ | ----------------------------------------------- |
| Purpose                 | Handles generic Kubernetes logic     | Handles **cloud-specific** infrastructure logic |
| Runs Controllers For    | Pods, ReplicaSets, Deployments, etc. | Nodes, LoadBalancers, Volumes, Routes           |
| Involved in Scheduling? | No                                   | No                                              |
| Talks to Cloud APIs?    | ❌ No                                 | ✅ Yes                                           |


✂️ Why Separate?
- Decouples cloud code from Kubernetes core.
- Enables out-of-tree providers (each cloud vendor can ship own CCM).


# =========================================================================================================================================================
🔁 Pod Lifecycle/Startup
-----------------------------------------------------------------------------------------------------------------------------------------------------------

🔁 1. What is the complete lifecycle of a Pod from scheduling to termination?
Pending → Scheduled → Running → Succeeded/Failed → Terminating

🧬 Workflow:
- Pending: Pod created in API Server, but not yet scheduled to a node.
- Scheduled: Scheduler assigns it to a node.
- Running: kubelet pulls images, runs initContainers, then containers.
- Succeeded / Failed: Completed based on container exit codes.
- Terminating: 
        * kubectl delete pod or controller scale-down.
        * PreStop hooks and graceful shutdown handled.

🔍 Follow-up:
Q: Who handles Pod creation?
A: kubelet on the target node fetches the Pod spec and manages container lifecycle via CRI.

-----------------------------------------------------------------------------------------------------------------------------------------------------------

⚙️ 2. How do initContainers influence startup order?
initContainers run sequentially before the main containers start. Each initContainer must complete successfully before the next one runs. If one fails → retried according to the restartPolicy (usually Always or OnFailure).

🧠 Use Case:
- Wait for DB schema to be ready
-Set permissions, download configs, pre-warm cache

🔄 Workflow:
```
initContainers:
  - name: init-db
containers:
  - name: app
```
➡️ init-db runs and exits → then app starts.

-----------------------------------------------------------------------------------------------------------------------------------------------------------

❤️‍🔥 3. What are the differences between liveness, readiness, and startup probes?
| Probe Type    | Purpose                                   | Effect when Fails                         |
| ------------- | ----------------------------------------- | ----------------------------------------- |
| **Liveness**  | Checks if container is **alive**          | Container is **restarted**                |
| **Readiness** | Checks if container is **ready**          | Pod is **removed from Service endpoints** |
| **Startup**   | Checks if container **finished starting** | Delays liveness & readiness checks        |

✅ Real-World Tip:
Use startupProbe for slow apps (e.g., Spring Boot) to avoid premature restarts.

🔍 Follow-up:
Q: What happens if readiness fails but liveness is OK?
A: Pod stays running, but won't receive traffic from Services.

-----------------------------------------------------------------------------------------------------------------------------------------------------------

🧨 4. What happens when a probe fails repeatedly?
✅ Liveness probe → Container is restarted by kubelet.
✅ Readiness probe → Pod is marked unready, removed from Service endpoints.
✅ Startup probe → If fails, treated like liveness → container restart.

Parameters that control behavior:
```
failureThreshold: 3
periodSeconds: 10
```
Means: after 3 failed checks (1 every 10s) = action taken

-----------------------------------------------------------------------------------------------------------------------------------------------------------

📦 5. What is the order of container startup in a multi-container Pod?
All app containers start in parallel (not sequential).  Init containers always run before app containers, in order.

🧠 Why?
Because containers in a pod share the same network/storage namespace — often for tightly coupled processes (e.g., logging agent + web app).

Follow-up:
Q: Can we make regular containers start in order?
A: Not directly — you need to manage it within your code or use initContainers.


# =========================================================================================================================================================
✨ Bonus – Troubleshooting Components
-----------------------------------------------------------------------------------------------------------------------------------------------------------

🟠 1. A pod is in Pending state for 10 minutes. How would you investigate?
✅ Root Cause Possibilities:
- No available nodes matching pod’s resource requests or constraints
- Unschedulable due to taints, affinity rules, or node selectors
- Image pull issues (if imagePullPolicy blocks scheduling)

🔍 Workflow to Debug:
```
kubectl describe pod <pod-name>  # Look under "Events"
kubectl get events --sort-by='.lastTimestamp'
```
🔍 Check for:
- “0/3 nodes available” → Scheduler can’t place Pod
- Insufficient memory/CPU → Node doesn't meet resource requests
- Taints/Tolerations mismatch
- PVC not bound (storage issue)

❓ Follow-up:
"What if it’s a DaemonSet pod?" → Check if host ports, node selectors, or tolerations block scheduling

-----------------------------------------------------------------------------------------------------------------------------------------------------------
🚫 2. Kubelet logs show container 'created' but not 'started' — what could be wrong?
✅ Probable Issues:
- Image pulled but container failed during startup
- Failing liveness/startup probe
- Permission issue (e.g., missing volume mount)
- Waiting for initContainers to finish

🧪 Workflow:
```
journalctl -u kubelet -f  # Live logs
kubectl describe pod <pod>
kubectl logs <pod-name> -c <container>
```
- Look for crash loops, probe failures
- Check container runtime logs (e.g., containerd)

❓ Follow-up:
"How does kubelet decide when to restart a container?" → Based on the pod's restartPolicy and probe failures

-----------------------------------------------------------------------------------------------------------------------------------------------------------
❌ 3. API Server is unresponsive, but etcd is running — what’s your next step?
✅ Check if API Server is able to reach etcd
Even if etcd is running, API Server may have:
- Bad etcd certs
- Network/DNS issue
- Resource exhaustion

🧪 Workflow:
- SSH into master node
- Check API server logs:
``` kubectl -n kube-system logs <kube-apiserver-pod> ```
Look for errors like:
     - "connection refused to etcd" , - "certificate expired"
- Check API Server health endpoint: ``` kubectl get --raw '/healthz' ```

❓ Follow-up:
"If etcd is healthy but API server isn't, is the cluster usable?"
→ No — API is the gateway to everything

-----------------------------------------------------------------------------------------------------------------------------------------------------------

🟥 4. A node reports NotReady — what logs or components would you check?
✅ Root Causes:
- kubelet is down or misconfigured
- Node network issue
- Docker/containerd failure
- High resource usage

🧪 Workflow:
- On the node:
```
systemctl status kubelet
journalctl -u kubelet -xe
```
- Check:   kubelet logs ,  container runtime status:
```
crictl info
systemctl status containerd
```
- Run:
```
kubectl describe node <node-name>
```
Check Conditions: block — it shows memory, disk pressure, PID pressure

❓ Follow-up:
“What if the kubelet is running but node is still NotReady?”
→ Check if it can reach the control plane and DNS works

-----------------------------------------------------------------------------------------------------------------------------------------------------------

📏 5. Controller manager is reporting resource quota violations — how would you debug?
✅ ResourceQuota limits the total CPU, memory, PVCs, etc., in a namespace.
🧪 Workflow:
```
kubectl describe resourcequota -n <namespace>
```
Look for usage vs hard limits

Then: ``` kubectl get pods -n <namespace> -o wide```
Identify which pods are consuming excess resources

⚠️ Example Scenario:
- Quota: 2 CPU, 4Gi memory
- Pod requests 3 CPU → pod will stay Pending with quota violation

❓ Follow-up:
“What if quota is correct but pod still fails?”
→ Might be a limitRange issue or request vs limit mismatch

-----------------------------------------------------------------------------------------------------------------------------------------------------------




