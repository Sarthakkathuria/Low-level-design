# Meeting Room Booking System

A meeting room booking system lets users reserve conference rooms in a building by specifying a floor and a desired time window. The system selects an available room on that floor, rejects requests that would conflict with an existing booking, and returns a confirmation the user can later reference to cancel.

---

## Requirements

You walk into the interview and get greeted with this prompt:

> "Design a Meeting Room Booking System where users can request a meeting room by specifying the floor number and a desired time slot. The system should allocate an available room, prevent conflicting bookings, and support booking cancellation."

### Clarifying Questions

**You:** "When a user specifies a floor, do they choose a specific room or does the system pick any available room on that floor?"
**Interviewer:** "The system picks. The user only specifies the floor — the system decides which room."

**You:** "What defines a time slot — fixed hourly blocks like 9am, 10am, or arbitrary start and end times?"
**Interviewer:** "Arbitrary start and end times. Callers pass a start datetime and an end datetime."

**You:** "Do rooms have a capacity, and can users specify a minimum number of seats they need?"
**Interviewer:** "Yes — rooms have a capacity, and users can optionally specify a minimum. The system should only assign rooms that meet the requirement."

**You:** "Are back-to-back bookings allowed — for example, one booking ending at 11:00 and the next starting at 11:00 in the same room?"
**Interviewer:** "Yes, back-to-back is fine. Only true overlaps are conflicts."

**You:** "Who is allowed to cancel a booking — anyone, or only the person who made it?"
**Interviewer:** "Only the original organizer. Cancellation requires the booking ID and the user ID to match."

**You:** "What does a successful booking return to the caller?"
**Interviewer:** "A confirmation: booking ID, the assigned room, the user, and the time slot."

**You:** "Should the room selection algorithm be fixed, or should it be swappable — for example, switching from first-available to smallest-room-that-fits?"
**Interviewer:** "Keep it extensible. Use first-available as the default."

**You:** "Room amenities like projectors or whiteboards — are those part of the request?"
**Interviewer:** "Out of scope for core design, but keep it extensible."

**You:** "Notifications, recurring bookings, payment, or floor-access restrictions?"
**Interviewer:** "All out of scope."

### Final Requirements

| # | Requirement |
|---|-------------|
| 1 | System manages rooms distributed across floors; each room has an ID, floor number, and seating capacity |
| 2 | Users request a room by specifying floor, time slot (start + end datetime), and an optional minimum capacity |
| 3 | System allocates any available room on the requested floor that meets the minimum capacity |
| 4 | No two bookings for the same room may have overlapping time slots; back-to-back slots sharing only an endpoint are allowed |
| 5 | A successful booking returns a confirmation: booking ID, assigned room ID, user ID, and time slot |
| 6 | Users may cancel their own bookings by providing a booking ID; only the original organizer may cancel |
| 7 | The room selection algorithm is configurable; the default is first-available |

**Out of scope:** room amenities/features, recurring bookings, notifications, payment, multi-tenant authorization, UI.

---

## Core Entities and Relationships

The central challenge here is **conflict detection and room selection without a monolithic booking manager**. A naive approach would put all logic in one `BookingSystem` class: loop over every booking in the entire system, check for overlaps, pick a room based on hard-coded rules. That class couples conflict detection to allocation logic, makes both untestable in isolation, and closes the door on swapping the selection algorithm.

Two principles solve this cleanly:

**Information Expert** for conflict detection: a room owns its own booking list, so the room itself is the right place to answer "am I available for this slot?" The `Room.isAvailable()` method iterates its own bookings and delegates the overlap predicate to `TimeSlot.overlaps()`. No external class reaches into a room's bookings to check availability — it just asks.

**Strategy pattern** for room selection: given a set of candidate rooms that all satisfy the floor and capacity requirement, choosing which one to assign is an independent, swappable algorithm. The `RoomSelectionStrategy` interface isolates that decision. Switching from first-available to smallest-fit or round-robin means writing one new class — nothing else changes.

