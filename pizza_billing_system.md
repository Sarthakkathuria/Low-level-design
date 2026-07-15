# Pizza Billing System

A pizza ordering system where customers build custom pizzas by selecting a base variety, size, and any number of toppings. Each topping is priced independently and stacked on top of the base. The system generates an itemized bill per order with tax applied.

---

## Requirements

You walk into the interview and get greeted with this prompt:

> "Design a pizza billing system where users can add toppings and cost would be calculated accordingly."

### Clarifying Questions

**You:** "What base pizza varieties do we need to support — margherita, BBQ chicken, veggie?"
**Interviewer:** "Yes, a few base varieties. Treat the exact list as configurable."

**You:** "Do pizzas come in sizes, and does size affect the price?"
**Interviewer:** "Yes — small, medium, and large. Each base variety has a different price per size."

**You:** "Can a customer add the same topping more than once — for example, double cheese?"
**Interviewer:** "Yes, double or triple of the same topping should be supported."

**You:** "Can a single order contain multiple pizzas?"
**Interviewer:** "Yes, an order can have one or more pizzas."

**You:** "Should the bill itemize each pizza's description and cost individually?"
**Interviewer:** "Yes — show each pizza as a line item with its full description and cost."

**You:** "Do we need to handle discounts or coupon codes?"
**Interviewer:** "Not in the core design, but keep it extensible."

**You:** "Should we apply tax on the order?"
**Interviewer:** "Yes — a fixed tax rate on the subtotal."

**You:** "Payment processing, delivery tracking, inventory?"
**Interviewer:** "All out of scope."

### Final Requirements

| # | Requirement |
|---|-------------|
| 1 | Customers can create an order containing one or more pizzas |
| 2 | Each pizza has a base variety (e.g., Margherita, BBQ Chicken, Veggie) and a size (small, medium, large) |
| 3 | Size determines the base price of the pizza |
| 4 | Any number of toppings can be added to a pizza, including duplicates (e.g., double cheese) |
| 5 | Each topping has a fixed price added on top of whatever the pizza costs so far |
| 6 | The system generates an itemized bill: one line item per pizza with full description and cost |
| 7 | A configurable tax rate is applied to the subtotal |
| 8 | Bill exposes: subtotal, tax amount, and total |

**Out of scope:** discounts/coupons (extensibility only), crust types (extensibility only), payment processing, delivery, inventory, UI, concurrency.

---

## Core Entities and Relationships

The central challenge here is **topping extensibility without modifying existing pizza classes**. A naive approach would hard-code all topping pricing in a central `PricingService` with a giant switch — adding a new topping means editing that service every time. Worse, tracking combinations (base + multiple toppings) in one place leads to a combinatorial mess.

The **Decorator pattern** solves this directly. A pizza is a thing that reports its cost and its description. A topping is a thing that *wraps* any pizza, adds its own price on top, and extends the description. The result: a `MargheritaPizza` wrapped in `ExtraCheeseTopping` wrapped in `MushroomTopping` is itself a `Pizza`. Calling `getCost()` on the outer layer triggers a chain: mushroom cost + (cheese cost + base cost). Adding a new topping means writing one new class — nothing else changes.

**Pizza (interface)** — The component interface. Every entity in the system that can be priced and described implements this: base pizzas and every topping decorator. This is what makes the chain composable.

**BasePizza** — Concrete base varieties. Each stores a `Size` and owns a price map from size to cost. These are the innermost objects in the decorator chain.

**ToppingDecorator (abstract)** — The abstract decorator. Holds a reference to the wrapped `Pizza` and delegates `getCost()` and `getDescription()` to it, adding its own contribution. Subclasses provide the topping name and price.

**Concrete toppings** — `ExtraCheeseTopping`, `PepperoniTopping`, `MushroomTopping`, etc. Each extends `ToppingDecorator` and provides only two values: the topping name and its price. That is all a new topping ever needs to add.

**Order** — Aggregates one or more fully-composed `Pizza` objects. Computes the subtotal by summing `getCost()` across all pizzas.

