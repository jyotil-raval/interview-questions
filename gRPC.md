<!-- Generated: 2026-03-29 | gRPC v1.71 | protobuf v26 -->

# gRPC Interview Preparation — Tech Lead / Architect Level

> gRPC v1.71 · Protocol Buffers v26 | Generated: 2026-03-29 | Career OS

---

## Scope & Calibration

gRPC is on your resume under "Backend & API Architecture: gRPC." At Tech Lead / Architect
level, interviewers don't ask "what does gRPC stand for." They ask:

- "Walk me through how a unary gRPC call actually traverses the stack."
- "How do you handle deadline propagation across a 5-service call chain?"
- "What's wrong with this proto definition — why will it break existing clients?"
- "You have 10 backend services — how do you manage the proto file contract?"
- "How does gRPC load balancing differ from HTTP/1.1 load balancing, and why does it matter?"
- "How do you implement auth in gRPC without API Gateway?"

This file covers all of these. The DataComms file covers the basics (Protobuf, HTTP/2,
4 patterns, gRPC vs REST). This file goes where the interview actually goes.

---

**L1–L6 Topics identified:**

- What is gRPC / HTTP/2 multiplexing / Protobuf wire format
- Service definition — .proto file design, field numbers, reserved fields
- Four communication patterns in depth — unary / server-streaming / client-streaming / bidi
- Interceptors — the middleware system of gRPC
- Deadlines, cancellation, and context propagation
- Error model — status codes, rich error details, error handling strategy
- Authentication — TLS, mTLS, token-based (metadata), per-call credentials
- Load balancing — client-side vs proxy-side, the L7 problem with L4 balancers
- Health checking — gRPC Health protocol, Kubernetes readiness
- Reflection & tooling — grpcurl, grpcui, Evans
- Protobuf schema evolution — backward/forward compatibility rules
- gRPC-Web — browser limitation and the proxy workaround
- Service mesh integration — Envoy, Istio, gRPC's native xDS
- gRPC vs REST decision logic — when each is right

---

## L1 — The Foundation: What gRPC Actually Is

---

### HTTP/2 Multiplexing & Protobuf Wire Format

- **Question:** How does HTTP/2 multiplexing make gRPC different from REST over HTTP/1.1, and what does the Protobuf wire format actually look like?
- **Answer:**
  - **HTTP/1.1 problem:** One request per TCP connection (or pipelining — which head-of-line blocking makes ineffective). A service making 10 concurrent gRPC calls over HTTP/1.1 needs 10 TCP connections. Connection setup overhead is significant; connection pools become bottlenecks.
  - **HTTP/2 multiplexing:** Multiple logical streams over a single TCP connection, each stream is independent (no head-of-line blocking at the stream level). A single TCP connection from service A to service B can carry 100 concurrent gRPC calls simultaneously. This is why gRPC benchmarks dramatically outperform HTTP/1.1 REST for concurrent workloads — the TCP handshake and TLS negotiation happen once.
  - **HTTP/2 frames:** Data is broken into frames. Each frame has a stream ID. The receiver reassembles frames by stream ID. gRPC maps one RPC call to one HTTP/2 stream. The request headers go in a `HEADERS` frame; the Protobuf-serialised body goes in `DATA` frames; end-of-stream is signalled by the `END_STREAM` flag.
  - **Protobuf wire format:** Binary encoding of field number + wire type + value. Fields are identified by their **field number** (not name) in the binary format. `field_number << 3 | wire_type` is the tag. Wire types: 0 = varint (int32, int64, bool, enum), 1 = 64-bit fixed, 2 = length-delimited (string, bytes, embedded messages, repeated fields), 5 = 32-bit fixed. Integers use varint encoding — small numbers take fewer bytes. `1` encodes as 1 byte; `128` encodes as 2 bytes. This is why Protobuf is compact: no field names, no brackets, binary integers.
- **Example:**

```protobuf
// user.proto
message GetUserRequest {
  string user_id = 1;  // field number 1, wire type 2 (length-delimited)
}

message User {
  string id    = 1;
  string email = 2;
  int32  age   = 3;  // wire type 0 (varint)
  bool   active = 4; // wire type 0 (varint — 0 or 1)
}
```

```
// Wire encoding of GetUserRequest{user_id: "abc"}:
// Tag: field 1, wire type 2 = (1 << 3) | 2 = 0x0A
// Length: 3 (bytes in "abc")
// Value: 0x61 0x62 0x63 ("abc" in UTF-8)
// Full: 0x0A 0x03 0x61 0x62 0x63 — 5 bytes total
// Compare to JSON: {"user_id":"abc"} = 17 bytes

// HTTP/2 stream — one RPC call = one stream
// Stream 1: GetUser request   → Stream 1: GetUser response
// Stream 3: ListOrders request → Stream 3: ListOrders response (concurrent)
// Stream 5: WatchStatus request → Stream 5: WatchStatus stream (ongoing)
// All on ONE TCP connection — no head-of-line blocking between streams
```

- **Technical Terms to Include:** HTTP/2, stream, multiplexing, frame, `HEADERS` frame, `DATA` frame, `END_STREAM`, head-of-line blocking, HPACK compression, Protobuf, varint, field number, wire type, tag encoding, length-delimited, binary protocol
- **Gotcha:** HTTP/2 eliminates head-of-line blocking **at the stream level**, not at the TCP level. TCP head-of-line blocking still exists — a lost TCP packet blocks all streams on that connection until retransmitted. This is why gRPC over HTTP/3 (QUIC) is being developed — QUIC provides true per-stream independence even under packet loss. For production gRPC in high-packet-loss environments, HTTP/2 multiplexing may perform worse than multiple HTTP/1.1 connections because one loss event blocks all streams.
- **Follow-Up:** "Why does gRPC use Protobuf by default and not JSON?" → JSON is human-readable but: (1) parsing is slow (character-by-character text processing), (2) no schema enforcement at the wire level, (3) field names are repeated in every message (bandwidth waste), (4) no native binary encoding (base64 for bytes). Protobuf: binary parsing is fast, field numbers enforce schema alignment, no repeated field names, varint encoding for integers. Trade-off: not human-readable — requires tooling to inspect. For debugging, `grpcurl` decodes binary to JSON. → They're testing format trade-off depth.
- **Conclusion:** gRPC's performance advantage comes from two compounding factors — HTTP/2 multiplexing (amortises connection cost across all concurrent RPCs) and Protobuf encoding (smaller payloads, faster serialisation) — together making it 5-10x more efficient than JSON/REST for high-concurrency internal service communication.

