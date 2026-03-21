<!-- Generated: 2026-03-12 | Version: 1.26.1 -->

# Go (GoLang) Interview Preparation — Tech Lead / Architect Level

> Go 1.26.1 | Generated: 2026-03-12 | Career OS

---

## 🆕 What's New — Last 3 Versions

### [Go 1.26] — February 2026

- **Green Tea GC (default):** The rewritten garbage collector — experimental since 1.25 — is now the default. Delivers noticeably lower GC pause times, especially for latency-sensitive services. Interviewers will ask: "What changed in Go's GC in 1.26 and how does it affect your service design?"
- **`new(expression)` syntax:** `new` can now take an expression as its operand to specify the initial value — `p := new(42)` creates a `*int` pointing to `42`. Interviewers probe whether you understand `new` vs `make` deeply enough to notice this distinction.
- **Recursive generic type parameters:** Generic types may now reference themselves in their own type parameter list — enables cleaner implementation of recursive data structures (trees, graphs) without workarounds. Know the motivating example.
- **~30% faster cgo baseline overhead:** Each cgo call is cheaper — impacts services with heavy C interop. Know when cgo is and isn't worth the cost.
- **Stack allocation for slice backing stores:** Compiler detects more cases where a slice's backing array can live on the stack instead of escaping to the heap — reduces GC pressure. Interviewers may probe escape analysis awareness.
- **Experimental goroutine leak profiles:** New profiling endpoint for detecting goroutines that never terminate — critical for long-running services.
- **Deprecated / Removed:** Windows 32-bit ARM support removed. macOS 12 Monterey is last supported version.

### [Go 1.25] — August 2025

- **Container-aware GOMAXPROCS:** Runtime automatically tunes `GOMAXPROCS` based on CPU cgroup limits — Go services in containers now self-configure correctly without `uber-go/automaxprocs`. Interviewers ask: "Why was this needed, and what happened before?"
- **Green Tea GC (experimental):** `GOEXPERIMENT=greenteagc` — 10–40% reduction in GC overhead for GC-heavy workloads. Know the trade-off between throughput and latency it makes.
- **Flight Recorder tracing:** Rolling in-memory trace buffer — dump the last few seconds of activity on demand. Eliminates the "enable tracing before the bug happens" problem. High interviewer signal for observability questions.
- **DWARF 5 debug format:** Smaller binaries, faster linking. Know that this affects debugging toolchain compatibility.
- **`go vet` WaitGroup.Add placement check:** Catches the classic goroutine launch-before-Add bug at compile time.
- **Stricter nil pointer enforcement:** Previously-silently-passing nil pointer code now correctly panics. Know the `check errors immediately` discipline this enforces.
- **Deprecated / Removed:** `GOEXPERIMENT=aliastypeparams` flag removed (generic aliases now stable).

### [Go 1.24] — February 2025

- **Generic type aliases (stable):** A type alias can now have type parameters — `type Set[T comparable] = map[T]struct{}`. Enables reusable type-level abstractions without full type definitions. Interviewers probe generics depth here.
- **Tool directives in `go.mod`:** `tool` directive tracks CLI tool dependencies (linters, generators) in `go.mod` — eliminates the `tools.go` blank import workaround. Know the migration path.
- **`weak` package (WeakPointer):** Standard library `weak.Pointer[T]` — a typed weak reference that doesn't prevent GC. Direct equivalent of `sync.Pool` usecase but more explicit. Interviewers ask about GC interaction.
- **Swiss Table map implementation:** Go's `map` now uses Swiss Tables internally (same as absl::flat_hash_map) — significant lookup performance improvement for large maps. Know that this is transparent to code but measurable in benchmarks.
- **`math/rand/v2` stable:** New RNG API with better defaults — `rand.N(n)` instead of `rand.Intn(n)`. Know that the old `math/rand` package is soft-deprecated.
- **Deprecated / Removed:** `testing.B.Loop()` introduced for benchmark loops — replaces `b.N` pattern for more accurate benchmarking.

> **Interview signal:** The version a candidate references unprompted reveals whether they are current or coasting on old knowledge. Interviewers at Tech Lead level will probe the latest version explicitly.

---

## L1 — Conceptual Foundations

---

**L1 — Topics identified:**

- What is Go / Design Philosophy
- Compiled, Statically Typed, Garbage Collected
- Go vs Other Languages (JS/Python/Java)
- Packages & Modules
- The Go Toolchain (`go build`, `go mod`, `go vet`, `go test`)
- Zero Values
- Go's Type System (structural typing vs nominal)

---

### Fundamentals

#### What is Go / Design Philosophy

- **Question:** What is Go, why was it designed, and what explicit trade-offs does it make that differentiate it from every other mainstream language?
- **Answer:**
  - Go was designed at Google in 2007 by Robert Griesemer, Rob Pike, and Ken Thompson — primarily to solve Google's software engineering problems at scale: fast compilation, efficient concurrency, readable codebases maintained by large teams.
  - Core design philosophy: **simplicity over expressiveness**. Go deliberately excludes features common in other languages — no inheritance, no function overloading, no operator overloading, no exceptions, no implicit conversions, no generics until 1.18. Every exclusion is a deliberate trade-off to keep the language small, readable, and toolable.
  - **Key trade-offs Go makes:**
    (1) Speed of compilation over runtime micro-optimisation — Go compiles to machine code fast enough to use as a scripting replacement.
    (2) Explicit error handling over exceptions — errors are values, not control flow.
    (3) Composition over inheritance — structs + interfaces, no class hierarchy.
    (4) Goroutines over OS threads — cheap concurrency without thread management overhead.
  - Go produces a single statically linked binary — no runtime dependency, no VM, no interpreter. Deploy anywhere the target OS/arch can run a binary.
- **Example:**

```go
// The entire philosophy in one function:
// explicit types, explicit errors, no exceptions, no magic
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero") // error as value, not throw
    }
    return a / b, nil
}

result, err := divide(10, 0)
if err != nil {
    log.Fatal(err) // handle immediately, not in a catch block somewhere else
}
```

- **Technical Terms to Include:** statically typed, compiled, garbage collected, CSP concurrency, goroutine, error as value, single binary, structural typing, composition over inheritance
- **Gotcha:** "Go is simple" is often mistaken for "Go is easy." The language syntax is small, but concurrent Go — managing goroutine lifetimes, channel directionality, context cancellation, and race conditions — has substantial depth that takes time to master.
- **Follow-Up:** "Why does Go not have exceptions?" → Exceptions create invisible control flow — a function call can exit via any of its callees without the caller knowing. Go's error-as-value forces every error to be an explicit decision at every call site. The cost is verbosity; the benefit is that error handling is visible in the code, not hidden in a catch hierarchy. → They're testing understanding of the intentional design choice.
- **Conclusion:** Go's power comes from what it chose not to include — every missing feature is a deliberate trade-off toward simplicity, readability, and predictable performance at the cost of expressiveness that other languages offer.

---

#### Compiled, Statically Typed, Garbage Collected

- **Question:** How does Go's compilation model work, what does static typing enforce, and how does Go's garbage collector differ from JVM/JS GC?
- **Answer:**
  - **Compilation:** Go compiles directly to native machine code — no VM, no bytecode. `go build` produces a statically linked binary that includes the Go runtime, standard library, and all dependencies. Compilation is extremely fast — the language was designed for sub-second compile times even at large scale.
  - **Static typing:** All types are resolved at compile time. The compiler rejects type mismatches, unused imports, and unused variables (hard errors, not warnings). This strictness is intentional — unused variables and imports are the most common sources of dead code.
  - **Type inference:** `:=` short variable declaration infers the type — `x := 42` infers `int`. But the type is still fixed at compile time — this is not dynamic typing.
  - **Garbage collector (Go 1.26):** Concurrent, tri-color mark-and-sweep. Runs concurrently with the application (not stop-the-world for most work). The new Green Tea GC (default in 1.26) reduces pause times further by restructuring the marking phase. Go's GC is tuned for low latency — not maximum throughput — making it suitable for server workloads.
- **Example:**

```go
// Static type enforcement — compile error examples
var x int = "hello" // compile error: cannot use "hello" as int
import "fmt"        // compile error if fmt is not used anywhere

// Type inference — still statically typed
x := 42        // x is int — inferred, but fixed
y := 3.14      // y is float64
z := "hello"   // z is string

// Zero value — every type has one
var i int      // 0
var s string   // ""
var b bool     // false
var p *int     // nil
```

