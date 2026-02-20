# CockroachDB — Complete Student Guide
## 2-Hour Classroom Training for DBAs

> **What is CockroachDB, Why it Exists, How it Works, and When to Use It**
> Written in plain English with diagrams, examples, and real-world use cases

---

# Chapter 1: What is CockroachDB?

## The One-Line Answer

CockroachDB is a **distributed SQL database** that gives you the familiar SQL interface of PostgreSQL but spreads your data across multiple servers, data centers, and even continents — automatically — while guaranteeing your data is always correct and always available.

## Why is it Called "CockroachDB"?

Just like a cockroach can survive almost anything — fire, flood, radiation — CockroachDB is designed to **survive any failure**. Kill a server? It keeps running. Lose an entire data center? It keeps running. Lose an entire region? It keeps running. The name literally means "the database that refuses to die."

## The Problem CockroachDB Solves

To understand CockroachDB, you need to understand the problem it was built to solve.

### The Traditional Database Problem

Traditional databases like MySQL, PostgreSQL, SQL Server, and Oracle were designed to run on a **single server**. All your data lives on one machine. This creates several problems as your application grows:

```
  TRADITIONAL DATABASE (Single Server):

  ┌──────────────────────────────────┐
  │         SINGLE SERVER            │
  │                                  │
  │   ALL your data is here          │
  │   ALL your queries run here      │
  │   ALL your users connect here    │
  │                                  │
  │   Problem 1: If this server      │
  │   dies, EVERYTHING is down       │
  │                                  │
  │   Problem 2: If you have 100     │
  │   million users, this one server │
  │   can't handle all the traffic   │
  │                                  │
  │   Problem 3: If your users are   │
  │   in India, US, and Europe,      │
  │   someone will have slow latency │
  └──────────────────────────────────┘
```

**Problem 1 — Single Point of Failure:** If your one database server crashes (hardware failure, power outage, software bug), your entire application goes down. You can set up replication (like MySQL Replica or PostgreSQL Standby), but failover is often manual, slow, and risky.

**Problem 2 — Scaling Limits:** A single server can only have so much CPU, RAM, and disk. When you hit that limit, you can't just add more servers — traditional databases don't know how to split data across machines. This is called the "scale-up" problem — you can only buy a bigger server, and eventually there's no bigger server to buy.

**Problem 3 — Geographic Latency:** If your database server is in Mumbai and your user is in New York, every database query crosses the ocean — adding 200+ milliseconds of latency. Traditional databases can't put data closer to the users who need it.

### The CockroachDB Solution

CockroachDB solves all three problems:

```
  COCKROACHDB (Distributed across multiple servers):

  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │   NODE 1     │  │   NODE 2     │  │   NODE 3     │
  │   Mumbai     │  │   New York   │  │   London     │
  │              │  │              │  │              │
  │  Part of     │  │  Part of     │  │  Part of     │
  │  your data   │  │  your data   │  │  your data   │
  │              │  │              │  │              │
  │  + Copies of │  │  + Copies of │  │  + Copies of │
  │  other data  │  │  other data  │  │  other data  │
  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
         │                 │                 │
         └─────── Automatic Sync ────────────┘
           Data is replicated 3 times
           If any node dies, the others continue
           Users connect to the nearest node
```

**Solution 1 — No Single Point of Failure:** Data is automatically copied to 3 (or more) nodes. If one node dies, the other two have the data. CockroachDB automatically detects the failure and continues serving requests. No manual failover needed. No downtime.

**Solution 2 — Horizontal Scaling:** Need more capacity? Just add more servers (nodes) to the cluster. CockroachDB automatically rebalances data across all nodes. This is "scale-out" — instead of buying one bigger server, you add many smaller servers.

**Solution 3 — Global Data Placement:** CockroachDB can pin specific data to specific regions. Indian customer data stays in Mumbai for low latency. European customer data stays in London for GDPR compliance. American data stays in New York. Each user gets fast, local access.

---

# Chapter 2: How CockroachDB Works Internally

## Architecture Overview

CockroachDB has a **layered architecture** where each layer handles a specific responsibility:

```
  COCKROACHDB ARCHITECTURE (LAYERED):

  ┌─────────────────────────────────────────────────┐
  │  LAYER 1: SQL LAYER                              │
  │  Parses SQL queries (PostgreSQL-compatible)      │
  │  Plans query execution                           │
  │  Your app connects here using any PostgreSQL     │
  │  driver (psql, pgAdmin, JDBC, Python psycopg2)  │
  └──────────────────────┬──────────────────────────┘
                         │
  ┌──────────────────────▼──────────────────────────┐
  │  LAYER 2: TRANSACTION LAYER                      │
  │  Handles ACID transactions across nodes          │
  │  Uses a protocol inspired by Google's Spanner    │
  │  Ensures Serializable Isolation (strongest!)     │
  └──────────────────────┬──────────────────────────┘
                         │
  ┌──────────────────────▼──────────────────────────┐
  │  LAYER 3: DISTRIBUTION LAYER                     │
  │  Splits data into "Ranges" (64 MB chunks)        │
  │  Distributes ranges across nodes                 │
  │  Each range has 3 replicas (by default)          │
  │  Uses Raft consensus for replication             │
  └──────────────────────┬──────────────────────────┘
                         │
  ┌──────────────────────▼──────────────────────────┐
  │  LAYER 4: STORAGE LAYER                          │
  │  Uses Pebble (a RocksDB-like key-value store)    │
  │  Stores data as sorted key-value pairs on disk   │
  │  Each node manages its own local storage         │
  └─────────────────────────────────────────────────┘
```

## Key Concepts Explained

### Concept 1: Nodes

A **node** is a single instance of CockroachDB running on a server (physical or virtual). A CockroachDB cluster is made up of multiple nodes working together.

Every node is **equal** — there is no master or slave. Any node can accept reads and writes for any data. This is fundamentally different from MySQL/PostgreSQL where there's one primary that handles writes and replicas that handle reads.

Minimum recommended: 3 nodes (for data safety). Production: 5 to hundreds of nodes.

### Concept 2: Ranges

CockroachDB doesn't store an entire table on one node. Instead, it splits tables into **ranges** — chunks of approximately 64 MB of contiguous data (sorted by primary key).

```
  HOW A TABLE IS SPLIT INTO RANGES:

  Table: customers (sorted by customer_id)
  Total size: 320 MB

  ┌─────────────────────────────────────────────────────────────┐
  │  Range 1: customer_id 1 - 25,000       (~64 MB)            │
  │  Range 2: customer_id 25,001 - 50,000  (~64 MB)            │
  │  Range 3: customer_id 50,001 - 75,000  (~64 MB)            │
  │  Range 4: customer_id 75,001 - 100,000 (~64 MB)            │
  │  Range 5: customer_id 100,001+         (~64 MB)            │
  └─────────────────────────────────────────────────────────────┘

  These ranges are distributed across nodes:

  Node 1 (Mumbai):    Range 1, Range 4
  Node 2 (New York):  Range 2, Range 5
  Node 3 (London):    Range 3

  PLUS: Each range has 3 copies (replicas) spread across nodes
```

As data grows, ranges automatically **split** when they exceed 64 MB. As nodes are added, ranges automatically **rebalance** to distribute evenly. You never manage this manually.

### Concept 3: Replicas and Raft Consensus

Every range is replicated **3 times** (by default) across different nodes. One replica is the **Leaseholder** (handles reads and coordinates writes), and they all participate in the **Raft consensus protocol** for writes.

**How a write works:**

```
  CLIENT: INSERT INTO customers (name) VALUES ('Venkat');

  Step 1: Client connects to ANY node (say Node 2)
  Step 2: Node 2 determines which range owns this key
  Step 3: The write goes to the Leaseholder of that range
  Step 4: Leaseholder proposes the write to Raft group
  Step 5: At least 2 of 3 replicas must agree (majority)
  Step 6: Once majority agrees → write is committed
  Step 7: Success returned to client

  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │ Replica 1│     │ Replica 2│     │ Replica 3│
  │ (Leader) │────►│ (Follower│────►│ (Follower│
  │ Node 1   │     │  Node 2) │     │  Node 3) │
  └──────────┘     └──────────┘     └──────────┘
       │                │                │
       ▼                ▼                ▼
     AGREE            AGREE           (can fail)
     
  2 out of 3 agreed → COMMITTED!
  Even if Node 3 is down, the write succeeds!
```

