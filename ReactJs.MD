<!-- Generated: 2026-03-12 | Version: 19.2.4 -->

# ReactJS Interview Preparation — Tech Lead / Architect Level

> React 19.2.4 | Generated: 2026-03-12 | Jyotil Raval — Career OS

---

## 🆕 What's New — Last 3 Versions

### [React 19.2] — October 2025

- **`<Activity />` Component:** Renders hidden content in the background without removing it from the DOM — interviewers will ask how this replaces conditional rendering and what it means for state preservation.
- **`useEffectEvent`:** Stable hook that extracts non-reactive logic from effects — tests whether you understand the reactive dependency problem and how this eliminates the stale-closure footgun.
- **`cacheSignal`:** Allows cache invalidation from Server Actions — signals whether you understand the RSC data-caching model at an architectural level.
- **Partial Pre-rendering (PPR):** Combines static shell with dynamic streaming slots — interviewers will probe whether you can design a page with both static and dynamic segments without a full SSR penalty.
- **eslint-plugin-react-hooks v6:** Stricter Rules of Hooks enforcement — expect questions on why certain hook patterns now throw lint errors they didn't before.
- **Deprecated / Removed:** `ReactDOM.render()` fully removed (deprecated in v18). `legacy mode` entirely gone.

### [React 19.1] — March 2025

- **React Compiler (stable):** Auto-memoization without `React.memo`, `useMemo`, or `useCallback` — the single biggest shift in mental model since hooks. Interviewers will ask: "Do you still need useMemo?"
- **`use()` Hook (stable):** Reads resources (Promises, Context) inside render — replaces the useEffect data-fetching pattern entirely. Interviewers test whether you know when `use()` suspends vs. resolves.
- **Server Actions (stable):** Async functions that run on the server, callable from the client — know the security boundary implications. This is a high-signal topic at architect level.
- **Deprecated / Removed:** `React.createFactory` removed. `defaultProps` on function components removed — use default parameters instead.

### [React 19.0] — December 2024

- **Actions API:** Native async form handling with `useFormStatus`, `useFormState`, `useOptimistic` — eliminates the boilerplate around loading/error/optimistic state that engineers previously solved with external state libraries.
- **Ref as Prop:** `ref` can now be passed as a plain prop — `forwardRef` wrapper is no longer required. Interviewers will ask if you've updated your component library patterns.
- **`<meta>` / `<title>` / `<link>` as first-class:** Manage document head directly from components — know the implications for SSR and deduplication.
- **Resource Preloading APIs:** `preload()`, `preinit()`, `prefetchDNS()` — signals whether you approach performance declaratively.
- **Deprecated / Removed:** `ReactDOM.render()` deprecated→removed cycle completed. `propTypes` checking removed from production builds entirely.

> **Interview signal:** The version a candidate references unprompted reveals whether they are current or coasting on old knowledge. Interviewers at Tech Lead level will probe the latest version explicitly.

---

## L1 — Conceptual Foundations

---

**L1 — Topics identified:**

- What is React
- Virtual DOM & Reconciliation
- Component-Based Architecture
- JSX
- Unidirectional Data Flow
- Declarative vs Imperative
- React Fiber Engine

---

### Fundamentals

#### What is React

- **Question:** What is React, why was it built, and what problem does it solve that vanilla JavaScript cannot?
- **Answer:**
  - React is a declarative UI library by Meta that lets you describe _what_ the UI should look like for a given state, and React figures out the minimal DOM operations needed to make it so.
  - Vanilla JS requires imperative DOM manipulation — you write _how_ to update the DOM. As state complexity grows, coordinating imperative updates becomes error-prone (race conditions, stale references, manual diff logic).
  - React solves this through a component model + virtual DOM diffing: state changes trigger a re-render of the affected component tree, React diffs the new virtual DOM against the previous snapshot, and only the delta hits the real DOM.
  - Its component model also solves the reuse problem: UI is decomposed into self-contained, composable units with their own state and lifecycle rather than global DOM manipulation logic scattered across files.
  - React is a library, not a framework — it handles only the view layer. Routing, state management, data fetching are ecosystem choices (React Router, Redux, React Query), giving teams architectural flexibility.
- **Example:** Before React, a large e-commerce checkout page updating item counts required 40+ `querySelector` + `innerHTML` calls scattered across event handlers. After React, `setItemCount(n)` triggers a targeted re-render — one declarative state change, React handles the rest.
- **Technical Terms to Include:** virtual DOM, reconciliation, declarative, component tree, state-driven rendering, library vs framework
- **Gotcha:** "React uses a real Virtual DOM — it just updates parts of it." Wrong. The virtual DOM is an in-memory JS object tree. React diffs this tree (reconciliation), then applies only the needed real DOM mutations (commit phase). Two distinct phases.
- **Follow-Up:** "Is React faster than vanilla JS?" → No — vanilla JS direct DOM mutations are faster. React trades raw speed for predictability and developer ergonomics at scale. → They're testing whether you understand React's value proposition vs. performance-first contexts.
- **Conclusion:** React's value is in making complex, stateful UIs predictable and maintainable — not in being the fastest DOM manipulator.

---

#### Virtual DOM & Reconciliation

- **Question:** Explain the Virtual DOM, how reconciliation works, and what heuristics React uses to make diffing O(n) instead of O(n³).
- **Answer:**
  - The Virtual DOM (vDOM) is a lightweight JS object tree mirroring the real DOM. On every render, React creates a new vDOM snapshot and diffs it against the previous one — this is reconciliation.
  - Naïve tree diffing is O(n³). React reduces this to O(n) using two heuristics: (1) Elements of different types produce completely different trees — React tears down and rebuilds rather than attempting a deep diff across type boundaries. (2) The `key` prop tells React which list elements are stable across renders — without keys, React assumes positional identity and diffs incorrectly.
  - Reconciliation is split into two phases in Fiber: the render phase (pure, interruptible — builds the work-in-progress tree) and the commit phase (impure, synchronous — flushes to the real DOM).
  - React 19's Compiler adds a pre-compilation step that annotates components with stability hints, reducing the vDOM work React has to do at runtime.
- **Example:** In a large product listing page, a list of 200 product cards re-rendered on filter change. Without `key={productId}`, React diffed by position — a prepend operation caused all 200 DOM nodes to update. Adding stable keys reduced the diff to 1 insertion.
- **Technical Terms to Include:** virtual DOM, reconciliation, diffing algorithm, O(n) heuristics, key prop, Fiber, render phase, commit phase, work-in-progress tree
- **Gotcha:** "The Virtual DOM makes React fast." The vDOM itself adds overhead. React's speed comes from _batching_ and _minimal DOM mutations_ — the vDOM is the mechanism, not the source of speed.
- **Follow-Up:** "When does React bail out of reconciliation entirely?" → When `React.memo`, `shouldComponentUpdate`, or the React Compiler's auto-memoization determine props haven't changed, React skips the render entirely — no new vDOM produced. → They're testing knowledge of the bail-out optimisation path.
- **Conclusion:** React's O(n) reconciler is a pragmatic trade-off — it's not a perfect diff, it's a fast-enough diff with predictable rules that developers can reason about.

---

#### JSX

- **Question:** What is JSX, how does it compile, and what are its constraints that reveal React's rendering model?
- **Answer:**
  - JSX is syntactic sugar over `React.createElement(type, props, ...children)` calls. Babel/SWC transforms JSX at build time — no browser understands JSX natively.
  - In React 17+, the new JSX transform auto-imports `jsx` from `react/jsx-runtime`, eliminating the `import React from 'react'` requirement that confused junior engineers.
  - JSX constraints reveal the rendering model: (1) Single root element — because `createElement` returns one node. (2) `className` not `class` — because JSX maps to JS objects, `class` is a reserved word. (3) `{}` for expressions only — JSX doesn't support statements (no `if`, no `for`) inside `{}`.
  - JSX is not HTML — it compiles to function calls that return plain JS objects. This is why React can render to DOM (ReactDOM), Native (React Native), or Canvas (React Three Fiber) — the renderer is swappable.
- **Example:** `<ProductCard id={product.id} />` compiles to `jsx(ProductCard, { id: product.id })` — a plain function call returning a React element descriptor object `{ type: ProductCard, props: { id: '...' }, key: null }`.
- **Technical Terms to Include:** JSX transform, React.createElement, jsx-runtime, React element descriptor, renderer-agnostic, Babel, SWC
- **Gotcha:** "JSX is HTML in JavaScript." No — it compiles to function calls returning plain objects. HTML attributes like `class`, `for`, `onclick` are illegal in JSX. Attribute names follow camelCase DOM properties.
- **Follow-Up:** "Can you write React without JSX?" → Yes — `React.createElement` directly. It's valid but unreadable at scale. Knowing this proves you understand JSX as syntactic convenience, not a core requirement. → They're testing whether you understand the compilation layer.
- **Conclusion:** JSX is compile-time syntactic sugar that makes React's element creation readable — understanding its compilation is what separates developers who use React from those who understand it.