---

## L2 — Proto File Design: The Contract Layer

---

### Field Numbers, Reserved Fields, Schema Evolution

- **Question:** What rules govern Protobuf schema evolution, which changes are safe vs breaking, and how do you safely deprecate fields?
- **Answer:**
  - **Field numbers are the identity** of a field in the binary format — not the name. The binary encoding uses field numbers, not names. Renaming a field is **safe** (binary-compatible — old clients still decode by field number). Changing or reusing a field number is **breaking** — old binaries misinterpret the data.
  - **Safe changes (backward and forward compatible):**
    - Add a new optional field with a new field number (old code ignores it, new code reads it)
    - Rename a field (field number unchanged — binary unaffected)
    - Change a field's default value (proto3 defaults are always zero-value)
    - Change `repeated` field to `optional` of the same type (and vice versa for scalars)
  - **Breaking changes:**
    - Remove a field and **reuse its field number** for a different field — old data is misinterpreted
    - Change a field's type to an incompatible wire type (e.g., `int32` → `string` — different wire types)
    - Change a field from `optional` to `required` (proto2 only — proto3 has no `required`)
  - **Safe field removal:** Use `reserved` to mark the field number as off-limits. Future additions cannot reuse it. Also reserve the field name (prevents reuse by generated code). `reserved 5; reserved "old_field_name";`
  - **Deprecation in proto3:** No formal deprecated keyword in proto3 (proto2 has `[deprecated=true]`). Convention: add a comment `// Deprecated: use new_field instead`, set `[deprecated = true]` option (still supported), and use `reserved` after removal.
- **Example:**

```protobuf
syntax = "proto3";
package order.v1;

message Order {
  string  id          = 1;
  string  customer_id = 2;
  float   total       = 3;

  // Deprecated — do not use. Use line_items instead.
  // Kept for backward compatibility with v1 clients.
  string  legacy_sku  = 4 [deprecated = true];

  repeated LineItem line_items = 5; // added in v2 — safe (new field number)

  // REMOVED field — field number 6 was: string promo_code
  // Reserved to prevent future misuse
  reserved 6;
  reserved "promo_code";

  // WRONG: reuse of field number 6 for a new field — DO NOT DO THIS
  // string discount_code = 6; // old clients will decode this as promo_code data
}

message LineItem {
  string product_id = 1;
  int32  quantity   = 2;
  float  price      = 3;
}

// Safe type changes — same wire type
// int32 → int64: both varint (wire type 0) — safe
// int32 → string: varint → length-delimited (wire type 0 → 2) — BREAKING
```

- **Technical Terms to Include:** field number, wire type, backward compatibility, forward compatibility, `reserved`, deprecated, `[deprecated = true]`, proto3 vs proto2, schema registry, Buf (schema management tool), breaking change detection, `optional`, `repeated`
- **Gotcha:** In proto3, **all fields are optional** — there is no `required`. A missing field is decoded as its zero value (`0`, `""`, `false`, `nil`). This means you cannot distinguish between "field was not set" and "field was explicitly set to zero" at the proto level. For nullable semantics, use `google.protobuf.StringValue` (wrapper type) or the proto3 optional feature (generates a `HasField` method). This is the most common proto3 design mistake.
- **Follow-Up:** "How do you manage proto files across 10 microservices without chaos?" → (1) **Centralised proto repository** — all `.proto` files in one repo (or monorepo), versioned independently. (2) **Buf** — linting (`buf lint`), breaking change detection (`buf breaking`), code generation (`buf generate`), and schema registry (Buf Schema Registry). (3) **Versioned packages** — `order.v1`, `order.v2` — breaking changes in a new package, old package maintained during migration. (4) **Generated code as packages** — publish generated SDKs to artifact registries (npm, PyPI, Maven). → They're testing enterprise-scale proto management depth.
- **Conclusion:** Protobuf's schema evolution safety comes entirely from the discipline of treating field numbers as permanent identifiers — `reserved` after removal, new field numbers for new fields, and Buf for automated breaking change detection are the operational practices that keep a multi-service proto ecosystem from becoming a minefield.

---

## L3 — Interceptors: The Middleware System of gRPC

---

### Unary & Streaming Interceptors

- **Question:** What are gRPC interceptors, how do they differ from HTTP middleware, and what are the production use cases for server-side and client-side interceptors?
- **Answer:**
  - **Interceptors** are gRPC's middleware — they wrap handler execution, allowing cross-cutting logic (auth, logging, metrics, tracing, retry, rate limiting) to be applied across all RPCs without modifying individual handlers.
  - **Two types:** (1) **Unary interceptor** — wraps unary RPCs (single request → single response). Equivalent to HTTP middleware. (2) **Stream interceptor** — wraps streaming RPCs (any of the three streaming patterns). Receives the stream object and can intercept send/recv operations.
  - **Server-side interceptors:** Applied to all incoming RPCs. Use cases: authentication (extract and validate token from metadata), authorisation (check claims against resource), request logging (log all RPCs with method name, duration, status), Prometheus metrics (increment counters per method), distributed tracing (extract trace context from metadata, create spans), rate limiting (per-method or per-client).
  - **Client-side interceptors:** Applied to all outgoing RPCs. Use cases: injecting auth tokens into metadata, adding trace context propagation, retry with backoff, timeout injection, request logging for outbound calls.
  - **Interceptor chaining:** Multiple interceptors compose as a chain. gRPC only natively supports one interceptor per server/client — use `go-grpc-middleware` (or equivalent) for chaining multiple interceptors.
