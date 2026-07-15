# Movie Ticket Booking System

A movie ticket booking system (like Fandango or BookMyShow) lets users search for movies, browse theaters and showtimes, select specific seats from a seat map, and reserve tickets. The system manages seat availability across multiple theaters, each with multiple screens, and prevents two people from booking the same seat.

---

## Requirements

You walk into the interview and get greeted with this prompt:

> "Design a movie ticket booking system similar to BookMyShow that allows users to browse movies, select theaters and showtimes, book tickets, and manage reservations."

Before touching a whiteboard, spend 3–5 minutes narrowing that down.

### Clarifying Questions

**You:** "When you say 'browse movies,' is that full-text search, fuzzy matching, or just simple title lookup?"
**Interviewer:** "Simple text matching on the movie title. Nothing fancy."

That means we can iterate through our movie list, check if each title contains the search term, and return matches. No need for Elasticsearch, inverted indexes, or ranking algorithms. With a few hundred movies in memory, a linear scan takes microseconds.

**You:** "How does seat selection work? Does the user pick specific seats from a map, or does the system auto-assign? And can they book more than one seat at a time?"
**Interviewer:** "Users pick specific seats from a seat map. And yes, they can book multiple seats in one transaction."

So we're building a seat picker, not just a ticket counter. That means we need per-seat availability tracking. The actual UI for rendering the seat map is out of scope, but our system needs to expose which seats are available so a frontend could display them.

**You:** "Are we designing for a single theater or multiple? And do theaters have multiple screens?"
**Interviewer:** "Multiple theaters, each with multiple screens. A user can search for a movie and see where it's playing, or go to a specific theater and see what's on."

Multi-theater with multiple screens. Two entry points into the system: search by movie title globally, or browse a specific theater's offerings. Both paths funnel into picking a showtime, then picking seats.

**You:** "Do different screens have different seat configurations? Or can we standardize?"
**Interviewer:** "Standardize it. Every screen has the same layout: rows A through Z, seats 0 through 20."

Big simplification. One constant seat layout for every screen in the system means we don't need to model per-screen configurations.

**You:** "What does 'manage reservations' include? Cancel, reschedule, modify?"
**Interviewer:** "Cancel only. If someone wants a different showtime, they cancel and rebook."

No rescheduling logic. That keeps the reservation model simple.

**You:** "A couple of scoping questions: are there different seat types with different prices? And is payment processing in scope?"
**Interviewer:** "No to both. All seats are identical, and payment is out of scope. Assume it always succeeds."

Two fewer features to model. No pricing tiers, no payment state machine.

**You:** "What about concurrency? If two people try to book the same seat at the same time?"
**Interviewer:** "Handle it. Exactly one should succeed."

Concurrency is a core requirement.

### Final Requirements

| # | Requirement |
|---|-------------|
| 1 | Users can search for movies by title (substring match) |
| 2 | Users can browse all movies playing at a given theater |
| 3 | Theaters have multiple screens; all screens share the same layout: rows A–Z, seats 0–20 |
| 4 | Users can view available seats for a showtime and select specific ones |
| 5 | Users can book multiple seats in a single reservation; system returns a confirmation ID |
| 6 | Concurrent booking of the same seat: exactly one succeeds |
| 7 | Users can cancel a reservation by confirmation ID, releasing all its seats |

**Out of scope:** payment processing, seat types or pricing tiers, rescheduling, UI/rendering, user authentication.

---

## Core Entities and Relationships

The central challenge here isn't data modeling — it's **concurrent seat ownership**. Two users can race to claim the same seat. That means seat availability must be mutable state protected by a lock, and that lock must live at the **Showtime** level (not screen or theater) because the same physical seat is available for one showtime and booked for another.

**BookingSystem** — The orchestrator and public entry point. Resolves movies, theaters, and showtimes by ID, routes operations, and stores reservations by confirmation ID.

**Movie** — A film with a title. The only thing users search against. Stateless after creation.

**Theater** — A physical location containing multiple screens. Owns the screen collection and can flatten all its showtimes for browsing.

**Screen** — A room within a theater. Owns the list of showtimes scheduled on it. Holds a back-reference to its theater so a showtime can be traced back to a location.