---

#### Unidirectional Data Flow

- **Question:** What is unidirectional data flow in React, why is it enforced, and what problem does it prevent?
- **Answer:**
  - Data in React flows in one direction: parent → child via props. Children cannot mutate parent state directly — they invoke callbacks passed as props to signal intent upward.
  - This is enforced because bidirectional data binding (Angular 1.x, Knockout) made state mutations difficult to trace — a change in child A could cascade into sibling B via shared model mutations, creating unpredictable render chains.
  - In React's model, you always know where state lives (the source of truth) and where it can be changed (only in the component that owns it, or via its exposed callbacks).
  - The trade-off: prop drilling — passing data through multiple intermediate layers. Mitigated by Context API, state management libraries, or component composition.
- **Example:** In an e-commerce checkout, the billing address form passed an `onAddressChange` callback up to the CartSummary container. The cart container owned the state, the form reported changes — clean audit trail of every state mutation.
- **Technical Terms to Include:** unidirectional data flow, props, callbacks, lifting state up, single source of truth, prop drilling, bidirectional binding
- **Gotcha:** "Context breaks unidirectional flow." No — Context still flows downward from Provider to consumers. It eliminates prop drilling but doesn't make data flow bidirectional. Data still originates from the context owner.
- **Follow-Up:** "How do you handle state that needs to flow between siblings?" → Lift the state to the nearest common ancestor and pass it down as props + callbacks. Or use Context / an external store. → They're testing your understanding of the composition hierarchy and when to reach for shared state.
- **Conclusion:** Unidirectional data flow is React's core predictability guarantee — you always know where state came from and who is allowed to change it.

---

#### React Fiber Engine

- **Question:** What is React Fiber, what architectural problem did it solve over the Stack reconciler, and what capabilities does it unlock?
- **Answer:**
  - React Fiber (introduced React 16, matured React 18-19) is a complete rewrite of React's reconciliation engine. The old Stack reconciler was synchronous and call-stack-based — once it started reconciling, it couldn't pause. This caused jank: long renders blocked the main thread, dropping frames.
  - Fiber represents each React element as a unit of work (a "fiber node") in a linked list structure. The render phase can now pause, resume, abort, and prioritize work — enabling cooperative scheduling with the browser.
  - This unlocks: concurrent rendering, time-slicing, priority-based updates (user input > data fetching), Suspense (pausing render until data is ready), and `useTransition` (marking updates as non-urgent).
  - Each fiber node stores: the component type, props, state, effects, and pointers to parent/child/sibling fibers. React maintains two trees: current (what's on screen) and work-in-progress (what's being built).
- **Example:** In a large filter-heavy dashboard, filtering 500 list items simultaneously while a search input was being typed caused dropped keystrokes before React 18 Fiber scheduling. After adopting `useTransition`, input remained responsive — the filter computation was deprioritized.
- **Technical Terms to Include:** Fiber, reconciler, cooperative scheduling, concurrent rendering, time-slicing, current tree, work-in-progress tree, render phase, commit phase, priority lanes
- **Gotcha:** "Fiber means React is asynchronous." The commit phase (DOM mutations) is always synchronous. Only the render phase is interruptible. Confusing these leads to wrong assumptions about DOM update timing.
- **Follow-Up:** "What are priority lanes in Fiber?" → React assigns different priority levels to updates: SyncLane (click handlers), InputContinuousLane (drag/scroll), DefaultLane (data fetching), IdleLane (offscreen). The scheduler processes higher-priority lanes first, interrupting lower-priority work. → They're testing architect-level understanding of the scheduler.
- **Conclusion:** Fiber transformed React from a synchronous blocking reconciler into a concurrent, prioritized work scheduler — the foundation for every modern React performance feature.

---

## L2 — Core APIs and Syntax

---

**L2 — Topics identified:**

- Functional Components
- Props & PropTypes
- useState
- useEffect
- useRef
- useContext
- Event Handling
- Conditional Rendering
- Lists & Keys
- Fragment

---

### Core Hooks

#### useState

- **Question:** How does `useState` work under the hood, what are its closure-based pitfalls, and how does React guarantee state identity across renders?
- **Answer:**
  - `useState` stores state in the fiber node of the component instance, indexed by the hook call order (hook slot). This is why hooks cannot be called conditionally — the slot index must be stable across renders.
  - Each `setState` call schedules a re-render. React batches multiple `setState` calls in the same synchronous event handler (and in React 18+, in async handlers via Automatic Batching) into a single re-render.
  - The stale closure problem: a callback that closes over a `count` value captures the value at the time of closure creation. If `count` changes before the callback fires, the captured value is stale. Fix: use the functional updater form `setState(prev => prev + 1)`.
  - State updates are not immediate — the new state value is available only on the _next_ render. Reading state immediately after `setState` gives the old value.
- **Example:**

```jsx
// WRONG — stale closure in async handler
function Counter() {
  const [count, setCount] = useState(0);
  const handleAsync = () => {
    setTimeout(() => setCount(count + 1), 1000); // count is stale
  };
}

// CORRECT — functional updater
const handleAsync = () => {
  setTimeout(() => setCount((prev) => prev + 1), 1000); // always fresh
};
```

- **Technical Terms to Include:** hook slot, call order, stale closure, functional updater, automatic batching, fiber state queue, re-render scheduling
- **Gotcha:** "useState causes one re-render per setState call." In React 18+, multiple setState calls inside a single event handler (or in async code) are automatically batched into one render. You can opt out with `flushSync`.
- **Follow-Up:** "When would you use `useReducer` instead of `useState`?" → When state transitions are complex, when next state depends on multiple parts of current state, or when you need to co-locate state logic with the component. `useReducer` also gives a stable `dispatch` reference — no re-renders caused by passing it as a prop. → They're testing architectural state management judgment.
- **Conclusion:** `useState` is deceptively simple — its closure behaviour and batching model are the source of the most common React bugs in production, making functional updater form a non-negotiable production habit.

---

#### useEffect

- **Question:** Explain the `useEffect` lifecycle, its dependency array semantics, and the most common architectural mistakes engineers make with it.
- **Answer:**
  - `useEffect` runs after the browser has painted — it is not a lifecycle hook, it is a synchronization mechanism. Its job is to synchronize a React component with an external system (DOM, API, subscription, timer).
  - Dependency array controls when the effect re-runs: empty array `[]` — runs once after mount; no array — runs after every render; `[dep]` — runs when `dep` changes (by reference equality).
  - Cleanup function: returned from the effect. Runs before the effect re-runs AND before the component unmounts. Critical for subscriptions, timers, and AbortControllers — not cleaning up causes memory leaks and stale state updates.
  - Common mistakes: (1) Missing dependencies — stale data reads. (2) Object/array literals in deps — reference changes every render, causing infinite loops. (3) Using effects for data fetching in 2024+ React — `use()` hook + Suspense or React Query is the correct pattern now.
- **Example:**

```jsx
// CORRECT — WebSocket sync for a real-time data dashboard
useEffect(() => {
  const ws = new WebSocket(`wss://api.example.com/device/${deviceId}`);
  ws.onmessage = (e) => setTelemetry(JSON.parse(e.data));
  return () => ws.close(); // cleanup prevents zombie connections
}, [deviceId]); // re-connects only when deviceId changes
```

- **Technical Terms to Include:** synchronization mechanism, dependency array, cleanup function, reference equality, StrictMode double-invoke, AbortController, race condition
- **Gotcha:** React 18 StrictMode deliberately double-invokes effects in development to surface cleanup bugs. If your effect breaks on double-invoke, your cleanup is wrong — not React.
- **Follow-Up:** "What's the difference between `useEffect` and `useLayoutEffect`?" → `useLayoutEffect` fires synchronously after DOM mutations but before the browser paints — used for measuring DOM or preventing flicker. `useEffect` fires after paint — non-blocking. Wrong choice causes visual artifacts. → They're testing DOM lifecycle precision.
- **Conclusion:** `useEffect` is a synchronization primitive, not a lifecycle hook — misunderstanding this is the root cause of the most subtle React bugs in production codebases.

---

#### useRef

- **Question:** What are the two distinct use cases for `useRef`, and why is mutating a ref not the same as mutating state?
- **Answer:**
  - Use case 1 — DOM reference: `ref.current` holds a direct reference to a DOM node after mount. Used for focus management, scroll position, animation integration, or measuring layout.
  - Use case 2 — Mutable value container: `useRef` stores a value that persists across renders _without_ causing a re-render when mutated. Unlike state, ref mutation is synchronous and immediate — no scheduling involved.
  - Mutating `ref.current` does not trigger a re-render. This is intentional — use it for values that affect _behaviour_ (timers, previous values, external library handles) but not _rendering_.
  - `useRef` gives a stable object reference across renders — the object itself never changes, only `ref.current` inside it. This makes it safe to include in dependency arrays without causing infinite loops.
- **Example:**

```jsx
// Tracking previous value — common interview pattern
function usePrevious(value) {
  const ref = useRef();
  useEffect(() => {
    ref.current = value;
  });
  return ref.current; // returns previous render's value
}

