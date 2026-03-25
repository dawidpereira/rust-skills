---
name: rust-tracing
description: >
  Rust observability with tracing — structured logging, spans, subscribers, and instrumentation.
  Use when choosing between tracing and log, setting up subscribers and layers, adding #[instrument]
  to functions, configuring RUST_LOG, instrumenting async code (.instrument() vs span.enter()),
  integrating tower-http or sqlx middleware, protecting sensitive data in logs, or preparing tracing
  for production with OpenTelemetry. Also use when reviewing code for logging quality or adding
  request correlation IDs.
---

# Tracing & Observability

## Core Question

**What happened, and why, at every layer of this request?**

Events say *what* happened. Spans say the *context* in
which it happened. If you only emit events without spans,
you lose the "why." `tracing` unifies both into one
structured, async-aware API.

---

## Error → Design Question

| Symptom                            | Don't Just Say              | Ask Instead                                                   |
| ---------------------------------- | --------------------------- | ------------------------------------------------------------- |
| Logs are noise, nobody reads them  | "Add more log levels"       | What decisions will an operator make from this output?         |
| `span.enter()` in async code       | "Use `.instrument()`"       | Does this span need to cross await points?                    |
| Sensitive data appears in logs     | "Filter it in the output"   | Should this field exist in the span at all?                   |
| No context in error logs           | "Add more fields"           | Should this be a span (context) rather than an event (fact)?  |
| `info!` scattered everywhere       | "Logging is good practice"  | Which layer owns this observability concern?                  |
| Perf regression after adding spans | "Remove some traces"        | Are you using compile-time filtering and non-blocking writers? |

---

## Quick Decisions

| Situation                              | Reach For                               | Why                                                    |
| -------------------------------------- | --------------------------------------- | ------------------------------------------------------ |
| New project, choosing a logging crate  | `tracing`                               | Structured, span-aware, async-compatible, `log`-compat |
| Library crate                          | `tracing` with `log` feature            | Consumers using `log` still see events                 |
| Dev/local output                       | `fmt::layer().pretty()`                 | Human-readable, colored                                |
| Production JSON output                 | `fmt::layer().json()`                   | Machine-parseable for log aggregators                  |
| Controlling verbosity at runtime       | `EnvFilter` + `RUST_LOG`               | Per-module filtering without recompile                 |
| Stripping debug/trace from release     | `tracing` `max_level_info` feature      | Zero cost for disabled levels at compile time          |
| Instrumenting an async function        | `#[instrument]`                         | Auto-creates span, captures args, async-safe           |
| Propagating span through `.await`      | `.instrument(span)`                     | Span stays active across yield points                  |
| Entering a span in sync code           | `let _guard = span.enter()`            | RAII guard, dropped at scope end                       |
| Axum/tower HTTP request tracing        | `tower_http::trace::TraceLayer`         | Per-request spans with method, URI, status, latency    |
| SQL query tracing                      | `sqlx` with `tracing` feature           | Automatic spans for every query                        |
| Outbound HTTP tracing                  | `reqwest-tracing`                       | Span per outbound request                              |
| Protecting sensitive fields            | `secrecy::Secret<T>` or `Redacted`     | Prevents accidental exposure in logs                   |
| Request ID / correlation               | Extract or generate UUID in span field  | Ties all spans/events for one request                  |
| Sending traces to OTel collector       | `tracing-opentelemetry` layer           | Bridges tracing spans to Jaeger/Tempo/Datadog          |
| Writing traces to files                | `tracing-appender` + `non_blocking`     | Rolling files without blocking the runtime             |
| Testing that traces were emitted       | `tracing-test` + `#[traced_test]`       | Captures logs per-test with `logs_contain!`            |

---

## The Async Span Problem

Never use `span.enter()` in async code. The guard is
tied to the current thread — when the future yields at
an `.await` and resumes on a different thread, the span
context is wrong.

```rust
// Bad: guard held across await
async fn handle(req: Request) -> Response {
    let span = info_span!("handle_request");
    let _guard = span.enter();
    let data = fetch_data().await; // span context lost here
    process(data)
}

// Good: #[instrument] handles async spans correctly
#[instrument(skip(req))]
async fn handle(req: Request) -> Response {
    let data = fetch_data().await;
    process(data)
}

// Good: .instrument() for dynamic span names
async fn handle(req: Request) -> Response {
    let span = info_span!("handle_request", method = %req.method());
    async {
        let data = fetch_data().await;
        process(data)
    }
    .instrument(span)
    .await
}
```

Spawned tasks also need explicit span propagation:

```rust
let span = info_span!("background_job", job_id = %id);
tokio::spawn(
    async move { run_job().await }.instrument(span)
);
```

---

## Log Levels

| Level   | Use For                                 | Example                                  |
| ------- | --------------------------------------- | ---------------------------------------- |
| `error` | Failures requiring operator attention   | DB connection lost, payment failed       |
| `warn`  | Degraded but recoverable                | Retry succeeded, cache miss fallback     |
| `info`  | Business-significant events             | Request handled, user created, deployed  |
| `debug` | Developer troubleshooting               | SQL text, parsed config, cache hit/miss  |
| `trace` | Fine-grained flow                       | Loop iterations, byte-level I/O          |

If you would wake someone at 3am for it, it is `error`.
If you would mention it in a standup, it is `info`.
Everything else is `debug` or `trace`.

---

## Usage Scenarios

**Scenario 1:** "I'm starting a new Axum service and need logging"
→ Set up `tracing_subscriber` with `fmt::layer()` for dev
and `json()` for production. Add `TraceLayer` to your router
for per-request spans. Use `#[instrument]` on handlers.
See `references/setup.md` for init patterns.

**Scenario 2:** "I have logs but can't tell which belong to the same request"
→ Extract or generate a request ID in middleware, add it as
a span field. All events within that span tree carry the ID.
See `references/instrumentation.md` for correlation patterns.

**Scenario 3:** "I need traces in Jaeger for production but plain text locally"
→ Use a layered subscriber: `fmt::layer()` for local,
`tracing-opentelemetry` for production. Switch via config.
See `references/production.md` for OpenTelemetry setup.

---

## Reference Files

| File                                                                   | Read When                                                                                      |
| ---------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| [references/setup.md](references/setup.md)                             | Cargo.toml deps, subscriber init (dev/prod), RUST_LOG syntax, EnvFilter, compile-time filtering |
| [references/instrumentation.md](references/instrumentation.md)         | #[instrument] options, tower-http/sqlx/reqwest middleware, correlation IDs, what to log by layer |
| [references/structured-fields.md](references/structured-fields.md)     | Field sigils (%, ?), naming conventions, error logging patterns, sensitive data protection       |
| [references/production.md](references/production.md)                   | OpenTelemetry, file output, non-blocking writers, performance, testing, production checklist     |

---

## Cross-References

| When                                          | Check                              |
| --------------------------------------------- | ---------------------------------- |
| Span context across `.await`, Send bounds     | rust-async → Quick Decisions       |
| Error logging, error chain display            | rust-errors → Quick Decisions      |
| Project Cargo.toml setup, lint defaults       | rust-quality → Quick Decisions     |
| Axum middleware and architecture patterns     | rust-architecture → Quick Decisions |
