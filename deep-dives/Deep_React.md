# Deep React — 40LPA Level
*Fiber Architecture · Reconciliation · Concurrent Mode · Hooks Internals · Performance*

---

# Chapter 1: React Fiber Architecture

## What Fiber Replaced and Why

React 15 used a "stack reconciler" — it recursively compared the old and new virtual DOM trees in one synchronous pass. For large trees, this blocked the main thread for hundreds of milliseconds, causing jank (dropped frames, frozen UI).

React 16 introduced **Fiber** — a complete rewrite of the reconciler that can pause, resume, and abort rendering work. This enables concurrent features (useTransition, Suspense, streaming SSR).

```
Stack Reconciler (React 15):
  Start rendering → process ALL components → commit to DOM
  Cannot stop midway. If it takes 200ms, the UI is frozen for 200ms.

Fiber Reconciler (React 16+):
  Start rendering → process some components → PAUSE
  → handle user input (high priority) → RESUME rendering
  → process more components → PAUSE if needed → commit to DOM
  
  Can split rendering into small chunks, yielding to the browser between them.
```

## What Is a Fiber?

A Fiber is a JavaScript object that represents a unit of work. Each React element (component, DOM node) has a corresponding Fiber node. The Fiber tree is a linked list (not a recursive tree), which makes it pausable.

```
Fiber Node Structure (simplified):
{
  tag:           3,              // FunctionComponent, ClassComponent, HostComponent, etc.
  type:          MyComponent,    // the component function/class
  key:           null,           // for list reconciliation
  stateNode:     <div>,          // the actual DOM node (for host components)
  
  // Tree links (linked list, not recursive calls)
  return:        parentFiber,    // parent
  child:         firstChildFiber, // first child
  sibling:       nextSiblingFiber, // next sibling
  
  // Work
  pendingProps:  { name: "Alice" },
  memoizedProps: { name: "Bob" },    // props from last render
  memoizedState: { count: 0 },       // state from last render
  updateQueue:   [...],              // pending state updates
  
  // Effects
  flags:         Placement | Update,  // what needs to happen in commit phase
  
  // Priority
  lanes:         0b0000010,          // scheduling priority (bitmask)
}
```

```
Fiber Tree for:
  <App>
    <Header />
    <Main>
      <Article />
      <Sidebar />
    </Main>
  </App>

Linked list structure (not recursion!):

  App
  │ child
  ▼
  Header ──sibling──▶ Main
                       │ child
                       ▼
                       Article ──sibling──▶ Sidebar

  Each fiber has: child, sibling, return (parent)
  Traversal: go to child → go to sibling → return to parent → go to sibling
  This traversal can be paused and resumed at ANY fiber node.
```

## Two-Phase Rendering

```
PHASE 1: RENDER (interruptible)
  Build/update the Fiber tree.
  Call component functions (render methods).
  Diff old vs new props/state.
  Determine what changed (effects list).
  NO DOM mutations happen here.
  CAN be interrupted, restarted, or abandoned.

PHASE 2: COMMIT (synchronous, NOT interruptible)
  Apply all DOM mutations at once.
  Run useLayoutEffect callbacks.
  Browser paints.
  Run useEffect callbacks (asynchronously, after paint).
  
  Must be synchronous — you can't show a half-updated UI.
```

---

# Chapter 2: Reconciliation Algorithm

## How React Diffs

React doesn't compare the entire tree (that would be O(n³)). It uses two heuristics to achieve O(n):

```
Heuristic 1: Different types → destroy and rebuild
  Old: <div><Counter /></div>
  New: <span><Counter /></span>
  
  <div> → <span> (different type) → destroy entire <div> subtree,
  create new <span> subtree from scratch.
  Counter's state is LOST (it's a completely new instance).

Heuristic 2: Same type → update in place
  Old: <div className="old" />
  New: <div className="new" />
  
  Same type (<div>) → keep the DOM node, just update className.
  
  Old: <Counter count={1} />
  New: <Counter count={2} />
  
  Same type (Counter) → keep the instance, update props.
  State is PRESERVED.
```

