<!-- Generated: 2026-03-12 | Redis 8.0 | gRPC v1.71 -->

# Data Layer & Service Communication Interview Preparation — Tech Lead / Architect Level

> Redis 8.0 · gRPC v1.71 | Generated: 2026-03-12 | Career OS

---

# SECTION 1 — Redis 8.0

---

**L1–L5 Topics identified:**

- What is Redis / Data structures as a service
- Core Data Structures — String / Hash / List / Set / Sorted Set / Stream
- Caching Patterns — Cache-aside / Write-through / Write-behind / Read-through
- Expiry, Eviction Policies & Memory Management
- Persistence — RDB / AOF / AOF+RDB
- Pub/Sub & Streams
- Redis Cluster & Sentinel (HA)
- Transactions, Lua scripting, Pipelining
- Redis 8.0 — Vector Sets, Query Engine

---

### Core Architecture

#### What is Redis / Data Structures

- **Question:** What is Redis, why is it not "just a cache," and what do its core data structures enable beyond simple key-value storage?
- **Answer:**
  - Redis (Remote Dictionary Server) is an in-memory data structure store. It is: a cache, a database (optional persistence), a message broker (Pub/Sub, Streams), a rate limiter, a session store, a leaderboard engine, and a distributed lock manager — all in one. "Just a cache" dramatically undersells it.
  - **Why in-memory:** All data is stored in RAM — reads and writes are O(1) for simple operations and achieve sub-millisecond latency (1–2ms p99 over the network). Disk is 1000x slower; Redis is fast because it avoids disk I/O entirely for hot operations.
  - **Core data structures:**
    (1) **String** — any binary data up to 512MB. Counters (`INCR`), session tokens, cached HTML.
    (2) **Hash** — field-value map. Ideal for objects (user profiles).
    (3) **List** — ordered, duplicates allowed. `LPUSH`/`RPOP` for queues; `LRANGE` for recent items.
    (4) **Set** — unordered, unique members. Union/intersection/difference operations. Tags, followers lists.
    (5) **Sorted Set (ZSet)** — members with scores. O(log n) insert/delete. Leaderboards, rate limiting, priority queues.
    (6) **Stream** — append-only log with consumer groups. Kafka-lite within Redis.
    (7) **HyperLogLog** — probabilistic cardinality (distinct count) in 12KB regardless of data size.
    (8) **Bitmap** — bit-level operations on strings. Daily active user tracking.
  - **Redis 8.0 (2025):** Vector Sets (native k-NN similarity search for embeddings) merged into core Redis. Query engine improvements (Redis Query Engine for full-text, vector, geo, numeric search). Active-Active Geo-Distribution improvements.
- **Example:**

```bash
# String — counter with atomic increment
SET page_views 0
INCR page_views        # atomic, returns 1
INCRBY page_views 10   # atomic, returns 11

# Hash — user object
HSET user:42 name "Alice" email "alice@example.com" plan "pro"
HGET user:42 name      # "Alice"
HGETALL user:42        # all fields

# Sorted Set — leaderboard
ZADD leaderboard 9850 "player:1"
ZADD leaderboard 7200 "player:2"
ZREVRANGE leaderboard 0 9 WITHSCORES   # top 10 with scores (desc)
ZINCRBY leaderboard 100 "player:2"     # atomic score increment

# Set — unique visitors
SADD visitors:2026-03-12 "user:42" "user:99"
SCARD visitors:2026-03-12  # count distinct visitors today

# HyperLogLog — distinct count at scale (~0.81% error)
PFADD hll:unique-visitors "user:1" "user:2" "user:1"  # deduped
PFCOUNT hll:unique-visitors  # 2 (not 3 — user:1 is counted once)
```

