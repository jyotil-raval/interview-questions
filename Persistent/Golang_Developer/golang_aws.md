<!-- Generated: 2026-04-12 | MH: Golang,AWS | GTH: none | Depth: L1-L6 | Combination: Cross-domain -->

# Persistent Systems — Golang Developer

## Golang + AWS Combination Interview Prep

> **Relationship Type:** Cross-domain — Go is the application layer; AWS is the infrastructure and managed services layer.
> **Interviewer Lens:** "You know Go and you know AWS individually — do you know how to build production Go services _on_ AWS? Show me the seams."

---

## How These Technologies Relate Architecturally

Go's explicit concurrency model, context-first design, and single binary deployment make it a natural fit for AWS's serverless and container-based runtimes. The pairing surfaces at three layers:

1. **Execution:** Go Lambda (event-driven), ECS Fargate / EKS (containerised), EC2 (long-running)
2. **Integration:** AWS SDK for Go v2 provides idiomatic Go clients (context-first, modular) for every AWS service
3. **Operational model:** Go's observability tools (pprof, structured logging, Prometheus) integrate with AWS's observability stack (CloudWatch, X-Ray)

The architectural tension: AWS managed services solve infrastructure problems; Go's design solves application concurrency problems. The interview probes whether you understand _which_ problem belongs to which layer.

---

## Common Pitfalls Specific to this Role at Persistent

1. **Lambda connection pool exhaustion:** spawning a new DB connection per Lambda invocation at scale
2. **Context not propagated through AWS SDK calls:** losing deadline enforcement in downstream service calls
3. **Goroutine leak in SQS consumer:** worker goroutines not terminated when context is cancelled
4. **IAM over-permissioning:** Go services with `*:*` execution roles because "it's easier in dev"
5. **Single-AZ Go deployment called production:** ECS tasks in one AZ with no ALB cross-zone
6. **SDK v1 vs v2 confusion:** mixing `github.com/aws/aws-sdk-go` (v1) and `github.com/aws/aws-sdk-go-v2` in the same Go module

---

## Q&A — Golang + AWS Interaction

---

### Category: Go on Lambda

> 📁 2 topics · 2 questions

#### Topic: Go Lambda Architecture and Handler Design — (Q count: 1)

- **[C.GoOnLambda.GoLambdaArchitectureAndHandlerDesign.Q1]**
  **Question:** Design a production Go Lambda function that consumes SQS messages, calls a downstream HTTP API, and writes results to DynamoDB — handle partial batch failures.
- **Answer:**
  - The Go Lambda handler receives `events.SQSEvent` containing a batch of up to 10 messages; each message must be processed idempotently — SQS at-least-once delivery means duplicates are guaranteed over time.
  - Partial batch failure: instead of returning a single error (which nacks the entire batch), return `events.SQSEventResponse` with `BatchItemFailures` — only failed message IDs are returned to the queue; successful messages are deleted.
  - Use a `context.WithTimeout` derived from the Lambda context for the downstream HTTP call — Lambda passes a deadline via context; if the function is approaching timeout, `ctx.Done()` fires and the HTTP call should be abandoned.
  - Initialise the DynamoDB client and HTTP client **outside** the handler (module-level) — Lambda reuses the execution environment; clients are warm and connection-pooled on subsequent invocations.
  - Structured logging per message: include `messageID` and `correlationID` in every log line — `log/slog.With("messageID", msg.MessageId)` creates a child logger with the field injected.
- **Example:**

```go
var (
    dbClient   *dynamodb.Client
    httpClient = &http.Client{Timeout: 5 * time.Second}
)

func init() {
    cfg, _ := config.LoadDefaultConfig(context.Background())
    dbClient = dynamodb.NewFromConfig(cfg)
}

func handler(ctx context.Context, event events.SQSEvent) (events.SQSEventResponse, error) {
    var failures []events.SQSBatchItemFailure
    for _, msg := range event.Records {
        if err := process(ctx, msg); err != nil {
            failures = append(failures, events.SQSBatchItemFailure{
                ItemIdentifier: msg.MessageId,
            })
        }
    }
    return events.SQSEventResponse{BatchItemFailures: failures}, nil
}

func main() { lambda.Start(handler) }
```

