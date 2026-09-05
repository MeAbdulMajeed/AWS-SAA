# EC2 — Solutions Architect Associate Level — Section Notes

Course: Ultimate AWS Certified Solutions Architect Associate (Maarek)
Tier: **1 — heavily tested.** Companion to `ec2-fundamentals-notes.md` — covers only the new SAA-level material (Private/Public/Elastic IP deep dive, Placement Groups, ENI, Hibernate). Budget setup, basic EC2 launch, Security Groups, SSH, Instance Roles, and Purchasing Options were already covered in the previous section's notes.

---

## 1. Private vs. Public vs. Elastic IP (deep dive)

- **IPv4** = 4 numbers separated by dots, ~3.7 (technically ~4.3) billion possible addresses — the space is effectively exhausted globally, which is part of why AWS is nudging people toward IPv6 and now charging for public IPv4 (see below).
- **Public IP**: unique across the *entire internet*. Identifies a machine publicly, geolocatable.
- **Private IP**: only unique *within its own private network*. Two totally different companies can reuse the same private IP range with zero conflict — the ranges are only meaningful inside their own network.
- To go from a private network to the public internet, traffic passes through a **NAT device / Internet Gateway** acting as a proxy.
- On EC2: SSH-ing into an instance **requires the public IP** (or a VPN) — you cannot use the private IP from outside AWS's network, since your laptop isn't part of that private network.
- **Public IP changes** every time you **stop and start** an instance (not on reboot — reboot ≠ stop/start). **Private IP never changes** for the life of the instance.

### Elastic IP
- A **public IPv4 address you own** for as long as you don't release it — doesn't change on stop/start.
- Attach to **one instance at a time**; can be quickly moved to a different instance (e.g. to mask a failed instance).
- **Limited to 5 per AWS account** by default (can request an increase, rarely needed in practice).
- **Instructor's explicit recommendation: avoid Elastic IPs where possible** — they're often considered a poor architectural choice. Preferred alternatives:
  - Use a **random public IP + a DNS name** (via **Route 53**) instead of relying on a fixed IP.
  - Or better yet, put a **Load Balancer** in front and don't rely on any specific public IP at all — this is called out as the best pattern for AWS architectures.
- **Cost, confirmed twice in these lectures and consistent with what we already covered**: **$0.005/hour (~$3.50–3.60/month) per public IPv4 — charged whether it's in use or not**, since Feb 1, 2024. This applies equally to a normal public IP and an Elastic IP.
- Reminder: legacy (pre–July 2025) accounts get 750 hrs/month of Public IPv4 free via the EC2 free tier for the first 12 months — see the Free Tier note in the companion EC2 fundamentals file for how this differs on newer accounts.

---

## 2. EC2 Placement Groups

Placement Groups let you influence *how* AWS physically places your instances relative to each other — without giving you direct hardware access. Three strategies:

| Strategy | Instances placed... | Networking | Failure behavior | Limit | Use case |
|---|---|---|---|---|---|
| **Cluster** | Close together, **single AZ** | ~10 Gbps low-latency, high-throughput | If the AZ fails, **all instances fail together** (high risk) | No specific count limit mentioned | Big data jobs needing fast completion; apps needing extremely low latency between instances |
| **Spread** | Each instance on **separate underlying hardware**, can span multiple AZs | Standard | Failure of one instance's hardware doesn't affect others — isolated failure | **Max 7 instances per AZ per placement group** | Critical applications where failures must be isolated from each other |
| **Partition** | Spread across up to **7 partitions per AZ**, each partition = a separate hardware rack | Standard | A partition failure doesn't affect other partitions | Scales to **hundreds of instances**, partitions can span multiple AZs in the same region | Partition-aware big data apps: Hadoop, HDFS, HBase, Cassandra, Kafka |

- **Cluster** = maximize performance, accept higher collective risk.
- **Spread** = maximize isolation, but capped at 7 per AZ — good for a *small* number of critical instances, not for scaling wide.
- **Partition** = the middle ground that scales — isolation *and* scale, by isolating at the rack level rather than the single-instance level.
- To find out which partition a specific instance is on: use the **instance metadata service**.
- In the console: **Placement Groups** is found under Network & Security. You select the strategy at creation, then choose the placement group when launching an instance (under Advanced Details).

---

## 3. Elastic Network Interfaces (ENI)

- A **logical, virtual network card** — the "how EC2 gets network access" concept.
- Every EC2 instance has at least one — the **primary ENI** (`eth0`), giving it its private IP.
- **An ENI can have:**
  - One **primary private IPv4** + one or more **secondary private IPv4s**
  - One **Elastic IP per private IPv4**, or one/more public IPv4s
  - One or more **security groups**
  - A **MAC address**
- **ENIs are bound to a specific Availability Zone** — you can't move one across AZs.
- **Key practical use: ENIs can be created independently of any instance**, then attached, detached, and reattached to a *different* instance — including while both instances are running. This gives you a controlled, movable private IP for **fast network failover**: if Instance A fails, move its ENI to Instance B, and traffic addressed to that private IP now reaches B instead.
- **Lifecycle detail confirmed in hands-on**: an ENI **auto-created alongside an instance** at launch is **automatically deleted when that instance terminates**. An ENI you **create manually and attach afterward is NOT deleted automatically** — it persists even after the instance is terminated, and doesn't cost anything extra to leave around.

---

## 4. EC2 Hibernate

- **Normal stop**: OS shuts down completely; EBS data persists; on next start, the OS **boots fresh** (slow — full boot process, app initialization, cache warm-up).
- **Hibernate**: the **in-memory (RAM) state is preserved**. Under the hood, RAM contents are dumped into a file on the **root EBS volume** before shutdown, then reloaded into memory on next start — effectively resuming exactly where it left off, without a real reboot. Verified via the `uptime` command in the hands-on: uptime continues counting from where it stopped hibernating, rather than resetting to zero like a normal stop/start would.
- **Requirements to enable Hibernate:**
  - Root volume must be an **EBS volume**, and it **must be encrypted**.
  - Root volume must have **enough free space to hold the full RAM dump**.
  - Instance RAM size must be **under 150 GB** (per the course; AWS limits can change over time — verify current limits if this matters for a real workload).
  - **Does not work on bare metal instances.**
  - Works across many instance families and both **Linux and Windows**.
  - Supported for **On-Demand, Reserved, and Spot** instances.
  - **Hibernation is meant for no more than 60 days** at a stretch.
- **Use cases**: long-running processes you don't want to fully restart, preserving RAM state (e.g. warmed caches, in-memory session data), or just wanting a much faster "resume" compared to a cold boot.

---

## Quick recap
- **Public IP** changes on stop/start; **Private IP** never does; **Elastic IP** stays fixed but costs money whether used or not — and is generally discouraged in favor of DNS (Route 53) or a Load Balancer.
- **Placement Groups**: Cluster = fast but fragile (single AZ); Spread = isolated but capped at 7/AZ; Partition = isolated *and* scalable (rack-level isolation, up to 7 partitions/AZ, hundreds of instances).
- **ENI** = a movable virtual network card; auto-created ENIs die with their instance, manually-created ones don't — useful for network-level failover.
- **Hibernate** = preserves RAM to disk instead of a cold restart; requires an encrypted root EBS volume large enough to hold the RAM dump.