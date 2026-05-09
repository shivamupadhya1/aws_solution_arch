# AWS Solutions Architect Associate (SAA-C03)
### Complete Study Plan — 8 Weeks to Certification

---

## THE GOAL

**Exam:** AWS Certified Solutions Architect – Associate (SAA-C03)  
**Pass score:** 720 / 1000  
**Duration:** 130 minutes, ~65 questions  
**Format:** Multiple choice + Multiple response  
**Cost:** $150 USD  
**Book at:** https://aws.amazon.com/certification/certified-solutions-architect-associate/

---

## EXAM DOMAIN BREAKDOWN (Know This Before You Start)

| Domain | Weight | What It Tests |
|--------|--------|--------------|
| **Design Secure Architectures** | 30% | IAM, VPC, encryption, security services |
| **Design Resilient Architectures** | 26% | Multi-AZ, Auto Scaling, DR, failover |
| **Design High-Performing Architectures** | 24% | Caching, databases, networking, compute |
| **Design Cost-Optimized Architectures** | 20% | Right-sizing, Reserved vs Spot, S3 tiers |

> **Rule:** Security is the biggest domain. When in doubt on exam questions, pick the most secure, least-privilege option.

---

## YOUR 8-WEEK STUDY ROADMAP

```
WEEK 1 ── IAM + VPC (Foundations — NEVER skip this)
WEEK 2 ── EC2 + Elastic Load Balancer + Auto Scaling
WEEK 3 ── S3 + EBS + EFS + Storage Gateway
WEEK 4 ── RDS + Aurora + DynamoDB + ElastiCache
WEEK 5 ── Route 53 + CloudFront + API Gateway
WEEK 6 ── Security (KMS, CloudTrail, CloudWatch, WAF, Shield)
WEEK 7 ── Serverless (Lambda, SQS, SNS, Step Functions, EventBridge)
WEEK 8 ── Cost Optimization + Practice Exams + Weak Area Review
```

---

## DAILY STUDY SCHEDULE (Recommended)

```
Monday     → Read theory notes (1.5 hrs) + do labs (1 hr)
Tuesday    → Review notes + Anki flashcards (45 min)
Wednesday  → Read theory notes (1.5 hrs) + do labs (1 hr)
Thursday   → Review notes + Anki flashcards (45 min)
Friday     → Mixed practice questions (1 hr) + review wrongs (30 min)
Saturday   → Full mock exam (2.5 hrs) + review every wrong answer
Sunday     → Rest OR light review of diagrams only
```

---

## RESOURCES (Prioritized — Don't Overwhelm Yourself)

### PRIMARY RESOURCES (Do All Of These)

| Resource | Type | Cost | Link |
|----------|------|------|------|
| **Stephane Maarek's SAA-C03 Course** | Video | ~$15 on Udemy sale | https://www.udemy.com/course/aws-certified-solutions-architect-associate-saa-c03/ |
| **Tutorials Dojo Practice Exams** | Practice Tests | ~$15 | https://portal.tutorialsdojo.com/courses/aws-certified-solutions-architect-associate-practice-exams/ |
| **AWS Free Tier Account** | Hands-on Labs | FREE | https://aws.amazon.com/free/ |
| **AWS Official SAA-C03 Exam Guide** | Reference | FREE | https://d1.awsstatic.com/training-and-certification/docs-sa-assoc/AWS-Certified-Solutions-Architect-Associate_Exam-Guide.pdf |

### SECONDARY RESOURCES (Use As Needed)

| Resource | Type | Cost | Notes |
|----------|------|------|-------|
| **Neal Davis's SAA Course** | Video | ~$15 | Good alternative to Maarek |
| **AWS Skill Builder** | Official Labs | FREE/paid | https://skillbuilder.aws/ |
| **A Cloud Guru** | Video + Labs | $40/mo | Expensive but great labs |
| **WhizLabs Practice Exams** | Practice Tests | ~$20 | More questions than Tutorials Dojo |
| **AWS Documentation** | Reference | FREE | Use for deep dives only |
| **r/AWSCertifications** | Community | FREE | https://www.reddit.com/r/AWSCertifications/ |

