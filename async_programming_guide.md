# Asynchronous Programming Guide

## Table of Contents
1. [Event Loop Blocking](#event-loop-blocking)
2. [Callback Hell](#callback-hell)
3. [Async Generators and Iterators](#async-generators-and-iterators)
4. [Error Handling in Async Code](#error-handling-in-async-code)
5. [Async Contracts](#async-contracts)
6. [Promises](#promises)
7. [Workers and Threads](#workers-and-threads)
8. [Performance Measurement](#performance-measurement)
9. [State Management](#state-management)
10. [Reactive Programming](#reactive-programming)
11. [Streams](#streams)
12. [Testing Async Code](#testing-async-code)
13. [Best Practices](#best-practices)
14. [Advanced Topics](#advanced-topics)

---

## Event Loop Blocking

### How can we write code to avoid blocking the event loop?

To avoid blocking the event loop:

1. **Use asynchronous APIs**: Prefer async I/O operations (fs.promises, fetch, etc.) over synchronous ones
2. **Break up CPU-intensive work**: Use `setImmediate()` or `process.nextTick()` to yield control
3. **Use Worker Threads**: Offload CPU-bound tasks to separate threads
4. **Batch operations**: Process data in chunks rather than all at once

```javascript
// Bad: Blocks event loop
function processLargeArray(data) {
  return data.map(item => heavyComputation(item));
}

// Good: Yields control periodically
async function processLargeArray(data) {
  const results = [];
  for (let i = 0; i < data.length; i++) {
    results.push(heavyComputation(data[i]));
    if (i % 100 === 0) {
      await new Promise(resolve => setImmediate(resolve));
    }
  }
  return results;
}
```

### How can we handle a blocked event loop to exit the blocking state from the same process?

Once the event loop is blocked, you cannot exit from the same process because the event loop is not processing. However, you can:

1. **Prevent blocking**: Use the techniques above before blocking occurs
2. **Use Worker Threads**: Run blocking code in a separate thread
3. **Set timeouts**: Use `setTimeout` with a callback to detect if the event loop is responsive
4. **Monitor event loop lag**: Use `performance.now()` to detect delays

```javascript
// Monitor event loop responsiveness
let lastCheck = Date.now();
setInterval(() => {
  const now = Date.now();
  const lag = now - lastCheck - 1000; // Expected 1 second
  if (lag > 100) {
    console.warn('Event loop lag detected:', lag, 'ms');
  }
  lastCheck = now;
}, 1000);
```

---

## Callback Hell

### What is callback hell, and how can we avoid it?

**Callback hell** is nested callbacks that make code hard to read and maintain:

```javascript
// Callback hell example
getData(function(a) {
  getMoreData(a, function(b) {
    getMoreData(b, function(c) {
      getMoreData(c, function(d) {
        // Deep nesting, hard to read
      });
    });
  });
});
```

**Solutions:**

1. **Use Promises**: Chain operations with `.then()`
2. **Use async/await**: Write sequential-looking code
3. **Use named functions**: Extract callbacks to named functions
4. **Use Promise utilities**: `Promise.all()`, `Promise.allSettled()`, etc.

```javascript
// With async/await
async function processData() {
  const a = await getData();
  const b = await getMoreData(a);
  const c = await getMoreData(b);
  return await getMoreData(c);
}
```

---

## Async Generators and Iterators

### What are async generators and iterators, how do they work, and what are their use cases?

**Async generators** are functions that return async iterables. They use `async function*` syntax and `yield` to produce values asynchronously.

**How they work:**
- `async function*` creates an async generator
- `yield` pauses execution and returns a value wrapped in a Promise
- `for await...of` iterates over async iterables
- `yield*` delegates to another async iterable

```javascript
// Async generator
async function* fetchPages(url) {
  let page = 1;
  while (true) {
    const response = await fetch(`${url}?page=${page}`);
    const data = await response.json();
    if (data.length === 0) break;
    yield data;
    page++;
  }
}

// Usage
for await (const page of fetchPages('/api/data')) {
  console.log(page);
}
```

**Use cases:**
- Streaming data processing
- Paginated API responses
- Real-time data feeds
- Processing large datasets in chunks
- Implementing async queues

---

## Error Handling in Async Code

### How do we handle errors in async code?

1. **try/catch with async/await**:
```javascript
async function example() {
  try {
    const result = await asyncOperation();
    return result;
  } catch (error) {
    console.error('Error:', error);
    throw error; // Re-throw if needed
  }
}
```

2. **Promise.catch()**:
```javascript
asyncOperation()
  .then(result => process(result))
  .catch(error => console.error('Error:', error));
```

3. **Error boundaries in async generators**:
```javascript
async function* safeGenerator() {
  try {
    yield await riskyOperation();
  } catch (error) {
    console.error('Generator error:', error);
    yield null; // Or handle gracefully
  }
}
```

### When does try/catch capture async errors and when does it not?

**try/catch captures:**
- Errors from `await` expressions
- Synchronous errors in async functions
- Errors thrown in the same async function

**try/catch does NOT capture:**
- Errors in `.then()` callbacks (use `.catch()`)
- Errors in `setTimeout`/`setInterval` callbacks
- Unhandled promise rejections
- Errors in event handlers (unless awaited)

```javascript
// Captured
async function example1() {
  try {
    await asyncOperation(); // ✅ Error caught
  } catch (error) {
    console.log('Caught');
  }
}

// NOT captured
async function example2() {
  try {
    asyncOperation().then(() => {
      throw new Error('Not caught'); // ❌ Not caught
    });
  } catch (error) {
    // Won't catch the error above
  }
}
```

### Which async abstraction supports the captureRejections flag, and what is it for?

**EventEmitter** supports `captureRejections` flag (Node.js 13+). When enabled, it automatically captures promise rejections from async event handlers.

```javascript
const EventEmitter = require('events');

const emitter = new EventEmitter({ captureRejections: true });

emitter.on('event', async () => {
  throw new Error('Async error');
});

emitter.on('error', (error) => {
  console.log('Caught rejection:', error); // Automatically caught
});
```

Without `captureRejections`, unhandled rejections in async event handlers would cause unhandled rejection warnings.

---

## Stack Traces and Debugging

### How can we avoid losing steps in the stack trace and improve debugging and understanding of control flow using async/await?

**Problems:**
- Callbacks lose stack traces
- Promise chains can truncate stack traces
- Async operations break call stack continuity

**Solutions:**

1. **Use async/await**: Preserves better stack traces than callbacks
2. **Use Error.captureStackTrace()**: Add context to errors
3. **Use async hooks**: Track async context (Node.js)
4. **Add error context**: Include operation details in errors

```javascript
// Better stack traces with async/await
async function operation1() {
  await operation2(); // Stack trace includes this
}

async function operation2() {
  await operation3(); // Stack trace includes this
}

async function operation3() {
  throw new Error('Error with full stack trace');
}

// Add context
async function fetchWithContext(url) {
  try {
    return await fetch(url);
  } catch (error) {
    error.context = { url, timestamp: Date.now() };
    throw error;
  }
}
```

---

## Cancelling Async Operations

### How can we cancel async operations?

1. **AbortController/AbortSignal** (modern approach):
```javascript
const controller = new AbortController();
const signal = controller.signal;

fetch('/api/data', { signal })
  .then(response => response.json())
  .catch(error => {
    if (error.name === 'AbortError') {
      console.log('Request cancelled');
    }
  });

// Cancel after 5 seconds
setTimeout(() => controller.abort(), 5000);
```

2. **Cancellation tokens** (custom implementation):
```javascript
class CancellationToken {
  constructor() {
    this.cancelled = false;
    this.listeners = [];
  }
  
  cancel() {
    this.cancelled = true;
    this.listeners.forEach(listener => listener());
  }
  
  onCancel(listener) {
    this.listeners.push(listener);
  }
}
```

3. **Promise.race() with timeout**:
```javascript
function withTimeout(promise, timeout) {
  return Promise.race([
    promise,
    new Promise((_, reject) => 
      setTimeout(() => reject(new Error('Timeout')), timeout)
    )
  ]);
}
```

---

## Async Contracts

### What is the difference between async contracts: callbacks, events, async/await, promises, etc.?

| Contract | Characteristics | Use Cases |
|----------|----------------|-----------|
| **Callbacks** | Function passed as argument, called when done | Node.js legacy APIs, event handlers |
| **Promises** | Object representing future value, chainable | Modern async APIs, composable operations |
| **async/await** | Syntactic sugar over Promises | Sequential async code, error handling |
| **Events** | Observer pattern, multiple listeners | Real-time updates, pub/sub systems |
| **Streams** | Continuous data flow | File I/O, network data, large datasets |
| **Async Iterators** | Iteration over async sequences | Pagination, streaming, generators |

### How are async contracts related, and is it possible to eliminate older ones?

**Relationships:**
- `async/await` is syntactic sugar over Promises
- Promises can wrap callbacks: `new Promise((resolve, reject) => callback(resolve, reject))`
- EventEmitters can be converted to Promises: `once(emitter, 'event')`
- Streams implement async iterables
- Callbacks are the foundation for all others

**Can we eliminate older ones?**
- **Callbacks**: Still needed for some APIs (setTimeout, event listeners)
- **Events**: Essential for real-time, multi-listener scenarios
- **Promises**: Foundation for async/await, still needed for utilities
- **async/await**: Preferred for most code, but can't replace all patterns

**Best practice**: Use the right tool for the job. Modern code should prefer async/await, but callbacks and events still have valid use cases.

---

## Promise Methods

### What is the difference between Promise.all() and Promise.allSettled()?

**Promise.all():**
- Fails fast: rejects immediately if any promise rejects
- Returns array of resolved values
- All promises must succeed

```javascript
Promise.all([promise1, promise2, promise3])
  .then(values => {
    // All succeeded: [value1, value2, value3]
  })
  .catch(error => {
    // First rejection stops everything
  });
```

**Promise.allSettled():**
- Waits for all promises to settle (resolve or reject)
- Returns array of status objects: `{status: 'fulfilled', value}` or `{status: 'rejected', reason}`
- Never rejects

```javascript
Promise.allSettled([promise1, promise2, promise3])
  .then(results => {
    // All settled: [
    //   {status: 'fulfilled', value: ...},
    //   {status: 'rejected', reason: ...}
    // ]
  });
```

### What is the difference between f2 and f3 in: promiseInstance.then(f1, f2).catch(f3)?

- **f2**: Error handler in `.then()` - catches rejections from the original promise
- **f3**: Error handler in `.catch()` - catches rejections from f1 (the success handler)

```javascript
promiseInstance
  .then(f1, f2)  // f2 catches promiseInstance rejection
  .catch(f3);    // f3 catches f1 rejection

// Example:
Promise.reject('error1')
  .then(
    value => value,           // f1: not called
    error => {                // f2: catches 'error1'
      throw 'error2';
    }
  )
  .catch(error => {           // f3: catches 'error2'
    console.log(error);       // 'error2'
  });
```

### When and why might we have multiple catch clauses: promiseInstance.catch(f1).catch(f2).catch(f3)?

Multiple `.catch()` clauses are useful when:
- Each catch handles different error types
- You want to transform errors at different stages
- You need fallback error handling

```javascript
promiseInstance
  .catch(f1)  // Handle specific error type
  .catch(f2)  // Handle other errors
  .catch(f3); // Final fallback

// Example:
fetch('/api/data')
  .catch(error => {
    if (error instanceof TypeError) {
      return retryRequest(); // f1: network error, retry
    }
    throw error;
  })
  .catch(error => {
    if (error.status === 404) {
      return getDefaultData(); // f2: not found, use default
    }
    throw error;
  })
  .catch(error => {
    logError(error); // f3: log any remaining errors
    return null;
  });
```

### Why do we have Promise method finally, and what are its use cases?

**Promise.finally()** executes regardless of resolve/reject, useful for cleanup.

**Use cases:**
- Cleanup resources (close connections, clear timers)
- Hide loading indicators
- Reset state
- Logging completion

```javascript
let loading = true;

fetch('/api/data')
  .then(data => process(data))
  .catch(error => handleError(error))
  .finally(() => {
    loading = false; // Always executed
    hideSpinner();
  });
```

---

## Generators and Async

### How can we write async code with sync generators? What are the advantages and disadvantages?

**Using generators for async flow control** (co library pattern):

```javascript
function* asyncGenerator() {
  const data1 = yield fetch('/api/1').then(r => r.json());
  const data2 = yield fetch('/api/2').then(r => r.json());
  return [data1, data2];
}

// Runner function
function run(generator) {
  const gen = generator();
  
  function handle(result) {
    if (result.done) return Promise.resolve(result.value);
    return Promise.resolve(result.value)
      .then(res => handle(gen.next(res)))
      .catch(err => handle(gen.throw(err)));
  }
  
  return handle(gen.next());
}
```

**Advantages:**
- Can be used before async/await was available
- Fine-grained control over execution
- Can yield multiple times

**Disadvantages:**
- More complex than async/await
- Requires a runner function
- Less intuitive
- No native support

**Modern alternative**: Use async/await instead.

---

## Workers and Threads

### Describe the differences between Web Workers, Shared Workers, and Worker Threads.

| Type | Environment | Shared State | Use Cases |
|------|-------------|--------------|-----------|
| **Web Workers** | Browser | No (message passing only) | CPU-intensive tasks, background processing |
| **Shared Workers** | Browser | Shared across tabs/windows | Shared state across browser contexts |
| **Worker Threads** | Node.js | SharedArrayBuffer, Atomics | CPU-bound tasks, parallel processing |

**Web Workers:**
```javascript
// main.js
const worker = new Worker('worker.js');
worker.postMessage({ data: 'hello' });
worker.onmessage = (e) => console.log(e.data);

// worker.js
self.onmessage = (e) => {
  const result = heavyComputation(e.data);
  self.postMessage(result);
};
```

**Worker Threads (Node.js):**
```javascript
const { Worker } = require('worker_threads');

const worker = new Worker(__filename, {
  workerData: { start: 0 }
});

worker.on('message', (result) => {
  console.log(result);
});
```

---

## Event Loop and Tasks

### Describe microtasks and macrotasks and their relation to the event loop.

**Macrotasks (Task Queue):**
- `setTimeout`, `setInterval`
- I/O callbacks
- UI rendering (browser)
- Executed one per event loop iteration

**Microtasks (Microtask Queue):**
- Promise callbacks (`.then()`, `.catch()`, `.finally()`)
- `queueMicrotask()`
- `process.nextTick()` (Node.js, higher priority)
- Executed after current task, before next macrotask

**Execution order:**
1. Execute current macrotask
2. Execute all microtasks (until queue is empty)
3. Render (browser)
4. Execute next macrotask

```javascript
console.log('1'); // Synchronous

setTimeout(() => console.log('2'), 0); // Macrotask

Promise.resolve().then(() => console.log('3')); // Microtask

queueMicrotask(() => console.log('4')); // Microtask

console.log('5'); // Synchronous

// Output: 1, 5, 3, 4, 2
```

---

## Performance Measurement

### How can we measure I/O operations performance and resource usage?

1. **performance.now()** (high resolution):
```javascript
const start = performance.now();
await fetch('/api/data');
const duration = performance.now() - start;
console.log(`Fetch took ${duration}ms`);
```

2. **process.hrtime()** and **process.hrtime.bigint()**:
```javascript
// process.hrtime() - returns [seconds, nanoseconds] array
const start = process.hrtime();
await operation();
const diff = process.hrtime(start);
const ms = diff[0] * 1000 + diff[1] / 1e6;

// process.hrtime.bigint() - returns BigInt nanoseconds
const start = process.hrtime.bigint();
await operation();
const duration = Number(process.hrtime.bigint() - start) / 1e6; // ms
```

**Difference:**
- `process.hrtime()`: Returns array `[seconds, nanoseconds]`
- `process.hrtime.bigint()`: Returns single BigInt (nanoseconds), more convenient

3. **perf_hooks module**:
```javascript
const { performance, PerformanceObserver } = require('perf_hooks');

const obs = new PerformanceObserver((list) => {
  const entries = list.getEntries();
  entries.forEach(entry => {
    console.log(`${entry.name}: ${entry.duration}ms`);
  });
});

obs.observe({ entryTypes: ['measure', 'function'] });

performance.mark('start');
await operation();
performance.mark('end');
performance.measure('operation', 'start', 'end');
```

### Tell us about: const { performance } = require('node:perf_hooks');

The `perf_hooks` module provides:
- **performance.now()**: High-resolution time
- **PerformanceObserver**: Monitor performance metrics
- **performance.mark()**: Create timing marks
- **performance.measure()**: Measure between marks
- **performance.timerify()**: Wrap functions to measure automatically

```javascript
const { performance, PerformanceObserver } = require('perf_hooks');

// Automatic function timing
const obs = new PerformanceObserver((list) => {
  console.log(list.getEntries()[0]);
});
obs.observe({ entryTypes: ['function'] });

const timedFunction = performance.timerify(async function() {
  await someOperation();
});

await timedFunction(); // Automatically measured
```

---

## Handling Large Data

### How can we efficiently handle asynchronous API requests at the client side that return large amounts of data?

1. **Streaming with ReadableStream**:
```javascript
const response = await fetch('/api/large-data');
const reader = response.body.getReader();

while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  processChunk(value); // Process in chunks
}
```

2. **Pagination**:
```javascript
async function* fetchAllPages() {
  let page = 1;
  while (true) {
    const data = await fetch(`/api/data?page=${page}`).then(r => r.json());
    if (data.length === 0) break;
    yield data;
    page++;
  }
}
```

3. **Lazy loading**: Load data as needed
4. **Compression**: Use gzip/brotli
5. **Virtual scrolling**: Render only visible items

### How can we efficiently handle API requests at the server side that return large amounts of data asynchronously?

1. **Streaming responses**:
```javascript
// Express example
app.get('/large-data', async (req, res) => {
  res.setHeader('Content-Type', 'application/json');
  res.write('[');
  
  const stream = db.queryStream('SELECT * FROM large_table');
  let first = true;
  
  for await (const row of stream) {
    if (!first) res.write(',');
    res.write(JSON.stringify(row));
    first = false;
  }
  
  res.write(']');
  res.end();
});
```

2. **Backpressure handling**: Use Node.js streams
3. **Compression**: `res.pipe(createGzip())`
4. **Chunked transfer encoding**: Automatic with streams

---

## State Management

### How can we ensure state isolation between different asynchronous requests in a single Node.js process?

1. **AsyncLocalStorage** (Node.js 13+):
```javascript
const { AsyncLocalStorage } = require('async_hooks');

const context = new AsyncLocalStorage();

// Set context per request
app.use((req, res, next) => {
  context.run({ requestId: req.id, user: req.user }, () => {
    next();
  });
});

// Access context anywhere in async chain
function log(message) {
  const ctx = context.getStore();
  console.log(`[${ctx.requestId}] ${message}`);
}
```

2. **Continuation Local Storage (CLS)** - legacy, use AsyncLocalStorage instead
3. **Request-scoped variables**: Pass context through function parameters
4. **Thread-local storage**: Not applicable (single-threaded event loop)

### What is CLS (continuation local storage), and do we have a modern substitution?

**CLS** was a technique to store context that follows async execution. It's been replaced by **AsyncLocalStorage** in Node.js 13+.

**Modern approach:**
```javascript
const { AsyncLocalStorage } = require('async_hooks');
const storage = new AsyncLocalStorage();

// Old CLS pattern (deprecated)
// const cls = require('continuation-local-storage');

// Modern AsyncLocalStorage
storage.run({ userId: 123 }, async () => {
  await operation1(); // Can access storage.getStore()
  await operation2(); // Context preserved
});
```

---

## Event Loop Phases

### Which event loop phases are related to pending callbacks?

Node.js event loop phases:
1. **Timers**: `setTimeout`, `setInterval`
2. **Pending callbacks**: Execute I/O callbacks deferred to next iteration
3. **Idle, prepare**: Internal use
4. **Poll**: Fetch new I/O events
5. **Check**: `setImmediate()` callbacks
6. **Close callbacks**: `socket.on('close')`

**Pending callbacks phase** executes callbacks that were deferred from the previous iteration (e.g., some TCP errors).

---

## Thenable Contract

### Tell us about Thenable contract and its relation to Promise.

A **Thenable** is any object with a `.then()` method. Promises implement the Thenable contract, but not all Thenables are Promises.

```javascript
// Thenable (not a Promise)
const thenable = {
  then(resolve, reject) {
    setTimeout(() => resolve(42), 1000);
  }
};

// Promise.resolve() accepts Thenables
Promise.resolve(thenable).then(value => {
  console.log(value); // 42
});

// Async/await works with Thenables
const value = await thenable; // 42
```

**Why it matters:**
- Allows interoperability between Promise implementations
- Enables async/await to work with non-native Promises
- Used by libraries like Bluebird

---

## Tracking Async Operations

### How can we associate some state (collection or data structure) with the chain of async calls?

1. **AsyncLocalStorage**:
```javascript
const storage = new AsyncLocalStorage();

storage.run({ requests: [] }, async () => {
  const store = storage.getStore();
  store.requests.push('request1');
  await operation1();
  // store.requests still accessible
});
```

2. **Pass context explicitly**:
```javascript
async function process(context) {
  context.steps.push('step1');
  await operation1();
  context.steps.push('step2');
  return context;
}
```

3. **Closure**:
```javascript
async function createProcessor() {
  const state = { requests: [] };
  
  return {
    async process() {
      state.requests.push('new');
      await operation();
    },
    getState: () => state
  };
}
```

### How can we track the chain of async calls from external requests (originating from API call via HTTP, UDP, IPC, WebSocket)?

1. **AsyncLocalStorage with request ID**:
```javascript
const storage = new AsyncLocalStorage();

// HTTP
app.use((req, res, next) => {
  const requestId = generateId();
  storage.run({ requestId, type: 'http' }, () => {
    next();
  });
});

// WebSocket
ws.on('connection', (socket) => {
  socket.on('message', (data) => {
    const requestId = generateId();
    storage.run({ requestId, type: 'websocket' }, async () => {
      await handleMessage(data);
    });
  });
});

// Track anywhere
function log(message) {
  const ctx = storage.getStore();
  logger.info(`[${ctx.requestId}] ${message}`);
}
```

2. **Async hooks** (advanced):
```javascript
const async_hooks = require('async_hooks');

const hook = async_hooks.createHook({
  init(asyncId, type, triggerAsyncId) {
    // Track async resource creation
  }
});
hook.enable();
```

---

## Concurrency Control

### How can we ensure safe processing of competing requests to a resource?

1. **Locks (Web Locks API)**:
```javascript
// Browser
navigator.locks.request('resource-key', async (lock) => {
  // Exclusive access
  await modifyResource();
});

// Node.js: Use async-mutex or similar
```

2. **Mutex/Semaphore**:
```javascript
class Mutex {
  constructor() {
    this.queue = [];
    this.locked = false;
  }
  
  async acquire() {
    return new Promise((resolve) => {
      if (!this.locked) {
        this.locked = true;
        resolve();
      } else {
        this.queue.push(resolve);
      }
    });
  }
  
  release() {
    if (this.queue.length > 0) {
      const next = this.queue.shift();
      next();
    } else {
      this.locked = false;
    }
  }
}
```

3. **Database transactions**: Use ACID properties
4. **Optimistic locking**: Version numbers, timestamps

### Why do we need locks API, such as Web Locks?

**Web Locks API** prevents race conditions when multiple tabs/workers access shared resources (IndexedDB, Cache API, etc.).

**Use cases:**
- Preventing concurrent writes to IndexedDB
- Coordinating cache updates
- Ensuring atomic operations across tabs

```javascript
// Without locks: race condition possible
async function updateCache() {
  const data = await cache.get('key');
  data.value++;
  await cache.put('key', data); // Another tab might overwrite
}

// With locks: safe
async function updateCache() {
  await navigator.locks.request('cache-key', async () => {
    const data = await cache.get('key');
    data.value++;
    await cache.put('key', data);
  });
}
```

### How can we use parallel programming primitives (semaphore, mutex, critical section, etc.) in async programming?

```javascript
// Semaphore: Limit concurrent operations
class Semaphore {
  constructor(count) {
    this.count = count;
    this.queue = [];
  }
  
  async acquire() {
    return new Promise((resolve) => {
      if (this.count > 0) {
        this.count--;
        resolve();
      } else {
        this.queue.push(resolve);
      }
    });
  }
  
  release() {
    this.count++;
    if (this.queue.length > 0) {
      const next = this.queue.shift();
      this.count--;
      next();
    }
  }
}

// Usage: Limit to 3 concurrent requests
const semaphore = new Semaphore(3);

async function limitedRequest(url) {
  await semaphore.acquire();
  try {
    return await fetch(url);
  } finally {
    semaphore.release();
  }
}
```

---

## Reactive Programming

### Tell us about «Reactive programming» paradigm.

**Reactive programming** is programming with asynchronous data streams and propagation of change.

**Key concepts:**
- **Observables**: Streams of values over time
- **Operators**: Transform, filter, combine streams
- **Subscribers**: React to stream events

**Benefits:**
- Declarative: Describe what, not how
- Composable: Chain operations
- Handles async naturally

### What is the difference between streams and signals approaches in reactive programming?

**Streams** (RxJS, most.js):
- Discrete events over time
- Can be empty, single, or multiple values
- Push-based: producer pushes to consumer
- Example: Mouse clicks, HTTP responses

**Signals** (Solid.js, Preact Signals):
- Current value + reactivity
- Always has a value
- Pull-based: consumer reads current value
- Example: Form state, computed values

```javascript
// Stream (RxJS)
const clicks$ = fromEvent(button, 'click');
clicks$.subscribe(click => console.log(click));

// Signal (concept)
const count = signal(0);
count.value++; // Triggers reactivity
console.log(count.value); // Read current value
```

---

## Streams

### Why are Streams useful to improve code semantics as a high-level abstraction?

**Streams** provide:
1. **Composability**: Chain operations (map, filter, reduce)
2. **Backpressure**: Handle slow consumers
3. **Memory efficiency**: Process data in chunks
4. **Unified API**: Same pattern for files, network, events

```javascript
// Without streams: load entire file
const data = fs.readFileSync('large.txt');
const processed = data.toString().split('\n').map(process);

// With streams: process in chunks
fs.createReadStream('large.txt')
  .pipe(split())
  .pipe(map(process))
  .pipe(writeStream);
```

### What is back pressure?

**Backpressure** occurs when data is produced faster than consumed. Streams handle this by pausing the producer when the buffer is full.

```javascript
// Producer (fast)
readable.on('data', (chunk) => {
  // Consumer (slow)
  await slowProcessing(chunk);
  // If buffer fills, readable pauses automatically
});

// Manual backpressure control
readable.on('data', (chunk) => {
  if (!writable.write(chunk)) {
    readable.pause(); // Pause when buffer full
  }
});

writable.on('drain', () => {
  readable.resume(); // Resume when buffer drains
});
```

### What is the difference between creating a Stream with extends vs. passing read, write, or transform function to a revealing constructor?

**Extending classes:**
```javascript
class MyReadable extends Readable {
  _read(size) {
    this.push(data);
    this.push(null); // End
  }
}
```

**Revealing constructor pattern:**
```javascript
const readable = new Readable({
  read(size) {
    this.push(data);
    this.push(null);
  }
});
```

**Differences:**
- **Revealing constructor**: More functional, less inheritance
- **Extending**: More OOP, can add methods/properties
- **Revealing constructor**: Preferred in modern code (simpler)

---

## Timers

### Why do we have three sets of timers: in the global context (e.g., setTimeout), node:timers, and node:timers.promises?

1. **Global timers** (`setTimeout`, `setInterval`):
   - Available everywhere
   - Callback-based
   - Part of Web/Node.js standard

2. **node:timers**:
   - Same functions, explicit import
   - Better for ESM modules
   - Explicit dependencies

3. **node:timers.promises**:
   - Promise-based versions
   - `setTimeout` returns Promise
   - Better with async/await

```javascript
// Global
setTimeout(() => console.log('done'), 1000);

// node:timers
const { setTimeout } = require('node:timers');
setTimeout(() => console.log('done'), 1000);

// node:timers.promises
const { setTimeout } = require('node:timers/promises');
await setTimeout(1000); // Promise-based
console.log('done');
```

---

## Promisification

### What promisified APIs do you know, and how can we manually promisify other APIs?

**Built-in promisified APIs:**
- `fs.promises` (Node.js)
- `util.promisify()` for callback-based APIs
- `fetch()` (browser/Node.js)
- `node:timers/promises`

**Manual promisification:**
```javascript
// Callback to Promise
function promisify(fn) {
  return function(...args) {
    return new Promise((resolve, reject) => {
      fn(...args, (error, result) => {
        if (error) reject(error);
        else resolve(result);
      });
    });
  };
}

const readFile = promisify(fs.readFile);
const data = await readFile('file.txt', 'utf8');

// EventEmitter to Promise
function once(emitter, event) {
  return new Promise((resolve) => {
    emitter.once(event, resolve);
  });
}

const data = await once(stream, 'data');
```

---

## Testing Async Code

### Tell us about testing of asynchronous code.

**Challenges:**
- Timing issues
- Race conditions
- Unhandled rejections
- Async cleanup

**Solutions:**

1. **Jest/Vitest async support**:
```javascript
test('async operation', async () => {
  const result = await asyncOperation();
  expect(result).toBe(expected);
});

test('promise rejection', async () => {
  await expect(asyncOperation()).rejects.toThrow();
});
```

2. **Fake timers**:
```javascript
jest.useFakeTimers();

test('timeout', async () => {
  const promise = delayedOperation();
  jest.advanceTimersByTime(1000);
  await promise;
});
```

3. **Mocking async functions**:
```javascript
jest.mock('./api', () => ({
  fetchData: jest.fn().mockResolvedValue({ data: 'test' })
}));
```

4. **Test utilities**:
```javascript
// Wait for condition
async function waitFor(condition, timeout = 5000) {
  const start = Date.now();
  while (!condition()) {
    if (Date.now() - start > timeout) {
      throw new Error('Timeout');
    }
    await new Promise(resolve => setImmediate(resolve));
  }
}
```

---

## TypeScript and Async

### Why can't TypeScript describe async contracts in all aspects?

**Limitations:**
1. **Promise types**: `Promise<T>` doesn't capture rejection types
2. **Error types**: No typed errors in catch blocks
3. **Thenable compatibility**: Hard to type-check
4. **Event types**: EventEmitter types are loose
5. **Stream types**: Complex to type accurately

**Workarounds:**
```typescript
// Custom Promise with error type (not native)
type Result<T, E = Error> = 
  | { success: true; value: T }
  | { success: false; error: E };

// Typed event emitter (library needed)
import { EventEmitter } from 'typed-event-emitter';
```

---

## Memory Leaks

### How can we prevent memory leaks in async code?

1. **Remove event listeners**:
```javascript
emitter.on('event', handler);
// Later:
emitter.off('event', handler);
```

2. **Clear intervals/timeouts**:
```javascript
const id = setInterval(() => {}, 1000);
clearInterval(id);
```

3. **Cancel async operations**:
```javascript
const controller = new AbortController();
fetch(url, { signal: controller.signal });
controller.abort(); // Cancel
```

4. **Close streams**:
```javascript
const stream = createReadStream('file.txt');
stream.on('end', () => stream.destroy());
```

5. **Avoid closures over large objects**:
```javascript
// Bad: Holds reference to large object
async function process() {
  const largeData = await loadData();
  setTimeout(() => {
    console.log(largeData); // Keeps largeData in memory
  }, 1000);
}

// Good: Extract needed data
async function process() {
  const largeData = await loadData();
  const summary = largeData.summary;
  setTimeout(() => {
    console.log(summary); // Only keeps summary
  }, 1000);
}
```

---

## Best Practices

### What are the best practices for managing concurrency in JavaScript?

1. **Limit concurrent operations**:
```javascript
// Use semaphore or p-limit
const limit = pLimit(3);
const results = await Promise.all(
  urls.map(url => limit(() => fetch(url)))
);
```

2. **Use Promise.all() for parallel operations**
3. **Use Promise.allSettled() when some may fail**
4. **Handle errors appropriately**
5. **Use AbortController for cancellation**
6. **Monitor resource usage**

### How can we use async/await with EventEmitter?

```javascript
// Convert event to Promise
function once(emitter, event) {
  return new Promise((resolve, reject) => {
    emitter.once(event, resolve);
    emitter.once('error', reject);
  });
}

// Usage
const data = await once(stream, 'data');

// Or use events.on() (Node.js 11+)
const { once } = require('events');
const data = await once(stream, 'data');
```

### What is the difference between EventEmitter and EventTarget?

| Feature | EventEmitter | EventTarget |
|---------|--------------|-------------|
| **Environment** | Node.js | Browser/Web Standard |
| **API** | `.on()`, `.emit()` | `.addEventListener()`, `.dispatchEvent()` |
| **Options** | More options (once, prependListener) | Standard options (once, capture) |
| **Performance** | Optimized for Node.js | Web standard |

**EventTarget** is the web standard; **EventEmitter** is Node.js-specific with more features.

### What is the role of the await keyword in async functions?

**await**:
1. Pauses execution until Promise settles
2. Unwraps Promise value (or throws if rejected)
3. Only works in async functions
4. Makes async code look synchronous

```javascript
async function example() {
  const value = await promise; // Pauses here
  return value; // Resumes after promise resolves
}
```

### What happens if we use await with non-promise values?

**await** converts non-promise values to resolved Promises:

```javascript
const value = await 42; // Same as await Promise.resolve(42)
console.log(value); // 42

const obj = await { data: 'test' }; // Works fine
console.log(obj); // { data: 'test' }
```

This allows consistent async/await usage even when values might be synchronous.

### How can we add timeouts in async operations (including await syntax)?

```javascript
// Using Promise.race()
async function withTimeout(promise, timeoutMs) {
  return Promise.race([
    promise,
    new Promise((_, reject) =>
      setTimeout(() => reject(new Error('Timeout')), timeoutMs)
    )
  ]);
}

// Usage
try {
  const result = await withTimeout(fetch('/api'), 5000);
} catch (error) {
  if (error.message === 'Timeout') {
    // Handle timeout
  }
}

// Using AbortController
async function fetchWithTimeout(url, timeoutMs) {
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), timeoutMs);
  
  try {
    return await fetch(url, { signal: controller.signal });
  } finally {
    clearTimeout(timeout);
  }
}
```

### What are the implications of the process.nextTick method?

**process.nextTick()**:
- Executes callback in the **current phase** (before next event loop phase)
- Higher priority than `setImmediate()` and `setTimeout()`
- Can starve the event loop if used recursively
- Useful for ensuring code runs after current operation but before I/O

```javascript
console.log('1');
process.nextTick(() => console.log('2'));
console.log('3');
// Output: 1, 3, 2

// Can cause starvation
function starve() {
  process.nextTick(starve); // Blocks event loop
}
```

**Best practice**: Use `setImmediate()` for deferring to next iteration, `process.nextTick()` only when you need immediate execution in current phase.

---

## Custom Async Iterables

### How can we create custom async iterables and what are their use cases?

```javascript
// Async iterable object
const asyncIterable = {
  async *[Symbol.asyncIterator]() {
    for (let i = 0; i < 5; i++) {
      await new Promise(resolve => setTimeout(resolve, 100));
      yield i;
    }
  }
};

// Usage
for await (const value of asyncIterable) {
  console.log(value);
}

// Use cases:
// - Paginated API responses
// - Database query results
// - File line-by-line reading
// - Real-time data streams
```

---

## Third-Party Libraries

### What are the advantages and disadvantages of using third-party async libraries like Promise polyfills and async.js?

**Advantages:**
- Additional utilities (retry, timeout, concurrency control)
- Better browser compatibility (polyfills)
- Advanced patterns (waterfall, series, parallel)

**Disadvantages:**
- Extra dependencies
- Bundle size increase
- Native APIs often sufficient
- Maintenance overhead

**Modern approach**: Use native Promises and async/await, add libraries only when needed (e.g., `p-limit` for concurrency).

---

## Legacy Systems

### How can we handle async code in legacy systems?

1. **Wrap callbacks in Promises**:
```javascript
const util = require('util');
const readFile = util.promisify(fs.readFile);
```

2. **Gradual migration**: Mix callbacks and Promises
3. **Adapter patterns**: Create Promise-based wrappers
4. **Use libraries**: `bluebird` for enhanced Promises

---

## Async vs Parallel vs I/O

### What is the difference between asynchronous, parallel, and I/O operations?

- **Asynchronous**: Non-blocking, doesn't wait for completion
- **Parallel**: Multiple operations simultaneously (requires multiple threads/processes)
- **I/O operations**: Input/Output (file, network, database)

**In JavaScript:**
- Single-threaded: Async operations are concurrent, not parallel
- I/O is async by nature (non-blocking)
- True parallelism requires Workers/Threads

### How can we parallelize I/O operations effectively?

```javascript
// Sequential (slow)
for (const url of urls) {
  await fetch(url);
}

// Parallel (fast)
await Promise.all(urls.map(url => fetch(url)));

// Limited concurrency (balanced)
const limit = pLimit(5);
await Promise.all(urls.map(url => limit(() => fetch(url))));
```

### How can we ensure thread safety in async programming?

**JavaScript is single-threaded**, so no thread safety issues in main thread. However:

1. **SharedArrayBuffer + Atomics**: For Worker Threads
2. **Message passing**: Workers communicate via messages
3. **No shared mutable state**: Each context has its own state

```javascript
// Worker Threads with SharedArrayBuffer
const sharedBuffer = new SharedArrayBuffer(1024);
const view = new Int32Array(sharedBuffer);

// Use Atomics for thread-safe operations
Atomics.add(view, 0, 1); // Atomic increment
```

### How are Atomics related to asynchronous and parallel programming? What are they used for?

**Atomics** provide atomic operations on SharedArrayBuffer for Worker Threads:

- **Atomic operations**: Guaranteed to complete without interruption
- **Synchronization**: Coordinate between threads
- **Use cases**: Counters, locks, flags, coordination

```javascript
// Atomic counter
const buffer = new SharedArrayBuffer(4);
const view = new Int32Array(buffer);

// Thread-safe increment
Atomics.add(view, 0, 1);

// Wait and notify (coordination)
Atomics.wait(view, 0, 0); // Wait for value to change
Atomics.notify(view, 0, 1); // Notify waiting threads
```

---

## Performance Optimization

### How can we optimize async code for performance?

1. **Parallelize independent operations**:
```javascript
// Bad: Sequential
const a = await op1();
const b = await op2();
const c = await op3();

// Good: Parallel
const [a, b, c] = await Promise.all([op1(), op2(), op3()]);
```

2. **Limit concurrency**: Use semaphores
3. **Cache results**: Memoization
4. **Use streams for large data**
5. **Debounce/throttle**: Reduce operation frequency
6. **Batch operations**: Group multiple operations

### How can we handle retries (calls, calculations, resource access) in async programming?

```javascript
async function retry(fn, maxAttempts = 3, delay = 1000) {
  for (let i = 0; i < maxAttempts; i++) {
    try {
      return await fn();
    } catch (error) {
      if (i === maxAttempts - 1) throw error;
      await new Promise(resolve => setTimeout(resolve, delay));
      delay *= 2; // Exponential backoff
    }
  }
}

// Usage
const result = await retry(() => fetch('/api'));
```

### What are the common pitfalls of async programming?

1. **Forgetting await**: Unhandled promises
2. **Error handling**: Not catching rejections
3. **Race conditions**: Order-dependent operations
4. **Memory leaks**: Not cleaning up listeners
5. **Blocking event loop**: CPU-intensive work
6. **Nested awaits**: Not parallelizing when possible
7. **Promise constructor anti-pattern**: Wrapping already-promise values

```javascript
// Pitfall: Missing await
async function example() {
  fetch('/api'); // Missing await - promise not handled
}

// Pitfall: Not parallelizing
const a = await op1(); // Wait
const b = await op2(); // Wait
// Should be: await Promise.all([op1(), op2()])
```

### How can we use async functions with Array.prototype.map?

```javascript
// Sequential (if you need order)
const results = [];
for (const item of array) {
  results.push(await process(item));
}

// Parallel (if order doesn't matter)
const results = await Promise.all(
  array.map(item => process(item))
);

// Limited concurrency
const limit = pLimit(5);
const results = await Promise.all(
  array.map(item => limit(() => process(item)))
);
```

### How can we debug async code effectively?

1. **Use async stack traces**: Node.js `--async-stack-traces`
2. **Add logging**: Track async flow
3. **Use debugger**: Set breakpoints in async functions
4. **Monitor unhandled rejections**:
```javascript
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled rejection:', reason);
});
```

5. **Use AsyncLocalStorage**: Track request context
6. **Performance profiling**: Use `perf_hooks`

### How can we ensure data consistency in async operations?

1. **Transactions**: Database transactions
2. **Locks**: Mutex/semaphore for critical sections
3. **Idempotency**: Make operations safe to retry
4. **Optimistic locking**: Version numbers
5. **Atomic operations**: Use database atomic operations

### What are the benefits of using async/await over callbacks?

1. **Readability**: Sequential-looking code
2. **Error handling**: try/catch instead of error callbacks
3. **Stack traces**: Better debugging
4. **Composability**: Easier to chain operations
5. **Less nesting**: Flatter code structure

### Which operations can't be rewritten from callbacks to async/await syntax (but are possible with Promises)?

**Event listeners** (though you can use `once()` helper):
```javascript
// Callback pattern (hard to convert)
emitter.on('event', handler); // Multiple events

// Can use helper
const { once } = require('events');
const data = await once(emitter, 'event'); // Single event
```

**Streams** (can use async iteration):
```javascript
// Callback
stream.on('data', handler);

// Async iteration
for await (const chunk of stream) {
  handler(chunk);
}
```

Most callback patterns can be converted with helpers or async iteration.

---

## AbortSignal

### Propose use cases for AbortSignal.timeout(). Which well-known APIs support it?

**AbortSignal.timeout()** creates a signal that aborts after specified time:

```javascript
// Automatic timeout
const signal = AbortSignal.timeout(5000);
fetch('/api', { signal }); // Aborts after 5 seconds

// Use cases:
// - API request timeouts
// - Long-running operations
// - User-initiated cancellations
```

**APIs supporting AbortSignal:**
- `fetch()`
- `ReadableStream`
- `WritableStream`
- `FileReader` (some browsers)
- Many Node.js stream APIs

### Where and for what purposes can we use AbortSignal.any(iterable)?

**AbortSignal.any()** creates a signal that aborts when any input signal aborts:

```javascript
// Abort if any condition is met
const userCancel = new AbortController();
const timeout = AbortSignal.timeout(10000);
const combined = AbortSignal.any([userCancel.signal, timeout]);

fetch('/api', { signal: combined });
// Aborts if user cancels OR timeout occurs

// Use cases:
// - Multiple cancellation conditions
// - Race between user action and timeout
// - Coordinated cancellation across operations
```

---

## Promise Methods

### What are the differences between Promise methods: resolve and reject?

**Promise.resolve()**:
- Creates resolved Promise
- Converts Thenables to Promises
- Returns Promise with value

**Promise.reject()**:
- Creates rejected Promise
- Always returns rejected Promise (even with non-Error values)

```javascript
Promise.resolve(42); // Resolved with 42
Promise.resolve(thenable); // Converts Thenable

Promise.reject('error'); // Rejected with 'error'
Promise.reject(new Error('error')); // Rejected with Error
```

### How can we handle errors in Promise.all?

```javascript
// Option 1: Catch individual promises
const results = await Promise.all(
  promises.map(p => p.catch(error => ({ error })))
);

// Option 2: Use Promise.allSettled
const results = await Promise.allSettled(promises);
results.forEach((result, i) => {
  if (result.status === 'rejected') {
    console.error(`Promise ${i} failed:`, result.reason);
  }
});

// Option 3: Wrap in try/catch (stops on first error)
try {
  const results = await Promise.all(promises);
} catch (error) {
  // First rejection stops all
}
```

### How can we chain async operations? (Please propose cases for as many contracts as you know)

**1. Promises:**
```javascript
promise
  .then(value => process1(value))
  .then(value => process2(value))
  .then(value => process3(value));
```

**2. async/await:**
```javascript
const v1 = await operation1();
const v2 = await process1(v1);
const v3 = await process2(v2);
return process3(v3);
```

**3. Callbacks:**
```javascript
operation1((err, v1) => {
  if (err) return handle(err);
  process1(v1, (err, v2) => {
    if (err) return handle(err);
    process2(v2, (err, v3) => {
      if (err) return handle(err);
      process3(v3);
    });
  });
});
```

**4. Async generators:**
```javascript
async function* chain() {
  const v1 = yield await operation1();
  const v2 = yield await process1(v1);
  const v3 = yield await process2(v2);
  yield await process3(v3);
}
```

**5. Streams:**
```javascript
readable
  .pipe(transform1)
  .pipe(transform2)
  .pipe(transform3)
  .pipe(writable);
```

---

## Event Loop

### What is the role of the event loop in async programming?

**Event loop**:
- Manages execution of async operations
- Processes callbacks when operations complete
- Ensures non-blocking I/O
- Coordinates microtasks and macrotasks

**Flow:**
1. Execute synchronous code
2. Process microtasks (Promises)
3. Process macrotasks (timers, I/O)
4. Repeat

Without event loop, JavaScript would block on I/O operations.

---

## Long-Running Operations

### How can we handle long-running async operations? (Processes may exit, results may become obsolete, etc.)

1. **Persistence**: Save state to database
2. **Job queues**: Use external queue (Bull, BullMQ)
3. **Checkpoints**: Save progress periodically
4. **Timeouts**: Abort if too long
5. **Status endpoints**: Check operation status

```javascript
// Save progress
async function longOperation(jobId) {
  const checkpoints = [0, 25, 50, 75, 100];
  for (let i = 0; i < 100; i++) {
    await processItem(i);
    if (checkpoints.includes(i)) {
      await saveProgress(jobId, i);
    }
  }
}

// With timeout
const signal = AbortSignal.timeout(3600000); // 1 hour
await longOperation({ signal });
```

---

## Idempotency

### How can we ensure idempotency in async operations and when do we need it?

**Idempotency**: Operation produces same result when called multiple times.

**Techniques:**
1. **Idempotency keys**: Unique identifier per operation
2. **Check before execute**: Verify if already done
3. **Idempotent operations**: Design operations to be safe to retry

```javascript
// With idempotency key
async function processPayment(idempotencyKey, amount) {
  // Check if already processed
  const existing = await db.findPayment(idempotencyKey);
  if (existing) return existing;
  
  // Process and save key
  const payment = await createPayment(amount);
  await db.savePayment(idempotencyKey, payment);
  return payment;
}
```

**When needed:**
- Payment processing
- API calls that might retry
- Distributed systems
- Network operations

---

## Real-Time Applications

### Can we write a real-time application in JavaScript and asynchronous programming?

**Yes!** Examples:
- WebSockets for bidirectional communication
- Server-Sent Events (SSE) for server push
- WebRTC for peer-to-peer
- Long polling

```javascript
// WebSocket example
const ws = new WebSocket('ws://server');
ws.onmessage = (event) => {
  updateUI(JSON.parse(event.data));
};

// Server
wss.on('connection', (ws) => {
  setInterval(() => {
    ws.send(JSON.stringify({ time: Date.now() }));
  }, 1000);
});
```

---

## Ordering Async Operations

### How can we ensure the order of async operations? Please suggest cases in which we might experience problems.

**Sequential execution:**
```javascript
// Ensures order
const a = await op1();
const b = await op2(a); // Waits for op1
const c = await op3(b); // Waits for op2
```

**Problems:**
1. **Race conditions**: Operations complete out of order
2. **Parallel execution**: `Promise.all()` doesn't guarantee order (though results array is ordered)
3. **Event ordering**: Events might arrive out of order

```javascript
// Problem: Race condition
let counter = 0;
await Promise.all([
  increment(), // Might execute after decrement
  decrement()
]);

// Solution: Sequential or use locks
await increment();
await decrement();
```

---

## High Availability

### How can we handle async code in a high-availability system?

1. **Error handling**: Comprehensive try/catch
2. **Retries**: Exponential backoff
3. **Circuit breakers**: Stop calling failing services
4. **Timeouts**: Don't wait indefinitely
5. **Health checks**: Monitor system status
6. **Graceful degradation**: Fallback options
7. **Load balancing**: Distribute load

```javascript
// Circuit breaker pattern
class CircuitBreaker {
  constructor(threshold = 5) {
    this.failures = 0;
    this.state = 'closed';
  }
  
  async call(fn) {
    if (this.state === 'open') {
      throw new Error('Circuit open');
    }
    
    try {
      const result = await fn();
      this.failures = 0;
      return result;
    } catch (error) {
      this.failures++;
      if (this.failures >= this.threshold) {
        this.state = 'open';
        setTimeout(() => this.state = 'closed', 60000);
      }
      throw error;
    }
  }
}
```

---

## Observables

### What are observables and how can we use them in JavaScript?

**Observables** (RxJS) represent streams of values:

```javascript
import { Observable } from 'rxjs';

const observable = new Observable(subscriber => {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.next(3);
  subscriber.complete();
});

observable.subscribe({
  next: value => console.log(value),
  complete: () => console.log('Done')
});

// Operators
import { fromEvent } from 'rxjs';
import { map, filter, debounceTime } from 'rxjs/operators';

fromEvent(button, 'click')
  .pipe(
    debounceTime(300),
    map(event => event.target.value),
    filter(value => value.length > 3)
  )
  .subscribe(value => console.log(value));
```

**Use cases:**
- Event handling
- Data streams
- Reactive UI updates
- Complex async flows

---

## State Management

### What are the main problems of handling state in asynchronous code in a stateful application?

1. **Race conditions**: Updates arrive out of order
2. **Stale state**: Reading state before updates complete
3. **Lost updates**: Concurrent modifications
4. **Inconsistent state**: Partial updates

**Solutions:**
- Immutable updates
- State machines
- Optimistic updates with rollback
- Transactions
- Event sourcing

```javascript
// Problem: Race condition
let state = { count: 0 };
async function increment() {
  const current = state.count;
  await delay(100);
  state.count = current + 1; // Might use stale value
}

// Solution: Atomic updates
async function increment() {
  await mutex.acquire();
  try {
    state.count++;
  } finally {
    mutex.release();
  }
}
```

---

## Queues

### When can we use internal async queues and when do we need external queue systems?

**Internal queues** (in-memory):
- Single process
- Small workloads
- No persistence needed
- Fast, simple

```javascript
class Queue {
  constructor() {
    this.queue = [];
    this.processing = false;
  }
  
  async add(task) {
    this.queue.push(task);
    this.process();
  }
  
  async process() {
    if (this.processing) return;
    this.processing = true;
    while (this.queue.length > 0) {
      const task = this.queue.shift();
      await task();
    }
    this.processing = false;
  }
}
```

**External queues** (Redis, RabbitMQ, etc.):
- Multiple processes/servers
- Persistence required
- Large scale
- Job scheduling
- Retries, priorities

---

## Caching and Memoization

### How can we use async functions with caching, memoization, and recalculations on state updates?

```javascript
// Memoization
const cache = new Map();

async function memoized(key, fn) {
  if (cache.has(key)) {
    return cache.get(key);
  }
  const result = await fn();
  cache.set(key, result);
  return result;
}

// With TTL
class AsyncCache {
  constructor(ttl = 60000) {
    this.cache = new Map();
    this.ttl = ttl;
  }
  
  async get(key, fn) {
    const cached = this.cache.get(key);
    if (cached && Date.now() < cached.expiry) {
      return cached.value;
    }
    const value = await fn();
    this.cache.set(key, {
      value,
      expiry: Date.now() + this.ttl
    });
    return value;
  }
  
  invalidate(key) {
    this.cache.delete(key);
  }
}
```

---

## Database Connections

### How can we use async functions with database connections and what are the use cases?

```javascript
// Connection pool
const pool = new Pool({
  max: 20,
  idleTimeoutMillis: 30000
});

async function query(sql, params) {
  const client = await pool.connect();
  try {
    const result = await client.query(sql, params);
    return result.rows;
  } finally {
    client.release();
  }
}

// Transaction
async function transfer(from, to, amount) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    await client.query('UPDATE accounts SET balance = balance - $1 WHERE id = $2', [amount, from]);
    await client.query('UPDATE accounts SET balance = balance + $1 WHERE id = $2', [amount, to]);
    await client.query('COMMIT');
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

---

## Separation of Concerns

### How can we separate async code from business logic and why might we want to do this?

**Benefits:**
- Testability: Mock async operations
- Reusability: Business logic independent of I/O
- Clarity: Separate concerns

```javascript
// Bad: Mixed concerns
async function processOrder(orderId) {
  const order = await db.getOrder(orderId);
  const user = await db.getUser(order.userId);
  const total = order.items.reduce((sum, item) => sum + item.price, 0);
  await db.updateOrder(orderId, { total });
  await email.send(user.email, 'Order processed');
}

// Good: Separated
// Business logic (pure)
function calculateTotal(items) {
  return items.reduce((sum, item) => sum + item.price, 0);
}

// Async operations (I/O)
async function processOrder(orderId) {
  const order = await db.getOrder(orderId);
  const user = await db.getUser(order.userId);
  const total = calculateTotal(order.items); // Pure function
  await db.updateOrder(orderId, { total });
  await email.send(user.email, 'Order processed');
}
```

---

## CPU vs I/O Bound

### What is the impact of async code on CPU-bound vs I/O-bound operations?

**I/O-bound operations:**
- Async is perfect: Non-blocking, efficient
- Event loop handles many concurrent operations
- Example: Network requests, file I/O

**CPU-bound operations:**
- Async doesn't help: Still blocks event loop
- Need Worker Threads for true parallelism
- Example: Image processing, calculations

```javascript
// I/O-bound: Async works great
async function fetchMultiple(urls) {
  return Promise.all(urls.map(url => fetch(url)));
}

// CPU-bound: Needs Worker Thread
const { Worker } = require('worker_threads');
function processImage(data) {
  return new Promise((resolve, reject) => {
    const worker = new Worker('./image-processor.js', {
      workerData: data
    });
    worker.on('message', resolve);
    worker.on('error', reject);
  });
}
```

---

## Security

### What are the security considerations in async programming?

1. **Timing attacks**: Async operations might leak timing information
2. **Race conditions**: Security checks might be bypassed
3. **Resource exhaustion**: Too many concurrent operations
4. **Error information**: Don't leak sensitive data in errors

```javascript
// Problem: Race condition in auth check
async function updateUser(userId, data) {
  const user = await db.getUser(userId);
  if (!user.isAdmin) throw new Error('Unauthorized');
  // Race condition: user might change between check and update
  await db.updateUser(userId, data);
}

// Solution: Atomic check
async function updateUser(userId, data) {
  const result = await db.updateUser(userId, data, {
    where: { isAdmin: true } // Atomic check
  });
  if (result.affectedRows === 0) {
    throw new Error('Unauthorized');
  }
}
```

---

## Priority Queue

### How can we implement a priority queue for async tasks?

```javascript
class PriorityQueue {
  constructor() {
    this.queue = [];
  }
  
  enqueue(task, priority) {
    this.queue.push({ task, priority });
    this.queue.sort((a, b) => b.priority - a.priority); // Higher priority first
  }
  
  async dequeue() {
    if (this.queue.length === 0) return null;
    const { task } = this.queue.shift();
    return task();
  }
  
  async process() {
    while (this.queue.length > 0) {
      await this.dequeue();
    }
  }
}

// Usage
const queue = new PriorityQueue();
queue.enqueue(() => fetch('/api/low'), 1);
queue.enqueue(() => fetch('/api/high'), 10);
await queue.process(); // High priority first
```

---

## File System Operations

### How can we use async functions with file system operations?

```javascript
const fs = require('fs').promises;

// Read file
const data = await fs.readFile('file.txt', 'utf8');

// Write file
await fs.writeFile('output.txt', data);

// Read directory
const files = await fs.readdir('/path/to/dir');

// Stream for large files
const stream = fs.createReadStream('large.txt');
for await (const chunk of stream) {
  process(chunk);
}
```

---

## Atomicity

### How can we ensure atomicity in async operations and what for?

**Atomicity**: Operations either complete fully or not at all.

**Techniques:**
1. **Database transactions**
2. **Locks**: Ensure exclusive access
3. **Idempotency keys**: Prevent duplicate operations

```javascript
// Atomic transfer
async function transfer(from, to, amount) {
  const client = await db.getConnection();
  try {
    await client.query('BEGIN');
    await client.query('UPDATE accounts SET balance = balance - $1 WHERE id = $2', [amount, from]);
    await client.query('UPDATE accounts SET balance = balance + $1 WHERE id = $2', [amount, to]);
    await client.query('COMMIT');
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  }
}
```

**Why needed:**
- Financial transactions
- Data consistency
- Preventing partial updates

---

## Trade-offs

### What are the trade-offs between using Promise and async/await?

| Aspect | Promise | async/await |
|--------|---------|-------------|
| **Readability** | Can be verbose | More readable |
| **Error handling** | `.catch()` | try/catch |
| **Composability** | `.then()` chains | Sequential code |
| **Debugging** | Stack traces can be truncated | Better stack traces |
| **Control flow** | More flexible | More linear |

**Best practice**: Use async/await for most code, Promises for utilities and composition.

---

## RxJS vs Simple Async

### What is the difference between simple async programming and the RxJS approach?

**Simple async:**
- Promises for single values
- async/await for sequential code
- Straightforward, easy to understand

**RxJS:**
- Observables for streams of values
- Rich operators for transformation
- More powerful but steeper learning curve

```javascript
// Simple async
async function fetchData() {
  const data = await fetch('/api');
  return data.json();
}

// RxJS
import { from } from 'rxjs';
import { map, filter, debounceTime } from 'rxjs/operators';

from(fetch('/api'))
  .pipe(
    map(response => response.json()),
    filter(data => data.active),
    debounceTime(300)
  )
  .subscribe(data => console.log(data));
```

**When to use RxJS:**
- Complex event streams
- Need advanced operators
- Real-time data processing
- Multiple sources/transformations

---

## Async Collections

### What are async collections and how can they improve developer experience?

**Async collections** provide array-like methods for async iterables:

```javascript
// Concept: Async array methods
class AsyncArray {
  constructor(iterable) {
    this.iterable = iterable;
  }
  
  async map(fn) {
    const results = [];
    for await (const item of this.iterable) {
      results.push(await fn(item));
    }
    return results;
  }
  
  async filter(fn) {
    const results = [];
    for await (const item of this.iterable) {
      if (await fn(item)) {
        results.push(item);
      }
    }
    return results;
  }
}

// Usage
const asyncArray = new AsyncArray(fetchPages('/api'));
const processed = await asyncArray
  .map(item => process(item))
  .filter(item => item.valid);
```

**Benefits:**
- Familiar API (like Array methods)
- Composable operations
- Better readability
- Handles async iteration automatically

---

## Conclusion

This guide covers the major aspects of asynchronous programming in JavaScript and Node.js. Key takeaways:

1. **Use async/await** for most async code
2. **Understand the event loop** for performance
3. **Handle errors properly** with try/catch or .catch()
4. **Use appropriate patterns** (Promises, streams, events) for different scenarios
5. **Consider performance** (parallelization, backpressure, concurrency limits)
6. **Test thoroughly** - async code has unique challenges
7. **Monitor and debug** using available tools

Mastering these concepts will help you write efficient, maintainable, and robust asynchronous code.

