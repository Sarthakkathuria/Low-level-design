# Elevator Control System

An elevator control system manages multiple elevator cars inside a building. It receives two kinds of floor requests — passengers pressing hallway buttons and passengers pressing buttons inside a car — and moves each elevator efficiently to service them. The system decides which elevator to dispatch, tracks each car's position and direction, and advances movement one floor at a time.

---

## Requirements

You walk into the interview and get greeted with this prompt:

> "Design an elevator control system for a building. The system should handle multiple elevators, floor requests, and move elevators efficiently to service requests."

### Clarifying Questions

**You:** "Are there two types of requests — external buttons pressed in the hallway and internal buttons pressed inside the elevator?"
**Interviewer:** "Yes, both. External requests include a direction (up or down). Internal requests target a specific floor inside a specific elevator."

**You:** "Is the number of elevators and floors fixed at startup, or dynamic?"
**Interviewer:** "Fixed at startup. Treat both as configurable construction parameters."

**You:** "What algorithm determines which elevator gets dispatched to an external request? Is that fixed, or should it be swappable?"
**Interviewer:** "It should be swappable. Use nearest-elevator as the default."

**You:** "When all elevators are busy, what happens to a new request?"
**Interviewer:** "Assign it to the best available elevator anyway. The elevator will service it after its current stops."

**You:** "Do elevators have passenger capacity limits?"
**Interviewer:** "Out of scope for the core design, but keep it extensible."

**You:** "Should we model door open/close timing or just focus on movement?"
**Interviewer:** "Just movement. Doors are out of scope."

**You:** "Real-time threading, or a step-based simulation?"
**Interviewer:** "Step-based simulation for now. We can discuss concurrency as an extension."

**You:** "Emergency stops, maintenance mode, floor access restrictions?"
**Interviewer:** "All out of scope."

### Final Requirements

| # | Requirement |
|---|-------------|
| 1 | System manages a configurable number of elevators across a configurable number of floors |
| 2 | External requests carry a floor number and intended direction (UP/DOWN) — pressed in the hallway |
| 3 | Internal requests carry an elevator ID and a target floor — pressed inside the car |
| 4 | System dispatches external requests to the most suitable elevator using a pluggable algorithm |
| 5 | Internal requests are routed directly to the named elevator, bypassing dispatch |
| 6 | Each elevator tracks its current floor, movement state, and the set of floor stops it must service |
| 7 | Elevators move one floor per simulation step and stop when the current floor has a pending stop |
| 8 | When an elevator exhausts stops in its current direction, it reverses if stops remain in the other direction; otherwise it becomes idle |
| 9 | The dispatch algorithm is configurable at construction; default is nearest-elevator |

**Out of scope:** capacity limits, door timing, emergency stops, maintenance mode, concurrency, UI.

---

## Core Entities and Relationships

The central challenge here is **elevator movement and dispatching without a monolithic controller**. A naive approach would put all logic in one `ElevatorController` god class — tracking all positions, deciding which elevator to send, and moving cars in one giant loop. That class would be untestable and closed to new dispatch algorithms.

Two patterns solve this cleanly:

**Strategy pattern** for dispatch: selecting which elevator services an external request is an independent, swappable algorithm. The `DispatchStrategy` interface isolates that decision so a new algorithm (round-robin, zone-based, load-balanced) can be dropped in without touching anything else.

**LOOK algorithm** per elevator: each elevator maintains two sorted stop sets — one for upward travel, one for downward — and services them in direction order before reversing. This models real elevator behavior and lives entirely inside the `Elevator` class. No central controller tells a car how to move; it steers itself.

**ElevatorSystem** — The orchestrator and public entry point. Receives external and internal requests, delegates dispatch to the strategy, routes internal requests directly to the named elevator by ID, and advances the simulation with `step()`.

**Elevator** — A single elevator car. Owns its own state: current floor, movement direction, and two sorted stop sets (`upQueue`, `downQueue`). Its `step()` method moves it one floor and removes the stop if it arrives. It manages direction reversal and idle transitions independently. No external class reaches into its queues.

**ElevatorState** — Enum: `IDLE`, `MOVING_UP`, `MOVING_DOWN`. An elevator starts `IDLE`. Receiving a stop makes it `MOVING_UP` or `MOVING_DOWN`. Exhausting both queues returns it to `IDLE`.

**Direction** — Enum: `UP`, `DOWN`. Carried by external requests to indicate which direction the waiting passenger intends to travel. Used by the dispatch strategy to determine whether a moving elevator is a compatible match.