This is why CockroachDB can **survive node failures** — as long as a majority of replicas are alive, the data is safe and the database keeps running.

### Concept 4: ACID Transactions

CockroachDB provides **full ACID transactions**, even across multiple nodes and ranges:

- **Atomicity:** All parts of a transaction succeed or all fail. No partial writes.
- **Consistency:** The database always moves from one valid state to another.
- **Isolation:** CockroachDB uses **Serializable isolation** — the strongest level. Concurrent transactions behave as if they ran one at a time.
- **Durability:** Once committed, data survives any failure.

This is what makes CockroachDB special. Other distributed databases (like Cassandra, MongoDB, DynamoDB) sacrifice some of these ACID guarantees for performance. CockroachDB gives you all of them.

---

# Chapter 3: CockroachDB vs Traditional Databases

## Comparison Table

| Feature | CockroachDB | PostgreSQL | MySQL | SQL Server | MongoDB | Cassandra |
|---------|------------|------------|-------|------------|---------|-----------|
| **SQL Support** | Full SQL (PG-compatible) | Full SQL | Full SQL | Full SQL | No SQL (document) | CQL (limited) |
| **Distributed** | Yes (built-in) | No (single node) | No (single node) | No (single node) | Yes (sharded) | Yes (ring) |
| **ACID Transactions** | Full (Serializable) | Full | Full | Full | Limited | No |
| **Horizontal Scaling** | Add nodes, auto-rebalance | Scale-up only | Scale-up only | Scale-up only | Add shards (manual) | Add nodes |
| **Auto Failover** | Yes (seconds) | Manual/Patroni | Manual | Always On AG | Yes | Yes |
| **Multi-Region** | Built-in | No | No | No | Zone-aware | Multi-DC |
| **No Single Point of Failure** | Yes | No (primary) | No (primary) | No (primary) | Yes (with config) | Yes |
| **PostgreSQL Compatible** | Yes | N/A | No | No | No | No |
| **Replication Factor** | Configurable (default 3) | 1 primary + N standby | 1 primary + N replica | 1 primary + N replica | Configurable | Configurable |
| **Consistency Model** | Strong (Serializable) | Strong | Strong | Strong | Eventual (default) | Eventual |
| **License** | BSL / Enterprise | Open Source | Open Source/GPL | Commercial | SSPL | Apache 2.0 |

## When to Use CockroachDB (and When NOT to)

### USE CockroachDB When:

1. **You need zero-downtime, always-on availability** — Your application cannot afford even minutes of downtime for failover. Banking, e-commerce, telecom, gaming platforms.

2. **You need to scale beyond a single server** — Your data or traffic is growing and you'll eventually hit the limits of the biggest server available. IoT, SaaS platforms, social media.

3. **You have users across multiple geographic regions** — You need low latency for users in India, US, and Europe simultaneously. Global applications, CDN-backed services.

4. **You need strong consistency with distributed data** — You cannot tolerate stale reads or lost writes. Financial transactions, inventory systems, reservation systems.

5. **You want SQL without the scaling headache** — You don't want to manually shard MySQL/PostgreSQL or manage complex replication topologies.

### DO NOT USE CockroachDB When:

1. **Single-region, single-server workload** — If all your data fits on one server and you don't need multi-region, traditional PostgreSQL is simpler and cheaper.

2. **Heavy analytics / OLAP** — CockroachDB is designed for OLTP (transactional) workloads. For data warehousing and analytics, use ClickHouse, Snowflake, or BigQuery.

3. **Very low-latency single-row lookups** — Redis, DynamoDB, or Cassandra are faster for simple key-value lookups.

4. **Tight budget** — CockroachDB Enterprise features (geo-partitioning, CDC, encryption at rest) require a license. The free Core edition has limitations.

5. **Team lacks distributed systems knowledge** — CockroachDB has a learning curve. If your team is only familiar with MySQL/PostgreSQL, factor in training time.

---

# Chapter 4: CockroachDB Key Features

## Feature 1: PostgreSQL Compatibility

CockroachDB uses the PostgreSQL wire protocol. This means:

- Connect using `psql`, pgAdmin, DBeaver, or any PostgreSQL client
- Use existing PostgreSQL drivers (JDBC, psycopg2, pg gem, node-postgres)
- Write standard SQL (SELECT, INSERT, UPDATE, DELETE, JOIN, subqueries, CTEs, window functions)
- Use PostgreSQL data types (INT, TEXT, TIMESTAMP, JSONB, ARRAY, UUID, INET)

```sql
-- This works in BOTH PostgreSQL and CockroachDB:
CREATE TABLE customers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name STRING NOT NULL,
    email STRING UNIQUE,
    region STRING,
    created_at TIMESTAMP DEFAULT now()
);

INSERT INTO customers (name, email, region) 
VALUES ('Venkat', 'venkat@telecom.com', 'ap-south-1');

SELECT * FROM customers WHERE region = 'ap-south-1';
```

**What's different from PostgreSQL:** CockroachDB doesn't support some PostgreSQL features like stored procedures (PL/pgSQL), triggers, custom extensions, or certain advanced data types. Check the compatibility docs before migrating.

## Feature 2: Automatic Sharding (Range Splitting)

Unlike MongoDB where you manually choose a shard key, or MySQL where you manually partition tables, CockroachDB **automatically splits and distributes data**:

```
  AUTOMATIC SHARDING IN ACTION:

  Day 1: You create a table. It starts as ONE range on ONE node.
  ┌──────────────────┐
  │  Range 1: 0-∞    │  (all data, < 64 MB)
  │  Node 1           │
  └──────────────────┘

  Day 30: Table grows to 200 MB. CockroachDB auto-splits into 4 ranges.
  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ Range 1  │  │ Range 2  │  │ Range 3  │  │ Range 4  │
  │ Node 1   │  │ Node 2   │  │ Node 3   │  │ Node 1   │
  └──────────┘  └──────────┘  └──────────┘  └──────────┘
  CockroachDB rebalanced ranges across 3 nodes automatically!

  Day 90: You add Node 4 and Node 5 to the cluster.
  CockroachDB automatically moves ranges to the new nodes.
  No downtime. No manual work. Data rebalances in the background.
```

## Feature 3: Multi-Region Capabilities

This is CockroachDB's flagship feature for global applications:

```sql
-- Create a multi-region database
ALTER DATABASE telecom PRIMARY REGION "ap-south-1";
ALTER DATABASE telecom ADD REGION "us-east-1";
ALTER DATABASE telecom ADD REGION "eu-west-3";

-- REGIONAL BY ROW: Each row lives in the region of its user
CREATE TABLE subscribers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name STRING,
    phone STRING,
    region crdb_internal_region NOT NULL DEFAULT gateway_region()
) LOCALITY REGIONAL BY ROW;

-- Indian subscriber → data stored in ap-south-1 (low latency for India)
-- US subscriber → data stored in us-east-1 (low latency for US)
-- EU subscriber → data stored in eu-west-3 (GDPR compliant!)

-- GLOBAL TABLE: Read from anywhere with low latency (rarely updated)
CREATE TABLE plans (
    id INT PRIMARY KEY,
    name STRING,
    price DECIMAL
) LOCALITY GLOBAL;
-- Plans table is read-optimized across all regions
-- Every region has an up-to-date copy for fast reads

-- SURVIVE REGION FAILURE: Database stays online even if entire region goes down
ALTER DATABASE telecom SURVIVE REGION FAILURE;
```

## Feature 4: Distributed SQL Queries

CockroachDB can execute queries that touch data on multiple nodes, performing distributed joins and aggregations:

```sql
-- This query might touch data on nodes in Mumbai, New York, and London
-- CockroachDB handles the distributed execution automatically
SELECT s.name, p.name as plan_name, COUNT(c.id) as total_calls
FROM subscribers s
JOIN plans p ON s.plan_id = p.id
JOIN call_records c ON c.subscriber_id = s.id
WHERE c.call_date > '2025-01-01'
GROUP BY s.name, p.name
ORDER BY total_calls DESC
LIMIT 10;
```

## Feature 5: Change Data Capture (CDC)

CockroachDB can stream every database change to external systems like Kafka, cloud storage, or webhooks:

