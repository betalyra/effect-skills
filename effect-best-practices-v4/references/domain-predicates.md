# Domain Predicates (Effect v4)

Generate complete sets of predicates, `Equivalence`s, and `Order`s for domain
types, derived from typeclass implementations rather than hand-written
comparisons.

## Table of Contents

- [What Changed From v3](#what-changed-from-v3)
- [Equality Is Structural by Default](#equality-is-structural-by-default)
- [Equivalence from Schema](#equivalence-from-schema)
- [Field-Based Equivalence](#field-based-equivalence)
- [Combining Equivalences](#combining-equivalences)
- [Order Instances](#order-instances)
- [Combining Orders](#combining-orders)
- [Usage Examples](#usage-examples)
- [Checklist for Complete Coverage](#checklist-for-complete-coverage)
- [Key Patterns Summary](#key-patterns-summary)

## What Changed From v3

| v3                          | v4                                          |
| --------------------------- | ------------------------------------------- |
| `Schema.Data(schema)`       | **Removed** — `Equal.equals` is deep by default |
| `Schema.equivalence(s)`     | `Schema.toEquivalence(s)`                   |
| `Equal.equivalence()`       | `Equal.asEquivalence()`                     |
| `Equivalence.string`        | `Equivalence.String`                        |
| `Equivalence.number`        | `Equivalence.Number`                        |
| `Order.string`              | `Order.String`                              |
| `Order.number`              | `Order.Number`                              |
| `combine(a, b, c)` variadic | `combine` is **binary**; use `combineAll([…])` |
| `Schema.arbitrary`          | `Schema.toArbitrary`                        |
| `Schema.pretty`             | `Schema.toFormatter`                        |

The lowercase → uppercase rename of the primitive instances is the one that
bites most often, because the old names simply don't exist and the error points
at the import, not the concept.

## Equality Is Structural by Default

`Schema.Data` is gone, and you don't need it: in v4 `Equal.equals` compares
plain objects, arrays, `Map`s, `Set`s, `Date`s, and `RegExp`s **by value**.

```typescript
import { Equal, Schema } from "effect"

export const Task = Schema.TaggedStruct("pending", {
  id: Schema.String,
  createdAt: Schema.DateTimeUtc,
})

const task1 = Task.make({ id: "123", createdAt: now })
const task2 = Task.make({ id: "123", createdAt: now })

Equal.equals(task1, task2) // true — structural, no opt-in required
```

Two other behaviour changes worth knowing:

- `Equal.equals(NaN, NaN)` is now `true` (it was `false` in v3).
- If you genuinely need reference identity, opt out explicitly with
  `Equal.byReference(obj)` (proxy-based, leaves the original alone) or
  `Equal.byReferenceUnsafe(obj)` (marks the object itself — faster, but
  permanent).

Types implementing the `Equal` interface still use their own logic, as before.

## Equivalence from Schema

When you need an `Equivalence` **value** to hand to a combinator, derive it from
the schema:

```typescript
import { Array, Schema } from "effect"

export const TaskEquivalence = Schema.toEquivalence(Task)

// Usage with combinators
const uniqueTasks = Array.dedupeWith(tasks, TaskEquivalence)
```

`Equal.asEquivalence<A>()` gives you the same thing derived from `Equal.equals`
rather than from a schema — useful for types that already implement `Equal`.

## Field-Based Equivalence

Compare by specific fields using `Equivalence.mapInput`:

```typescript
import { DateTime, Equivalence } from "effect"

/**
 * Compare tasks by ID only.
 *
 * @category Equivalence
 * @example
 * const uniqueById = Array.dedupeWith(tasks, Task.EquivalenceById)
 */
export const EquivalenceById = Equivalence.mapInput(
  Equivalence.String, // capitalised in v4
  (task: Task) => task.id,
)

/**
 * Compare by status tag.
 *
 * @category Equivalence
 */
export const EquivalenceByTag = Equivalence.mapInput(
  Equivalence.String,
  (task: Task) => task._tag,
)

/**
 * Compare by creation date.
 *
 * @category Equivalence
 */
export const EquivalenceByCreatedAt = Equivalence.mapInput(
  DateTime.Equivalence,
  (task: Task) => task.createdAt,
)
```

**Key pattern: `Equivalence.mapInput`**

- Signature: `Equivalence.mapInput(baseEquivalence, (value) => extractField)`
- Compose from simpler equivalences
- Maps the domain type onto a comparable value
- Dual API: data-first and data-last

Available primitives: `Equivalence.String`, `Number`, `Boolean`, `BigInt`,
`Date`, plus the structural combinators `Equivalence.Struct`,
`Equivalence.Elements`, and `Equivalence.Record`.

## Combining Equivalences

`Equivalence.combine` is **binary** in v4. For more than two, use
`combineAll`:

```typescript
import { Equivalence } from "effect"

/**
 * Compare by tag first, then by ID. Both must match.
 *
 * @category Equivalence
 */
export const EquivalenceByTagAndId = Equivalence.combine(
  EquivalenceByTag,
  EquivalenceById,
)

/**
 * Compare by every criterion — use combineAll for three or more.
 *
 * @category Equivalence
 */
export const EquivalenceComplete = Equivalence.combineAll([
  EquivalenceByTag,
  EquivalenceById,
  EquivalenceByCreatedAt,
])
```

**Key pattern: `Equivalence.combine` / `combineAll`**

- All must match (AND logic)
- Order doesn't affect the result (unlike `Order.combine`)
- `combineAll` takes an `Iterable<Equivalence<A>>`

For struct-shaped comparison, `Equivalence.Struct({ id: Equivalence.String, … })`
is often cleaner than chaining `mapInput` + `combine`.

## Order Instances

Compose orders from simpler base orders using `Order.mapInput`:

```typescript
import { DateTime, Order } from "effect"

/**
 * Order by ID.
 *
 * @category Orders
 * @example
 * const sorted = Array.sort(tasks, Task.OrderById)
 */
export const OrderById: Order.Order<Task> = Order.mapInput(
  Order.String, // capitalised in v4
  (task: Task) => task.id,
)

/**
 * Order by creation date.
 *
 * @category Orders
 */
export const OrderByCreatedAt: Order.Order<Task> = Order.mapInput(
  DateTime.Order,
  (task: Task) => task.createdAt,
)

/**
 * Order by status tag.
 *
 * @category Orders
 */
export const OrderByTag: Order.Order<Task> = Order.mapInput(
  Order.String,
  (task: Task) => task._tag,
)

/**
 * Order by priority (domain-specific logic).
 *
 * @category Orders
 */
export const OrderByPriority: Order.Order<Task> = Order.mapInput(
  Order.Number,
  (task: Task) => {
    const priorities = { pending: 0, active: 1, completed: 2 } as const
    return priorities[task._tag]
  },
)
```

**Key pattern: `Order.mapInput`**

- Signature: `Order.mapInput(baseOrder, (value) => extractField)`
- Compose from existing orders (`Order.String`, `Order.Number`, `Order.Date`,
  `DateTime.Order`, …)
- Dual API: data-first and data-last

`Order.flip` reverses an order — prefer it to writing a mirrored comparator:

```typescript
export const OrderByNewestFirst = Order.flip(OrderByCreatedAt)
```

## Combining Orders

Also **binary** in v4, with `combineAll` for longer chains:

```typescript
import { Order } from "effect"

/**
 * Sort by priority first, then by creation date.
 *
 * @category Orders
 */
export const OrderByPriorityThenDate: Order.Order<Task> = Order.combine(
  OrderByPriority,
  OrderByCreatedAt,
)

/**
 * Sort by tag, then ID, then creation date.
 *
 * @category Orders
 */
export const OrderComplex: Order.Order<Task> = Order.combineAll([
  OrderByTag,
  OrderById,
  OrderByCreatedAt,
])
```

**Key pattern: `Order.combine` / `combineAll`**

- The first order takes precedence; ties fall through to the next
- Order **does** matter (unlike `Equivalence.combine`)
- Usable directly with `Array.sort`

`Order` also carries useful derived predicates: `Order.isLessThan(order)`,
`isGreaterThan`, `isBetween`, `min`, `max`, `clamp` — reach for these instead of
comparing with `<` on extracted fields.

## Usage Examples

### Equality Examples

```typescript
import { Array, Equal } from "effect"
import * as Task from "@/schemas/Task"

// Structural equality — no Schema.Data needed in v4
const areSame = Equal.equals(task1, task2)

// Deduplicate by ID only
const uniqueById = Array.dedupeWith(tasks, Task.EquivalenceById)

// Deduplicate by tag and ID
const uniqueByTagAndId = Array.dedupeWith(tasks, Task.EquivalenceByTagAndId)

// Membership by a custom equivalence
const hasTask = Array.containsWith(Task.EquivalenceById)(tasks, searchTask)
```

### Sorting Examples

```typescript
import { Array, Order, pipe } from "effect"

// Single field
const sortedById = Array.sort(tasks, Task.OrderById)

// Multi-criteria
const sortedComplex = Array.sort(
  tasks,
  Order.combine(Task.OrderByPriority, Task.OrderByCreatedAt),
)

// Filter then sort
const sortedFiltered = pipe(
  tasks,
  Array.filter(Task.isPending),
  Array.sort(Task.OrderByCreatedAt),
)
```

### Complex Filtering

```typescript
import { Array, Duration, Order, pipe } from "effect"

// Find long appointments this week
const longThisWeek = pipe(
  appointments,
  Array.filter(Appointment.isScheduledThisWeek),
  Array.filter(Appointment.hasMinimumDuration(Duration.hours(2))),
)

// Deduplicate and sort
const uniqueSorted = pipe(
  appointments,
  Array.dedupeWith(Appointment.EquivalenceById),
  Array.sort(Appointment.OrderByPriorityThenDate),
)
```

### Predicates from the Predicate module

**Never hand-roll `isString`, `isRecord`, `isNotNull` and friends** — the
`Predicate` module has them, and they compose with `Predicate.and`,
`Predicate.or`, `Predicate.not`, and `Predicate.compose`:

```typescript
import { Predicate } from "effect"

const thing: unknown = { a: 1 }

if (Predicate.isObject(thing) && Predicate.isNumber(thing.a)) {
  // narrowed
}

const isActiveAndRecent = Predicate.and(Task.isActive, Task.isCreatedThisWeek)
```

For domain-level narrowing inside schemas, use `Schema.refine` (type
refinement) or `Schema.check(Schema.makeFilter(...))` (validation without
narrowing) rather than a bare predicate at the boundary — that way the check
runs on decode and produces a proper issue.

## Checklist for Complete Coverage

### Equality

- [ ] Rely on structural `Equal.equals()` — do **not** reach for `Schema.Data`
      (it no longer exists)
- [ ] Export `Schema.toEquivalence()` when a combinator needs an `Equivalence`
- [ ] Export field-based equivalences using `Equivalence.mapInput`
- [ ] Export combined equivalences using `combine` / `combineAll`
- [ ] Use `Equal.byReference` only where identity semantics are genuinely wanted

### Orders

- [ ] Export orders for all sortable fields using `Order.mapInput`
- [ ] Export combined orders using `combine` / `combineAll`
- [ ] Document which field takes precedence in combined orders
- [ ] Export reversed variants with `Order.flip` rather than duplicating logic

### Schedulable types

- [ ] `isScheduledBefore`, `isScheduledAfter`, `isScheduledBetween`
- [ ] `isScheduledOn`, `isScheduledToday`, `isScheduledThisWeek`,
      `isScheduledThisMonth`
- [ ] All Order instances
- [ ] Implement these against `DateTime` and the `Clock`, never `Date.now()`

### Durable types

- [ ] `hasMinimumDuration` (isMoreThan)
- [ ] `hasMaximumDuration` (isLessThan)
- [ ] `hasDurationBetween` (isBetween)
- [ ] `hasExactDuration`
- [ ] All Order instances

### Domain-specific fields

- [ ] A predicate for each variant (`isPending`, `isActive`, …)
- [ ] Order by field value using `Order.mapInput`
- [ ] Order by priority/importance if applicable
- [ ] Combined orders for common sorting patterns

## Key Patterns Summary

**1. Structural equality is free**

```typescript
Equal.equals(t1, t2) // deep by default in v4
```

**2. `Schema.toEquivalence()` for combinators**

```typescript
export const TaskEquivalence = Schema.toEquivalence(Task)
// Usage: Array.dedupeWith(tasks, TaskEquivalence)
```

**3. `Equivalence.mapInput` for field-based**

```typescript
Equivalence.mapInput(Equivalence.String, (t: Task) => t.id)
```

**4. `Equivalence.combine` / `combineAll` for multi-field**

```typescript
Equivalence.combine(EquivalenceByTag, EquivalenceById)
Equivalence.combineAll([EquivalenceByTag, EquivalenceById, EquivalenceByCreatedAt])
```

**5. `Order.mapInput` for field-based sorting**

```typescript
Order.mapInput(Order.String, (t: Task) => t.id)
```

**6. `Order.combine` / `combineAll` for multi-criteria sorting**

```typescript
Order.combine(OrderByPriority, OrderByCreatedAt)
```