### FLASHCARD DECKS (Anki)
- Search "AWS SAA-C03 Anki" on AnkiWeb
- Or use: https://ankiweb.net/shared/decks/AWS

---

## WEEKLY BREAKDOWN — WHAT TO STUDY

### WEEK 1: IAM + VPC
**See:** `week1-iam-vpc.md`
- [ ] IAM Users, Groups, Roles, Policies
- [ ] STS, AssumeRole, Federation, SSO
- [ ] VPC, Subnets, Route Tables
- [ ] Internet Gateway, NAT Gateway, NAT Instance
- [ ] Security Groups vs NACLs
- [ ] VPC Peering, Transit Gateway, VPN, Direct Connect
- [ ] VPC Endpoints (Gateway vs Interface)
- **Lab:** Create a custom VPC with public/private subnets, bastion host, and NAT Gateway

### WEEK 2: EC2 + ELB + Auto Scaling
**See:** `week2-ec2-elb-autoscaling.md`
- [ ] EC2 instance types and families
- [ ] Pricing models (On-Demand, Reserved, Spot, Dedicated)
- [ ] AMIs, User Data, Instance Metadata
- [ ] ELB — ALB vs NLB vs CLB vs GWLB
- [ ] Auto Scaling Groups, Launch Templates, Scaling Policies
- [ ] Placement Groups (Cluster, Spread, Partition)
- **Lab:** Launch EC2 with ALB + Auto Scaling Group with target tracking policy

### WEEK 3: S3 + Storage
**See:** `week3-s3-storage.md`
- [ ] S3 storage classes (Standard, IA, Glacier, etc.)
- [ ] S3 lifecycle policies, versioning, replication
- [ ] S3 encryption (SSE-S3, SSE-KMS, SSE-C, client-side)
- [ ] S3 Access Control — bucket policies, ACLs, pre-signed URLs
- [ ] EBS volume types (gp2, gp3, io1, io2, st1, sc1)
- [ ] EFS vs EBS vs S3 comparison
- [ ] Storage Gateway (File, Volume, Tape)
- **Lab:** Host static website on S3, set up lifecycle policy, cross-region replication

### WEEK 4: Databases
**See:** `week4-databases.md`
- [ ] RDS — engines, Multi-AZ, Read Replicas, snapshots
- [ ] Aurora — Aurora vs RDS, Global Database, Serverless
- [ ] DynamoDB — partitions, GSI/LSI, DAX, streams
- [ ] ElastiCache — Redis vs Memcached
- [ ] Redshift — data warehouse
- [ ] Database Migration Service (DMS)
- **Lab:** Create RDS with Multi-AZ standby + Read Replica, test failover

### WEEK 5: Networking & Content Delivery
**See:** `week5-networking-cdn.md`
- [ ] Route 53 — record types (A, AAAA, CNAME, Alias)
- [ ] Route 53 routing policies (Simple, Weighted, Latency, Failover, Geolocation)
- [ ] CloudFront — distributions, origins, behaviors, cache policies
- [ ] CloudFront + S3 + OAC (Origin Access Control)
- [ ] API Gateway — REST vs HTTP vs WebSocket
- [ ] Global Accelerator vs CloudFront
- **Lab:** Set up CloudFront distribution in front of S3, enable HTTPS

### WEEK 6: Security + Monitoring
**See:** `week6-security-monitoring.md`
- [ ] KMS — CMKs, key policies, envelope encryption
- [ ] CloudTrail — management events vs data events, log file integrity
- [ ] CloudWatch — metrics, alarms, logs, dashboards, Container Insights
- [ ] AWS Config — rules, conformance packs, remediation
- [ ] AWS Organizations — SCPs, Organizational Units
- [ ] WAF, Shield (Standard vs Advanced), Firewall Manager
- [ ] GuardDuty, Inspector, Macie, Security Hub
- **Lab:** Set up CloudWatch alarm → SNS email on high CPU

