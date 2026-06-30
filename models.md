# Dream Models - Detailed Reference

## Base Model

Every project has an `ApplicationModel` that extends Dream:

```typescript
import Dream from '@rvoh/dream'
import { DB } from '@src/types/db.js'
import schema from '@src/types/dream.js'
import { connectionTypeConfig, globalTypeConfig } from '@src/types/dream.globals.js'

export default class ApplicationModel extends Dream {
  declare public DB: DB
  public override get schema() { return schema }
  public override get connectionTypeConfig() { return connectionTypeConfig }
  public override get globalTypeConfig() { return globalTypeConfig }
}
```

## Model Organization & Namespacing

A model's file path under `src/app/models/` becomes its class name, table name, and namespace, so where you place it is a domain-modeling decision, not a generator mechanic. (For how the `g:resource` route / model-path / `--owning-model` arguments map to files, see [generators.md](generators.md#gresource-argument-contract).)

**Namespace a model by what it *is*, not by where it is routed or what owns it.** A model namespace should describe the model's identity, not its URL or its querying parent. Do not namespace a model under its route or association parent just because the endpoint is nested: a `Booking` routed at `v1/host/places/{}/bookings --owning-model=Place` is **not** `Place/Booking`. The nested route and `--owning-model` already give you parent-scoped queries; they say nothing about the model's namespace.

Use a `Parent/Child` namespace in two cases:

- **STI subtypes** — `Room/Bedroom`, `Room/Bathroom`. The namespace expresses "is a kind of." See [sti.md](sti.md).
- **Subdomain / bounded-context modules** — `Reservations/Booking`, `Billing/Invoice`. The namespace expresses which part of the application's domain the model belongs to.

**Flat when small, grouped by subdomain when large.** A small app is commonly flat — most models sit directly under `src/app/models/`, and that is fine. As the model set grows, group models into subdomain modules rather than leaving dozens of unrelated top-level peers; a sprawling flat directory is a sign the domain hasn't been carved into bounded contexts. Do not pre-create a one-model subdomain on day one either — introduce the module when there are models to put in it. The axis to organize on is the subdomain, never the owning model or the route.

**A model that belongs to multiple parents is its own aggregate root.** When a model `belongsTo` two parents — a `Booking` belongs to both a `Place` and a `Guest` — it is usually its own organizing concept, not a sub-part of either. Namespacing it under one parent (`Place/Booking`) wrongly couples a two-parent model to that parent. Leave it top-level, or place it in its subdomain module (`Reservations/Booking`) — never under one of its parents.

Getting the namespace wrong is expensive to undo: the model file path bakes into the class name, the table name, every import, the serializer/controller paths, and the migration. A later rename touches all of those plus generated types, OpenAPI, and front-end clients. Decide the namespace deliberately at generation time.

## Column Types

Use `DreamColumn<ModelClass, 'columnName'>` for all database-backed columns. The type is inferred from the database schema via Kysely codegen.

```typescript
public id: DreamColumn<User, 'id'>
public email: DreamColumn<User, 'email'>
public createdAt: DreamColumn<User, 'createdAt'>
public updatedAt: DreamColumn<User, 'updatedAt'>
```

**Never give a column field an `=` initializer.** Declare columns as bare fields. Dream serves column reads through prototype accessors and deletes any shadowing instance property after construction, so an initializer like `public status: DreamColumn<Place, 'status'> = 'draft'` is silently discarded — no error, the default just never takes effect. Set the default at the database layer instead (`col.defaultTo(value).notNull()` — see [migrations.md — NOT NULL columns and defaults](migrations.md)); fall back to a `@deco.BeforeCreate`/`@deco.BeforeSave` hook only for a caller-dependent value the DB can't express. To transform or clamp an *input* on write (normalize null, bound a range), use a custom getter/setter on the column — see [Transforming a column on write](#transforming-a-column-on-write).

**Decimal columns return JavaScript `number`, not `string` or a `Decimal` wrapper.** Dream's type-sync converts PostgreSQL `numeric`/`decimal` columns to TS `number`, so `DreamColumn<Place, 'latitude'>` resolves to `number | null` (or `number` if NOT NULL). No `Number(value)` coercion is needed when reading.

Use the serializer `precision` option (`.attribute('latitude', { precision: 7 })`, also supported on `.delegatedAttribute`) to control how many decimal places appear in the rendered response. This handles the typical floating-point quirk where `1.1 + 2.2` becomes `3.3000000000000003` — a value that rounds cleanly to `3.30` at the precision the schema actually cares about. The OpenAPI shape and the rendered output stay consistent because precision is declared once at the serializer.

## Decorator Setup

Each model creates its own decorator instance. Generators emit the active form when the model declares any field-level `@deco.*` decorator (`@deco.BelongsTo`, `@deco.Validates`, `@deco.Encrypted`, etc.), and a commented form otherwise — with `Decorators` dropped from the merged `@rvoh/dream` import so ESLint doesn't flag it as unused:

```typescript
// Models that use @deco.* decorators (active form):
import { Decorators } from '@rvoh/dream'
const deco = new Decorators<typeof MyModel>()

// Freshly generated models with no field-level decorators (commented form):
// Uncomment when adding decorators (@deco.BelongsTo, @deco.Validates, etc.):
// import { Decorators } from '@rvoh/dream'
// const deco = new Decorators<typeof MyModel>()
```

STI children and `@SoftDelete`-decorated models follow the same rule for the field-level `Decorators` import — `STI` and `SoftDelete` are class-level decorators imported directly from `@rvoh/dream` and are unaffected by this rule.

## Associations - Full Reference

### BelongsTo

Foreign key lives on THIS model. You must also declare the FK column.

