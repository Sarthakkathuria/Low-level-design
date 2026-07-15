# Splitwise

A shared expense tracking system. Users add expenses to groups, split costs using different schemes — equal, exact amounts, or percentages — and the system maintains a live view of who owes whom. When it is time to settle up, the system can suggest the minimum number of payments needed to clear all debts.

---

## Requirements

You walk into the interview and get greeted with this prompt:

> "Design the low-level design of a system like Splitwise, which allows users to track shared expenses and settle debts. The system should support features like adding expenses, splitting costs among users, and tracking balances."

### Clarifying Questions

**You:** "What split types should we support? Just equal, or also exact amounts and percentages?"
**Interviewer:** "All three — equal, exact amounts per person, and percentage-based splits."

**You:** "When splitting, must all group members participate, or can an expense involve only a subset?"
**Interviewer:** "A subset. A dinner might only involve three of the five people on a trip."

**You:** "Is splitting always within a group, or can two people split expenses directly?"
**Interviewer:** "Always within a group. Two people can form a group of two."

**You:** "What does 'tracking balances' mean exactly — net balance per user pair, or full transaction history?"
**Interviewer:** "Track the net balance between each pair of users within a group — what one owes the other."

**You:** "Should the system suggest simplified settlements to minimize the number of transactions needed to clear all debts?"
**Interviewer:** "Yes, include that."

**You:** "When someone pays a debt back, should we record that as a settlement?"
**Interviewer:** "Yes — there should be a way to record actual payments, which update the outstanding balances."

**You:** "Multiple currencies?"
**Interviewer:** "No, single currency throughout."

**You:** "Concurrency?"
**Interviewer:** "Not for now."

### Final Requirements

| # | Requirement |
|---|-------------|
| 1 | Users can be created and added to groups |
| 2 | Expenses are added within a group: who paid, total amount, description, split type, and which members participate |
| 3 | Split types: equal (each participant pays the same), exact (caller specifies each person's amount), percentage (caller specifies each person's percentage; must sum to 100%) |
| 4 | An expense can involve any subset of the group's members |
| 5 | System maintains normalized net pairwise balances within each group: for any two users, at most one owes the other |
| 6 | Users can record a settlement (actual payment), which reduces outstanding balances |
| 7 | System can compute a simplified settlement plan that clears all debts in the minimum number of transactions |

**Out of scope:** multiple currencies, payment processing, UI, recurring expenses, expense categories, notifications, concurrency.

---

## Core Entities and Relationships

The central challenge here is **two-layered** — split-type polymorphism at the expense level, and debt minimization at the group level.

When an expense is added, the system must compute each participant's share, but the calculation differs completely by split type: equal divides evenly, exact reads provided amounts, percentage multiplies by ratios. Hard-coding these with a switch statement means touching that code every time a new split type is added. This points to a **Strategy pattern** — each split type is a standalone class that computes shares given a total amount. Adding a new split type means implementing one interface.

The second challenge is debt simplification. After several expenses, each user pair accumulates a net balance: "A owes B $30, B owes C $20, A owes C $10." Naive settlement requires one payment per balance entry. But net positions reveal that a single payment from A can clear what B and C both owe. Finding the minimum transaction set is a greedy graph problem: compute each user's net position (sum of all credits minus all debts), then repeatedly match the most-owed creditor against the most-owing debtor until all positions reach zero.

**SplitwiseApp** — The orchestrator and public entry point. Manages users and groups, routes all operations, and is the only class a caller interacts with directly.

**User** — A person in the system. Immutable after creation.

**Group** — A collection of users who share expenses. Owns the expense list, membership, and the pairwise balance map. Balances belong here — not on `SplitwiseApp` — because a debt only exists in the context of a group's shared expenses. The same two users in two different groups have independent balances.

**Expense** — A record of a payment. Stores who paid, the total amount, and the final computed shares per participant. The strategy object is not retained — the computed shares are the authoritative record of what each person is responsible for.

**SplitStrategy (interface)** — The strategy abstraction. One method computes shares; a second validates the inputs before any state is touched. Adding a new split type means implementing this interface only.