**BookingSystem** — The orchestrator and public entry point. Receives booking and cancellation requests, filters rooms by floor and capacity, delegates the selection decision to the strategy, creates `Booking` objects, and maintains a flat index for O(1) cancellation lookups.

**Room** — A single conference room. Owns its booking list and exposes `isAvailable(slot)` — the only place overlap checking occurs. Callers never inspect a room's booking list directly; they ask the room.

**Booking** — An immutable value object representing a confirmed reservation. Carries a generated ID, the assigned room ID, the organizer's user ID, and the time slot. It is the permanent receipt returned to the caller.

**TimeSlot** — A value object: start datetime + end datetime. Encapsulates the overlap predicate as `overlaps(other)`. This is the mathematical heart of the system — every conflict check flows through it. Making it a value object ensures the predicate is tested independently of any room or booking logic.

**BookingRequest** — A value object bundling the caller's intent: floor, user ID, time slot, and minimum capacity. Created inside `bookRoom()` and passed to the strategy so the strategy has full context without taking four separate parameters.

**RoomSelectionStrategy** — The strategy interface. Receives a pre-filtered list of candidate rooms (already passing floor and capacity checks) and a `BookingRequest`, and returns the room to assign, or `null` if none is available.

**FirstAvailableStrategy** — Concrete strategy. Iterates candidates in order and returns the first room for which `isAvailable(slot)` is true. Simple and predictable.

### Entity Responsibilities

| Entity | Responsibility |
|--------|----------------|
| `BookingSystem` | Entry point. Filters rooms by floor and capacity, delegates selection, creates and stores bookings, handles cancellation authorization. |
| `Room` | Owns its booking list. Answers availability queries via `isAvailable()`. Adds and removes bookings from its own list. |
| `Booking` | Immutable value object. Confirmed reservation: ID, room ID, user ID, time slot, creation timestamp. |
| `TimeSlot` | Value object. Carries start and end datetimes. Owns the overlap predicate `overlaps(other)`. |
| `BookingRequest` | Value object. Bundles caller intent: floor, user ID, time slot, minimum capacity. Passed to the strategy. |
| `RoomSelectionStrategy` | Interface. Selects one room from a pre-filtered candidate list. No mutation — selection only. |
| `FirstAvailableStrategy` | Concrete strategy. Returns the first candidate for which `isAvailable(slot)` is true. |

### Relationships

```
BookingSystem
  ├── rooms:        Map<roomId, Room>
  ├── bookings:     Map<bookingId, Booking>
  └── roomSelector: RoomSelectionStrategy
                           │
                           │ selects from
                           ▼
           ┌──────────────────────────────┐
           │             Room             │
           ├──────────────────────────────┤
           │ id:       String             │
           │ floor:    int                │
           │ capacity: int                │
           │ bookings: List<Booking>      │ ← owns; checked by isAvailable()
           └──────────────────────────────┘

<<interface>> RoomSelectionStrategy
  + selectRoom(candidates: List<Room>, request: BookingRequest) -> Room?
          ^
          │
FirstAvailableStrategy

TimeSlot (value object)              BookingRequest (value object)
  ├── start: LocalDateTime             ├── userId:      String
  └── end:   LocalDateTime             ├── floor:       int
  + overlaps(other) -> boolean         ├── timeSlot:    TimeSlot
                                       └── minCapacity: int

Booking (value object)
  ├── id:        String
  ├── roomId:    String
  ├── userId:    String
  ├── timeSlot:  TimeSlot
  └── createdAt: DateTime
```

`BookingSystem` maintains two structures: `rooms` (keyed by room ID) for filtering and selection, and `bookings` (keyed by booking ID) for O(1) cancellation lookups. When a booking is cancelled, `BookingSystem` removes it from both the flat index and from the owning room's internal list.

---

## Class Design

Top-down: `BookingSystem`, then `Room`, then `Booking`, `TimeSlot`, `BookingRequest`, `RoomSelectionStrategy`, `FirstAvailableStrategy`.

### BookingSystem

**Deriving State**