```typescript
@deco.BelongsTo('User')
public user: User
public userId: DreamColumn<Post, 'userId'>

// Optional (nullable FK).
// The `optional` flag on a BelongsTo is the canonical declaration of nullability
// for that association. It is auto-inferred by serializer `rendersOne` and
// `delegatedAttribute`, and by `extractParams` / OpenAPI request-body shape. Don't
// restate `optional: true` in serializers — that's a DRY violation, and changing
// it in the serializer is almost always a mistake.
@deco.BelongsTo('User', { optional: true })
public approver: User | null
public approverId: DreamColumn<Post, 'approverId'>

// Custom FK column on THIS model (override the conventional `host_id`)
@deco.BelongsTo('Host', { on: 'ownerId' })
public owner: Host
public ownerId: DreamColumn<Place, 'ownerId'>

// Polymorphic BelongsTo
@deco.BelongsTo(['Host', 'Place', 'Room'], { polymorphic: true, on: 'localizableId' })
public localizable: Host | Place | Room
public localizableType: DreamColumn<LocalizedText, 'localizableType'>
public localizableId: DreamColumn<LocalizedText, 'localizableId'>

// Override the primary-key side of the join (defaults to `id`); pair with `on` to
// join on a natural key — see "Joining on a non-default key" below.
@deco.BelongsTo('Host', { primaryKeyOverride: 'externalId' })
public host: Host
public hostId: DreamColumn<Place, 'hostId'>

// Skip default scopes when loading
@deco.BelongsTo('Place', { withoutDefaultScopes: ['dream:SoftDelete'] })
public place: Place
public placeId: DreamColumn<Booking, 'placeId'>
```

#### BelongsTo Options

| Option | Type | Description |
|--------|------|-------------|
| `on` | column name | Custom foreign key column name |
| `optional` | boolean | Makes the association nullable (FK can be null) |
| `polymorphic` | boolean | Enables polymorphic association (requires `on`, model arg becomes an array) |
| `primaryKeyOverride` | column name \| null | Override the primary key column used for the join (defaults to `id`) |
| `withoutDefaultScopes` | scope name[] | Default scopes to skip when loading this association |

#### A required `BelongsTo` is a two-way non-nullable contract

`optional` is the single source of truth for whether a `BelongsTo` can be null. A non-optional `BelongsTo` (the default) is typed non-null and serializes to a non-nullable OpenAPI field — "required" means "always present" — and Dream defends that contract at both ends:

- **You may not conditionally load it.** A trailing constraint on a non-optional `BelongsTo` in `preload` / `load` / `leftJoinPreload` / `leftJoinLoad` is a **compile error** — the constraint could filter the parent out and null a field the spec declares non-nullable. (`innerJoin` / `leftJoin` are exempt; they don't hydrate a value.) See [querying.md — Conditions on associations in the chain](querying.md).
- **Accessing it when null throws `MissingRequiredBelongsToAssociation`.** When an *internal* mechanism nulls a required parent — a default scope (e.g. soft-delete) filtered it out, an `and`/`on` baked into the association definition excluded it, or the parent row was hard-deleted leaving a dangling FK — the getter throws this error instead of returning a null the types say is impossible.

The runtime error is a modeling bug, so the fix is a model change, not a looser spec:

- add `dependent: 'destroy'` to the inverse `HasOne` / `HasMany` so the parent can't be orphaned, or
- mark the `BelongsTo` `optional: true` if absence is legitimate — which makes the OpenAPI field nullable and re-allows conditional loading.

#### Anchor polymorphism to a stable model

When a record participates polymorphically in more than one direction, don't stack the polymorphic ownership directly onto a model that also has to stay generic elsewhere — the association shape becomes ambiguous and call sites get hard to read. Introduce a stable join model that owns the participant polymorphism and acts as the fixed boundary; the other polymorphic axis hangs off that boundary instead of off the same generic model.

```text
ConversationParticipant         # stable table — the polymorphic boundary
  ├── participant: polymorphic Guest | Host
  └── conversationThreads
ConversationThread
  ├── conversationParticipantId  # plain BelongsTo to the stable model
  └── context: polymorphic Booking | Place | ...
```

`ConversationParticipant` is the constant table: its `participant` association resolves to the domain record (`Guest` or `Host`), and the second polymorphic axis (`context`) lives on `ConversationThread` rather than being layered onto the participant. The call site stays readable — `conversationParticipant.participant` and `conversationThread.context` — and each model has exactly one polymorphic association to reason about.

### HasMany

Foreign key lives on the ASSOCIATED model.

```typescript
@deco.HasMany('Post')
public posts: Post[]

// With cascade delete
@deco.HasMany('Post', { dependent: 'destroy' })
public posts: Post[]

// With order
@deco.HasMany('Room', { order: 'position' })
public rooms: Room[]

// With conditions on the associated model
@deco.HasMany('Post', { and: { published: true } })
public publishedPosts: Post[]

// OR conditions — matches posts that are EITHER featured OR published
@deco.HasMany('Post', { andAny: [{ featured: true }, { published: true }] })
public visiblePosts: Post[]

// NOT conditions — excludes archived posts
@deco.HasMany('Post', { andNot: { archived: true } })
public activePosts: Post[]

// selfAnd — ADDS a condition on top of the normal FK join: an associated column must
// equal a column on THIS model. Format: { associatedColumn: 'thisModelColumn' }.
// (`peakSeasonMonth` on Place and `checkInMonth` on Booking are illustrative columns.)
@deco.HasMany('Booking', { selfAnd: { checkInMonth: 'peakSeasonMonth' } })
public peakSeasonBookings: Booking[]
// SQL: bookings.place_id = this.id  AND  bookings.check_in_month = this.peak_season_month
//      └──── normal FK join (kept) ───┘    └──────── selfAnd condition (added) ────────┘

// selfAndNot — excludes associated records where the columns match. A room's siblings
// are the other rooms in its Place, excluding itself.
@deco.HasMany('Room', { through: 'place', source: 'rooms', selfAndNot: { id: 'id' } })
public siblingRooms: Room[]
// SQL: ... AND rooms.id != this.id   (joined through the shared Place)

// Through (many-to-many)
@deco.HasMany('Host', { through: 'hostPlaces' })
public hosts: Host[]

// Nested through
@deco.HasMany('PostComment', { through: 'posts', source: 'comments' })
public postComments: PostComment[]

// Polymorphic
@deco.HasMany('LocalizedText', { polymorphic: true, on: 'localizableId', dependent: 'destroy' })
public localizedTexts: LocalizedText[]

// With distinct
@deco.HasMany('Tag', { through: 'postTags', distinct: true })
public uniqueTags: Tag[]

// Custom FK column on the associated table (override the conventional `host_id`)
@deco.HasMany('Place', { on: 'ownerId' })
public properties: Place[]
// SQL: places.owner_id = this.id
// (For `on` + `primaryKeyOverride` together — joining on a natural key — see below.)

// Skip default scopes when loading
@deco.HasMany('Booking', { withoutDefaultScopes: ['dream:SoftDelete'] })
public allBookings: Booking[]  // includes soft-deleted bookings
```

#### HasMany Options

| Option | Type | Description |
|--------|------|-------------|
| `and` | conditions object | WHERE conditions on the associated model's columns. Supports `DreamConst.passthrough` and `DreamConst.required` values |
| `andAny` | conditions object[] | OR conditions — association matches records satisfying ANY of the condition sets |
| `andNot` | conditions object | NOT conditions — excludes associated records matching these conditions |
| `dependent` | `'destroy'` | Cascade-delete the associated record(s) when this record is destroyed. **This is the standard answer when destroying a parent fails because of an FK constraint from a child.** Add `dependent: 'destroy'` to the parent's HasMany/HasOne — do not write a `BeforeDestroy` hook to manually delete children, and do not change the FK to `ON DELETE CASCADE` at the database level (which would bypass Dream's lifecycle and soft-delete handling). |
| `distinct` | column name \| boolean | Apply DISTINCT to the query (pass `true` for the primary key, or a specific column name) |
| `on` | column name | Custom foreign key column name on the associated model |
| `order` | column name \| order object | Default ordering for the association |
| `polymorphic` | boolean | Enables polymorphic association (requires `on`) |
| `primaryKeyOverride` | column name \| null | Override the primary key column used for the join |
| `selfAnd` | `{ associatedCol: 'thisCol' }` | Adds a join condition **on top of the normal FK join** — an associated column must equal a column on this model (e.g., `{ position: 'featuredRoomPosition' }` → `AND associated.position = this.featured_room_position`) |
| `selfAndNot` | `{ associatedCol: 'thisCol' }` | Inverse of `selfAnd` — excludes associated records where the columns match (e.g., `{ id: 'id' }` → `WHERE associated.id != this.id`) |
| `source` | association name | For through associations: the association name on the intermediate model |
| `through` | association name | Load through an intermediate association (many-to-many) |
| `withoutDefaultScopes` | scope name[] | Default scopes to skip when loading this association |

