# Parking Lot System

A parking lot system manages vehicle parking across multiple spots. When a vehicle enters, the system assigns an available compatible spot and issues a ticket. When the vehicle exits, the system calculates the fee based on time parked and frees the spot for the next customer.

---

## Requirements

You walk into the interview and get greeted with this prompt:

> "Design a parking lot system where vehicles are assigned to spots as they enter, and fees are calculated when they exit."

That's a start, but not enough detail for us to begin coding. Before touching a whiteboard, spend 3–5 minutes asking questions to nail down exactly what you're building.

### Clarifying Questions

Structure your questions around four areas: what are the core operations, what can go wrong, what's the scope boundary, and what might we need to extend later?

**You:** "What types of vehicles does the system support? And do different vehicle types need different spot sizes?"
**Interviewer:** "Three types: Motorcycle, Car, and Large Vehicle. Each type gets its own matching spot size. A car can't park in a motorcycle spot and vice versa."

Good. We know spot assignment is type-to-type — no cross-size fallback for now.

**You:** "How does pricing work? Is it flat rate, hourly, or different per vehicle type?"
**Interviewer:** "Hourly, same rate for all vehicles. Round up any partial hour."

Clean. One rate, ceil-based calculation.

**You:** "When a vehicle exits, how does the system identify the parking session? Does the driver hand over a ticket?"
**Interviewer:** "Yes. The system issues a ticket on entry. The driver presents the ticket ID on exit."

So there's a ticket lifecycle: issued on entry, consumed on exit.

**You:** "What happens if someone tries to exit with a fake or already-used ticket?"
**Interviewer:** "Reject it with a clear error. Used tickets should not work again."

**You:** "What if the lot is completely full for a given vehicle type when someone tries to enter?"
**Interviewer:** "Reject entry. Return an error. No waitlist or queueing."

**You:** "Are we handling payment processing? Or just calculating the fee and returning it?"
**Interviewer:** "Just calculate and return the fee. Payment is out of scope."

**You:** "Any physical hardware concerns — gates, displays, sensors?"
**Interviewer:** "No. Focus on the core logic only."

### Final Requirements

| # | Requirement |
|---|-------------|
| 1 | System supports three vehicle types: Motorcycle, Car, Large Vehicle |
| 2 | When a vehicle enters, system automatically assigns an available compatible spot |
| 3 | System issues a ticket at entry |
| 4 | When a vehicle exits, user provides ticket ID — system validates it, calculates fee (hourly, rounded up), and frees the spot |
| 5 | Pricing is hourly with the same rate for all vehicles |
| 6 | System rejects entry if no compatible spot is available |
| 7 | System rejects exit if ticket is invalid or already used |

**Out of scope:** payment processing, physical gate hardware, security cameras, UI/display systems, reservations or pre-booking.

---

## Core Entities and Relationships

Scan the requirements for nouns and decide which ones carry state or behavior versus which are just data.

**Vehicle** — At first glance, vehicles seem important. But what does our system actually do with a vehicle? It checks the type to find a matching spot. That's it. The system doesn't track individual vehicles, license plates, or any vehicle-specific history. Vehicle type is just an input parameter. This stays as an enum, not a class.

**Spot** — A physical parking space. It has a type and tracks whether it is currently occupied. If something maintains changing state, it deserves to be its own entity. Spot clearly qualifies.

**Ticket** — When a vehicle enters, we issue a ticket that ties together the entry time and the assigned spot. When exit happens, that same ticket is used to compute the fee. Ticket is more than a string ID — it owns the entry timestamp and the spot reference. Clear entity.

**ParkingLot** — Something needs to receive entry/exit requests, find available spots, issue tickets, and track active sessions. That's the orchestrator. Clear entity.

### Entity Responsibilities

| Entity | Responsibility |
|--------|---------------|
| `ParkingLot` | The orchestrator. Holds all spots and the active ticket map. Handles entry and exit operations. Computes parking fees. |
| `Ticket` | Issued at entry. Holds the spot reference and entry time. Tracks whether it has been consumed. |
| `Spot` | A physical parking space. Knows its vehicle type and manages its own occupancy state. |

### Relationships

```
ParkingLot ──────────────> Spot[]
     │                      (has many)
     │
     └──────────────> Map<id, Ticket>
                            │
                       Ticket ──────> Spot
                       (1:1, the spot assigned at entry)
```

---

## Class Design

We'll design top-down, starting with the orchestrator.

### ParkingLot

