# Logger

A logger is the in-process library an application uses to record what's happening at runtime. Code calls `logger.info("user signed in")` from anywhere in the app, and the library timestamps the message, attaches the severity level, and writes it to one or more places — the console, a file, or both. Think Log4j, SLF4J, or Python's logging module. We're designing the library that lives inside one application, not a distributed log aggregation service.

---

## Requirements

You walk into the interview and get greeted with this prompt:

> "Design a logging service. Or call it a logger, whichever you prefer."

Before touching a whiteboard, spend 3–5 minutes narrowing that down.

### Clarifying Questions

**You:** "When you say 'logging service,' do you mean an in-process library the application links against? Or something that ships logs over the network to a central aggregator?"
**Interviewer:** "In-process library. Network shipping, ingestion pipelines, and central aggregation are someone else's problem."

That answer cuts most of the design space. No queues, no schema registries, no fan-out across services. The deliverable is an object model that runs inside one application's process and writes to local destinations like stdout and files.

**You:** "What severity levels should we support, and is there an ordering between them?"
**Interviewer:** "DEBUG, INFO, WARN, ERROR, FATAL. Ordered from least to most severe in that order."

Five levels with a natural ordering. That's a finite set with no per-level behavior, which is a textbook enum. If you reach for a Level class hierarchy with `DebugLevel`, `InfoLevel`, and so on, you're doing too much.

**You:** "Can a single logger write to multiple destinations at the same time? Like sending the same record to both the console and a file?"
**Interviewer:** "Yes. That's the common case."

Now you know "destination" is a first-class concept and that one log call hits all of them. That implies the library holds a list of destinations and iterates over them on every call.

**You:** "Does each destination decide its own filter level, or is there one global level on the logger?"
**Interviewer:** "Per destination. The console might want everything from DEBUG up, and the file destination might only care about WARN and above."

This rules out putting the level filter on the logger and pushes it down to each destination. The same record gets evaluated independently by each destination, which is fine because the record itself is immutable once created.

**You:** "And the format the records are written in. Is that fixed, or does it vary?"
**Interviewer:** "Varies. Sometimes plain text, sometimes JSON. And the format is independent of the destination type. You should be able to write JSON to the console, plain text to a file, or any combination."

This is the requirement that shapes the class model. If format and destination were coupled, you'd end up with a class for every (format, target) pair — `JsonFileDestination`, `PlainConsoleDestination`, and so on. Add a third format and a third target and you have nine classes. Since they vary independently, the right move is to compose them.

When a requirement gives you two dimensions that vary independently, that's almost always a signal to use composition over inheritance. Two interfaces composed together let you mix any combination without writing N×M classes.

**You:** "What about concurrency? Multiple threads in the same app are going to be calling log() simultaneously."
**Interviewer:** "Thread-safe. Each record's bytes have to land on a destination atomically — one record's bytes can't be split across or mixed with another's."

Concurrency is in scope, so locking is part of the design, not a cleanup pass at the end. Strict global submission order across threads is not required — only per-record atomicity on a single destination.

**You:** "Is configuration static, set at startup, or do we need to handle hot-reloading destinations and levels at runtime?"
**Interviewer:** "Static. Configured once at startup. The design should not block adding a remote destination later."

The last clause shapes the design. You don't build remote destinations now, but destinations must be pluggable behind an interface so adding one later doesn't force a rewrite.

### Final Requirements

| # | Requirement |
|---|-------------|
| 1 | Five severity levels with natural ordering: DEBUG < INFO < WARN < ERROR < FATAL |
| 2 | Each record carries: timestamp, level, message, emitting thread name |
| 3 | Logger writes each record to one or more destinations, configured at startup |
| 4 | Each destination has its own minimum level threshold — records below it are dropped |
| 5 | Format and destination type vary independently — any formatter can pair with any destination |
| 6 | Concurrent calls are safe: a record's bytes never interleave with another's on the same destination |

**Out of scope:** hot-reloading config, async/buffered writes, remote/network destinations in v1, hierarchical named loggers.

---

## Core Entities and Relationships

The central design challenge here is the axes-of-variation problem. A destination has two independently variable dimensions: **where** bytes go (console, file) and **how** a record is serialized (plain text, JSON). Coupling them into subclasses produces an N×M explosion. Separating them into two composable interfaces keeps the class count flat and each class focused.

**Logger** — The entry point. Holds a list of Destinations. On each log call, creates an immutable `LogRecord` and fans it out to every destination. No filtering logic here — that belongs to each destination.

