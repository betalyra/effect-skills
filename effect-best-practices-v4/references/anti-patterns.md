# Anti-Patterns (Forbidden) — Effect v4

## Table of Contents

- [v3 APIs That No Longer Exist](#forbidden-v3-apis-that-no-longer-exist)
- [Effect.runSync/runPromise Inside Services](#forbidden-effectrunsyncrunpromise-inside-services)
- [throw Inside Effect.gen](#forbidden-throw-inside-effectgen)
- [Yielding an Error Without `return`](#forbidden-yielding-an-error-without-return)
- [Effect.catch Losing Type Information](#forbidden-effectcatch-losing-type-information)
- [Variadic catchTag](#forbidden-variadic-catchtag)
- [any/unknown Casts](#forbidden-anyunknown-casts)
- [Promise in Service Signatures](#forbidden-promise-in-service-signatures)
- [console.log](#forbidden-consolelog)
- [process.env Directly](#forbidden-processenv-directly)
- [null/undefined in Domain Types](#forbidden-nullundefined-in-domain-types)
- [Option.getOrThrow](#forbidden-optiongetorthrow)
- [Ignoring Errors with orDie](#forbidden-ignoring-errors-with-ordie)
- [mapError Instead of catchTag](#forbidden-maperror-instead-of-catchtag)
- [Mixing Effect and Promise Chains](#forbidden-mixing-effect-and-promise-chains)
- [Mutable State Without Ref](#forbidden-mutable-state-without-ref)
- [Yielding a Ref, Deferred, or Fiber](#forbidden-yielding-a-ref-deferred-or-fiber)
- [Using Date.now() or new Date() Directly](#forbidden-using-datenow-or-new-date-directly)
- [Deprecated `_` Adaptor in Effect.gen](#forbidden-deprecated-_-adaptor-in-effectgen)
- [Piping the Result of Effect.fn](#forbidden-piping-the-result-of-effectfn)
- [Service Accessors](#forbidden-service-accessors)

These patterns are **never acceptable** in Effect v4 code. Each is listed with
rationale and the correct alternative.

## FORBIDDEN: v3 APIs That No Longer Exist

The single largest source of broken v4 code is muscle memory. These do not
exist:

```typescript
// ❌ All removed in v4
Effect.Service<T>()("T", { effect, dependencies })  // → Context.Service
Context.Tag("T")<T, Shape>()                        // → Context.Service<T, Shape>()("T")
Context.GenericTag<T>("T")                          // → Context.Service<T>("T")
Effect.Tag("T")<T, Shape>()                         // → Context.Service<T, Shape>()("T")
Schema.TaggedError<E>()("E", fields)                // → Schema.TaggedErrorClass
Effect.catchAll(f)                                  // → Effect.catch
Effect.catchAllCause(f)                             // → Effect.catchCause
Effect.catchSome(f)                                 // → Effect.catchFilter
Effect.fork(e)                                      // → Effect.forkChild
Effect.forkDaemon(e)                                // → Effect.forkDetach
Effect.either(e)                                    // → Effect.result
Effect.zipRight(a, b)                               // → Effect.andThen
Layer.scoped(tag, e)                                // → Layer.effect
Layer.unwrapEffect(e)                               // → Layer.unwrap
Layer.lazy(f)                                       // → Layer.suspend
Metric.increment(c)                                 // → Metric.update(c, 1)
Config.integer("X")                                 // → Config.int("X")
Config.validate({...})                              // → Config.schema(codec, path)
Schema.UUID                                         // → Schema.String.check(Schema.isUUID())
Schema.Union(A, B)                                  // → Schema.Union([A, B])
Schema.filter(pred)                                 // → .check(Schema.makeFilter(pred))
Schema.Schema.Type<typeof S>                        // → typeof S.Type
Either                                              // → Result
Mailbox                                             // → Queue
import ... from "@effect/platform"                  // → "effect/unstable/http" etc.
import ... from "@effect/rpc"                       // → "effect/unstable/rpc"
```

See `v3-to-v4-migration.md` for the full map.

## FORBIDDEN: Effect.runSync/runPromise Inside Services

```typescript
// ❌ FORBIDDEN
static readonly layer = Layer.effect(
  UserService,
  Effect.gen(function* () {
    const findById = (id: UserId) => {
      // Running effects synchronously breaks composition
      const user = Effect.runSync(repo.findById(id))
      return user
    }
    return UserService.of({ findById })
  }),
)
```

**Why:** breaks Effect's composition model, loses error handling, can't be
tested, loses tracing.

**Correct:**

```typescript
const findById = Effect.fn("UserService.findById")(function* (id: UserId) {
  return yield* repo.findById(id)
})
```

Running an Effect belongs at the process boundary: `NodeRuntime.runMain`,
`Layer.launch`, or a `ManagedRuntime` you hand to a framework.

## FORBIDDEN: throw Inside Effect.gen

```typescript
// ❌ FORBIDDEN
Effect.gen(function* () {
  const user = yield* repo.findById(id)
  if (!user) {
    throw new Error("User not found") // Bypasses the error channel
  }
  return user
})
```

**Why:** throws become defects, can't be caught with `catchTag`, and don't
appear in the type.

**Correct:**

```typescript
Effect.gen(function* () {
  const user = yield* repo.findById(id)
  if (!user) {
    return yield* new UserNotFoundError({ userId: id, message: "Not found" })
  }
  return user
})
```

## FORBIDDEN: Yielding an Error Without `return`

```typescript
// ❌ FORBIDDEN
Effect.gen(function* () {
  if (invalid) {
    yield* new ValidationError({ message: "bad" }) // no return
  }
  return computeSomething(input) // TS still thinks this is reachable
})
```

**Why:** without `return`, TypeScript keeps narrowing the rest of the body as
reachable, so you lose exhaustiveness checking and can end up with a `never`
value flowing onward.

**Correct:**

```typescript
Effect.gen(function* () {
  if (invalid) {
    return yield* new ValidationError({ message: "bad" })
  }
  return computeSomething(input)
})
```

## FORBIDDEN: Effect.catch Losing Type Information

```typescript
// ❌ FORBIDDEN (Effect.catch is v4's catchAll)
yield* someEffect.pipe(
  Effect.catch(() => Effect.fail(new GenericError({ message: "Something failed" }))),
)
```

**Why:** loses specific error information, makes debugging harder, prevents
targeted recovery downstream.

**Correct:**

```typescript
// Different handlers per tag → catchTags
yield* someEffect.pipe(
  Effect.catchTags({
    DatabaseError: (err) => new ServiceUnavailableError({ message: err.message }),
    ValidationError: (err) => new BadRequestError({ message: err.message }),
  }),
)

// Same handler for several tags → catchTag with an ARRAY of tags
yield* someEffect.pipe(
  Effect.catchTag(
    ["DatabaseError", "ConnectionError"],
    (err) => new ServiceUnavailableError({ message: err.message }),
  ),
)
```

## FORBIDDEN: Variadic catchTag

```typescript
// ❌ FORBIDDEN — this was v3 syntax; it does not type-check in v4
Effect.catchTag("DatabaseError", "ConnectionError", handler)

// ✅ CORRECT
Effect.catchTag(["DatabaseError", "ConnectionError"], handler)
```

## FORBIDDEN: any/unknown Casts

```typescript
// ❌ FORBIDDEN
const data = someValue as any
const result = (await fetch(url)) as unknown as MyType
```

**Why:** completely bypasses type safety, can cause runtime errors, loses
Effect's guarantees.

**Correct:**

```typescript
// Use Schema for parsing unknown data
const result = yield* Schema.decodeUnknownEffect(MyType)(someValue)

// Or the Predicate module's guards — never hand-roll isString/isRecord
if (Predicate.isObject(someValue) && Predicate.isString(someValue.name)) {
  // Now safely typed
}
```

## FORBIDDEN: Promise in Service Signatures

```typescript
// ❌ FORBIDDEN
export class UserService extends Context.Service<UserService, {
  findById(id: UserId): Promise<User>
}>()("myapp/UserService") {}
```

**Why:** loses Effect's error handling, can't compose, loses tracing and
metrics, can't be interrupted.

**Correct:**

```typescript
export class UserService extends Context.Service<UserService, {
  findById(id: UserId): Effect.Effect<User, UserNotFoundError>
}>()("myapp/UserService") {}
```

Wrap third-party promises at the edge with `Effect.tryPromise` /
`Effect.promise`, not inside your service's public shape.

## FORBIDDEN: console.log

```typescript
// ❌ FORBIDDEN
console.log("Processing order:", orderId)
console.error("Error:", error)
```

**Why:** not structured, not captured by Effect's logging system, lost in
production telemetry, not testable.

**Correct:**

```typescript
yield* Effect.log("Processing order", { orderId })
yield* Effect.logError("Operation failed", { error: String(error) })
```

## FORBIDDEN: process.env Directly

```typescript
// ❌ FORBIDDEN
const apiKey = process.env.API_KEY
const port = parseInt(process.env.PORT || "3000")
```

**Why:** no validation, no type safety, fails silently if missing, hard to test.

**Correct:**

```typescript
const config = yield* Config.all({
  apiKey: Config.redacted("API_KEY"),
  port: Config.port("PORT").pipe(Config.withDefault(3000)),
})
```

## FORBIDDEN: null/undefined in Domain Types

```typescript
// ❌ FORBIDDEN
type User = {
  name: string
  bio: string | null
  avatar: string | undefined
}
```

**Why:** null/undefined handling is error-prone and loses the explicit
"absence" semantics that combinators can work with.

**Correct:**

```typescript
const User = Schema.Struct({
  name: Schema.String,
  bio: Schema.Option(Schema.String),
  avatar: Schema.Option(Schema.String),
})
```

At the *encoded* boundary (a database row, a JSON payload) `NullOr` and
`optional` are fine — that's the wire format. The rule is about the decoded
domain type.

## FORBIDDEN: Option.getOrThrow

```typescript
// ❌ FORBIDDEN
const user = Option.getOrThrow(maybeUser)
```

**Why:** throws, bypasses Effect's error handling, fails at runtime instead of
compile time.

**Correct:**

```typescript
// Handle both cases explicitly
yield* Option.match(maybeUser, {
  onNone: () => new UserNotFoundError({ userId, message: "Not found" }),
  onSome: Effect.succeed,
})

// Or provide a default
const name = Option.getOrElse(maybeName, () => "Anonymous")

// Or, inside a generator, yield it — None becomes a typed NoSuchElementError
const user = yield* maybeUser
```

## FORBIDDEN: Ignoring Errors with orDie

```typescript
// ❌ FORBIDDEN (in most cases)
yield* someEffect.pipe(Effect.orDie)
```

**Why:** converts recoverable errors into defects, losing both the information
and the ability to recover.

**Acceptable exceptions:** truly unrecoverable situations (invalid program
state), after exhausting all recovery options, finalizers that cannot
meaningfully fail, and test setup code.

**Correct:**

```typescript
yield* someEffect.pipe(
  Effect.catchTag(
    "RecoverableError",
    (err) => new DomainError({ message: err.message }),
  ),
)
```

## FORBIDDEN: mapError Instead of catchTag

```typescript
// ❌ FORBIDDEN
yield* effect.pipe(
  Effect.mapError((err) => new GenericError({ message: String(err) })),
)
```

**Why:** collapses the error union, so you can no longer discriminate.

**Correct:**

```typescript
yield* effect.pipe(
  Effect.catchTag(
    "SpecificError",
    (err) => new MappedError({ message: err.message }),
  ),
)
```

**Narrow exception:** `Effect.mapError` as an argument to `Effect.fn`, wrapping
a whole repository method's single failure mode into one domain error, is
idiomatic — there the error union is genuinely one thing:

```typescript
const findById = Effect.fn("UserRepo.findById")(
  function* (id: UserId) { /* … */ },
  Effect.mapError((cause) => new UserRepoError({ cause })),
)
```

## FORBIDDEN: Mixing Effect and Promise Chains

```typescript
// ❌ FORBIDDEN
const result = await someEffect
  .pipe(Effect.runPromise)
  .then((data) => Effect.runPromise(anotherEffect(data)))
```

**Why:** loses composition, error handling becomes inconsistent, interruption
is lost between the two runs.

**Correct:**

```typescript
const program = Effect.gen(function* () {
  const data = yield* someEffect
  return yield* anotherEffect(data)
})

const result = await Effect.runPromise(program)
```

## FORBIDDEN: Mutable State Without Ref

```typescript
// ❌ FORBIDDEN
let counter = 0
const increment = Effect.sync(() => {
  counter++
})
```

**Why:** race conditions under concurrency, not testable, not composable,
breaks referential transparency.

**Correct:**

```typescript
const program = Effect.gen(function* () {
  const counter = yield* Ref.make(0)
  yield* Ref.update(counter, (n) => n + 1)
  return yield* Ref.get(counter)
})
```

Closure-captured mutable state **inside a layer's construction effect** is
acceptable when the state is owned entirely by that service instance and never
escapes — but `Ref` is still safer under concurrency.

## FORBIDDEN: Yielding a Ref, Deferred, or Fiber

```typescript
// ❌ FORBIDDEN — worked in v3 via Effect subtyping, removed in v4
const value = yield* ref
const value = yield* deferred
const result = yield* fiber
```

**Why:** v4 replaced Effect subtyping with the narrower `Yieldable` trait
precisely because "I have a Ref" and "I have an Effect that reads the Ref" were
indistinguishable, which caused silent bugs.

**Correct:**

```typescript
const value = yield* Ref.get(ref)
const value = yield* Deferred.await(deferred)
const result = yield* Fiber.join(fiber)
```

Likewise, `Option` and `Result` are yieldable but no longer `Effect`s — passing
one to a combinator needs `.asEffect()`:

```typescript
Effect.map(maybeUser.asEffect(), (u) => u.name)
```

## FORBIDDEN: Using Date.now() or new Date() Directly

```typescript
// ❌ FORBIDDEN
const now = new Date()
const timestamp = Date.now()
```

**Why:** not testable, non-deterministic, can't be controlled by `TestClock`.

**Correct:**

```typescript
import { Clock, DateTime } from "effect"

const millis = yield* Clock.currentTimeMillis
const now = yield* DateTime.now // Clock-powered, testable
```

## FORBIDDEN: Deprecated `_` Adaptor in Effect.gen

```typescript
// ❌ FORBIDDEN
Effect.gen(function* (_) {
  const user = yield* _(repo.findById(id))
  return user
})
```

**Correct:**

```typescript
Effect.gen(function* () {
  const user = yield* repo.findById(id)
  return user
})
```

Also note: passing `this` changed shape in v4 — `Effect.gen({ self: this },
function* () { … })`, not `Effect.gen(this, function* () { … })`.

## FORBIDDEN: Piping the Result of Effect.fn

```typescript
// ❌ FORBIDDEN — pipes the function value, not the effect it returns
const findById = Effect.fn("UserRepo.findById")(function* (id: UserId) {
  // ...
}).pipe(Effect.mapError(/* … */))
```

**Correct:** pass combinators as extra arguments so they apply per call:

```typescript
const findById = Effect.fn("UserRepo.findById")(
  function* (id: UserId) {
    // ...
  },
  Effect.mapError((cause) => new UserRepoError({ cause })),
)
```

## FORBIDDEN: Service Accessors

```typescript
// ❌ FORBIDDEN — v3 accessors were removed
const user = yield* UserService.findById(id)
```

**Why:** the v3 accessor proxy was built from mapped types, which erased generic
type parameters and overloads (`get<T>(key): Effect<T>` collapsed to
`get(key): Effect<unknown>`).

**Correct:**

```typescript
// Preferred — the dependency is visible at the call site
const users = yield* UserService
const user = yield* users.findById(id)

// Acceptable one-liner, but it hides the dependency from the body
const user = yield* UserService.use((s) => s.findById(id))
```