// Abort controller — cancel stale API requests on id change
const abortRef = useRef(null);
useEffect(() => {
  abortRef.current = new AbortController();
  fetchProducts({ signal: abortRef.current.signal });
  return () => abortRef.current.abort();
}, [productId]);
```

- **Technical Terms to Include:** mutable container, DOM reference, ref.current, stable reference, no re-render on mutation, forwardRef, callback ref
- **Gotcha:** "I can use `useRef` to store derived data to avoid re-renders." If the derived data affects _what the user sees_, it must live in state. Storing render-critical data in a ref means the UI won't update when it changes — a silent bug.
- **Follow-Up:** "What is a callback ref and when do you use it over `useRef`?" → A callback ref is a function passed as the `ref` prop — called with the DOM node when mounted and `null` when unmounted. Used when you need to respond to the ref attaching/detaching (e.g., measuring after dynamic render). `useRef` doesn't fire on attach/detach. → They're testing edge case DOM lifecycle awareness.
- **Conclusion:** `useRef` is React's escape hatch for values that need to persist across renders without driving rendering — mastering the distinction between ref-domain and state-domain prevents an entire class of subtle bugs.

---

### Core Patterns

#### Event Handling

- **Question:** How does React's synthetic event system work, what changed in React 17, and what are the performance implications of inline handlers?
- **Answer:**
  - React wraps native browser events in `SyntheticEvent` — a cross-browser normalized wrapper. This allows React to pool events, control timing, and integrate with the Fiber scheduler's priority system.
  - Pre-React 17: all events were delegated to the `document` level — one listener per event type for the entire app. React 17+ changed delegation to the root DOM container — critical for micro-frontend architectures where multiple React versions share a page.
  - Inline handlers (`onClick={() => handler()}`) create a new function reference on every render. In most cases this is fine — the reconciler cost of comparing functions is negligible. It only matters when passing handlers as props to memoized children (`React.memo`) — the new reference breaks the memo.
  - `e.stopPropagation()` works within React's synthetic system. `e.nativeEvent.stopImmediatePropagation()` is needed to stop events from reaching non-React listeners on the same node.
- **Example:**

```jsx
// WRONG — breaks React.memo on ProductCard (new function ref each render)
<ProductCard onClick={() => handleSelect(product.id)} />;

// CORRECT — stable reference with useCallback
const handleSelect = useCallback((id) => selectProduct(id), []);
<ProductCard onClick={() => handleSelect(product.id)} />;
// Or pass id separately and memoize the callback fully
```

- **Technical Terms to Include:** SyntheticEvent, event delegation, root container delegation, event pooling, React 17 delegation change, stopPropagation, nativeEvent
- **Gotcha:** `e.persist()` was required pre-React 17 to keep the event object after async handling (due to event pooling). React 17 removed event pooling — `e.persist()` is now a no-op. Calling it still works but is dead code.
- **Follow-Up:** "How does React handle events in concurrent mode?" → Event handlers triggered by user input are assigned `SyncLane` priority — they are never interrupted by concurrent rendering. This guarantees input responsiveness even during expensive background renders. → They're testing Fiber scheduler integration knowledge.
- **Conclusion:** React's synthetic event system is not just a compatibility layer — its delegation model, priority assignment, and normalization are architectural decisions that directly affect micro-frontend feasibility and memoization correctness.

---

#### Lists & Keys

- **Question:** Why does React require keys on list elements, what makes a good key, and what failure modes emerge from wrong key choices?
- **Answer:**
  - Keys help React identify which list items changed, were added, or removed between renders. Without keys, React assumes positional identity — index 0 today is the same element as index 0 tomorrow, regardless of content.
  - A good key is: stable (doesn't change across renders), unique among siblings (not globally), and derived from the data's identity (database ID, UUID) — not the render index.
  - Using index as key is wrong when the list can be reordered, filtered, or prepended — React will reuse DOM nodes for wrong items, causing state from old items to persist on new ones (controlled input values are the classic example).
  - Keys also act as component identity signals — changing a key forces React to unmount and remount the component, resetting all its state. This is sometimes intentional (force-reset pattern).
- **Example:**

```jsx
// WRONG — index as key with sortable list
products.map((product, i) => <ProductCard key={i} product={product} />);

// CORRECT — stable identity
products.map((product) => <ProductCard key={product.id} product={product} />);

// INTENTIONAL remount pattern — force reset on ID change
<ProfileForm key={userId} userId={userId} />;
```

- **Technical Terms to Include:** key prop, positional identity, stable key, sibling uniqueness, force-reset pattern, reconciliation hint, DOM reuse
- **Gotcha:** "Keys must be globally unique." No — only unique among siblings. The same key can appear in different list contexts without conflict.
- **Follow-Up:** "Can key be used outside lists?" → Yes — placing a `key` on any component forces a remount when the key changes. Used to reset component state on route changes, user switches, or data identity changes without lifting state. → They're testing creative application of reconciliation primitives.
- **Conclusion:** The key prop is not a React formality — it is the primary mechanism by which you communicate list identity to the reconciler, and misusing it silently corrupts state in production.

---

## L3 — Common Patterns & Real-World Usage

---

**L3 — Topics identified:**

- Custom Hooks & Rules of Hooks
- Context API (deep)
- Controlled vs Uncontrolled Forms
- Error Boundaries
- Code Splitting & React.lazy
- React.memo
- useMemo
- useCallback
- Portals
- forwardRef & useImperativeHandle
- Compound Components
- Higher-Order Components
- Render Props

---

### Hooks Patterns

#### Custom Hooks & Rules of Hooks

- **Question:** What are the Rules of Hooks, why do they exist mechanically, and what makes a well-designed custom hook?
- **Answer:**
  - Two rules: (1) Only call hooks at the top level — not inside conditionals, loops, or nested functions. (2) Only call hooks from React function components or other custom hooks — not from plain JS functions.
  - These rules exist because React tracks hook state by call order (slot index) within a fiber. If a hook is conditionally called, subsequent hooks shift slots, corrupting the state mapping. The rules enforce a stable, predictable slot sequence.
  - A well-designed custom hook: (1) Encapsulates a single concern (not a grab-bag of unrelated logic). (2) Returns a minimal, stable interface. (3) Doesn't couple to component structure — it should be usable anywhere. (4) Has its own cleanup logic if it sets up subscriptions or timers.
  - Naming convention: `use` prefix is mandatory — it signals to the Rules of Hooks linter and to readers that this function participates in the hook system.
- **Example:**

```jsx
// Well-designed custom hook — real-time device telemetry
function useDeviceTelemetry(deviceId) {
  const [telemetry, setTelemetry] = useState(null);
  const [error, setError] = useState(null);

  useEffect(() => {
    const client = new MQTTClient(`device/${deviceId}`);
    client.on('data', setTelemetry);
    client.on('error', setError);
    return () => client.disconnect();
  }, [deviceId]);

  return { telemetry, error };
}
// Usage: const { telemetry, error } = useDeviceTelemetry('device-001');
```

- **Technical Terms to Include:** hook slot, call order, Rules of Hooks, eslint-plugin-react-hooks, single-concern hook, stable interface, use prefix convention
- **Gotcha:** "I can put a hook inside a regular function if it's called inside a component." No — the regular function is not tracked by React's hook system. The hook call must be directly inside the component or another hook. Wrapping breaks the linter and the mental model.
- **Follow-Up:** "How do you test custom hooks?" → `renderHook` from `@testing-library/react`. Wrap state updates in `act()`. Test the interface (return values and side effects), not the internals. → They're testing whether you know the hook testing primitive and the philosophy of testing behaviour, not implementation.
- **Conclusion:** A custom hook is a composable unit of stateful logic — its design quality determines whether your codebase accumulates reusable primitives or duplicated useEffect spaghetti.

---

#### React.memo, useMemo, useCallback

- **Question:** Differentiate `React.memo`, `useMemo`, and `useCallback` — what does each memoize, when is each actually useful, and when do they hurt more than they help?
- **Answer:**
  - `React.memo` memoizes a component — skips re-rendering if props haven't changed (shallow comparison by default). Operates at the component level.
  - `useMemo` memoizes a _value_ — the result of an expensive computation. Re-computes only when listed dependencies change. Operates at the value level.
  - `useCallback` memoizes a _function reference_ — returns the same function object across renders when dependencies don't change. Operates at the reference level. `useCallback(fn, deps)` is equivalent to `useMemo(() => fn, deps)`.
  - When they hurt: premature memoization adds memory overhead (storing previous values), comparison overhead (running the dependency equality check), and cognitive overhead (developers must now reason about whether deps are correct). Memoize only when profiling confirms a problem.
  - React 19 Compiler makes most `useMemo` and `useCallback` calls unnecessary — it auto-memoizes at compile time.
- **Example:**

```jsx
// useMemo — expensive derivation: filtering 500 products by active criteria
const filteredProducts = useMemo(() => products.filter((p) => p.category === activeCategory && p.price <= maxPrice), [products, activeCategory, maxPrice]);

