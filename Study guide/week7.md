# WEEK 7: Serverless + Application Integration
### AWS Solutions Architect Associate Study Notes

---

## WHY THIS MATTERS (Read This First)

Serverless is one of the fastest-growing areas of AWS. The exam tests whether you can design event-driven, decoupled architectures using Lambda, SQS, SNS, and ECS.

The key skill: **recognizing when to decouple** (separate) parts of an architecture using messaging queues and events.

Rule of thumb:
- **Lambda** = Run code without servers. Triggered by events. Max 15 minutes.
- **SQS** = Queue messages between components. Async. At-least-once delivery.
- **SNS** = Publish a message to many subscribers at once. Fan-out.
- **EventBridge** = Route events from AWS services or your app to targets.
- **Step Functions** = Orchestrate multi-step workflows with visual state machines.
- **Kinesis** = Real-time data streaming (think logs, clickstreams, IoT).

---

## SECTION 1: AWS Lambda

---

### 1.1 What Is Lambda?

Lambda runs your code in response to events — without you managing servers.

```
TRADITIONAL SERVER:
  → Rent EC2
  → Install OS, runtime, app
  → Keep it running 24/7
  → Pay 24/7

LAMBDA:
  → Upload code
  → Define trigger (API call, S3 upload, SQS message, etc.)
  → Lambda runs ONLY when triggered
  → Pay ONLY for execution time (milliseconds)
  → 0 cost when not running
```

**Lambda limits to know:**
| Limit | Value |
|-------|-------|
| Max execution time | **15 minutes** |
| Memory | 128 MB to 10 GB |
| CPU | Proportional to memory (more RAM = more CPU) |
| Deployment package (zip) | 50 MB zipped, 250 MB unzipped |
| Deployment package (container) | 10 GB |
| Concurrent executions (default) | 1,000 per region (soft limit, can increase) |
| Ephemeral storage (/tmp) | 512 MB to 10 GB |

---

### 1.2 Lambda Invocation Types

| Type | How It Works | Caller Waits? | Use Case |
|------|-------------|--------------|---------|
| **Synchronous** | Caller sends request, waits for response | YES | API Gateway → Lambda |
| **Asynchronous** | Lambda queues the event, processes later. Caller doesn't wait. | NO | S3 event → Lambda, SNS → Lambda |
| **Event Source Mapping** | Lambda polls a source and processes in batches | N/A (Lambda polls) | SQS → Lambda, DynamoDB Streams, Kinesis |

**Async retry behavior:**
```
Async invocation fails:
  → Retry 2 more times (with delays)
  → After all retries fail → send to Dead Letter Queue (SQS or SNS)
```

---

### 1.3 Lambda Concurrency

```
Concurrency = number of Lambda instances running simultaneously

Reserve concurrency: Set max for a specific function
  → Prevents function from using all 1000 regional concurrency
  → Also guarantees minimum availability for that function

Provisioned concurrency: Pre-warm Lambda instances
  → Eliminates "cold start" (first invocation delay)
  → Use for latency-sensitive functions behind API Gateway
```

**Cold start problem:**
```
Cold start: First invocation after Lambda scales up
  → AWS creates new container, loads runtime, loads your code
  → Adds ~100ms to 1s+ of latency

Cold start is an issue for:
  → APIs where users notice latency
  → VPC-connected Lambdas (historically worse, now improved)

Solutions:
  → Provisioned concurrency (pre-warm instances)
  → Keep functions warm (scheduled ping every 5 minutes)
  → Minimize package size (smaller = faster load)
  → Use Lambda SnapStart (for Java functions)
```

---

### 1.4 Lambda with VPC

By default, Lambda runs outside your VPC and cannot access private resources (RDS, ElastiCache, private EC2).

```
DEFAULT (No VPC):
  Lambda → Public internet → can't reach private RDS

VPC-connected Lambda:
  Lambda → VPC ENI → Private subnet → RDS (private)

Tradeoff:
  ✓ Can access VPC resources
  ✗ Longer cold start (historically — AWS improved this with HYPERPLANE)
  ✗ If Lambda needs internet access, needs NAT Gateway
```

---

### 1.5 Lambda Layers

Layers are reusable packages (libraries, dependencies, custom runtimes) that you attach to Lambda functions.