## Keys and List Reconciliation

```
WITHOUT keys (index-based):
  Old: [A, B, C]    indices: [0, 1, 2]
  New: [C, A, B]    indices: [0, 1, 2]
  
  React compares by position:
    index 0: A → C  → update in place (A becomes C — state gets mixed up!)
    index 1: B → A  → update in place
    index 2: C → B  → update in place
  3 updates. And state from component A is now on component C!

WITH keys:
  Old: [A(key=a), B(key=b), C(key=c)]
  New: [C(key=c), A(key=a), B(key=b)]
  
  React matches by key:
    key=c: was at index 2, now at index 0 → MOVE
    key=a: was at index 0, now at index 1 → MOVE
    key=b: was at index 1, now at index 2 → MOVE
  3 moves, 0 updates. State is preserved correctly.
```

**Why index as key is dangerous:**
```javascript
// Each Todo has internal state (e.g., "editing" mode)
{todos.map((todo, index) => (
  <TodoItem key={index} todo={todo} />  // BAD
))}

// User deletes the first todo:
// Old: [Todo1(key=0), Todo2(key=1), Todo3(key=2)]
// New: [Todo2(key=0), Todo3(key=1)]
//
// React thinks: key=0 changed content (Todo1→Todo2), key=1 changed (Todo2→Todo3)
// It UPDATES existing components instead of removing Todo1
// Result: Todo2 gets Todo1's state (editing mode, etc.)

// FIX: use unique IDs
{todos.map(todo => (
  <TodoItem key={todo.id} todo={todo} />  // GOOD
))}
```

---

# Chapter 3: Hooks Internals

## How Hooks Work Under the Hood

Hooks are stored as a linked list on the Fiber node. Each `useState`, `useEffect`, etc. creates a hook node. React tracks hooks by their ORDER of execution — this is why hooks can't be inside conditionals or loops.

```
Component:
  function Counter() {
    const [count, setCount] = useState(0);      // hook 1
    const [name, setName] = useState("Alice");   // hook 2
    useEffect(() => { ... }, [count]);            // hook 3
    return <div>{count} {name}</div>;
  }

Fiber's hook linked list:
  hook1 { memoizedState: 0, queue: [...], next: hook2 }
    → hook2 { memoizedState: "Alice", queue: [...], next: hook3 }
      → hook3 { memoizedState: { effect, deps: [0] }, next: null }

On re-render, React walks this list IN ORDER:
  Call 1 (useState): read hook1.memoizedState → 0
  Call 2 (useState): read hook2.memoizedState → "Alice"
  Call 3 (useEffect): compare deps with hook3.memoizedState.deps

If you put useState inside an `if`:
  if (condition) {
    const [x, setX] = useState(0);   // sometimes hook1, sometimes skipped
  }
  const [y, setY] = useState("");    // sometimes hook1, sometimes hook2

  The linked list order shifts → React reads the WRONG state.
  This is why "Rules of Hooks" exist.
```

## useState Batching

```javascript
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);    // schedules update
    setCount(count + 1);    // schedules update (same base value!)
    setCount(count + 1);    // schedules update
    // Result: count goes from 0 to 1 (NOT 3!)
    // Because all three read `count` from the same render (0)

    // FIX: use functional updates
    setCount(prev => prev + 1);   // 0 → 1
    setCount(prev => prev + 1);   // 1 → 2
    setCount(prev => prev + 1);   // 2 → 3
    // Result: count goes from 0 to 3 ✓
  }
}

// React 18: ALL updates are batched (even in setTimeout, promises, event handlers)
// React 17: only React event handlers were batched
```

## useEffect Lifecycle

```
MOUNT (first render):
  1. React renders component (builds Fiber, diffs)
  2. Commit: DOM is updated
  3. Browser paints
  4. useEffect runs (with empty cleanup)

RE-RENDER (props/state change):
  1. React re-renders component
  2. Commit: DOM is updated
  3. Browser paints
  4. CLEANUP of previous useEffect runs (return function)
  5. NEW useEffect runs

UNMOUNT:
  1. CLEANUP of useEffect runs
  2. DOM node removed

useLayoutEffect is the same but runs BEFORE the browser paints (step 3).
Use for DOM measurements that must happen before the user sees the screen.
```