// useCallback — stable handler for memoized child
const handleSelectProduct = useCallback(
  (id) => {
    dispatch({ type: 'SELECT_PRODUCT', payload: id });
  },
  [dispatch]
); // dispatch from useReducer is always stable
```

- **Technical Terms to Include:** memoization, shallow comparison, referential equality, dependency array, React Compiler auto-memoization, premature optimization, profiler
- **Gotcha:** `React.memo` does a shallow prop comparison — if a prop is an object or function created inline, it fails the comparison every render and the memo does nothing. The memo itself costs performance with zero benefit.
- **Follow-Up:** "With React 19 Compiler, should we remove all useMemo and useCallback?" → Not immediately. The compiler handles most cases but has explicit opt-outs and doesn't cover all patterns. Audit with the React DevTools Memo section. Remove gradually, verify with profiler. → They're testing pragmatic adoption judgment.
- **Conclusion:** `React.memo`, `useMemo`, and `useCallback` are optimization tools for specific referential stability problems — applied blindly they add complexity without benefit; applied after profiling they eliminate real bottlenecks.

---

### Architecture Patterns

#### Error Boundaries

- **Question:** What are Error Boundaries, what do they catch and not catch, and how do you architect them in a production React app?
- **Answer:**
  - An Error Boundary is a class component that implements `componentDidCatch` and/or `getDerivedStateFromError`. It catches JavaScript errors in its child component tree during rendering, lifecycle methods, and constructors — and renders a fallback UI instead of crashing the app.
  - What they do NOT catch: errors in event handlers (use try/catch), async code (Promises, setTimeout), server-side rendering, and errors in the error boundary itself.
  - As of React 19, there is no hook-based alternative — Error Boundaries must be class components. The `react-error-boundary` library (`useErrorBoundary`) provides a hook-friendly wrapper.
  - Architectural placement: (1) App-level boundary — last resort, prevents total white screen. (2) Route-level boundaries — isolate page crashes. (3) Widget-level boundaries — isolate high-risk components (ads, third-party embeds, real-time data widgets) so one failure doesn't kill the page.
- **Example:**

```jsx
// Production pattern — widget-level error boundary
class ProductsErrorBoundary extends React.Component {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, info) {
    errorTracker.log({ error: error.message, componentStack: info.componentStack });
  }

  render() {
    if (this.state.hasError) return <ProductsErrorFallback onRetry={() => this.setState({ hasError: false })} />;
    return this.props.children;
  }
}
```

- **Technical Terms to Include:** componentDidCatch, getDerivedStateFromError, error boundary placement, fallback UI, error isolation, react-error-boundary, class component requirement
- **Gotcha:** Error boundaries do not catch errors in event handlers. An onClick that throws will crash silently — you need a try/catch in the handler plus a way to set error state manually. Many engineers assume boundaries cover this.
- **Follow-Up:** "How do you reset an error boundary after a user action (like clicking Retry)?" → With `react-error-boundary`, use the `resetKeys` prop — when any key changes, the boundary resets. Or call the `resetErrorBoundary` function from the fallback. → They're testing UX recovery patterns, not just the boundary itself.
- **Conclusion:** Error Boundaries are the production resilience boundary between a component crash and a total UI failure — their placement architecture defines how much of your app survives a bug.

---

#### Code Splitting & React.lazy

- **Question:** How does React.lazy and Suspense implement code splitting, and how do you architect this for a large-scale application?
- **Answer:**
  - `React.lazy(() => import('./Component'))` creates a lazy component whose bundle chunk is only fetched when the component is first rendered. Bundlers (Webpack, Vite) split the import into a separate chunk at build time.
  - The lazy component must be wrapped in `<Suspense fallback={...}>` — Suspense catches the Promise thrown by the lazy component while loading and renders the fallback.
  - Architectural split points: (1) Route-level — each route is a separate chunk (most impactful — eliminates upfront bundle cost for unvisited pages). (2) Modal/dialog — loaded on first open. (3) Heavy third-party libraries (charting, rich text editors) — isolated to their usage.
  - React 19 Partial Pre-rendering (PPR) extends this: you can statically shell a page with a Suspense boundary and stream the dynamic content — combining code splitting with streaming SSR.
- **Example:**

```jsx
// Route-level splitting — large-scale application pattern
const ProductsPage = React.lazy(() => import('./pages/ProductsPage'));
const CheckoutPage = React.lazy(() => import('./pages/CheckoutPage'));

function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path='/products' element={<ProductsPage />} />
        <Route path='/checkout' element={<CheckoutPage />} />
      </Routes>
    </Suspense>
  );
}
```

- **Technical Terms to Include:** dynamic import, chunk splitting, Suspense boundary, fallback UI, route-level splitting, Webpack magic comments, Partial Pre-rendering, waterfall loading
- **Gotcha:** Nesting `React.lazy` inside another component definition causes it to re-initialize on every parent render, losing the module cache and refetching the chunk. `React.lazy` calls must be at module scope.
- **Follow-Up:** "How do you preload a lazy chunk before the user navigates to it?" → Call the import function directly (not through the lazy wrapper) in an event handler or on hover — `import('./pages/OffersPage')`. This primes the browser cache so the navigation feels instant. → They're testing UX-performance bridging.
- **Conclusion:** Code splitting is the highest-leverage performance optimization for large React apps — route-level lazy loading eliminates upfront bundle cost that no amount of component-level optimization can compensate for.

---

#### Compound Components

- **Question:** What is the Compound Component pattern, when do you use it, and how does it differ from prop-drilling and render props?
- **Answer:**
  - Compound Components model a collection of related components that share implicit state and work together to form a complete UI unit — like `<select>` and `<option>`. The parent manages the shared state; children consume it via Context.
  - Compared to prop drilling: instead of passing `selectedId`, `onSelect`, and `items` three levels deep into a Menu, Compound Components let each sub-component self-register and consume shared state without explicit prop plumbing.
  - Compared to render props: Compound Components are more readable (JSX composition vs callback nesting) but less dynamic (the composition happens at the call site, not inside the render prop callback).
  - Used in: design systems, UI libraries (Tabs, Accordion, Dropdown, Select, Modal) — any component that needs to be compositionally flexible without exposing internal complexity as props.
- **Example:**

```jsx
// Compound Component — design system Tabs
const TabsContext = React.createContext();

function Tabs({ children, defaultTab }) {
  const [active, setActive] = useState(defaultTab);
  return <TabsContext.Provider value={{ active, setActive }}>{children}</TabsContext.Provider>;
}
Tabs.Tab = function Tab({ id, children }) {
  const { active, setActive } = useContext(TabsContext);
  return (
    <button onClick={() => setActive(id)} aria-selected={active === id}>
      {children}
    </button>
  );
};
Tabs.Panel = function Panel({ id, children }) {
  const { active } = useContext(TabsContext);
  return active === id ? <div role='tabpanel'>{children}</div> : null;
};

// Usage:
<Tabs defaultTab='overview'>
  <Tabs.Tab id='overview'>Overview</Tabs.Tab>
  <Tabs.Panel id='overview'>
    <ProductOverview />
  </Tabs.Panel>
