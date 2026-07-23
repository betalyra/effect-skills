# Service Patterns (Effect v4)

## Table of Contents

- [Context.Service Is the Only Service API](#contextservice-is-the-only-service-api)
- [Declaring the Interface](#declaring-the-interface)
- [Building Layers](#building-layers)
- [The `make` Option](#the-make-option)
- [Accessing Services](#accessing-services)
- [Context.Reference for Defaults](#contextreference-for-defaults)
- [Effect.fn for Tracing](#effectfn-for-tracing)
- [Runtime-Injected Infrastructure](#runtime-injected-infrastructure)
- [Single Responsibility](#single-responsibility)
- [Capability-Based Services](#capability-based-services)
- [No Requirement Leakage in Service Interface](#no-requirement-leakage-in-service-interface)
- [Optional Capabilities](#optional-capabilities)
- [Service Interface Patterns](#service-interface-patterns)
- [Testing Services](#testing-services)

## Context.Service Is the Only Service API

v4 collapsed `Context.Tag`, `Context.GenericTag`, `Effect.Tag`, and
`Effect.Service` into a single API: **`Context.Service`**. There is no longer a
"tag vs service" decision to make — the difference between "infrastructure
injected at runtime" and "business logic with a constructor effect" is now
purely a question of *which layer you attach*, not which API you use.

```typescript
import { Context, Effect, Layer } from "effect"

export class UserService extends Context.Service<UserService, {
  findById(id: UserId): Effect.Effect<User, UserNotFoundError>
  findByEmail(email: string): Effect.Effect<User, UserNotFoundError>
  create(input: CreateUserInput): Effect.Effect<User, UserCreateError>
}>()("myapp/users/UserService") {
  static readonly layer = Layer.effect(
    UserService,
    Effect.gen(function* () {
      const findById = Effect.fn("UserService.findById")(function* (id: UserId) {
        // Implementation
      })

      const findByEmail = Effect.fn("UserService.findByEmail")(function* (
        email: string,
      ) {
        // Implementation
      })

      const create = Effect.fn("UserService.create")(function* (
        input: CreateUserInput,
      ) {
        // Implementation
      })

      return UserService.of({ findById, findByEmail, create })
    }),
  )
}
```

Three things to notice:

1. **The interface is the second type parameter.** This is the contract; the
   layer is the implementation. Consumers only ever see the interface.
2. **The identifier is namespaced** (`"myapp/users/UserService"`). Use the
   package name plus the path to the module. Bare `"UserService"` risks
   collisions once two libraries do the same.
3. **`UserService.of({...})`** wraps the implementation object so TypeScript
   checks it against the declared interface at the point of construction, where
   the error message is useful.

## Declaring the Interface

Prefer method shorthand over readonly arrow properties — it reads better and
supports overloads and generics, which the v3 accessor proxy could not:

```typescript
// ✅ Preferred
export class Cache extends Context.Service<Cache, {
  get<A>(key: string): Effect.Effect<Option.Option<A>>
  set<A>(key: string, value: A): Effect.Effect<void>
  readonly size: Effect.Effect<number>
}>()("myapp/Cache") {}
```

Non-function members (like `size` above) are plain `Effect` values and should be
`readonly`.

If you need the service's shape as a type elsewhere, use the `Service` member:

```typescript
export type CacheShape = Cache["Service"]
```

## Building Layers

There is no auto-generated `.Default`. You write the layer, and you wire its
dependencies with `Layer.provide`. The convention that scales best is to split
the layer in two:

```typescript
export class OrderService extends Context.Service<OrderService, {
  create(input: CreateOrderInput): Effect.Effect<Order, CreateOrderError>
}>()("myapp/orders/OrderService") {
  // Dependencies stay in the requirements channel — this is what tests target
  static readonly layerNoDeps: Layer.Layer<
    OrderService,
    never,
    UserService | ProductService | InventoryService
  > = Layer.effect(
    OrderService,
    Effect.gen(function* () {
      const users = yield* UserService
      const products = yield* ProductService
      const inventory = yield* InventoryService

      const create = Effect.fn("OrderService.create")(function* (
        input: CreateOrderInput,
      ) {
        const user = yield* users.findById(input.userId)
        const product = yield* products.findById(input.productId)
        const available = yield* inventory.checkAvailability(
          input.productId,
          input.quantity,
        )

        if (!available) {
          return yield* new InsufficientInventoryError({
            productId: input.productId,
            message: "Not enough inventory",
          })
        }

        // Create order...
      })

      return OrderService.of({ create })
    }),
  )

  // Production layer — dependencies provided, requirements channel is empty
  static readonly layer = this.layerNoDeps.pipe(
    Layer.provide([
      UserService.layer,
      ProductService.layer,
      InventoryService.layer,
    ]),
  )
}
```

### `provide` vs `provideMerge`

- `Layer.provide(deps)` — satisfies the requirements and **hides** the
  dependencies. The result only exposes `OrderService`. This is the default.
- `Layer.provideMerge(deps)` — satisfies the requirements and **also exposes**
  the dependencies. Use when callers legitimately need both, e.g. a test layer
  that exposes the underlying `Ref` so assertions can inspect it.

### Naming convention

| Name           | Meaning                                                  |
| -------------- | -------------------------------------------------------- |
| `layer`        | The primary, fully-wired production layer                |
| `layerNoDeps`  | The layer with its requirements still open               |
| `layerTest`    | Test/fake implementation                                 |
| `layerConfig`  | Variant parameterised by config                          |
| `layerInMemory`| In-memory variant                                        |

v3's `Default` and `Live` suffixes are gone. Don't reintroduce them.

## The `make` Option

`Context.Service` accepts a `make` option that stores the constructor effect on
the class. It does **not** create a layer for you — but it keeps the constructor
next to the interface, which reads nicely for small services:

```typescript
export class Logger extends Context.Service<Logger>()("myapp/Logger", {
  make: Effect.gen(function* () {
    const config = yield* AppConfig
    return {
      log: (msg: string) => Effect.log(`[${config.prefix}] ${msg}`),
    }
  }),
}) {
  static readonly layer = Layer.effect(this, this.make).pipe(
    Layer.provide(AppConfig.layer),
  )
}
```

Use `make` when the service has exactly one real implementation. Use the
explicit interface form when you expect multiple implementations (production,
test, in-memory), because then the interface is the thing worth writing down.

## Accessing Services

**Prefer `yield*`.** It makes the dependency visible at the call site and keeps
service access co-located with the rest of your effect logic:

```typescript
const program = Effect.gen(function* () {
  const users = yield* UserService
  const user = yield* users.findById(userId)
  yield* users.touch(user.id)
  return user
})
```

`use` / `useSync` exist as one-liners, but they make it easy to lose track of
which services a function depends on — the requirement ends up in the return
type without ever appearing in the body:

```typescript
// Effect<void, never, Notifications>
const notify = Notifications.use((n) => n.notify("hello"))

// Effect<number, never, AppConfig> — pure callback
const port = AppConfig.useSync((c) => c.port)
```

**`ServiceName.method(...)` accessors from v3 do not exist.** They were removed
because the proxy was built from mapped types, which erased generic type
parameters and overloads: a `get<T>(key: string): Effect<T>` collapsed to
`get(key: string): Effect<unknown>`.

## Context.Reference for Defaults

For configuration values, feature flags, and anything with a sensible default
that rarely needs overriding, use `Context.Reference` — it never has to be
provided:

```typescript
import { Context } from "effect"

export const FeatureFlag = Context.Reference<boolean>("myapp/FeatureFlag", {
  defaultValue: () => false,
})

// Read it like any service; no layer required
const program = Effect.gen(function* () {
  const enabled = yield* FeatureFlag
  // ...
})

// Override where needed
program.pipe(Effect.provideService(FeatureFlag, true))
```

Note the v4 signature is `Context.Reference<Value>(id, options)` — a plain
function call, not the v3 class-extension form.

## Effect.fn for Tracing

**Always wrap service methods with `Effect.fn`.** It attaches a span (via
`Effect.withSpan`) and improves stack traces.

### Naming Convention

Use `ServiceName.methodName` for span names:

```typescript
const findById = Effect.fn("UserService.findById")(function* (id: UserId) {
  yield* Effect.annotateCurrentSpan("userId", id)
  // Implementation
})

const processPayment = Effect.fn("PaymentService.processPayment")(function* (
  orderId: OrderId,
  amount: number,
  currency: string,
) {
  yield* Effect.annotateCurrentSpan("orderId", orderId)
  yield* Effect.annotateCurrentSpan("amount", amount)
  yield* Effect.annotateCurrentSpan("currency", currency)
  // Implementation
})
```

### Extra combinators go in the argument list

**Do not `.pipe(...)` the result of `Effect.fn`** — pass combinators as
additional arguments so they apply to each invocation:

```typescript
// ✅ CORRECT
const findById = Effect.fn("UserRepository.findById")(
  function* (id: UserId) {
    const rows = yield* sql`SELECT * FROM users WHERE id = ${id}`
    return Array.head(rows)
  },
  Effect.mapError((cause) => new UserRepositoryError({ cause })),
  Effect.annotateLogs({ method: "findById" }),
)

// ❌ WRONG - pipes the function value, not the effect it returns
const findById = Effect.fn("UserRepository.findById")(function* (id: UserId) {
  // ...
}).pipe(Effect.mapError(/* … */))
```

### Declaring the return type

`Effect.fn.Return` annotates the generator's return type without you having to
spell out the full `Effect` signature:

```typescript
const load = Effect.fn("Repo.load")(
  function* (id: UserId): Effect.fn.Return<User, UserNotFoundError> {
    // ...
  },
)
```

### Annotating Spans

Add important context, but don't overdo it:

```typescript
// ✅ CORRECT - Important business identifiers
yield* Effect.annotateCurrentSpan("userId", userId)
yield* Effect.annotateCurrentSpan("orderId", orderId)
yield* Effect.annotateCurrentSpan("amount", amount)

// ❌ WRONG - Too much detail, noise in traces
yield* Effect.annotateCurrentSpan("userEmail", user.email)
yield* Effect.annotateCurrentSpan("userName", user.name)
yield* Effect.annotateCurrentSpan("step", "validating")
yield* Effect.annotateCurrentSpan("step", "processing")
```

## Runtime-Injected Infrastructure

Services whose value is handed to you by the host (worker bindings, request
context) are still `Context.Service` — they just get `Effect.provideService`
instead of a constructed layer:

```typescript
import { Context, Effect } from "effect"

export class KVNamespace extends Context.Service<
  KVNamespace,
  CloudflareKVNamespace
>()("myapp/cf/KVNamespace") {}

export class R2Bucket extends Context.Service<
  R2Bucket,
  CloudflareR2Bucket
>()("myapp/cf/R2Bucket") {}

// In the worker entry point
const handler = {
  fetch(request: Request, env: Env) {
    return program.pipe(
      Effect.provideService(KVNamespace, env.MY_KV),
      Effect.provideService(R2Bucket, env.MY_BUCKET),
      Effect.runPromise,
    )
  },
}
```

Note the shape can be any type, not just an object of methods — here it's the
raw binding.

### Database / SQL clients

Library-provided services already come as `Context.Service`; just build their
layer at the app root:

```typescript
import { PgClient } from "@effect/sql-pg"
import { Config } from "effect"

const DatabaseLive = PgClient.layerConfig({
  url: Config.redacted("DATABASE_URL"),
})
```

## Single Responsibility

Each service should have a focused responsibility:

```typescript
// ✅ CORRECT - Focused services
export class UserService extends Context.Service<UserService, {/*…*/}>()("myapp/UserService") {}
export class AuthService extends Context.Service<AuthService, {/*…*/}>()("myapp/AuthService") {}
export class NotificationService extends Context.Service<NotificationService, {/*…*/}>()("myapp/NotificationService") {}

// ❌ WRONG - God service doing everything
export class AppService extends Context.Service<AppService, {
  createUser(/*…*/): Effect.Effect<User>
  deleteUser(/*…*/): Effect.Effect<void>
  login(/*…*/): Effect.Effect<Session>
  sendEmail(/*…*/): Effect.Effect<void>
  processPayment(/*…*/): Effect.Effect<Receipt>
  // ... 50 more methods
}>()("myapp/AppService") {}
```

## Capability-Based Services

Design services as focused capabilities that compose into complete solutions.
Because a layer can only satisfy the services it declares, splitting by
capability lets different implementations offer different subsets:

```typescript
// ❌ WRONG - Mixed concerns in one service
export class PaymentService extends Context.Service<PaymentService, {
  processPayment(/*…*/): Effect.Effect<HandoffResult, HandoffError>
  validateWebhook(/*…*/): Effect.Effect<void, WebhookValidationError>
  refund(/*…*/): Effect.Effect<RefundResult, RefundError>
  sendReceipt(/*…*/): Effect.Effect<void>      // Notification concern
  generateReport(/*…*/): Effect.Effect<Report> // Reporting concern
}>()("myapp/payment/PaymentService") {}

// ✅ CORRECT - Focused capabilities
export class PaymentGateway extends Context.Service<PaymentGateway, {
  handoff(intent: PaymentIntent): Effect.Effect<HandoffResult, HandoffError>
}>()("myapp/payment/PaymentGateway") {}

export class PaymentWebhookGateway extends Context.Service<PaymentWebhookGateway, {
  validateWebhook(payload: WebhookPayload): Effect.Effect<void, WebhookValidationError>
}>()("myapp/payment/PaymentWebhookGateway") {}

export class PaymentRefundGateway extends Context.Service<PaymentRefundGateway, {
  refund(paymentId: PaymentId, amount: Cents): Effect.Effect<RefundResult, RefundError>
}>()("myapp/payment/PaymentRefundGateway") {}
```

### Composing Capabilities

Different implementations support different capabilities:

```typescript
// Cash payments: basic handoff only
export const CashGatewayLayer = Layer.succeed(
  PaymentGateway,
  PaymentGateway.of({
    handoff: (intent) => fulfillCashPayment(intent),
  }),
)

// Stripe: full capability suite
export const StripeGatewayLayer = Layer.mergeAll(
  StripeHandoffLayer, // Implements PaymentGateway
  StripeWebhookLayer, // Implements PaymentWebhookGateway
  StripeRefundLayer,  // Implements PaymentRefundGateway
)
```

## No Requirement Leakage in Service Interface

Service operations should not carry requirements in their return type:

```typescript
export class Database extends Context.Service<Database, {
  query(sql: string): Effect.Effect<QueryResult, QueryError>
  //                                                     ▲
  //                                     Requirements = never (omitted)
}>()("myapp/db/Database") {}
```

Exceptions:

- Runtime dependencies (Cloudflare bindings, HTTP request details)
- Dependencies that differ per call (e.g. an AI provider chosen at call time)
- `Scope`, when a method genuinely hands back a scoped resource

Dependencies belong in **layer construction**, not the interface:

```typescript
export class Database extends Context.Service<Database, {
  query(sql: string): Effect.Effect<QueryResult, QueryError>
}>()("myapp/db/Database") {
  static readonly layerNoDeps = Layer.effect(
    Database,
    Effect.gen(function* () {
      const config = yield* AppConfig // Dependency
      const logger = yield* Logger    // Dependency

      const query = Effect.fn("Database.query")(function* (sql: string) {
        yield* logger.log(`Executing: ${sql}`)
        return yield* executeQuery(config.connection, sql)
      })

      return Database.of({ query })
    }),
  )

  static readonly layer = this.layerNoDeps.pipe(
    Layer.provide([AppConfig.layer, Logger.layer]),
  )
}
```

## Optional Capabilities

Use `Effect.serviceOption` for capabilities that may not be provided:

```typescript
const processPayment = Effect.fn("Payments.processPayment")(function* (
  order: Order,
) {
  const gateway = yield* PaymentGateway
  const result = yield* gateway.handoff(order.paymentIntent)

  // Optional capability — Effect<Option<PaymentRefundGateway>>, never fails
  const refundGateway = yield* Effect.serviceOption(PaymentRefundGateway)

  if (Option.isSome(refundGateway)) {
    yield* setupRefundPolicy(refundGateway.value, order)
  }

  return result
})
```

## Service Interface Patterns

### Return Types

Services return `Effect`, never `Promise`:

```typescript
// ✅ CORRECT
const findById = Effect.fn("UserService.findById")(
  function* (id: UserId): Effect.fn.Return<User, UserNotFoundError> {
    // ...
  },
)

// ❌ WRONG - Promise in service interface
const findById = async (id: UserId): Promise<User> => {
  // ...
}
```

### Use Option for Nullable Results

Expose both shapes when both are genuinely useful — one that fails, one that
returns `Option`:

```typescript
const findById = Effect.fn("UserService.findById")(function* (id: UserId) {
  const maybeUser = yield* repo.findById(id)
  if (Option.isNone(maybeUser)) {
    return yield* new UserNotFoundError({ userId: id, message: "Not found" })
  }
  return maybeUser.value
})

const findByIdOption = Effect.fn("UserService.findByIdOption")(function* (
  id: UserId,
) {
  return yield* repo.findById(id)
})
```

## Testing Services

Because the interface and the layer are separate, a fake is just another layer
for the same service:

```typescript
// Stateless fake
export const UserServiceTest = Layer.succeed(
  UserService,
  UserService.of({
    findById: () => Effect.succeed(mockUser),
    create: (input) => Effect.succeed({ ...mockUser, ...input }),
  }),
)
```

For stateful fakes, expose the state through its own service so tests can assert
on it directly — that's what `Layer.provideMerge` is for:

```typescript
export class UserStoreRef extends Context.Service<
  UserStoreRef,
  Ref.Ref<Map<UserId, User>>
>()("myapp/test/UserStoreRef") {
  static readonly layer = Layer.effect(UserStoreRef, Ref.make(new Map()))
}

export const UserServiceTestLayer = Layer.effect(
  UserService,
  Effect.gen(function* () {
    const store = yield* UserStoreRef

    const findById = Effect.fn("UserService.findById")(function* (id: UserId) {
      const users = yield* Ref.get(store)
      const user = users.get(id)
      if (user === undefined) {
        return yield* new UserNotFoundError({ userId: id, message: "Not found" })
      }
      return user
    })

    const create = Effect.fn("UserService.create")(function* (
      input: CreateUserInput,
    ) {
      const user = { id: UserId.make(crypto.randomUUID()), ...input }
      yield* Ref.update(store, (users) => new Map(users).set(user.id, user))
      return user
    })

    return UserService.of({ findById, create })
  }),
).pipe(
  // provideMerge so tests can `yield* UserStoreRef` and inspect the state
  Layer.provideMerge(UserStoreRef.layer),
)
```

See `effect-test-patterns.md` for `it.effect`, `layer(...)` blocks, and
`TestClock`.
