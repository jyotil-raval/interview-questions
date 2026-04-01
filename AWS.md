<!-- Generated: 2026-03-29 | AWS Services: Lambda · S3 · CloudFront · API Gateway · DynamoDB · Cognito · SQS · SNS -->

# AWS Interview Preparation — Tech Lead / Architect Level

> Lambda · Lambda@Edge · S3 · CloudFront · API Gateway · DynamoDB · Cognito · SQS · SNS
> Generated: 2026-03-29 | Career OS

---

## Scope & Calibration

This file covers the eight AWS services explicitly on your resume. Every question is
calibrated for the depth an interviewer expects from someone who has shipped production
systems using these services — not someone who read the docs.

Architect-level AWS interviews test three things:

1. **Mechanism** — how does it actually work under the hood
2. **Trade-offs** — when does this service / config / pattern hurt you
3. **Decision logic** — use X when Y, avoid X when Z

Generic "what is Lambda" answers fail at this level. Specific failure mode and
trade-off answers pass.

---

## Resume Claim Anchor

**IIoT Platform:** "built serverless backend services using AWS Lambda and DynamoDB"
**E-Commerce Platform:** "Managed Jenkins CI/CD pipelines and implemented Splunk dashboards"
**Your skills section:** Lambda, Lambda@Edge, API Gateway, S3, CloudFront, Cognito, SQS, SNS

Everything in this file is anchored to those claims. If an interviewer asks "walk me
through your AWS experience" — these are the architectural decisions and failure modes
you should reference from your production work.

---

## L1 — AWS Mental Model

---

**L1 Topics identified:**

- Shared Responsibility Model
- IAM — roles, policies, least privilege
- Regions, AZs, Edge Locations — the physical model
- The Serverless vs Managed vs Self-Managed spectrum

---

### IAM — Roles, Policies, Least Privilege

- **Question:** How does AWS IAM work, what is the difference between a role and a policy, and what does least privilege mean in practice for a serverless architecture?
- **Answer:**
  - **IAM Policy:** A JSON document defining permissions — what actions are allowed or denied on which resources. Two types: **identity-based** (attached to users/roles/groups) and **resource-based** (attached to the resource itself — S3 bucket policy, Lambda resource policy).
  - **IAM Role:** A trusted identity that can be assumed by AWS services, users, or external principals. A Lambda function doesn't have credentials — it assumes an IAM role at runtime. The role's attached policies determine what the Lambda can do. No permanent credentials — temporary credentials via STS (Security Token Service) are issued per invocation.
  - **Least privilege in practice:** Each Lambda function gets its own execution role. That role has only the permissions needed for that function's specific operations. A Lambda that reads from DynamoDB gets `dynamodb:GetItem` and `dynamodb:Query` on that specific table ARN — not `dynamodb:*` on `*`. This limits blast radius: if a function is compromised, the attacker can only access what that function's role permits.
  - **Evaluation order:** Explicit Deny always wins. Then explicit Allow. Then implicit Deny (default). Permissions boundaries, SCPs (Service Control Policies in AWS Organizations), and session policies can further restrict effective permissions — even if a role has an Allow, a more restrictive boundary can block it.
- **Example:**

```json
// Lambda execution role — minimal permissions for DynamoDB read + CloudWatch logs
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"],
      "Resource": "arn:aws:logs:us-east-1:123456789012:log-group:/aws/lambda/my-function:*"
    },
    {
      "Effect": "Allow",
      "Action": ["dynamodb:GetItem", "dynamodb:Query"],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/Orders"
      // NOT dynamodb:* on * — scoped to one table, two actions
    }
  ]
}

// Trust policy — who can assume this role
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "lambda.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
```

- **Technical Terms to Include:** IAM policy, execution role, trust policy, STS, temporary credentials, least privilege, explicit deny, resource ARN, S3 bucket policy, Lambda resource policy, permissions boundary, SCP
- **Gotcha:** `"Resource": "*"` in a Lambda execution role is a critical over-permission that most tutorials use for convenience. In production, always scope to specific ARNs. Wildcard resources mean a compromised function can access all DynamoDB tables, all S3 buckets — across the entire AWS account. Use IAM Access Analyzer to audit and tighten policies automatically.
- **Follow-Up:** "What is a resource-based policy and how does it interact with identity-based policies?" → A resource-based policy is attached to the resource (S3 bucket, Lambda function, SQS queue) rather than the caller. For cross-account access: the calling account's IAM role must Allow the action, AND the resource's policy must Allow the caller. Both must Allow — one isn't sufficient. For same-account access: either an identity-based Allow OR a resource-based Allow is sufficient. → They're testing cross-account permission model depth.
- **Conclusion:** IAM is the foundation of every AWS security posture — in a serverless architecture, function-level execution roles with ARN-scoped permissions are the unit of least privilege, and the absence of permanent credentials (temporary STS tokens per invocation) is what makes serverless inherently more secure than EC2-based deployments.

---

## L2 — Lambda: The Engine of Your Serverless Stack

---

**L2 Topics identified:**

- Execution model — cold start, warm start, execution environment lifecycle
- Concurrency — reserved, provisioned, burst limits
- Memory, timeout, ephemeral storage
- Triggers — event sources, async vs sync invocation
- Lambda@Edge vs CloudFront Functions
- Destinations — success/failure routing
- Lambda Layers
- SnapStart (Java, now available for Node/Python)

---

### Cold Start, Warm Start, Execution Environment

- **Question:** What is a Lambda cold start, what causes it, what does the execution environment lifecycle look like, and how do you mitigate cold starts in production?
- **Answer:**
  - A **cold start** occurs when Lambda must initialise a new execution environment to handle an invocation. The phases: (1) **Download code** — pull the deployment package or container image from S3/ECR. (2) **Start runtime** — initialise the language runtime (Node.js, Python, Java JVM). (3) **Run init code** — execute code outside the handler (imports, DB connection setup, SDK clients). Only then does the handler run. Cold start latency ranges from ~100ms (Node.js, Python) to several seconds (Java with heavy classpath).
  - A **warm start** reuses an existing execution environment — the code is already loaded, init code already ran, connections are established. The handler executes immediately. Lambda keeps environments warm for minutes to hours after the last invocation — no guarantee.
  - **Execution environment lifecycle:** Init → Invoke (repeated) → Shutdown. The `/tmp` directory (512MB default, up to 10GB) persists across invocations within the same environment. DB connections opened in init code persist across warm invocations — this is the key optimisation: initialise expensive resources outside the handler.
  - **Cold start mitigations:** (1) **Provisioned Concurrency** — pre-warms N execution environments, always ready. Eliminates cold starts for that N. Costs money even when idle. (2) **SnapStart** (Node.js, Python in 2025+, Java since 2023) — takes a snapshot of the initialised environment and restores from it — dramatically faster than a full init. (3) **Reduce package size** — smaller = faster download. (4) **Minimise init code** — lazy-load dependencies. (5) **Choose lighter runtimes** — Node.js/Python cold start < 200ms vs Java cold start 1–5s.
- **Example:**

