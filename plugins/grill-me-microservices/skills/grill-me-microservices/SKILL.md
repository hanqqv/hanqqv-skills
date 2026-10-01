---
name: grill-me-microservices
description: Relentlessly grill the user on a microservice or distributed-system design, round by round, covering boundaries, communication, data, resilience, deployment, observability and end-to-end traceability, with Go as the default implementation language. Use when the user says "grill me" about microservices or wants a service design stress-tested.
---

# Grill Me: Microservices

This skill is Matt Pocock's `grilling` skill (the engine behind `grill-me`) **unchanged**, with a microservices layer on top. Part 1 is his method, word for word, and it always wins: the layer below only tells you *what* to put on the design tree and *how to judge answers*, never how to run the interview.

Sources: `grilling` / `grill-me` by Matt Pocock (Part 1, verbatim), `microservices-architect` by Jeffallan (MIT; architecture checklist), the `microservice-observability` skill, plus traceability and Go best practices.

---

## Part 1: The grilling method (Matt Pocock, verbatim)

Interview the user relentlessly until you reach a shared understanding. Map this as a **design tree**: every decision branches into the decisions that hang off it.

Work the tree in **rounds**. The **frontier** is every decision whose prerequisites are already settled: the questions you can ask _now_ without guessing at answers you haven't heard yet. Ask the whole frontier in one round: number each question and give your recommended answer. Then wait for the user's answers before the next round.

Format a round like so:

```
❓ **Q1** - **<question title>**: <question body, might be multiple paragraphs, including multiple choices>

➡️ <your recommended answer>

---

❓ **Q2** - **<question title>**: <question body, might be multiple paragraphs, including multiple choices>

➡️ <your recommended answer>
```

Each round the user answers reshapes the tree: settled decisions push the frontier outward and unblock questions that depended on them. Recompute the frontier and ask the next round. A question whose answer depends on another question still open in this round belongs to a _later_ round, not this one.

Finding _facts_ is your job, never the user's. When a frontier question needs a fact from the environment (filesystem, tools, etc.), dispatch a sub-agent to find it; don't ask the user for anything you could look up yourself. Don't block on it: a running exploration is an unsettled prerequisite, so only the questions downstream of it wait for the sub-agent to report; ask the rest of the frontier now. The _decisions_ are the user's: put each to them and wait.

The session is done when the frontier is empty: every branch of the design tree visited, nothing left silently assumed. Do not act on it until the user confirms you have reached a shared understanding.

---

## The microservices layer (on top of Part 1)

What this layer adds, without changing the method above:

- **Seeds the design tree** with the pillars in Part 2. Every pillar is a branch that must be visited before the frontier can be empty. Example chain: "Order and Payment are separate services" → "how do they stay consistent?" → "saga: orchestrated or choreographed?" → "how is the saga traced?". Raise traceability branches as soon as each integration point is known.
- **Shapes your recommended answers:** base them on Part 2's checkpoints, Part 3's MUST / MUST NOT list, and Go as the default language (Part 2 Pillar 8, Part 5). If an answer you hear violates Part 3 (distributed monolith, shared database, chatty interface, sync call on a long-running path, a hop that drops trace context), the follow-up questions on that branch should surface the failure scenario and ask the user to fix it or accept it as a recorded trade-off.
- **Typical facts to look up yourself** (per Part 1): repo layout, existing services, `go.mod`, OpenAPI/proto files, k8s manifests, Collector config.
- **Go first:** services are implemented in Go unless the user gives a reason otherwise. If a service must use another language, that is a branch to grill, and it must still follow the same propagation, telemetry and contract rules.
- **After the user confirms shared understanding**, produce the design summary in Part 4.

---

## Part 2: The design tree (what to grill on)

These are the branches of the design tree. Prerequisites decide which questions are on the frontier, per Part 1; the pillar order is just the usual dependency direction.

### Pillar 0: Context and drivers