**Showtime** — A specific screening of a movie on a screen at a given time. This is where per-seat availability lives: a `Map<SeatId, SeatStatus>` initialized with all 546 seats (26 rows × 21 seats) as `AVAILABLE`. Also owns the `ReentrantLock` that serializes concurrent booking attempts.

**Reservation** — A confirmed booking: a user, a showtime, a list of seats, a confirmation ID, and a status. The authoritative record of who holds which seats.

**SeatId** — A value object: `(row: char, seatNumber: int)`. Used as the key in Showtime's seat map. Must implement `equals` and `hashCode` — without them, two `SeatId('A', 1)` objects would be different map keys.

### Entity Responsibilities

| Entity | Responsibility |
|--------|----------------|
| `BookingSystem` | Entry point. Routes all operations. Maintains flat registries for O(1) lookup by ID. |
| `Theater` | Owns screens. Aggregates showtimes across all screens for theater browsing. |
| `Screen` | Owns showtimes. Back-reference to theater for navigation. |
| `Showtime` | Owns per-seat availability map. Enforces atomic seat reservation under lock. |
| `Movie` | Stores title and metadata. Target of search. |
| `Reservation` | Records a confirmed booking. Tracks status for cancellation guard. |
| `SeatId` | Value object identifying a specific seat. Map key — must have correct equality. |

### Relationships

```
BookingSystem
  ├── movies:       Map<id, Movie>
  ├── theaters:     Map<id, Theater>
  ├── showtimes:    Map<id, Showtime>     ← flat registry for O(1) lookup
  └── reservations: Map<confId, Reservation>

Theater
  └── screens: List<Screen>
        └── Screen
              ├── theater: Theater        ← back-reference
              └── showtimes: List<Showtime>
                    └── Showtime
                          ├── movie: Movie
                          ├── screen: Screen      ← back-reference
                          ├── seatStatus: Map<SeatId, SeatStatus>
                          └── lock: ReentrantLock
```

The flat `showtimes` registry on `BookingSystem` is the key structural decision. Without it, `bookSeats` would scan theaters → screens → showtimes on every call — O(total showtimes). With it, every lookup is O(1). The same logic that drives using a `Map` for directory children over a `List` in a file system.

---

## Class Design

We design top-down: `BookingSystem` first, then `Showtime` (the most nuanced), then `Theater`, `Screen`, `Movie`, `Reservation`, and `SeatId`.

### BookingSystem

BookingSystem is what callers interact with. It resolves paths into the object graph, performs operations, and keeps state consistent.

**Deriving State**

| Requirement | What BookingSystem must track |
|-------------|-------------------------------|
| "Search for movies by title" | All movies (iterable) |
| "Browse movies at a theater" | All theaters by ID |
| "Book seats for a showtime" | All showtimes by ID for O(1) lookup |
| "Cancel by confirmation ID" | All reservations by confirmation ID |

**Deriving Behavior**

| Need from requirements | Method on BookingSystem |
|------------------------|------------------------|
| "Search for movies by title" | `searchMovies(query) -> List<Movie>` |
| "Browse movies at a theater" | `getShowtimesForTheater(theaterId) -> List<Showtime>` |
| "Find where a movie is playing" | `getShowtimesForMovie(movieId) -> List<Showtime>` |
| "View available seats" | `getAvailableSeats(showtimeId) -> List<SeatId>` |
| "Book multiple seats" | `bookSeats(userId, showtimeId, seats) -> Reservation` |
| "Cancel a reservation" | `cancelReservation(confirmationId) -> void` |

```
class BookingSystem:
    - movies:       Map<String, Movie>
    - theaters:     Map<String, Theater>
    - showtimes:    Map<String, Showtime>
    - reservations: Map<String, Reservation>

    + BookingSystem()
    + searchMovies(query) -> List<Movie>
    + getShowtimesForTheater(theaterId) -> List<Showtime>
    + getShowtimesForMovie(movieId) -> List<Showtime>
    + getAvailableSeats(showtimeId) -> List<SeatId>
    + bookSeats(userId, showtimeId, seats) -> Reservation
    + cancelReservation(confirmationId) -> void
```

---

### Showtime

Showtime is the most critical class. It owns the mutable per-seat state and the lock that protects it.

**Deriving State**