**Bill** — An immutable value object returned by `BillingSystem`. Carries line items, subtotal, tax, and total. The authoritative record of what the customer is charged.

**LineItem** — A value object inside `Bill`. One per pizza: description + cost.

**BillingSystem** — The orchestrator. Creates and tracks orders, accepts pizzas into orders, and generates bills. The only class a caller interacts with directly.

### Entity Responsibilities

| Entity | Responsibility |
|--------|----------------|
| `Pizza` | Interface: `getCost()` and `getDescription()`. The unit the billing system operates on. |
| `BasePizza` (per variety) | Concrete base: owns size-to-price mapping. The innermost decorator chain object. |
| `ToppingDecorator` | Abstract decorator: wraps a `Pizza`, delegates getCost/getDescription, adds its own contribution. |
| Concrete toppings | Each subclass provides a name and a price — nothing else. |
| `Order` | Aggregates pizzas. Computes subtotal as sum of pizza costs. |
| `Bill` | Value object: itemized line items, subtotal, tax amount, total. Immutable after creation. |
| `LineItem` | Value object: one pizza's description and cost. |
| `BillingSystem` | Entry point. Creates orders, accepts pizzas, generates bills. Applies tax. |

### Relationships

```
BillingSystem
  └── orders: Map<id, Order>
                  └── Order
                        └── pizzas: List<Pizza>

<<interface>> Pizza
  + getCost()        -> double
  + getDescription() -> String
        ^                       ^
        |                       |
  BasePizza             ToppingDecorator (abstract)
  (MargheritaPizza,       - wrapped: Pizza
   BBQChickenPizza,       + getCost()        -> wrapped.getCost() + toppingCost()
   VeggiePizza)           + getDescription() -> wrapped.getDescription() + toppingName()
  - size: Size                    ^
  - PRICES: Map<Size,Double>      |
                      ExtraCheeseTopping  PepperoniTopping  MushroomTopping  ...

Bill (value object)
  ├── lineItems: List<LineItem>
  ├── subtotal:  double
  ├── taxAmount: double
  └── total:     double

LineItem (value object)
  ├── description: String
  └── cost:        double
```

A fully composed pizza is a nested shell: `MushroomTopping(ExtraCheeseTopping(MargheritaPizza(LARGE)))`. Calling `getCost()` on the outermost shell walks the chain inward and accumulates the total.

---

## Class Design

Top-down: `BillingSystem`, then `Order`, then `Pizza` interface, `BasePizza` classes, `ToppingDecorator`, concrete toppings, and finally the value objects.

### BillingSystem

**Deriving State**

| Requirement | What BillingSystem must track |
|-------------|-------------------------------|
| "An order can contain multiple pizzas" | All active orders by ID |
| "Tax applied to subtotal" | The tax rate (configured at construction) |

**Deriving Behavior**

| Need from requirements | Method on BillingSystem |
|------------------------|-------------------------|
| "Create an order" | `createOrder() -> Order` |
| "Add pizza to order" | `addPizzaToOrder(orderId, pizza: Pizza) -> void` |
| "Generate itemized bill with tax" | `generateBill(orderId) -> Bill` |

```
class BillingSystem:
    - orders:   Map<String, Order>
    - TAX_RATE: double               // e.g., 0.10 for 10%

    + BillingSystem(taxRate: double)
    + createOrder() -> Order
    + addPizzaToOrder(orderId: String, pizza: Pizza) -> void
    + generateBill(orderId: String) -> Bill
```

---

### Order

**Deriving State**

| Requirement | What Order must track |
|-------------|----------------------|
| "Order contains one or more pizzas" | The list of fully-composed Pizza objects |
| "Show each pizza as a line item" | Identity (id) for bill association |

**Deriving Behavior**

| Need | Method on Order |
|------|----------------|
| "Add a pizza to the order" | `addPizza(pizza: Pizza) -> void` |
| "Read back the pizza list for billing" | `getPizzas() -> List<Pizza>` |
| "Compute subtotal for billing" | `getSubtotal() -> double` |