```
Without layers:
  Function 1: 50 MB (code + dependencies)
  Function 2: 50 MB (code + same dependencies)

With layers:
  Layer:      45 MB (shared dependencies like numpy, pandas)
  Function 1: 5 MB (just your code) + Layer
  Function 2: 5 MB (just your code) + Layer
```

Benefits:
- Smaller deployment packages
- Reuse across many functions
- Separate update cycles (update layer, all functions benefit)

---

### 1.6 Lambda Versions and Aliases

```
Versions: Immutable snapshots of your function code + config
  v1 → production stable
  v2 → new feature (being tested)
  $LATEST → current development

Aliases: Named pointers to versions (like a tag)
  "production" alias → v1 (100% traffic)
  "staging" alias → v2 (100% traffic)

Traffic shifting with alias:
  "production" alias → v1 (90%) + v2 (10%)  ← canary deployment!
```

---

## SECTION 2: SQS — Simple Queue Service

---

### 2.1 What Is SQS?

SQS is a fully managed message queue service. It decouples components so they don't directly call each other.

```
WITHOUT SQS (tightly coupled):
  Web App → directly calls → Order Processing Service
  If Order Service crashes → Web App fails too
  If traffic spikes → Order Service overwhelmed

WITH SQS (decoupled):
  Web App → SQS Queue → Order Processing Service
  Web App just puts message in queue and responds immediately
  Order Service reads from queue at its own pace
  If Order Service crashes → messages stay in queue, nothing lost
  If traffic spikes → queue absorbs the load (buffers it)
```

---

### 2.2 SQS Message Lifecycle

```
Producer                Queue                  Consumer
───────────────────────────────────────────────────────
1. Send message     →   message arrives
                        (visibility: visible)

                    2. Consumer polls         3. Consumer receives message
                       for messages     →        (visibility: HIDDEN — visibility timeout)

                                              4. Consumer processes message
                                              5a. SUCCESS → Consumer deletes message
                                              5b. FAILURE → Visibility timeout expires
                                                           → message becomes VISIBLE again
                                                           → another consumer can pick it up
```

**Visibility Timeout:** Time a message is hidden after being received. Default 30 seconds. If processing takes longer, consumer must extend the timeout. If not extended, message reappears (potential duplicate processing).

---

### 2.3 Standard Queue vs FIFO Queue

| Feature | Standard Queue | FIFO Queue |
|---------|---------------|-----------|
| Ordering | Best-effort (not guaranteed) | **Strict FIFO (guaranteed)** |
| Delivery | **At-least-once** (may get duplicates) | **Exactly-once** |
| Throughput | Unlimited | 300 TPS (3,000 with batching) |
| Deduplication | No (handle in app) | Yes (deduplication ID) |
| Use case | High-throughput, duplicates okay | Order matters (financial transactions, user actions in sequence) |
| Name suffix | Any name | Must end in `.fifo` |

```
STANDARD QUEUE:
  Message sent: 1, 2, 3
  Consumer may receive: 2, 1, 3 (or 1, 2, 2, 3 — duplicate!)
  → Your app must handle duplicates (idempotency)

FIFO QUEUE:
  Message sent: 1, 2, 3
  Consumer receives: 1, 2, 3 (always, exactly once)
  → Use for: payment processing, inventory updates
```

---

### 2.4 Dead Letter Queue (DLQ)

When a message fails processing repeatedly, send it to a DLQ for investigation.

```
Main Queue → Consumer fails to process → retry (based on maxReceiveCount)
           → After N failures → moved to DLQ

DLQ holds failed messages for:
  → Debugging (why did these fail?)
  → Manual reprocessing (fix the bug, drain DLQ)
  → Alerting (CloudWatch alarm on DLQ depth)
```

---

### 2.5 SQS Key Features

**Long Polling:** Consumer waits up to 20 seconds for messages instead of polling every second.
```
Short polling (default): Consumer asks queue → "any messages?" → "no" → asks again immediately
Long polling:            Consumer asks queue → "any messages?" → queue waits up to 20s
                         → message arrives → delivered immediately

Long polling reduces empty responses → lower cost, lower latency
```

**Message Retention:** Default 4 days, max 14 days.

**Message Size:** Max 256 KB. For larger, use SQS Extended Client Library (stores body in S3, sends pointer in SQS).

