<!-- Generated: 2026-04-12 | MH: Golang,AWS | GTH: none | Depth: L1-L6 -->

# Persistent Systems — Golang Developer

## Golang Interview Prep | L1–L6

---

## 🆕 What's New — Last 3 Versions

### Go 1.24 — February 2025

- **Generic Type Aliases:** Full support for parameterized type aliases, enabling cleaner generic library design — interviewers at architecture level will ask about this directly.
- **Swiss Table Map Implementation:** Go's built-in map switched to a Swiss Table-based hash map, improving memory efficiency and lookup performance — signals Go team's continued runtime investment.
- **`os.Root` for Sandboxed File I/O:** New API for scoped filesystem access — relevant in container/serverless contexts Persistent deploys to.
- **Deprecated:** `testing.B.N` direct assignment pattern officially discouraged in favour of `b.Loop()`.

### Go 1.23 — August 2024

- **Range over Functions (Iterator Protocol):** `range` can now iterate over functions with `func(yield func(K, V) bool)` signatures — foundational shift in how Go handles custom iterables.
- **Timer/Ticker Reset Behaviour Fixed:** `time.Timer` and `time.Ticker` reset semantics corrected — previously a source of subtle race conditions in production timers.
- **`unique` Package:** Canonical interning of comparable values — reduces allocations in high-throughput string-heavy services.

### Go 1.22 — February 2024

- **Loop Variable Capture Fix:** Per-iteration loop variable semantics — eliminates the infamous goroutine-capture bug that plagued concurrent code for a decade.
- **`math/rand/v2`:** Cryptographically cleaner API, rand.N replaces rand.Intn — signals intent to remove footguns from stdlib.
- **HTTP Routing Enhancements:** Method-based and wildcard routing in `net/http` — reduces dependency on third-party routers for standard API servers.

> **Interview signal:** A candidate referencing Go 1.22's loop variable fix unprompted signals they have debugged real concurrency bugs — not just read about them.

---

## L1 — Conceptual Foundations

> 📊 **L1:** 3 categories · 4 topics · 4 questions

---

### Category: Go Philosophy

> 📁 2 topics · 2 questions

#### Topic: Design Goals and Language Philosophy — (Q count: 1)

- **[L1.GoPhilosophy.DesignGoalsAndLanguagePhilosophy.Q1]**
  **Question:** Why did Google create Go, and what design constraints shape every decision in the language?
- **Answer:**
  - Go was designed to solve Google's specific pain: slow C++ compile times, verbose Java boilerplate, and Python's performance ceiling — the goal was a systems language with the productivity of a scripting language.
  - The three pillars are: fast compilation (dependency graph, no circular imports), safety without a type hierarchy (structural typing via interfaces), and built-in concurrency primitives (goroutines and channels as first-class citizens).
  - Explicit over implicit: Go has no exceptions, no function overloading, no default parameters, no inheritance — every control flow is visible in the code.
  - Simplicity is a feature: the language spec fits in a single page; this reduces cognitive overhead in large codebases with hundreds of contributors.
  - Opinionated tooling (gofmt, go vet, go build) enforces consistency at the ecosystem level — reducing bikeshedding at code review.
- **Example:** At TCS on the Apple Offers platform, integrating with a GoLang-based centralized platform service required clear API contracts at the boundary — Go's explicit error returns made every failure path visible in the consuming frontend, unlike Java exceptions which could be silently swallowed.
- **Technical Terms to Include:** structural typing, static compilation, CSP concurrency, opinionated tooling, single binary deployment
- **Gotcha:** "Go is a simple language" — interviewers probe whether you conflate "simple spec" with "simple to use at scale." Concurrency is famously simple to write incorrectly.
- **Follow-Up:** "What does Go sacrifice compared to Rust?" → Memory safety without a borrow checker: Go uses GC and accepts latency pauses instead. They ask this to calibrate your understanding of trade-off space.
- **Conclusion:** Go's design choices are deliberate constraints — understanding them tells you exactly when Go is the right tool and when it is not.

---

#### Topic: Compilation Model — (Q count: 1)

- **[L1.GoPhilosophy.CompilationModel.Q1]**
  **Question:** Walk me through what happens from `go build` to a running binary, and why Go compiles faster than C++.
- **Answer:**
  - Go compiles each package independently; the compiler reads only the exported interface of dependencies, not their full source — this is the fundamental reason compilation is fast.
  - Circular imports are forbidden — the dependency graph is always a DAG, enabling parallel compilation across packages.
  - The result is a single statically linked binary: all dependencies (including the Go runtime and standard library) are embedded — no shared library resolution at startup.
  - No header files: Go's `import` reads compiled package metadata, not raw source — eliminating C/C++'s preprocessor pass.
  - The Go runtime bootstraps goroutine scheduling, GC, and the memory allocator before `main()` executes.
- **Example:** Deploying a Go microservice to AWS Lambda produces a single binary; no runtime installation required in the container image — cold start is dominated by network, not binary loading.
- **Technical Terms to Include:** DAG dependency graph, static linking, package-level compilation, go runtime bootstrap, single binary
- **Gotcha:** "Static linking means large binaries" — probe: Go binaries are larger than C but smaller than JVM fat JARs, and eliminate dependency drift entirely.
- **Follow-Up:** "How does `go build` differ from `go install`?" → `install` caches the compiled binary to `$GOPATH/bin`; `build` outputs to current dir. Asked to check toolchain fluency.
- **Conclusion:** Go's compilation speed is architectural, not just an implementation optimization — the language was designed to compile fast.

---

### Category: Concurrency Model

> 📁 1 topic · 1 question

#### Topic: Goroutines vs OS Threads — (Q count: 1)

- **[L1.ConcurrencyModel.GoroutinesVsOSThreads.Q1]**
  **Question:** What is a goroutine, and why is it fundamentally different from an OS thread?
- **Answer:**
  - A goroutine is a user-space, cooperatively and preemptively scheduled unit of execution managed by the Go runtime — not the operating system kernel.
  - Initial stack size is ~2KB (vs 1–8MB for OS threads) and grows dynamically via stack segmentation/copying — you can run hundreds of thousands of goroutines without hitting OS limits.
  - Goroutines are multiplexed onto OS threads by the Go runtime scheduler (GMP model: G=goroutine, M=machine/OS thread, P=processor); the number of OS threads is bounded by `GOMAXPROCS`.
  - Context switching between goroutines happens in user space — no kernel syscall needed for goroutine scheduling, making it orders of magnitude cheaper than thread switching.
  - Goroutines are not free: each allocation, stack growth, and GC tracking has cost — "start a goroutine for every request" without pooling is a real production anti-pattern.
