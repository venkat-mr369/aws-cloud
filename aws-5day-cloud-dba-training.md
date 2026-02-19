# 5-Day AWS Training Program for Cloud DBAs

> **Audience:** Oracle & SQL Server DBAs working on AWS Cloud
> **Objective:** Enable DBAs to operate, secure, monitor, and support databases on AWS production environments.
> **Outcome:** DBAs will be cloud-ready to manage AWS-hosted databases with confidence in security, availability, monitoring, and cost control.

---

# DAY 1 — AWS Foundations for DBAs

---

## 1.1 AWS Global Infrastructure (Regions, AZs, Multi-AZ)

### What is AWS Global Infrastructure?

AWS runs data centers across the world. They are organized into three levels:

```
  AWS Global Infrastructure Hierarchy

  ┌─────────────────────────────────────────────────────┐
  │                    AWS CLOUD                         │
  │                                                     │
  │  ┌─── REGION (e.g., us-east-1 / N. Virginia) ───┐  │
  │  │                                               │  │
  │  │  ┌─── AZ: us-east-1a ───┐                    │  │
  │  │  │  Data Center 1        │                    │  │
  │  │  │  Data Center 2        │                    │  │
  │  │  └───────────────────────┘                    │  │
  │  │                                               │  │
  │  │  ┌─── AZ: us-east-1b ───┐                    │  │
  │  │  │  Data Center 3        │                    │  │
  │  │  │  Data Center 4        │                    │  │
  │  │  └───────────────────────┘                    │  │
  │  │                                               │  │
  │  │  ┌─── AZ: us-east-1c ───┐                    │  │
  │  │  │  Data Center 5        │                    │  │
  │  │  └───────────────────────┘                    │  │
  │  └───────────────────────────────────────────────┘  │
  │                                                     │
  │  ┌─── REGION (e.g., ap-south-1 / Mumbai) ────────┐  │
  │  │  AZ: ap-south-1a | ap-south-1b | ap-south-1c  │  │
  │  └────────────────────────────────────────────────┘  │
  └─────────────────────────────────────────────────────┘
```

### Key Concepts

| Concept | Definition | DBA Relevance |
|---------|-----------|---------------|
| **Region** | A geographic area with 2+ AZs (e.g., us-east-1, eu-west-3) | Choose region closest to your users for low latency |
| **Availability Zone (AZ)** | One or more isolated data centers in a region with independent power, cooling, and networking | Multi-AZ deployments protect against AZ failure |
| **Multi-AZ** | Deploying resources across 2+ AZs for high availability | RDS Multi-AZ = automatic failover if primary AZ fails |
| **Edge Locations** | CDN endpoints for CloudFront (not directly used by DBAs) | Cache static content closer to users |

### AWS CLI Commands

```bash
# List all available regions
aws ec2 describe-regions --query 'Regions[].RegionName' --output table

# List all AZs in your current region
aws ec2 describe-availability-zones --query 'AvailabilityZones[].ZoneName' --output table

# List AZs in a specific region
aws ec2 describe-availability-zones \
  --region ap-south-1 \
  --query 'AvailabilityZones[].{Zone:ZoneName,State:State}' \
  --output table
```

### Example: Why Multi-AZ Matters for DBAs

```
  WITHOUT Multi-AZ:                    WITH Multi-AZ (RDS):
  
  ┌──────────┐                         ┌──────────┐    ┌──────────┐
  │ RDS DB   │                         │ PRIMARY  │───►│ STANDBY  │
  │ us-east-1a                         │ us-east-1a    │ us-east-1b│
  └──────────┘                         └──────────┘    └──────────┘
       │                                    │               │
  AZ-a goes down                       AZ-a goes down      │
       │                                    │               │
     💀 DB DOWN!                         Automatic      ◄───┘
     Manual recovery                    Failover!
     needed                             (~60 seconds)
```

---

## 1.2 AWS Account Model & Shared Responsibility

### Shared Responsibility Model

```
  ┌──────────────────────────────────────────────────────────┐
  │           CUSTOMER RESPONSIBILITY ("IN the cloud")       │
  │                                                          │
  │  EC2-hosted DB:                  RDS Managed DB:         │
  │  • OS patching                   • DB user management    │
  │  • DB installation               • Query optimization    │
  │  • DB patching                   • Schema design         │
  │  • Backups                       • Parameter groups      │
  │  • HA configuration              • Monitoring setup      │
  │  • Firewall rules                • Security groups       │
  │  • Encryption setup              • Encryption (KMS keys) │
  │                                                          │
  ├──────────────────────────────────────────────────────────┤
  │           AWS RESPONSIBILITY ("OF the cloud")            │
  │                                                          │
  │  EC2-hosted DB:                  RDS Managed DB:         │
  │  • Physical hardware             • All of EC2 items PLUS │
  │  • Network infrastructure        • OS patching           │
  │  • Hypervisor                    • DB engine patching    │
  │  • Data center security          • Automated backups     │
  │  • Power & cooling               • HA/failover infra     │
  │                                  • Storage management    │
  └──────────────────────────────────────────────────────────┘
```

### DBA Key Takeaway

| On EC2 (Self-Managed) | On RDS (AWS Managed) |
|---|---|
| You install Oracle/SQL Server | AWS provisions the engine |
| You apply patches | AWS applies engine patches (you choose the window) |
| You configure backups | AWS does automated backups (you configure retention) |
| You set up replication/HA | AWS provides Multi-AZ failover |
| You manage OS | AWS manages OS completely |
| **Full control, more work** | **Less control, less work** |

---

## 1.3 IAM Basics for DBAs (Users, Roles, Policies)

### What is IAM?

IAM (Identity and Access Management) controls WHO can do WHAT on your AWS resources.

```
  IAM Structure for a DBA Team:

  ┌────────────────────────────────────────────────┐
  │                  AWS ACCOUNT                    │
  │                                                │
  │  ┌─── IAM GROUP: "DBA-Team" ──────────────┐   │
  │  │                                         │   │
  │  │  IAM User: john.dba                     │   │
  │  │  IAM User: priya.dba                    │   │
  │  │  IAM User: pierre.dba                   │   │
  │  │                                         │   │
  │  │  Attached Policy: AmazonRDSFullAccess   │   │
  │  │  Attached Policy: CloudWatchReadOnly     │   │
  │  │  Attached Policy: CustomDBBackupPolicy   │   │
  │  └─────────────────────────────────────────┘   │
  │                                                │
  │  ┌─── IAM ROLE: "RDS-Monitoring-Role" ────┐   │
  │  │  Trust: monitoring.rds.amazonaws.com    │   │
  │  │  Policy: CloudWatchLogsFullAccess       │   │
  │  └─────────────────────────────────────────┘   │
  └────────────────────────────────────────────────┘
```

### Key IAM Concepts

| Concept | What It Is | DBA Example |
|---------|-----------|-------------|
| **User** | A person or application with credentials | john.dba who manages RDS instances |
| **Group** | Collection of users with shared permissions | DBA-Team group with RDS access |
| **Role** | Temporary permissions assumed by services/users | EC2 instance role to access S3 backups |
| **Policy** | JSON document defining permissions | Allow: rds:CreateDBSnapshot |
| **MFA** | Multi-Factor Authentication | Required for production RDS access |

### AWS CLI Commands

```bash
# Create a DBA user
aws iam create-user --user-name john.dba

# Create DBA group
aws iam create-group --group-name DBA-Team

# Add user to group
aws iam add-user-to-group --user-name john.dba --group-name DBA-Team

# Attach RDS full access to the DBA group
aws iam attach-group-policy \
  --group-name DBA-Team \
  --policy-arn arn:aws:iam::aws:policy/AmazonRDSFullAccess

# Attach CloudWatch read-only for monitoring
aws iam attach-group-policy \
  --group-name DBA-Team \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchReadOnlyAccess

# Create access keys for CLI usage
aws iam create-access-key --user-name john.dba

# List all users in the DBA group
aws iam get-group --group-name DBA-Team \
  --query 'Users[].UserName' --output table
```

### Custom DBA Policy Example

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowRDSManagement",
      "Effect": "Allow",
      "Action": [
        "rds:Describe*",
        "rds:CreateDBSnapshot",
        "rds:RestoreDBInstanceFromDBSnapshot",
        "rds:RebootDBInstance",
        "rds:ModifyDBInstance"
      ],
      "Resource": "arn:aws:rds:us-east-1:123456789012:db:*"
    },
    {
      "Sid": "DenyDeleteProduction",
      "Effect": "Deny",
      "Action": [
        "rds:DeleteDBInstance",
        "rds:DeleteDBCluster"
      ],
      "Resource": "arn:aws:rds:*:*:db:prod-*"
    }
  ]
}
```

```bash
# Create custom policy from JSON file
aws iam create-policy \
  --policy-name DBA-RDS-Policy \
  --policy-document file://dba-policy.json