| Requirement | What BookingSystem must track |
|-------------|-------------------------------|
| "Rooms distributed across floors" | All rooms, indexed by room ID for direct lookup |
| "Cancel by booking ID" | All active bookings, indexed by booking ID for O(1) retrieval |
| "Configurable selection algorithm" | The active `RoomSelectionStrategy` |

**Deriving Behavior**

| Need from requirements | Method on BookingSystem |
|------------------------|-------------------------|
| "Request a room on a floor for a time slot" | `bookRoom(userId, floor, timeSlot, minCapacity) -> Booking` |
| "Cancel a booking" | `cancelBooking(bookingId, userId) -> void` |
| "Inspect existing bookings for a room" | `getBookingsForRoom(roomId) -> List<Booking>` |

```
class BookingSystem:
    - rooms:        Map<String, Room>
    - bookings:     Map<String, Booking>
    - roomSelector: RoomSelectionStrategy

    + BookingSystem(rooms: List<Room>, selector: RoomSelectionStrategy)
    + bookRoom(userId: String, floor: int, timeSlot: TimeSlot,
               minCapacity: int) -> Booking
    + cancelBooking(bookingId: String, userId: String) -> void
    + getBookingsForRoom(roomId: String) -> List<Booking>
```

---

### Room

**Deriving State**

| Requirement | What Room must track |
|-------------|----------------------|
| "Rooms have a floor and capacity" | `floor: int`, `capacity: int` — fixed at construction |
| "Check if room is available for a slot" | `bookings: List<Booking>` — the live booking list the room checks against |

The booking list is mutable state owned by the room. No external class is allowed to inspect or modify it directly — all access goes through `isAvailable()`, `addBooking()`, and `removeBooking()`.

**Deriving Behavior**

| Need | Method on Room |
|------|----------------|
| "Is this room free for this slot?" | `isAvailable(slot: TimeSlot) -> boolean` |
| "Record a confirmed booking" | `addBooking(booking: Booking) -> void` |
| "Remove a cancelled booking" | `removeBooking(bookingId: String) -> void` |

```
class Room:
    - id:       String
    - floor:    int
    - capacity: int
    - bookings: List<Booking>

    + Room(id: String, floor: int, capacity: int)
    + isAvailable(slot: TimeSlot) -> boolean
    + addBooking(booking: Booking) -> void
    + removeBooking(bookingId: String) -> void
    + getId()       -> String
    + getFloor()    -> int
    + getCapacity() -> int
    + getBookings() -> List<Booking>     // read-only copy for external inspection
```

---

### Booking

An immutable value object. Set at construction and never modified. It is the receipt the caller holds.

```
class Booking:
    - id:        String
    - roomId:    String
    - userId:    String
    - timeSlot:  TimeSlot
    - createdAt: DateTime

    + Booking(id, roomId, userId, timeSlot, createdAt)
    + getId()        -> String
    + getRoomId()    -> String
    + getUserId()    -> String
    + getTimeSlot()  -> TimeSlot
    + getCreatedAt() -> DateTime
```

---

### TimeSlot

The overlap predicate is the mathematical core of the system. Two intervals `[A.start, A.end)` and `[B.start, B.end)` overlap if and only if `A.start < B.end AND B.start < A.end`. Strict inequalities mean back-to-back slots sharing only an endpoint do not overlap.

```
class TimeSlot:
    - start: LocalDateTime
    - end:   LocalDateTime

    + TimeSlot(start, end)
    + overlaps(other: TimeSlot) -> boolean
    + getStart() -> LocalDateTime
    + getEnd()   -> LocalDateTime
```

---

### RoomSelectionStrategy (interface)

Receives a pre-filtered candidate list — rooms already on the right floor and above the capacity threshold — and returns the room to assign, or `null` if none is available for the time slot.

```
interface RoomSelectionStrategy:
    + selectRoom(candidates: List<Room>, request: BookingRequest) -> Room?
```

The floor and capacity filtering happens in `BookingSystem` before the strategy is called. The strategy only decides among valid candidates — it never needs to re-check floor or capacity.

---

### FirstAvailableStrategy implements RoomSelectionStrategy