**SQS with Auto Scaling:**
```
CloudWatch metric: ApproximateNumberOfMessagesVisible (queue depth)
ASG scaling policy: if depth > 1000 → add EC2 consumer instances
                    if depth < 100 → remove EC2 instances
This is the classic "queue-based scaling" pattern for batch workloads
```

---

## SECTION 3: SNS — Simple Notification Service

---

### 3.1 What Is SNS?

SNS is a pub/sub messaging service. Publish one message → all subscribers get it.

```
PUBLISHER → [SNS Topic] → SUBSCRIBER 1 (SQS Queue)
                       → SUBSCRIBER 2 (Lambda)
                       → SUBSCRIBER 3 (Email)
                       → SUBSCRIBER 4 (HTTP endpoint)
                       → SUBSCRIBER 5 (SMS)
```

**Push model (vs SQS pull model):** SNS pushes to subscribers. SQS waits to be polled.

---

### 3.2 SNS Fan-Out Pattern

**The most important SNS pattern on the exam.**

```
Scenario: An order is placed. Need to:
  1. Update inventory
  2. Send shipping notification
  3. Record analytics
  4. Update loyalty points

BAD (direct coupling):
  Order Service → calls Inventory Service
               → calls Notification Service
               → calls Analytics Service
               → calls Loyalty Service
  (all must be up, all failures cascade to order service)

GOOD (SNS Fan-Out):
  Order Service → SNS Topic "order-placed"
                      ├── SQS: Inventory Queue → Inventory Service
                      ├── SQS: Notification Queue → Notification Service
                      ├── SQS: Analytics Queue → Analytics Service
                      └── Lambda: Loyalty Updater

Each consumer is decoupled. Failures don't cascade.
```

---

### 3.3 SNS Message Filtering

Filter which messages each subscriber receives.

```
SNS Topic: orders

Subscriber 1 (Premium Orders SQS):
  Filter: { "customerType": ["premium"] }
  → Only gets messages where customerType = premium

Subscriber 2 (Standard Orders SQS):
  Filter: { "customerType": ["standard"] }
  → Only gets standard messages

Subscriber 3 (All Orders):
  No filter → gets all messages
```

---

## SECTION 4: EventBridge

---

### 4.1 What Is EventBridge?

EventBridge is a serverless event bus that routes events between AWS services, your apps, and 3rd-party SaaS.

```
EVENT SOURCES:
  AWS services (EC2 state change, S3 upload, RDS snapshot complete, etc.)
  Custom apps (your code sends events via API)
  SaaS partners (Zendesk, Shopify, Datadog, etc.)

EVENT ROUTING:
  EventBridge → Rules → filter by event pattern → route to target

TARGETS:
  Lambda, SQS, SNS, Step Functions, ECS task, Kinesis, API Gateway, etc.
```

**Example rule — trigger Lambda when EC2 terminates:**
```json
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"],
  "detail": {
    "state": ["terminated"]
  }
}
```
→ Route to Lambda for cleanup/alerting

---

### 4.2 SNS vs SQS vs EventBridge

| Feature | SNS | SQS | EventBridge |
|---------|-----|-----|-------------|
| Model | Pub/Sub (push) | Queue (pull) | Event bus (rule-based routing) |
| Persistence | No (fire and forget) | Yes (messages retained up to 14 days) | No (fire and forget) |
| Fanout | Yes (many subscribers) | No (one consumer group) | Yes (many rules/targets) |
| Filtering | Basic message attributes | No | Advanced event pattern matching |
| SaaS integration | No | No | Yes (50+ SaaS partners) |
| Schema registry | No | No | Yes |

---

## SECTION 5: Step Functions

---

### 5.1 What Is Step Functions?

Step Functions orchestrates multi-step workflows as state machines — with visual flow, error handling, retries, and branching.

```
Example: Order Fulfillment Workflow

[Start]
   ↓
[Validate Order] ← Task (calls Lambda)
   ↓ success
[Charge Payment] ← Task (calls Lambda)
   ↓ success
[Update Inventory] ← Task (calls Lambda)
   ↓ success
[Send Confirmation Email] ← Task (calls SNS)
   ↓
[End]

If Charge Payment fails:
   ↓ failure
[Refund Handler] ← Catch/error path
   ↓
[Notify Customer] ← SNS
```