- **Technical Terms to Include:** in-memory, sub-millisecond latency, String, Hash, List, Set, Sorted Set, Stream, HyperLogLog, Bitmap, Vector Set (8.0), RESP protocol, single-threaded command processing, I/O multiplexing
- **Gotcha:** Redis is **single-threaded for command processing** (though multi-threaded for I/O since Redis 6). A slow command (e.g., `KEYS *`, `SMEMBERS` on a large set, `HGETALL` on a large hash) blocks all other commands while it executes. Never use `KEYS *` in production — use `SCAN` (cursor-based, non-blocking). Never store unbounded collections in a single key — use time-bucketed keys or pagination.
- **Follow-Up:** "What is the time complexity of common Redis operations?" → Strings (`GET`/`SET`): O(1). Hash (`HGET`/`HSET`): O(1). List (`LPUSH`/`RPOP`): O(1). Set (`SADD`/`SISMEMBER`): O(1). Sorted Set (`ZADD`/`ZSCORE`): O(log N). Sorted Set range (`ZRANGE`): O(log N + M where M is returned elements). `SMEMBERS` (full set): O(N). The O(N) operations are the dangerous ones in production — always bound your data sizes. → They're testing Big-O operational awareness for Redis commands.
- **Conclusion:** Redis's value as a platform — not just a cache — comes from its data structures providing the right primitives for each problem: ZSets for leaderboards, Streams for event queues, HyperLogLog for cardinality, Pub/Sub for messaging — each replacing what would otherwise be a separate specialised system.

---

#### Caching Patterns

- **Question:** What are the four caching patterns — cache-aside, read-through, write-through, write-behind — and when does each apply?
- **Answer:**
  - **Cache-aside (lazy loading):** The application checks the cache first. Cache miss → fetch from DB → populate cache → return. Most common pattern. Cache only contains what's been requested (no wasted memory). Trade-off: first request is always a cache miss (cold start). Cache miss under high load causes a **thundering herd** — many requests hit the DB simultaneously for the same cold key.
  - **Read-through:** The cache sits in front of the database. On cache miss, the cache itself fetches from the DB, populates itself, and returns the data. Application always talks to the cache. Simpler application code; cache vendor must support it.
  - **Write-through:** Every write goes to both cache and DB synchronously. Cache is always up-to-date. Trade-off: write latency doubles; cache contains data that may never be read (write amplification).
  - **Write-behind (write-back):** Writes go to cache immediately; DB write is asynchronous (queued). Fast writes; trade-off: data loss risk if cache crashes before async flush. Use when write throughput is critical and occasional loss is acceptable.
  - **Thundering herd prevention:**

    (1) **Cache stampede lock** — first requester acquires a distributed lock (Redis `SET NX EX`), fetches from DB, populates cache. Other requesters wait.

    (2) **Probabilistic early expiry** — recompute cache before it expires (with some probability).

    (3) **Stale-while-revalidate** — serve stale data while refreshing in background.

- **Example:**

```python
import redis
r = redis.Redis(host='redis', port=6379, decode_responses=True)

# Cache-aside pattern
def get_user(user_id: str):
    cache_key = f"user:{user_id}"

    # Check cache first
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    # Cache miss — fetch from DB
    user = db.users.find_by_id(user_id)
    if user:
        r.setex(cache_key, 3600, json.dumps(user))  # cache for 1 hour
    return user

# Thundering herd prevention — distributed lock on cache miss
def get_user_safe(user_id: str):
    cache_key = f"user:{user_id}"
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    lock_key = f"lock:user:{user_id}"
    lock = r.set(lock_key, "1", nx=True, ex=5)  # atomic SET if Not eXists, 5s TTL
    if lock:
        # This requester owns the lock — fetch and populate
        user = db.users.find_by_id(user_id)
        r.setex(cache_key, 3600, json.dumps(user))
        r.delete(lock_key)
        return user
    else:
        # Another requester is fetching — wait and retry
        time.sleep(0.1)
        return get_user_safe(user_id)  # retry — will hit cache
```

