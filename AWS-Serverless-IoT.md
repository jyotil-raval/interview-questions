<!-- Generated: 2026-04-09 | AWS Serverless + AWS IoT Core | Architect Level -->

# AWS Serverless Architecture & IoT Core Interview Preparation

## Tech Lead / Architect Level

> Step Functions · EventBridge · X-Ray · SAM/CDK · AWS IoT Core · MQTT · Device Shadow
> Generated: 2026-04-09 | Career OS

---

## Relationship to Existing Files

**`AWS_Interview_QA.md` already covers (do not re-study):**
Lambda (cold start, concurrency, @Edge), S3, CloudFront, API Gateway, DynamoDB,
Cognito, SQS, SNS + the canonical serverless architecture pattern.

**This file covers the delta:**

**Section 1 — Serverless Architecture (deep):**

- Step Functions — state machine orchestration (the missing orchestration layer)
- EventBridge — event bus, rules, scheduling (vs SNS/SQS)
- AWS X-Ray — distributed tracing across Lambda chains
- SAM vs CDK — IaC for serverless, when each applies
- Serverless cost architecture — where bills come from, how to optimise
- Serverless failure modes at scale — what breaks that tutorials don't show

**Section 2 — AWS IoT Core:**

- What IoT Core is — the managed MQTT broker + device platform
- MQTT protocol — QoS levels, topic hierarchy, retained messages, LWT
- Device Shadow — the offline state synchronisation pattern
- Rules Engine — SQL-based telemetry routing to AWS services
- Security model — X.509 certificates, IoT policies, least-privilege
- IoT + Lambda/DynamoDB/Kinesis — the data pipeline
- IoT Greengrass — edge computing extension
- Production patterns — fleet management, OTA, telemetry at scale

**IoT anchor:** Your Volansys IIoT work (ClearBlade/MQTT, HVAC/water heater device
management, AWS Lambda + DynamoDB) maps directly to AWS IoT Core architecture.
ClearBlade is a private MQTT broker + device management platform — AWS IoT Core
is the fully managed AWS equivalent. Frame answers from that production anchor.

---

# SECTION 1 — AWS Serverless Architecture (Deep)

---

**Topics identified:**

- Step Functions — Express vs Standard, state types, error handling
- EventBridge — event bus vs SNS/SQS, rules, event patterns, scheduling
- X-Ray — distributed tracing for Lambda chains, sampling, annotations
- SAM vs CDK — IaC selection for serverless
- Serverless cost optimisation — Lambda, API GW, DynamoDB cost drivers
- Serverless failure modes — cold starts at scale, thundering herd, poison events

---

### Step Functions — Orchestrating Multi-Step Serverless Workflows

- **Question:** What is AWS Step Functions, how do Express and Standard workflows differ, and when does Step Functions replace SQS-chained Lambdas?
- **Answer:**
  - **Step Functions** is AWS's serverless state machine service. You define a workflow as a JSON/YAML state machine (Amazon States Language) — a graph of states: Task (invoke Lambda/SDK), Choice (branch on condition), Wait (pause), Parallel (concurrent branches), Map (fan-out over an array), Pass (transform data), Succeed/Fail. The Step Functions service manages state, handles retries, and tracks progress — you don't write coordination logic.
  - **Standard Workflows:** Exactly-once execution. Max duration: 1 year. Execution history retained for 90 days — every state transition is auditable. Price: per state transition ($0.025/1,000 transitions). Use for: long-running business processes (order fulfilment, data processing pipelines, human-approval workflows), anything needing audit trail, anything running >5 minutes.
  - **Express Workflows:** At-least-once execution. Max duration: 5 minutes. No execution history stored by default (send to CloudWatch Logs explicitly). Price: per execution + duration (cheaper for high-throughput, short workflows). Use for: high-throughput event processing (IoT data ingestion, log processing, real-time streaming), workflows where cost per invocation matters more than audit history.
  - **When Step Functions beats SQS-chained Lambdas:**
    - SQS chains: Lambda A writes to SQS → Lambda B reads from SQS → Lambda B writes to another SQS → Lambda C. Each step requires: SQS queue provisioning, Lambda trigger setup, DLQ config, manual state passing via queue messages, no visibility into where a specific workflow is in its execution.
    - Step Functions: state passed automatically between steps, execution graph visible in console, retries with exponential backoff built-in, parallel branches native, error handling (Catch/Retry) on each state, execution history queryable. The operational overhead is lower for anything beyond 2–3 chained steps.
  - **SDK integrations (direct):** Step Functions can call DynamoDB, SQS, SNS, ECS, Glue, Bedrock — without a Lambda in the middle. Reduces cost and cold start latency for service-to-service orchestration.
- **Example:**

```json
// Amazon States Language — Order Fulfilment workflow
{
  "Comment": "Order fulfilment — reserve inventory, charge payment, notify",
  "StartAt": "ReserveInventory",
  "States": {
    "ReserveInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:reserve-inventory",
      "Retry": [
        {
          "ErrorEquals": ["Lambda.ServiceException", "Lambda.AWSLambdaException"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["InsufficientStock"],
          "Next": "OrderFailed",
          "ResultPath": "$.error"
        }
      ],
      "Next": "ChargePayment"
    },
    "ChargePayment": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:putItem", // SDK integration — no Lambda
      "Parameters": {
        "TableName": "payments",
        "Item": {
          "orderId": { "S.$": "$.orderId" },
          "status": { "S": "CHARGED" }
        }
      },
      "Next": "NotifyCustomer"
    },
    "NotifyCustomer": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish", // direct SDK — no Lambda
      "Parameters": {
        "TopicArn": "arn:aws:sns:...:order-notifications",
        "Message.$": "States.Format('Order {} confirmed', $.orderId)"
      },
      "Next": "OrderComplete"
    },
    "OrderComplete": { "Type": "Succeed" },
    "OrderFailed": { "Type": "Fail", "Error": "InsufficientStock" }
  }
}
```

- **Technical Terms to Include:** Amazon States Language, Standard Workflow, Express Workflow, state types (Task/Choice/Wait/Parallel/Map/Pass/Succeed/Fail), Retry, Catch, SDK integrations, execution history, state transition pricing, `ResultPath`, `InputPath`, `OutputPath`, `Parameters`, `States.Format`
- **Gotcha:** Step Functions **passes the entire state between steps by default** — if an early step returns a large payload (5MB image, full list of records), subsequent steps receive that entire payload. The Step Functions execution input/output limit is 256KB. Exceeding it silently truncates or errors. For large payloads: store the data in S3, pass only the S3 key between steps. Use `ResultPath` to merge task output into a specific field rather than replacing the entire state.
- **Follow-Up:** "When would you use Step Functions Map state over SQS fan-out?" → Map state iterates over an array and runs each item through a sub-state-machine in parallel (up to 40 concurrent). The results are collected and returned as an array to the next state. Best for: bounded fan-out where you need to collect and aggregate all results (process 50 records → gather all results → proceed). SQS fan-out is better for: unbounded or very high-volume fan-out (thousands of messages), where results don't need to be collected synchronously. → They're testing orchestration vs messaging fan-out judgment.
- **Conclusion:** Step Functions is the missing coordination layer between Lambda functions — it replaces ad-hoc SQS chains with an explicit, auditable, retry-aware state machine, and SDK integrations (direct calls to DynamoDB, SNS, S3 without Lambda) reduce both cost and cold start latency for service orchestration.

