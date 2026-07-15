# CDN Push Mechanism

A push-based CDN system that propagates newly uploaded S3 objects to distributed edge nodes in near real time. When a file lands in S3, an event triggers the CDN controller, which normalizes the event, deduplicates it, selects target edge nodes via a pluggable routing strategy, and dispatches isolated per-node delivery tasks — each tracking its own retry state — so that one node's failure does not block or re-trigger delivery to others.

---

## Requirements

You walk into the interview and get greeted with this prompt:

> "Design a push-based CDN mechanism to propagate newly uploaded S3 content to distributed edge nodes in near real time."

### Clarifying Questions

**You:** "When a file is uploaded to S3, how does our system learn about it — an S3 event notification, a webhook, a polling loop?"
**Interviewer:** "An event notification. Treat the event as a callback into the system with the S3 object's metadata."

**You:** "Should every file go to every edge node, or is routing logic required — say, region-based or content-type-based?"
**Interviewer:** "Start with all active nodes, but the routing should be pluggable."

**You:** "Is the push to edge nodes synchronous — do we wait for all nodes before returning — or asynchronous?"
**Interviewer:** "Asynchronous. Return immediately and track delivery status separately."

**You:** "How should we handle an edge node that fails to receive the content? Retry? Give up?"
**Interviewer:** "Retry per node up to a configurable max attempts, with exponential backoff. After max retries, mark that node's delivery as failed."

**You:** "S3 guarantees at-least-once event delivery — can the same upload trigger two events?"
**Interviewer:** "Yes. Deduplication is required."

**You:** "If a node was offline and missed content, can we replay the push to it later?"
**Interviewer:** "Yes — support an on-demand replay to a specific edge node."

**You:** "Does the edge node need to verify the content it received — checksums, ETags?"
**Interviewer:** "Include the ETag in the push payload so the edge node can validate."

**You:** "Persistence, actual S3 SDK calls, real HTTP to edge nodes?"
**Interviewer:** "All out of scope. Focus on the object model and core logic."

### Final Requirements

| # | Requirement |
|---|-------------|
| 1 | The system receives S3 upload events containing object metadata (bucket, key, size, contentType, ETag, timestamp) |
| 2 | Duplicate events for the same upload (same bucket + key + ETag) are detected and silently skipped |
| 3 | A pluggable routing strategy determines which edge nodes receive each piece of content |
| 4 | Each selected edge node gets an independent `NodeDeliveryTask` with its own status and retry count |
| 5 | The push payload includes bucket, key, ETag, and source URL so the edge node can fetch and verify |
| 6 | Failed deliveries are retried per node, up to a configurable max, with exponential backoff |
| 7 | A `PropagationJob` aggregates all per-node tasks and derives an overall status (COMPLETED, PARTIALLY_FAILED, FAILED) |
| 8 | On-demand replay to a specific node delivers any content that node missed |

**Out of scope:** actual S3 SDK or HTTP client implementation, persistence layer, authentication, content transformation, pull-based CDN fallback, bandwidth throttling, distributed coordination, UI.

---

## Core Entities and Relationships

The central challenge is **two-layered** — event deduplication at the intake boundary, and per-node delivery isolation with independent retry state at the dispatch boundary.

A naive approach pushes to all edge nodes as a batch. One node's failure means you either lose that delivery or re-push to everyone. The fix is to model each node's delivery as a separate `NodeDeliveryTask` with its own `TaskStatus` and `attemptCount`. US-EAST succeeding does not affect AP-SOUTH retrying. Failures are targeted; each node's delivery is independently resumable from exactly where it failed.

S3 provides at-least-once event delivery. The same upload can trigger two identical events seconds apart. Without deduplication, every node would be pushed to twice. A `processedEvents` set keyed on `(bucket + key + ETag)` prevents this: the same ETag means the same bytes; if we have already dispatched it, we skip the duplicate event entirely.

The routing layer sits between these two concerns. A **ContentRoutingStrategy** interface decides which active edge nodes receive a given piece of content. `CDNController` injects the strategy at construction, so routing logic is swappable without touching any other class.

**CDNController** — The orchestrator and S3 event entry point. Deduplicates events, normalizes metadata, selects target nodes, creates `PropagationJob` instances, and dispatches `NodeDeliveryTask` objects. The only class a caller interacts with.

**S3Event** — The raw inbound event. Carries bucket, key, size, contentType, ETag, and a timestamp. This is the boundary object the external integration provides.

**ContentMetadata** — The normalized descriptor of the S3 object, derived from `S3Event` and passed to edge nodes. Decoupling from the raw event means the rest of the system is not tied to S3's event schema.

**EdgeNode** — A CDN edge location: region, endpoint URL, and availability status. Only `ACTIVE` nodes are candidates for delivery. Status can be updated when a node recovers.

**PropagationJob** — A job tracking delivery of one piece of content to a set of edge nodes. Holds a list of `NodeDeliveryTask` objects and derives its own `JobStatus` from their statuses after every task change.

