# WEEK 5: Networking & Content Delivery — Route 53 + CloudFront + API Gateway
### AWS Solutions Architect Associate Study Notes

---

## WHY THIS MATTERS (Read This First)

Every app on the internet needs: a domain name → DNS → network delivery. These are the services that sit between your users and your AWS infrastructure.

The exam loves routing policy questions (Route 53) and the difference between CloudFront and Global Accelerator. Get these concepts right and you score well on the "Design High-Performing Architectures" domain.

Mental model:
- **Route 53** = DNS. Takes "example.com" and tells users where to go.
- **CloudFront** = CDN. Caches your content at edge locations worldwide (fast delivery).
- **Global Accelerator** = Routing optimizer. Routes TCP/UDP traffic via AWS backbone (not cache).
- **API Gateway** = Front door for your APIs. Manages, throttles, authenticates API calls.

---

## SECTION 1: Route 53

---

### 1.1 What Is Route 53?

Route 53 is AWS's managed DNS (Domain Name System) service. It translates human-readable domain names into IP addresses.

```
User types: www.example.com
                │
                ▼ DNS query
           Route 53
                │
                ▼ Returns: 52.45.123.45 (IP of your server)
User's browser connects to 52.45.123.45
```

Why "Route 53"? Port 53 is the DNS port. It's also a reference to US Route 66, but with 53.

---

### 1.2 DNS Record Types — Know All Of These

| Record | What It Does | Example |
|--------|-------------|---------|
| **A** | Maps hostname to IPv4 address | example.com → 52.45.123.45 |
| **AAAA** | Maps hostname to IPv6 address | example.com → 2600:1f18::45 |
| **CNAME** | Maps hostname to another hostname | www.example.com → example.com |
| **Alias** | AWS-specific. Maps to AWS resources (ALB, CloudFront, S3) | example.com → alb-123.us-east-1.elb.amazonaws.com |
| **MX** | Mail exchange (email routing) | example.com → mail.example.com |
| **TXT** | Text record (domain verification, SPF) | Arbitrary text |
| **NS** | Name server records | Which servers are authoritative for your domain |
| **SOA** | Start of authority | Zone metadata |
| **SRV** | Service locator (SIP, XMPP) | Service-specific records |

---

### 1.3 CNAME vs Alias — Critical Difference

**This is one of the most tested Route 53 concepts.**

| Feature | CNAME | Alias |
|---------|-------|-------|
| AWS-specific | No | Yes |
| Works at zone apex (naked domain) | **NO** | **YES** |
| Charged for DNS queries | Yes | **Free** |
| Points to | Any hostname | AWS resources only (ALB, CloudFront, S3 website, etc.) |
| TTL | You set it | Set by AWS |

```
CNAME LIMITATION — Zone Apex Problem:
  www.example.com → CNAME → alb.amazonaws.com   ✓ (subdomain works)
  example.com     → CNAME → alb.amazonaws.com   ✗ (root/apex domain can't use CNAME!)

ALIAS SOLUTION:
  example.com → ALIAS → alb.amazonaws.com        ✓ (Alias works at apex)
  www.example.com → ALIAS → alb.amazonaws.com   ✓ (works for subdomains too)
```

