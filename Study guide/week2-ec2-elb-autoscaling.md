# WEEK 2: EC2 + Elastic Load Balancer + Auto Scaling
### AWS Solutions Architect Associate Study Notes

---

## WHY THIS MATTERS (Read This First)

EC2 is the backbone of AWS. Almost every architecture has EC2 in it — either directly, or powering services behind the scenes. The exam loves to test whether you know:

1. **Which EC2 option saves the most money** (Spot vs Reserved vs On-Demand)
2. **How to make EC2 highly available** (Auto Scaling + Load Balancer)
3. **Which load balancer to pick** (ALB vs NLB — this comes up constantly)

The architecture pattern you must memorize: **Users → Load Balancer → Auto Scaling Group → EC2 Instances → Database**

This pattern appears in 30%+ of exam questions.

---

## SECTION 1: EC2 Fundamentals

---

### 1.1 What Is EC2?

EC2 (Elastic Compute Cloud) = a virtual machine you rent from AWS.

You pick:
- **How powerful** (instance type)
- **What OS** (AMI — Amazon Machine Image)
- **Where** (region, AZ, VPC, subnet)
- **How long** (pricing model)

```
EC2 INSTANCE
───────────────────────────
  CPU:     vCPUs
  RAM:     GiB
  Storage: EBS (network) or Instance Store (local, ephemeral)
  Network: ENI (Elastic Network Interface) — your NIC
  OS:      From AMI (Amazon Linux, Ubuntu, Windows, etc.)
```

---

### 1.2 EC2 Instance Types — Families

| Family | Purpose | Examples | When to Pick |
|--------|---------|---------|-------------|
| **General Purpose** | Balanced CPU/RAM/network | t3, t4g, m6i | Web apps, small DBs, dev |
| **Compute Optimized** | More CPU power | c6i, c7g | Batch processing, gaming, ML inference |
| **Memory Optimized** | More RAM | r6i, x2iezn | Large in-memory DBs, SAP HANA, big caches |
| **Storage Optimized** | High disk I/O | i3, i4i, d3 | Data warehouses, Hadoop, NoSQL |
| **Accelerated Computing** | GPU | p4, g5, inf2 | ML training, video encoding |
| **HPC** | High-performance networking | hpc7g | Computational fluid dynamics, simulations |

**Reading an instance name:**
```
m6i.xlarge
│├─ Family (m = memory-balanced general purpose)
│└─ Generation (6 = 6th gen)
├─ Attribute (i = Intel processor)
└─ Size (xlarge)

Size scale: nano < micro < small < medium < large < xlarge < 2xlarge < ...
```

> **Exam tip:** "Compute-intensive batch jobs" → C family. "In-memory database" → R family. "Most cost-effective for web app" → T family (burstable).

---

### 1.3 T-Series Burstable Instances (CPU Credits)

T2/T3/T4g instances work differently than others. They earn **CPU credits** when idle, and spend them when busy.

```
Normal (low load):
  CPU usage: 5%  → earns credits passively

Burst (high load):
  CPU usage: 100% → spends credits fast

When credits run out:
  t2.micro (default): CPU throttled to baseline (6-20% depending on size)
  t3 with "Unlimited mode": continues at full speed (charged for extra)
```

**When this matters:**
- Good for spiky workloads (web servers that get traffic bursts)
- Bad for sustained high CPU (use c or m family instead)

---

### 1.4 AMI — Amazon Machine Image

An AMI is a **template** for your instance — OS, software, configuration pre-packaged.

```
AMI contains:
  ├── Root volume snapshot (OS, software)
  ├── Launch permissions (who can use this AMI)
  └── Block device mapping (what volumes to attach)

AMI types:
  AWS-provided  → Amazon Linux 2023, Ubuntu, Windows Server
  Marketplace   → Pre-installed software (WordPress, SAP, etc.)
  Custom        → You build it, snapshot it, reuse it
  Community     → Shared by other AWS users (careful — verify source)
```

**Golden AMI pattern (exam favorite):**
```
1. Launch base EC2
2. Install all software and configure
3. Create AMI from instance (snapshot of root volume)
4. Use this "golden AMI" in Auto Scaling Launch Template
5. New EC2s boot 100% pre-configured — no startup scripts needed
```

---

### 1.5 EC2 Pricing Models — The Most Tested Topic

**You MUST know when to use each. This appears in multiple exam questions.**

#### On-Demand
- Pay by the hour/second
- No commitment, most flexible
- Most expensive per hour
- **Use for:** Short, unpredictable workloads, development, first-time testing