**NodeDeliveryTask** — The unit of work: deliver content to one specific edge node. Owns `TaskStatus`, `attemptCount`, `lastAttemptAt`, and `failureReason`. The retry decision belongs here.

**ContentRoutingStrategy (interface)** — Given content metadata and the list of active nodes, returns the target subset. Injected into `CDNController`; new routing rules are new implementations.

**EdgeNodeClient** — The abstraction for pushing content to an edge node. Returns a `DeliveryResult`. In production this is an HTTP call; here it is a black box.

**RetryPolicy** — Encapsulates `maxAttempts` and per-attempt backoff delays. A separate class so retry behavior is independently replaceable.

**DeliveryResult** — A value object returned by `EdgeNodeClient.push()`. Carries success flag, HTTP status code, and error message on failure.

### Entity Responsibilities

| Entity | Responsibility |
|--------|----------------|
| `CDNController` | Entry point. Deduplicates events, creates jobs, selects nodes, dispatches tasks, handles retries and replay. |
| `S3Event` | Raw S3 upload event. The integration boundary object. |
| `ContentMetadata` | Normalized S3 object descriptor. Passed to edge nodes; decouples the system from S3's schema. |
| `EdgeNode` | One CDN edge location. Carries region, endpoint URL, and availability status. |
| `PropagationJob` | Groups all `NodeDeliveryTask` instances for one content push. Derives `JobStatus` from task statuses. |
| `NodeDeliveryTask` | Per-node delivery unit. Owns its own status, attempt count, and failure reason. The retry boundary. |
| `ContentRoutingStrategy` | Interface: returns the target subset of active nodes for a given piece of content. |
| `EdgeNodeClient` | Abstraction: pushes content metadata to a specific edge node. Returns `DeliveryResult`. |
| `RetryPolicy` | Encapsulates max attempts and per-attempt backoff delays. |
| `DeliveryResult` | Value object: success flag, HTTP status code, error message. |

### Relationships

```
S3Event
  ↓ handleS3Event()
CDNController
  ├── edgeNodes:       Map<id, EdgeNode>
  ├── jobs:            Map<id, PropagationJob>
  ├── processedEvents: Set<String>               // dedup key: "bucket/key:etag"
  ├── routingStrategy: ContentRoutingStrategy    // injected
  ├── edgeNodeClient:  EdgeNodeClient            // injected
  └── retryPolicy:     RetryPolicy

CDNController creates ──────────────────────────────────────────────────────┐
                                                                             ↓
                                                                  PropagationJob
                                                                    ├── content: ContentMetadata
                                                                    ├── status:  JobStatus
                                                                    └── tasks:   List<NodeDeliveryTask>
                                                                                      ├── node:         EdgeNode
                                                                                      ├── status:       TaskStatus
                                                                                      ├── attemptCount: int
                                                                                      └── failureReason: String?

CDNController calls ──────────────────────────────────────────────────────┐
                                                                           ↓
                                                               EdgeNodeClient.push(node, metadata)
                                                                           ↓
                                                                    DeliveryResult

<<interface>> ContentRoutingStrategy
  + selectNodes(metadata, availableNodes) -> List<EdgeNode>
        ^                   ^                        ^
        |                   |                        |
AllNodesStrategy   RegionFilterStrategy   ContentTypeRoutingStrategy

JobStatus:  PENDING | IN_PROGRESS | COMPLETED | PARTIALLY_FAILED | FAILED
TaskStatus: PENDING | IN_PROGRESS | SUCCESS   | FAILED
NodeStatus: ACTIVE  | INACTIVE
```

`balances` is replaced here by `processedEvents` — the deduplication guard. `jobs` is the flat registry of all propagation work, mirroring the flat `showtimes` or `groups` maps in other designs. Each `PropagationJob` owns its tasks the same way `Group` owns its expenses.

---

## Class Design

Top-down: `CDNController`, then `PropagationJob`, `NodeDeliveryTask`, `EdgeNode`, `ContentMetadata`, `S3Event`, the `ContentRoutingStrategy` interface and implementations, `EdgeNodeClient`, `RetryPolicy`, and value objects.

### CDNController

**Deriving State**

| Requirement | What CDNController must track |
|-------------|-------------------------------|
| "Duplicate events must be detected" | Set of already-processed dedup keys |
| "Routing strategy is pluggable" | A `ContentRoutingStrategy` reference |
| "Edge nodes can be registered and queried" | Registry of all edge nodes by ID |
| "Job status can be queried" | All propagation jobs by ID |
| "Retry behavior is configurable" | A `RetryPolicy` reference |
| "Push implementation is abstracted" | An `EdgeNodeClient` reference |

**Deriving Behavior**

