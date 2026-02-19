# 5-Day AWS Training Program for Cloud DBAs
## Telecom Domain — Telephony Platform

> **Company:** Global Telecom Services  
> **Application:** Telephony Platform (telephony1, telephony2, telephony3)  
> **Database Engines:** MS SQL Server, MySQL, PostgreSQL  
> **Regions:** us-east-1 (telephony1), eu-west-3 (telephony2), ap-south-1 (telephony3)  

### DBA Team

| Name | Role | Responsibility | Primary DB | Region |
|------|------|---------------|------------|--------|
| **Venkat** | Sr DBA | DB Architecture, HA/DR, Performance Tuning, Team Lead | MSSQL & PostgreSQL | telephony1 (us-east-1) |
| **Kishore** | Sr DBA | Multi-Cloud Infrastructure, Security, Automation | MySQL & PostgreSQL | telephony3 (ap-south-1) |
| **Meena** | Jr DBA | Monitoring, Backups, Daily Operations, Patching | All Engines | telephony2 (eu-west-3) |

### Telecom Application Architecture

```
  GLOBAL TELECOM PLATFORM — 3 REGIONS

  telephony1 (us-east-1)        telephony2 (eu-west-3)       telephony3 (ap-south-1)
  US Subscribers                EU Subscribers                India Subscribers
  ┌──────────────────┐         ┌──────────────────┐         ┌──────────────────┐
  │ App: Billing     │         │ App: Billing     │         │ App: Billing     │
  │ App: CDR         │         │ App: CDR         │         │ App: CDR         │
  │ App: CRM         │         │ App: CRM         │         │ App: CRM         │
  │ App: Network Ops │         │ App: Network Ops │         │ App: Network Ops │
  └───────┬──────────┘         └───────┬──────────┘         └───────┬──────────┘
          │                            │                            │
          ▼                            ▼                            ▼
  ┌──────────────────┐         ┌──────────────────┐         ┌──────────────────┐
  │ MSSQL: Billing   │         │ MSSQL: Billing   │         │ MSSQL: Billing   │
  │ MySQL: CDR/Logs  │         │ MySQL: CDR/Logs  │         │ MySQL: CDR/Logs  │
  │ PostgreSQL: CRM  │         │ PostgreSQL: CRM  │         │ PostgreSQL: CRM  │
  └──────────────────┘         └──────────────────┘         └──────────────────┘
       Venkat (Sr DBA)             Meena (Jr DBA)              Kishore (Sr DBA)

  DB Mapping:
  ┌─────────────────────────────────────────────────────────────────────┐
  │ MS SQL Server → Billing System (financial transactions, invoices)  │
  │ MySQL         → CDR (Call Detail Records), Logs, Network Events    │
  │ PostgreSQL    → CRM (Customer Relationship Management), Analytics  │
  └─────────────────────────────────────────────────────────────────────┘
```

---

# DAY 1 — AWS Foundations for DBAs

---

## 1.1 AWS Global Infrastructure (Regions, AZs, Multi-AZ)

### Telecom Platform Region Mapping

```
  TELECOM REGIONS:

  ┌─── telephony1 (us-east-1 / N. Virginia) ──────────┐
  │  US subscribers: 50 million                         │
  │  AZ-a: MSSQL Primary + MySQL Primary                │
  │  AZ-b: MSSQL Standby + MySQL Replica                │
  │  AZ-c: PostgreSQL Primary + PostgreSQL Standby       │
  │  DBA: Venkat (Sr DBA)                                │
  └─────────────────────────────────────────────────────┘

  ┌─── telephony2 (eu-west-3 / Paris) ────────────────┐
  │  EU subscribers: 30 million                         │
  │  AZ-a: MSSQL Primary + MySQL Primary                │
  │  AZ-b: MSSQL Standby + MySQL Replica                │
  │  AZ-c: PostgreSQL Primary + PostgreSQL Standby       │
  │  DBA: Meena (Jr DBA)                                 │
  └─────────────────────────────────────────────────────┘

  ┌─── telephony3 (ap-south-1 / Mumbai) ──────────────┐
  │  India subscribers: 120 million                     │
  │  AZ-a: MSSQL Primary + MySQL Primary                │
  │  AZ-b: MSSQL Standby + MySQL Replica                │
  │  AZ-c: PostgreSQL Primary + PostgreSQL Standby       │
  │  DBA: Kishore (Sr DBA)                               │
  └─────────────────────────────────────────────────────┘
```

### AWS CLI Commands

```bash
# Venkat: List all AZs in telephony1 region
aws ec2 describe-availability-zones \
  --region us-east-1 \
  --query 'AvailabilityZones[].{Zone:ZoneName,State:State}' \
  --output table

# Meena: List all AZs in telephony2 region
aws ec2 describe-availability-zones \
  --region eu-west-3 \
  --query 'AvailabilityZones[].{Zone:ZoneName,State:State}' \
  --output table

# Kishore: List all AZs in telephony3 region
aws ec2 describe-availability-zones \
  --region ap-south-1 \
  --query 'AvailabilityZones[].{Zone:ZoneName,State:State}' \
  --output table

# List all available AWS regions
aws ec2 describe-regions --query 'Regions[].RegionName' --output table
```

### Multi-AZ for Telecom Databases

```
  WHY MULTI-AZ IS CRITICAL FOR TELECOM:

  A single dropped call record = lost revenue + compliance issue
  A billing system outage = millions in lost billing cycles

  Multi-AZ gives:
  • Automatic failover in ~60 seconds
  • Zero data loss (synchronous replication)
  • No manual intervention needed

  Example: telephony3 (Mumbai) billing MSSQL
  ┌──────────────┐  sync  ┌──────────────┐
  │ MSSQL Primary│───────►│ MSSQL Standby│
  │ ap-south-1a  │        │ ap-south-1b  │
  └──────────────┘        └──────────────┘
  If AZ-a fails → AZ-b takes over in 60 sec
  120 million subscribers never notice!
```

---

## 1.2 AWS Account Model & Shared Responsibility