#### Reserved Instances (RI)
- Commit to 1 or 3 years
- Up to **72% cheaper** than On-Demand
- Types:
  - **Standard RI:** Can't change instance type. Biggest discount.
  - **Convertible RI:** Can change family/size. ~54% discount.
- Scopes:
  - **Regional RI:** Applies to any AZ in region. Flexible.
  - **Zonal RI:** Specific AZ. Reserves capacity (better if you need guaranteed capacity).
- **Use for:** Steady-state, always-on workloads (production web servers, databases)

#### Savings Plans
- Commit to a **spend amount** per hour (not specific instance)
- Two types:
  - **Compute Savings Plans:** Any EC2, Fargate, Lambda. Most flexible. ~66% discount.
  - **EC2 Instance Savings Plans:** Specific family in specific region. ~72% discount.
- Modern alternative to RIs — AWS recommends Savings Plans for new workloads

#### Spot Instances
- Bid on unused AWS capacity
- Up to **90% cheaper** than On-Demand
- **Catch:** AWS can terminate with 2-minute warning when capacity needed
- **Use for:** Fault-tolerant batch jobs, data analysis, stateless workers, CI/CD runners
- **Never use for:** Databases, anything stateful, anything that can't handle interruption

#### Dedicated Hosts
- Physical server dedicated to you
- Most expensive
- **Use for:** Compliance requirements, software licensing tied to physical core/socket (Oracle, Windows Server per-socket licensing)

#### Dedicated Instances
- Instance runs on hardware dedicated to your account
- May share hardware with other instances from YOUR account
- Cheaper than Dedicated Hosts

```
COST COMPARISON (approximate):
On-Demand:          $0.10/hr  (baseline)
Reserved (3yr):     $0.028/hr (72% off)
Savings Plan:       $0.034/hr (66% off)
Spot:               $0.01/hr  (90% off)
Dedicated Host:     $0.45/hr+ (hardware cost)
```

> **Exam pattern:** "Most cost-effective for a database that runs 24/7" → **Reserved Instance** or **Savings Plan**. "Batch processing that can be interrupted" → **Spot Instances**.

---

### 1.6 Instance Metadata and User Data

**User Data:** Script that runs when EC2 **first boots**.
```bash
#!/bin/bash
# This runs once at first boot
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Hello from EC2</h1>" > /var/www/html/index.html
```
- Used to bootstrap configuration
- Available at: http://169.254.169.254/latest/user-data

**Instance Metadata:** Info about the running instance, accessible from inside EC2.
```bash
# Get instance ID
curl http://169.254.169.254/latest/meta-data/instance-id

# Get private IP
curl http://169.254.169.254/latest/meta-data/local-ipv4

# Get public IP
curl http://169.254.169.254/latest/meta-data/public-ipv4

# Get IAM Role credentials (temp creds from attached role)
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/MyRoleName
```

> **IMDSv2 (Instance Metadata Service v2):** Newer, more secure version that requires a session token. AWS now recommends requiring IMDSv2 to prevent SSRF attacks.

---

### 1.7 Placement Groups

Control where EC2 instances physically sit relative to each other.

| Type | What It Does | Use Case | Risk |
|------|-------------|---------|------|
| **Cluster** | Pack instances close together in ONE AZ | Low latency, high throughput between instances (HPC, big data) | If AZ fails, all go down |
| **Spread** | Instances on different physical hardware | Max 7 per AZ. Critical apps that must not share hardware failure | Only 7 instances per AZ |
| **Partition** | Groups of instances in partitions, each partition on isolated hardware | Hadoop, Cassandra, Kafka (distributed systems need failure isolation) | Balanced risk |

```
CLUSTER:                  SPREAD:                   PARTITION:
[EC2][EC2][EC2]           [EC2]  [EC2]  [EC2]       Partition1: [EC2][EC2]
[EC2][EC2][EC2]           rack1  rack2  rack3        Partition2: [EC2][EC2]
All in same rack          each on separate hardware  each partition = isolated rack
10Gbps between them       max HA                     good for distributed DBs
```

---

## SECTION 2: Elastic Load Balancer (ELB)

---

### 2.1 Why Load Balancers?

Imagine 1000 users hitting your app. If you have one EC2, it might crash. A load balancer:
- Spreads traffic across multiple EC2 instances
- Sends traffic only to **healthy** instances
- Enables **zero-downtime deployments** (slowly replace instances)
- Gives you a **single DNS name** for multiple instances