## Stale Closures (The #1 Hooks Bug)

```javascript
function Timer() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      console.log(count);     // STALE! Always logs 0
      setCount(count + 1);    // STALE! Always sets to 1
    }, 1000);
    return () => clearInterval(id);
  }, []);   // empty deps → effect runs once → closure captures count=0 forever

  // FIX 1: add count to deps (but creates new interval every second)
  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1);
    }, 1000);
    return () => clearInterval(id);
  }, [count]);

  // FIX 2: use functional update (no dependency on count)
  useEffect(() => {
    const id = setInterval(() => {
      setCount(prev => prev + 1);   // doesn't read count, no stale closure
    }, 1000);
    return () => clearInterval(id);
  }, []);   // empty deps is fine now

  // FIX 3: useRef to escape the closure
  const countRef = useRef(count);
  countRef.current = count;   // always up-to-date

  useEffect(() => {
    const id = setInterval(() => {
      console.log(countRef.current);   // always current value
    }, 1000);
    return () => clearInterval(id);
  }, []);
}
```

---

# Chapter 4: Concurrent Features

## Concurrent Rendering Model

```
Synchronous (React 17):
  User types → setState → render → commit → THEN handle next input
  If render takes 200ms, input is delayed by 200ms.

Concurrent (React 18):
  User types → setState (urgent) → render starts → USER TYPES AGAIN →
  React INTERRUPTS the render → handles new input immediately →
  RESUMES or RESTARTS the previous render
  
  The render can be paused without committing, so the user never sees stale UI.
```

## useTransition

```javascript
function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  function handleChange(e) {
    const text = e.target.value;
    setQuery(text);                    // URGENT: update input immediately

    startTransition(() => {
      setResults(filterHugeList(text)); // NON-URGENT: can be interrupted
    });
  }

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending && <Spinner />}
      <ResultList results={results} style={{ opacity: isPending ? 0.7 : 1 }} />
    </>
  );
}
// While filtering, if the user types again:
// React ABANDONS the in-progress filter → handles the new keystroke first
```

## React Server Components (RSC)

```
Traditional React:
  Server sends HTML shell → Browser downloads JS → Hydration → Interactive
  ALL components run in the browser. All component JS is in the bundle.

Server Components:
  Server renders components → sends serialized output (not JS) → Browser renders
  Server components NEVER ship JS to the client. Zero bundle size impact.
  
  'use client'  → this component runs in the browser (interactive)
  (default)     → server component (no useState, no useEffect, but can do DB queries)

  // Server component (runs on server, not in bundle)
  async function UserProfile({ userId }) {
    const user = await db.query('SELECT * FROM users WHERE id = $1', [userId]);
    return <div>{user.name}</div>;   // no JS shipped for this component
  }

  // Client component (runs in browser, in bundle)
  'use client';
  function LikeButton() {
    const [liked, setLiked] = useState(false);
    return <button onClick={() => setLiked(!liked)}>Like</button>;
  }
```

---

# Chapter 5: Performance Patterns

## When and Why React Re-renders

```
A component re-renders when:
  1. Its own state changes (useState setter called)
  2. Its parent re-renders (props may have changed — React doesn't check by default!)
  3. A context it consumes changes

A component does NOT re-render when:
  • Its sibling re-renders
  • Its child re-renders
  • Its props haven't changed AND it's wrapped in React.memo

THE BIG MISCONCEPTION: "React only re-renders if props change."
WRONG. By default, React re-renders ALL children when a parent re-renders,
regardless of whether their props changed. React.memo opts into prop comparison.
```

## React.memo Deep Dive

