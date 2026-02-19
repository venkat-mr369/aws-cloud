# 5-Day AWS Training Program — Detailed Notes & Explanations
## Telecom Domain — Telephony Platform
## Database Engines: MS SQL Server | MySQL | PostgreSQL

> **DBA Team:** Venkat (Sr DBA), Kishore (Sr DBA), Meena (Jr DBA)
> **Regions:** telephony1 (us-east-1), telephony2 (eu-west-3), telephony3 (ap-south-1)
> **Applications:** Billing (MSSQL), CDR/Logs (MySQL), CRM (PostgreSQL)

---
---

# DAY 1 — AWS Foundations for DBAs

---

## 1.1 AWS Global Infrastructure (Regions, AZs, Multi-AZ)

### What is AWS Global Infrastructure?

AWS Global Infrastructure is the physical backbone of Amazon Web Services. Instead of one giant data center, AWS spreads its resources across the globe in a carefully organized hierarchy. Think of it like a telecom network itself — just as your telephony platform has cell towers organized into regions and zones, AWS organizes its computing power similarly.

AWS has three levels of organization:

**Regions** are independent geographic areas. Each region is a cluster of data centers located in a specific part of the world. For example, `us-east-1` is in Northern Virginia (USA), `eu-west-3` is in Paris (France), and `ap-south-1` is in Mumbai (India). Each region is completely independent — it has its own power grid, networking, and water supply. If one region goes down entirely (a very rare event), the other regions continue to operate normally.

**Availability Zones (AZs)** are isolated locations within a region. Each AZ is one or more physical data centers with independent power, cooling, and networking. AZs within a region are connected to each other through low-latency, high-bandwidth, private fiber-optic links. The physical distance between AZs is enough to protect against local disasters (fires, floods) but close enough to maintain very low latency (typically under 2ms).

**Edge Locations** are CloudFront CDN endpoints used for caching content closer to end users. While not directly used by DBAs, they're part of the global infrastructure.

```
  TELECOM PLATFORM — AWS Global Infrastructure Mapping

  ┌─────────────────────────────────────────────────────────┐
  │                      AWS CLOUD                           │
  │                                                         │
  │  ┌─── REGION: us-east-1 (telephony1) ───────────────┐  │
  │  │  Venkat (Sr DBA) — 50 million US subscribers      │  │
  │  │                                                   │  │
  │  │  AZ: us-east-1a                                   │  │
  │  │  ├── MSSQL Primary (Billing)                      │  │
  │  │  └── MySQL Primary (CDR)                          │  │
  │  │                                                   │  │
  │  │  AZ: us-east-1b                                   │  │
  │  │  ├── MSSQL Standby (Multi-AZ failover)            │  │
  │  │  └── MySQL Read Replica                           │  │
  │  │                                                   │  │
  │  │  AZ: us-east-1c                                   │  │
  │  │  ├── PostgreSQL Primary (CRM)                     │  │
  │  │  └── PostgreSQL Standby (Multi-AZ failover)       │  │
  │  └───────────────────────────────────────────────────┘  │
  │                                                         │
  │  ┌─── REGION: eu-west-3 (telephony2) ───────────────┐  │
  │  │  Meena (Jr DBA) — 30 million EU subscribers       │  │
  │  │  Same AZ layout as telephony1                     │  │
  │  └───────────────────────────────────────────────────┘  │
  │                                                         │
  │  ┌─── REGION: ap-south-1 (telephony3) ──────────────┐  │
  │  │  Kishore (Sr DBA) — 120 million India subscribers │  │
  │  │  Same AZ layout as telephony1                     │  │
  │  └───────────────────────────────────────────────────┘  │
  └─────────────────────────────────────────────────────────┘
```

### Why Regions Matter for Telecom DBAs

When Venkat, Kishore, and Meena choose a region for their telecom databases, they consider three critical factors:

**1. Latency:** The database should be physically close to the users and applications accessing it. A subscriber in Mumbai making a call generates a CDR (Call Detail Record). If the MySQL CDR database is in Mumbai (`ap-south-1`), the write latency is ~2-5ms. If it were in Virginia (`us-east-1`), the latency would be ~150-200ms — unacceptable for real-time CDR processing.

**2. Data Residency / Compliance:** The EU's GDPR regulation requires that EU subscriber data stays within Europe. This is why Meena manages `telephony2` in `eu-west-3` (Paris) — EU billing records, CRM data, and call records must remain in an EU region.

**3. Cost:** Different regions have different pricing. `us-east-1` is typically the cheapest because it's the oldest and largest region. `ap-south-1` is more expensive but necessary for Indian subscriber latency requirements.

### What is Multi-AZ and Why is it Critical?

Multi-AZ deployment means your database has a primary instance in one AZ and a synchronized standby copy in another AZ. AWS automatically replicates every write from the primary to the standby in real-time (synchronous replication).

For a telecom company, Multi-AZ is not optional — it's mandatory. Here's why:

Consider `telephony3` (Mumbai) managed by Kishore. This region handles 120 million Indian subscribers. The billing MSSQL database processes thousands of invoices per second. If the AZ hosting the primary MSSQL instance experiences a power failure:

- **Without Multi-AZ:** The billing database goes down. No invoices can be generated. No payments can be processed. The company loses revenue for every minute of downtime. Manual recovery could take hours.

- **With Multi-AZ:** AWS detects the failure within seconds. Within approximately 60 seconds, it automatically promotes the standby in the other AZ to become the new primary. The DNS endpoint doesn't change — your applications reconnect automatically. Data loss is zero because replication was synchronous.

```
  MULTI-AZ FAILOVER — telephony3 MSSQL Billing (Kishore)

  NORMAL OPERATION:
  ┌─────────────────────┐         ┌─────────────────────┐
  │  PRIMARY             │  sync  │  STANDBY             │
  │  MSSQL Billing       │───────►│  MSSQL Billing       │
  │  ap-south-1a         │        │  ap-south-1b         │
  │  Handles all reads   │        │  No connections      │
  │  and writes          │        │  (hot standby)       │
  │                      │        │                      │
  │  DNS: prod-telephony3│        │                      │
  │  -mssql-billing.xxx  │        │                      │
  │  .rds.amazonaws.com  │        │                      │
  └─────────────────────┘         └─────────────────────┘

  AZ-a FAILS! (power outage, hardware failure)

  AFTER FAILOVER (~60 seconds):
  ┌─────────────────────┐         ┌─────────────────────┐
  │  OLD PRIMARY         │        │  NEW PRIMARY         │
  │  (offline/recovering)│        │  MSSQL Billing       │
  │  ap-south-1a         │        │  ap-south-1b         │
  │                      │        │  NOW handles all     │
  │                      │        │  reads and writes    │
  │                      │        │                      │
  │                      │        │  SAME DNS endpoint!  │
  │                      │        │  Apps reconnect      │
  │                      │        │  automatically       │
  └─────────────────────┘         └─────────────────────┘
```

### AWS CLI Commands with Explanations

```bash
# Venkat checks which AZs are available in telephony1 (us-east-1)
# This is important before creating subnets and databases — you need to know
# which AZs exist and are healthy in your region
aws ec2 describe-availability-zones \
  --region us-east-1 \
  --query 'AvailabilityZones[].{Zone:ZoneName,State:State}' \
  --output table
# OUTPUT: Shows us-east-1a, us-east-1b, us-east-1c... with "available" state

# Meena checks AZs in telephony2 (eu-west-3)
aws ec2 describe-availability-zones \
  --region eu-west-3 \
  --query 'AvailabilityZones[].{Zone:ZoneName,State:State}' \
  --output table
# Paris typically has 3 AZs: eu-west-3a, eu-west-3b, eu-west-3c

# Kishore checks AZs in telephony3 (ap-south-1)
aws ec2 describe-availability-zones \
  --region ap-south-1 \
  --query 'AvailabilityZones[].{Zone:ZoneName,State:State}' \
  --output table
# Mumbai has 3 AZs: ap-south-1a, ap-south-1b, ap-south-1c

# List ALL AWS regions globally — useful when planning new telephony regions
aws ec2 describe-regions \
  --query 'Regions[].{Name:RegionName,Endpoint:Endpoint}' \
  --output table
```

---

## 1.2 AWS Account Model & Shared Responsibility

### Understanding the Shared Responsibility Model

The Shared Responsibility Model is one of the most important concepts for any DBA moving to AWS. It clearly defines what AWS takes care of and what YOU (the DBA) are responsible for.

Think of it like renting an apartment versus owning a house. When you rent (use RDS managed service), the landlord (AWS) handles the building structure, plumbing, and electrical. You handle the furniture, decoration, and keeping the inside clean. When you own the house (use EC2 self-managed), you handle everything.

### Self-Managed (EC2) vs Managed (RDS) — Detailed Comparison

**EC2 Self-Managed Databases** — When Venkat installs SQL Server on an EC2 instance, he is responsible for absolutely everything above the hypervisor layer. This includes:

- Installing the operating system updates and security patches
- Downloading, installing, and configuring the database engine
- Setting up backup scripts and scheduling them
- Configuring replication (Always On AG for MSSQL, MySQL replication, PostgreSQL streaming replication)
- Setting up monitoring agents and log collection
- Handling failover manually or building automation
- Managing disk space, adding volumes, managing RAID
- Tuning OS-level parameters (kernel settings, swap, file descriptors)

**RDS Managed Databases** — When Venkat creates an RDS SQL Server instance, AWS handles most of the heavy lifting:

- AWS provisions the underlying hardware and virtual machine
- AWS installs and configures the operating system (you never see it)
- AWS installs the database engine
- AWS handles automated daily backups
- AWS manages Multi-AZ replication automatically
- AWS patches the OS during your chosen maintenance window
- AWS patches the DB engine during your chosen maintenance window
- AWS monitors hardware health and replaces failed components

```
  SHARED RESPONSIBILITY — TELECOM DBA TEAM

  ╔══════════════════════════════════════════════════════════════╗
  ║  WHAT VENKAT, KISHORE, MEENA MANAGE (Customer Responsibility)
  ╠══════════════════════════════════════════════════════════════╣
  ║                                                              ║
  ║  MSSQL (Billing) — Venkat:                                   ║
  ║  • Design billing schema (invoices, payments, accounts)      ║
  ║  • Write and optimize billing stored procedures              ║
  ║  • Configure MAXDOP, Cost Threshold for Parallelism          ║
  ║  • Manage SQL Server logins and database users               ║
  ║  • Set up SQL Agent jobs for batch billing runs              ║
  ║  • Configure TDE (Transparent Data Encryption)               ║
  ║  • Monitor query performance via Query Store                 ║
  ║  • Choose instance size (r6i.4xlarge for billing workload)   ║
  ║  • Set backup retention (14 days for telecom compliance)     ║
  ║                                                              ║
  ║  MySQL (CDR) — Kishore:                                      ║
  ║  • Design CDR tables with proper partitioning by date        ║
  ║  • Size InnoDB buffer pool (75% of instance memory)          ║
  ║  • Configure binary logging for replication                  ║
  ║  • Create proper indexes for CDR lookup queries              ║
  ║  • Set up slow query log to catch poor CDR queries           ║
  ║  • Manage MySQL users and privileges                         ║
  ║  • Monitor replication lag on read replicas                  ║
  ║                                                              ║
  ║  PostgreSQL (CRM) — Meena (guided by Venkat):                ║
  ║  • Design CRM schema (customers, plans, interactions)        ║
  ║  • Tune shared_buffers, work_mem, effective_cache_size       ║
  ║  • Schedule VACUUM/ANALYZE for table maintenance             ║
  ║  • Create indexes for customer lookup queries                ║
  ║  • Manage PostgreSQL roles and row-level security            ║
  ║  • Monitor connection counts and configure pooling           ║
  ║                                                              ║
  ║  ALL TEAM MEMBERS:                                           ║
  ║  • Security group rules (who can access each database)       ║
  ║  • IAM policies (who on the team can do what)                ║
  ║  • Encryption key management (KMS keys)                      ║
  ║  • Secrets Manager (database credentials rotation)           ║
  ║  • CloudWatch alarms (monitoring thresholds)                 ║
  ║  • Choosing Multi-AZ vs Single-AZ                            ║
  ║  • Snapshot management and DR planning                       ║
  ║                                                              ║
  ╠══════════════════════════════════════════════════════════════╣
  ║  WHAT AWS MANAGES (AWS Responsibility)                       ║
  ╠══════════════════════════════════════════════════════════════╣
  ║                                                              ║
  ║  • Physical data center building, security guards, cameras   ║
  ║  • Server hardware, network switches, cables                 ║
  ║  • Power supply, backup generators, UPS                      ║
  ║  • Cooling systems, fire suppression                         ║
  ║  • Hypervisor layer (VMware-equivalent virtualization)       ║
  ║  • Operating system installation and patching (RDS only)     ║
  ║  • Database engine installation and patching (RDS only)      ║
  ║  • Automated backup infrastructure (RDS only)                ║
  ║  • Multi-AZ synchronous replication (RDS only)               ║
  ║  • Storage management and auto-scaling (RDS only)            ║
  ║  • Automatic failover mechanism (RDS only)                   ║
  ║  • Hardware health monitoring and replacement                ║
  ║                                                              ║
  ╚══════════════════════════════════════════════════════════════╝
```

### Real-World Example for Telecom

Scenario: A critical security vulnerability (CVE) is discovered in SQL Server 2022.

**If billing MSSQL is on EC2:** Venkat must manually download the CU (Cumulative Update) from Microsoft, plan a maintenance window, test the patch on dev/staging, then apply it to production — potentially causing downtime. This could take days from discovery to patch.

**If billing MSSQL is on RDS:** AWS makes the security patch available. Venkat sees it as "Pending Maintenance" in the RDS console. He can choose to apply it during the next maintenance window (Sunday 4-5 AM) or apply it immediately. With Multi-AZ, the patch process is: patch standby → failover → patch old primary. Total downtime: ~60 seconds.

---

## 1.3 IAM Basics for DBAs (Users, Roles, Policies)

### What is IAM?

IAM (Identity and Access Management) is AWS's security service that controls WHO can do WHAT on WHICH resources. For a DBA, understanding IAM is critical because it determines:

- Who on the team can create or delete databases
- Who can view monitoring data
- Who can create snapshots
- Who can modify production instances
- Which AWS services can talk to each other

IAM works on the principle of **least privilege** — give each person only the permissions they absolutely need, nothing more.

### IAM Concepts Explained

**IAM Users:** Think of this as a login account. Each person on the DBA team gets their own IAM user. Venkat gets `venkat.dba`, Kishore gets `kishore.dba`, and Meena gets `meena.dba`. Each user has their own credentials (password for console, access keys for CLI).

**IAM Groups:** Instead of assigning permissions to each user individually, you create groups and assign permissions to the group. All users in the group inherit those permissions. This makes management much easier — when a new DBA joins the team, you just add them to the appropriate group.

**IAM Policies:** A policy is a JSON document that defines permissions. It specifies which API actions are allowed or denied on which AWS resources. Policies can be attached to users, groups, or roles.

