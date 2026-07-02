# Chapter 8: Concurrency (Frontends often ignore it, production never will) ⚙️

> Your `outline.md` is painfully accurate: many frontend developers have almost zero concurrency model in their head.
> But concurrency is foundational to “can this system hold up”.
> AI can generate code, but concurrency problems are usually: the program becomes slow, stuck, deadlocked, or CPU-bound at runtime.
> You need a mental model, instincts, and a debugging path.

This chapter follows your outline:

- Process / Thread / Coroutine / Async / Event Loop
- then: thread pool / object pool / connection pool / Worker / Cluster
- CPU-bound vs IO-bound
- why Java needs thread pools, why Node “seems not to”
- why Node “doesn’t use threads” (and what libuv actually does)
- why image compression should not run on the main thread

---

## The Three Questions

### 1) Why do concurrency problems exist? 🤔

Because the real world does not send requests one-by-one in a neat queue:

- many users click at once
- cron jobs run at the same time
- IO completes at random times
- CPU cores are contested

If your system only knows “do one thing at a time”, you will see:

- latency spikes
- throughput collapses
- queues grow like holiday traffic

So we need concurrency:

> **Handle many tasks, and do not waste time waiting on IO.**

### 2) Why these concurrency models? Are there alternatives? 🧰

Common models:

- multi-process: strong isolation, higher overhead
- multi-thread: shared memory, requires synchronization
- coroutines/async: turns waiting into schedulable time (great for IO-bound work)
- event loop: single-thread scheduling for lots of IO

All models fight the same core constraint:

> **We want to do many things, but CPU and resources are finite.**

### 3) How does it work in practice (for developers)? 🔍

From a developer viewpoint, the key skills are:

1. classify work: CPU-bound vs IO-bound
2. understand your runtime model (especially Node)
3. use the right tools to move heavy work off the critical path (workers/queues/pools/async)

---

## Part 0: Fix your “concurrency worldview” 🧠

### 0.1 Concurrency vs parallelism

- concurrency: looks like doing many things (scheduling)
- parallelism: actually doing many things at the same time (multiple CPU cores)

```text
Concurrency: one cook handles 3 pots by switching
Parallelism: 3 cooks each handle one pot
```

### 0.2 “Slow” is often waiting, not computation

Many production slowdowns are not “the algorithm is too slow”.

They are:

- waiting for DB
- waiting for network
- waiting for disk
- waiting for locks
- waiting for a pool

Concurrency helps by:

> letting the CPU do other work while some tasks are waiting.

---

## Part 1: Process 🏭

A process is like an independent factory:

- separate memory space
- strong isolation
- one crash does not necessarily take others down

```text
Process A: memory A + resources A
Process B: memory B + resources B
```

Trade-offs:

- more expensive to create/switch
- communication needs IPC

Good for:

- isolation
- parallelism across cores

---

## Part 2: Thread 🧵

Threads are like workers inside one factory:

- threads in a process share memory
- sharing is fast
- sharing is also dangerous (race conditions) -> locks are needed

```text
Process
  |
  +-- Thread 1 (shared memory)
  +-- Thread 2 (shared memory)
  +-- Thread 3 (shared memory)
```

Pros:

- fast communication
- strong parallelism

Cons:

- thread-safety complexity
- debugging “sometimes” bugs is painful

---

## Part 3: Coroutine 🧶

A coroutine is:

> a lightweight task that can be suspended and resumed.

Often scheduled by the language/runtime, not directly by the OS.

Instinct:

- threads are OS-level workers
- coroutines are application-level flows

Coroutines shine for IO-bound workloads.

---

## Part 4: Async ⏳

Async is not “faster by default”.

Async is:

> turning waiting into manageable states, so you can do other work while waiting.

```text
Sync: do A -> wait A -> do B -> wait B
Async: start A -> do B -> when A completes, handle it
```

---

## Part 5: Event Loop (Node’s traffic controller) 🧑‍✈️

Node’s classic model is: event loop + non-blocking IO.

Think of the event loop as a single-thread dispatcher:

- you schedule work
- IO is delegated to the OS / a thread pool
- when IO completes, callbacks are queued
- the main thread executes callbacks one by one

```text
Main thread (Event Loop)
   |
   +-- start IO (network/disk)
   |
   +-- run other callbacks
   |
   +-- IO completes -> callback queued -> executed on main thread
```

Plain summary:

> Node is great at handling many IO-waiting tasks.
> Node is not great at heavy CPU computation on the main thread.

---