```
                        ┌─ EC2-1 (us-east-1a)
Users ──→ [ALB DNS] ────┼─ EC2-2 (us-east-1a)
                        ├─ EC2-3 (us-east-1b)
                        └─ EC2-4 (us-east-1b)
```

---

### 2.2 Four Types of Load Balancers

**This is heavily tested. Know which to use and why.**

| Feature | ALB | NLB | CLB | GWLB |
|---------|-----|-----|-----|------|
| Full Name | Application LB | Network LB | Classic LB | Gateway LB |
| OSI Layer | Layer 7 (HTTP) | Layer 4 (TCP/UDP) | Layer 4/7 | Layer 3 (IP) |
| Protocols | HTTP, HTTPS, gRPC, WebSocket | TCP, UDP, TLS | HTTP, TCP | IP packets |
| Use Case | Web apps, microservices, REST APIs | Ultra-low latency, static IP, non-HTTP | Legacy (avoid) | 3rd-party network appliances |
| Routing | URL path, hostname, headers, query | IP/port | Round robin | Transparent bump-in-the-wire |
| Static IP | No (use with NLB for static IP) | YES (1 per AZ) | No | No |
| Latency | Milliseconds | Microseconds | - | - |

#### ALB (Application Load Balancer) — Most Common

Operates at HTTP level. Can route based on:

```
URL Path routing:
  /api/*          → Target Group: API Servers (EC2 fleet)
  /images/*       → Target Group: Image Servers (EC2 fleet)
  /*              → Target Group: Web Servers (default)

Host-based routing:
  api.example.com → Target Group: API servers
  www.example.com → Target Group: Web servers

Header-based routing:
  X-User-Country: IN → Target Group: India servers

Query string routing:
  ?platform=mobile → Target Group: Mobile backend
```

ALB also supports:
- **WebSocket** connections
- **HTTP/2**
- **gRPC** (for microservices)
- **Authentication** via Cognito or OIDC (offload auth from app)
- **Sticky sessions** (route same user to same instance)

#### NLB (Network Load Balancer) — Pick When Performance Matters

