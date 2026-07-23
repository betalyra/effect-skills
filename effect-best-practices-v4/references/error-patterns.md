# Error Patterns (Effect v4)

## Table of Contents

- [Why Explicit Error Types?](#why-explicit-error-types)
- [Error Naming Conventions](#error-naming-conventions)
- [Schema.TaggedErrorClass for All Errors](#schemataggederrorclass-for-all-errors)
- [Error Handling with catchTag/catchTags](#error-handling-with-catchtagcatchtags)
- [Reason Errors (v4)](#reason-errors-v4)
- [Error Remapping Pattern](#error-remapping-pattern)
- [Retryable Errors Pattern](#retryable-errors-pattern)
- [Error Unions for Activities](#error-unions-for-activities)
- [HTTP Status Codes (Without Generic Errors)](#http-status-codes-without-generic-errors)
- [Defects and Causes](#defects-and-causes)
- [Error Logging](#error-logging)

## Why Explicit Error Types?

Generic errors like `BadRequestError` or `NotFoundError` seem convenient but
create problems:

| Generic Error         | Problems                                                |
| --------------------- | ------------------------------------------------------- |
| `NotFoundError`       | Which resource? How should frontend recover?            |
| `BadRequestError`     | What's invalid? Can user fix it?                        |
| `UnauthorizedError`   | Session expired? Wrong credentials? Missing permission? |
| `InternalServerError` | Retryable? User action needed?                          |

**Explicit errors enable:**

1. **Specific UI messages** — "Your session expired" vs generic "Unauthorized"
2. **Targeted recovery** — refresh token vs show login page
3. **Better observability** — group errors by specific type in dashboards
4. **Type-safe handling** — `catchTag("SessionExpiredError")` vs generic catch

### Anti-Pattern: Generic Error Mapping

```typescript
// ❌ WRONG - Collapsing to generic HTTP errors
export class NotFoundError extends Schema.TaggedErrorClass<NotFoundError>()(
  "NotFoundError",
  { message: Schema.String },
  { httpApiStatus: 404 },
) {}

// At API boundaries — collapsing many distinct failures into one:
Effect.catchTag(
  ["UserNotFoundError", "ChannelNotFoundError", "MessageNotFoundError"],
  () => new NotFoundError({ message: "Not found" }),
)

// Frontend receives: { _tag: "NotFoundError", message: "Not found" }
// - Can't show a specific message ("User doesn't exist" vs "Channel deleted")
// - Can't take a specific action (user search vs channel list)
// - Debugging is harder (which resource was missing?)
```

```typescript
// ✅ CORRECT - Keep explicit errors all the way to the frontend
export class UserNotFoundError extends Schema.TaggedErrorClass<UserNotFoundError>()(
  "UserNotFoundError",
  { userId: UserId, message: Schema.String },
  { httpApiStatus: 404 },
) {}

export class ChannelNotFoundError extends Schema.TaggedErrorClass<ChannelNotFoundError>()(
  "ChannelNotFoundError",
  { channelId: ChannelId, message: Schema.String },
  { httpApiStatus: 404 },
) {}

// Frontend can handle each case:
AsyncResult.builder(result)
  .onErrorTag("UserNotFoundError", (err) => <UserNotFoundMessage userId={err.userId} />)
  .onErrorTag("ChannelNotFoundError", () => <ChannelDeletedMessage />)
  .onErrorTag("SessionExpiredError", () => <RedirectToLogin />)
  .render()
```

## Error Naming Conventions

| Pattern                 | Example                                         | Use For                   |
| ----------------------- | ----------------------------------------------- | ------------------------- |
| `{Entity}NotFoundError` | `UserNotFoundError`, `ChannelNotFoundError`     | Resource lookups          |
| `{Entity}{Action}Error` | `UserCreateError`, `MessageUpdateError`         | Mutations that fail       |
| `{Feature}Error`        | `SessionExpiredError`, `RateLimitExceededError` | Feature-specific failures |
| `{Integration}Error`    | `WorkOSUserFetchError`, `StripePaymentError`    | External service errors   |
| `Invalid{Field}Error`   | `InvalidEmailError`, `InvalidPasswordError`     | Validation failures       |

### Rich Error Context

Include context fields that help with debugging and UI handling:

```typescript
// Entity errors → include entity ID
export class UserNotFoundError extends Schema.TaggedErrorClass<UserNotFoundError>()(
  "UserNotFoundError",
  {
    userId: UserId, // Which user?
    message: Schema.String,
  },
  { httpApiStatus: 404 },
) {}

// Action errors → include the input that failed
export class UserCreateError extends Schema.TaggedErrorClass<UserCreateError>()(
  "UserCreateError",
  {
    email: Schema.String, // What email failed?
    reason: Schema.String, // Why? "duplicate", "invalid domain"
    message: Schema.String,
  },
  { httpApiStatus: 400 },
) {}

// Integration errors → include service code and retryable flag
export class StripePaymentError extends Schema.TaggedErrorClass<StripePaymentError>()(
  "StripePaymentError",
  {
    stripeErrorCode: Schema.String,
    retryable: Schema.Boolean,
    message: Schema.String,
  },
  { httpApiStatus: 402 },
) {}

// Auth errors → include expiry info
export class SessionExpiredError extends Schema.TaggedErrorClass<SessionExpiredError>()(
  "SessionExpiredError",
  {
    sessionId: SessionId,
    expiredAt: Schema.DateTimeUtc,
    message: Schema.String,
  },
  { httpApiStatus: 401 },
) {}
```

## Schema.TaggedErrorClass for All Errors

**Always use `Schema.TaggedErrorClass`** (v3's `Schema.TaggedError`, renamed).
This provides:

1. **Serialization** — errors can be sent over RPC / HttpApi
2. **Type safety** — the `_tag` discriminator enables `catchTag`
3. **Consistent structure** — all errors have a predictable shape
4. **HTTP status mapping** — via the `httpApiStatus` annotation
5. **Yieldable** — usable directly without an `Effect.fail` wrapper

### The signature

```typescript
Schema.TaggedErrorClass<Self>()(
  tag,          // string literal, becomes _tag
  fields,       // Schema.Struct fields
  annotations?, // optional — e.g. { httpApiStatus: 404, description: "…" }
)
```

Annotations are a **plain object** in v4. `HttpApiSchema.annotations({ status })`
from v3 is replaced by the `httpApiStatus` key. (For non-class schemas the pipe
form `Schema.pipe(HttpApiSchema.status(404))` is also available.)

### TaggedErrors Are Yieldable

Instances implement `Yieldable`, so they can be yielded or returned directly —
no `Effect.fail(...)` wrapper needed:

```typescript
// ✅ CORRECT - TaggedErrors are yieldable
return yield* new UserNotFoundError({ userId: id, message: "Not found" })

// ❌ UNNECESSARY - Effect.fail wrapper is redundant
return yield* Effect.fail(
  new UserNotFoundError({ userId: id, message: "Not found" }),
)

// Works in catchTag handlers too:
Effect.catchTag(
  "DatabaseError",
  (err) => new UserNotFoundError({ userId: id, message: err.message }),
)

// And in Option.match:
Option.match(maybeUser, {
  onNone: () => new UserNotFoundError({ userId, message: "Not found" }),
  onSome: Effect.succeed,
})
```

**Always `return yield*` when raising an error** so TypeScript narrows the
remaining control flow to unreachable:

```typescript
const findById = Effect.fn("UserService.findById")(function* (id: UserId) {
  const maybeUser = yield* repo.findById(id)
  if (Option.isNone(maybeUser)) {
    // `return` matters — without it TS keeps analysing the rest of the body
    return yield* new UserNotFoundError({ userId: id, message: "Not found" })
  }
  return maybeUser.value
})
```

> **Note:** Only schema error classes (and `Data.TaggedError`) are yieldable.
> Plain `Error` objects still require `Effect.fail()`.

### Basic Error Definitions

```typescript
import { Schema } from "effect"

export class UserNotFoundError extends Schema.TaggedErrorClass<UserNotFoundError>()(
  "UserNotFoundError",
  {
    userId: UserId,
    message: Schema.String,
  },
  { httpApiStatus: 404 },
) {}

export class UserCreateError extends Schema.TaggedErrorClass<UserCreateError>()(
  "UserCreateError",
  {
    message: Schema.String,
    cause: Schema.optional(Schema.String),
  },
  { httpApiStatus: 400 },
) {}

export class ForbiddenError extends Schema.TaggedErrorClass<ForbiddenError>()(
  "ForbiddenError",
  {
    message: Schema.String,
    requiredPermission: Schema.optional(Schema.String),
  },
  { httpApiStatus: 403 },
) {}
```

### Required Fields

Every error should have:

- `message: Schema.String` — human-readable description
- Relevant context fields (IDs, etc.)
- Optionally `cause` — `Schema.optional(Schema.String)` for a serializable
  chain, or `Schema.Defect()` when you want to carry an arbitrary thrown value

### `Schema.Defect()` for wrapping unknown causes

When wrapping a `try`/`catch` boundary or a third-party throw, `Schema.Defect()`
encodes an arbitrary value safely:

```typescript
export class DatabaseError extends Schema.TaggedErrorClass<DatabaseError>()(
  "DatabaseError",
  { cause: Schema.Defect() },
) {}

const query = Effect.fn("Database.query")(function* (sql: string) {
  return yield* Effect.try({
    try: () => driver.query(sql),
    catch: (cause) => new DatabaseError({ cause }),
  })
})
```

### `Data.TaggedError` for internal-only errors

If an error never crosses a serialization boundary (never leaves the process,
never hits RPC or HTTP), `Data.TaggedError` is lighter and needs no schema:

```typescript
import { Data } from "effect"

class ParseFailure extends Data.TaggedError("ParseFailure")<{
  readonly input: string
}> {}
```

Prefer `Schema.TaggedErrorClass` by default; reach for `Data.TaggedError` only
when you're sure the error stays internal.

## Error Handling with catchTag/catchTags

**Never use `Effect.catch` (v4's `catchAll`) or `mapError`** when you can use
`catchTag`/`catchTags`. These preserve type information and enable precise
error handling.

### catchTag for a Single Error Type

```typescript
const findUser = Effect.fn("UserService.findUser")(function* (id: UserId) {
  return yield* repo.findById(id).pipe(
    Effect.catchTag(
      "DatabaseError",
      (err) =>
        new UserNotFoundError({
          userId: id,
          message: `Database lookup failed: ${err.message}`,
        }),
    ),
  )
})
```

### catchTag with Multiple Tags (Same Handler)

In v4, multiple tags go in an **array** — the v3 variadic form no longer
type-checks:

```typescript
// ✅ CORRECT (v4) - array of tags, single handler
yield* effect.pipe(
  Effect.catchTag(
    ["TokenExpiredError", "TokenInvalidError", "MissingTokenError"],
    () => new UnauthorizedError({ message: "Authentication failed" }),
  ),
)

// ❌ WRONG (v3 syntax) - variadic tags
yield* effect.pipe(
  Effect.catchTag(
    "TokenExpiredError",
    "TokenInvalidError",
    "MissingTokenError",
    () => new UnauthorizedError({ message: "Authentication failed" }),
  ),
)

// ❌ WRONG - catchTags with duplicate handlers (unnecessary boilerplate)
yield* effect.pipe(
  Effect.catchTags({
    TokenExpiredError: () => new UnauthorizedError({ message: "Authentication failed" }),
    TokenInvalidError: () => new UnauthorizedError({ message: "Authentication failed" }),
    MissingTokenError: () => new UnauthorizedError({ message: "Authentication failed" }),
  }),
)
```

`catchTag` also accepts an optional third argument: a catch-all handler for the
*remaining* error types, which lets you exhaustively recover in one call.

### catchTags for Multiple Error Types (Different Handlers)

Use `catchTags` only when each tag needs a **distinct** handler:

```typescript
const processOrder = Effect.fn("OrderService.processOrder")(function* (
  input: OrderInput,
) {
  return yield* validateAndProcess(input).pipe(
    Effect.catchTags({
      ValidationError: (err) =>
        new OrderValidationError({ message: err.message, field: err.field }),
      PaymentError: (err) =>
        new OrderPaymentError({
          message: `Payment failed: ${err.message}`,
          code: err.code,
        }),
      InventoryError: (err) =>
        new OrderInventoryError({
          productId: err.productId,
          message: "Insufficient inventory",
        }),
    }),
  )
})
```

### Why Not `Effect.catch`?

```typescript
// ❌ WRONG - Loses type information
yield* effect.pipe(
  Effect.catch(() => new InternalServerError({ message: "Something failed" })),
)

// Problems:
// 1. Can't distinguish error types downstream
// 2. Hides useful error context
// 3. Makes debugging harder
// 4. Frontend can't show specific messages
```

### The v4 `catch*` family

| v3                       | v4                        | Notes                                |
| ------------------------ | ------------------------- | ------------------------------------ |
| `catchAll`               | `catch`                   | Handle all typed failures            |
| `catchAllCause`          | `catchCause`              | Includes defects and interrupts      |
| `catchAllDefect`         | `catchDefect`             |                                      |
| `catchSome`              | `catchFilter`             | Now takes a `Filter`, not an `Option`|
| `catchSomeCause`         | `catchCauseFilter`        |                                      |
| `catchSomeDefect`        | —                         | Removed                              |
| `catchTag` / `catchTags` | unchanged                 | `catchTag` takes an array now        |
| —                        | `catchReason(s)`          | New — see below                      |
| —                        | `catchEager`              | New — eager synchronous recovery     |

`catchFilter` uses the `Filter` module:

```typescript
import { Filter } from "effect"

effect.pipe(
  Effect.catchFilter(
    Filter.fromPredicate((e: HttpError) => e.status >= 500),
    (e) => Effect.succeed(fallback),
  ),
)
```

## Reason Errors (v4)

v4 adds first-class support for a **tagged `reason` field** inside a tagged
error. This models "one error type, several distinct causes" without inflating
the error channel — very common for integration errors:

```typescript
export class RateLimitError extends Schema.TaggedErrorClass<RateLimitError>()(
  "RateLimitError",
  { retryAfter: Schema.Number },
) {}

export class QuotaExceededError extends Schema.TaggedErrorClass<QuotaExceededError>()(
  "QuotaExceededError",
  { limit: Schema.Number },
) {}

export class SafetyBlockedError extends Schema.TaggedErrorClass<SafetyBlockedError>()(
  "SafetyBlockedError",
  { category: Schema.String },
) {}

export class AiError extends Schema.TaggedErrorClass<AiError>()("AiError", {
  reason: Schema.Union([RateLimitError, QuotaExceededError, SafetyBlockedError]),
}) {}
```

Three ways to recover:

```typescript
// 1. Handle a single reason, leaving AiError in the error channel for the rest
callModel.pipe(
  Effect.catchReason(
    "AiError",        // parent error tag
    "RateLimitError", // reason tag
    (reason) => Effect.succeed(`Retry after ${reason.retryAfter}s`),
    // optional catch-all for the other reasons
    (reason) => Effect.succeed(`Model call failed: ${reason._tag}`),
  ),
)

// 2. Handle several reasons at once
callModel.pipe(
  Effect.catchReasons("AiError", {
    RateLimitError: (r) => Effect.succeed(`Retry after ${r.retryAfter}s`),
    QuotaExceededError: (r) => Effect.succeed(`Quota exceeded at ${r.limit}`),
  }),
)

// 3. Unwrap the reasons into the error channel, then use ordinary catchTags
callModel.pipe(
  Effect.unwrapReason("AiError"),
  Effect.catchTags({
    RateLimitError: (r) => Effect.succeed(`Back off ${r.retryAfter}s`),
    QuotaExceededError: (r) => Effect.succeed(`Increase quota past ${r.limit}`),
    SafetyBlockedError: (r) => Effect.succeed(`Blocked: ${r.category}`),
  }),
)
```

Use `reason` when the failures share a call site and a recovery *strategy* but
differ in detail. Use separate top-level error types when callers genuinely need
to handle them at different places in the program.

## Error Remapping Pattern

Create reusable error remapping functions for common transformations:

```typescript
import { Effect } from "effect"

export const withRemapDbErrors =
  (context: { entityType: string; entityId: string }) =>
  <A, E, R>(
    effect: Effect.Effect<A, E | DatabaseError | ConnectionError, R>,
  ): Effect.Effect<A, E | EntityNotFoundError | ServiceUnavailableError, R> =>
    effect.pipe(
      Effect.catchTag(
        "DatabaseError",
        () =>
          new EntityNotFoundError({
            entityType: context.entityType,
            entityId: context.entityId,
            message: `${context.entityType} not found`,
          }),
      ),
      Effect.catchTag(
        "ConnectionError",
        (err) =>
          new ServiceUnavailableError({
            message: "Database connection unavailable",
            cause: err.message,
          }),
      ),
    )

// Usage
const findUser = Effect.fn("UserService.findUser")(function* (id: UserId) {
  return yield* repo
    .findById(id)
    .pipe(withRemapDbErrors({ entityType: "User", entityId: id }))
})
```

Alternatively, pass the remapping as an extra argument to `Effect.fn`, which
applies it to every call:

```typescript
const findById = Effect.fn("UserRepository.findById")(
  function* (id: UserId) {
    return yield* sql`SELECT * FROM users WHERE id = ${id}`
  },
  Effect.mapError((cause) => new UserRepositoryError({ cause })),
)
```

## Retryable Errors Pattern

For errors that may be transient, add a `retryable` field. In v4 defaults use
`withDecodingDefaultType` rather than `optionalWith({ default })`:

```typescript
import { Effect, Schema } from "effect"

export class ServiceUnavailableError extends Schema.TaggedErrorClass<ServiceUnavailableError>()(
  "ServiceUnavailableError",
  {
    message: Schema.String,
    cause: Schema.optional(Schema.String),
    retryable: Schema.Boolean.pipe(
      Schema.withDecodingDefaultType(Effect.succeed(true)),
    ),
  },
  { httpApiStatus: 503 },
) {}

export class RateLimitError extends Schema.TaggedErrorClass<RateLimitError>()(
  "RateLimitError",
  {
    message: Schema.String,
    retryAfter: Schema.optional(Schema.Number),
    retryable: Schema.Boolean.pipe(
      Schema.withDecodingDefaultType(Effect.succeed(true)),
    ),
  },
  { httpApiStatus: 429 },
) {}

// Non-retryable error
export class ValidationError extends Schema.TaggedErrorClass<ValidationError>()(
  "ValidationError",
  {
    message: Schema.String,
    field: Schema.String,
    retryable: Schema.Boolean.pipe(
      Schema.withDecodingDefaultType(Effect.succeed(false)),
    ),
  },
  { httpApiStatus: 400 },
) {}
```

### Retry Based on an Error Property

`Schedule.whileInput` is gone in v4. Use the `Retry.Options` object form, which
takes `schedule`, `times`, `while`, and `until` together:

```typescript
import { Effect, Schedule } from "effect"

const withRetry = <A, E extends { retryable?: boolean }, R>(
  effect: Effect.Effect<A, E, R>,
): Effect.Effect<A, E, R> =>
  effect.pipe(
    Effect.retry({
      schedule: Schedule.exponential("100 millis"),
      times: 3,
      while: (err) => err.retryable === true,
    }),
  )

// Usage
yield* callExternalApi(request).pipe(withRetry)
```

Remember: the source effect always runs once before the policy applies, and
defects and interruptions are never retried.

## Error Unions for Activities

When defining workflow activities, use explicit error unions. Note the array
syntax for `Schema.Union` in v4:

```typescript
import { Schema } from "effect"
import { Activity } from "effect/unstable/workflow"

export class DatabaseError extends Schema.TaggedErrorClass<DatabaseError>()(
  "DatabaseError",
  {
    message: Schema.String,
    cause: Schema.optional(Schema.String),
    retryable: Schema.Boolean.pipe(
      Schema.withDecodingDefaultType(Effect.succeed(true)),
    ),
  },
) {}

export class ChannelNotFoundError extends Schema.TaggedErrorClass<ChannelNotFoundError>()(
  "ChannelNotFoundError",
  {
    channelId: ChannelId,
    message: Schema.String,
    retryable: Schema.Boolean.pipe(
      Schema.withDecodingDefaultType(Effect.succeed(false)),
    ),
  },
) {}

export type GetChannelMembersError = DatabaseError | ChannelNotFoundError

// In the activity definition
yield* Activity.make({
  name: "GetChannelMembers",
  success: ChannelMembersResult,
  error: Schema.Union([DatabaseError, ChannelNotFoundError]),
  execute: Effect.gen(function* () {
    // ...
  }),
})
```

## HTTP Status Codes (Without Generic Errors)

**Map HTTP status codes at the error level, not by creating generic error
classes.** Each explicit error carries its own status.

```typescript
// ✅ CORRECT - Domain errors with status annotations
export class UserNotFoundError extends Schema.TaggedErrorClass<UserNotFoundError>()(
  "UserNotFoundError",
  { userId: UserId, message: Schema.String },
  { httpApiStatus: 404 }, // Status on the specific error
) {}

export class ChannelNotFoundError extends Schema.TaggedErrorClass<ChannelNotFoundError>()(
  "ChannelNotFoundError",
  { channelId: ChannelId, message: Schema.String },
  { httpApiStatus: 404 }, // Same status, different error
) {}

export class SessionExpiredError extends Schema.TaggedErrorClass<SessionExpiredError>()(
  "SessionExpiredError",
  {
    sessionId: SessionId,
    expiredAt: Schema.DateTimeUtc,
    message: Schema.String,
  },
  { httpApiStatus: 401 },
) {}

export class InvalidCredentialsError extends Schema.TaggedErrorClass<InvalidCredentialsError>()(
  "InvalidCredentialsError",
  { message: Schema.String },
  { httpApiStatus: 401 }, // Same status, different meaning
) {}
```

```typescript
// ❌ WRONG - Generic HTTP error classes
export class UnauthorizedError extends Schema.TaggedErrorClass<UnauthorizedError>()(
  "UnauthorizedError",
  { message: Schema.String },
  { httpApiStatus: 401 },
) {}

// Then mapping everything to it — loses critical information
Effect.catchTag(
  ["SessionExpiredError", "InvalidCredentialsError", "MissingTokenError"],
  () => new UnauthorizedError({ message: "Unauthorized" }),
)
// Frontend can't distinguish expired session vs wrong password vs missing token
```

`effect/unstable/httpapi/HttpApiError` ships ready-made errors (`BadRequest`,
`Unauthorized`, `NotFound`, `Conflict`, …) for the cases where a bare status
genuinely is the whole story — middleware rejections, for example. Don't use
them as a dumping ground for domain failures.

### When Generic Errors Are Acceptable

Only for **truly unrecoverable internal errors** where:

- The frontend can only show "Something went wrong"
- No user action can fix it
- You're deliberately hiding internal details for security

```typescript
export class InternalServerError extends Schema.TaggedErrorClass<InternalServerError>()(
  "InternalServerError",
  { message: Schema.String, requestId: Schema.optional(Schema.String) },
  { httpApiStatus: 500 },
) {}

// Use sparingly — only for truly unexpected errors
Effect.catch(
  () =>
    new InternalServerError({
      message: "An unexpected error occurred",
      requestId: context.requestId,
    }),
)
```

## Defects and Causes

v4 flattened `Cause` from a recursive tree into a list of failures. Inspect it
with the module functions rather than pattern matching on node types:

```typescript
import { Cause, Effect } from "effect"

program.pipe(
  Effect.catchCause((cause) =>
    Effect.gen(function* () {
      // Result<Error, Defect> — the first failure in the cause
      const failure = Cause.findError(cause)
      yield* Effect.logError("Program failed", {
        pretty: Cause.pretty(cause),
        interrupted: Cause.hasInterrupts(cause),
      })
      return fallback
    }),
  ),
)
```

`Effect.catchDefect` handles thrown/unexpected values specifically. Reserve it
for process boundaries — a defect anywhere else is a bug you want to see.

## Error Logging

Log errors with structured context:

```typescript
const processWithLogging = Effect.fn("OrderService.process")(function* (
  orderId: OrderId,
) {
  return yield* processOrder(orderId).pipe(
    Effect.tapError((err) =>
      Effect.logError("Order processing failed", {
        orderId,
        errorTag: err._tag,
        errorMessage: err.message,
      }),
    ),
  )
})
```

`Effect.tapErrorCause` was renamed to `Effect.tapCause` in v4 — use it when you
also want to observe defects and interrupts.