## Part 6: CPU-bound vs IO-bound (your first decision tool) 📏

### 6.1 CPU-bound

Characteristics:

- time is spent computing
- CPU usage is high
- single-thread can get saturated

Examples:

- image compression
- video transcoding
- heavy crypto
- large JSON parse/serialize

### 6.2 IO-bound

Characteristics:

- time is spent waiting
- CPU might be low

Examples:

- DB queries
- third-party API calls
- file reads/writes

```text
CPU-bound: CPU is busy
IO-bound: CPU is waiting
```

---

## Part 7: Why Java has thread pools, why Node “seems not to” 🏊

### 7.1 Java (simplified)

Java servers often handle requests with threads:

- a request uses a thread (from a pool)
- blocking IO blocks that thread

Threads are not free, so you need a pool:

> threads are expensive and cannot be infinite.

### 7.2 Node (simplified)

Node tries to avoid blocking the main thread.

It uses the event loop to manage lots of connections and delegates IO.

When people say “Node is single-threaded”, they mostly mean:

> JavaScript execution runs on one main thread.

But Node is not “no threads at all”.

---

## Part 8: What does libuv do? 🧰

libuv provides:

- the event loop
- async IO abstraction
- a thread pool for certain blocking tasks

Instinct:

> Node’s “async power” often comes from libuv doing the dirty work underneath.

---

## Part 9: Why image compression must not run on the main thread 📸

It is a classic CPU-bound task.

If you do heavy CPU work on the Node main thread:

- the event loop stalls
- all requests queue up
- users experience “the whole site is frozen”

```text
Main thread compresses images (CPU heavy)
   |
   v
Other callbacks wait in queue
   |
   v
Latency spikes
```

So CPU-heavy work should go to:

- worker threads
- or a separate service / queue

---

## Part 10: Worker / Cluster (how Node uses multiple cores) 🧑‍🔧

### 10.1 Worker Threads: move CPU-heavy work away

Node has `worker_threads` to run CPU-heavy work off the main thread.

Minimal example (main sends work, worker replies):

`main.js`:

```js
import { Worker } from "node:worker_threads";

function runWorker(input) {
  return new Promise((resolve, reject) => {
    const worker = new Worker(new URL("./worker.js", import.meta.url), {
      workerData: input,
    });
    worker.on("message", resolve);
    worker.on("error", reject);
    worker.on("exit", (code) => {
      if (code !== 0) reject(new Error(`Worker exited with code ${code}`));
    });
  });
}

const result = await runWorker({ n: 42 });
console.log(result);
```

`worker.js`:

```js
import { parentPort, workerData } from "node:worker_threads";

function fib(n) {
  return n <= 1 ? n : fib(n - 1) + fib(n - 2);
}

parentPort.postMessage({ n: workerData.n, fib: fib(workerData.n) });
```

### 10.2 Cluster: multi-process to increase throughput

Cluster is a multi-process mode:

- a master process forks worker processes
- workers share the same listening port

It is like “multiple store branches” serving customers in parallel.

---

## Part 11: Pools (thread/object/connection pools) = “cap + reuse + queue” 🧠

All pools share the same underlying idea:

> resources are expensive; do not create infinitely; reuse and cap the maximum.

```text
Requests arrive
  |
  v
[Pool]
  | idle resource -> borrow
  | no resource   -> queue/reject/timeout
  v
Work runs
  |
  v
Resource returned
```

---

## Part 12: Dev practice: writing concurrency without melting systems 🧯

### 12.1 Do not blindly Promise.all huge arrays

This:

```js
await Promise.all(items.map((item) => fetchSomething(item)));
```

with large `items` can:

- blow up DB pools
- overload downstream services
- blow up memory

Correct direction: limit concurrency.

```text
Tasks -> queue -> run at most N concurrently -> continue
```

### 12.2 Backpressure: “I cannot eat more right now”

Concurrency control is essentially backpressure:

- downstream is slow -> upstream should not keep flooding

This idea shows up everywhere:

- Nginx rate limiting
- app queues
- DB connection pools

---

## Extra 1: What do concurrency bugs look like in real life? (race, deadlock, starvation) 🐛

You do not need to memorize theory first. You need to recognize failure patterns.

### 1) Race conditions

Two tasks touch shared state, and the final result depends on timing.

```text
shared balance = 100

Task A reads 100 -> plans to write 80
Task B reads 100 -> plans to write 50

Without synchronization:
final can be 80 or 50 (lost update)
```

The nasty part:

> often you cannot reproduce it locally; it appears “sometimes” in production. 😵

### 2) Deadlocks

```text
Task A holds lock1 -> waits for lock2
Task B holds lock2 -> waits for lock1
=> deadlock
```

### 3) Starvation

Some requests/tasks never get resources because others keep cutting in line.

---

## Extra 2: Synchronization primitives in plain English (mutex, semaphore, queue) 🔧

### 1) Mutex

Meaning:

> only one worker can enter a critical section at a time.

### 2) Semaphore

Meaning:

> allow up to N workers to enter at the same time.

This maps directly to “concurrency limits”.

### 3) Queue

Meaning:

> you line up; I process in order.

Many concurrency solutions are less about “faster” and more about “stable”.

---

## Extra 3: Node event loop details: why timers drift, why Promises “cut the line” ⏱️

You do not need to memorize every phase, but you need two instincts:

### 1) Microtasks often run “as soon as possible”

Promise callbacks (microtasks) often run before `setTimeout(fn, 0)` (macrotask).

### 2) setTimeout is not a precise alarm clock

`setTimeout(fn, 100)` means:

> run it after ~100ms _when the main thread is free_.

If the event loop is blocked, timers will be delayed.

---

## Extra 4: How do you detect a blocked event loop? (very useful in production) 📉

A practical metric is event loop lag.

Instinct:

- healthy: lag stays small
- blocked: lag spikes

Simple idea:

```text
schedule a tick every 100ms
measure how late it actually fires
the lateness = event loop lag
```

This helps you quickly distinguish:

> “DB/network is slow” vs “we blocked the main thread with CPU work”.

---

## Extra 5: A minimal concurrency limiter (stop using Promise.all as a self-DDoS) 🚦

Here is a minimal “map with concurrency” implementation without external libs:

```js
export async function mapWithConcurrency(items, concurrency, worker) {
  const results = new Array(items.length);
  let nextIndex = 0;

  async function run() {
    while (true) {
      const current = nextIndex;
      if (current >= items.length) return;
      nextIndex += 1;
      results[current] = await worker(items[current], current);
    }
  }

  const runners = Array.from({ length: concurrency }, () => run());
  await Promise.all(runners);
  return results;
}
```

This is a “forever useful” tool for batch processing in real systems.

---

## Extra 6: Workers are better as a pool 🧑‍🏭

In real life, CPU-heavy tasks are often continuous (compression, encryption, media, some AI preprocessing).

You usually do not want to `new Worker()` for every task.

You want:

> a fixed-size worker pool + a task queue.

```text
Task queue -> Worker1
          -> Worker2
          -> Worker3
```

This is the same idea as pools everywhere:

- reuse resources
- enforce a hard cap
- queue and timeout

---

## Extra 7: Backpressure shows up as Streams in Node 🌊

Many workloads are streams:

- uploads
- downloads
- large files
- large logs

If you load everything into memory:

- memory explodes
- GC thrashes
- latency spikes

Streaming is backpressure in action:

> if downstream cannot keep up, upstream must slow down.

```text
Readable -> Transform -> Writable
   |
   +-- if Writable is slow, Readable pauses/waits
```

---

## Extra 8: Timeouts and cancellation are the brakes 🛑

Without timeouts:

> one stuck dependency can hold resources forever.

Two habits:

1. timeouts for external dependencies (DB/HTTP/queues)
2. max wait time for internal queues (queues also need a limit)

Otherwise you get:

- pool saturation
- request pile-ups
- cascading failure

---

## Extra 9: More concurrency is not always better (find the bottleneck) 🪵

Think of the request chain as a pipe:

```text
Nginx limit -> app concurrency -> workers -> DB pool -> DB locks/IO
```

If you increase app concurrency from 100 to 1000, you may only:

- saturate DB pool
- increase lock contention
- explode p99 latency

Concurrency tuning is:

> find the bottleneck, then control the pressure upstream.

---

## What you should truly know in the AI era (the “god view”) 🧠

1. classify work: CPU-bound vs IO-bound
2. understand runtime: Node event loop + libuv; process/thread costs
3. enforce limits: pools, queues, timeouts, rate limiting are one family of tools
4. move heavy work away: workers / queues / offline jobs
5. observability: concurrency issues often require metrics/logs/traces, not guesswork

---

## Next questions 🚀

1. How do event loop phases affect behavior?
2. What is libuv thread pool size by default, and when does it become a bottleneck?
3. How does “concurrency control” connect end-to-end: Nginx limits -> app queue -> DB pool?