| Need from requirements | Method on CDNController |
|------------------------|-------------------------|
| "Register an edge node" | `registerEdgeNode(node: EdgeNode) -> void` |
| "Receive and process an S3 upload event" | `handleS3Event(event: S3Event) -> PropagationJob` |
| "Query propagation status for an event" | `getJob(jobId: String) -> PropagationJob` |
| "Replay content to a recovered node" | `replayToNode(nodeId: String) -> void` |
| "Update a node's availability" | `setNodeStatus(nodeId, status: NodeStatus) -> void` |
| "Create a job with per-node tasks" | `createPropagationJob(metadata, nodes) -> PropagationJob` (private) |
| "Push to one edge node and handle result" | `dispatchDeliveryTask(task, metadata) -> void` (private) |
| "Recompute job status after any task change" | `updateJobStatus(jobId) -> void` (private) |

```
class CDNController:
    - edgeNodes:       Map<String, EdgeNode>
    - jobs:            Map<String, PropagationJob>
    - processedEvents: Set<String>
    - routingStrategy: ContentRoutingStrategy
    - edgeNodeClient:  EdgeNodeClient
    - retryPolicy:     RetryPolicy

    + CDNController(routingStrategy, edgeNodeClient, retryPolicy)
    + registerEdgeNode(node: EdgeNode) -> void
    + setNodeStatus(nodeId: String, status: NodeStatus) -> void
    + handleS3Event(event: S3Event) -> PropagationJob
    + getJob(jobId: String) -> PropagationJob
    + replayToNode(nodeId: String) -> void
    - createPropagationJob(metadata: ContentMetadata, nodes: List<EdgeNode>) -> PropagationJob
    - dispatchDeliveryTask(task: NodeDeliveryTask, metadata: ContentMetadata) -> void
    - updateJobStatus(jobId: String) -> void
```

---

### PropagationJob

**Deriving State**

| Requirement | What PropagationJob must track |
|-------------|-------------------------------|
| "Job aggregates per-node tasks" | List of `NodeDeliveryTask` objects |
| "Overall status derived from task statuses" | `status: JobStatus` — recomputed after every task change |
| "Track which content is being propagated" | `content: ContentMetadata` |
| "Know when the job finished" | `completedAt: DateTime?` — set when status reaches a terminal state |

**Deriving Behavior**

| Need | Method on PropagationJob |
|------|--------------------------|
| "Add tasks during job creation" | `addTask(task: NodeDeliveryTask) -> void` |
| "Dispatch loop iterates tasks" | `getTasks() -> List<NodeDeliveryTask>` |
| "Replay scans for failed tasks" | `getFailedTasks() -> List<NodeDeliveryTask>` |
| "Recompute status after any task changes" | `updateStatus() -> void` |

```
class PropagationJob:
    - id:          String
    - content:     ContentMetadata
    - tasks:       List<NodeDeliveryTask>
    - status:      JobStatus
    - createdAt:   DateTime
    - completedAt: DateTime?

    + PropagationJob(id, content, createdAt)
    + getId()          -> String
    + getContent()     -> ContentMetadata
    + getStatus()      -> JobStatus
    + getTasks()       -> List<NodeDeliveryTask>
    + getFailedTasks() -> List<NodeDeliveryTask>   // status == FAILED
    + addTask(task: NodeDeliveryTask) -> void
    + updateStatus() -> void                        // recomputes from task statuses
```

---

### NodeDeliveryTask

The per-node delivery unit. This is where retry isolation lives — a failure here affects only this node's task.

**Deriving State**

| Requirement | What NodeDeliveryTask must track |
|-------------|----------------------------------|
| "Each node has its own delivery status" | `status: TaskStatus` |
| "Retry up to configurable max" | `attemptCount: int` — incremented on each attempt start |
| "Backoff requires knowing attempt number" | `lastAttemptAt: DateTime?` |
| "Record why delivery failed" | `failureReason: String?` |
| "Know which node this is for" | Reference to `EdgeNode` |
| "Know which job this belongs to" | `jobId: String` (for `updateJobStatus` lookup) |

**Deriving Behavior**

| Need | Method on NodeDeliveryTask |
|------|---------------------------|
| "Begin an attempt — increment count" | `markInProgress() -> void` |
| "Record a successful push" | `markSuccess() -> void` |
| "Record a failed push" | `markFailed(reason: String) -> void` |
| "Check if the retry policy allows another attempt" | `canRetry(maxAttempts: int) -> boolean` |

```
class NodeDeliveryTask:
    - id:            String
    - jobId:         String
    - node:          EdgeNode
    - status:        TaskStatus
    - attemptCount:  int        // starts at 0; incremented by markInProgress()
    - lastAttemptAt: DateTime?
    - failureReason: String?

    + NodeDeliveryTask(id, jobId, node)   // status=PENDING, attemptCount=0
    + getId()            -> String
    + getJobId()         -> String
    + getNode()          -> EdgeNode
    + getStatus()        -> TaskStatus
    + getAttemptCount()  -> int
    + getFailureReason() -> String?
    + markInProgress()   -> void           // attemptCount++, status=IN_PROGRESS, lastAttemptAt=now()
    + markSuccess()      -> void           // status=SUCCESS
    + markFailed(reason: String) -> void   // status=FAILED, failureReason=reason
    + canRetry(maxAttempts: int) -> boolean  // status==FAILED and attemptCount < maxAttempts
```

---

### EdgeNode