- **Example:** At PayPal's BFF layer, fan-out to 4 upstream services (payment, billing, subscription, user/identity) was modelled as 4 concurrent goroutines with a sync.WaitGroup — this pattern would be prohibitively expensive with OS threads at checkout scale.
- **Technical Terms to Include:** GMP model, GOMAXPROCS, user-space scheduling, cooperative/preemptive scheduling, stack growth, goroutine leak
- **Gotcha:** "Goroutines are lightweight so just spawn them" — real trap: goroutines that block on channels or I/O without being collected cause goroutine leaks, detectable only with `runtime.NumGoroutine()` or pprof.
- **Follow-Up:** "What is a goroutine leak and how do you detect it?" → A goroutine blocked indefinitely on a channel/lock that is never released. Detected via `pprof` goroutine profile or monitoring `runtime.NumGoroutine()`. Asked because leaks are the #1 silent failure in Go services.
- **Conclusion:** Goroutines are cheap to create but must be explicitly terminated — the absence of a lifecycle mechanism is Go's sharpest production edge.

---

### Category: Type System

> 📁 1 topic · 1 question

#### Topic: Interfaces and Structural Typing — (Q count: 1)

- **[L1.TypeSystem.InterfacesAndStructuralTyping.Q1]**
  **Question:** How does Go's interface system work, and what does "structural typing" mean in practice?
- **Answer:**
  - In Go, a type implements an interface implicitly — if the type has all the methods the interface declares, it satisfies it, with no `implements` keyword required.
  - This is structural (duck) typing: the compiler checks structure (method signatures), not declared intent — decoupling the implementor from the consumer.
  - Interfaces are zero-cost abstractions: calling a method through an interface involves two pointer dereferences (type pointer + value pointer in the interface header), not a vtable lookup like C++.
  - The empty interface `interface{}` (or `any` in Go 1.18+) accepts all types — use sparingly; it bypasses compile-time type safety.
  - Composition over inheritance: small, single-method interfaces (`io.Reader`, `io.Writer`, `error`) compose into larger behaviours — the standard library's power derives from this.
- **Example:** In the Apple Offers platform, wrapping the Go backend API client behind an interface allowed swapping real and mock implementations in tests without changing business logic — zero test coupling to HTTP transport.
- **Technical Terms to Include:** structural typing, implicit satisfaction, interface header (type + value pointer), `io.Reader`, composition, `any`, compile-time safety
- **Gotcha:** "A nil pointer satisfying an interface is not a nil interface" — the most common source of nil-pointer panics in production Go code; the interface value holds a non-nil type pointer even when the value is nil.
- **Follow-Up:** "When should you not use interfaces?" → When there is only one implementation and no testing boundary — premature abstraction in Go creates indirection without value. Asked to test architectural judgment.
- **Conclusion:** Go interfaces enforce contracts without coupling — their power comes from being small, composable, and implicit.

---

## L2 — Core Syntax, APIs, and Fundamental Features

> 📊 **L2:** 3 categories · 4 topics · 4 questions

---

### Category: Data Structures

> 📁 2 topics · 2 questions

#### Topic: Slice Internals — (Q count: 1)

- **[L2.DataStructures.SliceInternals.Q1]**
  **Question:** What is a slice in Go at the memory level, and what are the production failure modes this causes?
- **Answer:**
  - A slice is a three-field struct: `{pointer to backing array, length, capacity}` — the slice header is 24 bytes on 64-bit; the data lives on the heap.
  - Appending beyond capacity triggers a new backing array allocation and copy — the old slice header's pointer becomes stale; callers holding the old slice see the old data.
  - `append` is not thread-safe: two goroutines appending to the same slice without synchronization causes a data race detectable by `go test -race`.
  - Slicing a slice (`s[2:5]`) creates a new header pointing into the same backing array — modifications through the sub-slice mutate the original; this is intentional but surprises most.
  - `copy()` forces an independent backing array; use it when passing slices to goroutines that will mutate them.
- **Example:** A logging middleware at the PayPal BFF that buffered request IDs in a shared `[]string` slice across goroutines caused a race condition under load. Fix: per-goroutine slice or channel-based aggregation.
- **Technical Terms to Include:** backing array, slice header, capacity, append, `copy()`, data race, `go test -race`
- **Gotcha:** "Nil slice vs empty slice" — `var s []int` (nil, len=0, cap=0) vs `s := []int{}` (non-nil, len=0, cap=0). JSON marshalling produces `null` vs `[]` respectively — a real API contract bug.
- **Follow-Up:** "How do you pre-allocate a slice for performance?" → `make([]T, 0, expectedLen)` — avoids repeated reallocation. Asked because unnecessary allocations are the primary Go heap pressure source.
- **Conclusion:** A slice is a view into a backing array — sharing that view across goroutines without coordination is the single most common Go concurrency bug.

---

#### Topic: Map Internals and Safety — (Q count: 1)

- **[L2.DataStructures.MapInternalsAndSafety.Q1]**
  **Question:** How are Go maps implemented, and what makes concurrent map access a runtime panic?
- **Answer:**
  - Go maps are hash tables with a bucket structure (8 slots per bucket); Go 1.24 switched to Swiss Tables internally for improved cache locality and lookup performance.
  - Map access is not goroutine-safe: concurrent reads are safe, but any concurrent write — even to different keys — triggers a fatal runtime panic: `concurrent map read and map write`.
  - This is a deliberate design decision: adding a lock to every map operation would penalise single-goroutine use cases — Go places the responsibility on the caller.
  - Thread-safe map use requires either a `sync.Mutex` wrapping the map or `sync.Map` (optimised for read-heavy, write-sparse workloads with stable key sets).
  - Map iteration order is explicitly randomised each run — code that depends on insertion order is wrong by design.
- **Example:** A request-context cache in a Go API handler that stored intermediate results in a shared `map[string]interface{}` across goroutines caused intermittent panics under load. Fix: `sync.Map` or per-request map with no sharing.
- **Technical Terms to Include:** hash bucket, Swiss Tables (Go 1.24), concurrent map panic, `sync.Map`, `sync.Mutex`, iteration order randomisation
- **Gotcha:** "I'll just use `sync.Map` everywhere" — `sync.Map` has higher overhead than a mutex-wrapped map for write-heavy or heterogeneous workloads; use it only for its specific use case.
- **Follow-Up:** "What is the zero value of a map, and can you write to it?" → `nil`; writing to a nil map panics. Reading from a nil map returns the zero value safely. Asked to check understanding of zero values.
- **Conclusion:** Go maps are fast but deliberately non-concurrent — the runtime makes the failure loud and immediate to prevent silent data corruption.

