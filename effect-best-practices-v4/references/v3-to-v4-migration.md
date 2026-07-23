# v3 → v4 Migration Reference

Condensed from the official migration guides in the Effect monorepo
(`MIGRATION.md` + `migration/*.md` on `main`). Use this when converting an
existing v3 codebase, or when a v3 habit produces a type error in v4.

## The five changes that bite hardest

1. **`Effect.Service` is gone.** Every service is now `Context.Service`, with an
   explicit `static readonly layer`. No `dependencies`, no auto `.Default`, no
   accessors.
2. **`Schema.TaggedError` → `Schema.TaggedErrorClass`.**
3. **`catchTag` takes an array, not varargs**: `catchTag(["A", "B"], f)`.
   And `catchAll` → `catch`.
4. **Packages consolidated into `effect/unstable/*`.** `@effect/platform`,
   `@effect/rpc`, `@effect/cluster`, `@effect/workflow`, `@effect/experimental`,
   `@effect/cli`, `@effect/sql` (core) no longer exist as imports.
5. **Effect subtyping replaced by `Yieldable`.** `Ref`, `Deferred`, and `Fiber`
   are no longer yieldable at all; `Option`, `Result`, and `Config` are
   yieldable but are not `Effect`s.

## Versioning

All Effect ecosystem packages share **one version number** and release together.
If you use `effect@4.0.0-beta.101`, the matching SQL driver is
`@effect/sql-pg@4.0.0-beta.101`.

Packages that remain separate:

- `@effect/platform-*` — platform bindings (node, bun, browser)
- `@effect/sql-*` — SQL driver packages
- `@effect/ai-*` — AI provider packages
- `@effect/opentelemetry` — OpenTelemetry integration
- `@effect/atom-*` — framework-specific atom bindings
- `@effect/vitest` — Vitest testing utilities

## Unstable modules

`effect/unstable/*` may receive breaking changes in **minor** releases; the rest
of `effect/*` follows strict semver. Unstable namespaces today: `ai`, `cli`,
`cluster`, `devtools`, `encoding`, `eventlog`, `http`, `httpapi`, `jsonschema`,
`observability`, `persistence`, `process`, `reactivity`, `rpc`, `schema`,
`socket`, `sql`, `workflow`, `workers`.

## Import map (the ones you'll actually hit)

| v3                                | v4                                                |
| --------------------------------- | ------------------------------------------------- |
| `@effect/schema`                  | `effect` (already true in late v3)                |
| `@effect/platform/HttpApi*`       | `effect/unstable/httpapi/HttpApi*`                |
| `@effect/platform/HttpClient`     | `effect/unstable/http/HttpClient`                 |
| `@effect/platform/HttpServer*`    | `effect/unstable/http/HttpServer*`                |
| `@effect/platform/HttpApp`        | `effect/unstable/http/HttpEffect`                 |
| `@effect/platform/KeyValueStore`  | `effect/unstable/persistence/KeyValueStore`       |
| `@effect/platform/FileSystem`     | `effect/FileSystem`                               |
| `@effect/platform/Path`           | `effect/Path`                                     |
| `@effect/platform/Error`          | `effect/PlatformError`                            |
| `@effect/platform/Command`        | `effect/unstable/process/ChildProcess`            |
| `@effect/platform/Worker`         | `effect/unstable/workers/Worker`                  |
| `@effect/platform/Socket`         | `effect/unstable/socket/Socket`                   |
| `@effect/rpc/*`                   | `effect/unstable/rpc/*`                           |
| `@effect/cluster/*`               | `effect/unstable/cluster/*`                       |
| `@effect/workflow/*`              | `effect/unstable/workflow/*`                      |
| `@effect/sql/SqlClient`           | `effect/unstable/sql/SqlClient`                   |
| `@effect/sql/Model`               | `effect/unstable/sql/SqlModel` (or `schema/Model`)|
| `@effect/experimental/Persistence`| `effect/unstable/persistence/Persistence`         |
| `@effect/experimental/EventLog`   | `effect/unstable/eventlog/EventLog`               |
| `@effect/experimental/DevTools`   | `effect/unstable/devtools/DevTools`               |
| `@effect/opentelemetry/Otlp*`     | `effect/unstable/observability/Otlp*`             |
| `@effect/cli/Command`             | `effect/unstable/cli/Command`                     |
| `@effect/cli/Args`                | `effect/unstable/cli/Argument`                    |
| `@effect/cli/Options`             | `effect/unstable/cli/Flag`                        |
| `@effect-atom/atom-react`         | `@effect/atom-react` (+ `effect/unstable/reactivity`) |
| `effect/Either`                   | `effect/Result`                                   |
| `effect/FiberRef`                 | `effect/References`                               |
| `effect/JSONSchema`               | `effect/JsonSchema`                               |
| `effect/ParseResult`              | `effect/SchemaIssue` + `effect/SchemaParser`      |
| `effect/TestClock`, `effect/FastCheck` | `effect/testing/*`                           |
| `effect/TRef`, `TMap`, `TSet`, …  | `effect/TxRef`, `TxHashMap`, `TxHashSet`, …       |
| `effect/Mailbox`                  | `effect/Queue`                                    |
| `@effect/typeclass/Semigroup`     | `effect/Combiner`                                 |
| `@effect/typeclass/Monoid`        | `effect/Reducer`                                  |