```
class EdgeNode:
    - id:       String
    - region:   String     // e.g., "us-east-1", "eu-west-1", "ap-south-1"
    - endpoint: String     // e.g., "https://edge-us-east.cdn.example.com/ingest"
    - status:   NodeStatus

    + EdgeNode(id, region, endpoint)
    + getId()       -> String
    + getRegion()   -> String
    + getEndpoint() -> String
    + getStatus()   -> NodeStatus
    + setStatus(s: NodeStatus) -> void
```

---

### ContentMetadata

```
class ContentMetadata:
    - id:          String    // dedup key: "bucket/key:etag" — same as processedEvents entry
    - bucket:      String
    - key:         String    // e.g., "videos/promo.mp4"
    - size:        long      // bytes
    - contentType: String    // e.g., "video/mp4"
    - etag:        String    // S3 ETag; edge node verifies the fetched object against this
    - uploadedAt:  DateTime
    - sourceUrl:   String    // "s3://bucket/key" — the address edge node fetches from

    + ContentMetadata(id, bucket, key, size, contentType, etag, uploadedAt)
    + getSourceUrl() -> String
    // getters for all fields
```

`sourceUrl` is derived at construction: `"s3://" + bucket + "/" + key`. The edge node uses this to pull the object and then checks the ETag to confirm integrity.

---

### S3Event

```
class S3Event:
    - eventId:     String
    - bucket:      String
    - key:         String
    - size:        long
    - contentType: String
    - etag:        String
    - timestamp:   DateTime

    + S3Event(eventId, bucket, key, size, contentType, etag, timestamp)
    // getters for all fields
```

`S3Event` is the raw boundary object. `ContentMetadata` is the system's own model derived from it. Separating them means changing the event schema (e.g., a new S3 event format) only touches `handleS3Event`, not the rest of the design.

---

### ContentRoutingStrategy (interface)

```
interface ContentRoutingStrategy:
    + selectNodes(metadata: ContentMetadata,
                  availableNodes: List<EdgeNode>) -> List<EdgeNode>
```

`availableNodes` is always pre-filtered to `ACTIVE` nodes by `CDNController`. Implementations decide *which* of those active nodes are targeted, never *whether* a node is healthy.

```
class AllNodesStrategy implements ContentRoutingStrategy:
    + selectNodes(metadata, availableNodes) -> availableNodes    // push everywhere

class RegionFilterStrategy implements ContentRoutingStrategy:
    - allowedRegions: Set<String>
    + RegionFilterStrategy(regions: Set<String>)
    + selectNodes(metadata, availableNodes)
        return availableNodes.filter(n -> allowedRegions.contains(n.getRegion()))

class ContentTypeRoutingStrategy implements ContentRoutingStrategy:
    - rules: Map<String, Set<String>>    // contentType → set of node IDs to target
    + ContentTypeRoutingStrategy(rules)
    + selectNodes(metadata, availableNodes)
        targetIds = rules.getOrDefault(metadata.getContentType(), availableNodes.ids())
        return availableNodes.filter(n -> targetIds.contains(n.getId()))
```

---

### EdgeNodeClient

```
class EdgeNodeClient:
    + push(node: EdgeNode, metadata: ContentMetadata) -> DeliveryResult
    // Sends an HTTP POST to node.getEndpoint() with:
    //   { key, bucket, etag, sourceUrl, size, contentType }
    // Edge node fetches from sourceUrl, verifies etag, caches locally.
    // Returns DeliveryResult based on the HTTP response code.
```

---

### RetryPolicy

```
class RetryPolicy:
    - maxAttempts:   int            // e.g., 3
    - backoffDelays: List<Duration> // e.g., [5s, 30s, 120s]

    + RetryPolicy(maxAttempts, backoffDelays)
    + shouldRetry(task: NodeDeliveryTask) -> boolean
        return task.canRetry(maxAttempts)
    + getNextDelay(attemptCount: int) -> Duration
        index = min(attemptCount - 1, backoffDelays.size() - 1)
        return backoffDelays[index]
```

`getNextDelay` uses `min(index, last)` so that if `attemptCount` exceeds the number of configured delays, the last configured delay is reused rather than throwing an out-of-bounds error.

---

### DeliveryResult (value object)

```
class DeliveryResult:
    - success:      boolean
    - statusCode:   int?       // HTTP response code when available
    - errorMessage: String?

    + DeliveryResult(success, statusCode, errorMessage)
    + isSuccess()       -> boolean
    + getStatusCode()   -> int?
    + getErrorMessage() -> String?

    + static ok(statusCode: int) -> DeliveryResult
    + static failure(statusCode: int?, message: String) -> DeliveryResult
```

---

### Final Class Design

