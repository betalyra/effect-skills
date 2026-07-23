# Effect Testing Patterns (Effect v4)

## Table of Contents

- [Framework Selection](#framework-selection)
- [What Changed From v3](#what-changed-from-v3)
- [Test Variants](#test-variants)
- [Shared Layers with `layer(...)`](#shared-layers-with-layer)
- [Effect-Specific Assertions](#effect-specific-assertions)
- [Testing Success and Failure](#testing-success-and-failure)
- [Mock Layers for Testing](#mock-layers-for-testing)
- [Testing Error Scenarios](#testing-error-scenarios)
- [Time-Dependent Testing with TestClock](#time-dependent-testing-with-testclock)
- [Testing Resource Management](#testing-resource-management)
- [Property-Based Testing](#property-based-testing)
- [Testing Best Practices](#testing-best-practices)
- [Common Pitfalls](#common-pitfalls)

## Framework Selection

Use `@effect/vitest` (at the matching v4 version) when testing:

- Functions that return `Effect<A, E, R>`
- Code that uses services and layers
- Time-dependent operations with `TestClock`
- Asynchronous operations coordinated with Effect

```typescript
import { assert, it } from "@effect/vitest"
import { Effect } from "effect"

declare const fetchUser: (id: string) => Effect.Effect<{ id: string }, Error>

it.effect("should fetch user", () =>
  Effect.gen(function* () {
    const user = yield* fetchUser("123")
    assert.strictEqual(user.id, "123")
  }),
)
```

`@effect/vitest` re-exports everything from `vitest`, so `describe`, `expect`,
and `assert` all come from the same import.

## What Changed From v3

| v3                                  | v4                                            |
| ----------------------------------- | --------------------------------------------- |
| `it.scoped` / `it.scopedLive`       | Gone — `it.effect` / `it.live` include `Scope` |
| `import { TestClock } from "effect"`| `import { TestClock } from "effect/testing"`  |
| `import { FastCheck } from "effect"`| `import { FastCheck } from "effect/testing"`  |
| `assertRight` / `assertLeft`        | `assertSuccess` / `assertFailure` (on `Result`) |
| `Effect.fork`                       | `Effect.forkChild`                            |
| `Effect.catchAll`                   | `Effect.catch`                                |
| `yield* someRef`                    | `yield* Ref.get(someRef)`                     |
| `Context.Tag` test doubles          | `Context.Service` + `Layer.succeed` / `Layer.mock` |

## Test Variants

### `it.effect` — Default Test Environment

Runs with the test services: `TestClock`, deterministic `Random`, and a `Scope`.

```typescript
import { assert, it } from "@effect/vitest"
import { Effect } from "effect"

it.effect("test name", () =>
  Effect.gen(function* () {
    const result = yield* someEffect
    assert.strictEqual(result, expected)
  }),
)
```

Because the scope is built in, resource tests need no special variant:

```typescript
it.effect("test with resources", () =>
  Effect.gen(function* () {
    const resource = yield* Effect.acquireRelease(acquire, () => release)
    // Released when the test finishes
  }),
)
```

### `it.live` — Live Environment

Real clock, real randomness — for tests that must observe wall time or real I/O.

```typescript
import { it } from "@effect/vitest"
import { Clock, Effect } from "effect"

it.live("test with real time", () =>
  Effect.gen(function* () {
    const now = yield* Clock.currentTimeMillis
    // Uses actual system time
  }),
)
```

### Modifiers

`it.effect` and `it.live` both carry `.skip`, `.only`, `.fails`, `.skipIf`,
`.runIf`, `.each`, and `.prop`:

```typescript
it.effect.each([
  { input: " Ada ", expected: "ada" },
  { input: " Lin ", expected: "lin" },
])("normalizes %#", ({ input, expected }) =>
  Effect.gen(function* () {
    assert.strictEqual(input.trim().toLowerCase(), expected)
  }),
)
```

### `flakyTest`

For genuinely non-deterministic externals, `it.flakyTest` retries an effect for
a duration rather than failing on the first attempt. Use it sparingly — a flaky
test is usually a design signal, not a retry problem.

## Shared Layers with `layer(...)`

`layer(...)` builds a layer **once** for a describe block and tears it down in
`afterAll`. Every test inside can yield the services directly, without piping
`Effect.provide` onto each one:

```typescript
import { assert, layer } from "@effect/vitest"
import { Context, Effect, Layer, Ref } from "effect"

class TodoRepo extends Context.Service<TodoRepo, {
  create(title: string): Effect.Effect<Todo>
  readonly list: Effect.Effect<ReadonlyArray<Todo>>
}>()("app/TodoRepo") {
  static readonly layerTest = Layer.effect(
    TodoRepo,
    Effect.gen(function* () {
      const store = yield* TodoRepoTestRef

      const create = Effect.fn("TodoRepo.create")(function* (title: string) {
        const todos = yield* Ref.get(store)
        const todo = { id: todos.length + 1, title }
        yield* Ref.set(store, [...todos, todo])
        return todo
      })

      return TodoRepo.of({ create, list: Ref.get(store) })
    }),
  ).pipe(
    // provideMerge so tests can also reach the underlying Ref
    Layer.provideMerge(TodoRepoTestRef.layer),
  )
}

layer(TodoRepo.layerTest)("TodoRepo", (it) => {
  it.effect("starts empty", () =>
    Effect.gen(function* () {
      const repo = yield* TodoRepo
      assert.strictEqual((yield* repo.list).length, 0)
      yield* repo.create("Write docs")
      assert.strictEqual((yield* repo.list).length, 1)
    }),
  )

  it.effect("state is shared across tests in the block", () =>
    Effect.gen(function* () {
      const repo = yield* TodoRepo
      // The todo from the previous test is still there
      assert.strictEqual((yield* repo.list).length, 1)
    }),
  )
})
```

**The state persists between tests in the same `layer` block.** That's the
point — it lets you model a sequence — but it also means order matters. If you
want isolation, use `Effect.provide` per test instead.

`layer` blocks nest: the inner `it` argument is scoped to the merged services.

## Effect-Specific Assertions

```typescript
import { assert, it } from "@effect/vitest"
import {
  assertFailure, // Result.Failure
  assertNone,
  assertSome,
  assertSuccess, // Result.Success
  assertExitFailure,
  assertExitSuccess,
  assertInstanceOf,
  deepStrictEqual,
  strictEqual,
} from "@effect/vitest/utils"
import { Effect, Option, Result } from "effect"

it.effect("with effect assertions", () =>
  Effect.gen(function* () {
    const option = yield* someOptionalEffect
    assertSome(option, expectedValue)

    // Either → Result in v4; assertRight/assertLeft → assertSuccess/assertFailure
    const result = yield* someResultEffect
    assertSuccess(result, expectedValue)
  }),
)
```

`addEqualityTesters()` registers Effect's `Equal.equals` with Vitest so
`expect(a).toEqual(b)` respects structural equality on Effect data types. Call
it once in a setup file.

## Testing Success and Failure

```typescript
import { assert, describe, it } from "@effect/vitest"
import { Cause, Effect, Exit } from "effect"

describe("Validation", () => {
  it.effect("should succeed with valid email", () =>
    Effect.gen(function* () {
      const result = yield* validateEmail("alice@example.com")
      assert.strictEqual(result, "alice@example.com")
    }),
  )

  it.effect("should fail with invalid email", () =>
    Effect.gen(function* () {
      const exit = yield* Effect.exit(validateEmail("invalid"))
      assert.isTrue(Exit.isFailure(exit))
      if (Exit.isFailure(exit)) {
        const failure = Cause.findError(exit.cause)
        assert.isTrue(Result.isSuccess(failure))
        if (Result.isSuccess(failure)) {
          assert.strictEqual(failure.success._tag, "ValidationError")
        }
      }
    }),
  )
})
```

`Cause.findError` returns a `Result<E, Cause<never>>` — success means a typed
failure was found, failure means the cause held only defects/interrupts. This
replaces v3's `Cause.failureOrCause`.

## Mock Layers for Testing

### Static fakes

```typescript
import { Context, Effect, Layer, Option } from "effect"

class UserRepository extends Context.Service<UserRepository, {
  findById(id: string): Effect.Effect<Option.Option<User>, DbError>
  save(user: User): Effect.Effect<User, DbError>
}>()("app/UserRepository") {}

const UserRepositoryTest = Layer.succeed(
  UserRepository,
  UserRepository.of({
    findById: (id) =>
      Effect.succeed(
        id === "1"
          ? Option.some({ id: "1", name: "Alice", email: "alice@example.com" })
          : Option.none(),
      ),
    save: (user) => Effect.succeed(user),
  }),
)
```

### Partial mocks with `Layer.mock`

`Layer.mock` implements only the members you list. Anything else throws
`UnimplementedError` when called — which is what you want: an unexpected call
fails the test loudly instead of quietly succeeding against a stub.

```typescript
const testUserLayer = Layer.mock(UserService, {
  getUser: (id: string) => Effect.succeed({ id, name: "Test User" }),
  // deleteUser / updateUser omitted deliberately
})
```

### Stateful mocks

Expose the state as its own service and `provideMerge` it, so tests can assert
on the store directly rather than only through the interface:

```typescript
import { Array, Context, Effect, Layer, Option, Ref } from "effect"

class UserStoreRef extends Context.Service<
  UserStoreRef,
  Ref.Ref<ReadonlyArray<User>>
>()("app/test/UserStoreRef") {
  static readonly layer = Layer.effect(UserStoreRef, Ref.make(Array.empty()))
}

const UserRepositoryStateful = Layer.effect(
  UserRepository,
  Effect.gen(function* () {
    const store = yield* UserStoreRef

    return UserRepository.of({
      findById: (id) =>
        Ref.get(store).pipe(
          Effect.map(Array.findFirst((u) => u.id === id)),
        ),
      save: (user) =>
        Ref.update(store, (users) => [...users, user]).pipe(Effect.as(user)),
    })
  }),
).pipe(Layer.provideMerge(UserStoreRef.layer))

it.effect("should save and retrieve user", () =>
  Effect.gen(function* () {
    const repo = yield* UserRepository
    yield* repo.save({ id: "2", name: "Bob", email: "bob@example.com" })

    const result = yield* repo.findById("2")
    assertSome(result, { id: "2", name: "Bob", email: "bob@example.com" })

    // Assert on the raw state too
    const all = yield* Ref.get(yield* UserStoreRef)
    assert.strictEqual(all.length, 1)
  }).pipe(Effect.provide(UserRepositoryStateful)),
)
```

Note `Ref.get(store)` — in v4 a `Ref` is not yieldable, so `yield* store` no
longer reads it.

## Testing Error Scenarios

### Testing Expected Failures with `Effect.flip`

The cleanest way to assert on an error is to flip it into the success channel:

```typescript
import { assert, it } from "@effect/vitest"
import { Effect, Schema } from "effect"

class UserNotFoundError extends Schema.TaggedErrorClass<UserNotFoundError>()(
  "UserNotFoundError",
  { userId: Schema.String },
) {}

it.effect("should fail with error", () =>
  Effect.gen(function* () {
    const error = yield* Effect.flip(failingOperation())
    assert.instanceOf(error, UserNotFoundError)
    assert.strictEqual(error.userId, "123")
  }),
)
```

### Testing Error Recovery

```typescript
it.effect("should handle NotFoundError", () =>
  Effect.gen(function* () {
    const result = yield* fetchUser("999").pipe(
      Effect.catchTag("NotFoundError", () =>
        Effect.succeed({ id: "default", name: "Guest" }),
      ),
    )
    assert.strictEqual(result.name, "Guest")
  }).pipe(Effect.provide(TestLayer)),
)
```

Remember the v4 syntax for multiple tags:
`Effect.catchTag(["A", "B"], handler)`.

## Time-Dependent Testing with TestClock

`TestClock` now lives in `effect/testing`.

```typescript
import { assert, it } from "@effect/vitest"
import { Effect, Fiber } from "effect"
import { TestClock } from "effect/testing"

it.effect("should handle delays", () =>
  Effect.gen(function* () {
    const fiber = yield* Effect.forkChild(
      Effect.sleep("5 seconds").pipe(Effect.as("done")),
    )

    // Advance virtual time instantly
    yield* TestClock.adjust("5 seconds")

    const result = yield* Fiber.join(fiber)
    assert.strictEqual(result, "done")
  }),
)
```

Note `Effect.forkChild` (v3's `Effect.fork`) and `Fiber.join(fiber)` — a `Fiber`
is no longer yieldable in v4.

### Testing Recurring Effects

```typescript
import { Effect, Option, Queue } from "effect"
import { TestClock } from "effect/testing"

it.effect("should execute every minute", () =>
  Effect.gen(function* () {
    const queue = yield* Queue.unbounded<number>()

    yield* Effect.forkChild(
      Queue.offer(queue, 1).pipe(Effect.delay("60 seconds"), Effect.forever),
    )

    // Nothing before time passes
    assert.isTrue(Option.isNone(yield* Queue.poll(queue)))

    yield* TestClock.adjust("60 seconds")

    assert.strictEqual(yield* Queue.take(queue), 1)

    // Verify exactly one execution
    assert.isTrue(Option.isNone(yield* Queue.poll(queue)))
  }),
)
```

### TestClock with Deferred

```typescript
it.effect("should handle deferred with delays", () =>
  Effect.gen(function* () {
    const deferred = yield* Deferred.make<number, void>()

    yield* Effect.forkChild(
      Effect.sleep("10 seconds").pipe(
        // v3's zipRight is v4's andThen
        Effect.andThen(Deferred.succeed(deferred, 42)),
      ),
    )

    yield* TestClock.adjust("10 seconds")

    const result = yield* Deferred.await(deferred)
    assert.strictEqual(result, 42)
  }),
)
```

## Testing Resource Management

```typescript
import { assert, describe, it } from "@effect/vitest"
import { Effect, Ref } from "effect"

describe("Resource Management", () => {
  it.effect("should clean up resources on success", () =>
    Effect.gen(function* () {
      const cleaned = yield* Ref.make(false)

      yield* Effect.scoped(
        Effect.gen(function* () {
          yield* Effect.addFinalizer(() => Ref.set(cleaned, true))
          return "done"
        }),
      )

      assert.isTrue(yield* Ref.get(cleaned))
    }),
  )

  it.effect("should clean up resources on failure", () =>
    Effect.gen(function* () {
      const cleaned = yield* Ref.make(false)

      const result = yield* Effect.scoped(
        Effect.gen(function* () {
          yield* Effect.addFinalizer(() => Ref.set(cleaned, true))
          return yield* Effect.fail({ _tag: "TestError" as const })
        }),
      ).pipe(Effect.catch(() => Effect.succeed("handled")))

      assert.strictEqual(result, "handled")
      assert.isTrue(yield* Ref.get(cleaned))
    }),
  )
})
```

## Property-Based Testing

`FastCheck` moved to `effect/testing`, and arbitraries can now be **Schemas
directly** — no separate arbitrary derivation step.

### `it.prop` for Pure Properties

```typescript
import { it } from "@effect/vitest"
import { FastCheck } from "effect/testing"

it.prop(
  "addition is commutative",
  [FastCheck.integer(), FastCheck.integer()],
  ([a, b]) => a + b === b + a,
)

// Object syntax
it.prop(
  "multiplication distributes",
  { a: FastCheck.integer(), b: FastCheck.integer(), c: FastCheck.integer() },
  ({ a, b, c }) => a * (b + c) === a * b + a * c,
)
```

### `it.effect.prop` for Effect Properties

```typescript
import { assert, it } from "@effect/vitest"
import { Effect, Schema } from "effect"

it.effect.prop("reversing twice is identity", [Schema.String], ([value]) =>
  Effect.gen(function* () {
    const twice = value.split("").reverse().reverse().join("")
    assert.strictEqual(twice, value)
  }),
)
```

### With Schema Arbitraries

```typescript
const User = Schema.Struct({
  id: Schema.String,
  age: Schema.Int.check(Schema.isBetween(0, 120)),
})

it.effect.prop("user validation works", { user: User }, ({ user }) =>
  Effect.gen(function* () {
    assert.isTrue(user.age >= 0 && user.age <= 120)
  }),
)
```

Note `Schema.isBetween` — v3's `Schema.between` filter.

### Configuring FastCheck

```typescript
it.effect.prop(
  "property test",
  [FastCheck.integer()],
  ([n]) => Effect.succeed(n >= 0 || n < 0),
  {
    timeout: 10_000,
    fastCheck: { numRuns: 1000, seed: 42, verbose: true },
  },
)
```

## Testing Best Practices

1. **Use test layers.** Create dedicated test implementations for services, and
   reach for `Layer.mock` when you only need a couple of methods.
2. **Test error paths.** Both success and failure — `Effect.flip` makes the
   error assertion read as plainly as the success one.
3. **Mock dependencies, not the system under test.** The service you're testing
   uses its real implementation; only its collaborators are faked. This is what
   `layerNoDeps` is for.
4. **Test cleanup.** Assert that finalizers ran.
5. **Use property tests** for invariants; Schema arbitraries make them cheap.
6. **Isolate tests** unless you deliberately want a shared `layer(...)` block.
7. **Test interruption** for anything concurrent.
8. **Use `TestClock`, never real sleeps.** A test that waits is a test that
   flakes.

## Common Pitfalls

1. **Reaching for `it.scoped`.** It no longer exists — `it.effect` already
   provides a `Scope`.
2. **Importing `TestClock` or `FastCheck` from `effect`.** They live in
   `effect/testing`.
3. **`yield* ref` / `yield* fiber`.** Use `Ref.get` and `Fiber.join`.
4. **Forgetting that `layer(...)` shares state** across tests in the block.
5. **Not providing layers**, or providing them at the wrong granularity.
6. **Only testing happy paths.**
7. **Over-mocking** — mocking so much that the test asserts nothing real.
8. **Hardcoded timing** instead of `TestClock`.
9. **Missing Exit checks** — asserting an effect "didn't throw" rather than
   asserting the actual `Exit`.

## Resources

- [Vitest](https://vitest.dev/)
- [fast-check](https://github.com/dubzzz/fast-check)
