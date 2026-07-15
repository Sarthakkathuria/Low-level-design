# Inventory Management System

An inventory management system tracks product stock across multiple warehouse locations. When inventory arrives, the system records it. When orders ship, the system deducts stock. The system can also transfer inventory between locations and alert managers when stock runs low.

---

## Requirements

You walk into the interview and get greeted with this prompt:

> "Design an inventory management system that tracks products across multiple warehouses. The system needs to handle adding and removing inventory, transferring stock between locations, and alerting when inventory runs low."

Before touching a whiteboard, spend 3–5 minutes narrowing that down.

### Clarifying Questions

**You:** "When you say 'multiple warehouses,' are we talking about a fixed set configured at startup, or can warehouses be added dynamically?"
**Interviewer:** "Fixed set. You can assume they're configured when the system initializes."

Good. We're not building warehouse lifecycle management.

**You:** "For the low-stock alerts, how granular is the threshold? Per product globally, or per product per warehouse?"
**Interviewer:** "Per product per warehouse. Different warehouses might need different thresholds for the same product. When stock drops below the threshold, trigger a notification."

Important. A product could be low in Warehouse A but fine in Warehouse B. Alerts are not global.

**You:** "How should the notification happen — emails, webhooks, or just an interface the caller implements?"
**Interviewer:** "Pluggable. The system calls some callback interface when stock is low. What happens after that is someone else's problem."

We're building the alert mechanism, not the delivery. That's a Strategy, not a hardcoded implementation.

**You:** "What about invalid operations? Can stock go negative?"
**Interviewer:** "Reject them. If someone tries to remove 100 units but we only have 50, that should fail. Same with transfers — validate before moving anything."

The system enforces a hard invariant: stock never goes below zero.

**You:** "Concurrency — multiple operations happening simultaneously on the same warehouse?"
**Interviewer:** "Yes, thread-safe. Multiple operations could be happening simultaneously — one warehouse receiving a shipment while another is fulfilling an order for the same product."

Synchronization is in scope from the start, not a cleanup pass at the end.

**You:** "What's out of scope? Product catalogs, order management?"
**Interviewer:** "Products exist externally. Orders and payments are handled upstream. Focus on the inventory tracking logic."

### Final Requirements

| # | Requirement |
|---|-------------|
| 1 | Track stock for products across a fixed set of warehouses, configured at startup |
| 2 | Add stock to a specific warehouse (receiving shipments) |
| 3 | Remove stock from a specific warehouse (fulfilling orders); reject if it would go negative |
| 4 | Check availability: given a product and quantity, return which warehouses can fulfill it |
| 5 | Transfer stock between warehouses atomically; reject if source has insufficient stock |
| 6 | Per-product per-warehouse low-stock alert threshold; fire a pluggable listener when stock drops below it |
| 7 | All operations are thread-safe; concurrent operations on the same warehouse serialize correctly |

**Out of scope:** product catalog management, order processing, payment, persistence, warehouse lifecycle.

---

## Core Entities and Relationships

There are two structural challenges here. The first is the **alert granularity**: alerts are scoped to a `(warehouseId, productId)` pair, not globally per product. That points to a composite key, like the `ClientEndpointKey` in a rate limiter. The second is **transfer atomicity with deadlock prevention**: moving stock from A to B requires holding both warehouses' locks simultaneously, which creates deadlock risk if two concurrent transfers lock in opposite order.

**InventoryManager** — The orchestrator. Entry point for all operations. Owns the warehouse registry, alert configs, and the alert listener. Routes every call, acquires warehouse locks, and fires alerts after releasing locks.

**Warehouse** — A physical location. Owns a `Map<productId, quantity>` for stock and a `ReentrantLock`. Does not acquire its own lock internally — all locking is managed by `InventoryManager`. This is deliberate: transfer requires holding two warehouse locks simultaneously, and if Warehouse managed its own lock, there would be no way to keep both held atomically without exposing the lock externally anyway.

**AlertConfig** — A value object: `warehouseId`, `productId`, `threshold`. Defines when to trigger an alert for a specific (warehouse, product) pair.

**AlertListener** — An interface with a single method. The caller injects a concrete implementation — email, webhook, log line — without the system knowing or caring which.

### Entity Responsibilities

