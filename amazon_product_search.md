# Amazon Product Search Tool

A product search tool similar to Amazon's search functionality. Users type a keyword and optionally layer on filters — Prime eligibility, price range, category, minimum rating — and can ask for results sorted by price or rating. The system narrows a product catalog to the matching subset and returns it in the requested order.

---

## Requirements

You walk into the interview and get greeted with this prompt:

> "Design a product searching tool similar to Amazon's search functionality that allows users to filter and find products based on criteria such as Prime eligibility, price range, and category."

### Clarifying Questions

**You:** "What attributes can users filter on?"
**Interviewer:** "Prime eligibility, price range, and category. Minimum rating would be good to include too."

**You:** "For price range — do users specify a minimum, a maximum, or both?"
**Interviewer:** "Either or both. A user might only want a ceiling, or only a floor."

**You:** "Is there a keyword or text search in addition to filters?"
**Interviewer:** "Yes — users type a search term. It should match against the product name and description."

**You:** "Can users apply multiple filters at once? Like 'Prime, Electronics, under $50'?"
**Interviewer:** "Yes. All active filters apply together — a product must satisfy all of them."

**You:** "Should results be sortable? Amazon shows options like price low-to-high, highest rated."
**Interviewer:** "Yes. Support sorting by price ascending, price descending, and rating descending."

**You:** "What does the result look like — just IDs, or full product data?"
**Interviewer:** "Return full product objects."

**You:** "Should we handle pagination?"
**Interviewer:** "Leave that out for now."

**You:** "Is the product catalog stored in a real database or in memory?"
**Interviewer:** "In-memory. Assume the catalog is loaded at startup."

**You:** "Concurrency?"
**Interviewer:** "Not for now."

### Final Requirements

| # | Requirement |
|---|-------------|
| 1 | Users supply an optional keyword; the system matches it case-insensitively against product name and description |
| 2 | Filter: Prime eligibility (isPrime == true) |
| 3 | Filter: minimum price (price >= floor) and/or maximum price (price <= ceiling) — each optional and independent |
| 4 | Filter: category (exact match, case-insensitive) |
| 5 | Filter: minimum rating (rating >= threshold) |
| 6 | Multiple filters apply with AND logic — a product must satisfy every active filter |
| 7 | Results can be sorted by: price ascending, price descending, or rating descending |
| 8 | Returns a list of matching Product objects |
| 9 | Product catalog is loaded in-memory at startup |

**Out of scope:** pagination, real database, OR/NOT filter logic, relevance ranking, user authentication, inventory or availability tracking.

---

## Core Entities and Relationships

The central challenge here is **dual pluggability — filtering and sorting are two independent axes**. Unlike a simple attribute lookup, product search must both NARROW results (zero or more filters that all must pass) and ORDER them (one sort strategy). These two concerns are fully independent: any filter combination should work with any sort order. A single Strategy pattern handles one axis; we need it applied twice. Hard-coding either axis — `if filterType == "prime"` or `if sortType == "price"` — breaks every time a new filter or sort order is added. Keeping them as separate pluggable interfaces means adding a new filter never touches the sort code, and vice versa.

**ProductSearchAPI** — The orchestrator and public entry point. Holds the in-memory product catalog keyed by product ID. Accepts a `SearchRequest`, applies keyword matching and each filter to every product, sorts the passing set with the requested sort strategy, and returns a `SearchResult`.

**Product** — The unit the system operates on. Carries all searchable and filterable attributes: name, description, price, category, Prime eligibility, and rating.

**SearchRequest** — A value object bundling the full search intent: an optional keyword, a list of filters, and an optional sort strategy. Passing it as a single object keeps the `search()` signature stable as new search parameters are added.

**SearchResult** — A value object wrapping the list of matching products. Thin now, but gives a natural place to add total count, facet data, or page info without changing the `search()` signature.

**Filter (interface)** — The first strategy abstraction. One method: `matches(product) -> boolean`. Every attribute filter implements this. Adding a new filter type means implementing this interface — `ProductSearchAPI` never changes.

**PrimeFilter** — Passes products where isPrime is true. No parameters needed.

**MinPriceFilter** — Passes products whose price is at or above a given floor.

**MaxPriceFilter** — Passes products whose price is at or below a given ceiling. Separate from `MinPriceFilter` so the caller includes only what they need — a user setting only a ceiling doesn't pay for floor logic.

**CategoryFilter** — Passes products whose category matches a given string (case-insensitive).

**MinRatingFilter** — Passes products whose rating meets or exceeds a given threshold.

