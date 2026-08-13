# The Full-Stack Debugging Playbook
### React · NestJS · Express · FastAPI — bare metal, VS Code + Chrome DevTools

---

## 0. The one idea that makes all debuggers the same

Every debugger you will ever touch — Chrome, VS Code, `pdb`, `gdb`, a Java IDE — is built from the **same five primitives**. Learn them once and every new tool becomes a matter of "where's the button."

| Primitive | What it means | Why it beats `console.log` |
|---|---|---|
| **Breakpoint** | Freeze the program at a line | You choose *when* to look, not what to print |
| **Step** | Advance one line / into / out of a call | You see the actual path taken, not the path you assumed |
| **Call stack** | Who called whom to get here | Answers "how did we even get to this line?" — logs never answer this |
| **Scopes** | Every variable alive at this instant | You see the things you *forgot* to log — usually the bug |
| **Evaluate** | Run arbitrary code in the frozen state | Test a fix in 2 seconds without a reload cycle |

`console.log` gives you a *sample* of state you guessed was relevant. A breakpoint gives you *all* state plus the path. The whole skill is learning to trust the pause.

**The habit swap:** every time your hand reaches for `console.log`, ask "could this be a **logpoint** instead?" A logpoint is a breakpoint that prints and continues — same output, zero code edits, no rebuild, no forgotten `console.log` in a PR. Both Chrome and VS Code have them. This one swap gets you 60% of the way.

---

## 1. The universal debugging loop (the actual template)

Follow this in order. Most wasted debugging time comes from skipping steps 1–3 and breakpointing blind.

### Step 1 — Make it reproduce on demand
Do not debug an intermittent bug. Convert it into a deterministic one first: a URL, a click sequence, a `curl` command, or best of all a failing test. If you can't reproduce it in under 15 seconds, fix *that* first — you'll run this loop 20 times.

> Write the repro down as a literal command. `curl -X POST localhost:8000/orders -d '{"qty":0}'` is a better bug report to yourself than "checkout breaks sometimes."

### Step 2 — Bisect the boundary before you bisect the code
You have four processes (browser, Nest/Express, FastAPI, DB). Find which one owns the bug *before* opening any debugger. Three questions, in order:

1. **Did the request leave the browser?** → DevTools **Network** tab. No request = frontend bug. Stop here.
2. **What exactly went over the wire?** → Network → the request → **Payload** and **Response**. If the payload is wrong, it's frontend. If the payload is right and the response is wrong, it's backend. This single check settles most "whose bug is it" arguments.
3. **Did the backend see what you think it saw?** → one breakpoint at the top of the handler, look at the parsed body/params.

You've now cut the search space by ~75% without stepping through a single line.

### Step 3 — Breakpoint at the *symptom*, then walk backwards
Beginners breakpoint where they *think* the bug is. That's a guess. Instead:

- Put the breakpoint where the wrong value is **observed** (the render, the response serialization, the DB write).
- Confirm the value is wrong *there*.
- Now walk **up the call stack** frame by frame, checking the value at each level, until you find the frame where it was still correct.
- The bug lives between that frame and the one below it. Binary search on the stack, not on the code.

This works even in a codebase you've never seen, which is the whole point.

### Step 4 — Ask a yes/no question in the evaluate console
Once paused, don't just stare. Form a hypothesis and test it in the debug console:

```js
// paused in the frozen state:
user.permissions.includes('admin')   // → false. There it is.
```

```python
# paused in pdb / VS Code debug console:
[o.status for o in order.items]   # → ['pending','pending'] not ['paid',...]
```

One evaluated expression replaces a print-rebuild-reload cycle. This is where the real speed comes from.

### Step 5 — Prove the fix in the frozen state before writing it
You can mutate variables while paused. Set the value to what you *think* is right, hit Resume, and see if the symptom disappears. If it doesn't, your diagnosis was wrong — and you found that out before writing any code.

### Step 6 — Widen the net if you're stuck
If steps 3–5 fail after ~10 minutes, you're wrong about *where* the bug is, not *what* it is. Switch tools:

