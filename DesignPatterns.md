<!-- Generated: 2026-05-04 | Version: GoF Canonical + Modern Cloud-Native Patterns -->

# Programming Design Patterns — Interview Preparation

## Tech Lead / Architect Level

> GoF · Enterprise · Distributed Systems · Cloud-Native · Frontend Patterns
> Generated: 2026-05-04 | Career OS

---

## Scope & Lens

Design patterns at Tech Lead / Architect level are not a memorisation test.
Interviewers are testing three things:

1. **Recognition** — Can you identify the pattern being used in code you're shown?
2. **Judgment** — Can you decide when to apply a pattern vs when it adds unnecessary complexity?
3. **Trade-off articulation** — Can you explain what a pattern costs you while solving a problem?

Every answer in this file is framed from those three lenses — not from "here are the participants."

**Your production anchors:**

- **Observer/Pub-Sub:** MQTT device telemetry at Volansys (devices publish, dashboard subscribes)
- **BFF:** PayPal Checkout — Node.js BFF aggregating 4 backend services
- **Facade:** Apple Offers Platform — MFE shell hiding backend complexity behind a typed API contract
- **Strategy:** Billing/subscription payment routing at PayPal (different strategies per payment method)
- **Repository:** DynamoDB access at Volansys — data access abstracted from business logic
- **Decorator:** Splunk logging middleware wrapping Lambda handlers and API Gateway responses
- **CQRS:** Covered in EventDrivenArchitecture file — reference for extended depth

---

## L1 — Conceptual Foundations

---

L1 — Topics identified: What are design patterns, GoF categories, why patterns at Architect level, pattern vs anti-pattern, when NOT to use a pattern
L1 — Estimated questions: 8 | Topics: 5

---

