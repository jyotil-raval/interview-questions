<!-- Generated: 2026-03-12 | React 19.2.4 + Next.js 16.1.6 | Combined Master File -->

# React + Next.js Interview Preparation — Tech Lead / Architect Level

> React 19.2.4 + Next.js 16.1.6 | Combined Master Reference | Career OS
> Generated: 2026-03-12

---

## How to Use This File

This is a combined reference — React foundations first, Next.js architecture second.
The two are deeply linked: Next.js App Router is built on React Server Components,
React 19's Actions API is the engine behind Next.js Server Actions,
and React's Suspense model drives Next.js streaming.

Read them together, not as separate topics.

**React covers:** Core language model, hooks, state management, performance, concurrent rendering, component architecture.
**Next.js covers:** Rendering models (SSR/SSG/ISR/PPR), App Router, Server Components in a framework context, caching layers, deployment architecture, security.

**Cross-reference points to watch:**

- `"use client"` / `"use server"` — React's RSC model → Next.js's boundary enforcement
- Suspense + streaming — React primitive → Next.js `loading.tsx` and PPR
- Actions API (React 19) → Next.js Server Actions (`useActionState`, `useFormStatus`)
- `use()` hook (React 19) → Next.js data fetching in Server Components
- React Compiler (React 19.1 stable) → Next.js Turbopack integration

---

---

# PART 1 — React 19.2.4

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

---

---

## 🔗 React → Next.js Bridge: Where the Two Connect

The questions below surface at architect-level interviews when the interviewer
moves from "React knowledge" into "how you apply it in a production framework."
Know these connections cold — they are the seams where shallow prep breaks.

| React Concept           | Next.js Manifestation                                          | Interview Probe                                                                    |
| ----------------------- | -------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| Server Components (RSC) | App Router default — every `page.tsx` is a Server Component    | "How does RSC change your data fetching strategy?"                                 |
| `Suspense`              | `loading.tsx` + `<Suspense>` boundaries in pages               | "How does streaming work and when do you use explicit vs file-based Suspense?"     |
| React 19 Actions API    | Server Actions (`"use server"`, `useActionState`)              | "What is the relationship between React Actions and Next.js Server Actions?"       |
| `use()` hook            | Data fetching in Server Components + `<Suspense>`              | "What does `use()` replace and when do you use it over `await`?"                   |
| `useTransition`         | ISR/PPR re-renders, deferred navigation updates                | "How does `useTransition` interact with Next.js router state?"                     |
| React Compiler          | Turbopack integration — automatic memo elimination             | "What does the React Compiler change about `useMemo` / `useCallback` usage?"       |
| Error Boundaries        | `error.tsx` (must be `"use client"`)                           | "Why must `error.tsx` be a client component?"                                      |
| `startTransition`       | `router.push()` wraps navigation in a transition               | "Why does Next.js navigation use `startTransition` internally?"                    |
| Hydration               | Full Route Cache payload → client rehydration                  | "What is the RSC payload and how does it differ from the HTML sent for hydration?" |
| Context API             | Pattern: server-fetch → pass as prop → client context provider | "How do you share server-fetched data with deeply nested client components?"       |

# PART 2 — Next.js 16.1.6

> Next.js 16.1.6 | Generated: 2026-03-12 | Career OS

---

## 🆕 What's New — Last 3 Versions

### [Next.js 16.1] — December 2025

- **Turbopack File System Caching (`next dev`) — Stable:** Turbopack now persists its build cache to disk between dev server restarts. Cold starts after a restart are dramatically faster — only changed modules are re-bundled. Interviewers will ask: "What is Turbopack, and how does it differ from Webpack?" Know the underlying Rust-based incremental computation model.
- **Next.js Bundle Analyzer (experimental):** First-party `@next/bundle-analyzer` integration — view bundle composition directly in the Next.js build output. Previously required manual Webpack plugin setup. Know when to use this and what to look for (large deps, duplicate packages, unexpected client bundles).
- **`next dev --inspect`:** Starts the dev server with Node.js inspector attached — enables breakpoints in server-side code (Server Components, Route Handlers, Server Actions) directly in Chrome DevTools. This is a significant DX shift — know the previous workarounds this replaces.
- **Security patches for RSC vulnerabilities (CVE-2025-55184 / CVE-2025-55183):** A critical DoS and source code exposure in the RSC protocol. All 13.x–16.x users should be on patched versions. Interviewers at security-aware companies will probe your awareness.
- **Turbopack is now the default bundler:** `next dev` uses Turbopack by default. Webpack is opt-in via `--webpack`. Know the migration implications and what Turbopack does not yet support (some Webpack-specific plugins).

### [Next.js 16.0] — Late 2025

- **Cache Components — new programming model:** `use cache` directive + Partial Pre-Rendering (PPR) stable — a new model where individual components declare their own caching strategy. Eliminates the page-level `fetch` cache and `revalidate` confusion. Interviewers probe: "How does `use cache` differ from `unstable_cache` and from `fetch` caching?"
- **PPR (Partial Pre-Rendering) — stable:** Combines static shell rendering (instant from CDN) with dynamic streaming content — all in one request. The static outer shell (navbar, layout) is served instantly; dynamic inner content streams in. Know the trade-offs vs full SSG and full SSR.
- **Improved developer error messages:** Error overlay rebuilt — shows source maps directly in the browser overlay with file:line precision for Server Component errors. Significant DX improvement.
- **`next.config.ts` support:** TypeScript config file is now officially supported. Type-safe configuration with full IntelliSense.
- **Turbopack for production builds (experimental):** `next build --turbo` available for testing. Not production-recommended yet — but know the roadmap.

### [Next.js 15.x] — 2024–2025

- **Async Request APIs (breaking):** `cookies()`, `headers()`, `params`, `searchParams` are now all async. `await cookies()`, `await headers()`. This is the biggest migration breaking change in recent Next.js history. Interviewers probe why this was done — enables concurrent request processing without blocking.
- **React 19 support stable:** Full React 19 integration — `use()`, Actions API, `useOptimistic`, `useFormStatus`, `useActionState`, ref as prop, form actions. Know which React 19 features are Next.js-specific vs generic React.
- **`unstable_after` API:** Runs code after the response is sent — for analytics, logging, cleanup — without delaying the response to the user. Similar to Vercel's Edge `waitUntil`.
- **`instrumentation.ts` stable:** Define `onRequestError` hook — global error reporting from any rendering mode (SSR, SSG, RSC, Route Handler). Replaces error-tracking setup fragmented across multiple files.
- **Turbopack `next dev --turbo` — stable:** First stable Turbopack release for development. 76.7% of Next.js integration tests passing (at time of announcement).
- **Security — CVE-2025-66478 (CVSS 10.0):** Critical RSC protocol RCE vulnerability. Any candidate who doesn't know about this at an architect-level interview is leaving a gap.

> **Interview signal:** The App Router vs Pages Router distinction, PPR, and Server Components are the three axes on which Next.js architect-level questions concentrate. Every other topic branches from these.

---

## L1 — Conceptual Foundations

---

**L1 — Topics identified:**

- What is Next.js / Framework vs Library distinction
- App Router vs Pages Router — Architecture difference
- Rendering Models — SSR / SSG / ISR / CSR / PPR
- File-based Routing
- The Role of the Server vs Client Boundary
- When to use Next.js vs plain React

---

### Fundamentals

#### What is Next.js / Framework vs Library

- **Question:** What is Next.js, how does it extend React, and what does it mean that React is a library while Next.js is a framework?
- **Answer:**
  - Next.js is a **full-stack React framework** built and maintained by Vercel. It adds to React what React deliberately left out: routing, data fetching conventions, rendering strategy selection, image optimisation, font loading, bundling, and a server runtime layer.
  - **Library vs framework (inversion of control):** React is a library — you call it. You decide when to render, how to route, how to fetch data. Next.js is a framework — it calls you. It defines file conventions (`page.tsx`, `layout.tsx`, `loading.tsx`) and your code fills the slots. You follow the framework's structure; the framework handles the orchestration.
  - Next.js makes the following decisions for you (with escape hatches): bundler (Turbopack/Webpack), routing (file-system based), server runtime (Node.js or Edge), caching strategy (per fetch, per route, per segment), image and font optimisation pipeline.
  - The value: you get production-quality SSR, SSG, ISR, streaming, code splitting, and full-stack API routes out of the box without configuring Webpack, Babel, React Router, SWR, and a Node server individually.
