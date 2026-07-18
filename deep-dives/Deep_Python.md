# Deep Python — 40LPA Level
*CPython Internals · GIL · Memory · Descriptors · Metaclasses · Async Internals*

---

# Chapter 1: CPython Object Model

## Everything Is a PyObject

Every Python value — integers, strings, functions, classes, modules — is a C struct called `PyObject` on the heap. Even `42` is a heap-allocated object with a reference count and a type pointer.

```
Every PyObject has:
  ┌──────────────────────┐
  │ ob_refcnt (int)      │  ← reference count for GC
  │ ob_type  (pointer)   │  ← pointer to the type object (int, str, list, etc.)
  │ ... type-specific    │  ← actual data (value, length, etc.)
  └──────────────────────┘

For an integer 42:
  PyLongObject:
    ob_refcnt: 3          (3 variables point to this object)
    ob_type: → int        (this is an integer)
    ob_digit: [42]        (the actual value)
```

## id(), is, and Memory Layout

```python
a = 42
b = 42
print(id(a) == id(b))    # True — CPython caches integers -5 to 256 (same object)
print(a is b)             # True

a = 257
b = 257
print(a is b)             # False in REPL, possibly True in a .py file
# CPython may optimize within the same compilation unit

# CRITICAL: `is` checks identity (same object in memory)
#           `==` checks value equality (__eq__)
# RULE: Only use `is` for None, True, False. Use == for everything else.
```

## How Python Finds Attributes (MRO + Descriptor Protocol)

When you write `obj.attr`, Python's lookup order is:

```
1. type(obj).__mro__  → search for a DATA DESCRIPTOR on the class
   (data descriptor = defines __get__ AND __set__)

2. obj.__dict__       → search the instance dictionary

3. type(obj).__mro__  → search for a NON-DATA DESCRIPTOR on the class
   (non-data descriptor = defines only __get__)

4. Raise AttributeError

This is why @property (data descriptor) overrides instance attributes,
but regular methods (non-data descriptors) don't.
```

```python
class Descriptor:
    def __get__(self, obj, objtype=None):
        print(f"__get__ called, obj={obj}")
        return 42

    def __set__(self, obj, value):
        print(f"__set__ called, value={value}")

class MyClass:
    attr = Descriptor()    # data descriptor (has both __get__ and __set__)

obj = MyClass()
obj.attr                   # calls Descriptor.__get__ → 42
obj.attr = 99              # calls Descriptor.__set__ (doesn't create instance attr!)
obj.__dict__               # {} — empty! The descriptor intercepted the set.

# @property is implemented as a descriptor:
class Property:
    def __init__(self, fget):
        self.fget = fget
    def __get__(self, obj, objtype=None):
        if obj is None: return self
        return self.fget(obj)
    def __set__(self, obj, value):
        raise AttributeError("can't set")
```

---

# Chapter 2: The GIL — Deep Dive

## What It Actually Is

The GIL (Global Interpreter Lock) is a mutex in CPython that allows only ONE thread to execute Python bytecode at any given time. Even on a 16-core CPU, only 1 core runs Python code.

```
WITH GIL (CPython):

Thread 1         Thread 2         Thread 3
────────         ────────         ────────
[RUNNING]        [BLOCKED]        [BLOCKED]
  ↓ release GIL
[BLOCKED]        [RUNNING]        [BLOCKED]
  ↓ release GIL
[BLOCKED]        [BLOCKED]        [RUNNING]

Only ONE thread executes Python at a time.
Other threads wait for the GIL.

BUT: GIL is released during I/O operations!

Thread 1 (CPU)   Thread 2 (I/O)   Thread 3 (I/O)
──────────────   ──────────────   ──────────────
[RUNNING]        [reading file]   [HTTP request]
                 (GIL released)   (GIL released)

I/O threads run truly in parallel because they release the GIL
while waiting for the OS.
```

## When GIL Matters and When It Doesn't

```
GIL is a PROBLEM for:
  ✗ CPU-bound multi-threading (image processing, ML training, crypto)
  ✗ Pure Python computation across threads
  
  Fix: use multiprocessing (separate processes, each with own GIL)
       or use C extensions (NumPy, Pandas release GIL internally)

GIL is NOT a problem for:
  ✓ I/O-bound multi-threading (web servers, database queries, API calls)
    → GIL is released during I/O
  ✓ asyncio (single thread, cooperative multitasking)
  ✓ C extensions (NumPy releases GIL for array operations)
  ✓ multiprocessing (each process has its own GIL)
```

```python
# CPU-BOUND: threading is useless, multiprocessing helps
import multiprocessing

def cpu_heavy(n):
    return sum(i*i for i in range(n))

# WRONG: threads don't parallelize CPU work
with concurrent.futures.ThreadPoolExecutor(4) as pool:
    results = list(pool.map(cpu_heavy, [10**7]*4))   # NOT faster with 4 threads

# RIGHT: processes run on separate cores
with concurrent.futures.ProcessPoolExecutor(4) as pool:
    results = list(pool.map(cpu_heavy, [10**7]*4))   # 4x faster on 4 cores

# I/O-BOUND: threading works great
with concurrent.futures.ThreadPoolExecutor(20) as pool:
    results = list(pool.map(fetch_url, urls))    # 20 concurrent HTTP requests
```

