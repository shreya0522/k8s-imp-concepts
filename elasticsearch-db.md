
🔹 I. Core Concepts
| Concept                                 | Interview Focus                                              |
| --------------------------------------- | ------------------------------------------------------------ |
| ✅ **Index, Document, Shard, Replica**   | What are they? How are they related? How do you size shards? |
| ✅ **Cluster, Node, Node Roles**         | Difference between master, data, ingest, coordinating nodes  |
| ✅ **Shard Routing & Allocation**        | How docs are assigned to shards, how shards move             |
| ✅ **Cluster Health (green/yellow/red)** | What causes each state? How to resolve?                      |


🔹 II. Operational Concepts
| Concept                                      | Interview Focus                                            |
| -------------------------------------------- | ---------------------------------------------------------- |
| ✅ **Heap Sizing & JVM Tuning**               | Why 50% of RAM? Why limit to 32 GB heap?                   |
| ✅ **Disk Watermarks**                        | How does Elasticsearch respond when disk is full?          |
| ✅ **Index Lifecycle Management (ILM)**       | What is it? How do you implement log retention?            |
| ✅ **Rolling Restart / Upgrade**              | Step-by-step safe upgrade, what if node doesn’t come back? |
| ✅ **Snapshots and Restore**                  | How to back up and restore indices? To S3/shared FS?       |
| ✅ **Cluster Settings (Static vs Dynamic)**   | Which settings need restart vs can be updated live?        |
| ✅ **Cluster Coordination / Master Election** | Quorum logic, how split-brain is avoided                   |


🔹 III. Scaling and Performance
| Concept                         | Interview Focus                                |
| ------------------------------- | ---------------------------------------------- |
| ✅ **Hot-Warm Architecture**     | What is it? Why use it for time-based data?    |
| ✅ **Query Performance Tuning**  | How to detect and fix slow queries?            |
| ✅ **Shard Rebalancing**         | How shards move when new nodes added or failed |
| ✅ **Index Templates & Aliases** | How to route writes to rolling indices         |
| ✅ **Fielddata vs doc\_values**  | Why fielddata is risky on text fields          |
| ✅ **Force Merge**               | What is it? When to use? Risks?                |


🔹 IV. Security and Access Control
| Concept                               | Interview Focus                                |
| ------------------------------------- | ---------------------------------------------- |
| ✅ **Authentication / RBAC**           | Basic realm, native users, API keys            |
| ✅ **TLS Setup (Node-to-Node & HTTP)** | How to configure, what files needed?           |
| ✅ **Restricting API Access**          | IP whitelisting, firewall, nginx reverse proxy |
| ✅ **X-Pack Security**                 | Features unlocked, basic vs platinum licenses  |


🔹 V. Monitoring and Troubleshooting
| Concept                                     | Interview Focus                                     |
| ------------------------------------------- | --------------------------------------------------- |
| ✅ **\_cat APIs**                            | Use of `_cat/shards`, `_cat/indices`, `_cat/health` |
| ✅ **\_cluster/allocation/explain**          | Why is shard unassigned?                            |
| ✅ **Circuit Breakers**                      | What are they? When do they trigger?                |
| ✅ **GC Pressure**                           | What causes heap GC thrashing? How to diagnose?     |
| ✅ **Slow Logs**                             | Where to find? How to configure query logging?      |
| ✅ **Pending Tasks / Cluster State Updates** | What are they? How to clear backlog?                |


🔹 VI. Architecture & Design
| Concept                               | Interview Focus                                                  |
| ------------------------------------- | ---------------------------------------------------------------- |
| ✅ **Elasticsearch in Docker/K8s**     | How to handle persistent storage and networking?                 |
| ✅ **Dedicated Master Nodes**          | Why? How many?                                                   |
| ✅ **Coordinating Nodes**              | When and why to add them?                                        |
| ✅ **Rack Awareness / Zone Awareness** | How to avoid placing primaries/replicas on same rack             |
| ✅ **Custom Routing**                  | What is `_routing` used for?                                     |
| ✅ **Data Tiering in ES 7.10+**        | `hot`, `warm`, `cold`, `frozen`, `content`, `coordinating` roles |


🧠 Sample Interview Questions from These Topics
Q: What is the ideal heap size in Elasticsearch?
A: 50% of system RAM, capped at 30–31 GB to preserve compressed oops (compressed object pointers). Going above disables it and doubles object size.

Q: What causes a red cluster? How do you troubleshoot it?
A: Red = at least one primary shard is missing/unassigned
Causes: disk full, corrupted index, node failure, bad shard allocation filters
Troubleshooting:
 - GET _cat/health
 - GET _cat/shards
 - GET _cluster/allocation/explain
Restore from snapshot if required