**DispatchStrategy** — The strategy interface. Takes the full list of elevators and an `ExternalRequest`, returns the elevator that should service it. Callers never know which algorithm is running.

**NearestElevatorStrategy** — Concrete strategy. Scores each elevator by proximity and directional compatibility: idle elevators score by distance; elevators already moving toward the request floor in the right direction score highest; elevators moving away score zero. Returns the highest-scoring elevator.

**ExternalRequest** — Value object for a hallway button press: `(floor, direction)`. Created by the caller; consumed by the dispatch strategy.

**InternalRequest** — Value object for an in-elevator button press: `(elevatorId, targetFloor)`. Routed directly to the named elevator, bypassing the dispatch strategy entirely.

### Entity Responsibilities

| Entity | Responsibility |
|--------|----------------|
| `ElevatorSystem` | Entry point. Routes requests, drives simulation `step()`, holds elevators and active strategy. |
| `Elevator` | Owns movement state and stop queues. Steps itself forward one floor. Manages direction reversal and idle transitions. |
| `ElevatorState` | Enum: `IDLE`, `MOVING_UP`, `MOVING_DOWN`. Models the three movement states of an elevator car. |
| `Direction` | Enum: `UP`, `DOWN`. Carried by external requests to indicate passenger intent. |
| `DispatchStrategy` | Interface. Selects the best elevator for an external request. No mutation — selection only. |
| `NearestElevatorStrategy` | Concrete strategy. Scores elevators by distance and directional compatibility; returns highest-scoring. |
| `ExternalRequest` | Value object. Hallway button press: floor + intended direction. |
| `InternalRequest` | Value object. In-elevator button press: elevator ID + target floor. |

### Relationships

```
ElevatorSystem
  ├── elevators:        List<Elevator>
  └── dispatchStrategy: DispatchStrategy
                              │
                              │ selects from
                              ▼
                  ┌──────────────────────────┐
                  │         Elevator         │
                  ├──────────────────────────┤
                  │ id:           String     │
                  │ currentFloor: int        │
                  │ state: ElevatorState     │
                  │ upQueue:   TreeSet<int>  │   ascending  → picks first()
                  │ downQueue: TreeSet<int>  │   descending → picks first()
                  └──────────────────────────┘

<<interface>> DispatchStrategy
  + selectElevator(elevators, request) -> Elevator
          ^
          │
NearestElevatorStrategy
  - totalFloors: int

ExternalRequest (value object)      InternalRequest (value object)
  ├── floor:     int                  ├── elevatorId:  String
  └── direction: Direction            └── targetFloor: int
```

Each `Elevator` is self-steering: it knows its floor, its queued stops, and how to advance one step. `ElevatorSystem` coordinates them but never reaches into elevator state — it calls `elevator.addStop(floor)` and `elevator.step()`, nothing more.

---

## Class Design

Top-down: `ElevatorSystem`, then `Elevator`, then the enums, then `DispatchStrategy` and `NearestElevatorStrategy`, and finally value objects.

### ElevatorSystem

**Deriving State**

| Requirement | What ElevatorSystem must track |
|-------------|-------------------------------|
| "Multiple elevators in a building" | The list of all elevator instances |
| "Configurable dispatch algorithm" | The active `DispatchStrategy` |
| "Fixed number of floors" | `totalFloors` — used to validate requests and score dispatch |

**Deriving Behavior**

| Need from requirements | Method on ElevatorSystem |
|------------------------|--------------------------|
| "Hallway button pressed" | `requestExternal(floor, direction) -> void` |
| "In-elevator button pressed" | `requestInternal(elevatorId, targetFloor) -> void` |
| "Advance simulation one tick" | `step() -> void` |
| "Observe all elevator states" | `getElevators() -> List<Elevator>` |

```
class ElevatorSystem:
    - elevators:        List<Elevator>
    - dispatchStrategy: DispatchStrategy
    - totalFloors:      int

    + ElevatorSystem(numElevators: int, totalFloors: int, strategy: DispatchStrategy)
    + requestExternal(floor: int, direction: Direction) -> void
    + requestInternal(elevatorId: String, targetFloor: int) -> void
    + step() -> void
    + getElevators() -> List<Elevator>
```

---

### Elevator

**Deriving State**