### Shared Responsibility — Telecom Context

```
  ┌──────────────────────────────────────────────────────────┐
  │  DBA TEAM RESPONSIBILITY (Venkat, Meena, Kishore)       │
  │                                                          │
  │  MSSQL (Billing):                                        │
  │  • SQL Server parameter tuning (MAXDOP, Cost Threshold) │
  │  • Billing database schema optimization                  │
  │  • TDE encryption management                             │
  │  • SQL Agent jobs for billing batch processing           │
  │                                                          │
  │  MySQL (CDR/Logs):                                       │
  │  • InnoDB buffer pool sizing                             │
  │  • CDR table partitioning (by date)                      │
  │  • Replication monitoring for CDR distribution           │
  │  • Slow query optimization                               │
  │                                                          │
  │  PostgreSQL (CRM):                                       │
  │  • shared_buffers, work_mem tuning                       │
  │  • VACUUM/ANALYZE scheduling                             │
  │  • CRM schema design and indexing                        │
  │  • Connection pooling (PgBouncer)                        │
  ├──────────────────────────────────────────────────────────┤
  │  AWS RESPONSIBILITY                                      │
  │                                                          │
  │  • Physical data center security                         │
  │  • Hardware replacement                                  │
  │  • OS patching (on RDS)                                  │
  │  • Engine patching (on RDS)                              │
  │  • Automated backups infrastructure                      │
  │  • Multi-AZ replication infrastructure                   │
  │  • Storage management and auto-scaling                   │
  └──────────────────────────────────────────────────────────┘
```

---

## 1.3 IAM Basics for DBAs (Users, Roles, Policies)

### IAM Setup for Telecom DBA Team

```
  IAM Structure:

  ┌─── IAM GROUP: "Telecom-DBA-SeniorTeam" ────────────┐
  │  Users: Venkat, Kishore                              │
  │  Policies:                                           │
  │  ├── AmazonRDSFullAccess                            │
  │  ├── CloudWatchFullAccess                            │
  │  ├── AmazonS3FullAccess (for backups)               │
  │  ├── SecretsManagerReadWrite                         │
  │  └── Custom: Telecom-DBA-Senior-Policy               │
  └──────────────────────────────────────────────────────┘

  ┌─── IAM GROUP: "Telecom-DBA-JuniorTeam" ────────────┐
  │  Users: Meena                                        │
  │  Policies:                                           │
  │  ├── AmazonRDSReadOnlyAccess                        │
  │  ├── CloudWatchReadOnlyAccess                        │
  │  ├── Custom: Telecom-DBA-Junior-Policy               │
  │  │   (can create snapshots, view logs,               │
  │  │    CANNOT delete instances or modify prod)        │
  │  └── AmazonRDSPerformanceInsightsReadOnly            │
  └──────────────────────────────────────────────────────┘
```

### AWS CLI Commands

```bash
# Create DBA users
aws iam create-user --user-name venkat.dba
aws iam create-user --user-name kishore.dba
aws iam create-user --user-name meena.dba

# Create groups
aws iam create-group --group-name Telecom-DBA-SeniorTeam
aws iam create-group --group-name Telecom-DBA-JuniorTeam

# Add users to groups
aws iam add-user-to-group --user-name venkat.dba --group-name Telecom-DBA-SeniorTeam
aws iam add-user-to-group --user-name kishore.dba --group-name Telecom-DBA-SeniorTeam
aws iam add-user-to-group --user-name meena.dba --group-name Telecom-DBA-JuniorTeam

# Senior DBA policies
aws iam attach-group-policy --group-name Telecom-DBA-SeniorTeam \
  --policy-arn arn:aws:iam::aws:policy/AmazonRDSFullAccess
aws iam attach-group-policy --group-name Telecom-DBA-SeniorTeam \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchFullAccess

# Junior DBA policies (read-only + snapshot)
aws iam attach-group-policy --group-name Telecom-DBA-JuniorTeam \
  --policy-arn arn:aws:iam::aws:policy/AmazonRDSReadOnlyAccess
aws iam attach-group-policy --group-name Telecom-DBA-JuniorTeam \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchReadOnlyAccess
```

### Custom Policy: Junior DBA (Meena) — Can Monitor & Snapshot, Cannot Delete

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowMonitoringAndSnapshots",
      "Effect": "Allow",
      "Action": [
        "rds:Describe*",
        "rds:List*",
        "rds:CreateDBSnapshot",
        "rds:DownloadDBLogFilePortion",
        "rds:DownloadCompleteDBLogFile"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyDestructiveActions",
      "Effect": "Deny",
      "Action": [
        "rds:DeleteDBInstance",
        "rds:DeleteDBCluster",
        "rds:DeleteDBSnapshot",
        "rds:ModifyDBInstance",
        "rds:RebootDBInstance"
      ],
      "Resource": "arn:aws:rds:*:*:db:prod-telephony*"
    }
  ]
}
```

```bash
# Venkat creates the junior policy
aws iam create-policy \
  --policy-name Telecom-DBA-Junior-Policy \
  --policy-document file://meena-junior-policy.json

aws iam attach-group-policy --group-name Telecom-DBA-JuniorTeam \
  --policy-arn arn:aws:iam::123456789012:policy/Telecom-DBA-Junior-Policy
```

---

## 1.4 VPC Concepts — Telecom Platform

### VPC Architecture for Each Telephony Region

```
  VPC: telephony1-vpc (10.0.0.0/16) — us-east-1
  Managed by: Venkat (Sr DBA)

  ┌─── PUBLIC SUBNET (10.0.100.0/24) AZ-a ────────────┐
  │  Bastion Host (SSH jump) + NAT Gateway              │
  └─────────────────────────────────────────────────────┘
                       │
                   NAT Gateway
                       │
  ┌─── PRIVATE SUBNET 1 (10.0.1.0/24) AZ-a ───────────┐
  │  RDS MSSQL Primary (Billing)                        │
  │  RDS MySQL Primary (CDR)                            │
  └─────────────────────────────────────────────────────┘

  ┌─── PRIVATE SUBNET 2 (10.0.2.0/24) AZ-b ───────────┐
  │  RDS MSSQL Standby (Multi-AZ)                      │
  │  RDS MySQL Read Replica                             │
  └─────────────────────────────────────────────────────┘

  ┌─── PRIVATE SUBNET 3 (10.0.3.0/24) AZ-c ───────────┐
  │  RDS PostgreSQL Primary (CRM)                       │
  │  RDS PostgreSQL Standby (Multi-AZ)                  │
  └─────────────────────────────────────────────────────┘