---

### EventBridge — Event Bus vs SNS/SQS

- **Question:** What is EventBridge, how does it differ from SNS and SQS, and what are the production patterns for event-driven architectures on AWS?
- **Answer:**
  - **EventBridge** is AWS's serverless event bus. Publishers (sources) send events to an event bus. Rules on the bus match events by pattern and route to targets (Lambda, SQS, SNS, Step Functions, API Gateway, other buses — 20+ target types). Unlike SNS (push to all subscribers) and SQS (single consumer queue), EventBridge provides **content-based routing via event patterns** — route based on any field in the event body, not just topic name.
  - **Three bus types:** (1) **Default bus** — receives AWS service events automatically (EC2 state changes, S3 events, CodePipeline stage changes, etc.) at no cost. (2) **Custom bus** — your application events. (3) **Partner bus** — receive events from SaaS partners (Stripe, Zendesk, GitHub) directly into your account.
  - **EventBridge vs SNS/SQS:**
    - **SNS:** Push to multiple subscribers simultaneously (fan-out). Filter by message attribute (not message body content). No replay. Best for: immediate notification fan-out.
    - **SQS:** Decoupled queue with backpressure, DLQ, visibility timeout. One consumer group per queue. Best for: workload buffering, rate-limiting downstream consumers.
    - **EventBridge:** Content-based routing (filter on any JSON field), cross-account/region event routing, built-in schema registry, event archiving and replay, 14 target types without Lambda. Best for: cross-service event routing, microservice decoupling, AWS service event reactions, multi-target routing with business logic filtering.
  - **EventBridge Scheduler:** Cron/rate-based scheduling to any target. Replaces CloudWatch Events Scheduler (which it absorbed). Supports one-time schedules (fire at a future time once) and recurring schedules. DLQ support for missed executions.
  - **EventBridge Pipes:** Point-to-point connection from a source (SQS, Kinesis, DynamoDB Stream, Kafka) through optional filtering and enrichment (Lambda, Step Functions, API Gateway) to a target — without writing Lambda glue code. Significantly reduces boilerplate for source→transform→target pipelines.
- **Example:**

```json
// EventBridge Rule — route OrderPlaced events to multiple targets
{
  "source": ["com.company.order-service"],
  "detail-type": ["OrderPlaced"],
  "detail": {
    "status": ["CONFIRMED"],          // content-based filter on event body
    "amount": [{ "numeric": [">", 100] }]  // only orders > £100
  }
}

// Target 1: Lambda (process high-value orders)
// Target 2: SQS (queue for fulfilment service)
// Target 3: Step Functions (start fulfilment workflow)
// All three triggered by the SAME event — no SNS intermediate needed

// EventBridge Scheduler — daily report at 8am UTC
{
  "ScheduleExpression": "cron(0 8 * * ? *)",
  "FlexibleTimeWindow": { "Mode": "FLEXIBLE", "MaximumWindowInMinutes": 15 },
  "Target": {
    "Arn": "arn:aws:lambda:...:function:generate-daily-report",
    "RoleArn": "arn:aws:iam::...:role/scheduler-role"
  },
  "DeadLetterConfig": { "Arn": "arn:aws:sqs:...:scheduler-dlq" }
}
```

- **Technical Terms to Include:** EventBridge, event bus, event rule, event pattern, content-based routing, default bus, custom bus, partner bus, EventBridge Scheduler, EventBridge Pipes, schema registry, archive and replay, cross-account routing, `source`, `detail-type`, `detail`, target, flexible time window
- **Gotcha:** EventBridge rule **delivery is at-least-once** and does not guarantee ordering. If your target Lambda must process events in strict order (e.g., order state machine transitions), EventBridge alone is insufficient — add SQS FIFO as the target and process from there. EventBridge → SQS FIFO → Lambda preserves ordering within a message group. Direct EventBridge → Lambda does not.
- **Follow-Up:** "When would you choose EventBridge over a direct Lambda invocation?" → Direct invocation couples the caller to the callee (must know the Lambda ARN, handles retry logic itself, synchronous by default). EventBridge decouples them — the publisher emits an event to the bus, the bus routes to targets. Adding a new consumer requires a new rule (no producer change). Cross-account routing is native. Use EventBridge when: multiple targets need the same event, the consumer should change without producer code change, or AWS service events need to trigger your Lambda. Use direct invocation for: simple, internal, single-consumer calls where decoupling adds no value. → They're testing event routing architecture judgment.
- **Conclusion:** EventBridge is the AWS-native event backbone — content-based routing without Lambda glue, cross-account fan-out, AWS service event consumption, and Scheduler replace what previously required SNS + Lambda + CloudWatch Events, making it the default event routing layer in any well-architected serverless system.

---

### AWS X-Ray — Distributed Tracing for Serverless

- **Question:** What is AWS X-Ray, how does tracing propagate across Lambda invocations, and what does a production X-Ray setup look like?
- **Answer:**
  - **AWS X-Ray** is AWS's distributed tracing service. It traces a request as it flows through Lambda functions, API Gateway, DynamoDB, SQS, Step Functions, and other services. Each hop adds a **segment** (an X-Ray data record). Together, the segments form a **trace** — an end-to-end picture of the request lifecycle with timing, errors, and metadata.
  - **Trace propagation:** X-Ray uses a **trace header** (`X-Amzn-Trace-Id`) — injected by API Gateway on inbound requests. Lambda reads this header from the event, continues the trace, and propagates it to downstream calls (DynamoDB SDK, HTTP calls, SQS sends). The AWS SDK v3 automatically adds X-Ray segments for all SDK calls when X-Ray tracing is active. For HTTP calls to non-AWS services, use the X-Ray SDK's `captureHTTPS` wrapper.
  - **Enabling on Lambda:** `TracingConfig: Active` in CloudFormation/CDK or the console. Lambda automatically creates segments for each invocation — cold start segment (init phase) and invocation segment (handler phase). SDK calls within the handler appear as **subsegments**.
  - **Annotations vs Metadata:** **Annotations** — key-value pairs indexed for filtering in the X-Ray console (`userId`, `orderId`, environment). Query: "find all traces where orderId=42." **Metadata** — arbitrary data stored with the segment (full request body, response payload). Not indexed, not filterable. Use annotations for anything you'll query on.
  - **Sampling:** X-Ray samples traces by default (1 req/sec + 5% of additional requests) to control cost. Custom sampling rules per route: `POST /orders` → 100% sampling (critical path). `GET /health` → 0% (noise). Defined in the X-Ray console or as JSON sampling rules.
  - **Service Map:** X-Ray builds a visual graph of your services and their call relationships with average latency and error rate per edge — the fastest way to identify which service in a chain is the bottleneck.