| Requirement | What Showtime must track |
|-------------|--------------------------|
| "View available seats / book specific seats" | Per-seat availability — `Map<SeatId, SeatStatus>` |
| "Exactly one concurrent booking succeeds" | A lock serializing access to that map |
| "Back-navigate to theater/screen" | Reference to its Screen |
| "Filter showtimes by movie" | Reference to its Movie |

**Deriving Behavior**

| Need | Method on Showtime |
|------|-------------------|
| "View available seats" | `getAvailableSeats() -> List<SeatId>` |
| "Atomically check and claim seats" | `reserveSeats(seats) -> boolean` |
| "Release seats on cancellation" | `releaseSeats(seats) -> void` |

```
class Showtime:
    - id:         String
    - movie:      Movie
    - screen:     Screen
    - startTime:  DateTime
    - seatStatus: Map<SeatId, SeatStatus>    // initialized to all AVAILABLE
    - lock:       ReentrantLock

    + Showtime(id, movie, screen, startTime)
    + getId() -> String
    + getMovie() -> Movie
    + getScreen() -> Screen
    + getStartTime() -> DateTime
    + getAvailableSeats() -> List<SeatId>
    + reserveSeats(seats: List<SeatId>) -> boolean
    + releaseSeats(seats: List<SeatId>) -> void
```

`reserveSeats` and `releaseSeats` are the only methods that acquire the lock. `getAvailableSeats` is a best-effort read — the returned list is a snapshot hint for the UI; actual enforcement happens inside `reserveSeats`.

---

### Theater

**Deriving State**

| Requirement | What Theater must track |
|-------------|-------------------------|
| "Multiple screens per theater" | List of Screen objects |
| "Browse movies at a theater" | Its own identity for lookup |

```
class Theater:
    - id:       String
    - name:     String
    - location: String
    - screens:  List<Screen>

    + Theater(id, name, location)
    + getId() -> String
    + getScreens() -> List<Screen>
    + getAllShowtimes() -> List<Showtime>   // flattens screens → showtimes
    + addScreen(screen) -> void
```

---

### Screen

**Deriving State**

| Requirement | What Screen must track |
|-------------|------------------------|
| "Host multiple showtimes" | List of Showtime objects |
| "Navigate back to theater" | Back-reference to Theater |

```
class Screen:
    - id:        String
    - theater:   Theater
    - showtimes: List<Showtime>

    + Screen(id, theater)
    + getId() -> String
    + getTheater() -> Theater
    + getShowtimes() -> List<Showtime>
    + addShowtime(showtime) -> void
```

---

### Movie

**Deriving State**

| Requirement | What Movie must track |
|-------------|----------------------|
| "Search by title" | Title string |
| "Identify in flat showtimes registry" | Unique ID |

```
class Movie:
    - id:              String
    - title:           String
    - description:     String
    - durationMinutes: int

    + Movie(id, title, description, durationMinutes)
    + getId() -> String
    + getTitle() -> String
```

---

### Reservation

**Deriving State**

| Requirement | What Reservation must track |
|-------------|------------------------------|
| "Return confirmation ID on booking" | The confirmation ID |
| "Cancel by confirmation ID" | Which showtime and seats to release |
| "Guard against double-cancel" | Current status |

```
class Reservation:
    - confirmationId: String
    - userId:         String
    - showtime:       Showtime
    - seats:          List<SeatId>
    - status:         ReservationStatus   // CONFIRMED | CANCELLED

    + Reservation(confirmationId, userId, showtime, seats)
    + getConfirmationId() -> String
    + getSeats() -> List<SeatId>
    + getShowtime() -> Showtime
    + getStatus() -> ReservationStatus
    + cancel() -> void                    // sets status = CANCELLED
```

---

### SeatId (value object)

```
class SeatId:
    - row:        char    // 'A'..'Z'
    - seatNumber: int     // 0..20

    + SeatId(row, seatNumber)
    + equals(other) -> boolean    // compare both fields
    + hashCode() -> int           // hash both fields
    + toString() -> String        // e.g., "A1"
```

`SeatId` **must** override `equals()` and `hashCode()`. Without both, two `SeatId('A', 1)` objects would be treated as different map keys even though they represent the same seat. In Java, this is handled with `@EqualsAndHashCode` (Lombok) or implementing both methods manually.