Q: What is ILM and how does it help?
A: ILM = Index Lifecycle Management.
Used to automate:
  - Index rollover (e.g. daily logs)
  - Shrinking
  - Forcemerge
  - Delete after retention (e.g. 30 days)
  - Reduces manual index management, improves performance, avoids bloating.

Q: How do you handle a node failure in a 3-node cluster?
A:If quorum (2/3) is preserved → cluster is healthy or yellow
Missing node’s shards will reallocate (if replicas exist)
Monitor GET _cat/shards and logs for unassigned shards
Fix disk/network/heap → restart node → it rejoins
✅ Tip: What Makes a Good DevOps Elasticsearch Answer?
Mention exact API or CLI command
Show awareness of cluster behavior (e.g., shard movement, GC, master election)
Always think in terms of HA, scalability, automation, security






============================================================================

1- What is Elasticsearch?
A- Elasticsearch is:
    - A search engine that helps you quickly find information in big amounts of data.
    - It is built on Apache Lucene (a powerful library for search).
    - It’s distributed, meaning it can work across many servers together.
    - It provides a REST API, so you can interact with it using HTTP (like a web browser or Postman).

--------------------------------------------------------------------------------------------------------------------------------------------------------------

2- What problem does it solve?
A- Imagine you have logs, documents, metrics, or JSON data and want to:
     - 🔎 Search quickly (e.g., find errors in logs)
     - 📊 Analyze the data (e.g., how many errors per hour?)
     - 💡 Get results in near real-time
Traditional databases are not built for this kind of fast, flexible searching. Elasticsearch solves that.

--------------------------------------------------------------------------------------------------------------------------------------------------------------

3- 👩‍💻 Why does a DevOps/Ops person care?
A- 
   -  Used in ELK Stack (Elasticsearch + Logstash + Kibana) for log analysis
   -  Used in SIEM tools (security event monitoring)
   -  Need to understand:
         * Cluster setup (distributed system)
         * HA (High Availability)
         * Scaling (add/remove nodes)
         * Backups
         * Upgrades

--------------------------------------------------------------------------------------------------------------------------------------------------------------

ATOM CONCEPT : 
🔠 A — Atomicity
All or nothing: A transaction (like transferring ₹500 from A to B) must complete entirely or not at all. If anything fails, everything is rolled back. Think of it like a text message — it either sends completely or doesn't send at all, never halfway.

🔠 C — Consistency
The database must always remain in a valid state. Rules (like “you can’t have negative money in an account”) are always followed. Example: If A has ₹1000, and you transfer ₹1200 — the database rejects it because it would break a rule.

🔠 I — Isolation
Transactions run independently from each other.One user’s transaction won’t interfere with another’s. Imagine two people editing a Google Doc — isolation makes sure their changes don’t overwrite each other in a bad way.

🔠 D — Durability
Once a transaction is committed, it’s saved permanently — even if the server crashes. Like clicking “Save” on a file — even after your laptop crashes, the file is still there.

Is elasticserach atomic 
Atomicity means: A single operation (like inserting one document) either completes fully or not at all. No in-between state. This ensures the data is either written or rolled back cleanly.
✅ Does Elasticsearch support Atomicity?
Yes — but only in a limited way:
  - ✅ Atomic per single document: 
 * If you index (insert/update) one document, Elasticsearch ensures it's written fully or not at all on the primary shard.
 * You will never get a half-written document.

  - ❌ Not atomic for multiple documents
 * If you want to write/update/delete multiple documents as a group, there’s no rollback if something fails partway.
 * Elasticsearch does not support transactions like a relational DB (e.g., BEGIN, COMMIT, ROLLBACK).
Example: If you're trying to update 3 documents and 1 fails, the other 2 stay changed — no automatic rollback.

⚙️ How does it achieve this?
When a document is indexed:
   - It's written to the primary shard (main copy).
   - Once confirmed, it is asynchronously replicated to replica shards.
If something fails before writing to the primary shard completes, the operation fails completely — nothing is written.

💡 So, in summary:
| Feature                     | Supported in Elasticsearch? |
| --------------------------- | --------------------------- |
| Atomic write for 1 document | ✅ Yes                       |
| Atomic write for many docs  | ❌ No                        |
| Rollback on failure         | ❌ No                        |
| Asynchronous replication    | ✅ Yes                       |

--------------------------------------------------------------------------------------------------------------------------------------------------------------

🧱 1. Document (Smallest unit)
This is the actual data — a single JSON object. Example It's similar to one row in a traditional database.
```
{
  "user": "shreya",
  "action": "login",
  "timestamp": "2025-06-16T10:00:00"
}
```

--------------------------------------------------------------------------------------------------------------------------------------------------------------

📚 2. Index
A collection of many documents that are related.  You might have indexes like:
One Index = Many Documents
An index in Elasticsearch is like a database table — it's a logical namespace where documents (JSON objects) are stored and organized.