**SortStrategy (interface)** — The second strategy abstraction. One method: `compare(a, b) -> int` (negative if a sorts before b, positive if b sorts before a, zero if equal). Every sort order implements this. `ProductSearchAPI` calls `results.sort(sortStrategy::compare)` without knowing which sort is in use.

**PriceAscSorter** — Sorts by price, lowest first.

**PriceDescSorter** — Sorts by price, highest first.

**RatingDescSorter** — Sorts by rating, highest first.

### Entity Responsibilities

| Entity | Responsibility |
|--------|----------------|
| `ProductSearchAPI` | Entry point. Holds catalog. Applies keyword match, filters, sort, and wraps result. |
| `Product` | Stores all searchable attributes. The unit filters and sorters operate on. |
| `SearchRequest` | Bundles search intent: keyword + filters + sort. Keeps `search()` signature stable. |
| `SearchResult` | Wraps the list of matching products. Extension point for count, facets, pagination. |
| `Filter` | Interface. Single predicate: does this product match? |
| `PrimeFilter` | Concrete filter: isPrime == true. |
| `MinPriceFilter` | Concrete filter: price >= floor. |
| `MaxPriceFilter` | Concrete filter: price <= ceiling. |
| `CategoryFilter` | Concrete filter: category matches (case-insensitive). |
| `MinRatingFilter` | Concrete filter: rating >= threshold. |
| `SortStrategy` | Interface. Orders two products relative to each other. |
| `PriceAscSorter` | Concrete sorter: ascending price. |
| `PriceDescSorter` | Concrete sorter: descending price. |
| `RatingDescSorter` | Concrete sorter: descending rating. |

### Relationships

```
ProductSearchAPI
  ├── products: Map<id, Product>
  └── search(SearchRequest) -> SearchResult

SearchRequest
  ├── keyword: String (optional)
  ├── filters: List<Filter>
  └── sortStrategy: SortStrategy (optional)

SearchResult
  └── products: List<Product>

Product
  ├── id, name, description
  ├── price: double
  ├── category: String
  ├── isPrime: boolean
  └── rating: double

<<interface>> Filter                    <<interface>> SortStrategy
  + matches(product) -> boolean           + compare(a, b) -> int
      ^      ^      ^      ^      ^            ^          ^          ^
      |      |      |      |      |            |          |          |
  Prime  MinPrice MaxPrice Category MinRating  PriceAsc  PriceDesc  RatingDesc
  Filter Filter   Filter  Filter   Filter      Sorter    Sorter     Sorter
```

`Filter` and `SortStrategy` are parallel strategy hierarchies with no dependency on each other. `ProductSearchAPI` composes them: apply all filters (AND), then apply the sort. New filters never affect sorters and vice versa.

---

## Class Design

Top-down: `ProductSearchAPI` first, then `Product`, `SearchRequest`, `SearchResult`, then the `Filter` interface and each concrete filter, then `SortStrategy` and each concrete sorter.

### ProductSearchAPI

**Deriving State**

| Requirement | What ProductSearchAPI must track |
|-------------|----------------------------------|
| "Catalog loaded at startup, searched by keyword and filters" | All products — iterable for search, O(1) lookup by ID for `addProduct` |

**Deriving Behavior**

| Need from requirements | Method on ProductSearchAPI |
|------------------------|---------------------------|
| "Search by keyword, filters, sort" | `search(request: SearchRequest) -> SearchResult` |
| "Load catalog at startup" | `addProduct(product) -> void` |

```
class ProductSearchAPI:
    - products: Map<String, Product>

    + ProductSearchAPI()
    + addProduct(product) -> void
    + search(request: SearchRequest) -> SearchResult
    - matchesKeyword(product, keyword) -> boolean    // internal
    - matchesAllFilters(product, filters) -> boolean // internal
```

---

### Product

**Deriving State**

| Requirement | What Product must track |
|-------------|------------------------|
| "Keyword matches name or description" | name, description |
| "Prime filter" | isPrime |
| "Price range filters" | price |
| "Category filter" | category |
| "Minimum rating filter / sort by rating" | rating |
| "Sort by price" | price (already above) |
| "Return full product objects" | id (for identity) |

```
class Product:
    - id:          String
    - name:        String
    - description: String
    - price:       double
    - category:    String
    - isPrime:     boolean
    - rating:      double    // 0.0 to 5.0

    + Product(id, name, description, price, category, isPrime, rating)
    + getId()          -> String
    + getName()        -> String
    + getDescription() -> String
    + getPrice()       -> double
    + getCategory()    -> String
    + isPrime()        -> boolean
    + getRating()      -> double
```

---

### SearchRequest

**Deriving State**