**LogRecord** — An immutable value object. Created once per log call, passed to all destinations. Carries timestamp, level, message, and thread name. Immutability is load-bearing: because the record can't change after creation, formatters can read it concurrently without synchronization.

**LogLevel** — An enum: `DEBUG`, `INFO`, `WARN`, `ERROR`, `FATAL` with a numeric ordinal for comparison. Not a class hierarchy — there is no per-level polymorphic behavior, so a hierarchy would be pure ceremony.

**Destination** — A concrete class that owns the orchestration logic: check level, format the record, acquire the lock, write to the sink, release. It composes a `Formatter` and a `Sink`. The lock lives here because Destination is the unit that must serialize writes — not the Sink, which has no locking knowledge.

**Formatter** — An interface with one method: `format(record) -> String`. Stateless. Implementations — `PlainTextFormatter`, `JsonFormatter` — contain no mutable state, so they are inherently thread-safe.

**Sink** — An interface with one method: `write(formatted: String)`. The raw write target. Implementations — `ConsoleSink`, `FileSink` — know nothing about levels, formatting, or locks. They just write bytes.

### Entity Responsibilities

| Entity | Responsibility |
|--------|----------------|
| `Logger` | Entry point. Creates `LogRecord` per call, iterates destinations, delegates. No filter logic. |
| `LogRecord` | Immutable data carrier: timestamp, level, message, thread name. |
| `LogLevel` | Enum with ordinal for threshold comparison. |
| `Destination` | Orchestrates one write: filter → format → lock → sink.write → unlock. Owns the lock. |
| `Formatter` | Stateless strategy for serializing a `LogRecord` to a string. |
| `Sink` | Stateless write target. Knows only how to accept a formatted string. |

### Relationships

```
Logger
  └── destinations: List<Destination>
        └── Destination
              ├── minLevel:  LogLevel
              ├── formatter: Formatter    ← strategy: how to serialize
              ├── sink:      Sink         ← strategy: where to write
              └── lock:      ReentrantLock

Formatter (interface)
  ├── PlainTextFormatter
  └── JsonFormatter

Sink (interface)
  ├── ConsoleSink
  └── FileSink
```

The lock is per-Destination. Two threads writing to different destinations proceed in parallel — they have no reason to serialize. Only concurrent writes to the **same** destination need to be atomic. This maximizes throughput without over-synchronizing.

---

## Class Design

We design top-down: `Logger` first, then `Destination` (the most nuanced), then `Formatter`, `Sink`, `LogRecord`, and `LogLevel`.

### Logger

Logger is what application code calls. It creates records and fans them out. It has no filter logic of its own.

**Deriving State**

| Requirement | What Logger must track |
|-------------|------------------------|
| "Writes to one or more destinations" | List of Destination objects |

**Deriving Behavior**

| Need from requirements | Method on Logger |
|------------------------|-----------------|
| "Write a record at a given level" | `log(level, message) -> void` |
| "Convenience per-level calls" | `debug(msg)`, `info(msg)`, `warn(msg)`, `error(msg)`, `fatal(msg)` |

```
class Logger:
    - destinations: List<Destination>

    + Logger(destinations: List<Destination>)
    + log(level: LogLevel, message: String) -> void
    + debug(message: String) -> void
    + info(message: String) -> void
    + warn(message: String) -> void
    + error(message: String) -> void
    + fatal(message: String) -> void
```

The convenience methods are thin wrappers over `log()`. They exist so call sites read `logger.warn("disk full")` instead of `logger.log(LogLevel.WARN, "disk full")`.

---

### Destination

Destination is the most critical class. It owns the three-step algorithm — filter, format, write — and the lock that makes concurrent writes safe.

**Deriving State**

| Requirement | What Destination must track |
|-------------|------------------------------|
| "Each destination has its own minimum level" | `minLevel: LogLevel` |
| "Format varies independently from destination type" | `formatter: Formatter` |
| "Where bytes go varies independently from format" | `sink: Sink` |
| "A record's bytes never interleave with another's" | `lock: ReentrantLock` |

**Deriving Behavior**

| Need | Method on Destination |
|------|----------------------|
| "Receive a record, filter, format, write" | `write(record: LogRecord) -> void` |

```
class Destination:
    - minLevel:  LogLevel
    - formatter: Formatter
    - sink:      Sink
    - lock:      ReentrantLock

    + Destination(minLevel, formatter, sink)
    + write(record: LogRecord) -> void
```

---

### Formatter (interface)

Formatter is the strategy for serialization. It is stateless — reading fields from an immutable `LogRecord` and returning a string. No lock needed; stateless objects are inherently thread-safe.

