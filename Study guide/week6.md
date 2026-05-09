# WEEK 6: Security + Monitoring
### AWS Solutions Architect Associate Study Notes

---

## WHY THIS MATTERS (Read This First)

Security is the **#1 domain** on the SAA exam at 30% of questions. If you nail security, you've won a third of the exam.

The exam pattern is almost always the same:
1. Scenario describes an architecture with a security problem
2. You pick the correct AWS security service to solve it

The trick? AWS has **many overlapping security services** and you must know exactly which one handles which threat.

Mental model:
- **KMS** = Encryption keys. Any "encrypt at rest" question.
- **WAF** = Block bad web traffic (SQL injection, XSS, bad bots)
- **Shield** = DDoS protection
- **GuardDuty** = Threat detection (ML-based anomaly detection)
- **Inspector** = Vulnerability scanning (CVEs in your EC2/Lambda)
- **Macie** = PII/sensitive data detection in S3
- **CloudTrail** = Audit log. Who did what and when.
- **CloudWatch** = Metrics, alarms, logs. What is your system doing RIGHT NOW.

---

## SECTION 1: KMS — Key Management Service

---

### 1.1 What Is KMS?

KMS is AWS's managed key management service. It stores, rotates, and controls access to cryptographic keys.

Every time you encrypt something in AWS (S3, EBS, RDS, Lambda), KMS is involved.

```
ENCRYPTION FLOW WITH KMS:
─────────────────────────────────────────────────────
Application → "Please encrypt this data with key arn:aws:kms:us-east-1:123:key/abc"
                    ↓
              KMS generates data key
                    ↓
              Returns: encrypted data key + plaintext data key
                    ↓
Your code uses plaintext key to encrypt data → stores encrypted data + encrypted key
Discard plaintext key (never stored)

DECRYPTION:
Send encrypted key to KMS → KMS decrypts it → returns plaintext key
Use plaintext key to decrypt data
```

This "data key encryption" pattern is called **Envelope Encryption**.

---

### 1.2 KMS Key Types

| Key Type | Who Creates It | Who Controls It | Use Case |
|----------|--------------|----------------|---------|
| **AWS Managed Keys** | AWS | AWS (auto-rotates yearly) | Default for most services (free) |
| **Customer Managed Keys (CMK)** | You | You | Fine-grained control, audit, custom rotation |
| **Customer Provided Keys** | You | You (never sent to AWS) | SSE-C on S3 only |

**CMK (Customer Managed Key) gives you:**
- Define key policy (who can use, who can administer)
- Enable/disable the key
- Schedule key deletion (7-30 day waiting period)
- Manual rotation (or enable automatic yearly rotation)
- CloudTrail log of every key usage (who used the key, when, from where)

---

### 1.3 KMS Key Policy

Every KMS key has a **key policy** that controls access. Without a key policy entry, even the account root cannot use the key.

```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::123456789:root" },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::123456789:role/AppRole" },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "*"
    }
  ]
}
```

---

### 1.4 KMS Multi-Region Keys

Replicate a CMK to another region with the same key ID and key material.

```
Primary key (us-east-1)
  key ID: mrk-12345
      │ replicated
      ▼
Replica key (eu-west-1)
  key ID: mrk-12345 (same ID)

Use case: Encrypt in us-east-1 → move encrypted data to eu-west-1 → decrypt locally
(without cross-region KMS API calls, which would be slow)
```

---

### 1.5 Secrets Manager vs Parameter Store

Both store secrets (passwords, API keys) — but they're different.

| Feature | Secrets Manager | Parameter Store |
|---------|----------------|----------------|
| Auto-rotation | YES (built-in for RDS, Redshift, DocumentDB, custom Lambda) | No (you write custom rotation) |
| Cost | ~$0.40/secret/month + $0.05/10k API calls | Free (standard), ~$0.05/advanced |
| Versioning | Yes | Yes |
| Encryption | KMS | KMS |
| Use case | DB passwords, OAuth tokens (need auto-rotation) | Config values, feature flags, non-sensitive config |
| Cross-account | Yes | Yes (via sharing) |