- **Pause on exceptions** (including caught ones) and re-run — a swallowed `try/catch` is hiding it.
- **Pause on all network requests** (XHR breakpoint / `debugpy` on the route) — maybe a request you didn't know about is firing.
- **DOM breakpoints** — if the UI changes and you can't find who does it.
- **Time-based tools** (Performance / py-spy) — if it's slow rather than wrong.

### Step 7 — Close the loop
Remove breakpoints, and **convert the bug into a test**. A conditional breakpoint you needed today is an assertion you want forever.

---

## 2. Chrome DevTools — the parts nobody shows you

Open with `F12` / `Cmd+Opt+I`. Two shortcuts to learn before anything else:

- **`Cmd/Ctrl + Shift + P` — the Command Menu.** Type anything: "coverage", "rendering", "screenshot", "disable JavaScript". Every hidden panel lives here. This is the single highest-leverage key combo in the browser.
- **`Esc` — toggle the drawer.** Gives you a Console while staying in the Sources or Network panel. You want the console *while* paused at a breakpoint, not instead of it.

### 2.1 Sources panel — setup first, or nothing works

**Source maps.** You write TSX, the browser runs bundled JS. Source maps translate back. Vite dev has them on by default; for production debugging set `build.sourcemap: true`. Verify: in Sources, your file tree should show `.tsx` files under a `src/` node. If you only see `chunk-XYZ.js`, source maps are broken and every step below is unusable — fix that first.

**The Ignore List — the React-debugging superpower.** By default, stepping through React code drops you into `react-dom` internals and scheduler guts, and your call stack is 40 frames of framework noise.

- Settings (⚙) → **Ignore List** → enable **"Automatically add known third-party scripts to ignore list"**.
- Or right-click any stack frame → **"Add script to ignore list"**.

Effect: stepping *skips over* ignored code entirely, and the Call Stack collapses framework frames. Your stack becomes only *your* components. This is the difference between "debugging React is painful" and "debugging React is fine."

**Workspaces (persistent edits).** Sources → **Filesystem** → add your project folder → grant access. Now edits made in the DevTools Sources editor are **written to disk**. Combined with HMR you get a genuine in-browser editing loop with live values. Green dot next to the filename = successfully mapped.

**Snippets.** Sources → Snippets → write a reusable script, `Cmd+Enter` runs it on any page. Great for "log me in as the test user" or "dump the redux store." Beats pasting the same blob into the console forever.

### 2.2 The seven kinds of breakpoint

Most people know one. Here are all of them, with when each wins:

| Type | How | Use it when |
|---|---|---|
| **Line** | Click the line number | Default |
| **Conditional** | Right-click line number → Add conditional breakpoint → `id === 42` | The bug happens on item 400 of a loop |
| **Logpoint** | Right-click → Add logpoint → `'user:', user.id` | **Your `console.log` replacement.** No code edit, no rebuild |
| **DOM** | Elements → right-click node → Break on → subtree modified / attribute modified / node removed | "Something removes this element and I have no idea what" |
| **XHR/fetch** | Sources → XHR/fetch Breakpoints → `+` → `/api/orders` (substring match) | Pause the instant a request fires — the stack shows exactly which component triggered it |
| **Event listener** | Sources → Event Listener Breakpoints → e.g. Mouse → click | Find the handler for a button when you can't grep it |
| **Exception** | Sources → pause icon, tick **"Pause on caught exceptions"** | An error is being swallowed by a `try/catch` somewhere |

Conditional breakpoints also accept side effects — `if (x > 5) console.trace()` inside the condition works, and `false` as the tail means it never actually pauses.

### 2.3 While paused

- **Call Stack pane** — click any frame to time-travel your *view* to that frame's scope. Right-click → **Restart Frame** re-runs the current function from its start without re-running the whole flow. Enormous time-saver.
- **Scope pane** — Local / Closure / Global. The **Closure** section is where stale-closure React bugs are visible: you'll see the captured old value of a state variable, side by side with the current one.
- **Watch pane** — pin expressions like `items.filter(i => !i.valid).length` and see them update as you step.
- **Async stack traces** — on by default. Your stack continues *past* `await` and `setTimeout` boundaries into the code that scheduled the work. This is why "Promise rejection with useless stack" is mostly a solved problem now.
- **Step Into (F11) vs Step Over (F10) vs Step Out (Shift+F11)** — Over is your default; Into only when you actually suspect the callee; Out when you've stepped somewhere you don't care about.