| Requirement | What SearchRequest must carry |
|-------------|-------------------------------|
| "Optional keyword matched against name/description" | keyword (null or empty = no keyword filter) |
| "Zero or more attribute filters, AND logic" | filters: List<Filter> |
| "Optional sort by price or rating" | sortStrategy (null = preserve insertion order) |

```
class SearchRequest:
    - keyword:      String              // null or empty means no keyword constraint
    - filters:      List<Filter>
    - sortStrategy: SortStrategy        // null means no sorting

    + SearchRequest(keyword, filters, sortStrategy)
    + getKeyword()       -> String
    + getFilters()       -> List<Filter>
    + getSortStrategy()  -> SortStrategy
```

`SearchRequest` is the single place to add new search parameters (date, discount, etc.) without changing the `search()` signature.

---

### SearchResult

```
class SearchResult:
    - products: List<Product>

    + SearchResult(products: List<Product>)
    + getProducts() -> List<Product>
    + getCount()    -> int
```

Thin now. `getCount()` is already useful; adding pagination fields or facet counts later requires only changes here, not in `ProductSearchAPI`.

---

### Filter (interface)

The strategy abstraction for filtering. Every attribute filter implements exactly this.

```
interface Filter:
    + matches(product: Product) -> boolean
```

---

### PrimeFilter

```
class PrimeFilter implements Filter:
    + PrimeFilter()
    + matches(product) -> boolean    // product.isPrime()
```

No stored state — Prime eligibility is a boolean attribute on the product, so the filter is stateless.

---

### MinPriceFilter

```
class MinPriceFilter implements Filter:
    - floor: double

    + MinPriceFilter(floor: double)
    + matches(product) -> boolean    // product.getPrice() >= floor
```

---

### MaxPriceFilter

```
class MaxPriceFilter implements Filter:
    - ceiling: double

    + MaxPriceFilter(ceiling: double)
    + matches(product) -> boolean    // product.getPrice() <= ceiling
```

`MinPriceFilter` and `MaxPriceFilter` are separate classes so a user can pass only one without any null-checking inside a combined filter. A user wanting only a price ceiling creates `MaxPriceFilter(50)` and adds it to the list. That's it — no optional field to reason about.

---

### CategoryFilter

```
class CategoryFilter implements Filter:
    - category: String    // stored lowercase

    + CategoryFilter(category: String)    // lowercases at construction
    + matches(product) -> boolean         // product.getCategory().equalsIgnoreCase(this.category)
```

Normalizing to lowercase at construction means `matches()` is a simple equality check with no per-product string transformation.

---

### MinRatingFilter

```
class MinRatingFilter implements Filter:
    - minRating: double

    + MinRatingFilter(minRating: double)
    + matches(product) -> boolean    // product.getRating() >= minRating
```

---

### SortStrategy (interface)

The strategy abstraction for ordering. `ProductSearchAPI` calls `results.sort(sortStrategy::compare)` without knowing the sort order.

```
interface SortStrategy:
    + compare(a: Product, b: Product) -> int
        // negative  → a sorts before b
        // positive  → b sorts before a
        // zero      → equal
```

---

### PriceAscSorter

```
class PriceAscSorter implements SortStrategy:
    + compare(a, b) -> int
        return Double.compare(a.getPrice(), b.getPrice())
```

---

### PriceDescSorter

```
class PriceDescSorter implements SortStrategy:
    + compare(a, b) -> int
        return Double.compare(b.getPrice(), a.getPrice())
```

---

### RatingDescSorter

```
class RatingDescSorter implements SortStrategy:
    + compare(a, b) -> int
        return Double.compare(b.getRating(), a.getRating())
```

---

### Final Class Design

```
+------------------------------------------+
|            ProductSearchAPI              |
+------------------------------------------+
| - products: Map<String, Product>         |
+------------------------------------------+
| + addProduct(product)                    |
| + search(request) -> SearchResult        |
| - matchesKeyword(product, keyword)       |
| - matchesAllFilters(product, filters)    |
+------------------------------------------+
          |                    |
    uses  |              uses  |
          v                    v
+---------------------+   +----------------------------+
|    SearchRequest    |   |        SearchResult        |
+---------------------+   +----------------------------+
| - keyword: String   |   | - products: List<Product>  |
| - filters: List     |   +----------------------------+
| - sortStrategy      |   | + getProducts()            |
+---------------------+   | + getCount()               |
                           +----------------------------+

+------------------------------------------+
|                 Product                  |
+------------------------------------------+
| - id:          String                    |
| - name:        String                    |
| - description: String                    |
| - price:       double                    |
| - category:    String                    |
| - isPrime:     boolean                   |
| - rating:      double                    |
+------------------------------------------+

     <<interface>>                      <<interface>>
        Filter                          SortStrategy
+ matches(product) -> bool          + compare(a, b) -> int
    ^    ^     ^      ^    ^             ^       ^        ^
    |    |     |      |    |             |       |        |
 Prime MinPrice MaxPrice Category MinRating PriceAsc PriceDesc RatingDesc
 Filter Filter  Filter   Filter   Filter    Sorter   Sorter    Sorter
```