```

---

## 1.4 VPC Concepts (Private/Public Subnets)

### What is a VPC?

A VPC (Virtual Private Cloud) is your isolated network in AWS — like having your own private data center network inside AWS.

```
  VPC Architecture for Database Workloads

  ┌──────────────── VPC: 10.0.0.0/16 ─────────────────┐
  │                                                     │
  │  ┌─── PUBLIC SUBNET (10.0.100.0/24) ─────────┐     │
  │  │  • Bastion Host (SSH jump box)              │     │
  │  │  • NAT Gateway                              │     │
  │  │  • Load Balancers                           │     │
  │  │  Has route to Internet Gateway              │     │
  │  └─────────────────────────────────────────────┘     │
  │                    │                                  │
  │                NAT Gateway                           │
  │                    │                                  │
  │  ┌─── PRIVATE SUBNET 1 (10.0.1.0/24) AZ-a ───┐     │
  │  │  • RDS Primary Instance                     │     │
  │  │  • EC2 Oracle Database                      │     │
  │  │  NO direct internet access                  │     │
  │  │  Internet via NAT Gateway only              │     │
  │  └─────────────────────────────────────────────┘     │
  │                                                     │
  │  ┌─── PRIVATE SUBNET 2 (10.0.2.0/24) AZ-b ───┐     │
  │  │  • RDS Standby Instance (Multi-AZ)          │     │
  │  │  • Read Replicas                            │     │
  │  └─────────────────────────────────────────────┘     │
  │                                                     │
  │  ┌─── PRIVATE SUBNET 3 (10.0.3.0/24) AZ-c ───┐     │
  │  │  • Additional replicas                      │     │
  │  │  • DR instances                             │     │
  │  └─────────────────────────────────────────────┘     │
  └─────────────────────────────────────────────────────┘
```

### Public vs Private Subnet

| Feature | Public Subnet | Private Subnet |
|---------|--------------|----------------|
| **Internet access** | Direct (via Internet Gateway) | Via NAT Gateway only |
| **Public IP** | Auto-assigned | No public IP |
| **Use for DBAs** | Bastion host, NAT GW | Databases, RDS, application servers |
| **Security** | More exposed | More secure (no direct internet) |

### AWS CLI Commands

```bash
# Create VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16
# Copy VpcId → vpc-xxx

# Enable DNS hostnames (required for RDS)
aws ec2 modify-vpc-attribute --vpc-id vpc-xxx --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id vpc-xxx --enable-dns-support

# Create private subnets for databases (in different AZs for Multi-AZ)
aws ec2 create-subnet --vpc-id vpc-xxx --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a
aws ec2 create-subnet --vpc-id vpc-xxx --cidr-block 10.0.2.0/24 \
  --availability-zone us-east-1b

# Create DB Subnet Group (required for RDS)
aws rds create-db-subnet-group \
  --db-subnet-group-name my-db-subnets \
  --db-subnet-group-description "Subnets for RDS databases" \
  --subnet-ids subnet-aaa111 subnet-bbb222

# List DB subnet groups
aws rds describe-db-subnet-groups \
  --query 'DBSubnetGroups[].{Name:DBSubnetGroupName,VPC:VpcId,Subnets:Subnets[].SubnetAvailabilityZone.Name}' \
  --output table
```

> **DBA Key Point:** RDS requires a **DB Subnet Group** with subnets in at least 2 different AZs. This is mandatory even for single-AZ deployments!

---

## 1.5 Security Groups & Network ACLs

### Security Groups (Stateful Firewall)

Think of Security Groups as a firewall attached directly to your database instance.

```
  Security Group: db-sg (attached to RDS instance)

  INBOUND RULES:
  ┌──────────┬──────────┬──────────────────────────────┐
  │ Protocol │ Port     │ Source                       │
  ├──────────┼──────────┼──────────────────────────────┤
  │ TCP      │ 1433     │ app-sg (SQL Server)          │
  │ TCP      │ 1521     │ app-sg (Oracle)              │
  │ TCP      │ 3306     │ app-sg (MySQL/Aurora)        │
  │ TCP      │ 5432     │ app-sg (PostgreSQL)          │
  │ TCP      │ 1433     │ 10.0.100.0/24 (Bastion)     │
  └──────────┴──────────┴──────────────────────────────┘

  OUTBOUND RULES:
  ┌──────────┬──────────┬──────────────────────────────┐
  │ Protocol │ Port     │ Destination                  │
  ├──────────┼──────────┼──────────────────────────────┤
  │ All      │ All      │ 0.0.0.0/0 (default allow)    │
  └──────────┴──────────┴──────────────────────────────┘
```

### Security Groups vs NACLs

| Feature | Security Group | Network ACL |
|---------|---------------|-------------|
| **Level** | Instance level (RDS, EC2) | Subnet level |
| **Stateful/Stateless** | Stateful (return traffic auto-allowed) | Stateless (must allow both directions) |
| **Rules** | Allow rules only | Allow AND Deny rules |
| **Evaluation** | All rules evaluated together | Rules evaluated in order (by number) |
| **Default** | Denies all inbound, allows all outbound | Allows all inbound and outbound |
| **DBA Use** | Primary security control for DB access | Secondary defense layer |

### AWS CLI Commands

```bash
# Create security group for databases
aws ec2 create-security-group \
  --group-name db-sg \
  --description "Database Security Group" \
  --vpc-id vpc-xxx
# Copy GroupId → sg-dbxxx

# Allow SQL Server (1433) from application security group
aws ec2 authorize-security-group-ingress \
  --group-id sg-dbxxx \
  --protocol tcp \
  --port 1433 \
  --source-group sg-appxxx

# Allow PostgreSQL (5432) from application subnet
aws ec2 authorize-security-group-ingress \
  --group-id sg-dbxxx \
  --protocol tcp \
  --port 5432 \
  --cidr 10.0.10.0/24

# Allow Oracle (1521) from bastion host
aws ec2 authorize-security-group-ingress \
  --group-id sg-dbxxx \
  --protocol tcp \
  --port 1521 \
  --cidr 10.0.100.5/32

# View all rules in a security group
aws ec2 describe-security-groups \
  --group-ids sg-dbxxx \
  --query 'SecurityGroups[].IpPermissions[]' \
  --output table

# Remove a rule (revoke)
aws ec2 revoke-security-group-ingress \
  --group-id sg-dbxxx \
  --protocol tcp \
  --port 3306 \
  --cidr 0.0.0.0/0
```

> **DBA Best Practice:** Never open database ports (1433, 1521, 3306, 5432) to 0.0.0.0/0! Always restrict to specific application security groups or private subnet CIDRs.

---

---

# DAY 2 — Compute & Storage for Databases

---

## 2.1 EC2 for Oracle/SQL Databases

### When to Use EC2 vs RDS

```
  DECISION: EC2 or RDS?

  ┌────────────────────────────────────────────────────────┐
  │ Use EC2 when you need:          Use RDS when:          │
  │                                                        │
  │ • Full OS access                • You want managed     │
  │ • Custom Oracle RAC               service              │
  │ • SQL Server FCI (shared disk)  • Standard HA is enough│
  │ • Specific OS tuning            • You want auto-backups│
  │ • Non-standard DB configs       • Patching automated   │
  │ • Oracle Data Guard manual      • Quick provisioning   │
  │ • 3rd party backup tools        • Read replicas easy   │
  │                                                        │
  │ DBA manages: OS + DB + HA       DBA manages: DB only   │
  └────────────────────────────────────────────────────────┘
```

### Choosing EC2 Instance Types for Databases

| Instance Family | Best For | Example | vCPU | RAM | Network |
|----------------|----------|---------|------|-----|---------|
| **r6i / r7i** | Memory-optimized (most DBs) | r6i.2xlarge | 8 | 64 GB | Up to 12.5 Gbps |
| **r6g** | ARM-based, cost-effective | r6g.xlarge | 4 | 32 GB | Up to 10 Gbps |
| **m6i** | General purpose (smaller DBs) | m6i.xlarge | 4 | 16 GB | Up to 12.5 Gbps |
| **i3 / i4i** | Storage-optimized (high IOPS) | i3.2xlarge | 8 | 61 GB | Up to 10 Gbps |
| **x2idn** | Very large Oracle/SQL Server | x2idn.xlarge | 4 | 128 GB | Up to 25 Gbps |

### AWS CLI Commands

```bash
# Launch an EC2 instance for SQL Server
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type r6i.2xlarge \
  --key-name dba-key \
  --subnet-id subnet-private1 \
  --security-group-ids sg-dbxxx \
  --block-device-mappings '[
    {"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":100,"VolumeType":"gp3"}},
    {"DeviceName":"/dev/sdb","Ebs":{"VolumeSize":500,"VolumeType":"gp3","Iops":6000,"Throughput":400}},
    {"DeviceName":"/dev/sdc","Ebs":{"VolumeSize":200,"VolumeType":"gp3","Iops":3000}}
  ]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=sql-server-prod}]'

