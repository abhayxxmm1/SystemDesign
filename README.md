# Design Principles — Java Interview Cheat Sheet
### DRY · KISS · YAGNI · SOLID (SRP, OCP, LSP, ISP, DIP)

## Quick-Reference Table

| Principle | One-liner | Smell (violation) | Fix |
|---|---|---|---|
| **DRY** | Don't repeat the same logic | Same code copy-pasted in 2+ places | Extract to one method/class |
| **KISS** | Simplest solution that works | Overengineered logic for a trivial problem | Simplify, remove unneeded steps |
| **YAGNI** | Build only what's needed *now* | Speculative "future-proofing" features | Defer until actually required |
| **SRP** | One class = one reason to change | "God class" doing everything | Split into focused classes |
| **OCP** | Open for extension, closed for modification | `if/else` or `switch` chain on type | Interface + polymorphism (Strategy) |
| **LSP** | Subtype must be safely substitutable for base type | Override breaks caller's expectations | Don't weaken contracts; prefer composition |
| **ISP** | Don't force a class to depend on methods it doesn't use | One fat interface for many client types | Split into small, role-specific interfaces |
| **DIP** | Depend on abstractions, not concrete classes | High-level class does `new ConcreteImpl()` | Inject an interface (constructor injection) |

---

## 1. DRY — Don't Repeat Yourself
**Definition:** Every piece of logic should have exactly one authoritative place in the codebase.

**Bad:**
```java
int area1 = 10 * 5;
System.out.println("Area1: " + area1);
int area2 = 8 * 4;
System.out.println("Area2: " + area2);
```
**Good:**
```java
class AreaCalculator {
    static int calculateArea(int l, int w) { return l * w; }
}
int area1 = AreaCalculator.calculateArea(10, 5);
int area2 = AreaCalculator.calculateArea(8, 4);
```
**When NOT to DRY it (interview trap!):**
- **Premature abstraction** — two blocks look similar today but may evolve differently (e.g. `AdminReport` vs `UserReport`).
- **Performance-critical hot paths** — a function call/indirection can block inlining.
- **Readability** — two small explicit methods can beat one generic `process(flags...)`.
- **Untested legacy code** — don't merge duplicate logic without tests; "leave it alone unless you must touch it."

---

## 2. KISS — Keep It Simple, Stupid
**Definition:** Prefer the simplest working solution; avoid unnecessary complexity.

**Bad:**
```java
static boolean isEven(int n) {
    boolean result;
    if (n % 2 == 0) result = true; else result = false;
    return result;
}
```
**Good:**
```java
static boolean isEven(int n) { return n % 2 == 0; }
```
**Why it matters:** easier debugging, readability, maintainability, faster delivery.

---

## 3. YAGNI — You Aren't Gonna Need It
**Definition:** Don't implement something until you actually need it — not when you merely foresee needing it.

**Example trap:** Building "note create/view" but pre-adding categories, tags, and cloud sync nobody asked for yet → wasted time, more surface area to maintain.

**When NOT to apply YAGNI (interview trap!):**
- **Requirement is already committed** — e.g. image support is promised in 2 sprints → designing the data model for attachments now can save a rewrite.
- **Performance-critical systems** — preemptively testing real-world load patterns can catch bottlenecks early.

---

## 4. SRP — Single Responsibility Principle
**Definition:** A class should have only **one reason to change**.

**Analogy:** One chef shouldn't cook, clean, serve, *and* order groceries — split by role.

**Bad:** one `TUFplusCompiler` class does driver-code injection + syntax check + test running + DB storage + output formatting.

**Good:**
```java
class DriverCodeGenerator { /* adds driver code */ }
class SyntaxChecker        { /* checks syntax */ }
class TestRunner           { /* runs test cases */ }
class DatabaseManager      { /* stores output */ }
class UserOutputHandler    { /* returns result */ }
class Coordinator          { /* orchestrates the above */ }
```
**Common violations:** mixing DB logic with business logic; embedding business logic in the UI layer.

**Note:** SRP isn't just for classes — applies to methods, modules, microservices, whole systems.

---

## 5. OCP — Open/Closed Principle
**Definition:** Entities should be **open for extension, closed for modification**.

**Analogy:** A travel plug adapter extends your charger's usability without modifying the charger itself.

**Bad (if/else per type):**
```java
class InvoiceProcessor {
    double calculateTotal(String region, double amt) {
        if (region.equalsIgnoreCase("India")) return amt * 1.18;
        else if (region.equalsIgnoreCase("US")) return amt * 1.08;
        else if (region.equalsIgnoreCase("UK")) return amt * 1.12;
        return amt;
    }
}
```
**Good (Strategy + DI):**
```java
interface TaxCalculator { double calculateTax(double amount); }

class IndiaTaxCalculator implements TaxCalculator {
    public double calculateTax(double amt) { return amt * 0.18; }
}
class USTaxCalculator implements TaxCalculator {
    public double calculateTax(double amt) { return amt * 0.08; }
}

class Invoice {
    private double amount;
    private TaxCalculator taxCalculator;
    Invoice(double amount, TaxCalculator taxCalculator) {
        this.amount = amount; this.taxCalculator = taxCalculator;
    }
    double getTotalAmount() { return amount + taxCalculator.calculateTax(amount); }
}
```
Adding Germany → just add `GermanyTaxCalculator implements TaxCalculator`. **Zero changes** to `Invoice`.