**EqualSplitStrategy** — Divides total evenly. Carries the list of participants.

**ExactSplitStrategy** — Uses caller-provided amounts. Validates that they sum to the total.

**PercentageSplitStrategy** — Multiplies the total by each participant's percentage. Validates that percentages sum to 100.

**Settlement** — A value object representing a directed payment: from one user to another for a given amount. Used as the return type of `simplifyDebts()` and as the input to `recordSettlement()`.

### Entity Responsibilities

| Entity | Responsibility |
|--------|----------------|
| `SplitwiseApp` | Entry point. Manages users and groups. Routes all operations. |
| `Group` | Owns members, expense history, and pairwise balances. Enforces normalized balance updates. |
| `User` | Stores identity. The unit balances are tracked against. |
| `Expense` | Records a payment and its final computed shares. Immutable after creation. |
| `SplitStrategy` | Interface. Computes each participant's share from a total amount; validates inputs. |
| `EqualSplitStrategy` | Equal share for every participant. |
| `ExactSplitStrategy` | Caller-specified share per participant; validates sum equals total. |
| `PercentageSplitStrategy` | Percentage-based share per participant; validates percentages sum to 100. |
| `Settlement` | Value object: a directed payment from one user to another. |

### Relationships

```
SplitwiseApp
  ├── users:  Map<id, User>
  └── groups: Map<id, Group>
                  └── Group
                        ├── members:  List<User>
                        ├── expenses: List<Expense>
                        └── balances: Map<debtorId, Map<creditorId, Double>>

Expense
  ├── paidBy:      User
  ├── totalAmount: double
  └── shares:      Map<User, Double>    ← result of SplitStrategy at creation time

<<interface>> SplitStrategy
  + computeShares(totalAmount) -> Map<User, Double>
  + validate(totalAmount) -> void
        ^                  ^                     ^
        |                  |                     |
EqualSplitStrategy  ExactSplitStrategy  PercentageSplitStrategy

Settlement (value object)
  ├── from:   User
  ├── to:     User
  └── amount: double
```

`balances[A][B] = X` means user A owes user B the amount X. Balances are **normalized on every update** — if B already owes A $30 and A then owes B $20, the net result is stored as B owes A $10. Only one direction between any pair is ever non-zero at a time.

---

## Class Design

Top-down: `SplitwiseApp`, then `Group`, `User`, `Expense`, the `SplitStrategy` interface and each implementation, then `Settlement`.

### SplitwiseApp

**Deriving State**

| Requirement | What SplitwiseApp must track |
|-------------|------------------------------|
| "Users can be created" | All users by ID for lookup |
| "Expenses added to groups, settlements recorded per group" | All groups by ID |

**Deriving Behavior**

| Need from requirements | Method on SplitwiseApp |
|------------------------|------------------------|
| "Create a user" | `addUser(name) -> User` |
| "Create a group with initial members" | `createGroup(name, memberIds) -> Group` |
| "Add a member to an existing group" | `addMember(groupId, userId) -> void` |
| "Add an expense to a group" | `addExpense(groupId, description, amount, paidByUserId, splitStrategy) -> Expense` |
| "View net pairwise balances" | `getGroupBalances(groupId) -> Map<String, Map<String, Double>>` |
| "Record an actual settlement payment" | `recordSettlement(groupId, settlement) -> void` |
| "Compute minimum settlement plan" | `simplifyDebts(groupId) -> List<Settlement>` |

```
class SplitwiseApp:
    - users:  Map<String, User>
    - groups: Map<String, Group>

    + SplitwiseApp()
    + addUser(name) -> User
    + createGroup(name, memberIds: List<String>) -> Group
    + addMember(groupId, userId) -> void
    + addExpense(groupId, description, amount, paidByUserId, splitStrategy) -> Expense
    + getGroupBalances(groupId) -> Map<String, Map<String, Double>>
    + recordSettlement(groupId, settlement: Settlement) -> void
    + simplifyDebts(groupId) -> List<Settlement>
```

---

### Group