ParkingLot is the system's public API. External code calls `enter()` when a vehicle arrives and `exit()` when it leaves.

**Deriving State**

| Requirement | What ParkingLot must track |
|-------------|---------------------------|
| "System automatically assigns an available compatible spot" | All spots in the lot |
| "When a vehicle exits, user provides ticket ID" | A map from ticket ID → Ticket for O(1) lookup |
| "Pricing is hourly with same rate for all vehicles" | The hourly rate (a system-level constant) |

We store all spots in a flat `List<Spot>` rather than a `Map<VehicleType, List<Spot>>`. The flat list is simpler and the O(n) scan is fine for a realistic lot size. No premature optimization.

**Deriving Behavior**

| Need from requirements | Method on ParkingLot |
|------------------------|----------------------|
| "Vehicle enters, system assigns spot and issues ticket" | `enter(vehicleType) -> Ticket` |
| "Vehicle exits, fee calculated, spot freed" | `exit(ticketId) -> double` |

```
class ParkingLot:
    - spots: List<Spot>
    - tickets: Map<String, Ticket>
    - HOURLY_RATE: double

    + ParkingLot(spots, hourlyRate)
    + enter(vehicleType) -> Ticket
    + exit(ticketId) -> double
```

`enter()` returns the full Ticket object so the caller has the ticket ID to hand to the driver. `exit()` returns the calculated fee so the caller can pass it downstream to a payment system.

---

### Ticket

Ticket represents an active parking session.

**Deriving State**

| Requirement | What Ticket must track |
|-------------|------------------------|
| "System issues a ticket at entry" | A unique ticket ID |
| "System assigns an available compatible spot" | A reference to the assigned Spot |
| "Calculates fee based on time spent" | Entry timestamp |
| "System rejects exit if ticket is already used" | Whether the ticket has been consumed |

**Deriving Behavior**

| Need | Method on Ticket |
|------|-----------------|
| "Validate the ticket, read entry time for fee" | `getId()`, `getSpot()`, `getEntryTime()` |
| "Reject reuse on second exit attempt" | `isUsed()`, `markUsed()` |

```
class Ticket:
    - id: String
    - spot: Spot
    - entryTime: timestamp
    - used: boolean

    + Ticket(id, spot, entryTime)
    + getId() -> String
    + getSpot() -> Spot
    + getEntryTime() -> timestamp
    + isUsed() -> boolean
    + markUsed() -> void
```

Fee calculation lives in `ParkingLot.exit()`, not on Ticket. The hourly rate is a system-level concern owned by ParkingLot. Ticket owns the timestamps, not the pricing policy. Putting `calculateFee` on Ticket would couple it to a rate it shouldn't know about.

---

### Spot

Spot is the simplest entity. It represents one physical parking space.

**Deriving State**

| Requirement | What Spot must track |
|-------------|----------------------|
| "System supports Motorcycle, Car, Large Vehicle spots" | The type of vehicle it accepts |
| "System assigns an available compatible spot" | Whether it is currently occupied |

**Deriving Behavior**

| Need | Method on Spot |
|------|---------------|
| "Find available spot by type" | `getType()`, `isOccupied()` |
| "Assign spot on entry, free spot on exit" | `markOccupied()`, `markFree()` |

```
class Spot:
    - id: String
    - type: VehicleType
    - occupied: boolean

    + Spot(id, type)
    + getId() -> String
    + getType() -> VehicleType
    + isOccupied() -> boolean
    + markOccupied() -> void
    + markFree() -> void
```

Spot tracks its own physical state. Occupancy is intrinsic to the physical slot — it describes the slot's condition, not a system-managed relationship. If we stored occupancy in an external set on ParkingLot, we'd have to keep two things in sync. Keeping it on Spot eliminates that.

---

### Final Class Design

```
+----------------------+          +----------------------+          +--------------------+
|     ParkingLot       |          |        Ticket        |          |        Spot        |
+----------------------+          +----------------------+          +--------------------+
| - spots: List<Spot>  | 1      * | - id: String         | *      1 | - id: String       |
| - tickets: Map<>     |--------> | - spot: Spot         |--------> | - type: VehicleType|
| - HOURLY_RATE: double|          | - entryTime: timestamp|         | - occupied: boolean|
+----------------------+          | - used: boolean      |          +--------------------+
| + enter(vehicleType) |          +----------------------+          | + getType()        |
| + exit(ticketId)     |          | + getId()            |          | + isOccupied()     |
+----------------------+          | + getSpot()          |          | + markOccupied()   |
                                  | + getEntryTime()     |          | + markFree()       |
                                  | + isUsed()           |          +--------------------+
                                  | + markUsed()         |
                                  +----------------------+

enum VehicleType:  MOTORCYCLE | CAR | LARGE
```