# Check instance status
aws ec2 describe-instance-status --instance-ids i-xxx

# Stop/Start instance (for maintenance)
aws ec2 stop-instances --instance-ids i-xxx
aws ec2 start-instances --instance-ids i-xxx

# Get Windows Administrator password (SQL Server on Windows)
aws ec2 get-password-data --instance-id i-xxx --priv-launch-key dba-key.pem
```

### Best Disk Layout for DB on EC2

```
  Recommended EBS Volume Layout:

  C: / /dev/sda1 (100 GB gp3)  → OS + SQL Server binaries
  D: / /dev/sdb  (500 GB gp3)  → Database data files (.mdf/.ndf)
  E: / /dev/sdc  (200 GB gp3)  → Transaction log files (.ldf)
  F: / /dev/sdd  (100 GB gp3)  → TempDB
  G: / /dev/sde  (500 GB gp3)  → Backups (or use S3)

  For Oracle:
  /u01  (100 GB gp3)  → Oracle binaries
  /u02  (500 GB gp3)  → Datafiles
  /u03  (200 GB gp3)  → Redo logs
  /u04  (100 GB gp3)  → Archive logs
```

---

## 2.2 EBS Volume Types & Performance

### EBS Volume Comparison

| Volume Type | Use Case | Max IOPS | Max Throughput | Cost |
|------------|----------|----------|---------------|------|
| **gp3** (General Purpose) | Most databases, dev/test | 16,000 | 1,000 MB/s | Cheapest SSD |
| **gp2** (General Purpose) | Legacy, use gp3 instead | 16,000 | 250 MB/s | More expensive than gp3 |
| **io2** (Provisioned IOPS) | Critical OLTP, Oracle RAC | 64,000 | 1,000 MB/s | Expensive, guaranteed IOPS |
| **io2 Block Express** | Largest databases | 256,000 | 4,000 MB/s | Most expensive |
| **st1** (Throughput HDD) | Log archives, data warehouse | 500 | 500 MB/s | Cheap HDD |

### gp3 vs io2 Decision for DBAs

```
  ┌────────────────────────────────────────────────────────┐
  │  Need < 16,000 IOPS?  →  Use gp3 (cost effective)     │
  │  Need > 16,000 IOPS?  →  Use io2 (guaranteed IOPS)    │
  │  Need > 64,000 IOPS?  →  Use io2 Block Express        │
  │                                                        │
  │  gp3 DEFAULT: 3,000 IOPS + 125 MB/s (FREE baseline)   │
  │  gp3 MAX:    16,000 IOPS + 1,000 MB/s (extra cost)    │
  │                                                        │
  │  💡 TIP: gp3 lets you set IOPS independently of size!  │
  │  A 100 GB gp3 can have 16,000 IOPS.                   │
  │  A 100 GB gp2 only gets 300 IOPS (3 IOPS/GB).         │
  └────────────────────────────────────────────────────────┘
```

### AWS CLI Commands

```bash
# Create a gp3 volume with custom IOPS
aws ec2 create-volume \
  --volume-type gp3 \
  --size 500 \
  --iops 6000 \
  --throughput 400 \
  --availability-zone us-east-1a \
  --encrypted \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=sql-data-vol}]'

# Attach volume to instance
aws ec2 attach-volume \
  --volume-id vol-xxx \
  --instance-id i-xxx \
  --device /dev/sdb

# Modify volume (increase size or IOPS — NO downtime!)
aws ec2 modify-volume \
  --volume-id vol-xxx \
  --size 1000 \
  --iops 10000 \
  --throughput 500

# Check modification progress
aws ec2 describe-volumes-modifications --volume-ids vol-xxx

# List all volumes with details
aws ec2 describe-volumes \
  --query 'Volumes[].{ID:VolumeId,Size:Size,Type:VolumeType,IOPS:Iops,State:State,AZ:AvailabilityZone}' \
  --output table
```

---

## 2.3 EBS Snapshots & Restore

### How EBS Snapshots Work

```
  EBS Snapshot Lifecycle:

  ┌──────────┐     Snapshot      ┌──────────┐
  │  EBS     │ ────────────────► │   S3     │
  │  Volume  │   (incremental)   │ (stored) │
  │  500 GB  │                   │          │
  └──────────┘                   └──────────┘
                                      │
                                Restore │ Copy to
                                      │ another region
                                      ▼
                                ┌──────────┐
                                │  New EBS │
                                │  Volume  │
                                └──────────┘

  KEY POINTS:
  • First snapshot = full copy
  • Subsequent snapshots = incremental (only changed blocks)
  • Snapshots are stored in S3 (managed by AWS, you don't see them)
  • You can copy snapshots cross-region for DR
  • Restore creates a new volume from snapshot
```

### AWS CLI Commands

```bash
# Create snapshot of a database volume
aws ec2 create-snapshot \
  --volume-id vol-xxx \
  --description "SQL Server data vol - before patching" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=sql-data-pre-patch},{Key=Environment,Value=prod}]'

# List snapshots for a volume
aws ec2 describe-snapshots \
  --filters "Name=volume-id,Values=vol-xxx" \
  --query 'Snapshots[].{ID:SnapshotId,Date:StartTime,Size:VolumeSize,State:State,Desc:Description}' \
  --output table

# Create volume from snapshot (restore)
aws ec2 create-volume \
  --snapshot-id snap-xxx \
  --volume-type gp3 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=sql-data-restored}]'

# Copy snapshot to another region (for DR)
aws ec2 copy-snapshot \
  --source-region us-east-1 \
  --source-snapshot-id snap-xxx \
  --destination-region eu-west-3 \
  --description "DR copy of SQL data volume"

# Delete old snapshot
aws ec2 delete-snapshot --snapshot-id snap-xxx
```

---

## 2.4 S3 for DB Backups

### Using S3 for Database Backups

```
  S3 Storage Classes for DB Backups:

  ┌─────────────────────┬─────────────────────────────────────┐
  │ Storage Class        │ DBA Use Case                        │
  ├─────────────────────┼─────────────────────────────────────┤
  │ S3 Standard          │ Recent backups (last 7 days)        │
  │ S3 Standard-IA       │ Weekly backups (30-90 days)         │
  │ S3 Glacier Instant   │ Monthly backups (quick restore)     │
  │ S3 Glacier Flexible  │ Yearly backups (12hr restore OK)    │
  │ S3 Glacier Deep      │ Compliance archives (7+ years)      │
  └─────────────────────┴─────────────────────────────────────┘
```

### AWS CLI Commands

```bash
# Create S3 bucket for backups
aws s3 mb s3://company-db-backups-prod --region us-east-1

# Enable versioning (protect against accidental delete)
aws s3api put-bucket-versioning \
  --bucket company-db-backups-prod \
  --versioning-configuration Status=Enabled

# Enable server-side encryption (SSE-S3)
aws s3api put-bucket-encryption \
  --bucket company-db-backups-prod \
  --server-side-encryption-configuration '{
    "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "aws:kms"}}]
  }'

# Upload SQL Server backup to S3
aws s3 cp /backups/proddb_full_20250219.bak \
  s3://company-db-backups-prod/sqlserver/prod/full/

# Upload Oracle RMAN backup
aws s3 sync /u04/backup/rman/ \
  s3://company-db-backups-prod/oracle/prod/rman/

# Download backup for restore
aws s3 cp s3://company-db-backups-prod/sqlserver/prod/full/proddb_full_20250219.bak \
  /restore/

# List backups
aws s3 ls s3://company-db-backups-prod/sqlserver/prod/full/ --human-readable

# Set lifecycle policy (auto-move old backups to cheaper storage)
aws s3api put-bucket-lifecycle-configuration \
  --bucket company-db-backups-prod \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "DB-Backup-Lifecycle",
      "Status": "Enabled",
      "Filter": {"Prefix": ""},
      "Transitions": [
        {"Days": 30, "StorageClass": "STANDARD_IA"},
        {"Days": 90, "StorageClass": "GLACIER_IR"},
        {"Days": 365, "StorageClass": "DEEP_ARCHIVE"}
      ],
      "Expiration": {"Days": 2555}
    }]
  }'

# RDS: Export snapshot to S3
aws rds start-export-task \
  --export-task-identifier export-prod-db \
  --source-arn arn:aws:rds:us-east-1:123456789012:snapshot:prod-db-snapshot \
  --s3-bucket-name company-db-backups-prod \
  --iam-role-arn arn:aws:iam::123456789012:role/rds-s3-export-role \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/xxx