Group is the core state owner. It holds who belongs to it, what expenses have been logged, and the current normalized pairwise balances.

**Deriving State**

| Requirement | What Group must track |
|-------------|----------------------|
| "Any subset of members can be split participants" | Membership list (for validation on add) |
| "Net pairwise balances within the group" | `balances: Map<debtorId, Map<creditorId, Double>>` |
| "Track expense history" | `expenses: List<Expense>` |

**Deriving Behavior**

| Need | Method on Group |
|------|----------------|
| "Add a member" | `addMember(user) -> void` |
| "Store expense and update balances" | `addExpense(expense) -> void` |
| "Reduce balances when a payment is made" | `applySettlement(settlement) -> void` |
| "Expose current balances" | `getBalances() -> Map<String, Map<String, Double>>` |
| "Expose expense history" | `getExpenses() -> List<Expense>` |
| "Normalize a single debtor-creditor balance update" | `updateBalance(debtorId, creditorId, amount) -> void` (internal) |

```
class Group:
    - id:       String
    - name:     String
    - members:  List<User>
    - expenses: List<Expense>
    - balances: Map<String, Map<String, Double>>   // balances[debtorId][creditorId] = amount

    + Group(id, name)
    + addMember(user) -> void
    + addExpense(expense) -> void
    + applySettlement(settlement: Settlement) -> void
    + getBalances() -> Map<String, Map<String, Double>>
    + getExpenses() -> List<Expense>
    - updateBalance(debtorId, creditorId, amount) -> void
```

---

### User

```
class User:
    - id:   String
    - name: String

    + User(id, name)
    + getId()   -> String
    + getName() -> String
```

---

### Expense

```
class Expense:
    - id:          String
    - description: String
    - totalAmount: double
    - paidBy:      User
    - shares:      Map<User, Double>    // each participant's computed share
    - createdAt:   DateTime

    + Expense(id, description, totalAmount, paidBy, shares, createdAt)
    + getId()          -> String
    + getDescription() -> String
    + getTotalAmount() -> double
    + getPaidBy()      -> User
    + getShares()      -> Map<User, Double>
    + getCreatedAt()   -> DateTime
```

`shares` stores the output of the split strategy computed at creation time. The strategy object is not retained — the computed split is the permanent record. If the payer is included in the split (normal case), their share is recorded too; only non-payer participants generate a debt.

---

### SplitStrategy (interface)

```
interface SplitStrategy:
    + computeShares(totalAmount: double) -> Map<User, Double>
    + validate(totalAmount: double) -> void    // throws on invalid input
```

`validate` is called before `computeShares` in `addExpense`. Bad input — percentages that do not sum to 100, exact amounts that do not sum to the total — is caught immediately, before any expense is created or any balance is touched. Fail fast, not silently.

---

### EqualSplitStrategy

```
class EqualSplitStrategy implements SplitStrategy:
    - participants: List<User>

    + EqualSplitStrategy(participants: List<User>)
    + validate(totalAmount)
        if participants.isEmpty()
            throw Error("Must have at least one participant")
    + computeShares(totalAmount) -> Map<User, Double>
        share = totalAmount / participants.size()
        return { user -> share for each user in participants }
```

---

### ExactSplitStrategy

```
class ExactSplitStrategy implements SplitStrategy:
    - shares: Map<User, Double>    // caller-specified amount per participant

    + ExactSplitStrategy(shares: Map<User, Double>)
    + validate(totalAmount)
        sum = shares.values().sum()
        if abs(sum - totalAmount) > 0.01
            throw Error("Exact amounts must sum to total. Expected: " + totalAmount + ", got: " + sum)
    + computeShares(totalAmount) -> Map<User, Double>
        return shares    // validated; return as-is
```

The epsilon (`0.01`) in validation handles floating-point rounding. Without it, $33.33 + $33.33 + $33.34 would fail validation against $100.00.

---

### PercentageSplitStrategy

