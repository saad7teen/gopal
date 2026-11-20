# Disaster Recovery & Architecture Evaluation — Nexus Global

This document evaluates the proposed Disaster Recovery (DR) and backup strategy for Nexus Global (Vinmar) and compares each component against production-ready, cost-effective alternatives.

---

## 1. Master Architecture Comparison Table

| Component | Proposed Solution in Document | Is it OK? & Monthly Cost | Better Alternative(s) *(All in One Row)* | Drawbacks & Trade-offs | Final Recommendation (Best for Your Requirements) |
|---|---|---|---|---|---|
| **Database (MongoDB Atlas)** | 2 separate M10 clusters running 24/7 (1 in Mumbai, 1 in DR) + daily midnight script (`mongodump` & `mongorestore --drop`). | **NOT OK (High Risk)**<br><br>**Cost:** High (~$130–$140/mo for 2 M10 clusters = 6 nodes running 24/7). | **Alternative 1 (Multi-Region Cluster):**<br>1 single M10 cluster spanning both regions (3 nodes total: 2 in Mumbai, 1 in DR) with continuous real-time oplog sync.<br><br>**Alternative 2 (Cross-Region Snapshot):**<br>Keep 1 M10 in Mumbai (~$65/mo). Atlas automatically takes snapshots and copies them to DR (no 2nd cluster running 24/7). | • **Proposed Drawbacks:** Loses up to 24 hours of business data; the `--drop` script can wipe the database if it crashes; wastes ~$65/mo on an idle cluster.<br><br>• **Alt 1 (Multi-Region) Drawbacks:** Adds small cross-region network lag on writes (only ~10–15ms if Hyderabad is used).<br><br>• **Alt 2 (Snapshot) Drawbacks:** Takes 15–20 min to restore the snapshot during a disaster; loses 1–4 hours of recent data. | **Alternative 1 (Multi-Region Cluster) is Best.**<br><br>• **Cheaper than proposed:** 3 nodes (~$75–$85/mo) vs proposed 6 nodes (~$130–$140/mo), saving ~$50/month.<br>• **Zero data loss:** Real-time oplog sync (RPO < 1s).<br>• Eliminates fragile midnight dump scripts entirely. |
| **App Server (EC2 Compute)** | 1 primary running in Mumbai + 1 stopped server in DR (started via Lambda on failover). | **OK**<br>(Good, cost-effective setup).<br><br>**Cost:**<br>• Mumbai running: ~$20–$40/mo.<br>• Stopped DR EC2: **Incurs ~$7–$9/mo** (EBS disk ~$3–$5/mo + stopped Elastic IP ~$3.60/mo). | **Alternative 1 (AWS Elastic Disaster Recovery / DRS):**<br>AWS continuously clones Mumbai disk to DR in real-time. Boots a clone in 5 min on disaster without managing a 2nd server.<br>*Cost:* **Incurs ~$25–$28/mo** (~$20/mo flat DRS fee + ~$5–$8/mo staging disk).<br><br>**Alternative 2 (AWS Backup - Cross-Region AMI):**<br>Takes an automated daily server image and copies it to DR.<br>*Cost:* **Incurs ~$2–$4/mo** (snapshot storage only; $0 compute).<br><br>**Free Add-on (Local Auto-Recovery):**<br>Restarts crashed instance locally in 2 minutes.<br>*Cost:* **100% Free ($0.00)**. | • **Proposed Drawbacks:** You have to deploy code and patch *both* servers every release; a local server freeze triggers an unnecessary regional failover.<br><br>• **Alt 1 (AWS DRS) Drawbacks:** Additional ongoing cost of ~$25–$28/month.<br><br>• **Alt 2 (AWS Backup) Drawbacks:** Takes 15–20 min to launch an instance from an AMI during an outage. | **Proposed Stopped EC2 + Free Auto-Recovery is Best.**<br><br>Keeping the DR server stopped incurs only minimal disk/EIP storage costs (~$7–$9/mo). Free Auto-Recovery ensures minor local glitches don't trigger an unnecessary regional move. *(Choose **AWS DRS** only if you want zero maintenance and never want to deploy to a secondary server again).* |
| **File Storage (Amazon S3)** | 1 bucket in Mumbai + 1 replica bucket in DR using Cross-Region Replication (S3 Standard). | **OK**<br>(Works fine as proposed for basic disaster recovery).<br><br>**Cost:** Medium (Storage cost is doubled: ~$0.023/GB in both regions). | Keep Cross-Region Replication, but set the DR bucket storage class to **S3 Glacier Instant Retrieval** + add a **reverse replication rule** for failback. | • **Proposed Drawbacks:** The current proposal **works completely fine** for disaster recovery, but you pay full S3 Standard price in both places, and new files uploaded during DR will not copy back to Mumbai.<br><br>• **Alternative Drawbacks:** A tiny per-GB retrieval fee applies only when files are accessed during an actual disaster. | **S3 Glacier Instant Retrieval + Reverse Rule is Best.**<br><br>The proposed solution works, but switching DR storage to Glacier Instant Retrieval cuts your DR storage bill by ~83% while still opening files in milliseconds. |
| **DR AWS Region (Network Latency)** | Primary: Mumbai (`ap-south-1`)<br>DR: **Singapore (`ap-southeast-1`)** | **OK, but higher network latency.**<br><br>**Cost:** Standard AWS pricing. | Use **Hyderabad (`ap-south-2`)** as the DR region instead of Singapore. | • **Proposed Drawbacks:** Higher network latency (~50 to 70 ms ping between Mumbai and Singapore). Slower file transfers and slower app response times on failover.<br><br>• **Alternative Drawbacks:** Hyderabad has slightly fewer niche AWS services than Singapore (all core services needed here are fully supported). | **Hyderabad (`ap-south-2`) is Best.**<br><br>Latency between Mumbai and Hyderabad is only **~10 to 15 ms** (versus ~50–70 ms for Singapore). Lower latency means faster replication and a snappier app on failover. |
| **Deployment Pipeline (CI/CD)** | Automated CI/CD pipeline deploying via SSH / SCP (`scp-deploy` & `ssh-run`). | **OK & Approved**<br>(Standard, familiar workflow).<br><br>**Cost:** Free ($0). | Use AWS Systems Manager (SSM) Run Command to deploy with Port 22 closed. | • **Proposed Drawbacks:** Port 22 must remain open to runner IPs, and SSH keys must be maintained.<br><br>• **Alternative Drawbacks:** Adds new tooling and complexity when the team is already comfortable with SSH. | **Stick with SSH (Proposed is Best).**<br><br>Your team already runs this securely and successfully in production across existing projects. No need to overcomplicate it. |
| **Automated Failover (Route 53 & Lambda)** | CloudWatch detects server down $\rightarrow$ SNS $\rightarrow$ Lambda turns on DR EC2 and updates DNS. | **OK & Approved**<br>(Standard, clean setup).<br><br>**Cost:** Very Low (~$2–$3/mo). | Keep as proposed. | • **Drawbacks:** Route 53 DNS caching on external client networks can take 5–10 minutes to update. | **Proposed Solution is Best.**<br><br>Lightweight, automated, reliable, and costs almost nothing to run. |