```

---

## 2.5 AWS Backup Overview

### What is AWS Backup?

A centralized backup service that automates backups across multiple AWS services (RDS, EBS, EC2, S3, DynamoDB, etc.) using a single backup plan.

```
  AWS Backup Architecture:

  ┌──────────────────────────────────────────────┐
  │              AWS BACKUP SERVICE               │
  │                                              │
  │  ┌──────────────┐    ┌──────────────────┐    │
  │  │ Backup Plan  │───►│ Backup Vault     │    │
  │  │              │    │ (encrypted store) │    │
  │  │ Schedule:    │    │                  │    │
  │  │ Daily 2 AM   │    │ Keeps backups    │    │
  │  │ Weekly Sun   │    │ per retention    │    │
  │  │ Monthly 1st  │    │ rules            │    │
  │  └──────────────┘    └──────────────────┘    │
  │         │                                    │
  │  Protects:                                   │
  │  ├── RDS instances                           │
  │  ├── Aurora clusters                         │
  │  ├── EBS volumes                             │
  │  ├── EC2 instances                           │
  │  └── DynamoDB tables                         │
  └──────────────────────────────────────────────┘
```

### AWS CLI Commands

```bash
# Create backup vault
aws backup create-backup-vault \
  --backup-vault-name dba-prod-vault \
  --encryption-key-arn arn:aws:kms:us-east-1:123456789012:key/xxx

# Create backup plan
aws backup create-backup-plan --backup-plan '{
  "BackupPlanName": "DBA-Daily-Backup",
  "Rules": [{
    "RuleName": "DailyBackup",
    "TargetBackupVaultName": "dba-prod-vault",
    "ScheduleExpression": "cron(0 2 * * ? *)",
    "StartWindowMinutes": 60,
    "CompletionWindowMinutes": 180,
    "Lifecycle": {
      "MoveToColdStorageAfterDays": 30,
      "DeleteAfterDays": 365
    }
  }]
}'

# Assign RDS instance to backup plan
aws backup create-backup-selection \
  --backup-plan-id plan-xxx \
  --backup-selection '{
    "SelectionName": "RDS-Databases",
    "IamRoleArn": "arn:aws:iam::123456789012:role/AWSBackupDefaultServiceRole",
    "Resources": [
      "arn:aws:rds:us-east-1:123456789012:db:prod-sqlserver",
      "arn:aws:rds:us-east-1:123456789012:db:prod-oracle"
    ]
  }'

# List backup jobs
aws backup list-backup-jobs \
  --by-state COMPLETED \
  --query 'BackupJobs[].{Resource:ResourceArn,Status:State,Created:CreationDate,Size:BackupSizeInBytes}' \
  --output table
```

---

---

# DAY 3 — AWS Managed Database Services

---

## 3.1 Amazon RDS Architecture

### What is Amazon RDS?

Amazon RDS (Relational Database Service) is a managed service that handles provisioning, patching, backups, and failover for your database — so you focus on data, not infrastructure.

```
  RDS Architecture:

  ┌──────────────────────────────────────────────────────┐
  │                    AMAZON RDS                         │
  │                                                      │
  │  YOUR APP ──► RDS Endpoint (DNS) ──► RDS Instance    │
  │                                                      │
  │  ┌────────────────────────────────────────────────┐  │
  │  │  What AWS Manages:                             │  │
  │  │  ✓ Hardware provisioning                       │  │
  │  │  ✓ OS installation & patching                  │  │
  │  │  ✓ DB engine installation                      │  │
  │  │  ✓ Automated backups                           │  │
  │  │  ✓ Multi-AZ failover                           │  │
  │  │  ✓ Storage auto-scaling                        │  │
  │  │  ✓ Monitoring (CloudWatch)                     │  │
  │  └────────────────────────────────────────────────┘  │
  │                                                      │
  │  ┌────────────────────────────────────────────────┐  │
  │  │  What DBA Manages:                             │  │
  │  │  ✓ Database schema design                      │  │
  │  │  ✓ Query optimization                          │  │
  │  │  ✓ Parameter group tuning                      │  │
  │  │  ✓ User/role management                        │  │
  │  │  ✓ Choosing instance size & storage            │  │
  │  │  ✓ Backup retention & scheduling               │  │
  │  │  ✓ Security group rules                        │  │
  │  └────────────────────────────────────────────────┘  │
  │                                                      │
  │  Supported Engines:                                  │
  │  SQL Server | Oracle | MySQL | PostgreSQL |          │
  │  MariaDB | Aurora (MySQL/PostgreSQL)                 │
  └──────────────────────────────────────────────────────┘
```

### AWS CLI Commands

```bash
# Create RDS SQL Server instance
aws rds create-db-instance \
  --db-instance-identifier prod-sqlserver \
  --db-instance-class db.r6i.2xlarge \
  --engine sqlserver-se \
  --engine-version 16.00 \
  --master-username admin \
  --master-user-password 'SecureP@ss123!' \
  --allocated-storage 500 \
  --storage-type gp3 \
  --iops 6000 \
  --storage-throughput 400 \
  --multi-az \
  --db-subnet-group-name my-db-subnets \
  --vpc-security-group-ids sg-dbxxx \
  --backup-retention-period 14 \
  --preferred-backup-window "02:00-03:00" \
  --preferred-maintenance-window "sun:04:00-sun:05:00" \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/xxx \
  --license-model license-included \
  --no-publicly-accessible

# Create RDS Oracle instance
aws rds create-db-instance \
  --db-instance-identifier prod-oracle \
  --db-instance-class db.r6i.4xlarge \
  --engine oracle-ee \
  --engine-version 19.0.0.0.ru-2024-10.rur-2024-10.r1 \
  --master-username admin \
  --master-user-password 'SecureP@ss123!' \
  --allocated-storage 1000 \
  --storage-type gp3 \
  --multi-az \
  --db-subnet-group-name my-db-subnets \
  --vpc-security-group-ids sg-dbxxx \
  --backup-retention-period 14 \
  --storage-encrypted \
  --license-model bring-your-own-license \
  --no-publicly-accessible

# Create RDS PostgreSQL instance
aws rds create-db-instance \
  --db-instance-identifier prod-postgres \
  --db-instance-class db.r6i.xlarge \
  --engine postgres \
  --engine-version 16.4 \
  --master-username dbadmin \
  --master-user-password 'SecureP@ss123!' \
  --allocated-storage 200 \
  --storage-type gp3 \
  --multi-az \
  --db-subnet-group-name my-db-subnets \
  --vpc-security-group-ids sg-dbxxx \
  --backup-retention-period 7 \
  --storage-encrypted \
  --no-publicly-accessible

# List all RDS instances
aws rds describe-db-instances \
  --query 'DBInstances[].{Name:DBInstanceIdentifier,Engine:Engine,Class:DBInstanceClass,Status:DBInstanceStatus,MultiAZ:MultiAZ,Storage:AllocatedStorage}' \
  --output table

# Get endpoint for connection
aws rds describe-db-instances \
  --db-instance-identifier prod-sqlserver \
  --query 'DBInstances[0].Endpoint.{Address:Address,Port:Port}' \
  --output table
```

---

## 3.2 RDS Backups, PITR & Snapshots

### RDS Backup Types

```
  RDS Backup System:

  ┌─────────────── AUTOMATED BACKUPS ──────────────────┐
  │                                                     │
  │  • Daily full snapshot during backup window          │
  │  • Transaction logs every 5 minutes                  │
  │  • Retention: 1-35 days (default: 7)                 │
  │  • Enables PITR (Point-In-Time Recovery)             │
  │  • Stored in S3 (managed by AWS)                     │
  │  • Deleted when RDS instance is deleted              │
  │    (unless you choose to keep final snapshot)         │
  └─────────────────────────────────────────────────────┘
  
  ┌─────────────── MANUAL SNAPSHOTS ───────────────────┐
  │                                                     │
  │  • Triggered by DBA on demand                        │
  │  • Kept FOREVER until you delete them                │
  │  • Can be shared with other AWS accounts             │
  │  • Can be copied cross-region for DR                 │
  │  • Survives RDS instance deletion                    │
  └─────────────────────────────────────────────────────┘
  
  ┌─────────────── PITR (Point-In-Time Recovery) ──────┐
  │                                                     │
  │  Restore to ANY second within retention period       │
  │  Creates a NEW RDS instance (not in-place)           │
  │  Uses: daily snapshot + transaction log replay       │
  │                                                     │
  │  Example: DB corrupted at 2:30 PM                    │
  │  PITR to 2:29 PM → new instance with clean data     │
  └─────────────────────────────────────────────────────┘
```

### AWS CLI Commands

```bash
# Create manual snapshot
aws rds create-db-snapshot \
  --db-instance-identifier prod-sqlserver \
  --db-snapshot-identifier prod-sqlserver-pre-upgrade-20250219

# List snapshots
aws rds describe-db-snapshots \
  --db-instance-identifier prod-sqlserver \
  --query 'DBSnapshots[].{ID:DBSnapshotIdentifier,Status:Status,Created:SnapshotCreateTime,Size:AllocatedStorage,Type:SnapshotType}' \
  --output table