```
+--------------------------------------------------------------+
|                       CDNController                          |
+--------------------------------------------------------------+
| - edgeNodes:       Map<String, EdgeNode>                     |
| - jobs:            Map<String, PropagationJob>               |
| - processedEvents: Set<String>                               |
| - routingStrategy: ContentRoutingStrategy                    |
| - edgeNodeClient:  EdgeNodeClient                            |
| - retryPolicy:     RetryPolicy                               |
+--------------------------------------------------------------+
| + registerEdgeNode(node)                                     |
| + setNodeStatus(nodeId, status)                              |
| + handleS3Event(event: S3Event) -> PropagationJob            |
| + getJob(jobId) -> PropagationJob                            |
| + replayToNode(nodeId)                                       |
| - createPropagationJob(metadata, nodes) -> PropagationJob    |
| - dispatchDeliveryTask(task, metadata)                       |
| - updateJobStatus(jobId)                                     |
+--------------------------------------------------------------+
         |                                  |
 creates |                            uses  |
         v                                  v
+---------------------------+  +-----------------------------+
|      PropagationJob       |  |   ContentRoutingStrategy    |
+---------------------------+  |        <<interface>>        |
| - id, status              |  +-----------------------------+
| - content: ContentMetadata|       ^        ^        ^
| - tasks: List<Task>       |       |        |        |
| - createdAt, completedAt  |  AllNodes  Region  ContentType
+---------------------------+
| + addTask(task)           |
| + updateStatus()          |
| + getFailedTasks()        |
+---------------------------+
         |
   owns  | List<NodeDeliveryTask>
         v
+---------------------------+       +----------------------+
|    NodeDeliveryTask       |       |      EdgeNode        |
+---------------------------+  node +----------------------+
| - id, jobId               |------>| - id, region         |
| - status:   TaskStatus    |       | - endpoint: String   |
| - attemptCount: int       |       | - status: NodeStatus |
| - failureReason: String?  |       +----------------------+
+---------------------------+
| + markInProgress()        |       EdgeNodeClient
| + markSuccess()           |  .push(node, metadata) -> DeliveryResult
| + markFailed(reason)      |
| + canRetry(maxAttempts)   |
+---------------------------+

+----------------------------+    +-----------------------+
|      ContentMetadata       |    |     DeliveryResult    |
+----------------------------+    +-----------------------+
| - id, bucket, key          |    | - success: boolean    |
| - size, contentType        |    | - statusCode: int?    |
| - etag, uploadedAt         |    | - errorMessage: String|
| - sourceUrl: String        |    +-----------------------+
+----------------------------+
```

---

## Implementation

The four most important methods are `handleS3Event()` (deduplication + job orchestration), `dispatchDeliveryTask()` (delivery with per-node retry), `PropagationJob.updateStatus()` (derives aggregate state from task statuses), and `replayToNode()` (the node-recovery path).

### CDNController.handleS3Event()

**Core logic:**
1. Build the dedup key; if already seen, skip silently
2. Normalize `S3Event` → `ContentMetadata`
3. Filter to ACTIVE nodes; apply routing strategy
4. Create the `PropagationJob` with one `NodeDeliveryTask` per target node
5. Dispatch all tasks (asynchronously in production)
6. Return the job immediately — caller polls `getJob()` for status

**Edge cases:**
- Duplicate event (same bucket + key + ETag) — return null, no job created, no push
- No active nodes after filtering — throw before creating any job
- Routing strategy returns empty subset — same as above

```
handleS3Event(event: S3Event) -> PropagationJob
    dedupKey = event.getBucket() + "/" + event.getKey() + ":" + event.getEtag()
    if processedEvents.contains(dedupKey)
        return null    // duplicate event — already dispatched or dispatching

    processedEvents.add(dedupKey)

    metadata = ContentMetadata(
        id          = dedupKey,
        bucket      = event.getBucket(),
        key         = event.getKey(),
        size        = event.getSize(),
        contentType = event.getContentType(),
        etag        = event.getEtag(),
        uploadedAt  = event.getTimestamp()
    )

    activeNodes = edgeNodes.values().filter(n -> n.getStatus() == ACTIVE)
    targetNodes = routingStrategy.selectNodes(metadata, activeNodes)
    if targetNodes.isEmpty()
        throw Error("No eligible edge nodes for: " + metadata.getKey())

    job = createPropagationJob(metadata, targetNodes)
    jobs[job.getId()] = job

    for task in job.getTasks()
        dispatchDeliveryTask(task, metadata)    // async in production; serial here for clarity

    return job
```

**Inside `createPropagationJob`:**

```
createPropagationJob(metadata, targetNodes) -> PropagationJob
    job = PropagationJob(uuid(), metadata, now())
    for node in targetNodes
        task = NodeDeliveryTask(uuid(), job.getId(), node)
        job.addTask(task)
    job.updateStatus()    // initial update: all tasks PENDING → IN_PROGRESS
    return job
```

The job starts `IN_PROGRESS` because tasks exist and are `PENDING` — there is active work to do.

---

### CDNController.dispatchDeliveryTask()

**Core logic:**
1. Mark the task in-progress (increments `attemptCount`)
2. Call `edgeNodeClient.push()` — the actual delivery attempt
3. On success: mark the task succeeded, update job status
4. On failure: mark the task failed; if retries remain, schedule re-dispatch after backoff
5. After max retries exhausted: task stays `FAILED`; job's `updateStatus` handles the aggregate