```javascript
// Without memo: Child re-renders every time Parent re-renders
function Parent() {
  const [count, setCount] = useState(0);
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      <ExpensiveChild data={staticData} />  {/* re-renders on EVERY click! */}
    </>
  );
}

// With memo: Child only re-renders if its props actually change
const ExpensiveChild = React.memo(function ExpensiveChild({ data }) {
  // expensive rendering work
  return <div>{/* ... */}</div>;
});

// BUT: memo uses SHALLOW comparison
// If Parent creates a new object/function every render, memo is useless:
function Parent() {
  const config = { theme: 'dark' };              // NEW object every render
  const handleClick = () => console.log('click'); // NEW function every render

  return <ExpensiveChild config={config} onClick={handleClick} />;
  // memo compares: prevConfig === nextConfig → false (different references!)
  // Re-renders anyway!
}

// FIX: memoize the props
function Parent() {
  const config = useMemo(() => ({ theme: 'dark' }), []);
  const handleClick = useCallback(() => console.log('click'), []);

  return <ExpensiveChild config={config} onClick={handleClick} />;
  // Now memo works: same references → skip re-render ✓
}
```

## Children as Props Pattern (Avoiding Re-renders)

```javascript
// PROBLEM: Input re-renders ExpensiveTree on every keystroke
function App() {
  const [text, setText] = useState('');
  return (
    <div>
      <input value={text} onChange={e => setText(e.target.value)} />
      <ExpensiveTree />  {/* re-renders on EVERY keystroke! */}
    </div>
  );
}

// FIX: lift the input into its own component
function App() {
  return (
    <div>
      <InputSection />     {/* state is here, re-renders only this */}
      <ExpensiveTree />    {/* NOT a child of InputSection, doesn't re-render */}
    </div>
  );
}

function InputSection() {
  const [text, setText] = useState('');
  return <input value={text} onChange={e => setText(e.target.value)} />;
}
```

---

# Chapter 6: 🧩 40LPA Interview Questions

**Q: How does React Fiber make concurrent rendering possible?**
A: Fiber represents each component as a linked list node with child, sibling, and return pointers. Unlike the stack reconciler which used recursive function calls (which can't be paused), the linked list can be traversed incrementally. React processes one Fiber node, checks if there's time remaining (via `requestIdleCallback` / scheduler), and either continues or yields to the browser. This is why rendering can be interrupted for user input.

**Q: Explain the complete lifecycle of a useEffect.**
A: On mount: render → commit (DOM update) → browser paint → effect callback runs. On re-render: render → commit → paint → cleanup of PREVIOUS effect → new effect runs. On unmount: cleanup runs. `useLayoutEffect` is identical except it runs synchronously BEFORE paint. Effects are asynchronous by design — they don't block visual updates.

**Q: Why can't hooks be inside conditions?**
A: Hooks are stored as a linked list on the Fiber node, identified by their CALL ORDER (not by name). On re-render, React walks this list sequentially — call 1 reads hook 1, call 2 reads hook 2. If a condition skips a hook call, all subsequent hooks read the WRONG state. The linked list order must be identical between renders.

**Q: What's the difference between useMemo, useCallback, and React.memo?**
A: `useMemo(fn, deps)` — memoizes a computed VALUE. Re-computes only when deps change. `useCallback(fn, deps)` — memoizes a FUNCTION reference. Shorthand for `useMemo(() => fn, deps)`. `React.memo(Component)` — memoizes a COMPONENT. Skips re-render if props haven't changed (shallow compare). They work together: `memo` checks props, but won't help unless the parent memoizes the props it passes (using useMemo/useCallback).

**Q: How does React's key prop work in lists?**
A: Keys let React match elements between renders. Without keys, React compares by array index — reordering causes unnecessary updates and state corruption. With keys, React builds a map of key→element, then matches old and new lists by key. Matched elements are updated in place (state preserved); unmatched old elements are destroyed; unmatched new elements are created. Keys must be stable, unique among siblings, and predictable.

**Q: Explain React Server Components.**
A: Server Components render on the server and send serialized output (not JavaScript) to the client. They never appear in the client bundle — zero impact on bundle size. They can directly access databases, file systems, and server-only APIs. They can't use state, effects, or browser APIs. Client Components (marked with 'use client') are the traditional interactive components. A Server Component can render Client Components as children, but not vice versa.