```javascript
// WRONG — DB connection inside handler (new connection every cold AND warm start)
export const handler = async (event) => {
  const client = new DynamoDBClient({}); // initialised every invocation
  return client.send(new GetItemCommand({ TableName: 'Orders', Key: { id: event.id } }));
};

// CORRECT — DB connection outside handler (initialised once per execution environment)
import { DynamoDBClient, GetItemCommand } from '@aws-sdk/client-dynamodb';
const client = new DynamoDBClient({ region: 'us-east-1' }); // init code — runs once

export const handler = async (event) => {
  // client is reused across warm invocations — no connection overhead
  return client.send(new GetItemCommand({ TableName: 'Orders', Key: { id: { S: event.id } } }));
};

// Provisioned Concurrency — CloudFormation
MyFunction:
  Type: AWS::Lambda::Function
  Properties:
    FunctionName: api-handler
    MemorySize: 512

MyFunctionProvisionedConcurrency:
  Type: AWS::Lambda::ProvisionedConcurrencyConfig
  Properties:
    FunctionName: !Ref MyFunction
    Qualifier: !GetAtt MyFunctionAlias.FunctionArn
    ProvisionedConcurrentExecutions: 10  # 10 always-warm environments
```

- **Technical Terms to Include:** cold start, warm start, execution environment, init code, handler, `/tmp` ephemeral storage, Provisioned Concurrency, SnapStart, deployment package, container image, runtime, burst limit, connection reuse
- **Gotcha:** `/tmp` storage is shared across invocations within the **same execution environment** — not across different environments. If your function writes a cache file to `/tmp`, a subsequent warm invocation on the same environment reuses it. But a concurrent invocation running in a different environment has its own `/tmp`. Never treat `/tmp` as shared storage between concurrent invocations — use it only for per-environment caching.
- **Follow-Up:** "What is SnapStart and how does it differ from Provisioned Concurrency?" → Provisioned Concurrency keeps execution environments permanently warm — cost paid continuously. SnapStart takes a snapshot of the initialised state (after init code runs) and restores from it on cold start — the restore from snapshot is faster than full init, but environments are not pre-warmed (cold starts still occur, just shorter). SnapStart is cheaper than Provisioned Concurrency; Provisioned Concurrency eliminates cold starts entirely. Use SnapStart to reduce cold start duration; use Provisioned Concurrency to eliminate cold starts. → They're testing mitigation strategy trade-off depth.
- **Conclusion:** Cold starts are a Lambda-specific latency tax that appears only under certain conditions (new deployments, traffic spikes, infrequent invocations) — the mitigation strategy (Provisioned Concurrency vs SnapStart vs runtime choice vs init code optimisation) depends on the latency budget and cost constraints of the specific use case.

---

### Concurrency — Reserved, Unreserved, Burst

- **Question:** How does Lambda concurrency work, what are reserved and provisioned concurrency, and what happens at the account concurrency limit?
- **Answer:**
  - **Concurrency** = number of function instances handling requests simultaneously. Each invocation that arrives while another is still executing requires a new execution environment — Lambda scales out horizontally. Default account-level concurrency limit: **1,000 concurrent executions** per region (soft limit, raisable).
  - **Unreserved concurrency:** The pool of concurrency shared among all functions that don't have a reserved allocation. If one function spikes to 900 concurrent, other functions have only 100 left — they throttle. Throttled invocations: synchronous → `TooManyRequestsException` (HTTP 429). Asynchronous → retried automatically (up to 2 retries with exponential backoff), then → DLQ or EventBridge Destination.
  - **Reserved concurrency:** A hard cap per function — `ReservedConcurrentExecutions: 50`. The function can never use more than 50 concurrent executions. Also guarantees 50 is always available for that function. Use to: (1) Protect downstream services from being overwhelmed (DynamoDB throughput limits). (2) Guarantee capacity for critical functions. (3) Prevent a runaway function from consuming all account concurrency.
  - **Burst limit:** AWS allows a burst of concurrency above steady state (500–3,000 per region depending on region). Beyond burst, Lambda scales at 500 additional executions per minute. Designed traffic spikes are fine; sudden viral spikes may throttle before scaling catches up.
- **Example:**

```yaml
# Reserved concurrency — protect DynamoDB from overwhelm
OrderProcessor:
  Type: AWS::Lambda::Function
  Properties:
    ReservedConcurrentExecutions: 100
    # DynamoDB table has 100 WCU — each Lambda write uses ~1 WCU
    # 100 concurrent = max 100 parallel writes = safe within DynamoDB capacity

# Setting to 0 — effectively disables the function (useful for emergency circuit breaker)
EmergencyDisable:
  Type: AWS::Lambda::Function
  Properties:
    ReservedConcurrentExecutions: 0 # all invocations throttled immediately


# Monitoring concurrency
# CloudWatch metrics:
# - ConcurrentExecutions: current concurrent executions
# - Throttles: count of throttled invocations
# - UnreservedConcurrentExecutions: account-level unreserved pool consumption
```

- **Technical Terms to Include:** concurrency, reserved concurrency, unreserved concurrency, provisioned concurrency, burst limit, throttle, `TooManyRequestsException`, account-level limit, DLQ on async throttle, scaling rate (500/minute)
- **Gotcha:** Setting `ReservedConcurrentExecutions` to exactly your DynamoDB WCU capacity is a common anti-pattern — it assumes each Lambda invocation does exactly one DynamoDB write per second. In reality, a Lambda may do multiple DynamoDB operations, or run for multiple seconds. A safer approach: set reserved concurrency conservatively, enable DynamoDB auto-scaling, and monitor `ProvisionedThroughputExceededException` errors rather than guessing.
- **Follow-Up:** "What happens to asynchronously invoked Lambdas that are throttled?" → Async throttles (from S3 events, SNS, EventBridge) are automatically retried by the Lambda service — up to 2 retries with 1-minute, then 2-minute backoff. After all retries fail, the event is sent to the **Lambda Destination** (on failure) or the **Dead Letter Queue** (DLQ — SQS or SNS). If neither is configured, the event is **silently dropped**. Always configure a failure destination for async Lambdas — silent event loss is one of the most damaging serverless failure modes. → They're testing async invocation reliability depth.
- **Conclusion:** Lambda concurrency management is the primary lever for protecting downstream services and guaranteeing capacity — reserved concurrency is both a cap (protection from runaway scaling) and a floor (guaranteed availability), and throttle handling via DLQ/Destinations is what determines whether a serverless system is actually reliable under load.

---

### Lambda@Edge vs CloudFront Functions

- **Question:** What are Lambda@Edge and CloudFront Functions, how do they differ, and what problems does each solve?
- **Answer:**
  - **CloudFront Functions:** Lightweight JavaScript functions running at all 400+ CloudFront edge locations. Execution time limit: 1ms. Memory: 2MB. No network calls. No external dependencies. Use for: URL rewriting, header manipulation, A/B testing via cookie inspection, request normalisation, simple auth token validation (JWT signature check from a cached public key). Sub-millisecond latency, globally deployed instantly.
  - **Lambda@Edge:** Full Lambda running at ~13 CloudFront regional edge locations (fewer than CloudFront Functions). Execution time: up to 30s (viewer events), 30s (origin events). Memory: up to 10GB. Can make network calls (call DynamoDB, call an auth service). Supports Node.js and Python. Use for: dynamic content generation, complex auth (database token lookup), personalisation requiring external data, SSR at edge.
  - **Four hook points:** `viewer-request` (before cache), `origin-request` (on cache miss, before origin), `origin-response` (after origin responds), `viewer-response` (before response reaches user). CloudFront Functions run only at viewer request/response. Lambda@Edge runs at all four.
  - **Key trade-offs:** CloudFront Functions: globally consistent, near-zero latency, but can't make network calls — logic must be self-contained. Lambda@Edge: can call external services but adds latency (regional, not PoP-level), slower deploy propagation, higher cost per execution, debugging is harder (logs in the regional CloudWatch, not the edge PoP's region).