**IAM Roles:** A role is like a temporary hat that a service or user can put on to gain specific permissions. For example, an RDS instance might need a role to write monitoring data to CloudWatch, or an EC2 instance might need a role to read backups from S3. Roles don't have permanent credentials — they use temporary security tokens.

**MFA (Multi-Factor Authentication):** Adds a second factor (like a phone app code) on top of the password. Critical for production database access.

### Telecom IAM Architecture — Detailed

```
  IAM STRUCTURE FOR TELECOM DBA TEAM

  ┌─────────────────── AWS ACCOUNT ────────────────────────────┐
  │                                                             │
  │  ┌─── GROUP: Telecom-DBA-SeniorTeam ───────────────────┐  │
  │  │                                                      │  │
  │  │  Members: venkat.dba, kishore.dba                    │  │
  │  │                                                      │  │
  │  │  Attached Policies:                                  │  │
  │  │  ├── AmazonRDSFullAccess                            │  │
  │  │  │   (create, modify, delete RDS instances)          │  │
  │  │  ├── CloudWatchFullAccess                            │  │
  │  │  │   (create alarms, dashboards, read metrics)       │  │
  │  │  ├── AmazonS3FullAccess                             │  │
  │  │  │   (manage backup buckets)                         │  │
  │  │  ├── SecretsManagerReadWrite                         │  │
  │  │  │   (manage DB credentials)                         │  │
  │  │  └── Custom: Telecom-Senior-DBA-Policy               │  │
  │  │      (specific telecom permissions)                  │  │
  │  └──────────────────────────────────────────────────────┘  │
  │                                                             │
  │  ┌─── GROUP: Telecom-DBA-JuniorTeam ───────────────────┐  │
  │  │                                                      │  │
  │  │  Members: meena.dba                                  │  │
  │  │                                                      │  │
  │  │  Attached Policies:                                  │  │
  │  │  ├── AmazonRDSReadOnlyAccess                        │  │
  │  │  │   (can view instances, cannot modify)             │  │
  │  │  ├── CloudWatchReadOnlyAccess                        │  │
  │  │  │   (can view metrics, cannot create alarms)        │  │
  │  │  ├── AmazonRDSPerformanceInsightsReadOnly            │  │
  │  │  │   (can view Performance Insights dashboards)      │  │
  │  │  └── Custom: Telecom-Junior-DBA-Policy               │  │
  │  │      (can create snapshots + view logs,              │  │
  │  │       CANNOT delete/modify production instances)     │  │
  │  └──────────────────────────────────────────────────────┘  │
  │                                                             │
  │  ┌─── ROLE: RDS-Monitoring-Role ────────────────────────┐  │
  │  │  Trust: monitoring.rds.amazonaws.com                  │  │
  │  │  Policy: CloudWatchLogsFullAccess                     │  │
  │  │  Used by: RDS Enhanced Monitoring feature             │  │
  │  └──────────────────────────────────────────────────────┘  │
  │                                                             │
  │  ┌─── ROLE: RDS-S3-Export-Role ─────────────────────────┐  │
  │  │  Trust: rds.amazonaws.com                             │  │
  │  │  Policy: S3 write to telecom-db-backups-prod bucket  │  │
  │  │  Used by: RDS snapshot export to S3 feature          │  │
  │  └──────────────────────────────────────────────────────┘  │
  └─────────────────────────────────────────────────────────────┘
```

### Why Meena (Jr DBA) Has Restricted Access

As a Junior DBA, Meena handles daily monitoring, backup verification, and routine operations. She should NOT have the ability to:

- Delete a production database (catastrophic data loss)
- Modify production instance class (could cause unexpected downtime)
- Reboot production instances (service interruption)
- Change security groups on production (security risk)

However, she SHOULD be able to:

- View all instance details (for monitoring)
- Create snapshots (for pre-change backups)
- Download logs (for troubleshooting)
- View CloudWatch metrics (for daily health checks)
- View Performance Insights (for query analysis)

This is the principle of least privilege in action.

### Custom Policy Deep Dive — Junior DBA (Meena)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowViewAndMonitor",
      "Effect": "Allow",
      "Action": [
        "rds:Describe*",
        "rds:List*",
        "rds:DownloadDBLogFilePortion",
        "rds:DownloadCompleteDBLogFile"
      ],
      "Resource": "*",
      "Comment": "Meena can view all RDS instances and download logs across all regions"
    },
    {
      "Sid": "AllowSnapshotCreation",
      "Effect": "Allow",
      "Action": [
        "rds:CreateDBSnapshot",
        "rds:CreateDBClusterSnapshot"
      ],
      "Resource": "*",
      "Comment": "Meena can create snapshots - important for pre-change safety"
    },
    {
      "Sid": "DenyDestructiveActionsOnProd",
      "Effect": "Deny",
      "Action": [
        "rds:DeleteDBInstance",
        "rds:DeleteDBCluster",
        "rds:DeleteDBSnapshot",
        "rds:ModifyDBInstance",
        "rds:ModifyDBCluster",
        "rds:RebootDBInstance",
        "rds:StopDBInstance",
        "rds:StartDBInstance"
      ],
      "Resource": [
        "arn:aws:rds:*:*:db:prod-telephony*",
        "arn:aws:rds:*:*:cluster:prod-telephony*"
      ],
      "Comment": "Meena CANNOT delete, modify, reboot, stop, or start ANY production database. The Deny effect takes precedence over any Allow."
    },
    {
      "Sid": "AllowDevTestOperations",
      "Effect": "Allow",
      "Action": [
        "rds:ModifyDBInstance",
        "rds:RebootDBInstance",
        "rds:StopDBInstance",
        "rds:StartDBInstance"
      ],
      "Resource": [
        "arn:aws:rds:*:*:db:dev-telephony*",
        "arn:aws:rds:*:*:db:test-telephony*"
      ],
      "Comment": "Meena CAN modify/reboot/stop/start dev and test databases"
    }
  ]
}
```

**Important IAM Concept — Deny Always Wins:** In IAM, if there is both an Allow and a Deny for the same action, the Deny always takes precedence. This is why Meena cannot accidentally delete a production database even if some other policy grants her broad access.

### AWS CLI Commands with Explanations

```bash
# Step 1: Create IAM users for each team member
# Each user gets unique credentials for AWS Console and CLI access
aws iam create-user --user-name venkat.dba
aws iam create-user --user-name kishore.dba
aws iam create-user --user-name meena.dba

# Step 2: Create groups based on role level
aws iam create-group --group-name Telecom-DBA-SeniorTeam
aws iam create-group --group-name Telecom-DBA-JuniorTeam

# Step 3: Add users to appropriate groups
aws iam add-user-to-group --user-name venkat.dba --group-name Telecom-DBA-SeniorTeam
aws iam add-user-to-group --user-name kishore.dba --group-name Telecom-DBA-SeniorTeam
aws iam add-user-to-group --user-name meena.dba --group-name Telecom-DBA-JuniorTeam

# Step 4: Attach AWS managed policies to Senior DBA group
# AmazonRDSFullAccess = can create, modify, delete any RDS resource
aws iam attach-group-policy --group-name Telecom-DBA-SeniorTeam \
  --policy-arn arn:aws:iam::aws:policy/AmazonRDSFullAccess
# CloudWatchFullAccess = can create alarms, dashboards, view all metrics
aws iam attach-group-policy --group-name Telecom-DBA-SeniorTeam \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchFullAccess
# S3 access for managing backup buckets
aws iam attach-group-policy --group-name Telecom-DBA-SeniorTeam \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

# Step 5: Attach restricted policies to Junior DBA group
aws iam attach-group-policy --group-name Telecom-DBA-JuniorTeam \
  --policy-arn arn:aws:iam::aws:policy/AmazonRDSReadOnlyAccess
aws iam attach-group-policy --group-name Telecom-DBA-JuniorTeam \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchReadOnlyAccess

# Step 6: Create custom policy for Meena and attach to Junior group
aws iam create-policy \
  --policy-name Telecom-Junior-DBA-Policy \
  --policy-document file://meena-junior-policy.json
aws iam attach-group-policy --group-name Telecom-DBA-JuniorTeam \
  --policy-arn arn:aws:iam::123456789012:policy/Telecom-Junior-DBA-Policy

# Step 7: Create access keys for CLI usage
# Each DBA needs access keys to use the AWS CLI from their workstation
aws iam create-access-key --user-name venkat.dba
# OUTPUT: AccessKeyId and SecretAccessKey — save these securely!

# Step 8: Enable MFA for production access (critical security measure)
# Each DBA registers their virtual MFA device (Google Authenticator, Authy, etc.)
aws iam enable-mfa-device \
  --user-name venkat.dba \
  --serial-number arn:aws:iam::123456789012:mfa/venkat.dba \
  --authentication-code1 123456 \
  --authentication-code2 789012

# Verify: List all users in the DBA groups
aws iam get-group --group-name Telecom-DBA-SeniorTeam \
  --query 'Users[].UserName' --output table
aws iam get-group --group-name Telecom-DBA-JuniorTeam \
  --query 'Users[].UserName' --output table
```

---

## 1.4 VPC Concepts (Private/Public Subnets)

### What is a VPC?

A VPC (Virtual Private Cloud) is your own private, isolated network within AWS. Think of it as building your own corporate network inside AWS. You control everything: the IP address range, subnets, route tables, and access rules.

Every AWS resource that needs networking (EC2 instances, RDS databases, Lambda functions) must live inside a VPC. Without a VPC, your resources would be sitting on the public internet with no network isolation — extremely dangerous for databases.

### Why VPCs are Critical for Telecom Databases

The telecom platform handles highly sensitive data:
- **Billing data** (credit card numbers, bank accounts, payment history)
- **CDR data** (who called whom, when, how long — regulated by telecom authorities)
- **CRM data** (subscriber personal information — name, address, ID numbers)

This data MUST be in private subnets — never directly accessible from the internet. The VPC provides the network isolation that protects this data.

### Understanding Subnets — Public vs Private

A subnet is a smaller network within your VPC. You divide your VPC's IP range into subnets, and each subnet lives in a specific AZ.

**Public Subnets** have a route to an Internet Gateway, which means resources in public subnets can have public IP addresses and communicate directly with the internet. You put things here that NEED internet access: bastion hosts (SSH jump boxes), NAT Gateways, load balancers.

**Private Subnets** do NOT have a direct route to the internet. Resources here cannot be reached from the internet, and they cannot reach the internet unless they go through a NAT Gateway. You put your databases here — they're invisible and unreachable from the internet.

```
  TELECOM VPC ARCHITECTURE — telephony1 (Venkat)

  ┌────────────── VPC: 10.0.0.0/16 (telephony1-vpc) ──────────────┐
  │  Region: us-east-1                                              │
  │  Managed by: Venkat (Sr DBA)                                    │
  │  Total IPs available: 65,536                                    │
  │                                                                 │
  │  ┌─── PUBLIC SUBNET (10.0.100.0/24) — AZ: us-east-1a ───┐    │
  │  │                                                         │    │
  │  │  INTERNET GATEWAY ◄──── Direct internet access          │    │
  │  │       │                                                 │    │
  │  │  ┌────┴────┐  ┌───────────┐                            │    │
  │  │  │ NAT GW  │  │ Bastion   │  Venkat/Meena SSH in       │    │
  │  │  │(for DB  │  │ Host      │  through bastion to        │    │
  │  │  │ patches)│  │ 10.0.100.5│  reach private DBs         │    │
  │  │  └────┬────┘  └───────────┘                            │    │
  │  └───────┼─────────────────────────────────────────────────┘    │
  │          │                                                      │
  │          │ NAT Gateway allows private instances to              │
  │          │ reach internet (for patches, updates) but            │
  │          │ internet CANNOT reach private instances               │
  │          │                                                      │
  │  ┌───── ▼── PRIVATE SUBNET 1 (10.0.1.0/24) — AZ: us-east-1a ─┐│
  │  │                                                              ││
  │  │  RDS MSSQL Primary (Billing)     10.0.1.10                  ││
  │  │  RDS MySQL Primary (CDR)         10.0.1.20                  ││
  │  │                                                              ││
  │  │  These databases have NO public IP.                         ││
  │  │  They CANNOT be reached from the internet.                  ││
  │  │  Only accessible from within the VPC.                       ││
  │  └──────────────────────────────────────────────────────────────┘│
  │                                                                 │
  │  ┌─── PRIVATE SUBNET 2 (10.0.2.0/24) — AZ: us-east-1b ──────┐│
  │  │                                                              ││
  │  │  RDS MSSQL Standby (Multi-AZ)   10.0.2.10                  ││
  │  │  RDS MySQL Read Replica          10.0.2.20                  ││
  │  │                                                              ││
  │  │  In a DIFFERENT AZ for Multi-AZ high availability           ││
  │  └──────────────────────────────────────────────────────────────┘│
  │                                                                 │
  │  ┌─── PRIVATE SUBNET 3 (10.0.3.0/24) — AZ: us-east-1c ──────┐│
  │  │                                                              ││
  │  │  RDS PostgreSQL Primary (CRM)    10.0.3.10                  ││
  │  │  RDS PostgreSQL Standby          10.0.3.20                  ││
  │  │                                                              ││
  │  │  In a THIRD AZ for maximum fault tolerance                  ││
  │  └──────────────────────────────────────────────────────────────┘│
  └─────────────────────────────────────────────────────────────────┘
```

### What is a DB Subnet Group?

An RDS DB Subnet Group is a collection of subnets that you designate for your RDS database instances. When you create an RDS instance, you specify which DB Subnet Group to use, and RDS will place your database in one of those subnets.

**Critical Rule:** A DB Subnet Group MUST contain subnets in at least 2 different AZs. This is mandatory even for single-AZ deployments because AWS needs the flexibility to move your database to another AZ if needed.

For Multi-AZ deployments, the primary and standby will be placed in different subnets/AZs within your DB Subnet Group.

### AWS CLI Commands — Complete VPC Setup for telephony1

```bash
# =====================================================
# Venkat: Setting up VPC for telephony1 (us-east-1)
# =====================================================

# Step 1: Create the VPC with CIDR block 10.0.0.0/16
# This gives us 65,536 IP addresses — more than enough for all our databases
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --region us-east-1
# OUTPUT: VpcId = vpc-tel1 (save this ID)

# Step 2: Tag the VPC for easy identification
aws ec2 create-tags --resources vpc-tel1 \
  --tags Key=Name,Value=telephony1-vpc \
         Key=Environment,Value=production \
         Key=ManagedBy,Value=Venkat \
  --region us-east-1

# Step 3: Enable DNS hostnames and support
# CRITICAL for RDS! Without this, RDS endpoints won't resolve properly
aws ec2 modify-vpc-attribute --vpc-id vpc-tel1 --enable-dns-hostnames --region us-east-1
aws ec2 modify-vpc-attribute --vpc-id vpc-tel1 --enable-dns-support --region us-east-1

# Step 4: Create private subnets for databases (one per AZ)
# Subnet 1: AZ-a — MSSQL Primary + MySQL Primary
aws ec2 create-subnet \
  --vpc-id vpc-tel1 \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --region us-east-1
# → sub-tel1-priv1

