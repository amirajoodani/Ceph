# Waht is Ceph ?
<b>The High-Level Summary </b><br>
At its core, Ceph is an open-source, software-defined storage platform designed to provide highly scalable, reliable, and performant storage for modern applications and data centers. Instead of relying on expensive, proprietary hardware, Ceph turns a cluster of standard servers (with their hard drives and SSDs) into a unified, intelligent, and resilient storage system. <br>
<b>The Core Idea: The Ceph Storage Cluster</b><br>
Imagine you have a room with 100 hard drives. Instead of managing them individually, Ceph logically pools them all together. When you store a file, Ceph doesn't just put it on one disk. It automatically:

- Chops the file into smaller pieces (objects).
- Replicates or erasure codes those pieces for data protection.
- Distributes the pieces across many different disks and servers in the cluster.
- This architecture is the key to all of Ceph's benefits.

<b>Key Features & Why It's Powerful</b><br>
- Highly Scalable:<br>
Ceph is designed to scale horizontally (scale-out). To add more capacity or performance, you simply add more standard servers or disks to the cluster. It can scale from a few nodes to thousands, managing exabytes of data.<br>
- Fault-Tolerant and Self-Healing:<br>
There is no single point of failure. Data is replicated (e.g., 3 copies) or protected using erasure coding (like RAID but distributed across servers).
If a disk, a server, or even an entire rack fails, Ceph automatically detects the failure and begins replicating the affected data from the remaining copies to healthy disks elsewhere in the cluster, all without any downtime.<br>
- Software-Defined:<br>
Ceph runs on commodity hardware. You are not locked into a specific vendor. This drastically reduces costs and increases flexibility.<br>
- Unified Storage:<br>
This is one of its biggest strengths. A single Ceph cluster can simultaneously provide multiple types of storage interfaces, making it a "one-stop-shop" for many storage needs. <br>
<b>Ceph's Storage Interfaces (How You Access It) </b><br>
Ceph provides several gateway interfaces to access the storage, making it incredibly versatile.

<b>Interface	Protocol	What it's For (Analogy)</b></br>
- Ceph Object Gateway (RGW)	S3 and Swift	Like Amazon S3 or Dropbox. Ideal for storing massive amounts of unstructured data like photos, videos, backups, and logs. This is one of the most popular use cases.
- Ceph Block Device (RBD)	iSCSI-like block protocol	Like a virtual hard drive. Provides high-performance, reliable block storage that can be attached to virtual machines (e.g., in OpenStack, Kubernetes) or even physical servers.
- Ceph Filesystem (CephFS)	POSIX-compliant file system	Like a traditional network drive (NFS/SMB) but distributed. Provides a shared file system that multiple clients can mount simultaneously, with high availability and scalability.

<b>Ceph Architecture </b><br>
<img width="837" height="534" alt="ceph1" src="https://github.com/user-attachments/assets/da51cda1-6718-459a-9a3a-e0204aae0b2b" /> <br>
# How RADOS Works ? <br>

RADOS is the true heart of Ceph. Understanding RADOS is key to understanding how Ceph achieves its remarkable scalability and reliability.

**RADOS** stands for **Reliable Autonomic Distributed Object Store**. It's the foundational, scalable storage layer upon which all other Ceph services (RBD, RGW, CephFS) are built.

Think of it this way:
*   **RADOS** is the engine and chassis of a car.
*   **RBD, RGW, and CephFS** are the different body styles (sedan, truck, SUV) you can build on top of it.

---

### The Core Goal of RADOS

RADOS is designed to do one thing exceptionally well: **store objects reliably and automatically across a massive cluster of commodity hardware, with no single point of failure.**

---

### How RADOS Works: The Key Components

RADOS operates through the interaction of four key components and one brilliant algorithm.

#### 1. The Core Daemons

**a) OSD Daemons (Object Storage Daemon)**
*   **What they are:** The workhorses of the cluster. **One OSD daemon runs per disk** in the cluster (or sometimes per NVMe device for performance).
*   **What they do:**
    *   Store the actual data (as "objects") on the disk.
    *   Handle data replication, recovery, and rebalancing.
    *   Continuously "heartbeat" with other OSDs and Monitors to report their health.