- **Technical Terms to Include:** cache-aside, read-through, write-through, write-behind, thundering herd, cache stampede, `SET NX EX`, TTL, cache eviction, cold start, stale-while-revalidate, cache invalidation, cache warming
- **Gotcha:** **Cache invalidation is hard.** Setting a TTL and letting data expire is the simple path — but stale data within the TTL window is the trade-off. For correctness-critical data, invalidate explicitly on write: delete the cache key when the DB record changes (`cache.delete(key)` in the same transaction). "There are only two hard things in computer science: cache invalidation and naming things." Interviewers probe this directly.
- **Follow-Up:** "What is the cache penetration problem and how do you solve it?" → Cache penetration: requests for a key that doesn't exist in the cache or the DB (e.g., malicious requests with random IDs). Every request misses the cache and hits the DB — DoS via cache bypass. Solutions: (1) Cache the null result (`SET missing-key "NULL" EX 60`) — serves the null from cache for 60s. (2) Bloom filter — pre-screen requests against a probabilistic membership check; if definitely not in DB, reject before cache/DB. → They're testing cache security and negative-result caching.
- **Conclusion:** Caching patterns are not implementation details — they are architectural decisions about consistency, write latency, memory efficiency, and failure modes; choosing the wrong pattern for a workload's access pattern is how you introduce either stale data bugs or unnecessary DB pressure.

---

#### Expiry, Eviction & Persistence

- **Question:** How does Redis handle key expiry, what are the eviction policies, and which persistence mode should you choose?
- **Answer:**
  - **Key expiry:** `EXPIRE key seconds` or `SET key value EX seconds`. Redis checks expiry via two mechanisms:
    (1) Lazy deletion — key expires only when accessed.
    (2) Active deletion — Redis samples 20 random keys every 100ms and deletes expired ones. A key may remain in memory for up to seconds after its TTL if never accessed and not sampled.
  - **Eviction policies** (when `maxmemory` is reached):
    (1) `noeviction` — reject writes (default).
    (2) `allkeys-lru` — evict least recently used key from all keys.
    (3) `volatile-lru` — LRU eviction only from keys with TTL set.
    (4) `allkeys-lfu` — evict least frequently used (Redis 4.0+).
    (5) `volatile-ttl` — evict keys with shortest TTL first. For a pure cache: `allkeys-lru` or `allkeys-lfu`. For a mixed cache/store: `volatile-lru`.
  - **Persistence modes:**
    (1) **RDB (Redis Database)** — point-in-time snapshots at intervals. Fast restart, compact file. Data loss between snapshots.
    (2) **AOF (Append Only File)** — logs every write command. Configurable fsync: `always` (every write), `everysec` (every second), `no` (OS decides). `everysec` is the recommended default — at most 1 second of data loss.
    (3) **AOF+RDB (hybrid, Redis 7.0+)** — starts with RDB snapshot then appends AOF — fast restart + minimal data loss. Recommended for production.
- **Example:**

```bash
# Expiry
SET session:abc123 "{...}" EX 3600   # expire in 1 hour
TTL session:abc123                    # check remaining TTL
PERSIST session:abc123                # remove TTL — make permanent

# Eviction policy
# redis.conf
maxmemory 2gb
maxmemory-policy allkeys-lfu          # evict least frequently used

# Persistence config
save 900 1        # RDB: snapshot if 1 key changed in 900s
save 300 10       # RDB: snapshot if 10 keys changed in 300s
appendonly yes    # enable AOF
appendfsync everysec  # fsync every second — at most 1s data loss

# Check persistence status
redis-cli INFO persistence
# rdb_last_bgsave_status: ok
# aof_enabled: 1
# aof_last_write_status: ok
```