---

### Category: Concurrency Primitives

> 📁 1 topic · 1 question

#### Topic: Channels — (Q count: 1)

- **[L2.ConcurrencyPrimitives.Channels.Q1]**
  **Question:** What are the semantics of buffered vs unbuffered channels, and when does each cause a deadlock?
- **Answer:**
  - An unbuffered channel (`make(chan T)`) requires both sender and receiver to be ready simultaneously — it is a synchronisation point, not just a data transfer mechanism.
  - A buffered channel (`make(chan T, n)`) allows the sender to proceed without a receiver until the buffer is full — decouples producer and consumer temporally.
  - Deadlock with unbuffered: a goroutine sends to a channel with no receiver goroutine running — the scheduler detects all goroutines are blocked and panics with "all goroutines are asleep."
  - Deadlock with buffered: buffer is full, sender blocks; receiver never runs (e.g., closed before consuming) — a subtler production hang rather than an immediate panic.
  - Closing a channel is a broadcast: all goroutines blocked on `range ch` unblock when `close(ch)` is called — only the sender should close; closing a closed channel panics.
- **Example:** In the Apple Offers platform, distributing work across 3 frontend team goroutines used a buffered channel as a job queue — buffer size was tuned to `2 * numWorkers` to avoid sender blocking during brief worker spikes.
- **Technical Terms to Include:** unbuffered channel, buffered channel, deadlock, `close()`, `range` over channel, select statement, channel direction
- **Gotcha:** "Can you send on a closed channel?" → Panics immediately. "Can you receive from a closed channel?" → Returns zero value with `ok=false`. The asymmetry trips candidates who haven't debugged channel panics in production.
- **Follow-Up:** "How do you implement a timeout on a channel operation?" → `select { case v := <-ch: ... case <-time.After(d): ... }`. Asked because timeout handling is a production requirement, not a textbook exercise.
- **Conclusion:** Channels are synchronisation tools first and data pipes second — misunderstanding that ordering causes the majority of Go deadlocks.

---

### Category: Error Handling

> 📁 1 topic · 1 question

#### Topic: Go Error Handling Philosophy — (Q count: 1)

- **[L2.ErrorHandling.GoErrorHandlingPhilosophy.Q1]**
  **Question:** Why does Go not use exceptions, and how does the explicit error return pattern affect system reliability?
- **Answer:**
  - Go returns errors as values: `func Do() (Result, error)` — every error must be explicitly checked by the caller or explicitly discarded with `_`.
  - Exception-based languages allow errors to propagate invisibly through call stacks — a handler at the top may swallow context. Go's explicit pattern makes every failure path visible in the code.
  - `errors.Is()` and `errors.As()` (Go 1.13+) enable sentinel and type-based error matching without string comparison — critical for layered service error propagation.
  - `fmt.Errorf("context: %w", err)` wraps errors with context while preserving the original for `errors.Unwrap()` — enables error chains readable in Splunk/CloudWatch logs.
  - `panic/recover` is not a substitute for error handling — panic is for truly unrecoverable programmer errors (nil dereference, index out of bounds), not business logic failures.
- **Example:** In the PayPal BFF, each upstream service call returned `(Response, error)`; errors were wrapped with service identity (`fmt.Errorf("billing service: %w", err)`) before propagating — Splunk log queries could filter by originating service without parsing free-text.
- **Technical Terms to Include:** error value, `errors.Is`, `errors.As`, `%w` wrapping, `errors.Unwrap`, sentinel error, panic/recover distinction
- **Gotcha:** "Ignoring errors with `_` is fine in tests" — it is not fine in production code and is a code review red flag; even "impossible" errors should be logged.
- **Follow-Up:** "What is the difference between a sentinel error and a custom error type?" → Sentinel: `var ErrNotFound = errors.New("not found")` — comparison by value. Custom type: `type NotFoundError struct{ID string}` — matched by `errors.As`, carries context. Asked at architecture level to test design depth.
- **Conclusion:** Go's explicit error returns turn failure paths into first-class code — the verbosity is the feature, not the bug.

---

## L3 — Common Patterns, Real-World Usage

> 📊 **L3:** 3 categories · 4 topics · 4 questions

---

### Category: Concurrency Patterns

> 📁 2 topics · 2 questions

#### Topic: Worker Pool Pattern — (Q count: 1)

- **[L3.ConcurrencyPatterns.WorkerPoolPattern.Q1]**
  **Question:** Design a worker pool in Go that processes jobs from a queue with bounded concurrency, and explain the design decisions.
- **Answer:**
  - A worker pool consists of: a buffered job channel (bounded queue), N goroutines reading from it (workers), and a `sync.WaitGroup` to signal completion.
  - Bounded concurrency prevents goroutine explosion — without it, a spike of 10,000 requests spawns 10,000 goroutines, exceeding memory and scheduler bounds.
  - The job channel buffer decouples the producer (enqueuer) from workers — producer blocks only when the buffer is full, providing natural backpressure.
  - Closing the job channel signals workers to drain and exit — `range jobCh` unblocks when closed, eliminating explicit done-channel plumbing in simple pools.
  - Result collection uses a separate results channel or a `sync.Mutex`-protected slice — never write to a shared slice from multiple workers without synchronisation.
- **Example:**

```go
func workerPool(jobs []Job, numWorkers int) []Result {
    jobCh := make(chan Job, len(jobs))
    resultCh := make(chan Result, len(jobs))
    var wg sync.WaitGroup

    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobCh {
                resultCh <- process(job)
            }
        }()
    }

    for _, j := range jobs {
        jobCh <- j
    }
    close(jobCh)

    go func() { wg.Wait(); close(resultCh) }()

    var results []Result
    for r := range resultCh { results = append(results, r) }
    return results
}
```

- **Technical Terms to Include:** bounded concurrency, backpressure, `sync.WaitGroup`, job channel, drain pattern, goroutine pool
- **Gotcha:** "What if a worker panics?" → Without `recover`, a panicking goroutine crashes the entire process. Production worker pools wrap each job execution in a deferred recover.
- **Follow-Up:** "How would you add cancellation to this pool?" → Pass a `context.Context`; workers select on both `jobCh` and `ctx.Done()`. Asked because context propagation is a prod requirement.
- **Conclusion:** A worker pool is the Go pattern for transforming unbounded goroutine spawning into bounded, observable concurrency.

