# Deep JavaScript — 40LPA Level
*V8 Internals · JIT Compilation · Memory · Event Loop Micro-Details · Advanced Patterns*

---

# Chapter 1: V8 Engine Internals

## How JavaScript Actually Runs

```
Source Code ("let x = 5 + 3")
      │
      ▼
┌─────────────┐
│   Parser     │ → Tokens → AST (Abstract Syntax Tree)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Ignition    │ → Bytecode (interpreted, fast startup)
│ (Interpreter)│    Every function starts here
└──────┬──────┘
       │  (function called many times? → "hot")
       ▼
┌─────────────┐
│ TurboFan     │ → Optimized Machine Code (JIT compiled, fast execution)
│ (Compiler)   │    Only for hot functions
└──────┬──────┘
       │  (assumptions broken? → deoptimize)
       ▼
  Back to Ignition (re-interpret with new assumptions)
```

**Why this matters in interviews:** V8 doesn't compile everything upfront. Cold functions run as bytecode (slow but fast to start). Hot functions get JIT-compiled to machine code (fast execution). If V8's assumptions about types are wrong, it deoptimizes — throws away the compiled code and falls back to bytecode.

## Hidden Classes (Shapes/Maps)

V8 assigns a "hidden class" (called a Map internally) to every object. Objects with the same property names, added in the same order, share a hidden class. This enables fast property access (offset-based, like C structs).

```javascript
// FAST — same hidden class (properties added in same order)
function Point(x, y) {
  this.x = x;    // hidden class transition: {} → {x}
  this.y = y;    // hidden class transition: {x} → {x, y}
}

const p1 = new Point(1, 2);   // hidden class C0 → C1 → C2
const p2 = new Point(3, 4);   // reuses C0 → C1 → C2 (same path!)
// V8 knows: x is at offset 0, y is at offset 4. Direct memory access. O(1).

// SLOW — different hidden classes (different property order)
const obj1 = {};
obj1.x = 1;
obj1.y = 2;     // hidden class: {x, y}

const obj2 = {};
obj2.y = 2;     // DIFFERENT hidden class: {y}
obj2.x = 1;     // hidden class: {y, x} — NOT the same as {x, y}!

// V8 can't optimize: different shapes → falls back to hash table lookup. Slower.
```

**Interview answer:** "V8 uses hidden classes to turn dynamic objects into struct-like memory layouts. Objects with the same properties in the same order share a hidden class, enabling offset-based access instead of hash lookups. Adding properties in inconsistent order creates different hidden classes and kills this optimization."

## Inline Caching (IC)

When you access `obj.x` in a loop, V8 remembers which hidden class `obj` had last time and caches the property offset. Next iteration, it checks "same hidden class?" → yes → use cached offset (one instruction). This is called a **monomorphic** IC.

```javascript
// MONOMORPHIC (fast) — always same shape
function getX(point) {
  return point.x;    // IC: "last time point had hidden class C2, x was at offset 0"
}

for (const p of points) {   // all same shape
  getX(p);   // monomorphic IC → one check, direct access
}

// POLYMORPHIC (slower) — 2-4 different shapes
// V8 maintains a small lookup table of shapes

// MEGAMORPHIC (slowest) — too many shapes
// V8 gives up caching, falls back to dictionary lookup
function getX(thing) {
  return thing.x;    // called with 10 different object shapes
}
// IC becomes megamorphic → hash table lookup every time
```

**Optimization rule:** Keep object shapes consistent. Don't add/delete properties dynamically. Initialize all properties in the constructor.

## JIT Compilation and Deoptimization

```javascript
function add(a, b) {
  return a + b;
}

// First 100 calls: add(1, 2), add(3, 4), ...
// V8 observes: a and b are always integers
// TurboFan compiles: optimized machine code for integer addition (single CPU instruction)

// Call 101: add("hello", "world")
// V8: "Wait, these are strings now! My assumption was wrong."
// DEOPTIMIZATION: throw away machine code, fall back to bytecode
// Re-profile, maybe re-compile with string support

// This is why TypeScript/consistent types help performance:
// V8 can make stronger assumptions → better optimizations → fewer deoptimizations
```

**Interview answer:** "V8's TurboFan JIT compiler optimizes based on type feedback — it assumes types won't change. When they do, it deoptimizes: throws away the machine code and falls back to Ignition bytecode. Monomorphic functions (always called with the same types) get the best optimizations."

