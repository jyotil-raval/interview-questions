<!-- Generated: 2026-04-12 | MH: Golang,AWS | GTH: none | Depth: L1-L6 -->

# Persistent Systems — Golang Developer

## AWS Interview Prep | L1–L6

---

## 🆕 What's New — Last 3 Versions

### AWS 2025 (Major announcements — re:Invent 2024 / Q1 2025)

- **Amazon Nova Models on Bedrock:** AWS's own foundation models (Nova Micro, Lite, Pro) — signals AWS's commitment to owned AI infrastructure; interviewers at Persistent will ask about Bedrock for AI-integrated Go services.
- **EKS Auto Mode (GA):** EKS now manages node provisioning and lifecycle automatically — reduces ops overhead for Go microservice fleets on Kubernetes.
- **Lambda SnapStart for Go (Preview):** Reduces cold start latency by snapshotting the initialised execution environment — directly relevant for Go Lambda services with slow init paths.

### AWS re:Invent 2023 / H1 2024

- **Application Signals (CloudWatch):** Out-of-the-box SLI/SLO monitoring with auto-instrumented latency, error rate, and volume metrics — reduces custom Prometheus setup overhead for Go services.
- **Amazon S3 Express One Zone:** Single-AZ, 10x faster S3 for latency-sensitive workloads — trade-off: no cross-AZ durability. Interviewers will ask about when this trade-off is acceptable.
- **Graviton4 (r8g instances):** AWS's latest ARM-based processor for general-purpose workloads — Go binaries compile natively to ARM64 (`GOARCH=arm64`), making Graviton a natural Go deployment target with 20–40% better price/performance.

### AWS 2022–2023

- **Lambda Function URLs:** Direct HTTPS endpoints for Lambda without API Gateway — simplifies Go Lambda deployment for internal services where API Gateway features (auth, rate limiting) are not needed.
- **EventBridge Pipes:** Point-to-point integrations between event sources and targets without custom Lambda glue — reduces the Go Lambda code needed for event routing patterns.
- **AWS SDK for Go v2 (Stable):** Fully rewritten with context-first APIs, modular packages, and native `context.Context` propagation — all new Go services should use v2; v1 is maintenance-only.

> **Interview signal:** Referencing AWS SDK for Go v2's context-first design unprompted signals you understand both Go and AWS — not just one in isolation.

---

## L1 — Conceptual Foundations

> 📊 **L1:** 1 category · 2 topics · 2 questions

---

### Category: AWS Core Concepts

> 📁 2 topics · 2 questions

#### Topic: Regions, Availability Zones, and Edge Locations — (Q count: 1)

- **[L1.AWSCoreConcepts.RegionsAvailabilityZonesAndEdgeLocations.Q1]**
  **Question:** What is the difference between an AWS Region, an Availability Zone, and an Edge Location, and why does this hierarchy matter for system design?
- **Answer:**
  - A **Region** is a geographic cluster of data centres (e.g., `ap-south-1` = Mumbai) — each Region is isolated, data does not replicate across Regions unless explicitly configured.
  - An **Availability Zone (AZ)** is one or more physically separate data centres within a Region, connected by low-latency links — AZs have independent power, networking, and cooling; a failure in one AZ does not affect others.
  - **Edge Locations** (400+) are CloudFront CDN nodes distributed globally — they cache content and terminate TLS close to the user, reducing latency for static assets and API responses.
  - Designing for AZ failure (multi-AZ) is the baseline for production services; designing for Region failure (multi-Region) is for business-critical workloads with aggressive RTO/RPO targets.
  - A Go service deployed in a single AZ is a single point of failure — ALB + ECS/EKS spanning 3 AZs is the production baseline.
- **Example:** Persistent's enterprise client with healthcare data residency requirements deploys Go services exclusively in `ap-south-1` (Mumbai) — data sovereignty law prohibits cross-region replication, making multi-AZ within `ap-south-1` the resilience strategy.
- **Technical Terms to Include:** Region isolation, AZ independent failure domains, Edge Location, CloudFront, multi-AZ, multi-Region, RTO/RPO
- **Gotcha:** "Multi-AZ means high availability" — Multi-AZ protects against infrastructure failure, not application bugs. A Go service with a logic bug that panics on a specific input will fail across all AZs simultaneously.
- **Follow-Up:** "What is the difference between RTO and RPO?" → RTO (Recovery Time Objective): how long the system can be down before business impact. RPO (Recovery Point Objective): how much data loss is acceptable. Asked because these drive architectural choices, not the other way around.
- **Conclusion:** AWS's Region/AZ hierarchy is a risk distribution model — understanding it drives every resiliency decision in a production architecture.

---

#### Topic: IAM Fundamentals — (Q count: 1)

- **[L1.AWSCoreConcepts.IAMFundamentals.Q1]**
  **Question:** How does AWS IAM work, and what is the principle of least privilege in the context of a Go microservice running on Lambda?