```
class PercentageSplitStrategy implements SplitStrategy:
    - percentages: Map<User, Double>    // e.g., Alice: 50.0, Bob: 30.0, Charlie: 20.0

    + PercentageSplitStrategy(percentages: Map<User, Double>)
    + validate(totalAmount)
        sum = percentages.values().sum()
        if abs(sum - 100.0) > 0.01
            throw Error("Percentages must sum to 100, got: " + sum)
    + computeShares(totalAmount) -> Map<User, Double>
        return { user -> (pct / 100.0) * totalAmount for (user, pct) in percentages }
```

---

### Settlement (value object)

```
class Settlement:
    - from:   User
    - to:     User
    - amount: double

    + Settlement(from, to, amount)
    + getFrom()   -> User
    + getTo()     -> User
    + getAmount() -> double
```

Used in two contexts without modification: `simplifyDebts()` returns a list of suggested `Settlement` objects; `recordSettlement()` accepts one to record an actual payment.

---

### Final Class Design

```
+------------------------------------------+
|            SplitwiseApp                  |
+------------------------------------------+
| - users:  Map<String, User>              |
| - groups: Map<String, Group>             |
+------------------------------------------+
| + addUser(name) -> User                  |
| + createGroup(name, memberIds) -> Group  |
| + addMember(groupId, userId)             |
| + addExpense(groupId, desc, amount,      |
|     paidByUserId, splitStrategy)         |
|     -> Expense                           |
| + getGroupBalances(groupId)              |
| + recordSettlement(groupId, settlement)  |
| + simplifyDebts(groupId) -> List         |
+------------------------------------------+
              |
        owns  |
              v
+----------------------------------------------+
|                   Group                      |
+----------------------------------------------+
| - id:       String                           |
| - name:     String                           |
| - members:  List<User>                       |
| - expenses: List<Expense>                    |
| - balances: Map<id, Map<id, Double>>         |
+----------------------------------------------+
| + addMember(user)                            |
| + addExpense(expense)                        |
| + applySettlement(settlement)                |
| + getBalances()                              |
| - updateBalance(debtorId, creditorId, amt)   |
+----------------------------------------------+

+------------------+    +----------------------------------+
|       User       |    |             Expense              |
+------------------+    +----------------------------------+
| - id:   String   |    | - id:          String            |
| - name: String   |    | - description: String            |
+------------------+    | - totalAmount: double            |
                        | - paidBy:      User              |
                        | - shares: Map<User, Double>      |
                        | - createdAt:   DateTime          |
                        +----------------------------------+

             <<interface>>
            SplitStrategy
+ computeShares(totalAmount) -> Map<User, Double>
+ validate(totalAmount) -> void
      ^              ^                    ^
      |              |                    |
EqualSplit      ExactSplit        PercentageSplit
Strategy        Strategy          Strategy
-participants   -shares: Map      -percentages: Map

+------------------------+
|       Settlement       |
+------------------------+
| - from:   User         |
| - to:     User         |
| - amount: double       |
+------------------------+
```

---

## Implementation

The four most interesting methods are `addExpense()` (orchestration and validation), `Group.updateBalance()` (normalized balance maintenance), `simplifyDebts()` (the greedy debt-minimization algorithm), and `EqualSplitStrategy.computeShares()` to ground the Strategy pattern.

### addExpense()

**Core logic:**
1. Look up the group and the payer
2. Validate the split strategy against the total amount — fail before touching any state
3. Compute shares via the strategy
4. Create the expense and hand it to the group, which stores it and updates balances
5. Return the created expense