---

### 5.2 Standard vs Express Workflows

| Feature | Standard | Express |
|---------|---------|---------|
| Duration | Up to **1 year** | Up to **5 minutes** |
| Execution history | Yes (queried anytime) | CloudWatch Logs only |
| Execution semantics | **Exactly-once** | At-least-once |
| Rate | 2,000 executions/second | 100,000/second |
| Cost | Per state transition | Per execution + duration |
| Use case | Long-running workflows (order processing, human approval) | High-volume, short workflows (IoT data processing, streaming) |

---

## SECTION 6: Kinesis

---

### 6.1 Kinesis Family Overview

| Service | What It Does |
|---------|-------------|
| **Kinesis Data Streams** | Real-time streaming data pipeline. Custom consumers. |
| **Kinesis Data Firehose** | Managed delivery to S3, Redshift, OpenSearch, Splunk. No consumer code needed. |
| **Kinesis Data Analytics** | Run SQL queries on streaming data in real-time |
| **Kinesis Video Streams** | Ingest and process video streams |

---

### 6.2 Kinesis Data Streams

```
PRODUCERS → [Kinesis Data Streams] → CONSUMERS
  IoT sensors     Shard 1           Lambda
  App logs        Shard 2           Kinesis Data Analytics
  Clickstreams    Shard 3           EC2 consumer app
```

**Shards:**
- Each shard handles 1 MB/s write, 2 MB/s read
- Add shards to increase throughput (reshard)
- Data retained 24 hours (default) to 7 days (extended), up to 365 days

**Ordering:** Within a shard, records are ordered. Use a partition key to group related records in the same shard.

---

### 6.3 Kinesis Data Firehose

Managed delivery — you don't write consumer code. Just point it at a destination.

```
Data Sources → [Kinesis Firehose] → Destinations
  Direct PUT    (transforms data      S3
  Kinesis        with Lambda if        Redshift
  MSK            needed)              OpenSearch
  CloudWatch                          Splunk
  WAF                                 HTTP endpoints
```

**Key facts:**
- Near real-time (60-second buffer minimum, not truly real-time)
- Auto-scales (no shards to manage)
- No replay — data delivered to destination, not re-readable
- Can compress (gzip, snappy), convert format (JSON → Parquet), and transform (Lambda)

---

### 6.4 SQS vs Kinesis

| Feature | SQS | Kinesis Data Streams |
|---------|-----|---------------------|
| Message ordering | No (FIFO queue: yes) | Yes, within a shard |
| Consumer model | Competing consumers (message deleted after ack) | Multiple independent consumer groups |
| Replay | No (deleted after consumption) | Yes (24h-365d retention, replay from any point) |
| Throughput | Unlimited | Shard-based (1 MB/s per shard) |
| Use case | Task queues, job queues | Real-time streaming, multiple consumers, replay needed |

---

## SECTION 7: ECS + Fargate (Containers)

---

### 7.1 ECS Overview

Amazon Elastic Container Service (ECS) runs Docker containers on AWS.

```
ECS ARCHITECTURE:
────────────────────────────────────────────────────
Cluster
  ├── Service 1 (runs Task Definition: web-app, desired 3 tasks)
  │   ├── Task (container: web-app:latest on EC2 instance)
  │   ├── Task (container: web-app:latest on EC2 instance)
  │   └── Task (container: web-app:latest on Fargate)
  └── Service 2 (runs Task Definition: worker, desired 5 tasks)
```

**Launch types:**
- **EC2 launch type:** You provision and manage EC2 instances in the cluster. More control, cheaper for sustained load.
- **Fargate launch type:** AWS provisions compute. No EC2 to manage. Pay per task CPU/RAM. Better for serverless containers.

---

### 7.2 ECS vs EKS vs Fargate

| Service | What It Is | Use When |
|---------|-----------|---------|
| **ECS** | AWS-native container orchestration | Simpler setup, AWS-native tooling, not already using Kubernetes |
| **EKS** | Managed Kubernetes (k8s) | Already using Kubernetes, need k8s ecosystem |
| **Fargate** | Serverless compute for ECS or EKS | No EC2 management, variable/unpredictable load |

> **Exam tip:** "Kubernetes" in the question → EKS. "Managed containers, no EC2" → Fargate. "Simple container deployment" → ECS.

