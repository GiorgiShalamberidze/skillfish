---
name: refactoring-specialist
description: Systematic refactoring: code smell identification, refactoring patterns catalog, safe refactoring workflows with tests, IDE automation, and quality measurement.
---

# Refactoring Specialist

Systematic approach to improving code structure without changing external behavior. Provides a catalog of code smells, proven refactoring patterns, safe transformation workflows, IDE automation, and quality measurement. Use when code is hard to understand, painful to change, or accumulating bugs in the same areas.

## Table of Contents

- [Code Smell Catalog](#1-code-smell-catalog)
- [Extract Patterns](#2-extract-patterns)
- [Move and Reorganize](#3-move-and-reorganize)
- [Simplification](#4-simplification)
- [Safe Refactoring Workflow](#5-safe-refactoring-workflow)
- [IDE Automation](#6-ide-automation)
- [Measuring Quality](#7-measuring-quality)
- [Quick Reference](#8-quick-reference)

---

## 1. Code Smell Catalog

Code smells are surface indicators of deeper structural problems. The code works, but the design is fighting future changes.

### Long Method

A method that tries to do too much. When you need a comment to explain a block, that block should be its own method.

**Symptoms:** >20-30 lines, multiple abstraction levels, comments as section headers, hard to name in one phrase.

```python
# BEFORE: 40-line method doing validation, discounts, inventory, payment, notification
def process_order(order):
    # Validate order
    if not order.items:
        raise ValueError("Order must have items")
    for item in order.items:
        if item.quantity <= 0:
            raise ValueError(f"Invalid quantity for {item.name}")
    # Calculate discounts
    discount = 0
    if order.customer.is_premium:
        discount = order.total * 0.1
    if order.promo_code:
        promo = lookup_promo(order.promo_code)
        if promo and promo.is_valid():
            discount += promo.discount_amount
    order.discount = min(discount, order.total)
    # Update inventory, process payment, send confirmation...
    # (another 20 lines of mixed concerns)

# AFTER: Extract methods by intent
def process_order(order):
    validate_order(order)
    apply_discounts(order)
    reserve_inventory(order)
    try:
        charge_payment(order)
    except PaymentError:
        release_inventory(order)
        raise
    send_confirmation(order)
    return order
```

### Large Class

A class accumulating too many responsibilities -- the "god object."

**Symptoms:** >300 lines, unrelated instance variables, method clusters using different field subsets, names like "Manager" or "Handler."

```typescript
// ANTI-PATTERN: One class does auth, profiles, billing, notifications, reporting
class UserManager {
    login(email: string, password: string): Token { /* ... */ }
    updateProfile(userId: string, data: ProfileData): User { /* ... */ }
    addPaymentMethod(userId: string, card: CardInfo): void { /* ... */ }
    sendWelcomeEmail(userId: string): void { /* ... */ }
    getUserActivity(userId: string): ActivityReport { /* ... */ }
}

// FIX: Split into AuthService, ProfileService, BillingService, NotificationService
```

### Feature Envy

A method uses more data from another class than from its own.

```java
// BAD: OrderPrinter reaches into Order and Customer for everything
class OrderPrinter {
    String format(Order order) {
        return String.format("Order #%s\nCustomer: %s %s\nTotal: $%.2f",
            order.getId(), order.getCustomer().getFirstName(),
            order.getCustomer().getLastName(), order.getTotal());
    }
}

// BETTER: Move formatting to where the data lives
class Order {
    String formatSummary() {
        return String.format("Order #%s\nCustomer: %s\nTotal: $%.2f",
            id, customer.getFullName(), getTotal());
    }
}
```

### Data Clumps

Groups of data that repeatedly appear together -- a missing object waiting to be born.

```python
# BAD: Same 3 parameters everywhere
def calculate_distance(x1, y1, z1, x2, y2, z2): ...
def translate(x, y, z, dx, dy, dz): ...

# BETTER: Introduce a Point class
@dataclass
class Point:
    x: float
    y: float
    z: float

    def distance_to(self, other: "Point") -> float:
        return math.sqrt(sum((a - b) ** 2 for a, b in zip(
            (self.x, self.y, self.z), (other.x, other.y, other.z))))
```

### Primitive Obsession

Using strings/ints/bools for domain concepts that deserve their own type, scattering validation everywhere.

```typescript
// BAD: Primitives with implicit rules
function createUser(email: string, phone: string, amount: number) { /* ... */ }

// BETTER: Domain types enforce their own rules
class EmailAddress {
    private constructor(private readonly value: string) {}
    static create(raw: string): EmailAddress {
        if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(raw))
            throw new InvalidEmailError(raw);
        return new EmailAddress(raw.toLowerCase());
    }
}

class Money {
    constructor(readonly amount: number, readonly currency: Currency) {
        if (amount < 0) throw new NegativeAmountError(amount);
    }
    add(other: Money): Money {
        if (this.currency !== other.currency) throw new CurrencyMismatchError();
        return new Money(this.amount + other.amount, this.currency);
    }
}
```

### Shotgun Surgery

A single change requires edits across many files. The opposite of high cohesion.

**Symptoms:** Adding a model field touches 5+ files. A business rule change spans controller, service, repository, serializer, and tests.

**Diagnosis:** Check a recent small bug fix in git -- if it touched >4 files, shotgun surgery is present.

**Fix:** Move related logic into a single module, use Observer/event bus for cross-cutting concerns, or introduce a facade.

---

## 2. Extract Patterns

The most common refactoring family. Each variant pulls logic into a new, named construct.

### Extract Method

```javascript
// BEFORE: Permission check and formatting inline
function renderDashboard(user) {
    if (!user.roles.includes("admin") && !user.roles.includes("analyst")) {
        if (user.department !== "engineering")
            throw new ForbiddenError("Insufficient permissions");
    }
    const data = fetchMetrics();
    const formatted = data.map(d => ({
        label: d.name.replace(/_/g, " ").replace(/\b\w/g, c => c.toUpperCase()),
        value: d.type === "currency" ? `$${(d.value / 100).toFixed(2)}` : d.value.toLocaleString(),
    }));
    return render("dashboard", { metrics: formatted });
}

// AFTER: Each concern is a named function
function renderDashboard(user) {
    assertDashboardAccess(user);
    const formatted = formatMetricsForDisplay(fetchMetrics());
    return render("dashboard", { metrics: formatted });
}
```

### Extract Class

When a subset of fields and methods form a logical group.

```python
# BEFORE: Person holds address fields and logic
class Person:
    def __init__(self, name, street, city, state, zip_code, phone):
        self.name, self.street, self.city = name, street, city
        self.state, self.zip_code, self.phone = state, zip_code, phone
    def get_full_address(self):
        return f"{self.street}\n{self.city}, {self.state} {self.zip_code}"

# AFTER: Address is its own class
@dataclass
class Address:
    street: str
    city: str
    state: str
    zip_code: str
    @property
    def full(self) -> str:
        return f"{self.street}\n{self.city}, {self.state} {self.zip_code}"

class Person:
    def __init__(self, name: str, address: Address, phone: str):
        self.name, self.address, self.phone = name, address, phone
```

### Extract Interface

Decouple consumers from concrete implementations.

```typescript
// Define contract, swap implementations freely
interface UserRepository {
    getUser(id: string): Promise<User>;
    saveUser(user: User): Promise<void>;
}

class PostgresUserRepository implements UserRepository { /* real DB */ }
class InMemoryUserRepository implements UserRepository { /* for tests */ }

class UserService {
    constructor(private readonly repo: UserRepository) {}
    async getUser(id: string): Promise<User> { return this.repo.getUser(id); }
}
```

### Extract Variable

Name a complex expression to make it readable.

```java
// BEFORE
if (order.getTotal() > 500 && customer.getAccountAge() > 365
        && customer.getOrderCount() > 10 && !customer.hasOverduePayments()) {
    applyPremiumDiscount(order);
}

// AFTER
boolean isHighValueOrder = order.getTotal() > 500;
boolean isLoyalCustomer = customer.getAccountAge() > 365 && customer.getOrderCount() > 10;
boolean isInGoodStanding = !customer.hasOverduePayments();

if (isHighValueOrder && isLoyalCustomer && isInGoodStanding) {
    applyPremiumDiscount(order);
}
```

### Extract Parameter Object

Replace a group of parameters that travel together.

```python
# BEFORE: 9 parameters
def search_products(query, min_price, max_price, category, in_stock,
                    sort_by, sort_order, page, page_size): ...

# AFTER: Parameter object
@dataclass
class ProductSearchCriteria:
    query: str
    price_range: PriceRange = field(default_factory=PriceRange)
    category: str | None = None
    in_stock: bool = True
    sort: SortOptions = field(default_factory=SortOptions)
    pagination: Pagination = field(default_factory=Pagination)

def search_products(criteria: ProductSearchCriteria) -> list[Product]: ...
```

---

## 3. Move and Reorganize

Change where code lives without changing what it does.

### Move Method

Relocate a method to the class that owns the data it operates on.

```ruby
# BEFORE: Discount logic on Order, but uses only Customer data
class Order
  def customer_discount
    if customer.premium? && customer.orders_count > 50 then 0.15
    elsif customer.orders_count > 20 then 0.10
    elsif customer.orders_count > 5 then 0.05
    else 0.0
    end
  end
end

# AFTER: Move to Customer
class Customer
  def discount_rate
    if premium? && orders_count > 50 then 0.15
    elsif orders_count > 20 then 0.10
    elsif orders_count > 5 then 0.05
    else 0.0
    end
  end
end
```

### Move Field

When a field is used more by another class, move it there.

```typescript
// BEFORE: discountRate on Order, computed entirely from Customer
class Order {
    discountRate: number;
    constructor(customer: Customer) {
        this.discountRate = customer.isPremium ? 0.15 : 0.05;
    }
}

// AFTER: Field lives on Customer
class Customer {
    get discountRate(): number { return this.isPremium ? 0.15 : 0.05; }
}
```

### Inline Class

Reverse of Extract Class. When a class does too little to justify its existence, absorb it.

**When:** The class has 1-2 trivial methods, used by only one other class, or was created speculatively and never grew.

### Collapse Hierarchy

When parent and child are no longer different enough, merge them. Find all references to the subclass, replace with the parent, delete the subclass.

### Replace Inheritance with Delegation

When a subclass uses only a fraction of the parent's interface, switch from "is-a" to "has-a."

```typescript
// BEFORE: Stack extends ArrayList -- exposes unwanted .sort(), .remove(index), etc.
class Stack<T> extends ArrayList<T> {
    push(item: T): void { this.add(item); }
    pop(): T { return this.remove(this.size() - 1); }
}

// AFTER: Delegation hides the internal list
class Stack<T> {
    private items: T[] = [];
    push(item: T): void { this.items.push(item); }
    pop(): T {
        if (this.isEmpty()) throw new EmptyStackError();
        return this.items.pop()!;
    }
    peek(): T {
        if (this.isEmpty()) throw new EmptyStackError();
        return this.items[this.items.length - 1];
    }
    isEmpty(): boolean { return this.items.length === 0; }
    get size(): number { return this.items.length; }
}
```

---

## 4. Simplification

Reduce conditional complexity and remove magic values.

### Replace Conditional with Polymorphism

When a switch/if-else selects behavior by type, use polymorphic dispatch instead.

```python
# BEFORE: Type-checking conditional
class Shape:
    def area(self):
        if self.shape_type == "rectangle":
            return self.width * self.height
        elif self.shape_type == "circle":
            return math.pi * self.radius ** 2
        elif self.shape_type == "triangle":
            return 0.5 * self.base * self.height

# AFTER: Each shape knows how to compute its own area
class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...

class Rectangle(Shape):
    def __init__(self, width: float, height: float):
        self.width, self.height = width, height
    def area(self) -> float: return self.width * self.height

class Circle(Shape):
    def __init__(self, radius: float):
        self.radius = radius
    def area(self) -> float: return math.pi * self.radius ** 2
```

Adding a new shape = adding a new class, not editing every switch. Open/closed principle in action.

### Decompose Conditional

Extract complex conditions and branches into intention-revealing names.

```javascript
// BEFORE
if (date.getMonth() >= 5 && date.getMonth() <= 8
    && date.getDay() !== 0 && date.getDay() !== 6 && !isHoliday(date)) {
    charge = quantity * 1.25 * PEAK_RATE;
} else {
    charge = quantity * BASE_RATE;
}

// AFTER
if (isPeakBusinessDay(date)) {
    return peakCharge(quantity);
}
return standardCharge(quantity);
```

### Replace Magic Numbers with Named Constants

```typescript
// BEFORE
if (password.length < 8) return "weak";
if (total > 500) return total * 0.85;

// AFTER
const PASSWORD_MIN_LENGTH = 8;
const DISCOUNT_TIER_HIGH_THRESHOLD = 500;
const DISCOUNT_TIER_HIGH_RATE = 0.85;

if (password.length < PASSWORD_MIN_LENGTH) return "weak";
if (total > DISCOUNT_TIER_HIGH_THRESHOLD) return total * DISCOUNT_TIER_HIGH_RATE;
```

### Replace Nested Conditionals with Guard Clauses

```go
// BEFORE: 4 levels of nesting
func processPayment(order *Order) (*Receipt, error) {
    if order != nil {
        if order.IsValid() {
            if order.Customer != nil {
                if order.Customer.HasPaymentMethod() {
                    return chargeCustomer(order)
                } else { return nil, errors.New("no payment method") }
            } else { return nil, errors.New("no customer") }
        } else { return nil, errors.New("invalid order") }
    } else { return nil, errors.New("nil order") }
}

// AFTER: Flat guard clauses
func processPayment(order *Order) (*Receipt, error) {
    if order == nil          { return nil, errors.New("nil order") }
    if !order.IsValid()      { return nil, errors.New("invalid order") }
    if order.Customer == nil { return nil, errors.New("no customer") }
    if !order.Customer.HasPaymentMethod() {
        return nil, errors.New("no payment method")
    }
    return chargeCustomer(order)
}
```

---

## 5. Safe Refactoring Workflow

Refactoring without tests is just editing. This workflow ensures behavior is preserved at every step.

### Step 1: Write Characterization Tests

Capture current behavior before changing anything -- even if the behavior includes bugs.

```python
class TestLegacyPricingEngine:
    """Tests document what the code DOES, not what it SHOULD do."""

    def test_standard_customer_no_discount(self):
        assert PricingEngine().calculate("standard", 50.0) == 50.0

    def test_premium_customer_gets_10_percent(self):
        assert PricingEngine().calculate("premium", 100.0) == 90.0

    def test_negative_total_returns_negative(self):
        # Might be a bug -- document now, fix later as separate change
        assert PricingEngine().calculate("standard", -50.0) == -50.0
```

**Key:** Characterization tests must pass on the current code BEFORE you refactor. If they fail, the test is wrong, not the code.

### Step 2: Refactor in Small Steps

Each step is a single mechanical transformation. Run tests after every step.

```
1. Identify one smell
2. Choose one refactoring
3. Apply the transformation
4. Run all tests -- must pass
5. Commit
6. Repeat
```

**Anti-pattern:** "Big Bang" -- rewrite the entire module, spend hours debugging, give up and revert.

### Step 3: Commit After Each Refactoring

```bash
git commit -m "extract method: pull validation into validate_order()"
git commit -m "move method: relocate discount_rate() to Customer"
git commit -m "extract class: split Address out of Person"
```

### Step 4: Separate Refactoring from Behavior Changes

Never mix refactoring with feature/bugfix commits:

```
PR #1: "Refactor: extract PaymentProcessor from OrderService"  (zero behavior change)
PR #2: "Feature: add Apple Pay support"  (uses new clean structure)
```

### Step 5: Use the Mikado Method for Large Refactorings

Map dependencies and work bottom-up (leaves first):

```
[Replace string IDs with UserId] -- goal
  +-- [Create UserId value object]              -> commit 1
  +-- [Update UserRepository interface]          -> commit 2
  +-- [Update PostgresUserRepository]            -> commit 3
  +-- [Update UserService]                       -> commit 4
  +-- [Update API controllers]                   -> commit 5
  +-- [Update test factories]                    -> commit 6
  +-- [Remove old string ID paths]               -> commit 7
```

---

## 6. IDE Automation

Modern IDEs perform refactorings at the AST level -- safer and faster than manual text editing.

### VS Code

| Action | Mac | Win/Linux |
|--------|-----|-----------|
| Rename symbol | `F2` | `F2` |
| Extract to method | `Cmd+Shift+R` | `Ctrl+Shift+R` |
| Extract to variable | Select, `Cmd+.` | Select, `Ctrl+.` |
| Inline variable | `Cmd+.` on var | `Ctrl+.` on var |
| Move to new file | `Cmd+.` on symbol | `Ctrl+.` on symbol |
| Organize imports | `Option+Shift+O` | `Alt+Shift+O` |
| Quick fix menu | `Cmd+.` | `Ctrl+.` |

**Key extensions:** Abracadabra (JS/TS refactorings), SonarLint (smell detection), Ruff (Python auto-fix).

### JetBrains (IntelliJ, WebStorm, PyCharm)

| Action | Mac | Win/Linux |
|--------|-----|-----------|
| Rename | `Shift+F6` | `Shift+F6` |
| Extract method | `Cmd+Option+M` | `Ctrl+Alt+M` |
| Extract variable | `Cmd+Option+V` | `Ctrl+Alt+V` |
| Extract constant | `Cmd+Option+C` | `Ctrl+Alt+C` |
| Extract parameter | `Cmd+Option+P` | `Ctrl+Alt+P` |
| Inline | `Cmd+Option+N` | `Ctrl+Alt+N` |
| Move class/file | `F6` | `F6` |
| Change signature | `Cmd+F6` | `Ctrl+F6` |
| Safe delete | `Cmd+Delete` | `Alt+Delete` |
| Refactor this menu | `Ctrl+T` | `Ctrl+Alt+Shift+T` |

### Codemods (Large-Scale Transforms)

For codebase-wide refactorings, use AST-based transform tools:

```javascript
// jscodeshift: rename "color" prop to "variant" on Button across entire codebase
export default function transformer(file, api) {
    const j = api.jscodeshift;
    return j(file.source)
        .find(j.JSXAttribute, { name: { name: "color" } })
        .forEach(path => {
            if (path.parentPath.parentPath.value.name?.name === "Button")
                path.value.name.name = "variant";
        })
        .toSource();
}
// Run: npx jscodeshift -t rename-prop.js src/ --extensions=tsx,jsx
```

| Language | Tool | Use Case |
|----------|------|----------|
| JavaScript/TypeScript | jscodeshift | Rename props, migrate APIs, convert patterns |
| Python | libcst / bowler | Rename functions, update call signatures |
| Go | gorename / gopls | Rename identifiers across packages |
| Java | OpenRewrite | Recipe-based large-scale refactoring |
| Rust | rust-analyzer | IDE-integrated refactorings |
| C# | Roslyn analyzers | Custom code fixes |

---

## 7. Measuring Quality

Track metrics before and after to validate that refactoring produced real improvement.

### Cyclomatic Complexity

Counts independent paths through a function. Each `if`, `for`, `while`, `case`, `&&`, `||` adds 1.

| Score | Risk | Action |
|-------|------|--------|
| 1-5 | Low | Simple, well-tested |
| 6-10 | Moderate | Consider simplifying |
| 11-20 | High | Refactor -- hard to test all paths |
| 21+ | Critical | Break apart immediately |

**Tools:** `eslint` complexity rule (JS), `radon cc` (Python), PMD/SonarQube (Java), `gocyclo` (Go).

### Cognitive Complexity

Measures how hard code is to _understand_. Nesting increases the score more than flat structures.

```python
# Same cyclomatic complexity (4), very different cognitive complexity

# LOW cognitive complexity (flat guards)
def calculate_price(item):
    if item.is_on_sale:       return item.sale_price    # +1
    if item.has_coupon:       return item.price - item.coupon_value  # +1
    if item.bulk_qty > 10:    return item.price * 0.9   # +1
    return item.price

# HIGH cognitive complexity (nested)
def calculate_price(item):
    if item.is_on_sale:                         # +1
        if item.sale_type == "clearance":       # +2 (nesting)
            if item.category == "electronics":  # +3 (nesting)
                return item.price * 0.5
```

**Target:** Keep cognitive complexity below 15 per function.

### Coupling Metrics

| Metric | What It Measures | Healthy | Warning |
|--------|------------------|---------|---------|
| Afferent (Ca) | Modules depending ON this one | < 20 | > 50 = god module |
| Efferent (Ce) | Modules this one depends ON | < 15 | > 30 = too fragile |
| Instability (I) | Ce / (Ca + Ce) | 0-1 | Stable modules need low I |

**Tools:** `madge --circular` (JS/TS), `pydeps` / `import-linter` (Python), JDepend (Java), `dependency-cruiser` (JS/TS).

### Code Churn

Files with high churn AND high complexity are the best refactoring targets.

```bash
# Most-changed files in 6 months
git log --since="6 months ago" --pretty=format: --name-only | \
    sort | uniq -c | sort -rn | head -20
```

### Before/After Comparison Template

```markdown
## Refactoring Report: [Module Name]

| Metric | Before | After |
|--------|--------|-------|
| Files | 1 monolith | 5 focused modules |
| Lines | 450 | 290 (-36%) |
| Max cyclomatic | 23 | 6 |
| Max cognitive | 31 | 5 |
| Circular deps | 3 | 0 |
| Test coverage | 45% | 92% |
```

---

## 8. Quick Reference

### Code Smell to Refactoring Map

| Code Smell | Primary Refactoring | Secondary Options |
|------------|---------------------|-------------------|
| Long Method | Extract Method | Decompose Conditional |
| Large Class | Extract Class | Extract Interface, Move Method |
| Feature Envy | Move Method | Move Field |
| Data Clumps | Extract Parameter Object | Extract Class |
| Primitive Obsession | Replace Primitive with Object | Extract Class |
| Shotgun Surgery | Move Method/Field | Inline Class (consolidate) |
| Divergent Change | Extract Class | Split by responsibility |
| Parallel Inheritance | Move Method/Field | Collapse Hierarchy |
| Speculative Generality | Inline Class | Collapse Hierarchy |
| Dead Code | Delete it | Safe Delete (IDE) |
| Middle Man | Inline Class | Remove delegation |
| Inappropriate Intimacy | Move Method/Field | Replace Inheritance with Delegation |
| Long Parameter List | Extract Parameter Object | Preserve Whole Object |
| Switch Statements | Replace Conditional with Polymorphism | Strategy/Map |
| Duplicated Code | Extract Method | Extract Superclass |
| Comments (excessive) | Extract Method | Rename (self-documenting code) |

### Refactoring Decision Checklist

- [ ] Do I have tests covering the code I am about to change?
- [ ] Can I describe the smell in one sentence?
- [ ] Do I know which specific refactoring I am applying?
- [ ] Is this separate from any feature or bugfix work?
- [ ] Can I complete this step in under 30 minutes?
- [ ] Will I commit immediately after tests pass?

### When NOT to Refactor

| Situation | Reason | Alternative |
|-----------|--------|-------------|
| No tests and cannot add them | Too dangerous | Write characterization tests first |
| Code is being replaced soon | Wasted effort | Document the plan, leave it |
| On a deadline for unrelated work | Context switching cost | Log as tech debt, schedule later |
| Code works and is never touched | No benefit | Leave it alone |
| Matching personal style preference | Not a smell | Follow existing conventions |
| You do not understand the code yet | Premature | Read, test, THEN refactor |

### Refactoring Workflow Summary

```
Identify smell
      |
      v
Have tests? --NO--> Write characterization tests
      |                        |
     YES                       |
      |  <---------------------+
      v
Pick ONE refactoring
      |
      v
Apply transformation
      |
      v
Run tests --FAIL--> Undo, try smaller step
      |
    PASS
      |
      v
  Commit
      |
      v
More smells? --YES--> (loop back)
      |
      NO --> Done
```