> **Rule:** If the domain is an apex (no subdomain), always use **Alias**. For AWS resources, prefer Alias over CNAME (it's free and faster).

---

### 1.4 Route 53 Routing Policies — The Most Tested Topic

**You need to know which routing policy solves which problem.**

#### Simple Routing
```
DNS query → Route 53 → returns single IP (or one of multiple IPs randomly)
No health checks (default). If you specify multiple IPs, client picks randomly.
Use: Single resource, no health checks needed.
```

#### Weighted Routing
```
Route X% of traffic to Resource A
Route Y% of traffic to Resource B

Example:
  Weight 70: EC2 in us-east-1 (main)
  Weight 30: EC2 in us-west-2 (canary test)

Traffic split: 70/100 goes to us-east-1, 30/100 to us-west-2

Use: A/B testing, canary deployments, gradual migration.
```

#### Latency Routing
```
User in Tokyo → Route 53 checks: which AWS region has lowest latency from Tokyo?
  → us-west-2? 120ms
  → ap-northeast-1? 5ms   ← WINNER
  → Route user to ap-northeast-1 resource

Use: Global app. Route users to lowest-latency region.
Note: Latency is measured by AWS (not real-time ping from user).
```

#### Failover Routing
```
Primary resource: healthy → all traffic goes here
Primary fails health check → Route 53 routes all traffic to Secondary

PRIMARY:   my-app in us-east-1
SECONDARY: my-app in us-west-2 (DR site)

Use: Active-passive disaster recovery.
```

#### Geolocation Routing
```
Based on the geographic location of the user's DNS resolver:

Users in India → Route to Mumbai region
Users in EU → Route to Frankfurt region
Users elsewhere → Route to default (must have a default)

Use: Content localization, language-specific content, data residency compliance.
Note: This is based on LOCATION, not latency. Different from Latency routing.
```

#### Geoproximity Routing (Traffic Flow only)
```
Route based on location, with "bias" to expand/shrink geographic coverage area.

Example: Shift 20% of North American traffic from us-east-1 to us-west-2
  → Increase us-west-2 bias by +20

Use: Fine-grained geographic traffic shifting (not available without Traffic Flow).
```

#### Multi-Value Answer Routing
```
Returns up to 8 healthy IP addresses randomly.
Clients randomly select one.
Works with health checks (unhealthy resources removed from responses).

Use: Simple load distribution with health checks.
NOT a replacement for a real load balancer.
```

---

### 1.5 Route 53 Health Checks

Route 53 can monitor your endpoints and automatically route away from failures.

```
Health check types:
  1. HTTP/HTTPS endpoint health check (monitors URL, checks status code + string match)
  2. Other CloudWatch Alarm (composite health checks)
  3. Calculated health checks (combine multiple checks: all pass, at least X pass, etc.)

For private resources (in VPC, no public IP):
  → Use CloudWatch Alarm → Health Check monitors the alarm
  (Health checks come from Route 53 servers which are external to your VPC)
```

---

### 1.6 Route 53 Resolver (Hybrid DNS)

**For on-premises to AWS DNS resolution:**

```
SCENARIO: Your on-prem servers need to resolve AWS private hostnames
          (e.g., mydb.us-east-1.rds.amazonaws.com)

SOLUTION: Route 53 Resolver
  Inbound Endpoint: On-prem DNS → forwards queries to AWS Route 53
  Outbound Endpoint: AWS → forwards queries to on-prem DNS

On-Prem DNS Server
       │ DNS query for RDS endpoint
       ▼
  Route 53 Inbound Endpoint (in VPC)
       │
       ▼
  Route 53 Resolver → answers query with private IP
```

---

## SECTION 2: CloudFront

---

### 2.1 What Is CloudFront?

CloudFront is AWS's CDN (Content Delivery Network). It caches your content at **edge locations** worldwide, so users get content from the closest server instead of traveling all the way to your origin.

```
WITHOUT CloudFront:
  User in Tokyo → www.example.com → ALB in us-east-1 (Virginia)
  Distance: 10,000+ km → High latency

WITH CloudFront:
  User in Tokyo → www.example.com → CloudFront Edge in Tokyo
  Distance: ~10 km → Ultra-low latency
  (Edge already has the cached content from origin)

Origin regions: 200+ Edge Locations globally
```

---

### 2.2 CloudFront Origins

CloudFront can pull content from (origins):

| Origin Type | Example |
|-------------|---------|
| **S3 bucket** | Static website, images, files |
| **ALB** | Dynamic web content, API responses |
| **EC2 instance** | Direct EC2 (must have public IP) |
| **Custom HTTP server** | Any server with a public IP |

---

### 2.3 CloudFront + S3 — OAC (Origin Access Control)

**The correct, secure way to serve S3 content via CloudFront.**

```
WRONG (public S3 bucket):
  Users → S3 (public) ← anyone can bypass CloudFront, access S3 directly

CORRECT (private S3 + OAC):
  Users → CloudFront → S3 (private, OAC grants CF access)
                        ↑
                   Users CAN'T access S3 directly — only via CloudFront

Setup:
  1. Create CloudFront distribution with S3 origin
  2. Enable OAC (Origin Access Control) on the distribution
  3. CloudFront updates S3 bucket policy automatically to allow only CF principal
  4. Set S3 to Block All Public Access
```

OAC replaced the older OAI (Origin Access Identity) — know OAC is the newer preferred method.

---

### 2.4 CloudFront Cache Behaviors

Different paths can be routed to different origins with different cache settings.

```
CloudFront Distribution (d1234.cloudfront.net)
├── Behavior 1: /static/* → S3 origin → cache TTL 7 days
├── Behavior 2: /api/* → ALB origin → no caching (TTL 0)
└── Behavior 3: /* (default) → ALB origin → cache TTL 1 hour
```

**Cache key:** By default, the URL path. Can include query strings, headers, cookies.

**Cache invalidation:** When you update content at origin, CloudFront still serves old cache. To force refresh:
```bash
aws cloudfront create-invalidation \
  --distribution-id E1234567890 \
  --paths "/*"
# Charges apply per invalidation path (first 1000/month free)
```

---

### 2.5 CloudFront Security Features

| Feature | What It Does |
|---------|-------------|
| **HTTPS** | TLS at edge + TLS to origin (end-to-end encryption) |
| **WAF integration** | Attach AWS WAF to block malicious traffic at edge |
| **Geo Restriction** | Block or allow specific countries |
| **Signed URLs** | Temporary access to specific files (like pre-signed S3 URLs) |
| **Signed Cookies** | Access to multiple restricted files (better for premium subscriptions) |
| **Field-Level Encryption** | Encrypt specific form fields (credit card, SSN) at edge |

**Signed URL vs Signed Cookie:**
```
Signed URL:    Single file. User gets a URL with a timestamp and signature.
               Use for: individual file download, streaming video
Signed Cookie: Multiple files. User gets a cookie with permissions.
               Use for: premium content subscription, access to entire directory
```

---

### 2.6 CloudFront vs Global Accelerator

**The exam loves this comparison. Know it cold.**

| Feature | CloudFront | Global Accelerator |
|---------|-----------|-------------------|
| Purpose | **Cache** content at edge | Route traffic via AWS **backbone** |
| Content type | Static (and some dynamic) | Any TCP/UDP (no caching) |
| Protocol | HTTP/HTTPS | Any TCP/UDP |
| Static IP | No (uses anycast DNS) | **YES — 2 static Anycast IPs** |
| Use case | Website, images, video, APIs | Gaming, IoT, VoIP, non-HTTP apps |
| Performance gain | Cache hit = no origin roundtrip | Always uses fast AWS backbone |
| Health checks | No built-in (use ALB/Route 53) | Yes — automatic failover |

```
CLOUDFRONT PATH:
  User → CloudFront Edge (cache hit?) → Yes: return cached
                                      → No: go to origin

GLOBAL ACCELERATOR PATH:
  User → AWS Anycast IP → AWS Edge → AWS Backbone Network → your ALB/EC2
  (every request goes to origin, but via AWS's fast private network)

KEY DIFFERENCE:
  CloudFront CACHES content → great for static content
  Global Accelerator ROUTES traffic → great for dynamic content, TCP/UDP
```

> **Exam tip:** "Static IP for load balancer" → Global Accelerator. "Cache images globally" → CloudFront.

---

## SECTION 3: API Gateway

---

### 3.1 What Is API Gateway?

API Gateway is a fully managed service for creating, deploying, and managing APIs. It's the front door for your backend services.

```
Clients (mobile, web, 3rd party)
    │
    ▼
[API Gateway]
    │ Handles: auth, throttling, transformation, HTTPS, caching, logging
    │
    ├──→ Lambda (serverless)
    ├──→ EC2 / ECS
    ├──→ Any HTTP backend (on-prem, other services)
    └──→ AWS services directly (DynamoDB, SQS, etc.)
```

---

### 3.2 API Gateway Types

| Type | Protocol | Use Case |
|------|---------|---------|
| **REST API** | HTTP/REST | Standard web APIs. Lots of features. Most common. |
| **HTTP API** | HTTP/REST | Simpler, cheaper (~70% less), fewer features. Lambda + OIDC/JWT only. |
| **WebSocket API** | WebSocket | Real-time two-way communication (chat apps, live notifications, gaming) |

**REST API vs HTTP API:**
```
REST API features: 
  ✓ API keys
  ✓ Usage plans and throttling
  ✓ WAF integration
  ✓ Resource policies
  ✓ Private APIs
  ✓ Request/response transformation
  ✓ Caching

HTTP API features (subset):
  ✓ JWT authorization (Cognito, any OIDC provider)
  ✓ Lambda proxy
  ✓ Lower cost and latency
  ✗ No usage plans
  ✗ No WAF
  ✗ No transformation
```

> **Rule:** If the question needs WAF, API keys, or transformation → REST API. If it says "cheapest serverless API" → HTTP API.

---

### 3.3 API Gateway Integration Types

How API Gateway talks to your backend:

| Type | How It Works |
|------|-------------|
| **Lambda Proxy** | Passes entire request as-is to Lambda. Lambda returns full response. |
| **Lambda Custom** | You transform request/response in API Gateway (mapping templates). |
| **HTTP Proxy** | Passes entire request to HTTP endpoint unchanged. |
| **HTTP Custom** | Transform request/response in API Gateway. |
| **AWS Service** | Directly call AWS services (DynamoDB, SQS, SNS, Step Functions). |

---

### 3.4 API Gateway Caching

API Gateway can cache responses to reduce backend hits.

```
Config: Enable caching on stage with TTL (default 300 seconds, max 3600)

Client → API Gateway → (cache hit?) → Yes: return cached response immediately
                                    → No: call backend → cache result → return

Invalidate cache: Set Cache-Control: max-age=0 header in request
```

---

### 3.5 API Gateway Security

| Method | How |
|--------|-----|
| **IAM Authorization** | Sign requests with IAM credentials (SigV4). For AWS internal services. |
| **Cognito User Pools** | JWT tokens from Cognito. For your app users. |
| **Lambda Authorizer** | Custom auth logic (validate JWT, OAuth, call external IdP). |
| **API Keys + Usage Plans** | Give API keys to clients. Throttle and quota by key. |
| **Resource Policy** | JSON policy on the API itself. Allow specific IPs, VPCs, accounts. |

---

## SECTION 4: WEEK 5 LAB

---

### Lab: CloudFront + S3 with OAC + HTTPS

**Step 1: Create private S3 bucket**
```
S3 → Create bucket
  Name: my-cf-origin-[yourname]
  Region: us-east-1
  Block all public access: ON (all 4 checkboxes)
  Versioning: optional
```

**Step 2: Upload content**
```html
<!-- Upload as index.html -->
<!DOCTYPE html>
<html>
<body>
  <h1>Served via CloudFront!</h1>
  <p>This content is private in S3 but accessible via CloudFront.</p>
</body>
</html>
```

**Step 3: Create CloudFront Distribution**
```
CloudFront → Create Distribution
  Origin domain: my-cf-origin-[yourname].s3.us-east-1.amazonaws.com
  Origin access: Origin access control settings (OAC)
    → Create new OAC (CloudFront will update S3 policy automatically)
  Default cache behavior:
    Viewer protocol: Redirect HTTP to HTTPS
    Cache policy: CachingOptimized
  Default root object: index.html
  Price class: Use only North America and Europe (cheapest for testing)
  Create Distribution
```

**Step 4: Update S3 bucket policy**
```
CloudFront will show a notification: "Update S3 bucket policy"
Click "Copy policy" → Go to S3 bucket → Permissions → Bucket policy → Paste → Save
```

**Step 5: Test**
```
# Wait 5-10 minutes for CloudFront to deploy (global edge network)
# Access via CloudFront domain name:
  https://d1234567890.cloudfront.net/index.html

# Verify S3 is NOT directly accessible:
  https://my-cf-origin-[yourname].s3.amazonaws.com/index.html
  → Should get 403 Access Denied ✓
```

**Step 6: Set up geo restriction**
```
CloudFront Distribution → Security → Restrictions
  Restriction type: Block List (or Allow List)
  Countries: Add a country to block
  Save

# Test from blocked country (or use VPN) → should get 403
```

**Step 7: Cleanup**
```
Disable CloudFront distribution first (takes 5-10 min)
Delete distribution
Delete S3 bucket
```

---

### Lab 2: Route 53 Weighted Routing

**Step 2: Create two EC2 instances in different regions** (us-east-1 and us-west-2)
```bash
# On each EC2:
echo "<h1>Instance in us-east-1</h1>" > /var/www/html/index.html
# or
echo "<h1>Instance in us-west-2</h1>" > /var/www/html/index.html
```

**Step 2: Register or transfer a domain in Route 53** (or use the free subdomain from a free DNS service)

**Step 3: Create weighted records**
```
Route 53 → Hosted Zone → Create Record
  Record type: A
  Value: us-east-1 instance IP
  Weight: 70
  Set ID: "us-east-1"
  Health check: create health check for this endpoint

Create another record:
  Value: us-west-2 instance IP
  Weight: 30
  Set ID: "us-west-2"
```

**Step 4: Test**
```bash
# Run multiple times and observe different responses
for i in {1..10}; do
  curl -s http://your-domain.com
done
# Approximately 7/10 responses from us-east-1, 3/10 from us-west-2
```

---

## SECTION 5: EXAM QUICK REFERENCE

### Route 53 Routing Policy Cheat Sheet

| Scenario | Answer |
|----------|--------|
| Route 10% to new version, 90% to stable | Weighted |
| Route users to lowest latency region | Latency |
| Failover to DR if primary fails | Failover |
| Route EU users to EU, Asia to Asia | Geolocation |
| Block specific country | Geolocation (block default) |
| Return multiple IPs with health checks | Multi-Value |
| Simple single record | Simple |
| Map root domain (example.com) to ALB | Alias record (not CNAME) |
| Hybrid DNS: on-prem resolve Route 53 hostnames | Route 53 Inbound Resolver Endpoint |

### CloudFront Cheat Sheet

| Scenario | Answer |
|----------|--------|
| Cache static content globally | CloudFront |
| Serve S3 content privately | CloudFront + S3 OAC |
| Block users from specific country | CloudFront geo restriction |
| Temporary access to private S3 file | Signed URL |
| Access to premium subscription content | Signed Cookie |
| Cache dynamic API responses | CloudFront with cache behavior TTL |
| Block SQL injection/XSS at edge | CloudFront + WAF |
| Static IP for global load balancing | Global Accelerator |
| TCP/UDP game server global routing | Global Accelerator |

### API Gateway Cheat Sheet

| Scenario | Answer |
|----------|--------|
| Serverless REST API | API Gateway HTTP API + Lambda |
| API needs WAF, API keys, throttling | API Gateway REST API |
| Real-time chat application | API Gateway WebSocket |
| Cheapest API for Lambda | HTTP API (70% cheaper than REST) |
| Auth via Cognito User Pools | Cognito authorizer on API Gateway |
| Custom auth logic (OAuth, etc.) | Lambda Authorizer |
| Rate limiting by client | Usage Plans + API Keys |

---

*Next: [week6-security-monitoring.md](week6-security-monitoring.md)*