### WEEK 7: Serverless + Application Integration
**See:** `week7-serverless-integration.md`
- [ ] Lambda — invocation types, concurrency, layers, versions/aliases
- [ ] SQS — Standard vs FIFO, visibility timeout, dead letter queue
- [ ] SNS — topics, subscriptions, fanout pattern
- [ ] EventBridge — rules, event buses, patterns
- [ ] Step Functions — Standard vs Express workflows, states
- [ ] Kinesis — Data Streams vs Firehose vs Analytics
- [ ] ECS + Fargate — task definitions, services, ECR
- **Lab:** Build serverless API: API Gateway → Lambda → DynamoDB

### WEEK 8: Cost Optimization + Exam Prep
**See:** `week8-cost-exam-prep.md`
- [ ] Cost Explorer, Budgets, Cost Allocation Tags
- [ ] Savings Plans vs Reserved Instances
- [ ] Spot Instances — Spot Fleet, interruption handling
- [ ] Trusted Advisor checks
- [ ] AWS Compute Optimizer
- [ ] Full-length practice exam (Tutorials Dojo) + analyze every wrong answer
- [ ] Revisit weakest domain from practice exam stats
- **Lab:** Use Cost Explorer to analyze hypothetical architecture cost

---

## HIGH-YIELD EXAM TIPS (Read These Every Week)

### The "MOST" Questions
Almost every exam question asks for the "MOST cost-effective", "MOST secure", or "MOST resilient" solution.

**Hierarchy to remember:**
```
COST:       Spot > Reserved > Savings Plans > On-Demand > Dedicated
RESILIENCE: Multi-Region > Multi-AZ > Single-AZ
SECURITY:   Least privilege always wins
```

### Classic Traps