</Tabs>;
```

- **Technical Terms to Include:** implicit state sharing, Context as glue, dot notation sub-components, flexible composition, design system pattern, API surface
- **Gotcha:** "Compound Components scale to any depth of nesting." No — the Context-based implementation requires all sub-components to be descendants of the Provider. Deep nesting works; lateral composition across Provider boundaries doesn't.
- **Follow-Up:** "How do you enforce that compound sub-components are only used inside the parent?" → Check `useContext` returns a non-null value — throw a descriptive error if used outside the provider. → They're testing API design defensiveness.
- **Conclusion:** Compound Components are the pattern of choice for design system components that need to be composable at the call site without exposing internal state management as a configuration API.

---

## L4 — Advanced Patterns, Optimization & Edge Cases

---

**L4 — Topics identified:**

- useReducer
- Suspense
- useTransition & useDeferredValue
- React 19 Actions API
- React 19 use() Hook
- React 19 Activity Component
- Server Components (conceptual)
- Advanced Custom Hooks
- Accessibility (a11y)
- React Testing Library patterns

---

### Advanced Hooks

#### useReducer

- **Question:** When should you choose `useReducer` over `useState`, and what architectural advantages does it provide for complex state?
- **Answer:**
  - Choose `useReducer` when: state transitions depend on multiple current state values, you have multiple sub-values that update together, or you want to co-locate transition logic with the component.
  - The reducer pattern (pure function: `(state, action) => newState`) makes every state transition explicit, named, and testable in isolation without a component.
  - `dispatch` from `useReducer` has a stable reference across renders — unlike a setState callback, it never needs to be in a `useCallback` dependency array, preventing unnecessary child re-renders.
  - For global state, combining `useReducer` + `useContext` approximates Redux without the library — useful for app-scale state where Redux is overkill.
- **Example:**

```jsx
// E-commerce cart — state machine with useReducer
const cartReducer = (state, action) => {
  switch (action.type) {
    case 'ADD_ITEM':
      return { ...state, items: [...state.items, action.payload], total: state.total + action.payload.price };
    case 'REMOVE_ITEM':
      return { ...state, items: state.items.filter((i) => i.id !== action.payload), total: recalcTotal(state.items, action.payload) };
    case 'APPLY_COUPON':
      return { ...state, discount: action.payload.discount };
    default:
      return state;
  }
};
const [cart, dispatch] = useReducer(cartReducer, { items: [], total: 0, discount: 0 });
```

- **Technical Terms to Include:** reducer function, dispatch, action, stable dispatch reference, state machine, co-located logic, pure function, testability
- **Gotcha:** `useReducer` doesn't give you time-travel debugging or Redux DevTools unless you wire them up manually. If you need those, use Redux or Zustand with devtools middleware.
- **Follow-Up:** "Can you dispatch async actions in `useReducer`?" → No — reducer must be synchronous and pure. For async, dispatch a 'loading' action, execute the async operation, then dispatch 'success' or 'error'. Or use middleware patterns (thunks) with an external library. → They're testing the constraints of the pure reducer model.
- **Conclusion:** `useReducer` trades `useState`'s simplicity for explicit, testable state transitions — the right choice when state complexity makes individual setState calls harder to reason about than a named action system.

---

#### Suspense

- **Question:** What does Suspense do architecturally, what can it suspend on, and how does it interact with Error Boundaries?
- **Answer:**
  - Suspense catches a thrown Promise during rendering. When a component in its tree throws a Promise, Suspense renders the fallback until the Promise resolves, then re-renders the tree.
  - In React 18+, Suspense works with: `React.lazy` (code splitting), `use()` hook (data fetching, React 19), and any library that implements the Suspense data-fetching contract (React Query, SWR with suspense mode).
  - Suspense and Error Boundaries compose: if the Promise rejects, the error propagates to the nearest Error Boundary — not Suspense. This means you almost always want both: `<ErrorBoundary><Suspense fallback={...}><AsyncComponent /></Suspense></ErrorBoundary>`.
  - In React 19.2, Partial Pre-rendering uses Suspense boundaries as streaming slots — the server streams the static shell immediately, then streams each Suspense boundary's content as it resolves.
- **Example:**

```jsx
// React 19 use() + Suspense — async data with loading boundary
function ProductList({ productsPromise }) {
  const products = use(productsPromise); // suspends until resolved
  return products.map((p) => <ProductCard key={p.id} product={p} />);
}

<ErrorBoundary fallback={<ProductsError />}>
  <Suspense fallback={<ProductsSkeleton />}>
    <ProductList productsPromise={fetchProducts()} />
  </Suspense>
</ErrorBoundary>;
```

- **Technical Terms to Include:** thrown Promise, Suspense boundary, fallback, use() hook, React.lazy, streaming SSR, Partial Pre-rendering, Error Boundary composition
- **Gotcha:** Creating a new Promise on every render (e.g., `<OffersList offersPromise={fetchOffers()} />` in the parent without memoization) causes an infinite Suspense loop — the Promise is always new, always pending. Cache the Promise outside the render cycle.
- **Follow-Up:** "What is Suspense for SSR (streaming)?" → With `renderToPipeableStream`, the server sends the HTML shell immediately, then streams each Suspense boundary's resolved content as it becomes available. Users see the page faster; no total SSR waterfall. → They're testing streaming architecture understanding.
- **Conclusion:** Suspense is the async coordination primitive that makes loading states a first-class concern of the component model rather than imperative flag management scattered across effects.

---

#### useTransition & useDeferredValue

- **Question:** What is the difference between `useTransition` and `useDeferredValue`, and when do you reach for each in a production UI?
- **Answer:**
  - Both are concurrent features that mark work as non-urgent, allowing React to prioritize user input over expensive updates.
  - `useTransition`: wraps a state update in `startTransition`. React marks the update as non-urgent — input remains responsive while the transition (e.g., a re-render of 500 filtered items) completes in the background. Returns `[isPending, startTransition]`.
  - `useDeferredValue`: accepts a value and returns a deferred copy that lags behind the actual value during high-priority updates. Used when you don't control the state update (e.g., a value from props). The component renders twice — once with the immediate value, once with the deferred value.
  - Use `useTransition` when you own the state update. Use `useDeferredValue` when you receive the value from outside.
- **Example:**

```jsx
// useTransition — large list filter (you own the state)
const [isPending, startTransition] = useTransition();
const handleFilter = (category) => {
  startTransition(() => setActiveCategory(category)); // non-urgent
};
// Input stays responsive; isPending shows a loading indicator on the list

// useDeferredValue — receiving search value from a parent component
function SearchResults({ searchQuery }) {
  const deferredQuery = useDeferredValue(searchQuery);
  const results = useMemo(() => filterItems(items, deferredQuery), [deferredQuery]);
  return <ResultList items={results} />;
}
```

- **Technical Terms to Include:** concurrent rendering, priority lanes, startTransition, isPending, deferred value, double render, non-urgent update, Fiber scheduler
- **Gotcha:** `startTransition` does not delay the update — it marks it as interruptible. React may start the transition immediately if the main thread is idle. It's priority-lowering, not debouncing.
- **Follow-Up:** "How does `useTransition` interact with Suspense?" → If a state update wrapped in `startTransition` triggers a Suspense boundary, React shows the _previous_ content (not the fallback) while the new content loads, then swaps — no loading flicker. Without startTransition, the fallback appears. → They're testing the concurrent rendering UX model.
- **Conclusion:** `useTransition` and `useDeferredValue` are the developer interface to React's priority scheduler — using them correctly is the difference between a UI that feels instant and one that feels sluggish under load.

---

### React 19 Features

#### React 19 Actions API

- **Question:** What is the React 19 Actions API, what problem does it replace, and what are `useFormStatus` and `useOptimistic`?
- **Answer:**
  - React 19 Actions are async functions passed to form `action` props or triggered manually. React natively handles the loading, error, and success lifecycle — eliminating the boilerplate that previously required useState + useEffect + manual error handling for every form.
  - `useFormStatus`: reads the pending state of the parent form's action — available in child components without prop drilling. Replaces passing `isLoading` props through the form tree.
  - `useOptimistic`: accepts the current state and an update function. Immediately shows the optimistic state while the server action is pending — rolls back automatically on failure. Replaces the complex optimistic update pattern previously built with useState + rollback logic.
  - `useActionState` (previously `useFormState`): takes an action and initial state, returns the current state + a wrapped action + isPending. The primary hook for managing form action state.
- **Example:**

```jsx
// E-commerce checkout — address update with optimistic UI
function AddressForm({ currentAddress }) {
  const [optimisticAddress, updateOptimistic] = useOptimistic(currentAddress, (state, newAddress) => ({ ...state, ...newAddress }));

  async function updateAddress(formData) {
    const newAddress = Object.fromEntries(formData);
    updateOptimistic(newAddress); // immediate UI update
    await saveAddressToServer(newAddress); // actual mutation
  }

  return (
    <form action={updateAddress}>
      <input name='street' defaultValue={optimisticAddress.street} />
      <SubmitButton />
    </form>
  );
}