- What problem forces microservices? (team scaling, independent deploys, differing scale profiles, fault isolation). Would a modular monolith be enough?
- Greenfield or decomposing a monolith? If decomposing: strangler fig, which seam first, how is traffic split and rolled back?
- Non-functional targets: latency, throughput, availability, data residency, compliance (PII, PCI, audit).
- Team topology: how many teams, who owns what? (Conway's Law: service boundaries should match team boundaries.)

### Pillar 1: Domain and service boundaries (DDD)

- What are the bounded contexts? What does the event storming / domain map look like? What is the ubiquitous language in each?
- For each candidate service: what aggregate(s) does it own, what is its public API contract, can it deploy independently?
- Sizing: is any service so small it is always deployed with another (merge it), or so large two teams fight over it (split it)?
- Which changes would require coordinated releases across services? (A yes means the boundary is probably wrong.)
- Context mapping between services: customer/supplier, conformist, anti-corruption layer, shared kernel (avoid)?

*Checkpoint:* each service owns its data exclusively, has a clear contract, and deploys independently.

### Pillar 2: Communication

- For each interaction: sync (REST / gRPC) or async (events / commands over Kafka, RabbitMQ, etc.)? Justify against latency and coupling.
- Rule of thumb: sync only for query/command pairs needing a fast (< ~100 ms) answer; long-running or cross-aggregate work goes async.
- Choreography or orchestration for multi-step flows? Who owns the flow?
- API style: resource design, pagination, idempotency keys on mutating calls, error model, API versioning strategy (URL, header, proto package) and deprecation policy.
- Event design: event vs command, schema format (Avro / Protobuf / JSON Schema), schema registry, compatibility mode, ordering key, at-least-once handling, dead-letter queue.
- API gateway / BFF: what does it do (auth, rate limiting, aggregation) and what must it *not* do (business logic)?
- Chattiness: how many hops does the hottest user request make? Can it be cut?

### Pillar 3: Data

- Database per service: confirm no shared schema, no cross-service joins, no reading another service's tables.
- Consistency model per flow: strong vs eventual. Where is eventual consistency visible to users, and is that acceptable?
- Distributed transactions: saga (never 2PC across services). For each saga step: action, compensation, idempotency, timeout.
- Reliable publishing: transactional outbox or CDC (e.g. Debezium) so DB write and event publish can't diverge?
- Read models: CQRS / materialized views needed? How are they rebuilt?
- Event sourcing: needed or over-engineering? If used: snapshots, schema evolution/upcasting.
- Reference data duplicated across services: who is the source of truth, how is it synced?
- Partitioning / sharding keys, data retention, GDPR deletion across services.

*Checkpoint:* consistency boundaries align with bounded contexts.

### Pillar 4: Resilience

For **every** integration point, get explicit answers:

- Timeout (and how it relates to the caller's deadline; propagate deadlines, don't stack them).
- Retry policy: which errors, how many, exponential backoff + jitter, retry budget. Only idempotent operations retry.
- Circuit breaker thresholds and what the fallback returns.
- Bulkheads: separate pools / limits so one slow dependency can't exhaust the service.
- Graceful degradation: what does the user see when this dependency is down?
- Load shedding and rate limiting; backpressure on consumers.
- Poison messages: DLQ, replay procedure, alerting.

*Checkpoint:* every external call has a timeout, retry budget, and degradation path.

### Pillar 5: Deployment and operations

- Containers / Kubernetes, liveness (`/healthz`) vs readiness (`/readyz`) probes and what readiness checks.
- Progressive delivery: canary or blue-green, automated rollback signals (tie to SLOs).
- Service mesh (Istio / Linkerd) or library-based? mTLS, retries in mesh vs app (never both).
- Config and secrets management, environment parity.
- Backward compatibility during rolling deploys (expand/contract for DB and API changes).
- Contract tests (e.g. Pact, buf breaking) in CI.

*Checkpoint:* probes defined; rollout strategy documented.

### Pillar 6: Observability

- **OpenTelemetry everywhere**, OTLP to an OTel Collector; no vendor SDK in service code.
- Standard resource attributes on all telemetry: `service.name`, `service.version`, `deployment.environment.name`, k8s pod/namespace (via Collector `k8sattributes`).
- **Metrics:** RED per endpoint/consumer (rate, errors, duration *histogram*); USE for resources (DB pool, queue depth, consumer lag, memory/GC); business metrics for key domain events. No high-cardinality attributes (user IDs, request IDs, raw URLs, error messages); use route templates and bounded enums.
- **Logs:** structured JSON to stdout, context-aware logging so `trace_id`/`span_id` attach automatically; log errors once at the boundary; no per-success-request logs; never log secrets or PII.
- **SLOs:** for each user-facing journey, what is the SLI, target and window? Alerts on multi-window burn rate, each with a runbook and dashboard. Deploys annotated on dashboards.
- Which backend(s): Prometheus/Mimir, Loki, Tempo/Jaeger, or a vendor? Retention per signal?

### Pillar 7: Traceability (grill hardest here)

The goal: **given any user complaint, error log, alert, or business record, you can reach the full end-to-end trace in one hop, and from any trace you can reach the logs and metrics around it.** Grill every item below for the design at hand.

**7.1 Propagation standard**
- W3C Trace Context (`traceparent`, `tracestate`) is the single propagation format across HTTP, gRPC and messaging. If legacy B3 / vendor headers exist, configure a composite propagator during migration, then remove it.
- Every hop propagates context: gateway, BFF, service mesh sidecars, every service, every outbound client (HTTP, gRPC, DB, cache, queue). Ask: "Show me one hop where context could be dropped." Typical gaps: hand-built HTTP clients, background goroutines/threads/executors that don't carry context, scheduled jobs, webhooks, third-party callbacks, batch exports.
- Language rule: context is passed explicitly through every function that does I/O (Go `context.Context`; Java/Node/Python context propagation through async boundaries and thread pools verified).

**7.2 Edge and trust boundaries**
- Where does the trace start? Normally at the API gateway / ingress. Decide whether incoming `traceparent` from the public internet is trusted, re-rooted (start a new trace and link to the external one), or ignored.
- Return the trace ID to clients (e.g. `traceresponse` or `X-Trace-Id` response header) and include it in error response bodies, so support can paste it straight into the trace UI.
- Correlation ID vs trace ID: prefer the trace ID as the one correlation key. If a separate business `x-correlation-id` / `x-request-id` must exist (e.g. partner contracts), generate it at the edge, propagate it, and record it as a span attribute and log field so both are searchable.

**7.3 Asynchronous and event-driven flows**
- Producers inject trace context into message headers; consumers extract it and start a `CONSUMER` span (producers use `PRODUCER` spans), following OTel messaging semantic conventions.
- Batch consumers and fan-in (one span processing many messages) use **span links** to each message's context instead of a single parent.
- **Transactional outbox:** store the trace context in the outbox row so the relay publishes the event with the *original* request's context, not the relay's.
- Event envelope carries business traceability fields: `event_id`, `correlation_id` (the business flow), `causation_id` (the event/command that caused this one), plus trace context. Consider CloudEvents with its distributed-tracing extension.
- Long-running sagas that outlive a trace: keep one `saga_id` / `correlation_id` as a span attribute on every step, start new traces per step if needed, and link them, so the whole saga can be found by one ID.
- Retries and DLQ replays keep or link the original context, so a replayed message is traceable back to its origin.

**7.4 Span quality**
- Auto-instrument inbound and outbound (HTTP, gRPC, DB, cache, queue). Manual spans only for meaningful business steps or expensive work.
- Span names are low-cardinality operation names (`payment.charge`, `GET /orders/{id}`), never containing IDs.
- Domain IDs (`order.id`, `customer.tier`, `saga.id`) go on span *attributes* so traces are searchable by business key. Follow OTel semantic conventions for standard attributes.
- On error: record the exception and set span status to error, with a short message. Span status must agree with the HTTP/gRPC status semantics.
- No secrets or PII in span attributes, events, or **baggage** (baggage is propagated to every downstream service and often to third parties). Only allow-listed keys in baggage; strip baggage at trust boundaries.

**7.5 Sampling**
- SDKs use parent-based sampling so a trace is never half-sampled across services.
- Decide head vs tail: recommended is ~100% at SDK + **tail-based sampling in the Collector** (keep all errors, all slow traces above a latency threshold, a % of the rest, and always-keep rules for critical journeys like checkout/payment).
- Tail sampling needs all spans of a trace on the same Collector instance: use a load-balancing exporter keyed by trace ID in front of the sampling tier.
- Cost check: estimated spans/sec, storage per day, retention period.

**7.6 Linking the signals**
- Logs carry `trace_id` and `span_id` automatically; the log backend links to the trace UI and the trace UI links back to logs for that trace.
- Metrics carry **exemplars** (trace IDs on histogram samples) so a latency spike on a dashboard opens a representative trace.
- Alerts and runbooks link to a pre-filtered trace search.

**7.7 Business and audit traceability**
- Can you answer "what happened to order X?" across every service from one search (by `order.id` attribute or correlation ID)?
- Traces are sampled and short-lived, so they are not an audit log. If compliance needs a durable audit trail, design it separately (append-only events with `event_id`/`causation_id`/actor/timestamp) and link it to traces via IDs.
- Clock skew: rely on spans' parent/child structure, not wall-clock ordering, across services; keep NTP in place.

**7.8 Verification**
- How will you prove end-to-end tracing works before go-live? Recommend: a synthetic or integration test that sends one request through the critical path (including the async leg), then asserts a single trace with the expected service/span tree exists.
- Trace completeness checks in CI or staging (e.g. no orphan root spans from internal services).
- Service map / dependency graph generated from traces, reviewed against the intended architecture to catch hidden calls.

*Checkpoint:* one request on every critical journey (sync and async legs) can be traced end-to-end by one ID, from the gateway to the last consumer, and linked to its logs and metrics.

### Pillar 8: Go implementation

Grill the Go-level decisions once the architecture is settled. If a repo exists, have a sub-agent read `go.mod`, the layout and existing middleware before asking.

- **Go version** (1.22+ for method/pattern routing in `net/http`); one module per service or a monorepo with `go.work`?
- **Layout:** `cmd/<service>/main.go`, `internal/` for everything not meant to be imported (`internal/domain`, `internal/app`, `internal/transport/{http,grpc,kafka}`, `internal/store`, `internal/telemetry`), `api/` for OpenAPI/proto. Domain packages must not import transport or infrastructure.
- **Transport:** stdlib `net/http` (or `chi`) for REST; `google.golang.org/grpc` or `connectrpc.com/connect` for RPC; proto managed with `buf` (lint + `buf breaking` in CI). Avoid heavy frameworks unless justified.
- **Messaging:** `github.com/twmb/franz-go` or `segmentio/kafka-go` for Kafka; `rabbitmq/amqp091-go` for RabbitMQ; always wrapped so trace context is injected/extracted in one place.
- **Data:** `pgx` (+ `sqlc` for type-safe queries) for Postgres; migrations with `golang-migrate` or `goose`; outbox table written in the same `pgx.Tx` as the domain change.
- **Context discipline:** `ctx context.Context` is the first parameter of every function doing I/O; never stored in structs; deadlines set at the edge and inherited; no `context.Background()` inside request paths (only in `main`, consumers' extract step, and tests).
- **Concurrency:** `errgroup.WithContext` for fan-out so cancellation and trace context flow to goroutines; bounded worker pools (bulkheads); no fire-and-forget goroutines without context and recovery.
- **Errors:** wrap with `fmt.Errorf("op: %w", err)`, sentinel/typed errors for domain cases, mapped to HTTP/gRPC status codes in the transport layer only; log once at the boundary.
- **Resilience libs:** `github.com/sony/gobreaker/v2` (circuit breaker, generic API), `github.com/cenkalti/backoff/v4` or hand-rolled backoff + jitter, `golang.org/x/time/rate` (rate limiting), `http.Client{Timeout: ...}` never the zero-value default client.
- **Config & lifecycle:** config from env (`caarlos0/env` or `kelseyhightower/envconfig`), validated at startup; `signal.NotifyContext` for SIGTERM, `srv.Shutdown(ctx)`, drain consumers, then flush telemetry.
- **Telemetry:** follow the `microservice-observability` Go patterns: `telemetry.Setup` in `main`, `otelhttp`/`otelgrpc` handlers and transports, `otelsql`/`otelpgx`, `redisotel`, `log/slog` JSON with a trace-aware handler, runtime metrics.
- **Testing:** table-driven unit tests; `testcontainers-go` for Postgres/Kafka integration tests; contract tests; `go test -race`; an end-to-end trace assertion test (OTel `tracetest.SpanRecorder` in-process, or query the trace backend in staging).
- **Build & run:** static binary, distroless or scratch image, non-root; `govulncheck` and `golangci-lint` in CI; `GOMAXPROCS`/`GOMEMLIMIT` set for container limits (e.g. `go.uber.org/automaxprocs`).

---

## Part 3: Red flags (judge answers against these)

**MUST**
- DDD-derived boundaries; database per service; circuit breakers + timeouts on external calls; async for cross-aggregate and long-running operations; graceful degradation; health and readiness probes; API versioning; W3C trace context propagated on every hop including messaging; correlation/trace ID returned to clients.

**MUST NOT**
- Distributed monolith (services that must deploy together); shared databases; sync calls for long-running operations; chatty interfaces; shared mutable state without a pattern; ignoring network latency and partial failure; retries without idempotency; retries at both mesh and app layer; high-cardinality metric labels; PII in logs, spans or baggage; deploying anything without observability; any hop that drops trace context.

---

## Part 4: Wrap-up (only after the user confirms shared understanding)

Produce a design summary containing:

1. **Service boundary diagram** with bounded contexts and owning teams (Mermaid is fine).
2. **Communication map**: each interaction, sync/async, protocol, contract location, versioning.
3. **Data ownership and consistency model**, including sagas with compensations and outbox/CDC choice.
4. **Resilience table**: per integration point, timeout / retries / breaker / fallback.
5. **Deployment requirements**: probes, rollout strategy, mesh, config.
6. **Observability plan**: SLOs and alerts, key metrics, logging conventions, backends.
7. **Traceability plan**: propagation format, edge policy, async/outbox propagation, event envelope fields, sampling policy, signal linking, and the end-to-end trace verification test, plus one sequence diagram of the most critical journey showing where context is injected/extracted.
8. **Decision log**: every decision made in the grilling, with the alternative rejected and why (ADR-style), and every accepted trade-off / known risk.
9. **Go service skeleton** for each service: package layout, chosen libraries, and where telemetry, propagation, resilience and the outbox live.
10. **Open questions** if any remain (there should be none if the frontier is empty).

---

## Part 5: Go reference patterns

Use these when recommending answers or when the user asks "what would that look like?". Check the project's `go.mod` and match its OTel/semconv versions before suggesting imports.

### Edge: return the trace ID and keep a business correlation ID

```go
func TraceIDMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		ctx := r.Context()
		sc := trace.SpanContextFromContext(ctx) // set by otelhttp, which wraps this handler
		if sc.IsValid() {
			w.Header().Set("X-Trace-Id", sc.TraceID().String())
		}
		corrID := r.Header.Get("X-Correlation-Id")
		if corrID == "" {
			corrID = uuid.NewString()
		}
		w.Header().Set("X-Correlation-Id", corrID)
		trace.SpanFromContext(ctx).SetAttributes(attribute.String("app.correlation_id", corrID))
		next.ServeHTTP(w, r.WithContext(withCorrelationID(ctx, corrID)))
	})
}

// wiring: otelhttp outermost so the span exists before TraceIDMiddleware runs
srv := &http.Server{Handler: otelhttp.NewHandler(TraceIDMiddleware(mux), "http.server",
	otelhttp.WithFilter(func(r *http.Request) bool { return r.URL.Path != "/healthz" && r.URL.Path != "/readyz" }),
)}
```

Error responses include `trace_id` in the body so support can find the trace.

### Transactional outbox that preserves trace context

```go
// write side: same transaction as the domain change
func (s *Store) PlaceOrder(ctx context.Context, o Order) error {
	return pgx.BeginFunc(ctx, s.pool, func(tx pgx.Tx) error {
		if err := insertOrder(ctx, tx, o); err != nil {
			return fmt.Errorf("insert order: %w", err)
		}
		carrier := propagation.MapCarrier{}
		otel.GetTextMapPropagator().Inject(ctx, carrier) // capture the request's trace context
		hdrs, _ := json.Marshal(carrier)
		_, err := tx.Exec(ctx, `INSERT INTO outbox (id, topic, key, payload, trace_headers)
			VALUES ($1, 'orders.placed', $2, $3, $4)`, uuid.NewString(), o.ID, mustJSON(o), hdrs)
		return err
	})
}

// relay: publish with the ORIGINAL context, not the relay's own
func (r *Relay) publish(row OutboxRow) error {
	var carrier propagation.MapCarrier
	_ = json.Unmarshal(row.TraceHeaders, &carrier)
	ctx := otel.GetTextMapPropagator().Extract(context.Background(), carrier)
	ctx, span := tracer.Start(ctx, "orders.placed publish", trace.WithSpanKind(trace.SpanKindProducer))
	defer span.End()
	return r.producer.Produce(ctx, row.Topic, row.Key, row.Payload) // Produce injects ctx into headers
}
```

### Kafka produce / consume (franz-go) with propagation and batch links

```go
type headerCarrier struct{ r *kgo.Record }

func (c headerCarrier) Get(k string) string {
	for _, h := range c.r.Headers {
		if h.Key == k {
			return string(h.Value)
		}
	}
	return ""
}
func (c headerCarrier) Set(k, v string) { c.r.Headers = append(c.r.Headers, kgo.RecordHeader{Key: k, Value: []byte(v)}) }
func (c headerCarrier) Keys() []string {
	ks := make([]string, 0, len(c.r.Headers))
	for _, h := range c.r.Headers {
		ks = append(ks, h.Key)
	}
	return ks
}

// produce
rec := &kgo.Record{Topic: topic, Key: key, Value: payload}
otel.GetTextMapPropagator().Inject(ctx, headerCarrier{rec})
client.Produce(ctx, rec, onDone)

// consume one message: continue the producer's trace
ctx := otel.GetTextMapPropagator().Extract(context.Background(), headerCarrier{rec})
ctx, span := tracer.Start(ctx, rec.Topic+" process", trace.WithSpanKind(trace.SpanKindConsumer))
defer span.End()

// consume a batch: one span, linked to every message's context
links := make([]trace.Link, 0, len(recs))
for _, r := range recs {
	mctx := otel.GetTextMapPropagator().Extract(context.Background(), headerCarrier{r})
	links = append(links, trace.Link{SpanContext: trace.SpanContextFromContext(mctx)})
}
ctx, span := tracer.Start(context.Background(), "orders batch process",
	trace.WithSpanKind(trace.SpanKindConsumer), trace.WithLinks(links...))
defer span.End()
```

(`github.com/twmb/franz-go/plugin/kotel` provides this instrumentation out of the box; prefer it if adopted.)

### Outbound call: timeout + retry with jitter + circuit breaker

```go
var cb = gobreaker.NewCircuitBreaker[*StockResp](gobreaker.Settings{
	Name:        "inventory",
	Timeout:     30 * time.Second, // open → half-open
	ReadyToTrip: func(c gobreaker.Counts) bool { return c.ConsecutiveFailures >= 5 },
})

func (c *InventoryClient) Stock(ctx context.Context, sku string) (*StockResp, error) {
	ctx, cancel := context.WithTimeout(ctx, 800*time.Millisecond) // within caller's deadline
	defer cancel()
	op := func() (*StockResp, error) {
		return cb.Execute(func() (*StockResp, error) { return c.get(ctx, sku) }) // c.http uses otelhttp.NewTransport
	}
	resp, err := backoff.RetryWithData(op, backoff.WithContext(
		backoff.WithMaxRetries(backoff.NewExponentialBackOff(), 2), ctx)) // idempotent GET only
	if errors.Is(err, gobreaker.ErrOpenState) {
		return &StockResp{Available: false, Degraded: true}, nil // graceful degradation
	}
	return resp, err
}
```

### Orchestrated saga step with traceable saga ID

```go
type Step struct {
	Name       string
	Execute    func(ctx context.Context, s *OrderSaga) error
	Compensate func(ctx context.Context, s *OrderSaga) error
}

func Run(ctx context.Context, s *OrderSaga, steps []Step) error {
	ctx, span := tracer.Start(ctx, "saga.order", trace.WithAttributes(attribute.String("saga.id", s.ID)))
	defer span.End()
	var done []Step
	for _, st := range steps {
		sctx, sspan := tracer.Start(ctx, "saga.step "+st.Name)
		err := st.Execute(sctx, s)
		sspan.End()
		if err != nil {
			span.RecordError(err)
			span.SetStatus(codes.Error, "saga failed at "+st.Name)
			for i := len(done) - 1; i >= 0; i-- {
				cctx, cspan := tracer.Start(ctx, "saga.compensate "+done[i].Name)
				if cerr := done[i].Compensate(cctx, s); cerr != nil {
					cspan.RecordError(cerr) // compensation failures must alert, not just log
				}
				cspan.End()
			}
			return fmt.Errorf("saga %s step %s: %w", s.ID, st.Name, err)
		}
		done = append(done, st)
	}
	return nil
}
```

Persist saga state so it survives restarts; compensations must be idempotent.

### Fan-out without losing context

```go
g, gctx := errgroup.WithContext(ctx) // gctx carries the trace and the deadline
g.Go(func() error { return pricing.Quote(gctx, req) })
g.Go(func() error { return inventory.Reserve(gctx, req) })
if err := g.Wait(); err != nil {
	return fmt.Errorf("checkout fan-out: %w", err)
}
```

### End-to-end trace test (in-process)

```go
func TestCheckoutIsOneTrace(t *testing.T) {
	sr := tracetest.NewSpanRecorder()
	otel.SetTracerProvider(sdktrace.NewTracerProvider(sdktrace.WithSpanProcessor(sr)))
	otel.SetTextMapPropagator(propagation.TraceContext{})

	runCheckoutThroughHTTPAndKafka(t) // spin up handlers + testcontainers Kafka

	spans := sr.Ended()
	traceID := spans[0].SpanContext().TraceID()
	for _, s := range spans {
		if s.SpanContext().TraceID() != traceID && len(s.Links()) == 0 {
			t.Fatalf("span %q is orphaned from the checkout trace", s.Name())
		}
	}
}
```