---

## Implementation

The three most interesting methods are `ProductSearchAPI.search()` (the main flow), `matchesKeyword()` (how keyword search works), and the two internal helpers. We'll also show `PriceAscSorter.compare()` and `CategoryFilter.matches()` to make the pattern concrete.

### search()

**Core logic:**
1. Iterate every product in the catalog
2. Skip products that don't match the keyword (if one is provided)
3. Skip products that fail any filter
4. Collect passing products into a result list
5. If a sort strategy is provided, sort the list
6. Return a `SearchResult` wrapping the list

**Edge cases:**
- Null or empty keyword — treat as no keyword constraint; all products pass the keyword check
- Empty filter list — no filter rejects anything; only keyword check applies
- Null sort strategy — return results in catalog insertion order
- No products in catalog — return empty SearchResult immediately

```
search(request)
    results = []

    for product in products.values()
        if !matchesKeyword(product, request.getKeyword())
            continue
        if !matchesAllFilters(product, request.getFilters())
            continue
        results.add(product)

    if request.getSortStrategy() != null
        results.sort(request.getSortStrategy()::compare)

    return SearchResult(results)
```

The two-phase structure (filter first, sort after) is intentional. Sorting is O(n log n) — apply it to the smallest possible set by filtering first. Sorting 5 results is trivially faster than sorting 50,000 products and then discarding 49,995.

---

### matchesKeyword()

**Core logic:**
1. If keyword is null or blank, every product passes — return true immediately
2. Lowercase both the keyword and the product fields once before comparing
3. Return true if name or description contains the keyword

**Edge cases:**
- Keyword is whitespace only — treat as empty, return true
- Product has a null description — treat as empty string

```
matchesKeyword(product, keyword)
    if keyword == null || keyword.trim().isEmpty()
        return true

    lowerKeyword = keyword.toLowerCase()
    name        = product.getName().toLowerCase()
    description = product.getDescription() != null
                    ? product.getDescription().toLowerCase()
                    : ""

    return name.contains(lowerKeyword) || description.contains(lowerKeyword)
```

---

### matchesAllFilters()

**Core logic:**
- For each filter in the list, call `matches(product)`; return false immediately on first failure

**Edge cases:**
- Empty filter list — no iteration, returns true (all products pass)

```
matchesAllFilters(product, filters)
    for filter in filters
        if !filter.matches(product)
            return false    // short-circuit on first failure
    return true
```

Short-circuiting is deliberate. Filters are applied in the order the caller provides them. A caller who knows most products fail the Prime filter can put `PrimeFilter` first to cut the field quickly before evaluating costlier checks.

---

### CategoryFilter.matches()

```
CategoryFilter(category)
    this.category = category.toLowerCase()    // normalize once

matches(product)
    return product.getCategory().toLowerCase() == this.category
```

---

### RatingDescSorter.compare()

```
compare(a, b)
    return Double.compare(b.getRating(), a.getRating())    // b before a → descending
```

`Double.compare` handles floating-point edge cases (NaN, -0.0) correctly. A naïve `b.rating - a.rating` cast to int loses precision for ratings like 4.8 vs 4.9 that differ by less than 1.

---

## Verification

**Catalog setup:**

| ID | Name | Category | Price | Prime | Rating |
|----|------|----------|-------|-------|--------|
| P1 | iPhone 15 | Electronics | $999 | ✓ | 4.8 |
| P2 | USB Cable | Electronics | $9.99 | ✓ | 4.2 |
| P3 | Harry Potter | Books | $15 | ✗ | 4.9 |
| P4 | Wireless Headphones | Electronics | $79 | ✓ | 4.5 |
| P5 | Notebook | Books | $5 | ✓ | 3.8 |

---

**Search 1: keyword "wireless", no filters, sort by price descending**

```
request = SearchRequest("wireless", [], PriceDescSorter)

Keyword check:
  P1 "iPhone 15"           → name contains "wireless"? ✗ → skip
  P2 "USB Cable"           → name contains "wireless"? ✗ → skip
  P3 "Harry Potter"        → name contains "wireless"? ✗ → skip
  P4 "Wireless Headphones" → name contains "wireless"? ✓ → pass (case-insensitive)
  P5 "Notebook"            → name contains "wireless"? ✗ → skip

Filter check: no filters → all passing products stay
  P4 → add to results

Sort (PriceDescSorter): only one result, no reordering needed

SearchResult: [P4 "Wireless Headphones" ($79)]  ✓
```

