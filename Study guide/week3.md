# WEEK 3: S3 + EBS + EFS + Storage Gateway
### AWS Solutions Architect Associate Study Notes

---

## WHY THIS MATTERS (Read This First)

Storage questions are everywhere on this exam — and the traps are sneaky. AWS gives you 6 different S3 storage classes, 5 EBS volume types, and multiple ways to encrypt. If you don't know the differences cold, you'll pick the wrong (expensive) option.

The #1 mistake candidates make: **confusing when to use S3 vs EBS vs EFS**.

Rule of thumb to start:
- **S3** → store anything (objects, files, backups) at massive scale. No EC2 needed.
- **EBS** → your EC2's hard drive. Fast, attached, one EC2 at a time.
- **EFS** → shared file system. Many EC2s mount the same drive simultaneously.

---

## SECTION 1: S3 — Simple Storage Service

---

### 1.1 S3 Fundamentals

S3 stores **objects** (files) in **buckets** (containers).

```
S3 STRUCTURE
────────────────────────────────────────
Bucket: my-company-data           ← Global namespace, unique across ALL AWS
  ├── images/
  │   ├── logo.png                ← Object (key = "images/logo.png")
  │   └── banner.jpg
  ├── reports/
  │   └── annual-2025.pdf
  └── backup/
      └── db-snapshot.tar.gz
```

**Key facts:**
- Max object size: **5 TB**
- Objects uploaded in single PUT: max **5 GB** (use multipart upload for larger)
- Bucket name must be **globally unique** across all AWS accounts
- Bucket is region-specific even though the name is global
- S3 is **object storage** — not a file system (no actual folders, just key name prefixes)
- **Unlimited storage** — AWS scales it automatically

---

### 1.2 S3 Storage Classes — The Most Tested S3 Topic

**Know each class, its cost profile, retrieval speed, and minimum storage duration.**

| Storage Class | Access Pattern | Availability | Min Storage | Retrieval | Use Case |
|--------------|---------------|-------------|------------|-----------|---------|
| **S3 Standard** | Frequent | 99.99% | None | Immediate | Active data, web apps |
| **S3 Intelligent-Tiering** | Unknown/changing | 99.9% | None | Immediate | Auto-moves between tiers |
| **S3 Standard-IA** | Infrequent | 99.9% | 30 days | Immediate | Backups, older data still needed fast |
| **S3 One Zone-IA** | Infrequent | 99.5% (one AZ) | 30 days | Immediate | Replicated data, secondary backups |
| **S3 Glacier Instant** | Archive, rare | 99.9% | 90 days | Milliseconds | Archives needed occasionally |
| **S3 Glacier Flexible** | Archive | 99.99% | 90 days | Minutes to hours | Compliance archives |
| **S3 Glacier Deep Archive** | Rarely accessed | 99.99% | 180 days | 12-48 hours | Long-term compliance (7yr+ retention) |

```
COST vs SPEED (visualized):
                        Fast retrieval
                             ↑
Standard     ●──────────────┤ Expensive + fast
Intelligent  ●──────────────┤ Auto-tiering
Standard-IA  ●──────────────┤ Cheaper storage, charged per retrieval
One Zone-IA  ●──────────────┤ Even cheaper, one AZ risk
Glacier Inst ●──────────────┤ Archive but instant
Glacier Flex ●──────────────┤ Minutes to hours
Deep Archive ●──────────────┤ Cheapest + slowest
                             ↓
                        Slow retrieval (hours/days)
```

**Minimum storage duration explained:**
- If you upload to S3 Standard-IA and delete after 10 days, you're charged for 30 days
- This is AWS preventing you from using IA/Glacier as cheap Standard

---

### 1.3 S3 Lifecycle Policies

Automatically move objects between storage classes based on age.

```
Lifecycle Rule Example:
─────────────────────────────────────────────────────
Day 0:   Object uploaded → S3 Standard
Day 30:  Transition to S3 Standard-IA (not needed as often)
Day 90:  Transition to S3 Glacier Flexible (archived)
Day 365: Delete permanently (compliance window passed)
```

