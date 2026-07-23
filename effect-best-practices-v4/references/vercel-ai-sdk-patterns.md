# Vercel AI SDK Patterns with Effect Schema (Effect v4)

## What Changed From v3

| v3                                          | v4                                             |
| ------------------------------------------- | ---------------------------------------------- |
| `Schema.standardSchemaV1(s)`                | `Schema.toStandardSchemaV1(s)`                 |
| `Schema.optionalWith(s, { default })`       | `s.pipe(Schema.withDecodingDefaultType(...))`  |
| `Schema.Record({ key, value })`             | `Schema.Record(key, value)`                    |
| `Effect.runtime<R>()`                       | `Effect.context<R>()`                          |
| `Runtime.runPromise(runtime)`               | `Effect.runPromiseWith(services)`              |

The `Runtime<R>` type was removed in v4. Where v3 captured a `Runtime` and
called `Runtime.runPromise` on it, v4 captures a `Context<R>` (via
`Effect.context`) and runs with the `*With` run functions
(`Effect.runPromiseWith`, `Effect.runForkWith`, `Effect.runSyncWith`).

## Using Effect Schema as AI Tool Input

The Vercel AI SDK accepts tool input schemas via the [Standard Schema](https://standardschema.dev/)
interface. Effect's `Schema.toStandardSchemaV1` helper bridges Effect schemas to
this interface.

### Basic Tool Definition

```typescript
import { Effect, Schema } from "effect"
import { tool } from "ai"

const SearchInput = Schema.Struct({
  query: Schema.String,
  // v3's optionalWith({ default }) → withDecodingDefaultType
  limit: Schema.Number.pipe(Schema.withDecodingDefaultType(Effect.succeed(10))),
})

const search = tool({
  description: "Search for items matching a query",
  inputSchema: Schema.toStandardSchemaV1(SearchInput),
  execute: ({ query, limit }) => {
    // ...
  },
})
```

### Tools with No Arguments

Many AI providers reject empty or `"type": "None"` schemas. To define a tool
that takes no meaningful input, use a record type with an impossible value.
Note the positional `Schema.Record(key, value)` signature in v4:

```typescript
import { Schema } from "effect"

const NoArgs = Schema.Record(Schema.String, Schema.Never)

const getCurrentTime = tool({
  description: "Get the current server time",
  inputSchema: Schema.toStandardSchemaV1(NoArgs),
  execute: () => {
    // ...
  },
})
```

This produces a valid `{ "type": "object" }` JSON Schema that all providers
accept, while ensuring no actual arguments can be passed at the type level.

### Running Effects in Tool Execute Functions

Tool `execute` functions must return a `Promise`. When the tool's logic requires
Effect services, capture the current **context** with the needed dependencies
and use `Effect.runPromiseWith` to execute each tool:

```typescript
import { Effect, Schema } from "effect"
import { tool } from "ai"

const NoArgs = Schema.Record(Schema.String, Schema.Never)

export const createAgentTools = Effect.fn("createAgentTools")(function* () {
  // v3 captured Effect.runtime<Deps>(); v4 captures the Context instead
  const services = yield* Effect.context<UserService | NotificationService>()
  const runPromise = Effect.runPromiseWith(services)

  return {
    listUsers: tool({
      description: "List all active users",
      inputSchema: Schema.toStandardSchemaV1(NoArgs),
      execute: () =>
        runPromise(
          Effect.gen(function* () {
            const users = yield* UserService
            return yield* users.listActive()
          }),
        ),
    }),

    createUser: tool({
      description: "Create a new user and send a welcome notification",
      inputSchema: Schema.toStandardSchemaV1(CreateUserInput),
      execute: ({ name, email }) =>
        runPromise(
          Effect.gen(function* () {
            const users = yield* UserService
            const notifications = yield* NotificationService
            const user = yield* users.create({ name, email })
            yield* notifications.sendWelcome(user.id)
            return user
          }).pipe(
            Effect.catchTag("UserCreateError", (e) =>
              Effect.succeed({ error: e.message }),
            ),
          ),
        ),
    }),
  }
})
```

The key pattern: define a **factory function** that returns an `Effect` yielding
the tool definitions. Inside the generator, capture the context and derive
`runPromise` from it with `Effect.runPromiseWith`. Each tool's `execute` can then
run effects with full access to services and runtime configuration (log level,
loggers, spans, metrics).

`Effect.runPromiseWith(services)` returns a function that runs effects whose
requirements are satisfied by `services` — the type checker enforces that the
captured context covers every service the tools yield.

**Always use the context-capture pattern** — even for tools with no service
dependencies. Using bare `Effect.runPromise` bypasses the configured
environment, losing the log level, loggers, metrics, and other infrastructure
set up in your layers.

### Anti-Patterns

```typescript
// FORBIDDEN - empty Schema.Struct for no-arg tools (providers reject this)
const bad = tool({
  inputSchema: Schema.toStandardSchemaV1(Schema.Struct({})),
  // ❌ Fails with: Invalid schema for function: schema must be a JSON Schema of 'type: "object"'
})

// FORBIDDEN - using zod or other schema libs alongside Effect Schema
import { z } from "zod"
const bad = tool({
  parameters: z.object({ query: z.string() }),
  // ❌ Don't mix schema libraries — use Schema.toStandardSchemaV1 consistently
})

// FORBIDDEN - bare Effect.runPromise in tool execute functions
const bad = tool({
  execute: ({ message }) => Effect.runPromise(Effect.succeed(message)),
  // ❌ Bypasses the configured environment — loses log levels, loggers, metrics, spans
})
// ✅ Always use Effect.runPromiseWith with a captured context (see "Running Effects" above)
```