✅ Key Properties:
- Each index has a unique name (logs-2025-06-17)
- Holds many documents (JSON)
- Each index is physically divided into shards
You can create, delete, search, and manage indices independently

🧠 Real-World Examples:
- logs-2025-06-17 → today's logs
- products → all product catalog data
- users → all user profiles
🔹 What is an Index in Elasticsearch?
An index in Elasticsearch is like a database table — it's a logical namespace where documents (JSON objects) are stored and organized.

--------------------------------------------------------------------------------------------------------------------------------------------------------------

🧩 3. Shard
An index is split into shards for performance and scalability.  Each shard is like a mini-index that can be stored on different machines (nodes). 
Two types:
    🟢 Primary Shard: Original data         🟡 Replica Shard: Copy for backup and load balancing
1 Index = N Primary Shards + N Replica Shards

if you do : curl -X GET "http://localhost:9200/_cat/indices?v" so you below that index ttsn_request has primary shards and replica shards as 1 & requested_trips has pri shards as 10  & rep shards as 1 thta means an index is divided into shards 
health status index                               uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   ttsn_requests                       yZqX_LIVSqa-Mr77l3G6DQ   1   1     913216        18647    196.9mb         98.4mb
green  open   visa-inventories                    BDBz9Kc-ROOOAgEAqnk0Rg   1   1         22            4     73.3kb         37.8kb
green  open   requested_trips                     ZFkTNj1kQ62dYM7NaDeRLg  10   1  155004318      9420402    316.5gb        152.4gb

🧠 Follow-up Questions (with Answers)
Q: Can I change number of primary shards after index is created?
A: ❌ No, but you can reindex into a new index with different shard settings.

Q: Can I change number of replica shards dynamically?
A: ✅ Yes.
```
PUT /my-index/_settings
{
  "number_of_replicas": 2
}
```

Q: What is a good shard size?
A: ~10GB–50GB per shard is a good baseline. Avoid “many small shards” (called shard explosion).

--------------------------------------------------------------------------------------------------------------------------------------------------------------

4. Node
A server (physical or virtual) that stores shards. It does the work: storing data, handling search, etc.   Node = Can hold many Shards (from multiple indexes)

--------------------------------------------------------------------------------------------------------------------------------------------------------------

5. What is Sharding in Elasticsearch?
Ans: Sharding is the process of splitting an index into multiple smaller parts (shards) to enable horizontal scaling, fault tolerance, and parallel processing.
✅ In One Line: Sharding = dividing a large index into smaller pieces (called shards) so that Elasticsearch can store, distribute, and query data efficiently across multiple nodes.

🔁 Sharding Workflow in Elasticsearch (Detailed)
📌 Step 1: Index Creation (Sharding Plan)
When you create an index, you specify: number_of_shards: total primary shards (default: 1)  ||| number_of_replicas: number of replicas per primary (default: 1)
Example
```
PUT /logs-2025-06-17
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1
  }
}
```
➡️ This tells Elasticsearch: “I want 3 primary shards, and each should have 1 replica.”  → So total 6 shards: 3 primaries + 3 replicas

📌 Step 2: Cluster Master Allocates Shards
The master node checks available nodes. It assigns each primary shard to a node (based on disk, shard balancing, node availability). Then it assigns replica shards to different nodes.
Simplified Layout
| Node | Shards Placed        |
| ---- | -------------------- |
| A    | Primary 0, Replica 1 |
| B    | Primary 1, Replica 2 |
| C    | Primary 2, Replica 0 |
✅ Replica shards never live on the same node as their primary.

📌 Step 3: Document Indexing (Shard Routing)
Every document has a unique _id. Elasticsearch uses a routing algorithm:
```
shard = hash(_id) % number_of_primary_shards
```
This determines which primary shard will hold the document. ✅ Only primary shards are written to → then synced to replica asynchronously.

📌 Step 4: Replica Sync & Redundancy
Replica shards replicate the data of their corresponding primary.  If the primary shard/node fails: Replica is promoted to primary, A new replica is created elsewhere. This provides high availability and fault tolerance

📌 Step 5: Query Execution
When a search request is made: Coordinating node (any node receiving the request) broadcasts it to:
       - All primary or replica shard
       - Each shard executes query in parallel
       - Results are aggregated and returned
✅ Elasticsearch searches multiple shards at once → massive parallelism

📊 Visual Workflow Diagram (Conceptual)
```
[Client]
   |
   |  Search: logs-2025-06-17
   v
[Coordinating Node]
   |----> Primary Shard 0 (Node A)
   |----> Replica Shard 1 (Node A)
   |----> Primary Shard 1 (Node B)
   |----> Replica Shard 2 (Node B)
   |----> Primary Shard 2 (Node C)
   |----> Replica Shard 0 (Node C)
   |
   v
[Merge results] → Return response
```