---

## Implementation

Most interviewers are fine with pseudocode — confirm before diving in. The two most interesting methods are `enter()` and `exit()`.

### enter()

**Core logic:**
1. Find an available spot matching the vehicle type
2. Mark it occupied
3. Generate a ticket and store it in the map
4. Return the ticket

**Edge cases:**
- No compatible spot available → throw error before touching any state
- Invalid vehicle type → caught implicitly when the scan finds no match

```
enter(vehicleType)
    spot = findAvailableSpot(vehicleType)
    if spot == null
        throw Error("No available spot for " + vehicleType)

    spot.markOccupied()
    id = generateTicketId()
    ticket = Ticket(id, spot, now())
    tickets[id] = ticket

    return ticket

findAvailableSpot(vehicleType)
    for spot in spots
        if spot.getType() == vehicleType && !spot.isOccupied()
            return spot
    return null
```

We mark the spot occupied before creating the ticket. If ticket generation fails, the worst case is a spot stuck occupied for this session — easy to diagnose. The reverse ordering (generate ticket first, then mark occupied) risks issuing a ticket for a spot that never got marked, letting a second vehicle claim the same spot.

---

### exit()

**Core logic:**
1. Look up ticket by ID
2. Validate it exists and hasn't been used
3. Calculate fee: ceil(elapsed hours) × HOURLY_RATE
4. Free the spot and mark ticket used
5. Return fee

**Edge cases:**
- Null or empty ticket ID → reject immediately
- Ticket not found in map → invalid ticket
- Ticket already used → duplicate exit attempt

```
exit(ticketId)
    if ticketId == null || ticketId.isEmpty()
        throw Error("Invalid ticket ID")

    ticket = tickets[ticketId]
    if ticket == null
        throw Error("Ticket not found")

    if ticket.isUsed()
        throw Error("Ticket already used")

    hoursParked = ceil((now() - ticket.getEntryTime()) / 1 hour)
    fee = hoursParked * HOURLY_RATE

    ticket.getSpot().markFree()
    ticket.markUsed()

    return fee
```

Both cleanup steps — `markFree()` and `markUsed()` — are required. Forgetting `markUsed()` allows the same ticket to be reused indefinitely. Forgetting `markFree()` permanently locks the spot even after the car leaves.

We do **not** remove the ticket from the map after exit. The ticket stays as a historical record. The `isUsed()` flag prevents reuse. If an interviewer asks about memory growth from accumulating old tickets, the right answer is a background cleanup job or an archival process — not designing it upfront when the requirements don't ask for it.

---

## Design Patterns

### Information Expert
Each class owns the data it needs and the logic that uses it.
- `Spot` manages its own occupancy — it knows whether it is physically occupied.
- `Ticket` exposes its entry time — only Ticket knows when the session started.
- `ParkingLot` computes the fee — only ParkingLot knows the hourly rate.

This keeps methods short and makes bugs easy to localize. When something goes wrong with spot availability, you check `Spot`. When fee calculation is wrong, you check `ParkingLot.exit()`.

### Tell, Don't Ask
`ParkingLot` tells `Spot` to mark itself occupied rather than reading the occupied flag and setting it externally. `ParkingLot` tells `Ticket` to mark itself used. This keeps state mutations inside the class that owns the state, preventing callers from making decisions based on internals they shouldn't touch.

---

## Verification

Lot has spots: M1 (MOTORCYCLE), C1 (CAR), C2 (CAR), L1 (LARGE). All `occupied = false`. Tickets map empty. `HOURLY_RATE = 5.0`.

**Entry — Car arrives:**
```
enter(CAR)
  findAvailableSpot(CAR) -> C1 (first unoccupied CAR spot)
  C1.markOccupied()      -> C1.occupied = true
  ticket = Ticket("T001", C1, 10:00am)
  tickets["T001"] = ticket

  Result: Ticket("T001")
  State: C1.occupied=true, tickets={"T001" -> Ticket}
```

**Entry — Second Car arrives:**
```
enter(CAR)
  findAvailableSpot(CAR) -> C2 (C1 occupied, C2 free)
  C2.markOccupied()      -> C2.occupied = true
  ticket = Ticket("T002", C2, 10:15am)
```