- **Technical Terms to Include:** `events.SQSEvent`, `events.SQSBatchItemFailure`, `events.SQSEventResponse`, partial batch failure, `init()` initialisation, execution environment reuse, `context.WithTimeout`, `log/slog.With`
- **Gotcha:** "Returning a non-nil error from the top-level handler nacks the entire batch" — even if 9 out of 10 messages succeeded. Returning `SQSEventResponse` with per-message failures is the correct model; the top-level error should only be returned for unrecoverable infrastructure failures.
- **Follow-Up:** "How do you make the process function idempotent?" → Conditional write to DynamoDB using `ConditionExpression: "attribute_not_exists(messageID)"` — if the item already exists (duplicate delivery), the write fails with `ConditionalCheckFailedException`, which is treated as success. Asked because idempotency is the architectural requirement for at-least-once systems.
- **Conclusion:** A production Go Lambda on SQS requires three disciplines: partial batch failure handling, idempotent processing, and execution environment-aware initialisation — missing any one causes silent data loss or duplicate processing at scale.

---

#### Topic: Go Lambda Cold Start Optimisation — (Q count: 1)

- **[C.GoOnLambda.GoLambdaColdStartOptimisation.Q1]**
  **Question:** A Go Lambda serving a latency-sensitive API has P99 cold start of 800ms. Walk through the investigation and optimisation steps.
- **Answer:**
  - **Measure the cold start components:** Lambda `INIT_DURATION` in CloudWatch Logs (cold start total) vs `DURATION` (handler execution). `INIT_DURATION` > 200ms for a Go function indicates heavy `init()` or global variable initialisation.
  - **Binary size:** larger Go binaries take longer to load from the runtime image — `go build -ldflags="-s -w"` strips debug symbols and DWARF data, reducing binary size 30–40%; use `upx` compression as a last resort (decompression adds latency).
  - **Initialisation work:** defer expensive initialisation (DB connection establishment, secret fetching) to the first handler invocation with `sync.Once` — if the Lambda is killed before the first request (provisioned concurrency keep-alive), expensive init doesn't happen.
  - **Provisioned Concurrency:** pre-warms N execution environments — `INIT_DURATION` is paid at provision time, not on the request. Appropriate for P99 SLAs below 200ms where cold start is unacceptable.
  - **Lambda SnapStart (Preview for Go):** snapshots the initialised execution environment; subsequent cold starts restore from snapshot rather than re-initialising — reduces cold start to ~100ms regardless of init work.
- **Example:** A Go Lambda at Persistent initialising a gRPC connection pool to a downstream service added 600ms `INIT_DURATION`. Fix: deferred gRPC dial to first invocation with `sync.Once` + Provisioned Concurrency of 5 for the high-traffic hours — P99 cold start dropped to 120ms.
- **Technical Terms to Include:** `INIT_DURATION`, `DURATION`, `-ldflags="-s -w"`, `upx`, `sync.Once`, Provisioned Concurrency, SnapStart, binary size, `init()` weight
- **Gotcha:** "Provisioned Concurrency eliminates cold starts completely" — it eliminates cold starts for the provisioned count only. A traffic spike beyond provisioned count spawns new cold environments. Auto-scaling Provisioned Concurrency (via Application Auto Scaling) is required for variable traffic.
- **Follow-Up:** "Does ARM64 (Graviton) help with cold start?" → Slightly — ARM64 binaries are ~10% smaller than x86_64 equivalents for Go, and Graviton has better single-thread memory bandwidth. The primary benefit is 20% cost reduction; cold start improvement is secondary.
- **Conclusion:** Go Lambda cold start optimisation is a binary size + init work problem — `INIT_DURATION` is the metric, and the solution is always: do less work before the handler, or pay for Provisioned Concurrency.

---

### Category: AWS SDK for Go v2 Patterns

> 📁 1 topic · 1 question

#### Topic: Context Propagation through AWS SDK Calls — (Q count: 1)

- **[C.AWSSDKForGoV2Patterns.ContextPropagationThroughAWSSDKCalls.Q1]**
  **Question:** Why is context propagation critical when using the AWS SDK for Go v2, and what breaks if you pass `context.Background()` instead of the request context?
