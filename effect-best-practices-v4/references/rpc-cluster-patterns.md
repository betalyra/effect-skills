# RPC, HttpApi, Cluster & Workflow Patterns (Effect v4)

## Table of Contents

- [Import Paths](#import-paths)
- [RpcGroup for API Organization](#rpcgroup-for-api-organization)
- [Error Unions in RPC](#error-unions-in-rpc)
- [Implementing Handlers](#implementing-handlers)
- [RPC Middleware for Authentication](#rpc-middleware-for-authentication)
- [HttpApi Endpoints](#httpapi-endpoints)
- [Cluster Entities](#cluster-entities)
- [Workflow Definition](#workflow-definition)
- [Activity Patterns](#activity-patterns)
- [ClusterCron for Scheduled Jobs](#clustercron-for-scheduled-jobs)
- [Triggering Workflows](#triggering-workflows)

## Import Paths

Everything in this file used to be a separate package. In v4 it all lives in
`effect/unstable/*`:

| v3                     | v4                            |
| ---------------------- | ----------------------------- |
| `@effect/rpc`          | `effect/unstable/rpc`         |
| `@effect/cluster`      | `effect/unstable/cluster`     |
| `@effect/workflow`     | `effect/unstable/workflow`    |
| `@effect/platform` (HttpApi) | `effect/unstable/httpapi` |
| `@effect/platform` (HTTP) | `effect/unstable/http`     |

```typescript
import { Rpc, RpcGroup, RpcMiddleware } from "effect/unstable/rpc"
import { ClusterSchema, Entity } from "effect/unstable/cluster"
import { Activity, Workflow } from "effect/unstable/workflow"
import { HttpApi, HttpApiEndpoint, HttpApiGroup } from "effect/unstable/httpapi"
```

Platform bindings stay in their own packages: `@effect/platform-node`,
`@effect/platform-bun`, `@effect/platform-browser`.

## RpcGroup for API Organization

`RpcGroup.make` takes **individual `Rpc` definitions as arguments** — one
`Rpc.make(tag, options)` per operation:

```typescript
import { Schema } from "effect"
import { Rpc, RpcGroup } from "effect/unstable/rpc"

export const UserRpcs = RpcGroup.make(
  Rpc.make("UserFindById", {
    payload: { id: UserId },
    success: User,
    error: UserNotFoundError,
  }),

  Rpc.make("UserList", {
    payload: {
      organizationId: OrganizationId,
      limit: Schema.Int.pipe(Schema.withDecodingDefaultType(Effect.succeed(50))),
      offset: Schema.Int.pipe(Schema.withDecodingDefaultType(Effect.succeed(0))),
    },
    success: Schema.Array(User),
  }),

  Rpc.make("UserCreate", {
    payload: CreateUserInput,
    success: User,
    error: Schema.Union([UserCreateError, ValidationError]),
  }),

  Rpc.make("UserUpdate", {
    payload: { id: UserId, data: UpdateUserInput },
    success: User,
    error: Schema.Union([UserNotFoundError, ValidationError]),
  }),

  Rpc.make("UserDelete", {
    payload: { id: UserId },
    error: UserNotFoundError,
  }),
)
```

Notes on the v4 shape:

- `payload` accepts either **struct fields** (shorthand) or a full schema.
- `success` defaults to `Schema.Void`, `error` to `Schema.Never` — omit them
  rather than writing them out.
- `defect` defaults to `Schema.Defect()`, which serializes unexpected throws
  across the wire safely.
- There is no `Rpc.query` / `Rpc.mutation` split. If you want that distinction
  (for caching or retry policy), express it with annotations or naming
  convention.

### Streaming RPCs

```typescript
Rpc.make("UserEvents", {
  payload: { organizationId: OrganizationId },
  success: UserEvent,
  error: SubscriptionError,
  stream: true, // wraps success/error in a stream schema
})
```

### Deduplication with primaryKey

For persisted/cluster messages, `primaryKey` derives an idempotency key from
the payload:

```typescript
Rpc.make("SendEmail", {
  payload: { messageId: MessageId, to: Schema.String },
  primaryKey: ({ messageId }) => messageId,
})
```

## Error Unions in RPC

**Always use explicit error unions.** Note the array syntax for `Schema.Union`
in v4:

```typescript
// ✅ CORRECT — explicit union of possible errors
Rpc.make("OrderCreate", {
  payload: CreateOrderInput,
  success: Order,
  error: Schema.Union([
    ValidationError,
    InsufficientInventoryError,
    PaymentFailedError,
    UserNotFoundError,
  ]),
})

// ❌ WRONG — a generic error loses all the information the client needs
Rpc.make("OrderCreate", {
  payload: CreateOrderInput,
  success: Order,
  error: GenericError,
})
```

The client's generated types mirror this union exactly, so
`AsyncResult.builder(...).onErrorTag("InsufficientInventoryError", …)` on the
frontend just works.

## Implementing Handlers

`toLayer` implements the whole group; `toLayerHandler` implements one method
(useful for splitting a large group across modules or for tests):

```typescript
export const UserRpcLayer = UserRpcs.toLayer(
  Effect.gen(function* () {
    const users = yield* UserService

    return {
      UserFindById: ({ id }) => users.findById(id),
      UserList: ({ organizationId, limit, offset }) =>
        users.list(organizationId, { limit, offset }),
      UserCreate: (payload) => users.create(payload),
      UserUpdate: ({ id, data }) => users.update(id, data),
      UserDelete: ({ id }) => users.delete(id),
    }
  }),
)
```

## RPC Middleware for Authentication

v4 uses `RpcMiddleware.Service` (a `Context.Service`-shaped constructor) rather
than v3's `RpcMiddleware.Tag`:

```typescript
import { Context, Effect, Layer } from "effect"
import { RpcMiddleware } from "effect/unstable/rpc"

// The service the middleware provides to handlers
export class CurrentUser extends Context.Service<CurrentUser, {
  readonly id: UserId
  readonly role: UserRole
  readonly organizationId: OrganizationId
}>()("myapp/auth/CurrentUser") {}

// Declare the middleware: what it provides, and how it can fail
export class AuthMiddleware extends RpcMiddleware.Service<AuthMiddleware, {
  provides: CurrentUser
}>()("myapp/auth/AuthMiddleware", {
  error: UnauthorizedError,
}) {}

// Implement it
export const AuthMiddlewareLayer = Layer.effect(
  AuthMiddleware,
  Effect.gen(function* () {
    const auth = yield* AuthService

    return AuthMiddleware.of((options) =>
      Effect.gen(function* () {
        const token = options.headers["authorization"]?.replace("Bearer ", "")

        if (!token) {
          return yield* new UnauthorizedError({ message: "Missing token" })
        }

        return yield* auth.validateToken(token).pipe(
          // v4: multiple tags go in an ARRAY
          Effect.catchTag(
            ["TokenExpiredError", "TokenInvalidError"],
            () => new UnauthorizedError({ message: "Invalid or expired token" }),
          ),
        )
      }),
    )
  }),
)

// Apply it to a group — every handler now has CurrentUser available
export const ProtectedUserRpcs = UserRpcs.middleware(AuthMiddleware)
```

Handlers then read the provided service like any other:

```typescript
UserFindById: ({ id }) =>
  Effect.gen(function* () {
    const currentUser = yield* CurrentUser
    // authorization logic using currentUser.role
  })
```

## HttpApi Endpoints

The v4 HttpApi builder takes an **options object** instead of the
`.setPayload().addSuccess().addError()` chain:

```typescript
import { Schema } from "effect"
import {
  HttpApi,
  HttpApiEndpoint,
  HttpApiError,
  HttpApiGroup,
  HttpApiSchema,
  OpenApi,
} from "effect/unstable/httpapi"

export class UsersApiGroup extends HttpApiGroup.make("users")
  .add(
    HttpApiEndpoint.get("list", "/", {
      query: { search: Schema.optional(Schema.String) },
      success: Schema.Array(User),
    }),

    HttpApiEndpoint.get("getById", "/:id", {
      params: {
        // Path params must decode from strings — bridge with decodeTo
        id: Schema.String.check(Schema.isUUID()).pipe(Schema.decodeTo(UserId)),
      },
      success: User,
      error: UserNotFoundError,
    }),

    HttpApiEndpoint.post("create", "/", {
      // For POST, `payload` is the request body (JSON by default)
      payload: CreateUserInput,
      success: User,
      error: Schema.Union([UserCreateError, ValidationError]),
    }),
  )
  .middleware(Authorization)
  .prefix("/users")
  .annotateMerge(OpenApi.annotations({ title: "Users" }))
{}

export class Api extends HttpApi.make("user-api")
  .add(UsersApiGroup)
  .annotateMerge(OpenApi.annotations({ title: "Acme User API" }))
{}
```

Key points:

- `params` for path parameters, `query` for query string, `payload` for the
  body (or, on GET, the query string).
- `success` and `error` accept an **array** for multiple content types /
  multiple error types.
- Content negotiation via `HttpApiSchema.asText`, `asJson`, `asNoContent`,
  `asMultipart`; status via `HttpApiSchema.status(n)` or the `httpApiStatus`
  annotation on error classes.
- `HttpApiError` provides ready-made errors (`RequestTimeoutNoContent`,
  `Unauthorized`, `NotFound`, …) for the cases where a bare status is the whole
  story.

Serve it with `HttpApiBuilder.layer` + `HttpRouter.serve`, and launch with
`Layer.launch`:

```typescript
const ApiRoutes = HttpApiBuilder.layer(Api, { openapiPath: "/openapi.json" }).pipe(
  Layer.provide([UsersApiHandlers]),
)

const HttpServerLayer = HttpRouter.serve(ApiRoutes).pipe(
  Layer.provide(NodeHttpServer.layer(createServer, { port: 3000 })),
)

Layer.launch(HttpServerLayer).pipe(NodeRuntime.runMain)
```

`HttpRouter.toWebHandler` gives you a serverless-compatible handler from the
same layer.

## Cluster Entities

Entities are stateful, addressable actors distributed across the cluster. They
are defined from `Rpc`s:

```typescript
import { Effect, Layer, Ref, Schema } from "effect"
import { ClusterSchema, Entity } from "effect/unstable/cluster"
import { Rpc } from "effect/unstable/rpc"

export const Increment = Rpc.make("Increment", {
  payload: { amount: Schema.Number },
  success: Schema.Number,
})

export const GetCount = Rpc.make("GetCount", {
  success: Schema.Number,
})
  // Messages are volatile by default; annotate to persist them
  .annotate(ClusterSchema.Persisted, true)

export const Counter = Entity.make("Counter", [Increment, GetCount])

export const CounterEntityLayer = Counter.toLayer(
  Effect.gen(function* () {
    // In-memory state, held while the entity is active
    const count = yield* Ref.make(0)

    return Counter.of({
      Increment: ({ payload }) =>
        Ref.updateAndGet(count, (c) => c + payload.amount),
      GetCount: () =>
        Ref.get(count).pipe(
          // Handlers run sequentially per entity by default.
          // Rpc.fork opts a read out of that ordering.
          Rpc.fork,
        ),
    })
  }),
  // Passivation: stop the entity after idling, recreate on demand
  { maxIdleTime: "5 minutes" },
)
```

Calling an entity:

```typescript
const useCounter = Effect.gen(function* () {
  const clientFor = yield* Counter.client
  const counter = clientFor("counter-123") // the entity ID

  const afterIncrement = yield* counter.Increment({ amount: 1 })
  const current = yield* counter.GetCount()
})
```

For local development and tests, `SingleRunner.layer` or `TestRunner` give you
the entity runtime model without a real cluster.

## Workflow Definition

`Workflow.make` takes the **tag first**, then options — and the idempotency key
is mandatory:

```typescript
import { Schema } from "effect"
import { Workflow } from "effect/unstable/workflow"

export const OrderFulfillmentWorkflow = Workflow.make(
  "OrderFulfillmentWorkflow",
  {
    payload: {
      orderId: OrderId,
      userId: UserId,
      items: Schema.Array(OrderItem),
      shippingAddress: ShippingAddress,
    },
    // Deterministic execution ID — prevents duplicate processing
    idempotencyKey: ({ orderId }) => orderId,
    success: FulfillmentResult,
    error: Schema.Union([FulfillmentFailedError, PaymentFailedError]),
  },
)
```

The execution ID is derived from the workflow tag plus the idempotency key, so
re-submitting the same order resumes the existing run rather than starting a
second one. This is the single most important thing to get right in a workflow
definition.

### Workflow Implementation

```typescript
import { Effect } from "effect"
import { Activity } from "effect/unstable/workflow"

export const OrderFulfillmentWorkflowLayer = OrderFulfillmentWorkflow.toLayer(
  Effect.fn("OrderFulfillmentWorkflow")(function* (payload, executionId) {
    // Step 1: Reserve inventory
    const reservation = yield* Activity.make({
      name: "ReserveInventory",
      success: InventoryReservation,
      error: Schema.Union([InsufficientInventoryError, DatabaseError]),
      execute: Effect.gen(function* () {
        const inventory = yield* InventoryService
        return yield* inventory.reserve(payload.items)
      }),
    })

    // Step 2: Process payment
    const payment = yield* Activity.make({
      name: "ProcessPayment",
      success: PaymentResult,
      error: Schema.Union([PaymentFailedError, PaymentTimeoutError]),
      execute: Effect.gen(function* () {
        const payments = yield* PaymentService
        return yield* payments.charge(payload.userId, payload.items)
      }),
    })

    // Step 3: Create shipment
    const shipment = yield* Activity.make({
      name: "CreateShipment",
      success: Shipment,
      error: Schema.Union([ShippingError, AddressInvalidError]),
      execute: Effect.gen(function* () {
        const shipping = yield* ShippingService
        return yield* shipping.createShipment({
          items: payload.items,
          address: payload.shippingAddress,
          reservationId: reservation.id,
        })
      }),
    })

    // Step 4: Send confirmation
    yield* Activity.make({
      name: "SendConfirmation",
      error: NotificationError,
      execute: Effect.gen(function* () {
        const notifications = yield* NotificationService
        yield* notifications.sendOrderConfirmation({
          userId: payload.userId,
          orderId: payload.orderId,
          trackingNumber: shipment.trackingNumber,
        })
      }),
    })

    return { shipment, payment }
  }),
)
```

## Activity Patterns

**Always specify `success` and `error` schemas** when the activity produces a
value. They are how the result is persisted across restarts — an activity
without them can't be replayed, only re-executed.

```typescript
// ✅ CORRECT — schemas specified
yield* Activity.make({
  name: "SendEmail",
  success: EmailSentResult,
  error: Schema.Union([EmailDeliveryError, EmailTemplateError]),
  execute: Effect.gen(function* () {
    const mailer = yield* Mailer
    return yield* mailer.send(message)
  }),
})

// ⚠️ Acceptable only when the activity genuinely returns nothing —
// `success` defaults to Schema.Void
yield* Activity.make({
  name: "InvalidateCache",
  execute: cache.invalidate(key),
})
```

Activities are the **replay boundary**: on resume, completed activities return
their persisted result instead of re-running. Put every non-deterministic or
side-effecting step inside one, and keep pure orchestration logic between them.

### Activity Error Handling with Retryable

```typescript
export class ExternalApiError extends Schema.TaggedErrorClass<ExternalApiError>()(
  "ExternalApiError",
  {
    message: Schema.String,
    statusCode: Schema.Number,
    retryable: Schema.Boolean,
  },
) {
  static fromResponse(response: Response): ExternalApiError {
    return new ExternalApiError({
      message: `API error: ${response.statusText}`,
      statusCode: response.status,
      retryable: response.status >= 500, // 5xx are retryable
    })
  }
}

yield* Activity.make({
  name: "CallExternalApi",
  success: ApiResponse,
  error: ExternalApiError,
  execute: Effect.gen(function* () {
    const response = yield* callApi(url)
    if (!response.ok) {
      return yield* ExternalApiError.fromResponse(response)
    }
    return yield* parseResponse(response)
  }),
})
```

`Activity.make` also accepts `interruptRetryPolicy` for controlling how an
interrupted activity is retried on resume.

## ClusterCron for Scheduled Jobs

`ClusterCron.make` returns a `Layer` directly — there is no separate
`.toLayer()` step, and the schedule is a `Cron`, not a raw string:

```typescript
import { Cron, Effect } from "effect"
import { ClusterCron } from "effect/unstable/cluster"

export const DailyReportCronLayer = ClusterCron.make({
  name: "DailyReportCron",
  // Every day at 6 AM UTC
  cron: Cron.parseUnsafe("0 6 * * *"),
  execute: Effect.gen(function* () {
    yield* Effect.log("Starting daily report generation")
    const reports = yield* ReportService
    yield* reports.generateDailyReport()
    yield* Effect.log("Daily report generation complete")
  }),
  // Skip runs scheduled further in the past than this (default "1 day")
  skipIfOlderThan: "6 hours",
})
```

The cron runs as a cluster singleton, so exactly one runner executes it even
with many instances deployed.

## Triggering Workflows

In v4 you execute a workflow **directly on the workflow object** — there's no
`WorkflowClient` indirection. The requirement is `WorkflowEngine`, provided at
the app root.

```typescript
const createOrder = Effect.fn("OrderService.create")(function* (
  input: CreateOrderInput,
) {
  const orders = yield* OrderRepo
  const order = yield* orders.create(input)

  // Fire-and-forget: `discard: true` returns the execution ID immediately
  yield* OrderFulfillmentWorkflow.execute(
    {
      orderId: order.id,
      userId: input.userId,
      items: input.items,
      shippingAddress: input.shippingAddress,
    },
    { discard: true },
  )

  return order
})
```

Without `discard`, `execute` waits for the workflow to finish and returns its
typed success or fails with its typed error.

Other operations on the workflow object:

| Method                       | Purpose                                       |
| ---------------------------- | --------------------------------------------- |
| `execute(payload, options?)` | Start (or join) a run                         |
| `poll(executionId)`          | `Option<Result<Success, Error>>` — no blocking |
| `interrupt(executionId)`     | Cancel a run                                  |
| `resume(executionId)`        | Manually resume a suspended run               |
| `executionId(payload)`       | Compute the deterministic ID without running  |

`executionId(payload)` is the piece to reach for when you want to store the ID
alongside your own record before triggering, so a crash between the two leaves
you able to reconcile.

### Exposing Workflows over HTTP

`WorkflowProxy` and `WorkflowProxyServer` generate an RPC group or HttpApi
group from a set of workflows, rather than you hand-writing a
`/workflows/:name/execute` endpoint that loses type safety:

```typescript
import { WorkflowProxy, WorkflowProxyServer } from "effect/unstable/workflow"

const WorkflowRpcs = WorkflowProxy.toRpcGroup([
  OrderFulfillmentWorkflow,
  NotificationWorkflow,
])

const WorkflowRpcLayer = WorkflowProxyServer.layerRpcHandlers([
  OrderFulfillmentWorkflow,
  NotificationWorkflow,
])
```

Prefer this over a dynamic dispatch endpoint: it keeps payload validation and
error types intact all the way to the client.
