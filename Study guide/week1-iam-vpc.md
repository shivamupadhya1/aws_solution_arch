# WEEK 1: IAM + VPC — The Foundations
### AWS Solutions Architect Associate Study Notes

---

## WHY THIS MATTERS (Read This First)

Every AWS architecture question on the exam either starts with **"who has access?"** (IAM) or **"what network is it in?"** (VPC).

Get these wrong and you fail — even if you know S3, EC2, and RDS perfectly.

Think of it this way:
- **IAM** = Security guard at the door. Controls who can enter and what they can touch.
- **VPC** = The building itself. Controls what's inside, how rooms are connected, and what goes to the outside world.

Everything else you learn — Lambda, RDS, ECS — lives **inside** these two.

---

## SECTION 1: IAM — Identity and Access Management

---

### 1.1 The Core IAM Entities (Learn These Cold)

```
IAM UNIVERSE
────────────
Users     → Individual people or apps (long-term credentials)
Groups    → Collections of users (apply policies to many at once)
Roles     → Temporary identities (assumed by EC2, Lambda, services, other accounts)
Policies  → JSON documents that define what is allowed or denied
```

**The golden rule of IAM:** Explicit DENY always beats ALLOW.

```json
{
  "Effect": "Deny",
  "Action": "s3:DeleteObject",
  "Resource": "*"
}
```
Even if another policy allows `s3:DeleteObject`, this Deny wins. Period.

---

### 1.2 Users vs Roles — The Most Confused Concept

**User:** A permanent identity. Has a username + password (console) or access key + secret (CLI/API).
- Use for: humans, or apps running **outside** AWS (like your local dev script)

**Role:** A temporary identity. No password. Gets temporary credentials via STS.
- Use for: EC2 instances, Lambda functions, other AWS services, cross-account access

> **Exam trap:** An EC2 app needs to access S3. Do you create an IAM user and put the credentials in the code? **NO.** You attach an **IAM Role** to the EC2 instance. No hardcoded credentials ever.

```
WRONG ❌                        CORRECT ✓
──────────────────────────      ──────────────────────────
EC2 Instance                    EC2 Instance
  └─ code has AWS_ACCESS_KEY      └─ IAM Role: "EC2-S3-ReadRole"
     hardcoded in config               └─ Policy: s3:GetObject on bucket
```

---

### 1.3 IAM Policies — Types You Must Know

| Policy Type | Attached To | Use Case |
|-------------|-------------|----------|
| **Identity-based** | Users, Groups, Roles | "This user/role can do X" |
| **Resource-based** | Resources (S3, KMS) | "This resource allows principal Y" |
| **Permission boundaries** | Users, Roles | Max permissions a user/role can have |
| **SCPs (Service Control Policies)** | AWS Organizations | Account-level ceiling on all IAM |
| **Session policies** | Temporary sessions | Further restrict assumed role |

**Policy evaluation logic:**
```
1. Start with: DENY all
2. Evaluate all applicable policies
3. Any explicit DENY? → Final answer: DENY
4. Any explicit ALLOW? → Final answer: ALLOW
5. Neither? → Final answer: DENY (implicit deny)
```

---

### 1.4 IAM Policy Structure — Read Every JSON Field

```json
{
  "Version": "2012-10-17",          // Always this value
  "Statement": [
    {
      "Sid": "AllowS3ReadAccess",   // Optional label
      "Effect": "Allow",            // Allow or Deny
      "Principal": {                // WHO (only in resource-based policies)
        "AWS": "arn:aws:iam::123456789012:user/alice"
      },
      "Action": [                   // WHAT
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [                 // WHICH resource
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {                // WHEN/IF
        "IpAddress": {
          "aws:SourceIp": "192.168.1.0/24"
        }
      }
    }
  ]
}
```

> **Common exam question:** An S3 bucket needs to allow only your company IP. Use a **bucket policy** with a `Condition` on `aws:SourceIp`.

---

### 1.5 IAM Roles — How AssumeRole Works (STS)

