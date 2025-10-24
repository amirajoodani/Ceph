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
## 🧱 What is BlueStore?

**BlueStore** is the **default and modern storage backend** used by Ceph OSDs to store object data directly on raw block devices.

It **replaced FileStore**, which used a filesystem (like XFS) as an intermediate layer.
BlueStore eliminates this filesystem layer to improve **performance, efficiency, and reliability**.

So, instead of:

```
Ceph → FileStore → XFS → Disk
```

You now have:

```
Ceph → BlueStore → Disk (raw)
```

This direct access gives Ceph:

* Better performance (no double buffering)
* Less overhead
* Better space efficiency
* Built-in data checksumming and compression

---

## ⚙️ Main Components of BlueStore

BlueStore consists of **three main logical parts**:

| Component                 | Description                                                                                                                                                                |
| ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **BlueFS**                | A tiny internal filesystem used only by BlueStore to store metadata, rocksdb data, and small internal files. It’s optimized for Ceph’s needs and not a general-purpose FS. |
| **RocksDB**               | A key-value database embedded inside BlueStore that keeps all metadata, such as object locations, extents, and attributes.                                                 |
| **Allocator**             | Manages free/used space on the device efficiently (decides where to write new data).                                                                                       |
| **Write-ahead log (WAL)** | Used for consistency — ensures data integrity during crashes or power failures. Often stored on a faster device (SSD/NVMe).                                                |
| **Data area**             | The main area on the disk where actual object data (the payload) is stored.                                                                                                |

---

### 🗂️ Typical BlueStore Device Layout

If you look at an OSD’s storage, you’ll usually find **three partitions or devices** (sometimes combined):

| Name          | Purpose                                               | Typical Storage Device                            |
| ------------- | ----------------------------------------------------- | ------------------------------------------------- |
| **block**     | The main data store (where object data is written)    | HDD or SSD                                        |
| **block.db**  | Holds RocksDB data and small metadata for performance | SSD/NVMe                                          |
| **block.wal** | Write-ahead log for journaling                        | SSD/NVMe (optional, often merged with `block.db`) |

You can see this layout with:

```bash
ceph-bluestore-tool show-label --dev /dev/sdX
```

---

## 🔍 How BlueStore Works (Simplified Flow)

1. A client writes an object to Ceph.
2. The OSD receives it and passes it to BlueStore.
3. BlueStore:

   * Writes metadata to RocksDB.
   * Allocates space in the data area.
   * Writes the actual object data directly to the raw block device.
   * Updates the log (WAL) if needed for consistency.
4. Once acknowledged, data is safely stored without using any external filesystem.

---

## 🧩 Comparison: BlueStore vs. FileStore

| Feature          | **BlueStore**                | **FileStore**                      |
| ---------------- | ---------------------------- | ---------------------------------- |
| Filesystem Layer | None (direct on raw device)  | Uses XFS                           |
| Performance      | Higher (no double buffering) | Lower                              |
| Space Efficiency | Better                       | Lower (due to filesystem overhead) |
| Metadata Storage | RocksDB                      | Filesystem metadata                |
| Data Integrity   | Built-in checksums           | Relies on FS                       |
| Default in Ceph  | Yes (since Luminous)         | Deprecated                         |

---

## 📊 Summary Diagram

```
          ┌────────────────────────────┐
          │          Ceph OSD          │
          ├────────────────────────────┤
          │        BlueStore           │
          │   ├──────────────┬───────┐ │
          │   │  RocksDB     │ BlueFS│ │
          │   └──────────────┴───────┘ │
          │        │                    │
          │   Write-Ahead Log (WAL)     │
          │        │                    │
          │     Data Area (block)       │
          └────────┴────────────────────┘
```

---
Let’s look at a **real-world example** of how **BlueStore** organizes its storage components on a Ceph OSD node.

You can check this directly on any Ceph node (usually under `/var/lib/ceph/osd/ceph-<id>`).