```
class FirstAvailableStrategy implements RoomSelectionStrategy:
    + selectRoom(candidates, request) -> Room?
        for room in candidates
            if room.isAvailable(request.getTimeSlot())
                return room
        return null
```

---

### BookingRequest (value object)

```
class BookingRequest:
    - userId:      String
    - floor:       int
    - timeSlot:    TimeSlot
    - minCapacity: int

    + BookingRequest(userId, floor, timeSlot, minCapacity)
    + getUserId()      -> String
    + getFloor()       -> int
    + getTimeSlot()    -> TimeSlot
    + getMinCapacity() -> int
```

---

### Final Class Design

```
+---------------------------------------------------+
|                  BookingSystem                    |
+---------------------------------------------------+
| - rooms:        Map<String, Room>                 |
| - bookings:     Map<String, Booking>              |
| - roomSelector: RoomSelectionStrategy             |
+---------------------------------------------------+
| + bookRoom(userId, floor, timeSlot, minCap)       |
|     -> Booking                                    |
| + cancelBooking(bookingId, userId) -> void        |
| + getBookingsForRoom(roomId) -> List<Booking>     |
+---------------------------------------------------+
         |                         |
   owns  | Map<roomId, Room>       | uses
         v                         v
+---------------------------+   +--------------------------------+
|          Room             |   |        <<interface>>           |
+---------------------------+   |     RoomSelectionStrategy      |
| - id:       String        |   +--------------------------------+
| - floor:    int           |   | + selectRoom(                  |
| - capacity: int           |   |     candidates, request)       |
| - bookings: List<Booking> |   |     -> Room?                   |
+---------------------------+   +--------------------------------+
| + isAvailable(slot)       |                  ^
| + addBooking(booking)     |                  |
| + removeBooking(id)       |   +------------------------------+
+---------------------------+   |    FirstAvailableStrategy    |
                                +------------------------------+
                                | + selectRoom(...) -> Room?   |
                                +------------------------------+

+---------------------------+   +---------------------------+
|         Booking           |   |        TimeSlot           |
+---------------------------+   +---------------------------+
| - id:        String       |   | - start: LocalDateTime    |
| - roomId:    String       |   | - end:   LocalDateTime    |
| - userId:    String       |   +---------------------------+
| - timeSlot:  TimeSlot     |   | + overlaps(other)         |
| - createdAt: DateTime     |   |     -> boolean            |
+---------------------------+   +---------------------------+

+---------------------------+
|      BookingRequest       |
+---------------------------+
| - userId:      String     |
| - floor:       int        |
| - timeSlot:    TimeSlot   |
| - minCapacity: int        |
+---------------------------+
```

---

## Implementation

The three most interesting pieces are `BookingSystem.bookRoom()` (orchestration: filtering, strategy delegation, booking creation), `TimeSlot.overlaps()` (the conflict predicate that is the mathematical foundation of the system), and `BookingSystem.cancelBooking()` (authorization check and two-structure cleanup). We'll also walk through `Room.isAvailable()` since it ties the overlap check to a room's live booking history.

### BookingSystem.bookRoom()

**Core logic:**
1. Validate the time slot (start must precede end)
2. Filter all rooms to those on the requested floor with capacity at or above the minimum
3. If no rooms pass the filter, fail before calling the strategy
4. Build a `BookingRequest` and delegate to the strategy — let it pick the room
5. If the strategy returns `null`, no room is available for that slot
6. Generate a booking ID, create the `Booking`, record it in both the flat index and the room's own list
7. Return the booking

**Edge cases:**
- `timeSlot.start >= timeSlot.end` — throw before any work; an invalid slot cannot be booked
- No rooms on the requested floor — throw with a clear message; the strategy should never see an empty candidate list for a caller error
- No rooms meet the minimum capacity on that floor — same; fail early
- All candidate rooms have conflicts — the strategy returns `null`; throw "No available room on floor X for the requested time slot"