```sql
-- Stream all changes to the subscribers table to Kafka
CREATE CHANGEFEED FOR TABLE subscribers
INTO 'kafka://kafka-broker:9092?topic_name=subscriber_changes'
WITH updated, resolved;

-- Every INSERT, UPDATE, DELETE on subscribers is now
-- automatically streamed to Kafka in real-time!
-- Use this for: analytics pipelines, search index updates,
-- event-driven architectures, audit logs
```

## Feature 6: Online Schema Changes

Unlike MySQL/PostgreSQL where `ALTER TABLE` on a large table can lock it for minutes or hours, CockroachDB performs schema changes **online** — no downtime, no locks:

```sql
-- This does NOT lock the table, even if it has billions of rows!
ALTER TABLE subscribers ADD COLUMN address STRING;
ALTER TABLE subscribers ADD COLUMN verified BOOL DEFAULT false;
CREATE INDEX idx_subscribers_phone ON subscribers (phone);

-- CockroachDB applies schema changes in the background
-- Reads and writes continue normally during the change
```

---

# Chapter 5: CockroachDB Architecture Diagrams

## Single-Region 3-Node Cluster

```
  SINGLE-REGION CLUSTER (most common starting point)

  ┌─────────────────────── Region: ap-south-1 ───────────────────────┐
  │                                                                   │
  │  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
  │  │   NODE 1     │    │   NODE 2     │    │   NODE 3     │       │
  │  │   AZ: a      │    │   AZ: b      │    │   AZ: c      │       │
  │  │              │    │              │    │              │       │
  │  │  Port 26257  │    │  Port 26257  │    │  Port 26257  │       │
  │  │  (SQL)       │    │  (SQL)       │    │  (SQL)       │       │
  │  │              │    │              │    │              │       │
  │  │  Port 8080   │    │  Port 8080   │    │  Port 8080   │       │
  │  │  (Admin UI)  │    │  (Admin UI)  │    │  (Admin UI)  │       │
  │  │              │    │              │    │              │       │
  │  │  Range 1 (L) │    │  Range 1 (F) │    │  Range 1 (F) │       │
  │  │  Range 2 (F) │    │  Range 2 (L) │    │  Range 2 (F) │       │
  │  │  Range 3 (F) │    │  Range 3 (F) │    │  Range 3 (L) │       │
  │  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘       │
  │         │                   │                   │               │
  │         └───────── Gossip Protocol ─────────────┘               │
  │              (nodes discover and talk to each other)             │
  └─────────────────────────────────────────────────────────────────┘

  L = Leaseholder (handles reads + coordinates writes)
  F = Follower (receives replicated writes via Raft)

  Load Balancer (HAProxy or AWS ALB) sits in front
  and distributes client connections across all 3 nodes
```

## Multi-Region 9-Node Cluster

```
  MULTI-REGION CLUSTER (for global applications)

  ┌─── Region: ap-south-1 (Mumbai) ────┐
  │  Node 1  │  Node 2  │  Node 3      │  ← Indian subscriber data lives here
  │  AZ: a   │  AZ: b   │  AZ: c      │    (low latency for India)
  └──────────┴──────────┴──────────────┘
              │
              │  Raft replication
              │  across regions
              │
  ┌─── Region: us-east-1 (Virginia) ───┐
  │  Node 4  │  Node 5  │  Node 6      │  ← US subscriber data lives here
  │  AZ: a   │  AZ: b   │  AZ: c      │    (low latency for US)
  └──────────┴──────────┴──────────────┘
              │
              │
  ┌─── Region: eu-west-3 (Paris) ──────┐
  │  Node 7  │  Node 8  │  Node 9      │  ← EU subscriber data lives here
  │  AZ: a   │  AZ: b   │  AZ: c      │    (GDPR compliance)
  └──────────┴──────────┴──────────────┘

  Total: 9 nodes, 3 regions, 3 AZs per region
  Survives: any node failure, any AZ failure, any region failure
  Replication: 3 copies minimum, spread across regions
```

---

# Chapter 6: Hands-On SQL Examples

## Getting Connected

