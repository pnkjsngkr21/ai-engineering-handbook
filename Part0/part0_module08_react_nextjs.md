# PART 0 — Prerequisites
## Module 8: React & Next.js Essentials

---

### 1. Learning Objectives
By the end of this module you will be able to:
- Build component-based UIs in React with a correct mental model of
  rendering, state, and effects — not just pattern-matched hooks usage.
- Explain why React re-renders when it does (and doesn't), which is the
  root cause of most React bugs and performance issues.
- Use Next.js's App Router (Server Components vs. Client Components) well
  enough to build a real app with API routes, which is the exact
  architecture Part 7's chat UIs use.
- Manage state correctly at the right level (local component state vs.
  lifted state vs. server state) instead of defaulting to one pattern
  everywhere.
- Recognize when you need a state management library vs. when React's
  built-in tools are enough — an important judgment call, not a checklist.

### 2. Prerequisites
Module 7 (JS/TS async fundamentals) is required — React's rendering model
and hooks rules only make sense once the event-loop and closures mental
models are solid.

### 3. Estimated Study Time
12–16 hours over 6–8 days. This is the densest module in Part 0 —
budget accordingly.

### 4. Difficulty
⭐⭐⭐☆☆ (Medium — React's declarative model is a genuine paradigm shift
from imperative DOM manipulation, though the concepts themselves aren't
hard once the mental model clicks.)

### 5. Why This Matters
Part 7 (Frontend for AI) and every capstone in Part 11 use React/Next.js.
For your freelancing goal, "I can build a full AI product end to end,
frontend included" is a meaningfully more sellable service than backend
alone. This module is the direct on-ramp to that.

---

### 6. Theory

**What is it?**
React is a library for building UIs out of composable components, where
the UI is a **pure function of state**: `UI = f(state)`. You never directly
manipulate the DOM (`document.getElementById(...).innerHTML = ...`); instead
you describe what the UI *should* look like given the current state, and
React handles updating the actual DOM to match.

**Why do we need this (vs. manual DOM manipulation)?**
Manual DOM manipulation becomes unmanageable as UI complexity grows —
you end up with imperative spaghetti tracking "what needs updating when."
React's declarative model, where you re-describe the whole UI from state
and let React diff/reconcile the minimal actual DOM changes, scales far
better to complex, interactive interfaces (like a streaming chat UI with
constantly updating message lists).

**How does it work? (the core mental model: render, don't mutate)**

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

- `useState` gives you a piece of state **and** a setter function.
- Calling the setter doesn't mutate `count` in place — it schedules a
  **re-render**, where the *entire component function runs again* from
  scratch, with `count` now holding the new value.
- React then diffs the new returned JSX against the previous render (the
  "virtual DOM" diff) and applies only the minimal real DOM changes.

**This is the single most important mental model in React:** a component
function is not "set up once and then mutated" — it *re-runs completely*
on every re-render. Variables declared inside it are recreated every time.
This is *why* `useEffect` and `useCallback`/`useMemo` exist — they're tools
for **opting out of** the "recreate everything every render" default when
you specifically need something to persist across renders or run only when
certain values change.

**`useEffect` — the most commonly misunderstood hook:**
```jsx
useEffect(() => {
  const subscription = subscribeToSomething();
  return () => subscription.unsubscribe();   // cleanup, runs before next effect or unmount
}, [someDependency]);
```
Mental model: "after this render is painted to the screen, run this
side-effect; before the *next* time this effect runs (or the component
unmounts), run the cleanup." The dependency array tells React *when* to
re-run the effect — get this array wrong (missing a dependency) and you get
the classic "stale closure" bug, where the effect captures an old value of
a variable from a previous render and never sees updates.

**Server Components vs. Client Components (Next.js App Router) — the part
that's genuinely new even if you already knew React:**
- **Server Components** (the default in the App Router) render on the
  server, ship zero JS to the client for that component, and can directly
  access backend resources (databases, secrets) — but **cannot** use
  hooks like `useState` or handle browser events, because they never run
  in the browser.
- **Client Components** (marked `"use client"` at the top of the file) run
  in the browser, can use hooks and handle interactivity, but their code
  *is* shipped to the client as JS.

The practical rule: **default to Server Components; add `"use client"` only
where you need interactivity, state, or browser-only APIs** (like a chat
input box, or code that renders incoming streamed tokens). This is exactly
the pattern you'll use in Part 7 — the page shell and static parts are
Server Components; the chat message input and streaming message list are
Client Components.

**When should you reach for a state management library (Zustand,
Redux)?**
When state needs to be shared across many distant components and "lifting
state up" (passing it down through props, or via React Context) becomes
genuinely unwieldy. For most single-page AI-app UIs in this handbook,
built-in `useState`/`useContext` plus server state fetched via Next.js are
enough — don't reach for Redux by default; it's genuinely unnecessary
complexity for the scale of what you're building here.

---

### 7. Mental Models

**Model 1 — "A component is a function that reruns completely on every
render; hooks are how you opt out of that default."** This single sentence
demystifies `useState`, `useEffect`, `useMemo`, and `useCallback` all at
once — they all exist to preserve something across, or control the timing
of something relative to, re-renders.

**Model 2 — "Props flow down, events flow up."** Data only ever flows one
direction (parent → child via props); a child communicates back up by
calling a function the parent passed it (an event handler prop) — there's
no direct child-to-parent data flow, which is a deliberate constraint that
keeps data flow traceable in large apps.

**Model 3 — "Server Components are Java's server-rendered JSP/Thymeleaf
equivalent; Client Components are the interactive JS you'd bolt on top."**
If you've ever built a mostly-server-rendered page with a sprinkle of JS
interactivity in a traditional web app, the App Router's split is a
structured, modern version of that exact instinct.

---

### 8. Visual Explanation (described)

**Diagram: "Render cycle"**
```
State changes (setCount(1))
      |
      v
React schedules a re-render
      |
      v
Component function runs again, top to bottom, with new state value
      |
      v
React diffs new JSX output vs. previous render (virtual DOM diff)
      |
      v
React applies ONLY the minimal necessary changes to the real DOM
```

**Diagram: "App Router request flow"**
```
Browser requests /chat
      |
      v
[ Server Component renders page shell + static content on the server ]
      |
      v
HTML (with embedded Client Component "islands") sent to browser
      |
      v
Browser hydrates Client Components (chat input, message list) — now
interactive, running React in the browser for just those parts
```

---

### 9–16. Recommended Resources

**Reading order:**
1. **React official docs (react.dev) — the new "Learn React" section**,
   specifically "Thinking in React," "State: A Component's Memory," and
   "Synchronizing with Effects." This is now the best React resource that
   exists, written by the React team, and explicitly teaches the mental
   models above.
2. **"A Complete Guide to useEffect" — Dan Abramov (overreacted.io)** — the
   definitive deep-dive on the effect dependency array and stale closures,
   from a React core team member.
3. **Next.js official docs — "App Router" fundamentals**, specifically the
   "Server and Client Components" page — read this carefully, it's the
   single most important Next.js-specific concept for this handbook.
4. **Kent C. Dodds's blog, "When to useMemo and useCallback"** — practical
   guidance on when memoization actually matters vs. premature optimization.

**Official documentation:** react.dev (excellent, current), nextjs.org/docs.

**GitHub repos:** the Vercel AI SDK's example chat app (`vercel/ai` repo's
examples directory) — this is nearly exactly the architecture Part 7 will
have you build, worth reading closely once you've completed this module.

---

### 17. Exercises

1. Build a component with `useState` and deliberately trigger a stale
   closure bug by omitting a dependency from a `useEffect`'s dependency
   array; observe the bug, then fix it — this is the single best exercise
   for internalizing why the dependency array matters.
2. Convert a small piece of UI from a single Client Component into a
   Server Component + a small interactive Client Component "island,"
   explaining which parts needed `"use client"` and why.
3. Build a simple form with controlled inputs (`value` + `onChange` tied to
   state) and explain, in your own words, why this is "the React way"
   versus reading `document.querySelector(...).value` imperatively.
4. Deliberately cause an unnecessary re-render cascade (e.g., passing a new
   inline arrow function as a prop on every render to a child wrapped in
   `React.memo`), observe it via React DevTools' render highlighting, then
   fix it with `useCallback`.

### 18. Mini-Project
**Build:** A simple Next.js App Router app with one Server Component page
and one Client Component: a live character counter for a text input (state,
controlled input, no backend needed yet). Focus entirely on getting the
Server/Client Component split and the render-cycle mental model right.

### 19. Production Project
**Build:** `chat-shell` — a Next.js App Router app that's the *structural*
skeleton of a real AI chat UI (no real LLM call yet — that's Part 3/7):
- A Server Component page shell
- A Client Component chat panel: message list (rendered from local state),
  a controlled text input, and a "send" button that appends a fake
  user message and then a simulated streaming assistant reply (reuse the
  async-generator streaming pattern from Module 7's `token-stream-sim`)
- Proper loading and error states for the simulated "send" action
- At least one custom hook (e.g., `useChat()`) extracting the message-state
  logic out of the component, demonstrating you understand hooks as
  reusable, composable state logic, not just built-ins
- Basic tests (React Testing Library) for the chat panel's core
  interaction: typing, sending, seeing the message appear

This becomes the literal frontend skeleton you'll wire a real LLM API into
in Part 3/7 — you're building the real thing now, minus the real backend
call.

---

### 20–21. Practice & Interview Questions

1. Explain why a React component function "re-runs" on every render, and
   what problem hooks like `useState`/`useEffect`/`useMemo` each solve in
   light of that.
2. What causes a stale closure bug in `useEffect`, and how does the
   dependency array prevent it?
3. When would you mark a Next.js component `"use client"`, and what do you
   give up (server-only access, zero-JS shipping) by doing so?
4. Why does React encourage one-directional data flow (props down, events
   up), and what problem does that constraint solve at scale?
5. When would you introduce a state management library like Zustand
   instead of relying on `useState`/Context, and what's the actual signal
   that tells you it's time?

---

### 22. Common Mistakes
- Missing dependencies in a `useEffect` dependency array, causing stale
  closures that silently use outdated values.
- Marking entire pages `"use client"` unnecessarily, shipping far more JS
  to the browser than needed and losing server-only optimizations.
- Mutating state directly (`state.push(x)`) instead of creating a new
  value (`setState([...state, x])`) — React can't detect mutations, only
  new references, so direct mutation silently fails to trigger re-renders.
- Overusing `useMemo`/`useCallback` everywhere "for performance" without
  measuring — genuine premature optimization that adds complexity for no
  benefit in most cases.
- Reaching for Redux/global state management before it's actually needed.

### 23. Debugging Exercise
Given a component where clicking a button appears to update the UI with a
one-render-behind value (a classic symptom of a stale closure from a
missing effect dependency, or of directly mutating an array/object in
state), diagnose using React DevTools and the mental model above, then fix
it properly (correct dependency array, or replacing mutation with a new
reference).

---

### 24. Checklist
- [ ] I can explain, unprompted, why a component "re-running" on every
      render is the correct mental model
- [ ] I've deliberately caused and fixed a stale closure bug
- [ ] I default to Server Components and only add `"use client"` when I
      need interactivity
- [ ] I understand one-directional data flow well enough to explain it to
      someone else
- [ ] I've built `chat-shell` with a custom hook and basic tests

### 25. Summary
React's core mental model — a component re-runs entirely on every render,
and hooks exist to selectively preserve state or control timing across
those re-renders — explains nearly every hook and nearly every common React
bug. Next.js's Server/Client Component split lets you default to
zero-client-JS rendering and add interactivity only where genuinely needed.
The `chat-shell` project is the literal frontend skeleton for the real,
LLM-connected chat UI you'll build starting in Part 3/7.

### 26. Next Steps
Module 9: **FastAPI for Python Backends** — the backend framework nearly
every AI-Python project in this handbook will use going forward, taught
with direct comparisons to Spring Boot so it maps onto knowledge you
already have.

---

**Reply "continue" for Module 9, or flag anything to go deeper on.**
