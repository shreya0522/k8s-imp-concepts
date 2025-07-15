# =========================================================================================================================================================
# 🏗️ 1. Basic
------------------------------------------------------------------------------------------------------------------------------------------------------------
1. What is Kubernetes and how does it work?

Kubernetes is a tool that helps you run and manage many containers across multiple servers. It handles scaling, deployment, and restarting containers automatically.

-----------------------------------------------------------------

2. What is the difference between a Pod, ReplicaSet, Deployment, and StatefulSet?

Pod: Smallest unit in Kubernetes. It runs one or more containers together.
ReplicaSet: Ensures a set number of Pods are always running.
Deployment: Manages ReplicaSets and updates Pods easily.
StatefulSet: Like Deployment but used for apps that need stable names or storage (like databases).

-----------------------------------------------------------------

3. Why are Pods ephemeral?

Pods are temporary—they can be deleted or replaced at any time due to scaling or failures. That’s why they are not meant to store permanent data.

-----------------------------------------------------------------

4. What is the role of the Kubelet?

Kubelet runs on each node. It makes sure the containers in the Pods are running and healthy.

-----------------------------------------------------------------

5. Explain the difference between kubectl apply and kubectl create.

kubectl create: Used to create a resource for the first time.
kubectl apply: Used to create or update a resource. Good for making changes.

-----------------------------------------------------------------

6. What is a Namespace and why is it used?

A Namespace is a way to divide cluster resources between users or teams. It helps organize and manage large clusters.

-----------------------------------------------------------------

7. Why 2 containers are not recommened in a pod ?

Think of a Kubernetes Pod like a “single car” on a train track—it’s the smallest deployable unit, with one address, one schedule, and one set of resources. Packing two full-blown applications into that same car usually leads to headaches:

- Tied lifecycles: 
Both containers start, stop and restart together. If one crashes, Kubernetes will kill and recreate both—even if the other was doing perfectly fine.

- No independent scaling: 
You can’t say “I need three of Container A but only one of Container B.” The Pod is your scaling unit, so you end up over- or under-provisioning one just to scale the other.

- Resource contention:
CPU and memory limits apply to the whole Pod. Two similarly heavy processes end up fighting over that single quota.

- Complexity in updates: 
Rolling out a new version of one container forces you to redeploy the entire Pod, risking downtime or side effects for the other container.

- Networking aliasing: 
Containers in a Pod share the same network namespace (IP, port space). If both need port 80, you’ll clash or have to build convoluted workarounds.

When you do bundle multiple containers it’s for very narrow “sidecar” or “adapter” roles—things like:
- log-collector alongside your app
- A proxy that injects TLS certs
- A filesystem sync agent



# =========================================================================================================================================================
# 🏗️ 2. Architecture & Components
------------------------------------------------------------------------------------------------------------------------------------------------------------

🧱 1. Describe the Kubernetes architecture.
```
 Kubernetes has a Control Plane and Worker Nodes. The Control Plane manages the cluster.
 Worker Nodes run the actual applications in Pods.
Key Components:

* Control Plane: API Server, Scheduler, Controller Manager, etcd
* Node Components: Kubelet, Kube Proxy, Container Runtime

Follow-up Interview Point:
- “How do these components talk to each other?”
- Answer: Mostly through the Kube API Server, which acts as the central communication point.
```
-----------------------------------------------------------------

🧩 2. What are the components of the Control Plane?
```
(CASE)
- Controller Manager – Handles background tasks (like scaling, restarts). make sure cluster stays in desired state
- API Server – Frontend for the cluster, all requests go through this.
- Scheduler – Places new Pods on suitable Nodes.
- Etcd – Stores the cluster’s configuration and state (key-value store).

Follow-up Interview Point:
--------------------------
👉 “Is API Server stateless?”
* Yes. API Server is stateless and can be horizontally scaled.

#### Behind-the-scenes:
> Controllers watch the cluster state via API Server, then make changes by writing back to it. ETCD stores all this information.
```

-----------------------------------------------------------------

📦 3. How does the Scheduler decide where to place a pod?
```
Simple Answer:
- The Scheduler looks at available nodes and decides based on:
- Resource availability (CPU, RAM)
- Node labels, taints/tolerations
- Affinity/anti-affinity rules

Follow-up Interview Point:
- “Can we influence the scheduler decision?”
- Yes, using nodeSelector, affinity rules, and taints/tolerations.

Behind-the-scenes:
- Scheduler gets a list of candidate nodes from the API Server, applies filters, then scores them, and finally selects the best match.
```

-----------------------------------------------------------------

4. What is the role of etcd?
```
ETCD is a distributed key-value store that stores all cluster data — like config, status, secrets, etc.

Follow-up Interview Point:
-----------------------------

👉 “What happens if etcd fails?”
Cluster can’t function fully—no scheduling or state management.
Important: Take etcd backups regularly—losing etcd = losing cluster state.

Behind-the-scenes: 
-------------------
- All changes (like pod creation, scaling) are written to etcd via API Server, and controllers act based on this data.
```
-----------------------------------------------------------------

5. Explain the Controller Manager and what controllers it runs.
```
Controller Manager runs various controllers that make sure the cluster stays in the desired state.
Common Controllers:
-------------------
- Replication Controller / ReplicaSet
- Node Controller (monitors node health)
- Job Controller
- Endpoint Controller

Follow-up Interview Point:
-----------------------------
👉 “What happens when a node goes down?”
Node Controller notices and marks it as NotReady, and other controllers may restart pods elsewhere.

Behind-the-scenes:
-----------------
Each controller watches etcd (via API Server), compares current vs. desired state, and makes changes.
```

-----------------------------------------------------------------

6. What is the Container Runtime Interface (CRI)?
```
The CRI (Container Runtime Interface) is a standard API. It defines how Kubernetes talks to container runtimes like containerd, CRI-O, etc. This means Kubernetes doesn’t care who makes the runtime — as long as it follows the CRI standard, Kubernetes can talk to it.
Kubernetes talks to the container runtime (like containerd) through CRI. This standardizes how Kubernetes tells the runtime to:
- Start containers
- Stop containers
- Pull images
- Check container status

Follow-up Interview Point:
--------------------------

👉 “Is Docker a runtime now?”
Docker support was deprecated in Kubernetes 1.20+. Use containerd now.
What happened to Docker in Kubernetes?
Kubernetes deprecated Docker as a runtime starting from v1.20. It doesn’t mean Docker images won’t run — you can still build images using Docker, but Kubernetes uses containerd under the hood to run them.

Important Point: 
CRI ensures flexibility—you can plug in different runtimes without changing Kubernetes core. The Kubelet uses CRI to start/stop containers on a node.

 Behind-the-scenes:
Kubelet makes calls to containerd using the CRI to manage containers in Pods.
containerd runs a CRI plugin (containerd-shim) that implements those gRPC endpoints. So whenever the kubelet issues a CRI call, containerd’s CRI plugin receives it and uses its own internals (and ultimately an OCI runtime like runc) to actually create, start, stop, or remove the containers in your Pods
```
-----------------------------------------------------------------

7. How does the Kubelet use CRI to start/stop containers?
```
What is the Kubelet?
----------------------
The **Kubelet** is the agent that runs on every **worker node** in Kubernetes. Its job is to:
* Talk to the Kubernetes API server
* Ensure the containers (pods) defined in the API actually run on the node
* Report the node's status back to the control plane

What is the CRI?
-------------------
CRI = Container Runtime Interface.It’s a **standard API** that the Kubelet uses to **talk to the container runtime** on the node — such as `containerd` or `CRI-O`.

So, how does the Kubelet use CRI to start/stop containers?
----------------------------------------------------------
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
--------------------------
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

Summary
----------------
| Term        | Meaning                                                                                                                                                                                                          |
| ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sandbox** | The Pod-level sandbox that the container runtime (via the CRI) creates for network, PID, IPC namespaces, cgroups, etc. Kubernetes creates exactly one sandbox per Pod—no matter how many containers it contains. |
| **Parsing** | The kubelet’s process of reading and interpreting a Pod’s spec (the YAML/JSON) from the API server so it knows what containers, volumes, ports, and settings to bring up.                                        |
                                |

```

-----------------------------------------------------------------


# ======================================================================================

# 🧠 Kubelet – Node Agent

-----------------------------------------------------------------

## 1. How does the Kubelet detect changes in PodSpecs from the API server?

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

## 2. What is a Pod Sandbox, and how does the Kubelet manage it?

Answer: A Pod Sandbox is an isolated environment that Kubernetes sets up for every pod.
It handles things like network setup, hostname, IP address, and process isolation, so that containers inside the pod can communicate with each other, but are isolated from containers in other pods.
Kubernetes creates a separate sandbox for each pod, ensuring one pod’s environment is fully isolated from another.

Example: 
```
🎯 Pod 1: nginx-pod
Contains 2 containers: nginx , sidecar-logger

- Kubernetes creates a Pod Sandbox:
- Gives it one IP address (e.g., 10.244.0.5)
- Sets up network rules, hostname like nginx-pod
- Isolates its processes and filesystem access from other pods

➡️ Now both containers inside nginx-pod can:
- Talk to each other over localhost
- Share the same IP and port space
- See the same logs, mount same volumes, etc.

🎯 Pod 2: api-pod
Contains 1 container: python-api
- Kubernetes creates a separate Pod Sandbox:
- Gives it a different IP (e.g., 10.244.0.6)
- Own hostname like api-pod
- Separate process & network space

➡️ This pod is fully isolated from nginx-pod
- Even if they're on the same node
- It cannot access ports or files inside nginx-pod unless exposed
```

🧠 Behind the Scenes:
---------------------
The Kubelet is the agent running on each Kubernetes node.
When it needs to start a pod, it doesn’t directly run containers — it uses the CRI (Container Runtime Interface) to talk to the container runtime (like containerd or CRI-O).

```
🔁 KUBELET:
-------------------------------------
- Kubelet reads the Pod spec from the API server.
- It makes a "CRI call" like: " RunPodSandbox "  to the "container runtime" (e.g., containerd).
- This call:
   * Creates the network namespace
   * Assigns IP address
   * Sets up hostname, cgroups, and security context
- Once the sandbox is ready, Kubelet makes CRI calls like:
   * CreateContainer
   * StartContainer
to launch each container inside that sandbox.
```

Follow-up:
👉 “How do you check if the Pod sandbox is healthy?”
Check kubectl describe pod → under Events → look for FailedSandbox.

-------------------------------------------------------------------------------------------------

## 3. How does the Kubelet decide when to restart a failed container?

Answer: It follows the Pod's restartPolicy (Always, OnFailure, or Never).

#### Workflow:

- Kubelet watches container status via CRI (containerd).
- On failure (non-zero exit code):
  * If restartPolicy: Always → restart it immediately.
  * If OnFailure → restart only on non-zero exit.
  * If Never → don’t restart.

#### Important:
Restart logic is Kubelet’s job, not controlled by the container runtime.

#### Remember:
> The Kubelet watches pod specs using the Kubernetes Watch API mechanism — it doesn’t poll repeatedly, but instead maintains a long-lived HTTP connection to the API Server.

> For container status, The Kubelet "polls" the container runtime (e.g. containerd, CRI-O) using the CRI interface to : 
> Check if containers are still running
> Get the status of each container (e.g. Running, Exited, CrashLoopBackOff)
> Fetch logs, exit codes, and resource usage

#### Follow-up:

* Where is the backoff delay handled?
* Kubelet applies exponential backoff to avoid rapid restarts.

* How kubelet watches containers ?

1-Kubelet communicates with the container runtime (e.g., containerd) through the CRI gRPC API.
2-It uses CRI methods like:
    * ListContainers()  *ContainerStatus()   *ListPodSandbox()
3- Kubelet does not directly "watch" container state like the API Server watch mechanism. Instead, it polls container runtime status regularly through CRI calls.
4- Based on container status (e.g., exit code, health), Kubelet takes actions like:
    *Restarting containers *Reporting to the API Server  *Updating Pod status

-------------------------------------------------------------------------------------------------

## ⚠️ 4. What happens if the container runtime is down, but the Kubelet is running?

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

## 5. How does the Kubelet determine resource availability for scheduling pods?

> The **Kubelet doesn’t schedule pods** — that’s the Scheduler’s job. However, the **Kubelet reports node resource status** to the **API Server**, which the Scheduler uses to make placement decisions.

 ✅ Workflow:

* **Kubelet uses `cAdvisor`** to collect **container-level metrics**, such as:

  * CPU and memory usage per container
  * Disk and network stats
* **Kubelet gathers node-level metrics** from sources like:

  * `/proc`, sysfs, cgroups, and local node introspection
* These metrics are packaged into **NodeStatus** updates and sent to the **API Server**.
* The **Scheduler** uses this data to understand:

  * Current node usage
  * Available capacity for new Pods

---

📌 Important:

> The Kubelet also **enforces resource limits and requests** defined in the Pod using **cgroups**. It can throttle, OOM-kill, or deny pods based on resource pressure.

---

### 🔁 Follow-up:

> **👉 Can the Kubelet reject a Pod?**
> ✅ Yes — if the Pod's **requested resources** exceed what’s available on the node, the **Kubelet can reject or delay** the Pod.
> Especially true during:

* CPU pressure
* Memory pressure
* Disk pressure (via eviction policies)

-----------------------------------------------------------------------------

6. **Do we need to define resource limits before a pod is scheduled to a node?**

🔹 **No — it's not mandatory**, but **highly recommended**.

* If **resource `requests`** are specified:
  ➤ The **Kube-scheduler** uses them to **determine where to place** the pod.

* If **resource `limits`** are specified:
  ➤ The **Kubelet enforces** them using **cgroups** (throttling or OOM-killing if exceeded).

* If no requests are defined:
  ➤ The pod gets scheduled, but it's considered **Best-Effort**, and **may be evicted** under resource pressure.

---

### 🧠 Key Distinction:

| Field      | Used By          | Purpose                             |
| ---------- | ---------------- | ----------------------------------- |
| `requests` | **Scheduler**    | For pod placement decisions         |
| `limits`   | **Kubelet / OS** | For runtime enforcement via cgroups |

---

## 🎯 How Interviewers May Ask This (and twist it):

 🔸 Basic Questions:

* "What happens if you don’t set resource requests or limits for a pod?"
* "Who uses resource requests — the Scheduler or the Kubelet?"

 🔸 Twisted/Advanced Questions:

* ❓ *“Can a pod with no limits get scheduled? What are the risks?”*

  * Yes, it will be treated as **BestEffort** and **can starve other pods or get evicted**.

* ❓ *“If I set limits but not requests, what happens?”*

  * Scheduler **assumes request = 0** (i.e., BestEffort), but Kubelet still **enforces the limit**. This may lead to poor scheduling decisions.

* ❓ *“What if a pod’s request is higher than any node’s allocatable memory?”*

  * It **won’t be scheduled** — stays in **Pending**.

* ❓ *“How do resource limits help with multi-tenancy?”*

  * They prevent **noisy neighbor problems**, ensure **fair usage**, and allow **QoS classification** (`Guaranteed`, `Burstable`, `BestEffort`).

* ❓ *“Can Kubelet deny a pod even after Scheduler assigns it?”*

  * ✅ Yes — during admission if node conditions are bad (e.g., memory pressure) or insufficient resources.

---

💡 Pro Tip to Say in Interview:

> “I always define both resource **requests and limits** for production pods — requests help the **Scheduler**, and limits ensure **runtime enforcement** by the Kubelet via cgroups.”

---

# =====================================================================================
#   🌐 Kube-Proxy – Networking
----------------------------------------------------------------------------------------

# 1.  What kube proxy is ? 

 It's a networking component that runs on each node in a Kubernetes cluster.Its job is to make the Service IPs (like ClusterIP) actually work by sending traffic to the **right backend Pods**.

##### What kube-proxy does:

**Step 1: Watches for changes:**
kube-proxy is watching the Kubernetes API Server** for any changes in:
Service objects
Endpoints objects

**Step 2: Creates routing rules**
kube-proxy reads the Service IP and backend Pod IPs.
Based on your OS and kube-proxy settings, it sets up either:
iptables rules (default in most clusters), or
IPVS rules (for better performance).


-----------------------------------------------------------------------------------------------------------------------------------------------------------

##### A. How does kube-proxy manage Service IPs and route traffic to pods?


Answer: What kube-proxy is:
* It's a **networking component** that runs on **each node** in a Kubernetes cluster.
* Its job is to **make the Service IPs (like ClusterIP) actually work** by sending traffic to the right backend Pods.


##### What Happens When a Service is Created in Kubernetes (Rewritten & Corrected)

#### **1. You define a Service**

* Type: `ClusterIP`, `NodePort`, etc.
* You include a **label selector** like `app=web`.

```yaml
spec:
  selector:
    app: web
```

---

#### **2. Kubernetes (NOT CNI) assigns a ClusterIP**

* The **API Server allocates** a **virtual IP** (e.g., `10.0.0.5`) from the **Service IP range**.
* 📌 **This is not handled by CNI** — CNI assigns **Pod IPs**, not Service IPs.
* This virtual IP becomes the **ClusterIP** used to access the service.

---

#### **3. Endpoint Controller creates an Endpoints object**

* It watches Services and Pods.
* If any Pods match the Service’s label selector, it creates an `Endpoints` object:

```yaml
subsets:
  - addresses:
      - ip: 10.244.1.7
      - ip: 10.244.2.3
    ports:
      - port: 8080
```

📌 This object maps **Service → actual Pod IPs and target ports**.

---

#### **4. kube-proxy watches Endpoints and sets up routing**

* kube-proxy runs on each node.
* It watches for changes in:

  * `Service` objects
  * `Endpoints` objects
* Depending on the mode:

  * **iptables**: writes DNAT rules
  * **IPVS**: adds backend servers to kernel’s load balancer
* These rules say:
  ➤ "If traffic comes to `10.0.0.5:80`, forward it to one of the real pod IPs."

---

#### **5. Now traffic to the Service IP is load-balanced**

* From **any pod inside the cluster**, if you run:

```bash
curl http://10.0.0.5
```

* The packet is routed **by the Linux kernel**, not kube-proxy directly.
* It lands on either:

  * `10.244.1.7:8080`
  * `10.244.2.3:8080`
* Based on round-robin or IPVS scheduling algorithm.

---

##### ✅ Final Notes:

* **CNI only assigns Pod IPs**, not Service IPs.
* **Service IP (ClusterIP)** is managed internally by the Kubernetes control plane.
* **kube-proxy** only sets up routing once the **Endpoints** object is populated by the **Endpoint Controller**.


``````

WHAT IS AN ENDPOINT OVJECT IN K8S?  
- When you create a **Service** in Kubernetes (e.g., `my-service`), it uses a **label selector** to pick a set of pods that match.
- Once those pods are selected, Kubernetes **automatically creates an internal object called an `Endpoints` object** (created by endpoint controller, endpoint object service bnne k baad he bnta h ). This object holds the **real IP addresses and ports** of the matching pods.

- 💡 Think of it like this:
> The **Service** has a **virtual IP** (`10.0.0.5`)
> The **Endpoints** object maps that virtual IP to actual **Pod IPs** and **ports** behind the scenes.

Example:

Let’s say you define a Service like this:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
```

You have 2 running pods with the label `app=web`:

* Pod 1 IP: `10.244.1.7`
* Pod 2 IP: `10.244.2.3`

🔁 Kubernetes will create an **Endpoints** object like:

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: my-service
subsets:
  - addresses:
      - ip: 10.244.1.7
      - ip: 10.244.2.3
    ports:
      - port: 8080
```