---

#### Topic: Context Package — (Q count: 1)

- **[L3.ConcurrencyPatterns.ContextPackage.Q1]**
  **Question:** What is `context.Context` and why is it the correct mechanism for cancellation, deadlines, and request-scoped values in Go services?
- **Answer:**
  - `context.Context` is an immutable tree of values, cancellation signals, and deadlines that flows through call chains — it is the idiomatic mechanism for propagating request lifecycle across goroutines and service boundaries.
  - `context.WithCancel` creates a child context with a `cancel()` function — calling cancel signals all goroutines watching `ctx.Done()` to stop work, preventing resource leaks.
  - `context.WithTimeout` and `context.WithDeadline` enforce time bounds — a downstream HTTP call that never times out will block a goroutine indefinitely, exhausting the pool.
  - `context.WithValue` carries request-scoped data (trace IDs, auth tokens) — the key must be an unexported custom type to prevent key collisions across packages.
  - Context should always be the first parameter: `func Do(ctx context.Context, ...)` — never store it in a struct; it is per-request, not per-service.
- **Example:** At the PayPal BFF, fan-out to 4 upstream services used a single `context.WithTimeout(ctx, 500ms)` — if any call exceeded the budget, all in-flight goroutines received the cancellation signal and released their connections, preventing a cascading timeout from upstream.
- **Technical Terms to Include:** `context.WithCancel`, `context.WithTimeout`, `ctx.Done()`, cancellation propagation, `context.WithValue`, request-scoped, context tree
- **Gotcha:** "Can you use `context.Background()` everywhere?" → In production handlers: no. `context.Background()` has no deadline or cancellation — a long-running request that times out at the HTTP layer leaves orphaned goroutines if you don't propagate the request's context.
- **Follow-Up:** "What happens if you don't call the cancel function returned by `WithCancel`?" → The child context and its resources are never released until the parent is cancelled — a goroutine/memory leak. Asked because forgetting cancel is a very common production bug.
- **Conclusion:** `context.Context` is Go's solution to the distributed systems problem of "how do I tell everything downstream to stop" — it is not optional in production services.

---

### Category: HTTP and API Patterns

> 📁 1 topic · 1 question

#### Topic: HTTP Middleware in Go — (Q count: 1)

- **[L3.HTTPAndAPIPatterns.HTTPMiddlewareInGo.Q1]**
  **Question:** How do you build composable HTTP middleware in Go's `net/http`, and what are the production patterns?
- **Answer:**
  - Go middleware is a function that wraps an `http.Handler`: `func Middleware(next http.Handler) http.Handler` — the wrapper calls `next.ServeHTTP(w, r)` after its own logic.
  - Middleware chains execute in nesting order: outermost middleware runs first on the way in and last on the way out — this determines where logging, auth, and panic recovery sit.
  - Panic recovery middleware must be outermost: a panic in a handler without recovery crashes the goroutine serving that request but not the server — `recover()` in the handler's own goroutine is required.
  - Request ID injection middleware generates a trace ID and adds it to `context.WithValue(r.Context(), ...)` — all downstream functions receive it without parameter threading.
  - Go 1.22's enhanced `net/http` routing (method + path patterns) reduces the need for third-party routers like `gorilla/mux` for standard services.
- **Example:**

```go
func RequestID(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        id := uuid.New().String()
        ctx := context.WithValue(r.Context(), requestIDKey{}, id)
        w.Header().Set("X-Request-ID", id)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
// Chain: Recovery(Logger(RequestID(handler)))
```

- **Technical Terms to Include:** `http.Handler`, `http.HandlerFunc`, middleware chain, panic recovery, request context, `net/http` routing (Go 1.22)
- **Gotcha:** "Middleware that modifies `http.ResponseWriter` after `next.ServeHTTP` has been called has no effect" — headers must be set before the first write; once the status code is written, it is final.
- **Follow-Up:** "How do you write response status codes and body in middleware for logging?" → Wrap `http.ResponseWriter` in a custom struct that captures the status code before delegating — `ResponseRecorder` pattern. Asked to test real middleware implementation depth.
- **Conclusion:** Go's middleware pattern is a function composition model — its simplicity is its strength, but execution order determines correctness.

---

### Category: Testing

> 📁 1 topic · 1 question

#### Topic: Table-Driven Tests — (Q count: 1)

- **[L3.Testing.TableDrivenTests.Q1]**
  **Question:** Why does Go favour table-driven tests, and how do you structure them for a function with complex error cases?
- **Answer:**
  - Table-driven tests define test cases as a slice of structs (`[]struct{ name, input, want, wantErr }`), loop over them, and call `t.Run(tc.name, ...)` — each case runs as a named sub-test with isolated failure reporting.
  - `t.Run` enables parallel sub-tests via `t.Parallel()` and allows running a single case with `go test -run TestFunc/case_name` — critical for debugging specific failures in CI.
  - Test naming (`tc.name`) is the most important field: `"nil input panics"` is actionable; `"test3"` is not — naming discipline determines debugging speed.
  - `t.Helper()` in assertion helpers marks the helper frame invisible in stack traces — failure lines point to the test case, not the assertion function.
  - Error cases use `errors.Is(err, tc.wantErr)` — not string comparison — to remain stable across error message refactors.
- **Example:**

```go
func TestDivide(t *testing.T) {
    cases := []struct {
        name    string
        a, b    int
        want    int
        wantErr error
    }{
        {"normal division", 10, 2, 5, nil},
        {"divide by zero", 10, 0, 0, ErrDivByZero},
    }
    for _, tc := range cases {
        tc := tc // pre-Go 1.22 loop variable capture fix
        t.Run(tc.name, func(t *testing.T) {
            t.Parallel()
            got, err := Divide(tc.a, tc.b)
            if !errors.Is(err, tc.wantErr) {
                t.Errorf("err=%v, want=%v", err, tc.wantErr)
            }
            if err == nil && got != tc.want {
                t.Errorf("got=%d, want=%d", got, tc.want)
            }
        })
    }
}
```