```

### AWS CLI: VPC Setup for telephony1

```bash
# Venkat: Create VPC for telephony1
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --region us-east-1
# Copy VpcId → vpc-tel1

aws ec2 create-tags --resources vpc-tel1 \
  --tags Key=Name,Value=telephony1-vpc Key=Environment,Value=production Key=ManagedBy,Value=Venkat \
  --region us-east-1

aws ec2 modify-vpc-attribute --vpc-id vpc-tel1 --enable-dns-hostnames --region us-east-1
aws ec2 modify-vpc-attribute --vpc-id vpc-tel1 --enable-dns-support --region us-east-1

# Private subnets for databases
aws ec2 create-subnet --vpc-id vpc-tel1 --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a --region us-east-1
# → sub-tel1-priv1
aws ec2 create-subnet --vpc-id vpc-tel1 --cidr-block 10.0.2.0/24 \
  --availability-zone us-east-1b --region us-east-1
# → sub-tel1-priv2
aws ec2 create-subnet --vpc-id vpc-tel1 --cidr-block 10.0.3.0/24 \
  --availability-zone us-east-1c --region us-east-1
# → sub-tel1-priv3

# Public subnet for bastion
aws ec2 create-subnet --vpc-id vpc-tel1 --cidr-block 10.0.100.0/24 \
  --availability-zone us-east-1a --region us-east-1
# → sub-tel1-pub

# Create DB Subnet Group (required for RDS)
aws rds create-db-subnet-group \
  --db-subnet-group-name telephony1-db-subnets \
  --db-subnet-group-description "Telecom DB subnets telephony1" \
  --subnet-ids sub-tel1-priv1 sub-tel1-priv2 sub-tel1-priv3 \
  --region us-east-1
```

### Repeat for telephony2 (Meena) and telephony3 (Kishore)

```bash
# Meena: telephony2 VPC (Paris)
aws ec2 create-vpc --cidr-block 10.1.0.0/16 --region eu-west-3
# Tag: telephony2-vpc, ManagedBy=Meena

# Kishore: telephony3 VPC (Mumbai)
aws ec2 create-vpc --cidr-block 10.2.0.0/16 --region ap-south-1
# Tag: telephony3-vpc, ManagedBy=Kishore
```

---

## 1.5 Security Groups for Telecom Databases

### Security Groups per Database Engine

```
  SECURITY GROUP RULES FOR TELECOM:

  ┌─── SG: telephony-mssql-sg (Billing DB) ───────────┐
  │  TCP 1433 ← App servers (billing app SG)            │
  │  TCP 1433 ← Bastion subnet (10.0.100.0/24)         │
  │  TCP 1433 ← SSIS/SSRS servers                      │
  └─────────────────────────────────────────────────────┘

  ┌─── SG: telephony-mysql-sg (CDR/Logs DB) ──────────┐
  │  TCP 3306 ← App servers (CDR processing SG)        │
  │  TCP 3306 ← Bastion subnet (10.0.100.0/24)         │
  │  TCP 3306 ← ETL servers                            │
  └─────────────────────────────────────────────────────┘

  ┌─── SG: telephony-pg-sg (CRM DB) ──────────────────┐
  │  TCP 5432 ← App servers (CRM app SG)               │
  │  TCP 5432 ← Bastion subnet (10.0.100.0/24)         │
  │  TCP 5432 ← Analytics/reporting servers             │
  └─────────────────────────────────────────────────────┘
```

### AWS CLI Commands

```bash
# Venkat: Create security groups for telephony1

# MSSQL Security Group (Billing)
aws ec2 create-security-group --group-name telephony-mssql-sg \
  --description "Telecom MSSQL Billing DB" --vpc-id vpc-tel1 --region us-east-1
# → sg-mssql-tel1
aws ec2 authorize-security-group-ingress --group-id sg-mssql-tel1 \
  --protocol tcp --port 1433 --source-group sg-billing-app --region us-east-1
aws ec2 authorize-security-group-ingress --group-id sg-mssql-tel1 \
  --protocol tcp --port 1433 --cidr 10.0.100.0/24 --region us-east-1

# MySQL Security Group (CDR)
aws ec2 create-security-group --group-name telephony-mysql-sg \
  --description "Telecom MySQL CDR DB" --vpc-id vpc-tel1 --region us-east-1
# → sg-mysql-tel1
aws ec2 authorize-security-group-ingress --group-id sg-mysql-tel1 \
  --protocol tcp --port 3306 --source-group sg-cdr-app --region us-east-1
aws ec2 authorize-security-group-ingress --group-id sg-mysql-tel1 \
  --protocol tcp --port 3306 --cidr 10.0.100.0/24 --region us-east-1

# PostgreSQL Security Group (CRM)
aws ec2 create-security-group --group-name telephony-pg-sg \
  --description "Telecom PostgreSQL CRM DB" --vpc-id vpc-tel1 --region us-east-1
# → sg-pg-tel1
aws ec2 authorize-security-group-ingress --group-id sg-pg-tel1 \
  --protocol tcp --port 5432 --source-group sg-crm-app --region us-east-1
aws ec2 authorize-security-group-ingress --group-id sg-pg-tel1 \
  --protocol tcp --port 5432 --cidr 10.0.100.0/24 --region us-east-1