Now when traffic is sent to the **Service IP (`10.0.0.5`) on port 80**, kube-proxy knows to **forward it to one of the pods on port 8080** using this mapping.

---

🎯 Summary
| Component      | Purpose                                  |
| -------------- | ---------------------------------------- |
| **Service**    | Has a stable virtual IP (ClusterIP)      |
| **Endpoints**  | Maps the service to real Pod IPs + ports |
| **kube-proxy** | Handles traffic redirection using these  |

-----------------------------------------------------------------------------------------------

IS THE ENDPOINT OBEJECT CREATED ONLY WHEN SERVICE IS CREATED?
> **An `Endpoints` object is only created if a Service exists.**
> The `Endpoints` object **is created *after*** the Service is created — and **only if** the Service has:
     * A valid selector, and
     * There are Pods matching that selector

🔁 Behind the Scenes — Full Logic:

| Scenario                                        | Does `Endpoints` get created? | Why?                                                                              |
| ----------------------------------------------- | ----------------------------- | --------------------------------------------------------------------------------- |
| ✅ Service is created + selector matches Pods    | ✅ Yes                         | Endpoint Controller creates the `Endpoints` object listing matched Pod IPs        |
| ❌ Service is created but no Pods match selector | ✅ Yes (empty)                 | Endpoint Controller still creates the object, but with **no IPs**                 |
| ❌ No Service exists                             | ❌ No                          | There's **no reason** to create an `Endpoints` object — they are Service-specific |


🧠 Key Rule:

> 📌 `Endpoints` (or `EndpointSlice`) **exist only for Services**.
> They are **automatically created/updated by the Endpoint Controller** based on the Service's selector and the current set of matching Pods.

🧪 Example:

1. You define a Service with selector `app=api`:

```yaml
spec:
  selector:
    app: api
```

2. No Pods exist with `app=api` yet.

✅ Kubernetes will still create this:

```yaml
kind: Endpoints
metadata:
  name: my-service
subsets: []    # empty — no Pods match
```

3. Later, a Pod with `app=api` is created

🟢 The Endpoint Controller detects it and **updates the Endpoints object**:

```yaml
subsets:
  - addresses:
      - ip: 10.244.3.5
    ports:
      - port: 8080
```

✅ Conclusion:

> ✔️ **Endpoints object is tied to the Service**.
> ❌ It does **not exist independently**.
> 🔄 It is dynamically created and updated **only if a Service exists**, based on its **selector** and matching **Pods**.


``````
    

### What kube-proxy does:

> **Step 1: Watches for changes:** 
* kube-proxy is watching the Kubernetes API Server** for any changes in:
* Service objects
* Endpoints objects

> **Step 2: Creates routing rules**
 * kube-proxy reads the Service IP and backend Pod IPs.
 * Based on your OS and kube-proxy settings, it sets up either:
 * iptables rules (default in most clusters), or
 * IPVS rules (for better performance).

 These rules say: 
 > "If anyone tries to access 10.0.0.5:80, forward the request to one of these real Pods: 10.244.1.7:80 or 10.244.2.3:80."

###  **What happens during a real request:?**
 

 * Let’s say a pod in the cluster does: ``` curl http://10.0.0.5:80``` (service IP)
 * Behind the scenes:
 1. The **Linux networking stack** hits an **iptables rule** created by kube-proxy.
 2. This rule **redirects the request to one of the Pod IPs** randomly (load-balancing).
 3. The request **never even reaches kube-proxy as a process** — Linux handles it directly using the rules kube-proxy set earlier.  (IMP)
 * So kube-proxy **programs the rules**, but **doesn’t touch every packet**.
 
 **How kube-proxy load-balances traffic:?**
 -------------------------------------------
1. In iptables mode: It uses round-robin to forward to different Pods.
2. In IPVS mode**: It supports advanced load balancing algorithms like:
                   * Round Robin   * Least Connections   * Source Hashing
3. You can choose IPVS by passing: ``` --proxy-mode=ipvs ```


Bonus: What are ClusterIP and Endpoints?
* **ClusterIP**: is just a **virtual IP** — not tied to any real interface.  It only works inside the cluster.
* The **Endpoints object** holds the **real Pod IPs** to which traffic should go.

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

SUMMARY 
``````
When a Service (like ClusterIP) is created in Kubernetes:
1. Kubernetes assigns a virtual IP (e.g., 10.0.0.2) to the Service — this is the ClusterIP.
2. The Service uses a label selector to select matching Pods.
3. Kubernetes automatically creates an Endpoints object, which lists the real Pod IPs and ports (e.g., 10.244.1.7:8080, 10.244.2.3:8080).
4. The kube-proxy watches both Services and Endpoints and, based on the current networking mode:
  > 🧱 iptables mode: kube-proxy writes iptables rules
  > ipvs mode: kube-proxy writes IPVS routing table entries

After that, the Linux kernel handles the actual packet forwarding — not kube-proxy.

🧠 So in short:
kube-proxy sets up the routing logic, but it does not handle the traffic itself.
Once the rules are in place (via iptables or IPVS), the Linux kernel does all the heavy lifting of routing traffic from the Service IP to one of the Pod IPs
``````


------------------------------------------------------------------------------------------------

# 🔁 2. What is the difference between IPtables and IPVS modes in kube-proxy?

Both are proxy modes that kube-proxy can use to manage how traffic is routed from a Service IP to the actual Pod IPs.

🔹 1. IPtable mode – Traditional default
-----------------------------------------

**💡 How it works:**

* Kube-proxy creates IPtables rules (Linux firewall rules). 
* Every time a request hits a Service IP (like 10.0.0.5), the kernel checks the IPtables rules and redirects the packet to one of the backend Pods.
* Workflow:
       Example rule: If traffic is for 10.0.0.5:80, redirect to one of: 10.244.1.7:80  10.244.2.5:80
       Load balancing is done using random or statistic match in iptables.

**📉 Drawbacks:**
   * Performance degrades when there are many Services and Pods because:
   * IPtables is rule-order dependent — it has to process one rule at a time in order.
   * Every change (new Pod/Service) causes full rebuild of the rule chain — time-consuming.

🔹 2. IPVS mode – Advanced and optimized
-----------------------------------------

**💡 How it works:**

*  Uses Linux IPVS (IP Virtual Server), a built-in kernel-level load balancer.
*  kube-proxy programs IPVS rules using the ipvsadm tool or kernel APIs.
*  The kernel then uses a hash table, not a rule chain, for routing — much faster.
* Workflow: 
 Maintains a service table in the kernel:
                ``` IPVS: 10.0.0.5:80 → [10.244.1.7:80, 10.244.2.5:80] ```              
Picks one Pod using one of many algorithms: Round Robin ,Least Connections, Source IP Hash

**🚀 Advantages:**
*  Much faster routing, especially in large clusters (1000s of Services/Pods).
*  Rule updates are incremental — only changes are applied, not full table rewrite.

**🔁 Side-by-side comparison:**
| Feature         | **iptables**                           | **IPVS**                                |
| --------------- | -------------------------------------- | --------------------------------------- |
| **Mode**        | Sequential rule matching (`netfilter`) | Kernel-based virtual server             |
| **Performance** | Slower with scale                      | Faster, especially in large clusters    |
| **Rule Mgmt**   | Full table rebuild on updates          | Only affected entries updated           |
| **Algorithms**  | Only basic round-robin/statistic       | Many (Round Robin, Least Conn, etc.)    |
| **Memory use**  | Lighter, good for small clusters       | Slightly higher but scalable            |
| **Stability**   | Stable, default in most clusters       | Stable but needs IPVS modules installed |

**🧠 Key Point:**
  * Use iptables in small/simple clusters.
  * Prefer IPVS in production or large-scale clusters for: Faster failovers , Scalability , Advanced load balancing

**🛠️ How to enable IPVS mode:** 
         In your kube-proxy config or DaemonSet:
``` 
command:
  - kube-proxy
  - --proxy-mode=ipvs
```
Also ensure the node has IPVS kernel modules:
``` lsmod | grep ip_vs ```

------------------------------------------------------------------------------------------------------------------------------------------------------------

# 🔄 3. How does kube-proxy handle updates to Service or Endpoints objects?

Answer: Kube-proxy watches the Kubernetes API Server for changes to Service and Endpoints objects, and updates its routing rules (iptables or IPVS) accordingly — all this happens automatically and with minimal traffic disruption.

🧠 What triggers an update?
--------------------------

* When something changes like: A new Pod is created or deleted,  A Service selector is updated, A Pod becomes Ready/Unready (so its IP should be added/removed).
* This causes the Endpoints object to change, and that triggers kube-proxy to act.

⚙️ Behind the scenes: Workflow
---------------------------------

1- 🕵️‍♂️ kube-proxy is continuously watching the API Server (using a Watch API). It monitors Service and Endpoints objects.

2- 🔄 If an update is detected:  For example: a Pod is deleted → the Endpoints list is now shorter.

3- 🔍 kube-proxy compares the old state and the new state of these objects.
  - What was added? 
  - What was removed? 
  - What changed?

4- 🛠️ Based on the difference, it updates the networking rules:
   - If using iptables: it rebuilds the relevant rules.
   - If using IPVS: it incrementally updates the internal routing tables.

5- 🚀 These changes take effect immediately, allowing traffic to be routed to the new set of healthy Pod IPs.

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

# 🔄 4. Can kube-proxy be replaced or customized? (e.g., Cilium)

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

# 🧹 5. What happens in kube-proxy if a pod behind a service is deleted?

Answer:  When a Pod is deleted, the corresponding **"Endpoints object"** for the Service is updated, and kube-proxy removes that Pod’s IP from its routing rules — so traffic is no longer routed to the deleted Pod.

⚙️ Workflow (Step-by-step):
--------------------------

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

❓ Follow-up 

Q: "Is there a delay in removal? 
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
# 1- 📡 How does the API server ensure high availability and leader election? How does Kubernetes make sure the API server stays up and available even if one instance goes down?

The API Server is made highly available by running multiple replicas behind a load balancer. The leader election is relevant for certain control plane components (like controller-manager or scheduler), not the API server itself — since all API server replicas are active, not just one.

🧱 High Availability (HA) of API Server: ⚙️ Workflow:
----------------------------------------------------

1- You deploy multiple API server pods or instances (e.g., 3).

2- These are placed behind a load balancer (like AWS ELB or HAProxy).

3- Clients (like kubectl, kubelet, controllers) send requests to the load balancer.

4- The load balancer distributes traffic among healthy API servers.

5- All replicas are stateless — they store state in etcd, not locally.

📌 Important: API servers do not need leader election because they serve requests in parallel.

> 🤝 Leader Election – Applies to Controller Manager and Scheduler
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
--------------------------------------------
Q: Why does the API server not require leader election?

A: Because all API server instances are stateless and can serve traffic concurrently.

Q: What happens if etcd goes down?

A: The API servers will not function fully — they can’t store or retrieve cluster state.

Q: How is the health of API servers checked?

A: The load balancer uses health checks (e.g., /healthz) to route traffic only to healthy servers.

-----------------------------------------------------------------------------------------------------------------------------------------------------------

# 2 - ❓ What are Admission Controllers?

They are checkpoints inside the API server that act like gatekeepers for any create, update, delete requests in Kubernetes.

📦 Imagine this workflow:
--------------------------

> You run:  ```kubectl create pod mypod.yaml```

> Here’s what happens step-by-step:
   * ✅ Authentication – Are you a valid user?
   * ✅ Authorization – Are you allowed to create a Pod?
   * 🛑 Admission Controllers – Should we allow or reject this Pod based on: Rules, Security, Custom policies, Auto-editing (e.g., adding labels)

💾 If allowed → object is saved into etcd (the cluster database).

``````
🧠 Example Scenario:
Let’s say a developer tries to create a Pod without memory limits.

A. ✅ AuthN: The user is authenticated (valid token).
B. ✅ AuthZ: The user is authorized to create Pods.
C. 🛂 Admission Controller kicks in:
* LimitRanger sees no memory limits → rejects the Pod creation with:
"must specify memory limits"
D. Nothing is stored in etcd — the pod never makes it into the cluster.
``````

🔍 Common Admission Controllers:
---------------------------------------

| Controller Name              | What it does                               |
| ---------------------------- | ------------------------------------------ |
| `NamespaceLifecycle`         | Blocks resources in terminating namespaces |
| `LimitRanger`                | Enforces resource limits                   |
| `PodSecurity`                | Blocks insecure Pods                       |
| `MutatingAdmissionWebhook`   | Changes the object (e.g., injects sidecar) |
| `ValidatingAdmissionWebhook` | Validates but doesn't modify               |

-----------------------------------------------------------------------------------------------------------------------------------------------------------

# 3. 🔐 What is Authentication vs Authorization in the API Server?

✅ Short Summary:
--------------------

| Term                       | Purpose                                                   |
| -------------------------- | --------------------------------------------------------- |
| **Authentication (AuthN)** | Proves **who you are** (identity check)                   |
| **Authorization (AuthZ)**  | Decides **what you're allowed to do** (permissions check) |

 ⚙️ Full Request Processing Workflow (Behind the Scenes)
 ------------------------------------------------------------

When a client (like `kubectl`) sends a request to the **Kubernetes API server**, here’s what happens:

### 🧭 Step-by-step:

1. ✅ **Authentication**

   * First, the API server checks the **identity of the client**.
   * It asks:

     > *"Is this certificate/token/password valid?"*
   * If invalid, the request is rejected with **401 Unauthorized**.

2. ✅ **Authorization**

   * If authentication passes, now the server checks:

     > *"Is this user allowed to perform this action on this resource?"*
   * If the action is not permitted, request is rejected with **403 Forbidden**.

3. ✅ **Admission Controllers**

   * If both AuthN and AuthZ pass, the request goes to **Admission Controllers** (for policy checks like quotas, required labels, etc.)

4. ✅ **Persistence**

   * If everything passes, the request is processed and changes (if any) are stored in **etcd** (the backing data store of the cluster).

---

### 🛡️ **Authentication Methods (Who are you?)**

| Method                  | Description                                                          |
| ----------------------- | -------------------------------------------------------------------- |
| **Client Certificates** | Common in kubelet–API Server communication                           |
| **Bearer Tokens**       | ServiceAccounts or static secrets                                    |
| **OIDC (OAuth2)**       | For enterprise SSO integration                                       |
| **Static Files**        | Passwords or tokens from flat files (not recommended for production) |

---

### 🛂 **Authorization Modes (Can you do this?)**

| Mode                         | Description                                                              |
| -----------------------------| ------------------------------------------------------------------------ |
| ✅ **RBAC** (most common)    | Permissions based on roles and bindings (e.g., allow "devs" to get pods) |
| **ABAC (Attribute-Based)**   | JSON policies — deprecated and not dynamic                               |
| **Node Authorization**       | Special mode for kubelet to access only its own resources                |
| **Webhook Authorization**    | External system makes the decision via REST call                         |

> 🚦 The API Server can be configured with **multiple authZ modes enabled**. It evaluates them **in order**, and the **first definitive answer** (Allow or Deny) is used.

---

 ❓ Interview Follow-Up Q\&A
--------------------------------
🔸 Q: *What happens if authentication passes but authorization fails?*
✅ Answer:
> The API server returns **403 Forbidden** — the user is known, but **not allowed** to do the requested action.

---

🔸 Q: *Can multiple authorization modes be used together?*

> Yes. The API Server evaluates them **in the order configured**.
> It uses the **first mode** that returns a decision (`Allow` or `Deny`).
> If all modes **abstain**, the request is denied by default.

🔸 Q: *Who manages RBAC policies?*

> RBAC roles and bindings are managed via Kubernetes YAML or `kubectl`, e.g.:

```bash
kubectl create rolebinding view-pods \
  --role=view \
  --user=alice \
  --namespace=dev
```

✅ Final Analogy:

> Think of **Authentication** as checking your **ID at the door**, and **Authorization** as checking if you have **access to the room you’re trying to enter**.



-----------------------------------------------------------------------------------------------------------------------------------------------------------

# 4. 🚦 How does the API server handle excessive request loads (rate limiting)?

The Kubernetes API Server protects itself from overload by using rate limiting, priority levels, and concurrency controls. This ensures fair usage and prevents any user or component from overwhelming the server.

⚙️ Mechanisms Used:
--------------------

1. Client-side Rate Limiting via kubelet, kubectl, etc.:
 Clients like kubectl or kubelet have built-in limits: Example: default 5 QPS (queries per second) and 10 burst for kubelet. You can configure it:
 ``` kubectl --request-timeout=30s --v=9 ```

2. API Server Priority and Fairness (APF): 
* Since Kubernetes v1.20+, the API server uses Priority and Fairness (APF).
* It divides traffic into **“priority levels”** and **“flows”**, like: 
    ``````> System components (high priority) > user requests (medium)  > Background jobs (low)``````
* 🔁 Each flow gets a fair share of API server resources. If a request exceeds its limit: It is either queued or rejected with 429 Too Many Requests.

🔐 Example Flow:
------------------

 > You have a script sending 1000 pod creation requests.
 > API server groups those requests into a “flow”.
 > If that flow is over its fair share, Kubernetes:
       * Queues some requests temporarily.
       * Drops or delays others to avoid overloading etcd and the API server.


❓ Follow-up Questions You May Get:
----------------------------------------

Q: What happens when the API server gets a lot of requests at once?
A: It queues or throttles requests based on priority/fairness.

Q: What status code is returned when throttled?
A: HTTP 429 Too Many Requests.

Q: How can you control a component’s rate?
A: By adjusting client flags like --kube-api-qps and --kube-api-burst.

-----------------------------------------------------------------------------------------------------------------------------------------------------------

# 5. What’s the role of ETCD in response to an API request?

ETCD is the backend database of Kubernetes. It stores the entire cluster state — all objects like Pods, Services, ConfigMaps, Secrets, etc. When an API request reads or writes data, the API server interacts with ETCD to fetch or store that state.

⚙️ Workflow (Behind the Scenes):
---------------------------------
Let’s say you run ```kubectl create pod mypod.yaml```  Here’s what happens: 
- 🔐 Authentication & Authorization    
- ✅ Admission Controllers    
- 💾 etcd Write: API server stores the Pod object into etcd. etcd makes it durable, consistent, and replicated

Now, if you run ```kubectl get pods``` , API server reads the pod list from etcd , Returns it to you via kubectl

🔁 ETCD Ensures:
| Feature               | Role                                      |
| --------------------- | ----------------------------------------- |
| **Consistency**       | All API servers get the same latest state |
| **High Availability** | Replicated across nodes                   |
| **Durability**        | Data survives restarts and failures       |

❓ Follow-up Questions You Might Be Asked:
--------------------------------------------

Q: What happens if ETCD goes down?
A: The cluster becomes read-only or unstable — new objects can’t be stored, and existing state may be outdated.

Q: Is data cached somewhere?
A: Yes, the API server may cache some objects in memory for performance, but ETCD is the source of truth.

Q: What kind of database is ETCD?
A: It’s a distributed key-value store, built on the Raft consensus algorithm for strong consistency.

# =========================================================================================================================================================

# 🔐 ETCD – Key-Value Store
-----------------------------------------------------------------------------------------------------------------------------------------------------------
# 1. 📦 What exactly does ETCD store? Can you give examples of keys?