When a service needs temporary credentials:

```
Step 1: EC2 calls STS AssumeRole
         ↓
Step 2: STS validates the trust policy on the role
         ↓
Step 3: STS returns: AccessKeyId + SecretAccessKey + SessionToken (expires 1hr-12hrs)
         ↓
Step 4: EC2 uses these temp credentials to call AWS APIs
         ↓
Step 5: Credentials expire → EC2 requests new ones automatically (via instance metadata)
```

**Trust policy** (who can assume the role) lives on the **Role**, not the caller:
```json
{
  "Principal": {
    "Service": "ec2.amazonaws.com"   // EC2 can assume this role
  },
  "Action": "sts:AssumeRole",
  "Effect": "Allow"
}
```

---

### 1.6 Cross-Account Access

Scenario: Account A's Lambda needs to read S3 in Account B.

```
Account A (Lambda)                  Account B (S3 Bucket)
─────────────────                   ────────────────────
Lambda → AssumeRole ──────────────→ Role in Account B
                    ←── temp creds ──   (trust policy allows Account A)
Lambda uses creds to access S3 ──→ S3 Bucket
```

Key points:
- Role in **Account B** has trust policy allowing Account A's principal
- Lambda in Account A assumes that cross-account role
- No credentials are stored anywhere

---

### 1.7 IAM Best Practices (Exam + Real Life)

| Practice | Why |
|----------|-----|
| Enable MFA on root account | Root has unlimited power, protect it |
| Never use root for daily tasks | Create IAM admin user instead |
| Use roles for EC2/Lambda, not users | No hardcoded credentials |
| Grant least privilege | Only permissions actually needed |
| Use groups to assign permissions | Easier to manage at scale |
| Rotate access keys regularly | Leaked keys = breach |
| Use IAM Access Analyzer | Finds unintended external access |

---

### 1.8 Common IAM Services You'll See on the Exam

| Service | What It Does |
|---------|-------------|
| **AWS SSO / IAM Identity Center** | Centralized SSO for multiple AWS accounts, integrates with Active Directory |
| **STS (Security Token Service)** | Issues temporary credentials |
| **AWS Directory Service** | Managed Microsoft AD in AWS |
| **Cognito** | Identity for **your app's users** (not AWS users). Cognito User Pools = auth, Identity Pools = AWS credentials |
| **IAM Access Analyzer** | Finds resources accessible outside your account |

> **Trap:** Cognito is for YOUR application users (end customers). IAM is for AWS users. Never confuse them.

---

## SECTION 2: VPC — Virtual Private Cloud

---

### 2.1 What Is a VPC? (Easy Language)

A **VPC** is your own private slice of AWS's network. When you launch an EC2, RDS, or Lambda, you choose which VPC it lives in.

Think of AWS as a massive apartment complex:
- **AWS Region** = The entire city block
- **Availability Zone** = One floor of the building
- **VPC** = Your apartment (private, walled off from neighbors)
- **Subnet** = Rooms inside your apartment
- **Security Group** = Locks on each room's door
- **NACL** = Security guard at the apartment entrance (floor level)
- **Internet Gateway** = The main building entrance to the street

```
AWS REGION: us-east-1
├── VPC: 10.0.0.0/16 (your private network)
│   ├── AZ: us-east-1a
│   │   ├── Public Subnet:  10.0.1.0/24  (has route to Internet Gateway)
│   │   └── Private Subnet: 10.0.2.0/24  (no direct internet route)
│   ├── AZ: us-east-1b
│   │   ├── Public Subnet:  10.0.3.0/24
│   │   └── Private Subnet: 10.0.4.0/24
│   └── AZ: us-east-1c
│       ├── Public Subnet:  10.0.5.0/24
│       └── Private Subnet: 10.0.6.0/24
```

---

### 2.2 CIDR Notation Crash Course

CIDR (Classless Inter-Domain Routing) is how you define IP ranges.