- **Answer:**
  - IAM (Identity and Access Management) controls _who_ can do _what_ to _which_ AWS resource — identities (users, roles, services) are granted permissions via policies (JSON documents).
  - An IAM Role is a set of permissions that a service can assume — a Go Lambda function assumes an execution role; it does not have permanent credentials, only temporary session credentials via STS.
  - Least privilege: the execution role grants only the exact permissions the function needs — a Go Lambda that reads from DynamoDB and writes to SQS gets `dynamodb:GetItem` and `sqs:SendMessage` only, not `dynamodb:*`.
  - Resource-based policies (on S3 buckets, SQS queues) define who can access the resource from the outside — identity-based policies define what the identity can access.
  - IAM policy evaluation: explicit Deny > explicit Allow > implicit Deny (default). An explicit Deny in any attached policy overrides any Allow.
- **Example:** A Go Lambda at Persistent consuming from SQS was initially granted `sqs:*` for speed. Security review found it had `sqs:DeleteQueue` — a Go bug in error handling could have called `DeleteQueue` instead of `DeleteMessage`. Scope narrowed to `sqs:ReceiveMessage`, `sqs:DeleteMessage`, `sqs:GetQueueAttributes`.
- **Technical Terms to Include:** IAM Role, execution role, least privilege, STS temporary credentials, resource-based policy, identity-based policy, explicit Deny, `sqs:DeleteMessage`
- **Gotcha:** "IAM policy wildcards seem convenient" — `s3:*` on a production bucket is a misconfiguration waiting for a Go bug to exploit. Wildcards in production IAM policies are a security audit finding.
- **Follow-Up:** "What is IAM role assumption and how does cross-account access work?" → `sts:AssumeRole` allows a principal in one account to temporarily assume a role in another — trust policy on the target role allows the source account. Common in enterprise multi-account setups. Asked to test cross-account AWS architecture knowledge.
- **Conclusion:** IAM is the foundation of AWS security — misconfigured IAM is the root cause of the majority of cloud security breaches, and every engineer deploying Go services must own it.

---

## L2 — Core Services

> 📊 **L2:** 4 categories · 5 topics · 5 questions

---

### Category: Compute

> 📁 2 topics · 2 questions

#### Topic: AWS Lambda — Execution Model — (Q count: 1)

- **[L2.Compute.AWSLambda—ExecutionModel.Q1]**
  **Question:** Explain the Lambda execution model from invocation to billing, and what makes it the right or wrong choice for a Go service.
- **Answer:**
  - Lambda receives an event, instantiates a runtime (or reuses a warm execution environment), invokes the Go handler function, and returns the response — billing is per-millisecond of execution time + per-request count.
  - **Cold start:** when no warm execution environment is available, Lambda bootstraps the Go binary — Go cold starts are fast (~50–100ms for a small binary) vs Java (500ms+) because Go compiles to a self-contained binary.
  - **Warm execution environment reuse:** Lambda reuses the execution environment for subsequent invocations — initialisation code outside the handler (DB connections, config loading) runs only once per environment, not per invocation.
  - Lambda is stateless by contract — no local disk persistence between invocations; `/tmp` (512MB–10GB) is ephemeral scratch space only.
  - Not appropriate for: long-running tasks (>15-minute limit), persistent TCP connections, large memory workloads (10GB max), or latency-sensitive P99 paths where cold starts are unacceptable.
- **Example:** A Go Lambda processing Stripe webhooks at Persistent: handler runs ~80ms per event, costs ~$0.0002 per 1M invocations. Moved a reporting job that ran hourly for 14 minutes from EC2 to Lambda — saves the EC2 idle cost with no code change beyond packaging.
- **Technical Terms to Include:** cold start, warm execution environment, stateless, `/tmp` scratch, 15-minute limit, `aws-lambda-go` SDK, `lambda.Start(handler)`
- **Gotcha:** "Lambda concurrency scales automatically" — Lambda scales by spawning new execution environments, each with its own cold start. A burst of 1,000 concurrent events spawns 1,000 environments simultaneously — downstream DB connections spike to 1,000, potentially exhausting the RDS connection pool.
- **Follow-Up:** "How do you mitigate Lambda cold starts for a Go service?" → Provisioned Concurrency (keeps environments warm, costs money), SnapStart (Go preview), or minimise binary size and init work. Asked because cold start mitigation is a real prod decision.
- **Conclusion:** Lambda is the right choice for event-driven, short-lived Go workloads — its simplicity hides operational complexity that surfaces only at scale.

---

#### Topic: EC2 and Auto Scaling — (Q count: 1)

- **[L2.Compute.EC2AndAutoScaling.Q1]**
  **Question:** When would you run a Go service on EC2 with Auto Scaling instead of Lambda or EKS?