**CLI to create lifecycle rule:**
```json
{
  "Rules": [
    {
      "ID": "archive-old-objects",
      "Status": "Enabled",
      "Filter": { "Prefix": "logs/" },
      "Transitions": [
        { "Days": 30, "StorageClass": "STANDARD_IA" },
        { "Days": 90, "StorageClass": "GLACIER" }
      ],
      "Expiration": { "Days": 365 }
    }
  ]
}
```

> **Exam tip:** Lifecycle policies are the answer when the question says "automatically move to cheaper storage after X days" or "reduce storage costs for older data."

---

### 1.4 S3 Versioning

Keeps multiple versions of the same object. Protects against accidental overwrites/deletes.

```
Without versioning:           With versioning:
  PUT logo.png v2               PUT logo.png v1 → version: abc
  → v1 is gone                  PUT logo.png v2 → version: def (latest)
                                DELETE logo.png → adds "delete marker" (v3)
                                → GET still finds v2 (versioned)
                                → You can recover v1 anytime
```

**Key behaviors:**
- Once enabled on a bucket, **cannot be disabled** (only suspended)
- DELETE creates a "delete marker" — doesn't actually delete the file
- To permanently delete: delete the specific version ID
- Storage cost: all versions count toward your bill

---

### 1.5 S3 Replication

Automatically copy objects to another bucket.

| Type | Where | Use Case |
|------|-------|---------|
| **Same-Region Replication (SRR)** | Different bucket, same region | Log aggregation, dev/prod copy |
| **Cross-Region Replication (CRR)** | Different bucket, different region | DR, latency reduction for global users, compliance |

**Requirements:**
- Versioning must be enabled on **both** source and destination
- IAM role must allow S3 to replicate
- Replication is **not retroactive** — only new objects after rule is set are replicated

> **Exam trap:** Replication doesn't replicate delete markers by default (optional setting). Deleting an object in source does NOT delete it in destination (without explicit config).

---

### 1.6 S3 Encryption

S3 supports multiple encryption modes. Know them all.

| Type | Full Name | Key Management | When To Use |
|------|-----------|---------------|------------|
| **SSE-S3** | Server-Side Encryption with S3 Keys | AWS manages everything | Default, simplest |
| **SSE-KMS** | Server-Side Encryption with KMS Keys | You control the CMK in KMS | Audit trail needed, fine-grained key control |
| **SSE-C** | Server-Side Encryption with Customer Keys | You provide the key per request | You must own the encryption key |
| **Client-side** | Encrypt before uploading | You own everything | Maximum control |

```
SSE-S3:    AWS encrypts/decrypts transparently. You see nothing.
SSE-KMS:   Uses AWS KMS CMK. CloudTrail logs every use. Key rotation supported.
SSE-C:     You send key with every PUT/GET request. S3 never stores your key.
Client:    Your app encrypts → sends encrypted blob to S3 → decrypts on read.
```

**Enforce encryption with bucket policy:**
```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:PutObject",
  "Resource": "arn:aws:s3:::my-bucket/*",
  "Condition": {
    "StringNotEquals": {
      "s3:x-amz-server-side-encryption": "aws:kms"
    }
  }
}
```

This forces all uploads to use SSE-KMS or they're rejected.

---

### 1.7 S3 Access Control — Four Layers

S3 has multiple overlapping access control mechanisms. This confuses candidates.

```
Layer 1: Block Public Access Settings (account or bucket level)
  → Master switch. If ON, overrides all other settings.
  → Always ON for sensitive buckets.

Layer 2: Bucket Policy (resource-based JSON policy)
  → Controls who can access the bucket from anywhere
  → Used for: cross-account access, forcing HTTPS, IP restrictions

Layer 3: ACL (Access Control Lists) — LEGACY
  → Object-level permissions. AWS recommends disabling ACLs now.
  → Use bucket policies instead.

Layer 4: IAM Policies (identity-based)
  → Controls what YOUR IAM users/roles can do with S3
```

**Force HTTPS only — bucket policy:**
```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"],
  "Condition": {
    "Bool": { "aws:SecureTransport": "false" }
  }
}
```

---

### 1.8 S3 Pre-Signed URLs

Gives temporary access to private objects without changing permissions.