- **Technical Terms to Include:** static typing, type inference, native compilation, single binary, concurrent GC, tri-color mark-and-sweep, Green Tea GC, GOGC, GOMEMLIMIT, stop-the-world, escape analysis
- **Gotcha:** Go's unused variable error is a compile-time hard error — not a warning. `_ = variableName` is the idiomatic way to explicitly discard a value. Teams switching from dynamic languages frequently hit this, especially in debugging sessions where they comment out code that used a variable.
- **Follow-Up:** "How do you tune Go's garbage collector for a latency-sensitive service?" → `GOGC` controls the GC trigger ratio (default 100 — GC when heap doubles). Lower `GOGC` = more frequent GC = lower memory, higher CPU cost. `GOMEMLIMIT` (Go 1.19+) caps total memory — GC runs aggressively before hitting the limit. For latency: raise `GOGC` to reduce GC frequency, set `GOMEMLIMIT` to prevent OOM. Profile with `go tool pprof` to identify allocation hot spots. → They're testing production GC tuning depth.
- **Conclusion:** Go's compilation model (fast, native, single binary) and its latency-tuned GC make it the language of choice for cloud-native backends — but exploiting this correctly requires understanding escape analysis, GC tuning, and allocation patterns, not just the language syntax.

---

#### Zero Values

- **Question:** What is Go's zero value concept, why is it architecturally significant, and how does it change how you design types?
- **Answer:**
  - In Go, every variable declared without an explicit initialiser is automatically set to its **zero value** — `0` for numeric types, `false` for bool, `""` for string, `nil` for pointers/slices/maps/channels/functions/interfaces.
  - Zero values are not a convenience feature — they are a design principle. **Types should be usable at their zero value** without requiring explicit initialisation. `var mu sync.Mutex` is immediately usable — no `NewMutex()` call required. `var buf bytes.Buffer` is ready for writes immediately.
  - This principle guides API design: a struct's zero value should be a sensible default. If your type requires `New()` to be called before use, that's a signal that either the type has invariants that zero value violates, or the design could be simplified.
  - Composite type zero values: a zero-value slice is `nil` — `len(nil) == 0`, `append(nil, x)` works. A zero-value map is `nil` — reads return zero values but writes panic. A zero-value struct has all fields at their zero values.
- **Example:**

```go
// sync.Mutex is usable at zero value — no constructor needed
type SafeCounter struct {
    mu    sync.Mutex // zero value is an unlocked mutex — ready to use
    count int        // zero value is 0
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

// Zero value of a struct is valid
var sc SafeCounter // no New() call needed
sc.Increment()

// Common trap — zero value map panics on write
var m map[string]int    // m is nil
m["key"] = 1            // PANIC: assignment to nil map

// Fix — initialise before writing
m := make(map[string]int)
m["key"] = 1            // safe
```

- **Technical Terms to Include:** zero value, nil, usable at zero value, make vs var, nil slice, nil map, nil pointer dereference, struct zero value, sync.Mutex design
- **Gotcha:** `nil` slice and `nil` map behave differently. A nil slice is safe to `range`, `len()`, and `append()` to — it behaves like an empty slice. A nil map is safe to read from (returns zero value) but **panics** on write. Always `make` a map before writing to it.
- **Follow-Up:** "What is the difference between `var s []int` and `s := []int{}`?" → Both have `len == 0` and `cap == 0`. The difference: `var s []int` is nil (`s == nil` is true). `s := []int{}` is non-nil (`s == nil` is false). For JSON serialisation: nil slice encodes as `null`, empty slice encodes as `[]`. This distinction matters when designing API response types. → They're testing nil vs empty semantic precision.
- **Conclusion:** Zero values are Go's contract that a type should be ready to use at declaration — designing types that satisfy this principle reduces API surface, eliminates constructor ceremony, and prevents the class of "forgot to initialise" bugs common in other languages.

---

#### Packages & Modules

- **Question:** How do Go packages and modules work, what is the difference between them, and how does Go's visibility model differ from other languages?
- **Answer:**
  - **Package:** The unit of code organisation in Go. Every `.go` file declares its package with `package name`. A directory is a package. All files in a directory must share the same package name (except `_test` suffix files).
  - **Module:** The unit of versioning and distribution. Defined by `go.mod` in the root directory. A module contains one or more packages. Module path = the import path prefix for all packages in the module (e.g., `github.com/org/project`).
  - **Visibility:** Go's visibility is binary and name-based — names starting with a capital letter are **exported** (public), lowercase are **unexported** (package-private). No `public`, `private`, `protected` keywords. This applies to types, functions, methods, fields, and constants.
  - **`go.mod`:** Declares the module path, minimum Go version, and dependencies. `go.sum` records cryptographic hashes of each dependency version — tamper detection. `go mod tidy` synchronises the two files with actual imports.
  - **Go 1.24 tool directives:** `go.mod` now supports `tool` directive for CLI dependencies — replaces the `tools.go` hack.
- **Example:**

```go
// package declaration
package user // all files in this directory must be package user

// Exported — accessible from other packages
type User struct {
    ID    int    // exported field
    Email string // exported field
    password string // unexported — only accessible within package user
}

// Exported function
func NewUser(id int, email, password string) *User {
    return &User{ID: id, Email: email, password: hashPassword(password)}
}

// Unexported — internal implementation detail
func hashPassword(p string) string { /* ... */ }

// go.mod
// module github.com/myorg/myservice
// go 1.26
// require github.com/some/dep v1.2.3
// tool golang.org/x/tools/cmd/stringer // Go 1.24+
```

- **Technical Terms to Include:** package, module, go.mod, go.sum, exported, unexported, capital letter convention, import path, module path, go mod tidy, internal package, init function
- **Gotcha:** The `internal` package is a Go-enforced visibility boundary. Any package under an `internal` directory can only be imported by code in the parent tree of that `internal` directory. This is enforced by the compiler — not just convention — and is the correct mechanism for hiding implementation packages that shouldn't be part of the public API.
- **Follow-Up:** "What is the `init()` function and when should you use it?" → `init()` runs automatically after all package-level variables are initialised, before `main()`. A package can have multiple `init()` functions (even in the same file). Use sparingly — for registering drivers (database/sql), codec registrations, or global setup. Avoid using `init()` for logic that could be explicit — it makes code harder to test and reason about. → They're testing awareness of Go's implicit execution model.
- **Conclusion:** Go's package and module system is a deliberate simplification of dependency and visibility management — the capital-letter export rule and `internal` package enforcement make API boundaries explicit and compiler-enforced rather than relying on convention.

---

## L2 — Core Language

---

**L2 — Topics identified:**

- Variables, Types & Type Assertions
- Functions — Multiple Returns, Variadic, First-Class
- Pointers
- Structs & Methods
- Interfaces & Structural Typing
- Error Handling Patterns
- Slices — Internals & Gotchas
- Maps
- Goroutines (introduction)
- Defer, Panic, Recover

---

### Core Types

#### Interfaces & Structural Typing

- **Question:** How do Go interfaces work, what is structural (implicit) typing, and why is it architecturally more powerful than explicit interface declaration?
- **Answer:**
  - In Go, a type **implicitly satisfies** an interface if it implements all the interface's methods — no `implements` keyword, no explicit declaration. This is **structural typing** (also called "duck typing" at the type level).
  - The interface is defined by the consumer, not the producer. A library can define an `io.Reader` interface, and any type with a `Read([]byte) (int, error)` method satisfies it — even if that type was written before `io.Reader` existed. This is the inverse of Java's nominal typing where the producer must declare intent.
  - This enables **retroactive interface satisfaction** — you can write an interface for a third-party type without modifying the third-party package. Essential for testing (mock interfaces) and for adapting external dependencies.
  - **Interface internals:** An interface value is a two-word structure — a pointer to the type descriptor (dynamic type) and a pointer to the data. A nil interface has both words nil. A non-nil interface holding a nil pointer has a non-nil type descriptor — this is the notorious "nil interface != nil pointer" trap.
  - **Small interfaces:** Go idiom strongly favours small, single-method interfaces (`io.Reader`, `io.Writer`, `fmt.Stringer`). Large interfaces are hard to satisfy and hard to mock.
- **Example:**