```

---

# DAY 2 — Compute & Storage for Databases

---

## 2.1 EC2 Instance Types for Telecom Databases

### Recommended Instances per DB Engine

| Database | Telecom Use | Instance Type | vCPU | RAM | Why |
|----------|------------|---------------|------|-----|-----|
| **MSSQL** (Billing) | Financial transactions, invoicing | r6i.4xlarge | 16 | 128 GB | Memory-intensive billing queries |
| **MySQL** (CDR) | Call records, high write volume | r6i.2xlarge | 8 | 64 GB | High IOPS for CDR ingestion |
| **PostgreSQL** (CRM) | Customer data, analytics | r6i.2xlarge | 8 | 64 GB | Complex CRM queries need memory |
| **Bastion** | SSH jump box | t3.micro | 2 | 1 GB | Minimal resources needed |

### Best EBS Layout for Each Engine

```
  MSSQL on EC2 (Billing) — Venkat configures:
  D: (500 GB gp3, 6000 IOPS) → .mdf/.ndf data files
  E: (200 GB gp3, 3000 IOPS) → .ldf transaction logs
  F: (100 GB gp3, 3000 IOPS) → TempDB
  G: (500 GB gp3)             → Backups

  MySQL on EC2 (CDR) — Kishore configures:
  /var/lib/mysql      (1 TB gp3, 10000 IOPS)  → InnoDB data + CDR tables
  /var/lib/mysql-logs (200 GB gp3, 3000 IOPS) → Binary logs
  /backup             (1 TB gp3)               → mysqldump backups

  PostgreSQL on EC2 (CRM) — Venkat/Kishore configure:
  /var/lib/postgresql  (500 GB gp3, 6000 IOPS) → Data directory
  /var/lib/pg-wal      (100 GB gp3, 3000 IOPS) → WAL logs
  /backup              (500 GB gp3)             → pg_basebackup
```

---

## 2.2 EBS Volume Types — Telecom Requirements

### Volume Selection for Telecom Workloads

| Telecom Workload | Volume Type | IOPS | Why |
|-----------------|-------------|------|-----|
| MSSQL Billing data | gp3 (6000 IOPS) | 6,000 | Steady transactional billing |
| MySQL CDR ingestion | gp3 (10000 IOPS) | 10,000 | High-volume call record writes |
| PostgreSQL CRM | gp3 (6000 IOPS) | 6,000 | Mixed read/write CRM queries |
| MySQL CDR archives | st1 (HDD) | 500 | Old CDRs for compliance (cold) |

### AWS CLI Commands

```bash
# Kishore: Create high-IOPS volume for MySQL CDR in telephony3
aws ec2 create-volume \
  --volume-type gp3 --size 1000 --iops 10000 --throughput 500 \
  --availability-zone ap-south-1a --encrypted \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=tel3-mysql-cdr-data},{Key=Engine,Value=MySQL},{Key=ManagedBy,Value=Kishore}]' \
  --region ap-south-1

# Venkat: Create volume for MSSQL Billing in telephony1
aws ec2 create-volume \
  --volume-type gp3 --size 500 --iops 6000 --throughput 400 \
  --availability-zone us-east-1a --encrypted \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=tel1-mssql-billing-data},{Key=Engine,Value=MSSQL},{Key=ManagedBy,Value=Venkat}]' \
  --region us-east-1

# Meena: Check all volumes in telephony2
aws ec2 describe-volumes \
  --region eu-west-3 \
  --query 'Volumes[].{ID:VolumeId,Size:Size,Type:VolumeType,IOPS:Iops,State:State}' \
  --output table
```

---

## 2.3 EBS Snapshots — Telecom Backup Strategy

```bash
# Venkat: Snapshot MSSQL billing volume before month-end patching
aws ec2 create-snapshot \
  --volume-id vol-mssql-billing \
  --description "telephony1 MSSQL billing - pre-patch Feb 2025" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=tel1-mssql-prepatch},{Key=CreatedBy,Value=Venkat}]' \
  --region us-east-1

# Kishore: Copy MySQL CDR snapshot to telephony1 for DR
aws ec2 copy-snapshot \
  --source-region ap-south-1 \
  --source-snapshot-id snap-mysql-cdr \
  --destination-region us-east-1 \
  --description "telephony3 MySQL CDR DR copy to telephony1"

# Meena: List all snapshots in telephony2
aws ec2 describe-snapshots \
  --owner-ids self \
  --region eu-west-3 \
  --query 'Snapshots[].{ID:SnapshotId,Vol:VolumeId,Date:StartTime,Size:VolumeSize,Desc:Description}' \
  --output table
```

---

## 2.4 S3 for Telecom Database Backups

### S3 Bucket Strategy

```
  S3 BACKUP STRUCTURE:

  s3://telecom-db-backups-prod/
  ├── telephony1/
  │   ├── mssql/billing/full/          ← Venkat: SQL Server full backups
  │   ├── mssql/billing/diff/          ← Venkat: differential backups
  │   ├── mssql/billing/tlog/          ← Venkat: transaction log backups
  │   ├── mysql/cdr/full/              ← mysqldump full exports
  │   ├── mysql/cdr/binlog/            ← binary log backups
  │   ├── postgresql/crm/full/         ← pg_basebackup
  │   └── postgresql/crm/wal/          ← WAL archive files
  ├── telephony2/
  │   ├── mssql/billing/...            ← Meena manages
  │   ├── mysql/cdr/...
  │   └── postgresql/crm/...
  └── telephony3/
      ├── mssql/billing/...            ← Kishore manages
      ├── mysql/cdr/...
      └── postgresql/crm/...
```

### AWS CLI Commands

```bash
# Venkat: Create S3 bucket with encryption
aws s3 mb s3://telecom-db-backups-prod --region us-east-1
aws s3api put-bucket-encryption --bucket telecom-db-backups-prod \
  --server-side-encryption-configuration '{
    "Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"aws:kms"}}]}'

# Venkat: Upload MSSQL backup
aws s3 cp /backups/billing_full_20250219.bak \
  s3://telecom-db-backups-prod/telephony1/mssql/billing/full/

# Kishore: Upload MySQL CDR dump
aws s3 cp /backup/cdr_dump_20250219.sql.gz \
  s3://telecom-db-backups-prod/telephony3/mysql/cdr/full/