---

### Enums

```
enum SeatStatus:
    AVAILABLE
    BOOKED

enum ReservationStatus:
    CONFIRMED
    CANCELLED
```

**Why not the State Pattern?**

`ReservationStatus` has exactly two states and one legal transition: `CONFIRMED → CANCELLED`. There are no polymorphic behaviors — no method whose logic forks by state, no entry/exit actions, no guard conditions. The State Pattern adds a class hierarchy and a context object to model transitions; here that machinery would cost more than the problem it solves.

A plain enum plus a guard in `cancelReservation` (`if status == CANCELLED → throw`) is sufficient and immediately readable. Introducing State Pattern here would signal pattern overuse to a sharp interviewer, not depth.

Knowing when **not** to apply a pattern is as valuable as knowing when to apply one.

---

### Final Class Design

```
+-----------------------------------+
|          BookingSystem            |
+-----------------------------------+
| - movies:       Map<id, Movie>    |
| - theaters:     Map<id, Theater>  |
| - showtimes:    Map<id, Showtime> |
| - reservations: Map<id, Reserv.>  |
+-----------------------------------+
| + searchMovies(query)             |
| + getShowtimesForTheater(id)      |
| + getShowtimesForMovie(id)        |
| + getAvailableSeats(id)           |
| + bookSeats(userId, stId, seats)  |
| + cancelReservation(confId)       |
+-----------------------------------+
         |                |
   owns  |          owns  |
         v                v
+-------------+   +----------------------------------+
|   Theater   |   |            Showtime              |
+-------------+   +----------------------------------+
| - id        |   | - id: String                     |
| - name      |   | - movie: Movie           ------+ |
| - location  |   | - screen: Screen         ----+ | |
| - screens   |   | - startTime: DateTime       | | |
+-------------+   | - seatStatus:               | | |
| +getAllSTs() |   |     Map<SeatId, SeatStatus> | | |
| +addScreen()|   | - lock: ReentrantLock       | | |
+-------------+   +----------------------------------+ |
      |           | + getAvailableSeats()         |   |
      |           | + reserveSeats(seats) -> bool |   |
      v           | + releaseSeats(seats)         |   |
+-------------+   +----------------------------------+  |
|   Screen    |<-----------------------------------------+
+-------------+
| - id        |
| - theater   |----> Theater (back-ref)
| - showtimes |
+-------------+
| +addShowtime|
+-------------+

+------------------+   +------------------+   +------------------+
|    Reservation   |   |      Movie        |   |     SeatId       |
+------------------+   +------------------+   +------------------+
| - confirmationId |   | - id             |   | - row: char      |
| - userId         |   | - title          |   | - seatNumber: int|
| - showtime       |   | - description    |   +------------------+
| - seats: List    |   | - durationMins   |   | + equals()       |
| - status         |   +------------------+   | + hashCode()     |
+------------------+                          +------------------+
| + cancel()       |
+------------------+
```

---

## Implementation

The three most interesting methods are `reserveSeats()` on Showtime (the critical section), `bookSeats()` on BookingSystem (orchestration), and `cancelReservation()`. We'll also walk through Showtime initialization and `searchMovies`.

### Showtime Constructor — seat map initialization

Every Showtime starts with all 546 seats marked `AVAILABLE`. This is where the standard layout is applied once.

```
Showtime(id, movie, screen, startTime)
    this.id        = id
    this.movie     = movie
    this.screen    = screen
    this.startTime = startTime
    this.lock      = new ReentrantLock()
    this.seatStatus = {}

    for row in 'A'..'Z':
        for seatNum in 0..20:
            seatStatus[SeatId(row, seatNum)] = AVAILABLE
```

---

### reserveSeats() — the critical section

This is the heart of concurrency safety. The lock covers the entire **check-all-then-book-all** sequence. Without it, two threads could both observe all seats as `AVAILABLE`, both pass the check, and both write `BOOKED` — double-booking the same seat.

**Core logic:**
1. Acquire the lock
2. Verify every requested seat is `AVAILABLE` — if any one fails, return false without modifying state
3. Only once all checks pass, mark every seat `BOOKED`
4. Release the lock