```
Use case: Your app has private files in S3. A user logs in and wants to download their report.

Without pre-signed URL: Make the S3 object public? No — security risk.

With pre-signed URL:
  1. Your backend (with IAM access) generates pre-signed URL with 15-min expiry
  2. User gets the URL
  3. User downloads directly from S3 (no backend involved)
  4. URL expires after 15 minutes

Generate via CLI:
  aws s3 presign s3://my-bucket/report.pdf --expires-in 900
```

---

### 1.9 S3 Static Website Hosting

Host a website directly from S3 without any servers.

```
Config:
  Bucket Properties → Static website hosting → Enable
  Index document: index.html
  Error document: error.html

Endpoint: http://my-bucket.s3-website-us-east-1.amazonaws.com

Notes:
  - Uses HTTP only (not HTTPS) from S3 directly
  - For HTTPS: put CloudFront in front of S3
  - The bucket must be PUBLIC (or use CloudFront with OAC)
  - Cost: S3 storage + data transfer only (no EC2)
```

---

### 1.10 S3 Performance Optimization

- **Multipart Upload:** Split large files into parts and upload in parallel. Recommended for > 100 MB, required for > 5 GB.
- **S3 Transfer Acceleration:** Route uploads through CloudFront edge locations for faster upload from distant locations.
- **Byte-Range Fetches:** Download specific byte range of an object in parallel parts (like multipart for downloads).
- **Prefix Performance:** S3 supports 3,500 PUT/s and 5,500 GET/s **per prefix**. If you need more throughput, spread objects across multiple prefixes.

```
Low throughput:                    High throughput:
my-bucket/file-001.dat             my-bucket/a/file-001.dat
my-bucket/file-002.dat             my-bucket/b/file-002.dat
my-bucket/file-003.dat             my-bucket/c/file-003.dat
(all same prefix → shares limit)   (3 prefixes → 3x the limit)
```

---

## SECTION 2: EBS — Elastic Block Store

---

### 2.1 EBS Volume Types Deep Dive

| Type | Subtype | Max IOPS | Max Throughput | Best For |
|------|---------|---------|--------------|---------|
| SSD — General | **gp3** | 16,000 | 1,000 MB/s | Default. Most workloads. Choose over gp2. |
| SSD — General | **gp2** | 16,000 | 250 MB/s | Legacy default. IOPS linked to size (3 IOPS/GB) |
| SSD — Provisioned | **io2 Block Express** | 256,000 | 4,000 MB/s | SAP HANA, high-perf databases |
| SSD — Provisioned | **io1** | 64,000 | 1,000 MB/s | I/O-intensive databases |
| HDD — Throughput | **st1** | N/A | 500 MB/s | Big data, Kafka, sequential workloads |
| HDD — Cold | **sc1** | N/A | 250 MB/s | Infrequently accessed archives |

**gp2 vs gp3 — Know This:**
```
gp2: IOPS = 3 × size (GB). To get more IOPS, you must buy bigger volume.
gp3: IOPS and throughput are independent. Set them separately. Default 3,000 IOPS.
     → Cheaper than gp2 for same performance. Always choose gp3 for new volumes.
```

**Cannot use as boot volume:** st1, sc1 (HDD types)

---

### 2.2 EBS Snapshots

Snapshots are incremental backups of EBS volumes stored in S3.

```
Day 1: Full snapshot → stores all 20 GB
Day 2: Incremental → stores only changed blocks (e.g., 2 GB)
Day 3: Incremental → stores changed blocks (e.g., 1 GB)
Total stored: 23 GB (but each snapshot is independently restorable)
```

**Snapshot operations:**
- **Create:** From EC2 console or CLI while instance running (consistent snapshot requires quiescing app)
- **Copy:** Copy to another region for DR
- **Share:** Share with another AWS account
- **Create volume from snapshot:** Restore, or change volume type/size when restoring
- **AMI:** Create AMI from snapshot (includes volume + configuration)

**EBS Snapshot Archive:** Move snapshots to archive tier (~75% cheaper). Retrieval takes 24-72 hours.

---

### 2.3 EBS Multi-Attach

Only available for **io1/io2** volumes in the **same AZ**.