- **Example:**

```go
// Server-side unary interceptor — auth + logging
func authAndLogInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    start := time.Now()

    // Step 1: Extract token from metadata
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, status.Error(codes.Unauthenticated, "missing metadata")
    }
    tokens := md.Get("authorization")
    if len(tokens) == 0 {
        return nil, status.Error(codes.Unauthenticated, "missing authorization token")
    }

    // Step 2: Validate token and inject claims into context
    claims, err := validateJWT(strings.TrimPrefix(tokens[0], "Bearer "))
    if err != nil {
        return nil, status.Error(codes.Unauthenticated, "invalid token")
    }
    ctx = context.WithValue(ctx, userClaimsKey{}, claims)

    // Step 3: Call the actual handler
    resp, err := handler(ctx, req)

    // Step 4: Log after handler returns
    st, _ := status.FromError(err)
    log.Printf("method=%s duration=%v status=%s",
        info.FullMethod, time.Since(start), st.Code())

    return resp, err
}

// Client-side unary interceptor — inject auth token
func tokenInjectorInterceptor(token string) grpc.UnaryClientInterceptor {
    return func(
        ctx context.Context,
        method string, req, reply interface{},
        cc *grpc.ClientConn,
        invoker grpc.UnaryInvoker,
        opts ...grpc.CallOption,
    ) error {
        // Inject token into outgoing metadata
        ctx = metadata.AppendToOutgoingContext(ctx, "authorization", "Bearer "+token)
        return invoker(ctx, method, req, reply, cc, opts...)
    }
}

// Register interceptors on server
s := grpc.NewServer(
    grpc.ChainUnaryInterceptor(
        authAndLogInterceptor,
        metricsInterceptor,   // Prometheus counters
        tracingInterceptor,   // OpenTelemetry spans
    ),
    grpc.ChainStreamInterceptor(
        streamAuthInterceptor,
        streamTracingInterceptor,
    ),
)

// Register interceptors on client
conn, _ := grpc.Dial(target,
    grpc.WithChainUnaryInterceptor(
        tokenInjectorInterceptor(token),
        retryInterceptor,
    ),
)
```

- **Technical Terms to Include:** unary interceptor, stream interceptor, `grpc.UnaryServerInterceptor`, `grpc.UnaryClientInterceptor`, `grpc.StreamServerInterceptor`, interceptor chain, `grpc.ChainUnaryInterceptor`, metadata, `metadata.FromIncomingContext`, `metadata.AppendToOutgoingContext`, cross-cutting concern, handler, `UnaryHandler`, full method name
- **Gotcha:** Stream interceptors receive the **stream object**, not individual messages. To intercept individual messages in a stream (e.g., validate every message in a client-streaming RPC), you must wrap the stream object in a custom `grpc.ServerStream` implementation that overrides `RecvMsg` and `SendMsg`. This is significantly more complex than unary interceptors — most production uses of stream interceptors only handle the stream setup (auth check on first call) rather than per-message interception.
- **Follow-Up:** "How do you implement distributed tracing across gRPC services?" → Use OpenTelemetry. The client-side interceptor extracts the current trace context and injects it as gRPC metadata (`traceparent`, `tracestate` W3C headers or binary format). The server-side interceptor extracts the trace context from incoming metadata and creates a child span. Both sides use `otelgrpc` (the official OpenTelemetry gRPC instrumentation) which handles propagation automatically — you only need to register it as an interceptor. → They're testing observability integration depth.
- **Conclusion:** gRPC interceptors are the correct architectural location for cross-cutting concerns — auth, metrics, tracing, retry, and rate limiting belong in interceptors, not in individual service handlers; the interceptor chain is what makes gRPC services composable and observable without business logic pollution.

---

## L4 — Deadlines, Cancellation & Context Propagation

---

### Deadline Propagation Across Service Chains

- **Question:** How do gRPC deadlines work, how do they propagate across service chains, and what is the cascading timeout anti-pattern?
- **Answer:**
  - **Deadline:** A point in time by which an RPC must complete. Set by the caller, sent in the HTTP/2 `grpc-timeout` header, enforced by both the client and server. The server should check `ctx.Done()` during long operations to detect when the deadline has expired. If the deadline expires, the RPC returns `DEADLINE_EXCEEDED`.
  - **Timeout vs Deadline:** A timeout is a duration (`5s`). A deadline is an absolute time. gRPC uses deadlines internally. `context.WithTimeout(ctx, 5*time.Second)` creates a context with a deadline 5 seconds from now. The deadline is transmitted as a duration relative to transmission time (HTTP/2 `grpc-timeout: 5S`).
  - **Propagation:** When Service A calls Service B with a 10-second deadline, Service B should pass that same context (or a derived context with the same deadline) to any calls it makes to Service C. This ensures the entire call chain fails together when the original deadline expires. If B creates a fresh context for its call to C, C continues working after A's deadline has expired — wasting compute and downstream resources on work whose result will never be used.
  - **Cascading timeout anti-pattern:** Service A: 10s deadline. Service B (called by A): sets a fresh 10s deadline for its call to Service C. Total chain can take 20s before failing — 2x the client's deadline. The client fails at 10s, but B and C continue for another 10s. Fix: always propagate the incoming context downstream.
- **Example:**