Every one of these also has a barrel: `effect`, `effect/unstable/http`,
`effect/unstable/httpapi`, `effect/testing`, etc.

## Services: `Context.Tag` / `Effect.Service` → `Context.Service`

| v3                                    | v4                                      |
| ------------------------------------- | --------------------------------------- |
| `Context.GenericTag<T>(id)`           | `Context.Service<T>(id)`                |
| `Context.Tag(id)<Self, Shape>()`      | `Context.Service<Self, Shape>()(id)`    |
| `Effect.Tag(id)<Self, Shape>()`       | `Context.Service<Self, Shape>()(id)`    |
| `Effect.Service<Self>()(id, opts)`    | `Context.Service<Self>()(id, { make })` |
| `Context.Reference<Self>()(id, opts)` | `Context.Reference<T>(id, opts)`        |

Note the argument order flip: in v3 the id came first
(`Context.Tag("Db")<Self, Shape>()`); in v4 the type params come first and the
id last (`Context.Service<Self, Shape>()("Db")`).

**v3**

```ts
class Logger extends Effect.Service<Logger>()("Logger", {
  effect: Effect.gen(function* () {
    const config = yield* Config
    return { log: (msg: string) => Effect.log(`[${config.prefix}] ${msg}`) }
  }),
  dependencies: [Config.Default],
}) {}
```

**v4**

```ts
class Logger extends Context.Service<Logger, {
  log(msg: string): Effect.Effect<void>
}>()("myapp/Logger") {
  static readonly layer = Layer.effect(
    Logger,
    Effect.gen(function* () {
      const config = yield* AppConfig
      return Logger.of({
        log: (msg: string) => Effect.log(`[${config.prefix}] ${msg}`),
      })
    }),
  ).pipe(Layer.provide(AppConfig.layer))
}
```

`Context.Service` also accepts a `make` option that stores the constructor
effect on the class, but it does **not** auto-generate a layer:

```ts
class Logger extends Context.Service<Logger>()("myapp/Logger", {
  make: Effect.gen(function* () {
    /* … */
  }),
}) {
  static readonly layer = Layer.effect(this, this.make).pipe(
    Layer.provide(AppConfig.layer),
  )
}
```

### Accessors are gone

v3's `Effect.Tag` proxy erased generics and overloads, so it was removed. The
replacements:

```ts
// Preferred — explicit, dependencies visible at the call site
const program = Effect.gen(function* () {
  const notifications = yield* Notifications
  yield* notifications.notify("hello")
})

// One-liner alternative (leaks the dependency out of view — use sparingly)
const program2 = Notifications.use((n) => n.notify("hello"))

// Pure accessor callback
const port = AppConfig.useSync((c) => c.port)
```

### Layer naming convention

