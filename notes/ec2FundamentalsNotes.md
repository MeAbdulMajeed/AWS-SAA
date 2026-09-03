# EC2 Fundamentals — Section Notes

Course: Ultimate AWS Certified Solutions Architect Associate (Maarek)
Tier: **1 — heavily tested, know this deeply.** This is one of the largest, most exam-weighted sections in the whole course.

---

## ⚠️ Important update before anything else: AWS Free Tier has changed

The course describes the **old free tier model** (12 months free, 750 hrs/month of t2/t3.micro, etc.). This changed on **July 15, 2025**:

| | **Legacy accounts** (created before 15 Jul 2025) | **New accounts** (created on/after 15 Jul 2025) |
|---|---|---|
| Model | 12 months free usage on eligible services (EC2, RDS, S3, etc.) + "Always Free" services with permanent limits | **$100–$200 in credits** (choose Free Plan or Paid Plan at signup), valid for **6 months** |
| EC2 specifically | 750 hrs/month of t2.micro or t3.micro, free for 12 months | Draws down from the credit pool instead |
| What happens after | Reverts to standard pay-as-you-go | Free Plan: account auto-closes after 90-day grace period once credits/6 months run out, unless upgraded to Paid Plan |

**Practical takeaway:** if your AWS account was created recently, you're on the **credit-based model**, not the "750 free hours for 12 months" the course describes. Either way — **always set a budget alarm before hands-on labs** (which you already have from Class 1 of the bootcamp).

**"Always Free" services** (Lambda, DynamoDB, SNS, and others with permanent monthly limits) are unaffected by this change and remain free regardless of account age.

---

## 1. AWS Budgets & Billing (Course Prep)

- To access **Billing and Cost Management**, you may need to enable **IAM user and role access to billing information** from the **root account** (Root → Account → scroll to "IAM user and role access to billing information" → Activate). By default, an IAM user — even with `AdministratorAccess` — **cannot** see billing data until this is turned on.
- **Bills page**: shows month-by-month cost breakdown, and drilling into "Charges by service" shows exactly what's costing money (e.g. NAT Gateway, EBS, Elastic IP charges under EC2).
- **Free Tier page** (left sidebar): shows current + forecasted usage against free tier limits — turns red if you're forecast to exceed it.
- **AWS Budgets**: set up alerts on spend thresholds.
  - **Zero-spend budget**: alerts the moment you spend even $0.01 — good for a "should be free" learning account.
  - **Custom monthly budget**: e.g. alert at 85% of actual spend, 100% of actual spend, and 100% of *forecasted* spend — up to 3 separate email alerts per budget.

---

## 2. EC2 Overview

- **EC2 = Elastic Compute Cloud** — AWS's core **Infrastructure as a Service (IaaS)** offering.
- EC2 is really an umbrella covering several related pieces:
  - **EC2 instances** — rentable virtual machines
  - **EBS volumes** — virtual hard drives for storage
  - **ELB** — Elastic Load Balancer, distributes traffic across instances
  - **ASG** — Auto Scaling Groups, scale instance count automatically
- When configuring an instance, you choose: **OS** (Linux, Windows, or macOS), **compute** (vCPUs), **memory (RAM)**, **storage type** (network-attached via EBS/EFS, or hardware-attached via **Instance Store**), **network configuration** (speed, public IP), and **security group** (firewall rules).

### EC2 User Data (bootstrapping)
- A script passed to the instance that runs **automatically on first boot only** — never again after that, for the life of the instance.
- Used to automate setup tasks: installing updates/software, downloading files, configuring the server, etc.
- **Runs as the root user** — commands execute with full (sudo) privileges automatically, no need to prefix with `sudo`.
- The more you put in User Data, the longer the instance takes to become ready at boot.

---

## 3. Launching an EC2 Instance (Hands-On Recap)