```
class Order:
    - id:        String
    - pizzas:    List<Pizza>
    - createdAt: DateTime

    + Order(id, createdAt)
    + getId()        -> String
    + addPizza(pizza: Pizza) -> void
    + getPizzas()    -> List<Pizza>
    + getSubtotal()  -> double       // sum of getCost() across all pizzas
```

---

### Pizza (interface)

```
interface Pizza:
    + getCost()        -> double
    + getDescription() -> String
```

Every entity in the system — base varieties and topping decorators alike — satisfies this interface. Callers never need to know which layer they are holding.

---

### BasePizza classes

Each base variety carries a size and a fixed price map. Size is set at construction because it is fundamental — you cannot build a pizza without choosing a size.

```
class MargheritaPizza implements Pizza:
    - size: Size
    - PRICES: Map<Size, Double> = {SMALL: 8.00, MEDIUM: 12.00, LARGE: 16.00}

    + MargheritaPizza(size: Size)
    + getCost()        -> PRICES[size]
    + getDescription() -> size.label() + " Margherita"

class BBQChickenPizza implements Pizza:
    - size: Size
    - PRICES: Map<Size, Double> = {SMALL: 10.00, MEDIUM: 15.00, LARGE: 20.00}

    + BBQChickenPizza(size: Size)
    + getCost()        -> PRICES[size]
    + getDescription() -> size.label() + " BBQ Chicken"

class VeggiePizza implements Pizza:
    - size: Size
    - PRICES: Map<Size, Double> = {SMALL: 9.00, MEDIUM: 13.00, LARGE: 17.00}

    + VeggiePizza(size: Size)
    + getCost()        -> PRICES[size]
    + getDescription() -> size.label() + " Veggie"
```

`Size` is a simple enum:

```
enum Size:
    SMALL, MEDIUM, LARGE

    + label() -> String    // "Small", "Medium", "Large"
```

---

### ToppingDecorator (abstract)

The abstract decorator holds the wrapped pizza and implements the interface by delegating to it, adding the subclass's own contribution.

```
abstract class ToppingDecorator implements Pizza:
    - wrapped: Pizza

    + ToppingDecorator(pizza: Pizza)
    + getCost()        -> wrapped.getCost() + toppingCost()
    + getDescription() -> wrapped.getDescription() + ", " + toppingName()

    # abstract toppingCost() -> double    // each subclass provides its price
    # abstract toppingName() -> String    // each subclass provides its display name
```

The two abstract methods are the only thing a new topping needs to implement. `getCost()` and `getDescription()` are inherited and never need to be overridden.

---

### Concrete Toppings

```
class ExtraCheeseTopping extends ToppingDecorator:
    # toppingCost() -> 2.50
    # toppingName() -> "Extra Cheese"

class PepperoniTopping extends ToppingDecorator:
    # toppingCost() -> 3.00
    # toppingName() -> "Pepperoni"

class MushroomTopping extends ToppingDecorator:
    # toppingCost() -> 1.50
    # toppingName() -> "Mushrooms"

class OliveTopping extends ToppingDecorator:
    # toppingCost() -> 1.00
    # toppingName() -> "Olives"

class JalapenoTopping extends ToppingDecorator:
    # toppingCost() -> 1.00
    # toppingName() -> "Jalapeños"
```

Each class is three lines. The decorator chain logic lives once in `ToppingDecorator`.

---

### Bill and LineItem (value objects)

```
class LineItem:
    - description: String
    - cost:        double

    + LineItem(description, cost)
    + getDescription() -> String
    + getCost()        -> double

class Bill:
    - orderId:   String
    - lineItems: List<LineItem>
    - subtotal:  double
    - taxAmount: double
    - total:     double

    + Bill(orderId, lineItems, subtotal, taxAmount, total)
    + getOrderId()   -> String
    + getLineItems() -> List<LineItem>
    + getSubtotal()  -> double
    + getTaxAmount() -> double
    + getTotal()     -> double
```

`Bill` is immutable after construction by `generateBill`. It is the permanent receipt for the order.

---

### Final Class Design