*   **Key Concept:** Clients (users of RBD, RGW, etc.) communicate **directly with OSDs** to read and write data. This is crucial for performance and scalability, as it avoids a central bottleneck.

**b) Monitor Daemons (MON)**
*   **What they are:** The managers and consensus builders of the cluster. For high availability, you always have an odd number of them (typically 3 or 5).
*   **What they do:**
    *   Maintain the **cluster map**, which is a collection of several "maps" defining the state of the entire cluster.
    *   They do **NOT** store client data or handle data I/O. Their job is purely management and coordination.
    *   Provide a consistent view of the cluster state (e.g., which OSDs are up/down, what the PG layout is).
    *   Manage authentication.

#### 2. The Logical Constructs

**a) Objects**
*   This is the fundamental unit of data stored in RADOS.
*   When a file from CephFS or a block from RBD is stored, it's broken down into one or more fixed-size **objects** (default is 4MB).
*   Each object is a simple blob of data with a unique identifier (ID).

**b) Pools**
*   A **logical partition** for storing objects. You create pools for different purposes (e.g., a pool for VM disks, a pool for backup images, a pool for S3 objects).
*   Pools allow you to set data management policies:
    *   **Replication Factor:** How many copies of each object to store (e.g., size=3).
    *   **Erasure Coding Profile:** A more space-efficient data protection method.
    *   **Placement Groups:** The number of PGs for the pool (see below).
    *   **CRUSH Rules:** Which physical hardware to use (e.g., "store data only on SSDs" or "ensure copies are in different racks").

**c) Placement Groups (PGs)**
*   This is the most critical concept for understanding RADOS scalability.
*   A **Pool is split into a number of Placement Groups (PGs)**. This number is set by the administrator (e.g., `pg_num=256`).
*   **Think of a PG as a "shard" or a "container" within a pool.**
*   **How it works:**
    1.  When an object needs to be stored in a pool, it is hashed to a specific **PG**.
    2.  That **PG** is then mapped, via the CRUSH algorithm, to a set of **OSDs** (e.g., `[OSD.1, OSD.5, OSD.9]`).
    3.  The object is stored on all OSDs in that set.
*   **Why PGs are essential:** They dramatically reduce the metadata needed to track millions of objects. Instead of tracking every single object's location, the cluster only needs to track the location of a few thousand PGs. This makes the system massively scalable.

#### 3. The "Brain": The CRUSH Algorithm

**CRUSH** (Controlled Replication Under Scalable Hashing) is the algorithm that makes everything work seamlessly.

*   **What it does:** CRUSH is a deterministic algorithm that calculates **where data should be stored and where it should be found**.
*   **How it works:** It takes three inputs:
    1.  **The Cluster Map:** (From MONs) Knows all OSDs, hosts, racks, and their status.
    2.  **The CRUSH Rule:** A policy (e.g., "3 replicas, each in a different rack").
    3.  **The Object ID:** The unique name of the object.

    It performs a pseudo-random calculation and outputs a list of OSDs (e.g., `[OSD.3, OSD.8, OSD.12]`) where the object's PG should be placed.