```go
// Interface defined by the consumer — not the producer
type Storer interface {
    Store(key string, value []byte) error
    Retrieve(key string) ([]byte, error)
}

// RedisClient satisfies Storer implicitly — no 'implements Storer'
type RedisClient struct{ /* ... */ }
func (r *RedisClient) Store(key string, value []byte) error    { /* ... */ }
func (r *RedisClient) Retrieve(key string) ([]byte, error)     { /* ... */ }

// InMemoryStore also satisfies Storer — great for tests
type InMemoryStore struct{ data map[string][]byte }
func (m *InMemoryStore) Store(key string, value []byte) error  { m.data[key] = value; return nil }
func (m *InMemoryStore) Retrieve(key string) ([]byte, error)   { return m.data[key], nil }

// Function accepts the interface — works with any implementation
func processData(s Storer, key string, data []byte) error {
    return s.Store(key, data)
}

// The nil interface trap
var s Storer           // nil interface — both words nil
var r *RedisClient     // nil pointer to RedisClient
s = r                  // s is now non-nil interface holding nil pointer
s == nil               // FALSE — the interface has a non-nil type descriptor
```

- **Technical Terms to Include:** structural typing, implicit interface satisfaction, interface internals (type/data pair), retroactive satisfaction, nil interface, empty interface, type assertion, type switch, io.Reader, small interface principle
- **Gotcha:** The nil interface trap — a function returning an interface type that returns a typed nil (`return (*MyType)(nil), err`) returns a non-nil interface. The caller's `if err != nil` check passes, but calling methods on it panics. Always return untyped `nil` from interface-returning functions: `return nil, err`.
- **Follow-Up:** "What is the empty interface `any` / `interface{}` and when is it appropriate?" → `any` (alias for `interface{}` since Go 1.18) can hold any value. Appropriate for: generic containers before generics existed, JSON unmarshalling into unknown structure, `fmt.Println` (takes `...any`). Inappropriate for: typed APIs where you know the type — use generics instead. `any` loses type safety and requires runtime type assertions. → They're testing when to use vs avoid the escape hatch.
- **Conclusion:** Go's implicit interface satisfaction inverts the traditional dependency direction — the consumer defines the contract, producers satisfy it without knowing it exists, enabling decoupled architectures and effortless test doubles.

---

#### Pointers

- **Question:** How do pointers work in Go, when should you use pointer receivers vs value receivers, and what does Go's escape analysis do?
- **Answer:**
  - A pointer holds the memory address of a value. `&x` gets a pointer to `x`. `*p` dereferences the pointer to get the value. Go has no pointer arithmetic — you cannot do `p + 1` to advance to the next element (unlike C).
  - **Value receiver** (`func (t MyType) Method()`): Method receives a copy of the value. Mutations inside the method do not affect the original. Safe for concurrent use — no shared state.
  - **Pointer receiver** (`func (t *MyType) Method()`): Method receives a pointer to the original. Mutations affect the original.
    Required when:
    (1) The method must mutate the receiver.
    (2) The receiver is large (avoid copying).
    (3) Consistency — if any method needs a pointer receiver, use pointer receivers for all methods on that type.
  - **Escape analysis:** The Go compiler determines whether a variable's memory can live on the stack (local, fast, GC-free) or must be allocated on the heap (longer lifetime, GC-managed). A variable "escapes to the heap" when: its address is stored somewhere that outlives the function, it's returned as an interface, or it's too large for the stack. Use `go build -gcflags="-m"` to see escape analysis decisions.
- **Example:**

```go
// Value receiver — works on a copy
type Point struct{ X, Y float64 }
func (p Point) Distance() float64 {
    return math.Sqrt(p.X*p.X + p.Y*p.Y)
}

// Pointer receiver — mutates the original
func (p *Point) Scale(factor float64) {
    p.X *= factor
    p.Y *= factor
}

pt := Point{3, 4}
pt.Scale(2) // Go automatically takes &pt for pointer receiver call
fmt.Println(pt) // {6, 8} — original is modified

// Escape analysis — new() may not allocate on heap
func newPoint(x, y float64) *Point {
    p := &Point{x, y} // compiler may allocate on stack if p doesn't escape
    return p           // returning pointer — p escapes to heap
}

// Check escapes
// go build -gcflags="-m" ./...
// Output: ./main.go:5:7: &Point literal escapes to heap
```

- **Technical Terms to Include:** pointer, dereference, address-of operator, value receiver, pointer receiver, escape analysis, stack allocation, heap allocation, no pointer arithmetic, nil pointer dereference, GOGC
- **Gotcha:** A method with a value receiver can be called on a pointer — Go automatically dereferences. A method with a pointer receiver can be called on an addressable value — Go automatically takes the address. But if the value is not addressable (e.g., a map value, a function return), you cannot call a pointer-receiver method on it without assigning to a variable first.
- **Follow-Up:** "How do you pass a large struct to a function without copying it?" → Pass a pointer: `func process(cfg *Config)`. Or if it should not be mutated, still pass a pointer (more efficient than copying 100 fields). The convention: if a struct is large (more than a few fields) or has pointer receivers on its methods, pass by pointer. If it's small and you want a clear "this won't mutate" signal, pass by value. → They're testing practical pointer use judgment.
- **Conclusion:** Pointers in Go are a precise tool for mutation and performance — the choice between value and pointer receiver defines the mutability contract of a type's methods, and escape analysis determines whether that choice costs a heap allocation.

---

#### Structs & Methods

- **Question:** How does Go implement behaviour through structs and methods, and how does embedding replace inheritance?
- **Answer:**
  - Go has no classes. Behaviour is attached to types via **methods** — functions with a receiver argument. Any named type (struct, alias, custom type) can have methods.
  - **Embedding** promotes fields and methods of an embedded type into the embedding struct — providing composition that resembles inheritance without the coupling. An embedded type's methods are accessible as if they were the outer struct's methods.
  - Embedding is not inheritance — the embedded type has no knowledge of the outer type. There is no `super` call. Method promotion is syntactic convenience — under the hood, the method is called on the embedded field, not on the outer struct.
  - Embedding an interface in a struct is a common pattern for partial interface implementation or for constructing mock types.
  - **Struct tags:** Field annotations used by encoding packages (`json`, `xml`, `db`), ORMs, and validators. Not enforced by the language — read via reflection.
- **Example:**

```go
// Embedding — composition, not inheritance
type Logger struct {
    prefix string
}
func (l *Logger) Log(msg string) {
    fmt.Printf("[%s] %s\n", l.prefix, msg)
}

type Server struct {
    Logger          // embedded — Server now has a Log method
    host   string
    port   int
}

s := Server{Logger: Logger{"SERVER"}, host: "localhost", port: 8080}
s.Log("starting") // promoted method — calls s.Logger.Log("starting")
s.Logger.Log("also valid") // explicit — same result

// Struct tags
type User struct {
    ID    int    `json:"id" db:"user_id"`
    Email string `json:"email" validate:"required,email"`
    Pass  string `json:"-"` // omit from JSON
}

// Method on non-struct type
type Celsius float64
func (c Celsius) ToFahrenheit() float64 { return float64(c)*9/5 + 32 }
temp := Celsius(100)
temp.ToFahrenheit() // 212
```

- **Technical Terms to Include:** method, receiver, embedding, method promotion, composition over inheritance, struct tag, reflection, anonymous field, method set, named type
- **Gotcha:** Embedding a type and embedding a pointer to a type behave differently. Embedding `Logger` means the Logger is part of the struct's value (zero-initialized). Embedding `*Logger` means only the pointer is in the struct — the zero value of the pointer is `nil`, and calling promoted methods on a nil pointer panics. Always initialise embedded pointer fields before use.
- **Follow-Up:** "How do you resolve method name conflicts when embedding multiple types with the same method name?" → If two embedded types have a method with the same name, the method is "shadowed" at the outer level — neither is promoted. You must call it explicitly via the field name: `s.TypeA.Method()` or `s.TypeB.Method()`. If the outer struct defines the method itself, it always wins. → They're testing embedding conflict resolution awareness.
- **Conclusion:** Go's struct embedding replaces class hierarchies with a flat composition model — you get method promotion and code reuse without the tight coupling of inheritance, enabling more flexible and testable designs.

---

#### Error Handling