- **Technical Terms to Include:** table-driven test, `t.Run`, sub-test, `t.Parallel`, `t.Helper`, `errors.Is`, loop variable capture (Go 1.22 fix)
- **Gotcha:** "What does `tc := tc` inside the loop do?" → Pre-Go 1.22, loop variable capture would cause all goroutines to see the last value. Go 1.22 fixed per-iteration semantics — but expect the question from interviewers testing legacy code awareness.
- **Follow-Up:** "How do you test a function with external HTTP dependencies?" → `httptest.NewServer` or `httptest.NewRecorder` — no mocking framework needed. Asked to check stdlib fluency.
- **Conclusion:** Table-driven tests are Go's idiom for exhaustive, readable, maintainable test coverage — the loop is the pattern, the struct is the contract.

---

## L4 — Advanced Patterns, Optimization, Edge Cases

> 📊 **L4:** 3 categories · 4 topics · 4 questions

---

### Category: Memory and Performance

> 📁 2 topics · 2 questions

#### Topic: Escape Analysis — (Q count: 1)

- **[L4.MemoryAndPerformance.EscapeAnalysis.Q1]**
  **Question:** What is escape analysis in Go, and how does it determine whether a variable lives on the stack or heap?
- **Answer:**
  - Escape analysis is a compile-time pass where the Go compiler determines whether a variable's lifetime can be bounded to the current function's stack frame, or must "escape" to the heap.
  - A variable escapes to the heap when: it is returned by pointer from a function, assigned to an interface, captured by a closure that outlives the function, or sent over a channel.
  - Stack allocation is fast (just move the stack pointer) and GC-free; heap allocation requires the GC to eventually collect it — every escape is a potential GC pressure point.
  - Inspect escape decisions with `go build -gcflags="-m"` — output shows `variable escapes to heap` per variable, enabling targeted optimisation.
  - The common pattern of returning `*MyStruct` from a constructor always triggers escape — sometimes the correct trade-off, but worth knowing the cost.
- **Example:**

```go
// Escapes: returned pointer must outlive the function
func newConfig() *Config { return &Config{} } // heap

// Does NOT escape: used only within this function
func sum(nums []int) int {
    total := 0 // stack
    for _, n := range nums { total += n }
    return total
}
```

- **Technical Terms to Include:** escape analysis, stack allocation, heap allocation, `go build -gcflags="-m"`, GC pressure, closure capture, interface boxing
- **Gotcha:** "Passing a value to an interface causes escape" — assigning a concrete type to `interface{}` always causes the value to escape to the heap, even for small integers. This is why `fmt.Println(42)` allocates.
- **Follow-Up:** "How do you avoid allocations in a hot path?" → Use value receivers, avoid interfaces in the hot path, pre-allocate slices, use `sync.Pool` for temporary objects. Asked because allocation reduction is the primary Go performance lever.
- **Conclusion:** Escape analysis is the invisible boundary between stack and heap — understanding it is the difference between writing idiomatic Go and writing fast Go.

---

#### Topic: sync.Pool — (Q count: 1)

- **[L4.MemoryAndPerformance.SyncPool.Q1]**
  **Question:** What is `sync.Pool` and when is it the right tool for reducing allocations in high-throughput Go services?
- **Answer:**
  - `sync.Pool` is a cache of temporary objects that can be reused across goroutines — it reduces GC pressure by reusing allocations that would otherwise be created and immediately discarded.
  - The canonical use case is request-scoped buffers: `bytes.Buffer`, JSON encoders, or serialisation scratch space that is needed per-request but has identical lifetime.
  - `Pool.Get()` returns a pooled object or calls `New` if the pool is empty; `Pool.Put()` returns it for reuse — callers must reset the object before returning it to avoid data leakage between requests.
  - Objects in the pool can be collected by the GC between GC cycles — `sync.Pool` is not a cache with guaranteed retention, it is a best-effort reuse mechanism.
  - Wrong use: storing objects with non-trivial state (DB connections, goroutines) — use dedicated connection pools (`database/sql`, gRPC client pools) for those.
- **Example:**

```go
var bufPool = sync.Pool{
    New: func() any { return new(bytes.Buffer) },
}

func handler(w http.ResponseWriter, r *http.Request) {
    buf := bufPool.Get().(*bytes.Buffer)
    buf.Reset() // critical: clear state before use
    defer bufPool.Put(buf)
    // use buf for response building
}
```

- **Technical Terms to Include:** `sync.Pool`, GC pressure, object reuse, `Pool.Get`, `Pool.Put`, scratch buffer, GC collection cycle
- **Gotcha:** "Not calling `Reset()` before using a pooled buffer" — the buffer retains data from the previous request, creating an information leakage bug that is invisible in unit tests but dangerous in production.
- **Follow-Up:** "How would you measure whether `sync.Pool` is actually helping?" → `go test -bench . -benchmem` before and after — compare `allocs/op`. Also `pprof` heap profile to confirm reduction in allocations.
- **Conclusion:** `sync.Pool` is a targeted allocation reducer, not a general cache — its contract is "reuse if available," not "always available."

---

### Category: Advanced Concurrency

> 📁 1 topic · 1 question

#### Topic: sync.Mutex vs sync.RWMutex — (Q count: 1)

- **[L4.AdvancedConcurrency.SyncMutexVsSyncRWMutex.Q1]**
  **Question:** When do you choose `sync.RWMutex` over `sync.Mutex`, and what is the production risk of choosing wrong?
- **Answer:**
  - `sync.Mutex` provides exclusive access: one goroutine holds the lock at a time — reads and writes are serialised equally.
  - `sync.RWMutex` allows multiple concurrent readers (`RLock`/`RUnlock`) but exclusive write access (`Lock`/`Unlock`) — correct for read-heavy, write-sparse shared state.
  - Writer starvation is a real risk with `RWMutex`: if readers continuously hold `RLock`, a waiting writer may be starved indefinitely — Go's implementation gives writers priority once a writer is waiting.
  - `RWMutex` has higher overhead than `Mutex` for write-heavy workloads — the bookkeeping for reader count tracking costs more than a simple Mutex acquisition.
  - Rule of thumb: if reads outnumber writes 10:1+ and the critical section is non-trivial in cost, `RWMutex` is appropriate; otherwise use `Mutex`.
- **Example:** A Go service maintaining an in-memory route table (read on every request, updated on config reload every 60 seconds) uses `sync.RWMutex` — thousands of concurrent reads per second, one write per minute.
- **Technical Terms to Include:** `sync.Mutex`, `sync.RWMutex`, `RLock`, `RUnlock`, writer starvation, critical section, read-heavy workload
- **Gotcha:** "Copying a `sync.Mutex` after first use causes undefined behaviour" — `go vet` catches this, but it is a source of intermittent lock failures in code that passes mutexes by value instead of pointer.
- **Follow-Up:** "What is `sync.Once` and when does it replace a mutex?" → Executes a function exactly once across all goroutines — used for lazy singleton initialisation. Avoids the double-checked locking pattern. Asked because it is an idiom experienced Go engineers know cold.
- **Conclusion:** `RWMutex` is an optimisation, not a default — reach for it only when read-heavy contention is a measured bottleneck.

