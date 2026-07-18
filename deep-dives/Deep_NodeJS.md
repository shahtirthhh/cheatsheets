# Deep Node.js — 40LPA Level
*libuv · Event Loop Internals · Streams · Worker Threads · Performance Tuning*

---

# Chapter 1: libuv — The Engine Behind Node.js

## Architecture

```
┌───────────────────────────────────────────────────────────┐
│                      Node.js                              │
│                                                           │
│  ┌───────────┐    ┌──────────┐    ┌────────────────────┐  │
│  │    V8     │    │ Node.js  │    │     libuv          │  │
│  │ (JS exec) │    │ Bindings │    │ (async I/O)        │  │
│  │           │◄──►│ (C++)    │◄──►│                    │  │
│  │           │    │          │    │ ┌────────────────┐ │  │
│  └───────────┘    └──────────┘    │ │ Event Loop     │ │  │
│                                   │ │ (single thread)│ │  │
│  ┌───────────┐                    │ └────────────────┘ │  │
│  │  Node     │                    │ ┌────────────────┐ │  │
│  │  APIs     │                    │ │ Thread Pool    │ │  │
│  │ (fs,http, │                    │ │ (4 threads     │ │  │
│  │  crypto)  │                    │ │  default)      │ │  │
│  └───────────┘                    │ └────────────────┘ │  │
│                                   │ ┌────────────────┐ │  │
│                                   │ │ OS Async I/O   │ │  │
│                                   │ │ (epoll/kqueue/ │ │  │
│                                   │ │  IOCP)         │ │  │
│                                   │ └────────────────┘ │  │
│                                   └────────────────────┘  │
└───────────────────────────────────────────────────────────┘
```

## What Uses the Thread Pool vs OS Async

```
Thread Pool (blocking operations that libuv offloads):
  • fs.readFile, fs.writeFile (file system)
  • dns.lookup() (uses getaddrinfo — blocking)
  • crypto.pbkdf2, crypto.scrypt, crypto.randomBytes
  • zlib compression/decompression

  Default: 4 threads. Set via UV_THREADPOOL_SIZE (max 1024).
  If all 4 threads are busy, operations queue up!

  INTERVIEW GOTCHA: If you do 100 concurrent file reads,
  only 4 run at a time. The other 96 wait in a queue.

OS Async (kernel handles it, no thread pool):
  • TCP/UDP sockets (http, net)
  • DNS resolving (dns.resolve — uses c-ares, not getaddrinfo)
  • Pipes
  • TTY
  • Signals

  These scale to thousands of concurrent connections with ZERO thread pool usage.
```

## Thread Pool Starvation

```javascript
// PROBLEM: crypto operations consume all thread pool threads
const crypto = require('crypto');

// 100 concurrent pbkdf2 calls → 4 threads, 96 queued
for (let i = 0; i < 100; i++) {
  crypto.pbkdf2('password', 'salt', 100000, 64, 'sha512', (err, key) => {
    console.log(`Done ${i}`);
  });
}

// MEANWHILE: fs.readFile also needs the thread pool
fs.readFile('config.json', (err, data) => {
  // This waits until a thread pool thread is free!
  // If crypto operations take 2 seconds each, this file read
  // might wait 50+ seconds.
});

// FIX: increase thread pool size
// UV_THREADPOOL_SIZE=64 node app.js

// BETTER FIX: use Worker Threads for CPU-heavy crypto
```

---

# Chapter 2: Streams Deep Dive

## The 4 Stream Types and Their Events

```
                  ┌───────────┐
    Readable ────▶│ Transform │────▶ Writable
    (source)      │ (modify)  │      (destination)
                  └───────────┘
    
    Duplex: both Readable AND Writable (e.g., TCP socket)

Readable events:
  'data'     — chunk of data available
  'end'      — no more data
  'error'    — something went wrong
  'readable' — data available to read (pull mode)

Writable events:
  'drain'    — buffer emptied, safe to write more
  'finish'   — all data flushed
  'error'    — something went wrong

Transform: readable + writable, transforms data passing through
```

## Backpressure — The Most Important Stream Concept

```
PROBLEM: The readable produces data faster than the writable can consume it.

Without backpressure:
  Readable: [data] [data] [data] [data] [data] [data] ...
  Writable:        [processing...]
  Memory:   ↑↑↑↑↑↑ grows unboundedly → OOM crash!

With backpressure:
  Readable: [data] [data] → writable buffer full → PAUSE readable
  Writable:        [processing...] [processing...] → buffer drained → RESUME readable
  Memory:   stable ✓

The pipe() method handles backpressure automatically:
  readable.pipe(writable);
  // If writable is slow, pipe pauses readable. When writable drains, pipe resumes.

Manual backpressure:
  const ok = writable.write(chunk);     // returns false if buffer is full
  if (!ok) {
    readable.pause();                    // stop reading
    writable.once('drain', () => {
      readable.resume();                 // resume when buffer empties
    });
  }
```