- **Question:** How does Go's error handling model work, what are the `errors.Is`, `errors.As`, and `%w` patterns, and what are the architectural implications?
- **Answer:**
  - In Go, errors are values — any type implementing the `error` interface (`Error() string`) is an error. Functions return errors as the last return value by convention. Callers check `if err != nil` — immediately, at every call site.
  - **Sentinel errors:** Pre-declared error values compared with `==` — e.g., `io.EOF`, `sql.ErrNoRows`. Simple but opaque — callers can only check identity, not extract context.
  - **Custom error types:** Structs implementing `error` — can carry structured context (HTTP status, file path, operation name). Extracted with `errors.As(err, &target)`.
  - **Error wrapping (`%w`):** `fmt.Errorf("context: %w", err)` wraps an error, preserving the original while adding context. `errors.Unwrap(err)` peels one layer. `errors.Is(err, target)` traverses the entire chain looking for a match. `errors.As(err, &target)` traverses the chain looking for a type match.
  - **Architectural implication:** Errors propagate up the call stack with added context at each layer — like a stack trace but in prose. Callers can make decisions based on the wrapped error type or sentinel value without knowing the full chain.
- **Example:**

```go
// Sentinel error
var ErrNotFound = errors.New("not found")

// Custom error type — carries structured context
type ValidationError struct {
    Field   string
    Message string
}
func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed on %s: %s", e.Field, e.Message)
}

// Wrapping with context
func findUser(id int) (*User, error) {
    user, err := db.Query(id)
    if err != nil {
        return nil, fmt.Errorf("findUser(%d): %w", id, err) // wrap with context
    }
    return user, nil
}

// Checking wrapped errors
err := findUser(42)
if errors.Is(err, ErrNotFound) {
    // handle not found — works even if wrapped N layers deep
}

var ve *ValidationError
if errors.As(err, &ve) {
    fmt.Println(ve.Field) // extract structured info from the chain
}
```

- **Technical Terms to Include:** error interface, sentinel error, custom error type, error wrapping, `%w`, errors.Is, errors.As, errors.Unwrap, error chain, panic vs error, fmt.Errorf
- **Gotcha:** `errors.Is` uses `==` comparison by default. For custom error types to work correctly with `errors.Is`, they must either be pointer-comparable sentinel values or implement an `Is(target error) bool` method. A common mistake is wrapping a sentinel with `fmt.Errorf("msg: %w", ErrNotFound)` and then comparing with `err == ErrNotFound` — which fails. Always use `errors.Is`.
- **Follow-Up:** "When should you use `panic` vs returning an error?" → Return an error for expected failure conditions — bad input, network failure, missing resource. Use `panic` for truly unexpected states that indicate a programming bug — index out of range, nil dereference in code that should never reach that state. A library should never panic for conditions a caller could reasonably encounter — that forces the caller to use `recover`, which is cumbersome. Panics in a server process without recovery will crash the process. → They're testing the panic/error boundary judgment.
- **Conclusion:** Go's error-as-value model forces explicit, visible error handling at every call site — the wrapping and unwrapping chain via `%w`, `errors.Is`, and `errors.As` enables structured error propagation without exceptions while preserving the ability to make context-aware decisions at any layer.

---

#### Slices — Internals & Gotchas

- **Question:** How is a slice implemented internally, what are the append gotchas, and how does slice sharing cause unexpected mutations?
- **Answer:**
  - A slice is a three-field struct: a **pointer** to the underlying array, a **length** (number of elements accessible), and a **capacity** (total allocated space in the array). Slices do not own their data — they are views into an array.
  - `append(s, elem)`: if `len < cap`, appends in place (no new allocation), returning a slice with the same backing array and incremented length. If `len == cap`, allocates a new (larger) backing array, copies the data, and returns a slice pointing to the new array. The original slice is unaffected.
  - **Sharing trap:** Two slices can point to the same backing array. Modifying elements through one slice modifies the shared array, affecting the other slice. This is not visible from the slice values themselves.
  - **Three-index slice:** `s[low:high:max]` limits the capacity of the resulting slice to `max - low`, preventing the new slice from accessing (and potentially overwriting) elements beyond `high` in the original array. This is the defensive copy pattern.
  - `copy(dst, src)` copies `min(len(dst), len(src))` elements — always makes an independent copy.
- **Example:**

```go
// Slice internals
s := make([]int, 3, 5) // len=3, cap=5, pointer to 5-element backing array
s[0], s[1], s[2] = 1, 2, 3

// Sharing trap
a := []int{1, 2, 3, 4, 5}
b := a[1:3] // b = [2, 3], shares backing array with a
b[0] = 99
fmt.Println(a) // [1 99 3 4 5] — a is modified through b!

// Append reallocation break
c := a[1:3:3] // capacity limited to 3, so len=2, cap=2
c = append(c, 100) // cap exceeded — new backing array allocated
c[0] = 999
fmt.Println(a) // [1 99 3 4 5] — a is NOT modified — c got its own array

// Safe copy
original := []int{1, 2, 3}
duplicate := make([]int, len(original))
copy(duplicate, original) // fully independent
duplicate[0] = 999
fmt.Println(original) // [1, 2, 3] — unchanged
```

- **Technical Terms to Include:** slice header (pointer/len/cap), backing array, append reallocation, slice sharing, three-index slice, copy, make, nil slice vs empty slice, slice of slices, growth factor
- **Gotcha:** `append` is not safe for concurrent use. Two goroutines appending to the same slice can cause a data race even if the slice has enough capacity — the length update is not atomic. Always protect shared slices with a mutex or use channels.
- **Follow-Up:** "What is the growth strategy of `append` when it reallocates?" → Go's runtime uses a growth factor that starts at 2x for small slices and gradually decreases to ~1.25x for large slices (the exact formula changed in Go 1.18 to be smoother). This amortises the cost of append to O(1) average. Pre-allocating with `make([]T, 0, expectedLen)` avoids repeated reallocations when the final size is known. → They're testing allocation performance awareness.
- **Conclusion:** Slices are the most powerful and most dangerous data structure in Go — their shared-backing-array semantics make them efficient but require defensive copying at function boundaries to prevent the kind of silent mutation bugs that are extremely difficult to debug under concurrent load.

---

#### Defer, Panic, Recover

- **Question:** How do `defer`, `panic`, and `recover` work, and what is the correct architectural use of each?
- **Answer:**
  - **`defer`:** Schedules a function call to run when the surrounding function returns — regardless of whether it returns normally or via panic. Deferred calls execute in LIFO order. Arguments to deferred functions are evaluated at the `defer` statement, not at execution time.
  - **`panic`:** Immediately stops the current function, starts unwinding the stack, running deferred functions at each level. If panic reaches the top of the goroutine stack without being recovered, the program crashes with a stack trace.
  - **`recover`:** Only valid inside a deferred function. Stops the panic and returns the panic value. After `recover`, the function that deferred it returns normally. `recover` called outside a deferred function (or when there is no panic) returns `nil`.
  - **Correct use:** `defer` for cleanup (close files, release locks, send final metrics). `panic` only for programming errors (unrecoverable state). `recover` only at process boundaries (HTTP handler, goroutine root) to convert panics into error responses and prevent process crashes.
- **Example:**

```go
// defer for cleanup — guaranteed to run
func readFile(path string) ([]byte, error) {
    f, err := os.Open(path)
    if err != nil { return nil, err }
    defer f.Close() // always closes, even on early return or panic

    return io.ReadAll(f)
}

// defer LIFO order
func order() {
    defer fmt.Println("first deferred") // runs third
    defer fmt.Println("second deferred") // runs second
    defer fmt.Println("third deferred")  // runs first
    fmt.Println("function body")
    // Output: function body, third deferred, second deferred, first deferred
}

// recover — at HTTP handler boundary
func safeHandler(w http.ResponseWriter, r *http.Request) {
    defer func() {
        if rec := recover(); rec != nil {
            log.Printf("recovered panic: %v\n%s", rec, debug.Stack())
            http.Error(w, "internal error", http.StatusInternalServerError)
        }
    }()
    riskyOperation() // may panic
}

// defer argument evaluated immediately
x := 10
defer fmt.Println(x) // prints 10, not the value of x when function returns
x = 20
```

- **Technical Terms to Include:** defer, LIFO, panic, recover, stack unwinding, deferred argument evaluation, goroutine stack, debug.Stack, process boundary recovery
- **Gotcha:** Deferred function arguments are evaluated when `defer` is called, not when the deferred function runs. `defer log.Println(result)` captures `result`'s current value, not its final value. To capture the final value, use a closure: `defer func() { log.Println(result) }()`.
- **Follow-Up:** "Can you `recover` from a panic in a different goroutine?" → No — `recover` only works for panics in the same goroutine. A panic in goroutine A cannot be caught by a `recover` in goroutine B. Each goroutine must have its own panic recovery at its root if you want to prevent crashes. This is why every `go func()` that could panic should have a deferred recover. → They're testing goroutine panic isolation awareness.
- **Conclusion:** `defer` is Go's resource cleanup primitive, `panic` is reserved for unrecoverable programming bugs, and `recover` is the process-boundary safety net — together they provide structured cleanup and controlled crash prevention without exception hierarchies.