| Trap | Correct Answer |
|------|---------------|
| "Highly available database" → RDS | Use **Multi-AZ** (not Read Replica — that's for read scaling) |
| "Serverless SQL" | **Aurora Serverless** |
| "Cheapest for infrequent access objects" | **S3 Standard-IA** (not Glacier — too slow) |
| "Global static content fast delivery" | **CloudFront** |
| "Single-digit millisecond database" | **DynamoDB + DAX** |
| "Lift-and-shift with minimum changes" | **EC2** (not Lambda) |
| "Share VPC across accounts" | **RAM (Resource Access Manager)** |
| "Block traffic by country" | **WAF Geo Match** or **CloudFront geo restriction** |
| "Encrypt data, you manage the key" | **SSE-C** or **SSE-KMS with CMK** |
| "Connect on-prem to AWS privately" | **Direct Connect** (not VPN — VPN goes over internet) |

---

## LAB ENVIRONMENT SETUP

### Step 1: Create Free Tier Account
1. Go to https://aws.amazon.com/free/
2. Create account with credit card (you won't be charged if you stay in Free Tier)
3. Set up **MFA** on root account immediately
4. Create an **IAM admin user** — never use root for daily work

### Step 2: Set Up Billing Alerts
```
AWS Console → Billing → Billing Preferences
→ Enable "Receive Free Tier Usage Alerts"
→ Enable "Receive Billing Alerts"

CloudWatch → Alarms → Create Alarm
→ Metric: Billing → Total Estimated Charge
→ Threshold: > $5
→ Send SNS email to yourself
```

### Step 3: Install AWS CLI
```bash
# Mac/Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Windows
# Download: https://awscli.amazonaws.com/AWSCLIV2.msi

# Configure
aws configure
# Enter: Access Key, Secret Key, Region (us-east-1), Output (json)
```

### Step 4: Create IAM User for Labs
```
IAM → Users → Add User
Name: "lab-admin"
Permissions: AdministratorAccess (for labs only — not for prod!)
Create Access Key → CLI access
```

---

## PRACTICE EXAM STRATEGY

**Week 4 onward:** Take one Tutorials Dojo topic exam per week  
**Week 7:** First full-length mock exam  
**Week 8:** Full mock exam every 2 days

**After every wrong answer, write this down:**
```
Question: [What did it ask?]
My answer: [What I picked]
Correct answer: [What it was]
Why I was wrong: [One sentence]
Key rule to remember: [One sentence]
```

This is how you convert practice exams into actual learning.

---

## CHEAT SHEET — SERVICES AT A GLANCE

```
COMPUTE
  EC2         → Virtual machines, full control
  Lambda      → Serverless functions, event-driven, max 15 min
  ECS/Fargate → Containers, serverless containers
  Elastic Beanstalk → PaaS, auto-managed EC2/ALB (no IaC needed)

STORAGE
  S3          → Object storage, unlimited scale
  EBS         → Block storage, attached to one EC2 (mostly)
  EFS         → Shared file storage, multiple EC2s, Linux only
  FSx         → Managed Windows file server (FSx for Windows) or Lustre (HPC)
  Glacier     → Archive, retrieval in minutes to hours

DATABASES
  RDS         → Managed relational DB (MySQL, PostgreSQL, Oracle, SQL Server, MariaDB)
  Aurora      → AWS-optimized relational, 5x faster than RDS MySQL, serverless option
  DynamoDB    → NoSQL, key-value + document, single-digit ms
  ElastiCache → In-memory cache (Redis for persistence, Memcached for simple cache)
  Redshift    → Data warehouse, analytics, petabyte scale
  Neptune     → Graph database
  DocumentDB  → MongoDB-compatible

NETWORKING
  VPC         → Your private network in AWS
  Route 53    → DNS service
  CloudFront  → CDN, global edge caching
  ELB         → Load balancer (ALB=HTTP, NLB=TCP, GWLB=appliances)
  Direct Connect → Dedicated private connection to AWS
  Transit Gateway → Hub connecting multiple VPCs and on-prem

SECURITY
  IAM         → Identity and access management
  KMS         → Key management, encryption
  Secrets Manager → Store, rotate DB passwords and API keys
  Parameter Store → Simpler, cheaper secrets (no auto-rotation)
  WAF         → Web application firewall (L7)
  Shield      → DDoS protection (Standard=free, Advanced=$3000/mo)
  GuardDuty   → Threat detection (ML-based)
  Macie       → PII detection in S3
  Inspector   → Vulnerability scanner for EC2/Lambda/ECR

MONITORING
  CloudWatch  → Metrics, logs, alarms, dashboards
  CloudTrail  → API call audit log (who did what, when)
  Config      → Resource configuration change tracking and compliance
  X-Ray       → Distributed tracing

INTEGRATION
  SQS         → Message queue, decoupling, async
  SNS         → Pub/sub notifications, fanout
  EventBridge → Event bus, event-driven architecture
  Step Functions → Serverless workflow orchestrator
  Kinesis     → Real-time data streaming

IaC & DEPLOYMENT
  CloudFormation → IaC (like Terraform but AWS-native)
  CDK         → CloudFormation via code (Python, TypeScript, Java)
  CodePipeline/CodeBuild/CodeDeploy → CI/CD on AWS
```

---

## YOUR STUDY FILES (Bookmark These)

| File | Topic |
|------|-------|
| [week1-iam-vpc.md](week1-iam-vpc.md) | IAM + VPC — Foundations |
| [week2-ec2-elb-autoscaling.md](week2-ec2-elb-autoscaling.md) | EC2, Load Balancers, Auto Scaling |
| [week3-s3-storage.md](week3-s3-storage.md) | S3, EBS, EFS, Storage Gateway |
| [week4-databases.md](week4-databases.md) | RDS, Aurora, DynamoDB, ElastiCache |
| [week5-networking-cdn.md](week5-networking-cdn.md) | Route 53, CloudFront, API Gateway |
| [week6-security-monitoring.md](week6-security-monitoring.md) | KMS, WAF, CloudWatch, GuardDuty |
| [week7-serverless-integration.md](week7-serverless-integration.md) | Lambda, SQS, SNS, ECS, Kinesis |
| [week8-cost-exam-prep.md](week8-cost-exam-prep.md) | Cost Optimization + Exam Strategy |

---

*Start with Week 1. Do the lab. Don't move on until the lab works.*