**Through associations** (`through` option) cannot use: `dependent`, `primaryKeyOverride`, `withoutDefaultScopes`, `on`, or `polymorphic`.

### HasOne

Like HasMany but returns a single instance.

```typescript
@deco.HasOne('Profile')
public profile: Profile

// With cascade delete
@deco.HasOne('Profile', { dependent: 'destroy' })
public profile: Profile

// With condition using passthrough
@deco.HasOne('LocalizedText', {
  polymorphic: true,
  on: 'localizableId',
  and: { locale: DreamConst.passthrough },
})
public currentLocalizedText: LocalizedText

// With required condition — query will fail if the passthrough value is not provided
@deco.HasOne('LocalizedText', {
  polymorphic: true,
  on: 'localizableId',
  and: { locale: DreamConst.required },
})
public requiredLocalizedText: LocalizedText

// OR conditions
@deco.HasOne('Address', { andAny: [{ type: 'home' }, { primary: true }] })
public primaryAddress: Address

// NOT conditions
@deco.HasOne('Profile', { andNot: { deactivated: true } })
public activeProfile: Profile

// selfAnd — adds a condition on top of the FK join (see HasMany for details).
// A Place names its featured room by position (`featuredRoomPosition`), so rooms can be
// reordered without touching a foreign key.
@deco.HasOne('Room', { selfAnd: { position: 'featuredRoomPosition' } })
public featuredRoom: Room
// SQL: rooms.place_id = this.id  AND  rooms.position = this.featured_room_position

// Through
@deco.HasOne('CompositionAsset', { through: 'mainComposition' })
public mainCompositionAsset: CompositionAsset

// Skip default scopes
@deco.HasOne('Profile', { withoutDefaultScopes: ['dream:SoftDelete'] })
public profileIncludingDeleted: Profile
```

#### HasOne Options

| Option | Type | Description |
|--------|------|-------------|
| `and` | conditions object | WHERE conditions on the associated model's columns. Supports `DreamConst.passthrough` and `DreamConst.required` values |
| `andAny` | conditions object[] | OR conditions — association matches records satisfying ANY of the condition sets |
| `andNot` | conditions object | NOT conditions — excludes associated records matching these conditions |
| `dependent` | `'destroy'` | Cascade-delete the associated record when this record is destroyed |
| `on` | column name | Custom foreign key column name on the associated model |
| `polymorphic` | boolean | Enables polymorphic association (requires `on`) |
| `primaryKeyOverride` | column name \| null | Override the primary key column used for the join |
| `selfAnd` | `{ associatedCol: 'thisCol' }` | Adds a join condition **on top of the normal FK join** — an associated column must equal a column on this model (e.g., `{ position: 'featuredRoomPosition' }` → `AND associated.position = this.featured_room_position`) |
| `selfAndNot` | `{ associatedCol: 'thisCol' }` | Inverse of `selfAnd` — excludes associated records where the columns match (e.g., `{ id: 'id' }` → `WHERE associated.id != this.id`) |
| `source` | association name | For through associations: the association name on the intermediate model |
| `through` | association name | Load through an intermediate association |
| `withoutDefaultScopes` | scope name[] | Default scopes to skip when loading this association |

**Through associations** (`through` option) cannot use: `dependent`, `primaryKeyOverride`, `withoutDefaultScopes`, `on`, or `polymorphic`.

### Joining on a non-default key (`on` + `primaryKeyOverride`)

By default an association joins the conventional foreign key to the primary key — `associated.<model>_id = this.id`. Two options override the sides of that join, and used **together** they let you associate on any column pair, such as a natural key (a `uuid`) rather than the id-based FK:

- `on` — the column that **holds the reference**, on whichever side owns the FK. For `BelongsTo` it's a column on this model; for `HasMany` / `HasOne` it's a column on the associated model.
- `primaryKeyOverride` — the column the reference **matches against** on the other side, instead of the default `id`.

The join becomes `<fk-holder>.[on] = <other-side>.[primaryKeyOverride]`.

```typescript
// Associate Booking and Guest by a public `uuid` instead of the numeric `guest_id` → `id`.
// (`Guest.uuid` and `Booking.guestUuid` are illustrative columns.)

// Booking owns the reference:
@deco.BelongsTo('Guest', { on: 'guestUuid', primaryKeyOverride: 'uuid' })
public guest: Guest
public guestUuid: string

// Guest's inverse:
@deco.HasMany('Booking', { on: 'guestUuid', primaryKeyOverride: 'uuid' })
public bookings: Booking[]

// Both resolve to:  bookings.guest_uuid = guests.uuid
```