---

## L3 — Concurrency

---

**L3 — Topics identified:**

- Goroutines — Lifecycle & Scheduler
- Channels — Buffered, Unbuffered, Directional
- Select Statement
- sync Package (Mutex / RWMutex / WaitGroup / Once / Pool)
- Context — Cancellation / Timeout / Value
- Goroutine Leaks
- Race Conditions & the Data Race Detector
- Channel Patterns (Fan-out / Fan-in / Pipeline / Done Channel)

---

### Concurrency Model

#### Goroutines — Lifecycle & Scheduler

- **Question:** What is a goroutine, how does Go's runtime scheduler work (GMP model), and what is the cost of a goroutine?
- **Answer:**
  - A goroutine is a lightweight, independently executing function managed by Go's runtime — not a kernel thread. Goroutines start with a small stack (~2KB, grows dynamically) and are multiplexed onto OS threads by the Go scheduler.
  - **GMP Model:** The Go scheduler operates on three entities: **G** (goroutine — the unit of work), **M** (machine — OS thread), and **P** (processor — scheduling context, holds a run queue of G). `GOMAXPROCS` determines how many Ps exist. Each P runs one M at a time, each M runs one G. When a G blocks on I/O or a system call, its M is released, and a new M is created or unparked for that P.
  - **Work stealing:** When a P's run queue is empty, it steals goroutines from other Ps' run queues — maximising CPU utilisation without developer intervention.
  - **Cost:** A goroutine costs ~2KB of stack (grows as needed, up to `GOMAXPROCS * maxStackSize`). Creating a goroutine is O(microseconds). You can realistically run hundreds of thousands to millions of goroutines — unlike OS threads (where thousands is already expensive).
  - **Go 1.26 experimental goroutine leak profiles:** New pprof profile type for detecting goroutines that never terminate.
- **Example:**

```go
// Goroutine launch — just 'go' keyword
go func() {
    doWork() // runs concurrently
}()

// Common goroutine patterns
func processItems(items []Item) {
    var wg sync.WaitGroup
    for _, item := range items {
        wg.Add(1)
        go func(i Item) { // pass item as arg — avoid closure capture bug
            defer wg.Done()
            process(i)
        }(item)
    }
    wg.Wait() // block until all goroutines finish
}

// Goroutine closure capture bug
for _, item := range items {
    go func() {
        process(item) // WRONG — all goroutines share the same 'item' variable
    }()
}
```

- **Technical Terms to Include:** goroutine, GMP model, GOMAXPROCS, work stealing, goroutine stack growth, cooperative preemption, runtime scheduler, goroutine leak, go keyword, WaitGroup
- **Gotcha:** The goroutine closure capture bug — a `for` loop variable is shared across all goroutines spawned in the loop. By the time goroutines run, the loop may have already advanced. Fix: pass the variable as a function argument (a copy is made at call time) or use `:=` re-declaration inside the loop. Go 1.22 changed `for` loop variable semantics — each iteration creates a new variable by default, fixing this class of bug at the language level.
- **Follow-Up:** "What changed in Go 1.22 regarding `for` loop variable capture?" → Go 1.22 made each `for` loop iteration create a new variable for the loop variable — each goroutine launched in the loop now captures its own copy. The pre-1.22 behaviour of sharing one variable across all iterations is gone. Code that relied on the old behaviour (rare, unusual) may need updating. → They're testing awareness of the most impactful recent language change for concurrent code.
- **Conclusion:** Goroutines are Go's primary concurrency unit — cheap enough to launch per-request in a server, powerful enough to express complex concurrent patterns, but requiring disciplined lifecycle management to avoid leaks that silently accumulate in long-running processes.

---

#### Channels

- **Question:** How do channels work, what are the semantics of buffered vs unbuffered channels, and what are the directional channel types?
- **Answer:**
  - A channel is a typed conduit for communication between goroutines. Sending and receiving are the only operations — channels enforce the "share memory by communicating" principle.
  - **Unbuffered channel** (`make(chan T)`): Send blocks until a receiver is ready. Receive blocks until a sender is ready. Guarantees synchronisation — sender and receiver rendezvous at the channel.
  - **Buffered channel** (`make(chan T, n)`): Send blocks only when the buffer is full. Receive blocks only when the buffer is empty. Decouples sender and receiver — up to `n` values can be in-flight without synchronisation.
  - **Closing a channel:** `close(ch)` signals to receivers that no more values will be sent. A closed channel can still be received from — buffered values drain first, then zero values are returned. Sending to a closed channel panics. Closing an already-closed channel panics.
  - **Directional types:** `chan<- T` (send-only), `<-chan T` (receive-only). Used in function signatures to express intent and prevent misuse at compile time.
- **Example:**

```go
// Unbuffered — synchronisation point
ch := make(chan int)
go func() { ch <- 42 }()   // blocks until receiver is ready
val := <-ch                 // receives 42

// Buffered — decoupled
ch := make(chan int, 3)
ch <- 1; ch <- 2; ch <- 3  // non-blocking — buffer has space
// ch <- 4                  // would block — buffer full

// Closing and ranging
go func() {
    for i := 0; i < 5; i++ { ch <- i }
    close(ch) // signal done
}()
for v := range ch { // range receives until channel closed
    fmt.Println(v)
}

// Directional — at function boundary
func producer(out chan<- int) { // can only send
    out <- 42
}
func consumer(in <-chan int) {  // can only receive
    fmt.Println(<-in)
}

// Check if channel closed during receive
v, ok := <-ch
if !ok { /* channel closed */ }
```

- **Technical Terms to Include:** channel, buffered channel, unbuffered channel, send, receive, close, range, directional channel, channel as semaphore, nil channel, panic on closed send
- **Gotcha:** Receiving from a **nil channel** blocks forever — it never panics, it just blocks. This is sometimes used intentionally (a `nil` case in a `select` disables that case), but accidentally nil channels cause goroutine leaks that are very hard to detect.
- **Follow-Up:** "What happens when you send to a nil channel?" → Sending to a nil channel blocks forever — same as receiving. Closing a nil channel panics. This makes nil channel a valid tool for disabling a `select` case at runtime (set the channel to nil, and the select will never select that case again). → They're testing nil channel semantics precision.
- **Conclusion:** Channels are Go's synchronisation and communication primitive — the choice between buffered and unbuffered determines whether the sender and receiver must rendezvous (synchronous) or can operate independently (asynchronous), a distinction that drives the entire concurrent pipeline design.

---

#### Select Statement

- **Question:** What does `select` do, how does it handle multiple ready channels, and what are the `default` and `nil` channel patterns?
- **Answer:**
  - `select` waits on multiple channel operations simultaneously. When one or more cases are ready, Go chooses one **uniformly at random** — not in source order, not in priority order. This prevents starvation of any single case.
  - If no case is ready and there is a `default` case, `select` executes `default` immediately (non-blocking). Without `default`, `select` blocks until at least one case is ready.
  - **Timeout pattern:** Combine `select` with `time.After` (or `ctx.Done()`) to implement timeouts on channel operations — avoid waiting forever.
  - **Done channel pattern:** A `done` channel (or `ctx.Done()`) signals goroutines to stop. Goroutines `select` on their work channel and the done channel — whichever fires first is handled.
  - **nil channel disabling:** Setting a channel variable to `nil` removes it from consideration in `select` — the nil channel case is never selected. Useful for dynamically enabling/disabling cases.
- **Example:**

```go
// Basic select — handles first ready case
select {
case msg := <-ch1:
    fmt.Println("ch1:", msg)
case msg := <-ch2:
    fmt.Println("ch2:", msg)
case <-time.After(1 * time.Second):
    fmt.Println("timeout") // fires if neither ch1 nor ch2 is ready in 1s
}

// Non-blocking check with default
select {
case msg := <-ch:
    process(msg)
default:
    // channel empty — don't block
}

// Done channel — cooperative cancellation
func worker(done <-chan struct{}, jobs <-chan Job) {
    for {
        select {
        case <-done:
            return // shutdown signal received
        case job := <-jobs:
            process(job)
        }
    }
}

// nil channel disabling — toggle a case off
var ch2 chan int // nil — never selected
if condition {
    ch2 = make(chan int) // enable the case
}
select {
case v := <-ch1: /* always eligible */
case v := <-ch2: /* only eligible if ch2 is non-nil */
}
```