## Python 3.13+ Free-Threading (No-GIL)

```python
# Python 3.13 introduces an EXPERIMENTAL build without the GIL
# Install: python3.13t (free-threaded build)
# Enable: python -X gil=0

# This allows true parallel Python threads for the first time.
# Still experimental. Production code should still use multiprocessing.
```

---

# Chapter 3: Memory Management

## Reference Counting + Generational GC

```python
import sys

a = [1, 2, 3]
print(sys.getrefcount(a))    # 2 (a + getrefcount's argument)

b = a                        # refcount → 3
c = a                        # refcount → 4
del b                        # refcount → 3
c = None                     # refcount → 2

# When refcount reaches 0 → immediately freed (no waiting for GC)

# PROBLEM: circular references
class Node:
    def __init__(self):
        self.ref = None

a = Node()
b = Node()
a.ref = b    # a → b
b.ref = a    # b → a (circular!)
del a, b     # refcount of both is still 1 (they reference each other)
             # refcount GC can't free them!

# SOLUTION: generational garbage collector
import gc
gc.collect()   # finds and breaks circular references

# Three generations: 0 (new objects), 1, 2 (long-lived)
# Gen 0 collected most often, gen 2 least often
# Objects that survive collection are promoted to the next generation
gc.get_stats()   # [{'collections': 100, 'collected': 500, ...}, ...]
```

## Memory Optimization

```python
# __slots__ — eliminate __dict__ per instance
class PointWithDict:
    def __init__(self, x, y):
        self.x = x
        self.y = y
# Each instance has a __dict__ (hash table): ~100+ bytes overhead

class PointWithSlots:
    __slots__ = ('x', 'y')
    def __init__(self, x, y):
        self.x = x
        self.y = y
# No __dict__: ~50 bytes per instance. 40-50% memory savings.
# Tradeoff: can't add arbitrary attributes.

# For 1 million instances:
# With __dict__:  ~160 MB
# With __slots__: ~80 MB

# sys.getsizeof() — check object size
import sys
sys.getsizeof([])           # 56 bytes (empty list)
sys.getsizeof([1, 2, 3])    # 88 bytes
sys.getsizeof({})            # 64 bytes (empty dict)
sys.getsizeof("hello")      # 54 bytes

# Interning — reuse identical immutable objects
a = "hello"
b = "hello"
a is b      # True (interned automatically for simple strings)

# Explicit interning
import sys
a = sys.intern("some_long_string_used_many_times")
```

---

# Chapter 4: Asyncio Internals

## How asyncio Actually Works

```
asyncio is SINGLE-THREADED cooperative multitasking.
No threads, no parallelism. Just one thread switching between tasks at `await` points.

┌────────────────────────────────────────────────────┐
│                EVENT LOOP (single thread)           │
│                                                    │
│  Ready Queue:  [task_A, task_C, task_F]            │
│  Waiting:      {task_B: socket_read,               │
│                 task_D: sleep(1),                   │
│                 task_E: HTTP_response}              │
│                                                    │
│  Loop iteration:                                   │
│    1. Run task_A until it hits `await` → suspends  │
│    2. Run task_C until it hits `await` → suspends  │
│    3. Run task_F until it completes → done         │
│    4. Check if any waiting I/O is ready            │
│    5. task_E's HTTP response arrived → move to     │
│       ready queue                                  │
│    6. Repeat                                       │
└────────────────────────────────────────────────────┘

KEY: `await` is the cooperation point. Without it, other tasks can't run.
     Long computation without `await` blocks the ENTIRE event loop.
```

```python
# WRONG: blocks the event loop (CPU-bound, no await)
async def bad():
    result = heavy_computation()     # runs for 5 seconds, no await
    return result                     # nothing else ran during those 5 seconds

# RIGHT: offload CPU-bound work to a thread pool
import asyncio

async def good():
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(None, heavy_computation)
    return result    # other tasks ran while waiting

# RIGHT: use asyncio.to_thread (Python 3.9+)
async def good2():
    result = await asyncio.to_thread(heavy_computation)
    return result
```

## Task vs Coroutine vs Future

```python
# Coroutine: defined with `async def`, but doesn't run until awaited or wrapped in Task
async def fetch_data():
    await asyncio.sleep(1)
    return "data"

coro = fetch_data()          # coroutine OBJECT (not running yet!)
result = await coro          # NOW it runs

# Task: wraps a coroutine and schedules it to run concurrently
task = asyncio.create_task(fetch_data())    # starts running immediately
# ... do other work ...
result = await task                          # get the result when ready

# asyncio.gather: run multiple coroutines concurrently
results = await asyncio.gather(
    fetch_data(),
    fetch_data(),
    fetch_data(),
)   # all 3 run concurrently, completes in ~1 second (not 3)

# asyncio.TaskGroup (Python 3.11+): structured concurrency
async with asyncio.TaskGroup() as tg:
    task1 = tg.create_task(fetch_data())
    task2 = tg.create_task(fetch_data())
# Both complete before exiting the `async with` block
# If one fails, the other is cancelled automatically
```