This is distinct from `selfAnd`: `on` / `primaryKeyOverride` change **which columns the join uses**, while `selfAnd` keeps the FK join and **adds** a second condition on top of it. And unlike `and` — which filters the associated row against a literal value — all three name real join columns.

### DreamConst in Association Conditions

The `and` option on HasMany and HasOne supports two special `DreamConst` values instead of fixed values:

```typescript
import { DreamConst } from '@rvoh/dream'

// passthrough — value is supplied at query time via passthrough context, optional
@deco.HasOne('LocalizedText', {
  and: { locale: DreamConst.passthrough },
})
public currentLocalizedText: LocalizedText

// required — like passthrough, but the query will raise an error if the value is not provided
@deco.HasOne('LocalizedText', {
  and: { locale: DreamConst.required },
})
public requiredLocalizedText: LocalizedText
```

- **`DreamConst.passthrough`** — the value is provided at query time via `.passthrough({ column: value })`. Throws `MissingRequiredPassthroughForAssociationAndClause` if the value is `undefined`.
- **`DreamConst.required`** — the value must be provided via `{ and: { column: value } }` conditions whenever the association is used in any query — `load`, `preload`, `innerJoin`, `leftJoin`, `associationQuery`, `association`, etc. Throws `MissingRequiredAssociationAndClause` if the value is not provided.

Both are required — the difference is *how* the value is supplied. Use `passthrough` when the value comes from broad query context (e.g., locale from a request header). Use `required` when the value is specific to a particular association loading call.

**Neither can be combined with `dependent: 'destroy'`.**

```typescript
// passthrough — value supplied via .passthrough() on the query
@deco.HasOne('LocalizedText', {
  polymorphic: true, on: 'localizableId',
  and: { locale: DreamConst.passthrough },
})
public currentLocalizedText: LocalizedText

// Query must include .passthrough({ locale: 'en-US' })
await Place.passthrough({ locale: 'en-US' }).preloadFor('default').all()

// undefined causes MissingRequiredPassthroughForAssociationAndClause — use null instead
await Place.passthrough({ locale: this.currentUser?.locale ?? null }).preloadFor('default').all()

// required — value supplied via { and: {...} } in the chain
@deco.HasOne('Preference', {
  and: { category: DreamConst.required },
})
public preference: Preference

// The value must be provided in any query that touches this association:
await user.load('preference', { and: { category: 'notifications' } }).execute()
await user.associationQuery('preference').where({ category: 'notifications' }).first()
await User.preload('preference', { and: { category: 'notifications' } }).all()
await User.innerJoin('preference', { and: { category: 'notifications' } }).all()
```

## Hooks - Full Reference

### Before Hooks (run in same transaction, before DB write)

```typescript
@deco.BeforeCreate()
public setDefaults(this: Place) {
  if (!this.status) this.status = 'draft'
}

@deco.BeforeUpdate()
public trackChanges(this: Place) { ... }

@deco.BeforeSave()  // Runs on BOTH create and update
public normalizeEmail(this: User) {
  if (this.email) this.email = this.email.toLowerCase()
}

@deco.BeforeDestroy()
public validateCanDelete(this: Place) { ... }

// Conditional hooks
@deco.BeforeSave({ ifChanging: ['email'] })
public validateNewEmail(this: User) { ... }
```

**Do not call `save()` or `update()` on `this` from inside its own lifecycle hook.** If the hook is changing attributes on the record being persisted, use a before hook and assign the values directly so Dream persists them in the original write:

```typescript
// GOOD: same-record changes belong in a before hook.
@deco.BeforeSave()
public normalizeAddress(this: Place) {
  if (this.city) this.city = this.city.trim()
}

// BAD: this starts another persistence cycle from inside this model's hook.
@deco.AfterSave()
public async normalizeAddress(this: Place) {
  if (this.city) await this.update({ city: this.city.trim() })
}
```

Use after hooks for side effects that require the row to already exist, such as creating associated records or delegating to a service. If that side effect writes other records, pass the provided `txn` through; if it must run after commit, use a `Commit` hook variant.

### After Hooks (run in same transaction, after DB write)

```typescript
import { Dream, DreamTransaction } from '@rvoh/dream'

@deco.AfterCreate()
public async createDefaults(this: Host, txn: DreamTransaction<Dream> | null) {
  // Inline for simple cases
  await this.createAssociation('profile', {}, { txn })
}

@deco.AfterCreate()
public async seedRooms(this: Place, txn: DreamTransaction<Dream> | null) {
  // Delegate to a service for non-trivial logic; pass txn so all writes are atomic
  await SeedDefaultRooms.seed(this, txn)
}

@deco.AfterUpdate({ ifChanged: ['status'] })
public logStatusChange(this: Place) { ... }

@deco.AfterSave()  // Both create and update
public updateCache(this: Place) { ... }

@deco.AfterDestroy()
public cleanupReferences(this: Place) { ... }
```

**Hook signature.** Non-commit hooks (`AfterCreate`, `AfterUpdate`, `AfterSave`, `AfterDestroy`, and their `Before*` counterparts) receive the current transaction as a single argument. Type it as `DreamTransaction<Dream> | null` (imported from `@rvoh/dream`). Pass it through to any service the hook delegates to so all DB writes participate in the same transaction:

```typescript
import { Dream, DreamTransaction } from '@rvoh/dream'

export default class SeedDefaultRooms {
  public static async call(place: Place, txn: DreamTransaction<Dream> | null = null) {
    await Bedroom.txn(txn).create({ place })
    await Bathroom.txn(txn).create({ place })
  }
}
```

**`txn` is `null` unless the caller explicitly started a transaction.** Dream does NOT automatically wrap `Model.create()` in a transaction. The `txn` parameter is only non-null when the caller wraps the create in `ApplicationModel.transaction(async txn => await Model.txn(txn).create(...))`. Hooks should still accept and propagate `txn` so that *if* the create happens inside a transaction, the side effects are atomic with it.

`Model.txn(null).create(...)` is a no-op variant — the same service can be called from a hook (with `txn` possibly null) or from a controller (with `null`).

**Commit hooks do not receive a txn argument** because the transaction has already committed by the time they run.