- **Technical Terms to Include:** select, pseudorandom case selection, default case, non-blocking select, timeout pattern, done channel, nil channel case, fan-in, context cancellation
- **Gotcha:** `time.After(d)` creates a channel backed by a `time.Timer` that is **never garbage collected** until it fires — if used in a tight loop or frequently, it leaks timers. Use `time.NewTimer(d)` and call `timer.Stop()` in the non-timeout case, or use `context.WithTimeout` which cleans up properly.
- **Follow-Up:** "What is the fan-in pattern with select?" → Fan-in merges multiple input channels into one output channel. A goroutine `select`s from all input channels and forwards to a single output — consumers read from one channel instead of managing many. Commonly used to merge results from parallel workers. → They're testing concurrent pipeline pattern knowledge.
- **Conclusion:** `select` is Go's multiplexed wait primitive — random case selection prevents starvation, the `default` case enables non-blocking polls, and nil channel disabling enables dynamic case management, making it the control flow backbone of all non-trivial concurrent Go programs.

---

#### Context — Cancellation, Timeout, Value

- **Question:** What is `context.Context`, how does cancellation propagate through a call tree, and what are the rules for context usage?
- **Answer:**
  - `context.Context` is an interface for carrying deadlines, cancellation signals, and request-scoped values across API boundaries and goroutines. It is the standard mechanism for cooperative cancellation in Go.
  - **Propagation:** A parent context creates a child with `context.WithCancel`, `context.WithTimeout`, or `context.WithDeadline`. When the parent is cancelled, all children are cancelled — the cancellation tree mirrors the call tree. Cancellation flows top-down, never bottom-up.
  - **Usage rules:** (1) `ctx` is always the first parameter. (2) Never store context in a struct — pass it explicitly. (3) Never pass `nil` context — use `context.Background()` or `context.TODO()` as roots. (4) Context values are for request-scoped data (trace IDs, auth tokens) — not for optional function parameters.
  - **`ctx.Done()`:** A channel that is closed when the context is cancelled. Check this in long-running operations and goroutines — `select` on `ctx.Done()` and your work channel.
  - **`context.WithValue`:** Stores a value accessible to all downstream functions. Key must be an unexported type to avoid collisions. Avoid overusing — it's not a replacement for function parameters.
- **Example:**

```go
// HTTP handler — context from request
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context() // cancellation tied to client disconnect
    result, err := fetchData(ctx) // context propagated down the call tree
    if err != nil {
        if errors.Is(err, context.Canceled) {
            return // client disconnected — no need to respond
        }
        http.Error(w, err.Error(), 500)
    }
}

// Timeout — cancel if takes too long
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel() // always cancel — releases resources even if timeout not hit
result, err := db.QueryContext(ctx, "SELECT ...")

// Goroutine respecting context
func worker(ctx context.Context, jobs <-chan Job) {
    for {
        select {
        case <-ctx.Done():
            log.Println("worker stopping:", ctx.Err())
            return
        case job := <-jobs:
            if err := process(ctx, job); err != nil {
                log.Println("job error:", err)
            }
        }
    }
}

// Context value — request-scoped data
type traceKey struct{} // unexported key type prevents collisions
ctx = context.WithValue(ctx, traceKey{}, traceID)
traceID := ctx.Value(traceKey{}) // retrieve
```

- **Technical Terms to Include:** context.Context, context.Background, context.WithCancel, context.WithTimeout, context.WithDeadline, context.WithValue, ctx.Done(), ctx.Err(), cancellation tree, request-scoped, context leak
- **Gotcha:** Always call the cancel function returned by `WithCancel`/`WithTimeout` — even if the context was already cancelled. Not calling it leaks the goroutine managing the timer and the associated resources. The `defer cancel()` pattern immediately after creation is idiomatic precisely for this reason.
- **Follow-Up:** "Why should you never store a context in a struct?" → Context carries request-specific lifecycle information — it should not outlive the request. Storing it in a struct means the context's lifetime is tied to the struct's lifetime, not the request's. A struct with a stored context can be used across multiple requests with the wrong (stale or cancelled) context. Pass context explicitly to make the lifecycle relationship clear. → They're testing architectural context discipline.
- **Conclusion:** Context is Go's mechanism for threading cancellation and deadlines through an entire call tree — its propagation model means a single cancel at the root (HTTP request end, user cancellation, timeout) ripples through all downstream operations, enabling clean resource release without shared mutable state.

---

## L4 — Advanced Patterns

---

**L4 — Topics identified:**

- Generics (Type Parameters, Constraints, Type Sets)
- Reflection
- sync.Pool & Object Reuse
- Advanced Channel Patterns (Pipeline / Fan-out / Fan-in)
- Functional Options Pattern
- Table-Driven Testing
- Benchmarking

---

### Advanced Language Features

#### Generics

- **Question:** How do Go generics work, what are type constraints, and how do they differ from generics in Java/TypeScript?
- **Answer:**
  - Generics (Go 1.18+) allow writing functions and types parameterised by type. Type parameters are declared in square brackets: `func Map[T, U any](s []T, f func(T) U) []U`.
  - **Constraints** define what operations are allowed on a type parameter. `any` (alias for `interface{}`) permits no operations beyond assignment. `comparable` permits `==` and `!=`. Custom constraints are interfaces defining a type set — the set of types that satisfy the constraint.
  - **Type sets:** In Go's generics model, an interface used as a constraint defines a set of types. `~int` means "int or any type whose underlying type is int." `int | string` is a union — the constraint is satisfied by int or string. This is type-set algebra, not method-set-only.
  - **Difference from Java generics:** Java generics are erased at runtime — all type parameters become `Object` at the bytecode level. Go generics are implemented via **GC shapes** — types that share the same memory layout share compiled code (dictionary-based dispatch), but specialised code may be generated for performance-critical paths. No type erasure — type information is available at runtime.
  - **Go 1.26:** Generic types can now reference themselves in their type parameter list — enables cleaner recursive type definitions.
- **Example:**

```go
// Generic function
func Map[T, U any](slice []T, fn func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = fn(v)
    }
    return result
}
doubled := Map([]int{1, 2, 3}, func(n int) int { return n * 2 }) // [2, 4, 6]
strs := Map([]int{1, 2, 3}, strconv.Itoa) // ["1", "2", "3"]

// Constraint with type set
type Number interface {
    ~int | ~int64 | ~float64 // any type whose underlying type is one of these
}
func Sum[T Number](nums []T) T {
    var total T
    for _, n := range nums { total += n }
    return total
}

// Generic type
type Stack[T any] struct {
    items []T
}
func (s *Stack[T]) Push(v T) { s.items = append(s.items, v) }
func (s *Stack[T]) Pop() (T, bool) {
    var zero T
    if len(s.items) == 0 { return zero, false }
    n := len(s.items) - 1
    v := s.items[n]
    s.items = s.items[:n]
    return v, true
}

// Go 1.26 — recursive type parameter (self-referential)
type Tree[T interface{ comparable; Less(T) bool }] struct {
    Value       T
    Left, Right *Tree[T]
}
```

- **Technical Terms to Include:** type parameter, type constraint, type set, any, comparable, tilde (~), union constraint, GC shapes, dictionary dispatch, type inference, generic type alias (Go 1.24)
- **Gotcha:** Go generics do not support method-level type parameters — only function-level and type-level. You cannot write `func (s *Stack[T]) Map[U any](fn func(T) U) []U` — the `[U any]` on a method is not allowed. This limits some patterns that are natural in Haskell or Scala. The workaround is to use a package-level generic function.
- **Follow-Up:** "When should you use generics vs interfaces?" → Use interfaces when behaviour is the abstraction (different types do the same thing differently). Use generics when type is the abstraction (same algorithm works on different types). Rule of thumb: if you find yourself writing the same function for `[]int`, `[]string`, `[]float64` — that's a generic. If you find yourself writing different implementations of the same concept — that's an interface. → They're testing generics vs interfaces design judgment.
- **Conclusion:** Go generics bring type-safe, reusable algorithms to the language without sacrificing readability — but their design (no method-level type parameters, type-set constraints) reflects Go's preference for explicitness and simplicity over the full expressiveness of generics in other languages.