# Meena: Upload PostgreSQL backup
aws s3 cp /backup/crm_basebackup_20250219.tar.gz \
  s3://telecom-db-backups-prod/telephony2/postgresql/crm/full/

# Lifecycle: Move old backups to cheaper storage
aws s3api put-bucket-lifecycle-configuration \
  --bucket telecom-db-backups-prod \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "Telecom-Backup-Lifecycle",
      "Status": "Enabled",
      "Filter": {"Prefix": ""},
      "Transitions": [
        {"Days": 30, "StorageClass": "STANDARD_IA"},
        {"Days": 90, "StorageClass": "GLACIER_IR"},
        {"Days": 365, "StorageClass": "DEEP_ARCHIVE"}
      ],
      "Expiration": {"Days": 2555}
    }]}'

# Meena: Check backup sizes
aws s3 ls s3://telecom-db-backups-prod/telephony2/ --recursive --human-readable --summarize
```

---

## 2.5 AWS Backup for Telecom

```bash
# Venkat: Create centralized backup plan for all telecom databases
aws backup create-backup-vault --backup-vault-name telecom-prod-vault \
  --encryption-key-arn arn:aws:kms:us-east-1:123456789012:key/xxx

aws backup create-backup-plan --backup-plan '{
  "BackupPlanName": "Telecom-All-DB-Backup",
  "Rules": [
    {"RuleName": "Daily-2AM","TargetBackupVaultName": "telecom-prod-vault",
     "ScheduleExpression": "cron(0 2 * * ? *)",
     "Lifecycle": {"DeleteAfterDays": 35}},
    {"RuleName": "Weekly-Sunday","TargetBackupVaultName": "telecom-prod-vault",
     "ScheduleExpression": "cron(0 3 ? * SUN *)",
     "Lifecycle": {"MoveToColdStorageAfterDays": 30, "DeleteAfterDays": 365}}
  ]}'
```

---

# DAY 3 — AWS Managed Database Services

---

## 3.1 RDS Instance Creation — All 3 Engines

### MSSQL RDS — Billing System (Venkat creates)

```bash
# Venkat: Create MSSQL RDS for telephony1 Billing
aws rds create-db-instance \
  --db-instance-identifier prod-telephony1-mssql-billing \
  --db-instance-class db.r6i.4xlarge \
  --engine sqlserver-se \
  --engine-version 16.00 \
  --master-username billingadmin \
  --master-user-password 'Tel3com$ecure!2025' \
  --allocated-storage 500 \
  --storage-type gp3 --iops 6000 --storage-throughput 400 \
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
  --tags Key=Application,Value=Billing Key=Region,Value=telephony1 Key=ManagedBy,Value=Venkat \
  --region us-east-1
```

### MySQL RDS — CDR System (Kishore creates)

```bash
# Kishore: Create MySQL RDS for telephony3 CDR
aws rds create-db-instance \
  --db-instance-identifier prod-telephony3-mysql-cdr \
  --db-instance-class db.r6i.2xlarge \
  --engine mysql \
  --engine-version 8.0.36 \
  --master-username cdradmin \
  --master-user-password 'Tel3com$ecure!2025' \
  --allocated-storage 1000 \
  --storage-type gp3 --iops 10000 --storage-throughput 500 \
  --multi-az \
  --db-subnet-group-name telephony3-db-subnets \
  --vpc-security-group-ids sg-mysql-tel3 \
  --backup-retention-period 14 \
  --preferred-backup-window "02:00-03:00" \
  --preferred-maintenance-window "sun:04:00-sun:05:00" \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:ap-south-1:123456789012:key/xxx \
  --no-publicly-accessible \
  --tags Key=Application,Value=CDR Key=Region,Value=telephony3 Key=ManagedBy,Value=Kishore \
  --region ap-south-1
```

### PostgreSQL RDS — CRM System (Meena creates under Venkat's guidance)

```bash
# Meena: Create PostgreSQL RDS for telephony2 CRM
aws rds create-db-instance \
  --db-instance-identifier prod-telephony2-pg-crm \
  --db-instance-class db.r6i.2xlarge \
  --engine postgres \
  --engine-version 16.4 \
  --master-username crmadmin \
  --master-user-password 'Tel3com$ecure!2025' \
  --allocated-storage 500 \
  --storage-type gp3 --iops 6000 \
  --multi-az \
  --db-subnet-group-name telephony2-db-subnets \
  --vpc-security-group-ids sg-pg-tel2 \
  --backup-retention-period 14 \
  --preferred-backup-window "02:00-03:00" \
  --preferred-maintenance-window "sun:04:00-sun:05:00" \
  --storage-encrypted \
  --no-publicly-accessible \
  --tags Key=Application,Value=CRM Key=Region,Value=telephony2 Key=ManagedBy,Value=Meena \
  --region eu-west-3
```

### List All Telecom RDS Instances

```bash
# Meena: Daily check — list all instances across all regions
for region in us-east-1 eu-west-3 ap-south-1; do
  echo "=== $region ==="
  aws rds describe-db-instances --region $region \
    --query 'DBInstances[].{Name:DBInstanceIdentifier,Engine:Engine,Class:DBInstanceClass,Status:DBInstanceStatus,MultiAZ:MultiAZ,Storage:AllocatedStorage}' \
    --output table
done
```

---

## 3.2 RDS Backups, PITR & Snapshots — Telecom

```bash
# Venkat: Manual snapshot before MSSQL billing upgrade
aws rds create-db-snapshot \
  --db-instance-identifier prod-telephony1-mssql-billing \
  --db-snapshot-identifier tel1-mssql-billing-pre-upgrade-20250219 \
  --region us-east-1

# Kishore: PITR for MySQL CDR — restore to 2 minutes ago
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier prod-telephony3-mysql-cdr \
  --target-db-instance-identifier prod-telephony3-mysql-cdr-restored \
  --restore-time "2025-02-19T14:28:00Z" \
  --db-instance-class db.r6i.2xlarge \
  --db-subnet-group-name telephony3-db-subnets \
  --no-publicly-accessible \
  --region ap-south-1