### 2.4 Console commands you're probably not using

```js
$0, $1, $2            // last inspected DOM elements ($0 = current selection in Elements)
$_                    // result of the previous expression
$('.btn'), $$('.btn') // querySelector / querySelectorAll (returns a real Array)
$x('//div[@id="a"]')  // XPath

debug(myFunction)     // breakpoint on entry to that function, without finding its file
undebug(myFunction)

monitor(fn)           // log every call + arguments, without editing the function
monitorEvents(window, 'resize')      // log matching events
unmonitorEvents(window)

getEventListeners($0) // every listener attached to the selected element

queryObjects(MyClass) // every live instance of a class on the heap — great for leak hunting

copy(someBigObject)   // straight to clipboard, no truncated console output
console.table(rows)   // arrays of objects as an actual sortable table
console.dir(el)       // DOM element as a JS object instead of as HTML
console.trace()       // print the stack from here without pausing
console.count('render')            // how many times did this run?
console.time('x'); console.timeEnd('x')
console.group() / groupEnd()
```

**Live Expressions** — the 👁 icon at the top of the Console. Pins an expression that re-evaluates continuously above the log stream. Put `store.getState().cart.items.length` there and watch it change as you click around. It's a always-on dashboard, not a log.

**Preserve log** (Console + Network settings) — keeps output across navigations and reloads. Essential for debugging redirects, OAuth flows, and form submits that reload the page.

### 2.5 Network panel — advanced

- **Filter syntax** in the filter box: `status-code:500`, `method:POST`, `mime-type:application/json`, `larger-than:100k`, `has-response-header:set-cookie`, `domain:api.example.com`. Prefix with `-` to negate: `-.js` hides all scripts.
- **Initiator column** — hover with `Shift` held: the request that caused this one goes green, the requests this one caused go red. Traces dependency chains.
- **Right-click a request →** *Copy as cURL* (paste straight into a terminal or into an issue), *Copy as fetch*, *Replay XHR* (re-fire without redoing the UI flow), *Block request URL* / *Block request domain* (simulate a dead third party or a failing endpoint).
- **Local Overrides** — right-click → *Override content*. Chrome saves the response to a local folder and serves *your* edited version on every subsequent load, including for API JSON. You can reproduce a backend edge case ("what if the API returns `items: null`?") with zero backend changes. This is the most underused feature in DevTools.
- **Throttling** — presets plus custom profiles. Also **Network conditions** (drawer → ⋮ → Network conditions): disable cache, override User-Agent, disable HTTP/2.
- **Reading the Timing tab:** *Queueing* (browser held it back — too many parallel requests), *Stalled* (proxy/connection limits), *Waiting (TTFB)* — **this is your backend's time**, *Content Download* — payload is too big. If TTFB is 2s, stop looking at the frontend and go put a breakpoint in FastAPI.

### 2.6 Panels you didn't know existed (all via Command Menu)

