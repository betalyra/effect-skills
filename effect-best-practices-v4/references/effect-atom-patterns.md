# Effect Atom Patterns (Effect v4)

## Table of Contents

- [Where Atom Lives in v4](#where-atom-lives-in-v4)
- [Core Concepts](#core-concepts)
- [Creating Atoms](#creating-atoms)
- [Atom Families](#atom-families)
- [Function Atoms (Side Effects)](#function-atoms-side-effects)
- [Runtime with Services](#runtime-with-services)
- [React Integration](#react-integration)
- [Working with Effects and AsyncResults](#working-with-effects-and-asyncresults)
- [Stream Integration](#stream-integration)
- [Pull Atoms (Pagination)](#pull-atoms-pagination)
- [Scoped Resources & Finalizers](#scoped-resources--finalizers)
- [Common Patterns](#common-patterns)
- [Batching Updates](#batching-updates)
- [localStorage Persistence](#localstorage-persistence)
- [Anti-Patterns](#anti-patterns)
- [Performance Tips](#performance-tips)

## Where Atom Lives in v4

Two things moved:

| v3                                   | v4                                    |
| ------------------------------------ | ------------------------------------- |
| `@effect-atom/atom-react` (package)  | `@effect/atom-react`                  |
| `Atom` from the React package        | `effect/unstable/reactivity/Atom`     |
| `Result` from the React package      | `effect/unstable/reactivity/AsyncResult` |

The React package now only exports **hooks and registry plumbing**
(`useAtomValue`, `useAtomSet`, `useAtom`, `useAtomMount`, `useAtomRefresh`,
`useAtomSuspense`, `useAtomSubscribe`, `useAtomRef*`, `RegistryContext`,
`ScopedAtom`, hydration helpers). The atom *primitives* come from core:

```typescript
import { Atom, AsyncResult } from "effect/unstable/reactivity"
import { useAtomValue, useAtomSet } from "@effect/atom-react"
```

Sibling packages exist for other frameworks: `@effect/atom-solid`,
`@effect/atom-vue`.

**`Result` → `AsyncResult`** is a rename of the whole module. Everything you
know — `initial`, `success`, `failure`, `builder`, `getOrElse`, `match` — is
there under the new name. Note that `Result` in v4 core is a *different* type
(the replacement for `Either`), so mixing them up is a real hazard.

## Core Concepts

- **Atoms** — reactive state containers with automatic dependency tracking
- **AsyncResult** — models async/effectful computation: initial, waiting,
  success, failure
- **Finalizers** — built-in cleanup for resources and event listeners
- **Families** — dynamic atom creation for per-entity state

## Creating Atoms

### Basic Atoms

```typescript
import { Atom } from "effect/unstable/reactivity"

// Simple value atom
const countAtom = Atom.make(0)

// With keepAlive — persists when no components subscribe
const persistentCountAtom = Atom.make(0).pipe(Atom.keepAlive)
```

**Rule:** use `Atom.keepAlive` for global state that should survive component
unmounts. `Atom.autoDispose` is the explicit opposite, and `Atom.setIdleTTL`
lets you keep an atom alive for a grace period after its last subscriber goes
away — a good middle ground for expensive queries.

### Derived Atoms

```typescript
const countAtom = Atom.make(0)

// Derived using the get function
const doubleCountAtom = Atom.make((get) => get(countAtom) * 2)

// Derived using Atom.map
const tripleCountAtom = Atom.map(countAtom, (count) => count * 3)
```

### Atoms with Side Effects

```typescript
const scrollYAtom = Atom.make((get) => {
  const onScroll = () => get.setSelf(window.scrollY)

  window.addEventListener("scroll", onScroll)
  get.addFinalizer(() => window.removeEventListener("scroll", onScroll))

  return window.scrollY
}).pipe(Atom.keepAlive)
```

**Critical:**

- Use `get.setSelf` to update the atom's own value
- Always register cleanup with `get.addFinalizer()`
- Finalizers run when the atom is rebuilt or disposed

### Atom.transform for Self-Updating Derived State

```typescript
const resolvedThemeAtom = Atom.transform(themeAtom, (get) => {
  const theme = get(themeAtom)
  if (theme !== "system") return theme

  const matcher = window.matchMedia("(prefers-color-scheme: dark)")
  const onChange = () => get.setSelf(matcher.matches ? "dark" : "light")

  matcher.addEventListener("change", onChange)
  get.addFinalizer(() => matcher.removeEventListener("change", onChange))

  return matcher.matches ? "dark" : "light"
})
```

## Atom Families

Use `Atom.family` for per-entity state:

```typescript
import { Atom } from "effect/unstable/reactivity"

const replyToMessageAtomFamily = Atom.family((channelId: string) =>
  Atom.make<string | null>(null).pipe(Atom.keepAlive),
)

type ModalType = "settings" | "confirm" | "create"

interface ModalState {
  readonly type: ModalType
  readonly isOpen: boolean
  readonly metadata?: Record<string, unknown>
}

const modalAtomFamily = Atom.family((type: ModalType) =>
  Atom.make<ModalState>({ type, isOpen: false, metadata: undefined }).pipe(
    Atom.keepAlive,
  ),
)
```

**Use families for:** per-resource state (users, channels, documents), modal
instances, form state per entity — anything parameterised.

Family entries are held weakly (via `WeakRef`/`FinalizationRegistry` where
available), so unused keys are collected. Combining `family` with `keepAlive`
means "keep this key alive while something still references the key".

## Function Atoms (Side Effects)

Use `Atom.fn` for operations with side effects:

```typescript
import { Atom } from "effect/unstable/reactivity"
import { Effect } from "effect"

interface CartState {
  readonly items: ReadonlyArray<Item>
  readonly total: number
}

const cart = Atom.make<CartState>({ items: [], total: 0 })

const addItem = Atom.fn(
  Effect.fnUntraced(function* (item: Item) {
    const current = yield* Atom.get(cart)
    yield* Atom.set(cart, {
      items: [...current.items, item],
      total: current.total + item.price,
    })
  }),
)

const clearCart = Atom.fn(
  Effect.fnUntraced(function* () {
    yield* Atom.set(cart, { items: [], total: 0 })
  }),
)
```

`Atom.fnSync` is the synchronous variant. `Atom.optimisticFn` pairs a write with
an optimistic local update that rolls back on failure — see
[Optimistic Updates](#optimistic-updates).

## Runtime with Services

Wrap Effect layers for use in atoms. Note the v4 layer naming (`layer`, not
`Live`):

```typescript
import { Atom } from "effect/unstable/reactivity"
import { Effect, Layer } from "effect"

const runtime = Atom.runtime(
  Layer.mergeAll(DatabaseService.layer, LoggerService.layer, ApiClient.layer),
)

// runtime.atom for effectful reads
const currentUserAtom = runtime.atom(
  Effect.gen(function* () {
    const db = yield* DatabaseService
    return yield* db.currentUser
  }),
)

// runtime.fn for effectful writes
const fetchUserData = runtime.fn(
  Effect.fnUntraced(function* (userId: string) {
    const db = yield* DatabaseService
    const user = yield* db.getUser(userId)
    yield* Atom.set(userAtoms(userId), user)
    return user
  }),
)
```

### Global Layers

Configure shared layers once at app initialization:

```typescript
Atom.runtime.addGlobalLayer(
  Layer.mergeAll(LoggerLayer, TracerLayer, ConfigLayer),
)
```

## React Integration

### Reading Atom Values

```typescript
import { useAtomValue } from "@effect/atom-react"

function Counter() {
    const count = useAtomValue(countAtom)
    return <span>{count}</span>
}
```

### Updating Atom Values

```typescript
import { useAtomSet } from "@effect/atom-react"

function IncrementButton() {
    const setCount = useAtomSet(countAtom)
    return <button onClick={() => setCount((c) => c + 1)}>Increment</button>
}
```

### Reading and Writing Together

```typescript
import { useAtom } from "@effect/atom-react"

function CounterControl() {
    const [count, setCount] = useAtom(countAtom)
    return (
        <div>
            <span>{count}</span>
            <button onClick={() => setCount(count + 1)}>+1</button>
        </div>
    )
}
```

### Mounting Side-Effect Atoms

`useAtomMount` activates an atom without subscribing to its value:

```typescript
import { useAtomMount } from "@effect/atom-react"

function App() {
    useAtomMount(keyboardShortcutsAtom)
    useAtomMount(presenceTrackingAtom)
    useAtomMount(themeApplierAtom)

    return <>{children}</>
}
```

### Async Function Atoms

`useAtomSet` with a mode gives you a promise back, which is what you want for
"click, then await the result":

```typescript
import { useAtomValue, useAtomSet } from "@effect/atom-react"

function CartView() {
    const cartData = useAtomValue(cart)
    const add = useAtomSet(addItem)
    const clear = useAtomSet(clearCart)

    // Promise-returning variant, for awaiting the effect's result
    const fetchData = useAtomSet(fetchUserData, { mode: "promise" })

    return (
        <div>
            <div>Items: {cartData.items.length}</div>
            <button onClick={() => add(newItem)}>Add</button>
            <button onClick={() => clear()}>Clear</button>
        </div>
    )
}
```

`useAtom` accepts the same modes (`"value" | "promise" | "promiseExit"`).
`useAtomSuspense` integrates with React Suspense; `useAtomRefresh` forces a
re-run of an effectful atom.

## Working with Effects and AsyncResults

### Effectful Atoms Return AsyncResult

```typescript
import { Atom, AsyncResult } from "effect/unstable/reactivity"
import { Effect } from "effect"

const userAtom = Atom.make(
  Effect.gen(function* () {
    return yield* fetchUser()
  }),
) // Atom<AsyncResult<User, Error>>
```

### Handling with AsyncResult.builder (Recommended)

```typescript
import { AsyncResult } from "effect/unstable/reactivity"
import { useAtomValue } from "@effect/atom-react"

function UserProfile() {
    const userResult = useAtomValue(userAtom)

    return AsyncResult.builder(userResult)
        .onInitial(() => <div>Loading...</div>)
        .onError((error) => <div>Error: {error.message}</div>)
        .onSuccess((user) => <div>Hello, {user.name}!</div>)
        .render()
}
```

### builder with Tagged Errors

The key advantage: `onErrorTag` narrows the error type as you handle cases, so
the compiler tells you when you've covered them all.

```typescript
function ResourceEmbed({ url }: { url: string }) {
    const resourceResult = useAtomValue(resourceAtom)

    return AsyncResult.builder(resourceResult)
        .onInitial(() => <Skeleton />)
        .onErrorTag("NotFoundError", (error) => <ErrorCard message={error.message} />)
        .onErrorTag("UnauthorizedError", () => <ConnectPrompt provider="GitHub" />)
        .onErrorTag("RateLimitError", (error) => <RetryCard retryAfter={error.retryAfter} />)
        .onError(() => <ErrorCard message="Something went wrong" />)
        .onSuccess((data) => <ResourceCard data={data} />)
        .render()
}
```

`onErrorTag` also accepts an **array** of tags, mirroring `Effect.catchTag`:

```typescript
.onErrorTag(["TokenExpiredError", "MissingTokenError"], () => <RedirectToLogin />)
```

### builder Methods

| Method                      | Purpose                                            |
| --------------------------- | -------------------------------------------------- |
| `onInitial(fn)`             | Initial/loading state                              |
| `onInitialOrWaiting(fn)`    | Initial **and** waiting states                     |
| `onWaiting(fn)`             | Waiting/refetching state                           |
| `onSuccess(fn)`             | Success with value                                 |
| `onError(fn)`               | Any typed error                                    |
| `onErrorTag(tag \| tags, fn)` | Specific tagged error(s) (removed from the type) |
| `onErrorIf(predicate, fn)`  | Errors matching a predicate or refinement          |
| `onFailure(fn)`             | Failure with the full `Cause`                      |
| `onDefect(fn)`              | Unexpected defects                                 |
| `onInterrupt(fn)`           | Interruption                                       |
| `render()`                  | Return the output (`null` if an unhandled initial) |
| `orElse(fn)`                | Fallback value                                     |
| `orNull()`                  | `null` for unhandled cases                         |
| `exhaustive()`              | Only available once every case is handled          |

`exhaustive()` is worth reaching for in critical UI: it's a compile error until
every state has a branch.

### Extracting Values with orElse

```typescript
function useRepositories() {
  const reposResult = useAtomValue(repositoriesAtom)

  return AsyncResult.builder(reposResult)
    .onSuccess((data) => data.repositories)
    .orElse(() => [])
}
```

### AsyncResult.getOrElse for Simple Extraction

```typescript
function UserName() {
    const userResult = useAtomValue(userAtom)
    const user = AsyncResult.getOrElse(userResult, () => null)

    if (!user) return <span>Loading...</span>
    return <span>{user.name}</span>
}
```

### When to Use Each Pattern

| Pattern                         | Use Case                              |
| ------------------------------- | ------------------------------------- |
| `AsyncResult.builder`           | UI rendering with multiple error types |
| `builder` + `onErrorTag`        | APIs with tagged errors (HttpApi, RPC) |
| `builder` + `orElse`            | Extracting values with a fallback      |
| `AsyncResult.getOrElse`         | Simple value extraction                |
| `AsyncResult.match`             | Simple 3-case exhaustive matching      |

### Accessing Results in Derived Atoms

```typescript
const userProfileAtom = Atom.make(
  Effect.fnUntraced(function* (get: Atom.Context) {
    // Unwrap the AsyncResult (suspends until success)
    const user = yield* get.result(userAtom)
    const posts = yield* fetchUserPosts(user.id)
    return { user, posts }
  }),
)
```

## Stream Integration

Streams become atoms holding the latest value:

```typescript
import { Atom } from "effect/unstable/reactivity"
import { Stream } from "effect"

const notifications = Atom.make(
  Stream.fromEventListener(window, "notification").pipe(
    Stream.map(parseNotification),
    Stream.filter(isValid),
    Stream.scan([], (acc, n) => [...acc, n].slice(-10)),
  ),
)
```

Remember v4's Stream renames — `Stream.async` → `Stream.callback`,
`Stream.repeatEffect` → `Stream.fromEffectRepeat`, `Stream.catchAll` →
`Stream.catch`.

## Pull Atoms (Pagination)

```typescript
import { Atom } from "effect/unstable/reactivity"
import { Stream } from "effect"

const pagedItems = Atom.pull(
  Stream.fromIterable(itemsSource).pipe(Stream.grouped(10)),
)

function ItemList() {
    const loadMore = useAtomSet(pagedItems)
    return <button onClick={() => loadMore()}>Load More</button>
}
```

## Scoped Resources & Finalizers

Effectful atoms run in a `Scope`, so `Effect.acquireRelease` just works:

```typescript
const wsConnection = Atom.make(
  Effect.gen(function* () {
    return yield* Effect.acquireRelease(connectWebSocket(), (ws) =>
      Effect.sync(() => ws.close()),
    )
  }),
)
// The finalizer runs when the atom rebuilds or becomes unused
```

## Common Patterns

### Loading States

Note `Effect.result` — v3's `Effect.either` — and `Result` (not `Either`):

```typescript
import { Atom, AsyncResult } from "effect/unstable/reactivity"
import { Effect, Result } from "effect"

const userDataAtom = Atom.make<AsyncResult.AsyncResult<User, Error>>(
  AsyncResult.initial(),
)

const loadUser = runtime.fn(
  Effect.fnUntraced(function* (id: string) {
    yield* Atom.set(userDataAtom, AsyncResult.initial())

    const result = yield* Effect.result(userService.fetchUser(id))

    yield* Atom.set(
      userDataAtom,
      Result.isSuccess(result)
        ? AsyncResult.success(result.success)
        : AsyncResult.fail(result.failure),
    )
  }),
)
```

In practice, prefer `runtime.atom(effect)` — it produces the `AsyncResult`
lifecycle for you, and `Atom.swr` adds stale-while-revalidate semantics without
hand-rolled state.

### Optimistic Updates

Hand-rolled:

```typescript
const updateItem = runtime.fn(
  Effect.fnUntraced(function* (id: string, updates: Partial<Item>) {
    const current = yield* Atom.get(itemsAtom)

    // Optimistic update
    yield* Atom.set(
      itemsAtom,
      current.map((item) => (item.id === id ? { ...item, ...updates } : item)),
    )

    const result = yield* Effect.result(api.updateItem(id, updates))

    // Revert on failure
    if (Result.isFailure(result)) {
      yield* Atom.set(itemsAtom, current)
    }
  }),
)
```

v4 also ships `Atom.optimistic` / `Atom.optimisticFn`, which encode this
apply-then-reconcile pattern directly — prefer them for new code.

### Computed Queries

```typescript
const filteredItems = Atom.make((get) => {
  const items = get(itemsAtom)
  const searchTerm = get(searchAtom)
  const activeFilters = get(filtersAtom)

  return items.filter(
    (item) =>
      item.name.includes(searchTerm) &&
      activeFilters.every((f) => f.predicate(item)),
  )
})
```

`Atom.debounce` is useful for the search term atom itself, so the derived query
doesn't recompute on every keystroke.

## Batching Updates

```typescript
const openModal = (type: ModalType, metadata?: Record<string, unknown>) => {
  Atom.batch(() => {
    Atom.update(modalAtomFamily(type), (state) => ({
      ...state,
      isOpen: true,
      metadata,
    }))
  })
}
```

## localStorage Persistence

```typescript
import { BrowserKeyValueStore } from "@effect/platform-browser"
import { Atom } from "effect/unstable/reactivity"
import { Schema } from "effect"

const localStorageRuntime = Atom.runtime(BrowserKeyValueStore.layerLocalStorage)

// Note Schema.Literals (array) in v4
const themeAtom = Atom.kvs({
  runtime: localStorageRuntime,
  key: "app-theme",
  schema: Schema.Literals(["dark", "light", "system"]),
  defaultValue: () => "system" as const,
})
```

`Atom.searchParam` does the same thing against the URL query string.

## Anti-Patterns

### FORBIDDEN: Creating Atoms Inside Components

```typescript
// ❌ WRONG - creates a new atom on every render
function Counter() {
    const countAtom = Atom.make(0)
    const count = useAtomValue(countAtom)
    return <div>{count}</div>
}

// ✅ CORRECT - define atoms outside components
const countAtom = Atom.make(0)

function Counter() {
    const count = useAtomValue(countAtom)
    return <div>{count}</div>
}
```

### FORBIDDEN: Imperative Updates from React Components

```typescript
// ❌ WRONG - bypasses the hook, so React doesn't re-render
export const openModal = (type: string) => {
    Atom.batch(() => {
        Atom.update(modalAtomFamily(type), (s) => ({ ...s, isOpen: true }))
    })
}

// ✅ CORRECT - expose a hook
export const useModal = (type: string) => {
    const state = useAtomValue(modalAtomFamily(type))
    const setState = useAtomSet(modalAtomFamily(type))

    const open = useCallback(() => setState((p) => ({ ...p, isOpen: true })), [setState])
    const close = useCallback(() => setState((p) => ({ ...p, isOpen: false })), [setState])

    return { isOpen: state.isOpen, open, close }
}
```

**When imperative updates ARE acceptable:** event listeners outside React
(keyboard shortcuts), effects reacting to atom changes, non-UI state (analytics,
logging).

### FORBIDDEN: Missing Finalizers

```typescript
// ❌ WRONG - memory leak
const scrollAtom = Atom.make((get) => {
  const onScroll = () => get.setSelf(window.scrollY)
  window.addEventListener("scroll", onScroll)
  return window.scrollY
})

// ✅ CORRECT
const scrollAtom = Atom.make((get) => {
  const onScroll = () => get.setSelf(window.scrollY)
  window.addEventListener("scroll", onScroll)
  get.addFinalizer(() => window.removeEventListener("scroll", onScroll))
  return window.scrollY
})
```

### FORBIDDEN: Missing keepAlive for Global State

```typescript
// ❌ WRONG - state resets when the last subscriber unmounts
export const modalStateAtom = Atom.make({ isOpen: false })

// ✅ CORRECT
export const modalStateAtom = Atom.make({ isOpen: false }).pipe(Atom.keepAlive)
```

### FORBIDDEN: Ignoring AsyncResult States

```typescript
// ❌ WRONG - doesn't handle loading/error
const userResult = useAtomValue(userAtom)
return <div>Hello, {userResult.name}</div> // Type error

// ✅ CORRECT
return AsyncResult.builder(userResult)
    .onInitial(() => <div>Loading...</div>)
    .onError((error) => <div>Error: {error.message}</div>)
    .onSuccess((user) => <div>Hello, {user.name}</div>)
    .render()
```

### FORBIDDEN: Updating State During Render

```typescript
// ❌ WRONG - side effect during render
function Component() {
    const count = useAtomValue(countAtom)
    Atom.set(countAtom, count + 1)
    return <div>{count}</div>
}

// ✅ CORRECT - use effects or event handlers
function Component() {
    const count = useAtomValue(countAtom)
    const setCount = useAtomSet(countAtom)

    useEffect(() => { setCount((c) => c + 1) }, [])

    return <div>{count}</div>
}
```

### FORBIDDEN: Confusing `AsyncResult` with `Result`

In v4, `Result` (from `effect`) is the replacement for `Either` — a synchronous
success-or-failure value. `AsyncResult` (from `effect/unstable/reactivity`) is
the atom lifecycle type with initial/waiting states. They are different types
with overlapping method names. Import them explicitly and never alias either to
`Result` in atom code.

## Performance Tips

### Selective Re-rendering

```typescript
// ❌ WRONG - subscribes to the entire state object
const state = useAtomValue(appStateAtom)
const userName = state.user.name

// ✅ CORRECT - derive a focused atom
const userNameAtom = Atom.map(appStateAtom, (state) => state.user.name)
const userName = useAtomValue(userNameAtom)
```

`Atom.mapResult` is the equivalent for `AsyncResult`-bearing atoms — it maps the
success value without collapsing the loading/error states.

### When to Use keepAlive

Use `Atom.keepAlive` for:

- Global application state
- Modal/dialog state
- User preferences
- Authentication state
- Frequently accessed derived state

Skip `keepAlive` for:

- Component-local state that should reset
- Temporary form state
- State tied to component lifecycle

For the middle ground — expensive but not permanent — `Atom.setIdleTTL(atom,
"30 seconds")` keeps it warm briefly after the last subscriber leaves.