- **Example:**

```
// Next.js file-based routing — the framework calls your exports
app/
  layout.tsx    → wraps all pages (framework calls this)
  page.tsx      → renders at /
  dashboard/
    page.tsx    → renders at /dashboard
    loading.tsx → shown while dashboard/page suspends (framework calls this)
    error.tsx   → shown if dashboard/page throws (framework calls this)

// You fill the slot — the framework handles when/how it's called
// app/dashboard/page.tsx
export default async function DashboardPage() {
  const data = await fetchDashboardData(); // Next.js handles streaming/suspense
  return <Dashboard data={data} />;
}
```

- **Technical Terms to Include:** full-stack framework, inversion of control, file-system routing, rendering pipeline, bundler, App Router, Pages Router, Vercel, server runtime, escape hatch
- **Gotcha:** "Next.js is just React with SSR" undersells it significantly at an architect interview. Next.js is a full rendering pipeline, a server runtime, a CDN caching layer, a bundler, an image/font optimisation system, and a deployment target abstraction — all unified. Framing it as "just SSR" signals shallow familiarity.
- **Follow-Up:** "Can you use Next.js without Vercel?" → Yes — Next.js is open source. Self-hosting on Node.js, Docker, or any server works. OpenNext adapter enables deployment on Cloudflare Workers, AWS Lambda@Edge. The trade-off: Vercel's platform provides ISR storage, edge middleware, image optimisation CDN, and skew protection natively — self-hosting requires you to implement or proxy these. → They're testing deployment architecture awareness beyond Vercel.
- **Conclusion:** Next.js is a full-stack rendering framework, not a React wrapper — the distinction matters architecturally because it owns the full request lifecycle from CDN edge to database query, making framework-level decisions that a plain React app leaves to the developer.

---

#### App Router vs Pages Router

- **Question:** What is the fundamental architectural difference between the App Router and Pages Router, and what does the shift to React Server Components change?
- **Answer:**
  - **Pages Router (legacy, `pages/` directory):** Every page is a React component that runs entirely on the client after hydration. Data fetching is via `getServerSideProps` (SSR), `getStaticProps` (SSG), or `getStaticPaths` (SSG with dynamic routes). All components are client components by default. Limited server-side composition.
  - **App Router (current, `app/` directory):** Built on React Server Components (RSC). Components are **server by default** — they run on the server, have zero client-side JavaScript, and can directly access server resources (databases, file system, secrets). Client components are opt-in via `"use client"` directive.
  - **Key architectural shift:** In the Pages Router, the boundary is at the page level — the full page is SSR'd or SSG'd. In the App Router, the boundary is at the component level — each component independently decides server or client. This enables **nested layouts**, **server-side composition**, and **streaming** at component granularity.
  - **Layouts:** App Router introduces persistent layouts (`layout.tsx`) that do not re-render on navigation between sibling routes — the layout stays mounted, only the page slot changes. Pages Router had no equivalent — layout components re-rendered on every navigation.
  - **Coexistence:** Both routers can coexist in the same project during migration. `app/` takes precedence over `pages/` for the same route.
- **Example:**

```
// Pages Router — data fetching at page level
// pages/dashboard.tsx
export async function getServerSideProps() {
  const data = await fetchData(); // runs server-side
  return { props: { data } };
}
export default function Dashboard({ data }) { // client component
  return <div>{data.title}</div>;
}

// App Router — server component by default, data in-component
// app/dashboard/page.tsx
export default async function Dashboard() {
  const data = await fetchData(); // runs server-side, no prop drilling
  return <div>{data.title}</div>;
}

// app/dashboard/layout.tsx — persists across dashboard/* navigation
export default function DashboardLayout({ children }) {
  return (
    <div>
      <DashboardNav />  {/* stays mounted, never re-renders on page change */}
      {children}        {/* only this slot changes on navigation */}
    </div>
  );
}
```

- **Technical Terms to Include:** App Router, Pages Router, React Server Components, `"use client"`, `getServerSideProps`, `getStaticProps`, nested layouts, streaming, RSC payload, hydration, coexistence
- **Gotcha:** The `"use client"` directive does not mean "this component only runs on the client." It means "this component and its subtree are client components — they are included in the client bundle and run on both server (for HTML generation) and client (for hydration and interactivity)." "Use client" is a boundary declaration, not an execution location declaration.
- **Follow-Up:** "Why did Vercel/React decide to make components server by default?" → Defaulting to server eliminates the most expensive part of web performance: JavaScript sent to the browser. Server components have zero client-side JS, no hydration cost, no bundle size contribution. Client components are the exception — when interactivity is needed. The default should be the cheaper choice. → They're testing philosophical alignment with the RSC mental model.
- **Conclusion:** The App Router's RSC-based architecture is a paradigm shift — not an incremental update — from the Pages Router. The server/client boundary moving from page-level to component-level changes how you think about data flow, bundle size, and performance optimisation across the entire application.

---

#### Rendering Models — SSR / SSG / ISR / CSR / PPR

- **Question:** Explain Next.js's five rendering models, their data freshness trade-offs, and how to decide which to use for a given route.
- **Answer:**
  - **SSG (Static Site Generation):** HTML generated at build time. Served from CDN — fastest TTFB. Data is stale until next build. Best for: content that changes infrequently (blog posts, marketing pages, documentation).
  - **ISR (Incremental Static Regeneration):** SSG with a time-based or on-demand revalidation window. Stale page is served immediately (from CDN cache) while Next.js regenerates it in the background. `revalidate: 60` means the page is at most 60 seconds stale. Best for: frequently changing content that doesn't need real-time accuracy (product listings, news articles).
  - **SSR (Server-Side Rendering):** HTML generated per-request on the server. Always fresh — data is fetched at request time. Slower TTFB than SSG. Best for: user-personalised content, frequently changing real-time data, pages that depend on request cookies/headers.
  - **CSR (Client-Side Rendering):** Server sends a minimal HTML shell; data is fetched in the browser via `useEffect`/SWR/React Query. Worst initial load (blank screen), best for: dashboards behind auth where SEO doesn't matter and data is highly dynamic.
  - **PPR (Partial Pre-Rendering — stable in Next.js 16):** Static outer shell served from CDN instantly; dynamic inner content streams in from the server. A single route can be partially static and partially dynamic. Best for: routes with a static layout and dynamic personalised content.
- **Example:**

```tsx
// SSG — static at build time (App Router)
// app/blog/[slug]/page.tsx
export const revalidate = false; // never revalidate (pure SSG)
export async function generateStaticParams() {
  return posts.map((p) => ({ slug: p.slug }));
}

// ISR — revalidate every 60 seconds
export const revalidate = 60;
export default async function BlogPost({ params }) {
  const post = await fetchPost(params.slug);
  return <Post data={post} />;
}

// SSR — fresh per request
export const dynamic = 'force-dynamic'; // or use cookies()/headers()
export default async function PersonalisedPage() {
  const user = await getCurrentUser(); // reads request cookies
  return <UserDashboard user={user} />;
}

// PPR — static shell + dynamic slot
// next.config.ts
export default { experimental: { ppr: true } }; // enabled in Next.js 16

// app/product/[id]/page.tsx
import { Suspense } from 'react';
export default function ProductPage({ params }) {
  return (
    <div>
      <StaticProductLayout /> {/* pre-rendered at build time */}
      <Suspense fallback={<Skeleton />}>
        <DynamicRecommendations id={params.id} /> {/* streams in */}
      </Suspense>
    </div>
  );
}
```

