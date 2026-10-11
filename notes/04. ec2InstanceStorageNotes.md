# EC2 Instance Storage — Section Notes

Course: Ultimate AWS Certified Solutions Architect Associate (Maarek)
Tier: **1 — heavily tested.** EBS and EFS distinctions are classic exam material, and volume type selection questions come up constantly.

---

## 1. EBS Volumes — Overview

- **EBS = Elastic Block Store** — a **network drive** you attach to an EC2 instance while it runs. Not a physical drive — think of it as a **"network USB stick."**
- Because it's network-attached (not local hardware), there's a small amount of latency communicating between instance and volume — but it can be **detached from one instance and reattached to another very quickly**, which makes it great for failover scenarios.
- **Persists independently of the instance** — you can terminate an instance, recreate one, and remount the same EBS volume to get your data back.
- **At the associate level: one EBS volume attaches to one instance at a time** (the exception — Multi-Attach — is covered below and is specific to certain volume types).
- **One instance CAN have multiple EBS volumes attached** — think of it as plugging multiple network USB sticks into one machine.
- **Bound to a specific Availability Zone** — a volume created in `us-east-1a` cannot attach to an instance in `us-east-1b`. (A snapshot is the way to move data across AZs — see below.)
- Volumes are **provisioned in advance** — you specify size (GB) and performance (IOPS) upfront, get billed for that provisioned capacity, and can increase it later.
- **Free tier**: up to 30 GB/month of General Purpose SSD (gp2/gp3) storage.

### Delete on Termination attribute
- Controls whether an EBS volume is deleted when its attached EC2 instance is terminated.
- **Root/boot volume: enabled (Yes) by default** — deleted along with the instance.
- **Any additional (non-root) volume: disabled (No) by default** — survives instance termination.
- Configurable per-volume at launch (under Advanced storage settings) — e.g. disable it on the root volume if you want to preserve boot data after terminating an instance.
- **Verified hands-on**: after terminating an instance, its root volume (delete-on-termination = yes) disappeared; a separately-attached volume (delete-on-termination = no, in a different AZ, never even attached to that instance) remained untouched.

---

## 2. EBS Snapshots

- A **backup at a point in time** of an EBS volume. Doesn't require detaching the volume first, but detaching is recommended for full data consistency.
- **Snapshots can be copied across Availability Zones or Regions** — this is the mechanism for moving EBS data between AZs (since volumes themselves are AZ-locked): take a snapshot → copy/restore it into the target AZ → create a new volume from it there.

### Snapshot features
| Feature | What it does |
|---|---|
| **EBS Snapshot Archive** | Move a snapshot to a cheaper "archive tier" — **up to 75% cheaper**. Restoring from archive takes **24–72 hours**, not immediate. |
| **Recycle Bin (for snapshots)** | Deleted snapshots land in a Recycle Bin instead of being permanently deleted immediately — protects against accidental deletion. Retention configurable from **1 day to 1 year**. |
| **Fast Snapshot Restore (FSR)** | Forces full initialization of a snapshot so there's **zero latency on first use** when creating a volume/instance from it. Useful for very large snapshots needing fast spin-up — but **expensive**, use carefully. |

- **Verified hands-on**: copying a snapshot to a different AZ and creating a volume from it worked as described; deleting a snapshot sent it to the Recycle Bin, from which it was successfully recovered.

---

## 3. AMI (Amazon Machine Image)

- **AMI = a customization of an EC2 instance** — your own software configuration, OS setup, monitoring tools, etc., "frozen" into a reusable template.
- **Benefit: faster boot/configuration time** — since all your software is pre-packaged in the AMI, a new instance launched from it doesn't need to reinstall everything (verified in hands-on: a "Hello World" page loaded almost instantly from a custom AMI vs. ~2 minutes from a fresh User Data install).
- **Three sources for AMIs:**
  1. **Public AMI** — provided by AWS (e.g. Amazon Linux 2)
  2. **Your own AMI** — built and maintained by you
  3. **AWS Marketplace AMI** — built and sold by a third party; you can also sell your own AMIs this way
- **AMI creation process**: launch + customize an instance → **stop the instance** (for data integrity) → create an image from it (this also creates EBS snapshots behind the scenes) → launch new instances from that AMI.
- **Region-specific by default** — an AMI can be copied across Regions if you want to reuse it via AWS's global infrastructure.

---

## 4. EC2 Instance Store