**Deriving Behavior**

| Need | Method on Formatter |
|------|---------------------|
| "Format varies: plain text, JSON, others" | `format(record: LogRecord) -> String` |

```
interface Formatter:
    + format(record: LogRecord) -> String

class PlainTextFormatter implements Formatter:
    + format(record) -> String

class JsonFormatter implements Formatter:
    + format(record) -> String
```

---

### Sink (interface)

Sink is the strategy for the write target. It receives an already-formatted string and writes it. No knowledge of levels, formatting, or locks.

```
interface Sink:
    + write(formatted: String) -> void

class ConsoleSink implements Sink:
    + write(formatted) -> void    // System.out.println(formatted)

class FileSink implements Sink:
    - writer: BufferedWriter

    + FileSink(filePath: String)
    + write(formatted) -> void    // writer.write(formatted + newline)
```

---

### LogRecord (immutable value object)

**Deriving State**

| Requirement | What LogRecord must carry |
|-------------|--------------------------|
| "Record carries timestamp" | `timestamp: Instant` |
| "Record carries level" | `level: LogLevel` |
| "Record carries message" | `message: String` |
| "Record carries emitting thread name" | `threadName: String` |

```
class LogRecord:
    - timestamp:  Instant     // set at construction from system clock
    - level:      LogLevel
    - message:    String
    - threadName: String      // set at construction from current thread

    + LogRecord(level, message)    // timestamp and threadName captured internally
    + getTimestamp() -> Instant
    + getLevel() -> LogLevel
    + getMessage() -> String
    + getThreadName() -> String
```

All fields are set at construction and never mutated. This is load-bearing: because `LogRecord` is immutable, formatters can read it from multiple threads without synchronization.

---

### LogLevel (enum)

```
enum LogLevel:
    DEBUG(0)
    INFO(1)
    WARN(2)
    ERROR(3)
    FATAL(4)

    - ordinal: int

    + getOrdinal() -> int
```

**Why not a class hierarchy?**

Five levels, one linear ordering, no per-level polymorphic behavior. There is nothing a `WarnLevel` class does differently from a `ErrorLevel` class — they would carry identical logic and differ only by value. An enum is the right tool: it models a fixed set of named values with an ordering. A class hierarchy here signals a misunderstanding of when to apply OOP.

---

### Final Class Design

```
+-------------------------------+
|            Logger             |
+-------------------------------+
| - destinations: List          |
+-------------------------------+
| + log(level, message)         |
| + debug(message)              |
| + info(message)               |
| + warn(message)               |
| + error(message)              |
| + fatal(message)              |
+-------------------------------+
             |
             | fans out to each
             v
+-------------------------------+
|          Destination          |
+-------------------------------+
| - minLevel:  LogLevel         |
| - formatter: Formatter   -----+-----> <<interface>> Formatter
| - sink:      Sink        -----+-----> <<interface>> Sink
| - lock:      ReentrantLock    |
+-------------------------------+              ^            ^
| + write(record: LogRecord)    |              |            |
+-------------------------------+   PlainText  Json   (future:
                                    Formatter  Formatter  XmlFormatter)

<<interface>> Formatter                <<interface>> Sink
+---------------------------+          +---------------------------+
| + format(record)          |          | + write(formatted: String)|
|     -> String             |          +---------------------------+
+---------------------------+               ^             ^
                                            |             |
                                      ConsoleSink     FileSink
                                                      - writer: BufferedWriter

+---------------------------+
|         LogRecord         |   (immutable)
+---------------------------+
| - timestamp:  Instant     |
| - level:      LogLevel    |
| - message:    String      |
| - threadName: String      |
+---------------------------+

LogLevel (enum):  DEBUG(0) < INFO(1) < WARN(2) < ERROR(3) < FATAL(4)
```

---

## Implementation

The two most interesting methods are `Destination.write()` (the critical section with filter, format, and lock) and `Logger.log()` (the fan-out). We'll also walk through both formatter implementations.

### Logger.log()

**Core logic:**
1. Capture timestamp and thread name — do this once, before the fan-out
2. Create an immutable `LogRecord`
3. Call `write()` on every destination

**Edge cases:**
- No destinations configured — loop is a no-op, no error
- `log()` must never throw an exception back to the caller even if a sink fails — log calls can't crash the application