- **Technical Terms to Include:** TTL, `EXPIRE`, `PERSIST`, lazy deletion, active expiry sampling, `maxmemory`, eviction policy, LRU, LFU, RDB, AOF, fsync, `appendfsync`, hybrid persistence, `BGSAVE`, `BGREWRITEAOF`
- **Gotcha:** Redis's `maxmemory` setting is **not** a hard limit at the OS level — it's enforced by Redis's own eviction logic before accepting writes. If Redis is configured for `noeviction` and `maxmemory` is reached, every write command returns an OOM error. In a microservices environment where Redis is used as a session store with `noeviction`, a memory leak or spike can suddenly cause all session writes to fail — application-wide login failures. Always pair `noeviction` with memory alerts, and `allkeys-lru` for pure cache use cases.
- **Follow-Up:** "What is the difference between LRU and LFU eviction?" → LRU (Least Recently Used): evicts the key that was last accessed furthest in the past — optimised for temporal locality (recently used = likely to be used again). LFU (Least Frequently Used): evicts the key accessed fewest times over its lifetime — optimised for frequency locality (popular keys survive longer). LFU is better for workloads with "hot" keys that should stay in cache; LRU is better for streaming access patterns where recent ≈ relevant. → They're testing eviction policy selection judgment.
- **Conclusion:** Redis's expiry, eviction, and persistence configuration form the operational reliability contract — wrong eviction policy causes hot key loss; wrong persistence mode causes data loss on restart; combining `appendfsync everysec` with RDB snapshots provides the best balance of performance and durability for most production deployments.

---

#### Redis Pub/Sub, Streams & Distributed Patterns

- **Question:** How do Redis Pub/Sub and Streams differ, and what production patterns does Redis enable beyond caching?
- **Answer:**
  - **Pub/Sub:** `PUBLISH channel message` / `SUBSCRIBE channel`. Fire-and-forget — no persistence. Subscribers receive only messages published while connected. No message history, no consumer groups. Use for: real-time notifications, live chat, broadcast events where loss is acceptable (dashboard updates, stock ticker). Not suitable for reliable event delivery.
  - **Streams (Redis 5.0+):** Append-only log with persistent storage, consumer groups, acknowledgement, and replay. `XADD stream * field value` appends. `XREADGROUP GROUP g1 consumer1 COUNT 10 STREAMS stream >` reads undelivered messages. `XACK stream g1 message-id` acknowledges. A Kafka-lite for lower throughput use cases (thousands/sec vs millions/sec for Kafka). Retains messages; supports multiple consumer groups; supports replay from any offset.
  - **Distributed lock (`SET NX EX`):** `SET lock-key value NX EX 30` — atomically set only if not exists, with 30s TTL. If returns `OK`, lock acquired. Prevents concurrent execution across services. Use Redlock algorithm for multi-node Redis HA (controversial — understand the debate). Release lock only if value matches (Lua script for atomicity).
  - **Rate limiting (Sorted Set sliding window):** Store request timestamps in a ZSet per user. `ZREMRANGEBYSCORE` removes old timestamps. `ZCARD` counts current requests in window. If count < limit → add timestamp, allow. Atomic via Lua script.
- **Example:**

```python
# Pub/Sub — subscribe to channel
def listen_for_notifications():
    pubsub = r.pubsub()
    pubsub.subscribe('notifications')
    for message in pubsub.listen():
        if message['type'] == 'message':
            handle(message['data'])

# Streams — reliable event log with consumer groups
# Producer
r.xadd('events', {'event': 'order-paid', 'order_id': '42'})

# Consumer group
r.xgroup_create('events', 'processors', id='0', mkstream=True)
messages = r.xreadgroup('processors', 'worker-1', {'events': '>'}, count=10)
for stream, msgs in messages:
    for msg_id, data in msgs:
        process(data)
        r.xack('events', 'processors', msg_id)  # acknowledge

# Distributed lock — SET NX EX
import uuid
lock_value = str(uuid.uuid4())  # unique value to prevent accidental release
acquired = r.set('lock:resource', lock_value, nx=True, ex=30)
if acquired:
    try:
        do_critical_work()
    finally:
        # Atomic release — only if we own the lock (Lua script)
        release_script = """
        if redis.call('GET', KEYS[1]) == ARGV[1] then
            return redis.call('DEL', KEYS[1])
        else return 0 end
        """
        r.eval(release_script, 1, 'lock:resource', lock_value)
```

