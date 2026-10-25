# Roadmap

Six weeks, plus a bonus. Each week follows Learn → Build → Break → Fix → Document → Commit.
Tick the boxes as you go. A week is finished only when its **Definition of done** is fully ticked.

---

## Week 1 – TCP server and the RESP protocol

**Goal:** `redis-cli` can connect and run `PING`, `ECHO`, `SET` and `GET` against a single-threaded server.

### Concepts to learn
- [ ] TCP basics: `ServerSocket`, `Socket`, `accept()` blocking, ports, the connection lifecycle
- [ ] Byte streams vs character streams; `InputStream` buffering; why `readLine()` is a trap for binary data
- [ ] RESP2 spec: simple strings `+`, errors `-`, integers `:`, bulk strings `$`, arrays `*`, the `\r\n` framing
- [ ] Why a request may arrive across several `read()` calls (TCP is a stream, not messages)

### Build tasks
- [ ] Server listens on a configurable port (default 6379) and accepts one client at a time
- [ ] `RespParser`: reads one RESP array of bulk strings from an `InputStream`
- [ ] `RespWriter`: writes simple strings, errors, integers, bulk strings and the null bulk string
- [ ] Commands: `PING [msg]`, `ECHO msg`, `SET key value`, `GET key`
- [ ] Unknown command → `-ERR unknown command '<name>'`
- [ ] In-memory store backed by `HashMap<String, byte[]>`
- [ ] Unit tests for the parser and writer; an integration test that opens a real socket

### Break it
- [ ] Connect two `redis-cli` sessions at once and watch the second one hang. Explain why.
- [ ] Send a request one byte at a time (`nc` + a script). Does the parser cope with partial reads?
- [ ] Send malformed RESP (`$5\r\nhi\r\n`, a missing `\r\n`, a negative length). Crash, hang or error?
- [ ] Close the client mid-request. Does the server leak the socket or die?

### Definition of done
- [ ] `redis-cli PING`, `ECHO`, `SET` and `GET` all work
- [ ] Malformed input returns `-ERR` and doesn't kill the server
- [ ] `mvn -B verify` is green, with parser tests for partial and malformed input
- [ ] Concept notes: `tcp-sockets.md`, `resp-protocol.md`

---

## Week 2 – Concurrency

**Goal:** many clients at once, with correct results under contention.

### Concepts to learn
- [ ] Thread-per-client vs a fixed thread pool (`ExecutorService`); what happens when the pool is exhausted
- [ ] Java Memory Model basics: visibility, happens-before, `volatile`
- [ ] `synchronized` vs `ReentrantLock` vs `ReadWriteLock`
- [ ] `ConcurrentHashMap` internals: per-bin locking, `compute`/`merge` atomicity, why `get`-then-`put` isn't atomic
- [ ] Virtual threads (Project Loom): carrier threads, pinning, when they help and when they don't

### Build tasks
- [ ] Handle each client on a thread from a fixed pool
- [ ] Commands: `INCR`, `DECR`, `DEL`, `EXISTS`
- [ ] Implement `INCR` naively first (`get` → parse → `put`)
- [ ] Add a load-test harness: N clients × M `INCR`s against one key, then assert the final value is N × M
- [ ] Switch the store to `ConcurrentHashMap` and make `INCR` atomic with `compute`
- [ ] Add a config flag to run client handlers on virtual threads

### Break it
- [ ] Run 100 clients × 1,000 `INCR`s on the naive version and record the lost updates
- [ ] Wrap the whole store in `synchronized` and measure the throughput drop compared with `ConcurrentHashMap`
- [ ] Open 10,000 connections against the fixed pool and watch what happens to new clients
- [ ] Use `synchronized` around blocking I/O on virtual threads and observe pinning (`-Djdk.tracePinnedThreads=full`)

### Definition of done
- [ ] Load test passes: final `INCR` count is exact across 100 concurrent clients
- [ ] Throughput numbers recorded for `synchronized`, `ConcurrentHashMap`, platform threads and virtual threads
- [ ] Concept notes: `race-conditions.md`, `concurrenthashmap.md`, `virtual-threads.md`
- [ ] ADR: which threading model the server uses by default, and why

---

## Week 3 – Expiry and eviction