# Point-In-Time Recovery (restore to a specific time)
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier prod-sqlserver \
  --target-db-instance-identifier prod-sqlserver-restored \
  --restore-time "2025-02-19T14:29:00Z" \
  --db-instance-class db.r6i.2xlarge \
  --db-subnet-group-name my-db-subnets \
  --vpc-security-group-ids sg-dbxxx \
  --no-publicly-accessible

# Restore from manual snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier prod-sqlserver-from-snap \
  --db-snapshot-identifier prod-sqlserver-pre-upgrade-20250219 \
  --db-instance-class db.r6i.2xlarge \
  --db-subnet-group-name my-db-subnets

# Copy snapshot to another region (DR)
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier arn:aws:rds:us-east-1:123456789012:snapshot:prod-sqlserver-pre-upgrade \
  --target-db-snapshot-identifier prod-sqlserver-dr-copy \
  --region eu-west-3 \
  --kms-key-id arn:aws:kms:eu-west-3:123456789012:key/xxx

# Check latest restorable time for PITR
aws rds describe-db-instances \
  --db-instance-identifier prod-sqlserver \
  --query 'DBInstances[0].{LatestRestore:LatestRestorableTime,EarliestRestore:EarliestRestorableTime}'

# Delete snapshot
aws rds delete-db-snapshot \
  --db-snapshot-identifier old-snapshot-name
```

---

## 3.3 RDS Patching & Parameter Groups

### RDS Maintenance Windows

```
  RDS Patching Lifecycle:

  AWS releases      DBA chooses           AWS applies patch
  new DB patch ──► maintenance window ──► during window
                    (e.g., Sun 4-5 AM)     (brief downtime
                                            for single-AZ,
                                            failover for
                                            Multi-AZ)

  Multi-AZ Patching Process:
  1. Patch standby instance
  2. Promote standby to primary (failover ~60 sec)
  3. Patch old primary (now standby)
  4. Result: minimal downtime!
