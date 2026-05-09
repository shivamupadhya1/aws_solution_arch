# WEEK 8: Cost Optimization + Exam Strategy
### AWS Solutions Architect Associate Study Notes

---

## WHY THIS MATTERS (Read This First)

Cost Optimization is 20% of the exam — the smallest domain but highly predictable. The same 8-10 concepts appear repeatedly. Master them and you've locked in 20% of your score.

Also: Week 8 is about **converting your knowledge into exam performance**. Knowing the content is not enough. You need exam strategy.

The exam is 65 questions, 130 minutes = 2 minutes per question. Most candidates fail not because they don't know the material, but because they:
1. Picked the "technically correct but not most cost-effective" answer
2. Didn't read ALL options before picking
3. Spent too long on hard questions and rushed easy ones

---

## SECTION 1: Cost Optimization Concepts

---

### 1.1 EC2 Cost Hierarchy (Review from Week 2)

```
MOST EXPENSIVE → LEAST EXPENSIVE
──────────────────────────────────
Dedicated Host    → Pay for physical server
On-Demand         → Pay per hour, no commitment
Savings Plans     → Commit to $/hr spend, 1-3 years, up to 66% off
Reserved Instances → Commit to specific instance, 1-3 years, up to 72% off
Spot Instances    → Bid on spare capacity, up to 90% off, can be interrupted
```

**The exam pattern for EC2 cost:**

| Workload | Best Pricing |
|----------|-------------|
| Production database, 24/7, 3 years | Standard Reserved Instance (3yr, all upfront) |
| Web app, 24/7, flexible instance type | Compute Savings Plan (1 or 3yr) |
| Batch job that runs 8 hours a day | On-Demand (no RI needed for < ~50% utilization) |
| Batch job that can be interrupted | Spot Instances |
| Compliance: dedicated hardware | Dedicated Host |

---

### 1.2 Savings Plans vs Reserved Instances

| Feature | Reserved Instances | Savings Plans |
|---------|-----------------|--------------|
| Commitment | Specific instance type + region | $/hr spend commitment |
| Flexibility | Standard RI: none. Convertible RI: limited | Compute: any EC2, Fargate, Lambda |
| Max discount | 72% (Standard RI, 3yr) | 66% (Compute), 72% (EC2 Instance) |
| Sell unused | Yes (Standard RI on Marketplace) | No |
| AWS recommends | Legacy approach | Modern approach |

**Savings Plans types:**
- **Compute Savings Plans:** Most flexible. EC2 (any family, region, size, OS) + Fargate + Lambda.
- **EC2 Instance Savings Plans:** Specific instance family in specific region. More discount, less flexibility.
- **SageMaker Savings Plans:** Only for SageMaker ML instances.

---

### 1.3 Spot Instances — Deep Dive

**Spot Instances** use unused AWS EC2 capacity. Up to 90% cheaper than On-Demand. AWS can reclaim them with 2-minute warning.

```
Spot use cases:
  ✓ Big data analytics (Spark, Hadoop)
  ✓ Batch processing
  ✓ CI/CD build servers
  ✓ ML training jobs
  ✓ Stateless web workers (behind ASG)
  ✓ Image/video processing

Spot NOT suitable for:
  ✗ Databases (stateful, can't handle sudden termination)
  ✗ Anything that can't checkpoint/resume
  ✗ Real-time transactional workloads
```

**Spot Fleet:** A collection of Spot Instances (+ optionally On-Demand). Maintains target capacity, fills from cheapest pools.

**Spot strategies:**
| Strategy | How |
|----------|-----|
| `lowest-price` | Launch from cheapest Spot pool. Risk: one pool, higher interruption if that AZ/type gets claimed. |
| `diversified` | Spread across multiple pools. Lower interruption risk. |
| `capacity-optimized` | Launch where AWS has most spare capacity. Lowest interruption risk. |

> **Exam tip:** "Fault-tolerant" + "cheapest" = Spot Instances.

---

### 1.4 S3 Cost Optimization

**Choose the right storage class:**
```
Access frequency → Storage class decision:
  Frequent (daily)      → Standard
  Unknown               → Intelligent-Tiering (auto-optimizes)
  Monthly               → Standard-IA
  Rare (<once/quarter)  → One Zone-IA (if okay with single AZ)
  Archive, fast access  → Glacier Instant Retrieval
  Archive, hours okay   → Glacier Flexible Retrieval
  7+ year compliance    → Glacier Deep Archive
```