- **Answer:**
  - EC2 is appropriate when: execution duration exceeds Lambda's 15-minute limit, persistent in-process state is required (e.g., in-memory cache), the workload benefits from consistent baseline throughput (no cold starts), or the Go binary requires specialised hardware (GPU, custom networking).
  - Auto Scaling Groups (ASG) replace failed instances automatically and scale based on CloudWatch metrics (CPU, request count, custom metrics) — the target tracking policy is the simplest: "keep CPU at 50%."
  - Launch Templates define the Go service's AMI, instance type, IAM role, and user data (startup script) — treat Launch Templates as immutable; update by creating a new version.
  - Graviton3/4 (ARM64) instances provide 20–40% better price/performance for Go services — Go's native ARM64 cross-compilation (`GOOS=linux GOARCH=arm64`) is a first-class workflow.
  - EC2 adds operational overhead: OS patching, agent management, AMI baking — for most Go microservices, ECS Fargate or EKS is a better abstraction.
- **Example:** A Persistent Go service performing long-running video transcription (up to 4 hours) runs on EC2 Graviton3 Spot instances — Lambda's 15-minute limit eliminated it; ECS Fargate was considered but Spot instance cost savings (70%) made EC2 ASG the choice.
- **Technical Terms to Include:** Auto Scaling Group, Launch Template, Graviton3/4, ARM64, target tracking policy, Spot instance, AMI, user data
- **Gotcha:** "Auto Scaling handles everything" — ASG replaces instances on failure but cannot detect a Go service that is running but not serving traffic (stuck goroutines, deadlock). Health checks on the ALB target group detect application-level failure, not ASG.
- **Follow-Up:** "What is the difference between horizontal and vertical scaling?" → Horizontal: add more instances (ASG). Vertical: increase instance size (scale up). Go services are typically stateless, making horizontal scaling the default. Asked to calibrate basic architectural understanding.
- **Conclusion:** EC2 with Auto Scaling is the escape hatch from serverless and container constraints — choose it when Lambda or EKS cannot meet the workload's execution or hardware requirements.

---

### Category: Storage

> 📁 1 topic · 1 question

#### Topic: S3 Architecture and Go Integration — (Q count: 1)

- **[L2.Storage.S3ArchitectureAndGoIntegration.Q1]**
  **Question:** Explain S3's consistency model, storage classes, and how you integrate S3 into a Go service correctly using AWS SDK v2.
- **Answer:**
  - S3 provides **strong read-after-write consistency** for all operations since December 2020 — a `PutObject` followed immediately by `GetObject` returns the new version; this eliminates the consistency bugs that plagued pre-2020 architectures.
  - Storage classes: S3 Standard (hot), S3 Intelligent-Tiering (auto-tiering), S3 Standard-IA (infrequent access), S3 Glacier Instant/Flexible/Deep Archive (archival) — class selection is cost vs retrieval latency.
  - S3 is not a filesystem: key-based access, no rename operation (copy + delete), no atomic multi-key transactions — Go services treating S3 like a filesystem encounter these limitations under concurrent access.
  - AWS SDK for Go v2 uses context-first APIs: `s3Client.GetObject(ctx, input)` — the context carries cancellation and deadline, correctly propagated from the HTTP request context in Go handlers.
  - Multipart upload for objects >100MB: `CreateMultipartUpload`, upload parts in parallel goroutines, `CompleteMultipartUpload` — the Go `s3manager` (from `github.com/aws/aws-sdk-go-v2/feature/s3/manager`) handles this automatically.
- **Example:**

```go
client := s3.NewFromConfig(cfg)
result, err := client.GetObject(ctx, &s3.GetObjectInput{
    Bucket: aws.String("my-bucket"),
    Key:    aws.String("offers/2024/q1.json"),
})
if err != nil { return fmt.Errorf("s3 get: %w", err) }
defer result.Body.Close()
```

- **Technical Terms to Include:** strong read-after-write consistency, storage class, S3 Intelligent-Tiering, multipart upload, `s3manager`, presigned URL, S3 lifecycle policy, SDK v2 context-first
- **Gotcha:** "S3 lists are eventually consistent" — wait, S3 lists (ListObjectsV2) are now strongly consistent too since 2020. But list pagination with continuation tokens is stateful — storing the token and resuming in a new Lambda invocation requires careful handling.
- **Follow-Up:** "How do you generate a presigned URL for temporary S3 object access?" → `s3.NewPresignClient(client).PresignGetObject(ctx, input, func(o *s3.PresignOptions){ o.Expires = 15*time.Minute })` — never embed permanent AWS credentials in client-side code. Asked to test security-first thinking.
- **Conclusion:** S3 is the most reliable storage primitive on AWS — its consistency model, SDK integration, and storage tier flexibility make it the foundation of most data workflows in Go services.

---

### Category: Messaging

> 📁 1 topic · 1 question

#### Topic: SQS vs SNS — Architecture Patterns — (Q count: 1)

- **[L2.Messaging.SQSVsSNS—ArchitecturePatterns.Q1]**
  **Question:** When do you use SQS vs SNS, and how does the fan-out pattern combine them?