# Subnet 2: AZ-b — MSSQL Standby + MySQL Replica
aws ec2 create-subnet \
  --vpc-id vpc-tel1 \
  --cidr-block 10.0.2.0/24 \
  --availability-zone us-east-1b \
  --region us-east-1
# → sub-tel1-priv2

# Subnet 3: AZ-c — PostgreSQL Primary + Standby
aws ec2 create-subnet \
  --vpc-id vpc-tel1 \
  --cidr-block 10.0.3.0/24 \
  --availability-zone us-east-1c \
  --region us-east-1
# → sub-tel1-priv3

# Step 5: Create public subnet (for bastion and NAT Gateway)
aws ec2 create-subnet \
  --vpc-id vpc-tel1 \
  --cidr-block 10.0.100.0/24 \
  --availability-zone us-east-1a \
  --region us-east-1
# → sub-tel1-pub

# Step 6: Create Internet Gateway and attach to VPC
# This allows public subnet resources to reach the internet
aws ec2 create-internet-gateway --region us-east-1
# → igw-tel1
aws ec2 attach-internet-gateway --internet-gateway-id igw-tel1 --vpc-id vpc-tel1

# Step 7: Create NAT Gateway in public subnet
# First, allocate an Elastic IP for the NAT Gateway
aws ec2 allocate-address --domain vpc --region us-east-1
# → eipalloc-xxx
aws ec2 create-nat-gateway \
  --subnet-id sub-tel1-pub \
  --allocation-id eipalloc-xxx \
  --region us-east-1
# → nat-tel1

# Step 8: Create route tables
# Public route table: routes traffic to Internet Gateway
aws ec2 create-route-table --vpc-id vpc-tel1 --region us-east-1
# → rtb-pub-tel1
aws ec2 create-route --route-table-id rtb-pub-tel1 \
  --destination-cidr-block 0.0.0.0/0 --gateway-id igw-tel1
aws ec2 associate-route-table --route-table-id rtb-pub-tel1 --subnet-id sub-tel1-pub

# Private route table: routes internet traffic through NAT Gateway
aws ec2 create-route-table --vpc-id vpc-tel1 --region us-east-1
# → rtb-priv-tel1
aws ec2 create-route --route-table-id rtb-priv-tel1 \
  --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-tel1
aws ec2 associate-route-table --route-table-id rtb-priv-tel1 --subnet-id sub-tel1-priv1
aws ec2 associate-route-table --route-table-id rtb-priv-tel1 --subnet-id sub-tel1-priv2
aws ec2 associate-route-table --route-table-id rtb-priv-tel1 --subnet-id sub-tel1-priv3

# Step 9: Create DB Subnet Group (REQUIRED for RDS)
# This tells RDS which subnets it can use for database placement
aws rds create-db-subnet-group \
  --db-subnet-group-name telephony1-db-subnets \
  --db-subnet-group-description "Private subnets for telecom databases in telephony1" \
  --subnet-ids sub-tel1-priv1 sub-tel1-priv2 sub-tel1-priv3 \
  --region us-east-1

# =====================================================
# Meena: Setting up VPC for telephony2 (eu-west-3)
# =====================================================
aws ec2 create-vpc --cidr-block 10.1.0.0/16 --region eu-west-3
# Follow same steps with 10.1.x.x subnets

# =====================================================
# Kishore: Setting up VPC for telephony3 (ap-south-1)
# =====================================================
aws ec2 create-vpc --cidr-block 10.2.0.0/16 --region ap-south-1
# Follow same steps with 10.2.x.x subnets
```

---

## 1.5 Security Groups & Network ACLs

### What are Security Groups?

A Security Group acts as a virtual firewall for your AWS resources. Every RDS instance, EC2 instance, and other network-connected resource has one or more security groups that control inbound (incoming) and outbound (outgoing) traffic.

**Key characteristics of Security Groups:**

1. **Stateful:** If you allow incoming traffic on port 1433 (SQL Server), the response traffic is automatically allowed back out. You don't need to create a separate outbound rule for the response.

2. **Allow-only rules:** You can only create ALLOW rules. You cannot create DENY rules. If traffic doesn't match any rule, it's denied by default.

3. **Instance-level:** Security groups are attached directly to the network interface of each instance. This means different instances in the same subnet can have different security group rules.

4. **Reference other security groups:** Instead of specifying IP addresses, you can reference another security group as the source. This is powerful — it means "allow traffic from any instance that has this security group attached."

### What are Network ACLs?

Network ACLs (NACLs) are an additional layer of security at the subnet level. Key differences from Security Groups:

- **Stateless:** You must explicitly allow BOTH inbound AND outbound traffic.
- **Allow AND Deny rules:** Unlike security groups, NACLs can have deny rules.
- **Evaluated in order:** Rules are numbered and evaluated from lowest to highest. First match wins.
- **Subnet level:** Applies to ALL resources in the subnet.

For most DBA tasks, security groups are the primary tool. NACLs are a secondary defense layer.

### Security Groups for Telecom Databases — Deep Dive

Each database engine needs its own security group because they use different ports and have different access patterns:

```
  TELECOM SECURITY GROUPS — DETAILED

  ┌─── SG: telephony-mssql-sg (Billing MSSQL) ────────────────────┐
  │                                                                 │
  │  INBOUND RULES:                                                 │
  │  ┌──────────┬───────┬───────────────────────┬──────────────────┐│
  │  │ Protocol │ Port  │ Source                │ Reason           ││
  │  ├──────────┼───────┼───────────────────────┼──────────────────┤│
  │  │ TCP      │ 1433  │ sg-billing-app        │ Billing app      ││
  │  │          │       │ (application servers) │ queries billing  ││
  │  │          │       │                       │ database         ││
  │  ├──────────┼───────┼───────────────────────┼──────────────────┤│
  │  │ TCP      │ 1433  │ 10.0.100.0/24         │ Bastion host for ││
  │  │          │       │ (public subnet)       │ DBA SSMS access  ││
  │  ├──────────┼───────┼───────────────────────┼──────────────────┤│
  │  │ TCP      │ 1433  │ sg-ssis-servers       │ SSIS ETL jobs    ││
  │  │          │       │                       │ for billing data ││
  │  └──────────┴───────┴───────────────────────┴──────────────────┘│
  │                                                                 │
  │  OUTBOUND RULES:                                                │
  │  All traffic → 0.0.0.0/0 (default, allow all outbound)         │
  └─────────────────────────────────────────────────────────────────┘

  ┌─── SG: telephony-mysql-sg (CDR MySQL) ─────────────────────────┐
  │                                                                 │
  │  INBOUND RULES:                                                 │
  │  ┌──────────┬───────┬───────────────────────┬──────────────────┐│
  │  │ TCP      │ 3306  │ sg-cdr-processing     │ CDR ingestion    ││
  │  │          │       │ (CDR app servers)     │ from network     ││
  │  │          │       │                       │ switches         ││
  │  ├──────────┼───────┼───────────────────────┼──────────────────┤│
  │  │ TCP      │ 3306  │ 10.0.100.0/24         │ Bastion for DBA  ││
  │  │          │       │                       │ mysql CLI access  ││
  │  ├──────────┼───────┼───────────────────────┼──────────────────┤│
  │  │ TCP      │ 3306  │ sg-etl-servers        │ ETL for CDR      ││
  │  │          │       │                       │ aggregation      ││
  │  └──────────┴───────┴───────────────────────┴──────────────────┘│
  └─────────────────────────────────────────────────────────────────┘

  ┌─── SG: telephony-pg-sg (CRM PostgreSQL) ───────────────────────┐
  │                                                                 │
  │  INBOUND RULES:                                                 │
  │  ┌──────────┬───────┬───────────────────────┬──────────────────┐│
  │  │ TCP      │ 5432  │ sg-crm-app            │ CRM application  ││
  │  │          │       │                       │ servers          ││
  │  ├──────────┼───────┼───────────────────────┼──────────────────┤│
  │  │ TCP      │ 5432  │ 10.0.100.0/24         │ Bastion for DBA  ││
  │  │          │       │                       │ psql CLI access   ││
  │  ├──────────┼───────┼───────────────────────┼──────────────────┤│
  │  │ TCP      │ 5432  │ sg-analytics          │ Reporting and    ││
  │  │          │       │                       │ analytics tools  ││
  │  └──────────┴───────┴───────────────────────┴──────────────────┘│
  └─────────────────────────────────────────────────────────────────┘
```

### AWS CLI Commands with Full Explanations

```bash
# ===================================================
# Venkat: Create security groups for telephony1
# ===================================================

# MSSQL Security Group (Billing)
aws ec2 create-security-group \
  --group-name telephony-mssql-sg \
  --description "Telecom Billing MSSQL - port 1433" \
  --vpc-id vpc-tel1 \
  --region us-east-1
# → sg-mssql-tel1

# Allow billing application servers to reach MSSQL on port 1433
# Using security group reference — ANY server with sg-billing-app can connect
aws ec2 authorize-security-group-ingress \
  --group-id sg-mssql-tel1 \
  --protocol tcp \
  --port 1433 \
  --source-group sg-billing-app \
  --region us-east-1

# Allow bastion host subnet to reach MSSQL (for Venkat to use SSMS)
aws ec2 authorize-security-group-ingress \
  --group-id sg-mssql-tel1 \
  --protocol tcp \
  --port 1433 \
  --cidr 10.0.100.0/24 \
  --region us-east-1

# MySQL Security Group (CDR)
aws ec2 create-security-group \
  --group-name telephony-mysql-sg \
  --description "Telecom CDR MySQL - port 3306" \
  --vpc-id vpc-tel1 \
  --region us-east-1
# → sg-mysql-tel1

aws ec2 authorize-security-group-ingress \
  --group-id sg-mysql-tel1 \
  --protocol tcp \
  --port 3306 \
  --source-group sg-cdr-app \
  --region us-east-1

aws ec2 authorize-security-group-ingress \
  --group-id sg-mysql-tel1 \
  --protocol tcp \
  --port 3306 \
  --cidr 10.0.100.0/24 \
  --region us-east-1

# PostgreSQL Security Group (CRM)
aws ec2 create-security-group \
  --group-name telephony-pg-sg \
  --description "Telecom CRM PostgreSQL - port 5432" \
  --vpc-id vpc-tel1 \
  --region us-east-1
# → sg-pg-tel1

aws ec2 authorize-security-group-ingress \
  --group-id sg-pg-tel1 \
  --protocol tcp \
  --port 5432 \
  --source-group sg-crm-app \
  --region us-east-1

aws ec2 authorize-security-group-ingress \
  --group-id sg-pg-tel1 \
  --protocol tcp \
  --port 5432 \
  --cidr 10.0.100.0/24 \
  --region us-east-1

# VERIFY: View all rules in each security group
aws ec2 describe-security-groups \
  --group-ids sg-mssql-tel1 sg-mysql-tel1 sg-pg-tel1 \
  --query 'SecurityGroups[].{Name:GroupName,Rules:IpPermissions[].{Port:FromPort,Sources:IpRanges[].CidrIp,SG:UserIdGroupPairs[].GroupId}}' \
  --output table \
  --region us-east-1

# DANGER CHECK: Find any security group with 0.0.0.0/0 on database ports
# This would mean your database is open to the ENTIRE INTERNET — never allow this!
aws ec2 describe-security-groups \
  --region us-east-1 \
  --query 'SecurityGroups[].IpPermissions[?contains(IpRanges[].CidrIp, `0.0.0.0/0`) && (FromPort==`1433` || FromPort==`3306` || FromPort==`5432`)]' \
  --output table
```

> **CRITICAL SECURITY RULE:** Never open database ports (1433, 3306, 5432) to 0.0.0.0/0 (the entire internet). Always restrict to specific application security groups or private subnet CIDRs. A database exposed to the internet will be scanned and attacked within minutes.

---
---

# DAY 2 — Compute & Storage for Databases

---

## 2.1 EC2 for Oracle/SQL Databases

### When to Use EC2 vs RDS — Decision Framework

This is one of the most important decisions a telecom DBA makes. The choice between running databases on EC2 (self-managed) versus RDS (AWS-managed) affects your daily workload, automation capabilities, and how much infrastructure you need to manage.

**Use EC2 (Self-Managed) when you need:**

- Full operating system access (for installing agents, custom software)
- Non-standard database configurations (Oracle RAC, SQL Server FCI with shared disks)
- Specific OS-level tuning (kernel parameters, file system settings, huge pages)
- Third-party backup tools (Commvault, Veeam, Veritas)
- Database versions or features not supported by RDS
- Full control over patching schedules (no mandatory maintenance windows)

**Use RDS (AWS-Managed) when you want:**

- Reduced operational overhead (AWS handles backups, patching, HA)
- Quick provisioning (create a database in minutes, not hours)
- Built-in Multi-AZ with automatic failover
- Easy read replicas for scaling reads
- Automated backups with point-in-time recovery
- Storage auto-scaling

**For the Telecom Platform:**

The team has chosen RDS for all three database engines because:
- Venkat doesn't want to spend time on OS patching for 9 database servers across 3 regions
- Kishore values the automated Multi-AZ failover — manually managing failover for MySQL across 3 regions would be a full-time job
- Meena can focus on monitoring and daily operations rather than managing operating systems

### Choosing EC2 Instance Types — What the Letters Mean

AWS instance types follow a naming convention: `r6i.2xlarge`

```
  INSTANCE TYPE ANATOMY:  r6i.2xlarge

  r    = Instance Family (Memory-optimized)
  6    = Generation (6th generation)
  i    = Processor type (Intel)
  .    = separator
  2xlarge = Size (8 vCPU, 64 GB RAM)

  COMMON FAMILIES FOR DATABASES:
  ┌─────────┬───────────────────┬─────────────────────────────────┐
  │ Family  │ Optimized For     │ Telecom Use Case                │
  ├─────────┼───────────────────┼─────────────────────────────────┤
  │ r6i/r7i │ Memory            │ MSSQL billing (in-memory joins),│
  │         │ (high RAM/vCPU)   │ PostgreSQL CRM (large datasets) │
  ├─────────┼───────────────────┼─────────────────────────────────┤
  │ r6g     │ Memory + ARM      │ 20% cheaper, good for MySQL CDR │
  │         │ (Graviton)        │ read replicas, Aurora            │
  ├─────────┼───────────────────┼─────────────────────────────────┤
  │ m6i     │ General purpose   │ Dev/test databases, smaller DBs │
  │         │ (balanced)        │                                 │
  ├─────────┼───────────────────┼─────────────────────────────────┤
  │ i4i     │ Storage (NVMe)    │ High-IOPS CDR processing on EC2 │
  │         │ (local SSD)       │                                 │
  ├─────────┼───────────────────┼─────────────────────────────────┤
  │ x2idn   │ Extreme memory    │ Very large MSSQL billing with   │
  │         │ (up to 3 TB RAM)  │ in-memory OLTP                  │
  └─────────┴───────────────────┴─────────────────────────────────┘