**Edge cases:**
- One seat in the list is already `BOOKED` — return false, leave all other seats unchanged (no partial booking)
- Caller passes duplicate `SeatId` values — the second occurrence finds its own first-pass write and returns false; the whole call fails cleanly

```
reserveSeats(seats)
    lock.acquire()
    try
        for seat in seats
            if seatStatus[seat] != AVAILABLE
                return false          // abort — no partial booking

        for seat in seats
            seatStatus[seat] = BOOKED

        return true
    finally
        lock.release()
```

The two-pass structure (check all, then write all) is deliberate. A single combined pass would partially book seats before discovering a conflict — leaving the system in a dirty state that requires rollback.

---

### bookSeats()

**Core logic:**
1. Look up the showtime
2. Validate the seat list is non-empty
3. Delegate to `showtime.reserveSeats()` — if it returns false, the seats were taken
4. Generate a confirmation ID, create and store the `Reservation`, return it

**Edge cases:**
- Showtime ID not found
- Empty seat list
- One or more seats unavailable — `reserveSeats` returns false

```
bookSeats(userId, showtimeId, seats)
    showtime = showtimes[showtimeId]
    if showtime == null
        throw Error("Showtime not found: " + showtimeId)

    if seats.isEmpty()
        throw Error("Must select at least one seat")

    success = showtime.reserveSeats(seats)
    if !success
        throw Error("One or more seats are no longer available")

    confirmationId = generateUUID()
    reservation = Reservation(confirmationId, userId, showtime, seats)
    reservations[confirmationId] = reservation
    return reservation
```

All concurrency logic lives in `Showtime.reserveSeats`. `BookingSystem.bookSeats` has no locking knowledge — it only handles routing and lifecycle. This matches the Information Expert principle: the class that owns the state enforces the rules on that state.

---

### cancelReservation()

**Core logic:**
1. Look up the reservation by confirmation ID
2. Guard against double-cancel
3. Release the seats back in the showtime
4. Mark the reservation `CANCELLED`

**Edge cases:**
- Confirmation ID not found
- Already cancelled

```
cancelReservation(confirmationId)
    reservation = reservations[confirmationId]
    if reservation == null
        throw Error("Reservation not found: " + confirmationId)

    if reservation.getStatus() == CANCELLED
        throw Error("Reservation already cancelled")

    reservation.getShowtime().releaseSeats(reservation.getSeats())
    reservation.cancel()
```

`releaseSeats` on Showtime mirrors `reserveSeats` — it acquires the same lock so a cancel never races with a concurrent booking attempt on the same showtime.

```
releaseSeats(seats)
    lock.acquire()
    try
        for seat in seats
            seatStatus[seat] = AVAILABLE
    finally
        lock.release()
```

There is no recursive cleanup needed. Cancelling does not delete the `Reservation` object — it marks it `CANCELLED` and releases the seats. The record persists so the confirmation ID remains traceable.

---

### searchMovies()

Linear scan on movie titles. With a few hundred movies in memory this is microseconds and no index is needed.

```
searchMovies(query)
    results = []
    lowerQuery = query.toLowerCase()
    for movie in movies.values()
        if movie.getTitle().toLowerCase().contains(lowerQuery)
            results.add(movie)
    return results
```

---

## Verification

Let's trace a sequence of operations to verify state transitions.

**Setup:** Theater "AMC", Screen "S1", Movie "Inception" (id: M1), Showtime "ST1" at 7pm registered in BookingSystem. All 546 seats start `AVAILABLE`.

**View available seats:**
```
getAvailableSeats("ST1")
  showtime = showtimes["ST1"]  ✓
  showtime.getAvailableSeats()
    iterate seatStatus, collect AVAILABLE entries
  → [A0, A1, A2, ..., Z20]   (546 seats)
```

**Alice books A1 and A2:**
```
bookSeats("alice", "ST1", [SeatId('A',1), SeatId('A',2)])
  showtime = showtimes["ST1"]  ✓
  seats not empty  ✓
  showtime.reserveSeats([A1, A2]):
    lock.acquire()
    check A1: AVAILABLE  ✓
    check A2: AVAILABLE  ✓
    set A1 = BOOKED
    set A2 = BOOKED
    lock.release()  → return true
  confirmationId = "CONF-001"
  reservation = Reservation("CONF-001", "alice", ST1, [A1, A2], CONFIRMED)
  reservations["CONF-001"] = reservation
  → return reservation
```

