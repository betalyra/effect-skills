# Vercel AI SDK Patterns with Effect Schema

## Using Effect Schema as AI Tool Input

The Vercel AI SDK accepts tool input schemas via the [Standard Schema](https://standardschema.dev/) interface. Effect's `Schema.standardSchemaV1` helper bridges Effect schemas to this interface.

### Basic Tool Definition

```typescript
import { Schema } from "effect"
import { tool } from "ai"

const SearchInput = Schema.Struct({
  query: Schema.String,
  limit: Schema.optionalWith(Schema.Number, { default: () => 10 }),
})

const search = tool({
  description: "Search for items matching a query",
  inputSchema: Schema.standardSchemaV1(SearchInput),
  execute: ({ query, limit }) => {
    // ...
  },
})
```

### Tools with No Arguments

Many AI providers reject empty or `"type: "None"` schemas. To define a tool that takes no meaningful input, use a record type with an impossible value:

```typescript
import { Schema } from "effect"

const NoArgs = Schema.Record({ key: Schema.String, value: Schema.Never })

const getCurrentTime = tool({
  description: "Get the current server time",
  inputSchema: Schema.standardSchemaV1(NoArgs),
  execute: () => {
    // ...
  },
})
```

This produces a valid `{ "type": "object" }` JSON Schema that all providers accept, while ensuring no actual arguments can be passed at the type level.

### Running Effects in Tool Execute Functions

Tool `execute` functions must return a `Promise`. When the tool's logic requires Effect services, capture the runtime with the needed dependencies and use `Runtime.runPromise` to execute each tool:

```typescript
import { Effect, Runtime, Schema } from "effect"
import { tool } from "ai"

const NoArgs = Schema.Record({ key: Schema.String, value: Schema.Never })

export const createAgentTools = () =>
  Effect.gen(function* () {
    const runtime = yield* Effect.runtime<UserService | NotificationService>()
    const runPromise = Runtime.runPromise(runtime)

    return {
      listUsers: tool({
        description: "List all active users",
        inputSchema: Schema.standardSchemaV1(NoArgs),
        execute: () =>
          Effect.gen(function* () {
            const users = yield* UserService
            return yield* users.listActive()
          }).pipe(runPromise),
      }),

      createUser: tool({
        description: "Create a new user and send a welcome notification",
        inputSchema: Schema.standardSchemaV1(CreateUserInput),
        execute: ({ name, email }) =>
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
            runPromise,
          ),
      }),
    }
  })
```

The key pattern: define a **factory function** that returns an `Effect` yielding the tool definitions. Inside the generator, yield the required services, capture the runtime, and derive `runPromise` from it. Each tool's `execute` can then use `runPromise` to run effects with full access to those services.

For simple tools with **no dependencies**, `Effect.runPromise` directly is acceptable:

```typescript
const echo = tool({
  description: "Echo the input back",
  inputSchema: Schema.standardSchemaV1(EchoInput),
  execute: ({ message }) =>
    Effect.succeed(`Echo: ${message}`).pipe(Effect.runPromise),
})
```

### Anti-Patterns

```typescript
// FORBIDDEN - empty Schema.Struct for no-arg tools (providers reject this)
const bad = tool({
  inputSchema: Schema.standardSchemaV1(Schema.Struct({})),
  // ❌ Fails with: Invalid schema for function: schema must be a JSON Schema of 'type: "object"'
})

// FORBIDDEN - using zod or other schema libs alongside Effect Schema
import { z } from "zod"
const bad = tool({
  parameters: z.object({ query: z.string() }),
  // ❌ Don't mix schema libraries — use Schema.standardSchemaV1 consistently
})

// FORBIDDEN - Effect.runPromise when the tool needs service dependencies
const bad = tool({
  execute: ({ id }) =>
    UserService.pipe(
      Effect.flatMap((svc) => svc.findById(id)),
      Effect.runPromise, // ❌ Dependencies are not provided — this will fail at runtime
    ),
})
// ✅ Instead, capture the runtime with dependencies (see "Running Effects" section above)
```