- **Example:**

```javascript
// Lambda function with X-Ray SDK instrumentation
const AWSXRay = require('aws-xray-sdk-core');
const AWS = AWSXRay.captureAWS(require('aws-sdk')); // wrap SDK — auto-segments for all calls

// DynamoDB client — X-Ray subsegment created automatically for each call
const dynamo = new AWS.DynamoDB.DocumentClient();

exports.handler = async (event) => {
  // Add a custom annotation — filterable in X-Ray console
  const segment = AWSXRay.getSegment();
  const subsegment = segment.addNewSubsegment('processOrder');
  subsegment.addAnnotation('orderId', event.orderId); // indexed — queryable
  subsegment.addMetadata('event', event); // not indexed

  try {
    const result = await dynamo
      .get({
        // auto-traced by captureAWS
        TableName: 'orders',
        Key: { orderId: event.orderId }
      })
      .promise();

    subsegment.addAnnotation('userId', result.Item?.userId);
    subsegment.close();
    return result.Item;
  } catch (err) {
    subsegment.addError(err);
    subsegment.close();
    throw err;
  }
};
```

```yaml
# CDK — enable X-Ray on Lambda + API Gateway
const fn = new lambda.Function(this, 'OrderHandler', {
  tracing: lambda.Tracing.ACTIVE,  // enables X-Ray on Lambda
  ...
});

const api = new apigateway.RestApi(this, 'OrderApi', {
  deployOptions: {
    tracingEnabled: true,    // X-Ray on API Gateway
    dataTraceEnabled: true,  // log full request/response (expensive — dev only)
  }
});
```

- **Technical Terms to Include:** X-Ray trace, segment, subsegment, trace header (`X-Amzn-Trace-Id`), annotation, metadata, sampling rule, Service Map, `captureAWS`, `captureHTTPS`, `TracingConfig: Active`, cold start segment, CloudWatch ServiceLens, X-Ray Insights
- **Gotcha:** X-Ray **does not propagate automatically through SQS**. When Lambda A sends a message to SQS and Lambda B reads from it, the trace context is broken — X-Ray sees two separate traces. Workaround: embed the trace ID in the SQS message body or attributes manually, and read it in Lambda B to create a linked subsegment. Step Functions propagates X-Ray trace IDs automatically between states — SQS does not. This is a common "why are my traces incomplete?" complaint.
- **Follow-Up:** "How does X-Ray integrate with CloudWatch in a production observability setup?" → X-Ray traces feed into **CloudWatch ServiceLens** — a unified view combining X-Ray service maps, CloudWatch Logs, and CloudWatch Metrics in one console. A spike in Lambda error rate (metric) → click to ServiceLens → see the service map → identify which downstream service is failing → click trace → see full execution path with DynamoDB response times. This is the "three pillars connected" flow on AWS. **X-Ray Insights** automatically detects anomalies in trace data (sudden latency spikes, fault rate changes) without manual threshold configuration. → They're testing AWS observability integration depth.
- **Conclusion:** X-Ray is the distributed tracing layer that makes a Lambda chain debuggable — annotations enable correlation querying, the Service Map reveals bottlenecks visually, and sampling rules control cost; its integration into CloudWatch ServiceLens makes it the unified observability interface for production serverless systems.

---

### Serverless IaC — SAM vs CDK

- **Question:** What is the difference between AWS SAM and AWS CDK for serverless infrastructure, and when do you choose each?
- **Answer:**
  - **AWS SAM (Serverless Application Model):** A declarative YAML/JSON extension of CloudFormation with shorthand for serverless resources. `AWS::Serverless::Function` expands to a Lambda function + execution role + event source mappings in standard CloudFormation. `sam local invoke` runs Lambda locally. `sam local start-api` runs API Gateway locally. Limited to serverless resources — not the right tool for VPC, RDS, EKS, or complex cross-service infrastructure.
  - **AWS CDK (Cloud Development Kit):** TypeScript/Python/Java code that generates CloudFormation. Full AWS service coverage — every CloudFormation resource type is available as a CDK Construct. Higher-level constructs (L2) provide sensible defaults and best practices. Escape hatch to raw CloudFormation for anything the CDK doesn't yet have an L2 construct for. Type-safe — misconfigured constructs fail at `cdk synth` (compile time), not at deploy time.
  - **When SAM:** Simple serverless functions with API Gateway, SQS triggers, or DynamoDB streams. Teams already comfortable with YAML/CloudFormation. Local testing with `sam local`. Greenfield projects that are purely serverless.
  - **When CDK:** Any infrastructure beyond pure serverless (VPC, EKS, ECS, RDS alongside Lambda). Multi-stack applications with cross-stack references. Teams with TypeScript/Python expertise who want type-safety and IDE support. Complex infrastructure with loops, conditions, and reusable constructs. CDK can do everything SAM can — SAM is a subset.
  - **CDK for serverless:** `@aws-cdk/aws-lambda-nodejs` constructs bundle TypeScript Lambda functions with esbuild at synth time. `@aws-cdk/aws-apigatewayv2` for HTTP API. Full CDK Serverless stack matches SAM's DX with full CDK power.
- **Example:**

```yaml
# SAM — simple serverless API
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Runtime: nodejs20.x
    Timeout: 30
    Tracing: Active # X-Ray on all functions
    Environment:
      Variables:
        TABLE_NAME: !Ref OrdersTable

Resources:
  GetOrderFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: src/getOrder.handler
      Policies:
        - DynamoDBReadPolicy:
            TableName: !Ref OrdersTable
      Events:
        Api:
          Type: HttpApi
          Properties:
            Path: /orders/{id}
            Method: GET

  OrdersTable:
    Type: AWS::DynamoDB::Table
    Properties:
      BillingMode: PAY_PER_REQUEST
      KeySchema:
        - AttributeName: orderId
          KeyType: HASH
```