🎯 Why Sharding Workflow Matters in DevOps
| Problem              | Sharding Benefit                      |
| -------------------- | ------------------------------------- |
| Huge datasets (TBs+) | Split across many nodes               |
| Query latency        | Search runs in parallel across shards |
| Node failure         | Replica becomes new primary           |
| Scaling cluster      | New nodes take new shards             |


📋 DevOps Interview Questions & Follow-Ups
-------------------------------------------

❓Q1: What happens internally when an index is created?
Answer: * Master assigns primary shards to nodes
        * Waits until all primaries are active
        * Then assigns replicas
        * Index becomes green when all are active

❓Q2: How does Elasticsearch decide which shard a document goes to?
Answer: Via a routing hash on _id:
```
shard = hash(_id) % number_of_primary_shards
```
Follow-up Q: Can you change the routing mechanism?
A: Yes, using a custom routing key (_routing). Useful when grouping related documents.

❓Q3: How are shards allocated to new nodes when a node is added?
Answer: 
* Master triggers rebalancing
* Uses disk usage, number of shards, and allocation filters
* Shards may be migrated to new node in background

Command to view: ```GET _cat/shards?v```

❓Q4: Can the number of shards be changed after index creation?
Answer: - ❌ Primary shards cannot be changed ,  ✅ You can reindex into a new index with a different shard count
Workflow:
- Create new index    - Use _reindex API   - Delete old index

❓Q5: What happens if a primary shard goes down?
Answer: A replica shard is promoted to primary. Cluster health may go to yellow if a new replica cannot be allocated

❓Q6: How does Elasticsearch prevent replica-primary placement on same node?
Answer: It uses a shard allocation awareness mechanism:
- Ensures replica and primary are never co-located
- Uses node.attr.* and rack awareness for advanced control

--------------------------------------------------------------------------------------------------------------------------------------------------------------

6. TYpes of nodes

----------------------------------------------------------------------------------
🔹 1. Master-Eligible Node
-------------------------------
📌 Purpose: 
- Participates in master election
- Manages cluster state,
- handles index creation/deleteion ,
- manage shard allocation,
- tracks node joins/leaves
- Applying template or setting changes

🔁 Master Election Workflow
Step-by-Step: How Master Election Happens
a- Cluster starts or master fails
b- All nodes with node.roles: [master] participate in an election
c-The node that wins becomes the cluster master
d-Other master-eligible nodes act as followers (standby)

🧮 Quorum Logic (Zen Discovery): Cluster only becomes operational if a quorum of master-eligible nodes is present. ``` Quorum = (N / 2) + 1 ```
Example: * 3 master-eligible nodes → quorum = 2    * So cluster can tolerate 1 failure

🛠️ Configuration
------------------
a- In elasticsearch.yml: ```node.roles: [master]```
This makes the node master-eligible. It may or may not be elected master, depending on cluster state and quorum.
b- Minimum Master Nodes (pre 7.x): ```discovery.zen.minimum_master_nodes: 2  # deprecated in 7.x+```
c- From 7.0 onward, use: ```cluster.initial_master_nodes: ["node1", "node2", "node3"]```

🧯 What Happens If the Master Fails?
a- Remaining master-eligible nodes hold a new election
b- If quorum is preserved → new master elected, cluster continues
c- If quorum is lost → no master can be elected → cluster becomes unavailable
   * Indexing and cluster changes stop
   * Read-only queries may still work temporaril

✅ Best Practices for Master-Eligible Nodes
| Rule                                        | Reason                                          |
| ------------------------------------------- | ----------------------------------------------- |
| Use **odd number** (≥ 3)                    | Ensures quorum and avoids split-brain           |
| Separate from data nodes                    | Prevents query load from delaying cluster tasks |
| Use stable VMs/instances                    | Master nodes must have high uptime              |
| Don’t use coordinating-only nodes as master | They should only route requests                 |

🚫 Anti-Pattern:
Having only 1 master-eligible node → any restart causes cluster to go red or unresponsive.

📋 DevOps Interview Q&A
❓ Q: What happens if all master-eligible nodes go down?
A: Cluster becomes unavailable. Even if data nodes are alive, no cluster-level operations can proceed — you cannot create indices, reroute shards, or apply settings.

❓ Q: Why should master nodes not be data or ingest nodes?
A: To prevent GC pauses, CPU spikes, or overload from impacting the master’s ability to maintain cluster state and coordinate tasks. This improves cluster stability.

❓ Q: What if I have only one master-eligible node?
A: This is not recommended. If that node goes down, the cluster cannot elect a master, and becomes non-functional. Even in small setups, use 3 master-eligible nodes.