> **Exam tip:** "Rotate database credentials automatically" → Secrets Manager. "Store app config cheaply" → Parameter Store.

---

## SECTION 2: WAF, Shield, Firewall Manager

---

### 2.1 AWS WAF — Web Application Firewall

WAF operates at **Layer 7 (HTTP)** and blocks malicious web requests before they reach your app.

```
Internet → WAF (inspect request) → CloudFront / ALB / API Gateway / AppSync
                │
                ├── ALLOW → request passes to your app
                └── BLOCK → return 403 Forbidden

WAF can inspect:
  ✓ IP addresses
  ✓ HTTP headers
  ✓ HTTP body
  ✓ URL strings
  ✓ Country of origin
```

**WAF Rules — what it can block:**

| Rule Type | Blocks |
|-----------|--------|
| **IP Set** | Specific IPs or CIDR ranges |
| **Geo Match** | Specific countries |
| **Rate-based rules** | Too many requests from one IP (DDoS, brute force) |
| **SQL Injection** | SQLi patterns in request |
| **XSS (Cross-Site Scripting)** | XSS patterns in request |
| **Managed Rule Groups** | Pre-built rules from AWS or Marketplace (OWASP Top 10) |
| **String/Regex Match** | Custom patterns in URI, headers, body |

**WAF integrates with:**
- CloudFront (at edge — globally)
- ALB (regional)
- API Gateway (regional)
- AppSync (regional)
- Cognito User Pools

> **Key difference from NACL/Security Groups:** WAF understands HTTP content (body, headers). NACLs only understand IP/ports.

---

### 2.2 AWS Shield — DDoS Protection

| Tier | Cost | What It Does |
|------|------|-------------|
| **Shield Standard** | FREE | Automatic protection against most common DDoS (Layer 3/4) for all AWS customers |
| **Shield Advanced** | $3,000/month | Enhanced DDoS protection, 24/7 DDoS response team (DRT), financial protection, real-time visibility, WAF included |

```
SHIELD STANDARD (always on, free):
  Protects against: SYN floods, UDP floods, reflection attacks
  Applies to: EC2, ELB, CloudFront, Route 53, Global Accelerator

SHIELD ADVANCED:
  Protects against: More sophisticated volumetric attacks, Layer 7 attacks
  Extra: DDoS response team assists, cost protection if scaling from attack
  Required for: High-traffic, mission-critical apps
```

> **Exam tip:** "Protect against DDoS" → Shield. "Layer 7 DDoS (HTTP floods)" → Shield Advanced + WAF.

---

### 2.3 AWS Firewall Manager

Centrally manage WAF rules, Shield Advanced, Security Groups, and Network Firewall across **multiple AWS accounts** in an Organization.

```
Without Firewall Manager:
  Account 1: configure WAF manually
  Account 2: configure WAF manually
  Account N: configure WAF manually
  (inconsistent, error-prone, labor-intensive)

With Firewall Manager:
  Central account → define policies → automatically applies to all accounts in org
```

> **Use when:** "Apply security rules consistently across all accounts in an organization."

---

## SECTION 3: Threat Detection and Compliance

---

### 3.1 Amazon GuardDuty

GuardDuty is a **threat detection** service that uses ML to analyze AWS logs and find malicious behavior.

```
GuardDuty analyzes:
  ✓ CloudTrail event logs (API calls)
  ✓ VPC Flow Logs (network traffic)
  ✓ DNS Logs (domain queries)
  ✓ S3 Data Events
  ✓ EKS Audit Logs
  ✓ RDS Login Events
  ✓ Lambda Network Activity

Detects:
  → Compromised EC2 instance communicating with known C&C server
  → AWS credentials used from unusual location (possible theft)
  → Unusual API calls (creating IAM users at 3am)
  → Crypto-mining activity
  → Port scanning from within your VPC
  → Unusual S3 access patterns
```