```typescript
// CDK — same stack, type-safe
import * as cdk from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda-nodejs';
import * as apigwv2 from 'aws-cdk-lib/aws-apigatewayv2';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';

const table = new dynamodb.Table(stack, 'Orders', {
  partitionKey: { name: 'orderId', type: dynamodb.AttributeType.STRING },
  billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
  removalPolicy: cdk.RemovalPolicy.RETAIN
});

const getOrderFn = new lambda.NodejsFunction(stack, 'GetOrder', {
  entry: 'src/getOrder.ts', // esbuild bundles TypeScript automatically
  tracing: lambda.Tracing.ACTIVE,
  environment: { TABLE_NAME: table.tableName }
});
table.grantReadData(getOrderFn); // generates least-privilege IAM policy

const api = new apigwv2.HttpApi(stack, 'OrderApi');
api.addRoutes({
  path: '/orders/{id}',
  methods: [apigwv2.HttpMethod.GET],
  integration: new HttpLambdaIntegration('GetOrderInt', getOrderFn)
});
```

- **Technical Terms to Include:** SAM, CDK, CloudFormation, `Transform: AWS::Serverless`, L1/L2/L3 constructs, `sam local`, esbuild bundler, `cdk synth`, `cdk deploy`, `grantReadData`, least-privilege IAM generation, escape hatch, cross-stack reference, CDK Aspects, CDK Nag
- **Gotcha:** CDK's `RemovalPolicy.DESTROY` on a DynamoDB table **deletes the table and all data** when `cdk destroy` is run. The default (`RETAIN`) keeps the table even after stack deletion. For production databases: always `RETAIN`. For ephemeral test environments: `DESTROY` is appropriate. This defaults-driven behaviour catches engineers who run `cdk destroy` to "clean up" a dev environment and accidentally delete a shared table.
- **Follow-Up:** "How do you manage secrets in a CDK stack?" → Never hardcode secrets in CDK code or CloudFormation parameters (they appear in CloudFormation console in plaintext). Pattern: (1) Store the secret in Secrets Manager or SSM Parameter Store. (2) In CDK, reference by ARN: `secretsmanager.Secret.fromSecretNameV2()`. (3) Grant the Lambda `secret.grantRead(fn)`. (4) Lambda reads the secret at runtime via SDK call. CDK Nag (`cdk-nag`) is a linting tool that flags security violations (including hardcoded secrets, missing encryption, public S3 buckets) during `cdk synth`. → They're testing serverless security posture in IaC.
- **Conclusion:** CDK is the correct IaC choice for anything beyond a single-file serverless function — its type-safety catches misconfiguration at synthesis time, `grantRead`/`grantWrite` generates least-privilege IAM automatically, and TypeScript constructs make complex multi-service infrastructure maintainable across team members.

---

### Serverless Failure Modes at Scale

- **Question:** What are the production failure modes specific to serverless architectures that don't exist in container-based systems?
- **Answer:**
  - **1 — Thundering herd on cold start:** A traffic spike brings thousands of concurrent requests simultaneously to a Lambda that was idle. Thousands of cold starts fire in parallel — each taking 200ms–2s. The result: a wave of elevated latency responses at the peak. For user-facing APIs: Provisioned Concurrency on peak-traffic functions prevents this.
  - **2 — Account-level concurrency exhaustion:** All Lambda functions share the 1,000 concurrent execution limit per region. A rogue function or traffic spike consuming 950 concurrent executions leaves only 50 for all other functions. Functions throttle with 429. Fix: Reserved Concurrency (cap per function), monitor `UnreservedConcurrentExecutions` metric, raise the account limit via support ticket.
  - **3 — DynamoDB hot partition under Lambda fan-out:** A Lambda triggered by SQS fan-out starts 500 concurrent executions, all writing to the same DynamoDB partition key. Writes exceed the 1,000 WCU/partition limit → `ProvisionedThroughputExceededException`. Fix: exponential backoff in the SDK (default), distribute writes across sharded partition keys, use DynamoDB on-demand mode.
  - **4 — Lambda timeout cascade:** Lambda A (30s timeout) calls Lambda B synchronously (30s timeout). Lambda B calls Lambda C (30s timeout). Total possible wait: 90s — but API Gateway cuts the connection at 29s. Lambda A receives a 504 from API Gateway and tries to handle it — while still running. Lambda B and C continue executing for up to 30s each with no caller. Fix: always set downstream Lambda timeouts shorter than upstream. Pass context with deadlines.
  - **5 — Silent event loss without DLQ:** Asynchronous Lambda invocations (SNS → Lambda, EventBridge → Lambda, S3 event → Lambda) that fail and exhaust retries are silently dropped if no DLQ or EventBridge Destination is configured. Fix: every async Lambda must have a failure destination (SQS DLQ or EventBridge Destination) configured. Alert on DLQ depth > 0.
  - **6 — Step Functions execution history limit:** Standard Workflows retain 90 days of execution history. A long-running workflow (weeks) with many state transitions can hit the 25,000 event history limit per execution. Fix: design long workflows as nested workflows — the parent calls child state machines, each with their own history. Compose rather than monolith.
- **Example:**

```
Production readiness checklist for a serverless Lambda:

✅ Provisioned Concurrency on user-facing functions (cold start mitigation)
✅ Reserved Concurrency set (protect account pool + cap runaway function)
✅ DLQ or EventBridge Destination on all async triggers
✅ X-Ray tracing enabled (Active)
✅ CloudWatch alarm: Errors > 0 for 5 minutes
✅ CloudWatch alarm: Throttles > 0
✅ CloudWatch alarm: DLQ ApproximateNumberOfMessagesVisible > 0
✅ CloudWatch alarm: ConcurrentExecutions > 80% of reserved limit
✅ DynamoDB on-demand (or WCU provisioned with auto-scaling) to handle burst
✅ Lambda timeout < API Gateway integration timeout (29s max)
✅ Downstream Lambda timeout < upstream Lambda timeout
✅ Idempotent handler (at-least-once Lambda invocation is expected)
```

- **Technical Terms to Include:** thundering herd, Provisioned Concurrency, account concurrency limit, Reserved Concurrency, hot partition, `ProvisionedThroughputExceededException`, cascade timeout, silent event loss, DLQ, EventBridge Destination, execution history limit, nested workflows, `UnreservedConcurrentExecutions`
- **Gotcha:** Lambda's **retry behaviour differs by invocation type**: Synchronous (API Gateway) — no automatic retry; the caller gets the error immediately. Asynchronous (SNS, S3, EventBridge) — Lambda retries 2 times with backoff automatically. Event source mapping (SQS, Kinesis, DynamoDB Stream) — Lambda keeps retrying until success or record expires; a poison message blocks the entire shard/queue. Each invocation type has different failure handling requirements — treat them as three distinct systems.
- **Follow-Up:** "How do you handle a poison message in an SQS-triggered Lambda?" → SQS with Lambda trigger: a message that always fails keeps being redelivered until visibility timeout + maxReceiveCount is exhausted → message goes to DLQ. Enable `ReportBatchItemFailures` in the Lambda response — return only the failing message IDs; successful messages in the batch are deleted, only the failing ones are retried. Without this, a single failure in a batch of 10 causes all 10 to be retried. → They're testing SQS partial batch failure handling (covered in AWS file, reinforced here).
- **Conclusion:** Serverless failure modes are architectural — they stem from the shared concurrency pool, at-least-once delivery semantics, asynchronous retry differences by invocation type, and the absence of persistent execution environments; the production readiness checklist captures the eight configuration decisions that separate a reliable serverless system from one that fails silently under load.