```
10.0.0.0/16  → 10.0.0.0 to 10.0.255.255  → 65,536 IPs
10.0.0.0/24  → 10.0.0.0 to 10.0.0.255   → 256 IPs
10.0.0.0/28  → 10.0.0.0 to 10.0.0.15    → 16 IPs

Rule: Smaller number after / = BIGGER range
      /16 > /24 > /28 in terms of IP count

AWS reserves 5 IPs in every subnet:
  10.0.0.0   → Network address
  10.0.0.1   → VPC router
  10.0.0.2   → DNS server
  10.0.0.3   → Reserved for future use
  10.0.0.255 → Broadcast (AWS doesn't support broadcast, just reserves it)

So /24 subnet = 256 - 5 = 251 usable IPs
```

---

### 2.3 Public Subnet vs Private Subnet

This is the most fundamental VPC concept. **Learn the difference. Every architecture uses it.**

| Feature | Public Subnet | Private Subnet |
|---------|--------------|----------------|
| Has route to Internet Gateway | YES | NO |
| Resources get public IP | YES (if enabled) | NO |
| Accessible from internet | YES | NO |
| Can initiate internet traffic | YES | Only via NAT Gateway |
| Typical resources | Load Balancers, Bastion hosts, NAT Gateway | EC2 app servers, RDS, Lambda |

```
TYPICAL 3-TIER ARCHITECTURE
─────────────────────────────
Internet
    │
    ▼
[Internet Gateway]
    │
    ▼
Public Subnet
  [ALB — Application Load Balancer]
    │
    ▼
Private Subnet
  [EC2 App Servers]
    │
    ▼
Private Subnet (DB tier — often separate)
  [RDS Database]
```

---

### 2.4 Internet Gateway (IGW)

- **One per VPC** (attached to the VPC, not a subnet)
- Allows resources in public subnets to communicate with the internet
- Horizontally scaled, redundant, no bandwidth limits

**What makes a subnet "public":**
1. Route table has entry: `0.0.0.0/0 → igw-xxxxxxxx`
2. Resource in that subnet has a public IP (or Elastic IP)

```
Route Table for Public Subnet:
  Destination     Target
  10.0.0.0/16     local       ← traffic within VPC
  0.0.0.0/0       igw-abc123  ← everything else goes to internet
```

---

### 2.5 NAT Gateway vs NAT Instance

Private subnet resources need to download updates, reach the internet, but **must not be reachable FROM the internet**. That's NAT.

#### NAT Gateway (AWS Managed — Preferred)

```
Private EC2 wants to reach internet:
  EC2 (10.0.2.10) → NAT Gateway (in public subnet) → Internet Gateway → Internet
  Response comes back same way (NAT hides private IP)
```

| Feature | NAT Gateway | NAT Instance |
|---------|------------|--------------|
| Management | AWS managed | You manage the EC2 |
| Availability | Highly available within AZ | Single EC2 = single point of failure |
| Bandwidth | Up to 100 Gbps | Limited by instance type |
| Security Groups | Not supported | Supported |
| Cost | ~$0.045/hr + data | EC2 cost only |
| Use case | **Always use this** | Legacy / cost-sensitive |

> **Critical:** NAT Gateway is in a specific AZ. For HA, create one NAT Gateway per AZ, and have each AZ's private subnet route to its own NAT Gateway.

```
WRONG (single NAT):                CORRECT (HA NAT):
─────────────────                  ────────────────────────
AZ-a Public Subnet                 AZ-a Public Subnet
  [NAT Gateway] ←── AZ-a Private     [NAT Gateway] ←── AZ-a Private
                ←── AZ-b Private   AZ-b Public Subnet
                ←── AZ-c Private     [NAT Gateway] ←── AZ-b Private
If NAT dies, all 3 AZs lose        AZ-c Public Subnet
internet access                      [NAT Gateway] ←── AZ-c Private
```

---

### 2.6 Security Groups vs NACLs

**The most tested VPC concept on the exam.**