*   **Why it's brilliant:**
    *   **Decentralized:** There is no central directory or lookup table. *Any client* can independently calculate where an object is by running the same CRUSH algorithm with the same cluster map.
    *   **Efficient:** No lookups are required for data placement.
    *   **Aware:** It understands the underlying hardware topology (racks, rows, datacenters) and places data to avoid correlated failures (e.g., it won't put two replicas in the same rack).
    *   **Balanced:** It distributes data evenly across the cluster.
    *   **Dynamic:** When you add or remove an OSD, CRUSH automatically rebalances the data by recalculating the placement for some PGs.

---

### The Data Write Process: A Step-by-Step Example

Let's say a client wants to write an object named `"vm-image-part-1"` to a pool called `rbd-pool`.

1.  **Client Calculation:** The client, which has a copy of the latest cluster map, performs two calculations:
    *   `hash("vm-image-part-1") % pg_num` --> This identifies the target **PG** (e.g., `PG 5.a`).
    *   `CRUSH(PG 5.a, Cluster Map, CRUSH Rule)` --> This returns a list of OSDs, e.g., `[OSD.2, OSD.6, OSD.9]`. `OSD.2` is designated the "Primary" OSD for this write operation.

2.  **Direct Connection:** The client connects **directly** to the Primary OSD (`OSD.2`).

3.  **Replication:** The Primary OSD is responsible for replicating the write:
    *   It writes the object to its own local disk.
    *   It simultaneously forwards the data to the other OSDs in the set (`OSD.6` and `OSD.9`), the "Replica" OSDs.
    *   Each Replica OSD writes the object to its disk and sends an acknowledgment back to the Primary.

4.  **Acknowledgment:** Once the Primary OSD has received successful acknowledgments from all Replica OSDs (or a quorum, depending on configuration), it sends a final "write successful" acknowledgment back to the client.

### Summary: The Power of RADOS

*   **Scalable:** Clients talk directly to OSDs. Adding more OSDs adds more capacity and performance simultaneously.
*   **Reliable:** Data is replicated or erasure-coded. Failures are detected and data is automatically healed.
*   **Autonomic:** The system self-manages. It heals itself, rebalances data, and handles failures with minimal administrator intervention.
*   **Decentralized:** The use of the CRUSH algorithm eliminates central bottlenecks, making the system inherently robust and scalable.

In essence, **RADOS is a sophisticated, self-managing, and massively scalable "object storage engine" that provides the bedrock upon which Ceph's unified storage capabilities are delivered.**

# What is the Role of MON ? <br>
Let's dive deep into the role of the **MON (Monitor) daemon** in Ceph.

### The High-Level Analogy

If the OSDs (Object Storage Daemons) are the **workers** in a factory, storing and retrieving data, then the MONs are the **central nervous system and management team**. They don't touch the data itself, but they are absolutely essential for coordinating the entire operation, maintaining a consistent view of the cluster's health and state, and providing the "map" that everyone needs to do their job.

---

### The Core Purpose of MONs

The primary role of the MON daemon is to **maintain and provide a consistent, authoritative view of the cluster's state.** It does this by managing the **Cluster Map**, which is actually a collection of several maps.

### Key Responsibilities of the MON

#### 1. Maintaining the Cluster Maps
This is the MON's most critical job. The cluster map is the "source of truth" for the entire system. It consists of:

*   **The MON Map:** Contains the list of all MONs in the cluster, their IP addresses, and ports. It's like the contact list for the management team itself.
*   **The OSD Map:** Contains the list of all OSDs, their status (in, out, up, down), and their weights (how much data they should store). This is the "employee roster."
*   **The PG Map (Placement Group Map):** Shows the state of all PGs (e.g., `active+clean`, `active+degraded`, `recovering`). It's the "work assignment and status board."
*   **The CRUSH Map:** Contains the hierarchical topology of the cluster (which OSDs are in which hosts, which hosts are in which racks, etc.) and the storage policies (CRUSH rules). This is the "factory floor plan" and "work instructions."
*   **The MGR Map:** Contains the list of active and standby MGR (Manager) daemons.

#### 2. Providing Consensus and Consistency (The Paxos Algorithm)
*   **Why it's needed:** In a distributed system, nodes can have temporary network partitions or failures. To avoid a "split-brain" scenario (where two parts of the cluster believe they are in charge), you need a consensus mechanism.
*   **How it works:** MONs use a variant of the **Paxos protocol** to agree on the current state of the cluster before updating the cluster map. This is why you always run an **odd number of MONs** (1, 3, 5, etc.).
    *   A majority (quorum) of MONs must agree on any state change. For example, with 3 MONs, at least 2 must agree.
    *   If a MON loses connectivity to the quorum, it stops responding to clients, ensuring there is never more than one "source of truth."

#### 3. Authentication (The Key Distributor)
*   The MON cluster manages the initial authentication for clients and daemons.
*   When a client (e.g., someone using RBD or RGW) wants to connect, it first contacts a MON.
*   The MON authenticates the client using shared keys and then provides them with the latest cluster map, allowing the client to calculate data locations and connect directly to the OSDs.

#### 4. Tracking Cluster Health
*   The MONs are responsible for collecting and reporting the overall health of the cluster.
*   When you run `ceph -s` or `ceph health`, you are querying the MON cluster.
*   It aggregates status reports from OSDs and other daemons to present a unified health status (`HEALTH_OK`, `HEALTH_WARN`, `HEALTH_ERR`).

---

### What MONs Do NOT Do

It's equally important to understand what MONs are **not** responsible for:

*   **❌ They do NOT store client data.** They never see the actual file, block, or object data that clients write.
*   **❌ They are NOT in the data path.** Once a client has the cluster map, it communicates **directly with OSDs** for all read/write operations. MONs are not a bottleneck for I/O performance.
*   **❌ They do NOT manage individual OSD operations.** They don't tell OSDs how to replicate data or when to recover; they just provide the map that the OSDs use to figure it out for themselves.

---

### The MON in Action: A Practical Example

Let's trace what happens when a client wants to write an object.

1.  **Client Bootstrapping:**
    *   The client is configured with the address of at least one MON.
    *   It connects to a MON and authenticates.

2.  **Map Retrieval:**
    *   The MON provides the client with the latest **cluster map** (MON map, OSD map, CRUSH map, etc.).

3.  **Client-Side Calculation:**
    *   The client now has everything it needs to be self-sufficient. It uses the CRUSH algorithm and the map to independently calculate:
        *   Which **PG** the object belongs to.
        *   Which set of **OSDs** (e.g., `[OSD.1, OSD.4, OSD.7]`) are responsible for that PG.

4.  **Direct I/O:**
    *   The client connects **directly** to the primary OSD in the set (`OSD.1`) and performs the write.
    *   **From this point on, the MON is out of the loop.** The client and OSDs handle the transaction.

5.  **Background Monitoring:**
    *   Meanwhile, OSDs are constantly sending heartbeat messages to the MONs and to each other.
    *   If `OSD.4` were to fail, the other OSDs would report this. The MONs would use Paxos to achieve consensus, update the OSD map to mark `OSD.4` as `down`, and broadcast the new map to the entire cluster.
    *   Clients and OSDs would then use the new map to initiate recovery operations.

### Summary: The Role of the MON

| Aspect | Role of the MON |
| :--- | :--- |
| **Primary Function** | Maintain the "source of truth" for the cluster state (the Cluster Maps). |
| **Analogy** | The central nervous system, air traffic control, or management team. |
| **In the Data Path?** | **No.** It is out of band for client I/O, which is key to Ceph's scalability. |
| **Critical for** | **Consensus, coordination, and cluster stability.** Without a MON quorum, the cluster cannot safely process writes. |
| **Handles Client Data?** | **Never.** |

In essence, the MON daemon provides the **stability and consistency** that allows the highly dynamic and distributed OSD layer to function autonomously and at massive scale. It's the silent coordinator that makes the entire Ceph dance possible.

# What is The Role of OSDs?
The **OSD (Object Storage Daemon)** is the absolute workhorse of the Ceph cluster. If MONs are the managers, then OSDs are the factory workers that do all the heavy lifting.

### The High-Level Analogy

Think of a massive warehouse:
*   The **MON** is the manager who holds the master inventory list (what should be where).
*   The **OSD** is an individual warehouse worker, each responsible for one specific section of shelves (one disk). They store, retrieve, and manage the actual goods (data) in their section.

---

### The Core Purpose of the OSD

The primary role of the OSD daemon is to **store data on a physical disk, serve data to clients, and participate in all the data replication, recovery, and rebalancing operations that make Ceph reliable and scalable.**

An OSD does two fundamental things:
1.  Stores data as "objects" on a single physical storage device (HDD/SSD).
2.  Collaborates with other OSDs to provide a unified, reliable storage service.

---

### Key Responsibilities of the OSD

#### 1. Data Storage: The "Object" in Object Storage
*   An OSD's primary job is to store **objects**. When a client writes a block from RBD or a file chunk from CephFS, it is ultimately stored as an object by an OSD.
*   Each object is a simple file stored in the OSD's local filesystem (typically `XFS` or `Btrfs`), managed by Ceph's own storage backend (**BlueStore**, which is now the default).

#### 2. Data Replication (The "R" in RADOS)
*   When a client writes an object, it doesn't just send it to one OSD. It sends it to a **Primary OSD** for a specific Placement Group (PG).
*   The **Primary OSD** is then responsible for replicating that data to the **Secondary (Replica) OSDs** in the same PG set.
*   The Primary OSD waits for acknowledgments from the replicas before confirming the write to the client. This ensures data durability.

#### 3. Data Recovery and Self-Healing
*   OSDs constantly send "heartbeat" messages to each other and to the MONs.
*   If an OSD fails or falls behind, the other OSDs in its PG sets detect it.
*   The remaining OSDs automatically start **re-replicating** the objects that were on the failed OSD to other healthy OSDs in the cluster.
*   When the failed OSD comes back online, it is brought back up to date with any changes it missed. This is **self-healing** in action.

#### 4. Rebalancing Data
*   When you add a new OSD (and disk) to the cluster, the CRUSH algorithm will reassign some PGs to this new OSD.
*   The existing OSDs that are losing those PGs will **push** the relevant objects to the new OSD.
*   This process happens automatically in the background, distributing data evenly across all OSDs in the cluster.

#### 5. Serving Client I/O (The Critical Data Path)
*   **OSDs are directly in the data path.** Once a client uses the MON-provided map to calculate which OSDs hold its data, it communicates **directly with those OSDs** for all read and write operations.
*   This is a key architectural feature that prevents bottlenecks and allows Ceph to scale performance linearly as you add more OSDs.

#### 6. Providing Local Resource Management
*   Each OSD can be tuned independently (e.g., setting read/write cache limits, network priorities).
*   The OSD daemon is responsible for managing its own disk I/O queues and ensuring it uses its local resources efficiently.

---

### The OSD in Action: A Detailed Workflow

Let's trace a client write request from the OSD's perspective.

1.  **Client Sends Write:**
    *   A client has calculated that its object belongs to `PG 3.b` and that the OSDs for this PG are `[OSD.5 (Primary), OSD.12, OSD.21]`.
    *   The client sends the write operation directly to `OSD.5`.

2.  **Primary OSD Takes Charge:**
    *   `OSD.5` writes the object to its local disk (using BlueStore).
    *   Simultaneously, it forwards the write request to the replica OSDs (`OSD.12` and `OSD.21`).

3.  **Replica OSDs Write:**
    *   `OSD.12` and `OSD.21` also write the object to their respective local disks.
    *   Once successful, they send an acknowledgment back to the Primary OSD (`OSD.5`).

4.  **Primary OSD Acknowledges:**
    *   Once `OSD.5` has received successful acknowledgments from both replicas (or a quorum, depending on configuration), it sends a final "write successful" acknowledgment back to the client.

5.  **Peer-to-Peer Health Checks:**
    *   In the background, `OSD.5`, `OSD.12`, and `OSD.21` continuously exchange heartbeat messages to ensure they are all healthy and their data is consistent.

---

### The OSD and the Physical World

*   **1 OSD Daemon ≈ 1 Physical Disk:** The standard practice is to run one OSD daemon for each physical disk (or sometimes a high-performance NVMe device) in your cluster.
*   **Storage Backend:** The OSD daemon uses a storage backend to manage how objects are laid out on the disk. The modern, high-performance default is **BlueStore**, which writes objects directly to the raw block device, bypassing the host's local filesystem for data to reduce overhead.

### Summary: The Role of the OSD

| Aspect | Role of the OSD |
| :--- | :--- |
| **Primary Function** | Store and serve data objects on a physical disk. |
| **Analogy** | The factory worker or warehouse shelf-stocker. |
| **In the Data Path?** | **Absolutely Yes.** It is the *endpoint* for all client I/O. |
| **Critical for** | **Performance, data durability, and recovery.** The cluster's total capacity and throughput are the sum of its OSDs. |
| **Handles Client Data?** | **Constantly.** It is the *only* component that touches the actual data. |

In essence, the OSD is the fundamental unit of storage in Ceph. **You cannot have a Ceph cluster without OSDs.** They transform a collection of commodity disks into a unified, resilient, and massively scalable storage system by working together in a coordinated, peer-to-peer fashion. The reliability and performance of your entire Ceph cluster depend directly on the health and performance of your OSDs.<br>

---

### 🔹 1. What is a Placement Group (PG)?

In **Ceph**, data isn’t written directly to OSDs (Object Storage Daemons).
Instead, data is divided into logical groups called **Placement Groups (PGs)**.

Each **PG** is mapped to one or more OSDs, depending on the replication or erasure coding settings.
So the hierarchy looks like this:

```
Pool → PGs → OSDs
```

---

### 🔹 2. What is a PG Backend?

The **PG backend** defines **how data is stored inside the PG**, that is, the internal mechanism for writing and managing data across OSDs.

Ceph supports multiple PG backends, depending on the type of pool you create:

| Backend                                  | Description                                                                                                      |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| **replicated**                           | Stores full copies (replicas) of the data on multiple OSDs. This is the most common and straightforward type.    |
| **erasure-coded**                        | Splits data into chunks and stores them with parity (erasure coding) to save space while maintaining redundancy. |
| **plugin-specific (like mimic backend)** | In some Ceph versions, PGs can use plugin-based backends depending on pool type.                                 |

For example, when Ceph shows:

```
creating pg 1.2a on osd.5 (pg backend = replicated)
```

It means PG `1.2a` uses the **replicated backend**, so its data is stored as multiple replicas across OSDs.

---

### 🔹 3. PG Backend vs. OSD Backend

These two are often confused, but they’re different:

| Term            | Description                                                                     |
| --------------- | ------------------------------------------------------------------------------- |
| **PG Backend**  | Defines how data is managed within a PG (replicated or erasure-coded).          |
| **OSD Backend** | Defines how data is stored on disk inside the OSD (e.g., BlueStore, FileStore). |

So conceptually:

```
Client → Pool → PG (PG backend) → OSD (OSD backend)
```

---

### 🔹 4. Example

If you have a pool named `rbd` with 3 replicas:

1. Ceph assigns your object to a PG.
2. The PG uses the **replicated backend**, so it keeps 3 copies of the data.
3. Each OSD stores the data on disk using its **OSD backend** (usually BlueStore).

---

### 🔹 Summary

| Level           | Meaning                                      | Examples                  |
| --------------- | -------------------------------------------- | ------------------------- |
| **PG Backend**  | How PGs replicate or encode data across OSDs | replicated, erasure-coded |
| **OSD Backend** | How OSDs store data on physical disks        | BlueStore, FileStore      |

---


# Ceph Installation With Cephadm (Pacific Version )
<b>1-Config Ssh For Connection Between Servers (Do it on all your servers )</b> <br>
vi /etc/ssh/sshd_config <br>
Permitrootlogin --> yes <br>
Paaswordauthentication --> yes <br>
restart sshd service <br>
<b>2-Install Cephadm  (Do it on all your servers )</b><br>
curl --silent --remote-name --location curl --silent --remote-name --location https://github.com/ceph/ceph/raw/pacific/src/cephadm/cephadm <br>
chmod +x cephadm <br>
cephadm --version (needs Docker Installation) <br>
After Install Docker we can add below registery For resolving Image restriction <br>
vi /etc/docker/deamon.json <br>
```yaml
{ 
  "registery-mirrors":[ 
    "https://repo.ficld.ir" 
  ],
  "insecure-registries":[], 
  "debug":false,
  "log-deriver":"local", 
  "log-opts":{ 
     "max-size":"500M", 
     "max-file":"3", 
  } 
} 
```
systemctl restart docker <br>
cephadm --version <br>
cephadm add -repo --release pacific <br>
ceph install <br>
ceph bootstrap --mon-ip SERVERIP (Create Config File in /etc/ceph) <br>
<b>3-Shell Access </b><br> 
cephadm shell <br>
ceph -s (Cluster status)<br>
<b>4-Authorized servers </b><br>
copy public key from /etc/ceph/ceph.pub and paste it into /root/.ssh/authorized_key <br>
all hostnames should be in /etc/hosts of All servers <br>