---

# SECTION 2 — AWS IoT Core

---

**Topics identified:**

- What is AWS IoT Core and where it fits in the IoT stack
- MQTT — protocol, QoS, topic hierarchy, wildcards, retained messages, LWT
- Device Shadow — desired/reported/delta model, offline operation
- IoT Rules Engine — SQL, actions, Basic Ingest
- Security model — X.509 certificates, IoT policies, least privilege
- IoT + Lambda/DynamoDB/Kinesis — telemetry pipeline architecture
- IoT Greengrass — edge computing, local processing
- Production patterns — fleet at scale, OTA firmware, telemetry ingestion

---

### What is AWS IoT Core / Where It Fits

- **Question:** What is AWS IoT Core, what problems does it solve that a plain MQTT broker doesn't, and how does it relate to your IIoT device management experience?
- **Answer:**
  - **AWS IoT Core** is a fully managed cloud service that acts as the connection hub for IoT devices. It provides: a managed MQTT broker (scales to billions of devices, no infrastructure management), device identity (X.509 certificate-based authentication), per-device and per-topic authorisation (IoT policies), device state management (Device Shadow), message routing to AWS services (Rules Engine), and device fleet management (Registry).
  - **What a plain MQTT broker (e.g., Mosquitto, ClearBlade) doesn't provide out-of-the-box:**
    - Managed scaling (you provision Mosquitto; IoT Core scales automatically)
    - X.509 certificate-based auth per device (you implement this yourself on Mosquitto)
    - Device Shadow (you build this state layer yourself)
    - Rules Engine routing to DynamoDB/Lambda/Kinesis/S3 (you implement your own subscriptions)
    - CloudWatch metrics and X-Ray integration (you add your own observability)
  - **Your production anchor:** At Volansys, ClearBlade served as the managed private MQTT broker + device management platform for HVAC and water heater devices — equivalent function to AWS IoT Core. The architectural concepts are identical: devices publish telemetry to topics, the platform routes messages to backend services, device shadows track state. AWS IoT Core is the AWS-native, fully managed equivalent.
  - **IoT Core stack position:** Devices → [MQTT over TLS] → IoT Core (broker + shadow + rules) → [Rules Engine actions] → Lambda / DynamoDB / Kinesis / S3 / SQS / SNS → downstream analytics, dashboards, alerts.
- **Example:**

```
IoT Core Architecture — HVAC device fleet:

Device Layer:
  HVAC units (embedded firmware) → MQTT client (AWS IoT Device SDK C)
  Topic: devices/{deviceId}/telemetry
  Message: { "temp": 22.5, "humidity": 65, "mode": "COOLING", "ts": 1234567890 }

IoT Core Layer:
  - Managed MQTT broker (scales automatically — no ops)
  - Device Registry: thing "hvac-001", attributes: { location: "floor-3", model: "XR200" }
  - Device Shadow: { desired: { targetTemp: 20 }, reported: { temp: 22.5 }, delta: { targetTemp: 20 } }
  - Rules Engine: route telemetry to Lambda + DynamoDB + CloudWatch

Backend Layer:
  Lambda: processes telemetry, triggers alerts if temp > threshold
  DynamoDB: stores time-series readings (partition: deviceId, sort: timestamp)
  CloudWatch: custom metric for device health
  SNS: alert to on-call when device temp exceeds safety limit
```

- **Technical Terms to Include:** AWS IoT Core, Thing, Device Registry, MQTT broker, Device Shadow, Rules Engine, X.509 certificate, IoT Policy, AWS IoT Device SDK, ClearBlade (comparison), Mosquitto (comparison), managed scaling, telemetry, fleet management
- **Gotcha:** IoT Core **is not a message store**. Messages published to IoT Core topics are in-flight only — if no subscriber is active and no Rules Engine rule matches, the message is dropped. Unlike SQS (which queues messages), MQTT Pub/Sub has no persistence. If you need durability: use the Rules Engine to write every message to a Kinesis Data Stream (acts as a durable log) or DynamoDB. Design the data pipeline assuming messages are ephemeral at the broker level.
- **Follow-Up:** "How does AWS IoT Core compare to ClearBlade for IIoT workloads?" → ClearBlade: private/on-premise deployment option, edge-first architecture (code runs locally on the ClearBlade edge), good for air-gapped industrial environments. IoT Core: fully managed, cloud-native, deep AWS service integration, globally distributed (endpoint per region). For workloads requiring cloud-native AWS integration (Lambda triggers, DynamoDB writes, S3 archiving) — IoT Core. For edge-first, offline-capable industrial environments — ClearBlade or IoT Core + Greengrass. Both use MQTT as the device protocol; the management plane and integration model differ. → They're testing direct comparison from your experience.
- **Conclusion:** AWS IoT Core turns the MQTT broker from a self-managed infrastructure concern into a fully managed AWS service — its value compounds at fleet scale (thousands of devices → no broker tuning needed) and its Rules Engine eliminates the custom Lambda subscribers that a plain MQTT broker requires for every routing use case.

---

### MQTT — Protocol Depth

- **Question:** How does MQTT work, what are QoS levels, and what are wildcards, retained messages, and Last Will and Testament?
- **Answer:**
  - **MQTT (Message Queuing Telemetry Transport):** A lightweight publish/subscribe protocol designed for constrained devices (low power, low bandwidth, unreliable networks). A device (client) connects to a broker. Publishes messages to **topics** (hierarchical strings: `devices/hvac-001/telemetry`). Other clients subscribe to topics using exact match or wildcard patterns. The broker routes messages to all current subscribers. Connection is persistent (long-lived TCP) — reduces reconnect overhead.
  - **QoS Levels:**
    - **QoS 0 (At most once):** Fire and forget. Broker delivers or drops. Zero overhead. Use for: high-frequency telemetry where occasional loss is acceptable (temperature readings every second — one drop is fine).
    - **QoS 1 (At least once):** Broker ACKs the publish. Publisher retries until ACK received. Consumer may receive duplicates. Use for: commands where delivery matters but idempotent handling is possible. Most common in production IoT.
    - **QoS 2 (Exactly once):** 4-way handshake guarantees exactly-once delivery. Highest overhead. Use for: critical commands where duplicate execution is dangerous (actuator control, payment processing). AWS IoT Core supports QoS 0 and 1 (not 2).
  - **Topic wildcards:**
    - `+` — single-level wildcard: `devices/+/telemetry` matches `devices/hvac-001/telemetry` and `devices/hvac-002/telemetry` but not `devices/floor-3/hvac-001/telemetry`.
    - `#` — multi-level wildcard: `devices/#` matches all topics under `devices/` at any depth.
  - **Retained messages:** The last message published to a topic is stored by the broker. New subscribers receive the retained message immediately on subscription — without waiting for the next publish. Use for: device current state that new dashboard connections should see immediately.
  - **Last Will and Testament (LWT):** A message the broker publishes on a client's behalf if the client disconnects unexpectedly (network failure, crash). The client registers the LWT at connection time. Use for: publishing a "device offline" event to a `devices/{id}/status` topic when a device drops without a clean disconnect.