```go
// WRONG — creates fresh context, breaks deadline propagation
func (s *OrderService) GetOrderWithDetails(ctx context.Context, req *pb.GetOrderRequest) (*pb.OrderDetails, error) {
    order, err := s.orderRepo.Get(ctx, req.OrderId)
    if err != nil { return nil, err }

    // BUG: fresh context loses the caller's deadline
    freshCtx := context.Background() // deadline not propagated!
    inventory, err := s.inventoryClient.GetStock(freshCtx, &inventorypb.GetStockRequest{
        ProductId: order.ProductId,
    })
    // If caller deadline is 5s and this call takes 8s — caller times out
    // but inventory service keeps running for 3 more seconds uselessly
    return buildDetails(order, inventory), nil
}

// CORRECT — propagate incoming context
func (s *OrderService) GetOrderWithDetails(ctx context.Context, req *pb.GetOrderRequest) (*pb.OrderDetails, error) {
    order, err := s.orderRepo.Get(ctx, req.OrderId)
    if err != nil { return nil, err }

    // Propagate ctx — inherits deadline from caller
    inventory, err := s.inventoryClient.GetStock(ctx, &inventorypb.GetStockRequest{
        ProductId: order.ProductId,
    })
    if err != nil {
        if status.Code(err) == codes.DeadlineExceeded {
            // The caller's deadline expired — don't retry, return immediately
            return nil, err
        }
        return nil, err
    }
    return buildDetails(order, inventory), nil
}

// Checking deadline in long-running server-side work
func (s *Server) ProcessBatch(ctx context.Context, req *pb.BatchRequest) (*pb.BatchResponse, error) {
    results := make([]*pb.Result, 0, len(req.Items))
    for _, item := range req.Items {
        // Check if deadline expired before processing each item
        select {
        case <-ctx.Done():
            return nil, status.FromContextError(ctx.Err()).Err()
            // Returns DEADLINE_EXCEEDED or CANCELED as appropriate
        default:
        }
        result, err := processItem(ctx, item)
        if err != nil { return nil, err }
        results = append(results, result)
    }
    return &pb.BatchResponse{Results: results}, nil
}
```

- **Technical Terms to Include:** deadline, timeout, `grpc-timeout` header, `DEADLINE_EXCEEDED`, `CANCELED`, `context.WithTimeout`, `context.WithDeadline`, `ctx.Done()`, `status.FromContextError`, deadline propagation, cascading timeout, context cancellation, `ctx.Err()`
- **Gotcha:** `DEADLINE_EXCEEDED` and `CANCELED` are distinct status codes with different client handling implications. `DEADLINE_EXCEEDED` — the deadline set by the caller expired. `CANCELED` — the caller explicitly cancelled the RPC (client closed the connection, context was cancelled programmatically). Never retry `CANCELED` — if the caller cancelled, they no longer want the result. Never retry `DEADLINE_EXCEEDED` — the deadline has already passed; a retry won't complete in time. Both codes signal "stop, the client doesn't need this result anymore."
- **Follow-Up:** "How does a gRPC server know the client's deadline has expired during a long streaming response?" → The server checks `ctx.Done()` — a channel that closes when the context is cancelled or its deadline expires. In a server-streaming RPC, the server should check `ctx.Done()` between sending each message. If `ctx.Done()` is closed, `stream.Send()` will return an error. Alternatively, any `stream.Send()` call after the client disconnects returns `transport is closing`. Always respect `ctx.Done()` in streaming servers to avoid wasting compute on abandoned streams. → They're testing streaming context awareness.
- **Conclusion:** Deadline propagation is the most critical and most commonly broken aspect of multi-service gRPC architectures — always passing the incoming `ctx` downstream ensures the entire call chain fails together when the original deadline expires, preventing the silent resource waste of downstream services continuing work whose result will never be used.

---

## L5 — Authentication, Load Balancing & Health Checking

---

### Authentication — TLS, mTLS, Token-Based

- **Question:** What are the three authentication layers in gRPC, and how do you implement per-call token authentication alongside transport-level TLS?
- **Answer:**
  - **Layer 1 — Transport security (TLS):** Encrypts all gRPC traffic in transit. gRPC strongly recommends TLS for all production traffic. `grpc.WithTransportCredentials(credentials.NewTLS(tlsConfig))` on the client. Server loads certificate and private key. Without TLS, gRPC defaults to insecure (plaintext) — never acceptable in production.
  - **Layer 2 — Mutual TLS (mTLS):** The server also validates the client's certificate — bidirectional authentication. Every service has a certificate signed by a shared CA. The server verifies the client cert before accepting the connection. Service-to-service zero-trust authentication without application-level tokens. Used in service meshes (Istio, Linkerd automate mTLS between pods). `tls.Config{ClientAuth: tls.RequireAndVerifyClientCert}`.
  - **Layer 3 — Per-call credentials (metadata):** Authentication at the RPC level, not the connection level. The client attaches credentials (JWT, API key, OAuth token) to each call as gRPC metadata (HTTP/2 headers). The server reads metadata in a server-side interceptor. Multiple callers can share one connection with different per-call credentials. This is the most common auth pattern for public-facing gRPC APIs.
  - **`grpc.PerRPCCredentials` interface:** Implements `GetRequestMetadata(ctx, uri) (map[string]string, error)` — called before each RPC. Returns the metadata map (e.g., `{"authorization": "Bearer <token>"}`). Attach to a client with `grpc.WithPerRPCCredentials(creds)`. The framework calls `GetRequestMetadata` automatically — you don't manually inject metadata on each call.
- **Example:**