---

#### Functional Options Pattern

- **Question:** What is the functional options pattern, why does it exist in Go, and how does it compare to builder pattern or config struct?
- **Answer:**
  - The functional options pattern provides optional, named configuration for a constructor without requiring a large config struct or multiple constructor signatures. Introduced by Rob Pike — idiomatic in Go standard library and popular packages (`grpc.Dial`, `http.Server`).
  - **How it works:** The constructor accepts variadic `Option` functions (`...Option`). Each option is a function that modifies a private config struct. The caller provides only the options they care about; defaults handle the rest.
  - **Compared to config struct:** Config struct exposes all fields (even internal ones), zero values may be ambiguous (is `0` timeout "not set" or "0 seconds"?), and adding new options requires a struct change. Functional options are additive — new options don't break existing callers.
  - **Compared to builder pattern:** Builder requires chaining and a final `Build()` call, which returns an error that must be handled. Functional options are cleaner for cases where construction can't fail.
- **Example:**

```go
type Server struct {
    host    string
    port    int
    timeout time.Duration
    maxConn int
}

type Option func(*Server)

func WithHost(host string) Option {
    return func(s *Server) { s.host = host }
}
func WithPort(port int) Option {
    return func(s *Server) { s.port = port }
}
func WithTimeout(d time.Duration) Option {
    return func(s *Server) { s.timeout = d }
}

func NewServer(opts ...Option) *Server {
    s := &Server{ // sensible defaults
        host:    "localhost",
        port:    8080,
        timeout: 30 * time.Second,
        maxConn: 100,
    }
    for _, opt := range opts {
        opt(s) // apply each option
    }
    return s
}

// Usage — only specify what differs from defaults
srv := NewServer(
    WithPort(9090),
    WithTimeout(60 * time.Second),
)
// host and maxConn use defaults
```

- **Technical Terms to Include:** functional options, variadic options, Option type, constructor pattern, default values, additive API, backward compatibility, Rob Pike, grpc.DialOption
- **Gotcha:** Functional options make it impossible to validate the full configuration before applying all options — you only see the final state after all options run. If options have ordering dependencies or conflicts, validation must happen after all options are applied (at the end of `NewServer`), not incrementally.
- **Follow-Up:** "How do you make a functional option that requires a value but you want to enforce it's always provided?" → Use a required parameter in the constructor signature alongside the variadic options: `NewServer(host string, port int, opts ...Option)`. The required parameters cannot be omitted; options extend the optional configuration. → They're testing API design judgment for the boundary between required and optional.
- **Conclusion:** The functional options pattern is Go's idiomatic solution for constructors with optional configuration — it provides named, self-documenting options with backward-compatible extensibility, replacing the fragility of positional arguments and the verbosity of config structs.

---

## L5 — Architecture & System Design

---

**L5 — Topics identified:**

- HTTP Server Architecture (net/http, middleware, routing Go 1.22+)
- Clean Architecture & Dependency Injection in Go
- gRPC & Protocol Buffers
- Go Module Design & Internal Package Strategy
- Testing Architecture (interfaces, testify, table-driven)

---

### Architecture

#### HTTP Server Architecture

- **Question:** How do you architect a production HTTP server in Go using `net/http`, middleware, and the Go 1.22+ enhanced router?
- **Answer:**
  - Go's `net/http` stdlib is production-ready — many companies run it without a framework. Each request is handled in a **new goroutine**, providing automatic concurrency without configuration.
  - **Handler interface:** Any type with `ServeHTTP(ResponseWriter, *Request)` is an HTTP handler. `http.HandlerFunc` adapts a plain function to this interface.
  - **Middleware:** A function that wraps a handler, adding cross-cutting behaviour (auth, logging, panic recovery, rate limiting). Middleware chains are composed by wrapping handlers — the outer middleware calls the inner handler via `next.ServeHTTP`.
  - **Go 1.22 enhanced routing:** `http.ServeMux` now supports method-based routing (`GET /users/{id}`) and path parameters (`{id}` extracted via `r.PathValue("id")`). Eliminates the need for `gorilla/mux` or `chi` for most services.
  - **Production concerns:** Always set `ReadTimeout`, `WriteTimeout`, and `IdleTimeout` on `http.Server` — the zero value means no timeout, enabling slow-client attacks (Slowloris).
- **Example:**

```go
// Middleware type and chain
type Middleware func(http.Handler) http.Handler

func Chain(h http.Handler, middlewares ...Middleware) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        h = middlewares[i](h)
    }
    return h
}

// Logging middleware
func Logger(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
    })
}

// Go 1.22 routing — method + path parameters
mux := http.NewServeMux()
mux.HandleFunc("GET /users/{id}", func(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id") // Go 1.22+
    // handle GET /users/:id
})
mux.HandleFunc("POST /users", createUser)

// Production server with timeouts
srv := &http.Server{
    Addr:         ":8080",
    Handler:      Chain(mux, Logger, Auth, RateLimit),
    ReadTimeout:  5 * time.Second,
    WriteTimeout: 10 * time.Second,
    IdleTimeout:  60 * time.Second,
}
log.Fatal(srv.ListenAndServe())
```

- **Technical Terms to Include:** http.Handler, http.HandlerFunc, ServeMux, middleware, handler chain, path parameters (Go 1.22), ReadTimeout, WriteTimeout, IdleTimeout, goroutine-per-request, graceful shutdown
- **Gotcha:** `http.ListenAndServe` blocks forever and never returns `nil` — it always returns an error. `log.Fatal(srv.ListenAndServe())` is idiomatic. For graceful shutdown, use `srv.Shutdown(ctx)` in a signal handler — it stops accepting new connections, waits for active ones to finish, then returns.
- **Follow-Up:** "How do you implement graceful shutdown for a Go HTTP server?" → Listen for OS signals (`syscall.SIGINT`, `syscall.SIGTERM`) in a goroutine. On signal, call `srv.Shutdown(ctx)` with a timeout context — it stops the listener, waits for in-flight requests to complete (up to the timeout), then returns. Call `srv.Close()` if you need to force-terminate remaining connections. → They're testing production deployment lifecycle awareness.
- **Conclusion:** Go's `net/http` is a complete production HTTP stack — with Go 1.22's method routing and path parameters, third-party routers are optional for most services, and the middleware pattern provides a clean, composable way to attach cross-cutting concerns without framework lock-in.

---

#### Clean Architecture in Go

- **Question:** How do you implement Clean Architecture (or Hexagonal Architecture) in Go, and how do interfaces enable dependency inversion?
- **Answer:**
  - Clean Architecture separates concerns into layers: **Domain** (entities, business rules — no external dependencies), **Application/Use Case** (orchestrates domain, calls ports), **Infrastructure** (implements ports — database, HTTP, messaging), **Delivery** (HTTP handlers, gRPC, CLI).
  - Go interfaces enable **dependency inversion** — the use case layer defines interfaces (ports) for what it needs (a `UserRepository`, an `EmailSender`). Infrastructure provides concrete implementations. The use case never imports infrastructure — the dependency arrow points inward.
  - **In Go:** Define interfaces in the package that uses them (consumer-defined), not in the package that implements them. This is Go's structural typing strength — the `postgres.UserRepository` satisfies `user.Repository` without importing the `user` package.
  - **Dependency injection:** Pass implementations to use cases via constructor parameters (or functional options). No DI framework required for most services — manual injection in `main.go` is explicit and readable.
- **Example:**

```go
// Domain layer — no external imports
// domain/user/user.go
package user
type User struct { ID int; Email string }
type Repository interface {  // port — defined where it's used
    FindByID(ctx context.Context, id int) (*User, error)
    Save(ctx context.Context, u *User) error
}

// Application layer — orchestrates domain
// app/user/service.go
package userapp
type Service struct { repo user.Repository } // depends on interface, not postgres
func NewService(repo user.Repository) *Service { return &Service{repo: repo} }
func (s *Service) GetUser(ctx context.Context, id int) (*user.User, error) {
    return s.repo.FindByID(ctx, id)
}

// Infrastructure layer — implements the port
// infra/postgres/user_repo.go
package postgres
type UserRepo struct { db *sql.DB }
func (r *UserRepo) FindByID(ctx context.Context, id int) (*user.User, error) { /* sql query */ }
func (r *UserRepo) Save(ctx context.Context, u *user.User) error { /* sql insert */ }
// UserRepo implicitly satisfies user.Repository — no 'implements' keyword

// main.go — wire dependencies manually
db := connectDB()
repo := &postgres.UserRepo{db: db}    // concrete
svc  := userapp.NewService(repo)      // inject interface
handler := httphandler.New(svc)       // inject service
```