function SubmitButton() {
  const { pending } = useFormStatus(); // reads parent form state
  return <button disabled={pending}>{pending ? 'Saving...' : 'Save'}</button>;
}
```

- **Technical Terms to Include:** Server Actions, useFormStatus, useOptimistic, useActionState, optimistic update, rollback, async form action, pending state
- **Gotcha:** `useFormStatus` only works inside a component that is a _child_ of a `<form>` with an `action` prop — not a sibling or ancestor. Many engineers try to use it at the form level itself.
- **Follow-Up:** "How do Actions relate to Server Actions in Next.js?" → React Actions are the client-side primitive. Next.js Server Actions extend this — the async function runs on the server, callable from the client. The form `action` attribute can point to a server function with `'use server'`. → They're testing RSC ecosystem awareness.
- **Conclusion:** React 19 Actions collapse the loading/error/optimistic state pattern that previously required 5+ manual state variables into a single composable primitive — understanding it is a differentiator at Tech Lead level.

---

#### React 19 `use()` Hook

- **Question:** What does the `use()` hook do, how does it differ from `useEffect` for data fetching, and what are its constraints?
- **Answer:**
  - `use(resource)` reads a resource (Promise or Context) inside render. If the Promise is not yet resolved, it throws — Suspense catches this and shows the fallback. When resolved, React re-renders with the resolved value.
  - Compared to `useEffect` for data fetching: `useEffect` fetches _after_ render (client-only waterfall). `use()` integrates with Suspense — the component doesn't render until data is ready, enabling streaming SSR and avoiding client-side loading state boilerplate.
  - Key constraint: `use()` can be called conditionally (unlike other hooks). It is not bound by the Rules of Hooks slot system because it doesn't store state in a slot — it reads from the resource.
  - The Promise must be stable (not created on every render) — typically created outside the component or cached with a mechanism like React's `cache()` function (server-side) or a data library.
- **Example:**

```jsx
// React 19 use() — reading context conditionally
function ThemeToggle({ showLabel }) {
  if (!showLabel) return null;
  const theme = use(ThemeContext); // valid — use() can be conditional
  return <span>{theme.mode}</span>;
}

// use() for data — stable Promise created outside render
const productsPromise = fetchProducts(); // stable — created outside render

function ProductsPage() {
  const products = use(productsPromise); // suspends until resolved
  return <ProductGrid products={products} />;
}
```

- **Technical Terms to Include:** resource, thrown Promise, Suspense integration, conditional hook call, cache(), stable Promise, streaming SSR, Context reading
- **Gotcha:** `use()` cannot be used inside try/catch — the thrown Promise must propagate to Suspense. Wrapping in try/catch prevents Suspense from catching the suspend signal.
- **Follow-Up:** "What replaces the useEffect data fetching pattern entirely in React 19?" → `use()` + Suspense + `cache()` (server) or React Query/SWR with Suspense mode (client). useEffect for data fetching is now an anti-pattern in React 19. → They're testing whether you've internalized the React 19 mental model shift.
- **Conclusion:** `use()` is the hook that makes data fetching a first-class rendering concern rather than a post-render side effect — it is the single biggest mental model change in React 19.

---

#### React 19.2 `<Activity />` Component

- **Question:** What does the `<Activity />` component do, when do you use it, and how does it differ from conditional rendering?
- **Answer:**
  - `<Activity mode="visible" | "hidden">` keeps a component tree mounted but controls its visibility and priority. `hidden` mode: the tree is kept in memory (state preserved) but hidden from the user and deprioritized by React's scheduler.
  - Key difference from conditional rendering (`{isVisible && <Component />}`): conditional rendering unmounts the component (state is lost). `Activity hidden` keeps it mounted (state preserved) — no remount cost when switching back.
  - Use cases: tab panels (preserve scroll position and form state when tabs switch), offscreen pre-rendering (load the next likely view before the user navigates), multi-step wizards (step 2 is kept alive while user is on step 1).
  - Performance implication: hidden trees still consume memory. Don't use `Activity` to hide components that should be unmounted — only for components that will be shown again soon.
- **Example:**

```jsx
// Multi-tab dashboard — preserve tab state without remounting
import { Activity } from 'react';

function Dashboard() {
  const [activeTab, setActiveTab] = useState('overview');

  return (
    <>
      <TabBar onSelect={setActiveTab} active={activeTab} />
      <Activity mode={activeTab === 'overview' ? 'visible' : 'hidden'}>
        <OverviewPanel /> {/* state preserved when hidden */}
      </Activity>
      <Activity mode={activeTab === 'analytics' ? 'visible' : 'hidden'}>
        <AnalyticsPanel /> {/* state preserved when hidden */}
      </Activity>
    </>
  );
}
```

- **Technical Terms to Include:** Activity component, visible/hidden mode, state preservation, offscreen rendering, scheduler priority, memory trade-off, React 19.2
- **Gotcha:** `Activity` in `hidden` mode still runs effects and subscriptions unless you explicitly pause them (the component knows it's hidden via a future `useActivityState` hook — currently, you'd check visibility via Context from the Activity boundary). Unpaused subscriptions in hidden trees waste resources.
- **Follow-Up:** "How is `<Activity />` related to the old experimental `<Offscreen />` API?" → `<Activity />` is the stable rename and evolution of the experimental `<Offscreen />` component from React 18's concurrent mode exploration. Same concept, stable API. → They're testing awareness of React's evolution path.
- **Conclusion:** `<Activity />` solves the fundamental tension between conditional rendering's clean unmounting and the UX cost of remounting expensive components — state preservation with controlled visibility.

---

## L5 — Architecture & System Design

---

**L5 — Topics identified:**

- Micro-Frontend Architecture with React
- Design Systems
- State Management Architecture
- Data Fetching Architecture
- SSR & Streaming
- Component Library Architecture
- Monorepo Strategy

---

### Architecture

#### Micro-Frontend Architecture with React

- **Question:** How do you architect a React-based micro-frontend system using Module Federation, and what are the critical failure modes at scale?
- **Answer:**
  - Module Federation (Webpack 5) allows independently deployed React apps to expose and consume modules at runtime. Each MFE (micro-frontend) is a separate build with its own deployment lifecycle — teams ship independently.
  - Critical architectural decisions: (1) Shared dependencies — React and ReactDOM must be shared at `singleton: true` to avoid multiple React instances on the page (breaks hooks, causes "invalid hook call" errors). (2) Contract definition — exposed modules must have stable TypeScript interfaces; breaking changes are a deployment risk. (3) Shell application — the container that orchestrates MFEs, handles routing, and manages the shared authentication context.
  - React 17's event delegation change to root container (vs document) was specifically designed to support multiple React versions on the same page — enabling MFEs with different React versions without event conflicts.
  - Failure modes: shared dependency version drift, stale remote entry caches (CDN caching of remoteEntry.js), bootstrap initialization race conditions, and cross-MFE routing conflicts.
- **Example:**

```js
// Shell — webpack.config.js
new ModuleFederationPlugin({
  name: 'shell',
  remotes: {
    catalogApp: 'catalogApp@https://cdn.example.com/catalog/remoteEntry.js',
    checkoutApp: 'checkoutApp@https://cdn.example.com/checkout/remoteEntry.js'
  },
  shared: {
    'react': { singleton: true, requiredVersion: '^18.0.0' },
    'react-dom': { singleton: true, requiredVersion: '^18.0.0' }
  }
});

// catalogApp — webpack.config.js
new ModuleFederationPlugin({
  name: 'catalogApp',
  filename: 'remoteEntry.js',
  exposes: { './CatalogPage': './src/pages/CatalogPage' },
  shared: { react: { singleton: true } }
});
```

- **Technical Terms to Include:** Module Federation, remoteEntry.js, singleton shared dependency, shell application, contract testing, dynamic remotes, bootstrap chunk, independent deployment
- **Gotcha:** Not setting `singleton: true` on React in the shared config allows each MFE to load its own React instance. This causes hooks to fail with "rendered more hooks than previous render" because hook state is per-React-instance, not per-DOM-node.
- **Follow-Up:** "How do you handle authentication state across micro-frontends?" → Shared authentication state should not live in any single MFE — use the shell to manage auth, expose it via a shared Context or a custom event system. Alternatively, use a shared auth library that reads from cookies/localStorage and exposes a stable hook interface. → They're testing cross-MFE state architecture.
- **Conclusion:** Micro-frontend architecture solves the team scaling problem but introduces a distributed systems problem at the UI layer — every MFE boundary is a potential contract violation, deployment race, or shared dependency conflict.

---

#### Design Systems

- **Question:** How do you architect a React component library that scales across multiple product teams without becoming a bottleneck?
- **Answer:**
  - A design system at scale has three layers: (1) Tokens layer — design variables (color, spacing, typography) as CSS custom properties or a structured JS object. Consumed by all layers. (2) Primitive components layer — unstyled, accessible building blocks (Button, Input, Modal). Maximum flexibility, minimal opinion. (3) Composite components layer — opinionated, token-styled compositions (SearchBar, DataTable). Higher opinion, lower flexibility.
  - Versioning strategy: semantic versioning is required. Breaking changes in primitives cascade to all consumers — a major version bump must come with a migration guide and a codemoding path.
  - Contribution model: open contribution with design review prevents bottleneck. But requires contract testing — every component has a defined API spec, and PRs that break the spec are rejected automatically.
  - Published with Storybook for visual testing, Chromatic for visual regression, and a peer-reviewed accessibility audit (`jest-axe` in unit tests).
- **Example:**

```tsx
// Token layer — spacing system
const tokens = {
  space: { xs: '4px', sm: '8px', md: '16px', lg: '24px', xl: '32px' }
};