---

# Chapter 2: Event Loop — The Micro-Details

## The Complete Event Loop Model

```
┌─────────────────────────────────────────────────────┐
│                    CALL STACK                        │
│                (executes sync code)                  │
└────────────────────┬────────────────────────────────┘
                     │  (stack empty? check queues)
                     ▼
         ┌───── MICROTASK QUEUE ─────┐
         │  1. process.nextTick()    │  ← highest priority
         │  2. Promise callbacks     │
         │  3. queueMicrotask()      │
         │  (drain ALL before ANY    │
         │   macrotask)              │
         └───────────┬───────────────┘
                     │  (all microtasks done? next phase)
                     ▼
┌────────────────────────────────────────────────────┐
│                   EVENT LOOP PHASES                 │
│                                                    │
│  ┌── Timers ───────────────────────────────────┐   │
│  │  setTimeout(), setInterval() callbacks      │   │
│  │  (runs callbacks whose timer has expired)   │   │
│  └─────────────────────────────────────────────┘   │
│  ↓ drain microtasks                                │
│  ┌── Pending Callbacks ───────────────────────┐   │
│  │  System-level callbacks (TCP errors, etc.)  │   │
│  └─────────────────────────────────────────────┘   │
│  ↓ drain microtasks                                │
│  ┌── Poll ─────────────────────────────────────┐   │
│  │  Retrieve new I/O events                    │   │
│  │  Execute I/O callbacks (fs, network)        │   │
│  │  BLOCKS here if nothing else is scheduled   │   │
│  └─────────────────────────────────────────────┘   │
│  ↓ drain microtasks                                │
│  ┌── Check ────────────────────────────────────┐   │
│  │  setImmediate() callbacks                   │   │
│  └─────────────────────────────────────────────┘   │
│  ↓ drain microtasks                                │
│  ┌── Close ────────────────────────────────────┐   │
│  │  socket.on('close') callbacks               │   │
│  └─────────────────────────────────────────────┘   │
│  ↓ drain microtasks → loop back to Timers          │
└────────────────────────────────────────────────────┘

KEY INSIGHT: Microtasks are drained BETWEEN every phase, not just at the end.
```

## process.nextTick vs Promise vs queueMicrotask vs setTimeout vs setImmediate

```javascript
console.log('1 - start');

setTimeout(() => console.log('2 - setTimeout'), 0);

setImmediate(() => console.log('3 - setImmediate'));

Promise.resolve().then(() => console.log('4 - promise'));

process.nextTick(() => console.log('5 - nextTick'));

queueMicrotask(() => console.log('6 - queueMicrotask'));

console.log('7 - end');

// Output (Node.js):
// 1 - start
// 7 - end
// 5 - nextTick         ← nextTick queue (highest micro priority)
// 4 - promise          ← promise microtask queue
// 6 - queueMicrotask   ← same as promise queue
// 2 - setTimeout       ← timer phase (macrotask)
// 3 - setImmediate     ← check phase (macrotask)

// Priority: sync > nextTick > promises/queueMicrotask > setTimeout > setImmediate
```

## Microtask Starvation

```javascript
// DANGER: microtasks can starve the event loop
function recursive() {
  Promise.resolve().then(recursive);   // infinite microtask chain
}
recursive();
// setTimeout/setImmediate/I/O callbacks NEVER run
// Because microtasks are drained before moving to the next phase
// The microtask queue is never empty → event loop is stuck

// process.nextTick has the same problem:
function bad() {
  process.nextTick(bad);   // starves everything
}
```

## setTimeout(fn, 0) Is NOT Actually 0ms

```javascript
// Minimum delay is ~1ms (Node.js clamps to 1)
// Browser clamps to 4ms after 5 nested setTimeout calls
// Also: setTimeout fires in the Timer phase, which runs AFTER microtasks
// So the actual delay is: time to drain microtasks + timer check interval

const start = Date.now();
setTimeout(() => {
  console.log(Date.now() - start);  // usually 1-4ms, not 0
}, 0);
```

---

# Chapter 3: Memory Management Deep Dive

## V8 Heap Structure