```
+-------------------------------------------+
|             BillingSystem                 |
+-------------------------------------------+
| - orders:   Map<String, Order>            |
| - TAX_RATE: double                        |
+-------------------------------------------+
| + createOrder() -> Order                  |
| + addPizzaToOrder(orderId, pizza)         |
| + generateBill(orderId) -> Bill           |
+-------------------------------------------+
              |
        owns  |
              v
+----------------------------------+
|              Order               |
+----------------------------------+
| - id:        String              |
| - pizzas:    List<Pizza>         |
| - createdAt: DateTime            |
+----------------------------------+
| + addPizza(pizza)                |
| + getPizzas() -> List<Pizza>     |
| + getSubtotal() -> double        |
+----------------------------------+
              |
    contains  | List<Pizza>
              v

       <<interface>> Pizza
       + getCost()        -> double
       + getDescription() -> String
              ^                   ^
              |                   |
    +---------+--------+   +------+--------------------+
    |    BasePizza     |   |   ToppingDecorator (abs)  |
    | (per variety)    |   |   - wrapped: Pizza        |
    | - size: Size     |   +---------------------------+
    | - PRICES: Map    |         ^       ^       ^
    +------------------+         |       |       |
                          Cheese  Pepperoni  Mushroom ...
                          Topping  Topping   Topping


+----------------------------+    +---------------------+
|          Bill              |    |       LineItem       |
+----------------------------+    +---------------------+
| - orderId:   String        |    | - description: Str  |
| - lineItems: List<LineItem>|    | - cost: double      |
| - subtotal:  double        |    +---------------------+
| - taxAmount: double        |
| - total:     double        |
+----------------------------+
```

---

## Implementation

The three most interesting pieces are `generateBill()` (orchestration: iterate pizzas, build line items, apply tax), `ToppingDecorator.getCost()` / `getDescription()` (the decorator chain), and `Order.getSubtotal()` (trivial but worth showing to ground the chain call).

### BillingSystem.generateBill()

**Core logic:**
1. Retrieve the order; fail if not found or empty
2. For each pizza, call `getCost()` and `getDescription()` — this walks the decorator chain automatically
3. Accumulate line items and sum the subtotal
4. Apply tax; round to two decimal places
5. Return an immutable `Bill`

**Edge cases:**
- Order ID not found — throw before any computation
- Order is empty (no pizzas added) — throw; billing an empty order is a caller error

```
generateBill(orderId)
    order = orders[orderId]
    if order == null
        throw Error("Order not found: " + orderId)
    if order.getPizzas().isEmpty()
        throw Error("Cannot generate bill for an empty order")

    lineItems = []
    subtotal  = 0.0

    for pizza in order.getPizzas()
        cost = pizza.getCost()            // walks the entire decorator chain
        desc = pizza.getDescription()     // same chain, builds description string
        lineItems.add(LineItem(desc, cost))
        subtotal += cost

    taxAmount = round(subtotal * TAX_RATE, 2)
    total     = subtotal + taxAmount

    return Bill(orderId, lineItems, subtotal, taxAmount, total)
```

The loop body is three lines. All the complexity of topping composition is invisible here — it lives in the chain.

---

### ToppingDecorator.getCost() and getDescription()

These are inherited by every concrete topping. They define the recursive chain.

```
// Abstract ToppingDecorator — inherited by all concrete toppings

getCost()
    return wrapped.getCost() + toppingCost()
    // If wrapped is another ToppingDecorator, this recurses until it hits a BasePizza.
    // BasePizza.getCost() returns the flat map lookup — the base case.

getDescription()
    return wrapped.getDescription() + ", " + toppingName()
    // Builds the description from inside out:
    // BasePizza returns "Large BBQ Chicken"
    // PepperoniTopping prepends: "Large BBQ Chicken, Pepperoni"
    // ExtraCheeseTopping prepends again: "Large BBQ Chicken, Pepperoni, Extra Cheese"
```

The recursion terminates at `BasePizza.getCost()` (which reads from the price map) and `BasePizza.getDescription()` (which returns the fixed variety+size string).

---