# Meena: Copy PostgreSQL snapshot cross-region for DR
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier arn:aws:rds:eu-west-3:123456789012:snapshot:tel2-pg-crm-daily \
  --target-db-snapshot-identifier tel2-pg-crm-dr-copy \
  --region us-east-1

# Check latest restorable time for all databases
for db in prod-telephony1-mssql-billing prod-telephony1-mysql-cdr prod-telephony1-pg-crm; do
  echo "--- $db ---"
  aws rds describe-db-instances --db-instance-identifier $db \
    --query 'DBInstances[0].{Latest:LatestRestorableTime}' --region us-east-1
done
```

---

## 3.3 RDS Parameter Groups — All 3 Engines

### MSSQL Parameter Group (Venkat)

```bash
aws rds create-db-parameter-group \
  --db-parameter-group-name telecom-mssql-billing-params \
  --db-parameter-group-family sqlserver-se-16.0 \
  --description "Telecom Billing MSSQL params - Venkat"

aws rds modify-db-parameter-group \
  --db-parameter-group-name telecom-mssql-billing-params \
  --parameters \
    "ParameterName=cost threshold for parallelism,ParameterValue=50,ApplyMethod=immediate" \
    "ParameterName=max degree of parallelism,ParameterValue=4,ApplyMethod=immediate" \
    "ParameterName=optimize for ad hoc workloads,ParameterValue=1,ApplyMethod=immediate"
```

### MySQL Parameter Group (Kishore)

```bash
aws rds create-db-parameter-group \
  --db-parameter-group-name telecom-mysql-cdr-params \
  --db-parameter-group-family mysql8.0 \
  --description "Telecom CDR MySQL params - Kishore"

aws rds modify-db-parameter-group \
  --db-parameter-group-name telecom-mysql-cdr-params \
  --parameters \
    "ParameterName=innodb_buffer_pool_size,ParameterValue={DBInstanceClassMemory*3/4},ApplyMethod=pending-reboot" \
    "ParameterName=max_connections,ParameterValue=1000,ApplyMethod=pending-reboot" \
    "ParameterName=innodb_flush_log_at_trx_commit,ParameterValue=1,ApplyMethod=immediate" \
    "ParameterName=slow_query_log,ParameterValue=1,ApplyMethod=immediate" \
    "ParameterName=long_query_time,ParameterValue=2,ApplyMethod=immediate" \
    "ParameterName=innodb_io_capacity,ParameterValue=2000,ApplyMethod=immediate"
```

### PostgreSQL Parameter Group (Meena, guided by Venkat)

```bash
aws rds create-db-parameter-group \
  --db-parameter-group-name telecom-pg-crm-params \
  --db-parameter-group-family postgres16 \
  --description "Telecom CRM PostgreSQL params - Meena"

aws rds modify-db-parameter-group \
  --db-parameter-group-name telecom-pg-crm-params \
  --parameters \
    "ParameterName=shared_buffers,ParameterValue={DBInstanceClassMemory/4},ApplyMethod=pending-reboot" \
    "ParameterName=work_mem,ParameterValue=65536,ApplyMethod=immediate" \
    "ParameterName=maintenance_work_mem,ParameterValue=524288,ApplyMethod=immediate" \
    "ParameterName=max_connections,ParameterValue=500,ApplyMethod=pending-reboot" \
    "ParameterName=effective_cache_size,ParameterValue={DBInstanceClassMemory*3/4},ApplyMethod=pending-reboot" \
    "ParameterName=log_min_duration_statement,ParameterValue=1000,ApplyMethod=immediate" \
    "ParameterName=random_page_cost,ParameterValue=1.1,ApplyMethod=immediate"
```

---

## 3.4 RDS Security — Telecom Compliance

### Secrets Manager for All 3 Engines

```bash
# Venkat: Store MSSQL credentials
aws secretsmanager create-secret \
  --name telecom/telephony1/mssql/billing/admin \
  --secret-string '{"username":"billingadmin","password":"Tel3com$ecure!2025","engine":"sqlserver","host":"prod-telephony1-mssql-billing.xxx.us-east-1.rds.amazonaws.com","port":"1433"}'

# Kishore: Store MySQL credentials
aws secretsmanager create-secret \
  --name telecom/telephony3/mysql/cdr/admin \
  --secret-string '{"username":"cdradmin","password":"Tel3com$ecure!2025","engine":"mysql","host":"prod-telephony3-mysql-cdr.xxx.ap-south-1.rds.amazonaws.com","port":"3306"}'

# Meena: Store PostgreSQL credentials
aws secretsmanager create-secret \
  --name telecom/telephony2/postgresql/crm/admin \
  --secret-string '{"username":"crmadmin","password":"Tel3com$ecure!2025","engine":"postgres","host":"prod-telephony2-pg-crm.xxx.eu-west-3.rds.amazonaws.com","port":"5432"}'

# Enable rotation for all secrets
aws secretsmanager rotate-secret \
  --secret-id telecom/telephony1/mssql/billing/admin \
  --rotation-rules '{"AutomaticallyAfterDays": 30}'
```

---

## 3.5 Amazon Aurora — Telecom CRM Upgrade Path

```bash
# Venkat: Create Aurora PostgreSQL cluster for telephony1 CRM (upgraded from RDS)
aws rds create-db-cluster \
  --db-cluster-identifier tel1-aurora-pg-crm \
  --engine aurora-postgresql --engine-version 16.4 \
  --master-username crmadmin --master-user-password 'Tel3com$ecure!2025' \
  --db-subnet-group-name telephony1-db-subnets \
  --vpc-security-group-ids sg-pg-tel1 \
  --storage-encrypted --backup-retention-period 14 \
  --tags Key=Application,Value=CRM Key=ManagedBy,Value=Venkat