- **Example:**

```javascript
// CloudFront Function — URL normalisation (viewer-request)
// Remove trailing slash, lowercase path, strip index.html
function handler(event) {
  var request = event.request;
  var uri = request.uri;

  // /About/ → /about
  uri = uri.toLowerCase();

  // Remove trailing slash (except root)
  if (uri !== '/' && uri.endsWith('/')) {
    uri = uri.slice(0, -1);
  }

  // /index.html → /
  if (uri.endsWith('/index.html')) {
    uri = uri.slice(0, -'index.html'.length);
  }

  request.uri = uri;
  return request;
}

// Lambda@Edge — origin-request: personalised content based on geo + auth
exports.handler = async (event) => {
  const request = event.Records[0].cf.request;
  const headers = request.headers;

  // Read geo from CloudFront headers (injected automatically)
  const country = headers['cloudfront-viewer-country']?.[0]?.value ?? 'US';

  // Rewrite origin path based on country
  if (country === 'GB') {
    request.uri = request.uri.replace('/products/', '/products/uk/');
  }

  return request;
};
```

- **Technical Terms to Include:** CloudFront Functions, Lambda@Edge, edge location, regional edge, viewer-request, origin-request, origin-response, viewer-response, 1ms limit, network calls, cache hit ratio, `cloudfront-viewer-country`, A/B testing, SSR at edge
- **Gotcha:** Lambda@Edge functions cannot use environment variables — they are deployed globally as immutable packages and environment variables are not supported in the Lambda@Edge execution model. Any configuration must be baked into the function code or read from SSM Parameter Store / Secrets Manager via a network call at runtime. This surprises engineers used to normal Lambda development.
- **Follow-Up:** "When would you choose Lambda@Edge over an API Gateway + Lambda origin?" → When the request must be handled at the edge (before hitting the origin) and requires external data. For pure routing/header manipulation — CloudFront Functions. For personalisation requiring a DB call at the edge — Lambda@Edge at origin-request. For everything else (business logic, complex APIs) — API Gateway + Lambda at the origin is simpler to operate, has normal environment variables, and full Lambda tooling support. Lambda@Edge has significant operational overhead for complex use cases. → They're testing edge compute architecture judgment.
- **Conclusion:** CloudFront Functions and Lambda@Edge represent two different points on the latency/capability trade-off: Functions are 1ms globally with no network access, Edge is 30s regional with full network access — the right choice depends entirely on whether the edge logic requires external data, not on which sounds more sophisticated.

---

## L3 — S3 & CloudFront: Static Hosting + CDN Architecture

---

**L3 Topics identified:**

- S3 — storage classes, lifecycle, event notifications, presigned URLs, versioning
- S3 as a static site origin
- CloudFront — distributions, origins, behaviours, cache control
- CloudFront invalidation — cost and strategy
- OAC (Origin Access Control) — replacing OAI
- Cache-Control headers and TTL strategy

---

### S3 Core — Access Patterns, Presigned URLs, Event Notifications

- **Question:** What are S3's key operational patterns — presigned URLs, event notifications, and versioning — and what are the production gotchas for each?
- **Answer:**
  - **Presigned URLs:** Time-limited URLs that grant temporary access to a private S3 object, signed with AWS credentials. Enable browser-direct upload (PUT presigned URL) or download without exposing the bucket publicly. The signature embeds: bucket, key, expiry, HTTP method, and the signing credentials. Anyone with the URL can perform the operation until expiry — treat them as secrets.
  - **Event Notifications:** S3 can notify on `s3:ObjectCreated:*`, `s3:ObjectRemoved:*`, etc. Targets: SQS, SNS, Lambda, EventBridge. Use for: triggering image processing on upload, invalidating CDN cache on file replacement, running ETL on new data files. Delivery is **at-least-once** — Lambda triggers may receive the same event twice. Idempotent handlers are required.
  - **Versioning:** When enabled, every PUT/DELETE creates a new version rather than overwriting. Protects against accidental deletes (delete marker instead of true deletion). Enables cross-region replication and lifecycle rules per version. Cost: versioned objects accumulate; add lifecycle rules to expire non-current versions.
  - **S3 consistency model (since Dec 2020):** Strong read-after-write consistency for all GET/LIST operations. No more eventual consistency propagation delay — a PUT then immediate GET returns the new object. This simplified many retry and wait patterns.
- **Example:**

```javascript
// Presigned URL — browser-direct upload (bypasses your server)
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const s3 = new S3Client({ region: 'us-east-1' });

// Server-side: generate presigned PUT URL
async function generateUploadUrl(filename, contentType) {
  const key = `uploads/${Date.now()}-${filename}`;
  const command = new PutObjectCommand({
    Bucket: 'my-uploads-bucket',
    Key: key,
    ContentType: contentType
    // Optional: enforce content-type and size via conditions
  });
  const url = await getSignedUrl(s3, command, { expiresIn: 300 }); // 5 minutes
  return { url, key };
}

// Client-side: upload directly to S3 with presigned URL
const { url, key } = await fetch('/api/upload-url').then((r) => r.json());
await fetch(url, {
  method: 'PUT',
  body: file,
  headers: { 'Content-Type': file.type }
});

// S3 Event Notification → Lambda (process image after upload)
// S3 sends an event to Lambda when a new object is created
exports.handler = async (event) => {
  for (const record of event.Records) {
    const bucket = record.s3.bucket.name;
    const key = decodeURIComponent(record.s3.object.key.replace(/\+/g, ' '));
    await processImage(bucket, key);
    // Handler must be idempotent — S3 may deliver the same event twice
  }
};
```

- **Technical Terms to Include:** presigned URL, SigV4 signing, at-least-once delivery, idempotent handler, versioning, delete marker, lifecycle rule, non-current version expiration, strong consistency, event notification, S3 bucket policy, CORS, multipart upload
- **Gotcha:** S3 event notifications do not guarantee ordering — multiple concurrent uploads can trigger Lambdas out of order. If your processing depends on event sequence (e.g., thumbnail generation must complete before metadata indexing), use SQS FIFO as an intermediary to enforce order, or design processing to be order-independent.
- **Follow-Up:** "How do you implement browser-direct multipart upload to S3 for large files?" → For files >100MB, use S3 multipart upload: `CreateMultipartUpload` → multiple `UploadPart` (each presigned separately) → `CompleteMultipartUpload`. The server generates presigned URLs for each part; the browser uploads parts in parallel (5MB minimum per part). If upload fails midway, use `ListMultipartUploads` + `AbortMultipartUpload` in a lifecycle rule to clean up incomplete uploads (otherwise they incur storage charges indefinitely). → They're testing large-file upload architecture depth.
- **Conclusion:** S3's simplicity belies its operational complexity at scale — presigned URLs shift data transfer load from your servers to S3 directly (significant cost and latency win), event notifications enable event-driven pipelines, and the at-least-once delivery guarantee makes idempotency a requirement, not an option.

