<!-- Generated: 2026-03-12 -->

# React.js — Interview Q&A

**Level:** Tech Lead / Architect
**Target Role:** Assistant Architect → Tech Lead / Tech Manager
**Questions per Level:** 5 | **Total Questions:** 30

---

## [L1 — Conceptual Foundations](./L1.md)

### [Core Philosophy › Virtual DOM](./L1.md#core-philosophy--virtual-dom)

- **Question:** What is the Virtual DOM, and why did React introduce it instead of operating directly on the real DOM?

### [Core Philosophy › Component Model](./L1.md#core-philosophy--component-model)

- **Question:** React is described as component-based. What does that actually mean architecturally, and what problem does it solve that pre-React jQuery-era development had?

### [Core Philosophy › Unidirectional Data Flow](./L1.md#core-philosophy--unidirectional-data-flow)

- **Question:** What is unidirectional data flow in React, and why is it architecturally significant for large-scale applications?

### [Core Philosophy › Reconciliation](./L1.md#core-philosophy--reconciliation)

- **Question:** Explain React's reconciliation algorithm. What heuristics does it use, and what are the implications of those heuristics for developers?

### [Core Philosophy › Declarative vs Imperative](./L1.md#core-philosophy--declarative-vs-imperative)

- **Question:** React is declarative. Explain what that means, the mental model it produces, and where imperative code still legitimately appears in a React codebase.

---

## [L2 — Core APIs and Fundamental Features](./L2.md)

### [Hooks › useState](./L2.md#hooks--usestate)

- **Question:** How does `useState` work under the hood in terms of call order and state persistence, and what are the common bugs that emerge from misunderstanding it?

### [Hooks › useEffect](./L2.md#hooks--useeffect)

- **Question:** Explain `useEffect`'s execution model, the dependency array contract, and the three categories of bugs it produces when misused.

### [Component Lifecycle › Class vs Hooks Mental Model](./L2.md#component-lifecycle--class-vs-hooks-mental-model)

- **Question:** Map the class component lifecycle methods to their Hook equivalents. Where do the mental models diverge, and why does that divergence matter?

### [Props vs State › Ownership and Mutability](./L2.md#props-vs-state--ownership-and-mutability)

- **Question:** Define props and state precisely. What determines which one a piece of data should be, and what are the signs a developer has chosen wrong?

### [Keys in Lists › Identity and Reconciliation](./L2.md#keys-in-lists--identity-and-reconciliation)

- **Question:** Why is `key` a critical architectural concern in list rendering, and what failure modes emerge from incorrect key usage?

---

## [L3 — Common Patterns and Real-World Usage](./L3.md)

### [State Management › Context API](./L3.md#state-management--context-api)

- **Question:** What is the Context API, when should you use it, and what are the performance trade-offs that make it the wrong choice for high-frequency state?

### [Performance › Memoization (React.memo, useMemo, useCallback)](./L3.md#performance--memoization-reactmemo-usememo-usecallback)

- **Question:** Explain the three memoization primitives in React — `React.memo`, `useMemo`, and `useCallback` — their exact use cases, and the anti-pattern of over-memoizing.

### [Patterns › Controlled vs Uncontrolled Components](./L3.md#patterns--controlled-vs-uncontrolled-components)

- **Question:** Contrast controlled and uncontrolled components in React. When does each pattern fit, and what breaks when you mix them?

### [Patterns › Higher-Order Components (HOCs)](./L3.md#patterns--higher-order-components-hocs)

- **Question:** What is a Higher-Order Component, what problem does it solve, and why has the React community largely migrated away from it toward Hooks?

### [Patterns › Render Props](./L3.md#patterns--render-props)

- **Question:** Explain the render props pattern, its use case, and how it compares to HOCs and Hooks for logic reuse.

---

## [L4 — Advanced Patterns, Optimisation, Edge Cases](./L4.md)

### [Hooks › useReducer](./L4.md#hooks--usereducer)

- **Question:** When should you reach for `useReducer` instead of `useState`, and how does it change the state management mental model?

### [Concurrent Features › Suspense](./L4.md#concurrent-features--suspense)

- **Question:** What is React Suspense, how does it work mechanically, and what is its architectural significance for data fetching?

### [Advanced Patterns › Error Boundaries](./L4.md#advanced-patterns--error-boundaries)

- **Question:** What are Error Boundaries, how do you implement them, and what rendering errors do they NOT catch?

### [Performance › Code Splitting and Lazy Loading](./L4.md#performance--code-splitting-and-lazy-loading)

- **Question:** How does code splitting work in React, what is the loading cascade problem it creates, and how do you solve it without degrading UX?

### [Advanced Patterns › Custom Hooks](./L4.md#advanced-patterns--custom-hooks)

- **Question:** What makes a custom Hook architecturally significant beyond just being a shared function, and what are the rules that distinguish a valid custom Hook from an ordinary utility function?

---

## [L5 — Architecture and System Design](./L5.md)

### [Architecture › Micro-Frontend Integration with React](./L5.md#architecture--micro-frontend-integration-with-react)

- **Question:** Walk through how you would architect a React micro-frontend system using Module Federation. What are the hard problems you'd face at the seam between micro-frontend?

### [Architecture › Design Systems in React](./L5.md#architecture--design-systems-in-react)

- **Question:** How do you architect a React component library intended as a Design System for consumption by multiple product teams? What are the decisions that determine whether it succeeds or becomes abandoned?

### [Architecture › State Architecture at Scale](./L5.md#architecture--state-architecture-at-scale)

- **Question:** For a large React application with 50+ components, walk through how you decide where state lives and what signals indicate the current state architecture is breaking down.

### [Architecture › Server-Side Rendering with React](./L5.md#architecture--server-side-rendering-with-react)

- **Question:** What problems does SSR solve in a React application, what does it cost, and how do hydration errors surface at runtime?

### [Architecture › Performance at Scale (Virtualisation and Large Lists)](./L5.md#architecture--performance-at-scale-virtualisation-and-large-lists)

- **Question:** How would you architect a React UI that needs to display and interact with a list of 10,000 items without degrading to an unusable render performance?

---

## [L6 — Expert Level: Cross-Cutting Concerns, Security, Hard Trade-offs](./L6.md)

### [Expert › React Fiber Architecture](./L6.md#expert--react-fiber-architecture)

- **Question:** Explain React Fiber's architecture. What fundamental limitation of the previous stack reconciler did it solve, and what capabilities did it unlock?

### [Expert › Concurrent Mode and `useTransition`](./L6.md#expert--concurrent-mode-and-usetransition)

- **Question:** What is Concurrent Mode in React 18, how does `useTransition` expose it to developers, and what failure modes does it introduce?

### [Expert › Security (XSS in React)](./L6.md#expert--security-xss-in-react)

- **Question:** How does React protect against XSS by default, where does that protection break down, and what patterns introduce XSS vulnerabilities in a React application?

### [Expert › Accessibility (a11y) in React](./L6.md#expert--accessibility-a11y-in-react)

- **Question:** What are the key accessibility requirements a React Tech Lead should enforce across a component library, and where do React's patterns create accessibility anti-patterns?

### [Expert › Testing Strategy for React at Scale](./L6.md#expert--testing-strategy-for-react-at-scale)

- **Question:** Design the testing pyramid for a large-scale React application. What do you test at each layer, and what are the trade-offs that determine the right balance for a production system?

---

_End of ReactJS_Interview_QA.md_
_Total Questions: 30 | Levels: L1–L6 | 5 questions per level_
_Calibrated for: Tech Lead / Architect | Apple · PayPal · IoT context woven throughout_