// Primitive — Button (no styling opinion)
interface ButtonProps {
  variant: 'primary' | 'secondary' | 'ghost';
  size: 'sm' | 'md' | 'lg';
  isLoading?: boolean;
  children: React.ReactNode;
  onClick?: () => void;
}
// Composite — SearchBar (app-level composition of primitives)
<SearchBar onSearch={handleSearch} placeholder='Search products...' />;
// internally: Input primitive + IconButton primitive + token spacing
```

- **Technical Terms to Include:** design tokens, primitive components, composite components, Storybook, Chromatic, visual regression, semantic versioning, codemoding, peer dependency, accessibility audit
- **Gotcha:** Building composite components first (skipping primitives) creates a system that can't adapt — every new use case requires a new composite. The investment in primitives pays back in flexibility at every new product surface.
- **Follow-Up:** "How do you handle theming across multiple brands in a single component library?" → CSS custom properties (variables) scoped to a theme class on the root element. Each brand overrides the token values via their theme class. Components reference tokens, not hardcoded values — brand switching is a class swap. → They're testing multi-brand architecture.
- **Conclusion:** A design system is a product, not a project — it needs ownership, versioning, contribution governance, and migration tooling, or it becomes a dependency that teams avoid rather than adopt.

---

#### State Management Architecture

- **Question:** How do you decide between Redux, Zustand, Jotai, and React Context for state management in a large-scale React application?
- **Answer:**
  - State classification drives the decision: (1) Server state (async, remote) → React Query or SWR — not a state manager, a server cache. (2) Global client state (user session, theme, cart) → Zustand or Redux. (3) Component-local shared state (form, wizard steps) → Context + useReducer. (4) Atomic fine-grained state (per-cell table values) → Jotai or Recoil.
  - Redux use cases: large teams needing strict action traceability, time-travel debugging, Redux DevTools integration, or teams already invested in the Redux ecosystem. The boilerplate cost is justified by predictability at scale.
  - Zustand: minimal API, no boilerplate, no Provider required (module-level store), supports React 19 concurrent mode natively. Best for teams that need global state without Redux's ceremony.
  - Context + useReducer: correct for domain-scoped state that doesn't cross many component boundaries. Not suited for high-frequency updates — every context consumer re-renders on any value change.
- **Example:**

```tsx
// Zustand — e-commerce cart store (no Provider, module-level)
const useCartStore = create<CartStore>((set) => ({
  items: [],
  total: 0,
  addItem: (item) =>
    set((state) => ({
      items: [...state.items, item],
      total: state.total + item.price
    })),
  removeItem: (id) =>
    set((state) => ({
      items: state.items.filter((i) => i.id !== id),
      total: recalcTotal(state.items.filter((i) => i.id !== id))
    }))
}));

// Usage — no Provider needed
const { items, addItem } = useCartStore();
```

- **Technical Terms to Include:** server state, client state, React Query, Zustand, Redux Toolkit, Jotai, Context re-render cost, selector pattern, module-level store, state colocation
- **Gotcha:** React Context is not a state management solution — it is a dependency injection mechanism. Using Context for high-frequency state (like a search query that updates on every keystroke) causes every consumer to re-render on every keystroke, regardless of whether they use that specific value.
- **Follow-Up:** "How do you prevent unnecessary re-renders with Context?" → Split context into multiple providers by update frequency. Or use `useMemo` on the value object (but this only prevents new object reference, not re-renders from changed values). Or replace Context with Zustand, which supports selectors — components only re-render when the specific slice they select changes. → They're testing Context performance awareness.
- **Conclusion:** The state management decision is not a technology preference — it's a categorization problem: classify state by origin (server vs client), scope (local vs global), and frequency (static vs high-frequency), then pick the tool that fits each category.

---

#### Data Fetching Architecture

- **Question:** How do you architect data fetching in a large React application, and how has React 19 changed the canonical approach?
- **Answer:**
  - Pre-React 19 canonical pattern: useEffect + useState for fetch + loading + error. Problem: component mounts, renders empty, then fetches — a render-then-fetch waterfall. Also caused race conditions (response from a stale fetch arriving after a newer request).
  - React Query / SWR pattern: declarative data fetching with a cache layer. Components declare what data they need (`useQuery('offers', fetchOffers)`). React Query handles caching, deduplication, background revalidation, optimistic updates, and stale-while-revalidate semantics. No useEffect for data fetching.
  - React 19 pattern: `use()` hook + Suspense + `cache()` for server components, React Query with Suspense mode for client components. The component tree suspends on data needs — data fetching is integrated into rendering, not layered on top.
  - Key architectural rule: data fetching should happen as high in the tree as possible (or in parallel via parallel routes / parallel queries) to avoid request waterfalls.
- **Example:**

```tsx
// React Query — product listings with Suspense mode
function ProductList({ category }) {
  const {
    data: products,
    isError,
    refetch
  } = useQuery({
    queryKey: ['products', category],
    queryFn: () => fetchProducts(category),
    staleTime: 5 * 60 * 1000, // 5 min — products don't change frequently
    suspense: true // integrates with Suspense boundary
  });
  return <ProductGrid products={products} />;
}

// Parallel queries — avoid waterfall
const [productsQuery, userQuery] = useQueries({
  queries: [
    { queryKey: ['products', category], queryFn: fetchProducts },
    { queryKey: ['user'], queryFn: fetchUser }
  ]
});
```

- **Technical Terms to Include:** render-fetch waterfall, React Query, stale-while-revalidate, cache deduplication, optimistic updates, parallel queries, suspense mode, race condition, AbortController
- **Gotcha:** Setting `staleTime: 0` (React Query default) means every component mount triggers a background refetch, even if data was just fetched by a sibling. Set `staleTime` deliberately based on how frequently your data changes.
- **Follow-Up:** "How do you handle cache invalidation across mutations?" → After a mutation succeeds, call `queryClient.invalidateQueries(['offers'])`. This marks the cached data as stale, triggering a background refetch for all active offer queries. Alternatively, update the cache directly with `setQueryData` for optimistic consistency. → They're testing cache coherence strategy.
- **Conclusion:** Data fetching architecture is not a component-level concern — it is a caching, deduplication, and invalidation strategy that determines whether your app makes 1 API call per view or 10.

---

## L6 — Expert Level: Cross-Cutting Concerns, Scale, Security

---

**L6 — Topics identified:**

- React Fiber Internals & Reconciliation Deep Dive
- React 19 Compiler
- Security (XSS, CSP)
- Performance Profiling at Scale
- React Server Components (RSC) Deep Dive
- Concurrent Rendering Internals
- Cross-Cutting Concerns

---

### Expert Level

#### React 19 Compiler

- **Question:** What does the React 19 Compiler do, how does it change the memoization model, and what are its current limitations?
- **Answer:**
  - The React Compiler (previously "React Forget") is a build-time compiler that analyzes component code and automatically inserts memoization where components, values, and callbacks would benefit — without explicit `React.memo`, `useMemo`, or `useCallback` calls.
  - It works by analyzing the dependency graph of values in components and inferring which values are stable across renders. It then emits the equivalent of `useMemo` and `useCallback` wrapping in the compiled output.
  - Impact: eliminates an entire class of performance bugs (forgetting to memoize, incorrect deps), reduces cognitive overhead, and makes component code cleaner (no memoization boilerplate).
  - Current limitations: cannot compile components that violate React's rules (mutation of props, reading state after render). These components must be opted out with `'use no memo'`. Also cannot currently optimize across component boundaries or external library calls.
- **Example:**

```tsx
// Developer writes:
function ProductCard({ product, onSelect }) {
  const price = formatPrice(product.price); // auto-memoized if product.price stable
  return <div onClick={() => onSelect(product.id)}>{price}</div>; // callback auto-stabilized
}