- Handles **millions of requests per second**
- Ultra-low latency (~100 microseconds vs ALB's milliseconds)
- Gets **static Elastic IP per AZ** — useful when clients whitelist specific IPs
- Supports **TCP, UDP, TLS** (not HTTP-aware)
- Health checks at TCP level

**When to use NLB:**
```
✓ Game servers (UDP, low latency)
✓ IoT devices communicating over TCP
✓ Financial trading systems
✓ When you need to whitelist specific IP addresses
✓ PrivateLink (exposing services to other VPCs)
```

#### GWLB (Gateway Load Balancer) — Security Appliances

Transparent inspection of traffic through 3rd-party appliances (firewalls, IDS/IPS).

```
Internet → GWLB → [Palo Alto Firewall Fleet] → GWLB → Your App
                   (all traffic inspected)
```

> **Exam tip:** "Deploy 3rd-party firewall for all traffic inspection" = GWLB.

---

### 2.3 Target Groups

Load balancers don't route to instances directly — they route to **Target Groups**.

A Target Group is a collection of:
- EC2 instances
- IP addresses (private IPs, on-prem servers)
- Lambda functions (ALB only)
- Another ALB (ALB only)

```
ALB
 ├── Listener: HTTPS :443
 │    └── Rule 1: Path /api/* → Target Group: API-TG
 │         └── Target Group: API-TG
 │              ├── EC2: i-001 (healthy ✓)
 │              ├── EC2: i-002 (healthy ✓)
 │              └── EC2: i-003 (unhealthy ✗ — no traffic sent here)
 └── Rule 2: Path /* → Target Group: Web-TG
```

**Health checks:** ELB constantly pings each target. If it fails N consecutive checks, marked unhealthy. No traffic sent to unhealthy targets.

---

### 2.4 Cross-Zone Load Balancing

```
WITHOUT cross-zone:
  AZ-a: [EC2-1] [EC2-2]    ← each gets 50% from AZ-a LB node
  AZ-b: [EC2-3]            ← gets 50% from AZ-b LB node
  Result: EC2-1=25%, EC2-2=25%, EC2-3=50% (uneven!)

WITH cross-zone:
  Total requests spread evenly across ALL targets: each gets 33%
```

- ALB: cross-zone always ON (free)
- NLB/GWLB: cross-zone OFF by default (charged if enabled)

---

## SECTION 3: Auto Scaling Groups (ASG)

---

### 3.1 What Is Auto Scaling?

Auto Scaling automatically adds or removes EC2 instances based on demand.

```
9 AM  → traffic increases → ASG launches 2 more EC2s
2 PM  → traffic is normal  → ASG removes 1 EC2
2 AM  → traffic is minimal → ASG scales down to minimum
```

This is how you achieve:
- **High availability** — if one instance dies, ASG launches a replacement
- **Cost efficiency** — pay for what you use
- **Performance** — never run out of capacity

---

### 3.2 ASG Core Components

```
AUTO SCALING GROUP
├── Launch Template (or Launch Configuration — legacy)
│   ├── AMI ID
│   ├── Instance type
│   ├── Security groups
│   ├── Key pair
│   ├── User data
│   └── IAM role
├── Desired capacity: 2  (target number of instances right now)
├── Minimum capacity: 1  (never go below this)
├── Maximum capacity: 10 (never exceed this)
├── VPC + Subnets (spread across AZs)
└── Scaling Policies
```

---

### 3.3 Scaling Policies — Types

| Policy Type | How It Works | Use Case |
|-------------|-------------|---------|
| **Target Tracking** | "Keep CPU at 50%" — ASG figures out when to scale | Simple, recommended, most common |
| **Step Scaling** | "If CPU > 70%, add 2. If CPU > 90%, add 4." | More control over scale-out amounts |
| **Scheduled Scaling** | "At 8 AM weekdays, set desired=10" | Predictable traffic patterns (business hours) |
| **Predictive Scaling** | ML-based forecasting of future traffic | Preemptive scaling before traffic hits |

**Target Tracking example:**
```
Metric: Average CPU Utilization
Target value: 50%

If CPU > 50%: ASG adds instances (scale out)
If CPU < 50%: ASG removes instances (scale in)
ASG constantly tries to keep CPU ~= 50%
```

---

### 3.4 Scaling Cooldown Period

After a scaling event, ASG waits before doing another scale action. This prevents rapid thrashing.

```
Scale out triggered (add 2 instances)
    │
    ▼ Cooldown period (default 300 seconds)
    │ No scaling allowed during this time
    │ Let new instances warm up and metrics settle
    ▼
Next scale decision allowed
```

> **Tip:** If ASG is scaling in and out rapidly, it might be too sensitive. Increase cooldown or adjust target value.

---

### 3.5 ASG Lifecycle Hooks

Pause instance launch/termination to run custom actions.

```
LAUNCH HOOK:
  Instance launching
      ↓
  [Pause — your script runs]
      ↓  (e.g., install monitoring agent, warm cache, pull config from SSM)
  Instance enters service (receives traffic)

TERMINATION HOOK:
  Instance terminating
      ↓
  [Pause — your script runs]
      ↓  (e.g., drain connections, upload logs, deregister from service mesh)
  Instance terminated
```

---

### 3.6 Integration: ALB + ASG Together

```
ALB + ASG Architecture:
─────────────────────────
  Users → ALB (distributes traffic)
            │
            ▼
       [Target Group]
       linked to ASG
            │
       ┌────┴────┐
    EC2-1     EC2-2   ← managed by ASG
    (AZ-a)    (AZ-b)

When ASG launches new EC2:
  1. ASG launches EC2 from Launch Template
  2. ASG registers EC2 with ALB Target Group
  3. ALB starts health checking new EC2
  4. Once healthy, ALB routes traffic to it

When ASG terminates EC2:
  1. ALB deregisters EC2 (Connection draining waits for open connections to complete)
  2. ASG terminates EC2
```

---

### 3.7 Termination Policies

When ASG scales in, which instance gets killed? Default logic:

```
1. Find AZ with most instances
2. If tie: kill instance with oldest Launch Template/Configuration
3. If still tie: kill instance closest to billing hour (maximize savings)
```

You can customize: `OldestInstance`, `NewestInstance`, `OldestLaunchTemplate`, `ClosestToNextInstanceHour`, `AllocationStrategy`

---

## SECTION 4: EBS — Elastic Block Store (Quick Reference)

*(Deep dive in Week 3 — but critical for EC2 labs)*

EBS = the "hard drive" for your EC2.

| Volume Type | Name | IOPS | Use Case |
|-------------|------|------|---------|
| **General Purpose SSD** | gp3 | 3,000-16,000 | Default. Most workloads. |
| **General Purpose SSD** | gp2 | Up to 16,000 | Older default. Cost more than gp3. |
| **Provisioned IOPS SSD** | io2/io1 | Up to 256,000 | Databases needing high, consistent IOPS |
| **Throughput Optimized HDD** | st1 | N/A (MB/s) | Big data, Kafka, sequential reads |
| **Cold HDD** | sc1 | N/A (MB/s) | Archives, lowest cost |

**Key facts:**
- EBS volumes are in **one AZ** — can't span AZs
- EBS can be attached to **one EC2 at a time** (except io1/io2 Multi-Attach)
- Snapshots are stored in S3, and are **regional** (can copy cross-region)

---

## SECTION 5: WEEK 2 LAB

---

### Lab: Build Auto-Scaling Web App with ALB

**Architecture:**
```
Internet → [ALB] → [ASG: 2-6 EC2 instances across AZ-a and AZ-b]
```

**Step 1: Create Launch Template**
```
EC2 → Launch Templates → Create
  Name: web-server-lt
  AMI: Amazon Linux 2023
  Instance type: t2.micro
  Key pair: your key
  Security Group: allow HTTP (80) from ALB SG, SSH (22) from your IP
  User Data:
    #!/bin/bash
    yum update -y
    yum install -y httpd stress
    systemctl start httpd
    systemctl enable httpd
    TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
      -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
    INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
      http://169.254.169.254/latest/meta-data/instance-id)
    echo "<h1>Hello from EC2: $INSTANCE_ID</h1>" > /var/www/html/index.html
```

**Step 2: Create Target Group**
```
EC2 → Target Groups → Create
  Type: Instances
  Protocol: HTTP, Port: 80
  VPC: your VPC
  Health check path: /
  Name: web-tg
```

**Step 3: Create Application Load Balancer**
```
EC2 → Load Balancers → Create → Application Load Balancer
  Name: web-alb
  Scheme: internet-facing
  VPC: your VPC
  Subnets: Public-AZ-a, Public-AZ-b
  Security Group: allow HTTP (80) from 0.0.0.0/0
  Listener: HTTP:80 → Forward to: web-tg
```

**Step 4: Create Auto Scaling Group**
```
EC2 → Auto Scaling Groups → Create
  Name: web-asg
  Launch template: web-server-lt
  VPC: your VPC
  Subnets: Private-AZ-a, Private-AZ-b (or public for this lab)
  Load balancing: Attach to existing LB → web-tg
  Desired: 2, Min: 1, Max: 4
  Scaling policy: Target tracking → CPU 50%
```

**Step 5: Test it**
```
# Get ALB DNS name from console
# Open in browser — you should see the EC2 instance ID
# Refresh — you may hit a different instance (load balancing)

# Test Auto Scaling by simulating CPU load:
# SSH into one EC2 instance (via bastion or directly if public)
stress --cpu 4 --timeout 300   # stress CPU for 5 minutes

# Watch ASG in console → Activity → should see new instances launching
```

**Step 6: Cleanup**
```
1. Delete Auto Scaling Group
2. Terminate any remaining EC2 instances
3. Delete Load Balancer
4. Delete Target Group
5. Delete Launch Template
```

---

## SECTION 6: EXAM QUICK REFERENCE

### EC2 Pricing Cheat Sheet

| Scenario | Answer |
|----------|--------|
| 24/7 production database, 3-year commitment | Standard Reserved Instance |
| Web app, unpredictable traffic, start next week | On-Demand |
| Batch jobs, can handle interruptions, need cheapest | Spot Instances |
| Compliance: software licensed per physical socket | Dedicated Hosts |
| Flexible commitment, any EC2 family | Compute Savings Plan |

### Load Balancer Cheat Sheet

| Scenario | Answer |
|----------|--------|
| Route /api/* to API servers, /* to web servers | ALB (path-based routing) |
| TCP traffic, need static IP per AZ | NLB |
| Need static IP for Load Balancer | NLB |
| Ultra-low latency gaming server | NLB |
| Route traffic through firewall appliance | GWLB |
| WebSocket connections | ALB |
| gRPC microservices | ALB |
| User authentication at load balancer | ALB + Cognito |
| Old app using Classic LB | Migrate to ALB |

### Auto Scaling Cheat Sheet

| Scenario | Answer |
|----------|--------|
| Maintain CPU at 60% | Target Tracking Policy |
| Scale at 8 AM every Monday | Scheduled Scaling |
| Add 2 at 70% CPU, add 5 at 90% | Step Scaling |
| Pre-scale before Black Friday | Predictive Scaling |
| Run cleanup script before termination | Lifecycle Hook (termination) |
| New instances keep dying before they're ready | Lifecycle Hook (launch) or increase health check grace period |
| ASG scaling in/out rapidly | Increase cooldown period |

---

*Next: [week3-s3-storage.md](week3-s3-storage.md)*