**Edge cases:**
- `edgeNodeClient.push()` throws an exception — treat as delivery failure
- Network timeout counted as a failed attempt — same path as an HTTP error response

```
dispatchDeliveryTask(task: NodeDeliveryTask, metadata: ContentMetadata)
    task.markInProgress()

    try
        result = edgeNodeClient.push(task.getNode(), metadata)
    catch exception
        result = DeliveryResult.failure(null, exception.getMessage())

    if result.isSuccess()
        task.markSuccess()
    else
        task.markFailed(result.getErrorMessage())

        if retryPolicy.shouldRetry(task)
            delay = retryPolicy.getNextDelay(task.getAttemptCount())
            scheduleRetry(task, metadata, delay)
            // In production: submit to a delay queue (e.g., ScheduledExecutorService).
            // scheduleRetry re-invokes dispatchDeliveryTask after the delay.
            // We return here — updateJobStatus will be called when the retry eventually runs.
            return

    updateJobStatus(task.getJobId())
```

`retryPolicy.shouldRetry(task)` calls `task.canRetry(maxAttempts)`, which checks `status == FAILED and attemptCount < maxAttempts`. When the max is reached, `shouldRetry` returns false; `updateJobStatus` then sees the task's `FAILED` status and transitions the job accordingly.

---

### PropagationJob.updateStatus()

Derives the job's aggregate status from all task statuses. Called after every task state change.

```
updateStatus()
    statuses = tasks.map(t -> t.getStatus())

    if statuses.all(s -> s == SUCCESS)
        this.status      = COMPLETED
        this.completedAt = now()

    else if statuses.any(s -> s == PENDING or s == IN_PROGRESS)
        this.status = IN_PROGRESS    // tasks are still in flight or queued

    else if statuses.all(s -> s == FAILED)
        this.status      = FAILED
        this.completedAt = now()

    else
        // Mix of SUCCESS and FAILED — some nodes received it, some didn't
        this.status      = PARTIALLY_FAILED
        this.completedAt = now()
```

A job in `PARTIALLY_FAILED` or `FAILED` is terminal until `replayToNode()` creates new delivery attempts. `replayToNode` calls `updateJobStatus` again after re-dispatching, which can move the job back to `IN_PROGRESS` and eventually to `COMPLETED`.

---

### CDNController.replayToNode()

Used when an edge node that missed content comes back online and operator manually triggers recovery.

**Core logic:**
1. Verify the node exists and is now `ACTIVE`
2. Scan all jobs for tasks targeting this node that are in `FAILED` status
3. Re-dispatch those tasks — the same `dispatchDeliveryTask` path applies, with fresh retries

**Edge cases:**
- Node is still `INACTIVE` — throw; replaying to an inactive node would immediately fail again
- All tasks for this node already `SUCCESS` — no work to do (loop finds nothing to dispatch)

```
replayToNode(nodeId: String)
    node = edgeNodes[nodeId]
    if node == null
        throw Error("Edge node not found: " + nodeId)
    if node.getStatus() != ACTIVE
        throw Error("Cannot replay to inactive node: " + nodeId)

    for job in jobs.values()
        for task in job.getTasks()
            if task.getNode().getId() == nodeId and task.getStatus() == FAILED
                dispatchDeliveryTask(task, job.getContent())
                // dispatchDeliveryTask calls updateJobStatus internally,
                // so job status updates as each task resolves.
```

The replay reuses `dispatchDeliveryTask` entirely. No separate retry path.

---

## Design Patterns

### Strategy Pattern
`ContentRoutingStrategy` is the strategy interface. `AllNodesStrategy`, `RegionFilterStrategy`, and `ContentTypeRoutingStrategy` are concrete implementations. `CDNController` holds the injected strategy and calls `selectNodes()` without knowing which implementation is active.

Adding a new routing rule means implementing `ContentRoutingStrategy`. `CDNController`, `PropagationJob`, and `NodeDeliveryTask` are untouched.

### Job + Task Decomposition (Command Pattern)
`PropagationJob` is a unit of work that aggregates per-node `NodeDeliveryTask` commands. Each task is independently dispatchable and re-dispatchable. This is the structure that makes per-node retry isolation possible — if content propagation were one atomic operation, partial failure would require full re-execution.

### State Machine
Both `PropagationJob` (`JobStatus`) and `NodeDeliveryTask` (`TaskStatus`) are implicit state machines. `NodeDeliveryTask` transitions: `PENDING → IN_PROGRESS → SUCCESS` or `PENDING → IN_PROGRESS → FAILED → IN_PROGRESS → ...`. `PropagationJob` derives its state from its tasks rather than tracking transitions independently — `updateStatus()` recomputes on every call, so it is always consistent with the task states it reads.

### Information Expert
- `NodeDeliveryTask` owns `attemptCount` and `status` → it decides whether retrying is allowed (`canRetry`)
- `PropagationJob` owns its task list → it computes aggregate job status (`updateStatus`)
- `CDNController` owns `processedEvents` → it decides whether an event is a duplicate
- `RetryPolicy` owns backoff delays → it computes the next wait duration