```
bookRoom(userId, floor, timeSlot, minCapacity)
    if timeSlot.getStart() >= timeSlot.getEnd()
        throw Error("Time slot start must be before end")

    candidates = rooms.values()
                      .filter(r -> r.getFloor() == floor)
                      .filter(r -> r.getCapacity() >= minCapacity)

    if candidates.isEmpty()
        throw Error("No rooms on floor " + floor
                    + " with capacity >= " + minCapacity)

    request  = BookingRequest(userId, floor, timeSlot, minCapacity)
    selected = roomSelector.selectRoom(candidates, request)

    if selected == null
        throw Error("No available room on floor " + floor
                    + " for the requested time slot")

    bookingId = generateId()                     // e.g. "BK-" + UUID
    booking   = Booking(bookingId, selected.getId(), userId, timeSlot, now())

    bookings[bookingId] = booking                // flat index for cancellation
    selected.addBooking(booking)                 // room's own list for availability checks

    return booking
```

---

### TimeSlot.overlaps()

Two half-open intervals `[A.start, A.end)` and `[B.start, B.end)` overlap if and only if each starts before the other ends. The strict inequalities are deliberate — they make back-to-back slots legal by definition.

**Core logic:**
- Return `true` if `this.start < other.end AND other.start < this.end`
- Return `false` otherwise — slots are adjacent or disjoint

**Edge cases:**
- Back-to-back (`A.end == B.start`): `A.start < B.end` ✓ but `B.start < A.end` becomes `A.end < A.end` → `false`. Correctly treated as non-overlapping.
- Contained slot (`A fully inside B`): both conditions are true → `true`. Correctly detected.
- Identical slots: `A.start < A.end` ✓ and `A.start < A.end` ✓ → `true`. Correctly detected.

```
overlaps(other)
    return this.start < other.end
       AND other.start < this.end
```

Three characters prevent double-booking. Every conflict check in the system reduces to this predicate.

---

### Room.isAvailable()

**Core logic:**
1. Iterate the room's booking list
2. For each existing booking, check whether its time slot overlaps the requested slot
3. Return `false` on the first conflict found; `true` if none found

**Edge cases:**
- Empty booking list — loop body never executes; returns `true` immediately
- Large booking list — O(n) scan per check. For a room that is heavily booked, this is bounded by the number of bookings per room, not the total bookings in the system

```
isAvailable(slot)
    for booking in bookings
        if booking.getTimeSlot().overlaps(slot)
            return false
    return true
```

This method is called by `FirstAvailableStrategy` (and any future strategy). The room's encapsulation ensures that overlap logic is never duplicated or re-derived elsewhere.

---

### BookingSystem.cancelBooking()

**Core logic:**
1. Look up the booking in the flat index by booking ID
2. Verify the requesting user ID matches the organizer
3. Retrieve the owning room and remove the booking from its list
4. Remove the booking from the flat index

**Edge cases:**
- Booking ID not found — throw before any authorization check; no information about other users' bookings is leaked
- User ID does not match organizer — throw; the booking exists but the caller is not authorized
- Room for the booking cannot be found — this is a data integrity bug; throw an internal error

```
cancelBooking(bookingId, userId)
    booking = bookings[bookingId]
    if booking == null
        throw Error("Booking not found: " + bookingId)

    if booking.getUserId() != userId
        throw Error("Only the organizer may cancel this booking")

    room = rooms[booking.getRoomId()]
    if room == null
        throw Error("Data integrity error: room not found for booking " + bookingId)

    room.removeBooking(bookingId)
    bookings.remove(bookingId)
```

The order matters: authorization check comes before mutation. If the user ID check came after `room.removeBooking()`, a failed authorization check would leave the system in a partially modified state.

---

## Verification

**Setup:** `FirstAvailableStrategy`. Two rooms on floor 2:

- Room-2A: floor=2, capacity=4, bookings=[]
- Room-2B: floor=2, capacity=8, bookings=[]

---

**Booking 1: alice requests floor 2, 10:00–11:00, minCapacity=4**