### After Commit Hooks (run AFTER transaction commits)

**CRITICAL**: Always use commit hooks when queuing background jobs from model lifecycle hooks. This applies whether the hook calls a backgrounded service or backgrounds the model itself. It prevents races where the job runs before the transaction commits, so the worker cannot see a newly-created row or sees stale persisted data after an update.

```typescript
@deco.AfterCreateCommit()
public sendWelcomeEmail(this: User) {
  await EmailService.background('sendWelcome', this.id)
}

@deco.AfterUpdateCommit({ ifChanged: ['status'] })
public notifyStatusChange(this: Place) {
  await NotificationService.background('statusChanged', this.id)
}

@deco.AfterSaveCommit()
public invalidateCache(this: Place) { ... }

@deco.AfterDestroyCommit()
public removeFromSearchIndex(this: Place) { ... }
```

**Gating on an encrypted field.** When the gated field is an `@deco.Encrypted` field, list the persisted column name `encrypted<Name>` in `ifChanged` (e.g. `ifChanged: ['encryptedPhone']`), not the plaintext virtual (`phone`). `ifChanged` is typed over the real persisted columns (`DreamColumnNames`); setting the virtual marks the underlying `encrypted<Name>` column dirty, which is what change detection sees.

## Validations

```typescript
@deco.Validates('presence')
public name: DreamColumn<Place, 'name'>

@deco.Validates('length', { min: 4, max: 64 })
public email: DreamColumn<User, 'email'>

@deco.Validates('contains', '@')
public email: DreamColumn<User, 'email'>

@deco.Validates('numericality', { min: 0, max: 100 })
public volume: DreamColumn<User, 'volume'>

@deco.Validates('requiredBelongsTo')
public user: User  // Validates the association exists

// Custom validation method
@deco.Validate()
public validateWeightConsistency(this: Sandbag) {
  if (this.weight && this.weightKgs) {
    this.addError('weight', 'cannot include both weight and weightKgs')
  }
}
```

Checking validity:
```typescript
const user = User.new({ email: 'bad' })
if (user.isInvalid) {
  console.log(user.errors) // { email: ['must contain @'] }
}
```

## Scopes

```typescript
// Named scope - called explicitly
@deco.Scope()
public static active(query: Query<Place>) {
  return query.where({ status: 'active' })
}

// Default scope - applied to ALL queries automatically
@deco.Scope({ default: true })
public static hideArchived(query: Query<Post>) {
  return query.where({ archivedAt: null })
}

// Usage
const places = await Place.scope('active').all()

// Bypass default scopes
const withArchived = await Post.removeDefaultScope('hideArchived').all()
const specificPlace = await Place.removeAllDefaultScopes().findOrFail(id)
```

### Default Scopes

Default scopes are automatically applied to every query. They are used sparingly for cross-cutting concerns that should always apply unless explicitly removed.

Dream provides two built-in default scopes:
- **`dream:SoftDelete`** — added by `@SoftDelete()`, hides records where `deletedAt` is not null. See [soft-delete.md](soft-delete.md).
- **`dream:STI`** — added by `@STI()`, restricts STI child queries to their type

Models can also define custom default scopes:

```typescript
@deco.Scope({ default: true })
public static hideArchived(query: Query<Post>) {
  return query.where({ archivedAt: null })
}
```

### Removing Default Scopes

Use `removeDefaultScope('scopeName')` to remove a specific scope, or `removeAllDefaultScopes()` to remove all of them. Both work on model classes and query chains.

**Which to use:** Use `removeDefaultScope` when querying for a plurality (e.g. `.all()`) so that other default scopes remain in effect and don't bring extra records into scope. Use `removeAllDefaultScopes` when targeting a specific record (e.g. `.find(id)`, `.findOrFail(id)`) where you want to ensure the record is found regardless of any default scope.

```typescript
// Plurality — remove only the scope you need to bypass
await Place.removeDefaultScope('dream:SoftDelete').where({ style: 'cabin' }).all()
await Post.removeDefaultScope('hideArchived').all()

// Specific record — safe to remove all scopes since you're targeting one record
await Place.removeAllDefaultScopes().findOrFail(id)
```

Both methods also work on query chains:

```typescript
await user.associationQuery('places').removeDefaultScope('dream:SoftDelete').all()
```

### Inspecting Default Scopes (for AI agents)

The `schema` exported from `api/src/types/dream.ts` (`src/types/dream.ts` for api-only applications) contains the default scopes for any model:

```typescript
import schema from '@src/types/dream.js'

// schema[dreamModelInstance.table].scopes.default
// Returns an array of strings, e.g. ['dream:STI', 'dream:SoftDelete']
```

## Query Operators

```typescript
import { ops } from '@rvoh/dream'

// Comparison
ops.greaterThan(value)
ops.greaterThanOrEqualTo(value)
ops.lessThan(value)
ops.lessThanOrEqualTo(value)
ops.equal(value)
ops.not.equal(value)

// Collections
ops.in([1, 2, 3])
ops.not.in([1, 2, 3])

// String matching
ops.like('%pattern%')      // PostgreSQL LIKE — operator itself is case-sensitive
ops.ilike('%PATTERN%')     // PostgreSQL ILIKE — operator forces case-insensitive
ops.match('^[A-Z]')        // Regex
ops.match('^[A-Z]', { caseInsensitive: true })

// Fuzzy search (requires pg_trgm extension + GIN index)
ops.similarity('search term')
ops.wordSimilarity('search term')
ops.similarity('term', { score: 0.2 })

// Array containment
ops.any(5)  // Array contains value

// NULL checks
await Model.where({ field: null }).all()      // IS NULL
await Model.whereNot({ field: null }).all()    // IS NOT NULL
```

**`ops.like` vs `ops.ilike` — column-type-dependent behavior.** Against a regular `text` / `varchar` column, `ops.like` is case-sensitive and `ops.ilike` is case-insensitive — the operators differ. Against a `citext` column (case-insensitive text — see [migrations.md "citext"](migrations.md)), both operators match case-insensitively because the type itself ignores case; equality (`where({ name: 'sally' })`) on citext is case-insensitive for the same reason.