**S3 cost levers:**
- Lifecycle policies (auto-transition to cheaper classes)
- S3 Intelligent-Tiering (no retrieval fees, auto-moves objects)
- Multipart upload (required for large files, more reliable = no retransmission cost)
- S3 Transfer Acceleration costs extra — only enable if needed
- **GET request costs add up** — use CloudFront caching to reduce S3 requests

---

### 1.5 RDS Cost Optimization

```
Running costs:
  ✓ DB instance hours (biggest cost)
  ✓ Multi-AZ (2x instance cost)
  ✓ Storage (per GB/month)
  ✓ I/O requests (for gp2/io1)
  ✓ Backup storage > 100% of DB size
  ✓ Data transfer out

Cost reduction:
  → Use Reserved Instances for RDS (1yr or 3yr)
  → Stop RDS instances in dev/test when not in use
  → Use gp3 instead of io1 (often cheaper for same IOPS)
  → Use Aurora Serverless for intermittent workloads (pay per ACU-second)
  → Delete automated backup retention if not needed (or reduce from 35 to 7 days)
```

---

### 1.6 AWS Cost Explorer

Cost Explorer lets you visualize and analyze your AWS spending.

```
Key views:
  → Monthly cost by service
  → Daily cost trend (spot anomalies)
  → Cost by tag (which team/project is spending)
  → Reserved Instance utilization (are you using what you paid for?)
  → Reserved Instance coverage (which On-Demand could be RIs?)
  → Savings Plans recommendations (what should you commit to?)
```

**Cost allocation tags:** Tag your resources (Owner, Project, Environment) → Cost Explorer can group costs by tag.

---

### 1.7 AWS Budgets

Set cost/usage thresholds and get alerted before you exceed them.

```
Budget types:
  Cost budget    → Alert when cost exceeds $X
  Usage budget   → Alert when service usage exceeds X hours
  RI utilization → Alert when RI utilization drops below 80%
  Savings Plans  → Alert on SP utilization/coverage

Alert actions:
  → Email/SNS notification
  → Apply IAM policy (restrict spending — advanced use case)
```

> **Exam tip:** "Alert when projected cost exceeds $500 this month" → AWS Budgets. "See historical cost breakdown" → Cost Explorer.

---

### 1.8 AWS Trusted Advisor

Automated best-practices checker across 5 categories:

```
Categories:
  ✓ Cost Optimization    (underutilized EC2, idle RDS, unattached EBS)
  ✓ Performance          (over-utilized EC2, high-latency EBS)
  ✓ Security             (open security groups, MFA on root, public S3 buckets)
  ✓ Fault Tolerance      (no Multi-AZ, no backups, low Spot limits)
  ✓ Service Limits       (approaching EC2, VPC, IAM limits)

Free tier: 7 core checks (basic security + service limits)
Business/Enterprise support: ALL checks + API access + weekly notifications
```

---

### 1.9 AWS Compute Optimizer

Uses ML to recommend right-sizing for EC2, EBS, Lambda, and ECS on Fargate.

```
Analyzes CloudWatch metrics from past 14 days
Recommends: "Your m5.xlarge is only using 5% CPU on average.
             Consider downsizing to m5.large (same performance, lower cost)."

Also analyzes: EBS volumes, Lambda memory settings, ECS on Fargate
```

---

## SECTION 2: Well-Architected Framework

---

### 2.1 The Six Pillars

The AWS Well-Architected Framework has 6 pillars. The exam tests that you know which pillar a design choice belongs to.

| Pillar | What It's About | Key Principle |
|--------|---------------|--------------|
| **Operational Excellence** | Run and monitor systems | Automate, make small reversible changes |
| **Security** | Protect data and systems | Least privilege, defense in depth |
| **Reliability** | Recover from failures | Multi-AZ, Multi-Region, backups, retries |
| **Performance Efficiency** | Use resources efficiently | Right-sizing, serverless, caching |
| **Cost Optimization** | Minimize spending | Right pricing model, eliminate waste |
| **Sustainability** | Minimize environmental impact | Use efficient services, minimize idle resources |

---

### 2.2 Design Principles You Must Know