```bash
# Connect using cockroach sql CLI
cockroach sql --insecure --host=localhost:26257

# OR connect using standard psql (PostgreSQL client)
psql "postgresql://root@localhost:26257/defaultdb?sslmode=disable"

# OR connect using any PostgreSQL GUI tool
# Host: localhost, Port: 26257, User: root, Database: defaultdb
```

## Creating a Database and Tables

```sql
-- Create a database for our telecom application
CREATE DATABASE telecom;
USE telecom;

-- Subscribers table
CREATE TABLE subscribers (
    subscriber_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name STRING NOT NULL,
    phone STRING UNIQUE NOT NULL,
    email STRING,
    plan_type STRING DEFAULT 'prepaid',
    balance DECIMAL(10,2) DEFAULT 0.00,
    region STRING NOT NULL,
    activated_at TIMESTAMP DEFAULT now(),
    is_active BOOL DEFAULT true
);

-- Call Detail Records (CDR)
CREATE TABLE call_records (
    cdr_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_id UUID REFERENCES subscribers(subscriber_id),
    call_type STRING NOT NULL,          -- 'voice', 'sms', 'data'
    destination STRING,
    duration_seconds INT DEFAULT 0,
    data_usage_mb DECIMAL(10,2) DEFAULT 0.00,
    cost DECIMAL(10,2),
    call_timestamp TIMESTAMP DEFAULT now()
);

-- Plans table (reference data)
CREATE TABLE plans (
    plan_id INT PRIMARY KEY,
    plan_name STRING NOT NULL,
    monthly_cost DECIMAL(10,2),
    data_limit_gb INT,
    voice_minutes INT,
    sms_limit INT
);

-- Create indexes for common queries
CREATE INDEX idx_cdr_subscriber ON call_records (subscriber_id);
CREATE INDEX idx_cdr_timestamp ON call_records (call_timestamp DESC);
CREATE INDEX idx_subscribers_region ON subscribers (region);
CREATE INDEX idx_subscribers_phone ON subscribers (phone);
```

## Inserting Data

```sql
-- Insert plans
INSERT INTO plans VALUES
    (1, 'Basic',    199.00,  1,  100, 100),
    (2, 'Standard', 499.00,  3,  500, 500),
    (3, 'Premium',  999.00, 10, 1000, 1000),
    (4, 'Unlimited',1499.00, -1, -1,  -1);

-- Insert subscribers
INSERT INTO subscribers (name, phone, email, plan_type, balance, region)
VALUES
    ('Venkat Kumar',   '+91-9876543210', 'venkat@email.com',  'postpaid', 1500.00, 'ap-south-1'),
    ('Meena Sharma',   '+91-9876543211', 'meena@email.com',   'prepaid',  350.00,  'ap-south-1'),
    ('Kishore Reddy',  '+91-9876543212', 'kishore@email.com', 'postpaid', 2200.00, 'ap-south-1'),
    ('John Smith',     '+1-5551234567',  'john@email.com',    'postpaid', 800.00,  'us-east-1'),
    ('Marie Dupont',   '+33-612345678',  'marie@email.com',   'prepaid',  150.00,  'eu-west-3');

-- Insert some CDR records
INSERT INTO call_records (subscriber_id, call_type, destination, duration_seconds, cost)
SELECT subscriber_id, 'voice', '+91-9999999999', 120, 2.40
FROM subscribers WHERE name = 'Venkat Kumar';

INSERT INTO call_records (subscriber_id, call_type, destination, data_usage_mb, cost)
SELECT subscriber_id, 'data', 'internet', 256.50, 0.00
FROM subscribers WHERE name = 'Meena Sharma';
```

## Querying Data

