

**What makes these notes different from the previous file:**

Every topic now has **detailed explanations** written in plain English before any CLI commands:

**Day 1 — Foundations:**
- Explains WHY regions/AZs matter for telecom (latency for 120M Indian subscribers, GDPR for EU data residency)
- Multi-AZ failover diagram showing exact step-by-step process
- Shared Responsibility explained with apartment vs house analogy
- IAM deep dive — why Meena has restricted access with "Deny Always Wins" explanation
- VPC explained as "your private corporate network inside AWS" with full subnet flow
- Security Groups stateful vs NACLs stateless explained

**Day 2 — Storage:**
- EC2 vs RDS decision framework with telecom-specific reasoning
- Instance type naming decoded (r6i.2xlarge = what each letter means)
- gp3 vs gp2 cost comparison showing $145/month savings per volume
- IOPS vs Throughput vs Latency explained with CDR/billing examples
- Snapshot incremental backup explained with 1TB CDR volume example
- S3 lifecycle tiering: Standard → Standard-IA → Glacier → Deep Archive with costs

**Day 3 — Managed DB:**
- RDS architecture — what you lose (OS access) vs what you gain (automated everything)
- Full `create-db-instance` commands for MSSQL (1433), MySQL (3306), PostgreSQL (5432)
- PITR explained with timeline diagram (bad DELETE at 2:30 PM → restore to 2:29:59 PM)
- Parameter groups with **explanations of every parameter** (why MAXDOP=4, why innodb_buffer_pool=75%, why shared_buffers=25%)
- 4-layer security stack detailed

**Day 4 — Monitoring/HA/DR:**
- CloudWatch metrics table with warning/critical thresholds and telecom impact
- Multi-AZ patching process (patch standby → failover → patch old primary)
- DR strategy table with RPO/RTO for each engine
- Complete morning health check bash script for Meena

**Day 5 — Security/Automation/Cost:**
- KMS encryption flow, Secrets Manager auto-rotation
- Cost optimization table with savings percentages
- Complete DBA operating model (daily/weekly/monthly tasks)
- Full 9-database inventory across 3 regions