❓ Q: Can I make every node master-eligible?
A: Technically yes, but it’s best to dedicate master nodes and isolate them from load. Having many unnecessary master-eligible nodes increases election chatter and instability.

❓ Q: What’s split-brain and how do master nodes prevent it?
A: Split-brain = multiple nodes believe they are master. Elasticsearch avoids this by requiring quorum during master elections. If quorum isn't met, no master is elected.

----------------------------------------------------------------------------------

🔷 2. Data Node in Elasticsearch
✅ What is a Data Node?
A data node is responsible for:
- Storing data (shards)   - Executing queries and aggregations   - Indexing documents (write operations)
- Replicating shards      - Participating in search request fan-out

🛠️ Responsibilities
| Task                              | Description                               |
| --------------------------------- | ----------------------------------------- |
| 📦 Store primary & replica shards | Physically holds index data               |
| 🧮 Perform indexing               | Writes go to primary shards on data nodes |
| 🔍 Run search queries             | Shard-level search and aggregation        |
| 🔁 Shard replication & recovery   | Handles intra-cluster data sync           |

⚙️ Configuration in elasticsearch.yml ```node.roles: [data]```  Or to combine roles: ```node.roles: [data, ingest]```

📊 Resource Requirements (DevOps View)
| Resource            | Recommendation                                |
| ------------------- | --------------------------------------------- |
| **Disk**            | SSDs (IO-intensive)                           |
| **Heap**            | 50% of RAM, capped at 30–31 GB                |
| **CPU**             | High, for heavy aggregation/search            |
| **Disk Watermarks** | Watch `low`, `high`, `flood_stage` thresholds |

⚠️ DevOps Gotchas
| Mistake                        | Consequence                        |
| ------------------------------ | ---------------------------------- |
| Too many small shards          | Wastes heap/memory                 |
| Data node holds all roles      | Can cause GC or IO bottlenecks     |
| Ignoring disk watermark errors | Indexing blocks, shards unassigned |

✅ Best Practices for Data Nodes
- Separate from master nodes
- Use dedicated data nodes for large datasets
- Monitor disk usage and heap pressure
- Use index lifecycle management (ILM) to roll over old data

❓ DevOps Interview Questions on Data Node

Q: What happens when a data node goes down?
A: Its shards become unavailable. If replicas exist, queries continue. If primaries are lost and no replicas → index turns red. Master node tries to reallocate shards.

Q: Can a data node also be master or ingest?
A: Technically yes. But in production, roles should be isolated:
  - Data nodes → storage/query
  - Master nodes → cluster control
  - Ingest nodes → preprocessing

Q: How do you monitor data node health?
A: - GET _cat/nodes?v
   - GET _nodes/stats
   - GET _cluster/allocation/explain

----------------------------------------------------------------------------------


🔷 3. Ingest Node in Elasticsearch
-------------------------------------
✅ What is an Ingest Node?
An ingest node is used to preprocess documents before indexing, using ingest pipelines.  Think of it as a mini Logstash inside Elasticsearch.
An Ingest Node in Elasticsearch is like a “data pre-processor”. Before data is stored (indexed), it often needs to be:
   - Parsed (e.g., split a log line into fields)
   - Enriched (e.g., add GeoIP location from IP)
   - Cleaned (e.g., remove unnecessary fields)
Instead of using external tools like Logstash, you can do this inside Elasticsearch using Ingest Pipelines that run on ingest nodes.

🎯 Why Would You Need Ingest Nodes?
| Use Case                             | What You Need                           |
| ------------------------------------ | --------------------------------------- |
| Indexing raw Apache logs             | Use Grok to parse `message` into fields |
| Enriching data from IP address       | Use `geoip` processor to add `location` |
| Normalizing fields                   | Rename, set, or remove fields           |
| Light preprocessing without Logstash | Ingest node pipelines directly          |

🔁 Real-World Workflow: “Log Parsing and Enrichment”
🔧 Setup:
3-node Elasticsearch cluster:
 - node-1: master
 - node-2: ingest
 - node-3: data
Your app sends raw Apache logs to Elasticsearch
```
{
  "message": "192.168.1.1 - - [17/Jun/2025] \"GET /index.html HTTP/1.1\" 200 2326"
}
```

🚦 Full Workflow (Step-by-Step)
📥 Step 1: Request Comes In
A client sends a document to Elasticsearch via REST API with a pipeline
```POST /weblogs/_doc?pipeline=log_pipeline```
This hits a Coordinating Node (can be any node that received the request).

🔁 Step 2: Coordinating Node Forwards to Ingest Node
The coordinating node sees that pipeline=log_pipeline is mentioned. It forwards the document to a node with role ingest.That node runs the pipeline processors.