```

### Parameter Groups

```
  Parameter Group = Database configuration settings
  
  ┌──────────────────────────────────────────────────┐
  │ Default Parameter Group: READ-ONLY (can't modify)│
  │ Custom Parameter Group: YOUR settings             │
  └──────────────────────────────────────────────────┘
  
  Examples:
  ┌──────────────────────┬──────────────────────────┐
  │ SQL Server           │ Oracle                   │
  ├──────────────────────┼──────────────────────────┤
  │ max degree of        │ open_cursors             │
  │ parallelism          │ sessions                 │
  │ cost threshold for   │ pga_aggregate_target     │
  │ parallelism          │ sga_target               │
  │ max server memory    │ cursor_sharing           │
  └──────────────────────┴──────────────────────────┘
```

### AWS CLI Commands

```bash
# Create custom parameter group
aws rds create-db-parameter-group \
  --db-parameter-group-name prod-sqlserver-params \
  --db-parameter-group-family sqlserver-se-16.0 \
  --description "Production SQL Server parameters"

# Modify parameters
aws rds modify-db-parameter-group \
  --db-parameter-group-name prod-sqlserver-params \
  --parameters \
    "ParameterName=cost threshold for parallelism,ParameterValue=50,ApplyMethod=immediate" \
    "ParameterName=max degree of parallelism,ParameterValue=4,ApplyMethod=immediate"

# For PostgreSQL parameter group
aws rds create-db-parameter-group \
  --db-parameter-group-name prod-postgres-params \
  --db-parameter-group-family postgres16 \
  --description "Production PostgreSQL parameters"

aws rds modify-db-parameter-group \
  --db-parameter-group-name prod-postgres-params \
  --parameters \
    "ParameterName=shared_buffers,ParameterValue={DBInstanceClassMemory/4},ApplyMethod=pending-reboot" \
    "ParameterName=work_mem,ParameterValue=65536,ApplyMethod=immediate" \
    "ParameterName=max_connections,ParameterValue=500,ApplyMethod=pending-reboot"

# Associate parameter group with instance
aws rds modify-db-instance \
  --db-instance-identifier prod-sqlserver \
  --db-parameter-group-name prod-sqlserver-params \
  --apply-immediately

# View current parameter values
aws rds describe-db-parameters \
  --db-parameter-group-name prod-sqlserver-params \
  --query 'Parameters[?ParameterValue!=`null`].{Name:ParameterName,Value:ParameterValue,Apply:ApplyMethod}' \
  --output table

# Check pending maintenance
aws rds describe-pending-maintenance-actions \
  --query 'PendingMaintenanceActions[].{DB:ResourceIdentifier,Action:PendingMaintenanceActionDetails[].Action,AutoApply:PendingMaintenanceActionDetails[].AutoAppliedAfterDate}' \
  --output table

# Apply pending maintenance immediately
aws rds apply-pending-maintenance-action \
  --resource-identifier arn:aws:rds:us-east-1:123456789012:db:prod-sqlserver \
  --apply-action system-update \
  --opt-in-type immediate
```

---

## 3.4 RDS Security (KMS, IAM, Secrets Manager)

### RDS Security Layers

```
  RDS Security Architecture:

  ┌─── Network Layer ──────────────────────────────────┐
  │  VPC + Private Subnet + Security Groups + NACLs    │
  │  No public accessibility                           │
  └────────────────────────────────────────────────────┘
           │
  ┌─── Encryption at Rest ─────────────────────────────┐
  │  AWS KMS (Key Management Service)                  │
  │  Encrypts: data files, backups, snapshots, logs    │
  │  Cannot be enabled after creation (must create new)│
  └────────────────────────────────────────────────────┘
           │
  ┌─── Encryption in Transit ──────────────────────────┐
  │  SSL/TLS for connections                           │
  │  Force SSL: rds.force_ssl = 1 (PostgreSQL)         │
  └────────────────────────────────────────────────────┘
           │
  ┌─── Authentication ─────────────────────────────────┐
  │  DB username/password (stored in Secrets Manager)  │
  │  IAM DB Authentication (PostgreSQL, MySQL)         │
  └────────────────────────────────────────────────────┘
           │
  ┌─── Secrets Manager ────────────────────────────────┐
  │  Auto-rotate DB passwords                          │
  │  Applications retrieve credentials via API         │
  │  No hardcoded passwords in config files            │
  └────────────────────────────────────────────────────┘
```

### AWS CLI Commands

```bash
# Create KMS key for RDS encryption
aws kms create-key \
  --description "RDS Database Encryption Key" \
  --key-usage ENCRYPT_DECRYPT
# Copy KeyId → key-xxx

aws kms create-alias \
  --alias-name alias/rds-encryption-key \
  --target-key-id key-xxx

# Store DB credentials in Secrets Manager
aws secretsmanager create-secret \
  --name prod/sqlserver/admin \
  --description "Production SQL Server admin credentials" \
  --secret-string '{"username":"admin","password":"SecureP@ss123!","engine":"sqlserver","host":"prod-sqlserver.xxx.us-east-1.rds.amazonaws.com","port":"1433","dbname":"master"}'

# Enable auto-rotation (every 30 days)
aws secretsmanager rotate-secret \
  --secret-id prod/sqlserver/admin \
  --rotation-rules '{"AutomaticallyAfterDays": 30}'

# Retrieve secret (application use)
aws secretsmanager get-secret-value \
  --secret-id prod/sqlserver/admin \
  --query 'SecretString' --output text

# Enable IAM DB authentication on RDS (PostgreSQL/MySQL)
aws rds modify-db-instance \
  --db-instance-identifier prod-postgres \
  --enable-iam-database-authentication \
  --apply-immediately

# Generate IAM auth token (use instead of password)
aws rds generate-db-auth-token \
  --hostname prod-postgres.xxx.us-east-1.rds.amazonaws.com \
  --port 5432 \
  --username dbadmin \
  --region us-east-1
```

---

## 3.5 Amazon Aurora Overview

### What is Aurora?

Aurora is AWS's cloud-native database engine, compatible with MySQL and PostgreSQL but with up to 5x better performance and higher availability.

```
  Aurora Architecture (vs Standard RDS):

  STANDARD RDS:
  ┌──────────┐         ┌──────────┐
  │ Primary  │ ──EBS──►│ Standby  │    2 copies of data
  │ Instance │  replicate│ Instance │    in 2 AZs
  └──────────┘         └──────────┘

  AURORA:
  ┌──────────┐
  │ Primary  │ ──────────────────────────────────────┐
  │ Instance │                                       │
  └──────────┘                                       │
  ┌──────────┐    ┌──────────────────────────────┐   │
  │ Reader 1 │───►│  Aurora Storage Layer         │◄──┘
  └──────────┘    │                              │
  ┌──────────┐    │  6 copies of data            │
  │ Reader 2 │───►│  across 3 AZs               │
  └──────────┘    │  Auto-healing                │
                  │  Auto-grows up to 128 TB     │
                  │  Continuous backup to S3      │
                  └──────────────────────────────┘
```

### Aurora vs Standard RDS

| Feature | Standard RDS | Aurora |
|---------|-------------|--------|
| **Storage replication** | 2 copies (Multi-AZ) | 6 copies across 3 AZs |
| **Failover time** | ~60-120 seconds | ~30 seconds |
| **Storage auto-scaling** | Manual increase | Auto-grows to 128 TB |
| **Read replicas** | Up to 5 (async) | Up to 15 (millisecond lag) |
| **Backup** | Daily snapshot + logs | Continuous to S3 |
| **PITR** | Within retention period | Up to last 5 minutes |
| **Performance** | Standard | Up to 5x MySQL, 3x PostgreSQL |
| **Cost** | Lower baseline | ~20% more than RDS |
| **Global Database** | Cross-region read replicas | Sub-second cross-region replication |

### AWS CLI Commands

```bash
# Create Aurora PostgreSQL cluster
aws rds create-db-cluster \
  --db-cluster-identifier prod-aurora-pg \
  --engine aurora-postgresql \
  --engine-version 16.4 \
  --master-username dbadmin \
  --master-user-password 'SecureP@ss123!' \
  --db-subnet-group-name my-db-subnets \
  --vpc-security-group-ids sg-dbxxx \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/xxx \
  --backup-retention-period 14

# Create writer instance
aws rds create-db-instance \
  --db-instance-identifier prod-aurora-pg-writer \
  --db-cluster-identifier prod-aurora-pg \
  --db-instance-class db.r6g.2xlarge \
  --engine aurora-postgresql

# Create reader instance (in different AZ)
aws rds create-db-instance \
  --db-instance-identifier prod-aurora-pg-reader1 \
  --db-cluster-identifier prod-aurora-pg \
  --db-instance-class db.r6g.xlarge \
  --engine aurora-postgresql

# Get Aurora endpoints
aws rds describe-db-clusters \
  --db-cluster-identifier prod-aurora-pg \
  --query 'DBClusters[0].{Writer:Endpoint,Reader:ReaderEndpoint,Port:Port}' \
  --output table

# Aurora: Create Global Database (cross-region)
aws rds create-global-cluster \
  --global-cluster-identifier global-prod-aurora \
  --source-db-cluster-identifier arn:aws:rds:us-east-1:123456789012:cluster:prod-aurora-pg \
  --region us-east-1
```

---

---

# DAY 4 — Monitoring, HA & DR

---

## 4.1 Amazon CloudWatch Metrics & Alarms

### Key RDS Metrics Every DBA Must Monitor

```
  CRITICAL CLOUDWATCH METRICS FOR DBAs:

  ┌─────────────────────────┬────────────────────────────────────┐
  │ Metric                  │ What It Means / Threshold          │
  ├─────────────────────────┼────────────────────────────────────┤
  │ CPUUtilization          │ > 80% sustained = upgrade needed   │
  │ FreeableMemory          │ < 1 GB = dangerously low           │
  │ FreeStorageSpace        │ < 10% = add storage urgently       │
  │ DatabaseConnections     │ Near max_connections = pool issue   │
  │ ReadIOPS / WriteIOPS    │ Near volume IOPS limit             │
  │ ReadLatency/WriteLatency│ > 10ms = storage bottleneck        │
  │ SwapUsage               │ > 0 = memory pressure              │
  │ ReplicaLag              │ > 1 sec = replication behind       │
  │ DiskQueueDepth          │ > 1 sustained = I/O bottleneck     │
  │ BurstBalance (gp2 only) │ < 20% = IOPS credits running out  │
  └─────────────────────────┴────────────────────────────────────┘
```

### AWS CLI Commands

```bash
# Get current CPU utilization for an RDS instance
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=prod-sqlserver \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average Maximum \
  --output table

# Get free storage space
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name FreeStorageSpace \
  --dimensions Name=DBInstanceIdentifier,Value=prod-sqlserver \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average

# Create alarm: CPU > 80% for 5 minutes
aws cloudwatch put-metric-alarm \
  --alarm-name "RDS-ProdSQL-High-CPU" \
  --alarm-description "CPU above 80% for 5 min on prod SQL Server" \
  --namespace AWS/RDS \
  --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=prod-sqlserver \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:dba-alerts

# Create alarm: Free storage < 20 GB
aws cloudwatch put-metric-alarm \
  --alarm-name "RDS-ProdSQL-Low-Storage" \
  --namespace AWS/RDS \
  --metric-name FreeStorageSpace \
  --dimensions Name=DBInstanceIdentifier,Value=prod-sqlserver \
  --statistic Average \
  --period 300 \
  --threshold 21474836480 \
  --comparison-operator LessThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:dba-alerts

# Create alarm: Replica lag > 30 seconds
aws cloudwatch put-metric-alarm \
  --alarm-name "RDS-ReadReplica-Lag" \
  --namespace AWS/RDS \
  --metric-name ReplicaLag \
  --dimensions Name=DBInstanceIdentifier,Value=prod-postgres-replica \
  --statistic Average \
  --period 60 \
  --threshold 30 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:dba-alerts

# Create SNS topic for DBA alerts
aws sns create-topic --name dba-alerts
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:dba-alerts \
  --protocol email \
  --notification-endpoint dba-team@company.com

# List all alarms
aws cloudwatch describe-alarms \
  --query 'MetricAlarms[].{Name:AlarmName,State:StateValue,Metric:MetricName}' \
  --output table
```

---

## 4.2 RDS Performance Insights

### What is Performance Insights?

A free tool (for limited retention) that visualizes database load and identifies bottleneck queries — like a "top command" for your database.

```
  Performance Insights Dashboard:

  ┌──────────────────────────────────────────────────────┐
  │  DB Load (Average Active Sessions)                   │
  │                                                      │
  │  ████████████████████░░░░░░░░░░░░  vCPU line: ────  │
  │  ████████████████░░░░░░░░░░░░░░░░                    │
  │  ██████████████░░░░░░░░░░░░░░░░░░                    │
  │                                                      │
  │  Sliced by WAIT EVENTS:                              │
  │  ■ CPU       ■ I/O       ■ Lock       ■ Network     │
  │                                                      │
  │  TOP SQL (queries causing most load):                │
  │  1. SELECT * FROM orders WHERE... (45% of load)      │
  │  2. UPDATE inventory SET...       (20% of load)      │
  │  3. INSERT INTO audit_log...      (10% of load)      │
  └──────────────────────────────────────────────────────┘
```

### AWS CLI Commands

```bash
# Enable Performance Insights on existing instance
aws rds modify-db-instance \
  --db-instance-identifier prod-sqlserver \
  --enable-performance-insights \
  --performance-insights-retention-period 731 \
  --performance-insights-kms-key-id arn:aws:kms:us-east-1:123456789012:key/xxx \
  --apply-immediately

# Get Performance Insights data (resource metrics)
aws pi get-resource-metrics \
  --service-type RDS \
  --identifier db-XXXXXXXXXXXXXXXXXXXXXX \
  --metric-queries '[{"Metric":"db.load.avg"}]' \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period-in-seconds 300

# Get PI resource ID from instance
aws rds describe-db-instances \
  --db-instance-identifier prod-sqlserver \
  --query 'DBInstances[0].DbiResourceId' --output text
```

---

## 4.3 High Availability & Failover

### RDS Multi-AZ Failover

```
  MULTI-AZ FAILOVER PROCESS:

  BEFORE FAILOVER:
  ┌──────────────────┐        ┌──────────────────┐
  │ PRIMARY (AZ-a)   │ ─sync─►│ STANDBY (AZ-b)   │
  │ Handles all I/O  │        │ No connections    │
  │ DNS: mydb.rds... │        │                   │
  └──────────────────┘        └──────────────────┘

  PRIMARY FAILS! (hardware, AZ outage, manual reboot with failover)

  AFTER FAILOVER (~60 seconds):
  ┌──────────────────┐        ┌──────────────────┐
  │ OLD PRIMARY      │        │ NEW PRIMARY (AZ-b)│
  │ (recovering)     │        │ DNS updated!      │
  │                  │        │ Handles all I/O   │
  └──────────────────┘        └──────────────────┘

  WHAT TRIGGERS AUTOMATIC FAILOVER:
  • AZ outage
  • Primary instance failure
  • Instance type change
  • OS patching
  • Manual reboot with failover
```

### AWS CLI Commands

```bash
# Check if Multi-AZ is enabled
aws rds describe-db-instances \
  --db-instance-identifier prod-sqlserver \
  --query 'DBInstances[0].{MultiAZ:MultiAZ,AZ:AvailabilityZone,SecondaryAZ:SecondaryAvailabilityZone}'

# Enable Multi-AZ (if not already)
aws rds modify-db-instance \
  --db-instance-identifier prod-sqlserver \
  --multi-az \
  --apply-immediately

# Trigger manual failover (for testing)
aws rds reboot-db-instance \
  --db-instance-identifier prod-sqlserver \
  --force-failover

# Create read replica (for read scaling)
aws rds create-db-instance-read-replica \
  --db-instance-identifier prod-postgres-replica \
  --source-db-instance-identifier prod-postgres \
  --db-instance-class db.r6i.xlarge \
  --availability-zone us-east-1c

# Create cross-region read replica (for DR)
aws rds create-db-instance-read-replica \
  --db-instance-identifier prod-postgres-dr \
  --source-db-instance-identifier arn:aws:rds:us-east-1:123456789012:db:prod-postgres \
  --db-instance-class db.r6i.xlarge \
  --region eu-west-3

# Promote read replica to standalone (DR activation)
aws rds promote-read-replica \
  --db-instance-identifier prod-postgres-dr
```

---

## 4.4 Disaster Recovery Strategies

### DR Strategies on AWS

```
  DR STRATEGIES (fastest recovery → slowest, cheapest → most expensive):

  ┌──────────────────────────────────────────────────────────────┐
  │                                                              │
  │  1. BACKUP & RESTORE (RPO: hours, RTO: hours)               │
  │     Cross-region snapshot copies                             │
  │     Cheapest, slowest recovery                               │
  │     Cost: $  (snapshot storage only)                         │
  │                                                              │
  │  2. PILOT LIGHT (RPO: minutes, RTO: minutes-hours)           │
  │     Cross-region read replica (minimum size)                 │
  │     Promote replica when disaster occurs                     │
  │     Cost: $$  (small running instance)                       │
  │                                                              │
  │  3. WARM STANDBY (RPO: seconds, RTO: minutes)                │
  │     Cross-region read replica (production size)              │
  │     Ready to take traffic immediately                        │
  │     Cost: $$$  (full-size running instance)                  │
  │                                                              │
  │  4. MULTI-REGION ACTIVE-ACTIVE (RPO: ~0, RTO: ~0)            │
  │     Aurora Global Database                                   │
  │     Sub-second replication, instant failover                 │
  │     Cost: $$$$  (full infrastructure in 2 regions)           │
  │                                                              │
  └──────────────────────────────────────────────────────────────┘
```

---

## 4.5 DB Troubleshooting on AWS

### Common Issues & Quick Fixes

| Issue | Symptom | AWS CLI Diagnosis | Fix |
|-------|---------|-------------------|-----|
| **High CPU** | Slow queries | Check CPUUtilization metric | Optimize queries, upgrade instance |
| **Out of storage** | Writes failing | Check FreeStorageSpace | Modify instance to increase storage |
| **Too many connections** | Connection refused | Check DatabaseConnections | Increase max_connections or use PgBouncer |
| **Replication lag** | Stale reads | Check ReplicaLag metric | Upgrade replica, reduce write load |
| **Instance unreachable** | Can't connect | Check security groups + subnet routing | Verify SG rules and route tables |

```bash
# Quick health check script for DBAs
echo "=== RDS Instance Status ==="
aws rds describe-db-instances \
  --query 'DBInstances[].{Name:DBInstanceIdentifier,Status:DBInstanceStatus,Class:DBInstanceClass,Engine:Engine,MultiAZ:MultiAZ}' \
  --output table

echo "=== Recent RDS Events (last 24h) ==="
aws rds describe-events \
  --duration 1440 \
  --query 'Events[].{Source:SourceIdentifier,Type:SourceType,Message:Message,Date:Date}' \
  --output table

echo "=== Pending Maintenance ==="
aws rds describe-pending-maintenance-actions \
  --output table

echo "=== CloudWatch Alarms in ALARM state ==="
aws cloudwatch describe-alarms \
  --state-value ALARM \
  --query 'MetricAlarms[].{Name:AlarmName,Metric:MetricName,Threshold:Threshold}' \
  --output table
```

---

---

# DAY 5 — Security, Automation & Cost

---

## 5.1 KMS & Secrets Manager

### KMS (Key Management Service)

```
  KMS Encryption for Databases:

  ┌──────────────────────────────────────────────┐
  │               AWS KMS                         │
  │                                              │
  │  CMK (Customer Master Key)                   │
  │  ├── Encrypts: RDS data files               │
  │  ├── Encrypts: RDS automated backups        │
  │  ├── Encrypts: RDS manual snapshots         │
  │  ├── Encrypts: EBS volumes                  │
  │  ├── Encrypts: S3 backup objects            │
  │  └── Encrypts: CloudWatch Logs              │
  │                                              │
  │  Key Types:                                  │
  │  • AWS Managed: aws/rds (free, auto-managed) │
  │  • Customer Managed: your key (you control)  │
  └──────────────────────────────────────────────┘
```

### AWS CLI Commands

```bash
# Create CMK for database encryption
aws kms create-key \
  --description "Production Database Encryption Key" \
  --key-usage ENCRYPT_DECRYPT \
  --origin AWS_KMS

aws kms create-alias \
  --alias-name alias/prod-db-key \
  --target-key-id key-xxx

# Enable automatic key rotation (every year)
aws kms enable-key-rotation --key-id key-xxx

# List all KMS keys
aws kms list-aliases \
  --query 'Aliases[?starts_with(AliasName,`alias/prod`)].{Alias:AliasName,KeyId:TargetKeyId}' \
  --output table

# Store DB password in Secrets Manager
aws secretsmanager create-secret \
  --name prod/oracle/admin \
  --secret-string '{"username":"admin","password":"OracleP@ss123!","host":"prod-oracle.xxx.rds.amazonaws.com","port":"1521"}'

# Retrieve secret
aws secretsmanager get-secret-value \
  --secret-id prod/oracle/admin \
  --query 'SecretString' --output text

# Rotate secret manually
aws secretsmanager rotate-secret --secret-id prod/oracle/admin

# List all secrets
aws secretsmanager list-secrets \
  --query 'SecretList[].{Name:Name,LastRotated:LastRotatedDate}' \
  --output table
```

---

## 5.2 AWS CLI for DBAs

### Essential AWS CLI Setup

```bash
# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configure CLI
aws configure
# Enter: Access Key ID, Secret Key, Region, Output format (json/table)

# Use named profiles for different environments
aws configure --profile prod
aws configure --profile dev

# Use profile in commands
aws rds describe-db-instances --profile prod
```

### DBA Daily Operations Cheat Sheet

```bash
# ═══ INSTANCE MANAGEMENT ═══
aws rds describe-db-instances --output table                           # List all DBs
aws rds describe-db-instances --db-instance-identifier prod-sql        # Single DB details
aws rds reboot-db-instance --db-instance-identifier prod-sql           # Reboot
aws rds stop-db-instance --db-instance-identifier dev-sql              # Stop (dev/test)
aws rds start-db-instance --db-instance-identifier dev-sql             # Start

# ═══ SCALING ═══
aws rds modify-db-instance --db-instance-identifier prod-sql \
  --db-instance-class db.r6i.4xlarge --apply-immediately               # Scale up
aws rds modify-db-instance --db-instance-identifier prod-sql \
  --allocated-storage 1000 --apply-immediately                         # Add storage

# ═══ BACKUPS ═══
aws rds create-db-snapshot --db-instance-identifier prod-sql \
  --db-snapshot-identifier manual-snap-$(date +%Y%m%d)                 # Manual snapshot
aws rds describe-db-snapshots --db-instance-identifier prod-sql        # List snapshots

# ═══ MONITORING ═══
aws rds describe-events --duration 60                                  # Last 1 hour events
aws cloudwatch describe-alarms --state-value ALARM                     # Active alarms
aws rds describe-db-log-files --db-instance-identifier prod-sql        # Available logs
aws rds download-db-log-file-portion \
  --db-instance-identifier prod-sql \
  --log-file-name error/sqlserver-error.log --output text              # Download error log

# ═══ SECURITY ═══
aws rds describe-db-instances \
  --query 'DBInstances[].{DB:DBInstanceIdentifier,Encrypted:StorageEncrypted,KMS:KmsKeyId}' \
  --output table                                                       # Check encryption
```

---

## 5.3 Automation & Snapshot Scheduling

### Automated Snapshot with Lambda + EventBridge

```
  Automated Snapshot Architecture:

  EventBridge Rule              Lambda Function         RDS
  (cron: every 6 hours) ──────► create_snapshot() ────► Snapshot Created
                                      │
                                      ▼
                                Delete snapshots
                                older than 7 days
```

### AWS CLI: Schedule Snapshots

```bash
# Option 1: Use AWS Backup (recommended)
aws backup create-backup-plan --backup-plan '{
  "BackupPlanName": "RDS-6Hour-Snapshots",
  "Rules": [
    {
      "RuleName": "Every6Hours",
      "TargetBackupVaultName": "dba-prod-vault",
      "ScheduleExpression": "cron(0 0/6 * * ? *)",
      "Lifecycle": {"DeleteAfterDays": 7}
    },
    {
      "RuleName": "WeeklySunday",
      "TargetBackupVaultName": "dba-prod-vault",
      "ScheduleExpression": "cron(0 2 ? * SUN *)",
      "Lifecycle": {
        "MoveToColdStorageAfterDays": 30,
        "DeleteAfterDays": 365
      }
    }
  ]
}'