- **Technical Terms to Include:** Clean Architecture, Hexagonal Architecture, dependency inversion, port, adapter, use case, domain layer, infrastructure layer, interface segregation, dependency injection, internal package
- **Gotcha:** The most common mistake in Go Clean Architecture is placing interfaces in the infrastructure layer (where the implementation is) rather than the application layer (where the consumer is). This inverts the dependency arrow — the use case imports infrastructure — breaking the isolation. Always define interfaces where they're consumed.
- **Follow-Up:** "How do you test the application layer without a real database?" → The use case depends on `user.Repository` (an interface). In tests, provide an in-memory implementation (`InMemoryUserRepo`) or use a mock generated by `mockery` or `gomock`. The use case is tested in complete isolation — no database, no network, instant tests. → They're testing the testability benefit of the architecture.
- **Conclusion:** Clean Architecture in Go is enabled by structural typing — interfaces defined at the consumer naturally invert dependencies, keeping the domain and application layers free of infrastructure concerns and making the entire business logic layer independently testable.

---

## L6 — Expert Level

---

**L6 — Topics identified:**

- Go Runtime Internals (Scheduler, Memory Model, GC internals)
- Performance Profiling (pprof, trace)
- unsafe Package & cgo
- Memory Model & Happens-Before
- Cross-Cutting Concerns at Scale

---

### Expert Level

#### Performance Profiling with pprof

- **Question:** How do you profile a Go application in production, what profiles exist, and how do you interpret and act on pprof output?
- **Answer:**
  - Go's `pprof` provides six standard profiles: **CPU** (where time is spent executing), **heap** (live memory allocations), **goroutine** (stack traces of all goroutines), **allocs** (all past allocations including GC'd), **block** (where goroutines block on synchronisation), **mutex** (lock contention).
  - **CPU profiling:** `runtime/pprof.StartCPUProfile(w)` / `StopCPUProfile()` for programs. `net/http/pprof` exposes all profiles over HTTP — import `_` to register the endpoint automatically (`/debug/pprof/`). Collect with `go tool pprof http://host/debug/pprof/profile?seconds=30`.
  - **Heap profiling:** Identifies top allocators. Look for unexpected large allocators or frequent small allocations (GC pressure). `allocs` profile shows historical allocations — better for identifying hot allocation paths.
  - **`go tool pprof` commands:** `top` (top functions by time/memory), `list funcName` (annotated source), `web` (flame graph in browser — requires graphviz).
  - **Go 1.25 Flight Recorder:** In-memory rolling trace buffer — `runtime/trace` writes to a buffer that can be dumped on-demand. No pre-activation required — always on with minimal overhead.
- **Example:**

```go
// Expose pprof over HTTP — import side effect registers endpoints
import _ "net/http/pprof"

// In main:
go func() {
    log.Println(http.ListenAndServe("localhost:6060", nil))
}()

// Collect profiles:
// CPU: go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
// Heap: go tool pprof http://localhost:6060/debug/pprof/heap
// Goroutines: go tool pprof http://localhost:6060/debug/pprof/goroutine
// Block: go tool pprof http://localhost:6060/debug/pprof/block

// In pprof interactive:
// (pprof) top10         — top 10 functions by CPU time
// (pprof) list myFunc   — show annotated source for myFunc
// (pprof) web           — open flame graph in browser

// Benchmark with profiling
// go test -bench=. -cpuprofile=cpu.prof -memprofile=mem.prof ./...
// go tool pprof cpu.prof
```

- **Technical Terms to Include:** pprof, CPU profile, heap profile, goroutine profile, allocs profile, block profile, mutex profile, flame graph, go tool pprof, Flight Recorder, escape analysis, GC pressure, allocation hot path, benchmark
- **Gotcha:** Importing `net/http/pprof` in a production binary exposes sensitive profiling endpoints — never expose `/debug/pprof` publicly. Bind the pprof HTTP server to `localhost` only, or put it behind authentication. Information in goroutine and heap profiles can reveal application internals.
- **Follow-Up:** "How do you detect a goroutine leak using pprof?" → Collect the goroutine profile at two points in time (`/debug/pprof/goroutine?debug=1`). If goroutine count grows continuously and the diff shows goroutines stuck in the same function (e.g., blocking on a channel read), that function is the leak site. Go 1.26's goroutine leak profile automates this detection. → They're testing production observability depth.
- **Conclusion:** pprof is Go's production profiling system — understanding which profile to collect for which symptom (CPU for slowness, heap for memory growth, goroutine for leak, mutex for lock contention) determines whether you spend hours or minutes diagnosing a performance issue.

---

#### Go Memory Model & Happens-Before

- **Question:** What is the Go Memory Model, what does "happens-before" guarantee, and how do you write concurrent code that is safe under it?
- **Answer:**
  - The Go Memory Model defines when one goroutine is **guaranteed to observe** a write made by another goroutine. Without a happens-before relationship between a write and a read, the read may observe any value — including a stale or partially-written value.
  - **Happens-before guarantees in Go:** (1) Within a single goroutine, statements execute in order. (2) `go` statement happens-before the goroutine starts. (3) Channel send happens-before the corresponding receive completes. (4) Closing a channel happens-before a receive that returns zero because the channel is closed. (5) `sync.Mutex` Lock/Unlock pairs establish happens-before. (6) `sync.Once` ensures exactly one execution with full happens-before.
  - **Data race:** When two goroutines access the same memory concurrently, at least one is a write, and there is no synchronisation between them. Data races are undefined behaviour in the Go Memory Model — the compiler and hardware can reorder, cache, or optimise reads and writes in ways that make the race outcome unpredictable.
  - **Race detector:** `go test -race` or `go run -race` instruments the binary with race detection. Any data race encountered at runtime is reported with the stack traces of both conflicting accesses.
- **Example:**

```go
// DATA RACE — no synchronisation
var counter int
go func() { counter++ }() // write in goroutine 1
go func() { counter++ }() // write in goroutine 2
// counter is undefined — race detector will fire

// SAFE — mutex establishes happens-before
var mu sync.Mutex
var counter int
go func() {
    mu.Lock()
    counter++
    mu.Unlock()
}()
go func() {
    mu.Lock()
    counter++
    mu.Unlock()
}()

// SAFE — channel establishes happens-before
done := make(chan struct{})
go func() {
    counter++ // write
    done <- struct{}{} // send happens-before receive completes
}()
<-done // receive completes — guaranteed to see counter++

// sync.Once — guaranteed single execution
var once sync.Once
var instance *DB
func getDB() *DB {
    once.Do(func() { instance = connectDB() }) // safe for concurrent calls
    return instance
}
```

- **Technical Terms to Include:** Go Memory Model, happens-before, data race, race detector, synchronisation, atomic operation, sync.Mutex, sync.Once, channel happens-before, sync/atomic, memory visibility, cache coherence
- **Gotcha:** `sync/atomic` operations (`atomic.AddInt64`, `atomic.LoadInt64`) establish happens-before with respect to other atomic operations on the same variable — but not with respect to non-atomic accesses. If some goroutines use atomic and others use raw access on the same variable, it's still a data race. Atomics are not a replacement for mutexes when the guarded operation involves multiple variables.
- **Follow-Up:** "When would you use `sync/atomic` instead of `sync.Mutex`?" → Atomic operations are appropriate for single, simple operations on a single value — incrementing a counter, toggling a flag, publishing a value. Mutexes are required when multiple variables must be updated together (atomically as a group), or when the critical section involves multiple steps. Atomic is faster (no goroutine suspension), but the use cases are narrow. → They're testing synchronisation primitive selection judgment.
- **Conclusion:** The Go Memory Model defines the visibility contract of concurrent programs — every concurrent access without a documented happens-before relationship is a potential data race, and the race detector is the mandatory tool for catching these before they manifest as production heisenbugs.

---

_— End of GoLang_Interview_QA.md —_
_Total coverage: L1–L6 | 6 levels × 4-6 topics avg = ~34 interview-ready QA units_
_Calibrated for: Tech Lead / Architect level_
