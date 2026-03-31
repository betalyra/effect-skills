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

### Combining with Effect Services

When tools need to run effects, convert the result with `Effect.runPromise` at the tool boundary:

```typescript
import { Effect, Schema } from "effect"
import { tool } from "ai"

const CreateUserInput = Schema.Struct({
  name: Schema.String,
  email: Schema.String,
})

const createUser = tool({
  description: "Create a new user",
  inputSchema: Schema.standardSchemaV1(CreateUserInput),
  execute: ({ name, email }) =>
    createUserEffect(name, email).pipe(Effect.runPromise),
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
```