- **Technical Terms to Include:** Pub/Sub, Streams, XADD, XREADGROUP, XACK, consumer group, distributed lock, SET NX EX, Redlock, Lua script, rate limiting, sliding window, session store, leaderboard, geospatial
- **Gotcha:** The Redlock distributed locking algorithm (lock acquired on majority of N Redis nodes) is **controversial** — Martin Kleppmann published a detailed critique showing it does not provide safety guarantees under certain timing/GC pause conditions. For most use cases, single-node `SET NX EX` with a sufficiently short TTL is sufficient and simpler. Use Redlock only when you understand its limitations and have formally modelled your system's safety requirements.
- **Follow-Up:** "How is Redis Streams different from Pub/Sub for reliable message processing?" → Pub/Sub: no persistence, missed if not connected, no replay. Streams: persistent, consumer groups track per-consumer offsets, unacknowledged messages are retried, replay from any point. For reliable processing (order fulfillment, payment events), Streams. For ephemeral notifications (live dashboard updates, chat typing indicator), Pub/Sub. → They're testing Redis messaging primitive selection.
- **Conclusion:** Redis's production value extends far beyond caching into distributed primitives — Streams for reliable event queuing, `SET NX EX` for distributed locking, Sorted Sets for rate limiting and leaderboards — making Redis the Swiss Army knife of backend infrastructure that a single Redis cluster often replaces multiple specialised services.

---

# SECTION 2 — gRPC v1.71

---

**L1–L4 Topics identified:**

- What is gRPC / Protobuf / HTTP/2
- Service Definition — .proto files, code generation
- Communication patterns — Unary / Server streaming / Client streaming / Bidirectional
- Interceptors — middleware in gRPC
- Error handling — status codes, error details
- Load balancing & service discovery
- gRPC vs REST decision

---

### Core Architecture

#### What is gRPC / Protobuf / HTTP/2

- **Question:** What is gRPC, how does Protobuf serialisation work, and what does HTTP/2 enable that HTTP/1.1 cannot?
- **Answer:**
  - gRPC is a high-performance, open-source RPC (Remote Procedure Call) framework from Google. It uses **Protocol Buffers (Protobuf)** for serialisation and **HTTP/2** as the transport. It enables calling methods on a remote service as if calling local functions — with generated client and server stubs in any supported language.
  - **Protocol Buffers (Protobuf):** A binary serialisation format. Messages are defined in `.proto` files with typed fields and field numbers. Serialised to compact binary — significantly smaller than JSON (no field names in payload, binary encoding of numbers). Faster to serialise/deserialise. Backward and forward compatible — unknown fields are preserved. Requires proto file sharing between producer and consumer (schema-first).
  - **HTTP/2 advantages:**
    (1) **Multiplexing** — multiple concurrent RPC calls over a single TCP connection (no head-of-line blocking per stream).
    (2) **Binary framing** — headers and data in binary, more efficient than HTTP/1.1 text headers.
    (3) **Header compression (HPACK)** — repeated headers (like auth tokens) compressed.
    (4) **Server push** — server can proactively send data.
    (5) **Bidirectional streaming** — both sides can send independent streams simultaneously.
  - **Code generation:** `protoc` (protoc compiler) with language-specific plugins generates client stubs and server interfaces from `.proto` files. The generated code handles serialisation, transport, and retry logic — engineers implement only the business logic.
- **Example:**

```protobuf
// user.proto — schema-first service definition
syntax = "proto3";
package user.v1;

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc ListUsers(ListUsersRequest) returns (stream UserResponse); // server streaming
  rpc WatchUser(stream WatchRequest) returns (stream UserEvent); // bidi streaming
}

message GetUserRequest {
  string user_id = 1;  // field number 1 — compact binary key
}

message GetUserResponse {
  User user = 1;
}

message User {
  string id    = 1;
  string email = 2;
  string name  = 3;
  int64  created_at = 4;  // epoch ms — no date parsing overhead
}
```

