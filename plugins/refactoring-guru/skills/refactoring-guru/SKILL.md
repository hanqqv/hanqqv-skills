---
name: "refactoring-guru"
description: "Refactor code using the Refactoring Guru / Fowler catalog: detect code smells, apply the matching named refactoring in small behavior-preserving steps, verify after each. Use when asked to refactor, clean up, restructure, or review code for smells."
---

# Refactoring (Refactoring Guru method)

Refactoring is **restructuring existing code without changing its external behavior**. If behavior changes, it is not a refactoring — it is a rewrite or a bug. Hold that line.

## Core discipline

1. **Green before you start.** Confirm tests exist and pass. If there is no test covering the code, write a characterization test first (capture current behavior, even if that behavior is wrong). Never refactor untested code blind — say so and offer to add the test.
2. **One named refactoring at a time.** Each step is a single technique from the catalog below, small enough to name in one sentence.
3. **Verify after every step.** Run the tests. Green → continue. Red → revert that step, do not "fix forward".
4. **Never mix refactoring with behavior change.** No bug fixes, no new features, no "while I'm here" tweaks in the same commit. Separate commits, separate steps.
5. **Stop when the smell is gone.** Refactor toward a concrete complaint, not toward an abstract ideal. Over-abstraction is itself a smell (Speculative Generality).

## Working procedure

- **Read first.** Understand what the code does before judging it. Note the public surface (the contract that must not change).
- **Inventory smells.** List what you find, each with file:line and the smell's name.
- **Rank by payoff, not by offense.** Prioritize: (a) code being actively changed, (b) smells causing real bugs or slow changes, (c) everything else. Dead corners of the codebase rarely repay the risk.
- **Propose the plan before editing.** Show the ordered list of refactorings. Get agreement on anything that touches a public API, a shared module, or more than a few files.
- **Execute in small steps**, verifying between each.
- **Report** what changed, what was left alone and why, and any behavior risk you could not fully test.

## Smell → refactoring map

### Bloaters
| Smell | Signal | Apply |
|---|---|---|
| Long Method | Method does several things; needs comments to explain sections | Extract Method; Replace Temp with Query; Introduce Parameter Object; Decompose Conditional; Replace Method with Method Object |
| Large Class | Too many fields/methods; several responsibilities | Extract Class; Extract Subclass; Extract Interface; Extract Delegate |
| Primitive Obsession | Strings/ints standing in for concepts; type codes | Replace Data Value with Object; Replace Type Code with Class/Subclasses/State-Strategy; Introduce Parameter Object; Replace Array with Object |
| Long Parameter List | 4+ parameters, or flags | Replace Parameter with Method Call; Preserve Whole Object; Introduce Parameter Object; Remove Flag Argument |
| Data Clumps | Same group of values travels together | Extract Class; Introduce Parameter Object; Preserve Whole Object |

### Object-orientation abusers
| Smell | Signal | Apply |
|---|---|---|
| Switch Statements | switch/if-chain on a type or state | Replace Conditional with Polymorphism; Replace Type Code with Subclasses/State/Strategy; Introduce Null Object |
| Temporary Field | Field only set in some circumstances | Extract Class; Introduce Null Object; Replace Method with Method Object |
| Refused Bequest | Subclass ignores or throws on inherited members | Replace Inheritance with Delegation; Extract Superclass; Push Down Method/Field |
| Alternative Classes with Different Interfaces | Two classes do the same job, different names | Rename Method; Move Method; Extract Superclass; Unify Interfaces |

### Change preventers
| Smell | Signal | Apply |
|---|---|---|
| Divergent Change | One class changed for many unrelated reasons | Extract Class; Extract Superclass; Split Phase |
| Shotgun Surgery | One change forces edits in many classes | Move Method; Move Field; Inline Class; Combine Functions into Class |
| Parallel Inheritance Hierarchies | Adding a subclass here forces one there | Move Method; Move Field; collapse one hierarchy into the other |

### Dispensables
| Smell | Signal | Apply |
|---|---|---|
| Duplicate Code | Same logic in several places | Extract Method; Pull Up Method; Form Template Method; Substitute Algorithm; Extract Class |
| Dead Code | Unreachable or unused | Delete it. Version control is the archive. |
| Lazy Class | Class does too little to justify itself | Inline Class; Collapse Hierarchy |
| Speculative Generality | Abstraction with one implementation, "for later" | Collapse Hierarchy; Inline Class; Remove Parameter; Inline Method |
| Comments (as deodorant) | Comment explains *what* confusing code does | Extract Method with an intention-revealing name; Rename; Introduce Assertion. Keep comments that explain *why*. |
| Data Class | Fields + getters/setters, no behavior | Move Method (bring behavior to the data); Encapsulate Field/Collection; Remove Setting Method |

### Couplers
| Smell | Signal | Apply |
|---|---|---|
| Feature Envy | Method uses another object's data more than its own | Move Method; Extract Method then Move Method |
| Inappropriate Intimacy | Two classes dig into each other's internals | Move Method/Field; Extract Class; Hide Delegate; Replace Inheritance with Delegation |
| Message Chains | `a.getB().getC().getD()` | Hide Delegate; Extract Method then Move Method |
| Middle Man | Class only delegates | Remove Middle Man; Inline Method; Replace Delegation with Inheritance |

## Refactoring toward patterns

Use a design pattern only when a smell pulls you there — never because the pattern is elegant.

- Conditional on type/state → **Strategy** or **State**
- Conditional creation / `new` scattered in logic → **Factory Method** or **Abstract Factory**
- Duplicate algorithm skeleton, differing steps → **Template Method**
- Wrapping behavior around an object, combinatorial subclasses → **Decorator**
- Awkward third-party or legacy interface → **Adapter** or **Facade**
- Repeated null checks → **Null Object**
- Complex multi-step construction → **Builder**
- Operations scattered across a type hierarchy → **Visitor** (use sparingly; it trades one rigidity for another)

## Naming and mechanics

- Name by **intent**, not implementation: `isEligibleForDiscount()` not `checkFlagTwo()`.
- Extract Method: name it for *what* it accomplishes; if you cannot name it, the boundary is wrong.
- Preserve the public contract unless the change is explicitly requested. If a signature must change, add the new form, migrate callers, then remove the old one.
- Keep each commit a single refactoring with a message naming the technique: `Extract Class: pull ShippingCalculator out of Order`.

## Anti-patterns to refuse

- Refactoring with no tests and no offer to add them.
- "Big bang" rewrites presented as refactoring.
- Changing behavior and structure in the same step.
- Applying every smell in the catalog to a file the user asked one question about.
- Introducing abstraction layers for hypothetical future needs.

## Output format

When reviewing:

```
## Smells found
1. <Smell name> — path/to/file.ext:42
   Why: <one line>
   Fix: <named refactoring>

## Proposed order
1. ... (highest payoff / lowest risk first)

## Risks
<anything untested or behavior-adjacent>
```

When refactoring, report each completed step as `<Technique>: <what moved where>` plus the test result.

## Reference

For step-by-step mechanics of the most common refactorings — Extract Method, Replace Conditional with Polymorphism, Replace Data Value with Object, Move Method, Hide Delegate, Form Template Method, and writing a characterization test for untested code — read `references/worked-examples.md`.