- **Example:**

```python
# Python MQTT client — device publishing telemetry to IoT Core
import awsiot.mqtt_connection_builder as mqtt_builder
import json, time

def on_disconnect(disconnect_future):
    print("Disconnected:", disconnect_future.exception())

# Connect with X.509 certificate (no username/password — certificate is the identity)
mqtt_connection = mqtt_builder.mtls_from_path(
    endpoint="xxxxxx.iot.us-east-1.amazonaws.com",
    cert_filepath="/certs/device-cert.pem",
    pri_key_filepath="/certs/device-key.pem",
    ca_filepath="/certs/amazon-root-ca.pem",
    client_id="hvac-001",
    # Last Will and Testament — broker publishes this if device disconnects unexpectedly
    will=mqtt5.PublishPacket(
        topic="devices/hvac-001/status",
        payload=json.dumps({ "connected": False, "reason": "unexpected_disconnect" }),
        qos=mqtt5.QoS.AT_LEAST_ONCE,
        retain=True,   # retained — next subscriber sees it immediately
    )
)
mqtt_connection.connect().result()

# Publish telemetry — QoS 1 (at-least-once)
while True:
    reading = get_sensor_reading()
    mqtt_connection.publish(
        topic="devices/hvac-001/telemetry",
        payload=json.dumps(reading),
        qos=mqtt5.QoS.AT_LEAST_ONCE,
    )
    time.sleep(60)  # publish every minute

# Subscribe to commands from cloud (desired state changes)
def on_command(topic, payload, **kwargs):
    command = json.loads(payload)
    if command.get("targetTemp"):
        set_target_temperature(command["targetTemp"])

mqtt_connection.subscribe(
    topic="devices/hvac-001/commands",
    qos=mqtt5.QoS.AT_LEAST_ONCE,
    callback=on_command,
)
```

- **Technical Terms to Include:** MQTT, broker, topic, publish, subscribe, QoS 0/1/2, retained message, LWT (Last Will and Testament), wildcard (`+`, `#`), MQTT over TLS, MQTT 5 (user properties, request-response), persistent session, clean session, keep-alive, AWS IoT Device SDK, ALPN (MQTT over port 443)
- **Gotcha:** AWS IoT Core enforces **topic-level authorisation in IoT Policies**. If a device's IoT Policy allows `devices/${iot:ClientId}/telemetry` only, that device cannot publish to `devices/other-device/telemetry`. This is per-device least privilege. A common misconfiguration: using `*` as the topic in the IoT Policy, allowing any device to publish to any topic — any compromised device can spoof messages from any other device. Always scope policies to the specific topics a device legitimately owns.
- **Follow-Up:** "How does MQTT QoS interact with AWS IoT Core's Rules Engine?" → The Rules Engine processes messages regardless of the QoS level the publisher used. However, if the Rules Engine action (e.g., Lambda invocation) fails, retries depend on the action type — not MQTT QoS. For guaranteed processing, design the action target as idempotent and add an error action (route failures to SQS or CloudWatch Logs). QoS 1 guarantees the broker received the message; it doesn't guarantee the downstream Rules Engine action succeeded. → They're testing end-to-end delivery guarantee understanding.
- **Conclusion:** MQTT's lightweight protocol design (persistent connections, QoS tiers, retained messages, LWT) is specifically optimised for the constraints of IoT devices — understanding which QoS level suits each message type and how retained messages and LWT enable reliable device status tracking is the foundation for any IoT system design.

---

### Device Shadow — Offline State Synchronisation

- **Question:** What is the AWS IoT Device Shadow, how does the desired/reported/delta model work, and what are the production patterns for command-and-control architectures?
- **Answer:**
  - The **Device Shadow** is a JSON document stored in AWS IoT Core that represents the **last known and desired state** of a device. It decouples cloud applications from the device's connectivity status — applications can read and write the shadow whether the device is connected or not.
  - **Three-field model:**
    - **`reported`:** The device publishes its current actual state here. "My temperature is 22.5°C, my mode is COOLING."
    - **`desired`:** A cloud application (dashboard, automation rule) writes the desired target state here. "I want this device to be in HEATING mode with targetTemp 18°C."
    - **`delta`:** IoT Core automatically computes the difference between `desired` and `reported`. If the device is connected, it receives the delta on a reserved topic and acts on it. The delta tells the device: "here is what changed between what the app wants and what you last reported."
  - **Offline operation:** If the device is offline when the app writes to `desired`, the delta persists. When the device reconnects, it subscribes to `$aws/things/{thingName}/shadow/update/delta` and immediately receives the pending delta — no missed command.
  - **Named shadows:** Multiple shadows per device (supported since 2020). `$aws/things/hvac-001/shadow/name/climate` and `$aws/things/hvac-001/shadow/name/maintenance` — separate concerns in separate shadows.
  - **Command-and-control pattern:** App writes `desired.targetTemp = 18` → IoT Core computes delta → Device receives delta → Device sets temperature → Device publishes `reported.temp = 18` → IoT Core clears the delta. The delta is the signal; the clearing of the delta is the confirmation.
- **Example:**

```json
// Shadow document for hvac-001
{
  "state": {
    "desired": {
      "targetTemp": 18,
      "mode": "HEATING",
      "fanSpeed": "LOW"
    },
    "reported": {
      "temp": 22.5,
      "targetTemp": 22,    // device hasn't reached the target yet
      "mode": "HEATING",   // mode already changed
      "fanSpeed": "LOW",
      "connected": true
    },
    "delta": {
      "targetTemp": 18     // only the difference — device needs to reach 18
    }
  },
  "metadata": { ... },
  "version": 42,
  "timestamp": 1234567890
}

// MQTT topics (device subscribes to):
$aws/things/hvac-001/shadow/update/delta      // receive desired changes
$aws/things/hvac-001/shadow/update/accepted   // confirm shadow update accepted
$aws/things/hvac-001/shadow/update/rejected   // shadow update rejected (version conflict)

// Device receives delta — applies it — publishes new reported state:
{
  "state": {
    "reported": {
      "temp": 18.1,
      "targetTemp": 18    // matched — delta will be cleared
    }
  }
}
```