```
bookRoom("alice", 2, TimeSlot(10:00, 11:00), 4)

  candidates = [Room-2A (cap 4 ≥ 4), Room-2B (cap 8 ≥ 4)]

  selectRoom:
    Room-2A: isAvailable(10:00–11:00)?
      bookings=[] → true  ← return Room-2A immediately

  booking = Booking("BK-001", "Room-2A", "alice", 10:00–11:00, now())
  bookings["BK-001"] = booking
  Room-2A.bookings = [BK-001]

  return BK-001

State: Room-2A[bookings={10:00–11:00 alice}], Room-2B[bookings={}]
```

---

**Booking 2: bob requests floor 2, 10:30–11:30, minCapacity=4**

```
bookRoom("bob", 2, TimeSlot(10:30, 11:30), 4)

  candidates = [Room-2A, Room-2B]

  selectRoom:
    Room-2A: isAvailable(10:30–11:30)?
      Check BK-001 slot (10:00–11:00) vs (10:30–11:30):
        10:00 < 11:30 ✓ AND 10:30 < 11:00 ✓  → overlaps → false
    Room-2B: isAvailable(10:30–11:30)?
      bookings=[] → true  ← return Room-2B

  booking = Booking("BK-002", "Room-2B", "bob", 10:30–11:30, now())
  Room-2B.bookings = [BK-002]

  return BK-002

State: Room-2A[{10:00–11:00 alice}], Room-2B[{10:30–11:30 bob}]
```

---

**Booking 3: carol requests floor 2, 10:00–11:00, minCapacity=4**

```
bookRoom("carol", 2, TimeSlot(10:00, 11:00), 4)

  candidates = [Room-2A, Room-2B]

  selectRoom:
    Room-2A: isAvailable(10:00–11:00)?
      BK-001 (10:00–11:00): 10:00 < 11:00 ✓ AND 10:00 < 11:00 ✓ → overlaps → false
    Room-2B: isAvailable(10:00–11:00)?
      BK-002 (10:30–11:30): 10:00 < 11:30 ✓ AND 10:30 < 11:00 ✓ → overlaps → false
    selectRoom returns null

  throw Error("No available room on floor 2 for the requested time slot")   ✓
```

---

**Cancel booking BK-001 (alice)**

```
cancelBooking("BK-001", "alice")

  booking = bookings["BK-001"]  → found
  booking.getUserId() == "alice" ✓

  room = rooms["Room-2A"]
  Room-2A.removeBooking("BK-001") → Room-2A.bookings = []
  bookings.remove("BK-001")

State: Room-2A[bookings={}], Room-2B[{10:30–11:30 bob}]
```

---

**Booking 4: dave requests floor 2, 10:00–11:00, minCapacity=4**

```
bookRoom("dave", 2, TimeSlot(10:00, 11:00), 4)

  selectRoom:
    Room-2A: isAvailable(10:00–11:00)?
      bookings=[] (alice's booking was cancelled) → true  ← return Room-2A

  booking = Booking("BK-003", "Room-2A", "dave", 10:00–11:00, now())
  return BK-003   ✓

State: Room-2A[{10:00–11:00 dave}], Room-2B[{10:30–11:30 bob}]
```

---

**Edge case: back-to-back booking — eve requests floor 2, 11:00–12:00, minCapacity=4**

```
bookRoom("eve", 2, TimeSlot(11:00, 12:00), 4)

  selectRoom:
    Room-2A: isAvailable(11:00–12:00)?
      BK-003 slot (10:00–11:00) vs (11:00–12:00):
        10:00 < 12:00 ✓ AND 11:00 < 11:00 ✗ (strict) → no overlap → continue
      bookings exhausted → true  ← return Room-2A

  booking = Booking("BK-004", "Room-2A", "eve", 11:00–12:00, now())
  return BK-004   ✓

Room-2A now holds [10:00–11:00 dave, 11:00–12:00 eve] — back-to-back with no gap, both valid.
```

---

**Edge case: unauthorized cancellation**

```
cancelBooking("BK-002", "alice")
  booking = bookings["BK-002"]  → found (bob's booking)
  booking.getUserId() == "bob" ≠ "alice"
  throw Error("Only the organizer may cancel this booking")   ✓
```

---

**Edge case: invalid time slot**