# Writer instance
aws rds create-db-instance \
  --db-instance-identifier tel1-aurora-pg-crm-writer \
  --db-cluster-identifier tel1-aurora-pg-crm \
  --db-instance-class db.r6g.2xlarge --engine aurora-postgresql

# Reader instance (for CRM reporting)
aws rds create-db-instance \
  --db-instance-identifier tel1-aurora-pg-crm-reader1 \
  --db-cluster-identifier tel1-aurora-pg-crm \
  --db-instance-class db.r6g.xlarge --engine aurora-postgresql
```

---

# DAY 4 — Monitoring, HA & DR

---

## 4.1 CloudWatch Alarms for All 3 Engines

```bash
# Create SNS topic for DBA team alerts
aws sns create-topic --name telecom-dba-alerts
aws sns subscribe --topic-arn arn:aws:sns:us-east-1:123456789012:telecom-dba-alerts \
  --protocol email --notification-endpoint venkat@telecom.com
aws sns subscribe --topic-arn arn:aws:sns:us-east-1:123456789012:telecom-dba-alerts \
  --protocol email --notification-endpoint kishore@telecom.com
aws sns subscribe --topic-arn arn:aws:sns:us-east-1:123456789012:telecom-dba-alerts \
  --protocol email --notification-endpoint meena@telecom.com

# MSSQL Billing — CPU alarm (Venkat)
aws cloudwatch put-metric-alarm \
  --alarm-name "Tel1-MSSQL-Billing-HighCPU" \
  --alarm-description "MSSQL Billing CPU > 80% - Alert Venkat" \
  --namespace AWS/RDS --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=prod-telephony1-mssql-billing \
  --statistic Average --period 300 --threshold 80 \
  --comparison-operator GreaterThanThreshold --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:telecom-dba-alerts

# MySQL CDR — Storage alarm (Kishore)
aws cloudwatch put-metric-alarm \
  --alarm-name "Tel3-MySQL-CDR-LowStorage" \
  --alarm-description "MySQL CDR storage < 50GB - Alert Kishore" \
  --namespace AWS/RDS --metric-name FreeStorageSpace \
  --dimensions Name=DBInstanceIdentifier,Value=prod-telephony3-mysql-cdr \
  --statistic Average --period 300 --threshold 53687091200 \
  --comparison-operator LessThanThreshold --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:ap-south-1:123456789012:telecom-dba-alerts

# PostgreSQL CRM — Connections alarm (Meena)
aws cloudwatch put-metric-alarm \
  --alarm-name "Tel2-PG-CRM-HighConnections" \
  --alarm-description "PostgreSQL CRM connections > 400 - Alert Meena" \
  --namespace AWS/RDS --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=prod-telephony2-pg-crm \
  --statistic Average --period 60 --threshold 400 \
  --comparison-operator GreaterThanThreshold --evaluation-periods 3 \
  --alarm-actions arn:aws:sns:eu-west-3:123456789012:telecom-dba-alerts

# MySQL CDR — Replica Lag alarm (Kishore)
aws cloudwatch put-metric-alarm \
  --alarm-name "Tel3-MySQL-CDR-ReplicaLag" \
  --alarm-description "MySQL CDR replica lag > 30sec - Alert Kishore" \
  --namespace AWS/RDS --metric-name ReplicaLag \
  --dimensions Name=DBInstanceIdentifier,Value=prod-telephony3-mysql-cdr-replica \
  --statistic Average --period 60 --threshold 30 \
  --comparison-operator GreaterThanThreshold --evaluation-periods 3 \
  --alarm-actions arn:aws:sns:ap-south-1:123456789012:telecom-dba-alerts
```

---

## 4.2 Performance Insights

```bash
# Enable PI on all telecom databases
for db in prod-telephony1-mssql-billing prod-telephony1-mysql-cdr prod-telephony1-pg-crm; do
  aws rds modify-db-instance --db-instance-identifier $db \
    --enable-performance-insights \
    --performance-insights-retention-period 731 \
    --apply-immediately --region us-east-1
done
```

---

## 4.3 High Availability & Failover

```bash
# Venkat: Test MSSQL billing failover (during maintenance window)
aws rds reboot-db-instance \
  --db-instance-identifier prod-telephony1-mssql-billing \
  --force-failover

# Kishore: Create MySQL read replica for CDR reporting
aws rds create-db-instance-read-replica \
  --db-instance-identifier prod-telephony3-mysql-cdr-replica \
  --source-db-instance-identifier prod-telephony3-mysql-cdr \
  --db-instance-class db.r6i.xlarge \
  --region ap-south-1

# Cross-region DR: Copy MySQL CDR replica to telephony1
aws rds create-db-instance-read-replica \
  --db-instance-identifier tel1-mysql-cdr-dr \
  --source-db-instance-identifier arn:aws:rds:ap-south-1:123456789012:db:prod-telephony3-mysql-cdr \
  --db-instance-class db.r6i.xlarge \
  --region us-east-1
```

---

## 4.4 Disaster Recovery — Telecom DR Strategy

```
  TELECOM DR STRATEGY:

  ┌───────────────────────────────────────────────────────────────┐
  │  DB Engine    │ DR Method           │ RPO    │ RTO    │ Owner │
  ├───────────────┼─────────────────────┼────────┼────────┼───────┤
  │ MSSQL Billing │ Cross-region snap   │ 1 hour │ 2 hour │ Venkat│
  │ MySQL CDR     │ Cross-region replica│ ~1 sec │ 5 min  │Kishore│
  │ PostgreSQL CRM│ Aurora Global DB    │ ~1 sec │ 1 min  │ Venkat│
  └───────────────────────────────────────────────────────────────┘
```

---

## 4.5 DBA Troubleshooting

```bash
# Meena: Morning health check for all regions
for region in us-east-1 eu-west-3 ap-south-1; do
  echo "=== REGION: $region ==="
  aws rds describe-db-instances --region $region \
    --query 'DBInstances[].{Name:DBInstanceIdentifier,Status:DBInstanceStatus,Engine:Engine,MultiAZ:MultiAZ}' \
    --output table
  aws rds describe-events --duration 720 --region $region \
    --query 'Events[].{Source:SourceIdentifier,Message:Message}' --output table
  aws cloudwatch describe-alarms --state-value ALARM --region $region \
    --query 'MetricAlarms[].{Alarm:AlarmName,State:StateValue}' --output table
  echo ""