```sql
-- Basic SELECT
SELECT name, phone, plan_type, balance FROM subscribers WHERE region = 'ap-south-1';

-- JOIN: Get subscriber details with their call records
SELECT s.name, s.phone, c.call_type, c.duration_seconds, c.cost, c.call_timestamp
FROM subscribers s
JOIN call_records c ON s.subscriber_id = c.subscriber_id
ORDER BY c.call_timestamp DESC;

-- Aggregation: Total spend per subscriber
SELECT s.name, 
       COUNT(c.cdr_id) as total_calls,
       SUM(c.cost) as total_cost,
       SUM(c.duration_seconds) as total_seconds,
       SUM(c.data_usage_mb) as total_data_mb
FROM subscribers s
LEFT JOIN call_records c ON s.subscriber_id = c.subscriber_id
GROUP BY s.name
ORDER BY total_cost DESC;

-- Window function: Rank subscribers by spending
SELECT name, total_cost,
       RANK() OVER (ORDER BY total_cost DESC) as spending_rank
FROM (
    SELECT s.name, COALESCE(SUM(c.cost), 0) as total_cost
    FROM subscribers s
    LEFT JOIN call_records c ON s.subscriber_id = c.subscriber_id
    GROUP BY s.name
) sub;

-- CTE: Find subscribers with no calls
WITH active_callers AS (
    SELECT DISTINCT subscriber_id FROM call_records
)
SELECT s.name, s.phone
FROM subscribers s
LEFT JOIN active_callers ac ON s.subscriber_id = ac.subscriber_id
WHERE ac.subscriber_id IS NULL;
```

## Transactions

```sql
-- ACID Transaction: Transfer balance between subscribers
BEGIN;
  UPDATE subscribers SET balance = balance - 100 WHERE name = 'Venkat Kumar';
  UPDATE subscribers SET balance = balance + 100 WHERE name = 'Meena Sharma';
  -- If either UPDATE fails, BOTH are rolled back (Atomicity)
COMMIT;

-- Transaction with error handling
BEGIN;
  INSERT INTO call_records (subscriber_id, call_type, destination, cost)
  SELECT subscriber_id, 'voice', '+91-1111111111', 5.00
  FROM subscribers WHERE name = 'Kishore Reddy';
  
  UPDATE subscribers SET balance = balance - 5.00 WHERE name = 'Kishore Reddy';
COMMIT;
-- This is Serializable isolation — strongest guarantee!
```

---

# Chapter 7: CockroachDB Admin Commands

## Cluster Health

```sql
-- Show all nodes in the cluster
SELECT node_id, address, is_live, locality 
FROM crdb_internal.gossip_nodes;

-- Check cluster ranges and replicas
SHOW RANGES FROM TABLE subscribers;

-- Show database sizes
SELECT table_name, 
       pg_size_pretty(range_size_bytes) as size,
       ranges, replicas
FROM [SHOW RANGES FROM DATABASE telecom];

-- Show running queries (like pg_stat_activity)
SELECT query, phase, node_id 
FROM [SHOW CLUSTER QUERIES];

-- Show active sessions
SHOW SESSIONS;

-- Show cluster settings
SHOW ALL CLUSTER SETTINGS;
```

## Backup and Restore

```sql
-- Full cluster backup to cloud storage
BACKUP INTO 'gs://my-bucket/backups/telecom'
WITH revision_history;

-- Backup specific database
BACKUP DATABASE telecom 
INTO 's3://telecom-backups/daily?AUTH=implicit';

-- Incremental backup (only changes since last backup)
BACKUP DATABASE telecom 
INTO LATEST IN 's3://telecom-backups/daily?AUTH=implicit';

-- Restore from backup
RESTORE DATABASE telecom 
FROM LATEST IN 's3://telecom-backups/daily?AUTH=implicit';

-- Point-in-time restore
RESTORE DATABASE telecom 
FROM LATEST IN 's3://telecom-backups/daily?AUTH=implicit'
AS OF SYSTEM TIME '2025-02-19 14:30:00';
```

## Node Management

```bash
# Start a CockroachDB node
cockroach start \
  --insecure \
  --store=node1-data \
  --listen-addr=localhost:26257 \
  --http-addr=localhost:8080 \
  --join=localhost:26257,localhost:26258,localhost:26259

# Check node status
cockroach node status --insecure --host=localhost:26257

# Decommission a node (safely remove from cluster)
cockroach node decommission 4 --insecure --host=localhost:26257
# CockroachDB moves all data off this node before removing it

# Access the Admin UI
# Open browser: http://localhost:8080
# Shows: cluster health, node status, SQL metrics, query stats
```

---

# Chapter 8: Real-World Companies Using CockroachDB

