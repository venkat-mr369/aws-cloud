# 🔎 Advanced Interview Questions & Answers

## 1. AWS Architecture & Databases

**Q1: How would you design a highly available database architecture on AWS?**
**Answer:**

* Use **Amazon RDS Multi-AZ** for automated failover.
* For **Aurora**, use **Aurora Replicas** across AZs (up to 15 read replicas).
* **Read scaling**: Use **Aurora Global Database** for cross-region replication.
* **Backups**: RDS automated snapshots + PITR.
* **Network security**: Place DB in private subnets inside a VPC, control access via Security Groups & IAM.
* **Monitoring**: CloudWatch, Enhanced Monitoring, Performance Insights.

---

**Q2: Difference between Aurora and RDS?**
**Answer:**

* **RDS**: Manages traditional engines (MySQL, PostgreSQL, SQL Server, Oracle).
* **Aurora**: AWS proprietary MySQL/PostgreSQL-compatible engine with:

  * Storage auto-scales up to 128 TB.
  * Replication latency < 10ms.
  * 6-way replication across 3 AZs.
  * Up to 15 read replicas vs. 5 in RDS.
  * Faster failover (≈ 30 sec).
* Use **Aurora** when you need *performance + high availability at scale*.

---

## 2. Aurora Deep Dive

**Q3: Explain Aurora storage architecture.**
**Answer:**

* **Shared distributed storage** decoupled from compute.
* Data replicated 6 times across 3 AZs (two copies per AZ).
* **Quorum writes:** 4 of 6 copies must acknowledge.
* **Quorum reads:** 3 of 6 copies required.
* Auto-heals: failed block repaired from other copies.
* Compute layer (writer + replicas) runs separately, connects to storage.

---

**Q4: How does Aurora handle failover?**
**Answer:**

* Aurora cluster has:

  * **Writer node (primary)**
  * **Reader nodes (replicas)**
* If writer fails:

  * Aurora promotes the best replica (lowest lag).
  * DNS cluster endpoint automatically updates.
  * Failover typically < 30 seconds.
* Application should connect via **Cluster Endpoint**, not instance endpoint.

---

**Q5: Aurora Global Database use case?**
**Answer:**

* Supports **cross-region disaster recovery**.
* Writes in **primary region** replicate asynchronously to **read-only replicas** in other regions.
* **Use case:** A fintech app in India (primary in ap-south-1) with read replicas in Singapore (ap-southeast-1) for local read traffic & DR.

---

## 3. MSSQL on AWS (RDS & EC2)

**Q6: RDS SQL Server vs. SQL Server on EC2?**
**Answer:**

* **RDS SQL Server:**

  * Managed backups, patching, monitoring.
  * No OS-level access (limited customization).
  * Good for most OLTP workloads.
* **SQL Server on EC2:**

  * Full control (OS tuning, Agent jobs, clustering).
  * Can use **Always On Availability Groups**, SSIS, SSRS, SSAS.
  * Suitable for legacy apps requiring special configurations.

---

**Q7: How does RDS handle SQL Server HA/DR?**
**Answer:**

* Multi-AZ deployment uses **SQL Server Database Mirroring (DBM)** or **Always On Availability Groups (AG)** depending on edition.
* Automated failover with DNS redirection.
* For DR → manual setup of **cross-region read replicas** or log shipping.

---

**Q8: How to optimize SQL Server tempdb in AWS RDS?**
**Answer:**

* RDS manages storage automatically, but best practices:

  * Use instance class with more IOPS (Provisioned IOPS storage).
  * Monitor tempdb usage with Performance Insights.
  * For EC2-based SQL → configure **multiple tempdb files** (1 per CPU core up to 8).

---

## 4. PostgreSQL on AWS

**Q9: Difference between Aurora PostgreSQL and RDS PostgreSQL?**
**Answer:**

* **Aurora PostgreSQL:**

  * AWS-optimized, better performance (up to 3x faster).
  * Storage auto-healing, auto-scaling.
  * Faster failover.
* **RDS PostgreSQL:**

  * Vanilla PostgreSQL (open-source engine).
  * Full extension support (Aurora doesn’t support all extensions).
  * Useful when exact PostgreSQL compatibility is required.

---

**Q10: How do you scale PostgreSQL in AWS?**
**Answer:**

* **Vertical scaling**: Increase instance size.
* **Horizontal scaling**:

  * Use **Aurora Replicas** for read scaling.
  * Use **Logical replication** for cross-region or app-specific replication.
* **Partitioning/Sharding** for very large datasets.

---

**Q11: How does PostgreSQL handle MVCC (Multi-Version Concurrency Control)? Why important in AWS?**
**Answer:**

* PostgreSQL uses MVCC to allow **concurrent reads/writes without blocking**.
* Each transaction sees a snapshot of data at its start time.
* Deleted/updated rows create “dead tuples” cleaned by **VACUUM**.
* In AWS:

  * Autovacuum should be tuned (to prevent table bloat).
  * CloudWatch metrics + Performance Insights can monitor vacuum impact.

---

## 5. Cross-Database & Migration

**Q12: How to migrate on-prem SQL Server to AWS Aurora PostgreSQL?**
**Answer:**

* Use **AWS Database Migration Service (DMS)** + **Schema Conversion Tool (SCT)**.
* Steps:

  1. Assess schema compatibility with SCT.
  2. Convert schema → PostgreSQL.
  3. Use DMS for continuous replication (CDC) from SQL Server → Aurora.
  4. Cutover with minimal downtime.

---

**Q13: How would you choose between Aurora, RDS PostgreSQL, and SQL Server?**
**Answer:**

* **Aurora:** Best for high availability, auto-scaling, enterprise SaaS workloads.
* **RDS PostgreSQL:** Best when open-source PostgreSQL compatibility is required (e.g., custom extensions like PostGIS).
* **RDS SQL Server:** Use when app depends on SQL Server features (SSIS/SSRS/CLR).
* **EC2 SQL Server:** Only if OS-level control or clustering is required.

---

# ✅ Key Takeaways

* **Aurora**: Managed, HA, distributed storage (best for cloud-native apps).
* **RDS PostgreSQL**: Pure open-source engine, compatible, but less scalable.
* **RDS SQL Server**: Managed, limited features, but easy.
* **EC2 SQL Server**: Full control, but more admin overhead.
* **AWS Best Practices**: Always design with Multi-AZ, monitor with CloudWatch, secure with IAM + VPC, automate backups & PITR.

---