v4 uses `layer` instead of v3's `Default` / `Live`. Variants get descriptive
suffixes: `layerTest`, `layerConfig`, `layerNoDeps`, `layerInMemory`.

## Error handling renames

| v3                       | v4                        |
| ------------------------ | ------------------------- |
| `Effect.catchAll`        | `Effect.catch`            |
| `Effect.catchAllCause`   | `Effect.catchCause`       |
| `Effect.catchAllDefect`  | `Effect.catchDefect`      |
| `Effect.catchSome`       | `Effect.catchFilter`      |
| `Effect.catchSomeCause`  | `Effect.catchCauseFilter` |
| `Effect.catchSomeDefect` | removed                   |
| `Effect.catchTag`        | unchanged (array syntax)  |
| `Effect.catchTags`       | unchanged                 |
| `Effect.catchIf`         | unchanged                 |

`catchFilter` uses the new `Filter` module instead of returning an `Option`:

```ts
// v3
Effect.catchSome((e) => (e === 42 ? Option.some(recover) : Option.none()))

// v4
Effect.catchFilter(
  Filter.fromPredicate((e: number) => e === 42),
  (e) => Effect.succeed("caught"),
)
```

New in v4:

- `Effect.catchReason(errorTag, reasonTag, handler)` — handle a tagged `reason`
  nested inside a tagged error, without removing the parent error from the error
  channel.
- `Effect.catchReasons(errorTag, cases)` — the object-of-handlers variant.
- `Effect.catchEager(handler)` — evaluates synchronous recovery immediately.

## Other Effect renames

| v3                            | v4                       |
| ----------------------------- | ------------------------ |
| `Effect.async`                | `Effect.callback`        |
| `Effect.zipRight`             | `Effect.andThen`         |
| `Effect.zipLeft`              | `Effect.tap`             |
| `Effect.either`               | `Effect.result`          |
| `Effect.tapErrorCause`        | `Effect.tapCause`        |
| `Effect.ignoreLogged`         | `Effect.ignore`          |
| `Effect.optionFromOptional`   | `Effect.catchNoSuchElement` |
| `Effect.makeSemaphore`        | `Semaphore.make`         |
| `Effect.makeLatch`            | `Latch.make`             |
| `Effect.fork`                 | `Effect.forkChild`       |
| `Layer.scoped`                | `Layer.effect`           |
| `Layer.scopedDiscard`         | `Layer.effectDiscard`    |
| `Layer.tapErrorCause`         | `Layer.tapCause`         |
| `Scope.extend`                | `Scope.provide`          |
| `Either.right` / `Either.left`| `Result.succeed` / `Result.fail` |

Stream also renamed a family of combinators — `Stream.catchAll` →
`Stream.catch`, `Stream.async` → `Stream.callback`, `Stream.fromChunk` →
`Stream.fromArray`, `Stream.mapChunks` → `Stream.mapArray`,
`Stream.repeatEffect` → `Stream.fromEffectRepeat`,
`Stream.repeatEffectWithSchedule` → `Stream.fromEffectSchedule`,
`Stream.Context` → `Stream.Services`.

## Yieldable: what stopped being an Effect

v3 made many types structural subtypes of `Effect`. v4 introduces the narrower
`Yieldable` trait: you can `yield*` it in a generator, but it is **not**
assignable to `Effect`.

Still yieldable: `Effect`, `Option`, `Result`, `Config`, `Context.Service`,
`Schema.TaggedErrorClass` instances.

**No longer yieldable at all:**

```ts
// v3                      // v4
yield * ref //             yield* Ref.get(ref)
yield * deferred //        yield* Deferred.await(deferred)
yield * fiber //           yield* Fiber.join(fiber)
```

Passing a `Yieldable` to a combinator needs an explicit conversion:

```ts
// v3: Option is an Effect subtype
Effect.map(Option.some(42), (n) => n + 1)

// v4
Effect.map(Option.some(42).asEffect(), (n) => n + 1)
```

## Generators: passing `this`

```ts
// v3
Effect.gen(this, function* () {})

// v4
Effect.gen({ self: this }, function* () {})
```

