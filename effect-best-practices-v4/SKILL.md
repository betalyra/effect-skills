---
name: effect-best-practices-v4
description: Enforces Effect-TS v4 patterns for services, errors, layers, and atoms. Use when writing code with Context.Service, Schema.TaggedErrorClass, Layer composition, effect/unstable/* modules, or @effect/atom-react components.
---

# Effect-TS v4 Best Practices

This skill enforces opinionated, consistent patterns for **Effect v4** codebases
(`effect@4.x`). These patterns optimize for type safety, testability,
observability, and maintainability.

> **Using Effect v3?** Use the `effect-best-practices-v3` skill instead. The two
> versions differ substantially in service definition, error classes, module
> paths, and Schema APIs.

## Core Principles

### Effect Type Signature

```
Effect<Success, Error, Requirements>
//      ↑        ↑       ↑
//      |        |       └── Dependencies (provided via Layers)
//      |        └── Expected errors (typed, must be handled)
//      └── Success value
```

### Data-First Piped Style

**ALWAYS** prefer data-first pipe style for composition:

```typescript
// ✅ GOOD: Data-first with pipe
const result = value.pipe(
  Effect.map((n) => n * 2),
  Effect.flatMap((n) => processValue(n)),
  Effect.catchTag("NetworkError", () => Effect.succeed(fallback)),
)

// ❌ BAD: Function-first style
const result = Effect.catchTag(
  Effect.flatMap(
    Effect.map(value, (n) => n * 2),
    (n) => processValue(n),
  ),
  "NetworkError",
  () => Effect.succeed(fallback),
)
```

### Single Version, Consolidated Packages

In v4 the whole ecosystem shares **one version number**. `effect@4.0.0-beta.N`
pairs with `@effect/platform-node@4.0.0-beta.N`, `@effect/sql-pg@4.0.0-beta.N`,
etc.

Most of what used to be separate packages now lives inside `effect` itself,
under `effect/unstable/*`:

```typescript
// ✅ GOOD (v4)
import { Context, Effect, Layer, Schema } from "effect"
import { HttpApi, HttpApiEndpoint, HttpApiSchema } from "effect/unstable/httpapi"
import { HttpClient } from "effect/unstable/http"
import { Rpc, RpcGroup } from "effect/unstable/rpc"
import { SqlClient } from "effect/unstable/sql"
import { TestClock } from "effect/testing"

// ❌ BAD — these packages no longer exist in v4
import { Schema } from "@effect/schema"
import { HttpApi } from "@effect/platform"
import { RpcGroup } from "@effect/rpc"
import { Workflow } from "@effect/workflow"
```

Modules under `unstable/` may break in minor releases; they graduate to the
top-level `effect/*` namespace as they stabilize.

Packages that stay separate: `@effect/platform-*`, `@effect/sql-*`,
`@effect/ai-*`, `@effect/opentelemetry`, `@effect/atom-*`, `@effect/vitest`.

See `references/v3-to-v4-migration.md` for the complete import and API rename
map.

## Quick Reference: Critical Rules

| Category          | DO                                                             | DON'T                                          |
| ----------------- | -------------------------------------------------------------- | ---------------------------------------------- |
| Services          | `Context.Service<Self, Shape>()(id)`                           | `Effect.Service` / `Context.Tag` (v3 only)     |
| Service access    | `yield* MyService` in `Effect.gen`                             | `MyService.method(...)` accessors (removed)    |
| Layers            | `static readonly layer` on the service class                   | `Default` / `Live` naming, `dependencies: []`  |
| Dependencies      | `Layer.provide([Dep.layer, ...])` on the service's layer       | Wiring deps ad-hoc at every usage site         |
| Errors            | `Schema.TaggedErrorClass` with `message` and `cause` fields    | Plain classes or generic `Error`               |
| Error Specificity | `UserNotFoundError`, `SessionExpiredError`                     | Generic `NotFoundError`, `BadRequestError`     |
| Error Handling    | `catchTag("Tag", …)` / `catchTag(["A", "B"], …)` / `catchTags` | `Effect.catch` (v4's `catchAll`) or `mapError` |
| Multi-tag catch   | `catchTag(["A", "B"], handler)` — **array**                    | `catchTag("A", "B", handler)` — v3 variadic    |
| IDs               | `Schema.String.check(Schema.isUUID()).pipe(Schema.brand(…))`   | Plain `string`; `Schema.UUID` (removed)        |
| Functions         | `Effect.fn("Service.method")(function* …)`                     | Functions returning a bare `Effect.gen`        |
| Logging           | `Effect.log` with structured data                              | `console.log`                                  |
| Config            | `Config.*` with `Config.schema` / `mapOrFail`                  | `process.env` directly; `Config.validate`      |
| Options           | `Option.match` with both cases                                 | `Option.getOrThrow`                            |
| Nullability       | `Option<T>` in domain types                                    | `null`/`undefined`                             |
| Ref / Fiber       | `Ref.get(ref)`, `Fiber.join(fiber)`                            | `yield* ref`, `yield* fiber` (v3 subtyping)    |
| Atoms             | `Atom.make` outside components                                 | Creating atoms inside render                   |
| Atom State        | `Atom.keepAlive` for global state                              | Forgetting keepAlive for persistent state      |
| Atom Updates      | `useAtomSet` in React components                               | Mutating the registry imperatively from React  |
| Atom Cleanup      | `get.addFinalizer()` for side effects                          | Missing cleanup for event listeners            |
| Atom Results      | `AsyncResult.builder` with `onErrorTag`                        | Ignoring loading/error states                  |

## Service Definition Pattern

**Always use `Context.Service`.** In v4, `Effect.Service`, `Context.Tag`,
`Context.GenericTag`, and `Effect.Tag` are all gone — `Context.Service` replaces
all of them.

The idiomatic shape is: declare the **interface** as the second type parameter,
then attach the implementation as a **static `layer`** built with `Layer.effect`.

```typescript
import { Context, Effect, Layer, Option, Schema } from "effect"

export class UserService extends Context.Service<UserService, {
  findById(id: UserId): Effect.Effect<User, UserNotFoundError>
  create(data: CreateUserInput): Effect.Effect<User, UserCreateError>
}>()("myapp/users/UserService") {
  // The layer without its dependencies wired — keeps requirements visible,
  // and is what tests provide fakes to.
  static readonly layerNoDeps = Layer.effect(
    UserService,
    Effect.gen(function* () {
      const repo = yield* UserRepo
      const cache = yield* CacheService

      const findById = Effect.fn("UserService.findById")(function* (id: UserId) {
        const cached = yield* cache.get(id)
        if (Option.isSome(cached)) return cached.value

        const user = yield* repo.findById(id)
        yield* cache.set(id, user)
        return user
      })

      const create = Effect.fn("UserService.create")(function* (
        data: CreateUserInput,
      ) {
        const user = yield* repo.create(data)
        yield* Effect.log("User created", { userId: user.id })
        return user
      })

      return UserService.of({ findById, create })
    }),
  )

  // The production layer, with dependencies provided.
  static readonly layer = this.layerNoDeps.pipe(
    Layer.provide([UserRepo.layer, CacheService.layer]),
  )
}

// Usage — always yield the service, then call its methods
const program = Effect.gen(function* () {
  const users = yield* UserService
  return yield* users.findById(userId)
})

// At app root
const MainLive = Layer.mergeAll(UserService.layer, OtherService.layer)
```

**Key differences from v3:**

- **No `accessors`.** `UserService.findById(id)` no longer works. Use
  `yield* UserService` (preferred) or `UserService.use((s) => s.findById(id))`.
  Accessors were removed because the proxy erased generics and overloads.
- **No `dependencies` option.** Wire dependencies with `Layer.provide` on the
  layer itself.
- **No auto-generated `.Default`.** You define `static readonly layer` yourself.
  Convention: `layer` for the primary layer, descriptive suffixes for variants
  (`layerTest`, `layerConfig`, `layerNoDeps`).
- **Identifier strings are namespaced** — `"myapp/users/UserService"`, not
  `"UserService"`, to avoid collisions across packages.

See `references/service-patterns.md` for the `make`-option variant,
`Context.Reference`, and capability-based services.

## Error Definition Pattern

**Always use `Schema.TaggedErrorClass`** (v3's `Schema.TaggedError` was renamed).
This makes errors serializable (required for RPC/HttpApi), gives them a
consistent structure, and makes them **yieldable** — no `Effect.fail()` wrapper
needed, since the instances implement `Yieldable` directly.

```typescript
import { Schema } from "effect"

export class UserNotFoundError extends Schema.TaggedErrorClass<UserNotFoundError>()(
  "UserNotFoundError",
  {
    userId: UserId,
    message: Schema.String,
  },
  // Annotations are the third argument. HTTP status is `httpApiStatus`.
  { httpApiStatus: 404 },
) {}

export class UserCreateError extends Schema.TaggedErrorClass<UserCreateError>()(
  "UserCreateError",
  {
    message: Schema.String,
    // Schema.Defect() carries an arbitrary underlying cause safely
    cause: Schema.optional(Schema.Defect()),
  },
  { httpApiStatus: 400 },
) {}
```

Raise them by yielding the instance directly — and **always `return yield*`**, so
TypeScript knows control flow stops there:

```typescript
const findById = Effect.fn("UserService.findById")(function* (id: UserId) {
  const maybeUser = yield* repo.findById(id)
  if (Option.isNone(maybeUser)) {
    return yield* new UserNotFoundError({ userId: id, message: "Not found" })
  }
  return maybeUser.value
})
```

**Error handling — use `catchTag`/`catchTags`:**

```typescript
// ✅ CORRECT - single tag
yield* repo.findById(id).pipe(
  Effect.catchTag(
    "DatabaseError",
    (err) => new UserNotFoundError({ userId: id, message: "Lookup failed" }),
  ),
)

// ✅ CORRECT - multiple tags, same handler. In v4 the tags go in an ARRAY.
yield* effect.pipe(
  Effect.catchTag(
    ["TokenExpiredError", "TokenInvalidError", "MissingTokenError"],
    () => new AuthError({ message: "Authentication failed" }),
  ),
)

// ✅ CORRECT - multiple tags, different handlers (use catchTags)
yield* effect.pipe(
  Effect.catchTags({
    DatabaseError: (err) =>
      new UserNotFoundError({ userId: id, message: err.message }),
    ValidationError: (err) =>
      new InvalidEmailError({ email: input.email, message: err.message }),
  }),
)

// ❌ WRONG - v3 variadic form, no longer type-checks in v4
Effect.catchTag("TokenExpiredError", "TokenInvalidError", handler)
```

**`catch*` renames in v4:** `catchAll` → `catch`, `catchAllCause` →
`catchCause`, `catchAllDefect` → `catchDefect`, `catchSome` → `catchFilter`,
`catchSomeCause` → `catchCauseFilter`. `catchTag`, `catchTags`, and `catchIf`
keep their names.

### Prefer Explicit Over Generic Errors

**Every distinct failure reason deserves its own error type.** Don't collapse
multiple failure modes into generic HTTP errors like `NotFoundError` or
`BadRequestError`.

- `UserNotFoundError` with `userId` → Frontend shows "User doesn't exist"
- `ChannelNotFoundError` with `channelId` → Frontend shows "Channel was deleted"
- `SessionExpiredError` with `expiredAt` → Frontend shows "Session expired"

Generic errors lose context and prevent targeted recovery. See
`references/error-patterns.md` for complete patterns including the v4-only
`reason` errors (`Effect.catchReason` / `Effect.catchReasons`), error remapping,
and retry strategies.

## Schema & Branded Types Pattern

**Brand all entity IDs** for type safety across service boundaries. Note that
v4 removed `Schema.UUID` in favour of a checked `Schema.String`:

```typescript
import { Schema } from "effect"

// Entity IDs - always branded
export const UserId = Schema.String.check(Schema.isUUID()).pipe(
  Schema.brand("@App/UserId"),
)
export type UserId = typeof UserId.Type

export const OrganizationId = Schema.String.check(Schema.isUUID()).pipe(
  Schema.brand("@App/OrganizationId"),
)
export type OrganizationId = typeof OrganizationId.Type

// Domain types - use Schema.Struct
export const User = Schema.Struct({
  id: UserId,
  email: Schema.String,
  name: Schema.String,
  organizationId: OrganizationId,
  createdAt: Schema.DateTimeUtc,
})
export type User = typeof User.Type

// Input types for mutations
export const CreateUserInput = Schema.Struct({
  email: Schema.String.check(Schema.isPattern(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)),
  name: Schema.String.check(Schema.isMinLength(1)),
  organizationId: OrganizationId,
})
export type CreateUserInput = typeof CreateUserInput.Type
```

**v4 Schema essentials:**

- Type extraction is `typeof MySchema.Type` / `typeof MySchema.Encoded`.
  `Schema.Schema.Type<typeof X>` is gone.
- Filters are `check(...)` with `is`-prefixed names: `minLength` →
  `isMinLength`, `pattern` → `isPattern`, `int` → `isInt`, `between` →
  `isBetween`, and so on.
- Variadic constructors became arrays: `Schema.Union([A, B])`,
  `Schema.Tuple([A, B])`, `Schema.Literals(["a", "b"])`.
- `Schema.Record(key, value)` takes positional arguments.
- `annotations(...)` → `annotate(...)`.
- Decoding entrypoints: `decodeUnknownEffect`, `decodeEffect`,
  `decodeUnknownSync`, `decodeUnknownExit`. The `validate*` family was removed —
  use `decode*` on `Schema.toType(schema)`.

**When NOT to brand:**

- Simple strings that don't cross service boundaries (URLs, file paths)
- Primitive config values

See `references/schema-patterns.md` for transforms, `Schema.Class`, and the
full v3→v4 Schema rename table.

## Function Pattern with Effect.fn

**Always use `Effect.fn`** for service methods. This provides automatic tracing
with proper span names and better stack traces:

```typescript
// ❌ BAD - arrow function returning a generic effect
const findById = (id: UserId) =>
  Effect.gen(function* () {
    // ...
  })

// ✅ CORRECT - Effect.fn with descriptive name
const findById = Effect.fn("UserService.findById")(function* (id: UserId) {
  yield* Effect.annotateCurrentSpan("userId", id)
  const user = yield* repo.findById(id)
  return user
})

// ✅ CORRECT - extra combinators are passed as ADDITIONAL ARGUMENTS to
// Effect.fn, not chained with .pipe on the result
const transfer = Effect.fn("AccountService.transfer")(
  function* (fromId: AccountId, toId: AccountId, amount: number) {
    yield* Effect.annotateCurrentSpan("fromId", fromId)
    yield* Effect.annotateCurrentSpan("toId", toId)
    // ...
  },
  Effect.mapError((cause) => new TransferError({ cause })),
  Effect.annotateLogs({ method: "transfer" }),
)

// ✅ Annotate the return type explicitly with Effect.fn.Return
const load = Effect.fn("Repo.load")(
  function* (id: UserId): Effect.fn.Return<User, UserNotFoundError> {
    // ...
  },
)
```

## Layer Composition

**Wire dependencies on the service's own layer**, not at usage sites. `v4` has
no `dependencies` option — use `Layer.provide` (hides the dependency) or
`Layer.provideMerge` (also exposes it):

```typescript
export class OrderService extends Context.Service<OrderService, {
  place(input: PlaceOrderInput): Effect.Effect<Order, PlaceOrderError>
}>()("myapp/orders/OrderService") {
  static readonly layerNoDeps = Layer.effect(
    OrderService,
    Effect.gen(function* () {
      const users = yield* UserService
      const products = yield* ProductService
      const payments = yield* PaymentService
      // ...
      return OrderService.of({ place })
    }),
  )

  static readonly layer = this.layerNoDeps.pipe(
    Layer.provide([UserService.layer, ProductService.layer, PaymentService.layer]),
  )
}

// At app root - simple merge
const AppLive = Layer.mergeAll(
  OrderService.layer,
  // Infrastructure layers (intentionally provided at the root)
  DatabaseLive,
  RedisLive,
)
```

Also note: `Layer.scoped` → `Layer.effect` and `Layer.scopedDiscard` →
`Layer.effectDiscard` in v4. Every `Layer.effect` is scope-aware.

See `references/layer-patterns.md` for testing layers, `Layer.unwrap` for
config-dependent layers, and memoization semantics.

## Option Handling

**Never use `Option.getOrThrow`**. Always handle both cases explicitly:

```typescript
// ✅ CORRECT - explicit handling
yield* Option.match(maybeUser, {
  onNone: () => new UserNotFoundError({ userId, message: "Not found" }),
  onSome: (user) => Effect.succeed(user),
})

// ✅ CORRECT - with getOrElse for defaults
const name = Option.getOrElse(maybeName, () => "Anonymous")

// ✅ CORRECT - Option.map for transformations
const upperName = Option.map(maybeName, (n) => n.toUpperCase())
```

`Option` is still `Yieldable` in v4 — `yield* Option.some(42)` works inside
`Effect.gen` and fails with `NoSuchElementError` on `None`. But `Option` is **no
longer an `Effect` subtype**, so passing it to a combinator requires
`.asEffect()`:

```typescript
// ❌ v3 style — Option is not an Effect in v4
Effect.map(maybeUser, (u) => u.name)

// ✅ v4
Effect.map(maybeUser.asEffect(), (u) => u.name)
```

The same applies to `Ref`, `Deferred`, and `Fiber`, which are no longer
yieldable at all — use `Ref.get(ref)`, `Deferred.await(d)`, `Fiber.join(f)`.

## Effect Atom (Frontend State)

Effect Atom provides reactive state management for React. In v4 the core lives
in `effect/unstable/reactivity` and the React bindings in `@effect/atom-react`.

### Basic Atoms

```typescript
import { Atom } from "effect/unstable/reactivity"

// Define atoms OUTSIDE components
const countAtom = Atom.make(0)

// Use keepAlive for global state that should persist
const userPrefsAtom = Atom.make({ theme: "dark" }).pipe(Atom.keepAlive)

// Atom families for per-entity state
const modalAtomFamily = Atom.family((type: string) =>
  Atom.make({ isOpen: false }).pipe(Atom.keepAlive),
)
```

### React Integration

```typescript
import { useAtomValue, useAtomSet, useAtom, useAtomMount } from "@effect/atom-react"

function Counter() {
    const count = useAtomValue(countAtom)           // Read only
    const setCount = useAtomSet(countAtom)          // Write only
    const [value, setValue] = useAtom(countAtom)    // Read + write

    return <button onClick={() => setCount((c) => c + 1)}>{count}</button>
}

// Mount side-effect atoms without reading value
function App() {
    useAtomMount(keyboardShortcutsAtom)
    return <>{children}</>
}
```

### Handling Results with AsyncResult.builder

v3's `Result` is `AsyncResult` in v4. **Use `AsyncResult.builder`** for
rendering effectful atom results — it provides chainable, type-tracked error
handling with `onErrorTag`:

```typescript
import { AsyncResult } from "effect/unstable/reactivity"

function UserProfile() {
    const userResult = useAtomValue(userAtom) // AsyncResult<User, Error>

    return AsyncResult.builder(userResult)
        .onInitial(() => <div>Loading...</div>)
        .onErrorTag("NotFoundError", () => <div>User not found</div>)
        .onError((error) => <div>Error: {error.message}</div>)
        .onSuccess((user) => <div>Hello, {user.name}</div>)
        .render()
}
```

`onErrorTag` also accepts an array of tags: `.onErrorTag(["A", "B"], handler)`.

### Atoms with Side Effects

```typescript
const scrollYAtom = Atom.make((get) => {
  const onScroll = () => get.setSelf(window.scrollY)

  window.addEventListener("scroll", onScroll)
  get.addFinalizer(() => window.removeEventListener("scroll", onScroll)) // REQUIRED

  return window.scrollY
}).pipe(Atom.keepAlive)
```

See `references/effect-atom-patterns.md` for complete patterns including
families, localStorage, runtime atoms, and anti-patterns.

## RPC & Cluster Patterns

For RPC contracts and cluster workflows, see:

- `references/rpc-cluster-patterns.md` - RpcGroup, Workflow.make, Activity
  patterns, and the `effect/unstable/{rpc,cluster,workflow}` import paths

## Vercel AI SDK Integration

Use `Schema.toStandardSchemaV1` (v3's `Schema.standardSchemaV1`) to bridge Effect
schemas to the Vercel AI SDK's `inputSchema`. For tools with no arguments, use
`Schema.Record(Schema.String, Schema.Never)` — many providers reject empty
schemas. When tool `execute` functions need Effect services, capture the
services via `Effect.context<Deps>()` in a factory function and run with
`Effect.runPromiseWith(services)` — do not use bare `Effect.runPromise` with
unsatisfied dependencies.

Note the v4 shifts: `Effect.runtime<R>()` → `Effect.context<R>()`, and
`Runtime.runPromise(runtime)` → `Effect.runPromiseWith(services)` (the
`Runtime<R>` type was removed).

See `references/vercel-ai-sdk-patterns.md` for complete patterns.

## Anti-Patterns (Forbidden)

These patterns are **never acceptable**:

```typescript
// FORBIDDEN - runSync/runPromise inside services
const result = Effect.runSync(someEffect) // Never do this

// FORBIDDEN - throw inside Effect.gen
yield* Effect.gen(function* () {
  if (bad) throw new Error("No!") // yield the tagged error instead
})

// FORBIDDEN - Effect.catch (v4's catchAll) losing type info
yield* effect.pipe(Effect.catch(() => Effect.fail(new GenericError())))

// FORBIDDEN - console.log
console.log("debug") // Use Effect.log

// FORBIDDEN - process.env directly
const key = process.env.API_KEY // Use Config.string("API_KEY")

// FORBIDDEN - null/undefined in domain types
type User = { name: string | null } // Use Option<string>

// FORBIDDEN (v4) - v3 APIs that silently no longer exist
Effect.Service<T>()("T", { effect, dependencies }) // Use Context.Service
Schema.TaggedError<E>()("E", fields)               // Use Schema.TaggedErrorClass
Effect.catchAll(handler)                           // Use Effect.catch
yield* someRef                                     // Use Ref.get(someRef)
```

See `references/anti-patterns.md` for the complete list with rationale.

## Observability

```typescript
// Structured logging
yield* Effect.log("Processing order", { orderId, userId, amount })

// Metrics — v4 has no Metric.increment; use Metric.update
const orderCounter = Metric.counter("orders_processed", {
  description: "Orders processed",
  incremental: true,
})
yield* Metric.update(orderCounter, 1)

// Config with validation — Config.validate is gone. Validate with a Schema.
const config = Config.all({
  port: Config.port("PORT").pipe(Config.withDefault(3000)),
  apiKey: Config.redacted("API_KEY"),
  maxRetries: Config.schema(
    Schema.Int.check(Schema.isGreaterThan(0)),
    "MAX_RETRIES",
  ),
})
```

Note `Config.integer` → `Config.int`, and `Config.schema(codec, path)` is now
the primary way to build a validated `Config`.

For OTLP export, prefer `effect/unstable/observability` (`Otlp`, `OtlpTracer`,
`OtlpLogger`, `OtlpMetrics`) in new projects; use `@effect/opentelemetry`
`NodeSdk` only when integrating with an existing OpenTelemetry setup.

See `references/observability-patterns.md` for metrics and tracing patterns.

## Reference Files

For detailed patterns, consult these reference files in the `references/`
directory:

- `v3-to-v4-migration.md` - Import map, API renames, and the breaking changes
  that bite hardest
- `service-patterns.md` - `Context.Service`, `Effect.fn`, `Context.Reference`,
  capability-based services
- `error-patterns.md` - `Schema.TaggedErrorClass`, reason errors, error
  remapping, retry patterns
- `schema-patterns.md` - Branded types, `check` filters, transforms,
  `Schema.Class`
- `layer-patterns.md` - Dependency composition, testing layers,
  `provide` vs `provideMerge`, memoization
- `domain-predicates.md` - Equivalence, Order, typeclass-derived predicates
- `rpc-cluster-patterns.md` - RpcGroup, Workflow, Activity patterns
- `effect-atom-patterns.md` - Atom, families, React hooks, `AsyncResult`
- `anti-patterns.md` - Complete list of forbidden patterns
- `observability-patterns.md` - Logging, metrics, config patterns
- `effect-test-patterns.md` - Testing patterns with `@effect/vitest`
- `vercel-ai-sdk-patterns.md` - Vercel AI SDK tool definitions with Effect Schema
