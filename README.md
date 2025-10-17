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