```go
// TLS client — basic transport encryption
tlsConfig := &tls.Config{
    RootCAs: certPool,  // trust the server's CA
    ServerName: "api.internal.example.com",
}
conn, err := grpc.Dial(
    "api.internal.example.com:443",
    grpc.WithTransportCredentials(credentials.NewTLS(tlsConfig)),
)

// mTLS client — client also presents a certificate
mtlsConfig := &tls.Config{
    Certificates: []tls.Certificate{clientCert},  // client's cert + key
    RootCAs:      serverCertPool,
    ServerName:   "api.internal.example.com",
}
conn, err := grpc.Dial(
    "api.internal.example.com:443",
    grpc.WithTransportCredentials(credentials.NewTLS(mtlsConfig)),
)

// Per-call token credentials — JWT per RPC
type jwtCredentials struct {
    tokenSource oauth2.TokenSource
}

func (j *jwtCredentials) GetRequestMetadata(ctx context.Context, uri ...string) (map[string]string, error) {
    token, err := j.tokenSource.Token()
    if err != nil { return nil, err }
    return map[string]string{
        "authorization": "Bearer " + token.AccessToken,
    }, nil
}

func (j *jwtCredentials) RequireTransportSecurity() bool { return true }
// RequireTransportSecurity=true: rejects if connection is not TLS
// Prevents accidental token leakage over plaintext

conn, err := grpc.Dial(
    target,
    grpc.WithTransportCredentials(credentials.NewTLS(tlsConfig)),
    grpc.WithPerRPCCredentials(&jwtCredentials{tokenSource: ts}),
)

// Server-side: read token from metadata in interceptor
func authInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    md, _ := metadata.FromIncomingContext(ctx)
    tokens := md.Get("authorization") // reads the metadata key set by GetRequestMetadata
    if len(tokens) == 0 {
        return nil, status.Error(codes.Unauthenticated, "authorization required")
    }
    // validate token...
    return handler(ctx, req)
}
```