---

### Category: Generics

> 📁 1 topic · 1 question

#### Topic: Go Generics (Type Parameters) — (Q count: 1)

- **[L4.Generics.GoGenericsTypeParameters.Q1]**
  **Question:** What problem do generics solve in Go, and what are the constraints (pun intended) on their use?
- **Answer:**
  - Before generics (pre-Go 1.18), writing type-agnostic code required `interface{}` and runtime type assertions — losing compile-time safety and introducing boxing allocations.
  - Generics introduce type parameters: `func Map[T, U any](s []T, f func(T) U) []U` — the compiler generates type-specific code, preserving type safety without `interface{}` overhead.
  - Constraints (`comparable`, `any`, custom interfaces) restrict what types can be substituted — `comparable` allows `==` and `!=`; custom constraints define required methods.
  - Go generics are not Java generics: there is no type erasure — the compiler generates distinct code (or uses GC shape stenciling) for each concrete type.
  - Limitations: no parameterised methods (only parameterised types and functions), no operator overloading — Go generics are deliberately conservative.
- **Example:**

```go
type Number interface { ~int | ~int64 | ~float64 }

func Sum[T Number](values []T) T {
    var total T
    for _, v := range values { total += v }
    return total
}
// Caller: Sum([]int{1,2,3}) or Sum([]float64{1.1, 2.2})
```

- **Technical Terms to Include:** type parameter, type constraint, `comparable`, `any`, GC shape stenciling, type erasure (contrast with Java), union constraint (`~int | ~float64`)
- **Gotcha:** "Can I define a generic method on a struct?" → No — only functions and types can have type parameters; methods cannot introduce new type parameters. This is a hard language limit.
- **Follow-Up:** "When should you not use generics?" → When a simple interface satisfies the need — generics add cognitive overhead and slower compilation. Asked to distinguish "can use" from "should use."
- **Conclusion:** Go generics eliminate `interface{}` casts in collection-style utilities — they are a precision tool, not a general-purpose abstraction mechanism.

---

## L5 — Architecture and System Design

> 📊 **L5:** 2 categories · 3 topics · 3 questions

---

### Category: Microservice Architecture

> 📁 2 topics · 2 questions

#### Topic: Designing Go Microservices — (Q count: 1)

- **[L5.MicroserviceArchitecture.DesigningGoMicroservices.Q1]**
  **Question:** You are architecting a new Go microservice that will serve high-frequency API requests at Persistent's enterprise scale. Walk me through your architectural decisions from process startup to request handling to graceful shutdown.
- **Answer:**
  - **Startup:** Use `sync.Once` or constructor functions to initialise dependencies (DB pool, config, clients) before registering HTTP handlers — fail fast if any dependency is unreachable; don't serve traffic with a broken dependency.
  - **Configuration:** Prefer environment variables (12-factor) over config files for containerised deployments; use a typed config struct with validation at startup — not scattered `os.Getenv` calls throughout the code.
  - **Request handling:** Every handler receives a `context.Context` from the incoming request; propagate it to all downstream calls — DB queries, HTTP clients, gRPC calls; never create `context.Background()` inside a handler.
  - **Observability:** Structured logging (`log/slog` since Go 1.21) with trace IDs; Prometheus metrics via `/metrics`; distributed tracing with OpenTelemetry — all three must be present in a production service.
  - **Graceful shutdown:** `os.Signal` channel listening for `SIGTERM`/`SIGINT`; call `http.Server.Shutdown(ctx)` with a deadline — in-flight requests complete, new requests are rejected, connections close cleanly. Kubernetes sends SIGTERM on pod eviction.
- **Example:**

```go
srv := &http.Server{Addr: ":8080", Handler: router}
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
go func() { srv.ListenAndServe() }()
<-quit
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
srv.Shutdown(ctx)
```

- **Technical Terms to Include:** 12-factor app, `sync.Once`, graceful shutdown, `SIGTERM`, `http.Server.Shutdown`, structured logging (`log/slog`), Prometheus, OpenTelemetry, context propagation
- **Gotcha:** "Not draining in-flight requests on shutdown" — Kubernetes sends SIGTERM and waits `terminationGracePeriodSeconds` (default 30s) before SIGKILL; a service that exits immediately on SIGTERM drops active requests, causing 5xx errors during deployments.
- **Follow-Up:** "How would you implement a health check endpoint?" → `/healthz` (liveness) and `/readyz` (readiness) — readiness should check downstream dependency health; liveness should only fail if the process is deadlocked. Asked because Kubernetes probe misconfiguration causes cascading failures.
- **Conclusion:** A production Go service is defined by its startup contract, context discipline, and shutdown hygiene — the business logic is the easy part.

---

#### Topic: gRPC Service Design in Go — (Q count: 1)

- **[L5.MicroserviceArchitecture.GRPCServiceDesignInGo.Q1]**
  **Question:** When would you choose gRPC over REST for internal Go microservice communication, and what does the Go implementation look like architecturally?
- **Answer:**
  - gRPC uses Protocol Buffers for schema-first, strongly typed contracts — the `.proto` file is the source of truth, generating Go client and server stubs via `protoc-gen-go` and `protoc-gen-go-grpc`.
  - Binary serialisation (protobuf) is 3–10x smaller and faster than JSON — for high-frequency internal service calls (millions/day), this is a meaningful latency reduction.
  - HTTP/2 multiplexing enables bidirectional streaming and concurrent RPC calls on a single TCP connection — REST over HTTP/1.1 requires multiple connections for parallelism.
  - gRPC adds complexity: `.proto` schema management, code generation in CI, a gRPC gateway if REST compatibility is needed, and additional observability tooling (gRPC interceptors vs HTTP middleware).
  - Use REST for: public APIs, browser clients, simple request-response with low frequency. Use gRPC for: internal service mesh, streaming data, performance-critical paths, polyglot environments.
- **Example:**

```protobuf
service OfferService {
  rpc GetOffer (OfferRequest) returns (OfferResponse);
  rpc StreamOffers (OfferFilter) returns (stream OfferResponse);
}
```

