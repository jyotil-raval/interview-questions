# Persistent Systems — Golang Developer

## 🎯 Interview Prep Summary

---

### Role Context

Persistent Systems is an enterprise IT services and product engineering firm. A "Golang Developer" role at 10+ years seniority at Persistent almost certainly means: building backend microservices or platform services for an enterprise client, deploying them on AWS, and operating them in production. The JD's brevity (Golang + AWS, nothing else) is deliberate — at 10+ years, Persistent is hiring for judgment and architecture, not framework knowledge. Expect the interviewer to probe system design and trade-off reasoning, not syntax.

The two things they will test hardest: (1) whether you understand Go's concurrency model at production depth — goroutine lifecycle, context propagation, channel patterns — and (2) whether you can design a Go service on AWS that is observable, resilient, and cost-aware.

---

### Must Have — Priority Prep

**Golang v1.24** — The single most important topic: concurrency architecture.
An interviewer at this level will not ask "what is a goroutine." They will ask: "You have a service leaking goroutines in production — walk me through how you diagnose and fix it." That question requires GMP model knowledge, pprof fluency, context propagation discipline, and worker pool design — all of it.

Secondary focal points: Go's memory model (escape analysis, sync primitives), error handling philosophy at architecture scale, and generics trade-offs.
Files: `Persistent_Systems_Golang_Developer_Golang_QA.md`

**AWS** — The single most important topic for AWS at this role: serverless Go architecture and Go on EKS.
Persistent deploys enterprise client workloads on AWS. You will be asked to design a Go service deployment end-to-end. Know the Lambda execution model, SQS consumer patterns, DynamoDB data modelling, IAM role assumption (IRSA), and EKS deployment anatomy.

Secondary focal points: multi-AZ HA design, Secrets Manager integration, and distributed tracing (X-Ray / OpenTelemetry).
Files: `Persistent_Systems_Golang_Developer_AWS_QA.md`

---

### Combination File

**Golang + AWS** — The seams between Go and AWS are where the interview differentiates candidates.
The questions that test both simultaneously: partial SQS batch failure handling in Go Lambda, context propagation through AWS SDK v2 calls, DynamoDB transaction idempotency with `ClientRequestToken`, and distributed trace propagation through SQS message attributes. These are not textbook topics — they are production-failure lessons.
Files: `Persistent_Systems_Golang_Developer_Golang+AWS_QA.md`

---

### Prep Order Recommendation

Study in this sequence — rationale follows each step.

1. **Golang_QA.md → L1–L3** — Establish the mental model: GMP scheduler, goroutine vs thread, channels, context, error handling. These are the foundation all L4–L6 questions build on. Estimated: 3–4 hours.

2. **AWS_QA.md → L1–L3** — Establish the AWS primitives: IAM, Lambda, SQS, DynamoDB, S3. You will need these in the combination questions. Estimated: 2–3 hours.

3. **Golang+AWS_QA.md — Full file** — Read the combination questions after both individual L1–L3 passes. These questions assume you hold both mental models simultaneously. Estimated: 1.5–2 hours.

4. **Golang_QA.md → L4–L6** — Concurrency deep dive: memory model, escape analysis, pprof, GMP internals. This is where 10+ year differentiation lives. Estimated: 3–4 hours.

5. **AWS_QA.md → L4–L6** — Architecture and cost: multi-AZ design, EKS platform, Well-Architected trade-offs. Estimated: 2–3 hours.

6. **Mock system design question** — Before the interview, practice answering: "Design a Go microservice that processes orders from SQS, validates against an external API, writes to DynamoDB, and publishes a completion event to EventBridge. It must handle 10,000 orders/day with 99.9% uptime." Answer should cover: worker pool, context propagation, DynamoDB idempotency, EventBridge event schema, IAM role, ECS Fargate deployment, ALB health check, CloudWatch alarms. Estimated: 1 session, 45 min.

---

### Jyotil's Delivery Anchors to Pull In

Your resume tells a story that maps directly to this role. Use these in answers:

- **Apple Offers (TCS):** Go integration layer at the MFE shell → backend platform boundary. Pull this for Go API design, context propagation, contract-first API questions.
- **PayPal BFF (Altimetrik):** Fan-out to 4 upstream services using Go concurrency. Pull this for goroutine/channel patterns, worker pools, timeout handling. Also maps directly to SQS fan-out architecture.
- **Volansys IIoT:** AWS Lambda and DynamoDB for device management backend. Pull this for serverless Go architecture questions and DynamoDB data modelling.
- **mal-updater GitHub project:** OAuth2 PKCE, concurrent PATCH, channel-based concurrency in Go. Pull this for Go concurrency depth when no enterprise anchor is available.

---

### Key Gaps to Acknowledge Cleanly

Two gaps exist between your resume and this JD. Both are ownable in interview:

1. **GoLang seniority:** Your Go exposure is integration-layer and personal projects, not primary backend service ownership. Own it: "My Go work has been at the boundary layer — API contract design, type-safe response mapping, concurrency in my mal-updater project. I'm building toward backend service ownership, and that's the direction this role offers."

2. **AWS depth:** Your AWS exposure is real (Lambda, DynamoDB, S3, CloudFront at Volansys; Jenkins/CI context). The gap is: you haven't owned full AWS infrastructure design independently. Own it: "I've worked within AWS-deployed systems and contributed to the infrastructure decisions, but I haven't been the sole owner of the AWS architecture. The SAA-C03 certification I'm completing now is closing that gap deliberately."

Scripted cleanly, these gaps become signals of self-awareness — which is exactly what a 10-year experienced interviewer trusts more than overconfident claims.

---

### File Registry

| File                          | Classification | Depth | Focus                                                      |
| ----------------------------- | -------------- | ----- | ---------------------------------------------------------- |
| [Golang](./golang.md)         | Must Have      | L1–L6 | Concurrency, memory, architecture, runtime internals       |
| [AWS](./aws.md)               | Must Have      | L1–L6 | Lambda, EKS, DynamoDB, SQS, HA design, FinOps              |
| [Golang+AWS](./golang_aws.md) | Combination    | L1–L6 | SDK v2 patterns, Lambda SQS consumer, DynamoDB tx, tracing |
| [PrepSummary](./README.md)    | Summary        | —     | This file                                                  |