ETCD stores all cluster state as key-value pairs — this includes everything the Kubernetes API knows.

🔍 Examples of what ETCD stores:

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
-----------------------------

Q: Can I read etcd directly?
> A: Yes, but you should never manually edit etcd — always use kubectl or the API server.

-----------------------------------------------------------------------------------------------------------------------------------------------------------

# 2- 💥 What are the risks of a single ETCD node failure?

If your ETCD has only one node and it fails, the entire cluster becomes unstable.

| 🧩 **Risk Type**         | 💥 **What Happens**                                                               |
| ------------------------ | --------------------------------------------------------------------------------- |
| ❌ **No Writes**          | API Server can't store any changes (e.g., creating new Pods, updates, scaling)    |
| ⚠️ **Read Instability**  | May serve stale or inconsistent data — not guaranteed to reflect the latest state |
| 🔄 **Controllers Break** | Auto-scaling, self-healing, and rolling deployments may **fail to trigger**       |
| 🧨 **No Recovery**       | If the disk is lost and there’s no backup, the **cluster state is lost forever**  |


💡 Best Practice:
------------------

* Run etcd as a 3-node cluster (or 5 in production) for:
* High availability  
* Quorum-based consistency (Raft)
* 👉 etcd needs a majority quorum (e.g., 2 of 3) to work.

❓ Follow-up You Might Be Asked:
----------------------------------

Q: What’s the minimum number of etcd nodes for high availability?
A: 3 (majority is 2).

Q: What if 2 out of 3 etcd nodes fail?
A: Cluster becomes unavailable for writes (no quorum).

-----------------------------------------------------------------------------------------------------------------------------------------------------------

# 3- 💾 How do you take and restore a backup of ETCD?

* **Use the **etcdctl** snapshot command to take a backup and restore it later into a new etcd cluster or Node.** 
* **To Take a Backup (Snapshot):**

```
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /tmp/etcd-backup.db
```
✅ This saves a point-in-time copy of the etcd database to /tmp/etcd-backup.db.

* 🛠️ **To Restore a Backup**

```
  ETCDCTL_API=3 etcdctl snapshot restore /tmp/etcd-backup.db \
  --data-dir /var/lib/etcd-restore
```
* Then use the restored --data-dir in your etcd startup config.

🔐 **Important Notes:**
-------------------------

- Always take backups periodically (daily or more).
- Back up before upgrades or config changes.
- Store snapshots off-node (e.g., S3, NFS, etc.).

❓ **Follow-up Interview Questions:**
-----------------------------------

Q: Can you automate etcd backup?
A: Yes — use a cronjob or Kubernetes Job + PV to back up regularly.

Q: Will restoring etcd restore everything in the cluster?
A: Yes — etcd contains all cluster objects. But it won’t restore persistent volumes unless backed up separately.

-----------------------------------------------------------------------------------------------------------------------------------------------------------

# 4-  🐢 What is the impact of ETCD latency on the whole cluster?

High etcd latency slows down API operations, controller decisions, and scheduling — impacting the entire cluster’s responsiveness and stability.

⚠️ What Happens When etcd Is Slow:
-----------------------------------

| Area Affected                 | What You See                             |
| ----------------------------- | ---------------------------------------- |
| `kubectl` commands            | Slow or time out (e.g., `kubectl get`)   |
| Pod scheduling                | Scheduler can't read/write pod info fast |
| Controllers (e.g. Deployment) | Slow to create/update/delete pods        |
| Autoscaling & healing         | Delays in decision making                |
| Cert expiration               | May fail if renewal data can’t be stored |

🔍 Root Causes of Latency:🔧 How to Monitor and Fix:
---------------------------------------------------

Monitor metrics like:
- etcd_disk_wal_fsync_duration_seconds
- etcd_server_has_leader (should always be 1)
- Use fast SSDs or dedicated nodes for etcd
- Keep etcd data size under control (avoid many Events or large Secrets)

❓ Follow-up Interview Questions:
-----------------------------------

Q: What’s the recommended storage type for etcd?
> A: Fast SSDs, ideally on dedicated nodes.

Q: How can you reduce etcd load?
> A: Clean up old Events, don’t store large config/secrets, use kube-apiserver caching.

-----------------------------------------------------------------------------------------------------------------------------------------------------------

# 5- 🔐 How does Kubernetes ensure consistency and availability in ETCD reads/writes?

- Kubernetes relies on etcd, which uses the Raft consensus algorithm to ensure:
- Strong consistency (only one agreed-upon version of data) , 
- High availability (even if some etcd nodes fail)

 ⚙️ Workflow: etcd Write Operation
--------------------------------------
> Example: `kubectl create -f mypod.yaml`

## 🔄 Step-by-Step:

🧾 Step 1: kubectl create -f mypod.yaml
- ou make a write request (e.g. creating a Pod)

🧠 Step 2: API Server handles the request
- Authentication: Who are you?
- Authorization: Are you allowed to do this?
- Admission Controllers: Any rules to apply?
- ✅ If all good → prepares the object and sends it as a write request to etcd

📡 Step 3: etcd (Raft Protocol)
- Only the etcd leader can handle writes.
- The leader sends the proposed change to follower nodes.
- Followers vote on it.

📊 Step 4: Quorum and Commit
- If a majority (quorum) of nodes agree (e.g., 2 out of 3):
- The leader commits the change to its own log
- Then followers apply the committed change
- Only then does the API server get a ✅ success response

💡 Why this matters:
- Ensures strong consistency: no node accepts different versions of the same data
- Ensures availability: even if one node is down, 2 out of 3 is enough for quorum
> 🔐 This guarantees **strong consistency** — no changes are considered “committed” unless the majority of etcd nodes agree.


🔁 Workflow: etcd Read Operation
-----------------------------------
1. 👤 You make a request
Example: kubectl get pod mypod

2. 🧠 API Server handles the request
- Performs Authentication
- Performs Authorization
- Checks cache (in-memory) for the object
- If cache is not fresh or unavailable, it goes to etcd

3. 📖 etcd handles the read
- The API Server connects to any etcd node (not necessarily the leader)
- The etcd node may:
  * Return the value from its local state (fast, but might be stale), or
  * Forward the read to the etcd leader for a linearizable (strongly consistent) rea

> Example: `kubectl get pods`

🔁 Two Types of Reads in etcd


| Read Type                                       | How it works                                                                     | Guarantees                                                            | Use Case                           |
| ----------------------------------------------- | -------------------------------------------------------------------------------- | --------------------------------------------------------------------- | ---------------------------------- |
| **Default (linearizable)**                      | Reads from etcd **leader’s local memory** (cache)                                | Fast & often consistent — but not guaranteed to have the latest write | Most kubectl queries               |
| **Strong Consistency (`--consistency=strong`)** | Forces etcd to check with **quorum (majority of nodes)** before serving the read | ✅ Guaranteed latest committed value                                   | Read-after-write or critical reads |


```bash
kubectl get pods --consistency=strong
```

> ✅ This ensures you're reading the **latest committed state** across the cluster.




✅ Example

A. Imagine this situation: You run: `````` kubectl create pod mypod.yaml `````` This writes to etcd.
B. Immediately after, you run: ```kubectl get pod mypod``` . If you don’t use --consistency=strong, the leader may serve a stale read from memory (before quorum finished committing).
C.If you do use --consistency=strong, the read will wait until the value is confirmed committed across quorum — so it’s guaranteed fresh.
> --consistency=strong tells etcd: "Don't just give me what you have in memory — first make sure majority of your cluster agrees this value was committed. Then give it to me."



🧠 Why This Matters:
---------------------

* **Writes require quorum** → guarantees high durability and consistency
* **Reads can be fast (default)** or **fully consistent (with quorum)** depending on the use case
* A **write never succeeds unless persisted to multiple nodes**, making etcd fault-tolerant


🧠 How Raft Ensures Consistency:
----------------------------------
| Concept       | Role                                  |
| ------------- | ------------------------------------- |
| **Leader**    | Accepts and coordinates writes        |
| **Followers** | Replicate data from leader            |
| **Quorum**    | Majority needed to commit a write     |
| **Term**      | Time period where a leader is elected |
⚠️ If the leader fails, a new leader is elected automatically — so etcd stays available.

📈 High Availability (HA) Strategy:
-------------------------------------
- Always use an odd number of etcd nodes (e.g., 3 or 5)
- Quorum = (N/2) + 1
Without quorum, writes will stop, but reads may continue

❓ Possible Interview Follow-ups:
------------------------------------
Q: What happens if etcd loses quorum?
A: No writes can be made; the cluster becomes read-only or degraded.
Q: **What ensures consistency in etcd?**
A: **The Raft protocol, which replicates logs and requires majority consensus.**
Q: Can you improve etcd read performance?
A: Yes — by using API server caching, and not forcing strong reads unless necessary.

# =========================================================================================================================================================

# 📅 Scheduler
-----------------------------------------------------------------------------------------------------------------------------------------------------------

# 1- 📦 How does the Kubernetes Scheduler choose a node for a pod?

The Kube-scheduler selects the best node for a Pod by running a two-step process:
 - **Filtering** → **Eliminate** nodes that can't run the Pod
 - **Scoring** → **Rank** the remaining nodes and pick the best one

Here's a **clean, detailed, and interview-friendly rewrite** of your **Kubernetes Scheduler Workflow**, with clarifications, improved structure, and explanations for each step:

---

⚙️ Pod Scheduling Workflow (Behind the Scenes)
------------------------------------------------

🔁 Step-by-Step:

1. 🆕 **Pod is Created (Unscheduled)**

* A new Pod is created without a `nodeName`.
* Example:

  ```yaml
  spec:
    nodeName: null  # Unassigned
  ```
---

2. 📡 **API Server Adds the Pod to the Scheduling Queue**

* Since the Pod is unscheduled, it’s placed in the **pending scheduling queue**.
* The API server doesn’t assign a node — it waits for the **Scheduler** to do that.

---

3. 🧠 **Scheduler Detects the Pending Pod**

* The **Kube Scheduler** continuously watches for **unscheduled Pods** via a **watch mechanism** on the API server.
* It picks up the Pod for processing.

---

4. 🧪 **Scheduler Runs Filter Phase (a.k.a. "Predicates")**

* It eliminates nodes that **cannot** run the Pod based on constraints:

  | Predicate Check              | Description                                                      |
  | ---------------------------- | ---------------------------------------------------------------- |
  | ✅ **Resource Fit**           | Does the node have enough **CPU/Memory**?                        |
  | ✅ **Node Affinity / Taints** | Does the Pod tolerate the node's taints or match affinity rules? |
  | ✅ **Node Readiness**         | Is the node `Ready` and schedulable?                             |
  | ✅ **Port Conflicts**         | Are the ports required by the Pod already in use?                |

* After filtering, only **eligible nodes** remain.

---

5. 🧮 **Scheduler Runs Scoring Phase (a.k.a. "Priorities")**

* Assigns scores to remaining nodes to find the **best fit**:

  | Priority Function             | Description                                 |
  | ----------------------------- | ------------------------------------------- |
  | 🔄 **PodTopologySpread**      | Spread Pods evenly across zones/nodes       |
  | 🧮 **LeastRequestedPriority** | Prefer nodes with more free resources       |
  | 📌 **Node Affinity Priority** | Give higher score to nodes preferred by Pod |
  | ⚡ **Custom Plugins**          | If enabled, score based on external rules   |

---

6. ✅ **Scheduler Picks the Highest-Scoring Node**

* After scoring, the scheduler selects the **node with the highest score** to assign the Pod.

---

7. 🧷 **Scheduler Patches the Pod’s `nodeName`**

* It makes an API call to **bind the Pod**:

```yaml
spec:
  nodeName: chosen-node
```

* This officially **assigns the Pod** to a node.

---

8. 🔁 **Kubelet on the Target Node Now Takes Over**

* The Kubelet on `chosen-node`:

  * Sees the assigned Pod
  * Pulls container images
  * Creates sandbox and containers
  * Starts the Pod

---

### ✅ Summary (One-Liner Style):

> The Scheduler filters nodes based on constraints, scores the rest, picks the best node, and binds the Pod to it by setting `spec.nodeName`.

---

❓ Follow-up Questions:
----------------------------
Q: Who actually starts the Pod?
> A: The kubelet on the assigned node.

Q: What if all nodes fail filters?
> A: Pod stays unscheduled until conditions change.

-----------------------------------------------------------------------------------------------------------------------------------------------------------

# 2 - 🧮 What are Predicates and Priorities (now Filter and Score)?

These were renamed in Kubernetes v1.19+:
| Old Term (v1.18 and below) | New Term (v1.19+) |
| -------------------------- | ----------------- |
| **Predicate**              | **Filter plugin** |
| **Priority**               | **Score plugin**  |

🧪 Filter Plugins (Predicates):
---------------------------------
These eliminate non-eligible nodes:
* NodeUnschedulable (cordoned node)
* PodFitsResources (CPU/RAM not enough)
* PodToleratesNodeTaints
* NodeAffinity / NodeSelector
🔍 Think: "Can this node run the pod?"

📊 Score Plugins (Priorities):
---------------------------------
These rank eligible nodes:
* LeastRequestedPriority (prefer low usage)
* BalancedAllocation (CPU/mem balance)
* TopologySpreadConstraint (spread across zones/racks)
* NodeAffinityPriority (custom rules)
🔍 Think: "Which of these eligible nodes is best?"

❓ Follow-up Questions:
-------------------------------
Q: Can you customize this behavior?
> A: Yes, via scheduler profiles or custom plugins.

Q: What plugin would spread pods across zones?
> A: PodTopologySpread scoring plugin.

-----------------------------------------------------------------------------------------------------------------------------------------------------------

# 3 - 🚫  How do Taints and Tolerations influence scheduling decisions?

**Taints** are **set on nodes** to repel Pods. **Tolerations** are added to **Pods** to allow scheduling on tainted nodes. A Pod must tolerate all taints on a node to be scheduled there.

⚙️ Workflow:
----------------

- A node is tainted, for example: ```kubectl taint nodes node1 key=value:NoSchedule``` 
- The scheduler sees this taint. 
- If a Pod has a matching toleration, it's allowed to be scheduled. 
- If not, it's filtered out during scheduling.

🧠 Use Case:
--------------
- Dedicate nodes for GPU or system workloads.
- Prevent general Pods from landing on sensitive nodes.

❓ Follow-up Questions:
------------------------
Q: What are the taint effects?
> A: NoSchedule, PreferNoSchedule, NoExecute

Q: What happens with NoExecute?
> A: Evicts running Pods without a matching toleration.

-----------------------------------------------------------------------------------------------------------------------------------------------------------

# 4. What happens when no node fits the pod’s constraints?

- The Pod remains Pending until a suitable node becomes available.
- Common reasons: 
   * Insufficient CPU/RAM , 
   * Taints without matching tolerations , 
   * Node affinity rules not met ,  
   * Volume limits exceeded , 
   * Topology spread constraints too strict

❓ Follow-up Questions:
----------------------------
Q: How do you debug a Pending Pod?
> A: Use kubectl describe pod and check Events section.

-----------------------------------------------------------------------------------------------------------------------------------------------------------

# 5. 🔄  How do Affinity / Anti-affinity rules influence scheduling?

They guide the scheduler to prefer or require that Pods are co-located or spread across nodes based on labels.

💡 Types:
-----------
| Type                  | Purpose                         | Example                            |
| --------------------- | ------------------------------- | ---------------------------------- |
| **Node Affinity**     | Schedule Pods on specific nodes | Based on node labels               |
| **Pod Affinity**      | Place Pods together             | E.g. web + cache on same node      |
| **Pod Anti-affinity** | Spread Pods apart               | E.g. avoid single point of failure |

⚙️ Workflow Example (Pod Anti-affinity):
----------------------------------------
```
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: web
        topologyKey: "kubernetes.io/hostname"
```
> ➡️ This rule prevents multiple web Pods on the same node.

❓ Follow-up Questions:
--------------------------
Q: What’s the difference between preferred and required?
> A: required is mandatory (hard rule); preferred is best effort (soft rule).

Q: What is topologyKey?
> A: It's the label like kubernetes.io/hostname or topology.kubernetes.io/zone used to determine groupings for affinit

# =========================================================================================================================================================
🔄 Controller Manager
-----------------------------------------------------------------------------------------------------------------------------------------------------------

# 1- 🧠 What are the types of controllers running in the Controller Manager?

The Controller Manager runs multiple built-in controllers that monitor the cluster state and reconcile it to the desired state.

⚙️ Common Controllers:
-----------------------
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
-------------------
- Each controller uses the Informer pattern:
- Watches objects (e.g., Deployments, Nodes)
- Compares actual vs desired state
- Issues API calls to fix the drift (e.g., creating/deleting pods)

❓ Follow-up Questions:
------------------------
Q: Are these separate processes?
> A: No, they run as goroutines inside the same controller-manager process.

Q: Can you add custom controllers?
> A: Yes, via Operators or custom controllers using client-go.

-----------------------------------------------------------------------------------------------------------------------------------------------------------

# 2- 🚀  How does the Deployment controller ensure a rollout happens correctly?

The Deployment Controller manages rollouts by creating and updating ReplicaSets, ensuring Pods are updated gradually and safely.

⚙️ Workflow of a Rollout:
-------------------------
- You apply a new Deployment  ``` kubectl apply -f deployment.yaml ```
- The Deployment controller:
   * Detects the change
   * Creates a new ReplicaSet for the new version
   * Scales up the new ReplicaSet and scales down the old ReplicaSet
- It uses the strategy defined in the Deployment:
   * RollingUpdate (default)
   * Recreate (delete all, then create new)
- It honors parameters like:
   * maxUnavailable: how many old Pods can be down during update
   * maxSurge: how many extra new Pods can be created temporarily

💬 Example:
------------
```
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 2
```
🧠 Behind the Scenes:
---------------------------
- Uses ReplicaSet controller to handle pod scaling
- Ensures new Pods are Ready before proceeding to next batch
- If a failure is detected, rollout can be paused or rolled back

❓ Follow-up Questions:
-----------------------
Q: How does the controller detect a rollout failure?
> A: Based on readiness probes and Pod status, it monitors availability.

Q: How do you pause or undo a rollout?
A:
```
kubectl rollout pause deployment <name>
kubectl rollout undo deployment <name>
```

-----------------------------------------------------------------------------------------------------------------------------------------------------------

# 3- 🧨  What happens if the ReplicaSet controller fails?

If the ReplicaSet controller fails:
- No new Pods will be created or deleted for that ReplicaSet.
- The system becomes stale — current state will not be reconciled with desired state.

🧠 Important Notes:
- Existing Pods will keep running, since kubelet manages them on the nodes.
- Other controllers (e.g., Deployment controller) may still function.
- Once the controller comes back up, it resumes reconciliation using event-based watches and local cache.

❓ Follow-up Questions:
--------------------------

Q: Is there a backup ReplicaSet controller?
> A: No, but the controller-manager process is usually replicated (in HA setups) to prevent this issue.

Q: Can it recover state?
> A: Yes. Controllers are stateless; desired/actual state is stored in etcd, so they recover automatically.