### Order.getSubtotal()

```
getSubtotal()
    total = 0.0
    for pizza in pizzas
        total += pizza.getCost()
    return total
```

This exists because `BillingSystem` can call it for a quick subtotal check before billing. Internally, `generateBill` does the same loop to build line items simultaneously — it does not call `getSubtotal()` to avoid a second traversal.

---

### MargheritaPizza.getCost() and getDescription()

The base case of the chain — no delegation, no recursion.

```
getCost()
    return PRICES[size]    // {SMALL: 8.00, MEDIUM: 12.00, LARGE: 16.00}

getDescription()
    return size.label() + " Margherita"
    // SMALL  → "Small Margherita"
    // MEDIUM → "Medium Margherita"
    // LARGE  → "Large Margherita"
```

---

## Verification

**Setup:** Tax rate = 10%. One order (O-001) with two pizzas.

---

**Building Pizza 1:** Medium Margherita + Extra Cheese + Mushrooms

```
pizza = MargheritaPizza(MEDIUM)
    // getCost()        → PRICES[MEDIUM] = 12.00
    // getDescription() → "Medium Margherita"

pizza = ExtraCheeseTopping(pizza)
    // getCost()        → wrapped.getCost() + 2.50
    //                 = 12.00 + 2.50 = 14.50
    // getDescription() → "Medium Margherita, Extra Cheese"

pizza = MushroomTopping(pizza)
    // getCost()        → wrapped.getCost() + 1.50
    //                 = 14.50 + 1.50 = 16.00
    // getDescription() → "Medium Margherita, Extra Cheese, Mushrooms"

Final Pizza 1: cost = $16.00, description = "Medium Margherita, Extra Cheese, Mushrooms"
```

---

**Building Pizza 2:** Large BBQ Chicken + double Pepperoni

```
pizza = BBQChickenPizza(LARGE)
    // getCost()        → PRICES[LARGE] = 20.00
    // getDescription() → "Large BBQ Chicken"

pizza = PepperoniTopping(pizza)           // first pepperoni
    // getCost()        = 20.00 + 3.00 = 23.00
    // getDescription() = "Large BBQ Chicken, Pepperoni"

pizza = PepperoniTopping(pizza)           // second pepperoni — same class, wrapped again
    // getCost()        = 23.00 + 3.00 = 26.00
    // getDescription() = "Large BBQ Chicken, Pepperoni, Pepperoni"

Final Pizza 2: cost = $26.00, description = "Large BBQ Chicken, Pepperoni, Pepperoni"
```

Double topping requires no special handling — the same class is simply applied twice in the chain.

---

**Adding pizzas to order O-001:**

```
order = billingSystem.createOrder()
    // orders["O-001"] = Order("O-001", now())

billingSystem.addPizzaToOrder("O-001", pizza1)
    // order.pizzas = [MushroomTopping(ExtraCheeseTopping(MargheritaPizza(MEDIUM)))]

billingSystem.addPizzaToOrder("O-001", pizza2)
    // order.pizzas = [pizza1, PepperoniTopping(PepperoniTopping(BBQChickenPizza(LARGE)))]
```

---

**Generating the bill:**

```
generateBill("O-001")

  Retrieve order O-001  ✓  (not null, not empty)

  Pizza 1:
    pizza1.getCost()        → 16.00
    pizza1.getDescription() → "Medium Margherita, Extra Cheese, Mushrooms"
    lineItems.add(LineItem("Medium Margherita, Extra Cheese, Mushrooms", 16.00))
    subtotal = 16.00

  Pizza 2:
    pizza2.getCost()        → 26.00
    pizza2.getDescription() → "Large BBQ Chicken, Pepperoni, Pepperoni"
    lineItems.add(LineItem("Large BBQ Chicken, Pepperoni, Pepperoni", 26.00))
    subtotal = 42.00

  taxAmount = round(42.00 * 0.10, 2) = 4.20
  total     = 42.00 + 4.20 = 46.20

  return Bill("O-001",
    lineItems = [
      ("Medium Margherita, Extra Cheese, Mushrooms",  $16.00),
      ("Large BBQ Chicken, Pepperoni, Pepperoni",     $26.00)
    ],
    subtotal  = $42.00,
    taxAmount = $4.20,
    total     = $46.20
  )
```