# Option 2: Simple CLI-based cron (add to crontab)
# Every 6 hours — take snapshot
# 0 */6 * * * aws rds create-db-snapshot \
#   --db-instance-identifier prod-sqlserver \
#   --db-snapshot-identifier "auto-$(date +\%Y\%m\%d-\%H\%M)"

# Delete snapshots older than 7 days (cleanup script)
aws rds describe-db-snapshots \
  --db-instance-identifier prod-sqlserver \
  --snapshot-type manual \
  --query "DBSnapshots[?SnapshotCreateTime<='$(date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%S)'].DBSnapshotIdentifier" \
  --output text | tr '\t' '\n' | while read snap; do
    echo "Deleting old snapshot: $snap"
    aws rds delete-db-snapshot --db-snapshot-identifier "$snap"
  done
```

---

## 5.4 Cost Management & Optimization

### RDS Cost Breakdown

```
  WHERE DOES YOUR RDS MONEY GO?

  ┌───────────────────────────────────────────────────────┐
  │  Instance (compute)     ~60% of RDS cost              │
  │  ██████████████████████████████                       │
  │                                                       │
  │  Storage (EBS/Aurora)   ~20% of RDS cost              │
  │  ██████████████                                       │
  │                                                       │
  │  Backup storage         ~5%                           │
  │  ████                                                 │
  │                                                       │
  │  Data transfer          ~5%                           │
  │  ████                                                 │
  │                                                       │
  │  Snapshots (manual)     ~5%                           │
  │  ████                                                 │
  │                                                       │
  │  License (if included)  ~5%                           │
  │  ████                                                 │
  └───────────────────────────────────────────────────────┘