| Requirement | What Elevator must track |
|-------------|--------------------------|
| "Current floor and movement state" | `currentFloor: int`, `state: ElevatorState` |
| "Service stops in current direction before reversing" | `upQueue: TreeSet<int>` (ascending), `downQueue: TreeSet<int>` (descending) |
| "Identifiable for internal requests" | `id: String` |

`upQueue` is sorted ascending — the next up-stop is `first()` (lowest unreached floor above). `downQueue` is sorted descending — the next down-stop is `first()` (highest unreached floor below). Using sorted sets prevents duplicate stops and keeps next-stop lookup O(1).

**Deriving Behavior**

| Need | Method on Elevator |
|------|-------------------|
| "Add a stop to this elevator" | `addStop(floor: int) -> void` |
| "Move one floor in current direction" | `step() -> void` |
| "Expose state for dispatch scoring" | `getCurrentFloor() -> int`, `getState() -> ElevatorState` |

```
class Elevator:
    - id:           String
    - currentFloor: int
    - state:        ElevatorState
    - upQueue:      TreeSet<int>     // ascending  — next up-stop is first()
    - downQueue:    TreeSet<int>     // descending — next down-stop is first()

    + Elevator(id: String, startFloor: int)
    + addStop(floor: int) -> void
    + step() -> void
    + getId()           -> String
    + getCurrentFloor() -> int
    + getState()        -> ElevatorState
```

---

### ElevatorState and Direction (enums)

```
enum ElevatorState:
    IDLE, MOVING_UP, MOVING_DOWN

enum Direction:
    UP, DOWN
```

---

### DispatchStrategy (interface)

Every algorithm implements this. It receives the full elevator list and the external request and returns exactly one elevator. It never mutates elevator state — pure selection.

```
interface DispatchStrategy:
    + selectElevator(elevators: List<Elevator>, request: ExternalRequest) -> Elevator
```

---

### NearestElevatorStrategy implements DispatchStrategy

Scores each elevator on a single scale and returns the highest-scoring one.

**Scoring rules (higher is better):**
- `IDLE`: `score = totalFloors - distance`. Closer idle elevators score higher.
- `MOVING_UP`, request floor is at or above current floor, request direction is `UP`: `score = totalFloors - distance + totalFloors`. En-route bonus — this elevator will pass through the requested floor.
- `MOVING_DOWN`, request floor is at or below current floor, request direction is `DOWN`: same en-route bonus.
- All other cases (moving away, wrong direction): `score = 0`.

```
class NearestElevatorStrategy implements DispatchStrategy:
    - totalFloors: int

    + NearestElevatorStrategy(totalFloors: int)
    + selectElevator(elevators: List<Elevator>, request: ExternalRequest) -> Elevator
```

---

### Value Objects

```
class ExternalRequest:
    - floor:     int
    - direction: Direction

    + ExternalRequest(floor, direction)
    + getFloor()     -> int
    + getDirection() -> Direction

class InternalRequest:
    - elevatorId:  String
    - targetFloor: int

    + InternalRequest(elevatorId, targetFloor)
    + getElevatorId()  -> String
    + getTargetFloor() -> int
```

---

### Final Class Design

```
+-------------------------------------------+
|            ElevatorSystem                 |
+-------------------------------------------+
| - elevators:        List<Elevator>        |
| - dispatchStrategy: DispatchStrategy      |
| - totalFloors:      int                   |
+-------------------------------------------+
| + requestExternal(floor, direction)       |
| + requestInternal(elevatorId, floor)      |
| + step()                                  |
| + getElevators() -> List<Elevator>        |
+-------------------------------------------+
         |                      |
   owns  | List<Elevator>       | uses
         v                      v
+------------------------+    +-------------------------------+
|        Elevator        |    |       <<interface>>           |
+------------------------+    |      DispatchStrategy         |
| - id:           String |    +-------------------------------+
| - currentFloor: int    |    | + selectElevator(             |
| - state: ElevatorState |    |     elevators, request)       |
| - upQueue:   TreeSet   |    |     -> Elevator               |
| - downQueue: TreeSet   |    +-------------------------------+
+------------------------+                  ^
| + addStop(floor)       |                  |
| + step()               |    +-----------------------------+
| + getCurrentFloor()    |    | NearestElevatorStrategy     |
| + getState()           |    +-----------------------------+
+------------------------+    | - totalFloors: int          |
                              +-----------------------------+
                              | + selectElevator(...)       |
                              +-----------------------------+

+----------------------------+    +----------------------------+
|      ExternalRequest       |    |      InternalRequest       |
+----------------------------+    +----------------------------+
| - floor:     int           |    | - elevatorId:  String      |
| - direction: Direction     |    | - targetFloor: int         |
+----------------------------+    +----------------------------+

enum ElevatorState          enum Direction
  IDLE                        UP
  MOVING_UP                   DOWN
  MOVING_DOWN
```