- **Rendering** — *Paint flashing* (green flashes on repaint: find needless re-renders), *Layout Shift Regions* (blue flashes: CLS culprits), *Frame Rendering Stats* (live FPS + GPU memory), *Emulate CSS `prefers-color-scheme`*, *Emulate vision deficiencies*.
- **Coverage** — records which CSS/JS actually executed. Red = shipped but unused. The fastest way to justify a bundle-splitting ticket.
- **Performance** — record an interaction, look at the **Main** thread flame chart. Anything wider than 50ms is a *long task* and is what makes your app feel janky. Use **Bottom-Up** to find the single hottest function, **Call Tree** to find the expensive path. React work shows up under your component names if you've enabled the React DevTools profiler integration.
- **Memory** — take a **Heap snapshot**, do the leaky action, take another, then use the **Comparison** view filtered to *Detached* — detached DOM nodes still referenced by JS are the classic React/Node leak. For allocation over time, use *Allocation instrumentation on timeline*.
- **Application** — Service Workers (*Update on reload*, *Bypass for network* — fixes 90% of "my changes aren't showing"), Storage inspection, and **Clear site data** for a true clean slate.
- **Animations**, **Layers**, **Sensors** (fake geolocation / device orientation), **Issues** (Chrome's own static analysis of your page — CORS, deprecations, cookie problems, explained in prose).

### 2.7 React-specific

Install **React DevTools** (extension) on top of the above:

- **Components tab** — inspect props/state/hooks of any component. Select a component then use `$r` in the Console to poke at it directly.
- Settings → **Highlight updates when components render** — flashing borders show you exactly which components re-render on each interaction. Instant memoization audit.
- **Profiler tab** — record an interaction, see a flame chart of *render* time per component plus "why did this render" (props changed / hooks changed / parent rendered). This tells you the cause, which the Chrome Performance panel does not.

---

## 3. VS Code — one `launch.json` for the whole stack

### 3.1 The two concepts that unlock everything

**Launch vs Attach.**
- **Launch** = VS Code starts the process for you, under the debugger. Simple, but the debugger owns the terminal.
- **Attach** = you start the process yourself (`npm run dev`, `uvicorn ...`) and VS Code connects to it over a debug port. Better for real work: your normal dev server, normal logs, normal hot reload — and you can attach and detach at will.

Rule of thumb: **launch for scripts and tests, attach for dev servers.**

**The JavaScript Debug Terminal.** Command Palette → `Debug: JavaScript Debug Terminal`. Any Node process you start in that terminal is automatically debugged — no config file at all. `npm run dev`, `npx nest start`, `npx tsx watch src/index.ts` — all just work, breakpoints included. **If you only remember one thing about VS Code + Node, remember this.** Everything below is the more explicit, more configurable version of it.

### 3.2 The config file

Create `.vscode/launch.json` at the repo root. Full-stack monorepo version covering all four of your frameworks:

```jsonc
{
  "version": "0.2.0",
  "configurations": [
    // ---------- REACT (browser) ----------
    {
      "name": "React: Chrome (attach to running Vite)",
      "type": "chrome",
      "request": "launch",
      "url": "http://localhost:5173",
      "webRoot": "${workspaceFolder}/frontend",
      // Skip framework noise when stepping — VS Code's Ignore List equivalent
      "skipFiles": ["<node_internals>/**", "**/node_modules/**"],
      // Reuse your everyday Chrome profile (logins, extensions) instead of a blank one:
      "userDataDir": false
    },

    // ---------- EXPRESS ----------
    {
      // You run:  npx tsx watch --inspect src/server.ts
      "name": "Express: attach",
      "type": "node",
      "request": "attach",
      "port": 9229,
      "restart": true,               // reattach after nodemon/tsx restarts
      "skipFiles": ["<node_internals>/**", "**/node_modules/**"],
      "sourceMaps": true
    },
    {
      // Or let VS Code start it (no separate terminal)
      "name": "Express: launch",
      "type": "node",
      "request": "launch",
      "runtimeExecutable": "npx",
      "runtimeArgs": ["tsx", "src/server.ts"],
      "cwd": "${workspaceFolder}/api",
      "console": "integratedTerminal",
      "skipFiles": ["<node_internals>/**", "**/node_modules/**"]
    },

    // ---------- NESTJS ----------
    {
      // You run:  nest start --debug --watch     (opens inspector on 9229)
      "name": "Nest: attach",
      "type": "node",
      "request": "attach",
      "port": 9229,
      "restart": true,
      "autoAttachChildProcesses": true,
      "skipFiles": ["<node_internals>/**", "**/node_modules/**"]
    },
    {
      "name": "Nest: debug a single e2e/unit test",
      "type": "node",
      "request": "launch",
      "runtimeExecutable": "npx",
      "runtimeArgs": ["jest", "--runInBand", "--testPathPattern", "${fileBasenameNoExtension}"],
      "console": "integratedTerminal",
      "skipFiles": ["<node_internals>/**", "**/node_modules/**"]
    },

    // ---------- FASTAPI ----------
    {
      "name": "FastAPI: launch (uvicorn, no reload)",
      "type": "debugpy",
      "request": "launch",
      "module": "uvicorn",
      "args": ["app.main:app", "--host", "127.0.0.1", "--port", "8000"],
      "cwd": "${workspaceFolder}/backend",
      "jinja": true,
      "justMyCode": false            // lets you step into pydantic/sqlalchemy/starlette
    },
    {
      // You run:  python -m debugpy --listen 5678 --wait-for-client -m uvicorn app.main:app
      "name": "FastAPI: attach",
      "type": "debugpy",
      "request": "attach",
      "connect": { "host": "127.0.0.1", "port": 5678 },
      "justMyCode": false,
      "pathMappings": [
        { "localRoot": "${workspaceFolder}/backend", "remoteRoot": "." }
      ]
    },
    {
      "name": "Pytest: current file",
      "type": "debugpy",
      "request": "launch",
      "module": "pytest",
      "args": ["-x", "-s", "${file}"],
      "cwd": "${workspaceFolder}/backend",
      "justMyCode": false
    }
  ],

  // ---------- Start the whole stack under one debugger ----------
  "compounds": [
    {
      "name": "🚀 Full stack",
      "configurations": ["React: Chrome (attach to running Vite)", "Nest: attach", "FastAPI: launch (uvicorn, no reload)"],
      "stopAll": true
    }
  ]
}
```

With a compound running, a single click steps you from a React click handler → into the Nest controller → and (with a second breakpoint) into the FastAPI route. Same Call Stack UI throughout.

### 3.3 The `--reload` trap in FastAPI (this will bite you)

`uvicorn --reload` runs your app in a **child process**. The debugger attaches to the parent, so your breakpoints silently never hit and you conclude "the debugger is broken."

Three fixes, best first:
1. **Drop `--reload` while debugging.** You're pausing anyway; you don't need hot reload for the 3 minutes you're at a breakpoint. This is what the launch config above does.
2. Keep reload and rely on debugpy following subprocesses (`"subProcess": true` is the default) — works, but flaky with some reloaders.
3. Run uvicorn programmatically in a `main.py` with `reload=False` and debug that entrypoint.

The same trap exists for Gunicorn workers and Celery — always ask "is my code running in the process I attached to?"

### 3.4 VS Code features worth knowing

- **Conditional breakpoint / Hit count / Logpoint** — right-click the gutter. Hit count `>50` is perfect for "fails after a while." Logpoints use `{expression}` braces: `user {user.id} has {user.roles.length} roles`.
- **Breakpoints pane → Caught Exceptions / Uncaught Exceptions** checkboxes. Turning on *Caught* is the fastest way to find silently swallowed errors. Python has the same, plus "Raised Exceptions."
- **Debug Console** — full REPL in the paused frame. Type expressions, call functions, mutate state.
- **Watch pane** — same as Chrome's.
- **Inline values** — variable values render greyed-out at the end of each line while paused. Turn on with `"debug.inlineValues": "on"` in settings.
- **`justMyCode: false`** (Python) and removing `**/node_modules/**` from `skipFiles` (Node) — flip these when the bug is *in* the library, or when you want to read how a library actually behaves rather than how its docs claim it does.
- **Data breakpoints / `debugger;` statement** — a bare `debugger;` in JS/TS pauses in whichever debugger is attached (Chrome *or* VS Code). Sometimes the fastest way in.
- **Restart Frame** — right-click a stack frame, same as Chrome.
- **`"restart": true`** on attach configs so the debugger reconnects when nodemon/tsx restarts your server.

### 3.5 Making Node source maps actually work

If your breakpoints show as hollow grey circles ("unbound"), it's almost always source maps:

- Using `tsx` / `ts-node`: works out of the box.
- Compiling with `tsc`: set `"sourceMap": true` in `tsconfig.json`, and make sure you're running `dist/` while breakpointing `src/` — VS Code maps them via the `.js.map` files. If it can't, set `outFiles: ["${workspaceFolder}/dist/**/*.js"]` in the launch config.
- NestJS: `nest build` emits source maps by default; check `dist/*.js.map` exists.

---

## 4. The debuggers you haven't met yet

### 4.1 Node.js — the built-in inspector

Every Node process can be debugged without VS Code:

```bash
node --inspect server.js          # inspector on 127.0.0.1:9229, runs immediately
node --inspect-brk server.js      # pauses on line 1 — use for startup/import bugs
NODE_OPTIONS='--inspect' npm run dev   # inject into any npm script you don't control
```

Then open **`chrome://inspect`** in Chrome → "inspect" under Remote Target. You now get the **full Chrome DevTools UI attached to your Node backend** — same Sources panel, same breakpoints, plus Memory heap snapshots and the Performance profiler for server-side code. Most JS developers never learn this and it's the single best Node debugging tool.

Also built in, no tooling needed:

```bash
node --cpu-prof server.js     # writes a .cpuprofile → drag into DevTools Performance panel
node --heap-prof server.js    # heap allocation profile, same idea
node --trace-warnings app.js  # get stacks for those "unhandled rejection" warnings
node --stack-trace-limit=50 app.js
node inspect server.js        # terminal-only debugger, no GUI (works over SSH)
```

For deeper Node profiling: **clinic.js** (`clinic doctor -- node server.js` — diagnoses event-loop blocking, GC pressure, I/O bottlenecks in a browser report) and **0x** (flame graphs).

### 4.2 Python — `pdb` and friends

**The zero-install entry point.** Put `breakpoint()` anywhere in your Python code and run normally. You get an interactive `pdb` prompt in the terminal, paused at that line, with a full REPL.

```bash
PYTHONBREAKPOINT=ipdb.set_trace   # upgrade every breakpoint() to ipdb (tab-complete, colors)
PYTHONBREAKPOINT=0                # disable all breakpoint() calls — e.g. in CI
```

**pdb commands worth memorising** (these map 1:1 onto the DevTools buttons):

| Command | Meaning |
|---|---|
| `n` / `s` / `r` / `c` | next (step over) / step into / run to return / continue |
| `w` (`where`) | print the call stack |
| `u` / `d` | move up / down a stack frame — this is "click a frame" in a GUI |
| `l` / `ll` | list source around here / the whole function |
| `p x` / `pp x` | print / pretty-print an expression |
| `a` | print the current function's arguments |
| `b file.py:42` / `b` / `cl` | set breakpoint / list them / clear |
| `b 42, qty == 0` | **conditional** breakpoint |
| `display expr` | auto-print expr every time execution stops (a watch) |
| `interact` | drop into a full Python REPL in this frame |
| `j 38` | **jump** to another line — re-run a block without restarting |
| `q` | quit |

**Post-mortem debugging — the killer feature.** When something has already crashed, you don't need to reproduce it to inspect it:

```bash
python -m pdb -c continue myscript.py   # on crash, drops into pdb at the failing frame
```
```python
import pdb; pdb.pm()    # in a REPL, right after an exception: inspect the dead frame
```

**pytest integration:**
```bash
pytest --pdb          # drop into pdb at the first failure, in the failing frame
pytest --trace        # step through a test from its first line
pytest -x -s --lf     # stop at first failure, allow stdin, rerun only last-failed
pytest --pdbcls=IPython.terminal.debugger:TerminalPdb   # nicer REPL
```

**Remote / already-running processes:**
```bash
python -m debugpy --listen 5678 --wait-for-client -m uvicorn app.main:app
python -m pdb -p <PID>     # Python 3.14+: attach to a live process, no prior setup
```

### 4.3 Profilers — for "it's slow," not "it's wrong"

Debuggers answer *why is this value wrong*. Profilers answer *where did the time go*. Different tools; people reach for the wrong one constantly.

- **py-spy** — the one to install today. It attaches to an **already-running** Python process with zero code changes and zero restart, in production if you like:
  ```bash
  py-spy top --pid 12345              # htop, but for Python functions
  py-spy dump --pid 12345             # instant stack trace of a hung process ← use this on any hang
  py-spy record -o flame.svg --pid 12345
  ```
  `py-spy dump` on a stuck FastAPI worker tells you in 2 seconds what it's blocked on. Nothing else comes close.
- **memray** (Python memory), **scalene** (CPU + memory + GPU, line-level).
- **`PYTHONASYNCIODEBUG=1`** — warns about un-awaited coroutines and slow callbacks blocking the event loop. Turn it on when your FastAPI app is mysteriously serial.
- **Node:** `clinic.js`, `--cpu-prof`, or the Memory/Performance panels via `chrome://inspect`.

### 4.4 Debugging the wire, not the code

- **`curl -v`** / **HTTPie** (`http -v POST :8000/orders qty=0`) — takes the browser out of the equation entirely. If curl works and the app doesn't, it's a header, cookie, or CORS problem.
- **mitmproxy** — a terminal proxy that shows, edits, and replays every request between any two services. Invaluable when the traffic isn't from a browser (mobile app, service-to-service, webhooks).
- **DevTools → Copy as cURL** → paste into a terminal → edit and replay. Bridges the two worlds.
- **SQL:** turn on `echo=True` on the SQLAlchemy engine (or the equivalent logger) to see every emitted query. Most "the API is slow" bugs are N+1 queries, and this makes them obvious instantly.

### 4.5 Production-tier tools (for later)

- **Structured logging** — `pino` (Node) / `structlog` (Python) with a request ID threaded through every log line. Logs are the debugger you have when you can't attach one.
- **OpenTelemetry** — distributed tracing. Once you have three services, "which one was slow" needs a trace, not a breakpoint.
- **Sentry** — captures the stack, the local variables, and the breadcrumbs of errors you'll never reproduce locally.
- **Time-travel debuggers** — `rr` (Linux, record-and-replay: step *backwards*) and Replay.io (same idea for browsers). When you eventually meet a bug that only happens once, these are the answer.

---

## 5. Symptom → tool lookup

| Symptom | Reach for |
|---|---|
| Wrong value on screen | React DevTools Components → then a breakpoint in the render |
| Component re-renders too much | React Profiler ("why did this render") + Rendering → Paint flashing |
| Request never fires | DevTools Network (is it there?) → then an XHR/fetch breakpoint |
| 4xx/5xx from the API | Network → Payload/Response → then a breakpoint on the first line of the handler |
| Request is right, response is wrong | Backend breakpoint at the *serialization* step, then walk up the stack |
| Error swallowed somewhere | Pause on **caught** exceptions (both Chrome and VS Code) |
| Element disappears/changes mysteriously | DOM breakpoint → subtree modified |
| "It works on refresh but not on navigate" | Preserve log + check Service Worker (Bypass for network) |
| Backend hangs, no logs | `py-spy dump --pid` (Python) / `kill -QUIT` or `chrome://inspect` (Node) |
| Slow endpoint | Network Timing → TTFB → then SQL echo, then py-spy record |
| Janky UI | Performance panel → Main thread → tasks >50ms |
| Memory grows over time | Heap snapshot comparison → filter Detached; `queryObjects(Ctor)` |
| Only happens in prod | Sentry + structured logs + Local Overrides to replay the bad payload locally |
| Only happens for one user | Conditional breakpoint on their ID; Local Overrides to fake their API response |

---

## 6. Your ramp-off-`console.log` ladder

Don't try to adopt all of this at once. In order, one per few days:

1. **Logpoints instead of `console.log`.** Same information, no code edits. (Chrome: right-click the line number. VS Code: right-click the gutter.)
2. **Turn on the Ignore List / `skipFiles`.** Without it, stepping through React or Nest is miserable and you'll give up.
3. **One breakpoint + read the Scope pane** instead of five logs. Notice how often the bug is a variable you wouldn't have logged.
4. **Use the Call Stack** — click frames, walk up to where the value was still correct.
5. **Evaluate hypotheses in the Debug Console** rather than editing + reloading.
6. **Conditional breakpoints** — the moment you write `if (x === 42) console.log(...)`, you've reinvented one badly.
7. **`chrome://inspect` on a Node process** — realise your backend has the same tooling as your frontend.
8. **`breakpoint()` + `pytest --pdb`** in Python — post-mortem debugging on a failing test is the fastest loop in the whole stack.
9. **Pause on caught exceptions** when you're stuck. It finds the bugs the other eight steps miss.
10. **`py-spy dump`** the first time something hangs. You'll never `print()` your way through a hang again.

The end state: you open a bug you've never seen, in a file you've never read, and find it in five minutes without adding a single line of code. That's entirely a tooling skill, and it's a couple of weeks of deliberate practice away.