- **Answer:**
  - AWS SDK for Go v2 accepts `context.Context` as the first argument to every API call — this context carries cancellation, deadline, and trace metadata (X-Ray trace ID via the context).
  - If a Go Lambda's context carries a 29-second deadline from API Gateway, and you pass `context.Background()` to `dynamodb.GetItem`, the DB call has no deadline — it can run past the Lambda timeout, leaving a goroutine orphaned after the Lambda freezes the execution environment.
  - AWS X-Ray propagation: `xray.AWSv2` middleware on the SDK config automatically extracts the trace ID from the context and injects it into the outgoing AWS API call — passing `context.Background()` loses the trace, breaking distributed trace chains in X-Ray.
  - Cancellation: if the upstream caller cancels the request (user closes browser, API Gateway timeout), the context `Done()` channel fires — SDK calls using the request context automatically cancel in-flight HTTP calls to AWS services, releasing connections immediately.
  - Pattern: always thread the request context through the entire call chain — handler context → business logic → repository layer → AWS SDK call. Never create a new `context.Background()` inside a handler.
- **Example:**

```go
// WRONG: loses deadline, loses X-Ray trace
result, err := dbClient.GetItem(context.Background(), input)

// CORRECT: deadline enforced, X-Ray trace propagated
result, err := dbClient.GetItem(ctx, input)

// CORRECT with explicit timeout on top of request deadline:
callCtx, cancel := context.WithTimeout(ctx, 200*time.Millisecond)
defer cancel()
result, err := dbClient.GetItem(callCtx, input)
```

- **Technical Terms to Include:** `context.Context`, SDK v2 context-first, deadline enforcement, X-Ray trace propagation, `xray.AWSv2` middleware, `context.Background()` anti-pattern, connection release on cancellation
- **Gotcha:** "Adding a `context.WithTimeout` inside a handler that already has a deadline adds no safety if the timeout is longer than the remaining deadline" — `context.WithDeadline` and `context.WithTimeout` use the _earlier_ of the parent deadline and the specified timeout. A 500ms timeout added to a context with 100ms remaining still fires at 100ms.
- **Follow-Up:** "How do you configure AWS SDK retry behaviour for a Go service?" → `aws.Config` with a custom `retry.AddWithMaxAttempts(retry.NewStandard(), 3)` retryer — SDK v2's adaptive retry mode backs off on throttling responses automatically. Asked because retry configuration is a resilience design decision.
- **Conclusion:** The AWS SDK v2's context-first design is the correct abstraction precisely because it forces Go engineers to propagate deadlines and cancellation to every external call — violating this discipline at any layer creates resource leaks and broken traces.

---

### Category: Go + DynamoDB Patterns

> 📁 1 topic · 1 question

#### Topic: DynamoDB Transactions from Go — (Q count: 1)

- **[C.Go+DynamoDBPatterns.DynamoDBTransactionsFromGo.Q1]**
  **Question:** When do you use DynamoDB transactions in a Go service, and what are the cost and performance implications?
- **Answer:**
  - DynamoDB `TransactWriteItems` atomically writes up to 100 items across up to 100 tables — either all writes succeed or all fail, with ACID guarantees at DynamoDB scale.
  - Use case: a Go service transferring balance between two accounts needs both the debit and credit to succeed or both to fail — a `TransactWriteItems` with both updates handles this atomically without distributed transaction coordination.
  - Cost: transactions consume 2x the read/write capacity units vs non-transactional operations — a transaction writing 2 items consumes write capacity as if 4 items were written.
  - Go implementation: `dynamodbattribute.MarshalMap` (v1) or `attributevalue.MarshalMap` (v2) marshals Go structs to DynamoDB `AttributeValue` maps; the SDK handles the transaction request format.
  - Idempotency: include a `ClientRequestToken` (up to 10 characters) in `TransactWriteItems` — DynamoDB deduplicates transactions with the same token within 10 minutes, safe for at-least-once Lambda invocations.
- **Example:**

