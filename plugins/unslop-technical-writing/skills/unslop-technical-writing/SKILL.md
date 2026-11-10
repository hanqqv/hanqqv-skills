---
name: unslop-technical-writing
description: Use when writing or editing technical prose an AI produced or will produce — README/.md docs, design docs, code comments, godoc/docstrings, commit messages, PR descriptions, Jira text — or when the user says text is "AI slop", "too wordy", "sounds like ChatGPT", or asks to unslop, tighten, or de-AI it.
---

# Unslop Technical Writing

## Overview

Slop is text that costs the reader time without giving them facts. Every sentence must carry a fact the reader needs and cannot get faster from the code, a table, or the line above. Cut everything else. Never add a fact you have not verified.

## The recipe

Work in this order. Each pass is a check against the text, not a vibe.

1. **Facts.** Mark every claim: behaviour, status code, field name, limit, error, "tests pass". A source is: code, spec, command output, or text the user gave you (including the draft being edited). Writing new text: include a claim only if a source states it. Editing: keep every claim the draft makes, even if you cannot check it; never delete a fact as "unverified", flag it instead. Never fill gaps with plausible defaults (`500 on DB error`, `{"report_url": ...}`, "wrapped and logged"); write `TODO: confirm <x>` where the reader needs it.
2. **Once.** Each fact appears in one place. A limit stated in the field table is not restated in Behaviour, Responses and the example.
3. **Comments say why, not what.** Delete a comment that restates the next line (`// Load codes from S3` above `ReadCodes`). Keep comments that give a reason, constraint, or non-obvious consequence (`// presign instead of returning the key: the bucket is private`). Write a "why" only when a source gives it; if the reason is unknown, leave the comment out rather than inventing one. Doc comments: one line for what it does, then only contract details a caller needs (errors returned, limits, ownership). No restating the signature.
4. **Cut the slop patterns** below.
5. **Shape.** Lead with the thing (what it is / what changed). Headings and bullets only when there are 3+ parallel items; otherwise prose. Tables for lookups (fields, status codes), not for two rows.
6. **Read it as the reader.** Who opens this and what do they need first? A PR reviewer needs what changed and what to check; a README reader needs how to call it. Reorder for that.

## Slop patterns → fix

| Pattern | Example | Fix |
|---|---|---|
| Throat-clearing opener | "This section provides an overview of…", "In this PR, we…" | Start with the content |
| Hype / filler adjectives | robust, seamless, comprehensive, powerful, efficient, leverage, utilize, ensure | Delete, or replace with the measurable fact |
| Hedging | "may potentially", "it is worth noting that", "generally" | State the claim plainly; if editing and unsure it's true, also flag it |
| Summary/recap ending | "In summary…", "Overall, this change…" | Delete |
| Restated heading | `## Validation` → "Validation is performed to validate…" | Delete the sentence |
| Narrating steps the code shows | "1. The request is validated. 2. Codes are read…" | Keep only steps a caller must know |
| Rule of three / padded lists | "fast, reliable, and scalable" | Keep the one that's true and specific |
| Emoji, bold-everything, "Key Features" | ✨ **Robust** handling | Plain text |
| Status claims not run | "make test passes", "swagger regenerated" | Only if you ran it; else omit or `TODO` |
| Meta notes to the reader | "Note: replace the placeholder…" in the final text | Move to your chat reply, not the artifact |
| Em-dash chains and "not X, but Y" | "It's not just a check — it's a guarantee" | Plain sentence |

## Editing existing text

Output the rewrite, then a short list in your reply (not in the text): facts you flagged as doubtful, and any claim you dropped or changed in meaning. Slop you cut needs one summary line, not an item each. Adding facts the draft lacks: suggest them in the list, don't insert them. A heading over a single short paragraph can go.

## Common mistakes

- Cutting a real constraint because it looked like filler (TTLs, limits, error names stay).
- Making text shorter but adding invented specifics to sound concrete.
- Converting good prose into bullet fragments.