---

# Chapter 5: Metaclasses and Class Creation

## How Classes Are Created

```python
# When Python sees:
class MyClass(Base):
    x = 10
    def method(self): pass

# It actually calls:
# MyClass = type('MyClass', (Base,), {'x': 10, 'method': method})

# type(name, bases, namespace) → creates a new class
# So `type` is the metaclass of all classes by default.

# You can override this:
class Meta(type):
    def __new__(mcs, name, bases, namespace):
        print(f"Creating class: {name}")
        # Modify the class before creation
        namespace['created_by_meta'] = True
        cls = super().__new__(mcs, name, bases, namespace)
        return cls

    def __init__(cls, name, bases, namespace):
        super().__init__(name, bases, namespace)
        # Register the class somewhere
        registry.append(cls)

class MyClass(metaclass=Meta):
    pass

# Output: "Creating class: MyClass"
MyClass.created_by_meta    # True

# Simpler alternative: __init_subclass__ (Python 3.6+)
class Plugin:
    _plugins = []

    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        Plugin._plugins.append(cls)

class AudioPlugin(Plugin): pass
class VideoPlugin(Plugin): pass

Plugin._plugins    # [AudioPlugin, VideoPlugin]
```

---

# Chapter 6: Advanced Generators and Coroutine Protocol

```python
# Generators can RECEIVE values via .send()
def accumulator():
    total = 0
    while True:
        value = yield total     # yield sends total OUT, receives value IN
        total += value

gen = accumulator()
next(gen)            # 0 (initialize — runs to first yield)
gen.send(10)         # 10 (total = 0 + 10)
gen.send(20)         # 30 (total = 10 + 20)
gen.send(5)          # 35

# Generator-based context manager
from contextlib import contextmanager

@contextmanager
def timer(name):
    start = time.time()
    yield                          # code inside `with` block runs here
    elapsed = time.time() - start
    print(f"{name}: {elapsed:.4f}s")

with timer("database query"):
    db.execute("SELECT * FROM users")

# Generator delegation with yield from
def flatten(nested):
    for item in nested:
        if isinstance(item, (list, tuple)):
            yield from flatten(item)    # delegate to sub-generator
        else:
            yield item

list(flatten([1, [2, [3, 4]], [5, [6, [7]]]]))
# [1, 2, 3, 4, 5, 6, 7]
```

---

# Chapter 7: 🧩 40LPA Interview Questions

**Q: Explain the GIL. When is it a problem? How do you work around it?**
A: The GIL is a mutex in CPython that allows only one thread to execute Python bytecode at a time. It exists because CPython's memory management (reference counting) is not thread-safe. It's a problem for CPU-bound multi-threading — threads don't run in parallel. Workarounds: `multiprocessing` for CPU-bound work (separate processes, each with own GIL), C extensions like NumPy that release the GIL internally, or `asyncio` for I/O-bound concurrency. Python 3.13 has an experimental free-threaded build.

**Q: How does Python's attribute lookup work?**
A: Python follows the descriptor protocol and MRO. For `obj.attr`: first check the class MRO for a data descriptor (has both `__get__` and `__set__`), then check `obj.__dict__`, then check the MRO for a non-data descriptor (only `__get__`). This is why `@property` overrides instance attributes. `__getattr__` is the fallback if nothing is found. `__getattribute__` intercepts ALL access.

**Q: What's the difference between `__new__` and `__init__`?**
A: `__new__` creates the instance (allocates memory, returns the object). `__init__` initializes the instance (sets attributes). `__new__` receives the class as first argument and must return an instance. `__init__` receives the instance. Use `__new__` for immutable types (str, int — can't modify in __init__), singletons, and metaclass-like patterns.

**Q: How does asyncio achieve concurrency without threads?**
A: It uses an event loop on a single thread with cooperative multitasking. Coroutines voluntarily yield control at `await` points. The event loop uses OS-level I/O multiplexing (epoll/kqueue) to monitor multiple sockets/files simultaneously. When I/O is ready, the corresponding coroutine is resumed. No thread switching, no locks, no GIL contention. The tradeoff: CPU-bound work blocks the entire loop.

**Q: Implement a thread-safe singleton in Python.**
```python
import threading

class Singleton:
    _instance = None
    _lock = threading.Lock()

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:                    # fast check (no lock)
            with cls._lock:                           # acquire lock
                if cls._instance is None:             # double-check under lock
                    cls._instance = super().__new__(cls)
        return cls._instance
```

**Q: What are descriptors and why do they matter?**
A: Descriptors are objects that define `__get__`, `__set__`, or `__delete__`. They intercept attribute access at the class level. `@property`, `@classmethod`, `@staticmethod`, and `__slots__` are all implemented as descriptors. Understanding them is key to understanding how Python's attribute system works, and they're the mechanism behind ORMs like SQLAlchemy (column definitions are descriptors).