---

## Verification

**Setup:** 3 edge nodes: US-EAST (us-east-1, ACTIVE), EU-WEST (eu-west-1, ACTIVE), AP-SOUTH (ap-south-1, ACTIVE). Routing: `AllNodesStrategy`. Retry: `maxAttempts=2`, `backoffDelays=[5s, 30s]`.

---

**Step 1 — S3 event arrives**

```
event = S3Event("evt-1", bucket="assets", key="videos/promo.mp4",
                size=52428800, contentType="video/mp4",
                etag="abc123def456", timestamp=T0)

handleS3Event(event):
  dedupKey = "assets/videos/promo.mp4:abc123def456"
  processedEvents = {}  → not duplicate
  processedEvents.add("assets/videos/promo.mp4:abc123def456")

  metadata = ContentMetadata(
      id="assets/videos/promo.mp4:abc123def456", bucket="assets",
      key="videos/promo.mp4", size=52428800, contentType="video/mp4",
      etag="abc123def456", uploadedAt=T0,
      sourceUrl="s3://assets/videos/promo.mp4"
  )

  activeNodes  = [US-EAST, EU-WEST, AP-SOUTH]
  targetNodes  = AllNodesStrategy.selectNodes(metadata, activeNodes)
              = [US-EAST, EU-WEST, AP-SOUTH]

  job J-001 created:
    tasks  = [T-1(US-EAST, PENDING), T-2(EU-WEST, PENDING), T-3(AP-SOUTH, PENDING)]
    status = IN_PROGRESS    (tasks are PENDING → any PENDING → IN_PROGRESS)
```

---

**Step 2 — T-1 (US-EAST) succeeds on first attempt**

```
dispatchDeliveryTask(T-1, metadata):
  T-1.markInProgress()    → attemptCount=1, status=IN_PROGRESS

  result = edgeNodeClient.push(US-EAST, metadata)
         = DeliveryResult.ok(200)

  T-1.markSuccess()       → status=SUCCESS

  updateJobStatus("J-001"):
    statuses = [SUCCESS, PENDING, PENDING]
    → any PENDING → J-001.status = IN_PROGRESS   ✓

State: T-1=SUCCESS, T-2=PENDING, T-3=PENDING, J-001=IN_PROGRESS
```

---

**Step 3 — T-2 (EU-WEST) fails first attempt, succeeds on retry**

```
Attempt 1:
  T-2.markInProgress()    → attemptCount=1, status=IN_PROGRESS
  result = edgeNodeClient.push(EU-WEST, metadata)
         = DeliveryResult.failure(503, "Connection timeout")
  T-2.markFailed("Connection timeout") → status=FAILED
  retryPolicy.shouldRetry(T-2): canRetry(2) → status==FAILED and 1 < 2 → true
  getNextDelay(attemptCount=1) = backoffDelays[0] = 5s
  scheduleRetry(T-2, metadata, 5s)
  return   ← updateJobStatus NOT called yet (retry pending)

--- 5 seconds pass ---

Attempt 2:
  T-2.markInProgress()    → attemptCount=2, status=IN_PROGRESS
  result = edgeNodeClient.push(EU-WEST, metadata)
         = DeliveryResult.ok(200)
  T-2.markSuccess()       → status=SUCCESS
  updateJobStatus("J-001"):
    statuses = [SUCCESS, SUCCESS, PENDING]
    → still PENDING → IN_PROGRESS

State: T-1=SUCCESS, T-2=SUCCESS (2 attempts), T-3=PENDING, J-001=IN_PROGRESS
```

---

**Step 4 — T-3 (AP-SOUTH) exhausts both attempts**

```
Attempt 1:
  T-3.markInProgress()    → attemptCount=1, status=IN_PROGRESS
  result = DeliveryResult.failure(503, "HTTP 503 Service Unavailable")
  T-3.markFailed("HTTP 503 Service Unavailable") → status=FAILED
  retryPolicy.shouldRetry(T-3): 1 < 2 → true
  scheduleRetry(T-3, metadata, 5s); return

--- 5 seconds pass ---

Attempt 2:
  T-3.markInProgress()    → attemptCount=2, status=IN_PROGRESS
  result = DeliveryResult.failure(503, "HTTP 503 Service Unavailable")
  T-3.markFailed("HTTP 503 Service Unavailable") → status=FAILED
  retryPolicy.shouldRetry(T-3): canRetry(2) → 2 < 2 is false → false
  No retry scheduled.

  updateJobStatus("J-001"):
    statuses = [SUCCESS, SUCCESS, FAILED]
    → not all SUCCESS, not all FAILED, no PENDING/IN_PROGRESS
    → J-001.status = PARTIALLY_FAILED, completedAt = T5
```

---

**Step 5 — Query job status**

```
getJob("J-001"):
  id          = "J-001"
  status      = PARTIALLY_FAILED
  completedAt = T5
  getFailedTasks() = [T-3 (AP-SOUTH, FAILED, attempts=2,
                          reason="HTTP 503 Service Unavailable")]
```