```
io2 volume ──┬── EC2-1 (AZ-a) — read/write
             ├── EC2-2 (AZ-a) — read/write
             └── EC2-3 (AZ-a) — read/write

Use case: Clustered databases that manage concurrent access themselves
          (Oracle RAC, distributed FS like OCFS2)
Limit: Max 16 instances attached simultaneously
```

> **Exam trap:** Multi-Attach doesn't work across AZs. All instances must be in the same AZ.

---

### 2.4 Instance Store (vs EBS)

Instance Store is **physically attached** local storage (NVMe SSDs).

| Feature | Instance Store | EBS |
|---------|---------------|-----|
| Performance | Extremely fast (hundreds of thousands IOPS) | Fast (up to 256k IOPS with io2) |
| Persistence | **Ephemeral** — data lost on stop/terminate | Persistent — survives stop/start |
| Backup | No snapshot support | Full snapshot support |
| Cost | Included in instance price | Billed separately |
| Use case | Temporary buffers, cache, scratch data | Everything persistent |

> **Critical:** If an EC2 with Instance Store stops or terminates, **data is gone forever.** Only use for temporary data or as cache.

---

## SECTION 3: EFS — Elastic File System

---

### 3.1 What Is EFS?

EFS is a **managed NFS (Network File System)** that multiple EC2 instances can mount simultaneously. Think of it as a shared drive in the cloud.

```
BEFORE EFS:
  EC2-1 (AZ-a) → its own EBS → can't share files with EC2-2
  EC2-2 (AZ-b) → its own EBS

WITH EFS:
  EC2-1 (AZ-a) ──┐
  EC2-2 (AZ-b) ──┼──→ [EFS] (shared file system)
  EC2-3 (AZ-c) ──┘
  All see the same files simultaneously
```

---

### 3.2 EFS vs EBS vs S3

| Feature | EFS | EBS | S3 |
|---------|-----|-----|-----|
| Type | File storage (NFS) | Block storage | Object storage |
| Protocol | NFS v4 | Raw block device | HTTP REST API |
| Concurrent access | Yes (thousands of clients) | No (one EC2, except io1/io2 multi-attach) | Yes (via API) |
| OS compatibility | Linux only | Linux and Windows | Any |
| Performance mode | General, Max I/O | Depends on type | N/A |
| Throughput mode | Bursting, Provisioned, Elastic | Provisioned | Unlimited |
| Cost | ~$0.30/GB/month | ~$0.10/GB/month (gp3) | ~$0.023/GB/month (Standard) |
| Scalability | Petabytes, auto-scales | Manual resize | Unlimited |
| Use case | Shared content, CMS, home dirs | Database volumes, boot | Static files, backup, archive |

---

### 3.3 EFS Storage Classes

| Class | When Used | Cost |
|-------|----------|------|
| **EFS Standard** | Frequently accessed files | ~$0.30/GB |
| **EFS Standard-IA** | Files not accessed for 30+ days | ~$0.025/GB (90% cheaper) |
| **EFS One Zone** | Single AZ, less HA, less cost | ~$0.16/GB |
| **EFS One Zone-IA** | Single AZ + infrequent access | ~$0.01/GB |

**Lifecycle policy:** Automatically move files to IA after 7, 14, 30, 60, or 90 days of no access.

---

## SECTION 4: Storage Gateway

---

### 4.1 What Is Storage Gateway?

A hybrid service that lets your on-premises applications use AWS cloud storage. You install a VM (or hardware appliance) on-prem.

```
ON-PREMISES                         AWS CLOUD
────────────────────────────────────────────────
Your apps → [Storage Gateway VM] ──→ S3 / Glacier / EBS
            (caches hot data locally)
```

### 4.2 Three Types of Storage Gateway

| Gateway Type | Protocol | What It Does | Backend |
|-------------|---------|-------------|---------|
| **S3 File Gateway** | NFS/SMB | Mount S3 as file share on-prem | S3 |
| **Volume Gateway** | iSCSI (block) | On-prem block storage backed by S3 snapshots | S3 + EBS snapshots |
| **Tape Gateway** | iSCSI VTL | Replace physical tape backup with virtual tapes | S3 Glacier |