**Reliability:**
- Design for failure — assume everything fails
- Horizontal scaling (many small > one big)
- Stop guessing capacity — use Auto Scaling
- Test failure scenarios (Game Days)
- Use Multi-AZ for HA

**Performance Efficiency:**
- Use managed services (less undifferentiated heavy lifting)
- Use caching (CloudFront, ElastiCache, DAX)
- Use databases suited to the use case (don't use RDS for everything)
- Use serverless where possible

**Cost Optimization:**
- Measure efficiency (cost per request, cost per user)
- Avoid paying for unused capacity
- Use the managed cloud services (cheaper than doing it yourself on EC2)
- Use Reserved Instances / Savings Plans for steady workloads

---

## SECTION 3: Additional Services Quick Reference

---

### 3.1 Migration Services

| Service | What It Does |
|---------|-------------|
| **AWS DMS** (Database Migration Service) | Migrate databases to AWS. Continuous replication during migration. Heterogeneous (Oracle → PostgreSQL) or homogeneous (MySQL → MySQL). |
| **AWS SCT** (Schema Conversion Tool) | Convert DB schema from one engine to another (Oracle → PostgreSQL) |
| **AWS MGN** (Application Migration Service) | Lift-and-shift EC2 migrations. Replicates servers to AWS with minimal downtime. |
| **AWS Snowball** | Physical device to move large datasets (TBs) to AWS. Avoid slow internet transfer. |
| **AWS Snowball Edge** | Snowball + compute (run Lambda on device). For edge computing + data transfer. |
| **AWS Snowmobile** | 100 PB physical truck. For exabyte-scale migration. |

**Migration data size → which tool:**
```
< 1 GB   → AWS CLI / direct upload
1-10 TB  → Direct Connect or S3 transfer
> 10 TB  → Snowball
> 10 PB  → Snowmobile
```

---

### 3.2 Infrastructure as Code

| Service | What It Does |
|---------|-------------|
| **CloudFormation** | AWS-native IaC. Deploy resources via JSON/YAML templates. |
| **AWS CDK** | Define CloudFormation in Python/TypeScript/Java. Compiled to CF templates. |
| **Elastic Beanstalk** | PaaS. Upload code → AWS manages EC2, ASG, ALB, RDS. You just deploy code. |
| **AWS SAM** | Serverless Application Model. Simplifies CloudFormation for Lambda/API GW. |
| **Systems Manager (SSM)** | Operational management. Run commands, patch, parameter store. |

---

### 3.3 Analytics Services

| Service | What It Does |
|---------|-------------|
| **Athena** | SQL queries on S3 data. Serverless. Pay per query. |
| **Glue** | ETL service. Extract, Transform, Load data. Data catalog. |
| **EMR** | Managed Hadoop/Spark clusters. Big data processing. |
| **QuickSight** | BI dashboard tool. Visualize data from Athena, Redshift, S3. |
| **OpenSearch** (Elasticsearch) | Search and analytics engine. Log analysis, full-text search. |
| **Lake Formation** | Set up data lake in S3 with governance. |

---

### 3.4 Application Services

| Service | What It Does |
|---------|-------------|
| **SES** (Simple Email Service) | Send transactional/marketing emails at scale |
| **Pinpoint** | User engagement: push notifications, SMS, email campaigns |
| **AppSync** | Managed GraphQL API |
| **Amplify** | Full-stack web/mobile app framework on AWS |
| **Cognito** | Auth for YOUR app users (User Pools=auth, Identity Pools=AWS creds) |
| **ACM** (Certificate Manager) | Free SSL/TLS certificates for ALB, CloudFront, API Gateway |

---

## SECTION 4: Exam Strategy

---

### 4.1 The 2-Minute Rule

You have 2 minutes per question. If you're stuck after 90 seconds:
1. Eliminate obviously wrong answers
2. Mark for review
3. Make your best guess and move on
4. Come back at the end

**Do not spend 5 minutes on one question.** You'll run out of time.

---

### 4.2 How to Read Exam Questions

Every SAA question has this structure:
```
CONTEXT: "A company has a web application..."
REQUIREMENT: "...that needs to be highly available across 2 regions."
CONSTRAINT: "The solution must be cost-effective."
QUESTION: "Which solution meets these requirements?"
```

**Step 1:** Find the KEY WORDS in the requirement and constraint.
**Step 2:** Eliminate answers that don't satisfy the requirement.
**Step 3:** Among remaining answers, pick the one that best satisfies the constraint.

**Most common key words:**

| Key Word | What To Pick |
|----------|-------------|
| "most cost-effective" | Cheapest option that still works |
| "highly available" | Multi-AZ at minimum |
| "fault-tolerant" | Can survive component failures, often Spot-friendly |
| "minimum operational overhead" | Managed services (not self-managed on EC2) |
| "minimum latency" | CloudFront, ElastiCache, Aurora, DAX |
| "existing on-premises" | Usually Hybrid, Direct Connect, Storage Gateway |
| "lift and shift" | EC2, ECS — minimal code change |
| "re-architect" | Lambda, serverless, managed services |
| "data residency compliance" | Single-region, no cross-region data movement |
| "no downtime migration" | DMS with CDC (continuous replication) |

---

### 4.3 Elimination Strategy

For 4-option multiple choice, you can usually eliminate 2 quickly:
```
Option A: "Store session data in EC2 instance local disk"
  → EC2 local disk = ephemeral, not shared → ELIMINATE (ASG will lose sessions on scale)

Option B: "Store session data in RDS"
  → RDS can do it but expensive and slow for session data → WEAK

Option C: "Store session data in ElastiCache Redis"
  → Perfect for session data → KEEP

Option D: "Store session data in S3"
  → S3 is object storage, not suitable for frequent small reads/writes → ELIMINATE

Answer: C
```

---

### 4.4 Classic Multi-Select Questions

Some questions have 2 correct answers out of 5. Strategy:
1. Find answers you're 100% sure are correct → pick them
2. If you're not sure about the 2nd answer, pick the one that makes the most architectural sense with the first answer

---

### 4.5 Practice Exam Schedule (Final 2 Weeks)

```
Day 1:  Practice exam #1 (Tutorials Dojo) → record score and weak areas
Day 2:  Review all wrong answers → re-read relevant week notes
Day 3:  Practice exam #2 → focus on weak areas from exam #1
Day 4:  Review + Anki cards
Day 5:  Practice exam #3
Day 6:  Full review day → read ALL cheat sheets in this document
Day 7:  Rest
Day 8:  Practice exam #4 (should be scoring 75%+)
Day 9:  Targeted review of worst-performing topic
Day 10: Light review + exam day prep
Day 11: YOUR EXAM DAY
```

**Target scores on Tutorials Dojo:**
- First attempt: 50-60% (normal, don't panic)
- After week of review: 65-70%
- Ready to book exam: 75%+ consistently
- Aim for 78%+ on practice = ~80%+ on real exam (real exam is slightly different format)

---

## SECTION 5: The Last 48 Hours

---

### Before the Exam

**48 hours before:**
- Read ALL cheat sheets in this document once
- Do ONE more practice exam
- Don't learn new topics — reinforce what you know

**Night before:**
- Review your "wrong answers" notebook
- Get 8 hours sleep (seriously — more valuable than 2 more hours of study)
- Set two alarms

**Day of exam:**
- Eat a proper meal
- Arrive 30 min early (Pearson VUE) or log in 30 min early (online proctored)
- Have your government ID ready
- For online: clear desk, no second monitors, no phone nearby

---

### During the Exam

```
✓ Read EVERY word of the question
✓ Read ALL 4 options before selecting
✓ Mark difficult questions for review — don't waste time
✓ Use process of elimination (cross out clearly wrong answers)
✓ "Most cost-effective" → usually Spot or Reserved or S3-IA
✓ "Minimum operational overhead" → managed service wins
✓ When two answers are both correct, pick the MORE AWS-native one
✓ Trust your preparation — don't second-guess on obvious ones
```

---

## SECTION 6: MEGA CHEAT SHEET — All Domains

---

### Design Secure Architectures (30%)

| Scenario | Answer |
|----------|--------|
| Least privilege IAM | Create specific policy, not AdministratorAccess |
| EC2 access to S3 without credentials in code | IAM Role on EC2 |
| External users need AWS credentials (mobile app) | Cognito Identity Pools |
| Audit who made API changes | CloudTrail |
| Detect threats in VPC traffic | GuardDuty |
| Find CVEs in EC2 | Inspector |
| Find PII in S3 | Macie |
| Encrypt S3 with your own key | SSE-KMS (CMK) |
| Force HTTPS on S3 | Bucket policy deny non-SecureTransport |
| Block IPs at subnet level | NACL |
| Block SQL injection at HTTP level | WAF |
| DDoS protection, basic | Shield Standard (free) |
| Prevent region services from being used | SCP in Organizations |
| Private EC2 → S3 without internet | S3 Gateway Endpoint |
| Connect on-prem privately | Direct Connect |
| Rotate DB credentials automatically | Secrets Manager |
| Store app config values | Parameter Store |

### Design Resilient Architectures (26%)

| Scenario | Answer |
|----------|--------|
| DB survives AZ failure | RDS Multi-AZ |
| DB survives region failure | Aurora Global Database |
| App servers survive AZ failure | ASG across multiple AZs + ALB |
| App survives region failure | Route 53 failover + multi-region |
| DR with lowest RPO | Multi-region active-active |
| DR with low cost, high RTO | Backup and restore (S3 + CloudFormation) |
| DR, moderate cost, moderate RTO | Pilot light (minimal resources always running) |
| Messaging between decoupled services | SQS |
| Decouple without losing messages | SQS (persists up to 14 days) |
| Multiple services receive same event | SNS Fan-out |
| Long-running workflow with error handling | Step Functions |
| S3 object protection from accidental delete | Versioning + MFA Delete |
| S3 cross-region DR | CRR (Cross-Region Replication) |
| Cache for read-heavy DB | ElastiCache |

### Design High-Performing Architectures (24%)

| Scenario | Answer |
|----------|--------|
| Cache static content globally | CloudFront |
| Serve global users with low latency | CloudFront (static) or Global Accelerator (dynamic/TCP) |
| Single-digit ms NoSQL | DynamoDB |
| Microsecond NoSQL | DynamoDB + DAX |
| Shared file system for many EC2s | EFS |
| High-throughput EC2 networking | Placement Group — Cluster |
| Big data sequential reads | EBS st1 (Throughput Optimized HDD) |
| High IOPS database | io2 Block Express |
| Read-heavy relational DB | RDS Read Replicas |
| Real-time data streaming | Kinesis Data Streams |
| Managed delivery to S3 | Kinesis Firehose |
| Cache database query results | ElastiCache Redis or Memcached |
| Query S3 data with SQL | Athena |
| Static IP for global load balancing | Global Accelerator |

### Design Cost-Optimized Architectures (20%)

| Scenario | Answer |
|----------|--------|
| 24/7 production database, long-term | Reserved Instance (1yr or 3yr) |
| Flexible compute, any family, cost savings | Compute Savings Plan |
| Fault-tolerant batch processing, cheapest | Spot Instances |
| Reduce storage cost for aging data | S3 Lifecycle Policy |
| Unknown S3 access pattern, cost-optimize | S3 Intelligent-Tiering |
| Serverless containers | Fargate (pay per task) |
| Check for underutilized resources | Trusted Advisor |
| Right-size EC2 with ML recommendation | Compute Optimizer |
| Alert on cost threshold | AWS Budgets |
| Analyze historical spending | Cost Explorer |
| Static website hosting, cheapest | S3 Static Website (no EC2) |
| Lambda cold start cost | Provisioned Concurrency (costs more but predictable) |
| Transfer large data to AWS cost-effectively | Direct Connect (for ongoing) or Snowball (one-time) |

---

## SECTION 7: 30-Day Sprint Plan (If Short on Time)

If you only have 30 days:

```
DAYS 1-5:    IAM + VPC (Week 1 notes + lab)
DAYS 6-10:   EC2 + ELB + ASG + S3 (Weeks 2-3 notes + labs)
DAYS 11-15:  Databases + Networking (Weeks 4-5 notes)
DAYS 16-20:  Security + Serverless (Weeks 6-7 notes + labs)
DAYS 21-25:  Practice exams (Tutorials Dojo x3) + review wrong answers
DAYS 26-28:  Targeted weak-area study from exam results
DAYS 29:     Full cheat sheet review + rest
DAY 30:      EXAM
```

Minimum viable study: Stephane Maarek's course (watch at 1.5x) + Tutorials Dojo practice exams + do your own hands-on labs for at least IAM/VPC/EC2/S3/Lambda.

---

*You've finished all 8 weeks of notes. Now go pass that exam.*

*Return to: [aws-saa-study-plan.md](aws-saa-study-plan.md)*