```
┌─────────────────────────────────────────────────────┐
│                    V8 HEAP                           │
│                                                     │
│  ┌─── New Space (Young Generation) ───────────────┐ │
│  │  Semi-space A (From)  │  Semi-space B (To)     │ │
│  │  1-8 MB each          │  Empty (swap target)   │ │
│  │  Objects born here    │                        │ │
│  │  GC: Scavenge         │  Minor GC: copy alive  │ │
│  │  (very fast, ~1ms)    │  objects from A→B,     │ │
│  │                       │  then swap A and B     │ │
│  └───────────────────────┴────────────────────────┘ │
│                                                     │
│  ┌─── Old Space (Old Generation) ────────────────┐  │
│  │  Objects that survived 2 scavenges             │  │
│  │  Default limit: ~1.5 GB (64-bit)               │  │
│  │  GC: Mark-Sweep-Compact (slower, 10-100ms)     │  │
│  │                                                │  │
│  │  ┌── Old Pointer Space (objects with refs) ──┐ │  │
│  │  ├── Old Data Space (strings, numbers)       │ │  │
│  │  ├── Code Space (JIT compiled code)          │ │  │
│  │  ├── Map Space (hidden classes)              │ │  │
│  │  └── Large Object Space (> 512KB)            │ │  │
│  └────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

## Garbage Collection Algorithms

```
SCAVENGE (Minor GC — Young Generation):
  Copying collector. Copies live objects from "from" space to "to" space.
  Dead objects are just forgotten (no explicit deallocation).
  Very fast (~1-5ms) because young generation is small.
  Objects that survive 2 scavenges are promoted to old generation.

MARK-SWEEP-COMPACT (Major GC — Old Generation):
  Phase 1 - Mark: start from roots (global, stack), follow all references,
            mark every reachable object
  Phase 2 - Sweep: free memory of unmarked objects
  Phase 3 - Compact: move surviving objects together to eliminate fragmentation
  Slower (10-100ms) but runs less frequently.

INCREMENTAL MARKING:
  Instead of marking everything at once (long pause), do it in small increments
  interleaved with JS execution. Reduces max pause time.

CONCURRENT MARKING (V8):
  Background threads do the marking while JS continues running on the main thread.
  Only needs a short pause at the end to finalize.
```

## Common Memory Leaks and Detection

```javascript
// LEAK 1: Accidental globals
function process() {
  result = computeHeavy();    // no let/const → global variable → never GC'd
}

// LEAK 2: Forgotten timers
const data = loadHugeData();
setInterval(() => {
  // This closure keeps `data` alive forever
  sendMetrics(data.summary);
}, 5000);
// Fix: clearInterval when no longer needed

// LEAK 3: Detached DOM nodes
const elements = [];
function addElement() {
  const div = document.createElement('div');
  document.body.appendChild(div);
  elements.push(div);        // reference in array
  document.body.removeChild(div);  // removed from DOM but array keeps it alive
}

// LEAK 4: Closures retaining large scopes
function createHandler() {
  const hugeArray = new Array(1000000).fill('x');
  return function handler() {
    // handler doesn't use hugeArray, but the closure retains it
    console.log('handling');
  };
}

// Detection:
// process.memoryUsage() — rss, heapUsed, heapTotal, external
// --inspect + Chrome DevTools Heap Snapshots
// clinic.js, 0x (flame graphs)
```

---

# Chapter 4: Advanced Patterns for 40LPA Interviews

## Metaclass Pattern: Proxy-Based Validation

```javascript
function createValidatedObject(schema) {
  return new Proxy({}, {
    set(target, prop, value) {
      if (schema[prop]) {
        const { type, required, min, max } = schema[prop];
        if (type && typeof value !== type)
          throw new TypeError(`${prop} must be ${type}, got ${typeof value}`);
        if (min !== undefined && value < min)
          throw new RangeError(`${prop} must be >= ${min}`);
        if (max !== undefined && value > max)
          throw new RangeError(`${prop} must be <= ${max}`);
      }
      target[prop] = value;
      return true;
    },
    get(target, prop) {
      if (!(prop in target) && schema[prop]?.required)
        throw new Error(`${prop} is required but not set`);
      return target[prop];
    }
  });
}

const user = createValidatedObject({
  age: { type: 'number', min: 0, max: 150, required: true },
  name: { type: 'string', required: true },
});

user.age = 25;      // OK
user.age = -5;      // RangeError: age must be >= 0
user.age = "old";   // TypeError: age must be number
```

## Event Emitter from Scratch

```javascript
class EventEmitter {
  #listeners = new Map();