| Feature | Security Group | NACL (Network ACL) |
|---------|---------------|-------------------|
| Level | **Instance level** (ENI) | **Subnet level** |
| State | **Stateful** | **Stateless** |
| Rules | Allow rules only | Allow AND Deny rules |
| Rule evaluation | All rules evaluated | Rules evaluated **in order** (lowest number wins) |
| Default | Deny all inbound, allow all outbound | Allow all inbound and outbound |
| Applies to | Specific EC2 instances | All resources in subnet |

**Stateful vs Stateless — The Key Difference:**

```
SECURITY GROUP (Stateful):
  Inbound: Allow port 80 from 0.0.0.0/0
  → Response traffic allowed automatically (no rule needed for outbound)
  → "I remember the conversation"

NACL (Stateless):
  Inbound: Allow port 80 from 0.0.0.0/0    ← You must define this
  Outbound: Allow port 1024-65535 (ephemeral ports) ← You ALSO must define this
  → "I forget everything. Both directions need explicit rules"
```

**NACL Rule Example:**

```
NACL for Public Subnet — Inbound Rules:
  Rule #  Type          Port    Source        Allow/Deny
  100     HTTP (80)     80      0.0.0.0/0     ALLOW
  110     HTTPS (443)   443     0.0.0.0/0     ALLOW
  120     Custom TCP    1024-65535  0.0.0.0/0 ALLOW  ← ephemeral ports for responses
  *       ALL           ALL     0.0.0.0/0     DENY   ← default catch-all
```