**Bob concurrently tries to book A1 and A3 (lock serializes them):**
```
// Scenario: Alice's lock.release() happens first

bookSeats("bob", "ST1", [SeatId('A',1), SeatId('A',3)])
  showtime.reserveSeats([A1, A3]):
    lock.acquire()
    check A1: BOOKED  ✗         // Alice already claimed it
    lock.release()  → return false
  → throw Error("One or more seats are no longer available")
  State: Bob's booking fails. Exactly one succeeded. ✓
```

**Available seats after Alice's booking:**
```
getAvailableSeats("ST1")
  → [A0, A2..A20, B0..Z20]    (544 seats — A1 and A2 gone)
```

**Alice cancels CONF-001:**
```
cancelReservation("CONF-001")
  reservation = reservations["CONF-001"]  ✓
  status == CONFIRMED  ✓
  showtime.releaseSeats([A1, A2]):
    lock.acquire()
    set A1 = AVAILABLE
    set A2 = AVAILABLE
    lock.release()
  reservation.cancel()  → status = CANCELLED

  State: A1 and A2 available again; reservation record persists as CANCELLED.
```

**Attempt to cancel the same reservation again:**
```
cancelReservation("CONF-001")
  reservation.getStatus() == CANCELLED
  → throw Error("Reservation already cancelled")
  State: unchanged. ✓
```

**Cycle check — move equivalent: attempting to book on a cancelled reservation is impossible by design.** The reservation record is immutable after cancel; `bookSeats` always creates a new reservation via a new showtime lookup, never reusing an old one.

---

## Extensibility

### 1. "How would you add a seat hold — let users hold seats for 10 minutes while entering payment details?"

Right now booking is immediate and permanent. A hold is temporary ownership that expires.

> "I'd introduce `SeatStatus.HELD` and a parallel `Map<SeatId, HoldInfo>` on Showtime, where `HoldInfo` stores `userId` and `expiresAt`. A new `holdSeats(userId, showtimeId, seats, durationMs)` method acquires the same lock, checks all seats are `AVAILABLE`, marks them `HELD`, and records the expiry. `reserveSeats` would also accept `HELD` seats belonging to the same user — confirming the hold into a permanent booking. Expiry is handled lazily: at the top of `reserveSeats` and `holdSeats`, sweep expired holds back to `AVAILABLE` before checking. No background thread needed, and the existing lock already serializes these checks. The rest of the system — `BookingSystem`, `Reservation`, `cancelReservation` — is untouched."

---

### 2. "How would you support different seat types — Standard, Premium, Recliner — with different prices?"

Currently all seats are identical. Pricing and type differentiation require more structure.

> "I'd add a `Map<SeatId, SeatType>` to Screen, populated at setup time per seat. `SeatType` would be an enum (`STANDARD`, `PREMIUM`, `RECLINER`) with an associated price. `getAvailableSeats` would return `SeatInfo` objects (SeatId + SeatType + price) instead of bare SeatIds. `Reservation` would track the total amount charged. Showtime's `seatStatus` map keys are still `SeatId`, so the lock and reserve/release logic are completely unchanged. This is purely additive — no existing class needs modification beyond Screen gaining a type assignment at construction."

---

### 3. "How would you make this thread-safe for distributed booking across multiple servers?"

The current `ReentrantLock` per Showtime only works within a single process. With multiple servers, two requests on different nodes can both see a seat as available.

> "I'd replace the in-process `ReentrantLock` with a distributed lock — Redis's `SET NX PX` pattern works well here. The lock key would be the `showtimeId`, and seat status would move from an in-memory map to a shared store (Redis hash or a database row per seat). The `reserveSeats` logic is structurally identical: acquire distributed lock, check all seats, write if available, release. The tradeoff is that network round-trips make each booking slower. An alternative for higher throughput is optimistic locking with database-level compare-and-swap — attempt a conditional write (`UPDATE seat SET status='BOOKED' WHERE status='AVAILABLE'`) and retry on conflict. Both preserve the one-winner invariant; they differ in how contention is handled (blocking lock vs. retry loop)."