``````
✅ Corrected Understanding of HA Components in Kubernetes
---------------------------------------------------------
| Component                  | Stateless?     | HA Capable?     | Notes                                               |
|---------------------------|----------------|-----------------|-----------------------------------------------------|
| API Server                | ✅ Yes          | ✅ Yes          | Horizontally scalable; all replicas are active      |
| kube-controller-manager   | ❌ No *(Leader)*| ✅ Yes          | One active leader; others standby via leader election |
| kube-scheduler            | ❌ No *(Leader)*| ✅ Yes          | Same as controller-manager; leader-elected          |
| etcd                      | ❌ No           | ✅ Yes (Quorum) | Needs odd nodes (3/5); uses Raft consensus          |
| cloud-controller-manager  | ❌ No *(Leader)*| ✅ Yes          | Optional; used in cloud-native setups like GKE/EKS  |


## ✅ The Key Distinction:

> **"Controllers are stateless"** refers to **controller *logic***,
> while **"controller-manager is not stateless"** refers to **the running *process* needing leader election**.

---

### 🔍 Breakdown:

### ✅ **Yes, controllers are logically stateless:**

* Controllers (like ReplicaSet, Deployment, etc.) **do not keep in-memory state** between restarts.
* All **desired state** (specs) and **actual state** (Pod statuses, events, etc.) are stored in **etcd**.
* So if the controller process (inside `kube-controller-manager`) crashes or is restarted:

  * Nothing is lost.
  * It **re-reads everything from etcd** and continues reconciling state.

📌 **This is why we say: “Controllers are stateless.”**

---

### ❗ But: **`kube-controller-manager` process is not stateless in HA setups**

* In a multi-replica HA setup, you **can’t have all instances act at once**.
* Therefore, a **leader election mechanism** is needed to ensure **only one instance is active** at any given time.
* This doesn't mean the process holds permanent state — it just means:

  * **Only the leader executes logic** (like reconciling ReplicaSets)
  * Others **wait in standby**

📌 This is why we say: **"kube-controller-manager requires leader election"** — but **the logic it runs is stateless**.

---

## ✅ Final Summary (Interview-Ready):

> **Kubernetes controllers are logically stateless.**
> All state is stored in **etcd**, so if the controller restarts, it simply resumes by reading the desired and current state from etcd.
> However, in **HA setups**, only one instance of the controller-manager (or scheduler) is **active** at a time — this is enforced using **leader election**, not because of in-memory state, but to avoid conflicts in taking actions.

``````

-----------------------------------------------------------------------------------------------------------------------------------------------------------

# 4-🔁  What is a reconciliation loop, and how frequently do controllers run it?

A reconciliation loop is a logic loop where a controller:
"Continuously compares the desired state (from etcd) with the actual state (from the cluster), and makes changes to match them."

⚙️ Workflow:
-----------
- Controller watches a Kubernetes object (e.g., Deployment).
- Gets an event (add/update/delete).
- Fetches the current state (e.g., 1 Pod running, 3 desired).
- Takes action (e.g., create 2 more Pods).
- Waits for the next change/event.

⏱️ How frequently?
-------------------
- Based on events (watch API)
- Also runs periodic re-sync (every few minutes, e.g., 5 mins) to handle missed events.

❓ Follow-up Questions:
----------------------------
Q: Is reconciliation continuous?
> A: Yes, it’s reactive (event-driven) + proactive (periodic).

-----------------------------------------------------------------------------------------------------------------------------------------------------------

# 5 - ⚔️How does the controller handle conflicts between actual and desired state?

The controller overwrites the actual state to match the desired state defined in etcd (via Deployment, etc.).

⚙️ Workflow Example:
--------------------
- Desired state: replicas: 3
- Actual state: only 1 Pod running (maybe 2 were deleted manually)
- Controller detects mismatch
- It creates 2 new Pods to fix it

🛡️ Conflict Handling:
------------------------
- Controllers don’t care why the actual state is different 
- They just enforce the desired state.
- Manual changes not reflected in desired spec will be reverted.
- Example: You manually delete a Pod — controller will recreate it.

❓ Follow-up Questions:
---------------------------
Q: What if someone manually changes a pod created by a ReplicaSet?
> A: The controller will revert or replace it.

# =========================================================================================================================================================
🌩️ Cloud Controller Manager (if on cloud like AWS/GCP)
-----------------------------------------------------------------------------------------------------------------------------------------------------------

# ☁️ 1. What are the components of the Cloud Controller Manager?
CCM splits out cloud-specific logic from the main Kubernetes Controller Manager. It contains controllers that interact with the cloud provider’s APIs.

🧱 Key Components of CCM:
---------------------------
| Component              | Function                                              |
| ---------------------- | ----------------------------------------------------- |
| **Node Controller**    | Updates node status (e.g., shutdown, delete in cloud) |
| **Route Controller**   | Sets up network routes between nodes (VPC, etc.)      |
| **Service Controller** | Creates and manages LoadBalancers for Services        |
| **Volume Controller**  | Manages cloud storage (EBS, PD, etc.) for PVCs        |


🧠 Behind the scenes:
------------------------
- Watches API objects like Node, Service, PersistentVolume
- Calls cloud provider APIs (e.g., AWS SDK) to provision infra

-----------------------------------------------------------------------------------------------------------------------------------------------------------

# 🔄 2. How does CCM manage Load Balancers and Persistent Volumes?

✅ Workflow for LoadBalancer Service:
---------------------------------------
- You create a Service with type: LoadBalancer.
- The Service Controller in CCM sees it.
- It calls the cloud provider API (e.g., AWS ELB or GCP LB).
- A load balancer is created and its IP is added to the Service status.

✅ Workflow for PersistentVolume:
---------------------------------------
- You create a PVC.
- The Volume Controller:
    * Talks to the cloud provider (e.g., AWS EBS, GCP PD)
    * Provisions a disk
- A PersistentVolume is created and bound to the PVC.

❓ Follow-up:
---------------
Q: What is in-tree vs out-of-tree?
> A: CCM is part of the move to "out-of-tree" cloud providers (decouples cloud logic from Kubernetes core).

-----------------------------------------------------------------------------------------------------------------------------------------------------------

# 🌐 3. How does the CCM interact with cloud APIs?

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

# 💥 4. What happens if CCM fails or loses cloud API access?

❌ Impacts:
- New LoadBalancers or Volumes will not be provisioned.
- Node status updates (e.g., instance shutdowns) may be missed.
- Routes may not be created.
- Existing Pods and Volumes still work (managed by kubelet/kube-proxy).

🔁 Recovery:
- When CCM restarts or API access is restored, it resumes reconciliation from etcd state.

❓ Follow-up:
Q: Will pods crash?
A: No, unless they depend on a resource that CCM failed to provision (like a PVC).

-----------------------------------------------------------------------------------------------------------------------------------------------------------

# 🔄 5. How is CCM different from the regular Controller Manager?
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

# 🔁 1. What is the complete lifecycle of a Pod from scheduling to termination?

Pending → Scheduled → Running → Succeeded/Failed → Terminating

🧬 Workflow:
------------

- Pending: Pod created in API Server, but not yet scheduled to a node.
- Scheduled: Scheduler assigns it to a node.
- Running: kubelet pulls images, runs initContainers, then containers.
- Succeeded / Failed: Completed based on container exit codes.
- Terminating: 
   * kubectl delete pod or controller scale-down.
   * PreStop hooks and graceful shutdown handled.

🔍 Follow-up:
----------------
Q: Who handles Pod creation?
> A: kubelet on the target node fetches the Pod spec and manages container lifecycle via CRI.

-----------------------------------------------------------------------------------------------------------------------------------------------------------

# ⚙️ 2. How do initContainers influence startup order?

`InitContainers` are **special containers that run *before*** your main application containers start in a Pod. They **run one after the other**, and **must all succeed** before the regular containers are started.
(mtlb saare init containers start hone chaiye before main container starts)


 🧠 **Why do we need initContainers?**

You can think of them as **"setup steps"** for your application.

### ✅ **Use Cases:**

| Use Case                           | Example                                                        |
| -----------------------------------| -------------------------------------------------------------- |
| ✅ Wait for a database to be ready | Poll the DB until a connection is successful                   |
| ✅ Set permissions on a volume     | Run `chown` or `chmod` before the app starts                   |
| ✅ Download configs or secrets     | Pull a file from an external service before launching main app |
| ✅ Pre-warm a cache                | Populate Redis or local cache with required keys               |

---

### 🔄 **Startup Workflow (Visual):**

```yaml
spec:
  initContainers:
    - name: init-db
      image: busybox
      command: ['sh', '-c', 'until nc -z db 5432; do sleep 1; done']
  containers:
    - name: app
      image: my-web-app
```

➡️ Step-by-step execution:

```
[init-db] --> (runs, exits successfully)
       ↓
[app]     --> (starts only after all initContainers are done)
```

---

### 🔁 **Behavior Rules:**

* `initContainers` **always run first**, and in order (1 → 2 → 3…).
* If one fails, **Kubelet will retry it** according to the Pod's `restartPolicy`.
* Once all initContainers succeed, they **do not run again**, even if app container crashes later.
* App containers will **not start** unless all initContainers complete successfully.

---

### ❗ Important Notes:

* `initContainers` can have their **own images, volumes, env vars, etc.**
* They **share the same network and volume** space as the main containers.
* Logs for `initContainers` are separate:

```bash
kubectl logs <pod> -c <init-container-name>
```

---

### ✅ Summary (Interview-friendly line):

> “`initContainers` are like pre-steps in a Pod. They run sequentially before main containers and ensure preconditions (like DB readiness or volume setup) are met before the app starts. If any initContainer fails, the Pod won’t start until it succeeds.”


-----------------------------------------------------------------------------------------------------------------------------------------------------------

# ❤️‍🔥 3. What are the differences between liveness, readiness, and startup probes?

| Probe Type    | Purpose                                   | Effect when Fails                         |
| ------------- | ----------------------------------------- | ----------------------------------------- |
| **Liveness**  | Checks if container is **alive**          | Container is **restarted**                |
| **Readiness** | Checks if container is **ready**          | Pod is **removed from Service endpoints** |
| **Startup**   | Checks if container **finished starting** | Delays liveness & readiness checks        |

✅ Liveness Probe – How It Works (Step-by-Step)
------------------------------------------------

1. Probe worker starts: 
→ Kubelet reads the livenessProbe and starts a background checker for that container.

2. Wait before first check
→ It waits for initialDelaySeconds (e.g. 10s) to let the app start.

3. Probe runs every few seconds
→ It checks health using one of 3 ways:
* httpGet: Sends HTTP request (e.g. /status). 2xx/3xx = healthy.
* tcpSocket: Tries to connect to a port.
* exec: Runs a command inside the container (exit 0 = healthy).

4. If check fails: → After failureThreshold (e.g. 3 fails), container is marked Unhealthy.

5. Container is restarted → Kubelet stops it (SIGTERM then SIGKILL) and restarts it based on restartPolicy.

6. Cycle repeats after restart  → Probe starts again from step 2.

🔍 Example with HTTP /status (httpGet):
```
livenessProbe:
  httpGet:
    path: /status
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3
```
* If /status gives 200 → healthy → container keeps running
* If it gives 500 or no reply → fails → after 3 times → container restarts

✅ Other probe types:
* tcpSocket → Just checks if port is open.
* exec → Runs a script/command inside the container.

---

✅ Readiness Probe – How It Works (Step-by-Step)
-------------------------------------------------

1. Probe worker starts
→ Kubelet reads the readinessProbe and starts checking if the container is ready to receive traffic.

2. Wait before first check
→ It waits for initialDelaySeconds (e.g. 5s) before starting checks.

3. Probe runs repeatedly
→ It checks health using:
 * httpGet: Sends HTTP request to endpoint (e.g. /ready)
 * tcpSocket: Tries to connect to a port
 * exec: Runs a command inside the container

4. If check fails
→ Pod is removed from Service endpoints — it won't get traffic.

5. If check passes again
→ Pod is added back to the Service — starts receiving traffic again.

🔍 Example with HTTP /ready:
```
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 2
```
* If /ready gives 200 → pod stays in load balancer
* If it gives 500 or fails → pod is removed from load balancer (but not restarted)

✅ Key Points:
* Does NOT restart the container — only controls traffic flow
* Used when app is running but not yet ready (e.g., DB not connected)

✅ Other probe types:
* tcpSocket → Checks if port is listening
* exec → Runs a script or check inside the container

---

✅ Startup Probe – How It Works (Step-by-Step)
-------------------------------------------------

1. Probe starts immediately
→ Kubelet runs the startupProbe right after the container starts.

2. Runs until it succeeds or fails too many times
→ Only this probe runs during startup.

failureThreshold × periodSeconds = max startup time

3. If startup probe passes
→ Kubelet enables liveness and readiness probes, and normal checks begin.

4. If it fails continuously
→ After failureThreshold, container is marked Unhealthy and restarted.

🔍 Example with HTTP /startup
```
startupProbe:
  httpGet:
    path: /startup
    port: 8080
  failureThreshold: 30     # allows up to 150s (30 × 5)
  periodSeconds: 5

```
* App gets up to 150 seconds to start.
* If /startup gives 200 → success → normal liveness/readiness start.
* If it fails 30 times → container is restarted.

✅ Key Points:
* Used for slow-starting apps (e.g., Spring Boot)
* Prevents early liveness probe failures
* Disables other probes until it passes

✅ Other probe types:
* httpGet, tcpSocket, exec all supported — same as other probes



🔍 Follow-up:
-------------
Q: What happens if readiness fails but liveness is OK?
A: Pod stays running, but won't receive traffic from Services.

-----------------------------------------------------------------------------------------------------------------------------------------------------------

# 🧨 4. What happens when a probe fails repeatedly?

✅ Liveness Probe fails repeatedly
-----------------------------------

* If it fails failureThreshold times in a row (default 3):
   * → The container is marked Unhealthy
   * → Kubelet restarts the container
   * → After restart, the probe cycle starts again
   * → If this keeps happening → enters CrashLoopBackOff

✅ Readiness Probe fails repeatedly
-------------------------------------

* The pod is removed from the Service endpoints
* → It stays running, but won’t receive traffic
* → Once the probe passes again → it’s added back

✅ Startup Probe fails repeatedly
---------------------------------

* If it fails failureThreshold times
* → Container is restarted
* → Liveness and readiness probes never run if startup probe doesn’t succeed

---

🔁 Probe failure triggers summary:
-------------------------------------

| Probe Type | Action on repeated failure                                  |
| ---------- | ----------------------------------------------------------- |
| Liveness   | ❌ Restart container                                         |
| Readiness  | 🚫 Remove from Service (no restart)                         |
| Startup    | ❌ Restart container (disables other probes until it passes) |

-----------------------------------------------------------------------------------------------------------------------------------------------------------


# 📦 5. What is the order of container startup in a multi-container Pod?

#### ✅ **Startup Order:**

1. **`initContainers`** run **first**, one after another (sequentially).

   * Each must **complete successfully** before the next one starts.
2. After all `initContainers` finish:
   → **All app containers** (`containers:` block) start **in parallel**.

---

### 🔁 Example:

```yaml
initContainers:
  - name: init-db     # runs first
  - name: init-cache  # runs second, after init-db

containers:
  - name: web-app     # starts after all initContainers
  - name: sidecar     # starts at same time as web-app
```

---

### 🧠 Key Points:

* App containers **do not wait** for each other — they start together.
* **Startup order for main containers cannot be controlled**.
* If you need order between app containers → use a **startup probe** or control logic inside the app.

---

**In short:**
`initContainers` run one-by-one first. Once done, all main containers start **together**. You cannot enforce a startup order between main containers.


# =========================================================================================================================================================

# ✨ Bonus – Troubleshooting Components
-----------------------------------------------------------------------------------------------------------------------------------------------------------

# 🟠 1. A pod is in Pending state for 10 minutes. How would you investigate?

✅ Root Cause Possibilities:
-----------------------------

- No available nodes matching pod’s resource requests or constraints
- Unschedulable due to taints, affinity rules, or node selectors
- Image pull issues (if imagePullPolicy blocks scheduling)

🔍 Workflow to Debug:
---------------------
```
kubectl describe pod <pod-name>  # Look under "Events"
kubectl get events --sort-by='.lastTimestamp'
```

🔍 Check for:
---------------
- “0/3 nodes available” → Scheduler can’t place Pod
- Insufficient memory/CPU → Node doesn't meet resource requests
- Taints/Tolerations mismatch
- PVC not bound (storage issue)

❓ Follow-up:
--------------
"What if it’s a DaemonSet pod?"
If a DaemonSet pod doesn't show up on a node, check these common blockers:

✅ Things to check:
| Check                       | Why it matters                                                             |
| --------------------------- | -------------------------------------------------------------------------- |
| **NodeSelector**            | Does the Pod have a `nodeSelector` that excludes the current node?         |
| **Tolerations & Taints**    | Does the node have taints that the Pod can't tolerate?                     |
| **Host Ports**              | Is the host port already in use on the node? If yes → pod can't bind to it |
| **Resource Requests**       | Does the Pod request more CPU/memory than available on the node?           |
| **Affinity/Anti-affinity**  | Any rules that prevent pod placement on that node?                         |
| **Pod Conditions / Events** | Use `kubectl describe pod <pod>` to see why it’s pending or failed         |
| **Node Status**             | Is the node `Ready` and schedulable (`kubectl get nodes`)?                 |



-----------------------------------------------------------------------------------------------------------------------------------------------

# 🚫 2. Kubelet logs show container 'created' but not 'started' — what could be wrong?

✅ Probable Issues:
--------------------
- Image pulled but container failed during startup
- Failing liveness/startup probe
- Permission issue (e.g., missing volume mount)
- Waiting for initContainers to finish

🧪 Workflow:
--------------
```
journalctl -u kubelet -f  # Live logs
kubectl describe pod <pod>
kubectl logs <pod-name> -c <container>
```
- Look for crash loops, probe failures
- Check container runtime logs (e.g., containerd)

❓ Follow-up:
---------------
"How does kubelet decide when to restart a container?" 
> Based on the pod's restartPolicy and probe failures

-----------------------------------------------------------------------------------------------------------------------------------------------------------

# 3. API Server is unresponsive, but etcd is running — what’s your next step?

### ✅ **Answer (Short + Clear):**

> If the API Server is down but etcd is healthy, I’ll troubleshoot the API Server itself — checking its logs, static Pod health, and network access.

---

### 🔄 **Troubleshooting Workflow:**

#### 1. ✅ **Check API Server Pod health**

```bash
kubectl get pods -n kube-system -o wide | grep apiserver
```

OR if kubectl is down (likely):

```bash
docker ps -a | grep kube-apiserver
# or
crictl ps | grep kube-apiserver
```

---

#### 2. 📄 **Inspect logs**

```bash
docker logs <apiserver-container-id>
# or
crictl logs <container-id>
# or static pod:
cat /var/log/pods/kube-system_kube-apiserver*/kube-apiserver/*.log
```

Look for:

* TLS/cert issues
* etcd connection errors
* port binding issues

---

#### 3. 🧱 **Verify static Pod manifest**