## Schema highlights

The full table lives in `schema-patterns.md`. The renames you hit constantly:

| v3                       | v4                                          |
| ------------------------ | ------------------------------------------- |
| `Schema.TaggedError`     | `Schema.TaggedErrorClass`                   |
| `Schema.Schema.Type<T>`  | `typeof MySchema.Type`                      |
| `annotations(a)`         | `annotate(a)`                               |
| `Schema.UUID`            | `Schema.String.check(Schema.isUUID())`      |
| `Schema.ULID`            | `Schema.String.check(Schema.isULID())`      |
| `filter(pred)`           | `check(Schema.makeFilter(pred))`            |
| `pattern(re)`            | `check(Schema.isPattern(re))`               |
| `minLength(n)`           | `check(Schema.isMinLength(n))`              |
| `nonEmptyString`         | `check(Schema.isNonEmpty())`                |
| `Schema.Union(A, B)`     | `Schema.Union([A, B])`                      |
| `Schema.Tuple(A, B)`     | `Schema.Tuple([A, B])`                      |
| `Schema.Literal("a","b")`| `Schema.Literals(["a", "b"])`               |
| `Record({key, value})`   | `Record(key, value)`                        |
| `pick("a")`              | `mapFields(Struct.pick(["a"]))`             |
| `omit("a")`              | `mapFields(Struct.omit(["a"]))`             |
| `extend(B)`              | `pipe(Schema.fieldsAssign(fieldsB))`        |
| `compose(B)`             | `decodeTo(B)`                               |
| `transform(from,to,…)`   | `from.pipe(decodeTo(to, SchemaTransformation.transform({…})))` |
| `decodeUnknown`          | `decodeUnknownEffect`                       |
| `decodeUnknownEither`    | `decodeUnknownExit`                         |
| `validate*`              | removed — use `decode*` + `Schema.toType`   |
| `Schema.Data(s)`         | removed — `Equal.equals` is deep by default |
| `ParseResult.ArrayFormatter` | `SchemaIssue.makeFormatterStandardSchemaV1()` |

Filters are all `is`-prefixed now: `isGreaterThan`, `isLessThan`, `isBetween`,
`isInt`, `isMultipleOf`, `isFinite`, `isMinLength`, `isMaxLength`,
`isLengthBetween`. `positive`, `negative`, `nonNegative`, and `nonPositive` were
removed — use `isGreaterThan(0)` etc.

`*FromSelf` schemas dropped the suffix: `DateFromSelf` → `Date`,
`OptionFromSelf` → `Option`, `ChunkFromSelf` → `Chunk`, and so on. Meanwhile
`Schema.Redacted` now means the old `RedactedFromSelf`; the old `Redacted`
behavior is `RedactedFromValue`.

## Config

| v3                          | v4                                        |
| --------------------------- | ----------------------------------------- |
| `Config.integer`            | `Config.int`                              |
| `Config.validate({…})`      | `Config.schema(codec, path)` or `mapOrFail` |
| `ConfigError` (many tags)   | single `ConfigError` wrapping `SourceError \| SchemaError` |

```ts
// v4 — validate with a Schema
const maxRetries = Config.schema(
  Schema.Int.check(Schema.isGreaterThan(0)),
  "MAX_RETRIES",
)
```

## Metric

`Metric.increment` was removed. Use `Metric.update`:

```ts
const counter = Metric.counter("orders_processed", { incremental: true })
yield* Metric.update(counter, 1)
```

## Cause

`Cause` is now a flat list of failures rather than a recursive tree. Use
`Cause.findError`, `Cause.squash`, and `Cause.failures` rather than pattern
matching on `Sequential`/`Parallel` nodes.

## Runtime

`Runtime<R>` was removed. Build a `ManagedRuntime` from your application `Layer`
and use it at the boundary with non-Effect code.

## Layer memoization

Layers are memoized across separate `Effect.provide` calls within the same
runtime, not just within a single layer graph. This means a shared dependency
provided in two places is constructed once — a behavior change from v3 worth
knowing when you relied on isolation.