---

### CloudFront — Cache Architecture & Invalidation Strategy

- **Question:** How does CloudFront caching work, how do you design a TTL strategy for a frontend deployment, and what is the cost and strategy of cache invalidation?
- **Answer:**
  - **CloudFront cache hierarchy:** User → Edge PoP (400+ locations) → Regional Edge Cache (13 locations) → Origin (S3, API Gateway, ALB). Cache hit at PoP: served in <20ms, zero origin cost. Cache miss at PoP but hit at regional edge: served without hitting origin. Cache miss everywhere: origin request — your cost and latency.
  - **TTL sources (precedence):** `Cache-Control: max-age` on the object → CloudFront Behaviour `defaultTTL`/`maxTTL` → CloudFront default (86,400s = 24h). Set `Cache-Control` at the object level for fine-grained control.
  - **Frontend deployment strategy (two-category TTL):** (1) **Hashed assets** (`main.abc123.js`, `styles.def456.css`) — content-addressed, never change. Set `Cache-Control: public, max-age=31536000, immutable` (1 year). Cache forever. (2) **Non-hashed entry points** (`index.html`, `manifest.json`) — must always be fresh. Set `Cache-Control: no-cache` or `max-age=0, must-revalidate`. CloudFront re-validates on every request — serves stale only if origin confirms it's unchanged (304). With this strategy, deployments propagate instantly for `index.html`; static assets are cached forever until their hash changes.
  - **Cache Invalidation:** `CreateInvalidation` API. First 1,000 paths per month free; $0.005 per path after. Invalidation is **eventually consistent** — propagates to all edge PoPs within minutes but not instantly. `/*` is one path (wildcard). Use invalidation sparingly — the correct strategy is immutable URLs for static assets so invalidation is rarely needed.
  - **OAC (Origin Access Control):** Restricts S3 to only accept requests from your CloudFront distribution. Replaces the older OAI (Origin Access Identity). Required to prevent direct S3 access bypassing CloudFront.
- **Example:**

```javascript
// Webpack/Vite output — content-hashed filenames
// dist/
//   index.html          → Cache-Control: no-cache
//   assets/index.abc123.js  → Cache-Control: max-age=31536000, immutable
//   assets/styles.def456.css → Cache-Control: max-age=31536000, immutable

// S3 deployment script — set correct headers per file type
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
const s3 = new S3Client({ region: 'us-east-1' });

async function uploadFile(localPath, s3Key) {
  const isHashed = /\.[a-f0-9]{8,}\.\w+$/.test(s3Key); // detect hash in filename
  const cacheControl = isHashed ? 'public, max-age=31536000, immutable' : 'no-cache, no-store, must-revalidate';

  await s3.send(
    new PutObjectCommand({
      Bucket: 'my-frontend-bucket',
      Key: s3Key,
      Body: fs.readFileSync(localPath),
      CacheControl: cacheControl,
      ContentType: lookup(localPath) || 'application/octet-stream'
    })
  );
}

// CloudFront invalidation — only for index.html on deploy
// (hashed assets never need invalidation)
const cloudfront = new CloudFrontClient({});
await cloudfront.send(
  new CreateInvalidationCommand({
    DistributionId: 'E1EXAMPLE',
    InvalidationBatch: {
      CallerReference: Date.now().toString(),
      Paths: { Quantity: 1, Items: ['/index.html'] }
    }
  })
);
```