```

### Instance Sizing for Telecom Workloads

| Telecom Database | Instance | vCPU | RAM | Why This Size |
|-----------------|----------|------|-----|---------------|
| MSSQL Billing (telephony1) | db.r6i.4xlarge | 16 | 128 GB | Billing runs complex joins across millions of invoices. MSSQL needs memory for buffer pool + query execution |
| MSSQL Billing (telephony3) | db.r6i.8xlarge | 32 | 256 GB | India has 120M subscribers — more billing transactions require more CPU and memory |
| MySQL CDR (all regions) | db.r6i.2xlarge | 8 | 64 GB | CDR writes are sequential and fast. InnoDB buffer pool at 48GB holds most active CDR data |
| PostgreSQL CRM (all regions) | db.r6i.2xlarge | 8 | 64 GB | CRM queries need memory for sorting, hashing. shared_buffers at 16GB for working set |
| Dev/Test (all engines) | db.m6i.xlarge | 4 | 16 GB | Sufficient for development and testing. Saves ~75% vs production size |

### Recommended EBS Volume Layout for EC2-hosted Databases

If the team ever needs to run databases on EC2 (for special features not available in RDS), here's the recommended disk layout. Separating data files, log files, and TempDB onto different volumes is critical because they have different I/O patterns:

```
  MSSQL ON EC2 — DISK LAYOUT (Venkat's design)

  Why separate volumes?
  • Data files (.mdf/.ndf): Random read/write pattern
  • Log files (.ldf): Sequential write pattern — NEVER mix with data
  • TempDB: Heavy random I/O during sorts, joins, temp tables
  • Backups: Sequential write — low IOPS needed

  Volume Layout:
  ┌─────────────────────────────────────────────────────────────┐
  │ Drive │ Size    │ Volume  │ IOPS  │ Contents                │
  ├───────┼─────────┼─────────┼───────┼─────────────────────────┤
  │ C:    │ 100 GB  │ gp3     │ 3000  │ OS + SQL Server binaries│
  │ D:    │ 500 GB  │ gp3     │ 6000  │ .mdf/.ndf data files    │
  │ E:    │ 200 GB  │ gp3     │ 3000  │ .ldf transaction logs   │
  │ F:    │ 100 GB  │ gp3     │ 3000  │ TempDB files            │
  │ G:    │ 500 GB  │ gp3     │ 3000  │ Backup files            │
  └─────────────────────────────────────────────────────────────┘

  MySQL ON EC2 — DISK LAYOUT (Kishore's design)

  ┌──────────────────────────────────────────────────────────────┐
  │ Mount Point         │ Size    │ IOPS   │ Contents            │
  ├─────────────────────┼─────────┼────────┼─────────────────────┤
  │ /                   │ 100 GB  │ 3000   │ OS + MySQL binaries │
  │ /var/lib/mysql      │ 1 TB    │ 10000  │ InnoDB data + CDRs  │
  │ /var/lib/mysql-logs │ 200 GB  │ 3000   │ Binary logs (repli) │
  │ /backup             │ 1 TB    │ 3000   │ mysqldump backups   │
  └──────────────────────────────────────────────────────────────┘

  PostgreSQL ON EC2 — DISK LAYOUT (Venkat/Kishore's design)

  ┌──────────────────────────────────────────────────────────────┐
  │ Mount Point          │ Size    │ IOPS  │ Contents            │
  ├──────────────────────┼─────────┼───────┼─────────────────────┤
  │ /                    │ 100 GB  │ 3000  │ OS + PG binaries    │
  │ /var/lib/postgresql  │ 500 GB  │ 6000  │ Data directory      │
  │ /var/lib/pg-wal      │ 100 GB  │ 3000  │ WAL (Write-Ahead    │
  │                      │         │       │ Logs) — MUST be on  │
  │                      │         │       │ separate volume!    │
  │ /backup              │ 500 GB  │ 3000  │ pg_basebackup files │
  └──────────────────────────────────────────────────────────────┘
```

---

## 2.2 EBS Volume Types & Performance

### What is EBS?

EBS (Elastic Block Store) provides persistent block storage volumes for EC2 instances and is also the underlying storage for RDS instances. Think of EBS volumes as virtual hard drives that you can attach to your instances.

Unlike instance store (which is temporary and lost when the instance stops), EBS volumes persist independently. If you stop an EC2 instance and start it again, your EBS data is still there. EBS volumes are automatically replicated within their AZ for hardware fault tolerance.

### Understanding IOPS and Throughput

**IOPS (Input/Output Operations Per Second):** This is the number of read/write operations your volume can handle per second. Think of it as "how many times per second the disk can respond to a request." A billing query that reads 500 random blocks needs 500 IOPS. High IOPS is critical for OLTP workloads (billing, CDR).

**Throughput (MB/s):** This is the total amount of data that can be read/written per second. Think of it as the "bandwidth" of the disk. A backup operation that reads data sequentially cares more about throughput than IOPS. Reading 500 MB/s means you can scan a 50GB table in about 100 seconds.

**Latency:** The time between sending a request and getting a response. Database workloads need sub-millisecond latency. gp3 and io2 both deliver sub-millisecond latency.

### EBS Volume Types — Detailed Comparison

```
  EBS VOLUME TYPES FOR TELECOM DATABASES

  ┌─────────┬──────────┬─────────────┬───────────────┬──────────────────────────┐
  │ Type    │ Max IOPS │ Max Thruput │ Cost Model    │ Telecom Use Case         │
  ├─────────┼──────────┼─────────────┼───────────────┼──────────────────────────┤
  │ gp3     │ 16,000   │ 1,000 MB/s  │ Fixed $/GB +  │ Most databases.          │
  │         │          │             │ $/IOPS above  │ MSSQL billing data,      │
  │         │          │             │ 3,000 IOPS    │ MySQL CDR data,          │
  │         │          │             │               │ PostgreSQL CRM data      │
  ├─────────┼──────────┼─────────────┼───────────────┼──────────────────────────┤
  │ gp2     │ 16,000   │ 250 MB/s    │ $/GB (IOPS    │ LEGACY. Don't use for    │
  │ (old)   │          │             │ tied to size: │ new deployments.          │
  │         │          │             │ 3 IOPS/GB)    │ Migrate to gp3.          │
  ├─────────┼──────────┼─────────────┼───────────────┼──────────────────────────┤
  │ io2     │ 64,000   │ 1,000 MB/s  │ $/GB + $/IOPS │ Peak billing periods     │
  │         │          │             │ (guaranteed)  │ (month-end processing),  │
  │         │          │             │               │ if > 16K IOPS needed     │
  ├─────────┼──────────┼─────────────┼───────────────┼──────────────────────────┤
  │ io2     │ 256,000  │ 4,000 MB/s  │ Premium $/IOPS│ Only for extreme cases.  │
  │ Block   │          │             │               │ Rarely needed for        │
  │ Express │          │             │               │ telecom databases.       │
  ├─────────┼──────────┼─────────────┼───────────────┼──────────────────────────┤
  │ st1     │ 500      │ 500 MB/s    │ Cheap $/GB    │ CDR archives (old call   │
  │ (HDD)   │          │             │               │ records kept for 7 years │
  │         │          │             │               │ compliance). Cold data.  │
  └─────────┴──────────┴─────────────┴───────────────┴──────────────────────────┘
```

### Why gp3 is the Default Choice

gp3 is superior to gp2 in almost every way and is the recommended volume type for most telecom databases. Here's why:

**gp3 Advantages:**
- **Independent IOPS and throughput:** You can set IOPS (up to 16,000) and throughput (up to 1,000 MB/s) independently of volume size. A 100 GB volume can have 16,000 IOPS.
- **Free baseline:** Every gp3 volume gets 3,000 IOPS and 125 MB/s for free.
- **Cheaper:** gp3 is about 20% cheaper per GB than gp2.

**gp2 Disadvantage:**
- IOPS is tied to volume size: 3 IOPS per GB. So a 100 GB volume only gets 300 IOPS. To get 3,000 IOPS, you'd need a 1,000 GB volume — paying for storage you don't need.

```
  gp3 vs gp2 — COST COMPARISON FOR TELECOM CDR DATABASE

  Requirement: 500 GB storage, 6,000 IOPS

  gp2:
  • Need 2,000 GB to get 6,000 IOPS (3 IOPS × 2,000 GB)
  • Paying for 1,500 GB you don't need!
  • Cost: 2,000 GB × $0.10/GB = $200/month

  gp3:
  • 500 GB storage + 6,000 IOPS configured separately
  • Cost: 500 GB × $0.08/GB = $40 + (3,000 extra IOPS × $0.005) = $15
  • Total: $55/month

  SAVINGS: $145/month per volume = $1,740/year
  Across 9 databases × 3 regions = massive savings
```

### AWS CLI Commands with Explanations

```bash
# Kishore: Create high-IOPS volume for MySQL CDR in telephony3
# CDR ingestion writes thousands of call records per second
# 10,000 IOPS handles peak call volumes during business hours
aws ec2 create-volume \
  --volume-type gp3 \
  --size 1000 \
  --iops 10000 \
  --throughput 500 \
  --availability-zone ap-south-1a \
  --encrypted \
  --tag-specifications 'ResourceType=volume,Tags=[
    {Key=Name,Value=tel3-mysql-cdr-data},
    {Key=Engine,Value=MySQL},
    {Key=Application,Value=CDR},
    {Key=ManagedBy,Value=Kishore}]' \
  --region ap-south-1

# Venkat: Create volume for MSSQL billing transaction logs
# Log files need moderate IOPS but LOW latency
aws ec2 create-volume \
  --volume-type gp3 \
  --size 200 \
  --iops 3000 \
  --throughput 250 \
  --availability-zone us-east-1a \
  --encrypted \
  --tag-specifications 'ResourceType=volume,Tags=[
    {Key=Name,Value=tel1-mssql-billing-logs},
    {Key=Engine,Value=MSSQL},
    {Key=ManagedBy,Value=Venkat}]' \
  --region us-east-1

# Modify volume — increase IOPS during peak billing season (month-end)
# NO DOWNTIME REQUIRED — this is a live change!
aws ec2 modify-volume \
  --volume-id vol-xxx \
  --iops 12000 \
  --throughput 700 \
  --region us-east-1

# Check modification progress
aws ec2 describe-volumes-modifications \
  --volume-ids vol-xxx \
  --query 'VolumesModifications[].{ID:VolumeId,Progress:Progress,OriginalIOPS:OriginalIops,TargetIOPS:TargetIops}' \
  --region us-east-1

# Meena: List all volumes across telephony2 with details
aws ec2 describe-volumes \
  --region eu-west-3 \
  --query 'Volumes[].{ID:VolumeId,Name:Tags[?Key==`Name`].Value|[0],Size:Size,Type:VolumeType,IOPS:Iops,Throughput:Throughput,State:State,AZ:AvailabilityZone}' \
  --output table
```

---

## 2.3 EBS Snapshots & Restore

### How EBS Snapshots Work — The Details

An EBS snapshot is a point-in-time backup of your EBS volume, stored in Amazon S3 (managed by AWS — you don't see it in your S3 buckets). Here's what makes snapshots special:

**Incremental Backups:** The first snapshot copies the entire volume. Every subsequent snapshot only copies the blocks that have changed since the last snapshot. This means:
- First snapshot of a 500 GB volume: copies all 500 GB (takes longer)
- Second snapshot after 10 GB changed: copies only 10 GB (much faster)
- Cost: you only pay for the changed blocks

**Crash-consistent:** Snapshots capture what's on disk at that moment. For databases, this means you should either:
- Quiesce the database before snapshot (flush to disk)
- Or rely on the database's crash recovery (like MSSQL can recover from crash-consistent backups using transaction log)

**Cross-Region Copy:** You can copy snapshots to another AWS region. This is fundamental for disaster recovery — if the entire `ap-south-1` region goes down, Kishore can restore the telephony3 databases from snapshots copied to `us-east-1`.

```
  SNAPSHOT LIFECYCLE FOR TELECOM:

  telephony3 (Mumbai) — Kishore

  ┌──────────────┐    1st Snapshot     ┌─────────────────┐
  │  MySQL CDR   │ ──────────────────► │  S3 (ap-south-1)│
  │  Volume      │    (full: 1 TB)     │  Snapshot #1    │
  │  1 TB        │                     │  Cost: ~1 TB    │
  └──────────────┘                     └─────────────────┘
        │                                     │
        │  24 hours later,                    │
        │  50 GB new CDR data                 │
        │                                     │
        ▼                                     ▼
  ┌──────────────┐    2nd Snapshot     ┌─────────────────┐
  │  MySQL CDR   │ ──────────────────► │  S3 (ap-south-1)│
  │  Volume      │    (incremental:    │  Snapshot #2    │
  │  1 TB        │     only 50 GB)     │  Cost: ~50 GB   │
  └──────────────┘                     └─────────────────┘
                                              │
                                   Copy to us-east-1 for DR
                                              │
                                              ▼
                                       ┌─────────────────┐
                                       │  S3 (us-east-1) │
                                       │  DR Copy        │
                                       │  Can restore    │
                                       │  new volume     │
                                       │  in us-east-1   │
                                       └─────────────────┘
```

### AWS CLI Commands — Complete Snapshot Operations

```bash
# ===================================================
# Venkat: Pre-patching snapshot of MSSQL billing volume
# Always take a snapshot before any major change!
# ===================================================
aws ec2 create-snapshot \
  --volume-id vol-mssql-billing \
  --description "telephony1 MSSQL billing - pre CU patch Feb 2025" \
  --tag-specifications 'ResourceType=snapshot,Tags=[
    {Key=Name,Value=tel1-mssql-billing-prepatch-20250219},
    {Key=CreatedBy,Value=Venkat},
    {Key=Purpose,Value=pre-patch-safety},
    {Key=Environment,Value=production}]' \
  --region us-east-1

# List all snapshots for a volume (sorted by date)
aws ec2 describe-snapshots \
  --filters "Name=volume-id,Values=vol-mssql-billing" \
  --query 'sort_by(Snapshots, &StartTime)[].{ID:SnapshotId,Date:StartTime,Size:VolumeSize,State:State,Desc:Description}' \
  --output table \
  --region us-east-1

# ===================================================
# Kishore: Copy MySQL CDR snapshot to telephony1 for DR
# If all of ap-south-1 goes down, we can restore in us-east-1
# ===================================================
aws ec2 copy-snapshot \
  --source-region ap-south-1 \
  --source-snapshot-id snap-mysql-cdr-xxx \
  --destination-region us-east-1 \
  --description "DR copy: telephony3 MySQL CDR to telephony1" \
  --encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/xxx

# ===================================================
# Restore: Create new volume from snapshot
# Scenario: MSSQL billing volume corrupted, restore from snapshot
# ===================================================
aws ec2 create-volume \
  --snapshot-id snap-mssql-billing-xxx \
  --volume-type gp3 \
  --iops 6000 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=tel1-mssql-billing-RESTORED}]' \
  --region us-east-1