```go
// Server interceptor for tracing — equivalent to HTTP middleware
srv := grpc.NewServer(
    grpc.UnaryInterceptor(otelgrpc.UnaryServerInterceptor()),
)
pb.RegisterOfferServiceServer(srv, &offerServer{})
```

- **Technical Terms to Include:** Protocol Buffers, protoc, schema-first, HTTP/2 multiplexing, binary serialisation, interceptor, gRPC gateway, bidirectional streaming
- **Gotcha:** "gRPC error codes are not HTTP status codes" — `codes.NotFound` is not `404` without a gRPC-gateway translation layer. Mixing gRPC and REST clients without proper translation creates opaque errors.
- **Follow-Up:** "How do you handle versioning in a gRPC contract?" → Reserved field numbers in `.proto`, new optional fields — never delete or reuse field numbers. Breaking changes require a new service version. Asked because schema evolution discipline is a prod requirement.
- **Conclusion:** gRPC is the right choice for internal Go services at scale — schema-first contracts, binary efficiency, and streaming are production-grade requirements that REST approximates poorly.

---

### Category: Dependency Management

> 📁 1 topic · 1 question

#### Topic: Go Modules at Scale — (Q count: 1)

- **[L5.DependencyManagement.GoModulesAtScale.Q1]**
  **Question:** How do Go modules work, and what are the operational challenges in a large multi-service Go repository?
- **Answer:**
  - `go.mod` defines the module path and minimum version requirements; `go.sum` records cryptographic checksums of all dependencies — together they guarantee reproducible builds.
  - Go uses Minimum Version Selection (MVS): when two dependencies require different versions of a third, Go selects the minimum version satisfying both — not the latest. This is deliberately conservative and deterministic.
  - In a monorepo with multiple Go modules, each module has its own `go.mod` — tools like `workspace mode` (`go.work`) allow local development across modules without pushing changes.
  - `replace` directives in `go.mod` allow substituting a module with a local path or fork — useful in enterprise environments where dependency mirroring is required.
  - Vendoring (`go mod vendor`) captures all dependencies in a `vendor/` directory — required in air-gapped enterprise environments and deterministic CI builds.
