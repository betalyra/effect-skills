# Observability Patterns (Effect v4)

## Table of Contents

- [What Changed From v3](#what-changed-from-v3)
- [Structured Logging with Effect.log](#structured-logging-with-effectlog)
- [Effect.fn for Automatic Tracing](#effectfn-for-automatic-tracing)
- [Span Annotations](#span-annotations)
- [Metrics](#metrics)
- [Configuration with Config](#configuration-with-config)
- [Log Level and Logger Configuration](#log-level-and-logger-configuration)
- [Exporting Telemetry (OTLP)](#exporting-telemetry-otlp)
- [Combining Observability](#combining-observability)

## What Changed From v3

| v3                              | v4                                              |
| ------------------------------- | ----------------------------------------------- |
| `Metric.increment(counter)`     | `Metric.update(counter, 1)`                     |
| `Metric.set(gauge, n)`          | `Metric.update(gauge, n)`                       |
| `Metric.tagged(k, v)`           | `Metric.withAttributes(metric, { k: v })`       |
| `Metric.timerWithHistogram`     | `Metric.timer(name)` + `Metric.update(m, dur)`  |
| `Config.integer`                | `Config.int`                                    |
| `Config.validate({...})`        | `Config.schema(codec, path)` / `Config.mapOrFail` |
| `Logger.minimumLogLevel(lvl)`   | `Layer.succeed(References.MinimumLogLevel, lvl)` |
| `Logger.json`                   | `Logger.layer([Logger.consoleJson])`            |
| `Layer.unwrapEffect`            | `Layer.unwrap`                                  |
| `@effect/opentelemetry/Otlp*`   | `effect/unstable/observability/Otlp*`           |
| `Secret.value`                  | `Redacted.value`                                |

## Structured Logging with Effect.log

**Always use `Effect.log`** instead of `console.log`. It gives you structured
data, levels, telemetry integration, and testability.

### Basic Logging

```typescript
// Simple message
yield* Effect.log("Processing started")

// With structured data
yield* Effect.log("Processing order", { orderId, userId, amount, currency })

// Levels
yield* Effect.logDebug("Cache lookup", { key, hit: true })
yield* Effect.logInfo("User logged in", { userId })
yield* Effect.logWarning("Rate limit approaching", { current: 95, limit: 100 })
yield* Effect.logError("Payment failed", { orderId, reason: error.message })
yield* Effect.logFatal("Database connection lost")
```

### Annotations and Spans

`Effect.annotateLogs` attaches metadata to every log line inside an effect;
`Effect.withLogSpan` adds a duration measurement:

```typescript
const handler = program.pipe(
  Effect.annotateLogs({ service: "checkout-api", route: "POST /checkout" }),
  Effect.withLogSpan("checkout"), // each line carries checkout=<N>ms
)
```

Prefer annotations over repeating the same field in every `Effect.log` call —
they compose down the call tree and can't drift.

### Logging in Services

```typescript
const processOrder = Effect.fn("OrderService.processOrder")(function* (
  input: OrderInput,
) {
  yield* Effect.log("Starting order processing", { orderId: input.orderId })

  return yield* validateAndProcess(input).pipe(
    Effect.tap(() => Effect.log("Order processed successfully")),
    Effect.tapError((err) =>
      Effect.logError("Order processing failed", {
        orderId: input.orderId,
        error: err._tag,
        message: err.message,
      }),
    ),
  )
})
```

Use `Effect.tapCause` (v3's `tapErrorCause`) when you also want defects and
interrupts.

## Effect.fn for Automatic Tracing

**Always use `Effect.fn`** for service methods — it creates a span with the name
you give it (via `Effect.withSpan` under the hood) and improves stack traces:

```typescript
// Creates span "UserService.findById"
const findById = Effect.fn("UserService.findById")(function* (id: UserId) {
  // Automatic span with start/end timing and error capture
})

// Creates span "PaymentService.processPayment"
const processPayment = Effect.fn("PaymentService.processPayment")(function* (
  orderId: OrderId,
  amount: number,
) {
  // ...
})
```

Extra combinators are **additional arguments** to `Effect.fn`, not a `.pipe` on
the result:

```typescript
const findById = Effect.fn("UserRepo.findById")(
  function* (id: UserId) { /* … */ },
  Effect.annotateLogs({ method: "findById" }),
  Effect.mapError((cause) => new UserRepoError({ cause })),
)
```

### Naming Convention

Use `ServiceName.methodName` consistently:

- `UserService.findById`
- `OrderService.create`
- `PaymentService.refund`
- `NotificationService.sendEmail`

## Span Annotations

```typescript
const processOrder = Effect.fn("OrderService.process")(function* (
  orderId: OrderId,
) {
  // ✅ GOOD - important business identifiers
  yield* Effect.annotateCurrentSpan("orderId", orderId)
  yield* Effect.annotateCurrentSpan("userId", order.userId)
  yield* Effect.annotateCurrentSpan("totalAmount", order.total)

  // ❌ BAD - noise
  // yield* Effect.annotateCurrentSpan("step", "validating")
  // yield* Effect.annotateCurrentSpan("item0Name", order.items[0].name)
})
```

**Do annotate:** entity IDs, important business values (amounts, statuses),
error context when failing.

**Don't annotate:** step-by-step progress, individual item details, internal
implementation state, sensitive data (PII, secrets).

Tracing can be turned off wholesale via the `References.TracerEnabled`
reference — handy for benchmarks and hot paths.

## Metrics

### Counter

```typescript
import { Effect, Metric } from "effect"

// Define metrics at module level
const ordersProcessed = Metric.counter("orders_processed", {
  description: "Total orders processed",
  incremental: true,
})

const ordersFailed = Metric.counter("orders_failed", {
  description: "Total orders that failed processing",
  incremental: true,
})

const processOrder = Effect.fn("OrderService.process")(function* (
  input: OrderInput,
) {
  return yield* process(input).pipe(
    // v3's Metric.increment is gone — use Metric.update
    Effect.tap(() => Metric.update(ordersProcessed, 1)),
    Effect.tapError(() => Metric.update(ordersFailed, 1)),
  )
})
```

`incremental: true` declares a monotonically increasing counter, which lets
exporters treat it correctly. Pass `bigint: true` for counters that can exceed
`Number.MAX_SAFE_INTEGER`.

### Attributes (formerly tags)

```typescript
const httpRequests = Metric.counter("http_requests_total", {
  description: "Total HTTP requests",
})

// Attach attributes to a metric — returns a new metric
yield* Metric.update(
  httpRequests.pipe(
    Metric.withAttributes({
      method: request.method,
      status: String(response.status),
      path: route.pattern, // the ROUTE, not the raw path — avoid unbounded cardinality
    }),
  ),
  1,
)
```

You can also set ambient attributes for a whole subtree via the
`Metric.CurrentMetricAttributes` reference, which is the right place for
`{ service, version, region }`.

### Gauge

```typescript
const activeConnections = Metric.gauge("active_connections", {
  description: "Number of active connections",
})

// Set the current value
yield* Metric.update(activeConnections, connectionCount)

// Relative change
yield* Metric.modify(activeConnections, 1)
yield* Metric.modify(activeConnections, -1)
```

### Histogram and Timer

```typescript
const requestDuration = Metric.histogram("request_duration_ms", {
  description: "Request duration in milliseconds",
  boundaries: [10, 50, 100, 250, 500, 1000, 2500, 5000],
})

yield* Metric.update(requestDuration, durationMs)

// Metric.timer takes Duration input and defaults to exponential boundaries
const apiTimer = Metric.timer("api_request_duration", {
  description: "Duration of API requests",
})
yield* Metric.update(apiTimer, Duration.millis(120))
```

Boundary helpers: `Metric.linearBoundaries({ start, width, count })` and
`Metric.exponentialBoundaries({ start, factor, count })`.

### Runtime metrics

v4 can emit fiber runtime metrics. Enable them with
`Metric.enableRuntimeMetricsLayer` at the app root, or scope them to part of a
program with `Effect.enableRuntimeMetrics`.

## Configuration with Config

**Always use `Config` instead of `process.env`.**

### Basic Config

```typescript
import { Config, Effect, Layer } from "effect"

// Config is a lazy description, not a read
const config = Config.all({
  port: Config.port("PORT").pipe(Config.withDefault(3000)),
  host: Config.string("HOST").pipe(Config.withDefault("localhost")),
  // literals takes the values and the name together (not curried, as in v3)
  env: Config.literals(["development", "staging", "production"], "NODE_ENV"),
})

// Read it inside a layer
const ServerLive = Layer.unwrap(
  Effect.gen(function* () {
    const { port, host, env } = yield* config
    return Layer.succeed(ServerConfig, { port, host, env })
  }),
)
```

`Config` is `Yieldable` in v4 — `yield* config` works — but it is no longer an
`Effect` subtype, so combinators need `.asEffect()`.

### Config with Validation

`Config.validate` was removed. Validate with a Schema, which gives you the same
error reporting as the rest of your decoding:

```typescript
import { Config, Schema } from "effect"

const dbConfig = Config.all({
  host: Config.string("DB_HOST"),
  port: Config.schema(
    Schema.Int.check(Schema.isBetween(1, 65535)),
    "DB_PORT",
  ),
  database: Config.string("DB_NAME"),
  maxConnections: Config.schema(
    Schema.Int.check(Schema.isGreaterThan(0)),
    "DB_MAX_CONNECTIONS",
  ).pipe(Config.withDefault(10)),
})
```

For one-off checks, `Config.mapOrFail` takes a function returning
`Effect<B, ConfigError>`.

All config failures now surface as a single `ConfigError` wrapping either a
`SourceError` (couldn't read) or a `SchemaError` (read but invalid) — check
`error.cause` to distinguish.

### Redacted Config

```typescript
import { Config, Effect, Redacted } from "effect"

const secretConfig = Config.all({
  apiKey: Config.redacted("API_KEY"), // Redacted<string>
  dbPassword: Config.redacted("DB_PASSWORD"),
})

const program = Effect.gen(function* () {
  const { apiKey } = yield* secretConfig

  // Unwrap only at the point of use
  const key = Redacted.value(apiKey)

  // Logging the wrapper is safe — it renders as <redacted>
  yield* Effect.log("Config loaded", { apiKey })
})
```

Keep values `Redacted` as far into the call stack as you can; call
`Redacted.value` at the exact line that needs the plaintext.

### Nested Structure

```typescript
const appConfig = Config.all({
  server: Config.all({
    port: Config.port("PORT"),
    host: Config.string("HOST"),
  }).pipe(Config.nested("SERVER")),

  database: Config.all({
    url: Config.string("URL"),
    pool: Config.int("POOL_SIZE").pipe(Config.withDefault(10)),
  }).pipe(Config.nested("DATABASE")),

  features: Config.all({
    enableBeta: Config.boolean("ENABLE_BETA").pipe(Config.withDefault(false)),
    maxUploadSize: Config.int("MAX_UPLOAD_SIZE").pipe(
      Config.withDefault(10_485_760),
    ),
  }),
})
```

`Config.nested("SERVER")` prefixes the keys, so the above reads `SERVER_PORT`
and `SERVER_HOST` under the default env provider.

## Log Level and Logger Configuration

The minimum log level is a `Context.Reference` in v4, so you set it with a
layer:

```typescript
import { Config, Effect, Layer, Logger, References } from "effect"

// Static
const WarnAndAbove = Layer.succeed(References.MinimumLogLevel, "Warn")

// From config
const LogLevelLayer = Layer.unwrap(
  Effect.gen(function* () {
    const level = yield* Config.logLevel("LOG_LEVEL").pipe(
      Config.withDefault("Info" as const),
    )
    return Layer.succeed(References.MinimumLogLevel, level)
  }),
)
```

Loggers are installed with `Logger.layer([...])`:

```typescript
// Production: one JSON line per entry
export const JsonLoggerLayer = Logger.layer([Logger.consoleJson])

// Development: pretty
export const PrettyLoggerLayer = Logger.layer([Logger.consolePretty])

// To a file
export const FileLoggerLayer = Logger.layer([
  Logger.toFile(Logger.formatSimple, "app.log"),
]).pipe(Layer.provide(NodeFileSystem.layer))

// Pick per environment
export const LoggerLayer = Layer.unwrap(
  Effect.gen(function* () {
    const env = yield* Config.string("NODE_ENV").pipe(
      Config.withDefault("development"),
    )
    return env === "production" ? JsonLoggerLayer : PrettyLoggerLayer
  }),
)
```

Formats available as building blocks: `formatSimple`, `formatLogFmt`,
`formatStructured`, `formatJson`. `Logger.batched` wraps a logger so entries are
flushed in windows — the right shape for shipping to an external service.

## Exporting Telemetry (OTLP)

For new projects, use the lightweight OTLP modules built into core rather than
pulling in the OpenTelemetry SDK:

```typescript
import { Otlp } from "effect/unstable/observability"

const ObservabilityLayer = Otlp.layer({
  baseUrl: "http://localhost:4318",
  resource: { serviceName: "checkout-api" },
})
```

`OtlpTracer`, `OtlpLogger`, and `OtlpMetrics` are available individually if you
only want one signal. Use `@effect/opentelemetry`'s `NodeSdk` only when you must
integrate with an existing OpenTelemetry pipeline (auto-instrumentation, an
existing SDK config).

## Combining Observability

```typescript
const processOrder = Effect.fn("OrderService.process")(function* (
  input: OrderInput,
) {
  const startTime = yield* Clock.currentTimeMillis

  yield* Effect.annotateCurrentSpan("orderId", input.orderId)
  yield* Effect.annotateCurrentSpan("userId", input.userId)

  yield* Effect.log("Processing order", { orderId: input.orderId })

  return yield* process(input).pipe(
    Effect.tap(() =>
      Effect.gen(function* () {
        const duration = (yield* Clock.currentTimeMillis) - startTime

        yield* Metric.update(orderProcessingDuration, duration)
        yield* Metric.update(ordersProcessed, 1)

        yield* Effect.log("Order processed", {
          orderId: input.orderId,
          durationMs: duration,
        })
      }),
    ),
    Effect.tapError((err) =>
      Effect.gen(function* () {
        yield* Metric.update(ordersFailed, 1)
        yield* Effect.logError("Order processing failed", {
          orderId: input.orderId,
          error: err._tag,
        })
      }),
    ),
  )
})
```

A note on effort: most of this is already free. `Effect.fn` gives you the span
and its timing; the logger already stamps duration if you use
`Effect.withLogSpan`. Reach for manual `Clock` arithmetic only when you need the
number as a *metric*, as above.
