# grill-me-microservices

Matt Pocock's **grill-me** interview, unchanged, with a microservices layer on top.

## What it does

When you ask Claude to grill you on a microservice or distributed-system design, this skill loads and Claude:

1. **Runs Matt Pocock's grilling method word for word**: maps a design tree, asks every open question in numbered rounds with a recommended answer, looks up facts itself, and stops only when every branch is settled and you confirm.
2. **Seeds the tree with microservice pillars**: context, DDD boundaries, communication, data, resilience, deployment, observability, traceability and Go implementation.
3. **Judges answers against a MUST / MUST NOT list** (no shared databases, no distributed monoliths, no hop that drops trace context, and so on).
4. **Grills hardest on traceability**: W3C trace context on every hop, async and outbox propagation, span links, sampling, linking logs and metrics to traces, and an end-to-end trace test.
5. **Defaults to Go** for every recommendation and code example.
6. **Writes a design summary** after you confirm: boundary diagram, communication map, data and resilience tables, observability and traceability plans, decision log, and a Go service skeleton.

## Triggers

"grill me on this microservice design" · "stress-test my service architecture" · `/grill-me-microservices`

## Files

- `skills/grill-me-microservices/SKILL.md`: the method, the design tree, red flags, wrap-up template and Go reference patterns

## Tuning

Part 1 of `SKILL.md` is Matt Pocock's text verbatim. To change what gets asked, edit Parts 2–5 and leave Part 1 alone.

The Go reference patterns are illustrative. Check them against your `go.mod` versions before you copy them.