---

**Edge case: empty order**

```
order = billingSystem.createOrder()
// No pizzas added

generateBill(order.getId())
    order.getPizzas().isEmpty() → true
    throw Error("Cannot generate bill for an empty order")   ✓
```

---

**Edge case: unknown order ID**

```
generateBill("NONEXISTENT")
    orders["NONEXISTENT"] == null
    throw Error("Order not found: NONEXISTENT")   ✓
```

---

## Extensibility

### 1. "How would you add a new topping — say, caramelized onions?"

> "Add one class: `CaramelizedOnionTopping extends ToppingDecorator`. Implement `toppingCost()` returning the price and `toppingName()` returning `"Caramelized Onions"`. That is the entire change. `BillingSystem`, `Order`, `generateBill`, and every existing topping class are completely untouched. The Decorator pattern makes this purely additive."

```
class CaramelizedOnionTopping extends ToppingDecorator:
    # toppingCost() -> 1.75
    # toppingName() -> "Caramelized Onions"
```

---

### 2. "How would you add discount codes that apply a percentage or flat reduction?"

Right now there is no discount mechanism. Adding it should not touch the decorator chain or the billing logic — only the final subtotal calculation.

> "I'd introduce a `DiscountStrategy` interface and let `Order` hold an optional discount. `generateBill` applies it after summing the subtotal, before computing tax."

```
interface DiscountStrategy:
    + apply(subtotal: double) -> double    // returns the discount amount

class PercentageDiscount implements DiscountStrategy:
    - percentage: double    // e.g., 20.0 for 20% off
    + apply(subtotal) -> round(subtotal * (percentage / 100), 2)

class FlatDiscount implements DiscountStrategy:
    - amount: double
    + apply(subtotal) -> min(amount, subtotal)   // never exceeds the subtotal

// Order gains one field and one method:
class Order:
    - discount: DiscountStrategy    // null if none

    + applyDiscount(strategy: DiscountStrategy) -> void

// generateBill gains two lines after summing line items:
    discountAmount = 0.0
    if order.getDiscount() != null
        discountAmount = order.getDiscount().apply(subtotal)
        subtotal -= discountAmount
    taxAmount = round(subtotal * TAX_RATE, 2)
    total     = subtotal + taxAmount
```

The existing pizza chain and topping classes are untouched. New discount types are new `DiscountStrategy` implementations.

---

### 3. "How would you add crust types (thin, thick, stuffed) with different prices?"

Crust, like size, is a fundamental property of the pizza — not an optional add-on. It should be a constructor parameter on `BasePizza`, not a decorator layer.

> "I'd add a `CrustType` enum and a `CRUST_SURCHARGE` map to each `BasePizza` class. The surcharge is added to the size-based base price inside `getCost()`. The description is extended with the crust name. Topping decorators are unaffected — they delegate to whatever `wrapped.getCost()` returns, which now automatically includes the crust surcharge."

```
enum CrustType:
    THIN, CLASSIC, STUFFED

    + label() -> String    // "Thin Crust", "Classic", "Stuffed Crust"

class MargheritaPizza implements Pizza:
    - size:  Size
    - crust: CrustType
    - PRICES:          Map<Size, Double>      = {SMALL: 8.00, MEDIUM: 12.00, LARGE: 16.00}
    - CRUST_SURCHARGE: Map<CrustType, Double> = {THIN: 0.00, CLASSIC: 0.00, STUFFED: 2.50}

    + MargheritaPizza(size: Size, crust: CrustType)
    + getCost()        -> PRICES[size] + CRUST_SURCHARGE[crust]
    + getDescription() -> size.label() + " Margherita (" + crust.label() + ")"
```

No topping class changes. A `MushroomTopping(MargheritaPizza(LARGE, STUFFED))` automatically prices the stuffed crust surcharge because the chain delegates to the base's `getCost()`.