---

**Search 2: keyword empty, filters=[PrimeFilter, CategoryFilter("electronics"), MinPriceFilter($50)], sort by rating descending**

```
request = SearchRequest("", [PrimeFilter, CategoryFilter("electronics"), MinPriceFilter(50)], RatingDescSorter)

Keyword: empty → all products pass keyword check

Filter check:
  P1: isPrime ✓ | category "Electronics" ✓ | price $999 >= $50 ✓ → add
  P2: isPrime ✓ | category "Electronics" ✓ | price $9.99 >= $50  ✗ → skip (short-circuit at MinPriceFilter)
  P3: isPrime ✗ → skip (short-circuit at PrimeFilter)
  P4: isPrime ✓ | category "Electronics" ✓ | price $79 >= $50   ✓ → add
  P5: isPrime ✓ | category "Books" ✗       → skip (short-circuit at CategoryFilter)

Results before sort: [P1, P4]

Sort by rating descending:
  P1 (4.8) vs P4 (4.5) → P1 first

SearchResult: [P1 "iPhone 15" (4.8), P4 "Wireless Headphones" (4.5)]  ✓
```

---

**Search 3: keyword empty, filters=[MinRatingFilter(4.5)], no sort**

```
request = SearchRequest("", [MinRatingFilter(4.5)], null)

Filter check (rating >= 4.5):
  P1: 4.8 >= 4.5 ✓ → add
  P2: 4.2 >= 4.5 ✗ → skip
  P3: 4.9 >= 4.5 ✓ → add
  P4: 4.5 >= 4.5 ✓ → add
  P5: 3.8 >= 4.5 ✗ → skip

No sort → preserve catalog insertion order

SearchResult: [P1, P3, P4]  ✓
```

---

**Search 4: no keyword, no filters, no sort — returns entire catalog**

```
request = SearchRequest("", [], null)

All products pass keyword check (empty keyword)
All products pass filter check (empty filters list)
No sort

SearchResult: [P1, P2, P3, P4, P5]  ✓
```

---

## Extensibility

### 1. "How would you add a new filter — say, minimum review count?"

> "I'd implement `Filter` with a `MinReviewCountFilter` class. It stores a `minCount: int` and `matches(product)` returns `product.getReviewCount() >= minCount`. `Product` would need a `reviewCount` field added. That's it — `ProductSearchAPI`, `search()`, every existing filter, and all sorters are completely untouched. The caller adds the new filter to the request's filter list. This is the Strategy pattern working as intended: adding a filter is purely additive."

```
class MinReviewCountFilter implements Filter:
    - minCount: int

    + MinReviewCountFilter(minCount)
    + matches(product) -> boolean
        return product.getReviewCount() >= minCount
```

---

### 2. "How would you add a new sort order — say, newest arrivals first?"

> "I'd implement `SortStrategy` with a `NewestArrivalsSorter`. It compares products by `listedAt` timestamp descending. `Product` would need a `listedAt: DateTime` field. Nothing else changes — no filter code is touched, `ProductSearchAPI.search()` already calls `results.sort(sortStrategy::compare)` generically. The caller passes `NewestArrivalsSorter` in the request. Just like adding a filter, adding a sort order is purely additive."

```
class NewestArrivalsSorter implements SortStrategy:
    + compare(a, b) -> int
        return b.getListedAt().compareTo(a.getListedAt())    // descending
```

---

### 3. "How would you support OR filter logic — 'Electronics OR Books'?"

Right now all filters compose with AND — a product must satisfy every filter. OR requires a different combinator.

> "I'd add an `OrFilter` composite that implements `Filter` and wraps a list of child filters, returning true if any one of them passes. For the 'Electronics OR Books' case, the caller creates `OrFilter([CategoryFilter('Electronics'), CategoryFilter('Books')])` and adds that single `OrFilter` to the request's filter list. Nesting works naturally: an `AndFilter` can sit inside an `OrFilter` and vice versa, because all nodes are just `Filter` implementations. `ProductSearchAPI.search()` never needs to know whether a filter is a leaf or a composite."

```
class OrFilter implements Filter:
    - filters: List<Filter>

    + OrFilter(filters: List<Filter>)
    + matches(product) -> boolean
        for filter in filters
            if filter.matches(product)
                return true     // short-circuit on first match
        return false
```