```bash
cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

Check:

* Correct etcd endpoints
* Cert/key paths
* HostPort conflicts

---

#### 4. 🌐 **Check port 6443**

```bash
curl -k https://localhost:6443/healthz
```

If this fails, API server process may be running but unhealthy.

---

#### 5. 🧠 **System-level checks**

* `df -h`: Disk full?
* `top`: CPU/memory?
* `systemctl status kubelet`: Is kubelet healthy and managing the API server?

---

### ✅ Final line to say:

> I’ll check the API server logs and Pod status directly on the control plane node. Since etcd is fine, the issue is isolated to the API server itself — possibly config, port, or resource-related. My goal is to confirm whether it’s running but unhealthy, or not starting at all.

-----------------------------------------------------------------------------------------------------------------------------------------------------------

# 🟥 4. A node reports NotReady — what logs or components would you check?

✅ Root Causes:
- kubelet is down or misconfigured
- Node network issue
- Docker/containerd failure
- High resource usage

```
When a node shows up as NotReady, the first thing to realize is that Kubernetes marks it unhealthy because the kubelet on that node has stopped reporting a healthy status. To pinpoint why, you’ll want to check the following components and logs on the node itself as well as view any relevant events from the control-plane:

1. Node Events (from the control-plane):
------------------------------------------------
kubectl describe node <node-name>
Look under the “Conditions” section to see why it’s NotReady (e.g. KubeletNotReady, NetworkUnavailable, DiskPressure).
Scroll to the “Events” at the bottom—often you’ll see repeated NodeReady→NodeNotReady transitions with short error messages.

2. Kubelet Service & Logs
-------------------------
On the problematic node
# Check that kubelet is running
sudo systemctl status kubelet

# Follow its logs for errors
sudo journalctl -u kubelet -f

What to look for:
- Authentication or TLS errors (e.g. expired client cert)
- “Failed to register node” or “Connection refused” to the API server
- Cgroup driver mismatches or filesystem permission errors
- Node heartbeat timeouts or frequent restarts

3. Container Runtime (containerd / Docker)
----------------------------------------------
If the kubelet can’t manage containers, it will go NotReady:
sudo systemctl status containerd      # or `docker` if you use Docker
sudo journalctl -u containerd -f

Check for:
- Crashes or panics in the runtime
- Failed pulls or inability to start the pause/containerd-shim containers
- Disk full or permission issues under /var/lib/containerd (or /var/lib/docker)

4. CNI Plugin Health & Logs
------------------------------
Networking must be up for the kubelet’s network checks:

# If you’re using Calico, for example:
kubectl get pods -n kube-system | grep calico
kubectl logs -n kube-system calico-node-xxxxx

# For other CNIs (weave, flannel, ovn), swap names accordingly
Symptoms:
 - NetworkUnavailable condition in node status
 - CNI containers crash-looping or failing liveness/readiness probes


5. Node Problem Detector (if installed)
-----------------------------------------
This daemon picks up OS-level faults:
kubectl get ds -n kube-system node-problem-detector
kubectl logs -n kube-system node-problem-detector-xxxxx

Use it to catch:
Kernel panics, disk full, memory exhaustion, or hardware errors that the kubelet itself might not log clearly.

6. Disk, Memory & Pressure Checks
-----------------------------------
On the node itself:
# Disk usage
df -h

# Check inode exhaustion
df -i

# Memory and swap
free -m