# ===================================================
# Meena: Cleanup old snapshots (keep last 30 days)
# ===================================================
CUTOFF=$(date -u -d '30 days ago' +%Y-%m-%dT%H:%M:%S)
aws ec2 describe-snapshots \
  --owner-ids self \
  --query "Snapshots[?StartTime<='${CUTOFF}' && Tags[?Key=='Purpose' && Value=='daily-backup']].SnapshotId" \
  --output text --region eu-west-3 | tr '\t' '\n' | while read snap; do
    echo "Deleting old snapshot: $snap"
    aws ec2 delete-snapshot --snapshot-id "$snap" --region eu-west-3
done
```

---

## 2.4 S3 for DB Backups

### Why S3 for Database Backups?

Amazon S3 (Simple Storage Service) is an object storage service that provides 99.999999999% (11 nines) durability. This means if you store 10 million backup files, you can expect to lose one file every 10,000 years. For telecom compliance, this durability is essential.

S3 is ideal for database backups because:

- **Unlimited storage:** No capacity planning needed. Store terabytes or petabytes.
- **Cost-effective:** Starts at $0.023/GB/month for Standard, goes down to $0.00099/GB/month for Deep Archive.
- **Lifecycle policies:** Automatically move old backups to cheaper storage tiers.
- **Cross-region replication:** Automatically replicate backups to another region for DR.
- **Versioning:** Protect against accidental deletion. Even if someone deletes a backup file, the previous version is preserved.
- **Encryption:** Server-side encryption with KMS keys. Backups are encrypted at rest.

### S3 Storage Classes — Detailed for Telecom Backup Strategy

```
  TELECOM BACKUP TIERING WITH S3 LIFECYCLE:

  Day 0-30: S3 Standard ($0.023/GB)
  ├── Recent backups — need immediate access
  ├── Example: Last 30 days of MSSQL billing full backups
  ├── Restore time: milliseconds
  └── Used by: Venkat for quick restores after failed deployments

  Day 31-90: S3 Standard-IA ($0.0125/GB) — 46% cheaper
  ├── Weekly backups — accessed rarely
  ├── Example: Weekly billing backups from 1-3 months ago
  ├── Restore time: milliseconds (same as Standard)
  ├── Retrieval fee: $0.01/GB when accessed
  └── Used by: Venkat/Kishore for monthly compliance checks

  Day 91-365: S3 Glacier Instant Retrieval ($0.004/GB) — 83% cheaper
  ├── Monthly backups — accessed very rarely
  ├── Example: Monthly CDR archives for regulatory compliance
  ├── Restore time: milliseconds (instant!)
  ├── Retrieval fee: $0.03/GB when accessed
  └── Used by: Compliance team for quarterly audits

  Day 366-2555: S3 Glacier Deep Archive ($0.00099/GB) — 96% cheaper
  ├── Yearly backups — kept for 7 years (telecom regulation)
  ├── Example: Annual CDR archives, billing records
  ├── Restore time: 12-48 hours
  ├── Used by: Legal team for dispute resolution
  └── Telecom regulations require 7-year retention
```

### AWS CLI Commands — Complete S3 Backup Setup

```bash
# ===================================================
# Venkat: Create and configure S3 backup bucket
# ===================================================

# Create bucket
aws s3 mb s3://telecom-db-backups-prod --region us-east-1

# Enable versioning — protects against accidental deletion
aws s3api put-bucket-versioning \
  --bucket telecom-db-backups-prod \
  --versioning-configuration Status=Enabled

# Enable KMS encryption — all backups encrypted at rest
aws s3api put-bucket-encryption \
  --bucket telecom-db-backups-prod \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/xxx"
      }
    }]
  }'

# Block public access — databases backups must NEVER be public
aws s3api put-public-access-block \
  --bucket telecom-db-backups-prod \
  --public-access-block-configuration '{
    "BlockPublicAcls": true,
    "IgnorePublicAcls": true,
    "BlockPublicPolicy": true,
    "RestrictPublicBuckets": true
  }'

# ===================================================
# Upload backups — each team member to their region folder
# ===================================================

# Venkat: MSSQL billing full backup
aws s3 cp /backups/billing_full_20250219.bak \
  s3://telecom-db-backups-prod/telephony1/mssql/billing/full/

# Venkat: MSSQL billing transaction log backup
aws s3 cp /backups/billing_tlog_20250219_1400.trn \
  s3://telecom-db-backups-prod/telephony1/mssql/billing/tlog/

# Kishore: MySQL CDR compressed dump
aws s3 cp /backup/cdr_dump_20250219.sql.gz \
  s3://telecom-db-backups-prod/telephony3/mysql/cdr/full/

# Kishore: MySQL binary logs (for point-in-time recovery)
aws s3 sync /var/lib/mysql-logs/ \
  s3://telecom-db-backups-prod/telephony3/mysql/cdr/binlog/

# Meena: PostgreSQL CRM base backup
aws s3 cp /backup/crm_basebackup_20250219.tar.gz \
  s3://telecom-db-backups-prod/telephony2/postgresql/crm/full/

# Meena: PostgreSQL WAL archives
aws s3 sync /var/lib/pg-wal-archive/ \
  s3://telecom-db-backups-prod/telephony2/postgresql/crm/wal/

# ===================================================
# Lifecycle policy — auto-tier old backups to save cost
# ===================================================
aws s3api put-bucket-lifecycle-configuration \
  --bucket telecom-db-backups-prod \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "Telecom-Backup-Lifecycle",
      "Status": "Enabled",
      "Filter": {"Prefix": ""},
      "Transitions": [
        {"Days": 30,  "StorageClass": "STANDARD_IA"},
        {"Days": 90,  "StorageClass": "GLACIER_IR"},
        {"Days": 365, "StorageClass": "DEEP_ARCHIVE"}
      ],
      "Expiration": {"Days": 2555}
    }]
  }'
# 2555 days = ~7 years (telecom compliance requirement)

# ===================================================
# Meena: Daily check — verify backup sizes and latest uploads
# ===================================================
echo "=== telephony1 MSSQL Billing Backups ==="
aws s3 ls s3://telecom-db-backups-prod/telephony1/mssql/billing/full/ \
  --human-readable --summarize

echo "=== telephony3 MySQL CDR Backups ==="
aws s3 ls s3://telecom-db-backups-prod/telephony3/mysql/cdr/full/ \
  --human-readable --summarize

echo "=== telephony2 PostgreSQL CRM Backups ==="
aws s3 ls s3://telecom-db-backups-prod/telephony2/postgresql/crm/full/ \
  --human-readable --summarize

# Download backup for restore
aws s3 cp s3://telecom-db-backups-prod/telephony1/mssql/billing/full/billing_full_20250219.bak \
  /restore/
```

---

## 2.5 AWS Backup Overview

### What is AWS Backup?

AWS Backup is a centralized backup service that lets you automate and manage backups across multiple AWS services from a single place. Instead of setting up backup scripts for each individual RDS instance, EBS volume, and EC2 instance separately, you create one backup plan that covers everything.

For the telecom DBA team, this means Venkat can create a single backup plan that automatically backs up all 9 RDS databases (3 engines × 3 regions), all EBS volumes, and all EC2 instances according to a consistent schedule.

### Key Components

**Backup Plan:** The schedule and retention rules. "Take daily backups at 2 AM, keep for 35 days. Take weekly backups on Sunday, keep for 1 year."

**Backup Vault:** An encrypted container where backups are stored. Think of it as a secure safe. You can create separate vaults for different environments (prod vault, dev vault) with different access controls.

**Backup Selection:** Which resources to back up. You can select resources by ARN (specific instances), by tag (everything tagged Environment=production), or by resource type.

```
  AWS BACKUP FOR TELECOM:

  ┌─────────────────────────────────────────────────────────────┐
  │                    BACKUP PLAN                               │
  │             "Telecom-All-DB-Backup"                          │
  │                                                             │
  │  Rule 1: Daily at 2 AM UTC                                  │
  │  ├── Retention: 35 days                                     │
  │  └── Covers: All production RDS instances                   │
  │                                                             │
  │  Rule 2: Weekly (Sunday 3 AM UTC)                            │
  │  ├── Retention: 365 days                                    │
  │  ├── Move to cold storage after 30 days                     │
  │  └── Covers: All production RDS instances                   │
  │                                                             │
  │  Rule 3: Monthly (1st of month, 4 AM UTC)                   │
  │  ├── Retention: 2555 days (7 years for compliance)          │
  │  ├── Move to cold storage after 30 days                     │
  │  └── Covers: All production RDS instances                   │
  └──────────────────────────────┬──────────────────────────────┘
                                 │
                   Protects these resources:
                                 │
          ┌──────────────────────┼──────────────────────┐
          │                      │                      │
          ▼                      ▼                      ▼
  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐
  │ telephony1    │  │ telephony2    │  │ telephony3    │
  │ MSSQL Billing │  │ MSSQL Billing │  │ MSSQL Billing │
  │ MySQL CDR     │  │ MySQL CDR     │  │ MySQL CDR     │
  │ PostgreSQL CRM│  │ PostgreSQL CRM│  │ PostgreSQL CRM│
  └───────────────┘  └───────────────┘  └───────────────┘
```

### AWS CLI Commands

```bash
# Venkat: Create encrypted backup vault
aws backup create-backup-vault \
  --backup-vault-name telecom-prod-vault \
  --encryption-key-arn arn:aws:kms:us-east-1:123456789012:key/xxx

# Venkat: Create comprehensive backup plan
aws backup create-backup-plan --backup-plan '{
  "BackupPlanName": "Telecom-All-DB-Backup",
  "Rules": [
    {
      "RuleName": "Daily-2AM",
      "TargetBackupVaultName": "telecom-prod-vault",
      "ScheduleExpression": "cron(0 2 * * ? *)",
      "StartWindowMinutes": 60,
      "CompletionWindowMinutes": 180,
      "Lifecycle": {"DeleteAfterDays": 35}
    },
    {
      "RuleName": "Weekly-Sunday-3AM",
      "TargetBackupVaultName": "telecom-prod-vault",
      "ScheduleExpression": "cron(0 3 ? * SUN *)",
      "Lifecycle": {"MoveToColdStorageAfterDays": 30, "DeleteAfterDays": 365}
    },
    {
      "RuleName": "Monthly-1st-4AM",
      "TargetBackupVaultName": "telecom-prod-vault",
      "ScheduleExpression": "cron(0 4 1 * ? *)",
      "Lifecycle": {"MoveToColdStorageAfterDays": 30, "DeleteAfterDays": 2555}
    }
  ]
}'

# Assign ALL production telecom databases using tags
aws backup create-backup-selection \
  --backup-plan-id plan-xxx \
  --backup-selection '{
    "SelectionName": "All-Telecom-Prod-Databases",
    "IamRoleArn": "arn:aws:iam::123456789012:role/AWSBackupDefaultServiceRole",
    "Conditions": {
      "StringEquals": [
        {"ConditionKey": "aws:ResourceTag/Environment", "ConditionValue": "production"}
      ]
    },
    "Resources": []
  }'

# Meena: Check backup job status (daily verification)
aws backup list-backup-jobs \
  --by-state COMPLETED \
  --by-created-after $(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%S) \
  --query 'BackupJobs[].{Resource:ResourceArn,Status:State,Created:CreationDate,Size:BackupSizeInBytes}' \
  --output table
```

---
---

# DAY 3 — AWS Managed Database Services

---

## 3.1 Amazon RDS Architecture

### What is Amazon RDS?

Amazon RDS (Relational Database Service) is a managed database service that handles the heavy lifting of database administration. When you create an RDS instance, AWS automatically provisions a virtual machine, installs the database engine, configures storage, sets up automated backups, and manages the operating system — all behind the scenes.

You interact with RDS through a DNS endpoint (like `prod-telephony1-mssql-billing.crwxyz123.us-east-1.rds.amazonaws.com`). You connect to this endpoint just like you would connect to any on-premises database — using SSMS for SQL Server, mysql CLI for MySQL, or psql for PostgreSQL.

**What you DON'T get with RDS:**
- No SSH access to the underlying server
- No root/administrator OS access
- Cannot install software on the OS
- Cannot access the file system directly
- Limited to RDS-supported engine versions and configurations

**What you DO get:**
- Automated daily backups with point-in-time recovery
- One-click Multi-AZ deployment with automatic failover
- Easy read replica creation for scaling reads
- Push-button storage scaling (no downtime)
- Automated engine patching in your chosen maintenance window
- Integrated monitoring with CloudWatch and Performance Insights
- Encryption at rest and in transit

### RDS Supported Engines for Telecom

```
  RDS ENGINES USED BY TELECOM TEAM:

  ┌──────────────────────────────────────────────────────────────────┐
  │ Engine              │ RDS Versions     │ Telecom Use             │
  ├─────────────────────┼──────────────────┼─────────────────────────┤
  │ SQL Server SE 2022  │ 16.00.x          │ Billing system          │
  │                     │                  │ (transactions, invoices)│
  │                     │                  │ Managed by Venkat       │
  ├─────────────────────┼──────────────────┼─────────────────────────┤
  │ MySQL 8.0           │ 8.0.36+          │ CDR (Call Detail Records│
  │                     │                  │ & network logs)         │
  │                     │                  │ Managed by Kishore      │
  ├─────────────────────┼──────────────────┼─────────────────────────┤
  │ PostgreSQL 16       │ 16.4+            │ CRM (Customer Relations │
  │                     │                  │ & analytics)            │
  │                     │                  │ Managed by Meena        │
  └──────────────────────────────────────────────────────────────────┘

  Licensing:
  • SQL Server: "license-included" — AWS includes the SQL Server license
    in the hourly price. No need to bring your own MSSQL license.
  • MySQL: Open source — no license cost.
  • PostgreSQL: Open source — no license cost.
```

### AWS CLI — Create All 3 Telecom Databases

```bash
# ===================================================================
# Venkat: Create MSSQL RDS for telephony1 Billing
# SQL Server Standard Edition 2022, Multi-AZ, encrypted
# ===================================================================
aws rds create-db-instance \
  --db-instance-identifier prod-telephony1-mssql-billing \
  --db-instance-class db.r6i.4xlarge \
  --engine sqlserver-se \
  --engine-version 16.00 \
  --master-username billingadmin \
  --master-user-password 'Tel3com$Billing!2025' \
  --allocated-storage 500 \
  --storage-type gp3 \
  --iops 6000 \
  --storage-throughput 400 \
  --multi-az \
  --db-subnet-group-name telephony1-db-subnets \
  --vpc-security-group-ids sg-mssql-tel1 \
  --backup-retention-period 14 \
  --preferred-backup-window "02:00-03:00" \
  --preferred-maintenance-window "sun:04:00-sun:05:00" \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/xxx \
  --license-model license-included \
  --no-publicly-accessible \
  --enable-performance-insights \
  --performance-insights-retention-period 731 \
  --tags Key=Application,Value=Billing \
         Key=Region,Value=telephony1 \
         Key=Engine,Value=MSSQL \
         Key=ManagedBy,Value=Venkat \
         Key=Environment,Value=production \
  --region us-east-1