```bash
# Code generation
protoc --go_out=. --go-grpc_out=. user.proto
# Generates: user.pb.go (types), user_grpc.pb.go (server interface + client stub)
```

```go
// Server implementation
type UserServiceServer struct {
    db *Database
    pb.UnimplementedUserServiceServer  // forward compatibility
}

func (s *UserServiceServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.GetUserResponse, error) {
    user, err := s.db.FindUser(ctx, req.UserId)
    if err != nil {
        return nil, status.Errorf(codes.NotFound, "user %s not found: %v", req.UserId, err)
    }
    return &pb.GetUserResponse{User: toProto(user)}, nil
}
```

- **Technical Terms to Include:** gRPC, Protocol Buffers (Protobuf), `.proto` file, `protoc`, code generation, HTTP/2, multiplexing, binary framing, HPACK header compression, field number, schema-first, RPC, stub, channel
- **Gotcha:** Protobuf **field numbers, not names, identify fields in the binary format.** Renaming a field in the `.proto` file is backward compatible — the binary encoding is unchanged (the field number is what's serialised). **Changing or reusing a field number is binary-breaking** — existing serialised data is misinterpreted. Always reserve removed field numbers: `reserved 5;` — prevents accidental reuse.
- **Follow-Up:** "How does Protobuf backward and forward compatibility work?" → Backward compatibility: old code reading new messages — ignores unknown fields (new fields added with new numbers). Forward compatibility: new code reading old messages — new required fields absent → zero value. This is why `proto3` removed `required` fields — they break forward compatibility (old clients don't send them). Use `optional` semantics and check for presence with `HasField`. → They're testing schema evolution discipline.
- **Conclusion:** gRPC's combination of Protobuf (compact, typed, versioned binary serialisation) and HTTP/2 (multiplexed, bidirectional streaming) makes it the default choice for high-throughput, low-latency service-to-service communication — but schema-first development and strict field number discipline are the operational disciplines that make it maintainable at scale.

---

#### Communication Patterns & Error Handling

- **Question:** What are the four gRPC communication patterns, when is each used, and how does gRPC error handling work?
- **Answer:**
  - **Unary RPC:** Single request → single response. The standard request/response pattern. `rpc GetUser(Request) returns (Response)`. Most common — use for CRUD operations, lookups, one-shot commands.
  - **Server-side streaming:** Single request → stream of responses. `rpc ListOrders(Filter) returns (stream Order)`. Server sends multiple responses over the same connection until done. Use for: large dataset pagination, real-time feed of matching events, file download in chunks.
  - **Client-side streaming:** Stream of requests → single response. `rpc UploadLogs(stream LogEntry) returns (UploadSummary)`. Client streams data; server responds once. Use for: file upload, bulk data ingestion, telemetry batching.
  - **Bidirectional streaming:** Stream of requests + stream of responses, fully independent. `rpc Chat(stream Message) returns (stream Message)`. Use for: real-time chat, collaborative editing, live game state synchronisation, long-polling-free push.
  - **Error handling:** gRPC uses status codes (not HTTP codes) defined in `google.rpc.Code`: `OK`, `NOT_FOUND`, `INVALID_ARGUMENT`, `UNAUTHENTICATED`, `PERMISSION_DENIED`, `UNAVAILABLE`, `INTERNAL`, `DEADLINE_EXCEEDED`, `RESOURCE_EXHAUSTED`. Rich error details via `google.rpc.Status` (error details: `BadRequest`, `RetryInfo`, `ErrorInfo`). Clients should handle `UNAVAILABLE` with retry; `DEADLINE_EXCEEDED` with context cancellation; never retry `INVALID_ARGUMENT` (idempotent but wrong input won't improve).
- **Example:**

```go
// Server streaming — stream large result set
func (s *Server) ListOrders(req *pb.ListRequest, stream pb.OrderService_ListOrdersServer) error {
    cursor := db.NewOrderCursor(req.Filter)
    for cursor.HasNext() {
        order := cursor.Next()
        if err := stream.Send(toProto(order)); err != nil {
            return err  // client disconnected
        }
        // Check context cancellation — respect client timeout
        if stream.Context().Err() != nil {
            return status.Error(codes.Canceled, "client cancelled")
        }
    }
    return nil
}

// Bidirectional streaming — real-time updates
func (s *Server) WatchOrders(stream pb.OrderService_WatchOrdersServer) error {
    for {
        req, err := stream.Recv()  // receive from client
        if err == io.EOF { return nil }
        if err != nil { return err }

        // Send event back — independent of receive
        event := computeEvent(req)
        if err := stream.Send(event); err != nil { return err }
    }
}

// Client error handling
resp, err := client.GetUser(ctx, &pb.GetUserRequest{UserId: id})
if err != nil {
    st, ok := status.FromError(err)
    if !ok { return fmt.Errorf("non-gRPC error: %w", err) }
    switch st.Code() {
    case codes.NotFound:
        return ErrUserNotFound
    case codes.Unavailable:
        // retry with backoff
    case codes.DeadlineExceeded:
        // request timed out — don't retry
    }
}
```

- **Technical Terms to Include:** unary RPC, server streaming, client streaming, bidirectional streaming, status code, `google.rpc.Code`, `google.rpc.Status`, `DEADLINE_EXCEEDED`, `UNAVAILABLE`, `NOT_FOUND`, interceptor, deadline propagation, context cancellation
- **Gotcha:** **Deadline propagation is not automatic across service boundaries.** If service A has a 5-second deadline and calls service B, service B must receive and respect the same deadline via context propagation. In gRPC, the deadline is sent in the HTTP/2 header and the framework propagates it to `ctx`. But if service B makes a DB call without passing `ctx`, the deadline is silently lost — service B continues work even after the client has already timed out and moved on, wasting resources.
- **Follow-Up:** "How does gRPC handle retries and what is the retry policy?" → gRPC supports client-side retry policies defined in service config (JSON, loaded from service discovery or channel config). Configurable: `maxAttempts`, `initialBackoff`, `maxBackoff`, `backoffMultiplier`, `retryableStatusCodes` (typically `UNAVAILABLE`, `INTERNAL`). Hedging (send duplicate requests after a delay, use first response) is also supported. Never retry `INVALID_ARGUMENT` or `NOT_FOUND` — the input is wrong, retrying won't help. → They're testing gRPC resilience configuration depth.
- **Conclusion:** gRPC's four communication patterns — unary, server streaming, client streaming, bidirectional — cover the full spectrum of service interaction semantics; its structured error codes enforce contract clarity between services, and deadline propagation through context is the operational discipline that prevents cascading timeouts in deep call chains.

---

#### gRPC vs REST — Decision Framework

- **Question:** When do you choose gRPC over REST, and what are the trade-offs at an architectural level?
- **Answer:**
  - **Choose gRPC when:**
    - **Internal service-to-service communication** — both sides own the client/server code, proto file sharing is feasible.
    - **Performance is critical** — Protobuf is 3-10x smaller than JSON, binary parsing is faster, HTTP/2 multiplexing reduces connection overhead.
    - **Streaming is needed** — bidirectional streaming is native in gRPC, awkward in REST (WebSockets, SSE, long polling).
    - **Strongly typed contracts** — generated code eliminates serialisation bugs; contract changes are compile-time errors.
    - **Polyglot microservices** — code generation supports 10+ languages from one `.proto` file.
  - **Choose REST when:**
    - **Public API** — clients are browsers, mobile apps, third-party developers who cannot/will not manage proto files. REST + JSON + OpenAPI is universally understood.
    - **Browser clients** — gRPC over HTTP/2 requires gRPC-Web (proxy layer) in browsers; native gRPC is not browser-supported.
    - **Human readability matters** — JSON is debuggable with `curl`; Protobuf binary requires tooling.
    - **Cache-friendly resources** — REST leverages HTTP caching (`Cache-Control`, CDN) natively; gRPC POST requests are not cached by HTTP infrastructure.
    - **Ecosystem maturity** — REST has decades of tooling (Postman, OpenAPI, API gateways, rate limiting middleware); gRPC tooling is improving but narrower.

| Factor                | gRPC                          | REST                          |
| --------------------- | ----------------------------- | ----------------------------- |
| Payload size          | Compact binary (Protobuf)     | JSON text (3-10x larger)      |
| Browser support       | Needs gRPC-Web proxy          | Native                        |
| Streaming             | Native (4 patterns)           | Workarounds (SSE, WS)         |
| Schema enforcement    | Compile-time (generated code) | Runtime (optional validation) |
| Human debuggability   | Needs tooling                 | `curl` / browser              |
| Caching               | Not HTTP-cache-able           | Full HTTP cache support       |
| Public API            | Not recommended               | Standard                      |
| Microservice internal | Recommended                   | Viable                        |

- **Technical Terms to Include:** REST, gRPC, Protobuf, JSON, OpenAPI, gRPC-Web, Envoy, HTTP/2 vs HTTP/1.1, schema-first, code generation, type safety, API gateway, service mesh, Istio, transcoding
- **Gotcha:** gRPC is not natively supported in browsers — HTTP/2 requires control over headers that browsers restrict. The solution is **gRPC-Web** (a protocol variant) with a proxy (Envoy, nginx, or Envoy-based API gateway) that translates gRPC-Web to gRPC. This adds operational complexity. For browser clients, REST or GraphQL is architecturally simpler — gRPC shines in server-to-server communication.
- **Follow-Up:** "Can you use both gRPC and REST in the same service?" → Yes — via gRPC transcoding. A `.proto` service definition annotated with HTTP bindings generates both a gRPC server and a REST/JSON HTTP endpoint from the same handler code. Google Cloud's API Gateway and Envoy support this natively. This enables a single service implementation exposing gRPC for internal services and REST for external consumers without duplicating logic. → They're testing multi-protocol API architecture awareness.
- **Conclusion:** gRPC is the right default for internal microservice communication — its performance, type safety, and streaming capabilities are unmatched; REST is the right default for public APIs — its universal browser support, human readability, and cache-friendliness are not replicated by gRPC without significant additional infrastructure.

---

## Cross-Stack Patterns

| Pattern               | Tools                                         | Notes                                                 |
| --------------------- | --------------------------------------------- | ----------------------------------------------------- |
| Caching API responses | Redis (cache-aside) + any service             | TTL-based invalidation or event-driven `DEL` on write |
| Service communication | gRPC (internal) + REST (external)             | Transcoding for dual-protocol                         |
| Rate limiting         | Redis Sorted Set sliding window               | Atomic Lua script for correctness                     |
| Distributed lock      | Redis `SET NX EX`                             | Redlock for multi-node HA (with caveats)              |
| Pub/Sub notifications | Redis Pub/Sub or Kafka (if persistent)        | Redis for ephemeral; Kafka for durable                |
| Session store         | Redis (Hash or String with TTL)               | `allkeys-lru` eviction for pure session cache         |
| Queue-based work      | RabbitMQ (task queue) or Kafka (event stream) | RabbitMQ for routing; Kafka for replay/fan-out        |
| Metrics + alerting    | Prometheus + Alertmanager                     | PromQL recording rules for SLO                        |
| Log search            | Elasticsearch + Kibana                        | ILM for retention; `bool` filter for structured       |

---

_— End of DataComms_Interview_QA.md —_
_Coverage: Redis (L1–L5, 4 QA units) | gRPC (L1–L4, 3 QA units)_
_Calibrated for: Tech Lead / Architect level_
