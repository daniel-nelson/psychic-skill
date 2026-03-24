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

## Column Types

Use `DreamColumn<ModelClass, 'columnName'>` for all database-backed columns. The type is inferred from the database schema via Kysely codegen.

```typescript
public id: DreamColumn<User, 'id'>
public email: DreamColumn<User, 'email'>
public createdAt: DreamColumn<User, 'createdAt'>
public updatedAt: DreamColumn<User, 'updatedAt'>
```

## Decorator Setup

Each model creates its own decorator instance:

```typescript
import { Decorators } from '@rvoh/dream'
const deco = new Decorators<typeof MyModel>()
```

## Associations - Full Reference

### BelongsTo

Foreign key lives on THIS model. You must also declare the FK column.

```typescript
@deco.BelongsTo('User')
public user: User
public userId: DreamColumn<Post, 'userId'>

// Optional (nullable FK)
@deco.BelongsTo('User', { optional: true })
public approver: User | null
public approverId: DreamColumn<Post, 'approverId'>

// Custom FK name
@deco.BelongsTo('User', { on: 'authorId' })
public author: User
public authorId: DreamColumn<Post, 'authorId'>

// Polymorphic BelongsTo
@deco.BelongsTo(['Host', 'Place', 'Room'], { polymorphic: true, on: 'localizableId' })
public localizable: Host | Place | Room
public localizableType: DreamColumn<LocalizedText, 'localizableType'>
public localizableId: DreamColumn<LocalizedText, 'localizableId'>

// Override the primary key used for the join
@deco.BelongsTo('User', { primaryKeyOverride: 'externalId' })
public user: User
public userId: DreamColumn<Post, 'userId'>

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

// selfAnd — join where a column on the associated model matches a column on THIS model.
// Format: { associatedColumn: 'thisModelColumn' }
// Here, UserChallengeProgress links to DailyChallenge by matching position values
// instead of a direct FK, so DailyChallenges can be re-shuffled without affecting progress.
@deco.HasOne('DailyChallenge', {
  on: 'position',
  selfAnd: { position: 'currentPosition' },
})
public currentChallenge: DailyChallenge
// SQL: WHERE daily_challenges.position = user_challenge_progresses.current_position

// selfAndNot — exclude associated records where columns match.
// Here, a TreeNode's siblings are its parent's children excluding itself.
@deco.HasMany('TreeNode', {
  through: 'parent',
  source: 'children',
  selfAndNot: { id: 'id' },
})
public siblings: TreeNode[]
// SQL: WHERE tree_nodes.id != this.id

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

// Override the primary key used for the join
@deco.HasMany('Post', { primaryKeyOverride: 'externalId' })
public posts: Post[]

// Skip default scopes when loading
@deco.HasMany('Post', { withoutDefaultScopes: ['dream:SoftDelete'] })
public allPosts: Post[]  // includes soft-deleted posts
```

#### HasMany Options

| Option | Type | Description |
|--------|------|-------------|
| `and` | conditions object | WHERE conditions on the associated model's columns. Supports `DreamConst.passthrough` and `DreamConst.required` values |
| `andAny` | conditions object[] | OR conditions — association matches records satisfying ANY of the condition sets |
| `andNot` | conditions object | NOT conditions — excludes associated records matching these conditions |
| `dependent` | `'destroy'` | Cascade-delete the associated record(s) when this record is destroyed |
| `distinct` | column name \| boolean | Apply DISTINCT to the query (pass `true` for the primary key, or a specific column name) |
| `on` | column name | Custom foreign key column name on the associated model |
| `order` | column name \| order object | Default ordering for the association |
| `polymorphic` | boolean | Enables polymorphic association (requires `on`) |
| `primaryKeyOverride` | column name \| null | Override the primary key column used for the join |
| `selfAnd` | `{ associatedCol: 'thisCol' }` | Adds a join condition where a column on the associated model must equal a column on this model (e.g., `{ position: 'currentPosition' }` → `WHERE associated.position = this.currentPosition`) |
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

// selfAnd — see HasMany section for detailed examples
@deco.HasOne('DailyChallenge', {
  on: 'position',
  selfAnd: { position: 'currentPosition' },
})
public currentChallenge: DailyChallenge

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
| `selfAnd` | `{ associatedCol: 'thisCol' }` | Adds a join condition where a column on the associated model must equal a column on this model (e.g., `{ position: 'currentPosition' }` → `WHERE associated.position = this.currentPosition`) |
| `selfAndNot` | `{ associatedCol: 'thisCol' }` | Inverse of `selfAnd` — excludes associated records where the columns match (e.g., `{ id: 'id' }` → `WHERE associated.id != this.id`) |
| `source` | association name | For through associations: the association name on the intermediate model |
| `through` | association name | Load through an intermediate association |
| `withoutDefaultScopes` | scope name[] | Default scopes to skip when loading this association |