---

## 2. Key Architecture Insights & Detailed Cost Breakdowns

### EC2 Disaster Recovery: Exact Cost Breakdown
Every EC2 solution has specific cost implications:

1. **Proposed Solution (Stopped DR EC2):**
   * *Live Mumbai Instance:* Regular running cost (~$20–$40/mo depending on instance size).
   * *Stopped DR Instance Costs:* While stopped, compute is $0, but it **still incurs ongoing costs**:
     * **EBS Root Disk Storage:** ~$0.08/GB/month (e.g., 40 GB gp3 disk = **~$3.20/month**).
     * **Elastic IP (EIP):** AWS charges $0.005/hour for an allocated public IP that is attached to a stopped instance = **~$3.60/month**.
     * **Total DR Standby Cost:** **~$6.80 to $8.50/month**.

2. **Alternative 1 (AWS Elastic Disaster Recovery / DRS):**
   * *AWS DRS Service Charge:* $0.027/hour per server = **~$20.00/month**.
   * *Replication Staging Resources:* A tiny t3.nano instance and low-cost staging EBS disk in the DR region = **~$5.00 to $8.00/month**.
   * **Total Monthly Cost:** **~$25.00 to $28.00/month**.
   * *Advantage:* Zero maintenance; whatever runs on Mumbai continuously replicates to DR disks without deploying code to a secondary server.

3. **Alternative 2 (AWS Backup - Cross-Region AMI):**
   * *Compute Cost:* **$0.00** (No secondary server exists until disaster strikes).
   * *Snapshot Storage Cost:* Replicated Amazon EBS snapshot storage is billed at ~$0.05/GB/month = **~$2.00 to $4.00/month**.
   * **Total Monthly Cost:** **~$2.00 to $4.00/month** *(Lowest cost, but takes 15–20 minutes to launch an instance on failover)*.

4. **EC2 Instance Auto-Recovery (CloudWatch Setting):**
   * **Cost: 100% Free ($0.00).**
   * Automatically restarts a crashed or frozen EC2 instance on healthy hardware in Mumbai with the exact same IP and disk within 2 minutes.

---

### Why MongoDB Multi-Region is Cheaper than 2 Separate Clusters
* **Proposed Setup (2 Separate Clusters):**
  * Mumbai M10 = 3 nodes
  * DR M10 = 3 nodes
  * **Total = 6 running nodes 24/7 = ~$130 – $140/month.**
  * *Disadvantage:* High cost, and data is only updated once at midnight (up to 24-hour data loss).
* **Multi-Region Setup (1 Cluster spanning regions):**
  * 2 nodes in Mumbai + 1 node in Hyderabad = **3 nodes total = ~$75 – $85/month.**
  * *Advantage:* **Saves ~$50/month**, streams data live via MongoDB oplog, and achieves **zero data loss (RPO < 1s)**.

---

### Why S3 Glacier Instant Retrieval Saves 83% on DR Storage
* In standard S3 Cross-Region Replication, data is replicated to **S3 Standard** in both regions (~$0.023/GB in Mumbai and ~$0.023/GB in DR), doubling the storage bill.
* By changing the destination storage class in the replication rule to **S3 Glacier Instant Retrieval**:
  * Cost drops from **$0.023/GB** down to **~$0.004/GB** (an **83% savings** on DR storage).
  * On failover, files (PDFs, invoices, POs) load in **milliseconds**, with zero thaw or waiting time.
* Adding a **reverse replication rule** (DR $\rightarrow$ Mumbai) costs $0 while idle and ensures new files uploaded during DR are replicated back to Mumbai during failback.

---

### Network Latency Comparison (Hyderabad vs. Singapore)
* **Mumbai to Singapore (`ap-southeast-1`):** Network round-trip ping is **~50 to 70 ms**.
* **Mumbai to Hyderabad (`ap-south-2`):** Network round-trip ping is **~10 to 15 ms**.
* *Impact:* Using Hyderabad provides ~4x lower latency, ensuring real-time database replication, faster file sync, and a much faster application response time if failover occurs.