- **Technical Terms to Include:** SSG, ISR, SSR, CSR, PPR, `revalidate`, `dynamic`, `generateStaticParams`, stale-while-revalidate, on-demand revalidation, Suspense boundary, streaming, TTFB, CDN cache
- **Gotcha:** In the App Router, a route segment's rendering mode is determined by the **most dynamic thing** in that segment — not explicitly set as in Pages Router. If any component in a segment calls `cookies()`, `headers()`, or uses `dynamic = 'force-dynamic'`, the entire segment becomes dynamic (SSR). This opt-out-of-static behaviour is called "dynamic rendering inference" and frequently surprises teams.
- **Follow-Up:** "What is on-demand ISR revalidation and how does it work?" → Instead of time-based expiry, `revalidatePath('/blog/my-post')` or `revalidateTag('blog')` programmatically invalidates a cached page — triggered by a webhook (CMS publish event, database change). The next request regenerates that page. This enables content-driven invalidation rather than time-driven. → They're testing production CMS integration patterns.
- **Conclusion:** Rendering model selection is a trade-off matrix of freshness, performance, and cost — PPR is the architectural direction in Next.js 16 that dissolves the SSG-vs-SSR binary by enabling granular per-component rendering strategy within a single route.

---

## L2 — Core Concepts

---

**L2 — Topics identified:**

- React Server Components (RSC) — model, constraints, composition
- `"use client"` / `"use server"` directives
- Data Fetching in App Router
- Caching — four layers (Request Memoisation / Data Cache / Full Route Cache / Router Cache)
- Server Actions
- File Conventions — layout / page / loading / error / not-found / template
- Navigation — Link / useRouter / redirect / notFound

---

### Core Architecture

#### React Server Components — Model, Constraints, Composition

- **Question:** What are React Server Components, what are their constraints, and how do you compose server and client components correctly?
- **Answer:**
  - **React Server Components (RSC):** Components that run exclusively on the server. They can: use async/await directly, access server-only resources (DB, file system, environment variables), import server-only packages (no browser APIs). They produce zero client-side JavaScript — the output is a serialised React tree (RSC payload), not HTML or JS.
  - **Constraints:** Cannot use: `useState`, `useEffect`, `useRef`, or any hook that requires client runtime. Cannot use browser APIs (`window`, `document`, `localStorage`). Cannot attach event handlers (`onClick`). Cannot be dynamically imported with `ssr: false`.
  - **Client components (`"use client"`):** Run on both server (for initial HTML) and client (for hydration). Can use all hooks and browser APIs. Are included in the client JavaScript bundle.
  - **Composition rules:** You can pass Server Components as **props** (including `children`) to Client Components — the server component renders on the server, its output (serialised React element) is passed to the client component. You **cannot** import a Server Component from inside a Client Component file — that would include it in the client bundle.
  - **`server-only` package:** Import `'server-only'` at the top of a module to cause a build error if it's ever imported in a Client Component — explicit guard for server-only code.
- **Example:**

```tsx
// Server Component — async, direct DB access, no client bundle cost
// app/dashboard/page.tsx (server component by default)
import { db } from '@/lib/db'; // server-only — never reaches client
export default async function DashboardPage() {
  const stats = await db.query('SELECT * FROM stats'); // direct DB
  return (
    <main>
      <h1>Dashboard</h1>
      {/* Pass server data to client component as props */}
      <InteractiveChart data={stats} />
    </main>
  );
}

// Client Component — receives data as prop
// components/InteractiveChart.tsx
('use client');
import { useState } from 'react';
export function InteractiveChart({ data }) {
  // data came from server
  const [selected, setSelected] = useState(null);
  return <ChartLibrary data={data} onSelect={setSelected} />;
}

// CORRECT — pass server component as children to client component
// components/Modal.tsx
('use client');
export function Modal({ children }) {
  // children = server component output
  const [open, setOpen] = useState(false);
  return open ? <dialog>{children}</dialog> : null;
}

// app/page.tsx
export default function Page() {
  return (
    <Modal>
      <ServerSideContent /> {/* server component passed as prop — valid */}
    </Modal>
  );
}
```

- **Technical Terms to Include:** RSC payload, server component, client component, `"use client"`, composition pattern, prop passing, server-only package, hydration, zero client JS, serialisable props
- **Gotcha:** Props passed from Server to Client Components must be **serialisable** — plain objects, arrays, strings, numbers, dates. Functions, class instances, `Map`, `Set`, and `undefined` are not serialisable. If you try to pass a non-serialisable value as a prop across the server/client boundary, Next.js throws at build/runtime. This is a common mistake when passing callback props.
- **Follow-Up:** "What is the RSC payload and how does it differ from HTML?" → The RSC payload is a serialised representation of the React component tree — a custom JSON-like wire format Next.js sends from server to client. It contains the rendered output of server components and placeholder slots for client component positions. The client uses this to reconcile the React tree without needing to re-fetch data — enabling fast client-side navigation while maintaining server component benefits. → They're testing RSC protocol depth.
- **Conclusion:** RSC composition is the core architectural skill in Next.js App Router — the pattern of rendering expensive data operations in server components and pushing the client boundary as deep as possible determines both application performance and the clarity of data flow.

---

#### Caching — The Four Layers