---

## SECTION 8: WEEK 7 LAB

---

### Lab: Serverless API with API Gateway + Lambda + DynamoDB

**Architecture:**
```
User → API Gateway → Lambda → DynamoDB
```

**Step 1: Create DynamoDB table**
```
DynamoDB → Create Table
  Table name: Items
  Partition key: id (String)
  Settings: On-demand
```

**Step 2: Create Lambda function**
```
Lambda → Create function
  Name: ItemsHandler
  Runtime: Python 3.12
  Execution role: Create new with basic Lambda permissions
    (then add DynamoDB access)

Add code:
```

```python
import json
import boto3
import uuid

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Items')

def lambda_handler(event, context):
    method = event.get('httpMethod', '')
    
    if method == 'GET':
        result = table.scan()
        return {
            'statusCode': 200,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps(result['Items'])
        }
    
    elif method == 'POST':
        body = json.loads(event.get('body', '{}'))
        item = {
            'id': str(uuid.uuid4()),
            'name': body.get('name', 'unnamed'),
            'value': body.get('value', '')
        }
        table.put_item(Item=item)
        return {
            'statusCode': 201,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps({'message': 'Created', 'id': item['id']})
        }
    
    return {'statusCode': 405, 'body': 'Method Not Allowed'}
```

**Step 3: Add IAM permissions to Lambda role**
```
IAM → Roles → find ItemsHandler role → Add permissions
  → AmazonDynamoDBFullAccess (or create custom policy for Items table only)
```

**Step 4: Create API Gateway**
```
API Gateway → Create API → REST API
  Name: ItemsAPI
  
Resources → Create Resource: /items

Methods:
  /items → GET → Lambda Integration → ItemsHandler
  /items → POST → Lambda Integration → ItemsHandler

Deploy:
  Actions → Deploy API → Stage: prod
  
Copy the invoke URL: https://xxxxxxxxx.execute-api.us-east-1.amazonaws.com/prod
```

**Step 5: Test the API**
```bash
# Create an item
curl -X POST https://xxxxxxxxx.execute-api.us-east-1.amazonaws.com/prod/items \
  -H "Content-Type: application/json" \
  -d '{"name": "Widget", "value": "Blue"}'

# Get all items
curl https://xxxxxxxxx.execute-api.us-east-1.amazonaws.com/prod/items
```

**Step 6: Add SQS for async processing**
```
SQS → Create queue: item-processing-queue (Standard)

Modify Lambda to also send to SQS on POST:
  sqs = boto3.client('sqs')
  sqs.send_message(
      QueueUrl='https://sqs.us-east-1.amazonaws.com/123/item-processing-queue',
      MessageBody=json.dumps(item)
  )

Create second Lambda: ProcessItem (subscribes to SQS)
  → Add SQS trigger → item-processing-queue
  → Log item to CloudWatch or process further
```

**Cleanup:**
```
Delete API Gateway (APIs → delete)
Delete Lambda functions
Delete DynamoDB table
Delete SQS queue
```

---

## SECTION 9: EXAM QUICK REFERENCE

### Serverless Cheat Sheet

| Scenario | Answer |
|----------|--------|
| Run code in response to S3 upload | Lambda with S3 trigger |
| API without servers | API Gateway + Lambda |
| Long-running task > 15 minutes | EC2 or ECS/Fargate (not Lambda) |
| Process SQS messages in batch | Lambda with SQS Event Source Mapping |
| Schedule code to run every 5 minutes | EventBridge Scheduled Rule → Lambda |
| Orchestrate multi-step workflow with retries | Step Functions |
| Long workflow with human approval | Step Functions Standard Workflow |
| High-volume short workflows | Step Functions Express Workflow |
| One publisher, many consumers | SNS |
| Async task queue, at-most-once processing risk is OK | SQS Standard |
| Tasks must be processed exactly once in order | SQS FIFO |
| Failed messages for investigation | Dead Letter Queue |
| Real-time log/event streaming | Kinesis Data Streams |
| Stream data to S3/Redshift managed delivery | Kinesis Firehose |
| Containers, no EC2 management | Fargate |
| Containers with Kubernetes | EKS |

---

*Next: [week8-cost-exam-prep.md](week8-cost-exam-prep.md)*