- **Answer:**
  - **SQS** is a point-to-point queue: one producer, one consumer group — messages are delivered at least once, processed by exactly one consumer (standard queue) or with ordering guarantees (FIFO queue).
  - **SNS** is a pub-sub topic: one producer, multiple subscriber types (SQS, Lambda, HTTP endpoint, email) — a single `Publish` call delivers to all subscribers concurrently.
  - **Fan-out pattern:** one SNS topic → multiple SQS queues. An order event published to SNS simultaneously delivers to: billing SQS queue, inventory SQS queue, notification SQS queue — each consumer processes independently at its own pace.
  - SQS visibility timeout: when a Go consumer reads a message, it becomes invisible for the timeout duration — if processing fails or times out, the message reappears for another consumer (dead letter queue after N failures).
  - FIFO queues guarantee ordering and exactly-once delivery at the cost of lower throughput (3,000 msg/s vs standard's nearly unlimited) — use FIFO only when ordering is a business requirement.
- **Example:** At the PayPal BFF level, a subscription lifecycle event published to SNS was consumed by: billing service SQS (to update invoice), notification service SQS (to send email), and analytics SQS (to record event) — three independent consumers, zero coupling.
- **Technical Terms to Include:** at-least-once delivery, FIFO queue, visibility timeout, dead letter queue (DLQ), fan-out pattern, SNS topic, SQS consumer group, message retention
- **Gotcha:** "SQS standard queues guarantee exactly-once delivery" — they do not. Standard SQS guarantees at-least-once; idempotent Go handlers (using a deduplication key or DynamoDB idempotency table) are required to handle duplicates safely.
- **Follow-Up:** "How do you implement a Go Lambda consumer of SQS with error handling?" → Lambda's SQS event source mapping batches messages; return an error to nack the entire batch, or use `SQSBatchResponse` to nack individual failures. Asked because batch partial failure handling is a real operational gap.
- **Conclusion:** SQS is for reliable point-to-point work distribution; SNS is for event broadcast — combining them in fan-out is the foundation of decoupled event-driven Go architectures.

---

### Category: Database

> 📁 1 topic · 1 question

#### Topic: DynamoDB Data Modelling — (Q count: 1)

- **[L2.Database.DynamoDBDataModelling.Q1]**
  **Question:** Explain DynamoDB's primary key model, and how does access pattern-first design differ from relational modelling?
- **Answer:**
  - DynamoDB tables use a **partition key** (required) and optional **sort key** — together they form the primary key. All queries must include the partition key; the sort key enables range queries within a partition.
  - DynamoDB is schemaless except for the primary key — each item can have different attributes, but the partition key must be present on every item.
  - **Access pattern-first:** model the table structure around your query patterns, not entity relationships — `GetUserOrders(userID)` drives the partition key choice, not normalisation theory.
  - **Single-table design:** multiple entity types (User, Order, Product) stored in one table using composite keys (`PK: USER#123`, `SK: ORDER#2024-01`) — enables joining with `Query` on the partition key, eliminating cross-table joins.
  - Global Secondary Indexes (GSI) invert the key structure for alternate access patterns — `GSI: email → userID` supports `GetUserByEmail` without scanning the entire table.
- **Example:** At the Apple Offers platform, offer eligibility data was modelled with `PK: DEVICE#serialNumber`, `SK: OFFER#offerID` — `GetOffersForDevice(serialNumber)` is a single Query call returning all offers for a device, O(1) with no table scan.
- **Technical Terms to Include:** partition key, sort key, primary key, schemaless, access pattern-first, single-table design, GSI, composite key, `Query` vs `Scan`
- **Gotcha:** "`Scan` is fine for dev" — Scan reads every item in the table, consuming read capacity proportional to table size — a Scan against a 100GB DynamoDB table will exhaust provisioned capacity and cause throttling in production. Never use Scan in a production read path.
- **Follow-Up:** "What is DynamoDB's consistency model and how does it affect a Go service?" → Default: eventually consistent reads (cheaper, slightly stale). `ConsistentRead: true` = strongly consistent (2x read cost). Use strongly consistent reads only when stale data causes correctness issues. Asked because consistency choice is an architecture decision, not a default.
- **Conclusion:** DynamoDB rewards access pattern-first thinking — every table design decision flows from the queries the Go service will execute, not from the entity model.

---

## L3 — Common Patterns

> 📊 **L3:** 2 categories · 2 topics · 2 questions

---

### Category: Serverless Architecture

> 📁 1 topic · 1 question

#### Topic: Serverless Go Architecture on AWS — (Q count: 1)

- **[L3.ServerlessArchitecture.ServerlessGoArchitectureOnAWS.Q1]**
  **Question:** Design a serverless Go API on AWS. What are the components, the failure modes, and the operational trade-offs vs a containerised service?
- **Answer:**
  - Components: API Gateway (request routing, auth via Cognito JWT authorizer) → Lambda (Go handler) → DynamoDB (state) + SQS (async work) + S3 (object storage). CloudWatch for logs/metrics; X-Ray for distributed tracing.
  - Go Lambda packaging: `GOOS=linux GOARCH=arm64 go build -o bootstrap main.go` → zip → Lambda function with `provided.al2023` runtime — Graviton (ARM64) is 20% cheaper than x86 for Go workloads.
  - Failure modes: Lambda cold starts at traffic spikes (mitigate: Provisioned Concurrency or SnapStart); DynamoDB throttling at hot partitions (mitigate: exponential backoff in Go SDK); API Gateway 29-second timeout (a Go handler exceeding 29s is killed by API Gateway before Lambda timeout).
  - Operational trade-offs vs containers: No SSH/exec access for debugging — `pprof` must be embedded and exposed; log-based debugging is the norm. No long-lived connections — each Lambda invocation may use a new DB connection (use RDS Proxy for relational DBs).
  - Cost model: Lambda charges per invocation + duration — for consistent high-throughput traffic (10K+ req/s), ECS Fargate or EKS becomes cheaper than Lambda.
- **Example:**

```go
func main() {
    cfg, _ := config.LoadDefaultConfig(context.Background())
    db := dynamodb.NewFromConfig(cfg)
    h := &Handler{db: db}
    lambda.Start(h.Handle)
}
func (h *Handler) Handle(ctx context.Context, req events.APIGatewayProxyRequest)
    (events.APIGatewayProxyResponse, error) {
    // context carries X-Ray trace, deadline from API Gateway 29s timeout
}
```

- **Technical Terms to Include:** `provided.al2023`, Graviton ARM64, Provisioned Concurrency, API Gateway 29s timeout, X-Ray, RDS Proxy, cold start, `aws-lambda-go`
- **Gotcha:** "Lambda scales infinitely" — Lambda scales concurrency up to the account-level limit (default 1,000 concurrent executions per region) — a sudden traffic spike can hit this limit and return 429s while Lambda requests scaling limit increases asynchronously.
- **Follow-Up:** "How do you test a Go Lambda handler locally?" → AWS SAM CLI (`sam local invoke`) or `lambda-local` package — abstract the Lambda event types and test the handler function directly with table-driven tests. Asked because local development workflow for Lambda is non-obvious.
- **Conclusion:** Serverless Go on AWS eliminates infrastructure management but requires discipline around context propagation, connection lifecycle, and observability — the operational model shifts from "fix the server" to "read the logs."

---

### Category: Event-Driven Architecture

> 📁 1 topic · 1 question

#### Topic: Event-Driven Go Services with EventBridge — (Q count: 1)

- **[L3.EventDrivenArchitecture.EventDrivenGoServicesWithEventBridge.Q1]**
  **Question:** How would you design an event-driven Go service using EventBridge, and what distinguishes EventBridge from SQS/SNS for this use case?
- **Answer:**
  - EventBridge is a serverless event bus that routes events based on content rules — a rule pattern like `{"source": ["myapp.orders"], "detail-type": ["OrderPlaced"]}` routes to a specific Lambda or SQS target without custom routing code.
  - vs SNS: SNS fans out to all subscribers; EventBridge filters by event content before routing — reduces consumer coupling and eliminates the need for consumers to filter irrelevant events.
  - vs SQS: SQS is for work queues with at-least-once delivery; EventBridge is for event routing — EventBridge delivers to Lambda without a polling consumer, reducing latency.
  - EventBridge Pipes (2022) connect event sources (DynamoDB Streams, SQS, Kinesis) to targets with optional filtering and enrichment — replaces Go Lambda glue code for simple routing.
  - A Go service publishes to EventBridge via `PutEvents` API: custom `source` and `detail-type` fields define the schema; the detail body is JSON.
- **Example:**

```go
entry := types.PutEventsRequestEntry{
    Source:       aws.String("persistent.offers"),
    DetailType:   aws.String("OfferRedeemed"),
    Detail:       aws.String(`{"offerID":"o123","userID":"u456"}`),
    EventBusName: aws.String("default"),
}
_, err := ebClient.PutEvents(ctx, &eventbridge.PutEventsInput{
    Entries: []types.PutEventsRequestEntry{entry},
})
```

- **Technical Terms to Include:** event bus, event rule, event pattern, `PutEvents`, `source`, `detail-type`, EventBridge Pipes, content-based routing, schema registry
- **Gotcha:** "EventBridge guarantees exactly-once delivery" — EventBridge guarantees at-least-once delivery; duplicate events are possible, especially during retry. Go consumers must be idempotent.
- **Follow-Up:** "How do you version EventBridge event schemas without breaking consumers?" → EventBridge Schema Registry with schema versioning; additive changes (new fields) are backward compatible; removing fields is a breaking change requiring a new `detail-type`. Asked to test schema evolution discipline.
- **Conclusion:** EventBridge decouples producers and consumers at the routing layer — the event content, not the producer, determines which Go service handles it.

---

## L4 — Advanced Patterns

> 📊 **L4:** 2 categories · 2 topics · 2 questions

---

### Category: High Availability

> 📁 1 topic · 1 question

#### Topic: Multi-AZ Go Service Architecture — (Q count: 1)

- **[L4.HighAvailability.MultiAZGoServiceArchitecture.Q1]**
  **Question:** Design a multi-AZ Go service deployment on AWS that survives a full AZ failure with zero downtime, and specify the exact AWS components required.
- **Answer:**
  - **Compute:** ECS Fargate or EKS with tasks/pods spread across 3 AZs using placement constraints (`AZ spread` in ECS, `topologySpreadConstraints` in Kubernetes) — minimum 1 task per AZ.
  - **Load balancing:** Application Load Balancer (ALB) with cross-zone load balancing enabled — if `ap-south-1a` fails, the ALB routes all traffic to tasks in `1b` and `1c` automatically.
  - **Database:** RDS Multi-AZ (synchronous replication, automatic failover in ~30s) or DynamoDB (multi-AZ by default, transparent to the Go service) — connection string must use the RDS endpoint DNS (not IP) for failover to work.
  - **Caching:** ElastiCache Redis cluster mode with replicas across AZs — a primary node failure promotes a replica; Go clients using the cluster endpoint automatically redirect.
  - **State-free Go service:** the Go application must be stateless — session state in ElastiCache, not in-process. Any goroutine-local state lost on task replacement is a design flaw.
- **Example:** Persistent's Go API serving a healthcare client maintains 99.99% uptime SLA. Architecture: ALB (cross-zone) → ECS Fargate (3 tasks, 1 per AZ) → RDS PostgreSQL Multi-AZ → DynamoDB for session. An `ap-south-1a` failure: 1 task lost, ALB routes to 2 remaining tasks within 30s, RDS failover completes in 30s — cumulative downtime ~30s per year budget.
- **Technical Terms to Include:** AZ spread, cross-zone load balancing, ALB, RDS Multi-AZ failover, DynamoDB multi-AZ, ElastiCache cluster mode, stateless application, ECS placement constraints
- **Gotcha:** "Cross-zone load balancing costs money" — inter-AZ data transfer in cross-zone load balancing incurs data transfer charges. For high-throughput services, this is non-trivial — evaluate against the HA benefit. ALB cross-zone is free; NLB cross-zone has a charge.
- **Follow-Up:** "What is the difference between an ALB and an NLB?" → ALB: Layer 7, HTTP/HTTPS routing, header manipulation, target group health checks based on HTTP response. NLB: Layer 4, TCP/UDP, ultra-low latency, static IP — for Go services that need TCP passthrough or require a static IP for external whitelisting. Asked at architect level.
- **Conclusion:** Multi-AZ Go service survival requires every layer to be AZ-aware — compute placement, load balancer configuration, database replication, and cache replication must all be explicit.

---

### Category: Security

> 📁 1 topic · 1 question

#### Topic: AWS Secrets Manager and Parameter Store — (Q count: 1)

- **[L4.Security.AWSSecretsManagerAndParameterStore.Q1]**
  **Question:** How do you manage secrets for a Go service on AWS, and what is the operational difference between Secrets Manager and Parameter Store?
- **Answer:**
  - **Secrets Manager:** managed rotation (automatic credential rotation for RDS, Redshift, DocumentDB), versioned secrets, cross-account sharing — costs $0.40/secret/month + API call charges.
  - **Parameter Store:** hierarchical key-value store; `SecureString` type uses KMS encryption; no automatic rotation — Standard tier is free for up to 10,000 parameters; Advanced tier (larger values, higher throughput) has a small charge.
  - In a Go Lambda/ECS service: load secrets at **initialisation** (outside the handler), not per request — `secretsmanager.GetSecretValue` is a network call; calling it per-invocation adds latency and cost.
  - Cache the secret value in a module-level variable; implement a background goroutine that re-fetches periodically or on HTTP 403 (secret rotated) — Lambda extensions can also handle rotation notification.
  - Never log secret values, never embed in environment variables plaintext for sensitive credentials — use IAM role permissions to allow the Lambda execution role to call `secretsmanager:GetSecretValue`.
- **Example:**

```go
var dbPassword string
var once sync.Once

func getDBPassword(ctx context.Context, client *secretsmanager.Client) string {
    once.Do(func() {
        result, err := client.GetSecretValue(ctx, &secretsmanager.GetSecretValueInput{
            SecretId: aws.String("prod/myapp/db-password"),
        })
        if err != nil { log.Fatal("cannot load secret:", err) }
        dbPassword = aws.ToString(result.SecretString)
    })
    return dbPassword
}
```

- **Technical Terms to Include:** Secrets Manager rotation, Parameter Store SecureString, KMS, `secretsmanager:GetSecretValue`, `sync.Once` caching, IAM execution role, secret versioning
- **Gotcha:** "Environment variables are fine for secrets" — environment variables in Lambda are encrypted at rest with KMS but visible in plaintext in the Lambda console to anyone with `lambda:GetFunctionConfiguration`. Secrets Manager provides access control at the secret level.
- **Follow-Up:** "How do you handle secret rotation for a Go service without downtime?" → Secrets Manager supports two versions (AWSPENDING, AWSCURRENT) during rotation. Go service must catch authentication failures, re-fetch the secret, and retry — not assume the cached value is always valid. Asked because rotation without downtime requires explicit Go code design.
- **Conclusion:** Secret management in Go services is an operational discipline — Secrets Manager for sensitive credentials with rotation, Parameter Store for configuration — both require caching to be production-appropriate.

---

## L5 — Architecture and System Design

> 📊 **L5:** 1 category · 1 topic · 1 question

---

### Category: Microservices on AWS

> 📁 1 topic · 1 question

#### Topic: Go Microservices on EKS — (Q count: 1)

- **[L5.MicroservicesOnAWS.GoMicroservicesOnEKS.Q1]**
  **Question:** You are deploying a fleet of Go microservices to EKS. Walk me through the architectural decisions from container image to production observability.
- **Answer:**
  - **Container image:** multi-stage Dockerfile — build stage (`golang:1.24-alpine`) compiles the binary; runtime stage (`gcr.io/distroless/static`) ships only the binary with no shell or OS tools — results in ~5MB image with minimised attack surface.
  - **Kubernetes objects:** `Deployment` with rolling update strategy (maxUnavailable=1, maxSurge=1), `HorizontalPodAutoscaler` on CPU or custom KEDA metrics, `PodDisruptionBudget` (minAvailable=1) for safe node draining.
  - **Service discovery:** Kubernetes `Service` of type `ClusterIP` for internal service-to-service communication; `Ingress` (AWS ALB Ingress Controller) for external traffic. CoreDNS resolves `service-name.namespace.svc.cluster.local`.
  - **Observability:** Go binary exposes `/metrics` (Prometheus format via `promhttp`); Prometheus scrapes via ServiceMonitor; Grafana dashboards. `log/slog` structured JSON logs shipped to CloudWatch Logs via Fluent Bit DaemonSet. X-Ray or OpenTelemetry Collector sidecar for distributed tracing.
  - **IRSA (IAM Roles for Service Accounts):** Go pods assume IAM roles via OIDC — no long-lived AWS credentials; pod's service account is annotated with the IAM role ARN; AWS SDK auto-discovers credentials via the token file.
- **Example:**

```dockerfile
FROM golang:1.24-alpine AS build
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build -o /service ./cmd/server

FROM gcr.io/distroless/static-debian12
COPY --from=build /service /service
ENTRYPOINT ["/service"]
```

- **Technical Terms to Include:** distroless image, multi-stage build, IRSA, HorizontalPodAutoscaler, PodDisruptionBudget, KEDA, ALB Ingress Controller, ServiceMonitor, Fluent Bit, OIDC, `CGO_ENABLED=0`
- **Gotcha:** "IRSA requires the pod to use the correct service account" — a Go pod without the `eks.amazonaws.com/role-arn` annotation on its service account falls back to the node's instance profile, which often has broader permissions. IRSA must be explicitly configured per deployment.
- **Follow-Up:** "How would you implement zero-downtime deployments for a Go service on EKS?" → Rolling update with `preStop` hook (sleep 5s) to allow load balancer deregistration; `terminationGracePeriodSeconds` must exceed the Go service's graceful shutdown timeout. Asked because Kubernetes rolling updates cause brief 5xx without these settings.
- **Conclusion:** Go on EKS is the production standard for stateless microservice fleets — distroless images, IRSA, and structured observability are the three non-negotiable foundation pieces.

---

## L6 — Expert Level

> 📊 **L6:** 2 categories · 2 topics · 2 questions

---

### Category: Cost and Performance at Scale

> 📁 1 topic · 1 question

#### Topic: AWS Cost Architecture for Go Services — (Q count: 1)

- **[L6.CostAndPerformanceAtScale.AWSCostArchitectureForGoServices.Q1]**
  **Question:** A Persistent client's Go service AWS bill has grown 3x in 6 months. Walk me through your FinOps investigation methodology and the architectural levers you pull.
- **Answer:**
  - **Identify the driver:** AWS Cost Explorer → group by service, then by tag (service name, environment) → identify which service and which specific resource (EC2 instance type, data transfer, Lambda invocations) is growing.
  - **Data transfer:** inter-AZ data transfer is the most common hidden cost driver — a Go service fetching S3 objects in `us-east-1` from an EC2 in `us-east-1` in a different AZ incurs transfer charges. Use S3 endpoints, same-AZ routing, and caching (CloudFront, ElastiCache) to reduce.
  - **Compute right-sizing:** Go services often run on over-provisioned EC2 instances — CloudWatch memory metrics (via CloudWatch Agent) + CPU utilisation → Compute Optimizer recommendations → move to Graviton instances at 20% cost reduction.
  - **Lambda optimisation:** reduce execution duration (profile the Go handler, reduce init work) and memory (Lambda pricing = GB-seconds; over-provisioned memory is wasted cost). `lambda-power-tuning` tool finds the optimal memory/cost/performance trade-off.
  - **Reserved Instances / Savings Plans:** for predictable Go service baseline load, 1-year Compute Savings Plans provide 30–40% discount vs On-Demand — use after right-sizing, not before.
- **Example:** A Persistent Go API service had 40% of its bill from DynamoDB read costs. Investigation: `Scan` operations in a reporting Lambda were reading the entire table hourly. Fix: DynamoDB Streams → Lambda → S3 aggregated report — DynamoDB read cost dropped 90%, report latency improved.
- **Technical Terms to Include:** Cost Explorer, inter-AZ data transfer, S3 VPC endpoint, Graviton, Compute Optimizer, lambda-power-tuning, GB-seconds, Savings Plans, DynamoDB Scan cost
- **Gotcha:** "Spot instances solve all compute cost problems" — Spot instances can be interrupted with 2-minute notice. Go services must implement graceful shutdown on `SIGTERM` (already required for Kubernetes, applies here too) and be stateless. Stateful Go services on Spot lose state on interruption.
- **Follow-Up:** "What is an S3 VPC endpoint and why does it reduce cost?" → A gateway VPC endpoint routes S3 traffic through the AWS backbone instead of the internet — eliminates NAT Gateway data processing charges ($0.045/GB) for S3 traffic from within the VPC. For a Go service processing large S3 objects, this is significant. Asked because it is a real cost lever most engineers miss.
- **Conclusion:** AWS cost optimisation at scale requires application architecture changes — FinOps is not just reserved instances, it is redesigning Go services to not pay for what they don't need.

---

### Category: Well-Architected Framework

> 📁 1 topic · 1 question

#### Topic: Applying the AWS Well-Architected Framework to Go Services — (Q count: 1)

- **[L6.WellArchitectedFramework.ApplyingTheAWSWellArchitectedFrameworkToGoServices.Q1]**
  **Question:** Walk me through how the five pillars of the Well-Architected Framework translate into concrete decisions for a production Go service.
- **Answer:**
  - **Operational Excellence:** IaC (Terraform/CDK) for all AWS resources — no manual console changes; runbooks for Go service incidents; `go build` + Docker build + `terraform apply` in CI/CD pipeline; CloudWatch alarms on Go service error rate and P99 latency.
  - **Security:** IRSA for zero-credential Go deployments; Secrets Manager for DB credentials; `govulncheck` in CI; WAF on ALB for the public API; VPC with private subnets for Go services (no direct internet access from the Go runtime).
  - **Reliability:** Multi-AZ deployment; RDS Multi-AZ or DynamoDB; circuit breaker in Go service (avoid cascading failures to downstream); SQS DLQ for failed event processing; `context.WithTimeout` on all external calls.
  - **Performance Efficiency:** Go Graviton instances; DynamoDB for high-throughput state; ElastiCache for repeated computations; Lambda for event-driven spiky workloads; `pprof` before and after optimisation changes.
  - **Cost Optimisation:** Savings Plans for baseline compute; right-sizing via Compute Optimizer; DynamoDB on-demand for unpredictable workloads, provisioned for stable; S3 lifecycle policies for object archival; eliminate inter-AZ data transfer.
- **Example:** A Well-Architected Review at Persistent flagged a Go service with: no circuit breaker (Reliability gap), DB credentials in environment variables (Security gap), no structured logging (Operational Excellence gap). 3-sprint remediation: `go-resiliency` circuit breaker, Secrets Manager migration, `log/slog` adoption.
- **Technical Terms to Include:** Well-Architected pillars, IRSA, WAF, circuit breaker, SQS DLQ, Savings Plans, Compute Optimizer, IaC, CloudWatch alarms
- **Gotcha:** "Well-Architected is a checklist" — it is a framework for trade-off decisions, not a pass/fail audit. A Go service without multi-region may be fully correct if the business RTO is 4 hours — the trade-off is documented and accepted, not ignored.
- **Follow-Up:** "What is a Well-Architected Review and who runs it?" → AWS provides a Well-Architected Tool in the console; APN Partners (like Persistent) run reviews for clients. The output is a risk register with High/Medium risk findings and recommended remediation paths. Asked because Persistent delivers these reviews as a service.
- **Conclusion:** The Well-Architected Framework is the architectural vocabulary for senior engineers — every production Go service decision maps to one of the five pillars, whether you name it explicitly or not.

---

## 📊 File Summary

| File                          | Classification  | Depth | Version       | Categories | Topics | Questions |
| ----------------------------- | --------------- | ----- | ------------- | ---------- | ------ | --------- |
| [Golang](./golang.md)         | Must Have       | L1–L6 | Go v1.24      | 18         | 23     | 23        |
| [AWS](./aws.md)               | Must Have       | L1–L6 | AWS SDK Go v2 | 12         | 14     | 14        |
| [Golang+AWS](./golang_aws.md) | Combination     | L1–L6 | N/A           | 5          | 6      | 6         |
| [PrepSummary](./README.md)    | Session Summary | —     | —             | —          | —      | —         |
| **TOTAL**                     |                 |       |               | **35**     | **43** | **43**    |

---