**Apply OCP when:** a module evolves often, you need extension without touching tested code, or you're building plugin/framework-style systems.
**Misconception:** OCP ≠ "never change code again" — it means don't modify *stable, tested* core logic; extensions still get tested. Don't apply it preemptively without a real extension need.

---

## 6. LSP — Liskov Substitution Principle
**Definition:** If `S` is a subtype of `T`, objects of `T` must be replaceable with objects of `S` **without breaking correctness**.

**Classic violation — Rectangle/Square:**
```java
class Rectangle {
    int width, height;
    void setWidth(int w)  { width = w; }
    void setHeight(int h) { height = h; }
    int getArea() { return width * height; }
}
class Square extends Rectangle {
    @Override void setWidth(int w)  { width = w;  height = w; }
    @Override void setHeight(int h) { height = h; width = h; }
}
// printArea(new Square()) with setWidth(5); setHeight(10);
// Expected 50, Actual 100 -> LSP violated
```
**Correct extension (Notification example):**
```java
class Notification { void send() { System.out.println("Notification sent"); } }
class EmailNotification extends Notification {
    @Override void send() { System.out.println("Email sent"); }
}
class TextNotification extends Notification {
    @Override void send() { System.out.println("Text sent"); }
}
// Notification n = new EmailNotification(); n.send();  -- works everywhere Notification is expected
```
**Spot an LSP violation — ask:**
- Does the override change meaning/assumptions of the base method?
- Can I swap base → subclass *anywhere* without surprises?
- Does it throw new exceptions or weaken preconditions / strengthen postconditions?

**Fix:** honor the base contract; prefer composition over inheritance; subclasses should **extend**, never **restrict**, behavior.

---

## 7. ISP — Interface Segregation Principle
**Definition:** Don't force a class to depend on methods it doesn't use.

**Analogy:** An Uber rider shouldn't see `acceptRide()`/`trackEarnings()` — that's driver-only.

**Bad (fat interface):**
```java
interface UberUser {
    void bookRide();
    void acceptRide();
    void trackEarnings();
    void ratePassenger();
    void rateDriver();
}
class Rider implements UberUser {
    public void bookRide() { }
    public void acceptRide() { /* not needed! */ }
    public void trackEarnings() { /* not needed! */ }
    public void ratePassenger() { /* not needed! */ }
    public void rateDriver() { }
}
```
**Good (segregated interfaces):**
```java
interface RiderInterface  { void bookRide(); void rateDriver(); }
interface DriverInterface { void acceptRide(); void trackEarnings(); void ratePassenger(); }

class Rider  implements RiderInterface  { public void bookRide() {} public void rateDriver() {} }
class Driver implements DriverInterface { public void acceptRide() {} public void trackEarnings() {} public void ratePassenger() {} }
```
**Apply ISP when:** a class implements unused methods; one interface serves multiple unrelated client types; a new feature forces changes across unrelated classes.

---

## 8. DIP — Dependency Inversion Principle
**Definition:** High-level modules shouldn't depend on low-level modules — **both depend on abstractions**.

**Analogy:** You order food via an app (abstraction), not by calling the chef (low-level implementation) directly.

**Bad (tight coupling):**
```java
class RecentlyAdded {
    void getRecommendations() { System.out.println("Recently added..."); }
}
class RecommendationEngine {
    private RecentlyAdded recommender = new RecentlyAdded(); // hard-coded dependency
    void recommend() { recommender.getRecommendations(); }
}
```
**Good (depend on abstraction):**
```java
interface RecommendationStrategy { void getRecommendations(); }

class RecentlyAdded implements RecommendationStrategy {
    public void getRecommendations() { System.out.println("Recently added..."); }
}
class TrendingNow implements RecommendationStrategy {
    public void getRecommendations() { System.out.println("Trending now..."); }
}

class RecommendationEngine {
    private RecommendationStrategy strategy;
    RecommendationEngine(RecommendationStrategy strategy) { this.strategy = strategy; } // constructor injection
    void recommend() { strategy.getRecommendations(); }
}
// RecommendationEngine engine = new RecommendationEngine(new TrendingNow());
```
Switching strategy at runtime needs **zero changes** to `RecommendationEngine` — just pass a different implementation.

**Benefits:** flexibility (swap impls), testability (mock the interface), reusability, maintainability, scalability.

---

## Interview One-Liners (rapid fire)
- **DRY** → "Single source of truth for logic."
- **KISS** → "Simplicity beats cleverness."
- **YAGNI** → "Don't build it until you need it."
- **SRP** → "One class, one job."
- **OCP** → "Add new code, don't edit old code."
- **LSP** → "A subclass must not surprise the caller."
- **ISP** → "Slim interfaces > fat interfaces."
- **DIP** → "Depend on the interface, not the implementation; inject, don't `new`."

**SOLID acronym order:** **S**RP → **O**CP → **L**SP → **I**SP → **D**IP.