Because the case-sensitivity of `ops.like` is invisible at the call site (it depends on the column's type, not the operator), prefer `ops.ilike` when case-insensitivity is the intended semantic — the operator name documents the intent. Reach for `ops.like` only when case-sensitivity is intentional, in which case the column should be `text` / `varchar`, not `citext`.

## Special Decorators

```typescript
// Soft Delete - adds deletedAt-based soft deletion and a default scope (dream:SoftDelete)
// that hides deleted records. See soft-delete.md for full documentation.
@SoftDelete()
export default class Place extends ApplicationModel { ... }

// Sortable - maintains 1-based position within scope
@deco.Sortable({ scope: 'place' })
public position: DreamColumn<Room, 'position'>
// Requires deferrable unique constraint in migration

// Encrypted - auto encrypt/decrypt
@deco.Encrypted()
public phone: DreamColumn<User, 'encryptedPhone'>
// DB column is 'encrypted_phone', model property is 'phone'
// extractParams and model-derived OpenAPI request bodies use the public property
// name (`phone`), not the encrypted DB column name. The accepted type is string
// or string | null depending on whether the backing encrypted column is nullable;
// extractParams enforces that type at runtime and in TypeScript.
//
// Column encryption must be configured before the getter/setter will work. This is
// a Dream-app config in conf/dream.ts (distinct from the cookie encryption config in
// conf/app.ts — see controllers.md "Cookie Encryption"):
//   // conf/dream.ts — app is the DreamApp passed to the default export
//   app.set('encryption', {
//     columns: {
//       current: { algorithm: 'aes-256-gcm', key: AppEnv.string('COLUMN_ENCRYPTION_KEY') },
//     },
//   })
// Add an optional `legacy` key beside `current` to rotate keys (current is tried
// first, then legacy). Generate keys with `pnpm psy g:encryption-key`. Without this
// config the getter/setter throw. After adding @deco.Encrypted() and its migration,
// run `pnpm psy sync` so the `encrypted<Column>` column is in the generated types. The
// plaintext property (e.g. `phone`) is not assignable in `.create()` / `UpdateableProperties`
// until sync lists it under the model's `virtualColumns`; until then `tsc` reports
// `TS2353 ... 'phone' does not exist` even though the model declares the field.

// Virtual — accepted by create(), update(), and extractParams() but not stored directly in DB.
// Use getter/setter pairs to transform between the virtual and the actual DB column.
// Decorator goes on whichever accessor is declared first.
// Note the use of `DreamColumn<Place, 'grams'>` to specify the type of the
// getter return type and setter params because these types are simply
// mutations of what is in the database and the database is the source
// of truth for the type.
const GRAMS_PER_POUND = 453.592

@deco.Virtual(['number', 'null'])
public get pounds(): DreamColumn<Place, 'grams'> {
  const grams = this.getAttribute('grams')
  return grams === null ? null : grams / GRAMS_PER_POUND
}

public set pounds(pounds: DreamColumn<Place, 'grams'>) {
  this.setAttribute('grams', pounds === null ? null : pounds * GRAMS_PER_POUND)
}

// All Virtual types: 'boolean', 'date-time', 'date', 'integer', 'null', 'number',
// 'string', 'decimal', 'json', and array variants: 'boolean[]', 'date-time[]',
// 'date[]', 'decimal[]', 'integer[]', 'number[]', 'string[]'
// Use arrays for nullable: ['string', 'null']
//
// Virtual getters and setters MUST be synchronous. Dream reads the getter as a
// plain attribute, not as a Promise. An async getter returns a Promise instead
// of the declared type, which silently breaks serialization and is invisible to
// `preloadFor`'s DSL introspection. If you need data that requires an `await`
// (a DB lookup, a remote URL, a cross-association walk), express it as a real
// association (@deco.HasOne / @deco.BelongsTo) and surface it via
// .delegatedAttribute(...) or .rendersOne(...) in the serializer — never via
// a virtual.
//
// The type argument to @deco.Virtual(...) sets the OpenAPI shape wherever the
// attribute is surfaced in a serializer or model-derived OpenAPI request body,
// and it is the type extractParams enforces at runtime and in TypeScript. Just
// as a real column's OpenAPI shape is derived from the database schema, a
// virtual's shape is derived from the decorator's type argument.
// @deco.Virtual(['number', 'null']) produces and accepts a `number | null` shape
// in any serializer that exposes the attribute via .attribute('pounds') or
// .delegatedAttribute('foo', 'pounds'), and in requestBody.params when `pounds`
// is part of the accepted params. Don't pass an explicit `openapi:` to those
// serializer methods when surfacing a virtual — it overrides the auto-inferred
// shape and creates a place for the spec to drift when the virtual's declared
// type changes.

// STI - Single Table Inheritance child
@STI(Room)
export default class Bedroom extends Room { ... }
```

## Transforming a column on write

To normalize or clamp an *input* as it is written to a real column (lowercase an email, bound a number, coerce null), define a custom getter/setter pair on the model over the column's own name. Read through `getAttribute` and write through `setAttribute`: `setAttribute` writes the attribute bag directly and does **not** re-enter your setter, so it avoids the infinite recursion that `this.email = ...` inside the setter would cause.

```typescript
public get email(): DreamColumn<User, 'email'> {
  return this.getAttribute('email')
}

public set email(value: DreamColumn<User, 'email'>) {
  this.setAttribute('email', value === null ? null : value.toLowerCase())
}
```

This is distinct from `@deco.Virtual`: a virtual backs a property that is not a column (or maps to a differently-named one, like `pounds` → `grams` above), whereas here the accessors wrap the column under its own name. No decorator is needed — the model's accessor simply overrides the one Dream installs.

**Which writes run the setter.** Dream splits attribute assignment into two paths:

- `create()`, `update()`, `assignAttribute` / `assignAttributes`, and a plain `this.email = …` **run** your custom setter (`assignAttribute` is literally `this[attr] = value`).
- `setAttribute` / `setAttributes` (plural) **bypass** custom setters and write the bag raw — use these when you deliberately want the untransformed value, and inside the setter body itself.

So input arriving through `create` / `update` / extracted params is transformed, while internal hydration (loading a row from the DB) writes raw and is never double-transformed.

## Creating and Updating

**Instantiation**: Use `ModelName.new()` to create an unpersisted model instance. `new ModelName()` does not work and will throw a type error.

```typescript
// Create in one call
const user = await User.create({ email: 'charlie@peanuts.com', name: 'Charlie Brown' })

// Instantiate, assign, and save
const user = User.new({ email: 'charlie@peanuts.com' })
user.name = 'Charlie Brown'
await user.save()

// Update in one call
await user.update({ name: 'New Name' })

// Assign and save — equivalent to update
user.name = 'New Name'
await user.save()

// Assign and update — also valid (update persists any prior assignments too)
user.name = 'New Name'
await user.update({ email: 'gary@farside.com' })

// Skip lifecycle hooks — useful for bulk imports and historical data backfills
// where the new-record side effects shouldn't fire
await User.create({ email: 'imported@example.com' }, { skipHooks: true })
await user.update({ name: 'Updated' }, { skipHooks: true })
```

`save()` and `update()` trigger the same lifecycle hooks and are largely interchangeable:
- `save()` persists all dirty attributes but does not accept attributes itself
- `update()` accepts attributes, applies them, and persists — it also persists any attributes previously assigned via `=`
- On an unpersisted instance (`User.new()`), both `save()` and `update({})` will create the record

Query-level updates also run instance hooks by default: `User.where(...).update(attrs)`
loads each matched record and calls instance `.update()` on it. It is not a single
bulk SQL update unless you pass `{ skipHooks: true }`, which bypasses hooks and
validations.

## Dirty Tracking

```typescript
const user = await User.findOrFail(id)
user.name = 'New Name'

user.isDirty()                    // true
user.isClean()                    // false
user.changedAttributes()          // { name: 'New Name' }
user.hasChanges('name')           // true
user.hasChanges('email')          // false
```

`changedAttributes()` works before the first save too. `User.new({ name: 'Alice' })` marks `name` dirty immediately, so `changedAttributes()` is populated on the unpersisted instance.

## Batch Processing

```typescript
// Process records in batches (default batch size: 1000)
await User.findEach(async (user) => {
  await user.update({ processedAt: DateTime.now() })
})

// With custom batch size
await User.findEach({ batchSize: 100 }, async (user) => { ... })

// Pluck in batches
await User.pluckEach('email', async (email) => {
  await sendEmail(email)
})
```

## Find-or-Create and Upsert Methods

Dream provides four methods for atomically finding or creating/updating records. Each takes a set of lookup attributes and an options object. None of these methods can be chained with other query methods.

### findOrCreateBy

Finds a record matching the given attributes, or creates one if none exists. Safe to use in transactions.

```typescript
const user = await User.findOrCreateBy(
  { email: 'how@yadoin' },
  { createWith: { name: 'Chalupa Joe' } }
)

// With a transaction
await ApplicationModel.transaction(async txn => {
  const user = await User.txn(txn).findOrCreateBy(
    { email: 'how@yadoin' },
    { createWith: { name: 'Chalupa Joe' } }
  )
})
```

`createWith` attributes are applied only when creating a new record. There is a race condition risk between the find and create steps — if concurrent requests could match the same lookup attributes, prefer `createOrFindBy`.

### createOrFindBy

Like `findOrCreateBy`, but attempts to create first. Relies on a **unique constraint** at the DB level to reject the create if a matching record already exists, then falls back to finding it. This avoids the race condition in `findOrCreateBy`. **Cannot be used in a transaction** (since it relies on the create failing at the DB level).

```typescript
// Requires a unique index on the lookup column(s)
const user = await User.createOrFindBy(
  { email: 'how@yadoin' },
  { createWith: { name: 'Chalupa Joe' } }
)
```

### updateOrCreateBy

Finds a record and updates it, or creates one if none exists. Functionally equivalent to an upsert, but runs as two queries. Safe to use in transactions. There is a race condition risk between the find and create steps — if concurrent requests could match the same lookup attributes, prefer `createOrUpdateBy`.

```typescript
const user = await User.updateOrCreateBy(
  { email: 'how@yadoin' },
  { with: { name: 'New Name' } }
)

// With a transaction
await ApplicationModel.transaction(async txn => {
  const user = await User.txn(txn).updateOrCreateBy(
    { email: 'how@yadoin' },
    { with: { name: 'New Name' } }
  )
})
```

### createOrUpdateBy

Like `updateOrCreateBy`, but attempts to create first. Relies on a **unique constraint** at the DB level to reject the create if a matching record already exists, then falls back to updating. This avoids the race condition in `updateOrCreateBy`. **Cannot be used in a transaction**.

```typescript
// Requires a unique index on the lookup column(s)
const user = await User.createOrUpdateBy(
  { email: 'how@yadoin' },
  { with: { name: 'New Name' } }
)
```

### Skipping hooks

All four methods run lifecycle hooks by default. Pass `skipHooks: true` to bypass them:

```typescript
const user = await User.updateOrCreateBy(
  { email: 'how@yadoin' },
  { with: { name: 'New Name' }, skipHooks: true }
)
```

### Which method to use

| Method | Order | Requires unique constraint | Transaction-safe | Use when |
|--------|-------|---------------------------|------------------|----------|
| `findOrCreateBy` | find, then create | No | Yes | Finding is the common case, creating is rare |
| `createOrFindBy` | create, then find | **Yes** | **No** | Concurrent requests may race on the same lookup |
| `updateOrCreateBy` | find, then upsert | No | Yes | Need to update existing or create new |
| `createOrUpdateBy` | create, then update | **Yes** | **No** | Concurrent requests may race on the same lookup |

## Transactions

Two ways to start a transaction:

1. **Class-level** — `ApplicationModel.transaction(async (txn) => { ... })`
2. **Instance-level** — `someModel.transaction(async (txn) => { ... })`

If the callback throws, the entire transaction rolls back.

**Every model operation inside a transaction must be explicitly bound via `.txn(txn)`.** This includes creates, updates, destroys, queries, association operations — everything. If you forget `.txn(txn)`, the operation runs outside the transaction and won't roll back on failure.

```typescript
// Class-level transaction
await ApplicationModel.transaction(async (txn) => {
  const user = await User.txn(txn).create({ email: 'test@test.com' })
  const post = await Post.txn(txn).create({ userId: user.id, title: 'Test' })
  await user.txn(txn).createAssociation('profile', {})

  // Queries also need .txn(txn) to see uncommitted data
  const found = await User.txn(txn).findBy({ email: 'test@test.com' })
})

// Instance-level transaction
await user.transaction(async (txn) => {
  await user.txn(txn).update({ status: 'active' })
  await user.txn(txn).createAssociation('profile', {})
})
```

### Optional transaction participation with `.txn(null)`

`.txn()` accepts `null` as well as a `DreamTransaction`. When `null` is passed, the operation runs without a transaction — it's a no-op. This lets methods that _optionally_ participate in a transaction use a single code path instead of branching:

```typescript
async function doWork(user: User, txn: DreamTransaction<ApplicationModel> | null = null) {
  // No if/else needed — .txn(null) is a no-op
  await user.txn(txn).update({ status: 'done' })
  await user.txn(txn).createAssociation('posts', { title: 'New' })
}

// Call with a transaction
await ApplicationModel.transaction(async txn => {
  await doWork(user, txn)
})

// Call without — works the same, just no transaction wrapping
await doWork(user)
```

Both `Model.txn(null)` (class-level) and `instance.txn(null)` (instance-level) work the same way.

**Restrictions inside transactions:** Methods that rely on foreign key violations to function (`createOrFindBy`, `createOrUpdateBy`) cannot be used inside a transaction. Use their transaction-safe counterparts (`findOrCreateBy`, `updateOrCreateBy`) instead. See the [find-or-create methods](#find-or-create-and-upsert-methods) table for details.

**Background jobs and transactions:** When queuing background work from a Dream lifecycle hook, always use the `Commit` variant of the hook (e.g., `@deco.AfterCreateCommit` instead of `@deco.AfterCreate`). This applies to backgrounded services, model instance backgrounding, and indirect helper methods that enqueue jobs. Regular hooks run inside the transaction, so the worker may execute before the transaction commits and either miss a newly-created row or read stale persisted data after an update. This also applies to imperative `background()` calls from services with a `txn` parameter — see [workers.md](workers.md) for both forms.

**What goes inside `ApplicationModel.transaction(...)` is database writes only.** HTTP fetches, S3 / object-storage uploads, sending email, calling third-party APIs, file I/O, sleeps, and any other non-DB work belong outside the transaction.

Holding an open transaction across long-running I/O has three failure modes:

1. **Connection pool starvation.** A long-running txn pins a Postgres connection. Other requests block waiting for a free connection.
2. **Vacuum interference.** Open transactions hold the horizon for vacuum, preventing dead-tuple cleanup on hot tables until the txn closes.
3. **BullMQ race exposure.** If work inside the txn enqueues background jobs, workers dequeue immediately and race the commit.

**Do the external I/O *before* the transaction**, then record the result in a tight transaction at the end. If you invert this — commit first, then do the I/O — and the I/O fails, the database will claim work happened that actually didn't, and you have to write reconciliation code to find and recover those records.

```typescript
// WRONG — long-running I/O inside the txn
await ApplicationModel.transaction(async txn => {
  for (const record of records) {
    const place = await Place.txn(txn).create({ ... })
    await fetch(record.photoUrl)                          // ← network I/O
    await s3.send(new PutObjectCommand({ ... }))          // ← network I/O
  }
})

// ALSO WRONG — DB says the work happened, but if the I/O fails, external state is missing
const created: Place[] = []
await ApplicationModel.transaction(async txn => {
  for (const record of records) {
    created.push(await Place.txn(txn).create({ ... }))
  }
})
for (const place of created) {
  await s3.send(new PutObjectCommand({ ... }))  // ← if this fails, the DB row is orphaned
}

// RIGHT — do the I/O first, then record the result in a single fast transaction
const prepared: Array<{ record: Record; bucketPath: string }> = []
for (const record of records) {
  const buffer = await (await fetch(record.photoUrl)).arrayBuffer()
  const bucketPath = PlacePhotoMediaService.objectKey(host.id, record)
  await s3.send(new PutObjectCommand({ Bucket, Key: bucketPath, Body: Buffer.from(buffer) }))
  prepared.push({ record, bucketPath })
}

await ApplicationModel.transaction(async txn => {
  for (const { record, bucketPath } of prepared) {
    await Place.txn(txn).create({ ...record, bucketPath })
  }
})
```

If the I/O phase fails partway, nothing is in the database and the caller can safely retry. The transaction at the end is short, predictable, and never held open across network round-trips. If a later failure requires cleanup, orphaned S3 objects are cheaper to reconcile than orphaned DB rows pointing at nothing.

## Date/Time

**NEVER use JavaScript `Date`. Always use Dream's `DateTime`, `CalendarDate`, `ClockTime`, and `ClockTimeTz` classes.**

```typescript
import { CalendarDate, ClockTime, ClockTimeTz, DateTime } from '@rvoh/dream'

// DateTime for timestamp columns
const now = DateTime.now()
const specific = DateTime.fromISO('2024-01-15T10:30:00')
const future = now.plus({ days: 7, hours: 3 })

// CalendarDate for date-only columns
const today = CalendarDate.today({ zone: 'America/New_York' })
const birthday = CalendarDate.fromISO('1990-05-15')
const nextWeek = today.plus({ days: 7 })

// ClockTime for time-of-day columns WITHOUT a time zone (e.g. business hours)
const opensAt = ClockTime.fromISO('09:30:00')

// ClockTimeTz for time-of-day columns WITH a time zone
const callWindowStart = ClockTimeTz.fromISO('14:00:00-05:00')
```

Each Dream date/time class maps to exactly one Postgres column type, and `pnpm psy sync` rewrites `src/types/db.ts` so the matching `DreamColumn<Model, '...'>` resolves to the right class automatically (the same mechanism that makes a `timestamp` column resolve to `DateTime`):

| Postgres column type | Dream class |
|---|---|
| `timestamp` | `DateTime` |
| `date` | `CalendarDate` |
| `time` (TIME WITHOUT TIME ZONE) | `ClockTime` |
| `timetz` (TIME WITH TIME ZONE) | `ClockTimeTz` |

A `ClockTimeTz` parsed from SQL is interpreted as UTC. These four classes are what `castParam` / `extractParams` return for date/time params and columns, and what Dream hydrates from the database — so any code touching those values already holds them; there is never a JS `Date` to reach for in the first place.

These classes carry **microsecond precision** — unusual for a TypeScript framework, and worth knowing: the fractional second is six digits (first three milliseconds, next three microseconds). Microseconds are preserved when a value is supplied via the API (`fromISO`, etc.) or hydrated from the database. `.now()` is backed by JavaScript `Date` and is therefore millisecond-only *at the moment of creation*; if you need sub-millisecond values, set them explicitly via `plus` / `set` after construction.