- **Question:** Explain Next.js's four caching layers, how they interact, and how to control or bypass each one.
- **Answer:**
  - **Layer 1 — Request Memoisation:** Deduplicates identical `fetch` calls within a single server render pass. If the same URL is fetched multiple times during one request (across components), it is only called once — the result is cached in memory for the duration of the render. Automatic, no configuration. Implemented via React's `cache()` function.
  - **Layer 2 — Data Cache:** Persists `fetch` responses across requests and deployments (on Vercel, stored in Vercel's data cache; self-hosted, in the filesystem). Controlled via `fetch(url, { next: { revalidate: 60 } })` or `cache: 'no-store'` to opt out. `revalidateTag('tag')` or `revalidatePath('/path')` invalidates on demand.
  - **Layer 3 — Full Route Cache:** Stores the rendered HTML and RSC payload of a route at build time (for static routes) or after first render (for ISR). Served from CDN. Bypassed for dynamic routes.
  - **Layer 4 — Router Cache (Client-side):** The browser caches RSC payloads of visited routes in memory. On back/forward navigation or re-visiting a route within the same session, Next.js serves from this cache — no server round trip. Duration: 30 seconds for dynamically rendered, 5 minutes for statically rendered segments.
  - **Next.js 16 `use cache` directive:** Replaces the confusing per-fetch caching — components and functions declare `use cache` to opt into caching, with explicit `cacheLife` and `cacheTag` controls.
- **Example:**

```tsx
// Layer 1 — Request Memoisation (automatic)
// Both components fetch the same URL — only one network call per render
async function ComponentA() {
  const user = await fetch('/api/user'); // called
  return <div>{user.name}</div>;
}
async function ComponentB() {
  const user = await fetch('/api/user'); // memoised — same request, same response
  return <span>{user.email}</span>;
}

// Layer 2 — Data Cache control
// Cache for 60 seconds
const data = await fetch('https://api.example.com/data', {
  next: { revalidate: 60, tags: ['data-tag'] }
});

// Opt out of Data Cache entirely — always fresh
const liveData = await fetch('https://api.example.com/live', {
  cache: 'no-store'
});

// On-demand revalidation (e.g., in a webhook Route Handler)
import { revalidateTag } from 'next/cache';
revalidateTag('data-tag'); // invalidates all fetches tagged 'data-tag'

// Layer 4 — Router Cache — controlled via Link prefetch
<Link href='/dashboard' prefetch={false}>
  Dashboard
</Link>;
// prefetch=false disables pre-caching this route in the router cache

// Next.js 16 — use cache directive
async function getExpensiveData() {
  'use cache';
  cacheLife('hours'); // cache for hours
  cacheTag('product-data');
  return await fetchFromDB();
}
```

- **Technical Terms to Include:** request memoisation, Data Cache, Full Route Cache, Router Cache, `revalidate`, `cache: 'no-store'`, `revalidateTag`, `revalidatePath`, `cacheTag`, `cacheLife`, `use cache`, stale-while-revalidate, on-demand revalidation, PPR
- **Gotcha:** The biggest source of Next.js App Router confusion is **unexpected caching**. By default, `fetch` in Server Components is cached in the Data Cache indefinitely (in Next.js 14). In Next.js 15, the default was changed to `cache: 'no-store'` — fetch is no longer cached by default. This is a breaking behaviour change between versions that trips up teams upgrading. Always know which version's caching defaults you're on.
- **Follow-Up:** "How do you invalidate the Router Cache (client-side) programmatically?" → `router.refresh()` from `useRouter()` invalidates the Router Cache for the current page and fetches fresh data from the server — but stays on the same page. Alternatively, calling a Server Action that uses `revalidatePath` or `revalidateTag` automatically invalidates the Router Cache for the affected paths. → They're testing client-side cache interaction patterns.
- **Conclusion:** Next.js's four-layer cache is the single most complex aspect of the framework — misunderstanding the interaction between Data Cache and Router Cache is the root cause of the most common production bugs (stale data, unexpected fresh fetches), making cache architecture fluency non-negotiable at Tech Lead level.

---

#### Server Actions

- **Question:** What are Server Actions, how do they work under the hood, and what are their security and architecture implications?
- **Answer:**
  - Server Actions are async functions defined with `"use server"` that run on the server but can be called from client components like regular functions. They are the App Router's replacement for API routes for form submissions and mutations.
  - **How they work:** At build time, Next.js replaces each Server Action with a unique encrypted endpoint ID. When called from the client, a `POST` request is made to the Next.js server with the encrypted action ID and serialised arguments. The server decrypts, validates, executes the function, and returns the result. The encryption key is build-specific — a new build rotates the key (version skew concern).
  - **React 19 integration:** Server Actions integrate with React's `useActionState`, `useFormStatus`, and `useOptimistic` — enabling form state, pending states, and optimistic updates without manual state management.
  - **Security implications:** Server Actions are POST endpoints — they are callable by any HTTP client. CSRF is mitigated by the `Origin` header check Next.js performs. But input validation and authorisation are entirely the developer's responsibility — a Server Action that updates data without checking the current user's session is an authorisation vulnerability.
  - **Revalidation:** After a Server Action mutates data, call `revalidatePath` or `revalidateTag` inside the action to invalidate caches and trigger a re-render.
- **Example:**

```tsx
// Server Action — defined in a server component or separate actions file
// app/actions.ts
'use server';
import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';

export async function createItem(formData: FormData) {
  // Authorisation — always check first
  const session = await getSession();
  if (!session) throw new Error('Unauthorised');

  // Validation
  const name = formData.get('name') as string;
  if (!name || name.length < 2) throw new Error('Name too short');

  // Mutation
  await db.items.create({ name, userId: session.user.id });

  revalidatePath('/items'); // invalidate cached page
  redirect('/items'); // navigate after mutation
}

// Client Component — calls action like a function
('use client');
import { useActionState } from 'react'; // React 19
import { createItem } from './actions';

export function CreateItemForm() {
  const [state, action, isPending] = useActionState(createItem, null);
  return (
    <form action={action}>
      <input name='name' />
      <button disabled={isPending}>{isPending ? 'Creating...' : 'Create'}</button>
      {state?.error && <p>{state.error}</p>}
    </form>
  );
}
```

- **Technical Terms to Include:** Server Action, `"use server"`, action ID, encrypted endpoint, `useActionState`, `useFormStatus`, `useOptimistic`, `revalidatePath`, CSRF, authorisation, progressive enhancement, version skew, serialisable arguments
- **Gotcha:** Server Actions are **not** a replacement for API routes in all scenarios. They're designed for form mutations triggered by a user in your own application. For webhooks (third-party POSTs), public APIs, non-form data (file uploads, binary), streaming responses, or GET requests — use Route Handlers (`app/api/route.ts`). Mixing these concerns is a common architectural mistake.
- **Follow-Up:** "What is progressive enhancement with Server Actions?" → A form using a Server Action works even without JavaScript — the browser's native form submission sends the POST to Next.js, which executes the action and redirects. With JS enabled, React intercepts the submission, calls the action via fetch, and handles the response optimistically. This means the form works in degraded environments (slow JS load, JS disabled) without extra code. → They're testing understanding of Server Actions' design philosophy beyond React-only contexts.
- **Conclusion:** Server Actions collapse the client-server boundary for mutations — they eliminate the boilerplate of API route definition, client-side fetch calls, and manual revalidation, but introduce authorisation and version-skew responsibilities that architects must explicitly design for.

---

#### File Conventions

- **Question:** What are Next.js App Router file conventions, what is each responsible for, and how does the rendering hierarchy compose them?
- **Answer:**
  - **`layout.tsx`:** Wraps the page and all nested routes. Persists across navigation (does not unmount). Receives `children` prop (the page slot). Multiple layouts nest — root layout wraps all; segment layouts wrap their sub-tree. Cannot access the current URL directly (no `params` or `searchParams`).
  - **`page.tsx`:** The UI rendered for a specific route. Receives `params` (dynamic segments) and `searchParams` (query string). Required for a route to be publicly accessible — a folder with only a `layout.tsx` is not a route.
  - **`loading.tsx`:** Automatically wraps the page in a `<Suspense>` boundary. Shown while the page is loading (fetching data). Enables streaming — the layout is rendered immediately, the loading skeleton is shown, then replaced by the page once data is ready.
  - **`error.tsx`:** Error boundary for the segment. `"use client"` required (error boundaries must be client components). Receives `error` and `reset` props. `global-error.tsx` catches errors in the root layout.
  - **`not-found.tsx`:** Rendered when `notFound()` is called from within the segment. Replaces `error.tsx` for 404 scenarios.
  - **`template.tsx`:** Like layout but re-mounts on every navigation — creates a new instance each time the route changes. Use when you need per-navigation re-mount (animations, resetting state on navigate). Rarely needed.
  - **`route.ts`:** Defines an HTTP API endpoint (Route Handler). Exports named functions `GET`, `POST`, `PUT`, `DELETE`, etc.
- **Example:**

```
app/
├── layout.tsx          → root layout, always rendered, wraps everything
├── loading.tsx         → shown while root page loads (rare — usually per-segment)
├── error.tsx           → error boundary for root
├── page.tsx            → renders at /
├── dashboard/
│   ├── layout.tsx      → wraps all /dashboard/* routes, persists on navigation
│   ├── loading.tsx     → skeleton while dashboard/page fetches data
│   ├── error.tsx       → error boundary just for /dashboard
│   ├── page.tsx        → renders at /dashboard
│   └── settings/
│       ├── page.tsx    → renders at /dashboard/settings
│       └── not-found.tsx → shown when notFound() called in settings
├── api/
│   └── webhook/
│       └── route.ts    → POST /api/webhook (Route Handler)
```

```tsx
// loading.tsx — wraps page in Suspense automatically
export default function DashboardLoading() {
  return <DashboardSkeleton />; // shown while DashboardPage fetches
}

// error.tsx — must be client component
('use client');
export default function DashboardError({ error, reset }) {
  return (
    <div>
      <p>Failed to load: {error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  );
}
```

- **Technical Terms to Include:** layout, page, loading, error, not-found, template, route handler, Suspense boundary, error boundary, `notFound()`, `redirect()`, `params`, `searchParams`, colocation
- **Gotcha:** `layout.tsx` cannot read `searchParams` — it doesn't have access to the query string because the layout is shared across navigations and re-rendering it on every query param change would defeat its persistence. Access `searchParams` only in `page.tsx` or via `useSearchParams()` in a client component. This surprises developers who expect layout to behave like a component with full page context.
- **Follow-Up:** "What is the difference between `layout.tsx` and `template.tsx`?" → `layout` persists across navigations in its subtree — the component stays mounted, scroll position is preserved, state is kept. `template` creates a new instance on every navigation to a route within its subtree — useful for enter/exit animations or when you need `useEffect` to run on each navigation. The cost of `template` is that shared state is lost on navigation. → They're testing nuanced file convention awareness.
- **Conclusion:** Next.js file conventions define the rendering hierarchy declaratively — each file is a named slot in the framework's request pipeline, composing layouts, loading states, error boundaries, and data fetching into a structured, automatic streaming architecture.

---

## L3 — Advanced Patterns

---

**L3 — Topics identified:**

- Middleware — Edge Runtime, execution model, use cases
- Dynamic Routes — [slug] / [...slug] / [[...slug]] / (groups) / @slots
- Image Optimisation — next/image, LCP impact
- Font Optimisation — next/font
- Metadata API — static / dynamic / OpenGraph
- Route Groups & Parallel Routes & Intercepting Routes
- Environment Variables & Security boundary

---

### Advanced Routing & Optimisation

#### Middleware

- **Question:** What is Next.js Middleware, where does it run, and what are the architectural use cases and constraints?
- **Answer:**
  - Next.js Middleware runs on the **Edge Runtime** (V8 isolate — not Node.js) **before** the request reaches the cache or server. It can: rewrite URLs, redirect, modify request/response headers, set cookies. It cannot: access the database, use Node.js APIs (`fs`, `crypto` module), return a full response body (only headers/redirects/rewrites).
  - **Execution model:** Middleware runs at the CDN edge — closest to the user. It runs on every request to matched paths (configured via `matcher` in `middleware.ts`). Its latency budget is tiny — it must complete in milliseconds. Any heavy computation blocks all matched requests.
  - **Use cases:** (1) Authentication — check session cookie, redirect unauthenticated users. (2) Internationalisation — detect locale from `Accept-Language`, rewrite to `/en/page`. (3) A/B testing — assign variant via cookie, rewrite to variant URL. (4) Rate limiting (lightweight, edge-side). (5) Geolocation-based routing.
  - **Constraint:** Middleware cannot return a JSX response or access the Node.js runtime. It is strictly for routing decisions and header manipulation. Auth logic that requires a database call should be done in the component or Route Handler, not Middleware — Middleware should only verify a token (stateless check).
- **Example:**

```ts
// middleware.ts — runs on Edge Runtime
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';
import { verifyToken } from '@/lib/auth'; // must be edge-compatible (no Node.js)

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // Auth guard — stateless token check only
  const token = request.cookies.get('session')?.value;
  if (!token) {
    return NextResponse.redirect(new URL('/login', request.url));
  }
  try {
    const payload = await verifyToken(token); // JWT verify — edge-compatible
    // Forward user ID to server components via header
    const response = NextResponse.next();
    response.headers.set('x-user-id', payload.userId);
    return response;
  } catch {
    return NextResponse.redirect(new URL('/login', request.url));
  }
}

// Apply only to protected routes
export const config = {
  matcher: ['/dashboard/:path*', '/settings/:path*']
};
```

- **Technical Terms to Include:** Edge Runtime, V8 isolate, middleware, matcher, NextResponse.redirect, NextResponse.rewrite, NextResponse.next, request headers, response headers, stateless auth, `next/headers`, geolocation
- **Gotcha:** Middleware runs on the Edge Runtime — it has access only to Web API-compatible code. No `fs`, no native Node modules, no `crypto` (except Web Crypto API). Many auth libraries that use `jsonwebtoken` are Node.js-only and will fail in Middleware. Use edge-compatible alternatives like `jose` for JWT verification at the middleware layer.
- **Follow-Up:** "Can Middleware access a database?" → Not practically. The Edge Runtime has no Node.js bindings, so traditional ORMs (Prisma, Drizzle with pg driver) do not work. Edge-compatible databases (Neon serverless driver, PlanetScale edge, Upstash Redis) can be called via HTTP-based clients. But hitting a database on every request in Middleware — which runs before caching — creates latency and cost problems. The pattern: verify a stateless token in Middleware, do the authorisation DB check inside the component or Route Handler after the cache layer. → They're testing auth architecture depth at the edge.
- **Conclusion:** Middleware is a routing decision layer, not a business logic layer — its power is in making fast, stateless decisions at the edge (redirect, rewrite, header injection) before the request reaches the cache or server, and its constraint is that any logic requiring Node.js or a database belongs further into the request pipeline.

---

#### Dynamic Routes & Advanced Routing Patterns

- **Question:** Explain Next.js's advanced routing patterns — dynamic segments, catch-all routes, route groups, parallel routes, and intercepting routes — and when to use each.
- **Answer:**
  - **Dynamic segments:** `[slug]` matches a single segment. Available as `params.slug` in page/layout.
  - **Catch-all:** `[...slug]` matches one or more segments — `/docs/a/b/c` → `params.slug = ['a', 'b', 'c']`. **Optional catch-all:** `[[...slug]]` matches zero or more — `/docs` also matches (slug is `undefined`).
  - **Route Groups `(group)`:** Folder with name in parentheses. Excluded from the URL path. Used to: (1) Apply a shared layout to a subset of routes without affecting the URL. (2) Organise routes by feature without URL impact. Multiple root layouts are possible using route groups.
  - **Parallel Routes `@slot`:** Render multiple pages simultaneously in the same layout. Each `@slot` folder is a named slot in the parent layout via props (`{ children, @analytics, @dashboard }`). Enable modal patterns (one slot shows a modal while another shows the page), dashboards with independent loading states, and conditional rendering based on auth.
  - **Intercepting Routes `(.)path`:** Intercept a route and show it in a different context — the classic pattern is showing a photo in a modal from a feed, but navigating directly to the URL shows the full page. `(.)path` (same level), `(..)path` (one level up), `(...)path` (from root).
- **Example:**

```
app/
├── (marketing)/         → route group — no URL effect
│   ├── layout.tsx       → only applied to marketing pages
│   ├── page.tsx         → /
│   └── about/page.tsx   → /about
├── (app)/               → second route group — different layout
│   ├── layout.tsx       → app-specific layout with auth
│   └── dashboard/page.tsx → /dashboard
│
├── blog/
│   └── [slug]/page.tsx  → /blog/my-post (params.slug = 'my-post')
│
├── docs/
│   └── [...slug]/page.tsx → /docs/a/b/c (params.slug = ['a','b','c'])
│
├── feed/
│   ├── @modal/          → parallel route slot
│   │   └── (.)photo/[id]/page.tsx → intercepting route — shows modal
│   └── page.tsx
│
├── photo/[id]/page.tsx  → direct navigation shows full page
```

```tsx
// Parallel routes in layout
export default function FeedLayout({ children, modal }) {
  return (
    <>
      {children} {/* feed page */}
      {modal} {/* modal slot — null if no intercepted route active */}
    </>
  );
}
```

- **Technical Terms to Include:** dynamic segment, catch-all route, optional catch-all, route group, parallel route, intercepting route, `params`, `generateStaticParams`, slot, `default.tsx`, soft navigation, hard navigation
- **Gotcha:** Parallel routes require a `default.tsx` file in each slot — shown when the slot has no active match (e.g., on initial load or hard navigation to the parent). Without `default.tsx`, Next.js throws a 404 for the unmatched slot. This is the most common setup error when implementing modal-with-intercepting-route patterns.
- **Follow-Up:** "How do you implement a modal that shows on client navigation but shows a full page on direct URL access?" → Use parallel routes + intercepting routes. The feed layout has a `@modal` slot. The intercepting route `(.)photo/[id]` renders the photo in a modal (via the `@modal` slot). Direct navigation to `/photo/[id]` bypasses the intercepting route and renders the full page. The URL is the same in both cases — the context determines the rendering. → They're testing the most architecturally sophisticated Next.js routing pattern.
- **Conclusion:** Next.js's routing primitives — from route groups (organisation) to parallel routes (simultaneous rendering) to intercepting routes (context-sensitive rendering) — enable complex UI patterns like modals, split dashboards, and nested auth layouts without URL gymnastics or complex state management.

---

#### Environment Variables & Security Boundary

- **Question:** How does Next.js handle environment variables, what is the security boundary between server and client, and what are the leakage risks?
- **Answer:**
  - **Server-only variables:** Any variable in `.env` or `.env.local` without the `NEXT_PUBLIC_` prefix is available only in server-side code (Server Components, Route Handlers, Server Actions, Middleware). It is **never** included in the client bundle.
  - **Public variables (`NEXT_PUBLIC_`):** Prefixed variables are inlined into the client-side JavaScript bundle at build time. They are public — visible to any user who inspects the bundle. Never put secrets here.
  - **Build-time substitution:** `NEXT_PUBLIC_*` variables are substituted as string literals at build time — not at runtime. A Docker image built with `NEXT_PUBLIC_API_URL=https://api.staging.com` will always use that URL, even if you change the environment variable on the running container. This is a critical deployment architecture concern for multi-environment setups.
  - **Runtime environment variables (self-hosted):** For server-only variables in containerised deployments, values are read at runtime from the process environment (not build-time). Only `NEXT_PUBLIC_` is baked in at build.
  - **`server-only` package:** Importing `'server-only'` in a module causes a build error if that module is ever imported in a Client Component — prevents accidental server secret leakage.
- **Example:**

```bash
# .env.local
DATABASE_URL=postgres://...          # server-only — never in client bundle
API_SECRET_KEY=sk_live_...           # server-only — never in client bundle
NEXT_PUBLIC_API_URL=https://api.com  # public — inlined in client bundle
NEXT_PUBLIC_ANALYTICS_ID=UA-12345   # public — visible in source

# DANGEROUS — a developer who does this leaks the DB URL to every user:
# NEXT_PUBLIC_DATABASE_URL=postgres://...  # DO NOT DO THIS
```

```tsx
// Server Component — safe to use secret
export default async function Page() {
  const data = await fetch('https://api.com', {
    headers: { Authorization: `Bearer ${process.env.API_SECRET_KEY}` } // ✅ server-only
  });
  return <div>{data}</div>;
}

// Client Component — only public vars available
('use client');
export function Analytics() {
  // process.env.NEXT_PUBLIC_ANALYTICS_ID — available (baked in at build)
  // process.env.API_SECRET_KEY — undefined (correctly excluded)
}

// server-only guard
import 'server-only'; // this module will error if imported in client component
export async function getSecretData() {
  return await db.query(process.env.DATABASE_URL);
}
```

- **Technical Terms to Include:** `NEXT_PUBLIC_` prefix, build-time substitution, server-only variable, client bundle exposure, `.env.local`, `server-only` package, runtime variable, secret leakage, multi-environment build
- **Gotcha:** `NEXT_PUBLIC_` variables are baked into the bundle at **build time**. This means: (1) You cannot change a public env var at runtime by restarting the container. (2) You cannot use different `NEXT_PUBLIC_` values for different tenants serving from the same build. Both require a new build. Many teams discover this only when a staging URL shows up in production — because the build was done in the staging environment.
- **Follow-Up:** "How do you handle different `NEXT_PUBLIC_` values per deployment environment without rebuilding?" → You can't — not natively. Alternatives: (1) Use server-side variables and expose them to the client via a Route Handler or a Server Component that injects them as a script. (2) Use a runtime configuration endpoint that the client fetches on startup. (3) Rebuild per environment (correct approach, enforced via CI). → They're testing deployment architecture depth for multi-environment scenarios.
- **Conclusion:** The server/client environment variable boundary is one of the most impactful — and most violated — security boundaries in Next.js; understanding that `NEXT_PUBLIC_` means "deliberately public, baked at build time" is mandatory before any production deployment.

---

## L4 — Architecture & Performance

---

**L4 — Topics identified:**

- Performance — Core Web Vitals, LCP, INP, CLS optimisation
- next/image — internals, lazy loading, priority, sizes
- Streaming & Suspense — architecture
- Authentication Architecture patterns
- Internationalisation (i18n)
- Error Monitoring — `instrumentation.ts` / `onRequestError`

---

### Performance & Architecture

#### Streaming & Suspense Architecture

- **Question:** How does Next.js implement streaming with Suspense, what is the HTML delivery model, and when is streaming most impactful?
- **Answer:**
  - Streaming enables Next.js to send HTML to the browser in chunks, progressively — instead of waiting for all data to be ready before sending anything. The browser renders and displays content as each chunk arrives.
  - **How it works:** The server renders the page top-down. When it encounters a `<Suspense>` boundary, it sends the fallback HTML immediately and continues rendering the rest of the page. When the suspended component's data resolves, the server sends the resolved HTML chunk and a small script that swaps the fallback with the real content — in place, without a full page reload.
  - **Impact:** TTFB (Time to First Byte) is instant — the static layout/header/fallback arrives immediately. Slow data fetches do not block the entire page render. Users see content progressively — perceived performance improves dramatically vs waiting for all data before sending anything.
  - `loading.tsx` is the coarse-grained version — wraps the entire page in a Suspense boundary automatically. Explicit `<Suspense>` boundaries give fine-grained control — show a skeleton for just the slow component while the rest renders normally.
  - **PPR (Next.js 16):** Static outer shell is served from the CDN (instant), dynamic `<Suspense>`-wrapped content streams from the server — combining the best of SSG and SSR in a single request.
- **Example:**

```tsx
// app/dashboard/page.tsx — fine-grained Suspense
import { Suspense } from 'react';

export default async function DashboardPage() {
  return (
    <main>
      {/* Renders immediately — no data needed */}
      <DashboardHeader />

      {/* Fast data — resolves quickly */}
      <Suspense fallback={<StatsSkeleton />}>
        <StatsPanel /> {/* fetches fast analytics */}
      </Suspense>

      {/* Slow data — doesn't block StatsPanel */}
      <Suspense fallback={<ReportSkeleton />}>
        <ReportPanel /> {/* fetches slow report — streams in later */}
      </Suspense>
    </main>
  );
}

// StatsPanel — async server component
async function StatsPanel() {
  const stats = await fetchStats(); // fast fetch — resolves quickly
  return <Stats data={stats} />;
}

async function ReportPanel() {
  const report = await fetchReport(); // slow fetch — user sees skeleton while waiting
  return <Report data={report} />;
}
// Result: DashboardHeader + fallbacks render immediately
// StatsPanel streams in once its fetch resolves
// ReportPanel streams in later — independently
```

- **Technical Terms to Include:** streaming, Suspense boundary, fallback, progressive HTML, TTFB, hydration, PPR, `loading.tsx`, waterfall vs parallel fetch, `generateStaticParams`, chunk, Transfer-Encoding: chunked
- **Gotcha:** Parallel data fetching within a single server component — fetches are **sequential by default** if you `await` them one by one. To parallelise: use `Promise.all([fetch1(), fetch2()])` — both fetches start simultaneously. Forgetting this creates unnecessary request waterfalls that defeat the purpose of streaming.
- **Follow-Up:** "How does Suspense streaming interact with SEO?" → Streamed content is still rendered server-side — it's just sent in chunks. Search engines that support streaming (Googlebot does) receive the full HTML once all chunks are delivered. For SEO-critical content, ensure it's not wrapped in a Suspense boundary that delays it significantly — or prerender it as static. → They're testing SEO awareness within the streaming model.
- **Conclusion:** Streaming with Suspense is Next.js's mechanism for achieving both fast initial render and fresh data — by decomposing a page into independent loading regions, it enables the browser to display meaningful content immediately while data-dependent sections resolve progressively.

---

#### Authentication Architecture

- **Question:** How do you architect authentication in a Next.js App Router application, covering the full request lifecycle from Middleware to Server Component to Client?
- **Answer:**
  - **Recommended stack:** Session-based (cookie) or JWT-based auth. Libraries: NextAuth.js (Auth.js), Clerk, Lucia Auth. The choice depends on: managed vs self-hosted, database requirements, OAuth provider support.
  - **Three-layer auth pattern:**
    - **Layer 1 — Middleware:** Stateless token/cookie check. Redirect unauthenticated users before they hit the cache or server. Must be edge-compatible — no DB calls. Set user ID in a request header for downstream use.
    - **Layer 2 — Server Components / Route Handlers:** Full authorisation check. Verify the session from the database. Check permissions for the specific resource being accessed. Never trust just the Middleware check — it can be bypassed by manipulating headers.
    - **Layer 3 — Client:** UI-level gating — show/hide based on user role stored in session context. Never treat client-side auth checks as security boundaries — they're UX, not security.
  - **Session propagation:** Read the session cookie in Server Components via `cookies()` from `next/headers`. Pass the authenticated user object as a prop to client components that need it, or use a context provider.
  - **Server Action authorisation:** Every Server Action that mutates data must verify the session independently — Server Actions are HTTP endpoints callable by any client.
- **Example:**

```tsx
// middleware.ts — stateless JWT check, redirect if absent
export async function middleware(req: NextRequest) {
  const token = req.cookies.get('session-token')?.value;
  if (!token) return NextResponse.redirect(new URL('/login', req.url));

  const payload = await verifyJWT(token); // edge-compatible verify
  const res = NextResponse.next();
  res.headers.set('x-user-id', payload.sub); // forward to server components
  return res;
}

// Server Component — full DB session verification
import { cookies } from 'next/headers';
export default async function ProtectedPage() {
  const cookieStore = await cookies(); // Next.js 15 — async
  const sessionId = cookieStore.get('session-id')?.value;
  const session = await db.sessions.findById(sessionId); // DB check
  if (!session || session.expiresAt < new Date()) redirect('/login');

  return <Dashboard user={session.user} />;
}

// Server Action — always re-verify
('use server');
export async function deleteItem(itemId: string) {
  const session = await getSession(); // full DB check — NOT trusting middleware
  if (!session) throw new Error('Unauthorised');
  if (!session.user.permissions.includes('delete')) throw new Error('Forbidden');
  await db.items.delete(itemId);
  revalidatePath('/items');
}
```

- **Technical Terms to Include:** session, JWT, Middleware auth, Server Component auth, Server Action auth, edge-compatible, `cookies()`, `headers()`, authorisation, authentication, NextAuth, Clerk, session propagation, CSRF, defence in depth
- **Gotcha:** Relying solely on Middleware for authorisation is a **critical security vulnerability**. Middleware checks run before the cache — a cached response can be served without Middleware running. Always re-verify the session in the component or Route Handler for sensitive data. Middleware is a UX redirect layer, not the security boundary.
- **Follow-Up:** "How does NextAuth.js (Auth.js) integrate with the App Router?" → Auth.js provides a `auth()` function that reads the session from the cookie/database — callable in Server Components, Route Handlers, and Middleware. The session object is available server-side without prop drilling. The `<SessionProvider>` wraps the app for client-side session access via `useSession()`. Route protection is handled via the Middleware integration. → They're testing practical auth library integration depth.
- **Conclusion:** Auth in Next.js App Router requires a defence-in-depth approach — Middleware for UX redirects at the edge, Server Component/Route Handler for resource-level authorisation, and Server Action re-verification for every mutation — each layer independently enforcing security, none trusting the others.

---

## L5 — Production & Deployment Architecture

---

**L5 — Topics identified:**

- Deployment — Vercel vs Self-Hosted (Docker / AWS / Cloudflare)
- OpenTelemetry & `instrumentation.ts`
- Security — CSP, headers, RSC vulnerabilities
- Monorepo & Multi-Zone Architecture
- Testing Strategy — unit / integration / E2E

---

### Production Architecture

#### Deployment Architecture — Vercel vs Self-Hosted

- **Question:** What are the architectural trade-offs between deploying Next.js on Vercel versus self-hosting, and what must you implement yourself when self-hosting?
- **Answer:**
  - **Vercel:** Serverless functions per route segment, global edge network, automatic ISR storage, image optimisation CDN, Skew Protection, preview deployments per branch, automatic Turbopack integration. Zero configuration. Trade-off: cost at scale, vendor lock-in, cold starts on serverless functions, limited long-lived connection support (WebSockets).
  - **Self-hosted (Node.js server):** `next start` runs a Node.js HTTP server. Supports all Next.js features including Server Actions, ISR, and streaming. Trade-off: you manage infrastructure, scaling, ISR storage (filesystem by default — not compatible with multi-replica deployments), image optimisation (proxied through your server — CPU cost), and deployment orchestration.
  - **Self-hosted ISR:** ISR stores regenerated pages on the filesystem — incompatible with multiple server instances (each has a different cache). Fix: use a shared remote cache (`experimental.incrementalCacheHandlerPath`) — custom cache handler writing to Redis or S3.
  - **Docker deployment pattern:** Multi-stage build — `node:alpine` for the builder, `node:alpine` for the runner. `standalone` output mode (`output: 'standalone'` in `next.config.ts`) produces a self-contained directory with minimal runtime dependencies — ideal for Docker.
  - **OpenNext / Cloudflare:** OpenNext adapter wraps Next.js for deployment on AWS Lambda, Cloudflare Workers, SST. Enables full Next.js feature set outside Vercel.
- **Example:**

```dockerfile
# Dockerfile — Next.js standalone output
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production

# Standalone output — minimal runtime
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public

EXPOSE 3000
CMD ["node", "server.js"]
```

```ts
// next.config.ts — standalone output
export default {
  output: 'standalone',
  experimental: {
    // Custom ISR cache handler for multi-replica deployments
    incrementalCacheHandlerPath: './cache-handler.js'
  }
};
```

- **Technical Terms to Include:** Vercel, self-hosted, `next start`, standalone output, ISR storage, custom cache handler, Docker multi-stage build, OpenNext, Skew Protection, cold start, edge network, `output: 'standalone'`
- **Gotcha:** **Version Skew** — after a deployment, old clients may still be running the previous JS bundle. Server Actions use build-specific encryption keys — old clients cannot call new Server Actions and will see cryptic errors. Vercel's Skew Protection routes old clients to old deployments automatically. Self-hosted deployments need a custom solution: set `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` to a stable value (trades security for stability) or implement a version-check mechanism that prompts users to reload.
- **Follow-Up:** "How do you handle ISR with multiple server replicas (horizontal scaling)?" → By default, ISR stores regenerated pages in the local filesystem — each replica has its own cache, causing cache inconsistency (users get different page versions from different replicas). Solution: implement a custom `incrementalCacheHandlerPath` backed by Redis or S3 — all replicas read/write from the same shared cache. Vercel handles this automatically with their data cache. → They're testing production infra architecture awareness.
- **Conclusion:** The Vercel vs self-hosted choice is not just a cost decision — it's an architecture decision about which infrastructure concerns you accept responsibility for: ISR storage, image optimisation, Skew Protection, and edge network are platform features on Vercel and engineering problems on self-hosted deployments.

---

## L6 — Expert Level

---

**L6 — Topics identified:**

- RSC Protocol Internals
- PPR Architecture Deep Dive
- Security — CVE awareness, CSP with RSC, Server Action attack surface
- Performance Profiling — Bundle Analyser, Lighthouse CI, Web Vitals instrumentation
- Cross-Cutting Concerns at Scale

---

### Expert Level

#### RSC Security Attack Surface & CVE Awareness

- **Question:** What are the known security vulnerabilities in Next.js's RSC implementation, what causes them, and how do you architect defensively?
- **Answer:**
  - **CVE-2025-66478 (CVSS 10.0 — RCE):** A critical vulnerability in the RSC protocol enabling remote code execution via a crafted RSC payload. Affects Next.js 15.x and 16.x. Fixed in patched versions — all users should upgrade. Root cause: insufficient validation of the RSC wire format at the server boundary, enabling injection of executable payloads.
  - **CVE-2025-55184 (High — DoS):** Denial of Service via RSC — a malformed RSC payload could cause the server to exhaust resources. The RSC protocol's streaming nature means a crafted partial payload could keep server connections open indefinitely.
  - **CVE-2025-55183 (Medium — Source Code Exposure):** Source code path exposure via RSC error boundaries — stack traces included file system paths in error responses. Fixed by sanitising error output in production.
  - **Defensive architecture:** (1) Always run the latest patched version — pin to `16.1.6+`. (2) Never expose the RSC endpoint (`/_next/`) without a WAF rule filtering malformed payloads. (3) Never serialize sensitive data (secrets, tokens) in RSC payloads — client components receive the serialised output. (4) Validate all inputs in Server Actions — they are HTTP endpoints. (5) Use `server-only` imports to prevent accidental secret exposure through the RSC boundary.
  - **CSP with RSC:** RSC's inline script injection (for hydration) conflicts with strict CSP `script-src` policies. Use nonce-based CSP with Middleware-generated nonces passed to the `<Script>` tags.
- **Example:**

```tsx
// Defensive Server Action — input validation + authorisation
'use server';
import { z } from 'zod';

const schema = z.object({
  title: z.string().min(1).max(200),
  content: z.string().min(1).max(10000)
});

export async function createPost(formData: FormData) {
  // Authorisation — always first
  const session = await requireSession(); // throws if not authenticated

  // Input validation — always validate, never trust
  const result = schema.safeParse({
    title: formData.get('title'),
    content: formData.get('content')
  });
  if (!result.success) throw new Error('Invalid input');

  // Rate limiting
  await rateLimit(session.userId, 'create-post', 10, '1h');

  await db.posts.create({ ...result.data, authorId: session.userId });
  revalidatePath('/posts');
}

// CSP with nonce — middleware generates nonce per request
// middleware.ts
export function middleware(req: NextRequest) {
  const nonce = Buffer.from(crypto.randomUUID()).toString('base64');
  const res = NextResponse.next();
  res.headers.set('Content-Security-Policy', `script-src 'self' 'nonce-${nonce}' 'strict-dynamic'; ...`);
  res.headers.set('x-nonce', nonce); // pass to layout for <Script nonce>
  return res;
}
```

- **Technical Terms to Include:** RSC wire format, CVE-2025-66478, CVSS 10.0, Server Action attack surface, CSP, nonce, WAF, input validation, authorisation, rate limiting, `server-only`, secret serialisation, `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY`
- **Gotcha:** The RSC payload contains a serialised representation of your component tree — if a Server Component renders a value that includes sensitive information (even as part of a computed string that looks innocent), that value travels across the wire to the client. Always treat the RSC payload as client-visible data — the same rule applies as for API responses.
- **Follow-Up:** "How do you implement Content Security Policy in Next.js given RSC's inline script requirements?" → RSC hydration requires inline `<script>` tags. A strict `script-src: 'self'` without `'unsafe-inline'` blocks these. Solution: nonce-based CSP — Middleware generates a unique nonce per request, sets it in the CSP header, and passes it to the root layout via a header. The layout applies the nonce to all `<Script>` tags. Turbopack/Webpack must also respect the nonce in their chunk loading. → They're testing production security architecture depth.
- **Conclusion:** Next.js's RSC protocol — being a novel, complex wire format with a large attack surface — requires proactive version management, defensive Server Action design, WAF protection, and CSP hardening; architects who treat security as an afterthought in RSC-based apps are building systems with known critical vulnerabilities.

---

#### Performance Profiling & Core Web Vitals Architecture

- **Question:** How do you measure and systematically improve Core Web Vitals in a Next.js application at the architecture level?
- **Answer:**
  - **LCP (Largest Contentful Paint — target < 2.5s):** Dominated by the hero image or largest text block. Fix: `<Image priority />` on the LCP image (disables lazy loading, preloads), correct `sizes` attribute to avoid oversized images, CDN delivery via Next.js image optimisation, PPR to serve the static shell (containing the LCP element) from CDN instantly.
  - **INP (Interaction to Next Paint — target < 200ms):** Time from user interaction to next paint. Fix: reduce client-side JavaScript bundle (push to Server Components), avoid long tasks blocking the main thread, use `useTransition` for non-urgent state updates (defers re-render), debounce heavy event handlers.
  - **CLS (Cumulative Layout Shift — target < 0.1):** Unexpected layout jumps. Fix: explicit `width` and `height` on images (Next.js Image enforces this), skeleton loaders matching content dimensions, `font-display: optional` or `next/font` to eliminate FOUT, avoid inserting content above existing content.
  - **Measurement:** `useReportWebVitals` hook (Next.js built-in) sends CWV metrics to your analytics. Lighthouse CI in the deployment pipeline catches regressions. `@next/bundle-analyzer` (Next.js 16.1) reveals bundle composition — identify unexpected client-side bloat.
  - **`next/image` non-negotiable rules:** Always set `priority` on above-the-fold images. Always set `sizes` to avoid downloading full-width images on mobile. Use `fill` for images in responsive containers.
- **Example:**

```tsx
// LCP image — correct next/image setup
import Image from 'next/image';
export function HeroBanner() {
  return (
    <Image
      src='/hero.jpg'
      alt='Hero'
      width={1200}
      height={600}
      priority // disables lazy loading — preloads for LCP
      sizes='(max-width: 768px) 100vw, 1200px' // correct size hint
      quality={85}
    />
  );
}

// INP — defer non-urgent state updates
('use client');
import { useTransition } from 'react';
export function FilterPanel({ onFilter }) {
  const [isPending, startTransition] = useTransition();
  const handleChange = (value) => {
    startTransition(() => {
      onFilter(value); // deferred — doesn't block immediate interaction response
    });
  };
  return <select onChange={(e) => handleChange(e.target.value)} />;
}

// Web Vitals reporting
// app/layout.tsx
('use client');
import { useReportWebVitals } from 'next/web-vitals';
export function Analytics() {
  useReportWebVitals((metric) => {
    analytics.track('web-vital', { name: metric.name, value: metric.value });
  });
  return null;
}

// Bundle analysis — Next.js 16.1
// next.config.ts
import { withBundleAnalyzer } from '@next/bundle-analyzer';
export default withBundleAnalyzer({ enabled: process.env.ANALYZE === 'true' })({});
// ANALYZE=true npm run build
```

- **Technical Terms to Include:** LCP, INP, CLS, Core Web Vitals, `next/image`, `priority`, `sizes`, `useReportWebVitals`, `@next/bundle-analyzer`, `useTransition`, font optimisation, `next/font`, Lighthouse CI, TTFB, streaming, PPR
- **Gotcha:** `<Image>` without an explicit `sizes` prop will download the largest possible image on all screen sizes — often a 1200px image on a 375px mobile screen. This directly impacts LCP. The `sizes` prop tells the browser which image width to request at each viewport breakpoint. Always set it for any `Image` not at `fill` mode.
- **Follow-Up:** "How do you prevent Core Web Vitals regressions in a CI/CD pipeline?" → Lighthouse CI (`@lhci/cli`) runs Lighthouse against a staging deployment on every PR. Assert that LCP < 2500ms, CLS < 0.1, INP < 200ms — fail the build if assertions fail. Bundle size budgets (`next-bundle-analyzer` with a size limit assertion) catch bundle bloat before merge. → They're testing production performance quality gate awareness.
- **Conclusion:** Core Web Vitals in Next.js are an architectural responsibility, not a post-launch audit — the rendering model (PPR for instant static shell), image handling (`priority`, `sizes`), bundle composition (Server Components minimising client JS), and font loading (`next/font`) must all be correct from the start for scores to be in the green.

---

_— End of NextJS_Interview_QA.md —_
_Total coverage: L1–L6 | 6 levels × 4-6 topics avg = ~32 interview-ready QA units_
_Calibrated for: Tech Lead / Architect level_