# ===================================================================
# Kishore: Create MySQL RDS for telephony3 CDR
# MySQL 8.0, Multi-AZ, high IOPS for CDR write workload
# ===================================================================
aws rds create-db-instance \
  --db-instance-identifier prod-telephony3-mysql-cdr \
  --db-instance-class db.r6i.2xlarge \
  --engine mysql \
  --engine-version 8.0.36 \
  --master-username cdradmin \
  --master-user-password 'Tel3com$CDR!2025' \
  --allocated-storage 1000 \
  --storage-type gp3 \
  --iops 10000 \
  --storage-throughput 500 \
  --multi-az \
  --db-subnet-group-name telephony3-db-subnets \
  --vpc-security-group-ids sg-mysql-tel3 \
  --backup-retention-period 14 \
  --preferred-backup-window "02:00-03:00" \
  --preferred-maintenance-window "sun:04:00-sun:05:00" \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:ap-south-1:123456789012:key/xxx \
  --no-publicly-accessible \
  --enable-performance-insights \
  --performance-insights-retention-period 731 \
  --tags Key=Application,Value=CDR \
         Key=Region,Value=telephony3 \
         Key=Engine,Value=MySQL \
         Key=ManagedBy,Value=Kishore \
         Key=Environment,Value=production \
  --region ap-south-1

# ===================================================================
# Meena: Create PostgreSQL RDS for telephony2 CRM
# PostgreSQL 16, Multi-AZ (guided by Venkat)
# ===================================================================
aws rds create-db-instance \
  --db-instance-identifier prod-telephony2-pg-crm \
  --db-instance-class db.r6i.2xlarge \
  --engine postgres \
  --engine-version 16.4 \
  --master-username crmadmin \
  --master-user-password 'Tel3com$CRM!2025' \
  --allocated-storage 500 \
  --storage-type gp3 \
  --iops 6000 \
  --multi-az \
  --db-subnet-group-name telephony2-db-subnets \
  --vpc-security-group-ids sg-pg-tel2 \
  --backup-retention-period 14 \
  --preferred-backup-window "02:00-03:00" \
  --preferred-maintenance-window "sun:04:00-sun:05:00" \
  --storage-encrypted \
  --no-publicly-accessible \
  --enable-performance-insights \
  --performance-insights-retention-period 731 \
  --tags Key=Application,Value=CRM \
         Key=Region,Value=telephony2 \
         Key=Engine,Value=PostgreSQL \
         Key=ManagedBy,Value=Meena \
         Key=Environment,Value=production \
  --region eu-west-3

# ===================================================================
# Meena: Verify all instances are running across all regions
# ===================================================================
for region in us-east-1 eu-west-3 ap-south-1; do
  echo "=== $region ==="
  aws rds describe-db-instances --region $region \
    --query 'DBInstances[].{Name:DBInstanceIdentifier,Engine:Engine,Version:EngineVersion,Class:DBInstanceClass,Status:DBInstanceStatus,MultiAZ:MultiAZ,Storage:AllocatedStorage,Encrypted:StorageEncrypted}' \
    --output table
done

# Get connection endpoint for each database
aws rds describe-db-instances \
  --db-instance-identifier prod-telephony1-mssql-billing \
  --query 'DBInstances[0].Endpoint.{Address:Address,Port:Port}' \
  --region us-east-1
# OUTPUT: prod-telephony1-mssql-billing.crwxyz.us-east-1.rds.amazonaws.com:1433
```

---

## 3.2 RDS Backups, PITR & Snapshots — Detailed

### Three Types of RDS Backups

**1. Automated Backups** — AWS takes a full snapshot daily during your backup window and saves transaction logs every 5 minutes. This enables point-in-time recovery. Retention: 1-35 days (set at instance creation). These are automatically deleted when the RDS instance is deleted (unless you choose to keep a final snapshot).

**2. Manual Snapshots** — You trigger these on demand (before patching, before schema changes, etc.). They persist forever until you explicitly delete them. They survive instance deletion. You can share them with other AWS accounts or copy them to other regions.

**3. Point-In-Time Recovery (PITR)** — This combines the daily snapshot with transaction log replay to restore your database to any specific second within the retention period. It creates a NEW RDS instance — it does not restore in-place.

```
  HOW PITR WORKS:

  Timeline:
  2 AM    ──── Daily Snapshot Taken ────────────────────────────
  2:05 AM ──── Transaction logs saved ──────────────────────────
  2:10 AM ──── Transaction logs saved ──────────────────────────
  ...         (logs saved every 5 minutes)
  2:30 PM ──── BAD SQL EXECUTED: DELETE FROM billing_invoices;
  2:31 PM ──── Kishore: "Meena, what happened to the invoices?!"
  2:32 PM ──── Venkat: "Don't panic. We'll PITR to 2:29:59 PM"

  PITR Process:
  1. AWS takes the 2 AM daily snapshot
  2. AWS replays all transaction logs from 2 AM to 2:29:59 PM
  3. Result: A NEW RDS instance with data as of 2:29:59 PM
  4. The bad DELETE at 2:30 PM is NOT included
  5. Venkat renames the new instance and updates the application endpoint
```

### AWS CLI Commands — All Backup Operations

```bash
# ===================================================================
# Venkat: Create manual snapshot before MSSQL billing engine upgrade
# ALWAYS snapshot before major changes!
# ===================================================================
aws rds create-db-snapshot \
  --db-instance-identifier prod-telephony1-mssql-billing \
  --db-snapshot-identifier tel1-mssql-billing-pre-upgrade-20250219 \
  --region us-east-1

# Wait for snapshot to complete
aws rds wait db-snapshot-available \
  --db-snapshot-identifier tel1-mssql-billing-pre-upgrade-20250219 \
  --region us-east-1

# ===================================================================
# Kishore: PITR for MySQL CDR — someone deleted CDR records!
# Restore to 2 minutes before the incident
# ===================================================================

# First, check the latest restorable time
aws rds describe-db-instances \
  --db-instance-identifier prod-telephony3-mysql-cdr \
  --query 'DBInstances[0].{Latest:LatestRestorableTime,Earliest:EarliestRestorableTime}' \
  --region ap-south-1

# Restore to specific point in time
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier prod-telephony3-mysql-cdr \
  --target-db-instance-identifier prod-telephony3-mysql-cdr-pitr-restored \
  --restore-time "2025-02-19T14:28:00Z" \
  --db-instance-class db.r6i.2xlarge \
  --db-subnet-group-name telephony3-db-subnets \
  --vpc-security-group-ids sg-mysql-tel3 \
  --no-publicly-accessible \
  --region ap-south-1

# ===================================================================
# Meena: Restore PostgreSQL CRM from manual snapshot
# ===================================================================
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier prod-telephony2-pg-crm-restored \
  --db-snapshot-identifier tel2-pg-crm-pre-migration-snap \
  --db-instance-class db.r6i.2xlarge \
  --db-subnet-group-name telephony2-db-subnets \
  --vpc-security-group-ids sg-pg-tel2 \
  --no-publicly-accessible \
  --region eu-west-3

# ===================================================================
# Kishore: Copy MySQL CDR snapshot cross-region for DR
# If ap-south-1 goes down, we can restore in us-east-1
# ===================================================================
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier arn:aws:rds:ap-south-1:123456789012:snapshot:tel3-mysql-cdr-daily \
  --target-db-snapshot-identifier tel3-mysql-cdr-dr-copy-us-east-1 \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/xxx \
  --region us-east-1

# ===================================================================
# Meena: List all snapshots with sizes (daily check)
# ===================================================================
for region in us-east-1 eu-west-3 ap-south-1; do
  echo "=== $region Snapshots ==="
  aws rds describe-db-snapshots --region $region \
    --query 'sort_by(DBSnapshots, &SnapshotCreateTime)[-5:].{ID:DBSnapshotIdentifier,DB:DBInstanceIdentifier,Type:SnapshotType,Created:SnapshotCreateTime,Status:Status,SizeGB:AllocatedStorage}' \
    --output table
done
```

---

## 3.3 RDS Patching & Parameter Groups — Deep Dive

### RDS Maintenance Windows — How Patching Works

RDS patching happens automatically during your configured maintenance window. For Multi-AZ deployments, the process is designed to minimize downtime:

```
  MULTI-AZ PATCHING PROCESS (applies to MSSQL, MySQL, PostgreSQL):

  Step 1: AWS patches the STANDBY instance
  ┌──────────────┐         ┌──────────────┐
  │  PRIMARY     │         │  STANDBY     │
  │  (running)   │         │  (patching)  │
  │  No impact   │         │  ████████░░  │
  └──────────────┘         └──────────────┘
  Applications continue normally.

  Step 2: AWS promotes STANDBY to PRIMARY (failover)
  ┌──────────────┐         ┌──────────────┐
  │  OLD PRIMARY │         │  NEW PRIMARY │
  │  (demoted)   │  ◄────  │  (promoted)  │
  │              │ ~60 sec │  Running!    │
  └──────────────┘         └──────────────┘
  Brief connection drop (~60 seconds). Applications reconnect.

  Step 3: AWS patches the OLD PRIMARY (now standby)
  ┌──────────────┐         ┌──────────────┐
  │  STANDBY     │         │  PRIMARY     │
  │  (patching)  │         │  (running)   │
  │  ████████░░  │         │  No impact   │
  └──────────────┘         └──────────────┘
  Applications continue normally.

  Total downtime: ~60 seconds during Step 2 failover
```

### Parameter Groups — Database Configuration on RDS

Since you don't have OS access on RDS, you configure database parameters through Parameter Groups. A parameter group is a named collection of database engine settings.

**Important Rule:** The Default parameter group is read-only. You MUST create a custom parameter group if you want to change any settings.

**Static vs Dynamic Parameters:**
- **Dynamic:** Takes effect immediately, no reboot needed (e.g., `slow_query_log` in MySQL)
- **Static:** Requires a reboot to take effect (e.g., `shared_buffers` in PostgreSQL, `innodb_buffer_pool_size` in MySQL)

### Parameter Groups for Each Engine

```bash
# ===================================================================
# Venkat: MSSQL Parameter Group for Billing
# ===================================================================
aws rds create-db-parameter-group \
  --db-parameter-group-name telecom-mssql-billing-params \
  --db-parameter-group-family sqlserver-se-16.0 \
  --description "Telecom Billing MSSQL tuning - Venkat"

aws rds modify-db-parameter-group \
  --db-parameter-group-name telecom-mssql-billing-params \
  --parameters \
    "ParameterName=cost threshold for parallelism,ParameterValue=50,ApplyMethod=immediate" \
    "ParameterName=max degree of parallelism,ParameterValue=4,ApplyMethod=immediate" \
    "ParameterName=optimize for ad hoc workloads,ParameterValue=1,ApplyMethod=immediate"
# cost threshold for parallelism = 50: Only use parallel queries for operations
#   costing more than 50. Prevents small billing lookups from wasting threads.
# max degree of parallelism = 4: Use up to 4 CPU cores per query.
#   For 16-vCPU instance, this leaves cores available for other queries.
# optimize for ad hoc workloads = 1: Prevents plan cache bloat from
#   one-time billing queries. Saves memory.

# ===================================================================
# Kishore: MySQL Parameter Group for CDR
# ===================================================================
aws rds create-db-parameter-group \
  --db-parameter-group-name telecom-mysql-cdr-params \
  --db-parameter-group-family mysql8.0 \
  --description "Telecom CDR MySQL tuning - Kishore"

aws rds modify-db-parameter-group \
  --db-parameter-group-name telecom-mysql-cdr-params \
  --parameters \
    "ParameterName=innodb_buffer_pool_size,ParameterValue={DBInstanceClassMemory*3/4},ApplyMethod=pending-reboot" \
    "ParameterName=max_connections,ParameterValue=1000,ApplyMethod=pending-reboot" \
    "ParameterName=innodb_flush_log_at_trx_commit,ParameterValue=1,ApplyMethod=immediate" \
    "ParameterName=slow_query_log,ParameterValue=1,ApplyMethod=immediate" \
    "ParameterName=long_query_time,ParameterValue=2,ApplyMethod=immediate" \
    "ParameterName=innodb_io_capacity,ParameterValue=4000,ApplyMethod=immediate" \
    "ParameterName=innodb_io_capacity_max,ParameterValue=8000,ApplyMethod=immediate"
# innodb_buffer_pool_size = 75% of memory: For 64GB instance, this gives 48GB
#   buffer pool. Keeps most CDR data in memory for fast reads.
# innodb_flush_log_at_trx_commit = 1: Full ACID compliance. Every commit
#   is flushed to disk. Essential for CDR data integrity.
# slow_query_log = 1, long_query_time = 2: Log any CDR query taking > 2 seconds.
#   Kishore reviews these daily to optimize CDR lookups.
# innodb_io_capacity = 4000: Tells InnoDB it has fast storage (gp3 with 10K IOPS).
#   Allows more aggressive flushing and checkpointing.

# ===================================================================
# Meena: PostgreSQL Parameter Group for CRM
# ===================================================================
aws rds create-db-parameter-group \
  --db-parameter-group-name telecom-pg-crm-params \
  --db-parameter-group-family postgres16 \
  --description "Telecom CRM PostgreSQL tuning - Meena"

aws rds modify-db-parameter-group \
  --db-parameter-group-name telecom-pg-crm-params \
  --parameters \
    "ParameterName=shared_buffers,ParameterValue={DBInstanceClassMemory/4},ApplyMethod=pending-reboot" \
    "ParameterName=effective_cache_size,ParameterValue={DBInstanceClassMemory*3/4},ApplyMethod=pending-reboot" \
    "ParameterName=work_mem,ParameterValue=65536,ApplyMethod=immediate" \
    "ParameterName=maintenance_work_mem,ParameterValue=524288,ApplyMethod=immediate" \
    "ParameterName=max_connections,ParameterValue=500,ApplyMethod=pending-reboot" \
    "ParameterName=random_page_cost,ParameterValue=1.1,ApplyMethod=immediate" \
    "ParameterName=log_min_duration_statement,ParameterValue=1000,ApplyMethod=immediate" \
    "ParameterName=checkpoint_completion_target,ParameterValue=0.9,ApplyMethod=immediate"
# shared_buffers = 25% of memory: PG's internal cache. For 64GB, 16GB.
# effective_cache_size = 75% of memory: Tells PG's optimizer about OS cache.
#   Doesn't allocate memory, just helps query planning.
# work_mem = 64MB: Memory per sort/hash operation. CRM queries often sort
#   large customer lists. Too low = disk sorts = slow queries.
# random_page_cost = 1.1: Tells PG that random I/O on SSD is almost as fast
#   as sequential. Encourages index scans over sequential scans.
# log_min_duration_statement = 1000: Log any query taking > 1 second.

# ===================================================================
# Associate parameter groups with instances
# ===================================================================
aws rds modify-db-instance \
  --db-instance-identifier prod-telephony1-mssql-billing \
  --db-parameter-group-name telecom-mssql-billing-params \
  --apply-immediately --region us-east-1

aws rds modify-db-instance \
  --db-instance-identifier prod-telephony3-mysql-cdr \
  --db-parameter-group-name telecom-mysql-cdr-params \
  --apply-immediately --region ap-south-1

aws rds modify-db-instance \
  --db-instance-identifier prod-telephony2-pg-crm \
  --db-parameter-group-name telecom-pg-crm-params \
  --apply-immediately --region eu-west-3

