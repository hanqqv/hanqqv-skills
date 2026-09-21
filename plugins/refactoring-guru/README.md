# refactoring-guru

Refactor code using the Refactoring Guru / Fowler catalog.

## What it does

When you ask Claude to refactor, clean up, restructure, or review code for smells, this skill loads and changes how Claude works:

1. **Reads before judging**, and notes the public contract that must not change.
2. **Names the smell** — Long Method, Feature Envy, Shotgun Surgery, Primitive Obsession — with file and line.
3. **Applies the matching named refactoring** from the catalog, not an ad-hoc rewrite.
4. **Takes one step at a time**, verifying tests between each.
5. **Refuses to mix** structural change with behavior change in the same step.

## Triggers

"refactor this" · "clean up this code" · "review this for code smells" · "restructure this module" · "reduce the duplication here" · "this function is too long"

## Files

- `skills/refactoring-guru/SKILL.md` — the catalog, the map, the discipline
- `skills/refactoring-guru/references/worked-examples.md` — step-by-step mechanics, loaded on demand

## Tuning

The strictest rule is "tests green before you start" — on untested code Claude will stop and offer a characterization test instead of editing. Relax it by editing the **Core discipline** section of `SKILL.md`.
