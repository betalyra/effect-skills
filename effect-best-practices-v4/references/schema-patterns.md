# Schema Patterns (Effect v4)

Schema was substantially rewritten in v4. The mental model is the same —
a schema describes a bidirectional codec between an `Encoded` and a `Type` —
but the API surface moved: **filters became `check`**, **variadic constructors
became arrays**, and **type extraction moved onto the schema itself**.

## Table of Contents

- [The v4 API shifts you must internalise](#the-v4-api-shifts-you-must-internalise)
- [Branded Types for IDs](#branded-types-for-ids)
- [Schema.Struct for Domain Types](#schemastruct-for-domain-types)
- [Checks (formerly filters)](#checks-formerly-filters)
- [Transformations](#transformations)
- [Schema.Class for Entities with Methods](#schemaclass-for-entities-with-methods)
- [Annotations](#annotations)
- [Optional Fields](#optional-fields)
- [Union Types and Discriminated Unions](#union-types-and-discriminated-unions)
- [Enums and Literals](#enums-and-literals)
- [Recursive Schemas](#recursive-schemas)
- [Decoding and Encoding](#decoding-and-encoding)
- [JSON Encoding & Decoding](#json-encoding--decoding)
- [Formatting Errors](#formatting-errors)
- [Struct Manipulation](#struct-manipulation)

## The v4 API shifts you must internalise

| Concern             | v3                              | v4                                       |
| ------------------- | ------------------------------- | ---------------------------------------- |
| Type extraction     | `Schema.Schema.Type<typeof S>`  | `typeof S.Type`                          |
| Encoded extraction  | `Schema.Schema.Encoded<typeof S>` | `typeof S.Encoded`                     |
| Validation          | `Schema.filter(pred)`           | `.check(Schema.makeFilter(pred))`        |
| Named filters       | `minLength(1)`                  | `.check(Schema.isMinLength(1))`          |
| Refinements         | `filter(refinement)`            | `.pipe(Schema.refine(refinement))`       |
| Unions              | `Schema.Union(A, B)`            | `Schema.Union([A, B])`                   |
| Tuples              | `Schema.Tuple(A, B)`            | `Schema.Tuple([A, B])`                   |
| Multiple literals   | `Schema.Literal("a", "b")`      | `Schema.Literals(["a", "b"])`            |
| Records             | `Schema.Record({key, value})`   | `Schema.Record(key, value)`              |
| Annotations         | `.annotations({...})`           | `.annotate({...})`                       |
| Composition         | `Schema.compose(B)`             | `Schema.decodeTo(B)`                     |
| Transform           | `Schema.transform(from,to,…)`   | `from.pipe(Schema.decodeTo(to, SchemaTransformation.transform({…})))` |
| Decode (Effect)     | `Schema.decodeUnknown`          | `Schema.decodeUnknownEffect`             |
| Decode (Either)     | `Schema.decodeUnknownEither`    | `Schema.decodeUnknownExit`               |
| Encode (Effect)     | `Schema.encode`                 | `Schema.encodeEffect`                    |
| Validate            | `Schema.validate*`              | removed — `decode*` on `Schema.toType(s)`|
| Enums               | `Schema.Enums(obj)`             | `Schema.Enum(obj)`                       |
| Tagged errors       | `Schema.TaggedError`            | `Schema.TaggedErrorClass`                |
| UUID / ULID         | `Schema.UUID`                   | `Schema.String.check(Schema.isUUID())`   |
| JSON parsing        | `Schema.parseJson(S)`           | `Schema.fromJsonString(S)`               |
| `Schema.Data(s)`    | —                               | removed (`Equal.equals` is deep now)     |

Filters are all `is`-prefixed: `isMinLength`, `isMaxLength`, `isLengthBetween`,
`isGreaterThan`, `isGreaterThanOrEqualTo`, `isLessThan`, `isBetween`, `isInt`,
`isMultipleOf`, `isFinite`, `isNonEmpty`, `isPattern`, `isUUID`, `isULID`.
`positive`, `negative`, `nonNegative`, and `nonPositive` were removed — write
`isGreaterThan(0)` and friends.

`*FromSelf` schemas dropped their suffix (`DateFromSelf` → `Date`,
`OptionFromSelf` → `Option`, `ChunkFromSelf` → `Chunk`, …). Watch out for
`Redacted`: v4's `Schema.Redacted` is the old `RedactedFromSelf`, and the old
`Schema.Redacted` behaviour is now `Schema.RedactedFromValue`.

## Branded Types for IDs

**Always brand entity IDs** to prevent accidentally passing the wrong ID type:

```typescript
import { Schema } from "effect"

// Entity IDs — always branded with a namespace.
// Note: Schema.UUID is gone; validate with a check first, then brand.
export const UserId = Schema.String.check(Schema.isUUID()).pipe(
  Schema.brand("@App/UserId"),
)
export type UserId = typeof UserId.Type

export const OrganizationId = Schema.String.check(Schema.isUUID()).pipe(
  Schema.brand("@App/OrganizationId"),
)
export type OrganizationId = typeof OrganizationId.Type

export const OrderId = Schema.String.check(Schema.isUUID()).pipe(
  Schema.brand("@App/OrderId"),
)
export type OrderId = typeof OrderId.Type
```

`Schema.isUUID()` optionally takes a version: `Schema.isUUID(4)`.

### Branding Convention

Use `@Namespace/EntityName` format:

- `@App/UserId` — main application entities
- `@Billing/InvoiceId` — billing domain entities
- `@External/StripeCustomerId` — external system IDs

### Creating Branded Values

```typescript
// From an untrusted string (validates the UUID format)
const userId = Schema.decodeSync(UserId)("123e4567-e89b-12d3-a456-426614174000")

// From a value you already trust — `make` runs the checks and returns the type
const newUserId = UserId.make(crypto.randomUUID())

// Type error — can't mix ID types
const order = yield* orderService.findById(userId) // Error: UserId is not OrderId
```

`brand` only affects the TypeScript type — it adds no runtime check of its own.
That's why the `check(isUUID())` comes first. If you need the checks *and* the
brand from an existing `Brand.Constructor`, use `Schema.fromBrand`.

### When NOT to Brand

```typescript
// NOT branded — acceptable
export const Url = Schema.String
export const FilePath = Schema.String

// These don't need branding because:
// 1. They don't cross service boundaries in ways that could be confused
// 2. They're validated by format, not by identity
```

## Schema.Struct for Domain Types

**Prefer `Schema.Struct`** over TypeScript interfaces for domain types:

```typescript
export const User = Schema.Struct({
  id: UserId,
  email: Schema.String,
  name: Schema.String,
  organizationId: OrganizationId,
  role: Schema.Literals(["admin", "member", "viewer"]),
  createdAt: Schema.DateTimeUtc,
  updatedAt: Schema.DateTimeUtc,
})
export type User = typeof User.Type

// Encoded type for database/API boundaries
export type UserEncoded = typeof User.Encoded
```

### Input Types for Mutations

```typescript
import { Effect, Schema } from "effect"

export const CreateUserInput = Schema.Struct({
  email: Schema.String.check(
    Schema.isPattern(/^[^\s@]+@[^\s@]+\.[^\s@]+$/),
  ).annotate({ description: "Valid email address" }),

  name: Schema.String.check(Schema.isMinLength(1), Schema.isMaxLength(100)),

  organizationId: OrganizationId,

  // v3's optionalWith({ default }) → withDecodingDefaultType
  role: Schema.Literals(["admin", "member", "viewer"]).pipe(
    Schema.withDecodingDefaultType(Effect.succeed("member" as const)),
  ),
})
export type CreateUserInput = typeof CreateUserInput.Type

export const UpdateUserInput = Schema.Struct({
  name: Schema.optional(Schema.String.check(Schema.isMinLength(1))),
  role: Schema.optional(Schema.Literals(["admin", "member", "viewer"])),
})
export type UpdateUserInput = typeof UpdateUserInput.Type
```

## Checks (formerly filters)

`check` accepts one or more filters and composes them:

```typescript
const Password = Schema.String.check(
  Schema.isMinLength(12),
  Schema.isPattern(/[0-9]/),
)
```

For ad-hoc predicates use `Schema.makeFilter`:

```typescript
const EvenNumber = Schema.Number.check(
  Schema.makeFilter((n) => n % 2 === 0 || "must be even"),
)
```

A `makeFilter` predicate can return several shapes, which is what makes v4's
validation messages good:

| Return value                      | Meaning                                |
| --------------------------------- | -------------------------------------- |
| `undefined` / `true`              | success                                |
| `false`                           | generic failure                        |
| `string`                          | failure with that message              |
| `SchemaIssue.Issue`               | a fully-formed issue                   |
| `{ path, issue }`                 | failure at a nested path               |
| `ReadonlyArray<FilterIssue>`      | several failures reported together     |

Cross-field validation with a nested path:

```typescript
const Signup = Schema.Struct({
  password: Schema.String,
  confirmPassword: Schema.String,
}).check(
  Schema.makeFilter((o) =>
    o.password === o.confirmPassword
      ? undefined
      : { path: ["password"], issue: "passwords must match" },
  ),
)
```

For type refinements (narrowing the output type), use `Schema.refine`:

```typescript
const SomeOption = Schema.Option(Schema.String).pipe(Schema.refine(Option.isSome))
```

For async validation, `Schema.decode` with `SchemaGetter.checkEffect`:

```typescript
import { Effect, Schema, SchemaGetter } from "effect"

const AvailableUsername = Schema.String.pipe(
  Schema.decode({
    decode: SchemaGetter.checkEffect((username) =>
      checkAvailable(username).pipe(
        Effect.map((ok) => ok || "username already taken"),
      ),
    ),
    encode: SchemaGetter.passthrough(),
  }),
)
```

## Transformations

`Schema.transform` is gone; compose with `decodeTo` plus a
`SchemaTransformation`:

```typescript
import { Schema, SchemaTransformation } from "effect"

// Comma-separated string ⇄ array
export const CommaSeparatedList = Schema.String.pipe(
  Schema.decodeTo(
    Schema.Array(Schema.String),
    SchemaTransformation.transform({
      decode: (s) => s.split(",").map((x) => x.trim()).filter(Boolean),
      encode: (arr) => arr.join(","),
    }),
  ),
)

// Cents ⇄ dollars
export const DollarsFromCents = Schema.Int.pipe(
  Schema.decodeTo(
    Schema.Number,
    SchemaTransformation.transform({
      decode: (cents) => cents / 100,
      encode: (dollars) => Math.round(dollars * 100),
    }),
  ),
)
```

Fallible transformations use `SchemaGetter.transformOrFail`, returning an
`Effect` that fails with a `SchemaIssue`:

```typescript
import { Effect, Number, Option, Schema, SchemaGetter, SchemaIssue } from "effect"

export const NumberFromString = Schema.String.pipe(
  Schema.decodeTo(Schema.Number, {
    decode: SchemaGetter.transformOrFail((s) => {
      const n = Number.parse(s)
      return n === undefined
        ? Effect.fail(new SchemaIssue.InvalidValue(Option.some(s)))
        : Effect.succeed(n)
    }),
    encode: SchemaGetter.String(),
  }),
)
```

Literal transformations are now methods:

```typescript
const a = Schema.Literal(0).transform("a")
const b = Schema.Literals([0, 1, 2]).transform(["a", "b", "c"])
```

## Schema.Class for Entities with Methods

Use `Schema.Class` when entities need behaviour. The identifier should be
namespaced, like service identifiers:

```typescript
export class User extends Schema.Class<User>("myapp/domain/User")({
  id: UserId,
  email: Schema.String,
  name: Schema.String,
  role: Schema.Literals(["admin", "member", "viewer"]),
  createdAt: Schema.DateTimeUtc,
}) {
  get isAdmin(): boolean {
    return this.role === "admin"
  }

  get displayName(): string {
    return this.name || this.email.split("@")[0]
  }

  canAccessResource(resource: Resource): boolean {
    if (this.isAdmin) return true
    return resource.ownerId === this.id
  }
}

// The class type IS the decoded type
export type UserType = typeof User.Type // === User
export type UserEncoded = typeof User.Encoded

const user = new User({
  id: UserId.make(crypto.randomUUID()),
  email: "alice@example.com",
  name: "Alice",
  role: "member",
  createdAt: DateTime.nowUnsafe(),
})
```

## Annotations

`annotations(...)` was renamed to `annotate(...)` and is a method on the schema:

```typescript
export const CreateOrderInput = Schema.Struct({
  productId: ProductId.annotate({ description: "The product to order" }),

  quantity: Schema.Int.check(Schema.isGreaterThan(0)).annotate({
    description: "Number of items to order",
    examples: [1, 5, 10],
  }),

  shippingAddress: Schema.Struct({
    line1: Schema.String.annotate({ description: "Street address" }),
    line2: Schema.optional(Schema.String),
    city: Schema.String,
    state: Schema.String.check(Schema.isLengthBetween(2, 2)),
    zip: Schema.String.check(Schema.isPattern(/^\d{5}(-\d{4})?$/)),
  }).annotate({ description: "Shipping destination" }),
}).annotate({
  title: "Create Order Input",
  description: "Input for creating a new order",
})
```

Annotations also carry integration metadata — `httpApiStatus` for HttpApi error
schemas, and whatever JSON Schema generation needs.

## Optional Fields

v4 distinguishes two flavours precisely:

| Helper                  | Meaning                                          |
| ----------------------- | ------------------------------------------------ |
| `Schema.optionalKey(s)` | the key may be absent (exact optional)           |
| `Schema.optional(s)`    | the key may be absent **or** `undefined`         |
| `Schema.NullOr(s)`      | the value may be `null`                          |

```typescript
import { Effect, Schema } from "effect"

export const UserPreferences = Schema.Struct({
  // Absent or undefined
  theme: Schema.optional(Schema.Literals(["light", "dark"])),

  // Absent only — never explicitly undefined (v3's { exact: true })
  timezone: Schema.optionalKey(Schema.String),

  // With a default applied during decoding
  language: Schema.String.pipe(
    Schema.withDecodingDefaultType(Effect.succeed("en")),
  ),

  // Nullable, e.g. for database compatibility
  bio: Schema.NullOr(Schema.String),
})
```

`withDecodingDefaultTypeKey` is the exact-optional variant of
`withDecodingDefaultType`.

For the more involved `optionalWith` cases (nullable + default, or transforming
between optionalities), use `decodeTo` with `SchemaGetter.transformOptional`:

```typescript
import { Option, Predicate, Schema, SchemaGetter } from "effect"

// v3: optionalWith(NumberFromString, { nullable: true, exact: true, default: () => -1 })
const schema = Schema.Struct({
  a: Schema.optionalKey(Schema.NullOr(Schema.NumberFromString)).pipe(
    Schema.decodeTo(Schema.Number, {
      decode: SchemaGetter.transformOptional((o) =>
        o.pipe(
          Option.filter(Predicate.isNotNull),
          Option.orElseSome(() => -1),
        ),
      ),
      encode: SchemaGetter.required(),
    }),
  ),
})
```

## Union Types and Discriminated Unions

Unions take an **array** in v4:

```typescript
// Simple union of literals — use Literals, not Union of Literal
export const PaymentMethod = Schema.Literals(["card", "bank_transfer", "crypto"])

// Discriminated union (tagged)
export const PaymentDetails = Schema.Union([
  Schema.TaggedStruct("Card", {
    cardNumber: Schema.String,
    expiry: Schema.String,
    cvv: Schema.String,
  }),
  Schema.TaggedStruct("BankTransfer", {
    accountNumber: Schema.String,
    routingNumber: Schema.String,
  }),
  Schema.TaggedStruct("Crypto", {
    walletAddress: Schema.String,
    network: Schema.Literals(["ethereum", "bitcoin", "solana"]),
  }),
])
export type PaymentDetails = typeof PaymentDetails.Type

// Usage with Match.valueTags
const processPayment = (details: PaymentDetails) =>
  Match.valueTags(details, {
    Card: ({ cardNumber, expiry, cvv }) => processCard(cardNumber, expiry, cvv),
    BankTransfer: ({ accountNumber, routingNumber }) =>
      processBankTransfer(accountNumber, routingNumber),
    Crypto: ({ walletAddress, network }) => processCrypto(walletAddress, network),
  })
```

To attach a discriminator to existing structs (v3's
`attachPropertySignature`), use `mapFields` with `Schema.tagDefaultOmit`:

```typescript
const Shape = Schema.Union([
  Circle.mapFields((f) => ({ ...f, kind: Schema.tagDefaultOmit("circle") })),
  Square.mapFields((f) => ({ ...f, kind: Schema.tagDefaultOmit("square") })),
])
```

## Enums and Literals

```typescript
// Literals for small, fixed sets — note the array and the plural name
export const UserRole = Schema.Literals(["admin", "member", "viewer"])
export type UserRole = typeof UserRole.Type

// Schema.Enum (singular in v4) wraps a TypeScript enum object
enum OrderStatusEnum {
  Pending = "pending",
  Processing = "processing",
  Shipped = "shipped",
  Delivered = "delivered",
  Cancelled = "cancelled",
}
export const OrderStatus = Schema.Enum(OrderStatusEnum)
export type OrderStatus = typeof OrderStatus.Type
```

To narrow an existing literal set, use `.pick`:

```typescript
const Active = Schema.Literals(["pending", "shipped", "cancelled"]).pick([
  "pending",
  "shipped",
])
```

## Recursive Schemas

```typescript
interface Category {
  readonly id: string
  readonly name: string
  readonly children: ReadonlyArray<Category>
}

export const Category = Schema.Struct({
  id: Schema.String,
  name: Schema.String,
  children: Schema.Array(
    Schema.suspend((): Schema.Codec<Category> => Category),
  ),
})
```

## Decoding and Encoding

The naming is now explicit about the return type:

| Function                    | Returns              |
| --------------------------- | -------------------- |
| `decodeUnknownEffect(s)`    | `Effect<A, SchemaError>` |
| `decodeEffect(s)`           | `Effect<A, SchemaError>` |
| `decodeUnknownSync(s)`      | `A` (throws)         |
| `decodeUnknownExit(s)`      | `Exit<A, SchemaError>` |
| `encodeEffect(s)`           | `Effect<I, SchemaError>` |
| `encodeUnknownExit(s)`      | `Exit<I, SchemaError>` |

```typescript
// Build the parser once, at module scope — don't rebuild per request
const decodeUser = Schema.decodeUnknownEffect(User)
const encodeUser = Schema.encodeEffect(User)

const parseUser = Effect.fn("parseUser")(function* (input: unknown) {
  return yield* decodeUser(input).pipe(
    Effect.mapError((error) => new InvalidUserPayload({ message: error.message })),
  )
})

// Sync — only in controlled contexts (config, tests, trusted literals)
const user = Schema.decodeUnknownSync(User)(rawData)
```

`Schema.validate*` was removed. When you want to validate an already-decoded
value, decode against the type side:

```typescript
const validate = Schema.decodeSync(Schema.toType(User))
```

`Schema.asserts` now takes the input directly:

```typescript
Schema.asserts(Schema.String, input) // v3: Schema.asserts(Schema.String)(input)
```

## JSON Encoding & Decoding

`Schema.parseJson(S)` became `Schema.fromJsonString(S)`; the no-argument
`parseJson()` became `Schema.UnknownFromJsonString`.

```typescript
import { Effect, Schema } from "effect"

const Row = Schema.Literals(["A", "B", "C", "D", "E", "F", "G", "H"])
const Column = Schema.Literals(["1", "2", "3", "4", "5", "6", "7", "8"])

class Position extends Schema.Class<Position>("chess/Position")({
  row: Row,
  column: Column,
}) {}

class Move extends Schema.Class<Move>("chess/Move")({
  from: Position,
  to: Position,
}) {}

// A schema from JSON string → Move
const MoveFromJson = Schema.fromJsonString(Move)

const program = Effect.gen(function* () {
  const jsonString =
    '{"from":{"row":"A","column":"1"},"to":{"row":"B","column":"2"}}'

  // Use MoveFromJson (not Move) to decode from a JSON string
  const move = yield* Schema.decodeUnknownEffect(MoveFromJson)(jsonString)
  yield* Effect.log("Decoded move", move)

  // …and to encode back to a JSON string
  return yield* Schema.encodeEffect(MoveFromJson)(move)
})
```

## Formatting Errors

Parsing fails with `Schema.SchemaError`, which carries a nested
`SchemaIssue` in its `issue` field. `ParseResult.ArrayFormatter` is gone:

```typescript
import { Schema, SchemaIssue } from "effect"

decodeUser(input).pipe(
  Effect.tapError((error) =>
    Effect.logError("Validation failed", {
      issues: SchemaIssue.makeFormatterStandardSchemaV1()(error.issue).issues,
    }),
  ),
)
```

Each issue is `{ path: ReadonlyArray<PropertyKey>, message: string }`, which maps
cleanly onto form field errors.

## Struct Manipulation

Struct surgery moved onto `mapFields` with helpers from the `Struct` module:

```typescript
import { Schema, Struct } from "effect"

const Base = Schema.Struct({ a: Schema.String, b: Schema.Number })

const picked = Base.mapFields(Struct.pick(["a"]))         // v3: Schema.pick("a")
const omitted = Base.mapFields(Struct.omit(["b"]))        // v3: Schema.omit("b")
const partial = Base.mapFields(Struct.map(Schema.optional)) // v3: Schema.partial
const exact = Base.mapFields(Struct.map(Schema.optionalKey))
const somePartial = Base.mapFields(Struct.mapPick(["a"], Schema.optional))

// v3: Schema.extend(Schema.Struct({ c: … }))
const extended = Base.pipe(Schema.fieldsAssign({ c: Schema.Boolean }))

// Renaming keys on the encoded side (experimental)
const renamed = Base.pipe(Schema.encodeKeys({ a: "c" }))
```

For unions, the equivalent is `mapMembers` with `Tuple.map`:

```typescript
import { Schema, Tuple } from "effect"

const withKind = Schema.Union([A, B]).mapMembers(
  Tuple.map(Schema.fieldsAssign({ kind: Schema.String })),
)
```