🔧 Step 3: Ingest Node Runs Pipeline
The pipeline might look like:
```
PUT _ingest/pipeline/log_pipeline
{
  "processors": [
    {
      "grok": {
        "field": "message",
        "patterns": ["%{COMMONAPACHELOG}"]
      }
    },
    {
      "geoip": {
        "field": "clientip"
      }
    },
    {
      "remove": {
        "field": "message"
      }
    }
  ]
}
```
🔹 What it does:
- grok splits the message into fields like clientip, request, status, bytes
- geoip adds location info from IP
- remove deletes the raw message

✔️ Output becomes a clean structured doc:
```
{
  "clientip": "192.168.1.1",
  "request": "/index.html",
  "status": 200,
  "location": {
    "lat": 28.61,
    "lon": 77.23
  }
}
```

🧱 Step 4: Forward to Data Node for Indexing
After transformation, the ingest node forwards the cleaned document to the data node. The primary shard for the target index (e.g., weblogs) resides here. The document is stored

✅ Step 5: Master Node Maintains State
* Ensures pipeline exists
* Tracks node roles
* Allocates shards
* Handles errors if nodes go down

🧠 Visual Workflow (Roles Involved)
```
[ Client ] 
    |
    v
[ Coordinating Node ]
    |
    |  → pipeline=log_pipeline
    v
[ Ingest Node ]
    - grok parser
    - geoip enrichment
    - remove field
    |
    v
[ Data Node ]
    - Receives transformed doc
    - Stores in correct shard
    |
    v
[ Master Node ]
    - Watches over all this
```

🛡️ Why Use Ingest Node Over Logstash?
| Criteria                    | Ingest Node | Logstash                     |
| --------------------------- | ----------- | ---------------------------- |
| Built-in to Elasticsearch   | ✅ Yes       | ❌ No (external service)      |
| Lightweight parsing         | ✅           | ✅                            |
| Complex flows, conditionals | ❌ Limited   | ✅ Better                     |
| Runs inside cluster         | ✅           | ❌ Separate infra             |
| Auto-scales with cluster    | ✅           | ❌ Must be managed separately |

FYI
🧰 Common Processors in Pipelines
| Processor       | Purpose                     |
| --------------- | --------------------------- |
| `grok`          | Parse text via regex        |
| `set`, `rename` | Modify field values or keys |
| `geoip`         | Add location info from IP   |
| `remove`        | Drop unnecessary fields     |
| `date`          | Parse timestamps            |

⚙️ Configuration in elasticsearch.yml ```node.roles: [ingest]``` You can combine with other roles:
```node.roles: [data, ingest]```

📋 DevOps Interview Questions (with Answers)
---------------------------------------------
❓ Q: What is the role of an ingest node?
A: An ingest node runs ingest pipelines on incoming documents before indexing. It transforms or enriches data without external tools.

❓ Q: What happens if no ingest node is present but a pipeline is used?
A: The request will fail with a 400 error. Elasticsearch cannot execute the pipeline without a node that has ingest role.

❓ Q: Can a data node also be an ingest node?
A: Yes. In small clusters, it's common to combine data and ingest roles. But in production, for high ingestion throughput, it’s better to separate them.

❓ Q: Where are ingest pipelines defined and stored?
A: In the cluster state, managed by the master node. You define them via:
```
PUT _ingest/pipeline/<pipeline_name>
```

----------------------------------------------------------------------------------

✅ Remaining Node Types in Elasticsearch (Beyond Master, Data, Ingest)

4️⃣ Coordinating-Only Node 
 - Acts as a query router / load balancer.
 - Forwards search/indexing requests
 - Does not store data, does not ingest, not master
 - Used in front of Kibana / API clients for performance
 - ✅ Used in large clusters to offload query load

5️⃣ Machine Learning (ML) Node
- Runs anomaly detection and ML jobs.
- Used in Elastic Stack’s X-Pack features (commercial tier).
- Requires:   * More memory   * GPU (optional)    * Defined ML jobs

6️⃣ Transform Node
- Executes transform jobs (like pivoting, grouping, summarizing data streams).
- Used for rollup or aggregation pipelines.
- ✅ Useful in analytics-heavy environments.

7️⃣ Voting-Only Node
- Participates in master elections but can’t be elected.
- Used to:  * Maintain quorum   * Avoid split-brain   * Add a tie-breaker without giving full master responsibilities

8️⃣ Remote Cluster Client Node
- Used to query remote clusters via cross-cluster search (CCS) or cross-cluster replication (CCR).
- Does not store data or run queries on its own.

9️⃣ Frozen Node
- Stores frozen indices (archived, read-only, low-speed storage).
- Optimized for:  * Cold data    * Low-cost disk   * Snapshot-based restores

🔟 Data Tier Nodes (Hot/Warm/Cold/Content)
Introduced in Elasticsearch 7.10+, instead of just data, you now have more granular roles, Used with Index Lifecycle Management (ILM) to move data across tiers.