  on(event, callback) {
    if (!this.#listeners.has(event)) this.#listeners.set(event, []);
    this.#listeners.get(event).push(callback);
    return this;    // chainable
  }

  once(event, callback) {
    const wrapper = (...args) => {
      callback(...args);
      this.off(event, wrapper);
    };
    wrapper._original = callback;
    return this.on(event, wrapper);
  }

  off(event, callback) {
    const cbs = this.#listeners.get(event);
    if (!cbs) return this;
    this.#listeners.set(event, cbs.filter(
      cb => cb !== callback && cb._original !== callback
    ));
    return this;
  }

  emit(event, ...args) {
    const cbs = this.#listeners.get(event);
    if (!cbs) return false;
    cbs.forEach(cb => cb(...args));
    return true;
  }

  listenerCount(event) {
    return this.#listeners.get(event)?.length ?? 0;
  }
}
```

## Promise Implementation (Core Logic)

```javascript
class MyPromise {
  #state = 'pending';    // pending | fulfilled | rejected
  #value = undefined;
  #callbacks = [];

  constructor(executor) {
    const resolve = (value) => {
      if (this.#state !== 'pending') return;   // can only resolve once
      this.#state = 'fulfilled';
      this.#value = value;
      this.#callbacks.forEach(cb => cb.onFulfilled(value));
    };

    const reject = (reason) => {
      if (this.#state !== 'pending') return;
      this.#state = 'rejected';
      this.#value = reason;
      this.#callbacks.forEach(cb => cb.onRejected(reason));
    };

    try {
      executor(resolve, reject);
    } catch (err) {
      reject(err);
    }
  }

  then(onFulfilled, onRejected) {
    return new MyPromise((resolve, reject) => {
      const handle = () => {
        try {
          if (this.#state === 'fulfilled') {
            const result = onFulfilled ? onFulfilled(this.#value) : this.#value;
            resolve(result);
          } else if (this.#state === 'rejected') {
            if (onRejected) {
              resolve(onRejected(this.#value));
            } else {
              reject(this.#value);
            }
          }
        } catch (err) {
          reject(err);
        }
      };

      if (this.#state === 'pending') {
        this.#callbacks.push({
          onFulfilled: () => queueMicrotask(handle),
          onRejected: () => queueMicrotask(handle),
        });
      } else {
        queueMicrotask(handle);
      }
    });
  }

  catch(onRejected) { return this.then(null, onRejected); }

  static resolve(value) { return new MyPromise(r => r(value)); }
  static reject(reason) { return new MyPromise((_, r) => r(reason)); }
}
```

## Debounce and Throttle (Implement from Scratch)

```javascript
// Debounce: wait until user STOPS doing something for N ms
function debounce(fn, delay) {
  let timer;
  return function (...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}
// Use: search input — don't fire API call on every keystroke

// Throttle: execute at most once per N ms
function throttle(fn, interval) {
  let lastTime = 0;
  return function (...args) {
    const now = Date.now();
    if (now - lastTime >= interval) {
      lastTime = now;
      fn.apply(this, args);
    }
  };
}
// Use: scroll handler — don't fire 60 times per second

// Advanced debounce with leading/trailing options
function debounceAdvanced(fn, delay, { leading = false, trailing = true } = {}) {
  let timer;
  let lastArgs;

  return function (...args) {
    const isFirstCall = !timer;
    lastArgs = args;

    clearTimeout(timer);

    if (leading && isFirstCall) {
      fn.apply(this, args);
    }

    timer = setTimeout(() => {
      if (trailing && lastArgs) {
        fn.apply(this, lastArgs);
      }
      timer = null;
      lastArgs = null;
    }, delay);
  };
}
```

## Deep Clone (Handle Everything)

```javascript
function deepClone(value, seen = new WeakMap()) {
  // Primitives
  if (value === null || typeof value !== 'object') return value;

  // Circular reference detection
  if (seen.has(value)) return seen.get(value);

  // Date
  if (value instanceof Date) return new Date(value.getTime());

  // RegExp
  if (value instanceof RegExp) return new RegExp(value.source, value.flags);

  // Map
  if (value instanceof Map) {
    const clone = new Map();
    seen.set(value, clone);
    value.forEach((v, k) => clone.set(deepClone(k, seen), deepClone(v, seen)));
    return clone;
  }

  // Set
  if (value instanceof Set) {
    const clone = new Set();
    seen.set(value, clone);
    value.forEach(v => clone.add(deepClone(v, seen)));
    return clone;
  }

  // Array or Object
  const clone = Array.isArray(value) ? [] : {};
  seen.set(value, clone);

  for (const key of Reflect.ownKeys(value)) {     // includes symbols
    clone[key] = deepClone(value[key], seen);
  }

  return clone;
}
```

## Currying and Partial Application

```javascript
// Currying: transform f(a, b, c) into f(a)(b)(c)
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    }
    return (...moreArgs) => curried(...args, ...moreArgs);
  };
}

const add = curry((a, b, c) => a + b + c);
add(1)(2)(3);      // 6
add(1, 2)(3);      // 6
add(1)(2, 3);      // 6
add(1, 2, 3);      // 6

// Practical use: configuration factories
const logger = curry((level, prefix, message) =>
  console.log(`[${level}] ${prefix}: ${message}`)
);

const errorLog = logger('ERROR');
const apiErrorLog = errorLog('API');
apiErrorLog('Connection timeout');   // "[ERROR] API: Connection timeout"
```

## Async Patterns: Concurrency Control

```javascript
// Process items with a concurrency limit (like a thread pool)
async function asyncPool(limit, items, fn) {
  const results = [];
  const executing = new Set();

  for (const item of items) {
    const promise = fn(item).then(result => {
      executing.delete(promise);
      return result;
    });
    results.push(promise);
    executing.add(promise);

    if (executing.size >= limit) {
      await Promise.race(executing);   // wait for one to finish before adding more
    }
  }

  return Promise.all(results);
}

// Usage: embed 1000 documents, max 5 concurrent API calls
await asyncPool(5, documents, async (doc) => {
  return await embeddingAPI.embed(doc.text);
});
```

---

# Chapter 5: 🧩 40LPA Interview Questions

**Q: Explain how V8 optimizes property access.**
A: V8 uses hidden classes (Shapes/Maps). Objects with the same properties in the same order share a hidden class, which stores property offsets. This turns hash table lookups into direct memory offset access. Additionally, V8 uses inline caching at call sites — it remembers which hidden class an object had last time and skips the lookup. This breaks if you use objects with inconsistent shapes (polymorphic/megamorphic call sites).

**Q: What's the difference between process.nextTick and queueMicrotask?**
A: Both are microtasks, but `process.nextTick` has its own queue that drains BEFORE the Promise microtask queue. `queueMicrotask` goes into the same queue as Promise callbacks. In practice: `nextTick` runs first. Both can starve the event loop if used recursively. `queueMicrotask` is the standard (works in browsers too), `nextTick` is Node-specific.

**Q: How does V8 garbage collection work?**
A: Two generations. Young generation uses Scavenge (copying GC) — objects are allocated in "from" space, live objects are copied to "to" space, spaces swap. Fast (~1ms) because most objects die young. Objects surviving 2 scavenges are promoted to old generation, which uses Mark-Sweep-Compact — mark reachable objects from roots, sweep unmarked, compact to reduce fragmentation. V8 uses concurrent and incremental marking to minimize pause times.

**Q: Write a Promise.all from scratch.**
```javascript
function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    const results = [];
    let completed = 0;

    if (promises.length === 0) return resolve([]);

    promises.forEach((promise, index) => {
      Promise.resolve(promise).then(
        value => {
          results[index] = value;
          completed++;
          if (completed === promises.length) resolve(results);
        },
        reject    // first rejection rejects the whole thing
      );
    });
  });
}
```

**Q: What causes deoptimization in V8?**
A: Changing the type of a variable (number → string), accessing properties that weren't in the hidden class when TurboFan compiled, using `arguments` object in optimized functions, using `try/catch` around hot code (though modern V8 handles this better), using `eval` or `with`, and passing different object shapes to the same function (megamorphic call sites).

**Q: Explain WeakRef and FinalizationRegistry use cases.**
A: `WeakRef` holds a reference that doesn't prevent garbage collection — useful for caches where you want entries to be automatically evicted when memory is tight. `FinalizationRegistry` runs a callback after an object is garbage collected — useful for cleaning up external resources (closing file handles, deregistering from a server). Both are escape hatches, not everyday tools.