---

**Step 6 — Duplicate S3 event arrives**

```
event2 = S3Event("evt-2", bucket="assets", key="videos/promo.mp4",
                 etag="abc123def456", ...)    // same ETag — same bytes

handleS3Event(event2):
  dedupKey = "assets/videos/promo.mp4:abc123def456"
  processedEvents.contains(dedupKey) → true
  return null   // skipped ✓ — no new job, no re-push to US-EAST or EU-WEST
```

---

**Step 7 — AP-SOUTH recovers; replay triggered**

```
setNodeStatus("ap-south-1", ACTIVE)     // operator marks node recovered
replayToNode("ap-south-1"):
  node = edgeNodes["ap-south-1"] → ACTIVE ✓
  scan all jobs:
    J-001: T-3 targets ap-south-1 and status=FAILED → dispatch

    dispatchDeliveryTask(T-3, J-001.getContent()):
      T-3.markInProgress()    → attemptCount=3, status=IN_PROGRESS
      result = edgeNodeClient.push(AP-SOUTH, metadata)
             = DeliveryResult.ok(200)
      T-3.markSuccess()       → status=SUCCESS
      updateJobStatus("J-001"):
        statuses = [SUCCESS, SUCCESS, SUCCESS] → COMPLETED ✓
```

J-001 transitions from `PARTIALLY_FAILED` to `COMPLETED` after successful replay.

---

## Extensibility

### 1. "How would you add geo-based routing so a video only goes to European edge nodes?"

> "I'd implement `ContentRoutingStrategy` with a `MetadataAttributeRoutingStrategy`. `ContentMetadata` gains an optional `targetRegion` field set during ingest (by the caller or from a tagging convention). The strategy reads it: if set, filter `availableNodes` to that region; if null, return all nodes. Zero changes to `CDNController`, `PropagationJob`, or any other class — we swap the injected strategy."

```
class MetadataAttributeRoutingStrategy implements ContentRoutingStrategy:
    + selectNodes(metadata, availableNodes)
        region = metadata.getTargetRegion()    // null = no restriction
        if region == null
            return availableNodes
        return availableNodes.filter(n -> n.getRegion() == region)
```

The routing concern stays entirely inside the strategy. The controller never parses `targetRegion` directly.

---

### 2. "How would you add content prioritization so video files are pushed before CSS or JS?"

Right now all content types dispatch with equal urgency. A 50MB video and a 2KB CSS update compete on even footing for the dispatch slot.

> "I'd add a `ContentPriorityStrategy` interface that maps a `ContentMetadata` to an integer priority, and replace the sequential dispatch loop in `handleS3Event` with a `PriorityBlockingQueue<DispatchRequest>`. A pool of worker threads pulls highest-priority tasks first. Adding a new content type and its priority rank is one config entry in `ContentTypePriorityStrategy`. The delivery path — `dispatchDeliveryTask`, `NodeDeliveryTask`, `PropagationJob` — is completely unchanged."

```
interface ContentPriorityStrategy:
    + getPriority(metadata: ContentMetadata) -> int   // higher = dispatched sooner

class ContentTypePriorityStrategy implements ContentPriorityStrategy:
    - priorities: Map<String, Integer>
    // e.g., {"video/*": 10, "image/*": 5, "text/css": 1, "default": 3}
    + getPriority(metadata)
        for (pattern, priority) in priorities
            if metadata.getContentType().matches(pattern)
                return priority
        return priorities["default"]

// CDNController gains one more injected dependency:
- priorityStrategy: ContentPriorityStrategy
- dispatchQueue:    PriorityBlockingQueue<DispatchRequest>  // workers pull from here
```

---

### 3. "How would you automatically replay missed content when a down node comes back online, without operator intervention?"

Right now `replayToNode` is triggered manually. A node can be down for hours without the system noticing.

> "I'd add a `NodeHealthMonitor` background component that periodically pings each `INACTIVE` node's health endpoint. When a ping succeeds, it calls `controller.setNodeStatus(node.getId(), ACTIVE)` and then `controller.replayToNode(node.getId())`. The replay logic is already implemented and needs no changes. To bound memory, `jobs` would need a TTL-based eviction policy — jobs older than a configured window (say, 24 hours) are dropped and not replayed. Content older than the TTL would fall back to a manual re-upload or a pull-based refresh."

```
class NodeHealthMonitor:
    - controller:          CDNController
    - edgeNodeClient:      EdgeNodeClient
    - checkIntervalSeconds: int

    + start() -> void    // starts background polling loop
    - checkNode(node: EdgeNode) -> void
        result = edgeNodeClient.ping(node)   // lightweight health-check endpoint
        if result.isSuccess() and node.getStatus() == INACTIVE
            controller.setNodeStatus(node.getId(), ACTIVE)
            controller.replayToNode(node.getId())
```

`NodeHealthMonitor` is a separate class — not embedded in `CDNController` — so health-check frequency and ping logic are independently configurable and testable.