---

## Implementation

The three most interesting pieces are `Elevator.step()` (the LOOK algorithm — movement, stopping, direction reversal, idle transition), `NearestElevatorStrategy.selectElevator()` (the scoring logic), and `ElevatorSystem.requestExternal()` (validation and dispatch). We'll also walk through `Elevator.addStop()` since it drives queue insertion and initial direction assignment.

### Elevator.step()

**Core logic:**
1. If `IDLE`, nothing to do — return immediately
2. Move one floor in the current direction
3. If the new floor is in the active queue, remove it (stop here)
4. If the active queue is now empty, check the opposite queue:
   - Stops remain in opposite direction → reverse; set new state
   - Both queues empty → transition to `IDLE`

**Edge cases:**
- `IDLE` elevator receives `step()` — no-op, safe to call at any time
- Elevator drains its up-queue mid-trip and finds a down-queue — reverses without ever going idle
- Both queues empty after removing a stop — transitions to `IDLE` at that floor

```
step()
    if state == IDLE
        return

    if state == MOVING_UP
        currentFloor += 1
        upQueue.remove(currentFloor)           // stop here if this floor was queued
        if upQueue.isEmpty()
            if !downQueue.isEmpty()
                state = MOVING_DOWN            // reverse — stops remain below
            else
                state = IDLE                   // nothing left to do

    else if state == MOVING_DOWN
        currentFloor -= 1
        downQueue.remove(currentFloor)
        if downQueue.isEmpty()
            if !upQueue.isEmpty()
                state = MOVING_UP
            else
                state = IDLE
```

The elevator never skips a floor — one `step()` call equals exactly one floor of travel. The sorted sets keep the next stop at `first()` without any searching.

---

### Elevator.addStop()

**Core logic:**
1. Ignore the stop if it equals the current floor — already here
2. If the stop is above the current floor, add it to `upQueue`
3. If the stop is below the current floor, add it to `downQueue`
4. If the elevator is `IDLE`, assign the initial direction and update state

**Edge cases:**
- Stop equals current floor — no-op; the caller has nothing to wait for
- `IDLE` elevator receives its first stop — must choose a direction and begin moving

```
addStop(floor)
    if floor == currentFloor
        return

    if floor > currentFloor
        upQueue.add(floor)
        if state == IDLE
            state = MOVING_UP

    else                        // floor < currentFloor
        downQueue.add(floor)
        if state == IDLE
            state = MOVING_DOWN
```

Duplicate stops are silently absorbed — `TreeSet.add()` is idempotent. Adding a stop to an already-moving elevator never changes the elevator's current direction; it is simply queued for later.

---

### NearestElevatorStrategy.selectElevator()

**Core logic:**
1. For each elevator, compute a score based on state and distance to the requested floor
2. Track the highest score and the elevator that produced it
3. Return the best elevator after scanning all

**Edge cases:**
- All elevators moving away from the request — they all score 0; the first one in the list is returned (any elevator is better than none)
- Tie in score — first elevator encountered in the list wins

```
selectElevator(elevators, request)
    bestElevator = null
    bestScore    = -1

    for elevator in elevators
        distance = abs(elevator.getCurrentFloor() - request.getFloor())
        score    = 0

        if elevator.getState() == IDLE
            score = totalFloors - distance

        else if elevator.getState() == MOVING_UP
                and request.getFloor() >= elevator.getCurrentFloor()
                and request.getDirection() == UP
            score = totalFloors - distance + totalFloors    // en-route bonus

        else if elevator.getState() == MOVING_DOWN
                and request.getFloor() <= elevator.getCurrentFloor()
                and request.getDirection() == DOWN
            score = totalFloors - distance + totalFloors

        // else: moving away or wrong direction — score stays 0

        if score > bestScore
            bestScore    = score
            bestElevator = elevator

    return bestElevator
```

The en-route bonus is `totalFloors` — always larger than the maximum distance-based score of `totalFloors - 1`. This ensures a compatible moving elevator always outscores any idle elevator, regardless of distance.

---

### ElevatorSystem.requestExternal()