**Key facts:**
- Takes **one click to enable** — no agents, no infrastructure changes
- 30-day free trial
- Findings appear in GuardDuty console + EventBridge → trigger Lambda for automated remediation

---

### 3.2 Amazon Inspector

Inspector is a **vulnerability scanner** that checks EC2 instances, Lambda functions, and container images for known CVEs.

```
Inspector checks:
  EC2 instances → scans OS packages for known CVEs, network reachability
  Lambda functions → scans package dependencies for vulnerabilities
  ECR container images → scans on push, continuously

Output: List of vulnerabilities with severity score (Critical/High/Medium/Low)
```

**Difference from GuardDuty:**
```
GuardDuty = "Is someone attacking you RIGHT NOW?" (behavioral anomaly detection)
Inspector  = "Do you have known vulnerabilities that COULD be exploited?" (static vulnerability scanning)
```

---

### 3.3 Amazon Macie

Macie uses ML to **discover and protect sensitive data** in S3.

```
Macie scans S3 buckets and finds:
  → PII (names, addresses, SSNs, credit card numbers)
  → Credentials (API keys, passwords in files)
  → Encryption issues (unencrypted sensitive data)
  → Access control issues (publicly accessible buckets with sensitive data)

Alerts go to: Macie console + EventBridge → Lambda for remediation
```

> **Exam tip:** "Detect PII in S3" → Macie. "Detect threats/anomalies in VPC traffic" → GuardDuty.

---

### 3.4 AWS Config

Config is a **resource configuration auditing and compliance** service. It tracks the configuration history of every AWS resource.

```
Config answers:
  "Was this S3 bucket ever public?" (configuration timeline)
  "Which EC2 instances don't have encryption enabled?" (compliance check)
  "What changed in my RDS config at 2 PM yesterday?" (change tracking)

Config rules (compliance checks):
  AWS managed: "s3-bucket-public-read-prohibited" → checks if any S3 bucket allows public read
  Custom:       Your own Lambda-based compliance check

Non-compliant resource → notification (SNS) or auto-remediation (SSM Automation)
```

**Config Aggregator:** Collect configuration data across multiple accounts/regions in one view.

---

## SECTION 4: AWS Organizations + SCPs

---

### 4.1 AWS Organizations

Manage multiple AWS accounts under one organization.

```
Root
├── Management Account (master)
│   └── Organization-wide policies
├── OU: Production
│   ├── Account: Prod-App
│   └── Account: Prod-DB
├── OU: Development
│   ├── Account: Dev-App
│   └── Account: Dev-DB
└── OU: Security
    └── Account: Security-Logging
```

**Benefits:**
- Consolidated billing (single bill, volume discounts)
- Apply Service Control Policies (SCPs) to OUs
- Enable AWS services across all accounts (CloudTrail, GuardDuty, Config)

---

### 4.2 SCPs — Service Control Policies

SCPs are **permission guardrails** applied to accounts or OUs in an Organization.

```
SCP vs IAM Policy:
  IAM Policy: "What can this user/role DO?"
  SCP:        "What is the MAXIMUM that anyone in this account can do?"

SCP CEILING:
  Even if a user has AdministratorAccess IAM policy,
  if SCP denies S3 access → user CANNOT access S3.
  SCP wins.
```

**Example SCP — prevent disabling CloudTrail:**
```json
{
  "Effect": "Deny",
  "Action": [
    "cloudtrail:StopLogging",
    "cloudtrail:DeleteTrail"
  ],
  "Resource": "*"
}
```

Attached to the root or an OU → no account in that OU can ever disable CloudTrail, no matter what IAM says.

---

## SECTION 5: CloudTrail

---

### 5.1 What Is CloudTrail?

CloudTrail records **every API call** made in your AWS account — who made it, from where, when, and what parameters.