- **Example:** At Persistent's enterprise scale, a private module proxy (Athens, JFrog GoCenter) mirrors public modules — `GONOSUMCHECK` and `GONOSUMDB` environment variables configure the proxy chain, enabling reproducible builds without internet access.
- **Technical Terms to Include:** `go.mod`, `go.sum`, Minimum Version Selection (MVS), `go.work` workspace mode, `replace` directive, `go mod vendor`, module proxy, `GONOSUMCHECK`
- **Gotcha:** "The latest version of a dependency is not always selected" — MVS surprises engineers expecting npm's "latest compatible" behaviour. A dependency's security fix may not be adopted until `go get dependency@latest` is explicitly run.
- **Follow-Up:** "How do you audit your Go dependencies for known vulnerabilities?" → `govulncheck` (Go's official vulnerability scanner) scans `go.mod` against the Go vulnerability database. Asked because supply chain security is a 10-year engineer's responsibility.
- **Conclusion:** Go modules trade update aggressiveness for determinism — MVS ensures that adding a dependency never silently changes another, making large codebases auditable.

---

## L6 — Expert Level: Cross-Cutting Concerns, Performance at Scale, Security

> 📊 **L6:** 4 categories · 4 topics · 4 questions

---

### Category: Runtime Internals

> 📁 1 topic · 1 question

#### Topic: The GMP Scheduler — (Q count: 1)

- **[L6.RuntimeInternals.TheGMPScheduler.Q1]**
  **Question:** Explain the GMP scheduler model in Go's runtime and describe how it handles goroutine blocking without wasting OS threads.
- **Answer:**
  - GMP: **G** (goroutine — the unit of work), **M** (machine — an OS thread), **P** (processor — a logical CPU context holding a runqueue of goroutines). `GOMAXPROCS` controls the number of Ps.
  - Each P has a local runqueue; a global runqueue handles overflow. Ms pick goroutines from their P's local queue first (cache locality), then steal from other Ps (work-stealing), then check the global queue.
  - When a goroutine blocks on a **syscall** (disk I/O, cgo): the M detaches from its P (the P is free to run other goroutines on a new M from the thread pool) — no OS thread is wasted waiting.
  - When a goroutine blocks on a **channel or mutex**: it is parked (moved off the runqueue entirely) and the M finds another runnable goroutine — channel operations do not consume OS threads while waiting.
  - Preemption (Go 1.14+): the scheduler preempts goroutines at safe points (function calls, memory allocations) to prevent a long-running goroutine from monopolising its P — prior to this, a tight compute loop could starve other goroutines.
- **Example:** A CPU-bound goroutine running a cryptographic operation (`crypto/sha256`) was monopolising one P, causing latency spikes on the same process's HTTP handlers. Fix: `runtime.Gosched()` inserted in the loop to yield cooperatively, or splitting the work across multiple goroutines.
- **Technical Terms to Include:** GMP model, runqueue, work-stealing, syscall handoff, goroutine parking, preemption (Go 1.14), `GOMAXPROCS`, `runtime.Gosched`
- **Gotcha:** "Setting `GOMAXPROCS` higher than CPU core count doesn't always help" — beyond physical cores, context switching overhead between OS threads exceeds scheduling benefit; for CPU-bound Go services, `GOMAXPROCS = numCPUs` is optimal.
- **Follow-Up:** "How does the Go scheduler interact with cgo?" → CGO calls enter C code on a separate OS thread outside the GMP model — the M is locked to the goroutine for the duration. Excessive CGO calls can exhaust the OS thread pool (`runtime.LockOSThread`). Asked at expert level to test CGO awareness.
- **Conclusion:** The GMP scheduler is why Go can run millions of goroutines on dozens of OS threads — the secret is parking blocked goroutines and reusing Os threads aggressively.

---

### Category: Performance at Scale

> 📁 1 topic · 1 question

#### Topic: pprof and Production Profiling — (Q count: 1)

- **[L6.PerformanceAtScale.PprofAndProductionProfiling.Q1]**
  **Question:** Walk me through diagnosing a memory leak in a production Go service using `pprof`, including what you look for and how you minimise production impact.
- **Answer:**
  - Expose `net/http/pprof` on a separate internal port (never the public API port) — import `_ "net/http/pprof"` for side-effect registration; access via `GET /debug/pprof/heap`.
  - Heap profile (`/debug/pprof/heap?gc=1`) triggers a GC before sampling — distinguishes live objects from objects awaiting collection; the `inuse_objects` and `inuse_space` views show what is actually held.
  - Goroutine profile (`/debug/pprof/goroutine`) counts goroutines by stack trace — a leaked goroutine shows as a goroutine blocked indefinitely on a channel read with a count that grows over time.
  - `go tool pprof -http=:9090 http://service:6060/debug/pprof/heap` renders a flame graph — look for unexpected accumulations in middleware, connection pool, or global cache.
  - Differential profiling: two heap snapshots minutes apart; the diff reveals what is growing — `pprof.WriteHeapProfile` to file for offline analysis in incident response.
- **Example:** A Go API service at Persistent serving enterprise clients showed steady RSS growth over 24 hours. pprof heap profile revealed a `map[string]*Response` global cache with unbounded growth — requests were cached by URL but never evicted. Fix: `sync.Map` + TTL eviction or an LRU from `groupcache`.
- **Technical Terms to Include:** `net/http/pprof`, heap profile, goroutine profile, `inuse_objects`, `inuse_space`, flame graph, differential profiling, `pprof.WriteHeapProfile`, GC trigger
- **Gotcha:** "Profiling in production is free" — heap profiling has ~5% CPU overhead at default sampling rate; CPU profiling (`/debug/pprof/profile?seconds=30`) adds 10–30% overhead — always time-bound CPU profiles in production.
- **Follow-Up:** "What is the difference between a CPU profile and a trace?" → CPU profile samples goroutine stacks at fixed intervals (where time is spent). `runtime/trace` records every goroutine event (when goroutines block, run, GC pauses) — trace is for scheduler and latency problems; CPU profile is for throughput problems.
- **Conclusion:** `pprof` is Go's production debugger — every senior Go engineer must be able to interpret a heap profile from first principles, not just run the tool.

---

### Category: Security

> 📁 1 topic · 1 question

#### Topic: Go Security Concerns at Scale — (Q count: 1)

- **[L6.Security.GoSecurityConcernsAtScale.Q1]**
  **Question:** What are the critical security failure modes in a production Go service, and how does Go's design help or hinder you?
- **Answer:**
  - **Supply chain:** Go modules with cryptographic checksums (`go.sum`) prevent dependency tampering; `govulncheck` scans for known CVEs in imported modules — both must be in CI.
  - **Memory safety:** Go eliminates buffer overflows and use-after-free by design (bounds-checked slices, GC) — but CGO calls that pass Go pointers to C code bypass these guarantees entirely.
  - **Goroutine injection via reflection:** `reflect.MakeFunc` and `unsafe.Pointer` can bypass type safety — avoid `unsafe` except in performance-critical low-level code with heavy review gate.
  - **Secret management:** Never log request bodies or headers in plaintext — Go's structured logging (`log/slog`) with field redaction and AWS Secrets Manager for runtime secret injection are the patterns.
  - **TLS configuration:** `crypto/tls` defaults are safe but permissive — explicitly configure `MinVersion: tls.VersionTLS12`, disable weak cipher suites, and use `x509.CertPool` for mutual TLS in internal service mesh.
- **Example:** A Persistent enterprise client's Go service was passing `Authorization` headers through a `fmt.Sprintf` logger — rotating to `log/slog` with a custom `LogValue()` on the request type that redacted auth headers eliminated the credential leakage path.
- **Technical Terms to Include:** `go.sum`, `govulncheck`, CGO memory safety, `unsafe.Pointer`, `crypto/tls` `MinVersion`, mutual TLS, `log/slog` field redaction, AWS Secrets Manager
- **Gotcha:** "Go is memory-safe so you don't need security review" — Go eliminates memory corruption bugs but business logic vulnerabilities (SQL injection via string concatenation, SSRF, IDOR) are language-agnostic. Security review is still required.
- **Follow-Up:** "How do you safely pass a Go pointer to a C function via CGO?" → Only in the CGO call frame, not stored by C code; Go's CGO rules prohibit C from holding a Go pointer after the call returns — the runtime enforces this with a check in debug builds.
- **Conclusion:** Go's memory safety eliminates a class of vulnerabilities but business logic security is language-agnostic — secure Go services require both language discipline and application security review.

---

### Category: Hard Trade-offs

> 📁 1 topic · 1 question

#### Topic: When Not to Use Go — (Q count: 1)

- **[L6.HardTradeOffs.WhenNotToUseGo.Q1]**
  **Question:** At an architect level, describe the scenarios where Go is the wrong choice and what you would recommend instead.
- **Answer:**
  - **GC-sensitive, ultra-low-latency systems (sub-millisecond):** Go's GC introduces stop-the-world pauses (typically 0.1–1ms in modern Go, but non-deterministic) — for high-frequency trading, real-time control systems, or game engines, Rust or C++ is correct.
  - **Compute-heavy ML/data workloads:** Go has no mature tensor library ecosystem; Python (PyTorch, TensorFlow) or Rust (Candle) dominates — Go is appropriate only for inference serving infrastructure, not training pipelines.
  - **Scripting and glue code:** Go's compile step eliminates it from one-off scripts and shell automation — Python or Bash is the right tool.
  - **Complex GUI applications:** Go has no mature native GUI framework — Electron, Flutter (Dart), or Swift/SwiftUI are the correct choices.
  - **Very small team, domain with strong framework conventions:** A 2-person team building a CRUD API will ship faster with Rails or Django than with Go — framework convention over flexibility wins at small scale.
- **Example:** Recommending against rewriting a Persistent client's ML training pipeline in Go — the inference server (API, batching, observability) was rewritten in Go for 5x throughput improvement; the training workload stayed in Python.
- **Technical Terms to Include:** GC pause, stop-the-world, Rust borrow checker, sub-millisecond latency, inference serving vs training pipeline, framework convention
- **Gotcha:** "Go GC pauses are negligible now" — for 99.9th percentile latency SLOs below 10ms, even 1ms GC pauses appear in the tail. Know the GC pause characteristics of your target environment before committing.
- **Follow-Up:** "How would you reduce GC pause in a latency-sensitive Go service?" → Reduce heap size (fewer allocations via `sync.Pool`, escape analysis), tune `GOGC` env var (higher value = less frequent GC = larger pauses when they occur), or use arena allocators (experimental in Go 1.20+). Asked at L6 because this is real production trade-off territory.
- **Conclusion:** Recommending Go confidently requires knowing exactly where it fails — a Tech Lead who says "Go for everything" has not operated Go at scale.

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