- **Hardware disk physically attached** to the underlying server hosting your EC2 instance (not network-attached like EBS).
- **Much higher I/O performance** than EBS — e.g. certain instance-store-backed instance types can reach millions of IOPS vs. tens of thousands for a typical EBS volume.
- **Ephemeral storage** — data is **lost if the instance is stopped or terminated**. Cannot be used for durable, long-term storage.
- **Good use cases**: buffer, cache, scratch data, temporary content — not for anything you need to persist.
- **Risk**: if the underlying physical hardware fails, the instance store fails with it — backup/replication is entirely your responsibility if you choose to use it.
- **Exam pattern**: "extremely high, hardware-level disk performance for an EC2 instance" → **Instance Store**, not EBS.

---

## 5. EBS Volume Types

Six types today, grouped into two families — **SSD** (transactional, small/frequent I/O) and **HDD** (throughput-heavy, large sequential I/O).

| Type | Category | Boot volume? | Key trait |
|---|---|---|---|
| **gp2 / gp3** | General Purpose SSD | ✅ Yes | Balanced cost/performance for most workloads |
| **io1 / io2 Block Express** | Provisioned IOPS SSD | ✅ Yes | Highest performance, for mission-critical low-latency/high-throughput workloads |
| **st1** | Throughput Optimized HDD | ❌ No | Low-cost, frequently-accessed throughput-heavy workloads (big data, log processing) |
| **sc1** | Cold HDD | ❌ No | Lowest cost, infrequently-accessed archive-style data |

### gp2 vs gp3 (know this distinction well)
- **gp3** (newer): baseline of **3,000 IOPS** and **125 MB/s throughput**, independently increasable up to **16,000 IOPS** and **1,000 MB/s** — size and performance are **decoupled**.
- **gp2** (older): IOPS and volume size are **linked** — 3 IOPS per GB, up to 16,000 IOPS max (reached at 5,334 GB). Can burst to 3,000 IOPS for smaller volumes.
- **Exam takeaway**: gp3 lets you tune performance independently of size; gp2 does not.

### Provisioned IOPS (io1/io2)
- For workloads needing **sustained, guaranteed IOPS**, especially databases sensitive to storage performance/consistency.
- io1: 4–16 TB, up to ~64,000 IOPS (Nitro instances) / 32,000 (others).
- io2 Block Express: up to 64 TB, sub-millisecond latency, up to 256,000 IOPS.
- **io1/io2 are the only volume types supporting EBS Multi-Attach** (see below).

### st1 / sc1
- Cannot be boot volumes.
- st1: max throughput 500 MB/s, max IOPS 500 — good for big data/data warehousing/log processing.
- sc1: max throughput 250 MB/s, max IOPS 250 — cheapest, for rarely-accessed archive data.

**Note from the instructor**: exact numeric limits aren't something you need to memorize precisely for the exam — understand the relative positioning (general purpose vs. provisioned IOPS vs. throughput HDD vs. cold HDD) rather than exact figures, since AWS revises these over time.

---

## 6. EBS Multi-Attach

- Allows the **same EBS volume to attach to multiple EC2 instances simultaneously** — but **only within the same Availability Zone**, and **only for io1/io2 volume types**.
- Each attached instance gets **full read/write access** — all can read and write concurrently.
- **Use cases**: higher availability for clustered Linux applications (e.g. Teradata), or applications that need to manage concurrent write operations themselves.
- **Limit: up to 16 EC2 instances** can attach to the same volume at once — this specific number is called out as exam-relevant.
- Requires a **cluster-aware file system** (not standard XFS or EXT4) to avoid data corruption from concurrent writes.

---

## 7. EBS Encryption

- When you create an encrypted EBS volume, you automatically get:
  - Data **at rest** encrypted inside the volume
  - Data **in flight** between instance and volume encrypted
  - All **snapshots** of it encrypted
  - All **volumes created from those snapshots** encrypted
- Encryption/decryption is **fully transparent** — nothing for you to manage at the application level.
- **Minimal latency impact.** Uses **KMS keys** with **AES-256**.

### How to encrypt an already-unencrypted volume (important sequence)
Since you can't just "turn on" encryption on an existing volume directly:
1. Create a snapshot of the unencrypted volume (snapshot inherits "not encrypted" status)
2. **Copy that snapshot with encryption enabled** — this is the step that actually introduces encryption
3. Create a new volume from the encrypted snapshot (it will be encrypted)
4. Attach the new encrypted volume to your instance (replacing the old unencrypted one)