```
CloudTrail captures:
  ✓ Console sign-ins
  ✓ CLI commands (aws s3 cp → recorded)
  ✓ SDK API calls
  ✓ Service-to-service calls (Lambda calling S3, etc.)

Each event includes:
  WHO: IAM user/role ARN
  WHAT: API action (s3:DeleteObject)
  WHEN: Timestamp
  WHERE from: Source IP address
  TARGET: Resource ARN
  RESULT: Success or error code
```

---

### 5.2 CloudTrail Event Types

| Type | What It Logs | Default? |
|------|-------------|---------|
| **Management events** | Create/delete/modify resources (CreateBucket, TerminateInstances) | ON by default |
| **Data events** | Object-level operations (S3 GetObject, PutObject, Lambda Invoke) | OFF by default (high volume, extra cost) |
| **Insights events** | Unusual API activity detected by ML | OFF by default (extra cost) |

---

### 5.3 CloudTrail Integrity and Compliance

```
Log file integrity validation:
  CloudTrail creates SHA-256 hash of each log file
  You can verify files haven't been tampered with:
    aws cloudtrail validate-logs --trail-arn ... --start-time ...

Store logs in S3 with:
  → SSE-KMS encryption
  → S3 Object Lock (WORM — immutable for compliance period)
  → MFA Delete enabled on S3 bucket

Send to CloudWatch Logs:
  → Create metric filter: "How many root login events per day?"
  → Create alarm: Alert if root logs in
```

---

## SECTION 6: CloudWatch

---

### 6.1 CloudWatch Overview

CloudWatch is AWS's observability service — metrics, logs, alarms, and dashboards.

```
CloudWatch components:
  Metrics    → Numbers over time (CPU utilization, ALB request count, etc.)
  Logs       → Text log data from EC2, Lambda, ECS, RDS, etc.
  Alarms     → Trigger actions when metric crosses threshold
  Dashboards → Visualize metrics in charts/graphs
  Events     → React to state changes in AWS resources (now EventBridge)
  Insights   → Query and analyze CloudWatch Logs with SQL-like syntax
```

---

### 6.2 CloudWatch Metrics

AWS services automatically send metrics to CloudWatch. EC2 default metrics:
- CPUUtilization, NetworkIn, NetworkOut, DiskReadOps, DiskWriteOps

**EC2 does NOT send by default:**
- RAM usage
- Disk space (not EBS — Instance Store)
- Application-level metrics

To get these, install **CloudWatch Agent** on EC2.

```bash
# Install CloudWatch Agent
sudo yum install amazon-cloudwatch-agent -y

# Configure (specify which metrics and logs to collect)
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard

# Start agent
sudo systemctl start amazon-cloudwatch-agent
```

**Custom metrics:** Send your own app metrics to CloudWatch.
```bash
aws cloudwatch put-metric-data \
  --namespace "MyApp" \
  --metric-name "ActiveUsers" \
  --value 42 \
  --unit Count
```

---

### 6.3 CloudWatch Alarms

Alarms watch a metric and take action when threshold is crossed.

```
Alarm states:
  OK       → metric is within threshold
  ALARM    → metric crossed threshold
  INSUFFICIENT_DATA → not enough data yet

Alarm actions:
  → SNS notification (email, SMS, SQS, Lambda)
  → EC2 action (stop, start, terminate, reboot instance)
  → Auto Scaling action (scale out/in)
  → Systems Manager action

Example: High CPU alarm:
  Metric: AWS/EC2 CPUUtilization
  Statistic: Average
  Period: 5 minutes
  Condition: > 80%
  Datapoints to alarm: 2 of 3 (prevents noise from spikes)
  Action: Send SNS email + trigger Auto Scaling scale-out
```

---

### 6.4 CloudWatch Logs Insights

Query your logs like a database.

```sql
-- Find top 10 IPs with most 404 errors in ALB logs
fields @timestamp, @message
| filter status = 404
| stats count(*) as error_count by clientip
| sort error_count desc
| limit 10
```

---

### 6.5 CloudWatch vs CloudTrail vs Config

**This comparison is on the exam.**