# See cgroup pressures
cat /sys/fs/cgroup/*.cpu.stat
If the node is under DiskPressure or MemoryPressure, Kubernetes will mark it NotReady.

7. API Server Reachability
------------------------------
Make sure the kubelet can talk to the control-plane:
# Example; replace with your API server URL
curl -k https://<api-server>:6443/healthz
If this fails, kubelet heartbeats will time out and the node flips to NotReady.

Putting it all together
=========================
- Describe the node (kubectl describe node) → see the condition & events.
- Check kubelet status and logs for errors in authentication, cgroups, or API connectivity.
- Verify the container runtime (containerd/Docker) is up and not throwing storage or container-launch errors.
- Inspect your CNI pods and logs for network plugin failures.
- Look at Node Problem Detector (if you have it) for OS-level issues.
- Validate disk/memory health and API-server reachability.

Following that sequence will almost always reveal why the kubelet stopped reporting “Ready,” so you can fix the root cause—be it a cert expiry, a full disk, a crashed container runtime, or a broken network plugin.


```

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

# 📏 5. Controller manager is reporting resource quota violations — how would you debug?
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

```
ResourceQuota
-------------
A ResourceQuota, defined by a ResourceQuota object, provides constraints that limit aggregate resource consumption per namespace. A ResourceQuota can also limit the quantity of objects that can be created in a namespace by API kind, as well as the total amount of infrastructure resources that may be consumed by API objects found in that namespace. 

Key Points of the Definition
------------------------------
1- Namespace-Scoped
ResourceQuotas apply per namespace—each namespace can have its own quota object(s), and usage is tracked separately in each namespace

2- Aggregate Limits
You can cap both:
- Compute resources (e.g., requests.cpu, limits.memory)
- API object counts (e.g., number of Pods, Services, PersistentVolumeClaims)

3- Admission Enforcement
When a creation or update would exceed any “hard” limit defined in the ResourceQuota, the API server rejects the request with a 403 Forbidden error, including an explanatory message 

4- Scope and Granularity
Beyond simple totals, ResourceQuota supports scopes (e.g., “count only NotTerminating pods” or “count only BestEffort Pods”), letting you apply quotas to subsets of resources

Example Excerpt from the Kubernetes Docs
------------------------------------------
If creating or updating a resource violates a quota constraint, the control plane rejects that request with HTTP status code 403 Forbidden. The error includes a message explaining the constraint that would have been violated

==============================================================================================================

SET LIMIT PER POD IN NAMESPACE : "LimitRange"
--------------------------------

LimitRange: A“A LimitRange is a policy to constrain the resource allocations (limits and requests) that you can specify for each applicable object kind (such as Pod, Container, or PersistentVolumeClaim) in a namespace.” 


Is limit range admission controller?
--------------------------------------
When you create a LimitRange in a namespace, it’s enforced by the Kubernetes **LimitRanger admission controller**.
Admission controller is a piece of code the API server runs after authentication/authorization but before persisting an object.
Admission controllers only ever stop you at the API boundary—when you create or update a Pod object.Once a Pod is admitted, Kubernetes hands it off to the kubelet and container runtime, which enforce the limits at the OS level (cgroups throttle or kill).

What happens when there is no "LimitRange" but in deployment we place "resource.requests" & "resource.limits"
--------------------------------------------------------------------------------------------------------------
The key thing to remember is that the scheduler only looks at your requests, never your limits, when it decides where to place a Pod. Once the Pod is running, the kubelet + container runtime enforce your limits at the OS level—but they do not trigger a re-schedule if you suddenly “ask” for more CPU at runtime.

What actually happens:
1- Scheduling: 
You submit a Pod with
resources:
  requests:
    cpu: "2"
  limits:
    cpu: "5"
The kube-scheduler sees only the 2 CPU request, so it picks any node with ≥ 2 CPU free, even if that node only has 2.5 CPU available in total.

2- Runtime enforcement: 
When your container actually runs, Linux cgroups are configured with Guaranteed minimum of 2 CPU shares, and Maximum cap of 5 CPU shares.If your process tries to grab more than 5 CPU worth of cycles, the kernel would throttle it back down—but if the node doesn’t actually have 5 CPU to give (because other containers are using them), you’ll simply get whatever idle CPU remains, up to that 5 share cap.or if you ask for more mem than node actually have it will give OOM 

3- No re-scheduling:
Asking for extra CPU at runtime doesn’t make Kubernetes pick a new node. Your Pod stays on the original node. If the node is already busy, your container just competes (and is throttled) like any other Burstable QoS container.

In plain English
Requests = “I need at least this much to start.” ⇒ Used by the scheduler.
Limits = “I’ll never use more than this much.” ⇒ Enforced by cgroups at runtime.
Because you asked for 2 CPU, Kubernetes happily put you on a node that had at least 2 CPU free. Later, when you try to use more, the OS won’t hop to a new server—you’ll simply get throttled to whatever headroom is left under your 5 share cap.

--------

CASE 1: When i have defined "LimitRange" so my deployment should agree i.e what resources it is reuqesting it mmust be within "LimitRange".


LimitRange manifest: 
----------------------
- This enforces per-container/per-pod floors, ceilings, and defaults in the my-ns namespace
- A LimitRange can enforce rules either per-container or per-Pod (the sum of all its containers). You choose that by setting "type: Container" or "type: Pod" under the same limits: list.

apiVersion: v1
kind: LimitRange
metadata:
  name: container-limits
  namespace: my-ns
spec:
  limits:
    - type: Container or Pod 
      min:
        cpu:    "100m"    # at least 0.1 CPU
        memory: 64Mi      # at least 64 MiB RAM
      max:
        cpu:    "2"       # no more than 2 CPU
        memory: 1Gi       # no more than 1 GiB RAM
      defaultRequest:
        cpu:    "200m"    # if requests omitted, assume 0.2 CPU
        memory: 128Mi     # if requests omitted, assume 128 MiB RAM
      default:
        cpu:    "500m"    # if limits omitted, cap at 0.5 CPU
        memory: 512Mi     # if limits omitted, cap at 512 MiB RAM

 Deployment that agrees with those limits
--------------------------------------------
Here each pod’s container explicitly requests() and limits() within the allowed range: Because the pod’s values sit inside the LimitRange bounds, it will be admitted and scheduled normally.


apiVersion: apps/v1
kind: Deployment
metadata:
  name: lr-demo
  namespace: my-ns
spec:
  replicas: 2
  selector:
    matchLabels:
      app: lr-demo
  template:
    metadata:
      labels:
        app: lr-demo
    spec:
      containers:
        - name: web
          image: nginx:alpine
          resources:
            requests:
              cpu:    "200m"    # ≥ min (100m), ≤ max (2)
              memory: 128Mi     # ≥ min (64Mi), ≤ max (1Gi)
            limits:
              cpu:    "500m"    # ≤ max (2)
              memory: 512Mi     # ≤ max (1Gi)

--------

CASE 2: When i have not defined any "LimitRange" In this case the API server won’t enforce any namespace-wide per-pod caps. You still can—and should—declare resource requests/limits in your Pod spec so the scheduler and kubelet know what to reserve and cap:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: no-lr-demo
  namespace: my-ns
spec:
  replicas: 3               # limits how many pods you run
  selector:
    matchLabels:
      app: no-lr-demo
  template:
    metadata:
      labels:
        app: no-lr-demo
    spec:
      containers:
        - name: worker
          image: busybox
          command: ["sh", "-c", "sleep 3600"]
          resources:
            requests:
              cpu:    "400m"    # scheduler will reserve at least 0.4 CPU
              memory: 256Mi     # scheduler will reserve at least 256 MiB RAM
            limits:
              cpu:    "800m"    # kubelet will throttle above 0.8 CPU
              memory: 512Mi     # OOM-kill if usage > 512 MiB

```

-----------------------------------------------------------------------------------------------------------------------------------------------------------

# Deployment

✅ Concept:
-------------

* Manages stateless apps.
* Ensures desired number of replica Pods are running.
* Supports rolling updates, rollbacks.
* Built on top of ReplicaSet.

🧠 Key Features:
-----------------
* Declarative updates to Pods.
* Easy scaling via replicas:.
* Automatically replaces failed Pods.

🔍 Example:
----------
``````
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx
``````

---

## 🔁 **How Deployment Rollback Works**

Kubernetes **stores a revision history** for every Deployment update. You can **revert** to a previous working version using:

```
## 1. Start with a baseline Deployment
-----------------------------------------------------
kubectl create deployment blue-app --image=nginx:1.20
kubectl get pods -l app=blue-app
# You’ll see one Pod running nginx:1.20

What happened:
* Kubernetes created a ReplicaSet behind the scenes (revision 1)
* That ReplicaSet spun up Pods using nginx:1.20

## 2. Update the Deployment to a new version
-------------------------------------------------------
kubectl set image deployment/blue-app nginx=nginx:1.22 --record

* deployment/blue-app is the name of deployment.often aligned with your “app name,” but it’s really just the Kubernetes object’s name.
* nginx=nginx:1.22 Within that Deployment’s Pod template, find the container named nginx and switch its image to nginx:1.22
* This changes the Pod template in the Deployment spec to nginx:1.22.
* Kubernetes treats that as **revision 2** and begins the rolling update.

## 3. Watch the rollout in action
----------------------------------
kubectl rollout status deployment/blue-app

You’ll see output like:
Waiting for deployment "blue-app" rollout to finish: 0 of 1 updated replicas are available...
deployment "blue-app" successfully rolled out


**Under the hood:**
--------------------
1. **New ReplicaSet** (rev 2) for nginx:1.22 is created.
2. **Surge up**: Depending on your `maxSurge` (default 25%), it may briefly run more Pods than desired.
3. **Scale down** the old ReplicaSet (rev 1) as soon as the new Pod becomes **Ready**.
4. **Finish** when all Pods are running rev 2 and rev 1 is fully scaled to zero.

## 4. Inspect the rollout history
-----------------------------------
kubectl rollout history deployment/blue-app

# Example:
# REVISION  CHANGE-CAUSE
# 1         kubectl create deployment blue-app --image=nginx:1.20
# 2         kubectl set image deployment/blue-app nginx=nginx:1.22 --record

This shows you each revision and what command triggered it.

## 5. Pause and resume (optional)
-------------------------------------
If you want to **pause** mid-rollout to verify things:

kubectl rollout pause deployment/blue-app
# Now no further Pods will be updated until you resume.

# After your checks:
kubectl rollout resume deployment/blue-app

## 6. Roll forward or backward
---------------------------------
* **Continue to a newer version** by repeating step 2 with a different image.
* **Rollback** to the previous revision if something breaks:

kubectl rollout undo deployment/blue-app

## 7. To roll back a Deployment to a specific revision, you use:
> kubectl rollout undo deployment/<deployment-name> --to-revision=<revision-number>
> kubectl rollout undo deployment/blue-app --to-revision=3
> kubectl rollout history deployment/blue-app
## Which will list something like:
REVISION  CHANGE-CAUSE
1         kubectl create deployment blue-app --image=nginx:1.20
2         kubectl set image deployment/blue-app nginx=nginx:1.22 --record
3         kubectl set image deployment/blue-app nginx=nginx:1.23 --record



### Why rolling updates matter
--------------------------------
* **Zero-downtime**: Pods are updated gradually—old ones only go down once new ones are healthy.
* **Control surge**: You decide how many extra (“surge”) or unavailable Pods you’d tolerate during the update.
* **Safe verification**: Pause the rollout to test the new version before completing.

By using `kubectl set image` (or editing your YAML and `kubectl apply`), then watching with `kubectl rollout status` (and controlling with `pause`/`resume`), you get a powerful, controlled way to update your application in place.

```


❓ Interview Questions:
------------------------
1- How does Deployment manage updates and rollbacks?

2- What happens if a node crashes with a Deployment?

3- How does it differ from StatefulSet?

----------------------------------------------------------------

## 🧩 **1. What if a Deployment is scaled to 0 — can I still rollback?**

#### ✅ Short Answer:
Yes, you can rollback. Scaling to 0 removes all pods, but the **Deployment object**, **ReplicaSets**, and **revision history** are preserved.

#### 🔍 What Actually Happens Internally:

* When you run: ```kubectl scale deployment myapp --replicas=0``` Kubernetes sets `spec.replicas = 0`.
This means:
* All pods are deleted
* The Deployment object still exists in the cluster
* Associated ReplicaSets are retained (with 0 replicas)

> ❗ Nothing is deleted — just paused

Now, when you run: ``` kubectl rollout undo deployment myapp```

* Kubernetes reverts the Deployment spec to the **last known revision** (e.g. image, env, etc.)
* It **updates the ReplicaSet** template
* But **replicas = 0**, so no pods are created unless you scale up again

✅ So you'd usually follow up with: ```kubectl scale deployment myapp --replicas=3 ```

#### 🧠 Key Takeaway:

> Rollback works **independent of pod count**. If replicas = 0, Kubernetes won’t auto-spawn pods even after rollback — you must scale it manually.

---

## 🧩 **2. I updated the image, but pods didn’t restart — why?**

#### ✅ Short Answer: 
```
1. You didn’t actually change the Pod’s template
------------------------------------------------------
What Kubernetes cares about: it looks at your Deployment’s Pod template (the fields under spec.template) and only rolls out new Pods when something in that template is different.
Common pitfall: you push a new Docker image tagged latest, but your Deployment still says image: myapp:latest.
Outcome: because the text "myapp:latest" didn’t change in the template, Kubernetes thinks “nothing to do” and leaves the old Pods running.
EXP:
----
spec:
  template:
    spec:
      containers:
      - name: api
        image: myapp:latest   # unchanged, so no rollout

Even if myapp:latest now points to a different image digest, the Deployment sees the same tag and skips rolling the Pods.

2. Your pull policy prevented fetching the new image
-------------------------------------------------------
Default behavior: most Pods use imagePullPolicy: IfNotPresent.
What that means: “When starting a container, only go out to the registry if you don’t already have this exact image locally.”
Why it matters: if a node has previously pulled myapp:latest, it won’t check for any updates in the registry—so your new image never gets downloaded
EXP:
  containers:
  - name: api
    image: myapp:latest
    imagePullPolicy: IfNotPresent

Even if you restart the Pod, the node reuses its cached latest copy. To force a pull every time, you’d set imagePullPolicy: Always.

3. You edited something outside the template area
------------------------------------------------------
What triggers a rollout: only changes under spec.template (labels, container images, env vars inside the Pod spec).

spec:
  replicas: 5            # changing this just scales, doesn’t restart existing Pods
metadata:
  annotations:
    note: "updated"      # this is on the Deployment itself, not the Pod template

Say you have changed replicas count or anything else which does not fall under new templates. then restart wont happen until unless pod template is changed

How to force the update you intended
---------------------------------------
1- Change the image tag in spec.template.spec.containers[*].image to a new value (e.g. myapp:v2.0).
2- Or explicitly restart all Pods without changing the template:
kubectl rollout restart deployment myapp

3- Ensure your imagePullPolicy is Always (or you use unique, immutable tags) so the node actually fetches the new image.

That way, Kubernetes sees a real template change and pulls the updated image, then gracefully replaces your old Pods with the new version.


```
---

## **3. "You applied a faulty Deployment spec. How would you rollback if `kubectl` isn't working?"**

✅ **Answer:**

```
If you can’t use kubectl for some reason (CLI broken, control-plane access limited, etc.), you still have a few ways to roll back a bad Deployment spec:

1. Use Helm (if you deployed with Helm)
-----------------------------------------
If your Deployment was managed by Helm, you can revert with a single command—even without kubectl rollout:
helm rollback <release-name> <revision-number>
That tells Tiller (the Helm server-side) to restore the chart to a known good release.

2. Call the Kubernetes API directly
--------------------------------------
Every kubectl command is really just an HTTP call to the API server under the hood. You can mimic it with curl:

A- Grab a service-account token from any Pod on the cluster:

TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
APISERVER="https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT"

B- Fetch your old ReplicaSet’s Pod template (the revision you want to roll back to):

curl -k \
  -H "Authorization: Bearer $TOKEN" \
  $APISERVER/apis/apps/v1/namespaces/my-ns/replicasets?labelSelector=deployment=myapp \
  | jq '.items[] | select(.metadata.annotations."deployment.kubernetes.io/revision"=="2") | .spec.template' \
  > old-template.json

C- Patch your Deployment back to that template:

curl -k -X PATCH \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/strategic-merge-patch+json" \
  -d '{"spec":{"template":'"$(cat old-template.json)"'}}' \
  $APISERVER/apis/apps/v1/namespaces/my-ns/deployments/myapp

That replaces the live Deployment spec with the saved revision, triggering the controller to roll out pods from the old template.

3. Apply a known-good YAML from Git (GitOps)
----------------------------------------------
If you keep your manifests in Git (GitOps style), simply:
A- Check out the last working commit/branch.
B- Re-apply the old Deployment file via your CI/CD pipeline or raw API call.
Because the spec has changed back, the Deployment controller will do a normal rolling update to restore the previous state.

4. Use the Kubernetes Dashboard or Lens
------------------------------------------------
If your CLI is busted but you still have UI access:
A- Open the Kubernetes Dashboard (or Lens, or any other GUI).
B- Navigate to Deployments → myapp → Revision History.
C- Click Rollback on the revision you want.

Behind the scenes the UI issues the same API calls as kubectl rollout undo


```



---

## **4. "When I change something in the Deployment (e.g., new image), who ensures the right pods are running?"**
✅ **Answer:**

**The Deployment creates and manages the ReplicaSet**, but the **ReplicaSet is the one directly responsible** for:
* Keeping the desired number of pods running
* Re-creating pods if they crash or are deleted
So:
> ✅ **ReplicaSet is the object that directly manages the Pods.**
> 🔁 Deployment manages the lifecycle of ReplicaSets.

🧠 Example:

1. You run:
```bash
kubectl set image deployment myapp myapp=myapp:v2
```

2. Kubernetes does:
   * Creates a **new ReplicaSet** for v2
   * Scales down the **old ReplicaSet**
   * The new ReplicaSet launches new pods

3. If pods crash → the **ReplicaSet** recreates them

✅ Interview-Ready Answer:

> When I update a Deployment, it creates a new ReplicaSet version. The ReplicaSet is the object that directly manages the pods (ensuring the right number, restarting if needed). The Deployment itself manages rollout and rollback by controlling its ReplicaSets.

---

## **5. "What happens if I delete a Deployment manually — do pods stay?"**

✅ **Answer:**

✅ **Simplified Explanation:**

When you delete a Deployment in Kubernetes:

* It **automatically deletes** the ReplicaSet linked to it
* And the ReplicaSet then deletes the pods it created

So usually, the **pods also go away** — though you might still see them for a few seconds because Kubernetes shuts them down gracefully.

---

But if you delete the Deployment using a special flag like this:

```bash
kubectl delete deployment myapp --cascade=orphan
```

Then the Deployment is deleted, but the **ReplicaSet is left behind**.

In that case:

* The old ReplicaSet is still active
* It **can keep recreating pods** if they crash or are deleted

🧠 Think of it like this:
> Delete Deployment → normally deletes everything below it (ReplicaSet + Pods)
> Use `--orphan` → only deletes the Deployment, leaves the rest running



## **6. "Why might a Deployment rollback silently fail?"**

✅ **Answer:**

A rollback might silently fail if there's nothing to roll back to, or if the previous revision is identical to the current one. Kubernetes won’t roll out again unless it detects an actual change in the Deployment’s pod template.

✅ **Example Scenario: Silent Rollback Failure**

You have a Deployment running this container:

```yaml
spec:
  template:
    spec:
      containers:
        - name: app
          image: myapp:v1
```

#### 🔁 Step 1: You update the image to a new version

```kubectl set image deployment myapp app=myapp:v2```

✅ Kubernetes:
* Creates a new ReplicaSet
* Rolls out new pods using `myapp:v2`

#### 🔄 Step 2: You try to rollback
```kubectl rollout undo deployment myapp```
➡ This works. Now you're back to `myapp:v1`.

#### 🔁 Step 3: You **try to rollback again**
``` kubectl rollout undo deployment myapp```

❌ Now nothing happens. Why? What’s really going on:
Kubernetes compares the current pod template (with `myapp:v1`) to the previous revision — **which is also `myapp:v1`**. Since there’s **no change**, Kubernetes **does not trigger a rollout**.

> This is called a **no-op rollback** — rollback command ran, but the Deployment stayed the same.


#### 🧠 Another common case:

You apply a bad config once, then fix it manually (not via rollout), and now try to undo. But Kubernetes can’t find any previous version to go back to — so rollback does nothing.
Also happens if: ```revisionHistoryLimit: 1```
Old revisions are **garbage collected**, and rollback fails silently.

#### ✅ Final Summary:
* A Deployment rollback may silently fail when Kubernetes sees no meaningful change to apply — usually because:
 * You're already on the same version
 * There's no previous revision
 * History was deleted

---

## **7. "Can you control how many Pods are updated at once?"**

✅ **Answer:**
Yes—you control the “batch size” of a rolling update using the Deployment’s rollingUpdate strategy, via two knobs:
```
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1   # how many Pods can be down at once
      maxSurge:       2   # how many extra Pods above spec.replicas you allow during the update
```

##### maxUnavailable
- Can be an absolute number (e.g. 1) or a percentage (e.g. 10%).
- If you have 5 replicas and maxUnavailable: 1, then Kubernetes will ensure at least 4 Pods remain Ready while it replaces 1 at a time.

##### maxSurge
- Also an absolute or percentage.
- If you have 5 replicas and maxSurge: 2, Kubernetes may run up to 7 Pods temporarily (5 desired + 2 surge) so it can start 2 new Pods before taking down 2 old ones.

```
Examples

1- One-at-a-time updates
-------------------------
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge:       0
→ Always keep your replica count steady, replacing Pods one by one.

2-Faster, with some extra capacity
----------------------------------

strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge:       1
→ You’ll see at most 6 Pods for a moment (if you have 5 replicas): start the 6th, wait for it Ready, then delete one old, and repeat.

3- Bulk replace

strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 50%
    maxSurge:       50%
→ If you have 10 replicas, you can replace 5 at once, running up to 15 total temporarily.


By tuning maxUnavailable and maxSurge, you balance speed of rollout against service availability and cluster capacity.

```

---

## 8. "Does `kubectl delete pods` affect a Deployment?

No permanent effect — the Deployment’s **ReplicaSet recreates** the pods immediately. This is part of the self-healing behavior.
but agr delete deployement kra then deployement delete hoga wo replicaset ko delete krega and in turn pods ko 
and ya to replicaset ko 0 kro 

---

## **9. "Can I manually edit a ReplicaSet created by a Deployment?"**

✅ **Answer:**
You can, but you shouldn't — because the Deployment controller will overwrite your changes.

#### 🔍 Detailed Explanation:
* When you create a Deployment, Kubernetes automatically creates and manages a ReplicaSet underneath it.
* This ReplicaSet:
  * Is owned by the Deployment
  * Gets its spec (template, labels, etc.) directly from the Deployment
* If you try to manually edit the ReplicaSet, for example ```kubectl edit replicaset <name>```
* And change:
    * The pod image
    * The number of replicas
    * The pod labels or annotations
* Then: ❌ Your change will not persist.
* ✅ The Deployment controller will revert it back during its sync loop, because it expects full control over the ReplicaSet.

#### 🧠 Real-World Use Case:
* The only safe edits you can make are non-pod-affecting fields, like maybe metadata annotations, but even those might be overwritten.
* If you want to change how pods behave: ✅ Edit the Deployment, not the ReplicaSet.

#### ✅ Interview-Ready Answer:
No, I shouldn’t manually edit a ReplicaSet created by a Deployment. The Deployment controller manages it and will override any manual changes. If I need to change pod behavior, I should edit the Deployment instead.

=================================================================================

# Stateful 

A StatefulSet is a Kubernetes controller used to manage stateful applications that need:
* Stable network IDs (like pod-0, pod-1, ...)
* Stable persistent storage per pod
* Ordered startup and shutdown (one at a time)

#### ✅ Key Features:
| Feature               | Description                                                                                |
| --------------------- | ------------------------------------------------------------------------------------------ |
| Stable Pod names      | Pods named as `app-0`, `app-1`, etc. (not random)                                          |
| Sticky storage        | Each pod gets its own PVC (`claim-0`, `claim-1`), which is **not deleted** when the pod is |
| Ordered rollout       | Pods start **and stop in order**, not in parallel                                          |
| Headless service req. | Uses `clusterIP: None` to allow direct DNS (`pod-0.service.namespace.svc`)                 |

#### 🔧 Example Use Cases:
* Databases like Cassandra, MongoDB, PostgreSQL
* Zookeeper, Kafka brokers
* Applications needing fixed identity or persistent data

#### 🎯 Twisted Interview Questions & Answers

❓1. "Can I scale a StatefulSet down from 3 to 1? What happens to PVCs of pod-2 and pod-1?"
* ✅ Answer: 
```
Yes—you can absolutely scale a StatefulSet down from 3 replicas to 1. Here’s what Kubernetes does step by step, and what happens to the PVCs:

1. Pod scale-down order
-----------------------------
StatefulSets always remove pods in reverse ordinal order (highest index first):
- You had myapp-0, myapp-1, myapp-2.
- Scaling to 1 → Kubernetes deletes myapp-2 first, then myapp-1, and leaves myapp-0 running.

2. PVCs are not deleted by default
-------------------------------------
 Each pod in a StatefulSet gets its own PVC, typically named something like data-myapp-0, data-myapp-1, data-myapp-2. When the pods are torn down:
  - myapp-2 and myapp-1 pods go away
  - Their PVC objects remain in the namespace, still bound to their underlying PVs
  - You’ll see those PVCs sitting there (e.g. kubectl get pvc shows data-myapp-2, data-myapp-1) in Bound state, even though no pod is using them

3. Why PVCs stick around
----------------------------
Stateful workloads often need their data preserved if you later scale back up or reattach those volumes manually. Kubernetes leaves the PVCs so you don’t accidentally lose your state.

4. Cleaning up old PVCs
---------------------
If you really want to garbage-collect those volumes when a pod is removed, you have to:
- Manually delete the PVCs (kubectl delete pvc data-myapp-2 data-myapp-1), or
- Use a VolumeClaimDeletion finalizer via a CSI driver that supports automatic PVC/PV cleanup, or
- Write a small controller or script to watch for scaled-down pods and clean up their PVCs.

In summary
- Scale down → pods 2 then 1 get deleted.
- PVCs remain (data preserved) unless you explicitly delete them or use a dynamic cleanup mechanism.

```

---

❓ 2. Why does StatefulSet have a `replicas` field? OR kya statefulset me bhi replica set hota hai ?
* ✅ Answer: 

```
The `replicas` field in a StatefulSet **controls how many pods to create**, just like in a Deployment — **but with identity and ordering**.

🔁 StatefulSet vs Deployment:
----------------------------------
| Feature                  | Deployment                      | StatefulSet                                        |
| ------------------------ | ------------------------------- | -------------------------------------------------- |
| Pods have fixed names?   | ❌ No (`random-xyz`)             | ✅ Yes (`myapp-0`, `myapp-1`, etc.)                 |
| Ordered startup?         | ❌ No                            | ✅ Yes (starts `-0`, then `-1`, etc.)               |
| Persistent storage?      | Optional                        | ✅ Usually uses **one PVC per pod** (e.g. `data-0`) |
| Replica count (replicas) | ✅ Controls how many pods to run | ✅ Same, but pods have **stable identity**          |

What does `replicas` actually do in a StatefulSet?
--------------------------------------------------
It tells Kubernetes:  “I want exactly this many **uniquely identified pods** (`myapp-0`, `myapp-1`, ..., `myapp-N`) running, in order.”
So:
* `replicas: 3` → creates `myapp-0`, `myapp-1`, `myapp-2`
* If `myapp-1` crashes → it’s restarted with same identity + PVC
* Scaling to 1 → Kubernetes deletes `myapp-2`, `myapp-1` but **keeps their PVCs**

Why is this useful?
--------------------
Stateful apps like **databases, Zookeeper, Kafka, etc.** need:
* Persistent identity** (pod name matters)
* Stable storage** (linked PVCs)
* Orderly booting/shutdown

Summary (interview-friendly):
------------------------------
> The `replicas` field in a StatefulSet defines how many **ordered, uniquely identified pods** to run. Unlike a Deployment, each replica in a StatefulSet has a **persistent identity and storage**, which is critical for stateful workloads like databases.

```
---

❓ 3. Can I delete `pod-0` manually in a StatefulSet?

✅ **Yes,** you can: ```kubectl delete pod myapp-0```

Kubernetes will:
* Automatically recreate `myapp-0`
* Reattach its **existing PVC**

But ⚠️ **don’t delete the PVC** manually:

* PVC holds the pod’s **data**
* Deleting it means the new pod-0 **starts empty**

🧠 Bonus Tip:

To simulate a crash: ```kubectl delete pod myapp-0 --grace-period=0 --force```. It will still come back with the **same name and data**, as long as the PVC is intact.

✅ Interview-ready line:
> StatefulSet uses `replicas` to define how many uniquely named pods should run. You can delete individual pods like `pod-0`, and Kubernetes will recreate them with the same identity and volume — preserving their state.

---

❓4. "Why do StatefulSets need a headless service?"
*  ✅ Answer:
```
A Headless Service (i.e., a Service with clusterIP: None) is required by StatefulSets to give each pod a stable DNS hostname.

What is a Headless Service?
----------------------------------
Normally, a Kubernetes Service gives:
- One stable ClusterIP
- And load balances traffic across matching pods
But a Headless Service does:
- ❌ No ClusterIP (clusterIP: None)
- ❌ No load balancing

🧩 Why is this useful for StatefulSets?
---------------------------------------
StatefulSets need:
- Stable pod names like mysql-0, mysql-1, etc.
- And stable DNS names like:
mysql-0.mysql-headless.default.svc.cluster.local
mysql-1.mysql-headless.default.svc.cluster.local
That’s exactly what a headless service gives — per-pod DNS resolution.

🧠 Real Use Case
-----------------------
Imagine you're running a Kafka or MongoDB cluster:
- Each replica must know how to reach pod-0, pod-1, pod-2 by fixed address
- A normal service would load balance — that's bad for stateful apps!
- A headless service ensures each pod has a stable, direct hostname

📝 YAML Example
---------------------
apiVersion: v1
kind: Service
metadata:
  name: mongo
spec:
  clusterIP: None   # 👈 makes it headless
  selector:
    app: mongo
  ports:
  - port: 27017

In this case, pods like mongo-0 can talk to:
-------------------------------------------
mongo-1.mongo.default.svc.cluster.local

✅ Summary (Interview Friendly)
---------------------------------
StatefulSets use a Headless Service to assign stable DNS names to each pod. This is critical for stateful apps (like databases or brokers) that require stable identity and direct addressing for replication, clustering, or peer discovery.
```

---



```
NOTE:  HEADLESS SERVICE means "cluster IP : none" .

### 💡 **What is a Headless Service?**
> A **headless service** is just a Kubernetes Service with: ```clusterIP: None```
This tells Kubernetes:
❌ Don’t assign a virtual IP
✅ Just publish **DNS records of all backend pods**

 🤝 In context of controllers:
-----------------------------------------

| Controller      | Headless Service | Required?    | Behavior                                                                   |
| --------------- | ---------------- | ------------ | -------------------------------------------------------------------------- |
| **StatefulSet** | ✅ Yes (required) | ✅ Required   | Needed for **stable DNS per pod** like `pod-0.svc`                         |
| **Deployment**  | ✅ Optional       | ❌ Not needed | Default service gives 1 IP (load-balanced), headless gives **all pod IPs** |


### 🧪 `dig` / `nslookup` Behavior:
----------------------------------------

A. ▶️ Normal Service (with clusterIP): ```dig myapp.default.svc.cluster.local```
➡ Returns **1 IP only** (load balancer / kube-proxy does round-robin behind it)

B. ▶️ Headless Service (with clusterIP: None): ```dig myapp-headless.default.svc.cluster.local```
➡ Returns **all backend pod IPs**

 ✅ Final Summary (One-liner):
--------------------------------
> `clusterIP: None` makes a Service headless. It’s **required in StatefulSet** to give each pod a DNS name. In Deployments, it’s **optional** and only needed if you want to **access all pod IPs individually** (no load balancing).
```

---

❓5. "How does rolling update work in StatefulSets?"
* ✅ Answer: Pods are updated one at a time, in order (pod-0, then pod-1, ...). Each pod must be ready before the next one updates. Ensures consistency and safety for stateful workloads.

---

❓6. "Can StatefulSet guarantee data recovery after a crash?"
* ✅ Answer: Only if PVCs are backed by durable storage (e.g., EBS, GCE PD).
Kubernetes won’t delete the PVCs, but data durability is storage-dependent.

=================================================================================

#  PV & PVC 

✅ What is a PersistentVolume (PV)?
-------------------------------------
* A PV is a **cluster-wide** storage resource that’s provisioned by an admin(static) or dynamic storage class.
* It represents actual storage (like EBS, NFS, GCE disk) ```kind: PersistentVolume```

✅ What is a PersistentVolumeClaim (PVC)?
-----------------------------------------
* A PVC is a user request for storage.
* It says: “I want 10Gi of storage with ReadWriteOnce access.”
``` 
kind: PersistentVolumeClaim
```
* PVC is matched (bound) to a suitable PV behind the scenes.

✅ Real-life Analogy (technical):
-----------------------------------
* PV = physical hard disk available in cluster
* PVC = request to use a portion of it
* Pod = mounts PVC → accesses storage

✅ Workflow
--------------
``` Pod → PVC → StorageClass → CSI Driver → PV → EBS ```

#### 1. Pod declares a volume using a PVC
```
volumes:
  - name: my-vol
    persistentVolumeClaim:
      claimName: mypvc
```
* Pod just says: “I want the PVC named mypvc.” This Pod wants to use a volume that is already claimed by the PVC named mypvc.

#### 2. PVC is a storage request
```
kind: PersistentVolumeClaim
metadata:
  name: mypvc
spec:
  storageClassName: gp3      # uses EBS CSI driver
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 5G
```
* PVC says: “I need 5Gi storage, provisioned via gp3 StorageClass.”

#### 3. StorageClass → triggers CSI driver
```
kind: StorageClass
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
```
* ✅ The ebs.csi.aws.com driver is called → it creates an EBS volume

#### 4. CSI Driver creates actual storage (EBS)
* Behind the scenes:
  * AWS EBS volume is created (e.g., vol-0a123...)
  * Kubernetes auto-creates a PersistentVolume (PV) for it
  * The PV looks like:
```
kind: PersistentVolume
spec:
  capacity:
    storage: 5Gi
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-0a123456
    fsType: ext4
```
#### 5. PV binds to the PVC
* Kubernetes links the auto-created PV to the pending PVC
* → PVC status = Bound

#### 6. Pod mounts the volume
```
volumeMounts:
  - name: my-vol
    mountPath: /data
```
* Pod now sees EBS volume mounted at /data, and data is persistent across pod restarts or node failures.

✅ Final Visual (Cloud Dynamic Case)
```
Pod
 └─> PVC
      └─> StorageClass (gp3)
            └─> CSI Driver (ebs.csi.aws.com)
                  └─> EBS volume (AWS)
                        └─> PV (auto-created)

```

---

## Interview Questions on PV & PVC 

❓1. "What happens when a pod using a PVC is deleted?"
✅ Answer:
- The PVC stays (unless explicitly deleted).
- The PV stays, but whether it’s reused or deleted depends on reclaim policy:
   * Retain: PV stays, must be cleaned manually
   * Delete: PV & underlying storage auto-deleted

---

❓2. "Can multiple pods use the same PVC?"
✅ Answer:
```
It depends on the access mode and the underlying storage type.

🔓 PVC Access Modes
| Access Mode     | Meaning                                                              |
| --------------- | -------------------------------------------------------------------- |
| `ReadWriteOnce` | Only **one Pod** on **one node** can mount the volume **read-write** |
| `ReadOnlyMany`  | Many Pods can mount it, but only **read-only**                       |
| `ReadWriteMany` | Many Pods across nodes can **read and write** at the same time       |

📦 Storage Types Comparison
| Storage Type | ReadWriteMany Supported? | Multi-Pod Access? | Best Use Case                 |
| ------------ | ------------------------ | ----------------- | ----------------------------- |
| **EBS**      | ❌ No                     | ❌ Not recommended | Single-Pod storage per node   |
| **EFS**      | ✅ Yes                    | ✅ Yes             | Shared data across Pods/nodes |


 So What's the Catch?
 ------------------------

🚫 Amazon EBS (gp2/gp3)
--------------------
- Is block storage.
- Can only be attached to one EC2 instance at a time.
- Even if you try to mount it on multiple Pods, only one will succeed (usually).
- Access mode = ReadWriteOnce only.

✅ Amazon EFS
--------------------
- Is network file system (NFS).
- Can be mounted by many Pods across different nodes.
- Supports ReadWriteMany, so ideal for shared files (e.g., WordPress uploads, logs).
- Great for multi-tenant apps, ML pipelines, file servers.
```

---

❓3. "What happens if there’s no matching PV for a PVC?"
✅ Answer:
```
If a PersistentVolumeClaim (PVC) is created, but no matching PersistentVolume (PV) is available, here’s what happens step by step:

Kubernetes Workflow
----------------------
1- You create a PVC

accessModes:
  - ReadWriteOnce
resources:
  requests:
    storage: 10Gi

2- Kubernetes looks for a matching PV

It checks:
- Storage size (>= requested)
- Access mode (e.g. ReadWriteOnce)
- StorageClass (if specified)
 
3- If no matching PV is found:
- The PVC will stay in Pending state.
- No Pod using that PVC can start.
- You can see this with:
kubectl get pvc

4- Pod will also stay Pending
Because the Pod can't mount a volume that doesn't exist.

✅ Solutions
----------------
If using Dynamic Provisioning (via StorageClass):
  - Make sure the PVC has a valid storageClassName
  - The underlying provisioner (e.g., EBS CSI driver) must be installed
  - Kubernetes will create a new PV dynamically

If using Static PVs:
  - Manually create a PV that matches:
       - Size
       - Access mode
       - StorageClass (if set)

🔁 Summary
If there's no PV matching the PVC's request, the PVC stays Pending, and any Pods using it won’t start. Fix this by creating a matching PV or ensuring a dynamic provisioner is available with the correct StorageClass.

```

---

❓4. "Can a PV be reused after being released?"
```
Yes — but it depends on its reclaim policy.

🧩 1. What does "Released" mean?
-------------------------------
When a PVC (PersistentVolumeClaim) is deleted, the PV it was bound to may move to the Released phase — meaning:
“This PV is no longer claimed by a PVC, but the data still exists on the volume.”

🧾 2. Reclaim Policy Determines What Happens Next
--------------------------------------------------
| Reclaim Policy           | What Happens                                                      | Reusable Without Manual Cleanup? |
| ------------------------ | ----------------------------------------------------------------- | -------------------------------- |
| `Retain`                 | PV enters `Released`. Admin must manually clean and rebind it.    | ❌ Manual cleanup needed          |
| `Recycle` *(deprecated)* | Old data is scrubbed and PV is reused.                            | ✅ *(Legacy only)*                |
| `Delete`                 | The associated storage resource (e.g. EBS volume) is **deleted**. | ❌ Not reusable — volume gone     |

✅ 3. How to Reuse a Retained PV
----------------------------------------
If a PV is in Released state with Retain policy:

Steps:
-----------
A- Manually delete data on the backend disk (optional, but recommended).
B- Remove the old claimRef from the PV definition (this is key!): claimRef: null
C- Rebind it to a new PVC with matching size, access mode, and StorageClass.

🛑 Why Is This Not Automatic?
-------------------------------
Kubernetes does not automatically reuse Retain volumes to avoid data leakage between tenants or applications.

🧠 Summary
-----------
A PV can be reused after being released if its reclaim policy is Retain, but only after manual cleanup and unbinding. For automatic cleanup, use reclaim policy Delete with dynamic provisioning.

```

---

❓5. Is EBS CSI driver a PV?
✅ Answer:
The EBS CSI driver is not a PV, but a plugin that enables Kubernetes to dynamically create and manage EBS-backed PVs using PVCs. It’s part of the CSI standard and replaces the older in-tree AWS volume plugin.

---

❓6. "Does deleting a PVC delete data?"
✅ Answer:
* Only if the PV’s reclaimPolicy is set to Delete.
* If it’s Retain, data stays — even if PVC is deleted.

========================================================================================

# STORAGE CLASS

#### 📦 What is a **StorageClass** in Kubernetes?
> A **StorageClass** defines how **dynamic storage** should be **provisioned automatically** in Kubernetes.
> It tells Kubernetes:

* **Which storage plugin (provisioner)** to use
* **What type of disk** (e.g. gp3, standard, SSD)
* **How to create it** (parameters like fsType, encryption)

#### ✅ Why it's needed?

Without a StorageClass:

* Admin must create PersistentVolumes (PVs) **manually**
* No automation — error-prone and not scalable

With StorageClass:

* When you create a PVC, it **auto-creates a matching PV**
* Backed by cloud volumes like **EBS (AWS)**, **PD (GCP)**, **Disk (Azure)**, **NFS**, etc.


#### 🧠 Key Fields in StorageClass
| Field               | Purpose                                               |
| ------------------- | ----------------------------------------------------- |
| `provisioner`       | CSI driver to call (e.g., `ebs.csi.aws.com`)          |
| `parameters`        | Extra options (e.g., `type: gp3`, `fsType: ext4`)     |
| `reclaimPolicy`     | What to do when PVC is deleted (`Delete` or `Retain`) |
| `volumeBindingMode` | When to bind (e.g., `WaitForFirstConsumer`)           |


#### 📄 Sample StorageClass YAML (for AWS EBS)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

#### 🎯 Twisted Interview Questions

❓1. Can I have multiple StorageClasses?
> ✅ Yes. Each can provision a different type of disk (e.g., SSD, HDD, encrypted).

❓2. What if I don’t specify `storageClassName` in PVC?
> If a **default StorageClass** exists, it is used automatically.

❓3. What is `volumeBindingMode: WaitForFirstConsumer`?
> It delays volume creation **until a pod is scheduled**, so that Kubernetes knows which **zone** to provision the disk in (important for **zonal storage like EBS**).

#### ✅ Final Interview Summary:
> A **StorageClass** automates the creation of PersistentVolumes by defining **how to dynamically provision storage** using CSI drivers. It's like a storage blueprint for your PVCs.

=====================================================================================

# SERVICE ACCOUNT

#### 🧑‍💻 What is a **Service Account (SA)** in Kubernetes?
> A **ServiceAccount** is an identity used by **Pods** to interact securely with the **Kubernetes API server** or external services.

#### ✅ Why is it used?
* To **authenticate** Pods to the API Server
* To provide **fine-grained permissions** using RBAC (e.g., can read Secrets, can't delete Pods)
* Automatically **injects tokens** into running pods (`/var/run/secrets/kubernetes.io/serviceaccount/`)

#### 🧠 Key Points
| Feature       | Detail                                                                 |
| ------------- | ---------------------------------------------------------------------- |
| Default SA    | Each namespace has a built-in `default` ServiceAccount                 |
| Token Mount   | Pods get a JWT token automatically mounted inside                      |
| Permissions   | Defined using `Role` or `ClusterRole` + `RoleBinding`                  |
| Custom SA     | You can create your own and assign to specific Pods                    |
| Secure access | Used for access to secrets, configmaps, or calling API Server securely |

#### 📄 Example Workflow

### ✅ Step 1: Create a ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: reader
```
* This creates a ServiceAccount named reader.
* By default, Pods run using the default ServiceAccount in their namespace.
* Here, you're creating a custom identity (reader) to be used later by a Pod.


---

### ✅ Step 2: Create Role + RoleBinding

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-secrets
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
```

* This Role is scoped within the namespace (not cluster-wide).
* t allows get and list access on secrets only.
* apiGroups: [""] means it applies to the core API group, which includes secrets, pods, configmaps, etc.

---

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: bind-reader
subjects:
- kind: ServiceAccount
  name: reader
roleRef:
  kind: Role
  name: read-secrets
  apiGroup: rbac.authorization.k8s.io
```
* This RoleBinding links the reader ServiceAccount with the read-secrets Role.
* Only Pods using the reader ServiceAccount can now use the permissions from read-secrets.
* (mtlb rolebinding me hmne reader [service account] and read-secrets [role]se attach kr diya h )

---

### 3. Attach it to a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  serviceAccountName: reader   # 👈 assign SA to Pod
  containers:
  - name: app
    image: my-image
```
* This Pod explicitly declares serviceAccountName: reader.
* When the Pod is scheduled, Kubernetes attaches the ServiceAccount’s token to the Pod.
* Now, the app inside the container can authenticate to the Kubernetes API as reader.

### 🔐 Result: What Does the Pod Gain?
Pod mypod is allowed to:
- Access Kubernetes Secrets (via API calls)
- e.g., from code: kubectl get secrets or using K8s Python/Go SDK
- Only within its namespace, and only read operations (get, list)
- No access to other resources like Pods, Deployments, etc.

### 🧠 Why Use This?
| Reason                            | Benefit                                      |
| --------------------------------- | -------------------------------------------- |
| Principle of Least Privilege      | Only give exact permissions needed           |
| Fine-grained access control       | Restrict access to specific resource types   |
| ServiceAccount token auto-mounted | Apps can authenticate to Kubernetes securely |
| Namespace isolation               | Roles don't leak into other namespaces       |


```
❓ Why can’t we attach a Role directly to a Pod?
* Because in Kubernetes: 
  * 🔐 RBAC is designed to work with identities — not resources.
  * and Pods are not identities — ServiceAccounts are.

🔍 Core Reason:
* Roles and ClusterRoles define what an identity (user or ServiceAccount) can do.
* Pods don't have identity themselves.
* So you can't say: "This Pod can read secrets."
* Instead, you say: "This ServiceAccount (used by the Pod) can read secrets."

🔧 Bonus: Why not attach Role to Pod directly?
* Pods are ephemeral — they can be deleted/restarted any time.
* RBAC needs a stable, reusable identity — ServiceAccount provides that.
* Reusing a RoleBinding for many Pods becomes clean and maintainable.

```

---

## 🎯 EXAMPLE

### 🔍 Use Case:
> You have a **backend service** running inside a Pod that needs to:
  * Access a **Secret** in the cluster (e.g., DB credentials)
  * Read it directly using the **Kubernetes API** (not via mounted volume)

#### 🔁 What Happens:

##### 🔸 **1. App makes an HTTP call to the API Server**:
```http
GET /api/v1/namespaces/default/secrets/db-secret
Authorization: Bearer <token>
```

##### 🔸 **2. Where does the token come from?**
* Kubernetes automatically mounts a JWT **ServiceAccount token** inside the Pod at:

```
/var/run/secrets/kubernetes.io/serviceaccount/token
```

* Your app reads this token and includes it in the Authorization header.

##### 🔸 **3. If you're using the default ServiceAccount...**
* ❌ Access is **denied** (403 Forbidden)
* Because **default SA has no RBAC permissions**.

##### 🔸 **4. You create a custom ServiceAccount with a Role:**
* You define a `read-secrets` role (with access to Secrets)
* You bind it to a new ServiceAccount: `backend-reader`
* You assign this SA to your Pod using:

```yaml
serviceAccountName: backend-reader
```

Now your app’s in-cluster API request **succeeds**, because the token it uses is tied to a ServiceAccount **with proper permissions**.

## 🧪 Real-World Examples

| Use Case                                | Why ServiceAccount is Needed                          |
| --------------------------------------- | ----------------------------------------------------- |
| Jenkins Pod accessing K8s to deploy     | Jenkins needs SA token + Role to call `kubectl apply` |
| ArgoCD syncing apps                     | ArgoCD uses SA to interact with K8s manifests         |
| Fluentd reading logs                    | Needs access to Pod logs or container metadata        |
| Custom app fetching Secrets at runtime  | Needs RBAC + SA token to query K8s Secrets API        |
| K8s Job creating ConfigMaps dynamically | Needs Role to create/update ConfigMaps                |

#### ✅ Final Takeaway (One-liner):
> **ServiceAccounts are how Pods prove their identity inside the cluster.** With RBAC, they define what that Pod can do — like accessing Secrets, listing Pods, or applying configs.


#### 🎯 Common Interview Questions

❓1. What’s the difference between a user and a service account?
> ✅ User = human identity (external), ServiceAccount = used by pods

❓2. Can a ServiceAccount access all API resources by default?
> ✅ No. It has **no privileges** unless you bind a Role or ClusterRole

❓3. What happens if I don’t specify `serviceAccountName` in a pod?
> ✅ Kubernetes assigns the **default SA** of the namespace

#### ✅ Final Summary (Interview Line):
> A ServiceAccount provides **Pod-level identity** in Kubernetes, used to access the API securely. It works with **RBAC** for fine-grained permissions and is mounted as a token inside the container.

========================================================================

# ROLE , ROLE BINDING , CLUSTER ROLE , CLUSTER ROLE BINDING 

All four are used in **RBAC (Role-Based Access Control)** in Kubernetes to manage **who can do what**.

---

## 🧱 1. **Role**

> Defines **permissions** within a **specific namespace**

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: pod-reader
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

✅ Use when: You want to **limit access to one namespace** only.

---

## 🔗 2. **RoleBinding**

> Binds a **Role** to a **user, group, or ServiceAccount** — **in that same namespace**

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: bind-pod-reader
  namespace: dev
subjects:
- kind: ServiceAccount
  name: backend-app
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

✅ Use when: You want to apply the Role to a subject **in one namespace**.

---

## 🌍 3. **ClusterRole**

> Like `Role`, but:

* It’s **cluster-wide**
* Can grant access to **non-namespaced resources** (e.g., nodes, persistent volumes)
* Can also be reused across many namespaces

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-nodes
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list"]
```

✅ Use when: You need access across **all namespaces** or to **cluster-level resources**.

---

## 🌐 4. **ClusterRoleBinding**

> Binds a `ClusterRole` to a subject across the **whole cluster**

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: bind-read-nodes
subjects:
- kind: ServiceAccount
  name: backend-app
  namespace: dev
roleRef:
  kind: ClusterRole
  name: read-nodes
  apiGroup: rbac.authorization.k8s.io
```

✅ Use when: You want to give **cluster-level access** to a user/SA.

---

## 🧠 Quick Comparison Table

| Type                 | Scope        | Can Access Cluster Resources? | Used For                                   |
| -------------------- | ------------ | ----------------------------- | ------------------------------------------ |
| `Role`               | Namespace    | ❌ No                          | Namespaced access only                     |
| `RoleBinding`        | Namespace    | ❌ No                          | Bind Role to SA/user in a namespace        |
| `ClusterRole`        | Cluster-wide | ✅ Yes                         | Access cluster resources / all namespaces  |
| `ClusterRoleBinding` | Cluster-wide | ✅ Yes                         | Bind ClusterRole to subject (cluster-wide) |

---

## ✅ Final Interview Summary

> **Roles** define what actions are allowed.
> **Bindings** assign those roles to identities.
> Use **Role + RoleBinding** for namespace access, and **ClusterRole + ClusterRoleBinding** for cluster-wide or non-namespaced resource access.

## EXAMPLE 

### ✅ 1. **Role + RoleBinding**
#### 🎯 Use Case:

A dev team working in the `dev` namespace should **only view Pods and Secrets** in that namespace.

#### 🔧 Implementation:

* ✅ Create a **Role** to allow `get`, `list` on Pods and Secrets.
* ✅ Bind that Role to their **ServiceAccount** using a **RoleBinding**.

```yaml
# Role (namespaced)
kind: Role
metadata:
  name: dev-reader
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods", "secrets"]
  verbs: ["get", "list"]
```

```yaml
# RoleBinding
kind: RoleBinding
metadata:
  name: bind-dev-reader
  namespace: dev
subjects:
- kind: ServiceAccount
  name: app-reader
roleRef:
  kind: Role
  name: dev-reader
```

> 🔒 Access is limited to **dev** namespace only.

---

### ✅ 2. **ClusterRole + ClusterRoleBinding**

#### 🎯 Use Case:

You’re running **Prometheus** or a **monitoring agent** that needs to **read all Node and Pod metrics** across the cluster.

#### 🔧 Implementation:

* ✅ Create a **ClusterRole** to allow access to `nodes`, `pods`, etc.
* ✅ Bind that ClusterRole to the ServiceAccount of Prometheus using **ClusterRoleBinding**.

```yaml
# ClusterRole
kind: ClusterRole
metadata:
  name: prometheus-reader
rules:
- apiGroups: [""]
  resources: ["nodes", "pods"]
  verbs: ["get", "list"]
```

```yaml
# ClusterRoleBinding
kind: ClusterRoleBinding
metadata:
  name: bind-prometheus-reader
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: prometheus-reader
```

> 🔓 Access applies to **all namespaces + cluster resources** like nodes.

---

### ✅ Hybrid Use Case: Reuse `ClusterRole` in namespace

You can bind a **ClusterRole** to a **RoleBinding** for use **in one namespace**.

#### 🎯 Example:

You want the same "pod-reader" access in every namespace.

* Instead of creating 10 Roles (one per namespace)…
* You create **one ClusterRole**
* And use **RoleBindings** in each namespace

---

#### 🎯 Final Takeaway (Interview line):

> Use `Role`/`RoleBinding` for namespaced use cases (e.g., dev apps accessing secrets), and `ClusterRole`/`ClusterRoleBinding` for cluster-wide tools (like Prometheus, cert-manager) or to access non-namespaced resources like nodes or volumes.

=====================================================================================

# INGRES & INGRES CONTROLLER


## 🌐 What is an **Ingress** in Kubernetes?

> An **Ingress** is an API object that defines **HTTP/HTTPS routing rules** to expose services to external users.

It lets you:

* Route traffic by **hostnames** or **paths**
* Use **TLS/SSL termination**
* Avoid exposing each service via NodePort or LoadBalancer

```
Q- How Ingress avoids exposing each service via NodePort or LoadBalancer?

## ⚠️ Without Ingress:

Every external-facing service needs **its own NodePort or LoadBalancer**.

### Example:

| Service        | Type         | External Access                            |
| -------------- | ------------ | ------------------------------------------ |
| `frontend-svc` | LoadBalancer | [http://LB-IP-1](http://LB-IP-1)           |
| `api-svc`      | LoadBalancer | [http://LB-IP-2](http://LB-IP-2)           |
| `auth-svc`     | NodePort     | [http://NodeIP:32001](http://NodeIP:32001) |

🧨 Result:

* **Wasted public IPs**
* **Hard to manage**
* **No hostname/path-based routing**

---

## ✅ With Ingress:

You expose **just ONE** external endpoint via the **Ingress Controller**, then define **routes inside Ingress rules**.

### Example:

```yaml
rules:
- host: myapp.com
  http:
    paths:
    - path: /frontend
      backend:
        service:
          name: frontend-svc
          port:
            number: 80
    - path: /api
      backend:
        service:
          name: api-svc
          port:
            number: 80


🔁 Now, all traffic goes like this:

http://myapp.com/frontend → frontend-svc
http://myapp.com/api     → api-svc


✅ Only **one LoadBalancer** (for the Ingress Controller)

## 🧠 Summary (Interview-ready line):

> Ingress avoids the need to expose each service with its own LoadBalancer or NodePort by **consolidating traffic through a single entry point**, using **path- or host-based routing** handled by the Ingress Controller.
```

---


## 🚦 What is an **Ingress Controller**?

> It is the **actual software (pod)** that watches Ingress objects and configures **reverse proxy rules** accordingly.
* 🧩 Ingress = rules
* 🧩 Ingress Controller = enforcement engine

**Popular Ingress Controllers:**
* `nginx-ingress-controller`
* `traefik`
* `HAProxy`
* `AWS ALB Ingress Controller`

---

## 🔁 Workflow

```
Client → DNS → Ingress Controller (Nginx) → Routes → Service → Pod
```

1. You deploy an Ingress Controller (e.g., NGINX) as a Pod
2. You write Ingress rules (like routing `app1.example.com → app1-svc`)
3. Controller picks up rules and applies them to its internal proxy
4. External traffic comes to LoadBalancer IP → hits Ingress Controller → routed to correct service

---

## 🔐 TLS Support

Ingress supports:

* HTTPS termination via `tls` block
* Store certificate in a `Secret`
* Automatically integrate with cert-manager for Let’s Encrypt

---

## 🧠 Common Interview Questions

### ❓1. What is the difference between Ingress and Ingress Controller?
> Ingress defines **rules**, Ingress Controller implements those rules.

---

### ❓2. Can you expose TCP/UDP using Ingress?
> No. Ingress is for **HTTP/HTTPS**. For TCP/UDP, use a `LoadBalancer` or `NodePort`.

---

### ❓3. How is Ingress different from LoadBalancer service?
> `LoadBalancer` exposes one service directly.
> `Ingress` can **consolidate many services** behind one external IP and route by path or host.

---

### ❓4. Do you need both Ingress and Ingress Controller?
> ✅ Yes. Ingress alone does nothing without a controller.

---

## ✅ Final Summary

> **Ingress** is the way to expose multiple Kubernetes services over HTTP(S) using clean host/path-based routing.
> You must deploy an **Ingress Controller** (like NGINX) to actually process and forward traffic.

Q. What happens if there is only an Ingress object and no Ingress Controller
* The Ingress rules are not processed — no routing, no external access.
* Kubernetes just stores the Ingress resource, but:
  * No IP is assigned
  * No traffic is forwarded
  * No error shown, but it silently doesn't work
  * Ingress Controller is mandatory to make Ingress functional.

Q. Is ingres controller part of controller manager ?
* No , The Controller Manager handles native K8s objects.
* The Ingress Controller is a separate pod you must deploy to handle Ingress logic.

==============================================================================

# CONFIG MAP & SECRETS

> Use `ConfigMap` for non-sensitive app settings, and `Secret` for anything confidential like passwords, tokens, or API keys.
> Both can be **mounted as env vars or files**, and help decouple app logic from configuration.


#### 📦 What is a `ConfigMap`?
* A `ConfigMap` is used to **store configuration data** (non-sensitive) as **key-value pairs**.

### ✅ Use case:

### CASE 1: when i want to pass environment variables.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_MODE: "production"
  LOG_LEVEL: "debug"
```

### 🔧 Mount example:

```yaml
env:
- name: APP_MODE
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: APP_MODE
```

### CASE 2: when i want to pass nginx conf file 

1️⃣ Create the ConfigMap containing the nginx.conf file
```yaml
kubectl create configmap nginx-config \
  --from-file=nginx.conf
```

**This will create a ConfigMap like:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    user  nginx;
    worker_processes  1;
    events {
        worker_connections 1024;
    }
    http {
        server {
            listen 80;
            location / {
                root /usr/share/nginx/html;
                index index.html;
            }
        }
    }
```

2️⃣ Mount the ConfigMap as a file in your Pod
```yaml
  volumeMounts:
    - name: nginx-config-volume
      mountPath: /etc/nginx/nginx.conf    # 👈 Replace default nginx.conf
      subPath: nginx.conf                 # 👈 Important to mount single file!
  volumes:
  - name: nginx-config-volume
    configMap:
      name: nginx-config


🔷 1. volumes:
----------------
Defines what external data source you want to bring into the Pod (like a ConfigMap, Secret, persistent disk, etc.)
Think of it as: "Where is the data coming from?"

🔷 2. volumeMounts:
--------------------
Defines where you want to place that volume inside the container’s filesystem.
Think of it as: "Where inside the container should this volume appear?"
```


---

## 🔐 What is a `Secret`?

A `Secret` stores **sensitive data** like passwords, tokens, or keys — **base64-encoded**.

### ✅ Use case:

Pass credentials to containers **securely**.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  DB_USER: YWRtaW4=      # "admin"
  DB_PASS: cGFzc3dvcmQ=  # "password"
```

### 🔧 Mount example:

```yaml
env:
- name: DB_USER
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: DB_USER
```

## 🔐 Goal: Use AWS Secrets Manager instead of Kubernetes Secret — and inject those secrets into the Pod securely.

#### ✅ Real-Life Flow Using External Secrets Operator (ESO)
```
Pod → ServiceAccount → IAM Role → AWS Secrets Manager → ESO → Kubernetes Secret → Mounted in Pod
```

#### 🔁 Step-by-step Breakdown:

##### 1. Create secret in AWS Secrets Manager
Example:
```
{
  "DB_USER": "admin",
  "DB_PASS": "s3cr3t"
}
```

##### 2. Create IAM Role with access to that secret
* Allow secretsmanager:GetSecretValue
* Trust relationship: bound to your EKS ServiceAccount (IRSA)
```
{
  "Effect": "Allow",
  "Action": "secretsmanager:GetSecretValue",
  "Resource": "arn:aws:secretsmanager:region:acct-id:secret:my-db-secret"
}
```

##### 3. Create a Kubernetes ServiceAccount
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<acct>:role/SecretsReaderRole
```

##### 4. Install External Secrets Operator (ESO) in the cluster
* It will:
  * Run a controller Pod
  * Periodically read from AWS Secrets Manager
  * Create/update corresponding Kubernetes Secret

##### 5. Define an ExternalSecret
```
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-creds
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-store
    kind: SecretStore
  target:
    name: db-secret                 # Kubernetes Secret created
  data:
  - secretKey: DB_USER              # Key in Kubernetes Secret
    remoteRef:
      key: my-db-secret             # AWS Secrets Manager name
      property: DB_USER
  - secretKey: DB_PASS
    remoteRef:
      key: my-db-secret
      property: DB_PASS
```

##### 6. Mount the generated Kubernetes Secret into your Pod
```
env:
- name: DB_USER
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: DB_USER
---
```

#### 🧠 Final Summary (Interview-ready):
> Yes, in production we use a Kubernetes ServiceAccount bound to an IAM Role (IRSA), which allows the External Secrets Operator to read AWS Secrets Manager values. Those are converted into Kubernetes Secrets and then mounted into Pods — ensuring secrets never need to be hardcoded or managed manually in Kubernetes.

================================================================================

# DAEMON SETS

## 🧱 What is a **DaemonSet**?

> A **DaemonSet** ensures **exactly one Pod runs on every node** (or some selected nodes) in the cluster.

### 🔁 If a new node is added:
→ A DaemonSet Pod is automatically scheduled on it.

### 🧽 If a node is removed:
→ The Pod is automatically cleaned up.

---

## ✅ Real-World Use Cases:

| Use Case                 | What the DaemonSet Pod does                         |
| ------------------------ | --------------------------------------------------- |
| 🧪 **Monitoring Agents** | e.g., Prometheus Node Exporter, Datadog agent, etc. |
| 📦 **Log Collectors**    | e.g., Fluentd, Filebeat, Logstash                   |
| 🔐 **Security Daemons**  | e.g., Falco or Trivy to scan workloads on each node |
| 🧰 **Storage plugins**   | e.g., CSI drivers that need node-level components   |
| 🧪 **Custom node tools** | Run system checkers or cleanup tasks node-by-node   |

---

## 🧾 Example DaemonSet YAML:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-agent
spec:
  selector:
    matchLabels:
      app: log-agent
  template:
    metadata:
      labels:
        app: log-agent
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd
```

---

## 🧠 Interview Questions

### ❓1. What happens when a new node is added?

> A DaemonSet Pod is automatically created on it.

### ❓2. How is DaemonSet different from Deployment?

> Deployment: replicas across the cluster
> DaemonSet: **1 Pod per node**, not tied to replicas

### ❓3. Can you run DaemonSet only on some nodes?

> ✅ Yes, use:

* `nodeSelector`
* `affinity`
* `tolerations` (for tainted nodes)

---

## ✅ Summary (Interview line):

> DaemonSets are used to **run 1 pod per node**, ideal for node-level agents like log collectors or monitoring tools. They automatically handle node joins/leaves and are essential for system-wide observability and security.

======================================================================================

# JOBS 


#### ⚙️ What is a **Job** in Kubernetes?
> 💡 A Job in Kubernetes is a controller that runs a one-time or short-lived task to completion. Unlike a Deployment (which keeps Pods running continuously), a Job runs a fixed number of Pods until they successfully complete, and then stops.

✅ Use Cases for a Job:
* Database migrations (Apply DB schema changes at startup )
* Batch processing (e.g., process a file, send an email, or do cleanup)
* One-time cron-like tasks
* Machine learning model training
* Certificate Renewal: Generate or renew SSL certs programmatically
* Send Emails or Reports: Trigger ad-hoc notification workflows
* Data Export: Export logs or backup data from volumes


Unlike Deployments or DaemonSets, Jobs:
* Run to **completion**
* Ensure the task **finishes successfully a specific number of times**
* Retry on failure based on `backoffLimit`

🔁 Difference from Other Controllers
| Controller     | Purpose                                | Lifespan                | Auto-Restarts                  |
| -------------- | -------------------------------------- | ----------------------- | ------------------------------ |
| **Pod**        | One unit of execution                  | Manual                  | No                             |
| **Deployment** | Runs and manages **long-running Pods** | Forever (desired state) | Yes                            |
| **Job**        | Runs a task **once to completion**     | Until success/fail      | No (unless retries configured) |
| **CronJob**    | Runs a **Job on a schedule**           | Based on time           | Yes (per schedule)             |

```
Q. Is Job similar to cron Job?
A. A CronJob is like a Job with a schedule. Job runs once to completion. CronJob runs a Job on a schedule (e.g., every day at 2am).
So:
🔁 CronJob = Scheduled Job
🏁 Job = One-time task
```
---

## 🧾 Basic Job Example

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  template:
    spec:
      containers:
      - name: hello
        image: busybox
        command: ["echo", "Hello from Job"]
      restartPolicy: Never
```

---

## 🔁 Key Fields:

| Field           | Meaning                                                 |
| --------------- | ------------------------------------------------------- |
| `completions`   | Total successful runs needed (default: 1)               |
| `parallelism`   | How many pods can run in parallel                       |
| `backoffLimit`  | Max retries before job is marked as failed (default: 6) |
| `restartPolicy` | Should be `OnFailure` or `Never`                        |

---

## 🧠 Common Interview Questions

❓1. What’s the difference between Job and CronJob?
> Job runs **once**.
> CronJob runs **on a schedule** (like `cron` in Linux).

❓2. What happens if a Job Pod crashes?
> Kubernetes retries it, up to `backoffLimit`.
> If still failing → Job is marked as failed.

❓3. Can you run Jobs in parallel?
> ✅ Yes, use:

```yaml
parallelism: 3
completions: 6
```
This runs 3 at a time, until 6 finish successfully.

❓4. Will a Job clean up its pods?
> ❌ No. You must clean up Job pods or set `ttlSecondsAfterFinished`.

---

## ✅ Summary (Interview-style):

> A **Job** in Kubernetes runs one-off or batch tasks until they succeed.
> It supports retries, parallelism, and is ideal for tasks like DB migrations, backups, or report generation. For scheduled runs, use a **CronJob**.

==============================================================================

# PBD 

## 🧷 What is a **PDB**?
> A PodDisruptionBudget (PDB) is a Kubernetes resource that limits the number of Pods that can be voluntarily disrupted at a time.
It helps ensure high availability of applications during:
* Node draining (e.g. during maintenance)
* Rolling updates
* Voluntary evictions

>  Use Case: You want to ensure that some minimum number of pods stay running, even during cluster maintenance. Without a PDB, **all pods of a deployment might be evicted at once**, causing downtime.


## 🧾 Example:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: my-app
```

🎯 This means: At least 2 pods matching label "app: my-app" must be available at all times. Kubernetes will block evictions that violate this.

**⚠️ PDB only affects voluntary disruptions, not:**
- Node crashes
- OOM kills
- Force deletes

---

## ⚙️ Key Options

| Field            | Meaning                                     |
| ---------------- | ------------------------------------------- |
| `minAvailable`   | Minimum number of **available pods**        |
| `maxUnavailable` | Maximum number of pods that **can go down** |
| `selector`       | Match the pods this PDB applies to          |

---

## 🧠 Interview Questions

❓1. Will PDB stop all disruptions?
> ❌ No — it only affects **voluntary** disruptions (e.g., `kubectl drain`, node upgrades).
> It doesn't block **crashes**, `kill -9`, or node failures.


❓2. Can PDB block node drain?
> ✅ Yes, if the drain would violate the PDB, it **blocks the eviction**.

❓3. Is PDB useful for single-pod workloads?
> ❌ No. PDB is only meaningful when you have **2+ replicas**.

## ✅ Final Summary (interview-style):
> A **PDB ensures high availability** by controlling how many pods can be disrupted at a time. It’s critical during maintenance to avoid **accidental full outage** of services.

====================================================================

# k8s STEPS FOR CI-CD
---------------------------

### ✅ 1. **Clone code (from Git)**

* Source code along with `Dockerfile` is pulled in Jenkins.

### ✅ 2. **Build artifact**

* Run:

  ```bash
  mvn clean install
  ```
* Generates a `.jar` in `target/`, e.g. `target/app.jar`

### ✅ 3. **Build Docker image**

* Your `Dockerfile` copies the jar:

  ```Dockerfile
  COPY target/app.jar /app/app.jar
  ```
* Build:

  ```bash
  docker build -t your-app:<tag> .
  ```

### ✅ 4. **Login, tag, push Docker image**

* Authenticate to registry (Docker Hub, ECR, etc.)

  ```bash
  docker login
  docker tag your-app:<tag> your-registry/your-app:<tag>
  docker push your-registry/your-app:<tag>
  ```

### ✅ 5. **Connect to Kubernetes**

* Set `KUBECONFIG` so Jenkins shell can talk to the cluster:

  ```bash
  export KUBECONFIG=/path/to/kubeconfig
  ```

### ✅ 6. **Update image in deployment**

* Use:

  ```bash
  kubectl set image deployment/<deployment-name> <container-name>=your-registry/your-app:<tag>
  ```
>   eg kubectl set image deployment/nginx-deployment nginx=nginx:1.25.2

✅ This performs a **rolling update** in Kubernetes.

==========================================================================================

# KUBE CONFIG

Sure! Here's a complete explanation of **`KUBECONFIG`** in simple, technical, and **interview-friendly** language 👇

---

## 📁 What is `KUBECONFIG`?

> `KUBECONFIG` is an **environment variable** that tells the `kubectl` command **which cluster config file to use** to connect to Kubernetes.
> Kubeconfig is a configuration file that contains cluster access information (like server URL, credentials, and contexts) used by kubectl to interact with Kubernetes clusters.

By default, it uses:

```bash
~/.kube/config
```

But if you want to use a **different file or multiple configs**, you set `KUBECONFIG` manually.

---

## 🧠 Why it's important

It controls:

* ✅ Which **cluster** you talk to
* ✅ Which **user identity** is used (auth info)
* ✅ What **namespace** and **context** is active
* ✅ TLS settings, tokens, etc.

---

## 🔁 How to use it

### 🔹 1. Use a different config:

```bash
export KUBECONFIG=/path/to/custom/config.yaml
kubectl get pods
```

### 🔹 2. Use **multiple config files**:

```bash
export KUBECONFIG=~/.kube/config:/path/to/staging.yaml:/path/to/dev.yaml
kubectl config view --merge
```

---

## 🔍 What's inside a kubeconfig file?

Here’s a simplified version of `~/.kube/config`:

```yaml
apiVersion: v1
kind: Config

clusters:
- name: my-cluster
  cluster:
    server: https://api.k8s.example.com
    certificate-authority: ca.crt

users:
- name: admin
  user:
    token: <JWT-token>

contexts:
- name: admin@my-cluster
  context:
    cluster: my-cluster
    user: admin

current-context: admin@my-cluster
```

---

## ✅ Final Summary (Interview-Friendly):

> `KUBECONFIG` is an environment variable used by `kubectl` to locate one or more config files that store cluster connection details, credentials, and context.
> It's especially useful when you work with **multiple Kubernetes clusters** or CI/CD tools that use **custom kubeconfigs**.

---

# kube context

### ✅ What is a **Kube Context** in Kubernetes?

> A **kube context** is a named entry in your `kubeconfig` file that defines which **cluster**, **user**, and **namespace** you are interacting with when you run `kubectl` commands.

---

### 🧠 Why it's needed:

You may have access to **multiple clusters** (e.g., dev, staging, prod), and each has different:

* API server endpoint
* Credentials
* Default namespace

The **context** lets you switch between them **without editing commands every time**.

---

## 🔧 Structure of a Context

A context combines three things:

```yaml
contexts:
- name: dev-context
  context:
    cluster: dev-cluster
    user: dev-user
    namespace: dev
```

---

## 🛠️ How to Set Up a Context (from scratch)

1. **Add or define a cluster:**

```bash
kubectl config set-cluster dev-cluster \
  --server=https://dev.example.com \
  --certificate-authority=ca.crt
```

2. **Add user credentials:**

```bash
kubectl config set-credentials dev-user \
  --client-certificate=dev-user.crt \
  --client-key=dev-user.key
```

3. **Create a context that links them:**

```bash
kubectl config set-context dev-context \
  --cluster=dev-cluster \
  --user=dev-user \
  --namespace=dev
```

4. **Switch to that context:**

```bash
kubectl config use-context dev-context
```

---

## ✅ To View All Contexts:

```bash
kubectl config get-contexts
```

## ✅ To See the Current Context:

```bash
kubectl config current-context
```

---

## 📁 Where is it stored?

All this info is stored in:

```
~/.kube/config
```

You can inspect it:

```bash
kubectl config view
```

---

## 💡 Summary:

| Component     | Meaning                                |
| ------------- | -------------------------------------- |
| **Cluster**   | Kubernetes API server you connect to   |
| **User**      | Identity used for authentication       |
| **Namespace** | Default namespace for kubectl commands |
| **Context**   | Named shortcut for cluster + user + ns |

---

Let me know if you want help **merging multiple kubeconfigs**, or **setting up role-based access (RBAC)** tied to contexts.
