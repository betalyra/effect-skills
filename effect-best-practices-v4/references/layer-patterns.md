# Layer Patterns (Effect v4)

## Table of Contents

- [Layer Structure](#layer-structure)
- [Wiring Dependencies Without `dependencies`](#wiring-dependencies-without-dependencies)
- [Infrastructure Layers](#infrastructure-layers)
- [Layer.mergeAll Over Nested Provides](#layermergeall-over-nested-provides)
- [Layer Naming Conventions](#layer-naming-conventions)
- [Layer.unwrap for Config-Dependent Layers](#layerunwrap-for-config-dependent-layers)
- [Scoped Layers](#scoped-layers)
- [Background Tasks with Layer.effectDiscard](#background-tasks-with-layereffectdiscard)
- [Testing Layer Composition](#testing-layer-composition)
- [Layer.effect vs Layer.succeed](#layereffect-vs-layersucceed)
- [Lazy Layers](#lazy-layers)
- [Merge vs Provide](#merge-vs-provide)
- [Layer Memoization (changed in v4)](#layer-memoization-changed-in-v4)
- [LayerMap for Dynamic Resources](#layermap-for-dynamic-resources)
- [Application Entry Points](#application-entry-points)

## Layer Structure

Understanding the Layer type signature:

```typescript
Layer<RequirementsOut, Error, RequirementsIn>
         ▲                ▲           ▲
         │                │           └─ What this layer needs
         │                └─ Errors during construction
         └─ What this layer produces
```

Example:

```typescript
// Layer<Logger, never, AppConfig>
//         ▲      ▲       ▲
//         │      │       └─ Needs AppConfig
//         │      └─ Cannot fail
//         └─ Produces Logger
export const LoggerLayer = Layer.effect(
  Logger,
  Effect.gen(function* () {
    const config = yield* AppConfig
    return Logger.of({
      log: (message: string) => Effect.log(`[${config.logLevel}] ${message}`),
    })
  }),
)
```

## Wiring Dependencies Without `dependencies`

v4's `Context.Service` has no `dependencies` option — the v3 escape hatch that
auto-wired a `.Default` layer is gone. You compose layers explicitly, which is
more code but makes the dependency graph visible and lets you build variants.

The pattern that scales: keep an "open" layer whose requirements are still in
the type, then a "closed" layer that provides them.

```typescript
export class OrderService extends Context.Service<OrderService, {
  create(input: CreateOrderInput): Effect.Effect<Order, CreateOrderError>
}>()("myapp/orders/OrderService") {
  static readonly layerNoDeps: Layer.Layer<
    OrderService,
    never,
    UserService | ProductService | InventoryService | PaymentService
  > = Layer.effect(
    OrderService,
    Effect.gen(function* () {
      const users = yield* UserService
      const products = yield* ProductService
      const inventory = yield* InventoryService
      const payments = yield* PaymentService
      // ...
      return OrderService.of({ create })
    }),
  )

  static readonly layer = this.layerNoDeps.pipe(
    Layer.provide([
      UserService.layer,
      ProductService.layer,
      InventoryService.layer,
      PaymentService.layer,
    ]),
  )
}

// At app root — simple, flat composition
const AppLive = Layer.mergeAll(
  OrderService.layer,
  NotificationService.layer,
  AnalyticsService.layer,
)
```

Annotating `layerNoDeps` with an explicit `Layer.Layer<…>` type is worth the
keystrokes: it turns "I accidentally pulled in a new dependency" into a compile
error at the definition site rather than a surprise at the app root.

### Wrong Pattern (Leaked Dependencies)

```typescript
// ❌ WRONG - only exposing the open layer forces every call site to wire deps
export class OrderService extends Context.Service<OrderService, {/*…*/}>()(
  "myapp/orders/OrderService",
) {
  static readonly layer = Layer.effect(OrderService, Effect.gen(function* () {
    const users = yield* UserService // requirement escapes
    // ...
  }))
}

// Every usage now has to remember the full graph
const program = useOrders.pipe(
  Effect.provide([
    OrderService.layer,
    UserService.layer,
    ProductService.layer,
    // Easy to forget one — and it's a type error at the outermost call site,
    // far from the cause
  ]),
)
```

## Infrastructure Layers

Infrastructure layers (Database, Redis, HTTP clients) are **acceptable** to
leave in the requirements channel because:

1. They're provided once at the application root
2. They don't change between test and production (different implementations,
   same interface)
3. They're true infrastructure, not business logic

```typescript
import { PgClient } from "@effect/sql-pg"
import { Config, Context, Effect, Layer } from "effect"
import { SqlClient } from "effect/unstable/sql"

const DatabaseLive = PgClient.layerConfig({
  url: Config.redacted("DATABASE_URL"),
})

// Services use the SqlClient but don't provide it themselves
export class UserRepo extends Context.Service<UserRepo, {
  findById(id: UserId): Effect.Effect<Option.Option<User>, UserRepoError>
}>()("myapp/users/UserRepo") {
  static readonly layerNoDeps = Layer.effect(
    UserRepo,
    Effect.gen(function* () {
      const sql = yield* SqlClient.SqlClient

      const findById = Effect.fn("UserRepo.findById")(
        function* (id: UserId) {
          const rows = yield* sql<User>`SELECT * FROM users WHERE id = ${id}`
          return Array.head(rows)
        },
        Effect.mapError((cause) => new UserRepoError({ cause })),
      )

      return UserRepo.of({ findById })
    }),
  )

  static readonly layer = this.layerNoDeps.pipe(Layer.provide(DatabaseLive))
}
```

Or provide infrastructure once for a whole subtree:

```typescript
const AppLive = Layer.mergeAll(
  OrderService.layerNoDeps,
  UserService.layerNoDeps,
).pipe(
  Layer.provide([DatabaseLive, RedisLive]),
)
```

## Layer.mergeAll Over Nested Provides

**Use `Layer.mergeAll`** for composing layers at the same level:

```typescript
// ✅ CORRECT - Flat composition
const ServicesLive = Layer.mergeAll(
  UserService.layer,
  OrderService.layer,
  ProductService.layer,
  NotificationService.layer,
)

const InfrastructureLive = Layer.mergeAll(
  DatabaseLive,
  RedisLive,
  HttpClientLive,
)

const AppLive = ServicesLive.pipe(Layer.provide(InfrastructureLive))
```

```typescript
// ❌ WRONG - Deeply nested, hard to read
const AppLive = UserService.layer.pipe(
  Layer.provide(
    OrderService.layer.pipe(
      Layer.provide(ProductService.layer.pipe(Layer.provide(DatabaseLive))),
    ),
  ),
)
```

`Layer.provide` also accepts an array, which flattens most nesting:

```typescript
const AppLive = ServicesLive.pipe(
  Layer.provide([DatabaseLive, RedisLive, HttpClientLive]),
)
```

## Layer Naming Conventions

v4 dropped the `Live` / `Default` convention in favour of `layer` as a static
member on the service class:

| Name             | Meaning                                          |
| ---------------- | ------------------------------------------------ |
| `layer`          | Primary, fully-wired production layer            |
| `layerNoDeps`    | The layer with requirements still open           |
| `layerTest`      | Test/fake implementation                         |
| `layerInMemory`  | In-memory variant                                |
| `layerConfig`    | Variant parameterised by config                  |

```typescript
// Test with a static fake
export const UserServiceTest = Layer.succeed(
  UserService,
  UserService.of({
    findById: () => Effect.succeed(mockUser),
    create: (input) =>
      Effect.succeed({ id: UserId.make("00000000-0000-4000-8000-000000000000"), ...input }),
  }),
)
```

Standalone layers not attached to a class can still take a suffix
(`DatabaseLive`, `InMemoryDatabaseLive`) — the point is consistency, not the
particular word.

## Layer.unwrap for Config-Dependent Layers

`Layer.unwrapEffect` was renamed to `Layer.unwrap`. Use it to *choose* or
*parameterise* a layer based on an Effect (usually config):

```typescript
import { Config, Effect, Layer } from "effect"

const ApiClientLive = Layer.unwrap(
  Effect.gen(function* () {
    const apiKey = yield* Config.redacted("API_KEY")
    const baseUrl = yield* Config.string("API_BASE_URL")
    const timeout = yield* Config.int("API_TIMEOUT").pipe(
      Config.withDefault(5000),
    )

    return Layer.succeed(ApiClient, new ApiClientImpl({ apiKey, baseUrl, timeout }))
  }),
)

// Choosing between implementations at startup
export const MessageStoreLayer = Layer.unwrap(
  Effect.gen(function* () {
    const useInMemory = yield* Config.boolean("MESSAGE_STORE_IN_MEMORY").pipe(
      Config.withDefault(false),
    )
    if (useInMemory) return MessageStore.layerInMemory

    const remoteUrl = yield* Config.url("MESSAGE_STORE_URL")
    return MessageStore.layerRemote(remoteUrl)
  }),
)
```

Note the `ConfigError` from reading config surfaces in the layer's **error**
channel — the resulting type is `Layer<MessageStore, ConfigError>`, which is
exactly what you want: startup misconfiguration fails loudly at layer build.

## Scoped Layers

`Layer.scoped` is gone. **`Layer.effect` is already scope-aware**, so
`Effect.acquireRelease` works directly:

```typescript
import { Effect, Layer } from "effect"

const DatabaseConnectionLayer = Layer.effect(
  DatabaseConnection,
  Effect.acquireRelease(
    Effect.gen(function* () {
      const pool = yield* createPool(config)
      yield* Effect.log("Database pool created")
      return pool
    }),
    (pool) =>
      Effect.gen(function* () {
        yield* pool.end()
        yield* Effect.log("Database pool closed")
      }).pipe(Effect.orDie),
  ),
)
```

The finalizer runs when the layer's scope closes — i.e. when the runtime built
from it shuts down.

## Background Tasks with Layer.effectDiscard

For a layer that only runs a side effect (a poller, a subscriber, a metrics
flusher) and exposes no service, use `Layer.effectDiscard` (v3's
`Layer.scopedDiscard`):

```typescript
const HeartbeatLayer = Layer.effectDiscard(
  Effect.gen(function* () {
    const client = yield* ApiClient
    yield* client.heartbeat.pipe(
      Effect.repeat(Schedule.spaced("30 seconds")),
      Effect.forkScoped, // tied to the layer's scope
    )
  }),
)
```

## Testing Layer Composition

```typescript
// test/setup.ts
import { Layer } from "effect"

export const TestLive = Layer.mergeAll(
  UserServiceTest,
  OrderServiceTest,
  ProductServiceTest,
).pipe(Layer.provide(InMemoryDatabaseLive))
```

### Partial mocks with `Layer.mock`

`Layer.mock` builds a layer that implements only the members you care about;
anything else throws `UnimplementedError` if the test happens to call it. This
is far better than filling in dummy implementations, because an unexpected call
fails the test loudly instead of silently succeeding.

```typescript
const testUserLayer = Layer.mock(UserService, {
  getUser: (id: string) => Effect.succeed({ id, name: "Test User" }),
  // deleteUser, updateUser omitted — calling them fails the test
})
```

See `effect-test-patterns.md` for `@effect/vitest`'s `layer(...)` block, which
builds a layer once per describe block and tears it down in `afterAll`.

## Layer.effect vs Layer.succeed

```typescript
// Layer.succeed — for static values (no effects)
const ConfigLive = Layer.succeed(AppConfig, {
  port: 3000,
  env: "development",
})

// Layer.sync — for a value computed lazily but synchronously
const ClockLive = Layer.sync(SystemClock, () => createClock())

// Layer.effect — when construction needs effects (or a scope)
const LoggerLayer = Layer.effect(
  Logger,
  Effect.gen(function* () {
    const config = yield* AppConfig
    const transport =
      config.env === "production"
        ? createCloudTransport()
        : createConsoleTransport()
    return Logger.of({ log: (m) => transport.write(m) })
  }),
)
```

## Lazy Layers

`Layer.lazy` became `Layer.suspend`. Use it to defer construction of the layer
*value* — useful for breaking circular module references or avoiding expensive
work at import time:

```typescript
const ExpensiveServiceLayer = Layer.suspend(() =>
  Layer.effect(
    ExpensiveService,
    Effect.gen(function* () {
      yield* Effect.log("Initializing expensive service...")
      const client = yield* createExpensiveClient()
      return ExpensiveService.of({ client })
    }),
  ),
)
```

## Merge vs Provide

### Merge (Parallel Composition)

Combine independent layers:

```typescript
// Layer<AppConfig | Logger, never, AppConfig>
//         ▲                  ▲       ▲
//         │                  │       └─ LoggerLayer needs AppConfig
//         │                  └─ No errors
//         └─ Produces both
const Combined = Layer.merge(ConfigLive, LoggerLayer)
```

- **Requirements**: union
- **Outputs**: union

### Provide (Sequential Composition)

Chain layers where one satisfies another's requirements, hiding the dependency:

```typescript
// Layer<Logger, never, never>
const FullLogger = Layer.provide(LoggerLayer, ConfigLive)
```

- **Requirements**: the outer layer's remaining requirements
- **Output**: only the outer layer's output

### ProvideMerge

Same, but keeps both outputs:

```typescript
// Layer<AppConfig | Logger, never, never>
const FullLogger = Layer.provideMerge(LoggerLayer, ConfigLive)
```

- **Requirements**: `never`
- **Output**: union

Rule of thumb: `provide` for implementation details you don't want callers
reaching into; `provideMerge` when the dependency is genuinely part of the
public surface (test refs, a SQL client the caller also needs).

### Layered Architecture Example

```typescript
// Infrastructure: no dependencies once Config is provided
const InfrastructureLive = Layer.mergeAll(
  DatabaseLive, // Layer<Database, never, AppConfig>
  CacheLive,    // Layer<Cache, never, AppConfig>
).pipe(Layer.provideMerge(ConfigLive))

// Domain: depends on infrastructure
const DomainLive = Layer.mergeAll(
  PaymentDomain.layerNoDeps,
  OrderDomain.layerNoDeps,
).pipe(Layer.provide(InfrastructureLive))

// Application: depends on domain
const ApplicationLive = Layer.mergeAll(
  PaymentGateway.layerNoDeps,
  NotificationService.layerNoDeps,
).pipe(Layer.provide(DomainLive))
```

## Layer Memoization (changed in v4)

In v3, each `Effect.provide` call had its own memoization scope, so two
`provide` calls with an overlapping layer built that layer **twice**.

In v4 the `MemoMap` is shared across `Effect.provide` calls, so layers are
deduplicated globally within a runtime.

```typescript
// v4: DatabaseLive is built ONCE, shared by both
const a = programA.pipe(Effect.provide(ServiceA.layer))
const b = programB.pipe(Effect.provide(ServiceB.layer))
```

If you genuinely need an isolated instance, opt out per call:

```typescript
program.pipe(Effect.provide(SomeLayer, { local: true }))
```

…or use `Layer.fresh(SomeLayer)` to force a new build of that layer wherever it
appears. Reach for these only when isolation is the point (per-tenant
connections, test fixtures) — the shared default is what you want almost always.

## LayerMap for Dynamic Resources

When resources are keyed by a runtime value (a tenant ID, a model name), use
`LayerMap.Service` rather than building layers by hand. It builds and caches one
layer per key, and releases them when no longer used:

```typescript
import { Effect, LayerMap } from "effect"

export class PoolMap extends LayerMap.Service<PoolMap>()("myapp/PoolMap", {
  // How to build a layer for each key
  lookup: (tenantId: string) => DatabasePool.layer(tenantId),
  // Released automatically after this much idle time
  idleTimeToLive: "1 minute",
}) {}

// Tenant-agnostic work — the right pool is supplied by the map
const queryUsers = Effect.gen(function* () {
  const pool = yield* DatabasePool
  return yield* pool.query("SELECT id, email FROM users ORDER BY id")
})

const program = Effect.gen(function* () {
  yield* queryUsers.pipe(Effect.provide(PoolMap.get("acme")))

  // Force a rebuild on next access
  yield* PoolMap.invalidate("acme")
}).pipe(Effect.provide(PoolMap.layer))
```

A static set of keys can be declared with the `layers` option instead of
`lookup`.

## Application Entry Points

Two idioms, depending on whether your app is a program or a long-running
service:

```typescript
// A program that finishes
import { NodeRuntime } from "@effect/platform-node"

NodeRuntime.runMain(main.pipe(Effect.provide(AppLive)))

// A long-running service — Layer.launch builds the layer and never returns
NodeRuntime.runMain(Layer.launch(ServerLive))
```

For bridging into non-Effect code (a framework's request handler, a worker
queue), build a `ManagedRuntime` from `AppLive` once and reuse it — `Runtime<R>`
was removed in v4, so `ManagedRuntime` is the boundary tool.