- **Technical Terms to Include:** CDN, edge PoP, regional edge cache, cache hit, cache miss, TTL, `Cache-Control`, `max-age`, `immutable`, `no-cache`, OAC (Origin Access Control), cache invalidation, content-addressed, hashed filename, behaviours, geo-restriction, signed URLs/cookies
- **Gotcha:** `Cache-Control: no-cache` does NOT mean "don't cache." It means "cache, but always revalidate with the origin before serving." The browser (and CloudFront) may still cache the response — but will send a conditional request (`If-None-Match`) to confirm it's still valid. `Cache-Control: no-store` means truly do not cache. For `index.html` in a SPA deployment: use `no-cache` (revalidate always) rather than `no-store` (wastes bandwidth by never caching) — the 304 Not Modified response from S3 is nearly free.
- **Follow-Up:** "How do you handle CloudFront serving a stale `index.html` after a deployment while a user's browser has cached the old asset hashes?" → This is the classic "stale SPA" problem. Solution: (1) Deploy all new hashed assets before replacing `index.html`. (2) Replace `index.html` last — once deployed, new visitors get the new entry point with new asset hashes. (3) Old visitors with cached `index.html` reference old hashes — which still exist in S3 (don't delete immediately). Retain old assets for at least 24–48h. (4) If the old asset hash is no longer in S3, old `index.html` causes 404s on static assets. → They're testing deployment sequencing awareness for frontend assets.
- **Conclusion:** CloudFront's two-category TTL strategy (immutable hashed assets + no-cache entry points) eliminates the invalidation problem entirely — hashed assets never need invalidation because their URL changes with their content, and entry points always revalidate, ensuring instant propagation of new deployments without CDN cost or eventual-consistency delay.

---

## L4 — API Gateway, DynamoDB, Cognito

---

**L4 Topics identified:**

- API Gateway — REST vs HTTP API, authorisers, throttling, stages
- DynamoDB — partition key design, GSI, capacity modes, transactions, streams
- Cognito — User Pools vs Identity Pools, JWT flow, Hosted UI, Lambda triggers

---

### API Gateway — REST vs HTTP API, Authorisers

- **Question:** What is the difference between API Gateway REST API and HTTP API, how do authorisers work, and what are the throttling mechanics?
- **Answer:**
  - **REST API (v1):** Full-featured — request/response transformations (mapping templates), usage plans, API keys, resource policies, caching, WAF integration. Higher cost ($3.50/million requests). More complex to configure.
  - **HTTP API (v2):** Designed for Lambda and HTTP backends. Simpler, lower latency (~60% faster than REST API), significantly cheaper ($1.00/million requests). Supports: JWT authorisers (Cognito, Auth0), Lambda authorisers, OIDC/OAuth2. Missing: mapping templates, API key usage plans, endpoint-level caching.
  - **Authoriser types:** (1) **JWT Authoriser (HTTP API)** — validates a JWT token against an issuer (Cognito User Pool, Auth0). API Gateway validates signature and claims without calling Lambda. Fast (no Lambda invocation). (2) **Lambda Authoriser** — your Lambda function validates the token and returns an IAM policy or a simple allow/deny. Used for custom auth logic (API key lookup, session token validation, complex claim transformation). Result can be cached by TTL. (3) **Cognito User Pool Authoriser (REST API)** — validates Cognito JWT directly, no Lambda needed.
  - **Throttling:** Account-level default: 10,000 requests/second, 5,000 burst. Per-route or per-stage overrides available. Throttled requests return 429. At REST API level: usage plans per API key allow per-client throttle. Throttling protects Lambda from overwhelming concurrency — coordinate API Gateway throttle with Lambda reserved concurrency.
- **Example:**

```yaml
# HTTP API — JWT authoriser (Cognito)
HttpApi:
  Type: AWS::ApiGatewayV2::Api
  Properties:
    Name: my-api
    ProtocolType: HTTP
    CorsConfiguration:
      AllowOrigins: ['https://example.com']
      AllowMethods: ['GET', 'POST', 'OPTIONS']
      AllowHeaders: ['Authorization', 'Content-Type']

JwtAuthorizer:
  Type: AWS::ApiGatewayV2::Authorizer
  Properties:
    ApiId: !Ref HttpApi
    AuthorizerType: JWT
    IdentitySource: '$request.header.Authorization'
    JwtConfiguration:
      Audience: ['your-cognito-app-client-id']
      Issuer: !Sub 'https://cognito-idp.${AWS::Region}.amazonaws.com/${UserPoolId}'

# Route with authoriser
GetOrdersRoute:
  Type: AWS::ApiGatewayV2::Route
  Properties:
    ApiId: !Ref HttpApi
    RouteKey: 'GET /orders'
    AuthorizationType: JWT
    AuthorizerId: !Ref JwtAuthorizer
    Target: !Sub 'integrations/${LambdaIntegration}'
```

```javascript
// Lambda receives event with auth context (claims already validated)
exports.handler = async (event) => {
  // JWT claims are in event.requestContext.authorizer.jwt.claims
  const userId = event.requestContext.authorizer.jwt.claims.sub;
  const userEmail = event.requestContext.authorizer.jwt.claims.email;
  // No need to re-validate the token — API Gateway already did it
  return fetchOrdersForUser(userId);
};
```

- **Technical Terms to Include:** REST API (v1), HTTP API (v2), JWT authoriser, Lambda authoriser, Cognito User Pool authoriser, throttling, 429, usage plan, API key, CORS, mapping template, stage variable, caching, `$request.header.Authorization`, auth context claims
- **Gotcha:** API Gateway's **default timeout is 29 seconds** — any Lambda that runs longer is killed at the API Gateway level with a 504. Even if your Lambda has a 15-minute timeout, the API Gateway integration cuts it at 29s. This is non-configurable for synchronous integrations. For long-running operations: use async invocation (API Gateway → SQS → Lambda), or return immediately with a job ID and poll for completion.
- **Follow-Up:** "When would you use Lambda Authoriser over JWT Authoriser?" → JWT Authoriser: stateless, validates token signature and claims locally — no external call, very fast, no Lambda cost. JWT Authoriser can't look up revoked tokens (JWT is stateless). Lambda Authoriser: when you need to check a database (token revocation list, session table, API key lookup), transform claims (add permissions not in JWT), or support non-JWT auth schemes (custom header, API key). Cost: Lambda Authoriser adds ~50-100ms latency per cold start plus Lambda invocation cost. Cache the result with a TTL to amortise. → They're testing auth middleware architecture judgment.
- **Conclusion:** HTTP API is the correct choice for new serverless REST APIs — 60% faster, 70% cheaper, and JWT authorisers cover 90% of auth requirements without Lambda invocation overhead; REST API is justified only when usage plans, endpoint caching, or mapping templates are genuinely needed.

---

### DynamoDB — Data Modelling & Access Patterns

- **Question:** How do you design a DynamoDB table for a multi-entity access pattern, and what are the trade-offs of GSIs, LSIs, and single-table design?
- **Answer:**
  - **Core model:** DynamoDB is a key-value and document store. Every item is identified by a **partition key** (PK). An optional **sort key** (SK) enables range queries within a partition. Access is by key — there is no SQL `WHERE` on arbitrary attributes.
  - **Partition key design:** Determines data distribution across DynamoDB's internal partitions. A hot partition key (same PK for many items — e.g., `date` for a logging table) causes all writes to go to one partition — throughput limited to 1,000 WCU/partition. Good PKs have high cardinality and uniform access distribution. Use write sharding for hot keys: `pk = ITEM_${Math.floor(Math.random() * 10)}` → spread across 10 partitions, query all 10 and merge.
  - **GSI (Global Secondary Index):** Alternate PK + SK for the same table. Enables a completely different access pattern. Data is projected (copied) to the GSI asynchronously. Supports `ALL`, `KEYS_ONLY`, or `INCLUDE` projections. Read/write capacity is separate from the base table. Eventual consistency between table and GSI.
  - **Single-table design:** Store multiple entity types in one table. Use generic PK/SK attributes (`PK`, `SK`) with entity-specific values (`PK: USER#42`, `SK: ORDER#99`). Access patterns: get user = `PK: USER#42, SK: USER#42`; get user's orders = `PK: USER#42, SK begins_with ORDER#`. Reduces table proliferation, enables efficient queries across entity types. Trade-off: complex data modelling, hard to query ad-hoc.
  - **Capacity modes:** **On-demand (PAY_PER_REQUEST)** — scales automatically, pay per request. Best for unpredictable or spiky workloads. Higher per-request cost. **Provisioned** — declare WCU/RCU, pay for capacity whether used or not. Lower cost at steady predictable load. Use auto-scaling to adjust provisioned capacity.
- **Example:**

```javascript
// Single-table design — orders and users in one table
// Table: AppTable, PK: pk (String), SK: sk (String)

// Write a user
await dynamo.send(
  new PutItemCommand({
    TableName: 'AppTable',
    Item: {
      pk: { S: 'USER#42' },
      sk: { S: 'USER#42' },
      email: { S: 'alice@example.com' },
      entityType: { S: 'USER' }
    }
  })
);

// Write an order for that user
await dynamo.send(
  new PutItemCommand({
    TableName: 'AppTable',
    Item: {
      pk: { S: 'USER#42' },
      sk: { S: 'ORDER#2026-03-29T10:00:00' }, // sortable ISO timestamp
      total: { N: '99.99' },
      status: { S: 'PAID' },
      entityType: { S: 'ORDER' }
    }
  })
);

// Get all orders for user — single Query, no scan
await dynamo.send(
  new QueryCommand({
    TableName: 'AppTable',
    KeyConditionExpression: 'pk = :pk AND begins_with(sk, :prefix)',
    ExpressionAttributeValues: {
      ':pk': { S: 'USER#42' },
      ':prefix': { S: 'ORDER#' }
    }
  })
);

// GSI — query by order status across all users
// GSI: pk=status, sk=createdAt
await dynamo.send(
  new QueryCommand({
    TableName: 'AppTable',
    IndexName: 'status-createdAt-index',
    KeyConditionExpression: 'pk = :status AND sk > :since',
    ExpressionAttributeValues: {
      ':status': { S: 'PAID' },
      ':since': { S: '2026-03-01' }
    }
  })
);
```

- **Technical Terms to Include:** partition key, sort key, GSI, LSI, hot partition, write sharding, single-table design, on-demand vs provisioned capacity, `Query` vs `Scan`, `begins_with`, `between`, conditional write, transactions, DynamoDB Streams, projection
- **Gotcha:** **`Scan` is almost always wrong in production.** A `Scan` reads every item in the table — it consumes RCU proportional to the entire table size regardless of how many items match. A table with 10 million items filtered to 10 results reads all 10 million. Use `Query` (always) with appropriate GSIs for access patterns you didn't anticipate at table design time. If you need to `Scan`, do it off-peak with `Limit` and pagination to avoid consuming all throughput.
- **Follow-Up:** "What are DynamoDB Streams and when do you use them?" → DynamoDB Streams captures item-level changes (INSERT, MODIFY, REMOVE) as an ordered, sharded stream — retains records for 24 hours. Each stream record contains the before and after image of the changed item (configurable: KEYS_ONLY, NEW_IMAGE, OLD_IMAGE, NEW_AND_OLD_IMAGES). Use for: triggering downstream processing (Lambda reads stream → updates Elasticsearch), event sourcing (every change is an event), cross-region replication (Global Tables uses Streams internally), cache invalidation on DynamoDB write. Critical: DynamoDB Streams retention is 24 hours — your Lambda consumer must keep up or events are lost. → They're testing CDC (change data capture) pattern knowledge.
- **Conclusion:** DynamoDB's access pattern–first design philosophy is the fundamental shift from relational thinking — you design the table around known queries, not normalised entities, and GSIs enable additional access patterns at the cost of storage duplication and eventual consistency, making access pattern analysis the prerequisite for every DynamoDB schema decision.

---

### Cognito — User Pools, Identity Pools, JWT Flow

- **Question:** What is the difference between Cognito User Pools and Identity Pools, how does the JWT authentication flow work, and what are Lambda triggers used for?
- **Answer:**
  - **User Pool:** A user directory — handles registration, login, MFA, password reset, social IdP federation (Google, Facebook, SAML). Issues JWT tokens (ID token, Access token, Refresh token) on successful auth. The **ID token** contains user attributes (email, name, custom attributes). The **Access token** contains scopes and groups. Use User Pool tokens to authorise API Gateway calls.
  - **Identity Pool (Federated Identities):** Exchanges credentials (Cognito User Pool tokens, social IdPs, SAML, even unauthenticated) for **temporary AWS credentials** (via STS). Enables users to call AWS services directly (S3 PutObject, DynamoDB Query) with their IAM role — no backend intermediary. Two roles: authenticated and unauthenticated (guest).
  - **JWT flow:** (1) User authenticates → Cognito returns ID + Access + Refresh tokens. (2) Client stores tokens (memory or secure HttpOnly cookie). (3) Client sends `Authorization: Bearer <access_token>` with API requests. (4) API Gateway JWT Authoriser validates signature against Cognito's JWKS endpoint, checks expiry and audience. (5) Valid → Lambda receives event with claims. (6) Access token expires (default: 1h) → client uses refresh token to get new tokens silently.
  - **Lambda triggers:** Hooks into the auth flow. **Pre-signup** — validate/enrich user before account creation. **Post-confirmation** — create a user record in DynamoDB after email verification. **Pre-token generation** — add custom claims to the JWT (e.g., permissions from DynamoDB). **Custom authentication** — fully custom auth challenge (OTP via SMS, TOTP). Pre-token generation is the most powerful — inject role, tenant ID, feature flags into every token.
- **Example:**

```javascript
// Pre-token generation trigger — add custom claims to JWT
exports.handler = async (event) => {
  // Fetch user's permissions from DynamoDB
  const permissions = await getUserPermissions(event.userName);
  const tenantId = await getUserTenant(event.userName);

  // Inject into token
  event.response = {
    claimsOverrideDetails: {
      claimsToAddOrOverride: {
        permissions: permissions.join(','), // "read:orders,write:orders"
        tenantId: tenantId // multi-tenant isolation
      },
      claimsToSuppress: ['cognito:groups'] // remove default groups claim
    }
  };
  return event;
};

// Client-side — Amplify v6 auth flow
import { signIn, getCurrentUser, fetchAuthSession } from 'aws-amplify/auth';

const { tokens } = await fetchAuthSession();
const idToken = tokens.idToken.toString();
const accessToken = tokens.accessToken.toString();

// API call with access token
const response = await fetch('/api/orders', {
  headers: { Authorization: `Bearer ${accessToken}` }
});

// Decode ID token claims (without re-validation — trust API Gateway to validate)
const claims = JSON.parse(atob(idToken.split('.')[1]));
const { sub, email, tenantId, permissions } = claims;
```

- **Technical Terms to Include:** User Pool, Identity Pool, JWT, ID token, Access token, Refresh token, JWKS, token expiry, Lambda trigger, pre-token generation, hosted UI, social federation, STS temporary credentials, Amplify, `sub` claim, audience, issuer
- **Gotcha:** Cognito User Pool **Access tokens and ID tokens are different** — use the right one for the right purpose. API Gateway JWT Authoriser should validate the **Access token** (contains scopes, designed for API authorisation). ID token contains user attributes and is intended for the client application (knowing who the user is, not authorising API calls). Using the ID token with API Gateway is a common mistake — it works, but violates the OIDC specification intent and breaks if Cognito changes claim structure.
- **Follow-Up:** "How do you implement multi-tenancy with Cognito?" → Two patterns: (1) **Separate User Pool per tenant** — full isolation, independent configuration, separate Hosted UI domains. Complex to manage at scale (hundreds of tenants = hundreds of pools). (2) **Single User Pool with tenant attribute** — add a custom `tenantId` attribute, inject it into tokens via pre-token generation Lambda, and enforce tenant-based access control in API/Lambda by reading `tenantId` from token claims. Simpler to manage, slightly weaker isolation. Choose based on compliance requirements (separate pools for regulatory isolation) vs operational overhead. → They're testing multi-tenant architecture design with Cognito.
- **Conclusion:** Cognito's User Pool + API Gateway JWT Authoriser + Lambda pre-token generation is the canonical serverless auth stack — the pre-token generation trigger is the most powerful extension point, enabling custom claims (permissions, tenant ID, feature flags) to be injected into every token without a separate auth service.

---

## L5 — SQS, SNS: Event-Driven Decoupling

---

**L5 Topics identified:**

- SQS — standard vs FIFO, visibility timeout, DLQ, long polling, Lambda trigger
- SNS — topics, subscriptions, message filtering, fan-out pattern
- SQS + SNS combined — fan-out to multiple queues
- Event-driven architecture with Lambda

---

### SQS — Visibility Timeout, DLQ, Lambda Integration

- **Question:** How does SQS work, what is visibility timeout, how does the DLQ protect against poison messages, and how does the Lambda trigger interact with SQS?
- **Answer:**
  - **SQS standard queue:** At-least-once delivery, best-effort ordering. High throughput (unlimited TPS). Use for most workloads where order doesn't matter and idempotent processing is possible.
  - **SQS FIFO queue:** Exactly-once processing (within a message group), strict ordering within a group. 3,000 TPS with batching, 300 TPS without. Use for financial transactions, order state machines, anything where sequence and exactly-once matters.
  - **Visibility timeout:** When a consumer receives a message, it becomes invisible to other consumers for the visibility timeout period (default 30s). If the consumer acknowledges (deletes) the message within that time, it's removed. If not (consumer crashes, processing fails), it becomes visible again and is redelivered. Set visibility timeout to slightly longer than your maximum expected processing time — if timeout expires before processing completes, you get duplicate processing.
  - **DLQ (Dead Letter Queue):** After a message is received and returned (nack/visibility timeout expired) `maxReceiveCount` times, SQS moves it to a DLQ. Prevents poison messages (malformed, permanently failing) from looping forever and blocking the queue. DLQ is just another SQS queue — inspect, alert on, and replay from it.
  - **Lambda + SQS trigger:** Lambda polls the queue (SQS trigger handles the polling — you don't poll yourself). Lambda processes up to `batchSize` messages per invocation. If the Lambda succeeds: all messages in the batch are deleted. If it throws: all messages in the batch become visible again (full batch retry). Partial batch failures require `ReportBatchItemFailures` — return which message IDs failed, so only those are retried.
- **Example:**

```javascript
// Lambda SQS handler — with partial batch failure reporting
exports.handler = async (event) => {
  const batchItemFailures = [];

  for (const record of event.Records) {
    try {
      const body = JSON.parse(record.body);
      await processMessage(body);
    } catch (error) {
      console.error(`Failed to process ${record.messageId}:`, error);
      // Report this specific message as failed — others in batch succeed
      batchItemFailures.push({ itemIdentifier: record.messageId });
    }
  }

  // Return failed items — only these are retried, not the whole batch
  return { batchItemFailures };
};

// SQS queue configuration with DLQ
OrderQueue:
  Type: AWS::SQS::Queue
  Properties:
    QueueName: order-processing-queue
    VisibilityTimeout: 300        # 5 minutes — Lambda timeout should be ≤ this
    MessageRetentionPeriod: 86400  # 1 day
    RedrivePolicy:
      deadLetterTargetArn: !GetAtt OrderDLQ.Arn
      maxReceiveCount: 3           # after 3 failures → DLQ

OrderDLQ:
  Type: AWS::SQS::Queue
  Properties:
    QueueName: order-processing-dlq
    MessageRetentionPeriod: 1209600  # 14 days — time to investigate

# CloudWatch alarm on DLQ depth — alert on any dead letters
DLQAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    MetricName: ApproximateNumberOfMessagesVisible
    Namespace: AWS/SQS
    Dimensions: [{ Name: QueueName, Value: !GetAtt OrderDLQ.QueueName }]
    ComparisonOperator: GreaterThanThreshold
    Threshold: 0
    EvaluationPeriods: 1
    AlarmActions: [!Ref AlertSNSTopic]
```

- **Technical Terms to Include:** standard queue, FIFO queue, visibility timeout, at-least-once delivery, exactly-once, DLQ, `maxReceiveCount`, `ReportBatchItemFailures`, partial batch failure, long polling, `WaitTimeSeconds`, batch size, `ReceiveMessage`, message group ID, deduplication ID
- **Gotcha:** The Lambda SQS trigger's visibility timeout must be coordinated with Lambda's function timeout. If the function timeout (e.g., 5 minutes) exceeds the queue's visibility timeout (e.g., 30 seconds), the message becomes visible again while Lambda is still processing it — another Lambda instance picks it up — you have duplicate processing. Rule: **SQS visibility timeout ≥ Lambda function timeout × 6** (AWS recommendation for buffer). Set both explicitly and keep them in sync.
- **Follow-Up:** "How do you replay messages from a DLQ after fixing a bug?" → Use the SQS DLQ redrive feature (available in console and API): set the DLQ as the source queue and the original queue as the destination. AWS moves messages back. Alternatively, write a Lambda that reads from DLQ and writes to the main queue — gives you control over rate and filtering. For large DLQs: use SQS DLQ Management Console's batch redrive. Never manually process DLQ messages one-by-one — it doesn't scale and doesn't account for ordering in FIFO queues. → They're testing operational recovery patterns.
- **Conclusion:** SQS's visibility timeout is the mechanism that provides at-least-once delivery with automatic retry — DLQ protects against poison messages breaking queue throughput indefinitely, and partial batch failure reporting via `ReportBatchItemFailures` is the difference between retrying one failed message and retrying an entire batch of 10 unnecessarily.

---

### SNS Fan-out Pattern & Message Filtering

- **Question:** What is SNS, how does the fan-out pattern with SQS work, and how does message filtering reduce unnecessary Lambda invocations?
- **Answer:**
  - **SNS (Simple Notification Service):** A pub/sub messaging service. Publishers send to a **topic**; subscribers receive from it. Supports: SQS, Lambda, HTTP/HTTPS, email, SMS, mobile push as subscribers. Delivery is **at-most-once** to each subscriber (no retry guarantee for HTTP endpoints; SQS subscribers get retry via SQS).
  - **Fan-out pattern (SNS → multiple SQS):** One SNS publish delivers the message to multiple SQS queues simultaneously. Each queue has an independent consumer (Lambda or otherwise). Enables: parallel processing by multiple teams/services from a single event source, decoupled microservices, different processing speeds per consumer. This is the standard pattern for event-driven microservice integration.
  - **Message filtering:** Each SNS subscription can define a **filter policy** — JSON that matches attributes on the message. Only messages matching the filter are delivered to that subscription. A subscription with no filter receives all messages. Use to: route events by type (`eventType: order.paid` → payment-processor queue), by environment (`env: production` → skip dev queues), or by region.
  - **FIFO SNS + FIFO SQS:** SNS now supports FIFO topics — exactly-once delivery, strict ordering, fan-out to FIFO SQS queues. For financial event ordering across multiple consumers.
- **Example:**

```javascript
// Fan-out: one SNS publish → multiple SQS queues
// Architecture:
// OrderService → OrderEventsTopic (SNS)
//   → PaymentQueue (SQS) → PaymentLambda
//   → InventoryQueue (SQS) → InventoryLambda
//   → AuditQueue (SQS) → AuditLambda (all events)
//   → NotificationQueue (SQS, filtered: order.paid only) → EmailLambda

// Publish with message attributes (for filtering)
await sns.send(new PublishCommand({
  TopicArn: 'arn:aws:sns:us-east-1:123456789:order-events',
  Message: JSON.stringify(orderEvent),
  MessageAttributes: {
    eventType: { DataType: 'String', StringValue: 'order.paid' },
    region:    { DataType: 'String', StringValue: 'us-east-1' },
    amount:    { DataType: 'Number', StringValue: '99.99' },
  }
}));

// SNS Subscription filter policy — notification queue gets only order.paid
FilterPolicy:
  eventType:
    - "order.paid"           # only paid orders → email notification

# Inventory queue gets paid AND shipped (multiple values)
FilterPolicy:
  eventType:
    - "order.paid"
    - "order.shipped"

# Audit queue — no filter, receives everything
FilterPolicy: {}
```

- **Technical Terms to Include:** SNS topic, subscription, filter policy, message attributes, fan-out, at-most-once, FIFO SNS, SQS fan-out, dead-letter policy on SNS subscription, cross-account SNS, mobile push, HTTP subscription retry policy
- **Gotcha:** SNS delivers to Lambda **at-most-once** with no retry if Lambda throttles. If the Lambda subscriber is throttled (concurrency exhausted), SNS does not queue the event — it's lost. The correct pattern for reliable Lambda invocation from SNS: SNS → SQS → Lambda. SQS buffers messages when Lambda is throttled and retries automatically. Direct SNS → Lambda is appropriate only for non-critical, loss-tolerant events (notifications, metrics). For reliable event processing, always add SQS in between.
- **Follow-Up:** "How do you debug a fan-out architecture where some events are missing downstream?" → Check in order: (1) SNS delivery metrics — `NumberOfMessagesFailed` per subscription. (2) SQS DLQ depth — messages that failed processing. (3) Lambda `Throttles` metric — if Lambda was throttled, SNS direct deliveries are lost. (4) SNS subscription filter — does the filter policy match the message attributes? (5) Enable SNS access logs (CloudWatch Logs) to see all delivery attempts. → They're testing event-driven observability debugging approach.
- **Conclusion:** The SNS + SQS fan-out pattern is the foundational event distribution architecture in AWS — SNS delivers once to N SQS queues in parallel, SQS provides buffering and retry for each consumer independently, and filter policies ensure each consumer only processes events it cares about, preventing unnecessary compute and reducing coupling.

---

## L6 — Architecture Patterns & Decision Framework

---

### Serverless Architecture Patterns

- **Question:** Describe the standard serverless architecture for a full-stack web application on AWS, covering the frontend hosting, API layer, auth, data, and messaging planes.
- **Answer:**
  - **Frontend plane:** Static assets (JS, CSS, images) in S3 with content-addressed filenames and `immutable` cache headers. CloudFront distribution with S3 Origin Access Control — users never access S3 directly. `index.html` with `no-cache`. CloudFront Functions for URL normalisation, A/B routing, security headers injection.
  - **API plane:** HTTP API Gateway (cheaper, faster) in front of Lambda functions. JWT authoriser validates Cognito tokens — no Lambda invocation for auth. One Lambda per route group (not one per route — reduces cold starts, enables shared initialisation). API Gateway throttle aligned with Lambda reserved concurrency.
  - **Auth plane:** Cognito User Pool for user directory. Hosted UI or custom UI via Amplify. Pre-token generation Lambda injects permissions and tenant ID. ID token for user profile; Access token for API authorisation. Token refresh handled client-side via Cognito SDK.
  - **Data plane:** DynamoDB on-demand for primary data store. Single-table design for related entities. GSIs for alternate access patterns. DynamoDB Streams → Lambda → Elasticsearch (for search), → SQS (for downstream processing). S3 for blob storage (files, images) — never store blobs in DynamoDB.
  - **Messaging plane:** SNS → SQS → Lambda for async event processing. DLQ on all SQS queues. CloudWatch alarms on DLQ depth. EventBridge for scheduled jobs (cron) and cross-service event routing. SQS FIFO for ordered financial transactions.

```
┌─────────────────────────────────────────────────────────────┐
│                    CloudFront Distribution                   │
│  CloudFront Functions (URL rewrite, security headers)       │
├────────────────────┬────────────────────────────────────────┤
│    S3 (static)     │     API Gateway HTTP API               │
│  index.html (no-cache)│   JWT Authoriser (Cognito)          │
│  *.js/*.css (immutable)│  Lambda functions (route handlers) │
└────────────────────┴──────────┬─────────────────────────────┘
                                │
              ┌─────────────────┼─────────────────────┐
              │                 │                     │
         DynamoDB          S3 (uploads)            SNS Topic
         + Streams          (presigned PUT)         │
              │                                  SQS queues
         Lambda (CDC)                              │
              │                              Lambda consumers
         Elasticsearch                              │
         (search index)                          DLQ + Alarm
```

- **Technical Terms to Include:** static hosting, OAC, immutable cache, HTTP API, JWT authoriser, single-table design, DynamoDB Streams, fan-out, DLQ, EventBridge, VPC (when needed), Lambda@Edge, CloudFront Functions, serverless, IaC (CDK/SAM/Terraform)
- **Gotcha:** This architecture has **no VPC by default** — Lambda and API Gateway run outside a VPC. If your architecture adds RDS (PostgreSQL, MySQL), Lambda must be inside a VPC to reach it — which adds a cold start penalty (~500ms for ENI provisioning, now reduced with Hyperplane ENI but still non-zero). Design for VPC-less where possible. DynamoDB, S3, SQS, SNS all have VPC endpoints if you need private access without NAT Gateway cost.
- **Follow-Up:** "How do you implement infrastructure as code for this architecture?" → AWS CDK (TypeScript) is the current best practice — type-safe, composable constructs, escape hatch to raw CloudFormation. AWS SAM for pure serverless (simpler, native Lambda/API Gateway support). Terraform for multi-cloud or team preference. CDK generates CloudFormation under the hood — understanding CloudFormation concepts (stacks, resources, outputs, cross-stack references) is still necessary for debugging CDK deployments. → They're testing IaC architectural preference and tooling depth.
- **Conclusion:** The canonical AWS serverless stack (CloudFront + S3 + API Gateway + Lambda + DynamoDB + Cognito + SQS/SNS) covers the majority of web application requirements with zero server management, automatic scaling, and pay-per-use pricing — the architecture decision is not whether to use serverless, but which specific configuration of these services matches the access patterns, ordering requirements, and latency budget of the specific use case.

---

## Quick Reference — Service Decision Logic

| Decision                          | Service                               | Avoid                                              |
| --------------------------------- | ------------------------------------- | -------------------------------------------------- |
| Static frontend hosting           | S3 + CloudFront                       | EC2 for static files                               |
| REST API                          | HTTP API Gateway (v2)                 | REST API (v1) unless you need usage plans/caching  |
| Auth                              | Cognito User Pool + JWT Authoriser    | Rolling custom JWT validation                      |
| Serverless compute                | Lambda                                | EC2 for stateless, event-driven work               |
| Edge compute (no network calls)   | CloudFront Functions (1ms, 400+ PoPs) | Lambda@Edge for simple URL rewriting               |
| Edge compute (with network calls) | Lambda@Edge (30s, 13 regional)        | —                                                  |
| NoSQL flexible schema             | DynamoDB                              | RDS for schema-less data                           |
| Ordered financial transactions    | DynamoDB FIFO + SQS FIFO              | Standard queue for ordered data                    |
| Async task queue                  | SQS → Lambda                          | Direct Lambda invocation for async                 |
| Fan-out to multiple consumers     | SNS → SQS → Lambda                    | Direct SNS → Lambda (unreliable)                   |
| Blob/file storage                 | S3                                    | DynamoDB for files                                 |
| Cold start elimination            | Provisioned Concurrency               | SnapStart for elimination; SnapStart for reduction |
| Scheduled jobs                    | EventBridge Scheduler                 | CloudWatch Events (legacy, replaced)               |

---

_— End of AWS_Interview_QA.md —_
_Total coverage: Lambda · Lambda@Edge · S3 · CloudFront · API Gateway · DynamoDB · Cognito · SQS · SNS_
_L1–L6 | 14 QA units | Calibrated for Tech Lead / Architect level_