```
log(level, message)
    record = LogRecord(level, message)    // captures Instant.now() and Thread.currentThread().name internally

    for destination in destinations
        try
            destination.write(record)
        catch Exception e
            // swallow silently or write to stderr — never propagate to caller

debug(message)  ->  log(LogLevel.DEBUG, message)
info(message)   ->  log(LogLevel.INFO, message)
warn(message)   ->  log(LogLevel.WARN, message)
error(message)  ->  log(LogLevel.ERROR, message)
fatal(message)  ->  log(LogLevel.FATAL, message)
```

---

### Destination.write() — filter, format, lock

This method has three phases. The order matters.

**Core logic:**
1. Check level — if the record is below threshold, return immediately with no further work
2. Format the record — outside the lock, because formatting is stateless and can run in parallel
3. Acquire lock, write to sink, release

**Why format outside the lock?**

`LogRecord` is immutable and `Formatter` is stateless — reading fields and producing a string has no side effects. Keeping formatting outside the lock means two threads can format their records in parallel; only the actual write to the shared sink serializes. If formatting happened inside the lock, thread-2 would be blocked on formatting even while thread-1 is still formatting too — wasted serialization.

**Edge cases:**
- Record level below minLevel — short-circuit before any formatting work
- Sink throws on write — catch inside the lock, release the lock in a finally block, swallow the exception

```
write(record)
    if record.getLevel().getOrdinal() < minLevel.getOrdinal()
        return                              // filtered — no further work

    formatted = formatter.format(record)    // outside lock: immutable input, stateless formatter

    lock.acquire()
    try
        sink.write(formatted)
    catch Exception e
        // swallow — a failing sink must not crash the caller
    finally
        lock.release()                      // always release, even on exception
```

---

### PlainTextFormatter.format()

```
format(record)
    return "[" + record.getTimestamp()  + "]"
         + " [" + record.getLevel()    + "]"
         + " [" + record.getThreadName() + "]"
         + " " + record.getMessage()

// Output: [2024-01-15T10:00:00.123Z] [WARN] [main] disk space low
```

---

### JsonFormatter.format()

```
format(record)
    return "{"
         + "\"timestamp\":\"" + record.getTimestamp()   + "\","
         + "\"level\":\""     + record.getLevel()       + "\","
         + "\"thread\":\""    + record.getThreadName()  + "\","
         + "\"message\":\""   + escapeJson(record.getMessage()) + "\""
         + "}"

// Output: {"timestamp":"2024-01-15T10:00:00.123Z","level":"WARN","thread":"main","message":"disk space low"}
```

`escapeJson` handles backslashes and double quotes inside the message. A formatter is the right place for this — the Logger and Destination have no business knowing the encoding rules of a specific format.

---

## Design Patterns

### Strategy Pattern (twice)
`Formatter` and `Sink` are both strategy interfaces. `Destination` holds one of each and delegates to them without knowing their concrete type. Adding a new format (XML) means implementing `Formatter` and wiring it in — `Destination`, `Logger`, and all `Sink` implementations are untouched. Adding a new sink (network socket, syslog) means implementing `Sink` — same story.

This is the pattern that eliminates the N×M class explosion. With two strategies composing freely, you get N+M classes instead of N×M.

### Composition Over Inheritance
`Destination` is concrete and composed — it holds a `Formatter` and a `Sink` rather than subclassing for each combination. There is no `JsonFileDestination` or `PlainConsoleDestination`. Any formatter pairs with any sink, and a new destination for a new target (say, a network socket) is a ten-line `Sink` implementation, not a new subclass tree.

### Information Expert
- `LogRecord` knows its own fields — formatters ask it for data, it doesn't push data at them
- `Destination` owns the level threshold — it decides whether to process a record
- `Destination` owns the lock — it decides when to serialize, so the Sink doesn't need to know about threading
- `Logger` owns the destination list — it decides the fan-out

---

## Verification

**Setup:** Logger with two destinations:
- `D1`: ConsoleDestination — minLevel=DEBUG, PlainTextFormatter
- `D2`: FileDestination — minLevel=WARN, JsonFormatter

**`logger.warn("disk space low")` on thread "main" at t=T:**
```
log(WARN, "disk space low")
  record = LogRecord(T, WARN, "disk space low", "main")

  D1.write(record):
    WARN(2) >= DEBUG(0)  ✓
    formatted = "[T] [WARN] [main] disk space low"
    lock.acquire()
    ConsoleSink.write(formatted)  → stdout
    lock.release()

  D2.write(record):
    WARN(2) >= WARN(2)  ✓
    formatted = '{"timestamp":"T","level":"WARN","thread":"main","message":"disk space low"}'
    lock.acquire()
    FileSink.write(formatted)  → app.log
    lock.release()
```