## pipeline() — The Modern Way

```javascript
const { pipeline } = require('stream/promises');
const { createReadStream, createWriteStream } = require('fs');
const { createGzip } = require('zlib');
const { Transform } = require('stream');

// Custom transform stream
const uppercase = new Transform({
  transform(chunk, encoding, callback) {
    callback(null, chunk.toString().toUpperCase());
  }
});

// Pipe with automatic error handling and cleanup
await pipeline(
  createReadStream('input.txt'),
  uppercase,
  createGzip(),
  createWriteStream('output.txt.gz')
);
// If any stream errors, all are cleaned up automatically.
// Without pipeline, you'd need to handle errors on EACH stream.
```

## HTTP Streaming

```javascript
// Stream a response (don't buffer entire file in memory)
app.get('/download/:file', (req, res) => {
  const stream = fs.createReadStream(`./files/${req.params.file}`);

  // Set headers BEFORE piping
  res.setHeader('Content-Type', 'application/octet-stream');

  stream.pipe(res);    // backpressure handled automatically

  stream.on('error', (err) => {
    if (!res.headersSent) {
      res.status(404).json({ error: 'File not found' });
    }
  });
});

// Stream JSON (for large datasets — don't JSON.stringify a million rows)
app.get('/export', async (req, res) => {
  res.setHeader('Content-Type', 'application/json');
  res.write('[\n');

  const cursor = collection.find({}).stream();
  let first = true;

  cursor.on('data', (doc) => {
    if (!first) res.write(',\n');
    const ok = res.write(JSON.stringify(doc));
    if (!ok) cursor.pause();    // backpressure
    first = false;
  });

  res.on('drain', () => cursor.resume());   // resume when client catches up

  cursor.on('end', () => {
    res.write('\n]');
    res.end();
  });
});
```

---

# Chapter 3: Worker Threads vs Child Processes vs Cluster

```
                   Worker Threads    child_process.fork()    Cluster
                   ──────────────    ────────────────────    ───────
Memory             Shared process    Separate process        Separate process
Communication      MessagePort +     IPC (serialized JSON)   IPC
                   SharedArrayBuffer
Isolation          Partial           Full                    Full
Startup            ~5-30ms           ~100ms                  ~100ms
GC                 Shared            Independent             Independent
Use case           CPU tasks with    Untrusted code,         HTTP server
                   data sharing      full isolation           scaling
Max concurrency    UV_THREADPOOL_SIZE Based on cores          Based on cores
Can crash parent?  Yes               No                      No
```

## Worker Thread Pool (Production Pattern)

```javascript
// pool.js — reusable thread pool
const { Worker } = require('worker_threads');
const os = require('os');

class WorkerPool {
  constructor(workerFile, size = os.cpus().length) {
    this.workers = [];
    this.queue = [];
    this.activeCount = 0;

    for (let i = 0; i < size; i++) {
      this.workers.push(this._createWorker(workerFile));
    }
  }

  _createWorker(file) {
    const worker = new Worker(file);
    worker.busy = false;
    worker.on('message', (result) => {
      worker.busy = false;
      worker._resolve(result);
      this._processQueue();
    });
    worker.on('error', (err) => {
      worker.busy = false;
      worker._reject(err);
      this._processQueue();
    });
    return worker;
  }

  execute(data) {
    return new Promise((resolve, reject) => {
      const idle = this.workers.find(w => !w.busy);
      if (idle) {
        idle.busy = true;
        idle._resolve = resolve;
        idle._reject = reject;
        idle.postMessage(data);
      } else {
        this.queue.push({ data, resolve, reject });
      }
    });
  }

  _processQueue() {
    if (this.queue.length === 0) return;
    const idle = this.workers.find(w => !w.busy);
    if (!idle) return;
    const { data, resolve, reject } = this.queue.shift();
    idle.busy = true;
    idle._resolve = resolve;
    idle._reject = reject;
    idle.postMessage(data);
  }

  async destroy() {
    await Promise.all(this.workers.map(w => w.terminate()));
  }
}

module.exports = WorkerPool;
```

---

# Chapter 4: Production Performance

## Memory Leak Detection