done
```

---

# DAY 5 — Security, Automation & Cost

---

## 5.1 KMS & Secrets Manager

```bash
# Venkat: Create KMS keys for each region
for region in us-east-1 eu-west-3 ap-south-1; do
  aws kms create-key --description "Telecom DB Encryption - $region" \
    --key-usage ENCRYPT_DECRYPT --region $region
done

# List all secrets
aws secretsmanager list-secrets \
  --query 'SecretList[?starts_with(Name,`telecom/`)].{Name:Name,Rotated:LastRotatedDate}' \
  --output table
```

---

## 5.2 AWS CLI DBA Cheat Sheet

```bash
# ═══ ALL TEAM MEMBERS: Daily Commands ═══

# List all RDS instances
aws rds describe-db-instances --output table

# Manual snapshot
aws rds create-db-snapshot --db-instance-identifier INSTANCE_NAME \
  --db-snapshot-identifier "manual-$(date +%Y%m%d-%H%M)"

# Download error logs
# MSSQL:
aws rds download-db-log-file-portion --db-instance-identifier prod-telephony1-mssql-billing \
  --log-file-name error/sqlserver-error.log --output text
# MySQL:
aws rds download-db-log-file-portion --db-instance-identifier prod-telephony3-mysql-cdr \
  --log-file-name error/mysql-error.log --output text
# PostgreSQL:
aws rds download-db-log-file-portion --db-instance-identifier prod-telephony2-pg-crm \
  --log-file-name error/postgresql.log --output text

# Check events
aws rds describe-events --duration 60

# Scale up (Venkat/Kishore only)
aws rds modify-db-instance --db-instance-identifier INSTANCE_NAME \
  --db-instance-class db.r6i.4xlarge --apply-immediately
```

---

## 5.3 Automation & Cost

```bash
# Meena: Stop dev/test instances at 7 PM
for db in dev-telephony1-mssql dev-telephony1-mysql dev-telephony1-pg; do
  aws rds stop-db-instance --db-instance-identifier $db --region us-east-1
done

# Meena: Start dev/test instances at 8 AM
for db in dev-telephony1-mssql dev-telephony1-mysql dev-telephony1-pg; do
  aws rds start-db-instance --db-instance-identifier $db --region us-east-1
done

# Venkat: Check monthly RDS cost
aws ce get-cost-and-usage \
  --time-period Start=$(date -d "first day of this month" +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY --metrics BlendedCost \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Relational Database Service"]}}' \
  --output table
```

---

## 5.4 Telecom DBA Operating Model

### Daily Routine

| Time | Task | Who | Details |
|------|------|-----|---------|
| 8:00 AM | Morning Health Check | Meena | Run health check script across all 3 regions |
| 8:30 AM | Review Alarms | Meena | Check CloudWatch alarms, escalate to Venkat/Kishore |
| 9:00 AM | Backup Verification | Meena | Verify overnight automated backups completed |
| 9:30 AM | Performance Review | Venkat/Kishore | Check Performance Insights, top SQL across all engines |
| Ongoing | Ticket Resolution | All | Schema changes, user access, performance issues |

### Weekly Routine

| Task | Who | Details |
|------|-----|---------|
| Patching Review | Venkat | Review pending patches for MSSQL, MySQL, PostgreSQL |
| DR Snapshot Verification | Kishore | Verify cross-region snapshots exist and are current |
| Failover Test (non-prod) | Venkat | Test Multi-AZ failover on dev instances |
| Security Audit | Kishore | Review security group rules, IAM policies |

### Monthly Routine

| Task | Who | Details |
|------|-----|---------|
| Right-sizing Review | Venkat | Analyze CPU/memory usage, resize if needed |
| Cost Report | Meena | Generate RDS cost breakdown by engine/region |
| Snapshot Cleanup | Meena | Delete manual snapshots older than 90 days |
| Password Rotation | Kishore | Verify Secrets Manager auto-rotation working |

---

## Summary — Telecom Database Inventory

| Region | Instance Name | Engine | Class | Storage | Multi-AZ | Owner |
|--------|-------------|--------|-------|---------|----------|-------|
| telephony1 | prod-telephony1-mssql-billing | SQL Server SE 2022 | db.r6i.4xlarge | 500 GB gp3 | Yes | Venkat |
| telephony1 | prod-telephony1-mysql-cdr | MySQL 8.0 | db.r6i.2xlarge | 1 TB gp3 | Yes | Venkat |
| telephony1 | prod-telephony1-pg-crm | PostgreSQL 16 | db.r6i.2xlarge | 500 GB gp3 | Yes | Venkat |
| telephony2 | prod-telephony2-mssql-billing | SQL Server SE 2022 | db.r6i.4xlarge | 500 GB gp3 | Yes | Meena |
| telephony2 | prod-telephony2-mysql-cdr | MySQL 8.0 | db.r6i.2xlarge | 1 TB gp3 | Yes | Meena |
| telephony2 | prod-telephony2-pg-crm | PostgreSQL 16 | db.r6i.2xlarge | 500 GB gp3 | Yes | Meena |
| telephony3 | prod-telephony3-mssql-billing | SQL Server SE 2022 | db.r6i.4xlarge | 500 GB gp3 | Yes | Kishore |
| telephony3 | prod-telephony3-mysql-cdr | MySQL 8.0 | db.r6i.2xlarge | 1 TB gp3 | Yes | Kishore |
| telephony3 | prod-telephony3-pg-crm | PostgreSQL 16 | db.r6i.2xlarge | 500 GB gp3 | Yes | Kishore |

---

*5-Day AWS Cloud DBA Training — Telecom Domain — MSSQL, MySQL, PostgreSQL*
*DBA Team: Venkat (Sr), Kishore (Sr), Meena (Jr)*