**Entry — Third Car (lot full for CAR):**
```
enter(CAR)
  findAvailableSpot(CAR) -> null (C1 and C2 both occupied)
  throw Error("No available spot for CAR")
  State unchanged.
```

**Exit — T001 at 12:30pm (2.5 hours later):**
```
exit("T001")
  ticket = tickets["T001"]  -> found
  ticket.isUsed()           -> false
  hoursParked = ceil(2.5)   = 3
  fee = 3 * 5.0             = 15.0
  C1.markFree()             -> C1.occupied = false
  ticket.markUsed()         -> ticket.used = true

  Result: 15.0
  State: C1.occupied=false, ticket.used=true
```

**Exit — T001 attempted again:**
```
exit("T001")
  ticket = tickets["T001"]  -> found
  ticket.isUsed()           -> true
  throw Error("Ticket already used")
  State unchanged.
```

The trace confirms: spot freed correctly, ticket invalidated, duplicate exit rejected, new car can now enter spot C1.

---

## Extensibility

### 1. "How would you support different rates per vehicle type?"

Right now `HOURLY_RATE` is a single constant. To support per-type pricing, replace it with a map.

> "I'd change `HOURLY_RATE` to a `Map<VehicleType, Double>` on ParkingLot. In `exit()`, look up the rate using `ticket.getSpot().getType()`. Everything else stays the same — the ceiling logic, state transitions, cleanup. The Spot already carries VehicleType so we have what we need without any schema change."

```
class ParkingLot:
    - rates: Map<VehicleType, Double>   // replaces HOURLY_RATE

exit(ticketId)
    ...
    type = ticket.getSpot().getType()
    fee  = hoursParked * rates.get(type)
    ...
```

---

### 2. "How would you support multiple floors or sections?"

Right now the lot is a flat list of spots. Adding floors is a structural grouping concern, not a logic change.

> "I'd introduce a `Section` class that owns a subset of spots. ParkingLot holds a list of sections instead of a flat spot list. `findAvailableSpot()` iterates over sections and delegates to each one. Ticket, Spot, and fee logic don't change at all — they operate on individual spots, not sections."

```
class Section:
    - id: String
    - spots: List<Spot>

    + findAvailableSpot(vehicleType) -> Spot?

class ParkingLot:
    - sections: List<Section>   // replaces flat spot list

findAvailableSpot(vehicleType)
    for section in sections
        spot = section.findAvailableSpot(vehicleType)
        if spot != null
            return spot
    return null
```

---

### 3. "How would you make spot selection smarter — e.g. nearest spot to entry?"

Right now `findAvailableSpot()` uses a simple first-available linear scan. For an SDE2 role, this is perfectly acceptable. For a stronger signal, I'd mention the **Strategy pattern** without implementing it upfront.

> "I'd extract a `SpotAllocationStrategy` interface with a single method. Two natural implementations: `FirstAvailableStrategy` (what we have today) and `NearestSpotStrategy` (picks by proximity to entry). ParkingLot takes the strategy as a constructor argument. For now I'd inject `FirstAvailableStrategy` because the requirements don't ask for anything smarter — but the extension point is clean and requires no changes to Ticket or Spot."

```
interface SpotAllocationStrategy:
    + findSpot(spots, vehicleType) -> Spot?

class FirstAvailableStrategy implements SpotAllocationStrategy:
    + findSpot(spots, vehicleType)
        for spot in spots
            if spot.getType() == vehicleType && !spot.isOccupied()
                return spot
        return null

class NearestSpotStrategy implements SpotAllocationStrategy:
    + findSpot(spots, vehicleType)
        // pick the unoccupied spot of matching type
        // with smallest distance to entry point
        ...

class ParkingLot:
    - strategy: SpotAllocationStrategy

    + ParkingLot(spots, hourlyRate, strategy)
```

I would **not** implement `NearestSpotStrategy` in the interview. Just naming the interface and the two implementations — and stating which one I'm using and why — is the right level of detail.

---

### 4. "What if you wanted to reserve a spot in advance?"

Currently, `enter()` assigns and occupies the spot atomically. To support reservations, we'd need a two-phase approach.

> "I'd add a `RESERVED` state to Spot. `reserve(vehicleType)` marks the spot `RESERVED` and returns a reservation ID. `confirmEntry(reservationId)` converts the reservation into a Ticket and marks the spot `OCCUPIED`. We'd also need a timeout — reservations older than N minutes auto-expire and free the spot. Ticket doesn't change. The main additions are a `Reservation` class and a `reservations` map on ParkingLot alongside the existing tickets map."