# View current parameter values
aws rds describe-db-parameters \
  --db-parameter-group-name telecom-mssql-billing-params \
  --query 'Parameters[?ParameterValue!=`null`].{Name:ParameterName,Value:ParameterValue,Apply:ApplyMethod}' \
  --output table

# Check pending maintenance across all regions
for region in us-east-1 eu-west-3 ap-south-1; do
  echo "=== $region Pending Maintenance ==="
  aws rds describe-pending-maintenance-actions --region $region \
    --query 'PendingMaintenanceActions[].{DB:ResourceIdentifier,Action:PendingMaintenanceActionDetails[].Action,AutoApply:PendingMaintenanceActionDetails[].AutoAppliedAfterDate}' \
    --output table
done
```

---

## 3.4 RDS Security — Complete Security Stack

### Security Layers for Telecom Databases

Telecom databases contain highly sensitive data: billing records, subscriber personal information, and call records. Multiple layers of security are required:

```
  TELECOM RDS SECURITY ARCHITECTURE:

  Layer 1: NETWORK SECURITY
  ┌──────────────────────────────────────────────────────────┐
  │ VPC + Private Subnets (no internet access)               │
  │ Security Groups (only allow specific app SGs/CIDRs)     │
  │ Network ACLs (secondary defense)                         │
  │ no-publicly-accessible flag set to true                  │
  └──────────────────────────────────────────────────────────┘
           │
  Layer 2: ENCRYPTION AT REST
  ┌──────────────────────────────────────────────────────────┐
  │ AWS KMS (Key Management Service)                         │
  │ Encrypts: data files, backups, snapshots, logs, replicas│
  │ IMPORTANT: Must be enabled at creation time!             │
  │ Cannot add encryption to an existing unencrypted DB      │
  │ (must create new encrypted DB and migrate data)          │
  └──────────────────────────────────────────────────────────┘
           │
  Layer 3: ENCRYPTION IN TRANSIT
  ┌──────────────────────────────────────────────────────────┐
  │ SSL/TLS for all client connections                       │
  │ MSSQL: rds.force_ssl = 1 in parameter group              │
  │ MySQL: require_secure_transport = ON                     │
  │ PostgreSQL: rds.force_ssl = 1 in parameter group         │
  └──────────────────────────────────────────────────────────┘
           │
  Layer 4: AUTHENTICATION & SECRETS
  ┌──────────────────────────────────────────────────────────┐
  │ AWS Secrets Manager for DB credentials                   │
  │ Auto-rotation every 30 days                              │
  │ Applications retrieve credentials via API                │
  │ NO hardcoded passwords in config files!                  │
  │                                                          │
  │ IAM Database Authentication (MySQL/PostgreSQL only)      │
  │ Use IAM tokens instead of passwords                      │
  └──────────────────────────────────────────────────────────┘
```

### AWS CLI — Complete Security Setup

```bash
# ===================================================================
# Venkat: Create KMS keys for database encryption
# ===================================================================
aws kms create-key \
  --description "Telecom DB Encryption Key - telephony1" \
  --key-usage ENCRYPT_DECRYPT \
  --region us-east-1
# → KeyId = key-tel1

aws kms create-alias \
  --alias-name alias/telecom-db-key-tel1 \
  --target-key-id key-tel1 \
  --region us-east-1

# Enable automatic key rotation (annually)
aws kms enable-key-rotation --key-id key-tel1 --region us-east-1

# ===================================================================
# Store credentials in Secrets Manager
# ===================================================================

# Venkat: MSSQL Billing credentials
aws secretsmanager create-secret \
  --name telecom/telephony1/mssql/billing/admin \
  --description "MSSQL Billing admin - telephony1 - Venkat" \
  --secret-string '{
    "username": "billingadmin",
    "password": "Tel3com$Billing!2025",
    "engine": "sqlserver",
    "host": "prod-telephony1-mssql-billing.crwxyz.us-east-1.rds.amazonaws.com",
    "port": "1433",
    "dbname": "billing"
  }'

# Kishore: MySQL CDR credentials
aws secretsmanager create-secret \
  --name telecom/telephony3/mysql/cdr/admin \
  --description "MySQL CDR admin - telephony3 - Kishore" \
  --secret-string '{
    "username": "cdradmin",
    "password": "Tel3com$CDR!2025",
    "engine": "mysql",
    "host": "prod-telephony3-mysql-cdr.crwxyz.ap-south-1.rds.amazonaws.com",
    "port": "3306",
    "dbname": "cdr"
  }'

# Meena: PostgreSQL CRM credentials
aws secretsmanager create-secret \
  --name telecom/telephony2/postgresql/crm/admin \
  --description "PostgreSQL CRM admin - telephony2 - Meena" \
  --secret-string '{
    "username": "crmadmin",
    "password": "Tel3com$CRM!2025",
    "engine": "postgres",
    "host": "prod-telephony2-pg-crm.crwxyz.eu-west-3.rds.amazonaws.com",
    "port": "5432",
    "dbname": "crm"
  }'

# Enable auto-rotation every 30 days for all secrets
for secret in telecom/telephony1/mssql/billing/admin telecom/telephony3/mysql/cdr/admin telecom/telephony2/postgresql/crm/admin; do
  aws secretsmanager rotate-secret \
    --secret-id $secret \
    --rotation-rules '{"AutomaticallyAfterDays": 30}'
done

# Application retrieves credentials at runtime (never hardcoded!)
aws secretsmanager get-secret-value \
  --secret-id telecom/telephony1/mssql/billing/admin \
  --query 'SecretString' --output text
```

---

## 3.5 Amazon Aurora Overview

### What is Aurora and Why Consider It?

Amazon Aurora is AWS's cloud-native relational database. It's compatible with MySQL and PostgreSQL but redesigned from the ground up for cloud scalability. For the telecom team, Aurora is worth considering as an upgrade path for the PostgreSQL CRM database because of its superior availability.

**Standard RDS** stores data on EBS volumes with 2 copies (primary + standby in Multi-AZ). If both copies fail, you lose data.

**Aurora** stores data in a distributed storage layer with 6 copies across 3 AZs. It can lose 2 copies and still handle writes. It can lose 3 copies and still handle reads. Storage auto-heals — when a copy fails, Aurora automatically recreates it from the remaining copies.

```
  AURORA vs STANDARD RDS — STORAGE COMPARISON:

  Standard RDS PostgreSQL:
  ┌──────────┐         ┌──────────┐
  │ Primary  │──EBS───►│ Standby  │     2 copies in 2 AZs
  │ AZ-a     │ sync    │ AZ-b     │     EBS in each AZ
  └──────────┘         └──────────┘

  Aurora PostgreSQL:
  ┌──────────┐
  │ Writer   │──────────────────────────────────────┐
  │ Instance │                                      │
  └──────────┘                                      ▼
  ┌──────────┐   ┌─────────────────────────────────────────┐
  │ Reader 1 │──►│  AURORA SHARED STORAGE LAYER            │
  └──────────┘   │                                         │
  ┌──────────┐   │  6 COPIES across 3 AZs                 │
  │ Reader 2 │──►│  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐│
  └──────────┘   │  │Copy│ │Copy│ │Copy│ │Copy│ │Copy│ │Copy││
                 │  │ 1  │ │ 2  │ │ 3  │ │ 4  │ │ 5  │ │ 6  ││
                 │  │AZ-a│ │AZ-a│ │AZ-b│ │AZ-b│ │AZ-c│ │AZ-c││
                 │  └────┘ └────┘ └────┘ └────┘ └────┘ └────┘│
                 │                                         │
                 │  Auto-grows up to 128 TB                │
                 │  Continuous backup to S3                 │
                 │  Self-healing (auto-repairs corrupted    │
                 │  copies from remaining good copies)     │
                 └─────────────────────────────────────────┘
```

### AWS CLI — Aurora for Telecom CRM

```bash
# Venkat: Create Aurora PostgreSQL cluster for CRM upgrade
aws rds create-db-cluster \
  --db-cluster-identifier tel1-aurora-pg-crm \
  --engine aurora-postgresql \
  --engine-version 16.4 \
  --master-username crmadmin \
  --master-user-password 'Tel3com$CRM!2025' \
  --db-subnet-group-name telephony1-db-subnets \
  --vpc-security-group-ids sg-pg-tel1 \
  --storage-encrypted \
  --backup-retention-period 14 \
  --region us-east-1

# Writer instance (handles reads + writes)
aws rds create-db-instance \
  --db-instance-identifier tel1-aurora-pg-crm-writer \
  --db-cluster-identifier tel1-aurora-pg-crm \
  --db-instance-class db.r6g.2xlarge \
  --engine aurora-postgresql \
  --region us-east-1

# Reader instance (handles reads — for CRM reporting queries)
aws rds create-db-instance \
  --db-instance-identifier tel1-aurora-pg-crm-reader1 \
  --db-cluster-identifier tel1-aurora-pg-crm \
  --db-instance-class db.r6g.xlarge \
  --engine aurora-postgresql \
  --region us-east-1

# Get Aurora endpoints
aws rds describe-db-clusters \
  --db-cluster-identifier tel1-aurora-pg-crm \
  --query 'DBClusters[0].{WriterEndpoint:Endpoint,ReaderEndpoint:ReaderEndpoint,Port:Port}' \
  --output table --region us-east-1
# Writer endpoint: for the CRM application (reads + writes)
# Reader endpoint: for reporting and analytics (reads only, auto load-balanced)
```

---
---

# DAY 4 — Monitoring, HA & DR

---

## 4.1 CloudWatch Metrics & Alarms

### What is CloudWatch?

Amazon CloudWatch is AWS's monitoring and observability service. For DBAs, it provides real-time metrics about your database instances' health, performance, and resource utilization. Think of it as your database monitoring dashboard (like SQL Server Activity Monitor, MySQL Performance Schema, or PostgreSQL pg_stat_activity — but for infrastructure metrics).

CloudWatch collects metrics automatically from RDS instances every 60 seconds (or every 1 second with Enhanced Monitoring). You can set alarms to notify the team when metrics exceed thresholds.

### Critical Metrics Every Telecom DBA Must Monitor

| Metric | What It Measures | Warning Threshold | Critical Threshold | Telecom Impact |
|--------|-----------------|-------------------|-------------------|----------------|
| **CPUUtilization** | Percentage of CPU used | > 70% sustained | > 85% sustained | Billing queries slow, CDR processing delayed |
| **FreeableMemory** | Available RAM in bytes | < 2 GB | < 500 MB | Buffer pool/shared_buffers pressure, queries go to disk |
| **FreeStorageSpace** | Available disk space | < 20% of total | < 10% of total | Database stops accepting writes if full! |
| **DatabaseConnections** | Active connections | > 80% of max | > 90% of max | New subscriber connections rejected |
| **ReadIOPS / WriteIOPS** | I/O operations per second | Near 80% of provisioned | Near limit | CDR write latency increases |
| **ReadLatency / WriteLatency** | Average I/O latency | > 5ms | > 10ms | Billing transactions timeout |
| **SwapUsage** | Bytes of swap used | > 0 bytes | > 100 MB | Memory severely insufficient |
| **ReplicaLag** | Seconds behind primary | > 10 seconds | > 60 seconds | CDR read replicas serving stale data |
| **DiskQueueDepth** | Pending I/O operations | > 1 sustained | > 5 sustained | Storage bottleneck, all queries slow |

### AWS CLI — Create Alarms for All Telecom Databases

```bash
# ===================================================================
# Step 1: Create SNS topic for DBA team alerts
# ===================================================================
aws sns create-topic --name telecom-dba-alerts --region us-east-1
# → TopicArn

# Subscribe all team members to receive email alerts
aws sns subscribe --topic-arn arn:aws:sns:us-east-1:123456789012:telecom-dba-alerts \
  --protocol email --notification-endpoint venkat@telecom.com
aws sns subscribe --topic-arn arn:aws:sns:us-east-1:123456789012:telecom-dba-alerts \
  --protocol email --notification-endpoint kishore@telecom.com
aws sns subscribe --topic-arn arn:aws:sns:us-east-1:123456789012:telecom-dba-alerts \
  --protocol email --notification-endpoint meena@telecom.com

# ===================================================================
# Venkat: MSSQL Billing alarms (telephony1)
# ===================================================================

# CPU > 80% for 10 minutes — billing queries may be slow
aws cloudwatch put-metric-alarm \
  --alarm-name "Tel1-MSSQL-Billing-HighCPU" \
  --alarm-description "MSSQL Billing CPU > 80% for 10min - Escalate to Venkat" \
  --namespace AWS/RDS --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=prod-telephony1-mssql-billing \
  --statistic Average --period 300 --threshold 80 \
  --comparison-operator GreaterThanThreshold --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:telecom-dba-alerts \
  --region us-east-1

# Free storage < 50 GB — billing data growing fast
aws cloudwatch put-metric-alarm \
  --alarm-name "Tel1-MSSQL-Billing-LowStorage" \
  --alarm-description "MSSQL Billing storage < 50GB - Venkat must increase storage" \
  --namespace AWS/RDS --metric-name FreeStorageSpace \
  --dimensions Name=DBInstanceIdentifier,Value=prod-telephony1-mssql-billing \
  --statistic Average --period 300 --threshold 53687091200 \
  --comparison-operator LessThanThreshold --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:telecom-dba-alerts \
  --region us-east-1

# ===================================================================
# Kishore: MySQL CDR alarms (telephony3)
# ===================================================================

# Write latency > 5ms — CDR ingestion is bottlenecked
aws cloudwatch put-metric-alarm \
  --alarm-name "Tel3-MySQL-CDR-HighWriteLatency" \
  --alarm-description "MySQL CDR write latency > 5ms - Kishore check IOPS" \
  --namespace AWS/RDS --metric-name WriteLatency \
  --dimensions Name=DBInstanceIdentifier,Value=prod-telephony3-mysql-cdr \
  --statistic Average --period 60 --threshold 0.005 \
  --comparison-operator GreaterThanThreshold --evaluation-periods 5 \
  --alarm-actions arn:aws:sns:ap-south-1:123456789012:telecom-dba-alerts \
  --region ap-south-1

# Replica lag > 30 seconds — read replicas serving stale CDR data
aws cloudwatch put-metric-alarm \
  --alarm-name "Tel3-MySQL-CDR-ReplicaLag" \
  --alarm-description "MySQL CDR replica lag > 30sec - Kishore investigate" \
  --namespace AWS/RDS --metric-name ReplicaLag \
  --dimensions Name=DBInstanceIdentifier,Value=prod-telephony3-mysql-cdr-replica \
  --statistic Average --period 60 --threshold 30 \
  --comparison-operator GreaterThanThreshold --evaluation-periods 3 \
  --alarm-actions arn:aws:sns:ap-south-1:123456789012:telecom-dba-alerts \
  --region ap-south-1

# ===================================================================
# Meena: PostgreSQL CRM alarms (telephony2)
# ===================================================================