Key choices when launching:
- **AMI (Amazon Machine Image)** — the OS template. Amazon Linux 2 was used (quick-start, free-tier eligible).
- **Instance type** — `t2.micro` (or `t3.micro` if `t2.micro` isn't available in your region) is free-tier eligible.
- **Key pair** — required for SSH access. **RSA** key type; format is **`.pem`** for Mac/Linux/Windows 10+, **`.ppk`** for Windows 7/8 (via PuTTY).
- **Network settings** — assigns a public IP by default; a **security group** controls in/out traffic (created automatically as `launch-wizard-1` if none exists).
- **Storage** — default 8 GB `gp2` EBS root volume (free tier covers up to 30 GB of General Purpose SSD). **"Delete on Termination"** is enabled by default — the volume is deleted when the instance is terminated.
- **User Data** — pasted script to auto-install a web server on first boot (used to build the "Hello World" demo site).

### Public vs. Private IP
- **Public IPv4** — used to access the instance from the internet. **Changes** if you stop and restart the instance.
- **Private IPv4** — used for internal AWS network communication. **Stays the same** for the life of the instance, even across stop/start.
- Always use `http://` not `https://` for a plain HTTP demo server — HTTPS will hang since nothing is listening on port 443 unless configured.

### Instance lifecycle
- **Stop** — instance shuts down, AWS stops billing compute time, but the attached EBS volume (and its storage cost) remains. Public IP is released (and will likely change on next start).
- **Terminate** — instance and its root volume are deleted (unless "Delete on Termination" was disabled) — cannot be undone.

---

## 4. EC2 Instance Types

**Naming convention** (e.g. `m5.2xlarge`):
- **Letter (`m`)** = Instance Class / family (what it's optimized for)
- **Number (`5`)** = Generation (hardware version — higher = newer)
- **Size (`2xlarge`)** = How much CPU/memory within that family (small → large → 2xlarge → 4xlarge, etc.)

### The 4 main families (exam-relevant)

| Family | Optimized for | Use cases | Common prefixes |
|---|---|---|---|
| **General Purpose** | Balanced compute/memory/network | Web servers, code repos | `t`, `m` |
| **Compute Optimized** | High CPU | Batch processing, media transcoding, HPC, ML, gaming servers | `c` |
| **Memory Optimized** | High RAM, fast large-dataset-in-memory processing | In-memory DBs, distributed caches (ElastiCache), real-time big data processing | `r`, `x1`, `z1` |
| **Storage Optimized** | High-speed local storage access | High-frequency OLTP, NoSQL DBs, data warehousing, distributed file systems | `i`, `g`, `h1` |

- `t2.micro`: 1 vCPU, 1 GB RAM — free tier, General Purpose.
- Useful reference site for comparing all instance types: **instances.vantage.sh** (formerly ec2instances.info — name may have changed, worth verifying current URL).

---

## 5. Security Groups

- Act as a **firewall** around EC2 instances — control inbound (into instance) and outbound (out of instance) traffic.
- **Only support ALLOW rules** — there is no explicit "deny" rule type in a security group.
- Rules reference either **IP address ranges (CIDR)** or **other security groups**.
- Rule structure: **Type** (e.g. SSH, HTTP) → **Protocol** (e.g. TCP) → **Port** → **Source** (an IP range or security group).
  - `0.0.0.0/0` = from anywhere (all IPv4 addresses)
- **One instance can have multiple security groups attached** — rules from all attached groups combine (additive).
- **One security group can be attached to multiple instances.**
- **All timeouts (SSH timeout, HTTP timeout, any connection timeout) when connecting to an EC2 instance are almost always a security group misconfiguration** — this is one of the most repeated exam/practical tips in the course.
- Default outbound rule: **allow all traffic to anywhere** (full outbound internet access) unless changed.
- Classic ports to know: **22 = SSH**, **80 = HTTP**, **443 = HTTPS**, **3389 = RDP** (Windows remote desktop, mentioned generally in this course area even if not explicitly in this transcript batch).

---

## 6. Connecting to an EC2 Instance

Three main methods:

| Method | Requires | OS |
|---|---|---|
| **SSH (terminal)** | Key pair (`.pem`), correct security group rule (port 22) | Mac, Linux, Windows 10+ |
| **PuTTY** | Converted `.ppk` key (via PuTTYgen if needed) | Windows (any version, but mainly pre-Windows 10) |
| **EC2 Instance Connect** | Nothing extra — browser-based, temporary key auto-generated by AWS | Any OS, works via browser |

### SSH command structure
```bash
ssh -i EC2Tutorial.pem ec2-user@<public-ip>
```
- `ec2-user` is the default username baked into the **Amazon Linux 2** AMI.
- Must run this command **from the directory where the `.pem` file is saved** (or provide the full path).
- If you get `Permission denied` due to unprotected key file, fix file permissions:
```bash
chmod 0400 EC2Tutorial.pem
```
- ⚠️ **EC2 Instance Connect originally worked only with Amazon Linux 2** at the time this course was recorded — this has since been expanded by AWS to support other AMIs too (e.g. some Ubuntu versions). Worth checking current AWS docs rather than assuming it's Amazon-Linux-only today.

### Common SSH troubleshooting (from course's troubleshooting guide)
- **Connection timeout** → almost always a **security group** issue (port not open).
- **Still timing out after fixing security group** → likely a **corporate/personal firewall** blocking it — use EC2 Instance Connect instead.
- **Connection refused** → instance is reachable but no SSH service running — restart or recreate instance (use Amazon Linux 2).
- **Permission denied (publickey...)** → wrong key file attached, or wrong username (must be `ec2-user` for Amazon Linux 2).
- **Worked yesterday, not today** → the instance was stopped/restarted, so its **public IP changed** — update your command/config with the new IP.

---

## 7. IAM Roles for EC2 (critical security practice)

- **Never run `aws configure` and enter personal IAM access keys directly on an EC2 instance.** This is called out strongly in the course as a serious security mistake — anyone with access to that instance (e.g. via EC2 Instance Connect) could retrieve those credentials.
- Instead: create an **IAM Role**, attach the needed policy (e.g. `IAMReadOnlyAccess`), then **attach the role to the EC2 instance** (Instance → Actions → Security → Modify IAM role).
- Once attached, the instance can make authenticated AWS CLI calls **without ever running `aws configure`** — credentials are automatically provided and rotated by AWS behind the scenes.
- Detaching the role's policy (or the role itself) immediately revokes that access — confirmed by testing `aws iam list-users` before/after.
- **Roles can take a short time to propagate** after being attached/modified — if a command fails right after attaching a role, wait a moment and retry before assuming something's wrong.

---

## 8. Public IPv4 Charges (a real, current cost gotcha)

- **Since Feb 1, 2024**, AWS charges **$0.005/hour (~$3.60/month) for every Public IPv4 address**, whether it's actively used or not.
- **EC2 specifically** still gets **750 hours/month of Public IPv4 free** for new (legacy pre-July-2025) accounts during their first 12 months — covering roughly one instance running continuously.
- **No free tier at all** for Public IPv4 used by other services — e.g. a Load Balancer's IP per AZ, or an RDS database's public IP, will incur charges immediately, with no grace period.
- This is a genuine, easy-to-miss cost trap — always check for orphaned Elastic IPs or public-facing resources you forgot to delete after hands-on labs.
- AWS is pushing this charge partly to encourage **IPv6 adoption** (no equivalent charge on IPv6), though IPv6 isn't universally supported by all ISPs yet.
- To troubleshoot IPv4 charges: **Billing → Bills → Charges by service**, or use the **VPC IP Address Manager (IPAM)** → Public IP Insights.

---

## 9. EC2 Purchasing Options

| Option | Discount vs. On-Demand | Commitment | Best for |
|---|---|---|---|
| **On-Demand** | None (baseline) | None | Short-term, unpredictable workloads |
| **Reserved Instances (Standard)** | ~72% (varies) | 1 or 3 years, locked to instance type/region/tenancy/OS | Steady-state apps, e.g. databases |
| **Convertible Reserved Instances** | ~66% (slightly less than Standard) | 1 or 3 years, but can change instance family/OS/tenancy | Long workloads needing some flexibility |
| **Savings Plans** | ~72% (similar to Reserved) | 1 or 3 years, commit to $/hour spend | Long workloads, more flexible across instance size/OS/tenancy than Reserved |
| **Spot Instances** | Up to 90% | None — can be reclaimed anytime | Fault-tolerant, interruptible workloads (batch jobs, data analysis, image processing) |
| **Dedicated Host** | N/A (most expensive) | On-demand or 1-3yr reservation | Compliance needs, per-socket/core/VM software licensing (BYOL) |
| **Dedicated Instance** | N/A | None | Hardware isolated from other AWS customers, but no visibility into the physical host itself |
| **Capacity Reservations** | None — billed at on-demand rate regardless of use | None (cancel anytime) | Guarantee capacity in a specific AZ; combine with Reserved/Savings Plans for actual discount |

**Key distinction: Dedicated Host vs. Dedicated Instance**
- **Dedicated Host** = you get the entire physical server, full visibility into sockets/cores, can control instance placement.
- **Dedicated Instance** = hardware is dedicated to your account, but you might share it with other instances *of your own* in the same account, and you have no visibility/control over placement.

### Spot Instances — deeper detail
- You set a **max price** you're willing to pay; the instance runs as long as the current spot price stays below that.
- If the spot price exceeds your max: you get a **2-minute grace period**, then either **stop** (resume later, state preserved) or **terminate** (start fresh next time) — you choose the interruption behavior.
- **Spot Block** (reserving a spot instance for a fixed 1–6 hour window without interruption) — ⚠️ **deprecated as of December 31, 2022**; you may not see this option at all in a current AWS account.
- **Spot Requests**: **one-time** (request fulfilled once, then the request itself disappears) vs. **persistent** (if the instance is interrupted/stopped, the request automatically tries to relaunch it while still valid).
- To fully terminate spot instances and stop them from relaunching: **cancel the spot request FIRST**, then terminate the instances — doing it in the reverse order causes AWS to relaunch instances to satisfy the still-active request.

### Spot Fleets
- A way to request a **combination of Spot (and optionally On-Demand) instances** across multiple instance types/AZs/AMIs ("launch pools"), up to a target capacity.
- **Allocation strategies:**
  - **Lowest price** — cheapest pool, best for short workloads, most exam-relevant/common strategy.
  - **Diversified** — spreads across all defined pools, best for availability/long workloads (if one pool disappears, others remain).
  - **Capacity optimized** — picks the pool with the best available capacity.
  - **Price capacity optimized** — picks the highest-capacity pool first, then the lowest price within it — recommended for most workloads.

---

## 10. IAM Security Tools (recap, ties to IAM section)
- **IAM Credentials Report** (account-level, CSV) — password/MFA/access-key status for every user.
- **IAM Access Advisor** ("Last Accessed" tab, user-level) — which services a user has actually used, and when; helps trim unused permissions toward least privilege.

---

## Section Summary (quick recap)
- **EC2** = rentable virtual machines (IaaS), configurable across OS/CPU/RAM/storage/network.
- **User Data** = first-boot-only bootstrap script, runs as root.
- **Security Groups** = stateful-feeling allow-only firewall rules; timeouts = almost always a security group problem.
- **Connect via**: SSH (terminal), PuTTY (older Windows), or EC2 Instance Connect (browser, no key management).
- **Never hardcode IAM credentials on an EC2 instance** — always use an **IAM Role** instead.
- **Public IPv4 now costs money** (since Feb 2024) — watch for orphaned resources.
- **Purchasing options**: On-Demand (flexible, priciest) → Reserved/Savings Plans (steady workloads, big discount) → Spot (cheapest, interruptible) → Dedicated Host/Instance (compliance/licensing) → Capacity Reservation (guarantee availability, no discount by itself).
- **AWS Free Tier changed in July 2025** — check which model applies to your account before assuming "12 months free."