---

## 🧩 Command

Run the following command on one of your OSD nodes:

```bash
sudo ceph-volume lvm list
```

This shows how each OSD’s BlueStore layout is configured.

---

## 🧾 Example Output

Here’s an example from a real Ceph cluster:

```
====== osd.3 ======
  [block]       /dev/sdb
      LV Path         /dev/ceph-1e5c9c2f-1234-4f45-abc3-9b221fd06a9f/osd-block-1f3f2c21-0e6e-4b1d-8d47-2cb1f6c32b63
      LV Size         931.50 GiB
      Type            block
      FS Type         bluestore
      DB Device       /dev/nvme0n1p2
      WAL Device      /dev/nvme0n1p3
      OSD UUID        1f3f2c21-0e6e-4b1d-8d47-2cb1f6c32b63
      OSD ID          3
      Block UUID      b27bb52d-6b48-423b-beb1-bd7ac493fbd0
      Cephx Key       AQCsRFRkAAAAABAAK8Jrvy3tg3tCzjb1Rj6FNg==
      Cluster FSID    d8b5ab2a-8b3b-4d17-99db-65c4b6ec5b76
      Encryption      none
      Crush Device Class  hdd
```

---

## 🧠 Let’s break it down

| Field                         | Description                                                                |
| ----------------------------- | -------------------------------------------------------------------------- |
| **[block] /dev/sdb**          | The main BlueStore data device — stores actual object data.                |
| **DB Device /dev/nvme0n1p2**  | The device (often SSD/NVMe) that holds RocksDB metadata.                   |
| **WAL Device /dev/nvme0n1p3** | The write-ahead log for fast journaling (can be on the same device as DB). |
| **OSD UUID**                  | Unique ID for this OSD’s BlueStore data.                                   |
| **OSD ID**                    | Numeric OSD ID used by Ceph (like osd.3).                                  |
| **Cluster FSID**              | The UUID of the Ceph cluster this OSD belongs to.                          |

---

## 🧱 Disk Layout Visualization

Using the above example:

```
+-------------------+
| /dev/nvme0n1p2 -> block.db  (RocksDB metadata)
| /dev/nvme0n1p3 -> block.wal (Write-Ahead Log)
| /dev/sdb       -> block     (Main data area)
+-------------------+
```

You can verify what’s inside the OSD directory as well:

```bash
sudo ls /var/lib/ceph/osd/ceph-3/
```

Example output:

```
block
block.db
block.wal
current/
keyring
store.db/
superblock
```

---

## 📜 To check BlueStore details

You can inspect the device labels directly using:

```bash
sudo ceph-bluestore-tool show-label --dev /dev/sdb
```

Example output:

```
{
    "osd_uuid": "1f3f2c21-0e6e-4b1d-8d47-2cb1f6c32b63",
    "size": 1000204886016,
    "btime": "2024-08-15T10:22:11.503123+0000",
    "description": "main block device",
    "bluefs": "1"
}
```

---

## 🧾 Summary

| Component     | Description        | Typical Device      |
| ------------- | ------------------ | ------------------- |
| **block**     | Stores object data | HDD or SSD          |
| **block.db**  | RocksDB + metadata | SSD/NVMe            |
| **block.wal** | Write-ahead log    | SSD/NVMe (optional) |

---
Awesome 🔍 — here’s how you can **identify which OSDs use which NVMe devices** for `block.db` and `block.wal` across your entire Ceph cluster.

This is very useful for **performance analysis**, **capacity planning**, or when you’re trying to confirm that your metadata and WAL are on faster storage (like NVMe or SSD).

---

## 🧠 1. Using `ceph-volume lvm list`

Run the following command on **each Ceph node**:

```bash
sudo ceph-volume lvm list | egrep "osd.|block|DB Device|WAL Device"
```

This gives a concise view showing which block, DB, and WAL devices each OSD uses.
Example output:

```
====== osd.0 ======
  [block]       /dev/sdc
      DB Device       /dev/nvme0n1p1
      WAL Device      /dev/nvme0n1p2
====== osd.1 ======
  [block]       /dev/sdd
      DB Device       /dev/nvme0n1p1
      WAL Device      /dev/nvme0n1p2
====== osd.2 ======
  [block]       /dev/sde
      DB Device       None
      WAL Device      None
```

You can immediately see:

* **osd.0** and **osd.1** share the same NVMe for DB/WAL.
* **osd.2** stores both data and metadata on the same HDD (no NVMe).

---

## 🧩 2. Using a Cleaner Output (Script Format)

If you want a **clean table** with only the important info, run this shell snippet:

```bash
sudo ceph-volume lvm list | awk '
/osd\.[0-9]+/ {osd=$2}
/block\]/ {getline; block=$2}
/DB Device/ {db=$3}
/WAL Device/ {wal=$3; print osd, block, db, wal}
' | column -t
```

✅ Output example:

```
osd.0  /dev/sdc  /dev/nvme0n1p1  /dev/nvme0n1p2
osd.1  /dev/sdd  /dev/nvme0n1p1  /dev/nvme0n1p2
osd.2  /dev/sde  None             None
```

This table shows:

* Each **OSD ID**
* Its **block device** (main data area)
* Its **block.db** and **block.wal** devices (if any)

---

## 🧭 3. From the Ceph CLI (Cluster-Wide Overview)

If your cluster has multiple nodes and you have passwordless SSH or Ansible access, you can get this info from all OSD nodes in one go:

```bash
for host in $(ceph node ls osd | jq -r '.[].host' | sort -u); do
  echo "===== $host ====="
  ssh $host "sudo ceph-volume lvm list | egrep 'osd.|block|DB Device|WAL Device'"
done
```

This will print per-host results across the entire cluster.

---

## 🧾 Summary

| Field         | Description                                      |
| ------------- | ------------------------------------------------ |
| **block**     | Main data area (often HDD or large SSD)          |
| **block.db**  | RocksDB metadata (NVMe/SSD recommended)          |
| **block.wal** | Write-ahead log (optional, NVMe/SSD recommended) |
| **osd.N**     | OSD ID number that uses those devices            |

---

# possible status of OSDs:
Excellent question 👌 — understanding the **possible statuses of OSDs (Object Storage Daemons)** is critical for Ceph cluster administration, monitoring, and troubleshooting.

Let’s break it down clearly and completely 👇

---

## 🧱 What is an OSD in Ceph?

An **OSD (Object Storage Daemon)** is the service that actually stores your data as objects on a storage device.
Each OSD handles:

* Reading/writing object data
* Replication or erasure coding
* Data recovery and backfilling
* Reporting status to the Ceph monitor (MON)

Each OSD can have several **status attributes** that describe its **health, state, and cluster participation**.

---

## ⚙️ 1. Core OSD Statuses

### 🔸 **up / down**

This indicates whether the OSD **process is running and reachable**.

| Status   | Meaning                                                                   |
| -------- | ------------------------------------------------------------------------- |
| **up**   | The OSD daemon is running and responding to the monitor.                  |
| **down** | The OSD is not responding — process stopped, host down, or network issue. |

You can check it with:

```bash
ceph osd tree
```

or

```bash
ceph osd stat
```

Example:

```
ID  CLASS  WEIGHT   STATUS  REWEIGHT  PRI-AFF
0   hdd    0.93150  up      1.00000   1.00000
1   hdd    0.93150  down    1.00000   1.00000
```

---

### 🔸 **in / out**

This shows whether the OSD is **actively used for data placement**.

| Status  | Meaning                                                                                             |
| ------- | --------------------------------------------------------------------------------------------------- |
| **in**  | The OSD is part of the CRUSH map and actively stores data.                                          |
| **out** | The OSD is excluded from data placement (no new data, existing data might be rebalanced elsewhere). |