// Compiler emits (conceptually):
function ProductCard({ product, onSelect }) {
  const price = useMemo(() => formatPrice(product.price), [product.price]);
  const handleClick = useCallback(() => onSelect(product.id), [onSelect, product.id]);
  return <div onClick={handleClick}>{price}</div>;
}
```

- **Technical Terms to Include:** React Compiler, auto-memoization, dependency graph analysis, 'use no memo', build-time optimization, babel plugin, eslint-plugin-react-compiler
- **Gotcha:** The Compiler does not eliminate the need to understand memoization — it eliminates the need to _write_ it. When the compiler opts a component out (due to a rule violation), you need to understand why and fix the violation rather than add manual memoization on top.
- **Follow-Up:** "How do you know if a component was successfully compiled?" → React DevTools shows a "✓ Memo" badge on components the Compiler successfully memoized. Components that opted out show no badge. The `eslint-plugin-react-compiler` flags violations at lint time. → They're testing operational knowledge of the compiler toolchain.
- **Conclusion:** The React Compiler shifts memoization from a developer discipline to a build-time guarantee — but only for code that correctly follows React's rules, making rule compliance more important than ever.

---

#### Security — XSS, CSP, and React-Specific Vectors

- **Question:** What are the XSS vectors specific to React, how does React's design mitigate them, and where does it not?
- **Answer:**
  - React automatically escapes string values rendered in JSX — `<div>{userInput}</div>` will escape `<script>` tags. This is React's primary XSS defense and it operates at the JSX expression level.
  - `dangerouslySetInnerHTML` is the explicit escape hatch — it bypasses React's escaping and renders raw HTML. Any user-supplied content through this prop is a direct XSS vector. Always sanitize with DOMPurify before use.
  - URL-based XSS: `<a href={userUrl}>` — if `userUrl` is `javascript:alert(1)`, React will render it. Validate URL schemes before rendering user-supplied URLs (`url.startsWith('https://')` check or CSP's `default-src`).
  - React 19 Server Components introduce a new vector: passing user input directly into database queries or shell commands inside Server Actions — SQL injection and command injection, not just XSS. Input validation and parameterized queries are mandatory.
  - Content Security Policy (CSP): set `script-src 'self'` to block inline script injection even if XSS occurs. React's SSR hydration is compatible with strict CSP if you avoid inline event handlers in server-rendered HTML.
- **Example:**

```tsx
// WRONG — XSS via dangerouslySetInnerHTML
<div dangerouslySetInnerHTML={{ __html: userComment }} />

// CORRECT — sanitize first
import DOMPurify from 'dompurify';
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userComment) }} />

// WRONG — javascript: protocol XSS
<a href={user.profileUrl}>Profile</a>

// CORRECT — validate scheme
const safeUrl = /^https?:\/\//.test(user.profileUrl) ? user.profileUrl : '#';
<a href={safeUrl}>Profile</a>
```

- **Technical Terms to Include:** XSS, dangerouslySetInnerHTML, DOMPurify, javascript: protocol, Content Security Policy, nonce, Server Components injection, parameterized queries, CSRF
- **Gotcha:** React escaping only protects JSX text content. It does not protect HTML attribute values that accept arbitrary strings — `href`, `src`, `style` can all be XSS vectors with user-supplied content. Each attribute type needs its own validation.
- **Follow-Up:** "How does CSP interact with React's inline event delegation?" → React's synthetic event system adds listeners to the root container in JS — not inline HTML attributes — so it's compatible with strict CSP policies that block `unsafe-inline`. The risk is third-party scripts injected via `dangerouslySetInnerHTML` that add inline handlers. → They're testing deep CSP + React interaction knowledge.
- **Conclusion:** React's JSX escaping is a strong default, but the attack surface in a production React app extends to URL schemes, raw HTML injection, CSS injection, and — in React 19 — server-side injection vectors that have nothing to do with the DOM.

---

#### Performance Profiling at Scale

- **Question:** How do you identify and resolve a React performance problem in a production application with 500k+ active users?
- **Answer:**
  - Step 1 — Observe: use React DevTools Profiler to record a user interaction. Identify components with high render time (flame graph). Look for unnecessary renders (components that re-render without prop/state changes).
  - Step 2 — Classify the problem type: (1) Too many renders — a parent re-renders and pulls an entire subtree with it. Fix: `React.memo`, split context, or move state down. (2) Expensive renders — a render itself is slow (complex calculation, large list). Fix: `useMemo`, virtualization (`react-window`), `useTransition`. (3) Large DOM — too many nodes. Fix: virtualize with `react-window` or `react-virtual`.
  - Step 3 — Measure in production: `web-vitals` for LCP, FID, CLS. Custom performance marks with `performance.mark()` and `performance.measure()` around critical paths. Splunk/Datadog for aggregated RUM data.
  - Step 4 — Target the highest-leverage fix: bundle size reduction (code splitting) often has 10x more impact than component-level memoization. Always tackle bundle size, rendering architecture, and network before micro-optimizations.
- **Example:**

```tsx
// Identifying stale renders — React DevTools Profiler
// After recording: ProductCard renders 200 times for a single filter change
// Root cause: parent passes new object reference each render
// Before: <ProductCard config={{ maxItems: 10 }} /> — new object each render
// After: const config = useMemo(() => ({ maxItems: 10 }), []); <ProductCard config={config} />

// Production measurement — custom performance marks
performance.mark('product-filter-start');
startTransition(() => setActiveCategory(category));
performance.mark('product-filter-end');
performance.measure('product-filter', 'product-filter-start', 'product-filter-end');
```

- **Technical Terms to Include:** React DevTools Profiler, flame graph, unnecessary renders, virtualization, react-window, web-vitals, LCP, FID, CLS, bundle analysis, webpack-bundle-analyzer
- **Gotcha:** The React DevTools Profiler only works in development mode — it is disabled in production builds. For production performance data, use browser performance APIs, RUM tools, and `<Profiler>` component with `onRender` callback (available in production).
- **Follow-Up:** "What is the `<Profiler>` component and when do you use it in production?" → `<Profiler id="offers" onRender={callback}>` wraps a subtree and fires `onRender` with timing data on every render — including in production. Used to track specific high-risk components over time and alert on render time regressions. → They're testing production observability knowledge.
- **Conclusion:** React performance work is a measurement discipline first — every optimization without a profiler baseline is a guess, and in a 500k-user app, the wrong guess wastes engineering cycles while the actual bottleneck remains.

---

#### React Server Components (RSC) Deep Dive

- **Question:** What are React Server Components architecturally, how do they differ from SSR, and what constraints do they impose on your component model?
- **Answer:**
  - RSC (React Server Components) render on the server and send a serialized React tree (not HTML) to the client. This is fundamentally different from SSR: SSR renders HTML on the server, which the client then hydrates. RSC sends a React component tree description that React on the client merges into the existing tree without a full page hydration.
  - Server Components: run only on server. Zero JavaScript sent to client. Can access backend resources directly (databases, file system, APIs). Cannot use state, effects, or browser APIs. No interactivity.
  - Client Components: `'use client'` directive at file top. Traditional React — has state, effects, event handlers. Their JS bundle is sent to the client.
  - The boundary: you can nest Client Components inside Server Components, but you cannot nest Server Components inside Client Components (Server Components cannot import Client-bound dependencies). Pass Server Component output as `children` prop to Client Components.
  - RSC payload is streamed — as each Server Component resolves, its serialized output is streamed to the client and merged into the React tree.
- **Example:**

```tsx
// Server Component — no JS to client, direct DB access
// ProductsServerPage.tsx (no 'use client')
async function ProductsServerPage({ category }) {
  const products = await db.query('SELECT * FROM products WHERE category = ?', [category]);
  // products data is embedded in the RSC payload — no client fetch needed
  return (
    <div>
      <PageHeader /> {/* Server Component */}
      <InteractiveFilter defaultCategory={category} products={products} /> {/* Client Component */}
    </div>
  );
}

// Client Component — interactive
('use client');
function InteractiveFilter({ defaultCategory, products }) {
  const [active, setActive] = useState(defaultCategory);
  const filtered = products.filter((p) => p.category === active);
  return <ProductGrid products={filtered} onCategoryChange={setActive} />;
}
```

- **Technical Terms to Include:** RSC payload, serialized tree, 'use client', 'use server', hydration vs RSC merge, streaming, zero-bundle Server Component, backend resource access, component boundary
- **Gotcha:** `context` does not cross the Server-Client boundary. A Context Provider on the server cannot provide values to Client Components — the Provider must itself be a Client Component. Server data must be passed as props.
- **Follow-Up:** "Can you use React Server Components without Next.js?" → Technically yes — RSC is a React feature. But it requires a custom server that understands the RSC protocol and can stream the RSC payload. In practice, Next.js (App Router) and Remix are the production-viable RSC environments. → They're testing ecosystem boundary awareness.
- **Conclusion:** React Server Components are not an optimization layer on top of React — they are a new component model that fundamentally changes the server-client boundary, enabling zero-bundle server rendering and direct backend access as a first-class React capability.

---

_— End of ReactJS_Interview_QA.md —_
_Total coverage: L1–L6 | 7 topics × 2-4 questions avg = ~28 interview-ready QA units_
_Calibrated for: Tech Lead / Assistant Architect level_
