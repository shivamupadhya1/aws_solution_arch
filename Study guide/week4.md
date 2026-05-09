# WEEK 4: Databases — RDS, Aurora, DynamoDB, ElastiCache
### AWS Solutions Architect Associate Study Notes

---

## WHY THIS MATTERS (Read This First)

Database questions are the second-most common type on the SAA exam. The key challenge: AWS gives you 8+ database services and you need to know **exactly which one to pick and why**.

The mental model:
- **Relational data with SQL** → RDS or Aurora
- **Millisecond scale, serverless SQL** → Aurora Serverless
- **Single-digit millisecond NoSQL** → DynamoDB
- **Cache layer** → ElastiCache (Redis or Memcached)
- **Analytics / Data warehouse** → Redshift

If the question says "highly available" → think Multi-AZ.
If the question says "read scaling" → think Read Replicas.

These are NOT the same thing. This is the #1 database mistake on the exam.

---

## SECTION 1: RDS — Relational Database Service

---

### 1.1 What Is RDS?

RDS is a managed service that runs common relational database engines. You get the DB without managing the OS, patching, backups, or hardware.

**Supported engines:**
- MySQL
- PostgreSQL
- MariaDB
- Oracle
- Microsoft SQL Server
- (Note: Aurora is separate — see Section 2)

**What AWS manages for you:**
- OS patching
- DB software patching
- Automated backups
- Hardware provisioning
- Monitoring (CloudWatch metrics built-in)

**What YOU still manage:**
- DB schema
- Index optimization
- Performance tuning
- Connection management

---

### 1.2 RDS Multi-AZ — High Availability

**This is the most tested RDS concept. Learn it exactly.**

Multi-AZ creates a **synchronous standby replica** in a different AZ.

```
MULTI-AZ SETUP:
────────────────────────────────────────────────────
AZ-a                            AZ-b
────────────────────────────────────────────────────
Primary DB ─── synchronous ──→ Standby DB
(reads + writes)    replication   (no reads served — just standby)

Your app → endpoint: mydb.xyz.us-east-1.rds.amazonaws.com
           (always points to PRIMARY)
```

**Failover process:**
```
1. Primary fails (hardware issue, AZ issue, or you do manual failover)
2. AWS detects failure (typically 1-2 minutes)
3. DNS endpoint automatically points to Standby
4. Standby promoted to Primary
5. New Standby created in original AZ
Total downtime: ~1-2 minutes
```

**Critical Multi-AZ facts:**
- Standby is in a DIFFERENT AZ (different failure domain)
- Standby does NOT serve read traffic (only writes from primary sync to it)
- The DNS endpoint is the same — your app reconnects automatically
- Backups are taken from standby (no I/O impact on primary)
- You are charged for both instances