Check it with:

```bash
ceph osd dump | grep osd.X
```

or:

```bash
ceph osd tree
```

---

## 🧩 2. Combined OSD States

These statuses are often seen together:

| Combined State | Meaning                                                                                  |
| -------------- | ---------------------------------------------------------------------------------------- |
| **up + in**    | ✅ Normal — OSD is healthy and serving data.                                              |
| **up + out**   | OSD is alive but manually or automatically marked out (e.g., drained or being replaced). |
| **down + in**  | ⚠️ Problem — OSD is part of data placement but not reachable (data redundancy at risk).  |
| **down + out** | OSD is offline and not part of the data set (safe if intentionally removed).             |

---

## 🧮 3. Additional Internal or Transitional States

Ceph also tracks internal transient states during recovery or maintenance:

| State                            | Description                                                                     |
| -------------------------------- | ------------------------------------------------------------------------------- |
| **new**                          | OSD just created, not yet added to the cluster.                                 |
| **booting**                      | OSD process is starting up and initializing.                                    |
| **active**                       | OSD is fully initialized and communicating with MONs.                           |
| **clean**                        | All placement groups (PGs) on the OSD are in a consistent and replicated state. |
| **rebalancing / backfilling**    | The OSD is moving or replicating data due to CRUSH map or OSD changes.          |
| **recovering**                   | The OSD is restoring lost data from peers.                                      |
| **remapped**                     | PGs are temporarily remapped due to a topology change.                          |
| **scrubbing / deep-scrubbing**   | OSD is verifying data consistency (scheduled background task).                  |
| **noup / nodown / noin / noout** | Cluster flags that affect OSD status transitions (used for maintenance).        |

---

## 🧰 4. How to View OSD States

### Summary of OSD states:

```bash
ceph osd stat
```

Example:

```
osdmap e1234: 6 osds: 6 up, 6 in
```

### Detailed per-OSD view:

```bash
ceph osd tree
```

### Check PGs per OSD:

```bash
ceph pg dump | grep osd.X
```

---

## 🧾 Summary Table

| Category                | Status                                         | Meaning                                               |
| ----------------------- | ---------------------------------------------- | ----------------------------------------------------- |
| **Availability**        | up / down                                      | Whether the OSD daemon is running and reachable       |
| **CRUSH Participation** | in / out                                       | Whether the OSD is used for data placement            |
| **Combined State**      | up+in / up+out / down+in / down+out            | Overall OSD state                                     |
| **Operational State**   | booting, active, recovering, backfilling, etc. | Temporary working or recovery states                  |
| **Maintenance Flags**   | noup, nodown, noin, noout                      | Cluster-level flags controlling OSD state transitions |

---
Perfect 👍 — this is a key part of **Ceph OSD lifecycle management**.
Let’s go through all the common Ceph commands you’ll use to **manually change OSD states** — `out`, `in`, `down`, and `reweight` — plus when and why to use each.

---

## 🧱 1. Mark an OSD **out**

### 🔹 Command:

```bash
ceph osd out osd.<id>
```

### 🔹 Example:

```bash
ceph osd out osd.5
```

### 🔹 What it does:

* Removes OSD **from active data placement**.
* Ceph starts **backfilling** and **rebalancing** data from that OSD to others.
* It’s **still up** (daemon running), but no new data will be written to it.

### 🔹 When to use:

✅ Before maintenance or replacement
✅ When you want to **drain** an OSD safely without losing data.

---

## 🧱 2. Mark an OSD **in**

### 🔹 Command:

```bash
ceph osd in osd.<id>
```

### 🔹 Example:

```bash
ceph osd in osd.5
```

### 🔹 What it does:

* Adds the OSD back to the **CRUSH map**.
* The cluster will start **rebalancing** and sending data back to it.

### 🔹 When to use:

✅ After maintenance or re-adding a previously drained OSD.
✅ When you’ve replaced the disk and want Ceph to start using it again.

---

## 🧱 3. Mark an OSD **down**

### 🔹 Command:

```bash
ceph osd down osd.<id>
```

### 🔹 Example:

```bash
ceph osd down osd.5
```

### 🔹 What it does:

* Marks the OSD as **unreachable or failed** in the cluster map.
* Ceph monitors will stop sending IO to it.
* Usually done automatically when an OSD process or host fails.

### 🔹 When to use:

⚠️ Only if Ceph hasn’t automatically detected it as down (rare).
✅ For troubleshooting or simulating failure.

---

## 🧱 4. Reweight an OSD (Adjust Data Distribution)

### 🔹 Command:

```bash
ceph osd reweight osd.<id> <value>
```

### 🔹 Example:

```bash
ceph osd reweight osd.5 0.8
```

### 🔹 What it does:

* Changes the **relative weight** of the OSD in the CRUSH map.
* A lower weight means **less data** will be stored on that OSD.
* Helps to **balance disk usage** or **decrease load** on a specific device.

### 🔹 When to use:

✅ If one OSD is too full (e.g. 95%) and others are not.
✅ To gradually decommission a drive (by slowly lowering weight before marking it out).

---

## 🧱 5. Set CRUSH weight (Permanent Configuration-Level Weight)

If you replaced a drive and want to assign its correct physical size weight:

```bash
ceph osd crush reweight osd.<id> <weight>
```

Example:

```bash
ceph osd crush reweight osd.5 1.0
```

🔸 This is different from `ceph osd reweight`, which is **temporary** and only affects balancing logic.
`crush reweight` changes the actual **CRUSH map configuration**.

---

## 🧰 6. Verify OSD States

You can check OSDs after making any changes:

```bash
ceph osd tree
```

Example output:

```
ID  CLASS  WEIGHT   STATUS  REWEIGHT  PRI-AFF
 0   hdd    0.93150  up      1.00000   1.00000
 1   hdd    0.93150  down    1.00000   1.00000
 2   hdd    0.93150  up      0.80000   1.00000
 3   hdd    0.93150  up      1.00000   1.00000
```

You can also use:

```bash
ceph osd stat
```

Output:

```
osdmap e1234: 4 osds: 3 up, 3 in
```

---

## 🧩 7. Useful Maintenance Flags (Cluster-wide)

| Flag       | Effect                                                                           |
| ---------- | -------------------------------------------------------------------------------- |
| **noout**  | Prevents Ceph from automatically marking OSDs “out” (useful during maintenance). |
| **noup**   | Prevents OSDs from being automatically marked “up”.                              |
| **nodown** | Prevents OSDs from being marked “down”.                                          |
| **noin**   | Prevents OSDs from being automatically marked “in”.                              |

### Example:

```bash
ceph osd set noout
# ... perform maintenance ...
ceph osd unset noout
```

---

## 🧾 Summary Table

| Action           | Command                                     | Description                                | Typical Use               |
| ---------------- | ------------------------------------------- | ------------------------------------------ | ------------------------- |
| Mark Out         | `ceph osd out osd.<id>`                     | Remove OSD from data placement             | Maintenance / replacement |
| Mark In          | `ceph osd in osd.<id>`                      | Rejoin OSD to cluster                      | After maintenance         |
| Mark Down        | `ceph osd down osd.<id>`                    | Mark OSD as failed                         | Manual or test            |
| Reweight         | `ceph osd reweight osd.<id> <value>`        | Adjust data distribution                   | Balance disk usage        |
| CRUSH Reweight   | `ceph osd crush reweight osd.<id> <weight>` | Permanent weight change in CRUSH map       | Reflect disk capacity     |
| Maintenance Flag | `ceph osd set noout` / `unset noout`        | Stop auto state changes during maintenance | Node-level maintenance    |

