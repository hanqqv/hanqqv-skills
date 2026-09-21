# hanqqv-skills

A Claude plugin marketplace — a collection of skills that change how Claude approaches particular kinds of work.

## Install

```
/plugin marketplace add hanqqv/hanqqv-skills
```

Then install whichever plugin you want:

```
/plugin install refactoring-guru@hanqqv-skills
```

Or browse with `/plugin` and pick from the list.

## Plugins

### refactoring-guru

Refactoring the disciplined way: name the smell, apply the matching refactoring from the Fowler / Refactoring Guru catalog, take one small behavior-preserving step at a time, verify after each.

**Triggers on:** "refactor this" · "clean up this code" · "review this for code smells" · "restructure this module"

**What it gives Claude:**

- A smell catalog — bloaters, object-orientation abusers, change preventers, dispensables, couplers — each with its detection signal.
- A smell → refactoring map, so the fix is a named technique rather than an improvisation.
- Refactoring-toward-patterns guidance: Strategy, State, Template Method, Decorator, Adapter, Null Object and friends get introduced only when a smell pulls you there.
- Working discipline: tests green before starting, one refactoring per step, verify between each, never mix structural and behavioral change.
- Worked examples with step-by-step mechanics, loaded on demand.

**Expect it to be cautious.** Asked to refactor untested code, Claude stops and offers a characterization test rather than editing blind. Asked to refactor a large module, it inventories the smells, proposes an ordered plan, and waits for agreement before touching anything crossing a public API. If that's too strict, edit the **Core discipline** section of the skill.

## Repository layout

```
.claude-plugin/
  marketplace.json                # marketplace manifest — lists every plugin
plugins/
  refactoring-guru/
    .claude-plugin/plugin.json    # plugin manifest
    skills/
      refactoring-guru/
        SKILL.md
        references/
          worked-examples.md      # loaded on demand
    README.md
```

**Adding a skill to an existing plugin:** drop in `plugins/<plugin>/skills/<name>/SKILL.md`. No manifest change needed.

**Adding a new plugin:** create `plugins/<name>/` with its own `.claude-plugin/plugin.json`, then add an entry to the `plugins` array in `marketplace.json`.

## Credit

The refactoring-guru skill's smell catalog and technique names come from Martin Fowler's *Refactoring* and the [Refactoring Guru](https://refactoring.guru/) presentation of it. This repo contains original prose describing that method, not their text.

## License

MIT — see [LICENSE](LICENSE).
