# Node.js runtime architecture and web server performance

This document summarizes how Node.js is put together at a high level, what "non-blocking event loop" means in
practice, and how Node.js web servers tend to compare (performance-wise) with servers built on Java, Go, Rust, and
Python.

Performance comparisons are qualitative by design. Real-world results depend heavily on framework choice, HTTP
features enabled (TLS, HTTP/2), database and network latency, response sizes, logging, and deployment topology.

## Core idea

Node.js combines:

* **V8** to execute JavaScript.
* **libuv** to provide the event loop and cross-platform asynchronous I/O primitives.

The common goal is to keep the JavaScript thread available to run user code while I/O is handled asynchronously and
completions are delivered back to JavaScript as callbacks or resolved Promises.

## High-level startup map (where things live)

The top-level flow for the `node` executable is:

1. `main()` in `src/node_main.cc` calls `node::Start()`.
2. `node::Start()` in `src/node.cc` parses options, loads (or builds) the startup snapshot, and creates a
   `NodeMainInstance`.
3. `NodeMainInstance::Run()` in `src/node_main_instance.cc` creates:
   * a V8 `Isolate` (the JS engine instance),
   * a Node.js `Environment` (per-instance runtime state), and
   * a V8 `Context` (global object + built-ins).
4. `LoadEnvironment()` in `src/api/environment.cc` runs the internal bootstrapping JavaScript (for example,
   `lib/internal/bootstrap/node.js`) and then hands off to the user entry point (CommonJS or ESM loaders in
   `lib/internal/modules/`).
5. `SpinEventLoopInternal()` in `src/api/embed_helpers.cc` drives libuv (`uv_run`) and drains pending V8 tasks until
   the process is ready to exit.

Native "internal bindings" used by Node.js core JavaScript are exposed through `internalBinding()` implemented in
`src/node_binding.cc`.

## What "non-blocking event loop" means

"Non-blocking" means the JavaScript thread does not sit idle waiting for I/O to complete.

Instead:

* The JavaScript thread starts an operation (for example, a socket read, DNS lookup, or file read).
* libuv registers interest in the completion (via OS facilities like epoll/kqueue/IOCP) and/or runs the work on a
  helper threadpool for operations that do not have a portable non-blocking OS API.
* When the operation completes, libuv queues a callback for execution on the event loop thread.
* The event loop runs queued callbacks and Promise continuations on subsequent turns ("ticks").

This design enables high concurrency for I/O-heavy workloads, but it does not automatically make CPU-heavy JavaScript
run in parallel. CPU-heavy work on the main thread blocks progress for all requests handled by that process until the
work completes.

## How this affects web server performance

### What Node.js is typically good at

Node.js often performs well for services that are dominated by waiting:

* API gateways and backends-for-frontends (BFFs) that orchestrate multiple downstream calls.
* Real-time services (WebSockets, Server-Sent Events) with many concurrent connections.
* Streaming and proxying workloads where backpressure handling matters.

In these scenarios, throughput and latency are often dominated by network and database time rather than raw language
speed.

### Common performance hazards in Node.js servers

The biggest risk to throughput and tail latency is blocking the event loop thread, for example:

* Using synchronous APIs in request handlers (`fs.*Sync`, `child_process.execSync`, synchronous crypto, etc.).
* CPU-heavy work in JavaScript (large JSON parsing/serialization, heavy regex usage, image processing, compression,
  complex templating).
* Overloading libuv's threadpool with filesystem or crypto work, which can indirectly delay other async operations
  that also use the pool.

### Scaling across CPU cores

Node.js runs one event loop per thread. A single Node.js process typically uses one core for JavaScript execution.

Common approaches to scale are:

* Run multiple Node.js processes (for example, with clustering or a process manager) and load-balance across them.
* Offload CPU-heavy work to `worker_threads` or external services.

## Performance comparisons with other common server stacks

No single runtime wins at everything. The main differences are usually in CPU efficiency, latency under load, memory
footprint, and how easy it is to use multiple cores in one deployment unit.

The table below is a simplified summary for typical web APIs (not a benchmark result):

| Dimension | Node.js | Java (JVM) | Go | Rust | Python (CPython) |
| --- | --- | --- | --- | --- | --- |
| I/O concurrency model | Event loop | Threads or async I/O | Goroutines | Async runtime | Async + multiprocess |
| CPU-heavy request handling | Requires offload | Strong | Strong | Strong | Often limited |
| Tail-latency sensitivity to "bad" handlers | High | Medium | Medium | Low–Medium | High |
| Multi-core scaling per process | Manual (cluster/workers) | Built-in threads | Built-in | Built-in | Multiprocess |
| Startup and warmup | Fast | Slower (JIT warmup) | Fast | Fast | Medium |
| Typical memory per process | Medium | Medium–High | Low–Medium | Low | Low–Medium |

Key points to interpret the table:

* **Java** can be extremely fast with NIO/Netty-style servers and careful tuning. Some frameworks add overhead but can
  still be high throughput. The JVM may need warmup to reach steady-state performance.
* **Go** often delivers strong throughput with straightforward concurrency and good multi-core utilization. Garbage
  collection exists but is typically predictable for many web workloads.
* **Rust** can achieve very high performance and low memory use, especially for CPU efficiency and tail latency, at
  the cost of more complexity and compile-time overhead.
* **Python** is frequently limited by interpreter overhead and the GIL for CPU-bound work. High-performance Python
  servers typically rely on async I/O plus multiple processes and/or native extensions.

Across all stacks, the framework and server implementation often matters as much as the language. For example, a
minimal, async-first server framework can be dramatically faster than a feature-rich framework with heavy middleware,
regardless of runtime.

## Practical takeaways

Node.js tends to be a strong choice for web applications when:

* Work is primarily I/O-bound and benefits from high concurrency.
* Real-time connections and streaming responses are important.
* Sharing code and tooling across frontend and backend (JavaScript/TypeScript) is valuable.

Another runtime is often a better fit when:

* A large portion of request time is CPU-bound and must use all cores efficiently inside one process.
* Strict tail-latency SLOs must be met under heavy load without risk from occasional blocking handlers.
* A static, single-binary deployment model is a primary constraint (commonly satisfied by Go or Rust).

## References

* Node.js docs:
  * `doc/api/child_process.md` (sync vs async process APIs)
  * `doc/api/dns.md` (threadpool-backed operations)
  * `doc/api/worker_threads.md` (CPU offload within Node.js)
* Source map:
  * `src/node_main.cc`
  * `src/node.cc`
  * `src/node_main_instance.cc`
  * `src/api/environment.cc`
  * `src/api/embed_helpers.cc`
  * `src/node_binding.cc`
* External:
  * [libuv documentation][]
  * [TechEmpower Web Framework Benchmarks][]

[libuv documentation]: https://docs.libuv.org/
[TechEmpower Web Framework Benchmarks]: https://www.techempower.com/benchmarks/