| Company | Industry | Why CockroachDB | Scale |
|---------|----------|----------------|-------|
| **DoorDash** | Food delivery | Zero-downtime scaling during peak orders | Millions of orders/day |
| **FanDuel** | Sports betting | Low-latency across 28 US states, Super Bowl spikes | 13+ million users |
| **Netflix** | Streaming | Global content metadata distribution | Global scale |
| **Santander** | Banking | Multi-region financial transactions | Enterprise banking |
| **Booking.com** | Travel | Global hotel inventory, always available | Global scale |
| **Hard Rock Digital** | Gaming | Multi-state betting compliance | Millions of bets |
| **SumUp** | Payments | 4 million+ merchants, PCI compliance | 4M+ merchants |
| **Riskified** | Fraud detection | 83% lower latency, 4.5x throughput | E-commerce scale |

---

# Chapter 9: CockroachDB Installation Quick Reference

## Single-Node (Local Development)

```bash
# Download CockroachDB (Linux)
curl https://binaries.cockroachdb.com/cockroach-v24.3.1.linux-amd64.tgz | tar -xz
sudo cp cockroach-v24.3.1.linux-amd64/cockroach /usr/local/bin/

# Start single-node cluster (for development/learning)
cockroach start-single-node --insecure --store=cockroach-data \
  --listen-addr=localhost:26257 --http-addr=localhost:8080 --background

# Connect
cockroach sql --insecure --host=localhost:26257
```

## 3-Node Local Cluster (for practicing)

```bash
# Node 1
cockroach start --insecure --store=node1 --listen-addr=localhost:26257 \
  --http-addr=localhost:8080 --join=localhost:26257,localhost:26258,localhost:26259 --background

# Node 2
cockroach start --insecure --store=node2 --listen-addr=localhost:26258 \
  --http-addr=localhost:8081 --join=localhost:26257,localhost:26258,localhost:26259 --background

# Node 3
cockroach start --insecure --store=node3 --listen-addr=localhost:26259 \
  --http-addr=localhost:8082 --join=localhost:26257,localhost:26258,localhost:26259 --background

# Initialize the cluster (only once!)
cockroach init --insecure --host=localhost:26257

# Verify
cockroach node status --insecure --host=localhost:26257
```

---

# Chapter 10: Quick Reference Summary

## Key Ports

| Port | Purpose |
|------|---------|
| **26257** | SQL connections (PostgreSQL protocol) |
| **8080** | Admin Web UI (dashboards, metrics) |

## Key SQL Differences from PostgreSQL

| PostgreSQL | CockroachDB Equivalent |
|-----------|----------------------|
| `SERIAL` | `INT DEFAULT unique_rowid()` or `UUID DEFAULT gen_random_uuid()` |
| `STORED PROCEDURES` | Not supported (use application logic) |
| `TRIGGERS` | Not supported (use CDC changefeeds) |
| `EXTENSIONS` | Not supported (PostGIS, etc.) |
| `ENUM` | Supported since v20.2 |
| `JSONB` | Fully supported |
| `ARRAY` | Fully supported |

## Essential cockroach CLI Commands

```bash
cockroach start            # Start a node
cockroach start-single-node # Start single-node dev cluster
cockroach init             # Initialize cluster (once)
cockroach sql              # Open SQL shell
cockroach node status      # Check all nodes
cockroach node decommission # Safely remove a node
cockroach quit             # Stop a node gracefully
cockroach debug zip        # Collect diagnostic info
cockroach version          # Show version
```

## Key Takeaways for the Exam

1. CockroachDB is a **distributed SQL database** — SQL + automatic sharding + replication
2. It uses the **PostgreSQL wire protocol** — connect with any PG client
3. Data is split into **Ranges (64 MB)** and replicated **3 times** using **Raft consensus**
4. Every node is **equal** — no master/slave. Any node can serve any query
5. **Serializable isolation** — strongest ACID guarantee, even across nodes
6. **Horizontal scaling** — add nodes to add capacity (no manual sharding)
7. **Multi-region** — pin data to regions for low latency and compliance
8. **Zero-downtime** — survives node, AZ, and region failures automatically
9. **Online schema changes** — ALTER TABLE never locks the table
10. **CDC (Changefeeds)** — stream changes to Kafka, S3, webhooks in real-time

---

*CockroachDB Student Guide — 2-Hour Classroom Training*
*Prepared for DBA Training Program*