**Edge cases:**
- Group or payer not found — throw before calling validate
- Split validation fails (amounts don't sum to total, percentages off) — throw before any expense is created or balance touched
- Payer is included in the split (normal) — their share is recorded in the expense but generates no debt to themselves

```
addExpense(groupId, description, totalAmount, paidByUserId, splitStrategy)
    group  = groups[groupId]
    paidBy = users[paidByUserId]
    if group  == null  throw Error("Group not found: "  + groupId)
    if paidBy == null  throw Error("User not found: " + paidByUserId)

    splitStrategy.validate(totalAmount)
    shares = splitStrategy.computeShares(totalAmount)

    expense = Expense(uuid(), description, totalAmount, paidBy, shares, now())
    group.addExpense(expense)
    return expense
```

**Inside `Group.addExpense(expense)`:**

```
addExpense(expense)
    expenses.add(expense)

    paidById = expense.getPaidBy().getId()
    for (participant, share) in expense.getShares()
        if participant.getId() != paidById
            updateBalance(participant.getId(), paidById, share)
```

Only non-payer participants generate a debt. If Alice paid and she is included in the split, her $30 share means the group already "got" her $30 — no debt to record.

---

### Group.updateBalance()

This is the normalized balance update. When a new debt is recorded, we check whether the creditor already owes the debtor. If so, the amounts net before being stored — only one direction between any pair is ever non-zero.

**Core logic:**
1. Check if the creditor (person being owed) already has a reverse debt to the debtor
2. If the reverse debt is larger: reduce it — creditor's existing obligation absorbs the new one
3. If the reverse debt is smaller: clear it and record the net remainder as debtor → creditor
4. If no reverse debt: add directly to the existing debtor → creditor balance

**Edge cases:**
- No prior balance between this pair — create new entry
- Reverse debt exactly matches new amount — both entries drop to zero (no entry stored)

```
updateBalance(debtorId, creditorId, amount)
    // Does the creditor already owe the debtor something (reverse direction)?
    reverseOwed = balances.getOrDefault(creditorId, {}).getOrDefault(debtorId, 0.0)

    if reverseOwed > 0
        if reverseOwed >= amount
            // The reverse debt fully absorbs the new obligation
            balances[creditorId][debtorId] -= amount
            if balances[creditorId][debtorId] < 0.01
                balances[creditorId].remove(debtorId)      // clean up zero entry
        else
            // Reverse debt is absorbed; debtor still owes the remainder
            balances[creditorId].remove(debtorId)
            net = amount - reverseOwed
            balances[debtorId][creditorId] =
                balances.getOrDefault(debtorId, {}).getOrDefault(creditorId, 0.0) + net
    else
        // No reverse debt — accumulate directly
        balances[debtorId][creditorId] =
            balances.getOrDefault(debtorId, {}).getOrDefault(creditorId, 0.0) + amount
```

Normalization here keeps `getGroupBalances()` clean: for any two users, exactly one direction is non-zero (or neither is). This also makes `simplifyDebts` simpler — the input is already partially simplified.

---

### applySettlement()

Recording a payment is structurally identical to recording a debt in the reverse direction. If Bob owes Alice $30 and pays $30, that is equivalent to Alice now having a new $30 "credit" against Bob — which `updateBalance` normalizes away to zero.

```
applySettlement(settlement)
    // Payment from → to reduces from's debt to to.
    // Modeled as: to now "owes" from the settlement amount → updateBalance nets it away.
    updateBalance(settlement.getTo().getId(), settlement.getFrom().getId(), settlement.getAmount())
```

Reusing `updateBalance` for settlements means there is one place where balance normalization logic lives.

---

### simplifyDebts()

The greedy minimum-transaction algorithm. Three steps: flatten pairwise balances into net positions per user, split into creditors and debtors, greedily match them.

**Core logic:**
1. Compute each user's net position: sum all amounts they are owed, subtract all amounts they owe
2. Positive net = creditor (is owed money); negative net = debtor (owes money)
3. Repeatedly pair the largest-credit user with the largest-debt user — the settlement is the smaller of the two; the larger carries a remainder back into contention
4. Stop when all net positions reach zero

**Edge cases:**
- No outstanding balances → return empty list
- Two users with equal and opposite positions → one transaction, both drop to zero

```
simplifyDebts(groupId)
    group = groups[groupId]
    if group == null throw Error("Group not found: " + groupId)

    // Step 1: compute net balance per user
    // positive = is owed (creditor), negative = owes money (debtor)
    netBalance = Map<String, Double>()
    for (debtorId, creditorMap) in group.getBalances()
        for (creditorId, amount) in creditorMap
            netBalance[debtorId]   -= amount
            netBalance[creditorId] += amount

    // Step 2: separate into max-heaps
    creditors = MaxHeap<(User, Double)> by amount   // most-owed first
    debtors   = MaxHeap<(User, Double)> by amount   // most-owing first (absolute value)

    for (userId, balance) in netBalance
        if balance  >  0.01:  creditors.push((users[userId], balance))
        if balance  < -0.01:  debtors.push((users[userId], abs(balance)))

    // Step 3: greedy matching
    settlements = []
    while !creditors.isEmpty() and !debtors.isEmpty()
        (creditorUser, creditAmt) = creditors.pop()
        (debtorUser,  debtAmt)   = debtors.pop()

        amount = min(creditAmt, debtAmt)
        settlements.add(Settlement(debtorUser, creditorUser, amount))

        remaining_credit = creditAmt - amount
        remaining_debt   = debtAmt   - amount

        if remaining_credit > 0.01:  creditors.push((creditorUser, remaining_credit))
        if remaining_debt   > 0.01:  debtors.push((debtorUser,   remaining_debt))

    return settlements
```

This produces at most N-1 transactions for N users, which is provably optimal. Two users with matching but opposite positions cancel completely in one pairing — regardless of how many individual expenses built up those positions.

---

## Verification

**Setup:** Group "Weekend Trip" with members Alice (A), Bob (B), Charlie (C).

---

**Expense 1:** Alice pays $90 for dinner, equal split among A, B, C.

```
EqualSplitStrategy([A, B, C]).validate(90)   → participants not empty ✓
EqualSplitStrategy([A, B, C]).computeShares(90) → {A:30, B:30, C:30}

group.addExpense(expense):
  paidById = Alice
  for B ($30): updateBalance("B", "A", 30)
    reverseOwed = balances["A"]["B"] = 0   → no reverse
    balances["B"]["A"] = 30
  for C ($30): updateBalance("C", "A", 30)
    reverseOwed = balances["A"]["C"] = 0   → no reverse
    balances["C"]["A"] = 30

Balances after Expense 1:
  B → A: $30
  C → A: $30
```

---

**Expense 2:** Bob pays $60 for hotel, equal split among A, B, C.

```
computeShares(60) → {A:20, B:20, C:20}

group.addExpense(expense):
  paidById = Bob
  for A ($20): updateBalance("A", "B", 20)
    reverseOwed = balances["B"]["A"] = 30 > 0
    30 >= 20 → reduce: balances["B"]["A"] = 30 - 20 = 10   ← NORMALIZATION
  for C ($20): updateBalance("C", "B", 20)
    reverseOwed = balances["B"]["C"] = 0   → no reverse
    balances["C"]["B"] = 20

Balances after Expense 2:
  B → A: $10   (was $30; Alice's $20 share of Bob's hotel netted it down)
  C → A: $30
  C → B: $20
```

Note: no "A → B" entry exists. The $20 Alice owed Bob was absorbed against Bob's existing $30 debt to Alice, leaving only B → A = $10 net. This is the normalization working as intended.

---

**Simplify debts:**

```
simplifyDebts("TRIP")

Step 1: Net positions from balances
  B → A: $10  →  netBalance[B] -= 10,  netBalance[A] += 10
  C → A: $30  →  netBalance[C] -= 30,  netBalance[A] += 30
  C → B: $20  →  netBalance[C] -= 20,  netBalance[B] += 20

Net: A = +40,  B = +10,  C = -50

Step 2: Heaps
  creditors: [(A, 40), (B, 10)]
  debtors:   [(C, 50)]

Step 3: Greedy matching
  Round 1: pop A (+40) and C (-50)
    amount = min(40, 50) = 40
    → Settlement: C pays A $40
    creditAmt remaining = 0   → A done
    debtAmt remaining  = 10   → push (C, 10) back

  Round 2: pop B (+10) and C (-10)
    amount = min(10, 10) = 10
    → Settlement: C pays B $10
    both remaining = 0 → both done

Result: [C pays A $40,  C pays B $10]
```

Two transactions clear all three balance entries. Without simplification, settling the raw balances would require three separate payments.

---

**Record settlement: C pays A $40**

```
recordSettlement("TRIP", Settlement(C, A, 40))

applySettlement(Settlement(C, A, 40)):
  updateBalance(toId="A", fromId="C", amount=40)
  → Check: does A (creditorId) owe C (debtorId)? balances["A"]["C"] = 0  → no reverse
  → balances["C"]["A"] = 30 + ... wait, we need to reduce C→A

Wait — let me re-read applySettlement:
  settlement.from = C, settlement.to = A
  updateBalance(A_id, C_id, 40)
  → debtorId = A, creditorId = C
  → Check: does C (creditorId) owe A (debtorId)?
       balances["C"]["A"] = 30 > 0
  → reverseOwed (30) < amount (40)
       Clear balances["C"]["A"]
       net = 40 - 30 = 10
       balances["A"]["C"] = 10   ← A now owes C $10 (C overpaid by $10)

Balances after C pays A $40:
  B → A: $10      (unchanged)
  C → B: $20      (unchanged)
  A → C: $10      (C overpaid; Alice owes Charlie $10 back)
```

The overpayment is handled correctly — the normalization logic in `updateBalance` detects that C paying $40 exceeds C's $30 debt to Alice, and records the $10 surplus as Alice owing Charlie.

---

**Validation failure: bad percentage split**

```
addExpense("TRIP", "Snacks", 50, "alice",
    PercentageSplitStrategy({A: 60, B: 30}))   // percentages sum to 90, not 100

validate(50):
    sum = 60 + 30 = 90
    abs(90 - 100) = 10 > 0.01
    → throw Error("Percentages must sum to 100, got: 90")

No expense created. No balances touched. ✓
```

---

## Extensibility

### 1. "How would you add a new split type — say, shares/ratio based (Alice:Bob:Charlie = 2:1:1)?"

> "I'd implement `SplitStrategy` with a `SharesSplitStrategy`. It stores a `Map<User, Integer>` of share counts per participant. `validate` checks that share counts are positive. `computeShares` divides the total proportionally: each person's amount is `(theirShares / totalShares) * totalAmount`. For the 2:1:1 example, total shares = 4; Alice gets 50%, Bob and Charlie 25% each. Nothing else changes — `SplitwiseApp`, `Group`, `updateBalance`, and all existing split types are completely untouched."

```
class SharesSplitStrategy implements SplitStrategy:
    - shares: Map<User, Integer>    // e.g., {Alice: 2, Bob: 1, Charlie: 1}

    + validate(totalAmount)
        if shares.isEmpty() or shares.values().any { it <= 0 }
            throw Error("Each participant must have a positive share count")

    + computeShares(totalAmount) -> Map<User, Double>
        totalShares = shares.values().sum()
        return { user -> (userShares / totalShares) * totalAmount
                 for (user, userShares) in shares }
```

---

### 2. "How would you support expense categories and per-category spending reports?"

Right now `Expense` has a description but no structured category. Adding category reporting requires categorization at the expense level and aggregation at query time.

> "I'd add a `category: String` field to `Expense` — set at creation time via `addExpense`. No existing balance or split logic changes. For reporting, I'd add a `getCategoryReport(groupId, userId) -> Map<String, Double>` method on `SplitwiseApp` that iterates `group.getExpenses()`, filters for expenses where the user has a non-zero share, and accumulates totals by category. This is purely a read-path addition — no writes or balance updates are affected."

---

### 3. "How would you support cross-group debt simplification — settling across all groups a user belongs to?"

Right now `simplifyDebts` is scoped to one group. A user in three groups may have debts in each that partially cancel across groups.

> "I'd add a `simplifyDebtsForUser(userId) -> List<Settlement>` method on `SplitwiseApp`. It iterates all groups the user is a member of, aggregates their net balance across all groups per counterpart (summing A→B across every group they share), then runs the same greedy heap algorithm on those aggregated totals. The `simplifyDebts` algorithm itself is unchanged — only the input is aggregated differently. Settlements returned would then need to be applied group-by-group: `recordSettlement` would need to know which group to credit. Alternatively, introduce a special 'settlement group' concept and record cross-group payments there."