**`logger.debug("fetching user")` on thread "worker-1":**
```
log(DEBUG, "fetching user")
  record = LogRecord(T2, DEBUG, "fetching user", "worker-1")

  D1.write(record):
    DEBUG(0) >= DEBUG(0)  ✓
    → written to console as plain text

  D2.write(record):
    DEBUG(0) >= WARN(2)?  ✗
    → return immediately, nothing written to file
```

**Thread-1 calls `logger.error("out of memory")`, Thread-2 calls `logger.info("user logged in")` concurrently:**
```
Both create their own LogRecord with their own timestamp and thread name.
Both fan out to D1 and D2.

At D2 (FileDestination, minLevel=WARN):
  Thread-1's ERROR(3) >= WARN(2)  ✓  → proceeds to format and write
  Thread-2's INFO(1)  >= WARN(2)  ✗  → filtered immediately, never reaches the lock

At D1 (ConsoleDestination, minLevel=DEBUG):
  Both pass the filter.
  Both format their records in parallel (outside the lock — safe, immutable record).
  Thread-1 acquires D1's lock first → writes its record → releases.
  Thread-2 then acquires D1's lock → writes its record → releases.
  Records never interleave on stdout.  ✓
```

---

## Extensibility

### 1. "How would you make log() non-blocking — callers should never wait on I/O?"

Right now every `log()` call blocks until all destinations have finished writing. In a high-throughput service, a slow file write on the hot path adds latency to every request.

> "I'd introduce an async write path. Each `Destination` gets a `BlockingQueue<String>` (holding pre-formatted strings) and a single background writer thread that drains it and calls `sink.write()`. The `Destination.write()` method becomes: check level, format, enqueue — all non-blocking. The background thread owns the lock and the actual I/O. The calling thread is never blocked on disk. The tradeoff is durability: records sitting in the queue are lost if the process crashes before the thread drains them. You mitigate this with a bounded queue and a flush-on-shutdown hook. The `Formatter`, `Sink`, `Logger`, and `LogRecord` classes are all untouched — the async boundary is entirely inside `Destination`."

```
// Async Destination (sketch)
write(record)
    if record.level.ordinal < minLevel.ordinal
        return
    formatted = formatter.format(record)    // still outside lock
    queue.offer(formatted)                  // non-blocking enqueue; caller returns immediately

// Background thread (started at construction)
loop:
    formatted = queue.take()                // blocks until an item is available
    lock.acquire()
    try
        sink.write(formatted)
    finally
        lock.release()
```

---

### 2. "How would you support hierarchical loggers — com.app.service inheriting thresholds from com.app?"

Right now there is one Logger. Hierarchical logging lets different parts of the codebase have named loggers that inherit behavior from a parent chain.

> "I'd add a `name` and `parent: Logger?` field to Logger. When `log()` is called, it fans out to its own destinations, then — if propagation is enabled — calls `parent.log()` recursively, walking up to the root logger. A logger with no destinations of its own inherits everything from its parent. A logger that sets its own destinations can either add to the parent's outputs or, by disabling propagation, fully override them. The `LogRecord`, `Destination`, `Formatter`, and `Sink` classes are unchanged. The only new logic is in `Logger.log()`: after iterating own destinations, check `if propagate and parent != null → parent.log(record)`."

```
class Logger:
    - name:         String
    - destinations: List<Destination>
    - parent:       Logger?
    - propagate:    boolean    // default true

log(level, message)
    record = LogRecord(level, message)
    applyDestinations(record)

applyDestinations(record)
    for destination in destinations
        destination.write(record)
    if propagate and parent != null
        parent.applyDestinations(record)    // walk up the hierarchy
```

---

### 3. "The requirement says configured once at startup. Would you use a Builder here?"

> "Worth mentioning, yes — not implementing. Logger construction involves assembling a list of destinations, each of which composes a formatter and a sink. That's several objects wired together, and a constructor call doesn't read well with two or three destinations. A `LoggerBuilder` makes the startup configuration declarative and hard to misconfigure: each `addDestination` call is explicit, and `build()` validates that at least one destination exists before handing back the Logger. I wouldn't implement it now — the current design works and the interviewer hasn't asked for it — but I'd call it out as the natural API for the startup wiring requirement."

```
Logger logger = LoggerBuilder.create()
    .addDestination(new Destination(LogLevel.DEBUG, new PlainTextFormatter(), new ConsoleSink()))
    .addDestination(new Destination(LogLevel.WARN,  new JsonFormatter(),      new FileSink("app.log")))
    .build();
```