# Database connections > 400 (max_connections = 500)
aws cloudwatch put-metric-alarm \
  --alarm-name "Tel2-PG-CRM-HighConnections" \
  --alarm-description "PostgreSQL CRM connections > 400 (80% of max) - Meena alert" \
  --namespace AWS/RDS --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=prod-telephony2-pg-crm \
  --statistic Average --period 60 --threshold 400 \
  --comparison-operator GreaterThanThreshold --evaluation-periods 3 \
  --alarm-actions arn:aws:sns:eu-west-3:123456789012:telecom-dba-alerts \
  --region eu-west-3

# Swap usage > 0 — PostgreSQL should NEVER use swap
aws cloudwatch put-metric-alarm \
  --alarm-name "Tel2-PG-CRM-SwapUsage" \
  --alarm-description "PostgreSQL CRM using swap - memory pressure - Escalate to Venkat" \
  --namespace AWS/RDS --metric-name SwapUsage \
  --dimensions Name=DBInstanceIdentifier,Value=prod-telephony2-pg-crm \
  --statistic Average --period 300 --threshold 1048576 \
  --comparison-operator GreaterThanThreshold --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:eu-west-3:123456789012:telecom-dba-alerts \
  --region eu-west-3
```

---

## 4.2 RDS Performance Insights

### What is Performance Insights?

Performance Insights is a database monitoring feature that helps you quickly identify which queries are causing the most load on your database. It answers the question: "Why is my database slow right now?"

It shows you a graph of "Average Active Sessions" (AAS) broken down by wait events. If the AAS line goes above the number of vCPUs, your database is overloaded.

**Free tier:** 7 days of data retention. **Paid tier:** Up to 2 years (731 days).

```bash
# Enable Performance Insights on all telecom databases
for db in prod-telephony1-mssql-billing prod-telephony1-mysql-cdr prod-telephony1-pg-crm; do
  echo "Enabling PI on $db"
  aws rds modify-db-instance --db-instance-identifier $db \
    --enable-performance-insights \
    --performance-insights-retention-period 731 \
    --apply-immediately --region us-east-1
done
```

---

## 4.3 High Availability & Failover — Detailed

### Manual Failover Testing

Every telecom DBA must regularly test failover to ensure it works correctly. Venkat schedules monthly failover tests on non-production instances:

```bash
# Venkat: Test MSSQL billing failover (non-prod first!)
aws rds reboot-db-instance \
  --db-instance-identifier dev-telephony1-mssql-billing \
  --force-failover \
  --region us-east-1
# This forces a failover to the standby. Monitor:
# - How long does failover take?
# - Do applications reconnect automatically?
# - Is there any data loss? (there shouldn't be with synchronous replication)

# Kishore: Create MySQL read replica for CDR reporting queries
# This offloads read queries from the primary, protecting CDR write performance
aws rds create-db-instance-read-replica \
  --db-instance-identifier prod-telephony3-mysql-cdr-replica \
  --source-db-instance-identifier prod-telephony3-mysql-cdr \
  --db-instance-class db.r6i.xlarge \
  --region ap-south-1

# Meena: Check Multi-AZ status across all databases
for region in us-east-1 eu-west-3 ap-south-1; do
  echo "=== $region ==="
  aws rds describe-db-instances --region $region \
    --query 'DBInstances[].{DB:DBInstanceIdentifier,MultiAZ:MultiAZ,AZ:AvailabilityZone,SecondaryAZ:SecondaryAvailabilityZone}' \
    --output table
done
```

---

## 4.4 Disaster Recovery Strategies

### Telecom DR Architecture

```
  TELECOM DR STRATEGY — COMPLETE

  ┌──────────────┬──────────────────────┬────────┬─────────┬─────────┐
  │ Database     │ DR Method            │ RPO    │ RTO     │ Owner   │
  ├──────────────┼──────────────────────┼────────┼─────────┼─────────┤
  │ MSSQL Billing│ Cross-region snapshot │ 1 hour │ 2 hours │ Venkat  │
  │              │ copies to us-east-1  │        │         │         │
  │              │ (Backup & Restore)   │        │         │         │
  ├──────────────┼──────────────────────┼────────┼─────────┼─────────┤
  │ MySQL CDR    │ Cross-region read    │ ~1 sec │ 5 min   │ Kishore │
  │              │ replica in us-east-1 │        │         │         │
  │              │ (Warm Standby)       │        │         │         │
  ├──────────────┼──────────────────────┼────────┼─────────┼─────────┤
  │ PostgreSQL   │ Aurora Global DB     │ ~1 sec │ 1 min   │ Venkat  │
  │ CRM          │ (Multi-Region Active)│        │         │         │
  └──────────────┴──────────────────────┴────────┴─────────┴─────────┘

  RPO = Recovery Point Objective = How much data can we afford to lose?
  RTO = Recovery Time Objective = How quickly must we be back online?
```

---

## 4.5 Troubleshooting — Complete Health Check Script

```bash
#!/bin/bash
# Meena's Morning Health Check Script
# Run at 8 AM every day across all 3 telephony regions

echo "╔══════════════════════════════════════════════════════════╗"
echo "║  TELECOM DBA MORNING HEALTH CHECK                       ║"
echo "║  Date: $(date)                                           ║"
echo "║  Run by: Meena (Jr DBA)                                 ║"
echo "╚══════════════════════════════════════════════════════════╝"

for region in us-east-1 eu-west-3 ap-south-1; do
  echo ""
  echo "═══════════════════════════════════════════"
  echo "  REGION: $region"
  echo "═══════════════════════════════════════════"

  echo ""
  echo "--- 1. Instance Status ---"
  aws rds describe-db-instances --region $region \
    --query 'DBInstances[].{Name:DBInstanceIdentifier,Status:DBInstanceStatus,Engine:Engine,MultiAZ:MultiAZ}' \
    --output table

  echo ""
  echo "--- 2. Active Alarms ---"
  aws cloudwatch describe-alarms --state-value ALARM --region $region \
    --query 'MetricAlarms[].{Alarm:AlarmName,Metric:MetricName,Threshold:Threshold}' \
    --output table 2>/dev/null || echo "  No alarms in ALARM state"

  echo ""
  echo "--- 3. Events (last 12 hours) ---"
  aws rds describe-events --duration 720 --region $region \
    --query 'Events[].{Source:SourceIdentifier,Message:Message,Time:Date}' \
    --output table

  echo ""
  echo "--- 4. Pending Maintenance ---"
  aws rds describe-pending-maintenance-actions --region $region \
    --output table 2>/dev/null || echo "  No pending maintenance"

  echo ""
  echo "--- 5. Latest Automated Backups ---"
  aws rds describe-db-snapshots --snapshot-type automated --region $region \
    --query 'sort_by(DBSnapshots,&SnapshotCreateTime)[-3:].{DB:DBInstanceIdentifier,Created:SnapshotCreateTime,Status:Status}' \
    --output table
done

echo ""
echo "╔══════════════════════════════════════════════════════════╗"
echo "║  HEALTH CHECK COMPLETE                                   ║"
echo "║  If any ALARM found → escalate to Venkat/Kishore         ║"
echo "╚══════════════════════════════════════════════════════════╝"
```

---
---

# DAY 5 — Security, Automation & Cost

---

## 5.1 KMS & Secrets Manager — Deep Dive

### KMS (Key Management Service) — How It Works

KMS manages encryption keys for your databases. When you enable encryption on an RDS instance, KMS creates a data key that encrypts your database files, backups, snapshots, and replicas. You never handle the actual encryption — it all happens transparently.

```bash
# Venkat: Create KMS keys for each telecom region
for region in us-east-1 eu-west-3 ap-south-1; do
  KEY=$(aws kms create-key --description "Telecom DB Key - $region" \
    --key-usage ENCRYPT_DECRYPT --region $region \
    --query 'KeyMetadata.KeyId' --output text)
  aws kms create-alias --alias-name "alias/telecom-db-$region" \
    --target-key-id $KEY --region $region
  aws kms enable-key-rotation --key-id $KEY --region $region
  echo "Created KMS key in $region: $KEY"
done
```

---

## 5.2 AWS CLI for DBAs — Daily Operations Cheat Sheet

```bash
# ═══ INSTANCE MANAGEMENT ═══
aws rds describe-db-instances --output table                           # List all
aws rds reboot-db-instance --db-instance-identifier INSTANCE          # Reboot
aws rds stop-db-instance --db-instance-identifier DEV-INSTANCE        # Stop dev
aws rds start-db-instance --db-instance-identifier DEV-INSTANCE       # Start dev

# ═══ SCALING ═══
aws rds modify-db-instance --db-instance-identifier INSTANCE \
  --db-instance-class db.r6i.4xlarge --apply-immediately              # Scale up
aws rds modify-db-instance --db-instance-identifier INSTANCE \
  --allocated-storage 1000 --apply-immediately                        # Add storage

# ═══ BACKUPS ═══
aws rds create-db-snapshot --db-instance-identifier INSTANCE \
  --db-snapshot-identifier "manual-$(date +%Y%m%d-%H%M)"              # Snapshot

# ═══ LOGS ═══
# MSSQL error log:
aws rds download-db-log-file-portion --db-instance-identifier MSSQL_INSTANCE \
  --log-file-name error/sqlserver-error.log --output text
# MySQL error log:
aws rds download-db-log-file-portion --db-instance-identifier MYSQL_INSTANCE \
  --log-file-name error/mysql-error.log --output text
# PostgreSQL log:
aws rds download-db-log-file-portion --db-instance-identifier PG_INSTANCE \
  --log-file-name error/postgresql.log --output text

# ═══ MONITORING ═══
aws rds describe-events --duration 60                                  # Last hour
aws cloudwatch describe-alarms --state-value ALARM                     # Active alarms
```

---

## 5.3 Automation & Snapshot Scheduling

```bash
# Meena: Automate dev/test stop/start to save cost
# Add to crontab: Stop at 7 PM IST (1:30 PM UTC)
# 30 13 * * 1-5 /home/meena/stop-dev-dbs.sh
for db in dev-telephony1-mssql dev-telephony1-mysql dev-telephony1-pg \
          dev-telephony2-mssql dev-telephony2-mysql dev-telephony2-pg; do
  aws rds stop-db-instance --db-instance-identifier $db
done

# Start at 8 AM IST (2:30 AM UTC)
# 30 2 * * 1-5 /home/meena/start-dev-dbs.sh
for db in dev-telephony1-mssql dev-telephony1-mysql dev-telephony1-pg \
          dev-telephony2-mssql dev-telephony2-mysql dev-telephony2-pg; do
  aws rds start-db-instance --db-instance-identifier $db
done
```

---

## 5.4 Cost Management

### Cost Optimization Strategies

| Strategy | Savings | Who Implements | Details |
|----------|---------|---------------|---------|
| Reserved Instances | 30-60% | Venkat | 1-year commitment for all 9 production databases |
| Right-sizing | 20-40% | Venkat/Kishore | Review CPU/memory monthly, downsize over-provisioned |
| Stop dev/test | 70% | Meena | Automated stop at 7 PM, start at 8 AM (weekdays only) |
| gp3 over io2 | 40-50% | Kishore | All databases use gp3 unless >16K IOPS needed |
| Single-AZ dev | 50% | Meena | No Multi-AZ for non-production databases |
| Snapshot cleanup | 5-10% | Meena | Delete manual snapshots older than 90 days |

```bash
# Venkat: Monthly cost check
aws ce get-cost-and-usage \
  --time-period Start=$(date -d "first day of this month" +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY --metrics BlendedCost \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Relational Database Service"]}}' \
  --output table
```

---

## 5.5 Cloud DBA Operating Model — Complete

### Telecom DBA Daily/Weekly/Monthly Routine

| Schedule | Task | Who | Details |
|----------|------|-----|---------|
| **Daily 8:00 AM** | Morning Health Check | Meena | Run health check script, verify all 9 instances running |
| **Daily 8:30 AM** | Alarm Review | Meena | Check CloudWatch alarms, escalate ALARM state to seniors |
| **Daily 9:00 AM** | Backup Verification | Meena | Confirm overnight backups completed for all 3 engines |
| **Daily 9:30 AM** | Performance Review | Venkat/Kishore | Performance Insights — top SQL across MSSQL, MySQL, PG |
| **Daily ongoing** | Ticket Resolution | All | Schema changes, user access, query optimization |
| **Weekly Sunday** | Patching Review | Venkat | Check pending patches for all engines across 3 regions |
| **Weekly** | DR Snapshot Verify | Kishore | Confirm cross-region snapshots are current |
| **Weekly** | Failover Test (dev) | Venkat | Test Multi-AZ failover on dev instances |
| **Weekly** | Security Review | Kishore | Audit security group rules, check for 0.0.0.0/0 exposure |
| **Monthly 1st** | Right-sizing Review | Venkat | Analyze 30-day CPU/memory metrics, resize if needed |
| **Monthly** | Cost Report | Meena | Generate RDS cost breakdown by engine and region |
| **Monthly** | Snapshot Cleanup | Meena | Delete manual snapshots older than 90 days |
| **Monthly** | Password Rotation Verify | Kishore | Confirm Secrets Manager auto-rotation is working |

---

## Complete Telecom Database Inventory

| Region | Instance Name | Engine | Version | Class | Storage | IOPS | Multi-AZ | Owner |
|--------|-------------|--------|---------|-------|---------|------|----------|-------|
| telephony1 | prod-telephony1-mssql-billing | SQL Server SE | 2022 (16.00) | db.r6i.4xlarge | 500 GB gp3 | 6,000 | Yes | Venkat |
| telephony1 | prod-telephony1-mysql-cdr | MySQL | 8.0.36 | db.r6i.2xlarge | 1 TB gp3 | 10,000 | Yes | Venkat |
| telephony1 | prod-telephony1-pg-crm | PostgreSQL | 16.4 | db.r6i.2xlarge | 500 GB gp3 | 6,000 | Yes | Venkat |
| telephony2 | prod-telephony2-mssql-billing | SQL Server SE | 2022 (16.00) | db.r6i.4xlarge | 500 GB gp3 | 6,000 | Yes | Meena |
| telephony2 | prod-telephony2-mysql-cdr | MySQL | 8.0.36 | db.r6i.2xlarge | 1 TB gp3 | 10,000 | Yes | Meena |
| telephony2 | prod-telephony2-pg-crm | PostgreSQL | 16.4 | db.r6i.2xlarge | 500 GB gp3 | 6,000 | Yes | Meena |
| telephony3 | prod-telephony3-mssql-billing | SQL Server SE | 2022 (16.00) | db.r6i.4xlarge | 500 GB gp3 | 6,000 | Yes | Kishore |
| telephony3 | prod-telephony3-mysql-cdr | MySQL | 8.0.36 | db.r6i.2xlarge | 1 TB gp3 | 10,000 | Yes | Kishore |
| telephony3 | prod-telephony3-pg-crm | PostgreSQL | 16.4 | db.r6i.2xlarge | 500 GB gp3 | 6,000 | Yes | Kishore |

---

*5-Day AWS Cloud DBA Training — Detailed Notes & Explanations*
*Telecom Domain | MSSQL, MySQL, PostgreSQL*
*DBA Team: Venkat (Sr DBA) | Kishore (Sr DBA) | Meena (Jr DBA)*