**Shortcut confirmed in hands-on**: you can also go directly from an *unencrypted* snapshot to an *encrypted* volume in one step via "Create volume from snapshot" and enabling encryption there — skipping the separate "copy snapshot" step.

---

## 8. Amazon EFS (Elastic File System)

- A **managed NFS (Network File System)** — unlike EBS, it can be **mounted on many EC2 instances simultaneously, across multiple Availability Zones**.
- **Highly available, highly scalable** — but **more expensive than EBS** (roughly 3x the cost of gp2, per the course, though check current pricing).
- **Pay-per-use** — no need to provision capacity in advance; scales automatically, even to petabyte scale.
- **Linux-only** — compatible only with Linux-based AMIs, not Windows. Uses the **POSIX** file system standard.
- Can enable **encryption at rest via KMS**.
- Access is controlled via a **security group**, same mechanism as EC2.
- **Use cases**: content management, web serving, shared data, WordPress.

### Performance & Throughput modes
| Mode | Behavior |
|---|---|
| **General Purpose** (performance mode, default) | Low latency, good for latency-sensitive apps (web servers, CMS) |
| **Max I/O** (performance mode) | Higher throughput, higher latency, highly parallel — for big data/media processing |
| **Bursting** (throughput mode) | Throughput scales with storage size used |
| **Provisioned** (throughput mode) | Set a fixed throughput independent of storage size (you pay for it upfront) |
| **Elastic** (throughput mode, **AWS-recommended default**) | Automatically scales throughput up/down based on actual workload — best for unpredictable I/O, no manual tuning needed |

### Storage classes / lifecycle management
- **Standard** — frequently accessed files
- **EFS-IA (Infrequent Access)** — cheaper storage, costs more to retrieve
- **Archive** — cheapest, for rarely-accessed data (e.g. a few times a year)
- **Lifecycle policies** automatically move files between tiers based on days-since-last-access (e.g. move to IA after 30 days untouched, archive after 90, back to Standard on first re-access) — can save **up to 90%** in cost when configured well.

### Availability/durability options
- **Regional** — spans multiple AZs, highly available/durable — the right choice for **production**.
- **One Zone** — single AZ, cheaper, but not resilient to an AZ outage — fine for **development**, not production. Also compatible with the IA storage class (**One Zone-IA**).

### Hands-on details worth remembering
- EC2 console now lets you **attach an EFS file system directly during instance launch** (auto-creates and attaches the necessary security groups + mount user-data script) — no more manual mounting steps needed like in the past.
- Verified: a file created from one EC2 instance (in AZ-a) via the shared EFS mount was immediately visible and readable from a second EC2 instance in a different AZ (AZ-b) — confirming true multi-AZ shared access.

---

## 9. EFS vs. EBS — Quick Comparison

| | **EBS** | **EFS** |
|---|---|---|
| Attach to how many instances? | One at a time (unless Multi-Attach, io1/io2 only, same AZ) | Many instances, across multiple AZs |
| AZ-bound? | Yes — locked to one AZ | No — spans AZs within a Region |
| Moving across AZs | Requires a snapshot + restore | Native — just mount from any AZ |
| Backup/IO impact | Backups (snapshots) consume IO — avoid running under heavy load | N/A (managed differently) |
| Root/boot volume behavior | Deleted by default when instance terminates (configurable) | N/A — not a boot volume |
| Cost | Cheaper | Higher price point, but tiering can reduce cost significantly |
| OS support | Linux + Windows | **Linux only** |
| **Instance Store** (bonus mention) | N/A | Physically attached to host hardware — lost entirely if the instance is lost |

---

## Quick recap
- **EBS** = one-instance-at-a-time network drive, AZ-locked, persists independently of the instance.
- **Snapshots** = point-in-time EBS backups; the mechanism for moving volumes across AZs/Regions; support archiving, a recycle bin, and fast restore.
- **AMI** = a reusable, customized instance template; speeds up boot time by pre-baking your software.
- **Instance Store** = ephemeral, hardware-attached, extremely fast, but data lost on stop/terminate.
- **Volume types**: gp2/gp3 (general purpose), io1/io2 (provisioned IOPS, only ones supporting Multi-Attach), st1/sc1 (HDD, not bootable).
- **Multi-Attach**: io1/io2 only, same AZ, up to 16 instances, needs a cluster-aware file system.
- **Encryption**: transparent once enabled; retrofitting requires the snapshot → encrypted copy → new volume path.
- **EFS**: managed, multi-AZ, multi-instance NFS, Linux-only, pricier than EBS but tierable for savings.