| Service | Question It Answers |
|---------|-------------------|
| **CloudWatch** | What is my infrastructure doing RIGHT NOW? Metrics, performance. |
| **CloudTrail** | WHO made that API call? Audit trail, compliance. |
| **Config** | What was the configuration of this resource at this point in time? Drift detection. |

```
Example scenarios:
  "S3 bucket became public — who did it?" → CloudTrail
  "Was this S3 bucket public yesterday?" → Config
  "ALB is getting 500 errors — how many per minute?" → CloudWatch
  "Alert me when CPU > 80%" → CloudWatch Alarm
  "Detect if any S3 bucket policy allows public access" → Config Rule
  "Detect if IAM credentials are being used from unusual location" → GuardDuty
```

---

## SECTION 7: WEEK 6 LAB

---

### Lab: CloudWatch Alarm → SNS → Email Alert

**Step 1: Create SNS Topic**
```
SNS → Topics → Create topic
  Type: Standard
  Name: high-cpu-alerts
Create subscription:
  Protocol: Email
  Endpoint: your@email.com
Confirm subscription (click link in email)
```

**Step 2: Create CloudWatch Alarm**
```
CloudWatch → Alarms → Create Alarm
  → Select metric → EC2 → Per-Instance Metrics → CPUUtilization
  → Select your EC2 instance
  Conditions:
    Static threshold → Greater than 60%
    Datapoints to alarm: 1 out of 1
  Notification:
    Send to: SNS → high-cpu-alerts
  Alarm name: high-cpu-alert
```

**Step 3: Trigger the alarm**
```bash
# SSH into EC2
# Simulate 100% CPU:
stress --cpu 2 --timeout 120

# Within 1-5 minutes you should get an email alert
```

**Step 4: Enable CloudTrail**
```
CloudTrail → Create trail
  Trail name: my-audit-trail
  Storage: S3 bucket (create new: my-cloudtrail-[yourname])
  CloudWatch Logs: Enable → Create new log group
  Management events: Read + Write
  Create trail
```

**Step 5: Create metric filter for root login**
```
CloudWatch → Log groups → find CloudTrail log group
→ Metric filters → Create metric filter

Filter pattern:
{ $.userIdentity.type = "Root" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != "AwsServiceEvent" }

Metric name: RootAccountUsage
Namespace: CloudTrailMetrics
Value: 1

Create alarm on this metric:
  Threshold: >= 1 occurrence
  Action: SNS → your email
```

**Step 6: Enable GuardDuty**
```
GuardDuty → Get Started → Enable GuardDuty
(It's free for 30 days, minimal cost after)

Sample findings:
GuardDuty → Settings → Generate Sample Findings
→ This creates fake findings to see what alerts look like
```

---

## SECTION 8: EXAM QUICK REFERENCE

### Security Services Cheat Sheet

| Scenario | Answer |
|----------|--------|
| Encrypt EBS, S3, RDS | KMS |
| Manage and rotate DB passwords automatically | Secrets Manager |
| Store app config, feature flags | Parameter Store |
| Block SQL injection at HTTP level | WAF |
| Block traffic from specific country | WAF (Geo Match) or CloudFront Geo Restriction |
| DDoS protection, basic | Shield Standard (free, automatic) |
| DDoS protection + 24/7 response team | Shield Advanced |
| Detect compromised EC2 mining crypto | GuardDuty |
| Detect unusual API calls in account | GuardDuty |
| Scan EC2 for known CVE vulnerabilities | Inspector |
| Find PII data in S3 buckets | Macie |
| Who deleted that S3 bucket? | CloudTrail |
| Was security group inbound rule changed? | Config |
| Alert when CPU > 80% | CloudWatch Alarm |
| Apply WAF rules across all org accounts | Firewall Manager |
| Prevent devs from disabling CloudTrail | SCP in Organizations |
| Restrict what actions account members can take | SCP |

---

*Next: [week7-serverless-integration.md](week7-serverless-integration.md)*