> **Exam tip:** If you see "need to block a specific IP address", the answer uses **NACL** (Security Groups can't deny, only allow).

---

### 2.7 Route Tables

Every subnet is associated with a route table. The route table decides where traffic goes.

```
Main Route Table (default for all subnets):
  Destination     Target
  10.0.0.0/16     local

Public Route Table (associate with public subnets):
  Destination     Target
  10.0.0.0/16     local
  0.0.0.0/0       igw-xxxxx

Private Route Table (associate with private subnets):
  Destination     Target
  10.0.0.0/16     local
  0.0.0.0/0       nat-xxxxx  ← goes to NAT Gateway instead of IGW
```

**Longest prefix match rule:** More specific route wins.
```
If you have:
  10.0.0.0/16  → local
  10.0.0.0/8   → some-gateway
Traffic to 10.0.5.5 → uses /16 (more specific = longer prefix)
```

---

### 2.8 VPC Peering

Connects two VPCs so they can communicate using private IPs.

```
VPC A (10.0.0.0/16) ←──Peering──→ VPC B (172.16.0.0/16)
```

**Rules:**
- CIDRs cannot overlap
- Not transitive — if A peers with B and B peers with C, A cannot reach C
- Works across accounts and regions (inter-region peering)
- Must update route tables in BOTH VPCs

```
For A to reach B:
  VPC A route table: 172.16.0.0/16 → pcx-xxxxxxxxx (peering connection)
  VPC B route table: 10.0.0.0/16  → pcx-xxxxxxxxx (peering connection)
```

---

### 2.9 Transit Gateway

The solution when you have many VPCs and VPC Peering becomes a management nightmare.

```
WITHOUT Transit Gateway (N×(N-1)/2 peering connections):
VPC1 ─── VPC2
VPC1 ─── VPC3
VPC1 ─── VPC4
VPC2 ─── VPC3
VPC2 ─── VPC4
VPC3 ─── VPC4
(6 peerings for 4 VPCs)

WITH Transit Gateway (hub-and-spoke):
        [Transit Gateway]
       /      |      \    \
   VPC1    VPC2    VPC3   VPC4   On-Prem (via VPN)
(4 attachments, no mesh)
```

Transit Gateway is transitive — VPC1 can reach VPC4 through the hub.

---

### 2.10 VPC Endpoints

Connects your VPC to AWS services **without going through the internet**. Traffic stays on AWS backbone.

| Endpoint Type | Works With | How |
|--------------|-----------|-----|
| **Gateway Endpoint** | S3, DynamoDB ONLY | Added to route table |
| **Interface Endpoint** | 100+ services (SSM, SQS, ECR, etc.) | Creates ENI with private IP in your subnet |

```
WITHOUT Endpoint:
  EC2 (private subnet) → NAT Gateway → Internet → S3
  Cost: NAT data processing + transfer fees

WITH Gateway Endpoint:
  EC2 (private subnet) → S3 Gateway Endpoint → S3
  Cost: FREE (no NAT needed, no data processing fee)
```

> **Exam tip:** Private EC2 needs to access S3 without internet? Add **S3 Gateway Endpoint**. No NAT Gateway needed. Also cheaper.

---

### 2.11 VPN vs Direct Connect

Both connect your on-premises data center to AWS, but very differently.

| Feature | Site-to-Site VPN | Direct Connect |
|---------|-----------------|----------------|
| Connection type | Encrypted tunnel over **public internet** | Dedicated private fiber link |
| Setup time | Minutes | Weeks to months |
| Bandwidth | Limited by internet (~1.25 Gbps max) | 1 Gbps or 10 Gbps dedicated |
| Cost | Low (hourly + data) | High (port hours + data) |
| Latency | Variable (internet) | Consistent and low |
| Reliability | Best-effort | SLA-backed |
| Encryption | Yes (IPSec) | Not by default (add VPN on top) |

```
WHEN TO USE:
VPN  → Quick setup, cost-sensitive, backup for Direct Connect
Direct Connect → Consistent performance, large data transfer, compliance requirements
```

> **Exam tip:** "Private connection" always = Direct Connect. "Encrypted over internet" = VPN.

---

### 2.12 Bastion Host

A single EC2 in the **public subnet** used to SSH into instances in **private subnets**.

```
Your Laptop
    │
    │ SSH (port 22)
    ▼
[Bastion Host] ← in Public Subnet, has public IP
    │
    │ SSH (port 22, private IP)
    ▼
[Private EC2] ← in Private Subnet, no public IP
```

**Security rule:** The Bastion Host's Security Group should only allow SSH from **your IP address** — not 0.0.0.0/0.

---

### 2.13 Elastic IP (EIP)

A static public IP address that you own in AWS.

- Normal EC2 public IP changes when you stop/start the instance
- EIP is fixed — stays the same no matter what
- EIP can be moved between instances (for failover)
- **Cost:** EIP is FREE when associated with a running instance. Charged when NOT in use (to discourage wasting IPs)

---

## SECTION 3: WEEK 1 LAB

---

### Lab: Build a Production-Ready VPC from Scratch

**What you'll build:**
```
Internet
    │
[Internet Gateway]
    │
Public Subnets (AZ-a, AZ-b):
  ├─ Bastion Host (EC2)
  └─ NAT Gateway (one per AZ)
    │
Private Subnets (AZ-a, AZ-b):
  └─ App Server EC2 (no public IP)
```

**Step-by-step:**

**Step 1: Create VPC**
```
VPC Console → Create VPC
Name: my-prod-vpc
CIDR: 10.0.0.0/16
Tenancy: Default
```

**Step 2: Create Subnets**
```
Subnet 1 — Public AZ-a:   10.0.1.0/24  in us-east-1a
Subnet 2 — Public AZ-b:   10.0.2.0/24  in us-east-1b
Subnet 3 — Private AZ-a:  10.0.3.0/24  in us-east-1a
Subnet 4 — Private AZ-b:  10.0.4.0/24  in us-east-1b
```

**Step 3: Create and Attach Internet Gateway**
```
Internet Gateways → Create → Name: my-igw
Actions → Attach to VPC → select my-prod-vpc
```

**Step 4: Create NAT Gateways (one per AZ)**
```
NAT Gateways → Create
  Name: nat-az-a
  Subnet: Public-AZ-a
  Allocate Elastic IP → click Allocate EIP → Create

Repeat for AZ-b:
  Name: nat-az-b
  Subnet: Public-AZ-b
```

**Step 5: Create Route Tables**
```
Public Route Table:
  Routes → Add: 0.0.0.0/0 → Internet Gateway (my-igw)
  Subnet associations → Associate Public-AZ-a and Public-AZ-b

Private Route Table AZ-a:
  Routes → Add: 0.0.0.0/0 → NAT Gateway (nat-az-a)
  Subnet associations → Associate Private-AZ-a

Private Route Table AZ-b:
  Routes → Add: 0.0.0.0/0 → NAT Gateway (nat-az-b)
  Subnet associations → Associate Private-AZ-b
```

**Step 6: Launch Bastion Host**
```
EC2 → Launch Instance
  AMI: Amazon Linux 2023
  Instance type: t2.micro (free tier)
  Network: my-prod-vpc
  Subnet: Public-AZ-a
  Auto-assign public IP: Enable
  Security Group: allow SSH (22) from YOUR IP only
  Key pair: create new, download .pem
```

**Step 7: Launch Private App Server**
```
EC2 → Launch Instance
  AMI: Amazon Linux 2023
  Instance type: t2.micro
  Network: my-prod-vpc
  Subnet: Private-AZ-a
  Auto-assign public IP: DISABLE
  Security Group: allow SSH (22) from Bastion's Security Group ID
  Same key pair as above
```

**Step 8: Test Connectivity**
```bash
# On your local machine, SSH to Bastion
ssh -i your-key.pem ec2-user@<bastion-public-ip>

# From Bastion, SSH to private EC2
ssh -i your-key.pem ec2-user@<private-ec2-private-ip>

# Test internet via NAT (from private EC2)
curl https://checkip.amazonaws.com
# Should show NAT Gateway's EIP — not private EC2 IP
```

**Step 9: Test S3 Gateway Endpoint**
```
VPC Console → Endpoints → Create Endpoint
  Service: com.amazonaws.us-east-1.s3 (Gateway type)
  VPC: my-prod-vpc
  Route tables: select Private route tables

# From private EC2 (no NAT needed now for S3):
aws s3 ls --region us-east-1
```

**Step 10: CLEANUP (Important — avoid charges)**
```
1. Terminate EC2 instances
2. Delete NAT Gateways (wait for deletion)
3. Release Elastic IPs (Elastic IPs → Release)
4. Detach and delete Internet Gateway
5. Delete Subnets
6. Delete Route Tables (not main)
7. Delete VPC
```

> NAT Gateway costs ~$0.045/hr = ~$1.08/day. Always delete after labs.

---

## SECTION 4: EXAM QUICK REFERENCE

### IAM Cheat Sheet

| Scenario | Answer |
|----------|--------|
| EC2 needs to access S3 | IAM Role on EC2 |
| Block specific IP from S3 | Bucket policy with Condition + aws:SourceIp |
| App users need AWS credentials | Cognito Identity Pools |
| Multiple AWS accounts, single sign-on | IAM Identity Center (SSO) |
| Enforce max permissions across org | SCP in AWS Organizations |
| Detect external access to resources | IAM Access Analyzer |
| User needs temp credentials | STS AssumeRole |
| Audit who made API calls | CloudTrail |

### VPC Cheat Sheet

| Scenario | Answer |
|----------|--------|
| Private EC2 needs internet for updates | NAT Gateway in public subnet |
| Block specific IP at subnet level | NACL with Deny rule |
| Private EC2 needs to access S3 without internet | S3 Gateway Endpoint |
| Connect multiple VPCs (hub-and-spoke) | Transit Gateway |
| Connect two VPCs directly | VPC Peering |
| Connect on-prem to AWS privately | Direct Connect |
| Connect on-prem to AWS quickly/cheap | Site-to-Site VPN |
| SSH to private EC2 | Bastion Host in public subnet |
| Static IP for EC2 that survives stop/start | Elastic IP |
| Instance needs to be accessible on fixed port from internet | EIP + Security Group |

---

*Next: [week2-ec2-elb-autoscaling.md](week2-ec2-elb-autoscaling.md)*