| Entity | Responsibility |
|--------|----------------|
| `InventoryManager` | Entry point. Acquires locks, routes operations, checks thresholds, fires alerts. |
| `Warehouse` | Owns stock map and lock. Validates and applies stock changes when caller holds the lock. |
| `AlertConfig` | Value object: binds a (warehouse, product) pair to a numeric threshold. |
| `AlertListener` | Interface: receives low-stock notifications. Pluggable delivery mechanism. |

### Relationships

```
InventoryManager
  ├── warehouses:    Map<id, Warehouse>
  ├── alertConfigs:  Map<"warehouseId:productId", AlertConfig>
  └── alertListener: AlertListener

Warehouse
  ├── id:    String
  ├── name:  String
  ├── stock: Map<productId, quantity>
  └── lock:  ReentrantLock          ← acquired by InventoryManager, not by Warehouse itself
```

The lock lives on `Warehouse` (it protects Warehouse's data) but is acquired by `InventoryManager` (which controls multi-warehouse operations). This separation is the key design decision.

---

## Class Design

We design top-down: `InventoryManager` first, then `Warehouse`, then `AlertConfig` and `AlertListener`.

### InventoryManager

**Deriving State**

| Requirement | What InventoryManager must track |
|-------------|----------------------------------|
| "Track stock across warehouses" | Map from warehouse ID to Warehouse |
| "Per-product per-warehouse alert thresholds" | Map from composite key to AlertConfig |
| "Pluggable alert delivery" | An AlertListener reference |

**Deriving Behavior**

| Need from requirements | Method on InventoryManager |
|------------------------|---------------------------|
| "Add stock to a warehouse" | `addStock(warehouseId, productId, qty) -> void` |
| "Remove stock; reject if negative" | `removeStock(warehouseId, productId, qty) -> void` |
| "Check which warehouses can fulfill a quantity" | `checkAvailability(productId, qty) -> Map<String, Integer>` |
| "Transfer atomically between warehouses" | `transfer(sourceId, destId, productId, qty) -> void` |
| "Register an alert threshold" | `setAlertConfig(config: AlertConfig) -> void` |
| "Read current stock level" | `getStock(warehouseId, productId) -> int` |

```
class InventoryManager:
    - warehouses:    Map<String, Warehouse>
    - alertConfigs:  Map<String, AlertConfig>   // key: "warehouseId:productId"
    - alertListener: AlertListener

    + InventoryManager(warehouses: List<Warehouse>, alertListener: AlertListener)
    + addStock(warehouseId, productId, qty) -> void
    + removeStock(warehouseId, productId, qty) -> void
    + transfer(sourceId, destId, productId, qty) -> void
    + checkAvailability(productId, qty) -> Map<String, Integer>
    + getStock(warehouseId, productId) -> int
    + setAlertConfig(config: AlertConfig) -> void
```

`checkAndAlert` is a private helper called after any operation that reduces stock. It checks the threshold and fires the listener. It is always called **outside** any warehouse lock — the listener may do slow I/O (network calls, file writes) and must never be invoked while holding a lock.

---

### Warehouse

Warehouse owns the stock map and the lock. Its methods are not self-synchronized — `InventoryManager` always holds the lock before calling them. This is deliberate: transfer needs to hold two warehouse locks simultaneously, so the locking decision must live at the level that understands multi-warehouse operations.

**Deriving State**

| Requirement | What Warehouse must track |
|-------------|--------------------------|
| "Track product quantities" | `stock: Map<productId, int>` — starts empty for any product |
| "Thread-safe; concurrent operations serialize" | `lock: ReentrantLock` |

**Deriving Behavior**

| Need | Method on Warehouse |
|------|---------------------|
| "Read current stock for a product" | `getStock(productId) -> int` |
| "Add units to a product's count" | `addStock(productId, qty) -> void` |
| "Remove units; caller guarantees sufficient stock" | `removeStock(productId, qty) -> void` |
| "Check if sufficient stock exists" | `hasStock(productId, qty) -> boolean` |
| "Let InventoryManager acquire the lock" | `getLock() -> ReentrantLock` |

```
class Warehouse:
    - id:    String
    - name:  String
    - stock: Map<String, Integer>   // productId -> quantity; default 0 for unseen products
    - lock:  ReentrantLock

    + Warehouse(id, name)
    + getId() -> String
    + getLock() -> ReentrantLock
    + getStock(productId) -> int
    + hasStock(productId, qty) -> boolean
    + addStock(productId, qty) -> void
    + removeStock(productId, qty) -> void
```

`removeStock` trusts that the caller has already verified sufficient stock via `hasStock`. All validation is in `InventoryManager`. `Warehouse` does not duplicate the check.

---

### AlertConfig (value object)

```
class AlertConfig:
    - warehouseId: String
    - productId:   String
    - threshold:   int

    + AlertConfig(warehouseId, productId, threshold)
    + getWarehouseId() -> String
    + getProductId()   -> String
    + getThreshold()   -> int
```

---

### AlertListener (interface)

```
interface AlertListener:
    + onLowStock(warehouseId: String, productId: String, currentQty: int) -> void
```

No return value — the system fires and forgets. Whether the listener sends an email, calls a webhook, or logs a line is outside the system's concern.

**Why not the Observer pattern with a registered list of listeners?**

The requirement says "pluggable callback interface," not multi-cast. One listener is sufficient. If multi-cast were needed, you'd wrap multiple listeners in a `CompositeAlertListener` — an additive change, not a redesign.

---

### Final Class Design

```
+----------------------------------+
|        InventoryManager          |
+----------------------------------+
| - warehouses:   Map<id,Warehouse>|
| - alertConfigs: Map<key,Alert..> |
| - alertListener: AlertListener   |
+----------------------------------+
| + addStock(wId, pId, qty)        |
| + removeStock(wId, pId, qty)     |
| + transfer(src, dst, pId, qty)   |
| + checkAvailability(pId, qty)    |
| + getStock(wId, pId)             |
| + setAlertConfig(config)         |
+----------------------------------+
        |                   |
  owns  |             fires |
        v                   v
+--------------------+   +---------------------------+
|     Warehouse      |   |     <<interface>>         |
+--------------------+   |     AlertListener         |
| - id: String       |   +---------------------------+
| - name: String     |   | + onLowStock(wId,pId,qty) |
| - stock: Map       |   +---------------------------+
| - lock: RLock      |
+--------------------+
| + getLock()        |
| + getStock(pId)    |
| + hasStock(pId,qty)|
| + addStock(pId,qty)|
| + removeStock(...) |
+--------------------+

+--------------------+
|     AlertConfig    |   (value object)
+--------------------+
| - warehouseId      |
| - productId        |
| - threshold: int   |
+--------------------+
```

---

## Implementation

The three most interesting methods are `removeStock()` (validation + lock + alert-outside-lock pattern), `transfer()` (atomicity across two warehouses with deadlock prevention), and `checkAvailability()` (read across all warehouses).

### removeStock()

**Core logic:**
1. Resolve the warehouse
2. Acquire the warehouse lock
3. Validate sufficient stock — reject before any modification
4. Remove stock; capture the resulting quantity
5. Release the lock
6. Check threshold and fire alert if needed — always outside the lock

The alert fires outside the lock because the `AlertListener` may do slow or blocking work. Holding a warehouse lock while an external service times out would serialize all other operations on that warehouse. Capture the quantity under the lock, release, then decide whether to alert.

**Edge cases:**
- Warehouse ID not found
- Product not in stock at this warehouse — treated as zero
- Removal would make quantity negative — rejected before any modification
- `qty` is zero or negative — reject at entry

```
removeStock(warehouseId, productId, qty)
    if qty <= 0
        throw Error("Quantity must be positive")

    warehouse = warehouses[warehouseId]
    if warehouse == null
        throw Error("Warehouse not found: " + warehouseId)

    int remaining
    warehouse.getLock().lock()
    try
        if !warehouse.hasStock(productId, qty)
            throw Error("Insufficient stock: requested " + qty
                        + ", available " + warehouse.getStock(productId))
        warehouse.removeStock(productId, qty)
        remaining = warehouse.getStock(productId)
    finally
        warehouse.getLock().unlock()    // always release, even on exception

    checkAndAlert(warehouseId, productId, remaining)
```

`addStock` follows the same lock/unlock pattern but skips the validation and alert check — stock only rises on add.

---

### transfer()

Transfer is the most nuanced operation. Moving stock from A to B requires holding both warehouses' locks simultaneously. If two concurrent transfers — A→B and B→A — each acquire their first lock before the other releases it, they deadlock.

**Deadlock prevention:** always acquire warehouse locks in a consistent total order. We use lexicographic ordering of warehouse IDs. Two transfers involving the same pair of warehouses will always acquire in the same order, so neither can block the other's first acquisition.

**Core logic:**
1. Resolve source and destination
2. Guard against same-warehouse transfer
3. Determine lock acquisition order by comparing warehouse IDs
4. Acquire both locks in that order
5. Validate source has sufficient stock
6. Remove from source, add to destination — both under both locks
7. Capture resulting quantities
8. Release both locks in reverse order
9. Check and fire alert for source — outside locks

**Edge cases:**
- Source or destination not found
- Source equals destination
- Source has insufficient stock — reject before any modification; destination is untouched

```
transfer(sourceId, destId, productId, qty)
    if qty <= 0
        throw Error("Quantity must be positive")
    if sourceId == destId
        throw Error("Source and destination must differ")

    source = warehouses[sourceId]
    dest   = warehouses[destId]
    if source == null: throw Error("Source warehouse not found: " + sourceId)
    if dest   == null: throw Error("Destination warehouse not found: " + destId)

    // Consistent lock ordering to prevent deadlock
    first, second = sourceId < destId ? (source, dest) : (dest, source)

    first.getLock().lock()
    second.getLock().lock()
    try
        if !source.hasStock(productId, qty)
            throw Error("Insufficient stock for transfer: requested " + qty
                        + ", available " + source.getStock(productId))
        source.removeStock(productId, qty)
        dest.addStock(productId, qty)
        sourceRemaining = source.getStock(productId)
    finally
        second.getLock().unlock()
        first.getLock().unlock()

    checkAndAlert(sourceId, productId, sourceRemaining)
    // No alert for destination — stock rose, not fell
```

The two-warehouse lock pattern here is identical to the per-directory locking for `move()` in a file system. The same invariant applies: always acquire in a globally consistent order, always release in reverse.

---

### checkAvailability()

Returns all warehouses where current stock for the product meets or exceeds the required quantity.

**Core logic:** iterate all warehouses, read each under its own lock, collect those meeting the threshold.

Each warehouse is locked independently — we don't need a snapshot of all warehouses at the same instant, just a per-warehouse consistent read.

```
checkAvailability(productId, requiredQty)
    result = {}
    for (id, warehouse) in warehouses
        warehouse.getLock().lock()
        try
            qty = warehouse.getStock(productId)
        finally
            warehouse.getLock().unlock()
        if qty >= requiredQty
            result[id] = qty
    return result
```

---

### checkAndAlert() — private helper

Called after every stock-reducing operation, always outside any warehouse lock.

```
checkAndAlert(warehouseId, productId, currentQty)
    key    = warehouseId + ":" + productId
    config = alertConfigs[key]
    if config != null and currentQty < config.getThreshold()
        alertListener.onLowStock(warehouseId, productId, currentQty)
```

---

## Verification

**Setup:**
- WH-A and WH-B configured at startup
- AlertConfig: WH-A, P001, threshold = 10
- AlertListener: logs to console

**Add stock to WH-A:**
```
addStock("WH-A", "P001", 50)
  warehouse = warehouses["WH-A"]  ✓
  lock.lock()
  warehouse.addStock("P001", 50)  → stock["P001"] = 50
  lock.unlock()
  State: WH-A[P001] = 50
```

**Remove 45 units — triggers alert:**
```
removeStock("WH-A", "P001", 45)
  warehouse.hasStock("P001", 45)  → 50 >= 45  ✓
  warehouse.removeStock("P001", 45)  → stock["P001"] = 5
  remaining = 5
  lock.unlock()

  checkAndAlert("WH-A", "P001", 5)
    threshold = 10
    5 < 10  → alertListener.onLowStock("WH-A", "P001", 5)  ← alert fires
  State: WH-A[P001] = 5
```

**Transfer 20 units from WH-B to WH-A — insufficient stock, rejected:**
```
transfer("WH-B", "WH-A", "P001", 20)
  source = WH-B, dest = WH-A
  lock order: "WH-A" < "WH-B"  →  first = WH-A, second = WH-B
  WH-A.lock(), WH-B.lock()
  source.hasStock("P001", 20)  → WH-B has 0  ✗
  throw Error("Insufficient stock for transfer: requested 20, available 0")
  WH-B.unlock(), WH-A.unlock()
  State: unchanged
```

**Add 30 to WH-B, then transfer 20 to WH-A:**
```
addStock("WH-B", "P001", 30)  → WH-B[P001] = 30

transfer("WH-B", "WH-A", "P001", 20)
  lock order: "WH-A" < "WH-B"  →  first = WH-A, second = WH-B
  WH-A.lock(), WH-B.lock()
  source (WH-B).hasStock("P001", 20)  → 30 >= 20  ✓
  WH-B.removeStock("P001", 20)  → WH-B[P001] = 10
  WH-A.addStock("P001", 20)    → WH-A[P001] = 25
  sourceRemaining = 10
  WH-B.unlock(), WH-A.unlock()

  checkAndAlert("WH-B", "P001", 10)
    no AlertConfig for WH-B/P001  → no alert
  State: WH-A[P001] = 25, WH-B[P001] = 10
```

**Concurrent removals — Thread-1 and Thread-2 both remove 30 from WH-A (which has 25):**
```
// Stock is 25. Both threads race to removeStock("WH-A", "P001", 30)

Thread-1: warehouse.getLock().lock()  // acquires first
Thread-2: warehouse.getLock().lock()  // blocks — waits

Thread-1: hasStock("P001", 30)  → 25 >= 30?  ✗
  throw Error("Insufficient stock")
  lock.unlock()

Thread-2: lock.acquire() succeeds
  hasStock("P001", 30)  → 25 >= 30?  ✗
  throw Error("Insufficient stock")
  lock.unlock()

Both fail cleanly. Stock remains 25.  ✓
```

**Deadlock scenario prevented — concurrent A→B and B→A transfers:**
```
Thread-1: transfer("WH-A", "WH-B", ...)
  lock order: "WH-A" < "WH-B"  →  acquires WH-A first, then WH-B

Thread-2: transfer("WH-B", "WH-A", ...)
  lock order: "WH-A" < "WH-B"  →  acquires WH-A first, then WH-B

Both acquire in the same order. One acquires WH-A, the other waits.
No circular dependency. No deadlock.  ✓
```

---

## Extensibility

### 1. "How would you add a soft reservation — hold stock before an order is confirmed?"

Right now stock is either available or removed. A soft hold lets an order reserve stock before payment clears, preventing oversell without committing the deduction.

> "I'd add a `reserved: Map<productId, int>` map to Warehouse alongside `stock`. Available quantity becomes `stock[productId] - reserved[productId]`. A new `reserveStock(warehouseId, productId, qty) -> reservationId` method acquires the lock, validates available (not just total) quantity, increments reserved, and returns a UUID. Confirming an order calls `confirmReservation(reservationId)` which deducts from stock and clears the reserved count. Cancelling calls `releaseReservation(reservationId)` which decrements reserved. The lock, alert, and transfer logic are all unchanged — `hasStock` just changes its arithmetic to check available rather than total."

---

### 2. "How would you add an audit trail — record every stock change with before and after quantities?"

Right now changes are applied and forgotten. Audit requirements need an immutable history.

> "I'd add a `List<StockEvent>` (or a write to an append-only store) and populate it inside each operation, still under the warehouse lock, so the recorded before/after quantities are consistent with the actual change. `StockEvent` is a value object: `timestamp`, `warehouseId`, `productId`, `operation` (ADD/REMOVE/TRANSFER_IN/TRANSFER_OUT), `delta`, `before`, `after`. No existing class changes structure — each mutating method in InventoryManager appends an event after modifying stock, while still holding the lock. The `AlertListener` and `Warehouse` are untouched."

---

### 3. "How would you scale this to a distributed system with multiple application servers?"

The current per-Warehouse `ReentrantLock` only works within a single process. Two servers can both see sufficient stock and both approve removals that together exceed actual supply.

> "I'd replace the in-process `ReentrantLock` with a distributed lock — Redis `SET NX PX` keyed by `warehouseId` is the standard approach. Stock levels move from in-memory maps to a shared store (Redis hash or a database table). The `removeStock` and `transfer` logic stays structurally identical: acquire the distributed lock, validate, modify, release. For transfer, acquire both distributed locks in the same consistent lexicographic order to prevent distributed deadlock. The tradeoff is latency — each operation now makes network round-trips. An alternative is optimistic concurrency: read stock, compute new value, write with a conditional (`UPDATE WHERE qty = expected_value`), retry on conflict. Optimistic works well when contention is low; the lock-based approach is simpler to reason about under high contention."