**Core logic:**
1. Validate the floor is within bounds
2. Build the `ExternalRequest` value object
3. Delegate to the strategy for elevator selection
4. Add the stop to the selected elevator

**Edge cases:**
- Floor out of range (< 1 or > totalFloors) — throw before dispatching; the strategy should never receive an invalid request

```
requestExternal(floor, direction)
    if floor < 1 or floor > totalFloors
        throw Error("Invalid floor: " + floor)

    request  = ExternalRequest(floor, direction)
    elevator = dispatchStrategy.selectElevator(elevators, request)
    elevator.addStop(floor)
```

`requestInternal` bypasses the strategy entirely — it looks up the elevator by ID and calls `addStop` directly:

```
requestInternal(elevatorId, targetFloor)
    if targetFloor < 1 or targetFloor > totalFloors
        throw Error("Invalid floor: " + targetFloor)

    elevator = elevators.find(e -> e.getId() == elevatorId)
    if elevator == null
        throw Error("Unknown elevator: " + elevatorId)

    elevator.addStop(targetFloor)
```

---

## Verification

**Setup:** 10-floor building, 2 elevators, `NearestElevatorStrategy(totalFloors=10)`.

- Elevator A (id="A"): floor 1, IDLE
- Elevator B (id="B"): floor 8, IDLE

---

**External request: floor 5, direction UP**

```
requestExternal(5, UP)
  request = ExternalRequest(5, UP)

  selectElevator:
    Elevator A: IDLE, distance = |1 - 5| = 4, score = 10 - 4 = 6
    Elevator B: IDLE, distance = |8 - 5| = 3, score = 10 - 3 = 7
  → bestElevator = B  (score 7 > 6)

  B.addStop(5):
    5 < 8 (currentFloor) → downQueue.add(5)
    state was IDLE → state = MOVING_DOWN

State: A[floor=1, IDLE], B[floor=8, MOVING_DOWN, downQueue={5}]
```

---

**External request: floor 3, direction UP**

```
requestExternal(3, UP)
  request = ExternalRequest(3, UP)

  selectElevator:
    Elevator A: IDLE, distance = |1 - 3| = 2, score = 10 - 2 = 8
    Elevator B: MOVING_DOWN, request.floor=3 ≤ B.currentFloor=8 ✓
                but request.direction=UP ≠ DOWN → no en-route bonus → score = 0
  → bestElevator = A  (score 8 > 0)

  A.addStop(3):
    3 > 1 (currentFloor) → upQueue.add(3)
    state was IDLE → state = MOVING_UP

State: A[floor=1, MOVING_UP, upQueue={3}], B[floor=8, MOVING_DOWN, downQueue={5}]
```

---

**Internal request: elevator A, target floor 9**

```
requestInternal("A", 9)
  elevator = find id="A"  → Elevator A
  A.addStop(9):
    9 > 1 (currentFloor) → upQueue.add(9)
    state is already MOVING_UP → no state change

State: A[floor=1, MOVING_UP, upQueue={3, 9}], B[floor=8, MOVING_DOWN, downQueue={5}]
```

---

**Simulation: step() × 4**

```
step 1:
  A.step(): MOVING_UP → currentFloor=2. upQueue={3,9}. 2 not in queue → keep moving.
  B.step(): MOVING_DOWN → currentFloor=7. downQueue={5}. 7 not in queue → keep moving.

  A[floor=2, MOVING_UP, upQueue={3,9}], B[floor=7, MOVING_DOWN, downQueue={5}]

step 2:
  A.step(): currentFloor=3. upQueue.remove(3) → upQueue={9}. Not empty → keep MOVING_UP.
  B.step(): currentFloor=6. downQueue={5}. 6 not in queue → keep moving.

  A[floor=3 ← STOP, MOVING_UP, upQueue={9}], B[floor=6, MOVING_DOWN, downQueue={5}]

step 3:
  A.step(): currentFloor=4. upQueue={9}. No stop.
  B.step(): currentFloor=5. downQueue.remove(5) → downQueue={}.
            downQueue empty, upQueue also empty → state = IDLE.

  A[floor=4, MOVING_UP, upQueue={9}], B[floor=5 ← STOP, IDLE]

step 4:
  A.step(): currentFloor=5. upQueue={9}. No stop.
  B.step(): IDLE → no-op.

  A[floor=5, MOVING_UP, upQueue={9}], B[floor=5, IDLE]
```