```go
_, err = dbClient.TransactWriteItems(ctx, &dynamodb.TransactWriteItemsInput{
    ClientRequestToken: aws.String(idempotencyKey),
    TransactItems: []types.TransactWriteItem{
        {Update: &types.Update{
            TableName: aws.String("Accounts"),
            Key:       fromKey,
            UpdateExpression: aws.String("SET balance = balance - :amount"),
            ConditionExpression: aws.String("balance >= :amount"),
            ExpressionAttributeValues: map[string]types.AttributeValue{
                ":amount": &types.AttributeValueMemberN{Value: "100"},
            },
        }},
        {Update: &types.Update{
            TableName: aws.String("Accounts"),
            Key:       toKey,
            UpdateExpression: aws.String("SET balance = balance + :amount"),
        }},
    },
})
```

- **Technical Terms to Include:** `TransactWriteItems`, ACID, `ClientRequestToken`, idempotency, 2x capacity cost, `ConditionExpression`, `attributevalue.MarshalMap`
- **Gotcha:** "DynamoDB transactions solve the distributed transaction problem completely" — they are scoped to DynamoDB only. A Go service that needs to atomically update DynamoDB AND call an external HTTP API AND publish to SQS still requires saga pattern or outbox pattern — DynamoDB transactions don't cross service boundaries.
- **Follow-Up:** "How do you implement the outbox pattern in Go with DynamoDB?" → Write the domain record and an outbox event in a single `TransactWriteItems`; a separate Go Lambda/goroutine polls the outbox table and publishes events, deleting processed entries. Asked because outbox is the correct pattern for reliable event publishing from a Go service.
- **Conclusion:** DynamoDB transactions provide ACID at DynamoDB scale — use them for intra-database consistency requirements; saga or outbox for cross-service consistency.

---

### Category: Go + SQS Consumer Architecture

> 📁 1 topic · 1 question

#### Topic: Long-Polling SQS Consumer in Go — (Q count: 1)

- **[C.Go+SQSConsumerArchitecture.LongPollingSQSConsumerInGo.Q1]**
  **Question:** Design a Go SQS consumer service that runs as a persistent ECS Fargate task with graceful shutdown, rather than as a Lambda function.
- **Answer:**
  - Long-poll loop: call `sqs.ReceiveMessage` with `WaitTimeSeconds: 20` (long polling) — reduces empty responses and API call cost vs short polling; blocks up to 20s if no messages.
  - Concurrency: receive a batch (up to 10 messages), dispatch each to a worker goroutine via a buffered channel — process messages concurrently while bounding concurrency with `semaphore` or a worker pool.
  - Graceful shutdown: listen for `SIGTERM` in a separate goroutine; on signal, cancel the root context — the `ReceiveMessage` call checks `ctx.Done()` between polls; in-flight message processing completes before exit.
  - Visibility timeout management: for messages that take longer than the visibility timeout to process, call `ChangeMessageVisibility` to extend — prevents duplicate redelivery of in-progress messages.
  - Dead letter queue: after N failures (configurable on the SQS queue), messages move to a DLQ — Go consumer should not implement retry logic for non-transient failures; let the DLQ handle exhausted retries.
- **Example:**

```go
func consume(ctx context.Context, client *sqs.Client, queueURL string) {
    sem := make(chan struct{}, 10) // max 10 concurrent processors
    var wg sync.WaitGroup
    for {
        select {
        case <-ctx.Done():
            wg.Wait() // drain in-flight before exit
            return
        default:
        }
        result, err := client.ReceiveMessage(ctx, &sqs.ReceiveMessageInput{
            QueueUrl:            aws.String(queueURL),
            MaxNumberOfMessages: 10,
            WaitTimeSeconds:     20,
        })
        if err != nil { continue }
        for _, msg := range result.Messages {
            sem <- struct{}{}
            wg.Add(1)
            go func(m types.Message) {
                defer func() { <-sem; wg.Done() }()
                if err := process(ctx, m); err == nil {
                    client.DeleteMessage(ctx, &sqs.DeleteMessageInput{
                        QueueUrl:      aws.String(queueURL),
                        ReceiptHandle: m.ReceiptHandle,
                    })
                }
            }(msg)
        }
    }
}
```

