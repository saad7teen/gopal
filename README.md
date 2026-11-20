# Gopal — Nexus Global Infrastructure & Deployment Documentation

This repository contains the complete infrastructure setup, disaster recovery evaluation, and end-to-end deployment documentation for **Nexus Global (Vinmar)**.

---

## 📁 Repository Contents

### 1. [End-to-End Infrastructure, Setup & Deployment Guide](./end_to_end_deployment_guide.md)
A complete, phased step-by-step implementation guide:
* **Phase 1: AWS Setup & MongoDB Atlas**
  * Launching Ubuntu 24.04 `t3.small` in Mumbai (`ap-south-1`).
  * Elastic IP allocation and security groups.
  * Free CloudWatch Auto-Recovery alarm ($0.00).
  * MongoDB Atlas M10 dedicated cluster in Mumbai.
* **Phase 2: Plain HTTP Testing via Elastic IP**
  * NGINX configuration listening on Port 80 for `_` (IP).
  * Initial `.env` configuration (`COOKIE_SECURE=false`).
  * Smoke testing API health (`/server/health`) and frontend.
* **Phase 3: GoDaddy DNS Configuration**
  * Step-by-step `A Record` configuration without touching existing email MX/TXT records.
  * DNS propagation verification via `nslookup`.
* **Phase 4: Certbot SSL & HTTPS Cutover**
  * Free Let's Encrypt SSL via Certbot.
  * Automated HTTP $\rightarrow$ HTTPS redirection.
  * Updating `.env` to `COOKIE_SECURE=true`.
* **Phase 5: GitHub Actions Automated CI/CD Pipelines**
  * Ready-to-copy workflows for Backend (`deploy-backend.yml`) and Frontend (`deploy-frontend.yml`).
  * SSH/SCP zero-downtime deployment.
* **Phase 6: Automated Daily S3 Database Backups**
  * Automated `mongodump` cron script uploading to Amazon S3.
* **Phase 7: Disaster Recovery (DR) Setup**
  * S3 Cross-Region Replication to Hyderabad using **S3 Glacier Instant Retrieval** (83% cost savings).
  * Reverse replication rule for zero-data-loss failback.
  * MongoDB Atlas cross-region cloud backup.
  * Standby DR EC2 server in Hyderabad (stopped to pay $0/hr compute).

---

### 2. [Disaster Recovery & Architecture Evaluation](./disaster_recovery_evaluation.md)
A comprehensive comparison and trade-off analysis:
* Master architecture decision table.
* Why MongoDB Multi-Region is ~$50/month cheaper than 2 separate clusters (3 nodes vs 6 nodes) and guarantees zero data loss.
* S3 Glacier Instant Retrieval economics.
* Network latency comparison: Mumbai $\leftrightarrow$ Hyderabad (~10–15 ms) vs Mumbai $\leftrightarrow$ Singapore (~50–70 ms).
* Cost breakdown of EC2 options (stopped EC2 vs AWS DRS vs AWS Backup).