```javascript
// 1. Monitor in production
setInterval(() => {
  const mem = process.memoryUsage();
  metrics.gauge('heap_used_mb', mem.heapUsed / 1e6);
  metrics.gauge('rss_mb', mem.rss / 1e6);
  metrics.gauge('external_mb', mem.external / 1e6);

  // Alert if heap grows past threshold
  if (mem.heapUsed > 500 * 1e6) {
    logger.warn('High heap usage', { heapMB: mem.heapUsed / 1e6 });
  }
}, 30000);

// 2. Generate heap snapshots on demand
const v8 = require('v8');

app.get('/debug/heapdump', adminAuth, (req, res) => {
  const filename = `/tmp/heapdump-${Date.now()}.heapsnapshot`;
  const snapshotStream = v8.writeHeapSnapshot(filename);
  res.json({ file: filename });
  // Load in Chrome DevTools → Memory tab → Load
});

// 3. Expose GC stats
const { performance, PerformanceObserver } = require('perf_hooks');

const obs = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.entryType === 'gc') {
      metrics.histogram('gc_duration_ms', entry.duration);
      metrics.counter('gc_count', 1, { kind: entry.detail.kind });
    }
  }
});
obs.observe({ entryTypes: ['gc'] });
```

## Graceful Shutdown

```javascript
const server = app.listen(3000);
let isShuttingDown = false;

async function shutdown(signal) {
  if (isShuttingDown) return;
  isShuttingDown = true;
  logger.info(`${signal} received. Starting graceful shutdown...`);

  // 1. Stop accepting new connections
  server.close();

  // 2. Stop consuming from message queues
  await kafkaConsumer.disconnect();

  // 3. Wait for in-flight requests to complete (with timeout)
  const forceTimeout = setTimeout(() => {
    logger.error('Forced shutdown after timeout');
    process.exit(1);
  }, 30000);

  // 4. Close database connections
  await mongoose.connection.close();
  await redis.quit();

  clearTimeout(forceTimeout);
  logger.info('Graceful shutdown complete');
  process.exit(0);
}

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));

// Health check should return unhealthy during shutdown
app.get('/health', (req, res) => {
  if (isShuttingDown) return res.status(503).json({ status: 'shutting down' });
  res.json({ status: 'healthy' });
});
```

---

# Chapter 5: 🧩 40LPA Interview Questions

**Q: What operations use the libuv thread pool?**
A: File system operations (fs module), DNS lookups via `dns.lookup()` (which uses the blocking `getaddrinfo`), crypto operations (pbkdf2, scrypt, randomBytes), and zlib compression. Default pool size is 4, configurable via `UV_THREADPOOL_SIZE`. Network I/O (TCP, HTTP, UDP) does NOT use the thread pool — it uses OS async primitives (epoll on Linux). Thread pool starvation happens when CPU-heavy operations (like crypto) consume all threads, blocking file I/O.

**Q: Explain backpressure in Node.js streams.**
A: When a Readable produces data faster than a Writable can consume it, the Writable's internal buffer fills up. `writable.write()` returns `false` to signal "stop sending." The Readable should pause until the Writable emits `'drain'`. The `pipe()` method handles this automatically. Without backpressure handling, the buffer grows unboundedly until OOM. This is critical for HTTP file uploads, database exports, and any streaming pipeline.

**Q: How would you handle a memory leak in production Node.js?**
A: Step 1: Monitor `process.memoryUsage()` over time — if `heapUsed` grows monotonically, there's a leak. Step 2: Take periodic heap snapshots (`v8.writeHeapSnapshot()`). Step 3: Compare snapshots in Chrome DevTools — look for objects whose count grows between snapshots. Step 4: Common causes — unbounded caches (use LRU), event listeners not cleaned up, closures retaining large objects, global variables. Step 5: Use `--inspect` flag and clinic.js/0x for automated diagnosis.

**Q: Worker Threads vs child_process.fork — when to use each?**
A: Worker Threads share the same process memory, use MessagePort for communication, and can share data via SharedArrayBuffer (zero-copy). Use for CPU-intensive tasks where you need to share large datasets. `fork()` creates a completely separate process with its own V8 heap and GIL-equivalent. Use when you need full isolation (untrusted code, crash protection). Workers are ~3x faster to spin up. For HTTP scaling, use the Cluster module or PM2 (which uses fork internally).

**Q: Why does `setTimeout(fn, 0)` not execute immediately?**
A: The callback is placed in the Timer phase queue. Before it runs, the event loop must: finish the current synchronous code, drain ALL microtasks (process.nextTick, Promises), then enter the Timer phase. Also, Node.js clamps the delay to a minimum of 1ms. So the actual delay is at least: time to drain microtasks + 1ms + time to reach the Timer phase. `setImmediate` is similar but runs in the Check phase (after I/O polling), not the Timer phase.