- **Technical Terms to Include:** TLS, mTLS, transport credentials, per-call credentials, `grpc.PerRPCCredentials`, `GetRequestMetadata`, `RequireTransportSecurity`, metadata, service mesh, certificate authority (CA), `tls.Config`, `ClientAuth`, `credentials.NewTLS`, ALTS (Google's internal transport security)
- **Gotcha:** `RequireTransportSecurity() bool` returning `false` on a `PerRPCCredentials` implementation means the credentials will be sent over plaintext connections. This silently transmits auth tokens in the clear. Always return `true` — this causes gRPC to reject the `PerRPCCredentials` if the connection is not TLS-secured. It's a safeguard against misconfiguration. Almost every example online returns `false` for simplicity — don't copy that into production.
- **Follow-Up:** "How does a service mesh like Istio change gRPC authentication?" → Istio's Envoy sidecar handles mTLS automatically between all pods — certificates are rotated via SPIFFE/SPIRE, zero manual certificate management. The application code doesn't need to configure TLS at all — Envoy intercepts gRPC traffic and applies mTLS transparently. Application-level auth (JWT validation) still happens in your interceptor. The result: transport security is infrastructure's responsibility, application auth is the service's responsibility — clean separation. → They're testing service mesh + gRPC architecture integration.
- **Conclusion:** gRPC authentication is layered — TLS for transport encryption, mTLS for service identity, and per-call credentials for user identity; production gRPC requires at minimum TLS + per-call auth, and service meshes like Istio make mTLS operationally viable by automating certificate lifecycle management.

---

### Load Balancing — The L7 Problem

- **Question:** Why does standard L4 load balancing break gRPC's performance characteristics, and how do you implement correct L7 gRPC load balancing?
- **Answer:**
  - **The problem:** A standard L4 load balancer (e.g., AWS NLB, a basic TCP load balancer) routes TCP connections — it sees one TCP connection from Service A to the load balancer and forwards it to one backend instance. Because HTTP/2 multiplexes all gRPC calls over that one TCP connection, **all RPCs from Service A go to the same backend instance** — the load balancer never sees individual RPCs, only the TCP connection. Result: one backend is overwhelmed; others are idle.
  - **L7 gRPC load balancing (Envoy / Istio / nginx):** The proxy terminates the HTTP/2 connection from the client, inspects individual gRPC streams (HTTP/2 streams), and routes each stream independently to a backend. Each RPC is balanced independently regardless of TCP connection. This requires a proxy that speaks HTTP/2 — Envoy is the standard, nginx (with gRPC module) also works.
  - **Client-side load balancing:** The client resolves a service name to multiple backend IPs (via DNS or a service registry) and distributes RPCs across them directly — no proxy in the path. Supported by gRPC's built-in `round_robin` and `pick_first` load balancing policies. Requires the client to maintain connections to all backends. Works well in Kubernetes (headless Service → DNS returns all Pod IPs). Eliminates proxy latency; adds client complexity.
  - **xDS (gRPC native service mesh):** gRPC has native support for the xDS API (Envoy's service discovery API). A control plane (Istio, Cloud Traffic Director) pushes routing configuration directly to gRPC clients — client-side load balancing with dynamic configuration. No sidecar proxy needed. Currently the most advanced gRPC deployment model.
- **Example:**

```yaml
# Kubernetes — headless Service for client-side load balancing
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  clusterIP: None  # headless — DNS returns all Pod IPs
  selector:
    app: user-service
  ports:
    - port: 50051

# gRPC client — round_robin across all Pod IPs returned by DNS
conn, err := grpc.Dial(
    "dns:///user-service.default.svc.cluster.local:50051",
    grpc.WithDefaultServiceConfig(`{"loadBalancingPolicy":"round_robin"}`),
    grpc.WithTransportCredentials(credentials.NewTLS(tlsConfig)),
)
// gRPC resolves the headless Service DNS → gets all Pod IPs
// Establishes connections to each Pod
// Routes each RPC round-robin across all backends

# Envoy sidecar (Istio) — L7 proxy-side load balancing
# No change to application code — Istio's Envoy intercepts gRPC calls
# Envoy speaks HTTP/2, sees individual streams, balances per-stream
# Configured via VirtualService / DestinationRule Istio CRDs
```

- **Technical Terms to Include:** L4 load balancer, L7 load balancer, HTTP/2 multiplexing, client-side load balancing, proxy-side load balancing, Envoy, xDS, headless Service, `dns:///`, `round_robin`, `pick_first`, service discovery, GRPCLB (deprecated), `ServiceConfig`
- **Gotcha:** The `grpc.WithDefaultServiceConfig` JSON must be valid gRPC ServiceConfig JSON — not just any JSON. If the JSON is malformed or uses invalid policy names, gRPC silently falls back to `pick_first` (always uses the first address). There is no error thrown. You can verify by checking the connection state or by adding a channel connectivity callback. Always test that load balancing is actually working under load — RPC distribution should be visibly even across backends.
- **Follow-Up:** "What happens to in-flight gRPC calls when a backend pod restarts during rolling deployment?" → With client-side load balancing: the client has connections to all pods. When a pod terminates, its connection closes — the gRPC client detects the connection drop and retries the RPC on another backend (if the call is idempotent and retry is configured). With Envoy: Envoy detects the pod removal, removes it from the backend pool, and routes new RPCs to healthy pods. In-flight streaming RPCs to the terminating pod receive a `transport is closing` error — the client must handle this and reconnect the stream. → They're testing rolling deployment resilience for gRPC.
- **Conclusion:** L4 load balancing silently breaks gRPC's horizontal scalability by pinning all RPCs from one client to one backend — L7 balancing (Envoy sidecar) or client-side `round_robin` with headless Kubernetes Services are the two production-viable patterns, chosen based on whether a service mesh is already in the architecture.

---

### Health Checking — gRPC Health Protocol

- **Question:** What is the gRPC Health Checking Protocol and how do you integrate it with Kubernetes readiness and liveness probes?
- **Answer:**
  - The **gRPC Health Checking Protocol** is a standardised gRPC service defined by Google — `grpc.health.v1.Health` with a single RPC: `Check(HealthCheckRequest) → HealthCheckResponse` (and a streaming `Watch` variant). The response contains a `status` field: `SERVING`, `NOT_SERVING`, or `SERVICE_UNKNOWN`. This is the convention Kubernetes, load balancers, and service meshes use to determine gRPC service health.
  - **Why not just HTTP `/health`?** A gRPC server is not an HTTP server (without HTTP/2 co-hosting). Kubernetes readiness/liveness probes historically required HTTP or TCP — not gRPC. Since Kubernetes 1.24, gRPC probes are natively supported. Before 1.24: use `grpc-health-probe` (a small binary that speaks the gRPC Health protocol and exits 0/1) as the probe command.
  - **Service-level health:** `Check({service: ""})` checks overall server health. `Check({service: "order.v1.OrderService"})` checks a specific service. A server can report different health statuses per service — useful for progressive degradation (one service failing shouldn't fail the whole pod).
  - **`Watch` RPC:** Server streams health status changes. Load balancers and service meshes use this to get real-time push notifications when a service transitions from `SERVING` to `NOT_SERVING` — faster than polling.
- **Example:**

```go
// Server — register the health service
import "google.golang.org/grpc/health"
import "google.golang.org/grpc/health/grpc_health_v1"

healthServer := health.NewServer()
grpc_health_v1.RegisterHealthServer(grpcServer, healthServer)

// Set initial status
healthServer.SetServingStatus("order.v1.OrderService", grpc_health_v1.HealthCheckResponse_SERVING)

// Update status when dependencies are unhealthy
func monitorDependencies(ctx context.Context, healthServer *health.Server, db *sql.DB) {
    ticker := time.NewTicker(10 * time.Second)
    for {
        select {
        case <-ticker.C:
            if err := db.PingContext(ctx); err != nil {
                healthServer.SetServingStatus(
                    "order.v1.OrderService",
                    grpc_health_v1.HealthCheckResponse_NOT_SERVING,
                )
            } else {
                healthServer.SetServingStatus(
                    "order.v1.OrderService",
                    grpc_health_v1.HealthCheckResponse_SERVING,
                )
            }
        case <-ctx.Done():
            return
        }
    }
}
```

```yaml
# Kubernetes — native gRPC probe (K8s 1.24+)
livenessProbe:
  grpc:
    port: 50051
    service: ""    # empty = overall server health

readinessProbe:
  grpc:
    port: 50051
    service: "order.v1.OrderService"  # specific service health

# Before K8s 1.24 — grpc-health-probe binary
readinessProbe:
  exec:
    command: ["/bin/grpc_health_probe", "-addr=:50051", "-service=order.v1.OrderService"]
  initialDelaySeconds: 5
  periodSeconds: 10
```

- **Technical Terms to Include:** gRPC Health Checking Protocol, `grpc.health.v1.Health`, `HealthCheckRequest`, `HealthCheckResponse`, `SERVING`, `NOT_SERVING`, `SERVICE_UNKNOWN`, `Watch` RPC, `grpc-health-probe`, Kubernetes gRPC probe (1.24+), health server, `SetServingStatus`, readiness vs liveness
- **Gotcha:** The health server's `SetServingStatus` is not thread-safe without the official `health.NewServer()` implementation (which uses a mutex internally). Rolling your own health service and updating status from multiple goroutines without a lock will cause data races. Always use the official `google.golang.org/grpc/health` package — it handles concurrency correctly and also implements the `Watch` streaming RPC.
- **Follow-Up:** "How does a service mesh like Istio use the gRPC health protocol?" → Istio's Envoy sidecar uses the `Watch` RPC to subscribe to health status changes. When your service sets `NOT_SERVING`, Envoy is notified in real time and stops routing new RPCs to that instance — faster than polling-based readiness probes (which have a `periodSeconds` delay). The pod isn't terminated (Kubernetes readiness handles that separately), but it's drained from Envoy's upstream pool immediately. This gives you sub-second health propagation in the service mesh. → They're testing service mesh + health integration depth.
- **Conclusion:** The gRPC Health Checking Protocol is the single convention that makes gRPC services observable by Kubernetes, load balancers, and service meshes — implementing it correctly (per-service statuses, dependency-based transitions) and registering it with the `health.NewServer()` package is the baseline for any production gRPC service.

---

## L6 — Tooling, gRPC-Web & Architecture Decisions

---

### Tooling — grpcurl, gRPC Reflection, Evans

- **Question:** How do you inspect and test a gRPC service in production without the proto files, and what is gRPC server reflection?
- **Answer:**
  - **gRPC Server Reflection:** A standardised gRPC service (`grpc.reflection.v1alpha.ServerReflection`) that exposes the server's proto schema at runtime. Tools can query the service for its service definitions, message types, and methods — without needing the `.proto` files compiled locally. **Do not enable reflection in production** — it exposes your full API schema to anyone who can reach the server. Enable only in development/staging.
  - **`grpcurl`** — curl for gRPC. Makes gRPC calls from the command line. With reflection enabled: `grpcurl -plaintext localhost:50051 list` → lists services. `grpcurl -plaintext localhost:50051 describe order.v1.OrderService` → shows the service definition. `grpcurl -plaintext -d '{"order_id":"42"}' localhost:50051 order.v1.OrderService/GetOrder` → makes a call. Without reflection: `grpcurl -proto order.proto -d '...' ...` — supply the proto file directly.
  - **Evans** — interactive gRPC client (REPL). `evans -r repl` → interactive shell. Navigate services, methods, and construct messages interactively. Better than `grpcurl` for exploratory testing.
  - **Postman / BloomRPC** — GUI clients for gRPC. Postman natively supports gRPC (import proto or use reflection). BloomRPC is lightweight and widely used in development.
  - **`buf`** — proto linting, breaking change detection, code generation, and schema registry. `buf breaking --against .git#branch=main` — check if current changes break the API contract. Integrates with CI.
- **Example:**

```bash
# Enable reflection — development only
import "google.golang.org/grpc/reflection"
// In server setup:
reflection.Register(grpcServer) // exposes schema — never in production

# grpcurl — inspect and call
grpcurl -plaintext localhost:50051 list
# order.v1.OrderService
# grpc.health.v1.Health
# grpc.reflection.v1alpha.ServerReflection

grpcurl -plaintext localhost:50051 describe order.v1.OrderService
# order.v1.OrderService is a service:
# service OrderService {
#   rpc GetOrder ( .order.v1.GetOrderRequest ) returns ( .order.v1.Order );
#   rpc ListOrders ( .order.v1.ListOrdersRequest ) returns ( stream .order.v1.Order );
# }

grpcurl -plaintext -d '{"order_id": "42"}' \
  localhost:50051 order.v1.OrderService/GetOrder
# { "id": "42", "status": "PAID", "total": "99.99" }

# With metadata (auth token)
grpcurl -plaintext \
  -H "authorization: Bearer eyJhbGci..." \
  -d '{"order_id":"42"}' \
  localhost:50051 order.v1.OrderService/GetOrder

# buf lint — check proto style
buf lint

# buf breaking — detect breaking changes vs main branch
buf breaking --against .git#branch=main
# proto/order/v1/order.proto:15:3:Field "3" with name "legacy_sku" on
# message "Order" changed option "[deprecated]" from "false" to "true".
```

- **Technical Terms to Include:** gRPC reflection, `grpcurl`, Evans, BloomRPC, Postman gRPC, Buf, `buf lint`, `buf breaking`, `reflection.Register`, schema exposure risk, proto file import, metadata header flag, JSON transcoding
- **Gotcha:** Never enable gRPC reflection in production. Reflection exposes your entire API schema — all service names, method signatures, request/response types — to anyone who can reach port 50051. In an environment where the gRPC port is accidentally exposed beyond the cluster, reflection becomes an API reconnaissance tool for attackers. Enable only in dev/staging, disabled by default. Gate it with an environment variable check.
- **Follow-Up:** "How do you test gRPC services in unit tests without spinning up a full server?" → Use `bufconn` — an in-memory gRPC connection that bypasses the network. `bufconn.Listen()` creates an in-memory listener; the gRPC server and client connect via `grpc.Dial` using a custom dialer. No ports, no network, no race conditions on port allocation. The test is as fast as pure function tests. This is the official recommended pattern for gRPC unit testing. → They're testing gRPC testing architecture.
- **Conclusion:** The gRPC tooling ecosystem — grpcurl for command-line testing, Evans for interactive exploration, buf for schema management — is what makes gRPC operationally manageable; the reflection service is the enabler of all these tools but its production security risk means it must be environment-gated from day one.

---

### gRPC-Web — Browser Limitation & Proxy Workaround

- **Question:** Why can't browsers use native gRPC, what is gRPC-Web, and what is the architecture required to support it?
- **Answer:**
  - **Why not native gRPC in browsers:** gRPC requires HTTP/2 with full control over HTTP/2 trailers (trailing headers sent after the response body). Browsers expose `fetch` and `XMLHttpRequest` — neither gives JavaScript control over HTTP/2 trailers. The `Fetch API` does not expose trailers to JavaScript. Additionally, browsers' preflight CORS mechanism doesn't work with HTTP/2 in the way gRPC requires. Native gRPC is impossible from a browser without a proxy.
  - **gRPC-Web:** A modified protocol that works within browser constraints. Key difference: trailers are encoded at the end of the HTTP response body (not as HTTP trailers) — JavaScript can read the body. Uses HTTP/1.1 or HTTP/2 in a way browsers support. Not the same wire format as native gRPC — requires a proxy to translate between gRPC-Web and gRPC.
  - **Required proxy:** Envoy with the `grpc_web` filter, or `grpc-web-proxy` (a standalone Go proxy). The browser sends gRPC-Web requests; the proxy translates them to native gRPC and forwards to the backend. The backend sees native gRPC — no changes required in the backend server.
  - **gRPC-Web limitations vs native gRPC:** No client-side streaming (browser can't stream request bodies). No bidirectional streaming. Only unary and server-streaming. This covers the majority of browser use cases.
  - **Alternative for browser:** gRPC-Gateway — a protoc plugin that generates a reverse proxy from proto annotations, exposing gRPC services as REST/JSON endpoints. Browsers use REST/JSON; the gateway translates to gRPC. More REST-compatible; no streaming. Best for teams with REST-first API consumers.
- **Example:**

```javascript
// Browser client — gRPC-Web (using @grpc/grpc-js won't work in browsers)
// Use grpc-web npm package instead
import { OrderServiceClient } from './generated/order_grpc_web_pb';
import { GetOrderRequest } from './generated/order_pb';

const client = new OrderServiceClient('https://api.example.com', null, null);
// ^ This URL points to Envoy (or grpc-web-proxy) — NOT the gRPC backend directly

const request = new GetOrderRequest();
request.setOrderId('42');

client.getOrder(request, { authorization: 'Bearer ' + token }, (err, response) => {
  if (err) {
    console.error(err.code, err.message);
    return;
  }
  console.log(response.getId(), response.getStatus());
});

// Server streaming — works in gRPC-Web
const stream = client.listOrders(request, { authorization: 'Bearer ' + token });
stream.on('data', (order) => console.log(order.getId()));
stream.on('end', () => console.log('stream complete'));
stream.on('error', (err) => console.error(err));
```

```yaml
# Envoy config — gRPC-Web translation filter
http_filters:
  - name: envoy.filters.http.grpc_web
    typed_config:
      '@type': type.googleapis.com/envoy.extensions.filters.http.grpc_web.v3.GrpcWeb
  - name: envoy.filters.http.router
```

- **Technical Terms to Include:** gRPC-Web, HTTP trailers, `fetch` API limitation, Envoy `grpc_web` filter, grpc-web-proxy, gRPC-Gateway, protoc annotation, CORS for gRPC-Web, client-streaming limitation, `@grpc/grpc-js` vs `grpc-web`, Connect (Buf)
- **Gotcha:** **Buf Connect** (from the Buf team) is an alternative to gRPC-Web that is natively browser-compatible without a special proxy. Connect uses HTTP/1.1 or HTTP/2 in a way browsers support natively, and is wire-compatible with gRPC. If starting a new project that needs browser clients, evaluate Connect — it may eliminate the Envoy-as-gRPC-Web-proxy requirement. For existing gRPC backends, Connect clients can call existing gRPC servers directly.
- **Follow-Up:** "When would you choose gRPC-Gateway over gRPC-Web for browser clients?" → gRPC-Gateway generates REST endpoints from proto annotations — browsers use standard `fetch` with JSON. No JavaScript gRPC library needed, no Envoy required, works with existing API clients and tools (Postman, curl). gRPC-Web requires the browser to use a gRPC-Web client library and Envoy. Choose gRPC-Gateway when: the API has external/third-party consumers who expect REST, you have OpenAPI tooling requirements, or you want to avoid the browser JavaScript gRPC client dependency. Choose gRPC-Web when: you want to preserve streaming semantics to the browser and are willing to run Envoy. → They're testing browser API architecture trade-off judgment.
- **Conclusion:** Browser clients cannot use native gRPC — gRPC-Web + Envoy is the standard workaround (with client-streaming limitations), gRPC-Gateway provides REST/JSON translation for REST-first consumers, and Buf Connect is the emerging alternative that eliminates the proxy requirement; the right choice depends on streaming requirements and operational complexity tolerance.

---

## Quick Reference — gRPC Decision Logic

| Decision                             | Answer                                                                                                                            |
| ------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------- |
| Unary or streaming?                  | Unary unless: server pushes multiple responses, client sends chunked data, or real-time bidirectional exchange is needed          |
| gRPC or REST?                        | gRPC for internal services (performance, type safety, streaming). REST for public APIs, browser clients, REST-ecosystem consumers |
| gRPC-Web or gRPC-Gateway?            | gRPC-Web when streaming to browser matters. gRPC-Gateway for REST/JSON API consumers                                              |
| Client-side or proxy load balancing? | Proxy (Envoy/Istio) if service mesh exists. Client-side `round_robin` + Kubernetes headless Service if no mesh                    |
| TLS required?                        | Always in production. `RequireTransportSecurity: true` on PerRPCCredentials                                                       |
| Enable reflection?                   | Dev/staging only. Never production                                                                                                |
| Deadline propagation?                | Always pass incoming `ctx` downstream. Never `context.Background()` in handlers                                                   |
| Safe field removal?                  | `reserved <field_number>; reserved "<field_name>";` — both required                                                               |
| Add field to existing message?       | New field number only. Never reuse removed field numbers                                                                          |
| Auth mechanism?                      | mTLS for service-to-service (service mesh preferred). JWT metadata for user auth via interceptor                                  |
| Status code for bad input?           | `INVALID_ARGUMENT` — don't retry                                                                                                  |
| Status code for retry?               | `UNAVAILABLE` only. Never retry `DEADLINE_EXCEEDED`, `CANCELED`, `NOT_FOUND`, `INVALID_ARGUMENT`                                  |

---

_— End of gRPC_Interview_QA.md —_
_Coverage: L1–L6 | 11 QA units | gRPC v1.71 · Protobuf v26_
_Calibrated for: Tech Lead / Architect level_
_Supersedes the gRPC section in DataComms_Interview_QA.md for depth_