**Through associations** (`through` option) cannot use: `dependent`, `primaryKeyOverride`, `withoutDefaultScopes`, `on`, or `polymorphic`.

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

### After Hooks (run in same transaction, after DB write)

```typescript
@deco.AfterCreate()
public createDefaults(this: User) {
  await this.createAssociation('profile', {})
}

@deco.AfterUpdate({ ifChanged: ['status'] })
public logStatusChange(this: Place) { ... }

@deco.AfterSave()  // Both create and update
public updateCache(this: Place) { ... }

@deco.AfterDestroy()
public cleanupReferences(this: Place) { ... }
```

### After Commit Hooks (run AFTER transaction commits)

**CRITICAL**: Always use commit hooks when sending models to background jobs. This prevents race conditions where the job runs before the transaction commits.

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

@deco.Validates('inclusion', { in: ['active', 'inactive'] })
public status: DreamColumn<User, 'status'>

@deco.Validates('exclusion', { in: ['admin', 'root'] })
public username: DreamColumn<User, 'username'>

@deco.Validates('requiredBelongsTo')
public user: User  // Validates the association exists

// Custom validation method
@deco.Validate()
public validateWeightConsistency(this: Sandbag) {
  if (this.weight && this.weightKgs) {
    this.errors['weight'] = ['cannot include both weight and weightKgs']
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
ops.like('%pattern%')
ops.ilike('%PATTERN%')     // Case-insensitive
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

## Special Decorators

```typescript
// Soft Delete - adds deletedAt-based soft deletion and a default scope (dream:SoftDelete)
// that hides deleted records. See soft-delete.md for full documentation.
@SoftDelete()
export default class Place extends ApplicationModel { ... }

// Sortable - maintains position within scope
@deco.Sortable({ scope: 'place' })
public position: DreamColumn<Room, 'position'>
// Requires deferrable unique constraint in migration

// Encrypted - auto encrypt/decrypt
@deco.Encrypted()
public phone: DreamColumn<User, 'encryptedPhone'>
// DB column is 'encrypted_phone', model property is 'phone'

// Virtual - not stored in DB, not sent to DB on save
@deco.Virtual('string')
public password: string | undefined

@deco.Virtual('integer')
public age: number

@deco.Virtual(['string', 'null'])
public computedField: string | null

// All Virtual types: 'boolean', 'date-time', 'date', 'integer', 'null', 'number',
// 'string', 'decimal', 'json', and array variants: 'boolean[]', 'date-time[]',
// 'date[]', 'decimal[]', 'integer[]', 'number[]', 'string[]'
// Use arrays for nullable: ['string', 'null']

// STI - Single Table Inheritance child
@STI(Room)
export default class Bedroom extends Room { ... }
```

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
```

`save()` and `update()` trigger the same lifecycle hooks and are largely interchangeable:
- `save()` persists all dirty attributes but does not accept attributes itself
- `update()` accepts attributes, applies them, and persists — it also persists any attributes previously assigned via `=`
- On an unpersisted instance (`User.new()`), both `save()` and `update({})` will create the record

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

**Restrictions inside transactions:** Methods that rely on foreign key violations to function (`createOrFindBy`, `createOrUpdateBy`) cannot be used inside a transaction. Use their transaction-safe counterparts (`findOrCreateBy`, `updateOrCreateBy`) instead. See the [find-or-create methods](#find-or-create-and-upsert-methods) table for details.

**Background jobs and transactions:** When queuing background work from a Dream lifecycle hook, always use the `Commit` variant of the hook (e.g., `@deco.AfterCreateCommit` instead of `@deco.AfterCreate`). Regular hooks run inside the transaction, so the worker may execute before the transaction commits. See [workers.md](workers.md) for details.

## Date/Time

**NEVER use JavaScript `Date`. Always use Dream's DateTime and CalendarDate (powered by Luxon).**

```typescript
import { CalendarDate, DateTime } from '@rvoh/dream'

// DateTime for timestamp columns
const now = DateTime.now()
const specific = DateTime.fromISO('2024-01-15T10:30:00')
const future = now.plus({ days: 7, hours: 3 })

// CalendarDate for date-only columns
const today = CalendarDate.today({ zone: 'America/New_York' })
const birthday = CalendarDate.fromISO('1990-05-15')
const nextWeek = today.plus({ days: 7 })
```