**Goal:** keys can expire, and memory stays bounded.

### Concepts to learn
- [ ] How Redis expires keys: lazy (on access) plus active (periodic sampling)
- [ ] `System.currentTimeMillis()` vs `System.nanoTime()`; why wall-clock time is unsafe for TTLs
- [ ] `ScheduledExecutorService`: fixed rate vs fixed delay; what happens if a task throws
- [ ] LRU: `LinkedHashMap` with access order plus `removeEldestEntry`; then a hand-built doubly linked list plus hash map

### Build tasks
- [ ] `SET key value EX seconds | PX millis`, `EXPIRE`, `TTL`, `PTTL`, `PERSIST`
- [ ] Lazy expiry: an expired key behaves as missing on read
- [ ] Active expiry: a background task samples keys and deletes expired ones
- [ ] `maxkeys` limit with LRU eviction, using `LinkedHashMap` first
- [ ] Replace it with a custom O(1) LRU (hash map plus doubly linked list) and reuse the same tests
- [ ] Make LRU thread-safe and document the locking strategy

### Break it
- [ ] Set 1M keys with a 1s TTL, never read them, and turn off active expiry. Watch the heap.
- [ ] Throw an exception inside the scheduled task and confirm it silently stops running. Then fix that.
- [ ] Hammer the custom LRU from many threads without locking and capture the corruption (lost nodes, cycles)
- [ ] Change the system clock during a TTL test (or inject a clock) to show the wall-clock bug

### Definition of done
- [ ] TTL commands behave like Redis for the implemented cases
- [ ] Memory stays bounded under the 1M-key test with active expiry on
- [ ] Custom LRU passes the same tests as the `LinkedHashMap` version, including concurrent tests
- [ ] Concept notes: `ttl-expiry.md`, `scheduled-executors.md`, `lru-cache.md`

---

## Week 4 – Non-blocking I/O (NIO)

**Goal:** a single-threaded `Selector` event loop, benchmarked against thread-per-client.

### Concepts to learn
- [ ] Blocking vs non-blocking I/O; the C10K problem
- [ ] `Selector`, `SelectionKey`, `ServerSocketChannel`, `SocketChannel`, interest ops
- [ ] `ByteBuffer`: `flip`, `compact`, position/limit/capacity; heap vs direct buffers
- [ ] Partial reads and partial writes on non-blocking channels; per-connection read and write buffers
- [ ] How Redis itself does it: a single-threaded event loop

### Build tasks
- [ ] NIO server with one event loop thread handling accept, read and write
- [ ] Make the RESP parser incremental (resumable across partial buffers)
- [ ] Per-connection state: an input buffer, an output queue, and `OP_WRITE` only when there's output pending
- [ ] Server mode switch: `--io=blocking` or `--io=nio`
- [ ] Benchmark both modes with `redis-benchmark` (vary clients: 10, 100, 1,000, 10,000)

### Break it
- [ ] Forget `flip()` once and observe the garbage reads
- [ ] Keep `OP_WRITE` always registered and watch CPU hit 100%
- [ ] Send a 10 MB value and confirm partial writes are handled (no truncated replies)
- [ ] Run a slow command inside the event loop and show that it stalls every client

### Definition of done
- [ ] `redis-cli` and `redis-benchmark` work against NIO mode
- [ ] Benchmark table (ops/sec, p99 latency, threads, memory) committed in a concept note
- [ ] Concept notes: `nio-selector.md`, `bytebuffer.md`, `blocking-vs-nonblocking.md`
- [ ] ADR: default I/O model, backed by the benchmark numbers

---

## Week 5 – Persistence and command modelling

**Goal:** data survives restarts and crashes; commands are modelled as types.

### Concepts to learn
- [ ] Append-only file (AOF): write-ahead logging; `fsync` policies (always, every second, never)
- [ ] Snapshots (RDB-style): point-in-time copy; consistency while writes continue
- [ ] Crash recovery: torn writes, truncated last record, checksums
- [ ] `FileChannel`, `force()`, atomic rename for safe file replacement
- [ ] Records and sealed interfaces; exhaustive `switch` with pattern matching