Elevator A stops at floor 3 (picking up the UP passenger) then continues toward floor 9. Elevator B stops at floor 5 (the external UP request — mismatched direction but best available at dispatch time) and becomes idle. After 4 steps, A is at floor 5 en route to 9; B idles at 5.

---

**Edge case: floor out of range**

```
requestExternal(15, UP)
  15 > totalFloors (10)
  throw Error("Invalid floor: 15")   ✓
```

---

**Edge case: direction reversal**

```
// Elevator A is at floor 9 after servicing its internal stop.
// A new external request arrives at floor 2, direction DOWN.

A.addStop(2):
  2 < 9 (currentFloor) → downQueue.add(2)
  state is IDLE → state = MOVING_DOWN

A.step() × 7:
  currentFloor decrements 9 → 8 → 7 → 6 → 5 → 4 → 3 → 2
  At floor 2: downQueue.remove(2) → downQueue={}
              upQueue also empty → state = IDLE

A[floor=2 ← STOP, IDLE]   ✓
```

---

**Edge case: IDLE elevator receives step()**

```
B is IDLE at floor 5.
B.step()
  state == IDLE → return immediately.   ✓
```

---

## Extensibility

### 1. "How would you add a new dispatch algorithm — say, round-robin?"

> "Add one class: `RoundRobinStrategy implements DispatchStrategy`. It maintains an internal counter and cycles through elevators regardless of position. Implement `selectElevator()` to return `elevators[counter % size]` and increment. Pass `new RoundRobinStrategy()` to `ElevatorSystem` at construction. `ElevatorSystem`, `Elevator`, and `NearestElevatorStrategy` are completely untouched. The Strategy pattern makes this purely additive."

```
class RoundRobinStrategy implements DispatchStrategy:
    - counter: int = 0

    + selectElevator(elevators, request) -> Elevator
        elevator  = elevators[counter % elevators.size()]
        counter  += 1
        return elevator
```

---

### 2. "How would you add passenger capacity limits?"

Right now there is no concept of how many passengers a car holds. Dispatching to a full elevator wastes the assignment.

> "I'd add `currentLoad` and `capacity` fields to `Elevator`. The dispatch strategy filters out full elevators before scoring. If every elevator is at capacity, fall back to the one with the fewest remaining stops — the one nearest to freeing up. `addStop()` and `step()` are unchanged; capacity is a dispatch concern, not a movement concern."

```
class Elevator:
    - currentLoad: int
    - capacity:    int

    + isFull()        -> boolean
        return currentLoad >= capacity

    + board(count: int) -> void    // called when doors open at a floor
        currentLoad = min(currentLoad + count, capacity)

    + alight(count: int) -> void
        currentLoad = max(currentLoad - count, 0)

// In NearestElevatorStrategy.selectElevator():
    for elevator in elevators
        if elevator.isFull()
            continue               // skip — cannot accept more passengers
        // ... existing scoring logic unchanged
```

---

### 3. "How would you make this thread-safe for a real building?"

Right now `step()` runs in a single-threaded simulation loop. In a real gateway, requests arrive asynchronously and elevator movement runs on its own thread.

Two failure modes to protect against: concurrent `addStop()` calls from different request threads racing to modify `upQueue`/`downQueue`, and a movement thread reading `currentFloor` and `state` while a request thread writes them.

> "I'd synchronize at the `Elevator` level — one `ReentrantLock` per elevator instance. Both `addStop()` and `step()` acquire the lock before touching any mutable fields. Two different elevators never contend with each other. For `selectElevator()`, which only reads `currentFloor` and `state`, I'd declare those `volatile` so reads always see the latest written value without locking. Reads in the strategy are then lock-free while writes in `step()` and `addStop()` are always serialized."

```
class Elevator:
    - lock:         ReentrantLock = new ReentrantLock()
    - currentFloor: volatile int          // safe for lock-free reads
    - state:        volatile ElevatorState

    + addStop(floor)
        lock.lock()
        try
            // ... existing logic unchanged
        finally
            lock.unlock()

    + step()
        lock.lock()
        try
            // ... existing logic unchanged
        finally
            lock.unlock()

    + getCurrentFloor() -> int
        return currentFloor               // volatile read — no lock needed

    + getState() -> ElevatorState
        return state                      // volatile read — no lock needed
```

Per-elevator locks mean `addStop("A", 5)` and `addStop("B", 7)` run in parallel with zero contention. Only two concurrent calls targeting the exact same elevator need to serialize — which in a real building is the correct and expected behavior.