```
bookRoom("frank", 2, TimeSlot(11:00, 10:00), 4)
  timeSlot.start (11:00) >= timeSlot.end (10:00)
  throw Error("Time slot start must be before end")   ✓
```

---

## Extensibility

### 1. "How would you add a new selection algorithm — say, smallest-room-that-fits?"

Right now `FirstAvailableStrategy` returns the first available room regardless of wasted capacity. Assigning a 20-person room to a 2-person meeting is wasteful.

> "Add one class: `SmallestFitStrategy implements RoomSelectionStrategy`. It filters candidates to those available for the slot, then sorts by capacity ascending and returns the first. The strategy receives a pre-filtered candidate list whose capacity already meets the minimum — so 'smallest that fits' just means the one with the lowest capacity among the available ones. Pass `new SmallestFitStrategy()` to `BookingSystem` at construction. `BookingSystem`, `Room`, `Booking`, and `FirstAvailableStrategy` are completely untouched."

```
class SmallestFitStrategy implements RoomSelectionStrategy:
    + selectRoom(candidates, request) -> Room?
        available = candidates
                      .filter(r -> r.isAvailable(request.getTimeSlot()))
                      .sortBy(r -> r.getCapacity())     // ascending

        return available.isEmpty() ? null : available.first()
```

---

### 2. "How would you add amenity-based filtering — projectors, whiteboards?"

Right now rooms are matched only by floor and capacity. Some meetings require a projector or video conferencing equipment.

> "I'd add an `amenities: Set<String>` field to `Room` and a `requiredAmenities: Set<String>` field to `BookingRequest`. The filtering step in `bookRoom()` gains one more condition before calling the strategy. No strategy class changes — the amenity filter happens before candidates reach the strategy, so every existing and future strategy automatically benefits. Adding a new amenity type means adding a string constant — no class changes at all."

```
// Room gains one field:
class Room:
    - amenities: Set<String>    // e.g., {"PROJECTOR", "WHITEBOARD"}

    + hasAmenities(required: Set<String>) -> boolean
        return amenities.containsAll(required)

// BookingRequest gains one field:
class BookingRequest:
    - requiredAmenities: Set<String>    // empty set if none required

// bookRoom() gains one filter line:
    candidates = rooms.values()
                      .filter(r -> r.getFloor() == floor)
                      .filter(r -> r.getCapacity() >= minCapacity)
                      .filter(r -> r.hasAmenities(request.getRequiredAmenities()))
```

---

### 3. "How would you support recurring bookings — for example, every Monday 10:00–11:00?"

Right now each booking is a single time slot. Recurring meetings would create many individual bookings, one per occurrence.

> "I'd introduce a `RecurringBookingRequest` that carries a recurrence rule (start date, end date, day-of-week, time slot) and expand it into a list of individual `TimeSlot` objects before calling `bookRoom()`. Expansion is the only new logic — it converts the rule into concrete slots. `bookRoom()` is called once per slot in a loop; if any slot has no available room, the whole recurring request is rejected and all previously created bookings for that series are rolled back. I'd link each booking to a `seriesId` so the entire series can be cancelled with one call."

```
class RecurringBookingRequest:
    - userId:      String
    - floor:       int
    - minCapacity: int
    - startDate:   LocalDate
    - endDate:     LocalDate
    - dayOfWeek:   DayOfWeek
    - slotTime:    TimeSlot             // time-of-day template, e.g. 10:00–11:00

    + expand() -> List<TimeSlot>        // materializes one slot per matching date

// In BookingSystem:
bookRecurringSeries(request: RecurringBookingRequest) -> List<Booking>
    slots    = request.expand()
    created  = []

    for slot in slots
        try
            booking = bookRoom(request.getUserId(), request.getFloor(),
                               slot, request.getMinCapacity())
            created.add(booking)
        catch NoRoomAvailableException
            // rollback all bookings created so far
            for b in created
                cancelBooking(b.getId(), request.getUserId())
            throw Error("No consistent room available for all occurrences")

    seriesId = generateId()
    created.forEach(b -> b.setSeriesId(seriesId))   // link for bulk cancellation
    return created
```