```python
# Cloud app — set desired state (works whether device is on or offline)
import boto3
client = boto3.client('iot-data', region_name='us-east-1')

response = client.update_thing_shadow(
    thingName='hvac-001',
    payload=json.dumps({
        "state": {
            "desired": {
                "targetTemp": 18,
                "mode": "HEATING"
            }
        }
    })
)
# If device is offline: delta persists until device reconnects
# If device is online: delta delivered immediately via MQTT
```

- **Technical Terms to Include:** Device Shadow, desired, reported, delta, shadow document, `$aws/things/{thingName}/shadow/`, named shadow, shadow update, version conflict, conflict resolution, offline operation, command-and-control pattern, `iot-data` client, shadow REST API
- **Gotcha:** Shadow **version conflicts** — every shadow update increments the `version` number. If the device and cloud app both write to the shadow concurrently, the second write may fail with `409 Conflict` (version mismatch). Devices should always handle the `rejected` topic and reapply their state update after fetching the current version. Design device shadow updates as idempotent — applying the same desired state twice should have no adverse effect.
- **Follow-Up:** "How do you detect device connectivity in real time using shadows and LWT?" → Register an LWT message at MQTT connection time that publishes `{ "state": { "reported": { "connected": false } } }` to a shadow update topic. When the device connects, it publishes `connected: true` to its shadow. When it disconnects unexpectedly, the broker publishes the LWT, setting `connected: false` in the shadow. A Rules Engine rule on `$aws/things/+/shadow/update/documents` can trigger a Lambda when `connected` changes — enabling real-time device offline/online alerting without polling. → They're testing shadow + LWT integration for fleet monitoring.
- **Conclusion:** The Device Shadow's desired/reported/delta model enables robust command-and-control for intermittently connected devices — cloud applications write intents to `desired` without caring about device connectivity, and devices apply those intents at their own pace, with the delta persisting through disconnections as a durable pending command queue.

---

### IoT Rules Engine — Telemetry Routing

- **Question:** How does the IoT Rules Engine work, what is Basic Ingest, and how do you design a telemetry pipeline from device to analytics?
- **Answer:**
  - The **Rules Engine** evaluates an SQL-like query against every message published to matching IoT topics and routes matching messages to one or more AWS service targets — all without writing a Lambda to subscribe and forward.
  - **Rule SQL:** `SELECT temperature, humidity, clientId() as deviceId, timestamp() as ts FROM 'devices/+/telemetry' WHERE temperature > 0` — the `FROM` clause is the topic filter (supports wildcards), `WHERE` filters messages, `SELECT` shapes the output payload passed to actions.
  - **Actions (18+ types):** Lambda (custom processing), DynamoDB (direct write — no Lambda), Kinesis Data Streams (high-throughput buffering), Kinesis Firehose (direct to S3/Redshift), SQS (queue for processing), SNS (fan-out notification), S3 (archive raw telemetry), CloudWatch Logs/Metrics (observability), Elasticsearch/OpenSearch (search index), IoT Events (state machine for device anomaly detection), Republish (re-publish on another MQTT topic).
  - **Error action:** Each rule has an optional error action — fires when the primary action fails (Lambda throws, DynamoDB write errors). Route errors to SQS or CloudWatch Logs.
  - **Basic Ingest:** A cost-optimised feature for rules that don't need delivery to other MQTT subscribers. Publish directly to `$aws/rules/{ruleName}` instead of a regular topic — the message is processed by the Rules Engine only, skipping MQTT broker delivery. No messaging charge for Basic Ingest messages (only the Rules Engine execution cost). Use when devices send telemetry purely for backend processing with no other device subscribing.
- **Example:**

```sql
-- Rule 1: Archive all telemetry to S3 (cold storage)
SELECT * FROM 'devices/+/telemetry'
-- Action: Kinesis Firehose → S3 (partitioned by device, date)
-- Delivers: compressed JSON batches to s3://telemetry-archive/year=2026/month=03/

-- Rule 2: Alert on temperature threshold breach
SELECT clientId() as deviceId, temperature, timestamp() as ts
FROM 'devices/+/telemetry'
WHERE temperature > 35 OR temperature < 5
-- Action: Lambda → checks history, deduplicates alerts, sends SNS notification
-- Error action: SQS → DLQ for failed alert processing

-- Rule 3: Real-time indexing for dashboard (last hour of data)
SELECT temperature, humidity, clientId() as deviceId, timestamp() as ts
FROM 'devices/+/telemetry'
-- Action: OpenSearch (Elasticsearch) with index rotation by day
-- Dashboard queries OpenSearch for fast time-series queries

-- Rule 4: High-frequency write to DynamoDB (direct — no Lambda)
SELECT temperature, humidity, clientId() as deviceId, timestamp() as ts
FROM 'devices/+/telemetry'
-- Action: DynamoDB PutItem
--   partition: ${clientId()}, sort: ${timestamp()}
-- No Lambda cold start — DynamoDB write happens directly from Rules Engine

-- Basic Ingest — device publishes to $aws/rules/TelemetryRule (not a topic)
-- No MQTT broker delivery cost, only rules engine execution cost
```

- **Technical Terms to Include:** Rules Engine, SQL rule, topic filter, `clientId()`, `timestamp()`, `topic()`, action, error action, Lambda action, DynamoDB action, Kinesis action, Firehose action, Basic Ingest, `$aws/rules/{ruleName}`, substitution template, rule IAM role, IoT Events
- **Gotcha:** The Rules Engine **IAM role** is separate from Lambda execution roles. Each Rules Engine rule needs a role that grants it permission to write to the target (DynamoDB, Kinesis, Lambda invoke, S3 PutObject). A common mistake: granting the Rules Engine role `AdministratorAccess` for convenience. This means any device message that triggers a rule execution has write access to all AWS resources. Scope the Rules Engine role to the minimum required actions on the specific resources.
- **Follow-Up:** "How do you handle telemetry from 100,000 devices at 1 message per minute?" → 100K messages/minute = ~1,667/second. IoT Core scales automatically. Rules Engine processes in parallel. DynamoDB (on-demand) handles the write spike. The bottleneck is typically: (1) Lambda concurrency if using Lambda actions (1,000 concurrent default) — use Kinesis or DynamoDB direct action instead to bypass Lambda. (2) DynamoDB hot partitions (if all 100K devices write to the same partition key — e.g., a date-based key). Partition by `deviceId` as the sort key for time-series, not by date. → They're testing IoT at-scale data pipeline design.
- **Conclusion:** The Rules Engine is IoT Core's integration fabric — routing device telemetry to 18+ AWS targets with SQL-based filtering eliminates the custom Lambda subscriber pattern, and Basic Ingest removes the messaging cost for high-volume backend-only telemetry pipelines, making it the primary tool for designing IoT data flows without infrastructure management.

---

### IoT Security Model & Greengrass