- L1
  - WhatAreDesignPatterns
    - Definition & Purpose — (Q count: 2)
      - **[L1.WhatAreDesignPatterns.Definition.Q1]**
        **Question:** What is a design pattern and why does knowing pattern vocabulary matter more at Architect level than at Senior Engineer level?
      - **Answer:**
        - A design pattern is a named, reusable solution to a commonly recurring problem in software design within a given context. It is not code — it is a template for how to solve a class of problem that can be applied to many different situations.
        - **Why vocabulary matters at Architect level:** An architect communicates across teams. Saying "we'll use Observer here" in an architecture review gets immediate shared understanding. Explaining the same concept without the name takes 3 minutes of whiteboard time. Patterns are the vocabulary of technical communication — precision without verbosity.
        - **GoF (Gang of Four):** The 1994 book _Design Patterns_ by Gamma, Helm, Johnson, Vlissides codified 23 patterns into three categories: Creational (how objects are created), Structural (how objects are composed), Behavioural (how objects communicate). These remain the canonical reference.
        - **Beyond GoF:** Modern software adds: Architectural patterns (MVC, CQRS, Event Sourcing), Enterprise patterns (Fowler's PoEAA), Distributed system patterns (Circuit Breaker, Saga, Outbox), Cloud-native patterns (Sidecar, Ambassador). Architects operate at all four levels simultaneously.
        - **The real test in an interview:** Not "name the participants of the Decorator pattern" — but "you're adding cross-cutting logging to 12 Lambda functions without touching each one; what pattern solves this and what are the trade-offs?" Answer: Decorator (or Middleware/Chain of Responsibility depending on the implementation). Then articulate cost.
      - **Example:**

        ```
        Pattern vocabulary in action:
        Engineer without pattern vocabulary:
          "We need the thing where the dashboard updates automatically when
           device data changes, without the dashboard polling..."
        Architect with pattern vocabulary:
          "We'll use Observer — devices are subjects, dashboard components are
           observers. In our MQTT context that maps directly to pub/sub topics."

        Same concept. One took 30 seconds to communicate, one took 10.
        The pattern name carries 30 years of accumulated understanding.
        ```

      - **Technical Terms to Include:** Gang of Four (GoF), Creational/Structural/Behavioural categories, pattern context, pattern intent, participants, collaboration, consequences, anti-pattern, pattern language, architectural pattern, enterprise pattern
      - **Gotcha:** "Pattern" does not mean "always apply." The GoF explicitly states every pattern has a context — the set of conditions under which applying it is appropriate. An interviewer who asks "when would you NOT use a Singleton?" is testing whether you know the pattern's costs, not just its shape. The answer: when testability matters (Singletons are notoriously hard to mock/isolate in tests), when parallelism is required (Singleton shared state creates concurrency risks), or when the global state it introduces obscures dependencies.
      - **Follow-Up:** "What is the difference between a design pattern and an architectural pattern?" → A design pattern operates at the object/class level within a component — it addresses how objects collaborate. An architectural pattern operates at the system/module level — it defines the overall structure of a system (MVC separates concerns across the whole application; Observer is used within a component). The distinction is scope: patterns are within a module; architectural patterns span modules or the whole system. → They're testing level-of-abstraction precision.
      - **Conclusion:** Design patterns are a shared vocabulary for recurring solutions — their value at Architect level is communication speed and precision across teams, not pattern memorisation for its own sake.

      - **[L1.WhatAreDesignPatterns.Definition.Q2]**
        **Question:** What is an anti-pattern, what are the three most damaging ones in production systems, and how do you identify them in code review?
      - **Answer:**
        - An **anti-pattern** is a commonly used but counterproductive solution — it looks like it solves the problem, it may even work short-term, but it creates greater problems than it solves. Named anti-patterns are as valuable as named patterns — they give teams a shared vocabulary for what to avoid.
        - **The three most damaging in production:**
          1.  **God Object / God Class:** One class or module that knows too much and does too much. It accumulates responsibilities over time. In a frontend codebase: a 3,000-line `AppContext.tsx` that manages auth, cart, UI state, and API calls. Every feature touches it — merge conflicts daily, testing requires mocking the universe, and a bug in one area affects all others. Violates Single Responsibility. Fix: decompose into focused context providers or Pinia/Zustand slices.
          2.  **Magic Numbers / Strings:** Business logic embedded as literals without named constants. `if (status === 3)` — what does 3 mean? In a payment system, 3 might mean "declined." When the payment provider changes code 3 to code 7, the bug hunt takes hours. Fix: named constants or enums. At Architect level: this is a code review catch that signals a team without established standards.
          3.  **Lava Flow:** Dead code that nobody dare delete because nobody knows what it does. Happens when engineers leave without documenting their work, or when "we'll clean that up later" never comes. At scale: 20% of the codebase may be lava flow. It increases cognitive load for everyone reading the codebase. Fix: feature flags, explicit deletion with Git blame, characterisation tests before removal.
        - **Code review identification signals:** God Objects show as files with 1,000+ lines or classes imported by 50+ other modules. Magic literals show in grep: `git grep -n "[0-9]"` in business logic files. Lava Flow shows as code with no test coverage, no recent Git touches, and multiple "TODO: remove" comments.
      - **Example:**

        ```typescript
        // Anti-pattern: God Object — accumulates everything
        class AppManager {
          user: User
          cart: CartItem[]
          theme: 'light' | 'dark'
          notifications: Notification[]
          apiBaseUrl: string
          socketConnection: WebSocket

          login() { ... }
          addToCart() { ... }
          toggleTheme() { ... }
          fetchNotifications() { ... }
          connectSocket() { ... }
          formatPrice() { ... }      // utility sneaking in
          validateEmail() { ... }    // and another
        }

        // Refactored — Single Responsibility per concern
        class UserService      { login() { ... } }
        class CartService      { addItem() { ... } }
        class ThemeService     { toggle() { ... } }
        class NotificationService { fetch() { ... } }
        // Utilities in separate pure function modules
        ```

      - **Technical Terms to Include:** anti-pattern, God Object/God Class, Magic Numbers, Lava Flow, Spaghetti Code, Copy-Paste Programming, Single Responsibility Principle, technical debt, characterisation test, feature flag
      - **Gotcha:** The **Service Locator** pattern is a controversial anti-pattern — it solves dependency injection but hides dependencies behind a global registry. Code becomes hard to trace: `ServiceLocator.get(PaymentService)` buries the dependency. Modern DI containers (Angular's DI, InversifyJS) are structured alternatives that make dependencies explicit. Know the difference when an interviewer asks "what's wrong with Service Locator?"
      - **Follow-Up:** "You're reviewing a PR and see a 500-line React component. Is that always a God Component?" → Not always — a page-level component legitimately composes many child components and may have complex layout logic. The smell is not line count alone but: how many unrelated concerns does it manage (auth + form + data fetching + UI state + analytics)? Apply the "single responsibility test": if you can describe the component's responsibility without using "and", it passes. If you need "and", decompose. → They're testing nuanced code review judgment.
      - **Conclusion:** Anti-patterns are as important as patterns in an architect's vocabulary — recognising God Objects, Lava Flow, and Magic Literals in code review and naming them precisely signals technical leadership, not just critique.

  - PatternCategories
    - GoF Three Categories — (Q count: 2)
      - **[L1.PatternCategories.GoFThreeCategories.Q1]**
        **Question:** What do the three GoF categories — Creational, Structural, Behavioural — each solve, and can you give the most important pattern from each?
      - **Answer:**
        - **Creational Patterns:** Control how objects are instantiated. The problem they solve: tight coupling between client code and the specific classes it instantiates. If code calls `new PostgresDatabase()` directly, switching to `MongoDatabase` requires changing every call site. Creational patterns introduce a level of indirection — the client asks for an object through an interface, not through a constructor. **Most important: Factory Method / Abstract Factory** — used everywhere from React component factories to cloud SDK client construction.
        - **Structural Patterns:** Control how objects and classes are composed into larger structures. The problem they solve: inflexible composition — adding behaviour requires subclassing (class explosion), or existing structures can't work together. **Most important: Decorator** — adds behaviour to an object without subclassing, by wrapping it. Express.js middleware is the Decorator pattern applied to HTTP handlers. Lambda wrappers for logging/tracing are Decorators.
        - **Behavioural Patterns:** Control how objects communicate and distribute responsibility. The problem they solve: tight coupling between objects that need to interact. If `ComponentA` directly calls methods on `ComponentB`, changing `ComponentB`'s interface breaks `ComponentA`. **Most important: Observer/Pub-Sub** — the subject maintains a list of observers and notifies them of changes. React's event system, MQTT Pub/Sub, Redux's `store.subscribe()` are all Observer pattern applications.
        - **Mnemonic:** Creational = "how is it made", Structural = "how is it assembled", Behavioural = "how does it communicate".
      - **Example:**

        ```
        Pattern categories mapped to production experience:

        Creational:
          Factory Method → React.createElement() — client asks for a component,
                           framework decides which concrete class to instantiate

        Structural:
          Decorator → Splunk logging wrapper at TCS:
            const handler = withSplunkLogger(withAuth(withRateLimit(rawHandler)))
            Each wrapper adds behaviour without modifying rawHandler

        Behavioural:
          Observer → MQTT at Volansys:
            HVAC device (Subject) publishes to topic
            Dashboard components (Observers) subscribe
            Device doesn't know who's listening — loose coupling
        ```

      - **Technical Terms to Include:** Creational, Structural, Behavioural, instantiation, composition, communication, Factory Method, Decorator, Observer, coupling, indirection, class explosion, GoF 23 patterns
      - **Gotcha:** The category boundaries are blurry and interviewers sometimes test this. The Proxy pattern is Structural (it controls access to an object), but it overlaps with Decorator (which adds behaviour). The distinction: Decorator adds new behaviour, Proxy controls/restricts existing behaviour. A caching Proxy doesn't add capability — it controls when the underlying object's capability is invoked. A logging Decorator adds a new concern (logging) that wasn't there before.
      - **Follow-Up:** "Is Redux a design pattern?" → Redux is an architectural pattern — more specifically, it implements a variation of the Flux pattern with strong Elm/functional programming influences. Internally it uses Observer (subscribers to the store), Command (dispatched actions), and Mediator (the reducer mediates between actions and state). Named patterns are often compositions of multiple GoF patterns. → They're testing pattern composition awareness.
      - **Conclusion:** The three GoF categories address distinct software problems — Creational decouples instantiation, Structural decouples composition, Behavioural decouples communication — and every significant piece of production software uses patterns from all three simultaneously.

---

L1 — Actual questions: 6 | Topics covered: 4

---

## L2 — Core Patterns

---

L2 — Topics identified: Singleton, Factory/Abstract Factory, Builder (Creational); Adapter, Decorator, Facade, Proxy, Composite (Structural); Observer, Strategy, Command, Template Method, Chain of Responsibility, State (Behavioural)
L2 — Estimated questions: 22 | Topics: 14

---

- L2
  - Creational
    - Singleton — (Q count: 2)
      - **[L2.Creational.Singleton.Q1]**
        **Question:** What is the Singleton pattern, what problem does it solve, and why is it considered harmful in modern application architectures?
      - **Answer:**
        - **Definition:** Singleton ensures a class has only one instance and provides a global access point to it. It solves the problem of coordinating access to a shared resource — a database connection pool, a configuration object, a logging service — where multiple instances would cause conflicts or waste resources.
        - **Classic implementation:** Private constructor, static `instance` field, static `getInstance()` method that creates the instance on first call and returns the cached instance on subsequent calls (lazy initialisation). In module systems (Node.js, ES modules), the module itself acts as a Singleton — `import config from './config'` always returns the same module object.
        - **Why it's harmful in modern architectures:**
          1.  **Hidden global state:** Any code can access `Singleton.getInstance()` — dependencies are invisible. You cannot tell from a function's signature that it uses the Singleton. Testing becomes difficult because the Singleton carries state between tests.
          2.  **Testability:** You cannot inject a mock Singleton — you must mock the module or class statically. Jest's `jest.mock()` works around this, but it's a workaround for a pattern problem.
          3.  **Concurrency risk:** In multi-threaded environments (Go, Java), the naive Singleton is not thread-safe. The double-checked locking pattern is needed — and is error-prone. In Node.js (single-threaded), this is less of a concern.
          4.  **Violates Dependency Inversion:** High-level modules depend on a concrete Singleton rather than an abstraction.
        - **When Singleton is appropriate:** Stateless utilities (a logger that only formats and writes, with no mutable state). Configuration objects that are read-only after initialisation. ES module-level singletons in Node.js (the module system handles instantiation guarantees).
      - **Example:**

        ```typescript
        // ❌ Classic Singleton — hidden global, untestable
        class DatabasePool {
          private static instance: DatabasePool;
          private pool: Pool;

          private constructor() {
            this.pool = createPool(process.env.DB_URL);
          }

          static getInstance(): DatabasePool {
            if (!DatabasePool.instance) {
              DatabasePool.instance = new DatabasePool();
            }
            return DatabasePool.instance;
          }

          query(sql: string) {
            return this.pool.query(sql);
          }
        }

        // ❌ Consumer — hidden dependency
        function getUser(id: string) {
          return DatabasePool.getInstance().query(`SELECT * FROM users WHERE id = '${id}'`);
        }

        // ✅ Better: Dependency Injection — explicit, testable
        class UserService {
          constructor(private readonly db: DatabasePool) {}
          getUser(id: string) {
            return this.db.query(`SELECT * FROM users WHERE id = $1`, [id]);
          }
        }

        // Test — inject mock, no global state pollution
        const mockDb = { query: jest.fn().mockResolvedValue({ rows: [{ id: '1' }] }) };
        const userService = new UserService(mockDb);
        ```

      - **Technical Terms to Include:** Singleton, lazy initialisation, static instance, global access point, hidden global state, Dependency Injection, testability, module-level singleton, `getInstance()`, double-checked locking, thread safety, `jest.mock()`
      - **Gotcha:** **ES modules are already singletons.** In Node.js and browser bundlers, a module is executed once and the result is cached. `import { logger } from './logger'` returns the same object across all importers. This means the most common legitimate use case for Singleton (a shared logger, a configuration object) is already solved by the module system — you don't need to implement the pattern. Writing a classical Singleton class in a Node.js/TypeScript codebase adds ceremony for a guarantee the language already provides.
      - **Follow-Up:** "How do you convert a Singleton to a testable dependency?" → Replace `Singleton.getInstance()` calls with constructor injection. Define an interface: `interface ILogger { log(msg: string): void }`. The Singleton implements the interface. Consumers receive an `ILogger` via constructor. Tests inject a mock `ILogger`. The production app wires the Singleton implementation. This is the Singleton-to-DI migration path and one of the most common refactors in legacy codebases. → They're testing practical refactoring knowledge.
      - **Conclusion:** Singleton is the most overused GoF pattern — its global access point solves resource coordination but creates hidden dependencies and testability problems that modern Dependency Injection patterns solve more cleanly; in JavaScript/TypeScript, ES module caching makes the Singleton pattern largely unnecessary.

    - Factory Method & Abstract Factory — (Q count: 2)
      - **[L2.Creational.Factory.Q1]**
        **Question:** What is the difference between Factory Method and Abstract Factory, and what are their respective production use cases?
      - **Answer:**
        - **Factory Method:** Defines an interface for creating an object, but lets subclasses or implementations decide which class to instantiate. The creator depends on the abstract product interface — not on the concrete product class. "Let the subclass decide what to make."
        - **Abstract Factory:** A factory that creates families of related objects without specifying their concrete classes. When you need several related objects that must be consistent with each other. "Create a coordinated set of related objects."
        - **Factory Method use cases:** Any place where the type of object to create varies by context — a `createPaymentHandler(type: 'stripe' | 'paypal' | 'apple-pay')` function returns different concrete handlers. The caller only knows the `PaymentHandler` interface. Adding a new payment type = adding a new concrete class + updating the factory, not modifying consumers.
        - **Abstract Factory use cases:** UI component libraries that need to produce consistent families — `MacOSTheme.createButton()`, `MacOSTheme.createCheckbox()`, `WindowsTheme.createButton()`, `WindowsTheme.createCheckbox()`. All components from the same factory are guaranteed to be visually consistent. Your design system `ThemeProvider` is an Abstract Factory.
        - **Key difference:** Factory Method creates one product. Abstract Factory creates multiple related products that must work together. Factory Method is a single method; Abstract Factory is an object with multiple factory methods.
      - **Example:**

        ```typescript
        // Factory Method — single product, type determined at runtime
        interface NotificationSender {
          send(to: string, message: string): Promise<void>;
        }

        class EmailSender implements NotificationSender {
          /* ... */
        }
        class SMSSender implements NotificationSender {
          /* ... */
        }
        class PushSender implements NotificationSender {
          /* ... */
        }

        function createSender(channel: 'email' | 'sms' | 'push'): NotificationSender {
          const senders = { email: EmailSender, sms: SMSSender, push: PushSender };
          return new senders[channel]();
        }

        // Abstract Factory — family of related objects
        interface UIFactory {
          createButton(): Button;
          createInput(): Input;
          createModal(): Modal;
        }

        class MaterialUIFactory implements UIFactory {
          createButton() {
            return new MaterialButton();
          }
          createInput() {
            return new MaterialInput();
          }
          createModal() {
            return new MaterialModal();
          }
        }

        class TailwindUIFactory implements UIFactory {
          createButton() {
            return new TailwindButton();
          }
          createInput() {
            return new TailwindInput();
          }
          createModal() {
            return new TailwindModal();
          }
        }

        // Consumer — never touches concrete classes
        function renderForm(factory: UIFactory) {
          const btn = factory.createButton(); // always consistent family
          const inp = factory.createInput();
          // MaterialButton with MaterialInput — never a MaterialButton with TailwindInput
        }
        ```

      - **Technical Terms to Include:** Factory Method, Abstract Factory, product, creator, concrete product, product family, `createXxx`, interface, concrete class, dependency inversion, plugin architecture, registry pattern
      - **Gotcha:** A plain `if/else` or `switch` that returns different objects is not a Factory Method — it's just conditional instantiation. The Factory Method pattern specifically involves polymorphism: the factory logic is in a method that subclasses/implementations override. A `map` of type strings to constructors (`{ email: EmailSender, sms: SMSSender }`) is a **Registry** pattern — a practical simplification of Factory Method common in TypeScript/JavaScript.
      - **Follow-Up:** "Your team is adding a fifth payment method to a system. The current code is a 200-line `switch` statement creating payment handlers. What refactoring do you recommend?" → Replace the switch with a Registry (Factory Method). Create a `PaymentHandlerRegistry`: `registry.register('stripe', StripeHandler)`. Factory function: `registry.create(type)`. Adding a new payment method = write the new handler class + register it. Zero modification to existing code. Satisfies Open/Closed Principle. → They're testing OCP application to Factory evolution.
      - **Conclusion:** Factory Method decouples object creation from consumption by returning abstractions; Abstract Factory extends this to coordinated families of objects — both eliminate `new ConcreteClass()` at call sites, replacing them with interface-based creation that survives new type additions without client code changes.

    - Builder — (Q count: 2)
      - **[L2.Creational.Builder.Q1]**
        **Question:** What problem does the Builder pattern solve, and how does it differ from constructor telescoping?
      - **Answer:**
        - **Constructor telescoping:** When a class has many optional parameters, the naive approach is multiple constructor overloads — `new Request(url)`, `new Request(url, method)`, `new Request(url, method, headers)`, `new Request(url, method, headers, timeout)`, etc. This is constructor telescoping — the number of constructors explodes combinatorially. In TypeScript/JavaScript, options objects mitigate this, but they don't enforce construction order or validate intermediate state.
        - **Builder pattern:** Separates the construction of a complex object from its representation. A `Builder` class provides step-by-step methods — each method sets one aspect of the object and returns the builder (fluent interface). A terminal `build()` method validates the configuration and returns the fully constructed object.
        - **What Builder adds over options objects:**
          (1) **Validated construction** — `build()` can enforce that required fields are set and business rules are satisfied before creating the object.
          (2) **Readable construction sequences** — `new QueryBuilder().select('*').from('users').where('active = true').limit(10).build()` reads like the intent.
          (3) **Reusable partial configuration** — build a base builder, clone it, and apply specialisation.
        - **JavaScript/TypeScript idiom:** The method chaining / fluent interface version of Builder is ubiquitous — Knex.js, Prisma Client, AWS SDK v3 `Command` builders, GROQ query builders. You use Builder pattern daily without necessarily naming it.
      - **Example:**

        ```typescript
        // Problem: Complex notification with many optional fields
        // Constructor telescoping — which params are which?
        new Notification('Alert', 'System down', 'critical', true, 5000, 'admin@co.com', '#ops');

        // Builder — readable, validated
        class NotificationBuilder {
          private notification: Partial<Notification> = {};

          title(title: string): this {
            this.notification.title = title;
            return this;
          }
          message(msg: string): this {
            this.notification.message = msg;
            return this;
          }
          severity(level: 'info' | 'warning' | 'critical'): this {
            this.notification.severity = level;
            return this;
          }
          channel(channel: string): this {
            this.notification.channel = channel;
            return this;
          }
          ttl(ms: number): this {
            if (ms < 0) throw new Error('TTL must be positive');
            this.notification.ttl = ms;
            return this;
          }
          build(): Notification {
            if (!this.notification.title) throw new Error('title required');
            if (!this.notification.message) throw new Error('message required');
            return this.notification as Notification;
          }
        }

        const alert = new NotificationBuilder().title('System Alert').message('Database connection pool exhausted').severity('critical').channel('#ops').ttl(5000).build();

        // Real-world: Knex.js — Builder for SQL
        const users = await knex('users').select('id', 'email').where('active', true).orderBy('created_at', 'desc').limit(10);
        ```

      - **Technical Terms to Include:** Builder, fluent interface, method chaining, `build()`, constructor telescoping, step-by-step construction, validated construction, director, concrete builder, Knex, Prisma Client, immutable builder
      - **Gotcha:** The classic GoF Builder has a **Director** class that orchestrates the builder steps — in modern JavaScript implementations, the Director is usually omitted. The consumer calls builder methods directly. This is a practical simplification: the Director pattern makes more sense when construction sequences are reused across different contexts. When there's only one construction sequence, the Director adds ceremony without value.
      - **Follow-Up:** "When would you choose Builder over a plain TypeScript object spread?" → Builder wins when:
        (1) validation logic must run during construction (can't validate a raw object spread until you explicitly validate),
        (2) construction order matters (some properties can only be set after others are established),
        (3) the object is complex enough that a readable DSL aids maintainability. For simple DTOs with all-optional fields, a plain interface + object spread is cleaner. → They're testing practical judgment on pattern application vs simplicity.
      - **Conclusion:** Builder solves complex object construction by separating what is built from how it is assembled — its fluent interface pattern is the most widely used pattern in modern APIs, and its `build()` terminal method is where construction validation belongs, not scattered across consumer code.

  - Structural
    - Decorator — (Q count: 2)
      - **[L2.Structural.Decorator.Q1]**
        **Question:** How does the Decorator pattern work, how does it differ from subclassing, and what are its production applications in Node.js/AWS architectures?
      - **Answer:**
        - **Decorator:** Attaches additional behaviour to an object at runtime by wrapping it in a decorator object that implements the same interface. The decorator forwards calls to the wrapped object and adds its own behaviour before or after. It is a structural alternative to subclassing for extending behaviour.
        - **Subclassing problems:** To add logging to `UserService`, you'd create `LoggingUserService extends UserService`. Now you want auth checking — `AuthLoggingUserService extends LoggingUserService`. Now you want caching — three additional subclasses for each combination. This is class explosion — n behaviours × m classes = n\*m subclasses. Decorator eliminates this by composing behaviours.
        - **Decorator in Node.js — middleware pattern:** Express.js, Koa, and AWS Lambda wrappers are all Decorator pattern implementations. Each middleware wraps the next handler, adds behaviour (logging, auth, rate limiting), and either invokes the next or short-circuits.
        - **Production application at TCS — Splunk wrapper:** Lambda handlers wrapped with a `withSplunkLogger` decorator — the decorator logs the invocation, calls the wrapped handler, logs the response, and forwards any errors. The handler itself is unchanged. Adding a new cross-cutting concern (distributed tracing) = write a new decorator, compose it in.
        - **TypeScript decorators (`@`):** The TypeScript decorator syntax (`@Injectable`, `@Controller`, `@Get`) are a compile-time decorator pattern — they modify class/method metadata rather than wrapping behaviour at runtime. Angular, NestJS, and TypeORM use these heavily. Know the distinction: runtime Decorator (wrapping) vs. compile-time decorator (metadata/annotation).
      - **Example:**

        ```typescript
        // Lambda handler Decorator pattern — cross-cutting concerns
        type LambdaHandler = (event: APIGatewayEvent, ctx: Context) => Promise<APIGatewayProxyResult>;

        // Decorator 1 — Logging
        function withLogging(handler: LambdaHandler): LambdaHandler {
          return async (event, ctx) => {
            console.log('REQ', { path: event.path, method: event.httpMethod });
            try {
              const result = await handler(event, ctx);
              console.log('RES', { statusCode: result.statusCode });
              return result;
            } catch (err) {
              console.error('ERR', err);
              throw err;
            }
          };
        }

        // Decorator 2 — Auth validation
        function withAuth(handler: LambdaHandler): LambdaHandler {
          return async (event, ctx) => {
            const token = event.headers?.Authorization;
            if (!token || !isValidToken(token)) {
              return { statusCode: 401, body: JSON.stringify({ error: 'Unauthorized' }) };
            }
            return handler(event, ctx);
          };
        }

        // Decorator 3 — Rate limiting
        function withRateLimit(handler: LambdaHandler): LambdaHandler {
          return async (event, ctx) => {
            const ip = event.requestContext.identity.sourceIp;
            if (await isRateLimited(ip)) {
              return { statusCode: 429, body: JSON.stringify({ error: 'Too Many Requests' }) };
            }
            return handler(event, ctx);
          };
        }

        // Composition — behaviours stack, handler is untouched
        const getUser = async (event: APIGatewayEvent, ctx: Context) => {
          // Pure business logic — no cross-cutting concerns here
          const userId = event.pathParameters?.id;
          const user = await userService.getById(userId);
          return { statusCode: 200, body: JSON.stringify(user) };
        };

        export const handler = withRateLimit(withAuth(withLogging(getUser)));
        // Rate limit → Auth check → Log → Business logic → Log response
        ```

      - **Technical Terms to Include:** Decorator, runtime composition, class explosion, middleware pattern, cross-cutting concern, `withXxx` wrapper, TypeScript `@` decorator, metadata, Express middleware, Lambda wrapper, composition over inheritance
      - **Gotcha:** The Decorator pattern requires the decorator to implement the **same interface** as the wrapped object. If `LoggingUserService` adds a `getLogs()` method not on `UserService`'s interface, it is no longer a pure Decorator — it is a subclass with extended interface. The constraint is important: Decorators must be substitutable for the original, or consumers can't be agnostic to whether they're dealing with a decorated or undecorated object.
      - **Follow-Up:** "How does Decorator differ from Proxy?" → Both wrap an object. **Decorator adds new behaviour** — it extends what the object does. **Proxy controls access** — it intercepts calls to the same interface, but the intent is access control or caching, not enhancement. A logging wrapper is Decorator (adds logging). A caching layer that returns cached results without calling the underlying service is Proxy. A lazy-loading wrapper that defers object creation until first access is also Proxy. Intent is the distinguishing factor. → They're testing Decorator/Proxy distinction.
      - **Conclusion:** Decorator is composition over inheritance made explicit — each decorator is a single-responsibility wrapper that can be stacked in any order, making it the correct pattern for cross-cutting concerns (logging, auth, caching, rate limiting) that would otherwise require modifying or subclassing the wrapped object.

    - Facade — (Q count: 2)
      - **[L2.Structural.Facade.Q1]**
        **Question:** What is the Facade pattern, how does it simplify complex subsystems, and how does it appear in BFF architecture?
      - **Answer:**
        - **Facade:** Provides a simplified interface to a complex subsystem. The facade doesn't add functionality — it reduces the interface surface that clients need to interact with. The subsystem remains accessible directly for those who need its full power; the facade serves clients who need a simpler view.
        - **Problem it solves:** A complex subsystem with 15 classes, multiple initialisation sequences, and intricate interdependencies. Every client that needs any part of it must understand the whole. A Facade presents a simple, task-oriented API — `facade.generateReport()` instead of: `const extractor = new DataExtractor(); const transformer = new DataTransformer(); const aggregator = new ReportAggregator(extractor, transformer); const formatter = new ReportFormatter(); formatter.format(aggregator.aggregate())`.
        - **BFF as Facade:** The Node.js Backend-for-Frontend at PayPal was a Facade pattern — the frontend made one call to the BFF, which internally orchestrated calls to 4 backend services (payment, billing, subscription, user/identity), aggregated their responses, and returned a single contract tailored to the checkout UI. The frontend had no knowledge of the 4 services — it only knew the BFF's simplified interface.
        - **MFE shell as Facade:** The Apple Offers Platform MFE shell acted as a Facade — remote micro-frontends accessed configuration, auth tokens, and shared services through the shell's public API rather than directly. The shell's internal implementation (how it fetched auth, which endpoints it called) was hidden behind the API contract.
        - **Facade vs Adapter:** Facade simplifies a subsystem for a client. Adapter makes two incompatible interfaces work together. A Facade may internally use Adapters to bridge subsystem components — they are complementary.
      - **Example:**

        ```typescript
        // Complex subsystem — e-commerce order processing
        class InventoryService    { reserve(items: Item[]): Promise<Reservation> { ... } }
        class PaymentGateway      { charge(amount: number, token: string): Promise<Charge> { ... } }
        class FulfillmentService  { ship(orderId: string, reservation: Reservation): Promise<Shipment> { ... } }
        class NotificationService { sendOrderConfirmation(email: string, order: Order): Promise<void> { ... } }
        class AuditLogger         { log(event: AuditEvent): void { ... } }

        // Without Facade — consumer knows every subsystem
        async function checkout(cart: Cart, payment: PaymentDetails) {
          const reservation = await new InventoryService().reserve(cart.items)
          const charge = await new PaymentGateway().charge(cart.total, payment.token)
          const shipment = await new FulfillmentService().ship(cart.id, reservation)
          await new NotificationService().sendOrderConfirmation(payment.email, { charge, shipment })
          new AuditLogger().log({ event: 'ORDER_PLACED', orderId: cart.id })
        }

        // Facade — simplified interface hides subsystem complexity
        class CheckoutFacade {
          constructor(
            private inventory: InventoryService,
            private payment: PaymentGateway,
            private fulfillment: FulfillmentService,
            private notification: NotificationService,
            private audit: AuditLogger,
          ) {}

          async placeOrder(cart: Cart, paymentDetails: PaymentDetails): Promise<Order> {
            const reservation = await this.inventory.reserve(cart.items)
            const charge      = await this.payment.charge(cart.total, paymentDetails.token)
            const shipment    = await this.fulfillment.ship(cart.id, reservation)
            await this.notification.sendOrderConfirmation(paymentDetails.email, { charge, shipment })
            this.audit.log({ event: 'ORDER_PLACED', orderId: cart.id })
            return { id: cart.id, charge, shipment }
          }
        }

        // Consumer — knows only the Facade's simple interface
        const order = await checkoutFacade.placeOrder(cart, paymentDetails)
        ```

      - **Technical Terms to Include:** Facade, subsystem, simplified interface, BFF (Backend-for-Frontend), API gateway, abstraction layer, `placeOrder`, subsystem isolation, Adapter vs Facade, coupling reduction
      - **Gotcha:** Facade can become a **God Object** if it expands to handle every use case across all subsystems. The discipline is to keep the Facade task-oriented and thin — it orchestrates, it doesn't own business logic. A Facade that starts making business decisions (if payment fails, determine whether to partially fulfil the order based on...) has crossed into an Anti-pattern. Business logic belongs in the subsystem services; the Facade coordinates them.
      - **Follow-Up:** "How does the BFF pattern relate to both Facade and Anti-Corruption Layer?" → The BFF is a Facade in that it provides a simplified interface to multiple backend services. It is also an Anti-Corruption Layer (DDD pattern) — it translates between the backend service's domain model and the frontend's domain model, preventing the frontend from being "corrupted" by the backend's internal data structures. A BFF that just passes through backend responses without translation is only a Facade; one that actively translates models is also an ACL. → They're testing DDD pattern awareness in frontend architecture.
      - **Conclusion:** Facade is the pattern that makes complex systems approachable — by presenting a task-oriented API over a multi-service subsystem, it reduces client coupling, hides subsystem evolution, and is the conceptual foundation of BFF architecture and API gateway design.

  - Behavioural
    - Observer — (Q count: 2)
      - **[L2.Behavioural.Observer.Q1]**
        **Question:** What is the Observer pattern, how does it differ from Pub/Sub, and what are the failure modes of each in distributed systems?
      - **Answer:**
        - **Observer:** A subject (or publisher) maintains a list of observers (or subscribers) and notifies them directly when its state changes. The subject knows its observers — it holds direct references. Communication is synchronous in the classic pattern. Subject and observer are typically in the same process.
        - **Pub/Sub (Publish-Subscribe):** An asynchronous variant where publishers and subscribers are decoupled through a message broker (Kafka, SQS, Redis Pub/Sub, MQTT). The publisher does not know who subscribes. The broker routes messages. Publishers and subscribers may be in different processes, different services, different regions.
        - **Key difference:** Observer = direct notification, same process, publisher knows observers. Pub/Sub = broker-mediated notification, decoupled processes, publisher is unaware of subscribers.
        - **Observer failure modes:**
          (1) Memory leaks — observers that are never unregistered (removed from the subject's list) cause the subject to hold references to dead objects. In JavaScript DOM listeners: `element.addEventListener('click', handler)` without a corresponding `removeEventListener` in cleanup.
          (2) Notification order dependency — observers that depend on being notified before others create fragile ordering assumptions.
          (3) Cascading updates — an observer mutating the subject during notification triggers another notification cycle (infinite loop).
        - **Pub/Sub failure modes:**
          (1) Message loss — if the subscriber is down when the message is published and there's no durable queue, the message is lost.
          (2) Message ordering — out-of-order delivery in distributed systems (SQS does not guarantee ordering; Kafka guarantees ordering per partition).
          (3) Consumer lag — subscribers processing slower than publishers produce.
          (4) Duplicate delivery — at-least-once delivery requires idempotent consumers.
      - **Example:**

        ```typescript
        // Observer pattern — same process, direct notification (EventEmitter in Node.js)
        import { EventEmitter } from 'events';

        class DeviceStateManager extends EventEmitter {
          private state: DeviceState = {};

          updateState(deviceId: string, newState: Partial<DeviceState>) {
            this.state[deviceId] = { ...this.state[deviceId], ...newState };
            this.emit('stateChanged', { deviceId, state: this.state[deviceId] });
          }
        }

        const manager = new DeviceStateManager();
        // Observers register directly — manager knows about them
        manager.on('stateChanged', ({ deviceId, state }) => {
          dashboard.updateDevice(deviceId, state); // observer 1
        });
        manager.on('stateChanged', ({ deviceId, state }) => {
          alertService.checkThresholds(deviceId, state); // observer 2
        });
        // ⚠️ Memory leak risk: if dashboard/alertService are destroyed,
        // these listeners must be removed: manager.off('stateChanged', handler)

        // Pub/Sub — MQTT broker mediates (Volansys IIoT pattern)
        // Publisher (HVAC device firmware) — knows nothing about subscribers
        mqttClient.publish('devices/hvac-001/telemetry', JSON.stringify({ temp: 22.5 }));

        // Subscriber 1 (dashboard service) — subscribes via broker
        mqttClient.subscribe('devices/+/telemetry', (topic, message) => {
          dashboard.update(parseDeviceId(topic), JSON.parse(message));
        });
        // Subscriber 2 (alert service) — independent, broker routes to both
        mqttClient.subscribe('devices/+/telemetry', (topic, message) => {
          alerts.checkThresholds(JSON.parse(message));
        });
        // Publisher is unaware of subscribers — loose coupling
        ```

      - **Technical Terms to Include:** Observer, subject, observer registration, `addEventListener`, `EventEmitter`, `emit`, `on`, `off`, Pub/Sub, message broker, MQTT, Kafka, SQS, consumer lag, memory leak, duplicate delivery, at-least-once delivery, idempotent consumer
      - **Gotcha:** React's `useEffect` hook is an Observer pattern — the component observes state and props changes, and the effect runs when observed values change. The dependency array is the subscription list. The cleanup function returned from `useEffect` is the **unsubscribe** — failing to return cleanup from an effect that adds event listeners is the React equivalent of an Observer memory leak.
      - **Follow-Up:** "In a React application with real-time WebSocket updates, how do you implement the Observer pattern without memory leaks?" → Subscribe in `useEffect`, unsubscribe in the cleanup: `useEffect(() => { const ws = new WebSocket(url); ws.onmessage = handler; return () => ws.close() }, [url])`. The dependency array `[url]` is the subscription key — when `url` changes, the cleanup runs (closing the old WebSocket) before the new subscription opens. Without the cleanup return, every re-render with a new URL adds another WebSocket without closing the old one. → They're testing React-specific Observer lifecycle management.
      - **Conclusion:** Observer and Pub/Sub solve the same coupling problem at different scales — Observer is direct, synchronous, same-process notification ideal for component-level reactive updates; Pub/Sub is broker-mediated, asynchronous, cross-process notification ideal for distributed systems — and both require explicit cleanup of subscriptions to prevent resource leaks.

    - Strategy — (Q count: 2)
      - **[L2.Behavioural.Strategy.Q1]**
        **Question:** What is the Strategy pattern, how does it enable open/closed behaviour variation, and how does it appear in payment processing systems?
      - **Answer:**
        - **Strategy:** Defines a family of algorithms, encapsulates each one, and makes them interchangeable. The context object delegates to a strategy interface rather than implementing the algorithm directly. The algorithm can be changed at runtime by swapping the strategy.
        - **Problem solved:** When you have multiple variations of an algorithm (sorting algorithms, payment methods, discount calculation rules, shipping cost calculation) and the variation is hardcoded as `if/else` or `switch`, adding a new variation requires modifying the context class. Strategy externalises each variation into its own class, satisfying the Open/Closed Principle — open for extension (new strategy class), closed for modification (context class unchanged).
        - **Payment processing at PayPal:** The BFF routing logic for payment methods (PayPal, card, BNPL, bank transfer) maps cleanly to Strategy. Each payment method is a concrete strategy implementing `PaymentStrategy.process(amount, details)`. The checkout context delegates to the appropriate strategy based on the user's selection. Adding Apple Pay = new `ApplePayStrategy` class, registered in the strategy map. Zero changes to the checkout flow.
        - **Functional equivalent in TypeScript:** Strategy pattern reduces to passing a function — `processPayment(amount, strategyFn)` where `strategyFn` is `(amount) => Promise<Charge>`. In functional languages, higher-order functions ARE the Strategy pattern without the class ceremony.
        - **Strategy vs Template Method:** Both vary an algorithm. Strategy varies the whole algorithm (swappable object). Template Method fixes the algorithm skeleton and varies specific steps (subclass overrides). Strategy is delegation; Template Method is inheritance.
      - **Example:**

        ```typescript
        // Strategy interface
        interface DiscountStrategy {
          calculate(price: number, quantity: number): number;
        }

        // Concrete strategies
        class NoDiscount implements DiscountStrategy {
          calculate(price: number, quantity: number) {
            return price * quantity;
          }
        }

        class BulkDiscount implements DiscountStrategy {
          constructor(
            private threshold: number,
            private rate: number
          ) {}
          calculate(price: number, quantity: number) {
            const base = price * quantity;
            return quantity >= this.threshold ? base * (1 - this.rate) : base;
          }
        }

        class SeasonalDiscount implements DiscountStrategy {
          constructor(private rate: number) {}
          calculate(price: number, quantity: number) {
            return price * quantity * (1 - this.rate);
          }
        }

        // Context — delegates to strategy
        class PriceCalculator {
          constructor(private strategy: DiscountStrategy) {}

          setStrategy(strategy: DiscountStrategy) {
            this.strategy = strategy; // swappable at runtime
          }

          total(price: number, quantity: number): number {
            return this.strategy.calculate(price, quantity);
          }
        }

        // Runtime strategy selection (payment method example)
        const strategies: Record<string, DiscountStrategy> = {
          none: new NoDiscount(),
          bulk: new BulkDiscount(10, 0.15),
          seasonal: new SeasonalDiscount(0.2)
        };

        const calculator = new PriceCalculator(strategies['bulk']);
        console.log(calculator.total(100, 15)); // 1275 (15% discount)

        calculator.setStrategy(strategies['seasonal']);
        console.log(calculator.total(100, 15)); // 1200 (20% discount)
        ```

      - **Technical Terms to Include:** Strategy, context, concrete strategy, strategy interface, Open/Closed Principle, algorithm family, runtime substitution, higher-order function, Template Method (comparison), payment strategy, `setStrategy`, policy pattern
      - **Gotcha:** Strategy and **Policy** are often used interchangeably. In DDD contexts, a Policy (or Rule) is a domain-level Strategy — it encapsulates a business rule that varies by context (retry policy, pricing policy, access policy). If an interviewer asks about "policy objects," they're asking about Strategy applied to domain rules. Know the vocabulary equivalence.
      - **Follow-Up:** "How do you test a system that uses Strategy without testing all concrete strategies in the integration test?" → The context should be tested with a mock/stub strategy that has predictable behaviour — this tests that the context correctly delegates to the strategy. Each concrete strategy is tested independently in isolation (unit test). The strategy registry (which strategy maps to which key) is tested in a small integration test. This separation means adding a new strategy doesn't require changing integration tests — only a new unit test for the new strategy. → They're testing Strategy's testability advantage.
      - **Conclusion:** Strategy is the correct pattern when behaviour varies by type and new variations are expected — it replaces `if/else` chains with pluggable objects that satisfy OCP, making payment method routing, discount calculation, and rendering algorithm selection open to extension without touching the orchestrating context.

    - Command — (Q count: 2)
      - **[L2.Behavioural.Command.Q1]**
        **Question:** What is the Command pattern, what does it enable beyond simple method calls, and how does it appear in undo/redo and queue-based systems?
      - **Answer:**
        - **Command:** Encapsulates a request as an object — the request's parameters, receiver, and execution logic are all bundled into a Command object. The invoker (caller) operates on the Command interface without knowing the specific request. Commands can be queued, logged, undone, or retransmitted.
        - **What it enables beyond method calls:**
          (1) **Undo/Redo** — if every state mutation is a Command, each Command can implement an `undo()` method that reverts the mutation. A history stack of Commands enables arbitrary undo/redo.
          (2) **Transaction log** — Commands are serialisable — they can be persisted and replayed. Event Sourcing is Command pattern applied at the architectural level.
          (3) **Queuing and scheduling** — Commands can be queued for deferred execution. A task queue (BullMQ, AWS SQS) stores serialised Commands and executes them asynchronously.
          (4) **Macro recording** — a sequence of Commands can be saved and replayed (UI test recording, terminal session playback).
        - **Redux as Command:** Redux actions are Commands — each action is an immutable object describing what should happen. The dispatcher is the invoker. The reducer is the receiver. Redux DevTools' time-travel debugging is exactly the Command pattern enabling replay of the action (Command) history.
        - **AWS SDK v3 pattern:** Every AWS SDK v3 operation is a Command — `new GetObjectCommand({ Bucket, Key })` is the Command; `s3Client.send(command)` is the invoker. This separates command construction (which can be tested without network calls) from execution.
      - **Example:**

        ```typescript
        // Command pattern — text editor undo/redo
        interface Command {
          execute(): void;
          undo(): void;
        }

        class InsertTextCommand implements Command {
          constructor(
            private editor: TextEditor,
            private position: number,
            private text: string
          ) {}

          execute() {
            this.editor.insert(this.position, this.text);
          }
          undo() {
            this.editor.delete(this.position, this.text.length);
          }
        }

        class DeleteTextCommand implements Command {
          private deletedText: string = '';

          constructor(
            private editor: TextEditor,
            private position: number,
            private length: number
          ) {}

          execute() {
            this.deletedText = this.editor.getText(this.position, this.length);
            this.editor.delete(this.position, this.length);
          }
          undo() {
            this.editor.insert(this.position, this.deletedText);
          }
        }

        class CommandHistory {
          private history: Command[] = [];
          private redoStack: Command[] = [];

          execute(command: Command) {
            command.execute();
            this.history.push(command);
            this.redoStack = []; // new command clears redo stack
          }

          undo() {
            const cmd = this.history.pop();
            if (cmd) {
              cmd.undo();
              this.redoStack.push(cmd);
            }
          }

          redo() {
            const cmd = this.redoStack.pop();
            if (cmd) {
              cmd.execute();
              this.history.push(cmd);
            }
          }
        }

        // AWS SDK v3 — Command pattern in production
        import { S3Client, GetObjectCommand } from '@aws-sdk/client-s3';
        const client = new S3Client({ region: 'us-east-1' });
        const command = new GetObjectCommand({ Bucket: 'my-bucket', Key: 'file.json' });
        const response = await client.send(command);
        ```

      - **Technical Terms to Include:** Command, invoker, receiver, `execute()`, `undo()`, command history, redo stack, Redux action, event sourcing, serialisable command, task queue, BullMQ, AWS SDK v3 Command, macro recording, transaction log
      - **Gotcha:** Command pattern's `undo()` is only reliable if commands are **idempotent and reversible**. A command that writes to an external system (sends an email, charges a credit card) cannot be easily undone by calling `undo()` — the external side effect has already occurred. For irreversible external commands, "undo" means issuing a compensating command (send a cancellation email, issue a refund) — which is the Saga pattern's compensating transaction, applied at the domain level.
      - **Follow-Up:** "How does the Command pattern relate to CQRS?" → In CQRS, write operations are Commands (change state) and read operations are Queries (return state, no side effects). The Command in CQRS is the same concept as the GoF Command — an immutable object representing intent to change state. The CQRS architectural pattern formalises Command/Query separation at the API boundary level. → They're testing GoF-to-architectural pattern mapping.
      - **Conclusion:** Command pattern transforms method calls into first-class objects — enabling undo/redo, queuing, logging, and replay that simple function calls cannot provide; Redux's action/reducer model and AWS SDK v3's Command objects are its most recognisable modern implementations.

    - Chain of Responsibility — (Q count: 2)
      - **[L2.Behavioural.ChainOfResponsibility.Q1]**
        **Question:** How does Chain of Responsibility work and how does it manifest in Express middleware and AWS API Gateway request pipelines?
      - **Answer:**
        - **Chain of Responsibility:** Passes a request along a chain of handlers. Each handler either processes the request or passes it to the next handler. The sender doesn't know which handler will process the request. The chain can be dynamically configured.
        - **Middleware pipeline:** Every Node.js middleware framework (Express, Koa, Fastify) implements Chain of Responsibility. Each middleware is a handler — it receives the request, either handles it (returns a response) or calls `next()` to pass to the next middleware. The chain: logging middleware → auth middleware → rate limit middleware → route handler. Any handler can short-circuit the chain by not calling `next()`.
        - **Difference from Decorator:** Decorator always delegates to the wrapped object and adds behaviour. Chain of Responsibility may break the chain — a handler can decide the request should not continue (return 401, return 429). Decorator's wrapped object always executes; Chain of Responsibility's handlers are conditional.
        - **AWS API Gateway:** Request pipeline: WAF rules → request validators → usage plan check → Lambda authoriser → integration. Each stage is a handler in the chain. A WAF rule can block the request before it reaches the Lambda authoriser.
        - **Validation chains:** A sequence of validators where each validates one aspect of the input and either continues or returns an error. `validateSchema` → `validateBusinessRules` → `validatePermissions` → `processRequest`. The first failing validator short-circuits without executing the rest.
      - **Example:**

        ```typescript
        // Chain of Responsibility — request validation pipeline
        type RequestHandler = (req: Request, next: () => Promise<Response>) => Promise<Response>;

        function createPipeline(...handlers: RequestHandler[]): RequestHandler {
          return (req, finalNext) => handlers.reduceRight((next, handler) => () => handler(req, next), finalNext)();
        }

        const authHandler: RequestHandler = async (req, next) => {
          if (!req.headers.authorization) {
            return { status: 401, body: 'Unauthorized' };
          }
          return next(); // pass to next handler
        };

        const rateLimitHandler: RequestHandler = async (req, next) => {
          if (await isRateLimited(req.ip)) {
            return { status: 429, body: 'Too Many Requests' };
          }
          return next();
        };

        const loggingHandler: RequestHandler = async (req, next) => {
          console.log('REQ', req.method, req.path);
          const response = await next();
          console.log('RES', response.status);
          return response;
        };

        const pipeline = createPipeline(loggingHandler, authHandler, rateLimitHandler);
        // loggingHandler → authHandler → rateLimitHandler → actual handler
        // Any handler can stop the chain by returning without calling next()
        ```

      - **Technical Terms to Include:** Chain of Responsibility, handler, `next()`, middleware pipeline, short-circuit, Express middleware, Koa middleware, API Gateway pipeline, WAF, Lambda authoriser, validation chain, request pipeline
      - **Gotcha:** The order of handlers in a Chain of Responsibility matters significantly — and it's not always obvious. Logging should come before auth so that unauthorised requests are still logged. Auth should come before rate limiting so that unauthenticated requests aren't counted against rate limits (or should they? — depends on the threat model). This ordering is a business/security decision, not a technical one. When reviewing middleware order in a PR, always ask: "What happens if each handler fires in this order when X fails?"
      - **Follow-Up:** "How does Chain of Responsibility differ from an interceptor in Angular or Axios?" → Angular HTTP interceptors and Axios interceptors implement Chain of Responsibility — each interceptor can process/modify the request or response and pass it along. The difference from classic CoR: interceptors in these frameworks always pass to the next (they don't typically short-circuit for error handling — errors propagate via the error handler path). They're two-directional (request interceptors on the way out, response interceptors on the way back) — an extension of the pattern. → They're testing recognition of pattern variants across frameworks.
      - **Conclusion:** Chain of Responsibility is the middleware pattern formalised — it decouples request senders from processors by routing requests through a configurable chain of handlers, each of which can process, transform, or short-circuit; Express, Koa, API Gateway, and validation pipelines are all Chain of Responsibility in production.

---

L2 — Actual questions: 18 | Topics covered: 10

---

## L3 — Architectural Patterns

---

L3 — Topics identified: MVC / MVP / MVVM, Repository pattern, Dependency Injection & IoC, CQRS (overview — depth in EDA file), Saga overview, BFF deep dive, Module pattern, Service pattern
L3 — Estimated questions: 14 | Topics: 8

---

- L3
  - ArchitecturalPatterns
    - Repository Pattern — (Q count: 2)
      - **[L3.ArchitecturalPatterns.Repository.Q1]**
        **Question:** What is the Repository pattern, what does it abstract, and why is it important for testability in a DynamoDB or MongoDB-backed service?
      - **Answer:**
        - **Repository:** Provides a collection-like interface for accessing domain objects — abstracting the data store behind a domain-oriented API. Consumers call `userRepository.findById(id)` or `userRepository.save(user)` — they don't know whether the store is DynamoDB, PostgreSQL, MongoDB, or an in-memory cache.
        - **What it abstracts:** The query language (SQL, DynamoDB expression syntax, GROQ), the data mapping (DB row → domain object), the connection management, and the data access library (`$wpdb`, Prisma, `aws-sdk/DynamoDBDocumentClient`). The domain layer speaks domain language; the repository translates to storage language.
        - **Production at Volansys:** IIoT device state stored in DynamoDB. A `DeviceRepository` with `findByDeviceId(id)`, `updateState(deviceId, state)`, `listActiveDevices()` hid the DynamoDB `DocumentClient` calls, the expression attribute name mappings, and the pagination cursor logic. The service layer tested against a mock `DeviceRepository` — no real DynamoDB calls in unit tests.
        - **Testing benefit:** A unit test for `DeviceService.alertIfOverheat()` injects a mock `DeviceRepository` that returns controlled test data — no DynamoDB connection required, no test data setup, no cleanup. The test runs in milliseconds and is deterministic. Without Repository, the service has `DocumentClient` calls embedded — mocking at that level requires `aws-sdk-client-mock` and is significantly more brittle.
        - **Repository vs DAO:** Repository is domain-oriented — it returns domain objects and understands domain queries. DAO (Data Access Object) is table-oriented — it returns raw rows/documents and mirrors the storage schema. Repository is the DDD-aligned pattern; DAO is a simpler, less opinionated version.
      - **Example:**

        ```typescript
        // Repository interface — domain-oriented, storage-agnostic
        interface DeviceRepository {
          findById(deviceId: string): Promise<Device | null>
          findActiveDevices(): Promise<Device[]>
          updateState(deviceId: string, state: DeviceState): Promise<void>
          save(device: Device): Promise<void>
        }

        // DynamoDB implementation — storage details hidden
        class DynamoDeviceRepository implements DeviceRepository {
          constructor(private client: DynamoDBDocumentClient, private tableName: string) {}

          async findById(deviceId: string): Promise<Device | null> {
            const result = await this.client.send(new GetCommand({
              TableName: this.tableName,
              Key: { PK: `DEVICE#${deviceId}`, SK: 'STATE' },
            }))
            return result.Item ? this.toDomain(result.Item) : null
          }

          private toDomain(item: Record<string, unknown>): Device {
            return { id: item.deviceId as string, state: item.state as DeviceState, ... }
          }
          // ... other methods
        }

        // Service — depends on interface, not implementation
        class DeviceService {
          constructor(private deviceRepo: DeviceRepository) {}

          async alertIfOverheat(deviceId: string): Promise<void> {
            const device = await this.deviceRepo.findById(deviceId)
            if (!device) throw new Error('Device not found')
            if (device.state.temp > 80) await this.alertService.send(deviceId, 'OVERHEAT')
          }
        }

        // Unit test — mock repository, no DynamoDB
        const mockRepo: DeviceRepository = {
          findById: jest.fn().mockResolvedValue({ id: 'hvac-001', state: { temp: 85 } }),
          findActiveDevices: jest.fn(),
          updateState: jest.fn(),
          save: jest.fn(),
        }
        const service = new DeviceService(mockRepo)
        await service.alertIfOverheat('hvac-001')
        expect(alertService.send).toHaveBeenCalledWith('hvac-001', 'OVERHEAT')
        ```

      - **Technical Terms to Include:** Repository, domain object, storage abstraction, `findById`, `save`, DynamoDB DocumentClient, DAO vs Repository, unit test isolation, mock repository, `aws-sdk-client-mock`, expression attribute names, domain model, anti-corruption layer
      - **Gotcha:** Repository's abstraction promise is only kept if the interface is designed around **domain language**, not storage language. `repository.query("SELECT * FROM users WHERE status = 'active'")` is not a Repository — it leaks SQL into the domain. `repository.findActiveUsers()` is a Repository. If a repository method accepts raw query strings or DynamoDB `FilterExpression` strings, the abstraction is broken — the consumer is coupled to the storage technology through the interface.
      - **Follow-Up:** "Your team is switching from DynamoDB to PostgreSQL. How much of the codebase changes with a well-implemented Repository pattern?" → Only the concrete repository implementation changes — `DynamoDeviceRepository` is replaced by `PostgresDeviceRepository`. The domain model, service layer, business logic, tests, and API handlers are untouched. The repository interface remains the contract. This is the payoff: a well-implemented Repository makes data store migration a single-component swap rather than a full-stack change. → They're testing the practical benefit of the abstraction.
      - **Conclusion:** Repository abstracts data access behind a domain-oriented interface — hiding query language, data mapping, and connection management — making services testable in isolation from the data store and data store migrations a single implementation swap rather than a systemic change.

    - Dependency Injection & IoC — (Q count: 2)
      - **[L3.ArchitecturalPatterns.DI.Q1]**
        **Question:** What is Dependency Injection, what problem does it solve, and what are the three types of injection?
      - **Answer:**
        - **The problem:** When a class creates its own dependencies with `new ConcreteClass()`, it is tightly coupled to that class. Replacing the dependency (for testing, for different environments, for different implementations) requires modifying the class. The class violates Dependency Inversion — it depends on a concrete class, not an abstraction.
        - **Dependency Injection:** The class declares its dependencies (via interface) but does not create them. The dependencies are "injected" from outside — by a constructor, method, or property. The class depends on an abstraction; the concrete implementation is provided by the caller.
        - **Inversion of Control (IoC):** DI is the mechanism; IoC is the principle. IoC inverts who controls object creation — from the class itself (traditional: `this.service = new Service()`) to an external container or caller. DI containers (Angular's DI, InversifyJS, TSyringe) automate the wiring — you declare dependencies, the container resolves and injects them.
        - **Three injection types:**
          1.  **Constructor injection:** Dependencies declared in the constructor — most common, makes dependencies explicit at construction time. IDE can show all dependencies. Immutable after construction.
          2.  **Method injection (setter injection):** Dependency set via a method — used when the dependency is optional or needs to change after construction. Less transparent.
          3.  **Property injection:** Dependency set directly on a property — framwork-controlled (Angular `@Input`, NestJS `@Inject`). Least explicit.
      - **Example:**

        ```typescript
        // ❌ Without DI — tightly coupled, untestable
        class OrderService {
          private emailer = new SmtpEmailer(); // creates own deps
          private repo = new DynamoOrderRepository(); // couples to DynamoDB

          async placeOrder(order: Order) {
            await this.repo.save(order);
            await this.emailer.send(order.customerEmail, 'Order confirmed');
          }
        }

        // ✅ Constructor Injection — explicit, testable, swappable
        interface Emailer {
          send(to: string, msg: string): Promise<void>;
        }
        interface OrderRepository {
          save(order: Order): Promise<void>;
        }

        class OrderService {
          constructor(
            private readonly repo: OrderRepository,
            private readonly emailer: Emailer
          ) {}

          async placeOrder(order: Order) {
            await this.repo.save(order);
            await this.emailer.send(order.customerEmail, 'Order confirmed');
          }
        }

        // Test — inject mocks via constructor
        const mockRepo: OrderRepository = { save: jest.fn().mockResolvedValue(undefined) };
        const mockEmailer: Emailer = { send: jest.fn().mockResolvedValue(undefined) };
        const service = new OrderService(mockRepo, mockEmailer);

        // Production — inject real implementations
        const service = new OrderService(new DynamoOrderRepository(dynamoClient), new SmtpEmailer(smtpConfig));

        // DI Container (InversifyJS) — automatic wiring
        @injectable()
        class OrderService {
          constructor(
            @inject(TYPES.OrderRepository) private repo: OrderRepository,
            @inject(TYPES.Emailer) private emailer: Emailer
          ) {}
        }
        ```

      - **Technical Terms to Include:** Dependency Injection, Inversion of Control (IoC), DI container, constructor injection, setter injection, property injection, interface, abstraction, InversifyJS, TSyringe, Angular DI, NestJS, `@injectable`, `@inject`, Dependency Inversion Principle, SOLID
      - **Gotcha:** DI containers can introduce **over-engineering** in small codebases. A plain TypeScript service with constructor injection (no container) provides 90% of DI's testability benefit. A DI container adds: decorator requirements, configuration files, registration boilerplate, and runtime reflection overhead. Use containers when: the dependency graph is complex (20+ services with transitive dependencies), lifecycle management is needed (singleton vs transient vs scoped per request). For Lambda functions and small services — plain constructor injection without a container is often the right call.
      - **Follow-Up:** "How does Next.js handle dependency injection without a container?" → Next.js's Server Components use React context and module-level singletons — not a formal DI container. For testability: Abstract data access into repository functions, mock those modules in tests. NestJS (the server-side Node.js framework) does use a formal DI container (inspired by Angular). The practical answer: in most React/Node.js applications, module mocking with `jest.mock()` achieves DI's testability benefit without a container. Containers are for enterprise Java-style architectures that have made their way into TypeScript via NestJS and Angular. → They're testing DI pragmatism vs purity.
      - **Conclusion:** Dependency Injection inverts control of object creation from the class to its caller — making dependencies explicit, abstractions-based, and injectable with mocks for testing; constructor injection without a DI container is the practical starting point, with a DI container justified only when the dependency graph complexity exceeds manual wiring.

    - MVC / MVVM — (Q count: 2)
      - **[L3.ArchitecturalPatterns.MVVMMVC.Q1]**
        **Question:** What is the difference between MVC, MVP, and MVVM — and which pattern does React most closely resemble?
      - **Answer:**
        - **MVC (Model-View-Controller):** Model (data/business logic), View (UI rendering), Controller (user input handling, coordinates Model and View). The Controller receives user input, updates the Model, the Model notifies the View, the View re-renders. Classic pattern for server-side web apps (Rails, Django, ASP.NET MVC). The View can read directly from the Model.
        - **MVP (Model-View-Presenter):** View is completely passive — it delegates all logic to the Presenter. The View has no direct Model access. The Presenter updates the Model and updates the View (via interface). Testable because the Presenter operates against a View interface — mock the View, test the Presenter in isolation. Common in Android (pre-MVVM), WinForms.
        - **MVVM (Model-View-ViewModel):** ViewModel is a UI-representation of the Model — it exposes Observable properties that the View binds to. When ViewModel properties change, the View updates automatically (data binding). The View never directly calls ViewModel methods — it reacts to property changes. Angular, WPF, SwiftUI use this pattern. Vue and React both approximate it.
        - **React's pattern:** React is closest to **MVVM with one-way data binding**. Components are the View; component state/props (often managed by Zustand/Redux) are the ViewModel; data sources (APIs, stores) are the Model. Data flows one direction: Model → ViewModel (store) → View (component). User interactions flow back via events: View → action → store update → re-render. React is explicitly not two-way data binding (unlike Angular's `ngModel`) — it's unidirectional data flow, which is MVVM without the automatic two-way sync.
        - **Vue:** Closer to two-way MVVM — `v-model` creates a two-way binding between View and ViewModel, similar to Angular.
      - **Example:**

        ```
        MVC flow (server-side Rails):
        Browser → Controller#action → Model.query → View.render(data) → HTML response

        MVVM flow (React + Zustand):
        Store (Model+ViewModel) → component re-renders (View)
        User click → action dispatched → store updates → component re-renders

        React one-way data flow:
        [Store state] → [Component props] → [Rendered JSX]
                                 ↑
        [User event] → [dispatch/setState] (flows back up, not down)

        Angular two-way binding (MVVM):
        [(ngModel)]="username"
        View input → ViewModel property (automatic)
        ViewModel property → View input (automatic)
        React equivalent requires explicit: value={username} + onChange={setUsername}
        ```

      - **Technical Terms to Include:** MVC, MVP, MVVM, Controller, Presenter, ViewModel, data binding, one-way data flow, two-way binding, `v-model`, `ngModel`, unidirectional data flow, Redux, Zustand, React vs Angular architecture
      - **Gotcha:** React's component model doesn't force any particular architectural pattern — you can implement MVC, MVVM, or no pattern at all in React. The "React as MVVM" framing is an approximation; React itself is just a view layer. The architectural pattern in a React application is determined by how you organise business logic, data access, and state management — not by React's rendering model itself. An interviewer testing pattern recognition may accept "MVVM-like" with qualifications.
      - **Follow-Up:** "In a React application with complex business logic, where should that logic live according to MVVM?" → ViewModel = custom hooks or store slices. Business logic belongs in the ViewModel layer (hooks/stores), not in components (View). A component calls `useCheckout()` which handles cart validation, payment processing, and order creation — the component only renders state and calls actions. This keeps the View thin, the ViewModel testable (hooks can be tested with `renderHook`), and business logic separated from rendering. → They're testing MVVM application to React architecture.
      - **Conclusion:** MVC, MVP, and MVVM all solve UI separation of concerns at different points on the coupling spectrum — React's unidirectional data flow is closest to MVVM without two-way binding, and the architectural discipline of keeping business logic in the ViewModel layer (hooks/stores) rather than components determines whether a React application follows the pattern's intent.

---

L3 — Actual questions: 10 | Topics covered: 6

---

## L4 — SOLID Principles & Advanced Patterns

---

L4 — Topics identified: SOLID principles applied in TypeScript, Composition over inheritance, Circuit Breaker, Retry with exponential backoff, Bulkhead, Template Method vs Strategy, Pub/Sub vs Event Bus
L4 — Estimated questions: 14 | Topics: 7

---

- L4
  - SOLID
    - SOLID Applied — (Q count: 3)
      - **[L4.SOLID.Applied.Q1]**
        **Question:** Walk through each SOLID principle with a concrete violation and its fix — then identify which principles React and Node.js naturally support or violate.
      - **Answer:**
        - **S — Single Responsibility:** A class/function has one reason to change. Violation: a `UserController` that handles HTTP parsing, business validation, database calls, and email sending. Fix: separate `UserController` (HTTP), `UserService` (business logic), `UserRepository` (data), `EmailService` (notifications). React alignment: React components that do fetching + rendering + form validation + auth checking violate SRP — extract to custom hooks and service modules.
        - **O — Open/Closed:** Open for extension, closed for modification. Violation: a payment `switch` statement that must be edited for every new payment method. Fix: Strategy pattern — `PaymentHandler` interface, concrete implementations, registry. New payment method = new class, zero modification to existing code.
        - **L — Liskov Substitution:** Subtypes must be substitutable for their base types. Violation: `Square extends Rectangle` where `setWidth()` also sets height (because squares have equal sides) — breaks code that assumes `rectangle.width !== rectangle.height` after `setWidth`. Fix: don't model `Square extends Rectangle` — use composition or separate types. In TypeScript: an interface implementation that throws `NotImplementedException` on half the methods violates LSP.
        - **I — Interface Segregation:** Clients should not depend on interfaces they don't use. Violation: one fat `IAnimal` interface with `fly()`, `swim()`, `run()` — implementing `IAnimal` on a `Dog` forces a `fly()` that throws. Fix: `ICanFly`, `ICanSwim`, `ICanRun` — `Dog implements ICanSwim, ICanRun`. In TypeScript: a single `IStorage` interface with `readFile`, `writeFile`, `deleteFile`, `listBuckets`, `listObjects` — a component that only reads should depend on `IReadableStorage` only.
        - **D — Dependency Inversion:** High-level modules should not depend on low-level modules. Both should depend on abstractions. Violation: `OrderService` directly imports `DynamoDBDocumentClient`. Fix: `OrderService` depends on `IOrderRepository` interface; `DynamoOrderRepository` implements it. High-level: `OrderService`. Low-level: `DynamoOrderRepository`. Both depend on the `IOrderRepository` abstraction.
      - **Example:**

        ```typescript
        // SRP violation — UserController does too much
        class UserController {
          async createUser(req: Request, res: Response) {
            const { email, password } = req.body; // HTTP concern
            if (!email.includes('@')) return res.status(400).send('Invalid email'); // validation
            const hashed = await bcrypt.hash(password, 10); // security logic
            await db.query(`INSERT INTO users ...`); // data access
            await mailer.send(email, 'Welcome!'); // notification
            res.status(201).json({ email });
          }
        }

        // SRP fixed — each class has one reason to change
        class UserController {
          constructor(private userService: UserService) {}
          async createUser(req: Request, res: Response) {
            const result = await this.userService.create(req.body);
            res.status(201).json(result);
          }
        }
        class UserService {
          constructor(
            private userRepo: UserRepository,
            private emailer: Emailer
          ) {}
          async create(dto: CreateUserDto): Promise<User> {
            this.validateEmail(dto.email);
            const hashed = await this.hashPassword(dto.password);
            const user = await this.userRepo.save({ ...dto, password: hashed });
            await this.emailer.send(user.email, 'Welcome!');
            return user;
          }
        }
        ```

      - **Technical Terms to Include:** SOLID, Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion, cohesion, coupling, interface, abstraction, violation, `extends`, `implements`, TypeScript interface, fat interface
      - **Gotcha:** LSP violations are the trickiest to spot in code review. A common one in TypeScript: an interface with 10 methods where the mock implementation in tests throws `Error('not implemented')` on 8 of them. The test only covers 2 methods, so it passes — but the interface is fat (violates ISP) and the mock violates LSP (the mock is not a valid substitute for the real implementation). The tell: if you need to stub methods you don't call, your interface is too broad.
      - **Follow-Up:** "Can React hooks violate SOLID?" → Yes. A `useUserProfile` hook that fetches user data, manages form state, validates inputs, tracks analytics events, and controls a modal dialog violates SRP — it has 5 reasons to change. Split into: `useUserData()` (fetching), `useUserForm()` (form state + validation), `useAnalytics()` (tracking), `useModal()` (dialog control). Each hook has one responsibility. OCP: hooks extended by composition — `useAuthenticatedUserData = () => { const auth = useAuth(); const data = useUserData(auth.userId); ... }`. → They're testing SOLID applied to functional patterns.
      - **Conclusion:** SOLID principles are design heuristics, not laws — they guide toward code that is easier to change, test, and extend; in React and Node.js, SRP manifests as hook decomposition and service separation, OCP as Strategy/Registry, and DIP as constructor injection and repository interfaces.

  - CloudNativePatterns
    - Circuit Breaker & Retry — (Q count: 2)
      - **[L4.CloudNativePatterns.CircuitBreaker.Q1]**
        **Question:** What is the Circuit Breaker pattern, how does it differ from retry with backoff, and when does each apply?
      - **Answer:**
        - **Retry with exponential backoff:** When a transient failure occurs (network blip, 503 from a temporarily overloaded service), retry the operation after a short delay. Each retry doubles the delay (exponential). Add jitter (random offset) to prevent thundering herd — all retriers hitting at the same moment. Stop retrying after a maximum attempt count.
        - **Problem with retry alone:** If the downstream service is catastrophically unavailable (not temporarily overloaded), retries amplify the problem. Every service retrying 3× with backoff means 3× the load on an already-failing downstream — a Distributed Denial of Service by your own services.
        - **Circuit Breaker:** A stateful wrapper around calls to a downstream service. Three states:
          (1) **Closed** (normal) — calls pass through; failure count is tracked.
          (2) **Open** (failure threshold exceeded) — calls are immediately rejected without hitting the downstream. Returns a cached result or error response.
          (3) **Half-Open** (after a timeout) — a probe call is allowed through; if it succeeds, the circuit closes; if it fails, it stays open.
        - **Combined pattern:** Retry handles transient failures (flaky network, momentary overload). Circuit Breaker handles sustained failures (service down, database unreachable). Use both: retry 3× with backoff; if failures exceed the circuit breaker threshold, stop retrying and open the circuit.
        - **Production in AWS:** AWS Lambda retries async invocations 2× by default. But if the downstream RDS is down for 10 minutes, those retries × all Lambda invocations × 3 = massive load on an already-failing DB. Correct architecture: Circuit Breaker (via a Redis-backed state store or a library like `opossum`) in front of DB calls — open the circuit after 5 failures, probe after 30 seconds, close when healthy.
      - **Example:**

        ```typescript
        // Circuit Breaker implementation (simplified)
        type CircuitState = 'CLOSED' | 'OPEN' | 'HALF_OPEN';

        class CircuitBreaker<T> {
          private state: CircuitState = 'CLOSED';
          private failureCount = 0;
          private lastFailureTime = 0;

          constructor(
            private readonly threshold: number,
            private readonly timeout: number // ms before HALF_OPEN probe
          ) {}

          async call(fn: () => Promise<T>): Promise<T> {
            if (this.state === 'OPEN') {
              if (Date.now() - this.lastFailureTime > this.timeout) {
                this.state = 'HALF_OPEN';
              } else {
                throw new Error('Circuit open — service unavailable');
              }
            }

            try {
              const result = await fn();
              this.onSuccess();
              return result;
            } catch (err) {
              this.onFailure();
              throw err;
            }
          }

          private onSuccess() {
            this.failureCount = 0;
            this.state = 'CLOSED';
          }

          private onFailure() {
            this.failureCount++;
            this.lastFailureTime = Date.now();
            if (this.failureCount >= this.threshold || this.state === 'HALF_OPEN') {
              this.state = 'OPEN';
            }
          }
        }

        // Usage with retry + circuit breaker
        const dbBreaker = new CircuitBreaker(5, 30_000);

        async function getUserFromDB(id: string): Promise<User> {
          return dbBreaker.call(() => userRepo.findById(id));
        }
        ```

      - **Technical Terms to Include:** Circuit Breaker, CLOSED/OPEN/HALF_OPEN states, failure threshold, probe, exponential backoff, jitter, retry, thundering herd, `opossum` (Node.js CB library), cascading failure, bulkhead, Hystrix, AWS Lambda retry
      - **Gotcha:** Circuit Breaker state must be **shared across service instances** in a distributed system. A Circuit Breaker held in Node.js process memory only knows about failures in that process. If 10 Lambda instances each have their own Circuit Breaker, the circuit never opens — each instance sees only 1/10 of the failures. For Lambda: store Circuit Breaker state in a shared Redis instance (ElastiCache) or DynamoDB. This is the most common Circuit Breaker implementation mistake in serverless architectures.
      - **Follow-Up:** "How do you implement fallback behaviour when the Circuit Breaker is open?" → The Circuit Breaker should accept a `fallback` function that returns a degraded response when the circuit is open. Example: product catalogue service is down → return the last cached catalogue from Redis rather than an error. User profile service is down → return a default profile rather than blocking the page render. Graceful degradation maintains user experience during partial failures. The fallback is the second half of the Circuit Breaker pattern that's often omitted. → They're testing full Circuit Breaker design including graceful degradation.
      - **Conclusion:** Retry handles transient failures; Circuit Breaker handles sustained failures by short-circuiting calls to a failing service before they amplify the problem — the combination of retry-with-backoff inside a Circuit Breaker is the production-standard resilience pattern for any network call in a distributed system.

---

L4 — Actual questions: 8 | Topics covered: 5

---

## L5 — Distributed & Cloud-Native Patterns

---

L5 — Topics identified: Strangler Fig migration pattern, Sidecar / Ambassador (Kubernetes patterns), Anti-Corruption Layer (DDD), Outbox pattern (depth in EDA file), Event-Driven patterns at scale, Micro-frontend shell patterns
L5 — Estimated questions: 10 | Topics: 6

---

- L5
  - DistributedPatterns
    - Strangler Fig — (Q count: 2)
      - **[L5.DistributedPatterns.StranglerFig.Q1]**
        **Question:** What is the Strangler Fig pattern, when is it the correct migration strategy, and what are the practical implementation steps for migrating a monolith to microservices?
      - **Answer:**
        - **Strangler Fig** (Martin Fowler, named after the fig tree that grows around a host tree and eventually replaces it): Incrementally replace a legacy system by routing new or modified functionality through a new system, while the legacy system handles remaining functionality. Over time, the new system handles all functionality and the legacy is strangled.
        - **When to use:** When a big-bang rewrite is too risky (the codebase is large, business logic is complex, the team doesn't fully understand all edge cases). When the business cannot stop shipping features during the rewrite. When you want to validate the new architecture before fully committing. This is almost always the correct migration strategy — "strangle don't rewrite" is the practitioner's rule.
        - **Implementation steps:**
          1.  **Set up the facade/proxy layer:** An API Gateway, reverse proxy (nginx, Cloudflare), or a BFF sits in front of both old and new systems. All traffic enters through this layer.
          2.  **Identify the first seam:** Choose one bounded context or capability that can be extracted cleanly. Pick something with high business value or high technical debt — not the most complex.
          3.  **Build the new service:** Implement the extracted capability in the new architecture. Deploy it alongside the legacy.
          4.  **Route traffic:** The proxy layer routes requests for the extracted capability to the new service, all other requests to the legacy.
          5.  **Remove from legacy:** Once the new service is stable, delete the corresponding code from the monolith.
          6.  **Repeat:** Move to the next capability. The legacy system shrinks with each iteration.
        - **Data migration challenge:** The hardest part is data — the legacy and new services often share a database initially. The goal is to give each new service its own database over time. Patterns: Shared Database (temporary, start here) → Database per Service (target). During transition, use change data capture (CDC/Debezium) or synchronisation scripts to keep data consistent.
      - **Example:**

        ```
        Strangler Fig migration — monolith to microservices

        Phase 0 — Initial state:
        All traffic → Legacy Monolith (PHP/Rails/JSP)
                      └── Single database

        Phase 1 — Add facade:
        All traffic → API Gateway / nginx (routes by path)
                      ├── /orders/** → Legacy Monolith
                      ├── /products/** → Legacy Monolith
                      └── /auth/** → Legacy Monolith

        Phase 2 — Extract first service (Auth):
        API Gateway:
          ├── /auth/** → New Auth Service (Node.js + JWT)
          ├── /orders/** → Legacy Monolith
          └── /products/** → Legacy Monolith

        Legacy Monolith: Auth code still present but no longer receives traffic
        → Delete Auth code from monolith once new service is stable

        Phase 3 — Extract second service (Notifications):
        API Gateway:
          ├── /auth/** → Auth Service
          ├── /notifications/** → Notification Service (Node.js + SQS)
          ├── /orders/** → Legacy Monolith
          └── /products/** → Legacy Monolith

        ... (repeat until monolith is replaced)

        Phase N — Monolith eliminated:
        All routes → New microservices
        Legacy Monolith → decommissioned
        ```

      - **Technical Terms to Include:** Strangler Fig, big-bang rewrite, incremental migration, bounded context, proxy layer, API Gateway, seam, CDC (Change Data Capture), Debezium, Shared Database → Database per Service, parallel run, traffic routing, Martin Fowler, legacy system
      - **Gotcha:** The Strangler Fig's biggest risk is the **parallel run phase going on too long**. If both the legacy and new service are both handling requests for the same capability, you have two sources of truth — and divergence is guaranteed. Set an explicit date for each extraction phase to complete. "We'll clean it up later" applied to Strangler Fig leaves teams with 40% legacy + 60% new services indefinitely — worse than either the monolith or a full migration.
      - **Follow-Up:** "How do you handle shared database tables during a Strangler Fig migration?" → Start with the Shared Database pattern — both services read/write the same tables. As the new service stabilises, progressively own the data:
        (1) New service reads from shared tables, writes to both.
        (2) Legacy service reads from new service's API instead of the shared tables.
        (3) Legacy service stops writing to the shared tables.
        (4) Move data to the new service's dedicated database.
        (5) Legacy service reads from the new service's API.
        (6) Remove shared tables. This is the "expand then contract" pattern for data migration. → They're testing data migration strategy in the Strangler Fig context.
      - **Conclusion:** Strangler Fig is the production-proven alternative to big-bang rewrites — incremental extraction through a routing facade reduces migration risk to the size of each extracted capability, allows the business to keep shipping, and validates the new architecture before full commitment.

    - Sidecar & Ambassador — (Q count: 2)
      - **[L5.DistributedPatterns.SidecarAmbassador.Q1]**
        **Question:** What are the Sidecar and Ambassador patterns in container/Kubernetes architectures, and how do they solve cross-cutting concerns without modifying service code?
      - **Answer:**
        - **Sidecar:** A helper container deployed alongside a primary service container in the same Kubernetes Pod, sharing the same network namespace and storage volumes. The sidecar handles cross-cutting concerns — logging, monitoring, service mesh proxying (Envoy), certificate rotation, secret synchronisation — without the primary service being aware of it.
        - **Istio and sidecars:** Istio's Envoy proxy sidecar intercepts all inbound and outbound traffic from the service pod — applying mTLS, circuit breaking, retry logic, rate limiting, and distributed tracing without a single line of code in the service itself. The service code is not modified; the sidecar handles all mesh features.
        - **Vault Agent as Sidecar:** An init container (pre-start sidecar) that authenticates to HashiCorp Vault, retrieves secrets, writes them to a shared volume, and exits before the primary container starts. The primary service reads secrets from files — it never talks to Vault directly. Adding Vault secret management to a service = add the sidecar in Kubernetes config, no code change in the service.
        - **Ambassador:** A specialised sidecar that acts as a network proxy for a specific external service — providing service discovery, load balancing, circuit breaking for that one downstream dependency. Where a sidecar is general-purpose, an Ambassador is purpose-built for one outbound dependency. An Ambassador for a legacy service that only speaks HTTP/1.0 can translate modern HTTP/2 requests from services inside the cluster.
        - **Why these patterns matter for your stack:** The Splunk logging wrapper (Decorator pattern in Lambda) is the Lambda-native equivalent of a sidecar — cross-cutting logging without modifying business logic. In Kubernetes, a Sidecar achieves the same thing without even the Decorator code — it's infrastructure-level Decorator.
      - **Example:**

        ```yaml
        # Kubernetes Pod with Sidecar — Vault Agent for secret management
        apiVersion: v1
        kind: Pod
        metadata:
          name: order-service
          annotations:
            vault.hashicorp.com/agent-inject: 'true' # Vault Sidecar Agent
            vault.hashicorp.com/agent-inject-secret-db: 'database/creds/order-service'
            vault.hashicorp.com/role: 'order-service'
        spec:
          containers:
            - name: order-service # primary service
              image: order-service:v1.2
              env:
                - name: DB_PASSWORD_FILE
                  value: /vault/secrets/db-password # reads from shared volume
              volumeMounts:
                - name: vault-secrets
                  mountPath: /vault/secrets
            - name: vault-agent # sidecar
              image: vault:1.16
              # Authenticates to Vault, writes secrets to /vault/secrets volume
              # Primary service reads secrets as files — never talks to Vault
              volumeMounts:
                - name: vault-secrets
                  mountPath: /vault/secrets

        # Istio sidecar — injected automatically by Kubernetes admission controller
        # No pod spec change needed — Istio injects Envoy sidecar transparently
        # The order-service pod gets mTLS, circuit breaking, tracing for free
        ```

      - **Technical Terms to Include:** Sidecar, Ambassador, Kubernetes Pod, shared network namespace, Envoy proxy, Istio, init container, shared volume, Vault Agent, cross-cutting concern, mTLS, service mesh, admission controller, circuit breaker at infrastructure level
      - **Gotcha:** The Sidecar pattern's promise is "no service code changes" — but it requires **infrastructure code changes** (Kubernetes manifests, Helm charts, admission controller configuration). Teams that confuse "no code changes" with "no effort" are surprised by the operational complexity of managing sidecar configurations across hundreds of services. Istio's control plane itself requires significant operational expertise. Know both the benefit and the operational cost.
      - **Follow-Up:** "When would you use a Sidecar for logging instead of a logging library in the service code?" → Sidecar logging (Fluentd/Fluent Bit as a sidecar reading from stdout) makes sense when: multiple services across multiple languages need the same logging pipeline, you want to change logging destinations (e.g., Splunk → Elasticsearch) without deploying new service code, or compliance requires log immutability (service writes to stdout, sidecar captures — service cannot tamper with captured logs). Library-based logging makes sense when: the team values log formatting flexibility per service, or when the service is Lambda/serverless (no sidecar possible). → They're testing infrastructure vs application-layer trade-off judgment.
      - **Conclusion:** Sidecar and Ambassador patterns apply the Decorator/Proxy concept at the infrastructure layer — injecting cross-cutting concerns (observability, security, secret management, proxying) as co-deployed containers without modifying service code, making service mesh capabilities available to services that were not designed with those concerns in mind.

---

L5 — Actual questions: 8 | Topics covered: 4

---

## L6 — Expert Level

---

L6 — Topics identified: When patterns are wrong (over-engineering signal), Pattern selection under constraints, Domain-Driven Design pattern alignment, Architecture fitness functions, Anti-Corruption Layer deep dive
L6 — Estimated questions: 8 | Topics: 4

---

- L6
  - ExpertDecisions
    - When Patterns Become Anti-Patterns — (Q count: 2)
      - **[L6.ExpertDecisions.PatternOverengineering.Q1]**
        **Question:** How do you identify when a design pattern is being applied inappropriately — creating complexity rather than resolving it?
      - **Answer:**
        - **The signal:** A pattern is wrong when it adds more lines of code than the problem warrants, when junior engineers cannot understand the codebase without a pattern reference, or when the pattern is applied preemptively to a problem that doesn't yet exist.
        - **"You Ain't Gonna Need It" (YAGNI):** The most violated principle in pattern application. Applying Abstract Factory because "we might need to swap implementations someday" — when there is only one implementation and no concrete plan to add another — adds 3 extra files and 2 extra interfaces for a flexibility you will never use. Start simple; add the pattern when the second variation appears.
        - **Specific over-engineering signals:**
          1.  **Factory for a single product:** If `createLogger()` always returns `ConsoleLogger` and there is no other logger type, the Factory adds indirection for no benefit. Use `new ConsoleLogger()`.
          2.  **Strategy for a single strategy:** `SortingContext` with `BubbleSortStrategy` as the only implementation. The Strategy was added "for extensibility." Just write the sort function.
          3.  **Observer for a single subscriber:** A `EventEmitter` set up for one subscriber that never changes. Just call the function directly.
          4.  **Repository for a single query:** A `UserRepository` with one method `findById` that wraps one `db.get()` call. The repository adds a class and interface for one line of logic. Acceptable if the project will grow; premature if it's a one-off query.
          5.  **Singleton for stateless utilities:** `MathUtils.getInstance().add(2, 3)`. Stateless utilities are pure functions — they need no instance, no Singleton. `add(2, 3)` directly, exported from a module.
        - **The complexity test:** If a code reviewer asks "why is it a Factory here?" and the answer is "for extensibility," ask "what concrete extensibility requirement exists today?" If there is no answer, it's premature abstraction. If the answer is "we have 3 payment providers today and expect 5 more," the Factory is justified.
      - **Example:**

        ```typescript
        // Over-engineered — Strategy for a single algorithm
        interface SortStrategy<T> {
          sort(arr: T[]): T[];
        }
        class AscendingSort<T> implements SortStrategy<T> {
          sort(arr: T[]) {
            return [...arr].sort();
          }
        }
        class Sorter<T> {
          constructor(private strategy: SortStrategy<T>) {}
          sort(arr: T[]) {
            return this.strategy.sort(arr);
          }
        }
        const sorter = new Sorter(new AscendingSort());
        sorter.sort([3, 1, 2]);

        // Appropriate — there's one sort, just write it
        const sorted = [...arr].sort();

        // Justified Strategy — there ARE multiple strategies
        type SortKey = 'price' | 'rating' | 'distance' | 'relevance';
        const sortStrategies: Record<SortKey, (items: Product[]) => Product[]> = {
          price: (items) => [...items].sort((a, b) => a.price - b.price),
          rating: (items) => [...items].sort((a, b) => b.rating - a.rating),
          distance: (items) => [...items].sort((a, b) => a.distance - b.distance),
          relevance: (items) => [...items].sort((a, b) => b.score - a.score)
        };
        const display = sortStrategies[userSortPreference](products);
        // 4 real strategies — Strategy pattern justified
        ```

      - **Technical Terms to Include:** YAGNI, premature abstraction, over-engineering, single responsibility justification, complexity cost, accidental complexity, essential complexity, "the rule of three", pattern justification, indirection cost
      - **Gotcha:** The "rule of three" — abstract when you see the third occurrence of a pattern, not the first or second. One `switch` on payment type is fine. Two related switches might be coincidence. Three switches on the same type across the codebase is the signal to introduce Strategy. Premature abstraction based on one or two occurrences adds ceremony; waiting for three occurrences grounds the pattern in demonstrated need.
      - **Follow-Up:** "A senior engineer on your team says 'we should apply Repository pattern to all our DynamoDB calls' before any feature is built. How do you evaluate that?" → Valid assessment: will this service have 3+ data access patterns? Will there be a need to swap data stores? Are there performance-critical paths where the Repository's additional method call stack adds measurable overhead? If the service has 2 simple queries and will never change its storage: Repository adds ceremony. If the service is a core domain service with complex queries and testing requirements: Repository pays for itself. The answer is contextual — neither reflexively apply nor reflexively reject. → They're testing pragmatic pattern evaluation against concrete project constraints.
      - **Conclusion:** Design patterns are tools, not mandates — a pattern is wrong when its indirection cost exceeds its abstraction benefit, and the signal is always YAGNI: apply patterns when the second variation exists in reality, not when it exists in imagination.

    - Anti-Corruption Layer — (Q count: 2)
      - **[L6.ExpertDecisions.AntiCorruptionLayer.Q1]**
        **Question:** What is the Anti-Corruption Layer pattern, when is it essential, and how does it apply to frontend architecture?
      - **Answer:**
        - **Anti-Corruption Layer (ACL):** A translation layer between two domains with different models, preventing the "corruption" of a domain's model by an external system's concepts. When System A must integrate with System B, the ACL translates B's model into A's domain language — A never knows about B's internal structure.
        - **DDD origin:** In Domain-Driven Design, bounded contexts have their own ubiquitous language. When two bounded contexts must interact, a naive integration has each context using the other's terminology — over time, the boundary blurs and both contexts are "corrupted" with each other's concepts. The ACL prevents this.
        - **When it's essential:**
          1.  Integrating with a legacy system whose data model is messy or domain-irrelevant — your clean new service should not model the world like a 1995 mainframe just because you're reading from one.
          2.  Integrating with a third-party API (Salesforce, SAP, Stripe) whose model doesn't match your domain — `stripe.PaymentIntent` should not appear in your `Order` domain model.
          3.  Migrating from one system to another — the ACL provides the mapping layer during transition; it's the translation contract between old and new.
        - **Frontend application:** A BFF (Backend-for-Frontend) is an ACL for the frontend — it translates the backend services' models (often database-centric, backend-optimised) into the frontend's domain model (UI-optimised, view-centric). When the PayPal BFF translated the payment service's `PaymentTransaction` object into the checkout UI's `CheckoutDisplayModel`, it was acting as an ACL. The frontend never knew about `PaymentTransaction`.
        - **API client layer as ACL:** In frontend code, a `userApiClient` that maps a REST API response to a frontend `User` type is an ACL. `const user = mapResponseToUser(apiResponse)` — the `mapResponseToUser` function is the ACL. If the API response structure changes, only the mapping function changes, not the 20 components that use `User`.
      - **Example:**

        ```typescript
        // External payment API — complex, backend-centric model
        interface StripePaymentIntent {
          id: string
          amount: number          // in cents
          currency: string        // ISO 4217
          status: 'requires_payment_method' | 'requires_confirmation' | 'succeeded' | ...
          payment_method_types: string[]
          metadata: Record<string, string>
          created: number        // Unix timestamp
          livemode: boolean
        }

        // Your domain model — clean, UI-centric
        interface Payment {
          id: string
          amount: Money          // { value: number; currency: Currency; display: string }
          status: PaymentStatus  // your enum: PENDING | PROCESSING | COMPLETED | FAILED
          method: PaymentMethod  // your type
          createdAt: Date
        }

        // Anti-Corruption Layer — isolates Stripe's model from your domain
        function fromStripePaymentIntent(intent: StripePaymentIntent): Payment {
          return {
            id: intent.id,
            amount: {
              value: intent.amount / 100,       // cents → dollars (translation)
              currency: intent.currency.toUpperCase() as Currency,
              display: formatCurrency(intent.amount / 100, intent.currency),
            },
            status: mapStripeStatus(intent.status),  // Stripe enum → your enum
            method: mapStripePaymentMethod(intent.payment_method_types[0]),
            createdAt: new Date(intent.created * 1000), // Unix → Date
          }
        }

        function mapStripeStatus(stripeStatus: string): PaymentStatus {
          const map: Record<string, PaymentStatus> = {
            'succeeded':                    'COMPLETED',
            'requires_payment_method':      'PENDING',
            'requires_confirmation':        'PROCESSING',
            'canceled':                     'FAILED',
          }
          return map[stripeStatus] ?? 'PENDING'
        }

        // Your service — knows only your domain, never touches Stripe types
        class PaymentService {
          async getPayment(id: string): Promise<Payment> {
            const stripeIntent = await this.stripeClient.paymentIntents.retrieve(id)
            return fromStripePaymentIntent(stripeIntent)  // ACL applied here
          }
        }
        ```

      - **Technical Terms to Include:** Anti-Corruption Layer, bounded context, ubiquitous language, domain model, translation layer, DDD, Stripe → domain mapping, `fromXxx`, model corruption, BFF, facade, legacy integration, model translation
      - **Gotcha:** The ACL must be a **strict boundary** — if a Stripe type leaks into a service method signature or a component prop type, the ACL is broken. The test: grep for Stripe's SDK types outside the ACL module — any occurrence is a corruption. In a frontend codebase, if `StripePaymentIntent` appears in a React component's props or state, the ACL has failed. The discipline is to enforce that only your domain types appear outside the ACL module.
      - **Follow-Up:** "How do you test the Anti-Corruption Layer?" → The ACL mapping function is a pure function — it takes an external model and returns an internal model. Test it exhaustively with all known variants of the external model: successful payment, failed payment, pending payment, each `status` value, null/undefined optional fields, edge case amounts. The ACL is the highest-risk code in an integration — it's where misunderstandings about the external API become production bugs. High unit test coverage of mapping functions is non-negotiable. → They're testing ACL testing strategy.
      - **Conclusion:** The Anti-Corruption Layer prevents external system complexity from infecting your domain model — by translating between representations at the boundary, it keeps your domain clean, makes external API changes a single-point fix, and is the hidden foundation of every well-designed BFF and API client layer.

---

L6 — Actual questions: 8 | Topics covered: 4

---

## Quick Reference — Pattern Decision Logic

| Problem                                                | Pattern                  | Signal to use                                   |
| ------------------------------------------------------ | ------------------------ | ----------------------------------------------- |
| One concrete class, multiple consumers need the same   | Singleton (or ES module) | Shared stateless resource — BUT prefer DI       |
| Creating one of N possible object types                | Factory Method           | `switch/if` on type with `new ConcreteX()`      |
| Creating families of related objects                   | Abstract Factory         | Multiple factories needed, must stay consistent |
| Complex object with many optional params               | Builder                  | Constructor telescoping, validation at build    |
| Adding behaviour without subclassing                   | Decorator                | Cross-cutting concerns (logging, auth, caching) |
| Hiding subsystem complexity                            | Facade                   | BFF, API Gateway, orchestration service         |
| Making incompatible interfaces work                    | Adapter                  | Third-party library, legacy system              |
| Notify multiple objects of state change (same process) | Observer                 | DOM events, EventEmitter, React useEffect       |
| Notify across processes/services                       | Pub/Sub                  | Kafka, SQS, MQTT, Redis Pub/Sub                 |
| Swap algorithms at runtime                             | Strategy                 | Payment methods, discount rules, sort orders    |
| Encapsulate requests for undo/queue/log                | Command                  | Undo/redo, task queue, event sourcing           |
| Pass request through handler chain                     | Chain of Responsibility  | Middleware, validation pipeline, API Gateway    |
| Translate between domain models                        | Anti-Corruption Layer    | Legacy integration, third-party API             |
| Incremental legacy system replacement                  | Strangler Fig            | Monolith → microservices migration              |
| Cross-cutting concerns without code change             | Sidecar                  | Kubernetes, service mesh, Vault Agent           |
| Sustained failure protection                           | Circuit Breaker          | Downstream service reliability                  |
| Transient failure handling                             | Retry + backoff          | Network blips, rate limiting                    |
| Data access abstraction                                | Repository               | Testability, storage swap, DDD                  |
| Decoupled instantiation                                | DI / IoC                 | All services — always prefer over Singleton     |

---

_— End of DesignPatterns_Interview_QA.md —_
_Coverage: L1–L6 | GoF 23 + Enterprise + Distributed + Cloud-Native patterns_
_Total QA units: 36 | Calibrated for: Tech Lead / Architect level_
_Production anchors: Observer (MQTT/Volansys), Facade (BFF/PayPal, MFE/Apple), Strategy (billing routing), Repository (DynamoDB/Volansys), Decorator (Splunk wrappers)_