🎯 Goal: To **optimize cost and performance** for **time-based, log-heavy data** by **assigning data to storage tiers**, managed via **Index Lifecycle Management (ILM)**.

✅ What Are Data Tiers?
Instead of a generic `data` role, Elasticsearch now supports **tier-specific roles** that help:
* Separate **recent frequently queried data** (e.g., today's logs)
* From **older, infrequently accessed data** (e.g., last quarter’s logs)
* While still **keeping all data searchable**
These roles help Elasticsearch **automatically allocate shards** to nodes designed for specific purposes using **ILM policies**.

 📊 Data Tier Node Roles Overview

| Tier        | `node.roles`   | Best For                                      | Storage Type        | Example Use                        |
| ----------- | -------------- | --------------------------------------------- | ------------------- | ---------------------------------- |
| **Hot**     | `data_hot`     | New data, indexing + frequent queries         | Fast SSD            | Today’s logs, real-time dashboards |
| **Warm**    | `data_warm`    | Old but searchable logs, low updates          | HDD                 | Last week’s logs                   |
| **Cold**    | `data_cold`    | Rarely queried logs (archive but searchable)  | Slower HDD          | Last month’s logs                  |
| **Frozen**  | `data_frozen`  | Archived data, loaded on-demand from snapshot | S3 / object storage | Compliance data, historic audits   |
| **Content** | `data_content` | General purpose storage (non-log)             | SSD/HDD             | User data, CRM/Quotes/Bookings     |

 📦 Tier Roles in `elasticsearch.yml`

```yaml
node.roles: [data_hot]
node.roles: [data_warm]
node.roles: [data_cold]
node.roles: [data_frozen]
node.roles: [data_content]
```

⚠️ These roles are **mutually exclusive** — a node should have only one data tier role to avoid confusion.

 🔁 Data Flow (via ILM)
Example: Logs ILM Policy

| Phase  | Age      | Action                                | Moves To      |
| ------ | -------- | ------------------------------------- | ------------- |
| Hot    | 0d–7d    | Indexing, fast queries                | `data_hot`    |
| Warm   | 7d–30d   | Forcemerge, read-only                 | `data_warm`   |
| Cold   | 30d–180d | Read-only, low priority               | `data_cold`   |
| Frozen | >180d    | Unload from disk, search via snapshot | `data_frozen` |
| Delete | >1 year  | Delete index                          | (removed)     |
🧠 ILM handles automatic shard migration based on index age and tier availability.

🧠 Real-World Example: How Tiers Help
----------------------------------------
Company: `FlightFinder.com` **Scenario:**
* You receive 2 TB logs/day from search, booking, payment services
* You need:
  * Fast dashboards for last 7 days
  * Keep 6 months for audits
  * Reduce storage cost

Solution Using Tiers:
| Node Type   | Count | Role          | Storage              | Example             |
| ----------- | ----- | ------------- | -------------------- | ------------------- |
| Hot Nodes   | 3     | `data_hot`    | SSD                  | Recent logs         |
| Warm Nodes  | 2     | `data_warm`   | HDD                  | Logs 1–4 weeks old  |
| Cold Nodes  | 1     | `data_cold`   | Cheaper HDD          | 1–6 month-old logs  |
| Frozen Node | 1     | `data_frozen` | None (pulls from S3) | Older than 6 months |

✅ This setup:
* Saves money (SSD only for hot nodes)
* Keeps everything searchable
* Automatically migrates data with ILM

🔄 ILM + Tiers = Automated Storage Lifecycle
Example ILM Policy:

```json
PUT _ilm/policy/logs-tiered
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_age": "7d",
            "max_size": "50gb"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "allocate": { "include": { "data": "warm" } },
          "forcemerge": { "max_num_segments": 1 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "allocate": { "include": { "data": "cold" } },
          "set_priority": { "priority": 0 }
        }
      },
      "delete": {
        "min_age": "180d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

---

✅ Benefits of Tiered Architecture

| Benefit                   | Why It Matters                       |
| ------------------------- | ------------------------------------ |
| 💰 Cost efficiency        | SSDs only used where needed          |
| 🔄 Automated retention    | ILM moves/deletes data               |
| ⚡ Query speed optimized   | Fast access for recent data          |
| 🧠 Operational simplicity | No manual reindexing/migration       |
| 🧹 Disk health preserved  | Avoids overloading single node class |

❓ Common Interview Questions

Q: Why use warm/cold tiers if everything can be stored on hot nodes?
**A:** To reduce cost and improve performance. Hot nodes use expensive SSDs and hold indexing workload. Warm/cold tiers store read-only, infrequent data on cheaper disks.

Q: Can a frozen node store actual data?
A:** No. Frozen nodes **load snapshot shards on demand** from backup repositories (like S3). They don't hold full data permanently on disk.

Q: What if ILM moves data to a tier but that node type doesn't exist?
**A:** Shard allocation will **fail**, and index status may go `yellow`. You must ensure each ILM target tier exists in the cluster.

🧠 Final Summary Table

| Tier    | Role           | Purpose                       | Access Speed | Cost         |
| ------- | -------------- | ----------------------------- | ------------ | ------------ |
| Hot     | `data_hot`     | Fast ingest/search            | 🔥 Fastest   | 💰 Expensive |
| Warm    | `data_warm`    | Read-only recent data         | ⚡ Fast       | ✅ Cheaper    |
| Cold    | `data_cold`    | Archived, rarely used         | 🐢 Slower    | ✅✅           |
| Frozen  | `data_frozen`  | Load-on-demand from snapshots | ❄️ Slowest   | 💸 Cheapest  |
| Content | `data_content` | General purpose (non-log)     | ⚡ Depends    | 💰 Depends   |

----------------------------------------------------------------------------------

 Warm vs Content Tier
 | Feature                  | **Warm Tier** (`data_warm`)          | **Content Tier** (`data_content`)                          |
| ------------------------ | ------------------------------------ | ---------------------------------------------------------- |
| **Designed for**         | Archived **time-series/log data**    | **General-purpose search** (e.g. user data, catalog, CRM)  |
| **Index status**         | Usually **read-only**                | Can be **read-write**                                      |
| **ILM integration**      | ✅ Tight integration (auto migration) | ❌ Not part of ILM tier-based migration                     |
| **Shard placement**      | ILM moves old indices here           | Shards manually assigned or defaulted                      |
| **Use case**             | App logs, access logs, metrics       | Searchable content, e.g. product descriptions, agent notes |
| **Query frequency**      | Low (archived but still queried)     | Medium-to-high (ongoing operational use)                   |
| **Storage**              | Slower HDD (cost-optimized)          | Any (HDD or SSD based on performance need)                 |
| **Auto-rolled via ILM?** | ✅ Yes                                | ❌ No                                                       |

🔥 WARM TIER (data_warm) — In Detail
✅ Best For:
- Time-series logs that are not being written to anymore
- Archived logs still needed for search (last week’s logs, 30-day retention)
⚙️ Controlled By: ILM policies
- Usually added after the hot phase (first 7–14 days)
🧠 Behavior:
- Elasticsearch automatically marks indices read-only
- Forcemerge often applied to reduce storage
- Queries are slower, indexing is disabled
```
PUT _ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "warm": {
        "min_age": "7d",
        "actions": {
          "allocate": {
            "include": { "data": "warm" }
          },
          "forcemerge": {
            "max_num_segments": 1
          }
        }
      }
    }
  }
}
```

📦 CONTENT TIER (data_content) — In Detail
✅ Best For: User-facing or operational search data like:
   - Quotes
   - CRM records
   - Product catalogs
   - Knowledge base articles
   - Booking records
Anything that’s not logs, but still needs full-text search

⚙️ Not Used With: ILM phases like hot → warm → cold → frozen

🧠 Behavior:
- General-purpose data nodes
- Indexes can stay read-write
- Stored on performance-optimized or balanced hardware

🧠 Key Conceptual Difference
| Property        | `data_warm`              | `data_content`                             |
| --------------- | ------------------------ | ------------------------------------------ |
| Think of as     | “archive shelf for logs” | “active bookshelf of structured documents” |
| Controls        | ILM                      | Manual or static                           |
| Writeable?      | ❌ Read-only              | ✅ Usually read-write                       |
| Time-sensitive? | ✅ Time-based             | ❌ Not time-sensitive                       |

🏁 Real-World Analogy (Travel Company)
| Data                                    | Tier                      |
| --------------------------------------- | ------------------------- |
| Booking logs from last week             | `data_warm`               |
| Archived server logs older than 30 days | `data_cold`               |
| Agent quotes for customers              | `data_content`            |
| Customer service chat history           | `data_content`            |
| Failed payment events from yesterday    | `data_hot` or `data_warm` |
| Product catalog (hotels, packages)      | `data_content`            |

✅ Final Summary
| Feature          | **Warm Tier** (`data_warm`) | **Content Tier** (`data_content`)   |
| ---------------- | --------------------------- | ----------------------------------- |
| Purpose          | Time-based archival (logs)  | Operational data for general search |
| Write access     | ❌ Read-only                 | ✅ Often read-write                  |
| ILM support      | ✅ Fully supported           | ❌ Not part of ILM tier migration    |
| Typical usage    | Metrics, logs, time-series  | Quotes, profiles, product info      |
| Search frequency | Low                         | Medium to High                      |
| Data type        | Append-only logs            | Entity-based documents              |
| Backed by        | HDD                         | HDD or SSD (based on need)          |