### Build tasks
- [ ] Refactor commands into `sealed interface Command permits Get, Set, Incr, ...` with records
- [ ] Dispatch with an exhaustive pattern-matching `switch` (no `default` branch)
- [ ] AOF: append every write command in RESP format; replay it on startup
- [ ] Configurable `fsync` policy
- [ ] `SAVE` / `BGSAVE`: write a snapshot to a temp file, then rename it into place atomically
- [ ] Startup: load the snapshot, then replay the AOF tail
- [ ] AOF rewrite (compaction) to stop the file growing forever

### Break it
- [ ] `kill -9` the server during heavy writes with each `fsync` policy, and count the lost writes
- [ ] Truncate the AOF mid-record and make startup recover (skip or repair) instead of crashing
- [ ] Corrupt one byte in a snapshot and detect it with a checksum
- [ ] Add a new command record without handling it and confirm the compiler catches it

### Definition of done
- [ ] Data survives a clean restart and a `kill -9` (within the chosen `fsync` guarantee)
- [ ] A corrupted or truncated file produces a clear error or recovery, never silent data loss
- [ ] Lost-write counts per `fsync` policy recorded
- [ ] Concept notes: `append-only-file.md`, `snapshots.md`, `crash-recovery.md`, `sealed-interfaces-records.md`
- [ ] ADR: persistence format and the default `fsync` policy

---

## Week 6 – JVM internals and performance

**Goal:** know where time and memory go, and prove it with data.

### Concepts to learn
- [ ] Heap layout: young and old generations; object headers; what a `byte[]` and a `String` really cost
- [ ] GC algorithms: G1 vs ZGC vs Parallel; pause time vs throughput trade-offs
- [ ] JFR (Java Flight Recorder) and JDK Mission Control; VisualVM
- [ ] Allocation profiling; escape analysis; JIT warm-up and why the first benchmark run lies
- [ ] Load-testing method: warm-up, steady state, percentiles (not averages)

### Build tasks
- [ ] Load-test script using `redis-benchmark` with fixed scenarios (GET-heavy, SET-heavy, mixed)
- [ ] Record a JFR profile under load and identify the top 3 CPU and allocation hotspots
- [ ] Fix at least one hotspot (e.g. buffer reuse, avoiding `String` conversions) and measure it again
- [ ] Run the same load under G1, ZGC and Parallel, and compare pauses and throughput
- [ ] Add an `INFO`-style command: memory used, keys, connected clients, ops/sec

### Break it
- [ ] Run with `-Xmx64m` and load until `OutOfMemoryError`; analyse the heap dump
- [ ] Introduce a deliberate memory leak (e.g. a static list of clients) and find it with VisualVM
- [ ] Benchmark without warm-up vs with warm-up and explain the difference

### Definition of done
- [ ] Before and after numbers for at least one optimisation, with JFR evidence
- [ ] GC comparison table committed
- [ ] Heap dump analysis of the OOM written up
- [ ] Concept notes: `jvm-memory.md`, `garbage-collection.md`, `profiling-jfr.md`, `load-testing.md`

---

## Bonus – Mini dependency-injection container

**Goal:** understand what Spring does at startup by building a tiny version of it.

### Concepts to learn
- [ ] Reflection: `Class`, `Constructor`, `Field`, `Method`; `setAccessible` and module restrictions
- [ ] Annotations: `@Retention`, `@Target`; reading them at runtime
- [ ] Dependency graphs: topological order; detecting cycles
- [ ] Singleton vs prototype scope

### Build tasks
- [ ] Custom annotations: `@Component`, `@Inject`, `@Config("key")`
- [ ] Scan a package for `@Component` classes
- [ ] Constructor injection, resolving dependencies recursively
- [ ] Wire the server (store, I/O loop, persistence, config) through the container instead of `new`

### Break it
- [ ] Create a circular dependency (A → B → A) and produce a clear error with the cycle path
- [ ] Two implementations of one interface: what happens? Add a qualifier.
- [ ] Measure the startup cost of reflection vs manual wiring

### Definition of done
- [ ] Server starts entirely through the container
- [ ] Cycle detection has a test
- [ ] Concept notes: `reflection.md`, `annotations.md`, `dependency-injection.md`
- [ ] ADR: keep the container or go back to manual wiring
