# Worked examples

Each example shows the smell, the named refactoring, and the mechanics step by step. Examples are pseudocode-flavored and language-agnostic — translate the mechanics, not the syntax.

---

## 1. Long Method → Extract Method

**Before**

```
function printOwing(invoice) {
  let outstanding = 0

  print("***********************")
  print("**** Customer Owes ****")
  print("***********************")

  for (const o of invoice.orders) {
    outstanding += o.amount
  }

  const today = clock.today()
  invoice.dueDate = today.plusDays(30)

  print(`name: ${invoice.customer}`)
  print(`amount: ${outstanding}`)
  print(`due: ${invoice.dueDate}`)
}
```

**Mechanics**

1. Extract the banner block → `printBanner()`. Test.
2. Extract the summing loop → `calculateOutstanding(invoice)`, returning the total. Test.
3. Extract the due-date block → `recordDueDate(invoice)`. Test.
4. Extract the final three prints → `printDetails(invoice, outstanding)`. Test.

**After**

```
function printOwing(invoice) {
  printBanner()
  const outstanding = calculateOutstanding(invoice)
  recordDueDate(invoice)
  printDetails(invoice, outstanding)
}
```

**Note:** `printOwing` still mixes computation with I/O. That is a separate smell (Divergent Change) and a separate step — do not fold it into this one.

---

## 2. Switch Statement → Replace Conditional with Polymorphism

**Before**

```
class Bird {
  getSpeed() {
    switch (this.type) {
      case EUROPEAN: return this.baseSpeed()
      case AFRICAN:  return this.baseSpeed() - this.loadFactor() * this.coconuts
      case NORWEGIAN_BLUE: return this.isNailed ? 0 : this.baseSpeed(this.voltage)
    }
    throw new Error("unreachable")
  }
}
```

**Mechanics**

1. Create a subclass per branch: `European`, `African`, `NorwegianBlue`. Test (nothing uses them yet).
2. Add a factory that maps `type` to the right subclass. Route construction through it. Test.
3. Move one branch into its subclass's `getSpeed()` override, remove it from the switch. Test.
4. Repeat for each branch, one at a time, testing between each.
5. When the switch is empty, make the base `getSpeed()` abstract. Test.

**After**

```
class Bird { abstract getSpeed() }
class European extends Bird { getSpeed() { return this.baseSpeed() } }
class African  extends Bird { getSpeed() { return this.baseSpeed() - this.loadFactor() * this.coconuts } }
class NorwegianBlue extends Bird { getSpeed() { return this.isNailed ? 0 : this.baseSpeed(this.voltage) } }
```

**When not to do this:** a switch with two short branches that is never extended is fine. Polymorphism pays off when the same switch shape is repeated across several methods.

---

## 3. Primitive Obsession → Replace Data Value with Object

**Before**

```
function createOrder(customerName, customerEmail, customerPhone) {
  if (!customerEmail.includes("@")) throw new Error("bad email")
  ...
}
```

Validation of the same three primitives is duplicated across every call site.

**Mechanics**

1. Create a `Customer` class holding name, email, phone. Test.
2. Move the email validation into the `Customer` constructor. Test.
3. Change `createOrder` to accept a `Customer`; update callers one at a time. Test between each.
4. Delete the now-unused per-field validation. Test.

**After**

```
function createOrder(customer) { ... }
```

Invalid customers can no longer be constructed, so downstream code stops re-checking.

---

## 4. Feature Envy → Move Method

**Before**

```
class Invoice {
  overdueCharge() {
    return this.account.balance * this.account.overdueRate * this.account.daysOverdue
  }
}
```

`overdueCharge` touches `account` three times and `this` zero times.

**Mechanics**

1. `Account.overdueCharge()` — copy the body, replacing `this.account.` with `this.`. Test.
2. Turn `Invoice.overdueCharge()` into a delegation: `return this.account.overdueCharge()`. Test.
3. Inline the delegation at call sites, one at a time. Test between each.
4. Remove `Invoice.overdueCharge()`. Test.

---

## 5. Message Chain → Hide Delegate

**Before**

```
const manager = person.getDepartment().getManager()
```

Every caller knows that a person has a department and that a department has a manager.

**After**

```
class Person {
  getManager() { return this.department.getManager() }
}
const manager = person.getManager()
```

**Watch out:** doing this repeatedly turns `Person` into a Middle Man. If `Person` accumulates delegating methods and little else, reverse course with Remove Middle Man and let callers talk to `Department` directly.

---

## 6. Duplicate Code across subclasses → Form Template Method

**Before**

`ResidentialSite.getBillableAmount()` and `LifelineSite.getBillableAmount()` each compute base + tax + surcharge, differing only in how base and tax are derived.

**Mechanics**

1. Extract each differing chunk into a method with the *same name* in both subclasses (`getBaseAmount()`, `getTaxAmount()`). Test.
2. Both `getBillableAmount()` bodies are now identical. Pull one up to the superclass. Test.
3. Delete the subclass copies. Test.
4. Declare the varying methods abstract in the superclass. Test.

**After**

```
class Site {
  getBillableAmount() { return this.getBaseAmount() + this.getTaxAmount() }
  abstract getBaseAmount()
  abstract getTaxAmount()
}
```

---

## 7. Characterization test first (no tests exist)

When the code has no coverage, do not refactor yet. Pin current behavior:

1. Call the function with representative inputs.
2. Assert whatever it currently returns — even if it looks wrong.
3. Note the suspicious behavior as a separate bug, do not fix it here.
4. Now refactor. The test fails the moment behavior shifts.

```
test("pins current behavior", () => {
  expect(calculateDiscount(100, "GOLD")).toBe(15)
  expect(calculateDiscount(100, "")).toBe(0)      // probably a bug; pinned, not fixed
  expect(calculateDiscount(-5, "GOLD")).toBe(-0.75) // same
})
```

Fixing the pinned bug is a separate commit, after the refactoring lands.