- **Question:** How does AWS IoT Core's security model work, and what is IoT Greengrass for edge deployments?
- **Answer:**
  - **Security model — X.509 certificates:** Every IoT device is authenticated by an X.509 certificate signed by a CA (AWS IoT's CA or your own CA). No username/password. The certificate is provisioned at manufacturing time or through fleet provisioning (a bootstrap certificate exchanges for a device-specific certificate on first connection). Certificates can be revoked without touching the device — simply revoke in IoT Core and the device can no longer connect.
  - **IoT Policies:** JSON policies attached to certificates (like IAM policies, but for IoT). Define which MQTT operations (Connect, Publish, Subscribe, Receive) are allowed on which topics. Use `${iot:ClientId}` as a variable — a device can only publish to its own topics: `"Resource": "arn:aws:iot:*:*:topic/devices/${iot:ClientId}/*"`. This is per-device least privilege.
  - **Just-in-time provisioning (JITP):** When a device with a CA-signed certificate connects for the first time, IoT Core creates a Thing, registers the certificate, attaches the policy, and activates it automatically — zero pre-provisioning. Essential for large fleet rollouts where pre-registering every device individually is impractical.
  - **IoT Greengrass:** An AWS service that extends IoT Core capabilities to edge devices (local servers, gateways). Greengrass runs locally — Lambda functions, ML models, Docker containers, and custom components execute on the edge device itself. Benefits: (1) Local processing without cloud round-trip (sub-10ms vs 50–200ms cloud). (2) Offline operation — Greengrass continues processing when internet is unavailable. (3) Selective sync — only processed results go to the cloud, not raw telemetry (reduces data transfer cost and bandwidth). (4) Local ML inference — run TensorFlow/PyTorch models on edge for anomaly detection without sending raw sensor data to cloud.
- **Example:**

```json
// IoT Policy — per-device least privilege
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iot:Connect",
      "Resource": "arn:aws:iot:us-east-1:123456789012:client/${iot:ClientId}"
      // Device can only connect with its own client ID
    },
    {
      "Effect": "Allow",
      "Action": "iot:Publish",
      "Resource": ["arn:aws:iot:us-east-1:123456789012:topic/devices/${iot:ClientId}/telemetry", "arn:aws:iot:us-east-1:123456789012:topic/$aws/things/${iot:ClientId}/shadow/update"]
      // Can only publish to ITS OWN topics — not other devices' topics
    },
    {
      "Effect": "Allow",
      "Action": "iot:Subscribe",
      "Resource": ["arn:aws:iot:us-east-1:123456789012:topicfilter/devices/${iot:ClientId}/commands", "arn:aws:iot:us-east-1:123456789012:topicfilter/$aws/things/${iot:ClientId}/shadow/*"]
    },
    {
      "Effect": "Allow",
      "Action": "iot:Receive",
      "Resource": "arn:aws:iot:us-east-1:123456789012:topic/devices/${iot:ClientId}/*"
    }
  ]
}
```

- **Technical Terms to Include:** X.509 certificate, certificate CA, IoT Policy, `iot:ClientId`, `iot:Connect`, `iot:Publish`, `iot:Subscribe`, JITP (Just-in-time provisioning), fleet provisioning, certificate revocation, Greengrass, edge computing, local Lambda, offline operation, selective sync, OTA (over-the-air) firmware update
- **Gotcha:** IoT Policies require **both `iot:Subscribe` and `iot:Receive`** for a device to receive messages on a topic. `Subscribe` allows the subscription; `Receive` allows the actual message delivery. Forgetting `Receive` results in subscriptions that appear to succeed but messages are silently dropped. This is a common "why isn't my device receiving commands?" debugging mystery.
- **Follow-Up:** "How do you push OTA firmware updates to a fleet of IoT devices using AWS?" → AWS IoT Jobs service: create a Job with the firmware S3 URL as the target document, select a Thing Group (all devices of a given model). IoT Core delivers the job to each device. The device's firmware agent subscribes to `$aws/things/{thingName}/jobs/notify`, downloads the firmware from the pre-signed S3 URL, verifies the hash, applies the update, and reports `SUCCEEDED` or `FAILED` back to the Job. The Job tracks completion status per device. Rollout rate control (update N devices per hour) and abort criteria (abort if >10% failure rate) prevent fleet-wide outage from a bad firmware release. → They're testing production fleet OTA architecture.
- **Conclusion:** AWS IoT Core's security model — X.509 device identity, per-device IoT Policies with `${iot:ClientId}` scoping, and JITP for large fleet onboarding — provides zero-trust device authentication at scale without password management; Greengrass extends this model to the edge for offline-capable, low-latency local processing use cases that your IIoT background demonstrates directly.

---

## Quick Reference — Decision Logic

**Serverless:**

| Decision                            | Answer                                                                                  |
| ----------------------------------- | --------------------------------------------------------------------------------------- |
| Step Functions vs SQS chain         | Step Functions for >2 steps, compensations needed, state visibility required            |
| Express vs Standard Workflow        | Standard for audit trail, >5min, business processes. Express for high-throughput, <5min |
| EventBridge vs SNS                  | EventBridge for content-based routing, multi-target, AWS service events, cross-account  |
| SAM vs CDK                          | SAM for simple serverless-only. CDK for anything else, type-safety, multi-service       |
| Provisioned vs reserved concurrency | Provisioned = warm (no cold start). Reserved = cap (protect account pool)               |
| DLQ required?                       | Always on async Lambda, SQS consumers, EventBridge targets                              |

**IoT Core:**

| Decision                          | Answer                                                                            |
| --------------------------------- | --------------------------------------------------------------------------------- |
| QoS 0 vs 1                        | QoS 0 for high-frequency telemetry. QoS 1 for commands and critical data          |
| Shadow vs direct MQTT publish     | Shadow for state/command. Direct publish for ephemeral events (telemetry)         |
| Rules Engine vs Lambda subscriber | Rules Engine for standard routing. Lambda when transformation logic needed        |
| Basic Ingest vs regular topic     | Basic Ingest when no other device needs to receive the message (reduces cost)     |
| Greengrass vs cloud-only          | Greengrass when: latency <50ms required, offline operation needed, high bandwidth |
| JITP vs pre-provisioning          | JITP for large fleet rollout. Pre-provisioning for small, pre-known device sets   |

---

_— End of AWS_Serverless_IoT_Interview_QA.md —_
_Section 1: Serverless — Step Functions · EventBridge · X-Ray · SAM/CDK · Failure Modes (5 QA)_
_Section 2: IoT Core — Architecture · MQTT · Device Shadow · Rules Engine · Security · Greengrass (5 QA)_
_Total: 10 QA units | Calibrated: Tech Lead / Architect level_
_IoT anchored to Volansys IIoT production experience (ClearBlade/MQTT → AWS IoT Core)_