**S3 File Gateway — Most common exam question:**
```
On-prem apps (NFS/SMB) → File Gateway → S3 (files appear as objects)
Hot files cached locally on gateway → fast access
Cold files retrieved from S3 → slower but cheap
```

**Tape Gateway:**
```
Backup software → "Tape library" (virtual, on gateway) → Archives to S3 Glacier
No more physical tapes. Same backup software works unchanged.
```

---

## SECTION 5: WEEK 3 LAB

---

### Lab 1: S3 Static Website + Lifecycle Policy + CRR

**Step 1: Create source bucket (us-east-1)**
```
S3 → Create bucket
  Name: my-website-source-[yourname]
  Region: us-east-1
  Block all public access: OFF (needed for static website)
  Versioning: ENABLE
```

**Step 2: Enable static website hosting**
```
Bucket → Properties → Static website hosting → Enable
Index document: index.html
Note the endpoint URL
```

**Step 3: Add bucket policy for public read**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-website-source-[yourname]/*"
  }]
}
```

**Step 4: Upload index.html**
```html
<!DOCTYPE html>
<html>
<body>
  <h1>Hello from S3 Static Website!</h1>
</body>
</html>
```

**Step 5: Create lifecycle policy**
```
Bucket → Management → Lifecycle rules → Create rule
  Rule name: archive-old-content
  Filter: Prefix = logs/
  Transitions:
    30 days → Standard-IA
    90 days → Glacier Flexible Retrieval
  Expiration:
    365 days → expire current versions
    180 days → permanently delete noncurrent versions
```

**Step 6: Set up Cross-Region Replication**
```
Create destination bucket in us-west-2:
  Name: my-website-dest-[yourname]
  Region: us-west-2
  Versioning: ENABLE (required)

Source bucket → Management → Replication rules → Create rule
  Source: my-website-source-[yourname]
  Destination: my-website-dest-[yourname]
  IAM role: Create new role (AWS creates automatically)
  Enable

Upload a file to source → verify it appears in us-west-2 bucket
```

**Step 7: Test pre-signed URL**
```bash
aws s3 presign s3://my-website-source-[yourname]/index.html \
  --expires-in 60 \
  --region us-east-1

# Open the URL in browser — it works
# Wait 60 seconds and try again — it should fail (403)
```

**Cleanup:**
```bash
# Empty both buckets first (delete all versions)
aws s3 rm s3://my-website-source-[yourname] --recursive
aws s3 rm s3://my-website-dest-[yourname] --recursive

# Then delete buckets
aws s3 rb s3://my-website-source-[yourname]
aws s3 rb s3://my-website-dest-[yourname]
```

---

## SECTION 6: EXAM QUICK REFERENCE

### S3 Storage Class Cheat Sheet

| Scenario | Answer |
|----------|--------|
| Website images accessed constantly | S3 Standard |
| Unknown access pattern, optimize cost automatically | S3 Intelligent-Tiering |
| Monthly financial reports, accessed rarely | S3 Standard-IA |
| Compliance archive, read once in 7 years | S3 Glacier Deep Archive |
| Retrieve archived files in milliseconds | S3 Glacier Instant Retrieval |
| Cheapest single-AZ storage, infrequent access | S3 One Zone-IA |
| Automatically move old logs to cheaper storage | S3 Lifecycle Policy |
| Protect against accidental deletes | S3 Versioning |
| DR copy in another region | Cross-Region Replication |
| On-prem apps mount S3 as file share | S3 File Gateway |
| Replace tape backup library | Tape Gateway |

### EBS/EFS Cheat Sheet

| Scenario | Answer |
|----------|--------|
| Database volume needing highest IOPS | io2 Block Express |
| Default volume for most workloads, cost-effective | gp3 |
| Multiple Linux EC2s share same files | EFS |
| Windows shared file system in AWS | FSx for Windows |
| Scratch/temp data, max speed, okay to lose | Instance Store |
| EC2 volume in another AZ | Snapshot → Restore in new AZ |
| EC2 volume in another region | Snapshot → Copy snapshot → Restore |
| Shared block storage for clustered DB | io1/io2 with Multi-Attach |

---

*Next: [week4-databases.md](week4-databases.md)*