```

### Cost Optimization Tips for DBAs

| Strategy | Savings | How |
|----------|---------|-----|
| **Reserved Instances** | 30-60% | Commit to 1 or 3 years for production DBs |
| **Right-sizing** | 20-40% | Downsize over-provisioned instances |
| **Stop dev/test** | 70-100% | Stop non-prod RDS outside business hours |
| **gp3 over io2** | 40-50% | If you need < 16,000 IOPS |
| **Aurora Serverless** | Variable | For unpredictable dev/test workloads |
| **Delete old snapshots** | 5-10% | Automate cleanup of manual snapshots |
| **Single-AZ for dev** | 50% | No Multi-AZ needed for non-production |

### AWS CLI Commands

```bash
# Check current monthly RDS cost estimate
aws ce get-cost-and-usage \
  --time-period Start=$(date -d "first day of this month" +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Relational Database Service"]}}' \
  --output table

# List Reserved Instance recommendations
aws rds describe-reserved-db-instances-offerings \
  --db-instance-class db.r6i.2xlarge \
  --duration 31536000 \
  --product-description "SQL Server (License Included)" \
  --offering-type "Partial Upfront" \
  --query 'ReservedDBInstancesOfferings[].{Class:DBInstanceClass,Price:FixedPrice,Monthly:RecurringCharges[0].RecurringChargeAmount}' \
  --output table

# Purchase Reserved Instance
aws rds purchase-reserved-db-instances-offering \
  --reserved-db-instances-offering-id offering-id-xxx \
  --db-instance-count 1

# Find oversized instances (compare allocated vs used)
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=prod-sqlserver \
  --start-time $(date -u -d '30 days ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 86400 \
  --statistics Average Maximum
# If Max CPU < 40% consistently → consider downsizing

# Stop dev/test instances (run via cron at 7 PM)
for db in dev-sqlserver dev-postgres test-oracle; do
  aws rds stop-db-instance --db-instance-identifier $db
done

# Start dev/test instances (run via cron at 8 AM)
for db in dev-sqlserver dev-postgres test-oracle; do
  aws rds start-db-instance --db-instance-identifier $db
done
```

---

## 5.5 Cloud DBA Operating Model

### Daily DBA Routine on AWS

```
  CLOUD DBA DAILY OPERATIONS:

  ┌──────────────────────────────────────────────────────────────┐
  │  8:00 AM  │  Morning Health Check                           │
  │           │  • Check CloudWatch alarms (any ALARM state?)   │
  │           │  • Review RDS events from overnight              │
  │           │  • Verify backup completion                      │
  │           │  • Check replica lag                             │
  ├───────────┼────────────────────────────────────────────────  │
  │  9:00 AM  │  Performance Review                             │
  │           │  • Performance Insights — any spike?             │
  │           │  • Top SQL queries consuming resources           │
  │           │  • Storage space trending                        │
  ├───────────┼──────────────────────────────────────────────────│
  │  Ongoing  │  Tickets & Requests                             │
  │           │  • Schema changes (via parameter groups)         │
  │           │  • User access management                        │
  │           │  • Snapshot before changes                       │
  ├───────────┼──────────────────────────────────────────────────│
  │  Weekly   │  Maintenance                                    │
  │           │  • Review pending patches                        │
  │           │  • Verify cross-region DR snapshots              │
  │           │  • Review cost reports                           │
  │           │  • Test failover (non-prod)                      │
  ├───────────┼──────────────────────────────────────────────────│
  │  Monthly  │  Optimization                                   │
  │           │  • Right-size instances                           │
  │           │  • Review Reserved Instance utilization          │
  │           │  • Clean up old snapshots                        │
  │           │  • Security audit (IAM, SG rules)               │
  └───────────┴──────────────────────────────────────────────────┘
```

### Quick Morning Health Check Script

```bash
#!/bin/bash
# save as: dba-morning-check.sh

echo "============================================"
echo "  AWS DBA Morning Health Check"
echo "  Date: $(date)"
echo "============================================"

echo ""
echo "=== 1. RDS Instance Status ==="
aws rds describe-db-instances \
  --query 'DBInstances[].{Name:DBInstanceIdentifier,Status:DBInstanceStatus,Engine:Engine,Class:DBInstanceClass,MultiAZ:MultiAZ}' \
  --output table

echo ""
echo "=== 2. Active CloudWatch Alarms ==="
aws cloudwatch describe-alarms --state-value ALARM \
  --query 'MetricAlarms[].{Alarm:AlarmName,State:StateValue,Reason:StateReason}' \
  --output table 2>/dev/null || echo "No alarms in ALARM state ✅"

echo ""
echo "=== 3. RDS Events (Last 12 Hours) ==="
aws rds describe-events --duration 720 \
  --query 'Events[].{Source:SourceIdentifier,Message:Message,Time:Date}' \
  --output table

echo ""
echo "=== 4. Pending Maintenance ==="
aws rds describe-pending-maintenance-actions \
  --query 'PendingMaintenanceActions[].{DB:ResourceIdentifier,Action:PendingMaintenanceActionDetails[].Action}' \
  --output table 2>/dev/null || echo "No pending maintenance ✅"

echo ""
echo "=== 5. Latest Backup Status ==="
aws rds describe-db-snapshots \
  --snapshot-type automated \
  --query 'DBSnapshots | sort_by(@, &SnapshotCreateTime) | [-3:].{DB:DBInstanceIdentifier,Created:SnapshotCreateTime,Status:Status,Size:AllocatedStorage}' \
  --output table

echo ""
echo "============================================"
echo "  Health Check Complete"
echo "============================================"
```

---

## Summary: 5-Day Quick Reference Card

| Day | Topic | Key AWS Services | Critical CLI Commands |
|-----|-------|-----------------|----------------------|
| **Day 1** | AWS Foundations | IAM, VPC, Security Groups | `aws iam`, `aws ec2 create-vpc`, `authorize-security-group-ingress` |
| **Day 2** | Compute & Storage | EC2, EBS, S3, AWS Backup | `aws ec2 create-volume`, `create-snapshot`, `aws s3 cp` |
| **Day 3** | Managed DB Services | RDS, Aurora, KMS, Secrets Manager | `aws rds create-db-instance`, `create-db-snapshot`, `restore-db-instance-to-point-in-time` |
| **Day 4** | Monitoring, HA & DR | CloudWatch, Performance Insights | `aws cloudwatch put-metric-alarm`, `rds reboot-db-instance --force-failover` |
| **Day 5** | Security, Automation & Cost | KMS, Secrets Manager, AWS Backup, Cost Explorer | `aws kms create-key`, `secretsmanager create-secret`, `aws ce get-cost-and-usage` |

---

*5-Day AWS Training Program for Cloud DBAs — Complete Notes with Examples & Commands*