---
Perfect 👏 — let’s go through the **safe, step-by-step procedure to replace a failed OSD** in a Ceph cluster (Bluestore-based).

This is the **officially recommended workflow** used by Ceph admins to ensure **no data loss**, **clean rebalancing**, and **healthy recovery**.

---

## 🧩 Scenario

You have an OSD (for example, `osd.5`) that has failed — disk is bad, or the OSD process won’t start — and you want to **replace it with a new disk** safely.

---

## ⚙️ Step-by-Step OSD Replacement Procedure

### 🪛 Step 1. Verify the OSD failure

Check cluster status:

```bash
ceph osd tree
```

or

```bash
ceph health detail
```

If you see:

```
osd.5   down   in
```

it means the OSD is **not reachable** but still part of the CRUSH map (data redundancy at risk).

---

### ⚙️ Step 2. Set maintenance flag

Prevent Ceph from automatically marking other OSDs “out” during the maintenance:

```bash
ceph osd set noout
```

🔸 This keeps the cluster stable while you work.

---

### ⚙️ Step 3. Mark the failed OSD “out”

Tell Ceph to stop using this OSD for data placement:

```bash
ceph osd out osd.5
```

Ceph will begin **backfilling** and **rebalancing** data from that OSD’s replicas to others.

Monitor the progress:

```bash
ceph -s
```

Wait until the cluster shows `HEALTH_OK` or all PGs are `active+clean` before proceeding.

---

### ⚙️ Step 4. Stop and remove the old OSD

On the node hosting that OSD:

```bash
sudo systemctl stop ceph-osd@5
```

Then remove it from the Ceph cluster:

```bash
ceph osd purge osd.5 --yes-i-really-mean-it
```

💡 `purge` will:

* Remove it from the cluster map
* Delete the auth key
* Clean up CRUSH entries

If your Ceph version doesn’t have `purge`, use:

```bash
ceph osd crush remove osd.5
ceph auth del osd.5
ceph osd rm 5
```

---

### ⚙️ Step 5. Physically replace or prepare the new disk

Replace the failed drive with a new one and identify it, e.g. `/dev/sdf`:

```bash
lsblk
```

---

### ⚙️ Step 6. Prepare the new OSD device

If you’re using LVM (default for Bluestore):

```bash
sudo ceph-volume lvm create --data /dev/sdf
```

Or, if you want to specify DB/WAL devices:

```bash
sudo ceph-volume lvm create --data /dev/sdf --block-db /dev/nvme0n1p1
```

Ceph will automatically:

* Create a new OSD ID
* Initialize Bluestore
* Add the new OSD to the cluster and CRUSH map

---

### ⚙️ Step 7. Verify the new OSD is up and in

Check:

```bash
ceph osd tree
```

You should see something like:

```
ID  CLASS  WEIGHT   STATUS  REWEIGHT
5   hdd    0.93150  up      1.00000
```

---

### ⚙️ Step 8. Unset maintenance flag

Once the cluster is balanced and healthy again:

```bash
ceph osd unset noout
```

---

### ⚙️ Step 9. Verify cluster health

Finally, confirm everything is clean and stable:

```bash
ceph -s
```

Expected output:

```
cluster is healthy
HEALTH_OK
all PGs are active+clean
```

---

## 🧾 Summary — OSD Replacement Lifecycle

| Step | Command                                       | Purpose                                 |
| ---- | --------------------------------------------- | --------------------------------------- |
| 1    | `ceph osd set noout`                          | Prevent auto-marking during maintenance |
| 2    | `ceph osd out osd.<id>`                       | Stop data placement on failed OSD       |
| 3    | `ceph osd purge osd.<id>`                     | Remove OSD completely                   |
| 4    | `ceph-volume lvm create --data /dev/<device>` | Create a new OSD                        |
| 5    | `ceph osd unset noout`                        | Resume normal cluster behavior          |
| 6    | `ceph -s`                                     | Verify cluster health                   |

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