> **Exam trap:** "Scale reads" → NOT Multi-AZ (standby doesn't serve reads). Use **Read Replicas** for read scaling.

---

### 1.3 RDS Read Replicas — Read Scaling

Read Replicas create **asynchronous copies** of your primary for read-heavy workloads.

```
READ REPLICA SETUP:
──────────────────────────────────────────────────
Primary DB ─── async replication ──→ Read Replica 1 (AZ-b)
           └── async replication ──→ Read Replica 2 (AZ-c)
           └── async replication ──→ Read Replica 3 (us-west-2) ← cross-region!

Your app:
  Writes → Primary endpoint
  Reads  → Read Replica endpoint (each has its own DNS)
```

**Read Replica facts:**
- Replication is **asynchronous** (slight lag — eventual consistency)
- Can have up to **5** read replicas for MySQL/MariaDB/PostgreSQL, **15** for Aurora
- Read Replicas have their **own endpoint DNS** (you update your app to use it)
- Can be in a **different region** (cross-region read replicas)
- Can be **promoted to standalone DB** (breaks replication — for DR or migration)
- Read Replicas of Read Replicas possible (adds more lag)

---

### 1.4 Multi-AZ vs Read Replica — Side by Side

| Feature | Multi-AZ | Read Replica |
|---------|---------|-------------|
| Purpose | **High availability** | **Read scaling** |
| Replication | Synchronous | Asynchronous |
| Standby serves traffic | No | Yes (reads only) |
| Can promote to primary | Yes (automatic failover) | Yes (manual, breaks replication) |
| Number | 1 standby | Up to 5 (MySQL), 15 (Aurora) |
| Cross-region | Yes (Multi-AZ option) | Yes |
| Failover | Automatic | Manual |
| Use when | "Highly available" "failover" | "Read-heavy" "offload reports" |

---

### 1.5 RDS Backups

**Automated Backups (default):**
- Daily snapshots of full DB + transaction logs
- Retention: 1-35 days
- Stored in S3 (no extra cost beyond S3 standard pricing)
- Point-in-time recovery to any second within retention window
- Enabled by default

**Manual Snapshots:**
- You take them explicitly
- Retained until you delete them (don't expire)
- Can share with other AWS accounts
- Can copy to another region for DR

```
POINT-IN-TIME RESTORE:
  9:00 AM — daily snapshot
  9:00 AM - 1:00 PM — transaction logs captured
  1:30 PM — accidental data deletion
  
  You can restore to 1:28 PM (any second within retention window)
  → Creates a NEW RDS instance (not in-place restore)
```

---

### 1.6 RDS Security

**Encryption:**
- Encrypt at rest: Enable at creation time using KMS (can't enable on existing unencrypted DB)
- To encrypt existing unencrypted DB:
  1. Take snapshot
  2. Copy snapshot with encryption enabled
  3. Restore from encrypted snapshot
  4. Switch app to new encrypted instance

**Network security:**
- RDS should always be in **private subnets** (never public)
- Security Group controls which EC2s can connect
- Connect via EC2 application server — never expose DB to internet

**IAM Database Authentication:**
- Instead of username/password, use IAM tokens
- Supported for: MySQL and PostgreSQL on RDS
- Token is valid for 15 minutes, generated via AWS API call

---

## SECTION 2: Amazon Aurora

---

### 2.1 Aurora vs RDS — Key Differences

Aurora is AWS's cloud-optimized database. It's compatible with MySQL and PostgreSQL (you can use the same drivers/tools) but it's built differently underneath.

| Feature | RDS (MySQL/PostgreSQL) | Aurora |
|---------|----------------------|--------|
| Storage | Provisioned EBS | Distributed, auto-scaling (10 GB increments, up to 128 TB) |
| Replication | Up to 5 read replicas | Up to 15 read replicas |
| Failover speed | ~1-2 minutes | ~30 seconds |
| Storage redundancy | Multi-AZ standby | 6 copies across 3 AZs automatically |
| Cost | Standard | ~20% more than RDS, but better performance |
| Performance | Baseline MySQL/PostgreSQL | 5x MySQL, 3x PostgreSQL performance |
| Global Database | Manual setup | Built-in global replication |

---

### 2.2 Aurora Storage Architecture

This is what makes Aurora special — and why it's on the exam.

```
AURORA STORAGE ARCHITECTURE:
─────────────────────────────────────────────────
Aurora DB Instance (Primary)
    │
    └── Writes go to shared distributed storage cluster

Shared Storage Cluster:
  AZ-a: Storage node 1 + Storage node 2
  AZ-b: Storage node 3 + Storage node 4
  AZ-c: Storage node 5 + Storage node 6

→ 6 copies across 3 AZs
→ Can tolerate: 2 AZ failures for reads, 1 AZ failure for writes
→ Storage auto-grows (no manual resize)
→ Self-healing: bad storage blocks automatically detected and fixed
```

All Aurora Replicas share the **same storage** — no replication lag for data, only tiny lag for cache state.

---

### 2.3 Aurora Endpoints

Aurora gives you multiple endpoints:

```
CLUSTER ENDPOINT (Writer):
  mydb.cluster-xyz.us-east-1.rds.amazonaws.com
  → Always points to the PRIMARY instance
  → Use for writes

READER ENDPOINT:
  mydb.cluster-ro-xyz.us-east-1.rds.amazonaws.com
  → Load balances across ALL read replicas
  → Use for reads

INSTANCE ENDPOINT:
  mydb-instance-1.xyz.us-east-1.rds.amazonaws.com
  → Points to a specific instance
  → Use when you need to control exactly which instance

CUSTOM ENDPOINT (optional):
  Define a subset of instances (e.g., bigger instances for analytics)
```

---

### 2.4 Aurora Global Database

Replicates your Aurora cluster across multiple AWS regions with sub-second replication.

```
PRIMARY REGION (us-east-1):
  Writer instance + Read replicas
      │
      └── Replication lag < 1 second
          │
SECONDARY REGION (eu-west-1):
  Read-only replicas (up to 16)
  
DISASTER RECOVERY:
  If us-east-1 goes down → promote eu-west-1 to primary in < 1 minute
  RPO: ~1 second (sub-second replication)
  RTO: < 1 minute (promotion)
```

---

### 2.5 Aurora Serverless

Aurora Serverless automatically scales capacity up and down based on application needs.

```
Low traffic (2 AM):  Aurora scales down to minimum (0.5 ACUs)
High traffic (9 AM): Aurora scales up instantly as needed (up to 128 ACUs)

ACU = Aurora Capacity Unit (combo of CPU + RAM)
```

**When to use:**
- Infrequent, intermittent workloads
- Dev/test databases
- Multi-tenant apps with variable load
- When you don't know the right instance size

**Aurora Serverless v2:** Current version. Scales continuously (not in fixed steps). Supports Multi-AZ.

---

## SECTION 3: DynamoDB

---

### 3.1 What Is DynamoDB?

DynamoDB is a fully managed **NoSQL** database. Key-value and document store.

```
DynamoDB characteristics:
  ✓ Single-digit millisecond performance at any scale
  ✓ Automatically replicated across 3 AZs (built-in HA)
  ✓ Infinite scale (handles millions of requests per second)
  ✓ No servers to manage
  ✓ Supports key-value and document (JSON) data models
```

**Basic structure:**
```
Table: Users
┌─────────────┬──────────┬──────────────────────────────────────┐
│ UserID (PK) │ SortKey  │ Attributes                           │
├─────────────┼──────────┼──────────────────────────────────────┤
│ user-001    │ profile  │ name: "Alice", age: 30, city: "NYC"  │
│ user-001    │ orders   │ items: [...], total: 250             │
│ user-002    │ profile  │ name: "Bob", ...                     │
└─────────────┴──────────┴──────────────────────────────────────┘
PK = Partition Key (required)
SK = Sort Key (optional, enables range queries within a partition)
```

---

### 3.2 DynamoDB Capacity Modes

| Mode | How It Works | Use Case |
|------|-------------|---------|
| **Provisioned** | You set Read Capacity Units (RCUs) and Write Capacity Units (WCUs). You pay for provisioned capacity. | Predictable traffic |
| **On-Demand** | DynamoDB auto-scales with traffic. You pay per request. | Unpredictable/spiky traffic |

**RCU and WCU (for exam):**
```
1 RCU = 1 strongly consistent read/second of item up to 4 KB
      = 2 eventually consistent reads/second of item up to 4 KB

1 WCU = 1 write/second of item up to 1 KB

Example: Item is 8 KB, need 10 strong reads/sec
  → 2 RCUs per read × 10 reads = 20 RCUs needed
```

---

### 3.3 DynamoDB Indexes

#### Primary Key Types
- **Simple (Partition Key only):** Each item uniquely identified by PK alone.
- **Composite (Partition Key + Sort Key):** Multiple items with same PK, sorted by SK. Enables range queries.

#### GSI — Global Secondary Index
- Different partition key than the table
- Can project any attributes
- Has its own read/write capacity
- Can be created after table creation
- **Use when:** You need to query on a non-PK attribute frequently

#### LSI — Local Secondary Index
- Same partition key, different sort key
- Must be created at **table creation** (can't add later)
- Shares read/write capacity with table
- **Use when:** You need to sort/range query the same partition by a different attribute

```
Example: Orders table
  PK: CustomerID, SK: OrderDate

LSI: Same PK (CustomerID), different SK (OrderTotal)
  → "Get all orders for customer X sorted by amount"

GSI: Different PK (ProductID), SK: OrderDate
  → "Get all orders for a specific product across all customers"
```

---

### 3.4 DynamoDB DAX — In-Memory Cache

DynamoDB Accelerator (DAX) is an in-memory cache specifically for DynamoDB.

```
WITHOUT DAX:
  App → DynamoDB → response in ~single-digit ms

WITH DAX:
  App → DAX (in-memory cache) → response in microseconds
         ↓ (cache miss)
         DynamoDB
```

- Reduces read latency from milliseconds to microseconds
- Compatible with DynamoDB API — no code change needed
- Deployed in your VPC
- Clustered (HA) — 3+ nodes recommended
- **Use for:** Read-heavy, hot-key access patterns

---

### 3.5 DynamoDB Streams

Records every change (INSERT/MODIFY/DELETE) to a DynamoDB table as a stream.

```
DynamoDB Table change
    ↓
DynamoDB Stream
    ↓
Lambda function (processes changes)
    ↓
Action: sync to another DB, invalidate cache, trigger notification, etc.
```

**Use cases:**
- Cross-region replication (Global Tables use Streams)
- Real-time analytics
- Audit log of changes
- Event-driven architectures

---

### 3.6 DynamoDB Global Tables

Multi-region, multi-active replication.

```
us-east-1 Table ←── active-active replication ──→ eu-west-1 Table
                ←── active-active replication ──→ ap-southeast-1 Table

Any region can accept writes. Conflicts resolved by last-writer-wins.
Replication within seconds.
```

- Requires DynamoDB Streams enabled
- Applications write to closest region (low latency globally)

---

## SECTION 4: ElastiCache

---

### 4.1 What Is ElastiCache?

A managed in-memory caching service. Reduces load on databases by caching frequent queries in RAM.

```
WITHOUT CACHE:
  User request → App → Database (slow, expensive query)
  Every user request hits the database

WITH CACHE:
  User request → App → ElastiCache (cache hit — response in < 1ms)
                               ↓ (cache miss — first time)
                              Database → App stores result in cache → response
```

---

### 4.2 Redis vs Memcached

**This is the most tested ElastiCache question. Know when to pick each.**

| Feature | Redis | Memcached |
|---------|-------|-----------|
| Data structures | Strings, Hashes, Lists, Sets, Sorted Sets, Geospatial | Strings only |
| Persistence | Yes (RDB snapshots, AOF logs) | No |
| Replication | Yes (Multi-AZ) | No |
| Backup/restore | Yes | No |
| Pub/Sub | Yes | No |
| Cluster mode | Yes (horizontal partitioning) | Yes (simple sharding) |
| Threads | Single-threaded | Multi-threaded |
| Use case | Sessions, real-time leaderboards, pub/sub, complex data | Simple caching, multi-thread horizontal scaling |

```
CHOOSE REDIS WHEN:
  ✓ Need persistence (data survives restart)
  ✓ Need Multi-AZ with failover
  ✓ Complex data structures (sorted sets for leaderboards)
  ✓ Pub/Sub messaging
  ✓ Backup and restore required
  ✓ Session store (needs durability)

CHOOSE MEMCACHED WHEN:
  ✓ Pure simple object caching
  ✓ Need to scale out horizontally with multiple cores
  ✓ No persistence needed
  ✓ Simplest possible caching
```

---

### 4.3 Caching Strategies

**Lazy Loading (Cache-Aside):**
```
1. App requests data
2. Check cache
3a. Cache hit → return cached data
3b. Cache miss → query DB → store result in cache → return data

PRO: Cache only contains requested data (no wasted space)
CON: Initial requests are slow (cold start). Stale data possible.
```

**Write Through:**
```
On every write to DB:
  1. Write to DB
  2. Write to cache (immediately)

PRO: Cache always up to date
CON: Write latency doubles. May cache data that's never read.
```

**TTL (Time To Live):**
- Set expiry on cached items
- Forces refresh after TTL expires
- Balances freshness vs performance

---

## SECTION 5: Other Databases (Quick Reference)

### Redshift — Data Warehouse
- Petabyte-scale analytics
- Column-based storage (optimized for aggregations)
- Connect using SQL (PostgreSQL-compatible)
- NOT for OLTP (transactions) — for OLAP (analytics)
- **Redshift Spectrum:** Query S3 data directly from Redshift
- **Use when:** "Business intelligence", "analytics", "complex aggregate queries over historical data"

### Neptune — Graph Database
- Highly connected datasets
- Social networks, recommendation engines, fraud detection, knowledge graphs
- Query language: Gremlin or SPARQL
- **Use when:** "Relationships between entities", "social graph", "fraud patterns"

### DocumentDB — MongoDB-Compatible
- Managed MongoDB-compatible document database
- **Use when:** "MongoDB" appears in the question, or you need to migrate MongoDB workloads to AWS

### Amazon Keyspaces — Cassandra-Compatible
- Managed Apache Cassandra
- **Use when:** "Cassandra" appears in the question

### Timestream — Time-Series Database
- Purpose-built for time-series data
- IoT sensor data, application metrics, DevOps monitoring
- **Use when:** "time-series data", "IoT metrics", "log data over time"

---

## SECTION 6: WEEK 4 LAB

---

### Lab: RDS Multi-AZ + Read Replica + Failover Test

**Step 1: Create RDS MySQL with Multi-AZ**
```
RDS → Create database
  Engine: MySQL
  Template: Free tier (note: Free tier = no Multi-AZ. For lab, use Dev/Test)
  DB instance: db.t3.micro
  Master username: admin
  Multi-AZ: Yes (if using Dev/Test template)
  VPC: your VPC
  Subnet group: private subnets (AZ-a + AZ-b)
  Public access: No
  Security group: allow MySQL (3306) from EC2 security group
```

**Step 2: Create Read Replica**
```
Select your DB instance → Actions → Create read replica
  Region: same region, different AZ
  Instance class: db.t3.micro
```

**Step 3: Connect from EC2 (Bastion)**
```bash
# SSH to EC2 in same VPC
# Install MySQL client
sudo yum install -y mysql

# Connect to primary
mysql -h <rds-endpoint> -u admin -p
SHOW DATABASES;
CREATE DATABASE testdb;
USE testdb;
CREATE TABLE users (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(100));
INSERT INTO users (name) VALUES ('Alice'), ('Bob');
SELECT * FROM users;

# Connect to read replica (same query, read-only)
mysql -h <read-replica-endpoint> -u admin -p
USE testdb;
SELECT * FROM users;  # Should show Alice and Bob
```

**Step 4: Test Multi-AZ failover**
```
RDS Console → Select your DB → Actions → Reboot
  → Check "Reboot with failover"
  → Confirm

Monitor: The endpoint stays the same but now points to the standby
Typical downtime: 60-120 seconds
```

**Step 5: Cleanup**
```
Delete Read Replica first
Delete Primary DB (disable final snapshot for lab cleanup)
```

---

## SECTION 7: EXAM QUICK REFERENCE

### Database Selection Cheat Sheet

| Scenario | Answer |
|----------|--------|
| Relational DB, HA, auto failover | RDS with Multi-AZ |
| Scale reads for reporting | RDS Read Replica |
| Relational DB, maximum performance | Aurora |
| Variable workload, don't know size | Aurora Serverless |
| Global DB, < 1s replication | Aurora Global Database |
| NoSQL, single-digit ms | DynamoDB |
| NoSQL, microsecond latency (cache) | DynamoDB + DAX |
| Cache frequent DB queries | ElastiCache Redis or Memcached |
| Cache + need persistence/failover | ElastiCache Redis |
| Simple caching, no persistence | ElastiCache Memcached |
| Analytics, petabyte scale, SQL | Redshift |
| Graph relationships | Neptune |
| MongoDB-compatible | DocumentDB |
| Time-series IoT data | Timestream |
| Highly available, failover < 30s | Aurora (vs RDS ~2 min) |
| "Multi-AZ" for high availability | Standby replica, NOT for reads |
| "Read Replica" for performance | Read scaling only, NOT HA |

---

*Next: [week5-networking-cdn.md](week5-networking-cdn.md)*