- **Technical Terms to Include:** long polling, `WaitTimeSeconds: 20`, visibility timeout, `ChangeMessageVisibility`, DLQ, semaphore, `SIGTERM` graceful shutdown, `sync.WaitGroup` drain, `ReceiptHandle`
- **Gotcha:** "Not deleting the message after successful processing" — SQS at-least-once delivery means the message reappears after the visibility timeout and is reprocessed. Go consumers must explicitly `DeleteMessage` after confirmed successful processing.
- **Follow-Up:** "How do you handle a message that always fails processing?" → Let the DLQ take it after the configured `maxReceiveCount`. Never implement infinite retry in the consumer — it blocks the consumer goroutine indefinitely and starves healthy messages. Asked because consumer retry discipline is an operational requirement.
- **Conclusion:** A persistent Go SQS consumer on ECS requires explicit concurrency control, visibility timeout management, and context-aware graceful shutdown — it is a more complex operational model than Lambda, but avoids cold starts and supports sustained high throughput.

---

### Category: Observability

> 📁 1 topic · 1 question

#### Topic: Distributed Tracing for Go Services on AWS — (Q count: 1)

- **[C.Observability.DistributedTracingForGoServicesOnAWS.Q1]**
  **Question:** How do you implement end-to-end distributed tracing across multiple Go services on AWS using X-Ray and OpenTelemetry?
- **Answer:**
  - **X-Ray native:** import `github.com/aws/aws-xray-sdk-go`; wrap `http.Handler` with `xray.Handler`; wrap AWS SDK config with `awsmiddleware` to auto-instrument SDK calls — trace IDs propagate via `X-Amzn-Trace-Id` HTTP header automatically.
  - **OpenTelemetry (preferred for vendor neutrality):** `go.opentelemetry.io/otel` provides the SDK; `otlptracehttp` or `otlptracegrpc` exports to the OpenTelemetry Collector sidecar, which forwards to X-Ray or Jaeger — the Go service code is identical regardless of backend.
  - Span creation: `tracer.Start(ctx, "operation-name")` creates a child span; `defer span.End()` closes it; `span.SetAttributes(attribute.String("offerID", id))` adds searchable metadata.
  - Context propagation across HTTP: inject trace context into outgoing requests with `otel.GetTextMapPropagator().Inject(ctx, propagation.HeaderCarrier(req.Header))`; extract on the receiving service with `Extract`.
  - Sampling strategy: 5% head-based sampling for high-volume services; 100% sampling for error traces (tail-based via X-Ray groups or Grafana Tempo) — avoid 100% sampling in production at high RPS.
- **Example:**

```go
// Instrument outgoing HTTP call
req, _ := http.NewRequestWithContext(ctx, "GET", upstreamURL, nil)
otel.GetTextMapPropagator().Inject(ctx, propagation.HeaderCarrier(req.Header))
resp, err := httpClient.Do(req)

// Create a custom span for a business operation
ctx, span := tracer.Start(ctx, "validate-offer-eligibility",
    trace.WithAttributes(attribute.String("offerID", offerID)))
defer span.End()
result, err := validateEligibility(ctx, offerID)
if err != nil { span.RecordError(err); span.SetStatus(codes.Error, err.Error()) }
```

- **Technical Terms to Include:** OpenTelemetry, `tracer.Start`, `span.End`, `span.SetAttributes`, W3C TraceContext propagation, `TextMapPropagator`, X-Ray trace ID, head-based sampling, tail-based sampling, OpenTelemetry Collector
- **Gotcha:** "Not calling `span.End()` — ever" → An unclosed span is never exported; it leaks memory in the SDK's span buffer. Always use `defer span.End()` immediately after `tracer.Start`. The most common OpenTelemetry bug in Go services.
- **Follow-Up:** "How do you propagate trace context through an SQS message?" → Inject the W3C TraceContext headers into the SQS message attributes as a `String` attribute; the consumer extracts them and creates a linked span. SQS does not propagate headers automatically — this is a manual step. Asked because async trace propagation is a non-obvious requirement.
- **Conclusion:** End-to-end distributed tracing in Go on AWS requires explicit propagation at every service boundary — HTTP headers, SQS message attributes, and DynamoDB do not propagate trace context automatically; the Go service must inject and extract it.

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
